# Installation mit kubeadm

[TOC]

<figure markdown>
  ![HA Cluster](/images/kubernetes/ha-cluster.svg)
</figure>

## Einleitung
In diesem Kapitel wird beschrieben wir ein hochverfügbares Kubernetes Cluster mithilfe von `kubeadm` erstellen.
Für genaue Details ist es ratsam die [Dokumentation](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/) zu lesen.
In unserem Fall erstellen wir ein 
[HA-Cluster mit einem stacked (internen) etcd](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/high-availability/).

## Vorbereitungen
### Proxy
Wie in der oberen Grafik dargestellt, benötigen wir einen Layer-4 Loadbalancer, der alle API-Anfragen zum Port 6443
an alle Control-Planes des Cluster verteilt. [HAProxy](https://www.haproxy.org/) eignet sich sehr gut für diese Aufgabe.

Eine beispielhafte HA-Proxy Konfiguration (`/etc/haproxy/haproxy.cfg`) könnte wie folgt aussehen:

```
global
    log /dev/log    local0
    log /dev/log    local1 notice
    chroot /var/lib/haproxy
    stats socket /run/haproxy/admin.sock mode 660 level admin
    stats timeout 30s
    user haproxy
    group haproxy
    daemon
    
    # Default SSL material locations
    ca-base /etc/ssl/certs
    crt-base /etc/ssl/private
    
    # See: https://ssl-config.mozilla.org/#server=haproxy&server-version=2.0.3&config=intermediate
    ssl-default-bind-ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:DHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384
    ssl-default-bind-ciphersuites TLS_AES_128_GCM_SHA256:TLS_AES_256_GCM_SHA384:TLS_CHACHA20_POLY1305_SHA256
    ssl-default-bind-options ssl-min-ver TLSv1.2 no-tls-tickets

defaults
    log     global
    mode    http
    option  httplog
    option  dontlognull
    timeout connect 5000
    timeout client  50000
    timeout server  50000
    errorfile 400 /etc/haproxy/errors/400.http
    errorfile 403 /etc/haproxy/errors/403.http
    errorfile 408 /etc/haproxy/errors/408.http
    errorfile 500 /etc/haproxy/errors/500.http
    errorfile 502 /etc/haproxy/errors/502.http
    errorfile 503 /etc/haproxy/errors/503.http
    errorfile 504 /etc/haproxy/errors/504.http

frontend kubernetes_api
    bind 10.192.13.6:6443
    mode tcp
    option tcplog
    default_backend kubernetes_api_backend

backend kubernetes_api_backend
    mode tcp
    balance roundrobin
    option tcp-check
    server kube-controller-1 10.192.13.7:6443 check
    server kube-controller-2 10.192.13.8:6443 check
    server kube-controller-3 10.192.13.9:6443 check
```

### DNS
Sowohl Control-Planes, Worker-Nodes, als auch der Proxy müssen im DNS eingetragen sein.
Ebenso müssen auf allen Nodes die `/etc/resolv.conf` wie folgt aussehen:

```
domain domain.local
nameserver 10.192.13.1
```

Es ist wichtig, dass `search` **nicht** vorhanden ist, da es sonst zu Probleme bei der Namensauflösung innerhalb der
[Cilium Tests](#cilium-tests) kommt.

### Hardware und System
- Jede Maschine sollte mindestens 2 CPU Kerne und 2 GB RAM besitzen
- Auf allen Mschinen muss der Swap ausgeschaltet sein, damit Kubelet funktionsfähig ist
- Die GRUB Konfiguration muss angepasst werden, da sonst nicht mehrere Control-Plane dem Cluster hinzugefügt werden können.
```shell
sed '/GRUB_CMDLINE_LINUX_DEFAULT=/s|.*|GRUB_CMDLINE_LINUX_DEFAULT="systemd.unified_cgroup_hierarchy=0"|g;' /etc/default/grub
```
- Das Kernelmodule `overlay` und `br_netfilter` müssen aktiv sein
```shell
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

modprobe overlay
modprobe br_netfilter
```
- Die Kernelparamter `bridge.bridge-nf-call-iptables`, `bridge.bridge-nf-call-ip6tables` und `ipv4.ip_forward` müssen aktiv sein
```shell
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward = 1
EOF

sysctl --system
```

### Container Runtime Enginge (containerd) einrichten
Wir wählen [containerd](https://containerd.io/) als unserer Runtime Engine aus. Das Debian-Paket dafür befindet sich
im [APT-Repository von Docker](https://download.docker.com/linux/debian/).

```shell
apt-get update && apt-get install containerd
```

Anschließend generieren wir eine neue, umfangreiche Default-Konfiguration

```shell
containerd config default>/etc/containerd/config.toml
```

Da Debian verwendet standardmäßig die systemd cgroup driver. Somit muss laut
[Kubernetes Dokumentation](https://kubernetes.io/docs/setup/production-environment/container-runtimes/#containerd)
die containerd.toml angepasst werden.

```shell
sed -i 's,SystemdCgroup = false,SystemdCgroup = true,g' /etc/containerd/config.toml
```

Anschließend muss containerd neugestartet werden

```shell
systemctl daemon-reload
systemctl enable containerd
systemctl restart containerd
```

### Debian-Pakete für Kubernetes installieren
Um mit `kubeadm` das Cluster einrichten zu können, benötigen wir auf allen Maschinen die entsprechenden Debian-Pakete für Kubernetes,
aus dem offiziellen APT-Repository. Genaueres ist in der 
[Dokumentation](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/#installing-kubeadm-kubelet-and-kubectl)
beschrieben.

```shell
apt-get update && apt-get install kubeadm kubelet kubectl
```

### Optional: Package Manager HELM installieren
Zur Installation und Verwaltung externe Softwarepakete für Kubernetes wird der Package Manager [HELM](https://helm.sh/)
verwendet. Die Installation von HELM besteht nur aus einer einzigen [Binary](https://helm.sh/docs/intro/install/).
HELM kann von jedem System aus genutzt werden, das Vollzugriff auf das Cluster hat.

### Core-Images herunterladen
Bevor wir das Cluster initialisieren, laden wir uns im Vorfeld auf allen Maschinen die entsprechenden Images herunter.

```shell
kubeadm config images pull
```

## Cluster initialisieren
Um das Cluster zu initialisieren, führen wir auf dem ersten Control-Plane folgenden Befehl aus.
Dabei werden sämtliche Konfiguration und TLS-Zertifikate für das Cluster generiert. Nachdem die Initialisierung fertig
ist, wird als Output der Join-Befehl für weitere Control-Plane und die Worker-Nodes angezeigt

```shell
kubeadm init \
  --control-plane-endpoint "kube-proxy.domain.local:6443" \
  --upload-certs \
  --skip-phases=addon/kube-proxy \
  --apiserver-advertise-address=10.192.13.7 \
  --apiserver-cert-extra-sans="kube-proxy.domain.local,kube-proxy-1.domain.local,kube-proxy-1-cloud.domain.local,kube-controller-1.domain.local,kube-controller-2.domain.local,kube-controller-3.domain.local,kube-controller-1-cloud.domain.local,kube-controller-2-cloud.domain.local,kube-controller-3-cloud.domain.local"
```

- `--control-plane-endpoint` verweist auf den DNS Eintrag des [Proxies](#proxy)
- `--upload-certs` automatisiert die Einrichtung der TLS-Zertifikate für weitere Nodes
- `--skip-phases=addon/kube-proxy` ignoriert die Einrichtung vom Standard Kube-Proxy. Das ist für unser [CNI](#container-network-interface-cni) notwendig
- `--apiserver-advertise-address` Bindet den API Server an das gewünschte Netzwerkinterface
- `--apiserver-cert-extra-sans` Fügt den Kubernetes TLS Zertifikaten weitere SANS Adresse hinzu

Abschließend kopieren wir die Kube-Config in das Arbeitsverzeichnis des Benutzers und überprüfen mit `kubectl`
den aktuellen Status vom ersten Node. Der erste Node sollte angezeigt werden. Jedoch ist der Status aus `NotReady`,
da das [Container Network Interface](#container-network-interface-cni) konfiguriert werden muss.

```shell
mkdir -p $HOME/.kube
cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
chown $(id -u):$(id -g) $HOME/.kube/config
```

```shell
kubectl get nodes -o wide
```

```shell
kubectl get all --all-namespaces
```

### Control Planes
Um weitere Control-Plane-Nodes zum Cluster hinzuzufügen, führen wir den entsprechend Join-Befehl, den wir nach der [Initialisierung](#cluster-initialisieren)
angezeigt bekommen haben aus. Diesen Befehl führen wir auf den restlichen Controle-Plane-Nodes aus.

```shell
kubeadm join kube-proxy.domain.local:6443 --token z7nfiy.0f9ed5v2egqcid09 \
	--discovery-token-ca-cert-hash sha256:ef862226baab3360a0f1475d8488107e55f231fb29b27bd2795bdb44e6ddd772 \
	--control-plane --certificate-key 056225c2ba7becd66a311813c75e15428e34538755e0941cf7b8a6300824d29a
```

Anschließend kopieren wir auch hier die Kube-Config ins Arbeitsverzeichnis und überprüfen den Status des Clusters.

```shell
mkdir -p $HOME/.kube
cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
chown $(id -u):$(id -g) $HOME/.kube/config
```

```shell
kubectl get nodes -o wide
```

```shell
kubectl get all --all-namespaces
```

### Worker Nodes
Um weitere Worker-Nodes zum Cluster hinzuzufügen, führen wir den entsprechend Join-Befehl, den wir nach der [Initialisierung](#cluster-initialisieren)
angezeigt bekommen haben aus. Diesen Befehl führen wir auf allen Worker-Nodes aus.

```shell
kubeadm join kube-proxy.domain.local:6443 --token z7nfiy.0f9ed5v2egqcid09 \
	--discovery-token-ca-cert-hash sha256:ef862226baab3360a0f1475d8488107e55f231fb29b27bd2795bdb44e6ddd772
```

Nun sollten alle Maschinen teil des Cluster sein. Der Status der Nodes bleibt weiterhin `NotReady`, solange das [CNI](#container-network-interface-cni)
nicht installiert ist.

### Container Network Interface (CNI)
Damit das Komponenten des Cluster miteinander kommunizieren können, benötigt Kubernetes ein internes Netzwerk, 
welches von einem Container Network Interface Plugin gesteuert wird. Wir entscheiden uns für das CNI [Cilium](https://cilium.io/).
Dieses Plugin hat einen großen Funktionsumfang, unterstützt [Network Policies](https://kubernetes.io/docs/concepts/services-networking/network-policies/)
und arbeitet im Hintergrund mit [ePBF](https://cilium.io/blog/2020/11/10/ebpf-future-of-networking/).

Zur Installation verwenden wir den Package Manager [HELM](#package-manager-helm-installieren).

```shell
helm repo add cilium https://helm.cilium.io/

export API_SERVER_IP='kube-proxy.domain.local'
export API_SERVER_PORT='6443'
export CILIUM_VERION='1.15.1' # ANPASSEN!

helm install cilium cilium/cilium --version ${CILIUM_VERION} \
    --namespace kube-system \
    --set kubeProxyReplacement=true \
    --set k8sServiceHost=${API_SERVER_IP} \
    --set k8sServicePort=${API_SERVER_PORT}
```

#### Cilium Tests
Sobald Cilium eingerichtet ist, installieren wir auf dem `kube-proxy` die `cilium` und `hubble` Binary
([siehe Dokumenation](https://docs.cilium.io/en/stable/installation/k8s-install-kubeadm/)).

Mit dem folgende Befehl können wir uns den Status alles Pods und Operator angezeigen lassen.

```shell
cilium status --wait
```

Um später eine grafische Übersicht vom Netzwerk zu erhalten und erfolgreich die Tests zu durchlaufen,
aktivieren wir die Ciliums Web UI von Names
[Hubble](https://docs.cilium.io/en/stable/overview/intro/).

```shell
export CILIUM_VERION='1.15.1' # ANPASSEN!
helm upgrade cilium cilium/cilium --version ${CILIUM_VERION} \
   --namespace kube-system \
   --reuse-values \
   --set hubble.relay.enabled=true \
   --set hubble.ui.enabled=true
```

Auf dem `kube-proxy` schalten wir zunächst im Hintergrund ein Port-Forwarding zum Hubble an, um alle Tests vollständig
erfolgreich durchzuführen.

```shell
cilium hubble port-forward&
```

Anschließend führen wir einen Connectivity-Test aus. Diese kann bis zu 15-20 Minuten dauern.

```shell
cilium connectivity test
```

Sobald die Test erfolgreich durchgelaufen sind, kann der Port-Forward im Hintergrund geschlossen werden.
