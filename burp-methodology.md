# Burp Suite Methodology

> Mein Workflow für Web- und API-Tests mit Burp Suite Pro.
> Tools führen nicht — Hypothesen führen. Burp validiert sie.

## Methodik

```
Recon → Mapping → Active Test → Exploit → Validate → Report
```

Diese Notes sind komplementär zur [Web/API Testing Checklist](web-api-testing-checklist.md):
hier geht es um den Burp-Workflow, dort um die Liste "habe ich nichts vergessen".

---

## 1. Setup

### Projekt anlegen

- Burp-Project pro Engagement, **kein** Tempt-Project für ernsthafte Arbeit
- Project-Settings → Scope sofort definieren
- Logging in Datei aktivieren (Audit-Trail)
- TLS Pass-Through für Zahlungs-Provider, Drittanbieter-Domains

### Browser-Konfiguration

- Burp-CA Cert in dediziertem Browser-Profil installieren (nicht Daily-Driver)
- FoxyProxy mit Hotkey für An/Aus
- HTTP/2 in Burp aktivieren
- Cookies + Storage des Browser-Profils vor jedem Engagement clearen

### Extensions (Pro)

| Extension                         | Wofür                                       |
|-----------------------------------|---------------------------------------------|
| Logger++                          | granulare History mit Filtern               |
| Autorize                          | Authorization-Bug-Detection                 |
| Param Miner                       | Hidden-Parameter-Discovery                  |
| Active Scan++                     | erweitert Active-Scanner                    |
| Hackvertor                        | Custom-Tag-Encoding/Decoding                |
| JWT Editor                        | JWT-Manipulation                            |
| Turbo Intruder                    | Race-Conditions, Mass-Tests                 |
| Backslash Powered Scanner         | Smart Active Scanner                        |
| HTTP Request Smuggler             | Request-Smuggling-Detection                 |
| InQL                              | GraphQL-Introspection                       |
| Reflector                         | Reflected-Input-Detection                   |
| Software Vulnerability Scanner    | Outdated-Components-Check                   |

---

## 2. Scope-Definition

```
Target → Scope → "Use advanced scope control"
```

**Include:**
- `https://app.example.com/`
- `https://api.example.com/v2/`

**Exclude:**
- `*/logout`
- `*/payment/*` (wenn Out-of-Scope)
- Drittanbieter-Domains
- Heavy-Resource-Endpoints (`/export/*`, große Datenexporte)

`Show only in-scope items` immer aktiv → Proxy-History bleibt sauber.

---

## 3. Recon-Phase

### Passive Mapping

- Beim normalen Browsen läuft Burp im Hintergrund
- Site-Map füllt sich → später analysieren, **nicht** während aktivem Browsen
- HTTP History zeigt Header, Cookies, Tech-Stack

### Active Mapping

- Auf saubere Site-Map: rechte Maustaste auf Host → "Engagement Tools" → "Discover Content"
- "Find Hidden Files" / "Find Comments" / "Find Scripts"
- Param Miner: rechte Maustaste auf Request → "Guess parameters"

### JS-Analyse

- Site-Map → alle `.js` Dateien
- Mit Logger++ Filter `mime:script` → alle JS-Responses
- Search in Response auf:
  - `/api/`
  - `endpoint`, `route`, `path`
  - `secret`, `key`, `token`
  - `aws_`, `firebase`, `gcp`
  - Hardcoded URLs

---

## 4. Active Testing

### Repeater-Workflow

Für jeden Endpunkt:

1. Sauberen Baseline-Request in Repeater
2. **Eine** Variable mutieren (Parameter, Header, Method)
3. Response vergleichen (Status, Length, Timing, Body)
4. Hypothese formulieren ("wenn X, dann Y")
5. Mutation systematisch durchziehen

**Tipp:** Repeater-Tabs benennen (`order-id IDOR`, `auth bypass`) — sonst verliert man sich.

### Intruder

- **Sniper** — ein Payload-Set, eine Position
- **Battering Ram** — gleicher Payload an mehreren Positionen
- **Pitchfork** — pro Position ein Payload-Set, parallel
- **Cluster Bomb** — alle Kombinationen

**Praktische Anwendungen:**
- Username-Enumeration: Sniper auf `username`, beobachten von Length-Diff
- Brute-Force Login (in Lab/CTF): Pitchfork mit User+Pass-Listen
- Forced-Browsing: Sniper auf URL-Pfad mit SecLists-Wordlist
- IDOR: Sniper auf ID-Parameter, Status/Length-Diff

### Active Scanner

- **NICHT** auf Production ohne explizite Freigabe
- Scope-aware Scan, "Audit configuration" anpassen
- Insertion-Points kontrollieren: nur Body-Parameter? Nur Header?

---

## 5. Authentication-Tests

### Session-Handling-Rules

Wenn die App komplexes Auth-Verhalten hat (CSRF-Token pro Request, JWT-Refresh):
Project Options → Sessions → Session Handling Rules.

Beispiel: Token-Refresh-Macro, das vor jedem Repeater-Request läuft.

### Authorization-Tests mit Autorize

1. Als High-Privilege-User normal browsen
2. Autorize an, Low-Privilege-Cookies in "Cookies header" einfügen
3. Filter: `Bypassed!` zeigt Endpoints, die mit Low-Cookies gleich antworten

---

## 6. JWT-Tests

Mit JWT-Editor-Extension:

| Test                      | Vorgehen                                      |
|---------------------------|-----------------------------------------------|
| `alg: none`               | Header auf `none`, Signature leer             |
| HS256 Key-Confusion       | RS256 → HS256 mit Public Key als Secret       |
| `kid`-SQLi / LFI          | `kid` mutieren                                |
| Schwacher Secret          | Token exportieren, `hashcat -m 16500`         |
| Token-Replay              | Expired-Tokens senden                         |

```bash
hashcat -m 16500 jwt.txt jwt.secrets.list
```

---

## 7. Race Conditions (Turbo Intruder)

```python
def queueRequests(target, wordlists):
    engine = RequestEngine(endpoint=target.endpoint, concurrentConnections=30, requestsPerConnection=100, pipeline=False)
    for i in range(30):
        engine.queue(target.req)

def handleResponse(req, interesting):
    if req.status != 429:
        table.add(req)
```

Klassische Race-Condition-Targets:
- Coupon-Redemption
- Inventory-Buchung
- Password-Reset-Token-Validation
- 2FA-Submit

---

## 8. Common Findings Checklist

| Kategorie         | Findings                                                  |
|-------------------|-----------------------------------------------------------|
| Authentication    | User-Enum, schwache Lockout, Reset-Token-Reuse            |
| Authorization     | IDOR, BOLA, BFLA, Tenant-Isolation                        |
| Injection         | SQLi, NoSQLi, SSTI, OS-CmdInj, LDAP-Inj                   |
| XSS               | Reflected, Stored, DOM, mXSS                              |
| Cryptography      | JWT alg none, schwacher Secret, Key-Confusion             |
| File Handling     | Upload-Bypass, LFI, Path-Traversal, ZipSlip               |
| SSRF              | Cloud-Metadata, Localhost-Bypass, DNS-Rebinding           |
| CSRF              | Fehlende Tokens, schwacher SameSite                       |
| Logic             | Race-Condition, Workflow-Skip, Negative-Werte             |
| Disclosure        | Source-Maps, `.git/`, Verbose-Errors, Stack-Traces        |
| Misconfiguration  | CORS, CSP, fehlende Security-Header, Default-Creds        |
| Deserialization   | pickle, Java, .NET, PHP                                   |

---

## 9. Documentation während des Tests

**Pro gefundenem Issue sofort Notiz:**

```
ID: TEST-001
Endpoint: POST /api/v2/orders/123
Severity (estimate): High
Type: IDOR
Repro: Send POST as user A with order_id of user B → succeeds
Evidence: Burp Project file, Repeater tab "IDOR-orders"
Status: confirmed
```

→ direkt in Burp's "Issues" als manuelles Issue speichern.

Beim Reporting später ist diese Notiz Gold wert.

---

## 10. Methodologie-Prinzipien

- **Hypothese vor Tool.** Tool soll Hypothese validieren, nicht ersetzen.
- **Eine Variable pro Mutation.** Sonst weiß man am Ende nicht, was den Bug auslöste.
- **Unterschiede beobachten, nicht nur Erfolg.** Status-Code, Length, Timing, Behavior.
- **Notizen synchron mit Test.** Bug-Reports beim Test schreiben, nicht hinterher rekonstruieren.
- **Auf Annahmen achten.** "Das wird wohl XYZ machen" = Bug-Versteck. Verifizieren.

---

## References

- [PortSwigger Web Security Academy](https://portswigger.net/web-security)
- [Burp Suite Documentation](https://portswigger.net/burp/documentation)
- [HackTricks – Pentesting Web](https://book.hacktricks.wiki/en/network-services-pentesting/pentesting-web/index.html)
- [PayloadsAllTheThings](https://github.com/swisskyrepo/PayloadsAllTheThings)
- [SecLists](https://github.com/danielmiessler/SecLists)
- [GTFOBins](https://gtfobins.github.io/) (Linux Binary-Lookup)
- [LOLBAS](https://lolbas-project.github.io/) (Windows Binary-Lookup)
- [HackTricks](https://book.hacktricks.wiki/)
- [Impacket](https://github.com/fortra/impacket)
- [BloodHound](https://github.com/SpecterOps/BloodHound)
