# Çekirdek Mimari — Config Sistemi & Ağ Protokolleri

Paket: `com.worldforge.config` (2) + `config/client` (3) + `config/network` (3)

## 1. FeatureConfig — Merkezi Aç/Kapa Anahtar Panosu

`FeatureConfig`, **kendi ayrı config dosyası olmayan her modül** için ana switchboard: Farmer, Magnet, Furnace, VeinMiner, Grook, Image placement, Vault, MegaChest, Chunk safety, seçim parçacık sınırı. (Economy, Durability, CatBot, CatAsk kendi ayrı config dosyalarını tutar — aynı load/save/reset desenini izleyerek.)

- **Kalıcılık:** Tek bir JSON dosyası, `config/worldforge/worldforge_features.json`.
- **"Enabled" bayrakları hakkında kritik tasarım kararı:** Bloklar/itemler/entity'ler **her zaman kayıtlı kalır** — çalışma zamanında kaydı kaldırmak, onları zaten içeren dünyaları bozardı. Bir modülü devre dışı bırakmak, davranışını ilgili etkileşim/event/goal giriş noktasında **kapatır** (kayıt değil, davranış gate'lenir).
- **Tek istisna:** `chunkSafetyEnabled` — başlangıçta bir kez okunur çünkü sadece bir event kaydını gate'ler; etkili olması için **restart gerekir**.
- Operatörler `/wfconfig` komutuyla canlı ayarlar (`WorldForgeCommands`).

`FeatureConfigSchema` — şema/varsayılan değer tanımları (muhtemelen `FeatureConfig`'in JSON okurken/yazarken referans aldığı yapı).

## 2. Config GUI (`config/client`)

```java
WFConfigScreen extends Screen          // admin ayarlar ekranı
WFConfigClientState                    // istemci tarafı anlık config görüntüsü (son senkronize JSON)
WFConfigKeyInit implements ClientModInitializer   // tuş bağlama (GUI'yi açan kısayol)
```

## 3. Üç-Payload Senkronizasyon Deseni (`config/network`)

Bu, projede **birden fazla yerde tekrarlanan** bir ağ protokolü kalıbı (bkz. Vault arama, Image placement) — config için tam somut hali:

```java
record WFConfigRequestPayload() implements CustomPacketPayload            // C2S, boş — "güncel durumu gönder"
record WFConfigSetPayload(String module, String key, String value)        // C2S — tek bir değeri değiştir
record WFConfigSyncPayload(String json)                                   // S2C — güncel tam config JSON'ı
```

**Akış (`WorldForgeMod.registerConfigNetworking()`):**
1. GUI açılınca istemci `WFConfigRequestPayload` gönderir → sunucu `sendConfigSync(player)` ile o oyuncuya `WFConfigSyncPayload` yollar.
2. Bir kontrole tıklanınca istemci `WFConfigSetPayload(module, key, value)` gönderir:
   - `module == "_reset"` → `FeatureConfig.resetToDefaultsAndSave()`
   - `module == "_reload"` → diskten yeniden oku (`reloadFeatureConfig()`)
   - aksi halde → `FeatureConfig.set(module, key, value)`
   - **Admin izni kontrolü** (`PermissionChecker.requireAdmin`) her `Set` işleminde yapılır; yetkisizse istemciye hata mesajı + gerçek sunucu değerine geri senkron (`sendConfigSync`) gönderilir — böylece GUI asla sunucudan sapan bir değer göstermez.
3. Her başarılı `Set` sonrası `broadcastConfigSync(server)` çağrılır — **GUI'yi açık tutan TÜM oyunculara** (sadece isteği gönderene değil) yeni JSON yayınlanır, böylece birden fazla admin aynı anda GUI açıksa değerler senkron kalır.

## 4. Genel Desen — Yeni Bir Ağ Protokolü Eklerken

Projede gözlemlenen üçlü payload şablonu:
```
Request (C2S, genelde boş)  → "bana güncel durumu ver"
Set/Action (C2S, veri taşır) → "şunu değiştir/yap"
Sync (S2C, tam veya kısmi durum) → "işte güncel durum" (tek alıcıya veya broadcast)
```
Her payload `CustomPacketPayload` implement eden bir `record`; `PayloadTypeRegistry.serverboundPlay()`/`playS2C()` ile kayıt ve `ServerPlayNetworking.registerGlobalReceiver`/`.send()` çağrıları hep `WorldForgeMod`'daki adanmış bir `register*Networking()` metodunda toplanıyor — yeni bir sistem eklerken bu konvansiyonu koru.

## 5. Genişletme Notları

- Yeni bir modülün kendi ayrı config dosyası mı yoksa `FeatureConfig`'e mi ekleneceğine karar verirken: LLM/API anahtarı gibi hassas veya çok özel şema gerektiren modüller ayrı dosya (CatAskConfig deseni); basit aç/kapa + birkaç sayısal ayar `FeatureConfig`'e eklenmeli.
- Yeni bir `FeatureConfig` bayrağı eklerken **kaydı asla koşullu yapma** — sadece davranışı gate'le (bkz. §1 tasarım kararı), aksi halde var olan dünyalar bozulabilir.
- Yeni bir C2S/S2C senkron ihtiyacı olursa doğrudan mevcut Request/Set/Sync üçlüsünü kopyala.
