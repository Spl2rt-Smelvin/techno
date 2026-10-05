# Lab 5: Linux Routing and Unbound DNS Implementation

* **Course Module:** Network Services & Infrastructure
* **Student:** Ken-Andro Lüüs
* **Date:** 2026-10-05
* **Target Topology:** Multi-homed Linux Gateway with Ubuntu Client

---

## 1. Executive Summary & Topology Design

This lab documents the configuration and validation of a multi-homed Linux router connecting distinct broadcast domains, followed by the deployment of a centralized recursive caching DNS resolver using Unbound.

### Network Architecture

```text
               Windows Host
                    |
     [ Host-Only Management Network: 192.168.56.0/24 ]
                    |
        +-----------+-----------+
        |       Router VM       |
        |  Host-Only: .56.12    |
        +-----+-----------+-----+
              |           |
  Internal Network        Internal Network
     (labnet-a)              (labnet-b)
    10.10.10.1/24           10.10.20.1/24
              |
        +-----+-----+
        | Ubuntu VM |
        |   Node    |
        | .10.10.20 |
        +-----------+
```

### Interface Allocation Matrix

| Node | Interface | Network Type | Assigned IP/CIDR | Purpose |
| :--- | :--- | :--- | :--- | :--- |
| **Router** | `enp0s3` | NAT | DHCP (Dynamic) | Upstream Internet / Package Retrieval |
| **Router** | `enp0s8` | Host-Only | `192.168.56.12/24` | Administrative SSH Management |
| **Router** | `enp0s9` | Internal (`labnet-a`) | `10.10.10.1/24` | Gateway interface for LAN A |
| **Router** | `enp0s10` | Internal (`labnet-b`) | `10.10.20.1/24` | Gateway interface for LAN B |
| **Ubuntu Client** | `enp0s3` | NAT | DHCP (Dynamic) | Package access |
| **Ubuntu Client** | `enp0s8` | Host-Only | `192.168.56.20/24` | Administrative SSH Management |
| **Ubuntu Client** | `enp0s9` | Internal (`labnet-a`) | `10.10.10.20/24` | Primary lab communication interface |

---

## 2. Lab 5.A — Router Implementation & Transit Verification

### Step 1: Kernel IPv4 Forwarding Configuration
By default, the Linux kernel operates as an end-host and drops packets with foreign destination IPs. Forwarding was enabled persistently across reboots by modifying sysctl configurations:

```bash
echo "net.ipv4.ip_forward = 1" | sudo tee /etc/sysctl.d/99-router.conf
sudo sysctl -p /etc/sysctl.d/99-router.conf
```

**Verification:**
```bash
$ sysctl net.ipv4.ip_forward
net.ipv4.ip_forward = 1
```

### Step 2: Router Netplan Interface Configuration
On the Router VM, the interfaces were mapped and configured persistently under `/etc/netplan/`:

```yaml
network:
  version: 2
  ethernets:
    enp0s3:
      dhcp4: true
    enp0s8:
      addresses:
        - 192.168.56.12/24
    enp0s9:
      addresses:
        - 10.10.10.1/24
    enp0s10:
      addresses:
        - 10.10.20.1/24
```

### Step 3: Ubuntu Client Route Definition
To route packets destined for `labnet-b` through the router gateway, `/etc/netplan/50-cloud-init.yaml` was configured on the Ubuntu node:

```yaml
network:
  version: 2
  ethernets:
    enp0s3:
      dhcp4: true
    enp0s8:
      addresses:
        - 192.168.56.20/24
    enp0s9:
      addresses:
        - 10.10.10.20/24
      routes:
        - to: 10.10.20.0/24
          via: 10.10.10.1
      nameservers:
        addresses: [10.10.10.1]
```

Permissions and deployment:
```bash
sudo chmod 600 /etc/netplan/*.yaml
sudo netplan apply
```

### Step 4: Routing Table & Transit Verification
Validating that the Linux kernel selects the `enp0s9` interface for destinations on `10.10.20.0/24`:

```bash
$ ip route get 10.10.20.1
10.10.20.1 via 10.10.10.1 dev enp0s9 src 10.10.10.20 uid 1000
    cache
```

Ping execution from Ubuntu client to the Router's `labnet-b` interface:
```bash
$ ping -c 3 10.10.20.1
PING 10.10.20.1 (10.10.20.1) 56(84) bytes of data.
64 bytes from 10.10.20.1: icmp_seq=1 ttl=64 time=0.482 ms
64 bytes from 10.10.20.1: icmp_seq=2 ttl=64 time=0.512 ms
64 bytes from 10.10.20.1: icmp_seq=3 ttl=64 time=0.495 ms

--- 10.10.20.1 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2048ms
rtt min/avg/max/mdev = 0.482/0.496/0.512/0.012 ms
```

---

## 3. Lab 5.B — DNS Recursive Caching Resolver (Unbound)

### Step 1: Server Configuration
Installed `unbound` on the Router VM and created `/etc/unbound/unbound.conf.d/lab.conf`:

```text
server:
    interface: 10.10.10.1
    interface: 10.10.20.1
    access-control: 10.10.10.0/24 allow
    access-control: 10.10.20.0/24 allow

    # Authoritative local zone records
    local-zone: "lab.local." static
    local-data: "router.lab.local. IN A 10.10.10.1"
    local-data: "ubuntu.lab.local. IN A 10.10.10.20"
```

Restart and service verification:
```bash
sudo systemctl restart unbound
sudo systemctl status unbound
```

### Step 2: Resolver Status Verification
Confirmed upstream nameserver attachment using `resolvectl` on the Ubuntu client:

```bash
$ resolvectl status enp0s9
Link 4 (enp0s9)
    Current Scopes: DNS
         Protocols: -DefaultRoute +LLMNR -mDNS -DNSOverTLS DNSSEC=no/unsupported
Current DNS Server: 10.10.10.1
       DNS Servers: 10.10.10.1
```

### Step 3: Cache Timing Analysis & Scalability Discussion

#### Cold Cache (Iterative Resolution)
```text
$ dig example.com
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 62121
;; flags: qr rd ra; QUERY: 1, ANSWER: 2, AUTHORITY: 0, ADDITIONAL: 1

;; ANSWER SECTION:
example.com.        300 IN  A   104.20.23.154
example.com.        300 IN  A   172.66.147.243

;; Query time: 6 msec
;; SERVER: 127.0.0.53#53(127.0.0.53) (UDP)
;; WHEN: Mon Oct 05 06:17:31 UTC 2026
```

#### Warm Cache (Cached Memory Lookup)
```text
$ dig example.com
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 19076
;; flags: qr rd ra; QUERY: 1, ANSWER: 2, AUTHORITY: 0, ADDITIONAL: 1

;; ANSWER SECTION:
example.com.        296 IN  A   104.20.23.154
example.com.        296 IN  A   172.66.147.243

;; Query time: 1 msec
;; SERVER: 127.0.0.53#53(127.0.0.53) (UDP)
;; WHEN: Mon Oct 05 06:17:35 UTC 2026
```

#### Technical Justification: Local DNS Scalability
* **`/etc/hosts` Limitations:** Modifying host resolution locally requires distributing updates to $N$ independent files across the network, leading to operational complexity $O(N)$ and vulnerability to configuration drift.
* **Unbound Centralization:** Using authoritative `local-data` entries creates a centralized single point of administration. Updates propagate to all endpoints immediately upon cache expiration or query time.

### Step 4: Network Packet Inspection (`tcpdump`)
Captured DNS wire traffic on the Router VM listening on interface `enp0s9` during client lookup:

```bash
$ sudo tcpdump -i enp0s9 -n port 53
listening on enp0s9, link-type EN10MB (Ethernet), snapshot length 262144 bytes
06:21:50.413069 IP 10.10.10.20.57037 > 10.10.10.1.53: 34816+ [1au] A? example.com. (52)
06:21:50.413305 IP 10.10.10.1.53 > 10.10.10.20.57037: 34816$ 2/0/1 A 104.20.23.154, A 172.66.147.243 (72)
```

* **Packet Analysis:**
  * **Query (Line 1):** Ubuntu node (`10.10.10.20`) transmits a UDP datagram from ephemeral port `57037` to the gateway's port `53` asking for the `A` record of `example.com` (TxID `34816`).
  * **Response (Line 2):** Resolver (`10.10.10.1`) returns the answered resource records to port `57037` under matching TxID `34816` in 236 microseconds.

---

## 4. Deliverable: Diagnostic Failure Analysis

| Failure Scenario | Observed Symptom | Distinguishing Command | Root Cause Analysis | Remediation |
| :--- | :--- | :--- | :--- | :--- |
| **(a) Unreachable Resolver Address** | Resolution hangs; `dig` exits after timeout: `;; connection timed out; no servers could be reached`. | `ping <target-IP>` or `ip route get <target-IP>` fails to confirm path to host. | Client Netplan points to an unallocated or down IP address. | Reconfigure `/etc/netplan/*.yaml` with the correct gateway IP (`10.10.10.1`) and run `sudo netplan apply`. |
| **(b) Query Refused via Access-Control** | Immediate failure output; header reports `status: REFUSED` with 0 answers. | `dig @10.10.10.1 <domain>` returns instant `REFUSED` code. | Unbound lacks an explicit `access-control: <subnet> allow` directive for the source subnet. | Append `access-control: 10.10.10.0/24 allow` to `/etc/unbound/unbound.conf.d/lab.conf` and reload the service. |
| **(c) Stale Local Host Entry** | Client connects to outdated target IP; `ping` and `curl` hit wrong host while `dig` outputs the correct IP. | `getent hosts <hostname>` differs from `dig @10.10.10.1 <hostname> +short`. | `/etc/nsswitch.conf` prioritizes `files` prior to `dns`. An outdated entry in `/etc/hosts` overrides recursive resolution. | Purge or update the offending line within `/etc/hosts`. |
