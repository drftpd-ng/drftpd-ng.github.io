> Applies to DrFTPD 4.0.x.

# Introduction

DrFTPD is a distributed FTP daemon. It is not a normal FTP server that reads and writes files itself: it is split into one **master** and any number of **slaves**.

## The pieces

| | Master | Slave |
|---|---|---|
| Java main class | `org.drftpd.master.Master` | `org.drftpd.slave.Slave` |
| Directory after build | `runtime/master` | `runtime/slave` |
| Speaks FTP with clients | yes (default port 2121) | no, knows nothing about FTP |
| Knows users, groups, permissions | yes | no |
| Stores the virtual file system (VFS) | yes (the authoritative copy) | no |
| Stores the actual files | no (it can run a local slave if you want) | yes (`slave.root.N` directories) |
| Does file transfers | no | yes, all of them |
| Runs plugins and site commands | yes | only small helpers (for example checksum or mediainfo work) |

The master uses very little bandwidth: only the FTP control connections, directory listings and the instructions to the slaves pass through it. Neither the master nor the slaves need root privileges.

## The virtual file system

The master keeps a file list (the VFS) that records which slaves hold which files, plus all metadata: size, owner user and group, last modified time, checksum and so on. This list is the authoritative source. The slaves only store the bytes.

Consequences:

- A file can exist on several slaves for redundancy or more bandwidth.
- Because the master has no files locally, you cannot change the VFS by touching files on a slave's disk. The slave needs exclusive write access to its storage area, otherwise it goes out of sync with the master and errors occur.
- If you merge 10 sites of 1 Gbit you get a 10 x 1 Gbit site, not a 10 Gbit site: a single transfer never exceeds one slave's speed.

## Slave connection and remerge

A slave connects **to** the master (outgoing connection to `master.bindport`, default 1099, on the host given by `master.host`). The slave name in `slave.conf` must have been added on the master with `SITE ADDSLAVE`, and the slave's IP must match a mask added with `SITE SLAVE <name> ADDMASK <ident@host>`. Otherwise the master refuses it (see [Adding Slaves](Adding-Slaves.md)). The connection is TLS encrypted, see [SSL/TLS](SSL-TLS.md).

When a slave comes online it walks through its storage roots and sends its complete file list to the master. This is called a **remerge**. The master merges the list into its VFS: it adds files it did not know, and removes the slave from files the slave no longer has. The VFS is therefore rebuilt from the slaves the first time they connect after a fresh install.

Remerge behaviour is tuned in `master.conf` (`partial.remerge.mode`, `enableremergechecksums`, `remerge.pause.threshold`, `remerge.resume.threshold`) and `slave.conf` (`ignore.partialremerge`, `threadedremerge`, `threadedthreads`, `rootCollectionThreads`, `concurrent.root.iteration`). See [Master Configuration](Master-Configuration.md) and [Slave Configuration](Slave-Configuration.md). With `partial.remerge.mode=off` (the default) a full remerge is done on every connect.

While a slave is offline its files are still listed (`files.offline.enabled=true` by default) but cannot be transferred.

## How a transfer flows

1. The client connects to the master and logs in. All commands go to the master. It checks permissions and the VFS.
2. For an **upload**, the master chooses a slave (see [Slave Selection](Slave-Selection.md)). For a **download**, it picks one of the slaves that has the file.
3. The master instructs that slave to open a data connection and, in **passive mode**, tells the client the slave's address in its `227 Entering Passive Mode` reply. The client then connects directly to the slave. The file data never passes through the master. In **active mode** the master tells the slave to connect to the address the client gave with `PORT`.
4. When the transfer finishes, the slave reports size, checksum and speed to the master, which updates the VFS and runs hooks (zipscript, stats, IRC announcements, ...).

FXP (site-to-site) transfers between the slave and another FTP server work like this too, since the client only relays addresses between the two servers.

### PRET and passive mode

Standard FTP says `PASV` is sent before the client names the file. In DrFTPD the master must know the file (or direction) before it can pick the right slave, so it needs to know the transfer in advance. The `PRET` command (PRE Transfer) provides that: the client sends `PRET <command> <file>`, then `PASV`. If a client sends `PASV` without `PRET` the master answers `500 You need to use a client supporting PRET (PRE Transfer) to use PASV`. Active mode (`PORT`) needs no PRET. For a list of clients, see [Clients with PRET support](https://github.com/drftpd-ng/drftpd/wiki/Clients-with-PRET-support) and [PRET Specifications](https://github.com/drftpd-ng/drftpd/wiki/PRET-Specifications). Details on ports and NAT: [Firewalls and Passive Mode](Firewalls-and-Passive-Mode.md).

Directory listings are different: they are served by the master itself on a local data connection, using the master's `pasv_ports` range from `perms.conf`.

## What runs where

- **Master**: FTP server, command handling (`config/commands/`), `perms.conf`, user and group files (`userdata`), the VFS, the index (Lucene by default), all master-side plugins and the IRC sitebot, if you use it.
- **Slave**: file storage, data transfers, checksums, and slave-side plugin parts (for example mediainfo and the zipscript helpers). It needs `mediainfo` installed if you use the mediainfo plugin.

Plugins have a master part and, in some cases, a slave part. See [Plugins](Plugins.md).

## Next steps

[Requirements](Requirements.md), then [Installation on Linux](Installation-Linux.md), [Installation on Windows](Installation-Windows.md) or [Docker](Docker.md).
