<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="utf-8">
<meta name="viewport" content="width=device-width, initial-scale=1, viewport-fit=cover">
<title>Bluff – Card Game</title>
<!-- PeerJS = browser-to-browser (WebRTC) connections, no server of your own needed. Pinned version. -->
<script src="https://cdn.jsdelivr.net/npm/peerjs@1.5.4/dist/peerjs.min.js"></script>
<style>
  :root { --bg:#f6f5f1; --fg:#1d1d1b; --card:#fff; --muted:#6b6a65; --line:#dcdad2; color-scheme:light; }
  @media (prefers-color-scheme: dark) {
    :root { --bg:#161614; --fg:#f1efe8; --card:#232321; --muted:#a3a198; --line:#3a3936; color-scheme:dark; }
  }
  * { box-sizing:border-box; -webkit-tap-highlight-color:transparent; user-select:none; touch-action:manipulation; }
  html, body { margin:0; min-height:100%; }
  body { background:var(--bg); color:var(--fg); font-family:system-ui,-apple-system,"Segoe UI",sans-serif;
         max-width:560px; margin:0 auto; padding:12px; padding-top:calc(12px + env(safe-area-inset-top,0px));
         padding-bottom:calc(12px + env(safe-area-inset-bottom,0px)); }
  .screen { display:none; flex-direction:column; gap:12px; } .screen.on { display:flex; }
  .box { background:var(--card); border:2px solid var(--line); border-radius:16px; padding:14px; }
  h1 { font-size:26px; margin:8px 0 0; text-align:center; }
  p.muted, .muted { color:var(--muted); text-align:center; margin:0; font-size:14px; }
  input { width:100%; font-size:20px; padding:12px; border-radius:10px; border:2px solid var(--line);
          background:var(--bg); color:var(--fg); user-select:text; }
  #code { text-align:center; letter-spacing:.2em; text-transform:uppercase; }
  button.big { width:100%; font-size:19px; font-weight:700; padding:14px; border-radius:12px; border:none;
               background:var(--fg); color:var(--bg); cursor:pointer; margin-top:8px; }
  button.big:disabled { opacity:.3; } button.alt { background:#d94a3d; color:#fff; }
  #err { color:#d94a3d; text-align:center; min-height:22px; font-weight:600; }
  #roomCode { font-size:48px; font-weight:800; letter-spacing:.18em; text-align:center; }
  .pl { padding:8px 0; font-size:20px; border-bottom:1px solid var(--line); }
  #rules { font-size:14px; line-height:1.45; color:var(--muted); }
  /* game */
  #strip { display:flex; gap:8px; flex-wrap:wrap; }
  .pc { flex:1 1 28%; min-width:92px; text-align:center; border:3px solid var(--line); border-radius:12px; padding:6px; background:var(--card); }
  .pc span { display:block; font-size:13px; overflow:hidden; text-overflow:ellipsis; white-space:nowrap; }
  .pc b { font-size:26px; } .pc small { display:block; color:var(--muted); }
  .pc.turn { background:var(--fg); color:var(--bg); } .pc.turn small { color:var(--bg); }
  #table { text-align:center; display:flex; flex-direction:column; gap:8px; min-height:170px; justify-content:center; }
  #rank { font-size:34px; font-weight:800; } #pile { font-size:18px; } #log { font-size:15px; }
  .row { display:flex; gap:5px; justify-content:center; flex-wrap:wrap; }
  .cd { min-width:42px; height:58px; border-radius:8px; border:2px solid var(--line); background:var(--card);
        font-size:18px; font-weight:700; color:var(--fg); padding:0 4px; display:inline-flex; align-items:center; justify-content:center; }
  .cd.red { color:#d94a3d; } .cd.sel { transform:translateY(-10px); border-color:var(--fg); box-shadow:0 3px 0 var(--fg); }
  #hand { gap:6px; padding-top:10px; }
  #actions { display:flex; flex-direction:column; gap:4px; }
  #status { text-align:center; color:var(--muted); font-size:15px; min-height:20px; }
  #net { position:fixed; left:0; right:0; top:0; z-index:9; background:#d94a3d; color:#fff; text-align:center; font-weight:700;
         font-size:14px; padding:calc(6px + env(safe-area-inset-top,0px)) 10px 6px; display:none; }
  #net button { margin-left:8px; font-size:13px; padding:2px 8px; border-radius:6px; border:1px solid #fff; background:transparent; color:#fff; }
  .off { opacity:.45; }
  #leaveBtn { background:transparent; color:var(--muted); border:1px solid var(--line); font-size:14px; padding:8px; }
</style>
</head>
<body>
<div id="net"><span id="netTxt"></span><button id="netLeave">Leave</button></div>

<!-- ============ HOME ============ -->
<div class="screen on" id="home">
  <h1>🃏 Bluff</h1>
  <p class="muted">The card game of lying with a straight face</p>
  <div class="box"><input id="name" placeholder="Your name" maxlength="12" autocomplete="off">
    <button class="big" id="createBtn">Create room</button></div>
  <div class="box"><input id="code" placeholder="ROOM CODE" maxlength="4" autocomplete="off">
    <button class="big" id="joinBtn">Join room</button></div>
  <div id="err"></div>
</div>

<!-- ============ LOBBY ============ -->
<div class="screen" id="lobby">
  <h1>Room code</h1><div id="roomCode">----</div>
  <p class="muted">Share this code with everyone who is playing</p>
  <div class="box" id="plist"></div>
  <button class="big" id="startBtn" disabled>Deal the cards</button>
  <button class="big" id="kickBtn" style="display:none;background:transparent;color:var(--muted);border:1px solid var(--line);font-size:15px;padding:10px">Remove offline players</button>
  <p class="muted" id="lobbyNote"></p>
  <div class="box" id="rules"><b>Rules.</b> All the cards are dealt (a second deck is added above 8 players). Ranks go A, 2, 3 … K, A … in order, one rank per turn.
    On your turn put cards face down and say they are all of the required rank. You may lie.
    If the pile is empty (start of the game, or right after a Bluff call) you must play <b>at least 2 cards</b>; otherwise you can play <b>1 or as many as you like</b>.
    Then each opponent either <b>calls Bluff</b> or lets it pass. If you lied, you pick up the whole pile.
    If you told the truth, the caller picks it up. First player with no cards left wins.</div>
</div>

<!-- ============ GAME ============ -->
<div class="screen" id="game">
  <div id="strip"></div>
  <div class="box" id="table">
    <div class="muted">Required rank</div><div id="rank">-</div>
    <div id="pile"></div><div class="row" id="reveal"></div><div id="log"></div>
  </div>
  <div class="row" id="hand"></div>
  <div id="actions"><div id="status"></div>
    <button class="big" id="playBtn" style="display:none">Play</button>
    <button class="big alt" id="callBtn" style="display:none">Call BLUFF!</button>
    <button class="big" id="passBtn" style="display:none">Let it pass</button>
    <button class="big" id="againBtn" style="display:none">Play again</button>
    <button class="big" id="kickBtn2" style="display:none;background:transparent;color:var(--muted);border:1px solid var(--line);font-size:15px;padding:10px">Remove offline players</button>
    <button class="big" id="leaveBtn">Leave room</button></div>
</div>

<script>
/* =====================================================================
   BLUFF – ONLINE, 2 TO MAX_PLAYERS PLAYERS
   Networking: the HOST's phone is the referee and the only place the full
   game state lives. Each player is sent ONLY their own hand, so nobody can
   peek at the others' cards. Host creates PeerJS id "tws-bluff-<CODE>".
   Cards are numbers: rank = id % 13 (0=Ace..12=King), suit = floor((id % 52)/13),
   deck = floor(id / 52)  (a 2nd deck is added when there are more than 8 players).
   ===================================================================== */
const MAX_PLAYERS = 16;          // change this number if you ever need a bigger room
const MIN_PLAYERS = 2;
const COLORS = ['#d94a3d','#2f9e5b','#2f6fd9','#e0a21b','#8a4fd1','#17a2b8','#e0609c','#6b8e23','#c0651e','#4a5568','#0e8a7d','#a83279','#5b7de8','#b5892e','#7a3b2e','#3d9970'];
const RK = ['A','2','3','4','5','6','7','8','9','10','J','Q','K'];
const RW = ['Aces','2s','3s','4s','5s','6s','7s','8s','9s','10s','Jacks','Queens','Kings'];
const SU = ['♠','♥','♦','♣'];
const $ = id => document.getElementById(id);
const esc = s => String(s).replace(/[&<>"]/g, c => ({'&':'&amp;','<':'&lt;','>':'&gt;','"':'&quot;'}[c]));
const show = id => document.querySelectorAll('.screen').forEach(s => s.classList.toggle('on', s.id === id));
const err = t => { $('err').textContent = t; show('home'); };
const cardLabel = id => RK[id % 13] + SU[Math.floor((id % 52) / 13)];
const isRed = id => { const s = Math.floor((id % 52) / 13); return s === 1 || s === 2; };

// ---------- shared state ----------
let peer, isHost = false, me = 0, myName = '', hostConn = null;
let conns = [null];                      // host only: connection per seat (seat 0 = host)
let tokens = [''];                       // host only: secret id per seat so a player can rejoin after a drop
let lastSeen = [0];                      // host only: last time we heard from each seat
let names = [''];
let G = null;                            // latest personal view of the game (every player)
let sel = new Set();                     // card ids I have selected to play
let lobbyOnline = [true];
// ---------- host-only game state ----------
let S = null, inGame = false, hostWins = [0];

// ---------- session (lets a phone rejoin after lock / reload / signal loss) ----------
const SKEY = 'bluff-session';
let myToken = (() => { try { return localStorage.getItem('bluff-token') || (() => { const t = Math.random().toString(36).slice(2) + Date.now().toString(36); localStorage.setItem('bluff-token', t); return t; })(); } catch (e) { return Math.random().toString(36).slice(2); } })();
let curCode = '', opened = false, joined = false, resuming = false, hostRetries = 0, lastHostMsg = 0, rejoinTimer = null, wl = null;
const saveSession = extra => { try { localStorage.setItem(SKEY, JSON.stringify(Object.assign({ role: isHost ? 'host' : 'guest', code: curCode, name: myName, token: myToken, ts: Date.now() }, extra || {}))); } catch (e) {} };
const loadSession = () => { try { const x = JSON.parse(localStorage.getItem(SKEY)); return x && Date.now() - x.ts < 12 * 3600e3 ? x : null; } catch (e) { return null; } };
const clearSession = () => { try { localStorage.removeItem(SKEY); } catch (e) {} };
const saveHost = () => saveSession({ S, names, tokens, hostWins, inGame });
const banner = t => { $('net').style.display = t ? 'block' : 'none'; $('netTxt').textContent = t || ''; };
function leave() { clearSession(); try { peer && peer.destroy(); } catch (e) {} location.reload(); }
async function keepAwake() { try { if ('wakeLock' in navigator && document.visibilityState === 'visible' && !(wl && !wl.released)) wl = await navigator.wakeLock.request('screen'); } catch (e) {} }

// ======================= CONNECTION =======================
const randCode = () => { const a = 'ABCDEFGHJKLMNPQRSTUVWXYZ23456789'; let s = '';
  for (let i = 0; i < 4; i++) s += a[Math.floor(Math.random() * a.length)]; return s; };
const getName = () => $('name').value.trim() || 'Player';
const online = () => names.map((_, i) => i === 0 || !!(conns[i] && conns[i].open));
function toHost(m) { if (isHost) onHostMsg(0, m); else if (hostConn && hostConn.open) hostConn.send(m); else banner('Reconnecting…'); }
function sendLobby() {
  const on = online();
  conns.forEach((c, i) => c && c.open && c.send({ t: 'lobby', names, you: i, online: on }));
  onMsg({ t: 'lobby', names, you: 0, online: on }); saveHost();
}
function sendAll() { if (S) { conns.forEach((c, i) => c && c.open && S.hands[i] && c.send(view(i))); if (S.hands[0]) onMsg(view(0)); } saveHost(); }
function removeSeat(i) { names.splice(i, 1); tokens.splice(i, 1); conns.splice(i, 1); lastSeen.splice(i, 1); hostWins.splice(i, 1); }

// ----- HOST -----
function createRoom() { myName = getName(); hostStart(randCode(), false, null); }
function hostStart(code, restoring, saved) {
  if (typeof Peer === 'undefined') return err('PeerJS library did not load (check internet / blocked site)');
  try { peer && peer.destroy(); } catch (e) {}
  curCode = code; opened = false; isHost = true; me = 0;
  if (saved && saved.names) { myName = saved.name; S = saved.S; names = saved.names; tokens = saved.tokens; hostWins = saved.hostWins; inGame = !!saved.inGame;
    conns = names.map(() => null); lastSeen = names.map(() => Date.now()); }
  peer = new Peer('tws-bluff-' + code);
  peer.on('open', () => {
    opened = true; hostRetries = 0; banner('');
    if (!S && !(saved && saved.names)) { names = [myName]; tokens = [myToken]; conns = [null]; lastSeen = [0]; hostWins = [0]; }
    $('roomCode').textContent = code; keepAwake();
    if (S && S.hands[0]) onMsg(view(0)); else show('lobby');
    sendLobby(); sendAll();
  });
  peer.on('disconnected', () => banner('Reconnecting to server…'));
  peer.on('error', e => {
    if (e.type === 'unavailable-id') {
      if (restoring && hostRetries++ < 15) { banner('Restoring room…'); setTimeout(() => hostStart(code, true, saved), 2000); }
      else if (restoring) { clearSession(); err('Could not restore the room'); }
      else hostStart(randCode(), false, null);
    } else if (!opened) err('Connection problem: ' + e.type);
    else banner('Connection problem… retrying');
  });
  peer.on('connection', c => {
    c.on('data', m => {
      if (m.t === 'hello') {
        const nm = (m.name || '').slice(0, 12) || 'Player', tk = String(m.token || '');
        let idx = tokens.findIndex((t, i) => i > 0 && t === tk);                 // returning player keeps their seat
        if (idx < 0) {                                                           // new player
          if (inGame) { c.send({ t: 'busy' }); setTimeout(() => c.close(), 300); return; }
          if (names.length >= MAX_PLAYERS) { c.send({ t: 'full' }); setTimeout(() => c.close(), 300); return; }
          idx = names.length; names.push(nm); tokens.push(tk); conns.push(null); lastSeen.push(0); hostWins.push(0);
        }
        const old = conns[idx]; conns[idx] = c; names[idx] = nm; lastSeen[idx] = Date.now();
        if (old && old !== c) { try { old.close(); } catch (e) {} }
        sendLobby();
        if (S && S.hands[idx]) sendAll();                                        // game running: everyone sees who is back
        return;
      }
      const idx = conns.indexOf(c); if (idx < 1) return;
      lastSeen[idx] = Date.now();
      if (m.t === 'pong') return;
      onHostMsg(idx, m);
    });
    c.on('close', () => dropConn(c));
    c.on('error', () => dropConn(c));
  });
}
function dropConn(c) {
  const idx = conns.indexOf(c); if (idx < 1) return;
  conns[idx] = null;
  if (!S) { removeSeat(idx); sendLobby(); }               // lobby: free the seat (they auto-rejoin)
  else { sendLobby(); sendAll(); }                        // game: keep the seat, show offline
}
// host: drop everybody who is offline (so a game can start / go on without them)
function kickOffline() {
  if (!isHost) return;
  for (let i = names.length - 1; i > 0; i--) if (!(conns[i] && conns[i].open)) removeSeat(i);
  if (S) { S = null; inGame = false; conns.forEach(c => c && c.open && c.send({ t: 'toLobby' })); G = null; sel.clear(); show('lobby'); }
  sendLobby();
}

// ----- GUEST -----
function joinRoom(resumeCode) {
  myName = getName(); const code = (typeof resumeCode === 'string' ? resumeCode : $('code').value.trim()).toUpperCase();
  if (code.length !== 4) return err('Enter the 4-letter room code');
  if (typeof Peer === 'undefined') return err('PeerJS library did not load (check internet / blocked site)');
  isHost = false; curCode = code; joined = false; saveSession(); connectToHost();
}
function connectToHost() {
  clearTimeout(rejoinTimer);
  try { peer && peer.destroy(); } catch (e) {}
  try { hostConn && hostConn.close(); } catch (e) {}
  if (joined) banner('Connection lost - reconnecting…');
  let dead = false;
  const retry = () => { if (dead || isHost) return; dead = true; if (!joined && !loadSession()) return; banner(joined ? 'Connection lost - reconnecting…' : 'Joining room…'); rejoinTimer = setTimeout(connectToHost, 2000); };
  peer = new Peer();
  peer.on('error', e => {
    if (e.type === 'peer-unavailable' && !joined && !resuming) { clearSession(); return err('Room not found'); }
    retry();
  });
  peer.on('disconnected', () => { if (!peer.destroyed) try { peer.reconnect(); } catch (e) {} });
  peer.on('open', () => {
    hostConn = peer.connect('tws-bluff-' + curCode, { reliable: true });
    hostConn.on('open', () => { lastHostMsg = Date.now(); hostConn.send({ t: 'hello', name: myName, token: myToken }); });
    hostConn.on('data', m => {
      lastHostMsg = Date.now();
      if (m.t === 'ping') { hostConn.send({ t: 'pong' }); return; }
      if (m.t === 'full') { clearSession(); dead = true; return err('Room is full'); }
      if (m.t === 'busy') { clearSession(); dead = true; return err('That game has already started'); }
      if (m.t === 'lobby' || m.t === 'state') { joined = true; resuming = false; banner(''); keepAwake(); }
      onMsg(m);
    });
    hostConn.on('close', retry); hostConn.on('error', retry);
  });
}

// ----- keep-alive for both sides -----
setInterval(() => {
  const now = Date.now();
  if (isHost) {
    conns.forEach((c, i) => {
      if (!c) return;
      if (c.open) c.send({ t: 'ping' });
      if (now - lastSeen[i] > 15000) { try { c.close(); } catch (e) {} dropConn(c); }   // silent for 15 s = gone
    });
    if (peer && peer.destroyed && curCode) hostStart(curCode, true, loadSession());
    else if (peer && peer.disconnected && !peer.destroyed) { try { peer.reconnect(); } catch (e) {} }
  } else if (curCode && (joined || resuming)) {
    if (!hostConn || !hostConn.open || now - lastHostMsg > 12000) connectToHost();
  }
}, 4000);
function wake() {
  if (document.visibilityState !== 'visible') return;
  keepAwake();
  if (isHost) { if (peer && peer.disconnected && !peer.destroyed) try { peer.reconnect(); } catch (e) {} }
  else if (curCode && (joined || resuming) && (!hostConn || !hostConn.open || Date.now() - lastHostMsg > 8000)) connectToHost();
}
document.addEventListener('visibilitychange', wake); window.addEventListener('online', wake); window.addEventListener('focus', wake);

// ======================= HOST: GAME RULES =======================
function startGame() {
  const n = names.length;
  if (n < MIN_PLAYERS || !online().every(Boolean)) return;
  const decks = Math.ceil(n / 8), deck = [...Array(52 * decks).keys()];
  for (let i = deck.length - 1; i > 0; i--) { const j = Math.floor(Math.random() * (i + 1)); [deck[i], deck[j]] = [deck[j], deck[i]]; }
  const hands = names.map(() => []); deck.forEach((c, i) => hands[i % n].push(c));   // deal round-robin
  S = { hands, pile: [], last: null, turn: Math.floor(Math.random() * n), rank: 0, phase: 'play',
        decided: {}, log: '', reveal: null, over: null };
  S.log = 'Cards dealt - ' + names[S.turn] + ' starts.';
  inGame = true; pushState();
}
const minCards = () => S.pile.length === 0 ? 2 : 1;
// Build what ONE player is allowed to see
function view(i) {
  return { t: 'state', you: i, hand: S.hands[i].slice().sort((a, b) => a % 13 - b % 13 || a - b),
           counts: S.hands.map(h => h.length), pile: S.pile.length, turn: S.turn, rank: S.rank, phase: S.phase,
           passed: !!S.decided[i], wins: hostWins, online: online(), min: Math.min(minCards(), S.hands[i].length),
           log: S.log, reveal: S.reveal, over: S.over, names };
}
function pushState() { sendAll(); }
function advance() { S.turn = (S.turn + 1) % names.length; S.rank = (S.rank + 1) % 13; S.phase = 'play'; S.decided = {}; S.last = null; }
function win(i) { S.over = i; hostWins[i]++; inGame = false; S.log = names[i] + ' got rid of every card!'; pushState(); }

function onHostMsg(from, m) {
  if (!inGame || !S || S.over !== null || !S.hands[from]) return;
  const n = names.length;

  if (m.t === 'play' && S.phase === 'play' && S.turn === from) {
    const ids = [...new Set(m.ids)];                              // validate: distinct cards I really hold
    const need = Math.min(minCards(), S.hands[from].length);      // 2+ on an empty pile, else 1+
    if (ids.length < need || !ids.every(id => S.hands[from].includes(id))) return;
    S.hands[from] = S.hands[from].filter(id => !ids.includes(id));
    S.pile.push(...ids); S.last = { p: from, ids }; S.phase = 'call'; S.decided = {}; S.reveal = null;
    S.log = `${names[from]} played ${ids.length} card${ids.length > 1 ? 's' : ''} as ${RW[S.rank]}`;
    pushState();
  }
  else if (m.t === 'pass' && S.phase === 'call' && from !== S.turn && !S.decided[from]) {
    S.decided[from] = 1;
    if (Object.keys(S.decided).length === n - 1) {                // everybody else let it pass
      if (S.hands[S.turn].length === 0) return win(S.turn);       // survived with an empty hand = winner
      advance();
    }
    pushState();
  }
  else if (m.t === 'call' && S.phase === 'call' && from !== S.turn) {
    const truthful = S.last.ids.every(id => id % 13 === S.rank);
    const loser = truthful ? from : S.turn;                       // wrong caller or caught liar picks up
    S.reveal = { cards: S.last.ids, truthful, caller: from, loser, n: S.pile.length };
    S.log = `${names[from]} called BLUFF on ${names[S.turn]} - ${truthful ? 'it was TRUE' : 'it was a LIE'}! ${names[loser]} picks up ${S.pile.length} cards.`;
    S.hands[loser].push(...S.pile); S.pile = [];
    if (truthful && S.hands[S.turn].length === 0) return win(S.turn);
    advance(); pushState();
  }
}

// ======================= EVERY PHONE: UI =======================
function onMsg(m) {
  if (m.t === 'full') return err('Room is full');
  if (m.t === 'lobby') { if (!isHost) me = m.you; names = m.names; lobbyOnline = m.online || names.map(() => true); renderLobby(); if (!$('game').classList.contains('on')) show('lobby'); }
  else if (m.t === 'state') { G = m; names = m.names; me = m.you; lobbyOnline = m.online || lobbyOnline; sel = new Set([...sel].filter(id => m.hand.includes(id))); show('game'); render(); }
  else if (m.t === 'toLobby') { G = null; sel.clear(); show('lobby'); renderLobby(); }
}
function renderLobby() {
  $('plist').innerHTML = names.map((nm, i) => `<div class="pl" style="border-left:6px solid ${COLORS[i % COLORS.length]};padding-left:10px">${
    esc(nm) + (i === me ? ' (you)' : '') + (lobbyOnline[i] === false ? ' ⚠️ offline' : '')}</div>`).join('');
  const n = names.length, allOn = lobbyOnline.every(x => x !== false), ready = n >= MIN_PLAYERS && allOn;
  $('startBtn').style.display = isHost ? '' : 'none'; $('startBtn').disabled = !ready;
  const anyOff = isHost && !allOn;
  $('kickBtn').style.display = anyOff ? '' : 'none';
  $('lobbyNote').textContent = isHost
    ? (n < MIN_PLAYERS ? `Waiting for more players (${n}/${MAX_PLAYERS})…` : allOn ? `${n} players in - deal whenever everyone has joined (max ${MAX_PLAYERS})` : 'Someone is offline - wait for them or remove them')
    : `${n} player${n === 1 ? '' : 's'} in - waiting for the host to deal…`;
}
const cardHtml = id => `<span class="cd ${isRed(id) ? 'red' : ''}">${cardLabel(id)}</span>`;

function render() {
  const g = G, myTurn = g.turn === me, over = g.over !== null;
  // player strip: name, cards left, wins
  $('strip').innerHTML = g.counts.map((cnt, i) => `<div class="pc ${g.turn === i && !over ? 'turn' : ''}" style="border-color:${COLORS[i % COLORS.length]}">
    <span>${esc(names[i])}${i === me ? ' (you)' : ''}${g.online && g.online[i] === false ? ' ⚠️' : ''}</span><b>${cnt}</b><small>🏆 ${g.wins[i] || 0}</small></div>`).join('');
  // table
  $('rank').textContent = RW[g.rank];
  $('pile').textContent = '🂠 Pile: ' + g.pile + ' card' + (g.pile === 1 ? '' : 's');
  $('reveal').innerHTML = g.reveal ? g.reveal.cards.map(cardHtml).join('') : '';
  $('log').textContent = g.log;
  // my hand (tap to select while it is my turn)
  $('hand').innerHTML = g.hand.map(id => `<button class="cd ${isRed(id) ? 'red' : ''} ${sel.has(id) ? 'sel' : ''}" data-id="${id}">${cardLabel(id)}</button>`).join('');
  // action area
  const b = { play: 'none', call: 'none', pass: 'none', again: 'none' }; let st = '';
  if (over) { st = g.over === me ? '🏆 You win!' : names[g.over] + ' wins this game'; if (isHost) b.again = ''; else st += ' - waiting for host to redeal'; }
  else if (g.phase === 'play') {
    if (myTurn) { b.play = ''; st = `Your turn: select ${g.min >= 2 ? 'at least 2 cards (empty pile)' : 'as many cards as you like'} and claim they are ${RW[g.rank]}`; }
    else st = `Waiting for ${names[g.turn]} to play ${RW[g.rank]}…`;
  } else {
    if (myTurn) st = 'Waiting to see if anyone calls your bluff…';
    else if (g.passed) st = 'You let it pass. Waiting for the others…';
    else { b.call = ''; b.pass = ''; st = `${names[g.turn]} says those cards are ${RW[g.rank]}. Believe them?`; }
  }
  $('status').textContent = st;
  $('playBtn').style.display = b.play; $('callBtn').style.display = b.call;
  $('passBtn').style.display = b.pass; $('againBtn').style.display = b.again;
  $('kickBtn2').style.display = isHost && g.online && g.online.some(x => x === false) ? '' : 'none';
  $('playBtn').disabled = sel.size < (g.min || 1);
  $('playBtn').textContent = `Play ${sel.size || ''} card${sel.size === 1 ? '' : 's'} as ${RW[g.rank]}`;
}

// ======================= WIRING =======================
$('createBtn').onclick = createRoom; $('joinBtn').onclick = () => joinRoom();
$('leaveBtn').onclick = $('netLeave').onclick = () => { if (confirm('Leave this room?')) leave(); };
$('kickBtn').onclick = $('kickBtn2').onclick = () => { if (confirm('Remove offline players' + (inGame ? ' and end this game?' : '?'))) kickOffline(); };
$('startBtn').onclick = startGame; $('againBtn').onclick = startGame;
$('hand').addEventListener('click', e => {                       // select / deselect a card
  const el = e.target.closest('[data-id]'); if (!el || !G || G.phase !== 'play' || G.turn !== me || G.over !== null) return;
  const id = +el.dataset.id; sel.has(id) ? sel.delete(id) : sel.add(id); render();
});
$('playBtn').onclick = () => { toHost({ t: 'play', ids: [...sel] }); sel.clear(); };
$('callBtn').onclick = () => toHost({ t: 'call' });
$('passBtn').onclick = () => toHost({ t: 'pass' });

// ======================= AUTO-RESUME after reload / killed tab =======================
(() => {
  const sv = loadSession(); if (!sv) return;
  $('name').value = sv.name; myName = sv.name; banner('Rejoining room ' + sv.code + '…');
  if (sv.role === 'host') hostStart(sv.code, true, sv);
  else { resuming = true; joinRoom(sv.code); }
})();
</script>
</body>
</html>
