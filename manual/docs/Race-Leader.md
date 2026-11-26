> Applies to DrFTPD 4.0.x.

# Race-Leader

The raceleader plugin announces on IRC when somebody takes the lead in a race. It watches uploads in releases that have an SFV and sends a "new race leader" message when a different user becomes the top uploader of the release.

It needs the [Zipscript](Zipscript.md) (it reads the SFV information from it) and the [Sitebot](Sitebot.md) to show the announce.

## Configuration

The plugin has **no configuration file of its own**. There is no `raceleader.conf` and nothing to set in the plugin itself. What you can configure:

- Where the announce goes, with the `store.newraceleader` type in `irc.announce.conf` (see below).
- The text of the announce, in the IRC theme (see below).

## When it announces

On every successful upload, the plugin looks at the release directory:

- The directory must have an `.sfv`, and the SFV must list **more than 5 files**. Smaller releases are ignored, to avoid spamming.
- The uploaded file must be listed in the SFV. Other files (NFO, sample and so on) are not counted.
- The plugin remembers the current leader of each release. The leader is the user with the most bytes uploaded in the release. When the user who just uploaded a file is now first, and is not the user remembered as leader, the announce is sent and the new user is remembered.
- It does nothing until there are **at least two racers** in the release: the first user is just the starting leader.
- It is not sent at the halfway point of the release, because the halfway announce of the zipscript already names the leader.
- When the release is complete, the plugin forgets it.
- Old entries are removed as well: every 30 minutes the plugin drops the releases it started tracking more than 30 minutes ago. If such a race is still running, tracking starts again with the next upload.

The remembered leaders are only kept in memory. After a restart of the master, the plugin starts again for each release with the next upload.

## IRC announce

Announce type: `store.newraceleader`. It is path based and can be routed by path or by section in `irc.announce.conf`, with a fallback to the `default` rules (see [Sitebot](Sitebot.md)):

```
store.newraceleader.path.1=/path/subpath/*
store.newraceleader.path.1.destination=public
store.newraceleader.section.1=*
store.newraceleader.section.1.destination=public
```

If you do not add any rule, the event goes wherever your `default` rules send it.

The text is in `config/themes/irc/new.raceleader.announcer.theme` (see [Themes](Themes.md)). The default:

```
store.newraceleader=${color}15${bold}NEW RACE LEADER${bold}${coloroff} in ${color}15${bold}${section}${bold}${coloroff}: -> ${bold}${dir} : ${prevuser}${bold} has been surpassed by ${bold}${leaduser}${bold} [${size}/${files}F/${percent}%/${speed}/s] with ${bold}${filesleft}${bold} files left.
```

Tags:

| Tag | Meaning |
|-----|---------|
| `section` | The section name. |
| `sectioncolor` | The color of the section (not used by the default text). |
| `dir` | The release path, relative to the section. |
| `path` | The full path of the release. |
| `leaduser` | The new leader. |
| `prevuser` | The previous leader. |
| `size` | The bytes the new leader has uploaded in this release. |
| `files` | The number of files the new leader has uploaded in this release. |
| `speed` | The leader's upload speed. |
| `filesleft` | The number of files still missing in the release. |
| `percent` | See the note below. |

Note: the default text adds `/s` and `%` after `speed` and `percent`, and the speed value already ends in `/s`, so you may want to adjust that. The value of `percent` is calculated from the number of files left and the number of files the leader uploaded, which is not the leader's share of the release; do not rely on it. Remove `${percent}` from your theme if the number looks wrong.

## Related pages

- [Zipscript](Zipscript.md), [Sitebot](Sitebot.md), [Themes](Themes.md)
