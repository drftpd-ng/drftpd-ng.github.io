> Applies to DrFTPD 4.0.x.

# Docker

The Docker support is marked **experimental** in the README. Images are built automatically for every git tag starting with `4.` by the workflow `.github/workflows/build-and-push-docker-images.yml`, for `linux/amd64` only.

## Images

Published on the GitHub Container Registry (https://github.com/drftpd-ng/drftpd/pkgs/container/drftpd):

| Image | Content |
|---|---|
| `ghcr.io/drftpd-ng/drftpd:master-latest` | the master, latest tagged release |
| `ghcr.io/drftpd-ng/drftpd:master-<tag>` | the master of one release tag, for example `master-4.0.11` |
| `ghcr.io/drftpd-ng/drftpd:slave-latest` | a slave, latest tagged release |
| `ghcr.io/drftpd-ng/drftpd:slave-<tag>` | a slave of one release tag |

Both are targets (`drftpd-master`, `drftpd-slave`) of the same `Dockerfile`, based on `amazoncorretto:17-alpine3.19`. The slave image also installs `mediainfo`. Pin a `-<tag>` image in production instead of `-latest`.

## How the images are laid out

- The images run **rootless** as user `drftpd` (UID 1000). Every mounted volume must be writable by UID 1000.
- Master: files in `/home/drftpd/master` (working directory). Entry point `java -classpath lib/*:build/* org.drftpd.master.Master`.
- Slave: files in `/home/drftpd/slave`. Entry point `java -classpath lib/*:build/* org.drftpd.slave.Slave`.
- The shipped config is copied to `config.dist` inside the image, and an `initconfig` script is created.
- Default JVM options (environment `JAVA_TOOL_OPTIONS`):
  - master: `-Djdk.tls.acknowledgeCloseNotify=true -Xms3G -Xmx3G -XX:+UseG1GC -Dlog4j.configurationFile=config/log4j2-master.xml`
  - slave: `-Djdk.tls.acknowledgeCloseNotify=true -Xms1G -Xmx1G -XX:+UseG1GC -Dlog4j.configurationFile=config/log4j2-slave.xml`

  Check that the `-Xmx` value fits your machine and override `JAVA_TOOL_OPTIONS` if needed.

## `initconfig`: initialise the config directory

On the first start, mount an empty directory over `config` and run the container with the entry point `sh -c ./initconfig`. It does `cp -rn config.dist/* config/`, i.e. it copies the default files without overwriting existing ones. On the **master** it also creates `config/drftpd.key` (EC key, valid 365 days) if that file does not exist. The slave's `initconfig` does not create a key: copy a `drftpd.key` into the slave's `config`, see [SSL/TLS](SSL-TLS.md).

Then edit the config as for a regular installation ([Master Configuration](Master-Configuration.md), [Slave Configuration](Slave-Configuration.md)) and start the container with its default entry point. You can also mount existing DrFTPD directories.

## Compose examples

These come from the README. There are no `docker-compose` files in the repository, so adapt these snippets to your own compose file.

Master:

```
  drftpd-master:
    image: ghcr.io/drftpd-ng/drftpd:master-latest
    environment:
      - JAVA_TOOL_OPTIONS=-Djdk.tls.acknowledgeCloseNotify=true -Xms3G -Xmx3G -XX:+UseG1GC -Dlog4j.configurationFile=config/log4j2-master.xml
    ports:
      - 1099:1099/tcp
      - 2121:2121/tcp
      - 50000-60000:50000-60000/tcp
    # Note: UID 1000 must have write access
    volumes:
      - /path/to/drftpd/master/config:/home/drftpd/master/config
      - /path/to/drftpd/master/index.bkp:/home/drftpd/master/index.bkp
      - /path/to/drftpd/master/logs:/home/drftpd/master/logs
      - /path/to/drftpd/master/userdata:/home/drftpd/master/userdata
    # Optional: initialize the config directory with the default config files
    #entrypoint: sh -c ./initconfig
```

Slave:

```
  drftpd-slave:
    image: ghcr.io/drftpd-ng/drftpd:slave-latest
    environment:
      - JAVA_TOOL_OPTIONS=-Djdk.tls.acknowledgeCloseNotify=true -Xms1G -Xmx1G -XX:+UseG1GC -Dlog4j.configurationFile=config/log4j2-slave.xml
    ports:
      - 30000-33000:30000-33000/tcp
    # Note: UID 1000 must have write access
    volumes:
      - /path/to/drftpd/slave/config:/home/drftpd/slave/config
      - /path/to/drftpd/slave/logs:/home/drftpd/slave/logs
      - /path/to/drftpd/slave/site:/home/drftpd/slave/site
    # Optional: initialize the config directory with the default config files
    #entrypoint: sh -c ./initconfig
```

What must match in the configuration:

- Master `perms.conf`: set `pasv_ports 50000-60000` (and `pasv_addr` if needed) to match the published range.
- Slave `slave.conf`: set `slave.portfrom=30000` and `slave.portto=33000` to match the published range, `master.host` to the master's address as seen from the slave container, and `slave.root.1=site` (the default, mapped to the `site` volume).
- On the master, remember to run `SITE ADDSLAVE` and `SITE SLAVE <name> ADDMASK` with the address the master sees for the slave container (see [Adding Slaves](Adding-Slaves.md)).
- For slaves behind NAT, see [Firewalls and Passive Mode](Firewalls-and-Passive-Mode.md).

## Build your own image

The `Dockerfile` expects the repository checked out in a directory named `drftpd-source` inside the build context (it does `COPY drftpd-source /drftpd-source/`), builds it with Maven, and writes the commit id to `HEAD.version`:

```
git clone https://github.com/drftpd-ng/drftpd.git drftpd-source
docker build -f drftpd-source/Dockerfile --target drftpd-master -t my-drftpd:master .
docker build -f drftpd-source/Dockerfile --target drftpd-slave  -t my-drftpd:slave  .
```

## Size formatting option

Add `-Dbytes.maxunit=M` or `-Dbytes.binary=true` to `JAVA_TOOL_OPTIONS` if you want them, see [FAQ](FAQ.md).

## Upgrading containers

The new image brings new `.dist` files, but your mounted `config` is kept. Merge changes yourself, see [Upgrading](Upgrading.md) (especially the command configuration files).
