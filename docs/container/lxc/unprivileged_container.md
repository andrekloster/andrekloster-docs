# Einen unprivilegierten Linux Container erstellen

[TOC]

## Einleitung
Ein unprivilegierter Container bietet die Möglichkeit Prozessisolation als unprivilegierten Benutzer durchzuführen.
Bei dieser Art des LXCs findet auf dem Hostsystem ein Mapping der User/Group-IDs statt.
Beispiel:

| __User__ | __Container-UID__ | __Host-UID__ |
|----------|-------------------|--------------|
| root     | 0                 | 100000       |

Innerhalb des Containers besitzt der User `root` wie gewohnt die ID `0`. Sollte dieser Prozess aus Isolierung ausbrechen
können, dann besitzt er auf dem Hostsystem eine viel höhere ID. Damit ist dieser Prozess nicht in der Lage größeren
Schaden auf dem Hostsystem anzurichten.

Die einfachste Einrichtung eine unprivilegierte Containers erfolgt als Benutzer `root`.
Zunächst muss das Mapping der IDs eingetragen werden.

+ User-ID: `/etc/subuid`
+ Group-ID: `/etc/subgid`

In beide Dateien soll folgender Eintrag stehen: `root:1000000:65536`

Damit legen wir fest, dass die neuen Container-IDs für `root` ab `1000000` beginnen und die maximale Anzahl `65536` beträgt.

## Installation
Anschließend erfolgt die Einrichtung des Linux Containers. Für unprivilegierte Container muss explizit das Template Namens
`download` verwendet werden. Dieses Template ist speziell für das Mapping der UIDs und GIDs angepasst worden.

```shell
sudo lxc-create -t download -n unpriv-container-1 -- -d debian -r bullseye -a amd64
```

Sobald der Container heruntergeladen worden ist, muss das ID Mapping in der LXC Konfiguration (`/var/lib/lxc/unpriv-container-1/config`)
des neuen Container vermerkt werden:

```conf
# Template used to create this container: /usr/share/lxc/templates/lxc-download
# Parameters passed to the template:
# For additional config options, please look at lxc.container.conf(5)

# Uncomment the following line to support nesting containers:
#lxc.include = /usr/share/lxc/config/nesting.conf
# (Be aware this has security implications)

# Distribution configuration
lxc.include = /usr/share/lxc/config/common.conf
lxc.include = /usr/share/lxc/config/userns.conf
lxc.arch = amd64

# Container specific configuration
lxc.idmap = u 0 100000 65536 # WICHTIG FÜR UNPRIVILEGIERTE CONTAINER
lxc.idmap = g 0 100000 65536 # WICHTIG FÜR UNPRIVILEGIERTE CONTAINER
lxc.apparmor.profile = generated
lxc.apparmor.allow_nesting = 1
lxc.mount.auto = sys:mixed
lxc.mount.entry = /dev/fuse dev/fuse none bind,optional,create=file
lxc.rootfs.path = dir:/var/lib/lxc/unpriv-container-1/rootfs
lxc.uts.name = unpriv-container-1
lxc.tty.max = 2

# Network configuration
lxc.net.0.type = veth
lxc.net.0.hwaddr = 5a:18:a5:1e:dc:36
lxc.net.0.link = br0
lxc.net.0.flags = up
lxc.net.0.name = eth0
```

Anschließend kann der unprivilegierte Container gestartet werden:

```shell
sudo lxc-start unpriv-container-1
```

## Optional: Linux Container mit weiteren Debian-Paketen installieren
Falls gewünscht, kann schon während der Erstellung eines Linux Container noch zusätzliche Software installiert werden,
bevor der Container zum ersten Mal gestartet wurde.

```shell
sudo lxc-create -t download -n unpriv-container-1 -- -d debian -r bullseye -a amd64 \
  --packages='iputils-ping,man-db,vim,less,apt-transport-https,gnupg,net-tools,lsb-release,curl,dbus'
```
