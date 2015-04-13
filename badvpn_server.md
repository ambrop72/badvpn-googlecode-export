# Introduction #
This page documents the BadVPN server, which is used in a BadVPN VPN network by peers (**badvpn-client**) to
talk to each other in order to establish data connections. For a general description of BadVPN, see [badvpn](badvpn.md).

# Synopsis #
```
badvpn-server
    [ --help ]
    [ --version ]
    [ --logger <stdout/syslog> ]
    (logger=syslog?
        [ --syslog-facility <string> ]
        [ --syslog-ident <string> ]
    )
    [ --loglevel <0-5/none/error/warning/notice/info/debug> ]
    [ --channel-loglevel <channel-name> <0-5/none/error/warning/notice/info/debug> ] ...
    [ --listen-addr <addr> ] ...
    [ --ssl --nssdb <string> --server-cert-name <string> ]
    [ --comm-predicate <string> ]
    [ --relay-predicate <string> ]
    [ --client-socket-sndbuf <bytes / 0> ]
```

# Options #
The BadVPN server is configured entirely from command line.

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

`--listen-addr <addr>`

> Add an address for the server to listen on. See below for address format.

`--ssl`

> Use TLS. Requires --nssdb and --server-cert-name.

`--nssdb <string>`

> When using TLS, the NSS database to use. Probably something like sql:/some/folder.

`--server-cert-name <string>`

> When using TLS, the name of the certificate to use. The certificate must be readily accessible.

`--comm-predicate <string>`

> Set a predicate to define which pairs of clients are allowed to communicate. The predicate is a logical expression; see below for details. Available functions:
    1. `p1name(<string>)`
> > > true if the TLS common name of peer 1 equals the given string. If TLS is not used, the common name is assumed to be an empty string.
    1. `p1addr(<string>)`
> > > true if the IP address of peer 1 equals the given string. The string must not be a name.
    1. `p2name(<string>)`
> > > true if the TLS common name of peer 2 equals the given string. If TLS is not used, the common name is assumed to be an empty string.
    1. `p2addr(<string>)`
> > > true if the IP address of peer 2 equals the given string. The string must not be a name.

> There is no rule as to which is peer 1 and which peer 2. When the server needs to determine whether to allow two peers to communicate, it evaluates the predicate once and in no specific order.

`--relay-predicate <string>`

> Set a predicate to define how peers can relay data through other peers. The predicate is a logical expression; see below for details. If the predicate evaluates to true, peer P can relay data through peer R. Available functions:
    1. `pname(<string>)`
> > > true if the TLS common name of peer P peer equals the given string. If TLS is not used, the common name is assumed to be an empty string.
    1. `paddr(<string>)`
> > > true if the IP address of peer P equals the given string. The string must not be a name.
    1. `rname(<string>)`
> > > true if the TLS common name of peer R peer equals the given string. If TLS is not used, the common name is assumed to be an empty string.
    1. `raddr(<string>)`
> > > true if the IP address of peer R equals the given string. The string must not be a name.

`--client-socket-sndbuf <bytes / 0>`


> Sets the value of the SO\_SNDBUF socket option for client TCP sockets (zero to not set). Lower values will improve fairness when data from multiple peers is being sent to a given peer, but may result in lower bandwidth if the network's bandwidth-delay product to too big.

# Exit code #
If initialization fails, exits with code 1. Otherwise runs until termination is requested and exits with code 1.

# Address format #
Addresses have the form `<ipaddr>`:`<port>` where `<ipaddr>` is either an IPv4 address (name or numeric), or an
IPv6 address enclosed in brackets [.md](.md) (name or numeric again).

# Predicates #

The BadVPN server includes a small predicate language used to define certain policies.
Syntax and semantics of the language are described here.

  1. `true`
> > Logical true constant. Evaluates to 1.
  1. `false`
> > Logical false constant. Evaluates to 0.
  1. `NOT <expression>`
> > Logical negation. If the expression evaluates to error, the negation evaluates to error.
  1. `<expression> OR <expression>`
> > Logical disjunction. The second expression is only evaluated
> > if the first expression evaluates to false. If a sub-expression
> > evaluates to error, the disjunction evaluates to error.
  1. `<expression> AND <expression>`
> > Logical conjunction. The second expression is only evaluated
> > if the first expression evaluates to true. If a sub-expression
> > evaluates to error, the conjunction evaluates to error.
  1. `function_name(<arg>, ..., <arg>)`
> > Evaluation of a user-provided function (`function_name` is the name of the
> > function, `[a-zA-Z0-9_]+`).
> > If the function with the given name does not exist, it evaluates to
> > error.
> > Arguments are evaluated from left to right. Each argument can either
> > be a logical expression or a string (characters enclosed in double
> > quotes, without any double quote).
> > If an argument is encountered, but all needed arguments have already
> > been evaluated, the function evaluates to error.
> > If an argument is of wrong type, it is not evaluated and the function
> > evaluates to error.
> > If an argument evaluates to error, the function evaluates to error.
> > If after all arguments have been evaluated, the function needs more
> > arguments, it evaluates to error.
> > Then the handler function is called. If it returns anything other
> > than 1 and 0, the function evaluates to error. Otherwise it evaluates
> > to what the handler function returned.

# Examples #
For examples of using BadVPN, see [Examples](Examples.md).

# See also #
[badvpn](badvpn.md), [badvpn\_client](badvpn_client.md)