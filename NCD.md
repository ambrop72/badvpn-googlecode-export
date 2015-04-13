<font size='5'><b>NEW:</b> Try NCD directly <a href='http://badvpn.googlecode.com/svn/wiki/emncd.html'>in your browser</a></font>

<font size='5'><b>NEW:</b> <a href='https://code.google.com/p/badvpn/downloads/detail?name=NCD_programming_language_thesis.pdf'>My diploma thesis</a> documenting NCD</font>

# Contents #



# Introduction #

NCD, the Network Configuration Daemon, is a programming/scripting language originally designed for configuration of network interfaces. It implements various functionalities, both high- and low-level, as built-in commands, which may be used from an NCD program wherever and for whatever purpose the user needs them. The primary feature that distinguishes NCD from common imperative scripting languages is **backtracking**, which makes writing common configuration tasks so much easier. NCD does a very good job with hotplugging in various forms, like USB network interfaces and link detection for wired devices. New features can be added by implementing commands as C-language modules using a straightforward interface. However, NCD has some features that make it possible to implement complex functionalities directly within the language without having to resort to writing C code.

NCD is only available for Linux. It is included in the BadVPN software package.

# Why ? #

If you use Linux on a desktop system, you must have heard of a program called NetworkManager. It is a program that is meant to take care of network configuration on the OS. It is designed to be extremely easy and simple to use; it has a GUI with a wireless network list, easy wireless and IP configuration, support for VPN, etc..

This all seems good. Until you want to use two VPNs at the same time. Or add some nontrivial iptables rules. Or tell it not to mess with the network interface called eth1. Or setup a bridge interface. Basically, it's a hardcoded mess, and if you want it to do something the designers haven't specifically foreseen and implemented, better go crying right away.

Other network configuration frameworks have similar problems. The Gentoo init scripts, for example, can't even do DHCP right (it times out and forgets about the interface forever)!

My conclusion was that, for more powerful and extensible network configuration, the existing systems are not suitable; a new system better suited to the needs of power users needs to be designed. So here comes NCD!

# Basics #

NCD is used by writing an NCD program (the "configuration file"), then starting the `badvpn-ncd` program as a daemon (usually at system boot time). NCD will then keep running the supplied program, until it is requested to terminate (usually at shutdown time). NCD is designed to work in the background without any need to communicate with the user, and to automatically recover from any transient problems. The goal is to make things (e.g. network access) "just work", while still allowing dynamic and complex configuration. On the other hand, NCD tries not to limit the programmer; an [IPC mechanism](NCDRequestServer.md) is available which you can use to implement a GUI.

# Complete example #

Below is an example NCD program that works with a single wired network interface and uses DHCP to obtain the IP address, default route and DNS servers. The below code, when read from top to bottom, simply specifies which operations are to be performed in order to reach the desired state. However note that each NCD program has an implicit **backtracking** semantic. Here, for instance:

  * If the DHCP lease times out, the program will **backtrack** to the `net.ipv4.dhcp` statement, effectively deconfiguring the DNS servers, default route and IP address. When the DHCP client manages to renew the leased IP address or obtain a new one, the program will continue forward again, applying the new configuration.
  * If the network cable is pulled out, the program will backtrack to the `net.backend.waitlink` statement, deconfiguring the DNS/route/address, as above, plus stopping the DHCP client. The program will continue forward when the cable is plugged back in.
  * If the eth0 device disappears (for example, it is a USB network card and is pulled out), the program will backtrack to `net.backend.waitdevice`, deconfiguring everything. The program will continue forward when the device appears again.

```
process lan {
    # Set device.
    var("eth0") dev;

    # Wait for device, set it up, and wait for network cable.
    net.backend.waitdevice(dev);
    net.up(dev);
    net.backend.waitlink(dev);

    # DHCP configuration.
    # net.ipv4.dhcp() will block here until it obtaines an IP address.
    # Note that it will only obtain the IP address, and *not* assign it;
    # we do that with a separate command below.
    net.ipv4.dhcp(dev) dhcp;

    # Check IP address - make sure it's not local.
    # If you have other reserved subnets around, check for those too.
    # The statement below will do nothing if the address does not belong to the
    # specified subnet, but if it does, it will log an error and block the
    # process at this point, preventing the assignment of the address.
    net.ipv4.ifnot_addr_in_network(dhcp.addr, "127.0.0.1/8");

    # Assign IP address, as obtained by DHCP.
    net.ipv4.addr(dev, dhcp.addr, dhcp.prefix);

    # Add default route, as obtained by DHCP.
    net.ipv4.route("0.0.0.0", "0", dhcp.gateway, "20", dev);

    # Configure DNS servers, as obtained by DHCP.
    net.dns(dhcp.dns_servers, "20");
}
```

The NCD interpreter, as an event-driven system, is able to keep track of the execution state of any number of NCD processes. For example, if you insert a duplicate of the `lan` process into the program above, with a different process name and network interface name, NCD will effectively be configuring two network interfaces, concurrently.

However NCD is still a single-threaded program, so only one process can be executing on the CPU at any given time. NCD can not only manage multiple processes, but it also allows for **interactions** between processes; some examples of interaction will be explained later (see Dependencies chapter).

# More examples #

See [NCD\_examples](NCD_examples.md) for more complex examples. NCD is capable of much more than the above program may suggest. For example, it can handle multiple network interfaces with priorities for Internet access, it can work with wireless networks and BadVPN network interfaces.

There is also an alternative introduction to NCD, and more examples, which include BadVPN interfaces: http://code.google.com/p/badvpn/source/browse/trunk/ncd/README .

# Graphical interface #

A user-friendly network configuration system employing the NCD programming language and a GUI is being worked on. See the [ncdgui](http://code.google.com/p/ncdgui/) project. However, note that this is not a GUI to NCD itself (that does not make sense, as does not making a GUI to any other programming language), but to a program written in NCD that is programmed to work with a GUI.

# Requirements #

NCD requires various programs during execution. In particular:

  * **iproute2** (`ip` command) is needed by the `net.up`, `net.ipv4.addr` and `net.ipv4.route` modules. Not all distributions come with that; Gentoo for example doesn't.

  * **udev >=171** is needed for `net.backend.waitdevice` and `net.watch_interfaces`.

# Running it #

For installation instructions see [Installation](Installation.md). On Gentoo you can just emerge net-misc/badvpn. If you want to build NCD from source without any other part of the BadVPN package (the VPN, tun2socks) and therefore without a dependency on NSS or OpenSSL, run cmake like that:

```
cmake <path_to_source> -DBUILD_NOTHING_BY_DEFAULT=1 -DBUILD_NCD=1
```

Anyway, no matter how you build it, NCD will not be linked to any library except the system libraries, avoiding any breakages resulting from library upgrades.

## Disabling existing network configuration ##

Before you start NCD, you have to stop any existing network configuration system to avoid interference.

  * Gentoo: stop `NetworkManager` init script, stop `net.` init scripts, except `net.lo`.
  * Ubuntu: stop NetworkManager using `initctl stop network-manager`

Also, NetworkManager has a habit of not deconfiguring interfaces when stopped. If you had NetworkManager running:

  * Kill dhclient: `killall dhclient`
  * Remove IP addresses: `ip addr del <addr>/<prefix> dev <iface>`
  * Set down: `ip link set <iface> down`

## Testing from command line ##

Once you're sure your interfaces are deconfigured and there is nothing that could interfere, you can try out your program from command line (as root):
```
badvpn-ncd /etc/ncd.conf
```

This will start running your program. If you hit CTRL+C (more specifically, on SIGINT or SIGTERM), the interpreter will backtrack all processes to their beginning, then exit. If your program is not behaving as expected, or you want to gain a better understanding of how it works, you can adjust the loglevel to make NCD will print status messages as it executes your program:

```
badvpn-ncd --loglevel info /etc/ncd.conf
```

## Automatically ##

If you installed BadVPN via a package manager, NCD is integrated into your distro's init system, to use `/etc/ncd.conf` as the NCD program. To use NCD by default, you will have to permanently disable existing network configurations, and have NCD start on boot instead.

To disable existing network configurations:

  * Gentoo: Disable `NetworkManager` init script. Delete `net.` init scripts (which are symlinks to `net.lo`), except `net.lo` itself, to prevent Gentoo from autoconfiguring interfaces.
  * Ubuntu: Disable NetworkManager by editing `/etc/init/network-manager.conf`, commenting the two `start on` lines, or by uninstalling the `network-manager` package.

To have NCD start on boot:

  * Gentoo: Enable the `/etc/init.d/badvpn-ncd` init script: `rc-update add badvpn-ncd default`
  * Arch: Enable the `/etc/rc.d/badvpn-ncd` init script by adding it to `DAEMONS` in `/etc/rc.conf`. Make sure it comes after syslog.
  * Ubuntu: Use the `/etc/init/badvpn-ncd.conf` Upstart script. The script is disabled by default; the `start on` line inside it is commented out. Uncomment it to have it start on boot. However be aware that `apt-get` will start it regardless when it installs the `badvpn` package (the default `/etc/ncd.conf` is a no-op to avoid damage here). You can also manually control the service using `initctl <start/stop/restart/status> badvpn-ncd`.

**NOTE**: in Gentoo, starting the NCD init script may implicitly start some other unwanted script such as `dhcpcd` or `NetworkManager`. This is because the NCD script has `need net` to make sure the local interface has been set up. To avoid starting unwanted scripts, this can be added to `/etc/conf.d/net`: `rc_net_lo_provide="net"`.

# Module documentation #

Individual statement types (modules) are described briefly in the headers of their source files, under ncd/modules/ in the BadVPN source code: http://code.google.com/p/badvpn/source/browse/#svn%2Ftrunk%2Fncd%2Fmodules .

# Support #

If you have questions about NCD, you may be able to find me or other NCD users in #ncd on Freenode IRC. Alternatively, you can send me an email to <ambrop7@gmail.com>. If you have found a bug or want to propose a new feature, you can open an issue on the tracker.

# NCD lanaguage introduction #

Here's a quick introduction to the programming language of NCD.

While the NCD language was designed for programming dynamic configurations in a partially declarative style, it has some support for imperative programming, for cases when the integrated high-level commands do not suffice. NCD is in fact [turing complete](NCDTuringMachine.md).

You can try almost all of the example programs directly [in your browser](http://badvpn.googlecode.com/svn/wiki/emncd.html), without having to install anything. Of course, commands that do any real work are not available there.

## Diagnostic output ##

The [println() and rprintln()](http://code.google.com/p/badvpn/source/browse/trunk/ncd/modules/print.c) statements provide diagnostic output to standard output. `println()` prints a message on initialization, and `rprintln()` prints a message on deinitialization.

```
process foo {
    println("Starting up, please wait...");
    rprintln("Goodbye World!");

    sleep("500", "300"); # sleeps 500ms on init and 300ms on deinit

    println("Hello World!");
    rprintln("Shutting down, please wait...");
}
```

This should result in something like this:
```
$ badvpn-ncd hello.ncd
Starting up, please wait...
< 500ms passes >
Hello World!
< you hit CTRL+C >
Shutting down, please wait...
< 300ms passes >
Goodbye World!
$ 
```

**NOTE**: these printing commands will currently block the entire interpreter, so be careful when the standard output is slow or goes via the network. Also, the commands ignore any errors writing to standard output, so they should not be used to produce files via redirection.

## Running external programs ##

The [run()](http://code.google.com/p/badvpn/source/browse/trunk/ncd/modules/run.c) statement runs some command on initialization, waiting for it to complete, and also runs another command on deinitialization. If the initialization command fails, the `run()` statement reports an error, triggering the interpreter to retry the statement after some seconds. The following example demonstrates using `run()` to manipulate Linux bridge devices using the [brctl](http://linux.die.net/man/8/brctl) program. It first creates a bridge `br0`, then proceeds to make sure that the device `eth0` is in the bridge whenever it exists (it can be a USB network interface that is plugged in and out regularly, and we want to handle this).

```
process main {
    var("br0") bridge_dev;
    var("eth0") port_dev;

    # Create the bridge (and destroy it on deinit).
    # NOTE: the full path to the executable must be provided.
    run({"/sbin/brctl", "addbr", bridge_dev},
        {"/sbin/brctl", "delbr", bridge_dev});

    # Wait for the port device to start existing.
    net.backend.waitdevice(port_dev);

    # Add port device to the bridge (and remove it when
    # it stops existing or we're quitting).
    run({"/sbin/brctl", "addif", bridge_dev, port_dev},
        {"/sbin/brctl", "delif", bridge_dev, port_dev});

    # NOTE: this is just a sample. To be useful, the bridge
    # and port also need to be set up using net.up(), and more
    # than one port needs to be added.
}
```

To add more than one device to the bridge, the `provide()` an `depend()` statements (described in the next section) should be used so we could work with each device independently of the others.

Another command that runs external programs is [runonce()](http://code.google.com/p/badvpn/source/browse/trunk/ncd/modules/runonce.c). Compared to `run()`, `runonce()` takes a single command which it runs on initialization, and the statement goes up as soon as the command terminates, successfully or not. The exit status of the command is exposed via the `exit_status` variable, which is either the non-negative exit code of the program (if the program terminated normally), or -1 (if the program terminated with a signal). When `runonce()` is requested to deinitialize, it will wait for the command to terminate if it hasn't yet (but by default it will not send any signal).

```
process main {
    runonce({"/bin/ping", "-c1", "-w5", "www.google.com"}) run;
    println("Exit status: ", run.exit_status);
}
```

In order to do something useful with the exit status, the `If` clause or other ways of branching may be used (discussed later). Also, if the above program is requested to terminate while the ping is in progress, `runonce()` will wait for the ping to complete, which may not be desired. The `term_on_deinit` option of `runonce()` can be used to make it send `SIGTERM` to the ping process and cause it to exit early.

There is one more way to run external programs from NCD: the [daemon()](http://code.google.com/p/badvpn/source/browse/trunk/ncd/modules/daemon.c) statement. Unlike `run()` and `runonce()`, `daemon()` will only spawn the child process and go up immediately without waiting for it to terminate. If the process does terminate, it will wait some time (10 seconds, currently hardcoded) and start it again; and it will do so without going down. When `daemon()` is requested to terminate, it will send SIGTERM to the child process and wait for it to terminate (unless it terminated itself and hasn't yet been restarted). The following example demonstrates using `daemon()` to start the SSH server:

```
process sshd {
        # Make sure the daemon we're starting doesn't daemonize itself
        # by forking, and instead just does its job. For sshd, the -D
        # option ensures that.
        daemon({"/usr/sbin/sshd", "-D"});
}
```

## Dependencies ##

The [provide() and depend()](http://code.google.com/p/badvpn/source/browse/trunk/ncd/modules/depend.c) statements implement dependencies.

Suppose we want to wait for a network device, and have some kind of service that works with it:

```
process foo {
    var("eth1") dev;
    net.backend.waitdevice(dev);

    println("X: started on device ", dev);
    rprintln("X: stopped on device ", dev);

    # do something with device...
}
```

However, what if, after the device is available, we want to run _two services in parallel_?

```
process foo {
    var("eth1") dev;
    net.backend.waitdevice(dev);

    provide("DEVICE");
}

process device_service_x {
    depend("DEVICE") dep;

    println("X: started on device ", dep.dev);
    rprintln("X: stopped on device ", dep.dev);

    # do something with device...
}

process device_service_y {
    depend("DEVICE") dep;

    println("Y: started on device ", dep.dev);
    rprintln("Y: stopped on device ", dep.dev);

    # do something with device...
}
```

Note how the service processes access the `dev` variable within process `foo` through the dependency. A `depend()` allows access to any variable as seen from the point of the matched `provide()`.

This is the first time we've seen **process interaction**. In the example above, process `foo` interacted with process `device_service_x` (and `device_service_y`) upon reaching `provide("DEVICE")`. In particular, the interaction was the _completion_ of the `depend("DEVICE")` statement. This interaction resulted to the CPU being handed over to the process `device_service_x` in order to continue its execution from the statement it was blocking at. However, consider that exactly the same kind of interaction happened with the other process, `device_service_y`. The question now is: what happens first? Does "X: started..." or "Y: started..." get printed first? To see the issue in its full glory, take a look at the following program:

```
process A {
    provide("GO");
    println("A print");
}

process B1 {
    depend("GO");
    println("B1 first print");
    println("B1 second print");
}

process B2 {
    depend("GO");
    println("B2 first print");
    println("B2 second print");
}
```

Can it possibly happen that the execution of the `println` statements from the two processes `B1` and `B2` will be _interleaved_? Also, when will process `A` print its message?

To explain the behavior, we need to understand that execution of an NCD program involves the continuous processing of _jobs_ within a _last-in-first-out queue_. As a job is removed from the top of the LIFO queue and executed, it may itself may push more jobs to the queue, affecting future processing. The LIFO nature of the job queue makes the order of job pushing significant, and their execution order well-defined. For example, when some jobs are pushed from inside the execution of a certain job, these will be executed in the reverse order of their pushing - and before any jobs which were already present in the queue.

Now we return to the problem of the execution order in NCD. In the above program, when a `provide("GO")` is reached, its implementation ends up pushing three jobs to the LIFO job queue:

  * one for the side effects of the completion of `depend("GO")` in process `B1`,
  * one for the side effects of the completion of `depend("GO")` in process `B2`, and
  * one for the side effects of the completion of `provide("GO")` in process `A`.

See how there's nothing special about the completion of `provide("GO")` (its "returning", but this is the wrong word to use). This is just one of the things that needs to happen at this point, along with the completion of the two related `depend("GO")` statements. It is now easy to see that the order in which the processes B1 and B2 "wake up" is determined by the order in which the implementation of `provide()` posts the jobs corresponding to the completions. In particular, the implementation first posts the job for the completion of the `provide()`, and then, in an undefined order, jobs for the completions of related `depend()` statements. Which means that first the processes B1 and B2 will be woken up, and only after they have nothing to do immediately will process A continue.

However, note that this ordering guarantee only applies when the processes that have been woken up have something to do `immediately`. For example, if we added a `sleep("1000")` into process B1 between the two `println()` commands, at the point when the sleep is reached, process A will get to continue before the sleep is finished, and execution of the two processes will effectively be interleaved.

Additionally, an out-of-memory or other error may occur at any point in the execution and trigger an implicit sleep in the process to retry the failed statement. This means that the ordering described above cannot be completely relied upon unless you have ensured sufficient memory and made sure any other errors can't happen.

All this put in simpler terms:

  * When at a certain point execution has to go multiple ways, the statement in question decides in which order those ways are taken.
  * When one way is taken, this way will be keep being processed until there is nothing else to do there at the moment.
  * Statements define (or not) in their documentation in which order the ways are taken.
  * For `provide()`, it is guaranteed that the way that corresponds to its own completion (and the continuation of the process it appears in), is taken last, after any ways for `depend()`s that have just been satisfied.

**For the curious**: You can read a more detailed description of the LIFO-queue design in my [StackOverflow answer](http://stackoverflow.com/a/10065950/1020667). If you're going to be looking at the NCD/BadVPN source code, the job queue is implemented in the _BPending_ module [here](http://code.google.com/p/badvpn/source/browse/#svn%2Ftrunk%2Fbase). The _BPendingGroup_ object is the LIFO queue itself, and _BPending_ objects are the jobs, which need to be explicitly pushed by calling _BPending\_Set()_. Additionally, jobs can be removed from the queue prior to their execution using _BPending\_Unset()_.

## Branching ##

The latest version of NCD (1.999.120) provides an `If` clause as a special kind of statement:

```
process main {
    var("anything-but-true") x;
    var("true") y;
    var("Got ") outside;

    If (x) {
        println(outside, "X");
        var("A") choice;
    } elif (y) {
        println(outside, "Y");
        var("B") choice;
    } else {
        println(outside, "none");
        var("C") choice;
    } branch;

    println("Choice: ", branch.choice);
}
```

```
$ badvpn-ncd if_example.ncd
Got Y
Choice: B
```

Note that the `If` keyword begins with a capital `I`. You can use any number of `elif` parts, and optionally an `else` part. The clause (unlike in many other languages) must be terminated with a semicolon. The condition arguments to `If` and `elif` must be strings, and are considered true only if they are equal to "true".

Code inside the clause can directly access objects above it, like the example accesses the variable `outside`. You can also give the clause a name just before the semicolon; this will allow you to access objects within the branch that was taken - the example above does this when printing `branch.choice`.

The `If` clause is internally implemented as syntax sugar around the [embcall2\_multif()](http://code.google.com/p/badvpn/source/browse/trunk/ncd/modules/call2.c) statement (read on about templates and `call()` to see how that works).

## Process templates ##

Many of NCD's execution control features rely on a feature called **process templates**. A process template is written like a process, but using the `template` keyword instead of `process`. Unlike a regular process, which starts up automatically and is terminated automatically on NCD shutdown, a process template by itself does nothing. Instead, special statements are used to dynamically create **template processes** out of process templates.

```
# Does nothing by itself.
template my_template {
    println("If I'm saying this, I'm running from a template process!");
}
```

A template process, however, is an **actual process**; its code is that of the process template is was created from. Unlike regular processes, template processes do not terminate automatically when NCD is requested to shut down; termination of a template process needs to be requested by its controlling code (often by the same statement that created it).

Because all template processes created out of the same process template have the same code, there needs to be a way to distinguish them. All statements that create template processes take a list of arguments, which is accessible from resulting template processes via `_argN` special objects. Additionally, the code creating a template process can provide other special variables.

Some statements that create template processes are `call()`, `foreach()` and `process_manager()`. Read on to learn about them.

## Calling templates ##

The effect of [call()](http://code.google.com/p/badvpn/source/browse/trunk/ncd/modules/call2.c) is mostly equivalent to embedding the body of the called process template into the place of `call()`, including correct reverse execution.

```
process foo {
    println("Saying hello...");
    call("say_hello", {});
    println("Successfully said hello!");
}

template say_hello {
    println("Hello!");
    rprintln("Goodbye...");
}
```

Additionally, with `call()`, you can:
  * Pass arguments to the template process. These can be accessed from the template process via the `_argN` special variables (N-th argument, starting with zero), and the `_args` special variable (list of arguments).
  * Access caller's objects from within the called template process via `_caller`.
  * Once the called template process has initialized, access its objects through the name of the `call()` statement.

The following example demonstrates these features.

```
process foo {
    var("Hello") x;
    call("make_msg", {"Good", "World"}) c;
    println(c.msg); # Prints: HelloGoodWorld
}

template make_msg {
    concat(_caller.x, _arg0, _arg1) msg;
}
```

Note that it is impossible to define new statements from within the NCD programming language. The only way to define a new statement is to extend NCD by implementing the statement in C language using NCD's module interface. However, when considering a new statement, you should always first try to implement the same functionality in the NCD programming language, which would usually involve using process templates in some way.

You can compare the NCD interpreter and its programming language to a CPU and its machine language. Similarly to how you can't add new machine instructions to the CPU without changing its hardware (or microcode), you can't add new statements to NCD without changing its source code. On the other hand, like you _can_ write subroutines and call them using machine instructions, you can write process templates for NCD and call them using the `call()` statement.

## Branching with call() ##

The `call()` statement can be used for branching, in addition to the `If` clause described above. You can branch using `call()` by building the name of the called template dynamically based on runtime values. The following program demonstrates this.

```
process test {
   # We will be branching based on the value of 'x' here.
   # If 'x' equals "foo", we go one way, else the other way.
   var("bar") x;

   # Produce a value which is either "true" or "false", indicating
   # which way to branch.
   strcmp(x, "foo") is_foo;

   # Build template name based on the value of is_foo.
   concat("branch_foo_", is_foo) branch_template;

   # Branch.
   call(branch_template, {}) c;

   # Print the message.
   println(c.msg);
}

template branch_foo_true {
    var("x was foo!") msg;
}

template branch_foo_false {
    var("x was NOT foo!") msg;
}
```

Branching using `call()` may sometimes be preferable to the `If` clause because the branched code is specified externally to the calling process. However, note that the `If` clause is internally translated to a statement similar to `call()`.

## Multi-way branching ##

The [choose()](http://code.google.com/p/badvpn/source/browse/trunk/ncd/modules/choose.c) statement together with `call()` can provide "if, else if, ..., else" style branching, if for some reason you want to avoid the `If` clause.

```
process foo {
    var("false") is_x;
    var("true") is_y;
    var("false") is_z;

    # If is_x then do_x, else if is_y then do_y, else if is_z then do_z, else do_other.
    choose({{is_x, "do_x"}, {is_y, "do_y"}, {is_z, "do_z"}}, "do_other") func;
    call(func, {});
}

template do_x {
    println("Doing x");
}

template do_y {
    println("Doing y");
}

template do_z {
    println("Doing z");
}

template do_other {
    println("Doing other");
}
```

## One-way branching ##

It is possible to do a one-way branch by giving `call()` `"<none>"` as the template name to make it do nothing, possibly using `choose()`:

```
process foo {
    var("false") is_x;

    # If is_x, then call do_x, else do nothing.
    choose({{is_x, "do_x"}}, "<none>") func;
    call(func, {});
}

template do_x {
    println("Doing x");
}
```

## Foreach ##

[foreach()](http://code.google.com/p/badvpn/source/browse/trunk/ncd/modules/foreach.c) does something for each element of a list or map. It is mostly equivalent to putting multiple `call()` statements one after another, but it allows the elements to be specified dynamically.

```
process foo {
    var("World") world;
    foreach({"A", "B", "C"}, "foreach_func_list", {"Hello", "Goodbye"});
    foreach(["a":"1", "b":"2"], "foreach_func_map", {"Hello", "Goodbye"});
    println("Everyone said hello!");
}

template foreach_func_list {
    var(_arg0) hello;
    var(_arg1) goodbye;

    println(_elem, ": ", hello, _caller.world);
    rprintln(_elem, ": ", goodbye, _caller.world);
}

template foreach_func_map {
    var(_arg0) hello;
    var(_arg1) goodbye;

    println(_key, "=", _val, ": ", hello, _caller.world);
    rprintln(_key, "=", _val, ": ", goodbye, _caller.world);
}
```

This results in the following:

```
A: HelloWorld
B: HelloWorld
C: HelloWorld
a=1: HelloWorld
b=2: HelloWorld
Everyone said hello!
< you hit CTRL+C >
b=2: GoodbyeWorld
a=1: GoodbyeWorld
C: GoodbyeWorld
B: GoodbyeWorld
A: GoodbyeWorld
```

The template process created by `foreach()` for every element of the list can access the current element using `_elem`, or in case of a map using `_key` and `_val`. Additionally, like `call()`, it can access arguments passed to `foreach()`, and any variable or object `X` as seen from the point of `foreach()` via `_caller.X`.

Maps are always iterated as if they were sorted by keys; the order of the keys is [the usual value order](http://code.google.com/p/badvpn/source/browse/trunk/ncd/modules/valuemetic.c).

The template processes are managed by `foreach()` in the same way as the NCD interpreter initializes and deinitializes statements within a process. Because explaining exactly how `foreach()` behaves would require understanding the internal design of the interpreter, let's just say again that it behaves as if it was replaced with a sequence of `call()` commands, one for each element of the list or map - including reactions to statements that go down.

This means that you can use `foreach()` to acquire a set of resources at once. For example, to wait for to all network interfaces on a list to appear:

```
process foo {
    list("eth0", "eth1") ifaces;
    foreach(ifaces, "wait_device", {});

    println("all devices exits");
    rprintln("all devices no longer exist (or terminating)");

    # Configure those interfaces all at once here.
}

template wait_device {
    net.backend.waitdevice(_elem);
}
```

NCD also provides the `Foreach` clause which does not require explicitly creating a process template:

```
process main {
    var({"A", "B", "C"}) list;
    value(["a":"1", "b":"2"]) map;

    # Iterate list elements.
    Foreach (list As elem) {
        println(elem);
    };

    # Iterate list elements and indices (zero-based).
    Foreach (list As index:elem) {
        println("at ", index, ": ", elem);
    };

    # Iterate map keys.
    Foreach (map As key) {
        map->get(key) val; # use value::get() to retrieve the value
        println("key=", key, " val=", val);
    };

    # Iterate map keys and values.
    Foreach (map As key:val) {
        println("key=", key, " val=", val);
    };
}
```

The `Foreach` clause behaves very similarly to the `foreach()` statement. In particular, when `Foreach` is finished, there is a fully initialized process corresponding to each iteration step. In fact, `Foreach` is internally implemented by translating into the `foreach_emb()` clause.

## Method-like statements ##

Note how the arguments of statements in NCD are restricted to plain data, and there is no concept of a reference to an object. Sometimes, however, it is needed for two or more statements to cooperate in their implementation, and you might need to tell one statement which existing related statement to cooperate with.

NCD solves this with **method-like statements**. These statements are just like regular statements, except that they refer to some existing statement, and their implementation uses this information somehow. A method-like statement is written by preceding the method name with an object identifier and an arrow `->`. The following example demonstrates the [list](http://code.google.com/p/badvpn/source/browse/trunk/ncd/modules/list.c)::contains() statement.

```
process foo {
    list("First", "Second", "Third") l;
    l->contains("Second") c_second;
    println(c_second); # Prints: true
    l->contains("Fourth") c_fourth;
    println(c_fourth); # Prints: false
}
```

Note that the `list::contains()` statements here only use the referred list object when they initialize, and not after that. In general, however, method-like statements can cooperate with the referred object however long they like; they can also go up and down like regular statements can.

The restriction on arguments being plain data applies not only to statement arguments, but on template process arguments too. For example, the following code is **incorrect**:

```
process foo {
    # Make a list object using the list() statement.
    # http://code.google.com/p/badvpn/source/browse/trunk/ncd/modules/list.c
    list("First", "Second") l;

    # Call a template, which supposedly operates on the list object (not).
    # Here, "l" really means "value of variable named (empty string) in object l".
    # The value is a list value, per documentation of list().
    call("contains", {l, "Second"}) c;

    println(c.result);
}

template contains {
    # Incorrect!
    # The object _arg0 is an internal object which exposes the value
    # of the first argument; it is not the list object from which this
    # value was obtained, and it doesn't have a contains() method.
    _arg0->contains(_arg1) result;
}
```

In this case, the error can be corrected by having the template process construct its own list object:

```
process foo {
    list("First", "Second") l;
    call("contains", {l, "Second"}) c;
    println(c.result);
}

template contains {
    listfrom(_arg0) mylist; # like list(), only it concatenates the list arguments
    mylist->contains(_arg1) result; # all good: mylist is a list object with ::contains method
}
```

On the other hand, if it's truly necessary to invoke a method-like statement on an object of the caller, this can be done by going through `_caller`:

```
process foo {
    list("First", "Second") l;
    call("contains", {"Second"}) c;
    println(c.result);
}

template contains {
    _caller.l->contains(_arg0) result;
}
```

It's easy to see that how the above approach is clumsy to work with, because the template expects the object to have a predefined name. This can be fixed using the `alias()` statement, described next.

## Aliases ##

The [alias()](http://code.google.com/p/badvpn/source/browse/trunk/ncd/modules/alias.c) statement allows a group of variables and objects to be referred to using a new name.

```
process foo {
    list("hello", "world") x;
    alias("x") y;
    concatv(y) msg;
    println(msg, y.length); # Prints: helloworld2
    y->contains("world") c; # Method calls are forwarded too.
    println(c);
}
```

Notice how the target of the alias is given as a string. When the `alias()` statement initializes, the target is not resolved in any way. In fact, all that `alias()` does is forward variable and object resolution requests by prepending the target string, plus possibly a dot, to the requested name, and resolving it from its point of view.

An important use of `alias()` is to simulate passing actual objects through a `call()` (compared to passing just data), such that the called process can invoke the object's methods. Here's the last example from the description of `call()`, fixed to allow specifying the object name:

```
process foo {
    list("First", "Second") l;
    call("contains", {"_caller.l", "Second"}) c;
    println(c.result);
}

template contains {
    alias(_arg0) passed_list;
    passed_list->contains(_arg1) result;
}
```

There is currently a small bug in the `alias()` implementation which makes it impossible to have an alias to a variable within an object:

```
process foo {
    list("first", "second") l;
    println(l.length); # 2
    alias("l.length") a;
    println(a); # error!
}
```

However, this only applies when referred name within an object (`length` in the case above) is a "final" variable, rather than just pointing to another object. E.g. it is safe to refer to objects within a `call()` statement.

In general, aliases can be used to specify the arguments a process template takes. As was just demonstrated, they can simulate passing objects, but can also serve as a more efficient alternative to `var()` for passing non-object arguments:

```
process main {
    list("foobar") list;
    call("do_something", {"foo", "bar", "_caller.list"}) c;
    println(c.result);
}

template do_something {
    alias("_arg0") s1;
    alias("_arg1") s2;
    alias(_arg2) list;

    concat(s1, s2) s;
    list->contains(s) result;
}
```

## Multi-dependencies ##

The [multiprovide() and multidepend()](http://code.google.com/p/badvpn/source/browse/trunk/ncd/modules/multidepend.c) statements implement dependencies similar to `provide()` and `depend()`. Contrary to `provide()`, which will shout errors if another `provide()` for the same name is already active, it is possible to have multiple parallel `multiprovide()`s satisfying a single `multidepend()`. On the other side, a `multidepend()` will always attempt to bind itself to one of the `multiprovide()`s that satisfy it. It will always choose the best one based on its preference list, possibly un-binding itself from an existing `multiprovide()`.

The following example illustrates the behavior of `multiprovide()` and `multidepend()`:

```
process resource1 {
    var("Resource 1") name;
    sleep("2000");
    multiprovide("RESOURCE_1");
}

process resource2 {
    var("Resource 2") name;
    sleep("4000");
    multiprovide("RESOURCE_2");
}

process dependency {
    # I want either "RESOURCE_2" or "RESOURCE_1",
    # but I always prefer the former.
    multidepend({"RESOURCE_2", "RESOURCE_1"}) dep;

    println("Bound to ", dep.name);
    rprintln("Unbound from ", dep.name);
}
```

This will result in the following:

```
< NCD starts >
< 2 seconds pass >
Bound to Resource 1
< 2 more seconds pass >
Unbound from Resource 1
Bound to Resource 2
```

As can be seen, unlike `depend()`, which only goes down when its bound `provide()` is broken, `multidepend()` also goes down when a better `multiprovide()` comes in.

Note that the namespaces for dependency names of `provide()`/`depend()` and of `multiprovide()`/`multidepend()` are separate.

## Event-reporting modules ##

Many statements in NCD can be considered to report events. However, until now, events were reported only in form of a module going up or down. Some examples of this are `net.backend.waitdevice()`, `net.ipv4.dhcp()` and `depend()`. Sometimes, however, events may need to be treated as just events, without a natural correspondence to the up/down state of a statement.

General event reporting facilities in NCD are implemented in form of a statement which behaves as follows:

  1. When an event occurs, the statement goes up and exposes the information about the event via its variables.
  1. The statement has a `::nextevent()` method that is called to indicate that the current event has been handled; this makes the event reporting statement go back down, waiting for the next event.

The following example demonstrates the [sys\_evdev()](http://code.google.com/p/badvpn/source/browse/trunk/ncd/modules/sys_evdev.c) statement, which reports events from a Linux event device (evdev).

```
process main {
    sys.evdev("/dev/input/by-id/usb-BTC_USB_Multimedia_Keyboard-event-kbd") evdev;
    println("Event: ", evdev.type, " ", evdev.value, " ", evdev.code);
    rprintln("... handled.");
    evdev->nextevent();
}
```

To run this, you have to provide an existing `-event-` (!) device, and you need read permission on the device. If all goes well, this will result in NCD spamming the console with something in form of the following (the values here correspond to what is defined in the `linux/input.h` header file):

```
Event: EV_MSC 458792 MSC_SCAN
... handled.
Event: EV_KEY 0 KEY_ENTER
... handled.
Event: EV_SYN 0 unknown
... handled.
```

Note how the above example executes in a conceptually different way to what has been demonstrated before. In previous examples, processes were meant to advance forward towards a goal, and only revert to a previous state when something goes wrong. In this case, however, the process reverts as soon as `nextevent()` is reached, deinitializing all statements between `nextevent()` and `sys.evdev()`. While it may not appear as such at first look, it is really a **loop**.

## Process manager ##

Event-reporting modules are not of much use without a useful way to handle events. The [process\_manager()](http://code.google.com/p/badvpn/source/browse/trunk/ncd/modules/process_manager.c) module provides the ability to spawn new processes in response to events. The following example demonstrates the behavior of process\_manager():

```
process foo {
    process_manager() mgr;
    rprintln("Destroying manager...");
    println("Starting A");
    mgr->start("processA", "template_for_A", {});
    println("Starting B");
    mgr->start("processB", "template_for_B", {});
    println("Started all!");
}

template template_for_A {
    println("A: starting");
    rprintln("A: died");
    sleep("1000", "1000");
    println("A: finished");
    rprintln("A: dying");
}

template template_for_B {
    println("B: starting");
    rprintln("B: died");
    sleep("2000", "2000");
    println("B: finished");
    rprintln("B: dying");
}
```

This will result in this:

```
$ badvpn-ncd manager1.ncd
Starting A
A: starting
Starting B
B: starting
Started all!
< 1 second passes >
A: finished
< 1 second passes >
B: finished
< you hit CTRL+C >
Destroying manager...
B: dying
A: dying
< 1 second passes >
A: died
< 1 second passes >
B: died
$ 
```

See how start() creates a new process and gives control to it; however, as soon as the created process cannot continue (in this case, entering sleep()), control is returned to the process that called start(). In fact, start() is done after it has spawned the process - it will not do anything from that point on, and will **not** stop the process when deinitializing. Instead, process\_manager() will stop its processes when deinitializing, waiting for them to terminate.

Processes created via process\_manager() can also be stopped explicitly using the stop() method, by providing the same process identifier as in the corresponding start() call:

```
process foo {
    process_manager() mgr;
    mgr->start("processA", "template_for_A", {});
    mgr->stop("processA");
    println("Foo done.");
}

template template_for_A {
    println("A: starting");
    rprintln("A: died");
    sleep("1000", "3000");
    println("A: started"); # never called
}
```

This will produce the following:

```
$ badvpn-ncd manager2.ncd
A: starting
Foo done.
< 3 seconds pass >
A: died
```

In this case, start() spawns the process, which proceeds to sleep(), at which point control is returned to the `foo` process. This one then  calls stop(), triggering the deinitialization of the process that was just spawned, requesting its sleep() statement to terminate. sleep() again returns control to `foo`, which prints "Foo done.". After 3 seconds, sleep() finally deinitializes.

## Handling events with process manager ##

process\_manager() can be used in combination with event reporting modules, in particular with those that report the presence of hardware devices, to automatically create processes that configure them:

```
process main {
    process_manager() mgr;
    
    # Wait for network interface event (interface added/removed).
    net.watch_interfaces() watcher;

    println("Event: interface ", watcher.devname, " ", watcher.event_type);

    # Branch based on event type.
    val_equal(watcher.event_type, "added") is_added;
    val_equal(watcher.event_type, "removed") is_removed;
    If (is_added) {
        mgr->start(dev, "interface_worker", {dev});
    }
    Elif (is_removed) {
        mgr->stop(dev);
    };

    # Finish handling this event.
    watcher->nextevent();
}

template interface_worker {
    var(_arg0) dev;

    println(dev, ": starting");
    rprintln(dev, ": died");

    # Here comes your GENERIC interface configuration code.

    regex_match(dev, "^(eth|enp)") wired_match;
    regex_match(dev, "^(wlan|wlp)") wireless_match;

    If (wired_match.succeeded) {
        # Wired configuration...
    }
    Elif (wireless_match.succeeded) {
        # Wireless configuration...
    };
}
```

This results in automatic starting and stopping of network interface configuration processes based which network interfaces exist in the system at any given time. Be aware that an `interface_worker` will be created for **all** network interfaces, including the loopback, wired and wireless interfaces. To configure them properly, branching must be used.

## Storing state ##

In NCD, simple configurations can be implemented in a declarative style without directly managing any state. To do complex things, however, it is often required to in part resort to imperative programming.

NCD provides two ways of storing state. For simple things, the [var()](http://code.google.com/p/badvpn/source/browse/trunk/ncd/modules/var.c) statement may suffice. `var()` takes as argument a single value an allows later access to this value via the empty string variable (i.e. the name of the `var()` statement). Additionally, `var()` has a `set()` method, which allows changing the stored value subsequently. The following example demonstrates listening for keyboard events while remembering the last event that has occurred, using `var()`.

```
process main {
    # Previous event is stored here.
    var("none") prev_event;

    # Wait for keyboard event.
    sys.evdev("/dev/input/by-id/usb-BTC_USB_Multimedia_Keyboard-event-kbd") evdev;

    # Build a value (list) representing the event.
    var({evdev.type, evdev.value, evdev.code}) event;

    # Print this and previous event.
    to_string(event) str1;
    to_string(prev_event) str2;
    println("Event: ", str1, " Previous: ", str2);

    # Update previous event.
    prev_event->set(event);

    # Wait for next event.
    evdev->nextevent();
}
```

This will resort in something like this:

```
$ badvpn-ncd var_state.ncd
Event: {"EV_MSC", "458792", "MSC_SCAN"} Previous: "none"
Event: {"EV_KEY", "0", "KEY_ENTER"} Previous: {"EV_MSC", "458792", "MSC_SCAN"}
Event: {"EV_SYN", "0", "unknown"} Previous: {"EV_KEY", "0", "KEY_ENTER"}
Event: {"EV_MSC", "458760", "MSC_SCAN"} Previous: {"EV_SYN", "0", "unknown"}
Event: {"EV_KEY", "1", "KEY_E"} Previous: {"EV_MSC", "458760", "MSC_SCAN"}
Event: {"EV_SYN", "0", "unknown"} Previous: {"EV_KEY", "1", "KEY_E"}
Event: {"EV_MSC", "458760", "MSC_SCAN"} Previous: {"EV_SYN", "0", "unknown"}
Event: {"EV_KEY", "0", "KEY_E"} Previous: {"EV_MSC", "458760", "MSC_SCAN"}
Event: {"EV_SYN", "0", "unknown"} Previous: {"EV_KEY", "0", "KEY_E"}
```

## Value objects ##

The other, much more powerful, method of storing state is via the [value()](http://code.google.com/p/badvpn/source/browse/trunk/ncd/modules/value.c) statement. Like `var()`, `value()` accepts a single argument, which it stores, and allows reading the stored value via the empty string variable. However, `value()` allows fine-grained access and modification of the stored value.

When a value object contains a list or a map, the `get()` method can be used to access the i-th element of the list, or the value corresponding a given key in a map. For lists and map, the value object also exports the `length` variable, indicating the number of elements.

```
process main {
    # Create a value object storing a list of three elements,
    # with the third element being a list of two.
    value({"First", "Second", {"Third.First", "Third.Second"}}) v;

    # Get the element at index 1.
    v->get("1") second;
    println(second); # Second

    # Get the element at index 2.
    v->get("2") third;

    # The result of get() is another value object, so if it is a list,
    # we can call get() on this one too.
    third->get("0") third_first;
    println(third_first); # Third.First

    # We can also check the length of a list.
    println(third.length); # 2
}
```

Note that the `get()` method will fail if the given list index or map key does not exist, and the process will not continue. To check for the existence of an element, the `try_get()` method should be used. `try_get()` is like `get()`, except that it will not fail if the element specified does not exist; rather, existence is reported via the `exists` variable. The following program demonstrates using `value()` with maps and checking for the existence of a key.

```
process main {
    # Create a value storing a map with two elements.
    value(["key1":"val1", "key2":"val2"]) v;

    # Get the value under "key1".
    v->get("key1") v1;
    println(v1.exists); # true (always, because we used get and not try_get)
    println(v1); # val1

    # Check for existence of "key3".
    v->try_get("key3") v3;
    println(v3.exists); # false

    # Branch based on existence.
    concat("exists_", v3.exists) func;
    call(func, {});
}

template exists_true {
    println("key3 exists and has the value ", _caller.v3);
}

template exists_false {
    println("key3 does not exist");

    # We must not do anything else with v3!
    # If "key3" did in fact exist, we could now use v3 as if
    # it was the result of get().
}
```

So far we have only seen how value() allows examination of existing values. However, an important feature of value() is in-place modification. The following example demonstrates some simple ways to modify value objects.

```
process main {
    # Build a list value.
    value({"A", "B", "C"}) v;

    # Insert A2 to position 1.
    v->insert("1", "A2");
    to_string(v) str;
    println(str); # {"A", "A2", "B", "C"}

    # Replace element at position 2 with B2.
    v->replace("2", "B2");
    to_string(v) str;
    println(str); # {"A", "A2", "B2", "C"}

    # Remove element at position 0.
    v->remove("0");
    to_string(v) str;
    println(str); # {"A2", "B2", "C"}

    # Maps work similarly, except that keys are used
    # in place of indices, and insert() and replace()
    # are identical.
    value(["k1":"v1", "k2":"v2"]) v;
    v->replace("k1", "v1_changed");
    to_string(v) str;
    println(str); # ["k1":"v1_changed", "k2":"v2"]
}
```

## Blockers ##

The [blocker()](http://code.google.com/p/badvpn/source/browse/trunk/ncd/modules/blocker.c) module provides a statement whose up/down state can be changed imperatively.

```
process main {
    # Create blocker. This does nothing by itself.
    blocker() blk;

    # Use blocker. In this case, this will wait forever
    # because the blocker is initialized in down state.
    blk->use();

    println("This never happens.");
}
```

```
process main {
    # Create blocker.
    blocker() blk;

    # Put the blocker into up state.
    blk->up();

    # Use blocker. This time, however, execution will
    # proceed immediately because the blocker is in up date.
    blk->use();

    println("This always happens.");
}
```

```
process main {
    # Create blocker.
    blocker() blk;

    # Wake up the process below.
    provide("start_unblock_timer");

    # Use blocker. This blocks until the process below
    # puts the blocker into up state.
    # It also goes back down when the process puts the
    # blocker back into down state.
    blk->use();

    println("This happens after 1 second.");
    rprintln("This is printed after 2 seconds.");
}

process unblock_timer {
    depend("start_unblock_timer") dep;
    sleep("1000");
    dep.blk->up();
    sleep("1000");
    dep.blk->down();
}

```

The `downup()` method can be used to atomically put the blocker down and back up. This can be used to implement a loop.

```
process main {
    # Create blocker.
    blocker() blk;

    # Set blocker to up state.
    blk->up();

    # Use blocker.
    blk->use();

    println("This happens about every 1 second.");

    # Sleep one second.
    sleep("1000");

    # Atomically put blocker down and back up.
    # This makes the use() statement go down and immediately back up,
    # restarting anything that follows it.
    blk->downup();
}
```

The following script uses [sys.watch\_directory()](http://code.google.com/p/badvpn/source/browse/trunk/ncd/modules/sys_watch_directory.c) to receive directory change notifications, and runs an update external script every time the directory changes. It delays an update 1 second after a change occurs, and also restarts the update if something changed during it. It uses a blocker to help implement these semantics.

```
process watcher {
    # Blocker object used to trigger an update.
    blocker() blk;

    # State: updating - if updater script is running
    #        updating_dirty - if something changed while
    #                         script was running
    var("false") updating;
    var("false") updating_dirty;

    # Start update process.
    spawn("updater", {});

    # Wait for directory event.
    sys.watch_directory("/home/ambro") watcher;

    # Print event details (e.g. "added somefile").
    println(watcher.filename, " ", watcher.event_type);

    # If updating is in progress, mark dirty.
    If (updating) {
        updating_dirty->set("true");
    };

    # Request update. This makes use() proceed forward.
    blk->up();

    # Wait for next event (execution moves up).
    watcher->nextevent();
}

template updater {
    # Wait for update request.
    _caller.blk->use();

    # Wait some time.
    sleep("1000", "0");

    # We're about to start update script - set updating
    # variable and mark as non dirty.
    _caller.updating_dirty->set("false");
    _caller.updating->set("true");

    println("Update started");
    runonce({"/bin/echo", "Updater speaking"}, {"keep_stdout", "keep_stderr"});
    println("Update finished");

    # No longer running update script.
    _caller.updating->set("false");

    # If something changed while script was running, restart
    # update; else wait for next update request.
    If (_caller.updating_dirty) {
        _caller.blk->downup();
    } else {
        _caller.blk->down();
    };
}
```

However, be careful: if the update script changes files inside the directory being watched, this will result in constant updating. If the script really must change files in the watched directory, the watcher process should be changed to ignore changes of files which the script touches, possibly using [regex\_match()](http://code.google.com/p/badvpn/source/browse/trunk/ncd/modules/regex_match.c).

## Separating code into files ##

Definitions of processes and templates can be separated into files, and later inserted into a specific place in a program using the `include` directive. The `include` directive may only be used at the top level and not within a process or template. The path given to the `include` directive is interpreted relative to the directory where the including file resides, unless the path is absolute.

File `program.ncd`:
```
include "helper.ncdi"

process main {
    call("helper", {});
}
```

File `helper.ncdi`:
```
template helper {
    println("Hello from helper!");
}
```

However this does not quite work when two included programs end up including the same file, because any processes or templates in that file will appear multiple times in the composed program, causing interpreter initialization failure. To solve that, the `include_guard` directive was added. For simplicity, we demonstrate the solution with a single program including the same file twice, as opposed to different included files including the same file.

File `program.ncd`:
```
include "helper.ncdi"
include "helper.ncdi"

process main {
    call("helper", {});
}
```

File `helper.ncdi`:
```
include_guard "helper"

template helper {
    println("Hello from helper!");
}
```

The parameter to `include_guard` is a string which should uniquely identify the file in which it appears. The guard system works as follows. Just after NCD parses an included file, it scans it for any `include_guard` names. If any of the guard names has already been registered, the entire file is ignored. Otherwise, it registers all the guard names and proceeds to scan the file and resolve inclusions recursively. Note that this implies that it is irrelevant where in the file and in what order the `include_guard` directives appear.

## Callbacks in generic code ##

In generic code it is often needed to call back user-provided code. Suppose we want a template `call_times` which calls a user-provided template N times, where N is an argument to `call_times`. Of course, this can easily be done using the `call()` statement.

```
template call_times {
    alias("_arg0") callback_template;
    alias("_arg1") times;

    # Build a list of `times` elements which we will use in Foreach.
    value({}) list;
    var(times) i;
    backtrack_point() point;
    num_greater(i, "0") do_more;
    If (do_more) {
        list->insert(list.length, i);
        num_subtract(i, "1") new_i;
        i->set(new_i);
        point->go();
    };

    # Do the call()s.
    Foreach (list As elem) {
        call(callback_template, {});
    };
}

process main {
    call("call_times", {"callback", "10"});
    exit("0");
}

template callback {
    println("Hello from callback!");
}
```

This simple example works, however a problem arises when we wish to use `_caller` from inside `callback` to access objects in `main`. Because `callback` is called from inside `call_times` and not from `main`, we would actually have to use `_caller._caller` to reach objects in `main`. This problem can be solved by modifying `call_times` to use [call\_with\_caller\_target()](http://code.google.com/p/badvpn/source/browse/trunk/ncd/modules/call2.c) instead of `call()`. In contrast to `call()`, where `_caller` inside the called template always takes us to objects above the call, `call_with_caller_target()` takes a third parameter which specifies the object that `_caller` will refer to. In our case, `call_times` has to use `_caller` as the target.

```
template call_times {
    alias("_arg0") callback_template;
    alias("_arg1") times;

    # Build a list of `times` elements which we will use in Foreach.
    value({}) list;
    var(times) i;
    backtrack_point() point;
    num_greater(i, "0") do_more;
    If (do_more) {
        list->insert(list.length, i);
        num_subtract(i, "1") new_i;
        i->set(new_i);
        point->go();
    };

    # Do the call()s.
    Foreach (list As elem) {
        # _caller.X in the called template will refer to _caller.X
        # (specified by the third argument) as seen from this point.
        call_with_caller_target(callback_template, {}, "_caller");
    };
}

process main {
    var("Hello from callback!") object_in_main;
    call("call_times", {"callback", "10"});
    exit("0");
}

template callback {
    # Here, _caller.X refers to X as seen from call("call_times"...) in main.
    println(_caller.object_in_main);
}
```

If the above seems complicated, consider that the following program prints the length of a list value twice:

```
process main {
    value({"hello", "world", "!"}) list;
    call("helper1", {});
    call_with_caller_target("helper2", {}, "list");
    exit("0");
}

template helper1 {
    println(_caller.list.length);
}

template helper2 {
    println(_caller.length);
}
```