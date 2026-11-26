> Applies to DrFTPD 4.0.x.

# Adding Slaves

A slave connects to the master, so you set it up on both sides: register the slave name and its allowed address on the master, and configure `slave.conf` on the slave.

## 1. On the master

Log in as a siteop and register the slave. The name must match `slave.name` in `slave.conf` (names are case sensitive).

```
SITE ADDSLAVE <slavename>
SITE SLAVE <slavename> ADDMASK <ident@host>
```

`ADDMASK` takes an `ident@ip-or-host` mask. Only connections that match a mask of that slave are accepted. Examples:

```
SITE SLAVE slave1 ADDMASK *@10.0.0.20
SITE SLAVE slave1 ADDMASK *@slave1.example.org
```

You can remove a mask with `SITE SLAVE <slavename> DELMASK <ident@host>`. A connection from an unmatched address is refused and the master logs `... is not a valid mask for <slave>`. A slave name the master does not know is refused with the log message `Slave <name> does not exist, use "site addslave"`.

Related commands (see [Site Commands](Site-Commands.md)): `SITE DELSLAVE`, `SITE KICKSLAVE`, `SITE SLAVE <name> SET|UNSET <key> <value>`, `SITE SLAVE <name> SHUTDOWN`, `SITE SLAVE <name> QUEUES`, `SITE SLAVEUPTIME`.

## 2. On the slave: `config/slave.conf`

Copy `slave.conf.dist` to `slave.conf` and edit these keys (names exactly as in the dist file):

```
# What you set the name as with "site addslave <slavename>" on the master.
slave.name=slave1

# Root to serve contents from, is automatically created.
slave.root.1=/srv/drftpd/site

# Master host/ip and the port the master listens on for slaves
master.host=10.0.0.10
master.bindport=1099
```

- `slave.name`: default is `drftpd`. It must be unique among your slaves.
- `slave.root.1`: the storage directory, created automatically. The default is `site` (relative to the slave directory).
- `master.host`: default `localhost`.
- `master.bindport`: must equal `master.bindport` in the master's `master.conf` (default 1099).
- `slave.timeout=60000`: how long the slave waits for the master, in ms. The dist file says the timeout on the master side should be equal, set with `SITE SLAVE <slavename> SET timeout <value>`.

### Multiple roots

You can stripe files over several disks without RAID by adding more roots:

```
slave.root.1=/mnt/disk1/ftp
slave.root.2=/mnt/disk2/ftp
```

The slave chooses a root using the `diskselection` plugin (`diskselection.conf`, default `org.drftpd.slave.diskselection.filter.DiskSelectionFilter`). Do not change it unless you know why. Related keys: `rootCollectionThreads`, `concurrent.root.iteration`. The full key list is in [Slave Configuration](Slave-Configuration.md).

### Data ports and bind address

Optional keys `slave.portfrom`, `slave.portto` (passive port range on the slave) and `bind.ip`. See [Firewalls and Passive Mode](Firewalls-and-Passive-Mode.md).

### TLS

The slave talks to the master over TLS (`master.ssl.*`). With the shipped defaults you need a `drftpd.key` in the slave's `config/` directory (copy the one from the master, or generate one per slave to enable certificate pinning). See [SSL/TLS](SSL-TLS.md).

## 3. Start the slave

```
./slave.sh        # Linux
slave.bat         # Windows
```

Each time the slave connects it sends its file list to the master (remerge, which can be partial, see `partial.remerge.mode` in [Master Configuration](Master-Configuration.md)). The first full remerge of a big storage can take some time.

## 4. Check

On the master:

```
SITE SLAVES
```

lists every slave and whether it is online. `SITE SLAVE <slavename>` shows the properties stored for one slave. If the slave does not show up as online, read the logs in `runtime/master/logs` on the master (`master.log`, `remoteslave.log`) and `logs/slave.log` in the slave directory, and see the [FAQ](FAQ.md).

You can filter the output of `SITE SLAVES` with `default.slave.output` in `master.conf` (matched against the start of the name).

## Uploads

Which slave receives a new upload is decided by the slave selection configuration, see [Slave Selection](Slave-Selection.md).
