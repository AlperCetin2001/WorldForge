# WorldForge — Özellik Bazlı Çekirdek Mimari Dokümanları

Bu klasördeki her dosya, tek bir alt sistemin **çekirdek mimarisini** (sınıf sorumlulukları, tasarım kararları, veri akışı, genişletme kuralları) derinlemesine anlatır. Genel/üst seviye bakış için üst klasördeki `docs/ARCHITECTURE.md`'ye bakın.

| Dosya | Kapsam |
|---|---|
| [`farmer-robot.md`](farmer-robot.md) | Farmer Robot — Goal yığını, durum makinesi, tarla algılama, kasa entegrasyonu *(aktif geliştirme)* |
| [`catbot.md`](catbot.md) | CatBot NPC — sahiplik, goal'ler, LLM entegrasyonu (`/wfcat ask`, `/wfbuild` LLM yolu) |
| [`wfbuild.md`](wfbuild.md) | Prosedürel yapı motoru — 13 aşamalı Intent→Blueprint→Architecture→Style→Theme→Material Resolver→Palette→Decoration Resolver→Decoration Variants→Resource Logic→Validation→Planner→Builder zinciri |
| [`design-decisions.md`](design-decisions.md) | Proje genelindeki kilit mimari kararların **neden**leri (Theme/Style ilişkisi, LLM'in opsiyonelliği, JSON kalıtımı, overlay mantığı, veri-odaklı resource logic) |
| [`furnace.md`](furnace.md) | 6 katmanlı özel fırın sistemi (Copper→Netherite), çoklu giriş/yakıt hattı |
| [`vault-and-chest.md`](vault-and-chest.md) | WFVault, MegaChest, WorkChestManager çoklu-kasa orkestrasyonu |
| [`resourcelogic-and-builder-op.md`](resourcelogic-and-builder-op.md) | JSON-tanımlı blok kuralları (`extends` kalıtımı) + düşük seviye blok operasyonları |
| [`tools-and-mining.md`](tools-and-mining.md) | Omnitool, VeinMiner, Grook, Destruction — alet/alan-kırma sistemleri |
| [`magnet-link.md`](magnet-link.md) | Magnet Link — uzak envanter transferi, koordinat-tabanlı (canlı referanssız) bağlantılar |
| [`image-placement.md`](image-placement.md) | İnternet görseli → map-art yerleştirme, async indirme deseni |
| [`schematic-system.md`](schematic-system.md) | Litematica okuma/indirme/saklama, `ClipboardData` köprüsü |
| [`selection-clipboard-undo.md`](selection-clipboard-undo.md) | Bölge seçimi, kopyala-yapıştır, geri alma + ekonomi iadesi |
| [`job-engine.md`](job-engine.md) | **TickedTaskScheduler** — modun merkezi iş kuyruğu orkestratörü, chunk güvenliği, hologram önizleme |
| [`config-and-networking.md`](config-and-networking.md) | FeatureConfig switchboard + Request/Set/Sync ağ protokol deseni |
| [`economy.md`](economy.md) | Vault-uyumlu ekonomi köprüsü, reflection ile soft-dependency |
| [`worldedit-tools-and-commands.md`](worldedit-tools-and-commands.md) | Brush, Shape, Terrain, BlockPattern + tam komut yüzeyi envanteri |

## Alt Sistemler Arası Bağımlılık Haritası (özet)

```
command/WorldForgeCommands ─┬─► selection, clipboard, shape, fill, terrain, brush
                             ├─► placement.TickedTaskScheduler (ana iş kuyruğu)
                             ├─► schematic, undo
                             └─► wfbuild (CatBuildCommand üzerinden), catbot

placement.TickedTaskScheduler ─┬─► destruction, tool (repair), fluid
                                ├─► chunk (güvenlik), hologram, particle, util (drop merge)
                                ├─► undo, economy
                                └─► catbot (CatBot state güncellemeleri)

wfbuild (7 katman) ─► resourcelogic (validation), builder/op (düşük seviye), catbot/build (LLM yol)

vault/chest ─► farmer, tool (omnitool repair), placement (WorkChest malzeme kaynağı)

Tüm kullanıcıya-görünür metin ─► i18n.Lang (5 dil)
Tüm config ─► config.FeatureConfig (+ ayrı: Economy/Durability/CatBot/CatAsk config'leri)
```
