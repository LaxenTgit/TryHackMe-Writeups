# ✉️ Letter — TryHackMe Writeup

![Difficulty](https://img.shields.io/badge/Difficulty-Easy-brightgreen)
![Category](https://img.shields.io/badge/Category-OSINT%20|%20Historical%20Research%20|%20Puzzle-blue)
![Platform](https://img.shields.io/badge/Platform-TryHackMe-green)

> **Author:** Miraç Akkuş (LatenT)  
> **Date:** May 2026  
> **Room:** [Letter](https://tryhackme.com/room/letter)

---

## 📋 Case Overview

This room is entirely OSINT-based. We have a worn envelope, a newspaper clipping, and a handwritten note. Objective: find the postal code of the delivery address and identify the person mentioned in the note. No nmap, no exploits — just observation, French history, and some research.

---

## 🛠️ Tools

| Tool | Purpose |
|------|---------|
| Wikipedia | French postal barcode table |
| Google | Newspaper headlines and historical events |
| KBC PENMARC'H | Regional historical archive |
| Google Maps | Postal code verification |

---

## 🔍 Question 1: What is the postal code of the delivery address?

### French Postal Barcode

Orange bars on the envelope:

```
..||||| |.||.| ||..|| |||..| .||.||
```

French postal service barcode system. Wikipedia table:

| Digit | Barcode |
|-------|---------|
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

Decoding:

| Segment | Digit |
|---------|-------|
| `..|||||` | 0 (start bar, ignore) |
| `|.||.|` | 6 |
| `||..||` | 7 |
| `|||..|` | 9 |
| `.||.||` | 2 |

Result: `06792`

**But** French postal codes are read **right to left**. Reverse:

**Postal Code:** `29760`

This is the postal code of **Penmarc'h**, Finistère region.

---

## 🔍 Question 2: What is the flag? (Full name and age)

### Note Analysis

Handwritten note (translated):

> *"Dear Édouard, today while cleaning my grandparents' attic I came across an old newspaper clipping. When your great-grandfather showed himself that day, he wasn't even old enough to get a license. The youngest member of the crew, and certainly not the least brave. He would have been so proud to see you in the water. With all my love, Audette"*

**Clues:**
- Youngest crew member
- Under 18 (no license)
- Related to SNSM (French National Sea Rescue)

### Newspaper Clipping

**L'Ouest-Éclair** newspaper, May 28, 1925.

One headline: *"Amundsen a-t-il atteint le pôle Nord?"* (Did Amundsen reach the North Pole?)

Roald Amundsen's 1925 plane expedition was lost, so the newspaper is from 1925.

Another headline: *"sept noyés"* (seven drowned) — maritime disaster.

### Historical Event

Late 1925, off the coast of **Penmarc'h**, Finistère, two rescue boats capsized during a rescue operation:

- **Kérity** boat
- **Saint-Pierre** boat

**May 23, 1925 Penmarc'h disaster** — 7 rescue workers drowned.

### KBC PENMARC'H Archive

French regional historical research site: [kbcpenmarh.org](https://kbcpenmarh.org)

Section: **DISASTER OF MAY 23, 1925**

Crew list examined. Based on note clues, youngest member searched:

**GOURLAOUEN (Yves-Marie) — 15 years old**

- Youngest crew member
- 15 years old (license age is 18)
- Awarded Silver Medal, 2nd Class

### Flag Format

`THM{Name_Surname_age}` — only first letters capitalized:

**Flag:** `THM{Yves-Marie_Gourlaouen_15}`

---

## 📊 Investigation Summary

| Question | Answer | Technique |
|----------|--------|-----------|
| Postal Code | 29760 | French postal barcode decoding |
| Flag | THM{Yves-Marie_Gourlaouen_15} | Historical newspaper archives, OSINT |

---

## 🎓 Key Takeaways

1. **Postal barcodes** — Country-specific encoding systems exist, tables can be found on Wikipedia
2. **Newspaper clippings** — Headlines pinpoint historical events
3. **Regional archives** — Sites like KBC PENMARC'H can be more detailed than national archives
4. **Age calculation** — "No license" = under 18, narrowing is critical
5. **OSINT = patience** — Piecing together takes time but the chain completes

---

> *"Today while cleaning my grandparents' attic..."*  
> — Audette, granddaughter of the 1925 disaster

---

**Author:** LatenT  
**Date:** May 2026
