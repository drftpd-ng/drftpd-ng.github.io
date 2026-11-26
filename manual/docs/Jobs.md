> Applies to DrFTPD 4.0.x.

# Jobs

The `jobs` plugin is the master's queue of **file copy jobs between slaves**. A job says: "make sure file F exists on N of these slaves". The job manager picks jobs from the queue, chooses a source slave and a destination slave with [Slave Selection](Slave-Selection.md), and has the slaves transfer the file directly between themselves.

The plugin is used by other plugins, which create jobs for you:

- [Archive](Archive.md) creates one job per file of a release it archives.
- [Mirror](Mirror.md) creates jobs to keep a release on more than one slave.
- `SITE FIND` with the `sendToSlaves` action (see [Find](Find.md)).

You can also add jobs by hand with `SITE ADDJOB`.

If the plugin is not loaded, none of the above work. See [Plugins](Plugins.md).

## Configuration

`config/plugins/jobs.conf` on the master (copy from `jobs.conf.dist`). The file is reloaded with `SITE RELOAD`.

```
useCRC=true
sleepSeconds=10
useSSLTransfers=true
```

| Key | Default | Meaning |
|-----|---------|---------|
| `useCRC` | `true` | check the CRC of the file after each job transfer, so a corrupt copy is detected |
| `sleepSeconds` | `10` in the shipped file | how often (in seconds) the manager looks for a job to start. Each run starts at most one new transfer. If the key is missing the plugin uses 30. A value of `0` or less is treated as half a second. |
| `useSSLTransfers` | `true` | use an encrypted (SSL) connection for slave-to-slave transfers. Needs working [SSL/TLS](SSL-TLS.md) on the slaves. |

Because one transfer is started per `sleepSeconds` tick, a low value gets the queue done faster, a high one keeps the load on slaves low.

## Slave selection for jobs

Two chains in the master `config/` directory are used:

- `slaveselection-jobdown.conf`: picks the **source** slave (a slave that has the file and is not one of the destinations).
- `slaveselection-jobup.conf`: picks the **destination** slave (one of the job's destination slaves, online, that does not have the file yet).

The defaults limit a slave to 2 transfers and 1000 kB/s of load before it is used for a job, so jobs do not starve normal users. See [Slave Selection](Slave-Selection.md) for all filters and for how to test a chain with `SITE SLAVESELECT jobup /path` and `SITE SLAVESELECT jobdown /path`.

## How a job works

A job has:

- a **file**,
- a **priority** (higher number runs first; jobs with the same priority run in the order they were added),
- the number of slaves the file should be on (`timestomirror`),
- a list of **destination slaves**.

The manager:

1. skips jobs that are already done and jobs whose file is gone;
2. checks if the file is already on at least `timestomirror` of the destination slaves. If yes, the job is finished and the clean-up below runs;
3. otherwise picks a source and a destination slave and starts a transfer.

**Clean-up.** When a job finishes, the file is **deleted from every slave that is not in the destination list**. If the file is still on more slaves than `timestomirror`, extra copies are also deleted (offline slaves first). This is how the Archive plugin moves releases off slaves. Be careful with `SITE ADDJOB`: slaves you do not list lose their copy.

The queue is kept in memory only. Jobs are lost when the master restarts, or when the plugin is stopped. Archive and Mirror recreate their jobs on their next cycle.

## Site commands

All of them default to the `siteop` permission, set in `config/commands/ftp/jobs.conf` (see [Site Commands](Site-Commands.md) and [Permissions](Permissions.md)). The same commands exist for IRC in `config/commands/irc/jobs.conf` under the names `addjob`, `listjobs`, `listrunningjobs`, `removejob`, `removejobs`, `startjobs`, `stopjobs` (see [IRC Commands](IRC-Commands.md)).

### SITE ADDJOB

```
SITE ADDJOB <file> <priority> <timestomirror> <destslave> [destslave ...]
```

| Argument | Meaning |
|----------|---------|
| `<file>` | the file, relative to the current directory or a full path. Directories and links are not accepted. |
| `<priority>` | an integer. Higher runs first. |
| `<timestomirror>` | the number of slaves the file should end up on (among the destination slaves) |
| `<destslave>` | one or more slave names to copy the file to |

Unknown slave names are skipped with a message; if no valid slave is left the command gives a usage error. Example: put one file on two slaves:

```
SITE ADDJOB /MP3/Some.Release-GRP/01-track.mp3 3 2 slave1 slave2
```

To handle a whole release use [Archive](Archive.md) (`SITE ARCHIVE`) or [Mirror](Mirror.md), or `SITE FIND` with the `sendToSlaves` action (see [Find](Find.md)), instead of one job per file.

### SITE LISTJOBS

```
SITE LISTJOBS
```

Lists the queue by index number. Waiting jobs are shown with their index and file; running jobs also show speed, progress, total size, source slave and destination slave. The last line gives the total number of jobs.

### SITE LISTRUNNINGJOBS

```
SITE LISTRUNNINGJOBS
```

Same as `SITE LISTJOBS`, but only the jobs that are transferring now. The total at the end still counts the whole queue.

### SITE REMOVEJOB

```
SITE REMOVEJOB <jobindex> | <lowrange>-<highrange> [...]
```

Removes jobs by the index shown in `SITE LISTJOBS`. You can give several indexes and ranges separated by spaces, for example `SITE REMOVEJOB 3 7-12 20`. Indexes start at 1.

### SITE REMOVEJOBS

```
SITE REMOVEJOBS
```

Removes every job that is not transferring right now.

### SITE STOPJOBS and SITE STARTJOBS

```
SITE STOPJOBS
SITE STARTJOBS
```

`STOPJOBS` pauses the queue: no new transfer is started; transfers already running finish. Jobs stay queued. `STARTJOBS` resumes it. The paused state is not saved and the queue starts running again after a restart (it is empty then anyway).

## Announces

The plugin has no announces of its own. Archive announces its start, finish and failure, see [Archive](Archive.md).

The messages of the commands are in `config/themes/ftp/jobs.theme.default` and `config/themes/irc/jobs.theme.default` (see [Themes](Themes.md)).
