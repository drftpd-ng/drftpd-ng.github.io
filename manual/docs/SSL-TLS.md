> Applies to DrFTPD 4.0.x.

# SSL/TLS

DrFTPD uses TLS in three places:

1. **FTP clients to the master** (explicit FTPS, `AUTH TLS`, control connection).
2. **Data connections** between a client and a slave (or the master, for listings), protected with `PBSZ`/`PROT`.
3. **The master to slave connection** (the slave control connection).

Since 4.0, the master-slave connection is always TLS and is configured separately from the client-facing TLS. Client connections no longer dictate the options for slave connections.

## The default key: `drftpd.key`

Both the FTP/data TLS context and the default master-slave keystore use a single keystore file, `config/drftpd.key` (PKCS12). The FTP and data connections load it by this fixed name and with the fixed password `drftpd` (it is not configurable in 4.0.x). Without a readable `config/drftpd.key`, the master logs `Couldn't load SSLContext, SSL/TLS disabled`.

Create it with the script in the source root:

```
./genkey.sh        # Linux
genkey.bat         # Windows
```

Both run:

```
keytool -genkeypair -keyalg EC -groupname secp384r1 -alias drftpd -dname CN=drftpd -keypass drftpd -keystore "drftpd.key" -storetype pkcs12 -storepass drftpd -validity 365
```

(`genkey.sh` also passes any extra arguments you give it on to `keytool`.) The result is a self-signed EC key valid for **365 days**. Put the file into `config/` of the master, and of each slave. Generate a new one before it expires.

The Docker master image creates it automatically when you run `initconfig`, see [Docker](Docker.md).

## FTPS for clients

`perms.conf` decides what is allowed unencrypted. The shipped defaults are strict:

```
userrejectsecure        !*
userrejectinsecure      *
denydiruncrypted        *
denydatauncrypted       *
```

That means every user must use TLS for login, directory listings and data transfers. Clients must enable explicit FTPS (AUTH TLS) and accept the self-signed certificate. To allow plain FTP for some users, change these lines (see [Permissions](Permissions.md)). Which protocols and ciphers clients may use is controlled by the plain `protocol.N` and `cipher.whitelist.N` keys in `master.conf`:

```
cipher.whitelist.1=.*_(POLY1305_SHA256|GCM_SHA384|GCM_SHA256).*
protocol.1=TLSv1.2
protocol.2=TLSv1.3
```

The slaves have their own `cipher.whitelist.N` and `protocol.N` keys in `slave.conf`, used for **data transfers** with clients. The master connection is separate (`master.ssl.*`).

## Master-slave TLS

The configuration is based on the Elasticsearch `elasticsearch-ssl-config` library, which offers many options. Only the keys shipped in the dist files are shown below.

### `master.conf` (the master accepts slaves)

```
slavemanager.ssl.supported_protocols.1=TLSv1.3
slavemanager.ssl.cipher_suites.1=TLS_AES_128_GCM_SHA256
slavemanager.ssl.cipher_suites.2=TLS_AES_256_GCM_SHA384
slavemanager.ssl.cipher_suites.3=TLS_CHACHA20_POLY1305_SHA256
slavemanager.ssl.verification_mode=none
slavemanager.ssl.client_authentication=optional
# slavemanager.ssl.certificate_authorities=(empty)
# slavemanager.ssl.truststore.path=(empty)
# slavemanager.ssl.certificate=
# slavemanager.ssl.key=
slavemanager.ssl.keystore.path=config/drftpd.key
slavemanager.ssl.keystore.password=drftpd
```

### `slave.conf` (the slave connects to the master)

```
master.ssl.supported_protocols.1=TLSv1.3
master.ssl.cipher_suites.1=TLS_AES_128_GCM_SHA256
master.ssl.cipher_suites.2=TLS_AES_256_GCM_SHA384
master.ssl.cipher_suites.3=TLS_CHACHA20_POLY1305_SHA256
master.ssl.verification_mode=none
master.ssl.client_authentication=optional
master.ssl.keystore.path=config/drftpd.key
master.ssl.keystore.password=drftpd
```

By default only TLS 1.3 with three ciphers is allowed, and with `verification_mode=none` nothing is verified, so the self-signed `drftpd.key` is enough. The `slave.conf.dist` file also contains `slave.masterSSL=true`; I could not find any code in 4.0.x that reads this key, so do not rely on it to switch master TLS off.

### A more secure setup: your own CA

`docs/ssl.txt` recommends changing the master-slave TLS to a private certificate authority and gives easy-rsa (https://github.com/OpenVPN/easy-rsa) as an example. After creating a CA, a master certificate and a slave (client) certificate, the master needs `ca.crt`, the master key and the master certificate, and each slave needs `ca.crt`, the client key and the client certificate. Copy them into `config/ssl/` (for example as `cacert.pem`, `server.key`, `server.pem` on the master and `cacert.pem`, `client.key`, `client.pem` on the slave) and set:

Master:

```
slavemanager.ssl.verification_mode=certificate
slavemanager.ssl.client_authentication=required
slavemanager.ssl.key=config/ssl/server.key
slavemanager.ssl.certificate=config/ssl/server.pem
slavemanager.ssl.certificate_authorities.1=config/ssl/cacert.pem
```

Slave:

```
master.ssl.verification_mode=certificate
master.ssl.client_authentication=required
master.ssl.key=config/ssl/client.key
master.ssl.certificate=config/ssl/client.pem
master.ssl.certificate_authorities.1=config/ssl/cacert.pem
```

This makes the master trust only certificates signed by your CA and demand a client certificate from every slave. These example keys come from `docs/ssl.txt`; the full list of options is defined by the elasticsearch-ssl-config library.

## Certificate pinning of slaves

With `slavemanager.ssl.client_authentication` set to `optional` or `required`, a slave presents its certificate (from `master.ssl.keystore.path`) to the master. The master logs the SHA-256 fingerprint of every slave certificate when the slave connects, for example:

```
Slave slave1 connected with certificate fingerprint SHA256:.... To only accept this certificate, run: SITE SLAVE slave1 SET fingerprint SHA256:...
```

To accept only that certificate from that slave (in addition to its IP masks), pin it:

```
SITE SLAVE <slavename> SET fingerprint SHA256:<fingerprint>
```

Behaviour, as implemented:

- The value is validated: `SHA256:` (or `SHA-256:`, or no prefix) followed by 64 hex digits. Case does not matter and colons or spaces between bytes are accepted, so you can paste the output of `openssl` or `keytool`. It is stored in the canonical form `SHA256:<UPPERCASE HEX>`. An invalid value is rejected with a message.
- A pinned slave that presents no certificate, or a different one, is refused (fail closed). The master logs `fingerprint mismatch, refusing connection`, or a message about a missing client certificate.
- If the slave is online when you set the pin, its current certificate is checked right away. If it does not match, the slave is disconnected (`Certificate fingerprint does not match the pinned fingerprint`) and is only accepted again with the pinned certificate.
- Keep `slavemanager.ssl.client_authentication` at `optional` or `required` for pinned slaves, otherwise the slave never sends a certificate.
- Pinning only works if every slave has **its own key**. Run `./genkey.sh` on each slave instead of copying one `drftpd.key` to all of them, otherwise all slaves share one fingerprint.
- Without a `fingerprint` property nothing changes: the fingerprint is only logged.

Remove a pin with:

```
SITE SLAVE <slavename> UNSET fingerprint
```

Show the slave's stored properties, including the pin, with `SITE SLAVE <slavename>`.

See also [Adding Slaves](Adding-Slaves.md), [Master Configuration](Master-Configuration.md), [Slave Configuration](Slave-Configuration.md).
