# Introduction #
BadVPN is a peer-to-peer VPN system. It provides a Layer 2 (Ethernet) network between
the peers (VPN network nodes). The peers connect to a central server which acts as a chat
server for them to establish direct connections between each other (data connections).
These connections are used for transferring network data (Ethernet frames).

# Features #

## Data connections ##
Peers can transfer network data either over UDP or TCP. For both there are ways of
securing the data (see below).

## IPv6 support ##
IPv6 can be used for both server connections and data connections, alongside with IPv4.
Additionally, both can be combined to allow gradual migration to IPv6.

## Address selection ##

Because NATs and firewalls are widespread, it is harder for peer-to-peer services to operate.
In general, for two computers to be able to communicate, one computer must _bind_
to one of its addresses, and the other computer must _connect_
to the computer that binded (both for TCP and UDP). In a network with point-to-point
connectivity, the connecting computer can connect to the same address as the binding computer
bound to, so it is sufficient for the binding computer to send its address to the connecting
computer. However, NATs and firewalls break point-to-point connectivity. When a network is
behind a NAT, it is, by default, impossible for computers outside of that network to connect
to computers inside the network. This is because computers inside the network have no externally
visible IP address, and only communicate with the outside world through the external IP address
of the NAT router. It is however possible to manually configure the NAT router to _forward_
a specific port number on its external IP address to a specific computer inside the network.
This makes it possible for a computer outside of the network to connect to a computer inside
a network, however, it must connect to the external address of the NAT router (rather than
the address the computer inside bound to, which is its internal address). So there needs
to be some way for the connecting peer to know what address to connect to.

BadVPN solves this problem with so-called _address scopes_.
The peer that binds must have a list of external addresses for each address it can bind to,
possibly ordered from best to worst. Each external address has its scope name. A scope name
represents part of a network from which an external address can be reached. On the other hand,
the peer that connects must have a list of scopes which it can reach. When a peer binds to an
address, it sends the other peer a list of external addresses along with scope names. That peer
than chooses the first external address whose scope it recognizes and attempts to connect to it
(if there is one).

BadVPN also allows a peer to have multiple addresses for binding to. It is possible to specify
both an IPv4 and an IPv6 address to work in a multi-protocol environment.

## Relaying ##
BadVPN can be configured to allow pairs of peers that cannot communicate directly (i.e. because of
NATs or firewalls) to relay network data through a third peer. Relaying is only attempted if
none of the two peers recognize any of the other peer's external addresses (or there are none).
For relaying to work, for each of the two peers (P1, other one P2) there must be at least one
third peer (R) that P1 it is allowed to relay through and can communicate directly with, and all
such peers R must be able to communicate directly with P2.

**NOTE:** the relay will be able to see the plaintext data being relayed, and it will cost the relay twice as much CPU as it will the relayed peers.

## IGMP snooping ##
BadVPN nodes perform IGMP snooping in order to efficiently deliver multicast frames. For example,
this makes it possible to use BadVPN as a tunnel into an IPTV network of an Internet Service Provider
for you to watch TV from wherever you want (given sufficient link quality).

## Design and code quality ##
BadVPN has great focus on code quality and reliability. BadVPN is written in the C programming
language. It is a single-threaded event-driven program. This allows for low resource usage and
fast response times. Even though C is a relatively low-level language, the programs are made of
small, highly cohesive and loosely coupled modules that are combined into a complete program on
a high level. Modules are accesed and communicate through small, simple and to-the-point interfaces.
It utilizes a flow-based design which greatly simplifies processing of data and input and output
of the programs.
Furthermore, it is based on an original programming paradigm (the job queue paradigm), which greatly simplifies developing I/O modules. You can read more about the job queue paradigm at http://code.google.com/p/badvpn/source/detail?r=58 .

## Security features ##

BadVPN contains many security features, all of which are optional. The included security
features are described here.

### TLS for client-server connections ###

It is possible for the peers to communicate with the chat server securely with TLS. It is
highly recommended that this feature is used if any security whatsoever is needed. Not
using it renders all other security features useless, since clients exchange keys
unencrypted via the server. When enabled, the chat server requires each client to identify
itself with a certificate.

BadVPN uses Mozilla's NSS library for TLS support. This means that the required certificates
and keys must be available in a NSS database. The database and certificates can be
generated with the **certutil**
command. See the [Examples](Examples.md) on how to generate and distribute the certificates.

### TLS for peer messaging ###

If TLS is being used for client-server connections, it will also be used between each pair of peers communicating via the server, on top of the TLS connections to the server. This secures the messages from the server itself. It is important because the messages may include encryption keys and other private data.

### TLS for TCP data connections ###
If TCP is used for data connections between the peers, the data connections can be secured
with TLS. This requires using TLS for client-server connections. The clients need to trust
each others' certificates to be able to connect. Additionally, each client must identify to
its peers with the same certificate it used for connecting to the server.

### Encryption for UDP data connections ###
If UDP is used for data connections, it is possible for each pair of peers to encrypt their
UDP packets with a symmetric block cipher. Note that the encryption keys are transmitted
through the server unencrypted, so for this to be useful, server connections must be secured
with TLS. The encryption aims to prevent third parties from seeing the real contents of
the network data being transfered.

### Hashes for UDP data connections ###
If UDP is used for data connections, it is possible to include hashes in packets. Note that
hashes are only useful together with encryption. If enabled, the hash is calculated on the
packet with the hash field zeroed and then written to the hash field. Hashes are calculated
and included before encryption (if enabled). Combined with encryption, hashes aim to prevent
third parties from tampering with the packets and injecting them into the network.

### One-time passwords for UDP data connections ###
If UDP is used for data connections, it is possible to include one-time passwords in packets.
Note that for this to be useful, server connections must be secured with TLS.
One-time passwords are generated from a seed value by encrypting zero data with a block cipher.
The seed contains the encryption key for the block cipher and the initialization vector.
Only a fixed number of passwords are used from a single seed. The peers exchange seeds through
the server. One-time passwords aim to prevent replay attacks.

### Control over peer communication ###
It is possible to instruct the chat server to only allow certain peers to communicate. This
will break end-to-end connectivity in the virtual network. It is useful in certain cases
to improve security, for example when the VPN is used only to allow clients to securely connect
to a central service.

# Protocol #
The protocols used in BadVPN are described in the source code in the [protocol/](http://code.google.com/p/badvpn/source/browse/#svn/trunk/protocol) directory.

# See also #
[Examples](Examples.md), [badvpn\_server](badvpn_server.md), [badvpn\_client](badvpn_client.md)