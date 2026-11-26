> Applies to DrFTPD 4.0.x.

# Mediainfo

The mediainfo plugin reads technical information (container, video, audio and subtitle streams) from video files that users upload, checks whether the file looks complete, announces the result on IRC and can show it as an entry in directory listings. It can also nuke a release automatically when the audio language is not allowed.

## Requirements

The plugin uses the external `mediainfo` command line program (MediaInfo CLI). It must be installed on **every slave** that should handle these files, and it must be found in the `PATH` of the slave process. The slave plugin checks this at start up by running `mediainfo --version`; if that fails the plugin handler on that slave does not initialise ("Failed to initialize mediainfo plugin" is logged).

Install it with your package manager, for example `apt install mediainfo` on Debian or Ubuntu.

The program is only run on slaves. The master asks the slave that holds the uploaded file, so the slave with the file must be online.

## What it does

When a file with one of the configured extensions has been uploaded (a successful `STOR`), the slave runs `mediainfo` on it and the master:

1. Stores the result in the virtual file system for that file.
2. Sends an announce to IRC (type `mediainfo`).
3. Optionally checks the audio language and nukes the release (see below).

Sections listed in `sections.exclude` are skipped.

The information that is kept is the General, Video, Audio and Text (subtitle) blocks of the `mediainfo` output. Each value is available under its `mediainfo` name in the theme, with a prefix: `v_` for video, `a_` for audio and `s_` for subtitles (for example `${v_Width}`, `${a_Format}`). Only the first video stream and the first audio stream are shown in the announce; the audio and subtitle language lists include all streams.

### Integrity check

The plugin also tries to detect uploads that are incomplete or renamed:

| Format | What is checked |
|--------|-----------------|
| MP4 | The size computed from the MP4 structure is compared with the real file size. A difference marks the sample as bad. If the file cannot be parsed the check is skipped. |
| MKV | If `mediainfo` reports that the file is smaller than expected, or reports conformance errors, the sample is marked bad. **The slave deletes such an MKV file immediately.** |
| AVI | The size of the video and audio streams is compared with the file size. An AVI with no readable audio or video stream is marked unreadable. |
| any | If the container type reported by `mediainfo` differs from the file extension (for example an MKV renamed to `.mp4`), the sample is marked bad and the announce shows the real and the claimed format. |

The announce shows the outcome as `-OK-`, `-iNC-` (expected size versus received size), `-CORRUPT-` (could not verify) or `-RENAME-`.

## Configuration

The file is `config/plugins/mediainfo.conf` (copy it from `mediainfo.conf.dist`). Changes are picked up on `SITE RELOAD`.

| Key | Default | Meaning |
|-----|---------|---------|
| `sections.exclude` | `PRE` | Sections (separated by space) where the plugin does nothing. |
| `extensions` | empty | Extensions (separated by space, lower case, no dot) of the files to examine, for example `avi mkv mp4`. **Empty by default, which means the plugin is inactive** until you set it, because it needs the external program. |
| `mediabar.enabled` | `false` | Show the media bar in directory listings. |
| `mediabar.directory` | `true` | Media bar entries are directories (`true`) or files (`false`). |

Example, enabling the plugin for video files:

```
extensions=avi mkv mp4
mediabar.enabled=true
```

Announces and the media bar have a theme only for `avi`, `mkv`, `mp4` and `m2ts` files. Other extensions that you add to `extensions` are examined, but nothing is announced for them.

### Audio language nuke

The audio language nuke is off by default. It nukes a release when none of the audio streams of its sample has an allowed language. It only works for the sections where you set `<section>.audio.lang.allowed`.

| Key | Default | Meaning |
|-----|---------|---------|
| `<section>.audio.lang.allowed` | not set (check disabled) | Allowed languages for this section, separated by comma or semicolon. The section name is not case sensitive. |
| `<section>.audio.lang.nuke.multiplier` | `3` | Nuke multiplier. |
| `<section>.audio.lang.unknown` | `allow` | What to do when no audio stream has a language tag, or the sample has no audio: `allow` or `nuke`. |
| `audio.lang.nuker` | `drftpd` | The site user that performs the nuke. **This user must exist.** |
| `audio.lang.exempt.dirnames.regex` | empty | Release directory names matching any of these regular expressions (separated by semicolon) are never nuked. |

Notes:

- The comparison is against the `Language` field of the audio streams as printed by `mediainfo`. Allowed values can be English names (`German`) or ISO 639-1 / 639-2 codes (`de`, `ger`, `deu`); they are treated as the same language. A qualified value like `English (US)` or `en-US` matches by its primary part only, so `us` does not match `English (US)`. A value that is not a known language is logged as a warning and only matches the exact text.
- The check only runs when the sample itself is OK (a bad or unreadable sample is not used to judge the language).
- The release that is nuked is the parent of the directory when the sample was uploaded in a directory named `Sample`, otherwise the directory itself. Approved releases (see `SITE APPROVE`), directories that are already nuked and the site root are skipped.
- The nuke is announced like any other nuke (see [Sitebot](Sitebot.md) and [Nukefilter](Nukefilter.md) for related automatic nuking).

Example for a German TV section:

```
TV-DE.audio.lang.allowed=German,English
TV-DE.audio.lang.nuke.multiplier=3
TV-DE.audio.lang.unknown=allow
audio.lang.nuker=drftpd
audio.lang.exempt.dirnames.regex=
```

## Commands

The plugin has no commands.

## IRC announce

Announce type: `mediainfo`. It is path based, so it can be routed by path or by section in `irc.announce.conf`, with a fallback to the `default` rules (see [Sitebot](Sitebot.md)):

```
mediainfo.path.1=/path/subpath/*
mediainfo.path.1.destination=public
mediainfo.section.1=*
mediainfo.section.1.destination=public
```

Theme: `config/themes/irc/mediainfo.announcer.theme` (see [Themes](Themes.md)). For each of `mkv`, `m2ts`, `mp4` and `avi` there is a header line and a line each for video, audio and subtitles, for example:

```
mediainfo.mkv=${bold}[SAMPLE]${bold} ${filepath} :: ${sample_ok}
mediainfo.mkv.video=${bold}-VIDEO-${bold} ${v_Format} : ${v_Width}x${v_Height} @ ${v_Bit rate} : ${v_Frame rate} : ${v_Display aspect ratio} : ${v_Format profile} : ${v_Duration}
mediainfo.mkv.audio=${bold}-AUDIO-${bold} ${a_Languages} : ${a_Format} @ ${a_Bit rate} ${a_Bit rate mode} : ${a_Sampling rate} : ${a_Channel(s)}
mediainfo.mkv.sub=${bold}-SUB-${bold} ${s_Languages}
```

The result texts are `mediainfo.ok`, `mediainfo.nok`, `mediainfo.unreadable` and `mediainfo.type.missmatch`.

Tags available in the announce: `filepath`, `filename`, `dirpath`, `dirname`, `section`, `sectioncolor`, `sample_ok`, `real_filesize`, `cal_filesize`, `real_format`, `renamed_format`, the `v_*`, `a_*` fields (any field name that `mediainfo` printed, with spaces kept, such as `${v_Bit rate}`), `a_Languages` and `s_Languages`. The subtitle line is only printed when the file has subtitle streams.

## Media bar

With `mediabar.enabled=true` the directory listing shows up to three extra entries for each examined file: one for video, one for audio and one for subtitles. The text comes from `config/themes/ftp/mediainfo.list.theme` (keys `mkvbar.*`, `m2tsbar.*`, `mp4bar.*`, `avibar.*`). The values are shortened for use in a file name (spaces and the word "pixels" are removed, "channels" becomes "ch"). Example from the default theme:

```
mp4bar.video=[VIDEO]-${v_Format}_${v_Width}x${v_Height}@${v_Bit rate}_${v_Frame rate}_${v_Display aspect ratio}_${v_Format profile}_${v_Duration}-[VIDEO]
```

## Related pages

- [Zipscript](Zipscript.md), [Themes](Themes.md), [Sitebot](Sitebot.md), [Sections](Sections.md), [Adding-Slaves](Adding-Slaves.md)
