> Applies to DrFTPD 4.0.x.

# Mirror

The `mirror` plugin keeps **extra copies** of newly uploaded files on other slaves, so the loss or slowness of one slave does not lose a release. When an upload finishes, the plugin adds a [job](Jobs.md) that copies the file to more slaves. `SITE UNMIRROR` goes the other way and reduces a directory to one copy per file.

The **Jobs plugin must be loaded** for Mirror to work (see [Plugins](Plugins.md)). The jobs choose their slaves with the `jobup` and `jobdown` chains of [Slave Selection](Slave-Selection.md).

For moving releases between slaves on a timer, see [Archive](Archive.md) instead.

## Configuration

`config/plugins/mirror.conf` on the master (copy from `mirror.conf.dist`). It is reloaded with `SITE RELOAD`.

```
# Value in minutes, 0 to disable.
pre.unmirror.time=60
```

Then numbered blocks, starting at `1`:

```
1.nbrOfMirrors=2
1.priority=3
1.path1=/GROUPS/FIRSTGROUP/.*
1.path2=/GROUPS/SECONDGROUP/.*
1.excludePath1=(?i).*/(sample|cover)/.*
1.slaves=SLAVE1 SLAVE2 SLAVE3
1.excludeSlaves=SLAVE8 SLAVE9
1.unmirrorExclude=(?i).*\.(nfo|sfv)$
```

| Key | Required | Meaning |
|-----|----------|---------|
| `pre.unmirror.time` | no, default `60` | minutes after a `SITE PRE` before the pre'd release is unmirrored. `0` disables this. |
| `x.nbrOfMirrors` | yes | how many slaves each file should be on, **at least 2**. The block is the start of an entry: the plugin stops reading at the first number without `nbrOfMirrors`. |
| `x.priority` | no, default `3` | priority of the job in the queue (must be 1 or higher), see [Jobs](Jobs.md) |
| `x.pathN` | yes | regex for the files this block applies to, matched against the **full path including the file name**. `N` = 1, 2, 3, ... without gaps. You can give several. |
| `x.excludePathN` | no | regex of paths to leave out. `N` = 1, 2, ... |
| `x.slaves` | no | space-separated slaves that may hold copies. The slaves the file is already on are added automatically. If this key is not set, every online slave is allowed. |
| `x.excludeSlaves` | no | space-separated slaves that must not get copies. Only used when `x.slaves` is not set. |
| `x.unmirrorExclude` | no | regex of paths that `SITE UNMIRROR` (and the automatic unmirror after PRE) must leave alone. It is global: every block's `unmirrorExclude` is added to one list. |

Important:

- The path keys are `x.path1`, `x.path2` (no dot before the number) in 4.0.x. The comments in the shipped `mirror.conf.dist` write `x.path.y` and `1.path.1` in the example; those spellings are not read by the code.
- Regexes match the **whole** path, so end them with `.*`, for example `/MP3/.*`.
- A block whose `nbrOfMirrors` is not a number, or lower than 2, is an error; leave it out instead of writing `1`. (The master logs the error for it.)
- Slave names that do not exist are logged as a config error.
- The first block whose path matches (and that does not match an `excludePath`) is used; later blocks are ignored for that file.

## Behaviour

- **After an upload.** When a `STOR` finishes successfully (reply 226) and the file matches a block, the plugin collects the slaves: the slaves that already hold the file plus either `x.slaves`, or all online slaves minus `x.excludeSlaves`. If there are at least `nbrOfMirrors` of them, it queues a job to put the file on `nbrOfMirrors` slaves. If there are too few slaves, a warning is logged and nothing is queued.
- **SITE PRE.** When a release is pre'd (see [Site Commands](Site-Commands.md)), the waiting mirror jobs of that release are removed from the queue (running ones are left). If `pre.unmirror.time` is not `0`, the release is unmirrored that many minutes later, so a pre'd release does not keep occupying a copy on a second slave.
- **SITE DELSLAVE.** Deleting a slave is refused while the slave is named in `x.slaves` or `x.excludeSlaves` of any block. Remove it from `mirror.conf` first and run `SITE RELOAD`.
- Because a finished job deletes the file from slaves that are **not** in its destination list, a mirrored file ends up on exactly the slaves chosen above (see "Clean-up" in [Jobs](Jobs.md)).

The plugin has no announces and no theme of its own.

## Site commands

### SITE UNMIRROR

```
SITE UNMIRROR <path>
```

Unmirrors a directory and everything below it: for each file, one copy is kept (the first slave in its list) and the other copies are deleted from their slaves. Files and directories matching `unmirrorExclude` are skipped. Success is answered with "Directory successfully unmirrored!". It defaults to the `siteop` permission, configured in `config/commands/ftp/mirror.conf` (see [Site Commands](Site-Commands.md) and [Permissions](Permissions.md)). The IRC command is `unmirror` (see [IRC Commands](IRC-Commands.md)).

Note that unmirroring does not stop the mirror rules from creating new jobs for **new** uploads.

## Example

Keep every file under `/GROUPS` on two slaves, never on `slave9`, but not samples and covers, and never unmirror NFO and SFV files:

```
pre.unmirror.time=60

1.nbrOfMirrors=2
1.priority=3
1.path1=/GROUPS/.*
1.excludePath1=(?i).*/(sample|cover)/.*
1.excludeSlaves=slave9
1.unmirrorExclude=(?i).*\.(nfo|sfv)$
```
