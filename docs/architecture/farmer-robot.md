# Çekirdek Mimari — Farmer Robot

Paket: `com.worldforge.farmer` (+ `farmer/client`) · 19+4 dosya · **Aktif geliştirme alanı**

## 1. Amaç

Bir tarım robotu: `FarmerChestBlock`'a bağlanır, çevredeki tarlayı bulur, ekin toplar, kasaya bırakır; gece/fırtınada sığınır, N hasattan sonra dinlenir.

## 2. Varlık Modeli

```
FarmerRobotEntity extends PathfinderMob implements GeoEntity
```

- GeckoLib animasyonları: `idle`, `walk`, `run` (`RawAnimation`), hız çarpanları `WALK_SPEED_MODIFIER=1.0`, `RUN_SPEED_MODIFIER=1.6`.
- Senkronize veri (`EntityDataAccessor`): `DATA_MOVING`, `DATA_RUNNING`, `DATA_ACTION`, `DATA_ACTION_SEQ` — istemci animasyon durumunu sunucudan bağımsız okuyabilsin diye.
- İç enum `FarmerAction` — anlık eylem türü (muhtemelen harvest/deposit/idle animasyon tetikleyicisi).
- Taşıma envanteri: `CARRY_INVENTORY_SIZE = 9` (kasaya varmadan biriktirilen ürünler).
- Kasaya bağlanma: `linkToChest(BlockPos, ResourceKey<Level>)` → `Optional<BlockPos> getLinkedChestPos()`, `getLinkedChestDimension()`, `hasLinkedChest()`, `getResolvedChestCenter()`, `getLinkedChestBlockEntity()`, `isLinkedChestFull()`.
- `removeWhenFarAway()` **override edilip `false` döner** — robot sahibinden uzaklaşsa da despawn olmaz (kalıcı bir mülk gibi davranır).

## 3. Davranış = Goal Yığını

`registerGoals()` içinde öncelik sırasına göre (düşük sayı = yüksek öncelik):

| # | Goal | Faz | Tetik koşulu (`canUse`) |
|---|------|-----|--------------------------|
| 0 | `FloatGoal` | vanilla | Suda batmama |
| 1 | `FarmerNightShelterGoal` | 12 | Gece veya fırtına |
| 2 | `FarmerRestGoal` | 14 | N hasattan sonra yorgunluk |
| 3 | `FarmerHarvestGoal` | — | Olgun ekin mevcut |
| 4 | `FarmerReturnToChestGoal` | — | Kasadan `RETURN_TRIGGER_DISTANCE=3.0` uzakta ve taşıma dolu/iş bitti |
| 5 | `FarmerWanderFromChestGoal` | — | İş yok, rastgele dolaşma (`WANDER_RADIUS=8.0`) |
| 6 | `FarmerDepositGoal` | — | Kasaya `DEPOSIT_DISTANCE=1.5` yakınlıkta ve envanterde ürün var |
| 7 | `FarmerLookAroundGoal` | — | Boşta, düşük ihtimalli (`START_CHANCE=300`) bakınma animasyonu |

Vanilla `Goal` API'sinin standart yaşam döngüsü kullanılıyor: `canUse()` → `start()` → `tick()` → `canContinueToUse()` → `stop()`. Fabric/Minecraft `goalSelector` bu önceliklere göre en yüksek öncelikli çalıştırılabilir goal'ü seçer; her tick yeniden değerlendirilir.

**Genişletme kuralı:** yeni bir goal eklerken önce mevcut tablodaki numaralarla çakışmayan bir öncelik seç, `registerGoals()`'a ekle, gerekiyorsa `FarmerRobotStatus`'a yeni bir durum ekle (sona, ordinal korunarak).

## 4. Durum Makinesi — `FarmerRobotStatus`

```java
enum FarmerRobotStatus {
    WORKING, IDLE, WAITING_FOR_SEEDS, CHEST_MISSING,
    NO_CROPS, SHELTERING, CHEST_FULL, RESTING
}
```

GUI'nin "Status" satırında gösterilir (`translationKey()` ile i18n'e bağlanır). Her yeni faz kendi durumunu **sona ekler**:
- Faz 12 → `SHELTERING` (gece/fırtına sığınma; `IDLE`'dan farklı çünkü robot yine "çalışıyor" sayılır)
- Faz 13 → `CHEST_FULL` (bağlı kasa doluyken hasat otomatik durur/devam eder, oyuncu müdahalesi gerekmez)
- Faz 14 → `RESTING` (N hasattan sonra dinlenme; verim/hız cezası yok, salt davranışsal)

## 5. Tarla Algılama — `FarmerFarmDetector`

- `isFarmBlock(BlockState)` — bir bloğun ekilebilir tarla parçası olup olmadığını kontrol eder.
- `findNearestFarmSeed(ServerLevel, BlockPos center, int radiusBlocks)` — yakın tarla arar.
- `floodFillFarm(ServerLevel, BlockPos seed)` — bağlı tarla alanını **flood-fill** ile bulur; `MAX_FARM_BLOCKS = 20_000` sınırı ve `VERTICAL_BRIDGE_RANGE = 8` (kat farklarını köprüleme) var — sunucu performansını korumak için.
- `incrementalUpdate(ServerLevel, List<BlockPos> cache)` — tam yeniden tarama yerine artımlı güncelleme (performans optimizasyonu).

## 6. Ekin Mantığı — `FarmerCrops` / `FarmerCropPriority`

- `FarmerCrops.isSeedItem`, `isMatureHarvestable(BlockState)`, `harvest(ServerLevel, BlockPos, BlockState)` → `List<ItemStack>`, `getSeedItem(Block)`, `plant(...)` — tekil ekin işlemleri.
- `FarmerCropPriority` — hangi ekin türünün önce hasat edileceğini belirleyen sıralı liste (`DEFAULT_ORDER`), çalışma zamanında `setOrder`/`resetToDefault` ile değiştirilebilir, `priorityOf(Block)` ile sorgulanır.

## 7. Kasa — `FarmerChestBlockEntity`

```
extends BaseContainerBlockEntity   // 3×9 = 27 slot (ROWS=3, COLS=9)
```

- `setLinkedRobot(UUID)` / `getLinkedRobot()` / `clearLinkedRobot()` — kasa ↔ robot 1:1 bağlantısı, UUID ile.
- `depositItem(ItemStack)` — robotun ürün bıraktığı ana giriş noktası.
- `isFull()` — `FarmerRobotStatus.CHEST_FULL` tetikleyicisi.
- `consumeSeed(Item, int reserve)` — ekim için tohum tüketirken bir miktar rezerv bırakır.
- `dropAllContents(Level, BlockPos)` — blok kırıldığında içeriği dünyaya döker.

## 8. GUI — `FarmerRobotMenu`

`AbstractContainerMenu` alt sınıfı, buton ID'leri: `BUTTON_RESUME=0`, `BUTTON_PAUSE=1`, `BUTTON_PICK_UP=2`, `BUTTON_OPEN_CHEST=3`. `clickMenuButton(Player, int)` bu eylemleri sunucu tarafında işler. `getStatus()` ile `FarmerRobotStatus` GUI'ye yansıtılır.

## 9. Spawn & Kayıt

- `FarmerRobotSpawnItem` — item kullanılınca (`useOn`) dünyaya robot spawn eder; `appendHoverText` ile tooltip.
- `FarmerRegistry.register()` — blok/item/block-entity-type kayıtları (`FARMER_CHEST_BLOCK`, `FARMER_CHEST_ITEM`, `FARMER_CHEST_BE_TYPE`, `FARMER_ROBOT_SPAWN_ITEM`).
- `FarmerRobotEntities.bootstrap()` — entity type + `createAttributes()` (health, hareket hızı vb.) tanımı; `WorldForgeMod.onInitialize()` içinde çağrılıyor.
- `FarmerRobotMenus.register()` — menü tipi kaydı.

## 10. İstemci Tarafı (`farmer/client`)

`FarmerClientInit` (ClientModInitializer) → `FarmerRobotGeoModel` + `FarmerRobotGeoRenderer` (GeckoLib render) + `FarmerRobotScreen` (GUI ekranı, `FarmerRobotMenu`'yü görselleştirir).

## 11. Genişletme Kontrol Listesi

Bu projede benimsenen çalışma tarzına göre (bkz. bellek notları), her yeni faz için:
1. Etkilenen tüm dosyaları önce oku.
2. `str_replace` ile cerrahi düzenleme yap.
3. Değişiklik sonrası JSON/brace dengesini doğrula.
4. `grep` ile sembol bağlama kontrolü yap (yeni goal/enum değeri her yerde doğru referanslanıyor mu).
5. Enum'a **sona ekle**, asla araya sokma (ordinal stabilitesi).
6. Javadoc'a faz numarası ve çapraz link ekle (`{@link ...}`).
7. 5 dilin tamamında (`en_us, de_de, es_es, fr_fr, tr_tr`) çeviri anahtarlarını güncelle.
8. Kapsam dışı ön-var olan sorunları sessizce düzeltme, sadece not düş.
