# Einen Puppet Server erstellen

[TOC]

## Ziel
Um eine Infrastruktur mit Puppet aufzubauen, benötigen wir in erster Linie einen Puppet Server.
Dieser Server ist das zentrale Objekt der Infrastruktur, bei dem die Clients in regelmäßigen Abständen
nach Änderungen abfragen.

## Voraussetzungen
- Linux Server (z.Bsp. Debian Bookworm)
- Mindestens 1GB frei verfügbaren RAM

## Vorbereitungen
```shell
# Installiere notwendige Update und setze den Hostnamenyy
sudo apt update && sudo apt upgrade -y
sudo hostnamectl set-hostname <PUPPET-SERVER-FQDN> # Anpassen!
```

## Installation und Konfiguration des Puppet Servers

```shell
sudo apt install -y puppetserver
```

Nun öffnen wir die Puppet Konfigurationsdatei und legen grundsätzliche Werte für den Server fest.

```shell
sudo vim /etc/puppet/puppet.conf
```

```ini
[agent]
server = <PUPPET-SERVER-FQDN> # Anpassen!
environment = production

[server]
logdir = /var/log/puppet/puppetserver
rundir = /var/run/puppet/puppetserver
pidfile = /var/run/puppet/puppetserver/puppetserver.pid
codedir = /etc/puppet/code
```

Anschließend aktiven wir die systemd Unit und starten den Puppet Server.

```shell
sudo systemctl enable puppetserver
sudo systemctl start puppetserver
```
