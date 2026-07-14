# Çekirdek Mimari — İş Motoru (TickedTaskScheduler)

Paket: `com.worldforge.placement` (1 dosya, ama modun **merkezi orkestratörü**)

## 1. Neden Ayrı Bir Doküman?

`TickedTaskScheduler`, dosya sayısı olarak küçük bir paket olsa da, `/wffill`, `/wfbreak`, `/wfbuild` gibi tüm **uzun süren, bloklara dokunan işlemlerin** kalbi. Aşağıdaki 15+ alt sistemi tek bir tick-döngüsünde birleştirir: `CatBotEntity/Manager/State`, `WorkChestManager`, `ChunkSafetyMonitor`, `DestructionEngine`/`DurabilityHandler`/`ToolRequirement`/`ToolMatchResult`, `EconomyConfig`/`VaultEconomyBridge`, `FluidHandlingMode`/`FluidHandler`/`FluidStateTracker`, `BlueprintStore`/`HologramSender`, `Lang`, `RegionParticleRenderer`, `Region`, `OmnitoolRepairSystem`, `BlockChangeRecord`/`UndoManager`, `ItemDropMerger`.

## 2. Temel Model: Server-Tick İş Kuyruğu

```java
final class TickedTaskScheduler {
    static final int DEFAULT_OPS_PER_TICK = 200;      // her tick'te işlenen blok sayısı
    static final int MERGE_EVERY_BLOCKS   = 50;        // ItemDropMerger tetik aralığı
    static final int GIANT_TOOL_SWING_EVERY_TICKS = 4; // görsel efekt throttle
    static final int HOLOGRAM_REFRESH_EVERY_TICKS = 10;// hologram önizleme throttle

    Map<UUID, WFJob> activeJobs;   // oyuncu başına tek aktif iş
}
```

**Neden throttle edilmiş sabitler var:** 200 ops/tick'lik bir iş, her tick'te 200 entity/parçacık/hologram spawn ederse sunucuyu kilitler. Bu yüzden:
- Blok işleme **her tick** olur (performans için gerekli),
- ama **görsel geri bildirim** (dev alet sallanması, hologram, item birleştirme) çok daha seyrek aralıklarla tetiklenir.

Bu, "gerçek iş her tick, görsel/yan-etki işi seyrek" ayrımı — büyük hacimli işlemlerde tekrar kullanılabilir bir performans deseni.

## 3. Bir İşin (WFJob) Hayat Döngüsü

```
submitRegionJob(player, region, opsPerTick, fillState, isBreakMode)
   │
   ▼ her tick (ServerTickEvents.END_SERVER_TICK)
1. DestructionEngine / ToolRequirement / ToolMatchResult — doğru alet var mı, dayanıklılık yeterli mi?
2. OmnitoolRepairSystem — WorkChest'teki omnitool'ları öncelikli kullan, gerekirse tamir et
3. FluidHandler / FluidHandlingMode / FluidStateTracker — akışkan blokları özel işle
4. BlockChangeRecord üret → UndoManager'a devret (undo geçmişi)
5. Her MERGE_EVERY_BLOCKS'ta bir → ItemDropMerger (düşen item'ları birleştir, FPS düşüşünü önle)
6. Her GIANT_TOOL_SWING_EVERY_TICKS'te bir → GiantToolEffect (kozmetik dev alet sallama)
7. Her HOLOGRAM_REFRESH_EVERY_TICKS'te bir → HologramSender.sendPreview (canlı önizleme küçülür)
8. RegionParticleRenderer → seçim/iş sınırını göster
9. ChunkSafetyMonitor.register() ile kaydedilen chunk'lar boşaltılırsa → iş otomatik duraklatılır (bkz. §4)
10. Economy — iş ücreti tahsili/iadesi (VaultEconomyBridge)
```

## 4. ChunkSafetyMonitor — Büyük İş Güvenlik Ağı

**Sorun:** Bir oyuncu 100.000+ bloklu bir alanı `/wfdestroy` veya `/wffill` ile silerse ve iş sırasında bir chunk boşalırsa (oyuncu uzaklaşırsa), iş kesintiye uğrayabilir; ayrıca normal per-tick undo kaydı (bkz. `UndoManager`) bellek sınırı nedeniyle **büyük işler için kasıtlı olarak kaydı erken durdurur**.

**Çözüm — iki ayrı güvenlik mekanizması:**
```java
static final int BIG_JOB_SNAPSHOT_THRESHOLD = 100_000;  // blok sayısı eşiği

record SnapshotEntry(BlockPos pos, BlockState before);   // tek pozisyonun iş-öncesi durumu
```
- Kuyruğu bu eşiğin **üzerinde/eşit** olan her iş, `takeBigJobSnapshot()` ile **tam bir "öncesi durum" anlık görüntüsü** alır — normal per-tick undo yakalamasının yerine geçer. Bu, bellek sınırını (tick başına sınırlı kayıt) tek seferlik büyük bir tahsisle takas eder, **sırf** bir oyuncu kazayla devasa bir alanı yok ederse `/wfundo` ile hâlâ geri alabilsin diye.
- `ServerChunkEvents.CHUNK_UNLOAD` dinlenir: bir işin dokunduğu chunk'lardan biri boşalırsa (`jobChunks` haritasında kayıtlıysa), o iş `pausedByUnload` setine eklenir ve **otomatik duraklatılır** — sessizce devam edip yüklenmemiş chunk'ta hatalı işlem yapmaz.

`ChunkPos` MC 26.1+'da bir Java record; kanonik erişim `cp.x()`/`cp.z()` (kod içi not, önemli bir MC sürüm uyumluluk detayı).

## 5. Hologram / Önizleme Sistemi (`hologram/`)

```java
BlueprintStore    // duraklatılmış işlerin KALAN blok pozisyonlarını tutar (/wfblueprint)
HologramSender    // gerçek Display.BlockDisplay entity'leri ile hayalet blok gösterir
```

**İki farklı kullanım, aynı render kodu:**
- `sendBlueprint(ServerPlayer)` — **duraklatılmış** bir işin önizlemesi, doğrudan `BlueprintStore`'dan okunur (`/wfblueprint info|clear`, `/wfresume` ile ilişkili).
- `sendPreview(ServerPlayer, List)` — **çalışan** bir `/wfbuild` inşaatının **canlı** önizlemesi; `TickedTaskScheduler` bloklar gerçekten yerleştikçe tekrar tekrar çağırır, hologram küçüle küçüle gerçek bloklara dönüşür. Bu yol **asla** `BlueprintStore`'a dokunmaz — o depo modun geri kalanı için "duraklatılmış, devam ettirilebilir iş" anlamına gelir; aktif bir build ne duraklatılmış ne de devam ettirilmeyi bekliyordur.

**Teknik detay:** Vanilla `Display.BlockDisplay`'in transform/block-state setter'ları MC 26.2'de paket-özel (package-private) olduğu için, `GiantToolEffect` ile **aynı NBT-yükleme tekniği** kullanılıyor (doğrudan setter çağrısı yerine NBT compound inşa edip entity'ye yüklemek). Gerçek alfa saydamlığı client-mod'suz mümkün olmadığından, renkli parlama anahat (yeşil = yerleştirilecek, kırmızı = kaldırılacak) + tam parlaklık + gölgesiz kullanılıyor — vanilla araçlarla, kurulum gerektirmeden, duvarların arkasından bile net okunan bir önizleme.

Her hayalet `GHOST_TAG` ile işaretlenir — bu etiket `WorldForgeMod`'un üçüncü güvenlik ağı (chunk yüklenince hayalet temizleme) tarafından kullanılır (bkz. ana `ARCHITECTURE.md` §5).

## 6. Diğer Yardımcı Sistemler

| Sınıf | Amaç |
|---|---|
| `ItemDropMerger` | Bir konum etrafındaki aynı türden düşen item entity'lerini tarar, tip+component eşitliğine göre gruplar, ilk entity'ye birleştirir (yığın limitine kadar), fazlalıkları siler — yüzlerce ayrı item entity'sinin FPS düşürmesini önler. |
| `GiantToolEffect` | Salt kozmetik: `/wffill`/`/wfbreak` sırasında işlenen blokta büyütülmüş, sallanan bir alet (`Display.ItemDisplay`) gösterir. Display entity'leri vanilla olduğu için otomatik her istemciye replike olur — istemci kodu/mixin gerekmez. **Bug fix:** bu entity'ler `active` listesindeki bir iç sayaçla (8 tick) kendi kendine silinsin diye tasarlanmıştı, ama gerçek vanilla entity oldukları için bir chunk tam o 8 tick içinde otomatik kayıt alır (autosave) ya da sunucu çökerse (SERVER_STOPPING atlanırsa) diskte kalıcı olarak saklanabiliyorlardı — "iş bitince havada asılı kalma" hatasının kaynağı buydu. Artık `HologramSender.GHOST_TAG` ile aynı desende bir etiketle (`GiantToolEffect.isGiantTool`) işaretleniyorlar ve `WorldForgeMod`'un ENTITY_LOAD üçüncü güvenlik ağı, hologram hayaletleriyle aynı şekilde bunları da chunk yüklenince tanıyıp siliyor. |
| `RegionParticleRenderer` | Seçili/işlenen bölgenin 12 kenarı boyunca `HAPPY_VILLAGER` parçacığıyla tel çerçeve çizer; `SEND_EVERY_TICKS=5` ile throttle edilir, iş bitince/iptal edilince kaldırılır. |
| `FluidHandler`/`FluidHandlingMode`/`FluidStateTracker` | Akışkan (su/lava) bloklarının fill/break işlemlerinde nasıl ele alınacağını yönetir. |
| `PermissionChecker` | **Şu an tüm kontrolleri `true` döndürüyor** — MC 26.2 uyumluluğu için izin uygulaması devre dışı bırakılmış (kod içi not: "enforcement disabled for MC 26.2 compatibility"). `NODE_USE`/`NODE_ADMIN` sabitleri gelecekte gerçek bir izin sistemine bağlanmak üzere korunuyor. |

## 7. i18n Entegrasyonu — Otomatik Dil Algılama

`Lang` sınıfı **varsayılan olarak otomatik**: her oyuncunun komut/mesaj dili, Minecraft istemci dil ayarını takip eder (resource-pack `lang/*.json` seçiminde kullanılanla aynı kaynak) — `/wflang` çağrısına gerek yok, item/tooltip çevirileri gibi "otomatik eşleşir". Oyuncular `/wflang <code>` ile manuel geçersiz kılabilir; bu oturum boyunca (veya `/wflang auto`'ya kadar) kalıcıdır ve otomatik algılamadan **önceliklidir**.

## 8. Genişletme Notları

- Yeni bir uzun-süren iş türü eklerken `TickedTaskScheduler`'a entegre et — kendi tick döngünü yazma; mevcut throttle sabitlerini (ops/tick, merge/swing/hologram aralıkları) yeniden kullan.
- İş büyük hacimli olabilirse (`BIG_JOB_SNAPSHOT_THRESHOLD` sınırını aşabilirse) `ChunkSafetyMonitor` entegrasyonunu atlamamaya dikkat et — aksi halde undo/chunk-unload güvenliği o iş türü için çalışmaz.
- Yeni bir görsel geri bildirim eklerken (parçacık, hologram, kozmetik entity) **mutlaka bir throttle sabiti** tanımla — her tick'te tetiklenen bir görsel efekt sunucu performansını doğrudan etkiler.
- `PermissionChecker` şu an devre dışı — yeni bir komut/özellik yazarken izin kontrolü eklemek istersen bu sınıfın gerçek bir izin sistemine (LuckPerms vb.) bağlanmasının şu an **kasıtlı olarak** yapılmadığını unutma; `NODE_USE`/`NODE_ADMIN` sabitlerini kullan ama davranışın şu an herkese açık olduğunu bil.
