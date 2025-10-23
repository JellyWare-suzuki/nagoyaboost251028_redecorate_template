<script>
  import { gridState } from '$lib/stores/gridState';

  const LABELS = [
    '-', 'チェア↑', 'チェア→', 'チェア↓', 'チェア←',
    'テーブル↑', 'テーブル→', 'テーブル↓', 'テーブル←',
    'クローゼット↑', 'クローゼット→', 'クローゼット↓', 'クローゼット←'
  ];

  const GRID_SIZE = 8;
  const MAX_BUTTON_VALUE = 12;

  const COLOR_MAP = {
    0: '#f7f5f2',
    1: '#ffe4e1', 2: '#ffe4e1', 3: '#ffe4e1', 4: '#ffe4e1',
    5: '#e0f7fa', 6: '#e0f7fa', 7: '#e0f7fa', 8: '#e0f7fa',
    9: '#ede7f6', 10: '#ede7f6', 11: '#ede7f6', 12: '#ede7f6'
  };

  function animateClick(btn, event) {
    // クリック座標をCSS変数へ（リップルの中心）
    const rect = btn.getBoundingClientRect();
    const x = (event?.clientX ?? rect.left + rect.width / 2) - rect.left;
    const y = (event?.clientY ?? rect.top + rect.height / 2) - rect.top;
    btn.style.setProperty('--x', `${x}px`);
    btn.style.setProperty('--y', `${y}px`);

    // リップル再生のためクラスを付け直す
    btn.classList.remove('ripple');
    // reflowでアニメ再適用
    // eslint-disable-next-line no-unused-expressions
    btn.offsetWidth;
    btn.classList.add('ripple');

    // ポップ（縮小→戻る）
    btn.classList.add('pop');
    setTimeout(() => btn.classList.remove('pop'), 180);
  }

  function handleClick(index, event) {
    gridState.update((prev) => {
      const next = [...prev];
      next[index] = (next[index] + 1) % (MAX_BUTTON_VALUE + 1);
      return next;
    });

    animateClick(event.currentTarget, event);
  }

  // キーボード操作でも同じ効果（Enter/Space）
  function handleKey(e, index) {
    if (e.key === 'Enter' || e.key === ' ') {
      e.preventDefault();
      handleClick(index, e);
    }
  }
</script>

<div class="grid-wrapper">
  <h2>お部屋レイアウトプランナー</h2>
  <table>
    <tbody>
      {#each Array(GRID_SIZE) as _, row}
        <tr>
          {#each Array(GRID_SIZE) as _, col}
            <td>
              <button
                class="cell"
                style="background-color: {COLOR_MAP[$gridState[row * GRID_SIZE + col]]};"
                on:click={(e) => handleClick(row * GRID_SIZE + col, e)}
                on:keydown={(e) => handleKey(e, row * GRID_SIZE + col)}
                aria-label={`cell-${row}-${col}`}
              >
                {LABELS[$gridState[row * GRID_SIZE + col]]}
                <!-- 方向に合わせたアイコンの軽い演出（任意） -->
                <span class="hint">
                  {#if $gridState[row * GRID_SIZE + col] !== 0}
                    →
                  {/if}
                </span>
              </button>
            </td>
          {/each}
        </tr>
      {/each}
    </tbody>
  </table>
</div>

<style>
  @import url('https://fonts.googleapis.com/css2?family=Zen+Kaku+Gothic+New:wght@400;500&display=swap');

  :root{
    --card-bg: #faf6f1;
    --text-main: #554a43;
    --border: #d8cfc7;
    --shadow: 0 6px 18px rgba(0,0,0,0.10);
    --shadow-hover: 0 10px 24px rgba(0,0,0,0.16);
  }

  .grid-wrapper {
    display: inline-block;
    padding: 20px;
    border-radius: 16px;
    background-color: var(--card-bg);
    border: 1px solid var(--border);
    box-shadow: var(--shadow);
    font-family: 'Zen Kaku Gothic New', system-ui, -apple-system, sans-serif;
  }

  h2 {
    text-align: center;
    margin-bottom: 16px;
    color: var(--text-main);
    font-weight: 600;
    letter-spacing: .02em;
  }

  table {
    border-collapse: collapse;
  }
  td { padding: 0; }

  .cell {
    position: relative;
    width: 96px;
    height: 64px;
    border: 1px solid #cfc7bf;
    border-radius: 10px;
    cursor: pointer;
    text-align: center;
    font-size: 13px;
    line-height: 1.2;
    color: #333;
    transition: transform .18s ease, box-shadow .18s ease, border-color .18s ease;
    /* リップル用 */
    overflow: hidden;
    isolation: isolate; /* 擬似要素のブレンドを安全に */
  }

  /* 浮遊感 */
  .cell:hover {
    transform: translateY(-1px);
    box-shadow: var(--shadow-hover);
    border-color: #b8b0a8;
  }

  /* キーボードフォーカス可視化 */
  .cell:focus-visible {
    outline: 3px solid rgba(66, 153, 225, .45);
    outline-offset: 2px;
  }

  /* クリック瞬間の“ポップ” */
  .cell.pop {
    transform: scale(0.96);
  }

  /* リップル（クリック位置から広がる） */
  .cell::after {
    content: '';
    position: absolute;
    left: var(--x, 50%);
    top: var(--y, 50%);
    width: 12px;
    height: 12px;
    background: rgba(255,255,255,0.65);
    border-radius: 999px;
    transform: translate(-50%, -50%) scale(0);
    opacity: 0;
    pointer-events: none;
    mix-blend-mode: screen;
  }
  .cell.ripple::after {
    animation: ripple .55s ease-out forwards;
  }
  @keyframes ripple {
    0%   { transform: translate(-50%, -50%) scale(0); opacity: .65; }
    70%  { opacity: .25; }
    100% { transform: translate(-50%, -50%) scale(18); opacity: 0; }
  }

  /* ラベル補助の控えめな矢印（任意） */
  .hint {
    position: absolute;
    right: 6px;
    bottom: 4px;
    font-size: 12px;
    opacity: .18;
    user-select: none;
    pointer-events: none;
  }
</style>
