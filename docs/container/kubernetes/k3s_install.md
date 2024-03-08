# Kubernetes - K3s
## Einleitung
**Kubernetes** ist eine Open-Source-Plattform zur Automatisierung der Bereitstellung, Skalierung und Verwaltung von 
containerisierten Anwendungen. Es ermöglicht es Benutzern, Cluster von Hosts, die Container ausführen, zu verwalten 
und automatisiert viele Aspekte der Anwendungsverteilung und -wartung. **K3s** ist eine leichtgewichtige, einfach zu 
installierende Distribution von Kubernetes, die speziell für Edge-Computing, IoT und ressourcenbeschränkte Umgebungen 
entwickelt wurde. Es bietet eine vereinfachte und leichtgewichtige Implementierung von Kubernetes, die mit reduziertem 
Ressourcenbedarf auskommt, ohne dabei wesentliche Funktionen zu opfern.

## Vorbereitungen
### Proxy
Wie in der oberen Grafik dargestellt, benötigen wir einen Layer-4 Loadbalancer, der alle API-Anfragen zum Port 6443
an alle Control-Planes des Cluster verteilt. [HAProxy](https://www.haproxy.org/) eignet sich sehr gut für diese Aufgabe.

Eine beispielhafte HA-Proxy Konfiguration (`/etc/haproxy/haproxy.cfg`) könnte wie folgt aussehen:

```
global
    log /dev/log    local0
    log /dev/log    local1 notice
    chroot /var/lib/haproxy
    stats socket /run/haproxy/admin.sock mode 660 level admin
    stats timeout 30s
    user haproxy
    group haproxy
    daemon
    
    # Default SSL material locations
    ca-base /etc/ssl/certs
    crt-base /etc/ssl/private
    
    # See: https://ssl-config.mozilla.org/#server=haproxy&server-version=2.0.3&config=intermediate
    ssl-default-bind-ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:DHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384
    ssl-default-bind-ciphersuites TLS_AES_128_GCM_SHA256:TLS_AES_256_GCM_SHA384:TLS_CHACHA20_POLY1305_SHA256
    ssl-default-bind-options ssl-min-ver TLSv1.2 no-tls-tickets

defaults
    log     global
    mode    http
    option  httplog
    option  dontlognull
    timeout connect 5000
    timeout client  50000
    timeout server  50000
    errorfile 400 /etc/haproxy/errors/400.http
    errorfile 403 /etc/haproxy/errors/403.http
    errorfile 408 /etc/haproxy/errors/408.http
    errorfile 500 /etc/haproxy/errors/500.http
    errorfile 502 /etc/haproxy/errors/502.http
    errorfile 503 /etc/haproxy/errors/503.http
    errorfile 504 /etc/haproxy/errors/504.http

frontend kubernetes_api
    bind 10.123.3.5:6443
    mode tcp
    option tcplog
    default_backend kubernetes_api_backend

backend kubernetes_api_backend
    mode tcp
    balance roundrobin
    option tcp-check
    server kube-controller-1 10.123.3.20:6443 check
    server kube-controller-2 10.123.3.21:6443 check
    server kube-controller-3 10.123.3.22:6443 check
```

## Installation
K3s kann in verschiedenen Architekturen aufgebaut werden. Mit einem oder mit mehreren Nodes. Hochverfügbar oder nicht
hochverfügbar. Ebenso mit unterschiedlichen Storage Backends (sqlite, etcd, postgres, etc). Ebenso können auch
unterschiedliche Parameter bei der Installation mitgegeben werden, um gewisse Features (z.B. Loadbalancer, Networking etc.)
anzupassen oder zu deaktivieren. Das dazugehörige Installationsskript befindet sich bei [k3s.io](https://k3s.io/).

### Single Node (Controller & Worker) - Sqlite ohne LB
Das folgende Skript erstellt ein Single-Node Kubernetes Cluster, das sowohl Controller, als auch Worker ist.
Optional werden einige Netwerkkonfigurationen explizit angepasst. Zum einen deaktivieren wird `servicelb`, um
im späteren Verlauf [MetalLB](https://metallb.universe.tf/) ohne Probleme verwenden zu können. Ebenso optional ist
die Vergabe von `tls-san`, falls das Cluster über mehrere IPs erreichbar ist oder hinter einem LoadBalancer steht.

```shell
curl -sfL https://get.k3s.io | \
INSTALL_K3S_VERSION="v1.28.1+k3s1" \
  K3S_TOKEN="MY_SECRET_TOKEN" \
  INSTALL_K3S_EXEC="server \
  --disable servicelb \
  --node-name="$(hostname -f)" \
  --cluster-cidr=192.168.0.0/16 \
  --kube-controller-manager-arg="bind-address=0.0.0.0" \
  --kube-proxy-arg="metrics-bind-address=0.0.0.0" \
  --kube-scheduler-arg="bind-address=0.0.0.0" \
  --advertise-address=$(hostname -I | awk '{print $2}') \
  --node-ip=$(hostname -I | awk '{print $2}') \
  --node-external-ip=$(hostname -I | awk '{print $1}') \
  --tls-san=10.123.1.5 \
  --tls-san=10.123.3.5 \
  --tls-san=10.123.1.20 \
  --tls-san=10.123.3.20" sh -
```

### ECTD
Mit den folgenen Befehlen wir ein HA-Setup mit einem internen ECTD Server Cluster aufgebaut

```shell
# Init Control Plane
curl -sfL https://get.k3s.io | \
INSTALL_K3S_VERSION="v1.27.3+k3s1" \
K3S_TOKEN="MY_SECRET_TOKEN" \
INSTALL_K3S_EXEC="server \
--disable servicelb \
--node-name="$(hostname -f)" \
--cluster-cidr=192.168.0.0/16 \
--etcd-expose-metrics=true \
--kube-controller-manager-arg="bind-address=0.0.0.0" \
--kube-proxy-arg="metrics-bind-address=0.0.0.0" \
--kube-scheduler-arg="bind-address=0.0.0.0" \
--node-taint CriticalAddonsOnly=true:NoExecute \
--advertise-address=$(hostname -I | awk '{print $2}') \
--node-ip=$(hostname -I | awk '{print $2}') \
--node-external-ip=$(hostname -I | awk '{print $1}') \
--tls-san=10.123.1.5 \
--tls-san=10.123.3.5 \
--tls-san=10.123.1.20 \
--tls-san=10.123.3.20 \
--tls-san=10.123.1.21 \
--tls-san=10.123.3.21 \
--tls-san=10.123.1.22 \
--tls-san=10.123.3.22 \
--cluster-init" sh -

# Weitere Control Planes hinzufügen
curl -sfL https://get.k3s.io | \
INSTALL_K3S_VERSION="v1.27.3+k3s1" \
K3S_TOKEN="MY_SECRET_TOKEN" \
INSTALL_K3S_EXEC="server \
--disable servicelb \
--node-name="$(hostname -f)" \
--cluster-cidr=192.168.0.0/16 \
--etcd-expose-metrics=true \
--kube-controller-manager-arg="bind-address=0.0.0.0" \
--kube-proxy-arg="metrics-bind-address=0.0.0.0" \
--kube-scheduler-arg="bind-address=0.0.0.0" \
--node-taint CriticalAddonsOnly=true:NoExecute \
--advertise-address=$(hostname -I | awk '{print $2}') \
--node-ip=$(hostname -I | awk '{print $2}') \
--node-external-ip=$(hostname -I | awk '{print $1}') \
--flannel-iface=ens19 \
--tls-san=10.123.1.5 \
--tls-san=10.123.3.5 \
--tls-san=10.123.1.20 \
--tls-san=10.123.3.20 \
--tls-san=10.123.1.21 \
--tls-san=10.123.3.21 \
--tls-san=10.123.1.22 \
--tls-san=10.123.3.22 \
--server https://10.123.3.5:6443" sh -

# Worker Nodes hinzufügen
curl -sfL https://get.k3s.io | \
K3S_URL="https://10.123.3.5:6443" \
INSTALL_K3S_VERSION="v1.27.3+k3s1" \
K3S_TOKEN="MY_SECRET_TOKEN" \
INSTALL_K3S_EXEC="--node-external-ip=$(hostname -I | awk '{print $1}') --node-name=$(hostname -f)" \
sh -
```

### Externer PostgreSQL Datastore
Um eine PostgreSQL als externen Datastore für den Clusterzustand zu verwenden, muss zuũachst die Datenbank inkl.
Rechtevergabe eingerichtet werden. Anschließend können die Control Planes die Datenbank als `--datastore-endpoint`
verwenden.

```SQL
CREATE DATABASE k3s;
CREATE USER k3s WITH ENCRYPTED PASSWORD 'MY_DB_SECRET';
GRANT ALL PRIVILEGES ON DATABASE k3s TO k3s;
\c k3s postgres
GRANT ALL ON SCHEMA public TO k3s;
```

```shell
# Init Control Plane
curl -sfL https://get.k3s.io | \
K3S_TOKEN="MY_SECRET_TOKEN" \
INSTALL_K3S_EXEC="server \
--disable servicelb \
--datastore-endpoint="postgres://k3s:MY_DB_SECRET@db-pgsql-1.domain.local:5432/k3s" \
--node-name="kube-controller-1.domain.local" \
--kube-controller-manager-arg="bind-address=0.0.0.0" \
--kube-proxy-arg="metrics-bind-address=0.0.0.0" \
--kube-scheduler-arg="bind-address=0.0.0.0" \
--node-taint CriticalAddonsOnly=true:NoExecute \
--advertise-address=$(hostname -I | awk '{print $2}') \
--node-ip=$(hostname -I | awk '{print $2}') \
--node-external-ip=$(hostname -I | awk '{print $1}') \
--flannel-iface=ens19 \
--tls-san=10.123.1.5 \
--tls-san=10.123.3.5 \
--tls-san=10.123.1.20 \
--tls-san=10.123.3.20 \
--tls-san=10.123.1.21 \
--tls-san=10.123.3.21 \
--tls-san=10.123.1.22 \
--tls-san=10.123.3.22 \
--cluster-init" sh -

# Weitere Control Planes hinzufügen
curl -sfL https://get.k3s.io | \
K3S_TOKEN="MY_SECRET_TOKEN" \
INSTALL_K3S_EXEC="server \
--disable servicelb \
--datastore-endpoint="postgres://k3s:MY_DB_SECRET@db-pgsql-1.domain.local:5432/k3s" \
--node-name="kube-controller-2.domain.local" \
--kube-controller-manager-arg="bind-address=0.0.0.0" \
--kube-proxy-arg="metrics-bind-address=0.0.0.0" \
--kube-scheduler-arg="bind-address=0.0.0.0" \
--node-taint CriticalAddonsOnly=true:NoExecute \
--advertise-address=$(hostname -I | awk '{print $2}') \
--node-ip=$(hostname -I | awk '{print $2}') \
--node-external-ip=$(hostname -I | awk '{print $1}') \
--flannel-iface=ens19 \
--tls-san=10.123.1.5 \
--tls-san=10.123.3.5 \
--tls-san=10.123.1.20 \
--tls-san=10.123.3.20 \
--tls-san=10.123.1.21 \
--tls-san=10.123.3.21 \
--tls-san=10.123.1.22 \
--tls-san=10.123.3.22 \
--server https://10.123.3.5:6443" sh -

# Worker Nodes hinzufügen
curl -sfL https://get.k3s.io | \
K3S_URL="https://10.123.3.5:6443" \
K3S_TOKEN="MY_SECRET_TOKEN" \
INSTALL_K3S_EXEC="--node-external-ip=$(hostname -I | awk '{print $1}') --node-name='kube-worker-1.domain.local'" \
sh -
```
