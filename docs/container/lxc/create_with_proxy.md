# Container über einen Proxy erstellen

[TOC]

## Einleitung
In einigen Fällen kann sich der LXC Host Server hinter einer Firewall befinden, die den Traffic zum Internet nur über einen bestimmten Proxy erlaubt.
Das folgende Beispiel zeigt, wie wir diesen Proxy beim Erstellen eines Linux Containers verwendet können.

## Voraussetzung
Wir gehen davon aus, dass unser Proxy den Hostnamen `proxy.domain.local` hat und auf dem Port `3142` läuft.
Das ist ein üblicher Port für einen [Squid HTTP-Proxy](http://www.squid-cache.org/).

## Umgebungsvariablen setzen und LXC erstellen
Wir exportieren für die aktuelle Shell folgende Umgebungsvariablen, um die Debain apt-source-lists über den Proxy zu erreichen.

```shell
export MIRROR=http://proxy.domain.local:3142/ftp.de.debian.org/debian
export SECURITY_MIRROR=http://proxy.domain.local:3142/ftp.de.debian.org/debian-security
```

Anschließend erstellen wie gewohnt einen [privilegierten](privileged_container.md) oder [unprivilegierten](unprivileged_container.md) Linux Container.
