# Bolt Card Hub — Update-Anleitung (Standalone)

## Wichtig: Update-Button im Admin-Panel ignorieren

Der **"Update to X.X.X"**-Button auf der About-Seite funktioniert im Standalone-Setup **nicht**.
Er versucht das offizielle Docker-Hub-Image zu ziehen, das die Standalone-Änderungen nicht enthält.

Updates werden stattdessen über Git + lokalen Build gemacht.

## Update durchführen

### 1. Auf dem Windows-Rechner: Branch aktualisieren

```bash
cd d:\VSCode\hub
git fetch origin main
git rebase origin/main
git push origin stand-alone-hub --force-with-lease
```

### 2. Auf dem VPS: Neuen Stand ziehen und bauen

```bash
cd ~/boltcard-hub
git fetch origin
git reset --hard origin/stand-alone-hub
sudo docker compose -f docker-compose.standalone.yml build
sudo docker compose -f docker-compose.standalone.yml up -d
```

### 3. Prüfen

```bash
sudo docker logs card
```

Du solltest die neue Versionsnummer in der ersten Zeile sehen.

## Hinweise

- `git reset --hard` ist nötig, weil der Branch rebaset wird (neue Commit-Hashes). Deine `.env` bleibt davon unberührt.
- Der Build dauert beim ersten Mal einige Minuten (Go + Node), danach nutzt Docker den Cache und es geht schneller.
- Deine Datenbank (`card_data` Volume) wird bei Updates **nicht** gelöscht.
