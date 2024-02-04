# Kubernetes - K3s

## Einleitung
**Kubernetes** ist eine Open-Source-Plattform zur Automatisierung der Bereitstellung, Skalierung und Verwaltung von 
containerisierten Anwendungen. Es ermöglicht es Benutzern, Cluster von Hosts, die Container ausführen, zu verwalten 
und automatisiert viele Aspekte der Anwendungsverteilung und -wartung. **K3s** ist eine leichtgewichtige, einfach zu 
installierende Distribution von Kubernetes, die speziell für Edge-Computing, IoT und ressourcenbeschränkte Umgebungen 
entwickelt wurde. Es bietet eine vereinfachte und leichtgewichtige Implementierung von Kubernetes, die mit reduziertem 
Ressourcenbedarf auskommt, ohne dabei wesentliche Funktionen zu opfern.

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
  --tls-san=10.42.1.5 \
  --tls-san=10.42.3.5 \
  --tls-san=10.42.1.20 \
  --tls-san=10.42.3.20" sh -
```

