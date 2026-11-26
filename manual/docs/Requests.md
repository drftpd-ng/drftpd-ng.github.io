> Applies to DrFTPD 4.0.x.

# Requests

The request plugin gives users a way to ask for releases and to mark a request as filled. Requests are kept as directories in a request directory (by default `/REQUESTS`). The directory name shows who asked for what, and filling a request renames the directory.

Example of what you see in the request directory:

```
REQUEST-by.alice-Some.Release.Name-GRP
FILLED-for.bob-Another.Release.Name-GRP
```

## How it works

- `SITE REQUEST <name>` creates the directory `REQUEST-by.<user>-<name>` in the request directory and records the request (who, what, when). It counts against the weekly allowance of the user.
- Somebody uploads the release into that directory.
- `SITE REQFILLED <name>` renames the directory to `FILLED-for.<requester>-<name>`. It is announced on IRC, and the filler is counted in the user's request statistics.
- `SITE REQDEL <name>` deletes the request and its directory. Users can delete their own requests. Deleting the requests of others needs the `request.deleteOthers` permission (siteop by default).
- `SITE REQUESTS` lists the open requests.

The request name is one word (no spaces): `SITE REQUEST Some.Release.Name-GRP`. A name is refused if it is not a legal file name or if it matches one of the `request.deny.N` patterns. A request name that already exists is refused.

When a user is deleted (`SITE DELUSER`), the requests of that user are deleted with the directories. When a user is renamed (`SITE RENUSER`), the request directories of that user are renamed to match.

## Configuration

`config/plugins/request.conf` (copy `request.conf.dist` to `request.conf`). The values are the defaults of the dist file. Changes are read on `SITE RELOAD`.

| Key | Default | Meaning |
|-----|---------|---------|
| `request.dirpath` | `/REQUESTS` | The request directory. |
| `request.createpath` | `true` | Create `request.dirpath` when the master starts, if it does not exist. |
| `request.weekmax` | `2` | Maximum number of requests per user and week. `0` means unlimited. A user who has reached it gets "Access denied - You have reached max(N) number of requests per week". The count is reset to zero for all users at the weekly reset. |
| `request.weekexempt` | empty | Users who are not limited by `request.weekmax`. Permission syntax as in [Permissions](Permissions.md), for example `=siteop`. |
| `request.weekdecrease` | `false` | Give the allowance back when a request is deleted with `SITE REQDEL`. |
| `request.deny.1` | `[^\\p{ASCII}]` | A regular expression. A request name is refused if the expression is found in it. The default refuses any non-ASCII character. Add `request.deny.2`, `request.deny.3`, ... for more patterns (numbered without gaps). |
| `reqfilled.prefix` | `FILLED-for.` | Prefix of the directory name of a filled request. |
| `request.prefix` | `REQUEST-by.` | Prefix of the directory name of an open request. |
| `request.dateformat` | `yyyy-MM-dd @ HH:mm` | Date format used in the `SITE REQUESTS` listing. |

### Per command settings

The command files also carry settings (see [Site-Commands](Site-Commands.md)):

- `request.deleteOthers` on `SITE REQUEST` and `SITE REQDEL` is the permission that allows deleting other users' requests. The default is `=siteop`.
- A `request.dirpath` line inside a command definition overrides the request directory for that command, which allows several request directories (for example a separate directory for a second set of commands).

## Commands

| Command (FTP / IRC) | Default permission | Description |
|---------------------|--------------------|-------------|
| `SITE REQUEST <nameofrequest>` / `!request` | `=request =siteop` | Add a request. Needs the `request` group or siteop. |
| `SITE REQFILLED <nameofrequest>` / `!reqfilled` | `*` | Mark a request as filled. |
| `SITE REQDEL <nameofrequest>` / `!reqdel` | `=request =siteop` | Delete a request. |
| `SITE REQUESTS` / `!requests` | `*` | List the open requests. |
| `SITE FIXREQUESTS` / `!fixrequests` | `=siteop` | Repair the request directory so that it matches the stored requests: missing directories are created again, requests of users that no longer exist are removed, and unexpected empty directories are deleted. Unexpected directories that are not empty are only reported. |

The default permissions mean that a user has to be in a group named `request` to ask for requests. Create that group and add users, or change the `perms` line (for example to `*`). See [Users-and-Groups](Users-and-Groups.md) and [Site-Commands](Site-Commands.md). On IRC the commands are defined in `config/commands/irc/request.conf` / `request.conf.dist` (see [IRC-Commands](IRC-Commands.md)); they accept input from anywhere and answer where they were asked.

Example `SITE REQUESTS` output with the default theme:

```
REQUESTS:
[1] Some.Release.Name-GRP by alice on 2026-01-15 @ 10:30
Use SITE REQFILLED <name> after you fill a request.
```

## IRC announces

Announce types: `request`, `reqfilled`, `reqdel`. They have no path, so each has a single destination in `irc.announce.conf`, with a fallback to `default.destination` (see [Sitebot](Sitebot.md)):

```
request.destination=public
reqfilled.destination=public
reqdel.destination=public
```

The text is in `config/themes/irc/request.announcer.theme`:

```
request=${color}15${bold}REQUEST${bold}${coloroff}: ${bold}${requestname}${bold} by ${bold}${owner}${bold}${color}14${bold}/${bold}${coloroff}${ownerGroup}.
reqfilled=${color}15${bold}FILLED${bold}${coloroff}: ${bold}${issuer}${bold}${color}14${bold}/${bold}${coloroff}${issuerGroup} filled ${bold}${requestname}${bold} created by ${bold}${owner}${bold}${color}14${bold}/${bold}${coloroff}${ownerGroup}.
reqdel=${color}15${bold}REQDEL${bold}${coloroff}: ${bold}${issuer}${bold}${color}14${bold}/${bold}${coloroff}${issuerGroup} deleted ${bold}${requestname}${bold} created by ${bold}${owner}${bold}${color}14${bold}/${bold}${coloroff}${ownerGroup}.
```

Tags: `requestname`, `requestroot` (path of the request directory), `owner`, `ownerGroup` (who made the request), `issuer`, `issuerGroup` (who ran the command).

## Themes

| File | Covers |
|------|--------|
| `config/themes/ftp/request.theme` | Replies of the request commands over FTP. |
| `config/themes/irc/request.theme` | Replies of the request commands on IRC. |
| `config/themes/irc/request.announcer.theme` | The announces above. |

Keys in `request.theme` include `request.success`, `request.exists`, `request.error`, `reqfilled.success`, `reqfilled.error`, `reqfilled.exists`, `reqfilled.notfound`, `reqfilled.root.notfound`, `requests.header`, `requests.list`, `requests.footer`, `reqdel.success`, `reqdel.notfound`, `reqdel.notowner`, `reqdel.root.notfound` and `reqdel.error`. The tags are `request.name`, `request.root`, `request.user`, `request.date`, `num` and `user`. See [Themes](Themes.md) for how to change them.

## Related pages

- [Sitebot](Sitebot.md), [Permissions](Permissions.md), [Users-and-Groups](Users-and-Groups.md), [Themes](Themes.md)
- The default `sfvfirst.pathignore` of the [Zipscript](Zipscript.md) already skips `/REQUESTS/*`.
