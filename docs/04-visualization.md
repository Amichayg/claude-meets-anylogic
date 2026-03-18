# Building a Three.js Visualization

AnyLogic has built-in 3D, but it's locked inside the AnyLogic runtime. A standalone Three.js app lets you share your simulation's story with anyone who has a browser — stakeholders, teammates, or a GitHub audience.

This guide explains how we translated the grain terminal simulation into an interactive 3D web visualization.

## The Approach

We're not recreating the simulation engine. We're building a **visual narrative** that shows the same flows, timing, and logic as the AnyLogic model, rendered in real-time 3D. Think of it as an animated explainer, not a port.

The mapping:

| AnyLogic Concept | Three.js Equivalent |
|-----------------|---------------------|
| Agent (Truck, Ship) | `THREE.Group` with child meshes |
| Flowchart state (queuing, unloading) | `userData.state` driving position updates |
| Fluid flow (grain transfer) | Particle system (`THREE.Points`) |
| Silo fill level | Cylinder mesh with dynamic `scale.y` |
| Parameters (capacity, rates) | JavaScript constants |
| KPI counters | DOM overlay (HUD) |
| Simulation time | Scaled `requestAnimationFrame` delta |

## File Structure

Everything lives in a single `index.html` — no build step, no dependencies to install. Three.js loads from a CDN via import map:

```html
<script type="importmap">
{
  "imports": {
    "three": "https://cdn.jsdelivr.net/npm/three@0.164.1/build/three.module.js",
    "three/addons/": "https://cdn.jsdelivr.net/npm/three@0.164.1/examples/jsm/"
  }
}
</script>
```

## Scene Setup

### Camera and Controls

We use a perspective camera with `OrbitControls` for mouse interaction:

```javascript
const camera = new THREE.PerspectiveCamera(45, aspect, 0.5, 500);
camera.position.set(-60, 55, 80); // elevated angle looking at the terminal

const controls = new OrbitControls(camera, canvas);
controls.target.set(0, 5, 0);     // look at the center of the action
controls.maxPolarAngle = Math.PI / 2.15; // prevent going below ground
```

### Lighting

Three lights create depth without harshness:

```javascript
// Soft ambient fill
new THREE.AmbientLight(0x334466, 0.6);

// Main directional (warm, casts shadows)
const dirLight = new THREE.DirectionalLight(0xffeedd, 1.2);
dirLight.position.set(40, 60, 30);
dirLight.castShadow = true;

// Cool rim light for edge definition
new THREE.DirectionalLight(0x4466aa, 0.4);
```

### Materials

We define a material palette upfront. The key technique is **emissive materials** for grain — they glow slightly, making the colored grain visible inside dark silo shells:

```javascript
const matGlow = (hex) => new THREE.MeshStandardMaterial({
  color: hex,
  emissive: hex,
  emissiveIntensity: 0.3,
  roughness: 0.5
});
```

## Building the Layout

The layout mirrors the simulation's physical flow, left to right:

```
Road → Parking → Auto Silos → Conveyors → Main Silos → Conveyors → Piers → Water
  ↑                  ↑                         ↑                        ↑
x=-55             x=-30                    x=-4 to 15              x=42-56
```

### Silos (Cylinders)

Auto silos and main silos are `CylinderGeometry` with a nested fill indicator:

```javascript
// Outer shell
const silo = cylinder(2.5, 2.5, 12, matSiloShell);

// Inner fill (starts at 0 height, scales up as grain arrives)
const fill = cylinder(2.3, 2.3, 0.01, matGlow(GRAIN_COLORS[0]));
```

During the update loop, the fill scales with the amount:

```javascript
const fillH = (silo.amount / silo.capacity) * 11;
fill.scale.y = Math.max(0.001, fillH);
fill.position.y = 0.2 + fillH / 2; // anchor to bottom
```

### Ships (Composite Groups)

A ship is a `THREE.Group` with hull, deck, bridge, and 5 bilge holds:

```javascript
function createShip(z) {
  const ship = new THREE.Group();
  ship.add(hull);    // dark box
  ship.add(deck);    // wooden top
  ship.add(bridge);  // tall block at stern
  // 5 bilges with individual fill indicators
  for (let i = 0; i < 5; i++) {
    ship.add(bilgeShell);
    ship.add(bilgeFill); // scales up as grain loads
  }
  return ship;
}
```

Ships bob on the water using a sine wave:

```javascript
ship.position.y = -0.5 + Math.sin(simTime * 0.5 + i) * 0.15;
```

### Trucks (Composite Groups)

Trucks have a body, cab, cargo (colored by grain type), and 4 wheels:

```javascript
const cargo = box(2.2, 1.2, 1.8, matGlow(GRAIN_COLORS[grainType]));
```

The cargo color immediately tells you what grain type the truck carries.

## The Simulation Loop

The core pattern is a state machine per entity type, driven by `requestAnimationFrame`:

```javascript
function updateSim(dt) {
  const simDt = dt * speedMult * 30; // 30x real-time base speed
  simTime += simDt;

  spawnEntities();     // create trucks, trains, ships on schedule
  updateTrucks(dt);    // move through states: arriving → queuing → unloading → leaving
  updateAutoSilos(dt); // transfer to main silos when threshold hit
  updateTrains(dt);    // arriving → unloading → leaving
  updateShips(dt);     // arriving → loading → departing
  updateParticles(dt); // grain flow effects
  updateHUD();         // DOM overlay
}
```

### Entity State Machines

Each truck has a `state` field that drives its behavior:

```
arriving  → move toward parking slot
queuing   → wait, then find an auto silo
unloading → transfer grain, emit particles
leaving   → drive off-screen, then remove from scene
```

State transitions happen when conditions are met (arrival at target, grain depleted, timeout exceeded).

## Grain Flow Particles

The particle system makes grain transfers visible. We use a fixed-size pool of 600 particles with additive blending:

```javascript
const particleMat = new THREE.PointsMaterial({
  size: 0.4,
  vertexColors: true,
  transparent: true,
  blending: THREE.AdditiveBlending,
  depthWrite: false
});
```

When grain transfers happen, we emit particles at the source with a velocity toward the destination:

```javascript
function emitParticle(pos, vel, color) {
  particlePositions[i * 3]     = pos.x;
  particlePositions[i * 3 + 1] = pos.y;
  particlePositions[i * 3 + 2] = pos.z;
  particleVelocities[i].copy(vel);
  particleLife[i] = 1.0;
}
```

Particles have gravity (`vel.y -= 3 * dt`) and fade over time, creating an arc from source to destination.

## The HUD

The HUD is pure HTML/CSS overlaid on the canvas. Key design choices:

- **Glassmorphism panels** — `backdrop-filter: blur(12px)` with semi-transparent backgrounds
- **Monospace font** — consistent with a technical/simulation aesthetic
- **Accent color** — `#8ab4f8` (soft blue) for all data values
- **Pointer events** — `pointer-events: none` on the HUD container, `auto` on interactive children

The appointment panel updates from the simulation state:

```javascript
appointmentSlots.slice(-6).map(s => {
  const full = s.trucks >= MAX_PER_SLOT;
  return `<span style="color:${full ? '#f44336' : '#4caf50'}">
    ${s.trucks}/${MAX_PER_SLOT}
  </span>`;
});
```

## Performance Tips

- **Reuse geometries** — create one `BoxGeometry` and share it across multiple meshes
- **Particle pooling** — fixed-size array, recycle oldest particles instead of creating/destroying
- **Limit shadow casters** — only the main directional light casts shadows
- **Cap pixel ratio** — `Math.min(devicePixelRatio, 2)` prevents 3x rendering on high-DPI screens
- **Fog** — `FogExp2` hides the horizon and reduces draw calls for distant objects

## Extending the Visualization

Ideas for improvement:

- **Labels** — use `CSS2DRenderer` to add floating text labels over silos showing fill % and grain type
- **Camera presets** — add buttons for "Truck View", "Ship View", "Overview" with animated camera transitions
- **Charts** — embed a small Chart.js canvas showing throughput over time
- **Sound** — add ambient port sounds, truck engines, conveyor hums using the Web Audio API
- **Data export** — log simulation events to a JSON array and offer a download button for analysis
- **WebSocket** — connect to a running AnyLogic model via its REST API for live data

## Summary

The Three.js visualization translates simulation concepts into something anyone can understand at a glance. The key is the mapping: entities become mesh groups, states drive movement, fluid flow becomes particles, and fill levels become scaled geometry. No simulation engine needed — just the narrative.
