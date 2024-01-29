# Kleine Public-Key-Infrastruktur

!!! info "Hinweis"
    In diesem Kapitel wird beschrieben, wie eine kleine, eigenständige PKI erstellt werden kann.
    Diese Art von Mini-PKI kann für besondere Einsatzgebiete verwendet werden, wo keine große PKI-Infrastruktur notwendig ist.

## Root CA

Wir erstellen uns zunächst lokal ein temporäres Arbeitsverzeichnis.

```shell
MINI_CA_DIR='/tmp/mini_pki' # ggfs. anpassen
mkdir -p "${MINI_CA_DIR}"
cd "${MINI_CA_DIR}"
```

Anschließend generieren wir unsere `openssl.cnf`. Hierbei beinhaltet die
Konfigurationsdatei alle Parameter, die für eine Root CA benötigt wird.

```shell
vim openssl.cnf
```

```ini
[ x509_ca_ext ]
subjectKeyIdentifier   = hash
basicConstraints       = critical,CA:TRUE,pathlen:0
keyUsage               = critical,keyCertSign,cRLSign
certificatePolicies    = anyPolicy
```

Als Nächstes wird für die Root CA ein privater Schlüssel generiert,
der mit einem Passwort versehen wird. Dieser private Schlüssel
ist essentiel für die Signatur aller Zertifikate der PKI.

=== "EC"
    ```shell
    openssl genpkey -algorithm 'EC' -pkeyopt 'ec_paramgen_curve:secp384r1' \
        -aes128 -out 'root_ca_key.pem'
    ```

=== "RSA"
    ```shell
    openssl genpkey -algorithm 'RSA' -pkeyopt 'rsa_keygen_bits:4096' \
        -aes128 -out 'root_ca_key.pem'
    ```

Mithilfe des privaten Schlüssels können wir einen `Certificate Signing Request (CSR)` erstellen.
Wichtig ist hier der `Common Name (CN)` im Subject. Der `CN` legt den Namen der Root CA fest.

```shell
# Subject anpassen !
openssl req -new -key 'root_ca_key.pem' -sha256 -out 'req_ca.csr' \
   -subj "/C=DE/ST=Berlin/L=Berlin/O=Domain Local/OU=Test/CN=Mini Root CA G1"
```

Wir überprüfen gründlich den Inhalt vom Request.

```shell
openssl req -noout -text -in 'req_ca.csr'
```

Wir generieren eine zufällige und eindeutige ID, die der zukünftigen Root CA zugeordnet wird.

```shell
SSL_CURRENT_CA_ID=$(openssl rand -hex 8 | awk '{ print toupper($0) }')
```

Nun können wir den `Certificate Signing Request` mithilfe der `openssl.cnf` und des private Keys verarbeiten.
Am Ende erhalten wir ein selbstsigniertes Root Zertifikat.

```shell
openssl x509 -req -in 'req_ca.csr' \
  -extfile 'openssl.cnf' \
  -extensions 'x509_ca_ext' \
  -days '820' \
  -signkey 'root_ca_key.pem' \
  -sha256 \
  -out 'root_ca_cert.pem' \
  -set_serial $(echo "${SSL_CURRENT_CA_ID}" | awk '{printf "%u\n", "0x"$0}')
```

Wir überprüfen gründlich den Inhalt vom Zertifikat.

```shell
openssl x509 -noout -text -in 'root_ca_cert.pem'
```

## Server Zertifikat

!!! info "Hinweis"
    Wir gehen hier davon aus, dass wir uns weiterhin im temporären
    Arbeitsverzeichnis `${MINI_CA_DIR}` befinden. 

Zunächst bearbeiten wir die `openssl.cnf`. Dort sind alle Werte einzutragen,
die für ein normales Server-Zertifikat notwendig sind. Falls der Fall vorhanden
ist, dass das Zertifikat für mehrere Servernamen gelten soll,
so ist der Abschnitt `subjectAltName` zwingend notwenig.
Dort sind alle Servernamen einzutragen, für die das Zertifikat gelten soll.

```shell
vim openssl.cnf
```

```ini
## openssl.cnf
[ x509_ext ]
basicConstraints         = critical,CA:FALSE
subjectKeyIdentifier     = hash
authorityKeyIdentifier   = keyid:always
keyUsage                 = critical,nonRepudiation,digitalSignature,keyEncipherment
subjectAltName           = @server_alt_names
extendedKeyUsage         = serverAuth

[ server_alt_names ]
DNS.1 = test-1.example.local
DNS.2 = test.example.local
```

Anschließend generieren wir einen weiteren privaten Schlüssel für das Server-Zertifikat.
Dieser private Schlüssel wird ebenfalls mit einem eigenen Passwort versehen.

=== "EC"
    ```shell
    openssl genpkey -algorithm 'EC' -pkeyopt 'ec_paramgen_curve:secp384r1' \
        -aes128 -out 'server_key.pem'
    ```

=== "RSA"
    ```shell
    openssl genpkey -algorithm 'RSA' -pkeyopt 'rsa_keygen_bits:4096' \
        -aes128 -out 'server_key.pem'
    ```

Mithilfe des privaten Schlüssels können wir einen `Certificate Signing Request (CSR)` erstellen.
Wichtig ist hier auch der `Common Name (CN)` im Subject. Der `CN` legt den Servernamen fest,
an den das Zertifikat ausgestellt werden soll.

```shell
openssl req -new -key 'server_key.pem' -sha256 -out 'req_server.csr' \
    -subj '/C=DE/ST=Berlin/L=Berlin/O=example.local/OU=Test/CN=test-1.example.local'
```

Wir überprüfen gründlich den Inhalt vom Request.

```shell
openssl req -noout -text -in 'req_server.csr'
```

Anschließend können wir das Server-Zertifikat ausstellen. Dafür geben wir
innerhalb der Parameter den Server-Request, das Root CA Zertifikat und
den privaten Schlüssel der Root CA an. Am Ende erhalten wir ein gültiges
Server-Zertifikat. 

```shell
openssl x509 -req -in 'req_server.csr' \
    -days '397' \
    -extfile 'openssl.cnf' \
    -extensions 'x509_ext' \
    -CA 'root_ca_cert.pem' \
    -CAkey 'root_ca_key.pem' \
    -CAcreateserial \
    -sha256 \
    -out 'server_cert.pem'
```

Wir überprüfen gründlich den Inhalt vom Zertifikat.

```shell
openssl x509 -noout -text -in 'server_cert.pem'
```
