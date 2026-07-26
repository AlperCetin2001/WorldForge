# WorldForge v1.0

**Author:** Doshu  
**Website:** https://github.com/AlperCetin2001/WorldForge  
**Target:** Minecraft 26.2 · Fabric Loader 0.19.3 · Java 25  
**Fabric API:** 0.154.0+26.2 · Loom 1.17.13

---

## Overview

WorldForge is a large survival-friendly toolkit mod built around a WorldEdit-style region editor (select, fill, break — with full economy integration, pause/resume, chunk safety, and a WorkChest system), plus a wide set of standalone feature modules layered on top.

---

## Feature Overview

| Module | What it does |
|--------|--------------|
| **Region Editing** | `/wfpos1`/`/wfpos2` selection, `/wffill`, `/wfbreak`, `/wfclear`, undo, blueprint pause/resume, chunk-safe, economy-priced |
| **WorkChest / WFVault / MegaChest** | Auto-spawning chests for bulk jobs; searchable multi-page WFVault storage; multiblock MegaChest (L/T/+ footprints, one shared lid-opening animation across the whole structure) for large-volume drops |
| **CatBot** | GeckoLib-animated companion with cat/humanoid form switching, `/wfcat gui` customization (form, gender, hair, size, live 3D preview), cooldown-based respawn |
| **FarmerRobot** | Autonomous crop harvesting: smart farm detection (BFS flood-fill), incremental farm-cache updates, seed reserve management, crop priority, day/night & weather-aware behavior, optional Selection Area work-area boundary. Also doubles as the **Lumberjack Bot** work mode — clears whole trees (including tall jungle giants) from the ground without climbing, with its own "Logger Bot" look |
| **Farming & Food** | 12 new farmable crops (Tomato, Cucumber, Onion, Lettuce, Corn, Rice, Garlic, Chili Pepper, Bell Pepper, Cabbage, Eggplant, Strawberry), each with its own seed item, 8-stage crop block, and harvested product; 38+ prepared dishes across Fast Food, Pizza, Döner, Sandwich, Breakfast, Fries, Chicken, Rice, Pasta, Soups, Salads, **Desserts** (Chocolate Cookie, Cake Slice, Apple/Strawberry Pie, Donut), and Drinks (Coffee, Tea, Orange/Apple Juice) |
| **wfbuild** | Procedural structure/building generator with 500+ decoration, style, and structure-type variants (data-driven JSON), live ghost-block hologram previews during construction |
| **Furnaces** | 6-tier furnace line (Copper → Netherite) with per-tier cooking speed multipliers and multi-lane GUIs |
| **VeinMiner / VeinAxe / VeinTool / Excavator / Omnitool** | Tiered mining tools (Copper → Netherite/Emerald) with vein-following and area-mining, live line-outline previews before breaking |
| **Grook** | Leaf/foliage clearing tool with configurable reach and max-leaves cap |
| **Magnet Link** | Remote item transfer between chests/players |
| **Image Placement** | Downloads an internet image and places it as in-world map art |
| **Litematica Support** | Import/export schematics compatible with Litematica |
| **Tutorial Book** | Clickable, categorized in-game command reference (`/wf...` commands) with bilingual hover text |
| **Config GUI** | In-game key-bound config screen; every optional module toggleable/tunable live via `/wfconfig`, persisted to `worldforge_features.json` |
| **Localization** | English, Turkish, German, French, Spanish |

---

## Commands

| Command | Description |
|---------|-------------|
| `/wfpos1` | Set Pos1 at the **block you're looking at** (raycast) |
| `/wfpos2` | Set Pos2 at the **block you're looking at** |
| `/wfpos1 <x> <y> <z>` | Set Pos1 at explicit coordinates |
| `/wfpos2 <x> <y> <z>` | Set Pos2 at explicit coordinates |
| `/wfselection` | Show current selection info |
| `/wfclear` | Clear all blocks in selection (fill with air, no drops) |
| `/wffill <block>` | Fill selection with a block (uses **WorkChest**) |
| `/wfbreak` | **Break** all blocks in selection (drops go to **WorkChest**) |
| `/wfcancel` | Cancel active job (saves blueprint) |
| `/wfresume` | Resume from saved blueprint |
| `/wfblueprint info` | View saved blueprint status |
| `/wfblueprint clear` | Delete saved blueprint |
| `/wffluid preserve\|dry` | Fluid handling mode |
| `/wfchest` | Show WorkChest location |
| `/wfchest remove` | Remove WorkChest (contents dropped) |
| `/wflang <code>` | Set language: `en` `tr` `de` `fr` `es` |
| `/wfeconomy ...` | Economy admin commands |
| `/wfconfig list` | Show current settings for every optional module |
| `/wfconfig set <module> <key> <value>` | Tune or toggle a module (see below) |
| `/wfconfig reload` \| `save` | Reload/save the feature config file |

---

## WorkChest System

When you run `/wffill` or `/wfbreak`, a **chest automatically spawns** near you in a safe location.

### Fill mode (`/wffill <block>`)
- Place the blocks you want to fill with **in the WorkChest** (in order).
- The mod consumes blocks from the chest instead of your inventory.
- If the chest runs out, the job **pauses** — restock and `/wfresume`.
- If the target position already has a block, it is **broken first** using a tool from your inventory (or WorkChest), and the drops go back to the WorkChest.

### Break mode (`/wfbreak`)
- Place **tools** (pickaxe, shovel, axe, etc.) in the WorkChest.
- The mod uses the appropriate tool for each block type.
- **All drops go into the WorkChest** (overflow drops to the world).
- If a tool breaks, the job pauses — replace it and `/wfresume`.

### Important
- Do NOT break the WorkChest while a job is running — the job will pause.
- Use `/wfchest remove` to safely remove it when done.

---

## Selection

- Left-click with the **Selection Axe** → Pos1 (the block you hit)
- Right-click with the **Selection Axe** → Pos2 (the block you click)
- `/wfpos1` and `/wfpos2` also target the **block you're looking at** (not your feet)

Once both corners are selected, a **particle wireframe outline** appears around the region.  
The outline disappears automatically when the job completes or is cancelled.

---

## Languages

Set your language with `/wflang <code>`:

| Code | Language |
|------|----------|
| `en` | English (default) |
| `tr` | Türkçe |
| `de` | Deutsch |
| `fr` | Français |
| `es` | Español |

---

## Pause / Resume / Cancel

The system auto-pauses when:
- **Tool broke** (break mode or clear mode)
- **WorkChest empty** (fill mode or break mode)
- **Chunks unloaded** (move closer to the region)
- **Player disconnected**

Remaining blocks are saved as a **blueprint**. Use `/wfresume` to continue.  
Use `/wfcancel` to cancel and save a blueprint, or `/wfblueprint clear` to discard.

---

## Economy

Optional economy integration. Operators can configure:

```
/wfeconomy toggle
/wfeconomy set cost <amount>
/wfeconomy set currency <name>
/wfeconomy reload
/wfeconomy save
```

---

## Farmer Robot

- **Day:** works normally — harvests, replants, and deposits into its Farmer Chest.
- **Night:** returns to its chest and idles beside it, then resumes automatically the next morning.
- **Rain:** keeps working right through it.
- **Thunderstorm:** pauses and waits beside the chest until it passes.
- **Full chest:** pauses harvesting the moment its Farmer Chest has no room left — nothing is ever destroyed or dropped — and resumes automatically as soon as space frees up.

This behavior can be turned off with `/wfconfig set farmer daynightenabled false` (see Feature Config below), which restores the old day-and-weather-agnostic robot. The full-chest pause is not tied to that switch — it always applies.

### Lumberjack Bot

A second job for the same robot, chest, and GUI infrastructure — chops nearby trees instead of harvesting crops, walking itself across an entire connected tree one log at a time until none are left in range.

Two ways to spawn one:

1. **Totem:** place a pumpkin on top of any log, then right-click the pumpkin with any axe. Both blocks are cleared and replaced by a fresh chest + robot.
2. **Spawn item:** craft a Lumberjack Robot Spawn Item — log on the bottom, carved pumpkin in the middle, any axe on top — then right-click any block with it. Works the same as the Farmer Robot Spawn Item (re-links to an orphaned chest if one already exists at the clicked position).

Either way, the robot comes with its own chest and gets a distinct "Logger Bot" appearance (green beanie, plaid shirt, tree-emblem chest plate) instead of the default Farmer Robot look — everything else (GUI, Pause/Resume, day/night shelter, Pick Up Robot, Selection Area work-area boundary) is identical to the Farmer Robot.

Chops a whole tree from the ground without climbing: the chop-reach check is horizontal-only (distance to the trunk column), with a separate, configurable vertical allowance (`lumberjackVerticalChopRadius`, default 50) — so the robot clears an entire trunk, including the tallest 4-sapling jungle-style giants, log by log from where it's standing at the base.

### Work Area (Selection Area Tool 1/2)

Both the Farmer and Lumberjack Bot support an optional rectangular work-area boundary, on top of their default chest-centered range. The first robot a player spawns comes with a green (corner 1) and red (corner 2) Selection Area Tool — right-click-place each one at the corners you want, and that robot (Farmer or Lumberjack) never harvests/chops outside the marked rectangle. The boundary is X/Z only (Y is unconstrained, for multi-level farms/tall trees). No area marked — or only one corner — falls straight back to the old chest-centered behavior automatically. More tools can be requested from the robot's own GUI.

---

## Farming & Food System

### New Crops

12 new farmable crops, each a full seed → 8-stage crop block → harvestable product chain (plant on farmland exactly like vanilla wheat):

Tomato, Cucumber, Onion, Lettuce, Corn, Rice, Garlic, Chili Pepper, Bell Pepper, Cabbage, Eggplant, Strawberry.

The FarmerRobot's crop-priority and smart farm detection recognize these alongside vanilla crops out of the box.

### Prepared Food

38+ cookable/craftable dishes (plain vanilla crafting-table/furnace/campfire recipes — no new machine required), grouped by category:

| Category | Examples |
|---|---|
| Fast Food | Hamburger, Cheeseburger, Double Cheeseburger, Chicken/Fish Burger, Hot Dog |
| Pizza | Cheese, Pepperoni, Chicken, Vegetable, Supreme (plus a Pizza Dough → Raw Pizza crafting stage) |
| Döner | Chicken Döner, Meat Döner, Döner Wrap, Döner Plate |
| Sandwich | Sandwich, Chicken/Steak/Cheese Sandwich |
| Breakfast | Fried/Boiled Egg, Omelette, Toast, Cheese Toast |
| Fries | French Fries, Loaded Fries, Potato Wedges |
| Chicken | Fried Chicken, Chicken Wings/Nuggets, Roasted Chicken |
| Rice | Rice Bowl, Chicken Rice, Vegetable Rice |
| Pasta | Pasta, Spaghetti, Lasagna, Mac and Cheese |
| Soups | Tomato, Vegetable, Chicken, Mushroom Soup |
| Salads | Garden, Chicken, Greek Salad |
| **Desserts** | Chocolate Cookie, Cake Slice, Apple Pie, Strawberry Pie, Donut |
| Drinks | Coffee, Tea, Orange Juice, Apple Juice (bottled, stack-of-16 like Milk/Honey Bottle) |

Nutrition/saturation scale with dish complexity — snacks and desserts sit low (2-3 nutrition), sandwiches mid (5-6), burgers/pizza high (6-9), full plates (Döner Plate, Lasagna) highest (8-10) — matching vanilla's own food value spread.

---

## Feature Config

Every optional module (Farmer Robot, Lumberjack Bot, Magnet, Furnaces, Veinminer, Grook,
internet image placement, WFVault, MegaChest, chunk safety, selection
particle outline) can be toggled or tuned live by an operator, persisted at
`config/worldforge/worldforge_features.json`. Economy and omnitool
durability keep their own separate config files/commands (`/wfeconomy`,
`/wfdurability`).

```
/wfconfig list
/wfconfig set farmer enabled false
/wfconfig set farmer searchradius 12
/wfconfig set farmer harvestdistance 2.5
/wfconfig set farmer rescaninterval 10
/wfconfig set farmer seedreserve 1       # minimum seeds per type always kept for future planting
/wfconfig set farmer fatigueenabled true
/wfconfig set farmer fatiguethreshold 12 # crops harvested in a row before resting
/wfconfig set farmer restticks 200       # how long the robot rests once fatigued
/wfconfig set farmer daynightenabled false  # disable night/thunderstorm shelter-at-chest behavior
/wfconfig set lumberjack searchradius 16
/wfconfig set lumberjack chopdistance 3
/wfconfig set lumberjack rescaninterval 10
/wfconfig set lumberjack verticalchopradius 50   # tallest trunk the robot clears from the ground, no climbing
/wfconfig set lumberjack unreachablefailurethreshold 2
/wfconfig set lumberjack unreachablecooldownticks 6000
/wfconfig set magnet enabled false
/wfconfig set furnace speedstep 2        # 1 = vanilla speed, higher = faster
/wfconfig set veinminer enabled false
/wfconfig set veinminer reach 10
/wfconfig set grook enabled false
/wfconfig set grook maxreach 16
/wfconfig set grook maxleaves 200
/wfconfig set image enabled false
/wfconfig set image maxdownloadmb 10
/wfconfig set image timeoutseconds 15
/wfconfig set vault enabled false
/wfconfig set megachest enabled false
/wfconfig set chunksafety enabled false   # requires a restart to take effect
/wfconfig set particle outlineenabled false
/wfconfig reload
/wfconfig save
```

Disabling a module doesn't remove its blocks/items/entities from the world —
it just turns off that module's behavior (interactions, event handlers, or
AI goals) so nothing already placed gets orphaned or corrupted.

---

## Known Issues (MC 26.2 port, MegaChest)

**Texture continuity (Seçenek A, final):** `buildMesh()` no longer tiles or
wraps the texture per cell at all. The virtual UV canvas passed to
`LayerDefinition.create(...)` is now sized to the *exact* pixel dimensions
of the current MegaChest (`w*16 × h*16`, `d*16 × h*16`), and each block's
`texOffs` is simply `bx*16`/`bz*16`/`by*16` into that canvas — mathematically
identical to `u0=bx/width, u1=(bx+1)/width`. There is no modulo, no
wraparound, no repeat: the actual texture file is stretched exactly once
across the whole structure, however large it is.

The unavoidable tradeoff (a real GPU/asset limitation, not something the
renderer code can work around): if `mega_chest.png` / `mega_chest_frame.png`
stay at their current small resolution, large MegaChests will show a
visibly blurred/pixelated version of the design, since a small texture is
being stretched over a large area. To get a crisp look at scale, redraw
those two textures at a much higher resolution (512×512 or 1024×1024) —
detail density will then scale with structure size instead of blurring.

The MegaChest renderer and its opening logic were ported to 26.2's reworked
rendering/container APIs by cross-referencing `javap` output against the
target `client.jar`, since 26.2 changed several relevant classes (block
entity render-state architecture, `RenderType` factories, texture atlas
access, and the `Container` interface). Everything below compiles against
the confirmed API surface, but two points still rest on an assumption that
hasn't been independently verified against `client.jar` and should be
checked before release:

- `MegaChestBlockEntity#tick` now derives "is anyone looking at this chest"
  from `getEntitiesWithContainerOpen()` instead of the removed `openCount`
  field. This assumes that method is available on the `WFVaultBlockEntity` →
  `BaseContainerBlockEntity` chain, not only on `ChestBlockEntity`.
- `MultiVaultContainer#startOpen/stopOpen` now implement
  `Container.startOpen(ContainerUser)` / `stopOpen(ContainerUser)` (the 26.2
  signature, replacing `Player`). This assumes `Player` implements
  `ContainerUser`.
- `MegaChestBlockEntity#getRenderBoundingBox()` was removed outright (no
  longer present on `BlockEntity` in 26.2); custom multiblock frustum
  culling now relies solely on the distance check in
  `MegaChestRenderer#shouldRender`, so very large structures may get
  culled slightly earlier/later near screen edges than before.

If `./gradlew build` still fails on any of the three points above, that's
the first place to check.

---

## Building

GeckoLib 5.5.3 for MC 26.2 isn't on GeckoLib's Maven yet, so download the
Fabric jar manually (CurseForge or Modrinth, "GeckoLib Fabric 26.2 5.5.3")
and place it at `libs/geckolib-fabric-26.2-5.5.3.jar` before building.

```bash
./gradlew build
```

Requires JDK 25. Output: `build/libs/worldforge.jar` (version lives in
`fabric.mod.json`, not the filename).

---

## Compatibility Evidence

- ✅ **Minecraft 26.2**: Official Fabric blog confirms 26.2 support (Jun 2026)
- ✅ **Fabric Loader 0.19.3**: Confirmed current stable for 26.2 (fabricmc.net/2026/06/15/262.html)
- ✅ **Fabric API 0.154.0+26.2**: Available on CurseForge & Modrinth
- ✅ **Java 25**: Required by MC 26.1+; confirmed by Fabric blog
- ✅ **Loom 1.17**: Required for 26.2 dev environment
- ✅ **Server-only**: No Blaze3D/Vulkan client-rendering code used (safe for 26.2 renderer changes)

---

*WorldForge by Doshu — https://github.com/AlperCetin2001/WorldForge*
