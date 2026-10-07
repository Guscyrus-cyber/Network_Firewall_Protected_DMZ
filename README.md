Network Lab — Firewall-Protected DMZ

Introduction

This lab builds a small DMZ (Demilitarized Zone) protected by firewall rules. The goal is to understand how organizations separate public-facing systems from internal systems and control which connections are permitted between the Internet/external network, DMZ, and internal LAN.

Main skills covered

- DMZ network architecture

- Network segmentation

- Firewall allow/deny rules

- Inbound vs. outbound traffic

- Internal LAN protection

- Testing permitted and blocked connections

- TCP ports and services

- Packet verification with Wireshark

- Basic SOC interpretation of firewall behavior

\
\
\
(Image 1)\
\
\
\
\
\
\
\
\
\
\
\
For the hands-on environment, I can use my MacBook Pro + Docker to create isolated virtual networks and containers representing:

| Component       | Role                                    |
|-----------------|-----------------------------------------|
| External Client | Simulates an Internet user              |
| Firewall        | Controls communication between networks |
| DMZ Web Server  | Public-facing server                    |
| Internal Host   | Protected corporate workstation/server  |
| Wireshark       | Observes network traffic                |

The central security principle will be:

External → DMZ Web Server ALLOW

External → Internal LAN BLOCK

DMZ → Internal LAN RESTRICT/BLOCK

Internal LAN → DMZ ALLOW when needed

That is the basic architecture used to reduce the risk that compromising a public server immediately exposes the internal network.

Step 1 — Verify Docker\
\
I open Terminal and I run: docker –version\
Then I run: docker ps\
The first command confirms Docker is installed. The second confirms that the Docker engine is running. (Image 2)\
\
\
\
\
\
\
\
\
Step 2 — Create the External Network for the Firewall-Protected DMZ lab.

I pen Docker Desktop on my Mac:

I run this command on the terminal: docker ps (Image 2)

Now I will create the first network representing the external/Internet side of the firewall.

In the terminal, I run:

docker network create \\

--driver bridge \\

--subnet 172.20.0.0/24 \\

dmz_external

### Then verify it with command: docker network inspect dmz_external (Images 3 and 4)\
\
\
\
\
\
\
\
Step 3 — Create the DMZ Network

### Now I create a Step 3 — Create the DMZ Network

Now we'll create a separate subnet for the DMZ:

docker network create \\

--driver bridge \\

--subnet 172.21.0.0/24 \\

dmz_zone

Then verify it with: docker network inspect dmz_zone

\
\
\
\
(Image 5)\
\
\
I successfully created:

DMZ network: dmz_zone\
Subnet: 172.21.0.0/24\
Gateway: 172.21.0.1\
Driver: bridge\
\
(Image 6)\
\

## Step 4 — Create the Internal LAN

Now I create the third isolated network:

docker network create \\

--driver bridge \\

--subnet 172.22.0.0/24 \\

dmz_internal

Then I verify: docker network inspect dmz_internal

After this, I will have all three security zones ready: External → DMZ → Internal.

\
\
\
(Image 7)

The new Internal LAN is correctly configured as dmz_internal, subnet 172.22.0.0/24, gateway 172.22.0.1. (Image 8)\
\
\
\
Step 5 — Create the External Client
------------------------------------------------------------------------------------------------------------------

Now I'll create a container representing a computer outside the protected organization.

I run:

docker run -dit \\

--name external-client \\

--network dmz_external \\

alpine sh

Then I verify with: docker ps --filter name=external-client\
\
The outputs confirm the container external-client is running on the dmz_external network. (Image 9)\
\
\
\
Step 6 — Create the DMZ Web Server
----------------------------------------------------------------------------------------------------

Now I’ll deploy a simple web server inside the DMZ.

I run:

docker run -dit \\

--name dmz-web \\

--network dmz_zone \\

nginx

Then I verify with: docker ps --filter name=dmz-web

At this point, the topology will be: (Image 10)

\
\
\
\
\
\
\
\
\
\
\
The dmz-web container is Up with TCP port 80 available. This is now the simulated public-facing web server inside the DMZ. (Image 11)\
\
\
\
Step 7 — Create the Internal Host
--------------------------------------------------------------------------------------------------------------------------------------

Now I'll create a host representing a protected computer on the organization's internal LAN.

I run:

docker run -dit \\

--name internal-host \\

--network dmz_internal \\

alpine sh

Then I verify with: docker ps --filter name=internal-host

The output confirms internal-host is running successfully on the internal network.\
I now have the three endpoints ready: (Imagea 12 and 13)

\
\
\
\
\
\
\
\
\
\
\
\
\
\
\
\
\
\
\
Step 8 — Create the Firewall Container
--------------------------------------

Now I am reaching the main security component of the lab.

First, I create a Linux container on the external network with the networking privileges needed for firewall/routing operations:

docker run -dit \\

--name dmz-firewall \\

--network dmz_external \\

--cap-add NET_ADMIN \\

--cap-add NET_RAW \\

alpine sh

Then I verify by running: docker ps --filter name=dmz-firewall

Notice: I’m creating a Linux Alpine container inside Docker on my MacBook Pro. That container will act as the firewall.

So, the setup is: (Images 14 )

\
\
\
\
\
\
\
\
\
The command docker run ... alpine sh automatically starts that small Linux environment inside Docker.

The output confirms dmz-firewall is running with the required networking capabilities.

Right now, the firewall is connected only to the External network. Next it given it access to the other two security zones. (Image 15)\
\
\
\
Step 9 — Connecting the Firewall to the DMZ and Internal LAN
---------------------------------------------------------------------------------------------------------------------------------------

I run these two commands:

docker network connect dmz_zone dmz-firewall

docker network connect dmz_internal dmz-firewall

Then I verify all three connections:

docker inspect dmz-firewall --format '{{json .NetworkSettings.Networks}}'\
\
( In this command, JSON format, display the firewall container's network information)

This is important: The firewall now becomes the only lab system attached to all three security zones, allowing me to configure it later to decide which traffic may cross between them.\
\
The output confirms the firewall now has three network interfaces:

dmz_external → 172.20.0.3\
dmz_zone → 172.21.0.3\
dmz_internal → 172.22.0.3

So the firewall is now physically/logically positioned between all three lab networks. (Images 16 and 17)

\
\
Step 10 — Verifying the Firewall Interfaces
-------------------------------------------

Now let's look inside the firewall container and confirm Linux sees those interfaces.

I run: docker exec dmz-firewall ip addr

The output gives exactly what I needed.

The firewall has three active Ethernet interfaces:

| Firewall interface | IP address    | Security zone |
|--------------------|---------------|---------------|
| eth0               | 172.20.0.3/24 | External      |
| eth1               | 172.21.0.3/24 | DMZ           |
| eth2               | 172.22.0.3/24 | Internal LAN  |

The many other interfaces such as tunl0, gre0, and ip6tnl0 can be ignored for this lab. They are not our three Docker network connections.

The firewall now effectively looks like this: (Images 18 and 19)

\
\
\
\
\
\
\
\
\
\
\
\
\
\
\
\
\
\
\
This is important because later a firewall rule can essentially say, for example:

eth0 → eth1 : Allow HTTP

eth0 → eth2 : Block

eth1 → eth2 : Block\
\
\
\
\
Step 11 — Checking IP Forwarding
--------------------------------

I am going to see whether Linux currently allows packets to be routed between these interfaces.

I run: docker exec dmz-firewall sysctl net.ipv4.ip_forward

The 1 means IPv4 forwarding is already enabled inside the firewall container. Therefore, Linux is capable of forwarding packets between eth0, eth1, and eth2. I don't need to enable it manually.

One important distinction: IP forwarding does not mean traffic is automatically permitted by our firewall policy. It only gives the Linux system the ability to route packets. I still need firewall rules to decide what is allowed and blocked. (Image 20)\
\
\
Step 12 — Checking the Firewall Tool
-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

Before creating any rules, I am going to see whether iptables is available inside the Alpine firewall container.

I run only: docker exec dmz-firewall iptables --version

The output says: exec: "iptables": executable file not found in \$PATH\
\
Notice: “iptables” is the tool that lets the Linux container actually behave like a firewall.

So far, the dmz-firewall container has three network interfaces and can route packets: (Images 21)

\
\
\
\
\
\
But routing alone doesn't decide which traffic is permitted or denied. iptables lets me to create those security rules.

For example, later I can tell the firewall:

External → DMZ port 80 ALLOW

External → Internal DROP

DMZ → Internal DROP

In practical terms, iptables gives me commands that mean things like:

iptables ... -j ACCEPT

ACCEPT = permit the packet.

iptables ... -j DROP

DROP = silently discard the packet.

So there are two different functions in this lab:

IP forwarding = Can this Linux system route traffic between networks?

iptables = Should this particular traffic be allowed to cross between those networks?

That's why installing iptables is central to the Firewall-Protected DMZ lab.

So, Nothing is wrong with Docker or the firewall. It simply means the minimal Alpine Linux image does not have “iptables” installed. (Image 22)\
\
\
\
Step 13 — Installing iptables in the Firewall

I run: docker exec dmz-firewall apk add --no-cache iptables

When installation finishes, I need to verify: docker exec dmz-firewall iptables --version

iptables installed successfully, and the version is: iptables v1.8.13 (nf_tables)

The (nf_tables) means this iptables command is using the modern Linux nftables backend.\
(Image 23)\
\
\
\
Step 14 — View the Firewall Before Adding Rules
----------------------------------------------------------------------------------------

Before changing anything, I am going to inspect the current forwarding policy.

I run: docker exec dmz-firewall iptables -L FORWARD -n -v

This will show me the FORWARD chain, which controls packets traveling through the firewall from one network to another.

The output tells me something important: Chain FORWARD (policy ACCEPT 0 packets, 0 bytes)

policy ACCEPT means the firewall currently allows forwarded traffic by default. Also, there are no firewall rules underneath the headings yet.

So right now, conceptually:

External → DMZ ACCEPT

External → Internal ACCEPT

DMZ → Internal ACCEPT

That is not yet a protected DMZ. I am seeing the firewall's baseline state before applying security policy. (Image 24)\
\

## Step 15 — Set the Default Forwarding Policy to DROP

Now, make the first real firewall security change.

I run: docker exec dmz-firewall iptables -P FORWARD DROP

Then I verify: docker exec dmz-firewall iptables -L FORWARD -n -v

The first line to be changed from: policy ACCEPT to: policy DROP

This implements a fundamental firewall principle: deny traffic by default, then explicitly allow only the traffic that is required.

The firewall now shows: Chain FORWARD (policy DROP ...)

That means forwarded traffic is denied by default unless I explicitly create an allow rule.

This is the correct starting point for a protected DMZ. (Image 25)\
\
\
\
Step 16 — Allowing return traffic for established connections
-------------------------------------------------------------------

Before allowing new traffic, I should permit packets that belong to connections the firewall has already accepted.

I run:

docker exec dmz-firewall iptables -A FORWARD \\

-m conntrack --ctstate ESTABLISHED,RELATED \\

-j ACCEPT

Then I verify with: docker exec dmz-firewall iptables -L FORWARD -n -v\
\
The output shows:

Chain FORWARD (policy DROP)

...

ACCEPT ... ctstate RELATED,ESTABLISHED

The 0 packets, 0 bytes is also normal, I haven’t generated traffic through this rule yet. (Image 26)\
\
\
\
\
Step 17 A— Allowing External → DMZ Web Traffic
-----------------------------------------------------------------------------------------------------

Now creating the first rule that permits a new connection through the firewall.

I want an external client to reach the DMZ web server using HTTP (TCP port 80), but nothing else.

First, I need the exact IP address Docker assigned to dmz-web. I run only:

docker inspect dmz-web \\

--format '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}'

The DMZ web server's exact IP address is:

172.21.0.2

So knowing:

\
\
\
\
\
\
\
\
\
\
\
\
\
\
\
\
(Images 27 and 28)\
\
\
\
Step 17B — Add the HTTP Allow Rule
----------------------------------

Now tell the firewall: Allow traffic arriving from the External interface (eth0), going out through the DMZ interface (eth1), destined for the DMZ web server 172.21.0.2, but only TCP port 80 (HTTP).

I run:

docker exec dmz-firewall iptables -A FORWARD \\

-i eth0 -o eth1 \\

-p tcp \\

-d 172.21.0.2 \\

--dport 80 \\

-j ACCEPT

Then I verify: docker exec dmz-firewall iptables -L FORWARD -n -v

The firewall has exactly the two rules we wanted:

DEFAULT: DROP

Rule 1: ACCEPT RELATED,ESTABLISHED

Rule 2: ACCEPT eth0 → eth1 → TCP 80 → 172.21.0.2

I notice both counters still show 0 packets / 0 bytes. That's expected because I haven't sent test traffic through the firewall yet. (Image 29)\
\
\
\
\
Step 18 — Prepare the External Client for the HTTP Test
------------------------------------------------------------------------------------------------------------------------------------------------

Before testing, I need curl inside the external-client container.

I run: docker exec external-client apk add --no-cache curl

For now, only I install curl.\
The output confirms curl. installed successfully in external-client. (Image 30)

\
\
Step 19 A — Add the Route Through the Firewall
----------------------------------------------

First, I check the external client's current routing table: docker exec external-client ip route

The external client currently has:

External client IP: 172.20.0.2

Default gateway: 172.20.0.1

Interface: eth0

Notice: The default gateway is Docker's bridge gateway, not the firewall at 172.20.0.3

There is one setup detail I need to correct before adding the route: when I created external-client, I did not give it the NET_ADMIN capability. Without that capability, Linux will not allow me to modify its routing table. (Image 31)\
\
\
\
Step 19B — Recreate the External Client with Routing Permission
------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

First, I remove only the external-client container:b docker rm -f external-client

Then I recreate it with NET_ADMIN:

docker run -dit \\

--name external-client \\

--network dmz_external \\

--cap-add NET_ADMIN \\

alpine sh

I verify: docker ps --filter name=external-client

This does not affect the firewall, DMZ server, internal host, or any containers.

I will need to reinstall curl afterward because this is a new Alpine container, but that takes only one command.

The external-client container has been recreated successfully with NET_ADMIN, so now it has permission to modify its own routing table. (Image 32)\
\
\
\
Step 19C — Reinstall curl
---------------------------------------------------------------------------------------------------------------------------------------------------

Because this is a new Alpine container, I reinstall curl:

docker exec external-client apk add --no-cache curl

Then I add the route that forces DMZ traffic through the firewall:

docker exec external-client ip route add 172.21.0.0/24 via 172.20.0.3

Finally, I verify:

docker exec external-client ip route

The routing table now contains exactly: 172.21.0.0/24 via 172.20.0.3 dev eth0

That means traffic from the external client destined for the DMZ will be sent to the firewall (172.20.0.3), rather than Docker's default gateway. (Image 33)\
\
\
\
Step 20 — Test External → DMZ HTTP Through the Firewall
-------------------------------------------------------------------------------------------------------------------------------------------------------------

The first real firewall test.

I run: docker exec external-client curl --connect-timeout 5 <http://172.21.0.2>\
\
The key evidence is: 100 896 100 896

and especially: \<h1\>Welcome to nginx!\</h1\>

This proves the external client successfully reached the DMZ web server on HTTP port 80.

The traffic path was: (Image 34 and 35)

### \
\
Step 21 — Prove the firewall processed the traffic

I run: docker exec dmz-firewall iptables -L FORWARD -n -v

Earlier both counters were:0 packets 0 bytes

After the successful HTTP request, I expect the counters to be greater than zero. That gives me firewall-level evidence that packets matched the rules.

This output is strong evidence that the firewall rule actually processed the HTTP traffic.

The HTTP rule now shows:

7 packets 446 bytes ACCEPT

tcp eth0 → eth1

destination 172.21.0.2

tcp dpt:80

Earlier it showed 0 packets / 0 bytes. After the curl request, it increased to 7 packets / 446 bytes. So, the external client's HTTP packets definitely matched the firewall's External → DMZ TCP/80 ACCEPT rule. So, the ESTABLISHED,RELATED counter is still 0. (Image 36)\
\
\
\
Step 22 — Test that non-HTTP traffic is blocked
-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

Now I will test the opposite behavior. The firewall should permit HTTP, but its default DROP policy should block other forwarded traffic.

From the external client, I try to ping the DMZ server:

docker exec external-client ping -c 3 -W 1 172.21.0.2

Because I created no ICMP allow rule, I expect the ping to fail even though HTTP succeeded.

The output shows:

3 packets transmitted

0 packets received

100% packet loss\
\
The external client attempted ICMP ping to the DMZ server 172.21.0.2, but our firewall has no rule permitting ICMP.

So, it demonstrated:

External → DMZ → TCP/80 HTTP ACCEPT

External → DMZ → ICMP DROP

The DMZ server is reachable, but only for traffic explicitly permitted by the firewall. (Image 37)\
\
\
\
Step 23 — Verify the dropped traffic
---------------------------------------------------------------------------------------------------

Now I want to see whether the firewall's default DROP policy counted those three ping packets.

I run: docker exec dmz-firewall iptables -L FORWARD -n -v

The firewall now reports: Chain FORWARD (policy DROP 3 packets, 252 bytes)

Those 3 dropped packets correspond exactly to the 3 ping packets I sent in Step 22. This gives the firewall-level proof that ICMP was blocked.

At the same time, The HTTP rule still shows:

7 packets 446 bytes ACCEPT tcp eth0 → eth1

destination 172.21.0.2 tcp dpt:80

So I have proven both sides of the firewall policy:

HTTP TCP/80 → 7 packets ACCEPTED

ICMP ping → 3 packets DROPPED (Image 38)

\
\
Step 24 A— Prepare External → Internal LAN Test
-----------------------------------------------

Now I’ll prove that an external system cannot reach the protected Internal LAN.

First I need the IP address of internal-host. I run:

docker inspect internal-host \\

--format '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}'

The output shows protected internal host is:

Internal Host: 172.22.0.2

Internal subnet: 172.22.0.0/24

Firewall internal interface: 172.22.0.3. (Image 39)\
\
\
\
\
Step 24B — Route Internal-LAN Traffic Through the Firewall
----------------------------------------------------------

The external client needs a route telling it that traffic for 172.22.0.0/24 must go through the firewall at 172.20.0.3.

I run: docker exec external-client ip route add 172.22.0.0/24 via 172.20.0.3

Then I verify: docker exec external-client ip route

The routing table is exactly right:

172.21.0.0/24 via 172.20.0.3 dev eth0 ← DMZ

172.22.0.0/24 via 172.20.0.3 dev eth0 ← Internal LAN. (Image 40)\
\
\
\
Step 24C — Test External → Internal
-----------------------------------------------------------------

Now simulating an external system attempting to reach the protected internal host:

I run: docker exec external-client ping -c 3 -W 1 172.22.0.2\
\
The result confirms:

3 packets transmitted

0 packets received

100% packet loss

So External → Internal LAN is successfully blocked.

That means I have now proven:

External → DMZ HTTP/80 ALLOW

External → DMZ ICMP DROP

External → Internal ICMP DROP (Image 41)\
\
\
\
Step 25 — Verify the new blocked packets in the firewall
--------------------------------------------------------

I run: docker exec dmz-firewall iptables -L FORWARD -n -v

Earlier the default DROP counter was: 3 packets, 252 bytes

But now it shows: DROP 6 packets, 504 bytes

So, the three External → Internal ping packets were processed by the firewall and dropped, exactly as intended.

Meanwhile, HTTP remains: 7 packets / 446 bytes → ACCEPT → TCP port 80

So, the evidence now shows:

External → DMZ HTTP/80 ACCEPT

External → DMZ ICMP DROP (3 packets)

External → Internal ICMP DROP (3 packets)

─────────

Total DROP 6 packets (Image 42)\
\
\
\
Step 26A — Create a DMZ Test Client
-----------------------------------

I run:

docker run -dit \\

--name dmz-client \\

--network dmz_zone \\

--cap-add NET_ADMIN \\

alpine sh

Then I verify: docker ps --filter name=dmz-client\
\
The dmz-client is running successfully inside dmz_zone. (Image 43)\
\
\
\
Step 26B — Check the DMZ Client's Address and Route
-------------------------------------------------------------------

I run: docker exec dmz-client ip addr show eth0

Then I run: docker exec dmz-client ip route

The DMZ test client has:

IP address: 172.21.0.4

Default gateway: 172.21.0.1

Interface: eth0

Now I need traffic for the Internal LAN to go through the firewall at 172.21.0.3.\
(Image 44)\
\
\
Step 26C — Route Internal Traffic Through the Firewall
----------------------------------------------------------------------------------

I run: docker exec dmz-client ip route add 172.22.0.0/24 via 172.21.0.3

Then I verify: docker exec dmz-client ip route\
\
The routing table now correctly shows: 172.22.0.0/24 via 172.21.0.3 dev eth0

That means traffic from the DMZ client (172.21.0.4) toward the Internal LAN must pass through our firewall (172.21.0.3). (Image 45)\
\
\
\
Step 26D — Final DMZ → Internal Test
------------------------------------------------------------------------------------------------------------------------------------

Now I run: docker exec dmz-client ping -c 3 -W 1 172.22.0.2

The result:

3 packets transmitted

0 packets received

100% packet loss

It confirms that DMZ → Internal LAN traffic was blocked by the firewall.

I have now demonstrated the complete security behavior:

| Test                       | Result |
|----------------------------|--------|
| External → DMZ HTTP TCP/80 | ACCEPT |
| External → DMZ ICMP        | DROP   |
| External → Internal LAN    | DROP   |
| DMZ → Internal LAN         | DROP   |

(Image 46)\
\
Step 27 — Final Firewall Verification
-------------------------------------

Let's collect the final firewall evidence. I run:

docker exec dmz-firewall iptables -L FORWARD -n -v

Earlier the default DROP counter was 6 packets. Since I just generated another three blocked packets from DMZ → Internal, I expect it to increase to approximately 9 packets.\
\
The final firewall evidence is:

Chain FORWARD (policy DROP 9 packets, 756 bytes)

7 packets / 446 bytes → ACCEPT → TCP port 80 → 172.21.0.2

9 packets / 756 bytes → DROP by default

### The 9 dropped packets correspond to the three sets of three blocked ICMP tests: External → DMZ, External → Internal, and DMZ → Internal. (Image 47)\
\
\
\
Final SOC interpretation

The lab successfully implemented a segmented architecture consisting of External, DMZ, and Internal networks protected by a Linux firewall. A default-deny (DROP) forwarding policy was applied, while an explicit rule permitted external HTTP traffic to the DMZ web server on TCP port 80. Testing verified that authorized HTTP traffic reached the DMZ server, while unauthorized ICMP traffic to the DMZ and Internal network was blocked. DMZ-to-Internal communication was also successfully prevented, demonstrating network segmentation, least-privilege firewall policy, and protection of internal assets if a DMZ system were compromised.

Final result:

External → DMZ TCP/80 ACCEPT

External → DMZ ICMP DROP

External → Internal ICMP DROP

DMZ → Internal ICMP DROP

Firewall ACCEPT evidence: 7 packets / 446 bytes

Firewall DROP evidence: 9 packets / 756 bytes

## Key Networking Terminology

| Term | Definition |
|----|----|
| DMZ (Demilitarized Zone) | A separate network segment placed between an external/untrusted network and an organization's internal network. Public-facing services, such as web servers, can be placed in the DMZ so they can be accessed without directly exposing the internal network. |
| Firewall | A security system that controls network traffic according to predefined rules. It can allow (ACCEPT)or block (DROP) traffic based on factors such as source/destination IP address, protocol, port, and network interface. |
| ICMP (Internet Control Message Protocol) | A network-layer protocol primarily used for network diagnostics and error reporting. The pingcommand uses ICMP Echo Request and Echo Reply messages to test whether another host is reachable. |
| TCP (Transmission Control Protocol) | A connection-oriented transport protocol that provides reliable and ordered delivery of data. Services such as HTTP, HTTPS, and SSH commonly use TCP. |
| TCP Port | A numbered logical endpoint used to identify a particular TCP service. For example, HTTP commonly uses TCP port 80, HTTPS uses 443, and SSH uses 22. |
| External Network | An untrusted network located outside the protected organizational environment. In this lab, the external client simulated a system attempting to access resources from outside the protected networks. |
| Internal Network (LAN) | A trusted or more highly protected network containing internal organizational systems. It should normally not be directly accessible from an external network. |
| Network Segmentation | The practice of dividing a network into separate security zones or subnets. In this lab, External, DMZ, and Internal systems were placed on different Docker networks. |
| Default-Deny Policy | A firewall security approach in which traffic is blocked by default unless a rule explicitly permits it. The lab implemented this using the FORWARD chain's default DROP policy. |
| ACCEPT | A firewall action that permits matching network packets to continue through the firewall. |
| DROP | A firewall action that silently discards matching packets instead of allowing them through. |
| External → DMZ | Traffic originating from the external network and attempting to reach a system located in the DMZ. In this lab, HTTP/TCP port 80 was explicitly permitted to the DMZ web server. |
| External → DMZ ICMP | ICMP traffic originating from the external network and targeting the DMZ. The lab's ping test was blocked by the default DROP policy, demonstrating that DMZ access was limited to specifically authorized traffic. |
| External → Internal | Traffic originating from an external/untrusted network and attempting to directly reach the protected internal network. This traffic was blocked in the lab. |
| DMZ → Internal ICMP | ICMP traffic originating from a system inside the DMZ and attempting to reach the internal network. This was blocked, demonstrating that a DMZ system does not automatically have access to internal assets. |
| IP Forwarding | A Linux networking capability that allows a system with multiple network interfaces to route packets between different networks. The firewall container had IPv4 forwarding enabled so it could operate between the External, DMZ, and Internal networks. |
| iptables | A Linux firewall administration utility used to create rules that determine which packets are accepted, dropped, or otherwise processed. It was used in this lab to implement the DMZ firewall policy. |

### \
\
\
\
\
\
\
\
\
\
\
\
\
\
\
\
\
\
\
\
\

Top of Form

Bottom of Form

Top of Form

Bottom of Form
