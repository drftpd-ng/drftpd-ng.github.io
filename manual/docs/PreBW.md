> Applies to DrFTPD 4.0.x.

# PreBW

The `prebw` plugin measures how much bandwidth the site uses **right after a `SITE PRE`** and announces it on IRC: the speed at several moments, the average and the best speed, how much was leeched, and the top leechers of the release. It is a "pre bandwidth" report: it shows how fast the site's members pulled a new release.

The plugin has no commands. It reacts to `SITE PRE` and to downloads. See [Plugins](Plugins.md) for how plugins are loaded, and [Sitebot](Sitebot.md) for IRC.

## How it works

1. When `SITE PRE` succeeds (see [Site Commands](Site-Commands.md)) in a section listed in `sections`, and the release name does not match `exclude`, the plugin looks at the **size of the release** and picks the entries of `prebw.conf` whose size interval it falls into.
2. For each matching entry it starts a measurement. It waits the first time in `x.times`, takes the speed, waits the next time, takes the speed again, and so on. The speed is either the total upload from slaves to users (all downloads on the site) or, with `real.speed=true`, only the speed of downloads from the pre'd directory.
3. While measuring, every successful download from the pre'd directory is counted per user: files, bytes and speed. This builds the leech top.
4. After the last time the plugin announces the result. The data of the measurement is then dropped.

The measurements run in the background and the `SITE PRE` command is not delayed.

## Configuration

`config/plugins/prebw.conf` on the master (copy from `prebw.conf.dist`). It is reloaded with `SITE RELOAD`; a reload clears measurements that are running.

### Measurement entries

Numbered entries from `1`. The plugin stops at the first number without `times`.

| Key | Meaning |
|-----|---------|
| `x.times` | seconds to wait before each measurement, separated by commas. It is the time **between** measurements: `10,10,40` measures 10 s, 20 s and 60 s after the pre. |
| `x.interval` | the release size, in **MB**, this entry applies to: `<from>:<to>`. The release must be larger than `from` and at most `to`. `*` for `to` means no upper limit; `*` for `from` means all sizes. |
| `x.count.avg` | the average speed is the average of the highest `N` measurements. `*` uses all of them. |

The shipped file has five entries that give more measurements for bigger releases:

```
1.times=30,10,10,10,10,10,10,30
1.interval=0:200
1.count.avg=*
2.times=30,20,20,20,20,20,20,30
2.interval=200:500
2.count.avg=*
3.times=30,30,30,30,30,30,30,30
3.interval=500:1000
3.count.avg=*
4.times=30,30,30,30,30,30,30,30,30,30
4.interval=1000:2000
4.count.avg=*
5.times=30,30,30,30,30,30,30,30,30,30,30,30,30,30,30,30,30,30,30,30
5.interval=2000:*
5.count.avg=*
```

A release belongs to every entry whose interval contains its size, so make the intervals not overlap, or you get one announce per matching entry.

### General settings

```
sections=*
exclude=(?i).+\.(iNTERNAL|iNT)[-_.].+
real.speed=false
prebw.leechtop.count=3
```

| Key | Default | Meaning |
|-----|---------|---------|
| `sections` | none | the sections to measure for, or `*` for all. The comment in the shipped file says "comma separated", but the code splits on `;` (`MP3;XVID`). Use `;` for more than one section. |
| `exclude` | none | regex. Pre'd release names that match it are not measured. The whole name must match. The shipped regex skips internal releases. |
| `real.speed` | `false` | `false`: measure the site's total download speed (all users, all directories). `true`: only count the transfers that are downloading from the pre'd directory. |
| `prebw.leechtop.count` | `3` | the number of top leechers listed in the announce. `0` turns the leech top off. |

## Announce

The event is `prebw`. The text and its formats are in `config/themes/irc/pre.bw.announcer.theme.default`:

| Key | Use |
|-----|-----|
| `prebw.announce` | the whole announce |
| `prebw.bw.format` | one measurement, with `${time}` and `${speed}` |
| `prebw.bw.separator` | text between measurements (spaces are trimmed) |
| `prebw.leechtop.format` | one leecher |
| `prebw.leechtop.empty` | used when nobody downloaded from the release |

Variables of `prebw.announce`: `${section}`, `${sectioncolor}`, `${dir}`, `${bytes}` (total leeched), `${users}` (number of leechers), `${groups}` (number of groups), `${bw}` (all measurements), `${bwavg}`, `${bwtop}`, `${leechtop}`, `${messuretime}` (total seconds measured). Variables of `prebw.leechtop.format`: `${section}`, `${sectioncolor}`, `${dir}`, `${username}`, `${group}`, `${files}`, `${bytes}`, `${avgspeed}`, `${topspeed}`. Leechers are sorted by the bytes they downloaded. (The comment in the shipped theme mentions `bwusertop`; it is not set in 4.0.x.)

To choose the channels, use `irc.announce.conf` (see [Sitebot](Sitebot.md)):

```
#prebw.path.1=/path/subpath/*
#prebw.path.1.destination=public
#prebw.section.1=*
#prebw.section.1.destination=public
```
