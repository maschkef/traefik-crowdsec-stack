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
- Bouncer-Key-Erzeugung via `cscli bouncers add` mit
  `tr -dc 'A-Za-z0-9'`-Filterung des Zufalls-Strings.
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
