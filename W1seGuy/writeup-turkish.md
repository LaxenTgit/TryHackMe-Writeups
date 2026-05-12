# W1seGuy — TryHackMe Kriptografik Analiz

> **Sınıflandırma:** CTF Writeup | **Vektör:** Kriptografik Uygulama Hatası | **Şiddet:** Kritik

---

## 1. Tehdit Modeli

Uzak servis (`<TARGET_IP>:1337`), bayrakları **deterministik tekrarlayan XOR şifreleme** ile **5 karakterlik alfanümerik anahtar** kullanarak şifreliyor. Öngörülebilir anahtar uzunluğu ve bilinen bayrak formatı (`THM{...}`), şemayı **bilinen-açıkmetin kriptoanalizine** karşı savunmasız kılıyor.

---

## 2. Kriptografik Zafiyetler

| Kontrol | Başarısızlık | CWE Referansı |
|---------|-----------|---------------|
| Anahtar entropisi | 5 karakter `[A-Za-z0-9]` → ~29.9 bit | CWE-331 |
| Anahtar yeniden kullanımı | Açıkmetin boyunca `key[i % 5]` tekrarı | CWE-323 |
| IV/nonce eksikliği | Deterministik çıktı; aynı açıkmetinler aynı şifremetinleri üretir | CWE-329 |

---

## 3. Saldırı Vektörü: Bilinen-Açıkmetin Kurtarma

**Matematiksel temel:** Şifremetin `C` ve açıkmetin `P` verildiğinde, `C ⊕ P = K`.

```python
from pwn import remote, xor

# Oturum kur
r = remote('<TARGET_IP>', 1337)
hex_ct = r.recvline().strip().decode()
ct = bytes.fromhex(hex_ct)

# Aşama 1: Bilinen önek ile anahtar bayt 0-3 kurtarma
known_prefix = b"THM{"
key_partial = xor(ct[:4], known_prefix)

# Aşama 2: Yapısal doğrulama ile anahtar bayt 4 kaba-kuvvet
for candidate in (string.ascii_letters + string.digits):
    key = key_partial + candidate.encode()
    pt = xor(ct, key)
    if pt.startswith(b"THM{") and pt.endswith(b"}"):
        r.sendline(key.decode())
        flag2 = r.recvline().strip().decode()
        print(f"Kurtarılan Anahtar: {key.decode()}")
        print(f"Bayrak 1: {pt.decode()}")
        print(f"Bayrak 2: {flag2}")
        break
```

---

## 4. Sonuçlar

| Artefakt | Değer |
|----------|-------|
| **Şifreleme Anahtarı** | `IK0Gz` (örnek) |
| **Bayrak 1 (çözülmüş)** | `THM{p1alntExtAtt4ckcAnr3alLyhUrty0urxOr}` |
| **Bayrak 2 (sunucu yanıtı)** | `THM{BrUt3_ForC1nG_XOR_cAn_B3_FuN_nO?}` |

---

## 5. Düzeltme Önerileri

1. **XOR'u AES-256-GCM ile değiştirin** — kimlik doğrulamalı şifreleme
2. **Anahtar uzunluğu ≥ 128 bit** ve CSPRNG ile üretim
3. **Her şifreleme için benzersiz nonce** — anahtar akışı malzemesini asla yeniden kullanmayın

---

> **Analist:** Miraç Akkuş (LatenT) | **Tarih:** 2026-05-12 | **Sınıflandırma:** TLP:CLEAR
