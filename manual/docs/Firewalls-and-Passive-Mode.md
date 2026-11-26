> Applies to DrFTPD 4.0.x.

# Firewalls and Passive Mode

## Ports at a glance

| Connection | Where it is configured | Default |
|---|---|---|
| FTP control (client to master) | `master.port` in `master.conf` | 2121 |
| Slave to master | `master.bindport` in `master.conf` (and `slave.conf`) | 1099 |
| Passive data ports on the master (directory listings) | `pasv_ports` in `perms.conf` | commented out: random OS port |
| Passive data ports on a slave (transfers) | `slave.portfrom`, `slave.portto` in `slave.conf` | commented out: random OS port |

Open on the firewall:

- The master: the FTP port (2121) for clients, the slave port (1099) for slaves only, and the `pasv_ports` range for clients if you set one.
- Each slave: the `slave.portfrom` to `slave.portto` range, for clients (and other FTP servers when you FXP). The slave also needs to reach the master on port 1099 (outgoing).

If you do not set a range, the operating system picks random ports, which a firewall will block. Always set a range on the master and on each slave if there is a firewall.

## Master: `master.conf` and `perms.conf`

```
# master.conf
master.bindport=1099
#master.ip=xxx.xxx.xxx.xxx   # listen only on this IP; if commented, listen on all interfaces
master.port=2121
```

```
# perms.conf
# Accepts only one range of ports.
#pasv_ports 50000-60000

# To be used if your master does not report the correct outside ip for PASV commands.
#pasv_addr xxx.xxx.xxx.xxx
```

`pasv_ports` is a single `min-max` range (one range only), `pasv_addr` is the address the master reports in its `227` reply for **local** (listing) transfers. It also uses the `master.ip` bind address, if set, for its data sockets.

## Slave: `slave.conf`

```
# PASV port range.
#slave.portfrom=30000
#slave.portto=33000
# BIND IP - This will bind SLAVE ip to this address.
#bind.ip=xxx.xxx.xxx.xxx
```

If the range is missing or invalid, the slave logs `Unable to read port range from config` and falls back to a random OS port. With a range, the slave picks a random port inside it and moves on to the next free one if it is in use. A very small range can run out under load (`PortRange exhausted`), so give each slave a generous range. `bind.ip` makes the slave bind its data sockets (and its connection to the master) to one address, useful on multi-homed hosts.

## Passive mode and PRET

DrFTPD needs to know which file will be transferred before it can pick a slave. Clients therefore must send `PRET` before `PASV`, see [Introduction](Introduction.md). A client without PRET support gets `500 You need to use a client supporting PRET (PRE Transfer) to use PASV`. Use active mode with such a client, or choose a [PRET capable client](https://github.com/drftpd-ng/drftpd/wiki/Clients-with-PRET-support). The protocol is described in [PRET Specifications](https://github.com/drftpd-ng/drftpd/wiki/PRET-Specifications).

When the client sends `PASV`, the master answers with the **slave's** address and port in the `227 Entering Passive Mode (...)` reply, and the client connects straight to the slave. The slave must therefore be reachable from the client on the reported address.

## NAT and the reported address

The address in the `227` reply for slave transfers is, by default, the IP the slave connected to the master from. If the slave sits behind NAT or has a different public address, set the slave property on the master:

```
SITE SLAVE <slavename> SET pasv_addr <public-ip-or-hostname>
```

The master code reads the slave property `pasv_addr` and falls back to the slave's connection address. If the value cannot be resolved the client gets `Address is unresolvable, check pasv_addr setting on <slave>`. Forward the slave's port range on the NAT router to the slave.

For the master's own listing transfers, set `pasv_addr` in `perms.conf` (above) when the master is behind NAT, and forward the `pasv_ports` range. If `pasv_addr` is not set there, the master uses the local address of the client's control connection.

Behind NAT on the **client** side, an active-mode `PORT` command with a private (site-local) address from a client that connects from a public address is refused with `==YOU'RE BEHIND A NAT ROUTER==`. Configure the client to use its real IP and forward the port on its router, or use passive mode with PRET.

## Active mode

In active mode (`PORT`), the slave connects out to the client address. The slave needs outgoing access and, when `bind.ip` is set, uses it as source address. The client has to be reachable on the port it announces.

## Bind addresses

- Master FTP listener, slave listener and master data sockets: `master.ip` (`master.conf`).
- Slave data sockets and the slave's connection to the master: `bind.ip` (`slave.conf`).
- Neither key is set by default: DrFTPD listens on all interfaces.

## FTP bouncers

If a bouncer sits in front of the master, list its addresses in `perms.conf` with `bouncer_ips`, so DrFTPD can tell bouncer connections from users (see [Permissions](Permissions.md)).

## Docker

Port mapping examples for containers are in [Docker](Docker.md). TLS settings that affect data connections are in [SSL/TLS](SSL-TLS.md).
