> Applies to DrFTPD 4.0.x.

# Requirements

## Master

- **JDK 17** (Oracle JDK or OpenJDK). The root `pom.xml` compiles with `<release>17</release>`, so an older JDK cannot build it, and a JRE alone cannot build it either.
- **Apache Maven**, to build DrFTPD.
- **Git**, to fetch the source (you can also download a release archive from https://github.com/drftpd-ng/drftpd/releases).

## Slaves

- **Java SE or OpenJDK 17**.
- **MediaInfo (CLI)**, from https://mediaarea.net/en/MediaInfo. The mediainfo plugin runs the `mediainfo` executable on the slave and expects it in the `PATH`. See [Mediainfo](Mediainfo.md).

You can build once on any machine and copy the `runtime/slave` directory to your slaves, or build on every machine.

## Operating systems

- **Linux**: the reference platform. `master.sh`, `slave.sh` and `genkey.sh` are POSIX shell scripts. See [Installation on Linux](Installation-Linux.md).
- **Windows**: supported through `master.bat`, `slave.bat` and `genkey.bat`. See [Installation on Windows](Installation-Windows.md).
- Neither the master nor the slave needs root or Administrator privileges. Run them as a normal user.
- The slave needs exclusive write access to its storage directory (the `slave.root.N` paths). Do not modify files there from outside DrFTPD, or the slave goes out of sync with the master's file list.

## Memory

The shipped start scripts use `-Xms3G -Xmx3G` for the master and `-Xms1G -Xmx1G` for a slave, both with `-XX:+UseG1GC`. Adjust `JVM_OPTS` in the script to match your machine and the size of your file list. See [FAQ](FAQ.md).

## Network

- The master listens on one FTP port (default 2121) and one slave port (default 1099).
- Each slave opens an outgoing connection to the master's slave port.
- See [Firewalls and Passive Mode](Firewalls-and-Passive-Mode.md) for the data ports.
