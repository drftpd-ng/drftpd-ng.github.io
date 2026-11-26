> Applies to DrFTPD 4.0.x.

# Slave Selection

Every time a client uploads or downloads a file, the master has to decide **which slave** handles the transfer. [Jobs](Jobs.md) and [Archive](Archive.md) make the same decision for the transfers they start. DrFTPD does this with a *filter chain*: a list of filters, run one after the other, each of which changes a score table with one line per candidate slave. When the last filter has run, the slave with the highest score wins.

## Contents

- [How it works](#how-it-works)
- [Config files](#config-files)
- [Chain syntax](#chain-syntax)
- [Filters](#filters)
- [The assign syntax](#the-assign-syntax)
- [Ties](#ties)
- [Examples](#examples)
- [Testing a chain with SITE SLAVESELECT](#testing-a-chain-with-site-slaveselect)
- [Reloading](#reloading)

## How it works

1. The master builds a candidate list. For an upload that is every online slave. For a download it is every online slave that has the file. For a job download it is every slave that has the file and is not a destination of the job. For a job upload it is every online destination slave that does not already have the file.
2. Every candidate starts with a score of 0.
3. The filters of the matching chain run in numeric order. A filter can **add or subtract points**, or **remove** the slave from the table.
4. The slave with the highest score is used. If several slaves share the highest score, see [Ties](#ties).
5. If every slave was removed, the transfer fails because no slave is available.

The selection manager is chosen in `master.conf` (see [Master Configuration](Master-Configuration.md)):

```
slaveselection=org.drftpd.master.slaveselection.filter.SlaveSelectionManager
```

This is the default and the only manager in 4.0.x. The filters described on this page exist only in this manager.

## Config files

There are four chains, one file each, in the master `config/` directory:

| File | Used for |
|------|----------|
| `slaveselection-up.conf` | client uploads |
| `slaveselection-down.conf` | client downloads |
| `slaveselection-jobup.conf` | the receiving side of a job (see [Jobs](Jobs.md)) |
| `slaveselection-jobdown.conf` | the sending side of a job |

As with other config files, copy `slaveselection-*.conf.dist` to `slaveselection-*.conf` only if you want to change something.

`slaveselection-old.conf.dist` also ships in `config/`. Nothing in 4.0.x reads it, so you can ignore it.

The master reads all four files at start-up. A broken line (unknown filter, unknown slave name, missing key) stops the master from starting.

### Defaults

`slaveselection-up.conf.dist`:

```
1.filter=bandwidth
1.multiplier=1

2.filter=minfreespace
2.multiplier=1
2.minfreespace=1GB

3.filter=minfreespace
3.minfreespace=100MB
3.multiplier=0

4.filter=cycle
```

`slaveselection-down.conf.dist`:

```
1.filter=bandwidth
1.multiplier=1

2.filter=cycle
```

`slaveselection-jobup.conf.dist`:

```
1.filter=maxtransfers
1.maxtransfers=2

2.filter=maxbandwidth
2.maxbandwidth=1000kb

3.filter=bandwidth
3.multiplier=1

4.filter=minfreespace
4.multiplier=1
4.minfreespace=1GB

5.filter=minfreespace
5.minfreespace=100MB
5.multiplier=remove

6.filter=cycle
```

`slaveselection-jobdown.conf.dist`:

```
1.filter=maxtransfers
1.maxtransfers=2

2.filter=maxbandwidth
2.maxbandwidth=1000kb

3.filter=bandwidth
3.multiplier=1

4.filter=cycle
```

## Chain syntax

Every filter is a group of lines that start with the same number `N`:

```
N.filter=<filter name>
N.<setting>=<value>
```

- Filters run in increasing numeric order.
- Numbers should run from 1 without gaps. If they do not, the master logs a warning but still loads all of them in numeric order.
- `01.filter` (leading zero) is ignored with a warning, because its settings would be read from `1.*`.
- Filter names are not case-sensitive.
- Order matters. Put the filters that **remove** slaves (`maxtransfers`, `maxbandwidth`, `matchdir` with `remove`) early, then the filters that move scores, and `cycle` last.

### Multipliers

`bandwidth`, `minfreespace` and `mintimeonline` take a `multiplier`. It is a number such as `1`, `0.5` or `5`. You can chain operations with `*` and `/`, for example `2*3` or `1/4`. The word `remove` is the same as `0`.

### Sizes and times

Sizes (`minfreespace`, `maxbandwidth`) accept a plain number of bytes or a number with a unit: `K`, `M`, `G`, `T`, `P`, `E`, optionally followed by `B`. Units are decimal (`1k` = 1000 bytes); add an `i` for binary units (`1KiB` = 1024 bytes). The case does not matter.

`maxbandwidth` also accepts a trailing `/s` (`1000kb/s`), which is ignored.

Times (`mintimeonline`) accept `ms`, `s` or `m` suffixes (`500ms`, `30s`, `2m`). A bare number is milliseconds.

## Filters

These are all the filter types in 4.0.x. The first group is in the core, the last two come with the [Archive](Archive.md) plugin.

| Filter | Effect |
|--------|--------|
| [`bandwidth`](#bandwidth) | subtracts points for current throughput |
| [`cycle`](#cycle) | among equal scores, favours the slave idle the longest |
| [`matchdir`](#matchdir-and-matchdirex) | adds, subtracts or removes slaves for paths matching a glob |
| [`matchdirex`](#matchdir-and-matchdirex) | same, with a regular expression |
| [`maxbandwidth`](#maxbandwidth) | removes slaves above a throughput |
| [`maxtransfers`](#maxtransfers) | removes slaves with too many transfers |
| [`maxtransfersperslave`](#maxtransfersperslave) | per-slave transfer limits |
| [`minfreespace`](#minfreespace) | penalises or removes slaves low on disk space |
| [`mintimeonline`](#mintimeonline) | penalises slaves used a very short time ago |
| [`slavetop`](#slavetop) | favours slaves that already hold files of the release |
| [`stripefiles`](#stripefiles) | keeps only slaves with the fewest files in the directory |
| [`jobpoints`](#jobpoints-archive-plugin) | Archive plugin: bonus for archive destination slaves |
| [`directoryspace`](#directoryspace-archive-plugin) | Archive plugin: penalty if the archive does not fit |

### bandwidth

```
N.filter=bandwidth
N.multiplier=1
```

Subtracts `current throughput * multiplier` from the score. Throughput is the slave's current speed in the direction of the transfer: upload speed for the `up` and `jobup` chains, download speed for the `down` and `jobdown` chains. A busy slave gets a lower score than an idle one. A higher multiplier makes busy slaves lose more.

### cycle

```
N.filter=cycle
```

No settings. Among the slaves that have **exactly the same score** at this point, the one that was used least recently for this direction gets +1 point. Put it last, after all the filters that change scores.

### matchdir and matchdirex

```
N.filter=matchdir
N.match=/section/*
N.assign=slave1+500 slave2-500 slave3+remove
N.assume.remove=false
N.negate.expression=false
```

If the **full path** of the file being transferred matches `match`, the `assign` list is applied to the score table. Otherwise the filter does nothing.

| Key | Required | Meaning |
|-----|----------|---------|
| `match` | yes | `matchdir`: a glob (`*` any characters, `?` one character) that must match the whole path. `matchdirex`: a regular expression, case-insensitive, that may match anywhere in the path (use `^` and `$` to anchor it). |
| `assign` | yes | space-separated list of entries, see [the assign syntax](#the-assign-syntax) |
| `assume.remove` | no, default `false` | if `true`, every slave **not mentioned** in `assign` is removed when the path matches. This also covers slaves you add later. It has no effect if `assign` contains an `all` entry. |
| `negate.expression` | no, default `false` | if `true`, the filter applies to paths that do **not** match |

Example match values: `/MP3/*` (glob) or `^/(MP3|FLAC)/.*$` (regex).

For uploads the path is the path of the file being uploaded, so `/section/*` covers every upload under `/section`.

### maxbandwidth

```
N.filter=maxbandwidth
N.maxbandwidth=1000kb
```

Removes every slave whose current throughput in the transfer direction is **greater than** `maxbandwidth`. See [Sizes and times](#sizes-and-times) for the format. The value is per second, so `1000kb` means 1000 kB/s.

### maxtransfers

```
N.filter=maxtransfers
N.maxtransfers=2
```

Removes every slave that already has **more than** `maxtransfers` transfers in the same direction (uploads for the `up` and `jobup` chains, downloads for the `down` and `jobdown` chains). With `maxtransfers=2` a slave with 2 running uploads is still a candidate, a slave with 3 is not.

### maxtransfersperslave

```
N.filter=MaxtransfersPerslave
N.assign=slave1+10 slave2+5
N.match=^/section/.*$
N.negate.expression=false
```

Sets a separate limit for named slaves. A slave listed in `assign` is removed when it has **more than** that many transfers (uploads and downloads counted together). Slaves not listed are not affected.

| Key | Required | Meaning |
|-----|----------|---------|
| `assign` | yes | `slavename+limit` entries. `remove` is refused. |
| `match` | yes | regular expression (case-insensitive), searched anywhere in the path. The limits only apply to paths that match. Use `.*` to apply them everywhere. |
| `negate.expression` | no, default `false` | apply the limits to paths that do **not** match |

### minfreespace

```
N.filter=minfreespace
N.multiplier=1
N.minfreespace=1GB
```

For every slave whose free disk space is below `minfreespace`:

- if `multiplier` is `0` or `remove`, the slave is removed;
- otherwise `(minfreespace - free space) * multiplier` points are subtracted.

The further a slave is below the limit, the more it loses. The default `up` chain uses two lines: a soft penalty below 1 GB, and a hard removal below 100 MB.

### mintimeonline

```
N.filter=mintimeonline
N.mintime=30s
N.multiplier=1
```

If a slave was last used for this direction less than `mintime` ago, subtracts `time since last use in milliseconds * multiplier` from its score. Not used in the default chains.

### slavetop

```
N.filter=slavetop
N.topslaves=2
N.assign=1000
```

Looks at the release directory (the directory directly below the section root, see [Sections](Sections.md)) and counts the files each slave holds for that release. The `topslaves` slaves with the most files get `assign` points added. This keeps the files of one release together on a few slaves.

| Key | Required | Meaning |
|-----|----------|---------|
| `topslaves` | yes | how many slaves get the bonus |
| `assign` | yes | the bonus. With `assign=0` the slaves outside the top are **removed** from the table instead |

For a file directly in the section root directory, or in `/`, there is no release and the scores are left unchanged. With the `conf` section manager a path outside every configured section counts as a section rooted at `/`, so its top-level directory is treated as the release. If the per-slave counts ever look wrong, `SITE FIXSLAVECOUNT` rebuilds them.

### stripefiles

```
N.filter=stripefiles
```

No settings. The opposite of `slavetop`: it counts the files of the same directory held by each slave, and removes the slaves that hold more than the slave with the fewest. Files of a release end up spread over all slaves.

### jobpoints (Archive plugin)

```
N.filter=jobpoints
N.assign=50000
```

Only meaningful in `slaveselection-jobup.conf` and only with the [Archive](Archive.md) plugin loaded. Adds `assign` points to every slave that is the destination of a job created by a running archive of the section the file is in. This pushes the archive jobs to the slaves already chosen. It is commented out in the default file.

### directoryspace (Archive plugin)

```
N.filter=directoryspace
N.assign=1000000
```

Only with the [Archive](Archive.md) plugin loaded. Subtracts `assign` points from a slave when its free disk space (plus the part of the archived directories it already holds) is smaller than the total size of the directories being archived in the section. It is commented out in the default file.

## The assign syntax

`matchdir` and `matchdirex` use an `assign` list, and `maxtransfersperslave` uses similar entries. Entries are separated by spaces (commas are also allowed and ignored):

| Entry | Meaning |
|-------|---------|
| `slave1+500` | add 500 points to slave1 |
| `slave1-500` | subtract 500 points from slave1 |
| `slave1+remove` or `slave1-remove` | remove slave1 from the table |
| `all+500` / `all-500` | add or subtract 500 points for **every** slave in the table |
| `all+remove` / `all-remove` | remove every slave (the transfer fails if the filter applies) |

Rules:

- Entries are applied **in order**. A slave removed by an earlier entry cannot come back through a later one.
- A slave name can contain `-` or `+`: `slave-1-500` is slave `slave-1` with -500. The score (or `remove`) after the last `+` or `-` is mandatory, otherwise `slave-1` would be read as slave `slave` with -1.
- `all` is a reserved name, so do not call a slave `all`.
- A slave in `assign` that does not exist is an error when the config is loaded. A slave that exists but is not a candidate for this transfer (offline, does not have the file) is skipped silently.

## Ties

When several slaves share the highest score after the last filter:

1. the one that was used **least recently** for this direction wins;
2. if that is also tied (for example right after a restart), one of them is chosen **at random**.

Selections on a chain are handled one at a time and each pick is recorded, so transfers that start at the same moment do not all land on the same idle slave. The `cycle` filter adds one more point to the least recently used slave of each group of equal scores, which matters when a later filter would otherwise treat the groups differently.

## Examples

### Keep a section on specific slaves

Uploads to `/ARCHIVE` may only go to `slave1` or `slave2`; any other slave is removed.

```
1.filter=matchdir
1.match=/ARCHIVE/*
1.assign=slave1+0 slave2+0
1.assume.remove=true

2.filter=bandwidth
2.multiplier=1

3.filter=cycle
```

### Prefer one slave for a section

`slave3` gets 1,000,000 points for everything under `/0DAY`, so it is used unless a later filter removes it.

```
1.filter=matchdir
1.match=/0DAY/*
1.assign=slave3+1000000

2.filter=bandwidth
2.multiplier=1

3.filter=cycle
```

### Never use a slave for one section

```
1.filter=matchdir
1.match=/PRIVATE/*
1.assign=slave4+remove
```

### Everything except one section

`negate.expression` flips the match, so this penalises `slave4` for everything outside `/PRIVATE`:

```
1.filter=matchdir
1.match=/PRIVATE/*
1.negate.expression=true
1.assign=slave4-100000
```

### Keep a release together

```
1.filter=slavetop
1.topslaves=1
1.assign=100000

2.filter=bandwidth
2.multiplier=1

3.filter=cycle
```

### Spread the files of a release over all slaves

```
1.filter=stripefiles

2.filter=bandwidth
2.multiplier=1

3.filter=cycle
```

### Limit transfers per slave

At most 10 transfers on `slave1` and 5 on `slave2`, for every path:

```
1.filter=MaxtransfersPerslave
1.assign=slave1+10 slave2+5
1.match=.*

2.filter=bandwidth
2.multiplier=1

3.filter=cycle
```

## Testing a chain with SITE SLAVESELECT

```
SITE SLAVESELECT <TYPE> </FULLPATH>
```

`<TYPE>` is `up`, `down`, `jobup` or `jobdown`. `</FULLPATH>` must start with `/` (the file does not have to exist). The command runs the filters of that chain against all online slaves and prints the score of each as `slave=score`. A slave that a filter removed is left out of the list. By default it needs the `siteop` permission (see [Permissions](Permissions.md) and [Site Commands](Site-Commands.md)). The test runs without a user or client address.

Example:

```
SITE SLAVESELECT up /MP3/Some.Release-GRP/file.mp3
```

The command also exists in the IRC command set as `slave-select` (see [IRC Commands](IRC-Commands.md)).

## Reloading

`SITE RELOAD` reloads the slave selection files together with the other config files. If any of the four files has an error, the **previous chains stay active** and the error is logged, so a typo does not take the site down. At start-up, in contrast, an error is fatal.
