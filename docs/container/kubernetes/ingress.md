# Ingress

[TOC]

## Einleitung
!!! info
    Es wird davon ausgegangen, dass das Cluster externe IPs dynamisch vergeben kann. Beispielsweise via [MetalLB](metallb.md).

Um eine Applikation von außerhalb des Kubernetes Cluster zugänglich zu machen, wird in der Regel ein 
[Ingress](https://kubernetes.io/docs/concepts/services-networking/ingress/) Objekt erstellt, 
das den Traffic zu einem [LoadBalancer Service](https://kubernetes.io/docs/concepts/services-networking/service/#loadbalancer)
weiterleitet. Sobald das Ingress Objekt erstellt wurde, erhält es von Kubernetes eine externe IP Adresse.

## Installation
Um ein Ingress Objekt erstellen zu können, muss zuvor ein [Ingress Controller](https://kubernetes.io/docs/concepts/services-networking/ingress-controllers/)
installiert werden. In der Kubernetes Dokumentation gibt es Verweise auf unterschiedliche Provider, die eine Implementierung
eines Ingress Controllers bereitstellen

### NGINX Ingress Controller
Für die Installation des weit verbreiteten [Ingress Nginx Controller](https://kubernetes.github.io/ingress-nginx/)
führen wir folgenden HELM Befehl aus:

```shell
export INGRESS_NGINX_VERSION='4.9.1'
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update
helm install ingress-nginx ingress-nginx \
  --repo https://kubernetes.github.io/ingress-nginx \
  --namespace ingress-nginx --create-namespace \
  --version ${INGRESS_NGINX_VERSION}
```
