<template>
  <div class="game-container" @click="handleClick">
    <canvas ref="gameCanvas" class="game-canvas"></canvas>
    
    <!-- 游戏HUD -->
    <div class="hud" v-if="gameState === 'playing'">
      <div class="score">分数: {{ score }}</div>
      <div class="health">生命: {{ health }}/150</div>
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
const gameCanvas = ref<HTMLCanvasElement | null>(null);

// 游戏状态
const gameState = ref<'start' | 'playing' | 'gameover'>('start');
const score = ref(0);
const health = ref(150);
const maxBullets = 10;
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
const enemySpawnInterval = 4000;
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
  health.value = 150;
  
  if (playerBody) {
    playerBody.position.set(0, 0, 0);
    playerBody.velocity.set(0, 0, 0);
    playerBody.angularVelocity.set(0, 0, 0);
  }
  cleanupGame();
};

const cleanupGame = () => {
  bullets.value.forEach(b => { 
    scene.remove(b.mesh); 
    world.remove(b.body); 
  });
  bullets.value = [];
  
  enemies.value.forEach(e => { 
    scene.remove(e.mesh); 
    world.remove(e.body); 
  });
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
  const body = new CANNON.Body({ 
    mass: 0.1, 
    position: new CANNON.Vec3(position.x, position.y, 0), 
    shape, 
    linearDamping: 0 
  });
  
  const speed = 20;
  body.velocity.set(direction.x * speed, direction.y * speed, 0);
  world.addBody(body);
  bullets.value.push({ mesh, body, lifetime: 3.0 });
};

const createEnemy = () => {
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
  const body = new CANNON.Body({ 
    mass: 1, 
    position: new CANNON.Vec3(x, y, 0), 
    shape, 
    linearDamping: 0.4 
  });
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
    const bullet = bullets.value[i];
    if (!bullet) continue;
    
    bullet.lifetime -= dt;
    bullet.mesh.position.copy(bullet.body.position as any);
    
    if (bullet.lifetime <= 0 || 
        Math.abs(bullet.body.position.x) > 25 || 
        Math.abs(bullet.body.position.y) > 25) {
      scene.remove(bullet.mesh);
      world.remove(bullet.body);
      bullets.value.splice(i, 1);
    }
  }
};

const updateEnemies = (_dt: number) => {
  if (performance.now() - lastEnemySpawn > enemySpawnInterval) {
    createEnemy();
    lastEnemySpawn = performance.now();
  }
  
  for (const enemy of enemies.value) {
    if (!enemy) continue;
    
    enemy.mesh.position.copy(enemy.body.position as any);
    const dir = new THREE.Vector3().subVectors(player.position, enemy.mesh.position);
    const len = dir.length();
    if (len > 0.0001) {
      dir.normalize();
    }
    const enemySpeed = 1.1;
    enemy.body.velocity.set(dir.x * enemySpeed, dir.y * enemySpeed, 0);
    enemy.mesh.quaternion.setFromUnitVectors(new THREE.Vector3(0, 1, 0), dir);
  }
};

const updateExplosions = (dt: number) => {
  for (let i = explosions.value.length - 1; i >= 0; i--) {
    const explosion = explosions.value[i];
    if (!explosion) continue;
    
    explosion.lifetime -= dt;
    (explosion.mesh.material as THREE.PointsMaterial).opacity = explosion.lifetime;
    
    if (explosion.lifetime <= 0) {
      scene.remove(explosion.mesh);
      explosions.value.splice(i, 1);
    }
  }
};

const checkCollisions = () => {
  // 子弹与敌人的碰撞检测
  for (let i = bullets.value.length - 1; i >= 0; i--) {
    const bullet = bullets.value[i];
    if (!bullet) continue;
    
    for (let j = enemies.value.length - 1; j >= 0; j--) {
      const enemy = enemies.value[j];
      if (!enemy) continue;
      
      const distance = bullet.body.position.distanceTo(enemy.body.position);
      if (distance < 1) {
        // 创建爆炸效果
        createExplosion(new THREE.Vector3().copy(enemy.mesh.position));
        
        // 增加分数
        score.value += 10;
        
        // 移除敌人
        scene.remove(enemy.mesh);
        world.remove(enemy.body);
        enemies.value.splice(j, 1);
        
        // 移除子弹
        scene.remove(bullet.mesh);
        world.remove(bullet.body);
        bullets.value.splice(i, 1);
        
        // 跳出内层循环，因为子弹已经被移除
        break;
      }
    }
  }
  
  // 玩家与敌人的碰撞检测
  for (let i = enemies.value.length - 1; i >= 0; i--) {
    const enemy = enemies.value[i];
    if (!enemy) continue;
    
    const distance = enemy.body.position.distanceTo(playerBody.position);
    if (distance < 1.2) {
      // 创建爆炸效果
      createExplosion(new THREE.Vector3().copy(enemy.mesh.position));
      
      // 减少玩家生命值
      health.value -= 20;
      
      // 移除敌人
      scene.remove(enemy.mesh);
      world.remove(enemy.body);
      enemies.value.splice(i, 1);
      
      // 检查游戏是否结束
      if (health.value <= 0) {
        health.value = 0;
        gameOver();
        return; // 游戏结束，直接返回
      }
    }
  }
};

const updatePlayer = (_dt: number) => {
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
  if (!gameCanvas.value) return;
  
  // Scene setup
  scene = new THREE.Scene();
  camera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 0.1, 1000);
  camera.position.z = 15;
  
  renderer = new THREE.WebGLRenderer({ 
    canvas: gameCanvas.value, 
    antialias: true 
  });
  
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
  playerBody = new CANNON.Body({ 
    mass: 1, 
    position: new CANNON.Vec3(0, 0, 0), 
    shape: playerShape, 
    linearDamping: 0.8 
  });
  
  world.addBody(playerBody);

  // Event Listeners
  const onResize = () => {
    camera.aspect = window.innerWidth / window.innerHeight;
    camera.updateProjectionMatrix();
    renderer.setSize(window.innerWidth, window.innerHeight);
  };
  
  const onKeyDown = (e: KeyboardEvent) => { 
    keys[e.code] = true; 
    if (e.code === 'Space') shoot(); 
  };
  
  const onKeyUp = (e: KeyboardEvent) => { 
    keys[e.code] = false; 
  };
  
  window.addEventListener('resize', onResize);
  window.addEventListener('keydown', onKeyDown);
  window.addEventListener('keyup', onKeyUp);
  
  // 清理函数
  onUnmounted(() => {
    window.removeEventListener('resize', onResize);
    window.removeEventListener('keydown', onKeyDown);
    window.removeEventListener('keyup', onKeyUp);
    cancelAnimationFrame(gameLoopId);
    cleanupGame();
  });
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
  pointer-events: none;
}

.hud > div {
  margin-bottom: 10px;
  padding: 5px 10px;
  background-color: rgba(0, 0, 0, 0.5);
  border-radius: 4px;
  display: inline-block;
  margin-right: 10px;
}

.start-screen, .game-over {
  top: 0; 
  left: 0; 
  height: 100%;
  display: flex;
  flex-direction: column;
  justify-content: center;
  align-items: center;
  background-color: rgba(0, 0, 0, 0.7);
}

h1 { 
  font-size: 4em; 
  text-shadow: 0 0 10px #00ffff; 
  margin-bottom: 20px;
}

p { 
  font-size: 1.5em; 
  margin: 10px 0;
}

button {
  margin-top: 30px; 
  padding: 12px 30px; 
  font-size: 1.2em;
  color: white; 
  background-color: #0066ff; 
  border: none;
  border-radius: 5px; 
  cursor: pointer; 
  transition: all 0.3s;
  outline: none;
}

button:hover { 
  background-color: #0088ff; 
  transform: scale(1.05); 
}

button:active {
  transform: scale(0.98);
}

/* 响应式设计 */
@media (max-width: 768px) {
  h1 {
    font-size: 2.5em;
  }
  
  p {
    font-size: 1.2em;
  }
  
  button {
    padding: 10px 20px;
    font-size: 1em;
  }
  
  .hud > div {
    font-size: 0.9em;
    padding: 3px 8px;
    margin-bottom: 5px;
  }
}
</style>