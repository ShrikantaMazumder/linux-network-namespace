# linux-network-namespace

<p>Before starting the journey let’s have a glimpse about what we are going to know. We are about to learn Linux namespace, virtual ethernet, virtual bridge, and also some other networking concepts. With the help of all of these we can create network stacks and connect and communicate with them and also network namespaces will be able to communicate with the internet. This blog will help to understand docker internal networking. Besides conceptional discussion, we will see hands-on examples by creating them. Let’s dive into it…</p>

**Prerequisite:**

1. You need a Linux environment.
2. Iproute2 package installed on the machine

**What are Linux namespaces?**

<p>Namespaces are a feature of Linux Kernel that partitions kernel resources. These are fundamental aspects of containers in Linux. The key feature of namespaces is that they isolate processes from each other. 
There are different types of namespaces within Linux Kernel, today we are going to work with Network namespace. So let’s know about it.
Network namespaces virtualize the network stack. It provides isolation of the system resources associated with networking: network devices, IPv4 and IPv6 protocol stacks, IP routing tables, firewall rules, and other network-related resources.
</p>

<p>Now create two network namespaces. You can create any amount of network namespaces based on your requirement.</p>

```
$ sudo ip netns add web
$ sudo ip netns add database
```

<p>Use the below command to check whether namespaces were created or not.</p>

```
$ ip netns list
web
database
```

<p>Or you can open the namespace directory by: </p>

```
$ open /var/run/netns/ (This will open the directory and you will find created namespaces).
```

<p style="text-align:center">Here is a visual example</p>

![Create NET NS image](https://github.com/ShrikantaMazumder/linux-network-namespace/assets/38990863/c4545924-2b0d-4480-99b1-abfe8381ba7b)

<p>Our namespace creation has been done. To connect them to each other we need veth devices.</p>

**What is veth device?**

<p>The veth devices are virtual Ethernet devices.  They can act as tunnels between network namespaces to create a bridge to a physical network device in another namespace, but can also be used as standalone network devices.</p>

<p>Create two veth devices for two namespaces.</p>
```
$ sudo ip link add web_veth type veth peer name web_veth_peer
$ sudo ip link add db_veth type veth peer name db_veth_peer
```

<p>Each veth device has two endpoints and they have a given name.</p>

![VETH](https://github.com/ShrikantaMazumder/linux-network-namespace/assets/38990863/b37e9da6-3e57-41c8-bf16-4a476a4ad291)

<p>So, it’s time to connect veth devices with namespaces. Here is the command to do this:</p>

```
$ sudo ip link set web_veth netns web
$ sudo ip link set db_veth netns database
```

<p>Let’s see the current visual state after veth device association.</p>

![VETH to NS](https://github.com/ShrikantaMazumder/linux-network-namespace/assets/38990863/c2d71c0b-c699-4384-ae3d-4d95eb79408e)

<p>
Maybe you get the point that each veth device has two endpoints, but now we connect one part of devices with namespaces. What will happen with the other part? See two namespaces have their own veth devices but they are not still connected. So here a new device introduced named <strong>Bridge</strong> and also this bridge is virtual. Let’s create a bridge and connect veth’s other endpoints with the bridge.
</p>

<p>Here are bridge create command and veth connection command:</p>

```
$ sudo ip link add app_br type bridge (Here app_br is the bridge device name. It could your choice)
$ sudo ip link set web_veth_peer master app_brr
$ sudo ip link set db_veth_peer master app_br
```

<p>The final output is:</p>

![NS with Bridge](https://github.com/ShrikantaMazumder/linux-network-namespace/assets/38990863/5de4b9fa-4d5a-466b-9bc5-c661c4310d02)

<p>
Now, namespaces are connected with each other via virtual bridge and veth. But the journey is not finished yet, we should follow a few steps to get the final result.
</p>

<p>
Each veth device needs an ip address on the side that is connected with namespaces. So let's assign IP address to respective veth.
</p>

```
$ sudo ip netns exec web ip addr add 10.10.0.10/16 dev web_veth
$ sudo ip netns exec database ip addr add 10.10.0.20/16 dev db_veth
```

![Bridge NS IP](https://github.com/ShrikantaMazumder/linux-network-namespace/assets/38990863/c52513f3-a1ca-4d20-b38f-5a1636b6a988)

<p>
We have to up all of the virtual devices to connect. Currently all of these devices are in down state. Follow the commands to up:
</p>

```
$ sudo ip link set app_br up
$ sudo ip link set web_br_veth up
$ sudo ip link set db_br_veth up
$ sudo ip netns exec web ip link set dev lo up
$ sudo ip netns exec web ip link set dev web_veth up
$ sudo ip netns exec database ip link set dev lo up
$ sudo ip netns exec database ip link set dev db_veth up
```

<p>
So all of the devices are up now. We should now check each namespace's reachability through the network. We can ping a namespace from another by IP address. Let’s see.
</p>

```
$ sudo ip netns exec web ping 10.10.0.20

PING 10.10.0.20 (10.10.0.20) 56(84) bytes of data.
64 bytes from 10.10.0.20: icmp_seq=1 ttl=64 time=0.171 ms
64 bytes from 10.10.0.20: icmp_seq=2 ttl=64 time=0.170 ms
```

<p>It’s working.</p>

<p>
10.10.0.20 - this ip address is assigned to the database namespace.. But we used ping from web namespace.
</p>

<p>
Wait! Did you notice one thing? We didn’t add any route tables but still ping is working. But why? Because of the bridge. Check the route inside of the namespace there already route added and also ARP is populated.
</p>

```
$ sudo ip netns exec web ip route
10.10.0.0/16 dev web_veth proto kernel scope link src 10.10.0.10

$ sudo ip netns exec web arp
Address                  HWtype  HWaddress           Flags Mask            Iface
10.10.0.20               ether   de:b7:c0:92:b3:02   C                     web_veth
```

<p>
And finally we can ping and get a reply from it.
</p>

<p>
Are you happy with this? But I am not… Because I can not ping root/host. It returns “Network is unreachable” message. Let check:
To get root/host interface ip use the following command -
</p>

```
$ ip addr show
```

<p>This command returns all interfaces. Common names for the root interface include <strong>eth0</strong> or <strong>ensX</strong>, depending on your system. To be confirmed just ignore “lo” and veth named interfaces. For me it’s <strong>enp0s1</strong> and IP address is <strong>192.168.64.4</strong></p>

<p>Let’s ping the root ip from a network interface.</p>

```
$ sudo ip netns exec web ping 192.168.64.4
ping: connect: Network is unreachable
```

<p>
This one is not surprising behavior. If you check “route” of a namespace, you will not see any route table entry there to communicate with the host or anywhere. So the ip unknown and can’t establish a connection.
</p>

<p>
We have to add a default gateway so that we can forward any unknown ip address that does not match the route table via bridge. To achieve this we should add ip to bridge and default gateway to our namespaces route tables.
</p>

```
$ sudo ip addr add 10.10.0.1/16 dev app_br
$ sudo ip netns exec web ip route add default via 10.10.0.1
$ sudo ip netns exec database ip route add default via 10.10.0.1
```

<p>Want to see your route table and default entry?</p>

```
$ sudo ip netns exec web route
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
default         10.10.0.1       0.0.0.0         UG    0      0        0 web_veth
10.10.0.0       0.0.0.0         255.255.0.0     U     0      0        0 web_veth

$ sudo ip netns exec database route
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
default         10.10.0.1       0.0.0.0         UG    0      0        0 db_veth
10.10.0.0       0.0.0.0         255.255.0.0     U     0      0        0 db_veth
```

<p>Now ping again to root/host ns ip and see what happens.</p>

```
$ sudo ip netns exec web ping 192.168.64.4
```

<p>It’s working. Congrats! Another achievement unlocked.</p>

![NS Bridge IP](https://github.com/ShrikantaMazumder/linux-network-namespace/assets/38990863/ac17f2d4-b379-4d27-afa4-d95a092c0245)

<p>Now I am going to dive into the final step of this blog.</p>

<div style="text-align: center; font-weight: bold;">Network namespace to Internet</div>

<p>
Till now we can establish connection from net namespace to root/host namespace through bridge. But when we try to ping or establish a connection out of the box (internet), it now does not show packet drop or Network is unreachable. It just stuck somewhere.
</p>

<p>Let try the command below - </p>

```
$ sudo ip netns exec web ping 8.8.8.8
```

<p>8.8.8.8 is google dns ip</p>

<p>
What’s going on under the hood? Why is this one stuck? Let’s debug it, and to do this we need a tool called <strong>tcpdump</strong>.
</p>

<p>
<strong>Step-1:</strong> The first step is, when we make a ping request from namespace it goes to host/root via bridge (app_br). So first debug app_br to check if the packet goes there or not.
</p>
Open new tab on your terminal (new ssh session)

- (In terminal1):

```
$ sudo tcpdump -i app_br icmp (This will listen app_br bridge)
```

```
tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
listening on app_br, link-type EN10MB (Ethernet), snapshot length 262144 bytes
```

- (In terminal2):

```
$ sudo ip netns exec web ping 8.8.8.8 -c1
```

<p>Go to terminal1 and you will see changes that app_br receiving packets like below example:</p>

![tcpdump-app-br](https://github.com/ShrikantaMazumder/linux-network-namespace/assets/38990863/1031d8a8-1cdb-41e9-9561-70a27c3200be)

<p>It looks bridge is ok and receiving the packets.</p>

<p>
<strong>Step-2:</strong> In the next step the packet travels to the root/host ns interface. So let’s debug the root ns interface.
</p>

(terminal1):

```
$ sudo tcpdump -i enp0s1 icmp  // enp0s1 is my interface. Use your own
```

(termina2):

```
$ sudo ip netns exec web ping 8.8.8.8 -c1
```

<p>
This time after ping we can’t see any change in the tcpdump terminal. That means the packet is not reached to the root interface we can say.
</p>
<p>
The problem is here. ip_forward is disabled on my system. So it should be enabled. Let’s enabled it.
</p>

```
$ sudo sysctl -w net.ipv4.ip_forward=1
```

<p>
Now try again like step-2. This time we can see changes on tcpdump terminal. But ping is not successful yet. It’s just stuck. We are not getting any reply from the request.
</p>

<p style="text-align:center; font-weight: bold;">Hold On</p>

<p>
Did you notice anything on tcpdump logs? Yes, you are right. 10.10.0.10 (web net ns ip) is trying to reach out google dns. Our web NS ip is private. By private IP we can not reach the internet. Why? This is another discussion. Keep it for another blog or you can google it. Just keep in mind that by private ip we can not reach the internet and the internet can not reach up. So we should take help from the public ip.
</p>

<p>
In this scenario, NAT can help us to convert private ip to public ip.
</p>

<p>
The primary purpose of NAT is to conserve public IP addresses, as the number of available IPv4 addresses is limited. It provides a way to extend the usability of a limited number of public IP addresses across multiple devices in a private network.
</p>

<p>
When a device from the private network initiates communication with a device on the internet, NAT modifies the source IP address and port number of the outgoing packets to the public IP address and a unique port number assigned by the NAT device. This modified information allows the packets to be routed back to the appropriate device within the private network when the response is received.
</p>

<p>We have to add a source NAT rule in the <strong>POSTROUTING</strong> chain. Follow the command:</p>

```
$ sudo iptables --table nat -A POSTROUTING -s 10.10.0.0/16 ! -o app_br -j MASQUERADE
```

<p>
Now follow again <strong>step-2</strong>.

The output is:

</p>

![tcpdump-ok](https://github.com/ShrikantaMazumder/linux-network-namespace/assets/38990863/1061e36c-c829-4caf-847c-6293320b922d)
![ping-ok](https://github.com/ShrikantaMazumder/linux-network-namespace/assets/38990863/1480014d-37eb-44ea-a96b-a3427ec26baf)

<p>
Aaaaand finally, we got a reply from ping. Now we can access the internet from our network namespaces.

The final visual of the system we built:
</p>

![7 final-view](https://github.com/ShrikantaMazumder/linux-network-namespace/assets/38990863/f854b937-a4c1-4d76-9027-267f61c328d2)
