# Einen Puppet Server erstellen

[TOC]

## Ziel
Um eine Infrastruktur mit Puppet aufzubauen, benötigen wir in erster Linie einen Puppet Server.
Dieser Server ist das zentrale Objekt der Infrastruktur, bei dem die Clients in regelmäßigen Abständen
nach Änderungen abfragen.

## Voraussetzungen
- Linux Server (z.Bsp. Debian Bullseye)
- Mindestens 1GB frei verfügbaren RAM

## Vorbereitungen
Um den Puppet Server installieren zu können, müssen wir zunächst die apt-source-list inkl. GPG von Puppet auf unserem Linux Server einrichten.

```shell
# Installiere notwendige Pakete für apt-source-list via HTTPs und GPG
apt update
apt install -y wget gnupg apt-transport-https
```

```shell
# Für die apt-source-list hinzu
echo "deb https://apt.puppetlabs.com/ bullseye puppet7" > /etc/apt/sources.list.d/puppet.list
```

```shell
# Lade den GPG public keyring von Puppet herunter und füge ihn im System hinzu.
wget -4 https://apt.puppetlabs.com/keyring.gpg -O /tmp/puppet.gpg
cat /tmp/puppet.gpg | gpg --dearmor >/etc/apt/trusted.gpg.d/puppet.gpg
```

## Installation und Konfiguration des Puppet Servers

```shell
# Installiere puppetserver
apt update
apt install -y puppetserver
```

Nun öffnen wir die Puppet Konfigurationsdatei und legen grundsätzliche Werte für den Server fest.

```shell
vim /etc/puppetlabs/puppet/puppet.conf
```

```ini
[agent]
server = <SERVER-FQDN> # Anpassen!
environment = production

[server]
vardir = /opt/puppetlabs/server/data/puppetserver
logdir = /var/log/puppetlabs/puppetserver
rundir = /var/run/puppetlabs/puppetserver
pidfile = /var/run/puppetlabs/puppetserver/puppetserver.pid
codedir = /etc/puppetlabs/code
```

Anschließend aktiven wir die systemd Unit und starten den Puppet Server.

```shell
systemctl enable puppetserver
systemctl start puppetserver
```
