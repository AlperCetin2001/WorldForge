# Çekirdek Mimari — Ekonomi Köprüsü

Paket: `com.worldforge.economy` (2 dosya)

## 1. Amaç

WorldForge'un ücretli işlemlerini (wfbuild/wffill gibi kaynak tüketen komutlar) üçüncü parti bir ekonomi modu ile bağlamak; hiçbiri yoksa özelliğin sessizce devre dışı kalması.

## 2. Runtime Tespiti + Pluggable Provider

```java
final class VaultEconomyBridge {
    interface EconomyProvider {
        double  getBalance(ServerPlayer player);
        boolean withdraw(ServerPlayer player, double amount);
        String  getName();
        default boolean deposit(ServerPlayer player, double amount) { return false; }  // varsayılan no-op
    }
    // yansıma (reflection) ile üçüncü parti "Vault" ekonomi API'sini çalışma zamanında algılar
}
```

**Geriye dönük uyumluluk tasarımı:** `deposit()` metodu **sonradan eklendi** ve varsayılan olarak `false` döner (no-op). Bu sayede `deposit` eklenmeden önce yazılmış özel `EconomyProvider` implementasyonları **kod değişikliği olmadan derlenmeye devam eder** — sadece iade özelliğinden faydalanamazlar. Bu, Java `default` metodların arayüz genişletmede API kırmadan yeni yetenek eklemek için nasıl kullanılacağının iyi bir örneği.

`java.lang.reflect.Method` kullanımı, ekonomi eklentisinin derleme zamanı bağımlılığı olmadan (soft-dependency) çalışma zamanında algılanabilmesini sağlıyor — Vault API'si class path'te yoksa mod hâlâ derlenir/çalışır, sadece ekonomi özellikleri pasif kalır.

## 3. Config

`EconomyConfig.load(configDir)` — `WorldForgeMod.onInitialize()`'da diğer config'lerden ayrı, kendi dosyasında yüklenir (bkz. config-and-networking.md §1 — CatBotConfig ile aynı "özel modüller kendi dosyasını tutar" istisnası).

## 4. Tüketiciler

- `UndoManager` — `/wfundo` ile ücret iadesi (`deposit()`).
- Muhtemelen `WorkChestManager`/`CatBuildCommand` — iş başlatılırken `withdraw()` ile ücret tahsili (doğrudan kaynak taramasında görünmedi, ancak `EconomyConfig`'in varlığı ve `UndoManager`'ın iade mantığı bunu ima ediyor).

## 5. Genişletme Notları

- Yeni bir ücretli özellik eklerken **her zaman** karşılık gelen bir iade yolu düşün — `deposit()` default no-op olduğu için eski provider'larda sessizce çalışmayabilir, bu normal ve beklenen bir davranış.
- `EconomyProvider` arayüzüne yeni bir metot eklerken **her zaman `default` bir gövde ver** — mevcut implementasyonları kırma.
