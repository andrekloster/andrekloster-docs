# Einen Puppet Client registrieren

[TOC]

## Einleitung
Um einen Rechner via Puppet zu konfigurieren, muss der Rechner zunächst als Client registriert werden.

## Puppet Agent auf dem Client installieren

```shell
# Installiere notwendige Update und setze den Hostnamenyy
sudo apt update && sudo apt upgrade -y
sudo hostnamectl set-hostname <PUPPET-CLIENT-FQDN> # Anpassen!
```

!!! info
    Nach der Installation von `puppet-agent` ist es empfehlenswert eine neue SSH Verbindung mit dem Client aufzubauen,
    da somit die `$PATH` Variable aktualisiert wird. Ansonsten kriegen wir die Fehlermeldung `puppet: command not found`.

```shell
sudo apt update
sudo apt install -y puppet-agent
```

Nun öffnen wir die Puppet Konfigurationsdatei und legen grundsätzliche Werte für den Client fest.

```shell
sudo vim /etc/puppet/puppet.conf
```

```ini
[agent]
environment = production
server = <PUPPET-SERVER-FQDN> # Anpassen!
```

## Client authentifizieren
Bei der ersten Verbindung zum Puppet Server wird ein selbstsignierte Zertifikatsanfrage erstellt. Diese Anfrage muss vom Server 
bestätigt werden, um daraus ein TLS Zertifikat für die Kommunikation zwischen Server und Client zu erstellen.

### Client
```shell
# Führe den Puppet Agent zum ersten Mal im dry-run aus
puppet agent -t --noop
```

Es entsteht folgender Output:

```
root@puppet-client:~# puppet agent -t --noop
Info: Creating a new RSA SSL key for puppet-client.domain.local
Info: csr_attributes file loading from /etc/puppetlabs/puppet/csr_attributes.yaml
Info: Creating a new SSL certificate request for puppet-client.domain.local
Info: Certificate Request fingerprint (SHA256): 7B:E9:2D:51:F1:D6:43:33:64:59:98:DA:74:DD:91:AC:BC:27:CF:AD:86:B8:2E:BF:0F:80:85:5E:0B:9D:45:0B
Info: Certificate for puppet-client.domain.local has not been signed yet
Couldn't fetch certificate from CA server; you might still need to sign this agent's certificate (puppet-client.domain.local).
Exiting now because the waitforcert setting is set to 0.
```

Wie wir sehen wurde für `puppet-client.domain.local` eine Request erstellt. Dieser Request hat einen Fingerprint, den wir uns notieren.

### Server
```shell
# Zeige alle offenen Request an
puppetserver ca list
```

Wir sehen im Output alle offenen Zertifikatsanfragen:

```
root@puppet-server:~# puppetserver ca list
Requested Certificates:
    puppet-client.domain.local       (SHA256)  7B:E9:2D:51:F1:D6:43:33:64:59:98:DA:74:DD:91:AC:BC:27:CF:AD:86:B8:2E:BF:0F:80:85:5E:0B:9D:45:0B
```

Wir vergleichen den Fingerprint. Falls alles übereinstimmt, so verifizieren wird die Anfrage und erstellen ein Zertifikat.
Anschließend wäre der Client bereit, um konfiguriert zu werden.

```shell
puppetserver ca sign --certname='puppet-client.domain.local'
```