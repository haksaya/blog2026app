# Kişisel Blog — Web Programlama Ders Rehberi

**ASP.NET Core MVC + EF Core + Bootstrap 5**

Yazılarınızı ve kendinizi anlattığınız bir kişisel site, arkasında bir yönetim paneli.

---

## Bu rehber ne yapıyor?

Tek uygulamada iki yüz:

```
┌───────────────┬──────────────────────┐   ┌──────────────────────────────┐
│               │  24 OCAK 2026        │   │ ▓ Panel  Yazılar  Etiketler  │
│   ( foto )    │                      │   ├──────────────────────────────┤
│               │  Tuhaf Zamanlar      │   │ ┌────┐┌────┐┌────┐┌────┐     │
│  Dr. Harun    │                      │   │ │ 14 ││ 11 ││  3 ││4812│     │
│  AKSAYA       │  Zeki olmak bir      │   │ └────┘└────┘└────┘└────┘     │
│               │  zamanlar kapıları…  │   │                              │
│  HAKKIMDA     │                      │   │ Başlık      Durum   Tarih    │
│  ABONE OL     │  DEVAMINI OKU →      │   │ Tuhaf Z…   Yayında 24 Oca    │
│  ETİKETLER    │                      │   │ Bilgi D…   Taslak  12 Oca    │
└───────────────┴──────────────────────┘   └──────────────────────────────┘
         GENEL SİTE (herkese açık)              YÖNETİM PANELİ (sadece siz)
                     │                                     │
                     └──────────── aynı veritabanı ────────┘
```

İkisi de Bootstrap 5 kullanıyor. Fark, **hangi bileşenleri seçtiğimizde** ve genel sitede Bootstrap'in üzerine koyduğumuz ince katmanda (~140 satır).

---

## ⭐ Yeni kavramlar

| Konu | Önceki projelerde | Burada | Modül |
|---|---|---|---|
| **Bootstrap** | Hazır hâliyle | **CSS değişkenlerini ezme** | 1 |
| **Layout** | Tek layout | **İki ayrı layout** | 3 |
| **Adresler** | `/Gorev/Details/5` | **Slug** — `/yazi/merhaba-dunya` | 2, 6 |
| **Türkçe** | Konu olmadı | **`ToLowerInvariant` tuzağı** | 2, 9 |
| **İçerik** | Düz metin | **Markdown → HTML** | 5 |
| **Yetki yönü** | Her şey kapalı | **Tersi** — site açık, panel kapalı | 4 |
| **Anonim yazma** | Yoktu | **İletişim + abonelik** — bal küpü, XSS | 9 |
| **Dosya yükleme** | Yoktu | **Magic bytes, beyaz liste** | 10 |

---

## Modüller

| # | Modül | Süre | Ana kavram |
|---|---|---|---|
| — | [Proje planı](00-PROJE-PLANI.md) | — | Kurulum, veri modeli, takvim |
| 1 | [Bootstrap'i uyarlamak](01-bootstrap-ve-tasarim.md) | **40 dk** | CSS değişkenleri, `-rgb` tuzağı, kontrast |
| 2 | [Model, slug ve veritabanı](02-model-ve-veritabani.md) | 2 saat | Türkçe slug, enum, migration |
| 3 | [**İki layout**](03-iki-layout.md) | 1-2 saat | `_ViewStart`, yan sütun, `collapse` |
| 4 | [Yönetim girişi](04-yonetim-girisi.md) | 1 saat | Ters yetki, taban controller |
| 5 | [**Yazı yönetimi + Markdown**](05-yazi-yonetimi.md) | 2 saat | Markdig, `Html.Raw`, taslak/yayın |
| 6 | [**Genel site**](06-genel-site.md) | 2 saat | Slug rotası, okuma sayacı, sayfalama |
| 7 | [Dinamik sayfalar](07-sayfalar.md) | 1 saat | Yakalayıcı rota, View Component |
| 8 | [Etiketler](08-etiketler.md) | 1 saat | Skip navigation (bilinen konu) |
| 9 | [**İletişim + abonelik**](09-iletisim.md) | 1 saat | Bal küpü, XSS, yarış koşulu |
| 10 | [**Görsel yükleme**](10-gorsel-yukleme.md) | 1 saat | `IFormFile`, magic bytes |
| 11 | [Son dokunuşlar](11-son-dokunuslar.md) | 1 saat | Arama, RSS, sitemap, rastgele yazı |

Toplam **11-13 ders saati**.

---

## 🎨 Tasarım — kısa tutuluyor

Bu bir web programlama dersi, tasarım dersi değil. Modül 1 **40 dakika** ve üç şey anlatıyor:

1. Bootstrap'i bağlamak ve hazır bileşenlerini kullanmak
2. Altı CSS değişkeniyle siteyi kendinize ait hâle getirmek
3. "Niye kişiselleştiriyoruz?" sorusunun cevabı

Kalan tasarım işi Bootstrap'in kendi sınıflarıyla yapılıyor: `row`, `col-lg-3`, `card`, `btn`, `form-control`, `table`, `badge`, `alert`.

### Değişen altı değişken

```css
:root {
  --bs-body-font-family: 'Lato', system-ui, sans-serif;
  --bs-body-color: #4a4a4a;
  --bs-heading-color: #2b2b2b;
  --bs-link-color: #8a6800;
  --bs-link-color-rgb: 138, 104, 0;   /* ⚠️ bunsuz çalışmaz */
  --bs-border-color: #e3e7ec;
}
```

### ⚠️ Derste mutlaka gösterilecek tuzak

Bootstrap 5.3 bağlantı rengini `--bs-link-color`'dan **değil**, `--bs-link-color-rgb`'den okur:

```css
a { color: rgba(var(--bs-link-color-rgb), var(--bs-link-opacity, 1)); }
```

Sadece `--bs-link-color` yazan öğrenci bağlantıların mavi kaldığını görür ve nedenini bulamaz. Modül 1'de bu adım adım anlatılıyor.

### Ölçülmüş kontrast

Paletin her metin rengi WCAG AA eşiğini (**4.5:1**) geçiyor:

| Rol | Renk | Zemin | Oran |
|---|---|---|---|
| Başlık | `#2b2b2b` | beyaz | **14.16:1** ✅ |
| Gövde | `#4a4a4a` | beyaz | **8.86:1** ✅ |
| Soluk / tarih | `#6b6b6b` | `#f6f8fa` | **5.01:1** ✅ |
| Bağlantı | `#8a6800` | beyaz | **5.17:1** ✅ |
| Buton yazısı | `#212529` | `#ffd43b` | **10.82:1** ✅ |

**Sarı metin problemi:** `#ffd43b` beyaz üzerinde **1.43:1** — eşiğin üçte biri. Çözüm sarıdan vazgeçmek değil, **rolünü ayırmak**: sarı zemin ve çerçeve olarak kalıyor, metin olarak koyultulmuş `#8a6800` kullanılıyor. Göz aynı sarıyı görüyor, okunabilirlik kazanılıyor.

> Modül 1 bunun nasıl ölçüldüğünü de anlatıyor (Chrome DevTools, Lighthouse, WebAIM).

---

## Veri modeli

```
Kullanici          (tek admin)

Yazi               ──┬──< YaziEtiketleri >──┬── Etiket
  Baslik, Slug ⭐    │                      │
  Ozet, Icerik (md)  │
  KapakGorsel        │
  Durum (Taslak/Yayinda)
  YayinTarihi, OkunmaSayisi

Sayfa              (Hakkımda, Projelerim — panelden yönetilir)
Mesaj              (iletişim formundan — anonim yazma)
Abone              (yan sütundaki abone formundan — anonim yazma)
```

---

## Başlarken

**Gerekenler**
- .NET SDK 10.0 *(veya 8.0 — tüm kodlar ikisinde de çalışır)*
- SQL Server Express + SSMS
- Visual Studio 2022 veya VS Code + C# Dev Kit

**Kurulum**

```bash
dotnet new mvc -n KisiselBlog
cd KisiselBlog

dotnet add package Microsoft.EntityFrameworkCore.SqlServer
dotnet add package Microsoft.EntityFrameworkCore.Design
dotnet add package Markdig                    # ⭐ yeni
```

**Bootstrap kalıyor** — şablondan gelen `wwwroot/lib/bootstrap/` silinmiyor. Tek yapılacak: `wwwroot/css/site.css` içeriğini boşaltmak.

Ardından [Modül 1](01-bootstrap-ve-tasarim.md) ile başlayın.

**Giriş bilgileri:** `admin` / `blog2026`

---

## Modül şablonu

| Bölüm | İşlevi |
|---|---|
| 🎯 Bu derste ne yapacağız | Tahtaya yazılacak hedef |
| 📖 Kavram | Kod yazmadan önceki teori |
| ⌨️ Adım adım kod | **Eksiksiz**, satır satır açıklamalı |
| ▶️ Çalıştır ve gör | Ekranda görünmesi gereken |
| ⚠️ Sık yapılan hatalar | Derste karşılaşılacak sorunlar |
| ✏️ Alıştırmalar | Ders sonu ödevi |

Öğrenciye bırakılmış eksik kod **yoktur**. Tüm controller'lar, model'ler, view'lar ve CSS tam yazılmıştır.

---

## Güvenlik konuları

Her biri kendi bağlamında, kod yazılırken işleniyor:

| Konu | Modül |
|---|---|
| Varsayılan kapalı yetki (taban controller) | 4 |
| Kullanıcı sayımı (user enumeration) | 4, 9 |
| Açık yönlendirme (`Url.IsLocalUrl`) | 4, 9 |
| XSS — `Html.Raw` ne zaman kullanılır, ne zaman kullanılmaz | 5, 9 |
| Markdown'da ham HTML engelleme | 5 |
| Taslak sızıntısı (adres tahmini) | 6 |
| Yasaklı slug (sistem adreslerini koruma) | 7 |
| Bal küpü ve bot engelleme | 9 |
| CSRF ve anti-forgery token | 9 |
| Yarış koşulu ve veritabanı kısıtı | 2, 9 |
| Dosya yükleme: beyaz liste, magic bytes, path traversal | 10 |
| `robots.txt` ve `noindex` | 3, 11 |

---

## Ön koşul

Şunları bildiğiniz varsayılıyor ve tekrar anlatılmıyor:

- ASP.NET Core MVC temeli
- EF Core (DbContext, migration, LINQ, `Include`, query filter)
- Çerez tabanlı giriş sistemi
- CRUD kalıbı
- Bootstrap sınıflarının temeli (`row`, `col`, `btn`, `form-control`, `table`)

Eksiğiniz varsa önce **GörevTakip-EF** rehberini tamamlayın.

Yeni olan tek paket: **Markdig**.

---

## Lisans

Eğitim amaçlı serbestçe kullanılabilir, çoğaltılabilir ve uyarlanabilir.

> ⚠️ Şifre hash'leme örneği (SHA256) **öğretim amaçlıdır**, gerçek projeler için yeterli değildir. Gerçekte BCrypt, Argon2 veya ASP.NET Core Identity kullanın.
