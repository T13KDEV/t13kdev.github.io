---
title: Verwenden von launchd zur Ausführung von Skripten in macOS
description: >
  Obwohl es nach wie vor in macOS enthalten ist und auch noch unterstützt wird, ist crond keine von Apple empfohlene Lösung mehr.
image: 
  path: https://i.ibb.co/LpKx24H/ae5ebe8f5cce.jpg
sitemap: true
categories: [Apple]
tag: [macOS]
comments: false
---

## Warum launchd

Obwohl **cron** nach wie vor in macOS enthalten ist und auch noch unterstützt wird, ist **crond** keine von Apple empfohlene Lösung für das automatische Ausführen von Skripten mehr. Der offizielle Nachfolger auf macOS heißt **launchd**.

Cron-Jobs funktionieren nach wie vor wie wir es auch von allen anderen UX Systemen kennen. Die systemweite Installation von Cron-Jobs ändert allerdings eine gemeinsam genutzte Ressource (die crontab Datei) und sollte deshalb auch nicht automatisch über ein Skript angepasst werden. Diesen Nachteil hat **launchd** nicht, da hier lediglich Agents bzw. Daemons als XML Beschreibung hinzugefügt und auch einfach wieder gelöscht werden können. Ob ihr jetzt auf **launchd** umstellt oder weiter bei **crond** bleibt, ist euch überlassen. In den weiteren Plänen von Apple spielt **cron** aber keine Rolle mehr.

## Skripte schreiben

Um Agenten oder Daemons über launchd auszuführen, sollten wir zuerst die entsprechende Skripte erstellen, die dann über **launchd** regelmäßig ausgeführt werden sollen. Hier ändert sich im Vergleich zu **crond** nichts. Die gebräuchlichste Skriptsprache auf macOS ist bash, es geht aber natürlich auch mit Python, Perl oder natürlich auch Powershell (wird übrigens am besten über [HomeBrew][1] installiert).

> Wenn in einem Skript andere System Kommandos verwendet werden sollen, unbedingt darauf achten, dass die Pfade zu den Binaries mit angegeben werden. Aus einem `diskutil list` muss dann im Skript ein `/usr/sbin/diskutil list` werden. Unter **System Settings** -> **Privacy & Security** -> **Full Disk Access** kann es auch nicht Schaden, entsprechende Berechtigungen für die Tools zu vergeben, die z.B. auf die Disks zugreifen sollen.
{: .prompt-tip }

Unsere Skripte können wie bei **crond** überall im Filesystem liegen (`chmod +x` nicht vergessen), ich würde sie aber an einem Ort sammeln z.B. unter `/usr/local/scripts`. Die Job Beschreibungen **_müssen_** aber an zwei verschiedenen Orten gespeichert werden, je nachdem, ob sie als Agenten oder Daemons ausgeführt werden sollen:

* Job Beschreibung in Form von .plist Dateien, die als Agenten im Namen des angemeldeten Benutzers agieren sollen, müssen in `~/Library/LaunchAgents` gespeichert werden.
* Umgekehrt gehören Job Beschreibungen, die als Daemon fungieren und unabhängig vom angemeldeten Benutzer systemweit, also mit root Berechtigungen arbeiten, in `/Library/LaunchDaemons`.

Immer daran denken, dass Agenten keine Root-Rechte haben und daher keine Aufgaben ausführen können, die einen tiefen Systemzugriff erfordern. Daemons hingegen laufen mit Root-Rechten und können Aufgaben ausführen, die das gesamte System betreffen.

> Ein schlecht geschriebenes und angreifbares Skript als Daemon kann als Einfallstor für weitere böse Jungs dienen.
{: .prompt-warning }

## Job Definitionen

Skripte in **launchd** werden durch Job Definitionen ausgelöst, die als .plist Dateien in den o.g. Verzeichnissen gespeichert werden. Diese XML-Dateien geben dem Job einen Namen, spezifizieren das Skript, das gestartet werden soll, und geben an, wann es ausgeführt werden soll. Sobald das Skript geschrieben wurde, erstellen wir eine Auftragsdefinition, die das Skript zum richtigen Zeitpunkt und im gewünschten Interval startet. Eine solche Auftragsdefinition sieht etwa wie folgt aus:

```xml
<!-- file: 'local.thomas.mybackup.plist' -->
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
  <dict>
    <key>Label</key>
    <string>local.thomas.mybackup</string>
    <key>Program</key>
    <string>/Users/thomas/Scripts/rsync_backup.sh</string>
    <key>StartCalendarInterval</key>
    <dict>
        <key>Day</key>
        <integer>17</integer>
        <key>Hour</key>
        <integer>1</integer>
        <key>Minute</key>
        <integer>15</integer>
        <key>Month</key>
        <integer>9</integer>
        <key>Weekday</key>
        <integer>0</integer>
    </dict>
  </dict>
</plist>
```

**Beispiel:** .plist Datei
{:.figcaption}

Der Inhalt wird dann in eine Textdatei mit der Erweiterung **.plist** im richtigen Verzeichnis gespeichert (siehe oben).

Die Auftragsbeschreibung besteht aus einigen wichtigen Teilen (einfach das Beispiel oben anpassen):

* **Label:** der Name des Auftrags innerhalb von **launchd**. Er muss für jeden Job eindeutig sein. Diese werden in umgekehrter Domänenschreibweise geschrieben, und "local" ist eine gute Domäne für private Agenten.

* **Program:** der vollständige Pfad des Skripts, das durch diese Jobbeschreibung gestartet wird.

* **RunAtLoad:** beschreibt, wann das Skript ausgeführt werden soll. Hier gibt es einige verschiedene Optionen:

* **RunAtLoad:** wird ausgeführt, sobald die Job-Definition geladen ist. Wird nur einmal pro Ladevorgang ausgeführt.
* **StartInterval:** Startet den Job alle n Sekunden.
* **StartCalendarInterval:** Der Auftrag wird zu einem bestimmten Zeitpunkt und Datum ausgeführt.

> Das o.g, Beispiel führt das Skript **/Users/thomas/Scripts/rsync_backup.sh**  am 17. Tag um 1 Stunden und 15 Minuten, jeden 9. Monat (also September) am 0. Wochentag (also Sonntag) aus. Kurz gesagt das Backup wird immer um 1:15am gemacht, wenn der 17. September auf einen Sonntag fällt.
{: .prompt-tip }

Richtig komplex wird es erst, wenn wir zum Beispiel einen Task an jedem Werktag um 1:15am ausführen möchten. Hier zeigt sich dann, dass auch XML so seine Schwächen hat. Für einen Eintrag in der crontab Datei würde dafür folgender Eintrag reichen `15 1 * * 1-5 /Users/thomas/Scripts/rsync_backup.sh > /dev/null 2>&1`.

In der entsprechenden .plist Datei für launchd müssen wir ein Array anlegen und jeden Tag einzeln mit einem `<dict>` Element aufführen.

```xml
<!-- file: 'local.thomas.mybackup.plist' -->
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
  <dict>
    <key>Label</key>
    <string>local.thomas.rsyncbackup</string>
    <key>Program</key>
    <string>/Users/thomas/Scripts/rsync_backup.sh</string>
    <key>StartCalendarInterval</key>
    <array>
      <dict>
        <key>Weekday</key>
        <integer>1</integer>
        <key>Hour</key>
        <integer>1</integer>
        <key>Minute</key>
        <integer>15</integer>
      </dict>
     <!-- und so weiter mit Weekday 2 - 4 -->
      <dict>
        <key>Weekday</key>
        <integer>5</integer>
        <key>Hour</key>
        <integer>1</integer>
        <key>Minute</key>
        <integer>15</integer>
      </dict>
    </array>
  </dict>
</plist>
```

**Beispiel:** Task jeden Werktag ausführen
{:.figcaption}

**Vorteil:** Wir können jede Jobausführung einzeln konfigurieren. \
**Nachteil:** Viel Schreibarbeit und Fehleranfälligkeit. Am besten einen XML Editor mit Syntaxprüfung benutzen.

Wahrscheinlich auch der Grund, warum ich bei Quick & Dirty Tasks derzeit lieber noch bei **cron** bleibe.

Sobald wir aber entsprechende Skripte erstellt und den entsprechenden Agenten an der richtigen Stelle gespeichert haben, müssen wir ihn in **launchctl** laden. Dies wird dann in Zukunft bei jeder Anmeldung automatisch geschehen.

Um zu sehen, was gerade in **launchctl** läuft, können wir `launchctl list` im Terminal verwenden. Die Ausgabe kann natürlich mit `grep` gefiltert werden, da sonst die Liste etwas länger ist.

```~console
launchctl  list | grep local.mybackup
```

Um jetzt den erstellten Agent oder Daemon zu verwenden, müssen wir ihn in `launchctl` laden, das passiert mit dem folgenden Kommando:

```shell
# Agent
launchctl load ~/Library/LaunchAgents/local.mybackup.plist

# Daemon
launchctl load /Library/LaunchDaemons/local.mybackup.plist
```

Um den Agent bzw. Daemon wieder zu enfernen, wird folgendes Kommando verwendet:

```shell
launchctl  unload ~/Library/LaunchAgents/local.mybackup.plist
```

Wenn wir eine Auftragsdefinition in `launchctl` geladen haben, wird der Agent bzw. Daemon in die **launchd** Warteschlange gestellt und zu dem Zeitpunkt ausgeführt, der in den Startbedingungen angegeben ist. Das interessante dabei ist, dass wir unsere Job Definition problemlos in den entsprechenden Ordnern speichern und auch jederzeit ändern können. Solange sie nicht in `launchctl` geladen werden, werden sie auch nicht ausgeführt. Das macht es etwas flexibler als das mit `cron` und der crontab Datei möglich ist.

Wenn wir ein Skript auf jeden Fall sofort uber `launchd` ausführen wollen (natürlich können wir das Skript selbst auch manuell im Terminal ausführen), können wir den Befehl **start** verwenden:

```shell
launchctl start local.mybackup
```

Dieser Befehl nimmt das Label des Auftrags als Parameter und funktioniert natürlich nur, wenn der Auftrag auch bereits in `launchctl` geladen wurde. Das macht das testen von Skripten über launchd einfacher, denn oft verhalten sich Skripte anders wenn sie direkt im Benutzerkontext ausgeführt werden.

## Weiterführende Links

* [Mac crontab: Creating macOS startup jobs with crontab, er, launchd][2]
* [MacOS launchd plist StartInterval and StartCalendarInterval examples][3]
* [MacOS ‘launchd’ examples (launchd plist example files)][4]
* [Apple Developer - Daemons and Services Programming Guide][5]

[1]: https://blog.kngstn.eu/article/2022-05-15-homebrew/
[2]: https://alvinalexander.com/mac-os-x/mac-osx-startup-crontab-launchd-jobs/
[3]: https://alvinalexander.com/mac-os-x/launchd-plist-examples-startinterval-startcalendarinterval/
[4]: https://alvinalexander.com/mac-os-x/launchd-examples-launchd-plist-file-examples-mac/
[5]: https://developer.apple.com/library/archive/documentation/MacOSX/Conceptual/BPSystemStartup/Chapters/ScheduledJobs.html#//apple_ref/doc/uid/10000172i-CH1-SW2
