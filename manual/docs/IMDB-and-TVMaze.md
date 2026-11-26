> Applies to DrFTPD 4.0.x.

# IMDB and TVMaze

Two plugins add information about movies and TV shows to your site:

- **imdb** looks up a movie on IMDB when an NFO is uploaded, announces it, stores it and can show it as a bar in directory listings.
- **tvmaze** looks up a TV show and episode on TVmaze when a release directory is created, with the same announce and bar features.

Both talk to a website over the internet **from the master** (the slaves are not involved in the TVmaze lookup; the IMDB plugin also has a small part on the slaves that reads the NFO file). Both do the lookups in the background, one at a time with a random pause in between, so uploading is never delayed by them and the remote site is not hammered.

The information is stored with the release directory, so it can be searched (see [Find](Find.md) and [Index-and-Search](Index-and-Search.md)) and shown in listings. Announces go through the [Sitebot](Sitebot.md), the text can be changed in [Themes](Themes.md).

---

# IMDB

## How a lookup is triggered

1. A file ending in `.nfo` (not `imdb.nfo`) is uploaded in a section listed in `race.section.N`. The same also happens when such an NFO is added to the file system in another way.
2. The directory is put in a queue. A background worker takes one directory at a time and waits a random number of seconds (between `delay.start` and `delay.end`) before the next one.
3. The NFO is read on a slave and searched for an IMDB link (`.../title/tt1234567`). Only the title id is used. If the NFO has no IMDB link, nothing is found.
4. The plugin fetches the title page from the configured IMDB address, reads the data, and stores it with the directory.
5. The info is announced on IRC (`imdb` announce). After an NFO upload the announce waits up to 20 seconds for the lookup to finish. After a successful `SITE PRE` of a release in a listed section, the announce is sent as well, using the data stored for the release (it waits up to 20 seconds for it).

Only movies are handled. If the link points to a TV show, the lookup is skipped and a warning is logged (use [TVmaze](#tvmaze) for TV).

Deleting the NFO (or wiping the directory) removes the stored IMDB data.

What is stored and shown: title, year, language, country, director, genres, plot, rating (shown like `7.4`), votes, runtime in minutes and the IMDB URL. Values that were not found are shown as `N|A`, or as `0`, `-` or `9999` for numbers, depending on where they are shown. The vote count is derived from an abbreviated number on the page, so it is approximate.

Lookups depend on the layout of the IMDB web page. If IMDB changes its pages, lookups can stop working until the plugin is updated.

## Configuration

`config/plugins/imdb.conf` (copy `imdb.conf.dist` to `imdb.conf`). Changes are read on `SITE RELOAD`.

| Key | Default | Meaning |
|-----|---------|---------|
| `url.base` | `https://imdb.com` | The IMDB address. Title URLs are built as `<url.base>/title/<id>`. Change this if IMDB changes its domain. |
| `url.search` | `https://imdb.com/find?s=all&q=` | The address used by `SITE IMDB <name>` to search by name. |
| `race.section.N` | not set | Sections where IMDB info is looked up and announced (`race.section.1=MOVIES`, `race.section.2=...`). A section here must have its own directory, not the site root. **Nothing happens unless you set at least one.** |
| `search.release` | `false` | With `SITE IMDB`, also check whether the movie exists on the site. |
| `search.hd.section.N` | not set | Sections that count as HD for `search.release`. |
| `search.sd.section.N` | not set | Sections that count as SD for `search.release`. |
| `delay.start` | `2` | Minimum seconds between two lookups. |
| `delay.end` | `5` | Maximum seconds between two lookups. If `delay.start` is not smaller than `delay.end`, the values 5 and 10 are used. |
| `imdbbar.enabled` | `false` | Show the IMDB bar in directory listings. |
| `imdbbar.directory` | `true` | The bar is a directory entry (`true`) or a file entry (`false`). |
| `filter.N` | not set | Words (regular expressions) removed from a search string before `SITE IMDB` looks it up (`filter.1=...`, `filter.2=...`). |
| `exclude` | empty | A regular expression. Directories whose name matches it are skipped. The dist file notes that only directories that look like a release (`name.with.dots-GRP`) pass, so only add patterns for such names. |

Example:

```
race.section.1=MOVIES
race.section.2=MOVIES-HD
search.release=true
search.hd.section.1=MOVIES-HD
search.sd.section.1=MOVIES
imdbbar.enabled=true
```

## Commands

| Command (FTP / IRC) | Default permission | Description |
|---------------------|--------------------|-------------|
| `SITE IMDB [-v] <nameofmovie>` / `!imdb` | `*` | Search IMDB for a movie by name and show the result. `-v` gives the longer output. If `search.release` is on, it also says whether the movie exists in the HD and SD sections. |
| `SITE CREATEIMDB [path]` / `!createimdb` | `=siteop` | Create IMDB data for all movies below the current directory (or the given path) that have an NFO. Useful for existing releases. It works through the index, so the index must be up to date. |
| `SITE REMOVEIMDB [path]` / `!removeimdb` | `=siteop` | Remove IMDB data from all movies below the current directory or path. |
| `SITE IMDBQUEUE` / `!imdbqueue` | `=siteop` | Show how many directories are waiting in the lookup queue. |

See [Site-Commands](Site-Commands.md) and [IRC-Commands](IRC-Commands.md).

With [Find](Find.md) you can search by IMDB data: the options `imdbtitle`, `imdbdirector`, `imdbgenres`, `imdbvotes`, `imdbrating`, `imdbyear` and `imdbruntime` (ranges as `min:max`), and the action `PrintIMDB` prints the stored data for the matches.

## IRC announce

Announce type: `imdb`, routed by path or section (see [Sitebot](Sitebot.md)):

```
imdb.path.1=/path/subpath/*
imdb.path.1.destination=public
imdb.section.1=*
imdb.section.1.destination=public
```

The text, from `config/themes/irc/imdb.announcer.theme.default`:

```
imdb.announce=${color}${sectioncolor}${bold}${section}${bold}${coloroff} ${color}14-IMDB-${coloroff} ${color}07${title}${coloroff} (${year})\n...
```

(Three lines in the default: title and year; genres, language, country, director, rating, votes and runtime; plot; and the URL.) Tags: `title`, `year`, `language`, `country`, `director`, `genres`, `plot`, `rating`, `votes`, `url`, `runtime`, `release` (the directory name), `section`, `sectioncolor`.

## Themes

| File | Covers |
|------|--------|
| `config/themes/irc/imdb.announcer.theme` | The announce (`imdb.announce`). |
| `config/themes/ftp/imdb.theme`, `config/themes/irc/imdb.theme` | Replies of the commands: `announce`, `announce.verbose`, `announce.release`, `notfound`, `createimdb.*`, `removeimdb.*`, `imdb.queue`. |
| `config/themes/ftp/imdb.list.theme` | The bar in listings (`imdb.dir`). |

The default bar: `[IMDB] ${genres} - ${rating}/10 - ${votes} Votes - ${runtime} min [IMDB]`.

---

# TVMaze

## How a lookup is triggered

1. A directory is created (`MKD`), or a release is moved in with `SITE PRE`, or the directory appears in the file system by other means.
2. The directory name must look like a release, which means it has the form `Show.Name.S01E02.Tag.Tag-GRP` (at least three dot separated parts, then `-` and a group name: the pattern is `(\w+\.){3,}\w+-\w+`).
3. The directory must be in a section listed in `race.section.N`, and its name must not match `exclude`.
4. The directory is queued and processed in the background, one at a time, with a random pause (`delay.start` to `delay.end` seconds).
5. The show name is taken from the release name: season and episode (`S01E02`, `S01`, `1x02`), a year and a country code (`UK`, `GB`, `US`, `CA`, `AU`) are removed from the name and used to choose the right show. The name is then searched on the TVmaze web service, the best match is taken, and the episode is fetched.
6. The data is stored with the directory and announced on IRC (`tvmaze`). After `MKD` or `SITE PRE` the announce waits up to 10 seconds for the lookup to finish; if the data is not there by then, nothing is announced for that directory.

The address used is the public API of TVmaze (`api.tvmaze.com`); it is fixed in the plugin, there is no setting for it.

## Configuration

`config/plugins/tvmaze.conf` (copy `tvmaze.conf.dist` to `tvmaze.conf`). Changes are read on `SITE RELOAD`.

| Key | Default | Meaning |
|-----|---------|---------|
| `race.section.N` | not set | Sections where TVmaze info is looked up and announced. **Nothing happens unless you set at least one.** |
| `search.release` | `false` | With `SITE TV` for a single episode, also check whether the episode exists on the site. |
| `search.hd.section.N` | not set | Sections that count as HD for `search.release`. |
| `search.sd.section.N` | not set | Sections that count as SD for `search.release`. |
| `exclude` | empty | A regular expression. Directories whose name matches it are skipped. As with IMDB, only names that look like releases are processed anyway. |
| `filter.N` | not set | Words (regular expressions) removed from the show name before searching. |
| `date.show` | `yyyy-MM-dd` | Date format in the output. |
| `time.show` | `EEEE, HH:mm` | Time format in the output (episode air time). |
| `timezone` | `-05:00` | The time zone all dates and times are shown in. If you remove the line, the time zone of the master is used. |
| `delay.start` | `0` | Minimum seconds between two lookups. |
| `delay.end` | `5` | Maximum seconds between two lookups. If `delay.start` is not smaller than `delay.end`, the values 0 and 5 are used. |
| `tvmazebar.enabled` | `false` | Show the TVmaze bar in directory listings. |
| `tvmazebar.directory` | not set | Bar as a directory entry (`true`) or a file entry (`false`). Set it explicitly when you enable the bar. |

Example:

```
race.section.1=TV
race.section.2=TV-HD
search.release=true
search.hd.section.1=TV-HD
search.sd.section.1=TV
timezone=+01:00
tvmazebar.enabled=true
tvmazebar.directory=true
```

## Commands

| Command (FTP / IRC) | Default permission | Description |
|---------------------|--------------------|-------------|
| `SITE TV [-v] <nameofshow> [episodenumber]` / `!tv` | `*` | Look up a show. With only a name you get the show, its previous and next episode. Add a season and episode in the name (for example `s02e05` or `2x05`) to get one episode, or just a season (`s02`) to get the list of episodes. `-v` gives the longer output. |
| `SITE CREATETV [path]` / `!createtv` | `=siteop` | Create TVmaze data for all shows below the current directory or path. Useful for existing releases. It walks the directories below the path. |
| `SITE REMOVETV [path]` / `!removetv` | `=siteop` | Remove TVmaze data below the current directory or path. |
| `SITE TVQUEUE` / `!tvqueue` | `=siteop` | Show how many directories wait in the lookup queue. |

With [Find](Find.md) you can search by TVmaze data: `tvname`, `tvgenre`, `tvseason`, `tvnumber`, `tvtype`, `tvstatus`, `tvlanguage`, `tvcountry` and `tvnetwork`. The find action `PrintTvMaze` prints the stored data for the matches.

## IRC announce

Announce type: `tvmaze`, routed by path or section (see [Sitebot](Sitebot.md)):

```
tvmaze.path.1=/path/subpath/*
tvmaze.path.1.destination=public
tvmaze.section.1=*
tvmaze.section.1.destination=public
```

The text, from `config/themes/irc/tvmaze.announcer.theme.default`:

```
tvmaze.announce=${color}${sectioncolor}${bold}${section}${bold}${coloroff} ${color}14-TvMaze-${coloroff} ${color}07${tvname}${coloroff} (${epname} - ${epairdate}) :: ${color}07${country}${coloroff} :: ${color}07${network}${coloroff} :: ${color}07${status}${coloroff} :: ${color}07${type}${coloroff} :: ${color}07${genres}${coloroff} :: ${underline}${tvurl}${underline}
```

Tags for the announce:

- Show: `id`, `tvurl`, `tvname`, `type`, `language`, `genres`, `status`, `runtime`, `premiered`, `network`, `country`, `summary`.
- Episode: `epid`, `epurl`, `epname`, `epseason`, `epnumber`, `epairdate`, `epairtime`, `epruntime`, `epsummary`, `epage`.
- `section`, `sectioncolor`.

The command output has the same show and episode tags, and in addition `prevep...` and `nextep...` tags (the same names as the episode tags with the prefix `prevep` or `nextep`, for example `prevepname`).

## Themes

| File | Covers |
|------|--------|
| `config/themes/irc/tvmaze.announcer.theme` | The announce (`tvmaze.announce`). |
| `config/themes/ftp/tvmaze.theme`, `config/themes/irc/tvmaze.theme` | Replies of the commands: `tv.show`, `tv.prevep`, `tv.nextep`, `tv.ep`, `tv.ep.release`, `tv.ep.season.header`, `tv.ep.season`, `tv.none`, each also with `.verbose`, plus `createtvmaze.*`, `removetvmaze.*`, `tv.queue`. |
| `config/themes/ftp/tvmaze.list.theme` | The bar in listings (`tvmaze.dir`). |

The default bar: `[TvMaze]-${tvname}_${epseason}x${epnumber}_${epairdate}_${country}_${network}_${type}_${epruntime}m-[TvMaze]`.

---

## Notes for both plugins

- The master needs outgoing internet access (HTTPS) to reach the services. If the lookup fails (network error, show not found), nothing is stored for the directory, and the reason is logged. The `SITE TV` command shows the error text.
- Use a modest `delay.start`/`delay.end` on a busy site. A long queue just means info arrives later; see the queue length with `SITE IMDBQUEUE` or `SITE TVQUEUE`.
- To fill in data for releases that were uploaded before you turned the plugin on, use `SITE CREATEIMDB` or `SITE CREATETV`. `SITE CREATEIMDB` relies on the site index (see [Index-and-Search](Index-and-Search.md)); `SITE CREATETV` walks the directories.
- Sections for `race.section.N` must exist in [Sections](Sections.md), and must not be the site root.

## Related pages

- [Sitebot](Sitebot.md), [Themes](Themes.md), [Find](Find.md), [Index-and-Search](Index-and-Search.md), [Sections](Sections.md)
