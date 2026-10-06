# Modül 6 — Genel Site

**Süre:** 2 ders saati

---

## 🎯 Bu derste ne yapacağız

Okuyucunun gördüğü tarafı. **İki yeni konu:**

1. **Slug ile yönlendirme** — `/yazi/merhaba-dunya`
2. Markdown'ı sayfaya basmak

Yanında sayfalama, okunma sayacı, etiket sayfaları ve düzgün bir 404.

Hedef yerleşim:

```
┌───────────────┬──────────────────────────────────────┐
│               │  24 OCAK 2026              ← küçük   │
│  YAN SÜTUN    │                                      │
│  (Modül 3'te  │  Tuhaf Zamanlar            ← büyük   │
│   yazıldı)    │                                      │
│               │  Zeki olmak bir zamanlar kapıları    │
│               │  açardı; bugünse o kapıları…         │
│               │                                      │
│               │  DEVAMINI OKU →                      │
│               │  ──────────────────────────────      │
│               │  12 OCAK 2026                        │
└───────────────┴──────────────────────────────────────┘
```

Tasarım için yeni CSS yazmıyoruz. Modül 1'de hazırladığımız dört sınıf (`yazi-tarih`, `baslik-link`, `devam`, `icerik-alani`) ve Bootstrap'in boşluk sınıfları yeterli.

---

## 📖 Slug yönlendirmesi

Varsayılan MVC rotası şöyle:

```
{controller}/{action}/{id?}     →  /Yazi/Detay/5
```

Biz bunu istiyoruz:

```
/yazi/merhaba-dunya
```

**Neden?** Üçü birden:

| Sebep | Açıklama |
|---|---|
| Okunabilirlik | Adres, içeriğin ne olduğunu söylüyor |
| Paylaşılabilirlik | WhatsApp'ta görülen adres anlamlı |
| Arama motoru | Adresteki kelimeler sıralamaya katkı yapıyor |

### Öznitelik tabanlı rota

```csharp
[Route("yazi")]
public class YaziController : GenelController
{
    [HttpGet("{slug}")]              //  /yazi/merhaba-dunya
    public async Task<IActionResult> Detay(string slug) { }
}
```

`{slug}` bir **yer tutucu**. Adresteki o kısım metot parametresine geçer.

### ⚠️ Rota kısıtları (constraints)

Yer tutucuya tip veya desen şartı eklenebilir:

```csharp
[HttpGet("{id:long}")]           // sadece sayı
[HttpGet("{slug:minlength(2)}")] // en az 2 karakter
[HttpGet("{sayfa:int:min(1)}")]  // 1 veya daha büyük tam sayı
```

Kısıt sağlanmazsa rota **eşleşmez** — 404 döner. Bu iyi bir şey: geçersiz girdi controller'a hiç ulaşmaz.

### Sıralama önemli

```csharp
[HttpGet("arsiv")]          // 1. önce sabit yollar
[HttpGet("{slug}")]         // 2. sonra değişkenler
```

Ters yazsaydınız `/yazi/arsiv` adresi, slug'ı "arsiv" olan bir yazı arardı ve bulamazdı.

> ASP.NET Core aslında sabit segmentleri değişkenlere göre **öncelikli** değerlendirir, yani ters yazsanız da çalışır. Ama okunabilirlik için doğru sırayla yazmak alışkanlık olmalı.

---

## ⌨️ Adım 1 — Ana sayfa controller'ı

`Controllers/AnasayfaController.cs` — Modül 3'teki geçici hâlinin yerine:

```csharp
using Microsoft.AspNetCore.Mvc;
using Microsoft.EntityFrameworkCore;
using KisiselBlog.Data;
using KisiselBlog.Models;

namespace KisiselBlog.Controllers;

public class AnasayfaController : GenelController
{
    private readonly BlogDbContext _db;

    // Sayfa başına yazı sayısı
    private const int SayfaBoyutu = 10;

    public AnasayfaController(BlogDbContext db)
    {
        _db = db;
    }

    // GET: /  veya  /?sayfa=2
    public async Task<IActionResult> Index(int sayfa = 1)
    {
        if (sayfa < 1) sayfa = 1;

        // ⭐ GENEL SİTENİN ALTIN KURALI:
        //    Sadece YAYINDA olanlar görünür.
        //
        //    Query filter AktifMi'yi hallediyor ama durum
        //    filtresini her genel sorguda ELLE yazmamız gerekiyor.
        //
        //    ⚠️ Bunu unutmak = taslakların siteye sızması.
        IQueryable<Yazi> sorgu = _db.Yazilar
            .AsNoTracking()
            .Where(y => y.Durum == YaziDurum.Yayinda)
            .Include(y => y.Etiketler);

        int toplam = await sorgu.CountAsync();

        var yazilar = await sorgu
            .OrderByDescending(y => y.YayinTarihi)
            .Skip((sayfa - 1) * SayfaBoyutu)
            .Take(SayfaBoyutu)
            .ToListAsync();

        ViewBag.Sayfa = sayfa;
        ViewBag.ToplamSayfa = (int)Math.Ceiling((double)toplam / SayfaBoyutu);
        ViewBag.ToplamYazi = toplam;

        return View(yazilar);
    }
}
```

### ⚠️ `(double)` dönüşümü

```csharp
(int)Math.Ceiling((double)toplam / SayfaBoyutu)
```

`(double)` olmadan tam sayı bölmesi yapılır:

```
23 / 10  = 2      → 3. sayfadaki 3 yazı KAYBOLUR
23.0 / 10 = 2.3   → Ceiling → 3   ✓
```

**Derste gösterin:** `(double)`'ı silin, 11 yazı yayınlayın, sayfalamanın son yazıyı yuttuğunu görün.

---

## ⌨️ Adım 2 — Yazı kartı: tekrar etmeyen parça

Yazı listesi üç yerde görünecek: ana sayfa, etiket sayfası, arama sonuçları. Aynı markup'ı üç kere yazmak yerine kısmi görünüm yapıyoruz.

`Views/Shared/_YaziKarti.cshtml`:

```html
@model KisiselBlog.Models.Yazi

<article class="mb-5 pb-4 border-bottom">

    @* ─── TARİH ─── *@
    @if (Model.YayinTarihi.HasValue)
    {
        <p class="yazi-tarih mb-2">
            <time datetime="@Model.YayinTarihi.Value.ToString("yyyy-MM-dd")">
                @Model.YayinTarihi.Value.ToString("d MMMM yyyy").ToUpper(
                    new System.Globalization.CultureInfo("tr-TR"))
            </time>
        </p>
    }

    @* ─── BAŞLIK ─── *@
    <h2 class="h1 mb-3">
        <a asp-controller="Yazi" asp-action="Detay" asp-route-slug="@Model.Slug"
           class="baslik-link">@Model.Baslik</a>
    </h2>

    @* ─── ÖZET ─── *@
    <p class="mb-3">@Model.GorunenOzet</p>

    @* ─── ALT SATIR ─── *@
    <div class="d-flex flex-wrap align-items-center gap-3">

        <a asp-controller="Yazi" asp-action="Detay" asp-route-slug="@Model.Slug"
           class="devam">
            Devamını oku <span aria-hidden="true">&rarr;</span>
        </a>

        @if (Model.Etiketler.Any())
        {
            <span class="d-flex flex-wrap">
                @foreach (var etiket in Model.Etiketler.OrderBy(e => e.Ad))
                {
                    <a href="/etiket/@etiket.Slug" class="etiket-link">#@etiket.Ad</a>
                }
            </span>
        }
    </div>

</article>
```

### 📖 Üç ayrıntı

**`<h2 class="h1">`** — başlığı `h2` olarak işaretliyor ama `h1` boyutunda gösteriyor. Niye?

Sayfada **tek bir `h1`** olmalı; o da yan sütundaki site adı. Yazı listesindeki her başlık bir alt seviye, yani `h2`. Ama görsel olarak büyük olmalılar. Bootstrap `h1`…`h6` için hem etiket hem de **aynı adlı sınıf** tanımlar, tam bu ayrımı yapabilmek için.

> Ekran okuyucu başlık seviyelerinden sayfanın iskeletini çıkarır. `h1` tekrarlamak o iskeleti bozar. `h2.h1` doğru iskelet + doğru görünüm demek.

**`<time datetime="2026-01-24">`** — makine okunur tarih. Arama motorları ve okuyucu uygulamaları bunu kullanıyor. Görünen metin Türkçe, `datetime` özniteliği ISO 8601.

**`.ToUpper(CultureInfo("tr-TR"))`** — büyük harf çevirimi, Türkçe kurallarıyla. Niye kültür belirtiyoruz? Çünkü varsayılan çevirim `i` harfini `I` yapar, Türkçe'de ise `İ` olmalı:

```
"Nisan".ToUpper()                      →  "NISAN"   ✗  (I'sız)
"Nisan".ToUpper(new CultureInfo("tr"))  →  "NİSAN"   ✓
```

Modül 2'deki `ToLowerInvariant` tuzağının aynadaki görüntüsü. Orada **invariant** istiyorduk (slug için), burada **Türkçe** istiyoruz (gösterim için). Kural: *makine için invariant, insan için kültür.*

> Program.cs'te `tr-TR` kültürünü varsayılan yaptığımız için `.ToUpper()` da doğru çalışır. Yine de burada açıkça yazıyoruz — bu satırın niye var olduğu derste sorulacak bir şey.

---

## ⌨️ Adım 3 — Ana sayfa görünümü

`Views/Anasayfa/Index.cshtml`:

```html
@model List<KisiselBlog.Models.Yazi>
@{
    ViewData["Title"] = "";          // ana sayfada başlık eklenmesin

    int sayfa = ViewBag.Sayfa;
    int toplamSayfa = ViewBag.ToplamSayfa;
}

@* ⭐ visually-hidden: ekranda görünmez, ekran okuyucu okur.
     Sayfanın bir h1'i olmalı ama tasarımda yeri yok — liste
     sayfasının konusu zaten belli. display:none YAZMAYIN;
     o, ekran okuyucudan da gizler. *@
<h1 class="visually-hidden">Yazılar</h1>

@if (!Model.Any())
{
    <div class="text-center py-5">
        <i class="bi bi-journal-text display-4" style="color: var(--soluk);"></i>
        <p class="mt-3 mb-0" style="color: var(--soluk);">
            Henüz yayınlanmış bir yazı yok.
        </p>
    </div>
}
else
{
    @foreach (var yazi in Model)
    {
        <partial name="_YaziKarti" model="yazi" />
    }

    @* ─── SAYFALAMA ─── *@
    @if (toplamSayfa > 1)
    {
        <nav class="d-flex justify-content-between align-items-center pt-2"
             aria-label="Sayfalar">

            @* Yeni yazılar = önceki sayfa *@
            @if (sayfa > 1)
            {
                <a asp-action="Index" asp-route-sayfa="@(sayfa - 1)"
                   class="btn btn-outline-secondary btn-sm">
                    <span aria-hidden="true">&larr;</span> Yeni yazılar
                </a>
            }
            else
            {
                <span></span>
            }

            <span class="small" style="color: var(--soluk);">
                @sayfa / @toplamSayfa
            </span>

            @* Eski yazılar = sonraki sayfa *@
            @if (sayfa < toplamSayfa)
            {
                <a asp-action="Index" asp-route-sayfa="@(sayfa + 1)"
                   class="btn btn-outline-secondary btn-sm">
                    Eski yazılar <span aria-hidden="true">&rarr;</span>
                </a>
            }
            else
            {
                <span></span>
            }

        </nav>
    }
}
```

### 📖 Boş `<span></span>` ne işe yarıyor?

`justify-content-between` kendisine verilen öğeleri iki uca ve araya dağıtır. İlk sayfadayız ve "Yeni yazılar" butonu yoksa, iki öğe kalır ve sayaç sola kayar.

Boş `<span>` yer tutucu olarak duruyor; sayaç ortada kalıyor. Alternatifi `justify-content-end` ile `ms-auto` kombinasyonlarıyla uğraşmaktır — bu daha kısa.

### 📖 Sayfalama yönü: "Yeni" sola, "Eski" sağa

Blogda sıralama **yeniden eskiye**. Yani:

```
Sayfa 1 = en yeni yazılar
Sayfa 2 = daha eski yazılar
```

Bu yüzden `sayfa - 1` butonuna "Eski yazılar" yazmak **yanlış** olur; o yön daha yeni yazılara gider. Öğrencilerin en sık karıştırdığı yer burası.

> `<` ve `>` yerine `&larr;` `&rarr;` kullandık; `aria-hidden="true"` ile ekran okuyucudan sakladık — okla birlikte metin zaten var, ikisini de okumak gereksiz tekrar olurdu.

---

## ⌨️ Adım 4 — Yazı detay controller'ı

`Controllers/YaziController.cs`:

```csharp
using Microsoft.AspNetCore.Mvc;
using Microsoft.EntityFrameworkCore;
using KisiselBlog.Data;
using KisiselBlog.Models;

namespace KisiselBlog.Controllers;

[Route("yazi")]
public class YaziController : GenelController
{
    private readonly BlogDbContext _db;

    public YaziController(BlogDbContext db)
    {
        _db = db;
    }

    // ══════════════════════════════════════════════════════
    //  YAZI DETAYI
    //  GET: /yazi/merhaba-dunya
    // ══════════════════════════════════════════════════════
    [HttpGet("{slug}")]
    public async Task<IActionResult> Detay(string slug)
    {
        if (string.IsNullOrWhiteSpace(slug))
            return NotFound();

        var yazi = await _db.Yazilar
            .AsNoTracking()
            .Include(y => y.Etiketler)
            .FirstOrDefaultAsync(y => y.Slug == slug
                                   && y.Durum == YaziDurum.Yayinda);

        // ⚠️ Taslak yazının adresini bilen biri de 404 almalı.
        //    Yukarıdaki Durum koşulu bunu sağlıyor.
        if (yazi == null)
            return NotFound();

        // ── Okunma sayacı ───────────────────────────────────
        //
        // ⭐ ExecuteUpdateAsync: nesneyi çekmeden tek sorguda artır.
        //    AsNoTracking ile çektiğimiz için zaten güncelleyemezdik.
        //
        // ⚠️ Bu basit bir sayaç. Aynı kişi sayfayı 10 kez
        //    yenilerse 10 sayılır. Gerçek analitik için
        //    oturum/IP takibi gerekir — bu dersin kapsamı dışında.
        await _db.Yazilar
            .Where(y => y.YaziId == yazi.YaziId)
            .ExecuteUpdateAsync(s => s.SetProperty(
                y => y.OkunmaSayisi, y => y.OkunmaSayisi + 1));

        // ── Önceki / sonraki yazı ───────────────────────────
        //
        // Okuyucu yazıyı bitirince ne yapsın? Gezinmesi kolay olsun.
        ViewBag.OncekiYazi = await _db.Yazilar
            .AsNoTracking()
            .Where(y => y.Durum == YaziDurum.Yayinda
                     && y.YayinTarihi < yazi.YayinTarihi)
            .OrderByDescending(y => y.YayinTarihi)
            .Select(y => new { y.Baslik, y.Slug })
            .FirstOrDefaultAsync();

        ViewBag.SonrakiYazi = await _db.Yazilar
            .AsNoTracking()
            .Where(y => y.Durum == YaziDurum.Yayinda
                     && y.YayinTarihi > yazi.YayinTarihi)
            .OrderBy(y => y.YayinTarihi)
            .Select(y => new { y.Baslik, y.Slug })
            .FirstOrDefaultAsync();

        // ── Sayfa başlığı ve açıklaması ─────────────────────
        ViewData["Title"] = yazi.Baslik;
        ViewData["Aciklama"] = MarkdownYardimcisi.DuzMetin(
            yazi.Ozet ?? yazi.Icerik, 160);

        return View(yazi);
    }

    // ══════════════════════════════════════════════════════
    //  ETİKETE GÖRE YAZILAR
    //  GET: /etiket/veritabani
    //
    //  ⭐ Baştaki "/" ile controller'ın "yazi" önekini deliyoruz.
    // ══════════════════════════════════════════════════════
    [HttpGet("/etiket/{slug}")]
    public async Task<IActionResult> Etiket(string slug)
    {
        var etiket = await _db.Etiketler
            .AsNoTracking()
            .FirstOrDefaultAsync(e => e.Slug == slug);

        if (etiket == null)
            return NotFound();

        var yazilar = await _db.Yazilar
            .AsNoTracking()
            .Where(y => y.Durum == YaziDurum.Yayinda
                     && y.Etiketler.Any(e => e.EtiketId == etiket.EtiketId))
            .Include(y => y.Etiketler)
            .OrderByDescending(y => y.YayinTarihi)
            .ToListAsync();

        ViewData["Title"] = etiket.Ad;
        ViewData["Aciklama"] = $"\"{etiket.Ad}\" etiketli yazılar.";
        ViewBag.EtiketAd = etiket.Ad;

        return View(yazilar);
    }

    // ══════════════════════════════════════════════════════
    //  TÜM ETİKETLER
    //  GET: /etiketler
    // ══════════════════════════════════════════════════════
    [HttpGet("/etiketler")]
    public async Task<IActionResult> Etiketler()
    {
        // ⭐ Sadece yayında yazısı OLAN etiketleri göster.
        //    Boş etiket göstermek okuyucuyu yanıltır.
        var etiketler = await _db.Etiketler
            .AsNoTracking()
            .Select(e => new
            {
                e.Ad,
                e.Slug,
                YaziSayisi = e.Yazilar.Count(y => y.Durum == YaziDurum.Yayinda
                                               && y.AktifMi)
            })
            .Where(x => x.YaziSayisi > 0)
            .OrderByDescending(x => x.YaziSayisi)
            .ThenBy(x => x.Ad)
            .ToListAsync();

        ViewData["Title"] = "Etiketler";
        ViewBag.Etiketler = etiketler;

        return View();
    }
}
```

### ⚠️ `y.AktifMi` neden elle yazıldı?

```csharp
YaziSayisi = e.Yazilar.Count(y => y.Durum == YaziDurum.Yayinda && y.AktifMi)
```

**Sor:** *"Query filter zaten `AktifMi`'yi hallediyordu. Neden tekrar yazdık?"*

Çünkü bu, `Select` içindeki bir **alt sorgu**. Query filter ana sorguya (`Etiketler`) uygulanır; alt sorgudaki koleksiyon erişimine her zaman uygulanmayabilir.

> **Kural:** Alt sorgu yazdığınızda query filter'a güvenmeyin, koşulu elle yazın. Konsoldaki SQL'e bakıp doğrulayın.

---

## ⌨️ Adım 5 — Markdown gövdesinin biçimi

Yazı gövdesi Markdig'den HTML olarak geliyor: `<h2>`, `<p>`, `<ul>`, `<pre><code>`, `<blockquote>`, `<table>`, `<img>`. Bu etiketlerin **hiçbirinde** Bootstrap sınıfı yok — Markdig onları koymuyor.

İki seçenek var:

| Seçenek | Nasıl | Sonuç |
|---|---|---|
| Her etiketi elle sınıflamak | Markdig'in çıktısını işlemek | Karmaşık, kırılgan |
| **Bir sarmalayıcıya CSS yazmak** | `.yazi-govde h2 { ... }` | ⭐ **Basit** |

İkincisini seçiyoruz. `site.css`'in **sonuna** ekleyin:

```css
/* ═══ Markdown gövdesi ═══
   Markdig'in ürettiği etiketlerde Bootstrap sınıfı yok.
   Sarmalayıcı üzerinden biçimliyoruz. */

.yazi-govde { font-size: 1.125rem; line-height: 1.75; }

.yazi-govde > * + * { margin-top: 1.25rem; }   /* dikey ritim */

.yazi-govde h2 { font-size: 1.6rem; margin-top: 2.5rem; }
.yazi-govde h3 { font-size: 1.25rem; margin-top: 2rem; }

.yazi-govde img {
  max-width: 100%; height: auto;
  border-radius: .5rem; margin-block: 1.75rem;
}

.yazi-govde pre {
  background: var(--yan-zemin);
  border: 1px solid var(--bs-border-color);
  border-radius: .5rem; padding: 1rem;
  overflow-x: auto; font-size: .9rem;
}

.yazi-govde code {
  background: var(--yan-zemin);
  padding: .1rem .35rem; border-radius: .25rem;
  font-size: .9em; color: #9c3d1e;
}
.yazi-govde pre code { background: none; padding: 0; color: inherit; }

.yazi-govde blockquote {
  border-left: 3px solid var(--sari);
  padding-left: 1.1rem; margin-left: 0;
  color: var(--soluk); font-style: italic;
}

.yazi-govde table {
  width: 100%; border-collapse: collapse; font-size: .95rem;
}
.yazi-govde th, .yazi-govde td {
  border-bottom: 1px solid var(--bs-border-color);
  padding: .55rem .7rem; text-align: left;
}
.yazi-govde th { font-weight: 700; color: var(--bs-heading-color); }
```

### 📖 `.yazi-govde > * + *` ne demek?

Bu seçici **"lobotomized owl"** diye bilinir — `* + *` iki göz ve bir gaga gibi görünüyor diye.

Okunuşu: *"`.yazi-govde`'nin doğrudan çocuklarından, kendisinden önce bir kardeşi olanlar."*

Yani **ilki hariç tüm çocuklar**. Hepsinin üstüne boşluk koyuyor:

```
┌─ h2        ← ilk çocuk, margin-top YOK
│
│  (1.25rem)
├─ p         ← öncesinde kardeş var, margin-top VAR
│
│  (1.25rem)
├─ p
│
│  (1.25rem)
└─ ul
```

**Niye `.yazi-govde p { margin-top: 1.25rem }` değil?** Çünkü o zaman her etiket için ayrı kural yazmanız gerekir: `p`, `ul`, `ol`, `pre`, `blockquote`, `table`, `img`… Ve Markdig yeni bir etiket üretirse onu unutursunuz. Bu seçici **ne gelirse** araya boşluk koyar.

Bir de şu problemi çözüyor: iki komşu öğenin `margin-bottom` ve `margin-top`'u birleşince (**margin collapse**) boşluk tahmin edilemez hâle gelir. Sadece `margin-top` kullanmak bunu ortadan kaldırıyor.

---

## ⌨️ Adım 6 — Yazı detay sayfası

`Views/Yazi/Detay.cshtml`:

```html
@model KisiselBlog.Models.Yazi
@using KisiselBlog.Data
@{
    ViewData["Title"] = Model.Baslik;
    var onceki = ViewBag.OncekiYazi;
    var sonraki = ViewBag.SonrakiYazi;
    var tr = new System.Globalization.CultureInfo("tr-TR");
}

<article>

    @* ─── BAŞLIK BLOĞU ─── *@
    <header class="mb-4">

        @if (Model.YayinTarihi.HasValue)
        {
            <p class="yazi-tarih mb-2">
                <time datetime="@Model.YayinTarihi.Value.ToString("yyyy-MM-dd")">
                    @Model.YayinTarihi.Value.ToString("d MMMM yyyy").ToUpper(tr)
                </time>
            </p>
        }

        <h1 class="mb-3">@Model.Baslik</h1>

        <div class="d-flex flex-wrap align-items-center gap-3 small"
             style="color: var(--soluk);">

            <span>
                <i class="bi bi-clock" aria-hidden="true"></i>
                @Model.OkumaSuresi dk okuma
            </span>

            <span>
                <i class="bi bi-eye" aria-hidden="true"></i>
                @Model.OkunmaSayisi görüntülenme
            </span>

            @if (Model.Etiketler.Any())
            {
                <span class="d-flex flex-wrap">
                    @foreach (var etiket in Model.Etiketler.OrderBy(e => e.Ad))
                    {
                        <a href="/etiket/@etiket.Slug" class="etiket-link">#@etiket.Ad</a>
                    }
                </span>
            }
        </div>

    </header>

    @* ─── KAPAK GÖRSELİ (Modül 10'da dolacak) ─── *@
    @if (!string.IsNullOrEmpty(Model.KapakGorsel))
    {
        <img src="@Model.KapakGorsel" alt=""
             class="img-fluid rounded mb-4" />
    }

    @* ─── ⭐ GÖVDE ─── *@
    @*
        Html.Raw: Markdig'in ürettiği HTML'i olduğu gibi bas.

        Güvenli, çünkü:
          1. İçeriği sadece panel sahibi yazıyor
          2. Markdig pipeline'ında .DisableHtml() var —
             ham <script> etiketleri geçmiyor

        ⚠️ Kullanıcıdan gelen ham metni ASLA böyle basmayın.
           Modül 9'da iletişim formunda bunun karşıtını göreceğiz.
    *@
    <div class="yazi-govde">
        @Html.Raw(MarkdownYardimcisi.Cevir(Model.Icerik))
    </div>

</article>


@* ─── ÖNCEKİ / SONRAKİ ─── *@
@if (onceki != null || sonraki != null)
{
    <nav class="row g-3 mt-5 pt-4 border-top" aria-label="Diğer yazılar">

        <div class="col-6">
            @if (sonraki != null)
            {
                <p class="yan-baslik mb-1">
                    <span aria-hidden="true">&larr;</span> Sonraki
                </p>
                <a href="/yazi/@sonraki.Slug" class="fw-bold small">@sonraki.Baslik</a>
            }
        </div>

        <div class="col-6 text-end">
            @if (onceki != null)
            {
                <p class="yan-baslik mb-1">
                    Önceki <span aria-hidden="true">&rarr;</span>
                </p>
                <a href="/yazi/@onceki.Slug" class="fw-bold small">@onceki.Baslik</a>
            }
        </div>

    </nav>
}

<p class="mt-5">
    <a asp-controller="Anasayfa" asp-action="Index" class="devam">
        <span aria-hidden="true">&larr;</span> Tüm yazılar
    </a>
</p>
```

### 📖 Burada görünür `h1` var, listede gizli

Her sayfanın **tam bir** `h1`'i olmalı — ekran okuyucular sayfanın konusunu oradan anlıyor.

| Sayfa | `h1` | Niye |
|---|---|---|
| Yazı detayı | Yazının başlığı | Sayfanın konusu o yazı |
| Ana sayfa | `visually-hidden` "Yazılar" | Konu belli ama tasarımda başlığa yer yok |
| Etiket sayfası | `#etiketadı` | Sayfanın konusu o etiket |
| Yan sütun | **yok** (`p.h5`) | Her sayfada görünüyor; `h1` olsaydı çakışırdı |

> **Derste yapın:** F12 → Elements → `Ctrl+F` → `h1` yazın. Her sayfada kaç tane bulduğunuzu sayın. Bir tanesi fazlaysa hangisi gereksiz?

### ⚠️ `img-fluid` neden şart?

Kapak görseli 2000px genişliğinde yüklenebilir. `img-fluid` sınıfı `max-width: 100%; height: auto;` verir — görsel kabına sığar, oranı bozulmaz.

Olmazsa görsel yan sütunun ve sayfanın dışına taşar, yatay kaydırma çubuğu çıkar. Mobilde felaket görünür.

---

## ⌨️ Adım 7 — Etiket sayfaları

`Views/Yazi/Etiket.cshtml`:

```html
@model List<KisiselBlog.Models.Yazi>
@{
    ViewData["Title"] = ViewBag.EtiketAd;
}

<header class="mb-5">
    <p class="yan-baslik mb-1">Etiket</p>
    <h1 class="mb-2">#@ViewBag.EtiketAd</h1>
    <p class="mb-0" style="color: var(--soluk);">
        @Model.Count yazı
    </p>
</header>

@if (!Model.Any())
{
    <p style="color: var(--soluk);">Bu etiketle yayınlanmış yazı yok.</p>
}
else
{
    @foreach (var yazi in Model)
    {
        <partial name="_YaziKarti" model="yazi" />
    }
}

<p class="mt-4">
    <a asp-action="Etiketler" class="devam">Tüm etiketler</a>
</p>
```

`Views/Yazi/Etiketler.cshtml`:

```html
@{
    ViewData["Title"] = "Etiketler";
    var etiketler = ViewBag.Etiketler as IEnumerable<dynamic>;
}

<header class="mb-5">
    <h1 class="mb-2">Etiketler</h1>
    <p class="mb-0" style="color: var(--soluk);">
        Yazılar konularına göre.
    </p>
</header>

@if (etiketler == null || !etiketler.Any())
{
    <p style="color: var(--soluk);">Henüz etiket yok.</p>
}
else
{
    <div class="d-flex flex-wrap gap-2">
        @foreach (var e in etiketler)
        {
            <a href="/etiket/@e.Slug"
               class="btn btn-outline-secondary btn-sm rounded-pill">
                #@e.Ad
                <span class="badge text-bg-secondary ms-1">@e.YaziSayisi</span>
            </a>
        }
    </div>
}
```

> `rounded-pill` tam yuvarlak kenar verir. Etiket bulutlarında alışılmış bir görünüm. `badge text-bg-secondary` ise Bootstrap 5.3'ün yeni rozet sınıfı — zemin ve yazı rengini birlikte ayarlıyor (eski `badge bg-secondary`'de yazı rengini ayrıca vermek gerekiyordu).

---

## ⌨️ Adım 8 — 404 sayfası

Kişisel sitede 404 bile düzgün görünmeli.

`Program.cs`'e ekleyin — `UseRouting()`'den **önce**:

```csharp
// ⭐ 404 ve diğer durum kodlarını kendi sayfamıza yönlendir
app.UseStatusCodePagesWithReExecute("/hata/{0}");
```

`Controllers/HataController.cs`:

```csharp
using Microsoft.AspNetCore.Mvc;

namespace KisiselBlog.Controllers;

public class HataController : GenelController
{
    [Route("/hata/{kod:int}")]
    public IActionResult Durum(int kod)
    {
        ViewData["Title"] = kod == 404 ? "Sayfa bulunamadı" : "Bir şeyler ters gitti";
        ViewBag.Kod = kod;
        return View("Durum");
    }

    [Route("/hata")]
    public IActionResult Genel()
    {
        ViewData["Title"] = "Bir şeyler ters gitti";
        ViewBag.Kod = 500;
        return View("Durum");
    }
}
```

`Views/Hata/Durum.cshtml`:

```html
@{
    int kod = ViewBag.Kod;
}

<div class="text-center py-5">

    <p class="display-1 fw-bold mb-0" style="color: var(--bs-border-color);">
        @kod
    </p>

    <h1 class="mt-3">
        @(kod == 404 ? "Bu sayfa yok" : "Bir şeyler ters gitti")
    </h1>

    <p class="mx-auto mt-3" style="color: var(--soluk); max-width: 28rem;">
        @if (kod == 404)
        {
            <text>Aradığınız sayfa taşınmış veya hiç var olmamış olabilir.</text>
        }
        else
        {
            <text>Beklenmedik bir hata oluştu. Kısa süre sonra tekrar deneyin.</text>
        }
    </p>

    <p class="mt-4">
        <a asp-controller="Anasayfa" asp-action="Index"
           class="btn btn-sari rounded-0 px-4">
            Ana sayfaya dön
        </a>
    </p>

</div>
```

> `display-1` Bootstrap'in en büyük başlık boyutu (~5rem). Rakamı kenarlık rengiyle yazdık — dekoratif, okunması gerekmiyor, asıl mesaj altındaki `h1`'de.

---

## ▶️ Çalıştır ve gör

```bash
dotnet run
```

| # | Test | Beklenen |
|---|---|---|
| 1 | Panelden 3 yazı yayınla, 1 taslak bırak | — |
| 2 | `/` | **3 yazı** görünüyor, taslak yok |
| 3 | Tarihler | "24 OCAK 2026" — Türkçe, büyük harf, aralıklı |
| 4 | Başlıklar | Montserrat, kalın, **koyu** (sarı değil) |
| 5 | Başlığın üstüne gel | Sarı-kahveye dönüyor |
| 6 | "DEVAMINI OKU" | Sarı-kahve, büyük harf |
| 7 | Yazı başlığına tıkla | `/yazi/slug` açılıyor |
| 8 | Taslağın slug'ını adres çubuğuna yaz | **404** |
| 9 | Sayfayı 3 kez yenile | Görüntülenme sayısı 3 artıyor |
| 10 | Markdown gövdesi | `##` başlık oldu, `` ` `` kod oldu, `>` alıntı oldu |
| 11 | Kod bloğu | Gri zemin, yuvarlak köşe, taşarsa yatay kaydırma |
| 12 | Alıntı | Solunda sarı çizgi |
| 13 | Yazı sonu | Önceki/sonraki bağlantıları |
| 14 | 11 yazı yayınla | Altta "1 / 2" ve "Eski yazılar →" |
| 15 | `/?sayfa=2` | Kalan yazı + "← Yeni yazılar" |
| 16 | `/?sayfa=99` | Boş liste, çökme yok |
| 17 | `/?sayfa=-5` | 1. sayfa (controller düzeltiyor) |
| 18 | Etikete tıkla | `/etiket/slug` — o etiketli yazılar |
| 19 | `/etiketler` | Yuvarlak etiket butonları + sayılar |
| 20 | `/olmayan-bir-adres` | Düzgün 404, yan sütun duruyor |
| 21 | Pencereyi daralt | Yazılar tek sütun, yan sütun hamburgerde |
| 22 | Pencereyi genişlet | Metin `46rem`'de duruyor, yayılmıyor |

### Yerleşimi sınıfla karşılaştırın

Referans siteyi (`harunaksaya.blog`) ve kendi sayfanızı yan yana açın. Sırayla bakın:

1. **Tarih–başlık–özet–bağlantı** sırası aynı mı?
2. Başlık boyutu yeterince büyük mü? (`h2.h1` ~2.5rem)
3. Yazılar arası boşluk yeterli mi? (`mb-5 pb-4` ~3rem + ayırıcı çizgi)
4. Yan sütun kaydırmada yerinde kalıyor mu?

Farklar olacak — birebir kopya hedefimiz değil. Amaç, aynı **yerleşim fikrini** Bootstrap'in ızgarasıyla kurabilmek.

---

## ⚠️ Sık yapılan hatalar

| Hata | Sebep | Çözüm |
|---|---|---|
| Taslaklar sitede görünüyor | `Where(y => y.Durum == YaziDurum.Yayinda)` yazılmamış | Her genel sorguya ekleyin |
| Ay adı İngilizce | `CultureInfo` ayarlanmamış | `Program.cs`'teki `tr-TR` bloğu (Modül 3) |
| "NISAN" yazıyor, "NİSAN" değil | `.ToUpper()` kültürsüz çağrılmış | `.ToUpper(new CultureInfo("tr-TR"))` |
| Markdown gövdesi biçimsiz | `.yazi-govde` CSS'i eklenmemiş | Adım 5 |
| Gövde `<h2>`leri listedekiler gibi kocaman | `.yazi-govde h2` kuralı yok | Adım 5 |
| Kapak görseli sayfayı taşırıyor | `img-fluid` yok | Ekleyin |
| `Html.Raw` HTML'i metin olarak basıyor | `@Html.Raw(...)` yerine `@(...)` yazılmış | `Html.Raw` şart |
| Sayfalama butonları ters | "Eski/Yeni" karıştırılmış | Sayfa 1 = en yeni |
| `/?sayfa=0` çöküyor | `if (sayfa < 1) sayfa = 1;` yok | Ekleyin |
| 404'te yan sütun kayıp | `HataController` `GenelController`'dan türemiyor | Türetin |
| `/yazi/` (slug boş) 500 veriyor | Rota yine de eşleşiyor | `string.IsNullOrWhiteSpace(slug)` kontrolü |
| Görüntülenme sayısı artmıyor | `ExecuteUpdateAsync` `await`'siz | `await` ekleyin |
| `_YaziKarti` bulunamıyor | `Views/Shared/` içinde değil | Taşıyın |
| Etiket sayfası boş geliyor | `Include(y => y.Etiketler)` yok | Ekleyin |

---

## 📝 Özet

| Dosya | İşlevi |
|---|---|
| `Controllers/AnasayfaController.cs` | Liste + sayfalama |
| `Controllers/YaziController.cs` | ⭐ Slug rotası, sayaç, etiketler |
| `Controllers/HataController.cs` | 404 / 500 |
| `Views/Shared/_YaziKarti.cshtml` | ⭐ Tek yerde duran yazı kartı |
| `Views/Anasayfa/Index.cshtml` | Ana sayfa |
| `Views/Yazi/Detay.cshtml` | ⭐ Markdown gövdesi |
| `Views/Yazi/Etiket.cshtml` | Etikete göre liste |
| `Views/Yazi/Etiketler.cshtml` | Etiket bulutu |
| `Views/Hata/Durum.cshtml` | Hata sayfası |
| `wwwroot/css/site.css` | `.yazi-govde` kuralları eklendi |

**Öğrenilen kavramlar:**

- Öznitelik tabanlı rota ve `{slug}` yer tutucusu
- Rota kısıtları (`:int`, `:min(1)`)
- `[HttpGet("/etiket/{slug}")]` ile controller önekini delmek
- Genel sorgularda `Durum == Yayinda` disiplini
- `ExecuteUpdateAsync` ile tek sorguda sayaç artırma
- `h2.h1` — doğru başlık seviyesi + istenen görünüm
- `<time datetime>` ve makine okunur tarih
- `.ToUpper(CultureInfo)` — insan için kültür, makine için invariant
- `> * + *` seçicisiyle dikey ritim
- `Html.Raw`'ın güvenli olduğu bağlam
- `UseStatusCodePagesWithReExecute`

---

## ✏️ Alıştırmalar

1. **Arşiv sayfası.** `/arsiv` adresinde tüm yazıları yıla göre gruplayarak listeleyin. Sadece tarih ve başlık, özet yok. İpucu: `GroupBy(y => y.YayinTarihi.Value.Year)` ve veriyi belleğe aldıktan **sonra** gruplayın.

2. **Öne çıkan yazı.** Ana sayfanın en üstündeki yazıyı diğerlerinden büyük gösterin — `display-5` başlık, kapak görseli görünür. İpucu: `@foreach` içinde `Model.IndexOf(yazi) == 0` kontrolü, ya da listeyi `First()` ve `Skip(1)` olarak ikiye ayırmak.

3. **Okuma çubuğu.** Yazı detayında sayfanın üstünde, kaydırdıkça dolan ince bir ilerleme çubuğu ekleyin. Bootstrap'in `progress` bileşenini `position-fixed top-0` ile kullanın; dolguyu `scroll` olayında JavaScript'le güncelleyin.

4. **İlgili yazılar.** Yazı sonuna, aynı etiketleri paylaşan en fazla 3 yazıyı ekleyin. İpucu: `Where(y => y.Etiketler.Any(e => mevcutEtiketIdler.Contains(e.EtiketId)) && y.YaziId != yazi.YaziId)`.

5. **`visually-hidden`'ı sınayın.** Ana sayfadaki gizli `h1`'i `style="display:none"` ile değiştirin. Gözle hiçbir fark yok. Ama tarayıcının erişilebilirlik ağacına bakın (F12 → Elements → Accessibility paneli): başlık kayboldu mu? İkisinin farkını bir cümleyle yazın.

**Düşünme soruları:**

- Okunma sayacı her istekte bir `UPDATE` çalıştırıyor. Günde 50.000 ziyaretçide bu veritabanına ne yapar? Nasıl toplu hâle getirilirdi?
- `_YaziKarti` kısmi görünümü `Model.Etiketler`'e erişiyor. Controller'da `Include(y => y.Etiketler)` yazmayı unutsak ne olur? (İpucu: `AsNoTracking` + lazy loading kapalı.)
- `GorunenOzet` `[NotMapped]`. Bu yüzden `Where(y => y.GorunenOzet.Contains("x"))` yazamıyoruz. Niye?
- `Html.Raw` burada güvenli. Hangi tek değişiklik onu tehlikeli hâle getirirdi?

---

👉 Sıradaki: [`07-sayfalar.md`](07-sayfalar.md) — Dinamik sayfalar ve yakalayıcı rota
