# Çekirdek Mimari — WorldEdit-Tarzı Araçlar & Komut Yüzeyi

Paket: `com.worldforge.brush` (1) · `fill` (1) · `shape` (1) · `terrain` (1) · `book` (1) · `command` (1)

## 1. İki Yol Ayrımı: İş Kuyruğu vs. Anlık İşlem

Bu paketlerin ortak teması, **her blok işleminin `TickedTaskScheduler` kuyruğuna girmediği** açık tasarım kararı:

| Yol | Ne zaman | Örnek |
|---|---|---|
| **İş kuyruğu** (`TickedTaskScheduler`) | Büyük bölge operasyonları | `/wffill`, `/wfbreak`, `/wfexpand`, `/wfbuild` |
| **Anlık/senkron** (aynı tick'te, kuyruğa girmeden) | Küçük, tek-tıklık dokunuşlar | `/wfsmooth` (`TerrainSmoother` doğrudan çağrılır), fırça darbeleri (`BrushTool`) |

`BrushTool`'un kendi Javadoc'u bunu açıkça gerekçelendiriyor: *"İş sistemi büyük bölge operasyonları için ayrılmış kalır; fırça, 'bir WorkChest spawn et / iş bekle' törenine gerek kalmadan hızlı, tek-tıklık arazi dokunuşları içindir."*

## 2. BrushTool — Omnitool Fırça Modu

**Tetik:** Sneak + sağ-tık (Omnitool ile).

**Mod otomatik seçimi** — vanilla'nın ana-el-alet / boş-el-materyal geleneğini yansıtır:
```
Boş el bir yerleştirilebilir blok tutuyor → SPHERE modu
    → tıklanan noktaya merkezli küçük, dolu bir küre boyar o blokla
    → survival'da her değişen blok için 1 item envanterden düşer, creative'de bedava

Boş el boşsa (veya blok değilse) → SMOOTH modu
    → tıklanan sütun merkezli küçük bir kutu üzerinde TEK bir çoğunluk-oyu
      (majority-vote) heightmap düzeltme geçişi (TerrainSmoother çağrılır)
```

- Her iki mod da: elde tutulan Omnitool'dan **1 dayanıklılık** harcar, **tamamen geri alınabilir** (`UndoManager` üzerinden kaydedilir — `/wfsmooth` ve `/wfshape` ile aynı şekilde).
- **Spam koruması:** `COOLDOWN_TICKS = 4L` — aynı oyuncudan art arda iki fırça darbesi arasında minimum tick sayısı.
- Yarıçap sınırı: **≤ 3** — bilinçli olarak küçük tutuluyor, büyük alanlar için `/wfshape`/`/wffill` kullanılmalı.

## 3. ShapeGenerator — Prosedürel Geometri Kütüphanesi

`/wfshape` komutunun tüm alt komutlarını besleyen saf geometri üretici (state'siz, sadece `List<BlockPos>` üretir). **v3 itibariyle 17 şekil:**

**Orijinal 5 primitif:** sphere, cylinder, cone, pyramid, triangle
**v3'te eklenen 12 gelişmiş/bileşik "build" şekli:** tower, dome, archway, staircase, spiralStaircase, bridge, wall, gazebo, obelisk, helix, platform, watchtower

Tüm şekiller **tek bir orijin noktasına** (merkez/taban bloğu) göre çapalanır — çağıran kod (komut katmanı) sadece merkez + parametreleri verir, `ShapeGenerator` blok listesini döner; gerçek yerleştirme `TickedTaskScheduler` kuyruğu üzerinden yapılır.

## 4. TerrainSmoother — Çoğunluk-Oyu Heightmap Düzeltme

WorldEdit'in `//smooth` komutuna ruhen benziyor, ama **ağırlıklı ortalama yerine çoğunluk-oyu (mode) filtresi** kullanıyor:

```
Bölgedeki her (x,z) sütunu için:
  1. Yüzey yüksekliği = bölgenin Y aralığı içindeki en üstteki hava-olmayan blok
  2. Her sütunun yeni yüksekliği = komşu sütunlar arasında en sık görülen (mode) yükseklik
```

`BlockChangeRecord` üretir — `/wfsmooth` doğrudan senkron çalışsa da yine `UndoManager`'a kaydedilir (bkz. §1 tablosu, "anlık" yol da geri alınabilirlik gerektirir).

## 5. BlockPattern — Ağırlıklı Blok Deseni Ayrıştırıcı

`/wffill` için WorldEdit-tarzı sözdizimini ayrıştırır:
```
"stone"                    → tek blok
"stone,dirt"                → eşit ağırlıklı karışım
"stone:70,dirt:20,glass:10" → ağırlıklı rastgele karışım (weighted random)
```
`List<BlockState> states` + `List<Integer> weights` + `totalWeight` ile temsil edilir; `next(Random)` çağrısı ağırlıklara göre rastgele bir `BlockState` döner. `/wffill`'in "solid/air/hollow/replace" varyantlarının hepsi bu sınıfı kullanır.

## 6. WFTutorialBook — Uygulama İçi Rehber

`ItemStack` olarak yazılabilir bir kitap üretir (MC 26.2 `DataComponents` API'si, `WrittenBookContent`). Oyuncuya komut yüzeyinin tamamını (aşağıdaki §7) oyun içinde, ClickEvent/HoverEvent ile tıklanabilir komut önerileriyle sunar.

## 7. Komut Yüzeyi (`WorldForgeCommands`) — Tam Envanter

Tüm komutlar tek bir `CommandDispatcher` kaydında toplanıyor (`WorldForgeMod.registerCommands()` içinden çağrılır). Kategorilere göre:

**Seçim:** `/wfpos1`, `/wfpos2`, `/wfselection`
**Görsel seçim:** `/wfimgpos1`, `/wfimgpos2`, `/wfimgselection`
**Bölge dönüşümü:** `/wfexpand`, `/wfcontract`, `/wfshift`, `/wfclear`
**Doldurma:** `/wffill`, `/wffillsolid`, `/wffillair`, `/wfhollow`, `/wfreplace`
**Şekil (17 alt komut):** `/wfshape sphere|cylinder|cone|pyramid|triangle|tower|dome|archway|staircase|spiralstaircase|bridge|wall|gazebo|obelisk|helix|platform|watchtower`
**Yıkım:** `/wfbreak`
**Çizgi/eğri:** `/wfline`, `/wfcurve`
**Arazi:** `/wfsmooth`
**İş kontrolü:** `/wfcancel`, `/wfresume`, `/wfundo` (+ muhtemelen `/wfredo`, dosyanın görünen kısmında yok)

(Ayrıca bu dosyanın dışında ama aynı komut ailesinden: `/wfbuild`, `/wfschematic`, `/wfcopy`/`/wfpaste`, `/wfcat`, `/wfconfig`, `/wflang` — sırasıyla `catbot/build`, `schematic`, `clipboard`, `catbot`, `config`, `i18n` paketlerinde kayıtlı.)

## 8. Genişletme Notları

- Yeni bir "küçük, anlık dokunuş" özelliği eklerken `TickedTaskScheduler` kuyruğuna sokma — `BrushTool`/`TerrainSmoother` deseni gibi doğrudan senkron uygula, ama **yine de `UndoManager`'a kaydet**.
- Yeni bir şekil eklerken `ShapeGenerator`'a saf, state'siz bir statik metot ekle (`List<BlockPos>` döner) — yerleştirme mantığına karışma, bu iş kuyruğunun işi.
- Yeni bir komut eklerken `WorldForgeCommands`'a `d.register(Commands.literal(...))` ile ekle ve `WFTutorialBook`'u güncellemeyi unutma — kitap komut yüzeyinin canlı bir referansı, yeni komutlar eklenince senkron kalmalı.
