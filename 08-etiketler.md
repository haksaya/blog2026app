# Modül 8 — Etiketler

**Süre:** 1 ders saati

---

## 🎯 Bu derste ne yapacağız

Etiket yönetimi. **Çoktan çoğa ilişkiyi GörevTakip'te öğrendiniz** — burada uyguluyoruz.

Yeni olan tek şey: etiketin de slug'ı var ve **adı değişince slug ne olacak?**

---

## 📖 Hızlı hatırlatma: skip navigation

Ara tablo sınıfı yazmıyoruz. İki tarafa birbirinin listesini koyduk:

```csharp
public class Yazi   { public List<Etiket> Etiketler { get; set; } = new(); }
public class Etiket { public List<Yazi>   Yazilar   { get; set; } = new(); }
```

DbContext'te sadece ara tablonun adını verdik:

```csharp
modelBuilder.Entity<Yazi>()
    .HasMany(y => y.Etiketler)
    .WithMany(e => e.Yazilar)
    .UsingEntity(ara => ara.ToTable("YaziEtiketleri"));
```

EF ara tabloyu kendisi üretti ve yönetiyor. Yazı etiketlerini bağlama kodu Modül 5'te yazıldı (`EtiketleriBaglaAsync`).

---

## 📖 Etiket adı değişince slug ne olmalı?

**Sınıfa sor:** *"Yazıda slug'ı sabit tuttuk. Etikette de öyle mi olmalı?"*

Düşünelim:

| | Yazı slug'ı | Etiket slug'ı |
|---|---|---|
| Kaç tane var? | Yüzlerce | Onlarca |
| Dışarıdan bağlantı verilir mi? | ✅ Çok | ⚠️ Nadiren |
| Değişirse etkisi | Paylaşılmış tüm bağlantılar kırılır | Sadece site içi gezinme |

**Karar:** Etiket adı değişince slug'ı **güncelliyoruz**, ama kullanıcıyı uyarıyoruz.

> Bu, mühendislikte sık karşılaşılan bir durum: aynı sorunun iki yerde farklı cevabı olabilir. Önemli olan **gerekçenin** tutarlı olması, kuralın değil.

---

## ⌨️ Adım 1: Controller

`Controllers/EtiketYonetimController.cs`:

```csharp
using Microsoft.AspNetCore.Mvc;
using Microsoft.EntityFrameworkCore;
using KisiselBlog.Data;
using KisiselBlog.Models;

namespace KisiselBlog.Controllers;

[Route("yonetim/etiket")]
public class EtiketYonetimController : YonetimControllerTabani
{
    private readonly BlogDbContext _db;

    public EtiketYonetimController(BlogDbContext db)
    {
        _db = db;
    }

    private string SlugUret(string ad, long haricId = 0)
    {
        return SlugYardimcisi.BenzersizUret(ad, aday =>
            SlugYardimcisi.SlugYasakliMi(aday)
            || _db.Etiketler
                  .IgnoreQueryFilters()
                  .Any(e => e.Slug == aday && e.EtiketId != haricId));
    }

    // ══════════════════════════════════════════════════════
    //  LİSTE  — yazı sayılarıyla birlikte
    // ══════════════════════════════════════════════════════
    [HttpGet("")]
    public async Task<IActionResult> Index()
    {
        ViewData["Title"] = "Etiketler";

        // ⭐ Select içinde alt sorgu: Include'a gerek yok.
        //    Include kullansaydık TÜM yazılar belleğe gelirdi;
        //    biz sadece SAYI istiyoruz.
        var etiketler = await _db.Etiketler
            .AsNoTracking()
            .Select(e => new EtiketSatir
            {
                EtiketId     = e.EtiketId,
                Ad           = e.Ad,
                Slug         = e.Slug,
                ToplamYazi   = e.Yazilar.Count(y => y.AktifMi),
                YayindaYazi  = e.Yazilar.Count(y => y.AktifMi
                                                 && y.Durum == YaziDurum.Yayinda)
            })
            .OrderByDescending(x => x.ToplamYazi)
            .ThenBy(x => x.Ad)
            .ToListAsync();

        return View(etiketler);
    }

    // ══════════════════════════════════════════════════════
    //  EKLE  — liste sayfasındaki satır içi formdan
    // ══════════════════════════════════════════════════════
    [HttpPost("ekle")]
    [ValidateAntiForgeryToken]
    public async Task<IActionResult> Ekle(string ad)
    {
        if (string.IsNullOrWhiteSpace(ad))
        {
            TempData["Uyari"] = "Etiket adı boş olamaz.";
            return RedirectToAction(nameof(Index));
        }

        ad = ad.Trim();

        bool varMi = await _db.Etiketler
            .IgnoreQueryFilters()
            .AnyAsync(e => e.Ad == ad);

        if (varMi)
        {
            TempData["Uyari"] = $"\"{ad}\" etiketi zaten var.";
            return RedirectToAction(nameof(Index));
        }

        _db.Etiketler.Add(new Etiket
        {
            Ad = ad,
            Slug = SlugUret(ad),
            AktifMi = true
        });

        await _db.SaveChangesAsync();

        TempData["Basarili"] = $"\"{ad}\" etiketi eklendi.";
        return RedirectToAction(nameof(Index));
    }

    // ══════════════════════════════════════════════════════
    //  YENİDEN ADLANDIR
    // ══════════════════════════════════════════════════════
    [HttpPost("duzenle/{id:long}")]
    [ValidateAntiForgeryToken]
    public async Task<IActionResult> Duzenle(long id, string ad)
    {
        var etiket = await _db.Etiketler.FindAsync(id);

        if (etiket == null)
            return NotFound();

        if (string.IsNullOrWhiteSpace(ad))
        {
            TempData["Uyari"] = "Etiket adı boş olamaz.";
            return RedirectToAction(nameof(Index));
        }

        ad = ad.Trim();

        bool varMi = await _db.Etiketler
            .IgnoreQueryFilters()
            .AnyAsync(e => e.Ad == ad && e.EtiketId != id);

        if (varMi)
        {
            TempData["Uyari"] = $"\"{ad}\" adında başka bir etiket var.";
            return RedirectToAction(nameof(Index));
        }

        string eskiSlug = etiket.Slug;
        etiket.Ad = ad;
        etiket.Slug = SlugUret(ad, id);

        await _db.SaveChangesAsync();

        if (eskiSlug != etiket.Slug)
        {
            TempData["Uyari"] = $"Adres değişti: /etiket/{eskiSlug} → /etiket/{etiket.Slug}";
        }

        TempData["Basarili"] = "Etiket güncellendi.";
        return RedirectToAction(nameof(Index));
    }

    // ══════════════════════════════════════════════════════
    //  SİL
    // ══════════════════════════════════════════════════════
    [HttpPost("sil/{id:long}")]
    [ValidateAntiForgeryToken]
    public async Task<IActionResult> Sil(long id)
    {
        var etiket = await _db.Etiketler
            .Include(e => e.Yazilar)
            .FirstOrDefaultAsync(e => e.EtiketId == id);

        if (etiket == null)
            return NotFound();

        // ⭐ Etiket silinirken yazılarla bağlantısını da kes.
        //    Kesmezsek ara tabloda "hayalet" satırlar kalır:
        //    yazı pasif bir etikete bağlı görünür.
        etiket.Yazilar.Clear();

        etiket.AktifMi = false;
        await _db.SaveChangesAsync();

        TempData["Basarili"] = $"\"{etiket.Ad}\" etiketi silindi.";
        return RedirectToAction(nameof(Index));
    }
}

/// <summary>Liste ekranının taşıyıcısı — tablo değil.</summary>
public class EtiketSatir
{
    public long EtiketId { get; set; }
    public string Ad { get; set; } = "";
    public string Slug { get; set; } = "";
    public int ToplamYazi { get; set; }
    public int YayindaYazi { get; set; }
}
```

### ⚠️ `etiket.Yazilar.Clear()` — neden gerekli?

Soft delete yapıyoruz, etiket kaydı duruyor. Ama ara tablodaki satırlar da duruyor.

**Sonuç:** Yazı düzenleme ekranında o etiket görünmez (query filter gizliyor) ama `yazi.Etiketler` koleksiyonunda hâlâ var. `Clear()` + `Add()` mantığı bozulur.

`Clear()` ile ara tablo satırlarını siliyoruz — onlar zaten "kayıt" değil, sadece bağlantı.

---

## ⌨️ Adım 2: Liste görünümü

`Views/Yonetim/EtiketYonetim/Index.cshtml`:

```html
@model List<KisiselBlog.Controllers.EtiketSatir>
@{
    ViewData["Title"] = "Etiketler";
}

@* ─── HIZLI EKLEME ───────────────────────────────────── *@
<form asp-action="Ekle" method="post"
      class="d-flex gap-3 align-items-end mb-4" style="max-width: 28rem;">

    <div class="flex-fill">
        <label for="ad" class="form-label">Yeni etiket</label>
        <input type="text" id="ad" name="ad" maxlength="50" class="form-control"
               placeholder="Örn: Veritabanı" required />
    </div>

    <button type="submit" class="btn btn-primary">Ekle</button>
</form>


@if (!Model.Any())
{
    <div class="text-center py-5 text-secondary">
        <p>Henüz etiket yok. Yukarıdaki formdan ekleyebilirsiniz.</p>
    </div>
}
else
{
    <div class="table-responsive">
    <table class="table table-hover align-middle">
        <thead>
            <tr>
                <th>Etiket</th>
                <th class="text-center" style="width: 7rem;">Yayında</th>
                <th class="text-center" style="width: 7rem;">Toplam</th>
                <th class="text-end" style="width: 18rem;">İşlem</th>
            </tr>
        </thead>
        <tbody>
            @foreach (var e in Model)
            {
                <tr>
                    <td>
                        @* Satır içi düzenleme formu *@
                        <form asp-action="Duzenle" asp-route-id="@e.EtiketId"
                              method="post" class="d-flex gap-2 align-items-center">
                            <input type="text" name="ad" value="@e.Ad" maxlength="50"
                                   class="form-control form-control-sm"
                                   style="max-width: 16rem;" />
                            <button type="submit" class="btn btn-outline-secondary btn-sm">
                                Kaydet
                            </button>
                        </form>

                        <div class="small text-secondary font-monospace">
                            /etiket/@e.Slug
                        </div>
                    </td>

                    <td class="text-center">@e.YayindaYazi</td>

                    <td class="text-center text-secondary">
                        @e.ToplamYazi
                    </td>

                    <td class="text-end text-nowrap">
                        @if (e.YayindaYazi > 0)
                        {
                            <a href="/etiket/@e.Slug" target="_blank"
                               class="btn btn-outline-secondary btn-sm">Gör</a>
                        }

                        <a asp-controller="YaziYonetim" asp-action="Index"
                           class="btn btn-outline-secondary btn-sm">Yazılar</a>

                        <form asp-action="Sil" asp-route-id="@e.EtiketId"
                              method="post" style="display:inline;"
                              onsubmit="return confirm('@e.Ad etiketi silinecek ve @e.ToplamYazi yazıdan kaldırılacak. Emin misiniz?');">
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

---

## ⌨️ Adım 3: Panel menüsünde sayaçlar

Modül 3'te panel navbar'ında mesaj sayacı için yer bırakmıştık. Şimdi dolduruyoruz — hem okunmamış mesaj hem taslak yazı sayısı.

`_YonetimLayout.cshtml`'de doğrudan hesaplıyoruz — en az kodla:

```csharp
@inject BlogDbContext Db
@{
    var okunmamisMesaj = Db.Mesajlar.Count(m => !m.OkunduMu);
    var taslakYazi = Db.Yazilar.Count(y => y.Durum == YaziDurum.Taslak);
    // ... mevcut kod
}
```

Ve menü öğelerine:

```html
<li class="nav-item">
    <a class="nav-link d-flex align-items-center gap-2 @Aktif("yazi")"
       asp-controller="YaziYonetim" asp-action="Index">
        Yazılar
        @if (taslakYazi > 0)
        {
            <span class="badge text-bg-secondary">@taslakYazi</span>
        }
    </a>
</li>

<li class="nav-item">
    <a class="nav-link d-flex align-items-center gap-2 @Aktif("mesaj")"
       asp-controller="MesajYonetim" asp-action="Index">
        Mesajlar
        @if (okunmamisMesaj > 0)
        {
            <span class="badge text-bg-danger">@okunmamisMesaj</span>
        }
    </a>
</li>
```

> ⚠️ `nav-link` ve `<li class="nav-item">` sarmalayıcısını **düşürmeyin** — Modül 3'teki
> navbar bunlara dayanıyor. Düşerse bağlantılar navbar'da hizasız ve beyaz kalır.
>
> İki rozet farklı renkte: taslak **gri** (bilgi), okunmamış mesaj **kırmızı** (eylem
> gerektiriyor). Aynı rengi verseydiniz hangisinin acil olduğu belli olmazdı.

`_YonetimLayout.cshtml`'in başına da using ekleyin:

```csharp
@using KisiselBlog.Models
@using KisiselBlog.Data
```

---

## ▶️ Test

| # | Test | Beklenen |
|---|---|---|
| 1 | Etiket listesi | Seed'deki 5 etiket |
| 2 | "Makine Öğrenmesi" ekle | Slug: `makine-ogrenmesi` |
| 3 | Aynı adı tekrar ekle | "zaten var" uyarısı |
| 4 | Bir etiketi yeniden adlandır | Slug güncellendi, uyarı çıktı |
| 5 | Yazıya 2 etiket ata | Kaydediliyor |
| 6 | `SELECT * FROM YaziEtiketleri` | 2 satır |
| 7 | Bir etiketi çıkar, birini ekle | Konsolda **1 DELETE + 1 INSERT** |
| 8 | Etiketi sil | Yazılardan da kalktı |
| 9 | Sitede `/etiketler` | Sadece yayında yazısı olanlar |
| 10 | Panel menüsü | Taslak sayısı rozeti görünüyor |

> **7. madde EF'in değişiklik takibinin kanıtı.** 1 DELETE + 2 INSERT görüyorsanız `Include` eksik demektir.

---

## ⚠️ Sık yapılan hatalar

| Hata | Sebep | Çözüm |
|---|---|---|
| Etiketler kaydedilmiyor | `Include(y => y.Etiketler)` yok | Ekle |
| Etiketler birikiyor | Aynı sebep | Aynı çözüm |
| Silinen etiket yazıda görünüyor | `Yazilar.Clear()` yok | Ekle |
| Slug çakışması | `IgnoreQueryFilters()` yok | Ekle |
| Yazı formundaki etiket rozetleri fazla koyu | `text-bg-secondary` kullanılmış (Modül 5) | `text-bg-light border fw-normal` |
| Navbar bağlantıları hizasız | Snippet'ten `nav-link` düşmüş | `class="nav-link …"` + `<li class="nav-item">` |
| `/etiket/x` 404 | Slug yanlış | Panelden doğru slug'ı kontrol et |
| Liste yavaş | `Include` ile tüm yazılar çekiliyor | `Select` ile sayım yap |

---

## ✏️ Alıştırmalar

1. Etiket yönetimini tamamlayın, 10 test senaryosunu geçirin.

2. **Kullanılmayan etiketleri temizleme** butonu ekleyin: hiç yazısı olmayan etiketleri toplu silsin.
   İpucu: `Where(e => !e.Yazilar.Any()).ExecuteUpdateAsync(...)`

3. Panel menüsünün tamamını bir View Component'e taşıyın. Layout'taki `@inject`'ten kurtulun.

4. **Etiket birleştirme:** İki etiketi tek etikete dönüştüren bir işlem yazın (örn. "DB" ve "Veritabanı" → "Veritabanı").
   ⚠️ Ara tabloda çakışma olabilir — aynı yazı iki etikete de bağlıysa.

**Düşünme soruları:**
- Yazı slug'ını sabit tuttuk, etiket slug'ını güncelledik. Bu kararın gerekçesi neydi?
- `Select` ile sayım yaptık, `Include` kullanmadık. 5 etiket ve 50 yazıda fark eder mi? 500 yazıda?
- Etiketi soft delete yapıyoruz ama ara tablo satırlarını gerçekten siliyoruz. Neden farklı davranıyoruz?

---

👉 Sonraki: [`09-iletisim.md`](09-iletisim.md)
