<script>
  import { onDestroy, onMount } from 'svelte';

  const worldCols = 24;
  const worldRows = 24;
  const viewSize = 8;
  const tileW = 96;
  const tileH = 48;
  const JUMP_DURATION = 280;
  const SHOT_DURATION = 250;
  const SHOT_LENGTH = 260;

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
  const holePositions = new Set();

  function addObstacle(x, y) {
    obstaclePositions.add(`${x},${y}`);
  }

  function addHole(x, y) {
    holePositions.add(`${x},${y}`);
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

  // Carve out a few holes.
  [
    [9, 9],
    [4, 14],
    [15, 8],
    [19, 19],
    [8, 4]
  ].forEach(([x, y]) => addHole(x, y));

  let player = { x: 1, y: 1 };
  let status = 'Use WASD or arrow keys to move';
  let gameOver = false;
  let isJumping = false;
  let lastMoveDir = null;
  let jumpTimeout;
  let shots = [];
  let shotTimeouts = new Map();
  let aimPos = { x: 0, y: 0 };
  let aimAngle = 0;
  let worldEl;
  let enemies = [
    { id: 'e1', x: 10, y: 10, stunnedUntil: 0 },
    { id: 'e2', x: 18, y: 6, stunnedUntil: 0 },
    { id: 'e3', x: 6, y: 20, stunnedUntil: 0 }
  ];
  let enemyInterval;
  let viewOriginX = 0;
  let viewOriginY = 0;
  let viewTiles = [];
  let playerLocal = { x: 0, y: 0 };

  function isBlocked(x, y) {
    if (x < 0 || y < 0 || x >= worldCols || y >= worldRows) return true;
    return obstaclePositions.has(`${x},${y}`);
  }

  function isHole(x, y) {
    return holePositions.has(`${x},${y}`);
  }

  function inBounds(x, y) {
    return x >= 0 && y >= 0 && x < worldCols && y < worldRows;
  }

  function attemptMove(dx, dy) {
    if (gameOver) return;
    const nextX = player.x + dx;
    const nextY = player.y + dy;
    if (isBlocked(nextX, nextY)) {
      status = 'You bumped into an obstacle';
      return;
    }
    if (isHole(nextX, nextY)) {
      gameOver = true;
      status = 'Game Over';
      return;
    }
    player = { x: nextX, y: nextY };
    lastMoveDir = [dx, dy];
    refreshView();
    status = 'Walking...';
  }

  function attemptJump() {
    if (gameOver) return;
    if (!lastMoveDir) {
      status = 'Jump after moving to set a direction';
      return;
    }
    const [dx, dy] = lastMoveDir;
    const midX = player.x + dx;
    const midY = player.y + dy;
    const landingX = player.x + dx * 2;
    const landingY = player.y + dy * 2;

    if (isBlocked(midX, midY) || isBlocked(landingX, landingY)) {
      status = 'Too tall to jump over';
      return;
    }
    if (landingX < 0 || landingY < 0 || landingX >= worldCols || landingY >= worldRows) {
      status = 'Edge is too far';
      return;
    }
    if (isHole(landingX, landingY)) {
      status = 'Cannot land in a hole';
      return;
    }

    player = { x: landingX, y: landingY };
    refreshView();
    status = isHole(midX, midY) ? 'You leapt over a hole!' : 'Jump!';

    isJumping = true;
    clearTimeout(jumpTimeout);
    jumpTimeout = setTimeout(() => {
      isJumping = false;
    }, JUMP_DURATION);
  }

  function handleKey(event) {
    const key = event.key.toLowerCase();
    if (key === ' ' || event.code === 'Space') {
      event.preventDefault();
      attemptJump();
      return;
    }

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

  function handleMouseMove(event) {
    updateAimFromMouse(event.clientX, event.clientY);
  }

  function handleMouseDown(event) {
    if (event.button === 0) {
      shoot(event.clientX, event.clientY);
    }
  }

  function chasePlayer() {
    if (gameOver) return;
    enemies = enemies
      .map((enemy) => {
        if (Date.now() < enemy.stunnedUntil) return enemy;
        const dx = Math.sign(player.x - enemy.x);
        const dy = Math.sign(player.y - enemy.y);
        if (dx === 0 && dy === 0) return enemy;
        const targetX = enemy.x + dx;
        const targetY = enemy.y + dy;
        if (!inBounds(targetX, targetY)) return enemy;
        if (isBlocked(targetX, targetY)) return enemy;
        if (isHole(targetX, targetY)) return null;
        return { ...enemy, x: targetX, y: targetY };
      })
      .filter(Boolean);
  }

  onMount(() => {
    window.addEventListener('keydown', handleKey);
    enemyInterval = setInterval(chasePlayer, 900);
  });

  onDestroy(() => {
    window.removeEventListener('keydown', handleKey);
    clearTimeout(jumpTimeout);
    shotTimeouts.forEach((id) => clearTimeout(id));
    clearInterval(enemyInterval);
  });

  function viewStart(center, maxStart) {
    return Math.max(0, Math.min(center - Math.floor(viewSize / 2) + 1, maxStart));
  }

  function refreshView() {
    viewOriginX = viewStart(player.x, maxViewStartX);
    viewOriginY = viewStart(player.y, maxViewStartY);

    const nextTiles = [];
    for (let localY = 0; localY < viewSize; localY += 1) {
      for (let localX = 0; localX < viewSize; localX += 1) {
        const worldX = viewOriginX + localX;
        const worldY = viewOriginY + localY;

        nextTiles.push({
          id: `${worldX}-${worldY}`,
          localX,
          localY,
          isObstacle: obstaclePositions.has(`${worldX},${worldY}`),
          isHole: holePositions.has(`${worldX},${worldY}`),
          screen: toIsoLocal(localX, localY),
          z: localX + localY
        });
      }
    }
    viewTiles = nextTiles;
    playerLocal = {
      x: player.x - viewOriginX,
      y: player.y - viewOriginY
    };
  }

  function enemyScreenCenter(enemy) {
    const localX = enemy.x - viewOriginX;
    const localY = enemy.y - viewOriginY;
    const iso = toIsoLocal(localX, localY);
    return {
      x: iso.x + tileW / 2,
      y: iso.y + tileH / 2
    };
  }

  function playerScreenCenter() {
    const iso = toIsoLocal(playerLocal.x, playerLocal.y);
    return {
      x: iso.x + tileW / 2,
      y: iso.y + tileH / 2
    };
  }

  function muzzlePoint(angle) {
    const center = playerScreenCenter();
    const offset = 24;
    return {
      x: center.x + Math.cos(angle) * offset,
      y: center.y + Math.sin(angle) * offset
    };
  }

  function updateAimFromMouse(clientX, clientY) {
    if (!worldEl) return;
    const rect = worldEl.getBoundingClientRect();
    aimPos = { x: clientX - rect.left, y: clientY - rect.top };
    const center = playerScreenCenter();
    aimAngle = Math.atan2(aimPos.y - center.y, aimPos.x - center.x);
  }

  function shoot(clientX, clientY) {
    if (gameOver) return;
    updateAimFromMouse(clientX, clientY);
    const start = muzzlePoint(aimAngle);
    const dx = Math.cos(aimAngle);
    const dy = Math.sin(aimAngle);
    const shot = {
      id: crypto.randomUUID(),
      x: start.x,
      y: start.y,
      angle: aimAngle,
      length: SHOT_LENGTH,
      dx,
      dy
    };
    shots = [...shots, shot];
    handleShotHits(shot);
    const timeout = setTimeout(() => {
      shots = shots.filter((s) => s.id !== shot.id);
      shotTimeouts.delete(shot.id);
    }, SHOT_DURATION);
    shotTimeouts.set(shot.id, timeout);
    status = 'Bang!';
  }

  function knockbackEnemy(enemy, angle) {
    const stepX = Math.sign(Math.cos(angle));
    const stepY = Math.sign(Math.sin(angle));
    if (stepX === 0 && stepY === 0) return enemy;

    const targetX = enemy.x + stepX;
    const targetY = enemy.y + stepY;
    if (!inBounds(targetX, targetY)) return enemy;
    if (isBlocked(targetX, targetY)) return { ...enemy, stunnedUntil: Date.now() + 1000 };
    if (isHole(targetX, targetY)) {
      enemies = enemies.filter((e) => e.id !== enemy.id);
      return null;
    }
    return { ...enemy, x: targetX, y: targetY, stunnedUntil: Date.now() + 1000 };
  }

  function handleShotHits(shot) {
    const vx = Math.cos(shot.angle);
    const vy = Math.sin(shot.angle);
    const len = shot.length;
    const hitRadius = 18;

    enemies = enemies
      .map((enemy) => {
        const center = enemyScreenCenter(enemy);
        const px = center.x - shot.x;
        const py = center.y - shot.y;
        const proj = px * vx + py * vy;
        if (proj < 0 || proj > len) return enemy;
        const dist = Math.abs(px * vy - py * vx);
        if (dist > hitRadius) return enemy;
        const knocked = knockbackEnemy(enemy, shot.angle);
        return knocked ?? null;
      })
      .filter(Boolean);
  }

  refreshView();
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
    {#if gameOver}
      <div class="game-over">Game Over</div>
    {:else}
      <div class="hud">
        <span class="pill">WASD / Arrow keys</span>
        <span class="pill">Obstacles block movement</span>
        <span class="pill">Space to jump</span>
        <span class="pill">Holes end the run</span>
        <span class="pill">Only nearby 8x8 tiles render</span>
      </div>

      <div
        class="world"
        style={`--tile-w:${tileW}px; --tile-h:${tileH}px; --cols:${viewSize}; --rows:${viewSize};`}
        bind:this={worldEl}
        on:mousemove={handleMouseMove}
        on:mousedown={handleMouseDown}
        role="presentation"
        aria-label="Isometric world"
      >
        {#each viewTiles as tile (tile.id)}
          <div
            class={`tile ${tile.isObstacle ? 'obstacle' : ''} ${tile.isHole ? 'hole' : ''}`}
            style={`--tx:${tile.screen.x}px; --ty:${tile.screen.y}px; z-index:${tile.z};`}
        />
      {/each}

        {#each shots as shot (shot.id)}
          <div
            class="shot"
            style={`left:${shot.x}px; top:${shot.y}px; width:${shot.length}px; transform:rotate(${shot.angle}rad);`}
          />
        {/each}

        {#each enemies as enemy (enemy.id)}
          {#if enemy.x >= viewOriginX && enemy.x < viewOriginX + viewSize && enemy.y >= viewOriginY && enemy.y < viewOriginY + viewSize}
            <div
              class="enemy"
              style={`--tx:${toIsoLocal(enemy.x - viewOriginX, enemy.y - viewOriginY).x}px; --ty:${toIsoLocal(enemy.x - viewOriginX, enemy.y - viewOriginY).y}px; z-index:${enemy.x + enemy.y + 8};`}
            >
              <div class="enemy-body" />
            </div>
          {/if}
        {/each}

        <div
          class={`player ${isJumping ? 'jumping' : ''}`}
          style={`--tx:${toIsoLocal(playerLocal.x, playerLocal.y).x}px; --ty:${toIsoLocal(playerLocal.x, playerLocal.y).y}px; z-index:${playerLocal.x + playerLocal.y + 10};`}
        >
          <div class="shadow" />
          <div class="body">
            <div class="gun" style={`transform: translate(-50%, -50%) rotate(${aimAngle}rad);`} />
            <div class="head" />
          </div>
        </div>
      </div>

      <p class="status">{status}</p>
    {/if}
  </section>
</main>
