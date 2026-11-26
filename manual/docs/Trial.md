> Applies to DrFTPD 4.0.x.

# Trial

The `trial` plugin runs **upload quota competitions** for new users ("trials"). A trial looks at how much each user uploaded in a period (day, week or month). When the period ends, the users who reached the goal get a reward (for example a move into a normal group) and the others are punished (moved to another group, deleted or purged).

The plugin has two trial types:

| Type | Idea |
|------|------|
| `toptrial` | the top N uploaders of the period pass; everyone else fails. Optional minimum amount and minimum percentage of the best uploader. |
| `grouptop` | same settings, but `SITE TOP`, `SITE CUT` and `SITE PASSED` show the result per **group** (average upload per group member). |

You can define several trials in one file, for example a weekly trial for newcomers and a monthly one for another group.

See [Plugins](Plugins.md) for how plugins are loaded. The trial relies on the day, week and month counters of [Statistics](Statistics.md) (the `statistics` plugin).

## Configuration

`config/plugins/trial.conf` on the master (copy from `trial.conf.dist`). It is reloaded with `SITE RELOAD`. There is no trial in the shipped file; all lines are comments.

Trials are numbered from `1`. The plugin stops reading at the first number without a `type`. A trial that has an error (bad name or period) is skipped and the error is logged.

### Keys for all types

| Key | Required | Meaning |
|-----|----------|---------|
| `x.type` | yes | `toptrial` or `grouptop` |
| `x.name` | yes | the name of this trial, shown in the output and in the announces |
| `x.period` | yes | `3` = daily, `2` = weekly, `1` = monthly |
| `x.perms` | no | the users that take part, in permission syntax (for example `=newbies`), see [Permissions](Permissions.md). Deleted users never take part. |
| `x.pass` | no | what to do with a user who passes, see [Actions](#pass-and-fail-actions) |
| `x.fail` | no | what to do with a user who fails |

### Keys for `toptrial` and `grouptop`

| Key | Default | Meaning |
|-----|---------|---------|
| `x.list` | `10` | how many users the `SITE TOP` and `SITE CUT` lists show |
| `x.keep` | `10` | how many users can pass (the top `keep` uploaders) |
| `x.min` | none | minimum upload in **GB** (1 GB = 1,000,000,000 bytes) a user must have in the period |
| `x.percent` | `0` | dynamic quota: a user must also have at least this percent of what the best uploader of the trial uploaded |

A user passes when the upload in the period is more than `x.min`, at least `x.percent` percent of the top uploader, and the user is among the best `keep` uploaders.

### Pass and fail actions

Write the action word and its arguments after `x.pass` / `x.fail`:

| Value | Meaning | Allowed in |
|-------|---------|-----------|
| `chgrp <group> [group ...]` | toggle membership of the groups: a user who is in the group leaves it, one who is not joins | `pass`, `fail` |
| `setgrp <group>` | set the user's primary group | `pass`, `fail` |
| `delete` | mark the user deleted | `fail` only |
| `purge` | delete the user and purge the account | `fail` only |

The shipped comment says that `delete` and `purge` can also be used for `pass`; in 4.0.x they only work for `fail`. A missing or unknown action is only logged and nothing happens to the user.

Group names must exist (see [Users and Groups](Users-and-Groups.md)). For `grouptop` the pass and fail actions are applied to each **user** by their rank, just as for `toptrial`.

### Example

```
# Weekly trial for the group "newbies": the top 5 uploaders who upload at
# least 20 GB and at least 50% of the best uploader move to "members".
1.type=toptrial
1.name=Newbie Weekly
1.period=2
1.perms==newbies
1.list=10
1.keep=5
1.min=20
1.percent=50
1.pass=setgrp members
1.fail=delete
```

## When a trial is run

The trial is run by the master when the matching period resets: at the start of every day for `period=3`, every week for `period=2` and every month for `period=1`. At the start of a month the daily trials run as well. After a trial runs, the result is announced (see below). The upload counters for the period are then reset by the master (see [Statistics](Statistics.md)).

## Site commands

The commands are in `config/commands/ftp/core.conf` (see [Site Commands](Site-Commands.md)); IRC has `top`, `gtop`, `cut`, `passed` and `gpassed` in `config/commands/irc/core.conf` (see [IRC Commands](IRC-Commands.md)). They apply to **all** trials in the file and print one block for each. All of them default to permission `*`, except the IRC `gpassed` which is `=siteop`.

| Command | Meaning |
|---------|---------|
| `SITE TOP [num]` | the ranking of every trial: who is passing, who is failing, how far above or below the quota. `num` is the number of users to list (1 to 100, default `x.list`). |
| `SITE GTOP [num]` | same command as `SITE TOP` |
| `SITE CUT [num]` | only the users who are failing now |
| `SITE PASSED [user]` | whether you (or the named user) are passing, your rank, how much you uploaded, who is ahead of you and how much time is left |
| `SITE GPASSED [group]` | for `grouptop` trials: whether your (or the named) group is passing, with a line per group member |

If no trial is configured the commands answer "No Trial Types Loaded With This Command". Note that `SITE TOP` and `SITE GTOP` show trial rankings; for the ordinary upload and download top lists see [Statistics](Statistics.md) (`SITE ALUP`, `SITE WKUP`, ...).

## Announces

When a trial period ends, [Sitebot](Sitebot.md) announces the result: the trial name, then each user with rank, files, bytes and passed or failed. The event names are `trialmanager.toptrial` and `trialmanager.grouptop`. To send them to certain channels add to `irc.announce.conf`:

```
trialmanager.toptrial.destination=public
trialmanager.grouptop.destination=public
```

The texts are in `config/themes/irc/toptrial.announcer.theme.default` and `config/themes/irc/grouptop.announcer.theme.default`; the command output is in `config/themes/ftp/toptrial.theme.default` and `config/themes/irc/toptrial.theme.default` / `config/themes/irc/grouptop.theme.default` (see [Themes](Themes.md)).
