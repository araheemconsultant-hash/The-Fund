<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="utf-8">
<meta name="viewport" content="width=device-width, initial-scale=1, viewport-fit=cover">
<title>Bluff – 3 Player Card Game</title>
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
  #strip { display:flex; gap:8px; }
  .pc { flex:1; text-align:center; border:3px solid var(--line); border-radius:12px; padding:6px; background:var(--card); }
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
</style>
</head>
<body>

<!-- ============ HOME ============ -->
<div class="screen on" id="home">
  <h1>🃏 Bluff</h1>
  <p class="muted">The 3-player card game of lying with a straight face</p>
  <div class="box"><input id="name" placeholder="Your name" maxlength="12" autocomplete="off">
    <button class="big" id="createBtn">Create room</button></div>
  <div class="box"><input id="code" placeholder="ROOM CODE" maxlength="4" autocomplete="off">
    <button class="big" id="joinBtn">Join room</button></div>
  <div id="err"></div>
</div>

<!-- ============ LOBBY ============ -->
<div class="screen" id="lobby">
  <h1>Room code</h1><div id="roomCode">----</div>
  <p class="muted">Share this code with the other two players</p>
  <div class="box" id="plist"></div>
  <button class="big" id="startBtn" disabled>Deal the cards</button>
  <p class="muted" id="lobbyNote"></p>
  <div class="box" id="rules"><b>Rules.</b> All 52 cards are dealt. Ranks go A, 2, 3 … K, A … in order, one rank per turn.
    On your turn put 1–4 cards face down and say they are all of the required rank. You may lie.
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
    <button class="big" id="againBtn" style="display:none">Play again</button></div>
</div>

<script>
/* =====================================================================
   BLUFF – ONLINE, 3 PLAYERS
   Networking: the HOST's phone is the referee and the only place the full
   game state lives. Each player is sent ONLY their own hand, so nobody can
   peek at the others' cards. Host creates PeerJS id "tws-bluff-<CODE>".
   Messages  guest -> host : hello{name} | play{ids} | call | pass
             host  -> guest: lobby{names,you} | state{...personal view...} | abort
   Cards are numbers 0..51:  rank = id % 13 (0=Ace..12=King), suit = floor(id/13).
   ===================================================================== */
const COLORS = ['#d94a3d', '#2f9e5b', '#2f6fd9'];
const RK = ['A','2','3','4','5','6','7','8','9','10','J','Q','K'];
const RW = ['Aces','2s','3s','4s','5s','6s','7s','8s','9s','10s','Jacks','Queens','Kings'];
const SU = ['♠','♥','♦','♣'];
const $ = id => document.getElementById(id);
const esc = s => String(s).replace(/[&<>"]/g, c => ({'&':'&amp;','<':'&lt;','>':'&gt;','"':'&quot;'}[c]));
const show = id => document.querySelectorAll('.screen').forEach(s => s.classList.toggle('on', s.id === id));
const err = t => { $('err').textContent = t; show('home'); };
const cardLabel = id => RK[id % 13] + SU[Math.floor(id / 13)];
const isRed = id => { const s = Math.floor(id / 13); return s === 1 || s === 2; };

// ---------- shared state ----------
let peer, isHost = false, me = 0, myName = '', hostConn = null;
let conns = [null, null, null];          // host only: connection per player slot
let names = ['', '', ''];
let G = null;                            // latest personal view of the game (every player)
let sel = new Set();                     // card ids I have selected to play
// ---------- host-only game state ----------
let S = null, inGame = false, hostWins = [0, 0, 0];

// ======================= CONNECTION =======================
const randCode = () => { const a = 'ABCDEFGHJKLMNPQRSTUVWXYZ23456789'; let s = '';
  for (let i = 0; i < 4; i++) s += a[Math.floor(Math.random() * a.length)]; return s; };
const getName = () => $('name').value.trim() || 'Player';
function toHost(m) { if (isHost) onHostMsg(0, m); else if (hostConn && hostConn.open) hostConn.send(m); }
function sendLobby() {
  conns.forEach((c, i) => c && c.open && c.send({ t: 'lobby', names, you: i }));
  onMsg({ t: 'lobby', names, you: 0 });
}
function createRoom() {
  if (typeof Peer === 'undefined') return err('PeerJS library did not load (check internet / blocked site)');
  myName = getName(); const code = randCode();
  peer = new Peer('tws-bluff-' + code);
  peer.on('open', () => { isHost = true; me = 0; names = [myName, '', '']; $('roomCode').textContent = code; show('lobby'); sendLobby(); });
  peer.on('error', e => e.type === 'unavailable-id' ? createRoom() : err('Connection problem: ' + e.type));
  peer.on('connection', c => c.on('open', () => {
    const idx = [1, 2].find(i => !conns[i]);
    if (!idx) { c.send({ t: 'full' }); setTimeout(() => c.close(), 300); return; }
    conns[idx] = c;
    c.on('data', m => onHostMsg(idx, m));
    c.on('close', () => {                                  // a player left
      conns[idx] = null; names[idx] = '';
      if (inGame) { inGame = false; conns.forEach(x => x && x.open && x.send({ t: 'abort' })); onMsg({ t: 'abort' }); }
      sendLobby();
    });
  }));
}
function joinRoom() {
  if (typeof Peer === 'undefined') return err('PeerJS library did not load (check internet / blocked site)');
  myName = getName(); const code = $('code').value.trim().toUpperCase();
  if (code.length !== 4) return err('Enter the 4-letter room code');
  peer = new Peer();
  peer.on('error', e => err(e.type === 'peer-unavailable' ? 'Room not found' : 'Connection problem: ' + e.type));
  peer.on('open', () => {
    hostConn = peer.connect('tws-bluff-' + code, { reliable: true });
    hostConn.on('open', () => hostConn.send({ t: 'hello', name: myName }));
    hostConn.on('data', onMsg);
    hostConn.on('close', () => err('The host left the room'));
  });
}

// ======================= HOST: GAME RULES =======================
function startGame() {
  const deck = [...Array(52).keys()];
  for (let i = 51; i > 0; i--) { const j = Math.floor(Math.random() * (i + 1)); [deck[i], deck[j]] = [deck[j], deck[i]]; }
  const hands = [[], [], []]; deck.forEach((c, i) => hands[i % 3].push(c));   // deal round-robin
  S = { hands, pile: [], last: null, turn: Math.floor(Math.random() * 3), rank: 0, phase: 'play',
        decided: {}, log: 'Cards dealt. ' + names[0] + ' ... let the lying begin!', reveal: null, over: null };
  S.log = 'Cards dealt - ' + names[S.turn] + ' starts.';
  inGame = true; pushState();
}
// Build what ONE player is allowed to see
function view(i) {
  return { t: 'state', you: i, hand: S.hands[i].slice().sort((a, b) => a % 13 - b % 13 || a - b),
           counts: S.hands.map(h => h.length), pile: S.pile.length, turn: S.turn, rank: S.rank, phase: S.phase,
           passed: !!S.decided[i], wins: hostWins, log: S.log, reveal: S.reveal, over: S.over, names };
}
function pushState() { conns.forEach((c, i) => c && c.open && c.send(view(i))); onMsg(view(0)); }
function advance() { S.turn = (S.turn + 1) % 3; S.rank = (S.rank + 1) % 13; S.phase = 'play'; S.decided = {}; S.last = null; }
function win(i) { S.over = i; hostWins[i]++; inGame = false; S.log = names[i] + ' got rid of every card!'; pushState(); }

function onHostMsg(from, m) {
  if (m.t === 'hello') { names[from] = m.name || 'Player ' + (from + 1); sendLobby(); return; }
  if (!inGame || !S || S.over !== null) return;

  if (m.t === 'play' && S.phase === 'play' && S.turn === from) {
    const ids = [...new Set(m.ids)];                              // validate: 1-4 distinct cards I really hold
    if (!ids.length || ids.length > 4 || !ids.every(id => S.hands[from].includes(id))) return;
    S.hands[from] = S.hands[from].filter(id => !ids.includes(id));
    S.pile.push(...ids); S.last = { p: from, ids }; S.phase = 'call'; S.decided = {}; S.reveal = null;
    S.log = `${names[from]} played ${ids.length} card${ids.length > 1 ? 's' : ''} as ${RW[S.rank]}`;
    pushState();
  }
  else if (m.t === 'pass' && S.phase === 'call' && from !== S.turn && !S.decided[from]) {
    S.decided[from] = 1;
    if (Object.keys(S.decided).length === 2) {                    // both let it pass
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
  if (m.t === 'lobby') { if (!isHost) me = m.you; names = m.names; renderLobby(); if (!$('game').classList.contains('on')) show('lobby'); }
  else if (m.t === 'state') { G = m; names = m.names; me = m.you; sel = new Set([...sel].filter(id => m.hand.includes(id))); show('game'); render(); }
  else if (m.t === 'abort') { G = null; sel.clear(); err('A player left - back to the lobby'); show('lobby'); }
}
function renderLobby() {
  $('plist').innerHTML = [0, 1, 2].map(i => `<div class="pl" style="border-left:6px solid ${COLORS[i]};padding-left:10px">${
    names[i] ? esc(names[i]) + (i === me ? ' (you)' : '') : '<span class="muted">waiting…</span>'}</div>`).join('');
  const full = names.every(Boolean);
  $('startBtn').style.display = isHost ? '' : 'none'; $('startBtn').disabled = !full;
  $('lobbyNote').textContent = isHost ? (full ? 'Everyone is in!' : 'Waiting for 2 more players…') : 'Waiting for the host to deal…';
}
const cardHtml = id => `<span class="cd ${isRed(id) ? 'red' : ''}">${cardLabel(id)}</span>`;

function render() {
  const g = G, myTurn = g.turn === me, over = g.over !== null;
  // player strip: name, cards left, wins
  $('strip').innerHTML = [0, 1, 2].map(i => `<div class="pc ${g.turn === i && !over ? 'turn' : ''}" style="border-color:${COLORS[i]}">
    <span>${esc(names[i])}${i === me ? ' (you)' : ''}</span><b>${g.counts[i]}</b><small>🏆 ${g.wins[i]}</small></div>`).join('');
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
    if (myTurn) { b.play = ''; st = `Your turn: select 1–4 cards and claim they are ${RW[g.rank]}`; }
    else st = `Waiting for ${names[g.turn]} to play ${RW[g.rank]}…`;
  } else {
    if (myTurn) st = 'Waiting to see if anyone calls your bluff…';
    else if (g.passed) st = 'You let it pass. Waiting for the other player…';
    else { b.call = ''; b.pass = ''; st = `${names[g.turn]} says those cards are ${RW[g.rank]}. Believe them?`; }
  }
  $('status').textContent = st;
  $('playBtn').style.display = b.play; $('callBtn').style.display = b.call;
  $('passBtn').style.display = b.pass; $('againBtn').style.display = b.again;
  $('playBtn').disabled = sel.size < 1 || sel.size > 4;
  $('playBtn').textContent = `Play ${sel.size || ''} card${sel.size === 1 ? '' : 's'} as ${RW[g.rank]}`;
}

// ======================= WIRING =======================
$('createBtn').onclick = createRoom; $('joinBtn').onclick = joinRoom;
$('startBtn').onclick = startGame; $('againBtn').onclick = startGame;
$('hand').addEventListener('click', e => {                       // select / deselect a card
  const el = e.target.closest('[data-id]'); if (!el || !G || G.phase !== 'play' || G.turn !== me || G.over !== null) return;
  const id = +el.dataset.id; sel.has(id) ? sel.delete(id) : (sel.size < 4 && sel.add(id)); render();
});
$('playBtn').onclick = () => { toHost({ t: 'play', ids: [...sel] }); sel.clear(); };
$('callBtn').onclick = () => toHost({ t: 'call' });
$('passBtn').onclick = () => toHost({ t: 'pass' });
</script>
</body>
</html>
