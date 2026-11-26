> Applies to DrFTPD 4.0.x.

# Traffic (minimum and maximum transfer speed)

The `traffic` plugin sets a **minimum** and/or **maximum** transfer speed for users and paths, and decides what happens to a user whose transfer is too slow: nothing but an announce, a kick, or a temporary ban.

- The **maximum** speed is a limit: the slave throttles the transfer to it.
- The **minimum** speed is a threshold: if the transfer is slower, the slave aborts it, and the master then runs the action of the matching type.

The config file is `config/plugins/traffic.conf` on the master (copy it from `traffic.conf.dist`). The plugin is part of the normal build; see [Plugins](Plugins.md) for how to enable or disable plugins. The config is reloaded with `SITE RELOAD`.

## Config format

The file is a list of numbered blocks, starting from `1`. The master reads `1.type`, `2.type`, ... and stops at the first number that has no `type` line, so keep the numbers consecutive.

```
1.type=default
1.name=Example
1.minspeed=0
1.maxspeed=0
1.include=*
1.exclude=
1.perms=*
1.up=true
1.dn=true
```

### Keys common to all types

| Key | Required | Meaning |
|-----|----------|---------|
| `x.type` | yes | `default`, `kick` or `ban` |
| `x.name` | yes | name of this block, used in the log. A block without a name is skipped with an error. |
| `x.minspeed` | no, default `0` | minimum speed in **KB/s** (1 KB = 1000 bytes). `0` disables the minimum. |
| `x.maxspeed` | no, default `0` | maximum speed in **KB/s**. `0` disables the limit. |
| `x.include` | no | regular expression for the directories this block applies to. `*` means every directory. The directory is the **directory the file is in** (the current directory of the user for the speed settings). The regex must match the whole path. |
| `x.exclude` | no | regular expression for directories this block must **not** apply to. `.*` and `*` are refused. Empty means no exclusion. |
| `x.perms` | no | who the block applies to, in the usual permission syntax, for example `*`, `=siteop`, `!-nobody`, see [Permissions](Permissions.md). |
| `x.up` | no, default `false` | use this block for uploads (`true` / `false`) |
| `x.dn` | no, default `false` | use this block for downloads (`true` / `false`) |

Note that `include` is empty if you leave it out, which matches nothing except an empty path. Always write `x.include=*` when you want a block to cover everything.

A block with a bad number, an invalid regex or a missing name is skipped (an error is logged) and the others still load.

### First match wins

The blocks are checked in order, and the **first block** that fits the transfer (direction enabled with `up`/`dn`, path matches `include`, does not match `exclude`, user matches `perms`) sets the limits and runs its action. Later blocks are ignored for that transfer. Put the specific blocks first and a catch-all last.

### Type `default`

Only sets the limits and announces when the minimum speed was not met. No keys of its own.

### Type `kick`

Kicks the user's connection when the transfer is too slow (the transfer is aborted and the control connection is closed).

| Key | Default | Meaning |
|-----|---------|---------|
| `x.reason` | `Slow Kick` | text sent to the user (as an FTP 426 reply) |

### Type `ban`

Bans the user for a while when the transfer is too slow.

| Key | Default | Meaning |
|-----|---------|---------|
| `x.reason` | `Trasnfering Too Slow` (sic) | ban reason, sent to the user and stored as the ban reason of the user. Set your own to avoid the typo. |
| `x.bantime` | `300` | ban length in seconds |
| `x.kickall` | `true` | `true`: close every connection of the user; `false`: only the connection that was too slow |

The ban is stored on the user (ban time and reason), the same data a siteop sees on the user.

## Example

```
# siteops are never limited (first match wins)
1.type=default
1.name=Siteops
1.perms==siteop
1.include=*
1.up=true
1.dn=true

# Uploads in the 0DAY section: at least 200 KB/s, else ban for 10 minutes
2.type=ban
2.name=Slow0day
2.minspeed=200
2.include=/0DAY/.*
2.perms=*
2.up=true
2.reason=Too slow, come back in ten minutes
2.bantime=600

# Everyone else: downloads limited to 5000 KB/s, no minimum
3.type=default
3.name=DownCap
3.maxspeed=5000
3.include=*
3.perms=*
3.dn=true
```

Block 1 has no min or max speed, so users in group `siteop` are not limited and not checked against the later blocks.

## Announces

If the [Sitebot](Sitebot.md) plugin is running, a slow transfer produces an announce when the transfer was actually aborted for being too slow. There is one event type per action:

| Type | Announce event | Theme keys |
|------|----------------|------------|
| `default` | `trafficmanager.def` | `traffic.default.up`, `traffic.default.dn` |
| `kick` | `trafficmanager.kick` | `traffic.kick.up`, `traffic.kick.dn` |
| `ban` | `trafficmanager.ban` | `traffic.ban.up`, `traffic.ban.dn` |

The theme files are `traffic.default.announcer.theme.default`, `traffic.kick.announcer.theme.default` and `traffic.ban.announcer.theme.default`. Variables in the themes: `${nickname}`, `${file}`, `${path}`, `${slave}`, `${transfered}`, `${minspeed}`, `${speed}`. See [Themes](Themes.md) for how theme files are overridden.

To send these announces to a particular channel, use the lines from `irc.announce.conf`:

```
#trafficmanager.def.destination=public
#trafficmanager.ban.destination=public
#trafficmanager.kick.destination=public
```

The ban announce has an extra `bantime` available in its event; the shipped theme does not show it.

## Things to know

- Speeds are in KB/s in the config (value times 1000 bytes per second), and announced in the same way as the rest of the site.
- The minimum speed check on the slave starts after about half a second and then every 0.2 seconds, so a transfer is only aborted once it has really been slow.
- The plugin only acts on file transfers (STOR and RETR). It does not limit the control connection.
