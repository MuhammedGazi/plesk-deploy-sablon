# plesk-deploy-sablon

**ASP.NET Core uygulamanı paylaşımlı Windows Plesk hostinge `git push` ile otomatik deploy et.**
GitHub Actions + Plesk Git + IIS shadow copy. FTP yok, SSH yok, sunucuda SDK yok.

```
git push  →  GitHub Actions (dotnet publish)  →  release dalı  →  webhook  →  Plesk Git  →  canlı (~30 sn)
```

> Türk Ticaret Web Eko Hosting (Windows / Plesk / IIS 10 / .NET 10) üzerinde uçtan uca denendi.
> Başka Windows Plesk hostinglerde de büyük ölçüde aynı şekilde çalışır.

---

## Neler var?

| | |
|---|---|
| 📘 [`docs/REHBER.md`](docs/REHBER.md) | Adım adım kurulum, sorun giderme tablosu, güvenlik notları |
| ⚙️ [`sablonlar/deploy.yml`](sablonlar/deploy.yml) | GitHub Actions workflow'u |
| ⚙️ [`sablonlar/web.config`](sablonlar/web.config) | Shadow copy ayarlı `web.config` |
| ⚙️ [`sablonlar/csproj-ayarlari.xml`](sablonlar/csproj-ayarlari.xml) | `.csproj`'a eklenecek iki satır |

## Kimin için?

- Paylaşımlı **Windows Plesk** hostingde **.NET** uygulaması yayınlayanlar
- Her seferinde FTP ile dosya sürüklemekten sıkılanlar
- CI/CD'ye yeni başlayıp küçük, anlaşılır bir örnek arayanlar

## Hızlı başlangıç

1. **Depona dosyaları ekle:**
   - `sablonlar/deploy.yml` → `.github/workflows/deploy.yml` (`PROJECT_PATH`'i düzenle)
   - `sablonlar/web.config` → proje klasörüne (`MyApp.dll`'i kendi DLL adınla değiştir)
   - `sablonlar/csproj-ayarlari.xml` içindeki iki satırı `.csproj`'a ekle
2. **Push et.** Actions çalışır, `release` dalı oluşur.
3. **Hostingte siteyi hazırla:** belge kökü `<site>/app`, yanına `shadow` ve `logs` klasörleri.
4. **Plesk → Git:** depoyu SSH ile bağla, dal `release`, yol `\<site>\app`. Plesk'in verdiği anahtarı GitHub'a salt okunur **deploy key** olarak ekle.
5. **Webhook:** Plesk'in verdiği web kancası URL'sini GitHub'a ekle.
6. **SSL:** Plesk → Let's Encrypt, ücretsiz.

Ayrıntılar ve ekran ekran anlatım: **[docs/REHBER.md](docs/REHBER.md)**

## En çok takılınan üç yer

1. **DLL kilidi:** Uygulama çalışırken Plesk `MyApp.dll`'in üzerine yazamaz ve site eski sürümde kalır. Çözüm: `web.config`'teki **shadow copy** ayarı ve `shadow` klasörünün site klasörünün *içinde* olması.
2. **DNS (Türk Ticaret):** Subdomain'i Plesk'ten açarsan DNS kaydı oluşmuyor. Subdomain'i **müşteri paneli → Hosting İşlemleri → Subdomainler**'den aç.
3. **`www` kutusu:** Subdomain sertifikasında `www` işaretleme; ana alan adında işaretle.

## Katkı

Başka bir hostingte denediysen ya da bir adım farklı çalıştıysa issue açman ya da PR göndermen çok işe yarar.

## Lisans

[MIT](LICENSE)
