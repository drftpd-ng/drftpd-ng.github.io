> Applies to DrFTPD 4.0.x.

# First Login

## Default account

On first start the master creates one account:

| | |
|---|---|
| User | `drftpd` |
| Password | `drftpd` |
| Primary group | `drftpd` |
| Secondary group | `siteop` |
| Allowed IP masks | `*@127.0.0.1` and `*@0:0:0:0:0:0:0:1` |

Because of the IP masks, this account can only log in from the master machine itself. Connect to `127.0.0.1` on port 2121 (`master.port` in `master.conf`).

By default `perms.conf` rejects unencrypted connections (`userrejectinsecure *`, `denydiruncrypted *`, `denydatauncrypted *`). Use an FTP client with explicit FTPS (AUTH TLS) and accept the certificate, see [SSL/TLS](SSL-TLS.md).

## Change the password immediately

Log in and run:

```
SITE PASSWD <new-password>
```

An admin (`siteop` or group admin) can set another user's password with:

```
SITE CHPASS <user> <new-password>
```

## Let yourself in from elsewhere

Add an IP mask to the account (or to your own user) so you can log in from another address:

```
SITE ADDIP drftpd *@10.0.0.5
```

Once your own account works remotely, you can remove the `127.0.0.1` masks if you want. Check how IP masks are managed in [Users and Groups](Users-and-Groups.md).

## Add a user

```
SITE ADDUSER <user> <password> [<ident@ip> ...]
```

Allowed for `=gadmin` and `=siteop` as shipped. Group administration and the other user commands (`SITE CHGRP`, `SITE ADDGROUP`, `SITE GADDUSER`, and more) are listed in [Site Commands](Site-Commands.md). Defaults for new users (ratio, logins, idle time, and so on) come from `defaultuser.conf`, see [Users and Groups](Users-and-Groups.md). Who may run which command is set in `config/commands/ftp/core.conf` and `perms.conf`, see [Permissions](Permissions.md).

## Next

Your site has no storage until a slave is connected: [Adding Slaves](Adding-Slaves.md).
