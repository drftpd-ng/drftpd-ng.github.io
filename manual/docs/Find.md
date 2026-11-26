> Applies to DrFTPD 4.0.x.

# Find, Search and Dupe

The `find` plugin adds three search commands that use the master's file index (see [Index and Search](Index-and-Search.md)):

| Command | Use |
|---------|-----|
| `SITE SEARCH <word> [word ..]` | quick search for directories by wildcard words |
| `SITE DUPE <name>` | quick search for a file by its exact name |
| `SITE FIND <options> <actions>` | advanced search with filters, and actions on the results (delete, nuke, copy to other slaves, ...) |

The plugin has no config file of its own. All settings are lines in the command file `config/commands/ftp/find.conf` (copy it from `find.conf.dist`; see [Site Commands](Site-Commands.md)). The IRC versions are in `config/commands/irc/find.conf`.

Results that the user is not allowed to see (private paths) are not shown. See [Plugins](Plugins.md) for how plugins are loaded. For the dupe checks done while uploading, see [Dupecheck](Dupecheck.md): that is a different plugin.

## SITE SEARCH

```
SITE SEARCH <word> [word ..]
```

Finds directories whose name contains the words, using wildcards `*` and `?`. A wildcard cannot be in the first three characters. Prints the path of each result. Default permission `*` (everyone).

Settings in `find.conf`:

| Line | Default | Meaning |
|------|---------|---------|
| `limit` | `50` | maximum number of results |
| `type` | `d` | `d` = directories, `f` = files |
| `exact` | `false` | `true` = the argument must be the exact name |

## SITE DUPE

```
SITE DUPE <name>
```

Same code as `SITE SEARCH`, but with `type f` and `exact true` as shipped: it looks for **files** with exactly this name. Permission `*`, `limit 50`. Despite the name it does not use the options of `SITE FIND -dupe`.

## SITE FIND

```
SITE FIND <options> <actions>
SITE FIND options [name]
SITE FIND actions
```

Examples:

```
SITE FIND -section MP3 -age 2024.01.01:2024.03.31 -d -printf "#f by #u: #s"
SITE FIND -f -size 0:0 -user baduser -delete
SITE FIND -name "some words" -sendToSlaves "2 slave1,slave2 5"
SITE FIND -dupe -section MP3 -wipe
```

- Options and actions start with `-`. Their names are not case-sensitive.
- An argument with a space must be in double quotes.
- Giving several options means a result must match **all** of them.
- At least one action is needed (use `-print` to just list), except with `-dupe`, which lists the dupes when no action is given.
- `SITE FIND options` lists every option the master knows, `SITE FIND options <name>` shows the help for one, and `SITE FIND actions` lists the actions.
- Ranges are written `min:max`. Either side can be empty.
- Dates are `yyyy.MM.dd` or `yyyy.MM.dd.HH.mm.ss`, for example `2010.01.29.08.36.33`.
- The search runs on the index, then each hit is checked again against the file system.

Settings in `find.conf`:

| Line | Shipped | Meaning |
|------|---------|---------|
| `limit.default` | `50` | number of results shown when `-limit` is not given |
| `limit.max` | `100` | the most `-limit` can ask for |
| `observe.privpath` | `true` (default) | hide results inside private paths from users that may not see them. Set `false` to ignore the private path rules. |
| `sendToSlaves`, `deleteFromSlaves`, `wipe`, `nuke`, `unnuke` | `=siteop` (`nuke`, `unnuke`: `=siteop =nuke`) | who may use that action, see [Permissions](Permissions.md) |
| `dupewipe`, `dupenuke` | `=siteop` | who may combine `-dupe` with `-wipe` or `-nuke`, in addition to the action's own permission |
| `dupe.exclude` | the regex below | case-insensitive regex of directory names never treated as dupes. `none` turns it off. |
| `dupe.min.age` | `60` | minutes. Do not act on a dupe copy with a change inside it newer than this. `0` turns it off. |

The default `dupe.exclude` is `^((cd|dvd|disc|disk)[0-9]+|sample|samples|subs|proof|covers?|extras?)$`.

### Options

**Name and type**

| Option | Argument | Meaning |
|--------|----------|---------|
| `-name` | `<word[ word ..]>` | name contains these words. Wildcards `*` and `?` work, not in the first three characters. |
| `-regex` | `<pattern>` | regular expression, matched against the full path |
| `-exact` | `<name>` | exact name (wildcards allowed, not in the first three characters) |
| `-endswith` | `<name>` | name ends with this |
| `-f`, `-file` | none | files only |
| `-d`, `-dir` | none | directories only |
| `-section` | `<section name>` | only search in this section (see [Sections](Sections.md)) |

**Owner and time**

| Option | Argument | Meaning |
|--------|----------|---------|
| `-user` | `<user>` | owned by this user |
| `-group` | `<group>` | has this group |
| `-age` | `<start date>:<end date>` | created between the two dates |
| `-size` | `<min>:<max>` | size between min and max. Units as `1k`, `10M`, `2G` (decimal). |
| `-0byte` | none | files of size 0 |

**Slaves** (these only match files)

| Option | Argument | Meaning |
|--------|----------|---------|
| `-slaves` | `<slave[ slave ..]>` | files on these slaves |
| `-nbrofslaves` | `<min>:<max>` | files held by between min and max slaves |

**Output**

| Option | Argument | Meaning |
|--------|----------|---------|
| `-limit` | `<amount>` | number of results to show. Values above `limit.max` are lowered to it. |
| `-sort` | `<field> [asc\|desc]` | sort by `parentPath`, `fullPath` (default), `owner`, `group`, `type`, `nbrOfSlaves`, `lastModified` or `size` |
| `-random` | none | no sorting, first come first served |
| `-quiet` | none | do not print the found items (failed actions are still shown) |

**Duplicates**

| Option | Argument | Meaning |
|--------|----------|---------|
| `-dupe` | `[oldest\|newest]` | see [Finding dupe releases](#finding-dupe-releases) |

**Zipscript options** (releases that the [Zipscript](Zipscript.md) plugin tracked; they match directories)

| Option | Argument | Meaning |
|--------|----------|---------|
| `-incomplete` | none | incomplete releases |
| `-present` | `<min>:<max>` | releases with between min and max files present |
| `-missing` | `<min>:<max>` | releases with between min and max files missing |
| `-percent` | `<min>:<max>` | releases that are between min and max percent complete |

**MP3 and FLAC options** (use the audio data the index has stored)

| Option | Argument | Meaning |
|--------|----------|---------|
| `-mp3genre`, `-mp3title`, `-mp3artist`, `-mp3album` | `<name>` | MP3 releases with this genre, title, artist or album |
| `-mp3year` | `<min year>:<max year>` | MP3 releases within the years |
| `-flacgenre`, `-flactitle`, `-flacartist`, `-flacalbum` | `<name>` | FLAC releases with this genre, title, artist or album |
| `-flacyear` | `<min year>:<max year>` | FLAC releases within the years |

### Actions

An action runs on every result. `-print` and `-printf` can be used by anyone by default. Every other action needs the permission from `find.conf`; without a line for it, it is for `=siteop` only (`-delete` is open to everyone and then goes through the normal delete permissions).

| Action | Arguments | Works on | What it does |
|--------|-----------|----------|--------------|
| `-print` | none | files and dirs | prints the path |
| `-printf` | `<format>` | files and dirs | prints the format with the values filled in, see below |
| `-delete` | none | files | deletes the file as the user would with `DELE` and removes the credits that file earned from its uploader (size times ratio) |
| `-wipe` | none | files and dirs | wipes the file or directory (same permission checks as `SITE WIPE`) |
| `-nuke` | `"<multiplier> <reason>"` | dirs | nukes the directory, see [Autonuke](Autonuke.md#what-a-nuke-does). The reason must be one word. A name that is already in the nukelog is refused. |
| `-unnuke` | `[reason]` | dirs | unnukes the directory |
| `-sendToSlaves` | `"<times> <slave1,slave2,...> [priority]"` | files | adds a [job](Jobs.md) that copies the file to the listed slaves until it is on `<times>` of them. After the job the file is removed from the slaves **not** listed. Priority defaults to 0. |
| `-deleteFromSlaves` | `<slave[ slave ..]>` | files | deletes the file from the named slaves only (it stays in the virtual file system if other slaves still have it) |

`-sendToSlaves` needs the Jobs plugin loaded.

Printf format codes:

| Code | Value |
|------|-------|
| `#f` | name |
| `#p` | full path |
| `#h` | parent path |
| `#H` | parent name |
| `#s` | size |
| `#u` | user |
| `#g` | group |
| `#x` | slaves (a list for files, "no slaves" for dirs) |
| `#t` | last modified |

Example: `SITE FIND -f -user someone -printf "filename: #f size: #s"`.

### Finding dupe releases

`-dupe` looks for **release directories whose name exists more than once** (case-insensitive), for example the same release in two sections or two dated directories.

```
SITE FIND -dupe [oldest|newest] <other options> [actions]
```

- One copy of every dupe is kept: the oldest (first created) by default, or the newest with `newest`. Actions only run on the other copies.
- Only release directories are compared: directories directly inside a section, or inside a dated directory of a dated section. Section directories, dated directories and the sub-directories of releases are never compared. Nuked directories and names matching `dupe.exclude` are ignored.
- Without an action it is a **dry run** that only lists the dupes.
- Only `-print`, `-printf`, `-wipe` and `-nuke` are allowed with `-dupe`. `-wipe` needs the `dupewipe` permission and `-nuke` needs `dupenuke`, on top of the action's own permission.
- `-f`, `-slaves` and `-nbrofslaves` match files only, so they cannot be combined with `-dupe`.
- A copy that has any change inside it more recent than `dupe.min.age` minutes is shown as `SKIP (active)` and left alone (the upload may still be running). Releases with more than 4 levels or 10000 files are shown as `SKIP (too large to check)`.
- `-nuke` refuses a name that is already in the nukelog. With three or more copies only the first is nuked, and the rest are reported as `SKIP (name already nuked)`. Use `-wipe` for those.
- `-limit` counts dupe groups, not directories.
- A directory takes the creation time of its oldest file, so a copy that was merged back from a slave with old file times can look like the first one created.

Example (dry run, then wipe the extra copies in the MP3 section):

```
SITE FIND -dupe -section MP3
SITE FIND -dupe -section MP3 -wipe
```

## Themes

Output formats are in `config/themes/ftp/find.theme.default`, `config/themes/ftp/search.theme.default` and the IRC equivalents (see [Themes](Themes.md)). The plugin has no announces.
