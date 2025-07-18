
## Summary
This page provides an overview of basic networking concepts in Podman, focusing on the differences between rootful and rootless container networking, the impact of firewalls, and the main network modes available: bridge, macvlan, and slirp4netns. It explains how each mode works, their use cases, and provides practical examples for configuring and using them. The document also covers how containers and pods communicate, highlighting the network namespace sharing in pods and the implications for connectivity and port mapping.

## Source
https://github.com/containers/podman/blob/main/docs/tutorials/basic_networking.md

## Differences between rootful and rootless container networking

One of the guiding factors on networking for containers with Podman is going to
be whether or not the container is run by a root user or not. This is because
unprivileged users cannot create networking interfaces on the host. Therefore,
for rootless containers, the default network mode is slirp4netns. Because of the
limited privileges, slirp4netns lacks some of the features of networking
compared to rootful Podman's networking; for example, slirp4netns cannot give
containers a routable IP address. The default networking mode for rootful
containers on the other side is netavark, which allows a container to have a
routable IP address.

## Firewalls

The role of a firewall will not impact the setup and configuration of networking,
but it will impact traffic on those networks.  The most obvious is inbound network
traffic to the container host, which is being passed onto containers usually with
port mapping.  Depending on the firewall implementation, we have observed firewall
ports being opened automatically due to running a container with a port mapping (for
example).  If container traffic does not seem to work properly, check the firewall
and allow traffic on ports the container is using. A common problem is that
reloading the firewall deletes the netavark iptables rules resulting in a loss of
network connectivity for rootful containers. Podman v3 provides the podman
network reload command to restore this without having to restart the container.

## Basic Network Setups

Most containers and pods being run with Podman adhere to a couple of simple scenarios.
By default, rootful Podman will create a bridged network.  This is the most straightforward
and preferred network setup for Podman. Bridge networking creates an interface for
the container on an internal bridge network, which is then connected to the internet
via Network Address Translation(NAT).  We also see users wanting to use `macvlan`
for networking as well. The `macvlan` plugin forwards an entire network interface
from the host into the container, allowing it access to the network the host is connected
to. And finally, the default network configuration for rootless containers is `slirp4netns`.
The slirp4netns network mode has limited capabilities but can be run on users without
root privileges. It creates a tunnel from the host into the container to forward
traffic.

### Bridge

A bridge network is defined as an internal network is, created where both the container and host are attached.  Then this network is capable of allowing the containers to communicate outside the host.


![bridge_network](podman_bridge.png)


Consider the above illustration.  It depicts a laptop user running two containers:
a web and db instance.  These two containers are on the virtual network with the
host.  Additionally, by default, these containers can initiate communications outside
the laptop (to the Internet for example).  The containers on the virtual network
typically have non-routable, also known as private IP addresses.

When dealing with communication that is being initiated outside the host, the outside
client typically must address the laptop’s external network interface and given port
number.  Assuming the host allows incoming traffic, the host will know to forward
the incoming traffic on that port to the specific container.  To accomplish this,
firewall rules are added to forward traffic when a container requests a specific
port be forwarded.

Bridge networking is the default for Podman containers created as root. Podman provides
a default bridge network, but you can create others using the `podman network create`
command. Containers can be joined to a network when they are created with the
`--network` flag, or after they are created via the `podman network connect` and
`podman network disconnect` commands.

As mentioned earlier, slirp4netns is the default network configuration for rootless
users.  But as of Podman version 4.0, rootless users can also use netavark.
The user experience of rootless netavark is very akin to a rootful netavark, except that
there is no default network configuration provided.  You simply need to create a
network, and the one will be created as a bridge network. If you would like to switch from
CNI networking to netavark, you must issue the `podman system reset --force` command.
This will delete all of your images, containers, and custom networks.

```
$ podman network create
```

When rootless containers are run, network operations
will be executed inside an extra network namespace. To join this namespace, use
`podman unshare --rootless-netns`.

#### Default Network

The default network `podman` with netavark is memory-only.  It does not support dns resolution because of backwards compatibility with Docker.  To change settings, export the in-memory network and change the file.

For the default rootful network use
```
podman network inspect podman | jq .[] > /etc/containers/networks/podman.json
```

And for the rootless network use

```
podman network inspect podman | jq .[] > ~/.local/share/containers/storage/networks/podman.json
```


#### Example

By default, rootful containers use the netavark for its default network if
you have not migrated from Podman v3.
In this case, no network name must be passed to Podman.  However, you can create
additional bridged networks with the podman create command.

The following example shows how to set up a web server and expose it to the network
outside the host as both rootful and rootless.  It will also show how an outside
client can connect to the container.

```
(rootful) $ sudo podman run -dt --name webserver -p 8080:80 quay.io/libpod/banner
```

Now run the container.
```
$ podman run -dt --name webserver --network podman1 -p 8081:80 quay.io/libpod/banner
```
Note in the above run command, the container’s port 80 (where the Nginx server is
running) was mapped to the host’s port 8080.  Port 8080 was chosen to demonstrate
how the host and container ports can be mapped for external access.  The port could
very well have been 80 as well (except for rootless users).

To connect from an outside client to the webserver, simply point an HTTP client to
the host’s IP address at port 8080 for rootful and port 8081 for rootless.
```
(outside_host): $ curl localhost:8080

(outside_host): $ curl 192.168:8081
```

### Macvlan
With macvlan, the container is given access to a physical network interface on the
host. This interface can configure multiple subinterfaces.  And each subinterface
is capable of having its own MAC and IP address.  In the case of Podman containers,
the container will present itself as if it is on the same network as the host.
![macvlan_network](podman_macvlan.png)
In the illustration, outside clients will be able to access the web container by
its IP address directly.  Usually the network information, including IP address,
is leased from a DHCP server like most other network clients on the network.  If
the laptop is running a firewall, such as firewalld, then accommodations will need
to be made for proper access.

Note that Podman has to be run as root in order to use macvlan.

#### Example

The following example demonstrates how to set up a web container on a macvlan and
how to access that container from outside the host.  First, create the macvlan network.
 You need to know the network interface on the host that connects to the routable
network.  In the example case, it is eth0.

```
$ sudo podman network create -d macvlan -o parent=eth0 webnetwork
webnetwork
```

The next step is to ensure that the DHCP service is running. This handles
the DHCP leases from the network. If DHCP is not needed, the `--subnet` option
can be used to assign a static subnet in the `network create` command above.

CNI and netavark both use their own DHCP service; therefore, you need to know
what backend you are using. To see what you are using, run this command:
```
$ sudo podman info --format {{.Host.NetworkBackend}}
```
If this command does not work, you are using an older version prior to Podman
v4.0 which means you are using CNI.
If the netavark backend is used, at least Podman v4.5 with netavark v1.6 is
required to use DHCP.

For netavark use:
```
$ sudo systemctl enable --now netavark-dhcp-proxy.socket
```
Or if the system doesn't use systemd, start the daemon manually:
```
$ /usr/libexec/podman/netavark dhcp-proxy --activity-timeout 0
```


### Slirp4netns

Slirp4netns is the default network setup for rootless containers and pods.  It was
invented because unprivileged users are not allowed to make network interfaces on
the host.  Slirp4netns creates a TAP device in the container’s network namespace
and connects to the usermode TCP/IP stack.  Consider the following illustration.

![slirp_network](podman_rootless_default.png)
The unprivileged user on this laptop has created two containers: a DB container and
a web container.  Both of these containers have the ability to access content on
networks outside the laptop.  And outside clients can access the containers if the
container is bound to a host port and the laptop firewall allows it.  Remember, unprivileged
users must use ports 1024 through 65535 as lower ports require root privileges. (CAP_NET_BIND_SERVICE)
Note: this can be adjusted using the `sysctl net.ipv4.ip_unprivileged_port_start`

One of the drawbacks of slirp4netns is that the containers are completely isolated
from each other.  Unlike the bridge approach, there is no virtual network.  For containers
to communicate with each other, they can use the port mappings with the host system,
or they can be put into a Pod where they share the same network namespace. See [Communicating
between containers and pods](#Communicating-between-containers-and-pods) for more information.

#### Example

The following example will show how two rootless containers can communicate with
each other where one is a web server.  Then it will show how a client on the host’s
network can communicate with the rootless web server.

First, run the rootless web server and map port 80 from the container to a non-privileged
port like 8080.
```
$ podman run -dt --name webserver -p 8080:80 quay.io/libpod/banner
17ea33ccd7f55ff45766b3ec596b990a5f2ba66eb9159cb89748a85dc3cebfe0
```
Because rootless containers cannot communicate with each other directly with TCP/IP
via IP addresses, the host and the port mapping are used.  To do so, the IP address
of the host (interface) must be known.
```
$ ip address show eth0
3: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group
default qlen 1000
link/ether 3c:e1:a1:c1:7a:3f brd ff:ff:ff:ff:ff:ff
altname eth0
inet 192.168.99.109/24 brd 192.168.99.255 scope global dynamic noprefixroute eth0
valid_lft 78808sec preferred_lft 78808sec
inet6 fe80::5632:6f10:9e76:c33/64 scope link noprefixroute
valid_lft forever preferred_lft forever
```
From another rootless container, use the host’s IP address and port to communicate
between the two rootless containers successfully.
```
$ podman run -it quay.io/libpod/banner curl http://192.168.99.109:8080

```
From a client outside the host, the IP address and port can also be used:
```
(outside_host): $ curl http://192.168.99.109:8080
```

## Communicating between containers and pods

Most users of containers have a decent understanding of how containers communicate
with each other and the rest of the world.  Usually each container has its own IP
address and networking information.  They communicate amongst each other using regular
TCP/IP means like IP addresses or, in many cases, using DNS names often based on
the container name.  But pods are a collection of one or more containers, and with
that, some uniqueness is inherited.

By definition, all containers in a Podman pod share the same network namespace. This
fact means that they will have the same IP address, MAC addresses, and port mappings.
You can conveniently communicate between containers in a pod by using localhost.

![slirp_network](podman_pod.png)

The above illustration describes a Pod on a bridged network.  As depicted, the Pod
has two containers “inside” it: a DB and a Web container.  Because they share the
same network namespace, the DB and Web container can communicate with each other
using localhost (127.0.0.1).  Furthermore, they are also both addressable by the
IP address (and DNS name if applicable) assigned to the Pod itself.

For more information on container to container networking, see [Configuring container
networking with Podman](https://www.redhat.com/sysadmin/container-networking-podman).


### Slirp4netns and TAP Devices
Slirp4netns is a user-mode networking tool that allows rootless containers to have network connectivity without requiring root privileges. It uses TAP (Terminal Access Point) devices to create a virtual network interface within the container's network namespace.

### What is a TAP Device?
A TAP device is a virtual network interface that operates at the data link layer (Layer 2) of the OSI model. It allows user-space programs to interact with network packets as if they were dealing with a physical network interface. TAP devices are commonly used in virtualized environments to provide network connectivity to virtual machines or containers.

# How Slirp4netns Uses TAP ?

When using slirp4netns (a user-mode networking tool for rootless containers), it creates a TAP device inside the container’s network namespace. This TAP device acts as the container’s network interface, and slirp4netns bridges it to the host’s network stack — all without needing root privileges.
So yes — Slirp4netns creates a TAP device to simulate a network interface inside the container, allowing it to communicate with the outside world through a user-space bridge.

## How TAP Devices Work togeether with Bridge Networking ?
When using bridge networking with Podman, TAP devices play a crucial role in connecting the container's network namespace to the host's network. Here's how it works:

```
[ VM or container ]
        │
     [ TAP ]
        │
     [ BRIDGE ]──[ eth0 (host NIC) ]
        │
     [ LAN / Internet ]
   ```
In this setup, the TAP device acts as a virtual network interface for the container, while the bridge interface connects it to the host's network interface (e.g., `eth0`). The bridge allows multiple TAP devices (from different containers) to communicate with each other and with the host's network.

## How TAP Devices Work with Bridge Networking in Podman
When using bridge networking in Podman, TAP devices are created to allow containers to communicate with the host network and other containers. Here's a step-by-step explanation of how this works:
   1. **TAP Device Creation**: When a container is started with bridge networking, Podman creates a TAP device within the container's network namespace. This TAP device acts as the container's network interface.
   2. **Bridge Creation**: Podman also creates a bridge interface on the host (e.g., `br0`). This bridge acts as a virtual switch that connects multiple TAP devices and the host's network interface.
   3. **Connecting TAP to Bridge**: The TAP device created for the container is added to the bridge interface. This allows the container to communicate with other devices on the bridge, including the host's network interface.
   4. **Network Communication**: The bridge interface forwards packets between the TAP device and the host's network interface (e.g., `eth0`). This allows the container to access the LAN or Internet, as packets sent from the container through the TAP device are routed through the bridge to the host's network.
   ## Real Life Example of TAP Device with Bridge Networking
   When using Podman with root privileges, you can set up a TAP device and bridge networking to allow your containers or virtual machines to communicate with the outside world as if they were directly connected to your local area network (LAN).  
   **Example:**
   If you're running a Podman container or a virtual machine (VM) and want it to have its own IP address on your LAN, you can create a TAP device and a bridge. Here's how you might do it:
     if you're using QEMU or Podman with root privileges, you might:
     1. Create a TAP device (tap0)
     2. Create a bridge (br0)
     3. Add tap0 and eth0 to br0
   This lets your VM/container appear as a peer on your LAN, with its own IP address.



