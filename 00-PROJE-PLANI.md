# Kişisel Blog — Proje Planı

**ASP.NET Core MVC + EF Core + Bootstrap 5**
Dördüncü proje. Bir kişisel site ve blog, arkasında yönetim paneli.

---

## Ne yapacağız?

İki yüzü olan tek bir uygulama:

```
┌─────────────────────────────┐   ┌─────────────────────────────┐
│   GENEL SİTE                │   │   YÖNETİM PANELİ            │
│   herkese açık              │   │   sadece siz                │
│                             │   │                             │
│   /              ana sayfa  │   │   /yonetim      panel       │
│   /yazi/slug     yazı       │   │   /yonetim/yazi yazılar     │
│   /etiket/slug   etiket     │   │   /yonetim/sayfa sayfalar   │
│   /hakkimda      sayfa      │   │   /yonetim/mesaj mesajlar   │
│   /iletisim      form       │   │                             │
│                             │   │                             │
│   solda sabit yan sütun,    │   │   üstte navbar,             │
│   sağda yazı akışı          │   │   tablo ağırlıklı           │
└─────────────────────────────┘   └─────────────────────────────┘
            │                                  │
            └──────────── aynı veritabanı ─────┘
```

Bir kişinin kendi yazılarını yayınladığı, kendini anlattığı, mesaj alabildiği bir site. Arkasında da o içeriği yönettiği bir panel.

---

## Tasarımda yolumuz

Bootstrap 5 kullanıyoruz — üç projedir kullandığımızın aynısı. Yeni bir CSS dili öğrenmiyoruz.

Tek eklediğimiz şey şu: **Bootstrap'in hazır görünümünü beş-altı satırla kendimize uyarlıyoruz.** Yazı tipini, metin rengini, vurgu rengini değiştiriyoruz. Gerisi Bootstrap'in kendi sınıfları: `row`, `col-lg-3`, `btn`, `form-control`, `card`.

Bu 30-40 dakikalık bir konu (Modül 1), 2 saatlik değil. Ders ağırlığı C# tarafında: slug, Markdown, ters yetki, dosya yükleme.

### Referans

Ana sayfayı **harunaksaya.blog** düzenine göre kuruyoruz:

```
┌───────────────┬──────────────────────────────────────┐
│               │  24 OCAK 2026                        │
│   ( foto )    │                                      │
│               │  Tuhaf Zamanlar            ← büyük   │
│  Dr. Harun    │                                      │
│  AKSAYA       │  Zeki olmak bir zamanlar kapıları    │
│               │  açardı; bugünse o kapıları…         │
│  ○ ○ ○ ○ ○    │                                      │
│               │  DEVAMINI OKU →                      │
│  HAKKIMDA     │                                      │
│  ABONE OL     │  ──────────────────────────────      │
│  ARA          │                                      │
│  KEŞFET       │  12 OCAK 2026                        │
│  ETİKETLER    │                                      │
│               │  Bilgi Diyeti                        │
│   sabit       │  …                                   │
└───────────────┴──────────────────────────────────────┘
   col-lg-3                  col-lg-9
```

Bir `row`, iki `col`. Bootstrap'in ızgarası bunu zaten yapıyor.

---

## ⭐ Bu projede yeni olanlar

Üç proje boyunca CRUD, EF Core, login öğrendik. Bu rehber **onları tekrar anlatmaz.** Ağırlık yeni konularda:

| Konu | Önceki projelerde | Bu projede | Modül |
|---|---|---|---|
| **Bootstrap'i uyarlama** | Hazır hâliyle kullandık | ⭐ **CSS değişkenlerini ezmek** | 1 |
| **Layout** | Tek layout | ⭐ **İki ayrı layout** — genel site + yönetim | 3 |
| **Adresler** | `/Gorev/Details/5` | ⭐ **Slug** — `/yazi/merhaba-dunya` | 2, 6 |
| **Türkçe karakter** | Konu olmadı | ⭐ **`ToLowerInvariant` tuzağı**, harf çevirimi | 2 |
| **İçerik biçimi** | Düz metin | ⭐ **Markdown** → HTML | 5 |
| **Yetki yönü** | Her şey korumalı | ⭐ **Tersi** — genel site açık, yönetim kapalı | 4 |
| **Anonim yazma** | Yoktu | ⭐ **İletişim + abonelik** — giriş yapmamış kişi veritabanına yazıyor | 9 |
| **Dosya yükleme** | Yoktu | ⭐ **Kapak görseli** — `IFormFile`, doğrulama, güvenlik | 10 |
| **Yayın durumu** | — | Taslak / Yayında ayrımı | 2, 5 |
| CRUD, EF, login, N:N | Öğrenildi | Hızla geçilir | 5, 8 |

### En çok vakit ayrılacak konu

**Slug ve Türkçe (Modül 2).** `"Merhaba Dünya"` başlığından `merhaba-dunya` adresi üretmek göründüğünden zor. Türkçe karakterler, `I`/`ı` dönüşüm tuzağı, aynı slug'ın ikinci kez üretilmesi — hepsi gerçek problemler ve hiçbiri hazır paketle çözülmüyor.

---

## Veri modeli

```
Kullanici          (tek admin — bu kişisel bir site)

Yazi               ──┬──< YaziEtiketleri >──┬── Etiket
  Baslik             │                      │
  Slug  ⭐           │   (çoktan çoğa)      │
  Ozet               │                      │
  Icerik (markdown)  │
  KapakGorsel        │
  Durum (Taslak/Yayinda)
  YayinTarihi, OkunmaSayisi

Sayfa              (Hakkımda, Projelerim... — koddan değil veritabanından)
  Baslik, Slug, Icerik, MenudeGoster, Sira

Mesaj              (iletişim formundan gelenler)
  Ad, Eposta, Konu, Icerik, OkunduMu

Abone              (yan sütundaki abone ol formundan)
  Eposta, KayitTarihi
```

Altı tablo + bir ara tablo. `Etiket` ve `Sayfa` yapıları bildiğiniz CRUD'un aynısı — hızlı geçeceğiz.

---

## Ders takvimi

**11-13 ders saati.**

| # | Modül | Süre | Dosya |
|---|---|---|---|
| 1 | Bootstrap'i kendimize uyarlamak | **40 dk** | `01-bootstrap-ve-tasarim.md` |
| 2 | **Model, slug ve veritabanı** | 2 saat | `02-model-ve-veritabani.md` |
| 3 | **İki layout** | 1-2 saat | `03-iki-layout.md` |
| 4 | Yönetim girişi | 1 saat | `04-yonetim-girisi.md` |
| 5 | **Yazı yönetimi + Markdown** | 2 saat | `05-yazi-yonetimi.md` |
| 6 | **Genel site** | 2 saat | `06-genel-site.md` |
| 7 | Dinamik sayfalar | 1 saat | `07-sayfalar.md` |
| 8 | Etiketler | 1 saat | `08-etiketler.md` |
| 9 | **İletişim + abonelik** | 1 saat | `09-iletisim.md` |
| 10 | **Görsel yükleme** | 1 saat | `10-gorsel-yukleme.md` |
| 11 | Son dokunuşlar | 1 saat | `11-son-dokunuslar.md` |

---

## Proje yapısı

```
KisiselBlog/
│
├── Program.cs
├── appsettings.json
│
├── wwwroot/
│   ├── lib/bootstrap/          şablondan gelir, KALIYOR
│   ├── css/site.css            ⭐ ~140 satır: Bootstrap üzerine ince bir kat
│   └── yuklenen/               görseller (Modül 10)
│
├── Data/
│   ├── BlogDbContext.cs
│   ├── SlugYardimcisi.cs       ⭐ Türkçe slug
│   ├── MarkdownYardimcisi.cs   ⭐ Markdown → HTML
│   ├── GorselYardimcisi.cs     ⭐ dosya yükleme (Modül 10)
│   └── SifreYardimcisi.cs
│
├── Models/
│   ├── Yazi.cs, Etiket.cs, Sayfa.cs, Mesaj.cs, Abone.cs, Kullanici.cs
│   ├── Enums.cs                ⭐ YaziDurum
│   └── SiteAyarlari.cs         ⭐ appsettings → C#
│
├── Controllers/
│   ├── GenelController.cs         ⭐ taban — [AllowAnonymous]
│   ├── YonetimControllerTabani.cs ⭐ taban — [Authorize]
│   │
│   ├── AnasayfaController.cs      genel site
│   ├── YaziController.cs          genel site — slug, etiket, rastgele
│   ├── SayfaController.cs         genel site
│   ├── IletisimController.cs      genel site
│   ├── AbonelikController.cs      genel site
│   ├── AramaController.cs         genel site
│   ├── BeslemeController.cs       RSS, sitemap
│   ├── HataController.cs          404 / 500
│   ├── HesapController.cs         giriş / çıkış
│   │
│   ├── PanelController.cs         yönetim
│   ├── YaziYonetimController.cs   yönetim
│   ├── SayfaYonetimController.cs  yönetim
│   ├── EtiketYonetimController.cs yönetim
│   └── MesajYonetimController.cs  yönetim + aboneler
│
├── ViewComponents/
│   └── MenuViewComponent.cs       ⭐ Modül 7
│
└── Views/
    ├── Shared/
    │   ├── _Layout.cshtml         ⭐ genel site — yan sütunlu
    │   ├── _YanSutun.cshtml       ⭐ yan sütun
    │   ├── _YaziKarti.cshtml      ⭐ üç yerde kullanılan yazı kartı
    │   └── _YonetimLayout.cshtml  ⭐ yönetim — navbar'lı
    │
    ├── Anasayfa/, Yazi/, Sayfa/, Iletisim/, Arama/, Hata/, Hesap/
    │
    └── Yonetim/                   ⭐ panel view'ları
        ├── _ViewStart.cshtml      (layout'u ezer)
        ├── Panel/, YaziYonetim/, SayfaYonetim/,
        └── EtiketYonetim/, MesajYonetim/
```

> Yönetim **controller'ları** `Controllers/` kökünde, **view'ları** `Views/Yonetim/` altında.
> Bu ayrımın sebebi Modül 3'te: `_ViewStart.cshtml` klasör bazlı çalışıyor, controller'lar için
> böyle bir mekanizma yok.

---

## Kurulum

```bash
dotnet new mvc -n KisiselBlog
cd KisiselBlog

dotnet add package Microsoft.EntityFrameworkCore.SqlServer
dotnet add package Microsoft.EntityFrameworkCore.Design
dotnet add package Markdig                       # ⭐ yeni: Markdown
```

`appsettings.json`:

```json
{
  "ConnectionStrings": {
    "BlogDb": "Server=localhost\\SQLEXPRESS;Database=BlogDB;Trusted_Connection=True;TrustServerCertificate=True;"
  },
  "Logging": {
    "LogLevel": { "Default": "Information", "Microsoft.AspNetCore": "Warning" }
  },
  "AllowedHosts": "*"
}
```

> Veritabanını elle oluşturmayın — Modül 2'de migration üretecek.

### Bootstrap şablondan geliyor, silmiyoruz

`dotnet new mvc` şablonu Bootstrap 5'le birlikte gelir. Bu projede **kullanıyoruz.** Silmeyin.

Tek yapacağınız:

- [ ] `wwwroot/css/site.css` içeriğini boşaltın — Modül 1'de kendi katmanımızı yazacağız
- [ ] `Views/Shared/_Layout.cshtml`'i Modül 3'te baştan yazacağız

---

## Ön koşul

Bu rehber şunları bildiğinizi varsayar ve tekrar anlatmaz:

- ASP.NET Core MVC (controller, view, routing temeli)
- EF Core (DbContext, migration, LINQ, `Include`, query filter)
- Çerez tabanlı giriş sistemi
- CRUD kalıbı
- Bootstrap sınıflarının temeli (`row`, `col`, `btn`, `form-control`, `table`)

Eksiğiniz varsa önce GörevTakip-EF rehberini tamamlayın.

**Yeni olan tek paket:** Markdig.

---

👉 Başla: [`01-bootstrap-ve-tasarim.md`](01-bootstrap-ve-tasarim.md)
