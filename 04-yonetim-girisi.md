# Modül 4 — Yönetim Girişi

**Süre:** 1 ders saati

---

## 🎯 Bu derste ne yapacağız

Giriş sistemi kuracağız. **Ama bu sefer yetki yönü ters.**

---

## 📖 ⭐ Ters yetki: açık site, kapalı panel

Önceki üç projede her sayfa korumalıydı:

```csharp
// GörevTakip, UBYS — her şey kapalı
secenekler.Filters.Add(new AuthorizeFilter(politika));   // global
// ve HesapController'a [AllowAnonymous]
```

Blogda durum **tersine döndü:**

```
┌──────────────────────────────┬───────────────────────────┐
│  AÇIK (herkes)               │  KAPALI (sadece ben)      │
├──────────────────────────────┼───────────────────────────┤
│  /            ana sayfa      │  /yonetim                 │
│  /yazi/slug   yazı           │  /yonetim/yazi            │
│  /etiket/slug etiket         │  /yonetim/sayfa           │
│  /hakkimda    sayfa          │  /yonetim/etiket          │
│  /iletisim    form           │  /yonetim/mesaj           │
│  /giris       giriş formu    │                           │
└──────────────────────────────┴───────────────────────────┘
```

**Sınıfa sor:** *"Hangi yaklaşımı seçelim — global açık mı, global kapalı mı?"*

### İki yol

```csharp
// ── YOL A: Global AÇIK, yönetime [Authorize] ────────
// Her controller açık, korumalı olanlara öznitelik koy
[Authorize]
public class YaziYonetimController : Controller { }
```

```csharp
// ── YOL B: Global KAPALI, genele [AllowAnonymous] ───
// Her controller kapalı, açık olanlara öznitelik koy
[AllowAnonymous]
public class AnasayfaController : Controller { }
```

| | Yol A | Yol B |
|---|---|---|
| Unutursanız ne olur? | ⚠️ Yönetim sayfası **herkese açık** | Genel sayfa kapalı — kullanıcı şikayet eder, fark edersiniz |
| Hangi hata daha kötü? | **Güvenlik açığı** | Rahatsızlık |

> ⭐ **Güvenlik ilkesi: varsayılan en kısıtlayıcı olmalı.** Hata yaptığınızda sonucu "güvenli ama rahatsız" olsun, "rahat ama açık" değil.
>
> **Yol B'yi seçiyoruz.**

### Ama bir sorun var

Yol B'de her genel controller'a `[AllowAnonymous]` yazmak gerekiyor. Beş controller — biri unutulursa site yarım açık kalır.

**Daha iyisi: taban sınıf (base class).**

```csharp
[AllowAnonymous]
public abstract class GenelController : Controller { }

// Artık sadece miras almak yeter
public class AnasayfaController : GenelController { }
public class YaziController : GenelController { }
```

Öznitelik miras alınır. Tek yerde yazıldı, beş yerde geçerli.

Aynısını yönetim için de yapalım — orada ayrıca ortak yol öneki de var:

```csharp
[Authorize]
[Route("yonetim/[controller]")]
public abstract class YonetimControllerTabani : Controller { }
```

---

## ⌨️ Adım 1: Taban controller'lar

`Controllers/GenelController.cs`:

```csharp
using Microsoft.AspNetCore.Authorization;
using Microsoft.AspNetCore.Mvc;

namespace KisiselBlog.Controllers;

/// <summary>
/// Genel sitenin tüm controller'larının tabanı.
///
/// ⭐ [AllowAnonymous] burada yazıldı, miras alan herkeste geçerli.
///    Program.cs'teki global "giriş zorunlu" kuralını deler.
///
/// abstract: bu sınıftan doğrudan nesne üretilmez,
///           sadece miras alınır.
/// </summary>
[AllowAnonymous]
public abstract class GenelController : Controller
{
}
```

`Controllers/YonetimControllerTabani.cs`:

```csharp
using Microsoft.AspNetCore.Authorization;
using Microsoft.AspNetCore.Mvc;

namespace KisiselBlog.Controllers;

/// <summary>
/// Yönetim panelinin tüm controller'larının tabanı.
///
/// İki iş yapıyor:
///   1. [Authorize]  → giriş zorunlu
///   2. [Route]      → ortak yol öneki
///
/// ⭐ [controller] yer tutucusu, sınıf adından "Controller"
///    ekini atıp kalanı koyar:
///       YaziYonetimController  →  /yonetim/yaziyonetim
/// </summary>
[Authorize]
[Route("yonetim/[controller]")]
public abstract class YonetimControllerTabani : Controller
{
}
```

> **Not:** `/yonetim/yaziyonetim` biraz çirkin. Her controller kendi `[Route]`'unu yazarak bunu ezebilir — Modül 5'te `[Route("yonetim/yazi")]` diyeceğiz.

---

## ⌨️ Adım 2: HesapController

`Controllers/HesapController.cs`:

```csharp
using System.Security.Claims;
using Microsoft.AspNetCore.Authentication;
using Microsoft.AspNetCore.Authentication.Cookies;
using Microsoft.AspNetCore.Authorization;
using Microsoft.AspNetCore.Mvc;
using Microsoft.EntityFrameworkCore;
using KisiselBlog.Data;
using KisiselBlog.Models;

namespace KisiselBlog.Controllers;

[AllowAnonymous]
public class HesapController : Controller
{
    private readonly BlogDbContext _db;

    public HesapController(BlogDbContext db)
    {
        _db = db;
    }

    // ══════════════════════════════════════════════════════
    //  GİRİŞ FORMU
    //  GET: /giris
    // ══════════════════════════════════════════════════════
    [HttpGet("/giris")]
    public IActionResult Giris(string? donusUrl = null)
    {
        if (User.Identity?.IsAuthenticated == true)
            return RedirectToAction("Index", "Panel");

        ViewData["Title"] = "Giriş";
        ViewBag.DonusUrl = donusUrl;
        return View();
    }

    // ══════════════════════════════════════════════════════
    //  GİRİŞİ İŞLE
    //  POST: /giris
    // ══════════════════════════════════════════════════════
    [HttpPost("/giris")]
    [ValidateAntiForgeryToken]
    public async Task<IActionResult> Giris(GirisViewModel model, string? donusUrl = null)
    {
        ViewData["Title"] = "Giriş";
        ViewBag.DonusUrl = donusUrl;

        if (!ModelState.IsValid)
            return View(model);

        string hash = SifreYardimcisi.Hashle(model.Sifre);

        // Karşılaştırmayı veritabanında yapıyoruz — hash dolaşmıyor.
        // EF değerleri otomatik parametreler, SQL injection yok.
        Kullanici? kullanici = await _db.Kullanicilar
            .FirstOrDefaultAsync(k => k.KullaniciAdi == model.KullaniciAdi
                                   && k.SifreHash == hash);

        if (kullanici == null)
        {
            // ⚠️ "Kullanıcı yok" ile "şifre yanlış"ı AYIRMA.
            //    Ayırsaydık saldırgan kayıtlı kullanıcı adlarını
            //    tek tek deneyerek öğrenirdi (user enumeration).
            ModelState.AddModelError("", "Kullanıcı adı veya şifre hatalı.");
            return View(model);
        }

        var iddialar = new List<Claim>
        {
            new Claim(ClaimTypes.NameIdentifier, kullanici.KullaniciId.ToString()),
            new Claim(ClaimTypes.Name, kullanici.AdSoyad),
            new Claim("KullaniciAdi", kullanici.KullaniciAdi)
        };

        var kimlik = new ClaimsIdentity(iddialar,
            CookieAuthenticationDefaults.AuthenticationScheme);

        await HttpContext.SignInAsync(
            CookieAuthenticationDefaults.AuthenticationScheme,
            new ClaimsPrincipal(kimlik),
            new AuthenticationProperties
            {
                IsPersistent = model.BeniHatirla,
                AllowRefresh = true
            });

        // ⚠️ Url.IsLocalUrl ŞART — olmadan açık yönlendirme açığı olur.
        //    /giris?donusUrl=https://sahte-site.com
        if (!string.IsNullOrEmpty(donusUrl) && Url.IsLocalUrl(donusUrl))
            return Redirect(donusUrl);

        return RedirectToAction("Index", "Panel");
    }

    // ══════════════════════════════════════════════════════
    //  ÇIKIŞ
    //  POST: /cikis
    //
    //  ⚠️ POST olmalı. GET olsaydı kötü niyetli sitedeki
    //     <img src="...cikis"> etiketi sizi habersizce çıkartırdı.
    // ══════════════════════════════════════════════════════
    [HttpPost("/cikis")]
    [ValidateAntiForgeryToken]
    public async Task<IActionResult> Cikis()
    {
        await HttpContext.SignOutAsync(CookieAuthenticationDefaults.AuthenticationScheme);
        return RedirectToAction("Index", "Anasayfa");
    }
}
```

### 📖 `[HttpGet("/giris")]` — baştaki eğik çizgi

```csharp
[HttpGet("giris")]    →  /Hesap/giris      (controller adı önüne gelir)
[HttpGet("/giris")]   →  /giris            (⭐ kökten başlar)
```

Baştaki `/` yolu **mutlak** yapar. Giriş sayfası `/giris` gibi temiz bir adres hak ediyor.

---

## ⌨️ Adım 3: Giriş sayfası

Bu sayfa **kendi layout'unu kullanmıyor** — ne genel site yan sütunu, ne panel navbar'ı. Ortalanmış, tek başına bir kart.

`Views/Hesap/Giris.cshtml`:

```html
@model KisiselBlog.Models.GirisViewModel
@inject SiteAyarlari Site
@{
    Layout = null;      @* ⭐ ne genel ne yönetim layout'u *@
}

<!DOCTYPE html>
<html lang="tr">
<head>
    <meta charset="utf-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>Giriş — @Site.Baslik</title>
    <meta name="robots" content="noindex, nofollow" />

    <link href="https://cdn.jsdelivr.net/npm/bootstrap@@5.3.3/dist/css/bootstrap.min.css"
          rel="stylesheet">
    <link href="https://cdn.jsdelivr.net/npm/bootstrap-icons@@1.11.3/font/bootstrap-icons.css"
          rel="stylesheet">
</head>
<body class="bg-body-tertiary">

@* ⭐ min-vh-100 + d-flex + align-items-center = dikey ortalama *@
<div class="container min-vh-100 d-flex align-items-center justify-content-center py-5">

    <div style="width: 100%; max-width: 22rem;">

        <div class="text-center mb-4">
            <i class="bi bi-journal-text fs-1 text-secondary"></i>
            <h1 class="h4 mt-2 mb-1">@Site.Baslik</h1>
            <p class="text-secondary small mb-0">Yönetim paneline giriş</p>
        </div>

        <div class="card shadow-sm">
            <div class="card-body p-4">

                <form asp-action="Giris" method="post">

                    <input type="hidden" name="donusUrl" value="@ViewBag.DonusUrl" />

                    @* Alan bazlı olmayan hatalar (örn. "şifre hatalı") *@
                    @if (!ViewData.ModelState.IsValid &&
                         ViewData.ModelState[""]?.Errors.Count > 0)
                    {
                        <div class="alert alert-danger py-2 small">
                            <i class="bi bi-exclamation-triangle me-1"></i>
                            @ViewData.ModelState[""]!.Errors.First().ErrorMessage
                        </div>
                    }

                    <div class="mb-3">
                        <label asp-for="KullaniciAdi" class="form-label"></label>
                        <input asp-for="KullaniciAdi" class="form-control"
                               autocomplete="username" autofocus />
                        <span asp-validation-for="KullaniciAdi" class="text-danger small"></span>
                    </div>

                    <div class="mb-3">
                        <label asp-for="Sifre" class="form-label"></label>
                        <input asp-for="Sifre" type="password" class="form-control"
                               autocomplete="current-password" />
                        <span asp-validation-for="Sifre" class="text-danger small"></span>
                    </div>

                    <div class="form-check mb-3">
                        <input asp-for="BeniHatirla" type="checkbox" class="form-check-input" />
                        <label asp-for="BeniHatirla" class="form-check-label">
                            Beni hatırla
                        </label>
                    </div>

                    <button type="submit" class="btn btn-primary w-100">
                        Giriş yap
                    </button>

                </form>

            </div>
        </div>

        <p class="text-center mt-3 mb-0 small">
            <a asp-controller="Anasayfa" asp-action="Index" class="text-secondary">
                <span aria-hidden="true">&larr;</span> Siteye dön
            </a>
        </p>

        @* ⚠️ CANLIYA ÇIKMADAN ÖNCE SİLİN *@
        <div class="alert alert-warning py-2 small mt-4 mb-0">
            <strong>Ders ortamı:</strong> admin / blog2026
        </div>

    </div>
</div>

<partial name="_ValidationScriptsPartial" />

</body>
</html>
```

### 📖 Üç Bootstrap kalıbı

**Dikey ortalama.** `min-vh-100 d-flex align-items-center justify-content-center` dörtlüsü, içeriği ekranın tam ortasına koyar. Eskiden bunun için `position: absolute; top: 50%; transform: translateY(-50%)` gibi numaralar gerekirdi; flexbox'la üç sınıf yetiyor.

**`form-check`.** Onay kutuları `form-control` değil, `form-check-input` kullanır. Sarmalayıcı `form-check` kutu ile etiketi yan yana hizalar. `form-control` yazarsanız onay kutusu metin kutusu genişliğinde kocaman bir dikdörtgen olur — derste bir kere gösterin, unutulmaz.

**`w-100`.** Butonu kabının tamamına yayar. Giriş formlarında alışılmış: kullanıcı nereye basacağını aramasın.

### ⚠️ Niye `shadow-sm` var da panelde yok?

Gölge, bir öğenin "yüzeyin üstünde durduğunu" söyler. Giriş kartı tek başına, boş bir zeminde duruyor — gölge onu zeminden ayırıyor.

Panelde ise her şey kart; hepsine gölge verseniz hiçbiri öne çıkmaz, sadece gürültü olur.

---

## ⌨️ Adım 4: Program.cs

Modül 3'te yazdığımız dosyanın **tam hâli** — eklenen satırlar ⭐ ile işaretli.

```csharp
using System.Globalization;
using Microsoft.AspNetCore.Authentication.Cookies;      // ⭐ yeni
using Microsoft.AspNetCore.Authorization;               // ⭐ yeni
using Microsoft.AspNetCore.Mvc.Authorization;           // ⭐ yeni
using Microsoft.EntityFrameworkCore;
using KisiselBlog.Data;
using KisiselBlog.Models;

var builder = WebApplication.CreateBuilder(args);

// ── Türkçe kültür (Modül 3) ─────────────────────────────────
var kultur = new CultureInfo("tr-TR");
CultureInfo.DefaultThreadCurrentCulture = kultur;
CultureInfo.DefaultThreadCurrentUICulture = kultur;

// ── Site ayarları (Modül 3) ─────────────────────────────────
builder.Services.AddSingleton(
    builder.Configuration.GetSection("Site").Get<SiteAyarlari>()
    ?? new SiteAyarlari());

// ── MVC + global "giriş zorunlu" ⭐ ─────────────────────────
//
// ⭐ Varsayılan KAPALI. Genel sayfalar GenelController'dan
//    miras alarak [AllowAnonymous] kazanıyor.
builder.Services.AddControllersWithViews(secenekler =>
{
    var politika = new AuthorizationPolicyBuilder()
        .RequireAuthenticatedUser()
        .Build();

    secenekler.Filters.Add(new AuthorizeFilter(politika));
})
.AddRazorOptions(secenekler =>
{
    // Yönetim view'ları Views/Yonetim/ altında
    secenekler.ViewLocationFormats.Add("/Views/Yonetim/{1}/{0}.cshtml");
    secenekler.ViewLocationFormats.Add("/Views/Yonetim/Shared/{0}.cshtml");
});

// ── Çerez tabanlı kimlik doğrulama ──────────────────────────
builder.Services
    .AddAuthentication(CookieAuthenticationDefaults.AuthenticationScheme)
    .AddCookie(secenekler =>
    {
        secenekler.LoginPath = "/giris";
        secenekler.LogoutPath = "/cikis";
        secenekler.ReturnUrlParameter = "donusUrl";   // controller parametresiyle aynı!
        secenekler.ExpireTimeSpan = TimeSpan.FromDays(14);
        secenekler.SlidingExpiration = true;
        secenekler.Cookie.HttpOnly = true;            // JS erişemez → XSS koruması
        secenekler.Cookie.SameSite = SameSiteMode.Lax;// CSRF koruması
        secenekler.Cookie.Name = "KisiselBlog.Oturum";
    });

// ── EF Core ─────────────────────────────────────────────────
builder.Services.AddDbContext<BlogDbContext>(secenekler =>
{
    secenekler.UseSqlServer(
        builder.Configuration.GetConnectionString("BlogDb"));

    if (builder.Environment.IsDevelopment())
    {
        secenekler.LogTo(Console.WriteLine, LogLevel.Information);
        secenekler.EnableSensitiveDataLogging();
    }
});

var app = builder.Build();

if (!app.Environment.IsDevelopment())
{
    app.UseExceptionHandler("/hata");
    app.UseHsts();
}

app.UseHttpsRedirection();
app.UseStaticFiles();
app.UseRouting();

// ⭐⭐ SIRA KRİTİK
app.UseAuthentication();   // ÖNCE: "sen kimsin?"  (çerezi okur)
app.UseAuthorization();    // SONRA: "girebilir mi?" (filtreyi uygular)

// ── Yönlendirme ─────────────────────────────────────────────
//
// ⭐ Öznitelik tabanlı rotalar ([Route], [HttpGet("/giris")])
//    bu kuraldan ÖNCE değerlendirilir.
app.MapControllerRoute(
    name: "default",
    pattern: "{controller=Anasayfa}/{action=Index}/{id?}");

app.Run();
```

---

## ⌨️ Adım 5: Panel ana sayfası

`Controllers/PanelController.cs` — şimdi gerçek içerikle:

```csharp
using Microsoft.AspNetCore.Mvc;
using Microsoft.EntityFrameworkCore;
using KisiselBlog.Data;
using KisiselBlog.Models;

namespace KisiselBlog.Controllers;

// ⭐ Taban sınıf [Authorize]'ı getiriyor. [Route] burada eziliyor:
//    /yonetim/panel değil, sadece /yonetim olsun istiyoruz.
[Route("yonetim")]
public class PanelController : YonetimControllerTabani
{
    private readonly BlogDbContext _db;

    public PanelController(BlogDbContext db)
    {
        _db = db;
    }

    [HttpGet("")]
    public async Task<IActionResult> Index()
    {
        ViewData["Title"] = "Panel";

        // ⭐ Altı sayıyı TEK sorguda al.
        //    GroupBy(x => 1) numarası: hepsini tek gruba toplayıp
        //    grup üzerinde koşullu sayımlar yapıyoruz.
        var ozet = await _db.Yazilar
            .GroupBy(y => 1)
            .Select(g => new
            {
                Toplam  = g.Count(),
                Yayinda = g.Count(y => y.Durum == YaziDurum.Yayinda),
                Taslak  = g.Count(y => y.Durum == YaziDurum.Taslak),
                Okunma  = g.Sum(y => y.OkunmaSayisi)
            })
            .FirstOrDefaultAsync();

        // ⚠️ Hiç yazı yoksa GroupBy hiç grup üretmez → null döner
        ViewBag.ToplamYazi   = ozet?.Toplam  ?? 0;
        ViewBag.YayindaYazi  = ozet?.Yayinda ?? 0;
        ViewBag.TaslakYazi   = ozet?.Taslak  ?? 0;
        ViewBag.ToplamOkunma = ozet?.Okunma  ?? 0;

        ViewBag.OkunmamisMesaj = await _db.Mesajlar.CountAsync(m => !m.OkunduMu);
        ViewBag.ToplamSayfa    = await _db.Sayfalar.CountAsync();

        // Son düzenlenen 5 yazı
        ViewBag.SonYazilar = await _db.Yazilar
            .AsNoTracking()
            .OrderByDescending(y => y.UpdatedDate ?? y.CreatedDate)
            .Take(5)
            .Select(y => new
            {
                y.YaziId,
                y.Baslik,
                y.Slug,
                y.Durum,
                Tarih = y.UpdatedDate ?? y.CreatedDate
            })
            .ToListAsync();

        return View();
    }
}
```

`Views/Yonetim/Panel/Index.cshtml`:

```html
@using KisiselBlog.Models
@{
    ViewData["Title"] = "Panel";
}

@section BaslikEylem {
    <a asp-controller="YaziYonetim" asp-action="Olustur" class="btn btn-primary">
        <i class="bi bi-plus-lg me-1"></i> Yeni yazı
    </a>
}

@* ═══ SAYAÇ KARTLARI ═══ *@
@*
    row-cols-* : satır başına kaç kart düşeceğini ekran boyuna göre belirler.
    g-3        : kartlar arası boşluk (gutter).
*@
<div class="row row-cols-2 row-cols-md-3 row-cols-xl-6 g-3 mb-5">

    <div class="col">
        <a asp-controller="YaziYonetim" asp-action="Index"
           class="card h-100 text-decoration-none">
            <div class="card-body">
                <p class="text-secondary small mb-1">Toplam yazı</p>
                <p class="h3 mb-0">@ViewBag.ToplamYazi</p>
            </div>
        </a>
    </div>

    <div class="col">
        <a asp-controller="YaziYonetim" asp-action="Index" asp-route-durum="Yayinda"
           class="card h-100 text-decoration-none">
            <div class="card-body">
                <p class="text-secondary small mb-1">Yayında</p>
                <p class="h3 mb-0 text-success">@ViewBag.YayindaYazi</p>
            </div>
        </a>
    </div>

    <div class="col">
        <a asp-controller="YaziYonetim" asp-action="Index" asp-route-durum="Taslak"
           class="card h-100 text-decoration-none">
            <div class="card-body">
                <p class="text-secondary small mb-1">Taslak</p>
                <p class="h3 mb-0">@ViewBag.TaslakYazi</p>
            </div>
        </a>
    </div>

    <div class="col">
        <div class="card h-100">
            <div class="card-body">
                <p class="text-secondary small mb-1">Toplam okunma</p>
                <p class="h3 mb-0">@ViewBag.ToplamOkunma</p>
            </div>
        </div>
    </div>

    <div class="col">
        <a asp-controller="MesajYonetim" asp-action="Index"
           class="card h-100 text-decoration-none">
            <div class="card-body">
                <p class="text-secondary small mb-1">Okunmamış mesaj</p>
                <p class="h3 mb-0 @(ViewBag.OkunmamisMesaj > 0 ? "text-danger" : "")">
                    @ViewBag.OkunmamisMesaj
                </p>
            </div>
        </a>
    </div>

    <div class="col">
        <a asp-controller="SayfaYonetim" asp-action="Index"
           class="card h-100 text-decoration-none">
            <div class="card-body">
                <p class="text-secondary small mb-1">Sayfa</p>
                <p class="h3 mb-0">@ViewBag.ToplamSayfa</p>
            </div>
        </a>
    </div>

</div>

@* ═══ SON DÜZENLENENLER ═══ *@
<h2 class="h5 mb-3">Son düzenlenenler</h2>

@{
    var sonYazilar = ViewBag.SonYazilar as IEnumerable<dynamic>;
}

@if (sonYazilar == null || !sonYazilar.Any())
{
    <div class="card">
        <div class="card-body text-secondary">
            Henüz yazı yok.
            <a asp-controller="YaziYonetim" asp-action="Olustur">İlk yazını oluştur</a>.
        </div>
    </div>
}
else
{
    <div class="card">
        <div class="table-responsive">
            <table class="table table-hover align-middle mb-0">
                <thead>
                    <tr>
                        <th>Başlık</th>
                        <th style="width: 8rem;">Durum</th>
                        <th style="width: 10rem;">Son değişiklik</th>
                    </tr>
                </thead>
                <tbody>
                    @foreach (var y in sonYazilar)
                    {
                        <tr>
                            <td>
                                <a asp-controller="YaziYonetim" asp-action="Duzenle"
                                   asp-route-id="@y.YaziId">@y.Baslik</a>
                            </td>
                            <td>
                                @if (y.Durum == YaziDurum.Yayinda)
                                {
                                    <span class="badge text-bg-success">Yayında</span>
                                }
                                else
                                {
                                    <span class="badge text-bg-secondary">Taslak</span>
                                }
                            </td>
                            <td class="text-nowrap text-secondary">
                                @(((DateTime)y.Tarih).ToString("d MMM yyyy"))
                            </td>
                        </tr>
                    }
                </tbody>
            </table>
        </div>
    </div>
}
```

### 📖 Dört Bootstrap kalıbı

**`row-cols-2 row-cols-md-3 row-cols-xl-6`** — satır başına kart sayısı. Telefonda 2, tablette 3, geniş ekranda 6. `col-*` ile her karta ayrı sınıf yazmaya göre çok daha kısa; kart sayısı değişse bile dokunmanız gerekmiyor.

**`h-100`** — kartları eşit yüksekliğe getirir. Olmazsa metni uzun olan kart diğerlerinden uzun durur ve satır düzensiz görünür.

**`<a class="card">`** — kartın **tamamı** tıklanabilir. Sadece başlığı bağlantı yapmaktan iyi: tıklama hedefi büyüdüğü için fare ve parmakla isabet kolaylaşır. `text-decoration-none` ile alt çizgiyi kaldırıyoruz.

**`table-responsive`** — dar ekranda tabloyu yatay kaydırılabilir yapar. Olmazsa tablo sayfayı taşırır ve **tüm sayfa** yatay kayar. Panelde her tabloyu buna sarın.

---

## ⌨️ Adım 6: Anasayfa'yı taban sınıfa bağla

`Controllers/AnasayfaController.cs`:

```csharp
using Microsoft.AspNetCore.Mvc;

namespace KisiselBlog.Controllers;

// ⭐ GenelController'dan miras → [AllowAnonymous] otomatik
public class AnasayfaController : GenelController
{
    public IActionResult Index()
    {
        ViewData["Title"] = "";
        return View();
    }
}
```

---

## ▶️ Test

| # | Test | Beklenen |
|---|---|---|
| 1 | `/` (giriş yapmadan) | ✅ Ana sayfa **açılıyor** |
| 2 | `/yonetim` (giriş yapmadan) | `/giris?donusUrl=%2Fyonetim`'e yönleniyor |
| 3 | Yanlış şifre | "Kullanıcı adı veya şifre hatalı" |
| 4 | `admin` / `blog2026` | Panele gidiyor |
| 5 | Panel sayaçları | 0 yazı, 1 sayfa, 0 mesaj |
| 6 | Footer'daki "Yönetim" bağlantısı | Giriş yapınca görünüyor |
| 7 | Çıkış yap | Ana sayfaya dönüyor |
| 8 | Çıkıştan sonra `/` | ✅ Hâlâ açık |
| 9 | Şifre alanına `' OR '1'='1` | Girilemiyor (EF parametreliyor) |

> **2. maddeyi inceleyin:** Adres çubuğunda `donusUrl` parametresi var. Giriş yaptıktan sonra `/yonetim`'e değil, **istenen sayfaya** döndürüyor. Bu küçük detay kullanıcı deneyimini belirler.

---

## ⚠️ Sık yapılan hatalar

| Hata | Sebep | Çözüm |
|---|---|---|
| Ana sayfa giriş istiyor | `GenelController`'dan miras alınmamış | Miras ekle |
| Sonsuz yönlendirme döngüsü | `HesapController`'da `[AllowAnonymous]` yok | Ekle |
| Giriş oluyor ama panel açılmıyor | `UseAuthentication` yanlış sırada | `UseAuthorization`'dan **önce** |
| Giriş sonrası hep panele gidiyor | `ReturnUrlParameter` ile parametre adı farklı | İkisi de `donusUrl` |
| `/giris` yerine `/Hesap/Giris` | Rotada baştaki `/` yok | `[HttpGet("/giris")]` |
| Giriş sayfası biçimsiz | `Layout = null` olduğu için Bootstrap elle bağlanmalı | `<head>`e iki CDN satırı |
| Onay kutusu kocaman | `form-control` yazılmış | `form-check-input` olmalı |
| Panel tablosu sayfayı taşırıyor | `table-responsive` yok | Tabloyu ona sarın |
| Razor derleme hatası (CDN satırı) | Tek `@` yazılmış | `bootstrap@@5.3.3` |
| Şifre doğru ama girmiyor | Seed hash'i farklı | `SifreYardimcisi.Hashle("blog2026")` ile karşılaştır |
| `GroupBy` null hatası | Hiç yazı yok | `ozet?.Toplam ?? 0` |

---

## ✏️ Alıştırmalar

1. Giriş sistemini kurun, 9 test senaryosunu geçirin.

2. **Yetki yönü deneyi:** `AnasayfaController`'ın `GenelController` mirasını kaldırın. Ana sayfa artık giriş istiyor. Geri alın.

3. `UseAuthentication` ve `UseAuthorization` satırlarının yerini değiştirin. Doğru şifreyle bile giremediğinizi görün. Geri alın.

4. Şifre değiştirme sayfası ekleyin (`/yonetim/sifre`).
   İpucu: `[Compare(nameof(YeniSifre))]` iki alanın eşitliğini kontrol eder.

**Düşünme soruları:**
- "Global açık + `[Authorize]`" seçseydik ve bir yönetim controller'ında özniteliği unutsaydık ne olurdu?
- Taban sınıf yerine her controller'a tek tek öznitelik yazsaydık, 6. controller'da riski ne olurdu?
- Çıkış neden POST?

---

👉 Sonraki: [`05-yazi-yonetimi.md`](05-yazi-yonetimi.md)
