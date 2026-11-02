# Enterprise Network Routing & Security Lab

A hands-on networking and security lab environment built using **Linux network namespaces**, **virtual ethernet pairs (`veth`)**, **static routing**, and **`iptables` firewall policy rules** on a single host machine running Kali Linux.

---

## 📋 Overview

This project simulates a multi-site enterprise infrastructure consisting of a **Headquarters (HQ) Office** and a **Branch Office** linked over a Point-to-Point WAN connection. 

By leveraging native Linux kernel features (network namespaces), this setup models four isolated virtual network nodes—two routers and two client workstations—without requiring the overhead of running multiple Virtual Machines (VMs).

---

## 🌐 Network Topology

```
+-------------------------------------------------------------------------+
|                                HQ Office                                |
|  [ Kali Client ] (192.168.1.10/24) <---> (192.168.1.1/24) [ Router HQ ] |
+-------------------------------------------------------------------------+
                                                                | (10.0.0.1/30)
                                                          [ WAN Link 10.0.0.0/30 ]
                                                                | (10.0.0.2/30)
+-------------------------------------------------------------------------+
|                               Branch Office                             |
| [ Kali Client ] (192.168.2.10/24) <---> (192.168.2.1/24) [Router Branch]|
+-------------------------------------------------------------------------+
```

### IP Addressing Scheme

| Node Name | Network Namespace | Interface Name | IP Address / Subnet | Role / Description |
| :--- | :--- | :--- | :--- | :--- |
| **HQ Client** | `client-hq` | `veth-c-hq` | `192.168.1.10/24` | Workstation in HQ LAN |
| **HQ Router (LAN)** | `router-hq` | `veth-r-hq` | `192.168.1.1/24` | Gateway for HQ LAN |
| **HQ Router (WAN)** | `router-hq` | `veth-hq-wan` | `10.0.0.1/30` | WAN Interface (HQ) |
| **Branch Router (WAN)** | `router-branch` | `veth-branch-wan` | `10.0.0.2/30` | WAN Interface (Branch) |
| **Branch Router (LAN)**| `router-branch` | `veth-r-branch` | `192.168.2.1/24` | Gateway for Branch LAN |
| **Branch Client** | `client-branch` | `veth-c-branch` | `192.168.2.10/24` | Workstation in Branch LAN |

---

## ✨ Key Features & Concepts Demonstrated

* **Network Virtualization:** Uses `ip netns` to isolate network stacks (interfaces, routing tables, and firewall rules) on a single Linux machine.
* **Virtual Cables (`veth`):** Establishes point-to-point links connecting clients to their respective gateways and linking routers over a simulated WAN subnet.
* **IP Forwarding:** Enables Linux kernel packet routing via `sysctl net.ipv4.ip_forward=1` on intermediate nodes.
* **Static Routing:** Configures explicit routing entries using `ip route` to enable reachability between non-adjacent subnets (`192.168.1.0/24` $\leftrightarrow$ `192.168.2.0/24`).
* **Traffic Control & Firewalling:** Employs `iptables` rules in the `FORWARD` chain to enforce access control lists (ACLs) and restrict traffic flow between enterprise sites.
* **ICMP Verification & TTL Inspection:** Validates end-to-end communication and verifies hop counts via TTL decrement values.

---

## 🚀 Setup & Implementation Steps

### Prerequisites
Make sure your system is updated and has the necessary tools installed:
```bash
sudo apt update
sudo apt install iproute2 iptables iputils-ping -y
```

---

### Step 1: Create Network Namespaces
Create isolated network namespaces for both offices (routers and client nodes):
```bash
sudo ip netns add router-hq
sudo ip netns add client-hq
sudo ip netns add router-branch
sudo ip netns add client-branch
```

---

### Step 2: Virtual Cables (veth Pairs) & Namespace Assignment
Create virtual ethernet pairs and bind them to their respective namespaces:

```bash
# Link HQ Client <--> HQ Router
sudo ip link add veth-c-hq type veth peer name veth-r-hq
sudo ip link set veth-c-hq netns client-hq
sudo ip link set veth-r-hq netns router-hq

# Link HQ Router <--> Branch Router (WAN)
sudo ip link add veth-hq-wan type veth peer name veth-branch-wan
sudo ip link set veth-hq-wan netns router-hq
sudo ip link set veth-branch-wan netns router-branch

# Link Branch Client <--> Branch Router
sudo ip link add veth-c-branch type veth peer name veth-r-branch
sudo ip link set veth-c-branch netns client-branch
sudo ip link set veth-r-branch netns router-branch
```

---

### Step 3: IP Addressing & Interface Activation

#### 1. HQ Client (`client-hq`)
```bash
sudo ip netns exec client-hq ip addr add 192.168.1.10/24 dev veth-c-hq
sudo ip netns exec client-hq ip link set veth-c-hq up
sudo ip netns exec client-hq ip link set lo up
```

#### 2. HQ Router (`router-hq`)
```bash
# LAN Interface
sudo ip netns exec router-hq ip addr add 192.168.1.1/24 dev veth-r-hq
sudo ip netns exec router-hq ip link set veth-r-hq up

# WAN Interface
sudo ip netns exec router-hq ip addr add 10.0.0.1/30 dev veth-hq-wan
sudo ip netns exec router-hq ip link set veth-hq-wan up
sudo ip netns exec router-hq ip link set lo up
```

#### 3. Branch Router (`router-branch`)
```bash
# WAN Interface
sudo ip netns exec router-branch ip addr add 10.0.0.2/30 dev veth-branch-wan
sudo ip netns exec router-branch ip link set veth-branch-wan up

# LAN Interface
sudo ip netns exec router-branch ip addr add 192.168.2.1/24 dev veth-r-branch
sudo ip netns exec router-branch ip link set veth-r-branch up
sudo ip netns exec router-branch ip link set lo up
```

#### 4. Branch Client (`client-branch`)
```bash
sudo ip netns exec client-branch ip addr add 192.168.2.10/24 dev veth-c-branch
sudo ip netns exec client-branch ip link set veth-c-branch up
sudo ip netns exec client-branch ip link set lo up
```

---

### Step 4: Enable IP Forwarding on Routers
Permit the kernel inside the router namespaces to forward incoming packets:
```bash
sudo ip netns exec router-hq sysctl -w net.ipv4.ip_forward=1
sudo ip netns exec router-branch sysctl -w net.ipv4.ip_forward=1
```

---

### Step 5: Configure Default Gateways & Static Routing

#### Set Default Gateways on Clients
```bash
sudo ip netns exec client-hq ip route add default via 192.168.1.1
sudo ip netns exec client-branch ip route add default via 192.168.2.1
```

#### Add Static Routes on Routers
```bash
# Tell HQ Router how to reach the Branch network
sudo ip netns exec router-hq ip route add 192.168.2.0/24 via 10.0.0.2

# Tell Branch Router how to reach the HQ network
sudo ip netns exec router-branch ip route add 192.168.1.0/24 via 10.0.0.1
```

---

## 🧪 Verification & Security Testing

### 1. Test End-to-End Connectivity
Ping from **HQ Client** (`192.168.1.10`) to **Branch Client** (`192.168.2.10`):
```bash
sudo ip netns exec client-hq ping -c 4 192.168.2.10
```
* **Expected Result:** `0% packet loss`. Notice that the `ttl=62` (decremented by 2 hops across `router-hq` and `router-branch`), confirming active multi-hop routing.

---

### 2. Implement Security Firewall Policy
Block outbound traffic initiated from HQ LAN to Branch LAN by placing an `iptables` rule in the `FORWARD` chain of `router-hq`:
```bash
sudo ip netns exec router-hq iptables -A FORWARD -s 192.168.1.0/24 -d 192.168.2.0/24 -j DROP
```

---

### 3. Verify Firewall Rule Enforcement
Attempt to ping from **HQ Client** to **Branch Client** again:
```bash
sudo ip netns exec client-hq ping -c 4 192.168.2.10
```
* **Expected Result:** `100% packet loss`.

Inspect packet counters on `router-hq` to verify the rule blocked traffic:
```bash
sudo ip netns exec router-hq iptables -L FORWARD -n -v
```
* **Output:** Counter shows `4 pkts / 336 bytes DROP`, confirming that packets were intercepted and discarded by the rule.

---

## 🧹 Cleanup Script
To reset your environment and delete all created namespaces and virtual links, execute:
```bash
sudo ip netns del router-hq
sudo ip netns del client-hq
sudo ip netns del router-branch
sudo ip netns del client-branch
```

---

## 📄 License
Distributed under the MIT License. Feel free to modify and use this lab for learning and teaching networking principles.