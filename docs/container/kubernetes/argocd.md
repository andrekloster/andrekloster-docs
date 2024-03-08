# ArgoCD

[TOC]

## Einleitung
[ArgoCD](https://argo-cd.readthedocs.io/en/stable/) ist ein deklaratives, GitOps-basiertes Continuous Delivery Tool, 
das für die Automatisierung der Bereitstellung, Verwaltung und Wartung von Anwendungen in Kubernetes-Clustern entwickelt wurde. 
Es ermöglicht Entwicklerteams, ihre Anwendungs-Konfigurationen in Git-Repositories zu speichern, wodurch eine einzige Quelle der Wahrheit
für die Infrastruktur und Anwendungsdefinitionen geschaffen wird. ArgoCD automatisiert den Synchronisationsprozess
zwischen dem gewünschten Anwendungszustand, definiert in Git, und dem tatsächlichen Zustand im Kubernetes-Cluster,
wodurch eine konsistente und reproduzierbare Bereitstellung von Anwendungen sichergestellt wird.

## Installation
Die Installation erfolgt über die offiziellen YAML Dateien. ArgoCD ist grundliegend eine stateless Applikation.
Die persistenten Daten liegen im etcd. Jedoch entscheiden wir uns für das HA-Deployment, welches die Redis hochverfügbar macht.

```shell
export ARGOCD_VERSION='v2.10.1'
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/${ARGOCD_VERSION}/manifests/ha/install.yaml
```

## Ingress
Um auf die Weboberfläche von ArgoCD via Ingress zugreifen zu können, muss die folgende YAML deployt werden.
Wichtig ist, dass die nginx Annotation verwendet werden, um einen Redirect-Loop zu vermeiden.

```yaml
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: argocd-server-ingress
  namespace: argocd
  annotations:
    cert-manager.io/cluster-issuer: "domain-kube-issuer"
    nginx.ingress.kubernetes.io/ssl-passthrough: "true"
    nginx.ingress.kubernetes.io/backend-protocol: "HTTPS"
spec:
  ingressClassName: "nginx"
  tls:
  - hosts:
    - argocd.domain.local
    secretName: argocd-server-tls
  rules:
  - host: argocd.domain.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: argocd-server
            port:
              name: https
```

## Login
Nachdem ArgoCD erfolgreich installliert wurde, lassen wir uns zunächst das initiale Adminpasswort mithilfe 
der `argocd` Binary anzeigen.

```shell
argocd admin initial-password -n argocd
```

Um ein neues Passwort zu vergeben, öffnen wir zunächst einen **zweiten Terminal Tab** und starten ein lokales, temporäres  Port-Forwarding
zum `argocd-server` Service.

```shell
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

Anschließend können wir uns via CLI einloggen und ein neues Passwort setzen.

```shell
argocd login 127.0.0.1:8080 --insecure
```

```shell
argocd account update-password
```

Nachdem das Passwort erfolgreich angepasst wurde, kann das Port-Forwarding ausgeschaltet werden.

## Applikation definieren
Um eine Applikation deklarativ als YAML zu definieren und bei ArgoCD ausrollen, brauchen wir nur ein Secret mit
den privaten Schlüssel für die SSH Verbindung zu Git und die Custom-Resource Namens `Application`. Darin wird definiert,
in welchem Git Repository die Anwendung definiert und wie sie zu deployen ist.

```yaml
---
apiVersion: v1
kind: Secret
metadata:
  name: private-repo
  namespace: argocd
  labels:
    argocd.argoproj.io/secret-type: repository
stringData:
  type: git
  url: git@kube-proxy.domain.local:/home/git/repositories/my-app.git
  sshPrivateKey: |
    -----BEGIN OPENSSH PRIVATE KEY-----
    ...
    -----END OPENSSH PRIVATE KEY-----
```

```shell
---
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: my-app
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: default
  source:
    repoURL: git@kube-proxy.domain.local:/home/git/repositories/my-app.git
    targetRevision: main
    path: base
  destination:
    server: https://kubernetes.default.svc
    namespace: my-app
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

## Weiteren User hinzufügen
Um einen weiteren User hinzuzufügen, der ggf. weniger Rechte hat (z.B. nur App-Sync für CI)
müssen wir die ConfigMaps `argocd-cm` und `argocd-rbac-cm` bearbeiten. Dort definieren wir den neuen User
und die entsprechenden RBAC.

```shell
argocd login --insecure --grpc-web argocd.domain.local --username admin
```

```shell
kubectl edit configmap argocd-cm -n argocd
```

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  labels:
    app.kubernetes.io/name: argocd-cm
    app.kubernetes.io/part-of: argocd
  name: argocd-cm
  namespace: argocd
data:
  accounts.jenkins: login
```

```shell
argocd account list
argocd account update-password --account jenkins
```

```shell
kubectl edit configmap argocd-rbac-cm -n argocd
```

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  labels:
    app.kubernetes.io/name: argocd-rbac-cm
    app.kubernetes.io/part-of: argocd
  name: argocd-rbac-cm
  namespace: argocd
data:
  policy.csv: |
    p, role:readexecute, applications, sync, */*, allow
    g, jenkins, role:readexecute
  policy.default: role:readonly
```
