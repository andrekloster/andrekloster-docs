# Metrics Server

[TOC]

## Einleitung
Um via `kubectl` die Grundmetriken (CPU, RAM) vom Cluster anzeigen lassen zu können, benötigen wir
den [Metrics Server](https://github.com/kubernetes-sigs/metrics-server).

## Installation
Die Installation erfolgt via HELM mit folgendem Befehl:

```shell
export METRICS_VERSION='3.12.0'
helm repo add metrics-server https://kubernetes-sigs.github.io/metrics-server/
helm repo update
helm install metrics-server metrics-server/metrics-server \
  --version ${METRICS_VERSION} \
  --namespace kube-system
```

Damit die interne Namensauslösung funktioniert führen wir folgenden Patch beim Deployment aus:

```shell
kubectl patch deployment metrics-server \
  -n kube-system \
  --type='json' \
  -p='[{"op": "replace", "path": "/spec/template/spec/containers/0/args", "value": ["--secure-port=10250", "--cert-dir=/tmp", "--kubelet-preferred-address-types=InternalDNS,InternalIP,ExternalDNS,ExternalIP,Hostname", "--kubelet-use-node-status-port", "--metric-resolution=15s", "--kubelet-insecure-tls"]}]'
```

Wir überprüfen, ob das Deployment erfolgreich den Metric Server Pod zum laufen bekomen hat

```shell
kubectl get deployment metrics-server -n kube-system
```

Anschließend können wir mit `kubectl top` die Metriken abfragen:

```shell
kubectl top nodes
```

```shell
kubectl top pods --all-namespaces
kubectl top pods --all-namespaces --no-headers | sort -k3 -nr # CPU
kubectl top pods --all-namespaces --no-headers | sort -k4 -nr # RAM
```
