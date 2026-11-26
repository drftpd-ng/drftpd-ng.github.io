> Applies to DrFTPD 4.0.x.

# Sections

A section is a directory tree that DrFTPD treats as one area of the site, for example `/MP3` or
`/APPS`. Sections matter because several features work per section: `SITE PRE` moves a release into one, `SITE NEW`
lists the newest releases of each one, announcements can be sent to different IRC channels per section and show a
section name and colour, and plugins such as [Autonuke](Autonuke.md) or [Autofreespace](Autofreespace.md) are configured with
section names.

Sections do not give any permissions. Who may upload or download where is set with path rules in
`perms.conf`, see [Permissions](Permissions.md).

## Choosing a section manager

`master.conf` has a setting that selects how sections are defined:

```
# Default: [org.drftpd.master.sections.def.SectionManager]
sectionmanager=org.drftpd.master.sections.def.SectionManager
```

| Value | Behaviour |
| --- | --- |
| `org.drftpd.master.sections.def.SectionManager` | The default. Every directory directly under `/` is a section. There is nothing to configure |
| `org.drftpd.master.sections.conf.SectionManager` | Sections are defined in `config/sections.conf` |

Changing `sectionmanager` needs a restart of the master.

### The default section manager

Every top level directory is a section, named after the directory (a directory `/MP3` is the section `MP3`).
A directory deeper in the tree belongs to the section of its top level parent: `/MP3/Some.Release` is in `MP3`.
New top level directories become sections immediately, and removing a directory removes the section. The section
colour is always `15`.

Note that this manager lists every top level directory, including ones hidden from users with
`privpath`, and that you cannot have dated directories or a name different from the directory name.

### The configurable section manager

Set `sectionmanager=org.drftpd.master.sections.conf.SectionManager` and create `config/sections.conf` from
`sections.conf.dist`. The file is read again by `SITE RELOAD`.

```
# Create the section directories at start and on reload if they do not exist.
# Default: [true]
make.section.dirs=true

1.path=/APPS
1.name=APPS
1.color=15

2.type=dated
2.path=/MP3
2.name=MP3
2.dated=MMdd
2.now=-today
2.color=10
```

Sections are numbered `1`, `2`, `3`, ... without gaps: reading stops at the first number that has no `.name`.

| Key | Required | Meaning |
| --- | --- | --- |
| `make.section.dirs` | no | `true` or `false`. When true, missing section directories are created at start and on reload. The dist file sets it to `true`. If the key is absent, the code treats it as `false` |
| `N.name` | yes | The section name. This is what users type and what plugins and announcements use |
| `N.path` | yes | The base directory of the section, for example `/MP3` |
| `N.type` | no | `plain` (default) or `dated`. Any other value stops the master at startup with an "Unknown section type" error |
| `N.color` | no | A colour number used in announcements and some command output. Default `15` |
| `N.dated` | dated only | A Java date pattern (`SimpleDateFormat`, for example `MMdd`, `yyyy-MM-dd`, `YYYY-ww`) used to name the current directory |
| `N.now` | dated only | A suffix for a symbolic link in `/` that always points to the current dated directory. The link is named `<name><now>`, so `MP3` plus `-today` gives `/MP3-today`. The key must be present, leave the value empty (`2.now=`) for no link |

A missing required key stops the master at startup with an error naming the key.

**Plain sections** are just a name for a directory: releases are uploaded directly into `N.path`.

**Dated sections** put releases into a directory named after the current date, inside the base directory. With
`2.dated=MMdd` and base `/MP3`, uploads go to `/MP3/0131` on 31 January. The master works out from the pattern how
often the directory changes (every day for `MMdd`, every week for a week number pattern, every month for
`yyyy-MM` and so on), creates the new directory at the right moment, owned by `drftpd`/`drftpd`, and updates the `now`
link. `SITE PRE` into a dated section places the release in the current dated directory. With `european.cal=true` in
[Master Configuration](Master-Configuration.md) weeks start on Monday and week numbers follow ISO 8601.

A directory belongs to the section whose base directory is the longest match for its path. A path that matches no
section has no section, and commands that need one answer with "Invalid section" (for `SITE PRE`) or ignore it.

## Where sections are used

- `SITE SECTIONS` lists the sections and their current directories.
- `SITE PRE <release> <section>` moves a release into the section's current directory. See [Site Commands](Site-Commands.md).
- `SITE NEW` lists the newest directories per section (options `default`, `max` and `filtered_sections` of the command, see
  [Site Commands](Site-Commands.md)). `SITE NEW <section> [number]` shows one section.
- Announcements: the [Sitebot](Sitebot.md) fills the `section` and `sectioncolor` variables of its announcement
  templates from the section of the directory involved, and can route events to different channels per section, see
  [Themes](Themes.md).
- [Slave Selection](Slave-Selection.md): the score filters that work on "releases" use the section to find the release
  directory (the directory directly below the section base).
- Nuking, [Find](Find.md) (`-section`), [Autonuke](Autonuke.md), [Autofreespace](Autofreespace.md), [Mirror](Mirror.md),
  [IMDB and TVMaze](IMDB-and-TVMaze.md), [Nukefilter](Nukefilter.md), [PreBW](PreBW.md), [Zipscript](Zipscript.md) announcements and the
  [Archive](Archive.md) plugin use section names or section colours. Their pages describe the exact options.

## Examples

Keep the defaults and make every top level directory a section:

```
sectionmanager=org.drftpd.master.sections.def.SectionManager
```

Two plain sections and a dated one, with a link to today's directory:

```
make.section.dirs=true

1.name=APPS
1.path=/APPS

2.name=GAMES
2.path=/GAMES
2.color=9

3.type=dated
3.name=TV
3.path=/TV
3.dated=yyyy-MM-dd
3.now=-today
3.color=12
```
