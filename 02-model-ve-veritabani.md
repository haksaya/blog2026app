# Modül 2 — Model, Slug ve Veritabanı

**Süre:** 2 ders saati

---

## 🎯 Bu derste ne yapacağız

Altı entity, bir ara tablo, bir migration. **Yeni kavram: slug** — ve Türkçeye özgü bir tuzak.

> CRUD ve EF Core'u biliyorsunuz. Bu modül hızlı geçer. Asıl durulacak yer **slug üretimi.**

---

## 📖 Kavram: Slug nedir?

Önceki projelerde adresler böyleydi:

```
/Gorev/Details/5
/Ogrenci/Edit/142
```

Blogda böyle olacak:

```
/yazi/asp-net-core-a-giris
/etiket/veritabani
/hakkimda
```

Adresteki bu okunabilir kısma **slug** denir.

### Neden id değil slug?

| | `/yazi/5` | `/yazi/asp-net-core-a-giris` |
|---|---|---|
| Okunabilirlik | ❌ Ne olduğu belirsiz | ✅ Tıklamadan anlaşılıyor |
| Paylaşım | Çıplak sayı | ✅ Bağlantı kendini anlatır |
| Arama motoru | Bilgi yok | ✅ Adres içindeki kelimeler sayılır |
| Kalıcılık | Id değişmez | ⚠️ Başlık değişirse slug da değişir |

Son satır önemli: slug değişirse eski bağlantılar kırılır. Bu yüzden **slug bir kez üretilir ve sabit kalır** — başlık düzeltilse bile.

> **Sınıfa sor:** *"Bir haber sitesi başlığı düzeltti. Adres de değişmeli mi?"*
> Hayır. Dışarıdan verilen bağlantılar, paylaşımlar, yer imleri kırılır. Büyük siteler eski adresi yeni adrese **yönlendirir** (301), silmez.

---

## 📖 ⭐ Türkçe slug: üç problem

"Merhaba Dünya" başlığından `merhaba-dunya` üretmek göründüğünden zor.

### Problem 1: Türkçe karakterler

`ü`, `ş`, `ğ`, `ı`, `ö`, `ç` adreslerde sorun çıkarır. Tarayıcılar bunları `%C3%BC` gibi kodlar — paylaşılan bağlantı okunmaz hâle gelir.

Çözüm: harf çevirimi.

```
ç→c   ğ→g   ı→i   ö→o   ş→s   ü→u
Ç→c   Ğ→g   İ→i   Ö→o   Ş→s   Ü→u
```

### Problem 2: ⚠️ Türkçe `I` tuzağı

**Bu, Türk geliştiricilerin en sık düştüğü tuzaktır. Mutlaka gösterin.**

C#'ta `ToLower()` **işletim sisteminin diline göre** çalışır.

```csharp
// Bilgisayarın dili Türkçe ise:
"Istanbul".ToLower()   →  "ıstanbul"     ⚠️  I → ı  (noktasız!)

// Bilgisayarın dili İngilizce ise:
"Istanbul".ToLower()   →  "istanbul"     ✓   I → i
```

**Aynı kod, farklı bilgisayarda farklı sonuç.**

Bu neye yol açar? Slug üretiminde `[^a-z0-9]` dışındaki her şeyi tireye çeviriyoruz. `ı` harfi `a-z` aralığında **değil**:

```
"Istanbul Notları"
    ↓ ToLower()  — Türkçe sistemde
"ıstanbul notları"
    ↓ a-z dışını tireye çevir
"-stanbul-notlari"
    ↓ baştaki tireyi at
"stanbul-notlari"        ⚠️  İLK HARF KAYBOLDU
```

Sitede adres `/yazi/stanbul-notlari` olur. Kimse fark etmez — ta ki biri soruncaya kadar.

**Çözüm:** `ToLowerInvariant()` kullanmak. Bu metot **dilden bağımsızdır**, her bilgisayarda aynı sonucu verir.

```csharp
"Istanbul".ToLowerInvariant()   →  "istanbul"    ✓  her zaman
```

> **Genel kural:** Kullanıcıya **gösterilecek** metinlerde `ToLower()`, **programatik karşılaştırma** ve **adres üretimi** için `ToLowerInvariant()`.
>
> Bu kural sadece slug için değil: dosya adları, API anahtarları, kod karşılaştırmaları — hepsinde geçerli.

### Problem 3: Aynı slug iki kez

İki yazının başlığı aynı olabilir. Veya farklı başlıklar aynı slug'a dönüşebilir:

```
"C# Nedir?"    →  c-nedir
"C Nedir"      →  c-nedir      ⚠️ çakışma
```

Çözüm: sonuna sayı eklemek.

```
c-nedir
c-nedir-2
c-nedir-3
```

---

## ⌨️ Adım 1: `Data/SlugYardimcisi.cs`

```csharp
using System.Text;
using System.Text.RegularExpressions;

namespace KisiselBlog.Data;

/// <summary>
/// Başlıktan adres dostu slug üretir.
/// "Merhaba Dünya" → "merhaba-dunya"
/// </summary>
public static class SlugYardimcisi
{
    // Türkçe harfler ve karşılıkları — sıraları BİREBİR eşleşmeli
    private const string TurkceHarfler = "çğıöşüÇĞİÖŞÜ";
    private const string LatinKarsilik = "cgiosucgiosu";

    /// <summary>
    /// Metni slug'a çevirir.
    ///
    /// Sıra önemli:
    ///   1. Türkçe harfleri çevir   (küçültmeden ÖNCE!)
    ///   2. Küçük harfe çevir       (ToLowerInvariant — ToLower DEĞİL)
    ///   3. Harf/rakam dışını tire yap
    ///   4. Baştaki ve sondaki tireleri at
    /// </summary>
    public static string Uret(string metin)
    {
        if (string.IsNullOrWhiteSpace(metin))
            return "";

        // ── 1) Türkçe harf çevirimi ──────────────────────────
        //
        // ⚠️ Bu adım küçük harfe çevirmeden ÖNCE yapılmalı.
        //    Sonra yapsaydık 'İ' karakteri ToLowerInvariant ile
        //    "i̇" (i + ayrı nokta) olurdu ve eşleşmezdi.
        var tampon = new StringBuilder(metin.Length);

        foreach (char harf in metin)
        {
            int sira = TurkceHarfler.IndexOf(harf);
            tampon.Append(sira >= 0 ? LatinKarsilik[sira] : harf);
        }

        // ── 2) Küçük harfe çevir ─────────────────────────────
        //
        // ⚠️⚠️ ToLowerInvariant — ToLower DEĞİL!
        //    ToLower(), bilgisayarın diline göre çalışır.
        //    Türkçe sistemde 'I' harfini 'ı' yapar ve
        //    bir sonraki adımda o harf tireye dönüşür, kaybolur.
        string slug = tampon.ToString().ToLowerInvariant();

        // ── 3) Harf ve rakam dışındaki her şeyi tire yap ────
        //
        // Art arda gelen işaretler tek tireye düşsün:
        //   "C# -- 12'de"  →  "c-12-de"   (üç tire değil)
        slug = Regex.Replace(slug, @"[^a-z0-9]+", "-");

        // ── 4) Baştaki ve sondaki tireleri at ────────────────
        slug = slug.Trim('-');

        return slug;
    }

    /// <summary>
    /// Benzersiz slug üretir. Slug zaten varsa sonuna sayı ekler.
    ///
    /// slugVarMi: "bu slug kullanımda mı?" sorusunu cevaplayan fonksiyon.
    ///            Veritabanı erişimini dışarıda bırakıyoruz — bu sınıf
    ///            EF'i tanımasın. Böylece test edilebilir kalıyor.
    /// </summary>
    public static string BenzersizUret(string metin, Func<string, bool> slugVarMi)
    {
        string temelSlug = Uret(metin);

        // Başlık tamamen işaretlerden oluşuyorsa slug boş kalabilir
        if (string.IsNullOrEmpty(temelSlug))
            temelSlug = "yazi";

        if (!slugVarMi(temelSlug))
            return temelSlug;

        // Çakışma var — sona sayı ekle
        int sayac = 2;
        while (slugVarMi($"{temelSlug}-{sayac}"))
            sayac++;

        return $"{temelSlug}-{sayac}";
    }
}
```

### 📖 `Func<string, bool>` neden?

`BenzersizUret` metodunun veritabanına bakması gerekiyor. Ama bu sınıfa `DbContext` vermek istemiyoruz.

Çözüm: **soruyu dışarıdan alıyoruz.**

```csharp
// Controller'da kullanımı:
string slug = SlugYardimcisi.BenzersizUret(
    yazi.Baslik,
    s => _db.Yazilar.IgnoreQueryFilters().Any(y => y.Slug == s)
);
```

`SlugYardimcisi` "bu slug var mı?" diye soruyor, cevabı kim verirse versin umursamıyor.

> **Faydası:** Bu sınıf veritabanı olmadan test edilebilir:
> ```csharp
> var mevcutlar = new[] { "merhaba-dunya" };
> var slug = SlugYardimcisi.BenzersizUret("Merhaba Dünya", s => mevcutlar.Contains(s));
> // → "merhaba-dunya-2"
> ```

### ▶️ Slug testleri

Derste bu tabloyu tahtaya yazın ve beraber doğrulayın:

| Girdi | Çıktı |
|---|---|
| `Merhaba Dünya` | `merhaba-dunya` |
| `ASP.NET Core'a Giriş` | `asp-net-core-a-giris` |
| `Işık Hızı ve İzafiyet` | `isik-hizi-ve-izafiyet` |
| `C# 12'de Yenilikler` | `c-12-de-yenilikler` |
| `   Çok   Boşluklu   ` | `cok-bosluklu` |
| `Şiir: Güz Şarkıları` | `siir-guz-sarkilari` |
| `---tire---başta---` | `tire-basta` |

**Deney:** `ToLowerInvariant()` yerine `ToLower()` yazın. Bilgisayarınızın dili Türkçeyse `Işık Hızı` başlığının slug'ı `sik-hizi` olacak. Sonra geri alın.

---

## ⌨️ Adım 2: Enum

`Models/Enums.cs`:

```csharp
namespace KisiselBlog.Models;

/// <summary>
/// Yazının yayın durumu.
///
/// Veritabanına METİN olarak yazılacak (DbContext'te ayarlanıyor).
/// SSMS'te tabloya bakınca "Yayinda" görmek "1" görmekten iyidir.
/// </summary>
public enum YaziDurum
{
    Taslak,
    Yayinda
}
```

> Sadece iki değer var, `bool YayindaMi` da olabilirdi. Enum tercih ettik çünkü ileride `Arsiv`, `Zamanlanmis` gibi durumlar eklenebilir. Enum büyümeye açıktır.

---

## ⌨️ Adım 3: Yazi

`Models/Yazi.cs`:

```csharp
using System.ComponentModel.DataAnnotations;
using System.ComponentModel.DataAnnotations.Schema;

namespace KisiselBlog.Models;

public class Yazi
{
    public long YaziId { get; set; }

    [Required(ErrorMessage = "Başlık zorunludur.")]
    [StringLength(200, ErrorMessage = "En fazla 200 karakter.")]
    [Display(Name = "Başlık")]
    public string Baslik { get; set; } = "";

    // ⭐ SLUG
    // Kullanıcı formda GÖRMEZ ve DOLDURMAZ.
    // Controller, başlıktan üretir. Bir kez üretilir, sabit kalır.
    [StringLength(220)]
    public string Slug { get; set; } = "";

    /// <summary>
    /// Liste sayfalarında ve paylaşım önizlemelerinde görünen kısa tanıtım.
    /// Boş bırakılırsa içerikten otomatik üretilecek (Modül 5).
    /// </summary>
    [StringLength(300, ErrorMessage = "Özet en fazla 300 karakter.")]
    [Display(Name = "Özet")]
    public string? Ozet { get; set; }

    /// <summary>
    /// Yazının gövdesi — MARKDOWN olarak saklanır.
    /// HTML'e çevrim okuma anında yapılır (Modül 5).
    ///
    /// [StringLength] YAZMIYORUZ → NVARCHAR(MAX).
    /// Uzun yazılar için bilinçli tercih.
    /// </summary>
    [Required(ErrorMessage = "İçerik boş olamaz.")]
    [Display(Name = "İçerik")]
    public string Icerik { get; set; } = "";

    /// <summary>Kapak görseli yolu. Modül 10'da dolacak.</summary>
    [StringLength(300)]
    [Display(Name = "Kapak görseli")]
    public string? KapakGorsel { get; set; }

    [Required]
    [Display(Name = "Durum")]
    public YaziDurum Durum { get; set; } = YaziDurum.Taslak;

    /// <summary>
    /// İlk yayınlandığı an. Taslakta NULL.
    /// ⭐ CreatedDate'ten FARKLI: yazı bugün oluşturulup
    ///    gelecek hafta yayınlanabilir.
    /// </summary>
    [Display(Name = "Yayın tarihi")]
    public DateTime? YayinTarihi { get; set; }

    /// <summary>Kaç kez okundu. Modül 6'da artacak.</summary>
    public int OkunmaSayisi { get; set; }

    public DateTime CreatedDate { get; set; } = DateTime.Now;
    public DateTime? UpdatedDate { get; set; }
    public bool AktifMi { get; set; } = true;

    // ── Navigasyon ───────────────────────────────────────────
    // ⭐ Skip navigation — ara tablo sınıfı yazmıyoruz (Modül 8)
    public List<Etiket> Etiketler { get; set; } = new();

    /// <summary>Formdan gelen seçili etiket id'leri.</summary>
    [NotMapped]
    public List<long> SeciliEtiketIdler { get; set; } = new();

    // ── Hesaplanan ───────────────────────────────────────────
    //
    // ⚠️ Hepsi [NotMapped]. Where/OrderBy İÇİNDE KULLANILAMAZ —
    //    EF bunları SQL'e çeviremez. Sadece view'da, veri
    //    belleğe geldikten sonra kullanılır.

    [NotMapped]
    public bool YayindaMi => Durum == YaziDurum.Yayinda;

    /// <summary>
    /// Tahmini okuma süresi (dakika).
    /// Ortalama okuma hızı dakikada ~200 kelime kabul edilir.
    /// </summary>
    [NotMapped]
    public int OkumaSuresi
    {
        get
        {
            if (string.IsNullOrWhiteSpace(Icerik))
                return 1;

            int kelimeSayisi = Icerik.Split(
                new[] { ' ', '\n', '\r', '\t' },
                StringSplitOptions.RemoveEmptyEntries).Length;

            int dakika = (int)Math.Ceiling(kelimeSayisi / 200.0);
            return Math.Max(1, dakika);   // en az 1 dakika yaz
        }
    }

    /// <summary>Listede gösterilecek tanıtım metni.</summary>
    [NotMapped]
    public string GorunenOzet
    {
        get
        {
            if (!string.IsNullOrWhiteSpace(Ozet))
                return Ozet;

            // Özet girilmemişse içerikten üret.
            // Markdown işaretlerini kabaca temizle.
            string duz = Icerik
                .Replace("#", "").Replace("*", "")
                .Replace("`", "").Replace(">", "")
                .Replace("\r", " ").Replace("\n", " ");

            duz = duz.Trim();

            return duz.Length <= 160 ? duz : duz.Substring(0, 157) + "...";
        }
    }
}
```

### ⚠️ `YayinTarihi` ile `CreatedDate` neden ayrı?

**Sor:** *"Yazının oluşturulma tarihi varken yayın tarihine ne gerek var?"*

Gerçek kullanım:

```
2 Ekim   → yazıyı yazmaya başladım        CreatedDate = 2 Ekim
5 Ekim   → hâlâ taslak, düzenliyorum      UpdatedDate = 5 Ekim
8 Ekim   → yayınladım                     YayinTarihi = 8 Ekim
```

Sitede okuyucuya **8 Ekim** gösterilir. Sıralama da ona göre yapılır. `CreatedDate` sadece yönetim panelinde işe yarar.

---

## ⌨️ Adım 4: Etiket, Sayfa, Mesaj, Kullanici

### `Models/Etiket.cs`

```csharp
using System.ComponentModel.DataAnnotations;

namespace KisiselBlog.Models;

public class Etiket
{
    public long EtiketId { get; set; }

    [Required(ErrorMessage = "Etiket adı zorunludur.")]
    [StringLength(50)]
    [Display(Name = "Etiket adı")]
    public string Ad { get; set; } = "";

    // Etiketin de slug'ı var: /etiket/veritabani
    [StringLength(60)]
    public string Slug { get; set; } = "";

    public bool AktifMi { get; set; } = true;

    // Skip navigation
    public List<Yazi> Yazilar { get; set; } = new();
}
```

### `Models/Sayfa.cs`

```csharp
using System.ComponentModel.DataAnnotations;

namespace KisiselBlog.Models;

/// <summary>
/// "Hakkımda", "Projelerim" gibi sabit sayfalar.
///
/// ⭐ Neden veritabanında? Koda gömseydik her yeni sayfa için
///    kod yazıp yeniden yayınlamak gerekirdi. Böyle, panelden
///    eklenebiliyor.
/// </summary>
public class Sayfa
{
    public long SayfaId { get; set; }

    [Required(ErrorMessage = "Başlık zorunludur.")]
    [StringLength(100)]
    [Display(Name = "Sayfa başlığı")]
    public string Baslik { get; set; } = "";

    [StringLength(120)]
    public string Slug { get; set; } = "";

    [Required(ErrorMessage = "İçerik boş olamaz.")]
    [Display(Name = "İçerik")]
    public string Icerik { get; set; } = "";       // markdown

    [Display(Name = "Menüde göster")]
    public bool MenudeGoster { get; set; } = true;

    /// <summary>Menüdeki sırası. Küçük olan solda.</summary>
    [Display(Name = "Sıra")]
    [Range(0, 99, ErrorMessage = "Sıra 0-99 arasında olmalı.")]
    public int Sira { get; set; } = 10;

    public DateTime CreatedDate { get; set; } = DateTime.Now;
    public DateTime? UpdatedDate { get; set; }
    public bool AktifMi { get; set; } = true;
}
```

### `Models/Mesaj.cs`

```csharp
using System.ComponentModel.DataAnnotations;

namespace KisiselBlog.Models;

/// <summary>
/// İletişim formundan gelen mesajlar.
///
/// ⚠️ Bu tabloya GİRİŞ YAPMAMIŞ kişiler yazacak.
///    Projedeki tek anonim yazma noktası. Modül 9'da
///    doğrulama ve spam önlemlerini ele alacağız.
/// </summary>
public class Mesaj
{
    public long MesajId { get; set; }

    [Required(ErrorMessage = "Adınızı yazın.")]
    [StringLength(100)]
    [Display(Name = "Adınız")]
    public string Ad { get; set; } = "";

    [Required(ErrorMessage = "E-posta adresinizi yazın.")]
    [EmailAddress(ErrorMessage = "Geçerli bir e-posta adresi girin.")]
    [StringLength(255)]
    [Display(Name = "E-posta")]
    public string Eposta { get; set; } = "";

    [Required(ErrorMessage = "Konu yazın.")]
    [StringLength(150)]
    [Display(Name = "Konu")]
    public string Konu { get; set; } = "";

    [Required(ErrorMessage = "Mesajınızı yazın.")]
    [StringLength(2000, MinimumLength = 10,
        ErrorMessage = "Mesaj 10-2000 karakter arasında olmalı.")]
    [Display(Name = "Mesajınız")]
    public string Icerik { get; set; } = "";

    public bool OkunduMu { get; set; }

    public DateTime CreatedDate { get; set; } = DateTime.Now;
    public bool AktifMi { get; set; } = true;
}
```

### `Models/Abone.cs`

Yan sütundaki "Abone ol" formu buraya yazacak.

```csharp
using System.ComponentModel.DataAnnotations;
using Microsoft.EntityFrameworkCore;

namespace KisiselBlog.Models;

/// <summary>
/// E-posta bülteni aboneleri.
///
/// ⚠️ Bu tabloya da GİRİŞ YAPMAMIŞ kişiler yazıyor — Mesaj gibi.
///    Modül 9'da ikisini birlikte güvene alacağız.
/// </summary>
[Index(nameof(Eposta), IsUnique = true)]   // ⭐ aynı e-posta iki kere kaydolamaz
public class Abone
{
    public long AboneId { get; set; }

    [Required(ErrorMessage = "E-posta adresi gerekli.")]
    [EmailAddress(ErrorMessage = "Geçerli bir e-posta adresi girin.")]
    [StringLength(255)]
    [Display(Name = "E-posta")]
    public string Eposta { get; set; } = "";

    public DateTime CreatedDate { get; set; } = DateTime.Now;
    public bool AktifMi { get; set; } = true;
}
```

**`[Index(..., IsUnique = true)]` ne yapıyor?**

Veritabanı seviyesinde tekillik kısıtı kurar. Aynı e-posta ikinci kere kaydedilmeye çalışılırsa `SaveChanges` patlar.

> **Niye kodda kontrol etmek yetmiyor?** Yetmiyor çünkü iki kişi aynı anda kaydolursa ikisinin de kontrolü "yok" döner, ikisi de eklenir. Araya giren bu duruma **yarış koşulu** (race condition) denir. Veritabanı kısıtı son savunma hattıdır. Modül 9'da bu hatayı yakalayıp kullanıcıya düzgün mesaj göstereceğiz.

### `Models/Kullanici.cs`

```csharp
using System.ComponentModel.DataAnnotations;

namespace KisiselBlog.Models;

public class Kullanici
{
    public long KullaniciId { get; set; }

    [Required]
    [StringLength(100)]
    [Display(Name = "Kullanıcı adı")]
    public string KullaniciAdi { get; set; } = "";

    [Required]
    [StringLength(255)]
    public string SifreHash { get; set; } = "";

    [Required]
    [StringLength(150)]
    [Display(Name = "Ad soyad")]
    public string AdSoyad { get; set; } = "";

    public DateTime CreatedDate { get; set; } = DateTime.Now;
    public bool AktifMi { get; set; } = true;
}


/// <summary>Giriş formunun taşıyıcısı — tablo değil.</summary>
public class GirisViewModel
{
    [Required(ErrorMessage = "Kullanıcı adı gerekli.")]
    [Display(Name = "Kullanıcı adı")]
    public string KullaniciAdi { get; set; } = "";

    [Required(ErrorMessage = "Şifre gerekli.")]
    [DataType(DataType.Password)]
    [Display(Name = "Şifre")]
    public string Sifre { get; set; } = "";

    [Display(Name = "Beni hatırla")]
    public bool BeniHatirla { get; set; }
}
```

---

## ⌨️ Adım 5: Şifre yardımcısı

`Data/SifreYardimcisi.cs` — önceki projelerden aynen:

```csharp
using System.Security.Cryptography;
using System.Text;

namespace KisiselBlog.Data;

/// <summary>
/// ⚠️ ÖĞRETİM AMAÇLIDIR. Gerçek projede BCrypt/Argon2 kullanın.
/// SHA256 çok hızlıdır ve tuz içermez.
/// </summary>
public static class SifreYardimcisi
{
    public static string Hashle(string sifre)
    {
        byte[] bayt = Encoding.UTF8.GetBytes(sifre);
        byte[] hash = SHA256.HashData(bayt);
        return Convert.ToHexString(hash);
    }
}
```

---

## ⌨️ Adım 6: DbContext

`Data/BlogDbContext.cs`:

```csharp
using Microsoft.EntityFrameworkCore;
using KisiselBlog.Models;

namespace KisiselBlog.Data;

public class BlogDbContext : DbContext
{
    public BlogDbContext(DbContextOptions<BlogDbContext> options)
        : base(options)
    {
    }

    public DbSet<Yazi> Yazilar { get; set; }
    public DbSet<Etiket> Etiketler { get; set; }
    public DbSet<Sayfa> Sayfalar { get; set; }
    public DbSet<Mesaj> Mesajlar { get; set; }
    public DbSet<Abone> Aboneler { get; set; }
    public DbSet<Kullanici> Kullanicilar { get; set; }

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        base.OnModelCreating(modelBuilder);

        // ══════════════════════════════════════════════════════
        //  ENUM → METİN
        //  SSMS'te "Yayinda" görmek "1" görmekten iyi.
        //  ⚠️ Migration üretmeden ÖNCE yazın.
        // ══════════════════════════════════════════════════════
        modelBuilder.Entity<Yazi>()
            .Property(y => y.Durum)
            .HasConversion<string>()
            .HasMaxLength(20);

        // ══════════════════════════════════════════════════════
        //  BENZERSİZ SLUG'LAR
        //
        //  ⭐ Bu indeksler sadece veri bütünlüğü için değil,
        //     PERFORMANS için de kritik. Her yazı sayfası
        //     açılışında "slug'ı bu olan yazı" sorgusu çalışacak.
        //     İndeks olmadan tam tablo taraması yapılır.
        // ══════════════════════════════════════════════════════
        modelBuilder.Entity<Yazi>().HasIndex(y => y.Slug).IsUnique();
        modelBuilder.Entity<Etiket>().HasIndex(e => e.Slug).IsUnique();
        modelBuilder.Entity<Sayfa>().HasIndex(s => s.Slug).IsUnique();

        modelBuilder.Entity<Etiket>().HasIndex(e => e.Ad).IsUnique();
        modelBuilder.Entity<Kullanici>().HasIndex(k => k.KullaniciAdi).IsUnique();

        // Genel site "yayında olanları, tarihe göre" listeleyecek
        modelBuilder.Entity<Yazi>().HasIndex(y => new { y.Durum, y.YayinTarihi });

        // ══════════════════════════════════════════════════════
        //  ÇOKTAN ÇOĞA — ara tablo sınıfı YAZMIYORUZ
        //  Sadece adını veriyoruz (varsayılan "EtiketYazi" olurdu)
        // ══════════════════════════════════════════════════════
        modelBuilder.Entity<Yazi>()
            .HasMany(y => y.Etiketler)
            .WithMany(e => e.Yazilar)
            .UsingEntity(ara => ara.ToTable("YaziEtiketleri"));

        // ══════════════════════════════════════════════════════
        //  SOFT DELETE
        // ══════════════════════════════════════════════════════
        modelBuilder.Entity<Yazi>().HasQueryFilter(y => y.AktifMi);
        modelBuilder.Entity<Etiket>().HasQueryFilter(e => e.AktifMi);
        modelBuilder.Entity<Sayfa>().HasQueryFilter(s => s.AktifMi);
        modelBuilder.Entity<Mesaj>().HasQueryFilter(m => m.AktifMi);
        modelBuilder.Entity<Abone>().HasQueryFilter(a => a.AktifMi);
        modelBuilder.Entity<Kullanici>().HasQueryFilter(k => k.AktifMi);

        // ══════════════════════════════════════════════════════
        //  BAŞLANGIÇ VERİSİ
        //  ⚠️ DateTime.Now KULLANILAMAZ — sabit tarih zorunlu.
        //     (Her migrations add'de değişirse EF boş migration üretir.)
        // ══════════════════════════════════════════════════════
        var sabitTarih = new DateTime(2026, 1, 15, 10, 0, 0);

        modelBuilder.Entity<Kullanici>().HasData(
            new Kullanici
            {
                KullaniciId = 1,
                KullaniciAdi = "admin",
                SifreHash = SifreYardimcisi.Hashle("blog2026"),
                AdSoyad = "Site Sahibi",
                CreatedDate = sabitTarih,
                AktifMi = true
            }
        );

        modelBuilder.Entity<Etiket>().HasData(
            new Etiket { EtiketId = 1, Ad = "Yazılım",      Slug = "yazilim",      AktifMi = true },
            new Etiket { EtiketId = 2, Ad = "Veritabanı",   Slug = "veritabani",   AktifMi = true },
            new Etiket { EtiketId = 3, Ad = "Tasarım",      Slug = "tasarim",      AktifMi = true },
            new Etiket { EtiketId = 4, Ad = "Günlük",       Slug = "gunluk",       AktifMi = true },
            new Etiket { EtiketId = 5, Ad = "Kitap",        Slug = "kitap",        AktifMi = true }
        );

        modelBuilder.Entity<Sayfa>().HasData(
            new Sayfa
            {
                SayfaId = 1,
                Baslik = "Hakkımda",
                Slug = "hakkimda",
                MenudeGoster = true,
                Sira = 1,
                CreatedDate = sabitTarih,
                AktifMi = true,
                Icerik = @"Merhaba. Ben bir yazılım geliştiricisiyim.

Bu siteyi öğrendiklerimi not etmek ve paylaşmak için kurdum. Burada
çoğunlukla **web geliştirme**, **veritabanı tasarımı** ve ara sıra
okuduğum kitaplar üzerine yazıyorum.

## Neler yapıyorum

- ASP.NET Core ile web uygulamaları geliştiriyorum
- Veritabanı tasarımı ve optimizasyonu ile ilgileniyorum
- Arayüz tasarımına ve tipografiye meraklıyım

## İletişim

Bana [iletişim sayfasından](/iletisim) ulaşabilirsiniz."
            }
        );
    }
}
```

### ⚠️ Query filter ve slug çakışması

`Yazi` üzerinde hem benzersiz slug indeksi hem soft delete var. Bu bir tuzak doğurur:

```
1. "Merhaba Dünya" yazısını oluştur    → slug: merhaba-dunya
2. Yazıyı sil                          → AktifMi = 0 (kayıt DURUYOR)
3. Yeni "Merhaba Dünya" yazısı oluştur → slug: merhaba-dunya  ❌ ÇAKIŞMA
```

Silinmiş kayıt veritabanında durduğu için benzersiz indeks hata verir. Ama query filter yüzünden **kontrol sorgumuz onu göremez.**

**Çözüm:** Slug kontrolünde `IgnoreQueryFilters()` kullanmak.

```csharp
s => _db.Yazilar.IgnoreQueryFilters().Any(y => y.Slug == s)
```

> Modül 5'te bunu kodda göreceğiz. Şimdiden bilin: **benzersizlik kontrolü yapan her sorgu, soft delete'li kayıtları da görmelidir.**

---

## ⌨️ Adım 7: Program.cs

```csharp
using Microsoft.EntityFrameworkCore;
using KisiselBlog.Data;

var builder = WebApplication.CreateBuilder(args);

builder.Services.AddControllersWithViews();

builder.Services.AddDbContext<BlogDbContext>(secenekler =>
{
    secenekler.UseSqlServer(
        builder.Configuration.GetConnectionString("BlogDb"));

    if (builder.Environment.IsDevelopment())
    {
        secenekler.LogTo(Console.WriteLine, LogLevel.Information);
        secenekler.EnableSensitiveDataLogging();   // ⚠️ canlıda ASLA
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
app.UseAuthorization();

// Yönlendirme kuralları Modül 6'da genişleyecek
app.MapControllerRoute(
    name: "default",
    pattern: "{controller=Anasayfa}/{action=Index}/{id?}");

app.Run();
```

---

## ⌨️ Adım 8: Migration

```bash
dotnet ef migrations add IlkOlusturma
dotnet ef database update
```

### Doğrula

```sql
USE BlogDB;
GO

SELECT name FROM sys.tables ORDER BY name;
```

Beklenen:
```
Etiketler
Kullanicilar
Mesajlar
Sayfalar
YaziEtiketleri      ← ⭐ EF üretti, biz yazmadık
Yazilar
__EFMigrationsHistory
```

Sütun tiplerini kontrol edin:

```sql
SELECT COLUMN_NAME, DATA_TYPE, CHARACTER_MAXIMUM_LENGTH, IS_NULLABLE
FROM INFORMATION_SCHEMA.COLUMNS
WHERE TABLE_NAME = 'Yazilar'
ORDER BY ORDINAL_POSITION;
```

| Sütun | Tip | Not |
|---|---|---|
| `Baslik` | nvarchar(200) | `[StringLength(200)]` |
| `Slug` | nvarchar(220) | |
| `Ozet` | nvarchar(300) | **NULL olabilir** |
| `Icerik` | nvarchar(**-1**) | `[StringLength]` yok → MAX |
| `Durum` | **nvarchar(20)** | `HasConversion<string>()` |
| `YayinTarihi` | datetime2 | **NULL olabilir** |
| `OkunmaSayisi` | int | |

Hesaplanan özellikler (`OkumaSuresi`, `GorunenOzet`, `YayindaMi`) tabloda **olmamalı** — `[NotMapped]` sayesinde.

İndeksleri kontrol edin:

```sql
SELECT i.name, i.is_unique, COL_NAME(ic.object_id, ic.column_id) AS sutun
FROM sys.indexes i
JOIN sys.index_columns ic ON i.object_id = ic.object_id AND i.index_id = ic.index_id
WHERE i.object_id = OBJECT_ID('Yazilar') AND i.name IS NOT NULL;
```

`IX_Yazilar_Slug` benzersiz olmalı.

---

## ⚠️ Sık yapılan hatalar

| Hata | Sebep | Çözüm |
|---|---|---|
| Slug'da ilk harf kayboluyor | `ToLower()` kullanılmış | `ToLowerInvariant()` |
| Türkçe harfler tireye dönüşüyor | Harf çevirimi küçültmeden sonra yapılmış | Önce çevir, sonra küçült |
| `Cannot insert duplicate key 'IX_Yazilar_Slug'` | Silinmiş yazının slug'ı | Kontrolde `IgnoreQueryFilters()` |
| `Durum` sütunu `int` olmuş | `HasConversion` unutulmuş | Ekle, veritabanını sıfırla |
| Tabloda `OkumaSuresi` sütunu var | `[NotMapped]` unutulmuş | Ekle, migration üret |
| Sürekli boş migration üretiliyor | Seed'de `DateTime.Now` | Sabit tarih |
| `Icerik` nvarchar(450) olmuş | Gereksiz `[StringLength]` | Kaldır → MAX olur |
| Slug boş kalıyor | Başlık tamamen işaret | `BenzersizUret` içinde yedek ad var |

---

## ✏️ Alıştırmalar

1. `SlugYardimcisi.Uret` metodunu bir konsol uygulamasında test edin. Yukarıdaki tablodaki 7 girdiyi deneyin.

2. `ToLowerInvariant()` yerine `ToLower()` yazıp `"Işık Hızı"` başlığını deneyin. Sonucu not edin, sonra geri alın.

3. Aynı başlıkla iki yazı eklemeyi deneyin (Modül 5'ten sonra). İkinci slug ne oldu?

4. `OkumaSuresi` hesabını 200 yerine 150 kelime/dakika yapın. Uzun bir yazıda fark ne kadar?

**Düşünme soruları:**
- Slug'ı başlık değişince güncelleseydik hangi sorun çıkardı?
- `Func<string, bool>` yerine `SlugYardimcisi`'ye `DbContext` verseydik ne kaybederdik?
- `YaziEtiketleri` ara tablosunu biz yazmadık. Nerede oluştu?

---

👉 Sonraki: [`03-iki-layout.md`](03-iki-layout.md)
