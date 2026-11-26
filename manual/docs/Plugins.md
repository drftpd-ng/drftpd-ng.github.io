> Applies to DrFTPD 4.0.x.

# Plugins

Most of the features beyond basic FTP in DrFTPD come as plugins: the zipscript, the IRC bot, archiving, nuking, search and more. This page lists all plugins that are in the 4.0.x source tree, explains how they are switched on and off, and where their configuration is.

## Contents

- [The plugins](#the-plugins)
- [How plugins are enabled and disabled](#how-plugins-are-enabled-and-disabled)
- [What needs what](#what-needs-what)
- [Where the configuration is](#where-the-configuration-is)

## The plugins

There are 22 plugin modules under `src/plugins`. Some have a part that runs on the slaves too.

| Plugin | Runs on | What it does | Page |
|--------|---------|--------------|------|
| `sitebot` | master | the IRC bot: announces, IRC commands, channel access; also the SysOp log and the daily/weekly/monthly stats reports on IRC | [Sitebot](Sitebot.md) |
| `zipscript` | master and slaves | SFV, NFO, DIZ and ZIP handling, race stats, completion bars and checks for incomplete releases | [Zipscript](Zipscript.md) |
| `raceleader` | master | announces when a new user takes the lead in a race | [Race Leader](Race-Leader.md) |
| `mediainfo` | master and slaves | reads media information (resolution, codecs, audio, ...) with the MediaInfo tool and shows it in the release | [Mediainfo](Mediainfo.md) |
| `imdb` | master and slaves | IMDB information for movie releases | [IMDB and TVMaze](IMDB-and-TVMaze.md) |
| `tvmaze` | master | TVmaze information for TV releases | [IMDB and TVMaze](IMDB-and-TVMaze.md) |
| `links` | master | creates link directories such as "latest races", "no NFO" or "incomplete" | [Links](Links.md) |
| `dupecheck` | master | refuses uploads and directories that already exist on the site | [Dupecheck](Dupecheck.md) |
| `request` | master | the request system (`SITE REQUEST`, `REQFILLED`, ...) | [Requests](Requests.md) |
| `find` | master | `SITE FIND`, `SITE SEARCH`, `SITE DUPE` using the file index | [Find](Find.md) |
| `jobs` | master | the queue of slave-to-slave copy jobs | [Jobs](Jobs.md) |
| `archive` | master | moves or copies old releases to other slaves and directories, on a timer | [Archive](Archive.md) |
| `mirror` | master | keeps extra copies of uploaded files on other slaves | [Mirror](Mirror.md) |
| `autofreespace` | master | deletes the oldest releases when a slave is low on disk space or by age | [Autofreespace](Autofreespace.md) |
| `autonuke` | master | nukes incomplete, empty or otherwise bad releases automatically | [Autonuke](Autonuke.md) |
| `nukefilter` | master | nukes directories with banned or missing words, years or groups in the name | [Nukefilter](Nukefilter.md) |
| `traffic` | master | minimum and maximum transfer speed per user and path, with kick and ban | [Traffic](Traffic.md) |
| `statistics` | master | upload and download accounting, credits and `creditcheck`, `creditloss`, `creditlimit` lines | [Statistics](Statistics.md) |
| `trial` | master | upload quota trials for new users | [Trial](Trial.md) |
| `prebw` | master | measures and announces the site bandwidth after a pre | [PreBW](PreBW.md) |
| `speedtest` | master | announces the speed of test uploads to the site | [Speedtest](Speedtest.md) |
| `speedtest-net` | master and slaves | tests slave bandwidth against speedtest.net servers | [Speedtest](Speedtest.md) |

Notes:

- The `imdb`, `mediainfo`, `speedtest-net` and `zipscript` modules each consist of three jars: `*-common`, `*-master` and `*-slave`. The other plugins are master only.
- The file index, nuke commands, `SITE ALUP` and the other top lists, `SITE APPROVE`, the user and group commands and the SSL/TLS support are **not** plugins. They are part of the core and always available. See [Index and Search](Index-and-Search.md), [Autonuke](Autonuke.md#nuke-commands), [Statistics](Statistics.md), [Users and Groups](Users-and-Groups.md) and [SSL/TLS](SSL-TLS.md).

## How plugins are enabled and disabled

There is no plugin list in a configuration file and no `SITE` command to load or unload a plugin. A plugin is **on when its jar file is in the `build` directory** of the master (or slave) and **off when it is not**. At start-up the master looks at every class on its class path and starts each plugin it finds.

### Where the jars come from

The build (`mvn install`, see [Installation (Linux)](Installation-Linux.md)) compiles every module in `src/plugins/pom.xml` and puts the results in the `runtime` directory:

| Directory | Content |
|-----------|---------|
| `runtime/master/build` | the `*-master-*.jar` and `*-common-*.jar` of every plugin, and the core jars |
| `runtime/master/lib` | the libraries |
| `runtime/slave/build` | the `*-slave-*.jar` and `*-common-*.jar` of the plugins that have a slave part, and the core slave jars |
| `runtime/slave/lib` | the libraries |

`master.sh` and `slave.sh` start Java with the class path `lib/*:build/*`, so every jar in `build` counts. The jar names look like `drftpd-archive-master-4.0.12-SNAPSHOT.jar` (the artifact id is `drftpd-<plugin>-master`, for example `drftpd-jobs-master`, `drftpd-speedtestnet-master`).

### Switching a plugin off

Pick one:

1. **Delete the jar** from `runtime/master/build` (and `runtime/slave/build` for plugins with a slave part), then restart. Do this on the machine where you run the master; a new build puts the jar back.
2. **Leave the module out of the build**: remove its `<module>` line from `src/plugins/pom.xml` before running `mvn install`. This keeps the jar out of `runtime` for good. Plugins that need it (see below) will no longer compile, so remove those as well.

A change needs a restart of the master (and the slaves, for slave jars). `SITE RELOAD` only re-reads configuration.

### Switching a plugin on

Make sure its jar is in `runtime/master/build` (build it again if you removed it), keep its `.dist` config files in place, and restart. Check the master log: for every plugin that started you will see a line `Loaded plugin org.drftpd....`.

### Leftover files

- The config files of a plugin you removed (`config/plugins/<name>.conf[.dist]`, the command files and the themes) can stay. A command whose class is not found is skipped when the commands are loaded, with an `info` line in the log. If you prefer a clean `config/commands/ftp` and `config/commands/irc`, delete that plugin's command files.
- A `perms.conf` line that belongs to a plugin (for example `creditcheck` of [Statistics](Statistics.md)) has no effect without the plugin (the plugin is what reads that line).

### Removing plugins that others need

Some plugins can only start when others are present (see the next section). If a jar that another plugin needs is missing, the master logs errors about missing classes when it loads that plugin. Remove plugins together with everything that depends on them.

## What needs what

The table is taken from the build files: a plugin needs the plugins listed for its jars to load. Almost everything that announces needs `sitebot`.

| Plugin | Needs |
|--------|-------|
| `sitebot` | nothing |
| `zipscript` | `sitebot` |
| `raceleader` | `zipscript`, `sitebot` |
| `mediainfo` | `sitebot` |
| `imdb` | `find`, `sitebot` (`find` needs `jobs` and `zipscript`) |
| `tvmaze` | `find`, `sitebot` |
| `links` | `zipscript` |
| `dupecheck` | nothing |
| `request` | `sitebot` |
| `find` | `jobs`, `zipscript` |
| `jobs` | nothing |
| `archive` | `jobs`, `zipscript`, `sitebot` |
| `mirror` | `jobs` |
| `autofreespace` | `sitebot` |
| `autonuke` | `zipscript`, `imdb`, `sitebot` |
| `nukefilter` | `sitebot` |
| `traffic` | `sitebot` |
| `statistics` | nothing |
| `trial` | `sitebot` |
| `prebw` | `sitebot` |
| `speedtest` | `sitebot` |
| `speedtest-net` | nothing (its own common jar on both sides) |

The easiest rule: keep `sitebot`, `zipscript` and `jobs` unless you have removed everything on this table that uses them.

The core command file `config/commands/ftp/core.conf` already refers to commands of the `trial` plugin (`SITE TOP`, `SITE CUT`, `SITE PASSED`, ...), so those commands answer "not implemented" if `trial` is removed.

## Where the configuration is

All paths are relative to the master directory (`runtime/master` after a build). Files that ship as `*.conf.dist` are used as they are; copy one to the same name without `.dist` only if you change it. The `.conf` wins when both exist, and the `.dist` is read when there is no `.conf`. See [Master Configuration](Master-Configuration.md).

| What | Where |
|------|-------|
| Plugin settings | `config/plugins/<plugin>.conf` (`.dist` shipped) |
| FTP commands of the plugin (names, permissions, help) | `config/commands/ftp/<plugin>.conf` |
| IRC commands of the plugin | `config/commands/irc/<plugin>.conf` |
| Texts and announce formats | `config/themes/ftp/`, `config/themes/irc/` (see [Themes](Themes.md)) |
| IRC announce channels per plugin | `config/plugins/irc.announce.conf` (see [Sitebot](Sitebot.md)) |
| Slave-side settings | `config/plugins/<name>.conf` in the **slave** directory (only `speedtest.net.slave.conf` today, see [Speedtest](Speedtest.md)) |
| Slave selection used by jobs and archive | `config/slaveselection-*.conf` (see [Slave Selection](Slave-Selection.md)) |
| Credit and path permission lines of plugins | `config/perms.conf` (see [Permissions](Permissions.md)) |

The plugin settings files:

| Plugin | Settings file(s) in `config/plugins/` |
|--------|---------------------------------------|
| `sitebot` | `irc.conf`, `irc.announce.conf`, `sysop.conf`, `dailystats.conf` |
| `zipscript` | `zipscript.conf` |
| `raceleader` | none (texts only) |
| `mediainfo` | `mediainfo.conf` |
| `imdb` | `imdb.conf` |
| `tvmaze` | `tvmaze.conf` |
| `links` | `links.conf`, `latestdir.conf` |
| `dupecheck` | `dupecheck.conf` |
| `request` | `request.conf` |
| `find` | none (settings are lines in `config/commands/ftp/find.conf`) |
| `jobs` | `jobs.conf` |
| `archive` | `archive.conf` |
| `mirror` | `mirror.conf` |
| `autofreespace` | `autofreespace.conf` |
| `autonuke` | `autonuke.conf` |
| `nukefilter` | `nukefilter.conf` |
| `traffic` | `traffic.conf` |
| `statistics` | none (lines in `config/perms.conf`) |
| `trial` | `trial.conf` |
| `prebw` | `prebw.conf` |
| `speedtest` | `speedtest.conf` |
| `speedtest-net` | `speedtest.net.conf` (master), `speedtest.net.slave.conf` (slave) |

Most plugins read their file again with `SITE RELOAD`.
