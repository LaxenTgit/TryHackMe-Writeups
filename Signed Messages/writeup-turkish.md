# 🔐 Signed Messages — TryHackMe Writeup

![Zorluk](https://img.shields.io/badge/Zorluk-Orta-orange)
![Kategori](https://img.shields.io/badge/Kategori-Kriptografi%20|%20RSA%20|%20İmza%20Sahtekarlığı-blue)
![Platform](https://img.shields.io/badge/Platform-TryHackMe-green)

> **Yazar:** Miraç Akkuş (LatenT)  
> **Tarih:** Mayıs 2026  
> **Oda:** [Signed Messages](https://tryhackme.com/room/lafb2026e8)

---

## 📋 Yönetici Özeti

Bu write-up, **Signed Messages** makinesinin sömürülmesini belgelemektedir. Hatalı bir RSA implementasyonuna karşı kriptografik bir saldırıyı göstermektedir. Zafiyet, **tahmin edilebilir tohum (seed) tabanlı anahtar üretiminden** kaynaklanmakta olup, ortak anahtar parametrelerinden özel anahtarın yeniden yapılandırılmasına olanak tanımaktadır. Saldırı zinciri, `admin` kullanıcısı olarak imza sahtekarlığı yaparak doğrulama endpoint'inden flag'i almaya kadar ilerlemektedir.

---

## 🛠️ Kullanılan Araçlar ve Teknolojiler

| Araç | Amaç |
|------|------|
| `curl` | HTTP etkileşimi ve endpoint testi |
| `openssl` | RSA anahtar analizi |
| `python3` | Kriptografik hesaplamalar ve imza üretimi |
| `cryptography` kütüphanesi | RSA anahtar yapılandırması ve PSS dolgu |

---

## 🔍 Aşama 1: Keşif ve Enumerasyon

### Uygulama Endpoint'leri

```bash
curl -s http://<HEDEF_IP>:5000/
```

**Keşfedilen Endpoint'ler:**

| Endpoint | İşlev | Erişim |
|----------|-------|--------|
| `/register` | Kullanıcı kaydı ve RSA anahtar üretimi | Herkese açık |
| `/login` | Sadece kullanıcı adı ile kimlik doğrulama | Herkese açık |
| `/messages` | Herkese açık mesaj panosu | Herkese açık |
| `/compose` | Mesaj oluşturma | Kimliği doğrulanmış |
| `/verify` | Dijital imza doğrulama | Herkese açık |
| `/about` | Platform bilgisi | Herkese açık |

### Herkese Açık Mesaj Panosu (`/messages`)

- Ham imzalar veya anahtar verileri olmadan mesajlar görüntüleniyor
- Yöneticiden **karşılama duyurusu** tespit edildi — sahtekarlık hedef mesaj
- Kriptografik işlemler sunucu tarafında gerçekleştiriliyor

### İmza Doğrulama (`/verify`)

**Form Alanları:**
- `username` — Kullanıcı kimliği
- `message` — Düz metin içerik
- `signature` — Hex-kodlu dijital imza

**Kritik Gözlemler:**
- Durum banner'ı geçerli/geçersiz imza durumunu gösteriyor
- Gizli `flag-display` div'i sadece başarılı admin doğrulamasında dolduruluyor

---

## 🎯 Aşama 2: Kimlik Doğrulama Analizi

### Zayıf Giriş Mekanizması

```bash
curl -s -X POST \
     -d 'username=admin' \
     http://<HEDEF_IP>:5000/login
```

**Kritik Bulgu:** Parola alanı mevcut değil. `admin` gönderildiğinde hemen yönetici olarak giriş yapılıyor.

**Zafiyet:** Parola tabanlı kimlik doğrulamanın tamamen olmayışı.

---

## 🔐 Aşama 3: Kriptografik Analiz

### RSA Anahtar Üretim Zafiyeti

Kayıt sırasında sistem RSA anahtar çiftleri üretiyor. Analiz şunları ortaya koyuyor:

**Zafiyet:** Anahtarlar **tahmin edilebilir tohum değerleri** kullanılarak deterministik rastgele sayı üretimiyle oluşturuluyor.

**Teknik Detaylar:**
- Tohum, kullanıcı kontrollü girdiden veya zaman damgasından türetiliyor
- Hash çıktısına doğrudan `nextprime()` uygulanıyor
- Ortaya çıkan asallar yaklaşık **256-bit** (reklam edilen 2048-bit değil)
- Yaklaşık **512-bit RSA modülü** oluşturuluyor — kolayca çarpanlarına ayrılabilir

### Anahtar Yeniden Yapılandırma

**Adım 1:** Kayıttan veya anahtar dışa aktarımdan ortak anahtar parametrelerini çıkar.

**Adım 2:** Küçük modülü `n`'yi çarpanlarına ayır:

```python
from sympy import factorint

n = <çıkarılan_modül>
factors = factorint(n)
p, q = list(factors.keys())
```

**Adım 3:** Özel anahtarı yeniden yapılandır:

```python
from cryptography.hazmat.primitives.asymmetric import rsa
from cryptography.hazmat.primitives import serialization

# Özel üssü hesapla
phi = (p - 1) * (q - 1)
d = pow(e, -1, phi)

# Özel anahtarı yeniden yapılandır
private_key = rsa.RSAPrivateNumbers(
    d=d,
    p=p,
    q=q,
    dmp1=d % (p - 1),
    dmq1=d % (q - 1),
    iqmp=pow(q, -1, p),
    public_numbers=rsa.RSAPublicNumbers(e=e, n=n)
).private_key()
```

---

## ✍️ Aşama 4: İmza Sahtekarlığı

### Admin Mesajı Seçimi

Hedef mesaj: `/messages`'dan yöneticinin karşılama duyurusu

### İmza Üretimi

```python
from cryptography.hazmat.primitives import hashes
from cryptography.hazmat.primitives.asymmetric import padding
import binascii

message = b"<ADMIN_KARŞILAMA_MESAJI>"

signature = private_key.sign(
    message,
    padding.PSS(
        mgf=padding.MGF1(hashes.SHA256()),
        salt_length=padding.PSS.MAX_LENGTH
    ),
    hashes.SHA256()
)

signature_hex = binascii.hexlify(signature).decode()
```

**Kritik Not:** `PSS.MAX_LENGTH` kullanılmalıdır — sabit tuz uzunlukları küçük modül nedeniyle başarısız olur.

---

## 🏁 Aşama 5: Flag Çıkarımı

### Doğrulama İsteği

```bash
curl -X POST http://<HEDEF_IP>:5000/verify \
  -d "username=admin" \
  -d "message=<ADMIN_KARŞILAMA_MESAJI>" \
  -d "signature=<İMZA_HEX>"
```

**Sonuç:** Geçerli imza doğrulandı — yanıtta flag görüntülendi.

---

## 🗺️ Saldırı Yolu Görselleştirmesi

```
[Saldırgan]
    │
    ├───curl───► [Hedef: Port 5000]
    │                │
    │                ▼
    │        [Endpoint Enumerasyonu]
    │                │
    │                ▼
    │        [/register - Anahtar Üretimi]
    │                │
    │                ▼
    │        [Ortak Anahtarı Çıkar]
    │                │
    │                ▼
    │        [Küçük Modülü Çarpanlarına Ayır (512-bit)]
    │                │
    │                ▼
    │        [Özel Anahtarı Yeniden Yapılandır]
    │                │
    │                ▼
    │        [Admin İmzasını Sahtele]
    │                │
    │                ▼
    │        [POST /verify sahte imza ile]
    │                │
    │                ▼
    └────────► [Flag Alındı]
```

---

## 🛡️ Zafiyet Değerlendirmesi ve Çözümler

| # | Zafiyet | Şiddet | CVSS | Çözüm |
|---|---------|--------|------|-------|
| 1 | **Tahmin Edilebilir RSA Anahtar Üretimi** | 🔴 Kritik | 9.8 | Kriptografik olarak güvenli rastgele sayı üreticisi (CSPRNG) kullan; tohumu OS entropisinden al (`/dev/urandom`, `getrandom()`) |
| 2 | **Yetersiz Anahtar Boyutu** | 🔴 Kritik | 9.1 | Minimum 2048-bit RSA zorla; asal bit uzunluğunu doğrula |
| 3 | **Parolasız Kimlik Doğrulama** | 🔴 Kritik | 9.0 | Parola tabanlı kimlik doğrulama uygula; MFA ekle; oturum yönetimi |
| 4 | **RSA-PSS Tuz Uzunluğu Uyuşmazlığı** | 🟠 Yüksek | 7.5 | `PSS.MAX_LENGTH` kullan veya tuz uzunluğunu anahtar boyutuna göre açıkça doğrula |
| 5 | **İstemci Tarafı Anahtar Maruziyeti** | 🟠 Yüksek | 7.0 | Özel anahtar parametrelerini asla açıklama; sadece sunucu tarafı işlemler |

---

## 🎓 Temel Çıkarımlar

1. **Deterministik RNG = Bozuk Kripto:** Tahmin edilebilir tohumlar RSA güvenliğini tamamen ortadan kaldırır
2. **Küçük Modül = Kolay Çarpanlara Ayırma:** 512-bit RSA modern donanımda saniyeler içinde çarpanlarına ayrılabilir
3. **UI ≠ Gerçeklik:** "2048-bit" pazarlama metni gerçekte 512-bit implementasyonuyla çelişiyor
4. **PSS.MAX_LENGTH Kritik:** Tuz uzunluğu modül boyutuna uyum sağlamalı — sabit değerler küçük anahtarlarla başarısız olur
5. **Parolasız Kimlik Doğrulama Ölümcül:** Tek faktör kullanıcı adı sadece kimlik doğrulama değildir

---

---

## 🚩 Flag

| Flag | Değer |
|------|-------|
| **Flag** | `THM{PR3D1CT4BL3_S33D5_BR34K_H34RT5}` |

---

> **⚠️ Yasal Uyarı:** Bu write-up sadece eğitim ve araştırma amaçlıdır. Sahibi olmadığınız sistemleri test etmeden önce her zaman açık yazılı yetki alın.

---

**Yazar:** Miraç Akkuş (LatenT)  
**Tarih:** Mayıs 2026
```
