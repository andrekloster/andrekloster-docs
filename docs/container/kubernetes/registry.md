# Container Registry

[TOC]

## Einleitung
Da wir eine self-hosted Container Registry verwenden und diese Registry mit einem self-signed TLS Zertifikat
angesichert ist, muss auf jedem Kubernetes-Knoten die containerd Runtime angepasst werden

!!! info
    Es wird davon ausgegangen, dass die entsprechenden TLS Zertifikate auf dem System bereits installiert sind

```shell
mkdir -p /etc/containerd/certs.d/container-registry.domain.local
vim /etc/containerd/certs.d/container-registry.domain.local/hosts.toml
```

Folgendes ist in die `host.toml` einzutragen:
```toml
server = "https://container-registry.domain.local" # ANPASSEN!

[host."https://container-registry.domain.local"] # ANPASSEN!
capabilities = ["pull", "resolve"]
ca = ["/etc/ssl/certs/mein-ca-cert.pem"] # ANPASSEN!
```

Anschließend muss containerd neugestartet werden.

```shell
systemctl restart containerd
```

## Container Registry Secret hinterlegen
Damit Kubernetes in einem gewünschten Namespace die Container-Images herunterladen kann, muss ein Secret Objekt
mit den URL und den Logindaten erstellt werden.

```shell
kubectl create secret docker-registry harbor-secret \
  --docker-server="container-registry.domain.local" \
  --docker-username="robot\$kubernetes" \
  --docker-password="SECRET_PASSWORD" \
  -n adcell-portal-frontend
```
