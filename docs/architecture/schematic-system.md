# Çekirdek Mimari — Şematik Sistemi

Paket: `com.worldforge.schematic` (4 dosya)

## 1. Amaç

Harici (Litematica formatlı) veya WorldForge'a özgü şematikleri okuma, indirme, diskte saklama; hepsi mevcut `ClipboardData` formatına (dolayısıyla `/wfcopy`/`/wfpaste`/`/wfschematic` komutlarının aynı malzeme-maliyetli iş kuyruğuna) akar.

## 2. Bileşenler

### `LitematicaReader` — Üçüncü Parti Format Desteği

```java
final class LitematicaReader { ... }  // .litematic dosyalarını ClipboardData'ya çevirir
```

- **Amaç:** İnternetten indirilmiş herhangi bir Litematica şematiği (litematica-sharing siteleri, print-farm repo'ları, bir arkadaşın linki), oyuncu gerçek Litematica mod'una ihtiyaç duymadan `/wfschematic import` ile içeri alınabilir ve normal malzeme-maliyetli iş kuyruğundan geçirilebilir.
- **Format:** `fi.dy.masa.litematica.schematic.LitematicaSchematic` (GNU LGPL) — bağımlılık olarak o mod'un kodu değil, **belgeli, stabil bir NBT düzeni** kullanılıyor.
- **Kapsam sınırı:** Sadece blok verisi içe aktarılır; entity'ler/block-entity NBT'si (sandık içerikleri, tabelalar vb.) **görmezden gelinir** — `ClipboardData`'nın `/wfcopy` ve `/wfschematic save` için zaten desteklediği kapsamla aynı.
- **MC 26.2 uyumluluk notu:** `NbtIo.readCompressed` artık bir `NbtAccounter` boyut sınırı gerektiriyor; `unlimitedHeap()` kullanılıyor çünkü `MAX_BLOCKS` sabiti zaten devasa/bozuk bir dosyanın sunucuyu OOM'a düşürmesini önlüyor (iki katmanlı güvenlik: NBT boyutu sınırsız kabul edilir ama blok sayısı sınırlanır).

### `SchematicDownloader` — Sınırlı, Güvenli İndirme

```java
final class SchematicDownloader { ... }  // .litematic dosyasını doğrudan linkten indirir
```
- Sadece `http(s)` kabul edilir.
- Yanıt boyutu **sınırlanmış**.
- Sadece `.litematic` (veya belirtilmemiş) içerik kabul edilir.
- Bu **oyuncu tetiklemeli, sunucu tarafı bir indirme** olduğu için bilinçli olarak sınırlı ve basit tutuluyor — `ServerImageDownloader` (image placement) ile aynı güvenlik felsefesi.

### `SchematicStore` — Diskte Kalıcı İsimli Şematikler

```java
final class SchematicStore { ... }  // /wfschematic save|load|list
```
Göreli-ofset JSON formatı, `hologram/BlueprintStore` ile **aynı veri şekli** ama kalıcı (disk tabanlı). `Gson` ile pretty-print JSON.

### `BlockStateSerializer`

```java
static String serialize(BlockState state)  // "minecraft:stairs[facing=north,half=bottom]" gibi kompakt string
```
`BlockState`'i insan-okunabilir/JSON-dostu bir string formatına çevirir/geri çözer — tüm şematik/blueprint JSON serileştirmesinin temel yapı taşı.

## 3. Veri Akışı

```
.litematic dosyası (URL veya yerel)
   │
   ▼ SchematicDownloader (URL ise)
Yerel dosya (server imports klasörü)
   │
   ▼ LitematicaReader
ClipboardData  ──────► /wfcopy, /wfpaste, /wfschematic ile aynı boru hattı
   │
   ▼ SchematicStore.save() (isteğe bağlı)
Kalıcı JSON (BlockStateSerializer ile kodlanmış blok durumları)
```

## 4. Genişletme Notları

- Yeni bir üçüncü parti şematik formatı desteklemek istersen `LitematicaReader` desenini izle: **format-özel okuyucu → `ClipboardData`'ya çevir**, böylece geri kalan boru hattı (kopyala/yapıştır/iş kuyruğu) hiç değişmeden çalışır.
- Herhangi bir dış kaynaktan (URL) veri indiren yeni bir özellik eklerken `SchematicDownloader`'ın üç kuralını kopyala: sadece http(s), boyut sınırı, içerik/uzantı doğrulaması.
- Büyük/bozuk dosyalara karşı her zaman **iki katmanlı sınır** düşün (NBT boyutu + mantıksal blok sayısı) — `LitematicaReader`'daki `unlimitedHeap()` + `MAX_BLOCKS` deseni.
