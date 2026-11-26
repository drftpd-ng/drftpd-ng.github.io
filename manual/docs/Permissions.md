> Applies to DrFTPD 4.0.x.

# Permissions

DrFTPD controls access in two places:

- `config/perms.conf` decides what users may do with files and paths, who may connect, and a few
  site wide settings.
- The `perms` line of each command in the command files (see [Site Commands](Site-Commands.md) and
  [IRC Commands](IRC-Commands.md)) decides who may run the command.

Both use the same rule syntax, described first. This page then lists every directive of
`perms.conf`.

`perms.conf` ships as `perms.conf.dist`. If `config/perms.conf` exists it is used, otherwise the
`.dist` file is. Copy the dist file to `perms.conf` and edit your copy. `SITE RELOAD` re-reads
`perms.conf`. If you start the master with only the `.dist` file and create `perms.conf` later,
restart the master once so that it picks the new file.

## Rule syntax

A rule is a list of entries separated by spaces. It is checked from left to right and the first
entry that matches the user decides.

| Entry | Matches |
| --- | --- |
| `*` | any logged in user |
| `%` | a connection that is not logged in (only meaningful in command `perms`, for commands such as `USER`, `PASS`, `AUTH`) |
| `=group` | a member of `group`, as primary or secondary group. `=gadmin` matches any user that is group admin of some group |
| `-user` | the user called `user` |
| `!entry` | the same, but a match means "refused" |

- If the first matching entry is a plain one, the user is allowed. If it is a `!` entry, the user is refused.
- If no entry matches, the user is refused.
- Group names and user names are case sensitive.

Examples:

```
=siteop                 # only siteops
=siteop =gadmin         # siteops and group admins
-alice -bob             # exactly these two users
!=banned *              # everybody except members of the group "banned"
!*                      # nobody
*                       # every logged in user
```

The order matters: `* !=banned` allows everybody, because `*` matches first.

An empty `perms` for a command is treated as `=siteop`.

## Path rules

Most directives of `perms.conf` have the form

```
directive   <path pattern>   <rule>
```

for example `upload /incoming/* =siteop -alice`. Everything after the pattern is a rule as
described above. Patterns cannot contain spaces.

### How a path is matched

For the directives marked "glob" below, the pattern is a shell style glob:

| Pattern | Meaning |
| --- | --- |
| `*` | any characters, including `/` and including nothing |
| `?` | exactly one character |
| `[abc]`, `[a-z]`, `[!abc]` | one character from the set (or not from the set with `!` or `^`) |
| `\` | escapes the next special character |

- The whole path must match, not just a part of it.
- Paths start with `/` and are case sensitive.
- Directories are matched with a trailing `/`. `/home/user1/*` therefore also matches the
  directory `/home/user1` itself, and `/pre/*` matches the directory `/pre`.
- `*` crosses directory boundaries. `/archive/*` matches `/archive/MP3/some.release/file.mp3`.

The "regex" directives (those ending in `ex`) use a Java regular expression instead. It is
case insensitive and may match anywhere in the path (it is a search, not a full match), so anchor it
with `^` and `$` if you need to. Regex paths have no trailing `/` for directories.

### First match wins

For one kind of permission (for example `upload`), the master looks at all lines in this order:

1. all regex lines (`uploadex`), top to bottom
2. all glob lines (`upload`), top to bottom

The first line whose path matches decides, and only then is the rule checked: if the user passes the
rule the action is allowed, otherwise it is refused. Later lines are not looked at. This means:

- Put specific lines before general ones. `upload /home/alice/* -alice` must come before
  `upload /home/* =siteop`.
- A line whose path matches but whose rule excludes the user refuses the action. It does not
  fall through to later lines.
- If no line matches, the action is refused (for `upload`, `makedir`, `delete`, `deleteown`, `rename`,
  `renameown`, `download`), with two exceptions: `wipe` is allowed, and `privpath` means "not private".
  This is why the default file contains `upload * *`, `download * *` and so on.

The directives `hideinwho`, `pre`, `denydownload`, `deny_upfxp`, `deny_dnfxp`, `dupecheck`, `nostatsup`,
`nostatsdn` also use first match wins on glob patterns, but with no matching line the answer is "no".
The directives `creditcheck`, `creditloss` and `creditlimit` differ: a line only counts if both the
path and the rule match the user, otherwise the next line is tried.

## Connection and login directives

### `login_prompt`

```
login_prompt DrFTPD 4.0.12-git https://github.com/drftpd-ng/drftpd
```

The text of the first line the server sends to a new connection (the `220` greeting). Everything
after the word is used. If you remove the line, the version and project address are shown.

### `max_users` and `exempt`

```
max_users   100   10
exempt      =siteop
```

`max_users <total> <exempt>`: normal users can connect until `<total>` connections exist. Users matching
the `exempt` rule can go on until `<total> + <exempt>` connections. Beyond that the server answers
"The site is full". The size of the connection thread pool is `total + exempt`.

If there is no `exempt` line at all, every user counts as exempt.

### `allow_connections` and `allow_connections_deny_reason`

```
allow_connections         *
allow_connections         =siteop
allow_connections_deny_reason Site closed for maintenance
```

Who may log in at all. Users who do not match the rule are refused when they log in. The default
is `*`. With no `allow_connections` line, everybody may log in. `allow_connections_deny_reason`
is the text sent to the refused user, the rest of the line is used.

### `bouncer_ips`

```
bouncer_ips bouncer.example.com 192.0.2.10
```

A list of IP addresses or host names of your FTP bouncers. A connection from such an address is not
checked against the user host masks when it connects (the check is done later, on the `IDNT` command, with the
real client address the bouncer sends). Without this, a bouncer cannot be used with host mask protection.

### `pasv_ports`

```
pasv_ports 50000-60000
```

One range of ports (`from-to`) the master uses for passive data connections to users that are
sent to the master itself. See [Firewalls and Passive Mode](Firewalls-and-Passive-Mode.md).

### `pasv_addr`

```
pasv_addr 203.0.113.5
```

The address announced in a `PASV` reply when the master cannot work out the right outside address
by itself (for example behind NAT). See [Firewalls and Passive Mode](Firewalls-and-Passive-Mode.md).

### `hide_ips`

```
hide_ips true
```

`true` (the default) or `false`. When true, the master hides client IP addresses in its log output.
Any value other than `true` (case insensitive) means false.

### TLS requirements

```
userrejectsecure        !*
userrejectinsecure      *
denydiruncrypted        *
denydatauncrypted       *
```

Each takes a plain rule (no path). See [SSL/TLS](SSL-TLS.md).

| Directive | Effect for users matching the rule |
| --- | --- |
| `userrejectinsecure` | cannot log in over a connection that is not TLS encrypted ("USE SECURE CONNECTION") |
| `userrejectsecure` | cannot log in over an encrypted connection ("USE INSECURE CONNECTION") |
| `denydiruncrypted` | directory listings over an unencrypted data connection are refused ("Secure Listing Required") |
| `denydatauncrypted` | file transfers over an unencrypted data connection are refused ("USE SECURE DATA CONNECTION") |

With the defaults above, nobody is rejected for using TLS, everybody is rejected for not using it, and
listings and transfers must be encrypted. If a line is missing, nobody is rejected by it.

## File and path directives (glob)

The first column is the directive, followed by `<path pattern> <rule>`.

| Directive | What it controls |
| --- | --- |
| `upload` | who may upload (create) a file at a path |
| `download` | who may download a file |
| `makedir` | who may create a directory |
| `delete` | who may delete files and directories owned by other users |
| `deleteown` | who may delete files they uploaded themselves |
| `rename` | who may rename or move items owned by other users |
| `renameown` | who may rename or move items they own |
| `wipe` | where `SITE WIPE` and `SITE FIND -wipe` may be used |
| `privpath` | paths that are hidden from users who do not match the rule |

Notes:

- `deleteown` and `delete` work together: the owner of an item is checked against `deleteown` only, anyone else
  against `delete`. So an owner needs a matching, permitting `deleteown` line to delete their own files, even if
  a `delete` line would allow them. Same for `renameown` and `rename`.
- `upload` and `makedir` match the path of the item to be created. `download` matches the file.
- `privpath` is inverted: the rule lists the users who may see the path. For everybody else the
  path does not exist. Example: `privpath /hidden_path/ =siteop`.
- `wipe`: who may use the `SITE WIPE` command at all is set in the command file. The `wipe` lines
  only limit where it may be used. With no matching line, wiping is allowed (the delete permissions still apply).
  A recursive wipe of a directory is refused as a whole if any item below it is denied by a `wipe`
  line, hidden paths included. `wipe` rules do not apply to deletes done by the system itself, such as
  [Autofreespace](Autofreespace.md), `REQDELETE` and similar.

Examples:

```
upload      *               *
download    *               *
makedir     *               *
deleteown   *               *
delete      *               =siteop
renameown   *               *
rename      *               =siteop
privpath    /hidden_path/   =siteop

# a private home directory for user1 (lines must come BEFORE the wildcard lines)
privpath    /home/user1/    -user1 =siteop
upload      /home/user1/*   -user1
makedir     /home/user1/*   -user1
upload      /home/*         =siteop
makedir     /home/*         =siteop

# wipe inside /archive/<section>/<release>, but not the section dirs or /archive itself
wipe        /archive/*/?*   =siteop =wipe
wipe        /archive/*      !*
wipe        *               =siteop =wipe
```

A home directory (see `homedir` in [Users and Groups](Users-and-Groups.md)) only sets where a user
lands after login. It grants no permission, restrict it with lines like the ones above.

## File and path directives (regex)

Each glob directive above has a regex twin:

| Directive | Twin of |
| --- | --- |
| `uploadex` | `upload` |
| `downloadex` | `download` |
| `makedirex` | `makedir` |
| `deleteex` | `delete` |
| `deleteownex` | `deleteown` |
| `renameex` | `rename` |
| `renameownex` | `renameown` |
| `wipeex` | `wipe` |
| `privpathex` | `privpath` |

They are checked before the glob lines of the same kind. The syntax is `directive <regular expression> <rule>`.
The default file uses them to refuse non ASCII file names:

```
makedirex     [^\p{ASCII}]   !*
uploadex      [^\p{ASCII}]   !*
renameex      [^\p{ASCII}]   !*
renameownex   [^\p{ASCII}]   !*
```

Other examples:

```
makedirex   ^/SECTION1/.*(sectionkeyword1|sectionkeyword2).*$   !*
uploadex    ^.*(sitekeyword1|sitekeyword2).*$                    !*
wipeex      ^/[^/]+$                                              !*
```

The last one stops anybody from wiping an item directly in the root directory.

## Other path directives

| Directive | Syntax | What it does |
| --- | --- | --- |
| `hideinwho` | `hideinwho <path> <rule>` | Connections of users matching the rule whose current directory matches the path are left out of `SITE WHO`, `SITE SWHO`, `SITE BW`, `SITE IDLERS`, `SITE LEECHERS`, `SITE UPLOADERS` and `SITE SPEED` |
| `msgpath` | `msgpath <path> <file> <rule>` | When a user enters a matching directory with `CWD`, the lines of `config/themes/text/<file>` are added to the reply. Every matching line is shown, not only the first. The file must exist when the config is loaded, otherwise an error is logged (`Unable to read ... directive ignored`) and the line does not work. See [Themes](Themes.md) |
| `pre` | `pre <path> <rule>` | Who may use `SITE PRE` on a release directory matching the path |
| `denydownload` | `denydownload <path> <rule>` | Matching users cannot download a file from the directory while it is still being uploaded |
| `deny_upfxp` | `deny_upfxp <path> <rule>` | Matching users cannot use FXP for uploads to the directory: the data connection is only accepted from their own IP address |
| `deny_dnfxp` | `deny_dnfxp <path> <rule>` | The same for downloads (FXP out of the site) |
| `dupecheck` | `dupecheck <path> <rule>` | Turns on the dupe check, for matching users, in the directory. See [Dupecheck](Dupecheck.md) |
| `nostatsup` | `nostatsup <path> <rule>` | Uploads to matching paths by matching users do not update the user's upload stats (files, bytes, time) |
| `nostatsdn` | `nostatsdn <path> <rule>` | Same for downloads |

Examples:

```
hideinwho    /hidden_path/*    *
msgpath      /incoming/        incoming.txt   *
pre          /pre/*            =siteop
denydownload /incoming/*       *
deny_upfxp   /PRIVATE/*        =group -user
deny_dnfxp   /PRIVATE/*        *
dupecheck    /incoming/*       *
nostatsup    /pre/*            *
nostatsdn    /pre/*            *
```

The path of `hideinwho`, `msgpath` and the FXP, `dupecheck` and `denydownload` directives is the directory the user is in
(their current directory), not the file.

## Credit and ratio directives

These are provided by the statistics plugin, see [Statistics](Statistics.md). The path is matched like the glob
directives. For each of them a line only counts when the path matches and the user passes the rule.

### `creditcheck`

```
creditcheck   <path>   <ratio>   <rule>
creditcheck   /ARCHIVE/*   0   *
creditcheck   /MP3/*       3   =vip
```

Upload credits. For a matching upload the user is credited `ratio` times the transferred bytes. `0`
means no credits are given. If no line matches, the user's own ratio (`SITE CHANGE <user> ratio`) is used.
Fractions such as `1.5` are accepted.

### `creditloss`

```
creditloss   <path>   <ratio>   <rule>
creditloss   /FREELEECH/*   0   *
```

Download cost. A matching download takes `ratio` times the transferred bytes from the user's credits.
`0` means free. With no matching line the cost is 1. Users with a ratio of 0 (leech) are never charged,
whatever the lines say.

### `creditlimit`

```
creditlimit   <path>   <UP|DN>   <ALL|DAY|WEEK|MONTH>   <bytes>   <rule>
creditlimit   /*   DN   DAY   50GB   *
```

Limits how much a user can transfer in a period. `UP` limits uploads, `DN` downloads. The periods are
`ALL` (total ever), `DAY`, `WEEK` or `MONTH`. `<bytes>` is a number with an optional unit: `K`, `M`, `G`, `T`, `P`, `E` are
decimal (1000 based), `Ki`, `Mi`, `Gi` and so on are binary (1024 based), and a trailing `B` is optional (for example `500MB`, `50GB`, `1TiB`). A transfer in that direction is refused once the user has already transferred more than the limit
in the period.

See also the credit and ratio part of [Users and Groups](Users-and-Groups.md).

## Quick reference

Directives understood by `perms.conf` in 4.0.x:

- Built in: `login_prompt`, `max_users`, `exempt`, `allow_connections`, `allow_connections_deny_reason`, `bouncer_ips`,
  `pasv_ports`, `pasv_addr`, `hide_ips`, `hideinstats`
- File system, glob: `upload`, `download`, `makedir`, `delete`, `deleteown`, `rename`, `renameown`, `wipe`, `privpath`
- File system, regex: `uploadex`, `downloadex`, `makedirex`, `deleteex`, `deleteownex`, `renameex`, `renameownex`, `wipeex`, `privpathex`
- Core: `hideinwho`, `msgpath`, `pre`, `denydownload`, `deny_upfxp`, `deny_dnfxp`, `userrejectsecure`, `userrejectinsecure`,
  `denydiruncrypted`, `denydatauncrypted`
- Plugins: `dupecheck` (dupecheck plugin), `nostatsup`, `nostatsdn`, `creditcheck`, `creditloss`, `creditlimit` (statistics plugin)

`hideinstats` takes a rule (for example `hideinstats =deleted =exempt`). Users matching it are left out of the
upload and download top lists (`SITE ALUP`, `SITE WKDN` and the like, see [Statistics](Statistics.md)).

Unknown directives are ignored and an error is written to the log.
