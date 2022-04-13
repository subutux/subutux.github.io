---
title: "How I do backups"
date: 2022-01-16T21:34:43+01:00
draft: true
tags:
 - backup
 - pi
 - restic
 - truenas
 - truenas scale
---

For a good 3 years, I've been using restic as my main backup solution. Now that I bought myself a new server (more on that in another post)
It's about time I document my backup flow.

## Restic

[Restic](https://restic.net/) is a single binary, git inspired, lightweight backup software.

From their main site:

> Restic is a modern backup program that can back up your files:
> - from Linux, BSD, Mac and Windows
> - to many different storage types, including self-hosted and online services
> - easily, being a single executable that you can run without a server or complex setup
> - effectively, only transferring the parts that actually changed in the files you back up
> - securely, by careful use of cryptography in every part of the process
> - verifiably, enabling you to make sure that your files can be restored when needed
> - freely - restic is entirely free to use and completely open source

That just thicks all my boxes!

TODO why?

## Running restic

TODO Explain wrapper script

`/usr/local/sbin/restic_`

```sh
#!/bin/bash
set -o allexport
source /etc/restic/$1.conf
set +o allexport
shift
restic $@
```

Exampe configuration file:

#### `/etc/restic/zoo.conf`
```sh
BACKUP_PATHS="/opt /usr/src /etc /root /home"
BACKUP_EXCLUDES="--exclude-if-present .exclude_from_backup"
RETENTION_DAYS=7
RETENTION_WEEKS=4
RETENTION_MONTHS=6
RETENTION_YEARS=3
RESTIC_REPOSITORY=rest:http://user:pass@192.168.32.6:9012/
RESTIC_PASSWORD=password
```


### Systemd

TODO explain service and timers

#### `/etc/systemd/system/backup@.service`
```ini
[Unit]
Description=Restic backup service
[Service]
Type=oneshot
ExecStart=/usr/bin/restic backup --json --verbose --one-file-system --tag systemd.timer $BACKUP_EXCLUDES $BACKUP_PATHS
ExecStartPost=/usr/bin/restic forget --verbose --tag systemd.timer --group-by "paths,tags" --keep-daily $RETENTION_DAYS --keep-weekly $RETENTION_WEEKS --keep-monthly $RETENTION_MONTHS --keep-yearly $RETENTION_YEARS
EnvironmentFile=/etc/restic/%i.conf
```

#### `/etc/systemd/system/backup@.timer`
```ini
[Unit]
Description=Backup %i with restic on schedule

[Timer]
OnCalendar=daily
Persistent=true

[Install]
WantedBy=timers.target
```
## Backblaze b2

TODO explain why and why not

## Rest Server


TODO explain why and why not



