# Einen privilegierten Linux Container erstellen

Ein privilegierter Container hat im Gegensatz zu einem [unprivilegierten Container](/container/lxc/unprivileged_container)
mehr Rechte. Hier entfällt das Mapping der UIDs und GIDs. Dh., dass Prozesse die von root gestartet werden, sowohl innerhalb,
als auch außerhalb des Containers die gleiche UID 0 besitzen.


| __User__ | __Container-UID__ | __Host-UID__ |
|----------|-------------------|--------------|
| root     | 0                 | 0            |

Es ist empfehlenswert einen privilegierten Container nur im Ausnahmezustand zu erstellen. Ein Anwendungsfall wäre z.Bsp.
das Einhängen einer ein externen Partition in den Container. In unserem Beispiel gehen wir davon aus, dass wir eine
Partition `/dev/sda1` im `ext4` Format zur Verfügung haben. Die externe Partition werden wir in den Container einhängen.

Zunächst erstellen wir einen privilegierten Container, indem wir folgenden Befehl ausführen:

```shell
sudo lxc-create -n priv-container-1 -- -d debian -r bullseye -a amd64
```

Sobald der Container erstellt worden ist, passen die LXC Konfiguration (`/var/lib/lxc/priv-container-1/config`) an:

```conf
# For additional config options, please look at lxc.container.conf(5)

# Uncomment the following line to support nesting containers:
#lxc.include = /usr/share/lxc/config/nesting.conf
# (Be aware this has security implications)

# Distribution configuration
lxc.include = /usr/share/lxc/config/common.conf
lxc.arch = amd64

# Container specific configuration
lxc.apparmor.profile = generated
lxc.apparmor.allow_nesting = 1
lxc.mount.auto = sys:mixed
lxc.mount.entry = /dev/fuse dev/fuse none bind,optional,create=file
lxc.mount.fstab = /var/lib/lxc/priv-container-1/fstab # ACHTUNG! Die Datei fstab muss angelegt werden, sonst startet der Container nicht!
lxc.rootfs.path = dir:/var/lib/lxc/priv-container-1/rootfs
lxc.uts.name = priv-container-1
lxc.tty.max = 2

# Network configuration
lxc.net.0.type = veth
lxc.net.0.hwaddr = d6:92:b7:7b:1b:9e
lxc.net.0.link = br0
lxc.net.0.flags = up
lxc.net.0.name = eth0
```

!!! warning
    Die Datei `/var/lib/lxc/priv-container-1/fstab` muss vorhanden sein! Ansonsten wird der Container nicht starten können.
    Zusätzlich muss innerhalb des Container der Ordner (`/var/lib/lxc/priv-container-1/rootfs/mnt/data`) für den Mountpoint angelegt sein!

```shell
sudo vim /var/lib/lxc/priv-container-1/fstab
```

In diese `fstab` Datei tragen wir nun folgendes ein:

```
/dev/sda1 mnt/data ext4 defaults 0 0
```

Anschließend erstellen wir der Ordner für den Mountpoint innerhalb des Containers

```shell
sudo mkdir -p /var/lib/lxc/priv-container-1/rootfs/mnt/data
```

Anschließend kann der privilegierte Container gestartet werden:

```shell
sudo lxc-start priv-container-1
```