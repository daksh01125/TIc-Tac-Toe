<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="utf-8" />
<meta name="viewport" content="width=device-width, initial-scale=1" />
<title>Tic-Tac-Toe — Unbeatable AI (Minimax + Alpha-Beta)</title>
<style>
  :root { --bg:#0f172a; --card:#111827; --line:#374151; --x:#22d3ee; --o:#a78bfa; --text:#e5e7eb; --muted:#9ca3af; --accent:#f59e0b; }
  * { box-sizing: border-box; }
  body {
    margin: 0; min-height: 100vh; display: grid; place-items: center; background: radial-gradient(60vw 60vw at 50% -10%, #1f2937, var(--bg));
    color: var(--text); font: 16px/1.2 system-ui, -apple-system, Segoe UI, Roboto, Inter, sans-serif;
  }
  .app { width: 100%; max-width: 520px; padding: 24px; }
  .card {
    background: linear-gradient(180deg, rgba(255,255,255,0.03), rgba(0,0,0,0.25));
    border: 1px solid rgba(255,255,255,0.06);
    border-radius: 20px; padding: 20px; box-shadow: 0 10px 30px rgba(0,0,0,0.35);
  }
  h1 { margin: 0 0 10px; font-size: 24px; letter-spacing: 0.3px; }
  .controls { display: flex; gap: 12px; flex-wrap: wrap; align-items: center; margin: 10px 0 18px; }
  .select, .button {
    background: #0b1220; color: var(--text); border: 1px solid rgba(255,255,255,0.08); border-radius: 12px;
    padding: 10px 14px; cursor: pointer; transition: transform .08s ease, border-color .2s ease;
  }
  .button:active { transform: translateY(1px); }
  .grid {
    width: 100%; aspect-ratio: 1 / 1; display: grid; gap: 10px; grid-template-columns: repeat(3, 1fr); margin-bottom: 14px;
  }
  .cell {
    background: var(--card); border: 1px solid var(--line); border-radius: 16px; display: grid; place-items: center;
    font-size: clamp(40px, 10vw, 64px); font-weight: 800; user-select: none; cursor: pointer; position: relative;
    transition: border-color .2s ease, transform .06s ease;
  }
  .cell:hover { border-color: rgba(255,255,255,0.18); }
  .cell:active { transform: scale(0.99); }
  .cell.x { color: var(--x); text-shadow: 0 0 20px rgba(34,211,238,.25); }
  .cell.o { color: var(--o); text-shadow: 0 0 20px rgba(167,139,250,.25); }
  .cell.win { box-shadow: 0 0 0 2px var(--accent) inset, 0 0 40px rgba(245,158,11,.25); }
  .status { display: flex; justify-content: space-between; align-items: center; gap: 10px; }
  .status .text { color: var(--muted); }
  .pill {
    border-radius: 999px; background: #0b1220; border: 1px solid rgba(255,255,255,0.08);
    padding: 6px 10px; font-size: 13px; color: var(--muted);
  }
  .legend { display:flex; gap:8px; align-items:center; font-size:13px; color:var(--muted); }
  .dot { width:10px; height:10px; border-radius:50%; display:inline-block; }
  .dot.x { background: var(--x); }
  .dot.o { background: var(--o); }
  .footer { margin-top: 10px; font-size: 12px; color: var(--muted); text-align:center; }
  @media (prefers-reduced-motion: no-preference) {
    .appear { animation: pop .18s ease-out; }
    @keyframes pop { from { transform: scale(.96); opacity:.6;} to { transform: scale(1); opacity:1;} }
  }
</style>
</head>
<body>
  <div class="app card appear">
    <h1>Tic-Tac-Toe <span class="pill">Unbeatable AI</span></h1>
    <div class="controls">
      <label class="select">
        You play:
        <select id="sideSelect" style="background:transparent;border:none;color:inherit;margin-left:6px;outline:none;">
          <option value="X">X (first)</option>
          <option value="O">O (second)</option>
        </select>
      </label>
      <button id="resetBtn" class="button">Reset</button>
      <div class="legend" aria-hidden="true">
        <span class="dot x"></span> X &nbsp; <span class="dot o"></span> O
      </div>
    </div>

    <div id="grid" class="grid" role="grid" aria-label="Tic Tac Toe board"></div>

    <div class="status">
      <div id="statusText" class="text">Your turn.</div>
      <div id="scorePill" class="pill">X: <span id="scoreX">0</span> &nbsp; O: <span id="scoreO">0</span></div>
    </div>
    <div class="footer">Minimax + Alpha-Beta Pruning · Prefers quicker wins / delays losses</div>
  </div>

<script>
(() => {
  const gridEl   = document.getElementById('grid');
  const statusEl = document.getElementById('statusText');
  const sideSel  = document.getElementById('sideSelect');
  const resetBtn = document.getElementById('resetBtn');
  const scoreXEl = document.getElementById('scoreX');
  const scoreOEl = document.getElementById('scoreO');

  let board, human, ai, gameOver, scores = { X: 0, O: 0 };

  const WIN_LINES = [
    [0,1,2],[3,4,5],[6,7,8], // rows
    [0,3,6],[1,4,7],[2,5,8], // cols
    [0,4,8],[2,4,6]          // diags
  ];

  function init() {
    human = sideSel.value;
    ai = human === 'X' ? 'O' : 'X';
    board = Array(9).fill('');
    gameOver = false;
    render();
    statusEl.textContent = human === 'X' ? 'Your turn.' : 'AI thinking...';
    if (human === 'O') {
      // AI starts
      setTimeout(aiMove, 220);
    }
  }

  // UI
  function render(winningLine = null) {
    gridEl.innerHTML = '';
    for (let i = 0; i < 9; i++) {
      const cell = document.createElement('button');
      cell.className = 'cell' + (board[i] ? ' ' + board[i].toLowerCase() : '');
      cell.setAttribute('role','gridcell');
      cell.setAttribute('aria-label', board[i] ? board[i] : 'empty');
      cell.dataset.idx = i;
      cell.textContent = board[i];
      if (winningLine && winningLine.includes(i)) cell.classList.add('win');

      cell.disabled = !!board[i] || gameOver;
      cell.addEventListener('click', onHumanClick);
      gridEl.appendChild(cell);
    }
  }

  function onHumanClick(e) {
    const i = Number(e.currentTarget.dataset.idx);
    if (gameOver || board[i]) return;
    place(i, human);
    const result = evaluateBoard(board);
    if (result.done) {
      endGame(result);
    } else {
      statusEl.textContent = 'AI thinking...';
      // Let the UI update before AI calculates
      setTimeout(aiMove, 80);
    }
  }

  function place(i, p) { board[i] = p; render(); }

  // Game evaluation
  function evaluateBoard(b) {
    for (const line of WIN_LINES) {
      const [a,b2,c] = line;
      if (b[a] && b[a] === b[b2] && b[a] === b[c]) {
        return { done: true, winner: b[a], line };
      }
    }
    if (!b.includes('')) return { done: true, winner: 'draw', line: null };
    return { done: false, winner: null, line: null };
  }

  // Minimax with Alpha-Beta
  function minimax(b, current, depth, alpha, beta) {
    const evalResult = evaluateBoard(b);
    if (evalResult.done) {
      if (evalResult.winner === ai)   return { score: 10 - depth };
      if (evalResult.winner === human) return { score: depth - 10 };
      return { score: 0 }; // draw
    }

    const isMax = (current === ai);
    let bestScore = isMax ? -Infinity : Infinity;
    let bestMove = null;

    // Optional small move-ordering heuristic: pick center, corners, edges
    const order = [4,0,2,6,8,1,3,5,7].filter(i => b[i] === '');

    for (const i of order) {
      b[i] = current;
      const { score } = minimax(b, current === 'X' ? 'O' : 'X', depth + 1, alpha, beta);
      b[i] = '';

      if (isMax) {
        if (score > bestScore) { bestScore = score; bestMove = i; }
        alpha = Math.max(alpha, bestScore);
      } else {
        if (score < bestScore) { bestScore = score; bestMove = i; }
        beta = Math.min(beta, bestScore);
      }
      if (beta <= alpha) break; // prune
    }
    return { score: bestScore, move: bestMove };
  }

  function aiMove() {
    if (gameOver) return;

    // Quick tactical checks before full search (speeds UX)
    const winning = findImmediate(ai);
    const blocking = findImmediate(human);
    const move = winning ?? blocking ?? strategicOpening() ?? bestBySearch();

    place(move, ai);

    const result = evaluateBoard(board);
    if (result.done) {
      endGame(result);
    } else {
      statusEl.textContent = 'Your turn.';
    }
  }

  function findImmediate(player) {
    for (const line of WIN_LINES) {
      const [a,b2,c] = line;
      const trio = [board[a], board[b2], board[c]];
      if (trio.filter(v => v === player).length === 2 && trio.includes('')) {
        if (!board[a]) return a;
        if (!board[b2]) return b2;
        if (!board[c]) return c;
      }
    }
    return null;
  }

  function strategicOpening() {
    // Prefer center, then corners, early in the game
    const empties = board.reduce((acc,v,i)=>(v===''&&acc.push(i),acc),[]);
    if (empties.length >= 7) {
      if (board[4] === '') return 4;
      const corners = [0,2,6,8].filter(i => board[i] === '');
      if (corners.length) return corners[Math.floor(Math.random() * corners.length)];
    }
    return null;
  }

  function bestBySearch() {
    const { move } = minimax([...board], ai, 0, -Infinity, Infinity);
    return move;
  }

  function endGame(result) {
    gameOver = true;
    if (result.winner === 'draw') {
      statusEl.textContent = "It's a draw!";
    } else {
      statusEl.textContent = (result.winner === human) ? 'You win! 🎉' : 'AI wins! 🤖';
      scores[result.winner] += 1;
      scoreXEl.textContent = scores.X;
      scoreOEl.textContent = scores.O;
    }
    render(result.line);
  }

  // Events
  sideSel.addEventListener('change', init);
  resetBtn.addEventListener('click', init);

  // Kick off
  init();
})();
</script>
</body>
</html>
