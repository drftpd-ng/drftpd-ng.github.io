> Applies to DrFTPD 4.0.x.

# Statistics, credits and top lists

This page covers two connected things:

1. the `statistics` **plugin**, which does the accounting after each transfer (bytes, files, credits) and adds the `perms.conf` lines `creditcheck`, `creditloss`, `creditlimit`, `nostatsup` and `nostatsdn`;
2. the **top lists** (`SITE ALUP`, `SITE WKUP`, ...), `SITE STATS` and `SITE TRAFFIC`. The commands are part of the core and always available, but their output format ships with the statistics plugin, and the numbers they show are the ones the plugin counts.

See [Plugins](Plugins.md) for how plugins are loaded. The plugin has no config file of its own and no announces.

## What the plugin does

After each transfer it updates the user:

- **Upload (`STOR`)**: the user gets credits for the bytes uploaded (bytes times the `creditcheck` ratio, by default the user's ratio), and the upload counters go up (files, bytes, time) for all-time, month, week and day.
- **Download (`RETR`)**: the user loses credits for the bytes actually downloaded (bytes times the `creditloss` ratio, by default 1), and the download counters go up.
- **Delete (`DELE`) of a file**: the credits and upload counters that file gave to its uploader are taken back. The bytes are only taken out of the periods (month, week, day) in which the file was uploaded, and never below zero. The same happens when the site itself removes an upload that does not match its SFV (see [Zipscript](Zipscript.md)).
- **Login**: the last seen time is updated and the user's login counter goes up.

Before a download or upload it also enforces:

- `creditlimit` (below): refuses the transfer when the user is over the limit.
- On downloads, enough credits: if the file costs more credits than the user has, the download is refused with "Not enough credits". Users with ratio `0` (leech) are never charged and are never refused for credits.

The first day of the week for the week counters is Sunday, or Monday if `european.cal=true` in `master.conf` (see [Master Configuration](Master-Configuration.md)).

## Lines in perms.conf

These lines go in `config/perms.conf` (see [Permissions](Permissions.md)). They are only recognised when the statistics plugin is loaded. Lines are checked top to bottom and the first one that matches the path and the user is used. `<=group/-user>` stands for the usual list of permissions (`=group`, `-user`, `!` to negate, `*` for everyone).

| Line | Syntax | Meaning |
|------|--------|---------|
| `creditcheck` | `creditcheck <path> <ratio> <=group/-user>` | credit ratio for **uploads** in the path. `0` = no credits for uploads there. Without a matching line, the user's own ratio is used. |
| `creditloss` | `creditloss <path> <ratio> <=group/-user>` | credit ratio for **downloads** in the path. `0` = free downloads. Without a matching line, ratio 1 is used. Users with ratio `0` are never charged whatever the lines say. |
| `creditlimit` | `creditlimit <path> <direction> <period> <bytes> <=group/-user>` | limit transfers by amount over time. `direction` is `UP` or `DN`, `period` is `ALL`, `DAY`, `WEEK` or `MONTH`, `bytes` is a size such as `50GB`. When a user has already transferred **more than** that amount in the period, further transfers in that direction in the path are refused with "Credit limit for period exceeded". |
| `nostatsup` | `nostatsup <path> <=group/-user>` | uploads in the path do not count in the upload counters (credits are still given according to `creditcheck`) |
| `nostatsdn` | `nostatsdn <path> <=group/-user>` | downloads in the path do not count in the download counters |
| `hideinstats` | `hideinstats <=group/-user>` | users matching this are left out of the top lists. Shipped: `=deleted =exempt`. This line is part of the core. |

Examples:

```
# Uploads and downloads in /PRE do not count
nostatsup   /pre/*   *
nostatsdn   /pre/*   *

# Uploading in /REQUESTS gives no credits, downloading there is free
creditcheck /REQUESTS/*   0   *
creditloss  /REQUESTS/*   0   *

# Everybody gets 3x credits for uploads in /0DAY
creditcheck /0DAY/*   3   *

# At most 200 GB of downloads a week for everyone outside group vip
creditlimit /*   DN   WEEK   200GB   !=vip
```

A line that cannot be parsed (for example an unknown direction or period) is logged and skipped.

## Top lists

All of these default to permission `*` (everyone) and are defined in `config/commands/ftp/core.conf` (see [Site Commands](Site-Commands.md)). IRC names are the lower case words (`alup`, `wkup`, ... see [IRC Commands](IRC-Commands.md)).

| Command | Shows |
|---------|-------|
| `SITE ALUP` / `SITE ALDN` | top uploaders / downloaders of all time |
| `SITE MONTHUP` / `SITE MONTHDN` | the month |
| `SITE WKUP` / `SITE WKDN` | the week |
| `SITE DAYUP` / `SITE DAYDN` | the day |
| `SITE GALUP` / `SITE GALDN` | top uploading / downloading **groups** of all time |
| `SITE GMONTHUP` / `SITE GMONTHDN` | groups, the month |
| `SITE GWKUP` / `SITE GWKDN` | groups, the week |
| `SITE GDAYUP` / `SITE GDAYDN` | groups, the day |
| `SITE STATS <user>` | one user's files, bytes and rank for all-time, month, week and day |
| `SITE TRAFFIC` | summary of the site's uploads and downloads (totals and averages for all-time, month, week and day) |

Arguments of the user lists (`ALUP` ... `DAYDN`):

```
SITE ALUP [count] [<=group/-user>]
```

- `count`: how many users to show (default 10).
- an optional permission list limits the list to those users, for example `SITE WKUP 5 =vip`.

The group lists accept a `count` (default 10).

Users that match `hideinstats` in `perms.conf` are left out. Deleted users are left out as well by the shipped setting.

`SITE TOP` and `SITE GTOP` in the shipped command file are not transfer top lists: they belong to the [Trial](Trial.md) plugin.

### When the periods reset

The day, week and month counters are reset by the master at the start of each period (see `european.cal` above for the week). The all-time counters are never reset. The `SITE ALUP` family reads the live counters, so after a reset the day, week or month list starts empty.

### Output format

The line formats of these commands are in `config/themes/ftp/statistics.theme.default` and `config/themes/irc/statistics.theme.default`. Each list has a `<name>.header`, a `<name>` (one line per user) and a `<name>.footer` key. For example:

```
alup=[${pos,2}] ${user,-12} ${tagline,-24} ${upfiles,5} ${upbytes,11} ${uprate,8}
```

Variables: `${pos}`, `${user}`, `${tagline}`, files, bytes and rate for the period (`upfiles`, `upbytes`, `uprate`, `dnfiles`, `dnbytes`, `dnrate`, and with the suffix `month`, `week` or `day` for the period lists). The group lists use `morestats.grpstats` with `${pos}`, `${grp}`, `${files}`, `${megs}`; `SITE TRAFFIC` uses `morestats.trafficstats`; `SITE STATS` uses the multi-line `stats` key.

The header and footer of a list can also come from text files in `config/themes/text/` (for example the ASCII-art banner shipped as `alup_header.txt.dist`). For the user lists the text file is used only when the theme has **no** `<name>.header` / `<name>.footer` key; the shipped theme defines them empty, so delete those lines to use the text files. The group lists and `SITE TRAFFIC` always read their header and footer from text files. The files are `alup`, `aldn`, `monthup`, `monthdn`, `wkup`, `wkdn`, `dayup`, `daydn`, `galup`, `galdn`, `gmonthup`, `gmonthdn`, `gwkup`, `gwkdn`, `gdayup`, `gdaydn` and `traffic`, each with `_header.txt.dist` and `_footer.txt.dist`. Copy one without the `.dist` to use it (see [Themes](Themes.md)).

## Daily, weekly and monthly reports on IRC

The IRC reports ("top 5 uploaders of the day") belong to the [Sitebot](Sitebot.md) plugin, not to this one. Their config file is `config/plugins/dailystats.conf`:

```
showzero=true
outputnum=5
exempt=
```

| Key | Default | Meaning |
|-----|---------|---------|
| `showzero` | `true` in the shipped file | also list users with 0 bytes |
| `outputnum` | `5` | users per report |
| `exempt` | empty | users or groups, separated by spaces, left out of the reports |

The report is made just before the day, week or month counters are reset. The announce events are `dailystats.dayup`, `dailystats.daydn`, `dailystats.wkup`, `dailystats.wkdn`, `dailystats.monthup` and `dailystats.monthdn`. See [Sitebot](Sitebot.md) for channel settings.
