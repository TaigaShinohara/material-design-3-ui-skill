# Motion

Use this reference when specifying transitions, feedback, shape morphing, or expressive animation.

Values in the token tables below are the official Material 3 values, verified against
`https://m3.material.io/styles/motion/easing-and-duration/tokens-specs` (checked 2026-11-25).
If a value here ever disagrees with that page, the official page wins — re-verify before
asserting exact numbers in a handoff.

## Purpose

Motion must communicate.

Use it to: - connect origin and destination, - explain hierarchy changes, - acknowledge input, - reveal or hide content, - preserve continuity during navigation, - preserve continuity during adaptive layout changes.

## Rules

- Prefer Material motion schemes/tokens when available.
- Keep repeated utility interactions quick and unobtrusive.
- Reserve stronger spring or shape motion for moments that benefit from it.
- Do not animate everything.
- Do not delay task completion for spectacle.
- Do not make motion the only explanation of state.
- Respect reduced-motion preferences.
- Keep spatial transitions consistent with the interface’s navigation model.

## Easing tokens (official values)

Two sets. The **emphasized** set is the default for most transitions — it captures the
expressive style of M3. The **standard** set is for simple, small, or utility-focused
transitions.

| Token | Value | Use for |
|---|---|---|
| `md.sys.motion.easing.emphasized` | Two-segment path, **not** a single cubic-bezier (see below) | Elements that begin and end on screen |
| `md.sys.motion.easing.emphasized.decelerate` | `cubic-bezier(0.05, 0.7, 0.1, 1.0)` | Elements entering the screen |
| `md.sys.motion.easing.emphasized.accelerate` | `cubic-bezier(0.3, 0.0, 0.8, 0.15)` | Elements leaving the screen |
| `md.sys.motion.easing.standard` | `cubic-bezier(0.2, 0.0, 0.0, 1.0)` | Utility transitions, begin and end on screen |
| `md.sys.motion.easing.standard.decelerate` | `cubic-bezier(0.0, 0.0, 0.0, 1.0)` | Utility elements entering |
| `md.sys.motion.easing.standard.accelerate` | `cubic-bezier(0.3, 0.0, 1.0, 1.0)` | Utility elements leaving |

### The emphasized curve is a path, not a bezier

Approximating `emphasized` with a single cubic-bezier is a common imitation error.
The official definition is a two-segment path:

- Android: `pathInterpolator(M 0,0 C 0.05, 0, 0.133333, 0.06, 0.166666, 0.4 C 0.208333, 0.82, 0.25, 1, 1, 1)`
- Flutter: `Curves.easeInOutCubicEmphasized`
- CSS / iOS: no direct equivalent — the official guidance is to use **standard** as the fallback.
- In Jetpack Compose, take the curve from the material3 motion scheme / tokens instead of hand-writing a bezier.

### Legacy easing

`FastOutSlowIn` (`cubic-bezier(0.4, 0.0, 0.2, 1.0)`) is Material 2 easing. It still
appears in shipped code, but do not choose it for new M3 work — per the core rules,
do not silently mix Material 2 rules into Material 3.

## Duration tokens (official values)

| Token | Value | Typical use |
|---|---|---|
| `md.sys.motion.duration.short1` | 50ms | Small utility transitions |
| `md.sys.motion.duration.short2` | 100ms | Small utility transitions |
| `md.sys.motion.duration.short3` | 150ms | Small utility transitions |
| `md.sys.motion.duration.short4` | 200ms | Selection controls (with standard easing); exit transitions |
| `md.sys.motion.duration.medium1` | 250ms | Medium-area transitions |
| `md.sys.motion.duration.medium2` | 300ms | Medium-area transitions |
| `md.sys.motion.duration.medium3` | 350ms | Medium-area transitions |
| `md.sys.motion.duration.medium4` | 400ms | FAB expanding into a sheet (with emphasized easing) |
| `md.sys.motion.duration.long1` | 450ms | Large transitions, usually with emphasized easing |
| `md.sys.motion.duration.long2` | 500ms | Card expanding to full screen (with emphasized easing) |
| `md.sys.motion.duration.long3` | 550ms | Large / complex transitions |
| `md.sys.motion.duration.long4` | 600ms | Large / complex transitions |
| `md.sys.motion.duration.extra-long1` | 700ms | Rare; ambient transitions without user input |
| `md.sys.motion.duration.extra-long2` | 800ms | Rare; ambient transitions |
| `md.sys.motion.duration.extra-long3` | 900ms | Rare; ambient transitions |
| `md.sys.motion.duration.extra-long4` | 1000ms | Ambient carousel auto-advance (with emphasized easing) |

## Standard pairings

| Situation | Easing | Duration |
|---|---|---|
| Element begins and ends on screen (e.g. container transform, card → full screen) | emphasized | long2 (500ms) |
| Element enters the screen | emphasized decelerate | medium4 (400ms) |
| Element exits the screen | emphasized accelerate | short4 (200ms) |
| Utility transition, begins and ends on screen | standard | medium2 (300ms) |
| Utility element enters | standard decelerate | medium1 (250ms) |
| Utility element exits | standard accelerate | short4 (200ms) |
| Selection controls (switch, checkbox, radio) | standard | short4 (200ms) |

## Transition patterns (Material motion system)

Four canonical patterns. Choose by the relationship between the outgoing and incoming UI,
not by looks.

### Container transform

For transitions between UI elements that include a container: a list item expanding into a
detail page, a FAB morphing into a toolbar or sheet, a chip expanding into a card. One
element visibly transforms into the other, preserving a shared container so the connection
is obvious. Pairing: emphasized easing, long durations (long2 500ms is the reference case).

### Shared axis

For UI elements with a spatial or navigational relationship: steps in a flow, moving
between peers in a hierarchy. Both elements move together along a shared axis —
**X** (lateral steps), **Y** (vertical hierarchy), or **Z** (depth: e.g. a search icon
expanding into a search view). Direction matters: forward and backward travel must read
as opposite directions on the same axis.

### Fade through

For switching between UI elements that do **not** have a strong relationship: top-level
destinations, tabs, switching mailboxes. The outgoing element fades out first, then the
incoming element fades in while scaling from **92% to 100%**. The scale applies only to
the incoming element, to emphasise new content over old.

### Fade

For elements that enter or exit **within the bounds of the current screen**: dialogs,
menus, small overlays. Entering elements fade in while scaling from **80% to 100%**;
exiting elements simply fade out.

## M3 Expressive motion: springs

M3 Expressive adds **spring physics** for interactive component state changes — presses,
drags, expand/collapse, FAB morphs, chip selection — where the motion should feel physical
and interruptible. Two families:

- **Spatial springs** move position/size/shape.
- **Effects springs** animate non-spatial properties (e.g. colour, opacity).

And two schemes:

- **Standard motion scheme** — calmer, utilitarian springs.
- **Expressive motion scheme** — bouncier springs for deliberate expressive moments.

Division of labour:

- **Springs** → component state changes and direct manipulation.
- **Easing + duration tokens (above)** → screen transitions (enter/exit, shared axis,
  container transform), where the end time must be predictable.

In Jetpack Compose, springs come from `MotionScheme` in `androidx.compose.material3`
(e.g. `MotionScheme.expressive()` / `MotionScheme.standard()`, `defaultSpatialSpec()`,
`defaultEffectsSpec()`, plus fast variants). Do not hardcode spring damping/stiffness
numbers from memory: take them from the current library's motion scheme, and check the
current release notes for API stability before using Expressive APIs in production —
not every Expressive API is stable.

### The six named springs (Expressive scheme, proven values)

Where no library motion scheme exists (SwiftUI, web, custom), use the six named springs
distilled from ExpressiveUI (`references/expressiveui-swiftui.md` §1, sourced from
`ExpressiveMotionTokens`). Components ask for a spring **by role**, never tune their own:

| Role | Damping | Stiffness | SwiftUI `response` |
|---|---|---|---|
| `defaultSpatial` | 0.8 | 380 | 0.322 |
| `fastSpatial` | 0.6 | 800 | 0.222 |
| `slowSpatial` | 0.8 | 200 | 0.444 |
| `defaultEffects` | 1.0 | 1600 | 0.157 |
| `fastEffects` | 1.0 | 3800 | 0.102 |
| `slowEffects` | 1.0 | 800 | 0.222 |

The operational rule: **spatial = slight overshoot allowed; effects = critically damped
and ~3× faster.** Running a colour change at the movement spring's rate makes the control
read as fading — the single most common reason an M3 imitation feels soft. Conversely,
never let a colour/opacity change overshoot.

Expressive motion may add personality through spring behavior, shape changes, or stronger transitions, but it should still explain: - what changed, - where content came from, - what the user’s action affected.

If removing the animation makes the interaction incomprehensible, the static states are under-designed.

## Reduced motion

- Respect the platform reduced-motion setting (Android: animator duration scale /
  accessibility settings; Web: `prefers-reduced-motion`).
- Under reduced motion, replace spatial movement with short fades, and preserve the
  state information the motion carried — motion must never be the only explanation
  of state (see Rules).

## Review questions

- What information does this animation communicate?
- Is the duration proportional to frequency and importance?
- Does reduced motion preserve meaning?
- Does motion preserve spatial continuity?
- Are the easing and duration values taken from the tokens above (or the platform
  motion scheme), rather than invented per screen?
- For screen transitions: is the pattern (container transform / shared axis /
  fade through / fade) chosen by the relationship between the two UIs?
- For component state changes in an Expressive design: springs from the motion
  scheme, not a hand-tuned tween?
