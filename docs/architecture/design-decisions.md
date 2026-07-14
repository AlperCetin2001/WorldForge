# Tasarım Kararları

Bu doküman, WorldForge genelinde tekrar eden veya özellikle bilinçli alınmış mimari kararların **neden**lerini bir arada toplar. Her bölüm "ne yapıldığı" değil "neden böyle yapıldığı" sorusuna odaklanır; "ne" için ilgili özellik dosyasına bakın.

## 1. Neden Theme, Style'dan türetiliyor (yeniden icat edilmiyor)?

`wfbuild` zincirinde **Theme** (haunted/volcanic/gothic), **Style**'ın (~400 JSON) üstüne binen ayrı, isteğe bağlı bir katman olarak kuruldu — kendi başına bağımsız bir malzeme sistemi olarak değil.

- **Kombinatorik patlamayı önler:** Eğer her (stil × tema) kombinasyonu için ayrı bir JSON dosyası gerekseydi, ~400 stil × birkaç tema = binlerce dosya olurdu. Bunun yerine tema sadece birkaç alanı (`wall`, `roof`, `accent`) **override eden bir overlay** tanımlar; herhangi bir stille serbestçe eşleşir.
- **Tek sorumluluk korunur:** Style "bu yapı türü + bu isim için temel malzeme nedir" sorusuna cevap verir; Theme "bu atmosferi nasıl bindiririm" sorusuna. İkisi karıştırılırsa yeni bir stil eklemek tema mantığını da anlamayı gerektirirdi.
- **Sonuç:** Yeni bir tema eklemek Style katmanına dokunmaz; yeni bir stil eklemek Theme katmanına dokunmaz. İki eksen birbirinden bağımsız büyüyebilir.

## 2. Neden LLM opsiyonel?

Hem `IntentEngine` (wfbuild) hem `CatAskService`/`CatBuildLlmService` (catbot) aynı kuralı izler: **LLM her zaman ikincil, çevrimdışı bir yol her zaman birincil veya yedek.**

- **Sıfır-yapılandırma çalışma:** Bir sunucu operatörü hiçbir API anahtarı girmeden modu tam olarak kullanabilir — `KeywordIntentParser` (sözlük+preset eşleştirme) ve `BuildRequestParser`/`StructureTemplates` gibi çevrimdışı yollar her zaman mevcuttur.
- **Güvenilirlik:** Ağ çağrıları başarısız olabilir (timeout, kötü anahtar, sağlayıcı kesintisi). Temel özellik bir dış servise bağımlı olsaydı, o servis çöktüğünde `/wfbuild` veya `/wfcat ask` tamamen kırılırdı. Bunun yerine başarısızlık sessizce çevrimdışı yola düşer.
- **Maliyet kontrolü:** LLM çağrıları operatöre para/hız-limiti maliyeti getirir; zorunlu olmayan bir yol olarak tasarlanması, kullanmak istemeyen sunucuların hiçbir bedel ödememesini sağlar.
- **Asla exception fırlatmama kuralı:** Bu felsefenin doğal bir sonucu olarak `CatAskService`/`CatBuildLlmService` LLM çağrısının her başarısızlık türünü (ağ, format, doğrulama) bir sonuç nesnesiyle (`success=false`) bildirir, hiçbir zaman exception fırlatıp çağıranı kilitlemez.

## 3. Neden JSON kalıtımı (`extends`) kullanılıyor?

`resourcelogic`'teki kural tanımları (`RawRuleDefinition` → `ResolvedRuleDefinition`) bir `"extends": "parentId"` alanıyla başka bir kuraldan türeyebiliyor.

- **Tekrarı azaltır:** Birbirine çok benzeyen onlarca kural varyantı (ör. farklı ekin türleri için neredeyse aynı davranış) her seferinde tüm alanları yeniden yazmak yerine, sadece farkı belirtir.
- **Veri-odaklı genişletilebilirlik:** Yeni bir varyant eklemek çoğu zaman **hiç kod değişikliği gerektirmez** — sadece yeni bir JSON dosyası + `extends` referansı.
- **Bilinçli sınırlamalar (güvenlik):** `RuleJsonLoader`, zinciri çözerken (a) bilinmeyen bir ebeveyne referans varsa ve (b) döngüsel bir `extends` zinciri varsa **açıkça hata fırlatır** (`IllegalStateException`). Veri-odaklı kalıtım sistemlerinde bu iki kontrol olmadan, kötü biçimlendirilmiş bir JSON dosyası sunucuyu sonsuz döngüde kilitleyebilir veya sessizce yanlış davranabilirdi — bu yüzden hata erken ve gürültülü olacak şekilde tasarlandı.

## 4. Neden Material Resolver overlay mantığında çalışıyor?

`MaterialResolver.applyTheme(style, base)`, bir temayı **toptan değiştirme değil, alan bazlı üstüne yazma (overlay)** olarak uyguluyor.

- **Temel stilin bütünlüğü korunur:** Bir tema sadece `wall`/`roof`/`accent` gibi birkaç alanı değiştirirse, temel stilin belirtmediği/varsaymadığı diğer tüm malzeme rolleri (kapı, pencere çerçevesi, zemin vb.) hâlâ o stilin kendi tanımından gelir. Toptan değiştirme olsaydı, her yeni tema **tüm** malzeme rollerini yeniden tanımlamak zorunda kalırdı — bu hem JSON dosyalarını şişirir hem de bir alan unutulduğunda sessiz/varsayılan (muhtemelen yanlış) bir bloğa düşme riski yaratırdı.
- **Kompozisyon kolaylığı:** Overlay deseni, "temel + N tane bağımsız katman" şeklinde düşünmeyi mümkün kılıyor (bkz. zincir: Style → Theme → Biome). Her katman öncekini bütünüyle anlamak zorunda değil, sadece kendi ilgilendiği alanları bilir.
- **Tema yoksa no-op:** Bir stil için tanımlı bir tema yoksa `applyTheme` temel paleti olduğu gibi döner — "tema" kavramının var olmaması, sistemin çökmesi veya eksik davranması anlamına gelmez.

## 5. Neden Resource Logic veri odaklı (JSON) tutuluyor?

`resourcelogic` alt sistemi, su/lava/genel blok davranış kurallarını Java kodu yerine JSON dosyalarıyla tanımlıyor (`RawRuleDefinition → RuleJsonLoader → ResolvedRuleDefinition → ResourceRuleRegistry`).

- **Operatör/moddan bağımsız ince ayar:** Bir kuralın davranışını değiştirmek (ör. hangi blokların "su ile etkileşir" sayıldığı) yeniden derleme gerektirmez — sadece JSON düzenlenip sunucu yeniden başlatılır (veya bir reload komutu varsa anında).
- **wfbuild ile aynı yükleme deseni:** `resourcelogic` bilerek `wfbuild/style` ve `wfbuild/architecture/decoration` ile **aynı** Raw→Loader→Registry iskeletini paylaşıyor (bkz. wfbuild.md §16) — bu, projede yeni bir JSON-tanımlı sistem eklerken izlenecek tek, tutarlı bir şablon olduğu anlamına geliyor; her alt sistem kendi yükleme mekanizmasını icat etmiyor.
- **`ConstructionValidationEngine` ile ayrık sorumluluk:** Resource Logic "bu blok/akışkan nasıl davranır" sorusuna cevap verirken, Validation "yeterli malzeme var mı" sorusuna cevap veriyor — ikisi ayrı katmanlar olduğu için biri değişse diğerinin mantığı etkilenmiyor.

## 6. Diğer Tekrarlayan Desenler (Kısa Not)

Bu üç kararın ötesinde, proje genelinde tutarlı şekilde tekrarlanan ve benzer gerekçelere dayanan başka kalıplar da var (ayrıntılar ilgili özellik dosyalarında):

- **Item + EventHandler ayrımı** (VeinMiner, Grook) — item tanımını ince tutup ağır alan mantığını ayrı sınıfa koymak, test edilebilirlik ve okunabilirlik için.
- **Async iş + `server.execute()` ile ana thread'e dönüş** (Image placement, CatBuildLlmService, CatAskService) — ağır I/O'nun sunucu tick'ini bloklamaması için.
- **`default` metotla arayüz genişletme** (`EconomyProvider.deposit()`) — geriye dönük uyumluluğu bozmadan yeni yetenek eklemek için.
- **Koordinat/UUID tabanlı bağlantılar, canlı referans yok** (Magnet Link) — chunk yükleme/boşaltma döngülerinde hayalet referans hatalarından kaçınmak için.

Bu kalıpların her biri, yeni bir alt sistem tasarlarken ilk başvurulacak "zaten çözülmüş problem" listesi olarak düşünülebilir.
