<!doctype html>
<html lang="en">
<head>
  <meta charset="utf-8" />
  <meta name="viewport" content="width=device-width,initial-scale=1" />
  <title>Quick Tasks — Simple To-Do App</title>
  <style>
    /* Basic reset */
    * { box-sizing: border-box; }
    html,body { height:100%; margin:0; font-family: system-ui, -apple-system, "Segoe UI", Roboto, "Helvetica Neue", Arial; background:#f5f7fb; color:#0f1724; }
    .app {
      max-width:920px; margin:32px auto; padding:20px;
      background:linear-gradient(180deg, #ffffff 0%, #fbfdff 100%);
      border-radius:12px; box-shadow: 0 10px 30px rgba(12,24,48,0.08);
    }
    header { display:flex; align-items:center; gap:16px; }
    h1 { margin:0; font-size:20px; letter-spacing:0.2px; }
    .subtitle { color:#4b5563; font-size:13px; margin-top:4px; }
    .controls { display:flex; gap:8px; margin:18px 0; flex-wrap:wrap; }
    .input, .search {
      padding:10px 12px; border-radius:8px; border:1px solid #e6eef8; min-width:180px; background:#fff;
      box-shadow:inset 0 1px 0 rgba(16,24,40,0.02);
      font-size:14px;
    }
    .btn {
      padding:10px 14px; border-radius:8px; border:0; cursor:pointer; font-weight:600; font-size:14px;
      background:linear-gradient(180deg,#0ea5a4,#02848a); color:white;
    }
    .btn.secondary { background:transparent; border:1px solid #e6eef8; color:#0f1724; font-weight:600; }
    .list { margin-top:12px; display:grid; gap:8px; }
    .item {
      display:flex; align-items:center; gap:10px; padding:10px; border-radius:10px;
      border:1px solid #eef6fb; background:linear-gradient(180deg,#ffffff,#fbfdff);
    }
    .item.done { opacity:0.6; text-decoration:line-through; }
    .item .text { flex:1; word-break:break-word; }
    .meta { font-size:12px; color:#6b7280; }
    .actions { display:flex; gap:6px; }
    .icon-btn { border:0; background:transparent; cursor:pointer; padding:6px; border-radius:6px; }
    .small { font-size:13px; padding:6px 10px; border-radius:8px; }
    .footer { margin-top:14px; display:flex; justify-content:space-between; align-items:center; color:#6b7280; font-size:13px; }
    @media (max-width:560px){
      .controls { flex-direction:column; }
      header { gap:10px; }
    }
  </style>
</head>
<body>
  <main class="app" role="main" aria-labelledby="title">
    <header>
      <div>
        <h1 id="title">Quick Tasks</h1>
        <div class="subtitle">Add tasks, edit, mark done — saved in your browser.</div>
      </div>
    </header>

    <section class="controls" aria-label="task controls">
      <input id="taskInput" class="input" placeholder="Write a task or note (press Enter to add)" aria-label="New task" />
      <button id="addBtn" class="btn" aria-label="Add task">Add</button>

      <input id="search" class="search" placeholder="Search tasks" aria-label="Search tasks" />
      <select id="filter" class="input" aria-label="Filter tasks">
        <option value="all">All</option>
        <option value="active">Active</option>
        <option value="done">Done</option>
      </select>

      <button id="clearDone" class="btn secondary small" title="Remove all completed tasks">Clear done</button>
      <button id="export" class="btn secondary small" title="Export tasks as JSON">Export</button>
      <button id="importBtn" class="btn secondary small" title="Import JSON">Import</button>
      <input id="importFile" type="file" accept="application/json" style="display:none" />
    </section>

    <section>
      <div id="list" class="list" aria-live="polite"></div>
    </section>

    <div class="footer">
      <div id="count">0 tasks</div>
      <div class="meta">Local only • No account required</div>
    </div>
  </main>

  <script>
    // Simple Todo app — stores data in localStorage
    const KEY = 'quick_tasks_v1';
    const taskInput = document.getElementById('taskInput');
    const addBtn = document.getElementById('addBtn');
    const listEl = document.getElementById('list');
    const countEl = document.getElementById('count');
    const searchEl = document.getElementById('search');
    const filterEl = document.getElementById('filter');
    const clearDoneBtn = document.getElementById('clearDone');
    const exportBtn = document.getElementById('export');
    const importBtn = document.getElementById('importBtn');
    const importFile = document.getElementById('importFile');

    let tasks = [];

    // Utilities
    function save(){ localStorage.setItem(KEY, JSON.stringify(tasks)); render(); }
    function load(){ try{ const s = localStorage.getItem(KEY); tasks = s ? JSON.parse(s) : []; } catch(e){ tasks=[]; } }
    function uid(){ return Date.now().toString(36) + Math.random().toString(36).slice(2,8); }

    // Render tasks with filter & search
    function render(){
      listEl.innerHTML = '';
      const q = searchEl.value.trim().toLowerCase();
      const filter = filterEl.value;
      const visible = tasks.filter(t => {
        if(filter === 'active' && t.done) return false;
        if(filter === 'done' && !t.done) return false;
        if(q && !t.text.toLowerCase().includes(q)) return false;
        return true;
      });

      visible.sort((a,b)=> b.updatedAt - a.updatedAt); // most recent first

      visible.forEach(t => {
        const item = document.createElement('div');
        item.className = 'item' + (t.done? ' done' : '');
        item.setAttribute('data-id', t.id);

        const checkbox = document.createElement('input');
        checkbox.type = 'checkbox';
        checkbox.checked = !!t.done;
        checkbox.setAttribute('aria-label','Mark complete');
        checkbox.addEventListener('change', () => {
          t.done = checkbox.checked;
          t.updatedAt = Date.now();
          save();
        });

        const text = document.createElement('div');
        text.className = 'text';
        text.textContent = t.text;

        const meta = document.createElement('div');
        meta.className = 'meta';
        const dt = new Date(t.updatedAt);
        meta.textContent = dt.toLocaleString();

        const actions = document.createElement('div');
        actions.className = 'actions';

        const editBtn = document.createElement('button');
        editBtn.className = 'icon-btn';
        editBtn.title = 'Edit';
        editBtn.innerText = '✏️';
        editBtn.addEventListener('click', ()=> startEdit(t.id));

        const delBtn = document.createElement('button');
        delBtn.className = 'icon-btn';
        delBtn.title = 'Delete';
        delBtn.innerText = '🗑️';
        delBtn.addEventListener('click', ()=>{
          if(confirm('Delete this task?')) {
            tasks = tasks.filter(x=> x.id !== t.id);
            save();
          }
        });

        actions.appendChild(editBtn);
        actions.appendChild(delBtn);

        item.appendChild(checkbox);
        item.appendChild(text);
        item.appendChild(actions);
        item.appendChild(meta);

        listEl.appendChild(item);
      });

      countEl.textContent = `${tasks.length} task${tasks.length!==1? 's':''}`;
    }

    // Add new
    function addTask(text){
      const trimmed = text.trim();
      if(!trimmed) return;
      tasks.push({ id: uid(), text: trimmed, done:false, createdAt:Date.now(), updatedAt:Date.now() });
      save();
      taskInput.value = '';
      taskInput.focus();
    }

    // Edit
    function startEdit(id){
      const t = tasks.find(x=> x.id===id);
      if(!t) return;
      const newText = prompt('Edit task:', t.text);
      if(newText === null) return;
      const trimmed = newText.trim();
      if(!trimmed) return alert('Task cannot be empty.');
      t.text = trimmed;
      t.updatedAt = Date.now();
      save();
    }

    // Events
    addBtn.addEventListener('click', ()=> addTask(taskInput.value));
    taskInput.addEventListener('keydown', e => { if(e.key === 'Enter') addTask(taskInput.value); });
    searchEl.addEventListener('input', render);
    filterEl.addEventListener('change', render);

    clearDoneBtn.addEventListener('click', ()=>{
      if(!confirm('Remove all completed tasks?')) return;
      tasks = tasks.filter(t => !t.done);
      save();
    });

    exportBtn.addEventListener('click', ()=>{
      const dataStr = JSON.stringify(tasks, null, 2);
      const blob = new Blob([dataStr], {type:'application/json'});
      const url = URL.createObjectURL(blob);
      const a = document.createElement('a');
      a.href = url; a.download = 'quick-tasks.json'; document.body.appendChild(a); a.click();
      a.remove(); URL.revokeObjectURL(url);
    });

    importBtn.addEventListener('click', ()=> importFile.click());
    importFile.addEventListener('change', ()=>{
      const file = importFile.files[0];
      if(!file) return;
      const reader = new FileReader();
      reader.onload = e => {
        try {
          const imported = JSON.parse(e.target.result);
          if(!Array.isArray(imported)) throw new Error('Invalid format');
          // merge by id, keep imported items, keep existing ones not conflicting
          const map = new Map();
          tasks.forEach(t=> map.set(t.id, t));
          imported.forEach(t => {
            if(t && t.id && t.text) map.set(t.id, { ...t, updatedAt: t.updatedAt || Date.now() });
          });
          tasks = Array.from(map.values());
          save();
          alert('Imported tasks successfully.');
        } catch(err){
          alert('Failed to import: ' + err.message);
        }
      };
      reader.readAsText(file);
      importFile.value = '';
    });

    // init
    load();
    render();

    // keyboard shortcut: New task focus (press "N")
    document.addEventListener('keydown', e => {
      if(e.key.toLowerCase() === 'n' && document.activeElement.tagName !== 'INPUT' && document.activeElement.tagName !== 'TEXTAREA'){
        e.preventDefault(); taskInput.focus();
      }
    });

    // Autosave interval (defensive)
    setInterval(()=> localStorage.setItem(KEY, JSON.stringify(tasks)), 10000);
  </script>
</body>
</html>

