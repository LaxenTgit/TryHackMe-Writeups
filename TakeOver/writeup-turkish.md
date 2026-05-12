# 🌐 TakeOver — TryHackMe Writeup

![Zorluk](https://img.shields.io/badge/Zorluk-Kolay-green)
![Kategori](https://img.shields.io/badge/Kategori-Web%20|%20Alt%20Alan%20Adı%20Enumerasyonu%20|%20SSL%20Sertifika%20Analizi-blue)
![Platform](https://img.shields.io/badge/Platform-TryHackMe-green)

> **Yazar:** Miraç Akkuş (LatenT)  
> **Tarih:** Mayıs 2026  
> **Oda:** [TakeOver](https://tryhackme.com/room/takeover)

---

## 📋 Yönetici Özeti

Bu write-up, **TakeOver** makinesinin sömürülmesini belgelemektedir. Gizli altyapıyı keşfetmek için **alt alan adı enumerasyonu** ve **SSL sertifika analizi** tekniklerini göstermektedir. Saldırı zinciri, sanal ana bilgisayar fuzzing'i ile alt alan adlarını tanımlamayı, gizli alt alan adlarını ortaya çıkarmak için sertifika Konu Alternatif Adı (SAN) incelemesini ve hatalı yapılandırılmış bir AWS S3 yönlendirmesinden flag'i çıkarmak için HTTP başlık analizini kapsamaktadır.

---

## 🛠️ Kullanılan Araçlar ve Teknolojiler

| Araç | Amaç |
|------|------|
| `nmap` | Ağ keşfi ve servis enumerasyonu |
| `ffuf` | Sanal ana bilgisayar/alt alan adı keşfi için hızlı web fuzzer |
| `curl` | HTTP istek testi ve başlık analizi |
| `openssl` | SSL/TLS sertifika incelemesi |
| `gobuster` | Dizin enumerasyonu |

---

## 🔍 Aşama 1: Keşif ve Enumerasyon

### Ağ Taraması

```bash
nmap -sV -sC -T4 <HEDEF_IP>
```

### Bulgular

| Port | Durum | Servis | Versiyon | Notlar |
|------|-------|--------|----------|--------|
| 22/tcp | açık | SSH | OpenSSH 8.2p1 Ubuntu | Standart erişim |
| 80/tcp | açık | HTTP | Apache httpd 2.4.41 | HTTPS'e yönlendiriyor |
| 443/tcp | açık | SSL/HTTP | Apache httpd 2.4.41 | SSL sertifikası mevcut |

**Kritik Gözlemler:**
- Port 80, `https://futurevera.thm`'ye yönlendiriyor
- SSL sertifikası süresi dolmuş (Geçerlilik bitişi: 2023-03-13)
- Özel alan adı `futurevera.thm`, `/etc/hosts` yapılandırması gerektiriyor

### Host Dosyası Yapılandırması

```bash
echo "<HEDEF_IP> futurevera.thm" | sudo tee -a /etc/hosts
```

---

## 🎯 Aşama 2: Alt Alan Adı Enumerasyonu

### Temel Yanıt Analizi

Fuzzing öncesinde, mevcut olmayan alt alan adları için temel yanıt boyutunu belirle:

```bash
curl -I -k -H "Host: rastgeleisim.futurevera.thm" https://<HEDEF_IP>
```

**Sonuç:** Yanıt boyutu = **4605 bayt** (geçersiz alt alan adları için standart hata sayfası)

### Sanal Ana Bilgisayar Fuzzing'i ffuf ile

```bash
ffuf -u https://<HEDEF_IP> \
     -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt \
     -H "Host: FUZZ.futurevera.thm" \
     -fs 4605
```

**Keşfedilen Alt Alan Adları:**

| Alt Alan Adı | Durum | Boyut | Notlar |
|--------------|-------|-------|--------|
| `support.futurevera.thm` | 200 | 1522 | Destek sayfası |
| `blog.futurevera.thm` | 200 | 3838 | Blog sayfası |

### Host Dosyası Güncellemesi

```bash
echo "<HEDEF_IP> support.futurevera.thm blog.futurevera.thm" | sudo tee -a /etc/hosts
```

---

## 🔐 Aşama 3: SSL Sertifika Analizi

### Sertifika İncelemesi (support.futurevera.thm)

```bash
nmap -p 443 --script ssl-cert support.futurevera.thm
```

**Sertifika Detayları:**

| Alan | Değer |
|------|-------|
| Konu | `CN=support.futurevera.thm` |
| Kuruluş | Futurevera |
| Eyalet | Oregon |
| Ülke | US |
| **Konu Alternatif Adı** | `DNS:secrethelpdesk934752.support.futurevera.thm` |

**Kritik Bulgu:** SAN alanında gizli alt alan adı keşfedildi — `secrethelpdesk934752.support.futurevera.thm`

### Sertifika İncelemesi (blog.futurevera.thm)

```bash
nmap -p 443 --script ssl-cert blog.futurevera.thm
```

**Sonuç:** Ek SAN girişi yok — sadece standart sertifika.

---

## 🌐 Aşama 4: Gizli Alt Alan Adına Erişim

### Host Dosyası Yapılandırması

```bash
echo "<HEDEF_IP> secrethelpdesk934752.support.futurevera.thm" | sudo tee -a /etc/hosts
```

### HTTP vs HTTPS Davranış Analizi

| Protokol | Yanıt | Notlar |
|----------|-------|--------|
| HTTPS | Sertifika hatası / zaman aşımı | SSL sertifika uyuşmazlığı |
| HTTP | Anında 302 yönlendirme | Temiz yanıt |

### curl Başlık Analizi

```bash
curl -I http://secrethelpdesk934752.support.futurevera.thm
```

**Yanıt:**
```http
HTTP/1.1 302 Found
Date: Sat, 15 Apr 2023 18:03:13 GMT
Server: Apache/2.4.41 (Ubuntu)
Location: http://flag{beea0d6edfcee06a59b83fb50ae81b2f}.s3-website-us-west-3.amazonaws.com/
Content-Type: text/html; charset=UTF-8
```

---

## 🗺️ Saldırı Yolu Görselleştirmesi

```
[Saldırgan]
    │
    ├───nmap───► [Hedef: 22, 80, 443]
    │                │
    │                ▼
    │        [futurevera.thm /etc/hosts'ta yapılandırıldı]
    │                │
    │                ▼
    │        [Temel: 4605 bayt hata sayfası]
    │                │
    │                ▼
    │        [ffuf vhost fuzzing]
    │                │
    │                ▼
    │        [support.futurevera.thm keşfedildi]
    │                │
    │                ▼
    │        [SSL sertifika SAN analizi]
    │                │
    │                ▼
    │        [secrethelpdesk934752.support.futurevera.thm bulundu]
    │                │
    │                ▼
    │        [Gizli alt alan adına HTTP isteği]
    │                │
    │                ▼
    │        [302 Yönlendirme AWS S3'e]
    │                │
    │                ▼
    └────────► [Location başlığında flag açığa çıktı]
```

---

## 🛡️ Zafiyet Değerlendirmesi ve Çözümler

| # | Zafiyet | Şiddet | CVSS | Çözüm |
|---|---------|--------|------|-------|
| 1 | **Alt Alan Adı Enumerasyon Maruziyeti** | 🟠 Yüksek | 7.5 | Joker DNS yanıtları uygula; DNSSEC kullan; bölge transferlerini kısıtla |
| 2 | **SSL Sertifika SAN Sızdırma** | 🟠 Yüksek | 7.0 | SAN girişlerini minimize et; joker sertifikaları kısıtlı kullan; sertifika meta verilerini denetle |
| 3 | **Süresi Dolmuş SSL Sertifikası** | 🟠 Orta | 5.3 | Otomatik sertifika yenileme uygula (Let's Encrypt, cert-manager) |
| 4 | **AWS S3 Alt Alan Adı Ele Geçirme** | 🔴 Kritik | 9.0 | Sarkan DNS kayıtlarını kaldır; CNAME oluşturmadan önce S3 bucket varlığını doğrula; AWS Route 53 sağlık kontrolleri kullan |
| 5 | **Yönlendirme URL'lerinde Hassas Veri** | 🔴 Kritik | 9.1 | Sırları asla URL'lere gömme; POST gövdeleri veya güvenli başlıklar kullan; uygun oturum yönetimi uygula |

---

## 🎓 Temel Çıkarımlar

1. **Sertifika SAN Analizi:** SSL sertifikaları sıklıkla Konu Alternatif Adları aracılığıyla gizli altyapıyı sızdırır — her zaman incele
2. **Temel Filtreleme:** Yanlış pozitifleri ortadan kaldırmak için fuzzing öncesinde yanıt temellerini belirle (ffuf'ta `-fs` bayrağı)
3. **HTTP vs HTTPS:** Protokol farklılıkları farklı davranışları ortaya çıkarabilir — her zaman ikisini de test et
4. **Başlık İncelemesi:** `curl -I` yönlendirme zincirlerini ve HTTP başlıklarındaki gizli verileri ortaya çıkarır
5. **Alt Alan Adı Ele Geçirme:** Silinmiş bulut kaynaklarına işaret eden sarkan DNS kayıtları kritik bir saldırı vektörüdür

---

---

## 🚩 Flag

| Flag | Değer |
|------|-------|
| **Flag** | `flag{beea0d6edfcee06a59b83fb50ae81b2f}` |

---

> **⚠️ Yasal Uyarı:** Bu write-up sadece eğitim ve araştırma amaçlıdır. Sahibi olmadığınız sistemleri test etmeden önce her zaman açık yazılı yetki alın.

---
**Yazar:** Miraç Akkuş (LatenT)  
**Tarih:** Mayıs 2026
```
