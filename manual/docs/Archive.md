> Applies to DrFTPD 4.0.x.

# Archive

The `archive` plugin moves or copies old releases from one place to another, on a timer. It can:

- **move releases between slaves**, for example off fast slaves onto big slow ones after a week,
- **keep several copies** of releases on chosen slaves,
- **move the release directory** to another section or path in the virtual file system (for example `/MP3/0101/Rls` to `/ARCHIVE/MP3/R/Rls`).

Moving data between slaves is done with [Jobs](Jobs.md), so the **Jobs plugin must be loaded** too. The jobs choose their slaves with the `jobup` and `jobdown` chains of [Slave Selection](Slave-Selection.md). Announces go through the [Sitebot](Sitebot.md) plugin. See [Plugins](Plugins.md) for how plugins are loaded.

## How it works

Every `cycletime` minutes the plugin goes through the numbered entries of `archive.conf`. For each entry it starts a worker (at most `maxConcurrentActions` run at once). A worker:

1. looks in the entry's section for the **oldest release** that is older than `archiveafter` (and, if set, younger than `ignoreafter`), matches `archiveregex`, and is not yet archived according to the entry's type;
2. announces the start, creates jobs for every file (if the type moves data between slaves) and waits until all jobs are done;
3. moves the directory if `todirectory` is set;
4. announces the end (or a failure);
5. repeats this `repeat` times in the same cycle.

Notes:

- Only one worker per entry runs at a time. If one is still busy when the next cycle starts, the new one does nothing.
- A release that is already being archived (or is a parent or child of one) is skipped.
- A release is "already archived" when it is on the wanted slaves; such releases are only picked again if the entry also moves directories (`todirectory`).
- A release that failed to move is skipped for the rest of the cycle.
- The section base directory, the current dated directory of a dated section, and `/` are never archived as a release.

## Config file

`config/plugins/archive.conf` on the master (copy from `archive.conf.dist`). It is reloaded with `SITE RELOAD`; the timer restarts with the new `cycletime`.

### Global keys

```
# Default [30]
cycletime=30

# Default [10]
maxConcurrentActions=10
```

| Key | Default | Meaning |
|-----|---------|---------|
| `cycletime` | `30` | minutes between two cycles. The first cycle runs one `cycletime` after start-up. Examples: `1440` = 1 day, `10080` = 1 week. |
| `maxConcurrentActions` | `10` | maximum number of archive actions running at the same time. The plugin raises it to the number of archive types plus 1 if you set it lower. Do not set it very high, the master may become slow. |

### Entry keys

Entries are numbered from `1` without gaps (`1.type`, `2.type`, ...). The master stops reading at the first number that has no `type`.

| Key | Required | Meaning |
|-----|----------|---------|
| `x.type` | yes | the archive type, see [Types](#archive-types) |
| `x.section` | yes | the section to archive from (see [Sections](Sections.md)). An unknown section makes the entry fail with an error in the log. |
| `x.archiveafter` | no, default `0` | archive releases older than this many **minutes** |
| `x.ignoreafter` | no, default none | do not archive releases older than this many minutes (`-1` = no limit) |
| `x.repeat` | no, default `1` | how many releases to handle per cycle for this entry. Types that only move directories can do `repeat` per cycle. |
| `x.scansubdirs` | no, default `0` | `1` = look **one level deeper**: the releases are the sub-directories of the directories in the section (for example `/MP3/0101/Rls` with dated directories) |
| `x.scansubdirsregex` | no | with `scansubdirs=1`: directories matching this regex are not treated as releases, their children are. Default `(?i)^(season.*\|(\d+\|\d+\W\d+\|\d+\W\d+\W\d+)$)` (that is `Season...` and date-like names such as `0101`, `2008-12`, `2008-12-10`). Backslashes are doubled in the file. |
| `x.archiveregex` | no, default `.*` | only archive release **names** that match this regex (the whole name must match) |
| `x.skipincomplete` | no, default `false` | `true` = skip releases that the zipscript says are incomplete (every SFV in the release, also in CD1/CD2 sub-directories, must have all its files; a release without SFV counts as complete). Needs the [Zipscript](Zipscript.md) plugin. Skipped releases do not block newer ones. |
| `x.skipnuked` | no, default `false` | `true` = skip nuked releases (name with the nuke prefix, or with nuke data) |
| `x.todirectory` | no | after archiving, **move** the release to this section name or full path. It must exist (or at least its parent must). If it does not exist at all, nothing is moved. |
| `x.todirectorytype` | no | how to build a sub-directory below `todirectory`, see [todirectorytype](#todirectorytype) |
| `x.rlsregex` | no | regex used for `${rls}` in `rls:` destinations. Group 1 is the value. Default `(?i)(.*)(\.\|-\|_)S\d.*` (the part before `.S01...`). |
| `x.priority` | no, default `3` | priority of the jobs created, higher goes first, see [Jobs](Jobs.md) |
| `x.numofslaves` | by type | how many different slaves each file should end up on |
| `x.slavename.Y` | by type | destination slaves, `Y` = 1, 2, 3, ... without gaps |
| `x.offofslave.Y` | by type | slaves to move the release **off** |
| `x.slavedeadafter` | `ConstantMirroring` only | minutes after which an offline slave is ignored when checking the copies, see below. `0` = never. |

Time values in the entries (`archiveafter`, `ignoreafter`, `slavedeadafter`) are in minutes.

## Archive types

List the types the running master knows with `SITE LISTARCHIVETYPES`. In 4.0.x there are six.

| Type | What it does | Needs |
|------|--------------|-------|
| `ConstantMirroring` | keeps each file on exactly `numofslaves` of the listed slaves, and on **no other** slave | `slavename.Y`, `numofslaves` between 1 and the number of slaves listed |
| `MoveReleaseToSpecificSlaves` | moves the release to `numofslaves` of the listed slaves; copies on other slaves are removed | `slavename.Y`, `numofslaves` >= 1 |
| `MoveReleaseToMostFreeSlaves` | moves the release to the `numofslaves` slaves with the most free disk space | `numofslaves` >= 1 |
| `MoveReleaseOffSlavesToMostFreeSlaves` | moves files off the `offofslave` slaves onto the `numofslaves` slaves with the most free space. Files not on an `offofslave` are not touched. | `offofslave.Y`, `slavename.Y`, `numofslaves` >= 1 |
| `MoveReleaseOffMultipleSlavesToSpecificSlaves` | makes sure a file is on only `numofslaves` of the listed slaves; extra copies are removed | `slavename.Y`, `numofslaves` >= 1 |
| `MoveReleaseToSpecificFolder` | only **moves the directory** (no slave transfers) to `todirectory` | `todirectory` |

Details:

- `ConstantMirroring` with `slavedeadafter`: a slave that has been offline longer than that many minutes is not counted as holding a copy, so the files are copied to another listed slave.
- `MoveReleaseToMostFreeSlaves` and `MoveReleaseOffSlavesToMostFreeSlaves` choose the destination by free disk space; `MoveReleaseOffSlavesToMostFreeSlaves` never chooses an `offofslave` slave. `MoveReleaseOffSlavesToMostFreeSlaves` refuses to load unless it has at least one `offofslave.Y` and at least one `slavename.Y`; a slave listed in both is treated as a destination, not as an "off" slave.
- If an entry has a config error (missing slaves, `numofslaves` < 1, unknown slave name), that entry is skipped with an error in the log and the others continue.
- Any type can also have `todirectory` to move the release after the slaves are done.

### todirectorytype

With `todirectory` the release is moved below that directory. `todirectorytype` adds a sub-directory:

| Value | Result |
|-------|--------|
| (empty) | `<todirectory>/<release>` |
| `alpha` | `<todirectory>/<first letter>/<release>`. Names starting with a digit go to `0-9`, others to `UNKNOWN`. |
| `dated:<format>` | `<todirectory>/<date>/<release>`. `<format>` is a Java date pattern for today, for example `dated:MMdd` or `dated:yyyy-MM`. |
| `rls:<template>` | `<todirectory>/<template>/<release>`. The template may contain `${rls}` (the part of the name matched by `rlsregex`), `${season}` (two digits of the season in `S01E02` or `S01D01` style names), `${episode}` (episode number) and `${regex:<expr>}` (group 1 of your regex applied to the release name). If a variable cannot be resolved, the sub-directory is `UNKNOWN`. |
| `regex:<expr>` | `<todirectory>/<group 1>/<release>`: group 1 of the regex, which must match the whole release name. `UNKNOWN` if it does not match. |

Examples (release `Show.Name.S01E02.720p.WEB-GRP`):

```
# /TV-ARCHIVE/Show.Name/Show.Name.Season.01/Show.Name.S01E02.720p.WEB-GRP
x.todirectory=/TV-ARCHIVE
x.todirectorytype=rls:${rls}/${rls}.Season.${season}

# /TV-ARCHIVE/Show.Name/Show.Name.S01E02.720p.WEB-GRP
x.todirectory=/TV-ARCHIVE
x.todirectorytype=regex:(?i)(.*)(\\.|-|_)S\\d.*
```

For a dated section (`0102`), `rls:2010-${regex:(\\d\\d).*}` gives `2010-01`.

The directory is moved by renaming it in the file system, which keeps the release's file times. If the destination already contains a release with the same name, the move fails and a failure is announced.

## Examples

Mirror everything in `MP3` that is a week old to two of three slaves:

```
1.type=ConstantMirroring
1.section=MP3
1.archiveafter=10080
1.numofslaves=2
1.priority=3
1.slavedeadafter=1440
1.slavename.1=slave1
1.slavename.2=slave2
1.slavename.3=slave3
```

After a month, move releases to the slave with the most free space:

```
2.type=MoveReleaseToMostFreeSlaves
2.section=MP3
2.archiveafter=43200
2.numofslaves=1
2.priority=3
```

Move 1080 releases off two slaves onto two others:

```
3.type=MoveReleaseOffSlavesToMostFreeSlaves
3.section=MOVIES
3.archiveafter=43200
3.numofslaves=2
3.archiveregex=^.*(1080(p|i)).*$
3.offofslave.1=slave1
3.offofslave.2=slave2
3.slavename.1=slave3
3.slavename.2=slave4
3.slavename.3=slave5
```

Move complete, non-nuked releases to specific slaves, and into an alphabetical archive section:

```
4.type=MoveReleaseToSpecificSlaves
4.section=MOVIES
4.archiveafter=1440
4.skipincomplete=true
4.skipnuked=true
4.numofslaves=1
4.slavename.1=slave1
4.slavename.2=slave2
4.todirectory=ARCHIVE
4.todirectorytype=alpha
```

Only move directories (5 per cycle), without touching slaves:

```
5.type=MoveReleaseToSpecificFolder
5.section=MOVIES
5.archiveafter=1440
5.archiveregex=^.*(1080(p|i)).*$
5.todirectory=/folder2/subfolder1
5.todirectorytype=dated:MMdd
5.repeat=5
```

Archive the release directories of a dated section (`/MP3/0101/Rls`):

```
6.type=MoveReleaseToSpecificSlaves
6.section=MP3
6.scansubdirs=1
6.archiveafter=1440
6.numofslaves=1
6.slavename.1=slave1
6.slavename.2=slave2
```

## Site commands

All three default to the `siteop` permission, set in `config/commands/ftp/archive.conf` (see [Site Commands](Site-Commands.md) and [Permissions](Permissions.md)). They also exist for IRC as `archive`, `listarchivetypes` and `listarchivequeue` (see [IRC Commands](IRC-Commands.md)).

### SITE ARCHIVE

```
SITE ARCHIVE <directory> <ArchiveType> [property1] [property2] [property...]
```

Archives one release now, without waiting for the timer. `<directory>` is a release directory inside a section (relative to your current directory, or a full path). `<ArchiveType>` is one of the types above. The properties are the same keys as in `archive.conf`, **without the number**, written as `key=value` with no spaces, for example:

```
SITE ARCHIVE /MP3/Some.Release-GRP MoveReleaseToSpecificSlaves numofslaves=1 slavename.1=slave1 slavename.2=slave2
SITE ARCHIVE Some.Release-GRP MoveReleaseToSpecificFolder todirectory=/ARCHIVE
```

Rules:

- The command refuses the root, a section base directory and the current directory of a dated section. Give a release directory.
- Directories that are already being archived are refused.
- Values cannot contain `=` (the property is split at it), and a property cannot be given twice.
- Manual runs do not apply the `archiveafter` age rule, and `repeat` is not used.

### SITE LISTARCHIVETYPES

```
SITE LISTARCHIVETYPES
```

Prints the archive types known to the master (the list above), as a numbered list. Despite its help text it does not list running archives.

### SITE LISTARCHIVEQUEUE

```
SITE LISTARCHIVEQUEUE [details]
```

Shows how many archive actions are in the queue and how many are running (have jobs). With `details`, each action is listed with its UUID, the archive type class and how many jobs it has.

## Announces

If [Sitebot](Sitebot.md) is running, the plugin announces three events. The format is in `config/themes/irc/archive.theme.default`:

| Event | Theme key | Variables |
|-------|-----------|-----------|
| `archive_start` | `archive.start` | `${type}`, `${srcdir}`, `${rls}`, `${files}` (number of files to transfer) |
| `archive_finish` | `archive.finish`, or `archive.finish.move` when the release was moved to another directory | `${type}`, `${srcdir}`, `${rls}`, `${time}`, and `${destdir}` for `archive.finish.move` |
| `archive_failed` | `archive.failed` | `${type}`, `${srcdir}`, `${rls}`, `${time}`, `${reason}` |

To send them to certain channels, use the path or section lines in `irc.announce.conf` (see [Sitebot](Sitebot.md)):

```
#archive_start.path.1=/path/subpath/*
#archive_start.path.1.destination=public
#archive_start.section.1=*
#archive_start.section.1.destination=public
```

The same pattern exists for `archive_finish` and `archive_failed`.

## Other behaviour

- `SITE DELSLAVE` is refused with the message that the slave is still referenced in the archive configuration while `archive.conf` lists slaves with `slavename.Y`. Remove them from the config first and run `SITE RELOAD`.
- The `directoryspace` and `jobpoints` filters for slave selection are provided by this plugin, see [Slave Selection](Slave-Selection.md).
- Stopping the Jobs queue with `SITE STOPJOBS` (see [Jobs](Jobs.md)) pauses the transfers of running archives; the archive waits.
