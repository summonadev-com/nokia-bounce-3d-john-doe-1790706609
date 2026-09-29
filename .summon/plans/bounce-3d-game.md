---
status: pending
title: Bounce 3D — single-level remake of Nokia Bounce
---

## Technology decision

Use **@react-three/fiber** (React renderer for Three.js) + **@react-three/drei** (helpers: camera, environment, text) + **@react-three/rapier** (WASM Rapier physics bindings).

Rationale: Bounce is a physics game — a rubber ball that rolls, bounces off walls, and restitutes off surfaces. Hand-rolled kinematics would require re-implementing sphere-vs-mesh collision, restitution and friction. Rapier gives deterministic, fast WASM rigid-body simulation with built-in sensors (perfect for rings and hazard triggers) and costs far less code than a custom solver. drei supplies `<OrbitControls>`-free camera work, `<Environment>`, `<ContactShadows>` and `<Text>` so the "modern colorful with lighting and shadows" look is reachable quickly. All three are React-idiomatic and work unchanged inside a Vite + TS app.

---

## Steps

1. **Install dependencies.** Add `three`, `@types/three`, `@react-three/fiber`, `@react-three/drei`, `@react-three/rapier` with npm. Outcome: `package.json` lists the four runtime packages and the types package; `npm run dev` still boots with no errors.

2. **Add shared types.** Create `src/types/bounce.ts` describing the level data model: a `Vec3` tuple alias; `PlatformDef` (position, size, optional colour, optional `kind` for normal/bouncy); `WallDef`; `RingDef` (position, rotation axis, radius); `SpikeDef` (position, size); `GoalDef` (position, radius); and a `LevelDef` aggregating spawn point, gravity, arrays of each of the above, plus `lives` and `title`. Outcome: one import point for every level-shaped value used by the scene and HUD.

3. **Author the level data.** Create `src/lib/levels/level1.ts` exporting a single `LevelDef` for the complete first level: a start platform, a rolling straight with two low walls, a gap the player must jump, a raised mid-section reached via a bouncy pad, three rings spaced along the path (one requiring a jump), four spike clusters on the ground stretch and the raised section, side walls that keep the ball in bounds, and a goal pad at the far end. Include a killplane Y value in the level for out-of-bounds detection. Outcome: the entire level geometry is data, editable without touching components.

4. **Game constants and helpers.** Create `src/lib/bounce/constants.ts` (ball radius, mass, restitution, friction, roll torque/impulse strength, max angular speed, jump impulse, coyote-time window, camera offset and lerp factor, respawn delay) and `src/lib/bounce/math.ts` (lerp for numbers and vectors, clamp). Outcome: all tuning values live in one file so play-feel can be adjusted in one place.

5. **Game state store.** Create `src/hooks/useGameState.ts` — a React hook (backed by `useState`/`useReducer`, or a tiny module-level store with `useSyncExternalStore` so the physics loop can write without re-rendering the canvas tree) holding: `status` (`ready` | `playing` | `dead` | `won`), `lives`, `ringsCollected`, `ringsTotal`, `elapsedMs`, and actions `start`, `collectRing`, `loseLife`, `respawn`, `win`, `reset`. Losing the last life sets status `dead` with no lives left (game over). Outcome: HUD and scene read the same state; no prop drilling.

6. **Keyboard input hook.** Create `src/hooks/useKeyboardControls.ts` binding `keydown`/`keyup` on `window`, mapping ArrowUp/W, ArrowDown/S, ArrowLeft/A, ArrowRight/D and Space into a mutable ref of booleans (ref, not state, so the physics frame reads it without re-rendering). Prevent page scroll on Space and arrows. Clear all keys on window blur. Outcome: `controls.current.forward` etc. readable inside `useFrame`.

7. **Ball component.** Create `src/components/bounce/Ball.tsx`: a Rapier `RigidBody` (dynamic, ball collider, restitution and friction from constants, CCD enabled to stop tunnelling at speed) wrapping a mesh with a colourful glossy material and `castShadow`. In `useFrame`, read the input ref and apply torque/impulse relative to the camera's forward and right vectors so "up" always means away from the camera; clamp linear velocity to a max; apply jump impulse on Space only when grounded. Ground detection: short downward ray cast from ball centre, plus a coyote-time window. Expose the rigid body via a ref forwarded to the parent. Outcome: a ball that rolls responsively, bounces on landing, and jumps once per contact.

8. **Static level geometry components.** Create `src/components/bounce/Platform.tsx`, `src/components/bounce/Wall.tsx` and `src/components/bounce/BouncyPad.tsx`, each a fixed Rapier `RigidBody` with a cuboid collider and a `receiveShadow` mesh; the bouncy pad uses high restitution and a distinct emissive colour. Outcome: level solids render and collide from data.

9. **Ring component.** Create `src/components/bounce/Ring.tsx`: a torus mesh plus a Rapier **sensor** collider (thin cylinder through the ring's opening) that fires `onIntersectionEnter` when the ball passes through. On trigger, call `collectRing`, play a scale/emissive pop, and mark itself collected so it cannot re-trigger. Slow idle rotation via `useFrame`. Outcome: passing through a ring increments the counter exactly once.

10. **Hazard component.** Create `src/components/bounce/Spikes.tsx`: instanced cone meshes for the visual plus a single sensor collider box covering the cluster. On intersection with the ball, call `loseLife` and trigger respawn. Outcome: touching spikes costs a life.

11. **Goal component.** Create `src/components/bounce/Goal.tsx`: a glowing ring/pad with a sensor collider that calls `win` when the ball enters **and** `ringsCollected === ringsTotal`; otherwise it shows a "collect all rings" hint state. Outcome: the level can be completed only after all rings are taken.

12. **Respawn and kill plane.** Create `src/components/bounce/GameLogic.tsx` (renders nothing): each frame checks whether the ball's Y is below the level's killplane and, if so, calls `loseLife`. Owns the respawn routine — zero linear and angular velocity, teleport the rigid body to the level spawn (or last reached checkpoint platform if added later), short invulnerability window so spikes cannot double-fire. Also advances `elapsedMs` while status is `playing`. Outcome: falling off or dying always returns the ball cleanly to the start with one fewer life.

13. **Follow camera.** Create `src/components/bounce/FollowCamera.tsx`: a `PerspectiveCamera` made default, positioned each frame by lerping toward the ball position plus the configured offset, always looking slightly ahead of the ball. Damp the lerp so it feels smooth and does not jitter with the physics step. Outcome: camera trails the ball without snapping or overshooting.

14. **Lighting and environment.** Create `src/components/bounce/SceneLighting.tsx`: hemisphere light for ambient fill, one directional key light with `castShadow` and a shadow camera framed to the level bounds, drei `<Environment preset>` for reflections on the glossy ball, soft fog matched to the background colour. Enable `shadows` on the `<Canvas>`. Outcome: colourful, well-lit scene with grounded contact shadows.

15. **Level renderer.** Create `src/components/bounce/Level.tsx` taking a `LevelDef` and mapping each array to the matching component from steps 8–11. Outcome: swapping the level import is the only change needed to add level 2 later.

16. **HUD overlay.** Create `src/components/bounce/Hud.tsx` — an absolutely positioned Tailwind layer above the canvas showing lives (as icons), rings collected / total, elapsed time, and a small controls legend. Uses `pointer-events-none` except for buttons. Outcome: readable HUD that never blocks input.

17. **Overlays.** Create `src/components/bounce/Overlays.tsx` with three states: a start card ("Bounce 3D" + controls + Play), a game-over card (Retry), and a win card (time, rings, Play again). Each is a Tailwind-styled centred panel with a backdrop blur, shown based on `status`. Buttons call `start` / `reset`. Outcome: clear entry, failure and success flows.

18. **Game shell.** Create `src/components/bounce/BounceGame.tsx` composing everything: a full-height relative container, the `<Canvas shadows>` with `<Physics>` (gravity from the level, `debug` toggled by a dev flag), `<Suspense>` fallback, `FollowCamera`, `SceneLighting`, `Level`, `Ball`, `GameLogic`, plus `Hud` and `Overlays` as siblings outside the canvas. Pause physics (`Physics paused`) unless status is `playing`. Outcome: a single importable component that is the whole game.

19. **Routes.** Create `src/routes/bounce.tsx` rendering `BounceGame` inside a full-viewport layout, and ensure `src/routes/index.tsx` links to it (or renders the game directly if the app has no other content). Confirm `src/routes/__root.tsx` does not impose padding or scroll that breaks a full-bleed canvas. Outcome: the game is reachable at `/bounce` and fills the viewport.

20. **Performance pass.** Reuse a small set of shared materials/geometries across platforms and spikes, use instancing for spike cones, cap `dpr` on the Canvas to `[1, 2]`, keep the shadow map at 1024–2048 with a tight shadow camera frustum, avoid `setState` inside `useFrame` (write to refs, publish to the store only on discrete events), and memoise the level-derived arrays. Outcome: steady 60fps on a mid-range laptop.

21. **Polish.** Add subtle ball squash on landing, ring-collect particle or flash, colour-graded background gradient, and a brief camera shake on death. Keep every effect cheap and optional behind constants. Outcome: the game feels modern rather than a physics demo.

---

## Acceptance criteria by phase

- **Phase A (steps 1–4):** dependencies installed, level data and constants typecheck, no runtime yet.
- **Phase B (steps 5–8, 13, 14, 18, 19):** `/bounce` renders a lit 3D scene; the ball rolls with WASD/arrows relative to the camera, jumps on Space, bounces off platforms, and the camera follows smoothly.
- **Phase C (steps 9–12):** rings increment the counter once each, spikes and the killplane cost a life and respawn the ball at spawn, the goal completes the level only with all rings collected.
- **Phase D (steps 16, 17):** HUD shows live lives/rings/time; start, game-over and win overlays appear and their buttons restart correctly.
- **Phase E (steps 20, 21):** no frame-rate drops while rolling through the full level; no `setState`-per-frame warnings; visual polish present.
