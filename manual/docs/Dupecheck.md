> Applies to DrFTPD 4.0.x.

# Dupecheck

The dupecheck plugin refuses an upload or a new directory when a file or directory with the same name already exists somewhere on the site. It is how a site avoids getting the same release twice.

## Requirements

- The site **index** must be enabled and working: dupecheck asks the index for an exact name match across the whole site (see [Index-and-Search](Index-and-Search.md)). If the index does not know about a directory or file, it cannot be recognised as a dupe.
- Dupecheck only applies to the users and paths that you list in `perms.conf` (see below). **With no `dupecheck` line in `perms.conf`, nobody is checked**, whatever `type` says.

## How it works

Before a directory is created (`MKD`) or a file is uploaded (`STOR`), the plugin:

1. Looks at the `type` setting to see whether directories, files or both are checked.
2. Skips the name if it matches the `exempt` pattern.
3. Asks the index for entries with exactly that name, anywhere on the site.
4. If it finds one, and the user is covered by a `dupecheck` rule for the current directory, the request is refused with the reply `400 DUPE: <path of the existing entry>`.

An existing entry only counts as a dupe if it has not been un-duped with `SITE UNDUPE` (see below). Renaming a directory or moving it is not checked.

## Configuration

`config/plugins/dupecheck.conf` (copy `dupecheck.conf.dist` to `dupecheck.conf`):

| Key | Default | Meaning |
|-----|---------|---------|
| `type` | `3` | What is checked: `0` off, `1` files only, `2` directories only, `3` both. Any other value is logged as an error and switches dupe checking off. |
| `exempt` | `^(sample\|proof\|subs\|cd[1-9]\|dis[c\|k][1-9])$` | A regular expression for names that are never dupe checked (so every release can have its own `Sample` or `CD1`). The name is matched in lower case, and the expression may match anywhere in the name (use `^` and `$` to match the whole name). |

In the file the default is written with plain pipes:

```
type=3
exempt=^(sample|proof|subs|cd[1-9]|dis[c|k][1-9])$
```

Changes are read when the command configuration is reloaded (`SITE RELOAD`) or the master is restarted.

### Who is checked: perms.conf

The users and paths are set with a `dupecheck` path permission in `perms.conf` (see [Permissions](Permissions.md)). The users listed are the ones who **are subject** to the check:

```
dupecheck /PATH* =group -user
dupecheck /PATH* *
```

- `dupecheck /MP3/* *` checks every logged in user in `/MP3`.
- `dupecheck /MP3/* !=siteop *` checks everybody in `/MP3` except the siteop group.
- Paths that match no `dupecheck` line are not checked.

The first matching line decides. The path is compared to the directory the user is in when the command is issued.

## Commands

| Command | Default permission | Description |
|---------|--------------------|-------------|
| `SITE UNDUPE <directory/file>` (IRC: `undupe`) | `=siteop` | Marks every file or directory with that name as "not a dupe", so it can be uploaded again. |

`SITE UNDUPE` uses the index to find all entries with that exact name (give the name, not a path) and marks each as un-duped. After that those entries are ignored by the dupe check, but a further copy with the same name that has not been un-duped still counts. See [Site-Commands](Site-Commands.md) and [IRC-Commands](IRC-Commands.md).

## Announces and themes

Dupecheck has no IRC announce and no theme files.

## Related pages

- [Index-and-Search](Index-and-Search.md), [Permissions](Permissions.md), [Find](Find.md)
