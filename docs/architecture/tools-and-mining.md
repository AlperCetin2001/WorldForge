# Çekirdek Mimari — Aletler (Omnitool, VeinMiner, Grook) & Yıkım

Paket: `com.worldforge.tool` (10) · `com.worldforge.veinminer` (3) · `com.worldforge.grook` (2) · `com.worldforge.destruction` (4)

## 1. Ortak Desen: Item + EventHandler İkilisi

Bu alt sistemdeki üç alan-etkileşim aleti (Omnitool istisna, VeinMiner ve Grook) **aynı sorumluluk ayrımını** kullanır:

```
<X>Item extends Item          → sadece item tanımı/tooltip/temel davranış
<X>EventHandler                → asıl alan mantığı (flood-fill, blok kırma, event kaydı)
```

Bu ayrım kod yorumunda açıkça belirtiliyor (`GrookItem` Javadoc'u): *"That flood-fill and the actual breaking live in GrookEventHandler, same split of responsibility as VeinMinerItem / VeinMinerEventHandler."*

## 2. VeinMiner — Alan (NxN) Kırma

```java
VeinMinerItem extends Item
VeinMinerMaterial (enum)        // farklı seviyelerde alet materyali
VeinMinerRegistry               // .register(), .all() — tüm materyal varyantlarını kaydeder
VeinMinerEventHandler           // .register(), clearPendingFace(UUID) — asıl kırma mantığı
VeinMinerAreaUtil                // alan hesaplama yardımcıları
VeinMinerOutlineRenderer         // istemci tarafı sınır/önizleme render'ı
```

`clearPendingFace(UUID)` — `WorldForgeMod.registerVisualCleanupHooks()` tarafından oyuncu disconnect olduğunda çağrılıyor (bkz. ana ARCHITECTURE.md §5), yarım kalmış bir "bekleyen yüz" seçimini temizlemek için.

## 3. Grook — Yaprak Kümesi Temizleme

```java
GrookItem extends Item
GrookRegistry.register()
GrookEventHandler.register()
GrookAreaUtil
```

**Davranış:** Normal sol-tık kırma vanilla gibi davranır (elle yaprak kırma). **Sneak (shift) + sol-tık kırma** asıl özelliktir: kırılan bloğa bağlı her yaprak bloğunu (`BlockTags#LEAVES`, 6 yönlü komşuluk) **flood-fill** ile bulup tek seferde kırar — büyük bir orman tepesinin sunucuyu kilitlememesi için **sınırlandırılmış**.

## 4. Omnitool — Çok Amaçlı Alet

```java
WFOmnitoolItem extends Item
OmnitoolMaterial (enum)
OmnitoolRegistry.register(), .get(OmnitoolMaterial)
OmnitoolRepairSystem
DurabilityConfig.load(configDir)
```

VeinMiner/Grook'tan farklı olarak ayrı bir EventHandler'ı yok görünüyor — muhtemelen doğrudan `Item` override'ları (mineBlock, useOn vb.) üzerinden çalışıyor. `OmnitoolRepairSystem`, `WorkChestManager`'ın tekilleştirilmiş `Container` API'si üzerinden malzeme çekerek tamir yapıyor olabilir (bkz. vault-and-chest.md §2 — "hiçbir değişiklik gerektirmez" listesinde açıkça anılıyor).

`DurabilityConfig` — tüm alet ailelerinin dayanıklılık ayarlarını merkezi olarak diskten yükler (`config/worldforge/`).

## 5. Destruction — Alet-Blok Eşleştirme

```java
DestructionEngine
DurabilityHandler
ToolMatchResult
ToolRequirement
```

Hangi aletin hangi bloğu kırabileceğine ve dayanıklılık tüketimine karar veren merkezi mantık. `WorkChestManager`'ın tekilleştirilmiş `Container`'ını kullanan üç sistemden biri (diğerleri `TickedTaskScheduler`, `OmnitoolRepairSystem`) — yani bu sınıf çoklu-kasa senaryosundan tamamen izole.

## 6. Genişletme Notları

- **Yeni bir alan-etkileşim aleti** eklerken VeinMiner/Grook ikili deseni (Item + EventHandler) izle — item sınıfını ince tut, ağır mantığı EventHandler'a koy.
- Alan/flood-fill mantığı yazarken **her zaman bir üst sınır koy** (VeinMiner ve Grook'un ikisi de bunu yapıyor) — sunucu tick'ini kilitleyecek sınırsız bir flood-fill riskinden kaçın.
- Disconnect/cleanup senaryosu varsa (`clearPendingFace` gibi) `WorldForgeMod.registerVisualCleanupHooks()`'a yeni bir temizlik çağrısı eklemeyi unutma.
