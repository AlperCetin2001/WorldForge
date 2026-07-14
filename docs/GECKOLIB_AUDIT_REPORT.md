# WorldForge — GeckoLib Rendering Isolation Layer (formerly "Phase 1-3: Optional GeckoLib")

> **Final decision (post-Phase 3):** GeckoLib remains a **required** runtime dependency
> (`fabric.mod.json` keeps `depends: geckolib`). CatBot/FarmerRobot/MagnetCarrier implement
> `GeoAnimatable`, which the JVM resolves at class-load time regardless of any runtime check —
> confirmed against GeckoLib's own official wiki (github.com/bernie-g/geckolib/wiki), which
> documents no supported soft-dependency pattern for animatable entities. Splitting into a
> separate `WorldForge Core` + `WorldForge GeckoLib Compat` mod was considered and rejected for
> this project: CatBot/FarmerRobot/MagnetCarrier's gameplay (AI, NBT, inventory, goals,
> pathfinding, networking) lives on the same entity class that must implement `GeoAnimatable`,
> so a real split would mean moving half of gameplay into the compat mod, not just rendering —
> not worth the multi-module build/publish/CI overhead at WorldForge's current scale.
>
> **What this phase actually delivers, and why it's still worth keeping:** gameplay is fully
> decoupled from rendering (`compat/vanilla` has zero GeckoLib imports), there's a single
> renderer-selection point (`RendererCompatInit`), and every GeckoLib-specific class lives under
> `compat/geckolib`. If GeckoLib ever ships a breaking v6 API change, or if a future version of
> this project *does* want the Core+Compat split, only `compat/geckolib` needs to change —
> gameplay code is untouched either way. The vanilla renderer built here is currently unused at
> runtime (GeckoLib is mandatory), but it's validated, tested, and ready to be switched on the
> moment the dependency stance changes.

Scope: read-only audit. No code changed. Goal: map every compile-time
GeckoLib dependency so it can be made optional (vanilla renderer fallback
for users without GeckoLib installed).

Note: actual import root in this codebase is `com.geckolib.*`
(GeckoLib5, 5.5.3), not the older `software.bernie.geckolib.*`. The
`software.bernie.geckolib` string only appears as a comment/dead code
in `build.gradle` (an abandoned Cloudsmith repo filter). Confirmed via
full-project grep for both strings.

Three entities are affected: **CatBot**, **FarmerRobot**, **MagnetCarrier**.
No other systems (WFVault, MegaChest, furnace, tools, wfbuild, magnet
link item transfer, image/map-art, etc.) touch GeckoLib at all.

---

## 1. Every GeckoLib import (by file)

**CatBotEntity.java**
- `com.geckolib.animatable.GeoEntity`
- `com.geckolib.animatable.instance.AnimatableInstanceCache`
- `com.geckolib.animatable.manager.AnimatableManager`
- `com.geckolib.animation.AnimationController`
- `com.geckolib.animation.state.AnimationTest`
- `com.geckolib.animation.RawAnimation`
- `com.geckolib.animation.object.PlayState`
- `com.geckolib.util.GeckoLibUtil`

**catbot/client/CatBotGeoModel.java**
- `com.geckolib.model.GeoModel`
- `com.geckolib.renderer.base.GeoRenderState`
- `com.geckolib.constant.dataticket.DataTicket`

**catbot/client/CatBotGeoRenderer.java**
- `com.geckolib.renderer.GeoEntityRenderer`
- `com.geckolib.renderer.base.GeoRenderState`
- `com.geckolib.renderer.base.RenderPassInfo`
- `com.geckolib.renderer.layer.builtin.ItemInHandGeoLayer`

**MagnetCarrierEntity.java** — same 8 imports as CatBotEntity (GeoEntity,
AnimatableInstanceCache, AnimatableManager, AnimationController,
AnimationTest, RawAnimation, PlayState, GeckoLibUtil)

**magnet/client/MagnetCarrierGeoModel.java**
- `com.geckolib.model.GeoModel`
- `com.geckolib.renderer.base.GeoRenderState`

**magnet/client/MagnetCarrierGeoRenderer.java**
- `com.geckolib.renderer.GeoEntityRenderer`
- `com.geckolib.renderer.base.GeoRenderState`

**FarmerRobotEntity.java** — same 8 core imports as CatBotEntity, plus
two fully-qualified inline references (no import statement, used
in-line):
- `com.geckolib.animation.state.KeyFrameEvent`
- `com.geckolib.cache.animation.keyframeevent.SoundKeyframeData`
- `com.geckolib.cache.animation.keyframeevent.ParticleKeyframeData`

**farmer/client/FarmerRobotGeoModel.java**
- `com.geckolib.model.GeoModel`
- `com.geckolib.renderer.base.GeoRenderState`
- `com.geckolib.constant.dataticket.DataTicket`

**farmer/client/FarmerRobotGeoRenderer.java**
- `com.geckolib.renderer.GeoEntityRenderer`
- `com.geckolib.renderer.base.GeoRenderState`
- `com.geckolib.renderer.layer.builtin.ItemInHandGeoLayer`

**farmer/client/FarmerGlowGeoLayer.java**
- `com.geckolib.renderer.GeoEntityRenderer`
- `com.geckolib.renderer.base.GeoRenderState`
- `com.geckolib.renderer.layer.builtin.AutoGlowingGeoLayer`

**farmer/client/FarmerHeadTrackingGeoLayer.java**
- `com.geckolib.renderer.base.GeoRenderer`
- `com.geckolib.renderer.base.GeoRenderState`
- `com.geckolib.renderer.base.RenderPassInfo`
- `com.geckolib.renderer.layer.GeoRenderLayer`
- (inline) `com.geckolib.animation.state.BoneSnapshot`

**Comment-only mentions (no import, no code dependency):**
`CatHairColor.java`, `catbot/build/StructureTemplates.java`,
`catbot/client/CatBotClientInit.java`, `magnet/client/MagnetClientInit.java`,
`farmer/client/FarmerClientInit.java`, `FarmerRobotVariant.java` — these
just say "GeckoLib" in javadoc/comments; no `import` line.

---

## 2. Every class that depends on GeckoLib

| Class | Dependency type |
|---|---|
| `CatBotEntity` | implements `GeoEntity`, holds `AnimatableInstanceCache`, `AnimationController` registration |
| `CatBotGeoModel` | extends `GeoModel<CatBotEntity>` |
| `CatBotGeoRenderer` | extends `GeoEntityRenderer<CatBotEntity, R>`, generic bound `& GeoRenderState` |
| `MagnetCarrierEntity` | implements `GeoEntity`, holds `AnimatableInstanceCache`, `AnimationController` registration |
| `MagnetCarrierGeoModel` | extends `GeoModel<MagnetCarrierEntity>` |
| `MagnetCarrierGeoRenderer` | extends `GeoEntityRenderer<MagnetCarrierEntity, R>` |
| `FarmerRobotEntity` | implements `GeoEntity`, holds `AnimatableInstanceCache`, `AnimationController` registration, keyframe event handlers |
| `FarmerRobotGeoModel` | extends `GeoModel<FarmerRobotEntity>` |
| `FarmerRobotGeoRenderer` | extends `GeoEntityRenderer<FarmerRobotEntity, R>` |
| `FarmerGlowGeoLayer` | extends `AutoGlowingGeoLayer<FarmerRobotEntity, Void, R>` |
| `FarmerHeadTrackingGeoLayer` | extends `GeoRenderLayer<FarmerRobotEntity, Void, R>` |

`CatBotClientInit`, `MagnetClientInit`, `FarmerClientInit` don't import
GeckoLib types directly, but each *instantiates* a Geo renderer via
method reference (`CatBotGeoRenderer::new`, etc.) — so they carry a
transitive compile-time dependency through the renderer classes.

---

## 3. Every registration that depends on GeckoLib

- `CatBotClientInit` → `EntityRenderers.register(CatBotEntities.CAT_BOT, CatBotGeoRenderer::new)`
- `MagnetClientInit` → `EntityRenderers.register(MagnetCarrierEntities.MAGNET_CARRIER, MagnetCarrierGeoRenderer::new)`
- `FarmerClientInit` → `EntityRenderers.register(FarmerRobotEntities.FARMER_ROBOT, FarmerRobotGeoRenderer::new)`
- Inside `FarmerRobotGeoRenderer`'s constructor: registers `ItemInHandGeoLayer`, `FarmerHeadTrackingGeoLayer`, `FarmerGlowGeoLayer` as render layers.
- Inside `CatBotGeoRenderer`'s constructor: registers `ItemInHandGeoLayer`.
- `registerControllers(AnimatableManager.ControllerRegistrar)` in all three entity classes — this is the GeckoLib animation-controller registration, called by GeckoLib's own animation manager at runtime (not by WorldForge's registry code).

**Not GeckoLib-dependent:** the `EntityType<T>` registrations themselves
(`CatBotEntities.CAT_BOT`, `MagnetCarrierEntities.MAGNET_CARRIER`,
`FarmerRobotEntities.FARMER_ROBOT` in their respective `*Entities.java`
files) use only vanilla/Fabric API (`EntityType.Builder`,
`FabricDefaultAttributeRegistry`, `BuiltInRegistries`) — these are
Category A and register fine with or without GeckoLib present, since
the entity *type* doesn't require `GeoEntity` at the registry level,
only the entity *class* does (compile-time only, not registry-time).

**fabric.mod.json**: `"depends": { ... "geckolib": ">=5.5.3" }` — this
is a **hard dependency**. As currently written, Fabric Loader will
refuse to load WorldForge at all if GeckoLib isn't installed. This
single line defeats the entire goal of the phase and must move to
`"suggests"` (or be removed and handled by the runtime `FabricLoader.isModLoaded("geckolib")` check instead) as part of the eventual fix.

---

## 4. Files that can stay untouched (Category A)

Everything not listed in sections 1–3. Notably, despite mentioning
GeckoLib in comments only:
- `CatHairColor.java`
- `catbot/build/StructureTemplates.java`
- `FarmerRobotVariant.java`

Plus all AI/goal classes (`ApproachOwnerGoal`, `ReturnToOwnerGoal`,
`WorkSiteWanderGoal`), `CatBotCommands`, `CatBotManager`, `CatBotSounds`,
`CatRespawnScheduler`, `CatBuildCommand`, `WFCatCustomizeScreen`,
`WFCatSetPayload`, `FarmerChestBlockEntity`, `TickedTaskScheduler`,
and the three `*Entities.java` registry files — all reference the
*entity classes* (`CatBotEntity`, etc.) by type but call zero GeckoLib
APIs directly. These become Category C only insofar as they hold a
reference to a class that itself needs refactoring (see §6) — the
files themselves need no changes.

---

## 5. Files that must move to `compat/geckolib` (Category B)

Pure-GeckoLib client rendering/model/layer classes — no gameplay logic,
safe to relocate wholesale:

- `catbot/client/CatBotGeoModel.java`
- `catbot/client/CatBotGeoRenderer.java`
- `magnet/client/MagnetCarrierGeoModel.java`
- `magnet/client/MagnetCarrierGeoRenderer.java`
- `farmer/client/FarmerRobotGeoModel.java`
- `farmer/client/FarmerRobotGeoRenderer.java`
- `farmer/client/FarmerGlowGeoLayer.java`
- `farmer/client/FarmerHeadTrackingGeoLayer.java`

(8 files total)

---

## 6. Files that require refactoring (Category C)

These can't simply move — they mix gameplay logic with GeckoLib API
surface, so GeckoLib types must be abstracted out (e.g. via an
interface WorldForge owns, with a GeckoLib-backed impl in `compat/`
and a no-op/vanilla impl in core):

- **`CatBotEntity.java`** — `implements GeoEntity`, `AnimatableInstanceCache` field, `registerControllers(...)`, `getAnimatableInstanceCache()`. All gameplay state (AI, inventory, forms, hair color) lives in the same class.
- **`MagnetCarrierEntity.java`** — same pattern, smaller class.
- **`FarmerRobotEntity.java`** — same pattern; largest and most entangled (keyframe sound/particle event handlers reference GeckoLib event types directly inside otherwise-gameplay code, lines ~945–1030).
- **`CatBotClientInit.java`, `MagnetClientInit.java`, `FarmerClientInit.java`** — currently call `EntityRenderers.register(..., XGeoRenderer::new)` unconditionally. Needs a branch: register Geo renderer if GeckoLib present, else register a vanilla fallback renderer (which doesn't exist yet — new code, out of scope for this audit).
- **`fabric.mod.json`** — `geckolib` must move from `depends` to `suggests`.
- **`build.gradle` / `gradle.properties`** — GeckoLib jar inclusion needs to become a `compileOnly`/optional-runtime dependency rather than `implementation`, so the mod compiles and runs without the jar present (currently `implementation files(geckolibJarFile)` — hard compile dependency, and build already warns-but-continues if the jar is missing, meaning **the project currently only "compiles" today because the jar happens to be sitting in `libs/`**).

---

## 7. Potential `ClassNotFoundException` / `NoClassDefFoundError` risks

- **Highest risk — entity classes themselves**: `CatBotEntity`,
  `MagnetCarrierEntity`, `FarmerRobotEntity` all `implements GeoEntity`
  and hold `AnimatableInstanceCache` fields. If GeckoLib isn't on the
  classpath at runtime, these classes fail to *load* at all (not just
  fail to render) — since `GeoEntity` is referenced in the class
  `implements` clause, JVM class verification will throw
  `NoClassDefFoundError` the moment `EntityType.Builder.of(CatBotEntity::new, ...)`
  is touched, i.e. during mod init, crashing the whole game — not just
  disabling the cat bot. This is the central problem the refactor must solve.
- **Client init classes**: `CatBotClientInit`/`MagnetClientInit`/`FarmerClientInit`
  reference `XGeoRenderer::new`, which transitively loads `GeoEntityRenderer`
  → same `NoClassDefFoundError` risk at client entrypoint init time.
- **`registerControllers(AnimatableManager.ControllerRegistrar controllers)`**
  methods — signature itself references GeckoLib types, so even if the
  *body* were emptied, the method signature still fails to verify
  without GeckoLib on the classpath.
- **`FarmerRobotEntity` keyframe handlers** (`onActionSoundKeyframe`,
  `onActionParticleKeyframe`) — parameter types are GeckoLib event
  classes; same signature-verification risk as above.
- **fabric.mod.json hard `depends`** — currently this makes the
  NoClassDefFoundError scenario moot (Fabric Loader won't even start
  the mod without GeckoLib), but that's exactly the behavior this
  phase is meant to remove, which is what *creates* the above risks
  once removed.

---

## 8. Recommended migration order

1. **fabric.mod.json**: move `"geckolib": ">=5.5.3"` from `depends` to `suggests`. (Do this last in practice, once 2–6 are done — otherwise the game will crash-loop per item 7. Listed first only because it's the "declares intent" step conceptually.)
2. **build.gradle**: change GeckoLib jar inclusion to a soft/optional dependency (`compileOnly` + runtime detection), not `implementation`.
3. **Introduce a WorldForge-owned abstraction** for the three animated entities (e.g. `WFAnimatable` interface + `WFAnimatableInstanceCache` wrapper) that the entity classes implement instead of `GeoEntity` directly. This is the step that removes `implements GeoEntity` from `CatBotEntity`, `MagnetCarrierEntity`, `FarmerRobotEntity`.
4. **Move Category B files** (§5, the 8 pure renderer/model/layer classes) into `compat/geckolib`, unchanged internally.
5. **Create a GeckoLib-backed implementation** of the new abstraction inside `compat/geckolib` (wraps `AnimatableInstanceCache`, `AnimationController` registration) — isolates all `registerControllers(...)` signatures there instead of on the entity classes.
6. **Update the 3 client-init classes** to check `FabricLoader.getInstance().isModLoaded("geckolib")` and branch between the GeckoLib renderer (from `compat/geckolib`) and a vanilla fallback renderer (new class, not yet designed — separate phase).
7. **Rework `FarmerRobotEntity`'s keyframe handlers** last — most entangled piece; move the GeckoLib-typed handler methods into the `compat/geckolib` implementation, with the entity exposing plain gameplay hooks (e.g. `onActionSound()`, `onActionParticle()`) that the compat layer calls into.
8. **Re-test with GeckoLib jar removed from `libs/`** to confirm no `NoClassDefFoundError` at mod init or client init.

**Stopping here per instructions — no code has been modified. This is audit-only.**
