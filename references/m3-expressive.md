# Material 3 Expressive

Use this reference only when M3 Expressive is requested or clearly appropriate.

M3 Expressive extends Material 3. It is not a separate excuse to ignore M3 semantics.

## Expressive levers

- color
- shape
- size
- motion
- containment
- typography

## Rules

- Use expression to improve hierarchy, usability, recognition, or emotional clarity.
- Make important actions easier to find, not merely more decorative.
- Use size contrast to establish priority.
- Use shape contrast to distinguish meaningful roles or states.
- Use vibrant color selectively.
- Use containment to clarify groups and isolate key actions.
- Use motion to explain state and spatial relationships.
- Use emphasized typography selectively.
- Prefer a few strong expressive moments over uniform intensity.
- Preserve familiarity: buttons must still read as buttons, navigation as navigation, and selection as selection.
- Keep routine forms, settings, reading, and dense productivity surfaces calmer unless stronger expression measurably improves scanning.

## Candidate expressive patterns

Depending on current platform support: - button groups, - split buttons, - floating toolbars, - expressive list items, - richer shape families, - emphasized typography, - updated loading/progress experiences, - shape-morphing interactions.

Verify implementation-library support before naming an exact API.

## Proven expressive specifics (ExpressiveUI)

For concrete, shipping-proven values and behaviour for the patterns above, read
`references/expressiveui-swiftui.md`. The load-bearing ideas, in short:

- **Morphing loading indicator** — 7 shapes in Material's order, rotation and morph
  deliberately out of step; determinate form's shape *is* its progress.
- **Wavy progress** — only the filled part waves, the wave travels, and it flattens
  below 10% / above 95% so the indicator settles as it arrives.
- **Button-group squeeze** — pressing widens a segment by 15% and its neighbours give
  way; the group's width never changes.
- **Press = shape morph, not dimming** — a button's corner squares while held; a split
  button does the opposite and its halves pull apart.
- **Motion split** — spatial springs for movement, effects springs for colour (§1 of
  that reference). One spring for both is the most common expressive failure.
- **Restraint still applies** — these specifics strengthen one or two moments per
  screen; they are not a licence to make every control morph, wave, or squeeze.

## Restraint test

Ask: 1. What deserves the user’s attention first? 2. Which one or two expressive levers best reinforce that priority? 3. Can routine content remain calmer? 4. Does the UI remain understandable if expressive decoration is reduced? 5. Does accessibility remain intact?

If everything is expressive, nothing is emphasized.
