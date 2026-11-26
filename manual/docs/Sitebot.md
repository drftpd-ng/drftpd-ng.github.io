> Applies to DrFTPD 4.0.x.

# Sitebot

The sitebot is an IRC bot built into the master. It does two jobs:

- It **announces** what happens on the site (new directories, races, nukes, pre, requests and so on) in your IRC channels.
- It lets site users **run commands from IRC** (for example `!request`, `!imdb`, `!undupe`), with the same permission system as FTP.

It is a plugin of the master, there is nothing to install on the slaves. It is disabled by default.

Files, all in `config/` (copy the `.dist` file to the name without `.dist` before you change it, see [Master-Configuration](Master-Configuration.md)):

| File | Purpose |
|------|---------|
| `config/plugins/irc.conf` | The connection: servers, nick, channels, encryption, services, throttling. |
| `config/plugins/irc.announce.conf` | Which announce goes to which channel. |
| `config/plugins/dailystats.conf` | Settings of the daily, weekly and monthly top lists. |
| `config/plugins/sysop.conf` | Settings of the login and site command announces for staff. |
| `config/commands/irc/*.conf` | The IRC commands and who may use them (see [IRC-Commands](IRC-Commands.md)). |
| `config/themes/irc/*.theme` | The text of announces and replies (see [Themes](Themes.md)). |

## Quick start

1. Copy `irc.conf.dist` to `irc.conf`.
2. Set `activated=true`.
3. Set `server.1.host`, `server.1.port` and `server.1.use_ssl` for your IRC server.
4. Set `nick`, and at least one `channel.1` with its key and Blowfish key (or turn `blowfish.enable` off, see the warning below).
5. Copy `irc.announce.conf.dist` to `irc.announce.conf` if you want to change where announces go. The default sends everything to all channels the bot is in.
6. Restart the master. Later changes to these files are picked up with `SITE RELOAD` (the bot rejoins channels as needed and parts channels you removed). Changing `activated` or adding extra bots needs a restart.

Use `SITE IRC connect`, `SITE IRC disconnect` and `SITE IRC reconnect` (siteop only) to control the connection while the master runs.

> **Warning:** `blowfish.enable` is `true` by default. With Blowfish enabled the bot only reads **encrypted** lines from channels and from private messages. Plain text lines are ignored, so IRC commands from a client that does not encrypt do nothing. If you do not use an encrypting client (FiSH, Mircryption), set `blowfish.enable=false`.

## irc.conf reference

The values below are the defaults of `irc.conf.dist`.

### General

| Key | Default | Meaning |
|-----|---------|---------|
| `activated` | `false` | Turn the bot on. |
| `bot.name` | `DrFTPD-Bot` | Internal name of the bot. It is used in the stored idents and Blowfish keys, in `SITE IRC <botname> ...` and so on. It cannot contain `\|`. |
| `nick` | `drftpd` | The nick on the IRC server. |
| `name` | `drftpd` | The "real name" shown by `/whois`. |
| `user` | `drftpd` | The username part of the bot's hostmask (`user@host`). |
| `nick.auto` | `true` | If the nick is taken, try the nick with a number appended (`drftpd1`, `drftpd2`, ...). |
| `nick.usermodes` | not set | Extra user modes to set after connecting, for example `+h`. |
| `ctcp.version`, `ctcp.finger` | not set | Override the built-in reply to CTCP VERSION and FINGER. |
| `charset` | `UTF-8` | Character set used with the IRC server. |
| `command.trigger` | `!` | Lines (in a channel or private message) that start with this are treated as commands. |
| `bind.host` | not set | Local address to bind to on a machine with several IPs. |

### Servers, SSL and SASL

You can list several servers. The bot connects to the first one; if it fails, or it is disconnected, it tries the next, and goes back to the first after the last. Each server is numbered `N`, starting from 1, without gaps.

| Key | Default | Meaning |
|-----|---------|---------|
| `server.N.host` | `127.0.0.1` | Host name of the server. |
| `server.N.port` | `6697` | Port. |
| `server.N.password` | empty | Server password, if the server needs one. |
| `server.N.use_ssl` | `true` | Connect with SSL/TLS. |
| `connect.delay` | `30` | Seconds to wait before trying another server. |

SASL PLAIN login (IRCv3) is optional and set per server. When `server.N.sasl.username` and `server.N.sasl.password` are both set, the bot authenticates with the network's services before it joins any channel. Leave them empty to turn SASL off (for networks that do not support it).

| Key | Default | Meaning |
|-----|---------|---------|
| `server.N.sasl.username` | empty | Account name (for example your NickServ account). |
| `server.N.sasl.password` | empty | Account password. |
| `server.N.sasl.required` | `false` | If `true` and SASL fails, times out (30 seconds) or is not offered, the bot disconnects and tries the next server. If `false` a warning is logged and the bot connects without SASL. Repeated failures with `required=true` are retried with a growing delay (`connect.delay`, then twice as long each time, up to 1 hour). |
| `server.N.sasl.allow_plaintext` | `false` | SASL PLAIN sends the password only base64 encoded. By default SASL is refused on a server with `use_ssl=false`. Set this to `true` only if you accept sending the password in clear text. |
| `server.N.sasl.allow_insecure_tls` | `false` | When SASL is used over SSL, the server certificate is fully verified (trusted chain and host name) before the password is sent, so `server.N.host` must be the name on the certificate, not an IP address. Set this to `true` to skip the verification, for example for a private server with a self signed certificate. A warning is logged on every connect. |

Example for a network with SASL (taken from the dist file):

```
server.1.host=irc.libera.chat
server.1.port=6697
server.1.use_ssl=true
server.1.sasl.username=drftpd
server.1.sasl.password=yourNickServPassword
server.1.sasl.required=false
```

Certificate checking for servers **without** SASL is controlled by `ssl.strictTrust`:

| Key | Default | Meaning |
|-----|---------|---------|
| `ssl.strictTrust` | `false` | With `false` the server certificate is not checked at all (servers with SASL are always checked). With `true` the certificate chain must lead to a CA trusted by the Java runtime, or the certificate must be present as a PEM file in `config/plugins/irc/certs`. This allows a self signed certificate on a private server without giving up authenticity checks. This setting checks the chain only, not the host name. |
| `ssl.keystore.password` | not set | Only needed with `ssl.strictTrust=true` if you changed the password of the Java `cacerts` file. |

### NickServ and ChanServ

| Key | Default | Meaning |
|-----|---------|---------|
| `services.nickserv.enable` | `false` | Use NickServ. |
| `services.nickserv.register.new` | `false` | Register the nick the first time. Set it back to `false` once done. |
| `services.nickserv.register.joingroup` | `false` | Register the nick as grouped with an existing nick. |
| `services.nickserv.register.nick` | `drftpd` | The nick to register, or the existing nick to group with. |
| `services.nickserv.register.password` | `somepass` | The password for that nick (a placeholder in the dist file, change it). |
| `services.nickserv.register.email` | `drftpd@drftpd.org` | E-mail address used to register. |
| `services.nickserv.delayafter` | `0` | Milliseconds to wait after sending the NickServ command, before joining channels (0 = do not wait). |
| `services.chanserv.enable` | `false` | Use ChanServ. |
| `services.chanserv.invite.N` | `#channel` | Channels the bot asks ChanServ to invite it to. List as many as you need (`.1`, `.2`, ...). |

If your network supports SASL, using SASL is the simpler way to identify the bot.

### Extra commands after connecting

```
#connected.command.1=OPER drftpd somepass
```

`connected.command.N` lines are sent as raw IRC commands after the bot has connected, for example to become an IRC operator. They must follow the IRC protocol (not mIRC syntax).

### Channels

```
channel.1=#drftpd-test
channel.1.perms=*
channel.1.blowkey=drftpd
channel.1.blowmode=cbc
channel.1.chankey=drftpd
```

| Key | Default | Meaning |
|-----|---------|---------|
| `channel.N` | `#drftpd-test` | Channel name. Add `channel.2`, `channel.3`, ... for more channels. |
| `channel.N.perms` | `*` | Who may be invited to this channel and may see its Blowfish key with `SITE BLOWFISH`. Same syntax as in [Permissions](Permissions.md) (for example `=siteop` or `*`). |
| `channel.N.blowkey` | `drftpd` | The Blowfish key of the channel. |
| `channel.N.blowmode` | `cbc` | Blowfish mode of the channel (`cbc` or `ecb`). |
| `channel.N.chankey` | `drftpd` | The channel key (`+k`) used to join. |
| `channel.autojoin` | `true` | Join a channel when invited. For safety the bot only joins channels that are listed in `channel.N`. |

The channel keys in the dist file are examples. Choose your own.

### Blowfish encryption

| Key | Default | Meaning |
|-----|---------|---------|
| `blowfish.enable` | `true` | Encrypt channel and private messages with Blowfish, and only accept encrypted lines (see the warning above). |
| `blowfish.dh1080.enable` | `true` | Allow users' Blowfish keys to be set up through a DH1080 key exchange. Needs the unlimited strength crypto policy in your Java. |
| `blowfish.dh1080.cbc` | `true` | Allow DH1080 in CBC mode (more secure). |
| `blowfish.dh1080.ecb` | `false` | Allow DH1080 in ECB mode. |
| `blowfish.unencrypted.punish` | `false` | Act against users who send unencrypted text to a channel or to the bot while Blowfish is on. |
| `blowfish.unencrypted.action` | `kick` | `kick` or `ban` (a ban is followed by a kick), in all channels the user is in. Ban masks do not use wildcards, so they are easy to avoid; this is meant mostly as a warning. |
| `blowfish.unencrypted.reason` | `Unencrypted text is not permitted` | The kick reason. |

Private messages use a personal key per user, not the channel key. A user sets it with `SITE SETKEY` / `SITE SETBLOWFISH`, or, when DH1080 is enabled and the user is linked but has no key yet, the bot starts a DH1080 key exchange.

### Throttling and command handling

These keys limit how fast the bot sends and how many commands it handles. The first two are for advanced users.

| Key | Default | Meaning |
|-----|---------|---------|
| `message.sendDelay` | `300` | Milliseconds the bot waits after each message it sends. This is the main protection against being disconnected for flooding. |
| `line.length.max` | `512` | Maximum length of a raw line sent to the server, including the CR LF at the end. |
| `commands.max` | `5` | Maximum number of commands running at the same time. Must be greater than 1 (and preferably more) if you want aborting a command to work. |
| `commands.full` | `queue` | What to do when `commands.max` commands are running: `queue` (wait) or `block` (reject the command, the user is told all command threads are busy). |

### Several bots

| Key | Default | Meaning |
|-----|---------|---------|
| `bot.multiple.enable` | `false` | Start more than one bot, for example one per IRC network. The bots do not know about each other. |
| `bot.multiple.directories` | not set | Space separated list of directories with the configuration of the additional bots. They are subdirectories of `config/plugins/irc` and must contain a copy of every configuration file the sitebot and its plugins use. |

When more than one bot runs, the bot name (`bot.name`) is needed in commands such as `SITE IRC <botname> connect`. Give every bot its own `bot.name`.

## Linking site users to IRC nicks (ident)

IRC commands run with the permissions of a site user, so the bot has to know which site user is behind an IRC nick. This link is the **ident**: the full hostmask `nick!user@host` of the IRC user, stored in the user's account for each bot. It is compared literally, so if the nick, username or host changes the user has to link again.

A user can link in one of these ways:

- **`!ident <username> <password>`**, sent to the bot as a private message. If the password is right, the hostmask is stored. Only works in private.
- **`!invite <username> <password>`**, sent to the bot as a private message. It invites the user to all channels the user is permitted to enter (`channel.N.perms`) and stores the hostmask again. Its default permission is `*`, so it can only be used by a user who is already linked. A wrong password is announced as an invite failure. Only works in private.
- **`SITE INVITE <ircnick>`** (or `SITE INVITE <botname> <ircnick>` if several bots run), from FTP. The bot looks up the nick (WHOIS), stores its hostmask as the ident and invites the nick to the permitted channels. This is the easy way to onboard users: log in on FTP, give your IRC nick.
- **`!unident`** or `SITE UNIDENT` removes the link.

The FTP command file also lists `SITE IDENT`, but the code behind it expects an IRC session; use the IRC command `!ident`.

When Blowfish is on, the bot ignores private messages it cannot decrypt, so a new user needs a personal key before `!ident` can be sent. Use `SITE INVITE <ircnick>` or `SITE SETKEY <blowkey>` from FTP, and `SITE BLOWFISH` to see the channel keys.

A user that is not linked is treated as not logged in. Commands whose permission is `*` (logged in users) are refused for them. Only commands with `%` (not logged in) work, which is why `!ident` can be used before linking.

### Sitebot commands

| Command (FTP / IRC) | Default permission | Description |
|---------------------|--------------------|-------------|
| `SITE IRC` / `!irc` | `=siteop` | Control the bot: `connect`, `disconnect`, `reconnect`, `raw <command>`, `say <message>`, `emptyqueue`. With more than one bot, give the bot name first: `SITE IRC <botname> connect`. |
| `SITE BLOWFISH`, `SITE GETKEY` / `!blowfish`, `!getkey` | `*` | List the Blowfish keys of the channels you may enter. |
| `SITE SETKEY`, `SITE SETBLOWFISH` / `!setkey`, `!setblowfish` | `*` | Set your personal Blowfish key for private messages: `<blowkey>` (or `<botname> <blowkey>`). |
| `SITE INVITE <ircnick>` | `*` | Invite yourself to the channels using the bot, and link your nick (see above). |
| `!invite <username> <password>` | `*` | IRC only: invite, as described above (needs an already linked nick). |
| `!ident <username> <password>` | `%` | IRC only: link your nick to a site user. |
| `SITE UNIDENT` / `!unident` | `*` | Remove the link. |

In the IRC command files, each command also has `input` (where the command is accepted: `all`, `public`, `private` or a channel name) and `output` (where the reply goes: `private`, `public`, `source` or channel names). The sitebot commands use `input private` and `output source`, so they only work in a private message with the bot.

## IRC commands and permissions

All IRC commands are defined in `config/commands/irc/*.conf` (or the `.dist` file if there is no `.conf`). Each entry has the same `perms` line as the FTP commands and the `input` and `output` settings above. IRC commands run as the linked site user, with that user's permissions. The list of commands is on the [IRC-Commands](IRC-Commands.md) page, the permission syntax on [Permissions](Permissions.md).

If you have changed a command file, remember that your `.conf` replaces the `.dist` file completely, so new commands of a later version are not added automatically.

## Announces

### How announces are routed

Every announce has a **type** (for example `mkdir` or `store.complete`). `irc.announce.conf` tells the bot where to send each type. There are two kinds of types:

- **Types with a path** (they are about a directory or file): `mkdir`, `store.first`, `nuke` and so on. They can be routed by path or by section.
- **Types without a path** (they are not about a directory): `invite`, `request`, `sysop` and so on. They have a single destination.

Routing for types with a path, for a type called `type`:

```
type.path.1=/path/subpath/*
type.path.1.destination=#channel1 #channel2
type.path.1.displayname=PSEUDOSECTION
type.path.1.regex=false

type.section.1=SECTION_NAME
type.section.1.destination=#channel1 #channel2
```

- `destination` is a space separated list of channels. The special value `public` means every channel the bot is in. The special value `null` means: do not announce (this lets you silence one path or section while still using a catch-all `default`).
- `path.N` rules work like path permissions in [Permissions](Permissions.md). `regex=true` switches to regular expressions (case insensitive; use double backslashes). In both cases the whole path must match (a directory path ends with `/`). Use `displayname` to make the bot show the announce as if it came from a section with that name, with the path shortened to what is below the matched path. With regular expressions, a named group `truncate` removes that part of the path from the announce, and a named group `announce` keeps only that part.
- `section.N` rules use the names of your [Sections](Sections.md). A rule with the section name `*` matches every section.
- Numbered rules are tried in order, starting at 1, up to the first number that is missing.

Order of evaluation for an event: the path rules of its type, then the section rules of its type, then the path rules of the `default` type, then the section rules of the `default` type. If nothing matches, the event is not announced. Make sure your `default` rules catch everything.

Types without a path:

```
type.destination=#channel1 #channel2
default.destination=#channel1 #channel2
```

`default.destination` is used for any type that has no destination of its own.

The dist file ships with:

```
default.section.1=*
default.section.1.destination=public
default.destination=public
```

which announces everything everywhere. Example of the dist file for `mkdir`:

```
mkdir.path.1=/incoming/*
mkdir.path.1.destination=#newstuff
mkdir.path.2=/hiddendir/*
mkdir.path.2.destination=null
mkdir.section.1=MP3
mkdir.section.1.destination=public

default.path.1=*
default.path.1.destination=#mainchannel
default.path.1.displayname=DEFAULT
default.path.1.regex=false
```

Dated sections: to leave the dated directory out of the announced path, route with a path rule and a `displayname`, for example:

```
default.path.1=/SECTION1/****/*
default.path.1.destination=#irc-channel
default.path.1.displayname=SECTION1
default.path.1.regex=false
```

(`****` matches a `MMdd` directory name, `****-**` matches `yyyy-ww`.)

#### The `store` type

The `store` type announces the upload of single files, by file name. It is set up by file name patterns, which are the `store.path.N` rules themselves (a regular expression that is matched against the lower case file name). The dist file announces `.nfo` uploads:

```
store.path.1=^.+\\.nfo$
store.path.1.destination=public
store.path.1.regex=true
```

### All announce types

The type name is what you use in `irc.announce.conf`. "Path" means the type can be routed with `path.N` and `section.N` rules; "single" means only `type.destination` is used.

| Type | Routing | Produced by | Theme file (`config/themes/irc/`) |
|------|---------|-------------|-----------------------------------|
| `mkdir`, `rmdir`, `wipe` | path | Making, removing and wiping a directory | `core.theme` |
| `addslave`, `delslave`, `msgslave` | single | A slave coming online, going offline, a slave message | `core.theme` |
| `msgmaster` | single | A master message | `core.theme` |
| `invite` | single | `SITE INVITE` / `!invite` success or failure | `core.theme` (`invite.success`, `invite.failed`) |
| `nuke`, `unnuke` | path | Nuking and unnuking | `nuke.announcer.theme` |
| `pre` | path | `SITE PRE` | `pre.announcer.theme` |
| `store` | path (see above) | Upload of a file that matches a `store.path.N` pattern | `store.announcer.theme` |
| `dailystats.dayup`, `dailystats.daydn`, `dailystats.wkup`, `dailystats.wkdn`, `dailystats.monthup`, `dailystats.monthdn` | single | The top uploaders and downloaders of the day, week and month, when the stats reset | `dailystats.announcer.theme` |
| `sysop` | single | Logins and site commands of staff (see below) | `sysop.announcer.theme` |
| `store.first`, `store.race`, `store.halfway`, `store.complete` | path | Races, see [Zipscript](Zipscript.md) | `sfv.announcer.theme`, `zip.announcer.theme` |
| `mp3`, `flac` | path | First MP3 or FLAC of a release, see [Zipscript](Zipscript.md) | `mp3.announcer.theme`, `flac.announcer.theme` |
| `store.newraceleader` | path | A new race leader, see [Race-Leader](Race-Leader.md) | `new.raceleader.announcer.theme` |
| `mediainfo` | path | Video file info, see [Mediainfo](Mediainfo.md) | `mediainfo.announcer.theme` |
| `imdb` | path | IMDB info for an uploaded NFO, see [IMDB-and-TVMaze](IMDB-and-TVMaze.md) | `imdb.announcer.theme` |
| `tvmaze` | path | TVmaze info for a new release directory or pre, see [IMDB-and-TVMaze](IMDB-and-TVMaze.md) | `tvmaze.announcer.theme` |
| `request`, `reqfilled`, `reqdel` | single | Requests, see [Requests](Requests.md) | `request.announcer.theme` |
| `archive_start`, `archive_finish`, `archive_failed` | path | Archiving, see [Archive](Archive.md) | `archive.theme` |
| `autonuke` | path | See [Autonuke](Autonuke.md) | `autonuke.theme` |
| `autofreespace` | single | See [Autofreespace](Autofreespace.md) | `autofreespace.theme` |
| `nukefilter` | path | See [Nukefilter](Nukefilter.md) | `nuke.filter.announcer.theme` |
| `prebw` | path | See [PreBW](PreBW.md) | `pre.bw.announcer.theme` |
| `speedtest` | single | See [Speedtest](Speedtest.md) | `speedtest.announcer.theme` |
| `trafficmanager.def`, `trafficmanager.ban`, `trafficmanager.kick` | single | See [Traffic](Traffic.md) | `traffic.default.announcer.theme`, `traffic.ban.announcer.theme`, `traffic.kick.announcer.theme` |
| `trialmanager.grouptop`, `trialmanager.toptrial` | single | See [Trial](Trial.md) | `grouptop.announcer.theme`, `toptrial.announcer.theme` |

The announce `pre` is also registered by the SFV and zip announcers of the zipscript; the text that is actually sent for a pre comes from `pre.announcer.theme`.

Any type that is not mentioned in your `irc.announce.conf` is sent to the `default` destination. To turn one off, give it the destination `null`:

```
dailystats.dayup.destination=null
```

### Daily stats settings

`config/plugins/dailystats.conf`:

| Key | Default | Meaning |
|-----|---------|---------|
| `showzero` | `true` | Show users with 0 bytes. |
| `outputnum` | `5` | Number of users in each list. |
| `exempt` | empty | Groups to leave out (space separated). The value is converted to lower case, so only lower case group names work here. Deleted users are never listed. |

### Sysop announces

The `sysop` announce reports login steps and staff site commands. `config/plugins/sysop.conf` sets, for each action, what is shown:

- `0`: show successful and failed
- `1`: show nothing
- `2`: show only successful
- `3`: show only failed

All actions default to `0`. Login steps: `PASS`, `IDNT`, `USER`. Site commands: `ADDIP`, `ADDUSER`, `BAN`, `CHANGE`, `CHGRP`, `CHPASS`, `DELIP`, `DELUSER`, `GADDUSER`, `GIVE`, `GRPREN`, `KICK`, `PASSWD`, `PURGE`, `READD`, `RENUSER`, `TAGLINE`, `TAKE`, `UNBAN`. Send the `sysop` type to a private staff channel, since it reports account activity:

```
sysop.destination=#staff
```

## Related pages

- [Themes](Themes.md) for the text and colors of announces
- [Zipscript](Zipscript.md), [Requests](Requests.md), [IMDB-and-TVMaze](IMDB-and-TVMaze.md), [Mediainfo](Mediainfo.md)
- [IRC-Commands](IRC-Commands.md), [Permissions](Permissions.md), [Sections](Sections.md), [Log-Levels](Log-Levels.md)
