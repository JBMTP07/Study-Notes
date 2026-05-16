# Pentest Study Notes

![Focus](https://img.shields.io/badge/Focus-Offensive%20Security-red)
![Content](https://img.shields.io/badge/Content-Methodology-blue)
![Status](https://img.shields.io/badge/Status-Active-success)
![Language](https://img.shields.io/badge/Lang-DE%20%2F%20EN-lightgrey)

> Strukturierte Methodik-Sammlung aus meinem Lernpfad und der Praxis in
> Bug Bounty, HackTheBox und eigenem Security-Lab. Kein Cheatsheet-Dump —
> jede Note ist ein operativer Workflow mit Begründung, Tools und Referenzen.

---

## Inhalt

| Note                                                                | Schwerpunkt                                                                 |
|---------------------------------------------------------------------|-----------------------------------------------------------------------------|
| [Linux Privilege Escalation](linux-privesc-notes.md)                | SUID, Sudo, Cron, Capabilities, Container-Escape, Kernel-CVEs               |
| [Windows Privilege Escalation](windows-privesc-notes.md)            | Token-Privileges, Service-Misconfig, AlwaysInstallElevated, DLL-Hijack      |
| [Active Directory Attacks](ad-attack-notes.md)                      | Kerberoasting, AS-REP, RBCD, ADCS (ESC1–ESC11), NTLM-Relay, BloodHound      |
| [Nmap Enumeration Playbook](nmap-enumeration-playbook.md)           | Zwei-Stufen-Workflow, NSE, Service → Folge-Tools-Map                        |
| [Burp Suite Methodology](burp-methodology.md)                       | Recon, Mapping, Repeater/Intruder, JWT, Race Conditions, Reporting          |
| [Web & API Testing Checklist](web-api-testing-checklist.md)         | OWASP-aligned Checkliste — Auth, AuthZ, Injection, GraphQL, SSRF            |
| [Pentest Report Template](pentest-report-template.md)               | Industriestandard-Reportstruktur, CVSS, Finding-Template, Angriffskette     |

---

## Methodisches Prinzip

```
Recon → Enumerate → Hypothesize → Validate → Escalate → Report
```

Tools unterstützen Methodik — sie ersetzen sie nicht. Wer LinPEAS-Output blind
durcharbeitet, ohne zu wissen, was er sucht, verliert sich. Wer eine Hypothese
aufbaut und sie systematisch validiert, findet auch das, was Tools nicht erkennen.

---

## Skill-Themen im Überblick

**Linux:**
SUID Abuse · Sudo Misconfigurations · Cron Hijacking · PATH Injection ·
Capabilities · Wildcard Abuse · Container Escapes · Kernel CVEs (DirtyPipe,
PwnKit, OverlayFS)

**Windows:**
Token Privileges (SeImpersonate / SeBackup / SeDebug) · Unquoted Service Paths ·
Weak Service ACLs · AlwaysInstallElevated · DLL Search Order · Credential
Hunting (LSASS, SAM, DPAPI, GPP) · UAC-Bypass

**Active Directory:**
Kerberoasting · AS-REP Roasting · BloodHound Path Analysis · ACL Abuse
(GenericAll, GenericWrite, WriteDACL) · Resource-Based Constrained Delegation ·
S4U2Self / S4U2Proxy · ADCS ESC1 – ESC11 · NTLM Relay · Coercion (PetitPotam,
PrinterBug) · LAPS / gMSA Abuse

**Web & API:**
OWASP Top 10 Methodology · API Security Top 10 (BOLA, BFLA, Mass-Assignment) ·
JWT Attacks (alg:none, Key-Confusion, weak secret) · GraphQL Introspection
& Field-AuthZ · Race Conditions · SSRF / Cloud-Metadata · Deserialization
(Pickle / Java / .NET / PHP) · Business-Logic-Flaws

**Reporting:**
CVSS v3.1 Scoring · Finding-Template (Beschreibung, Repro, Impact, Remediation) ·
Angriffsketten-Erzählung · strategische Empfehlungen · OWASP-WSTG-Bezug

---

## Verwendung

Diese Notes dienen mir selbst als Operator-Referenz und sind öffentlich, weil sie
auch anderen helfen können. Ich halte sie aktuell, wenn ich neue Vektoren in
Bug Bounty oder im Lab entdecke.

**Lese-Reihenfolge** wenn du selbst einsteigen willst:

1. [Nmap Playbook](nmap-enumeration-playbook.md) — Service-Inventar bauen
2. [Burp Methodology](burp-methodology.md) — Web/API-Testing-Workflow
3. [Web & API Checklist](web-api-testing-checklist.md) — was systematisch geprüft wird
4. [Linux PrivEsc](linux-privesc-notes.md) / [Windows PrivEsc](windows-privesc-notes.md)
5. [AD Attacks](ad-attack-notes.md) — wenn AD im Spiel ist
6. [Report Template](pentest-report-template.md) — am Ende kommt die Ablieferung

---

## Verwandte Repositories

- [bug-bounty-reports](https://github.com/JBMTP07/bug-bounty-reports) — sanitisierte Public-Reports realer Findings (Chained RCE, Command Injection)
- [HTB-Writeups](https://github.com/JBMTP07/HTB-Writeups) — meine deutschsprachigen Walkthroughs zu retired HackTheBox Machines
- [JBMTP07.github.io](https://github.com/JBMTP07/JBMTP07.github.io) — mein Pentest-Portfolio

---

## Hinweis

Diese Unterlagen dienen ausschließlich Bildungs- und Lernzwecken im Rahmen
autorisierter Sicherheitstests. Keine Anwendung gegen Systeme ohne explizite
Erlaubnis.

---

**Josef Basner** · Penetration Tester · Red Team Operator · Security Researcher
[github.com/JBMTP07](https://github.com/JBMTP07) · [jbmtp07.github.io](https://jbmtp07.github.io)
