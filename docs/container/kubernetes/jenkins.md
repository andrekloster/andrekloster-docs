# Jenkins Worker in Kubernetes

[TOC]

## Einleitung
Mithilfe des [Kubernetes Plugin](https://plugins.jenkins.io/kubernetes/) für Jenkins, ist es uns möglich 
Build direkt auf dem Cluster durchzuführen. Bei jedem Build wird ein temporäre Pod erstellt. Nach Abschluss
des Build wird der Pod wieder verworfen. Somit haben wir ein skallierbares und plattformunabhängiges Jenkins-Deployment.

## Einrichtung
!!! info
    Eine detailierte Anleitung zum Konfigurieren des Plugins kann [hier](https://plugins.jenkins.io/kubernetes/#plugin-content--table-of-contents)
    entnommen werden.

### Vorbereitung
- [Plugin](https://plugins.jenkins.io/kubernetes/) installieren
- Jenkins und das Kubernetes Cluster müssen sich im selben Netzwerk befinden
- Jenkins Namespace erstellen (`kubectl create namespace jenkins`)

### RBAC und Kube-Config generieren
Wir erstellen im `jenkins` Namespace einen neuen Serviceaccount und restriktieren ihn für bestimmte Resourcen mithilfen
von RBAC.

```yaml
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: jenkins
  namespace: jenkins
---
kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: jenkins
  namespace: jenkins
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["create","delete","get","list","patch","update","watch"]
- apiGroups: [""]
  resources: ["pods/exec"]
  verbs: ["create","delete","get","list","patch","update","watch"]
- apiGroups: [""]
  resources: ["pods/log"]
  verbs: ["get","list","watch"]
- apiGroups: [""]
  resources: ["events"]
  verbs: ["watch"]
- apiGroups: [""]
  resources: ["secrets"]
  verbs: ["get"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: jenkins
  namespace: jenkins
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: jenkins
subjects:
- kind: ServiceAccount
  name: jenkins
```

Anschließend generieren wir ein Secret für den Access-Token

```yaml
---
apiVersion: v1
kind: Secret
type: kubernetes.io/service-account-token
metadata:
  name: jenkins
  namespace: jenkins
  annotations:
    kubernetes.io/service-account.name: jenkins
```

Wir lassen uns den Token angezeigen und sichern ihn zwischen.

```shell
kubectl get secret jenkins -n jenkins -o jsonpath="{.data.token}" | base64 --decode
```

Jetzt können wir eine Kube-Config bauen. Dazu benötigen wir die Adresse der Kube-API, ein base64 encodierte
Kube-API Zertifikat und den Jenkins Token.

```yaml
apiVersion: v1
kind: Config
clusters:
- name: kubernetes-domain
  cluster:
    server: https://kube-proxy.domain.local:6443
    certificate-authority-data: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSURCVENDQWUyZ0F3SUJBZ0lJRlRXQjd0TjdOc0F3RFFZSktvWklodmNOQVFFTEJRQXdGVEVUTUJFR0ExVUUKQXhNS2EzVmlaWEp1WlhSbGN6QWVGdzB5TkRBeU1qTXhNekF5TXpCYUZ3MHpOREF5TWpBeE16QTNNekJhTUJVeApFekFSQmdOVkJBTVRDbXQxWW1WeWJtVjBaWE13Z2dFaU1BMEdDU3FHU0liM0RRRUJBUVVBQTRJQkR3QXdnZ0VLCkFvSUJBUURnWkVXTzJjQW9vUU8yTUlsem9WbU1sN2FYc1RoMDV6aWhZaUZkZEF2M2grQU9uT2RULy9UbUU1ZTQKeDhEeU14ak8wYWFMUS81eWVIMy9hcWtBd1VvcFhTSFFBVGdkTkNrZkI5Zlp6VU41M1Riams5UmdnclNwSjducQpaZGVuR3Rna1laNm5JZUdlOGR2S1MrblRBV044MkFsQjM4WFJvZ1NlRXA4clUxZTAxWXJzSitKOWxONW5VY1hQCnhYVTdLYnk1eTVIQ0Y3QlE1dkpmZm9CaUNJWW8zamJOK0VTcy9VMU5kT29MTktMZXdXTXFpYTljV0d0SXRKaW0KWktWS3NqZHZVTkNJMy8wSXRYbHRwVTgwUDZtcWZMd3NPSTlESDJyTTNiejVxMGNnSXAwRTRuQTBCdnBEcmFvMgpZZ01LMnBwWU1lU3B4QVUwRWZvTlc4YklSc1ZOQWdNQkFBR2pXVEJYTUE0R0ExVWREd0VCL3dRRUF3SUNwREFQCkJnTlZIUk1CQWY4RUJUQURBUUgvTUIwR0ExVWREZ1FXQkJTV3o0Y0ZyaWFNRnNyOTZQOE8xUTdCcUlKRFZEQVYKQmdOVkhSRUVEakFNZ2dwcmRXSmxjbTVsZEdWek1BMEdDU3FHU0liM0RRRUJDd1VBQTRJQkFRQnBzbk5VbkczNApaTHhnNmpPRFQ3MUlQNWNJbHhnMzZzdHM1bldoRzIvZk1sNVZ6UzU5NlEzMFF4dGI2Y2krbnN3ZWhEd21BQTI0ClhQa1UvbmdGN0tPVm8wMmdVTmZGbTk4UXhMU2UyUlloaWk2TGFxbjkyWFlUZHNnWlRVMXBFcTRncWVrdndnOTIKcjRvZFo2VVRaa0ZPNFFZVVdlcWhSaUk3U0F2Unc1Tk5laWpxdG9QbUVrVzdOaE9ua3ZqOGRWWmtzb2RTaVFXbQpBeVE3UVN2TFg0Zmx5L05aNHNtaG5nQkNKT1dDTEJrN1FtWXZJNk11dUo1bzJDSWFHeWVKTGJCOXNjb3J2akV3CjZ0UityYS9uQjdJSG5GRUowa0d0NXRXVkY4dnd4anBjMzFqSlJ3UW1iQkFNRWN1M01ITXU4M1F0V3Zqc2oyVHgKVUN3ejhhTytwRWhkCi0tLS0tRU5EIENFUlRJRklDQVRFLS0tLS0K
contexts:
- name: jenkins
  context:
    cluster: kubernetes-domain
    user: jenkins
current-context: jenkins
users:
- name: jenkins
  user:
    token: MEIN_JENKINS_TOKEN
```

### Jenkins Plugin konfigurieren
Bevor wir das Cluster mit Jenkins verbinden, fügen wir die Kube-Config vom Jenkins Serviceaccount hinzu.
Anschließend öffnen wir die Plugin-Konfiguration, fügen eine neue Cloud hinzu und geben folgende Werte ein:

- Kubernetes URL: `https://kube-proxy.domain.local:6443`
- Kubernetes Namespace: `jenkins`
- Credentials: `kube-config-domain-jenkins`
- WebSocket: ☑️
- Jenkins URL: `https://jenkins-cloud.domain.local`

Die restlichen Felder behalten ihren Standardwert.

#### Apache WebSocket vHost konfigurieren
Damit Kubernetes mit Jenkins via WebSockets kommunizieren kann, muss ein entsprechender vHost inkl. TLS Zertifikat und 
`proxy_wstunnel` Module angelegt werden.  Hier eine Beispielkonfiguration:

```apache
<VirtualHost *:443>
  ServerName jenkins-cloud.domain.local

  ## Vhost docroot
  DocumentRoot "/var/www/worker"

  ## Directories, there should at least be a declaration for /var/www/worker

  <Directory "/var/www/worker">
    Options Indexes FollowSymLinks MultiViews
    AllowOverride None
    Require all granted
  </Directory>

  ## Logging
  ErrorLog "/var/log/apache2/worker.domain.local_cloud_443_error_ssl.log"
  ServerSignature Off
  CustomLog "/var/log/apache2/worker.domain.local_cloud_443_access_ssl.log" combined

  ## Header rules
  ## as per http://httpd.apache.org/docs/2.2/mod/mod_headers.html#header
  Header add Strict-Transport-Security "max-age=15768000"

  ## SSL directives
  SSLEngine on
  SSLCertificateFile      "/usr/share/domain-ca-certificates/domain.local_g2/crt_chained/73B7744A1882FB5A.pem"
  SSLCertificateKeyFile   "/etc/ssl/private/73B7744A1882FB5A.pem"
  SSLCACertificatePath    "/usr/share/domain-ca-certificates/domain.local_g2/ca"
  SSLCARevocationPath     "/usr/share/domain-ca-certificates/domain.local_g2/ca"
  SSLCARevocationCheck    chain
  SSLOptions +StrictRequire

  ## Custom fragment
  # Upgrade für WebSocket-Verbindungen erlauben
  RewriteEngine On
  RewriteCond %{HTTP:Upgrade} =websocket [NC]
  RewriteCond %{HTTP:Connection} Upgrade [NC]
  RewriteRule /(.*) ws://127.0.0.1:8080/$1 [P,L]
  
  ProxyPass        / http://127.0.0.1:8080/ nocanon
  ProxyPassReverse / http://127.0.0.1:8080/
  ProxyRequests    Off
  AllowEncodedSlashes NoDecode
  RequestHeader    Set X-Forwarded-Proto "https"
</VirtualHost>
```

#### Pod Template konfiguieren
Das Pod Template wird bei jedem Build verwenden, um einen temporären Worker zu generieren. In der Regel besteht
der Pod uns mehreren Containern. Einer davon ist der [Inbound Agent (jnlp)](https://hub.docker.com/r/jenkins/inbound-agent/)
der eine Verbindung zu Jenkins aufbaut. Zusätzlich können noch weitere Container Teil des Pods sein.
Zum bauen von Paketen innerhalb von Kubernetes verwenden wir z.B. [Kaniko](https://github.com/GoogleContainerTools/kaniko).

#### Optional: Custom Container Images bauen
Da unser Jenkins mit einem selbstsignierten TLS Zertifikat abgesichert sein könnte, müssen wir ein eigenes Container Image
für den Inbound Agent bauen. Wir bauen auf dem offizielen Base-Image auf und fügen unsere Root CA Zertifikate hinzu.
Um in der Jenkins-Pipline später damit `git` arbeiten zu können, fügen wir dem Image eine `ssh-config` und `git-config` mit folgendem
Inhalt hinzu:

```
# ssh-config
Host *
  UserKnownHostsFile /dev/null
  StrictHostKeyChecking no
```

```
# gitconfig
[user]
        name = Jenkins
        email = jenkins@localhost
```

Wir laden uns ebenso die Binary von [yq](https://github.com/mikefarah/yq/releases) und 
[argocd](https://github.com/argoproj/argo-cd/releases) herunter, um später in der Pipeline deployen zu können.

```Dockerfile
FROM jenkins/inbound-agent:latest-bookworm-jdk17
USER root
COPY yq_linux_amd64 /usr/local/bin/yq
COPY argocd-linux-amd64 /usr/local/bin/argocd
RUN chmod 0755 /usr/local/bin/yq /usr/local/bin/argocd
COPY meine-root-ca.pem /tmp/meine-root-ca.pem
RUN keytool -noprompt -storepass changeit -cacerts -import -file /tmp/A239DC9BFD2939FA.pem -alias Meine_Root_CA_G1
USER jenkins
RUN mkdir /home/jenkins/.ssh
COPY ssh-config /home/jenkins/.ssh/config
COPY git-config /home/jenkins/.gitconfig
```

```Dockerfile
FROM gcr.io/kaniko-project/executor:v1.21.0-debug
COPY meine-root-ca.pem /tmp/meine-root-ca.pem
```

Anschließend bauen wir das Image und laden es in unsere eigene Registry hoch.

```shell
docker build -t container-registry.domain.local/jenkins/inbound-agent:latest-bookworm-jdk17 -f Dockerfile_jnlp .
docker build -t container-registry.domain.local/jenkins/kaniko:v1.21.0-debug -f Dockerfile_kaniko .
docker login container-registry.domain.local
docker push container-registry.domain.local/jenkins/inbound-agent:latest-bookworm-jdk17
docker push container-registry.domain.local/jenkins/kaniko:v1.21.0-debug
docker logout container-registry.domain.local
```

Zusätzlich muss im `jenkins` Namespace ein Secret angelegt werden, damit das Custom Image heruntergeladen werden darf.

```shell
kubectl create secret docker-registry harbor-secret \
  --docker-server="container-registry.domain.local" \
  --docker-username="robot\$kubernetes" \
  --docker-password="GEHEIMES_PASSWORT" \
  -n jenkins --dry-run=client -o yaml | \
kubeseal \
  --controller-name=sealed-secrets \
  --controller-namespace=kube-system \
  --format yaml > harbor_secret_jenkins.yaml
kubectl apply -f harbor_secret_jenkins.yaml
```

Folgende Werte sind bei der Konfiguration relevant:

- Name: `worker-kaniko`
- Namespace: `jenkins`
- Labels: `worker-kaniko`
- Usage: `Diesen Knoten exklusiv für gebundene Projekte reservieren`

Bei Container Template tragen wir folgende Werte ein:

- Name: `kaniko`
- Docker image: `container-registry.domain.local/jenkins/kaniko:v1.21.0-debug`
- Allocate pseudo-TTY: ☑️
- Raw YAML for the Pod

Dieser YAML Patch wird benötigt, damit die Container unsere Custom Image verwenden und das Secret für die Container Registry
einbinden.

```yaml
---
apiVersion: "v1"
kind: "Pod"
metadata:
  name: "worker-test-df1dz"
  namespace: "jenkins"
spec:
  volumes:
  - name: harbor-secret
    secret:
      secretName: harbor-secret
  containers:
  - name: "jnlp"
    image: "container-registry.domain.local/jenkins/inbound-agent:latest-bookworm-jdk17"
    imagePullPolicy: Always
  - name: kaniko
    volumeMounts:
    - name: harbor-secret
      readOnly: true
      mountPath: "/kaniko/.docker/config.json"
      subPath: ".dockerconfigjson"
```

- Yaml merge strategy: `Merge`
- ImagePullSecrets: `harbor-secret`

Die restlichen Felder behalten ihren Standardwert.

### ArgoCD Login
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

### Test-Pipline
Eine Pipeline zum Testen könnte folgendermaßen aussehen:

```groovy
podTemplate (
    cloud: 'kubernetes-domain'
)
{
    node('worker-kaniko') {
        stage('Checkout') {
            // SSH-Agent mit Ihrem SSH-Key-Credential starten
            sshagent(credentials: ['my-ssh-creds']) {
                // Git-Repository klonen
                sh "git clone ssh://jenkins@git.domain.local:29418/debian/domain-docs"
                sh "git clone ssh://git@kube-proxy.domain.local:/home/git/repositories/my-app.git"
            }
        }
        stage('Build') {
            // Umgebungsvariablen definieren
            environment {
                PATH = "/busybox:/kaniko:$PATH"
            }
            // Kaniko-Container starten und das Image bauen
            container(name: 'kaniko', shell: '/busybox/sh') {
                sh '''#!/busybox/sh
                /kaniko/executor \
                  --context=/home/jenkins/agent/workspace/kube-test/domain-docs \
                  --dockerfile=/home/jenkins/agent/workspace/kube-test/domain-docs/Dockerfile \
                  --destination=container-registry.domain.local/domain/docs:latest \
                  --destination=container-registry.domain.local/domain/docs:${BUILD_NUMBER}
                '''
            }
        }
        stage('Update Version') {
            sshagent(credentials: ['my-ssh-creds']) {
                sh '''
                cd /home/jenkins/agent/workspace/kube-test/kubernetes-domain-docs
                yq eval '.spec.template.spec.containers[].image = \"\'container-registry.domain.local/domain/docs:${BUILD_NUMBER}\'\"' -i overlays/production/deployment.yaml
                git add .
                git commit -m "Update Deployment auf Version ${BUILD_NUMBER}"
                git push origin main
                '''
            }
        }
        stage('Trigger ArgoCD-Sync') {
            withCredentials([string(credentialsId: 'argocd-jenkins-password', variable: 'ARGOCD_JENKINS_PASSWORD')]) {
                sh '''
                argocd login --insecure --grpc-web argocd.domain.local --username jenkins --password ${ARGOCD_JENKINS_PASSWORD}
                argocd app sync domain-docs
                '''
            }
        }
    }
}
```
