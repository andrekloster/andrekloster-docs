# Besondersheiten beim Raspberry Pi

[TOC]

## Einleitung
In diesem Kapitel werden ein paar Besonderheiten gezeigt, wenn wir LXC auf einem Raspberry Pi verwenden möchten.

## DHCP Client deaktivieren
Um nicht mehrere IP Adresse im Netzwerk vergeben zu bekommen, müssen wir den `dhcpcd` Daemon permanent deaktivieren.
Erst dann können wir mit der [Einrichtung der Netzwerkbrücke](/container/lxc/preparation) starten

```shell
sudo systemctl stop dhcpcd
sudo systemctl disable dhcpcd
```

## CPU Architektur
Bei Erstellen eines Containers müssen angeben, dass wir die `armhf` CPU Architektur verwenden

+ privilegierter Container

```shell
sudo lxc-create -t download -n unpriv-container-1 -- -d debian -r bullseye -a armhf
```

+ privilegierter Container

```shell
sudo lxc-create -n priv-container-1 -- -d debian -r bullseye -a armhf
```

Zusätzlich müssen wir darauf achten, dass die richtige CPU Architektur im der LXC config angegeben wird.

```shell
sudo vim /var/lib/lxc/container-1/config
```

```config
# ...
lxc.arch = armhf
# ...
```
