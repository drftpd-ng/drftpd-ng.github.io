> Applies to DrFTPD 4.0.x.

# Master Configuration

The master reads its settings from files in the `config/` folder of the master installation. This page
describes `master.conf` key by key, then lists the other files of the core master and what they are for.
Settings of the slaves are on [Slave Configuration](Slave-Configuration.md).

## How the files are loaded

Every configuration file ships as `NAME.conf.dist` (and the plugin ones in `config/plugins/`).

- If `NAME.conf` exists, the master uses it. Otherwise it uses `NAME.conf.dist`.
- Copy the `.dist` file to `NAME.conf` and edit the copy. The `.dist` files are replaced when you upgrade, your
  `.conf` files are not. See [Upgrading](Upgrading.md) for what to do when a new version adds keys.
- `master.conf` and the other property style files use `key=value`. Lines that start with `#` are comments.
  Values are used exactly as written, so leave out spaces around `=` and do not add trailing spaces.
- Booleans are written `true` or `false`.
- If the environment variable `DRFTPD_CONFIG_PATH` is set, the master looks for `config/...` below that
  folder instead of the current folder (the [Docker](Docker.md) image uses this).
- `SITE RELOAD` re-reads `master.conf`, `perms.conf`, the sections, slave selection and command files, and
  the files of plugins that react to a reload. Keys that the master reads at the moment it needs them (for example
  `files.offline.enabled`, `delete.upload.on.abort`, `homedir`, `disk.commit.delay`) take effect after a reload.
  Keys that are read once at start (ports, the manager classes, TLS settings, the key store) need a restart of the
  master. If in doubt, restart.

## master.conf

### Network

| Key | Default | Meaning |
| --- | --- | --- |
| `master.bindport` | `1099` | TCP port where the master listens for slave connections. Slaves use the same number as `master.bindport` in their `slave.conf` |
| `master.port` | `2121` | TCP port where the master listens for FTP clients |
| `master.ip` | not set (all interfaces) | The IP address to bind the FTP port to. Leave it commented out to listen on all interfaces. It must be a valid address, otherwise a warning is logged and all interfaces are used |
| `ident.lookup.timeout` | `2000` | Milliseconds to wait for the ident answer (port 113) of a connecting client, used when a host mask contains an ident (see [Users and Groups](Users-and-Groups.md)). A higher value finds more idents, but many FTP clients give up after about 5 seconds. If the key is missing or not a number, 2000 is used |

Passive mode ports and addresses (`pasv_ports`, `pasv_addr`) are not set here but in `perms.conf`, see
[Permissions](Permissions.md) and [Firewalls and Passive Mode](Firewalls-and-Passive-Mode.md).

### File system and transfers

| Key | Default in the dist file | Meaning |
| --- | --- | --- |
| `files.offline.enabled` | `true` | Show files whose slaves are offline in directory listings. If the key is missing the code uses `true`. The files are listed with a marker in the name defined by the theme |
| `delete.collision.files` | `false` | What to do when a file found on a slave during a remerge has the same name as a directory or link that already exists in the master's file system. `true`: delete that file from the slave. `false`: keep it. Note: if the key is missing, the code uses `true` |
| `delete.upload.on.abort` | `false` | Delete a partially uploaded file from the slave when the upload is aborted. With `false` a post hook (for example the zipscript) may still delete it |
| `disk.commit.delay` | `10000` | Milliseconds changes to users, groups and similar data may stay in memory before they are written to disk. Higher values reduce disk I/O at the cost of losing more if the master crashes. Suggested: `10000` for a hard disk, `1000` for an SSD, `500` for a RAM disk |
| `european.cal` | `false` | `true`: weeks start on Monday (and week numbers follow ISO 8601) for weekly resets, weekly credits, top lists and dated sections. `false`: weeks start on Sunday |

### Plugins and managers (advanced)

Do not change these unless you write your own plugin.

| Key | Default | Meaning |
| --- | --- | --- |
| `usermanager` | `org.drftpd.master.usermanager.encryptedjavabeans.EncryptedBeanUserManager` | The class that stores users and groups. See [Users and Groups](Users-and-Groups.md) |
| `commandmanager` | `org.drftpd.master.commands.StandardCommandManager` | The class that runs commands |
| `slaveselection` | `org.drftpd.master.slaveselection.filter.SlaveSelectionManager` | The slave selection class, configured by the `config/slaveselection-*.conf` files, see [Slave Selection](Slave-Selection.md) |
| `sectionmanager` | `org.drftpd.master.sections.def.SectionManager` | `...sections.def.SectionManager` treats every top level directory as a section, `org.drftpd.master.sections.conf.SectionManager` reads `sections.conf`. See [Sections](Sections.md) |
| `indexingengine` | `org.drftpd.master.indexation.LuceneEngine` | The index engine, see [Index and Search](Index-and-Search.md) |

These five keys must be present: the master stops at start if one is missing or the class cannot be created.

### TLS for FTP clients

These settings apply to the TLS connections of FTP users. See [SSL/TLS](SSL-TLS.md) for the whole picture.

| Key | Default | Meaning |
| --- | --- | --- |
| `cipher.whitelist.N` | `.*_(POLY1305_SHA256\|GCM_SHA384\|GCM_SHA256).*` for `N=1` | Regular expressions (matched against the whole cipher suite name) of cipher suites to enable. `N` counts from 1 without gaps. If there is no usable whitelist the default set of the Java runtime is used |
| `cipher.blacklist.N` | not set | Regular expressions of cipher suites to remove. The blacklist wins over the whitelist, and can be used without a whitelist to remove suites from the default set |
| `protocol.N` | `TLSv1.2` and `TLSv1.3` | Enabled TLS protocol versions, one per line, `N` counting from 1. Unsupported names are ignored with a warning. If none are left the Java default is used |

### Slave connections: TLS

The connection between the master and its slaves is always encrypted. These keys, all starting with
`slavemanager.ssl.`, configure the master side. The matching slave side is `master.ssl.*` in `slave.conf`.

| Key | Default in the dist file | Meaning |
| --- | --- | --- |
| `slavemanager.ssl.supported_protocols.N` | `TLSv1.3` | TLS versions allowed |
| `slavemanager.ssl.cipher_suites.N` | `TLS_AES_128_GCM_SHA256`, `TLS_AES_256_GCM_SHA384`, `TLS_CHACHA20_POLY1305_SHA256` | Cipher suites allowed |
| `slavemanager.ssl.verification_mode` | `none` | How the certificate presented by a slave is verified |
| `slavemanager.ssl.client_authentication` | `optional` | Whether the master asks slaves for a certificate. `optional` or `required` are needed to pin a slave certificate (see below). Keep one of them if you pin |
| `slavemanager.ssl.keystore.path` | `config/drftpd.key` | The key store holding the master's certificate (create it with `genkey.sh` / `genkey.bat`, see below) |
| `slavemanager.ssl.keystore.password` | `drftpd` | Password of that key store |
| `slavemanager.ssl.certificate_authorities`, `slavemanager.ssl.truststore.path`, `slavemanager.ssl.certificate`, `slavemanager.ssl.key` | commented out, empty | Optional trust store, certificate authorities or PEM files, see the comments in the dist file |

To accept only one certificate from a slave, pin its fingerprint with `SITE SLAVE <slavename> SET fingerprint SHA256:<fingerprint>`
(remove with `SITE SLAVE <slavename> UNSET fingerprint`). Pinning only works if each slave has its own key store: generate one
per slave instead of copying the same key everywhere. The master logs each slave's certificate fingerprint when it connects.
See [Adding Slaves](Adding-Slaves.md) and [SSL/TLS](SSL-TLS.md).

### Slave remerge

When a slave connects, it tells the master which files it has (a "remerge").

| Key | Default | Meaning |
| --- | --- | --- |
| `partial.remerge.mode` | `off` | `off`: a full remerge on every connect. `connect`: only files newer than the slave's last connect are remerged. `disconnect`: only files newer than the last disconnect (the dist file says this mode is currently broken). `instant`: the slave goes online immediately and a full remerge runs in the background (the dist file calls it stable but under development). The key must be present. A slave can force a full remerge with `ignore.partialremerge` in its `slave.conf` |
| `enableremergechecksums` | `false` | `true`: when a remerged file has no checksum in the master's file system, ask the slave for it |
| `remerge.pause.threshold` | `250` | While a slave is remerging, the master pauses the slave when this many remerge messages are queued |
| `remerge.resume.threshold` | `50` | and resumes it when the queue has shrunk to this number. Do not change the two thresholds unless you know what you are doing |

### Users, pre and home directories

| Key | Default | Meaning |
| --- | --- | --- |
| `default.pre.user` | `drftpd` | On `SITE PRE`, directories and files are given to this user. If you rename the `drftpd` user, change it here |
| `default.pre.group` | `drftpd` | and to this group |
| `homedir` | not set | The directory a user lands in after login. Empty or not set: `/`. `${user}` is replaced by the user name and `${group}` by the primary group name. A per user value is set with `SITE CHANGE <user> homedir <path\|none>` |
| `homedir.create` | `false` | `true`: create a missing home directory at login. This uses the same permission checks as `MKD`, so the user needs the `makedir` permission for it in `perms.conf`. No MKD event is sent and MKD command hooks (dupecheck, nukefilter, tvmaze, ...) do not run. If the user lacks the permission, nothing is created and the user lands in `/`. The parent directory must already exist and be visible to the user |
| `default.slave.output` | not set | A filter for `SITE SLAVES` and `SITE REMERGEQUEUE` when no argument is given: only slaves whose name starts with this text (lower case) are shown. Example: `default.slave.output=race` |

The home directory is not a chroot and gives no permissions. See [Users and Groups](Users-and-Groups.md) and [Permissions](Permissions.md).

## Other core master files

| File (in `config/`) | Purpose |
| --- | --- |
| `perms.conf` | Permissions, connection limits, PASV settings. See [Permissions](Permissions.md) |
| `defaultuser.conf`, `defaultgroup.conf`, `encryptedbeanuser.conf`, `securepass.conf`, `ipsecurity.conf`, `expireduser.conf` | New user and group defaults, password hashing and account policies. See [Users and Groups](Users-and-Groups.md) |
| `sections.conf` | Sections, used with the configurable section manager. See [Sections](Sections.md) |
| `slaveselection-up.conf`, `slaveselection-down.conf`, `slaveselection-jobup.conf`, `slaveselection-jobdown.conf` (and `slaveselection-old.conf`) | How the master picks a slave for uploads, downloads and jobs. See [Slave Selection](Slave-Selection.md) |
| `commands/ftp/*.conf`, `commands/irc/*.conf` | Command definitions and their permissions. See [Site Commands](Site-Commands.md) and [IRC Commands](IRC-Commands.md) |
| `themes/text/*.txt` and the theme files | Texts shown to users. See [Themes](Themes.md) |
| `plugins/*.conf` | Settings of each plugin, see [Plugins](Plugins.md) |
| `log4j2-master.xml` | Logging. See [Log Levels](Log-Levels.md) |
| `drftpd.key` | The TLS key store. `genkey.sh` / `genkey.bat` (in the installation folder) create a `drftpd.key` in the current folder, a PKCS12 key store with an EC key valid for 365 days; put it in `config/` |

### indexation.conf

Settings of the Lucene search index (see [Index and Search](Index-and-Search.md)). The values in the middle column are in the dist file; the
code uses the values in the right column if a key is missing.

| Key | Dist file | If missing | Meaning |
| --- | --- | --- | --- |
| `max_hits` | `100` | `50` | The most results a search returns. `0` returns everything, which is not recommended: it costs CPU, I/O and memory |
| `maxdocs_buffer` | `120` | `-1` | Advanced. How many documents Lucene buffers in memory before flushing to disk. Raising it reduces I/O but uses more memory |
| `max_rambuffer` | `16` | `16` | Advanced. The RAM, in MB, buffered before flushing. Whichever of the two buffer limits is reached first causes a flush |
| `optimize_interval` | `15` | `15` | Minutes between index optimisations. Optimising speeds up searching but uses resources. Busy sites may use 30 or more |
| `native_locking` | `true` | `true` | Advanced. Use native file locking. Set `false` if the index is on a network file system |
| `do_backups` | `true` | `true` | Make backups of the index |
| `backup_interval` | `120` | `120` | Minutes between backups. Do not set it low, 60 or more is advised |
| `max_backups` | `2` | `2` | Number of backups to keep, older ones are deleted. `0` disables backups |

### fairness.conf

Used by `SITE FAIRNESS`, which divides each user's uploaded bytes by their downloaded bytes. All keys are commented out in
the dist file, the values shown are the defaults:

| Key | Default | Meaning |
| --- | --- | --- |
| `fairness.ok` | `0.5` | A ratio below this is shown as "bad", from here up to `fairness.good` as "ok" |
| `fairness.good` | `1.0` | From here up to `fairness.awesome` the user is "good" |
| `fairness.awesome` | `2.0` | From here up the user is "awesome" |

A user with no uploads or no downloads has no fairness ratio.

### Files not described here

The other `.conf.dist` files of the master and their keys belong to features and plugins; see the page of each feature
([Plugins](Plugins.md), [Traffic](Traffic.md), [Statistics](Statistics.md), [Trial](Trial.md) and so on).
