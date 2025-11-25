<script>
  import { onDestroy, onMount } from 'svelte';

  const worldCols = 24;
  const worldRows = 24;
  const viewSize = 8;
  const tileW = 96;
  const tileH = 48;

  const maxViewStartX = worldCols - viewSize;
  const maxViewStartY = worldRows - viewSize;
  const offsetX = ((viewSize - 1) * tileW) / 2;

  function toIsoLocal(localX, localY) {
    return {
      x: (localX - localY) * (tileW / 2) + offsetX,
      y: (localX + localY) * (tileH / 2)
    };
  }

  const obstaclePositions = new Set();

  function addObstacle(x, y) {
    obstaclePositions.add(`${x},${y}`);
  }

  // Generate a few clusters of rocks across the larger map.
  const clusters = [
    [
      [5, 5],
      [6, 5],
      [7, 5],
      [7, 6],
      [8, 6]
    ],
    [
      [12, 10],
      [12, 11],
      [13, 11],
      [14, 11],
      [15, 11],
      [15, 10]
    ],
    [
      [18, 4],
      [18, 5],
      [19, 5],
      [20, 5],
      [20, 6]
    ],
    [
      [6, 16],
      [7, 16],
      [8, 16],
      [8, 17],
      [9, 17],
      [10, 17]
    ],
    [
      [16, 16],
      [17, 15],
      [17, 16],
      [17, 17],
      [18, 17],
      [19, 17]
    ]
  ];
  clusters.forEach((cluster) => cluster.forEach(([x, y]) => addObstacle(x, y)));

  // Scatter some standalone rocks.
  [
    [3, 12],
    [9, 3],
    [14, 6],
    [4, 18],
    [11, 19],
    [20, 12],
    [2, 9]
  ].forEach(([x, y]) => addObstacle(x, y));

  let player = { x: 1, y: 1 };
  let status = 'Use WASD or arrow keys to move';

  function isBlocked(x, y) {
    if (x < 0 || y < 0 || x >= worldCols || y >= worldRows) return true;
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

  function viewStart(center, maxStart) {
    return Math.max(0, Math.min(center - Math.floor(viewSize / 2) + 1, maxStart));
  }

  $: viewOriginX = viewStart(player.x, maxViewStartX);
  $: viewOriginY = viewStart(player.y, maxViewStartY);

  function buildViewTiles() {
    const tiles = [];
    for (let localY = 0; localY < viewSize; localY += 1) {
      for (let localX = 0; localX < viewSize; localX += 1) {
        const worldX = viewOriginX + localX;
        const worldY = viewOriginY + localY;

        tiles.push({
          id: `${worldX}-${worldY}`,
          localX,
          localY,
          isObstacle: obstaclePositions.has(`${worldX},${worldY}`),
          screen: toIsoLocal(localX, localY),
          z: localX + localY
        });
      }
    }
    return tiles;
  }

  $: viewTiles = buildViewTiles();
  $: playerLocal = {
    x: player.x - viewOriginX,
    y: player.y - viewOriginY
  };
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
      <span class="pill">Only nearby 8x8 tiles render</span>
    </div>

    <div
      class="world"
      style={`--tile-w:${tileW}px; --tile-h:${tileH}px; --cols:${viewSize}; --rows:${viewSize};`}
    >
      {#each viewTiles as tile (tile.id)}
        <div
          class={`tile ${tile.isObstacle ? 'obstacle' : ''}`}
          style={`--tx:${tile.screen.x}px; --ty:${tile.screen.y}px; z-index:${tile.z};`}
        />
      {/each}

      <div
        class="player"
        style={`--tx:${toIsoLocal(playerLocal.x, playerLocal.y).x}px; --ty:${toIsoLocal(playerLocal.x, playerLocal.y).y}px; z-index:${playerLocal.x + playerLocal.y + 10};`}
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
