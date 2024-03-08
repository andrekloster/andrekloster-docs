# Sealed Secrets

[TOC]

## Einleitung
Sealed Secrets sind ein Werkzeug für Kubernetes, das es Benutzern ermöglicht, ihre Geheimnisse (Secrets) sicher 
zu speichern und zu verwalten. Sie ermöglichen es, Geheimnisse wie Passwörter, Zertifikate oder Schlüssel in 
einer verschlüsselten Form im Git-Repository zu speichern. Ein Sealed Secret wird vom Sealed Secrets Controller 
im Kubernetes-Cluster entschlüsselt und als normales Kubernetes Secret bereitgestellt, das nur innerhalb des Clusters zugänglich ist. 
Dadurch können Entwickler ihre Konfigurationen und Geheimnisse sicher verwalten, ohne sensible Informationen offenlegen zu müssen.

## Installation
Die Installation erfolgt über HELM:

```shell
export SEALED_SECRETS_VERSION='2.15.0'
helm repo add sealed-secrets https://bitnami-labs.github.io/sealed-secrets
helm repo update
helm install sealed-secrets sealed-secrets/sealed-secrets \
  --namespace kube-system \
  --version ${SEALED_SECRETS_VERSION}
```

Um mit dem Sealed Secrets Controller zu kommunizieren und um ein Secret zu verschlüsseln wird die Binary
[kubeseal](https://github.com/bitnami-labs/sealed-secrets/releases/) benötigt. Am Terminal erstellen wir ein Secret
und reichen es an `kubeseal` weiter, der das verschlüsselt und als YAML Datei ausgibt.

```shell
kubectl create secret docker-registry harbor-secret \
  --docker-server="container-registry.domain.local" \
  --docker-username="robot\$kubernetes" \
  --docker-password="SECRET_PASSWORD" \
  -n adcell-portal-frontend --dry-run=client -o yaml | \
kubeseal \
  --controller-name=sealed-secrets \
  --controller-namespace=kube-system \
  --format yaml > harbor_secret.yaml
```

## Backup
Das Zertifikat und der private Schlüssel mit dem  Sealed Secrets Controller arbeitet, befindet sich ausschließlich
innerhalb des Cluster. Es ist sehr ratsam diese Dateien zusätzlich wegzusichern. Als Administrator können
die Dateien via `kubectl` angezeigt werden.

```shell
# Zeige Zertifikat an
kubectl -n kube-system get secret \
  -l sealedsecrets.bitnami.com/sealed-secrets-key=active \
  -o jsonpath='{.items[].data.tls\.crt}' | base64 -d
# Zeige privaten Schlüssel an
kubectl -n kube-system get secret \
  -l sealedsecrets.bitnami.com/sealed-secrets-key=active \
  -o jsonpath='{.items[].data.tls\.key}' | base64 -d
```