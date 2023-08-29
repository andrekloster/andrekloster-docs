# MongoDB

[TOC]


## Einleitung
[MongoDB](https://www.mongodb.com/docs/manual/) ist eine universelle, dokumentorientierte, verteilte Datenbank. Im Folgenden wird die Einrichtung eines
5-Nodes Primary-Secondary-Hidden-Arbiter Clusters beschrieben. Dieses Cluster-System ist redundant und ermöglicht eine Konsistenz der Daten.

### Installation
```bash
sudo apt update
sudo apt-get install -y gnupg curl
curl -fsSL https://pgp.mongodb.com/server-7.0.asc | \
   sudo gpg -o /etc/apt/trusted.gpg.d/mongodb-server-7.0.gpg \
   --dearmor
echo "deb http://repo.mongodb.org/apt/debian bullseye/mongodb-org/7.0 main" | sudo tee /etc/apt/sources.list.d/mongodb-org-7.0.list
sudo apt-get install -y mongodb-org mongodb-mongosh
```

### Einrichtung der Konfiguration
Um die Datenbank automatisch mit bestimmten Parametern starten zu können, muss Folgendes unter `/etc/default/mongod` gesetzt sein:

```none
DAEMON_OPTS="--auth"
CONF=/etc/mongod/mongod.conf
```

Erstelle eine custom systemd-unit, um die `/etc/default/mongod` einzubinden:

```commandline
vim /etc/systemd/system/mongod.service
```

```ini
[Unit]
Description=MongoDB Database Server
Documentation=https://docs.mongodb.org/manual
After=network.target

[Service]
User=mongodb
Group=mongodb
EnvironmentFile=-/etc/default/mongod
Environment=CONF=/etc/mongod.conf
ExecStart=/usr/bin/mongod --config ${CONF} $DAEMON_OPTS
PIDFile=/var/run/mongodb/mongod.pid
# file size
LimitFSIZE=infinity
# cpu time
LimitCPU=infinity
# virtual memory size
LimitAS=infinity
# open files
LimitNOFILE=64000
# processes/threads
LimitNPROC=64000
# locked memory
LimitMEMLOCK=infinity
# total threads (user+kernel)
TasksMax=infinity
TasksAccounting=false

# Recommended limits for for mongod as specified in
# http://docs.mongodb.org/manual/reference/ulimit/#recommended-settings

[Install]
WantedBy=multi-user.target
```

```commandline
systemctl daemon-relaod
systemctl enable mongod
```

```shell
mkdir /etc/mongod
mv /etc/mongodb.conf /etc/mongod/
cd /etc/mongod
openssl rand -base64 756 > key
chmod 0400 key
chown mongodb key
```

Folgende Einträge müssen in der `mongod.conf` enthalten sein:

```yaml
# Where and how to store data.
storage:
  dbPath: /var/lib/mongodb
  journal:
    enabled: true
  engine: wiredTiger
  wiredTiger:
    engineConfig:
      cacheSizeGB: 8 # change value for more memory cache

# where to write logging data.
systemLog:
  destination: file
  logAppend: true
  logRotate: reopen
  path: /var/log/mongodb/mongod.log

# network interfaces
net:
  ipv6: true
  port: 27017
  bindIp: 127.0.0.1,::1,<LAN-IPs>

# how the process runs
processManagement:
  timeZoneInfo: /usr/share/zoneinfo

security:
  keyFile: /etc/mongod/key

replication:
  replSetName: "set-1"
```

```shell
systemctl start mongod
```

### Initialisierung des Replica-Set Clusters
Ein Replica-Set ist ein hochverfügbares Cluster, das mindestens 3 Nodes benötigt. Eins der Nodes ist der Primary.
Die anderen Nodes sind Read-Only Secondary.
Wir verbinden uns mit dem Primary und initialisieren den `root` Benutzer inkl. des Clusters.

```
# Wir verbinden uns auf dem Primary via CLI mit der MongoDB
mongosh
```

```js
use admin
rs.initiate( { _id : "set-1", members: [ { _id: 0, host: "db-mongodb-1.domain-server.lan:27017" }, ] } )
```

```js
db.createUser( { user : "root", pwd : "geheimes_passwort", roles : [ "readWriteAnyDatabase", "userAdminAnyDatabase", "dbAdminAnyDatabase", "clusterAdmin" ] } )
```

Um weitere Nodes zum Cluster hinzufügen, führen wir auf dem Primary Node folgende Befehle aus.

```
mongosh -u root -p --authenticationDatabase admin
```

```js
use admin
rs.add( { host: "db-mongodb-2.domain-server.lan:27017", priority: 1, votes: 1 } )
rs.add( { host: "db-mongodb-3.domain-server.lan:27017", priority: 1, votes: 1 } )
```

### MongoDB-Arbiter hinzufügen
Es besteht die Möglichkeit einen sogenannten Arbiter Node zum Cluster hinzuzufügen. Der Arbiter besitzt keine Daten
und dient ausschließlich als Schiedsrichter, um eine Zweidrittelmehrheit sicherzustellen.
Um einen Arbiter zum Replika Set hinzuzufügen, führen wir auf dem Primary folgende Befehle aus

```
mongosh -u root -p --authenticationDatabase admin
```

```js
use admin
rs.addArb("db-mongodb-arbiter.domain-server.lan:27017")
```

### MongoDB-Hidden-Node hinzufügen
Ein Hidden Node ist ein versteckter Knoten im Cluster. Diese Art von Server dient als Hot-Standby oder als Backupserver.
Die Clients können sich nicht von außen mit dem Hidden Node verbinden. Des Weiteren hat der Hidden Noden ebenfalls die
Möglichkeit für das Quorum abzustimmen.
Um einen Hidden Node zum Replika Set hinzuzufügen, führen wir folgende Befehle auf dem Primary aus.

```
mongosh -u root -p --authenticationDatabase admin
```

```js
use admin
rs.add( { host: "db-mongodb-4.domain-server.lan:27017", priority: 0, votes: 1, "hidden": true } )
```

Replika-Konfiguration und Knoten-Status anzeigen:

+ `rs.conf()`
+ `rs.status()`

### Weitere Datenbanken inkl. Benutzer anlegen

```
mongosh -u root -p --authenticationDatabase admin
```

```js
use new_db_name
db.createUser( { user : "new_user_name", pwd: "geheimes_passwort", roles : [ "readWrite" ] } )
```

### Umschaltung des Primary-Knotens

Zunächst verbinden wir uns beim aktuellen Primary mit der MongoDB.
```
mongosh -u root -p --authenticationDatabase admin
```

Zum Umschalten des Primary Knoten, muss sichergestellt sein, dass jeder Knoten berechtigt ist abzustimmen/primary werden:
Um einen anderen Node zum Primary zu erklären, müssen wir sicherstellen, dass alle Mitglieder des ein Stimmrecht für
das Quorum haben.

```js
cfg = rs.conf()
cfg.members[0].votes = 1
cfg.members[1].votes = 1
cfg.members[2].votes = 1
rs.reconfig(cfg)
```

Wenn das erfüllt ist, gibt es zwei mögliche Optionen den Primary umzuschalten:

- **Stepdown**
```js
rs.stepDown()
```
Diese Methode sollte bevorzugt werden. Hierbei sendet der Primary an alle Secondary ein Signal, dass der Primary zurücktritt.
Anschließend wählen die Secondaries automatisch einen neuen Primary. 

- **Manuelles Umschalten**
```js
cfg = rs.conf()
cfg.members[0].priority = 1
cfg.members[1].priority = 2
cfg.members[2].priority = 1
rs.reconfig(cfg)
```
Hierbei wird die Priorität der einzelnen Knoten rekalibriert. Der Knoten mit der höchsten Priorität wird zum neuen Primary.
Achtung: Diese Methode sollte mit bedacht verwendet werden, da man künstlich in den Quorum-Prozess eingreift.

### Primary-Knoten forcieren ###

!!! warning
    Es kann der Fall auftreten, dass zu viele Knoten ausfallen und kein Primary bestimmt werden kann.
    In diesem Fall muss das Replika Set neu konfiguriert werden.

Bei der Neukonfiguration eines Cluster, wählen wir nur die __gesunde__ Knoten aus.

```
mongosh -u root -p --authenticationDatabase admin
```

```js
cfg = rs.conf()
printjson(cfg)
cfg.members = [cfg.members[0] , cfg.members[2]]
```

Neue Konfiguration auf das Cluster forciert anwenden

```js
rs.reconfig(cfg, {force : true})
```

### Backup erstellen und einspielen ###

**Backup erstellen**

Zunächst müssen wir sicherstellen, dass ein unprivilegierter Backup-Benutzer erstellt ist

```
use admin
db.createUser( { 
    user : "backup",
    pwd : "GEHEIMES_PASSWORT", 
    roles : [ "backup", "restore" ] 
} )
```

Anschließend können wir mit folgendem Befehl ein Backup erstellen.

```
mongodump \
    --host="set-1/db-mongodb-1.domain-server.lan:27017,db-mongodb-2.domain-server.lan:27017,db-mongodb-3.domain-server.lan:27017" \
    --username="backup" \
    --password="GEHEIMES_PASSWORT" \
    --authenticationDatabase="admin" \
    --db="MY-DATABASE" \
    --forceTableScan \
    --out="MONGODB_DB_DUMP_DIR"
```

**Backup wiederherstellen**

```
mongorestore \
    --host="set-1/db-mongodb-1.domain-server.lan:27017,db-mongodb-2.domain-server.lan:27017,db-mongodb-3.domain-server.lan:27017" \
    --username="backup" \
    --password="GEHEIMES_PASSWORT" \
    --authenticationDatabase="admin" \
    --gzip \
    "MONGODB_DB_DUMP_DIR"
```
