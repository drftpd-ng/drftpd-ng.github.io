> Applies to DrFTPD 4.0.x.

# Installation on Windows

Check [Requirements](Requirements.md) first. The build machine needs JDK 17, Maven and git, and `java` must be on the `PATH`. Slaves need the MediaInfo CLI (`mediainfo` on the `PATH`).

## 1. Get the source

From a command prompt:

```
git clone https://github.com/drftpd-ng/drftpd.git
cd drftpd
git checkout tags/4.0.11
```

Or download and extract a release archive from https://github.com/drftpd-ng/drftpd/releases.

## 2. Build

```
mvn clean
mvn validate
mvn install
```

(`mvn clean` only on upgrades.) The result is the `runtime\master` and `runtime\slave` directories, each with `config`, `lib`, `build` and its start script.

## 3. Master

In `runtime\master`:

1. Copy `config\master.conf.dist` to `config\master.conf` (and any other `.dist` file you want to change). DrFTPD uses the `.conf` file when present, otherwise the `.dist` file.
2. TLS key (required, the master-slave connection is always TLS): run `genkey.bat` from the source root and move the resulting `drftpd.key` into `runtime\master\config`. See [SSL/TLS](SSL-TLS.md).
3. Start `master.bat`. It runs `java -Xms3G -Xmx3G -XX:+UseG1GC` with class path `lib/*;build/*` and `-Dlog4j.configurationFile=config/log4j2-master.xml`. Start it from inside `runtime\master`.

## 4. Slave

In `runtime\slave`: copy `config\slave.conf.dist` to `config\slave.conf`, set `slave.name`, `master.host` and `slave.root.1` (a root can be a drive, the dist file shows `d:/` as an example for a second root), put a `drftpd.key` in `config\` (required, the connection to the master is always TLS), then start `slave.bat` (`-Xms1G -Xmx1G`). See [Adding Slaves](Adding-Slaves.md).

## 5. Run as a Windows service

The repository does not ship a ready service definition. The header of `master.bat` only mentions a wrapper such as Apache Commons Daemon (`procrun`, https://commons.apache.org/proper/commons-daemon/procrun.html) and lists the service example as "TO DO". The README names `sc.exe` as an option. You have to set up the wrapper yourself: main class `org.drftpd.master.Master` (or `org.drftpd.slave.Slave`), class path `lib/*;build/*`, the master or slave directory as working directory, and the same JVM options as the `.bat` file.

## 6. First start

Connect to port 2121 and log in, see [First Login](First-Login.md). The default `perms.conf.dist` requires FTPS, see [SSL/TLS](SSL-TLS.md).
