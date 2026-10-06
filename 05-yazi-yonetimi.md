# Modül 5 — Yazı Yönetimi ve Markdown

**Süre:** 2 ders saati

---

## 🎯 Bu derste ne yapacağız

Yazı CRUD'u. **İki yeni konu:**

1. **Markdown** — içeriği nasıl saklayıp nasıl gösteriyoruz
2. **Slug üretimi** — Modül 2'deki yardımcıyı gerçek kullanıma sokmak

> CRUD kalıbı tanıdık. Hızlı geçeceğiz, Markdown'da duracağız.

---

## 📖 İçerik nasıl saklanmalı?

**Sınıfa sor:** *"Blog yazısında kalın yazı, başlık, liste, bağlantı olacak. Bunları veritabanında nasıl saklayalım?"*

### Üç seçenek

| Yol | Nasıl | Sorun |
|---|---|---|
| **Düz metin** | Sadece yazı | Biçimlendirme yok |
| **HTML** | `<p>Merhaba <b>dünya</b></p>` | ⚠️ XSS riski, elle yazması zor |
| **Markdown** | `Merhaba **dünya**` | ⭐ Okunabilir, güvenli, taşınabilir |

### Neden Markdown?

```markdown
## Başlık

Bu bir **kalın** ve bu *italik* yazı.

- Madde bir
- Madde iki

[Bağlantı](https://ornek.com)
```

Bu metin:
- **Ham hâliyle bile okunabilir** — biçimlendirme gürültü yapmıyor
- Yazması kolay — HTML etiketi ezberlemek gerekmiyor
- Taşınabilir — yarın başka sisteme geçerseniz metin elinizde kalır
- Güvenli — HTML'e çevirirken kontrolü biz elinde tutuyoruz

> **Sınıfa sor:** *"Aynı içeriği HTML olarak saklasaydık ve siteyi yenilesek ne olurdu?"*
> HTML, eski sitenin sınıflarına ve yapısına bağlı olurdu. Markdown nötrdür.

### Ne zaman çevireceğiz?

| Yol | Avantaj | Dezavantaj |
|---|---|---|
| **Kaydederken** çevir, HTML'i de sakla | Okuma hızlı | İki kopya, senkron sorunu |
| **Okurken** çevir ⭐ | Tek doğruluk kaynağı | Her istekte işlem |

Biz **okurken** çevireceğiz. Çevirme işlemi milisaniyeler sürer; blog trafiğinde sorun değil.

> Site çok büyürse önbellek (cache) eklenir — ama "önce çalışsın, sonra hızlansın".

---

## ⌨️ Adım 1: Markdig kurulumu

```bash
dotnet add package Markdig
```

`Data/MarkdownYardimcisi.cs`:

```csharp
using Markdig;

namespace KisiselBlog.Data;

/// <summary>
/// Markdown metnini HTML'e çevirir.
/// </summary>
public static class MarkdownYardimcisi
{
    /// <summary>
    /// ⭐ Pipeline bir kez kurulup tekrar tekrar kullanılır.
    ///    Her çağrıda yeniden kurmak israf olurdu.
    ///
    /// static readonly: uygulama ömrü boyunca tek nesne.
    /// </summary>
    private static readonly MarkdownPipeline Boru = new MarkdownPipelineBuilder()

        // ── Markdig'in ek özellikleri ────────────────────────
        //
        // UseAdvancedExtensions şunları açar:
        //   • Tablolar       | a | b |
        //   • Üstü çizili    ~~metin~~
        //   • Görev listesi  - [x] madde
        //   • Otomatik bağlantı (çıplak URL'ler tıklanabilir olur)
        //   • Dipnotlar
        .UseAdvancedExtensions()

        // Başlıklara otomatik id ver → /yazi/slug#bolum-adi
        // çalışsın, içindekiler listesi yapılabilsin
        .UseAutoIdentifiers()

        // ⭐⭐ GÜVENLİK: Markdown içindeki HAM HTML'i devre dışı bırak
        //
        //    Bu olmadan içeriğe <script>alert(1)</script> yazılabilir
        //    ve sayfada ÇALIŞIR.
        //
        //    "Ama içeriği sadece ben yazıyorum" — doğru, bu projede
        //    risk düşük. Ama alışkanlık olarak kapalı tutuyoruz:
        //      • Yarın yorum veya çok yazarlı yapı eklenebilir
        //      • Panel şifresi ele geçerse zarar sınırlı kalır
        //      • Varsayılan güvenli olmalı
        .DisableHtml()

        .Build();

    /// <summary>Markdown → HTML</summary>
    public static string Cevir(string? markdown)
    {
        if (string.IsNullOrWhiteSpace(markdown))
            return "";

        return Markdown.ToHtml(markdown, Boru);
    }

    /// <summary>
    /// Markdown → düz metin.
    /// Özet üretmek ve paylaşım açıklaması için.
    /// </summary>
    public static string DuzMetin(string? markdown, int enFazla = 160)
    {
        if (string.IsNullOrWhiteSpace(markdown))
            return "";

        string duz = Markdown.ToPlainText(markdown, Boru)
            .Replace("\r", " ")
            .Replace("\n", " ")
            .Trim();

        // Art arda boşlukları teke indir
        while (duz.Contains("  "))
            duz = duz.Replace("  ", " ");

        if (duz.Length <= enFazla)
            return duz;

        // Kelimeyi ortadan bölme — son boşluktan kes
        string kesilmis = duz.Substring(0, enFazla);
        int sonBosluk = kesilmis.LastIndexOf(' ');

        if (sonBosluk > 0)
            kesilmis = kesilmis.Substring(0, sonBosluk);

        return kesilmis + "…";
    }
}
```

### ⚠️ `.DisableHtml()` — bu satırı anlatın

**Deney (dikkatli yapın):** Bu satırı geçici olarak kaldırın, bir yazının içeriğine şunu yazın:

```html
<script>alert('Bu bir XSS saldırısı olabilirdi');</script>
```

Yazıyı görüntüleyin — uyarı kutusu çıkar.

**Sınıfa sor:** *"Bu sitede içeriği sadece ben yazıyorum. Risk var mı?"*

Şu an yok. Ama:
- Yarın yorum özelliği eklenirse
- Panel şifresi ele geçerse
- Çok yazarlı hâle gelirse

...bu kapı açık kalmış olur. **Varsayılan güvenli olmalı, sonradan gevşetilmeli** — tersi değil.

Satırı geri koyun.

---

## 📖 Markdown'ı sayfaya basmak: `Html.Raw`

Markdig HTML metni döndürür. Razor **varsayılan olarak HTML'i kaçırır** (escape eder):

```html
@markdownSonucu
@* → ekranda &lt;p&gt;Merhaba&lt;/p&gt; görünür — etiketler metin olur *@
```

Bunu engellemek için `Html.Raw` gerekir:

```html
@Html.Raw(markdownSonucu)
@* → gerçek <p> etiketi olarak işlenir *@
```

> ⚠️ **`Html.Raw` tehlikeli bir araçtır.** "Bu metni olduğu gibi tarayıcıya ver, hiç kontrol etme" demektir.
>
> **Kural:** `Html.Raw`'ı sadece **kaynağına güvendiğiniz** ve **temizlendiğini bildiğiniz** içerikte kullanın.
>
> Bizim durumumuzda: içerik Markdig'den geçiyor ve `DisableHtml()` ham HTML'i engelliyor. Bu yüzden güvenli.
>
> **Asla:** Kullanıcıdan gelen ham metni doğrudan `Html.Raw` ile basmayın.

---

## ⌨️ Adım 2: Controller

`Controllers/YaziYonetimController.cs`:

```csharp
using Microsoft.AspNetCore.Mvc;
using Microsoft.EntityFrameworkCore;
using KisiselBlog.Data;
using KisiselBlog.Models;

namespace KisiselBlog.Controllers;

// ⭐ Taban sınıftaki [Route("yonetim/[controller]")] yerine
//    kendi temiz yolumuzu veriyoruz: /yonetim/yazi
// ⭐ [Authorize] taban sınıftan geliyor — tekrar yazmıyoruz.
[Route("yonetim/yazi")]
public class YaziYonetimController : YonetimControllerTabani
{
    private readonly BlogDbContext _db;

    public YaziYonetimController(BlogDbContext db)
    {
        _db = db;
    }

    // ══════════════════════════════════════════════════════
    //  YARDIMCI: etiket listesi
    // ══════════════════════════════════════════════════════
    private async Task EtiketleriHazirlaAsync()
    {
        ViewBag.TumEtiketler = await _db.Etiketler
            .AsNoTracking()
            .OrderBy(e => e.Ad)
            .ToListAsync();
    }

    // ══════════════════════════════════════════════════════
    //  YARDIMCI: benzersiz slug üret
    //
    //  ⚠️⚠️ IgnoreQueryFilters() ŞART!
    //     Silinmiş (AktifMi = 0) yazılar veritabanında DURUYOR
    //     ve benzersiz indeks onları da sayıyor.
    //     Filtre açık kalsaydı çakışmayı göremez, kayıt
    //     anında DbUpdateException alırdık.
    // ══════════════════════════════════════════════════════
    private string SlugUret(string baslik, long haricTutulanId = 0)
    {
        return SlugYardimcisi.BenzersizUret(baslik, aday =>
            _db.Yazilar
               .IgnoreQueryFilters()
               .Any(y => y.Slug == aday && y.YaziId != haricTutulanId));
    }

    // ══════════════════════════════════════════════════════
    //  1) LİSTE
    //  GET: /yonetim/yazi?durum=Taslak&arama=...
    // ══════════════════════════════════════════════════════
    [HttpGet("")]
    public async Task<IActionResult> Index(YaziDurum? durum, string? arama)
    {
        ViewData["Title"] = "Yazılar";

        IQueryable<Yazi> sorgu = _db.Yazilar
            .AsNoTracking()
            .Include(y => y.Etiketler);

        if (durum.HasValue)
            sorgu = sorgu.Where(y => y.Durum == durum.Value);

        if (!string.IsNullOrWhiteSpace(arama))
        {
            string temiz = arama.Trim();
            sorgu = sorgu.Where(y => y.Baslik.Contains(temiz)
                                  || y.Icerik.Contains(temiz));
        }

        var liste = await sorgu
            // ⭐ Taslaklar üstte dursun — üzerinde çalışılan onlar
            .OrderBy(y => y.Durum == YaziDurum.Yayinda)
            .ThenByDescending(y => y.UpdatedDate ?? y.CreatedDate)
            .ToListAsync();

        ViewBag.SeciliDurum = durum;
        ViewBag.Arama = arama;

        return View(liste);
    }

    // ══════════════════════════════════════════════════════
    //  2) OLUŞTURMA FORMU
    //  GET: /yonetim/yazi/olustur
    // ══════════════════════════════════════════════════════
    [HttpGet("olustur")]
    public async Task<IActionResult> Olustur()
    {
        ViewData["Title"] = "Yeni yazı";
        await EtiketleriHazirlaAsync();
        return View(new Yazi());
    }

    // ══════════════════════════════════════════════════════
    //  3) OLUŞTUR
    //  POST: /yonetim/yazi/olustur
    // ══════════════════════════════════════════════════════
    [HttpPost("olustur")]
    [ValidateAntiForgeryToken]
    public async Task<IActionResult> Olustur(Yazi yazi)
    {
        ViewData["Title"] = "Yeni yazı";

        if (!ModelState.IsValid)
        {
            await EtiketleriHazirlaAsync();
            return View(yazi);
        }

        // ⭐ Slug'ı BİZ üretiyoruz — kullanıcı formda görmüyor
        yazi.Slug = SlugUret(yazi.Baslik);

        yazi.CreatedDate = DateTime.Now;
        yazi.UpdatedDate = null;
        yazi.AktifMi = true;
        yazi.OkunmaSayisi = 0;

        // ⭐ Yayın tarihi: sadece yayınlanıyorsa dolsun
        yazi.YayinTarihi = yazi.Durum == YaziDurum.Yayinda
            ? DateTime.Now
            : null;

        await EtiketleriBaglaAsync(yazi, yazi.SeciliEtiketIdler);

        _db.Yazilar.Add(yazi);
        await _db.SaveChangesAsync();

        TempData["Basarili"] = yazi.Durum == YaziDurum.Yayinda
            ? "Yazı yayınlandı."
            : "Taslak kaydedildi.";

        return RedirectToAction(nameof(Duzenle), new { id = yazi.YaziId });
    }

    // ══════════════════════════════════════════════════════
    //  4) DÜZENLEME FORMU
    //  GET: /yonetim/yazi/duzenle/5
    // ══════════════════════════════════════════════════════
    [HttpGet("duzenle/{id:long}")]
    public async Task<IActionResult> Duzenle(long id)
    {
        // ⭐ Include ŞART — etiketleri de çekmeliyiz ki
        //    formda hangilerinin işaretli geleceğini bilelim
        var yazi = await _db.Yazilar
            .Include(y => y.Etiketler)
            .FirstOrDefaultAsync(y => y.YaziId == id);

        if (yazi == null)
            return NotFound();

        yazi.SeciliEtiketIdler = yazi.Etiketler.Select(e => e.EtiketId).ToList();

        ViewData["Title"] = "Yazıyı düzenle";
        await EtiketleriHazirlaAsync();
        return View(yazi);
    }

    // ══════════════════════════════════════════════════════
    //  5) DÜZENLE
    //  POST: /yonetim/yazi/duzenle/5
    // ══════════════════════════════════════════════════════
    [HttpPost("duzenle/{id:long}")]
    [ValidateAntiForgeryToken]
    public async Task<IActionResult> Duzenle(long id, Yazi gelen)
    {
        ViewData["Title"] = "Yazıyı düzenle";

        if (!ModelState.IsValid)
        {
            await EtiketleriHazirlaAsync();
            return View(gelen);
        }

        var mevcut = await _db.Yazilar
            .Include(y => y.Etiketler)
            .FirstOrDefaultAsync(y => y.YaziId == id);

        if (mevcut == null)
            return NotFound();

        // ⭐ Taslaktan yayına mı geçiyor? Tarihi o zaman doldur.
        bool oncedenYayindaydi = mevcut.Durum == YaziDurum.Yayinda;
        bool simdiYayinda = gelen.Durum == YaziDurum.Yayinda;

        mevcut.Baslik      = gelen.Baslik;
        mevcut.Ozet        = gelen.Ozet;
        mevcut.Icerik      = gelen.Icerik;
        mevcut.Durum       = gelen.Durum;
        mevcut.UpdatedDate = DateTime.Now;

        if (simdiYayinda && !oncedenYayindaydi)
        {
            // İlk kez yayınlanıyor
            mevcut.YayinTarihi = DateTime.Now;
        }
        else if (!simdiYayinda)
        {
            // Yayından çekildi
            mevcut.YayinTarihi = null;
        }
        // Zaten yayındaydı ve hâlâ yayında → ESKİ TARİHİ KORU
        // (else bloğu yok — bilinçli)

        // ⚠️ SLUG'A DOKUNMUYORUZ
        //    Başlık değişse bile adres sabit kalır.
        //    Değiştirseydik paylaşılmış tüm bağlantılar kırılırdı.
        //    (Elle değiştirme seçeneği Alıştırma 4'te.)

        await EtiketleriBaglaAsync(mevcut, gelen.SeciliEtiketIdler);

        await _db.SaveChangesAsync();

        TempData["Basarili"] = "Yazı güncellendi.";
        return RedirectToAction(nameof(Duzenle), new { id });
    }

    // ══════════════════════════════════════════════════════
    //  6) HIZLI DURUM DEĞİŞTİR
    //  POST: /yonetim/yazi/durum/5
    // ══════════════════════════════════════════════════════
    [HttpPost("durum/{id:long}")]
    [ValidateAntiForgeryToken]
    public async Task<IActionResult> Durum(long id, YaziDurum yeniDurum)
    {
        var yazi = await _db.Yazilar.FindAsync(id);

        if (yazi == null)
            return NotFound();

        yazi.Durum = yeniDurum;
        yazi.UpdatedDate = DateTime.Now;

        if (yeniDurum == YaziDurum.Yayinda && yazi.YayinTarihi == null)
            yazi.YayinTarihi = DateTime.Now;

        await _db.SaveChangesAsync();

        TempData["Basarili"] = yeniDurum == YaziDurum.Yayinda
            ? "Yazı yayınlandı."
            : "Yazı taslağa alındı.";

        return RedirectToAction(nameof(Index));
    }

    // ══════════════════════════════════════════════════════
    //  7) SİLME ONAYI
    //  GET: /yonetim/yazi/sil/5
    // ══════════════════════════════════════════════════════
    [HttpGet("sil/{id:long}")]
    public async Task<IActionResult> Sil(long id)
    {
        var yazi = await _db.Yazilar
            .AsNoTracking()
            .FirstOrDefaultAsync(y => y.YaziId == id);

        if (yazi == null)
            return NotFound();

        ViewData["Title"] = "Yazıyı sil";
        return View(yazi);
    }

    // ══════════════════════════════════════════════════════
    //  8) SİL (soft delete)
    //  POST: /yonetim/yazi/sil/5
    // ══════════════════════════════════════════════════════
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

        TempData["Basarili"] = "Yazı silindi.";
        return RedirectToAction(nameof(Index));
    }

    // ══════════════════════════════════════════════════════
    //  YARDIMCI: etiketleri bağla
    //
    //  ⭐ Clear() + Add() → EF farkı kendisi hesaplar.
    //     Sadece gerçekten değişen bağlantılar için SQL üretir.
    //
    //  ⚠️ yazi nesnesi TAKİP EDİLİYOR olmalı ve Etiketler
    //     koleksiyonu YÜKLENMİŞ olmalı (Include).
    //     Yeni kayıtta ikisi de zaten sağlanıyor.
    // ══════════════════════════════════════════════════════
    private async Task EtiketleriBaglaAsync(Yazi yazi, List<long> seciliIdler)
    {
        yazi.Etiketler.Clear();

        if (seciliIdler == null || seciliIdler.Count == 0)
            return;

        var etiketler = await _db.Etiketler
            .Where(e => seciliIdler.Contains(e.EtiketId))
            .ToListAsync();

        foreach (var etiket in etiketler)
            yazi.Etiketler.Add(etiket);
    }
}
```

### ⚠️ Neden `RedirectToAction(nameof(Duzenle))`?

Kaydettikten sonra listeye değil, **yazının düzenleme sayfasına** dönüyoruz.

**Sor:** *"Neden?"*

Blog yazısı bir oturumda bitmez. Kaydet → devam et → kaydet → devam et. Her kaydetmede listeye atılsaydınız sinir bozucu olurdu.

> Bu, veri girişi uygulamalarıyla içerik uygulamaları arasındaki fark. Görev listesinde kaydet→liste doğruydu; blogda değil.

---

## ⌨️ Adım 3: Liste görünümü

`Views/Yonetim/YaziYonetim/Index.cshtml`:

```html
@model List<KisiselBlog.Models.Yazi>
@using KisiselBlog.Models
@{
    ViewData["Title"] = "Yazılar";
    var seciliDurum = ViewBag.SeciliDurum as YaziDurum?;
}

@section BaslikEylem {
    <a asp-action="Olustur" class="btn btn-primary">Yeni yazı</a>
}

@* ─── FİLTRE ─────────────────────────────────────────── *@
<form method="get" asp-action="Index" class="row g-2 align-items-end mb-4">

    <div class="col-sm">
        <label for="arama" class="form-label">Ara</label>
        <input type="text" id="arama" name="arama" value="@ViewBag.Arama"
               class="form-control" placeholder="Başlık veya içerik" />
    </div>

    <div class="col-sm-auto" style="min-width: 10rem;">
        <label for="durum" class="form-label">Durum</label>
        <select id="durum" name="durum" class="form-select">
            <option value="">Tümü</option>
            <option value="Taslak"  selected="@(seciliDurum == YaziDurum.Taslak)">Taslak</option>
            <option value="Yayinda" selected="@(seciliDurum == YaziDurum.Yayinda)">Yayında</option>
        </select>
    </div>

    <div class="col-sm-auto">
        <button type="submit" class="btn btn-primary">Filtrele</button>

        @if (seciliDurum != null || !string.IsNullOrEmpty(ViewBag.Arama as string))
        {
            <a asp-action="Index" class="btn btn-outline-secondary">Temizle</a>
        }
    </div>
</form>


@if (!Model.Any())
{
    <div class="text-center py-5 text-secondary">
        @if (seciliDurum != null || !string.IsNullOrEmpty(ViewBag.Arama as string))
        {
            <p>Filtrenize uyan yazı bulunamadı.</p>
            <p style="margin-top: 1rem;">
                <a asp-action="Index" class="btn btn-outline-secondary">Filtreleri temizle</a>
            </p>
        }
        else
        {
            <p>Henüz yazı yok.</p>
            <p style="margin-top: 1rem;">
                <a asp-action="Olustur" class="btn btn-primary">İlk yazını oluştur</a>
            </p>
        }
    </div>
}
else
{
    <div class="table-responsive">
    <table class="table table-hover align-middle">
        <thead>
            <tr>
                <th>Başlık</th>
                <th style="width: 7rem;">Durum</th>
                <th style="width: 9rem;">Tarih</th>
                <th class="text-center" style="width: 6rem;">Okunma</th>
                <th class="text-end" style="width: 13rem;">İşlem</th>
            </tr>
        </thead>
        <tbody>
            @foreach (var y in Model)
            {
                <tr>
                    <td>
                        <a asp-action="Duzenle" asp-route-id="@y.YaziId"
                           class="fw-semibold">@y.Baslik</a>

                        <div class="small text-secondary font-monospace">
                            /yazi/@y.Slug
                        </div>

                        @if (y.Etiketler.Any())
                        {
                            <div style="margin-top: .5rem;
                                        display:flex; gap:4px; flex-wrap:wrap;">
                                @foreach (var e in y.Etiketler.OrderBy(x => x.Ad))
                                {
                                    <span class="badge text-bg-light border fw-normal">@e.Ad</span>
                                }
                            </div>
                        }
                    </td>

                    <td>
                        @if (y.YayindaMi)
                        {
                            <span class="badge text-bg-success">Yayında</span>
                        }
                        else
                        {
                            <span class="badge text-bg-secondary">Taslak</span>
                        }
                    </td>

                    <td class="text-nowrap text-secondary">
                        @if (y.YayinTarihi.HasValue)
                        {
                            @y.YayinTarihi.Value.ToString("d MMM yyyy")
                        }
                        else
                        {
                            <span style="opacity:.6;">—</span>
                        }
                    </td>

                    <td class="text-center text-secondary">
                        @y.OkunmaSayisi
                    </td>

                    <td class="text-end text-nowrap">
                        @* Hızlı durum değiştirme — tek tık *@
                        <form asp-action="Durum" asp-route-id="@y.YaziId"
                              method="post" style="display:inline;">
                            <input type="hidden" name="yeniDurum"
                                   value="@(y.YayindaMi ? "Taslak" : "Yayinda")" />
                            <button type="submit" class="btn btn-outline-secondary btn-sm">
                                @(y.YayindaMi ? "Taslağa al" : "Yayınla")
                            </button>
                        </form>

                        @if (y.YayindaMi)
                        {
                            <a href="/yazi/@y.Slug" target="_blank"
                               class="btn btn-outline-secondary btn-sm">Gör</a>
                        }

                        <a asp-action="Sil" asp-route-id="@y.YaziId"
                           class="btn btn-outline-danger btn-sm">Sil</a>
                    </td>
                </tr>
            }
        </tbody>
    </table>
    </div>

    <p class="mt-4 small text-secondary">Toplam @Model.Count yazı</p>
}
```

> `badge text-bg-light border fw-normal` — Bootstrap'in rozeti, açık zeminli,
> ince kenarlıklı ve normal kalınlıkta. Varsayılan rozet kalın yazılıdır; etiket
> listesinde o kadar vurgu gerekmiyor.

---

## ⌨️ Adım 4: Yazı formu

Oluşturma ve düzenleme aynı formu kullanacak. Tekrar yazmamak için **partial view**.

`Views/Yonetim/YaziYonetim/_Form.cshtml`:

```html
@model KisiselBlog.Models.Yazi
@using KisiselBlog.Models
@{
    var tumEtiketler = ViewBag.TumEtiketler as List<Etiket> ?? new List<Etiket>();
    bool duzenleme = Model.YaziId > 0;
}

@* ⭐ Bootstrap ızgarası: geniş ekranda 8+4, dar ekranda alt alta.
     g-4 sütunlar arası boşluk. *@
<div class="row g-4">

    @* ═══════ SOL: içerik ═══════ *@
    <div class="col-lg-8">

        <div class="mb-3">
            <label asp-for="Baslik" class="form-label"></label>
            <input asp-for="Baslik" autofocus
                   placeholder="Yazının başlığı"
                   style="font-size: 1.3125rem;
                          font-weight: 500;" class="form-control" />
            <span asp-validation-for="Baslik" class="text-danger small"></span>

            @if (duzenleme)
            {
                <div class="form-text">
                    Adres: <code>/yazi/@Model.Slug</code>
                    — başlığı değiştirseniz bile <strong>bu adres değişmez</strong>.
                </div>
            }
        </div>

        <div class="mb-3">
            <label asp-for="Ozet" class="form-label"></label>
            <textarea asp-for="Ozet" rows="2"
                      placeholder="Boş bırakırsanız içerikten otomatik üretilir" class="form-control"></textarea>
            <span asp-validation-for="Ozet" class="text-danger small"></span>
            <div class="form-text">
                Liste sayfalarında ve paylaşım önizlemelerinde görünür. En fazla 300 karakter.
            </div>
        </div>

        <div class="mb-3">
            <label asp-for="Icerik" class="form-label"></label>
            @* ⭐ font-monospace → Markdown yazarken hizalama görünür *@
            <textarea asp-for="Icerik" class="form-control font-monospace" id="icerikAlani"
                      placeholder="## Başlık&#10;&#10;Yazınızı **Markdown** ile yazın."></textarea>
            <span asp-validation-for="Icerik" class="text-danger small"></span>

            <div class="form-text" style="display:flex; justify-content:space-between;
                                           flex-wrap:wrap; gap: .75rem;">
                <span>
                    <code>**kalın**</code> ·
                    <code>*italik*</code> ·
                    <code>## başlık</code> ·
                    <code>[bağlantı](url)</code> ·
                    <code>- liste</code> ·
                    <code>`kod`</code>
                </span>
                <span id="sayac"></span>
            </div>
        </div>

    </div>

    @* ═══════ SAĞ: ayarlar ═══════ *@
    <aside class="col-lg-4">
      <div class="card position-sticky" style="top: 1.5rem;">
        <div class="card-body">

        <div class="mb-3">
            <label asp-for="Durum" class="form-label"></label>
            <select asp-for="Durum" class="form-select">
                <option value="Taslak">Taslak</option>
                <option value="Yayinda">Yayında</option>
            </select>
            <div class="form-text">
                Taslaklar sitede görünmez.
            </div>
        </div>

        <div class="mb-3">
            <label>Etiketler</label>

            @if (!tumEtiketler.Any())
            {
                <p class="form-text">
                    Henüz etiket yok.
                    <a asp-controller="EtiketYonetim" asp-action="Index">Etiket ekle</a>
                </p>
            }
            else
            {
                @* ⭐ overflow-auto + yükseklik sınırı: etiket çoğalınca
                     form uzamasın, kendi içinde kaysın *@
                <div class="border rounded p-3 overflow-auto" style="max-height: 14rem;">
                    @foreach (var etiket in tumEtiketler)
                    {
                        <div class="form-check">
                            @* ⭐ Aynı name → ASP.NET Core bunları List<long> yapar *@
                            <input type="checkbox" name="SeciliEtiketIdler"
                                   class="form-check-input"
                                   id="etiket-@etiket.EtiketId"
                                   value="@etiket.EtiketId"
                                   checked="@Model.SeciliEtiketIdler.Contains(etiket.EtiketId)" />
                            <label class="form-check-label" for="etiket-@etiket.EtiketId">
                                @etiket.Ad
                            </label>
                        </div>
                    }
                </div>
            }
        </div>

        @if (duzenleme)
        {
            <div class="border-top pt-3 mt-3 small text-secondary d-flex flex-column gap-1">
                <span>Oluşturuldu: @Model.CreatedDate.ToString("d MMM yyyy, HH:mm")</span>

                @if (Model.UpdatedDate.HasValue)
                {
                    <span>Güncellendi: @Model.UpdatedDate.Value.ToString("d MMM yyyy, HH:mm")</span>
                }

                @if (Model.YayinTarihi.HasValue)
                {
                    <span>Yayın: @Model.YayinTarihi.Value.ToString("d MMM yyyy, HH:mm")</span>
                }

                <span>Okunma: @Model.OkunmaSayisi</span>
            </div>
        }

        <div style="margin-top: 1.5rem; display:flex;
                    flex-direction:column; gap: .5rem;">
            <button type="submit" class="btn btn-primary">Kaydet</button>

            @if (duzenleme && Model.YayindaMi)
            {
                <a href="/yazi/@Model.Slug" target="_blank"
                   class="btn btn-outline-secondary" style="text-align:center;">
                    Sitede gör ↗
                </a>
            }

            <a asp-action="Index" class="btn btn-outline-secondary" style="text-align:center;">
                Listeye dön
            </a>
        </div>

        </div>
      </div>
    </aside>

</div>

@section Scripts {
    <partial name="_ValidationScriptsPartial" />

    <script>
        // ─── Kelime sayacı ve okuma süresi ───────────────
        (function () {
            var alan  = document.getElementById('icerikAlani');
            var sayac = document.getElementById('sayac');
            if (!alan || !sayac) return;

            function guncelle() {
                var metin = alan.value.trim();
                var kelime = metin ? metin.split(/\s+/).length : 0;
                var dakika = Math.max(1, Math.ceil(kelime / 200));
                sayac.textContent = kelime + ' kelime · ~' + dakika + ' dk okuma';
            }

            alan.addEventListener('input', guncelle);
            guncelle();
        })();
    </script>
}
```

### 📖 `position: sticky`

Sağdaki ayar paneli `position: sticky; top: ...` ile tanımlı. Uzun bir yazı yazarken sayfa kayar ama **Kaydet butonu ekranda kalır.**

Küçük bir detay ama günde onlarca kez kullanılacak bir ekranda fark eder.

### Oluştur ve Düzenle view'ları

`Views/Yonetim/YaziYonetim/Olustur.cshtml`:

```html
@model KisiselBlog.Models.Yazi
@{
    ViewData["Title"] = "Yeni yazı";
}

<form asp-action="Olustur" method="post">
    <partial name="_Form" model="Model" />
</form>
```

`Views/Yonetim/YaziYonetim/Duzenle.cshtml`:

```html
@model KisiselBlog.Models.Yazi
@{
    ViewData["Title"] = "Yazıyı düzenle";
}

<form asp-action="Duzenle" asp-route-id="@Model.YaziId" method="post">
    <partial name="_Form" model="Model" />
</form>
```

> ⚠️ **`@section Scripts` partial içinde çalışmaz** — bu bir Razor kısıtıdır. Yukarıdaki `_Form.cshtml`'de `@section Scripts` yazdık ama **çalışmayacak.**
>
> **Düzeltme:** Script bloğunu `_Form.cshtml`'den çıkarıp hem `Olustur.cshtml` hem `Duzenle.cshtml`'e koyun:

`Olustur.cshtml` ve `Duzenle.cshtml`'in **ikisine de** ekleyin:

```html
@section Scripts {
    <partial name="_ValidationScriptsPartial" />
    <script>
        (function () {
            var alan  = document.getElementById('icerikAlani');
            var sayac = document.getElementById('sayac');
            if (!alan || !sayac) return;
            function guncelle() {
                var metin = alan.value.trim();
                var kelime = metin ? metin.split(/\s+/).length : 0;
                var dakika = Math.max(1, Math.ceil(kelime / 200));
                sayac.textContent = kelime + ' kelime · ~' + dakika + ' dk okuma';
            }
            alan.addEventListener('input', guncelle);
            guncelle();
        })();
    </script>
}
```

Ve `_Form.cshtml`'in sonundaki `@section Scripts { ... }` bloğunu **silin.**

> **Bu tuzağı derste gösterin.** Partial'a section yazmak sessizce başarısız olur — hata vermez, sadece script çalışmaz. Öğrenci saatlerce arayabilir.

---

## ⌨️ Adım 5: Silme onayı

`Views/Yonetim/YaziYonetim/Sil.cshtml`:

```html
@model KisiselBlog.Models.Yazi
@{
    ViewData["Title"] = "Yazıyı sil";
}

<div style="max-width: 32rem;">

    <div class="alert alert-warning">
        Bu yazıyı silmek üzeresiniz.
    </div>

    <div class="table-responsive">
    <table class="table table-hover align-middle" style="margin-bottom: 1.5rem;">
        <tbody>
            <tr>
                <th style="width:10rem;">Başlık</th>
                <td>@Model.Baslik</td>
            </tr>
            <tr>
                <th>Adres</th>
                <td><code>/yazi/@Model.Slug</code></td>
            </tr>
            <tr>
                <th>Durum</th>
                <td>@(Model.YayindaMi ? "Yayında" : "Taslak")</td>
            </tr>
            <tr>
                <th>Okunma</th>
                <td>@Model.OkunmaSayisi</td>
            </tr>
        </tbody>
    </table>
    </div>

    <p class="small text-secondary mb-4">
        Kayıt veritabanından tamamen silinmez, pasif duruma alınır.
        Ancak sitede ve listede görünmez olur.
    </p>

    <form asp-action="SilOnayla" asp-route-id="@Model.YaziId" method="post">
        <button type="submit" class="btn btn-outline-danger">Evet, sil</button>
        <a asp-action="Index" class="btn btn-outline-secondary">Vazgeç</a>
    </form>

</div>
```

> ⚠️ Controller'da POST metodunun adı `SilOnayla`, rota ise `sil/{id}`. `[HttpPost("sil/{id:long}")]` özniteliği sayesinde metot adı ile adres farklı olabiliyor — `[ActionName]` kullanmaya gerek kalmıyor.

---

## ▶️ Test

| # | Test | Beklenen |
|---|---|---|
| 1 | Yeni yazı, başlık `Merhaba Dünya` | Kaydediliyor |
| 2 | SSMS'te kontrol | `Slug = merhaba-dunya` |
| 3 | Aynı başlıkla ikinci yazı | `Slug = merhaba-dunya-2` |
| 4 | Başlığı `Yeni Başlık` yap, kaydet | **Slug DEĞİŞMEDİ** |
| 5 | Taslak kaydet | `YayinTarihi` NULL |
| 6 | Yayına al | `YayinTarihi` doldu |
| 7 | Taslağa geri al | `YayinTarihi` NULL oldu |
| 8 | Tekrar yayınla, sonra başlığı değiştir | Yayın tarihi **korundu** |
| 9 | İçerik yazarken | Kelime sayacı güncelleniyor |
| 10 | Etiket seç, kaydet, tekrar aç | Seçimler **işaretli** geliyor |
| 11 | Liste | Taslaklar üstte |
| 12 | "Yayınla" butonu | Tek tıkla durum değişti |

> **4. ve 8. maddeler bu modülün sınavıdır.** Slug ve yayın tarihi korunuyorsa mantık doğru.

---

## ⚠️ Sık yapılan hatalar

| Hata | Sebep | Çözüm |
|---|---|---|
| `Cannot insert duplicate key 'IX_Yazilar_Slug'` | Silinmiş yazının slug'ı | `IgnoreQueryFilters()` |
| Markdown etiketleri ekranda metin olarak | `Html.Raw` yok | Ekle (Modül 6) |
| Kelime sayacı çalışmıyor | `@section Scripts` partial'da | Ana view'a taşı |
| Etiketler kaydedilmiyor | `Include(y => y.Etiketler)` yok | Ekle |
| Etiketler birikiyor | Aynı sebep | Aynı çözüm |
| Yayın tarihi hep bugüne kayıyor | `else if` yerine `else` | Üç durumu ayır |
| Slug başlıkla değişiyor | `Duzenle`'de slug güncelleniyor | Dokunmayın |
| Taslak sitede görünüyor | Genel sorguda durum filtresi yok | Modül 6'da ekleyeceğiz |
| `<script>` çalışıyor | `.DisableHtml()` yok | Pipeline'a ekle |
| Checkbox'lar işaretsiz | `SeciliEtiketIdler` doldurulmamış | `Duzenle` GET'te doldur |

---

## ✏️ Alıştırmalar

1. Yazı CRUD'unu tamamlayın, 12 test senaryosunu geçirin.

2. **Slug çakışma testi:** Bir yazı oluşturun, silin, sonra aynı başlıkla yenisini oluşturun. Slug ne oldu? `IgnoreQueryFilters()` olmasaydı ne olurdu?

3. **XSS deneyi:** `.DisableHtml()` satırını kaldırın, içeriğe `<script>alert(1)</script>` yazın, yazıyı görüntüleyin (Modül 6'dan sonra). Sonra geri koyun.

4. **Slug'ı elle düzenlenebilir yapın.** Forma bir slug alanı ekleyin; boş bırakılırsa başlıktan üretilsin, doluysa onu kullansın.
   ⚠️ Benzersizlik kontrolünü unutmayın.

5. Pasif (silinmiş) yazılar sayfası ekleyin, geri getirme butonuyla.
   ⚠️ `IgnoreQueryFilters()` + `Where(y => !y.AktifMi)`.

**Düşünme soruları:**
- Markdown'ı kaydederken HTML'e çevirip saklasaydık, bir yazım hatası düzeltince ne yapmak gerekirdi?
- Slug'ı başlıkla birlikte güncelleseydik hangi sorun çıkardı? Büyük siteler bunu nasıl çözüyor?
- `Html.Raw` neden tehlikeli? Bizim kullanımımızı güvenli yapan ne?

---

👉 Sonraki: [`06-genel-site.md`](06-genel-site.md)
