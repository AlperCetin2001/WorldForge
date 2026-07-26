# ⚒️ WorldForge

**A large, survival-friendly toolkit mod** built around a WorldEdit-style region editor and an autonomous robot crew, packaged with more than **15 largely independent subsystems** — from procedural building generation to a 6-tier furnace line, a full farming & food chain, 8 new mobs, and a 7-tier crushing machine — all tuned to never break survival balance.

---

## ✨ Highlights

### 🧱 WorldEdit-Style Region Editing
Select a region, copy-paste, fill, break, replace, draw shapes (sphere/cylinder/pyramid), sculpt terrain with a brush — all with a **live hologram preview**, **unlimited undo/redo**, chunk-safe execution, and optional **economy pricing**. Materials are pulled straight from connected **WorkChests**, never your own inventory.

### 🐱 CatBot — Customizable NPC Companion
A GeckoLib-animated companion that can shift between a cat and a humanoid form. Eye color, fur color, gender, and size are all customizable through a live 3D preview GUI. Thanks to **LLM integration**, you can ask it questions in natural language — or even describe a structure in plain text and have it build it for you.

### 🌾 FarmerRobot — Autonomous Farmer
Automatically detects your farm, harvests by crop priority, takes shelter during storms, and hauls produce to a connected chest. Driven by a full state machine, so you can always see exactly what it's doing.

### 🪓 Lumberjack Bot — Autonomous Woodcutter
FarmerRobot's second work mode: instead of crops, it fells every tree in its area — no climbing required, working from the ground up, finishing even the tallest trunks (giant 2x2 jungle trees included) log by log — and hauls the wood to its own chest. Same GUI, Pause/Resume, night shelter, and "Pick Up Robot" mechanics — only the job and the look are different, complete with a dedicated "Logger Bot" appearance (green beret, flannel shirt, tree emblem on the chest).

Two ways to spawn one:
- **Totem:** place a carved pumpkin on top of a log, right-click the pumpkin with any axe.
- **Craftable item:** craft with a log on the bottom, a carved pumpkin in the middle, and any axe on top; right-click any block with the resulting item.

Either method spawns the robot together with its own chest. An optional **Area Selection Tool** (green/red corner markers, gifted on first spawn) lets you box in either the Farmer or the Lumberjack to a fixed rectangle — the robot will never leave it. The `/wfconfig` screen has a dedicated "Lumberjack" tab too: search/chop range, vertical chop radius (for tall trees), and more, all live-adjustable.

### 🏗️ wfbuild — Procedural Structure Engine
Generates full buildings from a single command (or a natural-language description) through a 13-stage pipeline: intent → blueprint → architecture → style → theme → material → palette → decoration → validation → build. **500+ style and decoration variants** — Japanese, Gothic, Cyberpunk, Victorian, and more. Can also generate standalone landscape features like fountains, ponds, bamboo gardens, and statues.

### 🔥 6-Tier Furnace Line
Copper through Netherite, each with its own multi-input/multi-fuel pipeline and scaling cook speed.

### 🧨 Crusher — 7-Tier Crushing Machine
Copper through Netherite (7 tiers total). Crushes ore, stone-family blocks, and select crops into dust/fragments — its own multi-slot input/output logic, separate from the furnace line.

### ⭐ XP Collector & ⚔️ Sword Matrix
Tiered experience-collecting blocks, and a standalone system for merging multiple swords into a single, more powerful weapon.

### ⛏️ Omnitool & Area Mining
An 8-tier multi-tool (Omnitool), plus VeinMiner/VeinAxe/VeinTool for bulk-breaking connected blocks and an area-based Excavator — all with a live boundary preview before you commit.

### 📦 WorkChest, WFVault & MegaChest
Auto-spawning bulk-job WorkChests feed the region editor and robots; **WFVault** is a paginated, searchable, categorized bulk storage vault; **MegaChest** is a buildable L/T/+-shaped multiblock chest (216 slots) with connected textures.

### 🌾 Farming & Food System
**12 new crops**, each a full seed → 8-stage crop block → harvestable product chain — plant on farmland exactly like vanilla wheat, and the FarmerRobot recognizes them automatically for crop-priority harvesting and smart farm detection:

> Tomato · Cucumber · Onion · Lettuce · Corn · Rice · Garlic · Chili Pepper · Bell Pepper · Cabbage · Eggplant · Strawberry

**38+ prepared dishes**, cooked/crafted with plain vanilla crafting-table, furnace, and campfire recipes — no new machine required — across 13 categories: Fast Food · Pizza · Döner · Sandwich · Breakfast · Fries · Chicken · Rice · Pasta · Soups · Salads · 🍰 Desserts · Drinks.

Desserts include Chocolate Cookie, Cake Slice, Apple Pie, Strawberry Pie, and Donut. Drinks (Coffee, Tea, Orange/Apple Juice) are bottled like Milk/Honey Bottle rather than plain 64-stack items. Nutrition and saturation scale with how substantial the dish is — light snacks and desserts sit lowest, sandwiches sit in the middle, burgers and pizza sit high, and full plates (Döner Plate, Lasagna) top the chart — matching vanilla's own food value curve, so nothing here out-values a full vanilla meal for free.

### 🧟 8 New Mobs
Vampire, Half-Zombie, Grave Wight, Bog Drowned, Sand Reaper, Cave Stalker, Night Howler, Ashen Revenant — hand-painted textures built on vanilla mob models, with natural night/biome-based spawning.

### 🧲 Magnet Link & 🖼️ Image Placement
Coordinate-based remote inventory transfer, and pixel-by-pixel rendering of internet images onto maps/item frames.

### 📐 Litematica Support
Import `.litematic` schematics directly and paste them with the mod's own clipboard system.

---

## 🌍 Localization

🇬🇧 English · 🇹🇷 Turkish · 🇩🇪 German · 🇫🇷 French · 🇪🇸 Spanish

Switch instantly in-game with `/wflang`.

---

## 🔧 Admin Control

- **Config GUI** — toggle every subsystem independently
- **Live sync** — settings apply to all clients instantly, no restart
- **Durability tuning** — `/wfdurability` changes tool/furnace durability at runtime
- **Economy integration** — bridges to Vault-compatible economy mods, per-block pricing

---

## 📖 Tutorial

An in-game clickable tutorial book (`/wfbook`) walks through every subsystem step by step.

---

## 📋 Dependencies

| Requires | Version |
|---|---|
| Fabric Loader | ≥ 0.19.3 |
| Fabric API | latest |
| Minecraft | 26.2 |
| Java | ≥ 25 |
| GeckoLib | ≥ 5.5.3 |

**Optional (soft dependency, auto-detected — never required):** JEI, Jade

---

## 📚 Full Documentation

See the mod's [README](https://github.com/AlperCetin2001/WorldForge) for the full command list, config options, and per-module details.

## 🐛 Bug Reports

Found an issue or have a suggestion? Reach us on [GitHub Issues](https://github.com/AlperCetin2001/WorldForge/issues).

---

**Author:** Doshu — [doshu.gamer.gd](https://doshu.gamer.gd)
