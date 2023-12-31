# linux-ns-networking-2

## Connecting a container to host using virtual Ethernet cable

In root ns, it looks like

![root-ns](https://lab-bucket.s3.brilliant.com.bd/labthumbnail/d5e142d5-3839-4c15-983b-140d30c75616.png)

Let's create a custom namespace using `ip netns add` utiliy.

```bash
sudo ip netns add red
sudo ip netns list
```

![red-ns](https://lab-bucket.s3.brilliant.com.bd/labthumbnail/a9d6a2b2-baed-473b-9944-a6ea4f88a95d.png)

From the root network namespace, let's create a veth cable:

```bash
sudo ip link add veth-red type veth peer name veth-host
```
We just created a pair of interconnected virtual Ethernet devices or interfaces, whatever we say. Both `veth-red` and `veth-host` lies inside the root ns.

```bash
ip link list
```

![link-list](https://lab-bucket.s3.brilliant.com.bd/labthumbnail/a7fd0d34-7648-4c12-abd1-9fa6020f6622.png)

To connect the root namespace with the `red` namespace, we need to keep one end of the cable in the root namespace and move another one end into the `red` ns.

```bash
sudo ip link set veth-red netns red
```
This moves one end of the veth pair (veth-red) into the "red" namespace. The other end (veth-host) remains in the default network namespace.

Now, let's configure IP Addresses to both end of this veth cable and once we turn up the interfaces, the peer device will instantly display any packet that appears on one of the devices.

In the "red" namespace:

```bash
sudo ip netns exec red ip addr add 192.168.1.1/24 dev veth-red
sudo ip netns exec red ip link set veth-red up
```

![red-interface](https://lab-bucket.s3.brilliant.com.bd/labthumbnail/7e6abdb2-3e1d-4e81-9881-1ca23346f633.png)

On the host side:

```bash
sudo ip addr add 192.168.1.2/24 dev veth-host
sudo ip link set veth-host up
```

![root-interface](https://lab-bucket.s3.brilliant.com.bd/labthumbnail/f6050972-f838-4d51-b926-4b5fd11fc38f.png)

The virtual ethernet pair is now ready.

![interfaces](https://lab-bucket.s3.brilliant.com.bd/labthumbnail/fd67f4f9-66a2-48e1-a625-c9db592180cc.png)

A route is needed to add on the host to direct traffic destined for 192.168.1.1 through the veth-host interface.

```bash
sudo ip route add 192.168.1.1 dev veth-host
```

## Why Add a Route?

Without adding the route, the system doesn't know how to reach 192.168.1.1. When we ping 192.168.1.1, the system checks its routing table to determine where to send the ping packets. If there's no specific route for 192.168.1.1, it won't know which interface to use.

By adding the route, we are explicitly telling the system that to reach 192.168.1.1, it should send the traffic through the veth-host interface, which is part of the veth pair connected to the "red" namespace.

## Test connectivity

Let's try to ping the `red` ns from the veth-host interface:

```bash
ping 192.168.1.1 -c 3
```

![host to ns ping](https://lab-bucket.s3.brilliant.com.bd/labthumbnail/00bf9321-386f-492f-a08a-28b6d42bdb9d.png)

Again, let's try to ping the veth-host interface from the `red` namespace:

```bash
sudo ip netns exec red bash
ping 192.168.1.2 -c 3
```

![ns to host ping](https://lab-bucket.s3.brilliant.com.bd/labthumbnail/15503c59-8766-4334-9d50-17f979d58ab0.png)