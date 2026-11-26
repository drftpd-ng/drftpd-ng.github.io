> Applies to DrFTPD 4.0.x.

# Autonuke and the nuke commands

This page covers two things:

1. the manual **nuke commands** (`SITE NUKE`, `SITE UNNUKE`, `SITE NUKES`, `SITE NUKESCLEAN`), which are part of the core and always available;
2. the `autonuke` **plugin**, which scans releases and nukes the ones that are incomplete, empty, missing a file, or bad by IMDB data.

For nuking releases by name (banned words and groups) see [Nukefilter](Nukefilter.md).

## Contents

- [What a nuke does](#what-a-nuke-does)
- [Nuke commands](#nuke-commands)
- [The autonuke plugin](#the-autonuke-plugin)
- [Autonuke config](#autonuke-config)
- [Autonuke commands](#autonuke-commands)
- [Announces](#announces)

## What a nuke does

When a directory is nuked:

- The directory is **renamed** to `[NUKED]-<name>` (in the same parent directory), and nuke information (nuker, reason, multiplier, amounts) is stored on the directory and in the **nukelog**.
- Transfers running in the directory are aborted.
- Every user who uploaded files in it loses credits and uploaded bytes. The amount lost per user is `size * ratio + size * (multiplier - 1)`. With multiplier `0` nobody loses credits; with `1` users lose exactly the credits they earned; with `2` or more they lose extra on top. A user with ratio `0` never loses credits. The user's nuked counter and nuked bytes go up. Users that have since been deleted are skipped.
- Nothing is deleted. The files stay on the slaves, so a nuke can be undone with `SITE UNNUKE`.
- While a nuked path is in the nukelog, **creating or renaming a directory to that path is refused** ("already nuked").

An unnuke renames the directory back, gives the credits and uploaded bytes back to the users, and removes the nukelog entry.

## Nuke commands

They are defined in `config/commands/ftp/core.conf` (see [Site Commands](Site-Commands.md)) and, for IRC, in `config/commands/irc/core.conf` (see [IRC Commands](IRC-Commands.md)). The permissions below are the defaults; change them in the command files (see [Permissions](Permissions.md)).

| Command | Default permission | IRC name |
|---------|--------------------|----------|
| `SITE NUKE` | `=nuke =siteop` (group `nuke` or `siteop`) | `nuke` |
| `SITE UNNUKE` | `=nuke =siteop` | `unnuke` |
| `SITE NUKES` | `*` (everyone) | `nukes` |
| `SITE NUKESCLEAN` | `=nuke =siteop` | `nukesclean` |

### SITE NUKE

```
SITE NUKE <directory> <multiplier> <message>
```

| Argument | Meaning |
|----------|---------|
| `<directory>` | the release directory: a name, a path relative to your current directory, or a full path. If the name has spaces, put it in braces, `{My directory name}`. |
| `<multiplier>` | a whole number, see [What a nuke does](#what-a-nuke-does) |
| `<message>` | the reason, the rest of the line |

Example:

```
SITE NUKE /MP3/Some.Release-GRP 2 bad.rip
```

If you give a plain name while you are in the root directory, the command can **search the index** (see [Index and Search](Index-and-Search.md)) instead of needing a full path: one match is nuked, several matches are listed so you can repeat the command with a full path. This is controlled by the `search` setting of the command in the command file: it is `false` for the FTP command and `true` for the IRC command by default.

The success reply lists each nuked user and the amount of credits lost. A nuke is announced if [Sitebot](Sitebot.md) is running.

### SITE UNNUKE

```
SITE UNNUKE <directory> [message]
```

Undoes a nuke. You can give the name with or without the `[NUKED]-` prefix. As with `SITE NUKE` a plain name in the root directory can be searched in the index. If the nuked directory no longer exists (it was deleted or wiped), the entry is removed from the nukelog instead.

### SITE NUKES

```
SITE NUKES [section]
```

Lists the nukelog: path, multiplier, number of users nuked, size, reason. With a section name only that section is listed.

### SITE NUKESCLEAN

```
SITE NUKESCLEAN
```

Removes nukelog entries whose nuked directory no longer exists.

The nuke messages are in `config/themes/ftp/nuke.theme.default`, `config/themes/ftp/nuke.list.theme.default` and `config/themes/irc/nuke.theme.default` (see [Themes](Themes.md)). The `nuke` and `unnuke` announces use the event names `nuke` and `unnuke` (see [Sitebot](Sitebot.md)).

## The autonuke plugin

The plugin keeps two queues:

- the **scan queue**: new release directories. A release is added when someone creates a directory in a section with `MKD`, and when you run `SITE AUTONUKESCAN`;
- the **nuke queue**: releases that failed a check, with the time they will be nuked.

Every minute:

1. the scanner looks at the releases in the scan queue. A release that is still younger than `minage` (counted from its last modified time) stays in the queue. Once it is old enough it is checked **once** against the entries. If it fails an entry, it goes to the nuke queue with a nuke time of *last modified time + nuke.delay*, and an announce is sent (unless the nuke time has already passed, then it is nuked at once). If it passes, or is older than `maxage`, or is in a section the entries do not cover, it leaves the scan queue;
2. the nuker looks at the nuke queue. When the nuke time has come, the release is checked **again**. If it still fails and has not been approved, it is nuked. If it is now fine or approved, it is dropped from the queue.

Other facts:

- Releases that someone approved with `SITE APPROVE` (see [Site Commands](Site-Commands.md)) are skipped, both when scanning and right before the nuke.
- A release that is deleted (`RMD` or `SITE WIPE`) leaves both queues.
- The queues are held in memory, so they are lost on a restart or when you reload the config in debug mode. `SITE AUTONUKESCAN` fills the scan queue again.
- The first scan runs one minute after start-up or reload, the first nuke check 90 seconds after.
- The check runs first on the release itself, then on its sub-directories that match `x.subdirs` (for example `CD1`, `CD2`). If a sub-directory fails, the sub-directory's **parent** (the release) is nuked.
- The user that appears as the nuker is the account named by `user`. It must exist.

Needs the [Zipscript](Zipscript.md) plugin for the `Incomplete` type and the [IMDB and TVMaze](IMDB-and-TVMaze.md) plugin for the `IMDB` type. See [Plugins](Plugins.md).

## Autonuke config

`config/plugins/autonuke.conf` on the master (copy from `autonuke.conf.dist`). It is reloaded with `SITE RELOAD`.

### General settings

```
user=drftpd
debug=true
exclude.sections=PRE,REQUESTS,ARCHIVE,SPEEDTEST
exclude.dirs=^(?i)((\\[NUKED\\]).*|SORTED|SINGLES)$
exclude.subdirs=^(?i)(Approved.by).*$
```

| Key | Default | Meaning |
|-----|---------|---------|
| `user` | `drftpd` | the user account that is recorded as the nuker |
| `debug` | `true` | **no real nuking**: what would be nuked is written to the log, the announces are still sent. Set to `false` to turn autonuke on. Reloading in debug mode clears the queues. |
| `exclude.sections` | none | comma-separated sections that are never scanned (no spaces) |
| `exclude.dirs` | none | regex of release directory names that are never scanned |
| `exclude.subdirs` | none | regex: skip a release if it has a sub-directory that matches (here, `Approved.by...`) |

### Check entries

After the general settings come numbered entries starting at `1` (`1.type`, `2.type`, ...). The plugin stops reading at the first number without a `type`. The entries are checked in order for each release, and the first one that fails the release queues the nuke.

Keys for all types (all optional except `type`):

| Key | Default | Meaning |
|-----|---------|---------|
| `x.type` | none | `Incomplete`, `Missing`, `Empty` or `IMDB` |
| `x.section` | `*` | `*` = all sections except the excluded ones, or a comma-separated list such as `MP3,XViD` (no spaces) |
| `x.minage` | `120` | minutes. Do not check releases younger than this. |
| `x.maxage` | `20160` | minutes (14 days). Skip releases older than this. |
| `x.dated` | none | for dated sections: `<SECTION>:<DAYS BACK>[,<SECTION>:<DAYS BACK>,...]`. Only check the last N days of dated directories, today included. `2` = today and yesterday. For weekly formats use multiples of 7. |
| `x.dated.format` | `MMdd` | date format of the dated directories |
| `x.subdirs` | none | regex. When set, sub-directories of the release that match are checked with the same type (so `Empty` with `^(?i)(cd|disk).*$` nukes empty `CD1` directories) |
| `x.nuke.delay` | `720` | minutes between the failed check and the nuke, counted from the release's last modified time. A release already older than that is nuked at once. |
| `x.nuke.reason` | `autonuke` | the nuke reason |
| `x.nuke.mult` | `1` | the nuke multiplier |
| `x.irc` | none | the IRC announce text when the release is queued. Users can list the queue with `!autonukes`. Not sent when the release is nuked directly. |
| `x.debug` | `false` | `true` makes this entry only log (like the global `debug`), without nuking |

Type specific keys:

| Type | Key | Meaning |
|------|-----|---------|
| `Incomplete` | `x.min.percent` (default `0`) | the lowest allowed completeness in percent. `0` means the release must be complete. Uses the DIZ file status if there is one, else the SFV status. A release with neither counts as complete. |
| `Missing` | `x.missing` (required) | regex. If **no** file or directory name in the release matches, the release is nuked. |
| `Empty` | none | nuke when there are no files in the release (searching all sub-directories) |
| `IMDB` | `x.imdb.field`, `x.imdb.operator`, `x.imdb.value` | see below |

#### IMDB entries

The release must have IMDB data (see [IMDB and TVMaze](IMDB-and-TVMaze.md)). A release without it is left alone.

| Key | Meaning |
|-----|---------|
| `x.imdb.field` | `Title`, `Language`, `Country`, `Director`, `Genres`, `Plot` (text fields), or `Year`, `Rating`, `Votes`, `Runtime` (number fields) |
| `x.imdb.operator` | for number fields: `<`, `>`, `=` or `!=`. For text fields: leave empty, or `!` |
| `x.imdb.value` | for text fields, a regex that must match the whole value. For number fields, a number; non-digits are removed, so a rating of `7.0` is written `70` |

The release is **nuked when the condition is true** for number fields (`Rating < 70` nukes ratings below 7.0). For text fields without `!` the release is nuked when the regex **matches** the value; with `!` it is nuked when the regex **does not match**.

The comment of the shipped `.dist` file also lists `Screens` and `Limited` as fields. They are not supported in 4.0.x; `Runtime` is.

#### IRC variables

In `x.irc`: `${dir}`, `${path}`, `${user}` (owner), `${group}`, `${timeleft}` (until the nuke), `${section}`, `${dirsize}`. For `Incomplete` also `${var1}` (files missing), `${var2}` (percent missing), `${var3}` (files present), `${var4}` (total files), `${var5}` (percent present).

### Examples

```
user=drftpd
debug=false
exclude.sections=PRE,REQUESTS,ARCHIVE,SPEEDTEST

# Incomplete releases of the last two days in dated sections, 3X
1.type=Incomplete
1.dated=0DAY:2,MP3:2
1.subdirs=^(?i)(cd|disk).*$
1.nuke.reason=auto.incomplete
1.nuke.mult=3
1.irc=${color}04${bold}iNCOMPLETE${bold}${coloroff} -> ${bold}${path}${bold} [ Complete: ${var5}% / AutoNuke: ${timeleft} ]

# MP3 releases with no m3u
2.type=Missing
2.missing=^(?i).+.m3u$
2.section=MP3
2.dated=MP3:2
2.nuke.reason=auto.missing.m3u
2.irc=${color}04${bold}NO M3U${bold}${coloroff} -> ${bold}${path}${bold} [ AutoNuke: ${timeleft} ]

# XViD releases without a sample directory
3.type=Missing
3.missing=^(?i)sample$
3.section=XViD
3.nuke.reason=auto.missing.sample

# Releases with an IMDB rating below 7.0
4.type=IMDB
4.imdb.field=Rating
4.imdb.operator=<
4.imdb.value=70
4.nuke.reason=auto.imdb.rating.below.7.0
```

## Autonuke commands

Defaults to `siteop`, set in `config/commands/ftp/autonuke.conf` (see [Site Commands](Site-Commands.md)). In IRC the commands are `autonukes`, `autonukescan` and `delqueue` (see [IRC Commands](IRC-Commands.md)).

| Command | Meaning |
|---------|---------|
| `SITE AUTONUKES [section]` | show the nuke queue: path, time left, multiplier and reason. With a section, only that section. |
| `SITE AUTONUKESCAN [section]` | add all releases of the sections to the scan queue now (all non-excluded sections if you are in `/` and give no section; the current section otherwise). Useful after a restart. It only queues; the scanner picks them up within a minute. |
| `SITE DELQUEUE [path]` | remove one release (name or path) from the nuke queue; with no argument, clear the whole nuke queue |

Note: there is no separate command to clear the scan queue.

## Announces

If [Sitebot](Sitebot.md) is running, a queued nuke is announced under the event name `autonuke`, with the text of the entry's `x.irc` key. Add lines like these to `irc.announce.conf` to choose channels:

```
#autonuke.path.1=/path/subpath/*
#autonuke.path.1.destination=public
#autonuke.section.1=*
#autonuke.section.1.destination=public
```

The text of the command replies is in `config/themes/ftp/autonuke.theme.default` and `config/themes/irc/autonuke.theme.default` (see [Themes](Themes.md)). The real nuke, when it happens, is announced as a normal `nuke` event.
