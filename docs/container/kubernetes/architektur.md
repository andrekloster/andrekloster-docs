# Architektur

[TOC]

<figure markdown>
  ![Cluster Architektur](/images/kubernetes/cluster-architecture.svg)
</figure>

## Control Plane
Die Control Plane in Kubernetes koordiniert das Cluster, indem sie Pods plant, auf Ereignisse reagiert und Netzwerke konfiguriert. 
Sie besteht aus dem API-Server, Scheduler, Controller Manager und etcd für die Datenspeicherung, ergänzt durch 
den Cloud Controller Manager in Cloud-Umgebungen. Diese Komponenten sichern den gewünschten Cluster-Zustand, ermöglichen Skalierung, Selbstheilung und Updates.

####  API Server
Der [API-Server](https://kubernetes.io/docs/concepts/overview/components/#kube-apiserver)
in Kubernetes ist die zentrale Schnittstelle, durch die alle internen und externen Kommunikationen verlaufen. 
Er verarbeitet REST-Anfragen, validiert sie und aktualisiert den Zustand der Kubernetes-Objekte in der etcd-Datenbank. 
Er dient als Gateway für das Cluster-Management und ermöglicht die Interaktion mit dem Cluster über die kubectl-Befehlszeilenschnittstelle, 
die Kubernetes API oder andere externe Tools.

#### etcd
[etcd](https://kubernetes.io/docs/concepts/overview/components/#etcd)
ist eine hochverfügbare Schlüsselwert-Datenbank, die als die primäre Datenspeicherung für den Zustand des gesamten Kubernetes-Clusters dient. 
Es speichert die Konfiguration und den Zustand der Cluster-Ressourcen, sodass der API-Server Änderungen abfragen und aktualisieren kann. 
Kubernetes lässt sich sowohl mit einem internen, als auch mit einem externen etcd betreiben. 

#### Scheduler
Der [Scheduler](https://kubernetes.io/docs/concepts/overview/components/#kube-controller-manager)
in Kubernetes ist für die Zuweisung von Pods zu Nodes im Cluster verantwortlich. Er trifft Entscheidungen 
basierend auf verschiedenen Kriterien wie Ressourcenanforderungen, Hardware-/Software-/Policy-Einschränkungen, Affinitäten, 
Anti-Affinitäten und anderen. Nachdem der API-Server neue Pods empfängt, die noch keinen Node zugewiesen bekommen haben, 
wählt der Scheduler den am besten geeigneten Node für den Pod aus, unter Berücksichtigung der aktuellen Arbeitslast und 
der Verfügbarkeit der Ressourcen auf den Nodes.

#### Controller Manager
Der [Controller Manager](https://kubernetes.io/docs/concepts/overview/components/#cloud-controller-manager)
in Kubernetes führt verschiedene Controller-Prozesse aus, die den gewünschten Zustand des Clusters 
überwachen und sicherstellen. Zu diesen Controllern gehören unter anderem der Node Controller, der Replication Controller, 
der Endpoints Controller und der Service Account & Token Controllers. Jeder dieser Controller beobachtet den Zustand 
des Clusters über den API-Server und führt Korrekturmaßnahmen durch, um den aktuellen Zustand an den gewünschten Zustand 
anzupassen. Dies kann beispielsweise das Starten eines Pods, das Wiederherstellen eines Node oder das Erstellen zusätzlicher 
Ressourcen umfassen. Der Controller Manager ist somit entscheidend für die Selbstheilungsfähigkeit und das automatische Skalieren des Kubernetes-Clusters.

#### Cloud Controller Manager
Der Cloud Controller Manager (CCM) in Kubernetes ist eine Komponente, die es ermöglicht, die Orchestrierung des Clusters 
mit den APIs der jeweiligen Cloud-Provider zu integrieren. Der CCM abstrahiert die Abhängigkeit des Clusters von 
der spezifischen Cloud-Infrastruktur, wodurch Kubernetes in verschiedenen Cloud-Umgebungen konsistent arbeiten kann. 
Er verwaltet spezifische Cloud-Dienste wie das Load Balancing, das Netzwerk-Routing und die Erstellung von Persistent Volumes, 
die für die Cloud-spezifischen Ressourcen erforderlich sind. Der CCM arbeitet eng mit dem Kubernetes API-Server und anderen Komponenten zusammen, 
um den Cluster-Zustand zu verwalten und Anpassungen vorzunehmen, die spezifisch für den jeweiligen Cloud-Provider sind.

## Worker Node
Ein Worker Node in Kubernetes ist eine Maschine, die die Container der Anwendungen in Form von Pods ausführt. 
Er enthält Dienste wie Kubelet zur Pod-Verwaltung, Kube-Proxy für das Netzwerk-Routing und eine Container-Runtime zum Ausführen der Container. 
Worker Nodes führen die Arbeitslasten des Clusters aus und werden von der Control Plane gesteuert.

#### Kubelet
Das [Kubelet](https://kubernetes.io/docs/concepts/overview/components/#kubelet) 
ist eine Schlüsselkomponente auf jedem Worker Node in Kubernetes, die die Ausführung von Pods überwacht und steuert.
Es kommuniziert mit der Control Plane, um Pods zu erhalten und zu starten, die auf dem Node ausgeführt werden sollen, 
basierend auf den Anweisungen des Schedulers. Das Kubelet überwacht den Zustand der Pods und Container, um sicherzustellen, 
dass sie wie erwartet laufen, und meldet den Zustand zurück an die Control Plane. Es handhabt auch die Lebenszyklusereignisse der Container, 
wie das Starten und Stoppen, basierend auf den Steuerbefehlen der Control Plane.

#### Kube Proxy
Das [Kube-Proxy](https://kubernetes.io/docs/concepts/overview/components/#kube-proxy) 
ist eine Komponente auf jedem Worker Node in Kubernetes, die für das Netzwerk-Routing und die Lastverteilung 
für die Pods und Services innerhalb des Clusters zuständig ist. Es sorgt dafür, dass die Netzwerkkommunikation zu den Pods 
über die richtigen IP-Adressen und Ports erfolgt, indem es Netzwerkregeln auf dem Node einrichtet. Kube-Proxy ermöglicht 
den Zugriff auf die Services von außerhalb des Clusters und leitet Anfragen an die korrekten Pods weiter, 
auch wenn diese über mehrere Nodes verteilt sind. Es unterstützt verschiedene Formen des Netzwerk-Proxying, 
um eine effiziente Kommunikation und Lastverteilung zu gewährleisten.

## Plugins
Plugins ermöglichen es Kubernetes, mit einer Vielzahl von Technologien und Diensten zu arbeiten, 
ohne dass diese direkt in den Kubernetes-Kern integriert werden müssen. Dies fördert die Modularität und Flexibilität. 
Plugins werden über definierte Schnittstellen angesprochen, wodurch Kubernetes mit unterschiedlichen Implementierungen 
dieser Dienste arbeiten kann, solange sie die Schnittstellenspezifikationen erfüllen.

#### CNI (Container Network Interface)
Das [Container Network Interface (CNI)](https://kubernetes.io/docs/concepts/extend-kubernetes/compute-storage-net/network-plugins/)
ist eine Spezifikation für Netzwerk-Plugins, die es ermöglichen, Netzwerkverbindungen für Container zu konfigurieren und zu verwalten. 
Wenn ein Pod gestartet oder gelöscht wird, ruft das Kubelet das CNI-Plugin auf, um das Netzwerk des Pods einzurichten bzw. zu entfernen. 
CNI-Plugins kümmern sich um die Netzwerkkonfiguration auf dem Node-Ebene, wie das Erstellen von Netzwerkbrücken, 
das Konfigurieren von IP-Adressen und das Einrichten von Routing-Regeln, um die Kommunikation zwischen Pods auf verschiedenen Nodes zu ermöglichen.
Kubernetes unterstützt CNIs wie z.B. [Cilium](https://cilium.io/), [Calico](https://www.tigera.io/project-calico/) oder
[Flannel](https://github.com/flannel-io/flannel)

#### CSI (Container Storage Interface)
Das [Container Storage Interface (CSI)](https://kubernetes.io/blog/2019/01/15/container-storage-interface-ga/) 
ist eine Spezifikation für Speicher-Plugins, die eine standardisierte Schnittstelle für Container-Orchestrierungssysteme bietet, 
um mit externen Speichersystemen zu interagieren. CSI ermöglicht es Kubernetes, Volumes von verschiedenen Speicheranbietern zu erstellen, 
anzuhängen, zu mounten und zu verwenden, ohne dass spezifischer Code für jeden Speicheranbieter in den Kubernetes-Kern aufgenommen werden muss. 
Dies ermöglicht Benutzern die Flexibilität, Speicherlösungen zu nutzen, die am besten zu ihren Anforderungen passen, 
einschließlich Cloud-basierter Speicher, Netzwerk-Dateisysteme und Blockspeicher.

#### CRI (Container Runtime Interface)
Das [Container Runtime Interface (CRI)](https://kubernetes.io/docs/concepts/architecture/cri/) ermöglicht Kubernetes die Integration verschiedener Container-Runtimes über eine standardisierte API-Schnittstelle. 
Dadurch kann Kubernetes unabhängig von der zugrunde liegenden Container-Technologie funktionieren und unterstützt Runtimes wie 
[containerd](https://containerd.io/) und [CRI-O](https://cri-o.io/).
