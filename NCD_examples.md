# Contents #



# Introduction #

NCD, the Network Configuration Daemon, is a daemon and programming language for configuration of network interfaces and other aspects of the operating system.

This page provides some network configuration examples using NCD. For a guide to NCD and its basic feaures, see [NCD](NCD.md).

# Multiple connections with priorities #

This is an example NCD program that works with two network interfaces, both of which may be used for Internet access.
When both are working, eth1 has priority for Internet access (e.g. if eth0 is up, but later eth1 also comes up, the configuration will be changed to use eth1 for Internet access).

```
process eth0 {
    # Set device.
    var("eth0") dev;

    # Wait for device.
    net.backend.waitdevice(dev);
    net.up(dev);
    net.backend.waitlink(dev);

    # DHCP configuration.
    # net.ipv4.dhcp() will block here until it obtaines an IP address.
    # It doesn't check the obtained address in any way,
    # so as a basic security measure, do not proceed if it is local.
    net.ipv4.dhcp(dev) dhcp;
    net.ipv4.ifnot_addr_in_network(dhcp.addr, "127.0.0.1/8");
    var(dhcp.addr) addr;
    var(dhcp.prefix) addr_prefix;
    var(dhcp.gateway) gateway;
    var(dhcp.dns_servers) dns_servers;

    # Assign IP address.
    net.ipv4.addr(dev, addr, addr_prefix);

    # Go on configuring the network.
    multiprovide("NET-eth0");
}

process eth1 {
    # Set device.
    var("eth1") dev;

    # Wait for device.
    net.backend.waitdevice(dev);
    net.up(dev);
    net.backend.waitlink(dev);

    # Static configuration.
    var("192.168.111.116") addr;
    var("24") addr_prefix;
    var("192.168.111.1") gateway;
    var({"192.168.111.14", "193.2.1.66"}) dns_servers;

    # Assign IP address.
    net.ipv4.addr(dev, addr, addr_prefix);

    # Go on configuring the network.
    multiprovide("NET-eth1");
}

process NETCONF {
    # Wait for some network connection. Prefer eth1 by putting it in front of eth0.
    multidepend({"NET-eth1", "NET-eth0"}) ifdep;

    # Alias device values.
    var(ifdep.dev) dev;
    var(ifdep.addr) addr;
    var(ifdep.addr_prefix) addr_prefix;
    var(ifdep.gateway) gateway;
    var(ifdep.dns_servers) dns_servers;

    # Add default route.
    net.ipv4.route("0.0.0.0", "0", gateway, "20", dev);

    # Configure DNS servers.
    net.dns(dns_servers, "20");
}
```

# Wireless, simple #

Create a `wpa_supplicant` configuration file, `/etc/wpa_supplicant/all.conf` for example:

```
network={
        ssid="Some Network"
        scan_ssid=1
        key_mgmt=WPA-PSK
        psk="password"
}

network={
        ssid="Other Network"
        scan_ssid=1
        key_mgmt=WPA-PSK
        psk="password"
}
```

And use this NCD program:

```
process wlan {
    # Set device.
    var("wlan0") dev;

    # Wait for device and rfkill.
    net.backend.waitdevice(dev);
    net.backend.rfkill("wlan", dev);

    # Connect to wireless network.
    # NOTE: you may need /sbin/wpa_supplicant instead of /usr/sbin/wpa_supplicant (Ubuntu for example)!
    net.backend.wpa_supplicant(dev, "/etc/wpa_supplicant/all.conf", "/usr/sbin/wpa_supplicant", {});

    # DHCP configuration.
    net.ipv4.dhcp(dev) dhcp;
    net.ipv4.ifnot_addr_in_network(dhcp.addr, "127.0.0.1/8");
    var(dhcp.addr) addr;
    var(dhcp.prefix) addr_prefix;
    var(dhcp.gateway) gateway;
    var(dhcp.dns_servers) dns_servers;

    # Assign IP address to interface.
    net.ipv4.addr(dev, addr, addr_prefix);

    # Add default route.
    net.ipv4.route("0.0.0.0", "0", gateway, "20", dev);

    # Add DNS servers.
    net.dns(dns_servers, "20");
}
```

# Wireless, network-specific IP configuration #

This program uses different IP configurations for different wireless networks. For two known networks a static IP address is used, and for others, DHCP is used. It also plays a sound when connection is established or lost.

```
process wlan {
    # Set device.
    var("wlan0") dev;

    # Wait for device and rfkill.
    net.backend.waitdevice(dev);
    net.backend.rfkill("wlan", dev);

    # Connect to wireless network.
    # NOTE: you may need /sbin/wpa_supplicant instead of /usr/sbin/wpa_supplicant (Ubuntu for example)!
    net.backend.wpa_supplicant(dev, "/etc/wpa_supplicant/all.conf", "/usr/sbin/wpa_supplicant", {}) sup;

    # wpa_supplicant exposes the 'bssid' and 'ssid' variables.
    println("Connected to wireless network: bssid=", sup.bssid, " ssid=", sup.ssid);

    # Call configuration, pass SSID as argument.
    call("wlan_config", {sup.ssid, dev}) config;
    alias("config.config") config;

    # Print configuration name.
    println("Using wireless configuration: ", config.config_name);

    # Assign IP address to interface.
    net.ipv4.addr(dev, config.addr, config.addr_prefix);

    # Add default route.
    net.ipv4.route("0.0.0.0", "0", config.gateway, "20", dev);

    # Add DNS servers.
    net.dns(config.dns_servers, "20");

    # Play a sound when connection is established and when it is lost.
    list("/usr/bin/ffplay", "-nodisp", "-autoexit", "/usr/share/sounds/freedesktop/stereo/network-connectivity-established.oga") do;
    list("/usr/bin/ffplay", "-nodisp", "-autoexit", "/usr/share/sounds/freedesktop/stereo/network-connectivity-lost.oga") undo;
    run(do, undo);
}

template wlan_config {
    # First argument in call() is SSID, second is device name.
    alias("_arg0") ssid;
    alias("_arg1") dev;

    # Compare SSID to known ones.
    val_equal(ssid, "Some Network") is_home1;
    val_equal(ssid, "Other Network") is_home2;

    # Branch.
    If (is_home1) {
        var("Home 1") config_name;
        var("192.168.8.96") addr;
        var("24") addr_prefix;
        var("192.168.8.1") gateway;
        var({"192.168.8.1"}) dns_servers;
    }
    elif (is_home2) {
        var("Home 2") config_name;
        var("192.168.6.57") addr;
        var("24") addr_prefix;
        var("192.168.6.1") gateway;
        var({"192.168.6.1"}) dns_servers;
    }
    else {
        var("Default DHCP") config_name;
        net.ipv4.dhcp(dev) dhcp;
        net.ipv4.ifnot_addr_in_network(dhcp.addr, "127.0.0.1/8");
        var(dhcp.addr) addr;
        var(dhcp.prefix) addr_prefix;
        var(dhcp.gateway) gateway;
        var(dhcp.dns_servers) dns_servers;
    } config;
}
```

# BadVPN network interface #

A VPN network interface depends on the availability of a network connection. We do a simple wired+DHCP here, as in [NCD](NCD.md). However anything could be used instead, provided that `provide("NET");` is executed when network connection is available.

```
process lan {
    # Set device.
    var("eth0") dev;

    # Wait for device, set it up, and wait for network cable.
    net.backend.waitdevice(dev);
    net.up(dev);
    net.backend.waitlink(dev);

    # DHCP configuration.
    net.ipv4.dhcp(dev) dhcp;
    net.ipv4.ifnot_addr_in_network(dhcp.addr, "127.0.0.1/8");

    # Alias IP configuration variables, to allow easy access
    # from the VPN process.
    var(dhcp.addr) addr;
    var(dhcp.prefix) addr_prefix;
    var(dhcp.gateway) gateway;
    var(dhcp.dns_servers) dns_servers;

    # Assign IP address.
    net.ipv4.addr(dev, addr, addr_prefix);

    # Add default route.
    net.ipv4.route("0.0.0.0", "0", gateway, "20", dev);

    # Configure DNS servers.
    net.dns(dns_servers, "20");

    # Go on configuring the VPN.
    # The "NET" is just an identifier for the dependency; it can
    # be any string as long as it's the same in the depend() a few
    # lines below.
    provide("NET");
}

process vpn {
    # Wait for network connection.
    depend("NET") netdep;

    # Choose TAP device name for the VPN.
    var("tap5") dev;

    # Choose bind port for the VPN.
    var("7716") bind_port;

    # Build bind address "0.0.0.0:<bind_port>".
    concat("0.0.0.0:", bind_port) bind_addr;

    # Build local address, "<local_ip>:<bind_port>".
    # We access the "addr" variable in "process lan" through the depend.
    concat(netdep.addr, ":", bind_port) local_ext_addr;

    # Set command line arguments to badvpn-client; --tapdev is provided automatically.
    list(
        "--logger", "syslog", "--syslog-ident", "badvpn-client",
        "--server-name", "<server_name>",
        "--server-addr", "<server-addr>:<port>",
        "--ssl", "--nssdb", "sql:/home/badvpn/nssdb", "--client-cert-name", "peer-<someone>",
        "--transport-mode", "udp", "--encryption-mode", "blowfish", "--hash-mode", "md5", "--otp", "blowfish", "3000", "2000",
        "--scope", "mylanscope", "--scope", "internet",
        "--bind-addr", bind_addr, "--num-ports", "20",
        "--ext-addr", local_ext_addr, "mylanscope"
    ) args;

    # Start badvpn-client. Second argument is the user account to run badvpn-client as.
    net.backend.badvpn(dev, "badvpn", "/usr/bin/badvpn-client", args);

    # Now configure IP on the VPN interface. We just assign an address.
    # DHCP can be used instead for instance.

    # Choose IP address.
    var("10.3.7.62") addr;
    var("24") addr_prefix;

    # Assign IP address.
    net.ipv4.addr(dev, addr, addr_prefix);
}
```

# Bus location based configuration #

This example shows how you can configure interfaces based on the bus location (PCI and USB only). It also prints added or removed events for all interfaces, including the bus identifier, so you can easily find the right value for your device. Only the last process is where the interface is actually configured based on bus location. Add more such processes to configure more interfaces.

```
process main {
    # Init process manager.
    process_manager() manager;

    # Wait for interface event.
    net.watch_interfaces() watcher;

    println("interface event: ", watcher.event_type, " ", watcher.devname, " ", watcher.bus);

    # Determine event type.
    val_equal(watcher.event_type, "added") is_added;
    val_equal(watcher.event_type, "removed") is_removed;

    # Dispatch.
    If (is_added) {
        var({watcher.devname, watcher.bus}) args;
        manager->start(watcher.devname, "interface", args);
    }
    elif (is_removed) {
        manager->stop(watcher.devname);
    };

    # Next event.
    watcher->nextevent();
}

template interface {
    var(_arg0) dev;
    var(_arg1) bus;

    # This template is started and stopped automatically for any interface as it comes and goes.
    # Since we only want to configure predefined interfaces based on bus name, we use provide()
    # to interface global processes dealing with those interfaces.

    # Ignore devices with unknown buses.
    val_equal(bus, "unknown") is_unknown;
    ifnot(is_unknown);

    # Build provide name.
    concat("IFACE-BUS-", bus) pname;

    # Provide.
    provide(pname);
}

process interface_1 {
    # Wait for interface by bus.
    depend("IFACE-BUS-pci:0000:06:00.0") busdep;
    var(busdep.dev) dev;

    println("interface_1 starting with name ", dev);
    rprintln("interface_1 stopped with name ", dev);

    # Here comes interface config ... For example:

    # Set device up and wait for link.
    net.up(dev);
    net.backend.waitlink(dev);

    # DHCP configuration.
    # net.ipv4.dhcp() will block here until it obtaines an IP address.
    # Note that it will only obtain the IP address, and *not* assign it;
    # we do that with a separate command below.
    net.ipv4.dhcp(dev) dhcp;

    # Check IP address - make sure it's not local.
    # If you have other reserved subnets around, check for those too.
    net.ipv4.ifnot_addr_in_network(dhcp.addr, "127.0.0.1/8");

    # Assign IP address, as obtained by DHCP.
    net.ipv4.addr(dev, dhcp.addr, dhcp.prefix);

    # Add default route, as obtained by DHCP.
    net.ipv4.route("0.0.0.0", "0", dhcp.gateway, "20", dev);

    # Configure DNS servers, as obtained by DHCP.
    net.dns(dhcp.dns_servers, "20");

    println("interface_1 up");
    rprintln("interface_1 down");
}

```