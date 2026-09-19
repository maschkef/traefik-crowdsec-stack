# Traefik-CrowdSec-Stacks

Diese Anleitung beschreibt die manuelle Installation und Konfiguration des Traefik-CrowdSec-Stacks, ohne Verwendung des automatischen Installationsskripts. Bitte folgen Sie den Schritten sorgfältig.

## Voraussetzungen

- Root-Zugriff auf den Server
- Docker und Docker Compose müssen installiert sein
- Apache2 Utils (htpasswd) und OpenSSL müssen installiert sein

## Script
![Ubuntu 20.04 - Testing](https://img.shields.io/badge/Ubuntu_20.04-07--10--2024-orange?logo=ubuntu)
![Ubuntu 22.04 - Testing](https://img.shields.io/badge/Ubuntu_22.04-07--10--2024-orange?logo=ubuntu)
![Ubuntu 24.04 - Testing](https://img.shields.io/badge/Ubuntu_24.04-07--10--2024-orange?logo=ubuntu)
![Debian 11 - Testing](https://img.shields.io/badge/Debian_11_(Bullseye)-07--10--2024-A81D33?logo=debian&logoColor=white)
![Debian 12 - Testing](https://img.shields.io/badge/Debian_12_(Bookworm)-07--10--2024-A81D33?logo=debian&logoColor=white)
### 1. Repository klonen

Als erstes müssen Sie das Repository auf Ihren Server klonen:

```bash
mkdir -p /opt/containers/
git clone https://github.com/psycho0verload/traefik-crowdsec-stack /opt/containers/traefik-crowdsec-stack
cd /opt/containers/traefik-crowdsec-stack
sudo chmod +x first_install.sh
sudo ./first_install.sh
```

## Manuelle Anleitung
![Ubuntu 20.04 - Testing](https://img.shields.io/badge/Ubuntu_20.04-07--10--2024-orange?logo=ubuntu)
![Ubuntu 22.04 - Testing](https://img.shields.io/badge/Ubuntu_22.04-07--10--2024-orange?logo=ubuntu)
![Ubuntu 24.04 - Testing](https://img.shields.io/badge/Ubuntu_24.04-07--10--2024-orange?logo=ubuntu)
![Debian 11 - Testing](https://img.shields.io/badge/Debian_11_(Bullseye)-07--10--2024-A81D33?logo=debian&logoColor=white)
![Debian 12 - Testing](https://img.shields.io/badge/Debian_12_(Bookworm)-07--10--2024-A81D33?logo=debian&logoColor=white)

Die gesamte Anleitung wird als `root`-User durchgeführt!
### 1. Repository klonen
Als erstes müssen Sie das Repository auf Ihren Server klonen:

```bash
sudo su
mkdir -p /opt/containers/
git clone https://github.com/psycho0verload/traefik-crowdsec-stack /opt/containers/traefik-crowdsec-stack
cd /opt/containers/traefik-crowdsec-stack
```

### 2. Docker und Docker Compose installieren

Falls Docker und Docker Compose noch nicht installiert sind, folgen Sie der offiziellen Anleitung:

- [Docker Installation](https://docs.docker.com/engine/install)
- [Docker Compose Installation](https://docs.docker.com/engine/install)

Verifizieren Sie die Installation mit den folgenden Befehlen:

```bash
docker --version
docker compose version
```

### 3. Apache2 Utils und OpenSSL installieren

Um einen Benutzer für die HTTP-Basic-Authentifizierung zu erstellen, benötigen Sie htpasswd, das in apache2-utils enthalten ist. Sie können es mit folgendem Befehl installieren:

```bash
apt update
apt install -y apache2-utils openssl
```

### 4. Konfigurationsdateien kopieren

Kopieren Sie die erforderlichen Konfigurationsdateien aus den .sample-Vorlagen. Stellen Sie sicher, dass Sie im Arbeitsverzeichnis des Projekts sind:

```bash
cp .env.sample .env
cp data/crowdsec/.env.sample data/crowdsec/.env
cp data/socket-proxy/.env.sample data/socket-proxy/.env
cp data/traefik/.env.sample data/traefik/.env
cp data/traefik/traefik.yml.sample data/traefik/traefik.yml
mkdir -p data/traefik/certs
echo '{}' > data/traefik/certs/dns_letsencrypt.json
chmod 600 data/traefik/certs/dns_letsencrypt.json
cp data/traefik/dynamic_conf/http.middlewares.default.yml.sample data/traefik/dynamic_conf/http.middlewares.default.yml
cp data/traefik/dynamic_conf/http.middlewares.traefik-bouncer.yml.sample data/traefik/dynamic_conf/http.middlewares.traefik-bouncer.yml
cp data/traefik/dynamic_conf/http.middlewares.traefik-dashboard-auth.yml.sample data/traefik/dynamic_conf/http.middlewares.traefik-dashboard-auth.yml
cp data/traefik/dynamic_conf/tls.yml.sample data/traefik/dynamic_conf/tls.yml
cp data/crowdsec/config/acquis.yaml.sample data/crowdsec/config/acquis.yaml
cp data/crowdsec/config/acquis.d/appsec.yaml.sample data/crowdsec/config/acquis.d/appsec.yaml
cp data/crowdsec/config/appsec-configs/custom-hooks.yaml.sample data/crowdsec/config/appsec-configs/custom-hooks.yaml
```

### 5. SSL-Zertifikate und Domain konfigurieren

Dieser Stack verwendet **DNS-01-Validierung über Cloudflare** für Let's Encrypt. Vorteile:

- Wildcard-Zertifikate möglich (`*.example.com`).
- Funktioniert auch für Dienste, die nicht öffentlich per HTTP erreichbar sind.
- Die Ports 80/443 müssen für die Validierung nicht offen sein (nur für den späteren Betrieb).

#### 5.1 Cloudflare-API-Token anlegen

Ohne gültiges Token schlägt die Zertifikatsausstellung fehl.

1. Cloudflare-Dashboard → „My Profile" → „API Tokens" → „Create Token" → „Create Custom Token".
2. Berechtigungen (**beide** zwingend):
    - **Zone / Zone / Read**
    - **Zone / DNS / Edit**

    `lego` (die ACME-Bibliothek von Traefik) muss den Domainnamen zunächst zu einer internen Zone-ID auflösen. Dafür reicht `DNS:Edit` allein nicht.
3. Zone Resources: **Include / All zones** (nicht auf eine einzelne Zone einschränken).
4. Token erzeugen und in die `.env` eintragen:
    ```bash
    echo 'CF_DNS_API_TOKEN=<hier-Token-einfuegen>' >> .env
    ```

**Warnungen:**

- Verwenden Sie **nicht** `CF_API_EMAIL` + `CF_API_KEY` (Global API Key). Dieser gewährt Vollzugriff auf den gesamten Cloudflare-Account. Sind beide Varianten gleichzeitig gesetzt, bevorzugt `lego` den Global API Key und das Token wird stillschweigend ignoriert.
- Es existiert **keine** Variable `CF_DNS_API_KEY`. Ein darunter hinterlegter Wert wird ohne Fehlermeldung verworfen.

#### 5.2 E-Mail-Adresse und Dashboard-Domain setzen

1. In `data/traefik/traefik.yml` die E-Mail-Adresse für Let's Encrypt eintragen:
    ```yaml
    certificatesResolvers:
      myresolver:
        acme:
          email: "deine@email.de"
    ```

2. In der Datei `.env` die Dashboard-Domain setzen:
    ```bash
    SERVICES_TRAEFIK_LABELS_TRAEFIK_HOST=HOST(`traefik.yourdomain.com`)
    ```

### 6. CrowdSec konfigurieren

1. CrowdSec initial starten und wieder stoppen (erzeugt Verzeichnisstruktur und Hub-Cache):
    ```bash
    cd /opt/containers/traefik-crowdsec-stack/
    docker compose up -d crowdsec && docker compose down
    ```

2. `acquis.yaml` wurde bereits aus dem Sample kopiert (Schritt 4) und deckt `auth.log`/`syslog` sowie Traefik-Access-Log ab. Zusätzliche Datenquellen bitte nicht hier, sondern in separaten Dateien unter `data/crowdsec/config/acquis.d/` ablegen (CrowdSec lädt das Verzeichnis automatisch).

3. AppSec-Konfiguration: Die für `data/crowdsec/config/acquis.d/appsec.yaml` benötigten Hub-Items werden bereits beim ersten CrowdSec-Start automatisch installiert (siehe `COLLECTIONS` und `APPSEC_CONFIGS` in `data/crowdsec/.env`). Beim Bearbeiten von `acquis.d/appsec.yaml` darauf achten, dass alle unter `appsec_configs:` referenzierten Namen entweder aus dem Hub installiert werden oder als lokale Datei existieren (z. B. `custom/hooks` → `data/crowdsec/config/appsec-configs/custom-hooks.yaml`).

4. In `data/crowdsec/config/appsec-configs/custom-hooks.yaml` die Platzhalter `<IP-1>`, `<IP-2>`, `<IP-3>` durch die IPs Ihrer vertrauenswürdigen internen Automatisierungs-Hosts ersetzen (z. B. Watchtower/shoutrrr-Absender, die an ntfy melden). Nicht benötigte Platzhalter entfernen; die gesamte Regel entfernen, wenn Sie keine solchen Ausnahmen brauchen.

5. Bouncer-Keys erzeugen. Traefik-Plugin- und Firewall-Bouncer bekommen je einen eigenen Key. Die Keys werden in die `.env` geschrieben und beim ersten Start des CrowdSec-Containers über die `BOUNCER_KEY_*`-Umgebungsvariablen automatisch als Bouncer registriert:
    ```bash
    BOUNCER_KEY_TRAEFIK=$(openssl rand -base64 48 | tr -dc 'A-Za-z0-9' | head -c 32)
    BOUNCER_KEY_FIREWALL=$(openssl rand -base64 48 | tr -dc 'A-Za-z0-9' | head -c 32)
    echo "BOUNCER_KEY_TRAEFIK=$BOUNCER_KEY_TRAEFIK" >> .env
    echo "BOUNCER_KEY_FIREWALL=$BOUNCER_KEY_FIREWALL" >> .env
    ```
    Die `tr -dc 'A-Za-z0-9'`-Filterung verhindert Sonderzeichen wie `+`, `/`, `=`, die in `.env`-Werten oder URLs Probleme machen. Zur Kontrolle nach dem nächsten Start:
    ```bash
    docker exec crowdsec cscli bouncers list
    ```

6. Den `BOUNCER_KEY_FIREWALL` separat notieren — er wird in Schritt 8 in der Firewall-Bouncer-Konfiguration außerhalb dieses Projekts benötigt.

### 7. Benutzer und Passwort für das Dashboard erstellen

Erstellen Sie einen Benutzer und ein Passwort für die HTTP-Basic-Authentifizierung im Traefik-Dashboard:

```bash
htpasswd -c /opt/containers/traefik-crowdsec-stack/data/traefik/.htpasswd <deinBenutzername>
```

### 8. Firewall Bouncer
1. Installieren Sie die Repositories von CrowdSec
    ```bash
    curl -s https://install.crowdsec.net | sudo sh
    ```
2. Installieren Sie den Service für Ihre Firewall

    **IPTables und UFW**
    ```bash
    sudo apt install crowdsec-firewall-bouncer-iptables
    ```
    **NFTables**
    ```bash
    sudo apt install crowdsec-firewall-bouncer-nftables
    ```

3.	Firewall-Konfiguration anpassen: Bearbeiten Sie die Datei `/etc/crowdsec/bouncers/crowdsec-firewall-bouncer.yaml`:
    ```
    api_url: http://<CROWDSEC-CONTAINER-IP>:8080/
    api_key: <BOUNCER_KEY_FIREWALL>
    ```
    Die `api_url` ist host-/netzwerkabhängig — sie muss vom Host aus (nicht aus einem Container heraus) auf den `crowdsec`-Container zeigen. Die konkrete IP hängt vom Docker-Netz ab; ermittelbar z. B. mit `docker inspect crowdsec | grep IPAddress`. Der `BOUNCER_KEY_FIREWALL` ist der in Schritt 6.5 registrierte Key.

4. Firewall neustarten
    ```
    systemctl enable crowdsec-firewall-bouncer
    systemctl restart crowdsec-firewall-bouncer
    ```


### 9. Firewall-Ports überprüfen

Stellen Sie sicher, dass die Firewall die Ports 80 (HTTP) und 443 (HTTPS) freigibt.

### 10. Domain überprüfen

Vergewissern Sie sich, dass die von Ihnen gewählte Domain korrekt auf die IP-Adresse des Servers verweist.

Hinweis zur DNS-01-Validierung: Für die reine Zertifikatsausstellung muss die Domain nicht zwingend öffentlich auf den Server zeigen, da die Validierung über einen TXT-Record im DNS erfolgt. Für den späteren HTTPS-Zugriff (Port 443) ist der A/AAAA-Record trotzdem erforderlich, sobald der Dienst öffentlich erreichbar sein soll.

### 11. Stack starten

Sobald alle Konfigurationen abgeschlossen sind, können Sie den Stack starten:

```bash
docker compose up -d
```

### 12. Zugriff auf das Traefik-Dashboard

Das Traefik-Dashboard sollte nun über die von Ihnen konfigurierte Domain erreichbar sein. Sie werden zur Eingabe des HTTP-Basic-Auth-Benutzernamens und Passworts aufgefordert.

https://traefik.yourdomain.com

---

## Abweichungen vom Upstream

Dieser Fork weicht in folgenden Punkten von `Psycho0verload/traefik-crowdsec-stack` (main) ab:

- **DNS-01 statt HTTP-01/TLS-ALPN** für die Let's-Encrypt-Validierung (Cloudflare als DNS-Provider). Erlaubt Wildcard-Zertifikate und funktioniert ohne öffentlich erreichbaren Port 80/443.
- **Nativer Traefik-Plugin-Bouncer** (`maxlerebourg/crowdsec-bouncer-traefik-plugin`) statt separatem ForwardAuth-Container. Der bisherige freifunkMUC-Fork des ForwardAuth-Bouncers wird nicht mehr gepflegt; das Plugin erlaubt zusätzlich die Anbindung von AppSec/WAF.
- **AppSec/OWASP-CRS in-band** aktiv, gesteuert über `data/crowdsec/config/acquis.d/appsec.yaml` mit lokalen False-Positive-Ausnahmen in `appsec-configs/custom-hooks.yaml`.
- **Zusammengeführte Middleware-Datei** `http.middlewares.default.yml` (statt getrennter Dateien für security-headers und gzip).
- **HTTP/3** bleibt aktiv (UDP 443 + `http3: {}`).

Details je Änderung siehe [`CHANGELOG.md`](CHANGELOG.md).

## Pro Host anzupassen

Beim Aufsetzen des Stacks auf einem neuen Host sind mindestens folgende Werte host-individuell:

| Was | Wo | Warum |
|---|---|---|
| Dashboard-Hostname | `.env` (`SERVICES_TRAEFIK_LABELS_TRAEFIK_HOST`) | Pro Host eigene Domain |
| Bouncer-Keys | `.env` (`BOUNCER_KEY_TRAEFIK`, `BOUNCER_KEY_FIREWALL`) | **Pro Host neu generieren, niemals kopieren.** Jeder Host bekommt eigene Credentials. |
| Cloudflare-Token | `.env` (`CF_DNS_API_TOKEN`) | Scope-abhängig; ein geteiltes Token bewusst wählen |
| E-Mail für Let's Encrypt | `data/traefik/traefik.yml` | Pro Betreiber |
| Firewall-Bouncer `api_url` | `/etc/crowdsec/bouncers/crowdsec-firewall-bouncer.yaml` | Adressiert den `crowdsec`-Container über das jeweilige Docker-Netz |
| AppSec-Custom-Hooks-IPs | `data/crowdsec/config/appsec-configs/custom-hooks.yaml` | Vertrauenswürdige interne Automatisierungs-Hosts |
| `forwardedHeadersCustomName` | `data/traefik/dynamic_conf/http.middlewares.traefik-bouncer.yml` | Standardwert `CF-Connecting-IP` setzt Cloudflare-Proxy voraus (orange cloud). Ohne Proxy leer lassen oder auf den tatsächlich verwendeten Header umstellen |
| ggf. Docker-Netzwerke | Compose-Dateien / Override | Wenn bereits andere Stacks Netz-Namen belegen |
| ggf. statische Backend-IPs | `data/traefik/dynamic_conf/` | Nur wenn Router auf feste IPs zeigen |

**Nicht zwischen Hosts kopieren:**

- `data/crowdsec/data/` (CrowdSec-Datenbank und Entscheidungshistorie)
- `data/crowdsec/config/*credentials*` (CAPI-Credentials)
- `data/traefik/certs/dns_letsencrypt.json` (host-spezifische ACME-Account-Keys)

## Host-/dienstspezifische Overrides

Der Stack ist bewusst schlank gehalten. Zusätzliche host- oder dienstspezifische Konfiguration (z. B. Log-Mounts für weitere Anwendungen, dienstspezifische Middleware-Chains) sollte **nicht** in die Standard-Compose-/Sample-Dateien einfließen, sondern in separate, optional eingebundene Dateien:

- **Zusätzliche Compose-Services / Volume-Mounts:** eigene `compose/<host>.<dienst>.yml` anlegen und im Haupt-`docker-compose.yml` per `include:` einbinden — oder mit `docker compose -f docker-compose.yml -f compose/override.yml up -d` starten.
- **Zusätzliche CrowdSec-Datenquellen:** eigene Datei unter `data/crowdsec/config/acquis.d/*.yaml`. CrowdSec lädt automatisch alle Dateien im Verzeichnis.
- **Dienstspezifische Traefik-Middleware-Chains:** eigene `data/traefik/dynamic_conf/http.middlewares.<name>.yml`. Der File-Provider lädt alle YAML-Dateien im Verzeichnis.

Die konkreten Inhalte solcher Overrides bleiben absichtlich außerhalb dieses Repositories, da sie auf einzelne Anwendungsfälle zugeschnitten sind.