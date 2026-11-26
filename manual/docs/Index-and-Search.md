> Applies to DrFTPD 4.0.x.

# Index and Search

The master keeps a **search index** of its whole virtual file system. It is a [Lucene](https://lucene.apache.org/) index with one entry for every file and directory (name, path, owner, group, size, dates, slaves, and data added by plugins such as zipscript, MP3 and FLAC info). Searches use the index instead of walking the file tree, so they are fast even on a site with millions of files.

The index is part of the core, there is nothing to enable. It lives on the master only.

## What uses the index

- `SITE FIND`, `SITE SEARCH` and `SITE DUPE` (see [Find](Find.md)).
- `SITE NUKE` and `SITE UNNUKE`, when you give a name instead of a path while you are in the root directory (see [Autonuke](Autonuke.md#nuke-commands)).
- `SITE APPROVE`, when you give a name instead of a full path.
- The [Dupecheck](Dupecheck.md) plugin (finding a release that already exists).
- The [IMDB and TVMaze](IMDB-and-TVMaze.md) plugin (finding NFO files and releases with the same title).

The index is kept up to date by the master itself: whenever a file or directory is created, deleted, renamed or changed, the index is updated.

## Where it is stored

Both are relative to the directory where the master runs:

| Path | Content |
|------|---------|
| `userdata/index/` | the live index |
| `index.bkp/<yyyyMMdd-HHmmss>/` | backups of the index, one directory per backup |

The index engine is selected in `master.conf` (see [Master Configuration](Master-Configuration.md)):

```
indexingengine=org.drftpd.master.indexation.LuceneEngine
```

This is the only engine in 4.0.x, leave it as it is. The 4.0.x index uses the Lucene 3.6 format.

## Configuration

`config/indexation.conf` on the master (copy from `indexation.conf.dist`; the master works without it, using defaults). It is read at start-up and with `SITE RELOAD`.

```
max_hits=100
maxdocs_buffer=120
max_rambuffer=16
optimize_interval=15
native_locking=true
do_backups=true
backup_interval=120
max_backups=2
```

| Key | Shipped | Meaning |
|-----|---------|---------|
| `max_hits` | `100` | the most results a search can return from the index. `0` returns everything and is not recommended. The commands have their own, lower limits too (see [Find](Find.md)). If the line is missing, 50 is used. |
| `maxdocs_buffer` | `120` | advanced: number of entries kept in memory before Lucene writes them out |
| `max_rambuffer` | `16` | advanced: MB of memory Lucene may use before it writes out. Whichever of the two limits is reached first applies. |
| `optimize_interval` | `15` | minutes between index optimizations (merging, which makes searches faster). Use a higher value (for example 30) on very busy sites. |
| `native_locking` | `true` | advanced: use operating system file locks for the index. Set `false` if the index is on a network file system. |
| `do_backups` | `true` | make periodic backups of the index |
| `backup_interval` | `120` | minutes between backups. Do not set it low; one hour or more. |
| `max_backups` | `2` | how many backups to keep. Older ones are deleted. `0` turns the backups off completely (the backup thread is not started). |

## Backups

If backups are on, the master copies the index files into a new directory `index.bkp/<date-time>` every `backup_interval` minutes, and keeps the newest `max_backups`. Backups are plain copies of the Lucene index directory.

The index can always be rebuilt from the file system (see below), so a backup is only a way to avoid a long rebuild. If you want to use one, stop the master, replace the contents of `userdata/index` with the files of one backup directory and start the master again. Then run `SITE REFRESHINDEX` on `/` (see below) if you suspect the backup is older than the changes you made.

## Commands

All three default to the `siteop` permission, set in `config/commands/ftp/core.conf` (see [Site Commands](Site-Commands.md)). IRC has `indexstatus`, `rebuildindex` and `refreshindex` (see [IRC Commands](IRC-Commands.md)).

### SITE INDEXSTATUS

```
SITE INDEXSTATUS [full]
```

Shows the number of entries ("inodes") in the index. With `full` it shows everything: the backend, inodes, deleted inodes, cached inodes, max hits, last optimization, last backup, RAM usage and disk size.

### SITE REBUILDINDEX

```
SITE REBUILDINDEX
```

Throws the whole index away and builds it again by walking the entire virtual file system on the master. Use it when the index is damaged, or after a plugin that adds index data was added. It can take a long time and uses a lot of I/O on large sites, and searches give incomplete results while it runs. Only one rebuild can run at a time. A message is sent to the master's IRC channel when it starts and when it ends, with the number of entries.

### SITE REFRESHINDEX

```
SITE REFRESHINDEX [-q]
```

Re-reads the information of the **current directory and everything below it**, and updates the index entries. It is much lighter than a rebuild: use it for one section or a single release that looks wrong in the search. `-q` hides the progress lines (one line per directory).

```
CWD /MP3
SITE REFRESHINDEX -q
```

## Not found or out of date?

- A release you know exists is not found: run `SITE REFRESHINDEX` in its directory. If many are missing, run `SITE REBUILDINDEX`.
- Results from a search are cut: that is `max_hits` and the `limit` of the command.
- Private paths: search results hide what the user is not allowed to see (see [Permissions](Permissions.md)).
- The index follows the master's virtual file system, not the disks of the slaves. If a slave was added or re-synced, the file system is updated first (see [Adding Slaves](Adding-Slaves.md)) and the index follows.
