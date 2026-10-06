# Modül 3 — İki Layout

**Süre:** 1-2 ders saati

---

## 🎯 Bu derste ne yapacağız

**Yeni kavram: tek uygulamada iki ayrı layout.**

Genel site ile yönetim paneli aynı projede ama bambaşka görünecek. İkisi de Bootstrap kullanacak — farkı Bootstrap'in **hangi** bileşenlerini seçtiğimiz yaratacak.

| | Genel site | Yönetim paneli |
|---|---|---|
| Kullanıcı | Okuyucu, misafir | Site sahibi |
| Amaç | **Okumak** | **İş bitirmek** |
| Yerleşim | Solda sabit yan sütun | Üstte navbar |
| Bootstrap | `row` + `col-lg-3/9` | `navbar` + `container` |
| Genişlik | Yazı `46rem`'de durur | Geniş, tablo için |
| Renk | Sarı vurgu | Bootstrap'in standart mavisi |

Son satır önemli: **panelde kişiselleştirme yapmıyoruz.** Sarı butonlar, Montserrat başlıklar genel sitede kalıyor. Panel, Bootstrap'in hazır hâliyle duruyor — çünkü oranın işi tanıdık olmak, ziyaretçiyi etkilemek değil.

---

## 📖 Neden iki layout?

**Sınıfa sor:** *"Blogun okuyucu tarafı ile yönetim paneli aynı mı görünmeli?"*

Hayır — çünkü farklı işler yapıyorlar. Aynı layout'u iki işe zorlamak ikisini de bozar: okuyucu tarafı panele benzer (soğuk), panel okuyucu tarafına benzer (verimsiz).

### Üç yol var

| Yol | Nasıl | Zorluk |
|---|---|---|
| Tek layout + `if` | Layout içinde "admin mi?" kontrolü | Kolay ama **kirli** |
| **İki layout dosyası** | Her view hangisini kullanacağını söyler | ⭐ **Bizim seçimimiz** |
| Areas | `/Areas/Yonetim/` klasör yapısı | Güçlü ama büyük projeler için |

> **Neden Areas değil?** Areas, ayrı controller/view/route ağacı kurar. 5 controller'lık bir blogda bu fazla gelir. İki layout dosyası aynı işi daha az dosyayla görür.
>
> Öğrenci merak ederse: *"Projeniz 30 controller'a çıkarsa Areas'a geçin."*

---

## 📖 Layout nasıl seçilir?

Üç yöntem var, ikisini kullanacağız:

```csharp
// 1) View'ın başında tek tek — tekrar çok
@{ Layout = "_YonetimLayout"; }

// 2) Klasör bazlı _ViewStart.cshtml  ⭐ bizim yolumuz
//    Views/Yonetim/_ViewStart.cshtml → o klasördeki TÜM view'lar

// 3) Controller'da
//    return View("Index", "_YonetimLayout");   ← az kullanılır
```

---

## ⌨️ Adım 1 — `_ViewImports.cshtml`

`Views/_ViewImports.cshtml` — tüm view'larda geçerli using'ler:

```csharp
@using KisiselBlog
@using KisiselBlog.Models
@using KisiselBlog.Data
@addTagHelper *, Microsoft.AspNetCore.Mvc.TagHelpers
```

---

## ⌨️ Adım 2 — `_ViewStart` dosyaları

`Views/_ViewStart.cshtml`:

```csharp
@{
    Layout = "_Layout";
}
```

`Views/Yonetim/_ViewStart.cshtml` — **bu klasör oluşturulacak**:

```csharp
@{
    // ⭐ Bu dosya, Views/Yonetim/ altındaki tüm view'ların
    //    layout'unu ezer. Tek tek yazmaya gerek kalmaz.
    Layout = "_YonetimLayout";
}
```

---

## ⌨️ Adım 3 — Site bilgileri `appsettings.json`'dan

Yan sütunda adınız, mesleğiniz, kısa tanıtımınız ve sosyal medya bağlantılarınız olacak. Bunları view'a **gömmek** kötü fikir: değiştirmek için kodu düzenleyip projeyi yeniden derlemek gerekir.

Ayara taşıyoruz.

### 3.1 Ayar sınıfı

`Models/SiteAyarlari.cs`:

```csharp
namespace KisiselBlog.Models;

/// <summary>
/// appsettings.json'daki "Site" bölümünün C# karşılığı.
/// Özellik adları JSON'daki anahtarlarla aynı olmalı.
/// </summary>
public class SiteAyarlari
{
    public string Baslik { get; set; } = "Blog";
    public string AdSoyad { get; set; } = "";
    public string Unvan { get; set; } = "";
    public string Hakkimda { get; set; } = "";
    public string ProfilFoto { get; set; } = "";
    public string Aciklama { get; set; } = "";

    /// <summary>Sosyal medya bağlantıları — ikon adı : adres</summary>
    public List<SosyalBaglanti> Sosyal { get; set; } = new();
}

public class SosyalBaglanti
{
    /// <summary>Bootstrap Icons adı: "github", "linkedin", "twitter-x"…</summary>
    public string Ikon { get; set; } = "";
    public string Adres { get; set; } = "";
    public string Etiket { get; set; } = "";
}
```

### 3.2 `appsettings.json`'a ekleyin

Mevcut `appsettings.json`'un içine, `ConnectionStrings`'in yanına:

```json
{
  "ConnectionStrings": {
    "BlogDb": "Server=localhost\\SQLEXPRESS;Database=BlogDB;Trusted_Connection=True;TrustServerCertificate=True;"
  },

  "Site": {
    "Baslik": "Harun's Blog",
    "AdSoyad": "Dr. Harun AKSAYA",
    "Unvan": "Computer Scientist",
    "Hakkimda": "Teknoloji, yaşam ve şeyleri gerçekleştirme sanatı üzerine düşünceler.",
    "ProfilFoto": "/img/profil.jpg",
    "Aciklama": "Yazılım, tasarım ve okuduklarım üzerine notlar.",
    "Sosyal": [
      { "Ikon": "mortarboard", "Adres": "https://scholar.google.com/", "Etiket": "Akademik" },
      { "Ikon": "github",      "Adres": "https://github.com/",         "Etiket": "GitHub" },
      { "Ikon": "twitter-x",   "Adres": "https://x.com/",              "Etiket": "X" },
      { "Ikon": "instagram",   "Adres": "https://instagram.com/",      "Etiket": "Instagram" },
      { "Ikon": "linkedin",    "Adres": "https://linkedin.com/",       "Etiket": "LinkedIn" },
      { "Ikon": "youtube",     "Adres": "https://youtube.com/",        "Etiket": "YouTube" },
      { "Ikon": "envelope",    "Adres": "/iletisim",                   "Etiket": "E-posta" }
    ]
  },

  "Logging": {
    "LogLevel": { "Default": "Information", "Microsoft.AspNetCore": "Warning" }
  },
  "AllowedHosts": "*"
}
```

`Ikon` değerleri Bootstrap Icons adlarıdır. `"github"` → `<i class="bi bi-github">`. Tüm liste: [icons.getbootstrap.com](https://icons.getbootstrap.com)

Profil fotoğrafınızı `wwwroot/img/profil.jpg` olarak koyun. (Yoksa Adım 5'teki yedek harf çemberi görünür.)

### 3.3 `Program.cs`'e bağlayın

```csharp
// ⭐ "Site" bölümünü SiteAyarlari nesnesine çevirip DI'a koy
builder.Services.AddSingleton(
    builder.Configuration.GetSection("Site").Get<SiteAyarlari>()
    ?? new SiteAyarlari());
```

`GetSection("Site").Get<SiteAyarlari>()` JSON'u sınıfa dönüştürür (bu işleme **binding** denir). `?? new SiteAyarlari()` ise bölüm hiç yoksa boş ayarla devam etmeyi sağlar — `null` dönerse layout'ta `NullReferenceException` alırdınız.

`AddSingleton` yeterli, çünkü bu değerler uygulama çalışırken değişmiyor.

> `using KisiselBlog.Models;` satırını `Program.cs`'in başına eklemeyi unutmayın.

---

## ⌨️ Adım 4 — Genel site layout'u

`Views/Shared/_Layout.cshtml`:

```html
@inject SiteAyarlari Site
@{
    var sayfaBasligi = ViewData["Title"] as string;
    var tamBaslik = string.IsNullOrEmpty(sayfaBasligi)
        ? Site.Baslik
        : $"{sayfaBasligi} — {Site.Baslik}";

    var aciklama = ViewData["Aciklama"] as string ?? Site.Aciklama;
}

<!DOCTYPE html>
<html lang="tr">
<head>
    <meta charset="utf-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />

    <title>@tamBaslik</title>
    <meta name="description" content="@aciklama" />

    @* Sosyal medya paylaşım önizlemesi *@
    <meta property="og:title" content="@tamBaslik" />
    <meta property="og:description" content="@aciklama" />
    <meta property="og:type" content="website" />

    @* ── Yazı tipleri ── *@
    <link rel="preconnect" href="https://fonts.googleapis.com">
    <link rel="preconnect" href="https://fonts.gstatic.com" crossorigin>
    <link href="https://fonts.googleapis.com/css2?family=Montserrat:wght@@700;800&family=Lato:wght@@400;700&display=swap"
          rel="stylesheet">

    @* ── Bootstrap ── *@
    <link href="https://cdn.jsdelivr.net/npm/bootstrap@@5.3.3/dist/css/bootstrap.min.css"
          rel="stylesheet">
    <link href="https://cdn.jsdelivr.net/npm/bootstrap-icons@@1.11.3/font/bootstrap-icons.css"
          rel="stylesheet">

    @* ── Bizim katmanımız — EN SONDA ── *@
    <link rel="stylesheet" href="~/css/site.css" asp-append-version="true" />

    @await RenderSectionAsync("Head", required: false)
</head>
<body>

@* ══════ MOBİL ÜST ÇUBUK — sadece küçük ekranda ══════ *@
<div class="d-lg-none d-flex justify-content-between align-items-center
            px-3 py-2 border-bottom" style="background: var(--yan-zemin);">
    <a href="/" class="h6 mb-0 text-decoration-none"
       style="color: var(--bs-heading-color);">@Site.AdSoyad</a>

    <button class="btn btn-sm btn-outline-secondary border-0"
            type="button"
            data-bs-toggle="collapse"
            data-bs-target="#yanSutun"
            aria-expanded="false"
            aria-controls="yanSutun"
            aria-label="Menüyü aç kapat">
        <i class="bi bi-list fs-5"></i>
    </button>
</div>

<div class="container-fluid">
    <div class="row">

        @* ══════ YAN SÜTUN ══════ *@
        <aside id="yanSutun" class="col-lg-3 yan-sutun p-4 collapse d-lg-block">
            <partial name="_YanSutun" />
        </aside>

        @* ══════ İÇERİK ══════ *@
        <main class="col-lg-9 p-4 p-lg-5">
            <div class="icerik-alani">
                @RenderBody()
            </div>
        </main>

    </div>
</div>

<script src="https://cdn.jsdelivr.net/npm/bootstrap@@5.3.3/dist/js/bootstrap.bundle.min.js"></script>
@await RenderSectionAsync("Scripts", required: false)

</body>
</html>
```

### ⚠️ Razor'da `@` işareti: `@@` yazmak

Yukarıdaki CDN adreslerinde `bootstrap@@5.3.3` yazdık, `bootstrap@5.3.3` değil.

Sebep: Razor'da `@` kod başlatır. `bootstrap@5.3.3` yazarsanız Razor `5` ile başlayan bir C# ifadesi arar ve derleme hatası verir:

```
error RZ1011: Unterminated string literal
```

`@@` tek bir `@` karakteri üretir. Aynı şey `wght@@700;800` için de geçerli.

> **Derste gösterin:** Önce tek `@` ile yazın, hatayı alın, sonra düzeltin. Bu hata ileride CSS veya e-posta adresi yazarken tekrar karşılarına çıkacak.

### 📖 `collapse d-lg-block` ikilisi nasıl çalışıyor?

Yan sütun mobilde ekranı kaplıyor; yazılara ulaşmak için uzun uzun kaydırmak gerekirdi. Bootstrap'in `collapse` bileşeniyle çözüyoruz:

| Sınıf | Ne yapar |
|---|---|
| `collapse` | `display: none` — öğe gizli |
| `d-lg-block` | ≥992px'te `display: block !important` |
| `data-bs-toggle="collapse"` | Butonu açıp kapatıcı yapar |
| `data-bs-target="#yanSutun"` | Neyi açıp kapatacağını söyler |

Küçük ekranda `collapse` kazanıyor (gizli, buton açıyor). Büyük ekranda `d-lg-block`'un `!important`'ı kazanıyor ve yan sütun **her zaman** görünüyor — buton da `d-lg-none` ile gizlendiği için kimse onu kapatamıyor.

`aria-expanded` ve `aria-controls` ekran okuyucular için: butonun bir şeyi açtığını ve şu an açık mı kapalı mı olduğunu bildirir. Bootstrap `aria-expanded`'ı tıklamada kendisi güncelliyor.

---

## ⌨️ Adım 5 — Yan sütun

Ayrı dosyaya alıyoruz, çünkü layout yeterince uzun.

`Views/Shared/_YanSutun.cshtml`:

```html
@inject SiteAyarlari Site
@inject BlogDbContext Db
@{
    // ⚠️ Normalde veri erişimi controller'da yapılır.
    //    Burada istisna yapıyoruz: yan sütun HER sayfada var,
    //    her controller'a aynı kodu yazmak tekrar olurdu.
    //    (Daha temiz yol: View Component — Modül 7'de değinilecek.)

    var menuSayfalari = Db.Sayfalar
        .Where(s => s.MenudeGoster)
        .OrderBy(s => s.Sira)
        .Select(s => new { s.Baslik, s.Slug })
        .ToList();

    var etiketler = Db.Etiketler
        .OrderBy(e => e.Ad)
        .Select(e => new { e.Ad, e.Slug })
        .ToList();
}

@* ══ Profil ══ *@
<div class="text-center mb-4">

    @if (!string.IsNullOrEmpty(Site.ProfilFoto))
    {
        <img src="@Site.ProfilFoto" alt="@Site.AdSoyad"
             class="rounded-circle profil-foto mb-3" />
    }
    else
    {
        @* Fotoğraf yoksa baş harfler *@
        <div class="rounded-circle profil-foto mb-3 d-flex align-items-center
                    justify-content-center mx-auto"
             style="background:#dfe4ea; color:#8b95a3;
                    font-family:'Montserrat',sans-serif; font-size:3rem; font-weight:800;">
            @GetBasHarfler(Site.AdSoyad)
        </div>
    }

    @* ⚠️ h1 DEĞİL, p. Yan sütun her sayfada görünüyor; buraya h1 koyarsak
         yazı sayfasında iki tane h1 olur. Sayfanın h1'i o sayfanın konusu
         olmalı — burada sadece büyük görünmesini istiyoruz, o kadar. *@
    <p class="h5 mb-1">@Site.AdSoyad</p>
    <p class="small mb-3" style="color: var(--soluk);">@Site.Unvan</p>

    @if (Site.Sosyal.Any())
    {
        <div class="sosyal d-flex justify-content-center flex-wrap gap-3">
            @foreach (var b in Site.Sosyal)
            {
                <a href="@b.Adres" aria-label="@b.Etiket" title="@b.Etiket"
                   target="_blank" rel="noopener">
                    <i class="bi bi-@b.Ikon"></i>
                </a>
            }
        </div>
    }
</div>

@* ══ Hakkımda ══ *@
<section class="mb-4">
    <p class="yan-baslik">Hakkımda</p>
    <p class="small mb-2">@Site.Hakkimda</p>

    @if (menuSayfalari.Any())
    {
        <ul class="list-unstyled small mb-0">
            @foreach (var s in menuSayfalari)
            {
                <li><a href="/@s.Slug">@s.Baslik</a></li>
            }
        </ul>
    }
</section>

@* ══ Abone ol — Modül 9'da çalışır hâle gelecek ══ *@
<section class="mb-4">
    <p class="yan-baslik">Abone ol</p>
    <form class="d-flex" method="post" asp-controller="Abonelik" asp-action="Kaydol">
        <input type="email" name="eposta" required
               class="form-control form-control-sm rounded-0"
               placeholder="eposta@@ornek.com" aria-label="E-posta adresiniz" />
        <button type="submit" class="btn btn-sari btn-sm rounded-0 px-3">Kaydol</button>
    </form>
    @if (TempData["AboneSonuc"] != null)
    {
        <p class="small mb-0 mt-2" style="color: var(--soluk);">@TempData["AboneSonuc"]</p>
    }
</section>

@* ══ Ara — Modül 11'de çalışır hâle gelecek ══ *@
<section class="mb-4">
    <p class="yan-baslik">Ara</p>
    <form method="get" asp-controller="Arama" asp-action="Index" class="position-relative">
        <input type="search" name="q" class="form-control arama-kutu pe-4"
               placeholder="Aramak için yazın…" aria-label="Sitede ara" />
        <i class="bi bi-search position-absolute end-0 top-50 translate-middle-y"
           style="color: var(--soluk);" aria-hidden="true"></i>
    </form>
</section>

@* ══ Keşfet — Modül 11'de çalışır hâle gelecek ══ *@
<section class="mb-4">
    <p class="yan-baslik">Keşfet</p>
    <a asp-controller="Yazi" asp-action="Rastgele"
       class="text-decoration-none d-inline-flex align-items-center gap-2"
       style="color: var(--bs-body-color);">
        <i class="bi bi-shuffle"></i> Rastgele yazı
    </a>
</section>

@* ══ Etiketler ══ *@
@if (etiketler.Any())
{
    <section class="mb-4">
        <p class="yan-baslik">Etiketler</p>
        <div>
            @foreach (var e in etiketler)
            {
                <a href="/etiket/@e.Slug" class="etiket-link">#@e.Ad</a>
            }
        </div>
    </section>
}

<hr />

<div class="d-flex justify-content-between align-items-center small"
     style="color: var(--soluk);">
    <span>&copy; @DateTime.Now.Year @Site.AdSoyad</span>
    <span>
        @if (User.Identity?.IsAuthenticated == true)
        {
            <a asp-controller="Panel" asp-action="Index">Yönetim</a>
        }
        else
        {
            <a asp-controller="Hesap" asp-action="Giris">Giriş</a>
        }
    </span>
</div>

@functions {
    /// <summary>"Dr. Harun AKSAYA" → "HA"</summary>
    static string GetBasHarfler(string adSoyad)
    {
        if (string.IsNullOrWhiteSpace(adSoyad)) return "?";

        var parcalar = adSoyad
            .Split(' ', StringSplitOptions.RemoveEmptyEntries)
            .Where(p => !p.EndsWith(".")) // "Dr." gibi ekleri atla
            .ToList();

        if (parcalar.Count == 0) return "?";
        if (parcalar.Count == 1) return parcalar[0][..1].ToUpperInvariant();

        return (parcalar[0][..1] + parcalar[^1][..1]).ToUpperInvariant();
    }
}
```

### 📖 Üç küçük ayrıntı

**`@functions { }`** — view'ın içinde yardımcı metot tanımlar. Küçük biçimlendirme işleri için uygun. İş mantığı buraya **yazılmaz**; `GetBasHarfler` sadece metin kesiyor.

**`parcalar[0][..1]`** — C# 8'in **aralık** (range) sözdizimi. `[..1]` = "baştan 1. karaktere kadar" = ilk karakter. `parcalar[^1]` = "sondan 1." = son eleman.

> Niye `parcalar[0][0]` değil? O bir `char` döner, `string` değil. `+` ile birleştirmek istediğimizde `char + char` sayı toplamasına dönüşür ve `"HA"` yerine `137` alırsınız (`'H'`=72, `'A'`=65). `[..1]` `string` döndüğü için bu tuzağı yaşamazsınız.

**`rel="noopener"`** — `target="_blank"` ile açılan sayfa, JavaScript'le sizin sayfanıza `window.opener` üzerinden erişebilir. `noopener` o bağı koparır. `target="_blank"` kullandığınız her yerde olmalı.

### ⚠️ Henüz olmayan action'lar

Yan sütun üç action'a bağlanıyor ama hiçbiri henüz yok:

| Bağlantı | Gelecek |
|---|---|
| `Abonelik/Kaydol` | Modül 9 |
| `Arama/Index` | Modül 11 |
| `Yazi/Rastgele` | Modül 11 |

**`asp-action` olmayan bir action'a işaret ederse ne olur?** Hata vermez — Tag Helper boş `href=""` üretir ve bağlantı sayfayı kendine yeniler. Bu yüzden şimdilik sorun değil; modüller geldikçe kendiliğinden çalışmaya başlayacaklar.

> **Derste gösterin:** Sağ tık → Kaynağı görüntüle → `href=""` olan bağlantıları bulun. Tag Helper'ın action'ı bulamadığında sessizce boş bıraktığını görmek, ileride "bağlantım neden çalışmıyor?" sorusunu çözer.

---

## ⌨️ Adım 6 — Yönetim layout'u

Panelde **kişiselleştirme yok.** Bootstrap'in hazır `navbar`'ı, hazır renkleri.

`Views/Shared/_YonetimLayout.cshtml`:

```html
@inject SiteAyarlari Site
@{
    var yol = Context.Request.Path.Value?.ToLowerInvariant() ?? "";

    // Menüde hangi öğe aktif? Yol parçasına bakıyoruz.
    string Aktif(string parca) => yol.Contains("/" + parca) ? "active" : "";
}

<!DOCTYPE html>
<html lang="tr">
<head>
    <meta charset="utf-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>@ViewData["Title"] — Yönetim</title>

    @* ⭐ Yönetim paneli arama motorlarına kapalı *@
    <meta name="robots" content="noindex, nofollow" />

    <link href="https://cdn.jsdelivr.net/npm/bootstrap@@5.3.3/dist/css/bootstrap.min.css"
          rel="stylesheet">
    <link href="https://cdn.jsdelivr.net/npm/bootstrap-icons@@1.11.3/font/bootstrap-icons.css"
          rel="stylesheet">

    @* ⚠️ site.css'i BİLEREK yüklemiyoruz.
         Panel Bootstrap'in standart görünümünde kalsın. *@
</head>
<body class="bg-body-tertiary">

<nav class="navbar navbar-expand-lg navbar-dark bg-dark">
    <div class="container-fluid">

        <a class="navbar-brand" asp-controller="Panel" asp-action="Index">
            <i class="bi bi-journal-text me-1"></i> @Site.Baslik
        </a>

        <button class="navbar-toggler" type="button"
                data-bs-toggle="collapse" data-bs-target="#panelMenu"
                aria-controls="panelMenu" aria-expanded="false"
                aria-label="Menüyü aç kapat">
            <span class="navbar-toggler-icon"></span>
        </button>

        <div class="collapse navbar-collapse" id="panelMenu">

            <ul class="navbar-nav me-auto">
                <li class="nav-item">
                    <a class="nav-link @(yol == "/yonetim" || yol == "/yonetim/" ? "active" : "")"
                       asp-controller="Panel" asp-action="Index">Panel</a>
                </li>
                <li class="nav-item">
                    <a class="nav-link @Aktif("yazi")"
                       asp-controller="YaziYonetim" asp-action="Index">Yazılar</a>
                </li>
                <li class="nav-item">
                    <a class="nav-link @Aktif("sayfa")"
                       asp-controller="SayfaYonetim" asp-action="Index">Sayfalar</a>
                </li>
                <li class="nav-item">
                    <a class="nav-link @Aktif("etiket")"
                       asp-controller="EtiketYonetim" asp-action="Index">Etiketler</a>
                </li>
                <li class="nav-item">
                    <a class="nav-link @Aktif("mesaj")"
                       asp-controller="MesajYonetim" asp-action="Index">
                        Mesajlar
                        @* Okunmamış sayacı Modül 8'de buraya gelecek *@
                    </a>
                </li>
            </ul>

            <ul class="navbar-nav">
                <li class="nav-item">
                    <a class="nav-link" asp-controller="Anasayfa" asp-action="Index"
                       target="_blank" rel="noopener">
                        Siteyi gör <i class="bi bi-box-arrow-up-right small"></i>
                    </a>
                </li>
                <li class="nav-item d-flex align-items-center">
                    <span class="navbar-text me-2">@User.Identity?.Name</span>
                    <form asp-controller="Hesap" asp-action="Cikis" method="post"
                          class="d-inline">
                        <button type="submit" class="btn btn-sm btn-outline-light">
                            Çıkış
                        </button>
                    </form>
                </li>
            </ul>

        </div>
    </div>
</nav>

<div class="container-fluid px-4 py-4">

    <div class="d-flex justify-content-between align-items-center mb-4">
        <h1 class="h3 mb-0">@ViewData["Title"]</h1>
        @await RenderSectionAsync("BaslikEylem", required: false)
    </div>

    @* Bildirimler — tüm panel sayfalarında ortak *@
    @if (TempData["Basarili"] != null)
    {
        <div class="alert alert-success alert-dismissible fade show">
            <i class="bi bi-check-circle me-1"></i> @TempData["Basarili"]
            <button type="button" class="btn-close" data-bs-dismiss="alert"
                    aria-label="Kapat"></button>
        </div>
    }
    @if (TempData["Uyari"] != null)
    {
        <div class="alert alert-warning alert-dismissible fade show">
            <i class="bi bi-exclamation-triangle me-1"></i> @TempData["Uyari"]
            <button type="button" class="btn-close" data-bs-dismiss="alert"
                    aria-label="Kapat"></button>
        </div>
    }
    @if (TempData["Hata"] != null)
    {
        <div class="alert alert-danger alert-dismissible fade show">
            <i class="bi bi-x-circle me-1"></i> @TempData["Hata"]
            <button type="button" class="btn-close" data-bs-dismiss="alert"
                    aria-label="Kapat"></button>
        </div>
    }

    @RenderBody()

</div>

<script src="https://cdn.jsdelivr.net/npm/bootstrap@@5.3.3/dist/js/bootstrap.bundle.min.js"></script>
@await RenderSectionAsync("Scripts", required: false)

</body>
</html>
```

### 📖 `RenderSectionAsync("BaslikEylem")`

Panel başlığının sağına "Yeni yazı" gibi butonlar gelecek. Her sayfa farklı buton koyabilsin diye **bölüm** (section) açtık:

```html
@* Herhangi bir yönetim view'ında: *@
@section BaslikEylem {
    <a asp-action="Olustur" class="btn btn-primary">
        <i class="bi bi-plus-lg me-1"></i> Yeni yazı
    </a>
}
```

`required: false` → bölümü tanımlamayan sayfalar hata vermez.

### 📖 İki Bootstrap ayrıntısı

**`alert-dismissible fade show` + `btn-close`** — kapatılabilir uyarı. `data-bs-dismiss="alert"` butonu kapatıcı yapar. Bootstrap JS'i yüklü olmazsa buton görünür ama çalışmaz.

**`bg-body-tertiary`** — Bootstrap 5.3'ün yeni "hafif gri zemin" sınıfı. Eski sürümlerdeki `bg-light`'ın yerini aldı. Beyaz kartlar bu zeminde ayrışıyor.

### ⚠️ Panel view'larında `site.css` değişkenlerini KULLANMAYIN

Panel `site.css`'i yüklemediği için `--soluk`, `--sari`, `--yan-zemin` orada **tanımsızdır.** Tanımsız bir `var()` sessizce düşer: hata almazsınız, kural hiç uygulanmaz.

```html
@* ❌ Panelde: renk uygulanmaz, metin siyah kalır *@
<td style="color: var(--soluk);">12 Oca 2026</td>

@* ✅ Bootstrap karşılığı *@
<td class="text-secondary">12 Oca 2026</td>
```

Panelde ihtiyaç duyacağınız karşılıklar:

| Genel sitede | Panelde |
|---|---|
| `var(--soluk)` | `class="text-secondary"` |
| `var(--yan-zemin)` | `class="bg-body-tertiary"` |
| `background: var(--sari)` (satır vurgusu) | `class="table-warning"` |
| `var(--bs-border-color)` | **ikisinde de çalışır** — Bootstrap'in kendi değişkeni |

Son satır önemli: `--bs-` ile başlayan değişkenler Bootstrap'ten geliyor, her iki layout'ta da var. Tanımsız olanlar sadece **bizim** eklediklerimiz.

---

## ⌨️ Adım 7 — `Program.cs` ve geçici controller'lar

### 7.1 View arama yolu

`PanelController.Index()` varsayılan olarak şurada arar:

```
Views/Panel/Index.cshtml
```

Ama biz `Views/Yonetim/Panel/Index.cshtml`'e koyacağız ki `Views/Yonetim/_ViewStart.cshtml` devreye girsin.

**Çözüm:** `Program.cs`'e arama yolu ekleyin:

```csharp
builder.Services.AddControllersWithViews()
    .AddRazorOptions(secenekler =>
    {
        // ⭐ Yönetim controller'ları view'larını Views/Yonetim/ altında arasın
        secenekler.ViewLocationFormats.Add("/Views/Yonetim/{1}/{0}.cshtml");
        secenekler.ViewLocationFormats.Add("/Views/Yonetim/Shared/{0}.cshtml");
    });
```

`{1}` = controller adı, `{0}` = view adı.

> **Alternatif:** Her yönetim view'ının başına `@{ Layout = "_YonetimLayout"; }` yazmak. Daha basit ama her dosyada tekrar eder. Biz klasör yolunu tercih ediyoruz.

### 7.2 `Program.cs`'in bu noktadaki tam hâli

```csharp
using System.Globalization;
using KisiselBlog.Data;
using KisiselBlog.Models;
using Microsoft.EntityFrameworkCore;

var builder = WebApplication.CreateBuilder(args);

// ⭐ TÜRKÇE KÜLTÜR
// Olmadan: ToString("d MMMM yyyy") → "24 January 2026"
// Olunca:                          → "24 Ocak 2026"
// Ayrıca ondalık ayracı ve sıralama da Türkçe kurallarına göre olur.
var kultur = new CultureInfo("tr-TR");
CultureInfo.DefaultThreadCurrentCulture = kultur;
CultureInfo.DefaultThreadCurrentUICulture = kultur;

// ── Veritabanı ──
builder.Services.AddDbContext<BlogDbContext>(secenekler =>
    secenekler.UseSqlServer(builder.Configuration.GetConnectionString("BlogDb")));

// ── Site ayarları (appsettings.json → "Site") ──
builder.Services.AddSingleton(
    builder.Configuration.GetSection("Site").Get<SiteAyarlari>()
    ?? new SiteAyarlari());

// ── MVC + yönetim view arama yolu ──
builder.Services.AddControllersWithViews()
    .AddRazorOptions(secenekler =>
    {
        secenekler.ViewLocationFormats.Add("/Views/Yonetim/{1}/{0}.cshtml");
        secenekler.ViewLocationFormats.Add("/Views/Yonetim/Shared/{0}.cshtml");
    });

var app = builder.Build();

if (!app.Environment.IsDevelopment())
{
    app.UseExceptionHandler("/Hata");
    app.UseHsts();
}

app.UseHttpsRedirection();
app.UseStaticFiles();
app.UseRouting();

// ⚠️ Kimlik doğrulama Modül 4'te eklenecek:
// app.UseAuthentication();
// app.UseAuthorization();

app.MapControllerRoute(
    name: "default",
    pattern: "{controller=Anasayfa}/{action=Index}/{id?}");

app.Run();
```

> `{controller=Anasayfa}` — varsayılan controller `Home` değil **`Anasayfa`**. Şablondan gelen `HomeController.cs` ve `Views/Home/` klasörünü silebilirsiniz.

### 7.3 Geçici controller'lar

Layout'ları görebilmek için iki basit controller.

`Controllers/AnasayfaController.cs`:

```csharp
using Microsoft.AspNetCore.Mvc;

namespace KisiselBlog.Controllers;

public class AnasayfaController : Controller
{
    public IActionResult Index()
    {
        ViewData["Title"] = "";     // ana sayfada başlık eklenmesin
        return View();
    }
}
```

`Views/Anasayfa/Index.cshtml`:

```html
@{
    ViewData["Title"] = "";
}

<article class="mb-5 pb-4">
    <p class="yazi-tarih mb-2">6 EKİM 2026</p>
    <h2 class="h1 mb-3">
        <a href="#" class="baslik-link">Örnek yazı başlığı</a>
    </h2>
    <p class="mb-3">
        Yazı listesi Modül 6'da buraya gelecek. Şimdilik yerleşimi
        görmek için tek bir örnek duruyor.
    </p>
    <a href="#" class="devam">Devamını oku <span aria-hidden="true">&rarr;</span></a>
</article>
```

`Controllers/PanelController.cs`:

```csharp
using Microsoft.AspNetCore.Mvc;

namespace KisiselBlog.Controllers;

[Route("yonetim")]
public class PanelController : Controller
{
    [HttpGet("")]
    public IActionResult Index()
    {
        ViewData["Title"] = "Panel";
        return View();
    }
}
```

`Views/Yonetim/Panel/Index.cshtml` — ⚠️ klasör yapısına dikkat:

```html
@{
    ViewData["Title"] = "Panel";
}

<div class="row g-3">
    <div class="col-md-3">
        <div class="card">
            <div class="card-body">
                <p class="text-secondary small mb-1">Yazılar</p>
                <p class="h2 mb-0">—</p>
            </div>
        </div>
    </div>
    <div class="col-md-3">
        <div class="card">
            <div class="card-body">
                <p class="text-secondary small mb-1">Taslaklar</p>
                <p class="h2 mb-0">—</p>
            </div>
        </div>
    </div>
    <div class="col-md-3">
        <div class="card">
            <div class="card-body">
                <p class="text-secondary small mb-1">Etiketler</p>
                <p class="h2 mb-0">—</p>
            </div>
        </div>
    </div>
    <div class="col-md-3">
        <div class="card">
            <div class="card-body">
                <p class="text-secondary small mb-1">Mesajlar</p>
                <p class="h2 mb-0">—</p>
            </div>
        </div>
    </div>
</div>

<p class="text-secondary mt-4">Sayaçlar Modül 4'te doldurulacak.</p>
```

---

## ▶️ Çalıştır ve gör

```bash
dotnet run
```

| Adres / eylem | Beklenen |
|---|---|
| `/` | Solda yan sütun, sağda yazı — Lato gövde, Montserrat başlık |
| `/` yan sütun | Profil çemberi sarı, sosyal ikonlar görünüyor |
| `/` etiketler | Boş (veritabanında etiket yok) — bölüm **hiç görünmüyor** |
| `/yonetim` | Üstte koyu navbar, altta dört kart — Bootstrap'in standart görünümü |
| `/yonetim` yazı tipi | **Sistem yazı tipi** — Montserrat/Lato yok. `site.css` yüklenmiyor çünkü |
| Pencereyi daralt (<992px) | Yan sütun kayboluyor, üstte ad + hamburger çıkıyor |
| Hamburgere bas | Yan sütun açılıyor |
| Pencereyi genişlet | Yan sütun geri geliyor, hamburger kayboluyor |
| Uzun sayfada kaydır | Yan sütun **yerinde kalıyor** (`sticky`) |
| `/yonetim` → sekme başlığı | "Panel — Harun's Blog" |

> **İki layout'u yan yana açın** (iki sekme). Aynı Bootstrap, tamamen farklı hissettiriyorlar. Fark: yerleşim, yazı tipi, renk. Tek bir satır bile ortak CSS yazmadık.

### Sticky'yi test etmek

Yan sütun uzun sayfada yerinde kalmalı. Test için `Views/Anasayfa/Index.cshtml`'deki `<article>`'ı 5-6 kere çoğaltın, kaydırın.

Kalmıyorsa: `position: sticky` bir **ata** öğede `overflow: hidden` varsa çalışmaz. Bootstrap'in `row`'unda yok, ama `body`'ye kendiniz eklediyseniz kaldırın.

---

## ⚠️ Sık yapılan hatalar

| Hata | Sebep | Çözüm |
|---|---|---|
| `RZ1011: Unterminated string literal` | CDN adresinde tek `@` | `bootstrap@@5.3.3` yazın |
| Panel genel layout'la açılıyor | `Views/Yonetim/_ViewStart.cshtml` yok | Oluşturun |
| `The view 'Index' was not found` | `ViewLocationFormats` eklenmemiş | Adım 7.1 |
| Hata mesajında aranan yollar listelenir | — | O listeyi okumak en hızlı teşhis yöntemi |
| `NullReferenceException` — `Site` null | `AddSingleton` satırı eklenmemiş, ya da JSON'daki bölüm adı `"Site"` değil | Adım 3.3 |
| Yan sütunda hiçbir şey yok | `_YanSutun.cshtml` `Views/Shared/` içinde değil | Taşıyın |
| Profil fotoğrafı kırık | `wwwroot/img/profil.jpg` yok | Dosyayı koyun, ya da `ProfilFoto`'yu `""` yapın (baş harfler görünür) |
| Hamburger çalışmıyor | Bootstrap JS yüklenmemiş | `bundle.min.js` satırını kontrol edin |
| Mobilde yan sütun hep açık | `collapse` sınıfı yazılmamış | `class="... collapse d-lg-block"` |
| Masaüstünde yan sütun kayıp | `d-lg-block` yazılmamış | Aynı satır |
| Panelde Montserrat başlıklar var | `_YonetimLayout`'a `site.css` eklenmiş | Kaldırın — kasıtlı bir tercih |
| Türkçe karakterler bozuk | `<meta charset="utf-8">` yok | `<head>`in ilk satırı |
| Ay adları İngilizce ("January") | `CultureInfo` ayarlanmamış | Adım 7.2'deki `tr-TR` bloğu |
| Yan sütundaki bağlantılar bir yere gitmiyor | Action henüz yok | Normal — Modül 9 ve 11'de gelecek |
| `Db.Etiketler` hata veriyor | Migration uygulanmamış | `dotnet ef database update` |

---

## 📝 Özet

| Dosya | İşlevi |
|---|---|
| `Views/_ViewImports.cshtml` | Ortak using'ler |
| `Views/_ViewStart.cshtml` | Varsayılan layout |
| `Views/Yonetim/_ViewStart.cshtml` | ⭐ Panel için layout'u ezer |
| `Models/SiteAyarlari.cs` | ⭐ appsettings → C# |
| `Views/Shared/_Layout.cshtml` | Genel site — yan sütunlu |
| `Views/Shared/_YanSutun.cshtml` | Yan sütun içeriği |
| `Views/Shared/_YonetimLayout.cshtml` | Panel — navbar'lı |

**Öğrenilen kavramlar:**

- Klasör bazlı `_ViewStart` ile layout seçimi
- `ViewLocationFormats` ile view arama yolu ekleme
- Yapılandırmayı sınıfa bağlama (`GetSection().Get<T>()`)
- Razor'da `@@` ile tek `@` yazma
- `collapse` + `d-lg-block` ile duyarlı yan sütun
- `position: sticky`
- `RenderSectionAsync` ile sayfaya özel başlık butonu
- Başlık seviyeleri: sayfa başına **tek** `h1`
- `rel="noopener"`

---

## ✏️ Alıştırmalar

1. **Kendi bilgilerinizi girin.** `appsettings.json`'daki `Site` bölümünü kendi adınız, unvanınız ve gerçek sosyal medya adreslerinizle doldurun. Kullanmadığınız sosyal bağlantıları listeden silin — view'a dokunmanız gerekmediğine dikkat edin.

2. **Fotoğrafsız hâli görün.** `ProfilFoto` değerini `""` yapıp yenileyin. Baş harfler çemberi çıkıyor mu? `AdSoyad`'ı "Harun" (tek kelime) yapıp tekrar bakın — ne görünüyor?

3. **Yeni bölüm ekleyin.** Yan sütuna "KİTAPLAR" başlıklı, üç kitap adı listeleyen bir bölüm ekleyin. Sadece `.yan-baslik` sınıfını ve Bootstrap'in `list-unstyled`'ını kullanın; yeni CSS yazmayın.

4. **Panelde sayfa ekleyin.** Panel navbar'ına "Ayarlar" bağlantısı ve karşılığında `/yonetim/ayarlar` adresinde boş bir sayfa ekleyin. `Views/Yonetim/Ayarlar/Index.cshtml` yolunun `ViewLocationFormats` sayesinde çalıştığını doğrulayın.

5. **`BaslikEylem` kullanın.** Panel ana sayfasına bir `@section BaslikEylem` ekleyip "Yeni yazı" butonu koyun. Başlığın sağında çıktığını görün.

**Düşünme soruları:**

- Tek layout + `if (User.IsInRole("Admin"))` ile yapsaydık hangi sorunlar çıkardı?
- `_ViewStart.cshtml` klasör bazlı çalışıyor. İç içe iki `_ViewStart` olsa hangisi kazanır? (Deneyin: `Views/_ViewStart.cshtml` ve `Views/Yonetim/_ViewStart.cshtml` ikisi de var.)
- Yan sütun her sayfada iki veritabanı sorgusu çalıştırıyor. 10.000 ziyaretçili bir günde bu kaç sorgu? Nasıl azaltılır? (İpucu: önbellek — `IMemoryCache`)
- Yönetim paneline neden `noindex` koyduk? Panelin adresi gizli değil mi?

---

👉 Sıradaki: [`04-yonetim-girisi.md`](04-yonetim-girisi.md) — Ters yetki modeli ve giriş ekranı
