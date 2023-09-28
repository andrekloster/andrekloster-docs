# Icinga

[TOC]

## Icinga DB Architektur
<figure markdown>
![Architektur](/monitoring/images/icinga/architecture.png){ width="1000", loading=lazy }
</figure>

Eine Icinga2 Installation mit der IcingaDB setzt sich auch mehrere Faktoren zusammen, die alle aufeinander aufbauen.

- **Icinga2**: Backend, das alle Konfigurationen für die Host- und Serviceüberwachungen bereitstellt.
- **Icinga Web**: Frontend, das für die grafische Darstellung im Browser zuständig ist.
- **Datenbank**: Eine SQL Dantebank (MySQL, PostgreSQL), die den Zustand aller Icingaüberwachungen speichert.
- **IcingaDB**: Einen Middleware, die das Backend mit der Datenbank verbindet.
- **Redis**: Ein Cache Server, der die Datenbankabfragen im RAM-Speicher performant aufbewahrt.

## Installation

### Icinga2
Zunächst installieren wir das Backend Paket `icinga2`. Dieses Paket beinhaltet alle Grundkonfigurationen für die Host- und 
Serviceüberwachung.

```shell
apt-get install -y icinga2
icinga2 daemon --validate
```

### API
Als Nächstes initialisieren wir die Icinga API:

```shell
icinga2 api setup
systemctl restart icinga2
```

Mit Hilfe der API kommunizieren alle Komponenten der Architektur miteinander.
Nachdem die API initialisiert wurde, öffnen wir die Datei `/etc/icinga2/conf.d/api-users.conf` und notieren uns
den automatisch angelegten Benutzer und das Passwort. Ggf. passen wir noch die Berechtigungen des API Benutzers an.
Eine API Benutzerkonfiguration könnte wie folgt aussehen:

```
object ApiUser "icingadb-web" {
    password = "API_PASSWORT"
    permissions = [ "actions/*", "objects/modify/*", "objects/query/*", "status/query" ]
}
```

### Datenbank
Für die Datenbankkonfiguration benötigen wir das Paket `icingadb` und in unserem Fall die SQL Datenbank `postgresql`.

```shell
apt-get install -y icingadb postgresql
```

Die IcingaDB dient als Middleware zwischen dem Icinga Backend und der PostgreSQL Datenbank.
Für die Einrichtung der PostgreSQL erstellen wir die notwendigen Zugangsdaten und die Datenbank.

```shell
su -l postgres
createuser -P icingadb # Hier wird nach einem Passwort gefragt. Passwort wird notiert!
createdb -E UTF8 -T template0 -O icingadb icingadb
psql icingadb <<<'CREATE EXTENSION IF NOT EXISTS citext;'
```

Anschließend ergänzen wir die `/etc/postgresql/<PG_VERSION>/main/pg_hba.conf` um folgende Werte, um den Zugang für
die Datenbank `icingadb` zu ermöglichen und starten die Datenbank darauf hin neu.

```
local   all      icingadb                            md5
host    all      icingadb      127.0.0.1/32          md5
host    all      icingadb      ::1/128               md5
```

```shell
systemctl restart postgresql
```

Jetzt wechseln wir zum UNIX-Benutzer `icingadb` und befüllen die neue Datenbank mit initialen Daten.

```shell
su -l icingadb -s /bin/bash
psql -U icingadb icingadb < /usr/share/icingadb/schema/pgsql/schema.sql
```

Die vorhin notierten Zugangsdaten für die Datenbank `icingadb` tragen wir in der Konfiguration `/etc/icingadb/config.yml` ein.

```yaml
[...]
database:
  type: pgsql
  host: localhost
  database: icingadb
  user: icingadb
  password: <PG_PASSWORT>
[...]
```

Jetzt aktivieren wir das icingadb Feature und starten das Icinga2 Backend neu.

```shell
icinga2 feature enable icingadb
systemctl restart icingadb
systemctl restart icinga2
```

### IcingaDB Web
Für die grafische Darstellung im Browser, benötigen wir das Paket `icingadb-web`.

```shell
apt-get install -y icingadb-web
```

### IcingaDB Redis
Der Redis Server wird dazu verwendet, um Datenbankabfragen zu cachen und dadurch die Performance zu erhöhen.
Um Redis zu installieren, führen wir folgende Befehle aus:

```shell
apt-get install -y icingadb-redis
systemctl enable icingadb-redis
systemctl enable icingadb-redis-server
```

Anschließend tragen wir in der Datei `/etc/icinga2/features-available/icingadb.conf` die Benutzerdaten für die `icingadb`
ein.

```
object IcingaDB "icingadb" {
  host = "127.0.0.1"
  port = 6380
  password = "PG_PASSWORT"
}
```

Jetzt starten wir den Redis Server

```shell
systemctl start icingadb-redis icingadb-redis-server
```

### Optional: Apache vHost Konfiguration generieren
Falls notwendig, kann mit folgendem Befehl eine minimale Apache vHost Konfiguration automatisch generiert werden:

```shell
icingacli setup config webserver apache --document-root /usr/share/icingaweb2/public
```

## Icinga Konfigurations-Wizzard
Nun erfolgt die Konfiguration aller Icinga Komponenten mit Hilfe eines Wizzards im Webbrowser. Nachdem diese Konfiguration
erfolgreich abgeschlossen ist, ist Icinga vollständig einsatzbereit.

### Init Setup
Um das Setup zu starten, müssen wir zunächst das `setup` Module aktivieren.

```shell
icingacli module enable setup
```

Anschließend öffnen wir den Webbrowser und navigieren zur folgenden URI:

```
http://<ICINGA_IP>/icingaweb2/setup
```

Dort sollten wir folgenden Startbildschirm sehen.

<figure markdown>
![Init Setup](/monitoring/images/icinga/init_setup.png){ width="1000", loading=lazy }
</figure>

Nun befolgen wir die Anweisungen vom Setup. Für den initialen Setup-Token sind folgende Schritte auszuführen

```shell
addgroup --system icingaweb2;
usermod -a -G icingaweb2 www-data;
icingacli setup config directory --group icingaweb2;
icingacli setup token create;
```

Den Einrichtungstoken tragen wir nun beim Setup im Webbrowser ein.

### Module
Wir benötigen für unser Setup nur das `icingadb` Modul

<figure markdown>
![Module](/monitoring/images/icinga/module.png){ width="1000", loading=lazy }
</figure>

### Debian Pakete
!!! info
    Falls Pakete nachträglich nachinstalliert wurden, ist es empfehlenswert den `apache2` und `icinga2` Daemon neuzustarten.

Hier müssen wir sicherstellen, dass alle notwendigen Debianpakete für das Setup installiert sind.

<figure markdown>
![Pakete](/monitoring/images/icinga/pakete.png){ width="1000", loading=lazy }
</figure>

### Icinga Web Authentifizierung
<figure markdown>
![Auth](/monitoring/images/icinga/auth.png){ width="1000", loading=lazy }
</figure>

### Icinga Web Datenbank-Ressource
Hier verwenden wir die Logindaten aus dem Abschnitt [Datenbank](#datenbank).

<figure markdown>
![IcingaWebDB](/monitoring/images/icinga/icingaweb_db.png){ width="1000", loading=lazy }
</figure>

### Authentifizierungs-Backend
<figure markdown>
![Backend Name](/monitoring/images/icinga/backend_name.png){ width="1000", loading=lazy }
</figure>

### Administrator Account anlegen
Mit diesem Account loggen wir uns später in der WebUI an. Von daher muss ein sicheres Passwort ausgewählt und notiert werden.

<figure markdown>
![Admin Account](/monitoring/images/icinga/admin_account.png){ width="1000", loading=lazy }
</figure>

### Logging Konfigurationen
<figure markdown>
![Log Config](/monitoring/images/icinga/log_config.png){ width="1000", loading=lazy }
</figure>

### Einrichtung von Icinga DB Web 
<figure markdown>
![Start IcingaDB](/monitoring/images/icinga/start_icingadb.png){ width="1000", loading=lazy }
</figure>

### Icinga DB Ressource konfigurieren
Hier verwenden wir die Logindaten aus dem Abschnitt [Datenbank](#datenbank). Zusätzlich müssen wir darauf achten,
dass im Feld `Zeichensatz` der gleiche Typ hinterlegt ist, wie bei der Initialisierung der Datenbank.

<figure markdown>
![IcingaDB](/monitoring/images/icinga/icingadb.png){ width="1000", loading=lazy }
</figure>

### Icinga DB Redis konfigurieren
Da wir ein lokalen Redis Server betreiben, bleibt das Feld `Redis Passwort` leer. Ebenso betreiben wir in unserem Setup
keinen sekundären Icinga-Master.

<figure markdown>
![Redis](/monitoring/images/icinga/redis.png){ width="1000", loading=lazy }
</figure>

### Icinga API konfigurieren
Hier verwenden wir die Logindaten aus dem Abschnitt [API](#api).

<figure markdown>
![API](/monitoring/images/icinga/api.png){ width="1000", loading=lazy }
</figure>

### Konfigurationen überprüfen
Sollten all unsere Angaben korrekt sein, ist das Setup erfolgreich abgeschlossen. Nun können wir uns mit dem Benutzer
einloggen, den wir im Abschnitt [Administrator Account anlegen](#administrator-account-anlegen) erstellt haben.

<figure markdown>
![Check](/monitoring/images/icinga/check.png){ width="1000", loading=lazy }
</figure>

<figure markdown>
![Check](/monitoring/images/icinga/success.png){ width="1000", loading=lazy }
</figure>
