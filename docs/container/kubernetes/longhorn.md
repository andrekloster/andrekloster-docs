# Longhorn

[TOC]

## Einleitung
Longhorn ist ein Open-Source-Projekt, das eine verteilte Speicherplattform für Kubernetes bereitstellt.
Es wurde entwickelt, um persistente Speicherlösungen in einer Cloud-nativen Umgebung zu vereinfachen,
indem es hochverfügbaren und skalierbaren Blockspeicher für Kubernetes-Pods anbietet. Longhorn ermöglicht es Benutzern,
persistenten Speicher nahtlos zu verwalten, zu skalieren und automatisch zu reparieren, was es ideal für den Einsatz
in Produktionsumgebungen mit Kubernetes macht. Es bietet auch Funktionen wie Snapshot- und Backup-Mechanismen,
die die Datenverwaltung und -wiederherstellung vereinfachen.

## Vorbereitungen
Im folgenden werden die wichtigsten Schritte zusammengefasst, um Longhorn zu installieren.
Eine genauere Erkläre ist der [Dokumentation](https://longhorn.io/docs/1.6.0/deploy/important-notes/) zu entnehmen.

Auf allen Data-Node (i.d.R. Worker) muss `iscsi` und `nfs` installiert sein, damit Longhorn funktionsfähig ist.

```shell
apt-get update && apt-get install open-iscsi nfs-common
```

```shell
echo 'iscsi_tcp' >> /etc/modules-load.d/k8s.conf
modprobe iscsi_tcp
```

Anschließend können wir mit folgendem Skript testen, ob unsere System für Longhorn vorbereitet sind.

```shell
export LONGHORN_VERSION='1.6.0'
curl -sSfL https://raw.githubusercontent.com/longhorn/longhorn/${LONGHORN_VERSION}/scripts/environment_check.sh | bash
```

## Installation
Die Installation wird mithilfe von HELM durchgeführt.

```shell
export LONGHORN_VERSION='1.6.0'
helm repo add longhorn https://charts.longhorn.io
helm repo update
helm install longhorn longhorn/longhorn \
  --namespace longhorn-system \
  --create-namespace \
  --version ${LONGHORN_VERSION}
```

### Mehrere Storage Classes erstellen
Nach der Grundinstallation von Longhorn wird die Storage Class Namens `longhorn` erstellt. Diese Storage Class
hat als Standardeinstellung `numberOfReplicas: "3"`. Das heißt, dass zu jedem Volume zusätzlich zwei weitere Kopien auf
unterschiedlichen Cluster-Nodes erstellt werden. In einigen Fällen ist es jedoch eine andere Anzahl von Kopien erwünscht
(oder gar keine Kopien). Deshalb definieren wir weitere Longhorn Storage Classes und passen den Wert `numberOfReplicas`
entsprechend an.

```yaml
---
allowVolumeExpansion: true
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  annotations:
    longhorn.io/last-applied-configmap: |
      kind: StorageClass
      apiVersion: storage.k8s.io/v1
      metadata:
        name: longhorn-standalone
        annotations:
          storageclass.kubernetes.io/is-default-class: "true"
      provisioner: driver.longhorn.io
      allowVolumeExpansion: true
      reclaimPolicy: "Delete"
      volumeBindingMode: Immediate
      parameters:
        numberOfReplicas: "1"
        staleReplicaTimeout: "30"
        fromBackup: ""
        fsType: "ext4"
        dataLocality: "disabled"
    storageclass.kubernetes.io/is-default-class: "false"
  name: longhorn-standalone
parameters:
  dataLocality: disabled
  fromBackup: ""
  fsType: ext4
  numberOfReplicas: "1"
  staleReplicaTimeout: "30"
provisioner: driver.longhorn.io
reclaimPolicy: Delete
volumeBindingMode: Immediate
---
allowVolumeExpansion: true
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  annotations:
    longhorn.io/last-applied-configmap: |
      kind: StorageClass
      apiVersion: storage.k8s.io/v1
      metadata:
        name: longhorn-duplicate
        annotations:
          storageclass.kubernetes.io/is-default-class: "true"
      provisioner: driver.longhorn.io
      allowVolumeExpansion: true
      reclaimPolicy: "Delete"
      volumeBindingMode: Immediate
      parameters:
        numberOfReplicas: "2"
        staleReplicaTimeout: "30"
        fromBackup: ""
        fsType: "ext4"
        dataLocality: "disabled"
    storageclass.kubernetes.io/is-default-class: "false"
  name: longhorn-duplicate
parameters:
  dataLocality: disabled
  fromBackup: ""
  fsType: ext4
  numberOfReplicas: "2"
  staleReplicaTimeout: "30"
provisioner: driver.longhorn.io
reclaimPolicy: Delete
volumeBindingMode: Immediate
```

## Ingress
Sobald Longhorn erfolgreich installiert ist, steht eine Web UI zur Verfügung. Jedoch hat diese UI kein natives Login
und muss somit z.B. via Basic Authentication am Ingress abgesichert werden.

Zunächst erstellen wir ein encodiertes Login in eine Datei Namen `auth` und generieren damit ein Kuberntes Secret
im `longhorn-system` Namespace.

```shell
USER=admin; PASSWORD=GEHEIMES_PASSWORT; echo "${USER}:$(openssl passwd -stdin -apr1 <<< ${PASSWORD})" >> auth
```

```shell
kubectl -n longhorn-system create secret generic basic-auth --from-file=auth
```

Anschließend verweisen wir in unserem Ingress Objekt auf das auth-Secret und aktviere zugleich die die Basic Authentication.

```shell
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: longhorn-frontend-ingress
  namespace: longhorn-system
  annotations:
    nginx.ingress.kubernetes.io/auth-type: basic
    nginx.ingress.kubernetes.io/auth-secret: basic-auth
    nginx.ingress.kubernetes.io/auth-realm: 'Authentication Required '
    cert-manager.io/cluster-issuer: "domain-kube-issuer"
    nginx.ingress.kubernetes.io/proxy-body-size: 10000m
spec:
  ingressClassName: "nginx"
  tls:
  - hosts:
    - longhorn.domain.local
    secretName: longhorn-frontend-tls
  rules:
  - host: longhorn.domain.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: longhorn-frontend
            port:
              name: http
```

Anschließend überprüfen wir, ob das Ingress Objekt inkl TLS-Zertifikat erstellt wurde. Es kann 1-2 Minuten dauern,
bis das Ingress Objekt eine Öffentliche IP Adresse zugewiesen bekommt.

```shell
kubectl apply -f ingress.yaml
kubectl get ingress,cert -n longhorn-system
```
