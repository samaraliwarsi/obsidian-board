```dataviewjs
// ── Config ────────────────────────────────────────────────────────────────────
const FOLDER = "UPDATE_THIS_TO_YOUR_TASKS_FOLDER_RELATIVE_PATH";
// ── Change the following columns as you wish
const COLUMNS = [
  { id: "someday", label: "Someday" },
  { id: "to-do", label: "To Do" },
  { id: "in-progress", label: "In Progress" },
  { id: "on-hold", label: "On Hold" },
  { id: "done", label: "Done" },
];
// ── Change the following priorities as you wish with their respecting colors
const P_LABEL = ["", "Lowest", "Low", "Medium", "High", "Highest"];
const P_COLOR = ["", "#89BDF4", "#89BDF4", "#FFE88B", "#FF7881", "#FF7881"];

// ── Things might break if you change after this line
// ── Persistent state ───────────────────────────────────────────────────────────
const STATE_KEY = "kb_" + dv.currentFilePath.replace(/[^\w]/g, "_");
if (!window[STATE_KEY]) {
  window[STATE_KEY] = { ov: {}, orderOv: {} };
}
const STATE = window[STATE_KEY];

// ── Ephemeral drag state ───────────────────────────────────────────────────────
const DRAG = { path: null };

// ── Load pages ─────────────────────────────────────────────────────────────────
const pages = dv.pages(`"${FOLDER}"`).array();

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
    priority: typeof p.priority === "number" ? p.priority : null,
    order:
      STATE.orderOv[p.file.path] ?? (typeof p.order === "number" ? p.order : i),
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
    hdr.createEl("span", { text: col.label });
    hdr.createEl("span", { cls: "kb-count", text: String(colTasks.length) });

    const cardList = colEl.createEl("div", { cls: "kb-cards" });

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

      // ── Top row: priority badge + move buttons ─────────────────────────
      const top = card.createEl("div", { cls: "kb-top" });

      if (task.priority !== null) {
        const badge = top.createEl("span", { cls: "kb-badge" });
        badge.textContent = P_LABEL[task.priority] ?? `P${task.priority}`;
        badge.style.background = P_COLOR[task.priority] ?? "#707BC2";
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

      // ── Title ──────────────────────────────────────────────────────────
      const a = card.createEl("a", { cls: "kb-title" });
      a.textContent = task.title;
      a.href = "#";
      a.onclick = (e) => {
        e.preventDefault();
        app.workspace.openLinkText(task.path, dv.currentFilePath, true);
      };
    });
  });
}

render();
```