# Çekirdek Mimari — CatBot

Paket: `com.worldforge.catbot` (+ `build`, `goal`, `client`) · 13+5+4+3 dosya

## 1. Amaç

Oyuncuya ait, takip eden/görev yapan genel amaçlı bir kedi-NPC. İki temel yeteneği var: (a) `/wfcat ask` ile serbest metin soru cevaplama, (b) `/wfbuild`/`CatBuildCommand` ile doğal dilden yapı inşası. Farmer Robot ile aynı `Goal`-tabanlı davranış desenini paylaşır ama ek olarak **LLM entegrasyonu** içerir.

## 2. Varlık Modeli

```
CatBotEntity extends PathfinderMob implements GeoEntity
```

- GeckoLib animasyon kontrolü: `movementController(AnimationTest<E>)` → `PlayState`.
- Görünüm varyasyonu: `CatForm` (enum), `CatGender` (enum), `CatHairColor` (enum) — kozmetik özelleştirme.
- Durum: `CatState` (enum) — muhtemelen idle/following/working gibi genel davranış modu.
- Sahiplik NBT'de tutulur (`addAdditionalSaveData`) — tek doğruluk kaynağı budur.

### CAT ↔ HUMANOID geçişi ve "devasa" (giant) boyut

`setState(CatState.WORKING)` çağrıldığı an `switchForm(CatForm.HUMANOID)` tetiklenir (poof parçacığı + ses ile), iş bitip `pollBuildJobCompletion()` `CatState.RETURNING`'e geçtiği an otomatik olarak `CatForm.CAT`'e geri döner — bu iki yönlü geçiş zaten bağlıydı, elle bir yerden çağrılması gerekmiyor.

HUMANOID formu artık gerçekten **devasa**: `CatBotEntity.GIANT_SCALE` (varsayılan `3.5f`) hem hitbox'ı (`HUMANOID_DIMENSIONS`, `getDefaultDimensions`) hem de render boyutunu büyütüyor. Render tarafı `CatBotGeoRenderer#scaleModelForRender` içinde yapılıyor — `CatBotGeoModel`'in render state'e zaten koyduğu `FORM` verisini okuyup HUMANOID iken `GIANT_SCALE` katsayısını, CAT iken `1.0f`'u uyguluyor. İkisi ayrı yerlerde tutulduğu için (hitbox `CatBotEntity`'de, görsel çarpan `CatBotGeoRenderer`'da) yeni bir katsayı denenirken **her iki sabiti birden** güncellemeyi unutma, yoksa hitbox ile görünen model boyutu birbirinden ayrışır.

## 3. Sahiplik Takibi — `CatBotManager`

```java
Map<UUID ownerUuid, UUID catUuid>   // ConcurrentHashMap, singleton
```

**Sadece çalışma zamanı önbelleği** — kalıcı veri değil. `/wfcat spawn|remove|info` komutlarını hızlandırmak için var; sunucu yeniden başladığında temizlenir (kabul edilen bir sınır — bir oyuncu restart sonrası ama önbellek dolmadan 2. kedi spawn edebilir, Faz 1 için kabul edilmiş, ileride sunucu başlangıcında yüklü entity'leri tarayarak kapatılabilir).

## 4. Davranış Goal'leri (`catbot/goal`)

| Sınıf | Amaç |
|---|---|
| `ApproachOwnerGoal` | Sahibine yaklaşma |
| `ReturnToOwnerGoal` | Sahibine geri dönme |
| `WorkSiteWanderGoal` | İş alanında dolaşma |
| `SmartNavigator` | **Yardımcı sınıf**, goal değil — pathfinding optimizasyonu |

### `SmartNavigator` — Yeniden-Yol-Bulma Kısıtlaması

Her goal, mob'un `PathNavigation`'ına doğrudan `moveTo()` çağırmak yerine bir `SmartNavigator` örneği üzerinden geçmeli. Yeni bir yol (gerçek A* hesaplaması) sadece şu durumlarda üretilir:
- Önbellekte hedef yoksa (ilk çağrı / `start()`/`reset()` sonrası),
- Hedef, son yol üretildiğinden beri `repathDistanceSq` (varsayılan `1.5²`) kadar hareket ettiyse,
- Navigator `isDone()` durumuna `stop()` çağrılmadan düşmüşse (önceki yol tamamlandı/geçersizleşti/başarısız oldu).

Diğer her tick no-op'tur — mob mevcut yolunu takip etmeye devam eder. Bu desen hem ucuz hem de "titremeyen" (steadier-looking) hareket sağlar; **Farmer/CatBot dışındaki yeni mob AI'larında da tekrar kullanılabilir bir kalıp.**

## 5. LLM Entegrasyonu

### 5.1 `CatAskService` — `/wfcat ask`

**Strateji: LLM önce, web arama yedek.**
1. `CatAskConfig.llmEnabled()` true ise (API anahtarı ayarlıysa) yapılandırılmış LLM endpoint'i dener (OpenAI-uyumlu `/chat/completions` şeması — OpenAI ve çoğu uyumlu sağlayıcıyla çalışır).
2. Anahtar yoksa VEYA LLM çağrısı herhangi bir nedenle başarısız olursa (ağ hatası, kötü anahtar, timeout, 2xx-olmayan yanıt) → anahtarsız web aramasına (DuckDuckGo Instant Answer API) düşer — böylece özellik sıfır yapılandırmayla da çalışır.

Her iki yol da **çağıranın thread'inde** çalışır — `CatBotCommands` bunu ana sunucu thread'i dışında çalıştırmaktan ve `server.execute()` ile geri dönmekten sorumludur. `AskResult(boolean success, String answer, String source)` — asla exception fırlatmaz, başarısızlık `success=false` olarak döner.

### 5.2 `CatBuildLlmService` — `/wfbuild` LLM yolu

`CatAskService` ile **aynı endpoint/config**'i (`CatAskConfig`) paylaşır — operatör tek bir anahtar girer, iki özellik de çalışır. Model'den **katı JSON** istenir: düz bir liste halinde göreli blok pozisyonları + vanilla blok id'leri + kısa bir etiket. Bu JSON, dünyaya dokunmadan önce sıkı doğrulanır:
- Sınırlayıcı-kutu boyutu,
- Toplam blok sayısı,
- Sunucuyu troll'leyebilecek/çökertebilecek blokların kara listesi (bedrock, komut blokları, portallar, TNT, spawner'lar vb.).

Herhangi bir hatada (anahtar yok, ağ hatası, bozuk/aşırı büyük JSON, doğrulamada her şeyin filtrelenmesi) başarısız sonuç döner, **asla exception fırlatmaz**. `CatBuildCommand` bu durumda mevcut çevrimdışı `BuildRequestParser` + `StructureTemplates` boru hattına düşer — yani `/wfbuild`'in temel davranışı hiçbir zaman bu servisin başarısına bağımlı değildir.

## 6. `/wfbuild` Komut Yolu — Eski vs Yeni

`catbot/build/` paketindeki `BuildRequestParser` ve `StructureTemplates` **`@Deprecated`** durumda:

```
BuildRequestParser  → yerini aldı: wfbuild.intent.KeywordIntentParser (via IntentEngine)
StructureTemplates  → yerini aldı: IntentEngine → BlueprintEngine → ArchitectureEngine
                       (+ StyleEngine malzeme için)
```

Eski sınıflar **hâlâ kodda duruyor** (kullanılmıyor) sadece başka bir yerden referans edilirse hiçbir şeyin kırılmaması için. `CatBuildCommand` artık çevrimdışı yolu `IntentEngine` üzerinden `wfbuild` motor zincirine yönlendiriyor (bkz. `docs/architecture/wfbuild.md`).

`BuildSpec` (record: `StructureType type, StructureSize size`) hâlâ eski API'nin veri taşıyıcısı.

## 7. Diğer Bileşenler

- `CatBotConfig` / `CatAskConfig` — sırasıyla genel bot davranış config'i ve LLM/ask config'i; ikisi de `config/worldforge/` diskinden `WorldForgeMod.onInitialize()`'da yüklenir.
- `CatRespawnScheduler` — ölüm sonrası cooldown ile yeniden spawn (Faz 6, `.getInstance().register()`).
- `CatBotCommands`, `CatApiCommand`, `CatBuildCommand` — Brigadier komut kayıtları (`WorldForgeMod.registerCommands()` içinden çağrılıyor).
- `CatBotEntities.bootstrap()` — entity type + attribute tanımı (Faz 1).

## 8. İstemci Tarafı (`catbot/client`)

`CatBotClientInit` (ClientModInitializer) → `CatBotGeoModel` + `CatBotGeoRenderer<R extends LivingEntityRenderState & GeoRenderState>` — GeckoLib render zinciri, `MagnetCarrierGeoRenderer`/`FarmerRobotGeoRenderer` ile aynı jenerik desende.

## 9. Genişletme Notları

- Yeni bir goal eklerken `SmartNavigator` deseninden geçmeyi unutma — doğrudan `getNavigation().moveTo()` çağırmak performans regresyonuna yol açar.
- LLM tabanlı yeni bir özellik eklerken `CatAskService`/`CatBuildLlmService` desenini izle: **asla exception fırlatma, her zaman sonuç objesiyle başarısızlığı bildir, çevrimdışı bir yedek yol bırak.**
- `BuildRequestParser`/`StructureTemplates`'e dokunma — deprecated ve ölü kod; yeni yapı mantığı `wfbuild` paketine gider.
