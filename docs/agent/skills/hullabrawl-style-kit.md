---
name: hullabrawl-style-kit
description: Use when adding, editing or reviewing a Hullabrawl Fighting Style kit (a Charm, to players), move, hitbox, frame data, Quirk, passive, Peak (Awakening) or Getaway (Realm) in src/Shared/Styles or src/Shared/Behaviors.
enabled: true
---

A Fighting Style kit in Hullabrawl is a builder function producing a StyleModule {def, hooks, moment, anim}: a typed StyleDef plus small passive hooks, an optional Snapshot (a style moment; field moment) and animation specs. Players call a style a Charm; code calls it a style. Kits are edited on disk by the coding agent; never edit them in Studio. Follow the hullabrawl-conventions skill as well.

## Files
- One kit per style: src/Shared/Styles/<id>.luau returns a builder function `(K: KitBuilder) -> StyleModule` where `StyleModule = {def: StyleDef, hooks: PassiveHooks, moment: MomentDef?, anim: {[slot]: AnimMoveSpec}}`; kit files require only `../Behaviors/Buff` (kit kernel) and never `Styles/init` (build spec §4.1.1). The shape is `return function(K: Kit.KitBuilder): Kit.StyleModule … end`; follow the kit pattern of build spec §4.5 (the attendant kit in §4.5.2 is the reference). The registry is src/Shared/Styles/init.luau; it calls each kit with the shared constructors and also holds the 12 stub StyleDefs.
- Shared move behaviors live in src/Shared/Behaviors (Projectile, Rush, Grab, Counter, Zone, Summon, Buff, Mobility, RealmCast). A kit selects a behavior by name and passes typed params; it does not reimplement behavior logic.
- Behaviors, carriers and hooks read config only from their context (build spec §4.2.0.2): `bc.cfg` is the pinned Combat section (read `bc.cfg.Chassis.X`, never `cfg.Combat.X`), `bc.kits` is the pinned Kits section and `bc.moves(moveId)` returns the pinned kit set's MoveDef (carriers and hooks get the same `cfg`, `kits` and `moves` on their own context). Only the boot validator `validate(def, cfg)` gets the full config. Never read Runtime or a bare Kits table.
- Style IDs are fixed internal keys, in Types.STYLE_IDS order: attendant, lifeguard, skimstone, masseur, bellhop, angler, kite, boxcamera, sandcastle, scoop, diver, busker, lamplighter, cablecar, dustsheet, raincloud, serpent, ferry. GDD §7.3.1 to §7.3.18 follow the same order. The vertical slice implements attendant, lifeguard, sandcastle, scoop, busker and dustsheet; every other style has implemented = false.

## Contract (from Shared/Types.luau)
- StyleDef: id, rarity, archetype, implemented, hp, palette {primary, secondary, accent as Color3}, passive {id, params}, quirk (MoveDef), moves (slots "1" to "4"), awakening (AwakeningDef), m1 {anim, vfx, sfx}, kitProps.
- MoveDef: id, slot, behavior, tags, signature, startup, active, recovery (frames at 60 Hz), damage, guardDamage, chip (fraction), hitstun (frames), knockback {dist, angleDeg, ragdoll, launch}, hitbox {shape Box, Sphere or Cast; size; offset; castDistance}, cooldown (seconds), parryable, superArmor, projectileArmor, telegraph {shape, leadFrames, size}, params, meterGain, anim, vfx, sfx.
- AwakeningDef (the Peak): id, durationSec, hurtboxScale, modifiers, moves (a1 to a4).
- MomentDef (the Snapshot): seq (ordered move tokens such as "attendant.1:hit"), windowF, sameVictim.
- Move IDs: <style>.m1, <style>.1 to <style>.4, <style>.q (Quirk), <style>.a1 to <style>.a4, <style>.passive, <style>.awaken.

## Rules (all values come from docs/GDD.md; never invent or change them)
- Copy every number from the style's GDD §7.3 entry and the GDD §6 rules. Every move line ends with a pointer comment -- gdd: §7.3.N <slot>, for example -- gdd: §7.3.1 row 3 or -- gdd: §7.3.1 Quirk; hook code points at the passive or Snapshot row the same way. Every number also carries a same-line comment naming its source, for example -- GDD §7.3.1 or -- GDD §6.9. If the GDD gives no number for a field (often startup or active frames), pick a value inside the GDD bands and rules and mark it -- (td), the tuning-default mark.
- hp is 100 for every style at launch (GDD §6.2).
- M1 frame data is shared by all styles (GDD §6.3). A kit only supplies M1 presentation (anim, vfx, sfx).
- Each kit has 4 base moves, 1 Quirk, 1 passive and a Peak with 4 awakened moves (GDD §6.7).
- Exactly one base move has signature = true (GDD §6.7). Every base kit has at least one opener, one extender and one defensive or mobility tool (GDD §6.7).
- No base move deals more than 18 damage. Quirks deal at most 6 damage and have a 3 to 8 s cooldown (up to 12 s for random-outcome Quirks). Cooldown bands: basic 6 to 10 s, utility 10 to 16 s, signature 14 to 22 s (GDD §6.7). Awakened moves a1 to a3 deal at most 20 damage and a4 at most 30 (GDD §7.1 rule 4).
- Latency-friendly startups: at least 6 frames on M1s and at least 8 frames for big hits; AoEs have a visible fuse (GDD §18.2).
- Peak durationSec is 60 (GDD §6.9). hurtboxScale is 1 for every kit: no Peak scales the hurtbox (GDD §6.2). Each slice Peak sets exactly one recognized modifier (build spec §4.4.2). At Peak the style's uniform appears for 60 s; nobody grows.
- Getaways (Realm in code; Legendary and above, awakened move a4, behavior RealmCast) follow GDD §6.10: 90 frame cast with a visible ring, interruptible in the first 45 frames, 40 stud radius, 12 s duration. Their house rules are keyed by move ID, for example Realm.rules["dustsheet.a4"].
- Engine additions are only E2 (chase targets the enemy nearest the carrier, so a speed-0 summon works as a trap or turret; it lands in M0 with the Summon behavior, because attendant.q needs it) and E1 (M1-G: a Realm tick rule can read the member's position). E3 to E7 are Year-1 future hooks; do not build them in a slice kit.
- Deferred features have no slice user and the validator rejects them in kit data: Buff toggleMode, Rush echo, Mobility modes other than dash, Summon ai "marked", and Counter reflectProjectiles = true. Keep their types; build them with the first style that needs them.
- Tags come only from the Tag type (Projectile, Rush, Grab, Counter, Zone, Summon, Buff, Debuff, Mobility, AoE, Utility, Stealth, Ultimate, Realm, Music, Opener, Extender).
- Hitboxes are queried against the fixed hurtbox capsule (GDD §6.2). Never size hitboxes from avatar parts.
- Frame windows and lengths are data in the kit. Simulation code never reads AnimationTrack.Length or animation markers.

## Names, assets and presentation
- Player-visible names come from the Base skin pack: style.<id>.name, style.<id>.epithet, move.<moveId>.name. Never write the name of an existing show, film, game or character into a kit, an identifier, an asset key or a comment, and never describe a move by comparison with one.
- anim, vfx and sfx fields are logical keys resolved by Assets.get(key). Placeholders are allowed while their status is "placeholder".
- kitProps lists exactly two Charm Gear model keys, the oversized held or back prop that carries the silhouette first and the head piece second (model.<id>.gear_<name>, for example model.attendant.gear_chair, model.attendant.gear_boater), followed by every summon model key (model.<id>.summon_<name>). Every Summon move's params.model must be in kitProps. Only the gear_ entries are attached to characters.
- A slice style may also have one optional Peak piece (model.<id>.peak_<name>), built from primitives like the Charm Gear and shown only at Peak. It is not listed in kitProps.
- palette colours feed the always-on Style Tell (nameplate chip and VFX palette), which stays visible even when the Full Uniform toggle is off.

## Checks before a kit change is done
- The Lune kit-validation spec passes (structure, bands, exactly one signature, move ID formats, GDD pointers present).
- lune run tools/check.luau prints [HB][CHECK] SUMMARY with fail=0 (this includes the string and retired-key lints).
- The in-Studio scenario tests for the style print [HB][TEST] SUMMARY with fail=0, with no Output errors.
- The frame-data export (tools/export_framedata.luau) still runs and the ARCHITECTURE.md row for the kit is updated.
- A change to what a move does or shows, a palette or a Charm Gear or summon recipe is a drift change: the same task appends pending T8 rows (and a T11 row for the style) to docs/agent/RECOGNIZABILITY.md. Humans run the blind tests and sign the results.
