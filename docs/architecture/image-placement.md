# Çekirdek Mimari — İnternet Görseli Yerleştirme

Paket: `com.worldforge.image` (8 dosya)

## 1. Amaç

Bir URL'den indirilen görseli (jpg/jpeg/png/gif) oyuncunun seçtiği düz bir yüzeye **map-art / item-frame** olarak yerleştirmek.

## 2. Akış (Uçtan Uca)

```
WFImageViewerItem (item) → sağ-tık → WFImageViewerScreen (istemci önizleme, WFImageDownloader ile)
        │
        │  oyuncu "Yerleştir" butonuna basar
        ▼
ImagePlacementPayload(String url)  — C2S CustomPacketPayload
        │
        ▼ (WorldForgeMod.registerImagePlacementNetworking() alıcısı)
1. FeatureConfig.isImageEnabled() kontrolü — kapalıysa hata mesajı, dur.
2. ImagePlacementManager.hasCompleteSelection(player) — bölge + yüz seçili mi?
3. CompletableFuture.runAsync(...) ana thread'i BLOKLAMADAN:
      ServerImageDownloader.download(url)  →  BufferedImage
4. server.execute(...) ile ana thread'e DÖN:
      MapArtPlacer.place(level, region, face, image)  →  gerçek item-frame'ler
```

## 3. Ana/Async Thread Ayrımı — Kritik Desen

```java
CompletableFuture.runAsync(() -> {
    // AĞIR İŞ: ağ indirmesi, resim decode — ana thread DIŞINDA
    BufferedImage image = ServerImageDownloader.download(url);
    server.execute(() -> {
        // DÜNYA DEĞİŞİKLİĞİ: sadece ana thread'de güvenli
        if (server.getPlayerList().getPlayer(player.getUUID()) == null) return; // oyuncu ayrılmış olabilir
        MapArtPlacer.place(...);
    });
});
```

Bu, projenin **her async I/O + dünya-değişikliği** senaryosunda tekrarlanan bir kalıp (`CatBuildLlmService`, `CatAskService` ile aynı desen): ağır iş asenkron, sonuç `server.execute()` ile ana thread'e taşınıp orada uygulanıyor. Oyuncunun bu sırada ayrılmış olma ihtimaline karşı **null kontrolü** her zaman yapılıyor.

## 4. Bileşenler

| Sınıf | Sorumluluk |
|---|---|
| `WFImageViewerItem extends Item` | Item tanımı, sağ-tık ile ekranı açar |
| `WFImageViewerScreen extends Screen` | İstemci önizleme ekranı, URL girişi + "Yerleştir" |
| `WFImageDownloader` | **İstemci** tarafı indirme (önizleme için) |
| `ServerImageDownloader` | **Sunucu** tarafı indirme (`NativeImage` değil `BufferedImage` — çünkü `NativeImage` client-only) |
| `ImagePlacementManager` | Oyuncu başına aktif seçim (bölge + yüz) durumu, `hasCompleteSelection`, `getRegion`, `getFace` |
| `MapArtPlacer` | `place(ServerLevel, Region, Direction, BufferedImage)` → gerçek map-art item-frame yerleşimi; düz olmayan yüzeyler için `NotFlatException` fırlatır |
| `ImagePlacementPayload` | C2S payload (`record`, sadece `url`) |
| `WFImageRegistry` | Item kaydı (`IMAGE_VIEWER_ITEM`) |

## 5. Hata Yolları (Hepsi i18n Üzerinden)

- Özellik kapalı → `imageplace.fail` ("feature disabled by admin")
- Seçim eksik → `imgselection.none`
- İndirme başlıyor → `imageplace.sending`
- İndirme başarısız → `imageplace.fail` (exception mesajıyla)
- Yüzey düz değil → `MapArtPlacer.NotFlatException` → `imageplace.fail`
- Başarı → `imageplace.success` (yerleştirilen frame sayısıyla)

## 6. Genişletme Notları

- Yeni bir async-indirme-sonra-dünya-değişikliği özelliği eklerken bu dosyanın §3'teki thread ayrım desenini birebir kopyala — özellikle "oyuncu hâlâ bağlı mı" null kontrolünü atlama.
- `NativeImage` sadece client-side kullanılabilir; sunucu tarafı görsel işleme için her zaman `BufferedImage`/`java.awt` kullan (bkz. `ServerImageDownloader`).

## 7. Blok İnşa Modu (BlockArtPlacer) — ek özellik

`WFImageViewerScreen`'e eklenen bir mod anahtarı ("Mod: Harita Sanatı" / "Mod: Blok İnşa"), "Yerleştir" butonunun hangi C2S payload'ını göndereceğini belirler:

| Mod | Payload (URL) | Payload (yerel dosya) | Sunucu tarafı yerleştirici |
|---|---|---|---|
| Harita Sanatı (varsayılan) | `ImagePlacementPayload` | `ImagePlacementLocalPayload` | `MapArtPlacer` — item-frame + map |
| Blok İnşa | `BlockArtPlacementPayload` | `BlockArtPlacementLocalPayload` | `BlockArtPlacer` — gerçek blok |

Mevcut altyapının (indirme, önizleme, seçim, §3'teki thread deseni) TAMAMI aynen paylaşılıyor; sadece son adım — hangi placer'ın çağrıldığı ve hangi i18n başarı mesajının (`imageplace.success` vs `blockart.success`) gösterildiği — dallanıyor. Bilinçli olarak `ImagePlacementPayload`'a bir "mode" alanı eklemek yerine paralel payload sınıfları (`BlockArtPlacementPayload`/`BlockArtPlacementLocalPayload`) tercih edildi, böylece çalışan harita-sanatı wire formatına hiç dokunulmadı.

`BlockArtPlacer`, `MapArtPlacer` ile birebir aynı bölge/eksen/flatness mantığını kullanır (§ tanımları senkron tutulmalı), ama:
- Bir görsel pikseli = bir blok konumu (128x128 harita fayansı ölçeklemesi yok).
- Palet, `BuiltInRegistries.BLOCK`'taki TÜM uygun bloklardan (katı, block-entity'siz, `DENYLIST`'te değil, tam küp) `BlockState#getMapColor` ile inşa edilir ve sunucu ömrü boyunca önbelleğe alınır.
- **Filtreler**: `DENYLIST` — barrier, light, command/chain-command/repeating-command block, structure block/void, jigsaw, spawner, **cobweb** (geçirgen, entity'leri yavaşlatır/tuzağa düşürür, seyrek bir örgü gibi görünür — statik piksele uymuyor) (bunların çoğu zaten `EntityBlock` kontrolüyle de eleniyor, ama açıkça listelenmesi bu davranışın o implementasyon detayına bağımlı kalmamasını sağlıyor). `isNonFullCubeType()` — iki grup blok eler: (1) tam küp OLMAYAN tipler: slab, stairs, fence, fence gate, wall, door, trapdoor, pressure plate, button, carpet, ladder, iron bars/pane, torch, sapling/bush (aksi halde resimde boşluklar oluşurdu); (2) tam küp OLSA BILE statik/tek renkli bir piksele uymayan tipler: **leaves** (biome tint'e göre renk değişir + zamanla çürür), **beds** (2 blok genişliğinde, yön bağımlı, zaten block-entity taşıyor), fire/campfire/scaffolding/tripwire (animasyonlu veya etkileşimli).
- **Doğrulandı**: `getMapColor(BlockGetter, BlockPos) → MapColor` çağrısı MC 26.2 client.jar'a göre teyit edildi (`BlockBehaviour.BlockStateBase`'den miras, imza değişmemiş).
