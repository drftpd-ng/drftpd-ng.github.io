> Applies to DrFTPD 4.0.x.

# Nukefilter

The `nukefilter` plugin checks the **name** of every new directory and nukes it, after a short warning time, if the name breaks your rules: a banned word, a missing required word, a banned or not allowed year, a banned or not allowed group. It is for site-wide naming rules ("no `.PROPER.` in MP3", "only releases from 1998 to 2010", "never group X").

It works on directory names only. To nuke releases because of their content (incomplete, empty, missing files, IMDB data), see [Autonuke](Autonuke.md), which also explains what a nuke does and the `SITE NUKE` commands.

See [Plugins](Plugins.md) for how plugins are loaded.

## How it works

- When a user creates a directory (`MKD`) the plugin checks its name. Nothing happens on later renames or on directories created by other means.
- Directory names that match `nuke.exempt.dirnames.regex` (by default `CD1`, `DVD2`, `Sample`, `Subs`, `Proof`, `Cover` and similar) are never checked.
- The **global** rules are checked first. If the directory passes them and its section has its own rules, the **section** rules are checked as well. The first rule that fails stops the checks.
- When a rule fails, the directory is announced and a nuke is scheduled `nuke.delay` seconds later. This is the time the uploader has to remove the directory. Right before the nuke the plugin skips directories that are already nuked or that were approved with `SITE APPROVE`.
- The nuke is a normal nuke (see [Autonuke](Autonuke.md#what-a-nuke-does)) by the user in `nuke.nuker`, with the multiplier ("nukex") of the rule.

The order of the checks in each group: banned strings, required strings, banned regexes, required regexes, banned years, required years, banned groups, required groups.

## Configuration

`config/plugins/nukefilter.conf` on the master (copy from `nukefilter.conf.dist`). It is reloaded with `SITE RELOAD`.

### General keys

| Key | Default | Meaning |
|-----|---------|---------|
| `nuke.nuker` | `drftpd` | the account that nukes. It **must exist**. |
| `nuke.exempt.dirnames.regex` | empty | regex(es) for directory names that are never checked. Several regexes separated by `;`. The whole name has to match. |

### Global rules

Global rules apply to every section.

| Key | Meaning |
|-----|---------|
| `global.enabled` | `true` / `false`. Turns the global rules on or off (default `true`). Section rules are not affected. |
| `global.filter.string` | banned words. If the name **contains** one (case-insensitive) it is nuked. |
| `global.enforce.string` | required words. If the name **lacks** one of them (case-insensitive), it is nuked. |
| `global.filter.regex` | banned regexes. If the name matches one, it is nuked. Case-sensitive unless you use `(?i)`. |
| `global.enforce.regex` | required regexes. If the name fails to match one of them, it is nuked. |
| `global.filter.year` | banned years or ranges. |
| `global.enforce.year` | allowed years or ranges. A name with a year outside all of them is nuked. |
| `global.enforce.year.nukex` | nuke multiplier for a failed `enforce.year` check (default `3`) |
| `global.filter.group` | banned groups. A name that **ends** with one is nuked (case-insensitive). |
| `global.enforce.group` | required groups. A name that does not end with one of them is nuked. |
| `global.enforce.group.nukex` | nuke multiplier for a failed `enforce.group` check (default `3`) |
| `global.nuke.delay` | seconds until the nuke (default `120`) |
| `global.exempt.sections` | sections that the global rules skip, separated by `;`, **case-sensitive** |

Lists use `;` as the separator. Where a rule says "nukex", add `~<multiplier>` to the item to set the multiplier of that item, for example `bannedstr~7`. Without it the multiplier is `3`. Required lists (`enforce.string`, `enforce.regex`) accept `~<multiplier>` per item too, but `enforce.year` and `enforce.group` do not: use the `.nukex` keys.

In regexes the backslash must be doubled: write `\\d` for `\d`.

Years: the plugin takes the **last** four-digit group of the form `dddd` or `ddd` followed by a letter `x` (like `199x`) from the name. If there is none, the year rules are skipped. Items are single years (`1999`) or ranges (`2004-2008`; the range may be written in either order, `2003-2000`). For `199x` the years 1990 to 1999 are each checked, and the name is nuked if any of them is banned.

### Section rules

For each section, add a numbered block starting at `1` (the plugin stops at the first number without `section`). It uses the same keys, without the `global.` prefix:

```
1.section=MP3
1.filter.string=...
1.enforce.string=...
1.filter.regex=...
1.enforce.regex=...
1.filter.year=...
1.enforce.year=...
1.enforce.year.nukex=...
1.filter.group=...
1.enforce.group=...
1.enforce.group.nukex=...
1.nuke.delay=...
```

The section name is case-sensitive (see [Sections](Sections.md)). Section rules are only used when the global rules did not already fail the directory. Sections do not inherit from the global rules; they are checked in addition.

The shipped file has the global switch on and everything else commented out, so after copying it nothing is filtered until you add rules.

### Example

```
nuke.nuker=drftpd
nuke.exempt.dirnames.regex=^(?i)(((cd|dvd|dis[ck]|part)[1-9]{1,2})|cover[s]?|sample|subs|proof)$

global.enabled=true
global.filter.string=bannedstr~7;unsexy;pretest~5
global.filter.group=BADGRP~5;WORSEGRP~10
global.enforce.year.nukex=3
global.enforce.group.nukex=3
global.nuke.delay=120
global.exempt.sections=REQUESTS;ARCHIVE;SPEEDTEST;PRE

1.section=MP3
1.filter.year=2004-2008
1.enforce.group=COOLGRP;COOLGRP_iNT;NICEGRP
1.enforce.group.nukex=5
1.nuke.delay=300
```

## Announces

When a rule fails, the plugin announces under the event name `nukefilter` (see [Sitebot](Sitebot.md)). The messages are in `config/themes/irc/nuke.filter.announcer.theme.default`, one key for each rule and scope:

- `global.filter.string.announce`, `global.enforce.string.announce`, `global.filter.regex.announce`, `global.enforce.regex.announce`, `global.filter.year.announce`, `global.enforce.year.announce`, `global.filter.group.announce`, `global.enforce.group.announce`
- the same with `section.` instead of `global.`

Variables: `${element}` (the string, regex, year or group that failed), `${path}` (the parent directory), `${dir}` (the directory name), `${section}`, `${delay}` (seconds until the nuke), `${nukex}` (multiplier).

The announce goes to the channels set for the `nukefilter` event. The shipped `irc.announce.conf.dist` does not list `nukefilter`; the same path and section pattern that the other events use is read for it:

```
nukefilter.path.1=/path/subpath/*
nukefilter.path.1.destination=public
nukefilter.section.1=*
nukefilter.section.1.destination=public
```

When the nuke itself happens, a normal `nuke` announce follows (see [Autonuke](Autonuke.md)).

## Commands

The plugin has no commands of its own.
