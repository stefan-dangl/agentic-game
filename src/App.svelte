<script>
  import { onDestroy, onMount } from 'svelte';

  const cols = 8;
  const rows = 8;
  const tileW = 96;
  const tileH = 48;

  const obstaclePositions = new Set(['3,3', '4,3', '5,4', '2,6', '6,2']);

  const tiles = Array.from({ length: rows * cols }, (_, index) => {
    const x = index % cols;
    const y = Math.floor(index / cols);
    return {
      id: `${x}-${y}`,
      x,
      y,
      screen: toIso(x, y),
      isObstacle: obstaclePositions.has(`${x},${y}`),
      z: x + y
    };
  });

  let player = { x: 1, y: 1 };
  let status = 'Use WASD or arrow keys to move';

  const offsetX = ((cols - 1) * tileW) / 2;

  function toIso(x, y) {
    return {
      x: (x - y) * (tileW / 2) + offsetX,
      y: (x + y) * (tileH / 2)
    };
  }

  function isBlocked(x, y) {
    if (x < 0 || y < 0 || x >= cols || y >= rows) return true;
    return obstaclePositions.has(`${x},${y}`);
  }

  function attemptMove(dx, dy) {
    const nextX = player.x + dx;
    const nextY = player.y + dy;
    if (isBlocked(nextX, nextY)) {
      status = 'You bumped into an obstacle';
      return;
    }
    player = { x: nextX, y: nextY };
    status = 'Walking...';
  }

  function handleKey(event) {
    const key = event.key.toLowerCase();
    const moveMap = {
      w: [0, -1],
      arrowup: [0, -1],
      s: [0, 1],
      arrowdown: [0, 1],
      a: [-1, 0],
      arrowleft: [-1, 0],
      d: [1, 0],
      arrowright: [1, 0]
    };

    const delta = moveMap[key];
    if (delta) {
      event.preventDefault();
      attemptMove(delta[0], delta[1]);
    }
  }

  onMount(() => {
    window.addEventListener('keydown', handleKey);
  });

  onDestroy(() => {
    window.removeEventListener('keydown', handleKey);
  });
</script>

<main class="app">
  <header>
    <p class="eyebrow">Tiny isometric playground</p>
    <h1>Roam the tiles</h1>
    <p class="lede">
      A simple isometric grid with a controllable player and solid obstacles. Use WASD
      or the arrow keys to walk around.
    </p>
  </header>

  <section class="world-panel">
    <div class="hud">
      <span class="pill">WASD / Arrow keys</span>
      <span class="pill">Obstacles block movement</span>
    </div>

    <div
      class="world"
      style={`--tile-w:${tileW}px; --tile-h:${tileH}px; --cols:${cols}; --rows:${rows};`}
    >
      {#each tiles as tile (tile.id)}
        <div
          class={`tile ${tile.isObstacle ? 'obstacle' : ''}`}
          style={`--tx:${tile.screen.x}px; --ty:${tile.screen.y}px; z-index:${tile.z};`}
        />
      {/each}

      <div
        class="player"
        style={`--tx:${toIso(player.x, player.y).x}px; --ty:${toIso(player.x, player.y).y}px; z-index:${player.x + player.y + 10};`}
      >
        <div class="shadow" />
        <div class="body">
          <div class="head" />
        </div>
      </div>
    </div>

    <p class="status">{status}</p>
  </section>
</main>
