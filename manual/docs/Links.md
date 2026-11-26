> Applies to DrFTPD 4.0.x.

# Links

The links plugin keeps directories of symbolic links that point to releases in a certain state, so users can find them quickly. Typical uses:

- `/_INCOMPLETES_` with links to all incomplete releases.
- `/_NO_NFO_` with links to releases that have no NFO.
- `/_NO_SFV_` with links to releases that have no SFV.
- `/_LAST_RACES_` with links to the newest releases.

The links are created and removed automatically as files are uploaded and deleted. `SITE FIXLINKS` rebuilds them.

The incomplete types use the SFV or zip information of the [Zipscript](Zipscript.md), so it has to be active.

## Configuration

Settings are in `config/plugins/links.conf` (copy `links.conf.dist` to `links.conf`) and, for the latest directory type, in `config/plugins/latestdir.conf`. The settings are re-read on `SITE RELOAD`.

**The shipped `links.conf.dist` contains no active rules**, only documentation and examples in comments. Nothing happens until you add rules to `links.conf`.

Each rule is a block of numbered keys. The number goes in front of the key name, starting at 1 and without gaps (the plugin stops reading at the first missing number):

```
1.type=SFVINCOMPLETE
1.dirname=/_INCOMPLETES_
1.section=*
...
2.type=NFOMISSING
2.dirname=/_NO_NFO_
...
```

### Keys common to all types

| Key | Default | Meaning |
|-----|---------|---------|
| `N.type` | required | The link type: `LATESTDIR`, `NFOMISSING`, `SFVINCOMPLETE`, `SFVMISSING` or `ZIPINCOMPLETE` (not case sensitive). A rule with an unknown type is ignored. |
| `N.dirname` | `%` | The directory the links are created in, for example `/_INCOMPLETES_`. It is created if it does not exist. `%` means the section directory (more exactly, the parent of the release), so the link is placed next to the release. |
| `N.section` | `*` | The section to work on. `*` means all sections. If you name a section that does not exist, the rule is skipped and an error is logged. |
| `N.exclude` | empty | A regular expression. Releases whose path matches it get no link. The expression has to match the whole path. |
| `N.sectionexclude` | empty | A regular expression matched against the section name. Sections that match get no links. It only works for real sections defined in [Sections](Sections.md). |
| `N.deleteon` | `*` | When links are deleted along with the release. A list separated by `;`. The values that have an effect are `rmd` (directory removed), `wipe` (directory wiped) and `*` (all of them). When a release is renamed, for example by a nuke, its links are updated to follow it. |
| `N.linkname` | required | The name of the link. `${dirname}` is the name of the release and `${section}` the section name. A rule without a `linkname` is skipped. |
| `N.addparentdir` | empty | A regular expression for directory names such as `CD1` or `Sample` that are subdirectories of a release (see below). |

`addparentdir` handles releases with subdirectories (CD1, CD2, ...): if the directory name matches, the link name is made from the last two parts of the path joined by a dash, so `/section/rls1/cd1` gets the name `(incomplete)-rls1-cd1`. For a rule with `dirname=%` the link is also placed in the parent of the release rather than inside it.

## Link types

These are all the types that exist.

### LATESTDIR

Links to the most recently created directories, for a "latest" listing. A link is added whenever a directory is created (`MKD`), and the oldest links are removed when there are more than allowed.

`latestdir.conf`:

| Key | Default | Meaning |
|-----|---------|---------|
| `maxcount` | `10` | How many links to keep. |

For this type `N.exclude` is matched against the directory name (not the whole path). Set a fixed `N.dirname` (not `%`). Example from the dist file:

```
1.type=LATESTDIR
1.dirname=/_LAST_RACES_
1.section=*
1.exclude=
1.sectionexclude=^(?i)(pre|staff|speedtests)$
1.deleteon=*
1.linkname=(last)-${dirname}
1.addparentdir=^(?i)(((cd|dvd|dis[ck])[1-9])|sample|subs|rarfix)$
```

### NFOMISSING

Links to directories that have no `.nfo` file. A link is added when a directory is created and removed when an `.nfo` is uploaded. It is added again if the last `.nfo` is deleted.

`addparentdir` is used so that only directories at the bottom of a tree get a link: a directory is linked if it has no subdirectories, or if it contains a subdirectory that matches `addparentdir`. For example, in `/section1/sub-section1/rls1` you do not want a "no NFO" link for `sub-section1`.

```
2.type=NFOMISSING
2.dirname=/_NO_NFO_
2.section=*
2.exclude=^/(.*)/(?i)(((cd|dvd|dis[ck])[1-9])|sample|subs|rarfix)$
2.sectionexclude=^(?i)(pre|staff|speedtests)$
2.deleteon=*
2.linkname=(no-nfo)-${dirname}
2.addparentdir=^(?i)(((cd|dvd|dis[ck])[1-9])|sample|subs|rarfix)$
```

### SFVMISSING

Links to directories that have no `.sfv` file. Added when a directory is created, removed when an `.sfv` is uploaded, and added again if the last `.sfv` is deleted. `addparentdir` is used the other way round: when a directory contains a subdirectory such as `CD1`, the release directory itself does not get a "no SFV" link, because the SFV is expected in `CD1`.

```
4.type=SFVMISSING
4.dirname=/_NO_SFV_
4.section=*
4.exclude=^/(.*)/(?i)(sample|nfofix|subs)$
4.sectionexclude=^(?i)(pre|staff|speedtests)$
4.deleteon=*
4.linkname=(no-sfv)-${section}-${dirname}
4.addparentdir=^(?i)(((cd|dvd|dis[ck])[1-9]))$
```

### SFVINCOMPLETE

Links to directories whose release is incomplete according to the `.sfv`. The link is created when the `.sfv` is uploaded and removed when the last missing file arrives. Deleting a file brings the link back, deleting the `.sfv` removes it.

Two usual setups. A central directory:

```
3.type=SFVINCOMPLETE
3.dirname=/_INCOMPLETES_
3.section=*
3.exclude=
3.sectionexclude=^(?i)(pre|staff|speedtests)$
3.deleteon=*
3.linkname=(incomplete)-${dirname}
3.addparentdir=^(?i)(((cd|dvd|dis[ck])[1-9])|sample|subs|rarfix)$
```

Or a link next to the release (`/section/rls1` gives `/section/(incomplete)-rls1`, and `/section/rls1/cd1` gives `/section/(incomplete)-rls1-cd1`):

```
3.type=SFVINCOMPLETE
3.dirname=%
3.section=*
3.exclude=
3.sectionexclude=^(?i)(pre|staff|speedtests)$
3.deleteon=*
3.linkname=(incomplete)-${dirname}
3.addparentdir=^/(.*)/((cd|dvd|dis[ck])[1-9])$
```

### ZIPINCOMPLETE

Like SFVINCOMPLETE, for releases made of `.zip` files: the number of expected zips comes from the `file_id.diz` (see [Zipscript](Zipscript.md)). The link is created when a zip is uploaded and the release is not finished, and removed when it is complete or no zip is left. The keys and the two setups are the same as for SFVINCOMPLETE, with `type=ZIPINCOMPLETE`:

```
5.type=ZIPINCOMPLETE
5.dirname=/_INCOMPLETES_
5.section=*
5.exclude=
5.sectionexclude=^(?i)(pre|staff|speedtests)$
5.deleteon=*
5.linkname=(incomplete)-${dirname}
5.addparentdir=^(?i)(((cd|dvd|dis[ck])[1-9])|sample|subs|rarfix)$
```

You can use `SFVINCOMPLETE` and `ZIPINCOMPLETE` with the same `dirname`, so incomplete SFV and zip releases end up in one directory.

## Commands

| Command | Default permission | Description |
|---------|--------------------|-------------|
| `SITE FIXLINKS` (IRC: `fixlinks`) | `=siteop` | Re-checks the current directory and everything below it and creates or removes links to match the real state. Runs in the background. Use it after you add rules, or if links got out of date (for example while slaves were offline). It takes no arguments. |

`SITE FIXLINKS` does nothing for `LATESTDIR` links. These commands are defined in the core command files, see [Site-Commands](Site-Commands.md).

## Notes

- Link directories like `/_INCOMPLETES_` are real directories on the site. Set [Permissions](Permissions.md) on them as you would for any other directory (for example, make them read only for normal users).
- The link types look at all the releases of the sections they match. Use `sectionexclude` to keep the `PRE`, `STAFF` or `SPEEDTEST` areas out.
- Links are created with the owner and group `drftpd`.

## Related pages

- [Zipscript](Zipscript.md), [Sections](Sections.md), [Permissions](Permissions.md)
