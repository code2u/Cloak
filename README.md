# Cloak
A Shadowsocks plugin that obfuscates the traffic as normal HTTPS traffic and disguises the proxy server as a normal webserver.

Cloak multiplexes all traffic through a fixed amount of underlying TCP connections which eliminates the TCP handshake overhead when using vanilla Shadowsocks. Cloak also provides user management, allowing multiple users to connect to the proxy server using **one single port**. It also provides QoS controls for individual users such as upload and download credit limit, as well as bandwidth control.

To external observers (such as the GFW), Cloak is completely transparent and behaves like an ordinary HTTPS server. This is done through several [cryptographic mechanisms](https://github.com/cbeuw/Cloak/wiki/Cryptographic-Mechanisms). This eliminates the risk of being detected by traffic analysis and/or active probing

This project is based on my previous project [GoQuiet](https://github.com/cbeuw/GoQuiet). The most significant improvement form GoQuiet is that there will not be new TLS handshake being done each time a client application establishes a new connection to the Shadowsocks client. This gives a siginifcant boost to webpage loading time (reduction in time ranges from 10% to 50+%, depending on the amount of content on the webpage, see [benchmarks](https://github.com/cbeuw/Cloak/wiki/Web-page-loading-benchmarks)).

## Build
Simply `make client` and `make server`. Output binary will be in the build folder.
Do `make server_pprof` if you want to access the live profiling data.

## Configuration

### Server
`WebServerAddr` is the redirection address and port when the incoming traffic is not from shadowsocks. It should correspond to the IP record of the `ServerName` set in `ckclient.json`.

`PrivateKey` is the static curve25519 Diffie-Hellman private key.

`AdminUID` is the UID of the admin user in base64.

`DatabasePath` is the path to userinfo.db. If userinfo.db doesn't exist in this directory, Cloak will create one automatically. **If Cloak is started as a Shadowsocks plugin and Shadowsocks is started with its working directory as / (e.g. starting ss-server with systemctl), you need to set this field as an absolute path to a desired folder. If you leave it as default then Cloak will attempt to create userinfo.db under /, which it doesn't have the permission to do so and will raise an error. See Issue #13.

`BackupDirPath` is the path to save the backups of userinfo.db whenever you delete a user. If left blank, Cloak will attempt to create a folder called db-backup under its working directory. This may not be desired. See notes above.

### Client
`UID` is your UID in base64.

`PublicKey` is the static curve25519 public key, given by the server admin.

`ServerName` is the domain you want to make the GFW think you are visiting.

`TicketTimeHint` is the time needed for a session ticket to expire and a new one to be generated. Leave it as the default.

`NumConn` is the amount of underlying TCP connections you want to use.

`MaskBrowser` is the browser you want to **make the GFW _think_ you are using, it has NOTHING to do with the web browser or any web application you are using on your machine**. Currently, `chrome` and `firefox` are supported.

## Setup
### For the administrator of the server
**Run this script: https://gist.github.com/cbeuw/37a9d434c237840d7e6d5e497539c1ca** or do it manually:

0. [Install and configure shadowsocks-libev on your server](https://github.com/shadowsocks/shadowsocks-libev#installation)
1. Download [the latest release](https://github.com/cbeuw/Cloak/releases) or clone and build this repo
2. Run ck-server -k. The base64 string before the comma is the **public** key to be given to users, the one after the comma is the **private** key to be kept secret
3. Run `ck-server -u`. This will be used as the AdminUID
4. Put the private key and the AdminUID you obtained previously into config/ckserver.json
5. Edit the configuration file of shadowsocks-libev (default location is /etc/shadowsocks-libev/config.json). Let `server_port` be `443`, `plugin` be the full path to the ck-server binary and `plugin_opts` be the full path to ckserver.json. If the fields `plugin` and `plugin_opts` were not present originally, add these fields to the config file.
6. Run ss-server as root (because we are binding to TCP port 443)

#### If you want to add more users
1. Run ck-server -u to generate a new UID
2. On your client, run `ck-client -a -c <path-to-ckclient.json>` to enter admin mode
3. Input as prompted, that is your ip:port of the server and your AdminUID. Enter 4 to create a new user.
4. Enter the UID in your ckclient.json as the prompted UID, enter SessionsCap (maximum amount of concurrent sessions a user can have), UpRate and DownRate (in bytes/s), UpCredit and DownCredit (in bytes) and ExpiryTime (as a unix epoch)
5. Give your **public** key and the newly generated UID to the new user

Note: the user database is persistent as it's in-disk. You don't need to add the users again each time you start ck-server.

### Instructions for clients
**Android client is available here: https://github.com/cbeuw/Cloak-android**

0. Install and configure a version of shadowsocks client that supports plugins (such as shadowsocks-libev and shadowsocks-windows)
1. Download [the latest release](https://github.com/cbeuw/Cloak/releases) or clone and build this repo
2. Obtain the public key and your UID (or the AdminUID, if you are the server admin) from the administrator of your server
3. Put the public key and the UID you obtained into config/ckclient.json
4. Configure your shadowsocks client with your server information. The field `plugin` should be the path to ck-server binary and `plugin_opts` should be the path to ckclient.json
