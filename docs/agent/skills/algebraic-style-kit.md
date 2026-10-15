---
name: algebraic-style-kit
description: Use when adding, editing or reviewing an Algebraic! Fighting Style kit, move, hitbox, frame data, Quirk, passive, Awakening or Realm in src/Shared/Styles or src/Shared/Behaviors.
enabled: true
---

A Fighting Style kit in Algebraic! is a builder function producing a StyleModule {def, hooks, moment, anim}: a typed StyleDef plus small passive hooks, an optional Style Moment and animation specs. Kits are edited on disk by the coding agent; never edit them in Studio. Follow the algebraic-conventions skill as well.

## Files
- One kit per style: src/Shared/Styles/<id>.luau returns a builder function `(K: KitBuilder) -> StyleModule` where `StyleModule = {def: StyleDef, hooks: PassiveHooks, moment: MomentDef?, anim: {[slot]: AnimMoveSpec}}`; kit files require only `../Behaviors/Buff` (kit kernel) and never `Styles/init` (build spec §4.1.1). The shape is `return function(K: Kit.KitBuilder): Kit.StyleModule … end`. The registry is src/Shared/Styles/init.luau; it calls each kit with the shared constructors.
- Shared move behaviors live in src/Shared/Behaviors (Projectile, Rush, Grab, Counter, Zone, Summon, Buff, Mobility, RealmCast). A kit selects a behavior by name and passes typed params; it does not reimplement behavior logic.
- Behaviors, carriers and hooks read config only from their context (build spec §4.2.0.2): `bc.cfg` is the pinned Combat section (read `bc.cfg.Chassis.X`, never `cfg.Combat.X`), `bc.kits` is the pinned Kits section and `bc.moves(moveId)` returns the pinned kit set's MoveDef (carriers and hooks get the same `cfg`, `kits` and `moves` on their own context). Only the boot validator `validate(def, cfg)` gets the full config. Never read Runtime or a bare Kits table.
- Style IDs are fixed and IP-neutral: hero, stretch, sentry, granny, diva, tyrant, console, grappler, monarch, flame, butler, rocker, frost, huntress, decay, demon, wish, chaos. The vertical slice implements hero, stretch, monarch, rocker, frost and decay; every other style has implemented = false.

## Contract (from Shared/Types.luau)
- StyleDef: id, rarity, archetype, implemented, hp, palette {primary, secondary, accent as Color3}, passive {id, params}, quirk (MoveDef), moves (slots "1" to "4"), awakening (AwakeningDef), m1 {anim, vfx, sfx}, kitProps.
- MoveDef: id, slot, behavior, tags, signature, startup, active, recovery (frames at 60 Hz), damage, guardDamage, chip (fraction), hitstun (frames), knockback {dist, angleDeg, ragdoll, launch}, hitbox {shape Box, Sphere or Cast; size; offset; castDistance}, cooldown (seconds), parryable, superArmor, projectileArmor, telegraph {shape, leadFrames, size}, params, meterGain, anim, vfx, sfx.
- AwakeningDef: id, durationSec, hurtboxScale, modifiers, moves (a1 to a4).
- Move IDs: <style>.m1, <style>.1 to <style>.4, <style>.q (Quirk), <style>.a1 to <style>.a4, <style>.passive, <style>.awaken.

## Rules (all values come from docs/GDD.md; never invent or change them)
- Copy every number from the style's GDD §7.3 entry and the GDD §6 rules, with a comment citing the section, for example -- GDD §7.3.1. If the GDD gives no number for a field (often startup or active frames), pick a value inside the GDD bands and rules and mark it -- (tuning default).
- hp is 100 for every style at launch (GDD §6.2).
- M1 frame data is shared by all styles (GDD §6.3). A kit only supplies M1 presentation (anim, vfx, sfx).
- Each kit has 4 base moves, 1 Quirk, 1 passive and an Awakening with 4 awakened moves (GDD §6.7).
- Exactly one base move has signature = true (GDD §6.7). Every base kit has at least one opener, one extender and one defensive or mobility tool (GDD §6.7).
- No base move deals more than 18 damage. Quirks deal at most 6 damage and have a 3 to 8 s cooldown (up to 12 s for random-outcome Quirks). Cooldown bands: basic 6 to 10 s, utility 10 to 16 s, signature 14 to 22 s (GDD §6.7).
- Latency-friendly startups: at least 6 frames on M1s and at least 8 frames for big hits; AoEs have a visible fuse (GDD §18.2).
- Awakening durationSec is 60 (GDD §6.9). hurtboxScale may only be 1 or larger, never smaller (GDD §6.2).
- Realms (Legendary and above, awakened move a4, behavior RealmCast) follow GDD §6.10: 90 frame cast with a visible ring, interruptible in the first 45 frames, 40 stud radius, 12 s duration.
- Tags come only from the Tag type (Projectile, Rush, Grab, Counter, Zone, Summon, Buff, Debuff, Mobility, AoE, Utility, Stealth, Ultimate, Realm, Music, Demon, Opener, Extender).
- Hitboxes are queried against the fixed hurtbox capsule (GDD §6.2). Never size hitboxes from avatar parts.
- Frame windows and lengths are data in the kit. Simulation code never reads AnimationTrack.Length or animation markers.

## Names, assets and presentation
- Player-visible names come from the skin layer: style.<id>.name, style.<id>.epithet, move.<moveId>.name. Never write a show, character or location name into a kit, an identifier, an asset key or a comment.
- anim, vfx and sfx fields are logical keys resolved by Assets.get(key). Placeholders are allowed while their status is "placeholder".
- palette colours feed the always-on Style Tell (nameplate chip and VFX palette), which stays visible even when Character Look is off.

## Checks before a kit change is done
- The Lune kit-validation spec passes (structure, bands, exactly one signature, move ID formats, GDD citations present).
- lune run tools/check.luau prints [ALG][CHECK] SUMMARY with fail=0.
- The in-Studio scenario tests for the style print [ALG][TEST] SUMMARY with fail=0, with no Output errors.
- The frame-data export (tools/export_framedata.luau) still runs and the ARCHITECTURE.md row for the kit is updated.
