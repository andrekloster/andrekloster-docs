# Einfaches Deployment inkl. Ingress
## Einleitung
In diesem Kapitell wird beschrieben, wie eine einfache Python Flask Applikation aus meinem
[Beispiel-Repository](https://github.com/andrekloster/test-color-app) innerhalb von Kubernetes deployt werden kann.
Zusätzlich wird diese Applikation mithilfe von Ingress über Port 80 erreichbar sein.

## Voraussetzungen
Es wird vorausgesetzt, dass ein Kubernetes Cluster inkl. Ingress-Controller und externer LoadBalancer IP vorhanden ist.
Die Installation eines solchen Cluster wird [hier](setup_k3s.md) beschrieben. Ebenso wird davon ausgegangen, dass 
die Beispiel-Applikation als Container Image gebaut und z.B. in der
[Docker Hub Registry veröffentlich wurde](https://hub.docker.com/repository/docker/5470/test-color-app/general).

## Manifests
Zunächst klonen wir das Git Repository von der Beispiel-Applikation

```shell
git clone https://github.com/andrekloster/test-color-app.git
```

Anschließend untersuchen wir den Inhalt vom `kubernetes` Ordner. Dort befinden sich folgende Dateien.

### Namespace
Mithilfe eines Namespaces können wir eine logische Trennung für all unsere Kubernetes Objekte definieren.
Der Namespace muss als erstes Objekt erstellt werden. Sobald der Namespace vorhanden ist, können die restlichen
Objekte ausgerollt werden.

```yaml
---
apiVersion: v1
kind: Namespace
metadata:
  name: test-color-app
spec: {}
status: {}
```

### Deployment
Das Deployment `test-color-app-deployment` definiert, dass `2 Replicas` eines Pods mit dem Image `5470/test-color-app:latest` gestartet werden sollen.
Der interne Containerport der Pods ist `5000`.

```yaml
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: test-color-app-deployment
  namespace: test-color-app 
spec:
  replicas: 2
  selector:
    matchLabels:
      app: test-color-app
  template:
    metadata:
      labels:
        app: test-color-app
    spec:
      containers:
      - name: test-color-app
        image: 5470/test-color-app:latest
        ports:
        - containerPort: 5000
```

### Service
Damit das Deployment zunächst innerhalb des Cluster eine DNS-Auflösung erhält, wird der Service `test-color-app-service`
erstellt. Dieser Service verbindet sich mithilfe des Labels `app: test-color-app` via TCP Port 5000 mit dem Pod
`test-color-app` und stellt für den internen Betrieb des Cluster eine Verbindung auf.

```yaml
---
apiVersion: v1
kind: Service
metadata:
  name: test-color-app-service
  namespace: test-color-app 
spec:
  selector:
    app: test-color-app
  ports:
    - protocol: TCP
      port: 5000
      targetPort: 5000
```

### Ingress
Ein Ingress Objekt ermöglicht es eine Applikation via Loadbalancer IP und eines vHosts die Applikation von außerhalb des
Cluster aufrufbar zu sein. Dabei wird der [Ingress Controller](https://kubernetes.io/docs/concepts/services-networking/ingress-controllers/)
und der entsprechende [Service](#service) miteiner verkũpft.

```yaml
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: test-color-app-ingress
  namespace: test-color-app
spec:
  rules:
    - host: testcolorapp.mydomain.com # ANPASSEN!
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: test-color-app-service
                port:
                  number: 5000
```

## Installation
Mit den folgenden Befehlen erstellen wir unserern Namespace und installieren alle Kubernetes Objekte aus den Manifests

```shell
cd test-color-app
```

```shell
kubectl apply -f kubernetes/namespace.yaml
```

```
kubectl apply -f kubernetes/deployment.yaml
kubectl apply -f kubernetes/service.yaml
kubectl apply -f kubernetes/ingress.yaml
```

Anschließend können wir uns alle Objekte im Namespace anzeigen lassen

```shell
kubectl get all,ingress -n test-color-app
```

```shell
NAME                                             READY   STATUS    RESTARTS   AGE
pod/test-color-app-deployment-6d6d6cdcd5-mk6mj   1/1     Running   0          48m
pod/test-color-app-deployment-6d6d6cdcd5-5vgg2   1/1     Running   0          48m

NAME                             TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
service/test-color-app-service   ClusterIP   10.43.106.191   <none>        5000/TCP   48m

NAME                                        READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/test-color-app-deployment   2/2     2            2           48m

NAME                                                   DESIRED   CURRENT   READY   AGE
replicaset.apps/test-color-app-deployment-6d6d6cdcd5   2         2         2       48m

NAME                                               CLASS     HOSTS                       ADDRESS          PORTS   AGE
ingress.networking.k8s.io/test-color-app-ingress   traefik   testcolorapp.mydomain.com   192.168.1.241   80      48m
```

Nun sollte die Applikation unter der gewünschten Domain erreichbar sein 

```shell
curl http://testcolorapp.mydomain.com
```

```html
<!DOCTYPE html>
<html>
<head>
    <title>Test color app</title>
</head>
<body>
    <h1 style="color: red;">Red</h1>
</body>
</html>
```
