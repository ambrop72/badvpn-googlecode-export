<font size='3'><b>See also:</b> <a href='NCD.md'>NCD</a>, a programming language for network configurations</font>

# Introduction #

`tun2socks` is used to "socksify" TCP (IPv4 and IPv6) connections at the network layer. It implements a TUN virtual network interface which accepts all incoming TCP connections (regardless of destination IP), and forwards them through a SOCKS server. This allows you to forward **all** connections through SOCKS, without any need for application support. It can be used, for example, to forward connections through a remote SSH server or through [Tor](https://www.torproject.org/). Because of how it works, it can even be installed on a Linux router to transparently forward clients through SOCKS.

# Installation #

`tun2socks` is part of BadVPN. If you're on Linux, just build BadVPN with its CMake build system (or use the Gentoo package net-misc/badvpn, the Arch AUR or Ubuntu PPA packages). You can build just tun2socks without any other software in the package; this way, you don't need to have the NSS and OpenSSL libraries installed:

```
mkdir badvpn-build
cd badvpn-build
cmake /path/to/badvpn -DBUILD_NOTHING_BY_DEFAULT=1 -DBUILD_TUN2SOCKS=1
make
```

Alternatively, you can use a shell script to compile `tun2socks` only, in case using CMake is a problem for you: http://badvpn.googlecode.com/svn/trunk/compile-tun2sock.sh .

If you're on Windows, simply grab the Windows build of BadVPN.

# Example (tunnelling through SSH) #

First create a TUN device:

  * On Linux, use `ip tuntap add dev tun0 mode tun user <someuser>`.
  * On Windows, [install OpenVPN](http://openvpn.net/index.php/download/community-downloads.html) (or, if you already have it, click the start menu shortcut that creates a new TAP-Win32 device). The new device will appear in `Network Adapters` and will be identifiable by its `Device Name` field (saying Tap-Win32 something).

Configure IP for the device: assign IP address 10.0.0.1, netmask 255.255.255.0.

Now start the program (on Linux, run it as `<someuser>`):
```
badvpn-tun2socks --tundev <tun_spec> --netif-ipaddr 10.0.0.2 --netif-netmask 255.255.255.0 --socks-server-addr 127.0.0.1:1080
```

where `<tun_spec>` is:

  * on Linux, `tun0`,
  * on Windows, `"tap0901:<human_name_of_TUN_device>:10.0.0.1:10.0.0.0:255.255.255.0"` (the three numbers are TUN interface IP address, network, and subnet mask).

NOTE: `--netif-ipaddr 10.0.0.2` is not a typo. It specifies the IP address of the virtual router inside the TUN device, and must be different than the IP of the TUN interface itself.

Now you should be able to ping the virtual router's IP (`10.0.0.2`).

Connect to the SSH server, passing `-D localhost:1080` to the `ssh` command to enable dynamic forwarding. This will make `ssh` open a local SOCKS server which `badvpn-tun2socks` will use.
If you use Putty, go to Connection->SSH->Tunnels, type `1080` in `Source port`, choose `Dynamic` and click `Add`.

All that remains is to route connections through the TUN device instead of the existing default gateway. This is done as follows:

  1. Add a route to the SSH server through your existing gateway, with a lower metric than the original default route.
  1. If your DNS servers are in the Internet (rather than your local network), also add routes for them (like for the SSH server). This is needed because `tun2socks` does not forward UDP by default (see below).
  1. Add default route through the virtual router in the TUN device, with a lower metric than the original default route, but higher than the SSH and DNS routes.

This will make all external connections go through the TUN device, except for the SSH connection (else SSH would go through the TUN device, which would go through... SSH).

For example (assuming there are no existing default routes with metric <=6; otherwise remove them or change their metrics), in Linux:

```
route add <IP_of_SSH_server> gw <IP_of_original_gateway> metric 5
<same for DNS>
route add default gw 10.0.0.2 metric 6
```

Or on Windows (NOTE: `tun2socks` must be running and the interface of the default gateway must be working for these to succeed):

```
route add <IP_of_SSH_server> <IP_of_original_gateway> metric 5
<same for DNS>
route add 0.0.0.0 mask 0.0.0.0 10.0.0.2 metric 6
```

These routes will not persist across a reboot. You should probably make scripts that install and remove them. You can remove a route by changing the `add` to `del` or `delete`, depending on whether you're in Linux or Windows.

**Windows 7**: This OS has problems with respecting route metrics. If after a few minutes of normal operation connections suddenly stop being routed into tun2socks and instead go out the original default gateway, for no apparent reason, take a look at [issue 5](http://code.google.com/p/badvpn/issues/detail?id=5&can=1). One workaround is to temporarily remove the original default route.

# UDP forwarding #

`tun2socks` can forward UDP, however this requires a daemon, `badvpn-udpgw` to run on the remote SSH server. To enable UDP forwarding:

  1. On the remote SSH server, start: `badvpn-udpgw --listen-addr 127.0.0.1:7300`
  1. Add the following arguments to `badvpn-tun2socks`: `--udpgw-remote-server-addr 127.0.0.1:7300`

# IPv6 support #

**NOTE:** IPv6 support is only available in the SVN repository, and is not yet in a release version.

IPv6 forwarding in tun2socks works much like IPv4 forwarding. It is enabled using the `--netif-ip6addr` command line option. For example, you can assign the address `fdfe:dcba:9876::1` to the TUN interface, and tell tun2socks to assume the address `fdfe:dcba:9876::2`, like this:

```
badvpn-tun2socks ...other..options... --netif-ip6addr fdfe:dcba:9876::2/126
```

Once this is done, you should be able to ping the virtual router inside tun2socks at `fdfe:dcba:9876::2`. To forward IPv6 through tun2socks, update your routing table appropriately.

UDP forwarding via `badvpn-udpgw` also supports IPv6. It is irrelevant whether the connection to the `badvpn-udpgw` program is made using IPv4 or IPv6, as long as it works.

# Using with Tor #

The goal here is to have all connections initiating from a virtual machine go through [Tor](https://www.torproject.org/) via tun2socks.

**NOTE**: It is hard, but not impossible, to use tun2socks with Tor on a single host without a virtual machine, since the OS would have to route tun2socks outgoing connections differently from other programs. This can be achieved using **policy routing**, but this guide does not provide any more information.

**WARNING**: software in the VM may [reveal information](https://trac.torproject.org/projects/tor/wiki/doc/TransparentProxyLeaks) about you without your knowledge. The Tor project recommends only using the [Tor Browser Bundle](https://www.torproject.org/download/download-easy.html.en) as your web browser. However, it is not possible to properly use this browser together with transparent proxying as described here. You should however at least use something like Chrome's [Incognito mode](https://support.google.com/chrome/?p=cpn_incognito); however, this is **not** equivalent to using the bundle. You can read more about the privacy features of the Tor Browser Bundle you may be missing on [in this document](https://www.torproject.org/projects/torbrowser/design/).

**NOTE**: DNS queries done by the guest will be slower than if applications were directly configured to use Tor.

The following steps show how to set transparent proxying for the virtual machine.

  1. Let's assume you use VirtualBox to run the VM, and you run Tor on the host.
  1. Set the Network Adapter type in VirtualBox to Host Only.
  1. On the host, identify the IP address of the host on the VirtualBox  interface. This IP address can found and configured in `File->Preferences->Network`. Most likely it will be `192.168.56.1`. On a Linux host, this interface will likely be called `vboxnet0`.
  1. On the host, configure Tor to provide a SOCKS and DNS server for the VM to use. Do this by adding the following options to the torrc file (use the right IP address): "`SocksListenAddress 192.168.56.1`" and "`DNSPort 192.168.56.1:53`".
  1. Start Tor on the host. If possible, verify that it's listening on  port numbers `9050` and `53` on the IP address of the VirtualBox interface (e.g. 192.168.56.1, not localhost!). On a Linux host, you can do this by running `netstat -nlp` as root.
  1. In the VM, configure the local network interface (the normal one, not TAP). Do not use DHCP; only set the IP address (e.g. `192.168.56.2`). At this point, is should be possible to ping the guest from the host, and the reverse.
  1. On the guest, create and configure the TUN device for `tun2socks`, as is described in SSH tunneling example above. Additionally, set the default gateway (`10.0.0.2`) as part of interface configuration, and use Tor's DNS server, e.g. `192.168.56.1`.
  1. Then finally start `tun2socks` in the guest, similarly to how it is done in the SSH example above. However, instead of using `127.0.0.1:8080` as the SOCKS server, use Tor's SOCKS server running on the host, e.g. `192.168.56.1:9050`. Note that you don't have to manually configure any routes; the default route on the TUN interface is all that is needed. This is because the SOCKS and DNS servers are on the local network, so you don't have to override the default tun2socks route.

All traffic from the VM should now be going through TOR. TCP connections will be intercepted by tun2socks and will be sent through Tor's SOCKS server; DNS queries will be sent by the guest's OS directly to Tor's DNS server. UDP will not work because Tor doesn't support UDP.

This configuration has been tested using a Linux host and a Windows XP guest; however, it should work with any OS combination assuming the relevant software (tor, tun2socks) is supported. In particular, you're limited to Linux and Windows guests.

**NOTE**: Tor will issue warnings that IP addresses come without hostnames: "Warning: Your application (using socks5 to port 80) is giving Tor only an IP address....". This is normal and you might be able to silence it by adding these to torrc:
```
SafeSocks 0
TestSocks 0
WarnUnsafeSocks 0
```

# Support #

If you have questions about tun2socks, you may be able to find me in #badvpn on Freenode IRC. Alternatively, you can send me an email to <ambrop7@gmail.com>. If you have found a bug or want to propose a new feature, you can open an issue on the tracker.