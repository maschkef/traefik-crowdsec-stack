# Changelog

Diese Datei dokumentiert die Abweichungen dieses Forks gegenüber
`Psycho0verload/traefik-crowdsec-stack` (main). Sie soll den Überblick
über die Änderungen erleichtern und ist so formuliert, dass die
Änderungen auch für Dritte nachvollziehbar bleiben.

## Changed

### TLS / Zertifikate

- Certresolver `http_resolver` (HTTP-01) und `tls_resolver` (TLS-ALPN)
  entfernt und durch `myresolver` (DNS-01 via Cloudflare) ersetzt.
  Grund: Wildcard-fähig und funktioniert auch für Dienste, die nicht
  öffentlich erreichbar sind.
- Authentifizierung gegen Cloudflare via `CF_DNS_API_TOKEN`
  (scoped Token, Zone:Read + DNS:Edit). `CF_API_EMAIL` und `CF_API_KEY`
  (Global API Key) werden bewusst nicht verwendet, da dieser Vollzugriff
  auf den gesamten Cloudflare-Account gewährt.
- Neuer Volume-Mount `data/traefik/certs/dns_letsencrypt.json`;
  alte ACME-Storage-Mounts (`acme_letsencrypt.json`,
  `tls_letsencrypt.json`) entfernt.
  Dateien: `data/traefik/traefik.yml.sample`, `compose/traefik.yml`,
  `.env.sample`.

### CrowdSec-Bouncer

- Der bisherige ForwardAuth-Bouncer-Container
  (`traefik-crowdsec-bouncer`, freifunkMUC-Fork) wurde entfernt und
  durch `maxlerebourg/crowdsec-bouncer-traefik-plugin` als natives
  Traefik-Plugin ersetzt. Grund: der ForwardAuth-Fork wird nicht mehr
  gepflegt; das Plugin erlaubt zusätzlich die Anbindung von AppSec/WAF.
  Dateien: gelöscht `compose/traefik-crowdsec-bouncer.yml` und
  `data/traefik-crowdsec-bouncer/`; geändert
  `data/traefik/traefik.yml.sample` (`experimental.plugins`),
  `data/traefik/dynamic_conf/http.middlewares.traefik-bouncer.yml.sample`.
- Der `BOUNCER_KEY_TRAEFIK` wird über die Go-Template-Syntax
  `{{ env "BOUNCER_KEY_TRAEFIK" }}` in die dynamische Config eingesetzt.
  Der Traefik-File-Provider substituiert kein `${VAR}` (das ist ein
  Docker-Compose-Feature); damit `env` auflösen kann, wird die Variable
  zusätzlich als Container-Umgebungsvariable im `compose/traefik.yml`
  weitergereicht.

### Traefik-Dashboard

- Middleware-Kette am Dashboard-Router zusätzlich um `default@file`
  erweitert (`default@file, traefik-dashboard-auth@file`), damit
  security-headers und gzip auch dort greifen.
- Certresolver-Referenz am Dashboard-Router von `tls_resolver` auf
  `myresolver` umgestellt.

### Middlewares (Konsolidierung)

- `http.middlewares.default-security-headers.yml.sample` und
  `http.middlewares.gzip.yml.sample` in
  `http.middlewares.default.yml.sample` zusammengeführt. Enthält jetzt
  in einer Datei: `chain: default`, `default-security-headers` und
  `gzip`.

### TLS-Cipher-Politik

- Legacy-Cipher `TLS_ECDHE_RSA_WITH_AES_256_CBC_SHA` entfernt.
- Kurven-Präferenz nach Mozilla-Intermediate: `X25519`, `CurveP256`,
  `CurveP384` (statt vorher `CurveP521`, `CurveP384`).

## Added

### CrowdSec-Basiskonfiguration (neue Sample-Dateien)

- `data/crowdsec/config/acquis.yaml.sample` — Datenquellen syslog
  (`/var/log/auth.log`, `/var/log/syslog`) und Traefik-Access-Log.
- `data/crowdsec/config/acquis.d/appsec.yaml.sample` —
  AppSec-Datenquelle auf `0.0.0.0:7422` mit den Collections
  `crowdsecurity/appsec-default`, `crowdsecurity/crs-inband`,
  `custom/hooks`.
- `data/crowdsec/config/appsec-configs/custom-hooks.yaml.sample` —
  lokale False-Positive-Ausnahme für CRS-Regel 920420 („Request
  content type is not allowed by policy", ausgelöst u. a. durch
  shoutrrr/Watchtower-Benachrichtigungen an ntfy). IPs als
  Platzhalter. Bewusst per custom-hooks statt cscli allowlists
  umgesetzt, damit die Ausnahme regel- und nicht IP-scoped ist und
  die restliche Erkennung für die betroffenen Quellen aktiv bleibt.

### `.env`-Variablen

- `SERVICES_TRAEFIK_IMAGE`, `SERVICES_TRAEFIK_IMAGE_VERSION` — statt
  compose-Fallback über `${VAR:-default}`.
- `BOUNCER_KEY_TRAEFIK`, `BOUNCER_KEY_FIREWALL` — pro Host neu
  generieren, siehe README.
- `CF_DNS_API_TOKEN` — Cloudflare-API-Token für DNS-01.

### README-Ergänzungen

- Cloudflare-Token-Anleitung inkl. der zwingend nötigen Berechtigungen
  (Zone:Read + DNS:Edit, alle Zonen) und expliziter Warnung vor
  Global API Key sowie der nicht existierenden Variable
  `CF_DNS_API_KEY`.
- Bouncer-Key-Erzeugung: `openssl rand -base64 48 | tr -dc 'A-Za-z0-9'
  | head -c 32`. Die Registrierung erfolgt automatisch beim ersten
  Container-Start ueber die `BOUNCER_KEY_*`-Umgebungsvariablen des
  CrowdSec-Docker-Images.
- Neue Abschnitte „Abweichungen vom Upstream", „Pro Host anzupassen"
  und „Host-/dienstspezifische Overrides" (nur der Override-
  Mechanismus, ohne konkrete Dienste zu nennen).

### Installations-Skript

- `first_install.sh`: Bouncer-Keys werden mit
  `tr -dc 'A-Za-z0-9'` und fixierter Länge (32 Zeichen) erzeugt, um
  Sonderzeichen in `.env`-Werten und URLs zu vermeiden.
- Kopierschritte an die neue Sample-Struktur angeglichen; für
  `dns_letsencrypt.json` wird die Datei vorab mit `chmod 600` angelegt,
  damit lego sie nicht world-readable erstellt.

## Removed

- `compose/traefik-crowdsec-bouncer.yml` und
  `data/traefik-crowdsec-bouncer/` (ForwardAuth-Bouncer-Container).
- `data/traefik/dynamic_conf/http.middlewares.default-security-headers.yml.sample`
  und `data/traefik/dynamic_conf/http.middlewares.gzip.yml.sample`
  (siehe Konsolidierung unter Changed).

## `.gitignore`

- `data/crowdsec/config` durch feingranulare Regeln ersetzt, damit
  neue Sample-Dateien in `acquis.d/` und `appsec-configs/`
  versioniert werden können, echte YAML-Configs aber weiterhin
  ignoriert bleiben (beide Endungen: `*.yaml` und `*.yml`).
- Ergänzt: `data/traefik/certs/*.json` (deckt `dns_letsencrypt.json`
  ab), `data/traefik/.env`, `data/socket-proxy/.env`,
  `data/crowdsec/.env`.

## Nachbesserungen an eigenen Fork-Änderungen

Die folgenden Punkte sind **keine** Fixes für Upstream-Bugs, sondern
Nachbesserungen an Regressionen, die durch die weiter oben gelisteten
Fork-Umbauten (DNS-01, `.env.sample`-Platzhalter, AppSec-Basiskonfiguration)
erst entstanden sind. Beim ersten End-to-End-Test aufgefallen.

### `.env`-Handling (Installations-Skript + README)

- Ursache: Durch die neu eingeführten leeren Platzhalter für
  `BOUNCER_KEY_TRAEFIK`, `BOUNCER_KEY_FIREWALL` und `CF_DNS_API_TOKEN`
  in `.env.sample` (siehe Added → `.env`-Variablen) erzeugten die
  bisherigen `>>`-Appends in `first_install.sh` und README jeweils
  eine zweite Zeile mit demselben Schlüssel. Docker Compose nahm zwar
  den letzten Wert, die Datei war aber unsauber und nicht idempotent
  (jeder Rerun verdoppelte weiter).
- Neue Helper-Funktion `upsert_env` in `first_install.sh`: ersetzt
  einen vorhandenen `KEY=`-Eintrag per `sed -i` in place (Delimiter
  `|`, Anker `^KEY=`) und fällt nur auf `>>` zurück, wenn der Key
  gar nicht existiert. Damit bleibt `.env.sample` als self-documenting
  Template erhalten und die `.env` bleibt duplikatfrei.
- README-Snippets für `CF_DNS_API_TOKEN` (Schritt 5.1) und
  Bouncer-Keys (Schritt 6.5) analog auf `sed -i` umgestellt.

### CrowdSec startet nicht (AppSec-Rule fehlt)

- Ursache: Die im Fork neu hinzugekommene AppSec-Konfiguration
  (siehe Added → CrowdSec-Basiskonfiguration) referenziert
  `crowdsecurity/crs-inband` in `APPSEC_CONFIGS` und in
  `data/crowdsec/config/acquis.d/appsec.yaml`. Dieses AppSec-Config
  verweist intern auf die appsec-rule `crowdsecurity/crs`, die aber
  weder von den installierten Collections
  (`appsec-virtual-patching`, `appsec-generic-rules`) noch anderweitig
  mit installiert wird. Ergebnis beim Start:
  `no appsec-rules found for pattern crowdsecurity/crs`.
- Fix: In `data/crowdsec/.env.sample`
  `APPSEC_RULES="crowdsecurity/crs"` aktiviert.
- README-Abschnitt 6.3 (AppSec) um einen Hinweis auf `APPSEC_RULES`
  ergänzt.

### Fehlender Cloudflare-Token-Prompt in `first_install.sh`

- Ursache: Mit dem Umstieg auf DNS-01 wurde `CF_DNS_API_TOKEN` als
  Platzhalter in `.env.sample` neu eingeführt, aber im
  Installations-Skript nie abgefragt.
- Neuer Schritt `Frage nach Cloudflare-API-Token (CF_DNS_API_TOKEN)`
  mit interaktivem Prompt, Nicht-Leer-Validierung und Bestätigung
  analog zum bestehenden E-Mail-Schritt. Der Wert wird via
  `upsert_env` in `.env` gesetzt. `total_steps` von 18 auf 19 erhöht.