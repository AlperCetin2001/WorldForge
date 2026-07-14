# Çekirdek Mimari — Magnet Link

Paket: `com.worldforge.magnet` (5) + `magnet/client` (3)

## 1. Amaç

İki envanter noktası arasında (mesafeden bağımsız) periyodik otomatik item transferi kuran bir "bağlantı" sistemi. `MagnetItem` ile iki nokta işaretlenir, aralarında bir `MagnetLink` oluşur.

## 2. Veri Modeli — Sadece Koordinat, Asla Canlı Referans

```java
final class MagnetLinkManager {   // singleton, ConcurrentHashMap<UUID, MagnetLink>
    // her 20 tick'te bir transfer döngüsü
    // dünyanın kayıt klasörüne (LevelResource) JSON olarak kalıcı
}
```

**Kritik tasarım kararı:** Bağlantılar sadece `UUID → MagnetLink` (boyut + `BlockPos` çifti) olarak saklanır. **Hiçbir `BlockEntity`, `Container` veya `Entity` referansı tick'ler arasında önbelleğe alınmaz** — her döngüde dünya durumundan yeniden çözümlenir, ve sadece zaten **yüklü olan chunk'lar** için (`processLink` metodunda kontrol edilir).

**Performans sonucu:** Transfer döngüsü sadece aktif bağlantılar haritasını dolaşır, dünyanın tüm block-entity'lerini değil — maliyet bağlantı sayısıyla ölçeklenir, dünya boyutuyla değil.

## 3. Kalıcılık

`ServerLifecycleEvents` ile entegre; JSON serileştirme `Gson`/`GsonBuilder` ile, `LevelResource` altında dünyanın save klasörüne yazılır. Sunucu tick'i `ServerTickEvents` ile bağlı.

## 4. Görsel Taşıyıcı — `MagnetCarrierEntity`

```java
class MagnetCarrierEntity extends Entity implements GeoEntity
```

Transferi **görsel olarak temsil eden kozmetik bir kurye entity** — gerçek transfer mantığı `MagnetLinkManager`'da zaten tamamlanmış durumda; bu entity sadece oyuncuya "bir şey taşınıyor" hissi veren bir animasyon katmanı (GeckoLib `movementController`). `MagnetCarrierEntities.bootstrap()` ile entity type kaydı yapılır (`WorldForgeMod.onInitialize()`).

## 5. İstemci Tarafı

`MagnetClientInit` (ClientModInitializer) → `MagnetCarrierGeoModel` + `MagnetCarrierGeoRenderer<R extends EntityRenderState & GeoRenderState>` — CatBot/FarmerRobot ile aynı GeckoLib render zinciri jenerik deseni.

## 6. Genişletme Notları

- Yeni bir "bağlantı" tipi (ör. akışkan transferi) eklenirse `MagnetLinkManager`'ın **"asla canlı referans önbelleğe alma, her tick yeniden çözümle"** kuralını koru — bu, chunk boşaltma/yeniden yükleme senaryolarında hayalet referans hatalarını önlüyor.
- Kozmetik taşıyıcı entity'ler (MagnetCarrier gibi) gerçek mantığı asla içermemeli — sadece görsel katman olarak kalmalı, böylece entity spawn/despawn sorunları veri bütünlüğünü etkilemez.
