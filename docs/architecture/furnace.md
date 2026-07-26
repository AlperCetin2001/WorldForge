# Çekirdek Mimari — 6 Katmanlı Fırın Sistemi

Paket: `com.worldforge.furnace.*` · 34 dosya (block, blockentity, menu, screen)

## 1. Köken ve Amaç

Bağımsız `copperfurnace-faz3` adlı bir projeden WorldForge'a entegre edilmiş (kod içi yorum, `WorldForgeMod.onInitialize()`). Vanilla `AbstractFurnaceBlock`'u genişleterek **6 metal katmanı** için çoklu giriş/yakıt hatlı fırınlar sunar: Copper, Iron, Gold, Diamond, Emerald, Netherite.

## 2. Katman Başına Üçlü Sınıf Seti

Her metal seviyesi tam olarak aynı üç sınıfı tekrarlar (kopyala-uyarla deseni):

```
<Metal>FurnaceBlock        extends AbstractFurnaceBlock
<Metal>FurnaceBlockEntity  extends BaseContainerBlockEntity implements WorldlyContainer
<Metal>FurnaceMenu         extends AbstractContainerMenu
<Metal>FurnaceScreen       (istemci, AbstractContainerScreen<...Menu>)
```

Merkezi kayıt sınıfları: `FurnaceBlocks`, `FurnaceBlockEntities`, `FurnaceMenus` — hepsi `WorldForgeMod.onInitialize()`'da sırayla çağrılıyor.

## 3. Blok — `<Metal>FurnaceBlock`

```java
protected MapCodec<? extends AbstractFurnaceBlock> codec() { ... }
protected <T extends BlockEntity> BlockEntityTicker<T> getTicker(Level, BlockState, BlockEntityType<T>) { ... }
private static <T extends BlockEntity, E extends BlockEntity> BlockEntityTicker<T> createFurnaceTicker(...) { ... }
```

Her katman aynı 3 metodu override eder; tek fark ilgili `<Metal>FurnaceBlockEntity` tipi.

## 4. Block Entity — Çoklu Giriş/Yakıt Hattı

`CopperFurnaceBlockEntity` (diğerleri aynı desende) **vanilla tek-slotlu modelin ötesine geçer**:

```java
SLOT_INPUT_1 = 0, SLOT_INPUT_2 = 1,   // 2 bağımsız pişirme girişi
SLOT_FUEL_1  = 2, SLOT_FUEL_2  = 3,   // 2 bağımsız yakıt slotu
SLOT_OUTPUT  = 4,
SLOT_COUNT   = 5

int[] cookProgress   = new int[2];    // her giriş için ayrı pişirme ilerlemesi
int[] cookTotalTime  = new int[2];
```

- `WorldlyContainer` implementasyonu ile yön bazlı otomasyon desteği: `SLOTS_UP` (giriş), `SLOTS_DOWN` (çıkış), `SLOTS_SIDE` (yakıt) — hopper/boru sistemleriyle uyumlu.
- `litTime`/`litDuration` — genel yanma durumu; `recipesUsed` (`Map<ResourceKey<Recipe<?>>, Integer>`) XP dağıtımı için tarif kullanım sayacı tutar.
- Vanilla `SmeltingRecipe`/`AbstractCookingRecipe`/`RecipeManager` ile entegre — özel tarif sistemi değil, vanilla eritme tariflerini kullanıyor, sadece paralel işleme ekliyor.

## 5. Menü — `<Metal>FurnaceMenu`

```java
extends AbstractContainerMenu
private static class FuelSlot extends Slot { ... }
private static class ResultSlot extends Slot { ... }
```

Özel `FuelSlot` (sadece yanabilir item kabul eder) ve `ResultSlot` (sadece çıkarılabilir, XP tetikler) — vanilla `FurnaceMenu`'nün 2x giriş/yakıt için genişletilmiş hali.

## 6. Genişletme Notları

- **Yeni bir metal katmanı** eklemek = mevcut 4 dosyayı (Block/BlockEntity/Menu/Screen) bir üsttekinden kopyalayıp isim + istatistik (yanma süresi çarpanı, pişirme hızı vb.) değiştirmek; merkezi `FurnaceBlocks`/`FurnaceBlockEntities`/`FurnaceMenus`'a kayıt eklemek.
- Slot indeksleri (`SLOT_INPUT_1` vb.) **tüm 6 katmanda aynı sırada** — bir değişiklik yapılırsa hepsinde tutarlı yapılmalı, aksi halde `WorldlyContainer` yön-slot eşlemeleri bozulur.
- `recipesUsed` haritası XP dağıtımı için kritik; yeni slot eklenirse bu haritanın anahtarlama mantığı (hangi recipe hangi girişe ait) gözden geçirilmeli.
