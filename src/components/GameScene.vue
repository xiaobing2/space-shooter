<template>
  <div ref="gameContainer" class="game-container" @click="handleClick">
    <canvas ref="gameCanvas" class="game-canvas"></canvas>
    
    <!-- 游戏HUD -->
    <div class="hud" v-if="gameState === 'playing'">
      <div class="score">分数: {{ score }}</div>
      <div class="health">生命: {{ health }}/100</div>
      <div class="ammo">子弹: {{ maxBullets - bullets.length }}/{{ maxBullets }}</div>
      <div class="enemies">敌人: {{ enemies.length }}</div>
    </div>
    
    <!-- 开始界面 -->
    <div class="start-screen" v-if="gameState === 'start'">
      <h1>太空射击</h1>
      <p>使用 WASD 或方向键移动</p>
      <p>点击鼠标左键或空格键射击</p>
      <button @click="startGame">开始游戏</button>
    </div>
    
    <!-- 游戏结束界面 -->
    <div class="game-over" v-if="gameState === 'gameover'">
      <h1>游戏结束</h1>
      <p>最终得分: {{ score }}</p>
      <button @click="restartGame">再玩一次</button>
    </div>
  </div>
</template>

<script setup lang="ts">
import { ref, onMounted, onUnmounted } from 'vue';
import * as THREE from 'three';
import * as CANNON from 'cannon';

// DOM 引用
const gameContainer = ref<HTMLElement | null>(null);
const gameCanvas = ref<HTMLCanvasElement | null>(null);

// 游戏状态
const gameState = ref<'start' | 'playing' | 'gameover'>('start');
const score = ref(0);
const health = ref(150); // 调低难度：提升初始生命
const maxBullets = 10;   // 提升弹药上限
const bullets = ref<Array<{mesh: THREE.Mesh, body: CANNON.Body, lifetime: number}>>([]);
const enemies = ref<Array<{mesh: THREE.Mesh, body: CANNON.Body, health: number}>>([]);
const explosions = ref<Array<{mesh: THREE.Points, lifetime: number}>>([]);

// 场景和物理引擎
let scene: THREE.Scene;
let camera: THREE.PerspectiveCamera;
let renderer: THREE.WebGLRenderer;
let world: CANNON.World;
let lastTime = 0;
let player: THREE.Mesh;
let playerBody: CANNON.Body;
let playerRotation = 0;
let lastEnemySpawn = 0;
const enemySpawnInterval = 4000; // 放慢节奏：每4秒生成一批敌人
let gameLoopId: number;

const keys: Record<string, boolean> = {};

// 游戏流程控制
const startGame = () => {
  resetGame();
  gameState.value = 'playing';
  lastTime = performance.now();
  lastEnemySpawn = lastTime;
  gameLoop();
};

const restartGame = () => {
  cleanupGame();
  startGame();
};

const gameOver = () => {
  gameState.value = 'gameover';
  cancelAnimationFrame(gameLoopId);
};

const resetGame = () => {
  score.value = 0;
  health.value = 100;
  
  if (playerBody) {
    playerBody.position.set(0, 0, 0);
    playerBody.velocity.set(0, 0, 0);
    playerBody.angularVelocity.set(0, 0, 0);
  }
  cleanupGame();
};

const cleanupGame = () => {
  bullets.value.forEach(b => { scene.remove(b.mesh); world.removeBody(b.body); });
  bullets.value = [];
  enemies.value.forEach(e => { scene.remove(e.mesh); world.removeBody(e.body); });
  enemies.value = [];
  explosions.value.forEach(e => scene.remove(e.mesh));
  explosions.value = [];
};

// 游戏对象创建
const createExplosion = (position: THREE.Vector3) => {
  const geometry = new THREE.BufferGeometry();
  const positions = new Float32Array(100 * 3);
  for (let i = 0; i < 100 * 3; i++) {
    positions[i] = (Math.random() - 0.5) * 2;
  }
  geometry.setAttribute('position', new THREE.BufferAttribute(positions, 3));
  const material = new THREE.PointsMaterial({ size: 0.2, color: 0xffa500, transparent: true, blending: THREE.AdditiveBlending });
  const points = new THREE.Points(geometry, material);
  points.position.copy(position);
  scene.add(points);
  explosions.value.push({ mesh: points, lifetime: 1.0 });
};

const createBullet = (position: THREE.Vector3, direction: THREE.Vector3) => {
  const geometry = new THREE.SphereGeometry(0.1, 8, 8);
  const material = new THREE.MeshBasicMaterial({ color: 0xffff00 });
  const mesh = new THREE.Mesh(geometry, material);
  mesh.position.copy(position);
  scene.add(mesh);

  const shape = new CANNON.Sphere(0.1);
  const body = new CANNON.Body({ mass: 0.1, position: new CANNON.Vec3().copy(position as any), shape, linearDamping: 0 });
  const speed = 20;
  body.velocity.set(direction.x * speed, direction.y * speed, 0);
  world.addBody(body);
  bullets.value.push({ mesh, body, lifetime: 3.0 }); // 子弹寿命稍长，提升可玩性
};

const createEnemy = () => {
  // 控制场上最大敌人数量，避免难度过高
  if (enemies.value.length >= 8) return;

  const angle = Math.random() * Math.PI * 2;
  const distance = 20;
  const x = Math.cos(angle) * distance;
  const y = Math.sin(angle) * distance;

  const geometry = new THREE.ConeGeometry(0.6, 1.2, 4);
  const material = new THREE.MeshPhongMaterial({ color: 0xff0000, flatShading: true });
  const mesh = new THREE.Mesh(geometry, material);
  mesh.position.set(x, y, 0);
  scene.add(mesh);

  const shape = new CANNON.Box(new CANNON.Vec3(0.6, 0.6, 0.6));
  const body = new CANNON.Body({ mass: 1, position: new CANNON.Vec3(x, y, 0), shape, linearDamping: 0.4 }); // 提高阻尼，减速
  world.addBody(body);
  enemies.value.push({ mesh, body, health: 1 });
};

// 玩家动作
const shoot = () => {
  if (bullets.value.length >= maxBullets) return;
  const direction = new THREE.Vector3(Math.sin(playerRotation), -Math.cos(playerRotation), 0).normalize();
  const startPos = new THREE.Vector3().copy(player.position).add(direction.multiplyScalar(1));
  createBullet(startPos, direction);
};

const handleClick = () => {
  if (gameState.value === 'start') startGame();
  else if (gameState.value === 'gameover') restartGame();
  else shoot();
};

// 游戏循环与更新
const updateBullets = (dt: number) => {
  for (let i = bullets.value.length - 1; i >= 0; i--) {
    const b = bullets.value[i];
    b.lifetime -= dt;
    b.mesh.position.copy(b.body.position as any);
    if (b.lifetime <= 0 || Math.abs(b.body.position.x) > 25 || Math.abs(b.body.position.y) > 25) {
      scene.remove(b.mesh);
      world.removeBody(b.body);
      bullets.value.splice(i, 1);
    }
  }
};

const updateEnemies = (dt: number) => {
  if (performance.now() - lastEnemySpawn > enemySpawnInterval) {
    createEnemy();
    lastEnemySpawn = performance.now();
  }
  for (const e of enemies.value) {
    e.mesh.position.copy(e.body.position as any);
    const dir = new THREE.Vector3().subVectors(player.position, e.mesh.position);
    const len = dir.length();
    if (len > 0.0001) {
      dir.normalize();
    }
    const enemySpeed = 1.1; // 降低敌人速度
    e.body.velocity.set(dir.x * enemySpeed, dir.y * enemySpeed, 0);
    e.mesh.quaternion.setFromUnitVectors(new THREE.Vector3(0, 1, 0), dir);
  }
};

const updateExplosions = (dt: number) => {
  for (let i = explosions.value.length - 1; i >= 0; i--) {
    const e = explosions.value[i];
    e.lifetime -= dt;
    (e.mesh.material as THREE.PointsMaterial).opacity = e.lifetime;
    if (e.lifetime <= 0) {
      scene.remove(e.mesh);
      explosions.value.splice(i, 1);
    }
  }
};

const checkCollisions = () => {
  for (let i = bullets.value.length - 1; i >= 0; i--) {
    for (let j = enemies.value.length - 1; j >= 0; j--) {
      const b = bullets.value[i];
      const e = enemies.value[j];
            if (b && e && b.body.position.distanceTo(e.body.position) < 1) {
        createExplosion(e.mesh.position);
        score.value += 10;
        scene.remove(e.mesh); world.removeBody(e.body); enemies.value.splice(j, 1);
        scene.remove(b.mesh); world.removeBody(b.body); bullets.value.splice(i, 1);
        break;
      }
    }
  }
  for (let i = enemies.value.length - 1; i >= 0; i--) {
    const e = enemies.value[i];
        if (e.body.position.distanceTo(playerBody.position) < 1.2) {
      createExplosion(e.mesh.position);
      health.value -= 20;
      scene.remove(e.mesh); world.removeBody(e.body); enemies.value.splice(i, 1);
      if (health.value <= 0) { health.value = 0; gameOver(); }
    }
  }
};

const updatePlayer = (dt: number) => {
  const speed = 10;
  const moveVector = new THREE.Vector3();
  if (keys['KeyW'] || keys['ArrowUp']) moveVector.y += 1;
  if (keys['KeyS'] || keys['ArrowDown']) moveVector.y -= 1;
  if (keys['KeyA'] || keys['ArrowLeft']) moveVector.x -= 1;
  if (keys['KeyD'] || keys['ArrowRight']) moveVector.x += 1;
  moveVector.normalize().multiplyScalar(speed);
  playerBody.velocity.set(moveVector.x, moveVector.y, 0);
  player.position.copy(playerBody.position as any);
  if (moveVector.lengthSq() > 0) {
    playerRotation = Math.atan2(moveVector.x, -moveVector.y);
    player.rotation.z = playerRotation;
  }
};

const gameLoop = () => {
  const now = performance.now();
  const dt = (now - lastTime) / 1000;
  lastTime = now;

  world.step(1 / 60, dt, 3);

  updatePlayer(dt);
  updateBullets(dt);
  updateEnemies(dt);
  updateExplosions(dt);
  checkCollisions();

  renderer.render(scene, camera);
  gameLoopId = requestAnimationFrame(gameLoop);
};

// 初始化
onMounted(() => {
  // Scene setup
  scene = new THREE.Scene();
  camera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 0.1, 1000);
  camera.position.z = 15;
  renderer = new THREE.WebGLRenderer({ canvas: gameCanvas.value!, antialias: true });
  renderer.setSize(window.innerWidth, window.innerHeight);
  renderer.setPixelRatio(window.devicePixelRatio);

  // Physics setup
  world = new CANNON.World({ gravity: new CANNON.Vec3(0, 0, 0) });

  // Lights
  scene.add(new THREE.AmbientLight(0x404040));
  const dirLight = new THREE.DirectionalLight(0xffffff, 1);
  dirLight.position.set(5, 5, 5);
  scene.add(dirLight);

  // Player setup
  const playerGeo = new THREE.ConeGeometry(0.5, 1, 4);
  const playerMat = new THREE.MeshPhongMaterial({ color: 0x00ff00, flatShading: true });
  player = new THREE.Mesh(playerGeo, playerMat);
  scene.add(player);
  const playerShape = new CANNON.Box(new CANNON.Vec3(0.5, 0.5, 0.5));
  playerBody = new CANNON.Body({ mass: 1, position: new CANNON.Vec3(0, 0, 0), shape: playerShape, linearDamping: 0.8 });
  world.addBody(playerBody);

  // Event Listeners
  window.addEventListener('resize', () => {
    camera.aspect = window.innerWidth / window.innerHeight;
    camera.updateProjectionMatrix();
    renderer.setSize(window.innerWidth, window.innerHeight);
  });
  window.addEventListener('keydown', (e) => { keys[e.code] = true; if (e.code === 'Space') shoot(); });
  window.addEventListener('keyup', (e) => { keys[e.code] = false; });
});

onUnmounted(() => {
  cancelAnimationFrame(gameLoopId);
  // Cleanup listeners if needed
});
</script>

<style scoped>
.game-container {
  position: relative;
  width: 100%;
  height: 100vh;
  overflow: hidden;
  cursor: crosshair;
  background-color: #000;
}

.game-canvas {
  display: block;
}

.hud, .start-screen, .game-over {
  position: absolute;
  width: 100%;
  color: white;
  font-family: 'Arial', sans-serif;
  text-align: center;
  z-index: 10;
}

.hud {
  top: 20px;
  left: 20px;
  text-align: left;
}

.hud > div {
  margin-bottom: 10px;
  padding: 5px 10px;
  background-color: rgba(0, 0, 0, 0.5);
  border-radius: 4px;
}

.start-screen, .game-over {
  top: 0; left: 0; height: 100%;
  display: flex;
  flex-direction: column;
  justify-content: center;
  align-items: center;
  background-color: rgba(0, 0, 0, 0.7);
}

h1 { font-size: 4em; text-shadow: 0 0 10px #00ffff; }
p { font-size: 1.5em; }

button {
  margin-top: 30px; padding: 12px 30px; font-size: 1.2em;
  color: white; background-color: #0066ff; border: none;
  border-radius: 5px; cursor: pointer; transition: all 0.3s;
}

button:hover { background-color: #0088ff; transform: scale(1.05); }
</style>