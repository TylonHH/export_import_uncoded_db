# PostgreSQL Datenbank-Umzug

Diese Anleitung beschreibt, wie eine PostgreSQL-Datenbank von einem Server (z. B. [Netcup](https://www.netcup.com/de/?ref=280996) oder Railyway) zu einem neuen Ziel (z. B. [Netcup](https://www.netcup.com/de/?ref=280996)) migriert wird.  
Die Verbindung erfolgt sicher über SSH-Tunneling, ohne Ports nach außen zu öffnen. Hier am Beispiel des Trading Bot von [uncoded](https://t.me/unCoded_bot?start=ref_1203406052).

---

## Voraussetzungen

Halte Folgendes bereit, bevor du beginnst:

- **Zugangsdaten zum Quell-Server**
  - IP-Adresse (z. B. aus dem Netcup-Interface)
  - SSH-Zugang (z. B. `root`-Benutzer und Passwort)
- **Docker-Zugriff** auf dem Quell-Server  
  (z. B. mit `docker ps`, `docker exec`)
- **Zugangsdaten zum Ziel-System**
  - Host, Port, Benutzer, Passwort
- **Software lokal installiert**
  - [PuTTY](https://www.putty.org/) oder PowerShell (für SSH)
  - [pgAdmin](https://www.pgadmin.org/) (für Export und Import)
- **Root- oder Admin-Passwort** des Servers
- Optional: Admin-Zugang zu CapRover (falls Apps pausiert werden müssen)

---

## 1. Interne IP des Datenbankcontainers ermitteln

1. Per SSH auf den Server verbinden:
   ```bash
   ssh root@<server-ip>

2. Container-Name herausfinden:

   ```bash
   docker ps | grep postgres
   ```
3. In den Container wechseln:

   ```bash
   docker exec -it <containername> bash
   ```
4. Interne IP anzeigen:

   ```bash
   hostname -I
   ```

   Beispielausgabe: `172.18.0.5`
   → Die **zweite IP rechts** ist die relevante.

---

## 2. SSH-Tunnel zur internen Datenbank aufbauen

In PowerShell auf deinem lokalen Rechner:

```bash
ssh -L 5433:172.18.0.5:5432 root@<SERVER_IP> -v
```

* `5433` = lokaler Port (z. B. 5434, falls 5433 schon belegt)
* `172.18.0.5` = interne IP des Containers
* `5432` = Standardport von PostgreSQL
* Verbindung offen lassen (mit **STRG + D** kann man später trennen)

Jetzt läuft der Datenverkehr lokal über Port 5433 sicher zum Container.

---

## 3. Verbindung in pgAdmin einrichten

### Quell-Server ([Netcup](https://www.netcup.com/de/?ref=280996) oder CapRover)

* **Host:** `localhost`
* **Port:** `5433`
* **Maintenance DB:** `postgres`
* **Username:** `postgres`

### Ziel-Server (Railway o. Ä.)

* Verbindungsdaten laut Screenshot oder Anbieterangabe
<img width="1493" height="396" alt="grafik" src="https://github.com/user-attachments/assets/12b37f84-364c-413d-9213-13c30cd309bd" />
Daten aus Public Network ermitteln.

---

## 4. Vorbereitungen in CapRover

* `TOS` sollte **auf `false`** stehen.
* Apps wie **Trading** und **Telegram** sollten auf **Instanz 0** gesetzt sein.

---

## 5. Datenbank exportieren (Backup)

1. In pgAdmin rechtsklick auf `uncoded_db` → **Backup**
2. **General**

   * Name: `backup.sql`
   * Format: `Plain`
3. **Data Options**

   * Alle **Sections aktivieren**
4. **Query Options**

   * **INSERT** aktivieren
   * **CREATE DATABASE** aktivieren
5. Backup starten und speichern

---

## 6. Alte Datenbank am Ziel löschen und neu anlegen

1. In pgAdmin zweite Verbindung (Ziel-DB) öffnen
2. Rechtsklick auf `postgres` → **Query Tool**
3. Aktive Verbindungen trennen:

   ```sql
   SELECT pg_terminate_backend(pid) FROM pg_stat_activity WHERE datname = 'uncoded_db';
   ```
4. Datenbank löschen:

   ```sql
   DROP DATABASE uncoded_db;
   ```
5. Neu anlegen:

   ```sql
   CREATE DATABASE uncoded_db WITH TEMPLATE = template0 ENCODING = 'UTF8' LOCALE_PROVIDER = libc LOCALE = 'en_US.utf8'
   ```

---

## 7. Backup importieren

1. In der Übersicht auf die neue `uncoded_db` rechtsklicken → **Query Tool**
2. Die zuvor gesicherte SQL-Datei öffnen und einfügen
3. Folgende Zeilen ggf. entfernen oder mit `--` auskommentieren:

   ```sql
   SET transaction_timeout = 0;      -- (kommt oft doppelt vor)
   \connect uncoded_db
   CREATE DATABASE uncoded_db WITH TEMPLATE = template0 ENCODING = 'UTF8' LOCALE_PROVIDER = libc LOCALE = 'en_US.utf8';
   ```
4. Ausführen und warten, bis alle Tabellen erstellt sind

---

## 8. Prüfung und Abschluss

* In pgAdmin unter `Schemas > Tables` prüfen, ob Tabellen vorhanden sind
  (z. B. `activeorders`)
* Eine Tabelle öffnen, um sicherzustellen, dass Daten geladen wurden
* Danach Apps auf CapRover wieder aktivieren

---

## 9. Optional: Verbindung trennen

Wenn alles geprüft ist, PowerShell-Verbindung mit **STRG + D** beenden.
Die lokale Portweiterleitung wird dadurch geschlossen.

---

## Unterstützung

Du möchtest dich für diese Anleitung bedanken?

<a href="https://www.buymeacoffee.com/WarsoWerk" target="_blank"><img src="https://cdn.buymeacoffee.com/buttons/v2/default-yellow.png" alt="Buy Me A Coffee" style="height: 60px !important;width: 217px !important;" ></a>

<sup>Die mit Sternchen (*) gekennzeichneten Links sind sogenannte Affiliate-Links</sup>

## Weitere Projekte

Weitere Repositories findest du hier: [https://github.com/TylonHH](https://github.com/TylonHH)
