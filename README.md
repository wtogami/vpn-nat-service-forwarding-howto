# VPN NAT Service Forwarding HOWTO - DRAFT v0.1

This guide describes how to configure a VPN frontend server where incoming TCP IPv4 connections are forwarded over a VPN to services running on a remote backend server. All outgoing connections from these services are routed via this same VPN frontend. Lightning Network nodes are described but this approach should work for just about any TCP service.

Why would you want this?
- Guards against the public easily ascertaining the location of the backend server.
- LN payments require lower latency and higher reliabilty than Tor is consistently capable.
- Unlike ssh port forwarding or proxy-based frontends the backend service is able to see the actual source IP address of incoming connections, and nobody ever sees connections coming from the backend's address.
- Allows for DoS filtering measures to happen at an external service provider before traffic reaches your backend network.

Unique to this guide: Only explicitly configured backend services are forwarded via the VPN frontend while other services on the same machine operate normally via their default gateway. This is desirable because bulk traffic (like syncing your full node) need not traverse metered data at the frontend network provider.

**Currently this guide describes configuration of VPN NAT service forwarding with NetworkManager-1.31.2 or later. The iptables configurations below are specific to how it works on Fedora or EL8. It is my hope that others figure out alternate persistent configurations or non-NetworkManager and this guide can be expanded. This capabilty would be quite powerful if included as an option to node appliance distros like Umbrel or MyNode.**

**Contact: warren -at- blockstream dot com**

# How does this work?
Linux Netfilter NAT is normally used on common Linux-based "Wifi Router" appliances where you want to share an Internet connection with multiple clients on your local area network (LAN). NAT routers have a private IPv4 address like `10.0.0.1/24` on the LAN-facing internal interface. Clients are assigned addresses like `10.0.0.2/24` with a default gateway of `10.0.0.1` which they use to get to the Internet. The NAT rewrites source and destination addresses on packets traversing this special router to allow those private IPv4 clients to communicate with the public IPv4 network.

This guide differs from a normal NAT network where instead of a local LAN the internal IPv4 subnet is comprised of remote VPN clients.
- The frontend server opens a wireguard VPN tunnel with the private address `10.0.0.1/24`.
- The backend server opens a wireguard VPN tunnel with the private address `10.0.0.2/24`. Additional VPN clients could be added with addresses like `10.0.0.3/24`, `10.0.0.4/24`, and so on by adding their respective wireguard pubkeys to the frontend server's wireguard configuration.

Normally VPN's are configured to route all traffic through the tunnel. The service forwarding use case differs in that the described backend server routes the traffic of only specific services via the VPN while other software on the same machine reaches the Internet normally via their default gateway.

```bash
## Confirm the ln user's userid
# id -u ln
1000

## Show default route
# ip route show
default via 142.250.179.1 dev eth0 proto static metric 100
10.0.0.0/24 dev wg0 proto kernel scope link src 10.0.0.2 metric 50
142.250.179.0/24 dev eth0 proto kernel scope link src 142.250.179.110 metric 100

## Show route table 5000
# ip route show table 5000
default via 10.0.0.1 dev wg0 proto static metric 100
10.0.0.1 dev wg0 proto static scope link metric 100
```

The method described in this guide uses `ip rule add uidrange 1000-1001 lookup 5000` to direct network traffic of processes running as userid 1000 and 1001 via a separate arbitrarily numbered routing table 5000. `lightningd` and any other process run as the `ln` user will use `10.0.0.1` as the gateway.

Lastly the frontend's iptables contains rules to port forward TCP `9735` to the backend `10.0.0.2`. The lightningd service accepts this incoming connection and sends TCP reply packets via the same gateway `10.0.0.1`. This prevents leaks that could identify the location of the backend server.

Note: Aside from the service forwarding use case the frontend server described in this guide could also be useful as a general purpose VPN server for arbitrary other desktops, laptops, or mobile phones with wireguard clients.

# Requirements
- Both Frontend and Backend Servers
  - Both the frontend and backend servers require wireguard. Linux Kernel 5.6 and later include wireguard by default. Older Linux distributions often offer wireguard via DKMS or CentOS Plus EL8 has `kernel-plus` which includes wireguard maintained by the upstream authors. See the [wireguard install guide](https://www.wireguard.com/install/) for distro-specific kernel guidance.
- Frontend Server
  - Any Linux server where you have manual control over the kernel and iptables. e.g. cheap $5/mo virtual machines would work.
  - Public IP address (or otherwise ability to listen to specific incoming TCP ports)
- Backend Server
  - This guide requires NetworkManager-1.31.2+ for [uidrange parameter for routing rules](https://github.com/NetworkManager/NetworkManager/commit/972d1ba0469b3a43fcee0ba3e04cccf06f6e2a00). You can achieve similar results without NetworkManager but you need to figure out for yourself how to make the configurations persistent.
    - Fedora 34 and EL8 users can most conveniently upgrade with the [NetworkManager COPR](https://copr.fedorainfracloud.org/coprs/networkmanager/) where 1.32 is the current stable branch. It is not ideal that this method currently requires bleeding edge NetworkManager but at least this repo is maintained by the authors of NetworkManager. Talk to the authors at #nm on Libera.chat if you run into any issues with these builds.
  - The backend machine does not require any incoming public ports. The backend server instigates an outgoing VPN connection to the frontend server, then incoming connections are port forwarded to the backend's private IPv4 VPN address.

# Configuration
## Step 1: Choose your own IPv4 Addresses
The configuration examples included with this guide will refer to the following demo addresses. These are examples that you must customize for your deployment scenario.

Name            | Demo Address    | Description
----------------|-----------------|------------------------------
FRONTENDPUBLIC  | 142.250.179.110 | Public IPv4 address of the frontend
FRONTENDPRIVATE | 10.0.0.1        | RFC1918-type private IPv4 address of frontend VPN
BACKENDPRIVATE  | 10.0.0.2        | RFC1918-type private IPv4 address of backend VPN
WIREGUARDPORT   | 12345           | UDP port doesn't matter, pick something that isn't filtered

## Step 2: Generate Wireguard Keys
**frontend server**
```bash
## GENERATE KEYS INTO TEMPORARY FILES
# wg genkey | tee frontend.key | wg pubkey > frontend.pub
# cat frontend.key
mH7ne9DOC4jCESHYjzsc91qDSVXhOLY3oPiADurPMHc=
# cat frontend.pub
tDbXnzbg81/H9HY1a7AhO+G7FsDkOuzbXJA6k8mGbiM=

## AFTER YOU HAVE COPIED THESE KEY STRINGS YOU CAN DELETE THESE FILES
```

**backend server**

```bash
## GENERATE KEYS INTO TEMPORARY FILES
# wg genkey | tee backend.key | wg pubkey > backend.pub
# cat backend.key
yKWZSjXoOaoEHF+wBmZVU+bIs9KGF49dD7pxxvghwH8=
# cat backend.pub
AaxtOheAXrhGAIrOboi2Cjq+u2vNmVDb3rSQSv02NXU=

## Generate Preshared Key known by both servers
# wg genpsk > frontend-backend.psk
# cat frontend-backend.psk
rzZQ8bvdD1W1wPzHKFWO5k1D/ddsGT9lYQPp05Cvdn0=

## AFTER YOU HAVE COPIED THESE KEY STRINGS YOU CAN DELETE THESE FILES
```

## Step 3: Setup the Wireguard VPN Tunnel
**NetworkManager Wireguard device Frontend** `/etc/NetworkManager/system-connections/wg0.nmconnection`

```ini
[connection]
id=wg0
type=wireguard
interface-name=wg0

[wireguard]
listen-port=12345
private-key=mH7ne9DOC4jCESHYjzsc91qDSVXhOLY3oPiADurPMHc=
private-key-flags=0

# Add more of these entries below to define additional VPN clients
[wireguard-peer.AaxtOheAXrhGAIrOboi2Cjq+u2vNmVDb3rSQSv02NXU=]
preshared-key=rzZQ8bvdD1W1wPzHKFWO5k1D/ddsGT9lYQPp05Cvdn0=
preshared-key-flags=0
allowed-ips=10.0.0.2/32

[ipv4]
address1=10.0.0.1/24
method=manual
```

**NetworkManager Wireguard device Backend** `/etc/NetworkManager/system-connections/wg0.nmconnection`
```ini
[connection]
id=wg0
type=wireguard
interface-name=wg0
permissions=

[wireguard]
listen-port=12345
private-key=yKWZSjXoOaoEHF+wBmZVU+bIs9KGF49dD7pxxvghwH8=

[wireguard-peer.tDbXnzbg81/H9HY1a7AhO+G7FsDkOuzbXJA6k8mGbiM=]
endpoint=142.250.179.110:12345
preshared-key=rzZQ8bvdD1W1wPzHKFWO5k1D/ddsGT9lYQPp05Cvdn0=
preshared-key-flags=0
persistent-keepalive=25
allowed-ips=0.0.0.0/0;

[ipv4]
address1=10.0.0.2/24
dns-search=
ignore-auto-routes=true
method=manual
never-default=true
route1=0.0.0.0/0,10.0.0.1,100
route1_options=table=5000
routing-rule1=priority 100 from 0.0.0.0/0 uidrange 1000-1001 table 5000

[ipv6]
addr-gen-mode=stable-privacy
dns-search=
method=ignore

[proxy]
```

- Set permissions `chmod 600 /etc/NetworkManager/system-connections/wg0.nmconnection`
- Reload configurations with `nmcli connection reload`
- Stop the frontend's firewall `systemctl stop firewalld` which should allow the VPN to connect.
- Test `ping 10.0.0.1` from backend then `ping 10.0.0.2` from the frontend. Also see [Troubleshooting](#trouleshooting) below.

## Step 4: Setup Frontend iptables firewall
We are permanently disabling the distro's firewall daemon in favor of manual iptables configuration.

- `systemctl disable firewalld`
- `systemctl stop firewalld`
- `dnf install iptables-services`
- `dnf enable iptables`
- `dnf enable ip6tables`

**Frontend `/etc/sysconfig/iptables`**

This example is far more complicated than necessary but it is based on a widely tested and known-good recipe. You will need to customize addresses and ports below.

```
*nat
:PREROUTING ACCEPT [0:0]
:INPUT ACCEPT [0:0]
:OUTPUT ACCEPT [0:0]
:POSTROUTING ACCEPT [0:0]
:WIREGUARD_PRT - [0:0]
# port forwards
-A PREROUTING  -p tcp -m tcp -d 142.250.179.110 --dport 9735 -j DNAT --to-destination 10.0.0.2:9735

-A POSTROUTING -j WIREGUARD_PRT
-A WIREGUARD_PRT -s 10.0.0.0/24 -d 224.0.0.0/24 -j RETURN
-A WIREGUARD_PRT -s 10.0.0.0/24 -d 255.255.255.255/32 -j RETURN
-A WIREGUARD_PRT -s 10.0.0.0/24 ! -d 10.0.0.0/24 -p tcp -j MASQUERADE --to-ports 1024-65535
-A WIREGUARD_PRT -s 10.0.0.0/24 ! -d 10.0.0.0/24 -p udp -j MASQUERADE --to-ports 1024-65535
-A WIREGUARD_PRT -s 10.0.0.0/24 ! -d 10.0.0.0/24 -j MASQUERADE
COMMIT

*mangle
:PREROUTING ACCEPT [0:0]
:INPUT ACCEPT [0:0]
:FORWARD ACCEPT [0:0]
:OUTPUT ACCEPT [0:0]
:POSTROUTING ACCEPT [0:0]
:WIREGUARD_PRT - [0:0]
-A POSTROUTING -j WIREGUARD_PRT
-A WIREGUARD_PRT -o wg0 -p udp -m udp --dport 68 -j CHECKSUM --checksum-fill
COMMIT

*filter
:INPUT ACCEPT [0:0]
:FORWARD ACCEPT [0:0]
:OUTPUT ACCEPT [0:0]
:WIREGUARD_FWI - [0:0]
:WIREGUARD_FWO - [0:0]
:WIREGUARD_FWX - [0:0]
:WIREGUARD_INP - [0:0]
:WIREGUARD_OUT - [0:0]
-A INPUT -m state --state RELATED,ESTABLISHED -j ACCEPT
-A INPUT -p icmp -j ACCEPT
-A INPUT -i lo -j ACCEPT
# ssh
-A INPUT -p tcp -m state --state NEW -m tcp --dport 22 -j ACCEPT
# wireguard
-A INPUT -p udp --dport 12345 -j ACCEPT

# port forward
-A FORWARD -m state -p tcp -d 10.0.0.2 --dport 9735 --state NEW,ESTABLISHED,RELATED -j ACCEPT

-A INPUT -j WIREGUARD_INP
-A FORWARD -j WIREGUARD_FWX
-A FORWARD -j WIREGUARD_FWI
-A FORWARD -j WIREGUARD_FWO
-A OUTPUT -j WIREGUARD_OUT

-A INPUT -j REJECT --reject-with icmp-host-prohibited
-A FORWARD -j REJECT --reject-with icmp-host-prohibited

-A WIREGUARD_FWI -d 10.0.0.0/24 -o wg0 -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT
-A WIREGUARD_FWI -o wg0 -j REJECT --reject-with icmp-port-unreachable
-A WIREGUARD_FWO -s 10.0.0.0/24 -i wg0 -j ACCEPT
-A WIREGUARD_FWO -i wg0 -j REJECT --reject-with icmp-port-unreachable
-A WIREGUARD_FWX -i wg0 -o wg0 -j ACCEPT
-A WIREGUARD_INP -i wg0 -p udp -m udp --dport 53 -j ACCEPT
-A WIREGUARD_INP -i wg0 -p tcp -m tcp --dport 53 -j ACCEPT
-A WIREGUARD_INP -i wg0 -p udp -m udp --dport 67 -j ACCEPT
-A WIREGUARD_INP -i wg0 -p tcp -m tcp --dport 67 -j ACCEPT
-A WIREGUARD_OUT -o wg0 -p udp -m udp --dport 53 -j ACCEPT
-A WIREGUARD_OUT -o wg0 -p tcp -m tcp --dport 53 -j ACCEPT
-A WIREGUARD_OUT -o wg0 -p udp -m udp --dport 68 -j ACCEPT
-A WIREGUARD_OUT -o wg0 -p tcp -m tcp --dport 68 -j ACCEPT
COMMIT
```

- `systemctl restart iptables`
- Test that you are still able to ssh into this server!
- Test `ping 10.0.0.1` from backend which should be working now.
- Allow the kernel to forward packets with `sysctl net.ipv4.ip_forward=1 -w`

## Step 5: Setup Backend iptables firewall

- `systemctl disable firewalld`
- `systemctl stop firewalld`
- `dnf install iptables-services`
- `dnf enable iptables`
- `dnf enable ip6tables`

**Backend `/etc/sysconfig/iptables`**

You do not need elaborate firewall rules on the backend. You need only to allow incoming connections to specified ports. In the example I explicitly allow connections to the LN TCP port 9735 only if they come from the wireguard VPN device. You probably want something similar because you don't want random port scans to be able to find the backend.

```
*filter
:INPUT ACCEPT [0:0]
:FORWARD ACCEPT [0:0]
:OUTPUT ACCEPT [0:0]
-A INPUT -m state --state RELATED,ESTABLISHED -j ACCEPT
-A INPUT -p icmp -j ACCEPT
-A INPUT -i lo -j ACCEPT
# allow incoming ssh
-A INPUT -p tcp -m state --state NEW -m tcp --dport 22 -j ACCEPT
# allow incoming LN only from the wireguard VPN device
-A INPUT -i wg0 -p tcp -m state --state NEW -m tcp --dport 9735 -j ACCEPT
-A INPUT -j REJECT --reject-with icmp-host-prohibited
-A FORWARD -j REJECT --reject-with icmp-host-prohibited
COMMIT
```

- `systemctl restart iptables`
- Test that you are still able to ssh into this server!
- From the public Internet you should now be able to TCP connect to the frontend IP and port number 9735.

## Step 6: Explicit LN IPv4 Address Advertising
Backend service will be unable to autodetect their public IPv4 address so you need to override the address it advertises to the network.

**c-lightning** `.lightning/config`
```ini
announce-addr=142.250.179.110:9735
```

**LND** `.lnd/lnd.conf`
```ini
; See LND documentation for these two options
; externalip=
; externalhost=
```

# non-VPN Route Exclusions and Localhost
If you are running `lightningd` as the `ln` user all network traffic will route via the VPN frontend gateway `10.0.0.1`. This can cause complications for `bitcoind` or other services running as the `ln` user where you may want to add route exclusions.

- Perhaps you don't want the bulk traffic of bitcoin syncing to go over the VPN because that external network provider charges for data transfer. e.g. Perhaps you want your frontend to be on AWS due to the in-built DoS protection. They charge per gigabyte of outgoing data transfer and there's no good reason for that bulk traffic to traverse the metered network.
- Perhaps you have multiple servers on a local network alongside your backend server and you want to be able to directly connect with each other.

**TODO: ADD ROUTE EXCLUSION EXAMPLES HERE .. not exactly sure how to do it in .nmconnection syntax yet**

**Loopback already works** - Due to the above complications you may find it easier to run bitcoind as a separate non-root user. The lightning daemon communicates with a bitcoind RPC usually via localhost or `127.0.0.1`. The loopback destination address is understood to be local so it does not try to reach it via the VPN gateway so bitcoind running anywhere on the same backend machine is accessible via the loopback address.

# Troubleshooting
- Sometimes you will be unable to ping the backend's VPN private address from the frontend because the tunnel has not yet been established by the backend. By default wireguard establishes tunnels only while traffic tries to traverse it. In normal operation your LN node will make outgoing TCP connections via the tunnel so this should not be an issue. It can however be confusing during configuration testing so it is worthwhile to be aware of how it works. You could keep a `ping 10.0.0.1` running from the backend to ensure the tunnel is kept open.
- As the backend non-root user try `links` or `wget` with services like `https://www.whatismyip.com` to determine if you are successfully routing via the frontend server IPv4 address.
- In a separate terminal run `journalctl -f`. Some configuration errors will be visible in these logs.
- You may need `tcpdump` running on various interfaces with appropriate filters to see if traffic is going through at all.

# FAQ

- Why not IPv6?
  - The same approach could be applied to IPv6 but I find it more difficult to add simple DoS protection layers on top of this.

# TODO
- **Alternative means of user-specific or service-specific routing may be possible without uidrange.** For example iptables UID owner match along with fwmark might achieve the same goal. If you have a working alternative method I am interested in hearing about it because it would be desirable to have a way to do this without requiring a bleeding edge version of NetworkManager.
- **Alternate configs for non-NetworkManager distros** Let me know what works for you and this guide will be expanded.
