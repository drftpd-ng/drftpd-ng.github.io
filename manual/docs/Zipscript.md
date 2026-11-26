> Applies to DrFTPD 4.0.x.

# Zipscript

The zipscript plugin checks the files users upload and keeps track of releases while they are being uploaded (races). It covers four kinds of content:

- **SFV releases**: every file is checked against the CRC32 listed in the release's `.sfv` file.
- **Zip releases**: the number of expected zips is read from the `file_id.diz` inside the zips, and every zip is integrity checked.
- **MP3 files**: ID3 tag information is read and shown.
- **FLAC files**: Vorbis tag information is read and shown.

It adds the following to your site:

- Checking of uploads against the SFV, with automatic deletion of files that do not match.
- Enforcing "SFV first" rules (deny uploads before the SFV, deny files not in the SFV, deny a second SFV).
- Race statistics shown when a user enters a directory or finishes an upload.
- A progress "status bar" entry in directory listings, and `-MISSING` placeholder entries for files that are still missing.
- IRC announces for the first file, new racers, halfway and complete (see [Sitebot](Sitebot.md)).
- `SITE RESCAN` to re-check a directory (see [Site-Commands](Site-Commands.md)).

All settings are in `config/plugins/zipscript.conf` (copy it from `zipscript.conf.dist` if you want to change anything). Most keys are read each time they are used, so a change takes effect without a restart. The `sfvfirst.*` and `sfvdeny.*` keys are only read when the plugin's upload hook is created, which happens when the master starts and when the command configuration is reloaded with `SITE RELOAD`. If a change to one of them does not seem to apply, restart the master.

## Master and slave parts

The plugin has a part on the master and a part on every slave. The master decides what is allowed and keeps the results, the slaves do the work that needs the actual files.

| Where | What it does |
|-------|--------------|
| Slave | Computes the CRC32 of files while they are uploaded and downloaded (see the note below), parses `.sfv` files, reads `file_id.diz` and checks the integrity of zip files, reads ID3 tags from MP3 files and Vorbis comments from FLAC files. |
| Master | Compares the CRC from the slave with the SFV, deletes bad files, tracks the race (who uploaded how many files and how fast), builds the status bar and the race stats, sends the IRC announces and caches the parsed SFV, diz and tag information in the virtual file system. |

Because the SFV, diz and tags are read on a slave, the slave holding the `.sfv` (or the zips, MP3s, FLACs) must be online for the checks to work. If it is offline, the master skips the check and tells the user (for example "SFV unavailable, slave(s) with .sfv file is offline").

The CRC32 of an upload is the one the slave calculated during the transfer. This needs checksums enabled in the slave configuration (`enableuploadchecksums` and `enabledownloadchecksums`, both `true` in the default `slave.conf.dist`, see [Slave-Configuration](Slave-Configuration.md)). If checksums are disabled, a file with CRC 0 and a size above zero cannot be verified and is let through (the user sees "checksum match: SLAVE/SFV: DISABLED"). A zero byte file that is listed in the SFV is deleted.

## What happens on upload

When a user uploads a file (`STOR`):

1. **Before the transfer starts** (checks that can deny the upload):
   - A second `.sfv` is refused when the directory already has one ("Multiple SFV files not allowed").
   - If `sfvdeny.subdir` is on, an `.sfv` is refused when the directory contains subdirectories such as `CD1` (see the section on SFV first and deny rules below).
   - If SFV first is required and the path is enforced, a file is refused with "You must upload sfv first" when there is no SFV yet, unless its extension is allowed.
   - If `sfv.restrict.files` is on, a file that is not listed in the SFV is refused with "File not found in sfv", unless its extension is allowed.
2. **After the transfer**:
   - The CRC from the slave is compared with the SFV entry. A match is reported to the user. A mismatch deletes the file ("Checksum mismatch, deleting file"). A file without an entry in the SFV is kept ("no entry in sfv for file").
   - When the `.sfv` itself is uploaded after some of its files, the files already in the directory are checked against it using the CRC stored when they were uploaded, and the ones that do not match are deleted and their upload credits and stats are taken back (`sfv.check.existing`).
   - `.zip` files get an integrity check on a slave (`stor.zip.integrity.check.enabled`). A damaged zip is deleted.
   - Race stats, MP3 info and FLAC info are printed in the upload reply if enabled.
   - The race events used for the IRC announces are generated (first file, new racer, halfway, complete).

When a user downloads a file (`RETR`), the CRC from the transfer is compared with the SFV and a comment is added to the reply ("checksum from transfer matched checksum in .sfv" or a warning if it did not). Nothing is deleted on download.

When an `.sfv` is deleted, the cached SFV information of the directory is dropped.

## SFV first and deny rules

These keys decide which uploads are denied. They only apply to directories where an SFV is used.

- `sfvfirst.required` turns the "SFV first" rule on.
- `sfvfirst.pathcheck` and `sfvfirst.pathignore` select the directories where the rule is enforced. Both are space separated lists of path patterns (glob style, as in [Permissions](Permissions.md)). The rule is enforced in a directory when it matches `sfvfirst.pathcheck` and does not match `sfvfirst.pathignore`.
- `sfvfirst.users` is a permission expression (as in [Permissions](Permissions.md)) that selects who is subject to the rule. `*` means everybody. To exempt the siteop group use `!=siteop *`.
- `allowedexts` is a list of file extensions (separated by spaces, without the dot) that may always be uploaded, even before the SFV and even if they are not in it. `sfv` is always allowed, you do not need to list it.
- `sfvfirst.allownoext` also allows files that have no extension.
- `sfv.restrict.files` additionally refuses files that are not listed in the SFV. It only takes effect where SFV first is enforced.

The SFV-first and file restriction checks only run when `sfvfirst.required=true`. The refusal of a second `.sfv` and the `sfvdeny.subdir` check run regardless.

`sfvdeny.subdir` is meant for multi disc releases. When it is `true`, an `.sfv` is refused in a directory that already contains a subdirectory whose name matches `sfvdeny.subdir.include` (a regular expression). Paths matching `sfvdeny.subdir.exclude` (a regular expression matched against the full path of the file) are not affected.

## Race stats, status bar and missing files

**Race stats.** When enabled, a box with the users and groups taking part in the race and their file count, size, speed and percentage is added to the reply when a user enters a directory with an SFV (`cwd.racestats.enabled`, and `cwd.zip.racestats.enabled` for zip releases) and after each upload (`stor.racestats.enabled`, `stor.zip.racestats.enabled`). The layout is in `zipscript.post.hook.theme` (see [Themes](Themes.md)).

**Status bar.** In a directory listing a progress entry is added. With the default theme it looks like this:

```
[12 of 15 files = 80% complete at 540MB]
```

(Exact byte formatting depends on the size.) If some of the files are on slaves that are offline, a second part is added, separated by `|`: `[12 of 15 files = 80% complete at 540MB | 3 files offline = 75% online]`. For MP3 and FLAC releases the genre and year from the tag are added to the same bar. For zip releases the bar uses the diz text, for example `5 of 8 zip files = 62% complete at 120MB`.

The bar is shown as a directory entry when `statusbar.directory=true`, otherwise as a file entry.

**Missing files.** With `files.missing.enabled=true` the listing shows a placeholder entry named `<filename>-MISSING` for each file listed in the SFV that does not exist (yet). Placeholders are only added for SFV releases. (Files that exist but are on an offline slave are shown as `<filename>-OFFLINE` by the core listing theme.)

## Configuration reference

The values below are the defaults shipped in `zipscript.conf.dist`.

### Listing

| Key | Default | Meaning |
|-----|---------|---------|
| `statusbar.enabled` | `true` | Show the status bar in directory listings. |
| `statusbar.directory` | `true` | Status bar is a directory entry (`true`) or a file entry (`false`). |
| `files.missing.enabled` | `true` | Show `-MISSING` entries for files not yet uploaded. |

### Race stats and info on CWD / STOR

| Key | Default | Meaning |
|-----|---------|---------|
| `cwd.racestats.enabled` | `true` | Race stats when entering an SFV directory. |
| `stor.racestats.enabled` | `true` | Race stats after an upload in an SFV directory. |
| `cwd.zip.racestats.enabled` | `true` | Race stats when entering a zip release directory. |
| `stor.zip.racestats.enabled` | `true` | Race stats after an upload in a zip release directory. |
| `cwd.diz.info.enabled` | `true` | Show the `file_id.diz` text when entering a zip release directory. |
| `cwd.mp3info.enabled` | `true` | Show MP3 info when entering a directory with MP3 files. |
| `stor.mp3info.enabled` | `true` | Show MP3 info after an MP3 upload. |
| `cwd.flacinfo.enabled` | `true` | Show FLAC info when entering a directory with FLAC files. |
| `stor.flacinfo.enabled` | `true` | Show FLAC info after a FLAC upload. |

### Checks and rules

| Key | Default | Meaning |
|-----|---------|---------|
| `sfv.restrict.files` | `true` | Only allow files that are listed in the SFV (where SFV first is enforced). |
| `sfv.check.existing` | `true` | When an SFV is uploaded after some of its files, check those files and delete the ones that do not match. A wrong SFV will delete the already uploaded files it does not match. |
| `sfvfirst.required` | `true` | Refuse files before the SFV. |
| `sfvfirst.allownoext` | `true` | Allow files without extension even if SFV first is required. |
| `allowedexts` | `nfo txt jpg diz avi vob m2ts mkv mp4 zip m3u` | Extensions allowed even if not in the SFV or before the SFV. |
| `sfvfirst.pathcheck` | `*` | Paths where SFV first is enforced. |
| `sfvfirst.pathignore` | `/PRE/* /REQUESTS/* */Subs/ */Cover/ */Covers/ /SPEEDTEST/*` | Paths where SFV first is not enforced. |
| `sfvfirst.users` | `*` | Who is subject to SFV first. |
| `sfvdeny.subdir` | `false` | Refuse an `.sfv` when a matching subdirectory exists. |
| `sfvdeny.subdir.include` | see below | Regular expression for subdirectory names that block an `.sfv`. |
| `sfvdeny.subdir.exclude` | see below | Regular expression for paths that are never affected. |
| `stor.zip.integrity.check.enabled` | `true` | Check the integrity of uploaded `.zip` files and delete damaged ones. |

The defaults of the two regular expressions, as written in the file:

```
sfvdeny.subdir.include=.*(?i)(cd[1-9]|dvd[1-9]|dis(c|k)[1-9]).*
sfvdeny.subdir.exclude=.*(?i)(SPEEDTEST|REQUESTS|STAFF|PRE).*
```

### IRC announce

| Key | Default | Meaning |
|-----|---------|---------|
| `irc.sfv.maxusers` | `10` | Maximum number of users in the "complete" announce of an SFV release. |
| `irc.sfv.maxgroups` | `10` | Maximum number of groups in the "complete" announce of an SFV release. |
| `irc.zip.maxusers` | `10` | Same for zip releases. |
| `irc.zip.maxgroups` | `10` | Same for zip releases. |

### Keys present in the file but not used

`zipscript.conf.dist` also contains `cwd.id3info.enabled` and `cwd.dizinfo.enabled`. The plugin does not read these two keys: MP3 info is controlled by `cwd.mp3info.enabled` and `stor.mp3info.enabled`, the diz text by `cwd.diz.info.enabled`. Changing them has no effect.

## Example: relaxed SFV first for staff

Everybody except the siteop group must upload the SFV first, and the `/STAFF` area is not checked at all:

```
sfvfirst.required=true
sfvfirst.pathcheck=*
sfvfirst.pathignore=/PRE/* /REQUESTS/* /STAFF/*
sfvfirst.users=!=siteop *
```

## Commands

The plugin adds one command. See [Site-Commands](Site-Commands.md) and [IRC-Commands](IRC-Commands.md) for how commands and their permissions are defined.

| Command | Default permission | Description |
|---------|--------------------|-------------|
| `SITE RESCAN` (IRC: `rescan`) | everybody (`*`) | Re-check the files of a directory against its SFV. |

```
SITE RESCAN [-r] [noforce] [nodelete] [nodelete0byte] [quiet] [file/path]
```

- With no path the current directory is scanned. A path that does not start with `/` is relative to your current directory. A path can be a directory or a single file (a single file cannot be combined with `-r`).
- `-r` scans all subdirectories as well.
- By default the CRC of every file is recalculated on a slave. `noforce` uses the CRC that is already stored instead, which is much faster but does not detect a file that changed on disk. If both `nodelete` and `nodelete0byte` are given, the rescan always uses the stored CRC.
- By default a file whose CRC does not match is **deleted**, and a zero byte file that is listed in the SFV is deleted. `nodelete` keeps files with a bad CRC (they are only reported), `nodelete0byte` keeps zero byte files. The built-in help text of the command words this the other way around; the behaviour described here is what the code does.
- `quiet` only prints the files that are not good.
- For each file the result is printed as `OK`, `FAILED - checksum mismatch`, `FAILED - failed to checksum file`, `ZEROBYTE`, or `MISSING` / `OFFLINE` (file not on the site, or no slave online with the file). Each line shows the CRC from the SFV and from the slave.
- After the SFV check, a rescan of a directory with `.zip` files also checks each zip's integrity (damaged zips are deleted) and re-reads the diz info, so that a corrected or late `file_id.diz` is picked up.

The default permission `*` lets any logged in user run a rescan, including its file deletions. If you do not want that, change the `perms` line of `RESCAN` in the command configuration (for example to `=siteop`).

## IRC announces

The plugin produces these announce types. Where each one goes is configured in `irc.announce.conf` (see [Sitebot](Sitebot.md)). The text is in the IRC theme files (see [Themes](Themes.md)).

| Announce type | Sent when | Theme file / keys |
|---------------|-----------|-------------------|
| `store.first` | The first file of an SFV or zip release is uploaded (only for releases with more than one file). | `sfv.announcer.theme`: `sfv.store.first`; `zip.announcer.theme`: `zip.store.first` |
| `store.race` | A new user joins the race. | `sfv.store.race`, `zip.store.race` |
| `store.halfway` | Half of the files are there (SFV releases with at least 4 files; zip releases use the diz total). | `sfv.store.halfway`, `zip.store.halfway` |
| `store.complete` | The release is complete. Followed by a list of the top users and top groups. | `sfv.store.complete`, `sfv.store.complete.racer`, `sfv.store.complete.group.header`, `sfv.store.complete.group` and the same with `zip.` |
| `mp3` | The first MP3 of a release is uploaded. | `mp3.announcer.theme`: `id3tag` |
| `flac` | The first FLAC of a release is uploaded. | `flac.announcer.theme`: `vorbistag` |

Example, taken from the default `sfv.announcer.theme.default` (the `${...}` fields are explained in [Themes](Themes.md)):

```
sfv.store.first=${color}15${bold}FIRST${bold}${coloroff} in ${color}${sectioncolor}${bold}${section}${bold}${coloroff}: -> ${bold}${path}${bold} by ${bold}${user}${bold}${color}14${bold}/${bold}${coloroff}${group} @ ${bold}${speed}${bold}. Expecting ${expectedsize} in ${files} files.
```

The tags available in the `store.*` announces include: `section`, `sectioncolor`, `path`, `user`, `group`, `speed`, `files`, `filesleft`, `size`, `averagespeed`, `secondstocomplete`, `expectedsize` (first), `leaduser`, `leadgroup`, `leadsize`, `leadfiles`, `leadpercent`, `leadspeed` (halfway), `racers`, `groups` (complete). The per-racer lines of the complete announce have `position`, `user`, `group`, `size`, `files`, `percent`, `speed`, and the user's totals `alup`, `aldn`, `monthup`, `monthdn`, `wkup`, `wkdn`, `dayup`, `daydn`. The group lines have `position`, `group`, `size`, `files`, `percent`, `speed`.

If a release is not announced at all, check that `store.first`, `store.race`, `store.halfway` and `store.complete` have a destination in `irc.announce.conf`, either through a section rule or the `default` rules.

## Theme files

| File (`config/themes/ftp/`) | What it controls |
|------------------------------|------------------|
| `zipscript.list.theme` | The status bar format, separator, the "complete" and "offline" parts, and the `-MISSING` entry name. |
| `zipscript.diz.theme` | The status bar parts for zip releases. |
| `zipscript.post.hook.theme` | The race stats box. |
| `zipscript.mp3.post.hook.theme` | The MP3 info box (`cwd.mp3info.text`, `stor.mp3info.text`). |
| `zipscript.mp3.statusbar.theme` | The genre/year part of the status bar for MP3 releases. |
| `zipscript.flac.post.hook.theme` | The FLAC info box. |
| `zipscript.flac.statusbar.theme` | The genre/year part of the status bar for FLAC releases. |

| File (`config/themes/irc/`) | What it controls |
|------------------------------|------------------|
| `sfv.announcer.theme` | IRC announces for SFV releases. |
| `zip.announcer.theme` | IRC announces for zip releases. |
| `mp3.announcer.theme` | IRC announce for MP3 info. |
| `flac.announcer.theme` | IRC announce for FLAC info. |

Remember that you change a theme by creating a file without the `.default` suffix, see [Themes](Themes.md).

## Search

The plugin also stores the information it reads in the site index, so it can be searched with the index and find commands (see [Index-and-Search](Index-and-Search.md)): the MP3 artist, album, title, genre and year, the FLAC artist, album, title, genre and year, and the number of files present and missing and the completion percentage of SFV releases.

## Related pages

- [Sitebot](Sitebot.md) and [Themes](Themes.md)
- [Race-Leader](Race-Leader.md): announces a change of race leader (needs an SFV)
- [Links](Links.md): link directories for incomplete releases and releases without SFV
- [Slave-Configuration](Slave-Configuration.md)
