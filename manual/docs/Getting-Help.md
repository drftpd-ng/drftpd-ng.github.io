> Applies to DrFTPD 4.0.x.

# Getting Help

## Read first

- This wiki, starting with the [FAQ](FAQ.md).
- The [Installation on Linux](Installation-Linux.md) or [Installation on Windows](Installation-Windows.md) page, and [Adding Slaves](Adding-Slaves.md).
- Your logs. The master writes to `runtime/master/logs` (`master.log`, `remoteslave.log`, `commands.log`, and per-plugin logs such as `sitebot.log`), the slave to `logs/slave.log` in the slave directory. Log verbosity is explained in [Log Levels](Log-Levels.md).

## IRC

- `#drftpd` on EFnet: ircs://irc.efnet.org:6697/drftpd (support)
- `#drftpd-devel` on EFnet: ircs://irc.efnet.org:6697/drftpd-devel (development, support too)

## Bug tracker

Report bugs and request features at https://github.com/drftpd-ng/drftpd/issues. The project supplies a bug report template. Security problems are handled the same way: `SECURITY.md` says to report them on the bug tracker or in a public IRC channel such as `#drftpd` / `#drftpd-devel`. Only the latest stable release of the 4.x branch is supported.

## What to put in a bug report

The issue template asks for:

- **Describe the bug**: a clear and concise description.
- **To reproduce**: the steps (log in on FTP, or issue an IRC command; execute the command; the error that happens).
- **Expected behavior**.
- **DrFTPD**: OS (for example Debian, Windows), Java version, DrFTPD version.
- **FTP client**: OS, software (FlashFXP, FTPRush, lftp, ...), version.
- **Additional context**. If the bug is related to a specific plugin, attach the plugin's configuration file (the template mentions `build.conf`; for 4.x that means the relevant `config/plugins/*.conf`).

Also helpful: the relevant lines of `master.log` and `slave.log` around the time of the problem, whether it happens on one slave or all, and the exact FTP reply or error message. Remove passwords, certificates and private IP addresses before posting logs or config files in public.
