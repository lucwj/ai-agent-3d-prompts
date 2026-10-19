# Build "AI Agent World" from scratch

You are building **AI Agent World**: a living 3D office in the browser where six AI agents (Orchestrator, Product Manager, Researcher, Software Engineer, QA Engineer, Code Reviewer) collaborate to ship software tasks. The world is a *visualisation of a multi-agent system's state*: every walk, gesture, screen glow, speech bubble, sound and communication arc is driven by backend state and a fixed vocabulary of structured events. Nothing on screen is decorative animation that lies about what the system is doing.

The default backend is a **deterministic local simulation**: no LLM is called, no code is written, outputs are canned templates. The UI must say so plainly and everywhere it matters. The architecture must make it trivial to swap the simulation for a real orchestrator over WebSocket without touching the 3D layer or the UI.

Work through the phases in order. After each phase: `npm run typecheck`, `npm run lint`, `npm test` must pass, and you must open the app in a browser and verify what the phase promises. Do not report a phase as done until you have seen it work.

---

## 0. Non-negotiables

1. **Rendering never mutates simulation state.** The 3D layer and the HTML UI read state; all commands go through the backend contract.
2. **Determinism.** Same seed + same inputs ⇒ same run, tick for tick. The demo task must be repeatable to the second.
3. **Honesty.** Label simulation mode in the top bar, the task dialog, settings, and the demo callout. Never imply inference is happening when it is not.
4. **Performance.** 30 Hz simulation tick; 3D reads a mutable per-frame buffer (no React re-render per tick); HTML UI subscribes to a throttled mirror. Shared geometry/material pools, `InstancedMesh` for repeated things, one shadow-casting light, no post-processing. Must hold 60 fps on a 2020 laptop and degrade to a "low" quality path automatically.
5. **Everything typed.** TypeScript strict. No `any` outside third-party boundary casts.
6. **No binary assets.** Textures are generated on a `<canvas>`, sounds are synthesised with Web Audio, characters are procedural primitives. The repo must run with `npm install && npm run dev` and nothing else.

---

## 1. Stack

- **Next.js 16** (App Router, Turbopack), **React 19.3**, **TypeScript 5.9**.
- **three ^0.186**, **@react-three/fiber ^9.8**, **@react-three/drei ^10.7** (use sparingly: `RoundedBox`, `OrbitControls`, `PerformanceMonitor`, `Lightformer`/`Environment`; **do not use drei `<Html>`** — see pitfalls).
- **Zustand 5** for UI state.
- **Tailwind CSS 4** via `@tailwindcss/postcss`, theme tokens declared with `@theme` in `globals.css`.
- **lucide-react** for icons.
- Dev: eslint 9 + eslint-config-next, no test framework — tests are a plain Node script run with `node --experimental-strip-types`.

`package.json` scripts:

```json
{
  "dev": "next dev",
  "build": "next build",
  "start": "next start",
  "typecheck": "tsc --noEmit",
  "lint": "eslint src",
  "test": "node --experimental-strip-types --no-warnings scripts/simulate.ts"
}
```

`next.config.ts`: `reactStrictMode: true`, `transpilePackages: ["three"]`.

Because the simulation is executed by Node directly in tests, **every file under `src/simulation`, `src/tasks`, `src/events`, `src/agents/*.ts` (non-React) and `src/integrations/AgentBackend.ts` must import siblings with explicit `.ts` extensions** (`import x from "./y.ts"`) and must not import anything from React, three, Next or `@/` aliases. Set `allowImportingTsExtensions: true` in `tsconfig.json`.

---

## 2. Folder layout and responsibilities

```
src/
  app/            layout.tsx, page.tsx (renders <AppShell/>), globals.css
  simulation/     SimulationEngine.ts (pure TS, the whole mock brain)
                  WorkflowStateMachine.ts, TaskScheduler.ts, EventBus.ts (+ seeded RNG)
                  simulationConfig.ts (ALL timings/limits live here), worldLayout.ts (positions + routing)
                  workContent.ts (canned subtasks/findings/tests keyed by regex on task text)
  agents/         agentTypes.ts, AgentRegistry.ts, AgentController.ts (action queue, walking),
                  AgentAnimator.ts (procedural pose blending), AgentCharacter.tsx (procedural robot meshes)
  tasks/          taskTypes.ts, taskActions.ts (DEMO_TASK + selectors), taskBacklog.ts (18 tasks + SPRINT_TASKS)
  events/         eventTypes.ts (vocabulary), eventStore.ts (pure selectors over the log)
  integrations/   AgentBackend.ts (contract + WorldSnapshot), MockAgentBackend.ts, WebSocketAgentBackend.ts,
                  backendClient.ts (selection by env + `commands` object the UI calls)
  state/          useWorldStore.ts (Zustand mirror + `liveWorld` mutable buffer)
  world/          WorldCanvas.tsx, OfficeEnvironment.tsx, Workstations.tsx, AgentsLayer.tsx (paths, comm arcs),
                  Lighting.tsx, CameraController.tsx, cameraPresets.ts, InteractionManager.tsx,
                  proceduralTextures.ts, sharedResources.ts, stationOwners.ts,
                  labelAnchors.ts + AgentLabelsOverlay.tsx (DOM speech bubbles)
  audio/          soundEngine.ts (procedural Web Audio cues), useSoundEffects.ts
  components/     layout/ (AppShell, NavRail, TopBar, LoadingScreen), panels/ (SidePanel, Panels, TaskCard,
                  TaskDetails, AgentDetails, TaskDialog), controls/ (SimulationControls, CameraBar),
                  activity/ActivityFeed.tsx, ui.tsx (shared atoms)
scripts/
  simulate.ts     headless engine tests (npm test)
  e2e.py          optional Playwright smoke test (screenshots)
```

Data flow: `AgentBackend` → events + snapshots → `useWorldStore.applySnapshot` writes `liveWorld` every snapshot (30 Hz) and pushes a React update at most every 160 ms unless something *significant* changed (status, speed, or an agent/task signature). The 3D layer reads `liveWorld` inside `useFrame`. The HTML UI reads the Zustand mirror.

---

## 3. Domain model

### 3.1 Agents

```ts
type AgentId = "orion" | "iris" | "piper" | "kai" | "quinn" | "rex";
type AgentRole = "orchestrator" | "researcher" | "product_manager" | "engineer" | "qa" | "reviewer";
type AgentStatus = "idle" | "walking" | "working" | "thinking" | "communicating" | "success" | "error";
```

| id | name | role | roleLabel | tagline | color | home |
|---|---|---|---|---|---|---|
| orion | Orion | orchestrator | Orchestrator | Coordinates work and delegates tasks | `#E8B04A` | hub |
| iris | Iris | researcher | Researcher | Researches technical topics and gathers information | `#5CC8D7` | research |
| piper | Piper | product_manager | Product Manager | Converts requirements into actionable tasks | `#A78BFA` | product |
| kai | Kai | engineer | Software Engineer | Implements software features | `#6C9EFF` | dev |
| quinn | Quinn | qa | QA Engineer | Tests completed work | `#5FD39A` | testing |
| rex | Rex | reviewer | Code Reviewer | Reviews implementation quality | `#F08A7E` | review |

Each agent also has a `seat` index (0..5) used at shared ring locations so agents never overlap.

`AgentState` (serialisable; what a real backend would also send): `id, status, activity (string), currentTaskId, progress 0..1, location | null, destination | null, position {x,z}, heading (rad), path: Vec2[] (remaining walk path), facingAgentId | null, paused: boolean`.

### 3.2 Tasks

```ts
type WorkflowPhase = "queued" | "intake" | "planning" | "research" | "implementation" | "testing" | "review" | "done" | "failed";
type TaskStatus = "queued" | "assigned" | "in_progress" | "testing" | "review" | "blocked" | "completed" | "failed";
type TaskPriority = "low" | "medium" | "high" | "critical";   // PRIORITY_ORDER critical=0 … low=3
```

`Task`: `id ("T-001"…), title, description, priority, status, phase, assignedAgentId, progress, createdAt, updatedAt, completedAt, iteration (implementation attempt counter, 1-based once implementation starts), subtasks[{id,title,done}], findings: string[], history: [{from,to,signal,at}], isDemo, preferredAgentId, blockedReason | null`.

`CreateTaskInput`: `title, description?, priority?, preferredAgentId?, isDemo?, script?: { qaFailsOn?: number[]; reviewChangesOn?: number[] }` — `script` forces outcomes per iteration so demos are repeatable.

**Status is a projection of phase** (`PHASE_STATUS`): queued→queued, intake→assigned, planning/research/implementation→in_progress, testing→testing, review→review, done→completed, failed→failed. While the responsible agent is paused the status is overridden to `blocked` with a `blockedReason`.

### 3.3 Workflow state machine (pure)

```
queued ──picked_up──▶ intake ──delegated──▶ planning ──plan_ready──▶ research
research ──findings_published──▶ implementation ──submitted──▶ testing
testing ──tests_passed──▶ review ──approved──▶ done
testing ──tests_failed──▶ implementation        (rework)
review  ──changes_requested──▶ implementation   (rework)
any non-terminal ──abort──▶ failed
```

`applySignal(task, signal, at)` returns a new task: sets phase/status, bumps `iteration` when entering implementation, resets progress to the phase start on rework (otherwise `max(progress, phaseStart)`), appends to `history`, sets `completedAt` on terminal. Export `canTransition`, `nextPhase`, `isTerminal`.

### 3.4 Event vocabulary (shared with real backends)

```ts
const EVENT_TYPES = [
  "system.ready","system.started","system.paused","system.resumed","system.reset","system.speed_changed",
  "task.created","task.assigned","task.status_changed","task.completed","task.failed","task.blocked","task.unblocked",
  "plan.created","research.completed","implementation.started","implementation.submitted",
  "qa.started","qa.passed","qa.failed","review.started","review.approved","review.changes_requested",
  "agent.message","agent.arrived","agent.paused","agent.resumed","agent.ambient",
] as const;

interface AgentEvent {
  id: string; type: AgentEventType; timestamp: number /* wall ms */; simTime: number;
  agentId?: AgentId; targetAgentId?: AgentId; taskId?: string;
  message: string; severity: "info"|"success"|"warning"|"error";
  feed: boolean;            // belongs in the human-facing activity feed?
  metadata?: Record<string, unknown>;
}
```

An event with `targetAgentId` and type `agent.message` or `task.assigned` is an agent-to-agent communication.

### 3.5 Snapshot and backend contract

```ts
type SimulationStatus = "ready" | "running" | "paused";
interface CommLink { id; from: AgentId; to: AgentId; kind: "message"|"assign"|"report"; startedAt; endsAt /* backend clock s */ }
interface WorldSnapshot { clock: number; status: SimulationStatus; speed: number; agents: AgentState[]; tasks: Task[]; links: CommLink[] }

interface AgentBackend {
  readonly kind: "mock" | "websocket"; readonly label: string; readonly simulated: boolean;
  connect(): Promise<void>; disconnect(): Promise<void>;
  start(); stop(); pause(); resume(); reset(); setSpeed(multiplier: number);   // all Promise<void>
  submitTask(input: CreateTaskInput): Promise<Task>;
  pauseAgent(id: AgentId): Promise<void>; resumeAgent(id: AgentId): Promise<void>;
  getSnapshot(): WorldSnapshot | null;
  subscribe(l: (e: AgentEvent) => void): () => void;
  subscribeSnapshot(l: (s: WorldSnapshot) => void): () => void;
}
```

**WebSocket protocol** (JSON): client sends `{type:"command", id:uuid, command: {name:"start"|"stop"|"pause"|"resume"|"reset"} | {name:"setSpeed", multiplier} | {name:"submitTask", input} | {name:"pauseAgent"|"resumeAgent", agentId}}`; server sends `{type:"event", event}`, `{type:"snapshot", snapshot}` (on connect and on change), `{type:"ack", id, ok:true, result?}` or `{type:"ack", id, ok:false, error}`. Selected by `NEXT_PUBLIC_AGENT_BACKEND=websocket` + `NEXT_PUBLIC_AGENT_WS_URL=wss://…`; otherwise `MockAgentBackend` wraps the `SimulationEngine` and ticks it with `setInterval` at `tickHz`; each real-time step is clamped to `maxStepSeconds` so a sleeping tab does not jump.

`backendClient.ts` exposes `getBackend()`, `connectBackend()` (wires events/snapshots into the store, returns a disposer) and a `commands` object: `start, pause, resume, reset, setSpeed, submitTask, submitTasks(inputs, gapMs=220) (staggered), pauseAgent, resumeAgent, autoStartIfReady`.

---

## 4. Simulation engine

### 4.1 Config (`simulationConfig.ts`, all durations in simulation seconds)

```ts
seed: 20261018, tickHz: 30, maxStepSeconds: 0.1, walkSpeed: 2.4,
speeds: [0.5, 1, 2, 4], defaultSpeed: 1, autoStart: true,
maxConcurrentTasks: 2, maxIterations: 3, qaFailRate: 0.3, reviewChangeRate: 0.25,
returnHomeDelay: 2.5, ambient: { minGap: 18, maxGap: 40, minStay: 6, maxStay: 10 }, linkDuration: 2.4,
durations: { intakeThink: 1.8, assign: 1.5, breakdown: 4.2, coordinate: 2, research: 5.5, publish: 1.4,
             implement: 6.5, fix: 4, submit: 1.4, test: 4.5, review: 4, report: 1.5, celebrate: 1.4,
             errorEmote: 1.6, announce: 2 },
phaseProgress: { queued:[0,0], intake:[0,.08], planning:[.08,.2], research:[.2,.36], implementation:[.36,.62],
                 testing:[.62,.78], review:[.78,.94], done:[1,1], failed:[0,0] },
maxEvents: 400
```

### 4.2 Engine shape

`class SimulationEngine { events: EventBus<AgentEvent>; snapshots: EventBus<WorldSnapshot>; start(); stop(); pause(); resume(); reset(); setSpeed(m); tick(realDtSeconds); submitTask(input): Task; pauseAgent(id); resumeAgent(id); getTask(id); snapshot(): WorldSnapshot; emitBootEvents() }`.

- Seeded RNG (mulberry32 or similar) from `seed`; per-task RNG seeded by `hashString(title) ^ seed ^ taskSeq` so each task's random QA/review outcomes are stable across runs.
- `tick(dt)`: if running, advance `clock += dt * speed`, step every `AgentController`, run the scheduler (Orchestrator picks the next queued task when `activeCount < maxConcurrentTasks`, highest priority then oldest), publish a snapshot.
- Agents hold an **action queue**: `move {to, slot?}`, `work {duration, status: working|thinking, activity, onProgress}`, `talk {target, duration, onStart}` (both agents face each other; the speaker's status is `communicating`), `emote {status: success|error, duration}`, `wait {until, timeout, face?}`, `call {fn}`. Walking follows `route()` from `worldLayout` (via a corridor at `CORRIDOR_Z`, so paths avoid furniture) at `walkSpeed`; `agent.arrived` is emitted on arrival (`feed:false`).
- **Phase choreography** (the task's `assignedAgentId` is whoever owns the current phase):
  - `queued → intake`: Orion walks to `hub`, thinks (`intakeThink`), picks up (`task.assigned` is *not* yet emitted), then `talk` Piper for `assign` seconds → emit `task.assigned` (agentId orion, target piper) + link kind `assign` → signal `delegated`.
  - `planning`: Piper walks to the `planning` board, `work` for `breakdown` (progress), creates subtasks from the work template → emit `plan.created` (metadata: subtasks) → `talk` Iris for `coordinate` → emit `agent.message` → signal `plan_ready`.
  - `research`: Iris walks to `research`, `work` for `research` (status `working`), sets `task.findings` from the template → `talk` Kai for `publish` → emit `research.completed` → signal `findings_published`.
  - `implementation`: Kai walks to `dev`, emit `implementation.started` (metadata iteration), `work` for `implement` (iteration 1) or `fix` (later iterations) while ticking subtasks `done` proportionally → `talk` Quinn for `submit` → emit `implementation.submitted` ("Kai submitted build #N …") → signal `submitted`.
  - `testing`: Quinn walks to `testing`, emit `qa.started` (`feed:false`), `work` for `test`. Outcome: `script.qaFailsOn.includes(iteration)` if a script exists, else `rng() < qaFailRate && iteration < maxIterations`. Fail ⇒ `emote error` with the failing test name from the template, `talk` Kai (`report`) → emit `qa.failed` (warning, metadata test) → `rework("tests_failed")`. Pass ⇒ `emote success`, `talk` Rex → emit `qa.passed` (success) → signal `tests_passed`.
  - `review`: Rex walks to `review`, emit `review.started` (`feed:false`), `work` for `review`. Same outcome logic with `reviewChangesOn` / `reviewChangeRate`. Changes ⇒ `emote error`, `talk` Kai → `review.changes_requested` → `rework("changes_requested")`. Approve ⇒ `emote success`, `talk` Orion → `review.approved` → signal `approved`.
  - `finalize` (after `done`): Orion returns to `hub`, `emote success` for `announce` → emit `task.completed` (success). Progress hits exactly 1.
  - `rework(signal)`: if `task.iteration >= maxIterations` ⇒ apply `abort` ⇒ `task.failed` (error, "… failed after N iterations"), unassign; else apply the signal (back to implementation, iteration+1).
- Each `talk` also pushes a `CommLink {from,to,kind,startedAt: clock, endsAt: clock+linkDuration}` into the snapshot so the 3D layer can draw an arc.
- `pauseAgent(id)`: sets `paused`, emits `agent.paused` (warning), and every non-terminal task assigned to that agent becomes `blocked` with reason "<Name> is paused" + `task.blocked`. The agent's queue freezes (animations damp to ~0.15 speed). `resumeAgent` reverses and emits `task.unblocked`.
- **Ambient behaviour** when an agent has no task: after `returnHomeDelay` it walks home; every `ambient.minGap..maxGap` seconds it may wander to `collab`/`lounge`/`planning` (slot = its `seat`), stay `minStay..maxStay`, emit `agent.ambient` (`feed:false`), and come back. Ambient never interrupts task work and never produces sounds.
- `reset()`: clear tasks, links, event counters, put agents at home slots idle, clock 0, status `ready`, emit `system.reset`.
- `emitBootEvents()`: `system.ready` plus one info line per agent ("Iris is at the Research Station").
- Snapshot publication: every tick while running; also immediately after any command.

### 4.3 Work content (`workContent.ts`)

A list of templates `{ match: RegExp, subtasks: string[], findings: string[], tests: string[], reviewNotes: string[] }` matched against `title + " " + description`, first match wins, plus a GENERIC fallback. Provide at least these domains: auth/jwt, payments/billing/stripe, search/index, realtime/websocket/presence, migration/pipeline/database, notification/email/digest, performance/latency/memory leak, deploy/kubernetes/canary/ci, mobile/offline/sync, export/csv/parquet/report, i18n/localisation, ai/llm/summarise, rate-limit/security/sso, api/endpoint/crud, ui/dashboard/component. Each template has 5–6 subtasks, 3 findings, 3 test names, 2 review notes, written like a real senior engineer would. `shortTitle(title, max=42)` truncates with an ellipsis.

### 4.4 Demo and backlog

- `DEMO_TASK`: title "Build a user authentication API with JWT support", description about register/login, short-lived access tokens with refresh rotation, middleware; priority high; `isDemo: true`; `script: { qaFailsOn: [1], reviewChangesOn: [] }`. At 1× it completes in ~75 simulation seconds with phases `queued → intake → planning → research → implementation → testing → implementation → testing → review → done` and `iteration === 2`.
- `TASK_BACKLOG`: 18 realistic product tasks with 1–2 sentence descriptions across the domains above, priorities mixed. Two are scripted: "Patch session token leak in password-reset flow" (critical, `qaFailsOn:[1], reviewChangesOn:[2]` ⇒ passes on iteration 3) and "Migrate legacy billing cron to event-driven pipeline" (high, `qaFailsOn:[1,2,3]` ⇒ fails after max iterations).
- `SPRINT_TASKS`: an 8-task subset containing both scripted tasks, two criticals (the hot-fix and "Fix memory leak in the image-processing worker"), and one low-priority export task that visibly waits. `pickRandomTask(exclude)` for a "Surprise me" button. `TASK_SUGGESTIONS` for dialog chips.

---

## 5. World layout (`worldLayout.ts`, pure data, metres; x left/right, z back(−)/front(+))

```
FLOOR = { width: 30, depth: 21, minX: -15, maxX: 15, minZ: -9.5, maxZ: 11.5 }; CORRIDOR_Z = 2.8
LocationId = hub | ops | research | product | dev | testing | review | planning | collab | lounge
ZoneId = operations | research | development | collaboration
```

| location | label | stand slot(s) | facing | zone |
|---|---|---|---|---|
| hub | Operations Hub | (0, 1.15) | π (towards −z) | operations |
| ops | Ops Console | (−8.25, 0.05) | π | research |
| research | Research Station | (−10, −3.75) | π | research |
| product | Product Station | (−6.5, −3.75) | π | research |
| dev | Engineering Station | (6.5, −3.75) | π | development |
| testing | QA Lab | (10, −3.75) | π | development |
| review | Review Station | (8.25, 0.05) | π | development |
| planning | Planning Board | (−11.4, 4.2), (−11.0, 5.6) | −π/2 | collaboration |
| collab | Collaboration Table | ring of 6 around (−5, 5.6), r 1.75 | inward | collaboration |
| lounge | Meeting Lounge | ring of 6 around (7.5, 6.2), r 2.1 | inward | collaboration |

Desks (`STATIONS`) sit 1.2 m behind their stand slot: ops (−8.25, −1.15), research (−10, −4.95), product (−6.5, −4.95), dev (6.5, −4.95), testing (10, −4.95), review (8.25, −1.15); rotation 0 (monitor faces +z). `HUB_CENTER = (0, −0.6)` is a round console; `MISSION_CONTROL = (0, −7.4)` is a large wall screen. Station owner map: ops→orion, research→iris, product→piper, dev→kai, testing→quinn, review→rex.

`route(from, to)`: go to the corridor z, travel along x, then to the destination; `pathLength`, `slotFor(location, seatIndex)`.

---

## 6. 3D rendering

### 6.1 Canvas (`WorldCanvas.tsx`)

`<Canvas shadows={high ? "percentage" : false} dpr={high ? [1,1.75] : [1,1.2]} camera={{ position: overview, fov: 40, near: 0.1, far: 220 }} gl={{ antialias: true, powerPreference: "high-performance" }}>`. ACESFilmic tone mapping, exposure 1.15, background `#080b10`, fog `#0a0e15` from 48 to 130. **Do not key the Canvas on quality** (see pitfalls); `shadows` and `dpr` are reactive props. Wrap children in drei `PerformanceMonitor` with `flipflops={2}` and `onFallback` ⇒ switch the store to quality `low` (auto). A `ReadySignal` component flips `sceneReady` after 3 rendered frames (drives the loading screen).

Click vs orbit: track pointer down/up, treat as a click only if movement ≤ 5 px; `onPointerMissed` with a real click clears the selection.

### 6.2 Environment (`OfficeEnvironment.tsx`)

Dark glass office at blue hour: large-format procedural floor tiles (canvas texture with soft variation and seams), back window wall with a dusk-gradient sky texture, side walls, low glass partitions between zones (transparent physical material, thin frames), potted plants (capsule/sphere primitives), slow-drifting dust motes (a `Points` cloud, 260 in high / 90 in low, animated in `useFrame`), and an **instanced skyline** behind the windows: ~260 towers in high quality / ~120 in low, one draw call, emissive lit-window texture. Zone labels on the floor are optional.

### 6.3 Workstations (`Workstations.tsx`)

- `Desk` per station: rounded desk slab, chair, 2–3 monitors with a scrolling procedural "code" screen texture whose emissive intensity rises when the owner's status is `working` (read `liveWorld` in `useFrame`). Invisible hit mesh for selection (selecting a station opens its panel and focuses the camera).
- `OperationsHub`: round console at `HUB_CENTER` with a glowing ring that pulses while Orion works.
- `MissionControl`: big wall screen at `MISSION_CONTROL` drawn on a canvas texture: active task titles, phase, progress bars, agent statuses. Redraw on a throttle (a few times per second) and only when the content signature changed.
- `PlanningBoard`: a whiteboard with up to 24 instanced sticky cards; cards appear as subtasks are created and change tint when done.
- `CollabTable` and `Lounge`: round table + 6 stools / low sofas, matching the ring slots.

### 6.4 Agents (`AgentCharacter.tsx`, `AgentAnimator.ts`, `AgentsLayer.tsx`)

- Procedural robots built from shared primitives (capsules, rounded boxes, cylinders): torso, head with a visor, two arms, two legs, a chest accent in the agent colour (emissive), a small status ring on the floor (agent colour, opacity blends with working/communicating/thinking weights), a selection ring (white, shown on select/hover), and an expanding pulse ring (green on `success`, amber on `error`). Each role has a distinct **rig** (scale 0.96–1.12, hip/shoulder/head heights, label height ~1.85–2.05) and a distinct head/torso silhouette: orchestrator taller with a crown-like visor, researcher with a lens, PM with a tablet, engineer with a headset, QA with twin sensors, reviewer with a monocle.
- `AgentAnimator`: each `AnimationState` is a pure pose function (idle breathing, walking leg/arm swing driven by actual speed, working forward lean + typing arms, thinking head tilt + hand to chin, communicating gesture, success bounce, error slump). Keep a weight per state, damp weights towards the target with delta time, return the weighted sum. Freeze (multiply animation speed by 0.15) when paused or when the sim is not running.
- Per frame: damp the root position towards `liveWorld` position (λ 12), damp heading (λ 9), snap if the jump is > 4 m (reset/teleport). Contact shadow is a soft radial blob sprite. Characters cast shadows.
- **Speech bubbles are DOM, not drei `<Html>`.** Inside `useFrame`, project `(x, labelY·scale, z)` with the camera and write `{x, y, scale, onScreen}` into `labelAnchors` (a module-level `Map<AgentId, LabelAnchor>`). `scale = clamp(6.5 / (2·tan(fov/2)·distance), 0.72, 1.05)`. `AgentLabelsOverlay` (mounted in `AppShell` above the canvas, `pointer-events: none`) renders one bubble per agent whose content comes from the store (name, activity or "Paused", tone border green/amber for success/error) and is visible when selected, hovered, paused, or (if "Agent speech indicators" is on) status is communicating/success/error; a single `requestAnimationFrame` loop sets each bubble's `transform` and `visibility` from the anchors. Bubble style: pill, dark glass, agent-colour dot, name bold, activity muted, 12 px, fade-in animation.
- `PathTrail` per agent: an `InstancedMesh` of up to 64 small discs (agent colour, additive-ish) flowing along the remaining path at 0.32 m spacing with a phase offset, plus a destination marker ring that breathes. Hidden when "Walking paths" is off.
- `CommLinks`: for every active `CommLink` draw a quadratic bezier arc between the two agents' chest heights with a glowing tube or line, and 2–3 instanced packet sprites travelling from `from` to `to` over `linkDuration`; colour from the sender; `assign` arcs slightly thicker.

### 6.5 Lighting (`Lighting.tsx`)

Blue-hour exterior + warm interior: one shadow-casting directional light with a tight orthographic frustum covering the floor only (shadow map 2048 in high, none in low), hemisphere light for cheap fill, a few warm point/rect lights over desks (no shadows), and a procedural `Environment` built from `Lightformer`s for PBR reflections (no HDR download). In low quality skip the environment and shadows. three r186 removed `PCFSoftShadowMap`; use `PCFShadowMap` with `shadow.radius`.

### 6.6 Camera (`cameraPresets.ts`, `CameraController.tsx`, `CameraBar.tsx`)

Presets (position → target): `overview` (−1.2, 20, 25)→(−1.2, 0, 0.8); `hub` "Operations Hub" (0.6, 7.2, 9.6)→(0, 1, −2.4); `development` (8.2, 6.2, 5.6)→(8.25, 0.6, −2.8); `research` (−8.2, 6.2, 5.6)→(−8.25, 0.6, −2.8); `collaboration` (−3.2, 8.4, 15)→(−6.8, 0.5, 4.6). Limits (`CAMERA_LIMITS`): minDistance 3.5, maxDistance 70, polar angle 0.18–1.36 rad, and the orbit target clamped to x ∈ [−14, 14], z ∈ [−8, 10], y ∈ [0, 3] so the user can never orbit out of the office.

Controller: drei `OrbitControls` with damping; store carries a `camera` request `{nonce, preset?, focus?, distance?}`; on a new nonce move position+target towards the goal with exponential damping (`THREE.MathUtils.damp`, λ≈4 for the target, λ≈2 for distance; frame-rate independent, settles in about a second); any user drag cancels the move. **Fit by free area**: the floating UI (nav rail, top bar, side panel, bottom sheet) reports its pixel insets to the store; the controller scales preset distance by `clamp(max(1.17·h/freeW, 0.96·h/freeH), 1, 2.2)` so the same part of the office stays visible on narrow layouts. Follow mode: when `followAgentId` is set, the target damps to the agent each frame and the camera keeps its offset; `Esc` or a drag releases it. Selecting an agent requests a focus at the agent with distance 7.5; selecting a station focuses the desk.

### 6.7 Interaction (`InteractionManager.tsx`)

Raycasting is R3F's pointer system against invisible simple hit meshes (cylinder on agents, box on desks). Hover sets `hovered: "agent:<id>" | "station:<id>"` and the cursor to pointer; pointer leaving the canvas clears hover. Click selects (respecting the 5 px click tolerance).

### 6.8 Shared resources

`sharedGeometry(key, make)` / `sharedMaterial(key, make)` pools created lazily on the client; `materials` (floor, wall, glass, metal, ceramic, joint, desk, screen off…) and `geometries` (box, capsule, cylinder, sphere, disc, ring); `accentMaterial(color)` and `tintedMaterial(color)` pooled per colour. Dispose per-instance materials on unmount. Canvas textures are built once and cached (`floorTexture, windowsTexture, skyTexture, screenTexture, blobTexture, glowTexture`) with correct colour space.

---

## 7. UI

### 7.1 Theme (`globals.css`)

Tailwind 4 `@theme` tokens: fonts Inter / JetBrains Mono; `ink-950 #06080c, ink-900 #0a0e14, ink-850 #0e131b, ink-800 #121823, ink-700 #1a2230, ink-600 #263041`; `mist-100 #eef2f7, mist-300 #b7c1cf, mist-500 #7c8797`; `signal #7cc4d6, signal-strong #9ad7e6, ok #6fe3a4, warn #f2b46a, bad #ff8a7a`. Dark only. Utilities: `.surface` (glass gradient, hairline border, inset highlight, deep shadow, 14 px backdrop blur), `.eyebrow` (10.5 px uppercase tracked label), `.btn .btn-primary .btn-ghost .btn-quiet`, `.field`, `.scroll-thin`, `.pulse-dot`, `.agent-bubble*`. Respect `prefers-reduced-motion` (disable bubble/pulse animations).

### 7.2 Layout (`AppShell.tsx`)

Full-viewport canvas; UI floats above it in a flex layout with 12 px gutters: left `NavRail` (icon rail: Overview, Agents, Tasks, Activity, Settings; logo on top; collapses to a bottom bar on mobile), top `TopBar`, right `SidePanel` 380–400 px (desktop) or a 40 dvh bottom sheet (mobile), `CameraBar` floating at the bottom of the free area, `TaskDialog` modal, `LoadingScreen` overlay. The shell measures the four UI regions with a `ResizeObserver` and writes `viewInsets` to the store. `WorldCanvas` is loaded with `next/dynamic` `ssr:false`. On mount: pick low quality up front if `hardwareConcurrency ≤ 4`, reduced motion, or width < 768, or `?quality=high|low` forces it; then `connectBackend()`; auto-start when `sceneReady` and `autoStart`.

Keyboard (ignored while typing in inputs or when the dialog is open): `Space` start/pause/resume, `N` new task, `B` load sprint (resumes if paused), `M` mute, `Esc` clear selection + stop following.

### 7.3 Top bar

Title "AI Agent World" + subtitle "Simulation mode · no live inference" (or "Live backend"); metrics derived from the store: Agents, Active (agents with a task or working/thinking/communicating and not paused), Pending (non-terminal tasks), Completed; Status dot (green pulsing running / amber paused / grey ready). Right: `SimulationControls`: Start / Pause / Resume, speed segmented control 0.5× 1× 2× 4×, mute toggle (Volume2/VolumeX), Reset with inline "Confirm reset / Cancel", **Run demo**, **Load sprint**, **New task** (primary).

### 7.4 Side panel tabs

- **Overview**: when there are no tasks, a callout "Watch the team ship a feature" explaining the demo in one sentence naming the agents, with Run demo / Load sprint / New task buttons and a line about the sprint; compact agent roster (avatar, name, role, status pill); "In flight" task cards (compact) with an "N active" counter; "Latest activity" (8 items).
- **Agents**: roster; selecting an agent shows `AgentDetails`: avatar, name, role, tagline; "Now" box with activity, location, current task; buttons Assign task (opens the dialog with `preferredAgentId`), Pause/Resume agent, Follow/Stop following, Focus camera; tabs "Activity · N" and "Communication · N" listing that agent's events.
- **Tasks**: New task / Demo / Sprint buttons; filter segmented control All / Active / Queued / Done with counts; sorted list (active first, then queued, then done; newest first within a group) of `TaskCard`s: id, status badge, DEMO tag, priority tag, title, 2-line description, progress bar in the owner's colour (green when done, amber when blocked), owner avatar + phase label or "Waiting for orchestrator", relative time, a rework counter when `iteration > 1`. Selecting opens `TaskDetails`: header, description, **Workflow** (phase label, progress, owner, "Reworked N× after QA/review feedback" in amber), **Task cards · N** (subtasks with checkmarks), **Research findings**, **Timeline** (history transitions with signal and clock), blocked reason banner, failure banner.
- **Activity**: full feed with a "Messages only" checkbox; each row has a severity dot, time, agent avatar(s) (sender → target), message; clicking jumps to the agent.
- **Settings**: Rendering (High quality / Performance + explanation, auto-switch note), Overlays (Walking paths & destinations, Agent speech indicators), **Sound** (Sound effects (M), Ambient office hum, Volume slider, one-line explanation), Backend (connection dot, label, error, simulation disclaimer, env vars), Simulation (a definition list of the key config values: max concurrent tasks, max iterations, …).

### 7.5 Task dialog

Modal with title, suggestion chips (demo title + 4 backlog titles), description, priority segmented control, "Use demo task", "Surprise me" (fills a random backlog item: title, description, priority), Cancel, Submit. Shows a note when opened with a preferred agent ("Requested from Kai — routed via the Orchestrator so the full workflow still applies"). Footer: "Simulation mode — agents act out the workflow; no AI inference or code execution happens." On submit: close, select the task, request the `collaboration` camera.

### 7.6 Loading screen

Dark overlay with the logo, three steps ("Booting simulation engine", "Building the office", "Waking up agents") ticking over at least 1.1 s total, fades out when `sceneReady`.

---

## 8. Audio (`audio/soundEngine.ts`, `useSoundEffects.ts`)

Procedural Web Audio, no files. Singleton `SoundEngine` with lazy `AudioContext`, master gain → `DynamicsCompressor` (threshold −18, ratio 6) → destination. Settings `{enabled: true, volume: 0.6, ambient: true}` persisted in `localStorage` key `aaw.sound.v1`, mirrored in the store (`sound`, `setSound(patch)`). Unlock on the first `pointerdown`/`keydown` (resume the context). `play(event)` maps event types to cues, with a per-type throttle (120 ms; 160 ms for arrivals; per-agent key for messages):

- `task.created`: two soft sine blips 880→1174 Hz.
- `task.assigned` / `agent.message`: two-note triangle ping at the **speaker's own pitch** (orion C5 523, piper E5 659, iris D5 587, kai A4 440, quinn F5 698, rex G4 392) then a fifth above.
- `plan.created`: quick major-triad arpeggio at the speaker pitch. `research.completed`: band-pass noise sweep 600→3200 Hz + a sine. `implementation.started`: six short noise clicks (keyboard). `implementation.submitted`: noise whoosh 300→4000 Hz.
- `qa.passed` / `review.approved`: ascending E5 major triad chime. `qa.failed` / `review.changes_requested`: sawtooth 220→150 Hz + square 165→110 Hz buzz, 0.28 s.
- `task.completed`: C5-E5-G5-C6 triangle fanfare + sustained C6. `task.failed`: sine thud 110→55 Hz + low noise + descending minor three notes.
- `task.blocked` / `agent.paused`: two square blips down; `task.unblocked` / `agent.resumed`: two up. `system.started/resumed`: sine 220→440; `system.paused`: 440→220; `system.reset`: noise sweep 3000→120. `agent.arrived` with a task: a 90→60 Hz footstep thump. Ambient events: silence.
- Ambient bed while `simStatus === "running"` and ambient is on: looped brown-ish noise (low-pass 260 Hz) + 58 Hz sine hum at very low gain, 1.5 s fade in, 0.25 s fade out.

`useSoundEffects()` (mounted once in `AppShell`): loads settings into the store, subscribes to backend events → `play`, adds unlock listeners, mirrors store `sound` → `configure` and `simStatus` → `setRunning`.

---

## 9. Tests (`scripts/simulate.ts`, run by `npm test`)

A plain Node script that imports the engine directly and prints ✓/✗ lines, exiting 1 on any failure. Required checks:

1. State machine: `testing --tests_failed-->` allowed; `queued --approved-->` rejected; `done` terminal.
2. Demo workflow: boot events ≥ 3; after `start()` + `submitTask(DEMO_TASK)` and ticking at 1/30 s, the task reaches `done`/`completed` with `iteration === 2`, `progress === 1`, all subtasks done, findings non-empty; events include task.created, task.assigned, plan.created, research.completed, implementation.started, implementation.submitted, qa.failed, qa.passed, review.approved, task.completed; ≥ 7 agent-to-agent communications; phase order exactly `queued → intake → planning → research → implementation → testing → implementation → testing → review → done`; completes in < 100 sim s.
3. Determinism: a fresh engine finishes the demo within 0.05 sim s of the first run.
4. Pause/resume/reset: paused snapshot does not change; resumed progresses; `pauseAgent(owner)` makes the task `blocked`, `resumeAgent` unblocks; a random task terminates; `reset()` returns 0 tasks, `ready`, clock 0, agents idle at stations.
5. Concurrency + priority: submit low, medium, critical ⇒ critical and medium start, low stays queued; all three eventually finish.
6. Backlog + sprint: ≥ 15 tasks with unique titles; at most 2 fall through to the generic template; the 8-task sprint terminates; the scripted hot-fix ends `done` at iteration 3; the scripted migration ends `failed`; ≥ 5 of 8 complete; every task has subtasks and findings.

Optional `scripts/e2e.py` (Playwright): load the page, run the demo at 4×, assert the Completed metric increments, exercise pause/resume/reset, save screenshots.

---

## 10. Known pitfalls (learned the hard way — read before coding)

1. **drei `<Html>` + React 19 + StrictMode.** `Html` creates a nested React root in a layout effect and unmounts it synchronously in the cleanup. React 19 throws "Attempted to synchronously unmount a root while React was already rendering" on every StrictMode double-invoke, and any Canvas remount adds "removeChild … not a child of this node". Do not use `<Html>`. Use the `labelAnchors` + DOM overlay pattern in §6.4.
2. **Never remount the Canvas** (`key={quality}` or similar). It tears down every WebGL resource and any nested roots mid-render. `shadows` and `dpr` are reactive props; environment/shadow components read quality from the store.
3. **three r186 removed `PCFSoftShadowMap`.** Use `PCFShadowMap` + `shadow.radius`.
4. **Node runs the simulation in tests.** Keep `src/simulation`, `src/tasks`, `src/events`, non-React `src/agents/*.ts` free of React/three/Next/`@/` imports and use explicit `.ts` import extensions; otherwise `node --experimental-strip-types` cannot load them.
5. **Audio needs a user gesture.** Create/resume the `AudioContext` only from a pointer/key handler; before that, `play()` must be a silent no-op.
6. **Tab sleep.** Clamp each real-time step to `maxStepSeconds` (0.1 s) in the mock backend tick, or agents teleport after the tab was hidden.
7. **Do not re-render React at 30 Hz.** Snapshots go into the mutable `liveWorld`; the Zustand mirror updates at most every 160 ms unless a signature of statuses/phases changed. Position changes alone must never cause React renders.
8. **Floating UI covers the world.** Without the inset-aware camera fit (§6.6), presets authored on a wide screen cut off the office on laptops and phones.
9. **Click vs orbit.** Without the 5 px movement tolerance, every camera drag that ends on empty space clears the selection.

---

## 11. Phases and acceptance criteria

**Phase 1 — Scaffold + domain + engine (no 3D).** Next app boots to a placeholder page; all types, state machine, scheduler, config, world layout, work content, demo + backlog; `SimulationEngine` complete with `MockAgentBackend`; `npm test` passes all checks in §9.

**Phase 2 — State + HTML UI.** Store with `liveWorld` + throttled mirror, backend client and `commands`, full shell with nav rail, top bar, side panel tabs, task dialog, loading screen, keyboard shortcuts. The app is fully usable with the 3D area blank: run the demo and watch task cards, agent statuses and the activity feed move.

**Phase 3 — World.** Canvas, lighting, environment, workstations, camera presets + controller with inset fit, interaction. Static office looks finished: glass partitions, skyline, desks with glowing screens, mission-control wall, planning board, collab table, lounge.

**Phase 4 — Agents.** Procedural robots per role, animator, walking along routed paths, path trails, comm arcs with packets, DOM speech bubbles, selection/hover/follow. Demo is watchable end to end: Orion → Piper → Iris → Kai → Quinn (fail) → Kai → Quinn (pass) → Rex → Orion.

**Phase 5 — Mission control + planning board live.** Wall screen and sticky cards reflect tasks/subtasks; desk screens glow when the owner works; hub ring pulses.

**Phase 6 — Quality + performance.** `PerformanceMonitor` fallback, low path (no shadows/env, fewer towers, capped DPR), reduced motion, mobile bottom sheet, `?quality=` flag. 60 fps in high on a modern laptop; no console errors with StrictMode on.

**Phase 7 — Sound.** §8 complete; mute button, `M`, settings section; verify in the browser that an `AudioContext` is running after a click and cues fire during the demo.

**Phase 8 — Polish + docs.** `README.md` (run, checks, architecture, workflow, demo/backlog, connecting a real backend, pitfalls), `scripts/e2e.py`. Lint clean, typecheck clean, tests green.

**Definition of done.** A newcomer runs `npm install && npm run dev`, presses **Run demo**, and within 90 seconds has watched six distinct robots walk between labelled stations, exchange glowing messages, fail a test, fix it, get approved, and celebrate, with sound, with every step mirrored in the side panel and the activity feed — and the top bar never once pretended an LLM was involved.
