# Adım Adım Rehber

Bu rehber, ASP.NET Core uygulamasını **paylaşımlı Windows Plesk hostinge** GitHub üzerinden otomatik deploy etmeyi anlatır.
Türk Ticaret Web Eko Hosting paketinde uçtan uca denendi. Başka bir Windows Plesk hostingte de büyük ölçüde aynı şekilde çalışır. Firmaya özel adımlar ayrıca işaretlendi.

Örneklerde alan adı `alanadi.com`, subdomain `test.alanadi.com`, proje `MyApp` olarak geçiyor. Kendi değerlerinle değiştir.

---

## İçindekiler
1. [Nasıl çalışıyor?](#1-nasıl-çalışıyor)
2. [Ön koşullar ve hostingi tanımak](#2-ön-koşullar-ve-hostingi-tanımak)
3. [Sunucudaki klasör yapısı](#3-sunucudaki-klasör-yapısı)
4. [Depoyu hazırlamak](#4-depoyu-hazırlamak)
5. [Subdomain açmak](#5-subdomain-açmak)
6. [Plesk Git bağlantısı](#6-plesk-git-bağlantısı)
7. [Webhook: push'ta otomatik deploy](#7-webhook-pushta-otomatik-deploy)
8. [Ücretsiz SSL (Let's Encrypt)](#8-ücretsiz-ssl-lets-encrypt)
9. [Test](#9-test)
10. [Sorun giderme](#10-sorun-giderme)
11. [Güvenlik notları](#11-güvenlik-notları)
12. [Veritabanı](#12-veritabanı)

---

## 1. Nasıl çalışıyor?

```
git push (main)
   │
   ▼
GitHub Actions ── dotnet publish ──► derlenmiş çıktı "release" dalına
   │
   ▼ (webhook)
Plesk Git ── "release" dalını çeker ──► <site>\app   (IIS buradan sunar)
                                         <site>\shadow (uygulamanın çalışan kopyası)
                                         <site>\logs   (hata günlükleri)
```

- Sunucuda .NET SDK'ya ihtiyaç yok; derleme GitHub'da yapılır.
- Sunucuya FTP şifresi, SSH ya da Web Deploy gerekmez. Plesk, depoyu **salt okunur bir deploy key** ile okur.
- Geri almak için `git revert` + push yeterli.

---

## 2. Ön koşullar ve hostingi tanımak

**Gerekenler**
- Windows Plesk hosting (IIS)
- GitHub hesabı
- .NET projesi (bu rehberde .NET 10)

**Hostingin ASP.NET Core destekleyip desteklemediğini öğren.**
Plesk'teki "ASP.NET 4.8" ayarı eski .NET Framework içindir; ASP.NET Core desteği hakkında bir şey söylemez. En kesin yol küçük bir test uygulaması yüklemek:

| Sonuç | Anlamı |
|---|---|
| Uygulama açılıyor | Sunucuda o .NET sürümü var; "framework-dependent" yayın yeterli |
| HTTP **500.31 / 500.30** | ASP.NET Core modülü var ama o runtime yok; `--self-contained` yayınla |
| HTTP **500.19** | ASP.NET Core modülü hiç yok; hosting firmasıyla konuş |

Test ettiğimiz sunucuda .NET 10 kuruluydu ve uygulama havuzu **32 bit** çalışıyordu. Native DLL içeren paketlerde (bazı PDF/görüntü kütüphaneleri) x86 sürümüne dikkat et, ya da Plesk'te "32 bit uygulamaları etkinleştir" seçeneğini kapat.

---

## 3. Sunucudaki klasör yapısı

```
test.alanadi.com\
  app\       ← belge kökü. Plesk Git buraya dağıtır.
  shadow\    ← shadow copy klasörü
  logs\      ← stdout günlükleri
```

**Neden shadow copy?** Uygulama çalışırken DLL dosyaları kilitlidir. Plesk yeni sürümü kopyalarken `MyApp.dll`'in üzerine yazamaz ve site eski sürümde kalır. Bunu denedik: `.pdb` güncellendi, `.dll` güncellenmedi. Shadow copy açıkken uygulama `shadow\` içindeki kopyadan çalışır ve `app\` serbest kalır.

**`shadow` site klasörünün içinde olmalı.** Site klasörünün dışına (ör. `../ShadowCopyDirectory`) uygulamanın yazma izni yoktur; site gövdesiz bir **500** hatası verir.

---

## 4. Depoyu hazırlamak

Bu depodaki [`sablonlar/`](../sablonlar) klasöründen üç dosya kullanacaksın:

1. **`.github/workflows/deploy.yml`** ← [`sablonlar/deploy.yml`](../sablonlar/deploy.yml)
   `PROJECT_PATH` satırını kendi projenin klasörüyle değiştir.
2. **`<proje klasörü>/web.config`** ← [`sablonlar/web.config`](../sablonlar/web.config)
   `MyApp.dll` yerine kendi DLL adını yaz.
3. **`.csproj`** ← [`sablonlar/csproj-ayarlari.xml`](../sablonlar/csproj-ayarlari.xml) içindeki iki satırı ekle.

Neden bu iki ayar?
- `UseAppHost=false`: GitHub Actions Linux'ta derler. Apphost üretilirse `web.config` Windows'ta çalışmayan bir dosyayı gösterebilir.
- `IsTransformWebConfigDisabled=true`: SDK'nın `web.config`'i yeniden yazıp shadow copy ayarını silmesini engeller.

Push et. GitHub'da **Actions** sekmesinde iş yeşil yanınca `release` dalı oluşur.

---

## 5. Subdomain açmak

> ⚠️ **Türk Ticaret'e özel:** Subdomain'i **Plesk'ten açma.** Plesk'te açılan subdomain için DNS kaydı oluşmuyor ve adres internette çözülmüyor. Alan adı panelindeki "DNS Yönetimi / DNS Pro" ekranına eklenen kayıtlar da yayına çıkmadı.
>
> **Doğru yol:** Türk Ticaret müşteri paneli → **Hosting İşlemleri** → alan adı → **İşlemler** → **Subdomainler**

- **Subdomain:** `test`
- **Kök dizin:** `test.alanadi.com/app`

Panel hem Plesk'te siteyi hem de DNS kaydını oluşturur.

Sonra Plesk → **Dosya Yöneticisi** → `test.alanadi.com` klasöründe, `app`'in **yanında** iki klasör aç:
- `shadow`
- `logs`

`app` içindeki varsayılan `index.html` gibi dosyaları sil.

Kontrol: Plesk → site → **Barındırma Ayarları** → belge kökü `test.alanadi.com/app` olmalı.

> Ana alan adına (`alanadi.com`) kuruyorsan subdomain adımı yok. Belge kökünü `httpdocs\app` gibi bir klasöre al ve önce mevcut `httpdocs` içeriğini yedekle.

---

## 6. Plesk Git bağlantısı

Plesk → `test.alanadi.com` → **Git** → depo ekle:

| Alan | Değer |
|---|---|
| Depo URL'si | `git@github.com:<kullanıcı>/<depo>.git` |
| Sunucu yolu | `\test.alanadi.com\app` |
| Dağıtım modu | Otomatik |

Plesk sana bir **SSH genel anahtarı** gösterir. **Oluştur'a basmadan önce** bu anahtarı GitHub'a ekle:

GitHub → depo → **Settings** → **Deploy keys** → **Add deploy key** → anahtarı yapıştır. **"Allow write access" işaretsiz** kalsın.

Ya da terminalden:
```bash
gh repo deploy-key add plesk.pub -R <kullanıcı>/<depo> --title "plesk"
```

Sonra **Oluştur**. Ardından **Depo Ayarları**:
- **Depo şubesi:** `release` (listede yoksa önce "Güncellemeleri Çek")
- **Uygula** → **Şimdi Dağıt**

---

## 7. Webhook: push'ta otomatik deploy

Plesk → Git → **Depo Ayarları**'nda **Web kancası URL'si** var (`https://<plesk-sunucusu>:8443/modules/git/public/web-hook.php?uuid=...`).

GitHub → depo → **Settings** → **Webhooks** → **Add webhook**:
- **Payload URL:** Plesk'in verdiği adres
- **Content type:** `application/json`
- **Events:** Just the push event

Ya da terminalden:
```bash
gh api repos/<kullanıcı>/<depo>/hooks --method POST \
  -f name=web -F active=true -f "events[]=push" \
  -f "config[url]=<PLESK WEBHOOK URL>" -f "config[content_type]=json"
```

Bu URL'deki `uuid` bir anahtar gibidir; herkese açık yerlere yazma.

---

## 8. Ücretsiz SSL (Let's Encrypt)

Plesk → `test.alanadi.com` → **SSL/TLS Sertifikaları** → **Let's Encrypt**:
- ✅ Alan adını koru
- ⬜ Joker alan adı
- ⬜ `www` alt alan adı (subdomain'de `www.test...` diye bir adres yok; işaretlersen doğrulama hata verebilir)
- Doğrulama: **HTTP (http-01)**

**Ücretsiz olarak alın.** Sertifika 90 günde bir Plesk tarafından otomatik yenilenir.

Ana alan adında `www` işaretli olmalı; insanlar iki şekilde de yazar.

> Hosting panelinde "SSL Aktif Et" gibi bir seçenek ücretli sertifika satıyor olabilir. Let's Encrypt'i Plesk'ten al.

---

## 9. Test

1. Kodda görünür bir şeyi değiştir (ör. sayfaya `v2` yaz) ve push et. ~30 saniye sonra sitede görünmeli.
2. Uygulama çalışırken bir kez daha push et (`v3`). Bu kilit testidir; shadow copy çalışıyorsa bu da görünür.

Webhook teslimatlarını GitHub → Settings → Webhooks → **Recent Deliveries**'ten görebilirsin. Plesk başarılı olunca `204` döner.

---

## 10. Sorun giderme

| Belirti | Olası neden | Çözüm |
|---|---|---|
| **500**, sayfa boş | Shadow klasörüne yazılamıyor | `shadow` site klasörünün içinde mi? `logs\stdout_*.log` dosyasına bak |
| **403 Forbidden** | Belge kökünde uygulama yok | Git dalı `release` mi? Sunucu yolu = belge kökü mü? "Şimdi Dağıt" |
| Push oldu, site **eski sürümde** | DLL kilitli | `web.config`'te shadow copy ayarı var mı? |
| **Site bulunamadı** | DNS kaydı yok | Subdomain'i hosting panelinden aç (5. bölüm). Kontrol: `nslookup test.alanadi.com` |
| Plesk Git: `Permission denied (publickey)` | Deploy key GitHub'da yok | Plesk'teki anahtarla GitHub'dakini karşılaştır |
| Plesk Git: `destination path ... already exists` | Silinmiş bir siteden kalan depo klasörü | Depo adını değiştir ya da Dosya Yöneticisi → `git` klasöründen sil |
| Plesk: `DB query failed ... lock table size` | Plesk'in kendi geçici hatası | 1-2 dakika bekleyip tekrar dene |
| Tarayıcıda sertifika uyarısı | Sertifika yok ya da adı kapsamıyor | 8. bölüm |
| Plesk panelinde sertifika uyarısı | Plesk'e alan adınla `:8443` üzerinden giriyorsun | Plesk'e sunucunun kendi adıyla gir |

---

## 11. Güvenlik notları

- Plesk'e **8880** üzerinden girme; o port şifresiz HTTP. Her zaman `https://...:8443` kullan.
- Deploy key **salt okunur** kalsın.
- Webhook URL'sini, FTP/Plesk şifrelerini ve bağlantı dizelerini depoya koyma.
- Alan adı panelinde **DNS koruması** varsa aç.
- Paylaşımlı hostingte veritabanı portları (1433/3306) dışarı açık olabilir; güçlü şifre kullan.

---

## 12. Veritabanı

- Paketlerde genelde küçük bir MSSQL kotası olur (denediğimiz pakette 50 MB). Küçük projeler için yeter.
- Daha fazlası için ücretsiz seçenek: [Azure SQL Database free offer](https://learn.microsoft.com/azure/azure-sql/database/free-offer) (veritabanı başına 32 GB, aylık işlem kotası; kota bitince "duraklat" seçeneğiyle ücret çıkmaz).
- **Migration'ları otomatik deploy'a bağlama.** Veritabanı değişikliklerini ayrı ve kontrollü bir adımda uygula.
- Bağlantı dizesini depoya değil, sunucudaki ayarlara (Plesk ortam değişkenleri ya da sunucuda tutulan `appsettings.Production.json`) koy.
