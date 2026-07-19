# Technical Notes — Network Baseline and Linux Firewall Hardening

These notes preserve the deeper concepts, commands, troubleshooting steps, and observations behind the polished project case study. It's intended to act as a detailed reference, which supplements the main findings and final results. Those can be found in the project's [README](./README.md).

## 1. Virtualization and Lab Design

### Core Terms

| Term | Meaning |
|---|---|
| Host | The physical Windows computer running VMware |
| Guest | An operating system running inside a virtual machine |
| Hypervisor | Software that creates and manages virtual machines |
| Virtual disk | A file presented to the guest as a storage device |
| Virtual NIC | A software-defined network interface |
| Snapshot | A restoration point for a VM's configuration and disk state |

VMware Workstation Pro is a Type 2 hypervisor because it runs as an application on top of the host operating system. The virtual machines still behave as independent computers with their own kernels, storage, interfaces, users, services, and network stacks.

Virtual machines were selected for this project because they provide:

- Isolation from the main host environment
- Revertible snapshots
- Reproducible hardware and software configurations
- Flexible software-defined networks
- A safe environment for scanning and service changes
- Easy expansion into later multi-system projects

Snapshots are useful for experiments, but they are not a substitute for backups. They depend on the original VM files and can consume significant storage if maintained indefinitely.

### Guest Roles

**Ubuntu Server** represented a typical Linux server being administered and hardened. A server installation without a desktop environment encouraged direct use of the command line, service manager, logs, and network tools.

**Kali Linux** represented the analyst workstation. Kali provided Nmap, Wireshark, SSH, curl, and other tools, but the distribution itself was not the source of the learning. The important part was understanding the traffic each tool generated and the evidence each tool returned.

## 2. NAT and Host-Only Networking

Each VM used two virtual NICs.

### NAT Adapter

The NAT adapter provided:

- DHCP addressing
- A default gateway
- DNS configuration
- Outbound access for package updates

In the completed environment:

```text
Kali NAT:   192.168.209.128/24
Ubuntu NAT: 192.168.209.129/24
Gateway:    192.168.209.2
```

NAT allows internal private addresses to communicate through a translating gateway. The gateway maintains state so return traffic can be associated with the correct internal connection.

### Host-Only Adapter

The host-only network used:

```text
Network: 10.10.10.0/24
Kali:    10.10.10.20/24
Ubuntu:  10.10.10.10/24
Host:    10.10.10.1/24
```

This network connected the two VMs and the host-side VMware adapter without directly bridging them onto the physical home LAN.

The host-only interface was assigned:

- A static address
- No default gateway
- No DNS server

Only the NAT adapter needed a default route. Adding gateways to both interfaces could create competing defaults and unpredictable path selection.

## 3. Interfaces, Addressing, and Routing

Useful commands:

```bash
ip -br link
ip -br address
ip route
ip route get <destination>
ip neighbor
```

These commands answer different questions:

- Does the interface exist?
- Is the virtual cable connected?
- Is the interface administratively up?
- Does it have an IP address?
- Is there a route for the destination?
- Has the destination's MAC address been resolved?
- Which interface and source address will Linux use?

Ubuntu's final routing table included:

```text
default via 192.168.209.2 dev ens33
10.10.10.0/24 dev ens37
192.168.209.0/24 dev ens33
```

Linux chooses the most specific matching route. Traffic to `10.10.10.20` matches the `/24` lab route, while an internet destination falls back to the default route.

A default route can be thought of as the answer to:

> Where should traffic go when no more specific route exists?

### Connectivity Tests

```bash
ping -c 4 10.10.10.20
ping -c 4 1.1.1.1
ping -c 4 ubuntu.com
```

These isolate different dependencies:

| Test | Primary Meaning |
|---|---|
| Lab IP | Local addressing, interfaces, routes, and Layer 2 delivery |
| `1.1.1.1` | External IP routing without requiring DNS |
| Hostname | External routing and DNS resolution |

Low VM-to-VM latency was expected because the packets traveled through a software-defined switch on the same physical host. External traffic had to pass through VMware NAT, Windows, the home router, the ISP, and the remote network.

## 4. ARP and Local Delivery

Ubuntu and Kali were on the same `/24` host-only subnet, so no router was required for direct communication.

Before Kali could send an Ethernet frame to Ubuntu, it needed the destination MAC address. Kali therefore issued an ARP request:

```text
Who has 10.10.10.10? Tell 10.10.10.20
```

Ubuntu replied with its virtual MAC address. Kali stored the mapping in its neighbor table.

The encapsulation path was conceptually:

```text
Application data
→ TCP, UDP, or ICMP
→ IPv4 packet
→ Ethernet frame
→ VMware virtual switch
```

The IP destination identified the interface at Layer 3, while the MAC destination allowed delivery across the local Layer 2 segment.

## 5. Unexpected Alternate Routing Observation

An especially useful experiment occurred when Kali's host-only interface was disconnected.

The direct route disappeared, and `ip route get 10.10.10.10` reported:

```text
10.10.10.10 via 192.168.209.2 dev eth0 src 192.168.209.128
```

This showed that Kali had fallen back to the NAT default gateway.

Despite the direct host-only path being unavailable, the ordinary ping still received replies. A capture on Ubuntu's `ens37` interface showed:

```text
10.10.10.1  > 10.10.10.10: ICMP echo request
10.10.10.10 > 10.10.10.1:  ICMP echo reply
```

Ubuntu did not see Kali's original NAT address. It saw the VMware host-side address on VMnet1.

The safest conclusion is that the VMware/host networking path forwarded or proxied the traffic between its virtual networks and translated the source address. The experiment demonstrated that:

- Disabling one interface removes one path, not necessarily every path
- A system may have multiple network identities
- A successful ping proves reachability but not the expected route
- `ip route get` shows the sender's selected path
- A capture on the destination shows how the packet actually arrived
- Network segmentation must be verified

Forcing the disconnected interface confirmed the distinction:

```bash
ping -I eth1 -c 2 10.10.10.10
```

This failed because `eth1` was down. The ordinary ping succeeded only because Linux was free to choose the NAT route.

## 6. Services, Sockets, and Binding

The local socket inventory was collected with:

```bash
sudo ss -tulpn
```

Flags:

- `-t`: TCP
- `-u`: UDP
- `-l`: listening sockets
- `-p`: associated process
- `-n`: numerical addresses and ports

Examples:

```text
0.0.0.0:22
10.10.10.10:22
127.0.0.1:53
```

These have different meanings:

- `0.0.0.0:22`: listen on every IPv4 interface
- `10.10.10.10:22`: listen only on the selected address
- `127.0.0.1:53`: listen only on loopback

`ss` answers whether a process is listening locally. It does not, by itself, prove that another host can reach the service. Remote reachability also depends on:

- Interface state
- Routing
- Firewall policy
- Upstream network controls
- Service-specific access controls

After nginx was installed, its process requested a listening socket on TCP 80. Incoming traffic addressed to that destination port could then be delivered to nginx by the Linux network stack.

## 7. HTTP and curl

Kali tested nginx with:

```bash
curl http://10.10.10.10
curl -I http://10.10.10.10
```

curl acted as an HTTP client:

1. Establish a TCP connection
2. Send an HTTP request
3. Receive an HTTP response
4. Display the response
5. Close or reuse the connection

A representative request was:

```http
GET / HTTP/1.1
Host: 10.10.10.10
User-Agent: curl
Accept: */*
```

Because the connection used unencrypted HTTP, the request and response were readable in Wireshark.

## 8. SSH and Encryption

SSH was tested with:

```bash
ssh dminh@10.10.10.10
```

The first connection displayed a host-key warning because Kali had no previously trusted record for that server. Accepting the key stored it for comparison during future connections.

This is distinct from encryption:

- Encryption protects the session content from passive network observers
- Host-key verification helps confirm the identity of the server

A simplified SSH sequence was:

1. TCP three-way handshake
2. Client and server version exchange
3. Algorithm negotiation
4. Key exchange
5. Server host-key proof
6. Session-key activation
7. Encrypted authentication and interactive data

The key exchange allowed both endpoints to derive shared secret material without sending the final secret directly over the network. SSH then used efficient symmetric encryption for the ongoing session and integrity protection to detect modification.

After key establishment, Wireshark could still observe:

- Source and destination addresses
- TCP ports
- Direction
- Packet size
- Timing
- Duration
- Approximate traffic volume

It could not directly read the shell commands or responses in the encrypted session.

Modern encryption is normally not defeated by simply capturing traffic and attempting to "look through" it. Investigations usually focus instead on authorized endpoint evidence, authentication logs, process telemetry, credentials, configuration mistakes, or compromised systems where plaintext exists before encryption or after decryption.

## 9. Nmap and TCP SYN Scanning

The main scan was:

```bash
sudo nmap -sS -sV 10.10.10.10
```

Options:

- `-sS`: TCP SYN scan
- `-sV`: service and version detection

### Normal TCP Handshake

```text
Client → Server: SYN
Server → Client: SYN-ACK
Client → Server: ACK
```

### Nmap SYN Scan

For an open port:

```text
Nmap → Server: SYN
Server → Nmap: SYN-ACK
Nmap → Server: RST
```

Nmap learns that the port is open without completing a normal application connection.

For a closed port:

```text
Nmap → Server: SYN
Server → Nmap: RST
```

For a filtered port:

```text
Nmap → Server: SYN
Firewall drops packet
Nmap receives no definitive response
```

Common states:

| State | Interpretation |
|---|---|
| `open` | A service accepted or acknowledged the connection attempt |
| `closed` | The host responded, but no service was listening |
| `filtered` | A network control prevented a definitive conclusion |
| `open|filtered` | Nmap could not distinguish open from filtered |
| `closed|filtered` | Nmap could not distinguish closed from filtered |

Nmap scanning is not inherently invisible. Firewalls, IDS tools, endpoint telemetry, and server logs can detect the probes.

Version detection may send protocol-specific requests and compare the returned behavior with Nmap's fingerprint database. The baseline capture therefore contained requests generated by Nmap as well as the manually generated curl traffic.

## 10. Packet Capture and Wireshark

The baseline capture used:

```bash
sudo tcpdump -ni ens37 \
  -w ~/baseline-services.pcap \
  'arp or icmp or tcp port 22 or tcp port 80'
```

Important options:

- `-n`: do not translate addresses and ports to names
- `-i ens37`: capture on the host-only interface
- `-w`: save raw packets to a PCAP file

The filtered-state capture used:

```bash
sudo tcpdump -ni ens37 \
  -w ~/firewall-filtered.pcap \
  'tcp and (port 22 or port 80 or port 81)'
```

Useful Wireshark display filters:

```text
arp
icmp
http
tcp.port == 22
tcp.port == 80
```

The captures provided packet-level evidence rather than relying only on tool summaries.

## 11. UFW Rules and Least Privilege

The firewall policy used:

```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
```

The SSH exception was:

```bash
sudo ufw allow in on ens37 \
  from 10.10.10.20 \
  to any port 22 \
  proto tcp \
  comment 'allow SSH from kali on host-only lab'
```

This was narrower than a general `allow 22/tcp` rule because it required:

- The expected interface
- The expected source IP
- TCP
- The SSH destination port

This follows least privilege by allowing only the communication required for the lab.

The final policy was:

```text
Default: deny incoming
Default: allow outgoing
SSH: allowed only from Kali over ens37
nginx: stopped and disabled
```

Service reduction and firewall filtering were both used. Disabling nginx removed the listener, while UFW protected against unintended inbound exposure.

## 12. UFW Logging and Evidence Correlation

Logging was enabled with:

```bash
sudo ufw logging medium
```

A blocked event contained fields such as:

```text
[UFW BLOCK]
IN=ens37
SRC=10.10.10.20
DST=10.10.10.10
PROTO=TCP
SPT=<temporary source port>
DPT=80
SYN
```

Interpretation:

| Field | Meaning |
|---|---|
| `UFW BLOCK` | Firewall denied the packet |
| `IN=ens37` | Packet arrived through the host-only interface |
| `SRC` | Kali's lab IP |
| `DST` | Ubuntu's lab IP |
| `PROTO=TCP` | Transport protocol |
| `SPT` | Temporary client source port |
| `DPT` | Targeted service port |
| `SYN` | Attempt to initiate a TCP connection |

The experiment correlated four views:

1. **Nmap:** remote classification
2. **Wireshark:** packet behavior
3. **UFW logs:** firewall decision
4. **`ss` and `systemctl`:** local service truth

Comparing these four views made the firewall decision much easier to verify. Nmap showed the remote result, Wireshark showed the packet behavior, UFW recorded the decision, and `ss` showed the target’s actual local state.

## 13. Final Hardening and Validation

Final commands included:

```bash
sudo systemctl disable --now nginx
sudo systemctl is-active nginx
sudo systemctl is-enabled nginx

sudo systemctl is-active ssh
sudo systemctl is-enabled ssh

sudo ss -ltnup
sudo ufw status verbose
sudo ufw status numbered
```

Final result:

- nginx inactive and disabled
- SSH active and enabled
- No listener on TCP 80
- SSH listener on TCP 22
- UFW active
- Medium logging enabled
- Default deny incoming
- SSH permitted only from Kali over the host-only interface

Both VMs were configured to use UTC before the final evidence was collected so timestamps could be correlated consistently.

## 14. Reusable Firewall Workflow

The project produced a workflow applicable to other Linux systems:

1. Inventory listening services
2. Determine which services are required
3. Stop and disable unnecessary services
4. Apply a default-deny inbound policy
5. Permit only required protocols and ports
6. Restrict by interface and source where practical
7. Enable appropriate logging
8. Test from another system
9. Compare remote results with local state
10. Preserve configuration and evidence

The same process can be applied to other Linux systems by changing the permitted services trusted sources, and interfaces to match the system’s actual requirements.

## 15. Selected Commands

### System and Network Inspection

```bash
hostname
cat /etc/os-release
ip -br link
ip -br address
ip route
ip route get <destination>
ip neighbor
```

### Package and Service Management

```bash
sudo apt update
sudo apt full-upgrade -y
sudo systemctl status nginx
sudo systemctl start nginx
sudo systemctl disable --now nginx
sudo systemctl status ssh
```

### Socket Inventory

```bash
sudo ss -tulpn
sudo ss -ltnup
sudo ss -ltnp | grep -E ':(22|80)\b'
```

### Testing and Scanning

```bash
ping -c 4 10.10.10.10
curl -I http://10.10.10.10
ssh dminh@10.10.10.10
sudo nmap -sS -sV 10.10.10.10
sudo nmap -sS -sV -p 22,80,81 10.10.10.10
```

### Packet Capture

```bash
sudo tcpdump -ni ens37 -w ~/baseline-services.pcap \
  'arp or icmp or tcp port 22 or tcp port 80'

sudo tcpdump -ni ens37 -w ~/firewall-filtered.pcap \
  'tcp and (port 22 or port 80 or port 81)'
```

### Firewall

```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing

sudo ufw allow in on ens37 \
  from 10.10.10.20 \
  to any port 22 \
  proto tcp

sudo ufw logging medium
sudo ufw enable
sudo ufw status verbose
sudo ufw status numbered
```

### Logs

```bash
sudo journalctl -k -f | grep --line-buffered UFW
```

### Artifact Transfer

```bash
scp dminh@10.10.10.10:~/baseline-services.pcap .
scp dminh@10.10.10.10:~/firewall-filtered.pcap .
```
