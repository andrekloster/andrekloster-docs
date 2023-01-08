# Vorbereitugen

[TOC]

## Einleitung ##
[LXC](https://linuxcontainers.org/lxc/introduction/) (Linux Containers) ist ein Verfahren zur Prozessisolierung auf Betriebssystemebene,
das mehrere laufende Linux-Systeme auf einem einzigen Host ermöglicht.

## LXC installieren
Zuerst installieren wir das Debian Paket für LXC:

```shell
sudo apt update
sudo apt install -y lxc
```

## Netzwerkbrücke einrichten
Bevor wir einen Linux Container starten müssen wir eine Netzwerkbrücke vorbereiten. Diese Netzwerkbrücke ist notwendig, damit den Traffic 
zwischen Host und Container durchreichen können. Nachdem wir `lxc` installiert haben, wird automatisch eine `lxcbr` Bridge erstellt.
Diese Netzwerkbrücke erzeugt ein separates Netzwerk, welches via NAT den Traffic verwaltet. Grundsätzlich spricht nichts dagegen diese Netzwerkbrücke zu verwenden.
Aus Bequemlichkeit möchten wir jedoch, dass der Host und der Container sich im selben Netzwerk befinden. Dazu deaktivieren wir die `lxcbr`
und erstellen mit Hilfe des Paketes `bridge-utils` unsere eigene Netzwerkbrücke.

Wir öffnen die Einstellungen der LXC Netzwerkbrücke und schalten sie aus.

```
sudo vim /etc/default/lxc-net
```

```
# ...
USE_LXC_BRIDGE="false"
# ...
```

```shell
sudo systemctl stop lxc-net
sudo systemctl disable lxc-net
```

Als nächstes installieren wir `bridge-utils` und konfigurieren unsere eigene Netzwerkbrücke.

```shell
sudo apt update
sudo apt install -y bridge-utils
```

```shell
sudo vim /etc/network/interfaces
```

```
# interfaces(5) file used by ifup(8) and ifdown(8)
# Include files from /etc/network/interfaces.d:
source /etc/network/interfaces.d/*
```

```shell
sudo vim /etc/network/interfaces.d/br0
```

In unserer Konfiguration der `br0` lassen wir den DHCP Servers des Netzwerkes die IPv4 Adresse vergeben.

```
# Set up interfaces manually, avoiding conflicts with, e.g., network manager
iface eth0 inet manual

# Bridge setup
auto br0
iface br0 inet dhcp
  bridge_ports eth0
```

Sobald alles vollständig und korrekt konfiguriert wurde, starten wir den Host Server neu.

```
sudo shutdown -r now
```
