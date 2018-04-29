Continues on from https://faucet.readthedocs.io/en/latest/tutorials.html
Next we are going to introduce VLANs.

ETA: ~25 mins.

We will create the following network to demonstrate the following: 
Connection within the same native VLAN.
Connection between native and tagged VLAN. 
Routing between different VLANs.
Trunk link 
ACL for a particular VLAN. 
Note: 
Hosts that are connected to the tagged vlan port require a vlan interface. Use the following script to do that.  

```
create_tagged_dev_ns () {
     NETNS=$1
     IP=$2
     VLAN=$3
     sudo ip netns exec $NETNS ip link add link veth0 name veth0.${VLAN} type vlan id $VLAN
     sudo ip netns exec $NETNS ip link set dev veth0.${VLAN} up
     sudo ip netns exec $NETNS ifconfig veth0 0
     sudo ip netns exec $NETNS ip addr add dev veth0.${VLAN}  $IP
 }
 ```
If you need to delete a host use:
```
clear_ns(){
   NETNS=$1
   sudo ip netns delete $NETNS
   sudo ovs-vsctl  del-port br0 veth-${NETNS}
}
```

Let’s start. Keep host1, host2 on the native vlan 100 (office vlan) as in the first tutorial, and add the following hosts:
For tagged vlan 100, create host3 and host4 and then create a vlan interface on each of them. 
```
create_ns  host3 0
create_ns  host4 0
create_tagged_dev_ns host3 192.168.0.3/24 100
create_tagged_dev_ns host4 192.168.0.4/24 100
```
For native vlan 200, we will create host5 and host6.
```
create_ns  host5 192.168.2.5/24
create_ns  host6 192.168.2.6/24
```
For tagged Vlan 300, create host7 and host8 and then create a vlan interface on each of them. 
```
create_ns  host7 0
create_ns  host8 0
create_tagged_dev_ns host7 192.168.3.7/24 300
create_tagged_dev_ns  host8 192.168.3.8/24 300
```
Then  connect the new hosts to the switch (br0)
```
sudo ovs-vsctl add-port br0 veth-host3 -- set interface veth-host3 ofport_request=3 -- add-port br0 veth-host4 -- set interface veth-host4 ofport_request=4 -- add-port br0 veth-host5 -- set interface veth-host5 ofport_request=5   -- add-port br0 veth-host6 -- set interface veth-host6 ofport_request=6   -- add-port br0 veth-host7 -- set interface veth-host7 ofport_request=7  -- add-port br0 veth-host8 -- set interface veth-host8 ofport_request=8   
```
