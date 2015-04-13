# Introduction #
This page documents the BadVPN client, a daemon for a node in a BadVPN VPN network.
For a general description of BadVPN, see [badvpn](badvpn.md).

# Synopsis #
```
badvpn-client
    [ --help ]
    [ --version ]
    [ --logger <stdout/syslog> ]
    (logger=syslog?
        [ --syslog-facility <string> ]
        [ --syslog-ident <string> ]
    )
    [ --loglevel <0-5/none/error/warning/notice/info/debug> ]
    [ --channel-loglevel <channel-name> <0-5/none/error/warning/notice/info/debug> ] ...
    [ --threads <integer> ]
    [ --ssl --nssdb <string> --client-cert-name <string> ]
    [ --server-name <string> ]
    --server-addr <addr>
    [ --tapdev <name> ]
    [ --scope <scope_name> ] ...
    [
        --bind-addr <addr>
        (transport-mode=udp? --num-ports <num> )
        [ --ext-addr <addr / {server_reported}:port> <scope_name> ] ...
    ] ...
    --transport-mode <udp/tcp>
    (transport-mode=udp?
        --encryption-mode <blowfish/aes/none>
        --hash-mode <md5/sha1/none>
        [ --otp <blowfish/aes> <num> <num-warn> ]
        [ --fragmentation-latency <milliseconds> ]
    )
    (transport-mode=tcp?
        (ssl? [ --peer-ssl ] )
        [ --peer-tcp-socket-sndbuf <bytes / 0> ]
    )
    [ --send-buffer-size <num-packets> ]
    [ --send-buffer-relay-size <num-packets> ]
    [ --max-macs <num> ]
    [ --max-groups <num> ]
    [ --igmp-group-membership-interval <ms> ]
    [ --igmp-last-member-query-time <ms >]
    [ --allow-peer-talk-without-ssl ]
```

# Options #
The BadVPN client is configured entirely from command line.

`--help`

> Print version and command line syntax and exit.

`--version`

> Print version and exit.

`--logger <stdout/syslog>`

> Select where to log messages. Default is stdout. Syslog is not available on Windows.

`--syslog-facility <string>`

> When logging to syslog, set the logging facility. The facility name must be in lower case.

`--syslog-ident <string>`

> When logging to syslog, set the ident.

`--loglevel <0-5/none/error/warning/notice/info/debug>`

> Set the default logging level.

`--channel-loglevel <channel-name> <0-5/none/error/warning/notice/info/debug>`

> Set the logging level for a specific logging channel.

`--threads <integer>`
> Hint for the number of additional threads to use for potentionally long  computations (such  as  encryption  and OTP generation). If zero (0) (default), additional threads will be disabled and all computations will be done in the  event  loop.  If  negative (<0),  a  guess will be made, possibly based on the number of CPUs. If positive (>0), the given number of threads will be used.

`--ssl`

> Use TLS. Requires --nssdb and --server-cert-name.

`--nssdb <string>`

> When using TLS, the NSS database to use. Probably something like sql:/some/folder.

`--client-cert-name <string>`

> When using TLS, the name of the certificate to use. The certificate must be readily accessible.

`--server-name <string>`

> Set the name of the server used for validating the server's certificate. The server name defaults
> to the the name in the server address (or a numeric address).

`--server-addr <addr>`

> Set the address of the chat server to connect to. See below for address format.

`--tapdev <name>`

> Set the TAP device to use. See below on how to configure the device. A TAP device is a virtual card
> in the operating system, but rather than receiving from and sending frames to a piece of hardware,
> a program (this one) opens it to read from and write frames into. If the VPN network is set up correctly,
> the TAP devices on the VPN nodes will act as if they were all connected into a network switch.

`--scope <scope_name>`

> Add an address scope allowed for connecting to peers. May be specified multiple times to add multiple
> scopes. The order of the scopes is irrelevant. Note that it must actually be possible to connect
> to addresses in the given scope; when another peer binds for us to connect to, we choose the first
> external address whose scope we recognize, and do not attempt further external addresses, even if
> establishing the connection fails.

`--bind-addr <addr>`

> Add an address to allow binding on. See below for address format. When attempting to bind in order
> for some peer to connect to us, the addresses will be tried in the order they are specified. If UDP
> data transport is being used, a --num-ports option must follow to specify how many continuous ports
> to allow binding to. For the address to be useful, one or more --ext-addr options must follow.
> Note that when two peers need to establish a data connection, it is arbitrary which one will attempt
> to bind first.

`--num-ports <num>`

> When using UDP transport, set the number of continuous ports for a previously specified bind address.
> Must follow a previous --bind-addr option.

`--ext-addr <addr / {server_reported}:port> <scope_name>`

> Add an external address for a previously specified bind address. Must follow a previous --bind-addr
> option. May be specified multiple times to add multiple external addresses. See below for address
> format. Additionally, the IP address part can be {server\_reported} to use the IPv4 address as the
> server sees us. The external addresses are tried by the connecting peer in the order they are specified.
> Note that the connecting peer only attempts to connect to the first address whose scope it recognizes
> and does not try other addresses. This means that all addresses must work for be able to communicate.

`--transport-mode <udp/tcp>`

> Sets the transport protocol for data connections. UDP is recommended and works best for most networks.
> TCP can be used instead if the underlying network has high packet loss which your virtual network
> cannot tolerate. Must match on all peers.

`--encryption-mode <blowfish/aes/none>`

> When using UDP transport, sets the encryption mode. None means no encryption, other options mean
> a specific cipher. Note that encryption is only useful if clients use TLS to connect to the server.
> The encryption mode must match on all peers.

`--hash-mode <md5/sha1/none>`

> When using UDP transport, sets the hashing mode. None means no hashes, other options mean a specific
> type of hash. Note that hashing is only useful if encryption is used as well. The hash mode must
> match on all peers.

`--otp <blowfish/aes> <num> <num-warn>`

> When using UDP transport, enables one-time passwords. The first argument specifies a block cipher
> used to generate passwords from a seed. The second argument specifies how many passwords are
> generated from a single seed. The third argument specifies after how many passwords used up for
> sending packets an attempt is made to negotiate a new seed with the other peer. num must be >0,
> and num-warn must be >0 and <=num. The difference (num - num-warn) should be large enough to allow
> a new seed to be negotiated before the sender runs out of passwords. Negotiating a seed involves
> the sending peer sending it to the receiving peer via the server and the receiving peer confirming
> it via the server. Note that one-time passwords are only useful if clients use TLS to connect to the
> server. The OTP option must match on all peers, except for num-warn.

`--fragmentation-latency <milliseconds>`

> When using UDP transport, sets the maximum latency to sacrifice in order to pack frames into data
> packets more efficiently. If it is >=0, a timer of that many milliseconds is used to wait for further
> frames to put into an incomplete packet since the first chunk of the packet was written. If it is
> <0, packets are sent out immediately. Defaults to 0, which is the recommended setting.

`--peer-ssl`

> When using TCP transport, enables TLS for data connections. Requires using TLS for server connection.
> For this to work, the peers must trust each others' cerificates, and the cerificates must grant the
> TLS server usage context. This option must match on all peers.

`--peer-tcp-socket-sndbuf <bytes / 0>`
> Sets the value of the SO\_SNDBUF socket option for peer TCP sockets (zero to not set). Lower values will improve fairness when data from multiple sources (local and relaying) is being sent to a given peer, but may result in lower bandwidth if the network's bandwidth-delay product is too big.

`--send-buffer-size <num-packets>`

> Sets the minimum size of the peers' send buffers for sending frames originating from this system, in
> number of packets.

`--send-buffer-relay-size <num-packets>`

> Sets the minimum size of the peers' send buffers for relaying frames from other peers, in number of
> packets.

`--max-macs <num>`
> Sets the maximum number of MAC addresses to remember for a peer. When the number is exceeded, the least recently used slot will be reused.

`--max-groups <num>`
> Sets the maximum number of IGMP group memberships to remember for a peer. When the number is exceeded, the least recently used slot will be reused.

`--igmp-group-membership-interval <ms>`
> Sets the Group Membership Interval parameter for IGMP snooping, in milliseconds.

`--igmp-last-member-query-time <ms>`
> Sets the Last Member Query Time parameter for IGMP snooping, in milliseconds.

`--allow-peer-talk-without-ssl`
> When SSL is enabled, the clients not only connect to the server using SSL, but also exchange messages through the server through another layer of SSL. This protects the messages from attacks on the server. Older versions of BadVPN (<1.999.109), however, do not support this. This option allows older and newer clients to interoperate by not using SSL if the other peer does not support it. It does however negate the security benefits of using SSL, since the (potentionally compromised) server can then order peers not to use SSL.

# Exit code #
If initialization fails, exits with code 1. Otherwise runs until termination is requested or server connection
is broken and exits with code 1.

# Address format #
Addresses have the form `<ipaddr>`:`<port>` where `<ipaddr>` is either an IPv4 address (name or numeric), or an
IPv6 address enclosed in brackets [.md](.md) (name or numeric again).

# TAP device configuration #

To use this program, you first have to configure a TAP network device that will act as an endpoint for
the virtual network. The configuration depends on your operating system.

Note that the client program does not configure the TAP device in any way; it only reads and writes
frames from/to it. You are responsible for configuring it (e.g. putting it up and setting its IP address).

## Linux ##

You need to enable the kernel configuration option CONFIG\_TUN. If you enabled it as a module, you may
have to load it (`modprobe tun`) before you can create the device.

Then you should create a persistent TAP device for the VPN client program to open. This can be done in two ways:
```
tunctl -u <user> -t tapN
```
or
```
openvpn --mktun --user <user> --dev tapN
```
where `<user>` is the user account the client program will run as (not root!), and tapN is the name of the network interface to create.

Once the TAP device is created, pass `--tapdev tapN` to the client program to make it use this device. Note that the
device will not be preserved across a shutdown of the system; consult your OS documentaton if you want to automate
the creation or configuration of the device.

## Windows ##

Windows does not come with a TAP driver. The client program uses the TAP-Win32 driver, which is part of OpenVPN.
You need to install the OpenVPN open source (!) version, and in the installer enable at least the
`TAP Virtual Ethernet Adapter` and `Add Shortcuts to Start Menu` options.
You can get the installer at http://openvpn.net/index.php/open-source/downloads.html .

The OpenVPN installer automatically creates one TAP device on your system when it's run for the first time.
To create another device, use `Programs -> OpenVPN -> Utilities -> Add a new TAP virtual ethernet adapter`.
You may have to install OpenVPN once again to make this shortcut appear. If creation fails, you should try again by right-clicking the shortcut and choosing `Run as Administrator`.

Once you have a TAP device, you can configure it like a physical network card. You can recognize TAP devices
by their `Device Name` field.

To use the device, pass `--tapdev "<driver_name>:<interface_name>"` to the client program, where `<driver_name>` is the name of
the TAP driver (tap0901 for OpenVPN 2.1 and 2.2) (case sensitive), and `<interface_name>` is the (human) name of the TAP
network interface (e.g. `Local Area Connection 2`).

# Examples #
For examples of using BadVPN, see [Examples](Examples.md).

# See also #
[badvpn](badvpn.md), [badvpn\_server](badvpn_server.md)