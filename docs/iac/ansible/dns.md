# Lokaler DNS Server mit Bind9 erstellen

[TOC]

## Ziel

Mit Hilfe von Ansible möchten wir die Einrichtung und Wartung eines lokalen Bind9-DNS automatisieren und somit den administrativen Aufwand vereinfachen.
Besonders im HomeLab ist ein lokaler DNS besonders hilfreich und ermöglicht eine bessere Kommunikation zwischen internen Diensten im Netzwerk.
Am Ende dieser Anleitung wird es möglich sein mit nur einer YAML Datei alle notwendigen DNS Records zu verwalten.

## Voraussetzungen

Zur Einrichtung des DNS wird ein Linux Server mit z.Bsp. Debian benötigt. Zusätzlich brauchen wir eine Maschine auf der Ansible installiert ist.
Beide Rechner müssen sich über ein Netzwerk erreichen können und wir müssen aufpassen, dass der Port 53 (TCP/UDP) nicht von einer Firewall blockiert wird.
Abschließend erfinden wir eine Domain, die zukünftig in unserem Netzwerk gelten soll.

```
# Domain
test.local

# IPv4 CIDR
10.192.1.0/24

# IPv6 CIDR
fd4b:7618:f5ec:798d::/64

# DNS Server
- Hostname: dns.test.local
- IPv4: 10.192.1.1
- IPv6: fd4b:7618:f5ec:798d::1

# Ansible
- ansible.test.local
- IPv4: 10.192.1.2
- IPv6: fd4b:7618:f5ec:798d::2
```

## Vorbereitung: Ansible Playbook und Role

Zunächst stellen wir sicher, dass der Ansible Rechner und der DNS Server sich gegenseitig im Netzwerk erreichen können.

```bash
# Folgenden Befehl für wir auf dem Ansible Rechner aus
ping 10.192.1.1
```

Anschließend klonen wir das [Git Repository](https://github.com/andrekloster/boilerplate), welches für diese Anleitung vorbereitet wurde.

```bash
git clone https://github.com/andrekloster/boilerplate.git
```

In diesem Repository befindet sich das Verzeichnis `iac/ansible/bind9`, das eine Vorlage für die Anleitung zur Verfügung stellt.

```
.
└── iac
    └── ansible
        └── bind9
            ├── dns.yaml
            ├── inventory
            └── roles
                └── dns
                    ├── handlers
                    │   └── main.yaml
                    ├── tasks
                    │   └── main.yaml
                    ├── templates
                    │   └── etc
                    │       └── bind
                    │           ├── named.conf.local
                    │           ├── named.conf.options
                    │           └── zones
                    │               ├── 1.192.10.in-addr.arpa
                    │               ├── d.8.9.7.c.e.5.f.8.1.6.7.b.4.d.f.ip6.arpa
                    │               └── test.local
                    └── vars
                        └── main.yaml
```

### Stammverzeichnis

**dns.yaml**

```yaml
---
- hosts: dns
  become: true
  roles:
    - dns
```

Das ist unser Ansible Playbook. Dieses Playbook führt mit privilegierte Rechten auf dem DNS Server unsere Ansible `dns` role aus.

**inventory**

```ini
[dns]
10.192.1.1

[all:children]
dns

[all:vars]
ansible_python_interpreter=/usr/bin/python3
```

In der `inventory` Datei definieren wir eine dns-Gruppe und tragen dort die IP vom DNS-Server ein. Diese Datei vereinfacht die Durchführung vom Playbook.

### roles/dns Verzeichnis

**handlers/main.yaml**

```yaml
---
# handlers file for Bind setup
- name: restart bind
  service:
    name: bind9
    state: restarted
```

Dieser Handler kann in einem Ansible Playbook aufgerufen werden, um den `bind9` Daemon neu zu starten.

**tasks/main.yaml**

```yaml
---
- include_vars: vars/main.yaml
- name: Install bind9
  apt:
    pkg:
      - bind9
    update_cache: true
- name: copy named.conf.local
  template:
    src: templates/etc/bind/named.conf.local
    dest: /etc/bind/named.conf.local
    owner: root
    group: bind
    mode: '0644'
- name: copy named.conf.options
  template:
    src: templates/etc/bind/named.conf.options
    dest: /etc/bind/named.conf.options
    owner: root
    group: bind
    mode: '0644'
- name: create zones directory
  ansible.builtin.file:
    state: directory
    path: /etc/bind/zones
    owner: root
    group: bind
    mode: '0750'
- name: create bind logs directory
  ansible.builtin.file:
    state: directory
    path: /var/log/named
    owner: bind
    group: bind
    mode: '0750'
- name: modify forward zone
  template:
    src: templates/etc/bind/zones/{{ item }}
    dest: /etc/bind/zones/{{ item }}
    owner: root
    group: bind
    mode: 0640
  loop:
    - 'test.local'
    - '1.192.10.in-addr.arpa'
    - 'd.8.9.7.c.e.5.f.8.1.6.7.b.4.d.f.ip6.arpa'
  notify: restart bind
```

Hier wird das `bind9` Paket installiert, sowie alle notwendigen Verzeichnisse und Konfigurationen mit den richtigen Berechtigungen hinterlegt.

**templates/etc/bind/named.conf.local**

```jinja
{% for zone in zones.keys() %}
zone "{{ zone }}" {
    type master;
    file "/etc/bind/zones/{{ zone }}";
};

{% endfor %}
```

In diesem bind Konfigurationstemplate werden mit Hilfe einer for-Schleife alle DNS-Zonen angegeben.

**templates/etc/bind/named.conf.local**

```
options {

  directory "/var/cache/bind";
  
  forwarders {
    1.1.1.1;
  };
  forward only;
  
  listen-on port 53 {
    127.0.0.1;
    10.192.1.1;
  };

  listen-on-v6 {
    ::1;
    fd4b:7618:f5ec:798d::1;
  };

  allow-query {
    127.0.0.1;
    10.0.0.0/8;
    ::1;
    fd4b:7618:f5ec:798d::/64;
  };
  
  allow-update-forwarding { none; };
  auth-nxdomain no;
  dnssec-validation auto;
};

logging {

  channel simple_log {
    file "/var/log/named/bind.log" versions 3 size 3m;
	  severity notice;
	  print-time yes;
	  print-severity yes;
	  print-category yes;
  };

  category default {
    simple_log;
    default_syslog;
  };
};
```

In dieser Datei wird der `bind9` Daemon konfiguriert. Hier wird angegeben, welchen Forwarder der DNS Server benutzt, auf welchem Socket der Dienst läuft und aus welchen Netzwerkbereichen die DNS-Queries kommen dürfen. Die Abschnitte `allow-transfer` und `also-notify` sind erstmal auskommentiert. Diese werden bei einem Master-Slave Setup benötigt, welcher im Kapitel `Extra` erklärt wird.

**templates/etc/bind/zones/test.local**

```jinja
$TTL 10m
@ IN SOA dns.test.local. info.dns.test.local. (
{% for serial in zones['test.local']['serial'] %}
    {{ serial }} ; serial
{% endfor %}
    3h ; refresh
    10m ; retry
    7d ; expire
    10m ; TTL
)

@ IN NS dns.test.local.

{% for hostname, ip in zones['test.local']['records']['a'].items() %}
{{ hostname }}  IN  A    {{ ip[0] }}
{% endfor %}

{% for cname, destination in zones['test.local']['records']['cname'].items() %}
{{ cname }}  IN  CNAME    {{ destination[0] }}
{% endfor %}
```

Das ist unser Template für die DNS-Zone. Darin verwalten wir alle DNS Records mit der Zuordnung Domain -> IP.

**templates/etc/bind/zones/1.192.10.in-addr.arpa**

```jinja
$TTL 60
@ IN SOA dns.test.local. info.dns.test.local. (
{% for serial in zones['1.192.10.in-addr.arpa']['serial'] %}
    {{ serial }} ; serial
{% endfor %}
    3h ; refresh
    10m ; retry
    7d ; expire
    10m ; TTL
)

@    IN NS  dns.test.local.

{% for hostname, ip_suffix in zones['1.192.10.in-addr.arpa']['records'].items() %}
{{ ip_suffix[0] }}  IN  PTR  {{ hostname }}.test.local.
{% endfor %}
```

**templates/etc/bind/zones/8.9.d.b.9.b.6.5.8.c.4.e.0.7.d.f.ip6.arpa**
```jinja
$TTL 60
@ IN SOA dns.test.local. info.dns.test.local. (
{% for serial in zones['8.9.d.b.9.b.6.5.8.c.4.e.0.7.d.f.ip6.arpa']['serial'] %}
    {{ serial }} ; serial
{% endfor %}
    3h ; refresh
    10m ; retry
    7d ; expire
    10m ; TTL
)

@    IN NS  dns.test.local.

{% for hostname, ip_suffix in zones['d.8.9.7.c.e.5.f.8.1.6.7.b.4.d.f.ip6.arpa']['records'].items() %}
{{ ip_suffix[0] }}  IN  PTR  {{ hostname }}.test.local.
{% endfor %}
```


Das ist unsere IPv4 und IPv6 Templates für die DNS Rückwärtsauflösung. Darin verwalten wir alle DNS Records mit der Zuordnung IP -> Domain.

**vars/main.yaml**

```yaml
---
zones:
  test.local:
    serial: ["2022090201"]
    records:
      a:
        dns: ["10.192.1.1"]
        ansible: ["10.192.1.2"]
      aaaa:
        dns: ["fd4b:7618:f5ec:798d::1"]
        ansible: ["fd4b:7618:f5ec:798d::2"]
      cname:
        test-cname: ["dns"]
  1.42.10.in-addr.arpa:
    serial: ["2022090201"]
    records:
      dns: ["1"]
      ansible: ["2"]
  d.8.9.7.c.e.5.f.8.1.6.7.b.4.d.f.ip6.arpa:
    serial: ["2022090201"]
    records:
      dns: ["1.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0"]
      ansible: ["2.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0"]
```

Diese YAML ist die wichtigste Konfigurationsdatei. Alle DNS Records werden ausschließlich hier definiert.

## Durchführung

Um das Playbook auszuführen, verbinden wir uns via SSH mit dem Ansible Server und führen folgenden Befehl aus:

```shell
ansible-playbook -i inventory dns.yaml -K
```
