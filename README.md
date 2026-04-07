(function () {
  "use strict";

  const STORAGE_KEY = "todo-dashboard-tasks-v3";
  const STORAGE_KEY_LEGACY = "todo-dashboard-tasks-v2";
  const SIDEBAR_KEY = "todo-sidebar-minimized";

  const DEFAULT_LISTS = [
    { id: "personal", name: "Personal", dotClass: "dot-red" },
    { id: "work", name: "Work", dotClass: "dot-blue" },
    { id: "list1", name: "List 1", dotClass: "dot-yellow" },
  ];

  const LIST_COLORS = [
    { dotClass: "dot-red", label: "Red" },
    { dotClass: "dot-blue", label: "Blue" },
    { dotClass: "dot-yellow", label: "Yellow" },
    { dotClass: "dot-green", label: "Green" },
    { dotClass: "dot-purple", label: "Purple" },
    { dotClass: "dot-gray", label: "Gray" },
  ];

  /** @type {{ id: string, name: string, dotClass: string }[]} */
  let lists = [];
  /** @type {string[]} */
  let globalTags = [];
  /** @type {{ mode: "view" | "list" | "tag", viewId: string, listId: string | null, tag: string | null }} */
  let nav = { mode: "view", viewId: "today", listId: null, tag: null };

  /** @type {{ id: string, title: string, description: string, listId: string, due: string, tags: string[], completed: boolean, subtasks: { id: string, text: string, done: boolean }[] }[]} */
  let tasks = [];
  let selectedId = null;
  let searchDebounceTimer = null;
  let toastTimer = null;
  let deleteModalPrevFocus = null;

  const els = {
    taskList: null,
    navBody: null,
    mainTitleText: null,
    titleCountEl: null,
    listModal: null,
    listModalName: null,
    listModalCancel: null,
    listModalSave: null,
    listColorOptions: null,
    searchInput: null,
    addTaskBtn: null,
    taskName: null,
    taskDesc: null,
    taskListSelect: null,
    taskDue: null,
    detailTagsPills: null,
    tagInput: null,
    subtaskInput: null,
    subtaskList: null,
    btnDelete: null,
    btnSave: null,
    detailPanel: null,
    detailHint: null,
    unsavedHint: null,
    deleteModal: null,
    deleteModalBody: null,
    deleteModalCancel: null,
    deleteModalConfirm: null,
    toastRegion: null,
    dashboard: null,
    menuToggle: null,
  };

  let sidebarMinimized = false;
  let mqMobile = null;

  function isMobileLayout() {
    return mqMobile ? mqMobile.matches : window.innerWidth <= 960;
  }

  function loadSidebarPreference() {
    try {
      const v = localStorage.getItem(SIDEBAR_KEY);
      sidebarMinimized = v === "1";
    } catch (_) {
      sidebarMinimized = false;
    }
  }

  function saveSidebarPreference() {
    try {
      localStorage.setItem(SIDEBAR_KEY, sidebarMinimized ? "1" : "0");
    } catch (_) {}
  }

  function applySidebarUI() {
    if (!els.dashboard || !els.menuToggle) return;
    els.dashboard.classList.toggle("sidebar-minimized", sidebarMinimized);

    const mobile = isMobileLayout();

    if (mobile) {
      els.menuToggle.setAttribute("aria-expanded", sidebarMinimized ? "false" : "true");
      els.menuToggle.removeAttribute("aria-pressed");
      els.menuToggle.setAttribute(
        "aria-label",
        sidebarMinimized ? "Open menu" : "Close menu"
      );
    } else {
      els.menuToggle.removeAttribute("aria-expanded");
      els.menuToggle.setAttribute("aria-pressed", sidebarMinimized ? "true" : "false");
      els.menuToggle.setAttribute(
        "aria-label",
        sidebarMinimized ? "Expand sidebar" : "Collapse sidebar"
      );
    }
  }

  function toggleSidebar() {
    sidebarMinimized = !sidebarMinimized;
    saveSidebarPreference();
    applySidebarUI();
  }

  function onSidebarMqChange() {
    applySidebarUI();
  }

  function uid(prefix) {
    return prefix + "-" + Math.random().toString(36).slice(2, 11);
  }

  function listIdSet() {
    return new Set(lists.map((l) => l.id));
  }

  function normalizeList(raw) {
    const allowed = new Set(LIST_COLORS.map((c) => c.dotClass));
    const dotClass = allowed.has(raw?.dotClass) ? raw.dotClass : "dot-gray";
    return {
      id: typeof raw?.id === "string" ? raw.id : uid("l"),
      name: String(raw?.name ?? "List").trim() || "List",
      dotClass,
    };
  }

  function cloneDefaultLists() {
    lists = DEFAULT_LISTS.map((l) => ({ ...l }));
  }

  function defaultNav() {
    return { mode: "view", viewId: "today", listId: null, tag: null };
  }

  function allTagsForSidebar() {
    const s = new Set(globalTags);
    tasks.forEach((t) => (t.tags || []).forEach((tag) => s.add(tag)));
    return Array.from(s).sort((a, b) => a.localeCompare(b));
  }

  function ensureNavValid() {
    if (nav.mode === "list" && (!nav.listId || !lists.some((l) => l.id === nav.listId))) {
      nav = defaultNav();
    }
    if (nav.mode === "tag" && (!nav.tag || !allTagsForSidebar().includes(nav.tag))) {
      nav = defaultNav();
    }
    if (nav.mode === "view" && !["upcoming", "today", "calendar", "sticky"].includes(nav.viewId)) {
      nav.viewId = "today";
    }
  }

  function normalizeTask(raw) {
    const listIds = listIdSet();
    const fallbackListId = lists[0]?.id || "personal";
    const subtasks = Array.isArray(raw.subtasks)
      ? raw.subtasks.map((s) => ({
          id: typeof s?.id === "string" ? s.id : uid("s"),
          text: String(s?.text ?? "").trim() || "Subtask",
          done: !!s?.done,
        }))
      : [];
    return {
      id: typeof raw?.id === "string" ? raw.id : uid("t"),
      title: String(raw?.title ?? "").trim() || "Untitled task",
      description: String(raw?.description ?? ""),
      listId: listIds.has(raw?.listId) ? raw.listId : fallbackListId,
      due: String(raw?.due ?? ""),
      tags: Array.isArray(raw?.tags) ? raw.tags.map((t) => String(t).trim()).filter(Boolean) : [],
      completed: !!raw?.completed,
      subtasks,
    };
  }

  function migrateFromV1() {
    try {
      const raw = localStorage.getItem("todo-dashboard-tasks-v1");
      if (!raw) return false;
      const parsed = JSON.parse(raw);
      if (!Array.isArray(parsed.tasks)) return false;
      cloneDefaultLists();
      globalTags = ["Tag 1", "Tag 2"];
      nav = defaultNav();
      tasks = parsed.tasks.map((t) => normalizeTask(t || {}));
      selectedId =
        parsed.selectedId && tasks.some((t) => t.id === parsed.selectedId)
          ? parsed.selectedId
          : tasks[0]?.id ?? null;
      saveApp();
      localStorage.removeItem("todo-dashboard-tasks-v1");
      return true;
    } catch (_) {
      return false;
    }
  }

  function migrateFromV2() {
    try {
      const raw = localStorage.getItem(STORAGE_KEY_LEGACY);
      if (!raw) return false;
      const parsed = JSON.parse(raw);
      if (!Array.isArray(parsed.tasks)) return false;
      cloneDefaultLists();
      globalTags = ["Tag 1", "Tag 2"];
      nav = defaultNav();
      tasks = parsed.tasks.map((t) => normalizeTask(t || {}));
      selectedId =
        parsed.selectedId && tasks.some((t) => t.id === parsed.selectedId)
          ? parsed.selectedId
          : null;
      saveApp();
      localStorage.removeItem(STORAGE_KEY_LEGACY);
      return true;
    } catch (_) {
      return false;
    }
  }

  function loadApp() {
    if (migrateFromV1()) {
      ensureNavValid();
      return;
    }
    if (migrateFromV2()) {
      ensureNavValid();
      return;
    }

    try {
      const raw = localStorage.getItem(STORAGE_KEY);
      if (raw) {
        const parsed = JSON.parse(raw);
        if (Array.isArray(parsed.tasks)) {
          lists =
            Array.isArray(parsed.lists) && parsed.lists.length
              ? parsed.lists.map((l) => normalizeList(l || {}))
              : [];
          if (!lists.length) cloneDefaultLists();

          globalTags =
            Array.isArray(parsed.globalTags) && parsed.globalTags.length
              ? parsed.globalTags.map((t) => String(t).trim()).filter(Boolean)
              : ["Tag 1", "Tag 2"];

          nav =
            parsed.nav && typeof parsed.nav === "object"
              ? {
                  mode: ["view", "list", "tag"].includes(parsed.nav.mode) ? parsed.nav.mode : "view",
                  viewId: String(parsed.nav.viewId || "today"),
                  listId: parsed.nav.listId == null ? null : String(parsed.nav.listId),
                  tag: parsed.nav.tag == null ? null : String(parsed.nav.tag),
                }
              : defaultNav();

          tasks = parsed.tasks.map((t) => normalizeTask(t || {}));
          selectedId =
            parsed.selectedId && tasks.some((t) => t.id === parsed.selectedId)
              ? parsed.selectedId
              : null;
          ensureNavValid();
          return;
        }
      }
    } catch (_) {}
    seedDefaultTasks();
  }

  function saveApp() {
    try {
      localStorage.setItem(
        STORAGE_KEY,
        JSON.stringify({
          version: 3,
          tasks,
          selectedId,
          lists,
          globalTags,
          nav,
        })
      );
    } catch (_) {
      showToast("Could not save (storage full or blocked).");
    }
  }

  function seedDefaultTasks() {
    cloneDefaultLists();
    globalTags = ["Tag 1", "Tag 2"];
    nav = defaultNav();
    tasks = [
      {
        id: uid("t"),
        title: "Buy groceries",
        description: "",
        listId: "personal",
        due: "",
        tags: [],
        completed: false,
        subtasks: [],
      },
      {
        id: uid("t"),
        title: "Renew driver's license",
        description: "",
        listId: "personal",
        due: "Tomorrow",
        tags: ["Tag 1"],
        completed: false,
        subtasks: [
          { id: uid("s"), text: "Gather required documents", done: false },
          { id: uid("s"), text: "Fill out online form", done: true },
        ],
      },
      {
        id: uid("t"),
        title: "Finish quarterly report",
        description: "",
        listId: "work",
        due: "",
        tags: [],
        completed: true,
        subtasks: [],
      },
      {
        id: uid("t"),
        title: "Book dentist appointment",
        description: "",
        listId: "personal",
        due: "",
        tags: [],
        completed: false,
        subtasks: [],
      },
      {
        id: uid("t"),
        title: "Water indoor plants",
        description: "",
        listId: "list1",
        due: "",
        tags: [],
        completed: false,
        subtasks: [],
      },
    ];
    selectedId = tasks[1] ? tasks[1].id : tasks[0]?.id ?? null;
  }

  function listById(id) {
    return lists.find((l) => l.id === id) || lists[0];
  }

  function matchesStickyTask(task) {
    return (
      (task.tags || []).some((x) => String(x).toLowerCase() === "sticky") ||
      /sticky/i.test(task.title || "")
    );
  }

  function matchesNavFilter(task) {
    if (nav.mode === "list") return task.listId === nav.listId;
    if (nav.mode === "tag") return (task.tags || []).includes(nav.tag);
    switch (nav.viewId) {
      case "upcoming":
        return !task.completed && !!String(task.due || "").trim();
      case "today":
        return !task.completed;
      case "calendar":
        return true;
      case "sticky":
        return matchesStickyTask(task);
      default:
        return !task.completed;
    }
  }

  function navSearchQuery() {
    return (els.searchInput?.value || "").trim().toLowerCase();
  }

  function sidebarLabelMatch(label) {
    const q = navSearchQuery();
    if (!q) return true;
    return String(label).toLowerCase().includes(q);
  }

  function getVisibleTasks() {
    let pool = tasks.filter(matchesNavFilter);
    const q = navSearchQuery();
    if (!q) return pool;
    return pool.filter((t) => {
      const inTitle = t.title.toLowerCase().includes(q);
      const inDesc = (t.description || "").toLowerCase().includes(q);
      const inTags = (t.tags || []).some((tag) => tag.toLowerCase().includes(q));
      return inTitle || inDesc || inTags;
    });
  }

  function countUpcomingNav() {
    return tasks.filter((t) => !t.completed && !!String(t.due || "").trim()).length;
  }

  function countTodayNav() {
    return tasks.filter((t) => !t.completed).length;
  }

  function ensureSelection() {
    const visible = getVisibleTasks();
    if (!visible.length) {
      selectedId = null;
      return;
    }
    if (!selectedId || !visible.some((t) => t.id === selectedId)) {
      selectedId = visible[0].id;
    }
  }

  function fillListSelect() {
    if (!els.taskListSelect) return;
    els.taskListSelect.innerHTML = lists.map(
      (l) => `<option value="${l.id}">${escapeHtml(l.name)}</option>`
    ).join("");
  }

  function escapeHtml(s) {
    const div = document.createElement("div");
    div.textContent = s;
    return div.innerHTML;
  }

  function escapeAttr(s) {
    return String(s)
      .replace(/&/g, "&amp;")
      .replace(/"/g, "&quot;")
      .replace(/</g, "&lt;");
  }

  function readFormIntoTask(task) {
    task.title = (els.taskName?.value || "").trim() || "Untitled task";
    task.description = els.taskDesc?.value || "";
    task.due = (els.taskDue?.value || "").trim();
    task.listId = els.taskListSelect?.value || lists[0]?.id || "personal";
  }

  function isDetailDirty() {
    const task = tasks.find((t) => t.id === selectedId);
    if (!task) return false;
    const title = (els.taskName?.value || "").trim() || "Untitled task";
    const desc = els.taskDesc?.value || "";
    const due = (els.taskDue?.value || "").trim();
    const listId = els.taskListSelect?.value || lists[0]?.id || "personal";
    return (
      task.title !== title ||
      (task.description || "") !== desc ||
      (task.due || "") !== due ||
      task.listId !== listId
    );
  }

  function updateUnsavedHint() {
    if (!els.unsavedHint) return;
    const empty = !tasks.find((t) => t.id === selectedId);
    const dirty = !empty && isDetailDirty();
    els.unsavedHint.hidden = !dirty;
    els.btnSave?.classList.toggle("btn-pulse", dirty);
  }

  function persistCurrentDetailIfNeeded() {
    const task = tasks.find((t) => t.id === selectedId);
    if (task) {
      readFormIntoTask(task);
      saveApp();
    }
  }

  let listModalPrevFocus = null;

  function updateMainTitle() {
    if (!els.mainTitleText) return;
    if (nav.mode === "list") {
      const L = lists.find((l) => l.id === nav.listId);
      els.mainTitleText.textContent = L ? L.name : "Tasks";
      return;
    }
    if (nav.mode === "tag") {
      els.mainTitleText.textContent = nav.tag ? nav.tag : "Tag";
      return;
    }
    const map = {
      upcoming: "Upcoming",
      today: "Today",
      calendar: "Calendar",
      sticky: "Sticky Wall",
    };
    els.mainTitleText.textContent = map[nav.viewId] || "Tasks";
  }

  function setNavView(viewId) {
    nav = { mode: "view", viewId, listId: null, tag: null };
    saveApp();
    renderTaskList();
    renderDetail();
  }

  function setNavList(listId) {
    nav = { mode: "list", viewId: "today", listId, tag: null };
    saveApp();
    renderTaskList();
    renderDetail();
  }

  function setNavTag(tag) {
    if (nav.mode === "tag" && nav.tag === tag) {
      nav = defaultNav();
    } else {
      nav = { mode: "tag", viewId: "today", listId: null, tag };
    }
    saveApp();
    renderTaskList();
    renderDetail();
  }

  function renderSidebar() {
    const body = els.navBody;
    if (!body) return;

    const viewDefs = [
      {
        id: "upcoming",
        label: "Upcoming",
        icon: "fa-regular fa-calendar",
        count: countUpcomingNav(),
        showCount: true,
      },
      {
        id: "today",
        label: "Today",
        icon: "fa-solid fa-list-check",
        count: countTodayNav(),
        showCount: true,
      },
      {
        id: "calendar",
        label: "Calendar",
        icon: "fa-regular fa-calendar-days",
        count: 0,
        showCount: false,
      },
      {
        id: "sticky",
        label: "Sticky Wall",
        icon: "fa-regular fa-note-sticky",
        count: 0,
        showCount: false,
      },
    ];

    const viewRows = viewDefs.filter(
      (v) => sidebarLabelMatch(v.label) || sidebarLabelMatch(v.id)
    );
    const listRows = lists.filter((l) => sidebarLabelMatch(l.name));
    const tagRows = allTagsForSidebar().filter((t) => sidebarLabelMatch(t));

    let html = "";

    html += `<p class="nav-label"${viewRows.length ? "" : " hidden"}>Tasks</p>`;
    html += `<ul class="nav-list"${viewRows.length ? "" : " hidden"}>`;
    for (const v of viewRows) {
      const active = nav.mode === "view" && nav.viewId === v.id;
      const countHtml = v.showCount ? `<span class="nav-count">${v.count}</span>` : "";
      html += `<li><a href="#" class="nav-link${active ? " active" : ""}" data-nav-view="${v.id}"${
        active ? ' aria-current="page"' : ""
      }><i class="${v.icon}" aria-hidden="true"></i> ${escapeHtml(v.label)}${countHtml}</a></li>`;
    }
    html += `</ul>`;

    html += `<p class="nav-label"${listRows.length ? "" : " hidden"}>Lists</p>`;
    html += `<ul class="nav-list"${listRows.length ? "" : " hidden"}>`;
    for (const l of listRows) {
      const active = nav.mode === "list" && nav.listId === l.id;
      html += `<li><a href="#" class="nav-link${active ? " active" : ""}" data-nav-list-id="${escapeAttr(
        l.id
      )}"${active ? ' aria-current="page"' : ""}><span class="dot ${l.dotClass}" aria-hidden="true"></span> ${escapeHtml(
        l.name
      )}</a></li>`;
    }
    html += `</ul>`;
    html += `<button type="button" class="nav-add" id="btn-nav-add-list"><i class="fa-solid fa-plus" aria-hidden="true"></i> Add New List</button>`;

    html += `<p class="nav-label"${tagRows.length ? "" : " hidden"}>Tags</p>`;
    html += `<div class="tag-row"${tagRows.length ? "" : " hidden"}>`;
    for (const t of tagRows) {
      const active = nav.mode === "tag" && nav.tag === t;
      html += `<button type="button" class="tag-pill${
        active ? " tag-pill-active" : ""
      }" data-nav-tag="${encodeURIComponent(t)}" aria-pressed="${active ? "true" : "false"}">${escapeHtml(
        t
      )}</button>`;
    }
    html += `</div>`;
    html += `<button type="button" class="nav-add nav-add-tag" id="btn-nav-add-sidebar-tag"><i class="fa-solid fa-plus" aria-hidden="true"></i> Add Tag</button>`;

    body.innerHTML = html;
  }

  function onNavBodyClick(e) {
    if (e.target.closest("#btn-nav-add-list")) {
      e.preventDefault();
      openListModal();
      return;
    }
    if (e.target.closest("#btn-nav-add-sidebar-tag")) {
      e.preventDefault();
      onSidebarAddTag();
      return;
    }
    const viewA = e.target.closest("[data-nav-view]");
    if (viewA) {
      e.preventDefault();
      setNavView(viewA.getAttribute("data-nav-view"));
      return;
    }
    const listA = e.target.closest("[data-nav-list-id]");
    if (listA) {
      e.preventDefault();
      const rawId = listA.getAttribute("data-nav-list-id");
      if (rawId) setNavList(rawId);
      return;
    }
    const tagB = e.target.closest("[data-nav-tag]");
    if (tagB) {
      e.preventDefault();
      const raw = tagB.getAttribute("data-nav-tag");
      const tag = raw ? decodeURIComponent(raw) : "";
      if (tag) setNavTag(tag);
      return;
    }
    const a = e.target.closest('a[href="#"]');
    if (a) e.preventDefault();
  }

  function buildListColorRadios() {
    if (!els.listColorOptions) return;
    els.listColorOptions.innerHTML = LIST_COLORS.map(
      (c, i) =>
        `<label class="list-color-swatch"><input type="radio" name="list-modal-color" value="${escapeAttr(
          c.dotClass
        )}" ${i === 0 ? "checked" : ""} /><span class="dot ${c.dotClass}" title="${escapeAttr(
          c.label
        )}" aria-hidden="true"></span><span class="visually-hidden">${escapeHtml(c.label)}</span></label>`
    ).join("");
  }

  function openListModal() {
    if (!els.listModal || !els.listModalName) return;
    els.listModalName.value = "";
    listModalPrevFocus = document.activeElement;
    els.listModal.hidden = false;
    els.listModalName.focus();
  }

  function closeListModal() {
    if (!els.listModal) return;
    els.listModal.hidden = true;
    if (listModalPrevFocus && typeof listModalPrevFocus.focus === "function") {
      listModalPrevFocus.focus();
    }
    listModalPrevFocus = null;
  }

  function confirmListModal() {
    const name = (els.listModalName?.value || "").trim();
    if (!name) {
      showToast("Enter a list name.");
      return;
    }
    const checked = els.listModal?.querySelector('input[name="list-modal-color"]:checked');
    const dotClass = checked?.value && LIST_COLORS.some((c) => c.dotClass === checked.value) ? checked.value : "dot-gray";
    lists.push({ id: uid("l"), name, dotClass });
    saveApp();
    closeListModal();
    fillListSelect();
    renderTaskList();
    renderDetail();
    showToast("List created.");
  }

  function onSidebarAddTag() {
    const name = prompt("New tag name");
    if (!name || !name.trim()) return;
    const tag = name.trim();
    if (!globalTags.includes(tag)) globalTags.push(tag);
    saveApp();
    renderTaskList();
    renderDetail();
    showToast("Tag added.");
  }

  function onListModalBackdropClick(e) {
    if (e.target === els.listModal) closeListModal();
  }

  function renderTaskList() {
    ensureSelection();
    const visible = getVisibleTasks();
    if (!els.taskList) return;

    els.taskList.innerHTML = visible
      .map((task) => {
        const selected = task.id === selectedId;
        const titleClass = task.completed ? "task-title task-done" : "task-title";
        const chevron = selected
          ? '<i class="fa-solid fa-chevron-down task-chevron" aria-hidden="true"></i>'
          : '<i class="fa-solid fa-chevron-right task-chevron" aria-hidden="true"></i>';
        const optId = `task-row-${task.id}`;
        const completeLabel = escapeAttr(`Mark "${task.title}" complete`);

        if (selected) {
          const list = listById(task.listId);
          const dueHtml = task.due
            ? `<span class="meta-line"><i class="fa-regular fa-calendar" aria-hidden="true"></i> ${escapeHtml(task.due)}</span>`
            : "";
          const subLabel =
            task.subtasks.length > 0 ? '<span class="meta-label">Subtasks</span>' : "";
          return `
            <li
              class="task-row task-row-expanded selected"
              data-task-id="${task.id}"
              id="${optId}"
              tabindex="0"
            >
              <div class="task-row-head">
                <label class="task-check">
                  <input type="checkbox" ${task.completed ? "checked" : ""} aria-label="${completeLabel}" />
                  <span class="task-check-ui" aria-hidden="true"></span>
                </label>
                <span class="${titleClass}">${escapeHtml(task.title)}</span>
                ${chevron}
              </div>
              <div class="task-meta">
                ${dueHtml}
                ${subLabel}
                <span class="meta-tag"><span class="dot ${list.dotClass}" aria-hidden="true"></span> ${escapeHtml(list.name)}</span>
              </div>
            </li>`;
        }

        return `
          <li
            class="task-row"
            data-task-id="${task.id}"
            id="${optId}"
            tabindex="0"
          >
            <label class="task-check">
              <input type="checkbox" ${task.completed ? "checked" : ""} aria-label="${completeLabel}" />
              <span class="task-check-ui" aria-hidden="true"></span>
            </label>
            <span class="${titleClass}">${escapeHtml(task.title)}</span>
            ${chevron}
          </li>`;
      })
      .join("");

    const openInView = visible.filter((t) => !t.completed).length;
    if (els.titleCountEl) els.titleCountEl.textContent = String(openInView);
    updateMainTitle();
    renderSidebar();
  }

  function renderDetail() {
    const task = tasks.find((t) => t.id === selectedId);
    const empty = !task;

    els.detailPanel?.classList.toggle("detail-empty", empty);
    if (!els.taskName) return;

    if (els.detailHint) {
      if (empty) {
        els.detailHint.hidden = false;
        els.detailHint.textContent =
          tasks.length === 0
            ? "No tasks yet. Use “Add New Task” to create one."
            : "No matching tasks. Adjust your search.";
      } else {
        els.detailHint.hidden = true;
      }
    }

    const composersLocked = [
      els.taskName,
      els.taskDesc,
      els.taskListSelect,
      els.taskDue,
      els.tagInput,
      els.subtaskInput,
      els.btnDelete,
      els.btnSave,
    ];

    if (empty) {
      els.taskName.value = "";
      els.taskDesc.value = "";
      els.taskDue.value = "";
      if (els.detailTagsPills) els.detailTagsPills.innerHTML = "";
      if (els.subtaskList) els.subtaskList.innerHTML = "";
      if (els.tagInput) els.tagInput.value = "";
      if (els.subtaskInput) els.subtaskInput.value = "";
      composersLocked.forEach((el) => {
        if (el) el.disabled = true;
      });
      updateUnsavedHint();
      return;
    }

    composersLocked.forEach((el) => {
      if (el) el.disabled = false;
    });

    els.taskName.value = task.title;
    els.taskDesc.value = task.description || "";
    els.taskDue.value = task.due || "";
    if (els.taskListSelect) els.taskListSelect.value = task.listId;

    if (els.detailTagsPills) {
      els.detailTagsPills.innerHTML = (task.tags || [])
        .map(
          (tag) =>
            `<button type="button" class="tag-pill tag-pill-dark detail-tag-pill" data-tag="${encodeURIComponent(tag)}" aria-label="Remove tag ${escapeAttr(tag)}">${escapeHtml(tag)}<span class="tag-remove" aria-hidden="true">×</span></button>`
        )
        .join("");

      els.detailTagsPills.querySelectorAll(".detail-tag-pill").forEach((btn) => {
        btn.addEventListener("click", () => {
          const raw = btn.getAttribute("data-tag");
          const tag = raw ? decodeURIComponent(raw) : "";
          removeTagFromSelected(tag);
        });
      });
    }

    if (els.subtaskList) {
      els.subtaskList.innerHTML = task.subtasks
        .map((s) => {
          const spanClass = s.done ? "task-done" : "";
          return `<li class="subtask-row" data-subtask-id="${s.id}">
            <label class="task-check task-check-sm">
              <input type="checkbox" ${s.done ? "checked" : ""} aria-label="Complete subtask" />
              <span class="task-check-ui" aria-hidden="true"></span>
            </label>
            <span class="${spanClass}">${escapeHtml(s.text)}</span>
          </li>`;
        })
        .join("");
    }

    updateUnsavedHint();
  }

  function selectTask(id, focusRow) {
    if (!tasks.some((t) => t.id === id)) return;
    if (selectedId && selectedId !== id) persistCurrentDetailIfNeeded();
    selectedId = id;
    saveApp();
    renderTaskList();
    renderDetail();
    if (focusRow) {
      requestAnimationFrame(() => {
        document.getElementById(`task-row-${selectedId}`)?.focus();
      });
    }
  }

  function onSave() {
    const task = tasks.find((t) => t.id === selectedId);
    if (!task) return;
    readFormIntoTask(task);
    saveApp();
    renderTaskList();
    renderDetail();
    showToast("Changes saved.");
  }

  function openDeleteModal() {
    const task = tasks.find((t) => t.id === selectedId);
    if (!task || !els.deleteModal || !els.deleteModalBody) return;
    els.deleteModalBody.textContent = `“${task.title}” will be removed. You can’t undo this.`;
    deleteModalPrevFocus = document.activeElement;
    els.deleteModal.hidden = false;
    els.deleteModalCancel?.focus();
  }

  function closeDeleteModal() {
    if (!els.deleteModal) return;
    els.deleteModal.hidden = true;
    if (deleteModalPrevFocus && typeof deleteModalPrevFocus.focus === "function") {
      deleteModalPrevFocus.focus();
    }
    deleteModalPrevFocus = null;
  }

  function confirmDelete() {
    const idx = tasks.findIndex((t) => t.id === selectedId);
    if (idx === -1) {
      closeDeleteModal();
      return;
    }
    tasks = tasks.filter((t) => t.id !== selectedId);
    selectedId = tasks[idx]?.id || tasks[idx - 1]?.id || tasks[0]?.id || null;
    saveApp();
    closeDeleteModal();
    renderTaskList();
    renderDetail();
    showToast("Task deleted.");
  }

  function onDeleteClick() {
    openDeleteModal();
  }

  function onAddTask() {
    persistCurrentDetailIfNeeded();
    const t = {
      id: uid("t"),
      title: "New task",
      description: "",
      listId: "personal",
      due: "",
      tags: [],
      completed: false,
      subtasks: [],
    };
    tasks.unshift(t);
    selectedId = t.id;
    if (els.searchInput) els.searchInput.value = "";
    saveApp();
    renderTaskList();
    renderDetail();
    requestAnimationFrame(() => {
      els.taskName?.focus();
      els.taskName?.select();
    });
  }

  function commitNewTag() {
    const task = tasks.find((t) => t.id === selectedId);
    if (!task || !els.tagInput) return;
    const tag = els.tagInput.value.trim();
    if (!tag) return;
    if (!task.tags) task.tags = [];
    if (!task.tags.includes(tag)) task.tags.push(tag);
    els.tagInput.value = "";
    saveApp();
    renderTaskList();
    renderDetail();
    els.tagInput.focus();
  }

  function removeTagFromSelected(tag) {
    const task = tasks.find((t) => t.id === selectedId);
    if (!task || !task.tags) return;
    task.tags = task.tags.filter((x) => x !== tag);
    saveApp();
    renderTaskList();
    renderDetail();
  }

  function commitNewSubtask() {
    const task = tasks.find((t) => t.id === selectedId);
    if (!task || !els.subtaskInput) return;
    const text = els.subtaskInput.value.trim();
    if (!text) return;
    task.subtasks.push({ id: uid("s"), text, done: false });
    els.subtaskInput.value = "";
    saveApp();
    renderTaskList();
    renderDetail();
    els.subtaskInput.focus();
  }

  function onTaskListClick(e) {
    const row = e.target.closest(".task-row");
    if (!row || !els.taskList.contains(row)) return;
    if (e.target.closest(".task-check")) return;
    const id = row.getAttribute("data-task-id");
    if (id) selectTask(id);
  }

  function onTaskListChange(e) {
    const input = e.target.closest('.task-row input[type="checkbox"]');
    if (!input || !els.taskList.contains(input)) return;
    const row = input.closest(".task-row");
    const id = row?.getAttribute("data-task-id");
    if (!id) return;
    const task = tasks.find((t) => t.id === id);
    if (!task) return;
    task.completed = input.checked;
    saveApp();
    renderTaskList();
    renderDetail();
  }

  function onTaskListKeydown(e) {
    const row = e.target.closest(".task-row");
    if (!row || !els.taskList.contains(row)) return;

    const visible = getVisibleTasks();
    const idx = visible.findIndex((t) => t.id === row.getAttribute("data-task-id"));

    if (e.key === "ArrowDown" || e.key === "ArrowUp") {
      e.preventDefault();
      const next =
        e.key === "ArrowDown"
          ? Math.min(idx + 1, visible.length - 1)
          : Math.max(idx - 1, 0);
      const t = visible[next];
      if (t) selectTask(t.id, true);
      return;
    }

    if (e.key === " " && !e.target.closest(".task-check")) {
      e.preventDefault();
      const id = row.getAttribute("data-task-id");
      const task = tasks.find((t) => t.id === id);
      if (task) {
        task.completed = !task.completed;
        saveApp();
        renderTaskList();
        renderDetail();
      }
    }
  }

  function onSubtaskChange(e) {
    const input = e.target.closest('.subtask-list input[type="checkbox"]');
    if (!input || !els.subtaskList?.contains(input)) return;
    const li = input.closest(".subtask-row");
    const sid = li?.getAttribute("data-subtask-id");
    const task = tasks.find((t) => t.id === selectedId);
    if (!task || !sid) return;
    const sub = task.subtasks.find((s) => s.id === sid);
    if (sub) sub.done = input.checked;
    saveApp();
    renderTaskList();
    renderDetail();
  }

  function scheduleSearch() {
    if (searchDebounceTimer) clearTimeout(searchDebounceTimer);
    searchDebounceTimer = setTimeout(() => {
      searchDebounceTimer = null;
      renderTaskList();
      renderDetail();
    }, 100);
  }

  function onNavClick(e) {
    const a = e.target.closest('a[href="#"]');
    if (a) e.preventDefault();
  }

  function showToast(message) {
    if (!els.toastRegion) return;
    if (toastTimer) clearTimeout(toastTimer);
    els.toastRegion.textContent = message;
    els.toastRegion.classList.add("toast-visible");
    toastTimer = setTimeout(() => {
      els.toastRegion.classList.remove("toast-visible");
      els.toastRegion.textContent = "";
      toastTimer = null;
    }, 2200);
  }

  function onBeforeUnload(e) {
    if (isDetailDirty()) {
      e.preventDefault();
      e.returnValue = "";
    }
  }

  function onGlobalKeydown(e) {
    if ((e.ctrlKey || e.metaKey) && e.key === "s") {
      const task = tasks.find((t) => t.id === selectedId);
      if (task) {
        e.preventDefault();
        onSave();
      }
    }
    if (e.key === "Escape" && els.listModal && !els.listModal.hidden) {
      e.preventDefault();
      closeListModal();
      return;
    }
    if (e.key === "Escape" && els.deleteModal && !els.deleteModal.hidden) {
      e.preventDefault();
      closeDeleteModal();
      return;
    }
    if (e.key === "Escape" && isMobileLayout() && !sidebarMinimized) {
      sidebarMinimized = true;
      saveSidebarPreference();
      applySidebarUI();
    }
  }

  function onModalBackdropClick(e) {
    if (e.target === els.deleteModal) closeDeleteModal();
  }

  function init() {
    els.taskList = document.getElementById("task-items");
    els.mainTitleText = document.getElementById("main-title-text");
    els.titleCountEl = document.getElementById("title-count");
    els.navBody = document.getElementById("nav-body");
    els.searchInput = document.getElementById("nav-search");
    els.addTaskBtn = document.getElementById("btn-add-task");
    els.taskName = document.getElementById("task-name");
    els.taskDesc = document.getElementById("task-desc");
    els.taskListSelect = document.getElementById("task-list-select");
    els.taskDue = document.getElementById("task-due");
    els.detailTagsPills = document.getElementById("detail-tags-pills");
    els.tagInput = document.getElementById("tag-input");
    els.subtaskInput = document.getElementById("subtask-input");
    els.subtaskList = document.getElementById("subtask-list");
    els.btnDelete = document.getElementById("btn-delete-task");
    els.btnSave = document.getElementById("btn-save");
    els.detailPanel = document.querySelector(".col-detail");
    els.detailHint = document.getElementById("detail-hint");
    els.unsavedHint = document.getElementById("unsaved-hint");
    els.deleteModal = document.getElementById("delete-modal");
    els.deleteModalBody = document.getElementById("delete-modal-body");
    els.deleteModalCancel = document.getElementById("delete-modal-cancel");
    els.deleteModalConfirm = document.getElementById("delete-modal-confirm");
    els.toastRegion = document.getElementById("toast-region");
    els.dashboard = document.getElementById("dashboard");
    els.menuToggle = document.getElementById("menu-toggle");
    els.listModal = document.getElementById("list-modal");
    els.listModalName = document.getElementById("list-modal-name");
    els.listModalCancel = document.getElementById("list-modal-cancel");
    els.listModalSave = document.getElementById("list-modal-save");
    els.listColorOptions = document.getElementById("list-color-options");

    mqMobile = window.matchMedia("(max-width: 960px)");
    mqMobile.addEventListener("change", onSidebarMqChange);

    loadSidebarPreference();
    applySidebarUI();

    loadApp();
    fillListSelect();
    buildListColorRadios();

    els.menuToggle?.addEventListener("click", toggleSidebar);
    els.navBody?.addEventListener("click", onNavBodyClick);
    els.listModalCancel?.addEventListener("click", closeListModal);
    els.listModalSave?.addEventListener("click", confirmListModal);
    els.listModal?.addEventListener("click", onListModalBackdropClick);
    els.listModalName?.addEventListener("keydown", (e) => {
      if (e.key === "Enter") {
        e.preventDefault();
        confirmListModal();
      }
    });

    els.taskList?.addEventListener("click", onTaskListClick);
    els.taskList?.addEventListener("change", onTaskListChange);
    els.taskList?.addEventListener("keydown", onTaskListKeydown);
    els.subtaskList?.addEventListener("change", onSubtaskChange);
    els.searchInput?.addEventListener("input", scheduleSearch);
    els.addTaskBtn?.addEventListener("click", onAddTask);
    els.btnDelete?.addEventListener("click", onDeleteClick);
    els.btnSave?.addEventListener("click", onSave);
    els.deleteModalCancel?.addEventListener("click", closeDeleteModal);
    els.deleteModalConfirm?.addEventListener("click", confirmDelete);
    els.deleteModal?.addEventListener("click", onModalBackdropClick);
    document.querySelector(".nav-footer")?.addEventListener("click", onNavClick);

    els.tagAddCommit = document.getElementById("tag-add-commit");
    els.subtaskAddCommit = document.getElementById("subtask-add-commit");
    els.tagAddCommit?.addEventListener("click", commitNewTag);
    els.tagInput?.addEventListener("keydown", (e) => {
      if (e.key === "Enter") {
        e.preventDefault();
        commitNewTag();
      }
    });
    els.subtaskAddCommit?.addEventListener("click", commitNewSubtask);
    els.subtaskInput?.addEventListener("keydown", (e) => {
      if (e.key === "Enter") {
        e.preventDefault();
        commitNewSubtask();
      }
    });

    ["input", "change"].forEach((ev) => {
      els.taskName?.addEventListener(ev, updateUnsavedHint);
      els.taskDesc?.addEventListener(ev, updateUnsavedHint);
      els.taskDue?.addEventListener(ev, updateUnsavedHint);
      els.taskListSelect?.addEventListener(ev, updateUnsavedHint);
    });

    window.addEventListener("beforeunload", onBeforeUnload);
    document.addEventListener("keydown", onGlobalKeydown);

    ensureSelection();
    renderTaskList();
    renderDetail();
  }

  if (document.readyState === "loading") {
    document.addEventListener("DOMContentLoaded", init);
  } else {
    init();
  }
})();
