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
ip addr add 192.168.1.11/24 dev etho0
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

