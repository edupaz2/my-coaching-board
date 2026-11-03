# Ice Hockey Drill Animator Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** A desktop web app where a coach builds an ice hockey drill with event-based timing, plays it as an animation, and exports it as an MP4/GIF for WhatsApp.

**Architecture:** A pure-TypeScript core (model, rule-based timing engine, puck timeline, frame sampling) with no DOM dependency, fully unit-tested. Rendering draws one frame onto a canvas through a thin `DrawTarget`, and is used by both the live player and the exporter. A small Preact UI sits on top of an undoable store.

**Tech Stack:** TypeScript, Vite, Vitest (+ jsdom), Preact, WebCodecs + `mp4-muxer` for MP4, `gifenc` for GIF.

**Spec:** `docs/superpowers/specs/2026-11-03-hockey-drill-animator-design.md`

## Global Constraints

- Web app, TypeScript, no server; everything runs on the coach's device.
- Target is a **desktop (laptop) browser** for editing. Phones only play the exported file.
- Fresh start. Do not copy code from `/Users/epaz/development/bitbucket/*`.
- One puck. Pass and shot speed is **one constant**, `PUCK_SPEED`, the same for both.
- Version 1 shows **one round** of **one drill**. Rink is `'full'` or `'half'`, chosen by the coach.
- The coach edits **rules only**. The timeline is read-only.
- The app only **warns** on timing conflicts and never changes the drill silently.
- Export: MP4 default and GIF secondary, 1280x720, at most 60 s, under 16 MB.
- Drill file format is versioned (`version: 1`) and import treats files as untrusted.

## Review Focus

1. A rule cycle ("A starts when B ends, B starts when A ends") must give a warning, not hang or crash.
2. Deleting a player or action that other actions and rules point to must leave no dangling references.
3. A malformed, oversized or future-version drill file must be rejected with a message and leave saved drills untouched.
4. An empty drill, a zero-length path or a zero speed must not produce NaN, and playback must not break.
5. Export in a browser without WebCodecs, or a result over 16 MB, must give a clear message and never a silently broken file.

---

## Domain decisions (used by every task)

- Units are **meters** and **seconds**. Rink is 60.96 x 25.9 m, with the origin at the top-left, x along the length. Full rink: x in 0..60.96. Half rink: x in 0..30.48.
- `PUCK_SPEED = 20` m/s. Speed presets `slow 4`, `medium 6`, `fast 8` m/s.
- A player's own actions are in list order. A **skate** starts no earlier than the previous skate of the same player ends. **Throw, drop and pickup** are instants and do not block.
- A pass is aimed at an explicit **receive point**, which is a waypoint on the receiver's path or a fixed point. Landing time is release time plus distance divided by `PUCK_SPEED`. The check is spatial: at landing time the receiver must be within `RECEIVE_RADIUS = 1.0` m of the receive point.
- A "crosses the blue line" trigger is a skate waypoint that the coach puts on the line.

---

### Task 1: Scaffold and geometry

**Files:**
- Create: `package.json`, `tsconfig.json`, `vite.config.ts`, `index.html`, `src/main.tsx`, `src/geometry/geometry.ts`, `tests/geometry.test.ts`, `.gitignore`, `README.md`

**Interfaces:**
- Produces:
  - `type Point = { x: number; y: number }`
  - `distance(a: Point, b: Point): number`
  - `pathLength(points: Point[]): number`
  - `pointAlong(points: Point[], dist: number): Point` (clamped to the ends; empty path throws)
  - `cumulativeLengths(points: Point[]): number[]`

- [ ] **Step 1: Failing test** `tests/geometry.test.ts`: `pathLength([{x:0,y:0},{x:3,y:4}]) === 5`; `pointAlong` at 2.5 on that path is `{x:1.5,y:2}`; at 99 clamps to `{x:3,y:4}`; at -1 clamps to start; `pathLength([])` and a single point are 0 (Review Focus 4).
- [ ] **Step 2:** Run `npx vitest run tests/geometry.test.ts`. Expected: FAIL, module missing.
- [ ] **Step 3:** Scaffold with `npm create vite@latest . -- --template preact-ts`, add `vitest`, `jsdom`. Set the vitest environment to `jsdom`. Implement the functions in `src/geometry/geometry.ts`. Add `.superpowers/` to `.gitignore`. README states how to run `npm run dev` and `npm test`.
- [ ] **Step 4:** Run `npx vitest run`. Expected: PASS.
- [ ] **Step 5: Commit** `feat: scaffold app and geometry helpers`.

---

### Task 2: Model types, factories, validation

**Files:**
- Create: `src/model/types.ts`, `src/model/drill.ts`, `tests/model.test.ts`

**Interfaces:**
- Consumes: `Point`.
- Produces (in `types.ts`):
  - `Role = 'goalie'|'forward'|'defence'`; `RinkMode = 'full'|'half'`; `SkateStyle = 'forward'|'backward'|'withPuck'`
  - `Player { id; number; role; start: Point; speed: number }`; `Cone { id; pos: Point }`
  - `EventRef = { actionId: string; at: 'start'|'end' } | { actionId: string; at: 'waypoint'; index: number }`
  - `Trigger = { kind: 'begin' } | { kind: 'after'; event: EventRef; delay: number }`
  - `SkateAction { id; type: 'skate'; trigger; playerId; path: Point[]; style }` (path excludes the start position)
  - `ThrowAction { id; type: 'throw'; trigger; playerId; kind: 'pass'|'shot'; oneTouch: boolean; target: { kind: 'player'; playerId: string; receiveAt: Point | { skateId: string; waypoint: number } } | { kind: 'net'; net: 'left'|'right' } }`
  - `DropAction { id; type: 'drop'; trigger; playerId }`; `PickUpAction { id; type: 'pickup'; trigger; playerId }`
  - `Action = SkateAction | ThrowAction | DropAction | PickUpAction`
  - `ArriveTogether { id; anchor: EventRef; skateIds: string[] }`
  - `Puck = { holder: string } | { at: Point }`
  - `Drill { version: 1; name: string; rink: RinkMode; players: Player[]; cones: Cone[]; puck: Puck; actions: Action[]; rules: ArriveTogether[] }`
- Produces (in `drill.ts`): `PUCK_SPEED`, `SPEED_PRESETS`, `newDrill(name: string, rink: RinkMode): Drill`, `newId(prefix: string): string`, `validateDrill(value: unknown): { ok: true; drill: Drill } | { ok: false; error: string }`

- [ ] **Step 1: Failing tests** in `tests/model.test.ts`:
  - `newDrill('x','half')` has `version 1`, `rink 'half'`, empty arrays.
  - `validateDrill` accepts a valid drill and rejects each of: non-object, `version: 2`, a missing `players` array, a player with a negative or non-finite speed, a path point that is not a number, a trigger or rule that references an unknown action id, a duplicate id, and more than 500 actions (size cap).
  - It never throws on `null`, a string or a deeply nested object (Review Focus 3).
- [ ] **Step 2:** Run it. Expected: FAIL.
- [ ] **Step 3:** Implement the types and `validateDrill` as hand-written checks, with no schema library. Return a specific message for each failure.
- [ ] **Step 4:** Run it. Expected: PASS.
- [ ] **Step 5: Commit** `feat: drill model and validation`.

---

### Task 3: Mutations and history (undo, delete)

**Files:**
- Create: `src/model/mutations.ts`, `src/model/history.ts`, `tests/mutations.test.ts`, `tests/history.test.ts`

**Interfaces:**
- Consumes: `Drill`, `Action`, `Player`, `Cone`, `ArriveTogether`.
- Produces (all pure, return a new `Drill`):
  - `addPlayer(d, role, start): Drill` (auto-assigns the next `number`, default speed `SPEED_PRESETS.medium`)
  - `addCone(d, pos)`, `addAction(d, action)`, `addRule(d, rule)`, `setPlayerSpeed(d, id, speed)`, `setRink(d, mode)`
  - `removeAction(d, id)`, `removePlayer(d, id)`, `removeCone(d, id)`, `removeRule(d, id)`
- Produces (`history.ts`): `class History<T> { constructor(initial: T); get current(): T; push(next: T): void; undo(): boolean; redo(): boolean; canUndo: boolean; canRedo: boolean }`

- [ ] **Step 1: Failing tests:**
  - `removePlayer` also removes that player's actions, throws targeting them, rules mentioning removed skates, and passes the puck holder to nothing when they held it (`puck` becomes `{at: start}`).
  - `removeAction` resets any trigger that pointed at that action to `{kind:'begin'}`, drops it from rules (a rule with no members is removed), and clears a `receiveAt` pointing at a removed skate by converting it to a fixed `Point`.
  - After any removal, `validateDrill` still returns `ok` (Review Focus 2).
  - History: `push` clears redo; `undo` then `redo` round-trips; `undo` on the initial state returns `false`.
- [ ] **Step 2:** Run it. Expected: FAIL.
- [ ] **Step 3:** Implement. Removal cascades live in one `pruneReferences(d: Drill): Drill` helper.
- [ ] **Step 4:** Run it. Expected: PASS.
- [ ] **Step 5: Commit** `feat: drill mutations with cascade delete and undo history`.

---

### Task 4: Rink geometry

**Files:**
- Create: `src/rink/rink.ts`, `tests/rink.test.ts`

**Interfaces:**
- Produces: `RINK_LENGTH = 60.96`, `RINK_WIDTH = 25.9`, `rinkBounds(mode: RinkMode): { width: number; height: number }`, `netPosition(net: 'left'|'right'): Point` (goal-line center), `blueLineX(side: 'left'|'right'): number`, `clampToRink(p: Point, mode: RinkMode): Point`

- [ ] **Step 1: Failing tests:** half-rink width is `RINK_LENGTH/2`; `netPosition('left')` is `{x:3.35, y: RINK_WIDTH/2}` and `'right'` is mirrored; `blueLineX('left') === 22.86`; `clampToRink` pulls a point outside the half rink back to its edge.
- [ ] **Step 2:** Run it. Expected: FAIL.
- [ ] **Step 3:** Implement.
- [ ] **Step 4:** Run it. Expected: PASS.
- [ ] **Step 5: Commit** `feat: rink dimensions`.

---

### Task 5: Timing engine, part 1: triggers, delays, throws, cycles

**Files:**
- Create: `src/engine/schedule.ts`, `tests/schedule.test.ts`

**Interfaces:**
- Consumes: `Drill`, geometry, `PUCK_SPEED`.
- Produces:
  - `ScheduledItem { id: string; start: number; end: number; from?: Point; to?: Point }` (`from`/`to` are set for throws: release point and landing point)
  - `WarningCode = 'cycle'|'late'|'receiver-not-at-spot'|'puck-missing'|'bad-reference'`
  - `Warning { code: WarningCode; actionId?: string; ruleId?: string; message: string; seconds?: number }`
  - `Schedule { items: Record<string, ScheduledItem>; duration: number; warnings: Warning[] }`
  - `computeSchedule(d: Drill): Schedule`
  - `positionOfPlayer(d: Drill, s: Schedule, playerId: string, t: number): Point` (before any skate it is `start`; during a skate it is along the path at the player's speed; after it is the path end)
  - `eventTime(s: Schedule, ref: EventRef, d: Drill): number | undefined` (`waypoint` time is the skate's start plus the length to that waypoint divided by the player's speed)

- [ ] **Step 1: Failing tests** (build drills by hand):
  - One player, speed 5, path of length 10 with `begin` trigger: item is `start 0, end 2`, duration 2.
  - Player B's skate has `{after: {A skate, 'end'}, delay: 1.5}`: starts at A's end plus 1.5.
  - A "starts when" event on a waypoint: B starts when A reaches waypoint 0, at exactly the time A takes to reach it.
  - A player's second skate does not start before their first ends, even with a `begin` trigger.
  - A pass over 20 m: `end - start === 20 / PUCK_SPEED`, `from` is the thrower's position at release, `to` is the receive point.
  - A shot at the left net lands at `netPosition('left')`.
  - Cycle: A after B's end, B after A's end gives a `cycle` warning for both ids, and the function terminates (Review Focus 1).
  - A trigger pointing at a missing action gives `bad-reference`, never an exception.
  - Empty drill: `duration 0`, no warnings. Speed 0 gives a warning-free finite duration, not `Infinity` or NaN (Review Focus 4).
- [ ] **Step 2:** Run it. Expected: FAIL.
- [ ] **Step 3:** Implement `computeSchedule` as an iterative resolve loop. Each pass schedules every action whose trigger event and same-player predecessors are resolved. A pass that makes no progress marks the remaining actions as `cycle`. Treat speed ≤ 0 as a duration of 0 and emit no NaN.
- [ ] **Step 4:** Run it. Expected: PASS.
- [ ] **Step 5: Commit** `feat: timing engine with triggers, delays and passes`.

---

### Task 6: Timing engine, part 2: arrive-together and spatial checks

**Files:**
- Modify: `src/engine/schedule.ts`
- Test: `tests/schedule-rules.test.ts`

**Interfaces:**
- Consumes: Task 5 output.
- Produces: `computeSchedule` also applies `Drill.rules` and emits spatial warnings (no new exports).

- [ ] **Step 1: Failing tests:**
  - **Arrive together:** A passes 20 m (1 s flight) to a receive point on B's path. B's skate has a different speed, with a rule anchored on the pass `end`. B's skate ends exactly at the pass landing time, so B's start is the anchor minus B's duration. Both a slow and a fast B give a correct start.
  - Two skates with different speeds and lengths in one rule both end at the same anchor.
  - **Impossible:** if the anchor minus the duration is earlier than the skate's trigger time, the skate starts at its trigger time and a `late` warning carries `seconds` equal to the lateness.
  - A player may wait at the destination: an early arrival gives no warning.
  - `receiver-not-at-spot`: the receiver is more than `RECEIVE_RADIUS` from the receive point at landing time. A receiver who is on it produces no warning.
  - A rule that references a missing skate gives `bad-reference`.
  - A player appearing in two rules is allowed; if the two give different starts, a `late` warning names the rule (spec: conflicts are warnings).
- [ ] **Step 2:** Run it. Expected: FAIL.
- [ ] **Step 3:** Extend the resolve loop: a skate in a rule resolves once its anchor is known. Its start is `max(trigger time, previous skate end, anchor - duration)` only when the third term is feasible; if not, use the max of the first two and warn. After all items resolve, run the spatial receive check using `positionOfPlayer`.
- [ ] **Step 4:** Run `npx vitest run`. Expected: all PASS.
- [ ] **Step 5: Commit** `feat: arrive-together rules and receive-spot warnings`.

---

### Task 7: Puck timeline, frame sampling, steps

**Files:**
- Create: `src/engine/puck.ts`, `src/engine/frame.ts`, `src/engine/steps.ts`, `tests/frame.test.ts`, `tests/steps.test.ts`

**Interfaces:**
- Consumes: `Drill`, `Schedule`, `positionOfPlayer`.
- Produces:
  - `puckPositionAt(d: Drill, s: Schedule, t: number): Point | undefined` (held: at the holder's position; in flight: linear from `from` to `to`; after a net shot: stays at the net; dropped: stays where dropped; `undefined` only if it cannot be known)
  - `type ActionPhase = 'past' | 'active' | 'future'`
  - `Frame { time: number; players: { id: string; pos: Point; activeActionId?: string }[]; puck: Point | undefined; phases: Record<string, ActionPhase> }`
  - `frameAt(d: Drill, s: Schedule, t: number): Frame` (t is clamped to `[0, duration]`)
  - `assignSteps(d: Drill): Record<string, number>` (actions grouped by trigger event identity; `begin` is step 1; others are ordered by the earliest scheduled time among the group; parallel actions share a number)

- [ ] **Step 1: Failing tests:**
  - Puck held by P1 follows P1 along their path. After P1 passes, it flies at `PUCK_SPEED` to the receive point, then follows the receiver.
  - **Drop and pickup:** the puck stays put after a drop, and a pickup by a player within 1.5 m resumes carrying. A pickup when the puck is farther away gives a `puck-missing` warning.
  - **One-touch:** a throw with `oneTouch: true` triggered `after` the incoming pass end with delay 0 means the puck never rests at the receiver.
  - `frameAt` at `t < 0` and `t > duration` clamps. With an empty drill it returns a frame with no NaN.
  - Phases: before an action starts it is `future`, between start and end `active`, after `past`.
  - `assignSteps`: two actions with the same trigger event share a number, and numbers are consecutive from 1.
- [ ] **Step 2:** Run it. Expected: FAIL.
- [ ] **Step 3:** Implement. `puckPositionAt` walks the puck-affecting items (throws, drops, pickups) sorted by time. Emit pickup warnings from `computeSchedule` through a helper in `puck.ts` that `schedule.ts` calls, so warnings stay in one list.
- [ ] **Step 4:** Run `npx vitest run`. Expected: PASS.
- [ ] **Step 5: Commit** `feat: puck timeline, frame sampling and step numbers`.

---

### Task 8: Acceptance fixtures (the four book drills, one round each)

**Files:**
- Create: `tests/fixtures/russian-stretch.ts`, `tests/fixtures/sweden.ts`, `tests/fixtures/short-pass.ts`, `tests/fixtures/short-pass-wide.ts`, `tests/fixtures/one-touch-shoot.ts`, `tests/fixtures.test.ts`

**Interfaces:**
- Consumes: the Task 2/3 factories and mutations, `computeSchedule`, `frameAt`.
- Produces: `russianStretch(): Drill`, `sweden(): Drill`, `shortPassShort(): Drill`, `shortPassWide(): Drill`, `oneTouchPassAndShoot(): Drill`.

- [ ] **Step 1: Failing test** `tests/fixtures.test.ts`, for each fixture:
  - `computeSchedule` returns **no warnings**.
  - `duration > 0` and `duration < 60`.
  - Every player's final position is inside the rink.
  - Specific checks: Russian Stretch has a delayed start for player 2 and a corner pickup, then a shot at a net. Sweden has two cones, a `backward` skate and at least one `oneTouch` throw. The short-pass drills have a goalie `throw` and the goalie's position never changes. The one-touch fixture uses `rink: 'half'`.
- [ ] **Step 2:** Run it. Expected: FAIL.
- [ ] **Step 3:** Build each fixture with the mutation helpers, with positions taken from the book diagrams (approximate is fine; timing relations are what matter).
- [ ] **Step 4:** Run it. Expected: PASS. If a fixture cannot be expressed, stop and record the gap in `docs/superpowers/specs/` before changing the engine.
- [ ] **Step 5: Commit** `test: acceptance fixtures from the four book drills`.

---

### Task 9: Renderer

**Files:**
- Create: `src/render/draw.ts`, `src/render/styles.ts`, `tests/draw.test.ts`

**Interfaces:**
- Consumes: `Drill`, `Frame`, `assignSteps`, `rinkBounds`.
- Produces:
  - `interface DrawTarget { setAlpha(a: number): void; line(points: Point[], style: LineStyle): void; circle(c: Point, r: number, fill: string): void; text(t: string, p: Point, style: TextStyle): void; rect(...): void; clear(): void }` plus `canvasTarget(ctx: CanvasRenderingContext2D, scale: number): DrawTarget`
  - `PHASE_ALPHA = { past: 0.3, active: 1, future: 0.3 }`
  - `drawFrame(target: DrawTarget, d: Drill, frame: Frame, steps: Record<string, number>): void`
  - `lineStyleFor(action: Action): LineStyle` (forward solid, backward dotted, withPuck wavy, pass/shot dashed)

- [ ] **Step 1: Failing tests** using a recording `DrawTarget`:
  - A `past` action's path is drawn at alpha 0.3, the `active` one at 1, and a `future` one at 0.3.
  - Each action draws a badge whose text equals its step number, and the badge shape differs from the player-number circle.
  - The active player is drawn with a highlight ring, and others are not.
  - The half rink draws only the half-width rink.
  - A frame with `puck: undefined` draws no puck and doesn't throw.
- [ ] **Step 2:** Run it. Expected: FAIL.
- [ ] **Step 3:** Implement. Draw order: rink, cones, action paths, badges, players, puck. Rink artwork is simple lines (boards, red line, blue lines, faceoff circles, creases, nets).
- [ ] **Step 4:** Run it. Expected: PASS.
- [ ] **Step 5: Commit** `feat: canvas renderer with phase transparency and badges`.

---

### Task 10: Playback controller

**Files:**
- Create: `src/playback/controller.ts`, `tests/controller.test.ts`

**Interfaces:**
- Produces: `class PlaybackController { constructor(getDuration: () => number, now?: () => number); play(): void; pause(): void; seek(t: number): void; setRate(r: number): void; readonly time: number; readonly playing: boolean; tick(): void; onChange(cb: () => void): () => void }`

- [ ] **Step 1: Failing tests** with an injected clock: advances at the rate (0.5 gives half speed); stops and pauses at `duration`; `seek` clamps to `[0, duration]`; `play` at the end restarts from 0; a duration of 0 never loops or throws (Review Focus 4).
- [ ] **Step 2:** Run it. Expected: FAIL.
- [ ] **Step 3:** Implement. The UI calls `tick()` from `requestAnimationFrame`.
- [ ] **Step 4:** Run it. Expected: PASS.
- [ ] **Step 5: Commit** `feat: playback controller`.

---

### Task 11: Persistence and file import/export

**Files:**
- Create: `src/io/storage.ts`, `src/io/file.ts`, `tests/storage.test.ts`, `tests/file.test.ts`, `docs/drill-file-format.md`

**Interfaces:**
- Produces:
  - `listDrills(): { id: string; name: string; updatedAt: number }[]`, `loadDrill(id: string): Drill | undefined`, `saveDrill(id: string, d: Drill): void`, `deleteDrill(id: string): void` (localStorage, key prefix `mcb.drill.`)
  - `serializeDrill(d: Drill): string` (JSON of `{ app: 'my-coaching-board', version: 1, drill }`)
  - `parseDrillFile(text: string): { ok: true; drill: Drill } | { ok: false; error: string }` (max 1 MB; uses `validateDrill`)
  - `downloadDrill(d: Drill): void`, `readDrillFile(file: File): Promise<ReturnType<typeof parseDrillFile>>`

- [ ] **Step 1: Failing tests:** save/list/load/delete round-trip; a round trip through `serializeDrill` and `parseDrillFile` gives an equal drill; rejected with a message: not JSON, wrong `app`, `version: 2`, text over 1 MB, invalid drill. A rejected import does not alter `localStorage` (Review Focus 3). A corrupt stored entry is skipped by `listDrills`, not thrown.
- [ ] **Step 2:** Run it. Expected: FAIL.
- [ ] **Step 3:** Implement. Write `docs/drill-file-format.md` describing the format and the rule for bumping `version`.
- [ ] **Step 4:** Run it. Expected: PASS.
- [ ] **Step 5: Commit** `feat: local storage and drill file import/export`.

---

### Task 12: Editor UI

**Files:**
- Create: `src/ui/App.tsx`, `src/ui/store.ts`, `src/ui/RinkEditor.tsx`, `src/ui/RulesPanel.tsx`, `src/ui/Timeline.tsx`, `src/ui/PlayBar.tsx`, `src/ui/DrillList.tsx`, `src/ui/warnings.ts`, `tests/store.test.ts`, `tests/timeline.test.tsx`, `tests/warnings.test.ts`

**Interfaces:**
- Consumes: everything above.
- Produces:
  - `store.ts`: `createStore(initial: Drill)` with `drill`, `schedule` (recomputed on every change), `selection`, and commands `addPlayer`, `addCone`, `drawSkate(playerId, path, style)`, `addPass(...)`, `addShot(...)`, `addDrop`, `addPickup`, `setTrigger(actionId, trigger)`, `addArriveTogether(...)`, `setSpeed`, `setRink`, `deleteSelected()`, `undo()`, `redo()`. Each command pushes onto `History`.
  - `warnings.ts`: `describeWarning(w: Warning, d: Drill): string` (plain text for the coach, naming players by number).
  - `Timeline.tsx`: read-only; one row per player plus a puck row; bars from `schedule.items`; a warning marker on the affected bar.

- [ ] **Step 1: Failing tests:**
  - Store: each command changes `drill` and `schedule`; `undo` restores both; `deleteSelected` on a player cascades (Task 3); redo works.
  - Timeline: renders one row per player plus the puck row, a bar's left offset and width are proportional to start and duration, and a warning marker appears for a `late` warning. There is no drag handler on bars (read-only).
  - `describeWarning` names the player number and the seconds late.
- [ ] **Step 2:** Run it. Expected: FAIL.
- [ ] **Step 3:** Implement the store first, then components. `RinkEditor` places players and cones from a palette, draws paths by clicking waypoints (double-click to finish), and picks a receive point by clicking a waypoint on the receiver's path. `RulesPanel` edits the selected action's trigger ("starts at beginning", "starts when [event], plus [delay] s") and the arrive-together rules. Player cards offer speed presets. `PlayBar` has play/pause, a scrubber, elapsed and total time, and a rate selector (0.25x, 0.5x, 1x, 2x). `DrillList` offers new, open, delete, import, export. Keyboard: Ctrl/Cmd+Z undo, Delete removes the selection.
- [ ] **Step 4:** Run `npx vitest run`. Expected: PASS. Then run `npm run dev` and, in a desktop browser, build the `russianStretch()` drill by hand, play it, and confirm warnings appear when a receiver is made too slow.
- [ ] **Step 5: Commit** `feat: editor UI`.

---

### Task 13: MP4 and GIF export

**Files:**
- Create: `src/export/plan.ts`, `src/export/mp4.ts`, `src/export/gif.ts`, `src/ui/ExportDialog.tsx`, `tests/export-plan.test.ts`

**Interfaces:**
- Consumes: `frameAt`, `drawFrame`, `canvasTarget`, `Drill`, `Schedule`.
- Produces:
  - `planExport(duration: number, format: 'mp4'|'gif'): { ok: true; fps: number; frames: number; width: number; height: number; bitrate: number } | { ok: false; error: string }` (MP4 1280x720 at 30 fps; GIF 480x270 at 15 fps; duration over 60 s is an error; MP4 bitrate is `min(4_000_000, floor(15.5e6 * 8 / duration))`; zero duration is an error with a clear message)
  - `isMp4ExportSupported(): boolean` (checks `VideoEncoder` and `isConfigSupported`)
  - `exportMp4(d: Drill, s: Schedule, onProgress: (p: number) => void): Promise<Blob>`
  - `exportGif(d: Drill, s: Schedule, onProgress: (p: number) => void): Promise<Blob>`

- [ ] **Step 1: Failing tests** for `planExport` only (encoders need a real browser): 10 s MP4 gives 300 frames and a bitrate of 4 Mbps; 60 s gives a bitrate of at most `floor(15.5e6*8/60)`, so estimated size stays under 16 MB; 61 s gives `ok: false` with a message containing "60"; duration 0 gives `ok: false`; GIF at 8 s gives 120 frames (Review Focus 5).
- [ ] **Step 2:** Run it. Expected: FAIL.
- [ ] **Step 3:** Implement `plan.ts`. Add `mp4-muxer` and `gifenc`. `exportMp4` renders each frame to an `OffscreenCanvas`, encodes with `VideoEncoder` (H.264, `avc1.42001f`), and muxes with `mp4-muxer` into an in-memory MP4. `exportGif` quantizes and writes frames with `gifenc`. After encoding, if the blob is over 16 MB return an error rather than the file. `ExportDialog` shows a clear message when `isMp4ExportSupported()` is false and offers GIF, shows progress, and downloads the result with `drill-name.mp4`.
- [ ] **Step 4:** Run `npx vitest run`. Expected: PASS. Then in desktop Chrome, export the `russianStretch()` drill as MP4 and GIF. Confirm the file opens, shows badges and highlights, and is under 16 MB. Send the MP4 to a phone through WhatsApp and confirm it plays.
- [ ] **Step 5: Commit** `feat: MP4 and GIF export`.

---

## Self-review

**Spec coverage:** Rink full/half (Tasks 4, 9, 12). Players and cones (Tasks 2, 3). Puck throw/receive/carry/drop/pickup/one-touch (Tasks 5, 7). Rules `starts when`, delay and arrive-together (Tasks 5, 6). Conflicts warn only (Tasks 5, 6, 12). Read-only timeline (Task 12). Undo and delete (Tasks 3, 12). Badges, highlight and transparency (Tasks 7, 9). Playback with speed and scrubbing (Task 10). Export MP4/GIF with limits (Task 13). Local storage and file import/export (Task 11). The four book drills (Task 8). Path styles (Task 9). Redo is included as a small extra at no cost.

**Calls made in this plan:** meters/seconds and rink dimensions, `PUCK_SPEED = 20`, speed presets `4/6/8`, `RECEIVE_RADIUS = 1.0`, the explicit receive point for passes, localStorage rather than IndexedDB, Preact, `mp4-muxer` and `gifenc`, the GIF size (480x270 at 15 fps), the 1 MB import cap and the 500-action cap.
