!!! note
    This documentation applies to DrFTPD 4.0.x only.

# DrFTPD 4.x Manual

DrFTPD is a distributed FTP daemon written in Java. One **master** handles FTP clients, users, permissions and the virtual file system. Any number of **slaves** store the files and do all the data transfers. A client talks to the master with an ordinary FTP client, and the master tells the slaves what to send or receive. Because the master does not carry file data, you can merge several sites into one: ten 1 Gbit slaves give you 10 Gbit of aggregate bandwidth, although a single transfer never goes above the speed of one slave.

This manual documents DrFTPD **4.0.x** only. Pages about DrFTPD 2.0 and 3 are kept on the old wiki for history, see [Legacy Pages](Legacy-Pages.md). They do not apply to 4.x.

## Start here

1. [Introduction](Introduction.md): how master and slaves fit together
2. [Requirements](Requirements.md): JDK 17, Maven, git, mediainfo
3. Install on [Linux](Installation-Linux.md) or [Windows](Installation-Windows.md), or use [Docker](Docker.md)
4. [First Login](First-Login.md): default account and password change
5. [Adding Slaves](Adding-Slaves.md): connect your first slave
6. [SSL/TLS](SSL-TLS.md) and [Firewalls and Passive Mode](Firewalls-and-Passive-Mode.md)

## Reference

- [Master Configuration](Master-Configuration.md) and [Slave Configuration](Slave-Configuration.md)
- [Site Commands](Site-Commands.md), [IRC Commands](IRC-Commands.md), [Permissions](Permissions.md)
- [Users and Groups](Users-and-Groups.md), [Sections](Sections.md), [Log Levels](Log-Levels.md)
- [Plugins](Plugins.md) overview. Zipscript, sitebot and the other plugins are listed in the sidebar.

## Operations

- [Upgrading](Upgrading.md), [Building from Source](Building-from-Source.md), [FAQ](FAQ.md)

## Getting help

See [Getting Help](Getting-Help.md): IRC on EFnet `#drftpd`, and the GitHub issue tracker.
