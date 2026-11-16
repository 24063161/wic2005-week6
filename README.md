# Week 6 Lab & Assignment — Virtualisation, Overlays and Software Forwarding

Student: FAN CHENHAO  
Student ID: 24063161 (NNNN: 3161, NN: 61)  
IPv4 Allocation: `10.61.0.0/16`  
IPv6 Allocation: `fd00:3161::/32`  

---

## Lab Wrap-up Answers

### 1. Linux building blocks behind one Mininet host and one Mininet cable
- **Mininet Host**: A dedicated Linux **network namespace (`netns`)** bound to a `bash` shell process. It provides isolated networking resources including independent network interfaces, routing tables, socket endpoints, and ARP/FDB tables.
- **Mininet Cable**: A **virtual Ethernet (`veth`) pair**. A veth pair acts like a bi-directional virtual pipe where packets sent into one peer emerge from the other. One peer interface is assigned inside the host network namespace (e.g., `h1-eth0`), and the opposite peer is plugged into the switch/bridge namespace (e.g., `s1-eth1`).

### 2. MTU and throughput measurements for underlay and overlay, 50-byte difference, and IPv6 cost
- **Measurements**:
  - **MTU**:
    - Underlay (`h1-eth0`): 1500 bytes.
    - IPv4 VXLAN Overlay (`vxlan100`): 1450 bytes. Maximum Don't Fragment (DF) ICMP payload is 1422 bytes (1422 + 28 bytes IP/ICMP = 1450 bytes). 1423 bytes fails with `message too long`.
    - IPv6 VXLAN Overlay (`vxlan106`): 1430 bytes. Maximum DF ICMP payload is 1402 bytes (1402 + 28 bytes IP/ICMP = 1430 bytes). 1403 bytes fails with `message too long`.
  - **Throughput (iperf3 5s)**:
    - Underlay (`10.61.0.2`): 33.4 GBytes transferred, **57.3 Gbits/sec**.
    - Overlay (`10.61.100.2`): 25.9 GBytes transferred, **44.5 Gbits/sec** (~77.6% of underlay).
- **Header-by-header breakdown of the 50-byte overhead (IPv4 underlay)**:
  - Inner Ethernet header: 14 bytes (6B Dest MAC + 6B Src MAC + 2B EtherType)
  - VXLAN header: 8 bytes (Flags + reserved + 24-bit VNI + reserved)
  - Outer UDP header: 8 bytes (Src Port + Dst Port 4789 + Length + Checksum)
  - Outer IPv4 header: 20 bytes (Standard IPv4 header without options)
  - **Total Overhead**: 14 + 8 + 8 + 20 = **50 bytes**.
- **Why the same tunnel costs 70 bytes over IPv6**:
  - The outer IPv4 header (20 bytes) is replaced with a standard IPv6 header, which has a fixed length of 40 bytes (+20 bytes).
  - Total overhead: 14 + 8 + 8 + 40 = **70 bytes**. The resulting MTU is 1500 - 70 = 1430 bytes.

### 3. Why small pings keep working during MTU mismatch while TCP almost stops
- A standard small ICMP echo request (56 bytes payload + 28 bytes IP/ICMP = 84 bytes) produces an encapsulated VXLAN packet of 84 + 50 = 134 bytes. This is far below the reduced 1400-byte underlay MTU, so packets traverse unhindered.
- TCP, on the other hand, negotiates an MSS based on the sender's overlay MTU (1450 - 40 = 1410 bytes). When transmitting bulk traffic, TCP emits full-sized packets resulting in 1500-byte encapsulated frames. Because intermediate/receiving devices have an MTU of 1400 bytes, these packets are discarded. Consequently, TCP suffers complete packet loss, triggers retransmission timeouts, and its congestion window collapses, reducing throughput to near zero (~419 Kbps).

### 4. What a Docker container has that a Mininet host does not
- While a Mininet host only virtualizes the **network stack** (network namespace), a Docker container is an isolated execution environment encompassing:
  1. **An Independent Filesystem**: A container has its own root filesystem image (e.g. Alpine Linux with its own `/etc`, `/bin`, libraries, package manager, and release files) mounted via pivot_root/chroot. Mininet hosts share the underlying host filesystem.
  2. **Process Isolation (PID Namespace)**: The container has its own isolated process table where its main application runs as PID 1.
  3. **Comprehensive Namespaces & Cgroups**: Additional namespaces (Mount, IPC, UTS, User) and control groups (cgroups) for enforcing hard CPU, memory, and I/O resource limits.

### 5. VXLAN Tunnel vs. Week 5 P4 Tunnel
- **Header Overhead**:
  - VXLAN adds **50 bytes** (IPv4) or **70 bytes** (IPv6) due to full L2-over-L3/L4 encapsulation.
  - The Week 5 custom P4 tunnel (`myTunnel`) added only **4 bytes** (16-bit `proto_id` + 16-bit `dst_id`).
- **Device Awareness on the Path**:
  - VXLAN requires only tunnel endpoints (**VTEPs**) to understand VXLAN; transit network switches simply forward standard UDP packets on port 4789.
  - The Week 5 P4 tunnel required **every transit switch** along the forwarding path to have P4 match-action tables specifically programmed to parse and forward `myTunnel` headers.
- **Loop Prevention**:
  - VXLAN leverages the **outer IP header TTL** (decremented at every L3 hop).
  - The Week 5 P4 tunnel initially had **no hop limit** (and relied on students adding an explicit custom hop limit field during the assignment).

---

## AI Use Declaration

- **Level**: Level 3 (Open)
- **Tool**: Antigravity (Google DeepMind Agent)
- **Scope & Role**: Used to assist in verifying experiment commands, reviewing network topologies, validating MTU boundary conditions, and evaluating open-ended network design questions.
