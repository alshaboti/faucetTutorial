# install BRO 
prerequisits
sudo apt-get install cmake make gcc g++ flex bison libpcap-dev libssl-dev python-dev swig zlib1g-dev

binary package
wget https://www.bro.org/downloads/bro-2.5.3.tar.gz
tar -xf bro-2.5.3.tar.gz 
cd bro-2.5.3/
./configure
make
sudo make install

export PATH=/usr/local/bro/bin:$PATH
export PREFIX=/usr/local/bro

In $PREFIX/etc/node.cfg, set veth0 as the interface to monitor 

[bro]
type=standalone
host=localhost
interface=veth0

Comment mailto in $PREFIX/etc/broctl.cfg

cd /usr/local/bro/bin
as_ns host1 ./broctl

Since this is the first-time use of the shell, perform an initial installation of the BroControl configuration:
[BroControl] > install

Then start bro instant 
[BroControl] > start

Check bro status 
[BroControl] > status
Name         Type       Host          Status    Pid    Started
bro          standalone localhost     running   15052  07 May 09:03:59

Now let's mirror office vlan traffic to BRO. We will use vlan acls. 

vlans:
    BROvlan:
        vid: 200
        description: "bro vlan"

    office:
        vid: 100
        description: "office network"
        acls_in: [mirror-acl]

acls:
    mirror-acl:
        - rule:
            actions:
                allow: true
                mirror: 1

dps:
    sw1:
        dp_id: 0x1
        hardware: "Open vSwitch"
        interfaces:
            1:
                name: "BRO"
                description: "BRO network namespace"
                native_vlan: BROvlan
            2:
                name: "host2"
                description: "host2 network namespace"
                native_vlan: office
            3:
                name: "host3"
                description: "host3 network namespace"
                native_vlan: office
 

Relaod  faucet configuration file


--------------
=================|  Broccoli Build Summary  |===================

Install prefix:    /usr/local/bro
Library prefix:    /usr/local/bro/lib
Debug mode:        false
Shared libs:       true
Static libs:       true

Config file:       /usr/local/bro/etc/broccoli.conf
Packet support:    true

CC:                /usr/bin/cc
CFLAGS:             -Wall -Wno-unused -O2 -g -DNDEBUG
CPP:               /usr/bin/cc
----------
====================|  Bro Build Summary  |=====================

Install prefix:    /usr/local/bro
Bro Script Path:   /usr/local/bro/share/bro
Debug mode:        false

CC:                /usr/bin/cc
CFLAGS:             -Wall -Wno-unused -O2 -g -DNDEBUG
CXX:               /usr/bin/c++
CXXFLAGS:           -Wall -Wno-unused -std=c++11 -O2 -g -DNDEBUG
CPP:               /usr/bin/c++

Broker:            false
Broker Python:     false
Broccoli:          true
Broctl:            true
Aux. Tools:        true

GeoIP:             false
gperftools found:  false
        tcmalloc:  false
       debugging:  false
jemalloc:          false

host1 will run BRO and host2 will run dhcp both have static ip 
192.168.0.1/24 and 192.168.0.2/24

create_ns host1 192.168.0.1/24 # BRO
create_ns host2 192.168.0.2/24 # DHCP server
create_ns host3 192.168.0.3/24 # Gateway

create_ns host4 0
create_ns host5 0


sudo ovs-vsctl add-br br0 -- set bridge br0 other-config:datapath-id=0000000000000001 \
                          -- set bridge br0 other-config:disable-in-band=true -- set bridge br0 fail_mode=secure \
                          -- add-port br0 veth-host1 -- set interface veth-host1 ofport_request=1 \
                          -- add-port br0 veth-host2 -- set interface veth-host2 ofport_request=2 \
                          -- add-port br0 veth-host3 -- set interface veth-host3 ofport_request=3 \
                          -- add-port br0 veth-host4 -- set interface veth-host4 ofport_request=4 \
                          -- add-port br0 veth-host5 -- set interface veth-host5 ofport_request=5 \
                          -- set-controller br0 tcp:127.0.0.1:6653 tcp:127.0.0.1:6654



sudo ip netns exec host2 dnsmasq --no-ping -p 0 -k  --dhcp-range=192.168.0.10,192.168.0.20  \
                                 --dhcp-option=option:router,192.168.0.3 -O option:dns-server,8.8.8.8  \
                                 -I lo -z -l /tmp/nfv-dhcp.leases -8 /tmp/nfv.dhcp.log -i veth0  --conf-file= &


/etc/faucet/faucet.yaml
```
vlans:
    BROvlan:
        vid: 200
        description: "bro vlan"

    office:
        vid: 100
        description: "office network"
#        acls_in: [mirror-acl]

acls:
    mirror-acl:
        - rule:
            actions:
                allow: true
                mirror: 1

dps:
    sw1:
        dp_id: 0x1
        hardware: "Open vSwitch"
        interfaces:
            1:
                name: "host1"
                description: "BRO network namespace"
                native_vlan: BROvlan
            2:
                name: "host2"
                description: "DHCP server  network namespace"
                native_vlan: office
            3:
                name: "host3"
                description: "gateway network namespace"
                native_vlan: office
            4:
                name: "host4"
                description: "host4 network namespace"
                native_vlan: office
            5:
                name: "host5"
                description: "host5 network namespace"
                native_vlan: office
```
now restart faucet
```
sudo systemctl restart faucet
```

as_ns host4 dhclient veth0
as_ns host5 dhclient veth0

You can check /tmp/nfv-dhcp.leases and /tmp/nfv.dhcp.log to find what ip assinged to host4 and host5. 
Alternatively, 
```
as_ns host4 ip add show
as_ns host5 ip add show
```

