```dataviewjs
// ── Config Folder ────────────────────────────────────────────────────────────────────
const FOLDER = "/";
// ── Exclude these folders (supports partial matching)───────────────
const EXCLUDE_PATHS = [
	"_templates/",
	"attachments/",
	"Clippings/",
	"Life/Daily Notes/"
	];
	
	// ── Exclude these folders (supports partial matching)───────────────
const EXCLUDE_TAGS = [
	"#Homepage",
	"#foldernote"
	];
	
// ── Columns | Priority ────────────────────────────────────────────────────────────
const COLUMNS = [
  { id: "to-do", label: "To-Do" },
  { id: "in-progress", label: "In Progress" },
  { id: "on-hold", label: "On Hold" },
  { id: "someday", label: "Someday" },
  { id: "done", label: "Done" },
];
// ── Folder Colors (root folder name - left border color) ────────────────
const FOLDER_COLORS = {
	"Azmeen": "#003166",
	"Life": "#ff5757",
	"Corner": "#91b60c",
	"3Diculous": "#6c045e",
	"Fuel-Forge": "#Fb752d",
	};

// ── Things might break if you change after this line
// ── Persistent state ───────────────────────────────────────────────────────────
const STATE_KEY = "kb_" + dv.currentFilePath.replace(/[^\w]/g, "_");
if (!window[STATE_KEY]) {
  window[STATE_KEY] = { ov: {}, orderOv: {}, collapsed: {} };
}
const STATE = window[STATE_KEY];

// ── Ephemeral drag state ───────────────────────────────────────────────────────
const DRAG = { path: null };

// ── Load pages and filter out excluded paths ─────────────────────────────────────────────────────────────────
let pages = dv.pages(`"${FOLDER}"`).array();

// filter function: keep only those not in exclude list
pages = pages.filter(p => {
	const path = p.file.path;
	if (EXCLUDE_PATHS.some(ex => path.includes(ex))) return false;
	const noteTags = p.file.tags || [];
	if (EXCLUDE_TAGS.some(tag => noteTags.includes(tag))) return false;
	return true;
	});

// Purge stale overrides once dataview confirms the file was persisted
for (const p of Object.keys(STATE.ov)) {
  const pg = pages.find((x) => x.file.path === p);
  if (pg && pg.status === STATE.ov[p]) delete STATE.ov[p];
}
for (const p of Object.keys(STATE.orderOv)) {
  const pg = pages.find((x) => x.file.path === p);
  if (pg && pg.order === STATE.orderOv[p]) delete STATE.orderOv[p];
}

// ── File I/O — always background, never awaited in the hot path ───────────────
async function patchFrontmatter(path, patches) {
  const file = app.vault.getAbstractFileByPath(path);
  if (!file) return;
  let content = await app.vault.read(file);
  const m = content.match(/^---\r?\n([\s\S]*?)\r?\n---/);
  if (!m) return;
  let fm = m[1];
  for (const [key, val] of Object.entries(patches)) {
    const re = new RegExp(`^(${key}:\\s*).*$`, "m");
    fm = re.test(fm) ? fm.replace(re, `$1${val}`) : `${fm}\n${key}: ${val}`;
  }
  await app.vault.modify(file, content.replace(m[0], `---\n${fm}\n---`));
}

// ── Task builder — always reads live STATE ─────────────────────────────────────
function buildTasks() {
  return pages.map((p, i) => ({
    path: p.file.path,
    title: p.title || p.file.name,
    status: STATE.ov[p.file.path] ?? p.status ?? "to-do",
    progress: typeof p.progress === "number" ? p.progress : null,
    order:
      STATE.orderOv[p.file.path] ?? (typeof p.order === "number" ? p.order : i),
    tags: p.file.tags || [],
  }));
}

function byOrder(arr) {
  return [...arr].sort((a, b) => a.order - b.order);
}

// ── Drop UI ────────────────────────────────────────────────────────────────────
function clearDropUI() {
  dv.container
    .querySelectorAll(".kb-col-over")
    .forEach((el) => el.classList.remove("kb-col-over"));
}

// ── Commit: drag card to a different column (always appends to end) ────────────
function commitColumnChange(srcPath, targetColId) {
  const tasks = buildTasks();
  const src = tasks.find((t) => t.path === srcPath);
  if (!src || src.status === targetColId) return;

  const colTasks = byOrder(tasks.filter((t) => t.status === targetColId));
  const newOrder = colTasks.length
    ? Math.max(...colTasks.map((t) => t.order)) + 1
    : 0;
    

  STATE.ov[srcPath] = targetColId;
  STATE.orderOv[srcPath] = newOrder;

  render();

  patchFrontmatter(srcPath, { status: targetColId, order: newOrder }).catch(
    console.error,
  );
}

// ── Commit: move card up or down within its column ────────────────────────────
function commitMove(srcPath, dir) {
  const tasks = buildTasks();
  const src = tasks.find((t) => t.path === srcPath);
  if (!src) return;

  const colTasks = byOrder(tasks.filter((t) => t.status === src.status));
  const idx = colTasks.findIndex((t) => t.path === srcPath);
  const newIdx = dir === "up" ? idx - 1 : idx + 1;
  if (newIdx < 0 || newIdx >= colTasks.length) return;

  // Swap the two cards
  [colTasks[idx], colTasks[newIdx]] = [colTasks[newIdx], colTasks[idx]];

  // Reassign clean sequential order values for the whole column
  colTasks.forEach((t, i) => {
    STATE.orderOv[t.path] = i;
  });

  render();

  colTasks.forEach((t, i) => {
    patchFrontmatter(t.path, { order: i }).catch(console.error);
  });
}

// ── Helper: get root folder name from file path ───────────────────────────
function getRootFolder(path) {
	const parts = path.split('/');
	return parts.length > 1 ? parts[0] : ""; // root folder is first segment
}

// ── Render ─────────────────────────────────────────────────────────────────────
function render() {
  const root = dv.container;
  root.empty();

  const tasks = buildTasks();
  const board = root.createEl("div", { cls: "kb-board" });

  COLUMNS.forEach((col) => {
    const colTasks = byOrder(tasks.filter((t) => t.status === col.id));
    const colEl = board.createEl("div", { cls: "kb-col" });

    const hdr = colEl.createEl("div", { cls: "kb-col-hdr" });
    // Add a fold/unfold indicator & make it clickable
    const hdrLeft = hdr.createEl("div", { cls: "kb-col-hdr-left" });
    const foldIcon = hdrLeft.createEl("span", { cls: "kb-fold-icon", text: "▼" });
    hdr.createEl("span", { text: col.label });
    hdr.createEl("span", { cls: "kb-count", text: String(colTasks.length) });
    
    const cardList = colEl.createEl("div", { cls: "kb-cards" });
    
    // Read collapsed state (not default)
    if (!STATE.collapsed) STATE.collapsed = {};
    const isCollapsed = STATE.collapsed[col.id] || false;
    if (isCollapsed) foldIcon.textContent = "▶";
    
    // toggle on click
    hdr.style.cursor = "pointer";
    hdr.addEventListener("click", (e) => {
    // prevent toggle if click on move button or links inside header
	    e.stopPropagation();
	    const newState = !STATE.collapsed[col.id];
	    STATE.collapsed[col.id] = newState;
	    foldIcon.textContent = newState ? "▶" : "▼";
	    cardList.style.display = newState ? "none" : "flex";
	    });
    
    // Apply initial visibility
    cardList.style.display = isCollapsed ? "none" : "flex";
    

    // ── Column accepts drops from any card ─────────────────────────────────
    colEl.addEventListener("dragover", (e) => {
      if (!DRAG.path) return;
      e.preventDefault();
      e.dataTransfer.dropEffect = "move";
      colEl.classList.add("kb-col-over");
    });
    colEl.addEventListener("dragleave", (e) => {
      if (!colEl.contains(e.relatedTarget)) clearDropUI();
    });
    colEl.addEventListener("drop", (e) => {
      e.preventDefault();
      clearDropUI();
      const srcPath = DRAG.path;
      DRAG.path = null;
      if (srcPath) commitColumnChange(srcPath, col.id);
    });

    if (colTasks.length === 0) {
      cardList.createEl("div", { cls: "kb-empty", text: "Drop here" });
    }

    // ── Cards ──────────────────────────────────────────────────────────────
    colTasks.forEach((task, idx) => {
      const card = cardList.createEl("div", { cls: "kb-card" });
      card.draggable = true;
      
      // ── Folder color: add a left border using root folder name──────
      const rootFolder = getRootFolder(task.path);
      const folderColor = FOLDER_COLORS[rootFolder];
      if (folderColor) {
	      card.style.borderLeft = `4px solid ${folderColor}`;
	      }

      card.addEventListener("dragstart", (e) => {
        DRAG.path = task.path;
        e.dataTransfer.effectAllowed = "move";
        e.dataTransfer.setData("text/plain", task.path);
        requestAnimationFrame(() => card.classList.add("kb-dragging"));
      });
      card.addEventListener("dragend", () => {
        card.classList.remove("kb-dragging");
        clearDropUI();
        DRAG.path = null;
      });

      // ── Top row: progress badge + move buttons ─────────────────────────
      const top = card.createEl("div", { cls: "kb-top" });

      if (task.progress !== null && typeof task.progress === "number") {
        const progressContainer = top.createEl("div", { cls: "kb-progress" });
        const filledCount = Math.min(5, Math.max(0, task.progress)); // clap 0-5
        const filledChar = "◉";   // solid circle inside empty
			  const emptyChar = "◎";    // empty circle
			  const filledSpan = progressContainer.createEl("span");
			  filledSpan.textContent = filledChar.repeat(filledCount);
			  const emptySpan = progressContainer.createEl("span");
			  emptySpan.textContent = emptyChar.repeat(5 - filledCount);
			  const progressColor = {
				  0: "var(--text-faint)",
				  1: "#bf4097",
				  2: "#5540bf",
				  3: "#30bfbd",
				  4: "#a4bf40",
				  5: "#bf4a40"
			  }[task.progress] || "var(--background-modifier-border)"
			  filledSpan.style.color = progressColor;
			  emptySpan.style.color = "var(--text-faint)";
	        }
      // ── Move buttons (always present, disabled at boundaries) ──────────
      const moveWrap = top.createEl("div", { cls: "kb-move" });

      const btnUp = moveWrap.createEl("button", {
        cls: "kb-move-btn",
        text: "↑",
      });
      btnUp.title = "Move up";
      if (idx === 0) btnUp.disabled = true;
      btnUp.addEventListener("click", (e) => {
        e.stopPropagation();
        commitMove(task.path, "up");
      });

      const btnDown = moveWrap.createEl("button", {
        cls: "kb-move-btn",
        text: "↓",
      });
      btnDown.title = "Move down";
      if (idx === colTasks.length - 1) btnDown.disabled = true;
      btnDown.addEventListener("click", (e) => {
        e.stopPropagation();
        commitMove(task.path, "down");
      });

      // ── Title link ────────────────────────────────────────────────────────
      const a = card.createEl("a", { cls: "kb-title" });
      a.textContent = task.title;
      a.href = "#";
      a.onclick = (e) => {
        e.preventDefault();
        app.workspace.openLinkText(task.path, dv.currentFilePath, true);
      };
      
      if (task.tags && task.tags.length > 0) {
	      const tagContainer = card.createEl("div",{ cls: "kb-tag-list" });
	      task.tags.forEach(tag => {
		      const tagSpan = tagContainer.createEl("span", { cls: "kb-tag", text: tag });
	      })
      }
    });
  });
}

render();
```
