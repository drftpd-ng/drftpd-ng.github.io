> Applies to DrFTPD 4.0.x.

# Site Commands

This page lists every FTP and `SITE` command that DrFTPD 4.0.x defines in its default
configuration, with the usage text, the default permission and any extra options. It is
generated from the `*.conf.dist` files shipped with the source (196 commands in
13 files), so it matches the defaults of a fresh install.

Run `SITE HELP` on your site for a one line description of every command, and
`SITE HELP <command>` for the usage text. Both show the `help` and `help.specific`
values given below. They come from the same configuration files, so if you changed a
file, your site shows your text.

## Where the commands are configured

Command definitions live in `config/commands/ftp/`, one file per feature:
`core.conf.dist`, `archive.conf.dist`, `jobs.conf.dist` and so on. On startup the master
loads every `X.conf` it finds in that folder (sub folders included). If there is no
`X.conf` for a given `X.conf.dist`, it loads the `.dist` file instead.

- To change a command, copy `X.conf.dist` to `X.conf` and edit the copy. Never edit the `.dist` file,
  it is replaced on upgrade.
- An `X.conf` replaces its `X.conf.dist` completely. After an upgrade, commands added in newer
  versions do not appear in your `X.conf` until you merge them by hand, or delete your `X.conf`
  to go back to the defaults. See [Upgrading](Upgrading.md).
- A command defined twice (for example one that moved to another file, with your old copy still in
  `core.conf`) stops the master at startup with an "already mapped" error. Remove the duplicate.
- Command names are case insensitive. Use `SITE RELOAD` or restart the master to reload.

## Block format

```
SITE ADDUSER {
plugin org.drftpd.master.commands.usermanagement
class UserManagementHandler
method doSITE_ADDUSER
perms =gadmin =siteop
help Add a new user.
help.specific ${command} <user> <password> [<ident@ip#1> ...]
}
```

- `plugin`, `class` and `method` select the Java code that runs the command. Leave them alone.
  The master refuses to load a command that lacks `plugin`, `class`, `method` or `perms`.
- `perms` says who may run the command (see below).
- `help` is the one line description shown by `SITE HELP`.
- `help.specific` is the usage text. `${command}` is replaced by the command name. The key can be
  repeated: every line is shown, in order.
- Any other key is an option read by the command itself. They are listed under "Options" for each
  command below.

## Options used by core commands

| Option | Used by | Meaning |
| --- | --- | --- |
| `file` | `SITE AFFILS`, `SITE BNC`, `SITE RULES`, `SITE WELCOME` | Name of the text file, inside `config/themes/text/`, that the command prints (see [Themes](Themes.md)). `X.txt` is used when it exists, otherwise `X.txt.dist` |
| `nohelp` | `SITE HELP` | Comma separated command names that `SITE HELP` leaves out of its list |
| `slavenames` | `LIST`, `STAT` | `true` or `false` (default `false`). When true, listings show the name of the slave holding a file |
| `default`, `max`, `filtered_sections` | `SITE NEW` | Number of directories shown when none is given (default 5), the largest number a user may ask for (default 25), and a space separated list of sections left out of the listing |
| `search` | `SITE NUKE`, `SITE UNNUKE` | `true` (default) or `false`. When true, a nuke given a bare directory name from the site root looks the directory up in the index |

## `perms` syntax

`perms` is a space separated list, checked from left to right. The first entry that matches the
user decides.

| Entry | Matches |
| --- | --- |
| `*` | any logged in user |
| `%` | a connection that has not logged in yet (commands such as `AUTH`, `USER`, `PASS`) |
| `=group` | members of the group `group` (primary or secondary). `=gadmin` matches any group admin |
| `-user` | the user named `user` |
| `!entry` | negates the entry: a user matching it is refused |

A user who matches nothing in the list is refused. See [Permissions](Permissions.md) for the full
rules, which are shared with `perms.conf`.

Examples:

```
perms =siteop =gadmin
perms *
perms !-baduser *
```

## Index

- Core (`core.conf.dist`): 154 commands
- Archive (`archive.conf.dist`): 3 commands
- Autonuke (`autonuke.conf.dist`): 3 commands
- Dupecheck (`dupecheck.conf.dist`): 1 command
- Find (`find.conf.dist`): 3 commands
- IMDB (`imdb.conf.dist`): 4 commands
- Jobs (`jobs.conf.dist`): 7 commands
- Mirror (`mirror.conf.dist`): 1 command
- Request (`request.conf.dist`): 5 commands
- Sitebot (`sitebot.conf.dist`): 8 commands
- Speedtest.net (`speedtest.net.conf.dist`): 2 commands
- TVMaze (`tvmaze.conf.dist`): 4 commands
- Zipscript (`zipscript.conf.dist`): 1 command

## Core (`core.conf.dist`)

### FTP protocol commands

Commands: `ABOR`, `AUTH`, `CDUP`, `CLNT`, `CPSV`, `CWD`, `DELE`, `FEAT`, `IDNT`, `LIST`, `MDTM`, `MKD`, `MLSD`, `MLST`, `MODE`, `NOOP`, `OPTS`, `PASS`, `PASV`, `PBSZ`, `PORT`, `PRET`, `PROT`, `PWD`, `QUIT`, `REST`, `RETR`, `RMD`, `RNFR`, `RNTO`, `SIZE`, `SSCN`, `STAT`, `STOR`, `SYST`, `TYPE`, `USER`, `XCRC`

These commands have no help text in the default configuration:

| Command | Default permission (`perms`) | Options |
| --- | --- | --- |
| `ABOR` | `*` |  |
| `AUTH` | `%` |  |
| `CDUP` | `*` |  |
| `CLNT` | `*` |  |
| `CPSV` | `*` |  |
| `CWD` | `*` |  |
| `DELE` | `*` |  |
| `FEAT` | `*` |  |
| `IDNT` | `%` |  |
| `LIST` | `*` | `slavenames=false` |
| `MDTM` | `*` |  |
| `MKD` | `*` |  |
| `MLSD` | `*` |  |
| `MLST` | `*` |  |
| `MODE` | `*` |  |
| `NOOP` | `*` |  |
| `OPTS` | `% *` |  |
| `PASS` | `%` |  |
| `PASV` | `*` |  |
| `PBSZ` | `% *` |  |
| `PORT` | `*` |  |
| `PRET` | `*` |  |
| `PROT` | `*` |  |
| `PWD` | `*` |  |
| `QUIT` | `*` |  |
| `REST` | `*` |  |
| `RETR` | `*` |  |
| `RMD` | `*` |  |
| `RNFR` | `*` |  |
| `RNTO` | `*` |  |
| `SIZE` | `*` |  |
| `SSCN` | `*` |  |
| `STAT` | `*` | `slavenames=false` |
| `STOR` | `*` |  |
| `SYST` | `*` |  |
| `TYPE` | `*` |  |
| `USER` | `%` |  |
| `XCRC` | `*` |  |

### SITE commands

Commands: `SITE ADDGROUP`, `SITE ADDIP`, `SITE ADDNOTE`, `SITE ADDSLAVE`, `SITE ADDUSER`, `SITE AFFILS`, `SITE ALDN`, `SITE ALUP`, `SITE APPROVE`, `SITE BAN`, `SITE BANALL`, `SITE BANS`, `SITE BNC`, `SITE BW`, `SITE CHANGE`, `SITE CHANGEGROUP`, `SITE CHANGEGROUPADMIN`, `SITE CHANGEUSER`, `SITE CHGRP`, `SITE CHOWN`, `SITE CHPASS`, `SITE CREDITS`, `SITE CUT`, `SITE DAYDN`, `SITE DAYUP`, `SITE DEBUG`, `SITE DELGROUP`, `SITE DELIP`, `SITE DELNOTE`, `SITE DELPURGE`, `SITE DELSLAVE`, `SITE DELUSER`, `SITE DF`, `SITE FAIRNESS`, `SITE FIXLINKS`, `SITE FIXSIZE`, `SITE FIXSLAVECOUNT`, `SITE GADDUSER`, `SITE GALDN`, `SITE GALUP`, `SITE GDAYDN`, `SITE GDAYUP`, `SITE GINFO`, `SITE GIVE`, `SITE GMONTHDN`, `SITE GMONTHUP`, `SITE GPASSED`, `SITE GROUPS`, `SITE GRPREN`, `SITE GTOP`, `SITE GWKDN`, `SITE GWKUP`, `SITE HELP`, `SITE IDLERS`, `SITE INDEXSTATUS`, `SITE KICK`, `SITE KICKALL`, `SITE KICKSLAVE`, `SITE KILL`, `SITE LEECHERS`, `SITE LINK`, `SITE LIST`, `SITE MASTERUPTIME`, `SITE MONTHDN`, `SITE MONTHUP`, `SITE NEW`, `SITE NUKE`, `SITE NUKES`, `SITE NUKESCLEAN`, `SITE PASSED`, `SITE PASSWD`, `SITE PRE`, `SITE PURGE`, `SITE READD`, `SITE REBUILDINDEX`, `SITE REFRESHINDEX`, `SITE RELOAD`, `SITE REMAPPROVE`, `SITE REMERGE`, `SITE REMERGEQUEUE`, `SITE REMOVEEXPIRE`, `SITE RENUSER`, `SITE RULES`, `SITE SECTIONS`, `SITE SEEN`, `SITE SETEXPIRE`, `SITE SHUTDOWN`, `SITE SLAVE`, `SITE SLAVES`, `SITE SLAVESELECT`, `SITE SLAVESUPTIME`, `SITE SLAVEUPTIME`, `SITE SPEED`, `SITE STAT`, `SITE STATS`, `SITE STATUS`, `SITE SWAP`, `SITE SWHO`, `SITE TAGLINE`, `SITE TAKE`, `SITE TIME`, `SITE TOP`, `SITE TRAFFIC`, `SITE UNBAN`, `SITE UNBANALL`, `SITE UNNUKE`, `SITE UPLOADERS`, `SITE USER`, `SITE USERS`, `SITE VERSION`, `SITE WELCOME`, `SITE WHO`, `SITE WIPE`, `SITE WKDN`, `SITE WKUP`, `SITE XDUPE`

#### `SITE ADDGROUP`

Add a new group.

Usage:

```
SITE ADDGROUP <group>
```

- Default permission (`perms`): `=siteop`

#### `SITE ADDIP`

Add a new mask to a user.

Usage:

```
SITE ADDIP <username> <ident@ip>
```

- Default permission (`perms`): `=gadmin =siteop`

#### `SITE ADDNOTE`

Add's Note for user.

Usage:

```
SITE ADDNOTE <username> <note>
```

- Default permission (`perms`): `=siteop`

#### `SITE ADDSLAVE`

Add a slave to the site.

Usage:

```
SITE ADDSLAVE <SLAVENAME>
Add the specified slave to the site.
Note: Be sure to use SITE SLAVE <SLAVENAME> ADDMASK <ident@host>
to allow the slave to connect.
```

- Default permission (`perms`): `=siteop`

#### `SITE ADDUSER`

Add a new user.

Usage:

```
SITE ADDUSER <user> <password> [<ident@ip#1> ...]
```

- Default permission (`perms`): `=gadmin =siteop`

#### `SITE AFFILS`

Displays the site affils.

Usage:

```
SITE AFFILS
```

- Default permission (`perms`): `*`
- Options:
  - `file`: `affils.txt`

#### `SITE ALDN`

Lists the top downloaders of all time.

Usage:

```
SITE ALDN
```

- Default permission (`perms`): `*`

#### `SITE ALUP`

Lists the top uploaders of all time.

Usage:

```
SITE ALUP
```

- Default permission (`perms`): `*`

#### `SITE APPROVE`

Approve release even though rules say it should get nuked.

Usage:

```
SITE APPROVE [path]
```

- Default permission (`perms`): `=siteop`

#### `SITE BAN`

Ban a user from logging onto the site for a specified amount of time.

Usage:

```
SITE BAN <user> <minutes> [reason]
```

- Default permission (`perms`): `=ban =siteop`

#### `SITE BANALL`

Ban all users from logging onto the site for a specified amount of time.

Usage:

```
SITE BANALL <minutes> [reason]
```

- Default permission (`perms`): `=ban =siteop`

#### `SITE BANS`

List the current users banned on the site.

Usage:

```
SITE BANS
```

- Default permission (`perms`): `=ban =siteop`

#### `SITE BNC`

Displays the site bnc details.

Usage:

```
SITE BNC
```

- Default permission (`perms`): `*`
- Options:
  - `file`: `bnc.txt`

#### `SITE BW`

Show total bandwidth use.

Usage:

```
SITE BW
```

- Default permission (`perms`): `*`

#### `SITE CHANGE`

Change parameters for a user.

Usage:

```
SITE CHANGE <username> <field> <value>
Valid fields: group, created, ratio, idle_time, credits, tagline
num_logins # [opt perIP], comment, wkly_allotment, max_sim #dn #up
homedir <path|none> (user and group placeholders are expanded at login)
homedir is NOT a chroot: the user can still CWD / and go anywhere perms.conf allows
```

- Default permission (`perms`): `=gadmin =siteop`

#### `SITE CHANGEGROUP`

Change parameters for a group.

Usage:

```
SITE CHANGEGROUP <groupname> <field> <value>
Valid fields: created, slots, leechslots
```

- Default permission (`perms`): `=siteop`

#### `SITE CHANGEGROUPADMIN`

Add/Remove users as group admin from a group.

Usage:

```
SITE CHANGEGROUPADMIN <group> <user> [<user> ...]
```

- Default permission (`perms`): `=siteop`

#### `SITE CHANGEUSER`

Change parameters for a user.

Usage:

```
SITE CHANGEUSER <username> <field> <value>
Valid fields: group, created, ratio, idle_time, credits, tagline
num_logins # [opt perIP], comment, wkly_allotment, max_sim #dn #up
homedir <path|none> (user and group placeholders are expanded at login)
homedir is NOT a chroot: the user can still CWD / and go anywhere perms.conf allows
```

- Default permission (`perms`): `=gadmin =siteop`

#### `SITE CHGRP`

Change secondary groups for a user.

Usage:

```
SITE CHGRP <user> <group> [<group> ...]
```

- Default permission (`perms`): `=siteop`

#### `SITE CHOWN`

Changes ownership of a file/directory.

Usage:

```
SITE CHOWN [-r] <user>[:<group>] <file>
```

- Default permission (`perms`): `=siteop`

#### `SITE CHPASS`

Change password for a user.

Usage:

```
SITE CHPASS <user> <password>
```

- Default permission (`perms`): `=gadmin =siteop`

#### `SITE CREDITS`

Show available credits.

Usage:

```
SITE CREDITS
```

- Default permission (`perms`): `*`

#### `SITE CUT`

Lists all cut users in trial period.

Usage:

```
SITE CUT
```

- Default permission (`perms`): `*`

#### `SITE DAYDN`

Lists the top downloaders for the day.

Usage:

```
SITE DAYDN
```

- Default permission (`perms`): `*`

#### `SITE DAYUP`

Lists the top uploaders for the day.

Usage:

```
SITE DAYUP
```

- Default permission (`perms`): `*`

#### `SITE DEBUG`

Allows you to see stacktraces on command errors.

Usage:

```
SITE DEBUG
```

- Default permission (`perms`): `*`

#### `SITE DELGROUP`

Delete a group.

Usage:

```
SITE DELGROUP <group>
```

- Default permission (`perms`): `=gadmin =siteop`

#### `SITE DELIP`

Remove a mask from a user.

Usage:

```
SITE DELIP <user> <ident@ip> ...
```

- Default permission (`perms`): `=gadmin =siteop`

#### `SITE DELNOTE`

Removes note for user.

Usage:

```
SITE DELNOTE <username> <noteID>
```

- Default permission (`perms`): `=siteop`

#### `SITE DELPURGE`

Permanently remove a user from the site.

Usage:

```
SITE DELPURGE <username>
```

- Default permission (`perms`): `=gadmin =siteop`

#### `SITE DELSLAVE`

Remove a slave from the site.

Usage:

```
SITE DELSLAVE <SLAVENAME>
Remove the specified slave from the site.
```

- Default permission (`perms`): `=siteop`

#### `SITE DELUSER`

Delete a user.

Usage:

```
SITE DELUSER <user> [reason]
```

- Default permission (`perms`): `=gadmin =siteop`

#### `SITE DF`

Displays current free space.

Usage:

```
SITE DF
```

- Default permission (`perms`): `*`

#### `SITE FAIRNESS`

Show ratio for all users or a specific user/group if given as input

Usage:

```
SITE FAIRNESS [user / =group]
```

- Default permission (`perms`): `=siteop`

#### `SITE FIXLINKS`

Fixes all links from LinkManager.

Usage:

```
SITE FIXLINKS
```

- Default permission (`perms`): `=siteop`

#### `SITE FIXSIZE`

Recursively walks through and validates the size information of the current working directory.

Usage:

```
SITE FIXSIZE
```

- Default permission (`perms`): `=siteop`

#### `SITE FIXSLAVECOUNT`

Recursively walks through and validates the slave reference counts held against the current working directory.

Usage:

```
SITE FIXSLAVECOUNT
This command should only be run when the site has a static VFS,
this means no file uploads, no slaves remerging and no active
archiving/jobs defined. If changes happen in the VFS whilst this
command is running the resulting counts set will be incorrect.
```

- Default permission (`perms`): `=siteop`

#### `SITE GADDUSER`

Add a new user to the specified group.

Usage:

```
SITE GADDUSER <group> <user> <password> [<ident@ip#1 ..]
```

- Default permission (`perms`): `=siteop`

#### `SITE GALDN`

Lists the top downloading group of all time.

Usage:

```
SITE GALDN
```

- Default permission (`perms`): `*`

#### `SITE GALUP`

Lists the top uploading group of all time.

Usage:

```
SITE GALUP
```

- Default permission (`perms`): `*`

#### `SITE GDAYDN`

Lists the top downloading group for the day.

Usage:

```
SITE GDAYDN
```

- Default permission (`perms`): `*`

#### `SITE GDAYUP`

Lists the top uploading group for the day.

Usage:

```
SITE GDAYUP
```

- Default permission (`perms`): `*`

#### `SITE GINFO`

Show information on a group.

Usage:

```
SITE GINFO <group>
```

- Default permission (`perms`): `=gadmin =siteop`

#### `SITE GIVE`

Give a user credits.

Usage:

```
SITE GIVE <user> <bytes>
```

- Default permission (`perms`): `=siteop`

#### `SITE GMONTHDN`

Lists the top downloading group for the month.

Usage:

```
SITE GMONTHDN
```

- Default permission (`perms`): `*`

#### `SITE GMONTHUP`

Lists the top uploading group of all time.

Usage:

```
SITE GMONTHUP
```

- Default permission (`perms`): `*`

#### `SITE GPASSED`

Lists if user has passed trial.

Usage:

```
SITE GPASSED <Group>
```

- Default permission (`perms`): `*`

#### `SITE GROUPS`

Lists the groups on the site

- Default permission (`perms`): `=siteop`

#### `SITE GRPREN`

Rename a group.

Usage:

```
SITE GRPREN <oldgroup> <newgroup>
```

- Default permission (`perms`): `=siteop`

#### `SITE GTOP`

Lists all users in trial period.

Usage:

```
SITE GTOP <num>
```

- Default permission (`perms`): `*`

#### `SITE GWKDN`

Lists the top downloading group for the week.

Usage:

```
SITE GWKDN
```

- Default permission (`perms`): `*`

#### `SITE GWKUP`

Lists the top uploading group for the week.

Usage:

```
SITE GWKUP
```

- Default permission (`perms`): `*`

#### `SITE HELP`

Duh! it does this dummy!

Usage:

```
SITE HELP <site cmd> will display specific information on that command.
```

- Default permission (`perms`): `*`
- Options:
  - `nohelp`: `ABOR,AUTH,CDUP,CLNT,CPSV,CWD,DELE,FEAT,IDNT,LIST,MDTM,MKD,MLSD,MLST,MODE,NOOP,OPTS,PASS,PASV,PBSZ,PORT,PRET,PROT,PWD,QUIT,REST,RETR,RMD,RNFR,RNTO,SIZE,SSCN,STOR,SYST,TYPE,USER,XCRC`

#### `SITE IDLERS`

Show who's idling.

Usage:

```
SITE IDLERS
```

- Default permission (`perms`): `*`

#### `SITE INDEXSTATUS`

Index status.

Usage:

```
SITE INDEXSTATUS
```

- Default permission (`perms`): `=siteop`

#### `SITE KICK`

Kick a users current connections to the site.

Usage:

```
SITE KICK <user> [message]
```

- Default permission (`perms`): `=siteop`

#### `SITE KICKALL`

Kick all users current connections to the site.

Usage:

```
SITE KICKALL [message]
```

- Default permission (`perms`): `=siteop`

#### `SITE KICKSLAVE`

Kick a slave offline.

Usage:

```
SITE KICKSLAVE <SLAVENAME>
```

- Default permission (`perms`): `=siteop`

#### `SITE KILL`

Kill's a specific connection from the FTP by thread ID.

Usage:

```
SITE KILL <threadID> [message]
```

- Default permission (`perms`): `=siteop`

#### `SITE LEECHERS`

Show who's downloading.

Usage:

```
SITE LEECHERS
```

- Default permission (`perms`): `*`

#### `SITE LINK`

Creates a symbolic link.

Usage:

```
SITE LINK <target> <linkname>
```

- Default permission (`perms`): `=siteop`

#### `SITE LIST`

This will raw list files in the current directory.

Usage:

```
SITE LIST
```

- Default permission (`perms`): `*`

#### `SITE MASTERUPTIME`

Will show current master uptime.

Usage:

```
SITE MASTERUPTIME
```

- Default permission (`perms`): `=siteop`

#### `SITE MONTHDN`

Lists the top downloaders for the month.

Usage:

```
SITE MONTHDN
```

- Default permission (`perms`): `*`

#### `SITE MONTHUP`

Lists the top uploaders for the month.

Usage:

```
SITE MONTHUP
```

- Default permission (`perms`): `*`

#### `SITE NEW`

Will list newly uploaded directories.

Usage:

```
SITE NEW - List the newest dirs in all sections.
SITE NEW <number> - List the <number> newest dirs in all sections.
SITE NEW <section> - List the newest dirs in <section>.
SITE NEW <section> <number> - List the <number> newest dirs in <section>.
```

- Default permission (`perms`): `*`
- Options:
  - `default`: `5`
  - `max`: `10`
  - `filtered_sections`: `REQUESTS`

#### `SITE NUKE`

Nuke a directory.

Usage:

```
SITE NUKE <directory> <multiplier> <message>
```

- Default permission (`perms`): `=nuke =siteop`
- Options:
  - `search`: `false`

#### `SITE NUKES`

List all nuked directories.

Usage:

```
SITE NUKES [section]
```

- Default permission (`perms`): `*`

#### `SITE NUKESCLEAN`

Clear nukelog of all invalid nukes.

Usage:

```
SITE NUKESCLEAN
```

- Default permission (`perms`): `=nuke =siteop`

#### `SITE PASSED`

Lists if user has passed trial.

Usage:

```
SITE PASSED <User>
```

- Default permission (`perms`): `*`

#### `SITE PASSWD`

Change your password.

Usage:

```
SITE PASSWD <password>
```

- Default permission (`perms`): `*`

#### `SITE PRE`

Moves dir to specified section.

Usage:

```
SITE PRE <dir> <section>
```

- Default permission (`perms`): `*`

#### `SITE PURGE`

Permanently remove a user from the site.

Usage:

```
SITE PURGE <username>
```

- Default permission (`perms`): `=gadmin =siteop`

#### `SITE READD`

Readd a deleted user.

Usage:

```
SITE READD <user>
```

- Default permission (`perms`): `=gadmin =siteop`

#### `SITE REBUILDINDEX`

Rebuild index.

Usage:

```
SITE REBUILDINDEX
```

- Default permission (`perms`): `=siteop`

#### `SITE REFRESHINDEX`

Requests a recursive refresh of data held for the current directory in the index.

Usage:

```
SITE REFRESHINDEX [-q]
The -q option suppresses details of the progress of the refresh.
```

- Default permission (`perms`): `=siteop`

#### `SITE RELOAD`

Reload the config files.

Usage:

```
SITE RELOAD
```

- Default permission (`perms`): `=siteop`

#### `SITE REMAPPROVE`

Removes approval of release.

Usage:

```
SITE REMAPPROVE [path]
```

- Default permission (`perms`): `=siteop`

#### `SITE REMERGE`

Rebuilds the file database of the specified slave.

Usage:

```
SITE REMERGE <SLAVENAME>
```

- Default permission (`perms`): `=siteop`

#### `SITE REMERGEQUEUE`

Displays remergequeue status.

Usage:

```
SITE REMERGEQUEUE
```

- Default permission (`perms`): `=siteop`

#### `SITE REMOVEEXPIRE`

Removes expiry date for user.

Usage:

```
SITE REMOVEEXPIRE <username>
```

- Default permission (`perms`): `=siteop`

#### `SITE RENUSER`

Rename a user.

Usage:

```
SITE RENUSER <old_username> <new_username>
```

- Default permission (`perms`): `=siteop`

#### `SITE RULES`

Displays the site rules.

Usage:

```
SITE RULES
```

- Default permission (`perms`): `*`
- Options:
  - `file`: `rules.txt`

#### `SITE SECTIONS`

List the currently defined sections on the site.

Usage:

```
SITE SECTIONS
```

- Default permission (`perms`): `*`

#### `SITE SEEN`

Show when a user last connected.

Usage:

```
SITE SEEN <user>
```

- Default permission (`perms`): `*`

#### `SITE SETEXPIRE`

Sets expiry date for user <yyyy-MM-dd>.

Usage:

```
SITE SETEXPIRE <username> <expiry-date>
```

- Default permission (`perms`): `=siteop`

#### `SITE SHUTDOWN`

Shutdown the ftp server.

Usage:

```
SITE SHUTDOWN
```

- Default permission (`perms`): `=siteop`

#### `SITE SLAVE`

Show or modify information about a specific slave.

Usage:

```
SITE SLAVE <SLAVENAME>
Show the stored information set on the specified slave.
SITE SLAVE <SLAVENAME> SET <key> <value>
Add any information you want to the slave.
SITE SLAVE <SLAVENAME> UNSET <key>
Remove any information you want from the slave.
SITE SLAVE <SLAVENAME> ADDMASK <ident@host>
Add a host mask that the specified slave is allowed to connect from.
SITE SLAVE <SLAVENAME> DELMASK <ident@host>
Remove a host mask that the specified slave is allowed to connect from.
SITE SLAVE <SLAVENAME> SHUTDOWN
Notify the specified slave to initiate a shutdown (and potentially restart).
SITE SLAVE <SLAVENAME> QUEUES
Show status of rename & remerge queues.
```

- Default permission (`perms`): `=siteop`

#### `SITE SLAVES`

List the slave added to the site and the status of each.

Usage:

```
SITE SLAVES
```

- Default permission (`perms`): `=siteop`

#### `SITE SLAVESELECT`

Gives the value of each slave for your current SlaveSelection config.

Usage:

```
SITE SLAVESELECT <TYPE> </FULLPATH>
<TYPE> is one of up,down,jobdown,jobup.
</FULLPATH> needs to start with "/" should not exist.
```

- Default permission (`perms`): `=siteop`

#### `SITE SLAVESUPTIME`

Will show current slaves uptime.

Usage:

```
SITE SLAVESUPTIME
```

- Default permission (`perms`): `=siteop`

#### `SITE SLAVEUPTIME`

Will show current slave uptime.

Usage:

```
SITE SLAVEUPTIME <slave>
```

- Default permission (`perms`): `=siteop`

#### `SITE SPEED`

Show the state of the specified users connections.

Usage:

```
SITE SPEED
```

- Default permission (`perms`): `*`

#### `SITE STAT`

Show simple user stats.

Usage:

```
SITE STAT
```

- Default permission (`perms`): `*`

#### `SITE STATS`

Show user stats.

Usage:

```
SITE STATS
```

- Default permission (`perms`): `*`

#### `SITE STATUS`

Will show current information about the server.

Usage:

```
SITE STATUS Usage:
SITE STATUS <os, vm, memory, threads, gc, classes, cm, cpu, network, hardware, slaves, all>
SITE STATUS hardware <master|slavename> - detail of one machine (alias: hw)
SITE STATUS OS - Will show Operational System Info
SITE STATUS VM - Will show VM Info.
SITE STATUS MEMORY - Will show Memory Info.
SITE STATUS THREADS - Will show Threads Info.
SITE STATUS GC - Will show Garbage Collector Info.
SITE STATUS CLASSES - Will show Classes Info.
SITE STATUS CM - Will show Commit Manager Info.
SITE STATUS CPU - Will show CPU Info.
SITE STATUS NETWORK - Will show Network Info.
SITE STATUS HARDWARE (or HW) - Will show a hardware summary (OS, cores, CPU, RAM, disks) of the master and of each slave. It can take a few seconds while slaves answer, so it is not included in ALL (like SLAVES). RAM is total minus available memory (on Linux the page cache counts as available).
SITE STATUS HARDWARE <master|slavename> - Will show the hardware detail of the master or of one slave, including each root.
SITE STATUS SLAVES - Will show Slave connection and TLS Info (not included in ALL).
SITE STATUS ALL - Shortcut to display ALL Info (except SLAVES and HARDWARE).
```

- Default permission (`perms`): `=siteop`

#### `SITE SWAP`

Swap credits between users.

Usage:

```
SITE SWAP <srcuser> <destuser> <bytes>
```

- Default permission (`perms`): `=siteop`

#### `SITE SWHO`

Show who's online.

Usage:

```
SITE SWHO
```

- Default permission (`perms`): `=siteop`

#### `SITE TAGLINE`

Change your tagline.

Usage:

```
SITE TAGLINE <tagline>
```

- Default permission (`perms`): `*`

#### `SITE TAKE`

Remove credits from a user.

Usage:

```
SITE TAKE <user> <bytes>
```

- Default permission (`perms`): `=siteop`

#### `SITE TIME`

Show current server time.

Usage:

```
SITE TIME
```

- Default permission (`perms`): `*`

#### `SITE TOP`

Lists all users in trial period.

Usage:

```
SITE TOP <num>
```

- Default permission (`perms`): `*`

#### `SITE TRAFFIC`

Summary of the daemon upload/download stats.

Usage:

```
SITE TRAFFIC
```

- Default permission (`perms`): `*`

#### `SITE UNBAN`

Removes a ban for the specified user.

Usage:

```
SITE UNBAN <user>
```

- Default permission (`perms`): `=ban =siteop`

#### `SITE UNBANALL`

Removes ban from all users.

Usage:

```
SITE UNBANALL
```

- Default permission (`perms`): `=ban =siteop`

#### `SITE UNNUKE`

Unnuke a directory.

Usage:

```
SITE UNNUKE <directory>
```

- Default permission (`perms`): `=nuke =siteop`
- Options:
  - `search`: `false`

#### `SITE UPLOADERS`

Show who's uploading.

Usage:

```
SITE UPLOADERS
```

- Default permission (`perms`): `*`

#### `SITE USER`

Show details for a user.

Usage:

```
SITE USER <user>
```

- Default permission (`perms`): `*`

#### `SITE USERS`

List all usernames added to the site.

Usage:

```
SITE USERS
```

- Default permission (`perms`): `=siteop`

#### `SITE VERSION`

Show DrFTPD version.

Usage:

```
SITE VERSION
Show DrFTPD version.
```

- Default permission (`perms`): `*`

#### `SITE WELCOME`

Displays site welcome message.

Usage:

```
SITE WELCOME
```

- Default permission (`perms`): `*`
- Options:
  - `file`: `welcome.txt`

#### `SITE WHO`

Show who's online.

Usage:

```
SITE WHO
```

- Default permission (`perms`): `=siteop`

#### `SITE WIPE`

Deletes files without removing credits. -r means delete recursively.

Usage:

```
SITE WIPE [-r] <file/dir>
```

- Default permission (`perms`): `=wipe =siteop`

#### `SITE WKDN`

Lists the top downloaders for the week.

Usage:

```
SITE WKDN
```

- Default permission (`perms`): `*`

#### `SITE WKUP`

Lists the top uploaders for the week.

Usage:

```
SITE WKUP
```

- Default permission (`perms`): `*`

#### `SITE XDUPE`

The X-DUPE mechanism helps you saving time when uploading files by avoiding "dupes".

Usage:

```
SITE XDUPE <mode>
Mode 1: Up to a maximum of 80 chars total line length. Filenames longer than 66 chars will be truncated.
Mode 2: The server sends only one filename per X-DUPE line, and the maximum line length is 80 chars. Filenames are also truncated.
Mode 3: Only one filename is sent per X-DUPE line, and the filename will not be truncated.
Mode 4: All files are listed on one long line, up to a maximum of 1024 characters. Files which are too long to fit will be skipped.
```

- Default permission (`perms`): `*`

## Archive (`archive.conf.dist`)

Feature documentation: [Archive](Archive.md).

Commands: `SITE ARCHIVE`, `SITE LISTARCHIVEQUEUE`, `SITE LISTARCHIVETYPES`

### `SITE ARCHIVE`

Create a new ArchiveHandler like defined in archive.conf

Usage:

```
SITE ARCHIVE Directory [ArchiveType] [property1] [property2] [property...]
- An example of a property is slavename.1=myslave just like in archive.conf
```

- Default permission (`perms`): `=siteop`
- Options:
  - `badarchivetype=ArchiveType`: `${archivetypename} was unable to be initialized - ${exception}`
  - `incompatible`: `ArchiveType ${archivetypename} is incompatible with this version of DrFTPD`
  - `baddir`: `Directory ${dirname} was unable to be found`
  - `badslave`: `Unable to add ${slavename} to the slaveList`
  - `loadarchive`: `The Archive plugin must be loaded, use SITE LOADPLUGIN Archive`
  - `success`: `Started ArchiveHandler for ${archivetypename} on ${dirname}`
  - `fail`: `${exception}`

### `SITE LISTARCHIVEQUEUE`

List the currently running ArchiveTypes

- Default permission (`perms`): `=siteop`

### `SITE LISTARCHIVETYPES`

List the currently running ArchiveTypes.

Usage:

```
SITE LISTARCHIVETYPES
```

- Default permission (`perms`): `=siteop`

## Autonuke (`autonuke.conf.dist`)

Feature documentation: [Autonuke](Autonuke.md).

Commands: `SITE AUTONUKES`, `SITE AUTONUKESCAN`, `SITE DELQUEUE`

### `SITE AUTONUKES`

Show AutoNuke queue.

Usage:

```
SITE AUTONUKES [section]
```

- Default permission (`perms`): `=siteop`

### `SITE AUTONUKESCAN`

Manual AutoNuke scan.

Usage:

```
SITE AUTONUKESCAN [path]
```

- Default permission (`perms`): `=siteop`

### `SITE DELQUEUE`

Delete one/all item(s) from AutoNuke queue.

Usage:

```
SITE DELQUEUE [path]
```

- Default permission (`perms`): `=siteop`

## Dupecheck (`dupecheck.conf.dist`)

Feature documentation: [Dupecheck](Dupecheck.md).

Commands: `SITE UNDUPE`

### `SITE UNDUPE`

UnDupe a File/Directory.

Usage:

```
SITE UNDUPE <directory/file>
```

- Default permission (`perms`): `=siteop`

## Find (`find.conf.dist`)

Feature documentation: [Find](Find.md).

Commands: `SITE DUPE`, `SITE FIND`, `SITE SEARCH`

### `SITE DUPE`

Searches for files and directories using exact name match.

Usage:

```
SITE DUPE <name>
```

- Default permission (`perms`): `*`
- Options:
  - `limit`: `50`
  - `type`: `f`
  - `exact`: `true`

### `SITE FIND`

Searches for files and directories in the filesystem w/ some neat filters and actions.

Usage:

```
SITE FIND <options> <actions>
SITE FIND options [name] - for a list of all registered options, if the name is provided print details of the option
SITE FIND actions - for a list of all registered actions
All options/actions with multiple or space in arguments must be enclosed with ".
Valid date format: yyyy.MM.dd OR yyyy.MM.dd.HH.mm.ss, Example: 2010.01.29.08.36.33
For range options min or max can be left empty
Valid sort fields: parentPath fullPath(default) owner group type nbrOfSlaves lastModified size
Regex option matches against full path.
Exact option makes a wildcard query against exact file/dir name.
Wildcards(* and ?) available but not within the first three chars.
Options for printf format:
#f - name
#p - path
#h - parent path
#H - parent name
#s - size
#u - user
#g - group
#x - slave
#t - last modified
Example: SITE FIND -printf "filename: #f size: #s"
Multiple options/actions are allowed. If multiple options are given a file/dir must match all options.
Dupe option: -dupe [oldest|newest] finds releases whose name (case-insensitive) exists more than once among the matches.
Only release dirs are compared: dirs directly in a section dir, or in a dated dir of a dated section.
One copy of every dupe is kept (default the oldest = first created), actions only run on the other copies.
Note: a dir takes the creation time of its oldest file, so a copy remerged from a slave with old file times can look first created.
Copies with a change anywhere inside them within dupe.min.age minutes are listed as SKIP (active) and never acted on.
Releases too large to check (more than 4 levels or 10000 files) are listed as SKIP (too large to check), whatever dupe.min.age is.
Without an action -dupe is a dry run that only lists the dupes. Use -wipe to remove them (needs the dupewipe permission)
or -nuke to nuke them (needs the dupenuke permission), other actions than -print and -printf are refused with -dupe.
-f, -slaves and -nbrofslaves only match files and can not be combined with -dupe.
Nuked dirs and names matching dupe.exclude are ignored, -limit is the number of dupe groups.
-nuke refuses a name that is already in the nuke log, so with 3 or more copies only the first one is nuked and the others are
reported as SKIP (name already nuked), as are all copies of a name nuked before. Use -wipe to remove those.
Example: SITE FIND -dupe -section MP3 -wipe
```

- Default permission (`perms`): `*`
- Options:
  - `limit.default`: `50`
  - `limit.max`: `100`
  - `sendToSlaves`: `=siteop`
  - `deleteFromSlaves`: `=siteop`
  - `wipe`: `=siteop`
  - `nuke`: `=siteop =nuke`
  - `unnuke`: `=siteop =nuke`
  - `dupewipe`: `=siteop`
  - `dupenuke`: `=siteop`

### `SITE SEARCH`

Searches for files and directories using wildcards.

Usage:

```
SITE SEARCH <word> [word ..]
```

- Default permission (`perms`): `*`
- Options:
  - `limit`: `50`
  - `type`: `d`
  - `exact`: `false`

## IMDB (`imdb.conf.dist`)

Feature documentation: [IMDB-and-TVMaze](IMDB-and-TVMaze.md).

Commands: `SITE CREATEIMDB`, `SITE IMDB`, `SITE IMDBQUEUE`, `SITE REMOVEIMDB`

### `SITE CREATEIMDB`

Create IMDB metadata for all movies below current dir.

Usage:

```
SITE CREATEIMDB [path]
```

- Default permission (`perms`): `=siteop`

### `SITE IMDB`

Search for a movie.

Usage:

```
SITE IMDB <nameofmovie>
```

- Default permission (`perms`): `*`

### `SITE IMDBQUEUE`

Show number of items in the queue to be processed.

Usage:

```
SITE IMDBQUEUE
```

- Default permission (`perms`): `=siteop`

### `SITE REMOVEIMDB`

Remove IMDB metadata for all movies below current dir.

Usage:

```
SITE REMOVEIMDB [path]
```

- Default permission (`perms`): `=siteop`

## Jobs (`jobs.conf.dist`)

Feature documentation: [Jobs](Jobs.md).

Commands: `SITE ADDJOB`, `SITE LISTJOBS`, `SITE LISTRUNNINGJOBS`, `SITE REMOVEJOB`, `SITE REMOVEJOBS`, `SITE STARTJOBS`, `SITE STOPJOBS`

### `SITE ADDJOB`

Add a job to the JobManager queue.

Usage:

```
SITE ADDJOB <file> <priority> <timestomirror> <destslave> [destslave ...]
<file> : The file to archive.
<priority> : The priority you want this job to have. Must be an Integer.
<timestomirror> : The max number of slaves each file should be on.
<destslave> : The name of the slave to transfer the files to.
```

- Default permission (`perms`): `=siteop`

### `SITE LISTJOBS`

List the current jobs queued by the JobManager by index #.

Usage:

```
SITE LISTJOBS
```

- Default permission (`perms`): `=siteop`

### `SITE LISTRUNNINGJOBS`

List the current jobs queued by the JobManager by index #.

Usage:

```
SITE LISTRUNNINGJOBS
```

- Default permission (`perms`): `=siteop`

### `SITE REMOVEJOB`

Removes a Job from the JobManager queue by index # or range.

Usage:

```
SITE REMOVEJOB <jobindex> <lowrange>-<highrange> will remove the job with
jobindex and all jobs including and between lowrange and highrange.
You can use as many ranges as your heart desires.
```

- Default permission (`perms`): `=siteop`

### `SITE REMOVEJOBS`

Removes all jobs that are not currently transferring.

Usage:

```
SITE REMOVEJOBS
```

- Default permission (`perms`): `=siteop`

### `SITE STARTJOBS`

Starts running the jobs in the JobManager queue.

Usage:

```
SITE STARTJOBS
```

- Default permission (`perms`): `=siteop`

### `SITE STOPJOBS`

Stops running the jobs in the JobManager queue.

Usage:

```
SITE STOPJOBS
```

- Default permission (`perms`): `=siteop`

## Mirror (`mirror.conf.dist`)

Feature documentation: [Mirror](Mirror.md).

Commands: `SITE UNMIRROR`

### `SITE UNMIRROR`

Unmirror path so files are kept on one slave only.

Usage:

```
SITE UNMIRROR <path>
```

- Default permission (`perms`): `=siteop`

## Request (`request.conf.dist`)

Feature documentation: [Requests](Requests.md).

Commands: `SITE FIXREQUESTS`, `SITE REQDEL`, `SITE REQFILLED`, `SITE REQUEST`, `SITE REQUESTS`

### `SITE FIXREQUESTS`

Tries to fix requests dir with the actual requests

Usage:

```
SITE FIXREQUESTS
```

- Default permission (`perms`): `=siteop`

### `SITE REQDEL`

Deletes a request.

Usage:

```
SITE REQDEL <nameofrequest>
```

- Default permission (`perms`): `=request =siteop`
- Options:
  - `request.deleteOthers`: `=siteop`

### `SITE REQFILLED`

Fill a request.

Usage:

```
SITE REQFILLED <nameofrequest>
```

- Default permission (`perms`): `*`

### `SITE REQUEST`

Add a request.

Usage:

```
SITE REQUEST <nameofrequest>
```

- Default permission (`perms`): `=request =siteop`
- Options:
  - `request.deleteOthers`: `=siteop`

### `SITE REQUESTS`

List unfilled requests.

Usage:

```
SITE REQUESTS
```

- Default permission (`perms`): `*`

## Sitebot (`sitebot.conf.dist`)

Feature documentation: [Sitebot](Sitebot.md).

Commands: `SITE BLOWFISH`, `SITE GETKEY`, `SITE IDENT`, `SITE INVITE`, `SITE IRC`, `SITE SETBLOWFISH`, `SITE SETKEY`, `SITE UNIDENT`

### `SITE BLOWFISH`

List blowfish keys for irc channels.

Usage:

```
SITE BLOWFISH
SITE BLOWFISH <botname>
<botname> is required to use a secondary sitebot.
```

- Default permission (`perms`): `*`

### `SITE GETKEY`

List blowfish keys for irc channels.

Usage:

```
SITE GETKEY
SITE GETKEY <botname>
<botname> is required to use a secondary sitebot.
```

- Default permission (`perms`): `*`

### `SITE IDENT`

Associates an irc nick with a ftp user.

Usage:

```
SITE IDENT <username> <password>
```

- Default permission (`perms`): `%`

### `SITE INVITE`

Invite yourself to irc channels using the sitebot.

Usage:

```
SITE INVITE <ircnick>
SITE INVITE <botname> <ircnick>
<botname> is required to use a secondary sitebot.
```

- Default permission (`perms`): `*`

### `SITE IRC`

Control the IRC sitebot.

Usage:

```
SITE IRC connect
SITE IRC <botname> connect
SITE IRC disconnect
SITE IRC <botname> disconnect
SITE IRC reconnect
SITE IRC <botname> reconnect
SITE IRC raw <commmand>
SITE IRC <botname> raw <command>
SITE IRC say <message>
SITE IRC <botname> say <message>
SITE IRC emptyqueue
<botname> is mandatory if more than one sitebot running.
```

- Default permission (`perms`): `=siteop`

### `SITE SETBLOWFISH`

Set your personal blowfish key on the sitebot.

Usage:

```
SITE SETBLOWFISH <blowkey>
SITE SETBLOWFISH <botname> <blowkey>
<botname> is required to use a secondary sitebot.
```

- Default permission (`perms`): `*`

### `SITE SETKEY`

Set your personal blowfish key on the sitebot.

Usage:

```
SITE SETKEY <blowkey>
SITE SETKEY <botname> <blowkey>
<botname> is required to use a secondary sitebot.
```

- Default permission (`perms`): `*`

### `SITE UNIDENT`

Unassociates an irc nick with a ftp user.

Usage:

```
SITE UNIDENT <username> <password>
```

- Default permission (`perms`): `*`

## Speedtest.net (`speedtest.net.conf.dist`)

Feature documentation: [Speedtest](Speedtest.md).

Commands: `SITE REFRESHSPEEDTEST`, `SITE SPEEDTEST`

### `SITE REFRESHSPEEDTEST`

Force an update of speedtest.net server list.

Usage:

```
SITE REFRESHSPEEDTEST
```

- Default permission (`perms`): `=siteop`

### `SITE SPEEDTEST`

Run a speedtest on slave against speedtest.net test servers, * for all slaves.

Usage:

```
SITE SPEEDTEST [<slavename> [-list | -update | <test server id>]]
-list option in conjunction with slave name lists closests servers to slave location based on its IP
-update option in conjunction with slave name to force location update
An optional test server id can be specified to force test server.
```

- Default permission (`perms`): `=siteop`

## TVMaze (`tvmaze.conf.dist`)

Feature documentation: [IMDB-and-TVMaze](IMDB-and-TVMaze.md).

Commands: `SITE CREATETV`, `SITE REMOVETV`, `SITE TV`, `SITE TVQUEUE`

### `SITE CREATETV`

Create TvMaze metadata for all shows below current dir.

Usage:

```
SITE CREATETV [path]
```

- Default permission (`perms`): `=siteop`

### `SITE REMOVETV`

Remove TvMaze metadata for all shows below current dir.

Usage:

```
SITE REMOVETV [path]
```

- Default permission (`perms`): `=siteop`

### `SITE TV`

Search for a tv show (-v for verbose output).

Usage:

```
SITE TV [-v] <nameofshow> [episodenumber]
```

- Default permission (`perms`): `*`

### `SITE TVQUEUE`

Show number of items in the queue to be processed.

Usage:

```
SITE TVQUEUE
```

- Default permission (`perms`): `=siteop`

## Zipscript (`zipscript.conf.dist`)

Feature documentation: [Zipscript](Zipscript.md).

Commands: `SITE RESCAN`

### `SITE RESCAN`

SFV scans the files in the current directory or a specific directory requested.

Usage:

```
SITE RESCAN [-r] [noforce] [nodelete] [nodelete0byte] [quiet] [file/path]
The -r option recursively scans all directories below the current directory as well.
The noforce option makes the slave recalculate the checksums.
The nodelete option removes bad files rather than just reporting them.
The nodelete0byte option removes any files listed in the SFV which exist but are 0 byte.
The quiet option only prints output for files which aren't good.
Specifying a file or path will rescan the requested file or directory instead of the current directory.
If the file or path does not start with '/' it will be relative to your current directory
```

- Default permission (`perms`): `*`
