# Changelog

All notable changes to WorldForge are documented in this file.

## [1.0] - Unreleased

### Added
- **Lumberjack Bot (`farmer` package, new `FarmerRobotEntity.WorkMode`):** a
  second job for the existing Farmer Robot infrastructure — build a log
  block with a pumpkin on top, right-click the pumpkin with any axe, and a
  robot spawns with its own chest (`LumberjackSpawnListener`) set to
  `WorkMode.LUMBERJACK`. `LumberjackChopGoal` reuses the exact same
  carried-item buffer, chest-deposit, wander/return/night-shelter/rest, and
  GUI machinery as the farming job — it just chops the nearest log in its
  patch instead of harvesting crops, walking itself across an entire
  connected tree one log at a time until none are left in range.
- **Lumberjack Robot Spawn Item (`LumberjackRobotSpawnItem`, `worldforge:lumberjack_robot_spawn`):**
  a craftable second way to get a Lumberjack Bot, mirroring
  `FarmerRobotSpawnItem`'s "right-click a block" flow instead of the totem.
  Recipe: log on the bottom, carved pumpkin in the middle, any axe (`#minecraft:axes`)
  on top. Right-clicking with it places a fresh chest and links a robot set to
  `WorkMode.LUMBERJACK`, or re-links to an already-placed orphaned chest, same
  as the Farmer Robot item. Picking a robot back up with "Pick Up Robot" and
  re-placing it preserves its work mode either way.
- **Logger Bot look (`farmer_robot_logger.png` + glowmask):** Lumberjack Bots
  now render with their own baked texture — green beanie, plaid shirt,
  tree-emblem chest plate, gray metal head with green LED eyes — instead of
  reusing the default Farmer Robot skin. `FarmerRobotGeoModel` now carries the
  robot's work mode into `GeoRenderState` (`IS_LUMBERJACK` data ticket) and
  picks the texture accordingly; the Farming work mode's texture and the
  existing color-variant tinting are untouched.
- **Jade compat (`compat.jade`):** hovering any tier of XP Collector with
  [Jade](https://modrinth.com/mod/jade) installed now shows its tier,
  stored/capacity XP, and collection radius directly in the tooltip
  overlay. Pure soft dependency — registered via the `"jade"` fabric.mod.json
  entrypoint, Jade API is `compileOnly` and never bundled, and nothing
  breaks if the player doesn't have Jade installed. Requires
  `libs/Jade-mc26_2-Fabric-26_2_9.jar` to compile (see build.gradle warning
  if missing).

### Fixed
- **MegaChest renderer / 26.2 port:** ported `MegaChestRenderer` and the
  MegaChest open/close logic to Minecraft 26.2's reworked APIs —
  `BlockEntityRenderState` package move, `TextureAtlasSprite` package move,
  `TextureAtlas.getSprite(Identifier)` (replacing the removed
  `ModelManager.atlasManager()`), `RenderTypes.entityCutout(Identifier)`
  (replacing the removed `RenderType.entityCutoutNoCull(...)`),
  `BlockEntityRenderState.extractBase(...)` for lighting (replacing the
  removed `LevelRenderer.getLightColor(...)`), and
  `Container.startOpen/stopOpen(ContainerUser)` (replacing the old
  `Player`-typed signature). Also removed `getRenderBoundingBox()`, which no
  longer exists on `BlockEntity` in 26.2. See README → *Known Issues* for two
  remaining assumptions that still need a clean `gradlew build` to confirm.
- **Recipe edit GUI:** the 3×3 ingredient grid's text fields were wider than
  the spacing between them, so adjacent boxes overlapped. Item ID text was
  unreadable and clicks often landed on the wrong cell, making some slots
  impossible to edit. Field width now matches the grid spacing.

### Changed
- Aligned `mod_version` with the README's documented release version.
- Synced documented Fabric API / Loom versions with the versions actually
  pinned in `gradle.properties`.

---

Older history predates this file; see commit history once the repository is
published.
