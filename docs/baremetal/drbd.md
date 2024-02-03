# Distributed Replicated Block Device (DBRD)

[TOC]

## Einleitung
__DRBD__ steht für Distributed Replicated Block Device und ist ein Open Source-Treiber,
welcher Geräte im System bereitstellt, die als Blockdevice eingebunden werden können.
Es ist eine Technik, die ein RAID1 (Spiegelung) übers Netzwerk mittels TCP/IP ermöglicht

## DRBD-Konfiguration erstellen

!!! warning "Vorsicht"
    Zunächst muss mit `crm status` sichergestellt werden,
    dass das [Pacemaker Cluster](cluster.md) im Wartungsmodus ist

Als Nächstes muss eine neue DRBD-Konfigurationsdatei für eine
neue Partition angelegt werden.

```none
resource my-resource { # Name der Resource

        protocol C;

        startup {
                wfc-timeout 15;
                degr-wfc-timeout 60;
        }

        net {
                cram-hmac-alg sha256;
                shared-secret "2w6wQ1qB+WClju7YKm8L87BJ"; # Passwort anpassen
        }

        on vm-host-1 { # Erster Knoten
                device /dev/drbd1; # Name der Partition
                disk /dev/vg0/my-resource; # LVM-Partition
                address ipv6 [fe80::ec4:7aff:fe78:f3f4%br0]:7801; # Adresse
                meta-disk internal;
        }

        on vm-host-2 { # Zweiter Knoten
                device /dev/drbd1; # Name der Partition
                disk /dev/vg0/my-resource; # LVM-Partition
                address ipv6 [fe80::ec4:7aff:fe78:f32e%br0]:7801; # Adresse
                meta-disk internal;
        }
}
```

Nach der Änderung des Passwortes `shared-secret` muss festgelegt werden, auf welchen
Knoten die Partitionen angelegt werden müssen. Auf jeden Knoten muss mit 

```commandline
ls /dev/drbd[0-9]* | sort -n
```

geprüft werden, welche Partitionsnummer noch nicht vergeben wurde.
Ebenfalls muss auf jedem Knoten geprüft werden, welcher Port für DRBD
noch nicht verwendet wird:

```commandline
sed '
    /on '$(hostname)'/,/}/!{d;}
    /address/!d
' /etc/drbd.d/*.res | sort -n
```

Um die Übersicht zu behalten, sollte der Port der Nummer der Partition `+ 7800`
entsprechen. Die angegebene Adresse `address` ist die jeweilige Link-Local-Adresse
des Knotens und des entsprechenden Interfaces.

Letztendlich wird die Datei auf den jeweiligen Knoten unter
`/etc/drbd.d/my-resource.res` abgelegt.


## DRBD-Partition initialisieren ##

In einer temporären Umgebungsvariablen legen wir auf beiden Knoten
für die folgenden Befehle den Namen der Resource fest:

```shell
DRBD_RESOURCE_NAME='my-resource' # Namen anpassen
```

Dann folgende Befehle auf beiden DRBD-Knoten ausführen:

```shell
drbdadm create-md "${DRBD_RESOURCE_NAME}" # Resource initialieren
drbdadm up "${DRBD_RESOURCE_NAME}" # Resource starten
```

Dann wird festgelegt, welcher der primäre Knoten ist
und dort die Synchronisation gestartet:

```shell
drbdadm -- --overwrite-data-of-peer primary "${DRBD_RESOURCE_NAME}"
```

Sobald das Synchronisieren fertig, muss auf dem primären Knoten 
die Partition formatiert werden:

```shell
mkfs.ext4 "/dev/drbd/by-res/${DRBD_RESOURCE_NAME}/0"
```

Mit `drbdadm status` oder `drbdmon` kann überprüft werden, ob alles erfolgreich war.
Nun kann die Partition verwendet werden. 

