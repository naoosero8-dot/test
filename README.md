<!doctype html>
<html lang="ja">
<head>
<meta charset="utf-8" />
<meta name="viewport" content="width=device-width,initial-scale=1" />
<title>オセロニアポーカー — Firebase版</title>
<style>
  :root{--bg:#f6f6f8;--card:#fff;--accent:#2b6df6}
  body{font-family:system-ui,-apple-system,"Hiragino Kaku Gothic ProN","メイリオ",sans-serif;margin:0;background:var(--bg);color:#111}
  header{background:#111;color:#fff;padding:10px 16px}
  .wrap{max-width:1100px;margin:18px auto;padding:12px;background:#fff;border-radius:8px;box-shadow:0 4px 18px rgba(0,0,0,.06)}
  h1{margin:0 0 12px 0;font-size:1.2rem}
  .row{display:flex;gap:12px;align-items:center}
  .col{display:flex;flex-direction:column;gap:8px}
  button{padding:8px 10px;border-radius:6px;border:1px solid #ddd;background:#fafafa;cursor:pointer}
  input,select{padding:8px;border-radius:6px;border:1px solid #ccc}
  .hidden{display:none}
  .card-grid{display:grid;grid-template-columns:repeat(auto-fill,minmax(90px,1fr));gap:8px}
  .card{background:var(--card);border:1px solid #ddd;border-radius:6px;padding:6px;text-align:center;font-size:12px}
  .card img{width:100%;height:96px;object-fit:cover;border-radius:4px}
  .small{font-size:12px;color:#666}
  .controls{display:flex;gap:8px;flex-wrap:wrap}
  .timer{font-weight:700;color:#c33}
  .status{padding:8px;background:#f2f8ff;border-radius:6px;border:1px solid #dfefff}
  textarea{width:100%;min-height:64px}
  .selected{outline:3px solid rgba(43,109,246,.18)}
  .hit{background:#eaf5ff}
  footer{font-size:12px;color:#666;margin-top:12px}
  .btn-primary{background:var(--accent);color:#fff;border:none}
  /* admin-login pane */
  #adminLogin{padding:12px;border:1px solid #eee;background:#fff;border-radius:8px}
  .card.small-thumb img{width:46px;height:68px;object-fit:cover}
</style>
</head>
<body>
<header><h1 style="display:inline">オセロニアポーカー — Firebase版</h1></header>
<div class="wrap">

  <!-- スタート / ルーム作成 -->
  <div id="startView">
    <div class="row">
      <div class="col" style="flex:1">
        <label>合言葉（4桁・新規作成）<input id="newCode" maxlength="4" placeholder="例: 1234"></label>
        <label>人数上限<select id="maxPlayers"><option>4</option><option>5</option><option>6</option></select></label>
        <label>あなたのニックネーム<input id="creatorName" value="Player"></label>
        <div class="controls"><button id="btnCreate" class="btn-primary">部屋を作る</button>
        <button id="btnJoinExisting">合言葉で入室</button>
        <button id="btnAdminLogin">管理者ログイン</button></div>
        <div id="startMsg" class="small"></div>
      </div>

      <div style="width:320px">
        <div class="status"><strong>操作説明</strong>
          <div class="small">このファイルは Firebase Realtime Database を使って複数端末で同期します。操作は「Act as」で操作するプレイヤーを切り替えてください。</div>
        </div>
      </div>
    </div>
  </div>

  <!-- Join dialog -->
  <div id="joinView" class="hidden">
    <div class="row">
      <div class="col" style="flex:1">
        <label>合言葉（4桁）<input id="joinCode" maxlength="4"></label>
        <label>ニックネーム<input id="joinName" value="Player"></label>
        <div class="controls">
          <button id="btnJoin" class="btn-primary">入室</button>
          <button id="btnBackStart">戻る</button>
        </div>
        <div id="joinMsg" class="small"></div>
      </div>
    </div>
  </div>

  <!-- Room Lobby -->
  <div id="lobbyView" class="hidden">
    <div class="row">
      <div style="flex:1">
        <div><strong>ルーム</strong> <span id="roomCodeDisplay"></span></div>
        <div class="small">部屋主: <span id="roomOwner"></span></div>
        <div style="margin-top:8px">
          <label>Act as:
            <select id="actAsSelect"></select>
          </label>
          <label style="margin-left:8px">自分の表示名<input id="myDisplay" style="width:160px"></label>
        </div>
        <div style="margin-top:8px">
          <button id="btnReady">準備OK</button>
          <button id="btnStartGame" class="hidden btn-primary">ゲーム開始（部屋主のみ）</button>
          <button id="btnLeave">退出</button>
          <button id="btnDissolve" class="hidden">部屋解散（部屋主）</button>
        </div>
        <div style="margin-top:8px"><strong>プレイヤー一覧</strong></div>
        <div id="playersList" class="card-grid" style="margin-top:8px"></div>
      </div>
      <div style="width:340px">
        <div class="status">
          <div><strong>ルーム操作</strong></div>
          <div class="small">部屋主はキック・解散権限を持ちます。</div>
        </div>
        <div style="margin-top:12px">
          <strong>管理者</strong>
          <div class="small">管理者ログインでカード編集が可能。</div>
          <button id="btnOpenAdmin">管理画面を開く</button>
        </div>
      </div>
    </div>
  </div>

  <!-- Game area -->
  <div id="gameView" class="hidden">
    <div class="row" style="align-items:flex-start">
      <div style="flex:1">
        <div style="display:flex;align-items:center;gap:12px">
          <strong>状態:</strong> <div id="phaseLabel" class="small hit">待機</div>
          <div class="small">順番表示: <span id="orderDisplay"></span></div>
          <div class="small">現在ターン: <span id="currentTurnDisplay"></span></div>
          <div style="margin-left:auto"><button id="btnBackLobby">ロビーへ戻る</button></div>
        </div>

        <!-- Hand area -->
        <h3>あなたの手札（Act asに合わせて表示）</h3>
        <div id="handArea" class="card-grid"></div>

        <!-- Exchange area -->
        <div id="exchangePanel" class="hidden">
          <h4>交換フェーズ（順番制）</h4>
          <div>ラウンド: <span id="exchangeRoundDisplay">1</span> / <span id="exchangeIndexDisplay">0</span></div>
          <div>残り時間: <span id="exchangeTimer" class="timer">30</span></div>
          <div class="controls">
            <button id="btnExchangeDo">選択したカードを交換</button>
            <button id="btnExchangeOK">OK（交換なし）</button>
          </div>
          <div class="small">※捨てたカードは山札の一番下へ入ります</div>
        </div>

        <!-- Submit area -->
        <div id="submitPanel" class="hidden">
          <h4>提出フェーズ（全員同時）</h4>
          <div>残り時間: <span id="submitTimer" class="timer">100</span></div>
          <div class="small">カードをドラッグで並べ替え、役名を入力して提出</div>
          <div id="submitHand" class="card-grid" style="margin-top:8px"></div>
          <div style="margin-top:8px">
            役名: <input id="roleInput" style="width:60%"><button id="btnSubmit" class="btn-primary">提出</button>
          </div>
        </div>

        <!-- Vote area -->
        <div id="votePanel" class="hidden">
          <h4>投票フェーズ</h4>
          <div>残り時間: <span id="voteTimer" class="timer">120</span></div>
          <div id="submissionsArea" style="margin-top:8px"></div>
        </div>

        <!-- Result area -->
        <div id="resultPanel" class="hidden">
          <h4>結果</h4>
          <div id="resultDisplay"></div>
          <div style="margin-top:8px"><button id="btnNextGame">次のゲーム（リセット）</button></div>
        </div>
      </div>

      <!-- Right column: deck / discard / log -->
      <div style="width:320px">
        <div style="margin-bottom:8px"><strong>山札</strong> <span id="deckCount"></span> / <strong>捨て札</strong> <span id="discardCount"></span></div>
        <div style="margin-bottom:8px">
          <button id="btnShowDeck">山札の中身（デバッグ）</button>
        </div>
        <div style="margin-top:12px">
          <strong>操作ログ</strong>
          <div id="logArea" style="height:320px;overflow:auto;border:1px solid #eee;padding:8px;background:#fafafa"></div>
        </div>
      </div>
    </div>
  </div>

  <!-- Admin panel -->
  <div id="adminPanel" class="hidden" style="margin-top:12px">
    <h3>管理画面 — カード編集</h3>
    <div class="small">カードは Firebase Realtime Database に保存されます。画像は DataURL（base64）で保存しています。</div>
    <div style="margin-top:8px">
      <label>カード検索（ID or 名前）<input id="adminSearch" placeholder="例: 001"></label>
      <label style="margin-left:8px">管理者パスワード（ローカル簡易）<input id="adminPass" type="password" placeholder="password"></label>
      <button id="btnAdminAuth">ログイン</button>
    </div>
    <div id="adminCards" class="card-grid" style="margin-top:8px;height:420px;overflow:auto"></div>
    <div style="margin-top:8px">
      <button id="btnExport">カードデータをエクスポート（JSON）</button>
      <button id="btnImport">カードデータをインポート（JSON）</button>
      <input type="file" id="importFile" accept="application/json" class="hidden">
    </div>
  </div>

  <!-- Admin login pane (inline form) -->
  <div id="adminLogin" class="hidden" style="margin-top:12px">
    <h3>管理者ログイン</h3>
    <div class="small">管理者操作をするにはパスワードが必要です（開発用）。本番では Firebase Authentication を推奨します。</div>
    <div style="margin-top:8px">
      <input id="adminPassword" type="password" placeholder="パスワード">
      <button id="adminLoginBtn" class="btn-primary">ログイン</button>
      <button id="adminLoginCancel">キャンセル</button>
    </div>
    <div id="adminLoginMsg" class="small"></div>
  </div>

  <footer style="margin-top:12px" class="small">※このファイルは Firebase にデータを保存します。公開時の認証・ルール設定は慎重に行ってください。</footer>
</div>

<!-- Firebase (compat) -->
<script src="https://www.gstatic.com/firebasejs/10.8.0/firebase-app-compat.js"></script>
<script src="https://www.gstatic.com/firebasejs/10.8.0/firebase-database-compat.js"></script>

<script>
/* -----------------------
   Firebase設定（提供済み）
   ----------------------- */
const firebaseConfig = {
    apiKey: "AIzaSyD3VpbKxDec8f5nY4iazDxp-wI_EBRlKI0",
    authDomain: "othelonia-poker.firebaseapp.com",
    databaseURL: "https://othelonia-poker-default-rtdb.firebaseio.com",
    projectId: "othelonia-poker",
    storageBucket: "othelonia-poker.firebasestorage.app",
    messagingSenderId: "106216770630",
    appId: "1:106216770630:web:9bf578ea363da82ee60e47",
    measurementId: "G-3BMTTJX3QT"
};
firebase.initializeApp(firebaseConfig);
const db = firebase.database();

/* -----------------------
   定数・初期データ
   ----------------------- */
const CARD_TOTAL = 90;
const ADMIN_PASSWORD = 'nanamiya333'; // ※開発用。公開時は Auth へ移行してください。

/* -----------------------
   グローバル状態（クライアント側）
   ----------------------- */
let cards = [];            // 最新カード配列（DB同期）
let rooms = {};            // ルーム一覧（DB同期キャッシュ）
let currentRoom = null;    // 現在入っているルームコード
let actingPlayerId = null; // Act as で操作中のプレイヤーID
let myLocalPlayerId = null; // このブラウザインスタンスで作った player id
let isAdmin = false;

/* -----------------------
   ユーティリティ
   ----------------------- */
function log(msg){ const el = document.getElementById('logArea'); el.innerHTML = `<div class="small">[${(new Date()).toLocaleTimeString()}] ${escapeHtml(msg)}</div>` + el.innerHTML; }
function escapeHtml(s){ return (s+'').replace(/&/g,'&amp;').replace(/</g,'&lt;').replace(/>/g,'&gt;'); }
function uid(prefix='p'){ return prefix + Math.random().toString(36).slice(2,9); }
function shuffle(arr){ for(let i=arr.length-1;i>0;i--){ const j=Math.floor(Math.random()* (i+1)); [arr[i],arr[j]]=[arr[j],arr[i]] } }

/* -----------------------
   Firebase リスナー：cards（管理画面で編集されたカード）
   ----------------------- */
db.ref('cards').on('value', (snap) => {
  const val = snap.val();
  if(!val){
    // 初期カードを DB に書き込む（最初の一度だけ）
    const init = [];
    for(let i=1;i<=CARD_TOTAL;i++){
      const id = String(i).padStart(3,'0');
      init.push({id, name: `Card ${id}`, img: null});
    }
    db.ref('cards').set(init);
    return;
  }
  cards = val;
  saveCardsLocal(cards);
  renderAdminCards();
  refreshGameUI(); // 画像やカードが変わった場合に UI 更新
  log('カードデータがサーバから更新されました');
});

/* -----------------------
   Firebase リスナー：rooms（全ルームの変更を監視）
   ----------------------- */
db.ref('rooms').on('value', (snap) => {
  rooms = snap.val() || {};
  // if in a room, and that room exists, refresh lobby/game UI
  if(currentRoom && rooms[currentRoom]){
    refreshLobby();
    refreshGameUI();
  } else {
    // if lobby visible, update listing
    if(!document.getElementById('lobbyView').classList.contains('hidden')){
      refreshLobby();
    }
  }
  log('ルームデータがサーバから更新されました');
});

/* -----------------------
   ローカル保存（バックアップ） — 必須ではないが便利
   ----------------------- */
function saveCardsLocal(arr){ localStorage.setItem('op_cards', JSON.stringify(arr)); }
function loadCardsLocal(){ const raw = localStorage.getItem('op_cards'); try{ return JSON.parse(raw); }catch(e){ return null; } }

/* -----------------------
   DOM イベント割当
   ----------------------- */
document.getElementById('btnCreate').onclick = createRoom;
document.getElementById('btnJoinExisting').onclick = ()=>{ showView('joinView'); };
document.getElementById('btnBackStart').onclick = ()=> showView('startView');
document.getElementById('btnJoin').onclick = joinRoom;
document.getElementById('btnStartGame').onclick = startGameAsOwner;
document.getElementById('btnBackLobby').onclick = ()=> { showView('lobbyView'); };
document.getElementById('btnReady').onclick = toggleReady;
document.getElementById('btnLeave').onclick = leaveRoom;
document.getElementById('btnDissolve').onclick = dissolveRoom;
document.getElementById('btnOpenAdmin').onclick = ()=> showView('adminPanel');
document.getElementById('btnShowDeck').onclick = ()=>{ const r=currentRoomObj(); if(r) alert(JSON.stringify(r.deck.map(c=>c.id).slice(0,50))); else alert('no room'); };
document.getElementById('btnExchangeDo').onclick = doExchange;
document.getElementById('btnExchangeOK').onclick = exchangeOK;
document.getElementById('btnSubmit').onclick = submitWork;
document.getElementById('btnNextGame').onclick = resetAfterGame;
document.getElementById('btnExport').onclick = exportCards;
document.getElementById('btnImport').onclick = ()=> document.getElementById('importFile').click();
document.getElementById('importFile').onchange = importCardsFromFile;

/* admin login handling */
document.getElementById('btnAdminLogin').addEventListener('click', ()=> showView('adminLogin'));
document.getElementById('adminLoginCancel').addEventListener('click', ()=> showView('startView'));
document.getElementById('adminLoginBtn').addEventListener('click', ()=> {
  const pass = document.getElementById('adminPassword').value || '';
  if(pass === ADMIN_PASSWORD){ isAdmin = true; showView('adminPanel'); renderAdminCards(); document.getElementById('adminPassword').value=''; log('管理者でログインしました'); }
  else { document.getElementById('adminLoginMsg').innerText = 'パスワードが違います'; }
});

/* -----------------------
   View 切り替え
   ----------------------- */
function showView(id){
  const views = ['startView','joinView','lobbyView','gameView','adminPanel','adminLogin'];
  views.forEach(v=>{ const el=document.getElementById(v); if(el) el.classList.add('hidden'); });
  const target = document.getElementById(id);
  if(target) target.classList.remove('hidden');
}

/* -----------------------
   Room 管理（Firebase版）
   ----------------------- */
function createRoom(){
  const code = (document.getElementById('newCode').value || '').trim();
  const max = parseInt(document.getElementById('maxPlayers').value||4);
  const creator = document.getElementById('creatorName').value || 'Owner';
  if(!/^\d{4}$/.test(code)){ alert('合言葉は4桁の数字にしてください'); return; }
  if(rooms[code]){ alert('その合言葉は既に使われています'); return; }
  // room template
  const room = {
    code,
    owner: null,
    maxPlayers: Math.min(6, Math.max(4,max)),
    players: {},
    playersOrder: [],
    state: 'waiting',
    deck: [],
    discard: [],
    turnOrder: [],
    exchangeRound: 1,
    exchangeIndex: 0,
    submitted: {},
    votes: {},
    voteCounts: {},
    phase: 'waiting',
    phaseStart: null,
    phaseDuration: 0
  };
  db.ref('rooms/' + code).set(room).then(()=>{
    joinRoomWithName(code, creator, true);
    document.getElementById('startMsg').innerText = `部屋 ${code} を作成しました。`;
    log('部屋を作成しました: ' + code);
  }).catch(err=>{ alert('作成失敗: ' + err.message); });
}

function joinRoom(){
  const code = (document.getElementById('joinCode').value || '').trim();
  const name = (document.getElementById('joinName').value || ('P'+Math.floor(Math.random()*1000)));
  if(!/^\d{4}$/.test(code)){ document.getElementById('joinMsg').innerText = '合言葉は4桁'; return; }
  if(!rooms[code]){ document.getElementById('joinMsg').innerText = 'その合言葉の部屋はありません'; return; }
  joinRoomWithName(code, name, false);
}

function joinRoomWithName(code, name, isCreator){
  const room = rooms[code];
  if(!room){ alert('その合言葉の部屋はありません（再読み込み後）'); return; }
  if(Object.keys(room.players || {}).length >= room.maxPlayers){ alert('満員です'); return; }
  const pid = uid('p');
  const player = {id:pid, nick: name, ready:false, hand:[], score:0, connected:true};
  // write to DB under rooms/{code}/players/{pid}
  const updates = {};
  updates['rooms/' + code + '/players/' + pid] = player;
  // ensure playersOrder for deterministic Act order
  updates['rooms/' + code + '/playersOrder'] = (room.playersOrder || []).concat(pid);
  if(isCreator){ updates['rooms/' + code + '/owner'] = pid; }
  db.ref().update(updates).then(()=>{
    currentRoom = code;
    actingPlayerId = pid;
    myLocalPlayerId = pid;
    document.getElementById('myDisplay').value = player.nick;
    refreshLobby();
    showView('lobbyView');
    log(`${player.nick} が部屋 ${code} に参加しました`);
  });
}

/* refresh lobby display (based on rooms[currentRoom]) */
function refreshLobby(){
  const room = currentRoomObj();
  if(!room) return;
  document.getElementById('roomCodeDisplay').innerText = room.code;
  document.getElementById('roomOwner').innerText = (room.owner? (room.players && room.players[room.owner] ? room.players[room.owner].nick : '') : '');
  // player list
  const wrap = document.getElementById('playersList'); wrap.innerHTML = '';
  const playersArray = room.playersOrder ? room.playersOrder.map(id => room.players ? room.players[id] : null).filter(Boolean) : (room.players ? Object.values(room.players) : []);
  playersArray.forEach(p=>{
    const div = document.createElement('div'); div.className='card';
    div.innerHTML = `<div style="font-weight:700">${escapeHtml(p.nick)}</div><div class="small">ready: ${p.ready? '✅': '—'}</div>`;
    if(room.owner === actingPlayerId && p.id !== room.owner){
      const b = document.createElement('button'); b.textContent='Kick'; b.onclick = ()=> { kickPlayer(p.id); };
      div.appendChild(b);
    }
    wrap.appendChild(div);
  });
  // populate actAs
  const sel = document.getElementById('actAsSelect'); sel.innerHTML='';
  playersArray.forEach(p => { const o=document.createElement('option'); o.value=p.id; o.text=p.nick; sel.appendChild(o); });
  // if actingPlayerId not in players, select first
  if(!playersArray.find(x=>x.id===actingPlayerId) && playersArray.length>0){ actingPlayerId = playersArray[0].id; }
  sel.value = actingPlayerId || '';
  sel.onchange = ()=> { actingPlayerId = sel.value; renderHandArea(); };
  // start button visible only if all ready and caller is owner
  const allReady = playersArray.length >= 4 && playersArray.every(p=>p.ready);
  document.getElementById('btnStartGame').classList.toggle('hidden', !(allReady && actingPlayerId===room.owner));
  document.getElementById('btnDissolve').classList.toggle('hidden', actingPlayerId===room.owner? false : true);
}

/* helper */
function currentRoomObj(){ return currentRoom ? rooms[currentRoom] : null; }

/* leave room */
function leaveRoom(){
  const room = currentRoomObj(); if(!room) return;
  if(!myLocalPlayerId){ currentRoom=null; actingPlayerId=null; showView('startView'); return; }
  const updates = {};
  updates['rooms/' + room.code + '/players/' + myLocalPlayerId] = null;
  updates['rooms/' + room.code + '/playersOrder'] = (room.playersOrder || []).filter(id=>id!==myLocalPlayerId);
  // if owner leaves, transfer
  if(room.owner === myLocalPlayerId){
    const remain = (room.playersOrder || []).filter(id=>id!==myLocalPlayerId);
    updates['rooms/' + room.code + '/owner'] = remain.length>0 ? remain[0] : null;
    if(remain.length===0){
      // delete room entirely
      updates['rooms/' + room.code] = null;
    }
  }
  db.ref().update(updates).then(()=>{
    currentRoom = null; actingPlayerId = null; myLocalPlayerId = null;
    showView('startView');
    log('退出しました');
  });
}

/* dissolve room */
function dissolveRoom(){
  const room = currentRoomObj(); if(!room) return;
  if(room.owner !== actingPlayerId){ alert('権限なし'); return; }
  if(!confirm('本当に部屋を解散しますか？')) return;
  db.ref('rooms/' + room.code).remove().then(()=>{
    currentRoom=null; actingPlayerId=null; showView('startView'); log('部屋を解散しました');
  });
}

/* kick player */
function kickPlayer(targetId){
  const room = currentRoomObj(); if(!room) return;
  if(room.owner !== actingPlayerId){ alert('権限なし'); return; }
  const updates = {};
  updates['rooms/' + room.code + '/players/' + targetId] = null;
  updates['rooms/' + room.code + '/playersOrder'] = (room.playersOrder || []).filter(id=>id!==targetId);
  db.ref().update(updates).then(()=> log('キックしました'));
}

/* toggle ready */
function toggleReady(){
  const room = currentRoomObj(); if(!room) return;
  const player = room.players ? room.players[actingPlayerId] : null;
  if(!player) return;
  player.ready = !player.ready;
  db.ref('rooms/' + room.code + '/players/' + actingPlayerId + '/ready').set(player.ready);
  log(`${player.nick} の準備状態: ${player.ready}`);
}

/* -----------------------
   ゲーム開始（オーナーが実行） — DB に状態を書き込む
   ----------------------- */
function startGameAsOwner(){
  const room = currentRoomObj(); if(!room) return;
  if(room.owner !== actingPlayerId){ alert('部屋主のみ開始可能'); return; }
  const playersArray = room.playersOrder ? room.playersOrder.map(id => room.players[id]) : Object.values(room.players || {});
  if(playersArray.length < 4 || !playersArray.every(p=>p.ready)){ alert('参加者が揃っていないか準備ができていません'); return; }

  // deck は cards のコピーをシャッフル
  const deck = JSON.parse(JSON.stringify(cards)).map(c=>({id:c.id, name:c.name, img:c.img}));
  shuffle(deck);
  const order = playersArray.map(p=>p.id);
  shuffle(order);

  // deal 5 each
  const playersUpdate = {};
  for(const pid of order){
    const hand = deck.splice(0,5);
    playersUpdate['rooms/' + room.code + '/players/' + pid + '/hand'] = hand;
    playersUpdate['rooms/' + room.code + '/players/' + pid + '/score'] = 0;
  }

  const updates = {};
  updates['rooms/' + room.code + '/deck'] = deck;
  updates['rooms/' + room.code + '/discard'] = [];
  updates['rooms/' + room.code + '/state'] = 'playing';
  updates['rooms/' + room.code + '/turnOrder'] = order;
  updates['rooms/' + room.code + '/exchangeRound'] = 1;
  updates['rooms/' + room.code + '/exchangeIndex'] = 0;
  updates['rooms/' + room.code + '/submitted'] = {};
  updates['rooms/' + room.code + '/votes'] = {};
  updates['rooms/' + room.code + '/voteCounts'] = {};
  updates['rooms/' + room.code + '/phase'] = 'exchange';
  // phase timing: owner sets start time & duration for timer-based sync
  updates['rooms/' + room.code + '/phaseStart'] = Date.now();
  updates['rooms/' + room.code + '/phaseDuration'] = 30; // exchange per-player duration
  // merge
  db.ref().update(Object.assign({}, updates, playersUpdate)).then(()=>{
    currentRoom = room.code;
    showView('gameView');
    refreshGameUI();
    log('ゲーム開始: 順番 ' + order.map(id=>room.players[id].nick).join(', '));
  });
}

/* -----------------------
   タイマー表示（すべてのクライアント側で計算）
   phaseStart と phaseDuration を参照して残り時間を計算する
   ----------------------- */
let globalTimerInterval = null;
function startGlobalTimerLoop(){
  if(globalTimerInterval) return;
  globalTimerInterval = setInterval(()=>{
    const room = currentRoomObj();
    if(!room) return;
    const phaseStart = room.phaseStart || null;
    const duration = room.phaseDuration || 0;
    let timeLeft = duration;
    if(phaseStart){
      const elapsed = Math.floor((Date.now() - phaseStart) / 1000);
      timeLeft = Math.max(0, duration - elapsed);
    }
    // set corresponding DOM
    if(room.phase === 'exchange') document.getElementById('exchangeTimer').innerText = timeLeft;
    if(room.phase === 'submit') document.getElementById('submitTimer').innerText = timeLeft;
    if(room.phase === 'vote') document.getElementById('voteTimer').innerText = timeLeft;
    // automatic transitions based on timeLeft and owner responsibilities:
    // Only owner will advance the game when timeouts occur to avoid conflicting writes.
    if(timeLeft === 0 && room.phase === 'exchange' && room.owner === actingPlayerId){
      // advance to next player's turn or next round or submit phase
      advanceExchangeTimeout(room);
    }
    if(timeLeft === 0 && room.phase === 'submit' && room.owner === actingPlayerId){
      autoSubmitRemaining(room);
    }
    if(timeLeft === 0 && room.phase === 'vote' && room.owner === actingPlayerId){
      finishVoteOwner(room);
    }
  }, 800);
}

/* -----------------------
   Exchange turn loop advance done by owner when timeouts
   ----------------------- */
function advanceExchangeTimeout(room){
  // move to next player: increase exchangeIndex, if beyond turnOrder then next round or submit phase
  const nextIndex = (room.exchangeIndex || 0) + 1;
  const updates = {};
  if(nextIndex >= (room.turnOrder || []).length){
    const nextRound = (room.exchangeRound || 1) + 1;
    if(nextRound > 2){
      // move to submit phase
      updates['rooms/' + room.code + '/exchangeRound'] = nextRound;
      updates['rooms/' + room.code + '/exchangeIndex'] = 0;
      updates['rooms/' + room.code + '/phase'] = 'submit';
      updates['rooms/' + room.code + '/phaseStart'] = Date.now();
      updates['rooms/' + room.code + '/phaseDuration'] = 100; // submit duration
      // prepare submitted entries for players who haven't submitted
      const sub = {};
      for(const pid of room.turnOrder){
        const p = room.players ? room.players[pid] : null;
        sub[pid] = {order: p && p.hand ? p.hand.map(c=>c.id) : [], roleName: '', submitted: false};
      }
      updates['rooms/' + room.code + '/submitted'] = sub;
    } else {
      updates['rooms/' + room.code + '/exchangeRound'] = nextRound;
      updates['rooms/' + room.code + '/exchangeIndex'] = 0;
      // reset phaseStart to new player
      updates['rooms/' + room.code + '/phaseStart'] = Date.now();
      updates['rooms/' + room.code + '/phaseDuration'] = 30;
    }
  } else {
    updates['rooms/' + room.code + '/exchangeIndex'] = nextIndex;
    updates['rooms/' + room.code + '/phaseStart'] = Date.now();
    updates['rooms/' + room.code + '/phaseDuration'] = 30;
  }
  db.ref().update(updates).then(()=> log('交換フェーズのタイムアウトで進行しました'));
}

/* owner auto-submit remaining */
function autoSubmitRemaining(room){
  const updates = {};
  const submitted = room.submitted || {};
  for(const pid of room.turnOrder || []){
    if(!submitted[pid] || !submitted[pid].submitted){
      const p = room.players ? room.players[pid] : null;
      submitted[pid] = {order: p && p.hand ? p.hand.map(c=>c.id) : [], roleName: '', submitted: true};
    }
  }
  updates['rooms/' + room.code + '/submitted'] = submitted;
  updates['rooms/' + room.code + '/phase'] = 'vote';
  updates['rooms/' + room.code + '/phaseStart'] = Date.now();
  updates['rooms/' + room.code + '/phaseDuration'] = 120;
  updates['rooms/' + room.code + '/votes'] = {};
  updates['rooms/' + room.code + '/voteCounts'] = {};
  db.ref().update(updates).then(()=> log('未提出を自動提出し、投票フェーズへ移行しました'));
}

/* owner finish vote if timeouts */
function finishVoteOwner(room){
  // tally and set state to waiting and record results in room.result
  const voteCounts = room.voteCounts || {};
  for(const pid of room.turnOrder || []) voteCounts[pid] = voteCounts[pid] || 0;
  let max = -1;
  for(const pid of room.turnOrder || []) if(voteCounts[pid] > max) max = voteCounts[pid];
  const winners = (room.turnOrder || []).filter(pid => voteCounts[pid] === max);
  const updates = {};
  updates['rooms/' + room.code + '/result'] = { voteCounts, winners, finishedAt: Date.now() };
  updates['rooms/' + room.code + '/state'] = 'waiting';
  updates['rooms/' + room.code + '/phase'] = 'waiting';
  db.ref().update(updates).then(()=> log('投票終了。結果を保存しました'));
}

/* -----------------------
   UI: render hand / submit / vote / result for acting player
   ----------------------- */
function renderHandArea(){
  const room = currentRoomObj(); if(!room) return;
  const player = room.players ? room.players[actingPlayerId] : null;
  const wrap = document.getElementById('handArea'); wrap.innerHTML = '';
  if(!player) return;
  player.hand = player.hand || [];
  player.hand.forEach((c, idx)=>{
    const div = document.createElement('div'); div.className='card'; div.dataset.index = idx;
    const img = document.createElement('img'); img.src = c.img || placeholderForId(c.id);
    const nm = document.createElement('div'); nm.textContent = c.name;
    div.appendChild(img); div.appendChild(nm);
    div.onclick = ()=> { div.classList.toggle('selected'); };
    wrap.appendChild(div);
  });
  renderSubmitHand();
}

/* placeholder image */
function placeholderForId(id){
  const svg = `<svg xmlns='http://www.w3.org/2000/svg' width='200' height='300'><rect width='100%' height='100%' fill='#eee'/><text x='50%' y='50%' dominant-baseline='middle' text-anchor='middle' fill='#666' font-size='18'>${id}</text></svg>`;
  return 'data:image/svg+xml;base64,' + btoa(svg);
}

/* doExchange: owner will update deck/players accordingly */
function doExchange(){
  const room = currentRoomObj(); if(!room) return;
  const pid = room.turnOrder ? room.turnOrder[room.exchangeIndex] : null;
  if(pid !== actingPlayerId){ alert('今はこのプレイヤーの番ではありません。Act as を切り替えてください'); return; }
  const player = room.players ? room.players.find ? room.players.find(p=>p.id===pid) : room.players[pid] : null;
  // But our room structure uses players map; adapt:
  const p = room.players ? room.players[pid] : null;
  if(!p) return;
  const selEls = Array.from(document.getElementById('handArea').querySelectorAll('.selected'));
  if(selEls.length === 0){ alert('交換したいカードを選択してください（0-5枚）'); return; }
  const idxs = selEls.map(el=>parseInt(el.dataset.index)).sort((a,b)=>b-a);
  const discarded = [];
  for(const idx of idxs){
    if(idx>=0 && idx<p.hand.length) discarded.push(p.hand.splice(idx,1)[0]);
  }
  // Update DB: push discarded to deck bottom, draw same number
  const updates = {};
  const newDeck = (room.deck || []).concat(discarded);
  const draw = newDeck.splice(0, discarded.length);
  updates['rooms/' + room.code + '/deck'] = newDeck;
  updates['rooms/' + room.code + '/players/' + pid + '/hand'] = p.hand.concat(draw);
  // advance exchangeIndex and reset timer
  const nextIndex = (room.exchangeIndex || 0) + 1;
  updates['rooms/' + room.code + '/exchangeIndex'] = nextIndex;
  updates['rooms/' + room.code + '/phaseStart'] = Date.now();
  updates['rooms/' + room.code + '/phaseDuration'] = 30;
  db.ref().update(updates).then(()=> log(`${p.nick} が ${discarded.length} 枚交換しました`));
}

/* exchangeOK (no exchange) */
function exchangeOK(){
  const room = currentRoomObj(); if(!room) return;
  const pid = room.turnOrder ? room.turnOrder[room.exchangeIndex] : null;
  if(pid !== actingPlayerId){ alert('今はこのプレイヤーの番ではありません'); return; }
  const updates = {};
  const nextIndex = (room.exchangeIndex || 0) + 1;
  updates['rooms/' + room.code + '/exchangeIndex'] = nextIndex;
  updates['rooms/' + room.code + '/phaseStart'] = Date.now();
  updates['rooms/' + room.code + '/phaseDuration'] = 30;
  db.ref().update(updates).then(()=> log(`${room.players[pid].nick} がOKしました`));
}

/* -----------------------
   Submit Phase
   ----------------------- */
function startSubmitPhase_localTrigger(){
  // owner can trigger submit phase early
  const room = currentRoomObj(); if(!room) return;
  if(room.owner !== actingPlayerId) return;
  const updates = {};
  updates['rooms/' + room.code + '/phase'] = 'submit';
  updates['rooms/' + room.code + '/phaseStart'] = Date.now();
  updates['rooms/' + room.code + '/phaseDuration'] = 100;
  // ensure submitted entries exist
  const sub = {};
  for(const pid of room.turnOrder || []){
    const p = room.players ? room.players[pid] : null;
    sub[pid] = {order: p && p.hand ? p.hand.map(c=>c.id) : [], roleName:'', submitted:false};
  }
  updates['rooms/' + room.code + '/submitted'] = sub;
  db.ref().update(updates).then(()=> log('提出フェーズを開始しました'));
}

/* renderSubmitHand for acting player (draggable) */
function renderSubmitHand(){
  const container = document.getElementById('submitHand'); container.innerHTML = '';
  const room = currentRoomObj(); if(!room) return;
  const player = room.players ? room.players[actingPlayerId] : null;
  if(!player) return;
  player.hand.forEach((c, idx)=>{
    const div = document.createElement('div'); div.className='card'; div.dataset.id=c.id;
    const img = document.createElement('img'); img.src = c.img || placeholderForId(c.id);
    const nm = document.createElement('div'); nm.textContent = c.name;
    div.appendChild(img); div.appendChild(nm);
    container.appendChild(div);
  });
  makeSortable(container);
}

/* lightweight sortable */
function makeSortable(container){
  let dragEl = null;
  Array.from(container.children).forEach(ch=>{
    ch.draggable = true;
    ch.ondragstart = (e)=>{ dragEl = ch; e.dataTransfer.setData('text/plain',''); ch.style.opacity = .4; };
    ch.ondragend = ()=>{ if(dragEl) dragEl.style.opacity = 1; dragEl = null; };
    ch.ondragover = (e)=>{ e.preventDefault(); };
    ch.ondrop = (e)=>{ e.preventDefault(); if(!dragEl || dragEl===ch) return; container.insertBefore(dragEl, ch.nextSibling); };
  });
}

/* submitWork: write to room.submitted/{pid} */
function submitWork(){
  const room = currentRoomObj(); if(!room) return;
  if(room.phase !== 'submit'){ alert('今は提出フェーズではありません'); return; }
  const nodes = Array.from(document.getElementById('submitHand').children);
  if(nodes.length !== 5){ alert('5枚を並べてください'); return; }
  const order = nodes.map(n=>n.dataset.id);
  const roleName = document.getElementById('roleInput').value.trim();
  const data = {order, roleName, submitted:true};
  db.ref('rooms/' + room.code + '/submitted/' + actingPlayerId).set(data).then(()=> {
    log(`${room.players[actingPlayerId].nick} が提出しました: ${roleName || '(無題)'}`);
    // check all submitted and if owner, move to vote
    setTimeout(()=> checkAllSubmittedAndMaybeAdvance(room), 500);
  });
}

function checkAllSubmittedAndMaybeAdvance(room){
  room = rooms[room.code]; // refresh from cached
  const all = (room.turnOrder || []).every(pid => {
    return room.submitted && room.submitted[pid] && room.submitted[pid].submitted;
  });
  if(all && room.owner && room.owner === actingPlayerId){
    // owner transitions to vote
    const updates = {};
    updates['rooms/' + room.code + '/phase'] = 'vote';
    updates['rooms/' + room.code + '/phaseStart'] = Date.now();
    updates['rooms/' + room.code + '/phaseDuration'] = 120;
    updates['rooms/' + room.code + '/votes'] = {};
    updates['rooms/' + room.code + '/voteCounts'] = {};
    db.ref().update(updates).then(()=> log('全員提出済。投票フェーズへ移行しました'));
  }
}

/* -----------------------
   Vote Phase: rendering and voting
   ----------------------- */
function renderSubmissions(){
  const room = currentRoomObj(); if(!room) return;
  const container = document.getElementById('submissionsArea'); container.innerHTML = '';
  for(const pid of room.turnOrder || []){
    const sub = (room.submitted && room.submitted[pid]) ? room.submitted[pid] : {order: room.players && room.players[pid] ? room.players[pid].hand.map(c=>c.id) : [], roleName:''};
    const box = document.createElement('div'); box.className='card';
    box.innerHTML = `<div style="font-weight:700">${escapeHtml(room.players[pid].nick)}</div><div class="small">${escapeHtml(sub.roleName)}</div>`;
    const row = document.createElement('div'); row.style.display='flex'; row.style.gap='6px'; row.style.marginTop='6px';
    sub.order.forEach(cid=>{
      const cc = cards.find(x=>x.id===cid) || {id:cid,name:cid,img:null};
      const thumb = document.createElement('img'); thumb.src = cc.img || placeholderForId(cc.id); thumb.style.width='46px'; thumb.style.height='68px'; thumb.style.objectFit='cover';
      row.appendChild(thumb);
    });
    box.appendChild(row);
    if(actingPlayerId !== pid && !(room.votes && room.votes[actingPlayerId])){
      const voteBtn = document.createElement('button'); voteBtn.textContent='投票'; voteBtn.onclick = ()=>{
        const updates = {};
        updates['rooms/' + room.code + '/votes/' + actingPlayerId] = pid;
        updates['rooms/' + room.code + '/voteCounts/' + pid] = ((room.voteCounts && room.voteCounts[pid])||0) + 1;
        db.ref().update(updates).then(()=> {
          log(`${room.players[actingPlayerId].nick} が ${room.players[pid].nick} に投票しました`);
        });
      };
      box.appendChild(voteBtn);
    } else {
      const note = document.createElement('div'); note.className='small'; note.innerText = actingPlayerId === pid ? 'あなたの提出' : (room.votes && room.votes[actingPlayerId] ? '投票済' : '');
      box.appendChild(note);
    }
    container.appendChild(box);
  }
}

/* finishVote invoked by owner (if all voted or timeout) */
function finishVote(){
  const room = currentRoomObj(); if(!room) return;
  if(room.owner !== actingPlayerId){ alert('オーナーのみ処理可能'); return; }
  // compute if all voted
  const allVoted = (room.turnOrder || []).every(pid => {
    return room.votes && Object.values(room.votes).includes(pid);
  });
  if(!allVoted && !confirm('全員が投票していません。強制終了しますか？')) return;
  finishVoteOwner(room);
}

/* -----------------------
   Reset after game
   ----------------------- */
function resetAfterGame(){
  const room = currentRoomObj();
  if(!room) return;
  const updates = {};
  // clear hands, submitted, votes; set ready=false
  for(const pid of room.turnOrder || []){
    updates['rooms/' + room.code + '/players/' + pid + '/hand'] = [];
    updates['rooms/' + room.code + '/players/' + pid + '/ready'] = false;
    updates['rooms/' + room.code + '/players/' + pid + '/score'] = 0;
  }
  updates['rooms/' + room.code + '/deck'] = [];
  updates['rooms/' + room.code + '/discard'] = [];
  updates['rooms/' + room.code + '/turnOrder'] = [];
  updates['rooms/' + room.code + '/exchangeRound'] = 1;
  updates['rooms/' + room.code + '/exchangeIndex'] = 0;
  updates['rooms/' + room.code + '/submitted'] = {};
  updates['rooms/' + room.code + '/votes'] = {};
  updates['rooms/' + room.code + '/voteCounts'] = {};
  updates['rooms/' + room.code + '/phase'] = 'waiting';
  updates['rooms/' + room.code + '/state'] = 'waiting';
  db.ref().update(updates).then(()=> {
    document.getElementById('resultPanel').classList.add('hidden');
    showView('lobbyView');
    refreshLobby();
    log('次のゲームのためにリセットしました');
  });
}

/* -----------------------
   Admin: render / edit cards (images as DataURL)
   ----------------------- */
function renderAdminCards(){
  const wrap = document.getElementById('adminCards'); wrap.innerHTML = '';
  const search = (document.getElementById('adminSearch')||{value:''}).value.toLowerCase();
  for(const c of cards || []){
    if(search && !(c.id.includes(search) || c.name.toLowerCase().includes(search))) continue;
    const box = document.createElement('div'); box.className='card';
    const img = document.createElement('img'); img.src = c.img || placeholderForId(c.id); img.style.height='90px';
    const name = document.createElement('input'); name.value = c.name; name.onchange = ()=> { c.name = name.value; saveCardsToServer(cards); };
    const file = document.createElement('input'); file.type='file'; file.accept='image/*'; file.onchange = (ev)=> {
      const f = ev.target.files[0];
      const r = new FileReader();
      r.onload = (e)=> { c.img = e.target.result; saveCardsToServer(cards); };
      if(f) r.readAsDataURL(f);
    };
    box.appendChild(img); box.appendChild(name); box.appendChild(file);
    wrap.appendChild(box);
  }
}

/* save whole cards array to server (DB) */
function saveCardsToServer(arr){
  db.ref('cards').set(arr).then(()=> log('カードを保存しました（DB）'));
}

/* export / import (JSON) */
function exportCards(){
  const data = JSON.stringify(cards);
  const blob = new Blob([data], {type:'application/json'});
  const url = URL.createObjectURL(blob);
  const a = document.createElement('a'); a.href = url; a.download = 'cards.json'; a.click();
  URL.revokeObjectURL(url);
}
function importCardsFromFile(e){
  const f = e.target.files[0]; if(!f) return;
  const r = new FileReader();
  r.onload = (ev)=> {
    try {
      const parsed = JSON.parse(ev.target.result);
      if(Array.isArray(parsed) && parsed.length>0 && parsed[0].id){
        saveCardsToServer(parsed);
        alert('インポート完了');
      } else alert('不正なファイル');
    } catch(err){ alert('読み込み失敗'); }
  };
  r.readAsText(f);
}

/* -----------------------
   init
   ----------------------- */
(function init(){
  // load local cache if any
  const lc = loadCardsLocal();
  if(lc) cards = lc;
  // ensure DB listeners (already set above)
  // initial view
  showView('startView');
  startGlobalTimerLoop();
})();

/* -----------------------
   refreshGameUI: reflect DB room state into DOM
   ----------------------- */
function refreshGameUI(){
  const room = currentRoomObj();
  if(!room) return;
  document.getElementById('phaseLabel').innerText = room.phase || 'playing';
  document.getElementById('orderDisplay').innerText = (room.turnOrder || []).map(id=> (room.players && room.players[id]?room.players[id].nick:'')).join(' → ');
  document.getElementById('currentTurnDisplay').innerText = (room.turnOrder && room.turnOrder.length>0 && typeof room.exchangeIndex !== 'undefined') ? (room.players && room.players[room.turnOrder[room.exchangeIndex]] ? room.players[room.turnOrder[room.exchangeIndex]].nick : '—') : '—';
  document.getElementById('deckCount').innerText = (room.deck || []).length;
  document.getElementById('discardCount').innerText = (room.discard || []).length;
  document.getElementById('exchangeRoundDisplay').innerText = room.exchangeRound || 1;
  document.getElementById('exchangeIndexDisplay').innerText = ((room.exchangeIndex||0)+1) + '/' + ((room.turnOrder && room.turnOrder.length) || 0);
  // toggle panels per phase
  document.getElementById('exchangePanel').classList.toggle('hidden', room.phase !== 'exchange');
  document.getElementById('submitPanel').classList.toggle('hidden', room.phase !== 'submit');
  document.getElementById('votePanel').classList.toggle('hidden', room.phase !== 'vote');
  document.getElementById('resultPanel').classList.toggle('hidden', !(room.result && room.result.winners));
  // render acting player's hand
  renderHandArea();
  // render submissions/vote if in those phases
  if(room.phase === 'vote') renderSubmissions();
  if(room.result && room.result.winners){
    // show result
    let html = '<div class="small">得票数</div>';
    for(const pid of room.turnOrder || []){
      html += `<div>${escapeHtml(room.players && room.players[pid] ? room.players[pid].nick : pid)}: ${(room.result.voteCounts && room.result.voteCounts[pid])||0}</div>`;
    }
    html += `<div style="margin-top:8px;"><strong>勝者: ${room.result.winners.map(w=>escapeHtml(room.players && room.players[w] ? room.players[w].nick : w)).join(', ')}</strong></div>`;
    document.getElementById('resultDisplay').innerHTML = html;
  }
}

/* -----------------------
   utility: ensure cards variable persisted when edited in admin
   ----------------------- */
function saveCards(arr){ saveCardsToServer(arr); }

/* -----------------------
   helpers for owner actions when timeouts triggered automatically
   (advanceExchangeTimeout / autoSubmitRemaining / finishVoteOwner already implemented)
   ----------------------- */

/* end of script */
</script>
</body>
</html>

