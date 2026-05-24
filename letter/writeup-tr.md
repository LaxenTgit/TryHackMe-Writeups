# ✉️ Letter — TryHackMe Writeup

![Zorluk](https://img.shields.io/badge/Zorluk-Kolay-brightgreen)
![Kategori](https://img.shields.io/badge/Kategori-OSINT%20|%20Tarihi%20Araştırma%20|%20Puzzle-blue)
![Platform](https://img.shields.io/badge/Platform-TryHackMe-green)

> **Yazar:** Miraç Akkuş (LatenT)  
> **Tarih:** Mayıs 2026  
> **Oda:** [Letter](https://tryhackme.com/room/letter)

---

## 📋 Olay Özeti

Bu oda tamamen OSINT tabanlı. Elimizde yıpranmış bir zarf, bir gazete kupürü ve el yazısı bir not var. Amacımız: teslimat adresinin posta kodunu bulmak ve notta bahsedilen kişinin kim olduğunu tespit etmek. Nmap yok, exploit yok — sadece gözlem, Fransız tarihi ve biraz araştırma.

---

## 🛠️ Araçlar

| Araç | Amaç |
|------|------|
| Wikipedia | Fransız posta barkodu tablosu |
| Google | Gazete başlıkları ve tarihi olaylar |
| KBC PENMARC'H | Bölgesel tarihi arşiv |
| Google Maps | Posta kodu doğrulama |

---

## 🔍 Soru 1: Zarfın teslimat adresi posta kodu nedir?

### Fransız Posta Barkodu

Zarfın üzerinde turuncu çubuklar:

```
..||||| |.||.| ||..|| |||..| .||.||
```

Fransız posta servisinin kullandığı barkod sistemi. Wikipedia'dan tablo:

| Rakam | Barkod |
|-------|--------|
| 0 | `..||||` |
| 1 | `.|.|||` |
| 2 | `.||.||` |
| 3 | `.|||.` |
| 4 | `|..|||` |
| 5 | `|.|.||` |
| 6 | `|.||.` |
| 7 | `||..||` |
| 8 | `||.|.` |
| 9 | `|||..|` |

Çözümleme:

| Segment | Rakam |
|---------|-------|
| `..|||||` | 0 (start bar, ignore) |
| `|.||.|` | 6 |
| `||..||` | 7 |
| `|||..|` | 9 |
| `.||.||` | 2 |

Sonuç: `06792`

**Ama** Fransız posta kodları **sağdan sola** okunur. Ters çevir:

**Posta Kodu:** `29760`

Bu, Finistère bölgesindeki **Penmarc'h** kasabasının posta kodu.

---

## 🔍 Soru 2: Flag nedir? (Kişinin tam adı ve yaşı)

### Not Analizi

El yazısı not (çeviri):

> *"Sevgili Édouard, bugün büyükanne ve büyükbabamın çatı katını temizlerken eski bir gazete kupürüne rastladım. Büyük büyükbaban o gün kendini gösterdiğinde ehliyet alacak kadar bile yaşında değildi. Ekibin en genç üyesi, ve kesinlikle en cesur olmayan değildi. Seni de suda görseydi çok gurur duyardı. Tüm sevgimle, Audette"*

**İpuçları:**
- En genç ekip üyesi
- 18 yaş altı (ehliyet yok)
- Denizci/SNSM (Fransız Ulusal Deniz Kurtarma) ile ilgili

### Gazete Kupürü

**L'Ouest-Éclair** gazetesi, 28 Mayıs 1925.

Başlıklardan biri: *"Amundsen a-t-il atteint le pôle Nord?"* (Amundsen Kuzey Kutbu'na ulaştı mı?)

Roald Amundsen'in 1925 uçak seferi kaybolmuştu, bu yüzden gazete 1925'ten.

Diğer başlık: *"sept noyés"* (yedi boğulan) — deniz faciası.

### Tarihi Olay

1925'in sonlarında, Finistère bölgesindeki **Penmarc'h** açıklarında bir kurtarma operasyonu sırasında iki kurtarma botu alabora oldu:

- **Kérity** botu
- **Saint-Pierre** botu

**23 Mayıs 1925 Penmarc'h faciası** — 7 kurtarma görevlisi boğuldu.

### KBC PENMARC'H Arşivi

Fransız bölgesel tarihi araştırma sitesi: [kbcpenmarh.org](https://kbcpenmarh.org)

Bölüm: **DISASTER OF MAY 23, 1925**

Mürettebat listesi incelendi. Notun ipuçlarına göre en genç üye arandı:

**GOURLAOUEN (Yves-Marie) — 15 yaşında**

- En genç mürettebat üyesi
- 15 yaşında (ehliyet yaşı 18)
- Gümüş Madalya, 2. Sınıf ile ödüllendirildi

### Flag Formatı

`THM{Name_Surname_age}` — sadece ilk harfler büyük:

**Flag:** `THM{Yves-Marie_Gourlaouen_15}`

---

## 📊 Soruşturma Özeti

| Soru | Cevap | Teknik |
|------|-------|--------|
| Posta Kodu | 29760 | Fransız posta barkodu çözme |
| Flag | THM{Yves-Marie_Gourlaouen_15} | Tarihi gazete arşivleri, OSINT |

---

## 🎓 Çıkarımlar

1. **Posta barkodları** — Ülkelere özgü encoding sistemleri var, Wikipedia'dan tablo bulunabilir
2. **Gazete kupürleri** — Başlıklar tarihi olayları pinpoint eder
3. **Bölgesel arşivler** — KBC PENMARC'H gibi siteler, ulusal arşivlerden daha detaylı olabilir
4. **Yaş hesabı** — "Ehliyet yok" = 18 altı, daraltma kritik
5. **OSINT = sabır** — Parçaları birleştirmek zaman alır ama zincir tamamlanır

---

> *"Bugün büyükanne ve büyükbabamın çatı katını temizlerken..."*  
> — Audette, 1925 faciasının torunu

---

**Yazan:** LatenT  
**Tarih:** Mayıs 2026
