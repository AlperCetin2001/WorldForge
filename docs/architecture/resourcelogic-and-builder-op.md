# Çekirdek Mimari — Resource Logic & Builder Operations

Paket: `com.worldforge.resourcelogic` (14 dosya + `json/` 3) · `com.worldforge.builder.op` (10 dosya)

## 1. Resource Logic — JSON-Tanımlı Blok Kuralları

**Amaç:** Su/lava ve genel blok davranışlarını kod değişikliği gerektirmeden, veri (JSON) ile tanımlamak.

### 1.1 Çekirdek Soyutlama

```java
interface ResourceRule { ... }               // bir kuralın "arayüzü"
enum ResourceType { ... }                    // WATER, LAVA, GENERIC vb.
enum ResourceOperation { ... }                // hangi işlem (fill, drain, vb.)
interface ResourceOperationHandler { ... }    // bir operasyonu yürüten mantık
class ResourceResult { ... }                  // işlem sonucu (başarı/başarısızlık + detay)
class ResourceContext { ... }                 // işlemin bağlamı (dünya, oyuncu, konum vb.)
```

**Somut kural implementasyonları:** `WaterRule`, `LavaRule`, `GenericBlockRule` (`implements ResourceRule`) — her biri `ResourceRuleRegistry` üzerinden erişilir, `ResourceRuleManager` çalışma zamanı orkestrasyonunu yapar. `FluidOperation implements ResourceOperationHandler` somut akışkan operasyon uygulaması.

`ResourceOperationRegistry` — operasyon türü → handler eşlemesi. `ResourceLogicBootstrap` — başlangıçta tüm kuralları/operasyonları kaydeden giriş noktası.

### 1.2 JSON Yükleme + "extends" Kalıtımı (`resourcelogic/json`)

```
RawRuleDefinition        (ham JSON DTO, Gson @SerializedName("extends") → extendsFrom alanı)
      │
      ▼  RuleJsonLoader.walk "extends" chain
ResolvedRuleDefinition   ("extends" kalıtımı tamamen düzleştirilmiş hali)
```

**Kalıtım mekanizması:** Bir kural JSON'da `"extends": "crop"` diyerek başka bir kuralın **belirtmediği tüm alanlarını devralır**, sadece override etmek istediği alanları yazar. `RuleJsonLoader`:
- Zinciri (`extendsFrom`) tek bir ID için yürüyüp tek bir birleşik tanıma düzleştirir (`resolve` metodu).
- Bilinmeyen bir ebeveyn ID'sine `extends` edilmişse `IllegalStateException` fırlatır.
- **Döngüsel `extends` zincirini tespit eder** ve `IllegalStateException` ile durdurur — bu, veri odaklı kalıtım sistemlerinde kritik bir güvenlik kontrolü.

`data/worldforge/resourcelogic/` altındaki JSON dosyaları bu formatı kullanır.

## 2. Builder Operations — Düşük Seviye Blok İşlemleri

**Amaç:** WorldEdit tarzı, tek-adımlık, birleştirilebilir blok operasyonları. Her operasyon bir `record` — immutable, parametreleriyle birlikte kendini tanımlayan bir veri parçası.

```java
interface BuilderOperation { ... }
record BuilderContext(...) { ... }              // operasyonun çalıştığı bağlam
```

**Somut operasyonlar (hepsi `implements BuilderOperation`):**
| Operasyon | Amaç |
|---|---|
| `PlaceBlockOperation(BlockState state)` | Tek blok yerleştirme |
| `PlaceDoorOperation(DoorBlock doorBlock)` | Kapı yerleştirme (özel — çift blok yüksekliği/menteşe yönü gerektirir) |
| `RotateStairOperation()` | Merdiven döndürme |
| `FillWaterOperation()` | Su doldurma |
| `CreateInfiniteWaterOperation()` | Sonsuz su kaynağı oluşturma (2 su kaynağı yan yana → vanilla mekaniği) |
| `MergeChestOperation()` | Çift sandık birleştirme |

`BuilderOperationRegistry` + `BuilderOperationBootstrap` — kayıt/başlatma; `wfbuild/architecture`'daki `ArchitectureOperations.register()` çağrısıyla aynı desende (`ConstructionOperationRegistry`, bkz. wfbuild.md §5).

## 3. İki Sistemin İlişkisi

- **`resourcelogic`** → "bu blok/akışkan nasıl davranır" sorusuna JSON'la cevap verir; `ConstructionValidationEngine` (wfbuild Layer 6) malzeme kontrolü için bunu kullanır.
- **`builder/op`** → "bu tek adımı nasıl uygularım" sorusuna kod'la (ama veri gibi, record olarak) cevap verir; `wfbuild/plan.ConstructionOperation` ile kavramsal olarak paralel ama farklı bir soyutlama seviyesinde (builder/op daha genel/düşük seviyeli WorldEdit-tarzı komutlar için, ConstructionOperation ise wfbuild'in kendi inşaat fazları için).

## 4. Genişletme Notları

- **Yeni bir resourcelogic kuralı** eklerken JSON tanımına `extends` ile mevcut bir kuraldan türet — kod değişikliği gerekmeyebilir; sadece yeni davranış gerekiyorsa `ResourceRule` implementasyonu yaz ve `ResourceRuleRegistry`'ye kaydet.
- **Yeni bir builder operasyonu** eklerken `record ... implements BuilderOperation` desenini izle (immutable, parametrelerle kendini tanımlayan), `BuilderOperationRegistry`'ye kaydet.
- `extends` zinciri ekleyen/değiştiren her değişiklikten sonra döngüsel referans olup olmadığını manuel kontrol et — `RuleJsonLoader` çalışma zamanında yakalar ama derleme zamanında değil.
