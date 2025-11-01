<template>
  <div class="app">
    <header class="hud">
      <div class="title">
        <span class="title-main">🌕 행복한 한가위 🌕</span>
        <span class="title-sub">연등 잡기 게임</span>
      </div>

      <div class="stats">
        <div class="stat"><label>점수</label><strong>{{ Math.floor(score) }}</strong></div>
        <div class="stat"><label>콤보</label><strong>x{{ combo }}</strong></div>
        <div class="stat"><label>남은시간</label><strong>{{ timeLeft }}</strong>s</div>
        <div class="stat"><label>기회</label><strong>{{ lives }}</strong></div>
      </div>

      <div class="controls">
        <button v-if="!running && !finished" class="btn primary" @click="startGame">시작</button>
        <button v-else-if="running" class="btn" @click="togglePause">일시정지</button>
        <button v-else class="btn primary" @click="startGame">다시하기</button>
      </div>
    </header>

    <div ref="sceneRef" class="scene">
      <div class="moon"></div>
      <div class="cloud c1"></div>
      <div class="cloud c2"></div>
      <canvas ref="canvasRef" class="game-canvas"
              @pointerdown="onPointerDown"
              @pointermove="onPointerMove"
              @pointerup="onPointerUp"></canvas>

      <div v-if="!running && !finished" class="overlay">
        <p class="hint">연등을 클릭해서 잡아보세요</p>
        <p class="hint-small">연속으로 잡으면 <strong>콤보 보너스!</strong></p>
      </div>

      <div v-if="paused && !finished" class="overlay">
        <p class="hint">⏸ 일시정지</p>
        <p class="hint-small">재개하려면 버튼을 눌러주세요</p>
      </div>

      <div v-if="finished" class="overlay result">
        <p class="final">최종 점수</p>
        <p class="score">{{ Math.floor(score) }}</p>
        <p class="hint-small">최대 콤보 <strong>x{{ maxCombo }}</strong></p>
        <button class="btn primary big" @click="startGame">다시 도전</button>
      </div>
    </div>

    <footer class="footer">
      <p>© 2025 행복한 한가위 with Vue ✨</p>
    </footer>
  </div>
</template>

<script setup lang="ts">
import { onMounted, onBeforeUnmount, ref, reactive } from 'vue';

type Lantern = {
  id: number;
  x: number;
  y: number;
  r: number;
  vx: number;
  vy: number;
  rot: number;
  rotSpeed: number;
  hue: number;
  glow: number;
  caught?: boolean;
};

type Spark = {
  x: number;
  y: number;
  vx: number;
  vy: number;
  life: number;
  maxLife: number;
};

const canvasRef = ref<HTMLCanvasElement | null>(null);
const sceneRef = ref<HTMLDivElement | null>(null);
const running = ref(false);
const paused = ref(false);
const finished = ref(false);

const score = ref(0);
const combo = ref(0);
const maxCombo = ref(0);
const lives = ref(3);
const timeLeft = ref(60);

const state = reactive({
  width: 0,
  height: 0,
  lanterns: [] as Lantern[],
  sparks: [] as Spark[],
  nextId: 1,
  spawnAcc: 0,
  lastTs: 0,
});

let rafId = 0;
// ✅ 타입 안정화 (ReturnType<typeof setInterval>)
let timerId: ReturnType<typeof setInterval> | null = null;

const dpr = () => Math.max(1, Math.min(2, window.devicePixelRatio || 1));

function resizeCanvas() {
  const canvas = canvasRef.value!;
  const scene = sceneRef.value!;
  const rect = scene.getBoundingClientRect();
  state.width = Math.floor(rect.width);
  state.height = Math.floor(rect.height);

  const ratio = dpr();
  canvas.width = Math.floor(state.width * ratio);
  canvas.height = Math.floor(state.height * ratio);
  canvas.style.width = `${state.width}px`;
  canvas.style.height = `${state.height}px`;
}

function startGame() {
  running.value = true;
  paused.value = false;
  finished.value = false;
  score.value = 0;
  combo.value = 0;
  maxCombo.value = 0;
  lives.value = 3;
  timeLeft.value = 60;
  state.lanterns = [];
  state.sparks = [];
  state.nextId = 1;
  state.spawnAcc = 0;
  state.lastTs = performance.now();

  if (timerId) clearInterval(timerId);
  timerId = setInterval(() => {
    if (!running.value || paused.value) return;
    timeLeft.value -= 1;
    if (timeLeft.value <= 0) {
      timeLeft.value = 0;
      endGame();
    }
  }, 1000);

  loop(state.lastTs);
}

function endGame() {
  running.value = false;
  finished.value = true;
  paused.value = false;
  if (timerId) clearInterval(timerId);
}

function togglePause() {
  if (finished.value || !running.value) return;
  paused.value = !paused.value;
  if (!paused.value) {
    state.lastTs = performance.now();
    loop(state.lastTs);
  }
}

function spawnLantern() {
  const w = state.width;
  const lanePadding = Math.max(20, w * 0.04);
  const x = lanePadding + Math.random() * (w - lanePadding * 2);
  const r = clamp(randRange(w * 0.025, w * 0.045), 14, 48);
  const speedBase = lerp(140, 260, difficulty());
  const vy = randRange(speedBase * 0.8, speedBase * 1.2);
  const vx = randRange(-40, 40);
  const rotSpeed = randRange(-2, 2);
  const hue = randChoice([10, 28, 35, 45, 320, 200])!;
  const glow = randRange(0.5, 1);

  state.lanterns.push({
    id: state.nextId++,
    x,
    y: -r - 6,
    r,
    vx,
    vy,
    rot: Math.random() * Math.PI * 2,
    rotSpeed,
    hue,
    glow,
  });
}

function difficulty() {
  const t = 1 - Math.max(0, timeLeft.value) / 60;
  const s = Math.min(1, score.value / 600);
  return clamp(t * 0.7 + s * 0.5, 0, 1);
}

function loop(ts: number) {
  if (!running.value || paused.value) return;
  const canvas = canvasRef.value!;
  const ctx = canvas.getContext('2d')!;
  const ratio = dpr();
  const dt = Math.min(0.032, (ts - state.lastTs) / 1000 || 0);
  state.lastTs = ts;

  ctx.setTransform(ratio, 0, 0, ratio, 0, 0);
  ctx.clearRect(0, 0, state.width, state.height);

  const baseSpawn = lerp(0.55, 0.22, difficulty());
  state.spawnAcc += dt;
  while (state.spawnAcc >= baseSpawn) {
    state.spawnAcc -= baseSpawn;
    spawnLantern();
  }

  // ✅ 안전한 lantern 업데이트
  for (let i = 0; i < state.lanterns.length; i++) {
    const L = state.lanterns[i];
    if (!L) continue;
    L.x += L.vx * dt;
    L.y += L.vy * dt;
    L.rot += L.rotSpeed * dt;
  }

  // ✅ 안전한 제거 로직
  for (let i = state.lanterns.length - 1; i >= 0; i--) {
    const L = state.lanterns[i];
    if (!L) continue;
    if (L.y - L.r > state.height + 8) {
      state.lanterns.splice(i, 1);
      combo.value = 0;
      if (lives.value > 0) {
        lives.value -= 1;
        if (lives.value <= 0) endGame();
      }
    }
  }

  // ✅ 안전한 draw
  for (const L of state.lanterns) {
    if (!L) continue;
    drawLantern(ctx, L);
  }

  updateSparks(dt);
  drawSparks(ctx);

  rafId = requestAnimationFrame(loop);
}

function drawLantern(ctx: CanvasRenderingContext2D, L: Lantern) {
  ctx.save();
  ctx.globalAlpha = 0.35 * L.glow;
  const grad = ctx.createRadialGradient(L.x, L.y, 2, L.x, L.y, L.r * 2.2);
  grad.addColorStop(0, `hsla(${L.hue}, 90%, 70%, 1)`);
  grad.addColorStop(1, `hsla(${L.hue}, 90%, 50%, 0)`);
  ctx.fillStyle = grad;
  ctx.beginPath();
  ctx.arc(L.x, L.y, L.r * 1.8, 0, Math.PI * 2);
  ctx.fill();
  ctx.restore();

  ctx.save();
  ctx.translate(L.x, L.y);
  ctx.rotate(L.rot);

  const bodyGrad = ctx.createLinearGradient(0, -L.r, 0, L.r);
  bodyGrad.addColorStop(0, `hsl(${L.hue}, 90%, 80%)`);
  bodyGrad.addColorStop(1, `hsl(${L.hue}, 90%, 55%)`);
  ctx.fillStyle = bodyGrad;
  roundedRect(ctx, -L.r * 0.9, -L.r, L.r * 1.8, L.r * 2, L.r * 0.45);
  ctx.fill();

  ctx.strokeStyle = `hsla(${L.hue}, 70%, 35%, 0.35)`;
  ctx.lineWidth = Math.max(1, L.r * 0.08);
  for (let i = -3; i <= 3; i++) {
    const yy = (i / 3) * L.r * 0.85;
    ctx.beginPath();
    ctx.ellipse(0, yy, L.r * 0.85, L.r * 0.35, 0, 0, Math.PI * 2);
    ctx.stroke();
  }

  ctx.fillStyle = `hsl(${L.hue}, 60%, 35%)`;
  roundedRect(ctx, -L.r * 0.6, -L.r * 1.15, L.r * 1.2, L.r * 0.25, L.r * 0.08);
  ctx.fill();
  roundedRect(ctx, -L.r * 0.6, L.r * 0.9, L.r * 1.2, L.r * 0.25, L.r * 0.08);
  ctx.fill();

  ctx.strokeStyle = `hsl(${L.hue}, 70%, 30%)`;
  ctx.lineWidth = Math.max(1, L.r * 0.06);
  ctx.beginPath();
  ctx.moveTo(0, L.r * 1.15);
  ctx.lineTo(0, L.r * 1.6);
  ctx.stroke();

  ctx.fillStyle = `hsl(${L.hue}, 80%, 40%)`;
  roundedRect(ctx, -L.r * 0.12, L.r * 1.6, L.r * 0.24, L.r * 0.24, L.r * 0.05);
  ctx.fill();

  ctx.restore();
}

function roundedRect(ctx: CanvasRenderingContext2D, x: number, y: number, w: number, h: number, r: number) {
  const rr = Math.min(r, Math.abs(w) / 2, Math.abs(h) / 2);
  ctx.beginPath();
  ctx.moveTo(x + rr, y);
  ctx.arcTo(x + w, y, x + w, y + h, rr);
  ctx.arcTo(x + w, y + h, x, y + h, rr);
  ctx.arcTo(x, y + h, x, y, rr);
  ctx.arcTo(x, y, x + w, y, rr);
  ctx.closePath();
}

function onPointerDown(e: PointerEvent) {
  if (!running.value || paused.value) return;
  const pos = getPointerPos(e);
  let hit = false;
  let bestIdx = -1;
  let bestD2 = Number.POSITIVE_INFINITY;

  for (let i = 0; i < state.lanterns.length; i++) {
    const L = state.lanterns[i];
    if (!L) continue;
    const dx = pos.x - L.x;
    const dy = pos.y - L.y;
    const d2 = dx * dx + dy * dy;
    if (d2 <= (L.r * 0.95) ** 2 && d2 < bestD2) {
      bestD2 = d2;
      bestIdx = i;
    }
  }

  if (bestIdx >= 0) {
    hit = true;
    const L = state.lanterns[bestIdx];
    if (L) {
      popLantern(L);
      state.lanterns.splice(bestIdx, 1);
      combo.value += 1;
      maxCombo.value = Math.max(maxCombo.value, combo.value);
      const base = 20;
      const bonus = base * (1 + (combo.value - 1) * 0.15);
      score.value += bonus;
    }
  }

  if (!hit) combo.value = 0;
}

function onPointerMove(_: PointerEvent) {}
function onPointerUp(_: PointerEvent) {}

function popLantern(L: Lantern) {
  const k = 18;
  for (let i = 0; i < k; i++) {
    const a = (i / k) * Math.PI * 2 + Math.random() * 0.4;
    const s = randRange(90, 220);
    state.sparks.push({
      x: L.x,
      y: L.y,
      vx: Math.cos(a) * s,
      vy: Math.sin(a) * s,
      life: 0,
      maxLife: randRange(0.45, 0.8),
    });
  }
}

function updateSparks(dt: number) {
  const g = 420;
  for (const p of state.sparks) {
    p.life += dt;
    p.x += p.vx * dt;
    p.y += p.vy * dt + g * dt * dt * 0.5;
    p.vy += g * dt;
  }
  state.sparks = state.sparks.filter((p) => p.life < p.maxLife);
}

function drawSparks(ctx: CanvasRenderingContext2D) {
  for (const p of state.sparks) {
    const t = p.life / p.maxLife;
    const a = 1 - t;
    ctx.globalAlpha = a;
    ctx.fillStyle = `hsla(${lerp(20, 55, Math.random())}, 100%, ${lerp(65, 80, 1 - t)}%, ${a})`;
    ctx.beginPath();
    ctx.arc(p.x, p.y, lerp(3.5, 0.8, t), 0, Math.PI * 2);
    ctx.fill();
    ctx.globalAlpha = 1;
  }
}

function getPointerPos(e: PointerEvent) {
  const rect = canvasRef.value!.getBoundingClientRect();
  return { x: e.clientX - rect.left, y: e.clientY - rect.top };
}

function randRange(min: number, max: number) {
  return min + Math.random() * (max - min);
}
function randChoice<T>(arr: T[]) {
  return arr[Math.floor(Math.random() * arr.length)];
}
function clamp(v: number, a: number, b: number) {
  return Math.max(a, Math.min(b, v));
}
function lerp(a: number, b: number, t: number) {
  return a + (b - a) * t;
}

onMounted(() => {
  resizeCanvas();
  window.addEventListener('resize', resizeCanvas, { passive: true });
  state.lastTs = performance.now();
  const ctx = canvasRef.value!.getContext('2d')!;
  ctx.clearRect(0, 0, canvasRef.value!.width, canvasRef.value!.height);
});

onBeforeUnmount(() => {
  cancelAnimationFrame(rafId);
  if (timerId) clearInterval(timerId);
  window.removeEventListener('resize', resizeCanvas);
});
</script>


<style scoped>
@import url('https://cdn.jsdelivr.net/gh/orioncactus/pretendard/dist/web/static/pretendard.css');

.app {
  font-family: 'Pretendard', 'Noto Sans KR', sans-serif;
  text-align: center;
  background: linear-gradient(to top, #1a1a2e, #16213e);
  color: #fff;
  min-height: 100vh;
  display: flex;
  flex-direction: column;
}

/* 상단 HUD */
.hud {
  display: grid;
  grid-template-columns: 1fr auto auto;
  align-items: center;
  padding: 1.6rem 2rem; /* 🔹 상단 여백 확장 */
  backdrop-filter: blur(8px);
  background: rgba(20, 20, 40, 0.45);
  border-bottom: 1px solid rgba(255,255,255,0.15);
}

.title {
  display: flex;
  flex-direction: column;
  align-items: flex-start;
  gap: .4rem;
}
.title-main {
  font-size: 2.2rem; /* 🔹 글씨 확대 */
  font-weight: 800;
  color: #ffda79;
  text-shadow: 0 0 10px #ffda79;
}
.title-sub {
  font-size: 1.2rem;
  color: #ffeeb5;
}

.stats, .controls {
  display: flex;
  gap: 1.2rem; /* 🔹 버튼 간 간격 확대 */
  flex-wrap: wrap;
  padding-left: 1.5rem; /* 🔹 왼쪽 여백 추가 */
}


.stat {
  background: rgba(255,255,255,0.08);
  padding: 0.6rem 1rem;
  border-radius: 0.8rem;
}
.stat label {
  display: block;
  font-size: 0.75rem;
  color: #ffeeb5;
}
.stat strong {
  font-size: 1.1rem;
  color: #fff8d6;
}

/* 버튼 */
.btn {
  font-family: 'Pretendard', sans-serif;
  padding: 0.9rem 1.4rem; /* 🔹 버튼 크기 확대 */
  border-radius: 0.9rem;
  border: 1px solid rgba(255,255,255,0.25);
  font-size: 1.05rem;
  color: #fff;
  background: rgba(255,255,255,0.08);
  cursor: pointer;
  transition: all 0.2s ease;
}
.btn:hover {
  transform: translateY(-2px);
  background: rgba(255,255,255,0.15);
}
.btn.primary {
  background: linear-gradient(90deg, #ffcc66, #ff9f50);
  color: #3b2500;
  font-weight: 700;
  border: none;
  box-shadow: 0 0 16px rgba(255,190,60,0.45);
}
.btn.primary.big {
  font-size: 1.3rem;
  padding: 1rem 1.8rem;
  border-radius: 1rem;
}

/* 배경/연등/문양 */
.scene {
  position: relative;
  flex: 1;
  overflow: hidden;
  background: radial-gradient(ellipse at top, #243b55, #141e30);
  height: calc(100vh - 120px);
}

.moon {
  position: absolute;
  top: 8%;
  right: 10%;
  width: 120px;
  height: 120px;
  border-radius: 50%;
  background: radial-gradient(circle at 30% 30%, #fff8c6, #ffd966 70%);
  box-shadow: 0 0 40px 10px #ffeb8a;
  animation: glow 3s ease-in-out infinite alternate;
}
@keyframes glow {
  from { box-shadow: 0 0 20px 5px #ffd966; }
  to { box-shadow: 0 0 60px 20px #fff8c6; }
}

/* 오버레이 */
.overlay {
  position: absolute;
  inset: 0;
  display: grid;
  place-items: center;
  color: #ffeeb5;
  text-shadow: 0 0 10px rgba(255,255,200,0.4);
}
.hint {
  font-size: 1.8rem;
  font-weight: 700;
}
.hint-small {
  font-size: 1rem;
  color: #fff7d6;
}
.score {
  font-size: 3rem;
  font-weight: 800;
  color: #ffda79;
  text-shadow: 0 0 12px #ffda79;
}

.footer {
  padding: 1.2rem;
  font-size: 1rem;
  color: #ffeeb5;
  border-top: 1px solid rgba(255,255,255,0.1);
  background: rgba(0,0,0,0.25);
}
</style>
