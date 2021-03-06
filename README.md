# UPnP daemon

A daemon for continuously opening ports via UPnP.

## Motivation

There are quite some programs out there that need certain network ports to be
open to work properly, but do not provide the capability for opening them
automatically via UPnP. Sure, one could always argue about the security
implications that come with UPnP, but if you are willing to take the risk, it
is just annoying, that for example your webserver is not reachable from the
internet, because you forgot to open port 80, or your router rebooted and
cleared the table of open ports. Or your machine does for whatever reason not
have a static IP address, so you cannot add a consistent port mapping.

Because of this frustration, I created `upnp-daemon`, a small service written
in Rust, that will periodically check a file with your defined port mappings
and send them to your router. The main usage will be that you start it once
and let it run as a background service forever. The file with the port
mappings will be newly read in on each iteration, so you can add new mappings
on the fly.

## Installation

upnp-daemon can be installed easily through Cargo via `crates.io`:

```shell script
cargo install upnp-daemon
```

## Usage

In the most basic case, a call might look like so:

```shell script
upnp-daemon --file ports.csv
```

This will start a background process (daemon) that reads in port mappings from
a CSV file (see [config file format](#config-file-format)) every minute and
ask the appropriate routers to open those ports.

The PID of the process will be written to `/tmp/upnp-daemon.pid` and locked
exclusively, so that only one instance is running at a time. To quit it, kill
the PID that is written in this file.

Bash can do it like so:

```shell script
kill $(</tmp/upnp-daemon.pid)
```

### Foreground Operation

Some service monitors expect services to start in the foreground, so they can
handle them with their own custom functions. For this use case, you can use
the `foreground` flag, like so:

```shell script
upnp-daemon --foreground --file ports.csv
```

This will leave the program running in the foreground. You can terminate it by
issuing a `SIGINT` (Ctrl-C), for example.

### Oneshot Mode

If you just want to test your configuration, without letting the daemon run
forever, you can use the `oneshot` flag, like so:

```shell script
upnp-daemon --foreground --oneshot --file ports.csv
```

You could of course leave off the `foreground` flag, but then you will not
know when the process has finished, which could take some time, depending on
the size of the mapping file.

### Logging

If you want to activate logging to have a better understanding what the
program does under the hood, you need to set the environment variable
`RUST_LOG`, like so:

```shell script
RUST_LOG=info upnp-daemon --foreground --file ports.csv
```

To make the logger even more verbose, try to set the log level to `debug`:

```shell script
RUST_LOG=debug upnp-daemon --foreground --file ports.csv
```

Please note that it does not make sense to activate logging without using
`foreground`, since the output (stdout as well as stderr) will not be saved in
daemon mode. This might change in a future release.

## Config File Format

The format of the port mapping file is a simple CSV file, like the following
example:

```text
address;port;protocol;duration;comment
192.168.0.10;12345;UDP;60;Test 1
;12346;TCP;60;Test 2
```

Please note that the first line is mandatory at the moment, it is needed to
accurately map the fields to the internal options.

### Fields

-   address

    The IP address for which the port mapping should be added. This field can
    be empty, in which case every connected interface will be tried, until one
    gateway reports success. Useful if the IP address is dynamic and not
    consistent over reboots.
    
    Fill in an IP address if you want to add a port mapping for a foreign
    device, or if you know your machine's address and want to slightly speed
    up the process.

-   port

    The port number to open for the given IP address. Note that upnp-daemon is
    greedy at the moment, if a port mapping is already in place, it will be
    deleted and re-added with the given IP address. This might be configurable
    in a future release.

-   protocol

    The protocol for which the given port will be opened. Possible values are
    `UDP` and `TCP`.

-   duration

    The lease duration for the port mapping in seconds. Please note that some
    UPnP capable routers might choose to ignore this value, so do not
    exclusively rely on this.

-   comment

    A comment about the reason for the port mapping. Will be stored together
    with the mapping in the router.
