# Modül 11 — Son Dokunuşlar

**Süre:** 1 ders saati

---

## 🎯 Bu derste ne yapacağız

Site çalışıyor. Şimdi onu **bitmiş** hissettiren parçalar:

1. Arama
2. RSS beslemesi
3. Site haritası (sitemap) ve SEO etiketleri
4. Rastgele yazı
5. Yayına çıkış kontrol listesi

---

## ⌨️ Adım 1: Arama

Basit ama işe yarar: başlık ve içerikte metin arama.

`Controllers/AramaController.cs`:

```csharp
using Microsoft.AspNetCore.Mvc;
using Microsoft.EntityFrameworkCore;
using KisiselBlog.Data;
using KisiselBlog.Models;

namespace KisiselBlog.Controllers;

public class AramaController : GenelController
{
    private readonly BlogDbContext _db;

    public AramaController(BlogDbContext db)
    {
        _db = db;
    }

    // GET: /ara?q=veritabani
    [Route("/ara")]
    public async Task<IActionResult> Index(string? q)
    {
        ViewData["Title"] = string.IsNullOrWhiteSpace(q) ? "Ara" : $"\"{q}\" araması";
        ViewBag.Sorgu = q;

        if (string.IsNullOrWhiteSpace(q) || q.Trim().Length < 2)
        {
            ViewBag.Uyari = q != null ? "En az 2 karakter yazın." : null;
            return View(new List<Yazi>());
        }

        string temiz = q.Trim();

        var sonuclar = await _db.Yazilar
            .AsNoTracking()
            .Where(y => y.Durum == YaziDurum.Yayinda
                     && (y.Baslik.Contains(temiz)
                      || y.Icerik.Contains(temiz)
                      || (y.Ozet != null && y.Ozet.Contains(temiz))))
            .Include(y => y.Etiketler)
            // ⭐ Başlıkta geçenler önce — daha alakalı
            .OrderByDescending(y => y.Baslik.Contains(temiz))
            .ThenByDescending(y => y.YayinTarihi)
            .Take(50)
            .ToListAsync();

        return View(sonuclar);
    }
}
```

### ⚠️ `LIKE '%...%'` neden yavaş?

`Contains("x")` SQL'de `LIKE N'%x%'` olur. Baştaki `%` yüzünden veritabanı **indeks kullanamaz** — her satıra tek tek bakar.

| Yazı sayısı | Yaklaşık süre |
|---|---|
| 100 | Fark edilmez |
| 10.000 | Yavaşlamaya başlar |
| 1.000.000 | Kullanılamaz |

Kişisel blogda 100-500 yazı olur; sorun değil. Ama öğrenci bunun bir **maliyeti** olduğunu bilmeli.

**Büyük siteler ne yapıyor?**
- SQL Server Full-Text Search
- Elasticsearch / Meilisearch
- PostgreSQL `tsvector`

> **İlke:** Önce çalışsın, ölç, gerekirse hızlandır. Erken optimizasyon zaman kaybıdır.

`Views/Arama/Index.cshtml`:

```html
@model List<KisiselBlog.Models.Yazi>
@{
    ViewData["Title"] = "Ara";
    string? sorgu = ViewBag.Sorgu;
    string? uyari = ViewBag.Uyari;
}

<h1 class="mb-4">Ara</h1>

<form method="get" action="/ara" class="mb-5" style="max-width: 30rem;">
    <div class="input-group">
        <input type="search" name="q" value="@sorgu" class="form-control"
               placeholder="Ne arıyorsunuz?" aria-label="Arama terimi" autofocus />
        <button type="submit" class="btn btn-sari">Ara</button>
    </div>
</form>

@if (uyari != null)
{
    <p style="color: var(--soluk);">@uyari</p>
}
else if (!string.IsNullOrWhiteSpace(sorgu))
{
    @if (!Model.Any())
    {
        <div class="text-center py-5" style="color: var(--soluk);">
            <i class="bi bi-search display-6"></i>
            <p class="mt-3">"<strong>@sorgu</strong>" için sonuç bulunamadı.</p>
            <p class="mb-0">
                <a asp-controller="Anasayfa" asp-action="Index">Tüm yazılara bakın</a>
            </p>
        </div>
    }
    else
    {
        <p class="small mb-4" style="color: var(--soluk);">@Model.Count sonuç</p>

        @* ⭐ Modül 6'da yazdığımız kartı tekrar kullanıyoruz *@
        @foreach (var yazi in Model)
        {
            <partial name="_YaziKarti" model="yazi" />
        }
    }
}
```

### 📖 `input-group`

Metin kutusuyla butonu **tek bir parça** gibi birleştirir: aradaki boşluk kalkar, köşeler
sadece dışta yuvarlanır. Arama, abonelik, miktar girişi gibi "kutu + buton" ikililerinde
Bootstrap'in standart çözümü.

> Yan sütundaki arama kutusu bu değil — orada `arama-kutu` sınıfıyla alt çizgili, butonsuz
> bir alan var. Aynı işi iki farklı biçimde sunuyoruz: yan sütunda sessiz, arama sayfasında
> belirgin. Bu bilinçli bir tercih; yan sütun her sayfada görünüyor ve dikkat çekmemeli.

### ⭐ Kart yeniden kullanımı

`_YaziKarti` bu modülle birlikte **üç** yerde kullanılıyor: ana sayfa, etiket sayfası ve
arama. Markup'ı kısmi görünüme almasaydık burada dördüncü kopyayı yazıyor olurduk.

> **Sor:** *"Yazı kartına bir 'okuma süresi' eklemek istesek kaç dosyaya dokunurduk?"*
> Şimdi: bir. Kopyalasaydık: üç — ve birini unutmak kesin.

Arama kutusu zaten yan sütunda (Modül 3), bu yüzden ayrıca menü bağlantısı eklemeye gerek yok.

---


## ⌨️ Adım 2: RSS beslemesi

**Sınıfa sor:** *"RSS nedir, neden hâlâ var?"*

Okuyucu, sitenizi ziyaret etmeden yeni yazılarınızdan haberdar olmak ister. RSS, bunun **açık standardı**. Bir XML dosyası; okuyucu uygulamaları onu takip eder.

Sosyal medya algoritmalarına bağımlı olmayan tek dağıtım kanalı.

`Controllers/BeslemeController.cs`:

```csharp
using System.Text;
using System.Xml;
using Microsoft.AspNetCore.Mvc;
using Microsoft.EntityFrameworkCore;
using KisiselBlog.Data;
using KisiselBlog.Models;

namespace KisiselBlog.Controllers;

public class BeslemeController : GenelController
{
    private readonly BlogDbContext _db;
    private readonly SiteAyarlari _site;      // ⭐ appsettings'ten (Modül 3)

    public BeslemeController(BlogDbContext db, SiteAyarlari site)
    {
        _db = db;
        _site = site;
    }

    // GET: /rss
    [Route("/rss")]
    public async Task<IActionResult> Rss()
    {
        var yazilar = await _db.Yazilar
            .AsNoTracking()
            .Where(y => y.Durum == YaziDurum.Yayinda)
            .OrderByDescending(y => y.YayinTarihi)
            .Take(20)
            .ToListAsync();

        string temelAdres = $"{Request.Scheme}://{Request.Host}";

        var sb = new StringBuilder();

        // ⭐ XmlWriter kullanıyoruz, elle string birleştirmiyoruz.
        //    Sebep: başlıkta & veya < geçerse XML bozulur.
        //    XmlWriter kaçırmayı (escaping) kendisi yapar.
        var ayarlar = new XmlWriterSettings
        {
            Indent = true,
            Encoding = Encoding.UTF8,
            OmitXmlDeclaration = false
        };

        using (var yazici = XmlWriter.Create(sb, ayarlar))
        {
            yazici.WriteStartDocument();
            yazici.WriteStartElement("rss");
            yazici.WriteAttributeString("version", "2.0");

            // atom: ad alanı — self link için gerekli
            yazici.WriteAttributeString("xmlns", "atom", null,
                "http://www.w3.org/2005/Atom");

            yazici.WriteStartElement("channel");

            yazici.WriteElementString("title", _site.Baslik);
            yazici.WriteElementString("link", temelAdres);
            yazici.WriteElementString("description", _site.Aciklama);
            yazici.WriteElementString("language", "tr");

            yazici.WriteElementString("lastBuildDate",
                DateTime.Now.ToString("R"));     // RFC 1123 biçimi

            // Beslemenin kendi adresi
            yazici.WriteStartElement("atom", "link",
                "http://www.w3.org/2005/Atom");
            yazici.WriteAttributeString("href", $"{temelAdres}/rss");
            yazici.WriteAttributeString("rel", "self");
            yazici.WriteAttributeString("type", "application/rss+xml");
            yazici.WriteEndElement();

            foreach (var yazi in yazilar)
            {
                string adres = $"{temelAdres}/yazi/{yazi.Slug}";

                yazici.WriteStartElement("item");

                yazici.WriteElementString("title", yazi.Baslik);
                yazici.WriteElementString("link", adres);

                // guid: bu öğenin benzersiz kimliği.
                // Okuyucu uygulamalar "bunu gördüm mü?" için kullanır.
                yazici.WriteStartElement("guid");
                yazici.WriteAttributeString("isPermaLink", "true");
                yazici.WriteString(adres);
                yazici.WriteEndElement();

                yazici.WriteElementString("description",
                    MarkdownYardimcisi.DuzMetin(yazi.Ozet ?? yazi.Icerik, 300));

                if (yazi.YayinTarihi.HasValue)
                {
                    yazici.WriteElementString("pubDate",
                        yazi.YayinTarihi.Value.ToString("R"));
                }

                yazici.WriteEndElement();   // item
            }

            yazici.WriteEndElement();       // channel
            yazici.WriteEndElement();       // rss
            yazici.WriteEndDocument();
        }

        return Content(sb.ToString(), "application/rss+xml", Encoding.UTF8);
    }

    // GET: /sitemap.xml
    [Route("/sitemap.xml")]
    public async Task<IActionResult> SiteHaritasi()
    {
        string temelAdres = $"{Request.Scheme}://{Request.Host}";

        var yazilar = await _db.Yazilar
            .AsNoTracking()
            .Where(y => y.Durum == YaziDurum.Yayinda)
            .Select(y => new { y.Slug, y.YayinTarihi, y.UpdatedDate })
            .ToListAsync();

        var sayfalar = await _db.Sayfalar
            .AsNoTracking()
            .Select(s => new { s.Slug, s.UpdatedDate, s.CreatedDate })
            .ToListAsync();

        var sb = new StringBuilder();
        var ayarlar = new XmlWriterSettings { Indent = true, Encoding = Encoding.UTF8 };

        using (var yazici = XmlWriter.Create(sb, ayarlar))
        {
            yazici.WriteStartDocument();
            yazici.WriteStartElement("urlset",
                "http://www.sitemaps.org/schemas/sitemap/0.9");

            // Ana sayfa
            YazAdres(yazici, temelAdres, DateTime.Now, "daily", "1.0");

            foreach (var s in sayfalar)
            {
                YazAdres(yazici, $"{temelAdres}/{s.Slug}",
                    s.UpdatedDate ?? s.CreatedDate, "monthly", "0.8");
            }

            foreach (var y in yazilar)
            {
                YazAdres(yazici, $"{temelAdres}/yazi/{y.Slug}",
                    y.UpdatedDate ?? y.YayinTarihi ?? DateTime.Now, "monthly", "0.6");
            }

            yazici.WriteEndElement();
            yazici.WriteEndDocument();
        }

        return Content(sb.ToString(), "application/xml", Encoding.UTF8);
    }

    private static void YazAdres(XmlWriter yazici, string adres,
                                 DateTime tarih, string siklik, string oncelik)
    {
        yazici.WriteStartElement("url");
        yazici.WriteElementString("loc", adres);
        yazici.WriteElementString("lastmod", tarih.ToString("yyyy-MM-dd"));
        yazici.WriteElementString("changefreq", siklik);
        yazici.WriteElementString("priority", oncelik);
        yazici.WriteEndElement();
    }

    // GET: /robots.txt
    [Route("/robots.txt")]
    public IActionResult Robots()
    {
        string temelAdres = $"{Request.Scheme}://{Request.Host}";

        // ⭐ Yönetim panelini arama motorlarına kapat
        string icerik = $"""
            User-agent: *
            Allow: /
            Disallow: /yonetim
            Disallow: /giris

            Sitemap: {temelAdres}/sitemap.xml
            """;

        return Content(icerik, "text/plain", Encoding.UTF8);
    }
}
```

> ⚠️ `robots.txt` bir **rica**dır, kilit değil. İyi niyetli arama motorları uyar, kötü niyetli botlar umursamaz. Asıl koruma `[Authorize]`.

### Layout'a besleme bağlantısı

`_Layout.cshtml`'in `<head>` kısmına:

```html
<link rel="alternate" type="application/rss+xml"
      title="@Site.Baslik — RSS" href="/rss" />
```

Bu satır, tarayıcı eklentilerinin ve okuyucu uygulamalarının beslemeyi **otomatik bulmasını** sağlar.

### ⚠️ Yasaklı slug listesini güncelleyin

Yeni rotalar ekledik. `SlugYardimcisi.YasakliSluglar` listesine ekleyin:

```csharp
"rastgele", "sitemap-xml", "robots-txt", "besleme"
```

> Modül 7'de uyarmıştık: *"Bu listeye yeni rota eklediğinizde buraya da ekleyin."* İşte o an.
>
> ⚠️ **Noktalar neden tireye döndü?** `SlugHazirla` önce `SlugYardimcisi.Uret()` çağırıyor,
> o da noktayı tireye çeviriyor. Yani aday slug hiçbir zaman `"sitemap.xml"` olamaz —
> `"sitemap-xml"` olur. Listeye noktalı hâlini yazsaydınız kural **hiç çalışmazdı.**
>
> **Derste yapın:** Önce `"sitemap.xml"` yazın, bir sayfaya "Sitemap.xml" başlığı verin,
> engellenmediğini görün. Sonra düzeltin.

---

## ⌨️ Adım 3: Kanonik adres

Aynı içerik birden fazla adresten erişilebiliyorsa arama motorları kafa karışıklığı yaşar:

```
https://site.com/yazi/merhaba
https://www.site.com/yazi/merhaba
https://site.com/yazi/merhaba?utm_source=twitter
```

**Çözüm: kanonik etiket.** "Bu içeriğin gerçek adresi şu" demek.

`_Layout.cshtml`'in `<head>` kısmına:

```html
@{
    var kanonik = $"{Context.Request.Scheme}://{Context.Request.Host}" +
                  $"{Context.Request.Path}";
}
<link rel="canonical" href="@kanonik" />
```

> Sorgu parametresini (`?utm_source=...`) bilerek dışarıda bıraktık — kanonik adres temiz olmalı.

---

## ⌨️ Adım 4: Rastgele yazı

Yan sütundaki "KEŞFET → Rastgele yazı" bağlantısı Modül 3'ten beri bir yere gitmiyordu. Şimdi dolduruyoruz.

Eski yazılar zamanla görünmez olur: ana sayfanın üçüncü sayfasına kimse inmiyor. Rastgele bağlantı, arşivi tekrar dolaşıma sokan ucuz bir yöntem.

`Controllers/YaziController.cs`'e ekleyin:

```csharp
// ══════════════════════════════════════════════════════
//  RASTGELE YAZI
//  GET: /rastgele
// ══════════════════════════════════════════════════════
[HttpGet("/rastgele")]
public async Task<IActionResult> Rastgele()
{
    // ⭐ Guid.NewGuid() ile sıralamak, SQL Server'da NEWID()'ye çevrilir.
    //    Her satıra rastgele bir değer atanıp ona göre sıralanır.
    var slug = await _db.Yazilar
        .AsNoTracking()
        .Where(y => y.Durum == YaziDurum.Yayinda)
        .OrderBy(y => Guid.NewGuid())
        .Select(y => y.Slug)
        .FirstOrDefaultAsync();

    // Hiç yayınlanmış yazı yoksa ana sayfaya dön
    if (slug == null)
        return RedirectToAction("Index", "Anasayfa");

    // ⭐ 302 (geçici) yönlendirme — 301 DEĞİL.
    //    301 kalıcı demektir; tarayıcı onu önbelleğe alır ve
    //    bir daha sunucuya sormadan hep aynı yazıya gider.
    //    "Rastgele" o anda bozulur ve düzeltmesi zordur.
    return RedirectToAction(nameof(Detay), new { slug });
}
```

### ⚠️ `OrderBy(y => Guid.NewGuid())` ve maliyeti

Bu sorgu **tüm tabloyu** okuyup her satıra bir GUID atar, sonra sıralar. 200 yazıda fark etmez; 200.000 satırda ciddi yavaşlar.

Büyük tablolar için yaygın yol: önce `Count()` al, `Random` ile bir sayı seç, `Skip(n).Take(1)` yap. İki sorgu olur ama ikisi de hızlıdır.

```csharp
// Büyük tablolar için alternatif — bu projede gerekmiyor
int adet = await _db.Yazilar.CountAsync(y => y.Durum == YaziDurum.Yayinda);
int atla = Random.Shared.Next(adet);
var slug = await _db.Yazilar
    .Where(y => y.Durum == YaziDurum.Yayinda)
    .OrderBy(y => y.YaziId).Skip(atla).Take(1)
    .Select(y => y.Slug).FirstOrDefaultAsync();
```

> **Sor:** *"İlk yol daha kısa. Neden ikincisini bilmemiz gerekiyor?"*
> Çünkü "çalışıyor" ile "ölçeklenir" aynı şey değil. Veri büyüdüğünde ilk çözen kodun nerede çatırdayacağını önceden bilmek, sonradan keşfetmekten ucuzdur.

### Yan sütun bağlantısı

Modül 3'te yazdığımız bağlantı zaten doğru action'a işaret ediyor:

```html
<a asp-controller="Yazi" asp-action="Rastgele" ...>
```

Şimdi `href="/rastgele"` üretecek. Sayfayı yenileyip bağlantının artık boş olmadığını kontrol edin — Modül 3'te "henüz olmayan action'lar boş `href` üretir" demiştik, bunun karşılığını şimdi görüyorsunuz.

---

## 📋 Yayına çıkış kontrol listesi

Siteyi gerçekten yayınlamadan önce. **Bu listeyi derste tek tek geçin.**

### Güvenlik

```
☐ Giriş sayfasındaki "Ders ortamı: admin / blog2026" kutusu SİLİNDİ
☐ Şifre değiştirildi (seed'deki değil)
☐ EnableSensitiveDataLogging() sadece IsDevelopment() içinde
☐ LogTo(Console) sadece IsDevelopment() içinde
☐ appsettings.json'daki bağlantı dizesi git'e girmiyor
☐ Markdig pipeline'ında .DisableHtml() var
☐ Mesaj içeriğinde Html.Raw YOK
☐ Cookie.HttpOnly = true
☐ Cookie.SameSite = Lax
☐ UseHttpsRedirection() açık
☐ UseHsts() üretim ortamında açık
```

### İçerik

```
☐ Hakkımda sayfası gerçek metinle dolduruldu
☐ Site adı ve açıklaması güncellendi
☐ En az 3 gerçek yazı yayınlandı
☐ Taslaklar sitede görünmüyor (kontrol edildi)
☐ 404 sayfası düzgün görünüyor
```

### Teknik

```
☐ /rss geçerli XML döndürüyor
☐ /sitemap.xml geçerli
☐ /robots.txt yönetimi kapatıyor
☐ Yönetim sayfalarında noindex meta etiketi var
☐ Tüm görsellerde alt özniteliği var
☐ Mobilde test edildi (yan sütun hamburgerde açılıyor)
☐ Yatay kaydırma çubuğu hiçbir sayfada yok
☐ Klavyeyle gezinilebiliyor (Tab)
```

### Erişilebilirlik

```
☐ Metin kontrastları en az 4.5:1 (Lighthouse ile ölçüldü)
☐ Sarı (#ffd43b) metin olarak hiçbir yerde kullanılmıyor
☐ Odak halkası görünür (Bootstrap'in varsayılanı kaldırılmadı)
☐ Sayfa başına tek h1 var
☐ Bal küpü alanında aria-hidden var
☐ Form alanlarının label'ı var
```

### Performans

```
☐ Liste görsellerinde loading="lazy"
☐ Google Fonts'ta display=swap
☐ Listelerde AsNoTracking kullanılıyor
☐ Konsolda N+1 sorgu yok (kontrol edildi)
```

---

## ▶️ Son test turu

| # | Test | Beklenen |
|---|---|---|
| 1 | `/ara?q=veritabani` | Sonuçlar, başlıkta geçenler üstte |
| 2 | `/ara?q=x` | "En az 2 karakter" |
| 3 | `/rss` | Geçerli XML |
| 4 | `/rss`'te taslak yazı | **Yok** |
| 5 | `/sitemap.xml` | Tüm yayın ve sayfalar |
| 6 | `/robots.txt` | `Disallow: /yonetim` |
| 7 | Yazı sayfası → F12 → Elements | `<link rel="canonical">` var |
| 8 | Tarayıcıda RSS eklentisi | Beslemeyi otomatik buluyor |
| 9 | Başlığında `&` olan yazı → `/rss` | XML bozulmuyor |
| 10 | Tüm sayfalar, telefon genişliği | Yan sütun hamburgerde, yatay kaydırma yok |

> **9. madde önemli.** `XmlWriter` kullanmasaydık, başlıktaki `&` karakteri XML'i bozardı. Elle string birleştirmenin tehlikesi bu.

---

## 🎓 Kurs sonu

### Bu projede ne öğrendik?

| Konu | Anahtar |
|---|---|
| **Bootstrap'i uyarlama** | CSS değişkenlerini ezme, `-rgb` tuzağı, ölçülmüş kontrast |
| **İki layout** | `_ViewStart` klasör bazlı, aynı Bootstrap farklı bileşenler |
| **Duyarlı yerleşim** | `row`/`col`, `collapse d-lg-block`, `sticky` yan sütun |
| **Slug** | Türkçe harf çevirimi, `ToLowerInvariant` tuzağı, benzersizlik |
| **Markdown** | Okurken çevirme, `DisableHtml`, `Html.Raw` sorumluluğu |
| **Ters yetki** | Varsayılan kapalı, taban sınıfla açma |
| **Anonim yazma** | Bal küpü, zaman kontrolü, XSS |
| **Dosya yükleme** | Beyaz liste, magic bytes, GUID adlandırma |
| **Rota** | Öznitelik rotaları, yakalayıcı rota, yasaklı slug |

### Dört projeye bakış

| | Okul (ADO.NET) | GörevTakip | UBYS-EF | **Blog** |
|---|---|---|---|---|
| Veri erişimi | ADO.NET | ADO.NET | EF Core | EF Core |
| Kullanıcı | Tek admin | Tek admin | Tek admin | Tek admin + **herkese açık** |
| Arayüz | Bootstrap | Bootstrap | Bootstrap | Bootstrap + ⭐ **kendi katmanı** |
| Adresler | `/X/Y/5` | `/X/Y/5` | `/X/Y/5` | ⭐ **Slug** |
| Yeni kavram | CRUD | Enum, soft delete | Migration, LINQ | ⭐ **Slug, Markdown, yükleme** |

**Sınıfa son soru:** *"Dört proje de aslında aynı şeyi yapıyordu: veritabanından kayıt okuyup ekrana bastı. Peki neden dördünü de yaptık?"*

Çünkü her seferinde **aynı problemi farklı bir açıdan** gördünüz. Ve bu projede, aynı CRUD'un bir **kişisel site** hâline gelmesi için neyin gerektiğini öğrendiniz: adres yapısı, içerik biçimi, güvenlik katmanları, iki farklı kullanıcı deneyimi — ve hazır bir arayüz kütüphanesinin kendinize ait hissettirecek kadar uyarlanabileceğini.

### Bundan sonra

```
BURADASINIZ
    │
    ├──▶ Yayınlama           (Azure, Railway, kendi sunucunuz)
    ├──▶ Alan adı ve HTTPS   (Let's Encrypt)
    ├──▶ ASP.NET Core Identity  (hazır kullanıcı sistemi)
    ├──▶ Web API + JavaScript   (mobil uygulama da bağlanabilir)
    ├──▶ Blazor / React         (daha etkileşimli arayüz)
    └──▶ Birim testleri, CI/CD
```

---

## ✏️ Son alıştırmalar

1. Yayına çıkış kontrol listesini tek tek geçin. Kaç madde eksikti?

2. **İlgili yazılar:** Yazı sonuna, aynı etiketleri paylaşan 3 yazı ekleyin.
   İpucu: `Where(y => y.Etiketler.Any(e => mevcutEtiketIdler.Contains(e.EtiketId)))`

3. **Arşiv sayfası:** `/arsiv` — yazıları yıla göre gruplanmış liste.

4. **Okuma ilerleme çubuğu:** Yazı sayfasında, sayfanın üstünde kaydırma ilerlemesini gösteren ince bir çubuk.
   İpucu: `window.scrollY / (document.body.scrollHeight - window.innerHeight)`

5. **Yazdırma stili:** `@media print` ile yan sütunu ve butonları gizleyin (`.yan-sutun { display: none }`), yazıyı sayfa genişliğine yayın.

6. **Siteyi gerçekten yayınlayın.** Ücretsiz seçenekler: Azure App Service (F1), Railway, Render.

**Son düşünme soruları:**
- Bootstrap'i altmış satırla kendimize uyarladık. Hangi noktada "artık kendi CSS'imi yazsam daha iyi" derdiniz?
- Bu projede en çok hangi güvenlik kararını önemli buldunuz? Neden?
- Siteyi bir arkadaşınıza gösterseniz, ilk neyi değiştirmek isterdi? Siz neyi savunurdunuz?

---

🏁 **Kurs tamamlandı.**
