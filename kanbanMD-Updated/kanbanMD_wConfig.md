```dataviewjs
// 1. Configuration – separate JSON file

const CONFIG_FILE = ".kanban/kanban-config.json";

// Default settings (used if config file doesn't exist)
const DEFAULT_CONFIG = {
  folder: "/",
  excludePaths: ["_templates/", "attachments/", "Clippings/", "Life/Daily Notes/"],
  excludeTags: ["#Homepage", "#foldernote"],
  defaultCardLimit: 10,
  columns: [
    { id: "to-do", label: "To-Do" },
    { id: "in-progress", label: "In Progress" },
    { id: "on-hold", label: "On Hold" },
    { id: "someday", label: "Someday" },
    { id: "done", label: "Done" }
  ],
  folderColors: {
    "Azmeen": "#003166",
    "Life": "#ff5757",
    "Corner": "#91b60c",
    "3Diculous": "#6c045e",
    "Fuel-Forge": "#Fb752d"
  },
  progressColors: {
    0: "var(--text-faint)",
    1: "#bf4097",
    2: "#5540bf",
    3: "#30bfbd",
    4: "#a4bf40",
    5: "#bf4a40"
  }
};

// Global mutable variables (will be set by loadConfig)
let FOLDER, EXCLUDE_PATHS, EXCLUDE_TAGS, COLUMNS, FOLDER_COLOR_MAP, PROGRESS_COLORS, DEFAULT_LIMIT;

// Load config (creates file and folder automatically if needed)
async function loadConfig() {
  try {
    const exists = await app.vault.adapter.exists(CONFIG_FILE);
    if (!exists) {
      await app.vault.create(CONFIG_FILE, JSON.stringify(DEFAULT_CONFIG, null, 2));
    }
    const content = await app.vault.adapter.read(CONFIG_FILE);
    const config = JSON.parse(content);
    FOLDER = config.folder;
    EXCLUDE_PATHS = config.excludePaths;
    EXCLUDE_TAGS = config.excludeTags;
    COLUMNS = config.columns;
    FOLDER_COLOR_MAP = config.folderColors;
    PROGRESS_COLORS = config.progressColors || DEFAULT_CONFIG.progressColors;
    DEFAULT_LIMIT = config.defaultCardLimit || DEFAULT_CONFIG.defaultCardLimit;
  } catch (e) {
    console.error("Using default config due to error:", e);
    FOLDER = DEFAULT_CONFIG.folder;
    EXCLUDE_PATHS = DEFAULT_CONFIG.excludePaths;
    EXCLUDE_TAGS = DEFAULT_CONFIG.excludeTags;
    COLUMNS = DEFAULT_CONFIG.columns;
    FOLDER_COLOR_MAP = DEFAULT_CONFIG.folderColors;
    PROGRESS_COLORS = DEFAULT_CONFIG.progressColors;
    DEFAULT_LIMIT = DEFAULT_CONFIG.defaultCardLimit;
  }
}

async function saveConfig(newConfig) {
  try {
    const exists = await app.vault.adapter.exists(CONFIG_FILE);
    if (!exists) {
      await app.vault.create(CONFIG_FILE, JSON.stringify(newConfig, null, 2));
    } else {
      await app.vault.adapter.write(CONFIG_FILE, JSON.stringify(newConfig, null, 2));
    }
  } catch (e) {
    console.error("Save config failed:", e);
    throw e;
  }
}

// ───────────────────────────────────────────────────────────────────────────────
// 2. Persistent state (drag overrides, collapsed columns, etc.)
// ───────────────────────────────────────────────────────────────────────────────
const STATE_KEY = "kb_" + dv.currentFilePath.replace(/[^\w]/g, "_");
if (!window[STATE_KEY]) {
  window[STATE_KEY] = { ov: {}, orderOv: {}, collapsed: {} };
}
const STATE = window[STATE_KEY];
const DRAG = { path: null };

// ───────────────────────────────────────────────────────────────────────────────
// 3. Helper: get root folder name from file path
// ───────────────────────────────────────────────────────────────────────────────
function getRootFolder(path) {
  const parts = path.split('/');
  return parts.length > 1 ? parts[0] : "";
}

// ───────────────────────────────────────────────────────────────────────────────
// 4. File I/O for task status/order (unchanged, uses frontmatter patch)
// ───────────────────────────────────────────────────────────────────────────────
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

// ───────────────────────────────────────────────────────────────────────────────
// 5. Task list and filtering
// ───────────────────────────────────────────────────────────────────────────────
let pages = [];

function refreshPages() {
  pages = dv.pages(`"${FOLDER}"`).array().filter(p => {
    const path = p.file.path;
    if (EXCLUDE_PATHS.some(ex => path.includes(ex))) return false;
    const noteTags = p.file.tags || [];
    if (EXCLUDE_TAGS.some(tag => noteTags.includes(tag))) return false;
    return true;
  });
  
  // Purge stale overrides
  for (const p of Object.keys(STATE.ov)) {
    const pg = pages.find(x => x.file.path === p);
    if (pg && pg.status === STATE.ov[p]) delete STATE.ov[p];
  }
  for (const p of Object.keys(STATE.orderOv)) {
    const pg = pages.find(x => x.file.path === p);
    if (pg && pg.order === STATE.orderOv[p]) delete STATE.orderOv[p];
  }
}

function buildTasks() {
  return pages.map((p, i) => ({
    path: p.file.path,
    title: p.title || p.file.name,
    status: STATE.ov[p.file.path] ?? p.status ?? "to-do",
    progress: typeof p.progress === "number" ? p.progress : null,
    order: STATE.orderOv[p.file.path] ?? (typeof p.order === "number" ? p.order : i),
    tags: p.file.tags || [],
  }));
}

function byOrder(arr) {
  return [...arr].sort((a, b) => a.order - b.order);
}

// ───────────────────────────────────────────────────────────────────────────────
// 6. Drag & drop / move commit functions (unchanged)
// ───────────────────────────────────────────────────────────────────────────────
function clearDropUI() {
  dv.container.querySelectorAll(".kb-col-over").forEach(el => el.classList.remove("kb-col-over"));
}

function commitColumnChange(srcPath, targetColId) {
  const tasks = buildTasks();
  const src = tasks.find(t => t.path === srcPath);
  if (!src || src.status === targetColId) return;
  const colTasks = byOrder(tasks.filter(t => t.status === targetColId));
  const newOrder = colTasks.length ? Math.max(...colTasks.map(t => t.order)) + 1 : 0;
  STATE.ov[srcPath] = targetColId;
  STATE.orderOv[srcPath] = newOrder;
  renderFullBoard();
  patchFrontmatter(srcPath, { status: targetColId, order: newOrder }).catch(console.error);
}

function commitMove(srcPath, dir) {
  const tasks = buildTasks();
  const src = tasks.find(t => t.path === srcPath);
  if (!src) return;
  const colTasks = byOrder(tasks.filter(t => t.status === src.status));
  const idx = colTasks.findIndex(t => t.path === srcPath);
  const newIdx = dir === "up" ? idx - 1 : idx + 1;
  if (newIdx < 0 || newIdx >= colTasks.length) return;
  [colTasks[idx], colTasks[newIdx]] = [colTasks[newIdx], colTasks[idx]];
  colTasks.forEach((t, i) => { STATE.orderOv[t.path] = i; });
  renderFullBoard();
  colTasks.forEach(t => patchFrontmatter(t.path, { order: STATE.orderOv[t.path] }).catch(console.error));
}

// ───────────────────────────────────────────────────────────────────────────────
// 7. Rendering functions
// ───────────────────────────────────────────────────────────────────────────────
function renderBoard() {
  const root = dv.container;
  const tasks = buildTasks();
  const totalBefore = dv.pages(`"${FOLDER}"`).length;
  const statusBar = root.createEl("div", { cls: "kb-status" });
  statusBar.createEl("span", { 
    text: `${pages.length} notes loaded (${totalBefore - pages.length} excluded)` 
  });

  const board = root.createEl("div", { cls: "kb-board" });

  COLUMNS.forEach(col => {
    const colTasks = byOrder(tasks.filter(t => t.status === col.id));
    if (!STATE.cardsToShow) STATE.cardsToShow = {};
    const limit = STATE.cardsToShow[col.id] || DEFAULT_LIMIT;
    const visibleTasks = colTasks.slice(0, limit);
    const hasMore = colTasks.length > limit;
    const colEl = board.createEl("div", { cls: "kb-col" });

    const hdr = colEl.createEl("div", { cls: "kb-col-hdr" });
    const hdrLeft = hdr.createEl("div", { cls: "kb-col-hdr-left" });
    const foldIcon = hdrLeft.createEl("span", { cls: "kb-fold-icon", text: "▼" });
    hdr.createEl("span", { text: col.label });
    hdr.createEl("span", { cls: "kb-count", text: String(colTasks.length) });

    const cardList = colEl.createEl("div", { cls: "kb-cards" });

    if (!STATE.collapsed) STATE.collapsed = {};
    if (STATE.collapsed[col.id] === undefined) {
	    STATE.collapsed[col.id] = false;
    }
    const isCollapsed = STATE.collapsed[col.id];
    if (isCollapsed) foldIcon.textContent = "▶";
    cardList.style.display = isCollapsed ? "none" : "flex";

    hdr.style.cursor = "pointer";
    hdr.addEventListener("click", e => {
      e.stopPropagation();
      const newState = !STATE.collapsed[col.id];
      STATE.collapsed[col.id] = newState;
      foldIcon.textContent = newState ? "▶" : "▼";
      cardList.style.display = newState ? "none" : "flex";
    });

    // Drop events
    colEl.addEventListener("dragover", e => {
      if (!DRAG.path) return;
      e.preventDefault();
      e.dataTransfer.dropEffect = "move";
      colEl.classList.add("kb-col-over");
    });
    colEl.addEventListener("dragleave", e => {
      if (!colEl.contains(e.relatedTarget)) clearDropUI();
    });
    colEl.addEventListener("drop", e => {
      e.preventDefault();
      clearDropUI();
      const srcPath = DRAG.path;
      DRAG.path = null;
      if (srcPath) commitColumnChange(srcPath, col.id);
    });

    if (visibleTasks.length === 0) {
      cardList.createEl("div", { cls: "kb-empty", text: "Drop here" });
    }

    // Cards
    visibleTasks.forEach((task, idx) => {
      const card = cardList.createEl("div", { cls: "kb-card" });
      card.draggable = true;

      const rootFolder = getRootFolder(task.path);
      const borderColor = FOLDER_COLOR_MAP[rootFolder] || 'var(--interactive-accent)';
      card.style.borderLeft = `4px solid ${borderColor}`;

      card.addEventListener("dragstart", e => {
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

      const top = card.createEl("div", { cls: "kb-top" });

      if (task.progress !== null && typeof task.progress === "number") {
        const progressContainer = top.createEl("div", { cls: "kb-progress" });
        const filledCount = Math.min(5, Math.max(0, task.progress));
        const filledSpan = progressContainer.createEl("span");
        filledSpan.textContent = "◉".repeat(filledCount);
        const emptySpan = progressContainer.createEl("span");
        emptySpan.textContent = "◎".repeat(5 - filledCount);
        const progressColor = PROGRESS_COLORS[task.progress] || "var(--background-modifier-border)";
        filledSpan.style.color = progressColor;
        emptySpan.style.color = "var(--text-faint)";
      }

      const moveWrap = top.createEl("div", { cls: "kb-move" });
      const btnUp = moveWrap.createEl("button", { cls: "kb-move-btn", text: "↑" });
      btnUp.title = "Move up";
      if (idx === 0) btnUp.disabled = true;
      btnUp.addEventListener("click", e => {
        e.stopPropagation();
        commitMove(task.path, "up");
      });

      const btnDown = moveWrap.createEl("button", { cls: "kb-move-btn", text: "↓" });
      btnDown.title = "Move down";
      if (idx === colTasks.length - 1) btnDown.disabled = true;
      btnDown.addEventListener("click", e => {
        e.stopPropagation();
        commitMove(task.path, "down");
      });

      const a = card.createEl("a", { cls: "kb-title" });
      a.textContent = task.title;
      a.href = "#";
      a.onclick = e => {
        e.preventDefault();
        app.workspace.openLinkText(task.path, dv.currentFilePath, true);
      };

      if (task.tags && task.tags.length) {
        const tagContainer = card.createEl("div", { cls: "kb-tag-list" });
        task.tags.forEach(tag => {
          tagContainer.createEl("span", { cls: "kb-tag", text: tag });
        });
      }
    });

    if (hasMore) {
			  const loadMoreBtn = cardList.createEl("button", {
			    cls: "kb-load-more-btn",
			    text: `Load ${Math.min(DEFAULT_LIMIT, colTasks.length - limit)} more...`
			  });
			  loadMoreBtn.addEventListener("click", (e) => {
			    e.stopPropagation();
			    STATE.cardsToShow[col.id] = (STATE.cardsToShow[col.id] || DEFAULT_LIMIT) + DEFAULT_LIMIT;
			    renderFullBoard(); // re-render with new limit
			  });
			}
  });
}

function renderConfigPanel() {
	if (!FOLDER_COLOR_MAP) FOLDER_COLOR_MAP = {};
	if (!PROGRESS_COLORS) PROGRESS_COLORS = {};
  
  const root = dv.container;
  const configCol = root.createEl("div", { cls: "kb-col kb-config-col" });

  const header = configCol.createEl("div", { cls: "kb-col-hdr kb-config-hdr" });
  const headerLeft = header.createEl("div", { cls: "kb-col-hdr-left" });
  const foldIcon = headerLeft.createEl("span", { cls: "kb-fold-icon", text: "▼" });
  headerLeft.createEl("span", { text: " Board Settings", cls: "kb-config-title" });

  const content = configCol.createEl("div", { cls: "kb-cards kb-config-content" });
  const form = content.createEl("div", { cls: "kb-config-form" });

  if (!STATE.collapsed) STATE.collapsed = {};
  const panelKey = "__kb_config_panel__";
  if (STATE.collapsed[panelKey] === undefined) {
	  STATE.collapsed[panelKey] = true;
  }
  const isCollapsed = STATE.collapsed[panelKey];
  if (isCollapsed) {
    foldIcon.textContent = "▶";
    content.style.display = "none";
  }

  header.style.cursor = "pointer";
  header.addEventListener("click", e => {
    e.stopPropagation();
    const newState = !STATE.collapsed[panelKey];
    STATE.collapsed[panelKey] = newState;
    foldIcon.textContent = newState ? "▶" : "▼";
    content.style.display = newState ? "none" : "flex";
  });

  // Form fields (populated with current config)
  form.createEl("label", { text: "Main Folder" });
  const folderInput = form.createEl("input", { type: "text", placeholder: "/", cls: "kb-config-input" });
  folderInput.value = FOLDER;

  form.createEl("label", { text: "Exclude Paths (comma separated)" });
  const excludePathsInput = form.createEl("textarea", { placeholder: "_templates/, attachments/, ...", cls: "kb-config-textarea" });
  excludePathsInput.value = EXCLUDE_PATHS.join(", ");

  form.createEl("label", { text: "Exclude Tags (comma separated)" });
  const excludeTagsInput = form.createEl("textarea", { placeholder: "#Homepage, #foldernote", cls: "kb-config-textarea" });
  excludeTagsInput.value = EXCLUDE_TAGS.join(", ");
  
  form.createEl("label", { text: "Default Cards per Column" });
  const limitInput = form.createEl("input", {
  type: "number",
  min: 1,
  max: 100,
  value: String(DEFAULT_LIMIT),
  cls: "kb-config-input"
  });
  
  // ── Columns Editor ───────────────────────────────────────────────────────── 
  
  const columnsHeader = form.createEl("div", { cls: "kb-config-section-header" });
	const columnsHeaderLeft = columnsHeader.createEl("div", { cls: "kb-col-hdr-left" });
	const columnsFoldIcon = columnsHeaderLeft.createEl("span", { cls: "kb-fold-icon", text: "▶" });
	columnsHeaderLeft.createEl("span", { text: " Columns", cls: "kb-config-section-title" });
	
	const columnsContent = form.createEl("div", { cls: "kb-config-section-content" });
	columnsContent.style.display = "none"; // collapsed by default
	
	const columnsContainer = columnsContent.createEl("div", { cls: "kb-config-list" });
	
	// State key
	const columnsSectionKey = "__kb_config_columns__";
	if (STATE.collapsed[columnsSectionKey] === undefined) {
	  STATE.collapsed[columnsSectionKey] = true; // default collapsed
	}
	const columnsCollapsed = STATE.collapsed[columnsSectionKey];
	if (!columnsCollapsed) {
	  columnsFoldIcon.textContent = "▼";
	  columnsContent.style.display = "block";
	}
	
	columnsHeader.style.cursor = "pointer";
	columnsHeader.addEventListener("click", (e) => {
	  e.stopPropagation();
	  const newState = !STATE.collapsed[columnsSectionKey];
	  STATE.collapsed[columnsSectionKey] = newState;
	  columnsFoldIcon.textContent = newState ? "▶" : "▼";
	  columnsContent.style.display = newState ? "none" : "block";
	});

function renderColumnsEditor() {
  columnsContainer.empty();
  COLUMNS.forEach((col, index) => {
    const row = columnsContainer.createEl("div", { cls: "kb-config-row" });
    
    const idInput = row.createEl("input", {
      type: "text",
      value: col.id,
      placeholder: "id",
      cls: "kb-config-input-small"
    });
    const labelInput = row.createEl("input", {
      type: "text",
      value: col.label,
      placeholder: "Label",
      cls: "kb-config-input-small"
    });
    
    const removeBtn = row.createEl("button", {
      text: "✕",
      cls: "kb-config-remove-btn"
    });
    removeBtn.addEventListener("click", () => {
      COLUMNS.splice(index, 1);
      renderColumnsEditor();
    });
    
    // Update the column object on input change
    idInput.addEventListener("input", () => { col.id = idInput.value; });
    labelInput.addEventListener("input", () => { col.label = labelInput.value; });
  });
  
  const addBtn = columnsContainer.createEl("button", {
    text: "+ Add Column",
    cls: "kb-config-add-btn"
  });
  addBtn.addEventListener("click", () => {
    COLUMNS.push({ id: "new", label: "New Column" });
    renderColumnsEditor();
  });
}

// Initial render
renderColumnsEditor();

// ── Folder Colors Editor ───────────────────────────────────────────────────
const colorsHeader = form.createEl("div", { cls: "kb-config-section-header" });
const colorsHeaderLeft = colorsHeader.createEl("div", { cls: "kb-col-hdr-left" });
const colorsFoldIcon = colorsHeaderLeft.createEl("span", { cls: "kb-fold-icon", text: "▶" });
colorsHeaderLeft.createEl("span", { text: " Folder Colors", cls: "kb-config-section-title" });

const colorsContent = form.createEl("div", { cls: "kb-config-section-content" });
colorsContent.style.display = "none";
const colorsContainer = colorsContent.createEl("div", { cls: "kb-config-list" });
const colorsSectionKey = "__kb_config_colors__";
if (STATE.collapsed[colorsSectionKey] === undefined) {
  STATE.collapsed[colorsSectionKey] = true;
}
const colorsCollapsed = STATE.collapsed[colorsSectionKey];
if (!colorsCollapsed) {
  colorsFoldIcon.textContent = "▼";
  colorsContent.style.display = "block";
}

colorsHeader.style.cursor = "pointer";
colorsHeader.addEventListener("click", (e) => {
  e.stopPropagation();
  const newState = !STATE.collapsed[colorsSectionKey];
  STATE.collapsed[colorsSectionKey] = newState;
  colorsFoldIcon.textContent = newState ? "▶" : "▼";
  colorsContent.style.display = newState ? "none" : "block";
});

// Convert FOLDER_COLOR_MAP (object) to array for editing
let folderColorsArray = Object.entries(FOLDER_COLOR_MAP).map(([folder, color]) => ({ folder, color }));

function renderColorsEditor() {
  colorsContainer.empty();
  folderColorsArray.forEach((item, index) => {
    const row = colorsContainer.createEl("div", { cls: "kb-config-row" });
    
    const folderInput = row.createEl("input", {
      type: "text",
      value: item.folder,
      placeholder: "Folder",
      cls: "kb-config-input-small"
    });
    
    const colorPicker = row.createEl("input", {
      type: "color",
      value: item.color,
      cls: "kb-config-color-picker"
    });
    
    const colorText = row.createEl("input", {
      type: "text",
      value: item.color,
      placeholder: "#RRGGBB",
      cls: "kb-config-input-small"
    });
    
    // Sync color picker and text field
    colorPicker.addEventListener("input", () => {
      colorText.value = colorPicker.value;
      item.color = colorPicker.value;
    });
    colorText.addEventListener("input", () => {
      colorPicker.value = colorText.value;
      item.color = colorText.value;
    });
    folderInput.addEventListener("input", () => { item.folder = folderInput.value; });
    
    const removeBtn = row.createEl("button", {
      text: "✕",
      cls: "kb-config-remove-btn"
    });
    removeBtn.addEventListener("click", () => {
      folderColorsArray.splice(index, 1);
      renderColorsEditor();
    });
  });
  
  const addBtn = colorsContainer.createEl("button", {
    text: "+ Add Folder Color",
    cls: "kb-config-add-btn"
  });
  addBtn.addEventListener("click", () => {
    folderColorsArray.push({ folder: "NewFolder", color: "#888888" });
    renderColorsEditor();
  });
}

renderColorsEditor();

// ── Progress Colors Editor ───────────────────────────────────────────────────

const progressHeader = form.createEl("div", { cls: "kb-config-section-header" });
const progressHeaderLeft = progressHeader.createEl("div", { cls: "kb-col-hdr-left" });
const progressFoldIcon = progressHeaderLeft.createEl("span", { cls: "kb-fold-icon", text: "▶" })
progressHeaderLeft.createEl("span", { text: " Progress Colors", cls: "kb-config-section-title" });

const progressContent = form.createEl("div", { cls: "kb-config-section-content" });
progressContent.style.display = "none";
const progressContainer = progressContent.createEl("div", { cls: "kb-config-list" });

const progressSectionKey = "__kb_config_colors__";
if (STATE.collapsed[progressSectionKey] === undefined) {
  STATE.collapsed[progressSectionKey] = true;
}
const progressCollapsed = STATE.collapsed[progressSectionKey];
if (!progressCollapsed) {
  progressFoldIcon.textContent = "▼";
  progressContent.style.display = "block";
}

progressHeader.style.cursor = "pointer";
progressHeader.addEventListener("click", (e) => {
  e.stopPropagation();
  const newState = !STATE.collapsed[progressSectionKey];
  STATE.collapsed[progressSectionKey] = newState;
  progressFoldIcon.textContent = newState ? "▶" : "▼";
  progressContent.style.display = newState ? "none" : "block";
});

// Convert PROGRESS_COLORS object to array of entries [level, color]
let progressArray = Object.entries(PROGRESS_COLORS).map(([level, color]) => ({ level: parseInt(level), color }));

function renderProgressEditor() {
  progressContainer.empty();
  progressArray
  .filter(item => item.level !== 0)
  .sort((a,b) => a.level - b.level)
  .forEach((item) => {
    const row = progressContainer.createEl("div", { cls: "kb-config-row" });
    
    row.createEl("span", { text: `Level ${item.level}:`, cls: "kb-config-label-small" });
    
    const colorPicker = row.createEl("input", {
      type: "color",
      value: item.color,
      cls: "kb-config-color-picker"
    });
    const colorText = row.createEl("input", {
      type: "text",
      value: item.color,
      placeholder: "#RRGGBB",
      cls: "kb-config-input-small"
    });
    
    colorPicker.addEventListener("input", () => {
      colorText.value = colorPicker.value;
      item.color = colorPicker.value;
    });
    colorText.addEventListener("input", () => {
      colorPicker.value = colorText.value;
      item.color = colorText.value;
    });
  });
}
renderProgressEditor();

form.createEl("label", { text: "Config File Location" });
const locationText = form.createEl("div", { 
  cls: "kb-config-info-text",
  text: CONFIG_FILE 
}); 

  const buttonRow = form.createEl("div", { cls: "kb-config-buttons" });
  const saveBtn = buttonRow.createEl("button", { text: "Save & Refresh", cls: "kb-config-save-btn" });
  const cancelBtn = buttonRow.createEl("button", { text: "Cancel", cls: "kb-config-cancel-btn" });

  saveBtn.addEventListener("click", async () => {
  try {
    const newFolder = folderInput.value.trim() || "/";
    const newExcludePaths = excludePathsInput.value.split(",").map(s => s.trim()).filter(s => s);
    const newExcludeTags = excludeTagsInput.value.split(",").map(s => s.trim()).filter(s => s);
    
    const newDefaultLimit = parseInt(limitInput.value) || 10;
    
    // Columns are already updated live in the COLUMNS array
    const newColumns = COLUMNS;
    
    // Convert folderColorsArray back to object
    const newFolderColors = {};
    folderColorsArray.forEach(item => {
      if (item.folder.trim()) newFolderColors[item.folder.trim()] = item.color;
    });
    
    // Convert progressArray back to object
		const newProgressColors = { ...PROGRESS_COLORS };
		progressArray.forEach(item => { newProgressColors[item.level] = item.color; });

    const newConfig = {
      folder: newFolder,
      excludePaths: newExcludePaths,
      excludeTags: newExcludeTags,
      defaultCardLimit: newDefaultLimit,
      columns: newColumns,
      folderColors: newFolderColors,
      progressColors: newProgressColors
    };
    

    await saveConfig(newConfig);

    // Update globals
    FOLDER = newFolder;
    EXCLUDE_PATHS = newExcludePaths;
    EXCLUDE_TAGS = newExcludeTags;
    COLUMNS = newColumns;
    FOLDER_COLOR_MAP = newFolderColors;
    PROGRESS_COLORS = newProgressColors;
    DEFAULT_LIMIT = newDefaultLimit;
    
    STATE.cardsToShow = {};   // clear all saved limits

    // Collapse panel
    content.style.display = "none";
    STATE.collapsed[panelKey] = true;
    foldIcon.textContent = "▶";

    // Refresh board
    refreshPages();
    renderFullBoard();
    setTimeout(scrollToBoardTop, 20);

    new Notice("Settings saved & board updated.");
  } catch (e) {
    console.error(e);
    new Notice("Error: " + e.message);
  }
});

  cancelBtn.addEventListener("click", () => {
    content.style.display = "none";
    STATE.collapsed[panelKey] = true;
    foldIcon.textContent = "▶";
  });
}

function scrollToBoardTop() {
  const boardEl = dv.container.closest('.markdown-preview-view') || dv.container;
  boardEl.scrollTop = 0;
  // Alternative: scroll the container itself
  dv.container.scrollIntoView({ behavior: 'auto', block: 'start' });
}

function renderFullBoard() {
  const root = dv.container;
  root.empty();
  renderConfigPanel();
  renderBoard();
  setTimeout(scrollToBoardTop, 20);
}

// ───────────────────────────────────────────────────────────────────────────────
// 8. Initialisation
// ───────────────────────────────────────────────────────────────────────────────
(async () => {
  await loadConfig();
  refreshPages();
  renderFullBoard();
})();