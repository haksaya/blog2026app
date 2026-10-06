# Modül 1 — Bootstrap'i Kendimize Uyarlamak

**Süre:** 40 dakika
**Önkoşul:** Proje oluşturuldu, paketler eklendi (`00-PROJE-PLANI.md`)

---

## 🎯 Bu derste ne yapacağız

1. Bootstrap'i projeye bağlayıp hazır bileşenlerini kullanacağız
2. **Altı CSS değişkeniyle** siteyi kendimize ait hâle getireceğiz
3. "Niye uğraşıyoruz, hazır hâli yetmiyor mu?" sorusuna cevap vereceğiz

Ders sonunda `wwwroot/css/site.css` içinde **yaklaşık 95 satır** CSS olacak — yorumlarla birlikte. Projenin kalan tasarımı Bootstrap'in kendi sınıflarıyla yapılacak.

---

## 📖 Kavram — Niye kişiselleştiriyoruz?

Üç projede Bootstrap'i olduğu gibi kullandık ve doğru olanı yaptık. Bir görev takip ekranında, bir öğrenci bilgi sisteminde amaç **tanıdık olmak**: butonun mavi ve köşeli olması, tablonun çizgili olması iyidir. Kullanıcı o arayüzü daha önce yüz kere gördü, nereye basacağını düşünmeden biliyor. Oradaki başarı ölçütü "dikkat çekmemek"ti.

Kişisel sitede ölçüt değişiyor. Burası bir insanın adını taşıyan tek sayfa; ziyaretçi buraya bir şey yapmaya değil, **okumaya** geliyor. Ve Bootstrap'in varsayılan hâli tam olarak "bir yönetim panosu" gibi görünür — çünkü o işe göre ayarlanmış. Sistem yazı tipi, mavi bağlantılar, 16px gövde. Aynı varsayılanlarla kurulmuş milyonlarca sayfa var; ziyaretçi bunu fark etmese bile sayfanın size ait olmadığını hissediyor.

İyi haber: Bootstrap bunu bekliyor. Sürüm 5.3'ten beri kendi renklerini, yazı tiplerini ve kenarlıklarını **CSS değişkenleri** üzerinden okuyor. Yani Bootstrap'in kaynağını değiştirmeden, kendi dosyanızda altı satır yazarak bütün sitenin tonunu değiştirebiliyorsunuz. `btn`, `card`, `form-control` — hepsi çalışmaya devam ediyor, sadece sizin renginizde ve yazı tipinizde çalışıyor. Yazacağımız şey Bootstrap'in **yerine** değil, **üzerine** ince bir kat.

---

## ⌨️ Adım 1 — Bootstrap'i bağlamak

### 1.1 Şablondan gelen Bootstrap

`dotnet new mvc` projeyi oluştururken Bootstrap'i de indirir:

```
wwwroot/lib/bootstrap/dist/css/bootstrap.min.css
wwwroot/lib/bootstrap/dist/js/bootstrap.bundle.min.js
```

Bu dosyalar durabilir ama **sürümü eski** olabilir ve ikon seti yok. Biz CDN kullanacağız — hem güncel sürüm, hem ikonlar, hem de tarayıcı önbelleğinden gelme şansı.

### 1.2 Layout'un `<head>` kısmı

Modül 3'te layout'u baştan yazacağız. Şimdilik sadece hangi üç bağlantının gerektiğini bilin:

```html
<!-- 1 ▸ Yazı tipleri (Google Fonts) -->
<link rel="preconnect" href="https://fonts.googleapis.com">
<link rel="preconnect" href="https://fonts.gstatic.com" crossorigin>
<link href="https://fonts.googleapis.com/css2?family=Montserrat:wght@@700;800&family=Lato:wght@@400;700&display=swap"
      rel="stylesheet">

<!-- 2 ▸ Bootstrap -->
<link href="https://cdn.jsdelivr.net/npm/bootstrap@@5.3.3/dist/css/bootstrap.min.css"
      rel="stylesheet">
<link href="https://cdn.jsdelivr.net/npm/bootstrap-icons@@1.11.3/font/bootstrap-icons.css"
      rel="stylesheet">

<!-- 3 ▸ Bizim katmanımız — SIRA ÖNEMLİ, en sonda olmalı -->
<link rel="stylesheet" href="~/css/site.css" asp-append-version="true">
```

Ve `</body>` kapanmadan hemen önce:

```html
<script src="https://cdn.jsdelivr.net/npm/bootstrap@@5.3.3/dist/js/bootstrap.bundle.min.js"></script>
```

> ⚠️ **`@@` çift yazılıyor, dikkat.** Bu satırlar bir `.cshtml` dosyasına girecek ve Razor'da
> tek `@` kod başlatır. `bootstrap@5.3.3` yazarsanız `RZ1011` derleme hatası alırsınız.
> `@@` tarayıcıya tek bir `@` olarak gider. Modül 3'te bunu ayrıntısıyla göreceğiz.

**Satır satır ne yapıyor:**

| Satır | İşlevi |
|---|---|
| `preconnect` | Tarayıcı, yazı tipi sunucusuna CSS'i beklemeden bağlanmaya başlar. İlk görüntülenmeyi hızlandırır. |
| `family=Montserrat:wght@700;800` | Sadece 700 ve 800 kalınlıklarını indirir. Hepsini indirmek sayfayı yavaşlatır. |
| `&display=swap` | Yazı tipi inene kadar metni sistem yazı tipiyle gösterir. Olmazsa metin bir an **görünmez** kalır (FOIT). |
| `bootstrap-icons.css` | `<i class="bi bi-github"></i>` yazabilmek için. 2000'den fazla ikon. |
| `site.css` **en sonda** | CSS'te eşit özgüllükte son yazılan kazanır. Bootstrap'ten önce koyarsanız bizim kuralların hiçbiri işlemez. |
| `asp-append-version` | Dosya değişince adresin sonuna `?v=hash` ekler. Öğrenci "ben CSS'i değiştirdim ama ekran değişmiyor" derse sebep budur. |
| `bundle.min.js` | Dropdown, modal, navbar açılıp kapanması için. `bundle` sürümü Popper'ı içinde taşır. |

> **⚠️ İnternet yoksa:** CDN erişimi olmayan bir laboratuvarda `wwwroot/lib/bootstrap/` içindeki yerel dosyaları kullanın; sınıf adları birebir aynıdır. Yazı tipleri inmezse sayfa sistem yazı tipiyle görünür — düzen bozulmaz.

### 1.3 Bootstrap bize ne veriyor?

Projede kullanacağımız bileşenlerin tamamı bunlar. Yenisini öğrenmeyeceğiz:

| İhtiyaç | Bootstrap sınıfı | Nerede |
|---|---|---|
| İki sütunlu sayfa | `row` + `col-lg-3` / `col-lg-9` | Modül 3, 6 |
| Üst menü | `navbar navbar-expand-lg` | Modül 3 (panel) |
| Form alanı | `form-control`, `form-label`, `form-select` | Modül 4, 5, 9 |
| Buton | `btn btn-primary`, `btn-outline-secondary` | her yerde |
| Tablo | `table table-hover align-middle` | Modül 5, 8 |
| Durum etiketi | `badge text-bg-secondary` | Modül 5 |
| Uyarı mesajı | `alert alert-success alert-danger` | Modül 4, 9 |
| Kart | `card card-body` | Modül 3 (panel özetleri) |
| Boşluk/hizalama | `mb-3`, `p-4`, `d-flex`, `gap-3`, `text-center` | her yerde |

---

## ⌨️ Adım 2 — Altı değişkenle kişiselleştirme

Şimdi tek CSS dosyamızı yazıyoruz. **Tamamı aşağıda** — kopyalayın, sonra satırları açıklayacağız.

**`wwwroot/css/site.css`**

```css
/* ═══════════════════════════════════════════════════════════
   KisiselBlog — Bootstrap üzerine ince bir kat
   Bootstrap'in kendi CSS değişkenlerini eziyoruz.
   ═══════════════════════════════════════════════════════════ */

:root {
  /* ── Bootstrap'in değişkenleri ── */
  --bs-body-font-family: 'Lato', system-ui, sans-serif;
  --bs-body-font-size: 1.0625rem;      /* 17px — varsayılan 16px */
  --bs-body-color: #4a4a4a;            /* saf siyah değil, yumuşak gri */
  --bs-heading-color: #2b2b2b;         /* başlıklar gövdeden koyu */

  /* ⚠️ Bootstrap 5.3 bağlantı rengini --bs-link-color-rgb üzerinden
     hesaplıyor. Sadece --bs-link-color yazmak YETMEZ. */
  --bs-link-color: #8a6800;
  --bs-link-color-rgb: 138, 104, 0;
  --bs-link-hover-color: #6b5100;
  --bs-link-hover-color-rgb: 107, 81, 0;

  --bs-border-color: #e3e7ec;          /* çizgiler daha açık */

  /* ── Kendi değişkenlerimiz ── */
  --sari: #ffd43b;                     /* vurgu: çerçeve ve buton zemini */
  --yan-zemin: #f6f8fa;                /* yan sütunun zemini */
  --soluk: #6b6b6b;                    /* tarih, ikincil metin */
}

/* ── Başlıklar ayrı yazı tipi ── */
h1, h2, h3, .h1, .h2, .h3 {
  font-family: 'Montserrat', sans-serif;
  font-weight: 800;
  letter-spacing: -.02em;
  line-height: 1.15;
}

/* ── Okuma genişliği ── */
.icerik-alani { max-width: 46rem; }

/* ── Yan sütun ── */
.yan-sutun {
  background: var(--yan-zemin);
  border-right: 1px solid var(--bs-border-color);
}
@media (min-width: 992px) {
  .yan-sutun { position: sticky; top: 0; height: 100vh; overflow-y: auto; }
}

.profil-foto {
  width: 150px; height: 150px;
  border: 4px solid var(--sari);
  padding: 4px; background: #fff;
}

/* ── Küçük, aralıklı, büyük harf başlıklar ── */
.yan-baslik, .yazi-tarih, .devam {
  font-size: .8rem; font-weight: 700;
  text-transform: uppercase; letter-spacing: .09em;
}
.yan-baslik { font-size: .75rem; color: var(--soluk); margin-bottom: .75rem; }
.yazi-tarih { color: var(--soluk); }
.devam { text-decoration: none; }

/* ── Sosyal ikonlar ── */
.sosyal a { color: var(--soluk); font-size: 1.15rem; }
.sosyal a:hover { color: var(--bs-link-color); }

/* ── Etiketler ── */
.etiket-link {
  color: var(--soluk); text-decoration: none;
  font-size: .9rem; margin-right: .45rem; line-height: 1.9;
  white-space: nowrap;      /* "#yapay zekâ" ortadan bölünmesin */
}
.etiket-link:hover { color: var(--bs-link-color); }

/* ── Yazı başlığı bağlantısı sarı değil, koyu ── */
.baslik-link { color: var(--bs-heading-color); text-decoration: none; }
.baslik-link:hover { color: var(--bs-link-color); }

/* ── Sarı buton ── */
.btn-sari {
  background: var(--sari); border-color: var(--sari); color: #212529;
  font-weight: 700; font-size: .8rem;
  text-transform: uppercase; letter-spacing: .06em;
}
.btn-sari:hover { background: #f3c51f; border-color: #f3c51f; color: #212529; }

/* ── Alt çizgili arama kutusu ── */
.arama-kutu {
  border: none; border-bottom: 1px solid var(--bs-border-color);
  border-radius: 0; background: transparent; padding-left: 0;
}
.arama-kutu:focus {
  box-shadow: none; border-bottom-color: var(--sari); background: transparent;
}
```

### Hepsi bu kadar

~95 satır, yorumlar dahil. Bu dosyadan sonra yazacağımız **tek** ek CSS, Modül 6'daki Markdown gövdesi bloğu olacak (~45 satır) — çünkü Markdig'in ürettiği `<h2>`, `<pre>`, `<blockquote>` etiketlerinde Bootstrap sınıfı olmuyor.

Toplam: **~140 satır.** Kalan her şey Bootstrap sınıfı.

---

## 📖 Kavram — Bu değişkenler nasıl çalışıyor?

Bootstrap 5.3'ün kendi CSS'inin içi şöyle görünür:

```css
/* bootstrap.min.css — Bootstrap'in kendi kodu */
:root {
  --bs-body-font-family: var(--bs-font-sans-serif);
  --bs-body-color: #212529;
  ...
}
body {
  font-family: var(--bs-body-font-family);   /* ← değişkeni OKUYOR */
  color: var(--bs-body-color);
}
```

`body` kuralı sabit bir değer değil, **değişkeni** okuyor. Biz o değişkeni kendi dosyamızda yeniden tanımlayınca Bootstrap'in `body` kuralı yeni değeri okumaya başlıyor. Bootstrap'in dosyasına dokunmadık, `body` için kendi kuralımızı yazmadık — sadece değeri değiştirdik.

Aynı mantık `card`, `table`, `modal`, `dropdown` için de geçerli. Hepsi aynı değişkenleri okuyor. Bu yüzden altı satır bütün siteyi etkiliyor.

### ⚠️ Derste mutlaka gösterin: `--bs-link-color-rgb` tuzağı

"Devamını oku" bağlantısının sarı-kahve olmasını istiyoruz. Mantıklı olan şunu yazmak:

```css
:root {
  --bs-link-color: #8a6800;   /* ← tek başına İŞE YARAMAZ */
}
```

Kaydedin, yenileyin — bağlantı **hâlâ mavi.** Sebebi Bootstrap'in kaynağında:

```css
/* Bootstrap'in kendi kodu */
a {
  color: rgba(var(--bs-link-color-rgb), var(--bs-link-opacity, 1));
}
```

Bootstrap rengi `--bs-link-color`'dan değil, **`--bs-link-color-rgb`'den** okuyor. Böyle yapıyor ki `link-opacity-50` gibi sınıflarla saydamlık ayarlanabilsin — saydamlık için rengin R, G, B bileşenlerine ayrı ayrı erişmesi gerekiyor. Hex kodundan bunu yapamaz.

**Çözüm: ikisini birlikte yazmak.**

```css
--bs-link-color: #8a6800;
--bs-link-color-rgb: 138, 104, 0;    /* ← aynı rengin RGB karşılığı */
```

`#8a` = 138, `#68` = 104, `#00` = 0. Tarayıcı geliştirici araçlarında `Computed` sekmesine bakıp rengin nereden geldiğini okumak, bu tür takılmaları çözmenin standart yolu.

> Aynı tuzak `--bs-primary-rgb`, `--bs-body-color-rgb`, `--bs-danger-rgb` için de var. Kural: **Bootstrap'te bir renk değişmiyorsa `-rgb` eşini arayın.**

---

## 📖 Kavram — Renk seçiminde ölçülebilir olan kısım

Palet seçimi zevk işidir, ama bir kısmı değildir: **kontrast.**

WCAG AA standardı normal boyutlu metin için metin-zemin kontrast oranının en az **4.5:1** olmasını ister. Bu keyfi bir sayı değil; düşük görme, yaşa bağlı görme kaybı ve güneş altında telefon kullanma gibi durumların ölçümüyle belirlenmiş.

Paletimizin ölçülmüş değerleri:

| Rol | Renk | Zemin | Oran | AA |
|---|---|---|---|---|
| Başlık | `#2b2b2b` | beyaz | **14.16:1** | ✅ |
| Gövde | `#4a4a4a` | beyaz | **8.86:1** | ✅ |
| Soluk / tarih | `#6b6b6b` | beyaz | **5.33:1** | ✅ |
| Soluk / tarih | `#6b6b6b` | `#f6f8fa` | **5.01:1** | ✅ |
| Bağlantı | `#8a6800` | beyaz | **5.17:1** | ✅ |
| Buton yazısı | `#212529` | `#ffd43b` | **10.82:1** | ✅ |

### Sarı metin problemi

Vurgu rengimiz `#ffd43b` — canlı bir sarı. Beyaz zeminde **metin olarak** ölçüldüğünde:

```
#ffd43b / #ffffff  →  1.43:1        AA eşiği: 4.5:1
```

Üç kat uzağında. Bu sarıyla yazılmış bir "DEVAMINI OKU" bağlantısı, normal görüşlü biri için bile beyaz zeminde zor okunur.

**Çözüm sarıdan vazgeçmek değil, nerede kullanıldığını ayırmak:**

| Kullanım | Renk | Niye |
|---|---|---|
| Profil fotoğrafı çerçevesi | `#ffd43b` | Dekoratif — metin değil, kontrast kuralı işlemez |
| "Kaydol" butonunun **zemini** | `#ffd43b` | Zemin; üstündeki koyu yazı 10.82:1 veriyor |
| Arama kutusunun odak çizgisi | `#ffd43b` | Dekoratif |
| Bağlantı **metni** | `#8a6800` | Aynı sarının koyultulmuşu — 5.17:1 |

Göz hâlâ aynı sarıyı görüyor, okunabilirlik kazanılıyor. Tasarımda tipik bir karar: rengi değiştirmek yerine **rolünü** değiştirmek.

### Nasıl ölçülür?

| Yöntem | Nasıl |
|---|---|
| Tarayıcı | F12 → bir metne sağ tık → Inspect → renk kutusuna tıkla. Chrome oranı ve ✅/⚠️ işaretini gösterir. |
| Lighthouse | F12 → Lighthouse → Accessibility → Run. Tüm sayfayı tarar. |
| Web | WebAIM Contrast Checker — iki hex kodunu yazın |

> **Ders notu:** 10 dakikada bir Lighthouse taraması çalıştırıp sonucu tahtaya yansıtmak etkili oluyor. Öğrenci kendi seçtiği "çok hoş" açık gri metnin uyarı verdiğini görüyor.

---

## ▶️ Çalıştır ve gör

Layout'u henüz yazmadık, ama değişkenlerin çalıştığını şimdi test edebiliriz. `Views/Home/Index.cshtml` dosyasının içeriğini silip şunu yazın:

```html
@{
    ViewData["Title"] = "Tasarım testi";
}

<div class="container py-5 icerik-alani">

    <p class="yazi-tarih mb-2">6 EKİM 2026</p>
    <h1 class="mb-3">Tasarım testi</h1>

    <p>Bu paragraf <strong>Lato</strong> ile 17px görünmeli. Başlık ise
       <strong>Montserrat 800</strong> ile ve daha koyu olmalı.</p>

    <p>Bu bir <a href="#">bağlantı</a> — mavi değil, sarı-kahve olmalı.
       Mavi görünüyorsa <code>--bs-link-color-rgb</code> satırını yazmayı
       atlamışsınız.</p>

    <a href="#" class="devam">Devamını oku <span aria-hidden="true">&rarr;</span></a>

    <hr class="my-4">

    <button class="btn btn-sari rounded-0 px-3">Kaydol</button>
    <button class="btn btn-outline-secondary">İptal</button>

    <div class="card mt-4">
        <div class="card-body">
            <h3 class="h5">Hazır bileşen</h3>
            <p class="mb-0">Bu kart Bootstrap'in <code>card</code> sınıfı.
               Biz ona dokunmadık ama bizim yazı tipimizi ve renklerimizi
               kullanıyor.</p>
        </div>
    </div>

</div>
```

> ⚠️ Bu `<head>` bağlantılarını henüz layout'a eklemediyseniz sayfa biçimsiz görünür. Modül 3'te layout'u yazdığımızda hepsi yerine oturacak. Hemen görmek isterseniz yukarıdaki üç `<link>` satırını geçici olarak `Views/Shared/_Layout.cshtml`'in `<head>`'ine ekleyin.

```bash
dotnet run
```

**Ekranda görmeniz gerekenler:**

- [ ] Gövde metni Lato, biraz büyük (17px)
- [ ] "Tasarım testi" başlığı Montserrat, kalın, sıkışık harf aralıklı
- [ ] "bağlantı" kelimesi **sarı-kahve** (mavi değil)
- [ ] "DEVAMINI OKU" büyük harf, aralıklı, sarı-kahve
- [ ] "Kaydol" butonu sarı zemin, koyu yazı, köşesiz
- [ ] "İptal" butonu Bootstrap'in standart hâli — dokunmadık, öyle kalmalı
- [ ] Kart bizim yazı tipimizi kullanıyor
- [ ] Metin sayfanın tamamına yayılmıyor, `46rem`'de duruyor

Son maddeyi test etmek için tarayıcı penceresini genişletin: paragraflar bir noktadan sonra uzamayı bırakmalı. `.icerik-alani` sınıfını geçici olarak kaldırıp tekrar bakın — satırlar ekran boyunca uzayınca okumanın nasıl zorlaştığını görün. 60-75 karakter aralığı okuma araştırmalarında en rahat bulunan aralıktır.

---

## ⚠️ Sık yapılan hatalar

| Belirti | Sebep | Çözüm |
|---|---|---|
| **Hiçbir şey değişmedi** | `site.css`, Bootstrap'ten **önce** bağlanmış | `<link>` sırasını kontrol edin: Bootstrap önce, `site.css` sonra |
| **Bağlantı hâlâ mavi** | `--bs-link-color-rgb` yazılmamış | İki satırı birlikte yazın |
| **Yazı tipi gelmedi** | Google Fonts adresinde yazım hatası, ya da internet yok | F12 → Network → kırmızı satır var mı? |
| **CSS'i değiştirdim, ekran aynı** | Tarayıcı önbelleği | `Ctrl+F5`; `asp-append-version="true"` kalıcı çözüm |
| **Başlıklar hâlâ Lato** | `h1, h2, h3` kuralı dosyaya hiç yazılmamış, ya da `'Montserrat'` tırnakları unutulmuş | Kuralı ekleyin; yazı tipi adı tek tırnak içinde olmalı |
| **`bi bi-github` ikonu kare çıkıyor** | `bootstrap-icons.css` bağlanmamış | İkinci CDN satırını ekleyin |
| **Türkçe karakter bozuk** | `<meta charset="utf-8">` yok | Layout'un ilk satırlarına ekleyin |
| **Panelde de sarı butonlar var** | `.btn-sari` panel view'larında kullanılmış | Panelde Bootstrap'in standart `btn-primary`'si kalsın — ayrımı Modül 3'te anlatıyoruz |

---

## 📝 Özet

| Yaptığımız | Satır |
|---|---|
| Bootstrap + ikonlar + yazı tipleri bağlandı | 3 `<link>` + 1 `<script>` |
| Bootstrap değişkenleri ezildi | 9 satır |
| Kendi değişkenlerimiz | 3 satır |
| Yardımcı sınıflar | ~55 satır |

**Öğrenilen kavramlar:**

- Bootstrap 5.3 renk ve yazı tiplerini CSS değişkenlerinden okur; onları ezmek Bootstrap'i değiştirmekten farklıdır
- `-rgb` eşi olmayan renk değişikliği işlemez
- CSS'te sıra önemlidir — kendi dosyanız en sonda
- Kontrast ölçülebilir bir şeydir; 4.5:1 keyfi değil
- Bir rengin metin olarak mı, zemin olarak mı kullanıldığı okunabilirliği belirler

---

## ✏️ Alıştırmalar

1. **Palet değiştir.** `--sari` ve `--bs-link-color` değerlerini kendi seçtiğiniz bir renkle değiştirin. Bağlantı renginin `-rgb` eşini de yazmayı unutmayın. Chrome'un kontrast ölçerinde 4.5:1'i geçtiğini doğrulayın.

2. **Yazı tipi değiştir.** Google Fonts'tan başka bir başlık yazı tipi seçin (Playfair Display, Bitter, Oswald…). Türkçe karakterleri desteklediğini kontrol edin — Google Fonts'ta `Language` filtresinden `Turkish` seçebilirsiniz. "Ğ ğ İ ı Ş ş Ü ü Ö ö Ç ç" yazıp bakın.

3. **Okuma genişliğini sına.** `.icerik-alani`'nın `max-width` değerini `30rem`, `46rem` ve `80rem` yapıp aynı paragrafı üç kere okuyun. Hangisinde satır sonuna geldiğinizde gözünüz sıranın başını kaybediyor?

4. **Değişkeni izle.** F12 → bir `card`'a tıklayın → `Computed` sekmesi → `font-family` satırını açın. Değerin hangi dosyadan, hangi kuraldan geldiğini okuyun. Bootstrap'in kuralı mı görünüyor, bizimki mi?

5. **Karanlık mod denemesi (ileri).** `<html>` etiketine `data-bs-theme="dark"` ekleyip sayfayı yenileyin. Bootstrap'in kendi karanlık teması devreye girer. Hangi renklerimiz bozuldu? (İpucu: `--yan-zemin` ve `#fff` yazdığımız yerler.) Bunu düzeltmek bir sonraki adım olurdu — bu projede karanlık mod yapmıyoruz.

---

👉 Sıradaki: [`02-model-ve-veritabani.md`](02-model-ve-veritabani.md) — Yazı modeli, Türkçe slug üretimi ve migration
