---
title: Hetzner x Restic
---

# Hetzner x Restic 
## Einführung
-------------

Hetzner

<img src="../img/hetzner.png" width="120" height="40" style="float:left;">
 
- Ein deutscher Hosting-, Cloud- und Storageanbieter. Als Speicherort der Backups wird Hetzners [Storage Box](https://www.hetzner.com/storage/storage-box) genutzt.  

</br >
</br >

Restic

<img src="../img/restic.png" width="120" height="40" style="float:left;">

- Ein Programm zur Erstellung von Backups. Diese werden automatisch verschlüsselt, inkremmentell erzeugt und dedupliziert. Die zu sichernden Daten werden in einem Restic Repository gespeichert. Das Repository kann entweder lokal oder auf einem entfernten Server angelegt werden.

## Vorbereitung
----------------
Sofern der zu sichernde Server außerhalb der Hetzner Cloud steht, muss im [Hetzner Robot](https://robot.your-server.de/storage) die "externe Erreichbarkeit" aktiviert werden. Für den Zugriff auf die Storage Box wird im weiteren SFTP genutzt. Neben der Anmeldung über Passwort, ist auch die Authentifizierung über SSH-Key möglich.

### SSH-Key generieren

Zunächst wird der SSH-Key generiert und die authorized_keys Datei vorbereitet.
```bash
ssh-keygen
```

Wenn SFTP über SSH-Port 22 genutzt werden soll, muss der öffentliche Schlüssel in das RFC4716-Format umgewandelt werden.
```bash
ssh-keygen -e -f .ssh/id_rsa.pub | grep -v "Comment:" > .ssh/id_rsa_rfc.pub
```

Der öffentliche Schlüssel wird sowohl im OpenSSH- als auch im RFC4716-Format zur Datei storagebox_authorized_keys hinzugefügt. 
```bash
cat .ssh/id_rsa.pub >> storagebox_authorized_keys
cat .ssh/id_rsa_rfc.pub >> storagebox_authorized_keys
``` 

Auf der Storage Box legen wir das Verzeichnis .ssh an und kopieren den Inhalt der storagebox_authorized_keys nach .ssh/authorized_keys. 
```bash
echo -e "mkdir .ssh \n chmod 700 .ssh \n put storagebox_authorized_keys .ssh/authorized_keys \n chmod 600 .ssh/authorized_keys" | sftp u123456@u123456.your-storagebox.de
```

### SSH-Config anlegen
Die SSH-Config verkürzt den Befehl zur Verbindung mit dem SFTP-Server, da hier alle wichtigen Parameter zur Anmeldung mitgegeben werden können.
```bash
vi ~/.ssh/config
```

Und folgenden Inhalt hinzufügen:
```bash
host storagebox 
    HostName u123456.your-storagebox.de
    User u123456
    IdentityFile ~/.ssh/id_rsa
    ServerAliveInterval 60
    ServerAliveCountMax 240
```

### Verzeichnis anlegen
```bash
echo -e "mkdir backup_server_1" | sftp u123456@u123456.your-storagebox.de
```
Hier wird später das Backup Repository des Servers auf der Storage Box abgelegt.

## Restic
---------
### Installation
=== "Debian / Ubuntu"
    ```bash
    apt install restic
    ```
=== "RHEL / AlmaLinux"
    ```bash
    dnf install epel
    dnf update
    dnf install restic
    ```

### Initialisierung
Mit folgendem Befehl wird das Restic-Repository auf der Storage Box initialisiert.
```bash
restic -r sftp:storagebox:/backup_server_1 init
```

### Automatisierung
#### Skript
Unter `/opt/scripts` müssen die folgenden Dateien angelegt werden:

<details>
<summary>restic_env</summary>

```bash
export RESTIC_REPOSITORY=sftp:storagebox:/backup_server_1
export RESTIC_PASSWORD=PASSWORD
```
</details>

<details>
<summary>backup.sh</summary>

```bash
#!/usr/bin/env bash
# This script is intended to be run by a systemd timer

# Exit on failure or pipefail
set -euo pipefail

#Set this to any location you like
BACKUP_PATHS="/var/lib/docker/volumes /opt/docker"

BACKUP_TAG=systemd.timer

# How many backups to keep.
RETENTION_DAYS=7
RETENTION_WEEKS=4
RETENTION_MONTHS=12
RETENTION_YEARS=1

source /opt/scripts/restic_env

# Remove locks in case other stale processes kept them in
restic unlock &
wait $!

#Do the backup

restic --verbose \
       --tag $BACKUP_TAG \
       backup $BACKUP_PATHS &
wait $!

# Remove old Backups

restic forget \
       --verbose \
       --tag $BACKUP_TAG \
       --prune \
       --keep-daily $RETENTION_DAYS \
       --keep-weekly $RETENTION_WEEKS \
       --keep-monthly $RETENTION_MONTHS \
       --keep-yearly $RETENTION_YEARS &
wait $!

# Check if everything is fine
restic check &
wait $!

echo "Backup done!"
```
</details>

Nachdem die backup.sh angelegt wurde, muss das Skript ausführbar gemacht werden.
```bash
chmod u+x /opt/scripts/backup.sh
```

#### Daemon
Unter `/etc/systemd/system/` werden die folgende Dienste angelegt:

=== "backup.timer"
    ```bash
    [Unit]
    Description=Backup on schedule
    
    [Timer]
    OnCalendar=daily
    Persistent=true
    
    [Install]
    WantedBy=timers.target
    ```
=== "backup.service"
    ```bash
    [Unit]
    Description=Backup with restic
    
    [Service]
    Type=simple
    Nice=10
    ExecStart=/opt/scripts/backup.sh
    #$HOME must be set for restic to find /root/.cache/restic/
    Environment="HOME=/root"
    ```

Abschließend muss der systemd Manager neu geladen und backup.timer in den automatischen Start aufgenommen werden.
```bash
systemctl daemon-reload
systemctl enable backup.timer
systemctl start backup.timer
```

Jetzt startet backup.timer den Dienst backup.service täglich gegen Mitternacht. Darüber hinaus kann backup service auch manuell gestartet werden.
```bash
systemctl start backup.service
# Prüfen, ob Fehler aufgetreten sind
journalctl -u backup.service
# Prüfen, ob Backup erfolgreich
restic -r sftp:storagebox:/backup_server_1 snapshots
```

## Backups wiederherstellen

[Restic Restore](https://restic.readthedocs.io/en/latest/050_restore.html)

Um einen einzelnen Ordner aus dem letzten Backup wiederherzustellen, kann zum Beispiel folgender Befehl genutzt werden:
```Bash
restic -r sftp:storagebox:/backup_server_1 restore latest --target /tmp/restore --include /opt/docker/folder
```

## Quellen
----------
- https://docs.hetzner.com/de/robot/storage-box
- https://restic.readthedocs.io/en/stable/
- https://blog.bithive.space/post/automatic-backups-with-restic/
