> Applies to DrFTPD 4.0.x.

# Slave Configuration

A slave reads its settings from the `config/` folder of the slave installation. The main file is
`slave.conf`; `diskselection.conf` controls which of the slave's root directories receives new files. The
master side is described on [Master Configuration](Master-Configuration.md). How to set up and register a slave is on
[Adding Slaves](Adding-Slaves.md).

## How the files are loaded

- A file shipped as `NAME.conf.dist` is used when there is no `NAME.conf`. Copy the dist file to `NAME.conf` and edit
  the copy, so an upgrade does not overwrite your settings. See [Upgrading](Upgrading.md).
- The files use `key=value`, lines starting with `#` are comments. Write booleans as `true` or `false`.
- If the environment variable `DRFTPD_CONFIG_PATH` is set, the slave looks in `$DRFTPD_CONFIG_PATH/config/` instead of `config/`.
- The slave reads its configuration when it starts. Restart the slave after changing a file.
- The slave writes its log to `logs/slave.log`, see [Log Levels](Log-Levels.md).

## slave.conf

### Identity and master connection

| Key | Default in the dist file | Meaning |
| --- | --- | --- |
| `slave.name` | `drftpd` | The name of this slave. It must be the name you gave with `SITE ADDSLAVE <slavename>` on the master. Required |
| `master.host` | `localhost` | Host name or IP address of the master. Required |
| `master.bindport` | `1099` | Port the master listens on for slaves. It must match `master.bindport` in the master's `master.conf`. Required |
| `slave.timeout` | `60000` | Milliseconds the slave waits for the master before it considers the connection dead. The slave's timeout on the master side must be the same, set it with `SITE SLAVE <slavename> SET timeout <value>`. The code default is `60000` |
| `bind.ip` | not set | Bind the slave to this IP address, useful on machines with several addresses. The connection to the master and the data connections use it. If not set, the primary address is used. An invalid address is logged and ignored |

The address of the slave that the master hands out in `PASV` replies is the address the slave uses to connect to the master,
unless it is changed on the master for this slave. See [Firewalls and Passive Mode](Firewalls-and-Passive-Mode.md).

The connection to the master is always TLS encrypted, with the settings in `master.ssl.*` below. The dist file also contains
`slave.masterSSL=true`; this key is not read by the code in 4.0.x, so it changes nothing.

### Storage roots

| Key | Default in the dist file | Meaning |
| --- | --- | --- |
| `slave.root.N` | `slave.root.1=site` | The directories the slave serves files from. `N` counts from 1 without gaps: the slave reads `slave.root.1`, `slave.root.2`, ... and stops at the first missing number. Each directory is created automatically if needed. Relative paths are relative to the slave folder. Use several roots to spread files over several disks without RAID |
| `maxPathLength` | `4096` | The longest path (in characters) that the slave reports to the master as allowed by its file system |

Example with two disks:

```
slave.root.1=/mnt/disk1/site
slave.root.2=/mnt/disk2/site
```

Which root gets a new file is decided by `diskselection.conf`, see below. Root numbers used there refer to this numbering.

### Transfers

| Key | Default in the dist file | Meaning |
| --- | --- | --- |
| `slave.portfrom`, `slave.portto` | commented out | Port range for passive data connections on this slave, for example `30000` and `33000`. Both must be set. If not set, or not numbers, the operating system picks random ports |
| `enableuploadchecksums` | `true` | Calculate a CRC32 checksum while a file is uploaded |
| `enabledownloadchecksums` | `true` | Calculate a checksum while a file is downloaded |
| `bufferSize` | `0` | Socket buffer size in bytes. `0` lets the operating system decide. The transfer copy buffer is at least 32768 bytes |
| `cipher.whitelist.N`, `cipher.blacklist.N` | `cipher.whitelist.1=.*_(POLY1305_SHA256\|GCM_SHA384\|GCM_SHA256).*` | Cipher suites for encrypted data transfers with FTP clients (not the master connection). Regular expressions, matched against the whole suite name. The blacklist wins. With no usable whitelist the Java default set is used |
| `protocol.N` | `TLSv1.2`, `TLSv1.3` | TLS versions for encrypted data transfers with FTP clients |

See [SSL/TLS](SSL-TLS.md) and [Firewalls and Passive Mode](Firewalls-and-Passive-Mode.md).

### Remerge

When a slave connects it lists its files to the master (a "remerge"). These keys tune that.

| Key | Default in the dist file | Meaning |
| --- | --- | --- |
| `ignore.partialremerge` | `false` | `true`: always do a full remerge when connecting, whatever `partial.remerge.mode` says on the master |
| `threadedremerge` | `true` | Use several threads for the remerge, which makes it faster. Note: if the key is missing, the code uses `false` |
| `threadedthreads` | `0` | Number of remerge threads when `threadedremerge` is on. `0` means the number of CPU cores, minus one when there are more than two cores. A higher number overrides this |
| `rootCollectionThreads` | `3` | How many I/O actions per root run at the same time when collecting files. With one root, 3 actions run at once, with four roots 12. Raise it for ZFS or RAID systems that have one root on many disks |
| `concurrent.root.iteration` | `false` | List all roots at the same time during a remerge. Only worth setting if the slave has more than one root and more than one CPU, otherwise it is probably slower |

### Disk selection

| Key | Default | Meaning |
| --- | --- | --- |
| `diskselection` | `org.drftpd.slave.diskselection.filter.DiskSelectionFilter` | The class that chooses a root for a new file. Do not change it. The filters it uses are set in `diskselection.conf` |

### TLS connection to the master

These keys have the prefix `master.ssl.` and are the slave side of the `slavemanager.ssl.*` keys on the master. Keep them
in step with the master.

| Key | Default in the dist file | Meaning |
| --- | --- | --- |
| `master.ssl.supported_protocols.N` | `TLSv1.3` | TLS versions allowed |
| `master.ssl.cipher_suites.N` | `TLS_AES_128_GCM_SHA256`, `TLS_AES_256_GCM_SHA384`, `TLS_CHACHA20_POLY1305_SHA256` | Cipher suites allowed |
| `master.ssl.verification_mode` | `none` | How the master's certificate is verified |
| `master.ssl.client_authentication` | `optional` | |
| `master.ssl.keystore.path` | `config/drftpd.key` | The key store holding the certificate this slave presents to the master. If the master pins slave certificates (`SITE SLAVE <name> SET fingerprint`), use a key unique to this slave. `genkey.sh` / `genkey.bat` create one |
| `master.ssl.keystore.password` | `drftpd` | Password of that key store |

See [SSL/TLS](SSL-TLS.md) and [Adding Slaves](Adding-Slaves.md).

## diskselection.conf

With several roots (`slave.root.N`), the slave builds a score for each root for every file it receives and uses the
root with the highest score. `diskselection.conf` is a numbered chain of filters, `1.filter`, `2.filter`, ... applied in
order (it stops at the first missing number).

The default file:

```
1.filter=minfreespace
1.minfreespace=5GB
1.assign=all
1.multiplier=1

2.filter=cycle
```

Filters, with their keys (`N` is the position in the chain):

| Filter | Keys | What it does |
| --- | --- | --- |
| `minfreespace` | `N.minfreespace` (required, bytes with unit, for example `5GB`), `N.assign` (default `all`), `N.multiplier` (default `0`) | For each assigned root with less free space than `minfreespace`: if the multiplier is `0` the root is removed from the choice, otherwise its score is lowered by `(minfreespace - free) x multiplier` |
| `cycle` | none | Adds one point to the best scoring root, and on a tie the one that was used longest ago. Put it last |
| `randomspread` | none | Adds one point to a random root, to spread files over all roots |
| `priority` | `N.assign` (required) | Gives fixed scores to roots, for example `N.assign=1+100 2+200` makes root 2 the first choice |
| `matchdir` | `N.match` (required), `N.assign` (required), `N.assume.remove` (default `false`), `N.negate.expression` (default `false`) | If the file's path in the site matches the glob `N.match` (case insensitive, the whole path must match), apply the assign scores. `N.negate.expression=true` applies them to paths that do not match. With `assume.remove=true` roots not mentioned in `assign` are removed |
| `matchdirex` | the same keys as `matchdir` | The same with a regular expression (case insensitive, found anywhere in the path) |

The `N.assign` value is a space separated list (commas are ignored): `all`, or entries `root`, `root+score`, `root-score` or
`root+remove`, where `root` is the number of the `slave.root.N`. `1+100 2+remove` gives root 1 100 points and removes
root 2 from the choice. An entry naming a root that does not exist stops the slave with an error. `all` is not useful for
`matchdir`, `matchdirex` and `priority`. `N.multiplier` accepts a number, or products and quotients such as `2*3` or `1/2`, and `remove` is the same as `0`.

The paths matched by `matchdir` are paths on the site as the master sees them (for example `/PHOTOS/...`), not paths on the disk.
The disk selection must work together with the slave selection on the master, see [Slave Selection](Slave-Selection.md).

Example: keep `/ARCHIVE` on the second disk, everything else prefers the first, never fill a disk below 5 GB:

```
1.filter=minfreespace
1.minfreespace=5GB
1.assign=all
1.multiplier=1

2.filter=matchdir
2.match=/ARCHIVE*
2.assign=1+remove 2+100

3.filter=priority
3.assign=1+10

4.filter=cycle
```

## Plugin settings on the slave

Some plugins have a slave side configuration file in `config/plugins/`. In 4.0.x the only one is for the Speedtest.net plugin,
`config/plugins/speedtest.net.slave.conf`, which sets the number of threads and the payload used when the slave measures its speed.
It is explained on [Speedtest](Speedtest.md). Do not change it unless you know what you are doing.
