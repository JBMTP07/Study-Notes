# Nmap Enumeration Playbook

> Praxis-Workflow für Nmap-basierte Netzwerk-Enumeration —
> von Host-Discovery bis Service-Validation.

## Methodik

```
Sweep → Target Confirm → Service Enum → Script Run → Validate
```

Nmap ist das Fundament. Alle weiteren Tools (smb-, http-, ldap-Enum) bauen auf
einem präzisen Service-Inventar auf, das Nmap liefert.

---

## 1. Host Discovery

### Lokales Subnetz

```bash
sudo nmap -sn 192.168.1.0/24 -PE -PP -PM -PR -oA host-discovery
```

Flags:
- `-sn` = Skip port scan, Ping only
- `-PR` = ARP-Ping (nur lokal, am zuverlässigsten)
- `-PE -PP -PM` = ICMP Echo, Timestamp, Netmask Requests

### Über Routing-Grenzen hinweg

ICMP wird oft gefiltert. Stattdessen:

```bash
nmap -sn -PS22,80,443,3389 -PA80,443 -PU53,161 <RANGE>
```

### Top-Speed mit `masscan` für große Ranges

```bash
sudo masscan -p1-65535 10.0.0.0/16 --rate=10000 -oG masscan.gnmap
```

`masscan` liefert Inventar, Nmap liefert dann Detail-Enumeration auf den Treffern.

---

## 2. Initial Port Scan

### Zwei-Stufen-Workflow

**Stufe 1 — schneller breiter Sweep:**

```bash
sudo nmap -sS -p- --min-rate=2000 -T4 -oA scan-tcp <TARGET>
```

**Stufe 2 — gezielte Service-Enumeration auf den offenen Ports:**

```bash
PORTS=$(grep -oP '\d{1,5}/open' scan-tcp.gnmap | cut -d/ -f1 | tr '\n' ',' | sed 's/,$//')
sudo nmap -sV -sC -p $PORTS -oA scan-services <TARGET>
```

### Warum Stufe 1 / Stufe 2?

`-sV -sC` auf alle 65k Ports ist langsam und unnötig. Erst Inventar bauen, dann gezielt vertiefen.

### UDP-Scans

UDP ist langsam und unzuverlässig. Top-100 reicht meist:

```bash
sudo nmap -sU --top-ports 100 -oA scan-udp <TARGET>
```

Wenn Verdacht auf SNMP / DNS / DHCP / TFTP / NTP:

```bash
sudo nmap -sU -p 53,67,68,69,123,161,500,514,1900,5353 <TARGET>
```

---

## 3. Service Detection & NSE

### Standard-Vertiefung

```bash
sudo nmap -sV -sC -O -A -p $PORTS -oA detail <TARGET>
```

- `-sV` Service Version
- `-sC` Default-Skripte
- `-O` OS Detection
- `-A` Aggressive (kombiniert obige + Traceroute)

### Targeted NSE-Scripts

```bash
nmap --script=banner,http-title,http-headers,http-methods,http-enum -p 80,443 <TARGET>
nmap --script=smb-enum-shares,smb-enum-users,smb-os-discovery,smb-security-mode -p 445 <TARGET>
nmap --script=ldap-rootdse,ldap-search -p 389 <TARGET>
nmap --script=ssl-enum-ciphers,ssl-cert -p 443 <TARGET>
nmap --script=ssh-auth-methods,ssh2-enum-algos -p 22 <TARGET>
nmap --script=dns-zone-transfer -p 53 <TARGET>
```

### Vuln-Scripts

```bash
nmap --script=vuln -p $PORTS <TARGET>
```

⚠️ Manche Vuln-Scripts sind invasiv (DoS-Risiko). Out-of-Scope-Liste prüfen.

---

## 4. Stealth & Speed Tuning

### Stealth

```bash
sudo nmap -sS -f --data-length 24 -D RND:10 -T2 <TARGET>
```

- `-f` fragmentiert Pakete
- `--data-length 24` Padding gegen Signature-Match
- `-D RND:10` 10 zufällige Decoys
- `-T2` langsam (gegen IDS-Detection)

### Speed

```bash
sudo nmap -sS -p- --min-rate=5000 -T4 --max-retries=2 --host-timeout=5m <TARGET>
```

| Timing | Anwendung                      |
|--------|--------------------------------|
| `-T0`  | extrem langsam (IDS-Evasion)   |
| `-T1`  | sneaky                         |
| `-T2`  | polite                         |
| `-T3`  | normal                         |
| `-T4`  | aggressive (LAN/Lab-Standard)  |
| `-T5`  | insane (oft fehlerhaft)        |

---

## 5. Output-Formate

```bash
nmap ... -oA scanname
```

`-oA` produziert drei Files:
- `.nmap` — menschenlesbar
- `.gnmap` — grepbar
- `.xml` — strukturiert (für `nmap-formatter`, `nmap-parse-output`, Reporting)

### Aus XML weiterverarbeiten

```bash
nmap-converter scan-services.xml output.xlsx
xsltproc scan-services.xml -o scan.html
```

---

## 6. IPv6-Targets

```bash
sudo nmap -6 -sS -p- <TARGET-IPv6>
```

Viele Hosts sind IPv4 gehärtet, IPv6-Stack vergessen. Lohnt sich oft.

---

## 7. Typischer Engagement-Workflow

```bash
# 1. Host Discovery
sudo nmap -sn -PR <RANGE>/24 -oA discovery

# 2. Schneller TCP-All-Ports auf lebenden Hosts
sudo nmap -iL alive.txt -sS -p- --min-rate=2000 -oA tcp-all

# 3. Top-100 UDP
sudo nmap -iL alive.txt -sU --top-ports 100 -oA udp-top100

# 4. Service-Detail nur auf offene Ports
PORTS=$(grep -oP '\d{1,5}/open' tcp-all.gnmap | sort -u | cut -d/ -f1 | tr '\n' ',')
sudo nmap -iL alive.txt -sV -sC -p $PORTS -oA detail

# 5. Targeted NSE pro Service
sudo nmap --script=smb-* -p 445 -iL alive.txt -oA smb
sudo nmap --script=http-* -p 80,443,8080,8443 -iL alive.txt -oA http
```

---

## 8. Service → Folge-Tools

| Service       | Folge-Tools                                                     |
|---------------|-----------------------------------------------------------------|
| 21 FTP        | `nmap --script=ftp-*`, anonyme Logins, `bruteforce`             |
| 22 SSH        | `ssh-audit`, Key-Auth-Discovery, User-Enum (`ssh-username-enum`)|
| 25/465/587    | `smtp-user-enum`, `swaks`, Open-Relay-Test                      |
| 53            | `dig`, `dnsrecon`, Zone-Transfer                                |
| 80/443        | `whatweb`, `wappalyzer`, `ffuf`, `gobuster`, Burp               |
| 88            | `kerbrute`, Impacket Kerberos-Tools                             |
| 110/995       | `hydra` POP3                                                    |
| 111/2049      | `showmount`, NFS-Mounten                                        |
| 135/139/445   | `nxc`, `enum4linux-ng`, `rpcclient`, `smbclient`                |
| 161 SNMP      | `snmpwalk -c public -v1`, `onesixtyone`                         |
| 389/636       | `ldapsearch`, `windapsearch`, BloodHound                        |
| 443           | `testssl.sh`, `sslyze`                                          |
| 873           | `rsync` Module-Listing                                          |
| 1433 MSSQL    | `mssqlclient.py`, `nxc mssql`                                   |
| 2049 NFS      | `showmount -e`, mount mit `nolock`                              |
| 3306 MySQL    | `mysql -h`, `nmap --script=mysql-*`                             |
| 3389 RDP      | `rdesktop`, `xfreerdp`, `hydra rdp`                             |
| 5432 PostgreSQL | `psql -h`, `nmap --script=pgsql-*`                            |
| 5985/5986     | `evil-winrm`, `nxc winrm`                                       |
| 6379 Redis    | `redis-cli -h`, RCE via `module load`                           |
| 8080/8443     | wahrscheinlich Tomcat/Jenkins/Jboss/Weblogic                    |
| 27017 MongoDB | `mongo --host`, NoSQL-Discovery                                 |

---

## 9. Häufige Fehler vermeiden

- **`-Pn` immer setzen, wenn Host-Discovery scheitert** — viele Hosts blocken ICMP
- **Erst gefundene Ports detail-scannen, nicht alle 65k mit `-sC -sV`**
- **Nicht `nmap -A` blind starten** — `-A` macht OS-Detection, das braucht oft Root und ist verräterisch
- **TCP UND UDP scannen** — UDP-Services (SNMP, DNS) sind oft die schwächsten
- **`-T5` vermeiden** — Pakete gehen verloren, Ergebnisse unzuverlässig
- **Output IMMER speichern** (`-oA`) — nichts ist ärgerlicher als verlorene Scans

---

## 10. Quick-Reference

```bash
# Schneller Sanity-Check
sudo nmap -sV --top-ports 1000 <TARGET>

# Voller Sweep
sudo nmap -sS -p- --min-rate=2000 -T4 -oA full <TARGET>

# Detail nach Sweep
sudo nmap -sV -sC -p $(cat ports.txt) -oA detail <TARGET>

# UDP Top-100
sudo nmap -sU --top-ports 100 <TARGET>

# OS Detection
sudo nmap -O <TARGET>

# Aggressive
sudo nmap -A <TARGET>

# Vuln-Scripts
sudo nmap --script=vuln <TARGET>
```

---

## References

- [Nmap Reference Guide](https://nmap.org/book/man.html)
- [NSE Script Documentation](https://nmap.org/nsedoc/)
- [HackTricks – Pentesting Network](https://book.hacktricks.wiki/en/generic-methodologies-and-resources/pentesting-network/index.html)
- [SecLists Wordlists](https://github.com/danielmiessler/SecLists)
