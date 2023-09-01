# Installation und Konfiguration

[TOC]

## Debian-Paket installieren

Auf dem Host wird XEN mit
```shell
apt install xen-system-amd64
```
installiert.

Damit XEN verwendet werden kann, muss die Datei `/etc/default/grub.d/xen.cfg`
wie folgt angepasst werden:

```
XEN_OVERRIDE_GRUB_DEFAULT=1
#
if [ "$XEN_OVERRIDE_GRUB_DEFAULT" = "" ]; then
        echo "WARNING: GRUB_DEFAULT changed to boot into Xen by default!"
        echo "         Edit /etc/default/grub.d/xen.cfg to avoid this warning."
        XEN_OVERRIDE_GRUB_DEFAULT=1
fi
if [ "$XEN_OVERRIDE_GRUB_DEFAULT" = "1" ]; then
        GRUB_DEFAULT=$( \
                printf "$(gettext "%s, with Xen hypervisor")" \
                "GNU/Linux"
```

Anschließend muss der Host neu gestartet werden.

## Installation von PvGrub

Um einen Kernel im Gastsystem zu verwenden, wird PvGrub verwendet.
PvGrub lädt ein initram-image in den Speicher statt z.Bsp. in den Master-Boot-Record (MBR)
oder in die GUID-Partitionstabelle.

Zum Erstellen des Images legen wir zunächst einen Ordner `/etc/xen/pvgrub` an.
Dort erstellen wir zwei Dateien mit folgendem Inhalt:

```
# /etc/xen/pvgrub/grub-bootstrap.cfg
normal (memdisk)/grub.cfg
```

```
# /etc/xen/pvgrub/grub.cfg
if search -s -f /boot/grub/grub.cfg ; then
        echo "Reading (${root})/boot/grub/grub.cfg"
        configfile /boot/grub/grub.cfg
fi
if search -s -f /grub/grub.cfg ; then
        echo "Reading (${root})/grub/grub.cfg"
        configfile /grub/grub.cfg
fi
```

Nun kann das Image gebaut werden:
```shell
cd /etc/xen/pvgrub
tar cf memdisk.tar grub.cfg
grub-mkimage -O x86_64-xen -c grub-bootstrap.cfg -m memdisk.tar \
    -o grub-x86_64-xen.bin /usr/lib/grub/x86_64-xen/*.mod
```

Quelle: [https://wiki.xenproject.org/wiki/PvGrub2](https://wiki.xenproject.org/wiki/PvGrub2)

## VM-Konfiguration

!!! warning
    Die hier beschriebene Einrichtung ist nur beispielhaft!

### Bootstrapping

Zunächst richten wir mit LVM zwei Partitionen ein.
Die erste Partition ist für den Swap, die zweite Partition
ist die Root-Partition (`/`). Weitere Partitionen können bei Bedarf
hinzugefügt werden (zum Beispiel für `/var`)
```shell
lvcreate -L 15G -n xen-test-root vg0
lvcreate -L 3G -n xen-test-swap vg0
```

Anschließend werden die Partitionen entsprechend formatiert:
```shell
mkfs.ext4 /dev/vg0/xen-test-root
mkswap /dev/vg0/xen-test-swap
```

Um Debian zu installieren, muss mindestens die Root-Partition eingehangen werden:
```shell
mkdir -p /root/vm-x
mount /dev/vg0/xen-test-root /root/vm-x
```

und danach kann mit `debootstrap` Debian installiert werden:
```shell
debootstrap \
    --include='ssh,vim,less,iputils-ping,locales' \
    bullseye \
    /root/vm-x \
    http://ftp.de.debian.org/debian
```

Wir kopieren vom Hostsystem den Ordner `/root/.ssh` mit
```shell
cp -a /root/.ssh /root/vm-x/root/
```

Weiterhin müssen folgende Dateien ggfs. angepasst werden:

* `/root/vm-x/etc/hostname`
* `/root/vm-x/etc/resolv.conf`

Zum automatischen Einhängen der Dateisysteme beim Booten muss die Datei
`/root/vm-x/etc/fstab` erstellt werden.
Die benötigten `UUID`'s erhalten wir mit dem Kommando `blkid /Pfad/zur/Partition`.
Beispiel-Datei:

```
# /etc/fstab: static file system information.
#
# Use 'blkid' to print the universally unique identifier for a
# device; this may be used with UUID= as a more robust way to name devices
# that works even if disks are added and removed. See fstab(5).
#
# <file system> <mount point>   <type>  <options>       <dump>  <pass>
UUID=484fe89d-cbc2-4724-85b5-5aef6ca6f8eb /     ext4    errors=remount-ro 0       1
UUID=80f724e0-0782-4ee4-a62e-417452a026cf none  swap    sw                0       0
```

Damit wir Pakete aus dem Netz laden können, sollte die Datei
`/root/vm-x/etc/network/interfaces.d/eth0` zum Beispiel
mit folgendem Inhalt angelegt werden:

```
auto eth0
iface eth0 inet dhcp
```

Nun wechseln wir mit `chroot` in das künftige Gastsystem:
```shell
chroot /root/vm-x/ /bin/bash
```

um die Spracheinstellungen vorzunehmen:
```shell
dpkg-reconfigure locales # de_DE.UTF-8 und en_US.UTF-8 (default) einstellen
```

Zusätzlich muss ein Passwort für den Nutzer `root`  festgelegt werden:
```shell
passwd root
```

Mit `exit` können wir nun die `chroot`-Umgebung verlassen und anschließend
muss das Dateisystem ausgehangen werden:
```shell
umount /root/vm-x
```

Es muss nun noch eine Konfigurationsdatei für den XEN-Gast angelegt werden.
Hier erfolgt die Konfiguration des zu ladenden Kernels und der "initram",
der Partitionen, Anzahl der CPU's, verfügbarer Arbeitsspeicher und des Netzwerkes.
Beispiel:

```
# /etc/xen/xen-test.conf
#
name = 'xen-test' #  Name des Gastes

#
#  Kernel + CPU + memory size
#
kernel = '/boot/vmlinuz-5.10.0-9-amd64'     # ls -1 /boot/vmlinuz-*
ramdisk = '/boot/initrd.img-5.10.0-9-amd64' # ls -1 /boot/initrd.img-*
vcpus = '1'
memory = '2048'
maxmem = '4096'

#
#  Disk device(s).
#
disk = [
  'phy:/dev/vg0/xen-test-swap,xvda1,w',
  'phy:/dev/vg0/xen-test-root,xvda2,w',
]
root = '/dev/xvda2 ro'

#
#  Networking
#
vif         = [
    'mac=f6:73:55:4d:7d:2d,bridge=br0'
]

#
#  Behaviour
#
on_poweroff = 'destroy'
on_reboot   = 'restart'
on_crash    = 'restart'
```

Nun kann das Gast-System mit
```shell
xl create /etc/xen/xen-test.conf
```

gestartet und mit
```shell
xl console xen-test
```

kann der Bootvorgang verfolgt werden. Erscheint ein Login-Prompt,
war das booten erfolgreich. Anderenfalls kann man mit ++control+"]"++
die Konsole generell verlassen werden.


### Kernel und Grub einrichten

Wir wollen nun erreichen, dass das Gast-System mit einem eigenen Kernel arbeitet.
Damit machen wir uns unabhängig vom Host-System und können den Kernel konfigurieren.

Zunächst melden wir uns mit der Konsole am Gast-System mit
```shell
xl console xen-test
```

an und installieren den Kernel und den Bootloader `grub2`:

```shell
apt install linux-image-amd64 grub2
```

Um ein Grub-Menü zu erstellen, muss die Datei `/etc/default/grub` bearbeitet werden
und folgender Eintrag zur Datei hinzugefügt werden:
```
# Boot Loader Specification (BLS)
# Populate the bootloader's menu entries.
GRUB_ENABLE_BLSCFG=true
```

Nun kann das Grub-Menü mit
```shell
update-grub
```

erstellt werden. Die angezeigten Warnungen können hierbei ignoriert werden.
Das Gastsystem muss nun mit
```shell
shutdown -h now
```

heruntergefahren werden. Dabei wird gleichzeitig die Konsole beendet und wir
befinden uns auf dem XEN-Host.

Damit der installierte Kernel auch geladen wird, muss die Konfiguration des Gasts
in `/etc/xen/xen-test.conf` angepasst. Der Parameter `kernel` wird auf

```shell
kernel = '/etc/xen/pvgrub/grub-x86_64-xen.bin'
```

gesetzt und die Zeile mit dem Parameter `ramdisk` entfernt.

Das Gastsytem kann nun mit
```shell
xl create /etc/xen/xen-test.conf
```

gestartet werden und es kann überprüft werden, ob alles funktioniert.

