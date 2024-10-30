---
title: Ubuntu SSH Zugang absichern
description: >
  Container oder VPS Linux Systeme sind schnelle bereitgestellt, aber sind diese auch sicher?
image: 
  path: https://i.ibb.co/SQLVscz/7a51928aaac5.jpg
sitemap: true
categories: [Linux]
tag: [SSH, Security]
comments: false
---

## SSH Grundkonfiguration

### Generieren von SSH Keys

Es gibt hosting Anbieter bei denen man SSH-Keys direkt bei der Einrichtung eines Servers generieren kann oder gar seine eigenen Keys hochladen kann. 

SSH Keys generieren:

```bash
ssh-keygen -t rsa -b 4096 -f azure-ssh
```

 ![](https://i.ibb.co/g9vp8Hn/2024-10-28-15-44-16-025.png)

Durch den oben stehenden Befehl, haben wir ein Schlüsselpaar generiert. Dabei ist `azure-ssh` der private Key, dieser sollte mit niemanden geteilt werden. Der zweite Key `azure-ssh.pub` ist der Public key, welchen wir später auf unserem Server hinterlegen. 

### Anmelden mit dem User root und Passwort

Als nächstes melden wir uns mit dem `root` User auf dem Server an, um einen eigenen User für weitere Zugriffe einzurichten. Bei einer frischen Ubuntu installation wird in der Regel bereits ein solcher Benutzer erstellt. Falls nicht, können wir auch einen weiteren erstellen.

```bash
adduser thomas
usermod -aG sudo thomas
su - thomas
```

Danach müssen wir noch den zuvor erstellten **Public Key** mit dem Benutzer verbinden. 

In das Home Verzeichnis des Benutzer und folgendes ausführen:

```bash
mkdir ~/.ssh
echo '<Inhalt von azure-ssh.pub>' > ~/.ssh/authorized_keys
chmod 600 ~/.ssh/authorized_keys
```

Die Anmeldung per **Private Key** sollte jetzt bereits funktionieren

```bash
ssh thomas@<HOSTNAME oder IP> -i ~/.ssh/azure-ssh
```

### OpenSSH Konfiguration anpassen

Wir passen ebenfalls einige Grundeinstellungen in der OpenSSH Konfigurationsdatei an, auch hier machen wir aber vorher wieder ein Backup der Datei.

```bash
sudo cp /etc/ssh/sshd_config /etc/ssh/sshd_config.bak
sudo nano /etc/ssh/sshd_config
```

In dem File setzen wir jetzt folgende Parameter bzw. fügen sie ggf. auch neu hinzu.

| Settings                                                 | Description                                                                                                                                                                                                       |
| -------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **LogLevel VERBOSE**                                     | Gibt die Ausführlichkeitsstufe an, die bei der Protokollierung von Meldungen des SSH-Daemons verwendet wird.                                                                                                      |
| **PermitRootLogin no**                                   | Gibt an, ob sich root über SSH anmelden kann.                                                                                                                                                                     |
| **MaxAuthTries 3**                                       | Gibt die maximale Anzahl der zulässigen Authentifizierungsversuche pro Verbindung an.                                                                                                                             |
| **MaxSessions 5**                                        | Gibt die maximale Anzahl offener Shell-, Anmelde- oder Subsystem-Sitzungen (z. B. SFTP) an, die pro Netzwerkverbindung zulässig sind.                                                                             |
| **HostbasedAuthentication no**                           | Gibt an, ob die rhosts- oder /etc/hosts.equiv-Authentifizierung zusammen mit einer erfolgreichen Client-Host-Authentifizierung mit öffentlichem Schlüssel zulässig ist (hostbasierte Authentifizierung).          |
| **PasswordAuthentication no**                            | Gibt an, ob eine Passwortauthentifizierung zulässig ist.                                                                                                                                                          |
| **PermitEmptyPasswords no**                              | Wenn die Kennwortauthentifizierung erlaubt ist, gibt sie an, ob der Server die Anmeldung bei Konten mit leeren Kennwortzeichenfolgen erlaubt.                                                                     |
| **ChallengeResponseAuthentication yes**                  | Gibt an, ob die Challenge-Response-Authentifizierung zulässig ist.                                                                                                                                                |
| **KbdInteractiveAuthentication yes**                     | Legt fest, ob die tastaturinteraktive Authentifizierung erlaubt ist.                                                                                                                                              |
| **UsePAM yes**                                           | Gibt an, ob PAM-Module für die Authentifizierung verwendet werden sollen.                                                                                                                                         |
| **X11Forwarding no**                                     | Gibt an, ob die X11-Weiterleitung erlaubt ist.                                                                                                                                                                    |
| **ClientAliveInterval 600**                              | Legt ein Timeout-Intervall in Sekunden fest, nach dem der SSH-Daemon, wenn keine Daten vom Client empfangen wurden, eine Nachricht über den verschlüsselten Kanal sendet, um eine Antwort vom Client anzufordern. |
| **ClientAliveCountMax 0**                                | Legt die Anzahl der Client-Alive-Nachrichten fest, die gesendet werden dürfen, ohne dass der SSH-Daemon eine Nachricht vom Client zurückerhält.                                                                   |
| **AllowUsers**                                           | Wenn angegeben ist die Anmeldung nur für Benutzernamen zulässig, die angegeben wurden.                                                                                                                            |
| **Protocol 2**                                           | Gibt die Verwendung des neueren und sichereren Protokolls an.                                                                                                                                                     |
| **AuthenticationMethods publickey,keyboard-interactive** | Gibt die Authentifizierungsmethoden an, die erfolgreich abgeschlossen werden müssen, damit ein Benutzer Zugang erhält.                                                                                            |

### Updaten des Systems

Bevor wir nun mit dem absichern fortfahren, bringen wir das System ersteinmal auf den neusten Stand.

```bash
sudo apt update && sudo apt upgrade
```

## Fail2Ban

Im zweiten Schritt installieren und konfigurieren wir Fail2Ban.

```none
sudo apt install fail2ban
```

Nach der Installation sichern wir erst einmal die default config Datei, falls wir irgend etwas falsch machen sollten, können wir jederzeit auf die Datei zurückgreifen.

```bash
sudo cp /etc/fail2ban/jail.conf /etc/fail2ban/jail.conf.bak
sudo nano /etc/fail2ban/jail.conf
```

In der Datei finden wir die auskommentierte Zeile # \[sshd\]. Darunter setzen wir dann folgende Einträge.

```bash
# [sshd]
enabled = true
bantime = 4w
maxretry = 3
```

Damit aktivieren wir das monitoring für den SSH-Server, setzen die Banzeit auf 4 Wochen und erlauben maximal 3 Versuche.

 ![](https://i.ibb.co/n1vP0F1/2024-10-28-16-00-02.png)

## 2FA konfigurieren

2 Faktor wird mit dem Google Authenticator ermöglicht. Dafür brauchen wir das [Google Authenticator PAM Module](https://github.com/google/google-authenticator-libpam)

```bash
sudo apt install libpam-google-authenticator
google-authenticator
```

```bash
Do you want authentication tokens to be time-based (y/n) y

Warning: pasting the following URL into your browser exposes the OTP secret to Google:
  https://www.google.com/chart?chs=200x200&chld=M|0&cht=qr&chl=otpauth://xxxxxxxxxx

   [ ---- QR Code ---- ]

Your new secret key is: ***************
Enter code from app (-1 to skip):
```

Der QR Code muss mit der 2 Faktor App gescannt bzw. der Secret Key kopiert (KeepassX, Bitwarden) werden. Anschließend erhalten wir noch Backupcodes, welche wir aufbewahren sollten.

```bash
Code confirmed
Your emergency scratch codes are:
  21323478
  43822347
  60232018
  73234726
  45456791

Do you want me to update your "/home/jusec/.google_authenticator" file? (y/n) y

Do you want to disallow multiple uses of the same authentication
token? This restricts you to one login about every 30s, but it increases
your chances to notice or even prevent man-in-the-middle attacks (y/n) y

By default, a new token is generated every 30 seconds by the mobile app.
In order to compensate for possible time-skew between the client and the server,
we allow an extra token before and after the current time. This allows for a
time skew of up to 30 seconds between authentication server and client. If you
experience problems with poor time synchronization, you can increase the window
from its default size of 3 permitted codes (one previous code, the current
code, the next code) to 17 permitted codes (the 8 previous codes, the current
code, and the 8 next codes). This will permit for a time skew of up to 4 minutes
between client and server.
Do you want to do so? (y/n) n

If the computer that you are logging into isn't hardened against brute-force
login attempts, you can enable rate-limiting for the authentication module.
By default, this limits attackers to no more than 3 login attempts every 30s.
Do you want to enable rate-limiting? (y/n) y
```

Als nächstes müssen wir das PAM Modul für den SSH-Daemon anpassen um 2FA nutzen zu können. Dafür machen wir als erstes wieder ein Backup der Datei und bearbeiten die Datei.

```bash
sudo cp /etc/pam.d/sshd /etc/pam.d/sshd.bak
sudo nano /etc/pam.d/sshd
```

Wir setzen ein `#` vor die Zeile `@include common-auth` um sie auszukommentieren. Außerdem fügen wir 2 weitere Zeilen am ende hinzu.

```bash
#@include common-auth

...

auth required pam_google_authenticator.so
auth required pam_permit.so
```

Zu guter letzt, starten wir den SSh Daemon neu.

```bash
sudo service ssh restart
```

Fertig! ab sofort kann sich auf dem Server nur noch mit dem angegebene User angemeldet werden. Außerdem wird der private-SSH-Key benötigt und der Code aus er 2 Faktor App.
