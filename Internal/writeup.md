🕵️‍♂️ Internal - TryHackMe Writeup

    Difficulty: Medium / Hard | Focus: WP Exploitation, Pivoting, Jenkins

Yo! This is my walkthrough for the Internal machine on THM. This one was a bit of a grind with the pivoting part, but we got it. If you're looking for the flags, go get 'em yourself—I'm just showing you how I did it. 😉
🔍 1. Recon (Scanning the target)

First things first, I hit it with nmap to see what we're dealing with.
Bash

nmap -sC -sV -oA nmap/internal.thm internal.thm

Port	Service	Version
22	SSH	OpenSSH 7.6p1 (Ubuntu)
80	HTTP	Apache httpd 2.4.29
🌐 2. Web & WordPress Pwnage

Did some directory busting and found a /blog directory. It’s a WordPress site. Classic.
🛡️ Exploitation

I grabbed wpscan to find users and then started a brute-force on the admin user.
Bash

# Brute-forcing the admin login
wpscan --url http://internal.thm/blog --passwords /usr/share/wordlists/rockyou.txt --usernames admin

    [!IMPORTANT]
    Found it: Cracked the admin password, logged in, and headed straight to the Theme Editor. I swapped the 404.php code with a PHP reverse shell. Fired up my listener, hit the page, and boom: Initial Access.

🔄 3. Lateral Movement (Moving to Aubreanna)

I was stuck as www-data, so I started digging around. Found a file at /opt/wp-save.txt that had some creds.
Bash

cat /opt/wp-save.txt
# Found: aubreanna:[REDACTED]

Logged in via SSH and grabbed the User Flag:
cat /home/aubreanna/user.txt
🎯 4. PrivEsc (The Jenkins Part)

Checking internal ports, I saw something running on 127.0.0.1:8080. It’s Jenkins, but it's only accessible locally.
🚇 SSH Tunneling (Pivoting)

To get to that Jenkins panel from my own machine, I set up a tunnel:
Bash

ssh -L 9999:127.0.0.1:8080 aubreanna@internal.thm

Now I just go to http://localhost:9999 on my browser.
⚙️ Jenkins Exploitation

I got into the Jenkins dashboard and used the Script Console to run a Groovy reverse shell. Now I'm in as the jenkins user.
🏁 5. Root Flag (Final Boss)

Under /opt, there was a note.txt file. Some guy named Will left the root password just sitting there. Thanks, Will!
Bash

# Switched to root
su root
# Checked the flag
cat /root/root.txt

🖼️ Proof of Pwn
🛡️ Vulnerabilities Found

    Weak Credentials: Brute-force worked too easily on WP.

    Sensitive Info in Plaintext: wp-save.txt and note.txt had passwords. Absolute fail.

    Jenkins Script Console: Allowed RCE (Remote Code Execution) way too easily.

    [!NOTE]
    WANT THE FLAG? GO GET IT THEN.
    Made by LatenT. This is my 2nd writeup, don't roast me too hard.
