# MetalLB

[TOC]

## Einleitung
[MetalLB](https://metallb.universe.tf/) ermöglicht die Vergabe externe IPs innerhalb eines Kubernetes Cluster.
Es ist mit einem DHCP für Kubernetes vergleichbar. Innerhalb einer Cloud-Umgebung (AWS, Azure, GCP) ist MetalLB nicht notwendig, 
da die Cloud für die dynamische Vergabe externen IPs zuständig ist. Bei einem Bare-Metal-Cluster muss die Funktion
jedoch manuell konfiguriert werden.

## Installation
Wir installieren MetalLB via HELM:

**HELM**
```shell
export METALLB_VERSION='v0.14.3'
helm repo add metallb https://metallb.github.io/metallb
helm install metallb metallb/metallb \
  -n metallb-system \
  --create-namespace \
  --version ${METALLB_VERSION}
```

## Konfiguration
Anschließend muss MetalLB [konfiguriert](https://metallb.universe.tf/configuration/) werden. Für unsere Fälle
reicht die `Layer 2 Configuration`, in der wir den Adress-Pool definieren, aus dem die IPs dynamisch allokiert werden.

- `metallb-config.yaml`
```yaml
---
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: domain-cloud-ip-adress-pool # ANPASSEN!
  namespace: metallb-system
spec:
  addresses:
  - 10.192.13.100-10.192.13.250 # ANPASSEN!
---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: domain-cloud-l2-advertisement # ANPASSEN!
  namespace: metallb-system
```
