# Çekirdek Mimari — Seçim, Klipboard, Geri Alma

Paket: `com.worldforge.selection` (3) + `com.worldforge.clipboard` (2) + `com.worldforge.undo` (3)

## 1. Seçim — `Region` / `SelectionManager`

```java
final class Region implements Iterable<BlockPos>   // dikdörtgen prizma bölge, doğrudan for-each ile gezilebilir
final class SelectionManager                         // oyuncu başına aktif seçim (pos1/pos2) durumu
final class WoodenAxeSelectionItem extends Item      // sol-tık pos1, muhtemelen sağ-tık pos2
```

`WorldForgeMod.registerSelectionAxe()` + `registerAttackBlockHook()` ile bağlı: `AttackBlockCallback.EVENT` **sadece sunucu tarafında** işlenir (client'ta `PASS` döner, çift-tetiklenmeyi önler); `WoodenAxeSelectionItem.handlePos1(serverPlayer, pos)` çağrılır ve `InteractionResult.CONSUME` döner (ekstra client→server round-trip'i önlemek için `SUCCESS` yerine).

`Region implements Iterable<BlockPos>` — bölgedeki her bloğu döngüyle gezmeyi doğal hale getiriyor; `resourcelogic`, `builder/op`, `wfbuild` gibi bölge-bazlı işlem yapan her sistem muhtemelen bunu tüketiyor.

## 2. Klipboard — Kopyala/Yapıştır

```java
final class ClipboardData    // kopyalanan blokların anlık görüntüsü (göreli konum + BlockState)
final class ClipboardManager // oyuncu başına klipboard durumu, kopyala/yapıştır orkestrasyonu
```

## 3. Undo/Redo — `UndoManager`

```java
record BlockChangeRecord(BlockPos pos, BlockState before, BlockState after)
final class UndoEntry    // bir işin (job) tüm BlockChangeRecord'larının grubu + meta veri
final class UndoManager  // oyuncu başına /wfundo ve /wfredo geçmişi (ArrayDeque<UndoEntry>)
```

### 3.1 Yakalama Zinciri

> *"Bir işin blok değişiklikleri iş çalışırken TickedTaskScheduler tarafından yakalanır ve [UndoManager'a] devredilir."*

Yani gerçek blok değişikliği kaydı **iş çalıştıran** `TickedTaskScheduler`'da olur; `UndoManager` bu kayıtları biriktirip geri-al/ileri-al yığınına organize eder.

### 3.2 Ekonomi Entegrasyonu

`UndoManager`, `EconomyConfig` ve `VaultEconomyBridge`'i import ediyor — bir iş için para tahsil edilmişse, `/wfundo` bu parayı **iade edebiliyor** (`EconomyProvider.deposit()`, bkz. economy). Bu, geri-alma sisteminin sadece blokları değil, işin yan etkilerini (ekonomi) de tersine çevirdiği anlamına gelir.

### 3.3 i18n ve Zamanlayıcı Entegrasyonu

`Lang` (mesajlar) ve `TickedTaskScheduler` (iş yürütme) ile sıkı bağlı; `ServerTickEvents` ile kayıtlı (`UndoManager.getInstance().register()`, `WorldForgeMod.onInitialize()`'da).

## 4. Genişletme Notları

- Yeni bir "geri alınabilir" işlem türü eklerken, işlemi `TickedTaskScheduler` üzerinden çalıştır ve `BlockChangeRecord` üretmesini sağla — `UndoManager` bunun ötesinde bir şeye ihtiyaç duymaz.
- Yeni bir işlem ekonomi ücreti tahsil ediyorsa, `/wfundo`'nun bunu iade edebilmesi için `VaultEconomyBridge.EconomyProvider.deposit()`'in çağrıldığından emin ol.
- `Region`'ı `Iterable<BlockPos>` olarak tüketen yeni bir sistem yazarken çok büyük bölgelerde performans için bir üst sınır (blok sayısı) düşünmeyi unutma — `FarmerFarmDetector.MAX_FARM_BLOCKS` benzeri bir desen bu projede standarttır.
