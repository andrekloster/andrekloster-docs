# KVM #

[TOC]

## Bootstrapping ##

Zunächst richten wir mit LVM eine virtuelle Festplatte ein.

```shell
lvcreate -L 30G -n kvm-example vg0
```

Als Nächstes wird die Festplatte `cfdisk` partitioniert.
Um auf die LVM-Partition wie auf eine Festplatte zugreifen zu können,
erstellen wir temporär ein Loopback-Device:

```shell
losetup -Pf --show /dev/vg0/kvm-example
```

In unserem Beispiel wird das Device `/dev/loop0` erstellt,
was die virtuelle Festplatte darstellt, die nun mit `cfdisk /dev/loop0`
partitioniert werden kann. Wir benötigen in diesem Beispiel 3 Partitionen:

- `1MB` `BIOS boot`
- `1GB` `Linux swap`
- der Rest `Linux filesystem`.

Mit

```shell
fdisk -l /dev/loop0
```

können wir uns die Partitionen anzeigen lassen.

```
Disk /dev/loop0: 30 GiB, 32212254720 bytes, 62914560 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: gpt
Disk identifier: AD8E9C13-BA17-7B47-A5FA-2962452EFF6D

Device         Start      End  Sectors Size Type
/dev/loop0p1    2048     4095     2048   1M BIOS boot
/dev/loop0p2    4096  2101247  2097152   1G Linux swap
/dev/loop0p3 2101248 62912511 60811264  29G Linux filesystem
```

Wir können nun die Partitionen formatieren und notieren uns die dabei
ausgegebenen UUID's der Dateisysteme, die wir später für die `/etc/fstab` benötigen:

```shell
mkswap /dev/loop0p2
mkfs.ext4 /dev/loop0p3
```

Anschließend hängen wir die Root-Partition in ein Verzeichnis ein.

```shell
mkdir -p /root/vm-host-x
mount /dev/loop0p3 /root/vm-host-x
```

Mit `debootstrap` installieren wir ein minimales Linux-System:

```shell
debootstrap \
  --include='linux-image-amd64,grub2,dbus,ssh,vim,less,iputils-ping,locales,lsb-release,xterm' \
  bookworm /root/vm-host-x http://ftp.de.debian.org/debian
```

Für die folgende Netzwerkkonfiguration generieren wir uns eine
[zufällig lokal verwaltete Unicast-MAC-Adresse](/server/tools/snippets/#zufallig-lokal-verwaltete-unicast-mac-adresse)
und hängen danach die Geräte-Verzeichnisse ein:

```shell
mount --bind /dev /root/vm-host-x/dev
mount --bind /dev/pts /root/vm-host-x/dev/pts
mount -t proc  proc /root/vm-host-x/proc
mount -t sysfs sysfs /root/vm-host-x/sys
[ -d "/root/vm-host-x/run/udev" ] || mkdir /root/vm-host-x/run/udev 
mount --bind /run/udev /root/vm-host-x/run/udev
```

Mit 

```shell
chroot /root/vm-host-x
```

wechseln wir das erste Mal in das Gast-System.
Hier bereiten wir das Gast-System für den ersten Bootvorgang vor.

Zuerst wird das _Locale_ konfiguriert:

```shell
sed -i '/en_US.UTF-8/s,^.*en_US.UTF-8,en_US.UTF-8,g;' /etc/locale.gen
echo 'LANG=en_US.UTF-8' >/etc/default/locale
locale-gen
```

und setzen den Namen des Gastsystemes mit:

```shell
echo 'kvm-example' >/etc/hostname 
```

Die Datei `/etc/resolv.conf` sollte bereits vom Host-System übernommen worden sein.
Mit `passwd` setzen wir das Passwort für den Nutzer `root` und aktivieren
die Konsole mit

```shell
systemctl enable serial-getty@ttyS0.service
```

Für den ersten Bootvorgang konfigurieren wir die Netzwerkverbindung mit DHCP
und legen dazu die Datei `/etc/network/interfaces.d/eth0` mit
folgendem Inhalt an

```
auto eth0
iface eth0 inet dhcp
iface eth0 inet6 auto
```

und setzen mit der bereits generierten Mac-Adresse den Namen des Interfaces
in der Datei `/etc/systemd/network/10-persistent-net-eth0.link` mit folgendem Inhalt:

```
[Match]
MACAddress=3a:8c:bd:1b:2b:c3
OriginalName=*

[Link]
Name=eth0
```

Dann konfigurieren wir das Dateisystem in der `/etc/fstab` mit
den uns bereits bekannten UUIDs der Dateisyteme.

```
# <file system> <mount point>   <type>  <options>       <dump>  <pass>
UUID=d27d0904-4636-4d75-9e0d-3b8bf0195add /     ext4    errors=remount-ro 0       1
UUID=d5fa554b-530f-41e6-9054-f6e99d5f797c none  swap    sw                0       0
```

Als Letztes wird der Bootloader _Grub_ konfiguriert und installiert:

```shell
sed -i '/GRUB_CMDLINE_LINUX=/s|.*|GRUB_CMDLINE_LINUX="console=tty0 console=ttyS0,115200n8"|g;' /etc/default/grub
echo 'GRUB_TERMINAL=serial' >>/etc/default/grub
echo 'GRUB_SERIAL_COMMAND="serial --unit=0 --speed=9600 --stop=1"' >>/etc/default/grub

grub-mkconfig -o /boot/grub/grub.cfg
grub-install --target=i386-pc --recheck /dev/loop0
```

Wir verlassen nun das Gastsystem in der `chroot`-Umgebung mit dem Befehl `exit`
und hängen alle Dateisysteme aus:

```shell
for i in dev/pts dev proc sys run/udev; do umount /root/vm-host-x/$i; done
umount /root/vm-host-x
```

Die virtuelle Festplatte `/dev/loop0` wird nun nicht mehr benötigt:

```shell
losetup -d /dev/loop0
```

und wir können nun die Virtualisierung das erste Mal booten:

```shell
qemu-system-x86_64 \
  -enable-kvm -m 2048 -cpu host -smp 2 \
  -hda /dev/vg0/kvm-example \
  -boot c  \
  -nic user,model=e1000,mac=3a:8c:bd:1b:2b:c3 \
  -nographic 
```

Nach einiger Zeit sollte das Login erscheinen, mit dem man sich beim Gast-System
anmelden kann. Das System ist dabei bereit netzwerkfähig, aber von außen nicht erreichbar.

!!! info "Tipp"
    Mit ++ctrl+"a"++ und anschließenden Drücken von ++"c"++ kann das Terminal verlassen werden.
    Es erscheint der Befehls-Prompt von _qemu_. Mit _quit_ wird _qemu_ beendet.
    Besser ist es das Gast-System mit `shutdown -h now` sauber herunterzufahren.

!!! info "Größe der Konsole"
    Nach dem erfolgreichen Login kann mit dem Befehl `resize` die Größe des Terminals
    angepasst werden.

## libvirt

Mit _libvirt_ persistieren wir die erstellte Virtualisierung.  

```shell
virt-install \
  --import \
  --name 'kvm-example' \
  --virt-type kvm \
  --cpu host \
  --os-variant debian11 \
  --disk 'path=/dev/vg0/kvm-example,format=raw' \
  --memory 2048 \
  --vcpus 2 \
  --network 'bridge=br0,mac=3a:8c:bd:1b:2b:c3' \
  --graphics none
```

Die Virtualisierung wird erneut gestartet und wir können uns mit Nutzername und Passwort
am Gast-System anmelden.

Typischerweise können nun Netzwerkanpassungen durchgeführt werden.
Das bereits eingerichtete Netzwerk über DHCP sollte bereits einen Netzwerkzugang
über ssh zulassen.

!!! info "Tipp"
    Mit ++ctrl+"]"++ kann das Terminal verlassen werden.
    Vorher nicht vergessen sich vom Gast-System abzumelden.
    Die Virtualisierung läuft weiter.

Mit `virsh list --all` kann man sich alle Virtualisierungen anschauen.
`virsh` ist auch das Werkzeug, mit dem die Virtualisierung verwaltet werden kann.
So kann man sich mit `virsh console` per Terminal bei der Virtualisierung anmelden,
falls man nicht per Netzwerk darauf zugreifen kann. 

## Vorbereitung für das Cluster

Aktuell ist die Virtualisierung nur auf einem Host verwendbar.
Ziel ist es, dass die Virtualisierung weitestgehend unabhängig vom Host ist.
Davon betroffen sind insbesondere UUID's der Dateisysteme.

Dazu muss als Erstes die `/etc/fstab` angepasst werden.
Da die virtualisierten Devices nun bekannt sind,
wird die `/etc/fstab` wie folgt angepasst:

```
# <file system> <mount point>   <type>  <options>       <dump>  <pass>
/dev/vda3 /     ext4    errors=remount-ro 0       1
/dev/vda2 none  swap    sw                0       0
```

Als Nächstes müssen wir noch den Bootloader anpassen:

```shell
sed -i '/GRUB_DISABLE_LINUX_UUID/s,.*,GRUB_DISABLE_LINUX_UUID=true,g;' /etc/default/grub
update-grub
```

Abschließend neu booten.

## Data-Partition

Wir widerholen die oben beschriebenen Schritte, wie das Einrichten einer virtuellen Festplatte, 
das Erstellen eines Loopback-Geräts, das Partitionieren und Formatieren der Festplatte und das Mounten der Root-Partition.

```shell
lvcreate -L 50G -n kvm-example-data vg0
losetup -Pf --show /dev/vg0/kvm-example-data
cfdisk /dev/loop0
mkfs.ext4 /dev/loop0p1
mount /dev/loop0p1 /root/vm-host-x/
```

die Datenpartition kann jetzt mit unserem KVM verbunden werden.

```shell
virsh attach-disk kvm-example --source /dev/vg0/kvm-example-data --target vdb --persistent
```

Dann loggen wir uns über ssh in die KVM ein, erstellen einen Mount-Ordner und ergänzen `/etc/fstab/`.

```shell
# <file system> <mount point>   <type>  <options>       <dump>  <pass>
/dev/vda2 /     ext4    errors=remount-ro 0       1
/dev/vdb1 /mnt/data     ext4    errors=remount-ro 0       1
```

Nun hängen wir sowohl die Festplatte als auch die virtuelle Festplatte aus.

```shell
umount /root/vm-host-x/
losetup -d /dev/loop0
```
