# Bolt Card Hub — Standalone Installation (bestehender VPS)

Anleitung für die Installation auf einem VPS, auf dem bereits **Caddy** und **Phoenixd** laufen (z.B. zusammen mit LNbits oder Alby Hub).

## Voraussetzungen

- Caddy läuft bereits als Reverse Proxy
- Phoenixd läuft bereits (Port 9740)
- Docker ist installiert
- Port 8000 ist frei (prüfen mit `sudo ss -tlnp | grep 8000`)

Falls Port 8000 belegt ist (z.B. durch icecast2):

```bash
sudo systemctl stop icecast2
sudo apt remove --purge icecast2 -y
sudo apt autoremove -y
```

## Schritt 1: Repository clonen

```bash
cd ~
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

## Schritt 3: Image bauen und Container starten

```bash
sudo docker compose -f docker-compose.standalone.yml build
sudo docker compose -f docker-compose.standalone.yml up -d
```

Der erste Build dauert ein paar Minuten (Go + Node werden kompiliert).

## Schritt 4: Prüfen ob alles läuft

```bash
sudo docker logs card
```

Du solltest sehen:

```
INFO using phoenix password from PHOENIX_PASSWORD env var
INFO card service started
```

Und **keine** `connection refused`-Fehler. Zusätzlich testen:

```bash
curl -s -o /dev/null -w "%{http_code}" http://127.0.0.1:8000/
# Sollte 200 zurückgeben
```

## Schritt 5: Caddy konfigurieren

Füge in deine bestehende Caddyfile einen Block für die Bolt Card Hub hinzu:

```bash
sudo nano /etc/caddy/Caddyfile
```

```caddy
boltcard.deinedomain.com {
    header {
        Access-Control-Allow-Origin *
        Access-Control-Allow-Methods "GET, POST, OPTIONS"
        Access-Control-Allow-Headers "Authorization, Content-Type"
    }

    reverse_proxy 127.0.0.1:8000
}
```

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

## Komplett deinstallieren

```bash
cd ~/boltcard-hub
sudo docker compose -f docker-compose.standalone.yml down -v
sudo docker image rm boltcard/card:standalone
cd ~
rm -rf boltcard-hub
```

## Hinweise

- **Host-Netzwerk-Modus:** Der Container nutzt `network_mode: host` — er teilt sich das Netzwerk mit dem Host. Dadurch kann er direkt auf Phoenixd (127.0.0.1:9740) zugreifen und lauscht auf Port 8000.
- **Geteilter Saldo:** Da du Phoenixd mit LNbits/Alby teilst, verwenden alle Apps denselben Lightning-Saldo. Zahlungen über Bolt Cards reduzieren dasselbe Guthaben.
- **Rückwärtskompatibel:** Ohne die Env-Variablen `PHOENIX_URL` und `PHOENIX_PASSWORD` funktioniert das Original-Docker-Setup wie bisher.
- **Datenbank:** Die SQLite-Datenbank liegt im Docker-Volume `card_data` unter `/card_data/cards.db`. Für Backups: `docker cp card:/card_data/cards.db ./backup.db`
