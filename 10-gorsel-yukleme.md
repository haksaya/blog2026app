# Modül 10 — Görsel Yükleme

**Süre:** 1 ders saati

---

## 🎯 Bu derste ne yapacağız

Yazılara kapak görseli. **Yeni kavram: `IFormFile` ve dosya güvenliği.**

> Dosya yükleme, web uygulamalarındaki en riskli işlemlerden biridir. Bu modülde kodun yarısı güvenlik kontrolü.

---

## 📖 Dosya yükleme neden riskli?

**Sınıfa sor:** *"Kullanıcı sunucuya dosya yükleyebiliyorsa en kötü ne olabilir?"*

Dört tehlike:

| Risk | Nasıl olur | Önlemimiz |
|---|---|---|
| **Çalıştırılabilir dosya** | `.aspx`, `.php` yükleyip çalıştırmak | Uzantı beyaz listesi + içerik kontrolü |
| **Disk doldurma** | 10 GB'lık dosya göndermek | Boyut sınırı |
| **Yol gezinmesi** | `../../appsettings.json` adlı dosya | Dosya adını **asla** kullanıcıdan alma |
| **Sahte içerik** | `.jpg` adında ama içi script | İmza (magic bytes) kontrolü |

### ⚠️ Uzantı kontrolü tek başına yetmez

```csharp
// ❌ YETERSİZ
if (Path.GetExtension(dosya.FileName) == ".jpg") { /* güvenli sanıyoruz */ }
```

Herhangi biri `zararli.aspx` dosyasının adını `resim.jpg` yapabilir. Uzantı sadece bir **isim**, içerikle ilgisi yok.

**Çözüm: dosyanın ilk baytlarına bakmak.** Her görsel biçiminin kendine özgü bir imzası vardır:

| Biçim | İlk baytlar (hex) |
|---|---|
| JPEG | `FF D8 FF` |
| PNG | `89 50 4E 47` |
| GIF | `47 49 46 38` |
| WEBP | `52 49 46 46` ... `57 45 42 50` |

Bu baytlara **magic bytes** denir. Uzantı yalan söyleyebilir, imza söyleyemez.

### ⚠️ Dosya adını asla kullanma

```csharp
// ❌ TEHLİKELİ
string yol = Path.Combine(klasor, dosya.FileName);
```

Kullanıcı dosya adını `../../../Program.cs` yaparsa ne olur? Bu saldırıya **path traversal** denir.

```csharp
// ✅ GÜVENLİ — adı biz üretiyoruz
string ad = $"{Guid.NewGuid():N}.jpg";
```

`Guid.NewGuid()` benzersiz rastgele bir değer üretir. Kullanıcının verdiği adın hiçbir parçası kullanılmaz.

---

## ⌨️ Adım 1: Yükleme servisi

`Data/GorselYardimcisi.cs`:

```csharp
namespace KisiselBlog.Data;

/// <summary>
/// Görsel yükleme ve silme.
///
/// Dört katmanlı kontrol:
///   1. Boyut sınırı
///   2. Uzantı beyaz listesi
///   3. MIME tipi kontrolü
///   4. Dosya imzası (magic bytes)
/// </summary>
public class GorselYardimcisi
{
    private readonly IWebHostEnvironment _ortam;
    private readonly ILogger<GorselYardimcisi> _gunluk;

    // En fazla 3 MB
    private const long EnFazlaBoyut = 3 * 1024 * 1024;

    // ⭐ BEYAZ LİSTE — "şunlar yasak" değil, "sadece şunlar serbest"
    //    Kara liste yaklaşımı her zaman eksik kalır.
    private static readonly string[] IzinliUzantilar =
        { ".jpg", ".jpeg", ".png", ".webp", ".gif" };

    private static readonly string[] IzinliMimeTipleri =
        { "image/jpeg", "image/png", "image/webp", "image/gif" };

    public GorselYardimcisi(IWebHostEnvironment ortam, ILogger<GorselYardimcisi> gunluk)
    {
        _ortam = ortam;
        _gunluk = gunluk;
    }

    /// <summary>Yükleme sonucunu taşır: başarı, yol ve hata birlikte.</summary>
    public record YuklemeSonucu(bool Basarili, string? Yol, string? Hata);

    /// <summary>
    /// Görseli doğrular ve kaydeder.
    /// Başarılıysa web yolunu döner ("/yuklenen/2026-10/abc.jpg").
    /// </summary>
    public async Task<YuklemeSonucu> KaydetAsync(IFormFile? dosya)
    {
        if (dosya == null || dosya.Length == 0)
            return new YuklemeSonucu(false, null, "Dosya seçilmedi.");

        // ── 1) BOYUT ────────────────────────────────────────
        if (dosya.Length > EnFazlaBoyut)
            return new YuklemeSonucu(false, null,
                $"Dosya çok büyük. En fazla {EnFazlaBoyut / 1024 / 1024} MB olabilir.");

        // ── 2) UZANTI ───────────────────────────────────────
        //
        // ⚠️ ToLowerInvariant kullanıyoruz. Bu örnekte ToLower() de
        //    aynı sonucu verirdi (".JPG" içinde 'I' harfi yok), ama
        //    uzantı karşılaştırması PROGRAMATİK bir karşılaştırmadır.
        //    Modül 2'deki kural: makine için invariant, insan için kültür.
        string uzanti = Path.GetExtension(dosya.FileName).ToLowerInvariant();

        if (!IzinliUzantilar.Contains(uzanti))
            return new YuklemeSonucu(false, null,
                $"Sadece şu biçimler yüklenebilir: {string.Join(", ", IzinliUzantilar)}");

        // ── 3) MIME TİPİ ────────────────────────────────────
        //
        // Tarayıcının bildirdiği tip. Kolayca sahte olabilir
        // ama bedava bir kontrol katmanı.
        if (!IzinliMimeTipleri.Contains(dosya.ContentType.ToLowerInvariant()))
            return new YuklemeSonucu(false, null, "Dosya türü desteklenmiyor.");

        // ── 4) ⭐ DOSYA İMZASI ──────────────────────────────
        //
        // Asıl kontrol bu. Uzantı ve MIME yalan söyleyebilir,
        // dosyanın ilk baytları söyleyemez.
        if (!await ImzaGecerliMiAsync(dosya))
        {
            _gunluk.LogWarning("Geçersiz dosya imzası: {ad}", dosya.FileName);
            return new YuklemeSonucu(false, null, "Dosya geçerli bir görsel değil.");
        }

        // ── 5) KAYDET ───────────────────────────────────────
        //
        // ⭐ Dosya adını BİZ üretiyoruz.
        //    Kullanıcının verdiği adın hiçbir parçası kullanılmıyor
        //    → path traversal imkânsız.
        string yeniAd = $"{Guid.NewGuid():N}{uzanti}";

        // Tarihe göre klasörle — tek klasörde 10.000 dosya olmasın
        string altKlasor = DateTime.Now.ToString("yyyy-MM");
        string klasorYolu = Path.Combine(_ortam.WebRootPath, "yuklenen", altKlasor);

        Directory.CreateDirectory(klasorYolu);   // varsa dokunmaz

        string tamYol = Path.Combine(klasorYolu, yeniAd);

        using (var akis = new FileStream(tamYol, FileMode.Create))
        {
            await dosya.CopyToAsync(akis);
        }

        // Web'den erişilecek yol — ters eğik çizgi DEĞİL
        return new YuklemeSonucu(true, $"/yuklenen/{altKlasor}/{yeniAd}", null);
    }

    /// <summary>
    /// Dosyanın ilk baytlarını okuyup gerçekten görsel olup
    /// olmadığını kontrol eder.
    /// </summary>
    private static async Task<bool> ImzaGecerliMiAsync(IFormFile dosya)
    {
        // İlk 12 bayt tüm biçimleri ayırt etmeye yeter
        byte[] tampon = new byte[12];

        using (var akis = dosya.OpenReadStream())
        {
            int okunan = await akis.ReadAsync(tampon, 0, tampon.Length);

            if (okunan < 4)
                return false;      // dosya çok küçük, görsel olamaz
        }

        // JPEG:  FF D8 FF
        if (tampon[0] == 0xFF && tampon[1] == 0xD8 && tampon[2] == 0xFF)
            return true;

        // PNG:   89 50 4E 47
        if (tampon[0] == 0x89 && tampon[1] == 0x50 &&
            tampon[2] == 0x4E && tampon[3] == 0x47)
            return true;

        // GIF:   47 49 46 38   ("GIF8")
        if (tampon[0] == 0x47 && tampon[1] == 0x49 &&
            tampon[2] == 0x46 && tampon[3] == 0x38)
            return true;

        // WEBP:  52 49 46 46 ("RIFF") ... 57 45 42 50 ("WEBP")
        if (tampon.Length >= 12 &&
            tampon[0] == 0x52 && tampon[1] == 0x49 &&
            tampon[2] == 0x46 && tampon[3] == 0x46 &&
            tampon[8] == 0x57 && tampon[9] == 0x45 &&
            tampon[10] == 0x42 && tampon[11] == 0x50)
            return true;

        return false;
    }

    /// <summary>
    /// Görseli diskten siler.
    ///
    /// ⚠️ Yol kontrolü yapıyoruz — "/yuklenen/" ile başlamayan
    ///    hiçbir şeyi silmiyoruz. Bu kontrol olmasaydı
    ///    veritabanındaki bozuk bir yol sistem dosyası sildirebilirdi.
    /// </summary>
    public void Sil(string? webYolu)
    {
        if (string.IsNullOrWhiteSpace(webYolu))
            return;

        if (!webYolu.StartsWith("/yuklenen/", StringComparison.Ordinal))
        {
            _gunluk.LogWarning("Şüpheli silme yolu reddedildi: {yol}", webYolu);
            return;
        }

        try
        {
            // Web yolunu disk yoluna çevir: /yuklenen/x/y.jpg → wwwroot\yuklenen\x\y.jpg
            string gorece = webYolu.TrimStart('/').Replace('/', Path.DirectorySeparatorChar);
            string tamYol = Path.Combine(_ortam.WebRootPath, gorece);

            if (System.IO.File.Exists(tamYol))
                System.IO.File.Delete(tamYol);
        }
        catch (Exception ex)
        {
            // Silme başarısız olursa uygulamayı çökertme —
            // yetim dosya kalır, o kadar.
            _gunluk.LogError(ex, "Görsel silinemedi: {yol}", webYolu);
        }
    }
}
```

### 📖 Niye `record` döndürüyoruz?

Metodun üç şey söylemesi gerekiyor: **oldu mu**, **nereye kaydedildi**, **olmadıysa niye**. `string?` döndürüp `null`'ı "hata" saymak bu üçünden ikisini kaybettirir — kullanıcıya "dosya çok büyük" mü yoksa "geçersiz biçim" mi diyeceğinizi bilemezsiniz.

```csharp
public record YuklemeSonucu(bool Basarili, string? Yol, string? Hata);
```

`record`, birkaç değeri bir arada döndürmenin en kısa yolu. Tek satırda sınıf, yapıcı metot, `Equals` ve `ToString` birden gelir.

Controller'daki kullanımı:

```csharp
var sonuc = await _gorsel.KaydetAsync(kapakDosya);

if (!sonuc.Basarili)
    ModelState.AddModelError("KapakDosya", sonuc.Hata!);
else
    yazi.KapakGorsel = sonuc.Yol;
```

> **Alternatifler:** `out` parametresi (`async` metotlarda **kullanılamaz**), özel durum fırlatmak (doğrulama hatası için fazla ağır), tuple döndürmek (`(bool, string?, string?)` — alan adları kaybolur). `record` üçünden de temiz.

---


## ⌨️ Adım 2: Servisi kaydet

`Program.cs`:

```csharp
builder.Services.AddScoped<GorselYardimcisi>();
```

Ve klasörü oluşturun:

```
wwwroot/yuklenen/
```

> ⚠️ Git kullanıyorsanız bu klasöre bir `.gitkeep` dosyası koyun — boş klasörler git'e girmez.
>
> `.gitignore`'a da şunu ekleyin ki yüklenen görseller depoya girmesin:
> ```
> wwwroot/yuklenen/*
> !wwwroot/yuklenen/.gitkeep
> ```

---

## ⌨️ Adım 3: Model

`Models/Yazi.cs`'e ekleyin:

```csharp
    /// <summary>
    /// Formdan gelen dosya. Veritabanına YAZILMAZ —
    /// işlendikten sonra KapakGorsel alanına yol yazılır.
    /// </summary>
    [NotMapped]
    [Display(Name = "Kapak görseli")]
    public IFormFile? KapakDosya { get; set; }

    /// <summary>Formdan gelen "görseli kaldır" işareti.</summary>
    [NotMapped]
    public bool KapakKaldir { get; set; }
```

> `[NotMapped]` şart. Olmazsa EF `IFormFile` tipini sütuna çevirmeye çalışır ve migration hata verir.

---

## ⌨️ Adım 4: Controller

`YaziYonetimController`'ı güncelleyin.

### Yapıcı metot

```csharp
    private readonly BlogDbContext _db;
    private readonly GorselYardimcisi _gorsel;

    public YaziYonetimController(BlogDbContext db, GorselYardimcisi gorsel)
    {
        _db = db;
        _gorsel = gorsel;
    }
```

### Oluştur (POST)

`ModelState.IsValid` kontrolünden **sonra**, `_db.Yazilar.Add` öncesine:

```csharp
        // ── Kapak görseli ───────────────────────────────────
        if (yazi.KapakDosya != null && yazi.KapakDosya.Length > 0)
        {
            var sonuc = await _gorsel.KaydetAsync(yazi.KapakDosya);

            if (!sonuc.Basarili)
            {
                ModelState.AddModelError(nameof(Yazi.KapakDosya), sonuc.Hata!);
                await EtiketleriHazirlaAsync();
                return View(yazi);
            }

            yazi.KapakGorsel = sonuc.Yol;
        }
```

### Düzenle (POST)

`mevcut.UpdatedDate = DateTime.Now;` satırından sonra:

```csharp
        // ── Kapak görseli ───────────────────────────────────
        if (gelen.KapakKaldir)
        {
            // Kullanıcı "kaldır" dedi
            _gorsel.Sil(mevcut.KapakGorsel);
            mevcut.KapakGorsel = null;
        }
        else if (gelen.KapakDosya != null && gelen.KapakDosya.Length > 0)
        {
            var sonuc = await _gorsel.KaydetAsync(gelen.KapakDosya);

            if (!sonuc.Basarili)
            {
                ModelState.AddModelError(nameof(Yazi.KapakDosya), sonuc.Hata!);
                await EtiketleriHazirlaAsync();
                return View(gelen);
            }

            // ⭐ Yeni görsel kaydedildikten SONRA eskisini sil.
            //    Sıra önemli: önce silseydik ve yükleme başarısız
            //    olsaydı, hem eski hem yeni görsel kaybolurdu.
            string? eski = mevcut.KapakGorsel;
            mevcut.KapakGorsel = sonuc.Yol;
            _gorsel.Sil(eski);
        }
```

### Sil (POST)

```csharp
    [HttpPost("sil/{id:long}")]
    [ValidateAntiForgeryToken]
    public async Task<IActionResult> SilOnayla(long id)
    {
        var yazi = await _db.Yazilar.FindAsync(id);

        if (yazi == null)
            return NotFound();

        yazi.AktifMi = false;
        yazi.UpdatedDate = DateTime.Now;
        await _db.SaveChangesAsync();

        // ⚠️ Görseli SİLMİYORUZ.
        //    Yazı soft delete — geri getirilebilir.
        //    Görseli silseydik geri gelen yazının kapağı kaybolurdu.
        //
        //    Yetim dosyalar birikir; temizlik için ayrı bir
        //    bakım işi yazılır (Alıştırma 4).

        TempData["Basarili"] = "Yazı silindi.";
        return RedirectToAction(nameof(Index));
    }
```

---

## ⌨️ Adım 5: Form

### ⚠️ `enctype` olmadan dosya gitmez

```html
<!-- ❌ Dosya alanı boş gelir, hata da vermez -->
<form method="post">

<!-- ✅ -->
<form method="post" enctype="multipart/form-data">
```

Bu, dosya yüklemede **en sık yapılan hata**. Form sessizce çalışır, `IFormFile` hep `null` gelir, saatlerce aranır.

`Olustur.cshtml` ve `Duzenle.cshtml`'de form etiketini güncelleyin:

```html
<form asp-action="Olustur" method="post" enctype="multipart/form-data">
```

```html
<form asp-action="Duzenle" asp-route-id="@Model.YaziId"
      method="post" enctype="multipart/form-data">
```

### Form alanı

`Views/Yonetim/YaziYonetim/_Form.cshtml`'de, sağ paneldeki Durum alanının altına:

```html
<div class="mb-3">
    <label asp-for="KapakDosya" class="form-label"></label>

    @if (!string.IsNullOrEmpty(Model.KapakGorsel))
    {
        <div class="mb-2">
            <img src="@Model.KapakGorsel" alt="Kapak görseli"
                 class="img-fluid rounded border" />

            <div class="form-check mt-2">
                <input asp-for="KapakKaldir" type="checkbox" class="form-check-input" />
                <label asp-for="KapakKaldir" class="form-check-label small text-danger">
                    Görseli kaldır
                </label>
            </div>
        </div>
    }

    <input asp-for="KapakDosya" type="file" class="form-control form-control-sm"
           accept="image/jpeg,image/png,image/webp,image/gif" />

    <span asp-validation-for="KapakDosya" class="text-danger small"></span>

    <div class="form-text">
        JPG, PNG, WEBP veya GIF · en fazla 3 MB
    </div>
</div>
```

> `accept` özniteliği tarayıcının dosya seçme penceresini filtreler. **Güvenlik önlemi değil** — sadece kolaylık. Gerçek kontrol sunucuda.

---

## ⌨️ Adım 6: Görseli göster

### Yazı listesinde

`Views/Shared/_YaziKarti.cshtml`'de, başlık `<h2>` etiketinin **üstüne**:

```html
@if (!string.IsNullOrEmpty(Model.KapakGorsel))
{
    <a asp-controller="Yazi" asp-action="Detay" asp-route-slug="@Model.Slug"
       class="d-block mb-3">
        <img src="@Model.KapakGorsel" alt="" loading="lazy"
             class="img-fluid rounded w-100"
             style="max-height:14rem; object-fit:cover;" />
    </a>
}
```

> ⭐ Tek dosyaya ekliyoruz ve görsel **üç** yerde birden çıkıyor: ana sayfa, etiket
> sayfası, arama sonuçları. Modül 6'da kartı kısmi görünüme almanın karşılığı bu.

### 📖 Üç küçük ama önemli ayrıntı

**`loading="lazy"`** — görsel ekrana gelene kadar indirilmez. 20 yazılık bir listede büyük hız farkı.

**`object-fit: cover`** — görseli kutuya **kırparak** sığdırır, oranını bozmaz. `fill` olsaydı ezilmiş görünürdü.

**`alt=""`** — görsel dekoratif. Ekran okuyucu atlasın. Eğer görsel **bilgi taşıyorsa** `alt` doldurulmalı:

```html
<img src="..." alt="2026 yılı satış grafiği, mart ayında zirve yapıyor" />
```

> **Kural:** `alt` yazmayı unutmak ≠ `alt=""` yazmak. İlki hata, ikincisi bilinçli karar.

### Yazı detayında

Modül 6'da zaten eklemiştik:

```html
@if (!string.IsNullOrEmpty(Model.KapakGorsel))
{
    <img src="@Model.KapakGorsel" alt=""
         class="img-fluid rounded mb-4" />
}
```

### Paylaşım önizlemesi

`_Layout.cshtml`'in `<head>` kısmına, diğer `og:` etiketlerinin yanına:

```html
@if (ViewData["Gorsel"] is string gorselYolu && !string.IsNullOrEmpty(gorselYolu))
{
    var tamAdres = $"{Context.Request.Scheme}://{Context.Request.Host}{gorselYolu}";
    <meta property="og:image" content="@tamAdres" />
    <meta name="twitter:card" content="summary_large_image" />
}
```

`YaziController.Detay`'da:

```csharp
        ViewData["Gorsel"] = yazi.KapakGorsel;
```

> `og:image` **tam adres** ister (`https://...`), göreceli yol kabul etmez. Bu yüzden `Scheme` ve `Host`'u birleştiriyoruz.

---

## ⌨️ Adım 7: Yükleme boyutu sınırı

ASP.NET Core varsayılan olarak 28.6 MB'a izin verir. Biz 3 MB istiyoruz.

`YaziYonetimController`'daki POST metotlarına:

```csharp
    [HttpPost("olustur")]
    [ValidateAntiForgeryToken]
    [RequestSizeLimit(4 * 1024 * 1024)]      // ⭐ 4 MB (3 MB dosya + form verisi)
    public async Task<IActionResult> Olustur(Yazi yazi)
```

> **Neden 4 MB?** Dosya 3 MB olsa bile form verisi (başlık, içerik, etiketler) ve `multipart` ek yükü biraz daha yer kaplar. Pay bırakmak gerekir.

IIS veya Kestrel seviyesinde de sınır vardır; `appsettings.json`'a ekleyebilirsiniz:

```json
"Kestrel": {
  "Limits": {
    "MaxRequestBodySize": 4194304
  }
}
```

---

## ▶️ Test

| # | Test | Beklenen |
|---|---|---|
| 1 | Yazıya JPG yükle | Kaydediliyor, önizleme çıkıyor |
| 2 | `wwwroot/yuklenen/2026-10/` | Dosya orada, adı GUID |
| 3 | SSMS: `KapakGorsel` | `/yuklenen/2026-10/abc....jpg` |
| 4 | Ana sayfa | Görsel listede görünüyor |
| 5 | Yazı detayı | Görsel üstte |
| 6 | Yeni görsel yükle | Eski dosya **silindi** |
| 7 | "Görseli kaldır" | Dosya silindi, alan NULL |
| 8 | 5 MB dosya | "Dosya çok büyük" |
| 9 | **PDF yükle** | ⬇ aşağıya bakın |
| 10 | **Sahte uzantı** | ⬇ aşağıya bakın |
| 11 | `enctype` sil, dene | Dosya gelmiyor (hata da yok) |

### 9. PDF testi

Bir PDF'in adını `resim.jpg` yapıp yükleyin.

**Beklenen:** "Dosya geçerli bir görsel değil."

Uzantı `.jpg`, MIME tipi tarayıcıdan `image/jpeg` gelebilir — ama **imza** `%PDF` (`25 50 44 46`). İmza kontrolü yakalar.

### 10. Sahte uzantı testi

Bir metin dosyası oluşturun, içine şunu yazın:

```
<%@ Page Language="C#" %>
<% Response.Write("Bu bir saldırı olabilirdi"); %>
```

Adını `resim.jpg` yapıp yükleyin.

**Beklenen:** Reddedilir. İmza kontrolü geçilemez.

> **Bu iki testi mutlaka yaptırın.** Uzantı kontrolünün neden yetmediği ancak böyle anlaşılır.

---

## ⚠️ Sık yapılan hatalar

| Hata | Sebep | Çözüm |
|---|---|---|
| `IFormFile` hep null | `enctype` yok | `enctype="multipart/form-data"` |
| Migration hatası | `[NotMapped]` yok | `IFormFile` alanına ekle |
| `DirectoryNotFoundException` | Klasör yok | `Directory.CreateDirectory` |
| Görsel 404 | Yol ters eğik çizgili | `/` kullan, `\` değil |
| 413 hatası | Boyut sınırı aşıldı | `RequestSizeLimit` ayarla |
| Eski görseller birikiyor | Silme yapılmıyor | Yeni kaydettikten sonra sil |
| PDF yüklenebiliyor | İmza kontrolü yok | `ImzaGecerliMiAsync` ekle |
| Görsel ezik görünüyor | `object-fit` yok | `cover` kullan |
| Liste yavaş | `loading="lazy"` yok | Ekle |
| Paylaşımda görsel çıkmıyor | `og:image` göreceli | Tam adres yap |

---

## ✏️ Alıştırmalar

1. Görsel yüklemeyi tamamlayın, 11 test senaryosunu geçirin — özellikle 9 ve 10'u.

2. **İmza kontrolünü kaldırın**, PDF'i `.jpg` adıyla yükleyin. Kaydedildiğini görün. Sonra **geri koyun.**

3. **Görsel boyutlandırma ekleyin:** Yüklenen görsel 1600px'ten genişse küçültülsün.
   İpucu: `SixLabors.ImageSharp` paketi.
   ```bash
   dotnet add package SixLabors.ImageSharp
   ```

4. **Yetim dosya temizleyici:** Veritabanında referansı olmayan dosyaları bulup silen bir panel sayfası.
   İpucu: Disk dosyalarını listele, `Yazilar.Select(y => y.KapakGorsel)` ile karşılaştır.
   ⚠️ `IgnoreQueryFilters()` — silinmiş yazıların görselleri de referanslı sayılmalı.

5. Yazı içine görsel eklemek için bir yükleme aracı yapın: panelde yükle, Markdown kodunu (`![](yol)`) kopyalanabilir göster.

**Düşünme soruları:**
- Dosya adını `Guid` ile üretiyoruz. Orijinal adı veritabanında saklasak ne kazanırdık, ne riske girerdik?
- Yazı silinince görseli silmiyoruz. Bu doğru mu? Hangi durumda yanlış olurdu?
- İmza kontrolü ilk 12 baytı okuyor. Saldırgan dosyanın başına geçerli bir JPEG imzası koyup devamına zararlı kod yazsa ne olurdu? Bu neden yine de sorun değil?

---

👉 Sonraki: [`11-son-dokunuslar.md`](11-son-dokunuslar.md)
