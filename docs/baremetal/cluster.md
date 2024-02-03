# Cluster Manager - Pacemaker & Corosync

[TOC]

## Einleitung

Für die Sicherstellung einer hochverfügbaren Serverlandschaft, wird **Corosync** und **Pacemaker** als Cluster Manager verwendet.
Mit Hilfe dieser Software, können Ressourcen wie z.B. virtuelle Maschinen, Linux Container, Floating IPs und DNS verwaltet werden.
Im Falle einer Downtime sorgt, der Cluster Manager dafür, dass die betroffenen Komponenten auf einem andren verfügbaren
Server umgelagert werden.

## Einrichtung von Corosync

Corosync installieren

```shell
apt-get update
apt-get install corosync
```

Corosync Konfiguration (`/etc/corosync/corosync.conf`) anpassen und auf alle Knoten verteilen

```none
# Please read the corosync.conf.5 manual page
totem {
        version: 2

        # Corosync itself works without a cluster name, but DLM needs one.
        # The cluster name is also written into the VG metadata of newly
        # created shared LVM volume groups, if lvmlockd uses DLM locking.
        cluster_name: office-2
        transport: knet
        token_retransmits_before_loss_const: 10

        # crypto_cipher and crypto_hash: Used for mutual node authentication.
        # If you choose to enable this, then do remember to create a shared
        # secret with "corosync-keygen".
        # enabling crypto_cipher, requires also enabling of crypto_hash.
        # crypto works only with knet transport
        crypto_cipher: none
        crypto_hash: none
}

logging {
        # Log the source file and line where messages are being
        # generated. When in doubt, leave off. Potentially useful for
        # debugging.
        fileline: off
        # Log to standard error. When in doubt, set to yes. Useful when
        # running in the foreground (when invoking "corosync -f")
        to_stderr: yes
        # Log to a log file. When set to "no", the "logfile" option
        # must not be set.
        to_logfile: yes
        logfile: /var/log/corosync/corosync.log
        # Log to the system log daemon. When in doubt, set to yes.
        to_syslog: yes
        # Log debug messages (very verbose). When in doubt, leave off.
        debug: off
        # Log messages with time stamps. When in doubt, set to hires (or on)
        #timestamp: hires
        logger_subsys {
                subsys: QUORUM
                debug: off
        }
}

quorum {
        # Enable and configure quorum subsystem (default: off)
        # see also corosync.conf.5 and votequorum.5
        provider: corosync_votequorum
}

nodelist {
        # When knet transport is used it's possible to define up to 8 links
        node {
            name: vm-host-1
            ring0_addr: 192.168.0.11
            nodeid: 1
        }
        node {
            name: vm-host-2
            ring0_addr: 192.168.0.12
            nodeid: 2
        }
        node {
            name: vm-host-3
            ring0_addr: 192.168.0.13
            nodeid: 3
        }
}
```

Corosync starten

```shell
service corosync start
```

## Einrichtung Pacemaker

Pacemaker installieren

```shell
apt-get update
apt-get install pacemaker pcs crmsh
```

Cluster authentifizieren

```shell
passwd hacluster # auf allen Nodes
pcs cluster auth # Benutzer: hacluster; Password: <hacluster_password>
pcs cluster status
```

Pacemaker starten

```shell
service pacemaker start
```

Ersten Knoten (DRBD und LXC) eintragen

```shell
crm conf edit
```

```pcmk
node 1: vm-host-1
node 2: vm-host-2
node 3: vm-host-3

primitive res_cnt_lxc_www-1 lxc \
    params container=www-1 config="/var/lib/lxc/www-1/config" \
    op monitor interval=60s \
    op stop interval=0 timeout=30s
primitive res_drbd_lxc_www-1 ocf:linbit:drbd \
    params drbd_resource=lxc-www-1 \
    op monitor interval=29s role=Master \
    op monitor interval=31s role=Slave \
    op start interval=0 timeout=240s \
    op stop interval=0 timeout=100s
primitive res_fs_lxc_www-1 Filesystem \
    params device="/dev/drbd/by-res/lxc-www-1/0" directory="/var/lib/lxc/www-1" fstype=ext4 \
    op start interval=0 timeout=60s \
    op stop interval=0 timeout=60s
group grp_lxc_www-1 res_fs_lxc_www-1 res_cnt_lxc_www-1
ms ms_drbd_lxc_www-1 res_drbd_lxc_www-1 \
    meta master-max=1 master-node-max=1 clone-max=2 clone-node-max=1 notify=true
colocation cl_fs_lxc_www-1 inf: grp_lxc_www-1 ms_drbd_lxc_www-1:Master
location loc_drbd_lxc_www-1 ms_drbd_lxc_www-1 \
    rule 0: #uname eq vm-host-1 \
    rule 0: #uname eq vm-host-2
location loc_lxc_www-1 grp_lxc_www-1 \
    rule 0: #uname eq vm-host-1 \
    rule 100: #uname eq vm-host-2
order ord_fs_lxc_www-1 ms_drbd_lxc_www-1:promote grp_lxc_www-1:start

primitive st-null stonith:null \
        params hostlist="vm-host-1 vm-host-2 vm-host-3"
clone fencing st-null

property cib-bootstrap-options: \
        have-watchdog=false \
        dc-version=2.0.1-9e909a5bdd \
        cluster-infrastructure=corosync \
        cluster-name=hec1-a \
        stonith-enabled=true \
        symmetric-cluster=false \
        last-lrm-refresh=1567415442 \
        no-quorum-policy=ignore \
        maintenance-mode=true
rsc_defaults rsc-options: \
        resource-stickiness=100
```

!!! danger "Achtung"
    Bei den folgenden Handlungsanweisungen muss unbedingt der Cluster Manager im Wartungsmodus sein,
    da sonst Resourcen unkontrolliert ausgeschaltet werden könnten: <br>
    `/usr/sbin/crm_attribute --type crm_config --name maintenance-mode --update true`

## Pacemaker um eine Ressource erweitern

Falls schon eine aktive Konfiguration vorhanden ist, können wir diese Konfiguration verwenden, um eine Resource zu erstellen.
Dafür kopieren wir zunächst die aktuelle Konfiguration lokal ab und generieren mit `sed` eine neue Resource.

```shell
mkdir -p /tmp/pacemaker; cd /tmp/pacemaker
crm conf show > current_config.pcmk
sed '/^[[:space:]]/{H; d;};  x; /www-1/!d;  s,www-1,www-1,g;' current_config.pcmk > www-1.pcmk
```

Jetzt bearbeiten wir die aktuelle Konfiguration im `Edit-Mode` und fügen ganzen oben unseren neu generierten Teil hinzu.
Die Sortierung der einzelnen Resourcen wird von Pacemaker automatisch übernommen. Sobald alles erfolgreich hinterlegt wurde,
kann die Konfiguration gespeichert und geschlossen werden.

- `crm conf edit`
- `:r www-1.pcmk`
- `:wq`

Anschließend erstellen wir eine [Sandbox](#pacemaker-cluster-testen) und testen die neue Konfiguration.

## Pacemaker Ressource entfernen

Um eine Resource aus der bestehenden Konfiguration zu entfernen, erstellen wir zunächst eine lokale Kopie der Konfiguration.

```shell
mkdir -p /tmp/pacemaker; cd /tmp/pacemaker
crm conf show > current_config.pcmk
```

Anschließend können wir mit `sed` den zu löschenden Teil aus der Konfiguration entfernen.

```shell
{ cat current_config.pcmk; echo; echo placeholder; } | sed '/^[[:space:]]/{H;d;}; x; /www-1/d;' > no_www-1.pcmk
```

Mit Hilfe von `vimdiff` können wir überprüfen, ob das Entfernen erfolgreich war.

```shell
diff current_config.pcmk no_www-1.pcmk
```

Jetzt bearbeiten wir die aktuelle Konfiguration im `Edit-Mode`, löschen den gesamten Inhalt und fügen die neu generierte
Konfiguration ein. Sobald alles erfolgreich hinterlegt wurde, kann die Konfiguration gespeichert und geschlossen werden.

- `crm conf edit`
- `:%d`
- `:r no_www-1.pcmk`
- `:wq`

Anschließend erstellen wir eine [Sandbox](#pacemaker-cluster-testen) und testen die neue Konfiguration.

## Pacemaker Cluster testen

### Sandbox aktivieren

Zuerst stellen wir sicher, dass keine veralteten Pacemaker State Dateien im Cache hinterlegt sind.

```shell
rm /var/run/resource-agents/*
```

Jetzt überprüfen wir die Pacemaker Resourcen.

`-C` löscht aktuelle Zustände, `-R` prüft alle Resourcen neu.

```shell
crm_resource -C
crm_resource -R
```

Anschließend warten wir, bis alle Resourcen geprüft wurden.

Um sicherzustellen, dass Resourcen nicht fehlerhaft verschoben werden, starten wir Pacemaker auf allen Knoten neu.

```shell
service pacemaker stop
sleep 1
service pacemaker start
service pacemaker status
```

Anschließend warten wir erneut, bis alle Resourcen geprüft wurden.

Nun können wir auf einem belieben Knoten die Sandbox erstellen

```shell
crm_shadow --create test_sandbox --force
```

Innerhalb der Sandbox stoppen wir den maintenance mode

```shell
/usr/sbin/crm_attribute --type crm_config --name maintenance-mode --update false
```

Jetzt können wir die Simulation starten, um das Verhalten vom Cluster vorhersagen zu können.

```shell
crm_simulate -Ls
```

Das Feld `Transition Summary:` gibt an, wie sich das Cluster produktiv verhalten würde.

### Sandbox ausführen/verwerfen

Wenn wir mit dem Zustand der Sandbox zufrieden sind, können wir das Cluster in Betrieb nehmen. Damit wird das Cluster
online gesetzt und alle Änderungen werden durchgeführt.

```shell
crm_shadow --commit test_sandbox --force
```

Sollte der Zustand der Sandbox den Erwartungen nicht entsprechen, kann die Sandbox gelöscht und alle Änderung verworfen werden.
Dadurch bleibt das Cluster weiterhin im maintenance mode.

```shell
crm_shadow --delete test_sandbox --force
```

### Cluster prüfen

Sobald das Cluster online ist, kann geprüft werden, ob alle Resourcen vom Pacmaker korrekt gestartet werden.

```shell
pcs resource relocate show
```

## Einzelne Ressource auf einen anderen Knoten umziehen

```shell
crm resource move res_cnt_lxc_www-1 vm-host-1
```

Ggf. auch mit dem Tool `pcs` möglich

```shell
pcs resource move res_cnt_lxc_www-1 vm-host-1
```

Nachträglich die Bindung an den Knoten lösen

```shell
crm resource unban res_cnt_lxc_www-1
```

## Knoten im Cluster auf Standby setzen

Mit dem folgenden Befehl wird der angegebene Knoten in den Standby-Modus gesetzt.
Alle Ressourcen, die derzeit auf dem Knoten aktiv sind, werden auf einen anderen Knoten verschoben.

```shell
pcs node standby vm-host-x
```

Mit dem folgenden Befehl wird der angegebene Knoten aus dem Standby-Modus entfernt.

```shell
pcs node unstandby vm-host-x
```

## Sonderfälle

Falls Container absolut nicht umziehen möchte und sich immer an eine Node bindet

```shell
crm conf show > current_config.pcmk
```

```shell
service pacemaker stop; service corosync stop
```

```shell
mv /var/lib/pacemaker/cib /root/
```

```shell
rm /var/run/resource-agents/*
```

```shell
service corosync start && service pacemaker start
```

Anschließend die Pacemaker Konfiguration mit `crm conf edit` eintragen
