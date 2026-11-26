> Applies to DrFTPD 4.0.x.

# Installation on Linux

Check [Requirements](Requirements.md) first (JDK 17, Maven, git; mediainfo on slaves).

## 1. Get the source

Use a tag for a stable install (replace the tag with the release you want):

```
git clone https://github.com/drftpd-ng/drftpd.git
cd drftpd
git checkout tags/4.0.11
```

Or download a release archive from https://github.com/drftpd-ng/drftpd/releases and extract it. The `master` branch is the unstable development version.

## 2. Build

```
mvn clean
mvn validate
mvn install
```

`mvn clean` is only needed when you upgrade an existing source tree. The build creates a `runtime` directory:

```
runtime/
  master/    config/, lib/, build/, logs/, master.sh, master.bat
  slave/     config/, lib/, build/, logs/, slave.sh, slave.bat
```

Dependencies go to `lib/`, DrFTPD itself to `build/`. The start scripts use the class path `lib/*:build/*`.

## 3. Master

Work from `runtime/master`.

1. **Configuration.** Config files ship as `*.dist`. DrFTPD uses `name.conf` when it exists and falls back to `name.conf.dist`, so you only need to copy a `.dist` file to `.conf` when you want to change it. At minimum you will edit `master.conf`:
   ```
   cd runtime/master/config
   cp master.conf.dist master.conf
   ```
   Plugin configuration lives in `config/plugins/` and follows the same rule. See [Master Configuration](Master-Configuration.md).
2. **TLS key (required).** Run `./genkey.sh` from the source root and put the resulting `drftpd.key` into `runtime/master/config/`. The master-slave connection is always TLS and uses this keystore (`slavemanager.ssl.keystore.path`), and without it FTPS for clients is disabled. See [SSL/TLS](SSL-TLS.md).
3. **Start.** From `runtime/master`:
   ```
   chmod +x master.sh
   ./master.sh
   ```
   `master.sh` runs `java -Xms3G -Xmx3G -XX:+UseG1GC ... org.drftpd.master.Master` with `-Dlog4j.configurationFile=config/log4j2-master.xml`. Start it from `runtime/master`, because the paths are relative.

## 4. Slave

Work from `runtime/slave` (on the slave machine).

1. Copy `config/slave.conf.dist` to `config/slave.conf` and set at least `slave.name`, `master.host` and `slave.root.1`. See [Adding Slaves](Adding-Slaves.md).
2. Put a `drftpd.key` in `config/`: the connection to the master is always TLS and uses it (`master.ssl.keystore.path`), and so do TLS data transfers (see [SSL/TLS](SSL-TLS.md)).
3. Start with `./slave.sh` (`-Xms1G -Xmx1G` by default).

## 5. Run as a service (systemd)

The header comments of `master.sh` and `slave.sh` contain a ready-to-use template. It is a **user** unit. For the master, save this as `~/.config/systemd/user/drftpd-master.service`:

```
[Unit]
Description=DrFTPD Master
After=network.target
StartLimitIntervalSec=500
StartLimitBurst=5

[Service]
Restart=always
RestartSec=5s
Type=simple
UMask=007
WorkingDirectory=/home/mastersitename/master
ExecStart=/home/mastersitename/master/master.sh

[Install]
WantedBy=default.target
```

For a slave use the same layout with `Description=DrFTPD Slave`, your slave directory and `slave.sh`. The slave template also has an optional `Requires=mnt-ftp1.mount mnt-ftp2.mount` line (commented out) to start the slave only after its disks are mounted.

Then:

```
systemctl daemon-reload --user
systemctl enable --user drftpd-master.service
systemctl start --user drftpd-master.service
systemctl stop --user drftpd-master.service
```

(Use `drftpd-slave.service` for slaves.) The scripts also mention `jsvc` (Apache Commons Daemon) as an alternative wrapper.

## 6. First start

Connect with an FTP client to port 2121 and log in, see [First Login](First-Login.md). Then add your slaves: [Adding Slaves](Adding-Slaves.md).

By default `perms.conf.dist` refuses unencrypted logins and data connections, so your client must use explicit FTPS (AUTH TLS) unless you change `userrejectinsecure`, `denydiruncrypted` and `denydatauncrypted`. See [SSL/TLS](SSL-TLS.md) and the [FAQ](FAQ.md).
