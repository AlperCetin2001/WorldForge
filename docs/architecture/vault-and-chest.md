# Çekirdek Mimari — Vault & Chest Depolama Sistemleri

Paket: `com.worldforge.vault` (12 dosya) + `com.worldforge.chest` (6 dosya)

## 1. WFVault — Sayfalanmış, Aranabilir Depo

### 1.1 Blok/Entity
```
WFVaultBlock extends BaseEntityBlock
WFVaultBlockEntity extends BaseContainerBlockEntity
```
216 slotluk, obsidyen-demir görünümlü tamamen özel bir kasa (kod içi yorum: "Vanilla double-chest cluster KALDIRILDI").

### 1.2 Kategorileme ve Sıralama
- `ItemCategory` (enum) — item'ları kategorilere ayırır (görüntüleme/filtreleme için).
- `VaultSorter` — kategoriye/isim/miktara göre sıralama mantığı.
- `PagedVaultView implements Container` — büyük envanteri sayfalara böler; GUI tek seferde tüm 216+ slotu göstermek yerine sayfa sayfa sunar.

### 1.3 Çoklu Kasa Birleştirme — `MultiVaultContainer`
```java
class MultiVaultContainer implements Container
```
Birden fazla `WFVaultBlockEntity`'yi **dışarıya tek bir `Container` gibi** sunar. Bu, `WorkChestManager`'ın büyük alan seçimlerinde ikinci/üçüncü bir kasa spawn edip hepsini şeffaf şekilde birleştirmesini sağlar (bkz. §2).

### 1.4 Menü/Ekran + Arama
```
WFVaultMenu extends AbstractContainerMenu
WFVaultScreen extends AbstractContainerScreen<WFVaultMenu>
```
`WFVaultScreen`'deki arama kutusuna Enter'a basınca, sorgu `VaultSearchJumpPayload` (C2S) ile sunucuya gönderilir; `WFVaultMenu.searchAndJumpToPage(query)` eşleşen item'ın bulunduğu sayfaya menüyü kaydırır. Kayıt `WorldForgeMod.registerVaultSearchNetworking()`'de yapılır — payload tipi kaydı her iki mantıksal tarafta (client/server) gereklidir, alıcı sadece sunucuda tetiklenir.

### 1.5 Kayıt ve Client Init
`WFVaultRegistry.register()` (blok/item/block-entity-type), `WFVaultClientInit` (ClientModInitializer), `WFVaultNetwork` (genel network yardımcıları).

## 2. WorkChestManager — Otomatik Vault Spawn Orkestratörü

`wffill`/`wfbreak` (blok operasyon) komutları için WFVault blok(lar)ı spawn eder. **Stage 3 — çoklu sandık desteği** kod yorumlarına göre:

- Vanilla double-chest cluster kaldırıldı; her oyuncu için **bir veya daha fazla** `WFVaultBlock` spawn edilir.
- Kapasite kuralı: tek bir WFVault, aynı item türünden en fazla `WFVaultBlockEntity.CAPACITY` (216 × 64 = **13.824**) tutabilir. İşlem hacmi (region volume / seçim boyutu) bunu aşarsa **ikinci bir WFVault otomatik spawn edilir** — büyük alan seçimlerinde tek sandığın kapasitesi dolup fazla blokların/düşen item'ların yere düşmesinin önüne geçilir.
- Container arayüzü dışarıya **her zaman tek bir Container** olarak sunulur: 1 sandık varsa doğrudan, 2+ sandık varsa `MultiVaultContainer` ile birleştirilerek. Bu sayede `DestructionEngine`, `TickedTaskScheduler`, `OmnitoolRepairSystem` **hiçbir değişiklik gerektirmez** — hepsi tek bir `Container` API'siyle konuşur.

**Tasarım dersi:** Çağıran kodun (destruction/repair/scheduler) çoklu-kasa karmaşıklığından tamamen izole edilmesi — bu izolasyon `Container` arayüzünün tekilleştirilmesiyle sağlanıyor. Yeni bir "birden fazla depo kaynağı" senaryosu eklenirse aynı `MultiVaultContainer` sarmalama deseni tekrar kullanılabilir.

## 3. MegaChest — Craftlanabilir 216 Slotluk Sandık

```java
MegaChestBlockEntity extends WFVaultBlockEntity   // WFVault'un doğrudan alt sınıfı
```

`MegaChestBlock`, `MegaChestRegistry.register()`, `MegaChestNetwork` — WFVault'un tüm depolama/arama altyapısını miras alan, oyuncunun **craftlayabildiği** bir kasa versiyonu. Yeni işlevsellik eklemek yerine mevcut `WFVaultBlockEntity` davranışını genişletiyor.

## 4. ChestMoverItem

`extends Item` — bir sandığı içeriğiyle birlikte taşımaya yarayan araç; `WorldForgeMod.registerChestMover()` içinde kaydediliyor, Tools & Utilities creative tab'ında.

## 5. Genişletme Notları

- Yeni bir depo türü eklerken `WFVaultBlockEntity`'den türetmeyi düşün (MegaChest deseni) — arama/sayfalama/kategori mantığını bedavaya kazanırsın.
- Kapasiteyi aşan senaryolarda her zaman `MultiVaultContainer` ile sarmala; çağıran kodun tekil-`Container` varsayımını asla boz.
- Arama/senkron için yeni bir C2S/S2C payload eklerken `VaultSearchJumpPayload` desenini (kayıt + `WorldForgeMod`'da özel bir `register*Networking()` metodu) izle.
