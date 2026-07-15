<div align="center">

# ⚒️ WorldForge

**Hayatta kalma moduna uygun, dev ölçekli bir dünya düzenleme ve otomasyon araç seti.**

Fabric • Minecraft 26.2 • Java 25 • GeckoLib 5

[![License: CC0-1.0](https://img.shields.io/badge/license-CC0--1.0-lightgrey.svg)](LICENSE)
[![Fabric](https://img.shields.io/badge/mod%20loader-Fabric-3f4e5a)](https://fabricmc.net)
[![Minecraft](https://img.shields.io/badge/minecraft-26.2-brightgreen)](https://minecraft.net)

</div>

---

## İçindekiler

- [WorldForge Nedir?](#worldforge-nedir)
- [Öne Çıkan Özellikler](#öne-çıkan-özellikler)
  - [Dünya Düzenleme (WorldEdit-tarzı)](#-dünya-düzenleme-worldedit-tarzı)
  - [CatBot — Akıllı NPC Yoldaş](#-catbot--akıllı-npc-yoldaş)
  - [FarmerRobot — Otonom Çiftçi](#-farmerrobot--otonom-çiftçi)
  - [wfbuild — Prosedürel Yapı Motoru](#-wfbuild--prosedürel-yapı-motoru)
  - [6 Katmanlı Fırın Serisi](#-6-katmanlı-fırın-serisi)
  - [XP Collector](#-xp-collector)
  - [Sword Matrix](#-sword-matrix)
  - [Omnitool & Madencilik Araçları](#-omnitool--madencilik-araçları)
  - [WFVault & MegaChest](#-wfvault--megachest)
  - [Magnet Link](#-magnet-link)
  - [Görsel → Map-Art Yerleştirme](#-görsel--map-art-yerleştirme)
  - [Litematica Şematik Desteği](#-litematica-şematik-desteği)
  - [Ekonomi Entegrasyonu](#-ekonomi-entegrasyonu)
  - [Config GUI & Çoklu Dil](#-config-gui--çoklu-dil)
- [Komut Listesi](#komut-listesi)
- [Mimari Genel Bakış](#mimari-genel-bakış)
- [Kurulum](#kurulum)
- [Bağımlılıklar](#bağımlılıklar)
- [Uyumluluk](#uyumluluk)
- [Geliştirme](#geliştirme)
- [Katkıda Bulunma](#katkıda-bulunma)
- [Lisans](#lisans)

---

## WorldForge Nedir?

**WorldForge**, hayatta kalma modunda WorldEdit'in gücünü sunan tek bir modda toplanmış, birbirinden çoğunlukla bağımsız çalışabilen **~15 alt sistemden** oluşan büyük bir araç setidir. Toplu bölge düzenlemeden akıllı NPC yoldaşlara, prosedürel bina üretiminden çok katmanlı fırın serilerine kadar geniş bir yelpazeyi kapsar. Tüm ağır işlemler **tick tabanlı bir zamanlayıcı** üzerinden kademeli olarak yürütülür; böylece binlerce bloklu operasyonlar bile sunucuyu dondurmaz.

- **Mod ID:** `worldforge`
- **Ana sınıf:** `com.worldforge.WorldForgeMod`
- **~290 Java kaynak dosyası**, 15+ bağımsız alt sistem
- **500+ JSON tanımlı** stil/dekorasyon/yapı varyantı (`wfbuild`)
- **5 dil desteği:** İngilizce, Türkçe, Almanca, Fransızca, İspanyolca

---

## Öne Çıkan Özellikler

### 🧱 Dünya Düzenleme (WorldEdit-tarzı)

Survival'a uygun, ekonomi destekli toplu blok düzenleme araç seti:

- **Bölge seçimi & kopyala-yapıştır** — axe ile seçim, `//copy`/`//paste` benzeri iş akışı (`selection`, `clipboard`)
- **Doldurma / kırma / değiştirme** — `wffill`, `wfbreak`, `wfreplace`, `wffillair`, `wffillsolid`
- **Şekil araçları** — küre, silindir, piramit gibi geometrik formlar (`shape`)
- **Fırça & arazi düzenleme** — brush tabanlı terrain sculpting (`brush`, `terrain`)
- **Eğri/çizgi çizimi** — `wfline`, `wfcurve`
- **Genişletme/daraltma/kaydırma** — `wfexpand`, `wfcontract`, `wfshift`, `wfhollow`, `wfsmooth`, `wfcut`, `wfclear`
- **Sınırsız geri al / ileri al** — `wfundo` / `wfredo`, blok bazlı değişiklik kaydı ve ekonomi iadesi ile
- **Canlı hologram önizlemesi** — her operasyon gerçek bloklar yerleşmeden önce "hayalet blok" önizlemesi gösterir
- **Chunk güvenliği** — büyük işlemler öncesi otomatik anlık görüntü (`/wfundo` ile geri alınabilir güvenlik snapshotu)
- **WorkChest entegrasyonu** — malzemeler envanterden değil, bağlı kasalardan çekilir

### 🐱 CatBot — Akıllı NPC Yoldaş

GeckoLib ile animasyonlu, kedi ve insansı (humanoid) formları olan özelleştirilebilir bir arkadaş NPC:

- **İki form:** Cat (kedi) ve Humanoid, sorunsuz form geçişi
- **Özelleştirme:** göz rengi (`IrisColor`), tüy/saç rengi (`CatHairColor`), cinsiyet (`CatGender`)
- **Ses sistemi:** durum bazlı miyavlama/idle sesleri (`CatBotSounds`) + opsiyonel TTS (`CatTtsService`)
- **LLM entegrasyonu:** `/wfcat ask` komutuyla CatBot'a doğal dilde soru sorulabilir (`CatAskService`)
- **LLM destekli inşa:** aynı LLM altyapısı `/wfbuild` için de kullanılıp serbest metinden özel yapı üretebilir (`CatBuildLlmService`)
- **Goal tabanlı davranış:** `SmartNavigator`, sahibine yaklaşma/dönme, iş alanında dolaşma
- **Otomatik yeniden doğma:** `CatRespawnScheduler` ile ölüm sonrası zamanlanmış respawn

### 🌾 FarmerRobot — Otonom Çiftçi

Fabric `PathfinderMob` + GeckoLib tabanlı, tarlaları otomatik tarayıp hasat eden bir robot *(aktif geliştirme alanı)*:

- **Akıllı tarla algılama** (`FarmerFarmDetector`) ve **ekin önceliklendirme** (`FarmerCropPriority`)
- **Goal yığını:** gece/fırtınada kasaya sığınma → dinlenme → hasat → kasaya dönüş/dolaşma → ürün bırakma → boşta bakınma
- **Durum makinesi** (`FarmerRobotStatus`): `WORKING`, `IDLE`, `WAITING_FOR_SEEDS`, `CHEST_MISSING`, `NO_CROPS`, `SHELTERING`, `CHEST_FULL`, `RESTING`
- **FarmerChest entegrasyonu** — hasat edilen ürünler doğrudan bağlı kasaya depolanır
- **Kendi GUI ekranı** ile durum takibi

### 🏗️ wfbuild — Prosedürel Yapı Motoru

Modun en büyük ve en karmaşık alt sistemi (~75+ dosya): tek bir komuttan veya doğal dil açıklamasından tam teşekküllü binalar üretir.

**13 aşamalı boru hattı:**

```
Intent → Blueprint → Architecture → Style → Theme → Material Resolver
       → Palette → Decoration Resolver → Decoration Variants
       → Resource Logic → Validation → Planner → Builder
```

- **İki niyet ayrıştırıcı:** kural tabanlı `KeywordIntentParser` ve LLM tabanlı `LlmIntentParser`
- **500+ stil/dekorasyon JSON dosyası** — ~400 malzeme teması (ör. `ashen_spire`, `azure_boathouse`) + ~35 kültürel/estetik dekorasyon teması (Japon, Gotik, Cyberpunk, Viktoryen, vb.)
- **Mimari üreticiler:** temel, iskelet, duvar, çatı, kolon, merdiven, balkon, kapı/pencere boşlukları, dekorasyon
- **Bağımsız peyzaj öğeleri:** çeşme, havuz, koi göleti, bambu bahçesi, heykel, çit, lamba direği, kamp ateşi alanı, çiçek yatağı, kuyu, bahçe, yol
- **Biome'a duyarlı stil değişimi** (`BiomeStyleModifier`)
- **Yerleştirme öncesi doğrulama** (`ConstructionValidationEngine`) ve canlı hologram önizlemesi
- `/wfbuild` (CatBuildCommand) üzerinden veya CatBot'a doğal dilde tarif ederek tetiklenir

### 🔥 6 Katmanlı Fırın Serisi

Vanilla `AbstractFurnaceBlock`'u genişleten, her metal seviyesi için tam bağımsız blok/entity/menü/ekran seti sunan özel fırın hattı:

| Katman | Varsayılan Dayanıklılık* |
|---|---|
| Copper | 1.300 |
| Iron | 2.100 |
| Gold | 4.500 |
| Diamond | 6.500 |
| Emerald | 9.500 |
| Netherite | 15.000 |

*Alet dayanıklılığı, aynı dosyadaki `OmnitoolMaterial` sınıfına aittir; fırınlar için çoklu giriş/yakıt hattı (vanilla'nın tek-slot modelinin ötesinde) sunulur, admin `/wfdurability` ile çalışma zamanında değerleri ayarlayabilir.

### ⭐ XP Collector

Katmanlı (Copper → Netherite) deneyim puanı toplama/depolama blokları — yakındaki XP orb'larını otomatik emer ve depolar, oyuncu ihtiyaç duyduğunda geri verir.

### ⚔️ Sword Matrix

Birden fazla kılıcı tek bir güçlü silahta birleştirmeye yarayan özel bir menü/hesaplama sistemi (`SwordMatrixCalculator`, `SwordMatrixInventory`) — kendi GUI'si, tamir kancaları (`SwordMatrixRepairHooks`) ve tooltip önizlemesi ile.

### ⛏️ Omnitool & Madencilik Araçları

- **Omnitool** — 8 katmanlı (Wood → Netherite) çok işlevli alet, tek eşyada birden fazla alet türünün işlevini birleştirir
- **VeinMiner / VeinAxe / VeinTool** — bağlı blokları/ağaçları toplu kırma, canlı sınır (outline) önizlemesiyle
- **Excavator** — alan bazlı kazı aracı
- **Grook** — sneak + kırma ile yaprak temizleme
- **Dayanıklılık sistemi** — `DurabilityConfig` ile admin tarafından `/wfdurability` üzerinden runtime'da ayarlanabilir

### 📦 WFVault & MegaChest

- **WFVault** — sayfalanmış, kategorilere ayrılmış (`ItemCategory`), sunucu tarafı gerçek zamanlı aramalı dev depolama kasası
- **MegaChest** — L/T/+ şekilli inşa edilebilir çoklu blok sandık, paylaşımlı kapak animasyonu ve dinamik UV doku sistemi, 216 slot; büyük hacimli drop'lar için ideal
- **WorkChestManager** — dünya düzenleme operasyonlarının malzeme kaynağı olarak kasaları kullanmasını sağlar

### 🧲 Magnet Link

İki nokta arasında koordinat tabanlı (canlı referans gerektirmeyen), görsel bir kurye entity (`MagnetCarrierEntity`) ile temsil edilen uzak envanter/kaynak transfer sistemi.

### 🖼️ Görsel → Map-Art Yerleştirme

İnternetten bir görsel URL'si indirip (ana thread'i bloklamadan async) haritalara veya item-frame'lere piksel piksel basma (`MapArtPlacer`), istemci tarafı önizleme ekranı ile.

### 📐 Litematica Şematik Desteği

Litematica `.litematic` dosyalarını okuma, indirme ve modun kendi clipboard sistemine köprüleme (`LitematicaReader`).

### 💰 Ekonomi Entegrasyonu

Vault-uyumlu üçüncü parti ekonomi modlarına reflection tabanlı soft-dependency ile bağlanan `VaultEconomyBridge`; dünya düzenleme operasyonları blok başına ücretlendirilebilir, geri alma işlemlerinde iade edilir.

### ⚙️ Config GUI & Çoklu Dil

- **`FeatureConfig`** — admin'in her alt sistemi ayrı ayrı açıp kapatabildiği merkezi switchboard
- **Canlı config senkronizasyonu** — Request/Set/Sync network payload deseni ile istemci-sunucu arası anlık güncelleme
- **5 dil:** `en_us`, `tr_tr`, `de_de`, `fr_fr`, `es_es` — `/wflang` ile değiştirilebilir
- **Tıklanabilir öğretici kitap** (`WFTutorialBook`) — `/wfbook`

---

## Komut Listesi

<details>
<summary>Tüm <code>/wf*</code> komutlarını görmek için tıklayın</summary>

| Komut | Açıklama |
|---|---|
| `/wfselection` | Bölge seçimi araçları |
| `/wfcopy` / `/wfpaste` | Kopyala / yapıştır |
| `/wfundo` / `/wfredo` | Geri al / ileri al |
| `/wffill` / `/wffillair` / `/wffillsolid` | Doldurma varyantları |
| `/wfbreak` | Toplu blok kırma |
| `/wfreplace` | Blok değiştirme |
| `/wfshape` | Geometrik şekil çizimi |
| `/wfline` / `/wfcurve` | Çizgi / eğri çizimi |
| `/wfexpand` / `/wfcontract` / `/wfshift` | Seçim boyutlandırma/taşıma |
| `/wfhollow` / `/wfsmooth` / `/wfcut` / `/wfclear` | Ek düzenleme işlemleri |
| `/wfbuild` | Prosedürel yapı üretimi |
| `/wfblueprint` | Blueprint yönetimi |
| `/wfcat` | CatBot etkileşimi (`ask` dahil) |
| `/wfschematic` | Litematica şematik içe/dışa aktarma |
| `/wfimgselection` | Görsel → map-art seçim aracı |
| `/wfchest` | Kasa yönetimi |
| `/wffluid` | Akışkan (su/lava) araçları |
| `/wfdurability` | Alet/fırın dayanıklılık ayarları (admin) |
| `/wfeconomy` | Ekonomi ayarları (admin) |
| `/wfconfig` | Config GUI açma (admin) |
| `/wflang` | Dil değiştirme |
| `/wfbook` | Öğretici kitabı açma |
| `/wfapi` | LLM/API bağlantı ayarları |
| `/wfcancel` / `/wfresume` | Devam eden işlemi durdurma/sürdürme |

</details>

---

## Mimari Genel Bakış

```
com.worldforge
├── WorldForgeMod.java          # Ana giriş noktası
├── farmer/                     # Farmer Robot
├── catbot/                     # CatBot NPC + LLM entegrasyonu
├── wfbuild/                    # Prosedürel bina üretim motoru (13 aşamalı pipeline)
├── furnace/                    # 6 katmanlı özel fırın sistemi
├── xpcollector/                # Katmanlı XP toplama/depolama blokları
├── swordmatrix/                # Kılıç birleştirme sistemi
├── vault/, chest/               # WFVault, MegaChest, WorkChestManager
├── resourcelogic/, builder/op/  # JSON tabanlı blok kuralları + düşük seviye operasyonlar
├── tool/                        # Omnitool, VeinMiner, Grook, dayanıklılık
├── magnet/                      # Magnet Link uzak transfer
├── image/                       # Görsel → map-art yerleştirme
├── schematic/                   # Litematica desteği
├── selection/, clipboard/, undo/ # Seçim, kopyala-yapıştır, geri alma
├── veinminer/, grook/, destruction/ # Alan kırma + alet-blok eşleştirme
├── config/, economy/, permission/   # Config, ekonomi köprüsü, izinler
├── hologram/, particle/, chunk/     # Önizleme, sınır efektleri, chunk güvenliği
├── placement/                   # TickedTaskScheduler — merkezi iş kuyruğu
├── i18n/                        # 5 dilli çeviri erişimi
└── command/                     # Komut kayıtları
```

Alt sistemlerin her biri hakkında derinlemesine mimari dokümantasyon için [`docs/architecture/`](docs/architecture/README.md) klasörüne bakın — her dosya, ilgili alt sistemin sınıf sorumluluklarını, tasarım kararlarını ve veri akışını detaylandırır.

Modun kalbinde **`TickedTaskScheduler`** bulunur: tüm büyük ölçekli operasyonlar (dünya düzenleme, wfbuild inşaatları) buraya kademeli görevler olarak kuyruğa alınır, chunk güvenliği kontrol edilir ve hologram önizlemesiyle senkronize çalışır.

---

## Kurulum

1. [Fabric Loader](https://fabricmc.net/use/) ve [Fabric API](https://modrinth.com/mod/fabric-api) kurulu olmalı.
2. [GeckoLib 5](https://modrinth.com/mod/geckolib) (≥ 5.5.3) kurulu olmalı.
3. WorldForge `.jar` dosyasını `mods/` klasörüne kopyalayın.
4. (Opsiyonel) JEI / REI / EMI ve Jade destekleniyor — kurulu ise otomatik entegre olur.

## Bağımlılıklar

| Bağımlılık | Sürüm |
|---|---|
| Fabric Loader | ≥ 0.19.3 |
| Minecraft | ~26.2 |
| Java | ≥ 21 |
| Fabric API | * |
| GeckoLib | ≥ 5.5.3 |

**Önerilen (opsiyonel) modlar:** JEI, REI, EMI, Jade

## Uyumluluk

WorldForge; Sodium, Lithium, Iris, Continuity, Inventory Profiles Next, Xaero's Minimap, Backpacks!, Tree Vein Miner, Nature's Compass, Quick Leaf Decay, Mouse Tweaks ve Polymer tabanlı modlarla test edilmiş sunucularda sorunsuz çalışır.

## Geliştirme

Proje bir Gradle/Fabric Loom projesidir:

```bash
./gradlew build
```

Kaynak ağacı ~290 Java dosyası ve `src/main/resources` altında JSON tabanlı veri/varlık dosyalarından oluşur (özellikle `wfbuild` altında 500+ stil/dekorasyon JSON'u). Mimari kararların gerekçeleri için [`docs/architecture/design-decisions.md`](docs/architecture/design-decisions.md) dosyasına bakın.

## Katkıda Bulunma

Hata bildirimleri ve öneriler için [Issues](https://github.com/AlperCetin2001/WorldForge/issues) sayfasını kullanabilirsiniz.

## Lisans

Bu proje [CC0-1.0](LICENSE) lisansı ile yayınlanmıştır.

---

<div align="center">

Geliştirici: **Doshu** — [doshu.gamer.gd](https://doshu.gamer.gd)

</div>
