# etcd

[TOC]

## Was ist etcd?

_etcd_ ist eine hochverfügbare verteilte key-value-Datenbank,
deren Schlüssel (key) hierarchisch sind.
Mit der Datenbank verbundene Anwendungen können sich benachrichtigen lassen,
wenn sich ein bestimmter Wert ändert.
Daher ist diese Datenbank besonders für Hochverfügbarkeitssysteme geeignet,
um zum Beispiel ein Failover auszuführen.

Dokumentation: [https://etcd.io/docs/](https://etcd.io/docs/)

## API-Version

_etcd_ nutzt eine REST-Schnittstelle, welche verschiedene Versionen hat (`2` und `3`).
Um die Version an der Kommandozeile auszuwählen, muss eine Umgebungsvariable gesetzt
werden. Zum Beispiel:

```commandline
export ETCDCTL_API=2 # oder
export ETCDCTL_API=3
```

Aktuell (Debian _Bullseye_) ist Version `2` Standard. Es muss daher keine
Umgebungsvariable gesetzt werden.

!!! warning "Vorsicht"
    Zu beachten ist, dass ein Schlüssel, der mit der v2-API erstellt wurde,
    nicht über die v3-API abgefragt werden kann.
    Das gilt auch umgekehrt.

Praktisch bedeudet das, dass das _etcd_-Cluster im Dual-Stack-Modus
betrieben wird. Die Daten unterschiedlicher API-Versionen "sehen"
sich gegenseitig nicht.

Einige Programme sind noch nicht kompatibel zur Version `3`.
Daher muss vorher geprüft werden, welche Programme welche API-Version
unterstützen.

## Standalone

Um ein _etcd_-Cluster aufzubauen, muss zunächst ein einzelner Endpoint
erstellt werden. Dazu wir _etcd_ mit

```commandline
apt install etcd
```

installiert. Mit `etcdctl member list` erhalten wir z.Bsp. folgende Ausgabe:

=== "API v2"
    ```
    8e9e05c52164694d: name=node-1 peerURLs=http://localhost:2380 clientURLs=http://localhost:2379 isLeader=true
    ```

    und mit `ETCDCTL_API=2 etcdctl cluster-health`

    ```
    member 8e9e05c52164694d is healthy: got healthy result from http://localhost:2379
    cluster is healthy
    ```

=== "API v3"
    ```
    8e9e05c52164694d, started, node-1, http://localhost:2380, http://localhost:2379
    ```

    und mit `ETCDCTL_API=3 etcdctl endpoint status -w table`
    
    ```
    +----------------+------------------+---------+---------+-----------+-----------+------------+
    |    ENDPOINT    |        ID        | VERSION | DB SIZE | IS LEADER | RAFT TERM | RAFT INDEX |
    +----------------+------------------+---------+---------+-----------+-----------+------------+
    | 127.0.0.1:2379 | 8e9e05c52164694d |  3.3.25 |   20 kB |      true |         2 |          4 |
    +----------------+------------------+---------+---------+-----------+-----------+------------+
    ```

Der Endpoint ist automatisch ein `Leader`, da er der einzige Endpoint ist.
Voreinstellung ist, dass er nur auf `localhost` läuft.

Damit der Endpoint auch im LAN erreichbar ist, halten wir den Endpoint an und
löschen alle Daten:

```commandline
systemctl stop etcd
rm -rf /var/lib/etcd/* 
```

Als Beispiel verwenden wir den Endpoint-Namen `node-1`. Die IP des Endpunktes soll
`192.168.0.1` sein.

```commandline
sudo -u etcd etcd \
    --name 'node-1' \
    --data-dir='/var/lib/etcd/default' \
	--listen-client-urls 'http://127.0.0.1:2379,http://192.168.0.1:2379' \
	--listen-peer-urls 'http://192.168.0.1:2380' \
	--advertise-client-urls 'http://192.168.0.1:2379' \
	--initial-advertise-peer-urls 'http://192.168.0.1:2380' \
	--initial-cluster 'node-1=http://192.168.0.1:2380' \
	--initial-cluster-token 'xr2phkpj96fT9uHQ' \
	--initial-cluster-state 'new' \
	--enable-v2
```

Damit startet _etcd_ im Vordergrund und kann mit ++control+"c"++ abgebrochen werden.
Anschliesend erstellen wir die Datei `/etc/default/etcd` mit folgendem Inhalt

```
ETCD_NAME="node-1"
ETCD_LISTEN_PEER_URLS="http://192.168.0.1:2380"
ETCD_LISTEN_CLIENT_URLS="http://127.0.0.1:2379,http://192.168.0.1:2379"
ETCD_ADVERTISE_CLIENT_URLS="http://192.168.0.1:2379"
DAEMON_ARGS="--enable-v2=true"
```
und starten _etcd_ mit `systemctl start etcd` neu.

## Endpoint hinzufügen

In diesem Beispiel fügen wir einen neuen Endpunkt mit dem Namen `node-2`
und der IP-Adresse `192.168.0.2` hinzu.

Wie beim Standalone installieren wir _etcd_ und löschen die Daten:

```commandline
apt install etcd
systemctl stop etcd
rm -rf /var/lib/etcd/* 
```
Anschliesend erstellen wir die Datei `/etc/default/etcd` mit folgendem Inhalt

```
ETCD_NAME="node-2"
ETCD_LISTEN_PEER_URLS="http://192.168.0.2:2380"
ETCD_LISTEN_CLIENT_URLS="http://127.0.0.1:2379,http://192.168.0.2:2379"
ETCD_ADVERTISE_CLIENT_URLS="http://192.168.0.2:2379"
DAEMON_ARGS="--enable-v2=true"
```

Zunächst muss der neue Endpoint beim _etcd_-Cluster registriert werden:

```commandline
ETCDCTL_API=3 etcdctl member add 'node-2' --peer-urls='http://192.168.0.2:2380'
```

Nun können wir den neuen Endpunkt das erste Mal starten:

```commandline
sudo -u etcd etcd \
    --name 'node-2' \
    --data-dir='/var/lib/etcd/default' \
	--listen-client-urls 'http://127.0.0.1:2379,http://192.168.0.2:2379' \
	--listen-peer-urls 'http://192.168.0.2:2380' \
	--advertise-client-urls 'http://192.168.0.2:2379' \
	--initial-advertise-peer-urls 'http://192.168.0.2:2380' \
	--initial-cluster 'node-1=http://192.168.0.1:2380,node-2=http://192.168.0.2:2380' \
	--initial-cluster-token 'xr2phkpj96fT9uHQ' \
	--initial-cluster-state 'existing' \
	--enable-v2
```

und brechen mit ++control+"c"++ den Prozess ab.
Danach starten wir den neuen Endpunkt mit `systemctl start etcd`.

Mit `etcdctl member list` sollten dann alle Mitglieder sichtbar sein.

=== "API v2"
    `ETCDCTL_API=2 etcdctl cluster-health` gibt zurück, ob das Cluster verfügbar ist.

=== "API v3"
    `ETCDCTL_API=3 etcdctl --endpoints=http://192.168.0.1:2379,http://192.168.0.2:2379 endpoint status -w table`
    gibt zurück, ob das Cluster verfügbar ist.

## Authentifizierung einrichten ##

Um Authentifizierung zu ermöglichen, muss zunächst der Superuser `root`
mit einem Passwort angelegt werden. Der Nutzer wird für beide API-Versionen
angelegt. Praktisch sind das verschiedene Nutzer.

```commandline
ETCDCTL_API=2 etcdctl user add root
ETCDCTL_API=3 etcdctl user add root
```

Anschliesend wird die Authentifizierung aktiviert. Dabei ist die
API-Version egal.

```commandline
etcdctl auth enable
```

Wir können dann prüfen, ob die Authentifizierung funktioniert:

=== "API v2"
    ```commandline
    ETCDCTL_API=2 etcdctl -u root user list
    ```

=== "API v3"
    ```commandline
    ETCDCTL_API=3 etcdctl --user=root user list
    ```

## Authorisierung

_etcd_ verwendet Nutzer und Rollen. Wir gehen hierbei davon aus, das
die [Authentifizierung](#authentifizierung-einrichten) bereits eingerichtet ist.

Beispiel, um einen Nutzer `m.mustermensch` anzulegen: 

=== "API v2"
    ```commandline
    ETCDCTL_API=2 etcdctl --total-timeout 10s -u root user add 'm.mustermensch'
    ```

=== "API v3"
    ```commandline
    ETCDCTL_API=3 etcdctl --user=root user add 'm.mustermensch'
    ```

Beispiel, um eine Rolle `my-role` anzulegen: 

=== "API v2"
    ```commandline
    ETCDCTL_API=2 etcdctl -u root role add 'my-role'
    ```

=== "API v3"
    ```commandline
    ETCDCTL_API=3 etcdctl --user=root role add 'my-role'
    ```

Anschließend können wir den Nutzer `m.mustermensch` der Rolle `my-role`
hinzufügen:

=== "API v2"
    ```commandline
    ETCDCTL_API=2 etcdctl -u root user grant --roles 'my-role' 'm.mustermensch'
    ```

=== "API v3"
    ```commandline
    ETCDCTL_API=3 etcdctl --user=root user grant-role 'm.mustermensch' 'my-role'
    ```

Beispiel, um der Rolle `my-role` lesende und schreibende Zugriffsrechte auf einen Pfad '/my-dir'
und seine Unterobjekte zu geben:

=== "API v2"
    ```commandline
    ETCDCTL_API=2 etcdctl -u root role grant --readwrite --path '/my-dir/*' 'my-role'
    ```

=== "API v3"
    ```commandline
    ETCDCTL_API=3 etcdctl --user=root role grant-permission --prefix=true 'my-role' readwrite '/my-dir/'
    ```
