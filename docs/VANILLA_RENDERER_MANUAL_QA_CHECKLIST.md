# Vanilla Renderer Manual Visual QA Checklist

# Vanilla Renderer Manual Visual QA Checklist

> **Status:** GeckoLib is a required dependency for WorldForge (see
> `docs/GECKOLIB_AUDIT_REPORT.md`'s final decision note) — `RendererCompatInit` currently always
> takes the GeckoLib branch in practice. This checklist stays relevant for whenever the vanilla
> path is actually exercised (dependency-stance change, or a future Core+Compat split).

Automated checks (bone/cube/mirror counts, pivot & rotation math) passed — see
`docs/GECKOLIB_AUDIT_REPORT.md`. That covers *geometry correctness*, not *how it looks moving*.
Nothing below can be verified from source; it requires running the client and looking.

For every entity/state combo, compare the **vanilla renderer** against the **GeckoLib renderer**
side by side (spawn one of each, or toggle the config flag) from these camera angles:
front, back, left, right, 45°, and one close-up on the head/face.

## CatBot — CAT form
- [ ] Idle (check tail sway, ear twitch, whisker rest position, eyelid blink timing)
- [ ] Walk (front/back leg pairing — front-left+back-right should swing together)
- [ ] Run
- [ ] Sitting
- [ ] Sleeping (`resting` flag — body should flatten/lie down, not just stop)
- [ ] Attack / hurt / death (if CatBot has combat states)
- [ ] Blink close-up — confirm the eyelid-yRot approximation reads as a blink and not a visible glitch

## CatBot — HUMANOID form
- [ ] Idle
- [ ] Walk / run (arm+leg swing symmetry)
- [ ] Sitting
- [ ] Sleeping
- [ ] Attack / hurt / death
- [ ] Cape (3-segment) — check it doesn't clip through the legs or jacket overlay during walk
- [ ] Jacket/pants overlay — confirm no z-fighting with the base body/leg cubes
- [ ] Form transition CAT ↔ HUMANOID — no visual pop/snap during the switch itself

## FarmerRobot
- [ ] Idle
- [ ] Walk / run
- [ ] Harvest action pose (ease-in/hold/ease-out — check the hold doesn't look frozen)
- [ ] Plant action pose
- [ ] Open chest action pose
- [ ] Store items action pose
- [ ] Look-around action pose
- [ ] Celebrate action pose
- [ ] Hurt / death
- [ ] Verify action poses don't fight with locomotion if triggered while walking

## MagnetCarrier
- [ ] Idle
- [ ] Walk / run
- [ ] Carrying-parcel state — parcel bone position/orientation looks attached, not floating or clipped
- [ ] Non-carrying state — parcel bone hidden/empty cleanly, no stray geometry
- [ ] Hurt / death (note: extends `Entity`, not `LivingEntity` — confirm death behaves sensibly given that)

## Cross-cutting
- [ ] Texture UVs line up correctly on every cube (no shifted/mirrored texture patches) — spot-check ears, whiskers, and both furnace-adjacent overlay parts
- [ ] No z-fighting or gap at any parent/child pivot joint at rest pose
- [ ] Vanilla and GeckoLib versions look like *the same pose* at idle frame 0 (this is the cheapest way to catch a leftover sign error the automated pivot check didn't happen to exercise)

Do not proceed to Phase 3 until this checklist has been run once by a developer in-game.
