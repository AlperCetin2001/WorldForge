# Changelog

All notable changes to WorldForge are documented in this file.

## [2.0.0] - Unreleased

### Added
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
