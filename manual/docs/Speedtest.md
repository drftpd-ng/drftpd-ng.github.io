> Applies to DrFTPD 4.0.x.

# Speedtest

DrFTPD has two separate plugins to measure how fast your slaves are:

| Plugin | What it measures | How |
|--------|------------------|-----|
| `speedtest` | the speed between **a user and a slave** | the user uploads a file to a speedtest directory, the site announces the speed and deletes the file |
| `speedtest-net` | the speed between **a slave and the Internet** | the slave runs a test against speedtest.net servers, started with `SITE SPEEDTEST` |

They do not need each other. See [Plugins](Plugins.md) for how plugins are loaded. `speedtest-net` has a master part and a slave part; the slave part must be installed on the slaves you want to test (see [Adding Slaves](Adding-Slaves.md)).

## Upload speedtest (plugin `speedtest`)

Users upload a file into a speedtest path. The master announces who sent it, how big it was, the speed and the slave that received it, and then **deletes the file** again. Nothing else happens, there are no commands.

### Configuration

`config/plugins/speedtest.conf` on the master (copy from `speedtest.conf.dist`):

```
path.1=/SPEEDTEST
```

| Key | Meaning |
|-----|---------|
| `path.N` | a path in the virtual file system. Any directory at or below it is a speedtest directory. `N` = 1, 2, 3, ... without gaps. Nothing is configured by default. |

The directory must exist and users need permission to upload there (see [Permissions](Permissions.md) and [Sections](Sections.md)). Files uploaded in these paths are deleted after the transfer, whether it worked or not, so do not use a real section.

You may want to exclude the speedtest path in other plugins, for example `exclude.sections=...SPEEDTEST` in [Autonuke](Autonuke.md), or an `nostatsup` line for the path in `perms.conf` (see [Statistics](Statistics.md)).

### Announce

The event is `speedtest`. The text is in `config/themes/irc/speedtest.announcer.theme.default`:

```
speedtest=SPEEDTEST: -> ${transferfile} (${size}) sent by ${user}/${group} @ ${speed} to ${slave}.
```

Variables: `${transferfile}`, `${size}`, `${user}`, `${group}`, `${speed}`, `${slave}`. To choose the channel add to `irc.announce.conf` (see [Sitebot](Sitebot.md)):

```
speedtest.destination=public
```

The announce is only sent for transfers that finished successfully.

## Slave to Internet speedtest (plugin `speedtest-net`)

The master tells a slave to test its connection against speedtest.net servers. The slave measures latency, upload and download and the master prints the result. Needs outbound Internet access on the slave. The master also needs Internet access to download the server list and to look up locations.

### Master config

`config/plugins/speedtest.net.conf` on the master (copy from `speedtest.net.conf.dist`):

```
distance.unit=K
server.selection=geo
```

| Key | Default | Meaning |
|-----|---------|---------|
| `distance.unit` | `K` | the unit of the distance shown to a test server: `K` (kilometers), `M` (miles), `N` (nautical miles) |
| `server.selection` | `geo` | how the nearest test servers are chosen, see below |

`server.selection`:

- `geo`: the master downloads the full list of speedtest.net servers, finds where the slave is (a lookup on ipinfo.io using the slave's PASV address, then saved on the slave as `lat` and `lon`) and uses the closest servers by distance.
- `slave`: each slave asks speedtest.net directly for the servers closest to it. The slave needs outbound HTTPS access to `www.speedtest.net`. If a slave cannot give a list, the master falls back to `geo` for it.

You can override this for one slave:

```
SITE SLAVE <slavename> SET speedtest.selection slave
SITE SLAVE <slavename> SET speedtest.selection geo
SITE SLAVE <slavename> UNSET speedtest.selection
```

`UNSET` makes the slave follow `server.selection` again. Any other value is ignored with a warning. A test server id given in `SITE SPEEDTEST` is always looked up in the full server list, in every mode.

If the location lookup for a slave fails, set its public address by hand: `SITE SLAVE <slavename> SET pasv_addr <public ip>` (see [Adding Slaves](Adding-Slaves.md)).

### Slave config

`config/plugins/speedtest.net.slave.conf` on the slave (copy from `speedtest.net.slave.conf.dist`). The shipped note says to leave the defaults unless you know what you are doing.

```
threads.up=3
payload.string=0123456789ABCDEFGHIJKLMNOPQRSTUVWXYZ
payload.repeat=7000
payload.loop=20
max.up.time=8

threads.down=3
sizes=350,500,750,1000,1500,2000,2500,3000,3500,4000
size.loop=4
max.down.time=8
```

| Key | Default | Meaning |
|-----|---------|---------|
| `threads.up` | `3` | parallel upload connections |
| `payload.string`, `payload.repeat`, `payload.loop` | as above | the upload payload: the string repeated `payload.repeat` times, and the whole payload sent `payload.loop` times, growing as the test goes on |
| `max.up.time` | `8` | seconds to measure the upload |
| `threads.down` | `3` | parallel download connections |
| `sizes` | as above | the sizes of the test images to download (as the speedtest.net server provides them) |
| `size.loop` | `4` | how many times each size is downloaded |
| `max.down.time` | `8` | seconds to measure the download |

### Commands

Both commands default to the `siteop` permission, set in `config/commands/ftp/speedtest.net.conf` (see [Site Commands](Site-Commands.md)). In IRC the command file is `config/commands/irc/speedtest.net.conf` (see [IRC Commands](IRC-Commands.md)).

```
SITE SPEEDTEST [<slavename> [-list | -update | <test server id>]]
SITE REFRESHSPEEDTEST
```

| Command | Meaning |
|---------|---------|
| `SITE SPEEDTEST <slavename>` | run the test on the slave against the 5 closest servers |
| `SITE SPEEDTEST *` | run the test on all slaves at the same time |
| `SITE SPEEDTEST <start>*` | run it on all slaves whose name starts with `<start>` |
| `SITE SPEEDTEST <slavename> -list` | only list the closest servers to the slave (5 of them), with sponsor, country, distance and id |
| `SITE SPEEDTEST <slavename> -update` | look up the slave's location again before testing |
| `SITE SPEEDTEST <slavename> <id>` | test against one server by its speedtest.net id |
| `SITE REFRESHSPEEDTEST` | download the speedtest.net server list again |

The result is printed for each slave as the server (sponsor, country, distance, id), the latency in ms and the up and down speed in Mbit/s. Offline slaves are skipped with a message. The server list is downloaded the first time it is needed and kept in memory; use `SITE REFRESHSPEEDTEST` to renew it.

The messages are in `config/themes/ftp/speedtest.theme.default` and `config/themes/irc/speedtest.theme.default` (see [Themes](Themes.md)). The plugin has no announces.
