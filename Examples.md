# Permissions #

On Linux and other unix-like systems, it is preferable if both `badvpn-client` and `badvpn-server` programs run as seperate user accounts with limited privileges. Each program will only need read access to its corresponding NSS database (if SSL/certificates are being used), and preferably no write access to avoid accidental (or malicious) corruption of the databases.

On Gentoo, a user `badvpn` will be automatically created for `badvpn-server`, and the `badvpn-server` init script will use this user by default. However, for `badvpn-client`, no init script and user account is created; you should use [NCD](NCD.md) and the [net.backend.badvpn()](http://code.google.com/p/badvpn/source/browse/trunk/ncd/modules/net_backend_badvpn.c) command, or a custom solution to start and restart the client automatically when it exits due to connection problems. Of course you can also use NCD to start `badvpn-server` via the [daemon()](http://code.google.com/p/badvpn/source/browse/trunk/ncd/modules/daemon.c) command, which will also restart it in case it crashes (but it shouldn't).

# Setting up certificates #

If you want to use TLS for server connections (recommended), the server and all the peers will
need certificates. You may choose not to use certificates if security is not an issue, or if you just want to quickly try out BadVPN. This section explains how to generate and distribute the certificates using
NSS command line tools.

**NOTE**: the `certutil` and `pk12util` commands are part of the NSS package. On Gentoo, you need to enable the `utils` USE flag for dev-libs/nss. On Windows, the programs are shipped with the BadVPN binary.

## Setting up the Certificate Authority (CA) ##

On the system that will host the CA, create a NSS database for the CA and generate a CA certificate
valid for 24 months:

```
vpnca $ certutil -d sql:/home/vpnca/nssdb -N
vpnca $ certutil -d sql:/home/vpnca/nssdb -S -n "vpnca" -s "CN=vpnca" -t "TC,," -x -2 -v 24
> Is this a CA certificate [y/N]? y
> Enter the path length constraint, enter to skip [<0 for unlimited path]: > -1
> Is this a critical extension [y/N]? n
```

Export the public CA certificate (this file is public):

```
vpnca $ certutil -d sql:/home/vpnca/nssdb -L -n vpnca -a > ca.pem
```

## Setting up the server certificate ##

On the CA system, generate a certificate for the server valid for 24 months, with TLS server usage context:

```
vpnca $ certutil -d sql:/home/vpnca/nssdb -S -n "<insert_server_name>" -s "CN=<insert_server_name>" -c "vpnca" -t ",," -2 -6 -v 24
> 0
> -1
> Is this a critical extension [y/N]? n
> Is this a CA certificate [y/N]? n
> Enter the path length constraint, enter to skip [<0 for unlimited path]: >
> Is this a critical extension [y/N]? n
```

The `<insert_server_name>` may be a domain name of the server, or just about any word.

Export the server certificate to a PKCS#12 file (this file must be kept secret):

```
vpnca $ pk12util -d sql:/home/vpnca/nssdb -o server.p12 -n "<insert_server_name>"
```

On the system that will run the server, create a NSS database (do not use a password) and import the CA certificate
and the server cerificate (or do it on the CA system and transfer the nssdb folder):

**NOTE**: you can build these databases on any machine, and transfer them to the peer machines (e.g. you don't need to mess with the Windows command line to build them).

**WARNING**: The resulting database will contain the unencrypted private key of the peer.

```
vpnserver $ certutil -d sql:/home/vpnserver/nssdb -N
vpnserver $ certutil -d sql:/home/vpnserver/nssdb -A -t "CT,," -n "vpnca" -i /path/to/ca.pem
vpnserver $ pk12util -d sql:/home/vpnserver/nssdb -i /path/to/server.p12
```

## Setting up peer certificates ##

On the CA system, generate a certificate for the peer valid for 24 months, with TLS client and
TLS server usage contexts:

```
vpnca $ certutil -d sql:/home/vpnca/nssdb -S -n "peer-<insert_name>" -s "CN=peer-<insert_name>" -c "vpnca" -t ",," -2 -6 -v 24
> 0
> 1
> -1
> Is this a critical extension [y/N]? n
> Is this a CA certificate [y/N]? n
> Enter the path length constraint, enter to skip [<0 for unlimited path]: >
> Is this a critical extension [y/N]? n
```

Export the peer certificate to a PKCS#12 file (this file must be kept secret):

```
vpnca $ pk12util -d sql:/home/vpnca/nssdb -o peer-<insert_name>.p12 -n "peer-<insert_name>"
```

On the system that will run the VPN client, create a NSS database (do not use a password) and import the CA certificate
and the peer cerificate (or do it on the CA system and transfer the nssdb folder):

```
vpnclient $ certutil -d sql:/home/vpnclient/nssdb -N
vpnclient $ certutil -d sql:/home/vpnclient/nssdb -A -t "CT,," -n "vpnca" -i /path/to/ca.pem
vpnclient $ pk12util -d sql:/home/vpnclient/nssdb -i /path/to/peer-<insert_name>.p12
```

# Setting up TAP devices #

You need to create and configure TAP devices on all computers that will participate in the virtual network
(i.e. run the client program). See [here](http://code.google.com/p/badvpn/wiki/badvpn_client#TAP_device_configuration) for details.

# Example: Local IPv4 network (or network with full connectivity), UDP transport, zero security #

Starting the server:

```
badvpn-server
  --listen-addr 0.0.0.0:7000
```

**NOTE**: badvpn-server does not actually participate in the virtual network. If you want the server machine to be part of the network, run a local badvpn-client, like on other peers.

Starting the peers:

```
badvpn-client
  --server-addr <insert_server_local_address>:7000
  --transport-mode udp --encryption-mode none --hash-mode none
  --scope local1
  --bind-addr 0.0.0.0:8000 --num-ports 30 --ext-addr {server_reported}:8000 local1
  --tapdev tap0
```

**NOTE**: badvpn-client exits if it fails to connect to the server, or the connection is broken. For a persistent link, you should run badvpn-client in a loop that restarts it a few seconds after it exits.

On Windows, you can use a batch script for that:

```
@echo off
:loop
<badvpn-client command>
ping -n 6 127.0.0.1 >NUL
call :loop
```

# Example: Adding TLS and UDP security #

Starting the server (other options as above):

```
badvpn-server
  ...
  --ssl --nssdb sql:/home/vpnserver/nssdb --server-cert-name "<insert_server_name>"
```

Starting the peers (other options as above):

```
badvpn-client
  ...
  --server-name "<insert_server_name>"
  --ssl --nssdb sql:/home/vpnclient/nssdb --client-cert-name "peer-<insert_name>"
  --encryption-mode blowfish --hash-mode md5 --otp blowfish 3000 2000
```

You may omit the `--server-name` option if the address portion of `--server-addr` is the server name (as was specified while generating the server's certificate).

# Example: Multiple local networks behind NATs, all connected to the Internet #

For each peer in the existing local network, configure the NAT router to forward its
range of ports to it (assuming their port ranges do not overlap). The clients also need
to know the external IP address of the NAT router. If you don't have a static one,
you'll need to discover it before starting the clients. Also forward the server port to
the server.

Starting the peers in the local network where the server resides (keep all the options above and add these):

```
badvpn-client
  ...
  --scope internet
  ...
  --ext-addr <insert_NAT_routers_external_IP>:<insert_start_of_forwarded_port_range> internet
  ...
```

The `--ext-addr` option applies to the previously specified `--bind-addr` option, and must come after
the first `--ext-addr` option which specifies a local address. This is because we want to prefer local connections to Internet ones.

Now perform a similar setup in other networks behind a NAT. However:

  * Don't set up a new server, instead make the peers connect to the existing server in the first local network.
  * You can't use `{server_reported}` in the `--ext-addr` option for the local address, because the server would report the NAT router's external address rather than the peer's internal address. Instead each peer has to know its internal IP address.
  * You don't need to know the NAT router's external address; you can use `{server_reported}` in the `--ext-addr` option for the internet address. This will work because the server will see the clients connecting from the NAT router's external address.
  * Use a different scope name for this local network, e.g. "local2" instead of "local1". This should be changed in the `--scope` and `--ext-addr` options. Each local network should have a unique scope name. This will allow peers in a given local network to communicate directly.

If setup correctly, all peers will be able to communicate: those in the same local network will
communicate directly through local addresses, and those in different local networks will
communicate through the Internet.

# Relaying #

For various reasons, it may be impossible or impractical to set up port forwardings or firewall rules. For example, you're in a third party network and can't configure the router. In this case, the VPN client will not be able to communicate directly with outside peers which _also_ didn't forward ports. BadVPN provides a fallback solution for this case: it allows two peers that cannot communicate directly to relay data through a specially designated peer.

Relaying will _not_ be used by default; instead, you have to manually configure a relay peer. This is done by choosing a relay peer, which must be reachable from all peers (e.g. ports forwarded, as in the example above), and telling `badvpn-server` about the relay peer.

If you want `badvpn-server` to identify the relay peer based on its IP address (_not_ domain name):

```
badvpn-server
  ...
  --relay-predicate 'raddr("127.0.0.1")'
```

Or, if you use SSL and want to identify it based on the common name:

```
badvpn-server
  ...
  --relay-predicate 'rname("peer-<insert_name>")'
```

**WARNING**: On each peer, you _must not_ specify external addresses (`--ext-addr` options) which do not work. This applies both to internal and Internet addresses. If you specify an address which does not work, the system will _not_ figure that out; it will keep trying the non-working address, and will not try external addresses that follow nor will it fallback to relaying.

**WARNING**: The relay peer will see the raw frames being relayed (even if data links are otherwise secured).

# Integration #

In practice, specifying correct addresses may be hard, for example if you move your computer between different networks. In a perfect setup, you'd want your system to figure out in which network it resides, along with additional information, and automatically pass `badvpn-client` relevant addresses (e.g. `--server-addr`, `--bind-addr`, `--ext-addr`).

For this purpose I have developed a daemon, called the [Network Configuration Daemon](NCD.md) (Linux only) which can completely take over the network configuration of the system, and allows you to implement your desired configuration in a specially designed programming language. However, please be aware that NCD is a new project in its early stages, and may not have some particular feature you need.