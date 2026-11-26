> Applies to DrFTPD 4.0.x.

# Upgrading

## From 3.x to 4.x

You cannot keep the virtual file system or the slave files' metadata: 4.x builds the VFS again when the slaves connect and remerge for the first time. In short, from `UPGRADING.txt`:

- Install 4.x as a fresh installation, see [Installation on Linux](Installation-Linux.md) or [Installation on Windows](Installation-Windows.md). Do not install over the old version.
- **User files** that predate 4.x have to be converted. Userfiles are converted correctly if the password is not encrypted. The repository ships a converter in `tools/upgrade` (`migrate-v3-v4.py`). Its README says:
  - copy the directory contents to `runtime/master/`, and put the converter next to your v3 `userdata` directory;
  - run it first in dry-run mode: `python3 migrate-v3-v4.py` (install missing Python libraries as needed);
  - then in execute mode: `python3 migrate-v3-v4.py -e`. This converts the user files and writes a `groups.properties` file with the groups that do not exist yet;
  - options: `-d/--debug`, `-u/--userdata-dir` (default `userdata`), `-e/--exec`, `-g/--group-output-file` (default `groups.properties`);
  - make the shell scripts executable (`chmod 755 *.sh`), compile the Java helper with `./compile.sh` (Linux) or `compile.bat` (Windows), then create the group objects with `./run.sh` (Linux) or `run.bat` (Windows).
  The README of that tool calls the Java part "WIP", so check the result.
- Start the master and add your slaves again ([Adding Slaves](Adding-Slaves.md)). The VFS is built when the slaves merge.
- **Themes and all `.conf` files have to be rebuilt for 4.x.** Copying the old files over gives you a lot of problems.

SSL/TLS also changed in 4.0, see [SSL/TLS](SSL-TLS.md): master-slave connections are always TLS and configured through `slavemanager.ssl.*` and `master.ssl.*`.

## From 4.x to the latest 4.x

`UPGRADING.txt` says to delete the `src` folder of the old source tree before upgrading, then:

```
mvn clean
mvn validate
mvn install
```

(`mvn validate` copies the new default config files into `runtime`, `mvn clean` removes the `lib` and `build` directories there.) Afterwards check that your config files use the new options: compare your `.conf` files with the new `.conf.dist` files.

Remember that a `name.conf` file completely replaces `name.conf.dist`: when a release adds a new key to a `.dist` file, your `.conf` does not get it. Merge new options by hand.

### Command configuration files (4.0.12 and later)

The command definitions in `config/commands/ftp` and `config/commands/irc` now ship as `X.conf.dist`. DrFTPD loads `X.conf` when it exists and falls back to `X.conf.dist` otherwise.

- If you kept `X.conf` files from an earlier version, they now completely replace the new `X.conf.dist` files. Commands added in later releases will not show up until you merge the new `X.conf.dist` into your `X.conf` by hand, or delete your `X.conf` to use the shipped defaults.
- If a command moved from one file to another between versions (for example the request commands moving out of `core.conf`), an old customised `core.conf` together with the new `request.conf.dist` defines the command twice, and the master stops at startup with "already mapped". Remove the duplicate definition from your own `X.conf`.
- If you never customised a command file, delete your `X.conf` so the shipped `X.conf.dist` is used.

The same fallback applies to the main and plugin config files: `config/master.conf`, `config/perms.conf`, `config/plugins/*.conf` and so on.

### Docker

With Docker, `initconfig` copies new defaults with `cp -rn`, which never overwrites existing files. Compare your mounted `config` against `config.dist` in the new image after each upgrade. See [Docker](Docker.md).

See also the `CHANGELOG.txt` and `MAJORCHANGES.txt` in the source tree, and [Building from Source](Building-from-Source.md).
