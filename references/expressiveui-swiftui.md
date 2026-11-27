# ExpressiveUI — M3 Expressive 実装ノウハウ（SwiftUI 実証済み部品集）

Use this reference when:
- implementing M3 / M3 Expressive on **iOS / SwiftUI** (where no official Material library exists),
- specifying the expressive **loading indicator, wavy progress, button groups, split button,
  FAB menu, segmented menu, or carousel** on any platform — the token values and behaviour
  rules here are platform-independent,
- a design "looks Material-ish" but feels wrong — check the failure notes (§11) first.

## Source and licence

- **ExpressiveUI** — https://github.com/edenkollcinaku/ExpressiveUI
  Material 3 Expressive components for SwiftUI, extracted from a shipping app
  (Pixel Habit Tracker). Checked 2026-11-27 at commit `e6d1b42`, version line `0.x`,
  requires **iOS 16 / macOS 13**, Swift Package:
  `.package(url: "https://github.com/edenkollcinaku/ExpressiveUI.git", from: "0.1.0")`
- Licence: the Swift implementation is **MIT**. The carousel keyline maths, the material
  shape builder (`RoundedPolygon` / `MaterialShapes` figures) and the cubic-bezier solver
  are ports of Google / AOSP code under **Apache-2.0** — if you copy *code* from those
  parts into a project, carry the repo's `NOTICE` attribution with it. Token values and
  behaviour rules (everything in this file) are published spec numbers, free to use.
- Introduced to this workspace by X post 2126165114843984030 (@protoduct_ai, 2026-11-27).
- **Governing principle of the library, adopt it everywhere:** every measurement is
  traced back to the token it comes from, never eyeballed from a screenshot. If a value
  in a handoff cannot name its token, it is a guess.

## 1. The six springs — spatial vs effects (most important takeaway)

Compose does not hand-tune a spring per component. It names **six**, and every component
asks for one by role. The split that matters:

- **Spatial** springs — for anything that moves or resizes. Loose enough to overshoot slightly.
- **Effects** springs — for anything that only changes colour or opacity. Critically damped
  (no overshoot) and roughly 3× faster. A colour that eases as slowly as a movement reads
  as a *fade*, and fading is what makes a control feel soft when the Android one does not.

| Role | Damping | Stiffness | SwiftUI `response` | Used for |
|---|---|---|---|---|
| `defaultSpatial` | 0.8 | 380 | 0.322 | default movement |
| `fastSpatial` | 0.6 | 800 | 0.222 | press squeeze, switch thumb, FAB menu opening |
| `slowSpatial` | 0.8 | 200 | 0.444 | large / slow movement, loading morph |
| `defaultEffects` | 1.0 | 1600 | 0.157 | default colour change |
| `fastEffects` | 1.0 | 3800 | 0.102 | colour lands in ~0.1s — reads as instant without cutting |
| `slowEffects` | 1.0 | 800 | 0.222 | slow colour change |

(SwiftUI conversion, unit mass: `response = 2π / √stiffness`, `dampingFraction` = damping ratio.)

Rule for any platform: **never run a colour change and a geometry change on the same
spring.** Split them — geometry on spatial, colour on effects.

## 2. Loading indicator (the morphing one)

A shape that spins while morphing through a sequence of others.

- **Indeterminate shape sequence (Material's own order, 7 shapes):**
  soft burst → cookie 9 → pentagon → pill → sunny → cookie 4 → oval.
- **Determinate** uses only two — circle opening into soft burst — so its shape *is* its progress.
- Geometry: container 48×48, active shape 38. Colour: `primary`;
  contained variant = `onPrimaryContainer` shape on a `primaryContainer` circle.
- **Two motions run at once, deliberately out of step:**
  whole shape rotates 4666ms/revolution, linear; next shape every 650ms on a spring
  (damping 0.6, stiffness 200), and each morph also brings a quarter turn of its own on
  the same spring. Because the spring settles before the next morph starts, the indicator
  keeps *arriving somewhere and setting off again* — that is what stops it reading as a
  mechanical spinner. Do not sync the two motions.
- Determinate: no clock — shape position in the sequence is driven by progress, and the
  whole thing turns counter-clockwise through half a revolution across the full run.
- **Reduce Motion:** spinning stops, morphing continues — something still has to say the
  app is working. Never freeze it entirely, never keep spinning.
- Morph technique (why naive morphs look wrong): shapes are star-shaped about their
  centre, stored as a radius per angle, so morphing is interpolation of radii at matching
  angles. Mid-morph radii are *blurred* by an amount peaking mid-morph and vanishing at
  both ends — unmatched lobes (9 fading into 10) otherwise grow small extra bumps belonging
  to neither end. Corners share edges proportionally (do **not** cap every corner at half
  an edge — it starves a big corner next to a small one); straight edges are walked, not
  interpolated in polar coordinates (that bows a pentagon into a blob).

## 3. Progress indicators — flat and wavy

### Three parts, not two (linear)

A Material progress bar is **filled part + 4pt gap + track**, plus a **stop dot** at the
far end that stays where it is (`TrackActiveSpace` = 4, `StopSize` = 4). The gap and the
dot are what make it read as Material. The dot is drawn at every progress, including zero.
Colours: filled + dot = `primary`, track = `secondaryContainer`.

### The wave (Expressive)

- Only the **filled part** waves; the track stays straight.
- Linear: amplitude 3, wavelength 40 (indeterminate wavelength 20), container height 10,
  thickness 4. Circular: amplitude 1.6, wavelength 15, container 48 (flat 40), thickness 4.
- The wave **travels at one wavelength per second**, and the phase belongs to the track,
  not the bar — an indeterminate bar sliding along moves *through* the wave instead of
  carrying its own. A wave that wobbles in place is wrong.
- **The wave flattens at both ends:** amplitude is 0 below 10% and above 95%, full between,
  easing over 500ms — the indicator settles as it arrives rather than stopping mid-crest.
- Circular only: the wavelength is **snapped so a whole number of waves fits the ring**,
  otherwise a full ring shows a seam where the ends meet at different heights. The 4pt gap
  on a ring is an *angle* — convert it through the radius, do not use it as a length.

### Indeterminate motion

- Linear: four keyframes inside a 1750ms loop — two bars, each with a head and a tail
  (bar 1: head delay 0 / 1000ms, tail 250 / 1000ms; bar 2: head 650 / 850ms, tail 900 / 850ms),
  all on `EasingEmphasizedAccelerate` = cubic-bezier(0.3, 0, 0.8, 0.15). That curve is most
  of what makes it read as Material. Track is drawn in whatever is left between the bars,
  keeping the 4pt gap on both sides of each.
- Circular: a 6-second cycle of **three motions laid over each other** — steady rotation
  (3 full turns, 1080°), stepped rotation (four quarter-turns, each 300ms then holding
  until the next 1500ms mark — the lunging character comes from these steps), and a sweep
  growing from 10% of the ring to 87% by 3000ms then closing back on `EasingStandard`.
  A cycle ends 1440° round, which is why it wraps without a jump.
- **Circular indeterminate has NO track** (`circularIndeterminateTrackColor` is transparent;
  determinate track is `secondaryContainer`). A lone arc travelling an empty ring is what
  tells the user the app does not know how long this will take.

## 4. Switch — not a tinted Toggle

| Token | Value |
|---|---|
| Track | 52 × 32 |
| Handle | 16 off / 24 on / **28 while pressed** |
| Track outline | 2pt, **off only** — on, the track is solid fill and the outline disappears |
| Selected icon | 16 (check, on only) |

- Colours — off: track `surfaceContainerHighest`, border `outline`, thumb `outline`.
  On: track `primary`, thumb `onPrimary`, check glyph `onPrimaryContainer`
  (the check is the accent again, not a hole punched in the thumb).
- Pressing grows the handle for as long as the finger is down, around its centre —
  express travel as centres so the pressed thumb grows rather than shifts.
- Motion split (§1): thumb travels on `fastSpatial`, track colour lands on `fastEffects`.
  Running both at the movement rate is what makes a switch read as fading rather than flipping.
- Disabled: whole control at 38% (Material's disabled alpha) and non-responsive.
- SwiftUI note: implement as a `ToggleStyle` so call sites stay ordinary `Toggle`s and
  keep their accessibility for free — but the style must render the label itself, or it
  silently discards whatever the call site put in the `Toggle`'s body.

## 5. Button — press feedback is a shape morph

- Five emphases: filled (`primary`/`onPrimary`) > elevated (`surfaceContainerLow`,
  elevation level 1, content `primary`) > tonal (`secondaryContainer`/`onSecondaryContainer`)
  > outlined (no container, 1pt `outlineVariant`, content `onSurfaceVariant`)
  > text (no container, `onSurfaceVariant`). Filled is the one action that completes a flow.
- Five sizes — a size is not just a height; padding, label and pressed corner move with it:

| Size | Height | Padding | Label | Pressed corner |
|---|---|---|---|---|
| extraSmall | 32 | 16 | 14 | 8 |
| small (default) | 40 | 16 | 14 | 8 |
| medium | 56 | 24 | 16 | 12 |
| large | 96 | 48 | 24 | 16 |
| extraLarge | 136 | 64 | 32 | 16 |

- **The press:** the container is a pill at rest and morphs to a squarer corner while held
  (round shape → pressed shape) on `fastSpatial`. That morph is the *whole* press feedback:
  **no dimming, no scaling.** (Implementation consequence: the container must be a rounded
  rectangle, not a `Capsule`, so the two corners can animate into each other.)
- Disabled is a *pair*, not a whole-control opacity: container = `onSurface` at 10%,
  label = `onSurfaceVariant` at 38%. An outlined button keeps its border at 38%.

## 6. Button groups — the squeeze

- Two kinds: **standard** (buttons 12pt apart) and **connected** (segments 2pt apart,
  corners squared off against each other to read as one control).
- **The squeeze:** pressing a button expands it by **15% of its width**
  (`ExpandedRatio` = 0.15) and its neighbours give up that space between them, so the
  group's own width never changes. A neighbour is never asked to give up more than 24pt
  (past that its label clips rather than tightens); when the cap bites, the pressed button
  grows by less. `expandedRatio: 0` turns the squeeze off.
- Connected corner rules *are* the component: outer corners full, inner corners 8
  (4 while pressed); a **selected** segment goes full on both sides, because it is the one
  being read rather than part of a run.
- Colours: selected / plain action = `primary`/`onPrimary`; unselected =
  `surfaceContainer`/`onSurfaceVariant`. Toggleable items: filled when checked,
  `surfaceContainer` when not.
- Overflow: buttons that do not fit collapse into a menu behind a trailing indicator —
  nothing is dropped and nothing wraps. A group whose buttons all carry weights never
  overflows; weights share whatever width is left over.
- Motion: press squeeze on `fastSpatial`, every colour change on `fastEffects` (§1).

## 7. Segmented buttons vs connected group — do not confuse them

| | Segmented buttons | Connected button group |
|---|---|---|
| Container | outlined, transparent | filled, `surfaceContainer` |
| Between segments | one shared border line | 2pt gap |
| Selection reads as | checkmark + `secondaryContainer` fill | segment becomes a pill, `primary` fill |
| Press | no size change | segment widens, neighbours give way |

The connected group is what Expressive reaches for now — prefer it in new work; use
segmented buttons when the outlined, checkmarked look is wanted. Material asks for
**2–5 segments**; past five, use chips instead.

- Shared-border technique: segments overlap by exactly the border width (1pt) so two
  neighbours' borders land on one line rather than stacking into a 2pt seam; the selected
  segment draws above its neighbours so its border survives on the shared line.
- Checkmark: a segment with no icon grows the checkmark out of the label's leading edge
  and the label slides over to make room (scale-in on the spatial spring, fade on the
  effects spring). A segment that *does* carry an icon crossfades icon ↔ checkmark instead.
- Geometry: height 40, outline 1, icon 18, horizontal content padding 12, icon–label 8;
  corners full at the row's ends, square between. Disabled: label `onSurface` 38%,
  border `onSurface` 12%, per segment or whole row.

## 8. Split button — the halves pull apart

- One button runs the primary action; a narrower one beside it opens what is contextually
  related. The halves sit 2pt apart at every size.
- Corner behaviour is the **opposite** of the standalone button (§5): outer corners full,
  the facing inner corners are small (4 / 4 / 4 / 8 / 12 by size) and **open up while
  pressed** (12 / 12 / 12 / 20 / 20) — the two halves pull away from each other so you can
  see which one you are holding.
- The trailing half is a *toggle*, not a menu: its chevron **turns over** rather than
  swapping for a second glyph, and the trailing button becomes a **circle** for as long as
  the thing it opened is open. Presentation (popover / sheet / panel) stays the caller's.
- Both halves are filled buttons (`primary`/`onPrimary`); disabled uses the §5 pair.

## 9. FAB menu

- Closed FAB: 56×56, 16pt corners. Open, it becomes a **circle** and the icon **turns 45°** —
  a plus becomes a close without needing a second symbol.
- Menu items: 56pt capsules (as tall as the FAB, as wide as their label), icon 24 leading,
  container `primaryContainer`, content `onPrimaryContainer` — a step below the FAB
  (`primary`/`onPrimary`) on purpose: the FAB is the accent, and a column of equally loud
  buttons above it would have nothing to point back to.
- Items come out **one at a time, 65ms apart, starting nearest the FAB**, and go back in
  the opposite order — the column reads as unfolding from the button rather than arriving
  as a block. Everything moves on `fastSpatial`.
- Choosing an item **closes the menu first, then runs its action** — a sheet or navigation
  push never animates on top of a menu that is still open.
- Closed, items are scaled to nothing against their trailing edge, hidden from hit testing
  **and** from accessibility. The FAB's accessibility label changes with state
  ("Open menu" / "Close menu", both parameterisable).

## 10. Segmented dropdown menu & carousel & elevation

### Segmented menu — the corner rule, twice

- A run keeps its outer radius where it faces the outside and a smaller one where it faces
  a sibling — applied **twice**: groups ends 16 / seam 8; items ends 12 / seam 4.
  Everything sits 2pt apart (`SegmentedGap`, same between items and between groups).
- A **selected** item leaves the run: round on all four corners wherever it sits, filled
  `tertiaryContainer` / `onTertiaryContainer`. Selection uses the *third* accent, not
  `primary` — a menu is usually opened by something already wearing the accent, and the
  two must not compete.
- **The menu paints no background surface.** Each group is its own surface; the 2pt gaps
  are real gaps and whatever is behind shows through — that is what makes it *segmented*.
- Groups are flat by default even though the token names elevation level 2: two cards
  2pt apart each cast a shadow into the gap and the seam muddies. Reach for elevation
  only when the menu must hold its own against busy content behind it.
- Opening: scales up from 0.8 and fades in. Geometry: item row min height 48, icons 20,
  item padding 12, group padding 4, width 112 min / 280 max. Group container
  `surfaceContainerLow`; labels `onSurface`; icons / supporting / trailing text
  `onSurfaceVariant`; divider `outlineVariant`.

### Carousel — keylines, not item sizes

- Items do not own their sizes: the **viewport owns a list of keylines** (positions with
  sizes attached) and an item takes the size of wherever it has scrolled to.
- The arrangement is *searched*: small wants to be ⅓ of large, medium halfway between;
  every plausible count of large/medium/small is fitted by bending small items first,
  then medium, then — reluctantly — large; the candidate that bent the large item least
  wins. That is why the width parameter is *preferred*, not exact.
- **Items are masked, not scaled:** a small item is a large item with most of it clipped
  away, so text is cropped, never shrunk into unreadability. Sizes are continuous while
  dragging — the row breathes rather than stepping.
- Ends: per-keyline steps walk the focal range to the container's edge, so the first and
  last items can each be large at their end (scroll to the end and the arrangement mirrors
  itself). Tokens: small item 40–56, anchor 10, item corner defaults to 28 (extra-large shape).

### Elevation — two shadows, not one

Android lights an elevated surface with two sources; reproduce both or it reads as fog:

| Level | dp | Use | Key shadow (30% black) | Ambient shadow (15% black) |
|---|---|---|---|---|
| 0 | 0 | flat | none | none |
| 1 | 1 | elevated button | radius 1, y 1 | radius 1.5, y 1 |
| 2 | 3 | menu group | radius 1, y 1 | radius 3, y 2 |
| 3 | 6 | FAB | radius 1.5, y 1 | radius 4, y 4 |
| 4 | 8 | — | radius 1.5, y 2 | radius 5, y 6 |
| 5 | 12 | — | radius 2, y 4 | radius 6, y 8 |

The tight key shadow is what makes an edge look *just* off the surface rather than
floating — it matters most where two elevated things sit a couple of points apart and
their shadows would otherwise pool in the gap.

## 11. Failure notes — where imitations go wrong

- Tinting the platform's stock control (a coloured iOS `Toggle`) and calling it Material:
  the geometry, colour roles and motion are all different. Rebuild the control, keep the
  platform's semantics/accessibility shell around it (style protocols, not custom gestures).
- One spring / one easing for everything → colour fades and mushy controls. Split §1.
- Progress bar drawn as one bar over another → missing gap + stop dot = not Material.
- Wave drawn on the track, or waving in place, or stopping mid-crest at 100%.
- Circular indeterminate with a visible track ring.
- Button press feedback as dimming or scaling instead of the corner morph.
- Disabled as a blanket opacity over the whole control (except the switch's 38% rule).
- Menu selection in `primary` competing with the control that opened the menu.
- Single blurred shadow under elevated surfaces (fog) instead of key + ambient.
- Carousel items scaled down (unreadable text) instead of masked/clipped.
- Values eyeballed from screenshots instead of named tokens — the root failure behind
  most of the above.

## 12. SwiftUI integration pattern

- Inject the palette **once near the root** by mapping the app's colours onto the role
  set (`ExpressiveColors`: primary, onPrimary, primaryContainer, onPrimaryContainer,
  secondaryContainer, onSecondaryContainer, tertiaryContainer, onTertiaryContainer,
  surfaceContainer, surfaceContainerLow, surfaceContainerHighest, onSurface,
  onSurfaceVariant, outline, outlineVariant). Components read it from the environment —
  styling is one call, never per-component. Roles grow as components land; additions are
  minor versions and existing initialiser calls keep compiling.
- Call sites stay ordinary SwiftUI: `.toggleStyle(.expressive)`,
  `.buttonStyle(.expressive)` / `.buttonStyle(.expressive(.tonal, size: .medium))`.
  Prefer this style-protocol shape for custom M3 controls on iOS — it preserves native
  accessibility instead of re-implementing it.
- Status caveat: the library is `0.x` and its colour surface is still settling; component
  APIs are stable in shape. Verify the current release before pinning it in production,
  and re-check token values against m3.material.io if they ever disagree — the official
  spec wins over both the library and this file.
