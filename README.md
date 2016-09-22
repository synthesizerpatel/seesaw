# Seesaw v2

[![GoDoc](https://godoc.org/github.com/google/seesaw?status.svg)](https://godoc.org/github.com/google/seesaw)

Note: This is not an official Google product.

## About

Seesaw v2 is a Linux Virtual Server [(LVS)](https://linuxvirtualserver.org) based 
[load balancing](https://en.wikipedia.org/wiki/Load_balancing) platform written in 
[Go](https://golang.org).

It is capable of providing basic load balancing for servers that are on the
same network, through to advanced load balancing functionality such as [anycast](https://en.wikipedia.org/wiki/Anycast),
Direct Server Return (DSR), support for multiple VLANs and centralised
configuration.

Above all, it is designed to be reliable and easy to maintain.

## Requirements

A Seesaw v2 load balancing cluster requires two Seesaw nodes - these can be
physical machines or virtual instances. Each node must have two network
interfaces - one for the host itself and the other for the cluster VIP. All
four interfaces should be connected to the same layer 2 network.

## Prerequisites 

### Tools

- [golang](http://golang.org) >= 1.5

### Libraries

|Compile   	            |Runtime   	         |Package   	                                                                          |
|---                    |---	               |---	                                                                                  |
|  :white_check_mark:   | :white_check_mark: | [libnl](https://www.infradead.org/~tgr/libnl/)   	                                  |
|  :white_check_mark:   |                    | [Go protobuf compiler](https://github.com/golang/protobuf/protoc-gen-go/)           |
|  :white_check_mark:   |                    | [golang.org/x/crypto/ssh](http://godoc.org/golang.org/x/crypto/ssh)                  |
|  :white_check_mark:   |                    | [github.com/dlintw/goconf](http://godoc.org/github.com/dlintw/goconf)                |
|  :white_check_mark:   |                    | [github.com/golang/glog](http://godoc.org/github.com/golang/glog)                    |
|  :white_check_mark:   |                    | [github.com/golang/protobuf/proto](http://godoc.org/github.com/golang/protobuf/proto)|
|  :white_check_mark:   |                    | [github.com/dlintw/goconf](http://godoc.org/github.com/dlintw/goconf)                |
|  :white_check_mark:   |                    | [github.com/miekg/dns](http://godoc.org/github.com/miekg/dns)                        |

:grey_question: [Go protobuf compiler](https://github.com/golang/protobuf/protoc-gen-go/) is optional

## Building

On a Debian/Ubuntu style system, you can install the required tools using apt-get

```bash
  apt-get install -y \
    golang           \
    libnl-3-dev      \
    libnl-genl-3-dev \
    git              \
    make
```

Setup your `GOPATH` and `GOROOT` environment variables and update 
your PATH.

```bash
  export GOPATH=~/go
  export GOROOT=/usr/local/go
  export PATH=${GOROOT}/bin:${GOPATH}/bin:${PATH}
```

### Download all of the golang libraries required by the project

```bash
  go get -u golang.org/x/crypto/ssh
  go get -u github.com/dlintw/goconf
  go get -u github.com/golang/glog
  go get -u github.com/miekg/dns
  go get -u github.com/kylelemons/godebug/pretty
  go get github.com/google/seesaw
````

### Build [seesaw](http://github.com/google/seesaw)


```bash
  make test
  
  make install
  
  # optionally if you wish to regenerate the protobuf code, you can run:
  make proto 
```

## Installing

After `make install` has run successfully, there should be a number of
binaries in `${GOPATH}/bin` with a `seesaw_` prefix. Install these to the
appropriate locations:

    SEESAW_BIN="/usr/local/seesaw"
    SEESAW_ETC="/etc/seesaw"
    SEESAW_LOG="/var/log/seesaw"

    INIT=`ps -p 1 -o comm=`

    install -d "${SEESAW_BIN}" "${SEESAW_ETC}" "${SEESAW_LOG}"

    install "${GOPATH}/bin/seesaw_cli" /usr/bin/seesaw

    for component in {ecu,engine,ha,healthcheck,ncc,watchdog}; do
      install "${GOPATH}/bin/seesaw_${component}" "${SEESAW_BIN}"
    done

    if [ $INIT = "init" ]; then
      install "etc/init/seesaw_watchdog.conf" "/etc/init"
    elif [ $INIT = "systemd" ]; then
      install "etc/systemd/system/seesaw_watchdog.service" "/etc/systemd/system"
      systemctl --system daemon-reload
    fi
    install "etc/seesaw/watchdog.cfg" "${SEESAW_ETC}"

    # Enable CAP_NET_RAW for seesaw binaries that require raw sockets.
    /sbin/setcap cap_net_raw+ep "${SEESAW_BIN}/seesaw_ha"
    /sbin/setcap cap_net_raw+ep "${SEESAW_BIN}/seesaw_healthcheck"

The `setcap` binary can be found in the libcap2-bin package on Debian/Ubuntu.

## Configuring

Each node needs a `/etc/seesaw/seesaw.cfg` configuration file, which provides
information about the node and who its peer is. Additionally, each load
balancing cluster needs a cluster configuration, which is in the form of a
text-based protobuf - this is stored in `/etc/seesaw/cluster.pb`.

An example seesaw.cfg file can be found in
[etc/seesaw/seesaw.cfg.example](etc/seesaw/seesaw.cfg.example) - a minimal
seesaw.cfg provides the following:

- `anycast_enabled` - True if anycast should be enabled for this cluster.
- `name` - The short name of this cluster.
- `node_ipv4` - The IPv4 address of this Seesaw node.
- `peer_ipv4` - The IPv4 address of our peer Seesaw node.
- `vip_ipv4` - The IPv4 address for this cluster VIP.

The VIP floats between the Seesaw nodes and is only active on the current
master. This address needs to be allocated within the same netblock as both
the node IP address and peer IP address.

An example cluster.pb file can be found in
[etc/seesaw/cluster.pb.example](etc/seesaw/cluster.pb.example) - a minimal
`cluster.pb` contains a `seesaw_vip` entry and two `node` entries. For each
service that you want to load balance, a separate `vserver` entry is
needed, with one or more `vserver_entry` sections (one per port/proto pair),
one or more `backends` and one or more `healthchecks`. Further information
is available in the protobuf definition - see
[pb/config/config.proto](pb/config/config.proto).

On an upstart based system, running `restart seesaw_watchdog` will start (or
restart) the watchdog process, which will in turn start the other components.

### Anycast

Seesaw v2 provides full support for anycast VIPs - that is, it will advertise
an anycast VIP when it becomes available and will withdraw the anycast VIP if
it becomes unavailable. For this to work the Quagga BGP daemon needs to be
installed and configured, with the BGP peers accepting host-specific routes
that are advertised from the Seesaw nodes within the anycast range (currently
hardcoded as `192.168.255.0/24`).

## Command Line

Once initial configuration has been performed and the Seesaw components are
running, the state of the Seesaw can be viewed and controlled via the Seesaw
command line interface. Running `seesaw` (assuming `/usr/bin` is in your path)
will give you an interactive prompt - type `?` for a list of top level
commands. A quick summary:

- `config reload` - reload the cluster.pb from the current config source.
- `failover` - failover between the Seesaw nodes.
- `show vservers` - list all vservers configured on this cluster.
- `show vserver <name>` - show the current state for the named vserver.

## Troubleshooting

A Seesaw should have five components that are running under the watchdog - the
process table should show processes for:

- `seesaw_ecu`
- `seesaw_engine`
- `seesaw_ha`
- `seesaw_healthcheck`
- `seesaw_ncc`
- `seesaw_watchdog`

All Seesaw v2 components have their own logs, in addition to the logging
provided by the watchdog. If any of the processes are not running, check the
corresponding logs in `/var/log/seesaw` (e.g. `seesaw_engine.{log,INFO}`).
