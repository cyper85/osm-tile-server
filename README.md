# Eigener Tile-Server

## Vorgeplänkel
2018 kam die DSGVO. Natürlich völlig überraschend und alles verändert sich (eine aktive Sarkasmuserkennung sollte hier 
anschlagen ;)). Ich habe mir dabei meine eigenen Webseiten angeschaut und überlegt, wo ich Nutzerdaten potentiell an 
Dritte weitergebe. Grundsätzlich ist jede extern eingebundene Resource kritisch zu sehen, denn jedesmal, wenn ein 
Nutzer auf meine Website schaut wird auch ein Request an die Server eines dritten gesendet. Dieser sieht den User, 
kann über Cookies ihn über viele Seiten tracken und die gesammelten Daten gewinnbringend weiterverkaufen.

Wer sich heute Websites anschaut sieht sehr schnell, wieviele externe Resourcen eingebunden sind. Angefangen bei 
Javascript- und CSS-Bibliotheken aus dem CDN, über Bilder, WebFonts und Videos bin hin zu Socialmedia-Interaktionen 
und klassischen Webanalysen.

Die Dritt-Anbieter-Bibliotheken hoste ich selbst. Web-Analytic wird mit einem selbstgehosteten und konfigurierten 
[Matomo](https://matomo.org) (ehemals Piwik) realisiert. Fehlte nur noch eins: Die Kartenkacheln von OpenStreetMap.

Mein erster Versuch war plump. Ich baute einen ReverseProxy auf meinem eigenen Server und nutzte dennoch weiterhin den 
kostenlosen Service von OpenStreetMap. 2018, quasi mit dem Hype um die DSGVO, stieg ich bei Docker ein und fand das 
System der Micro-Services faszinierend. Ein Beispiel-[NGINX-Server](https://hub.docker.com/_/nginx) für eine statische 
Website war sehr schnell eingerichtet, doch ein eigener Tile-Server war eine ganz andere Nummer. Durch meine 
Datenauswertung und -aufbereitung für die Stadtpläne von [Erfurt](https://map4erfurt.de), 
[Ilmenau](https://stadtplan-ilmenau.de) und [Jena](https://map4jena.de) habe ich mir bereits viele Grundlagen im 
Bereich um PostGIS und Datenimport aus Openstreetmap angeeignet. Es musste damit doch *nur noch* ein Tile-Server her.

Klingt einfach, war es aber nicht. Dazu kamen meine anfänglichen geringen Skills im Docker-Bereich. Irgendwann stieß 
ich auf das fertige Image Openstreetmap-Tile-Server von [overv](https://github.com/Overv/openstreetmap-tile-server). 
Es war ein All-Inclusive Angebot. Ein Container der alles beinhaltet. Die Lernkurve war niedrig und der Service 
innerhalb zweier Abende auf dem Stadtplan integriert. Da war er, mein aller erster Tile-Server.

Im Laufe der Zeit kamen aber Probleme auf bei der Nutzung des Containers. Angefangen bei der Datenwartbarkeit. Ich 
musste jedesmal den Container deaktivieren (und damit den Tile-Server abschalten) um neue Daten in die Datenbank 
einspielen zu können. Eine solche Downtime verhinderte, dass ich die Daten automatisiert regelmäßig aktualisierte. 
Auch widersprach das All-In-One Prinzip den dezidierten Microservices. Zum überlaufen kam das Fass im Winter 2019, als 
das Image nicht mehr von Docker-Hub beziehbar wurde. Dank des Github-Repositories konnte ich mir das Image lokal 
erneut bauen und verwenden, jedoch arbeitete ich wieder an meinem Plan ein eigenes Image zu erstellen. Am Ende waren 
es vier und steigend.

## Die verschiedenen Server
Der Tile-Server braucht drei Komponenten. Zum ersten ist da die **Datenbank**. In unserem Fall eine 
PostgreSQL-Datenbank mit PostGIS-Erweiterung. Die zweite Komponente der **Datenimporter**. Im Gegensatz zur Datenbank 
braucht man ihn nur für den Import zu starten und kann ihn danach wieder stoppen. Nummero drei ist der eigentliche 
**Tile-Server**, der, analog zur Website [OpenStreetMap.org](https://openstreetmap.org) mit einem Apache, Mapnik und 
Carto arbeitet.

### Die Datenbank
[PostGIS](https://de.wikipedia.org/wiki/PostGIS) bietet selbst ein 
[aktuelles Images auf Docker-Hub](https://registry.hub.docker.com/r/postgis/postgis) an. 

#### Installation
Die Installation ist einfach:

```bash
docker run --detach postgis/postgis
```

### Der Importer
Für den Import wird [osm2pgsql](https://github.com/openstreetmap/osm2pgsql) verwendet. Im Gegensatz zum Datenbank-
und Tile-Container muss der Importer nur während des Imports laufen. Wenn er seine Arbeit fertiggestellt hat kann er 
gestoppt werden und ohne RAM- und CPU-Verbrauch auf seinen nächsten Einsatz warten.

Der Importer ist sehr einfach gestrickt. Als ENV gibt man ihm eine URL zu einem PBF mit. Die Datei wird heruntergeladen 
und via osm2pgsql mit den aktuekllen Schema vom Carto-Projekt in den PostgreSQL-Server transferiert. Der Server steht 
sowohl als [Ubuntu](https://de.wikipedia.org/wiki/Ubuntu), als auch als Alpine-Version zur Verfügung.

#### Installation
Wir brauchen einen PostGIS-Server und ein gemeinsames Netzwerk, in dem beide Container sind. Außerdem sollten wir dem 
PostGIS-Container und dem Import-Container individuelle Namen vergeben. Der PostGIS, damit wir vom Importer aus den 
Datenbank-Container einfach adressieren können und den Import-Container, damit wir ihn später einfacher neustarten 
können.

```bash
# Create a Network to use the Postgis-Server in an other container
docker network create postgis-net

# Install a postgis-instance
docker run --detach --name test-postgis --network postgis-net postgis/postgis

# Install a osm2pgsql-instance
docker run \
       --env POSTGRES_HOST=test-postgis \
       --env PBF_URL=https://download.geofabrik.de/europe/germany/thueringen-latest.osm.pbf \
       --name test-osm2pgsql \
       --network postgis-net \
       cyper85/osm2pgsql
```

Port, Datenbankname, -nutzer und -passwort sind die Standardeinstellungen von Postgres. Wenn man diese im 
PostGIS-Container individualisiert, müssen die Werte über die gleichbenannten ENVs an den Importer (und später auch an 
den Tile-Container) übergeben werden. In unserem Fall reicht es dem Importer zu sagen, wie der Name unseres 
Postgres-Host ist.

#### Daten-Update
Der erste Start importiert automatisch unsere Daten aus dem verlinkten PBF und stoppt anschließend den 
Import-Container. Um die Daten zu aktualisieren, muss der Container nur neugestartet werden:

```bash
docker start test-osm2pgsql 
```

### Der Tile-Server
Bisher wurde nur vorbereitet, jetzt wollen wir unsere Früchte ernten.

Als Webserver kommt ein Apache zum Einsatz. Als Renderer wird renderd mit den aktuellem Mapnik-Style verwendet. 
Außerdem werden gerenderte Tiles in einem Cache abgelegt.

#### Installation
Der Tile-Server muss im selben Netzwerk, wie unser Datenbank-Container sein. Außerdem braucht er, analog zum Importer, 
den Containernamen der Datenbank um sich mit dieser zu verbinden.

```bash
# Create a Network to use the Postgis-Server in an other container
docker network create postgis-net

# Install a postgis-instance
docker run --detach --name test-postgis --network postgis-net postgis/postgis

# Install a osm2pgsql-instance
docker run --env POSTGRES_HOST=test-postgis --name test-osm2pgsql --network postgis-net cyper85/osm2pgsql

# Install a mapnik-instance
docker run --detach --env POSTGRES_HOST=test-postgis --name test-mapnik --network postgis-net --port 80:80 cyper85/mapnik
```
Startet man den Container kann es ein paar Minuten dauern, bis der Server betriebsbereit ist.

Wir haben den Port 80 exposed. Das heißt unser Container lauscht nun auf Requests: http://<your-ip-adress-here>:80/

Beim Aufruf der Seite sollte die Standard-Apache-Statusseite erscheinen.

Nun müssen wir alles noch in unsere Website einbauen. Mittels [Leaflet](https://leafletjs.com) ist das ganz einfach:
```html
<div id="map"></div>
<script>
var map = L.map('map').setView([0, 0], 3);
L.tileLayer("http://<your-ip-adress-here>:80/tile/{z}/{x}/{y}.png", {
  attribution: "Daten: OSM.org (<a class='extern' href='https://opendatacommons.org/licenses/odbl/'>ODbL</a>) | Darstellung: <a class='extern' href='https://openstreetmap.org/'>OSM.org</a> (<a class='extern' href='https://creativecommons.org/licenses/by-sa/2.0/de/'>CC-By-SA-2.0</a>)",
  maxZoom: 20
}).addTo(map);
</script>
```

## Real-Life-Betrieb
Bis hier hin habe ich mir schöne einfache Beispiele ausgedacht, doch wie setze ich das ganze wirklich ein?

```bash
# Create a Network to use the Postgis-Server in an other container
docker network create tile-net

# Create Volumes for Database-Data und Tile-Cache
docker volume create tile-db
docker volume create tile-cache

# Install a postgis-instance
docker run --detach \ 
           --env POSTGRES_USER=bubsi \
           --env POSTGRES_PASSWORD=RGKLLUN4x5Qu7a \
           --name tile-postgis \
           --network tile-net \
           --volume tile-db:/var/lib/postgresql/data \
           postgis/postgis

# Install a osm2pgsql-instance
docker run --env POSTGRES_USER=bubsi \ 
          --env POSTGRES_PASSWORD=RGKLLUN4x5Qu7a \
          --env POSTGRES_HOST=tile-postgis \
          --name tile-osm2pgsql \
          --network tile-net \
          cyper85/osm2pgsql

# Install a mapnik-instance
docker run --detach \
           --env POSTGRES_USER=bubsi \
           --env POSTGRES_PASSWORD=RGKLLUN4x5Qu7a \
           --env POSTGRES_HOST=tile-postgis \
           --name tile-mapnik \
           --network tile-net \
           --port 12345:80 \
           --volume tile-cache:/var/lib/mod_tile \
           cyper85/mapnik
```

Vor dem Docker-Container ist noch ein Revers-Proxy mit nginx geschaltet. Er verwandelt den einen HTTP-Service in vier 
HTTPS-Services. Der hier beschriebene Port sowie die User-Daten sind natürlich frei erfunden 😉.

Um das ganze noch besser wartbar zu machen, gibt es ein docker-compose-Skript um den Tile-Server einzurichten: 
[docker-compose.yml](docker-compose.yml).

Wichtig hier ist die ENV-Datei [database.env](database.env). Hier sind unsere Zugangsdaten zum PostGIS-Server. Leider 
kommt der PostGIS-Server durcheinander, wenn wir hier die Environment-Variable $POSTGRE_HOST setzen. Daher ist sie 
als extra *env* im *docker-compose.yml* enthalten.

Um den Server zu starten, fahren wir Service für Service hoch:

```bash
# PostGIS-Server
docker-compose up -d postgis

# Importer
docker-compose up osm2pgsql

# Tile-Server
docker-compose up -d mapnik
```

Der Importer wird auch hier ohne *--detach* gestartet, da er nach dem Ausführen wieder heruntergefahren wird. Um ein 
neues PBF zu importieren, führen wir einfach obigen Befehl erneut aus:

```bash
docker-compose up osm2pgsql
```