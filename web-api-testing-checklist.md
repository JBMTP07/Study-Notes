# Web & API Testing Checklist

> Strukturierte Test-Checkliste für Webanwendungen und REST-/GraphQL-APIs.
> Komplementär zur Burp-Methodik — diese hier ist die "habe ich nichts vergessen?"-Liste.

Orientiert an OWASP WSTG, OWASP API Security Top 10 und eigenen Bug-Bounty-Erfahrungen.

---

## Pre-Engagement

- [ ] Scope schriftlich bestätigt (Domains, IPs, Endpoints)
- [ ] Out-of-Scope dokumentiert (Subdomains, Drittanbieter, Zahlungs-Provider)
- [ ] Test-Accounts vorhanden (mind. 2 Rollen: Admin + Standard-User)
- [ ] Rate-Limit-Policy bekannt (Throttle pro Sekunde)
- [ ] Notfall-Kontakt für DoS-Verdacht
- [ ] Burp-Lizenz, VPN, Saubere VM bereit

---

## 1. Recon & Mapping

### Domain & Infrastructure

- [ ] DNS-Records geprüft: `A`, `AAAA`, `CNAME`, `MX`, `TXT`, `NS`
- [ ] Subdomain-Enumeration: `subfinder`, `amass`, `crt.sh`, Wayback
- [ ] WAF / CDN-Detection (`wafw00f`)
- [ ] Reverse-DNS / IP-Range-Mapping
- [ ] CloudFront / Cloudflare-Origin-IP-Suche

### Tech Stack Fingerprinting

- [ ] HTTP-Header geprüft (`Server`, `X-Powered-By`)
- [ ] Cookies geprüft (Naming-Konventionen verraten Stack)
- [ ] HTML-Kommentare, Meta-Tags
- [ ] `wappalyzer` / `whatweb`
- [ ] Favicon-Hash via Shodan
- [ ] Error-Pages bewusst triggern (verraten Frameworks)

### Content Discovery

- [ ] `robots.txt`, `sitemap.xml`, `.well-known/`
- [ ] Directory-Brute mit `ffuf` / `feroxbuster` (Wordlist: SecLists)
- [ ] JS-Files geparsed (`linkfinder`, `gau`, `katana`) für API-Endpoints
- [ ] GitHub-Dorks für Public-Leaks (`org:target password`)
- [ ] Wayback Machine: alte Pfade
- [ ] Source-Maps verfügbar? (`*.js.map`)

---

## 2. Authentication

- [ ] Account-Registrierung erlaubt? Mailadressen-Validierung?
- [ ] Username-Enumeration (Login, Signup, Password-Reset, Error-Messages)
- [ ] Brute-Force-Protection (Lockout, Captcha, Rate-Limit)
- [ ] Password-Policy (Mindestlänge, Komplexität, gesperrte Listen)
- [ ] Session-Token: zufällig, ausreichend lang, signiert
- [ ] "Remember Me" sicher implementiert
- [ ] Logout: invalidiert Session serverseitig?
- [ ] Concurrent-Session-Handling
- [ ] OAuth/SSO: `state`-Parameter, Redirect-URI-Validation
- [ ] MFA-Bypass via API direkt vs. Web-UI
- [ ] Password-Reset-Token: vorhersagbar? Single-Use? Time-Limited?
- [ ] Password-Reset-Link: Host-Header-Injection?

---

## 3. Authorization

### Object-Level (IDOR)

- [ ] Numerische IDs durchprobieren (`/users/1234`)
- [ ] UUIDs durchprobieren (sind sie wirklich zufällig?)
- [ ] HPP / Parameter-Pollution (`?id=1&id=2`)
- [ ] Role-Switching via Cookie / Header
- [ ] Direct-Object-Access in Admin-Endpoints als normaler User

### Function-Level

- [ ] Admin-Endpoints von Standard-User erreichbar?
- [ ] Method-Tampering (`GET → POST`, `POST → PUT`)
- [ ] Verb-Tunneling (`X-HTTP-Method-Override`)
- [ ] API-Versions parallel testen (`/v1/admin` vs `/v2/admin`)

### Tenant-Isolation

- [ ] Cross-Tenant-Zugriff bei Multi-Tenant-Apps
- [ ] Org-/Workspace-Switching im Token

---

## 4. Input Handling

### Injection

- [ ] SQL-Injection (Burp Active + manuell + `sqlmap`)
- [ ] NoSQL-Injection (`{"$ne": null}`, MongoDB-Operatoren)
- [ ] LDAP-Injection
- [ ] OS-Command-Injection (Filenames, Profile-Felder, Hostnames)
- [ ] Template-Injection (SSTI)
- [ ] Header-Injection (CRLF, Host-Header)
- [ ] HQL / ORM-Specific
- [ ] Prompt-Injection bei LLM-Backends

### XSS

- [ ] Reflected XSS (Query-Parameter, Headers)
- [ ] Stored XSS (Profile, Comments, Filenames, Image-Metadata)
- [ ] DOM-XSS (Sources & Sinks, `location.hash`, `postMessage`)
- [ ] Mutation-XSS in Sanitizer
- [ ] CSP-Bypass (Trusted Domains, Script-Gadgets)

### File Uploads

- [ ] Extension-Filter umgehen (`.php.png`, doppelte Endungen)
- [ ] MIME-Type-Spoofing
- [ ] Polyglot-Files (gültiges Bild + PHP)
- [ ] Path-Traversal in Filename
- [ ] Zip-Slip
- [ ] XXE in `.docx` / `.xlsx` / SVG

### Deserialization

- [ ] Java (`O:` `rO0AB`), .NET (`AAEAAAD`), Python (`pickle`), PHP (`O:`)
- [ ] YAML mit `!!python/object`
- [ ] Cookies / Tokens als Deserialisierungs-Eingang

---

## 5. API-spezifisch (REST / GraphQL)

### REST

- [ ] Schema-Disclosure (`/openapi.json`, `/swagger`)
- [ ] HTTP-Verbs: `OPTIONS`, `TRACE`, `PATCH` testen
- [ ] Mass-Assignment / Property-Injection (`{"role":"admin"}`)
- [ ] Excessive-Data-Exposure (gibt API mehr zurück, als UI braucht?)
- [ ] Lack-of-Resources (Rate-Limit, Pagination-Limits, große Payloads)
- [ ] BOLA / Broken Object-Level-Authorization (siehe IDOR)
- [ ] Inkonsistente Auth zwischen `v1` / `v2` / `internal`

### GraphQL

- [ ] Introspection aktiviert?
- [ ] `__schema`-Query erlaubt? Mutations?
- [ ] Batched-Queries (DoS-Vektor + Auth-Bypass)
- [ ] Field-Level-Authorization
- [ ] Aliases für Rate-Limit-Bypass
- [ ] Query-Depth-Limit
- [ ] CSRF auf GraphQL-Endpoints

---

## 6. Business Logic

- [ ] Race Conditions (parallele Requests, z.B. Coupon-Redemption)
- [ ] Negative-Werte (Quantität `-5` → Refund-Bug)
- [ ] Integer-Overflows
- [ ] Multi-Step-Workflows: Schritte überspringen
- [ ] Workflow-State-Manipulation (`status: pending → completed`)
- [ ] Abuse von Reset-/Refund-Funktionen
- [ ] Coupon-/Voucher-Replay
- [ ] TOCTOU bei Inventory / Buchung

---

## 7. Sensitive Data Exposure

- [ ] PII in JSON-Responses (Felder, die UI nicht braucht)
- [ ] PII in Error-Messages
- [ ] PII in URL-Parameters (Logging-Risiko)
- [ ] Backup-Files (`.bak`, `.old`, `.swp`)
- [ ] `.git/` exposed (`.git/config`, `git-dumper`)
- [ ] Source-Maps in Production
- [ ] API-Keys in JS-Bundles
- [ ] AWS-Keys, GCP-Keys, JWT-Secrets in Repos
- [ ] PostgREST / Supabase Default-API exposed

---

## 8. SSRF / RFI / RCE

- [ ] URL-Felder testen (`?url=`, `?image=`, Webhooks, OAuth-Callbacks)
- [ ] Localhost-Bypass: `127.0.0.1`, `[::1]`, `2130706433`, `0.0.0.0`, `localhost.example.com`
- [ ] Cloud-Metadata: `169.254.169.254` (AWS, Azure, GCP)
- [ ] DNS-Rebinding bei zeitversetzten Lookups
- [ ] Blind-SSRF via DNS-Out-Of-Band (Burp Collaborator)
- [ ] Gopher-/File-Schemes
- [ ] PDF-Generators / Headless-Chrome-Backends (oft SSRF-anfällig)

---

## 9. Session & Cookie Hygiene

- [ ] `Secure`-Flag
- [ ] `HttpOnly`-Flag
- [ ] `SameSite=Strict|Lax`
- [ ] Session-Token rotiert nach Login
- [ ] CSRF-Token: pro Session, nicht raterbar
- [ ] CSRF-Schutz auf state-changing Endpoints
- [ ] CORS-Policy nicht zu permissiv (`Access-Control-Allow-Origin: *` mit Credentials? → kritisch)

---

## 10. Crypto & Tokens

- [ ] JWT: `alg: none`-Bypass
- [ ] JWT: HS256 mit Public-Key signiert (Key-Confusion)
- [ ] JWT: schwacher Secret (`hashcat -m 16500`)
- [ ] JWT: `kid`-Parameter für SQLi / LFI
- [ ] Session-Token: ausreichend Entropy?
- [ ] TLS-Konfiguration (`testssl.sh`)
- [ ] HSTS, Certificate-Transparency

---

## 11. Headers & Security Misconfig

- [ ] `Content-Security-Policy` vorhanden und sinnvoll
- [ ] `X-Frame-Options` / `frame-ancestors`
- [ ] `Referrer-Policy`
- [ ] `X-Content-Type-Options: nosniff`
- [ ] `Permissions-Policy`
- [ ] `Strict-Transport-Security`
- [ ] Verbose Server-Banner deaktiviert
- [ ] Default-Errors deaktiviert
- [ ] Debug-Modi (Django Debug, Symfony Profiler) nicht in Prod

---

## 12. Client-Side

- [ ] PostMessage-Origin-Check
- [ ] LocalStorage-Token / Sensitive-Data
- [ ] Unsichere `eval()` / `Function()`
- [ ] DOM-Clobbering
- [ ] OAuth-Token in URL-Fragment-Logging

---

## Tools-Quick-Reference

| Phase                 | Tool                                  |
|-----------------------|---------------------------------------|
| Subdomains            | subfinder, amass, crt.sh              |
| Content Discovery     | ffuf, feroxbuster, gobuster, katana   |
| JS-Parsing            | linkfinder, gau, GoLinkFinder         |
| Active Web-Test       | Burp Suite Pro, ZAP                   |
| API-Test              | Postman + Burp, Insomnia, mitmproxy   |
| GraphQL               | InQL, GraphQL Voyager, graphw00f      |
| SQLi                  | sqlmap                                |
| WAF-Detection         | wafw00f                               |
| Cloud-Metadata        | Burp Collaborator                     |
| TLS                   | testssl.sh, sslyze                    |
| JWT                   | jwt_tool, jwt.io                      |

---

## References

- [OWASP Web Security Testing Guide v4.2](https://owasp.org/www-project-web-security-testing-guide/v42/)
- [OWASP API Security Top 10 (2023)](https://owasp.org/API-Security/editions/2023/en/0x11-t10/)
- [PortSwigger Web Security Academy](https://portswigger.net/web-security)
- [HackTricks](https://book.hacktricks.wiki/)
- [PayloadsAllTheThings](https://github.com/swisskyrepo/PayloadsAllTheThings)
- [SecLists Wordlists](https://github.com/danielmiessler/SecLists)
