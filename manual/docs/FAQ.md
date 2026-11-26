> Applies to DrFTPD 4.0.x.

# FAQ

### Why do I get "500 You need to use a client supporting PRET (PRE Transfer) to use PASV"?

Your client sent `PASV` without `PRET`. The master has to know the transfer before it can pick a slave, so passive mode needs a client that sends `PRET` first. Use a client with PRET support ([list](https://github.com/drftpd-ng/drftpd/wiki/Clients-with-PRET-support)) or switch the client to active mode. See [Firewalls and Passive Mode](Firewalls-and-Passive-Mode.md) and [PRET Specifications](https://github.com/drftpd-ng/drftpd/wiki/PRET-Specifications).

### My client cannot log in or list directories, it says encryption is required.

The shipped `perms.conf` has `userrejectinsecure *`, `denydiruncrypted *` and `denydatauncrypted *`: everything must be TLS. Use explicit FTPS (AUTH TLS) and accept the certificate, or relax those lines in `perms.conf` ([Permissions](Permissions.md)). The master also needs `config/drftpd.key`, see [SSL/TLS](SSL-TLS.md).

### The default `drftpd` user cannot log in from another machine.

The auto-created account only has the IP masks `*@127.0.0.1` and `*@0:0:0:0:0:0:0:1`. Add a mask with `SITE ADDIP drftpd *@10.0.0.5`. See [First Login](First-Login.md).

### What does "533 Requested action not taken. You must upload sfv first." mean?

The zipscript plugin enforces SFV-first. In a directory without an `.sfv`, only files with an allowed extension (`allowedexts` in `zipscript.conf`, which by default includes nfo, txt, jpg, diz, avi, vob, m2ts, mkv, mp4, zip, m3u) or without an extension (`sfvfirst.allownoext`) may be uploaded before the SFV. Upload the `.sfv` first, or change `sfvfirst.required`, `sfvfirst.pathcheck`, `sfvfirst.pathignore` or `sfvfirst.users`. See [Zipscript](Zipscript.md).

### What does "533 Requested action not taken. File not found in sfv." mean?

The directory has an `.sfv` and the file you try to upload is not listed in it (and not an allowed extension). See [Zipscript](Zipscript.md).

### My slave does not come online.

Check on the master log (`logs/master.log`, `logs/remoteslave.log`):

- `Slave <name> does not exist, use "site addslave"`: add it with `SITE ADDSLAVE <name>`. The name is case sensitive and must equal `slave.name` in `slave.conf`.
- `<ip> is not a valid mask for <name>`: add a mask with `SITE SLAVE <name> ADDMASK <ident@ip>`.
- `fingerprint mismatch, refusing connection`: the slave has a pinned certificate that does not match, see [SSL/TLS](SSL-TLS.md).
- TLS problems: check `master.ssl.*` and `slavemanager.ssl.*`, and that a `drftpd.key` exists on both sides.
- Check `master.host` and `master.bindport` in `slave.conf`, and that the master slave port (default 1099) is open.

Details: [Adding Slaves](Adding-Slaves.md).

### Files from an offline slave are listed but cannot be downloaded.

That is by design: `files.offline.enabled=true` (default) in `master.conf` shows files that are only on offline slaves. Set it to `false` to hide them. When the slave comes back and remerges, the files are available again.

### Why does a slave take so long to start?

On connect the slave sends its full file list to the master (remerge), which is slow for big storage. See `partial.remerge.mode`, `enableremergechecksums` in `master.conf` and `threadedremerge`, `threadedthreads`, `rootCollectionThreads`, `concurrent.root.iteration` in `slave.conf`. See [Introduction](Introduction.md).

### Can I add, move or delete files directly on a slave's disk?

No. The master's VFS is the authoritative list, and the slave needs exclusive write access to its storage. Changes outside DrFTPD put the slave out of sync with the master.

### How much memory does DrFTPD need? How do I change `-Xmx`?

The start scripts set `-Xms3G -Xmx3G -XX:+UseG1GC` for the master and `-Xms1G -Xmx1G -XX:+UseG1GC` for slaves (`JVM_OPTS` in `master.sh`/`slave.sh`, `master.bat`/`slave.bat`). The scripts say to check that `Xmx` suits your master. Edit `JVM_OPTS`. In Docker, set `JAVA_TOOL_OPTIONS`, see [Docker](Docker.md).

### Times are wrong, how do I set the time zone?

There is no time zone key in `master.conf`. DrFTPD runs in the JVM, which uses the operating system's time zone by default. To force one, add the standard JVM option `-Duser.timezone=<zone>` to `JVM_OPTS` in the start script. Note that MLST/MLSD timestamps are always sent in GMT, as the protocol requires.

### The master refuses to start with "already mapped".

A command is defined twice in your command configuration. This happens after an upgrade when an old customised `config/commands/ftp/core.conf` still defines commands that moved to another file such as `request.conf.dist`. Remove the duplicate from your `.conf`, or delete the `.conf` to use the `.dist`. See [Upgrading](Upgrading.md).

### Sizes show as `51200.0MB` or `50.0GB`, can I change the unit?

Yes, with the JVM options `-Dbytes.maxunit=<K|M|G|T|P|E>` and `-Dbytes.binary=true` (KiB/MiB instead of KB/MB). They affect IRC announcements and FTP replies. See [Building from Source](Building-from-Source.md).

### Do I need root?

No. Neither master nor slave needs root or Administrator rights. Use a normal user and ports above 1024.

### Where do I report a bug or ask a question?

See [Getting Help](Getting-Help.md).
