# WorldForge → Minecraft 26.2 API Fix Report

Verification method: bytecode inspection of the provided `client.jar` (readable/mapped
names — this build ships with Mojang mappings baked in) using a constant-pool/method
table parser. No replacement below was guessed; every "Fixed" row was confirmed by
finding the exact member (or its verified replacement) in the jar. Everything under
"Not fixed" is left untouched in the source because it could not be verified from
client.jar, the project's mappings, or its declared dependencies.

## PART 1 — catbot module (this delivery)

| # | Class | Line(s) | Old API | Status | New API used | Confidence |
|---|-------|---------|---------|--------|--------------|------------|
| 1 | CatBotCommands.java | 110,134,170,198,287 | `ServerPlayer.getServer()` | Fixed | `player.level().getServer()` (`ServerPlayer.level()` returns `ServerLevel`, which has `getServer()`) | High |
| 2 | CatBuildCommand.java | 98 | `ServerPlayer.getServer()` | Fixed | `player.level().getServer()` | High |
| 3 | CatRespawnScheduler.java | 97 | `ServerPlayer.getServer()` | Fixed | `owner.level().getServer()` | High |
| 4 | CatBotCommands.java | 78 | `Entity.moveTo(double,double,double,float,float)` | Fixed | `Entity.snapTo(double,double,double,float,float)` — `moveTo` no longer exists on `Entity`; `snapTo` has the identical `(D,D,D,F,F)` signature | High |
| 5 | CatRespawnScheduler.java | 103 | `Entity.moveTo(D,D,D,F,F)` | Fixed | `Entity.snapTo(D,D,D,F,F)` | High |
| 6 | CatBotEntity.java | 161,207,485 | `Level.isClientSide` (field access) | Fixed | `Level.isClientSide()` — field is now private, public accessor method of the same name added | High |
| 7 | CatBotEntity.java | 239,255 | `MobEffects.MOVEMENT_SPEED` | Fixed | `MobEffects.SPEED` — field renamed (still `Holder<MobEffect>`, `MobEffectInstance` constructors unchanged) | High |
| 8 | CatBotEntity.java | 248 | `SoundEvents.GENERIC_EAT` used as `SoundEvent` | Fixed | `SoundEvents.GENERIC_EAT.value()` — the constant is now `Holder.Reference<SoundEvent>`; `Entity.playSound` still wants a raw `SoundEvent`, obtained via `Holder.value()` | High |
| 9 | CatBotEntity.java | 383 | `@Override getDimensions(Pose)` | Fixed | `@Override protected getDefaultDimensions(Pose)` — `LivingEntity.getDimensions(Pose)` is now `public final`; the intended override point is the new protected `getDefaultDimensions(Pose)` hook | High |
| 10 | CatBotEntity.java | 554-556 | `ValueInput.getByte(String)` (returned `Optional<Byte>`) | Fixed | `ValueInput.getByteOr(String, byte)` — returns the primitive directly, no `Optional` wrapper exists anymore | High |

## PART 1 — NOT FIXED (left unchanged, needs a human decision)

| Class | Line(s) | Old API | Why it can't be safely auto-fixed | Probable new API | Confidence |
|---|---|---|---|---|---|
| CatBotEntity.java | 453 | `SoundEvents.CAT_AMBIENT` | Field removed. Vanilla `Cat` no longer has flat `CAT_AMBIENT/HURT/DEATH/PURR` constants — sounds are now resolved per cat variant through `net.minecraft.world.entity.animal.feline.CatSoundVariants` / `CatSoundVariant.CatSoundSet` (confirmed: vanilla `Cat.getAmbientSound()` now calls a private `getSoundSet()` backed by a `Holder`-keyed variant map, not a static constant). Picking *which* variant's sound to reuse for a non-cat-variant custom mob is a design decision, not a mechanical rename. | `CatSoundVariants` / `CatSoundVariant.CatSoundSet` (needs manual mapping) | Low |
| CatBotEntity.java | 458 | `SoundEvents.CAT_HURT` | same as above | same | Low |
| CatBotEntity.java | 463 | `SoundEvents.CAT_DEATH` | same as above | same | Low |
| CatBotEntity.java | 507 | `SoundEvents.CAT_PURR` | same as above | same | Low |
| CatBotEntity.java, CatBotGeoModel.java (client/) | 40-45, 89-92, 98, 325-326, 329, 339 (Entity); 36,62,67 (GeoModel) | `com.geckolib.*` imports (`AnimatableInstanceCache`, `AnimatableManager`, `AnimationController`, `AnimationTest`, `RawAnimation`, `PlayState`), `GeoModel.getTextureResource(GeoRenderState)` | GeckoLib is a third-party library, not part of `client.jar`. The project's own `build.gradle` documents that GeckoLib 5.5.3 for MC 26.2 is not published to GeckoLib's Maven yet and the `libs/geckolib-fabric-26.2-5.5.3.jar` fallback file was **not present** in the uploaded project. There is no jar to verify any GeckoLib-side API/signature change against, so per the "never guess" instruction these are left untouched. | Unknown — depends entirely on the eventual GeckoLib 26.2 release | None (unverifiable) |

Both `com.worldforge.catbot.client.CatBotGeoModel` and the GeckoLib-related lines inside
`CatBotEntity.java` are unchanged. They will keep failing to compile until a real
`geckolib-fabric-26.2-5.5.3.jar` (or whatever version actually ships) is supplied — at
that point the same bytecode-verification pass can be re-run against that jar.

---

## PART 2 — farmer / magnet / chest / config / furnace / placement / undo modules

Fixes below split into (a) the errors from the original excerpted log, and (b) the
same verified patterns caught by a project-wide sweep afterwards — the excerpted log
was incomplete (e.g. it didn't mention the `furnace` module at all, even though it has
the identical `isClientSide` issue).

### (a) From the original error log

| # | Class | Line(s) | Old API | Status | New API used | Confidence |
|---|-------|---------|---------|--------|--------------|------------|
| 11 | ChestMoverItem.java | 58 | `CustomData.contains(String)` | Fixed | `CustomData.copyTag().contains(String)` — `CustomData` has no `contains`; `copyTag()` returns the `CompoundTag`, which does | High |
| 12 | ChestMoverItem.java | 86,90 | `ValueOutput.buildResult()` | Fixed | Declared the variable as `TagValueOutput` instead of the `ValueOutput` interface — `buildResult()` only exists on the concrete `TagValueOutput` class, not the interface | High |
| 13 | WFConfigKeyInit.java | 39 | `KeyMapping(..., String category)` | Fixed | `KeyMapping(..., KeyMapping.Category)` — the 4th constructor argument is now a `KeyMapping.Category` record; built with the public `KeyMapping.Category.register(Identifier)` factory. **Assumption flagged:** I registered it under `Identifier.fromNamespaceAndPath("worldforge", "worldforge")` — the category's *id* itself isn't specified anywhere verifiable, so double check this string is what you want it to display as/be keyed by. | Medium (API shape verified; exact id string is a judgment call) |
| 14 | WFConfigKeyInit.java | 44 | `Minecraft.screen` | Fixed | `Minecraft.getInstance().gui.screen()` — the `screen` field and its accessors moved from `Minecraft` onto `Minecraft.gui` (a `Gui` instance): `Gui.screen()` getter, `Gui.setScreen(Screen)` setter | High |
| 15 | WFConfigKeyInit.java | 53 | `client.screen` | Fixed | `client.gui.screen()` | High |
| 16 | WFConfigKeyInit.java | 54 | `client.setScreen(...)` | Fixed | `client.gui.setScreen(...)` | High |
| 17 | WFConfigScreen.java | 172 | `Screen.hasShiftDown()` (static) | Fixed | `Minecraft.getInstance().hasShiftDown()` — no longer static on `Screen`; it's a public instance method on `Minecraft` (also present on `InputWithModifiers`) | High |
| 18 | FarmerRobotEntity.java | 747,823 | `ResourceKey.location()` | Fixed | `ResourceKey.identifier()` — `ResourceLocation` was renamed to `Identifier` project-wide in 26.2, and `ResourceKey.location()` was renamed to `.identifier()` accordingly | High |
| 19 | FarmerRobotEntity.java | 831,833 | `ValueInput.getBoolean(String)` (`Optional<Boolean>` + `.orElse`) | Fixed | `ValueInput.getBooleanOr(String, boolean)` | High |

### (a) NOT fixed

| Class | Line | Old API | Why it can't be safely auto-fixed | Probable new API | Confidence |
|---|---|---|---|---|---|
| FarmerRobotEntity.java | 462 | `ServerLevel.getDayTime()` | Method removed outright (confirmed absent anywhere in `Level`/`ServerLevel` — even the datafixer for old data is literally named `DayTimeToClockFix`). It was replaced by a new per-dimension **Clock** system: `Level.getOverworldClockTime()`, `Level.getDefaultClockTime()`, and `Level.clockManager().getTotalTicks(Holder<?>)`. I can't verify whether any of these already returns a 0-24000 "time of day" value the same way the old `getDayTime() % 24000` did, or whether they use different semantics (e.g. absolute ticks, or per-custom-dimension clocks) — guessing here could silently break your day/night detection instead of just failing to compile. | `Level.getOverworldClockTime()` / `getDefaultClockTime()` / `clockManager()` (needs manual verification of semantics, ideally against GeckoLib/Fabric docs or a decompile of the method bodies) | Low |
| FarmerRobotGeoModel.java, MagnetCarrierGeoModel.java, MagnetCarrierEntity.java (GeckoLib imports) | various | `com.geckolib.*` | Same reason as Part 1: GeckoLib jar not present in the upload, nothing to verify against. | Unknown | None (unverifiable) |

### (b) Additional occurrences of the same verified patterns, found by a full-project sweep (not in the original log excerpt)

| Class | Line(s) | Old API | Fix applied |
|---|---|---|---|
| MagnetCarrierEntity.java | 74 | `Level.isClientSide` field | → `isClientSide()` |
| furnace/block/DiamondFurnaceBlock.java | 39, 46 | `Level.isClientSide` field | → `isClientSide()` |
| furnace/block/IronFurnaceBlock.java | 39, 46 | same | → `isClientSide()` |
| furnace/block/GoldFurnaceBlock.java | 39, 46 | same | → `isClientSide()` |
| furnace/block/EmeraldFurnaceBlock.java | 39, 46 | same | → `isClientSide()` |
| furnace/block/NetheriteFurnaceBlock.java | 39, 46 | same | → `isClientSide()` |
| furnace/block/CopperFurnaceBlock.java | 39, 46 | same | → `isClientSide()` |
| placement/TickedTaskScheduler.java | 432 | `ServerPlayer.getServer()` | → `player.level().getServer()` |
| magnet/MagnetLinkManager.java | 108, 142 | `ServerPlayer.getServer()` | → `player.level().getServer()` |
| farmer/FarmerRobotMenu.java | 140 | `ServerPlayer.getServer()` (x2 on one line) | → `player.level().getServer()` |
| farmer/FarmerRobotSpawnItem.java | 107 | `Entity.moveTo(D,D,D,F,F)` | → `Entity.snapTo(D,D,D,F,F)` |
| magnet/MagnetLinkManager.java | 346 | `ResourceKey.location()` (via `MagnetLink.dimension()`) | → `.identifier()` |

Everything above was double-checked against `client.jar` the same way as Part 1 — no
new guesses, just the same already-verified replacements applied consistently
everywhere they occur. `undo/UndoManager.java` (lines 142, 176) and
`command/WorldForgeCommands.java` (line 1534) also matched the sweep's regex but were
confirmed **already correct** (`((ServerLevel) player.level()).getServer()` and
`CommandSourceStack.getServer()`, respectively, both still valid in 26.2) — left
untouched.

## Summary of everything still failing to compile after both halves

Only two categories of failure remain, both because the required artifact wasn't
available to verify against, not because the fix is unknown-in-principle:

1. **All GeckoLib usage** (`com.geckolib.*` imports/types in `CatBotEntity.java`,
   `MagnetCarrierEntity.java`, `FarmerRobotEntity.java` and the three `*GeoModel.java`
   classes). Supply `libs/geckolib-fabric-26.2-5.5.3.jar` (or whatever version
   actually ends up targeting 26.2) and this same verification pass can be re-run
   against it.
2. **`ServerLevel.getDayTime()`** in `FarmerRobotEntity.isNightTime()` — needs a human
   call on which of `getOverworldClockTime()` / `getDefaultClockTime()` /
   `clockManager()` matches the old day/night-window semantics.

---

## PART 3 — GeckoLib (`geckolib-fabric-26_2-5_5_3.jar` now supplied) + `getDayTime()`

Verification method for this part: a custom classfile parser (no `javap`/decompiler
available in this environment — this sandbox ships a JRE, not a JDK) reading the
constant pool, method table, and generic `Signature` attributes directly out of the
`.class` files inside the supplied jar and the supplied `client.jar`. Same rule as
before: nothing below was guessed from memory of older GeckoLib versions.

### (a) Fixed

| # | Class | Old | Problem found | Fix applied | Confidence |
|---|-------|-----|--------|--------------|------------|
| 20 | FarmerRobotEntity.java, CatBotEntity.java, MagnetCarrierEntity.java | `import com.geckolib.core.animatable.instance.AnimatableInstanceCache;` + 5 sibling imports (`core.animation.AnimatableManager/AnimationController/AnimationTest/RawAnimation`, `core.object.PlayState`) | The `com.geckolib.core.*` package doesn't exist in this jar at all — every single one of these 6 imports (×3 files = 18 lines) would fail with "package does not exist" | Corrected to the real packages: `com.geckolib.animatable.instance.AnimatableInstanceCache`, `com.geckolib.animatable.manager.AnimatableManager`, `com.geckolib.animation.AnimationController`, `com.geckolib.animation.state.AnimationTest`, `com.geckolib.animation.RawAnimation`, `com.geckolib.animation.object.PlayState` | High |
| 21 | FarmerRobotGeoModel.java, MagnetCarrierGeoModel.java, CatBotGeoModel.java | `@Override Identifier getModelResource(T animatable)` / `getTextureResource(T animatable)` | Real `GeoModel<T>` signatures are `getModelResource(GeoRenderState)` and `getTextureResource(GeoRenderState)` — **the parameter is the render state, not the animatable.** (`getAnimationResource(T animatable)` is the one method of the three that *did* keep the animatable parameter — confirmed via the generic `Signature` attribute.) All three subclasses had `@Override` on a method that doesn't override anything; would not compile. | Farmer/Magnet: trivial, since both always returned a constant regardless of the parameter — just retyped the parameter to `GeoRenderState`. CatBot: genuinely needs the entity's `getForm()`/`getHairColor()`, which the new signature no longer provides at that point, so added an `addAdditionalStateData(CatBotEntity, Object, GeoRenderState)` override (this one *does* still receive the animatable) that stashes both values onto the render state via two new `DataTicket`s (`DataTicket.create(String, Class)`, confirmed factory), and `getModelResource`/`getTextureResource` now read them back off `GeoRenderState.getGeckolibData(ticket)` | High |
| 22 | FarmerRobotEntity.java line ~462 | `serverLevel.getDayTime()` | Confirmed removed from both `Level` and `ServerLevel` — no method by that name anywhere in `client.jar`. Replacements present: `Level.getOverworldClockTime()`, `Level.getDefaultClockTime()`, `Level.getClockTimeTicks(Optional<Holder<WorldClock>>)`, `Level.clockManager()`. Checked the actual registered overworld clock data (`data/minecraft/timeline/day.json`): `period_ticks` is still `24000`, and vanilla's own `night` time-marker still fires at tick `13000` — same numbers the mod's own window already used. | `getDayTime()` → `getDefaultClockTime()` (per-dimension default clock, since this entity isn't overworld-only), keeping the existing `% 24000L` and `13000..23000` window unchanged | High (API existence + period/marker ticks verified from real registry data; the `13000-23000` window itself was always the mod's own constant, not something vanilla exposes as a single call, so there's nothing further to verify there) |

### (b) Resolved (previously an open naming question)

`KeyMapping.Category.register(Identifier.fromNamespaceAndPath("worldforge", "worldforge"))`
(item #13) compiles correctly, and the actual translation key it needs has now been
pinned down exactly by disassembling `KeyMapping.Category.label()` and
`Identifier.toLanguageKey(String)` byte-by-byte (no `javap` in this sandbox, so I wrote
a small opcode-length-aware disassembler to read the `Code` attribute and the
`invokedynamic`/`StringConcatFactory` recipe strings directly): the real key is
**`"key.category." + namespace + "." + path`**, not the old pre-refactor
`"key.categories.<name>"` style. For this project's id that's
**`key.category.worldforge.worldforge`** (the "worldforge.worldforge" duplication is
just because the id's namespace and path were both set to `"worldforge"` — swap the
path for something else, e.g. `"general"`, if you'd rather avoid the repeat).
`en_us.json` and `tr_tr.json` had a stale `key.categories.worldforge` entry from
before this API existed — renamed to `key.category.worldforge.worldforge` in both.
(`es_es.json`, `fr_fr.json`, `de_de.json` have no `key.*` entries at all yet — that's a
pre-existing gap unrelated to this fix, not something newly broken by it.)

## PART 4 — `StringWidget#alignCenter()` / `#setColor(int)` (15 errors)

Verification method: user-supplied `javap` output confirmed `StringWidget` now exposes
only `setMessage(...)`, `setMaxWidth(...)`, `getWidth()`, `visitLines(...)` — no
`alignCenter()`, no `setColor(int)`.

| # | Class | Line(s) | Old API | Status | New API used | Confidence |
|---|-------|---------|---------|--------|--------------|------------|
| 23 | WFCatCustomizeScreen.java | 77-78, 187, 260 | `StringWidget.alignCenter()` / `.setColor(int)` | Fixed | Centering: `widget.setX((screenWidth - widget.getWidth()) / 2)`, using the fact that `getWidth()` still returns the text's actual rendered pixel width capped by the box width. Color: no longer a widget property — colored text is now passed in as a styled `Component` (`base.copy().withStyle(Style.EMPTY.withColor(TextColor.fromRgb(rgb)))`) via the constructor/`setMessage`. | Medium (mechanism matches vanilla's known `AbstractStringWidget` evolution; not independently re-verified against `client.jar` in this pass — only the user's `javap` excerpt for `StringWidget` itself, not `AbstractWidget.setX`/`getWidth`'s exact semantics) |
| 24 | WFConfigScreen.java | 65-66, 77, 143, 154 | same | Fixed | same pattern | Medium |
| 25 | WFImageViewerScreen.java | 126, 130-131, 147, 152, 157 | same | Fixed | same pattern, plus re-centering (`centerX()`) on every `setMessage()` call in `syncStatusWidgets()` since that text (and thus its width) changes every frame | Medium |

**Flag for next compile:** `AbstractWidget.setX(int)` is assumed to still exist and be
public (it wasn't in the errors reported, only `StringWidget`'s own removed methods
were) — if it's been renamed/hidden too, centering will fail to compile and needs the
same javap treatment as `StringWidget` got here.

## PART 5 — `MegaChestRenderer.submit()` (`OrderedSubmitNodeCollector` chain)

Verification method: chat-session bytecode inspection of `client.jar`, confirmed
across multiple turns for `OrderedSubmitNodeCollector.submitModelPart(...)`,
`SpriteId`, `AtlasManager`, `TextureAtlas.LOCATION_BLOCKS`, and the
`ResourceLocation` → `Identifier` rename. `submit()` was previously left
intentionally empty pending that verification; it is now filled in.

| # | Class | Line(s) | Old API | Status | New API used | Confidence |
|---|-------|---------|---------|--------|--------------|------------|
| 26 | MegaChestRenderer.java | field decl | `ResourceLocation LOCK_TEXTURE` (leftover from before submit() was blocked; project-wide `ResourceLocation`→`Identifier` rename, see #18, was never applied to this file) | Fixed | `Identifier LOCK_SPRITE` + `SpriteId LOCK_SPRITE_ID = new SpriteId(TextureAtlas.LOCATION_BLOCKS, LOCK_SPRITE)` — also switched from a `textures/block/vault_lock.png` texture path to the atlas-sprite path convention `block/vault_lock` (no `textures/` prefix, no extension) | High (`SpriteId` constructor signature confirmed against client.jar) |
| 27 | MegaChestRenderer.java | `submit()` | *(empty stub)* | Fixed | `Minecraft.getInstance().getModelManager().atlasManager().get(SpriteId)` → `TextureAtlasSprite`, then `collector.order(0).submitModelPart(ModelPart, PoseStack, RenderType, int light, int overlay, TextureAtlasSprite)` | High for the collector/sprite chain (client.jar-verified); see flag below for the one unverified piece |
| 28 | MegaChestRenderer.java | constructor | *(none — ModelPart didn't exist yet)* | Fixed | Added a fixed 1×1-block (16×16px) `ModelPart` decal plane, baked once via `MeshDefinition`/`LayerDefinition`/`CubeListBuilder`/`PartPose` in the constructor (not per-frame). Per-instance sizing (`state.halfSize`, which varies with the MegaChest's multi-block bounding box) is applied at `submit()` time via `PoseStack.scale(2*halfSize, ...)` instead of rebuilding the mesh, since the renderer instance is shared across all block entities of this type. | Medium — package paths (`net.minecraft.client.model.geom.*` / `.builders.*`) are historically stable Mojmap names but were **not** individually re-verified against this project's 26.2 `client.jar` in this pass |
| 29 | MegaChestRenderer.java | `extractRenderState()` | *(no light computation)* | Fixed | Added `state.packedLight = LevelRenderer.getLightColor(be.getLevel(), self)`, computed once in `extractRenderState()` and carried on the state object (not recomputed in `submit()`) | Medium — `LevelRenderer.getLightColor` signature not independently re-verified against this project's 26.2 `client.jar` |

**Flag for next compile — the one real unknown left:** `RenderType` selection.
`RenderType.entityCutoutNoCull(TextureAtlas.LOCATION_BLOCKS)` was used as the most
reasonable choice for a block-atlas-backed `ModelPart` decal (UV remapping is expected
to come from the `TextureAtlasSprite` argument passed separately to
`submitModelPart`, not from the `RenderType` itself), but — unlike every other row in
this table — this specific factory call was **not** confirmed against client.jar.
If compilation fails here, check `RenderType`'s available static factories first
(`VERIFY-ON-COMPILE` comment left at the call site).

**Düzeltme (manuel kontrol turu sonrası):** Part 5'teki ilk sürümde iki kesin hata
tespit edildi ve düzeltildi:
- `RenderType` importu yanlıştı: `net.minecraft.client.renderer.RenderType` değil,
  `net.minecraft.client.renderer.rendertype.RenderType` olmalı (26.2'de paket taşınmış).
- `MegaChestRenderState`, `BlockEntityRenderState`'i `implements` ediyordu; client.jar'da
  bu artık bir interface değil, bir `class` — `extends` kullanılması gerekiyor.

Geri kalan işaretli noktalar (`RenderType.entityCutoutNoCull(...)`'un var olup olmadığı,
`LayerDefinition.create(...).bakeRoot()` zinciri, `submitModelPart(...)` parametre
sırası, `LevelRenderer.getLightColor(...)` imzası) hâlâ yalnızca derleme ile teyit
edilebilir — bu geçişte client.jar'dan bağımsız olarak doğrulanmadılar.

## Updated summary

Nothing known to this project is left unverified for compile-blocking reasons anymore,
**except** the `RenderType` factory call flagged in Part 5 above, which is the last
genuinely unconfirmed API surface in the codebase. Every item that was previously
blocked on a missing artifact (`geckolib-fabric-26.2-5.5.3.jar`) or an ambiguous
replacement (`getDayTime()`) has now been resolved and fixed in the source. The one
remaining naming-preference note (`worldforge:worldforge` category id) is unchanged.

