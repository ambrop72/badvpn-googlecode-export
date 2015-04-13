# Introduction #

The [sys.request\_server()](http://code.google.com/p/badvpn/source/browse/trunk/ncd/modules/sys_request_server.c) module in NCD provides an interface for client programs on the same system or on the network to communicate with a running NCD program. Such an interface opens many new possibilities for things to use NCD for. The long-term goal is to implement a (better) equivalent to the NetworkManager daemon completely in the NCD programming language, and also a corresponding GUI.

Currently, a simple GUI network status indicator is available which retrieves information from an NCD network configurator via the interface described here; see the [ncdgui](http://code.google.com/p/ncdgui/) project.

# Example #

Here's a small example that illustrates how NCD's request interface works.

```
process request_server {
    # Listen on a Unix socket. It is also possible to listen on TCP.
    var({"unix", "/home/ambro/request.socket"}) listen_addr;
    #var({"tcp", "127.0.0.1", "8000"}) listen_addr;

    # Start the request server. For each request, template request_handler
    # is instantiated to handle it. 
    sys.request_server(listen_addr, "request_handler", {});

    println("Up and running!");
}

template request_handler {
    # Create value objects for request data, to allow examining it.
    value(_request.data) req;

    # Create value object for parse results. The parser will write results here.
    # In case parsing fails, assume it's a bad command.
    value(["cmd":"bad"]) res;

    # Parse request data. This writes fields into the 'res' map above.
    try("parse_request", {"_caller.req", "_caller.res"}) parse;

    # Dispatch. If parsing failed, this calls template bad_request.
    res->get("cmd") cmd;
    concat(cmd, "_request") func;
    call(func, {});
}

# This handles invalid requests.
template bad_request {
    _caller._request->reply({"Error", "Bad request."});
    _caller._request->finish();
}

# This handles ping requests.
template ping_request {
    _caller._request->reply({"OK", "Pong."});
    _caller._request->finish();
}

# This handles echo requests.
template echo_request {
    alias("_caller._request") request;
    alias("_caller.res") res;

    # Reply with echo argument and finish. The argument was written to the map in 'res' by parse_echo.
    res->get("echo_arg") echo_arg;
    request->reply({"OK", echo_arg});
    request->finish();
}

template parse_request {
    alias(_arg0) req;
    alias(_arg1) res;

    # If an assertion here fails, we will be immediately stopped; see try() documentation.

    # Request must be a list...
    strcmp(req.type, "list") a;
    _try->assert(a);

    # ...with at least the first element...
    num_greater_equal(req.length, "1") a;
    _try->assert(a);

    # ...which is a string...
    req->get("0") cmd;
    strcmp(cmd.type, "string") a;
    _try->assert(a);

    # ...and is one of the supported commands.
    strcmp(cmd, "ping") is_ping;
    strcmp(cmd, "echo") is_echo;
    or(is_ping, is_echo) a;
    _try->assert(a);

    # Parse specific commands.
    concat("parse_", cmd) func;
    call(func, {});

    # Only on success, set 'cmd' in the result, else "bad" is left by default.
    res->insert("cmd", cmd);
}

template parse_ping {
    alias("_caller._try") try;
    alias("_caller.req") req;

    # Request must have zero arguments.
    num_equal(req.length, "1") a;
    try->assert(a);
}

template parse_echo {
    alias("_caller._try") try;
    alias("_caller.req") req;
    alias("_caller.res") res;

    # Request must have one argument.
    num_equal(req.length, "2") a;
    try->assert(a);

    # Remember that argument
    req->get("1") echo_arg;
    res->insert("echo_arg", echo_arg);
}

```

Whenever a request from a client is received, a new `request_handler` process is created to handle it. There, the request data sent by the client is accessed via `_request.data`, and replies can be sent using `_request.reply()` and `_request.finish()`. The request handler code uses the [value()](http://code.google.com/p/badvpn/source/browse/trunk/ncd/modules/value.c) and [try()](http://code.google.com/p/badvpn/source/browse/trunk/ncd/modules/try.c) commands to check the format of the request.

# Usage #

Requests can be issued to an NCD daemon running a request server using the standalone `badvpn-ncd-request` program (installed by the BadVPN package).

```
$ badvpn-ncd-request unix:/home/ambro/request.socket '{}'
{"Error", "Bad request."}
$ badvpn-ncd-request unix:/home/ambro/request.socket '{"ping"}'
{"OK", "Pong."}
$ badvpn-ncd-request unix:/home/ambro/request.socket '{"echo", {"Hello", "server!"}}'
{"OK", {"Hello", "server!"}}
$ badvpn-ncd-request unix:/home/ambro/request.socket '{"echo", "Please", "Respond"}'
{"Error", "Bad request."}
$ 
```

It is also possible to issue requests directly from another NCD program using the [sys.request\_client()](http://code.google.com/p/badvpn/source/browse/trunk/ncd/modules/sys_request_client.c) module.