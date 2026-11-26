> Applies to DrFTPD 4.0.x.

# IRC Commands

This page lists the commands of the [Sitebot](Sitebot.md) that DrFTPD 4.0.x defines in its default
configuration, with the usage text, the default permission and the options of each command. It is
generated from the `*.conf.dist` files shipped with the source (159 commands in 13 files).

## Where the commands are configured

IRC commands live in `config/commands/irc/`, one file per feature (`core.conf.dist`,
`sitebot.conf.dist`, ...). The format is the same as for [Site Commands](Site-Commands.md): a
`NAME { ... }` block with `plugin`, `class`, `method`, `perms`, `help`, `help.specific`. To change
a command, copy `X.conf.dist` to `X.conf` and edit the copy. An `X.conf` replaces its `X.conf.dist`
completely, so after an upgrade you have to merge new commands in by hand (or delete your `X.conf`).
Details and the "already mapped" pitfall are explained on [Site Commands](Site-Commands.md).

Commands are typed in a channel the bot is in, or sent to the bot in a private message,
starting with the command trigger. The default trigger is `!` and it is set by `command.trigger` in
`irc.conf` (see [Sitebot](Sitebot.md)). Example: `!bw`. The tables below write the commands with the
default trigger. The command name is the name in the block, in lowercase.

## Options specific to IRC commands

| Option | Meaning |
| --- | --- |
| `input` | Where the command is accepted from. Space separated: `all`, `public` (any channel), `private` (private message to the bot) or a channel name such as `#staff` |
| `output` | Where the reply goes. Space separated: `public` (all channels the bot is in), `private` (private message to the user), `source` (where the command came from) or a channel name such as `#staff` |

A command with no valid `input`, or an `output` that resolves to nowhere, is silently ignored.

## How an IRC user is mapped to a site user

The bot does not ask for a password on every command. It maps the IRC identity (`nick!user@host`)
of the sender to a site user, and then runs the command as that site user, with that user's
permissions:

1. A site user links the IRC identity once with `!ident <username> <password>` (in a private message to the bot).
   The bot stores `botname|nick!user@host` in the user's `ircident` field, one entry per bot.
   `!invite <username> <password>` also stores the identity, and in addition invites the user into the channels.
   `!unident` removes the stored identity.
2. On each later command the bot looks the sender's full `nick!user@host` up among the stored
   identities. The match is literal, it is not a wildcard mask, so if your host changes you must `!ident` again.
3. The command is then checked against its `perms` like an FTP `SITE` command. `*` means any user
   that was recognised, `=group` and `-user` work as usual. `%` means "not recognised": the
   `ident` command uses it so unknown people can link themselves.
4. If the sender is not recognised, the command runs as no user and only `%` entries match.

Because `ident` takes the site password, send it in a private message
(the default configuration only accepts the sitebot commands in private, with `input private`).

Everything the master accepts as `perms` ([Permissions](Permissions.md)) is valid here as well, so you can,
for example, let only the `=siteop` group use `!nuke`.

## Index

- Core (`core.conf.dist`): 117 commands
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

Commands: `!abort`, `!addgroup`, `!addip`, `!addnote`, `!addslave`, `!adduser`, `!affils`, `!aldn`, `!alup`, `!approve`, `!ban`, `!banall`, `!bans`, `!bnc`, `!bw`, `!change`, `!changegroup`, `!changegroupadmin`, `!changeuser`, `!chgrp`, `!chown`, `!chpass`, `!credits`, `!cut`, `!daydn`, `!dayup`, `!debug`, `!delgroup`, `!delip`, `!delnote`, `!delpurge`, `!delslave`, `!deluser`, `!df`, `!fairness`, `!fixlinks`, `!fixsize`, `!fixslavecount`, `!gadduser`, `!galdn`, `!galup`, `!gdaydn`, `!gdayup`, `!ginfo`, `!give`, `!gmonthdn`, `!gmonthup`, `!gpassed`, `!groups`, `!grpren`, `!gtop`, `!gwkdn`, `!gwkup`, `!help`, `!idlers`, `!indexstatus`, `!kick`, `!kickall`, `!kickslave`, `!kill`, `!leechers`, `!link`, `!list`, `!master-uptime`, `!monthdn`, `!monthup`, `!new`, `!nuke`, `!nukes`, `!nukesclean`, `!passed`, `!passwd`, `!pre`, `!purge`, `!readd`, `!rebuildindex`, `!refreshindex`, `!reload`, `!remapprove`, `!remerge`, `!remergequeue`, `!removeexpire`, `!renuser`, `!rules`, `!sections`, `!seen`, `!setexpire`, `!shutdown`, `!slave`, `!slave-select`, `!slave-uptime`, `!slaves`, `!slaves-uptime`, `!speed`, `!stat`, `!stats`, `!status`, `!swap`, `!swho`, `!tagline`, `!take`, `!time`, `!top`, `!traffic`, `!unban`, `!unbanall`, `!unnuke`, `!uploaders`, `!user`, `!users`, `!version`, `!welcome`, `!who`, `!wipe`, `!wkdn`, `!wkup`, `!xdupe`

### `!abort`

Abort all commands being run by the user issuing the abort.

Usage:

```
!abort
```

- Default permission (`perms`): `*`
- Options:
  - `input`: `all`
  - `output`: `source`

### `!addgroup`

Add a new group.

Usage:

```
!addgroup <group>
```

- Default permission (`perms`): `=siteop`
- Options:
  - `input`: `all`
  - `output`: `source`

### `!addip`

Add a new mask to a user.

Usage:

```
!addip <username> <ident@ip>
```

- Default permission (`perms`): `=gadmin =siteop`
- Options:
  - `input`: `all`
  - `output`: `source`

### `!addnote`

Add's Note for user.

Usage:

```
!addnote <username> <note>
```

- Default permission (`perms`): `=siteop`
- Options:
  - `input`: `all`
  - `output`: `source`

### `!addslave`

Add a slave to the site.

Usage:

```
!addslave <SLAVENAME>
Add the specified slave to the site.
Note: Be sure to use SLAVE <SLAVENAME> ADDMASK <ident@host>
to allow the slave to connect.
```

- Default permission (`perms`): `=siteop`
- Options:
  - `input`: `all`
  - `output`: `source`

### `!adduser`

Add a new user.

Usage:

```
!adduser <user> <password> [<ident@ip#1> ...]
```

- Default permission (`perms`): `=gadmin =siteop`
- Options:
  - `input`: `all`
  - `output`: `source`

### `!affils`

Displays the site affils.

Usage:

```
!affils
```

- Default permission (`perms`): `*`
- Options:
  - `file`: `affils.txt`
  - `input`: `public`
  - `output`: `source`

### `!aldn`

Lists the top downloaders of all time.

Usage:

```
!aldn
```

- Default permission (`perms`): `*`
- Options:
  - `input`: `all`
  - `output`: `source`

### `!alup`

Lists the top uploaders of all time.

Usage:

```
!alup
```

- Default permission (`perms`): `*`
- Options:
  - `input`: `all`
  - `output`: `source`

### `!approve`

Approve release even though rules say it should get nuked.

Usage:

```
!approve [path]
```

- Default permission (`perms`): `=siteop`
- Options:
  - `input`: `all`
  - `output`: `source`

### `!ban`

Ban a user from logging onto the site for a specified amount of time.

Usage:

```
!ban <user> <minutes> [reason]
```

- Default permission (`perms`): `=ban =siteop`
- Options:
  - `input`: `all`
  - `output`: `source`

### `!banall`

Ban all users from logging onto the site for a specified amount of time.

Usage:

```
!banall <minutes> [reason]
```

- Default permission (`perms`): `=ban =siteop`
- Options:
  - `input`: `all`
  - `output`: `source`

### `!bans`

List the current users banned on the site.

Usage:

```
!bans
```

- Default permission (`perms`): `=ban =siteop`
- Options:
  - `input`: `all`
  - `output`: `source`

### `!bnc`

Displays the site bnc details.

Usage:

```
!bnc
```

- Default permission (`perms`): `*`
- Options:
  - `file`: `bnc.txt`
  - `input`: `public`
  - `output`: `source`

### `!bw`

Show total bandwidth use.

Usage:

```
!bw
```

- Default permission (`perms`): `*`
- Options:
  - `input`: `all`
  - `output`: `source`

### `!change`

Change parameters for a user.

Usage:

```
!change <username> <field> <value>
Valid fields: group, created, ratio, idle_time, credits, tagline
num_logins # [opt perIP], comment, wkly_allotment, max_sim #dn #up
homedir <path|none> (user and group placeholders are expanded at login)
homedir is NOT a chroot: the user can still CWD / and go anywhere perms.conf allows
```

- Default permission (`perms`): `=gadmin =siteop`
- Options:
  - `input`: `all`
  - `output`: `source`

### `!changegroup`

Change parameters for a group.

Usage:

```
!changegroup <groupname> <field> <value>
Valid fields: created, slots, leechslots
```

- Default permission (`perms`): `=siteop`
- Options:
  - `input`: `all`
  - `output`: `source`

### `!changegroupadmin`

Add/Remove users as group admin from a group.

Usage:

```
!changegroupadmin <group> <user> [<user> ...]
```

- Default permission (`perms`): `=siteop`
- Options:
  - `input`: `all`
  - `output`: `source`

### `!changeuser`

Change parameters for a user.

Usage:

```
!changeuser <username> <field> <value>
Valid fields: group, created, ratio, idle_time, credits, tagline
num_logins # [opt perIP], comment, wkly_allotment, max_sim #dn #up
homedir <path|none> (user and group placeholders are expanded at login)
homedir is NOT a chroot: the user can still CWD / and go anywhere perms.conf allows
```

- Default permission (`perms`): `=gadmin =siteop`
- Options:
  - `input`: `all`
  - `output`: `source`

### `!chgrp`

Change secondary groups for a user.

Usage:

```
!chgrp <user> <group> [<group> ...]
```

- Default permission (`perms`): `=siteop`
- Options:
  - `input`: `all`
  - `output`: `source`

### `!chown`

Changes ownership of a file/directory.

Usage:

```
!chown [-r] <user>[:<group>] <file>
```

- Default permission (`perms`): `=siteop`
- Options:
  - `input`: `all`
  - `output`: `source`

### `!chpass`

Change password for a user.

Usage:

```
!chpass <user> <password>
```

- Default permission (`perms`): `=gadmin =siteop`
- Options:
  - `input`: `all`
  - `output`: `source`

### `!credits`

Show available credits.

Usage:

```
!credits
```

- Default permission (`perms`): `*`
- Options:
  - `input`: `all`
  - `output`: `source`

### `!cut`

Lists all cut users in trial period.

Usage:

```
!cut <num>
```

- Default permission (`perms`): `*`
- Options:
  - `input`: `all`
  - `output`: `source`

### `!daydn`

Lists the top downloaders today.

Usage:

```
!daydn
```

- Default permission (`perms`): `*`
- Options:
  - `input`: `all`
  - `output`: `source`

### `!dayup`

Lists the top uploaders today.

Usage:

```
!dayup
```

- Default permission (`perms`): `*`
- Options:
  - `input`: `all`
  - `output`: `source`

### `!debug`

Allows you to see stacktraces on command errors.

Usage:

```
!debug
```

- Default permission (`perms`): `*`
- Options:
  - `input`: `all`
  - `output`: `source`

### `!delgroup`

Delete a group.

Usage:

```
!delgroup <group>
```

- Default permission (`perms`): `=gadmin =siteop`
- Options:
  - `input`: `all`
  - `output`: `source`

### `!delip`

Remove a mask from a user.

Usage:

```
!delip <user> <ident@ip> ...
```

- Default permission (`perms`): `=gadmin =siteop`
- Options:
  - `input`: `all`
  - `output`: `source`

### `!delnote`

Removes Note For User.

Usage:

```
!delnote <username> <noteID>
```

- Default permission (`perms`): `=siteop`
- Options:
  - `input`: `all`
  - `output`: `source`

### `!delpurge`

Permanently remove a user from the site.

Usage:

```
!delpurge <username>
```

- Default permission (`perms`): `=gadmin =siteop`
- Options:
  - `input`: `all`
  - `output`: `source`

### `!delslave`

Remove a slave from the site.

Usage:

```
!delslave <SLAVENAME>
Remove the specified slave from the site.
```

- Default permission (`perms`): `=siteop`
- Options:
  - `input`: `all`
  - `output`: `source`

### `!deluser`

Delete a user.

Usage:

```
!deluser <user> [reason]
```

- Default permission (`perms`): `=gadmin =siteop`
- Options:
  - `input`: `all`
  - `output`: `source`

### `!df`

Displays current free space.

Usage:

```
!df
```

- Default permission (`perms`): `*`
- Options:
  - `input`: `all`
  - `output`: `source`

### `!fairness`

Show ratio for all users or a specific user/group if given as input

Usage:

```
!fairness [user / =group]
```

- Default permission (`perms`): `=siteop`
- Options:
  - `input`: `all`
  - `output`: `source`

### `!fixlinks`

Fixes all links from LinkManager.

Usage:

```
!fixlinks
```

- Default permission (`perms`): `=siteop`
- Options:
  - `input`: `all`
  - `output`: `source`

### `!fixsize`

Recursively walks through and validates the size information of the current working directory.

Usage:

```
!fixsize
```

- Default permission (`perms`): `=siteop`
- Options:
  - `input`: `all`
  - `output`: `source`

### `!fixslavecount`

Recursively walks through and validates the slave reference counts held against the current working directory.

Usage:

```
!fixslavecount
This command should only be run when the site has a static VFS,
this means no file uploads, no slaves remerging and no active
archiving/jobs defined. If changes happen in the VFS whilst this
command is running the resulting counts set will be incorrect.
```

- Default permission (`perms`): `=siteop`
- Options:
  - `input`: `all`
  - `output`: `source`

### `!gadduser`

Add a new user to the specified group.

Usage:

```
!gadduser <group> <user> <password> [<ident@ip#1 ..]
```

- Default permission (`perms`): `=siteop`
- Options:
  - `input`: `all`
  - `output`: `source`

### `!galdn`

Lists the top downloading group of all time.

Usage:

```
!galdn
```

- Default permission (`perms`): `*`
- Options:
  - `input`: `all`
  - `output`: `source`

### `!galup`

Lists the top uploading group of all time.

Usage:

```
!galup
```

- Default permission (`perms`): `*`
- Options:
  - `input`: `all`
  - `output`: `source`

### `!gdaydn`

Lists the top downloading group for the day.

Usage:

```
!gdaydn
```

- Default permission (`perms`): `*`
- Options:
  - `input`: `all`
  - `output`: `source`

### `!gdayup`

Lists the top uploading group for the day.

Usage:

```
!gdayup
```

- Default permission (`perms`): `*`
- Options:
  - `input`: `all`
  - `output`: `source`

### `!ginfo`

Show information on a group.

Usage:

```
!ginfo <group>
```

- Default permission (`perms`): `=gadmin =siteop`
- Options:
  - `input`: `all`
  - `output`: `source`

### `!give`

Give a user credits.

Usage:

```
!give <user> <bytes>
```

- Default permission (`perms`): `=siteop`
- Options:
  - `input`: `all`
  - `output`: `source`

### `!gmonthdn`

Lists the top downloading group for the month.

Usage:

```
!gmonthdn
```

- Default permission (`perms`): `*`
- Options:
  - `input`: `all`
  - `output`: `source`

### `!gmonthup`

Lists the top uploading group of all time.

Usage:

```
!gmonthup
```

- Default permission (`perms`): `*`
- Options:
  - `input`: `all`
  - `output`: `source`

### `!gpassed`

Lists if group has passed trial.

Usage:

```
!gpassed <Group>
```

- Default permission (`perms`): `=siteop`
- Options:
  - `input`: `public`
  - `output`: `source`

### `!groups`

Lists the groups on the site.

Usage:

```
!groups
```

- Default permission (`perms`): `=siteop`
- Options:
  - `input`: `all`
  - `output`: `source`

### `!grpren`

Rename a group.

Usage:

```
!grpren <oldgroup> <newgroup>
```

- Default permission (`perms`): `=siteop`
- Options:
  - `input`: `all`
  - `output`: `source`

### `!gtop`

Lists all users in trial period.

Usage:

```
!gtop <num>
```

- Default permission (`perms`): `*`
- Options:
  - `input`: `all`
  - `output`: `source`

### `!gwkdn`

Lists the top downloading group for the week.

Usage:

```
!gwkdn
```

- Default permission (`perms`): `*`
- Options:
  - `input`: `all`
  - `output`: `source`

### `!gwkup`

Lists the top uploading group for the week.

Usage:

```
!gwkup
```

- Default permission (`perms`): `*`
- Options:
  - `input`: `all`
  - `output`: `source`

### `!help`

Duh! it does this dummy!

Usage:

```
!help <command> will display specific information on that command.
```

- Default permission (`perms`): `*`
- Options:
  - `input`: `all`
  - `output`: `source`

### `!idlers`

Show who's idling.

Usage:

```
!idlers
```

- Default permission (`perms`): `*`
- Options:
  - `input`: `all`
  - `output`: `source`

### `!indexstatus`

Index status.

Usage:

```
!indexstatus
```

- Default permission (`perms`): `=siteop`
- Options:
  - `input`: `all`
  - `output`: `source`

### `!kick`

Kick a users current connections to the site.

Usage:

```
!kick <user> [message]
```

- Default permission (`perms`): `=siteop`
- Options:
  - `input`: `all`
  - `output`: `source`

### `!kickall`

Kick all users current connections to the site.

Usage:

```
!kickall [message]
```

- Default permission (`perms`): `=siteop`
- Options:
  - `input`: `all`
  - `output`: `source`

### `!kickslave`

Kick a slave offline.

Usage:

```
!kickslave <SLAVENAME>
```

- Default permission (`perms`): `=siteop`
- Options:
  - `input`: `all`
  - `output`: `source`

### `!kill`

Kill's a specific connection from the FTP by thread ID.

Usage:

```
!kill <threadID> [message]
```

- Default permission (`perms`): `=siteop`
- Options:
  - `input`: `all`
  - `output`: `source`

### `!leechers`

Show who's downloading.

Usage:

```
!leechers
```

- Default permission (`perms`): `*`
- Options:
  - `input`: `all`
  - `output`: `source`

### `!link`

Creates a symbolic link.

Usage:

```
!link <target> <linkname>
```

- Default permission (`perms`): `=siteop`
- Options:
  - `input`: `all`
  - `output`: `source`

### `!list`

This will raw list files in the current directory.

Usage:

```
!list
```

- Default permission (`perms`): `*`
- Options:
  - `input`: `all`
  - `output`: `source`

### `!master-uptime`

Will show current master uptime.

Usage:

```
!master-uptime
```

- Default permission (`perms`): `=siteop`
- Options:
  - `input`: `all`
  - `output`: `source`

### `!monthdn`

Lists the top downloaders this month.

Usage:

```
!monthdn
```

- Default permission (`perms`): `*`
- Options:
  - `input`: `all`
  - `output`: `source`

### `!monthup`

Lists the top uploaders this month.

Usage:

```
!monthup
```

- Default permission (`perms`): `*`
- Options:
  - `input`: `all`
  - `output`: `source`

### `!new`

Will list newly uploaded directories.

Usage:

```
!new - List the newest dirs in all sections.
!new <number> - List the <number> newest dirs in all sections.
!new <section> - List the newest dirs in <section>.
!new <section> <number> - List the <number> newest dirs in <section>.
```

- Default permission (`perms`): `*`
- Options:
  - `default`: `5`
  - `max`: `10`
  - `filtered_sections`: `REQUESTS`
  - `input`: `all`
  - `output`: `source`

### `!nuke`

Nuke a directory.

Usage:

```
!nuke <directory> <multiplier> <message>
```

- Default permission (`perms`): `=nuke =siteop`
- Options:
  - `search`: `true`
  - `input`: `all`
  - `output`: `source`

### `!nukes`

List all nuked directories.

Usage:

```
!nukes [section]
```

- Default permission (`perms`): `*`
- Options:
  - `input`: `all`
  - `output`: `source`

### `!nukesclean`

Clear nukelog of all invalid nukes.

Usage:

```
!nukesclean
```

- Default permission (`perms`): `=nuke =siteop`
- Options:
  - `input`: `all`
  - `output`: `source`

### `!passed`

Lists if user has passed trial.

Usage:

```
!passed <User>
```

- Default permission (`perms`): `*`
- Options:
  - `input`: `all`
  - `output`: `source`

### `!passwd`

Change your password.

Usage:

```
!passwd <password>
```

- Default permission (`perms`): `*`
- Options:
  - `input`: `all`
  - `output`: `source`

### `!pre`

Moves dir to specified section.

Usage:

```
!pre <dir> <section>
```

- Default permission (`perms`): `*`
- Options:
  - `input`: `all`
  - `output`: `source`

### `!purge`

Permanently remove a user from the site.

Usage:

```
!purge <username>
```

- Default permission (`perms`): `=gadmin =siteop`
- Options:
  - `input`: `all`
  - `output`: `source`

### `!readd`

Readd a deleted user.

Usage:

```
!readd <user>
```

- Default permission (`perms`): `=gadmin =siteop`
- Options:
  - `input`: `all`
  - `output`: `source`

### `!rebuildindex`

Rebuild index.

Usage:

```
!rebuildindex
```

- Default permission (`perms`): `=siteop`
- Options:
  - `input`: `all`
  - `output`: `source`

### `!refreshindex`

Requests a recursive refresh of data held for the current directory in the index.

Usage:

```
!refreshindex [-q]
The -q option suppresses details of the progress of the refresh.
```

- Default permission (`perms`): `=siteop`
- Options:
  - `input`: `all`
  - `output`: `source`

### `!reload`

Reload the config files.

Usage:

```
!reload
```

- Default permission (`perms`): `=siteop`
- Options:
  - `input`: `all`
  - `output`: `source`

### `!remapprove`

Removes Approval of release.

Usage:

```
!remapprove [path]
```

- Default permission (`perms`): `=siteop`
- Options:
  - `input`: `all`
  - `output`: `source`

### `!remerge`

Rebuilds the file database of the specified slave.

Usage:

```
!remerge <SLAVENAME>
```

- Default permission (`perms`): `=siteop`
- Options:
  - `input`: `all`
  - `output`: `source`

### `!remergequeue`

Displays remergequeue status.

Usage:

```
!remergequeue
```

- Default permission (`perms`): `=siteop`
- Options:
  - `input`: `public`
  - `output`: `source`

### `!removeexpire`

Removes expiry date for user.

Usage:

```
!removeexpire <username>
```

- Default permission (`perms`): `=siteop`
- Options:
  - `input`: `all`
  - `output`: `source`

### `!renuser`

Rename a user.

Usage:

```
!renuser <old_username> <new_username>
```

- Default permission (`perms`): `=siteop`
- Options:
  - `input`: `all`
  - `output`: `source`

### `!rules`

Displays the site rules.

Usage:

```
!rules
```

- Default permission (`perms`): `*`
- Options:
  - `file`: `rules.txt`
  - `input`: `all`
  - `output`: `source`

### `!sections`

List the currently defined sections on the site.

Usage:

```
!sections
```

- Default permission (`perms`): `*`
- Options:
  - `input`: `all`
  - `output`: `source`

### `!seen`

Show when a user last connected.

Usage:

```
!seen <user>
```

- Default permission (`perms`): `*`
- Options:
  - `input`: `all`
  - `output`: `source`

### `!setexpire`

Sets expiry date for User <yyyy-MM-dd>.

Usage:

```
!setexpire <username> <expiry-date>
```

- Default permission (`perms`): `=siteop`
- Options:
  - `input`: `all`
  - `output`: `source`

### `!shutdown`

Shutdown the ftp server.

Usage:

```
!shutdown
```

- Default permission (`perms`): `=siteop`
- Options:
  - `input`: `all`
  - `output`: `source`

### `!slave`

Display a slaves status.

Usage:

```
!slave <slavename> <more>
```

- Default permission (`perms`): `=siteop`
- Options:
  - `input`: `all`
  - `output`: `source`

### `!slave-select`

Gives the value of each slave for your current SlaveSelection config.

Usage:

```
!slave-select <TYPE> </FULLPATH>
<TYPE> is one of up,down,jobdown,jobup.
</FULLPATH> needs to start with "/" should not exist.
```

- Default permission (`perms`): `=siteop`
- Options:
  - `input`: `all`
  - `output`: `source`

### `!slave-uptime`

Will show current slave uptime.

Usage:

```
!slave-uptime <slave>
```

- Default permission (`perms`): `=siteop`
- Options:
  - `input`: `all`
  - `output`: `source`

### `!slaves`

List the slave added to the site and the status of each.

Usage:

```
!slaves
```

- Default permission (`perms`): `=siteop`
- Options:
  - `input`: `all`
  - `output`: `source`

### `!slaves-uptime`

Will show current slaves uptime.

Usage:

```
!slaves-uptime
```

- Default permission (`perms`): `=siteop`
- Options:
  - `input`: `all`
  - `output`: `source`

### `!speed`

Show the state of the specified users connections.

Usage:

```
!speed
```

- Default permission (`perms`): `*`
- Options:
  - `input`: `all`
  - `output`: `source`

### `!stat`

Show simple user stats.

Usage:

```
!stat
```

- Default permission (`perms`): `*`
- Options:
  - `input`: `all`
  - `output`: `source`

### `!stats`

Show user stats.

Usage:

```
!stats
```

- Default permission (`perms`): `*`
- Options:
  - `input`: `all`
  - `output`: `source`

### `!status`

Will show current status of the daemon.

Usage:

```
!status Usage:
!status <os,vm,memory,threads,gc,classes,cm,cpu,network,hardware,slaves,all>
!status hardware <master|slavename> - detail of one machine (alias: hw)
!status OS - Will show Operational System Info
!status VM - Will show VM Info.
!status MEMORY - Will show Memory Info.
!status THREADS - Will show Threads Info.
!status GC - Will show Garbage Collector Info.
!status CLASSES - Will show Classes Info.
!status CM - Will show Commit Manager Info.
!status CPU - Will show CPU Info.
!status NETWORK - Will show Network Info.
!status HARDWARE (or HW) - Will show a hardware summary (OS, cores, CPU, RAM, disks) of the master and of each slave. It can take a few seconds while slaves answer, so it is not included in ALL (like SLAVES). RAM is total minus available memory (on Linux the page cache counts as available).
!status HARDWARE <master|slavename> - Will show the hardware detail of the master or of one slave, including each root (shown as root 1, 2, ...).
!status SLAVES - Will show Slave TLS Info (not included in ALL).
!status ALL - Shortcut to display ALL Info (except SLAVES and HARDWARE).
```

- Default permission (`perms`): `=siteop`
- Options:
  - `input`: `all`
  - `output`: `source`

### `!swap`

Swap Credits Between Users.

Usage:

```
!swap <srcuser> <destuser> <bytes>
```

- Default permission (`perms`): `=siteop`
- Options:
  - `input`: `all`
  - `output`: `source`

### `!swho`

Show who's online.

Usage:

```
!swho
```

- Default permission (`perms`): `=siteop`
- Options:
  - `input`: `all`
  - `output`: `source`

### `!tagline`

Change your tagline.

Usage:

```
!tagline <tagline>
```

- Default permission (`perms`): `*`
- Options:
  - `input`: `all`
  - `output`: `source`

### `!take`

Remove credits from a user.

Usage:

```
!take <user> <bytes>
```

- Default permission (`perms`): `=siteop`
- Options:
  - `input`: `all`
  - `output`: `source`

### `!time`

Show current server time.

Usage:

```
!time
```

- Default permission (`perms`): `*`
- Options:
  - `input`: `all`
  - `output`: `source`

### `!top`

Lists all users in trial period.

Usage:

```
!top <num>
```

- Default permission (`perms`): `*`
- Options:
  - `input`: `all`
  - `output`: `source`

### `!traffic`

Summary of the daemon upload/download stats.

Usage:

```
!traffic
```

- Default permission (`perms`): `*`
- Options:
  - `input`: `all`
  - `output`: `source`

### `!unban`

Removes a ban for the specified user.

Usage:

```
!unban <user>
```

- Default permission (`perms`): `=ban =siteop`
- Options:
  - `input`: `all`
  - `output`: `source`

### `!unbanall`

Removes ban from all users.

Usage:

```
!unbanall
```

- Default permission (`perms`): `=ban =siteop`
- Options:
  - `input`: `all`
  - `output`: `source`

### `!unnuke`

Unnuke a directory.

Usage:

```
!unnuke <directory>
```

- Default permission (`perms`): `=nuke =siteop`
- Options:
  - `search`: `true`
  - `input`: `all`
  - `output`: `source`

### `!uploaders`

Show who's uploading.

Usage:

```
!uploaders
```

- Default permission (`perms`): `*`
- Options:
  - `input`: `all`
  - `output`: `source`

### `!user`

Show details for a user.

Usage:

```
!user <user>
```

- Default permission (`perms`): `*`
- Options:
  - `input`: `all`
  - `output`: `source`

### `!users`

List all usernames added to the site.

Usage:

```
!users
```

- Default permission (`perms`): `=siteop`
- Options:
  - `input`: `all`
  - `output`: `source`

### `!version`

Show DrFTPD version.

Usage:

```
!version
```

- Default permission (`perms`): `*`
- Options:
  - `input`: `all`
  - `output`: `source`

### `!welcome`

Displays site welcome message.

Usage:

```
!welcome
```

- Default permission (`perms`): `*`
- Options:
  - `file`: `welcome.txt`
  - `input`: `all`
  - `output`: `source`

### `!who`

Show who's online.

Usage:

```
!who
```

- Default permission (`perms`): `=siteop`
- Options:
  - `input`: `all`
  - `output`: `source`

### `!wipe`

Deletes files without removing credits. -r means delete recursively.

Usage:

```
!wipe [-r] <file/dir>
```

- Default permission (`perms`): `=wipe =siteop`
- Options:
  - `input`: `all`
  - `output`: `source`

### `!wkdn`

Lists the top downloaders this week.

Usage:

```
!wkdn
```

- Default permission (`perms`): `*`
- Options:
  - `input`: `all`
  - `output`: `source`

### `!wkup`

Lists the top uploaders this week.

Usage:

```
!wkup
```

- Default permission (`perms`): `*`
- Options:
  - `input`: `all`
  - `output`: `source`

### `!xdupe`

The X-DUPE mechanism helps you saving time when uploading files by avoiding "dupes".

Usage:

```
!xdupe <mode>
Mode 1: Up to a maximum of 80 chars total line length. Filenames longer than 66 chars will be truncated.
Mode 2: The server sends only one filename per X-DUPE line, and the maximum line length is 80 chars. Filenames are also truncated.
Mode 3: Only one filename is sent per X-DUPE line, and the filename will not be truncated.
Mode 4: All files are listed on one long line, up to a maximum of 1024 characters. Files which are too long to fit will be skipped.
```

- Default permission (`perms`): `*`
- Options:
  - `input`: `all`
  - `output`: `source`

## Archive (`archive.conf.dist`)

Feature documentation: [Archive](Archive.md).

Commands: `!archive`, `!listarchivequeue`, `!listarchivetypes`

### `!archive`

Create a new ArchiveHandler like defined in archive.conf

Usage:

```
!archive Directory [ArchiveType] [property1] [property2] [property...]
- An example of a property is slavename.1=myslave just like in archive.conf
```

- Default permission (`perms`): `=siteop`
- Options:
  - `badarchivetype=ArchiveType`: `${archivetypename} was unable to be initialized - ${exception}`
  - `incompatible`: `ArchiveType ${archivetypename} is incompatible with this version of DrFTPD`
  - `baddir`: `Directory ${dirname} was unable to be found`
  - `badslave`: `Unable to add ${slavename} to the slaveList`
  - `success`: `Started ArchiveHandler for ${archivetypename} on ${dirname}`
  - `fail`: `${exception}`
  - `input`: `all`
  - `output`: `source`

### `!listarchivequeue`

List the currently running ArchiveTypes

- Default permission (`perms`): `=siteop`
- Options:
  - `input`: `all`
  - `output`: `source`

### `!listarchivetypes`

List the currently running ArchiveTypes.

Usage:

```
!listarchivetypes
```

- Default permission (`perms`): `=siteop`
- Options:
  - `input`: `all`
  - `output`: `source`

## Autonuke (`autonuke.conf.dist`)

Feature documentation: [Autonuke](Autonuke.md).

Commands: `!autonukes`, `!autonukescan`, `!delqueue`

### `!autonukes`

Show AutoNuke queue.

Usage:

```
!autonukes [section]
```

- Default permission (`perms`): `=siteop`
- Options:
  - `input`: `all`
  - `output`: `source`

### `!autonukescan`

Manual AutoNuke scan.

Usage:

```
!autonukescan [path]
```

- Default permission (`perms`): `=siteop`
- Options:
  - `input`: `all`
  - `output`: `source`

### `!delqueue`

Delete one/all item(s) from AutoNuke queue.

Usage:

```
!delqueue [path]
```

- Default permission (`perms`): `=siteop`
- Options:
  - `input`: `all`
  - `output`: `source`

## Dupecheck (`dupecheck.conf.dist`)

Feature documentation: [Dupecheck](Dupecheck.md).

Commands: `!undupe`

### `!undupe`

UnDupe a File/Directory.

Usage:

```
!undupe <directory/file>
```

- Default permission (`perms`): `=siteop`
- Options:
  - `input`: `all`
  - `output`: `source`

## Find (`find.conf.dist`)

Feature documentation: [Find](Find.md).

Commands: `!dupe`, `!find`, `!search`

### `!dupe`

Searches for files and directories using exact name match.

Usage:

```
!dupe <name>
```

- Default permission (`perms`): `*`
- Options:
  - `limit`: `5`
  - `observe.privpath`: `false`
  - `type`: `f`
  - `exact`: `true`
  - `input`: `all`
  - `output`: `source`

### `!find`

Searches for files and directories in the filesystem w/ some neat filters and actions.

Usage:

```
!find <options> <actions>
!find options [name] - for a list of all registered options, if the name is provided print details of the option
!find actions - for a list of all registered actions
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
Example: !FIND -printf "filename: #f size: #s"
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
Example: !FIND -dupe -section MP3 -wipe
```

- Default permission (`perms`): `*`
- Options:
  - `limit.default`: `5`
  - `limit.max`: `20`
  - `observe.privpath`: `false`
  - `sendToSlaves`: `=siteop`
  - `deleteFromSlaves`: `=siteop`
  - `wipe`: `=siteop`
  - `nuke`: `=siteop =nuke`
  - `unnuke`: `=siteop =nuke`
  - `dupewipe`: `=siteop`
  - `dupenuke`: `=siteop`
  - `input`: `all`
  - `output`: `source`

### `!search`

Searches for files and directories using wildcards.

Usage:

```
!search <word> [word ..]
```

- Default permission (`perms`): `*`
- Options:
  - `limit`: `5`
  - `observe.privpath`: `false`
  - `type`: `d`
  - `exact`: `false`
  - `input`: `all`
  - `output`: `source`

## IMDB (`imdb.conf.dist`)

Feature documentation: [IMDB-and-TVMaze](IMDB-and-TVMaze.md).

Commands: `!createimdb`, `!imdb`, `!imdbqueue`, `!removeimdb`

### `!createimdb`

Create IMDB metadata for all movies below current dir.

Usage:

```
!createimdb [path]
```

- Default permission (`perms`): `=siteop`
- Options:
  - `input`: `all`
  - `output`: `source`

### `!imdb`

Search for a movie.

Usage:

```
!imdb <nameofmovie>
```

- Default permission (`perms`): `*`
- Options:
  - `input`: `all`
  - `output`: `source`

### `!imdbqueue`

Show number of items in the queue to be processed.

Usage:

```
!imdbqueue
```

- Default permission (`perms`): `=siteop`
- Options:
  - `input`: `all`
  - `output`: `source`

### `!removeimdb`

Remove IMDB metadata for all movies below current dir.

Usage:

```
!removeimdb [path]
```

- Default permission (`perms`): `=siteop`
- Options:
  - `input`: `all`
  - `output`: `source`

## Jobs (`jobs.conf.dist`)

Feature documentation: [Jobs](Jobs.md).

Commands: `!addjob`, `!listjobs`, `!listrunningjobs`, `!removejob`, `!removejobs`, `!startjobs`, `!stopjobs`

### `!addjob`

Add a job to the JobManager queue.

Usage:

```
!addjob <file> <priority> <timestomirror> <destslave> [destslave ...]
<file> : The file to archive.
<priority> : The priority you want this job to have. Must be an Integer.
<timestomirror> : The max number of slaves each file should be on.
<destslave> : The name of the slave to transfer the files to.
```

- Default permission (`perms`): `=siteop`
- Options:
  - `input`: `public`
  - `output`: `source`

### `!listjobs`

List the current jobs queued by the JobManager by index #.

Usage:

```
!listjobs
```

- Default permission (`perms`): `=siteop`
- Options:
  - `input`: `public`
  - `output`: `source`

### `!listrunningjobs`

List the current jobs queued by the JobManager by index #.

Usage:

```
!listrunningjobs
```

- Default permission (`perms`): `=siteop`
- Options:
  - `input`: `public`
  - `output`: `source`

### `!removejob`

Removes a Job from the JobManager queue by index # or range.

Usage:

```
!removejob <jobindex> <lowrange>-<highrange> will remove the job with
jobindex and all jobs including and between lowrange and highrange.
You can use as many ranges as your heart desires.
```

- Default permission (`perms`): `=siteop`
- Options:
  - `input`: `public`
  - `output`: `source`

### `!removejobs`

Removes all jobs that are not currently transferring.

Usage:

```
!removejobs
```

- Default permission (`perms`): `=siteop`
- Options:
  - `input`: `public`
  - `output`: `source`

### `!startjobs`

Starts running the jobs in the JobManager queue.

Usage:

```
!startjobs
```

- Default permission (`perms`): `=siteop`
- Options:
  - `input`: `public`
  - `output`: `source`

### `!stopjobs`

Stops running the jobs in the JobManager queue.

Usage:

```
!stopjobs
```

- Default permission (`perms`): `=siteop`
- Options:
  - `input`: `public`
  - `output`: `source`

## Mirror (`mirror.conf.dist`)

Feature documentation: [Mirror](Mirror.md).

Commands: `!unmirror`

### `!unmirror`

Unmirror path so files are kept on one slave only.

Usage:

```
!unmirror <path>
```

- Default permission (`perms`): `=siteop`
- Options:
  - `input`: `all`
  - `output`: `source`

## Request (`request.conf.dist`)

Feature documentation: [Requests](Requests.md).

Commands: `!fixrequests`, `!reqdel`, `!reqfilled`, `!request`, `!requests`

### `!fixrequests`

Tries to fix requests dir with the actual requests

- Default permission (`perms`): `=siteop`
- Options:
  - `input`: `all`
  - `output`: `source`

### `!reqdel`

Deletes a request.

Usage:

```
!reqdel <nameofrequest>
```

- Default permission (`perms`): `=request =siteop`
- Options:
  - `input`: `all`
  - `output`: `source`
  - `request.deleteOthers`: `=siteop`

### `!reqfilled`

Fill a request.

Usage:

```
!reqfilled <nameofrequest>
```

- Default permission (`perms`): `*`
- Options:
  - `input`: `all`
  - `output`: `source`

### `!request`

Add a request.

Usage:

```
!request <nameofrequest>
```

- Default permission (`perms`): `=request =siteop`
- Options:
  - `input`: `all`
  - `output`: `source`
  - `request.deleteOthers`: `=siteop`

### `!requests`

List current requests.

- Default permission (`perms`): `*`
- Options:
  - `input`: `all`
  - `output`: `source`

## Sitebot (`sitebot.conf.dist`)

Feature documentation: [Sitebot](Sitebot.md).

Commands: `!blowfish`, `!getkey`, `!ident`, `!invite`, `!irc`, `!setblowfish`, `!setkey`, `!unident`

### `!blowfish`

List blowfish keys for irc channels.

Usage:

```
!blowfish
!blowfish <botname>
<botname> is required to use a secondary sitebot.
```

- Default permission (`perms`): `*`
- Options:
  - `input`: `private`
  - `output`: `source`

### `!getkey`

List blowfish keys for irc channels.

Usage:

```
!getkey
!getkey <botname>
<botname> is required to use a secondary sitebot.
```

- Default permission (`perms`): `*`
- Options:
  - `input`: `private`
  - `output`: `source`

### `!ident`

Associates an irc nick with a ftp user.

Usage:

```
!ident <username> <password>
```

- Default permission (`perms`): `%`
- Options:
  - `input`: `private`
  - `output`: `source`

### `!invite`

Invite yourself into irc channels using the sitebot.

Usage:

```
!invite <username> <password>
```

- Default permission (`perms`): `*`
- Options:
  - `input`: `private`
  - `output`: `source`

### `!irc`

Control the IRC sitebot.

Usage:

```
!irc connect
!irc <botname> connect
!irc disconnect
!irc <botname> disconnect
!irc reconnect
!irc <botname> reconnect
!irc raw <commmand>
!irc <botname> raw <command>
!irc say <message>
!irc <botname> say <message>
!irc emptyqueue
<botname> is mandatory if more than one sitebot running.
```

- Default permission (`perms`): `=siteop`
- Options:
  - `input`: `private`
  - `output`: `source`

### `!setblowfish`

Set your personal blowfish key on the sitebot.

Usage:

```
!setblowfish <blowkey>
!setblowfish <botname> <blowkey>
<botname> is required to use a secondary sitebot.
```

- Default permission (`perms`): `*`
- Options:
  - `input`: `private`
  - `output`: `source`

### `!setkey`

Set your personal blowfish key on the sitebot.

Usage:

```
!setkey <blowkey>
!setkey <botname> <blowkey>
<botname> is required to use a secondary sitebot.
```

- Default permission (`perms`): `*`
- Options:
  - `input`: `private`
  - `output`: `source`

### `!unident`

Unassociates an irc nick with a ftp user.

Usage:

```
!unident <username> <password>
```

- Default permission (`perms`): `*`
- Options:
  - `input`: `private`
  - `output`: `source`

## Speedtest.net (`speedtest.net.conf.dist`)

Feature documentation: [Speedtest](Speedtest.md).

Commands: `!refreshspeedtest`, `!speedtest`

### `!refreshspeedtest`

Force an update of speedtest.net server list.

Usage:

```
!refreshspeedtest
```

- Default permission (`perms`): `=siteop`
- Options:
  - `input`: `all`
  - `output`: `source`

### `!speedtest`

Run a speedtest on slave against speedtest.net test servers, * for all slaves.

Usage:

```
!speedtest [<slavename> [-list | -update | <test server id>]]
-list option in conjunction with slave name lists closests servers to slave location based on its IP
-update option in conjunction with slave name to force location update
An optional test server id can be specified to force test server.
```

- Default permission (`perms`): `=siteop`
- Options:
  - `input`: `all`
  - `output`: `source`

## TVMaze (`tvmaze.conf.dist`)

Feature documentation: [IMDB-and-TVMaze](IMDB-and-TVMaze.md).

Commands: `!createtv`, `!removetv`, `!tv`, `!tvqueue`

### `!createtv`

Create TvMaze metadata for all shows below current dir.

Usage:

```
!createtv [path]
```

- Default permission (`perms`): `=siteop`
- Options:
  - `input`: `all`
  - `output`: `source`

### `!removetv`

Remove TvMaze metadata for all shows below current dir.

Usage:

```
!removetv [path]
```

- Default permission (`perms`): `=siteop`
- Options:
  - `input`: `all`
  - `output`: `source`

### `!tv`

Search for a tv show (-v for verbose output).

Usage:

```
!tv [-v] <nameofshow> [episodenumber]
```

- Default permission (`perms`): `*`
- Options:
  - `input`: `all`
  - `output`: `source`

### `!tvqueue`

Show number of items in the queue to be processed.

Usage:

```
!tvqueue
```

- Default permission (`perms`): `=siteop`
- Options:
  - `input`: `all`
  - `output`: `source`

## Zipscript (`zipscript.conf.dist`)

Feature documentation: [Zipscript](Zipscript.md).

Commands: `!rescan`

### `!rescan`

SFV scans the files in the current directory or a specific directory requested.

Usage:

```
!rescan [-r] [noforce] [nodelete] [nodelete0byte] [quiet] [file/path]
The -r option recursively scans all directories below the current directory as well.
The noforce option makes the slave recalculate the checksums.
The nodelete option removes bad files rather than just reporting them.
The nodelete0byte option removes any files listed in the SFV which exist but are 0 byte.
The quiet option only prints output for files which aren't good.
Specifying a file or path will rescan the requested file or directory instead of the current directory.
If the file or path does not start with '/' it will be relative to your current directory
```

- Default permission (`perms`): `*`
- Options:
  - `input`: `all`
  - `output`: `source`
