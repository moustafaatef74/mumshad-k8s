# Networking

## Prerequisites Switch Routing

So what is a network?

- We have two computers, A and B,laptops, desktops, VMs on the cloud, wherever, how does system A reach B? We connect them to a switch and the switch creates a network containing the two systems. To connect them to a switch, we need an interface on each host: physical or virtual, depending on the host.

To see the interfaces for the host

```bash
ip link
```

we then assign the systems with ip address on the same network

```bash
ip addr add 192.168.1.11/24 dev eth0
```

How does a system in one network reach a system in another?

- That where a router comes in, it's an intelligent device with many network ports. since it connects to two networks it get 2 ips assigned

When system A tries to communicate with system B how does it know where the router is on the network

- That where we configure the network with a gateway or a route they systems need to know where that door is

```bash
route
```

To configure a gateway for system A to reach system B

```bash
ip route add 192.168.2.0/24 via 192.168.1.1
```

```bash
ip route add 192.168.1.0/24 via 192.168.2.1
```

Now suppose this systems need access to the internet

- then you add the routes you need to access and connect the router to the internet

```bash
ip route add 172.217.194.9/24 via 192.168.1.1
```

there are many sites on different networks on the internet, instead o adding those ips manually

```bash
ip route add default via 192.168.1.1
```

instead of the word default you can use 0.0.0.0

### Linux Host as a router

If we have 3 hosts A, B and C

A and B are connected to a network 192.168.1.0 and B and C are connected through 192.168.2.0

A has an IP 192.168.1.5
B has 2 IPs 192.168.1.6 and 192.168.2.6
C has and IP 192.168.2.5

IF we tried to ping 192.168.2.5 from a it will be unreachable

- Host A
```bash
ping 192.168.2.5
```

We need to tell a is the route to C is through B

- Host A
```bash
ip route add 192.168.2.0/24 via 192.168.1.6
```

- Host C
```bash
ip route add 192.168.1.0/24 via 192.168.2.6
```

Now we need to configure Host B to forward the packets from A to C an vice versa

- Host B
```bash
cat /proc/sys/net/ipv4/ip_forward
```
```
0
```

```bash
echo 1 /proc/sys/net/ipv4/ip_forward
```
```
1
```

but this value does not persist changes across reboots

so you must modify the same value in the etc/sys/control.conf file.

```
net.ipv4.ip_forward = 1
```

### Take aways

```
ip link
```

- used to modify interfaces on the host

```
ip addr
```

- used to see the ip addresses assigned

```
ip addr add 192.168.1.10/24 dev eth0
```

- is to set ip addresses on interfaces

If you want to persist these changes, you must set them in the etc/network/interfaces file.

```
ip route
route
```

- is used to view the routing table

```
ip route add
```

- is used to add entires to the routing table

```
cat /proc/sys/net/ipv4/ip_forward
```

- is used to check if IP forwarding is enabled

##  Prerequisites DNS

We have two computers, A and B, both part of the same network, and they've been assigned with IP addresses 192.168.1.10, and 1.11.

You're able to ping one computer from the other using the other computer's IP address.

You know that system B has DB service on it so instead of remembering host B IP you decide to give it a name DB

Going forward you'd like to ping host be using the name db

```
ping db
```

How do you configure that?

- you want to tell system A that when you say db you mean IP address 192.168.1.11

```bash
cat >> /etc/hosts
```
```
192.168.1.11    db
```

We told system A that the IP at 192.168.1.11 is a host named db. Host A takes that for granted. Whatever we put in the /etc/hosts file is the source of truth for host A, but that may not be the truth. Host A does not check to make sure if system B's actual name is db. For instance, running a host name command on system B

- Host B
```bash
hostname
```
```
host-2
```

Managing these entries is too cumbersome, so we decided to move all these entries in one place called a dns server

our dns host has IP of 192.168.1.100

every host has a DNS resolution file at /etc/resolv.conf

you add and entry to it specifying the DNS server

```bash
cat /etc/resolv.conf
```
```
nameserver  192.168.1.100
```

You now don't need a entries in the etc/hosts file, but that does not mean you can't have entries in the hosts file

For example when testing something locally on a specific machine for example say you were to provision a test server for your own needs.

What if you have an entry in both places?

- first the host looks in the local /etc/hosts file then looks at the name server. but that could be changes

That order is defined by an entry in the file /etc/nsswitch.conf.

```bash
cat /etc/nsswitch.conf
```
```
hosts:  files dns
```

as you can see it first has files then dns, files refers to the /etc/hosts file, however this could be modified by changing the order in /etc/nsswitch.conf file

What if you try to ping www.facebook.com?

- We don't have this entry in either files, so when we try to ping www.facebook.com it will fail

we can add another entry in /etc/resolv.conf file pointing to a known name server

```bash
cat >> /etc/resolv.conf
```
```
nameserver  192.168.1.100
nameserver  8.8.8.8
```

### What to do when you want to configure web to resolve web.mycompany.com?

- for that you make an entry in your /etc/resolv.conf called search and specify the domain name you want to append

```
nameserver  192.168.1.100
search  mycompany.com
```

- next time you ping web, you will see it tries web.mycompany.com
- your host is intelligent enough to exclude the search domain if you specified a query like this

```
ping web.mycompany.com
```

### Record types

- A -> when you map ipv4 record to a host name
- AAAA -> when you map ipv6 record to a host name
- CNAME -> when you map a host to name to another host name

Ping might not be the best tool to test DNS resolution

```
nslookup www.google.com
```

but nslookup does not consider entries in the /etc/hosts files

nslookup only queries your dns server

Same goes with dig

```
dig www.google.com
```

## Prerequisite - Network namespaces in Linux

Containers are separated from the underlying host using network namespaces

For example, network namespaces are like rooms in a house, each child can only see what's in his/her room but cannot see what is happening outside, however as a parent you have visibility into all rooms in the house, if you wish you can establish connectivity between 2 rooms in the house

When you create a container you want to make sure it's isolated, that it does not see any other processes on the host, so we create a special room for it on our host using a namespace, as far as the container is concerned it only sees the processes run by it, and it thinks that it is on it's own host, the underlying host, however, has visibility to all of the processes including those running inside the containers.

When it comes to networking, our host has its own interfaces that connects to the local area network.

Our host has its own routing and ARP tables with info about the rest of the network, when the container is created we create a network namespace for it that way it has no visibility to any network related information on the host

within its namespace, the container can have its own virtual interfaces, routing and ARP table

to create a new network namespace

```
ip netns add red
ip netns add blue
```

to list the network namespacces

```
ip netns
```

to list the interfaces on the host

```
ip link
```

Now how to view the same within the network name we created?

```
ip netns exec red ip link
```

another way to do it is to add -n flag specifying the namespace you want to run this command it

```
ip -n red link
```

the same is true for the ARP table

if run the arp command on the host you see a list of entries

```
arp
```

```
ip netns exec red arp
```

and the same for routing tables

```
route
```

```
ip netns exec red route
```

so as of now these network namespaces have no network connectivity, they have no interfaces of their own and they cannot see the underlying host network

just like how we would connect two physical machines through a cable to an internet interface on each machine, you can connect two namespaces together using virtual ethernet pair, or a virtual cable, it's often referred to as a pipe

to create a pipe (virtual cable)

```
ip link add veth-red type veth peer name veth-blue
```

the next step is to attach each interface to the corresponding namespace

```
ip link set veth-red netns red
ip link set veth-blue netns blue
```

we will assign an ip address within each namespace

```
ip -n red addr add 192.168.15.1 dev veth-red
ip -n blue addr add 192.168.15.2 dev veth-blue
```

Then we bring up the interfaces for each namespace

```
ip -n red link set veth-red up
ip -n blue link set veth-blue up
```

now the interface can reach each other

```
ip netns exec red ping 192.168.15.2
```

if you lookup the arp table, you can see it has identified its neighbor at 192.168.15.2

```
ip netns exec red arp
```

same goes for the blue namespace

if you compare it with the host arp table, you can see the host arp has no idea about the new namespaces we created

now this works when we have 2 namespaces, what do you do when you have more of them?, How do you enable all of them to communicate with each other?

- Just like in the physical world, you create a virtual network inside your host. to create a network you need a switch, so you create a virtual switch within our host, and connect the namespaces to it, but how do you create a virtual switch within a host and connect the namespaces to it

There a multiple solutions such as linux bridge and open V switch, etc.

In this eample we will use the Linux bridge option using:

```bash
ip link add v-net-o type bridge
```
As far as our host is concerned, it is just another interface, just like the eth0 interface. It appears in the output of the ip link command

this interface is an interface for the host and a switch for the namespaces

the next step is to connect the namespaces to he new virtual network switch

we need to connect all the namespaces to this switch, so we need to git rid of the previous pipe we created

```
ip -n red link del veth-red
```

the other link is deleted automatically since they are a pair, let create a pair with veth-red on one and veth-red-br on the other end, this will help us identify interfaces associated with the red namespace

```
ip link add veth-red type veth peer name veth-red-br
ip link add veth-blue type veth peer name veth-blue-br
```

To attach one end of this, of the interface to the red namespace

```
ip link set veth-red netns red
ip link set veth-red-br master v-net-0
```

same for the blue

```
ip link set veth-blue netns blue
ip link set veth-blue-br master v-net-0
```

lets now setup ip addresses for these links and turn them up

```
ip -n red addr add 192.168.15.1 dev veth-red
ip -n blue addr add 192.168.15.2 dev veth-blue
```

```
ip -n red link set veth-red up
ip -n blue link set veth-blue up
```

We now have all four namespaces connected to our internal bridge network and they can all communicate with each other. They have all ip addresses. 192.168.15.1, 2, 3, and 4. And remember, we assigned our host the IP 192.168.1.2 from my host. What if I tried to reach one of these interfaces in these namespaces? Will it work? No.

My host is on one network and the namespaces are on another. But what if I really want to establish connectivity between my host and these namespaces?

Remember we said the the virtual switch is a network interface for the host, so we do have an interfave on the the 192.168.15 network on our host

- Since this just another interface all we need to do is assign an ip address to it so we can reach the namespaces through it.

```
ip addr add 192.168.15.5/24 dev v-net-0
```

We can now ping the red namespace from our local host. Now remember, this entire network is still private and restricted within the host. From within the namespaces, you can't reach the outside world nor can anyone from the outside world reach the services or applications hosted inside. The only door to the outside world is the ethernet port on the host.

So how can we configure this bridge to reach the line network through the ethernet port?

say there is another host attach to the lan network with the address 198.168.1.3, how to reach this host from my namespace?

The blue namespace sees that I'm trying to reach a network at 192.168.1, which is different from my current network of 192.168.15. So it looks at its routing table to see how to find that network. The routing table has no information about other network.

```
ip netns exec blue ping 192.168.1.3
ip netns exec blue route
```

- so it comes back as unreachable, so we need to add an entry into the routing table to provide a gateway or door to the outside world, so how do we find that gateway

that gateway is the network 192.168.15 network which is local to the blue namespace and is also connected to the outside lan network

- so our local host is the gateway that connects the two networks together we can now add a row entry to route all traffic to the 192.168.1 network through the 192.168.15.5 gateway

-  now remember our host has 2 IPs one on the bridge network at 192.168.15.5 and another on the external 192.168.1.2

Can you use any in the route? No, because the blue namespace can only reach the gateway in its local network at 192.168.15., The default gateway should be reachable from your namespace when you add it to your route.

```
ip netns exec blue ip route add 192.168.1.0/24 via 192.168.15.5
```
now try pinging it now

```
ip netns exec blue ping 192.168.1.3
```

What might be the problem? We talked about a similar situation in one of our earlier lectures where, from our home network, we tried to reach the external internet through our router. Our home network has our internal private ip addresses that the destination network don't know about so they cannot reach back. For this, we need NAT enable on our host acting as a gateway here so that it can send the messages to the LAN in its own name with its own address. So how do we add NAT functionality to our host?

- You should do that using IP tables. Add a new rule in the NAT IP table in the post routing chain to masquerade or replace the from address on all packets coming from the source network. 192.168.15.0 with its own ip address. That way, anyone receiving these packets outside the network will think that they're coming from the host and not from within the namespaces.

```
iptables -t nat -A POSTROUTING -s 192.168.15.0/24 -j MASQUERADE
```

- That way, anyone receiving these packets outside the network will think that they're coming from the host and not from within the namespaces. When we try to ping now, we see that we are able to reach the outside world.

Finally, say the lan is connected to the internet, we want the namespaces to access the internet

```
ping 8.8.8.8
```

The network is unreachable from the blue namespace

```
ip netns exec blue route
```

We see that we have route to the 192.168.1 network but no other network, we can simply say to reach any other network we need to talk to the host, we add a default gateway specifying the host

```
ip netns exec blue ip route default via 192.168.15.5
```

So what about connectivity from the outside world to inside the namespaces?

- In order to make that communication possible you have two options. The two options that we saw in the previous lecture on that. The first is to give away the identity of the private network to the second host. So we basically add an IP route entry to the second host telling the host that the network 192.168.15 can be reached through the host at 192.168.1.2. But we don't want to do that. The other option is to add a port forwarding role

```
iptables -t nat -A PREROUTING --dport 80 --to-destination 192.168.15.2:80 -j DNAT
```

## Prerequisite - Networking in Docker

When you run a container, you have different networking options to choose from. First, let's see the none network.

```
docker run --network none nginx
```

Next is the host network. With the host network, the container is attached to the host network. There is no network isolation between the host and the container.

```
docker run --network host nginx
```

The third networking option is the bridge. In this case, an internal private network is created which the Docker host and containers attach to. The network has an address 172.17.0.0 by default, and each device connecting to this network get their own internal private network address on this network. This is the network that we are most interested in, so we will take a deeper look at how exactly

```
docker run nginx
```

When Docker is installed on the host, it creates an internal private network called Bridge by default. You can see this when you're the docker network ls command.

```
docker network ls
```

Now, Docker calls the network by the name Bridge, but on the host, the network is created by the name Docker0.

```
ip link
```

So how does Docker attach the container to the bridge? As we did before, it creates a cable, a virtual cable (pipe) , with two interfaces on each end.

- Docker host
```
ip link
```

- Docker Container

```
ip -n b3165c10a92b link
```

```
ip -n b3165c10a92b addr
```

The interface pairs can be identified using their numbers.

Odd and even form a pair.

## CNI (Container Networking Interface)

We saw how to :
- Create isolated network namespaces
- Create bridge network/interface
- Create vETH Pairs (Pipes, Virtual Cables)
- Attach vEth to Namespaces
- Attach other vETH to bridge
- Assign IP Addresses
- Bring the interfaces up
- Enable NAT - IP Masquerade

We saw how to do it using namespaces and using docker

There are other ways to do the same using other container solutions such as rkt, mesos containerizer or Kubernetes

So we take all of these ideas from the different solutions and move all the networking portions of it into a single program or code.

and since this is for the bridge network we call it bridge

For example, you could run this program using its name, Bridge and specify that you want to add this container to a particular network name space.

```
bridge add 2e34dcf34 /var/run/netns/2e34dcf34
bridge add <cid> <namespace>
```

The Bridge program takes care of the rest so that the container runtime environments are relieved of those tasks.

So what if you wanted to create such a program for yourself, maybe for a new networking type?

That's where container network interface comes in.

The CNI is a set of standards that define how programs should be developed to solve networking challenges in a container runtime environments.

The programs are referred to as plugins. In this case, Bridge program that we have been referring to is a plugin for CNI.

CNI defines how the plugin should be developed:

- Container Runtime must create network namespace
- identify network the container must attach to
- Container Runtime to invoke Network Plugin (bridge) when container is ADDed
- Container Runtime to invoke Network Plugin (bridge) when container is DELeted
- JSON format of the Network Configuration

On the plugin side, it defines that the plugin:

- Must support command line arguments ADD/DEL/CHECK
- Must support parameters container id, network ns etc..
- Must manage IP Address assignment to PODs
- Must Return results in a specific format

CNI comes with a set of supported plugins already such as Bridge, VLAN, IP VLAN, MAC VLAN, one for Windows

as well as IPAM plugins like Host Local and DHCP.

There are other plugins available from third party organizations as well. Some examples are Weave, Flannel, Cilium, VMware NSX, Calico, Infoblox, et cetera.

Docker does not implement CNI. Docker has its own set of standards known as CNM

which stands for container network model

But that doesn't mean you can't use Docker with CNI at all.

You just have to work around it yourself.

```
docker run --network=none nginx
bridge add 2e34dcf34 /var/run/netns/2e34dcf34
```

That is pretty much how Kubernetes does it.

When Kubernetes creates Docker containers, it creates them on the non-network.

It then invokes the configured CNI plugins who take care of the rest of the configuration.

## Cluster Networking

The K8s cluster consists of master and worker nodes, each node must have an interface connected to a network and an ip address configured.

The hosts must have a unique host name set as well as a unique MAC address

You should note this, especially if you created the VMs by cloning from existing ones.

There are some ports that needs to be opened as well.

These are used by the various components in the control plane. The master should accept connections on 6443 for the API server.

The worker nodes, kube control tool, external users, and all other control plane components access the kube API server via this port.

The kubelets on the master and worker nodes listen on port 10250.

The kube scheduler requires port 10259 to be open.

The kube controller manager requires port 10257 to be open.

The worker nodes expose services for external access on ports 30000 to 32767,

Finally, the ETCD server listens on port 2379.

If you have multiple master nodes, all of these ports need to be open on those as well, and you also need an additional port, 2380, open, so the ETCD clients can communicate with each other.

The list of ports to be opened are also available in the Kubernetes' documentation page. So consider these when you set up networking for your nodes in your firewalls or IP table rules or network security group in a cloud environment, such as GCP or Azure or AWS, and if things are not working, this is one place to look for while you're investigating.