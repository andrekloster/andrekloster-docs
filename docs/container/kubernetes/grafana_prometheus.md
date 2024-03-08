# Grafana-Prometheus-Stack

[TOC]

## Einleitung
Der [Grafana-Prometheus-Stack(kube-prometheus-stack)](https://github.com/prometheus-community/helm-charts/tree/main/charts/kube-prometheus-stack)
ist eine Ansammlung von HELM Charts, die im Zusammenspiel alles bereitsstellen, um mit Grafana das Kubernetes-Cluster
zu überwachen. Die mitinstallierten Grafana-Dashboard sind ideal für Kubernetes voreingerichtet.

## Installation
Die Installation erfolgt über HELM. Wir verwenden zusätzlich noch eine `values.yaml`, um die Dienste besser zu konfigurieren
(Ingress, Grfana Admin-Passwort, Persistent Storages, Retention Size).

```shell
vim values.yaml
```

```yaml
---
grafana:
  adminPassword: GEHEIMES_GRAFANA_PASSWORT # ANPASSEN!
  persistence:
    enabled: true
    type: sts
    storageClassName: "longhorn-duplicate" # ANPASSEN!
    accessModes:
      - ReadWriteOnce
    size: 5Gi
    finalizers:
      - kubernetes.io/pvc-protection
  ingress:
    enabled: true
    ingressClassName: nginx
    annotations:
      cert-manager.io/cluster-issuer: "domain-kube-issuer"
    hosts: ['grafana-kube.domain.local']
    path: /
    tls:
    - secretName: grafana-prometheus-cert
      hosts:
      - grafana-kube.domain.local
prometheus:
  prometheusSpec:
    retentionSize: "14GiB"
    storageSpec:
      volumeClaimTemplate:
        spec:
          storageClassName: longhorn-duplicate # ANPASSEN!
          accessModes: ["ReadWriteOnce"]
          resources:
            requests:
              storage: 15Gi
alertmanager:
  alertmanagerSpec:
      storage:
        volumeClaimTemplate:
          spec:
            storageClassName: longhorn-duplicate # ANPASSEN!
            accessModes: ["ReadWriteOnce"]
            resources:
                requests:
                  storage: 10Gi
```

```shell
export GRAFANA_PROMETHEUS_VERSION='56.14.0'
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
helm install grafana-prometheus prometheus-community/kube-prometheus-stack \
  --namespace monitoring --create-namespace \
  --values values.yaml
  --version ${GRAFANA_PROMETHEUS_VERSION}
```

