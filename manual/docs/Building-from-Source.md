> Applies to DrFTPD 4.0.x.

# Building from Source

## Layout of the source tree

```
pom.xml                  root, version 4.0.12-SNAPSHOT on master, Java release 17
src/core/common          shared code (network, SSL, config loading, extensibility annotations)
src/core/master          the master (FTP server, VFS, user manager, commands, slave management)
src/core/slave           the slave
src/plugins/<name>       one directory per plugin, see below
dist/                    bundled jar (json-io)
docs/                    old text docs (commands.txt, perms.txt, ssl.txt, ...)
tools/upgrade            3.x to 4.x user file converter
.dev/pom.xml             project used to run master and slave from an IDE
Dockerfile, genkey.sh, genkey.bat
```

The plugin modules listed in `src/plugins/pom.xml` are: imdb, autonuke, speedtest, speedtest-net, mediainfo, jobs, mirror, tvmaze, archive, find, zipscript, links, raceleader, autofreespace, dupecheck, prebw, nukefilter, trial, traffic, statistics, request and sitebot. Some have separate `-common`, `-master` and `-slave` sub-modules (for example mediainfo and zipscript), others are a single module.

## Build

```
mvn clean       # optional, only for upgrades
mvn validate
mvn install
```

- `mvn validate` copies the default resources (the `config/*.dist` files, scripts) of each module into `runtime/master` or `runtime/slave`.
- `mvn install` compiles, runs the unit tests, and copies the dependency jars into `runtime/<master|slave>/lib` and the DrFTPD jars into `runtime/<master|slave>/build`.
- `mvn clean` removes `runtime/*/lib` and `runtime/*/build`, it does not touch your `config` directories.

Result:

```
runtime/master/{config,lib,build,logs,master.sh,master.bat}
runtime/slave/{config,lib,build,slave.sh,slave.bat}
```

See [Installation on Linux](Installation-Linux.md) for what to do next. CI (CircleCI) runs `mvn package`.

## Tests

The tests are JUnit 5 (Jupiter) tests under `src/**/src/test/java` and run through the Maven Surefire plugin:

```
mvn test
```

or `mvn -pl src/core/master -am test` to test only the master and what it depends on (standard Maven options).

## Run from an IDE

The README describes it for IntelliJ IDEA Community: open `.dev/pom.xml`, run `mvn package`, then create two Application run configurations.

| | Master | Slave |
|---|---|---|
| Main class | `org.drftpd.master.Master` | `org.drftpd.slave.Slave` |
| Working directory | `<checkout>/runtime/master` | `<checkout>/runtime/slave` |
| Environment variable | `DRFTPD_CONFIG_PATH=<checkout>/runtime/master` | `DRFTPD_CONFIG_PATH=<checkout>/runtime/slave` |
| JDK | Java 17 SDK of the `drftpd-dev` module | same |

`DRFTPD_CONFIG_PATH` is the directory under which DrFTPD looks for `config/`.

## Optional JVM options

- `-Dbytes.maxunit=M`: caps the largest unit used when DrFTPD formats sizes (`K`, `M`, `G`, `T`, `P` or `E`, case insensitive). Unset by default. Invalid values are ignored. It is global: it affects IRC announcements and FTP replies.
- `-Dbytes.binary=true`: use binary units (KiB, MiB, ...) instead of decimal ones.

Add them to `JVM_OPTS` or `OPTIONS` in `master.sh` / `master.bat`, or to `JAVA_TOOL_OPTIONS` in Docker.

## How plugins, commands and hooks are found

There is no plugin registry file. The master scans the class path (`lib/*` and `build/*`) at start-up with the Reflections library, in the package `org.drftpd`. Put your code under that package.

- **Plugins.** Every class implementing `org.drftpd.common.extensibility.PluginInterface` (`startPlugin()` and `stopPlugin(String reason)`) is instantiated at start-up. A plugin can declare start order with the `@PluginDependencies(refs = {...})` annotation. The master logs `Loaded plugin <class>` for each.
- **Commands.** FTP and IRC commands are defined in `config/commands/ftp/*.conf(.dist)` and `config/commands/irc/*.conf(.dist)` files. Each block names a `plugin` (a Java package), a `class`, a `method`, the `perms`, and help text. The master loads `<plugin>.<class>` by name and calls the method. For example (from the shipped file):
  ```
  SITE ADDSLAVE {
  plugin org.drftpd.master.commands.slavemanagement
  class SlaveManagement
  method doSITE_ADDSLAVE
  perms =siteop
  help Add a slave to the site.
  }
  ```
  A block missing `method`, `class`, `perms` or `plugin` is rejected, and the same command mapped twice stops the master with "already mapped". If `X.conf` exists it replaces `X.conf.dist`, see [Upgrading](Upgrading.md).
- **Hooks.** A method annotated with `@CommandHook(commands = {...}, priority = N, type = HookType.PRE|POST)` (defined in `org.drftpd.common.extensibility`) runs before or after the named command methods (for example `doSTOR`, `doMKD`). `type` defaults to `POST` and `priority` to 1. A PRE hook can deny a request. Example in the dupecheck plugin: `@CommandHook(commands = "doMKD", type = HookType.PRE)`. All annotated methods are collected with Reflections when `GlobalContext` starts.
- **Other extension points.** The master also discovers, by class path scan, implementations of `ExtendedPermissions` (extra `perms.conf` directives), list-element providers (`AddListElementsInterface`), and Lucene index extensions. Look at an existing plugin for the pattern.
- **Config files of a plugin** go in the plugin module's `src/main/resources/master/config/plugins/` (copied into `runtime/master/config/plugins` by `mvn validate`). Ship them as `.conf.dist`: code reads `name.conf` if present and falls back to `name.conf.dist`.

Start from a small plugin such as `src/plugins/links`, `src/plugins/dupecheck` or `src/plugins/trial`, and see [Plugins](Plugins.md) for what the existing ones do.

## Reporting and contributing

Bugs and pull requests go to https://github.com/drftpd-ng/drftpd, see [Getting Help](Getting-Help.md).
