> Applies to DrFTPD 4.0.x.

# Users and Groups

This page explains how DrFTPD stores users and groups, which settings a new account starts with,
what `SITE CHANGE` can modify, how group admins work, and how credits and ratios behave. The commands
themselves are listed on [Site Commands](Site-Commands.md); path and command permissions are on
[Permissions](Permissions.md).

## The user model

A user has:

- a name (no spaces, no `;`, no `!`) and a password
- a primary group and any number of secondary groups
- one or more `ident@ip` host masks that decide where the user may connect from
- credits, a ratio and limits (see below)
- statistics: bytes and files up and down, per day, week, month and total
- free text fields: tagline, comment, and the IRC identity set by the sitebot

A group has a name (no spaces, no `;`), a creation date, a list of group admins, the number of slots
and leech slots, and a minimum and maximum ratio for its admins.

## Where users are stored

With the default user manager (`usermanager` in [Master Configuration](Master-Configuration.md)), the
master keeps one JSON file per user and per group, written in pretty printed form:

```
userdata/users/javabeans/<username>.json
userdata/groups/javabeans/<groupname>.json
```

The folders are created when the master starts. Files are written when a user or group changes, and
`disk.commit.delay` in `master.conf` controls how long changes may be held in memory before they are
written. The files are read when needed, so editing them while the master runs is not safe: stop the
master first, or use the `SITE` commands. Back up the whole `userdata/` folder.

Passwords are not stored in clear text. The hash scheme is chosen with `passcrypt` in
`config/encryptedbeanuser.conf`:

```
# Default: [bcrypt]
passcrypt=bcrypt
```

Valid values (case insensitive): `bcrypt`, `argon2`, `glftpd`, `md2`, `md5`, `sha-1`, `sha-256`, `sha-384`,
`sha-512`, and `none`, `clear` or `empty` (clear text, not recommended). Use `bcrypt` or `argon2`. The
other hashes are fast, unsalted hashes that are only kept so existing user files keep working, and
the master logs a warning at startup if they, or clear text, are used. If `passcrypt` is missing or not
recognised, an error is logged and `bcrypt` is used. A password stored with another scheme keeps
working and is converted to the configured scheme on the user's next successful login.

### First start

If there are no user files when the master starts, it creates:

- the groups `drftpd` and `siteop`
- the user `drftpd`, with the password `drftpd`, primary group `drftpd`, secondary group `siteop`, ratio 0 (leech),
  no login or transfer slot limits, and the host masks `*@127.0.0.1` and `*@0:0:0:0:0:0:0:1`

See [First Login](First-Login.md) and change that password straight away.

## Special groups

Most group names mean nothing to DrFTPD, they are only labels you use in `perms` lines. A few names are
treated specially by the code:

| Group | Meaning |
| --- | --- |
| `siteop` | Site administrators. A member (primary or secondary) is "admin": allowed to use all of `SITE CHANGE`, add and delete groups, change group settings, and so on. The default `perms` of most admin commands is `=siteop` |
| `deleted` | Marks deleted users. `SITE DELUSER` adds the user to it (and removes their group admin rights), `SITE READD` takes them out. A user in this group cannot log in. The group is created automatically if it is missing |
| `gadmin` | Not a real group. In a `perms` rule, `=gadmin` matches every user who is group admin of at least one group (see below) |
| `drftpd` | The group created together with the default user. It has no special meaning afterwards, but it is the default group for `SITE PRE` (`default.pre.group` in `master.conf`) |

Other names you see in the default files, such as `=ban` (`SITE BAN` and friends), `=nuke`, `=wipe` or
`=exempt`, are ordinary groups that only matter because a `perms` line mentions them. Create them
with `SITE ADDGROUP` and add users to them with `SITE CHGRP` if you want to use them.

## Creating users and groups

Users are created with `SITE ADDUSER <user> <password> [<ident@ip> ...]` (siteops and group admins) or
`SITE GADDUSER <group> <user> <password> [<ident@ip> ...]` (siteops). Groups are created with
`SITE ADDGROUP <group>` (siteops).

A new user starts with the values of `config/defaultuser.conf`. Every key is optional, the value
shown is used when the key is missing:

| Key | Default | Meaning |
| --- | --- | --- |
| `ratio` | `3.0` | Upload credit ratio. `0` means leech: unlimited downloads |
| `max_logins` | `2` | Simultaneous logins of this user (0 = unlimited) |
| `max_logins_ip` | `2` | Simultaneous logins from one IP (0 = unlimited) |
| `max_uploads` | `2` | Simultaneous uploads (`max_sim`) |
| `max_downloads` | `2` | Simultaneous downloads (`max_sim`) |
| `idle_time` | `300` | Seconds without activity before the connection is closed (0 = never) |
| `wkly_allotment` | `0` | Weekly credit allotment, `0` disables it. Accepts units such as `10GB` |
| `credits` | `0b` | Starting credits. Accepts units such as `5GB` |
| `tagline` | `No tagline set.` | The tagline |
| `group` | `NoGroup` | Primary group of a user added with `SITE ADDUSER` |

The default group `NoGroup` is not created for you. Either create it (`SITE ADDGROUP NoGroup`) or set
`group` in `defaultuser.conf` to a group that exists, otherwise `SITE ADDUSER` fails with "Defaultuser
group does not exist". `SITE GADDUSER` puts the user in the group you name instead.

A new group starts with the values of `config/defaultgroup.conf`:

```
# min/max ratio that gadmins can set on their group members
min_ratio=3.0
max_ratio=3.0
```

and with 0 group slots and 0 leech slots.

Copy the `.dist` files to `defaultuser.conf` and `defaultgroup.conf` and edit those. They are read at the moment a user or group is created.

## Host masks (ident@ip)

A user can log in only from an address that matches one of their masks. A mask has the form
`ident@address`:

- `address` is a glob (`*` any characters, `?` one character) matched against both the IP address and
  the resolved host name of the client, for example `*@192.0.2.*` or `bob@*.example.com`
- the `ident` part is `*` for "do not check ident", otherwise it is matched (also as a glob) against the
  answer of the client's ident service (port 113). A mask without `@` is the same as `*@<mask>`
- masks are added with `SITE ADDIP <user> <mask> ...` and removed with `SITE DELIP`; both accept several masks
  and strip commas, so you can paste a list

`ident.lookup.timeout` in [Master Configuration](Master-Configuration.md) sets how long the master waits for
the ident answer.

Which masks a user may add can be limited with `config/ipsecurity.conf`, see
[Account policy files](#account-policy-files) below. The sitebot does not use these masks: it
recognises users by their linked IRC identity, see below.

## SITE CHANGE

```
SITE CHANGE <user> <field> <value>
SITE CHANGE =<group> <field> <value>
SITE CHANGE * <field> <value>
```

`SITE CHANGE` and `SITE CHANGEUSER` are the same command. The target is a user, `=group` for every member
of the group, or `*` for every user. Fields:

| Field | Value | Notes |
| --- | --- | --- |
| `ratio` | number, for example `3` or `1.5` | Upload credit ratio. `0` = leech (unlimited downloads). Group admins may use it too, see below |
| `credits` | bytes with optional unit, for example `5GB` | Sets the credits to this value (it does not add) |
| `idle_time` | seconds | Idle timeout. `0` = no timeout |
| `num_logins` | `<max> [<max per IP>]` | Simultaneous logins, and per IP. If the second number is omitted the per IP value is kept |
| `max_sim` | `<downloads> <uploads>` | Simultaneous downloads and uploads (download first) |
| `wkly_allotment` | bytes with optional unit | Weekly credit allotment, `0` disables it |
| `group` | group name | Primary group. The group must exist |
| `created` | `yyyy-MM-dd` | Creation date. Without a value, today |
| `tagline` | text | Tagline, the rest of the line is the value |
| `comment` | text | Comment, the rest of the line is the value |
| `homedir` | `<path>` or `none` | Home directory (an absolute path without `.` or `..`). `none` removes the override. See below |

Any other field gives the usage message. Only siteops can change anything other than `ratio`.

`homedir` overrides the global `homedir` of `master.conf` for that user. The path may use `${user}` and `${group}`.
It only sets the directory the user lands in after login, it is not a chroot and grants nothing: use
`privpath`, `upload` and `makedir` lines in `perms.conf` to restrict access. If the directory does not
exist or is hidden from the user, the user lands in `/`.

Other commands change other things: `SITE CHPASS` and `SITE PASSWD` (password), `SITE TAGLINE` (own tagline),
`SITE CHGRP` (secondary groups, toggles), `SITE RENUSER`, `SITE GIVE`, `SITE TAKE`, `SITE SWAP` (credits),
`SITE SETEXPIRE`, `SITE BAN`.

## SITE CHANGEGROUP

```
SITE CHANGEGROUP <group> <field> <value>
```

Siteops only. Fields:

| Field | Value | Notes |
| --- | --- | --- |
| `slots` | number | How many users the group's admins may add (see below) |
| `leechslots` | number | How many users of the group may have ratio 0 |
| `ratio` | `<min> <max>` | Lowest and highest ratio the group's admins may set. The minimum must be at least 1 and the maximum not below the minimum |
| `created` | `yyyy-MM-dd` | Creation date |

Note: the built in help text of `SITE CHANGEGROUP` only lists `created`, `slots` and `leechslots`, but the command also accepts `ratio`.

## Group admins

A group admin is a user who administers a group without being a siteop. Siteops make users group admins
with `SITE CHANGEGROUPADMIN <group> <user> [<user> ...]`: it toggles, a user who is already admin of that group
loses the right. Deleting a user removes their group admin rights.

Group admins can use the commands whose `perms` contain `=gadmin` (by default `SITE ADDUSER`, `SITE ADDIP`,
`SITE DELIP`, `SITE CHPASS`, `SITE CHANGE`, `SITE DELUSER`, `SITE PURGE`, `SITE READD`, `SITE GINFO`; `SITE DELGROUP` is listed too but the code only lets siteops delete groups), with these limits in the code:

- `SITE ADDUSER` adds the user to the admin's own primary group, and only if that group has a free slot:
  the number of users in the group must be below the group's `slots` value. Siteops are not limited.
- `SITE CHANGE` is limited to the `ratio` field, and only for members of a group the user administers.
  The ratio must lie between the group's `min_ratio` and `max_ratio`. A ratio of `0` (leech) is only
  allowed while the group has a free leech slot, that is, fewer members with ratio 0 than `leechslots`.
- `SITE DELUSER`, `SITE PURGE`, `SITE READD`, `SITE ADDIP`, `SITE DELIP`, `SITE CHPASS` work on members of a group the user administers.

## Credits and ratio

- A user's credits go up when they upload and down when they download.
- On upload the user gains `ratio x bytes` credits. The ratio is the user's own `ratio`, unless a
  `creditcheck` line in `perms.conf` matches the path and the user, in which case that line's ratio is used.
  A `creditcheck` ratio of 0 gives no credits for that path.
- On download the user loses `bytes x cost` credits. The cost is 1 unless a `creditloss` line matches. The
  download is refused with "Not enough credits" if the user has fewer credits than the file costs.
- A user with ratio 0 is a leech: downloads never cost credits (even if a `creditloss` line is set) and they
  never get "Not enough credits".
- Deleting an uploaded file (`DELE`) takes back the credits and the upload stats it earned.
- `creditlimit` lines in `perms.conf` cap the bytes a user may transfer per day, week, month or in total.
- With `wkly_allotment` greater than 0, the user's credits are reset to that value at the start of every
  week (they are replaced, not added to). The week starts on Sunday, or on Monday if `european.cal=true` in
  `master.conf`. With `wkly_allotment` 0, nothing happens.
- Siteops can adjust credits with `SITE GIVE`, `SITE TAKE`, `SITE SWAP` and `SITE CHANGE <user> credits`.
  `SITE CREDITS` shows your own credits.
- `SITE FAIRNESS` compares uploaded to downloaded bytes of users, see [Master Configuration](Master-Configuration.md).

The path rules `creditcheck`, `creditloss` and `creditlimit` are described on [Permissions](Permissions.md).

## Deleting, purging and expiring users

- `SITE DELUSER <user> [reason]` marks the user as deleted. The file stays and the user cannot log in.
  `SITE READD` restores the user.
- `SITE PURGE` (or `SITE DELPURGE`) removes the user file for good.
- `SITE SETEXPIRE <user> <yyyy-MM-dd>` sets an expiry date and `SITE REMOVEEXPIRE` removes it. A user whose date has passed
  cannot log in, and once a day the master applies the actions of `config/expireduser.conf` to expired users:

```
delete = true        # mark the user deleted
purge  = true        # remove the user for good
chgrp  = group1 group2   # toggle these secondary groups
setgrp = groupname   # set the primary group
```

All keys are optional and `delete` and `purge` default to `false`. If `purge` is true the other actions are not run. The dist file ships with
everything commented out, so by default expired users only lose the ability to log in.

## Account policy files

These files are read at start and again on `SITE RELOAD`. Each ships as `.dist`: copy it to the name
without `.dist` to change it.

### securepass.conf: password strength

```
#          LENGTH LOWER UPPER NUMERIC SPECIAL PERMS
securepass    5     2     2      1       0    *
securepass    7     2     2      2       2    =siteop
```

Columns: minimum length, minimum lower case letters, upper case letters, digits and special characters
(`0` = not required), then the rule saying who the line applies to. It is checked when `SITE ADDUSER`,
`SITE GADDUSER`, `SITE CHPASS` and `SITE PASSWD` set a password. **Every** line whose rule matches the user whose password is being set must be satisfied.
In the defaults a siteop must meet both lines. When a new user is added, only lines with `*` as rule apply.

### ipsecurity.conf: which masks users may add

```
#            IDENT  OCTETS   HOSTMASK  NUMIP  PERMS
ipsecurity     0    X.X.X.*     0        3     *
ipsecurity     0    *.X.X.X     0        3     *
ipsecurity     1    X.X.*.*     0        3     *
ipsecurity     1    *.*.X.X     1        3     *
ipsecurity     1    X.X.X.X     0        3     =siteop
```

It is checked when a host mask is added with `SITE ADDIP`, `SITE ADDUSER`, `SITE GADDUSER` or
`SITE SLAVE <name> ADDMASK`. A mask is accepted if at least one line accepts it. The loopback address
`127.0.0.1` is always accepted. The columns:

- `IDENT`: `1` means the mask must name an ident (it may not be `*@...`), `0` means anything goes
- `OCTETS`: the allowed shape of the address. `X` stands for a real number in that position and `*` for a wildcard.
  `X.X.X.*` allows `1.2.3.*`
- `HOSTMASK`: `1` allows a host name mask (`*.*.X.X` stands for `*.*.isp.example`), `0` allows only IP addresses
- `NUMIP`: the most masks an account may have, `0` is unlimited
- `PERMS`: the rule saying who the line applies to. For `SITE ADDIP` this is the user who gets the mask; when a
  new user is created or a slave mask is added only lines with `*` apply

### expireduser.conf

See the previous section.

## Sitebot: linking an IRC identity

The sitebot recognises a site user by a stored `bot|nick!user@host` IRC identity, written to the user file by the bot
command `!ident <user> <password>` (and by `!invite`). It is removed with `!unident`. See [IRC Commands](IRC-Commands.md).
