> Applies to DrFTPD 4.0.x.

# Log Levels

DrFTPD logs with Apache Log4j 2. The master and each slave have their own log configuration file and
write their own log files into a `logs/` folder. This page lists the files, explains the settings and
shows how to raise the level of a part of the system when you need more detail.

## Configuration files

| Process | Configuration file | Log folder |
| --- | --- | --- |
| Master | `config/log4j2-master.xml` | `logs/` in the master folder |
| Slave | `config/log4j2-slave.xml` | `logs/` in the slave folder |

The start scripts select the file with a Java option:

```
-Dlog4j.configurationFile=config/log4j2-master.xml
-Dlog4j.configurationFile=config/log4j2-slave.xml
```

(`master.sh`/`master.bat` and `slave.sh`/`slave.bat`). If you start DrFTPD without that option, a built in fallback
writes everything at level `WARN` to the console.

Both files contain `monitorInterval="30"`: Log4j checks the file every 30 seconds and applies
changes without a restart. Just save the edited file and wait half a minute.

## Log files

### Master

| File | What goes into it | Logger (package prefix) |
| --- | --- | --- |
| `master.log` | Everything not covered below, and the FTP connection service messages | root, `org.drftpd.master.network.BaseFtpConnection.service` |
| `commands.log` | Messages from command handlers (all `org.drftpd.master.commands.*` classes that have no more specific logger below, for example nuke, dir, login) | `org.drftpd.master.commands` |
| `sysop.log` | User and group management: add, change, delete users, group admins, IP masks | `org.drftpd.master.commands.usermanagement` |
| `remoteslave.log` | Slave connections, remerges and slave management | `org.drftpd.master.slavemanagement.RemoteSlave`, `org.drftpd.master.slavemanagement.SlaveManager` |
| `vfs.log` | The virtual file system | `org.drftpd.master.vfs` |
| `index.log` | The Lucene index and the index commands | `org.drftpd.master.indexation`, `org.drftpd.commands.indexmanager` |
| `events.log` | The internal event bus | `org.bushe.swing.event` |
| `sitebot.log` | The IRC sitebot | `org.drftpd.master.sitebot` |
| `archive.log` | The archive plugin | `org.drftpd.archive.master` |
| `jobmanager.log` | The job manager | `org.drftpd.jobs.master` |
| `imdb.log` | The IMDB plugin | `org.drftpd.commands.imdb.master` |
| `tvmaze.log` | The TVMaze plugin | `org.drftpd.commands.tvmaze.master` |

A message goes to the most specific matching logger, and with `additivity="false"` only to that file. So a message
from the sitebot goes to `sitebot.log` and not to `master.log`.

### Slave

The slave has a single file, `slave.log`, for all its classes.

### Rotation

Each file is rolled over at midnight. The old file is compressed and moved to
`logs/<year>-<month>/<name>-<date>.log.gz`, and compressed logs older than 90 days are deleted.
Note that in the shipped master file the rolled files of `commands.log`, `vfs.log` and `events.log` are named
`index-<date>.log.gz`, and the clean up rules for those three look for different names, so those archives are
not removed automatically after 90 days. Delete old archives by hand if disk space matters.

Each line looks like this:

```
WARN 2025-01-31 12:00:00,123 [thread name] org.drftpd.master.Class - message
```

## Levels

Every logger and the root have a level. A message is written only if its level is at least as high as the
configured level. From the most to the least verbose:

| Level | Meaning |
| --- | --- |
| `ALL` | everything |
| `TRACE` | finest detail |
| `DEBUG` | detailed information for troubleshooting |
| `INFO` | progress and normal events |
| `WARN` | potentially harmful situations. The default for every logger |
| `ERROR` | errors that still allow the program to continue |
| `FATAL` | very severe errors |
| `OFF` | nothing |

Level names are case insensitive in the file.

## Changing a level

Open `config/log4j2-master.xml` (or the slave file) and change the `level` attribute. The shipped file
sets `level="warn"` on the root and on every logger.

**The logger levels override the root level.** Every logger in the master file has its own
`level="warn"`, so changing only `<Root level="warn">` has an effect only on classes that have no
specific logger. To get more detail from a specific area, change that area's logger. Examples:

Log slave connection and remerge details:

```xml
<Logger name="org.drftpd.master.slavemanagement.RemoteSlave" level="debug" additivity="false">
    <AppenderRef ref="remoteslave" />
</Logger>
```

Record who added or changed which user in `sysop.log` (these messages are written at `INFO`, so they are not
recorded at the default `WARN`):

```xml
<Logger name="org.drftpd.master.commands.usermanagement" level="info" additivity="false">
    <AppenderRef ref="sysop" />
</Logger>
```

Debug a class that has no logger of its own (for example the zipscript plugin) by adding one that writes to
an existing file:

```xml
<Logger name="org.drftpd.zipscript" level="debug" additivity="false">
    <AppenderRef ref="root" />
</Logger>
```

For a slave:

```xml
<Logger name="org.drftpd.slave.Slave" level="debug" additivity="false">
    <AppenderRef ref="root"/>
</Logger>
```

Debug and trace logging produce a lot of output and slow the daemon down. Turn them back to `warn` when
you are done, and remember that `master.log` and the other files can then contain IP addresses and user names (see
`hide_ips` in [Permissions](Permissions.md)).

## Tips

- When asking for help ([Getting Help](Getting-Help.md)), attach the relevant part of the log at `WARN` or
  `INFO` level, not a full `DEBUG` log, and remove any private information first.
- If a command fails with "Command execution failed" the stack trace is in `commands.log` (or the
  more specific file for that area, for example `sitebot.log`).
- For a problem that shows up on a slave, check `slave.log` on the slave and `remoteslave.log` on the master.
