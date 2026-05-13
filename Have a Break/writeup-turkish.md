# 🕵️ Have a Break — Digital Forensics Investigation

![Zorluk](https://img.shields.io/badge/Zorluk-Orta-orange)
![Kategori](https://img.shields.io/badge/Kategori-Dijital%20Adli%20Bilişim%20|%20OSINT%20|%20Olay%20Müdahale-blue)
![Platform](https://img.shields.io/badge/Platform-TryHackMe-green)

> **Yazar:** Miraç Akkuş (LatenT)  
> **Tarih:** Mayıs 2026  
> **Oda:** [Have a Break](https://tryhackme.com/room/haveabreak)

---

## 📋 Olay Özeti

Çek bir lojistik şirketinde iç veri sızıntısı şüphesi. Elimizdeki kanıtlar: anonim e-posta, dash cam görüntüsü, rota planlama logları, çalışan veritabanı, iç memo ve iletişim kayıtları. Amacımız: VPN servisini, konumu, zaman damgasını ve sızıntının arkasındaki kişiyi tespit etmek.

---

## 🛠️ Araçlar

| Araç | Ne İşe Yaradı |
|------|---------------|
| EML Parser | E-posta header analizi |
| ipinfo.io / bgp.tools | IP lookup ve ASN sorgulama |
| Google Maps | Benzin istasyonu konumlandırma |
| Epieos | Ücretsiz OSINT ve e-posta pivot |
| CSV Analysis | Log ve çalışan verisi korelasyonu |

---

## 🔍 Soru 1: Hangi VPN servisi kullanıldı?

### E-posta Header Analizi

`exhibit_a.eml` açıldı. Gönderen: `notmyname2847@gmail.com`, alıcı: `redakce@novinybrno.cz` (Çek haber ajansı).

**Kritik Header:**
```
Received: from [193.32.249.132] ([193.32.249.132])
        by smtp.gmail.com with ESMTPSA id
        4fb4d7f45d1cf-66e02d37620sm407278a12.2.2026.03.27.23.14.55
```

Sonraki `Received:` hop'lar Google altyapısı (`209.85.220.41` vb.) — origin IP için irrelevant.

### IP Araştırması

| Özellik | Değer |
|---------|-------|
| IP | `193.32.249.132` |
| ASN | AS39351 |
| Organizasyon | 31173 Services AB |
| Lokasyon | Amsterdam, Hollanda |
| Flag | VPN exit node |

31173 Services, İsveçli hosting sağlayıcı. Mullvad VPN ile ortaklığı documented. `193.32.249.0/24` subnet, Mullvad'ın Amsterdam sunucu altyapısının parçası.

**Cevap:** `Mullvad VPN`

---

## 🗺️ Soru 2: Benzin istasyonunun tam adresi?

### İlk Yaklaşım

Orlen'in resmi sitesi (`orlen.cz/stanice`) kullanıldı — Google Maps'ten daha güvenilir. LPG uyumlu istasyonlar filtrelendi.

### Çıkmaz Sokak

D1 otoyolu boyunca LPG'li Orlen istasyonları arandı, ancak dash cam'deki **Brno ve Olomouc** yön tabelaları hiçbirinde yoktu.

### Kırılma Noktası

`ecta_memo.pdf` incelendi:

> *"Mart 2026, Hulín yakınlarında trafik meselesi nedeniyle durdurulan bir araçtan dashcam SD kartı kurtarıldı."*

### Konum Doğrulama

Google Maps'te **Hulín bölgesi** arandı. Tek eşleşen istasyon:

| Özellik | Değer |
|---------|-------|
| Marka | Orlen |
| Lokasyon | Hulín, Çek Cumhuriyeti |
| Özellik | LPG mevcut |
| Yön Tabelası | Brno / Olomouc doğrulandı |

**Cevap:** `Hulín Orlen istasyon adresi`

---

## ⏰ Soru 3: Şüpheli eylemin zaman damgası?

### Log Analizi

`access_log.csv`, 25 Mart 2026 için filtrelendi.

### Anomali Tespiti

| Zaman Damgası | Kullanıcı | Eylem | Dosya |
|---------------|-----------|-------|-------|
| 22:14:09 | BR-0291 | **EXPORT** | Hassas rota PDF'i |
| [Çeşitli] | [Diğer kullanıcılar] | VIEW / EDIT | [Çeşitli dosyalar] |

**Kırmızı Bayraklar:**
- **EXPORT** — diğer tüm kullanıcılar sadece VIEW/EDIT yapmış
- **Mesai saati dışı** — 22:14 normal iş aktivitesi için şüpheli
- **Önceki başarısız auth** — BR-0291, 24 Mart'ta auth başarısızlığı yaşamış (keşif faaliyeti)
- **Olası admin müdahalesi** — 27 Mart'ta erişim kısıtlamaları uygulanmış

**Cevap:** `22:14:09`

---

## 👤 Soru 4: Anonim e-postayı gönderen çalışan ID'si?

### Zamansal Korelasyon

`access_log.csv` ve `employees.csv` cross-reference edildi:

| Zaman Damgası | Kullanıcı | Rol | Aktivite |
|---------------|-----------|-----|----------|
| 23:41 (25 Mar) | BR-0312 | Dispatch Operatörü | Sistem erişimi |
| 23:12 (24 Mar) | BR-0312 | Dispatch Operatörü | Gece aktivitesi |

### Davranış Profili

- BR-0312, düzenli gece vardiyası desenine sahip Dispatch Operatörü
- BR-0291'nin şüpheli EXPORT'undan (22:14) hemen sonra aktif
- Anonim e-postada: *"Şu anda şirket içinde kime güveneceğimi bilmiyorum"*

Bu, BR-0312'nin anomaliyi gözlemlediğini ancak IT personeli olmadığı için iç raporlama yerine dışarıya sızdırdığını gösteriyor.

**Cevap:** `BR-0312`

---

## 🎯 Soru 5: Sızıntıdan sorumlu çalışan ID'si?

Soru 3 bulgularına dayanarak, 25 Mart 22:14:09'daki EXPORT eylemi veri exfiltrasyonunu temsil ediyor.

**Cevap:** `BR-0291`

---

## 🔎 Soru 6: Suçlunun tam adı?

### OSINT Pivot

Sağlanan dosyalarda tam isim yok. Harici istihbarat gerekiyor.

### E-posta Keşfi

`comms_export.txt`'den şüpheli e-posta: `kraliknovak09@gmail.com`

### Araç Değerlendirmesi

| Platform | Sonuç |
|----------|-------|
| TruePeopleSearch | Ücretli — reddedildi |
| BeenVerified | Ücretli — reddedildi |
| **Epieos** | **Ücretsiz, kapsamlı sonuçlar** |

### Epieos Analizi

**Girdi:** `kraliknovak09@gmail.com`

**Sonuç:** İlişkili Google Maps profil linki — konum geçmişi ve kullanıcı tarafından oluşturulan içerik.

### Google Maps Doğrulama

Epieos linki takip edildi. Profilde:
- Konum check-in'leri
- Public yorumlar
- İlişkili işletme listelemeleri

**Çıkarılan İsim:** Radovan Blšťák

**Cevap:** `Radovan Blšťák`

---

## 📊 Soruşturma Özeti

| Soru | Cevap | Teknik |
|------|-------|--------|
| VPN Servisi | Mullvad VPN | E-posta header analizi, ASN lookup |
| Benzin İstasyonu | Hulín Orlen | PDF analizi, Google Maps doğrulama |
| Şüpheli Zaman | 22:14:09 | CSV log analizi, anomali tespiti |
| Muhbir ID | BR-0312 | Zamansal korelasyon, davranış analizi |
| Suçlu ID | BR-0291 | Log'dan eylem ataması |
| Suçlu İsim | Radovan Blšťák | Epieos OSINT, Google Maps pivot |

---

## 🛡️ Çözümler ve Öneriler

| # | Zafiyet | Şiddet | Çözüm |
|---|---------|--------|-------|
| 1 | E-posta Header Sızıntısı | 🟠 Yüksek | Outbound e-postalardan client IP'yi temizle; kurumsal VPN kullan |
| 2 | Mesai Saati Dışı EXPORT | 🔴 Kritik | Zaman bazlı erişim kontrolleri; çifte yetkilendirme |
| 3 | Başarısız Auth İzleme | 🟠 Yüksek | Tekrarlayan başarısız auth'lar için alert; hesap kilitleme |
| 4 | OSINT Maruziyeti | 🟠 Yüksek | Çalışan sosyal medya eğitimi; gizlilik ayarları denetimi |
| 5 | Log Retansiyonu | 🟡 Orta | Log retansiyonunu uzat; SIEM korelasyonu uygula |

---

## 🎓 Temel Çıkarımlar

1. **E-posta header'ları yalan söylemez** — Client submission sonrası ilk `Received:` hop gerçek origin IP'yi ortaya çıkarır
2. **Resmi kaynaklar > üçüncü parti** — Orlen'in kendi istasyon bulucusu Google Maps'ten daha güvenilir
3. **Bağlamsal dokümanlar önemlidir** — `ecta_memo.pdf` görsel kanıt yetersiz olduğunda coğrafi pivot sağladı
4. **Zamansal korelasyon güçlüdür** — BR-0312'nin olay sonrası erişim deseni muhbir kimliğini ortaya çıkardı
5. **Ücretsiz OSINT araçları mevcut** — Epieos, ücretli alternatiflerden daha iyi performans gösterdi
6. **Ödeme duvarları çıkmaz sokak değil** — Ücretsiz alternatifler sıklıkla eşdeğer veya üstün sonuçlar verir

---

> *"Şu anda şirket içinde kime güveneceğimi bilmiyorum."*  
> — BR-0312, Dispatch Operatörü, muhbir

---

**Yazar:** Miraç Akkuş (LatenT)  
**Tarih:** Mayıs 2026
