# WorldForge — Mimari Dokümantasyonu

> Otomatik olarak proje kaynak ağacı taranarak çıkarılmıştır (289 Java dosyası, `src/main/resources` altındaki veri/varlık dosyaları). Fabric modding framework, Minecraft 26.2, Java 25.

## 1. Genel Bakış

**WorldForge**, hayatta kalma (survival) modunda WorldEdit tarzı blok operasyonları sunan bir Fabric modudur. Tek bir mod olsa da, birbirinden bağımsız çalışabilen ~15 alt sistemden oluşur: seçim/klipboard araçları, otomatik yapı üretimi (wfbuild), NPC botlar (CatBot, FarmerRobot), akıllı fırınlar, kasa/depolama sistemleri, kaynak mantığı (su/lava), ve LLM destekli doğal dil arayüzleri.

- **Mod ID:** `worldforge`
- **Giriş noktası (main):** `com.worldforge.WorldForgeMod` (`ModInitializer`)
- **Client giriş noktaları:** `WFVaultClientInit`, `CatBotClientInit`, `FurnaceClientInit`, `MagnetClientInit`, `FarmerClientInit`, `WFConfigKeyInit`
- **Render:** GeckoLib (entity modelleri için)
- **Mixin:** `worldforge.mixins.json` kayıtlı ama şu an boş (`mixins: []`)

## 2. Başlatma Akışı (`WorldForgeMod.onInitialize`)

Sıra önemlidir; her adım bir sonrakinin varsayımlarını kurar:

1. **Config yükleme** — `EconomyConfig`, `DurabilityConfig`, `CatBotConfig`, `CatAskConfig`, `FeatureConfig` diskten okunur (`config/worldforge/`).
2. **Bridge/monitor kurulumu** — `VaultEconomyBridge.initialize()`, koşullu `ChunkSafetyMonitor`.
3. **Item/blok kayıtları** — selection axe, chest mover, magnet, omnitool, veinminer, grook, vault, mega chest, image viewer, farmer chest/spawn item, 6 katmanlı fırın seti.
4. **Creative tab + attack-block hook kayıtları.**
5. **Komut kayıtları** (`CommandRegistrationCallback`).
6. **Entity bootstrap** — `CatBotEntities`, `FarmerRobotEntities`, `MagnetCarrierEntities` (attribute + entity type tanımları, spawn değil).
7. **Zamanlayıcı/undo/network kayıtları** — `TickedTaskScheduler`, `UndoManager`, vault-search / image-placement / config-sync payload'ları.
8. **Temizlik hook'ları** (`registerVisualCleanupHooks`) — disconnect, server-stop ve chunk-load anlarında hayalet hologram/parçacık/entity temizliği için üç ayrı güvenlik katmanı.

## 3. Paket Haritası

```
com.worldforge
├── WorldForgeMod.java          # Ana giriş noktası
├── farmer/                     # Farmer Robot (aktif geliştirme alanı)
├── catbot/                     # CatBot NPC + LLM entegrasyonu
├── wfbuild/                    # Prosedürel bina/yapı üretim motoru
├── furnace/                    # 6 katmanlı özel fırın sistemi
├── vault/                      # WFVault — aranabilir/sayfalı depolama
├── chest/                      # MegaChest, chest mover, work chest
├── resourcelogic/              # Su/lava/genel blok kuralları (JSON tabanlı)
├── builder/op/                 # WorldEdit-tarzı düşük seviye blok operasyonları
├── tool/                       # Omnitool, VeinMiner, Grook aletleri + dayanıklılık
├── magnet/                     # Magnet Link — uzak envanter transferi
├── image/                      # İnternet görseli → map-art/item-frame yerleştirme
├── schematic/                  # Litematica okuma, şematik indirme/saklama
├── selection/, clipboard/      # Bölge seçimi ve kopyala-yapıştır
├── undo/                       # Blok değişikliği geri alma
├── veinminer/, grook/          # Alan kırma etkileşim işleyicileri
├── destruction/                # Alet-blok eşleştirme, dayanıklılık tüketimi
├── config/                     # FeatureConfig (admin GUI + network senkron)
├── economy/                    # Ekonomi köprüsü (vault ile)
├── hologram/, particle/        # Önizleme ghost-blok ve sınır efektleri
├── placement/                  # Tick tabanlı görev zamanlayıcı
├── permission/                 # Admin izin kontrolü
├── i18n/                       # 5 dilli çeviri erişimi (Lang.get)
├── command/                    # Genel komut kayıtları
├── chunk/                      # Chunk güvenlik izleyici
├── brush/, fill/, shape/, terrain/, fluid/  # Yardımcı blok araçları
├── book/                       # Öğretici kitap (WFTutorialBook)
└── util/                       # ItemDropMerger vb. genel yardımcılar
```

## 4. Alt Sistemler

### 4.1 Farmer Robot (`com.worldforge.farmer`) — *aktif geliştirme*

Fabric `PathfinderMob` + GeckoLib tabanlı bir tarım robotu. `FarmerChestBlock`/`FarmerChestBlockEntity`'ye bağlanır, ekin toplar ve depolar.

**Davranış = `Goal` yığını** (`FarmerRobotEntity.registerGoals()`, öncelik sırasıyla):

| # | Goal | Faz | Amaç |
|---|------|-----|------|
| 0 | `FloatGoal` | vanilla | Suda yüzme |
| 1 | `FarmerNightShelterGoal` | 12 | Gece/fırtınada kasaya sığınma |
| 2 | `FarmerRestGoal` | 14 | N hasattan sonra dinlenme |
| 3 | `FarmerHarvestGoal` | — | Ekin toplama (ana iş) |
| 4 | `FarmerReturnToChestGoal` | — | Kasaya dönüş |
| 5 | `FarmerWanderFromChestGoal` | — | Kasa civarında dolaşma |
| 6 | `FarmerDepositGoal` | — | Kasaya ürün bırakma |
| 7 | `FarmerLookAroundGoal` | — | Boşta bakınma animasyonu |

Durum makinesi `FarmerRobotStatus` enum'unda tutulur (GUI'nin "Status" satırı): `WORKING, IDLE, WAITING_FOR_SEEDS, CHEST_MISSING, NO_CROPS, SHELTERING, CHEST_FULL, RESTING`. Enum değerleri **sıra korunarak** eklenmiş (ordinal stabilitesi için ipucu — bu proje serileştirmede ordinal kullanıyor olabilir, dikkatli genişletilmeli).

Diğer parçalar: `FarmerCrops`/`FarmerCropPriority` (hangi ekin öncelikli), `FarmerFarmDetector` (tarla algılama), `FarmerRobotMenu`/`Menus` (GUI), `FarmerRobotSpawnItem`. İstemci tarafı: `farmer/client/` altında GeckoLib model/renderer + `FarmerRobotScreen`.

### 4.2 CatBot (`com.worldforge.catbot`)

Genel amaçlı arkadaş NPC. `catbot/build/` alt paketi dikkat çekici: **`CatBuildLlmService`**, `/wfcat ask` ile aynı LLM endpoint'ini kullanarak serbest metin açıklamasından özel yapı üretir (`BuildRequestParser`, `BuildSpec`, `StructureTemplates`). `catbot/goal/` altında `SmartNavigator`, `ApproachOwnerGoal`, `ReturnToOwnerGoal`, `WorkSiteWanderGoal` bulunur — Farmer'daki goal deseniyle paralel bir yapı.

### 4.3 wfbuild — Prosedürel Yapı Motoru

Modun en büyük ve en karmaşık alt sistemi (~75+ dosya), niyet-tabanlı bina üretimi yapar:

- **`intent/`** — Doğal dil/anahtar kelime girdisini yapılandırılmış isteğe çevirir: `IntentParser` arayüzünün iki implementasyonu, `KeywordIntentParser` (kural tabanlı) ve `LlmIntentParser` (LLM tabanlı); `BuildIntent`, `RoofType`, `Size` veri sınıfları. `intent/semantics/` JSON tanım dosyalarını (`SemanticDictionaries`, `PresetDefinition`) yükler.
- **`plan/`** — `ConstructionPlan`, fazlara bölünmüş (`ConstructionPhase`) bir işlem listesi (`ConstructionOperation`) olarak temsil edilir; `StructureRules`/`DefaultStructureRules` geçerlilik kurallarını tanımlar.
- **`architecture/`** — Somut inşa mantığı: `FoundationBuilder`, `FrameBuilder`, `WallsOperation`, `RoofBuilder`, `ColumnBuilder`, `StairBuilder`, `BalconyBuilder`, `OpeningCarver` (kapı/pencere boşlukları), `DecorationBuilder`. `architecture/decoration/` alt paketi tema bazlı dekorasyon varyantlarını (`DecorationVariantRegistry`) JSON'dan yükler.
- **`style/`** — Malzeme temaları: `MaterialThemeRegistry`, `MaterialThemeLoader`, `StyleEngine`, `BiomeStyleModifier`, `MaterialPalette`. `data/worldforge/wfbuild/styles/` altında **~400 stil JSON dosyası** (ör. `ashen_spire.json`, `azure_boathouse.json`) ve `decorations/` altında ~35 kültürel/estetik tema (japanese, gothic, cyberpunk, victorian, vb.).
- **`palette/`** — `PaletteEngine`, `Palette`, `PlacementContext`, `PaletteCategory` — hangi blok türünün nerede kullanılacağını çözer.
- **`blueprint/`** — Soyut bina planı: `Blueprint`, `RoomPlanner`, `DimensionPlanner`, `RoofPlanner`, `DecorationPlanner`, `BoxRegion`, `Point3I`, `RoomConnection`.
- **`feature/`** — Bağımsız peyzaj öğeleri üreticileri: çeşme, havuz, koi göleti, bambu bahçesi, heykel, çit, lamba direği, kamp ateşi alanı, çiçek yatağı, kuyu, bahçe, yol (`FeatureGeneratorRegistry` üzerinden kayıtlı).
- **`validation/`** — `ConstructionValidationEngine` (yerleşim öncesi kontrol).

Bu alt sistem `catbot/build/CatBuildCommand` (`/wfbuild`) üzerinden ve LLM tabanlı paralel yol (`CatBuildLlmService`) ile tetikleniyor. Tam 13 aşamalı boru hattı (Intent → Blueprint → Architecture → Style → Theme → Material Resolver → Palette → Decoration Resolver → Decoration Variants → Resource Logic → Validation → Planner → Builder) ve her aşamanın gerekçesi için bkz. `docs/architecture/wfbuild.md` ve `docs/architecture/design-decisions.md`.

### 4.4 Furnace — 6 Katmanlı Fırın Sistemi

`copperfurnace-faz3` adlı bağımsız bir projeden entegre edilmiş (kod içi yorum). Vanilla `AbstractFurnaceBlock`'u genişleterek her metal seviyesi için ayrı blok/entity/menü/ekran üçlüsü sağlar:

- Malzemeler: Copper, Iron, Gold, Diamond, Emerald, Netherite — her biri kendi `*FurnaceBlock`, `*FurnaceBlockEntity`, `*FurnaceMenu`, `*FurnaceScreen` sınıfına sahip.
- Ortak kayıt noktaları: `FurnaceBlocks`, `FurnaceBlockEntities`, `FurnaceMenus`.
- Çoklu giriş/yakıt hattı sunduğu belirtiliyor (vanilla tek-slot modelinin ötesinde).

### 4.5 Vault & Chest Sistemleri

- **`vault/`** — `WFVaultBlockEntity` + `MultiVaultContainer` + `PagedVaultView` + `VaultSorter` + `ItemCategory`: sayfalanmış, aranabilir, kategorize edilmiş büyük depolama. Arama, C2S `VaultSearchJumpPayload` ile sunucuya gönderilip menüde ilgili sayfaya atlanıyor.
- **`chest/`** — `MegaChestBlockEntity` (216 slot, craftlanabilir), `ChestMoverItem` (kasa taşıma), `WorkChestManager`.

### 4.6 Resource Logic & Builder Op

- **`resourcelogic/`** — Su/lava ve genel blok davranışlarını **JSON tanımlı kurallarla** yönetir (`ResourceRule`, `GenericBlockRule`, `WaterRule`, `LavaRule`, `ResourceRuleRegistry`, `ResourceOperationHandler`). `resourcelogic/json/` alt paketi (`RuleJsonLoader`, `RawRuleDefinition`, `ResolvedRuleDefinition`) veri odaklı kural yükleme sağlıyor — `data/worldforge/resourcelogic/` içindeki JSON dosyalarından beslenir.
- **`builder/op/`** — Düşük seviyeli, tek adımlık blok operasyonları (WorldEdit tarzı): `PlaceBlockOperation`, `PlaceDoorOperation`, `RotateStairOperation`, `FillWaterOperation`, `CreateInfiniteWaterOperation`, `MergeChestOperation`. `BuilderOperationRegistry`/`Bootstrap` ile kayıt.

### 4.7 Diğer Araç ve Yardımcı Sistemler

- **`tool/`** — `OmnitoolRegistry`/`OmnitoolMaterial` (çok işlevli araç), `VeinMinerRegistry`/`Item`/`Material` (bağlı blokları toplu kırma), `GrookItem` (sneak+kırma ile yaprak temizleme), `DurabilityConfig`/`OmnitoolRepairSystem`.
- **`magnet/`** — `MagnetItem` + `MagnetLinkManager`: iki nokta arası envanter/kaynak bağlantısı; `MagnetCarrierEntity` bunu görsel olarak taşıyan kozmetik bir kurye entity.
- **`image/`** — URL'den görsel indirip (`ServerImageDownloader`, ana thread'i bloklamadan async) haritaya/item-frame'lere basma (`MapArtPlacer`); istemci tarafı önizleme `WFImageViewerScreen`.
- **`schematic/`** — Litematica şematiklerini okuma (`LitematicaReader`), indirme, saklama.
- **`selection/`, `clipboard/`, `undo/`** — Bölge seçme, kopyala/yapıştır, blok değişikliklerini geri alma (`BlockChangeRecord`).
- **`veinminer/`, `grook/`** — Sol-tık alan kırma etkileşim işleyicileri + parçacık sınır render (`VeinMinerOutlineRenderer`, `particle/RegionParticleRenderer`).
- **`destruction/`** — Hangi aletin hangi bloğu kırabileceği ve dayanıklılık tüketimi mantığı.
- **`hologram/`** — Yerleştirme öncesi "hayalet blok" önizlemesi (`HologramSender`, `BlueprintStore`); bkz. §5 temizlik mekanizması.
- **`economy/`** — `EconomyConfig` + `VaultEconomyBridge` (muhtemelen üçüncü parti ekonomi modlarıyla köprü).
- **`config/`** — `FeatureConfig`/`FeatureConfigSchema`: admin'in özellik aç/kapa yapabildiği merkezi config; `config/client/WFConfigScreen` + `config/network/` üç payload'lu (Request/Set/Sync) canlı senkronizasyon protokolü sağlar (bkz. §6).
- **`i18n/`** — `Lang.get(player, key, args...)` ile 5 dilde (en_us, de_de, es_es, fr_fr, tr_tr) merkezi çeviri erişimi.

## 5. Ghost-Entity Temizlik Deseni

`WorldForgeMod.registerVisualCleanupHooks()` üç katmanlı bir savunma uyguluyor — bu, projenin genelinde tekrar eden bir güvenlik deseni olarak not edilmeye değer:

1. **Disconnect anı** — oyuncu ayrılınca hologram/parçacık/bekleyen-yüz state'i temizlenir.
2. **Server durma anı** — tüm hologramlar/parçacıklar/dev-alet efektleri global olarak temizlenir.
3. **Chunk yükleme anı** (üçüncü güvenlik ağı) — çökme/`kill -9`/elektrik kesintisi gibi `SERVER_STOPPING`'i atlayan senaryolarda, kalıcı olarak kaydedilmiş bir "hayalet" entity, chunk'ı her yüklendiğinde `HologramSender.isGhost()` etiketiyle tanınıp `entity.discard()` ile silinir.

## 6. Ağ Protokolleri (Client ↔ Server)

Üç ayrı payload seti, hepsi `WorldForgeMod` içinde kayıtlı:

| Payload | Yön | Amaç |
|---|---|---|
| `VaultSearchJumpPayload` | C2S | Vault arama kutusu → sunucu ilgili sayfaya atlar |
| `ImagePlacementPayload` | C2S | Görsel yerleştirme isteği → async indirme → map-art |
| `WFConfigRequestPayload` | C2S | Config GUI açılışında anlık görüntü isteği |
| `WFConfigSetPayload` | C2S | Tek bir config değeri değişikliği (admin izinli); `_reset`/`_reload` özel modülleri |
| `WFConfigSyncPayload` | S2C | Güncel config JSON'ı; her başarılı `Set`'te ekranı açık olan **tüm** oyunculara broadcast edilir |

## 7. Veri/Varlık Yapısı (`src/main/resources`)

```
assets/worldforge/
├── lang/            # en_us, de_de, es_es, fr_fr, tr_tr — 5 dil, tam paralel
├── geckolib/        # Entity animasyon dosyaları
├── models/, textures/, items/, blockstates/

data/worldforge/
├── wfbuild/
│   ├── styles/          # ~400 malzeme-teması JSON (renk+isim kombinasyonları, ör. azure_boathouse)
│   ├── decorations/     # ~35 kültürel/estetik dekorasyon teması (japanese, gothic, cyberpunk...)
│   ├── material_themes/ # haunted, volcanic, gothic override'ları
│   ├── semantics/       # intent-parsing sözlükleri (presets, structure, size, constraint, decoration, roof, style)
│   └── decoration_themes.json
├── resourcelogic/   # Su/lava/genel blok kuralları JSON'ları
├── recipe/, loot_table/, advancement/, enchantment/, tags/
```

**Not:** `styles/` klasöründeki ~400 dosya, bir "sıfat + yapı-türü" kombinatoryel üretimi izliyor (ashen/azure/blazing/bronze/celestial/charred/amber/autumn + spire/boathouse/tavern/keep/...). Bu muhtemelen elle değil bir üretim scripti/şablonuyla oluşturulmuş; yeni bir stil eklerken bu deseni takip etmek tutarlılık sağlar.

## 8. Geliştirme Konvansiyonları (proje genelinde gözlemlenen)

- Her yeni davranış bir **fazla** (Phase N) numaralandırılıp Javadoc'ta çapraz referanslanıyor (bkz. `FarmerRobotStatus`, `FarmerRobotEntity`).
- Enum'lara yeni değerler **sona ekleniyor**, ordinal stabilitesi korunuyor.
- JSON-tanımlı kural/tema sistemleri (`resourcelogic`, `wfbuild/style`, `wfbuild/architecture/decoration`) hepsi aynı üçlü desenle: `Raw*Definition` (ham JSON DTO) → `*Loader` (parse) → `*Registry` (çalışma zamanı erişim).
- Ağ paketleri her zaman üçlü: Request (C2S) / Set veya Action (C2S) / Sync (S2C, broadcast).
- Her kullanıcıya görünür metin `i18n.Lang.get()` üzerinden 5 dilde sağlanıyor.
- Async I/O (görsel indirme, LLM çağrıları) `CompletableFuture` + `server.execute()` ile ana thread'e güvenli dönüş deseniyle yapılıyor.

## 9. Genişletme Sırasında Dikkat Edilecekler

- **Farmer Robot** üzerinde çalışırken: yeni bir `Goal` eklerken `goalSelector.addGoal()` içindeki öncelik numarasını mevcut listeyle çakışmayacak şekilde seç; yeni bir durum gerekiyorsa `FarmerRobotStatus`'a **sona ekle**, GUI/lang dosyalarını 5 dilde güncelle.
- **wfbuild** stiline yeni tema eklerken mevcut `sıfat_yapıTürü.json` adlandırma desenini ve JSON şemasını (`RawStyleDefinition`) koru.
- Yeni bir ağ paketi eklerken 3-payload (Request/Set/Sync) desenini takip et ve `PayloadTypeRegistry` kaydını `WorldForgeMod`'daki ilgili `register*Networking()` metoduna ekle.
