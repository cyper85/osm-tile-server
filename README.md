# Eigener Tile-Server

## Vorgeplänkel
2018 kam die DSGVO. Natürlich völlig überraschend und alles verändert (eine aktive Sarkasmuserkennung sollte hier anschlagen ;)). Ich habe mir dabei meine eigenen Webseiten angeschaut und überlegt, wo ich Nutzerdaten potentiell an Dritte weitergebe. Grundsätzlich ist jede extern eingebundene Resource kritisch zu betrachten, denn jedesmal, wenn ein Nutzer auf meine Website schaut wird auch ein Request an die Server eines dritten gesendet. Dieser sieht den User, kann über Cookies ihn über viele Seiten tracken und die gesammelten Daten gewinnbringend weiterverkaufen.

Wer sich heute Websites anschaut sieht sehr schnell, wieviele externe Resourcen eingebunden sind. Angefangen bei Javascript- und CSS-Bibliotheken aus dem CDN, über Bilder und Videos bin hin zu Socialmedia-Interaktionen und klassischen Webanalysen.

Die Dritt-Anbieter-Bibliotheken hoste ich meist selber. Web-Analytic wird mit einem selbstgehosteten und konfigurierten Matamo (ehemals Piwik) realisiert. Fehlte nur noch eins: Die Kartenkacheln von OpenStreetMap.

Mein erster Versuch war plump. Ich baute einen ReverseProxy auf meinem eigenen Server und nutzte dennoch weiterhin den kostenlosen Service von OpenStreetMap. 2018, quasi mit dem Hype um die DSGVO stieg ich bei Docker ein und fand das System der Micro-Services faszinierend. Ein Beispiel-NGINX-Server für eine statische Website war sehr schnell eingerichtet, doch ein eigener Tile-Server war eine ganz andere Nummer. Durch meine Datenauswertung und -aufbereitung für die Stadtpläne von [Erfurt](https://map4erfurt.de), [Ilmenau](https://stadtplan-ilmenau.de) und [Jena](https://map4jena.de) habe ich mir bereits viele Grundlagen im Bereich um PostGIS und Datenimport aus Openstreetmap angeeignet. Es musste damit doch *nur noch* ein Tile-Server her.

Klingt einfach, war es aber nicht. Dazu kamen meine anfänglichen geringen Skills im Docker-Bereich. Irgendwann stieß ich auf das fertige Image von [ovserv](). Es war ein All-Inclusive Angebot. Ein Container der alles beinhaltet. Die Lernkurve war niedrig und der Service innerhalb zweier Abende auf dem Stadtplan integriert.. Da war er, mein aller erster Tile-Server.

Im Laufe der Zeit kamen aber Probleme auf bei der Nutzung des Containers. Angefangen bei der Datenwartbarkeit. Ich musste jedesmal den Container deaktivieren (und damit den Tile-Server abschalten) um neue Daten in die Datenbank einspielen zu können. Eine solche Downtime verhinderte, dass ich die Daten automatisiert regelmäßig aktualisierte. Auch widersprach das All-In-One Prinzip den dezidierten Microservices. Zum überlaufen kam das Fass im Winter 2019, als das Image nicht mehr von Docker-Hub beziehbar wurde. Dank des Github-Repositories konnte ich mir das Image lokal erneut bauen und verwenden, jedoch arbeitete ich wieder an meinem Plan ein eigenes Image zu erstellen. Am Ende waren es vier und steigend.

## Die verschiedenen Server
Der Tile-Server braucht drei Komponenten. Zum ersten ist da die Datenbank. In unserem Fall eine Postgres-Datenbank mit PostGIS-Erweiterung. Die zweite Komponente der Datenimporter. Im Gegensatz zur Datenbank braucht man ihn nur für den Import zu starten und kann ihn danach wieder stoppen. Nummero drei ist der eigentliche Tile-Server, der, analog zur Website [OpenStreetMap.org](https://openstreetmap.org) mit einem Apache, Mapnik und Carto arbeitet.

### Die Datenbank
Postgres gibt bietet selbst aktuelle Images auf Docker-Hub an. Leider kommen diese erst einmal ganz ohne Postgis daher. Ich habe daher, basierend auf *posgres:latest* einen eigenen postgis-Container gebaut. Der Vorteil ist, dass der Postgres-Container für die Datenbank bereits optimiert wurde und ich auf viele nützlcihe Standards zurückgreifen kann (wie den vordefinierten Environments).

#### Installation
Die Installation ist relativ einfach:
```bash
docker run --detach cyper85/postgis
```

Im Container ist PostGIS nachinstalliert und die Datenbank bekommt automatisch die *postgis* und *hstore* Erweiterung.

Leider habe ich es bisher nicht hinbekommen das ganze unter Alpine installiert zu bekommen. Daher läuft der ganze Container in einem ???. Eventuell wird der Alpine-Container nachgereicht.

### Der Importer
Für den Import wird osm2pgsql verwendet. Im Gegensatz zum Datenbank. und Tile-Container muss der Importer nur während des Imports laufen. Wenn er seine Arbeit fertiggestellt hat kann er gestoppt werden und ohne RAM- und CPU-Verbrauch auf seinen nächsten Einsatz warten.

Der Importer ist sehr einfach gestrickt. Als ENV gibt man ihm eine URL zu einem PBF mit. Dieses wird heruntergeladen und dann via osm2pgsql mit den aktuekllen Schema vom Carto-Projekt in den Postgres-Server übertragen. Der Server steht sowohl als Ubuntu, als auch als Alpine-Version zur Verfügung.

#### Installation
Wir brauchen einen Postgis-Server und ein gemeinsames Netzwerk, in dem beide Container sind. Außerdem sollten wir dem PostGIS-Container und dem Import-Container individuelle Namen vergeben. Der PostGIS, damit wir vom Importer aus den Datenbank-Container einfach adressieren können und den Import-Container, damit wir ihn später einfacher neustarten können.

```bash
# Create a Network to use the Postgis-Server in an other container
docker network create postgis-net

# Install a postgis-instance
docker run --detach --name test-postgis --network postgis-net cyper85/postgis

# Install a osm2pgsql-instance
docker run --env POSTGRES_HOST=test-postgis --env PBF_URL=https://download.geofabrik.de/europe/germany/thueringen-latest.osm.pbf --name test-osm2pgsql --network postgis-net cyper85/osm2pgsql
```

Port, Datenbankname, -nutzer und -passwort sind die Standardeinstellungen von Postgres. Wenn man diese im PostGIS-Container individualisiert, müssen die Werte über die gleichbenannten ENVs an den Importer (und später auch an den Tile-Container) übergeben werden. In unserem Fall reicht es dem Importer zu sagen, wie der Name unseres Postgres-Host ist.

#### Daten-Update
Der erste Start importiert automatisch unsere Daten aus dem verlinkten PBF und stoppt anschließend den Import-Container. Um die Daten zu aktualisieren, muss der Container nur neugestartet werden:

```bash
docker start test-postgis 
```

### Der Tile-Server
Bisher wurde nur vorbereitet, jetzt wollen wir unsere Früchte ernten.

Der Tile-Server muss im selben Netzwerk, wie unser Datenbank-Container sein. Außerdem braucht er, analog zum Importer, den Containernamen der Datenbank um sich mit dieser zu verbinden.

Startet man den Container kann es ein paar Minuten dauern, bis der Server betriebsbereit ist.

* URL zu Tiles
* Mount-Dir Datenbank und Tile-Cache
* Explizite Anleitung für den Stadtplan
