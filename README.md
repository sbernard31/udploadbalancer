# udploadbalancer
An UDP load-balancer prototype using bcc (XDP/Bpf)

```
usage: ulb.py [-h] -vs VIRTUAL_SERVER -rs REAL_SERVER -p PORT [PORT ...]
              [-d {0,1,2,3,4}]
              ifnet

positional arguments:
  ifnet                 network interface to load balance (e.g. eth0)

optional arguments:
  -h, --help            show this help message and exit
  -vs VIRTUAL_SERVER, --virtual_server VIRTUAL_SERVER
                        <Required> Virtual server address (e.g. 10.40.0.1)
  -rs REAL_SERVER, --real_server REAL_SERVER
                        <Required> Real server address(es) (e.g. 10.40.0.1)
  -p PORT [PORT ...], --port PORT [PORT ...]
                        <Required> UDP port(s) to load balance
  -d {0,1,2,3,4}, --debug {0,1,2,3,4}
                        Use to set bpf verbosity (0 is minimal)
```
Eg : `sudo python3 ulb.py eth0 -vs 10.188.7.99 -rs 10.188.100.163 10.188.100.230  -p 5683 5684
`

# Behavior
This load balancer can be considered as a Layer-4 NAT load-balancer as it only modifies IP address.

For ingress traffic : 
- we search if we have a `clientip:port/realserverip` association.
- if yes, we modify destination address (**dest NAT**) replacing **virtual IP address** by the **real server IP one**.
- if no, we pick a real server and create a new association, and do dest NAT as above.

For egress traffic :
- we search if we have an `clientip:port/realserverip` association.
- if yes, we drop the packet (as this server is not associated to this `clientip:port`)
- if no, we create a new association using the source IP address(**real server IP address**) and replacing source address(**source NAT**) by the **virtual server ip address**.

We keep this association is a large LRU map as long as possible, meaning the oldest association is only removed if LRU map is full and new association must be created.

The algorithm used is [a simple round-robin](https://github.com/sbernard31/udploadbalancer/issues/8).

:warning: All packets from the realservers to the client must go through the udp load-balancer machine/director ([like with LVS-NAT](http://www.austintek.com/LVS/LVS-HOWTO/HOWTO/LVS-HOWTO.LVS-NAT.html#NAT_default_gw)).

# Why create a new load balancer ?
In a cluster, generally the good practice is to share states between each server instances, but sometime some states can not be shared...  
E.g. a cluster of servers which can not share DTLS connection, in this case you want to always send packet from a given client to the same server to limit the number of handshakes.  
To do that you need to create **a long-lived association** between the client and the server, but most of the UDP loadbalancer are thougth to have ephemere association. Most of the time this association lifetime can be configured and you can set a large value, but here thanks to the LRU map we can keep the association as long as we can.

The other point is **server initiated communication**. We want to be able to initiate communication from a server exactly as if communication was initiated by a client. Meaning same association table is used.

# Limitation
This is a simple load-balancer and so it have some limitations :

- IPv4 support only for now.
- All traffic (ingress and egress) should be handled by the same network interface.
- All traffic should go to the same ethernet gateway (which is the case most of the time).
- Does not support IP fragmentation.
- Does not support IP packet with header options for now. Meaning IP header size (number of 32 bits word) must be set to 5.

# Requirements & dependencies
You need : 
 - a recent linux kernel to be able to launch xdp/bpf code. (currently tested with 4.19.x package)
 - [bcc](https://github.com/iovisor/bcc) installed. (currently tested with v0.8 : package [python3-bpfcc](https://packages.debian.org/search?suite=all&section=all&arch=any&searchon=names&keywords=python3-bpfcc) on debian)
 - linux-headers installed to allow bcc to compile bpf code.
 
# XDP/Bpf

Why XDP/Bpf ? [Why is the kernel community replacing iptables with BPF?](https://cilium.io/blog/2018/04/17/why-is-the-kernel-community-replacing-iptables/).

Read about XDP/Bpf : [Dive into BPF: a list of reading material](https://qmonnet.github.io/whirl-offload/2016/09/01/dive-into-bpf/).

Inspirations :
- [bcc example / xdp drop_count](https://github.com/iovisor/bcc/blob/master/examples/networking/xdp/xdp_drop_count.py)
- [Netronome / l4lb demo](https://github.com/Netronome/bpf-samples/tree/master/l4lb)
- [Facebook / katran](https://github.com/facebookincubator/katran)
- [xdp_load_balancer proto](https://gist.github.com/summerwind/080750455a396a1b1ba78938b3178f6b)
- [Cilium / cilium](https://github.com/cilium/cilium)

Documentations :
 - [bcc](https://github.com/iovisor/bcc/blob/master/docs/reference_guide.md)
 - [kernel version](https://github.com/iovisor/bcc/blob/master/docs/kernel-versions.md)
 - [bpf and xdp reference guide from cilium](https://cilium.readthedocs.io/en/v1.5/bpf/)

