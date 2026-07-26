# Çekirdek Mimari — wfbuild (Prosedürel Yapı Motoru)

Paket: `com.worldforge.wfbuild.*` · ~78 dosya + ~400 stil JSON'ı + ~35 dekorasyon teması

## 1. Felsefe

**"Sınırsız çeşitlilik, sınırsız veri olmadan."** Sabit bir bina şablonu kütüphanesi tutmak yerine, aynı `BuildIntent`, farklı bir seed ile besleneninde her seferinde farklı ama her zaman geçerli bir sonuç üretir. Bunu **13 aşamalı, sıkı sorumluluk ayrımlı bir boru hattı** ile başarır — her aşama yalnızca bir önceki aşamanın çıktısını bilir, sonraki aşamanın nasıl çalıştığını asla bilmez.

## 2. Boru Hattı (Uçtan Uca)

```
Metin girdisi
   │
   ▼
 1. Intent               IntentEngine (KeywordIntentParser → LlmIntentParser yedek)
   │                      ──► BuildIntent (jenerik, Minecraft-tipsiz)
   ▼
 2. Blueprint             BlueprintEngine
   │                      dimensions → room graph → openings → roof → decorations
   │                      ──► Blueprint (hâlâ Minecraft-tipsiz)
   ▼
 3. Architecture          ArchitectureEngine.materialize(blueprint)
   │                      ──► Map<BlockPos, BlockState> sözleşmesi (yerel orijinli)
   │
   │   ┌─── malzeme çözümleme alt-zinciri (Architecture'ın Palette bağımlılığı) ───┐
   │   │  4. Style              StyleEngine.resolveBase(style, buildingType)       │
   │   │  5. Theme               MaterialThemeRegistry (haunted/volcanic/gothic)   │
   │   │  6. Material Resolver   MaterialResolver.applyTheme(style, base)          │
   │   │  7. Palette             PaletteEngine.resolve(style, buildingType[,biome])│
   │   └──────────────────────────────────────────────────────────────────────────┘
   │
   │   ┌─── dekorasyon alt-zinciri ──────────────────────────────────────────┐
   │   │  8. Decoration Resolver  DecorationVariantLoader                     │
   │   │  9. Decoration Variants  DecorationVariantRegistry (JSON temalar)   │
   │   └───────────────────────────────────────────────────────────────────┘
   │
   ▼
10. Resource Logic        resourcelogic (ResourceRuleManager) — malzeme/blok kuralları
   ▼
11. Validation            ConstructionValidationEngine — "WorkChest'te yeterli malzeme var mı?"
   ▼
12. Planner               ConstructionPlanner.plan(blueprint) → ConstructionPlan (fazlara bölünmüş)
   ▼
13. Builder               FoundationBuilder, FrameBuilder, OpeningCarver, ColumnBuilder,
                           RoofBuilder, StairBuilder, BalconyBuilder, DecorationBuilder
                           (+ FeatureManager: ev bittikten sonra peyzaj öğeleri)
   ▼
Gerçek dünya yerleştirmesi
```

Aşama 3 (Architecture) aslında 4-9'u (malzeme+dekorasyon çözümleme) ve 10-13'ü (kaynak kuralları → doğrulama → planlama → inşa) kendi içinde çağıran bir **orkestratördür** — `ArchitectureEngine.materialize()` çağrıldığında zincirin geri kalanı otomatik tetiklenir. Aşağıdaki bölümler her aşamayı ayrı ayrı derinlemesine anlatıyor.

## 3. Aşama 1 — Intent Engine (`wfbuild/intent`)

**Görev:** Serbest metni, tamamen jenerik bir isteğe (`BuildIntent`) çevirmek. Bu paketteki hiçbir sınıf `BlockPos` veya `BlockState` bilmez.

```java
record BuildIntent(
    String type, String style, int floors, Size size, RoofType roof,
    int windowCount, int doorCount, Set<String> constraints,
    Map<String, String> attributes, Set<String> siteFeatures
)
```

- Kompakt constructor'da **tüm alanlar için sane default** var: `type` boşsa `"house"`, `size` null ise `MEDIUM`, `roof` null ise `AUTO`, sayılar negatifse `UNSPECIFIED(-1)`. Bu sayede sadece birkaç kelimeyi tanıyan bir parser bile geçerli, inşa edilebilir bir intent üretir.
- `type` ve `style` **bilerek enum değil, serbest string**: yeni bir bina türü/stil eklemek bu sınıfa, parser'a veya herhangi bir enum'a dokunmayı gerektirmez — sadece sonraki aşamaların bilmesi gereken bir kayıt/registry büyür.

**`IntentEngine` çözümleme sırası:**
1. `KeywordIntentParser` her zaman önce çalışır — senkron, çevrimdışı, sıfır maliyetli sözlük+preset eşleştirme.
2. Sadece o boş dönerse VE bir LLM yapılandırılmışsa (`CatAskConfig.llmEnabled()`) `LlmIntentParser` off-thread olarak devreye girer.

`resolveAsync()` (tam çözümleme) ve `resolveOffline()` (sadece anahtar kelime, gerçek zamanlı yollar için) iki ayrı giriş noktası sunar. LLM'in neden opsiyonel bir yedek olduğuna dair gerekçe için bkz. `design-decisions.md`.

`intent/semantics/` (`SemanticDictionaries`, `PresetDefinition`) `data/worldforge/wfbuild/semantics/*.json` dosyalarından (presets, structure, size, constraint, decoration, roof, style) sözlükleri yükler.

## 4. Aşama 2 — Blueprint Engine (`wfbuild/blueprint`)

**Görev:** `BuildIntent`'i tam, jenerik bir `Blueprint`'e çevirmek — hâlâ hiçbir Minecraft bloğu yok. Sabit bir pipeline çalıştırır:

```
DimensionPlanner → RoomPlanner → OpeningPlanner (kapı+pencere) → RoofPlanner → DecorationPlanner
```

Her adım sadece intent'i, türün küçük temel-boyutlandırma profilini (`BuildingTypeProfile`) ve paylaşılan seed'li bir `Random`'ı okur. **"Aynı intent + farklı seed = farklı ama geçerli plan"** deseni tam olarak burada gerçekleşir:

```java
Blueprint generate(BuildIntent intent)             // rastgele seed
Blueprint generate(BuildIntent intent, long seed)   // deterministik — aynı çift her zaman aynı planı üretir
```

Veri taşıyıcı record'lar (hepsi immutable): `Dimensions`, `Room`, `RoomGraph`, `RoomConnection`, `DoorSpec`, `WindowSpec`, `RoofSpec`, `DecorationSlot`, `Point3I`, `BoxRegion`, `Side` (enum).

## 5. Aşama 3 — Architecture Engine (`wfbuild/architecture`)

**Görev:** Bir `Blueprint`'i somut `Map<BlockPos, BlockState>`'e çevirmek, eski `StructureTemplates`'in ürettiğiyle **aynı sözleşmeyle** (yerel (0,0,0) orijinli) — böylece mevcut çağıranlar (WorkChest malzeme sayımı, `TickedTaskScheduler`, undo/chunk-safety) değişmeden çalışmaya devam eder.

```java
static Map<BlockPos, BlockState> materialize(Blueprint blueprint) {
    Palette palette = PaletteEngine.resolve(blueprint.style(), blueprint.type());  // 4-7. aşamalar
    ConstructionPlan plan = ConstructionPlanner.plan(blueprint, palette);          // 10-12. aşamalar
    return plan.flatten();                                                        // 13. aşama (Builder'lar)
}
```

**Kritik tasarım kuralı:** Her üretici (Builder) çıktısını **yalnızca** blueprint'in kendi geometrisinden (dimensions, room graph, kapı/pencere/çatı/dekorasyon specs) + çözümlenmiş Palette + blueprint'in seed'li Random'ından türetir. Hiçbiri kayıtlı bir şablon yüklemez, hiçbiri önceden tanımlı bir "ev" şekli gerektirmez. Daha önce hiç görülmemiş bir oda grafiği, görülmüş biriyle tamamen aynı kurallarla çerçevelenir.

### `ConstructionPhase` (enum)
```
FOUNDATION, WALLS, COLUMNS, ROOF, INTERIOR, DECORATIONS
```
`StructureRules` kural seti, bir yapı türü için bunlardan hangilerinin hangi sırayla uygulanacağına karar verir; `ConstructionPlanner` (motor) hiçbir zaman yapı türüne göre dallanmaz — sadece kural setinin verdiği sırayı izler.

## 6. Aşama 4 — Style (`wfbuild/style`)

**Görev:** Bir (style, buildingType) çiftinin **temel** malzeme atamasını çözer — tema/biyom katmanlarından önceki ham eşleme.

```java
MaterialPalette resolveBase(String style, String buildingType) {
    return StyleDefinitions.REGISTRY.lookup(style, buildingType);
}
```

`data/worldforge/wfbuild/styles/` altında **~400 JSON dosyası**, "sıfat + yapı-türü" kombinasyonu üretiyor (ashen/azure/blazing/bronze/celestial/charred/amber/autumn/... × spire/boathouse/tavern/keep/cottage/...).

## 7. Aşama 5 — Theme (`wfbuild/style/material_themes`)

**Görev:** Stilin üstüne **isteğe bağlı** bir atmosfer/tema katmanı bindirmek: `haunted`, `volcanic`, `gothic`. Bir stil hiçbir temaya bağlı değildir — tema, sonradan uygulanan bir overlay'dir, stilin kendisinin parçası değildir.

`MaterialThemeRegistry`, `data/worldforge/wfbuild/material_themes/*.json` dosyalarından hangi alanların (`wall`, `roof`, `accent` vb.) hangi temada neyle değiştirileceğini yükler. Bir stilin **hangi temalarla eşleştirilebileceğine** dair kısıtlama yok — herhangi bir stil + herhangi bir tema kombinasyonu geçerlidir. Neden Theme'in Style'dan **türetildiğine** (yeni bir kavram olarak yeniden icat edilmediğine) dair gerekçe `design-decisions.md`'de.

## 8. Aşama 6 — Material Resolver (`wfbuild/style`)

**Görev:** Aşama 4'ün temel paletini, Aşama 5'in tema tanımıyla **birleştirmek (overlay)** — asla toptan değiştirmemek.

```java
MaterialPalette applyTheme(String style, MaterialPalette base) {
    ThemeDefinition theme = MaterialThemeRegistry.themeFor(style);  // yoksa base aynen döner
    return base.withOverrides(theme.fieldOverrides());              // sadece temanın belirttiği alanlar değişir
}
```

`MaterialResolver` **overlay mantığında** çalışır — temanın belirtmediği her alan, temel paletten olduğu gibi kalır. Bu tasarımın gerekçesi `design-decisions.md`'de detaylandırılıyor.

`BiomeStyleModifier` bu zincirin bir sonraki (biyom) katmanı olarak devreye girer (ör. çöl → kumtaşı), `/wfconfig set style biomemodifierenabled false` ile kapatılabilir.

## 9. Aşama 7 — Palette (`wfbuild/palette`)

**Görev:** Aşama 6 HANGİ bloğun her malzeme rolünü temsil ettiğine karar verir ("ortaçağ duvarları taş tuğla"); Palette bu blokların NEREDE yerleştirilmesine izin verildiğine karar verir.

```java
Palette resolve(String style, String buildingType)                    // temel
Palette resolve(String style, String buildingType, String biomeTag)   // + biyom override
```

Çözümlenmiş `MaterialPalette`'i bir `Palette`'e sarar; bu `Palette`'in blok vermenin **tek** yolu `blockFor(PlacementContext)`. Builder'lar (Aşama 13), ham `MaterialPalette`'i asla görmez — her üretici bir bağlam adlandırır ("şu an bir WALL bloğu yerleştiriyorum"), bir blok torbasına elini sokup umut etmez.

## 10. Aşama 8 — Decoration Resolver (`wfbuild/architecture/decoration`)

**Görev:** Bir yapı türü + tema kombinasyonu için hangi dekorasyon varyant setinin geçerli olduğunu çözer, JSON'dan yükler.

```
RawDecorationVariant → DecorationVariantLoader → DecorationVariantRegistry
```

## 11. Aşama 9 — Decoration Variants

`data/worldforge/wfbuild/decorations/` altında **~35 kültürel/estetik tema** (japanese, gothic, cyberpunk, victorian, vb.) — her biri kendi dekorasyon öğesi setini (raf, halı, aydınlatma, duvar süsü vb.) tanımlar. `DecorationBuilder` (Aşama 13) bu registry'den bağlama uygun varyantı çeker.

## 12. Aşama 10 — Resource Logic (`resourcelogic`)

**Görev:** Malzeme/blok davranış kurallarını (hangi blok neyle etkileşir, hangi akışkan kuralı geçerli) **veri odaklı** (JSON) sağlamak. `ResourceRuleManager`, `ResourceContext`, `ResourceOperation`, `ResourceType` — kural + operasyon eşlemesi. `resourcelogic`'in neden koda gömülü yerine veri odaklı tutulduğuna dair gerekçe `design-decisions.md`'de.

Detaylı mimari için bkz. `resourcelogic-and-builder-op.md`.

## 13. Aşama 11 — Construction Validation Engine (`wfbuild/validation`)

Blok-haritası üretimi ile gerçek yerleştirme **arasında** oturur. **Tek sorusu:** "Oyuncunun WorkChest'i bu inşaatın gerektirdiği her şeyi zaten tutuyor mu?"

**İstisnasız katı kurallar:**
- Asla bir malzemeyi başka biriyle **ikame etmez**.
- Asla eksikliği kapatmak için bir şey **craftlamaz**.
- Asla oyuncunun kişisel envanterinden, ender chest'ten veya başka bir konteynerden **çekmez** — sadece WorkChest sayılır.
- Asla eldekine uyması için blueprint/blok haritasını **düzenlemez**.

## 14. Aşama 12 — Planner (`wfbuild/architecture/ConstructionPlanner`)

**Görev:** Doğrulanmış blueprint + palette'i, adlandırılmış inşaat fazlarına (`ConstructionPhase`) bölünmüş sıralı bir `ConstructionPlan`'a çevirmek. Motor kendisi hiçbir zaman yapı türüne göre dallanmaz — `StructureRules`'ın verdiği faz sırasını izler.

## 15. Aşama 13 — Builder

`FoundationBuilder`, `FrameBuilder`, `OpeningCarver`, `ColumnBuilder`, `RoofBuilder`, `StairBuilder`, `BalconyBuilder`, `DecorationBuilder` — planı gerçek `BlockPos → BlockState` eşlemesine döken somut üreticiler. Ev tamamen inşa edildikten **sonra** `FeatureManager` (`wfbuild/feature`) devreye girer: `PoolGenerator`, `KoiPondGenerator`, `GardenGenerator`, `PathGenerator`, `FenceGenerator`, `LampPostGenerator`, `WellGenerator`, `FountainGenerator`, `StatueGenerator`, `BambooGardenGenerator`, `CampfireAreaGenerator`, `FlowerBedGenerator` gibi bağımsız peyzaj öğesi üreticileriyle bloklar birleştirilir.

`FeatureManager.applyFeatures(houseBlocks, blueprint, rng, player)` her `blueprint.siteFeatures()` etiketi için `FeatureGeneratorRegistry`'den ilgili üreticiyi bulur. **Yeni bir peyzaj öğesi eklemek** = yeni bir `FeatureGenerator` yazıp kaydetmek; `FeatureManager`'ın kendisi değişmez.

## 16. Yükleme Deseni — Raw → Loader → Registry

Bu zincirdeki (ve `resourcelogic`'teki) **her** JSON-tanımlı sistem aynı üçlü desenle kurulu:

```
Raw*Definition (ham JSON DTO, Gson field eşlemesi)
      │  parse
      ▼
*Loader (dönüşüm/doğrulama/varsayılan uygulama)
      │  yükle
      ▼
*Registry (çalışma zamanı, salt-okunur erişim noktası)
```

Örnekler: `RawStyleDefinition → MaterialThemeLoader → MaterialThemeRegistry`, `RawDecorationVariant → DecorationVariantLoader → DecorationVariantRegistry`, `RawMaterialThemeOverlay → StyleJsonLoader → StyleDefinitions`. JSON kalıtımının (`extends`) neden bu yükleme deseninin bir parçası olduğuna dair gerekçe `design-decisions.md`'de.

## 17. Giriş Noktası — `CatBuildCommand`

`/wfbuild` komutu bu tüm zinciri şu sırayla çağırır:
```
IntentEngine.resolve*(text)                              [Aşama 1]
   → BlueprintEngine.generate(intent[, seed])             [Aşama 2]
   → ArchitectureEngine.materialize(blueprint)             [Aşama 3, içinde 4-13 zinciri]
   → gerçek dünya yerleştirmesi
```
LLM tabanlı alternatif yol (`CatBuildLlmService`, bkz. `catbot.md`) bu zincirin dışında, paralel bir yoldur ve başarısız olursa **bu** zincire (çevrimdışı `IntentEngine` yoluyla) düşer.

## 18. Genişletme Kontrol Listesi

- **Yeni yapı türü** → `BuildingTypeProfile`'a ekle + gerekirse `StructureRulesRegistry`'ye kural seti. `BuildIntent.type` bir string olduğu için enum değişikliği gerekmez.
- **Yeni stil** → `data/worldforge/wfbuild/styles/` altına `sıfat_yapıTürü.json` desenine uygun bir dosya + `StyleDefinitions`'a kayıt. Architecture/Planner/Builder aşamalarına dokunma.
- **Yeni tema** → `data/worldforge/wfbuild/material_themes/` altına yeni JSON + `MaterialThemeRegistry`'ye kayıt. Style aşamasına dokunma.
- **Yeni dekorasyon varyantı** → `data/worldforge/wfbuild/decorations/` altına yeni tema JSON'ı + `DecorationVariantRegistry`'ye kayıt.
- **Yeni peyzaj öğesi** → yeni bir `FeatureGenerator` implementasyonu + `FeatureGeneratorRegistry`'ye kayıt. `FeatureManager`'a dokunma.
- **Yeni inşaat fazı davranışı** → `ConstructionPhase` enumuna **sona ekle**, ilgili `*Operation`/`*Builder` sınıfını yaz, `StructureRules.phaseOrder()`'a dahil et.
- **Yeni bir resourcelogic kuralı** → JSON'a `extends` ile mevcut kuraldan türet; kod değişikliği gerekmeyebilir.
