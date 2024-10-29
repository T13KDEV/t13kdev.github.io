---
title: Docker Backup
image: 
  path: https://i.ibb.co/fvhfFMv/10416564-m.jpg
sitemap: true
categories: [Docker]
tag: [HowTo, Backup]
comments: false
---

Das Sichern von Docker Containern und dessen Volumes sollte man auf jeden Fall nicht dem Zufall überlassen. Ein Restore Test ist ebenfalls obligatorisch, denn was nutzt das beste Backup wenn man es im Ernstfall nicht schnell und einfach zurückspielen kann. In der Theorie sieht ein Backup Szenario bei Docker so aus, dass man Volumes mittels "Hilfscontainer" via `tar`sichert. Datenbanken können entweder über den eigenen Container, oder auch über einen "Hilfscontainer" mit den gängigen DUMP-Befehlen gesichert werden. 

Zusätzlich sollte man alle Projekt spezifischen Dateien, zum Beispiel das `docker-compose.yml` sowie das eventuell vorhandene `.env` File sowie sämtliche andere Konfigurationsdateien, die man in den Container mountet gesichert werden. Diese liegen alle normalerweise im Docker Compose Projekt Ordner z.B. `/opt/` bzw. `/opt/stacks`, also diese Daten einfach zuerst sichern.

Zusammengefasst ist das Sicherungskonzept wie folgt:

* Docker Compose Projekt Ordner sichern
* Docker Container bzw. Image sichern Volumes via Hilfskontainer mit `tar`  sichern
* Datenbanken via Container und Dump- (Datenbank Sicherungs-) Tool sichern
* Dateien, die außerhalb des Compose Projekt Ordners liegen und in einen Container gemountet sind, sichern

## Docker Compose Projekt Ordner sicher

Zuerst benötigen wir einen lokalen Backup Ordner, diesen legen wir im Root oder in unserem Benutzerprofil an.

```bash
mkdir -p ~/backup
cp -r /opt/stacks/authentik ~/backup/
cp -r /opt/stacks/drawio ~/backup/
cp -r /opt/stacks/outline ~/backup/
cp -r /opt/stacks/planka ~/backup/
cp -r /opt/stacks/uptime-kuma ~/backup/
cp -r /opt/stacks/vaultwarden ~/backup/
cp -r /opt/dockge ~/backup/
cp -r /opt/nginx ~/backup/
```

## Docker Container bzw. Image sichern

Zuerst sollte man wissen, wie das zu sichernde Docker Volume heißt. Eine Übersicht ergibt der folgende Befehl:

```bash
sudo docker volume ls
```

Ich möchte die Volumes mit dem Namen **outline_storage-data und outline_database-data** sichern, das Backup Verzeichnis für den Container soll **/backup/outline/volumes/** sein, welches vorher angelegt werden muss:

```bash
mkdir -p ~/backup/outline/volumes
sudo docker run --rm -v ~/backup/outline/volumes:/backup -v outline_storage-data:/data:ro debian:stretch-slim bash -c "cd /data && /bin/tar -czvf /backup/outline_storage-data.tar.gz ."
sudo docker run --rm -v ~/backup/outline/volumes:/backup -v outline_database-data:/data:ro debian:stretch-slim bash -c "cd /data && /bin/tar -czvf /backup/outline_database-data.tar.gz ."
```

Analog dazu werden die anderen Volumes jeweils unter **\~/backup/<CONTAINER NAME>** gesichert.

Zum Restore muss das Ganze einfach umgedreht werden.

```bash
sudo docker run --rm -v ~/backup/outline/volumes:/backup -v outline_storage-data:/data debian:stretch-slim bash -c "cd /data && /bin/tar -xzvf /backup/outline_storage-data.tar.gz"
sudo docker run --rm -v ~/backup/outline/volumes:/backup -v outline_database-data:/data debian:stretch-slim bash -c "cd /data && /bin/tar -xzvf /backup/outline_database-data.tar.gz"
```

Nun sollten wieder alle Dateien an Ort und Stelle sein. Falls Dateien am Ort vorhanden sind, werden diese mit den Dateien aus dem Backup überschrieben.

* <https://www.laub-home.de/wiki/Docker_Volume_Backup_Script>

## Datenbanken sichern

### MySQL

So kann man ganz einfach eine MySQL oder eine ältere MariaDB mittels mysqldump und mysql Sichern und Wiederherstellen:

#### Backup unkomprimiert

```bash
docker exec CONTAINERNAME /usr/bin/mysqldump -u root --password=ROOTPASSWORD DATABASE > backup.sql
```

#### Restore unkomprimiert

```bash
cat backup.sql | docker exec -i CONTAINERNAME /usr/bin/mysql -u root --password=ROOTPASSWORD DATABASENAME
```

und das Ganze nun noch komprimiert mit gzip:

#### Backup komprimiert

```bash
docker exec CONTAINERNAME /usr/bin/mysqldump -u root --password=ROOTPASSWORD DATABASE | gzip > backup.sql.gz
```

#### Restore komprimiert

```bash
zcat backup.sql.gz | docker exec -i CONTAINERNAME /usr/bin/mysql -u root --password=ROOTPASSWORD DATABASENAME
```

### MariaDB

So kann man ganz einfach eine MariaDB mittels mariadb-dump sichern und mit dem mysql command wiederherstellen:

#### Backup unkomprimiert

```bash
docker exec CONTAINERNAME /usr/bin/mariadb-dump -u root --password=ROOTPASSWORD DATABASE > backup.sql
```

#### Restore unkomprimiert

```bash
cat backup.sql | docker exec -i CONTAINERNAME /usr/bin/mysql -u root --password=ROOTPASSWORD DATABASENAME
```

und das Ganze nun noch komprimiert mit gzip:

#### Backup komprimiert

```bash
docker exec CONTAINERNAME /usr/bin/mariadb-dump -u root --password=ROOTPASSWORD DATABASE | gzip > backup.sql.gz
```

#### Restore komprimiert

```bash
zcat backup.sql.gz | docker exec -i CONTAINERNAME /usr/bin/mysql -u root --password=ROOTPASSWORD DATABASENAME
```

* <https://www.laub-home.de/wiki/Docker_MySQL_and_MariaDB_Backup_Script>

### PostgreSQL

Um PostgreSQL zu sichern, verwenden wir einfach pg_dump oder für alle Datenbanken pg_dumpall. Die Befehle schicken wir direkt in den Datenbank Container:

#### Backup Aller Datenbanken unkomprimiert

```bash
docker exec CONTAINERNAME pg_dumpall -c -U POSTGRESUSER > backup.sql
```

#### Backup Aller Datenbanken komprimiert

```bash
docker exec CONTAINERNAME pg_dumpall -c -U POSTGRESUSER | gzip > backup.sql.gz
```

#### Backup einer Bestimmten Datenbank unkomprimiert

```bash
docker exec CONTAINERNAME pg_dumpall -c -U POSTGRESUSER -d DATABASENAME > backup.sql
```

#### Backup einer Bestimmten Datenbank komprimiert

```bash
docker exec CONTAINERNAME pg_dumpall -c -U POSTGRESUSER -d DATABASENAME | gzip > backup.sql.gz
```

Der Restore erfolgt dann so:

#### Restore Aller Datenbanken unkomprimiert

```bash
cat backup.sql | docker -i exec CONTAINERNAME psql -U POSTGRESUSER
```

#### Restore Aller Datenbanken komprimiert

```bash
zcat backup.sql.gz |docker -i exec CONTAINERNAME psql -U POSTGRESUSER
```

#### Restore einer Bestimmten Datenbank unkomprimiert

```bash
cat backup.sql | docker -i exec CONTAINERNAME psql -U POSTGRESUSER -d DATABASENAME
```

#### Restore einer Bestimmten Datenbank komprimiert

```bash
zcat backup.sql.gz | docker -i exec CONTAINERNAME psql -U POSTGRESUSER -d DATABASENAME
```

* <https://www.laub-home.de/wiki/Docker_Postgres_Backup_Script>

### InfluxDB v2.x

Die InfluxDB in Version 2.x bringt mit der CLI influx einen Parameter backup / restore mit. Mit diesem kann man ganz einfach die InfluxDB Datenbank sichern. Voraussetzung hierfür ist, das ihr ein Backup Folder in euren Container vorab nach /backup mountet.

```
volumes:
  - "/backup/influxdb:/backup/"
```

dann kann man ein komplettes komprimiertes Backup der Datenbank ganz einfach mit dem folgenden Befehl ausführen:

```bash
docker exec CONTAINERNAME influx backup --compression gzip /backup/backup1
```

Das Backup file sollte dann im Ordner unter /backup/backup1 abgelegt werden. Den Backup Status sieht man im STDOUT. Der Restore kann dann durch den restore Parameter ausgeführt werden:

#### Restore aller Timeseries

```bash
docker exec CONTAINERNAME influx restore /backup/backup1
```

#### Restore und alles ersetzen

```bash
docker exec CONTAINERNAME influx restore /backup/backup1
```

## Weitere Informationen

* <https://success.docker.com/article/backup-restore-best-practices>
* <https://gist.github.com/spalladino/6d981f7b33f6e0afe6bb>
* <https://blog.ssdnodes.com/blog/docker-backup-volumes/>
* <https://bobcares.com/blog/docker-backup/2/>
