import React, { useEffect, useMemo, useRef, useState } from "react";
import { Canvas, useFrame, useThree } from "@react-three/fiber";
import { Sky, PerspectiveCamera, OrbitControls, Text } from "@react-three/drei";
import * as THREE from "three";

const ARENA_SIZE = 70;
const PLAYER_RADIUS = 1.1;
const DIFFICULTIES = {
  facil: { enemySpeed: 2.2, fireRate: 0.24, enemyCount: 6, enemyHealth: 2 },
  medio: { enemySpeed: 3.1, fireRate: 0.18, enemyCount: 9, enemyHealth: 3 },
  dificil: { enemySpeed: 4.2, fireRate: 0.13, enemyCount: 13, enemyHealth: 4 },
};

function clamp(v, min, max) {
  return Math.max(min, Math.min(max, v));
}

function randRange(min, max) {
  return min + Math.random() * (max - min);
}

function spawnEnemy(id, difficulty) {
  const side = Math.floor(Math.random() * 4);
  const pad = ARENA_SIZE * 0.45;
  let x = 0;
  let z = 0;
  if (side === 0) { x = randRange(-pad, pad); z = -pad; }
  if (side === 1) { x = randRange(-pad, pad); z = pad; }
  if (side === 2) { x = -pad; z = randRange(-pad, pad); }
  if (side === 3) { x = pad; z = randRange(-pad, pad); }
  return {
    id,
    x,
    y: 1,
    z,
    vx: 0,
    vz: 0,
    health: DIFFICULTIES[difficulty].enemyHealth,
    bob: Math.random() * Math.PI * 2,
    hue: Math.random() > 0.5 ? "#ff4d6d" : "#8b5cf6",
  };
}

function Ground() {
  const grid = useMemo(() => new THREE.GridHelper(ARENA_SIZE, 40, "#ffffff", "#444444"), []);
  return (
    <group>
      <primitive object={grid} position={[0, 0.01, 0]} />
      <mesh rotation={[-Math.PI / 2, 0, 0]} receiveShadow>
        <planeGeometry args={[ARENA_SIZE, ARENA_SIZE]} />
        <meshStandardMaterial color="#111318" />
      </mesh>
      <mesh position={[0, -0.2, 0]} receiveShadow>
        <boxGeometry args={[ARENA_SIZE + 6, 0.4, ARENA_SIZE + 6]} />
        <meshStandardMaterial color="#0a0b0f" />
      </mesh>
    </group>
  );
}

function ArenaWalls() {
  const h = 4;
  const s = ARENA_SIZE / 2;
  return (
    <group>
      {[
        [0, h / 2, s, ARENA_SIZE, h, 1],
        [0, h / 2, -s, ARENA_SIZE, h, 1],
        [s, h / 2, 0, 1, h, ARENA_SIZE],
        [-s, h / 2, 0, 1, h, ARENA_SIZE],
      ].map((w, i) => (
        <mesh key={i} position={[w[0], w[1], w[2]]} castShadow receiveShadow>
          <boxGeometry args={[w[3], w[4], w[5]]} />
          <meshStandardMaterial color="#1b1f28" />
        </mesh>
      ))}
    </group>
  );
}

function Obstacles() {
  const blocks = useMemo(
    () => [
      [-12, 1.8, -10, 5, 3.6, 5],
      [10, 1.3, -6, 4, 2.6, 4],
      [0, 2.2, 10, 7, 4.4, 4],
      [-4, 1.1, 0, 3.5, 2.2, 3.5],
      [15, 1.8, 12, 4.5, 3.6, 4.5],
      [-16, 1.6, 14, 4.5, 3.2, 6],
    ],
    []
  );
  return (
    <group>
      {blocks.map((b, i) => (
        <mesh key={i} position={[b[0], b[1], b[2]]} castShadow receiveShadow>
          <boxGeometry args={[b[3], b[4], b[5]]} />
          <meshStandardMaterial color="#202633" metalness={0.1} roughness={0.8} />
        </mesh>
      ))}
    </group>
  );
}

function Player({ playerRef, weaponKick }) {
  return (
    <group ref={playerRef} position={[0, 1.15, 0]}>
      <mesh castShadow position={[0, 0.8, 0]}>
        <capsuleGeometry args={[0.8, 1.5, 6, 12]} />
        <meshStandardMaterial color="#f5f7fb" />
      </mesh>
      <mesh castShadow position={[0, 2.25, 0]}>
        <sphereGeometry args={[0.52, 20, 20]} />
        <meshStandardMaterial color="#f5f7fb" />
      </mesh>
      <group position={[0.55, 1.5, 0.65 - weaponKick * 0.18]} rotation={[0, 0, -0.15]}>
        <mesh castShadow>
          <boxGeometry args={[0.28, 0.28, 1.25]} />
          <meshStandardMaterial color="#111111" />
        </mesh>
        <mesh position={[0, 0, 0.72]} castShadow>
          <boxGeometry args={[0.18, 0.18, 0.35]} />
          <meshStandardMaterial color="#d1d5db" />
        </mesh>
      </group>
    </group>
  );
}

function Drone({ enemy }) {
  return (
    <group position={[enemy.x, enemy.y + Math.sin(enemy.bob) * 0.15, enemy.z]}>
      <mesh castShadow>
        <sphereGeometry args={[0.85, 18, 18]} />
        <meshStandardMaterial color={enemy.hue} emissive={enemy.hue} emissiveIntensity={0.35} />
      </mesh>
      <mesh position={[0, -0.75, 0]} castShadow>
        <cylinderGeometry args={[0.16, 0.16, 1.1, 10]} />
        <meshStandardMaterial color="#dfe5ef" />
      </mesh>
      <mesh position={[0.95, -0.2, 0]} rotation={[0, 0, Math.PI / 2]} castShadow>
        <cylinderGeometry args={[0.09, 0.09, 1.2, 10]} />
        <meshStandardMaterial color="#dfe5ef" />
      </mesh>
      <mesh position={[-0.95, -0.2, 0]} rotation={[0, 0, Math.PI / 2]} castShadow>
        <cylinderGeometry args={[0.09, 0.09, 1.2, 10]} />
        <meshStandardMaterial color="#dfe5ef" />
      </mesh>
    </group>
  );
}

function Projectile({ shot }) {
  return (
    <mesh position={[shot.x, shot.y, shot.z]} castShadow>
      <sphereGeometry args={[0.14, 10, 10]} />
      <meshStandardMaterial color="#facc15" emissive="#f59e0b" emissiveIntensity={1.5} />
    </mesh>
  );
}

function EnemyProjectile({ shot }) {
  return (
    <mesh position={[shot.x, shot.y, shot.z]} castShadow>
      <sphereGeometry args={[0.16, 10, 10]} />
      <meshStandardMaterial color="#fb7185" emissive="#e11d48" emissiveIntensity={1.4} />
    </mesh>
  );
}

function CameraRig({ cameraMode, playerState, aim }) {
  const { camera } = useThree();
  useFrame((_, dt) => {
    const playerPos = new THREE.Vector3(playerState.x, 1.6, playerState.z);
    const forward = new THREE.Vector3(Math.sin(aim.yaw), 0, Math.cos(aim.yaw));
    if (cameraMode === "primeira") {
      const eye = playerPos.clone().add(new THREE.Vector3(0, 1.7, 0));
      camera.position.lerp(eye, 1 - Math.pow(0.0001, dt));
      const target = eye.clone().add(forward.multiplyScalar(10)).add(new THREE.Vector3(0, aim.pitch * 4, 0));
      camera.lookAt(target);
    } else if (cameraMode === "terceira") {
      const behind = playerPos.clone().add(new THREE.Vector3(-forward.x * 6, 3.8, -forward.z * 6));
      camera.position.lerp(behind, 1 - Math.pow(0.0001, dt));
      camera.lookAt(playerPos.clone().add(new THREE.Vector3(0, 1.2, 0)));
    } else {
      const top = playerPos.clone().add(new THREE.Vector3(0, 21, 0.001));
      camera.position.lerp(top, 1 - Math.pow(0.0001, dt));
      camera.lookAt(playerPos);
    }
  });
  return null;
}

function HUD({ health, ammo, reserveAmmo, difficulty, cameraMode, score, combo, onRestart, onDifficulty, onCamera, paused, setPaused }) {
  return (
    <div className="absolute inset-0 pointer-events-none">
      <div className="pointer-events-auto absolute left-4 top-4 rounded-3xl border border-white/10 bg-black/45 backdrop-blur-xl p-4 text-white shadow-2xl w-[min(360px,calc(100vw-32px))]">
        <div className="text-xs uppercase tracking-[0.3em] text-white/45">Sandbox Blaster 3D</div>
        <div className="mt-2 text-3xl font-black tracking-[-0.05em]">Treino de Arena</div>
        <div className="mt-3 grid grid-cols-2 gap-3 text-sm">
          <div className="rounded-2xl border border-white/10 bg-white/5 p-3"><div className="text-white/50">Vida</div><div className="text-xl font-black">{health}</div></div>
          <div className="rounded-2xl border border-white/10 bg-white/5 p-3"><div className="text-white/50">Score</div><div className="text-xl font-black">{score}</div></div>
          <div className="rounded-2xl border border-white/10 bg-white/5 p-3"><div className="text-white/50">Ammo</div><div className="text-xl font-black">{ammo} / {reserveAmmo}</div></div>
          <div className="rounded-2xl border border-white/10 bg-white/5 p-3"><div className="text-white/50">Combo</div><div className="text-xl font-black">x{combo}</div></div>
        </div>
        <div className="mt-4 flex flex-wrap gap-2">
          {Object.keys(DIFFICULTIES).map((key) => (
            <button key={key} onClick={() => onDifficulty(key)} className={`rounded-2xl px-4 py-2 text-sm font-bold border ${difficulty === key ? "bg-white text-black border-white" : "bg-white/5 text-white border-white/10"}`}>{key}</button>
          ))}
        </div>
        <div className="mt-3 flex flex-wrap gap-2">
          {[
            ["primeira", "1ª pessoa"],
            ["terceira", "3ª pessoa"],
            ["topo", "topo"],
          ].map(([key, label]) => (
            <button key={key} onClick={() => onCamera(key)} className={`rounded-2xl px-4 py-2 text-sm font-bold border ${cameraMode === key ? "bg-fuchsia-400 text-black border-fuchsia-300" : "bg-white/5 text-white border-white/10"}`}>{label}</button>
          ))}
        </div>
        <div className="mt-3 flex flex-wrap gap-2">
          <button onClick={onRestart} className="rounded-2xl bg-white px-4 py-2 text-sm font-black text-black">reiniciar</button>
          <button onClick={() => setPaused((v) => !v)} className="rounded-2xl bg-white/8 px-4 py-2 text-sm font-black text-white border border-white/10">{paused ? "continuar" : "pausar"}</button>
        </div>
      </div>

      <div className="absolute right-4 top-4 pointer-events-auto rounded-3xl border border-white/10 bg-black/45 backdrop-blur-xl p-4 text-white shadow-2xl w-[min(340px,calc(100vw-32px))]">
        <div className="text-sm font-bold text-white/60">Controles</div>
        <div className="mt-2 text-sm text-white/80 leading-6">
          WASD mover · Mouse arrasta a mira · Clique esquerdo atira · R recarrega · C troca câmera · P pausa
        </div>
        <div className="mt-3 text-sm text-white/60">Inimigos são drones de treino, não pessoas.</div>
      </div>

      <div className="absolute left-1/2 top-1/2 -translate-x-1/2 -translate-y-1/2 text-white text-3xl font-black select-none">+</div>
    </div>
  );
}

function GameWorld({ difficulty, cameraMode, setCameraMode, onDifficulty }) {
  const playerRef = useRef();
  const keys = useRef({});
  const drag = useRef(false);
  const aim = useRef({ yaw: 0, pitch: 0 });
  const fireCooldown = useRef(0);
  const enemyFireCooldown = useRef(0);
  const [weaponKick, setWeaponKick] = useState(0);
  const [health, setHealth] = useState(100);
  const [ammo, setAmmo] = useState(24);
  const [reserveAmmo, setReserveAmmo] = useState(120);
  const [score, setScore] = useState(0);
  const [combo, setCombo] = useState(1);
  const [paused, setPaused] = useState(false);
  const [playerState, setPlayerState] = useState({ x: 0, z: 0 });
  const [shots, setShots] = useState([]);
  const [enemyShots, setEnemyShots] = useState([]);
  const [enemies, setEnemies] = useState(() => Array.from({ length: DIFFICULTIES[difficulty].enemyCount }, (_, i) => spawnEnemy(i + 1, difficulty)));

  const restart = () => {
    setHealth(100);
    setAmmo(24);
    setReserveAmmo(120);
    setScore(0);
    setCombo(1);
    setPaused(false);
    setPlayerState({ x: 0, z: 0 });
    aim.current = { yaw: 0, pitch: 0 };
    fireCooldown.current = 0;
    enemyFireCooldown.current = 0;
    setShots([]);
    setEnemyShots([]);
    setEnemies(Array.from({ length: DIFFICULTIES[difficulty].enemyCount }, (_, i) => spawnEnemy(i + 1, difficulty)));
  };

  useEffect(() => { restart(); }, [difficulty]);

  useEffect(() => {
    const onKeyDown = (e) => {
      keys.current[e.key.toLowerCase()] = true;
      if (e.key.toLowerCase() === "r") {
        setAmmo((curr) => {
          if (curr === 24 || reserveAmmo <= 0) return curr;
          const need = 24 - curr;
          const taken = Math.min(need, reserveAmmo);
          setReserveAmmo((r) => r - taken);
          return curr + taken;
        });
      }
      if (e.key.toLowerCase() === "c") {
        setCameraMode((prev) => prev === "primeira" ? "terceira" : prev === "terceira" ? "topo" : "primeira");
      }
      if (e.key.toLowerCase() === "p") setPaused((v) => !v);
    };
    const onKeyUp = (e) => { keys.current[e.key.toLowerCase()] = false; };
    const onMouseDown = (e) => {
      if (e.button === 0) drag.current = true;
      shoot();
    };
    const onMouseUp = () => { drag.current = false; };
    const onMouseMove = (e) => {
      if (!drag.current && cameraMode === "topo") return;
      aim.current.yaw -= e.movementX * 0.005;
      aim.current.pitch = clamp(aim.current.pitch - e.movementY * 0.003, -0.8, 0.8);
    };
    window.addEventListener("keydown", onKeyDown);
    window.addEventListener("keyup", onKeyUp);
    window.addEventListener("mousedown", onMouseDown);
    window.addEventListener("mouseup", onMouseUp);
    window.addEventListener("mousemove", onMouseMove);
    return () => {
      window.removeEventListener("keydown", onKeyDown);
      window.removeEventListener("keyup", onKeyUp);
      window.removeEventListener("mousedown", onMouseDown);
      window.removeEventListener("mouseup", onMouseUp);
      window.removeEventListener("mousemove", onMouseMove);
    };
  }, [cameraMode, reserveAmmo]);

  const shoot = () => {
    if (paused || health <= 0) return;
    if (fireCooldown.current > 0) return;
    setAmmo((curr) => {
      if (curr <= 0) return curr;
      fireCooldown.current = DIFFICULTIES[difficulty].fireRate;
      setWeaponKick(1);
      const dir = new THREE.Vector3(Math.sin(aim.current.yaw), aim.current.pitch * 0.4, Math.cos(aim.current.yaw)).normalize();
      setShots((s) => [...s, {
        id: performance.now() + Math.random(),
        x: playerState.x + dir.x * 1.6,
        y: 2.1,
        z: playerState.z + dir.z * 1.6,
        dx: dir.x * 1.2,
        dy: dir.y * 1.2,
        dz: dir.z * 1.2,
        life: 1.4,
      }]);
      return curr - 1;
    });
  };

  useFrame((_, dt) => {
    if (paused || health <= 0) return;
    fireCooldown.current = Math.max(0, fireCooldown.current - dt);
    enemyFireCooldown.current = Math.max(0, enemyFireCooldown.current - dt);
    setWeaponKick((v) => Math.max(0, v - dt * 8));

    setPlayerState((prev) => {
      let x = prev.x;
      let z = prev.z;
      const speed = 9 * dt;
      const forward = { x: Math.sin(aim.current.yaw), z: Math.cos(aim.current.yaw) };
      const right = { x: Math.cos(aim.current.yaw), z: -Math.sin(aim.current.yaw) };
      if (keys.current["w"]) { x += forward.x * speed; z += forward.z * speed; }
      if (keys.current["s"]) { x -= forward.x * speed; z -= forward.z * speed; }
      if (keys.current["a"]) { x -= right.x * speed; z -= right.z * speed; }
      if (keys.current["d"]) { x += right.x * speed; z += right.z * speed; }
      x = clamp(x, -ARENA_SIZE / 2 + 2, ARENA_SIZE / 2 - 2);
      z = clamp(z, -ARENA_SIZE / 2 + 2, ARENA_SIZE / 2 - 2);
      return { x, z };
    });

    setEnemies((prev) => {
      let next = prev.map((e) => {
        const dx = playerState.x - e.x;
        const dz = playerState.z - e.z;
        const len = Math.max(0.001, Math.hypot(dx, dz));
        const speed = DIFFICULTIES[difficulty].enemySpeed * dt;
        const tooClose = len < 5.5;
        const nx = e.x + (tooClose ? -dx / len : dx / len) * speed;
        const nz = e.z + (tooClose ? -dz / len : dz / len) * speed;
        return { ...e, x: clamp(nx, -ARENA_SIZE / 2 + 2, ARENA_SIZE / 2 - 2), z: clamp(nz, -ARENA_SIZE / 2 + 2, ARENA_SIZE / 2 - 2), bob: e.bob + dt * 5 };
      });

      if (enemyFireCooldown.current <= 0 && next.length) {
        enemyFireCooldown.current = 0.9;
        const shooter = next[Math.floor(Math.random() * next.length)];
        const dx = playerState.x - shooter.x;
        const dz = playerState.z - shooter.z;
        const len = Math.max(0.001, Math.hypot(dx, dz));
        setEnemyShots((s) => [...s, {
          id: performance.now() + Math.random(),
          x: shooter.x,
          y: 1.1,
          z: shooter.z,
          dx: (dx / len) * 0.42,
          dz: (dz / len) * 0.42,
          life: 3,
        }]);
      }

      return next;
    });

    setShots((prev) => prev
      .map((s) => ({ ...s, x: s.x + s.dx, y: s.y + s.dy, z: s.z + s.dz, life: s.life - dt }))
      .filter((s) => s.life > 0 && Math.abs(s.x) < ARENA_SIZE && Math.abs(s.z) < ARENA_SIZE)
    );

    setEnemyShots((prev) => prev
      .map((s) => ({ ...s, x: s.x + s.dx, z: s.z + s.dz, life: s.life - dt }))
      .filter((s) => s.life > 0)
    );
  });

  useEffect(() => {
    if (health <= 0) return;

    let hitScore = 0;
    setShots((currShots) => {
      const remainingShots = [];
      const updatedEnemies = [...enemies];
      for (const shot of currShots) {
        let hit = false;
        for (let i = 0; i < updatedEnemies.length; i++) {
          const e = updatedEnemies[i];
          const d = Math.hypot(shot.x - e.x, shot.z - e.z);
          if (d < 1.1) {
            hit = true;
            e.health -= 1;
            if (e.health <= 0) {
              updatedEnemies[i] = spawnEnemy(e.id, difficulty);
              hitScore += 100 * combo;
              setCombo((c) => Math.min(12, c + 1));
            } else {
              hitScore += 25;
            }
            break;
          }
        }
        if (!hit) remainingShots.push(shot);
      }
      if (hitScore) {
        setScore((s) => s + hitScore);
        setEnemies(updatedEnemies);
      }
      return remainingShots;
    });
  }, [shots]);

  useEffect(() => {
    if (health <= 0) return;
    setEnemyShots((curr) => {
      const remain = [];
      let hits = 0;
      for (const shot of curr) {
        const d = Math.hypot(shot.x - playerState.x, shot.z - playerState.z);
        if (d < 1.4) hits += 1;
        else remain.push(shot);
      }
      if (hits) {
        setHealth((h) => Math.max(0, h - hits * 8));
        setCombo(1);
      }
      return remain;
    });
  }, [enemyShots, playerState.x, playerState.z, health]);

  useEffect(() => {
    if (playerRef.current) {
      playerRef.current.position.x = playerState.x;
      playerRef.current.position.z = playerState.z;
      playerRef.current.rotation.y = aim.current.yaw;
    }
  }, [playerState, aim.current.yaw]);

  return (
    <div className="relative h-screen w-full bg-[#06070b]">
      <HUD
        health={health}
        ammo={ammo}
        reserveAmmo={reserveAmmo}
        difficulty={difficulty}
        cameraMode={cameraMode}
        score={score}
        combo={combo}
        onRestart={restart}
        onDifficulty={onDifficulty}
        onCamera={setCameraMode}
        paused={paused}
        setPaused={setPaused}
      />

      {health <= 0 && (
        <div className="absolute inset-0 z-20 grid place-items-center bg-black/60">
          <div className="rounded-[32px] border border-white/10 bg-black/60 p-8 text-center text-white backdrop-blur-xl">
            <div className="text-5xl font-black tracking-[-0.06em]">Fim da rodada</div>
            <div className="mt-3 text-white/70">Score final: {score}</div>
            <button onClick={restart} className="mt-5 rounded-2xl bg-white px-5 py-3 font-black text-black">jogar de novo</button>
          </div>
        </div>
      )}

      <Canvas shadows gl={{ antialias: true }}>
        <PerspectiveCamera makeDefault fov={cameraMode === "primeira" ? 74 : cameraMode === "terceira" ? 65 : 50} position={[0, 8, 12]} />
        <CameraRig cameraMode={cameraMode} playerState={playerState} aim={aim.current} />
        <Sky sunPosition={[20, 8, 5]} turbidity={6} rayleigh={0.25} mieCoefficient={0.015} mieDirectionalG={0.8} />
        <ambientLight intensity={0.7} />
        <directionalLight position={[8, 16, 10]} intensity={1.3} castShadow shadow-mapSize-width={2048} shadow-mapSize-height={2048} />
        <fog attach="fog" args={["#07090f", 22, 90]} />

        <Ground />
        <ArenaWalls />
        <Obstacles />
        <Player playerRef={playerRef} weaponKick={weaponKick} />

        {enemies.map((enemy) => <Drone key={enemy.id} enemy={enemy} />)}
        {shots.map((shot) => <Projectile key={shot.id} shot={shot} />)}
        {enemyShots.map((shot) => <EnemyProjectile key={shot.id} shot={shot} />)}

        {cameraMode === "topo" && <OrbitControls enablePan={false} enableZoom={false} enableRotate={false} />}

        <Text position={[0, 10, -22]} fontSize={2.2} color="#ffffff" anchorX="center" anchorY="middle">
          SANDBOX BLASTER 3D
        </Text>
      </Canvas>
    </div>
  );
}

export default function SandboxBlaster3D() {
  const [difficulty, setDifficulty] = useState("facil");
  const [cameraMode, setCameraMode] = useState("terceira");

  return (
    <div className="h-screen w-full overflow-hidden bg-[#06070b] text-white">
      <GameWorld
        difficulty={difficulty}
        cameraMode={cameraMode}
        setCameraMode={setCameraMode}
        onDifficulty={setDifficulty}
      />
    </div>
  );
}
