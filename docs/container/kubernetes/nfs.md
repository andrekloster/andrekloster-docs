# NFS

[TOC]

## Einleitung
Der [Kubernetes NFS Subdir External Provisioner](https://github.com/kubernetes-sigs/nfs-subdir-external-provisioner)
ist ein Plugin, welches es uns erleichter PersistentVolumeClaim via NFS zu erstellen. Beim Erstellen wird
auf der NFS-Freigabe für den Claim ein Unterordner erstellt.

## Installation
Die Installation erfolgt über HELM. Es werden Parameter mitgegeben, die die neue Storage Class konfiguriert.

!!! info
    Es wird davon ausgegangen, dass auf dem SHARE eine NFS-Freigabe unter `/mnt/DATA/kubernetes` vorhanden ist.

```shell
export NFS_PROVISIONER_VERSION='4.0.18'
helm repo add nfs-subdir-external-provisioner https://kubernetes-sigs.github.io/nfs-subdir-external-provisioner/
helm repo update
helm install nfs-subdir-external-provisioner nfs-subdir-external-provisioner/nfs-subdir-external-provisioner \
  --set nfs.server=kube-nfs.domain.local \
  --set nfs.path=/mnt/DATA/kubernetes  \
  --set storageClass.name=share-nfs \
  --set nfs.reclaimPolicy=Delete \
  --set storageClass.archiveOnDelete=false \
  --namespace nfs --create-namespace \
  --version ${NFS_PROVISIONER_VERSION}
```

Anschließend können wir testen, ob wir einen PersistentVolumeClaim mit der NFS Storage Class erstellen können

```yaml
---
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: test-claim
  annotations:
    nfs.io/storage-path: "/mnt/DATA/kubernetes"
spec:
  storageClassName: share-nfs
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 1Mi
```

```yaml
kubectl get sc
kubectl apply -f my-pvc.yaml
kubectl get pv,pvc
```
