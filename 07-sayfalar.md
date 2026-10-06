# Modül 7 — Dinamik Sayfalar

**Süre:** 1 ders saati

---

## 🎯 Bu derste ne yapacağız

"Hakkımda", "Projelerim" gibi sayfaları **koddan değil panelden** yönetilebilir yapacağız.

**Yeni kavram: kök seviyesinde slug ve rota çakışması.**

---

## 📖 Neden veritabanında?

**Sınıfa sor:** *"Hakkımda sayfasını nasıl yapardınız?"*

İlk akla gelen:

```csharp
public IActionResult Hakkimda() => View();
```

Ve `Views/.../Hakkimda.cshtml` içine metin.

**İkinci soru:** *"Peki yarın 'Projelerim' sayfası isteyince?"*

Yeni controller metodu + yeni view + yeniden derle + yeniden yayınla.

**Üçüncü soru:** *"Hakkımda metnindeki bir yazım hatasını düzeltmek için?"*

Aynı döngü. Kod değişikliği, yayın.

> ⭐ **İlke:** Değişen şey **içerik**se, koda gömülmez. Veritabanına konur ve panelden yönetilir.

---

## 📖 ⚠️ Rota çakışması

Sayfaların adresi kök seviyede olacak:

```
/hakkimda
/projelerim
/kitaplar
```

Ama varsayılan MVC rotası da kök seviyede:

```
{controller=Anasayfa}/{action=Index}/{id?}
```

Yani `/hakkimda` adresi **"Hakkimda adında bir controller"** olarak yorumlanır.

### Sıra belirleyici

ASP.NET Core rotaları **tanımlanma sırasına** göre dener. Çözüm: sayfa rotasını **en sona** koymak.

```csharp
// Program.cs — SIRA ÖNEMLİ

// 1. Öznitelik rotaları ([Route], [HttpGet("/giris")]) — otomatik önce
// 2. Varsayılan controller rotası
app.MapControllerRoute(
    name: "default",
    pattern: "{controller=Anasayfa}/{action=Index}/{id?}");

// 3. ⭐ EN SON: hiçbiri eşleşmezse sayfa slug'ı dene
app.MapControllerRoute(
    name: "sayfa",
    pattern: "{slug}",
    defaults: new { controller = "Sayfa", action = "Goster" });
```

Böylece:

```
/giris        → öznitelik rotası    (HesapController)
/yazi/x       → öznitelik rotası    (YaziController)
/Anasayfa     → varsayılan rota     (AnasayfaController)
/hakkimda     → hiçbiri eşleşmedi   → sayfa rotası ✓
/olmayan-sey  → sayfa rotası        → Sayfa bulunamadı → 404
```

> ⚠️ **Dikkat:** Bu yakalayıcı rota **tek segmentli** her adresi yakalar. Bir sayfa slug'ı `giris` olsaydı asla erişilemezdi — çünkü öznitelik rotası önce eşleşirdi.
>
> Panelde bunu kontrol edeceğiz.

---

## ⌨️ Adım 1: Yasaklı slug kontrolü

Sistemde kullanılan adresleri sayfa slug'ı olarak kullandırmayalım.

`Data/SlugYardimcisi.cs`'e ekleyin:

```csharp
    /// <summary>
    /// Sistem tarafından kullanılan, sayfa slug'ı olamayacak adresler.
    ///
    /// ⚠️ Bu listeye yeni rota eklediğinizde buraya da ekleyin.
    ///    Unutulursa o sayfaya hiç erişilemez — ve sebebi
    ///    bulunması zor bir hata olur.
    /// </summary>
    private static readonly HashSet<string> YasakliSluglar = new(StringComparer.Ordinal)
    {
        "yazi", "etiket", "etiketler", "iletisim", "giris", "cikis",
        "yonetim", "hata", "rss", "arsiv", "ara", "css", "js",
        "lib", "yuklenen", "favicon"
    };

    public static bool SlugYasakliMi(string slug)
    {
        return YasakliSluglar.Contains(slug);
    }
```

> `StringComparer.Ordinal` kullandık — slug'lar zaten küçük harfli ve ASCII. Kültüre duyarlı karşılaştırma gereksiz ve yavaş olurdu.

---

## ⌨️ Adım 2: Genel site controller'ı

`Controllers/SayfaController.cs`:

```csharp
using Microsoft.AspNetCore.Mvc;
using Microsoft.EntityFrameworkCore;
using KisiselBlog.Data;

namespace KisiselBlog.Controllers;

public class SayfaController : GenelController
{
    private readonly BlogDbContext _db;

    public SayfaController(BlogDbContext db)
    {
        _db = db;
    }

    /// <summary>
    /// Kök seviyesindeki sayfalar: /hakkimda, /projelerim ...
    /// Rota Program.cs'te tanımlı (en sondaki yakalayıcı).
    /// </summary>
    public async Task<IActionResult> Goster(string slug)
    {
        if (string.IsNullOrWhiteSpace(slug))
            return NotFound();

        var sayfa = await _db.Sayfalar
            .AsNoTracking()
            .FirstOrDefaultAsync(s => s.Slug == slug);

        if (sayfa == null)
            return NotFound();

        ViewData["Title"] = sayfa.Baslik;
        ViewData["Aciklama"] = MarkdownYardimcisi.DuzMetin(sayfa.Icerik, 160);

        return View(sayfa);
    }
}
```

`Views/Sayfa/Goster.cshtml`:

```html
@model KisiselBlog.Models.Sayfa
@using KisiselBlog.Data
@{
    ViewData["Title"] = Model.Baslik;
}

<article>

    <h1 style="margin-bottom: 3rem;">@Model.Baslik</h1>

    <div class="yazi-govde">
        @Html.Raw(MarkdownYardimcisi.Cevir(Model.Icerik))
    </div>

</article>
```

> Yazı detayıyla neredeyse aynı — ama meta bilgi yok. Sabit sayfada tarih ve okuma süresi anlamsız.

---

## ⌨️ Adım 3: Yönetim controller'ı

`Controllers/SayfaYonetimController.cs`:

```csharp
using Microsoft.AspNetCore.Mvc;
using Microsoft.EntityFrameworkCore;
using KisiselBlog.Data;
using KisiselBlog.Models;

namespace KisiselBlog.Controllers;

[Route("yonetim/sayfa")]
public class SayfaYonetimController : YonetimControllerTabani
{
    private readonly BlogDbContext _db;

    public SayfaYonetimController(BlogDbContext db)
    {
        _db = db;
    }

    // ══════════════════════════════════════════════════════
    //  YARDIMCI: slug üret ve doğrula
    //
    //  Üç kontrol:
    //    1. Boş mu?
    //    2. Yasaklı mı? (sistem adresi)
    //    3. Başka sayfada var mı?
    // ══════════════════════════════════════════════════════
    private string SlugHazirla(string? istenenSlug, string baslik, long haricId = 0)
    {
        // Kullanıcı slug yazdıysa onu temizle, yazmadıysa başlıktan üret
        string temel = !string.IsNullOrWhiteSpace(istenenSlug)
            ? SlugYardimcisi.Uret(istenenSlug)
            : SlugYardimcisi.Uret(baslik);

        if (string.IsNullOrEmpty(temel))
            temel = "sayfa";

        return SlugYardimcisi.BenzersizUret(temel, aday =>
            SlugYardimcisi.SlugYasakliMi(aday)
            || _db.Sayfalar
                  .IgnoreQueryFilters()
                  .Any(s => s.Slug == aday && s.SayfaId != haricId)
            // ⭐ Yazı slug'larıyla da çakışmasın —
            //    ikisi de farklı seviyede ama karışıklık yaratır
            || _db.Yazilar
                  .IgnoreQueryFilters()
                  .Any(y => y.Slug == aday));
    }

    // ══════════════════════════════════════════════════════
    //  LİSTE
    // ══════════════════════════════════════════════════════
    [HttpGet("")]
    public async Task<IActionResult> Index()
    {
        ViewData["Title"] = "Sayfalar";

        var sayfalar = await _db.Sayfalar
            .AsNoTracking()
            .OrderBy(s => s.Sira)
            .ThenBy(s => s.Baslik)
            .ToListAsync();

        return View(sayfalar);
    }

    // ══════════════════════════════════════════════════════
    //  OLUŞTUR
    // ══════════════════════════════════════════════════════
    [HttpGet("olustur")]
    public IActionResult Olustur()
    {
        ViewData["Title"] = "Yeni sayfa";
        return View(new Sayfa());
    }

    [HttpPost("olustur")]
    [ValidateAntiForgeryToken]
    public async Task<IActionResult> Olustur(Sayfa sayfa, string? istenenSlug)
    {
        ViewData["Title"] = "Yeni sayfa";

        if (!ModelState.IsValid)
            return View(sayfa);

        sayfa.Slug = SlugHazirla(istenenSlug, sayfa.Baslik);
        sayfa.CreatedDate = DateTime.Now;
        sayfa.UpdatedDate = null;
        sayfa.AktifMi = true;

        _db.Sayfalar.Add(sayfa);
        await _db.SaveChangesAsync();

        TempData["Basarili"] = $"\"{sayfa.Baslik}\" sayfası oluşturuldu.";
        return RedirectToAction(nameof(Duzenle), new { id = sayfa.SayfaId });
    }

    // ══════════════════════════════════════════════════════
    //  DÜZENLE
    // ══════════════════════════════════════════════════════
    [HttpGet("duzenle/{id:long}")]
    public async Task<IActionResult> Duzenle(long id)
    {
        var sayfa = await _db.Sayfalar.FindAsync(id);

        if (sayfa == null)
            return NotFound();

        ViewData["Title"] = "Sayfayı düzenle";
        return View(sayfa);
    }

    [HttpPost("duzenle/{id:long}")]
    [ValidateAntiForgeryToken]
    public async Task<IActionResult> Duzenle(long id, Sayfa gelen, string? istenenSlug)
    {
        ViewData["Title"] = "Sayfayı düzenle";

        if (!ModelState.IsValid)
            return View(gelen);

        var mevcut = await _db.Sayfalar.FindAsync(id);

        if (mevcut == null)
            return NotFound();

        mevcut.Baslik       = gelen.Baslik;
        mevcut.Icerik       = gelen.Icerik;
        mevcut.MenudeGoster = gelen.MenudeGoster;
        mevcut.Sira         = gelen.Sira;
        mevcut.UpdatedDate  = DateTime.Now;

        // ⭐ Yazılardan FARKLI: sayfa slug'ı değiştirilebilir.
        //    Sayfalar nadiren paylaşılır ve sayıları azdır.
        //    Kullanıcı açıkça yeni bir slug yazarsa güncelliyoruz.
        if (!string.IsNullOrWhiteSpace(istenenSlug))
        {
            string yeni = SlugHazirla(istenenSlug, gelen.Baslik, id);

            if (yeni != mevcut.Slug)
            {
                TempData["Uyari"] = $"Adres değişti: /{mevcut.Slug} → /{yeni}. " +
                                     "Eski bağlantılar artık çalışmayacak.";
                mevcut.Slug = yeni;
            }
        }

        await _db.SaveChangesAsync();

        TempData["Basarili"] = "Sayfa güncellendi.";
        return RedirectToAction(nameof(Duzenle), new { id });
    }

    // ══════════════════════════════════════════════════════
    //  SİL
    // ══════════════════════════════════════════════════════
    [HttpPost("sil/{id:long}")]
    [ValidateAntiForgeryToken]
    public async Task<IActionResult> Sil(long id)
    {
        var sayfa = await _db.Sayfalar.FindAsync(id);

        if (sayfa == null)
            return NotFound();

        sayfa.AktifMi = false;
        sayfa.UpdatedDate = DateTime.Now;
        await _db.SaveChangesAsync();

        TempData["Basarili"] = "Sayfa silindi.";
        return RedirectToAction(nameof(Index));
    }
}
```

---

## ⌨️ Adım 4: Yönetim görünümleri

`Views/Yonetim/SayfaYonetim/Index.cshtml`:

```html
@model List<KisiselBlog.Models.Sayfa>
@{
    ViewData["Title"] = "Sayfalar";
}

@section BaslikEylem {
    <a asp-action="Olustur" class="btn btn-primary">Yeni sayfa</a>
}

@if (!Model.Any())
{
    <div class="text-center py-5 text-secondary">
        <p>Henüz sayfa yok.</p>
        <p style="margin-top: 1rem;">
            <a asp-action="Olustur" class="btn btn-primary">İlk sayfayı oluştur</a>
        </p>
    </div>
}
else
{
    <div class="table-responsive">
    <table class="table table-hover align-middle">
        <thead>
            <tr>
                <th style="width: 4rem;">Sıra</th>
                <th>Başlık</th>
                <th style="width: 8rem;">Menüde</th>
                <th class="text-end" style="width: 12rem;">İşlem</th>
            </tr>
        </thead>
        <tbody>
            @foreach (var s in Model)
            {
                <tr>
                    <td class="text-secondary">@s.Sira</td>

                    <td>
                        <a asp-action="Duzenle" asp-route-id="@s.SayfaId"
                           class="fw-semibold">@s.Baslik</a>
                        <div class="small text-secondary font-monospace">
                            /@s.Slug
                        </div>
                    </td>

                    <td>
                        @if (s.MenudeGoster)
                        {
                            <span class="badge text-bg-success">Evet</span>
                        }
                        else
                        {
                            <span class="badge text-bg-secondary">Hayır</span>
                        }
                    </td>

                    <td class="text-end text-nowrap">
                        <a href="/@s.Slug" target="_blank"
                           class="btn btn-outline-secondary btn-sm">Gör</a>

                        <form asp-action="Sil" asp-route-id="@s.SayfaId"
                              method="post" style="display:inline;"
                              onsubmit="return confirm('Bu sayfa silinecek. Emin misiniz?');">
                            <button type="submit" class="btn btn-outline-danger btn-sm">
                                Sil
                            </button>
                        </form>
                    </td>
                </tr>
            }
        </tbody>
    </table>
    </div>
}
```

`Views/Yonetim/SayfaYonetim/_Form.cshtml`:

```html
@model KisiselBlog.Models.Sayfa
@{
    bool duzenleme = Model.SayfaId > 0;
}

@* ⭐ Bootstrap ızgarası: geniş ekranda 8+4, dar ekranda alt alta. *@
<div class="row g-4">

    <div class="col-lg-8">
        <div class="mb-3">
            <label asp-for="Baslik" class="form-label"></label>
            <input asp-for="Baslik" autofocus
                   style="font-size: 1.3125rem;
                          font-weight: 500;" class="form-control" />
            <span asp-validation-for="Baslik" class="text-danger small"></span>
        </div>

        <div class="mb-3">
            <label for="istenenSlug" class="form-label">Adres</label>

            @* ⭐ input-group ile "/" öneki kutuya yapışık görünüyor *@
            <div class="input-group">
                <span class="input-group-text font-monospace">/</span>
                <input type="text" id="istenenSlug" name="istenenSlug"
                       class="form-control font-monospace"
                       value="@Model.Slug"
                       placeholder="başlıktan otomatik üretilir" />
            </div>
            <div class="form-text">
                @if (duzenleme)
                {
                    <text>
                        ⚠️ Adresi değiştirirseniz eski bağlantılar çalışmaz.
                    </text>
                }
                else
                {
                    <text>
                        Boş bırakırsanız başlıktan üretilir.
                    </text>
                }
            </div>
        </div>

        <div class="mb-3">
            <label asp-for="Icerik" class="form-label"></label>
            <textarea asp-for="Icerik" class="form-control font-monospace" rows="18"></textarea>
            <span asp-validation-for="Icerik" class="text-danger small"></span>
            <div class="form-text">
                Markdown kullanabilirsiniz.
            </div>
        </div>
    </div>

    <aside class="col-lg-4">
      <div class="card position-sticky" style="top: 1.5rem;">
        <div class="card-body">

        <div class="form-check mb-3">
            <input asp-for="MenudeGoster" type="checkbox" class="form-check-input" />
            <label asp-for="MenudeGoster" class="form-check-label">Menüde göster</label>
        </div>

        <div class="mb-3">
            <label asp-for="Sira" class="form-label"></label>
            <input asp-for="Sira" type="number" min="0" max="99" class="form-control" />
            <span asp-validation-for="Sira" class="text-danger small"></span>
            <div class="form-text">Küçük olan solda görünür.</div>
        </div>

        <div style="display:flex; flex-direction:column; gap: .5rem;
                    margin-top: 1.5rem;">
            <button type="submit" class="btn btn-primary">Kaydet</button>

            @if (duzenleme)
            {
                <a href="/@Model.Slug" target="_blank"
                   class="btn btn-outline-secondary" style="text-align:center;">Sitede gör ↗</a>
            }

            <a asp-action="Index" class="btn btn-outline-secondary"
               style="text-align:center;">Listeye dön</a>
        </div>

        </div>
      </div>
    </aside>

</div>
```

`Views/Yonetim/SayfaYonetim/Olustur.cshtml`:

```html
@model KisiselBlog.Models.Sayfa
@{
    ViewData["Title"] = "Yeni sayfa";
}

<form asp-action="Olustur" method="post">
    <partial name="_Form" model="Model" />
</form>

@section Scripts {
    <partial name="_ValidationScriptsPartial" />
}
```

`Views/Yonetim/SayfaYonetim/Duzenle.cshtml`:

```html
@model KisiselBlog.Models.Sayfa
@{
    ViewData["Title"] = "Sayfayı düzenle";
}

<form asp-action="Duzenle" asp-route-id="@Model.SayfaId" method="post">
    <partial name="_Form" model="Model" />
</form>

@section Scripts {
    <partial name="_ValidationScriptsPartial" />
}
```

---

## ⌨️ Adım 5: Program.cs — yakalayıcı rota

```csharp
app.MapControllerRoute(
    name: "default",
    pattern: "{controller=Anasayfa}/{action=Index}/{id?}");

// ⭐ EN SON — hiçbiri eşleşmezse sayfa slug'ı olarak dene
app.MapControllerRoute(
    name: "sayfa",
    pattern: "{slug}",
    defaults: new { controller = "Sayfa", action = "Goster" });

app.Run();
```

---

## 📖 Menü sorgusu: daha temiz bir yol

Modül 3'te layout içinde `@inject BlogDbContext Db` kullanmıştık. Çalışıyor ama ideal değil:

- View'da veri erişimi — katman karışıyor
- Her sayfa yüklemesinde sorgu — önbelleklenemiyor
- Test edilemiyor

**Daha temiz yol: View Component.**

`ViewComponents/MenuViewComponent.cs`:

```csharp
using Microsoft.AspNetCore.Mvc;
using Microsoft.EntityFrameworkCore;
using KisiselBlog.Data;

namespace KisiselBlog.ViewComponents;

/// <summary>
/// Menüdeki sayfaları getirir.
///
/// View Component = mini controller + view.
/// Layout gibi her yerde kullanılan parçalar için uygun.
/// </summary>
public class MenuViewComponent : ViewComponent
{
    private readonly BlogDbContext _db;

    public MenuViewComponent(BlogDbContext db)
    {
        _db = db;
    }

    public async Task<IViewComponentResult> InvokeAsync()
    {
        var sayfalar = await _db.Sayfalar
            .AsNoTracking()
            .Where(s => s.MenudeGoster)
            .OrderBy(s => s.Sira)
            .ThenBy(s => s.Baslik)
            .Select(s => new MenuOgesi { Baslik = s.Baslik, Slug = s.Slug })
            .ToListAsync();

        return View(sayfalar);
    }
}

public class MenuOgesi
{
    public string Baslik { get; set; } = "";
    public string Slug { get; set; } = "";
}
```

`Views/Shared/Components/Menu/Default.cshtml` — ⚠️ klasör yapısı birebir böyle olmalı:

```html
@model List<KisiselBlog.ViewComponents.MenuOgesi>
@{
    var yol = Context.Request.Path.Value?.ToLowerInvariant() ?? "/";
}

<ul class="list-unstyled small mb-0">
    @foreach (var oge in Model)
    {
        <li>
            <a href="/@oge.Slug" class="@(yol == "/" + oge.Slug ? "fw-bold" : "")">
                @oge.Baslik
            </a>
        </li>
    }
    <li>
        <a asp-controller="Iletisim" asp-action="Index"
           class="@(yol == "/iletisim" ? "fw-bold" : "")">İletişim</a>
    </li>
</ul>
```

`_YanSutun.cshtml`'de "Hakkımda" bölümündeki sayfa listesini silip yerine View Component
çağrısı koyun:

```html
<section class="mb-4">
    <p class="yan-baslik">Hakkımda</p>
    <p class="small mb-2">@Site.Hakkimda</p>

    @* ⭐ View Component çağrısı — sorgu artık C# sınıfında *@
    @await Component.InvokeAsync("Menu")
</section>
```

Sonra `_YanSutun.cshtml`'in başındaki `menuSayfalari` sorgusunu kaldırın. `@inject BlogDbContext Db`
satırı kalsın — etiket listesi hâlâ onu kullanıyor.

> **Aktif sayfa nasıl belli oluyor?** `fw-bold` ile. Bootstrap'in "font weight bold"
> yardımcı sınıfı. Eskiden bunun için `.aktif` diye kendi sınıfımızı yazardık; burada
> hazır olanı kullanıyoruz.

> **Sor:** *"İki yol da çalışıyor. Neden View Component daha iyi?"*
> Çünkü sorumluluk ayrıldı: veri erişimi C# sınıfında, görüntüleme view'da. İleride önbellek eklemek isterseniz tek yerde yaparsınız.

---

## ▶️ Test

| # | Test | Beklenen |
|---|---|---|
| 1 | `/hakkimda` | Seed'deki sayfa açılıyor |
| 2 | Yan sütunda "Hakkımda" | Görünüyor, bulunduğunuz sayfa kalın |
| 3 | Panelden yeni sayfa: "Projelerim" | `/projelerim` çalışıyor |
| 4 | Menüde göster = hayır | Menüden kayboldu, adres hâlâ çalışıyor |
| 5 | Sıra değerlerini değiştir | Menü sırası değişti |
| 6 | Slug olarak `yazi` yaz | **Engellendi** → `yazi-2` oldu |
| 7 | Slug olarak `giris` yaz | Engellendi |
| 8 | `/olmayan-sayfa` | 404 |
| 9 | Markdown içerik | İşlenmiş görünüyor |
| 10 | Sayfayı sil | Menüden ve adresten kayboldu |

> **6. ve 7. maddeler kritik.** Yasaklı slug kontrolü olmasaydı `/giris` adresini bir sayfaya verip giriş ekranını erişilemez hâle getirebilirdiniz.

---

## ⚠️ Sık yapılan hatalar

| Hata | Sebep | Çözüm |
|---|---|---|
| `/hakkimda` 404 veriyor | Yakalayıcı rota yok veya yanlış sırada | En sona ekleyin |
| `/giris` sayfa olarak açılıyor | Yasaklı slug kontrolü yok | `SlugYasakliMi` ekleyin |
| Menü boş | `MenudeGoster = false` | Panelden işaretleyin |
| View Component bulunamıyor | Klasör yolu yanlış | `Views/Shared/Components/Menu/Default.cshtml` |
| Sayfa slug'ı yazı slug'ıyla çakışıyor | Kontrol eksik | `SlugHazirla` içinde yazıları da kontrol edin |
| Markdown işlenmiyor | `Html.Raw` yok | Ekleyin |
| Slug değişince eski bağlantı kırık | Beklenen davranış | Kullanıcıyı uyarın (yaptık) |

---

## ✏️ Alıştırmalar

1. Sayfa yönetimini tamamlayın, 10 test senaryosunu geçirin.

2. View Component'e geçin ve layout'u sadeleştirin.

3. **Yönlendirme tablosu ekleyin:** Slug değişince eski adresi yenisine yönlendiren bir `Yonlendirme` tablosu (`EskiSlug`, `YeniSlug`). Böylece bağlantılar kırılmaz.
   İpucu: `SayfaController.Goster` içinde sayfa bulunamazsa önce yönlendirme tablosuna bakın, varsa `RedirectPermanent` döndürün.

4. Menüyü önbelleğe alın. Sayfalar nadiren değişiyor, her istekte sorgulamaya gerek yok.
   İpucu: `[ResponseCache]` veya `IMemoryCache`.

**Düşünme soruları:**
- Yasaklı slug listesine yeni bir rota eklemeyi unutursanız ne olur? Bunu otomatikleştirmenin bir yolu var mı?
- Sayfa slug'ının değiştirilmesine izin verdik ama yazınınkine vermedik. Bu tutarsızlık mı?
- Yakalayıcı rota `{slug}` tek segmentli. `/blog/hakkimda` gibi iki segmentli bir sayfa isteseydik ne yapardık?

---

👉 Sonraki: [`08-etiketler.md`](08-etiketler.md)
