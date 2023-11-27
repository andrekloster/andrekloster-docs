# Besonderheiten beim Raspberry Pi

[TOC]

## Einleitung
In diesem Kapitel werden ein paar Besonderheiten gezeigt, wenn wir LXC auf einem Raspberry Pi verwenden möchten.

## Aktuellen Debian GPG Key hinzufügen
Nachdem wir das Debian Paket `lxc` installiert haben, erscheint beim Erstellen eines Container folgender Fehler:

```
# sudo lxc-create -t debian -n testcontainer -- -d debian -r bullseye -a armhf
debootstrap is /usr/sbin/debootstrap
Checking cache download in /var/cache/lxc/debian/rootfs-bullseye-armhf ...
gpg: key 7638D0442B90D010: 4 signatures not checked due to missing keys
gpg: key 7638D0442B90D010: "Debian Archive Automatic Signing Key (8/jessie) <ftpmaster@debian.org>" not changed
gpg: Total number processed: 1
gpg:              unchanged: 1
Downloading debian minimal ...
I: Retrieving InRelease
I: Checking Release signature
E: Release signed by unknown key (key id DCC9EFBF77E11517)
   The specified keyring /var/cache/lxc/debian/archive-key.gpg may be incorrect or out of date.
   You can find the latest Debian release key at https://ftp-master.debian.org/keys.html
Failed to download the rootfs, aborting.
Failed to download 'debian base'
failed to install debian
lxc-create: testcontainer: lxccontainer.c: create_run_template: 1626 Failed to create container from template
lxc-create: testcontainer: tools/lxc_create.c: main: 319 Failed to create container testcontainer
```

Es gibt diesen Bug, dass der aktuelle GPG Key, der für die Signatur des Debian Templates verwendet wird, nicht im Paket enthalten ist.
Von daher müssen wir den GPG Key händisch herunterladen und einbinden.

```shell
wget "https://ftp-master.debian.org/keys/release-11.asc" -O /tmp/release-11.asc
gpg --no-default-keyring --keyring /var/cache/lxc/debian/archive-key.gpg --import /tmp/release-11.asc 
gpg --no-default-keyring --keyring /var/cache/lxc/debian/archive-key.gpg --list-key
```

## DHCP Client deaktivieren
Um nicht mehrere IP Adresse im Netzwerk vergeben zu bekommen, müssen wir den `dhcpcd` Daemon permanent deaktivieren.
Erst dann können wir mit der [Einrichtung der Netzwerkbrücke](preparation.md) starten

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
