> Applies to DrFTPD 4.0.x.

# Themes

Themes control the text DrFTPD sends: replies to FTP users (directory listings, `SITE` command output, race stats, status bars) and what the sitebot says on IRC (announces and replies to IRC commands). The text is not hard coded: it lives in plain theme files that you can edit.

## Two sets of themes

| Set | Directory | Used for |
|-----|-----------|----------|
| FTP themes | `config/themes/ftp/` | Replies to commands run over FTP, listings, banners and the information boxes of plugins (race stats, MP3 info, status bars). |
| IRC themes | `config/themes/irc/` | Announces on IRC and the replies to IRC commands (see [Sitebot](Sitebot.md)). |

A third directory, `config/themes/text/`, holds plain text files such as `welcome.txt`, `rules.txt` and the `*_header.txt` / `*_footer.txt` files of the stats listings. These use the same rule: if `name.txt` exists it is used, otherwise `name.txt.dist` is used.

The FTP theme and the IRC theme of a command usually have the same file name and the same keys, but the IRC version has IRC colors and formatting. For example `request.theme` exists in both directories.

## `.theme.default` and `.theme`

Every theme file ships as `<name>.theme.default`. **Do not edit the `.default` file**, it is replaced when you update DrFTPD. To change a theme:

1. Copy `<name>.theme.default` to `<name>.theme` in the same directory.
2. Edit the copy. You only need to keep the keys you want to change: any key that is missing from `<name>.theme` falls back to the value in `<name>.theme.default`.
3. Run `SITE RELOAD` (siteop) to load the change. FTP themes and IRC themes are both reloaded.

Example: to change only the announce for a new directory, create `config/themes/irc/core.theme` containing just:

```
mkdir=NEW in ${section}: ${path} by ${user}/${group}
```

All other keys of `core.theme.default` keep their default text.

Rules to know:

- A key name must be unique across all theme files of one directory. If two files define the same key, an error `Theme key collision` is logged. Do not copy keys from one theme file into another.
- If the code asks for a key that does not exist in any file, the text `No theme available` is shown and an error is logged.
- When you upgrade, new keys appear in the `.default` files. Because your `.theme` file falls back to the default, new keys work immediately, but compare your file with the new default if you want to restyle them.

## File format

Theme files are Java properties files:

```
key=text
```

- One `key=value` per line. Lines starting with `#` are comments.
- `\n` in a value is a line break in the output.
- A `\` at the end of a line continues the value on the next line (used for the multi line boxes).
- Special characters such as box drawing characters can be written as `\uXXXX` escapes. Some default files, such as the MP3 info box, contain them.

## Variables

A value can contain `${name}`, which is replaced by the data of the event. The names that exist depend on the key; the comment lines above a key in the default files list them ("Tags:"), and the plugin pages list them too.

An excerpt of `cwd.racers.body` (race stats box, shortened):

```
#${position,3.3} ${user,-13} @ ${group,-11} ${files,7}F
```

Padding and truncating, as used in the boxes:

| Form | Meaning |
|------|---------|
| `${name}` | The value as is. |
| `${name,10}` | Right aligned in a field of 10 characters. |
| `${name,-10}` | Left aligned in a field of 10 characters. |
| `${name,-22.22}` | Left aligned in 22 characters and cut off after 22 characters. |
| `${position,3.3}` | Right aligned in 3 characters and cut after 3. |

A variable that does not exist for that key is replaced by `[Unknown]`.

### Colors and formatting on IRC

IRC themes can use these four names, which the sitebot adds to every announce:

| Name | Meaning |
|------|---------|
| `${bold}` | Toggles bold. |
| `${underline}` | Toggles underline. |
| `${color}` | Starts a color. Follow it with a color number: `${color}04` is red. |
| `${coloroff}` | Turns off color and bold. |

Colors are the standard mIRC numbers (for example `04` red, `08` yellow, `14` dark grey, `15` light grey). Because `${color}` is only the color escape, the number comes right after it. `${color}04,08` sets a foreground and background.

Many announces also have `${sectioncolor}`, which is the `color` set for the section in [Sections](Sections.md), so the section name shows in the section's color:

```
mkdir=${color}15${bold}NEW${bold}${coloroff} in ${color}${sectioncolor}${bold}${section}${bold}${coloroff}: -> ${bold}${path}${bold} by ${bold}${user}${bold}${color}14${bold}/${bold}${coloroff}${group}.
```

This is the default `mkdir` announce from `config/themes/irc/core.theme.default`. Read it piece by piece: a light grey bold `NEW`, then ` in `, then the section name in the section color, then the path in bold, and finally `user/group` with a dark grey slash.

More default examples:

```
pre=${color}15${bold}PRE${bold}${coloroff} in ${color}${sectioncolor}${bold}${section}${bold}${coloroff}: -> ${bold}${grpname}${bold} gives us ${bold}${name}${bold} (${bold}${files} files${bold} - ${bold}${bytes}${bold}).
```

(`config/themes/irc/pre.announcer.theme.default`)

```
store=${color}15${bold}${extUpperCase}${bold}${coloroff} in ${color}${sectioncolor}${bold}${section}${bold}${coloroff}: -> ${path}/${bold}${file}${bold} sent by ${bold}${user}${bold}${color}14${bold}/${bold}${coloroff}${group} @ ${bold}${speed}${bold}.
```

(`config/themes/irc/store.announcer.theme.default`)

An FTP theme example, the progress entry of a directory listing (`config/themes/ftp/zipscript.list.theme.default`):

```
zip.statusbar.format=[${statusbar}]
zip.statusbar.separator=|
zip.statusbar.complete=${complete.number} of ${complete.total} files = ${complete.percent}% complete at ${complete.totalbytes}
zip.statusbar.offline=${offline.number} files offline = ${online.percent}% online
zip.files.missing.filename=${mfilename}-MISSING
```

And one that uses padding, the `SITE NEW` listing (`config/themes/ftp/new.theme.default`):

```
new=[${pos,2}] ${age,7} ${diruser,-12} ${files,3}F ${size,7} ${name,-40}
```

## Theme files

Directory and file names below are without `.default`.

### FTP themes (`config/themes/ftp/`)

| File | Covers |
|------|--------|
| `core.theme` | The status line (uploaded, downloaded, speed, credits, ratio). |
| `login.theme` | Login replies (user, password). |
| `dataconnection.theme` | Transfer messages (complete, max simultaneous transfers). |
| `list.theme` | The `-OFFLINE` entries in listings and the text of the FTP `STAT` reply. |
| `misc.theme` | Header and footer of `SITE HELP`. |
| `new.theme` | The listing of new directories. |
| `sections.theme` | The listing of sections. |
| `serverstatus.theme` | Server status output (memory, classes, garbage collection and so on). |
| `slavemanagement.theme` | Replies of the slave commands (add slave, delete slave, ...). |
| `usermanagement.handler.theme` | Replies of the user and group commands. |
| `expired.user.post.hook.theme` | The expiry text added to user info. |
| `notes.post.hook.theme` | User notes text. |
| `approve.theme` | `SITE APPROVE` replies. |
| `nuke.theme`, `nuke.list.theme` | Replies of the nuke and unnuke commands, and the reason shown in listings of nuked directories. |
| `grouptop.theme` | Group top and group quota replies. |
| `archive.theme` | Archive replies ([Archive](Archive.md)). |
| `autonuke.theme` | AutoNuke replies ([Autonuke](Autonuke.md)). |
| `find.theme`, `search.theme` | Find and search output ([Find](Find.md), [Index-and-Search](Index-and-Search.md)). |
| `jobs.theme` | Job queue replies ([Jobs](Jobs.md)). |
| `statistics.theme` | The stats listings (all time, month, week and day top lists, [Statistics](Statistics.md)). |
| `toptrial.theme` | Trial rankings ([Trial](Trial.md)). |
| `speedtest.theme` | Output of the speedtest.net commands ([Speedtest](Speedtest.md)). |
| `request.theme` | Request replies ([Requests](Requests.md)). |
| `imdb.theme`, `imdb.list.theme` | IMDB replies and the IMDB bar ([IMDB-and-TVMaze](IMDB-and-TVMaze.md)). |
| `tvmaze.theme`, `tvmaze.list.theme` | TVmaze replies and the TVmaze bar. |
| `mediainfo.list.theme` | The media bar ([Mediainfo](Mediainfo.md)). |
| `zipscript.list.theme`, `zipscript.diz.theme` | Status bar and `-MISSING` entries ([Zipscript](Zipscript.md)). |
| `zipscript.post.hook.theme` | Race stats box. |
| `zipscript.mp3.post.hook.theme`, `zipscript.mp3.statusbar.theme` | MP3 info box and the genre/year part of the status bar. |
| `zipscript.flac.post.hook.theme`, `zipscript.flac.statusbar.theme` | FLAC info box and the genre/year part of the status bar. |

### IRC themes (`config/themes/irc/`)

| File | Covers |
|------|--------|
| `core.theme` | Announces `mkdir`, `rmdir`, `wipe`, `addslave`, `delslave`, `msgslave`, `msgmaster`, `invite.success`, `invite.failed`. |
| `nuke.announcer.theme` | Announces `nuke`, `nuke.nukees`, `unnuke`, `unnuke.nukees`. |
| `pre.announcer.theme` | Announce `pre`. |
| `store.announcer.theme` | Announce `store` (single file uploads, for example NFO). |
| `dailystats.announcer.theme` | The daily, weekly and monthly top lists. |
| `sysop.announcer.theme` | Login and site command announces for staff. |
| `sfv.announcer.theme`, `zip.announcer.theme` | Race announces ([Zipscript](Zipscript.md)). |
| `mp3.announcer.theme`, `flac.announcer.theme` | Tag info announces. |
| `new.raceleader.announcer.theme` | New race leader ([Race-Leader](Race-Leader.md)). |
| `mediainfo.announcer.theme` | Video info ([Mediainfo](Mediainfo.md)). |
| `imdb.announcer.theme`, `tvmaze.announcer.theme` | Info announces ([IMDB-and-TVMaze](IMDB-and-TVMaze.md)). |
| `request.announcer.theme` | Request, filled, deleted ([Requests](Requests.md)). |
| `archive.theme` | Archive announces. |
| `autonuke.theme` | AutoNuke announce. |
| `autofreespace.theme` | AutoFreeSpace announce. |
| `nuke.filter.announcer.theme` | NukeFilter announce. |
| `pre.bw.announcer.theme` | PreBW announce. |
| `speedtest.announcer.theme` | Speedtest announce. |
| `traffic.default.announcer.theme`, `traffic.ban.announcer.theme`, `traffic.kick.announcer.theme` | Traffic manager announces. |
| `grouptop.announcer.theme`, `toptrial.announcer.theme` | Trial manager announces. |
| Other files with the same name as an FTP theme: `approve`, `expired.user.post.hook`, `grouptop`, `misc`, `new`, `notes.post.hook`, `nuke`, `sections`, `serverstatus`, `slavemanagement`, `usermanagement.handler`, `find`, `search`, `jobs`, `statistics`, `request`, `imdb`, `tvmaze`, `speedtest`, `toptrial` (each as `.theme`) | Replies to the IRC versions of those commands. They mirror the FTP theme of the same name, with IRC colors. |

(The IRC announces use the names listed in the table in [Sitebot](Sitebot.md).)

Remember the IRC announce text is only sent where `irc.announce.conf` routes it.

## Tips

- Keep announce lines short: the bot waits `message.sendDelay` between lines, and long lines are cut at `line.length.max` (see [Sitebot](Sitebot.md)).
- When restyling a box with box drawing characters, use a font and encoding your clients support. If the boxes look broken, try plain `+`, `-` and `|`.
- After editing, test on a quiet directory: enter it, upload a small file, and check the result before you go live.
