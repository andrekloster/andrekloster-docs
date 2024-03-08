# Cert Manager

[TOC]

## Einleitung
Der [Cert-Manager](https://cert-manager.io/) erweitert die Kubernetes API mit 
[Custom Resources](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/) und
ermöglicht Kubernetes die automatisierte Verwaltung von x.509 Zertifikaten. Dieses Plugin kann sowohl mit externen
Zertifikaten (z.B. Let's Encrypt) oder auch mit Self-Signed Zertifikaten arbeiten.

## Installation
Die Installation erfolgt über HELM:

```shell
export CERT_MANAGER_VERSION='v1.14.3'
helm repo add jetstack https://charts.jetstack.io --force-update
helm repo update
helm install \
  cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --version ${CERT_MANAGER_VERSION} \
  --set installCRDs=true
```

## Self-Signed Zertifikate

!!! info
    Es wird davon ausgegangen, dass ein selbstsigniertes [Root CA Zertifikat](../../pki/mini_pki.md) 
    inkl. eines unverschlüsselten Private Keys.

Zunächst muss das Root CA Zertifikat und der unverschlüsselte Private Key in base64 enkodiert werden,
damit es von der Kubernetes API ausgewertet werden kann.

### Root CA Zertifikat
```shell
cat root_ca_cert.pem | base64 -w 0
cat root_ca.key | base64 -w 0
```

Anschließend generieren wir damit im `cert-manager` Namespace ein Secret Objekt. 

```shell
vim ca-secret.yaml
```

```yaml
---
apiVersion: v1
kind: Secret
metadata:
  name: domain-kube-root-ca # ANPASSEN!
  namespace: cert-manager
type: kubernetes.io/tls
data:
  tls.crt: <CERT_BASE64_HASH> # ANPASSEN!
  tls.key: <DEC_PRIV_KEY_BASE64_HASH> # ANPASSEN!
```

```shell
kubectl apply -f ca-secret.yaml
```

### ClusterIssuer

Jetzt definieren wir die Custom Resource `ClusterIssuer`, die zukünftig die x.509 Zertifikate ausstellen soll.
Mit dem Verweis auf das Secret Objekt verknüpfen wir `ClusterIssuer` und das Root CA Zertifikat.

```shell
vim cluster-issuer.yaml
```

```yaml
---
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: domain-kube-issuer # ANPASSEN!
spec:
  ca:
    secretName: domain-kube-root-ca # ANPASSEN!
```

```shell
kubectl apply -f cluster-issuer.yaml
```
