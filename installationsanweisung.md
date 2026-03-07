# Bolt Card Hub — Standalone Installation (bestehender VPS)

Anleitung für die Installation auf einem VPS, auf dem bereits **Caddy** und **Phoenixd** laufen (z.B. zusammen mit LNbits oder Alby Hub).

## Voraussetzungen

- Caddy läuft bereits als Reverse Proxy
- Phoenixd läuft bereits (Port 9740)
- Docker ist installiert

## Schritt 1: Repository clonen

```bash
git clone -b stand-alone-hub https://github.com/AxelHamburch/hub.git boltcard-hub
cd boltcard-hub
```

## Schritt 2: Konfiguration erstellen

```bash
cp .env.standalone .env
nano .env
```

Folgende Werte eintragen:

```ini
HOST_DOMAIN=boltcard.deinedomain.com
PHOENIX_URL=http://127.0.0.1:9740
PHOENIX_PASSWORD=dein-phoenix-http-password
```

Das Phoenix-Passwort findest du so:

```bash
grep http-password ~/.phoenix/phoenix.conf
```

## Schritt 3: Image lokal bauen und Container starten

Da das offizielle Docker-Hub-Image die Standalone-Änderungen noch nicht enthält, muss das Image lokal gebaut werden:

```bash
sudo docker compose -f docker-compose.standalone.yml build
sudo docker compose -f docker-compose.standalone.yml up -d
```

Prüfe, ob er läuft:

```bash
sudo docker logs card
curl -s -o /dev/null -w "%{http_code}" http://127.0.0.1:8000/
# Sollte 200 zurückgeben
```

## Schritt 4: Netzwerk — Zugriff auf Phoenixd

Der Card-Container muss `http://127.0.0.1:9740` (dein Phoenixd) erreichen können. Am einfachsten geht das mit Host-Netzwerk-Modus.

Bearbeite `docker-compose.standalone.yml` und ersetze den `ports:`-Block durch `network_mode`:

```yaml
services:
  card:
    network_mode: host
    # ports: entfernen — im Host-Modus nicht nötig
```

Der Container lauscht dann direkt auf Port 8000 und kann Phoenixd auf 127.0.0.1:9740 erreichen.

**Alternative** (ohne Host-Modus):

```yaml
services:
  card:
    extra_hosts:
      - "host.docker.internal:host-gateway"
```

Dann in `.env`:

```ini
PHOENIX_URL=http://host.docker.internal:9740
```

## Schritt 5: Caddy konfigurieren

Füge in deine bestehende Caddyfile einen Block für die Bolt Card Hub hinzu:

```caddy
boltcard.deinedomain.com {
    # CORS Headers für Lightning Wallets
    header {
        Access-Control-Allow-Origin *
        Access-Control-Allow-Methods "GET, POST, OPTIONS"
        Access-Control-Allow-Headers "Authorization, Content-Type"
    }

    # Rate Limiting für Auth-Endpunkte (empfohlen)
    @auth_paths {
        path /admin/login/ /admin/api/auth/login /auth
    }
    handle @auth_paths {
        rate_limit {
            zone auth_zone {
                key {remote_host}
                events 10
                window 1m
            }
        }
        reverse_proxy 127.0.0.1:8000
    }

    # Rate Limiting für API-Endpunkte (empfohlen)
    @api_paths {
        path /create /payinvoice /addinvoice /ln /cb /.well-known/lnurlp/*
    }
    handle @api_paths {
        rate_limit {
            zone api_zone {
                key {remote_host}
                events 30
                window 1m
            }
        }
        reverse_proxy 127.0.0.1:8000
    }

    handle {
        encode zstd
        reverse_proxy 127.0.0.1:8000
    }
}
```

> **Hinweis:** Die `rate_limit`-Direktive erfordert das Caddy-Plugin `caddy-ratelimit`. Falls dein Caddy das nicht hat, entferne die `rate_limit`-Blöcke und verwende einfach `reverse_proxy 127.0.0.1:8000` direkt.

Dann Caddy neu laden:

```bash
sudo systemctl reload caddy
```

## Schritt 6: Testen

1. Öffne `https://boltcard.deinedomain.com/admin` im Browser
2. Setze ein Admin-Passwort
3. Im Dashboard solltest du den Phoenix-Saldo sehen (denselben wie in LNbits/Alby Hub)

## Updates

```bash
cd ~/boltcard-hub
git pull
sudo docker compose -f docker-compose.standalone.yml build
sudo docker compose -f docker-compose.standalone.yml up -d
```

## Hinweise

- **Geteilter Saldo:** Da du Phoenixd mit LNbits/Alby teilst, verwenden alle Apps denselben Lightning-Saldo. Zahlungen über Bolt Cards reduzieren dasselbe Guthaben.
- **Rückwärtskompatibel:** Die Änderungen am Code sind rückwärtskompatibel. Ohne die Env-Variablen `PHOENIX_URL` und `PHOENIX_PASSWORD` funktioniert das Original-Docker-Setup wie bisher.
- **Datenbank:** Die SQLite-Datenbank liegt im Docker-Volume `card_data` unter `/card_data/cards.db`. Für Backups: `docker cp card:/card_data/cards.db ./backup.db`
