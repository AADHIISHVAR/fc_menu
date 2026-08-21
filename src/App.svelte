<script>
  import { onMount } from 'svelte';
  import { initializeApp } from 'firebase/app';
  import { getDatabase, ref, onValue, set } from 'firebase/database';

  // ================================================================
  // FIREBASE CONFIGURATION
  // Replace these placeholders with your actual Firebase config.
  // ================================================================
  const firebaseConfig = {
    apiKey: "YOUR_API_KEY",
    authDomain: "YOUR_PROJECT_ID.firebaseapp.com",
    databaseURL: "YOUR_DATABASE_URL",
    projectId: "YOUR_PROJECT_ID",
    storageBucket: "YOUR_PROJECT_ID.appspot.com",
    messagingSenderId: "YOUR_SENDER_ID",
    appId: "YOUR_APP_ID"
  };

  const isConfigured = firebaseConfig.databaseURL && 
                       firebaseConfig.databaseURL !== "YOUR_DATABASE_URL" && 
                       !firebaseConfig.databaseURL.includes("YOUR_");

  const TIMES = {
    open:            9*60,
    tiffinEnds:      11*60+45,
    tiffinCutoff:    12*60+10,
    close:           18*60
  };

  const CATEGORY_META = {
    tiffin: { label:'Tiffin', color:'var(--tiffin)', soft:'var(--tiffin-soft)' },
    lunch:  { label:'Lunch',  color:'var(--accent)',  soft:'var(--accent-soft)' }
  };

  const DEFAULT_ITEMS = [
    { id:'t1', name:'Idli',         category:'tiffin', available:true },
    { id:'t2', name:'Vada',         category:'tiffin', available:true },
    { id:'t3', name:'Pongal',       category:'tiffin', available:true },
    { id:'t4', name:'Dosa',         category:'tiffin', available:true },
    { id:'t5', name:'Upma',         category:'tiffin', available:true },
    { id:'t6', name:'Poori',        category:'tiffin', available:true },
    { id:'l1', name:'Steamed Rice', category:'lunch',  available:true },
    { id:'l2', name:'Sambar',       category:'lunch',  available:true },
    { id:'l3', name:'Rasam',        category:'lunch',  available:true },
    { id:'l4', name:'Curd Rice',    category:'lunch',  available:true },
    { id:'l5', name:'Veg Kurma',    category:'lunch',  available:true },
    { id:'l6', name:'Chapati',      category:'lunch',  available:true }
  ];

  // Svelte 5 Reactive States
  let items = $state([]);
  let pin = $state("1234");
  let loading = $state(isConfigured);
  let error = $state(null);
  
  let timeString = $state("");
  let currentPhase = $state("before_open");
  let currentView = $state("menu"); // "menu" or "admin"
  let showPinModal = $state(false);
  
  // Bindings
  let pinInputVal = $state("");
  let pinErrorText = $state("");
  
  let curPinInput = $state("");
  let newPinInput = $state("");
  let confirmPinInput = $state("");
  let pinChangeMsg = $state("");
  let pinChangeStatus = $state(""); 

  let newTiffinName = $state("");
  let newLunchName = $state("");

  const PHASE_LABEL = {
    before_open:  'Opens 9:00 AM',
    tiffin:       'Tiffin Time',
    tiffin_lunch: 'Lunch',
    lunch:        'Lunch',
    closed:       'Closed'
  };

  // Derive categories to display on public menu
  const displayedCategories = $derived.by(() => {
    if (currentPhase === 'tiffin') return ['tiffin'];
    if (currentPhase === 'tiffin_lunch') return ['lunch', 'tiffin'];
    if (currentPhase === 'lunch') return ['lunch'];
    return [];
  });

  function todayStr(){
    const d = new Date();
    return d.getFullYear()+'-'+(d.getMonth()+1)+'-'+d.getDate();
  }

  let db;

  onMount(() => {
    if (!isConfigured) return;

    try {
      const app = initializeApp(firebaseConfig);
      db = getDatabase(app);
      const stateRef = ref(db, 'foodcourt_state');

      // Realtime Database listener
      onValue(stateRef, (snapshot) => {
        const val = snapshot.val();
        if (!val) {
          // Initialize DB with defaults
          const initialData = {
            pin: "1234",
            lastReset: todayStr(),
            items: DEFAULT_ITEMS
          };
          set(stateRef, initialData);
          items = DEFAULT_ITEMS;
          pin = "1234";
        } else {
          // Daily reset logic: fresh day resets all items to available
          if (val.lastReset !== todayStr()) {
            val.items.forEach(i => i.available = true);
            val.lastReset = todayStr();
            set(stateRef, val);
          }
          items = val.items || [];
          pin = val.pin || "1234";
        }
        loading = false;
      }, (err) => {
        error = err.message;
        loading = false;
      });
    } catch (e) {
      error = e.message;
      loading = false;
    }

    // Time ticker
    const updateTime = () => {
      const d = new Date();
      let h = d.getHours(), m = d.getMinutes();
      const ampm = h >= 12 ? 'PM' : 'AM';
      h = h % 12; if(h === 0) h = 12;
      timeString = h + ':' + String(m).padStart(2,'0') + ' ' + ampm;

      const mins = d.getHours()*60 + d.getMinutes();
      if(mins < TIMES.open)          currentPhase = 'before_open';
      else if(mins < TIMES.tiffinEnds)    currentPhase = 'tiffin';
      else if(mins < TIMES.tiffinCutoff)  currentPhase = 'tiffin_lunch';
      else if(mins < TIMES.close)         currentPhase = 'lunch';
      else                                currentPhase = 'closed';
    };

    updateTime();
    const interval = setInterval(updateTime, 15000);
    return () => clearInterval(interval);
  });

  function saveStateToDb(newItems, newPin) {
    if (!db) return;
    const stateRef = ref(db, 'foodcourt_state');
    set(stateRef, {
      pin: newPin !== undefined ? newPin : pin,
      lastReset: todayStr(),
      items: newItems !== undefined ? newItems : items
    });
  }

  function toggleAvail(id, checked) {
    const updated = items.map(item => {
      if (item.id === id) return { ...item, available: checked };
      return item;
    });
    saveStateToDb(updated);
  }

  function removeItem(id) {
    const updated = items.filter(item => item.id !== id);
    saveStateToDb(updated);
  }

  function addItem(cat) {
    let name = "";
    if (cat === 'tiffin') {
      name = newTiffinName.trim();
      newTiffinName = "";
    } else if (cat === 'lunch') {
      name = newLunchName.trim();
      newLunchName = "";
    }
    if (!name) return;

    const newItem = {
      id: cat[0] + Date.now(),
      name,
      category: cat,
      available: true
    };
    saveStateToDb([...items, newItem]);
  }

  function changePin() {
    if (curPinInput !== pin) {
      pinChangeMsg = "Current PIN is incorrect.";
      pinChangeStatus = "err";
      return;
    }
    if (newPinInput.length < 4) {
      pinChangeMsg = "New PIN must be at least 4 digits.";
      pinChangeStatus = "err";
      return;
    }
    if (newPinInput !== confirmPinInput) {
      pinChangeMsg = "New PIN and confirmation do not match.";
      pinChangeStatus = "err";
      return;
    }
    saveStateToDb(undefined, newPinInput);
    pinChangeMsg = "PIN updated.";
    pinChangeStatus = "ok";
    curPinInput = "";
    newPinInput = "";
    confirmPinInput = "";
  }

  function tryUnlock() {
    if (pinInputVal === pin) {
      showPinModal = false;
      currentView = "admin";
      pinInputVal = "";
      pinErrorText = "";
    } else {
      pinErrorText = "Incorrect PIN. Try again.";
      pinInputVal = "";
    }
  }

  function adminCategoryAllowed(cat) {
    const d = new Date();
    const mins = d.getHours()*60 + d.getMinutes();
    if (cat === 'tiffin') return mins < TIMES.tiffinCutoff;
    return true;
  }
</script>

{#if !isConfigured}
  <!-- Setup Instructions -->
  <div class="setup-container">
    <div class="setup-card">
      <h2>Database Setup Required</h2>
      <p class="subtitle">Please configure Firebase in your app to sync data between the Cashier and Customers.</p>
      
      <ol class="steps">
        <li>Go to the <a href="https://console.firebase.google.com/" target="_blank" rel="noreferrer">Firebase Console</a>.</li>
        <li>Create a project, then build a <strong>Realtime Database</strong>.</li>
        <li>In your database <strong>Rules</strong> tab, set read and write to <code>true</code> (for testing):
          <pre><code>{`{
  "rules": {
    ".read": true,
    ".write": true
  }
}`}</code></pre>
        </li>
        <li>Add a Web App under Project Settings, copy the configuration, and paste it into the <code>firebaseConfig</code> object at the top of <code>src/App.svelte</code>.</li>
      </ol>
    </div>
  </div>
{:else if loading}
  <div class="empty-state">
    <div class="big">Connecting...</div>
    <div>Loading food court menu state from Firebase.</div>
  </div>
{:else if error}
  <div class="empty-state">
    <div class="big" style="color:var(--danger)">Connection Error</div>
    <div>{error}</div>
  </div>
{:else}
  <!-- Main Application -->
  {#if currentView === 'menu'}
    <div id="menuView">
      <div class="top-bar">
        <div class="brand">
          <div class="name">Campus Food Court</div>
          <div class="sub">Today's menu</div>
        </div>
        <div class="status">
          <span class="clock">{timeString}</span>
          <span class="period-badge">{PHASE_LABEL[currentPhase]}</span>
        </div>
      </div>

      <div class="menu-body">
        {#if displayedCategories.length === 0}
          <div class="empty-state">
            <div class="big">
              {currentPhase === 'before_open' ? 'Opening soon' : 'Closed for today'}
            </div>
            <div>
              {currentPhase === 'before_open' ? 'The food court opens at 9:00 AM.' : 'See you tomorrow from 9:00 AM.'}
            </div>
          </div>
        {:else}
          {#each displayedCategories as cat}
            {@const meta = CATEGORY_META[cat]}
            {@const catItems = items.filter(i => i.category === cat && i.available)}
            <div class="category-block">
              <div class="category-head">
                <span class="dot" style="background:{meta.color}"></span>
                <h2>{meta.label}</h2>
                {#if currentPhase === 'tiffin_lunch' && cat === 'tiffin'}
                  <span class="note">— while stocks last</span>
                {/if}
              </div>

              {#if catItems.length === 0}
                <div class="empty-state" style="padding:30px;">
                  <div>No {meta.label.toLowerCase()} items available right now.</div>
                </div>
              {:else}
                <div class="item-grid">
                  {#each catItems as item (item.id)}
                    <div class="item-card">
                      <span class="avail-dot" style="background:{meta.color}"></span>
                      {item.name}
                    </div>
                  {/each}
                </div>
              {/if}
            </div>
          {/each}
        {/if}
      </div>

      <button class="gear-btn" onclick={() => { showPinModal = true; pinInputVal = ""; pinErrorText = ""; }} title="Staff access">&#9881;</button>
    </div>

    <!-- PIN Overlay Modal -->
    {#if showPinModal}
      <div class="overlay">
        <div class="modal">
          <h3>Staff Access</h3>
          <p>Enter PIN to manage item availability</p>
          <input 
            type="password" 
            inputmode="numeric" 
            maxlength="6" 
            class="pin-input" 
            bind:value={pinInputVal} 
            onkeydown={(e) => e.key === 'Enter' && tryUnlock()}
            autocomplete="off" 
          />
          <div class="error-text">{pinErrorText}</div>
          <div class="btn-row">
            <button class="btn" onclick={() => showPinModal = false}>Cancel</button>
            <button class="btn btn-primary" onclick={tryUnlock}>Unlock</button>
          </div>
        </div>
      </div>
    {/if}
  {:else if currentView === 'admin'}
    <!-- Admin View -->
    <div id="adminView">
      <div class="admin-bar">
        <div class="name">Admin — Item Availability</div>
        <button class="btn" onclick={() => currentView = 'menu'}>Back to Menu</button>
      </div>
      <div class="admin-body">
        {#each ['tiffin', 'lunch'] as cat}
          {@const meta = CATEGORY_META[cat]}
          {@const allowed = adminCategoryAllowed(cat)}
          {@const catItems = items.filter(i => i.category === cat)}
          
          <div class="admin-section">
            <div class="admin-section-head">
              <span class="dot" style="background:{meta.color}"></span>
              <h3>{meta.label}</h3>
            </div>

            {#if !allowed}
              <div class="locked-note">Tiffin is closed for the day after 12:10 PM.</div>
            {:else}
              {#each catItems as item (item.id)}
                <div class="item-row">
                  <label>
                    <input 
                      type="checkbox" 
                      checked={item.available} 
                      onchange={(e) => toggleAvail(item.id, e.target.checked)} 
                    />
                    {item.name}
                  </label>
                  <button class="remove-btn" onclick={() => removeItem(item.id)} title="Remove item">&times;</button>
                </div>
              {/each}
              
              <div class="add-item-row">
                {#if cat === 'tiffin'}
                  <input 
                    type="text" 
                    placeholder="Add tiffin item" 
                    bind:value={newTiffinName} 
                    onkeydown={(e) => e.key === 'Enter' && addItem('tiffin')} 
                  />
                  <button onclick={() => addItem('tiffin')}>Add</button>
                {:else if cat === 'lunch'}
                  <input 
                    type="text" 
                    placeholder="Add lunch item" 
                    bind:value={newLunchName} 
                    onkeydown={(e) => e.key === 'Enter' && addItem('lunch')} 
                  />
                  <button onclick={() => addItem('lunch')}>Add</button>
                {/if}
              </div>
            {/if}
          </div>
        {/each}

        <div class="pin-settings">
          <h3>Change PIN</h3>
          <div class="pin-fields">
            <input type="password" inputmode="numeric" placeholder="Current PIN" bind:value={curPinInput} />
            <input type="password" inputmode="numeric" placeholder="New PIN" bind:value={newPinInput} />
            <input type="password" inputmode="numeric" placeholder="Confirm new PIN" bind:value={confirmPinInput} />
          </div>
          <div class={`pin-msg ${pinChangeStatus}`}>{pinChangeMsg}</div>
          <button class="btn" onclick={changePin}>Update PIN</button>
        </div>
      </div>
    </div>
  {/if}
{/if}

<style>
  :root{
    --bg:#F6F5F1;
    --surface:#FFFFFF;
    --text:#1C201E;
    --text-muted:#6B716D;
    --border:#E4E1D8;
    --accent:#0E6E5B;
    --accent-soft:#E5F1EE;
    --tiffin:#C97A2B;
    --tiffin-soft:#FBEEDF;
    --danger:#B3261E;
    --danger-soft:#FBEAE9;
    --radius:14px;
    --shadow:0 1px 2px rgba(20,20,10,0.05), 0 6px 18px rgba(20,20,10,0.06);
  }

  /* Setup screen */
  .setup-container {
    display: flex;
    justify-content: center;
    align-items: center;
    min-height: 100vh;
    padding: 24px;
    background: var(--bg);
  }
  .setup-card {
    background: var(--surface);
    border: 1px solid var(--border);
    border-radius: var(--radius);
    padding: 32px;
    max-width: 600px;
    width: 100%;
    box-shadow: var(--shadow);
  }
  .setup-card h2 {
    margin: 0 0 8px 0;
    font-size: 24px;
    color: var(--text);
  }
  .setup-card .subtitle {
    color: var(--text-muted);
    font-size: 15px;
    margin: 0 0 24px 0;
    line-height: 1.5;
  }
  .setup-card .steps {
    margin: 0;
    padding-left: 20px;
    color: var(--text);
    line-height: 1.6;
  }
  .setup-card .steps li {
    margin-bottom: 16px;
  }
  .setup-card a {
    color: var(--accent);
    text-decoration: underline;
  }
  .setup-card pre {
    background: var(--bg);
    padding: 12px;
    border-radius: 8px;
    overflow-x: auto;
    margin: 8px 0 0 0;
  }
  .setup-card code {
    font-family: monospace;
    font-size: 13px;
  }

  /* App Views */
  #menuView{
    min-height:100vh;
    display:flex;
    flex-direction:column;
  }

  .top-bar{
    background:var(--surface);
    border-bottom:1px solid var(--border);
    padding:20px 28px;
    display:flex;
    align-items:center;
    justify-content:space-between;
    flex-wrap:wrap;
    gap:10px;
  }
  .brand{
    display:flex;
    flex-direction:column;
    gap:2px;
  }
  .brand .name{
    font-size:22px;
    font-weight:700;
    letter-spacing:-0.01em;
  }
  .brand .sub{
    font-size:13px;
    color:var(--text-muted);
  }
  .status{
    display:flex;
    align-items:center;
    gap:14px;
  }
  .clock{
    font-variant-numeric: tabular-nums;
    font-size:15px;
    color:var(--text-muted);
  }
  .period-badge{
    font-size:13px;
    font-weight:700;
    text-transform:uppercase;
    letter-spacing:0.06em;
    padding:8px 14px;
    border-radius:999px;
    background:var(--accent-soft);
    color:var(--accent);
  }

  .menu-body{
    flex:1;
    padding:28px;
    max-width:1080px;
    margin:0 auto;
    width:100%;
  }

  .category-block{
    margin-bottom:34px;
    animation:fadeIn .35s ease;
  }
  .category-head{
    display:flex;
    align-items:center;
    gap:10px;
    margin-bottom:14px;
  }
  .dot{
    width:10px; height:10px; border-radius:50%;
    flex-shrink:0;
  }
  .category-head h2{
    font-size:15px;
    margin:0;
    text-transform:uppercase;
    letter-spacing:0.07em;
    color:var(--text-muted);
    font-weight:700;
  }
  .category-head .note{
    font-size:12px;
    color:var(--text-muted);
    font-weight:400;
    text-transform:none;
    letter-spacing:normal;
  }

  .item-grid{
    display:grid;
    grid-template-columns:repeat(auto-fill, minmax(200px, 1fr));
    gap:12px;
  }
  .item-card{
    background:var(--surface);
    border:1px solid var(--border);
    border-radius:var(--radius);
    padding:16px 18px;
    box-shadow:var(--shadow);
    font-size:16px;
    font-weight:600;
    display:flex;
    align-items:center;
    gap:10px;
    color: var(--text);
  }
  .item-card .avail-dot{
    width:8px; height:8px; border-radius:50%;
    flex-shrink:0;
  }

  .empty-state{
    text-align:center;
    padding:80px 20px;
    color:var(--text-muted);
  }
  .empty-state .big{
    font-size:22px;
    font-weight:700;
    color:var(--text);
    margin-bottom:8px;
  }

  .gear-btn{
    position:fixed;
    bottom:18px;
    right:18px;
    width:38px;
    height:38px;
    border-radius:50%;
    border:1px solid var(--border);
    background:var(--surface);
    color:var(--text-muted);
    opacity:0.35;
    font-size:16px;
    display:flex;
    align-items:center;
    justify-content:center;
    transition:opacity .15s ease;
    cursor: pointer;
  }
  .gear-btn:hover{ opacity:1; }

  /* Pin Modal */
  .overlay{
    position:fixed; inset:0;
    background:rgba(20,22,20,0.45);
    display:flex; align-items:center; justify-content:center;
    padding:20px;
    z-index:50;
  }
  .modal{
    background:var(--surface);
    border-radius:var(--radius);
    padding:28px;
    width:100%;
    max-width:320px;
    box-shadow:var(--shadow);
    text-align:center;
  }
  .modal h3{
    margin:0 0 6px;
    font-size:18px;
    color: var(--text);
  }
  .modal p{
    margin:0 0 18px;
    color:var(--text-muted);
    font-size:13px;
  }
  .pin-input{
    width:100%;
    font-size:24px;
    letter-spacing:0.5em;
    text-align:center;
    padding:12px;
    border-radius:10px;
    border:1px solid var(--border);
    margin-bottom:14px;
  }
  .pin-input:focus{
    outline:2px solid var(--accent);
    outline-offset:1px;
  }
  .error-text{
    color:var(--danger);
    font-size:13px;
    margin:-8px 0 14px;
    min-height:16px;
  }
  .btn-row{ display:flex; gap:10px; }
  .btn{
    flex:1;
    padding:11px;
    border-radius:10px;
    border:1px solid var(--border);
    background:var(--surface);
    font-weight:600;
    font-size:14px;
    cursor: pointer;
  }
  .btn-primary{
    background:var(--accent);
    border-color:var(--accent);
    color:#fff;
  }

  /* Admin View */
  #adminView{
    min-height:100vh;
  }
  .admin-bar{
    background:var(--surface);
    border-bottom:1px solid var(--border);
    padding:18px 28px;
    display:flex;
    align-items:center;
    justify-content:space-between;
    flex-wrap:wrap;
    gap:10px;
  }
  .admin-bar .name{ font-size:18px; font-weight:700; color: var(--text); }
  .admin-body{
    max-width:900px;
    margin:0 auto;
    padding:24px 20px 60px;
  }

  .admin-section{
    background:var(--surface);
    border:1px solid var(--border);
    border-radius:var(--radius);
    padding:20px;
    margin-bottom:18px;
  }
  .admin-section-head{
    display:flex;
    align-items:center;
    gap:10px;
    margin-bottom:14px;
  }
  .admin-section-head h3{
    margin:0;
    font-size:15px;
    text-transform:uppercase;
    letter-spacing:0.06em;
    color: var(--text);
  }

  .locked-note{
    padding:16px;
    border-radius:10px;
    background:#F5F4EF;
    color:var(--text-muted);
    font-size:14px;
  }

  .item-row{
    display:flex;
    align-items:center;
    gap:12px;
    padding:10px 4px;
    border-bottom:1px solid var(--border);
  }
  .item-row:last-child{ border-bottom:none; }
  .item-row label{
    flex:1;
    display:flex;
    align-items:center;
    gap:10px;
    font-size:15px;
    font-weight:600;
    color: var(--text);
  }
  .item-row input[type="checkbox"]{
    width:18px; height:18px;
    accent-color:var(--accent);
  }
  .remove-btn{
    border:none;
    background:none;
    color:var(--text-muted);
    font-size:18px;
    line-height:1;
    padding:4px 8px;
    border-radius:6px;
    cursor: pointer;
  }
  .remove-btn:hover{ background:var(--danger-soft); color:var(--danger); }

  .add-item-row{
    display:flex;
    gap:8px;
    margin-top:12px;
  }
  .add-item-row input{
    flex:1;
    padding:9px 12px;
    border-radius:8px;
    border:1px solid var(--border);
    font-size:14px;
  }
  .add-item-row button{
    padding:9px 16px;
    border-radius:8px;
    border:1px solid var(--accent);
    background:var(--accent-soft);
    color:var(--accent);
    font-weight:700;
    font-size:14px;
    cursor: pointer;
  }

  .pin-settings{
    background:var(--surface);
    border:1px solid var(--border);
    border-radius:var(--radius);
    padding:20px;
  }
  .pin-settings h3{
    margin:0 0 12px;
    font-size:15px;
    text-transform:uppercase;
    letter-spacing:0.06em;
    color: var(--text);
  }
  .pin-fields{
    display:grid;
    grid-template-columns:repeat(auto-fit, minmax(140px,1fr));
    gap:10px;
    margin-bottom:12px;
  }
  .pin-fields input{
    padding:9px 12px;
    border-radius:8px;
    border:1px solid var(--border);
    font-size:14px;
  }
  .pin-msg{
    font-size:13px;
    margin-bottom:10px;
    min-height:16px;
  }
  .pin-msg.ok{ color:var(--accent); }
  .pin-msg.err{ color:var(--danger); }

  @keyframes fadeIn{
    from{ opacity:0; transform:translateY(4px); }
    to{ opacity:1; transform:translateY(0); }
  }

  @media (max-width:520px){
    .top-bar, .admin-bar{ padding:16px; }
    .menu-body{ padding:18px; }
  }
</style>
