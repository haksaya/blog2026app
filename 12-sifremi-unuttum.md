# Modül 12 — Şifremi Unuttum

**Süre:** 2-3 ders saati
**Önkoşul:** Modül 4 (giriş sistemi) tamamlanmış olmalı

---

## 🎯 Bu derste ne yapacağız

Şifresini unutan site sahibi, e-postasına gelen **tek kullanımlık bağlantıyla** yeni şifre belirleyecek.

```
┌──────────────────────┐
│  /sifremi-unuttum    │  e-posta adresini yaz
└──────────┬───────────┘
           │ POST
           ▼
┌──────────────────────────────────────────────┐
│ 1. Kullanıcıyı bul (BULAMASAN DA aynı mesaj) │
│ 2. 32 baytlık rastgele token üret            │
│ 3. Token'ın HASH'ini veritabanına yaz        │
│ 4. Token'ın KENDİSİNİ e-postaya koy          │
└──────────┬───────────────────────────────────┘
           │ SMTP
           ▼
   ┌───────────────┐
   │ Gelen kutusu  │  bağlantıya tıkla
   └───────┬───────┘
           ▼
┌──────────────────────────┐
│ /sifre-sifirla?token=…   │  yeni şifreyi iki kez yaz
└──────────┬───────────────┘
           │ POST
           ▼
┌───────────────────────────────────────┐
│ 1. Token'ı TEKRAR doğrula             │
│ 2. Şifreyi değiştir                   │
│ 3. Token'ı kullanıldı işaretle        │
│ 4. Bekleyen diğer token'ları iptal et │
│ 5. Açık oturumları düşür              │
└───────────────────────────────────────┘
```

**Yeni teknikler:** kriptografik rastgelelik, token hash'leme, SMTP ile e-posta, kullanıcı sırrını yapılandırmada saklama (user secrets), çerez oturumunu sunucudan iptal etme.

---

## 📖 Kavram — Niye yeni şifreyi e-postayla göndermiyoruz?

**Sınıfa sor:** *"En kolay çözüm ne? Rastgele bir şifre üret, e-postayla yolla, bitti."*

Dört sorunu var:

| Sorun | Açıklama |
|---|---|
| **E-posta güvenli bir kanal değil** | Sunucular arasında şifresiz taşınabilir; yolda okunabilir. |
| **Kalıcı** | Gelen kutusunda yıllarca durur. Kişi e-posta hesabını birine devretse şifre de gider. |
| **Biz şifreyi bilmiş oluruz** | Modül 2'de şifreyi `hash`liyoruz ki **biz bile** bilmeyelim. Üretip gönderirsek bu ilke bozulur. |
| **Kullanıcı değiştirmeyebilir** | Gönderdiğimiz geçici şifre kalıcı şifre hâline gelir. |

Doğrusu: **kısa ömürlü, tek kullanımlık bir bağlantı gönder, şifreyi kullanıcı belirlesin.** Biz hiçbir zaman şifreyi görmeyiz.

---

## 📖 Kavram — Token nedir, niye hash'liyoruz?

**Token**, bağlantının içindeki rastgele dizedir:

```
https://site.com/sifre-sifirla?token=xK3nQ9vL2pR7mT4wY8cZ1aB5dF6gH0jN
```

Bunu bilen kişi şifreyi değiştirebilir — yani token, geçici bir **anahtar**.

### Veritabanına token'ın kendisini değil, hash'ini yazıyoruz

**Sor:** *"Token'ı veritabanına olduğu gibi yazsak ne olurdu?"*

Veritabanı sızarsa (yedek dosyası çalınır, SQL injection, yanlış yapılandırılmış sunucu), saldırgan tüm bekleyen token'ları eline geçirir ve hesapları ele geçirir.

Hash saklarsak sızan veri işe yaramaz: hash'ten token geri üretilemez.

```
Kullanıcıya giden   →  xK3nQ9vL2pR7...        (ham token)
Veritabanına yazılan →  A4F2...9B1C           (SHA256 hash'i)
Bağlantı tıklanınca  →  gelen token'ı hash'le, veritabanındakiyle KARŞILAŞTIR
```

Bu, şifre saklamanın aynı mantığı: **sırrın kendisini değil, doğrulanabilir izini sakla.**

### ⭐ Buradaki nüans: şifre için SHA256 yetersizdi, token için yeterli

Modül 2'de uyarmıştık: *"SHA256 öğretim amaçlıdır, gerçek şifreler için BCrypt/Argon2 kullanın."* Peki token için niye SHA256 diyoruz?

Fark **entropi** — yani tahmin edilemezlik miktarı:

| | Şifre | Sıfırlama token'ı |
|---|---|---|
| Kaynağı | İnsan seçer | `RandomNumberGenerator` üretir |
| Entropi | ~30 bit (`blog2026`) | **256 bit** |
| Olası değer | Milyarlarca — sözlük saldırısı işe yarar | 2²⁵⁶ ≈ 10⁷⁷ |
| Savunma | Hash'i **yavaşlatmak** (BCrypt) | Gerek yok |

BCrypt, şifreyi her denemeyi yavaşlatarak korur — çünkü deneme sayısı makul bir aralıkta. 256 bitlik rastgele bir değeri kaba kuvvetle bulmaya evrenin ömrü yetmez; yavaşlatmanın anlamı yok.

> **Kural:** Yavaş hash (BCrypt, Argon2) → **insan seçtiği** sırlar için.
> Hızlı hash (SHA256) → **makine ürettiği** yüksek entropili sırlar için.

---

## ⌨️ Adım 1 — Model değişiklikleri

Buraya kadar şema hiç değişmemişti: Modül 2'de tasarladık, bir kere migration ürettik. Şimdi **içinde veri olan** bir veritabanına sütun ekliyoruz. Gerçek projelerde olan tam olarak budur.

### 1.1 `Kullanici`'ye iki alan

`Models/Kullanici.cs` — sınıfın içine ekleyin:

```csharp
using System.ComponentModel.DataAnnotations;
using Microsoft.EntityFrameworkCore;          // ⭐ [Index] için

namespace KisiselBlog.Models;

[Index(nameof(Eposta), IsUnique = true)]      // ⭐ yeni
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

    // ⭐ YENİ — sıfırlama bağlantısı buraya gidecek
    [Required(ErrorMessage = "E-posta adresi gerekli.")]
    [EmailAddress(ErrorMessage = "Geçerli bir e-posta adresi girin.")]
    [StringLength(255)]
    [Display(Name = "E-posta")]
    public string Eposta { get; set; } = "";

    // ⭐ YENİ — açık oturumları iptal etmek için (Adım 6)
    //
    //    Şifre değişince bu damga yenilenir. Çerezdeki eski damga
    //    artık tutmaz, o çerez geçersiz olur.
    [Required]
    [StringLength(36)]
    public string GuvenlikDamgasi { get; set; } = Guid.NewGuid().ToString();

    [Required]
    [StringLength(150)]
    [Display(Name = "Ad soyad")]
    public string AdSoyad { get; set; } = "";

    public DateTime CreatedDate { get; set; } = DateTime.Now;
    public bool AktifMi { get; set; } = true;
}
```

### 1.2 Yeni tablo: `SifreSifirlama`

`Models/SifreSifirlama.cs`:

```csharp
using System.ComponentModel.DataAnnotations;
using Microsoft.EntityFrameworkCore;

namespace KisiselBlog.Models;

/// <summary>
/// Her şifre sıfırlama isteği için bir satır.
///
/// ⭐ Token'ın KENDİSİ burada yok — sadece SHA256 hash'i.
/// ⭐ AktifMi/soft delete YOK: bu bir denetim kaydı, silinmez.
/// </summary>
[Index(nameof(TokenHash))]        // aramayı hızlandırır
public class SifreSifirlama
{
    public long SifreSifirlamaId { get; set; }

    public long KullaniciId { get; set; }
    public Kullanici? Kullanici { get; set; }

    /// <summary>SHA256 hex çıktısı — her zaman 64 karakter.</summary>
    [Required]
    [StringLength(64)]
    public string TokenHash { get; set; } = "";

    public DateTime SonKullanma { get; set; }

    /// <summary>
    /// Ne zaman kullanıldı. NULL ise henüz kullanılmamış.
    ///
    /// ⚠️ bool yerine DateTime? seçtik: "kullanıldı mı" sorusunu da
    ///    cevaplıyor, "ne zaman" sorusunu da. Bir sorun araştırırken
    ///    ikincisi birincisinden değerli olur.
    /// </summary>
    public DateTime? KullanildiTarih { get; set; }

    /// <summary>İsteği yapan IP. IPv6 en fazla 45 karakter.</summary>
    [StringLength(45)]
    public string? IstekIp { get; set; }

    public DateTime CreatedDate { get; set; } = DateTime.Now;
}
```

### 1.3 DbContext

`Data/BlogDbContext.cs`:

```csharp
public DbSet<SifreSifirlama> SifreSifirlamalari { get; set; }
```

`OnModelCreating` içine:

```csharp
// Kullanıcı silinirse token'ları da gitsin
modelBuilder.Entity<SifreSifirlama>()
    .HasOne(s => s.Kullanici)
    .WithMany()
    .HasForeignKey(s => s.KullaniciId)
    .OnDelete(DeleteBehavior.Cascade);
```

⚠️ **`SifreSifirlama`'ya query filter EKLEMİYORUZ.** Diğer tüm tablolarımızda
`HasQueryFilter(x => x.AktifMi)` var; burada bilerek yok.

### 📖 Konsolda göreceğiniz uyarı

Projeyi çalıştırdığınızda EF şöyle bir uyarı yazacak:

```
warn: Microsoft.EntityFrameworkCore.Model.Validation[10622]
      Entity 'Kullanici' has a global query filter defined and is the required
      end of a relationship with the entity 'SifreSifirlama'. This may lead to
      unexpected results when the required entity is filtered out.
```

**Bu bir hata değil, bir uyarı** — uygulama çalışır. EF şunu söylüyor:

> `Kullanici`'nin bir filtresi var (`AktifMi`) ve `SifreSifirlama` ilişkinin
> **zorunlu** ucunda duruyor. Kullanıcı filtreyle elenirse token satırı da
> beklemediğin biçimde kaybolabilir.

> ⚠️ **Mesajı dikkatli okuyun:** uyarı `SifreSifirlama`'da filtre olduğunu
> söylemiyor — filtresi olan `Kullanici`. Biz zaten `SifreSifirlama`'ya filtre
> koymadık; uyarının sebebi ilişkinin **diğer** ucundaki filtre.

**Sor:** *"Peki bu bizim için sorun mu?"*

Hayır — tam tersine, **istediğimiz davranış bu.** Pasife alınmış bir kullanıcının
sıfırlama token'ı çalışmamalı. Zorunlu ilişki olduğu için EF `INNER JOIN` üretiyor,
filtrelenen kullanıcının satırları da sonuçtan düşüyor.

Yine de koda bir güvenlik ağı koyuyoruz (Adım 5.4'teki `kayit?.Kullanici == null`
kontrolü) — çünkü *"EF'nin şu an ürettiği SQL böyle"* ile *"bu her zaman böyle kalacak"*
aynı şey değil.

Uyarıyı günlükten temizlemek isterseniz, **bilerek** sustuğunuzu belgeleyin:

```csharp
// Program.cs — AddDbContext içinde
secenekler.ConfigureWarnings(u => u.Ignore(
    CoreEventId.PossibleIncorrectRequiredNavigationWithQueryFilterInteractionWarning));
```

`using Microsoft.EntityFrameworkCore.Diagnostics;` gerekiyor.

> ⚠️ Bir uyarıyı susturmadan önce **ne dediğini anlayın.** Anlamadan susturmak,
> ileride gerçek bir sorunu görmemek demektir. Burada susturuyoruz çünkü uyardığı
> davranışı zaten istiyoruz — ve bunu yorum satırında yazıyoruz ki altı ay sonra
> kendimiz de hatırlayalım.

### 📖 Niye bu tabloda soft delete yok?

Diğer tablolarda `AktifMi` var: bir yazıyı silince kayıt duruyor, geri alınabiliyor.

`SifreSifirlama` farklı — bu bir **denetim kaydı**. Kimse onu "silmiyor", kullanıyor
veya kullanmıyor. `KullanildiTarih` zaten durumu taşıyor. Üstüne bir de `AktifMi`
koymak, aynı bilgiyi iki yerde tutmak olurdu.

> **Kural:** Soft delete, kullanıcının *silebildiği* şeyler içindir. Günlük
> kayıtları, denetim izleri ve token'lar için anlamı yoktur; onlar yaşlanınca
> toplu silinir (Adım 5.2'deki 30 günlük temizlik).

### 1.4 Seed verisini güncelleyin

`HasData` içindeki `Kullanici` kaydına iki alan ekleyin:

```csharp
new Kullanici
{
    KullaniciId = 1,
    KullaniciAdi = "admin",
    SifreHash = SifreYardimcisi.Hashle("blog2026"),
    AdSoyad = "Site Sahibi",

    // ⚠️ BUNU KENDİ ADRESİNİZLE DEĞİŞTİRİN —
    //    sıfırlama postası buraya gidecek, test edemezsiniz yoksa.
    //
    // ⚠️ KÜÇÜK HARFLE yazın. Controller gelen adresi
    //    ToLowerInvariant() ile küçültüp arıyor; burada
    //    "Siz@Ornek.com" yazarsanız eşleşme yalnızca SQL Server'ın
    //    varsayılan büyük/küçük harf duyarsız collation'ı sayesinde
    //    çalışır. Buna bel bağlamayın — PostgreSQL'de çalışmazdı.
    Eposta = "siz@ornek.com",

    // ⚠️ Guid.NewGuid() YAZMAYIN.
    //    HasData her migration üretiminde çalışır; değişken bir değer
    //    yazarsanız EF her seferinde "veri değişmiş" sanıp gereksiz
    //    migration üretir. Sabit bir dize olmalı.
    GuvenlikDamgasi = "11111111-1111-1111-1111-111111111111",

    CreatedDate = sabitTarih,
    AktifMi = true
}
```

### 1.5 Migration

```bash
dotnet ef migrations add SifreSifirlamaEklendi
dotnet ef database update
```

Üretilen migration dosyasını **açıp okuyun** — bu derste bakılması gereken yer:

```csharp
migrationBuilder.AddColumn<string>(
    name: "Eposta",
    table: "Kullanicilar",
    maxLength: 255,
    nullable: false,
    defaultValue: "");            // ⭐ buraya dikkat

migrationBuilder.UpdateData(
    table: "Kullanicilar",
    keyColumn: "KullaniciId",
    keyValue: 1L,
    columns: new[] { "Eposta", "GuvenlikDamgasi" },
    values: new object[] { "siz@ornek.com", "1111...1111" });
```

### ⚠️ `defaultValue: ""` niye orada?

Tabloda zaten bir satır var (seed'den gelen `admin`). `Eposta` sütunu `NOT NULL` olacak — peki mevcut satıra ne yazılacak?

SQL Server bunu kendiliğinden bilemez, bu yüzden EF boş dize veriyor. Sonra `UpdateData` o satırı gerçek değerle güncelliyor.

**Tek bir satırda bu sorunsuz çalışıyor. Ama 50.000 kullanıcılı bir tabloda ne olurdu?** Hepsi `""` olurdu ve benzersiz indeks ikinci satırda patlardı. Gerçek projelerde bu işlem üç adımda yapılır:

1. Sütunu **nullable** ekle
2. Veriyi doldur (script veya kod)
3. Ayrı bir migration ile `NOT NULL` yap

> **Derste gösterin:** `dotnet ef migrations script` komutu üretilen SQL'i yazdırır. Canlı veritabanında migration çalıştırmadan önce bu çıktıyı okumak standart bir alışkanlıktır.

---

## ⌨️ Adım 2 — E-posta gönderme

### 2.1 Paket

```bash
dotnet add package MailKit
```

> **Niye `System.Net.Mail.SmtpClient` değil?** `[Obsolete]` ile işaretli değil, yani derleyici uyarmıyor — ama Microsoft'un kendi dokümantasyonu *"yeni geliştirmeler için önerilmez"* diyor ve doğrudan **MailKit**'e yönlendiriyor. Sebep: modern kimlik doğrulama (OAuth2) ve TLS seçeneklerini tam desteklemiyor.

### 2.2 SMTP hesabı

Üç seçenek — laboratuvar için **Mailtrap** en rahatı:

| Seçenek | Nasıl | Not |
|---|---|---|
| **Mailtrap** (önerilen) | mailtrap.io → ücretsiz hesap → Email Testing → SMTP bilgileri | Gerçek SMTP'dir ama posta **dışarı çıkmaz**, Mailtrap'in web arayüzünde birikir. Sınıfta herkes rahatça test eder, kimseye spam gitmez. |
| **Gmail** | 2 adımlı doğrulamayı aç → "Uygulama şifreleri" → 16 haneli şifre üret | Normal Gmail şifreniz **çalışmaz**. Günlük gönderim sınırı var. |
| **Outlook / Yandex** | Benzer şekilde uygulama şifresi | Gmail'e alternatif |

### 2.3 Ayarlar

`appsettings.json`'a ekleyin:

```json
"Eposta": {
  "Sunucu": "sandbox.smtp.mailtrap.io",
  "Port": 587,
  "KullaniciAdi": "abc123def456",
  "Sifre": "",
  "GonderenAd": "Harun's Blog",
  "GonderenAdres": "bildirim@harunaksaya.blog"
}
```

### ⚠️ `Sifre` boş — ve öyle kalacak

**Şifreyi `appsettings.json`'a YAZMAYIN.** O dosya projenin içinde; Git'e gönderirseniz şifre dünyaya açılır. GitHub'da sızmış SMTP şifresi aramak bir saldırgan için rutin iştir.

Geliştirme ortamında doğru yer **user secrets**:

```bash
dotnet user-secrets init
dotnet user-secrets set "Eposta:Sifre" "buraya-gercek-sifre"
```

Nerede duruyor?

```
Windows : %APPDATA%\Microsoft\UserSecrets\<id>\secrets.json
Linux   : ~/.microsoft/usersecrets/<id>/secrets.json
macOS   : ~/.microsoft/usersecrets/<id>/secrets.json
```

**Proje klasörünün dışında** — yani `.gitignore`'a bir şey eklemenize bile gerek yok, Git onu hiç görmüyor.

`dotnet user-secrets init` komutu `.csproj` dosyasına bir `<UserSecretsId>` satırı ekler. Bu **gizli değil**, sadece hangi klasöre bakılacağını söyleyen bir kimlik; Git'e gitmesi sorun değil.

Çalışma mantığı: `IConfiguration` kaynakları sırayla okur, **sonraki öncekini ezer**:

```
appsettings.json  →  appsettings.Development.json  →  user secrets  →  ortam değişkenleri
                                                      ↑
                                        "Eposta:Sifre" buradan geliyor
```

Kodda hiçbir şey değişmiyor — `Configuration["Eposta:Sifre"]` aynı şekilde çalışıyor.

> ⚠️ **User secrets sadece Development ortamında okunur.** Canlıda ortam değişkeni veya Azure Key Vault / AWS Secrets Manager kullanılır. Ortam değişkeninde iç içe anahtarlar iki alt çizgiyle yazılır: `Eposta__Sifre`.

### 2.4 Ayar sınıfı

`Models/EpostaAyarlari.cs`:

```csharp
namespace KisiselBlog.Models;

public class EpostaAyarlari
{
    public string Sunucu { get; set; } = "";
    public int Port { get; set; } = 587;
    public string KullaniciAdi { get; set; } = "";
    public string Sifre { get; set; } = "";
    public string GonderenAd { get; set; } = "";
    public string GonderenAdres { get; set; } = "";

    /// <summary>
    /// Gerçekten gönderim yapabilir miyiz?
    ///
    /// ⚠️ Şifreyi de kontrol ediyoruz. Sadece Sunucu'ya baksaydık,
    ///    appsettings.json'da sunucu adı yazan ama user secrets'ı
    ///    kurmamış öğrencide "yapılandırıldı" çıkar, SMTP kimlik
    ///    doğrulaması patlar ve bağlantı hiçbir yerde görünmezdi.
    /// </summary>
    public bool Yapilandirildi =>
        !string.IsNullOrWhiteSpace(Sunucu) &&
        !string.IsNullOrWhiteSpace(GonderenAdres) &&
        !string.IsNullOrWhiteSpace(KullaniciAdi) &&
        !string.IsNullOrWhiteSpace(Sifre);
}
```

### 2.5 Arayüz ve SMTP uygulaması

`Data/IEpostaGonderici.cs`:

```csharp
namespace KisiselBlog.Data;

public interface IEpostaGonderici
{
    /// <summary>
    /// ⭐ İki gövde birden istiyoruz: HTML ve düz metin.
    ///    Nedeni 2.5'in sonunda.
    /// </summary>
    Task GonderAsync(string aliciAdres, string aliciAd, string konu,
                     string htmlGovde, string duzMetinGovde);
}
```

**Niye arayüz?** Controller'ın MailKit'i tanımasına gerek yok; "e-posta gönder" diyor, nasıl gönderildiği onu ilgilendirmiyor. Yarın SendGrid'e geçerseniz tek sınıf değişir. Ve SMTP'si olmayan bir öğrenci aşağıdaki konsol sürümünü takıp dersi bitirebilir.

`Data/SmtpEpostaGonderici.cs`:

```csharp
using MailKit.Net.Smtp;
using MailKit.Security;
using MimeKit;
using KisiselBlog.Models;

namespace KisiselBlog.Data;

public class SmtpEpostaGonderici : IEpostaGonderici
{
    private readonly EpostaAyarlari _ayar;
    private readonly ILogger<SmtpEpostaGonderici> _gunluk;

    public SmtpEpostaGonderici(EpostaAyarlari ayar, ILogger<SmtpEpostaGonderici> gunluk)
    {
        _ayar = ayar;
        _gunluk = gunluk;
    }

    public async Task GonderAsync(string aliciAdres, string aliciAd,
                                  string konu, string htmlGovde,
                                  string duzMetinGovde)
    {
        if (!_ayar.Yapilandirildi)
            throw new InvalidOperationException(
                "E-posta ayarları eksik. appsettings.json'daki \"Eposta\" bölümünü " +
                "ve user secrets'taki \"Eposta:Sifre\" değerini kontrol edin.");

        var ileti = new MimeMessage();
        ileti.From.Add(new MailboxAddress(_ayar.GonderenAd, _ayar.GonderenAdres));
        ileti.To.Add(new MailboxAddress(aliciAd, aliciAdres));
        ileti.Subject = konu;

        // ⭐ multipart/alternative: iki gövdeyi birlikte gönderir.
        //    İstemci hangisini gösterebiliyorsa onu seçer.
        ileti.Body = new BodyBuilder
        {
            HtmlBody = htmlGovde,
            TextBody = duzMetinGovde
        }.ToMessageBody();

        using var istemci = new SmtpClient();

        try
        {
            // ⭐ StartTls: önce şifresiz bağlanır, sonra TLS'e yükseltir.
            //    Port 587 bunu bekler.
            //    Port 465 kullanıyorsanız → SecureSocketOptions.SslOnConnect
            await istemci.ConnectAsync(_ayar.Sunucu, _ayar.Port, SecureSocketOptions.StartTls);
            await istemci.AuthenticateAsync(_ayar.KullaniciAdi, _ayar.Sifre);
            await istemci.SendAsync(ileti);

            _gunluk.LogInformation("E-posta gönderildi → {adres} ({konu})", aliciAdres, konu);
        }
        finally
        {
            // ⚠️ finally içinde: gönderim patlasa da bağlantı kapanmalı.
            //    quit: true → sunucuya düzgünce "QUIT" der, bağlantıyı
            //    yarıda kesmez.
            if (istemci.IsConnected)
                await istemci.DisconnectAsync(quit: true);
        }
    }

}
```

### ⚠️ Düz metin sürümünü HTML'den ÜRETMEYİN, elle yazın

İlk akla gelen, HTML'i ayrıştırıp etiketleri atmak. İki sebeple yapmıyoruz:

**1. Hazır bir araç yok.** MimeKit'te `HtmlToHtml`, `TextToHtml`, `FlowedToText`
dönüştürücüleri var ama **`HtmlToText` yok.** Regex ile etiket silmek ise klasik
bir hatadır: iç içe etiketler, `&amp;` gibi varlıklar, `<style>` blokları hep bozulur.

**2. Zaten istemiyoruz.** Düz metin sürümü HTML'in soluk bir kopyası olmamalı; kendi
başına okunabilir bir mektup olmalı. Düğme düz metinde düğme olamaz — adresin kendisi
görünmeli.

Bu yüzden iki gövdeyi **ayrı ayrı** yazıyoruz (Adım 5.3). Gerçek e-posta sistemleri de
böyle çalışır: her şablonun bir `.html` bir de `.txt` sürümü olur.

### 📖 Niye düz metin sürümü zorunlu?

| Sebep | Açıklama |
|---|---|
| **Spam puanı** | Sadece HTML gönderen postalar spam filtrelerinden daha düşük puan alır. Çift gövde "düzgün bir gönderici" işaretidir. |
| **Erişilebilirlik** | Bazı ekran okuyucular ve eski istemciler düz metni tercih eder. |
| **Akıllı saat / bildirim** | Önizlemeler genelde düz metin gövdesinden okunur. |

### 2.6 Program.cs'e kaydedin

```csharp
// ── E-posta ──
builder.Services.AddSingleton(
    builder.Configuration.GetSection("Eposta").Get<EpostaAyarlari>()
    ?? new EpostaAyarlari());

builder.Services.AddScoped<IEpostaGonderici, SmtpEpostaGonderici>();
```

> `EpostaAyarlari` **singleton** (uygulama boyunca sabit), `SmtpEpostaGonderici` **scoped** (her istekte yeni). Bir SMTP istemcisini istekler arasında paylaşmak bağlantı durumu sorunlarına yol açar.

### ⚠️ Laboratuvarda SMTP kurulamadıysa

Arayüzün ikinci bir uygulamasını yazıp onu kaydedin. Akışın tamamı çalışır, tek fark bağlantının gelen kutusu yerine **konsolda** görünmesi:

```csharp
// Data/KonsolEpostaGonderici.cs — sadece geliştirme içindir
namespace KisiselBlog.Data;

public class KonsolEpostaGonderici : IEpostaGonderici
{
    private readonly ILogger<KonsolEpostaGonderici> _gunluk;

    public KonsolEpostaGonderici(ILogger<KonsolEpostaGonderici> gunluk) => _gunluk = gunluk;

    public Task GonderAsync(string aliciAdres, string aliciAd, string konu,
                            string htmlGovde, string duzMetinGovde)
    {
        // ⭐ Konsola DÜZ METİN sürümünü yazıyoruz — HTML'i terminalde
        //    okumak işkence olurdu. Bağlantı düz metinde zaten açıkta.
        _gunluk.LogWarning("\n─── E-POSTA (gönderilmedi) ───\nKime: {ad} <{adres}>\n" +
                           "Konu: {konu}\n\n{govde}\n──────────────────────────────",
                           aliciAd, aliciAdres, konu, duzMetinGovde);
        return Task.CompletedTask;
    }
}
```

`Program.cs`'te, ayarlar eksikse otomatik ona düşsün:

```csharp
var epostaAyar = builder.Configuration.GetSection("Eposta").Get<EpostaAyarlari>()
                 ?? new EpostaAyarlari();
builder.Services.AddSingleton(epostaAyar);

if (epostaAyar.Yapilandirildi)
    builder.Services.AddScoped<IEpostaGonderici, SmtpEpostaGonderici>();
else
    builder.Services.AddScoped<IEpostaGonderici, KonsolEpostaGonderici>();
```

> ⚠️ Bunu **canlıya çıkarmayın.** Ayar bir şekilde eksik kalırsa sıfırlama bağlantıları sessizce log dosyasına yazılır ve kimse haberdar olmaz. Canlıda eksik ayar, uygulamanın **açılmaması** gereken bir durumdur.

---

## ⌨️ Adım 3 — Token yardımcısı

`Data/SifirlamaTokeni.cs`:

```csharp
using System.Security.Cryptography;
using System.Text;
using Microsoft.AspNetCore.WebUtilities;

namespace KisiselBlog.Data;

public static class SifirlamaTokeni
{
    /// <summary>Bağlantının geçerlilik süresi.</summary>
    public const int GecerlilikDakika = 60;

    /// <summary>
    /// 32 baytlık (256 bit) kriptografik güvenli rastgele token üretir.
    /// </summary>
    public static string Uret()
    {
        byte[] bayt = RandomNumberGenerator.GetBytes(32);

        // ⭐ Base64Url — düz Base64 DEĞİL.
        //    Base64 çıktısında '+', '/' ve '=' bulunur; üçü de URL'de
        //    özel anlam taşır ve kaçırılmaları gerekir. Base64Url
        //    bunları '-', '_' ile değiştirir ve dolguyu atar.
        return WebEncoders.Base64UrlEncode(bayt);
    }

    /// <summary>Veritabanına yazılacak hash. Çıktı 64 karakterlik hex.</summary>
    public static string Hashle(string token)
    {
        byte[] hash = SHA256.HashData(Encoding.UTF8.GetBytes(token));
        return Convert.ToHexString(hash);
    }
}
```

### ⚠️ `Random` ile `RandomNumberGenerator` farkı

Bu, dersin en önemli iki satırından biri.

```csharp
// ❌ ASLA — güvenlik için kullanılamaz
var r = new Random();
byte[] bayt = new byte[32];
r.NextBytes(bayt);

// ✅ Doğrusu
byte[] bayt = RandomNumberGenerator.GetBytes(32);
```

`Random`, hızlı olmak için tasarlanmış **deterministik** bir üreteçtir. Bir iç durumu vardır ve her çağrı o durumu belli bir formülle ilerletir. Yeterince çıktı gören biri iç durumu geri hesaplayıp **sonraki tüm değerleri** öngörebilir. Bir saldırgan için bu, "kendi hesabına sıfırlama iste, gelen token'a bak, sizinkini hesapla" demektir.

`RandomNumberGenerator`, işletim sisteminin kriptografik üretecini kullanır (Windows'ta CNG, Linux'ta `getrandom`). Çıktısı geriye doğru çözülemez.

> **Kural:** Token, oturum kimliği, API anahtarı, tuz (salt) — güvenliğe dokunan **her** rastgelelik `RandomNumberGenerator`'dan gelir. `Random` zar atmak ve test verisi üretmek içindir.
>
> ⚠️ .NET 6 ile gelen `Random.Shared` daha iyi tohumlanır ama **hâlâ kriptografik değildir.** Adındaki "paylaşımlı" kelimesi güvenlik vaat etmez.

---

## ⌨️ Adım 4 — ViewModel'ler

`Models/Kullanici.cs` dosyasının sonuna (giriş ViewModel'inin yanına):

```csharp
/// <summary>"Şifremi unuttum" formu.</summary>
public class SifremiUnuttumViewModel
{
    [Required(ErrorMessage = "E-posta adresinizi yazın.")]
    [EmailAddress(ErrorMessage = "Geçerli bir e-posta adresi girin.")]
    [Display(Name = "E-posta adresiniz")]
    public string Eposta { get; set; } = "";
}

/// <summary>Yeni şifre belirleme formu.</summary>
public class SifreSifirlaViewModel
{
    // ⚠️ ErrorMessage'ı boş bırakmayın: varsayılan mesaj İngilizce
    //    ("The Token field is required.") ve kullanıcıya hiçbir şey
    //    anlatmaz. Zaten bu hatayı kullanıcıya hiç göstermeyeceğiz —
    //    controller onu "bağlantı geçersiz" ekranına çeviriyor.
    [Required(ErrorMessage = "Bağlantı bilgisi eksik.")]
    public string Token { get; set; } = "";

    [Required(ErrorMessage = "Yeni şifreyi yazın.")]
    [StringLength(100, MinimumLength = 8,
        ErrorMessage = "Şifre en az 8 karakter olmalı.")]
    [DataType(DataType.Password)]
    [Display(Name = "Yeni şifre")]
    public string YeniSifre { get; set; } = "";

    [Required(ErrorMessage = "Şifreyi tekrar yazın.")]
    [DataType(DataType.Password)]
    [Display(Name = "Yeni şifre (tekrar)")]
    [Compare(nameof(YeniSifre), ErrorMessage = "Şifreler eşleşmiyor.")]
    public string YeniSifreTekrar { get; set; } = "";
}
```

> `[Compare]` iki alanın eşitliğini kontrol eder; hem sunucuda hem (jQuery validation sayesinde) tarayıcıda çalışır. Modül 4'ün 4. alıştırmasında ipucu olarak geçmişti.
>
> `[DataType(DataType.Password)]`, Tag Helper'a `type="password"` ürettirir — alan yazarken noktalarla görünür.

---

## ⌨️ Adım 5 — Controller

`Controllers/HesapController.cs`'e ekliyoruz. Önce yapıcı metodu güncelleyin:

> ⚠️ **Bu bölümdeki kod parçaları tek bir sınıfın içine giriyor.** Aşağıdaki ilk blok
> sınıfı açıyor, 5.6'daki son blok kapatıyor. Aradaki parçaları sırayla sınıfın içine yapıştırın —
> her birini ayrı ayrı derlemeye çalışmayın, süslü parantezler eşleşmez.

```csharp
using System.Text.Encodings.Web;      // ⭐ yeni — HtmlEncoder
// ... Modül 4'teki mevcut using'ler duruyor

[AllowAnonymous]
public class HesapController : Controller
{
    private readonly BlogDbContext _db;
    private readonly IEpostaGonderici _eposta;              // ⭐ yeni
    private readonly SiteAyarlari _site;                    // ⭐ yeni
    private readonly ILogger<HesapController> _gunluk;      // ⭐ yeni

    public HesapController(BlogDbContext db,
                           IEpostaGonderici eposta,
                           SiteAyarlari site,
                           ILogger<HesapController> gunluk)
    {
        _db = db;
        _eposta = eposta;
        _site = site;
        _gunluk = gunluk;
    }

    // ... mevcut Giris / Cikis action'ları duruyor
```

### 5.1 Şifremi unuttum — form

```csharp
    // ══════════════════════════════════════════════════════
    //  ŞİFREMİ UNUTTUM — FORM
    //  GET: /sifremi-unuttum
    // ══════════════════════════════════════════════════════
    [HttpGet("/sifremi-unuttum")]
    public IActionResult SifremiUnuttum()
    {
        if (User.Identity?.IsAuthenticated == true)
            return RedirectToAction("Index", "Panel");

        ViewData["Title"] = "Şifremi unuttum";
        return View(new SifremiUnuttumViewModel());
    }
```

### 5.2 Şifremi unuttum — işle

```csharp
    // ══════════════════════════════════════════════════════
    //  ŞİFREMİ UNUTTUM — İŞLE
    //  POST: /sifremi-unuttum
    // ══════════════════════════════════════════════════════
    [HttpPost("/sifremi-unuttum")]
    [ValidateAntiForgeryToken]
    public async Task<IActionResult> SifremiUnuttum(SifremiUnuttumViewModel model)
    {
        ViewData["Title"] = "Şifremi unuttum";

        if (!ModelState.IsValid)
            return View(model);

        // ⚠️ ToLowerInvariant — ToLower() DEĞİL.
        //    Modül 2'deki Türkçe I tuzağı: "ALI@x.com".ToLower()
        //    Türkçe kültürde "alı@x.com" olur ve adres bozulur.
        string temizEposta = model.Eposta.Trim().ToLowerInvariant();

        // ── Hız sınırı: 15 dakikada en fazla 3 istek ─────────
        var pencereBasi = DateTime.Now.AddMinutes(-15);

        int sonIstekSayisi = await _db.SifreSifirlamalari
            .CountAsync(s => s.Kullanici!.Eposta == temizEposta
                          && s.CreatedDate > pencereBasi);

        var kullanici = await _db.Kullanicilar
            .FirstOrDefaultAsync(k => k.Eposta == temizEposta);

        if (kullanici != null && sonIstekSayisi < 3)
        {
            // ── Eski kayıtları temizle (30 günden eski) ──────
            //
            // ⭐ ExecuteDeleteAsync: nesneleri belleğe çekmeden tek
            //    DELETE sorgusu çalıştırır. (EF Core 7+)
            //
            // ⚠️ Bu satır BİLEREK burada — if'in içinde.
            //    Dışarıda olsaydı, kayıtsız bir adresle yapılan HER
            //    anonim istek tabloya DELETE atardı. Bir saldırgan
            //    bunu dakikada binlerce kez tetikleyebilirdi.
            //    Burada hız sınırı onu üç istekle kapatıyor.
            await _db.SifreSifirlamalari
                .Where(s => s.CreatedDate < DateTime.Now.AddDays(-30))
                .ExecuteDeleteAsync();

            // ── Token üret ───────────────────────────────────
            string token = SifirlamaTokeni.Uret();

            _db.SifreSifirlamalari.Add(new SifreSifirlama
            {
                KullaniciId = kullanici.KullaniciId,
                TokenHash   = SifirlamaTokeni.Hashle(token),   // ⭐ hash'i
                SonKullanma = DateTime.Now.AddMinutes(SifirlamaTokeni.GecerlilikDakika),
                IstekIp     = HttpContext.Connection.RemoteIpAddress?.ToString()
            });

            await _db.SaveChangesAsync();

            // ── Bağlantıyı kur ───────────────────────────────
            //
            // Request.Scheme → "https", Request.Host → "localhost:7123"
            //
            // ⚠️ Request.Host, HTTP isteğindeki "Host:" başlığından gelir —
            //    yani KULLANICIDAN. Açıklaması hemen aşağıda.
            string baglanti = Url.Action(nameof(SifreSifirla), "Hesap",
                                         new { token }, Request.Scheme)!;

            // ── Gönder ───────────────────────────────────────
            try
            {
                await _eposta.GonderAsync(
                    kullanici.Eposta,
                    kullanici.AdSoyad,
                    $"{_site.Baslik} — şifre sıfırlama",
                    EpostaHtmlOlustur(kullanici.AdSoyad, baglanti),
                    EpostaDuzMetinOlustur(kullanici.AdSoyad, baglanti));
            }
            catch (Exception ex)
            {
                // ⚠️ Kullanıcıya hata GÖSTERMİYORUZ.
                //    "E-posta gönderilemedi" demek, o adresin kayıtlı
                //    olduğunu ele verirdi. Sorunu log'a yazıp sessiz
                //    kalıyoruz — bedeli: SMTP bozuksa kullanıcı
                //    boşuna bekler. Alternatifleri aşağıda tartışıyoruz.
                _gunluk.LogError(ex, "Sıfırlama postası gönderilemedi: {adres}",
                                 kullanici.Eposta);
            }
        }
        else if (kullanici != null)
        {
            _gunluk.LogWarning("Hız sınırı aşıldı: {adres}", temizEposta);
        }

        // ⭐⭐ HER DURUMDA AYNI SONUÇ.
        //     Kullanıcı var/yok, sınır aşıldı/aşılmadı, posta gitti/gitmedi —
        //     dışarıdan hiçbiri ayırt edilemiyor.
        return RedirectToAction(nameof(SifremiUnuttumGonderildi));
    }

    [HttpGet("/sifremi-unuttum/gonderildi")]
    public IActionResult SifremiUnuttumGonderildi()
    {
        ViewData["Title"] = "Bağlantı gönderildi";
        return View();
    }
```

### ⚠️ Host başlığı zehirlenmesi — şifre sıfırlamanın klasik açığı

`Url.Action(..., Request.Scheme)` mutlak bir adres üretiyor. Alan adını nereden alıyor? İsteğin `Host:` başlığından. Ve o başlık **saldırganın yazabileceği** bir şey:

```http
POST /sifremi-unuttum HTTP/1.1
Host: saldirgan.com          ← istediğini yazabilir
Content-Type: application/x-www-form-urlencoded

eposta=kurban@ornek.com
```

Sonuç: kurbanın gelen kutusuna **gerçek sitenizden**, doğru görünümlü bir posta gider — ama içindeki bağlantı `https://saldirgan.com/sifre-sifirla?token=...` adresini gösterir. Kurban tıklar, token saldırganın sunucusuna düşer.

**Savunma — ikisi de yapılmalı:**

**1. `AllowedHosts`'u doldurun.** `appsettings.json`'da varsayılan `"*"` yazıyor, yani her Host başlığı kabul ediliyor:

```json
"AllowedHosts": "harunaksaya.blog;www.harunaksaya.blog"
```

Böylece `HostFilteringMiddleware` tanımadığı bir Host gelince isteği **400** ile reddeder. (Geliştirme yaparken `"*"` kalabilir; canlıya çıkarken doldurun.)

**2. Bağlantıyı istekten değil ayardan kurun.** Daha sağlamı: temel adresi `SiteAyarlari`'ne koyup `Request.Host`'a hiç güvenmemek. Alıştırma 7'de yapacaksınız.

> **Genel kural, Modül 4'ten tanıdık:** `donusUrl`'e güvenmemiştik, `Referer`'e güvenmemiştik — `Host` da aynı listede. **İstekten gelen her şey kullanıcı girdisidir**, başlıklar dâhil.

### 📖 Üç güvenlik kararı

**1. Kullanıcı sayımı (user enumeration) — üçüncü kez.**

Modül 4'te girişte, Modül 9'da abonelikte aynı ilkeyi uyguladık: *"kayıtlı mı değil mi" sorusunu cevaplama.* Burada da öyle. Saldırgan adres listesini formda deneyerek hangilerinin sistemde olduğunu öğrenemiyor.

**Sor:** *"Peki kullanıcı dostu mu? Adresini yanlış yazan kişi niye bir şey gelmediğini anlamayacak."*

Gerçek siteler bu yüzden mesajı dikkatli yazar: *"Bu adres kayıtlıysa bir bağlantı gönderdik. Gelmezse spam klasörünü ve adresin doğruluğunu kontrol edin."* Bilgi vermeden yönlendiriyor.

**2. Hız sınırı neyi koruyor?**

Sınır olmasaydı biri aynı adrese saniyede yüzlerce istek atıp kişinin gelen kutusunu doldurabilirdi (mail bombing). Üstelik SMTP sağlayıcınız sizi kara listeye alırdı.

⚠️ Bu sınır **kayıtlı adresler** için çalışıyor — kayıtsız bir adres için hiç satır oluşmadığı için sayım hep 0. Onun için IP bazlı sınır gerekir; .NET 7+ `builder.Services.AddRateLimiter(...)` ile gelir.

**3. Zamanlama sızıntısı — dürüst olalım**

Kullanıcı varsa e-posta gönderiyoruz: ~1 saniye. Yoksa göndermiyoruz: ~10 milisaniye. Yanıt süresini ölçen biri farkı görür ve sayım korumamızı **delip geçer.**

Gerçek çözüm: e-postayı arka plan kuyruğuna atıp isteği hemen bitirmek (`Channel` + `IHostedService`). Bu dersin kapsamı dışında ama bilinmesi gereken bir açık. Rastgele gecikme eklemek **çözüm değildir** — yeterli örnekle ortalama yine ayrışır.

### 5.3 E-posta gövdesi

```csharp
    /// <summary>
    /// Sıfırlama postasının HTML gövdesi.
    ///
    /// ⚠️ Tüm stiller SATIR İÇİ (inline). E-posta istemcilerinin
    ///    çoğu <style> bloğunu ve harici CSS'i atar. Gmail
    ///    <head> içindeki her şeyi siler.
    /// </summary>
    private string EpostaHtmlOlustur(string adSoyad, string baglanti)
    {
        // ⭐ HtmlEncoder: ada veya bağlantıya HTML kaçamasın.
        //    Yazı gövdesinde Html.Raw güvenliydi çünkü içeriği biz
        //    yazıyorduk; burada da öyle ama alışkanlık doğru olanı.
        var kod = HtmlEncoder.Default;

        return $@"
<!DOCTYPE html>
<html lang=""tr"">
<body style=""margin:0; padding:24px; background:#f6f8fa;
              font-family:Arial, Helvetica, sans-serif; color:#4a4a4a;"">

  <div style=""max-width:520px; margin:0 auto; background:#ffffff;
              border:1px solid #e3e7ec; border-radius:8px; padding:32px;"">

    <h1 style=""margin:0 0 16px; font-size:20px; color:#2b2b2b;"">
      Şifre sıfırlama
    </h1>

    <p style=""margin:0 0 16px; line-height:1.6;"">
      Merhaba {kod.Encode(adSoyad)},
    </p>

    <p style=""margin:0 0 24px; line-height:1.6;"">
      <strong>{kod.Encode(_site.Baslik)}</strong> için şifre sıfırlama isteği aldık.
      Yeni şifrenizi belirlemek için aşağıdaki düğmeye basın.
    </p>

    <p style=""margin:0 0 24px;"">
      <a href=""{kod.Encode(baglanti)}""
         style=""display:inline-block; background:#ffd43b; color:#212529;
                text-decoration:none; font-weight:bold; font-size:14px;
                padding:12px 24px; border-radius:4px;"">
        Yeni şifre belirle
      </a>
    </p>

    <p style=""margin:0 0 8px; line-height:1.6; font-size:14px; color:#6b6b6b;"">
      Bu bağlantı <strong>{SifirlamaTokeni.GecerlilikDakika} dakika</strong>
      geçerlidir ve yalnızca bir kez kullanılabilir.
    </p>

    <p style=""margin:0 0 24px; line-height:1.6; font-size:14px; color:#6b6b6b;"">
      Bu isteği siz yapmadıysanız bu postayı yok sayabilirsiniz —
      şifreniz değişmeyecek.
    </p>

    <hr style=""border:none; border-top:1px solid #e3e7ec; margin:0 0 16px;"">

    <p style=""margin:0; font-size:12px; color:#6b6b6b; line-height:1.6;"">
      Düğme çalışmazsa bu adresi tarayıcınıza yapıştırın:<br>
      <span style=""word-break:break-all;"">{kod.Encode(baglanti)}</span>
    </p>

  </div>
</body>
</html>";
    }
```

> **Niye düğmenin altında düz adres de var?** Bazı kurumsal e-posta ağ geçitleri bağlantıları yeniden yazar veya düğmeleri bozar. Yapıştırılabilir bir adres, standart bir yedek.

Ve düz metin eşi — aynı bilgi, biçimlendirmesiz:

```csharp
    /// <summary>
    /// Postanın düz metin sürümü.
    ///
    /// ⭐ HTML'den üretilmiyor, ayrı yazılıyor. Burada düğme yok;
    ///    adresin kendisi satır başında duruyor ki tıklanabilsin.
    /// </summary>
    private string EpostaDuzMetinOlustur(string adSoyad, string baglanti)
    {
        return $@"Merhaba {adSoyad},

{_site.Baslik} için şifre sıfırlama isteği aldık.
Yeni şifrenizi belirlemek için aşağıdaki adresi tarayıcınıza yapıştırın:

{baglanti}

Bu bağlantı {SifirlamaTokeni.GecerlilikDakika} dakika geçerlidir ve
yalnızca bir kez kullanılabilir.

Bu isteği siz yapmadıysanız bu postayı yok sayabilirsiniz —
şifreniz değişmeyecek.

—
{_site.Baslik}";
    }
```

> Düz metinde `HtmlEncoder` **kullanmıyoruz** — kaçırılacak bir biçimlendirme yok.
> HTML'de kullanıyoruz çünkü ada veya adrese `<` karakteri girerse düzeni bozar.

### 5.4 Token doğrulama yardımcısı

```csharp
    /// <summary>
    /// Token geçerliyse kaydı döner, değilse null.
    ///
    /// Üç koşulun ÜÇÜ de sağlanmalı:
    ///   1. Hash eşleşiyor
    ///   2. Daha önce kullanılmamış
    ///   3. Süresi dolmamış
    /// </summary>
    private async Task<SifreSifirlama?> TokenBulAsync(string? token)
    {
        if (string.IsNullOrWhiteSpace(token))
            return null;

        string hash = SifirlamaTokeni.Hashle(token);

        var kayit = await _db.SifreSifirlamalari
            .Include(s => s.Kullanici)
            .FirstOrDefaultAsync(s => s.TokenHash == hash
                                   && s.KullanildiTarih == null
                                   && s.SonKullanma > DateTime.Now);

        // Query filter pasif kullanıcıyı eler; yine de savunmada kalıyoruz
        return kayit?.Kullanici == null ? null : kayit;
    }
```

### 📖 Zamanlama saldırısı burada niye sorun değil?

Sırları karşılaştırırken `a == b` yazmak tehlikeli olabilir: dize karşılaştırması ilk farklı karakterde durur, bu yüzden "ilk 3 karakteri doğru" tahmin "ilk karakteri yanlış" tahminden ölçülebilir biçimde daha yavaştır. Saldırgan sırrı karakter karakter çözebilir. Önlemi `CryptographicOperations.FixedTimeEquals`.

Burada **o durum yok**, çünkü karşılaştırmayı biz bellekte yapmıyoruz — veritabanına *"şu hash'e sahip satırı getir"* diyoruz. Bu bir indeks araması; süresi hash'in içeriğine değil ağaç derinliğine bağlı ve pratikte sızdırmıyor.

> Kural: **sırrı bellekte elle karşılaştırıyorsanız** `FixedTimeEquals`; **indeksli bir sütunda arıyorsanız** gerek yok. Ezberlemek yerine "karşılaştırmayı kim, nerede yapıyor?" diye sormak yeterli.

### 5.5 Yeni şifre — form

```csharp
    // ══════════════════════════════════════════════════════
    //  YENİ ŞİFRE — FORM
    //  GET: /sifre-sifirla?token=xK3n...
    // ══════════════════════════════════════════════════════
    [HttpGet("/sifre-sifirla")]
    public async Task<IActionResult> SifreSifirla(string? token)
    {
        ViewData["Title"] = "Yeni şifre belirle";

        var kayit = await TokenBulAsync(token);

        if (kayit == null)
        {
            ViewBag.Gecersiz = true;
            return View(new SifreSifirlaViewModel());
        }

        // ⚠️ Token'ı forma gizli alan olarak taşıyoruz.
        //    Adres çubuğunda bırakıp POST'ta oradan okumak da
        //    çalışırdı ama gizli alan daha açık.
        return View(new SifreSifirlaViewModel { Token = token! });
    }
```

### 5.6 Yeni şifre — işle

```csharp
    // ══════════════════════════════════════════════════════
    //  YENİ ŞİFRE — İŞLE
    //  POST: /sifre-sifirla
    // ══════════════════════════════════════════════════════
    [HttpPost("/sifre-sifirla")]
    [ValidateAntiForgeryToken]
    public async Task<IActionResult> SifreSifirla(SifreSifirlaViewModel model)
    {
        ViewData["Title"] = "Yeni şifre belirle";

        // ⚠️ Token boşsa bu bir form hatası değil, bozuk bir bağlantıdır.
        //    "Bağlantı bilgisi eksik" diye kırmızı yazı göstermek yerine
        //    doğrudan "bu bağlantı geçersiz" ekranına düşürüyoruz.
        if (string.IsNullOrWhiteSpace(model.Token))
        {
            ViewBag.Gecersiz = true;
            return View(model);
        }

        if (!ModelState.IsValid)
            return View(model);

        // ⭐ Token'ı TEKRAR doğruluyoruz.
        //    GET'teki doğrulama yetmez: araya kullanıcının formu
        //    doldurduğu süre girdi, bu arada süre dolmuş veya başka
        //    bir sekmede kullanılmış olabilir.
        var kayit = await TokenBulAsync(model.Token);

        if (kayit == null)
        {
            ViewBag.Gecersiz = true;
            return View(model);
        }

        var kullanici = kayit.Kullanici!;

        // ── 1) Şifreyi değiştir ──────────────────────────────
        kullanici.SifreHash = SifreYardimcisi.Hashle(model.YeniSifre);

        // ── 2) Açık oturumları düşür ─────────────────────────
        //
        // ⭐ Damga değişince, eski çerezlerdeki damga tutmaz.
        //    Şifre sızdıysa saldırganın açık oturumu da kapanır.
        //    Bunu işleten kod Adım 6'da.
        kullanici.GuvenlikDamgasi = Guid.NewGuid().ToString();

        // ── 3) Bu token'ı harca ──────────────────────────────
        kayit.KullanildiTarih = DateTime.Now;

        // ── 4) Bekleyen DİĞER token'ları da iptal et ─────────
        //
        // ⚠️ Kullanıcı üst üste üç kere "şifremi unuttum" demiş
        //    olabilir. Üç bağlantı da geçerliyse ve biri eski bir
        //    postadan sızmışsa, şifre değiştikten SONRA bile
        //    tekrar değiştirilebilirdi.
        await _db.SifreSifirlamalari
            .Where(s => s.KullaniciId == kullanici.KullaniciId
                     && s.KullanildiTarih == null)
            .ExecuteUpdateAsync(g => g.SetProperty(
                s => s.KullanildiTarih, DateTime.Now));

        await _db.SaveChangesAsync();

        _gunluk.LogInformation("Şifre sıfırlandı: {kullanici} (IP: {ip})",
            kullanici.KullaniciAdi, HttpContext.Connection.RemoteIpAddress);

        // ── 5) Bu tarayıcıda oturum açıksa onu da kapat ──────
        await HttpContext.SignOutAsync(CookieAuthenticationDefaults.AuthenticationScheme);

        TempData["Basarili"] = "Şifreniz değiştirildi. Yeni şifrenizle giriş yapabilirsiniz.";
        return RedirectToAction(nameof(Giris));
    }
}
```

### ⚠️ `ExecuteUpdateAsync` ile `SaveChangesAsync` sırası

```csharp
await _db.SifreSifirlamalari.Where(...).ExecuteUpdateAsync(...);   // önce
await _db.SaveChangesAsync();                                      // sonra
```

Bu iki satır **farklı mekanizmalar**:

| | Nasıl çalışır |
|---|---|
| `ExecuteUpdateAsync` | Doğrudan `UPDATE` SQL'i gönderir, change tracker'a uğramaz. **Hemen** çalışır. |
| `SaveChangesAsync` | Takip edilen nesnelerdeki (`kullanici`, `kayit`) değişiklikleri yazar. |

`ExecuteUpdateAsync` bizim `kayit` nesnemizi de veritabanında günceller ama bellekteki kopyasını bilmez. Burada sorun çıkmıyor çünkü ikisi de aynı değeri (`KullanildiTarih`) yazıyor.

### ⚠️ Sıra tesadüf değil — hangi başarısızlık daha az zararlı?

İki satır arasında uygulama çökerse ne olur? Sıraya göre iki farklı sonuç çıkıyor:

| Sıra | Çökünce ne olur | Sonuç |
|---|---|---|
| **Token iptali → şifre** *(bizimki)* | Token'lar yanmış, şifre eski kalmış | Kullanıcı yeni bağlantı ister. **Can sıkıcı ama güvenli.** |
| Şifre → token iptali | Şifre değişmiş, eski bağlantılar hâlâ geçerli | Sızmış eski bir bağlantıyla şifre **tekrar** değiştirilebilir. **Güvenlik açığı.** |

İkisi de hatalı sonuç veriyor, ama biri sadece kullanıcıyı yoruyor, diğeri kapıyı açık bırakıyor.

> **İlke:** Bir işlem yarıda kalabiliyorsa, **kapalı tarafta kalacak** şekilde sıralayın. Buna *fail closed* denir; kapıların elektrik kesilince kilitli kalması gibi.

Doğru çözüm tabii ki ikisini **tek bir işleme (transaction)** almak — o zaman "yarım kalmış" bir durum hiç oluşmaz:

```csharp
await using var islem = await _db.Database.BeginTransactionAsync();
// ... iki işlem de burada
await islem.CommitAsync();
```

Alıştırma 4'te bunu yapacaksınız.

---

## ⌨️ Adım 6 — Açık oturumları düşürmek

Şifre değişti. Peki şifreyi ele geçirmiş olan kişinin **zaten açık** olan oturumu?

Çerez tabanlı kimlik doğrulamada çerez **kendi kendine yeten** bir belgedir: imzalıdır, içinde kullanıcı bilgileri vardır, sunucu onu doğrulamak için veritabanına bakmaz. Hızlı olmasının sebebi bu. Ama bedeli şu: **sunucu bir çerezi geri alamaz.** 14 gün geçerliyse 14 gün geçerlidir.

Çözüm: çereze bir **güvenlik damgası** koymak ve her istekte veritabanındakiyle karşılaştırmak. Şifre değişince damga yenilenir, eski çerezler düşer.

### 6.1 Girişte damgayı çereze koyun

`HesapController.Giris` içindeki iddia listesine bir satır:

```csharp
var iddialar = new List<Claim>
{
    new Claim(ClaimTypes.NameIdentifier, kullanici.KullaniciId.ToString()),
    new Claim(ClaimTypes.Name, kullanici.AdSoyad),
    new Claim("KullaniciAdi", kullanici.KullaniciAdi),
    new Claim("GuvenlikDamgasi", kullanici.GuvenlikDamgasi)   // ⭐ yeni
};
```

### 6.2 Her istekte doğrulayın

`Program.cs`'teki `.AddCookie(...)` bloğunun içine:

```csharp
.AddCookie(secenekler =>
{
    secenekler.LoginPath = "/giris";
    secenekler.LogoutPath = "/cikis";
    secenekler.ReturnUrlParameter = "donusUrl";
    secenekler.ExpireTimeSpan = TimeSpan.FromDays(14);
    secenekler.SlidingExpiration = true;
    secenekler.Cookie.HttpOnly = true;
    secenekler.Cookie.SameSite = SameSiteMode.Lax;
    secenekler.Cookie.Name = "KisiselBlog.Oturum";

    // ⭐ YENİ — çerez her kullanıldığında çalışır
    secenekler.Events = new CookieAuthenticationEvents
    {
        OnValidatePrincipal = async baglam =>
        {
            // ⚠️ Bu olay istek kapsamının İÇİNDE çalışır — scoped
            //    BlogDbContext'i almamız bu yüzden mümkün.
            //    Constructor enjeksiyonu kullanamıyoruz çünkü bu bir
            //    lambda; DI tarafından inşa edilen bir sınıf değil,
            //    dolayısıyla yapıcı metodu yok.
            var db = baglam.HttpContext.RequestServices
                          .GetRequiredService<BlogDbContext>();

            string? kimlikMetni = baglam.Principal?
                .FindFirst(ClaimTypes.NameIdentifier)?.Value;
            string? cerezDamgasi = baglam.Principal?
                .FindFirst("GuvenlikDamgasi")?.Value;

            // Eski bir çerezde damga iddiası hiç olmayabilir
            if (!long.TryParse(kimlikMetni, out long kullaniciId)
                || string.IsNullOrEmpty(cerezDamgasi))
            {
                // ⚠️ SignOutAsync'i unutmayın: sadece RejectPrincipal
                //    deseydiniz bayat çerez tarayıcıda kalır ve her
                //    istekte yeniden reddedilirdi.
                baglam.RejectPrincipal();
                await baglam.HttpContext.SignOutAsync(
                    CookieAuthenticationDefaults.AuthenticationScheme);
                return;
            }

            string? guncelDamga = await db.Kullanicilar
                .Where(k => k.KullaniciId == kullaniciId)
                .Select(k => k.GuvenlikDamgasi)
                .FirstOrDefaultAsync();

            // Kullanıcı silinmiş VEYA damga değişmiş → çerez geçersiz
            if (guncelDamga is null || guncelDamga != cerezDamgasi)
            {
                baglam.RejectPrincipal();
                await baglam.HttpContext.SignOutAsync(
                    CookieAuthenticationDefaults.AuthenticationScheme);
            }
        }
    };
});
```

`Program.cs`'in başına gereken using'ler:

```csharp
using Microsoft.AspNetCore.Authentication;        // SignOutAsync
using System.Security.Claims;                     // ClaimTypes
```

### 📖 Her istekte veritabanı sorgusu — pahalı değil mi?

Dürüst cevap: **evet, bir maliyeti var** ve bilerek kabul ediyoruz.

Ölçelim: bu olay sadece **kimliği doğrulanmış** isteklerde çalışır. Bir kişisel blogda kimlik doğrulaması yapan tek kişi var — siz. Okuyucular anonim, onların isteklerinde hiç çalışmıyor. Yani günde birkaç yüz ek sorgu; hiç önemi yok.

Ölçek büyüyünce durum değişir. ASP.NET Core Identity aynı mekanizmayı kullanır (`SecurityStampValidator`) ama **30 dakikada bir** doğrular, her istekte değil. Son doğrulama zamanını çerezin kendi içinde taşır.

> **Takas net:** anında iptal (her istek) ↔ düşük maliyet (aralıklı kontrol). Hangisini seçeceğiniz, çalınmış bir oturumun ne kadar süre açık kalmasına razı olduğunuza bağlı.

---

## ⌨️ Adım 7 — Ortak hesap layout'u

Artık dört hesap sayfamız var: giriş, şifremi unuttum, gönderildi, yeni şifre. Dördü de aynı ortalanmış kart düzenini kullanıyor.

Modül 4'te kuralı söylemiştik: **üçüncü tekrarda ortak dosyaya taşıyın.** Dördüncüdeyiz.

`Views/Shared/_HesapLayout.cshtml`:

```html
@inject SiteAyarlari Site

<!DOCTYPE html>
<html lang="tr">
<head>
    <meta charset="utf-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>@ViewData["Title"] — @Site.Baslik</title>
    <meta name="robots" content="noindex, nofollow" />

    @* ⭐ Sıfırlama bağlantısı adres çubuğunda token taşıyor.
         Bu sayfadan dışarı bir bağlantıya tıklanırsa tarayıcı
         Referer başlığında TÜM adresi, yani token'ı gönderir.
         no-referrer bunu engeller. *@
    <meta name="referrer" content="no-referrer" />

    <link href="https://cdn.jsdelivr.net/npm/bootstrap@@5.3.3/dist/css/bootstrap.min.css"
          rel="stylesheet">
    <link href="https://cdn.jsdelivr.net/npm/bootstrap-icons@@1.11.3/font/bootstrap-icons.css"
          rel="stylesheet">
</head>
<body class="bg-body-tertiary">

<div class="container min-vh-100 d-flex align-items-center justify-content-center py-5">
    <div style="width: 100%; max-width: 22rem;">

        <div class="text-center mb-4">
            <i class="bi bi-journal-text fs-1 text-secondary"></i>
            <h1 class="h4 mt-2 mb-1">@Site.Baslik</h1>
            <p class="text-secondary small mb-0">@ViewData["AltBaslik"]</p>
        </div>

        <div class="card shadow-sm">
            <div class="card-body p-4">
                @RenderBody()
            </div>
        </div>

        <p class="text-center mt-3 mb-0 small">
            <a asp-controller="Anasayfa" asp-action="Index" class="text-secondary">
                <span aria-hidden="true">&larr;</span> Siteye dön
            </a>
        </p>

    </div>
</div>

@* ⭐ jQuery, Scripts bölümünden ÖNCE — _ValidationScriptsPartial ona bağımlı.
     Gerekçesi Modül 3, Adım 4'te. *@
<script src="~/lib/jquery/dist/jquery.min.js"></script>

@await RenderSectionAsync("Scripts", required: false)

</body>
</html>
```

### ⚠️ `<meta name="referrer" content="no-referrer">` — niye şart?

Sıfırlama sayfasının adresi şu:

```
https://site.com/sifre-sifirla?token=xK3nQ9vL2pR7mT4wY8cZ1aB5dF6gH0jN
```

Kullanıcı bu sayfadayken dışarıdaki bir bağlantıya tıklarsa, tarayıcı hedef siteye bir `Referer` başlığı gönderir. Soru şu: o başlıkta **ne kadarı** var?

Bugünkü Chrome, Firefox ve Edge varsayılan olarak `strict-origin-when-cross-origin` uygular: başka bir siteye giderken yalnızca **köken** (`https://site.com`) gider, yol ve sorgu dizesi gitmez. Yani token zaten sızmaz.

**O hâlde niye `no-referrer` yazıyoruz?** Üç sebeple:

1. Varsayılana bel bağlamamak için. Bu varsayılan 2020'de geldi; eski tarayıcılar ve bazı gömülü web görünümleri tam URL gönderiyor.
2. **Aynı kaynaklı** isteklerde varsayılan hâlâ tam URL'i gönderir — kendi sayfalarımız arasında token dolaşır.
3. Bir sunucu veya CDN `Referrer-Policy: unsafe-url` yollarsa varsayılan eziliyor.

`no-referrer` bu başlığı tamamen kapatır. Güvenliği **varsayıma** değil, açık bir karara bağlamış oluyoruz.

> Bu, token'ı URL'de taşımanın bedellerinden biri. Diğerleri: tarayıcı geçmişinde kalması, paylaşılan ekran görüntülerinde görünmesi, ara sunucu loglarına yazılması. Hepsinin panzehiri aynı: **kısa ömür + tek kullanım.** Bu yüzden 60 dakika ve tek kullanım kuralları pazarlık konusu değil.

### Giriş sayfasını bu layout'a taşıyın

`Views/Hesap/Giris.cshtml`'in başındaki `Layout = null` ve `<!DOCTYPE html>` … `</html>` kabuğunu silin, geriye sadece form kalsın:

```html
@model KisiselBlog.Models.GirisViewModel
@{
    ViewData["Title"] = "Giriş";
    ViewData["AltBaslik"] = "Yönetim paneline giriş";
    Layout = "_HesapLayout";
}

@if (TempData["Basarili"] != null)
{
    <div class="alert alert-success py-2 small">
        <i class="bi bi-check-circle me-1"></i> @TempData["Basarili"]
    </div>
}

<form asp-action="Giris" method="post">

    <input type="hidden" name="donusUrl" value="@ViewBag.DonusUrl" />

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

    <div class="d-flex justify-content-between align-items-center mb-3">
        <div class="form-check mb-0">
            <input asp-for="BeniHatirla" type="checkbox" class="form-check-input" />
            <label asp-for="BeniHatirla" class="form-check-label">Beni hatırla</label>
        </div>

        @* ⭐ YENİ *@
        <a asp-action="SifremiUnuttum" class="small">Şifremi unuttum</a>
    </div>

    <button type="submit" class="btn btn-primary w-100">Giriş yap</button>

</form>

@* ⚠️ CANLIYA ÇIKMADAN ÖNCE SİLİN *@
<div class="alert alert-warning py-2 small mt-4 mb-0">
    <strong>Ders ortamı:</strong> admin / blog2026
</div>

@section Scripts {
    <partial name="_ValidationScriptsPartial" />
}
```

---

## ⌨️ Adım 8 — Kalan üç görünüm

### `Views/Hesap/SifremiUnuttum.cshtml`

```html
@model KisiselBlog.Models.SifremiUnuttumViewModel
@{
    ViewData["Title"] = "Şifremi unuttum";
    ViewData["AltBaslik"] = "Sıfırlama bağlantısı gönderelim";
    Layout = "_HesapLayout";
}

<p class="small text-secondary">
    Hesabınızın e-posta adresini yazın. Kayıtlıysa yeni şifre belirlemeniz için
    bir bağlantı göndereceğiz.
</p>

<form asp-action="SifremiUnuttum" method="post">

    <div class="mb-3">
        <label asp-for="Eposta" class="form-label"></label>
        <input asp-for="Eposta" class="form-control"
               autocomplete="email" autofocus
               placeholder="siz@@ornek.com" />
        <span asp-validation-for="Eposta" class="text-danger small"></span>
    </div>

    <button type="submit" class="btn btn-primary w-100">
        Sıfırlama bağlantısı gönder
    </button>

</form>

<p class="text-center mt-3 mb-0 small">
    <a asp-action="Giris">Giriş ekranına dön</a>
</p>

@section Scripts {
    <partial name="_ValidationScriptsPartial" />
}
```

### `Views/Hesap/SifremiUnuttumGonderildi.cshtml`

```html
@{
    ViewData["Title"] = "Bağlantı gönderildi";
    ViewData["AltBaslik"] = "Gelen kutunuzu kontrol edin";
    Layout = "_HesapLayout";
}

<div class="text-center">

    <i class="bi bi-envelope-check fs-1 text-success"></i>

    <p class="mt-3 mb-3">
        Bu adres kayıtlıysa bir sıfırlama bağlantısı gönderdik.
    </p>

    @* ⭐ Dikkat: "gönderdik" DEMİYORUZ, "kayıtlıysa gönderdik" diyoruz.
         Adresin sistemde olup olmadığını ele vermeyen tek ifade bu. *@

    <p class="small text-secondary mb-0">
        Birkaç dakika içinde gelmezse <strong>spam klasörünü</strong> ve
        adresi doğru yazdığınızı kontrol edin.
        Bağlantı @KisiselBlog.Data.SifirlamaTokeni.GecerlilikDakika dakika geçerlidir.
    </p>

</div>

<hr class="my-4">

<p class="text-center mb-0 small">
    <a asp-action="Giris">Giriş ekranına dön</a>
</p>
```

### `Views/Hesap/SifreSifirla.cshtml`

```html
@model KisiselBlog.Models.SifreSifirlaViewModel
@{
    ViewData["Title"] = "Yeni şifre belirle";
    ViewData["AltBaslik"] = "Yeni şifrenizi seçin";
    Layout = "_HesapLayout";

    bool gecersiz = ViewBag.Gecersiz == true;
}

@if (gecersiz)
{
    <div class="text-center">

        <i class="bi bi-x-circle fs-1 text-danger"></i>

        <p class="mt-3 mb-2"><strong>Bu bağlantı artık geçerli değil.</strong></p>

        <p class="small text-secondary">
            Bağlantıların ömrü @KisiselBlog.Data.SifirlamaTokeni.GecerlilikDakika
            dakikadır ve yalnızca bir kez kullanılabilir. Yeni bir tane isteyin.
        </p>

        <a asp-action="SifremiUnuttum" class="btn btn-primary w-100 mt-2">
            Yeni bağlantı iste
        </a>

    </div>
}
else
{
    <form asp-action="SifreSifirla" method="post">

        @* ⭐ Token gizli alanda taşınıyor *@
        <input asp-for="Token" type="hidden" />

        <div class="mb-3">
            <label asp-for="YeniSifre" class="form-label"></label>
            <input asp-for="YeniSifre" class="form-control"
                   autocomplete="new-password" autofocus />
            <span asp-validation-for="YeniSifre" class="text-danger small"></span>
            <div class="form-text">En az 8 karakter.</div>
        </div>

        <div class="mb-3">
            <label asp-for="YeniSifreTekrar" class="form-label"></label>
            <input asp-for="YeniSifreTekrar" class="form-control"
                   autocomplete="new-password" />
            <span asp-validation-for="YeniSifreTekrar" class="text-danger small"></span>
        </div>

        <button type="submit" class="btn btn-primary w-100">Şifreyi değiştir</button>

    </form>
}

@section Scripts {
    <partial name="_ValidationScriptsPartial" />
}
```

> **`autocomplete` değerleri boşuna değil.** `current-password` tarayıcıya "kayıtlı şifreyi doldur" der, `new-password` ise "doldurma, hatta güçlü bir şifre öner" der. Yanlışını yazarsanız şifre yöneticisi eski şifreyi yeni şifre alanına doldurur.

---

## ▶️ Çalıştır ve gör

```bash
dotnet ef database update
dotnet run
```

| # | Test | Beklenen |
|---|---|---|
| 1 | `/giris` | "Şifremi unuttum" bağlantısı görünüyor |
| 2 | Bağlantıya tıkla | `/sifremi-unuttum` açılıyor |
| 3 | Kayıtlı adresi gönder | "Bu adres kayıtlıysa…" sayfası |
| 4 | Mailtrap / gelen kutusu | Posta geldi, düğme ve düz adres var |
| 5 | **Kayıtsız** bir adres gönder | **Aynı** sayfa, aynı metin |
| 6 | SSMS → `SifreSifirlamalari` | Satır var; `TokenHash` 64 karakter, ham token **yok** |
| 7 | Postadaki düğmeye tıkla | Yeni şifre formu açılıyor |
| 8 | Şifreleri farklı yaz | "Şifreler eşleşmiyor" |
| 9 | 5 karakterlik şifre | "En az 8 karakter olmalı" |
| 10 | Geçerli şifre gir | Giriş sayfası + yeşil "Şifreniz değiştirildi" |
| 11 | **Eski** şifreyle gir | Başarısız |
| 12 | **Yeni** şifreyle gir | Panel açılıyor |
| 13 | **Aynı bağlantıya tekrar tıkla** | "Bu bağlantı artık geçerli değil" |
| 14 | Adresteki token'ın son harfini değiştir | Aynı hata |
| 15 | Token'sız `/sifre-sifirla` | Aynı hata |
| 16 | Üst üste 4 kez istek gönder | 4.'sünde posta gelmiyor; ekran aynı; log'da "Hız sınırı aşıldı" |
| 17 | Giriş yapmışken `/sifremi-unuttum` | Panele yönleniyor |

### 18. Süre dolması testi

60 dakika beklemeden sınamak için SSMS'te satırı geriye alın:

```sql
UPDATE SifreSifirlamalari
SET SonKullanma = DATEADD(MINUTE, -1, GETDATE())
WHERE KullanildiTarih IS NULL;
```

Bağlantıya tıklayın → "artık geçerli değil".

### 19. ⭐ Oturum düşürme testi — derste mutlaka yapın

Bu, Adım 6'nın neden var olduğunu gösteren test:

1. **Chrome'da** `admin` ile giriş yapın, panelde kalın
2. **Firefox'ta** (veya gizli pencerede) `/sifremi-unuttum` → şifreyi sıfırlayın
3. Chrome'a dönüp herhangi bir panel sayfasını **yenileyin**

Giriş ekranına atılmalısınız. Adım 6'yı yapmasaydınız Chrome'daki oturum **14 gün** daha açık kalırdı.

**Sor:** *"Bu niye önemli? Şifreyi zaten ben değiştirdim."*

Senaryo şu: şifrenizi birisi ele geçirdi ve sizin haberiniz olmadan giriş yaptı. Siz şifreyi sıfırlıyorsunuz. Oturum düşürme olmadan o kişi hiçbir şey kaybetmez — çerezi çalışmaya devam eder.

---

## ⚠️ Sık yapılan hatalar

| Hata | Sebep | Çözüm |
|---|---|---|
| `AuthenticationException` / `SocketException` | SMTP sunucu adı, port veya TLS seçeneği yanlış | 587 → `StartTls`, 465 → `SslOnConnect` |
| Gmail'de `535 Username and Password not accepted` | Normal hesap şifresi kullanılmış | 2 adımlı doğrulama + **uygulama şifresi** |
| `Eposta:Sifre` boş geliyor | `dotnet user-secrets set` çalıştırılmamış, ya da ortam Development değil | `dotnet user-secrets list` ile kontrol edin |
| Posta gitti ama gelmedi | Spam klasörü; ya da `GonderenAdres` ile SMTP hesabı farklı | İkisini aynı yapın |
| `InvalidOperationException: E-posta ayarları eksik` | `Eposta` bölümü veya `Eposta:Sifre` eksik — **ve** konsol yedeğini kurmamışsınız | Adım 2.3, ya da 2.6'nın sonundaki yedek |
| Bağlantı `http://localhost` diyor | `Request.Scheme` http | `app.UseHttpsRedirection()` açık mı? |
| `Url.Action` null dönüyor | Action adı yanlış yazılmış | `nameof(SifreSifirla)` kullanın |
| Token her zaman geçersiz | `Hashle` POST'ta çağrılmamış, ham token aranıyor | `TokenBulAsync` hash'liyor olmalı |
| Token URL'de bozuluyor | Düz `Convert.ToBase64String` kullanılmış | `WebEncoders.Base64UrlEncode` |
| `NullReferenceException` — `kayit.Kullanici` | `Include(s => s.Kullanici)` yok | Ekleyin |
| Migration `Eposta` için patlıyor | Tabloda birden fazla satır var, hepsi `""` oluyor | Sütunu önce nullable ekleyin (Adım 1.5) |
| Her migration'da aynı `GuvenlikDamgasi` değişiyor | `HasData`'da `Guid.NewGuid()` yazılmış | Sabit dize yazın |
| Şifre değişince kendi oturumum düşmüyor | `OnValidatePrincipal` eklenmemiş veya claim yok | Adım 6.1 ve 6.2 |
| Giriş yaptıktan sonra **hemen** atılıyorum | Girişte `GuvenlikDamgasi` claim'i eklenmemiş, doğrulama onu bulamıyor | Adım 6.1 |
| `_HesapLayout` bulunamıyor | `Views/Shared/` içinde değil | Taşıyın |
| "Şifreler eşleşmiyor" sadece gönderince çıkıyor | jQuery yüklenmemiş | `_HesapLayout`'ta `jquery.min.js`, `RenderSectionAsync`'ten önce |
| Razor `RZ1011` hatası | CDN adresinde tek `@` | `bootstrap@@5.3.3` |

---

## 📝 Özet

| Dosya | İşlevi |
|---|---|
| `Models/Kullanici.cs` | `Eposta` + `GuvenlikDamgasi` + iki ViewModel |
| `Models/SifreSifirlama.cs` | ⭐ Token kayıtları (hash, süre, kullanım) |
| `Models/EpostaAyarlari.cs` | SMTP yapılandırması |
| `Data/IEpostaGonderici.cs` | ⭐ Gönderim soyutlaması |
| `Data/SmtpEpostaGonderici.cs` | ⭐ MailKit uygulaması |
| `Data/SifirlamaTokeni.cs` | ⭐ Üretim ve hash'leme |
| `Controllers/HesapController.cs` | 4 yeni action |
| `Views/Shared/_HesapLayout.cshtml` | ⭐ Dört hesap sayfasının ortak kabuğu |
| `Views/Hesap/SifremiUnuttum.cshtml` | Form |
| `Views/Hesap/SifremiUnuttumGonderildi.cshtml` | Onay |
| `Views/Hesap/SifreSifirla.cshtml` | Yeni şifre |
| `Program.cs` | E-posta DI + `OnValidatePrincipal` |

**Öğrenilen kavramlar:**

- Dolu bir veritabanına `NOT NULL` sütun eklemek ve `defaultValue` tuzağı
- `RandomNumberGenerator` ile `Random` farkı
- Base64Url ve URL'de güvenli taşıma
- Sırrın kendisini değil hash'ini saklamak
- Entropi: hangi sır için yavaş, hangisi için hızlı hash
- Tek kullanımlık + süreli token
- Kullanıcı sayımını önlemenin üçüncü uygulaması
- Zamanlama sızıntısı: nerede gerçek, nerede değil
- MailKit, StartTls, `multipart/alternative` ile çift gövdeli posta
- Bir EF uyarısını okumak, anlamak ve bilerek susturmak
- User secrets ve yapılandırma kaynaklarının önceliği
- `Referrer-Policy` ile URL'deki sırrı korumak
- Host başlığı zehirlenmesi ve `AllowedHosts`
- Çerez tabanlı oturumu sunucudan iptal etmek (`OnValidatePrincipal`)
- `ExecuteDeleteAsync` / `ExecuteUpdateAsync`
- Soft delete'in her tabloya uymaması; denetim kayıtlarının farkı

---

## ⚠️ Gerçekte eksik olan

Bu sistem ders düzeyinde doğru çalışıyor. Canlıya çıkmadan önce şunlar da olmalı:

| Eksik | Neden |
|---|---|
| **İşlem (transaction)** | Şifre güncellemesi ile token iptali tek parça olmalı |
| **Arka plan kuyruğu** | Zamanlama sızıntısını kapatır, SMTP yavaşlığını kullanıcıdan gizler |
| **IP bazlı hız sınırı** | Kayıtsız adreslere yapılan denemeleri de sınırlar (`AddRateLimiter`) |
| **BCrypt / Argon2** | `SifreYardimcisi` hâlâ SHA256 — Modül 2'deki uyarı geçerli |
| **"Şifren değişti" bildirimi** | Değişiklikten sonra kullanıcıya ikinci bir posta; saldırı olduysa haberi olur |
| **Sabit temel adres** | `Request.Host`'a güvenmemek; bağlantıyı ayardan kurmak |
| **Şifre gücü kontrolü** | 8 karakter alt sınır zayıf; sızmış şifre listesine bakmak (k-anonymity ile Have I Been Pwned API) gerçek projelerde yaygın |

> **Sor:** *"Altı eksik saydık. Bu kadar iş varken niye ASP.NET Core Identity kullanmıyoruz?"*
>
> Gerçek projede **kullanmalısınız** — Identity bunların hepsini kutudan çıkar çıkmaz getirir. Elle yazmamızın tek sebebi, Identity'nin arkasında ne olduğunu görmeniz. Mekanizmayı anlamadan kullanmak, yanlış yapılandırdığınızda sorunu teşhis edememek demektir.

---

## ✏️ Alıştırmalar

1. **Tüm akışı çalıştırın.** 19 test senaryosunu geçirin — özellikle 5, 13 ve 19'u.

2. **"Şifren değişti" bildirimi ekleyin.** Şifre başarıyla değiştikten sonra kullanıcıya ikinci bir e-posta gidin: *"Şifreniz az önce değiştirildi. Bu siz değilseniz hemen bizimle iletişime geçin."* İpucu: `EpostaHtmlOlustur` ve `EpostaDuzMetinOlustur` gibi ikişer yardımcı metot — iki gövde de gerekiyor.

3. **Panelden şifre değiştirme.** `/yonetim/sifre` adresinde, giriş yapmış kullanıcının **mevcut şifresini girerek** yeni şifre belirlediği bir sayfa yapın. Mevcut şifre kontrolünü atlamayın — biri bilgisayarınızı açık bulursa şifreyi değiştiremesin. Burada da `GuvenlikDamgasi` yenilenmeli, ama **kendi oturumunuz düşmemeli**: değişiklikten sonra `SignInAsync` ile yeni damgayla tekrar giriş yapın.

4. **İşleme (transaction) sarın.** Adım 5.6'daki şifre güncelleme + token iptali işlemlerini tek bir transaction içine alın:
   ```csharp
   await using var islem = await _db.Database.BeginTransactionAsync();
   // ... işlemler
   await islem.CommitAsync();
   ```
   Sonra `CommitAsync`'ten hemen önce `throw new Exception("test");` yazıp çalıştırın. Veritabanında ne değişti?

5. **Token ömrünü ayara taşıyın.** `GecerlilikDakika` sabitini `appsettings.json`'a alın. E-posta metnindeki ve görünümlerdeki sayıların da ayardan okunduğunu doğrulayın — üç yerde sabit kalmasın.

6. **Sızmış şifre kontrolü (ileri).** Have I Been Pwned'in Pwned Passwords API'si, şifrenin SHA1 hash'inin **ilk 5 karakterini** alıp o önekle başlayan tüm hash'leri döner. Yani şifreniz hiç ağdan geçmez (buna *k-anonymity* deniyor). `https://api.pwnedpasswords.com/range/{ilk5}` adresini çağırıp yeni şifrenin listede olup olmadığını kontrol edin.

7. **Bağlantıyı Host başlığından kurtarın.** `SiteAyarlari`'ne `TabanAdres` diye bir alan ekleyin (`"https://harunaksaya.blog"`). Doluysa bağlantıyı ondan kurun, boşsa `Request.Scheme`'e düşün:
   ```csharp
   string taban = !string.IsNullOrWhiteSpace(_site.TabanAdres)
       ? _site.TabanAdres.TrimEnd('/')
       : $"{Request.Scheme}://{Request.Host}";
   string baglanti = $"{taban}{Url.Action(nameof(SifreSifirla), "Hesap", new { token })}";
   ```
   Sonra `curl` ile sahte bir `Host` başlığı gönderip bağlantının değişmediğini doğrulayın:
   ```bash
   curl -k -X POST https://localhost:7123/sifremi-unuttum \
        -H "Host: saldirgan.com" -d "eposta=siz@ornek.com"
   ```

**Düşünme soruları:**

- Token'ı veritabanına ham saklasaydık, hangi saldırı mümkün olurdu? Hash'lemek onu nasıl engelliyor?
- Şifreler için BCrypt diyoruz ama token için SHA256 yeterli. Farkı tek cümleyle nasıl açıklarsınız?
- Token süresini 60 dakika yaptık. 5 dakika yapsak hangi sorun çıkar? 7 gün yapsak?
- `SifreSifirlama` tablosuna soft delete (`AktifMi`) koymadık. Niye bazı tablolarda var, bazılarında yok?
- Çerez "kendi kendine yeten bir belge" dedik. Bu özellik hem hızın hem de iptal sorununun sebebi. Oturumları veritabanında tutsaydık (server-side session) hangi takas değişirdi?

---

👈 Önceki: [`11-son-dokunuslar.md`](11-son-dokunuslar.md)
🏠 [`README.md`](README.md)
