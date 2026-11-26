> Applies to DrFTPD 4.0.x.

# Autofreespace

The `autofreespace` plugin frees disk space automatically by **deleting the oldest releases** when a slave is getting full, or when releases are older than you want to keep.

> **Warning.** The plugin deletes whole releases from the site, not only from the slave that is low on space. The release that is picked is the oldest one that has at least one file on that slave; it is removed from the virtual file system and from every slave that holds a copy. Start with `announce.only=true` and watch the announces and the log until you trust the settings.

The plugin is off by default (`mode=disabled`). See [Plugins](Plugins.md) for how plugins are loaded.

## Configuration

`config/plugins/autofreespace.conf` on the master (copy from `autofreespace.conf.dist`). It is reloaded with `SITE RELOAD`; a reload restarts the timer.

```
cycleTime=1440
#excluded.file.1=.*speedtest.*
#excluded.slaves=SLAVE1
announce.only=true
max.iterations=5
mode=disabled
keepFree=100GB
wipe.all.sections=false

#1.section=SECTION1
#1.wipeAfter=43200
#2.path=/MP3/*
#2.wipeAfter=43200
```

| Key | Default | Meaning |
|-----|---------|---------|
| `cycleTime` | `1440` | minutes between two runs. `1440` = 1 day, `10080` = 1 week, `43200` = 1 month. The first run is one `cycleTime` after the start. |
| `excluded.file.N` | none | regex (`N` = 1, 2, ... without gaps). A release whose **name** matches is never deleted, for example `.*speedtest.*`. |
| `excluded.slaves` | none | space-separated slaves that are never cleaned |
| `announce.only` | `true` in the shipped file | `true` = only announce what would be deleted, delete nothing. `false` = really delete. |
| `max.iterations` | `5` | the most releases deleted (or announced) per slave in one run |
| `mode` | `disabled` | `disabled`, `date` or `space`, see below |
| `keepFree` | `100GB` | `space` mode: the free space to keep on each slave, a size such as `100GB` or `500G` |
| `wipe.all.sections` | `false` | `space` mode only, see below |
| `X.section` | none | a section name to consider. Use either `X.section` **or** `X.path` for an entry, with `X` = 1, 2, 3, ... without gaps. |
| `X.path` | none | a path in the virtual file system. A `*` stands for every sub-directory at that level, so `/MP3/*` covers dated MP3 directories. It cannot be `/`, and cannot contain `.` or `..`. If an entry has both `section` and `path`, the path is used. |
| `X.wipeAfter` | none | minutes. Do not delete releases younger than this. **Required in `date` mode**; an entry without a valid value is skipped. |

`cycleTime`, `keepFree` and `max.iterations` must be present in the file (even in `date` mode or when disabled), otherwise the plugin cannot read its settings. Keep the lines from the `.dist` file.

An entry that points at a section that does not exist is skipped with a log message. Releases are the sub-directories of the section (or of each directory matched by `path`); the base directory and `/` are never deleted.

## Modes

### `date`

On each run, every online slave is checked. Releases that have files on the slave and are **older than `wipeAfter`** are removed, oldest first, at most `max.iterations` per slave per run. The age is the creation time of the release in the file system. This mode does not look at free space.

### `space`

On each run, a slave whose free disk space is **at or below `keepFree`** gets cleaned. Releases with files on that slave are deleted, oldest first, until the free space is above `keepFree` again, or `max.iterations` releases have been deleted. `wipeAfter` is not used in this mode.

With `announce.only=true` a low-space slave is only announced, nothing else happens.

Which releases are candidates in `space` mode:

- the entries you wrote (`X.section` / `X.path`);
- if you wrote **no** entry and `wipe.all.sections=true`: every section. Be careful: this can include PRE, REQUESTS, group directories and other non-release sections. If you write any entry at all, even a bad one such as a mistyped section name, `wipe.all.sections` is ignored;
- if you wrote no entry and `wipe.all.sections=false`: nothing is deleted, and a warning is logged.

### General rules

- A slave that is still **remerging** is skipped (its files are not all known yet). If you use `partial.remerge.mode=instant` in `master.conf`, a slave can be online during a long remerge, so its disk may fill up before cleaning resumes (see [Master Configuration](Master-Configuration.md)).
- Slaves that are offline are skipped. Only one run happens at a time.
- A release is skipped if its name matches `excluded.file.N`.
- A run stops when `SITE SHUTDOWN` happens.
- `SITE DELSLAVE` is refused while the slave is in `excluded.slaves`.

## Announces

If [Sitebot](Sitebot.md) is running, the plugin announces under the event name `autofreespace`. The messages are in `config/themes/irc/autofreespace.theme.default`:

| Theme key | When | Variables |
|-----------|------|-----------|
| `afs.announce` | `space` mode: a slave is below `keepFree` (no release picked yet) | `${slave}`, `${slavesize}` |
| `afs.delete` | a release was deleted (or would be, with `announce.only=true`) | `${path}`, `${dir}`, `${size}`, `${date}`, `${slave}`, `${slavesize}` |

To send them to a certain channel, add to `irc.announce.conf` (see [Sitebot](Sitebot.md)):

```
autofreespace.destination=public
```

## Examples

Delete everything in `MP3` and `TV` older than 30 days, but only after you checked the announces for a few days:

```
cycleTime=1440
announce.only=true
max.iterations=5
mode=date
keepFree=100GB

1.section=MP3
1.wipeAfter=43200
2.section=TV
2.wipeAfter=43200
```

Switch to `announce.only=false` once you are happy.

Keep 200 GB free on every slave, taking the oldest releases from two sections and from the dated `MP3` directories, and never touching speedtest files:

```
cycleTime=60
announce.only=false
max.iterations=5
mode=space
keepFree=200GB
excluded.file.1=.*speedtest.*

1.section=0DAY
2.section=TV
3.path=/MP3/*
```
