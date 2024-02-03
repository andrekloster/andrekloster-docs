# Docker

[TOC]

## Einleitung

**Docker** ist eine Open-Source-Plattform für die Containerisierung, die es Entwicklern und Systemadministratoren ermöglicht, Anwendungen in leichtgewichtigen, portablen Containern zu verpacken und zu verteilen. Diese Container enthalten alles, was eine Softwareanwendung zum Ausführen benötigt, einschließlich der Bibliotheken, Systemwerkzeuge, Code und Laufzeitumgebungen. Docker vereinfacht die Bereitstellung und den Betrieb von Anwendungen, da es die Konsistenz über Entwicklungs-, Test- und Produktionsumgebungen hinweg gewährleistet und dabei Plattformunabhängigkeit bietet. Durch die Isolation der Anwendungen in separaten Containern verbessert Docker auch die Sicherheit und ermöglicht eine effizientere Nutzung der Systemressourcen im Vergleich zu traditionellen Virtualisierungstechnologien.

## Vorbereitungen

### Installation

Es wird davon ausgegangen, dass Docker bereits auf dem System installiert ist. Falls nicht,
befindet sich in der [Docker Dokumentation die entsprechende Anleitung](https://docs.docker.com/manuals/).

### Beispiel-Repository klonen

Die folgenden Erklärungen stützen sich auf ein vorbereitetes [Beispiel-Repository](https://github.com/andrekloster/test-color-app), welches wir uns klonen. In diesem Repository befindet sich eine einfache Python Flask Applikation.

```shell
git clone https://github.com/andrekloster/test-color-app
```

## Docker Image bauen

Um ein wiederverwendbares Docker Image zu bauen, benötigen wir eine sogenannte `Dockerfile`. Für unsere Beispiel-Flask-App
sieht die Dockerfile folgendermaßen aus:

```Dockerfile
# Baseimage
FROM python:3.12-slim
# Setze das Standard Workdir
WORKDIR /app
# Kopiere die requirements.txt und installiere sie via pip
COPY requirements.txt requirements.txt
RUN pip install -r requirements.txt
# Kopiere der Source Code
COPY . .
# Veröffentliche den Container auf Port 5000
EXPOSE 5000
# Starte die app.py
CMD ["python", "app.py"]
```

Jetzt können wir mit dem folgenden Befehl das Container Image bauen. Für unser Image wählen wir den Namen `test-color-app`
und verwendet das Tag `latest`

```shell
docker build -t test-color-app:latest -f Dockerfile ./
```

Anschließend können wir überprüfen, ob das Image erfolgreich bei uns lokal gebaut wurde

```shell
docker image ls test-color-app
```

### Docker Image veröffentlichen

Ein lokal gebautes Image kann in eine zentrale Registry hochgeladen werden. Ein solche Registry ist z.B.
[Docker Hub](https://hub.docker.com/). Images die in einer Registry angelegt wurde, können von allen berechtigten
Parteien verwendet werden.

Zunächst muss das lokal gebaut Image für die entsprechende Registry getagged werden

```shell
docker tag test-color-app:latest 5470/test-color-app:latest
```

Anschließend für wir auf der CLI ein Login durch

```
docker login
```

Nachdem wir uns erfolgreich eingeloggt haben, können wir das Image pushen.

```shell
docker push 5470/test-color-app:latest
```

## Container lokal starten und verwendet

Um den Container lokal auf dem Rechner zu starten, um auf die Applikation zuzugreifen, führen wir folgenden Befehl aus.
Dabei wird der Container mit dem gebauten Image im Hintergrund gestartet und ist über den Port 5000 erreichbar.

```shell
docker run \
  --name test-color-app \
  --publish 5000:5000 \
  --rm \
  -d \
  test-color-app:latest
```

Mit dem folgenden Befehl können wir uns alle laufenden Container anzeigen lassen:

```shell
docker ps
```

Jetzt können wir überprüfen, ob die Flask Applikation für `curl` aufrufbar ist.

```shell
curl http://localhost:5000/
```

Darauf sollte diese Antwort zurückgegeben werden

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

### Mit dem Container verbinden

Um sich mit dem laufenden Container zu verbinden, ist dieser Befehl notwendig:

```shell
docker exec -it test-color-app sh
```

Damit befinden wir uns im Container und können gekappselt Kommandos ausführen.  
**Achtung:** Der aktuelle Container ist nicht persistent! Alle durchgeführten Anpassungen innerhalb des Container
werden gelöscht, sobald der Container gestoppt wird.

### Container stoppen

Um den Container sofort zu stoppen, muss dieser Befehl ausgeführt werden:

```shell
docker stop test-color-app -t 0
```

## Docker Compose

Mithilfe von Docker Compose können wir innerhalb einer YAML Datei ein oder mehrere Docker Container konfigurieren.
Dadurch wird ein deklaratives Setup ermöglicht, das zentral in Git gespeichert werden kann.
Das folgende Beispiel zeigt eine Docker Compose Datei aus dem Beispiel-Repository, welches einen Reverse Proxy Nginx Container
und einen Backend Flask Container erstellt.

```yaml
---
version: "3.9"
services:
  nginx:
    image: nginx:stable
    container_name: nginx
    restart: always
    volumes:
      - ./docker/nginx/nginx.conf:/etc/nginx/nginx.conf
      - ./docker/nginx/sites-enabled:/etc/nginx/sites-enabled
    ports:
      - "80:80"
      - "8080:8080"
    command: 'nginx -g "daemon off;"'
    depends_on:
      - test-color-app
  test-color-app:
    build:
      context: ./
      dockerfile: Dockerfile
    container_name: test-color-app
    working_dir: /app
    ports:
      - "127.0.0.1:5000:5000"
```

Mit `docker compose` ist es uns somit möglich alle Container zeitgleich zu starten/stoppen.

```shell
docker compose up -d --build
```

Anschließend lässt sich die Flask Applikation über den Nginx Reverse Proxy am Port `8080` erreichen.

```shell
curl http://localhost:8080/
```

Um alle Container via Docker Compose zu stoppen, reicht dieser Befehl:

```shell
docker compose down
```
