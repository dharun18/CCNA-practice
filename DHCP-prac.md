# DHCP Lab — One Router Serving Two LANs (Build → Break → Fix)


---

## Why I built this

To communicate with systems every device will need an IP address, but how do we get the IP address is where DHCP comes in

Every network runs DHCP, and "a user can't get an IP" is one of the most common tickets a NOC handles. Reading about DORA wasn't enough for me. I wanted to:

1. Configure a Cisco router as a **DHCP server for two separate networks** at the same time.
2. **Break it on purpose** and fix each fault using `show` commands, the way I'd have to on a live network.

This README is my notes from that lab: the theory I needed, my configs, what I broke, and how I found each problem.

---

## Table of contents

1. [What DHCP is, from first principles](#1-what-dhcp-is-from-first-principles)
2. [How DORA works](#2-how-dora-works)
3. [How one router knows which pool to use](#3-how-one-router-knows-which-pool-to-use)
4. [Lab topology](#4-lab-topology)
5. [Configuration](#5-configuration)
6. [Verification](#6-verification)
7. [What I broke and how I fixed it](#7-what-i-broke-and-how-i-fixed-it)
8. [My troubleshooting method](#8-my-troubleshooting-method)
9. [Symptom → cause cheat-sheet](#9-symptom--cause-cheat-sheet)
10. [Key takeaways](#10-key-takeaways)
11. [What's next](#11-whats-next)

---

## 1. What DHCP is, from first principles

Every device on an IP network needs **four things** before it can talk beyond its own cable:

| Parameter | Why it's needed |
|---|---|
| **IP address** | Its own identity on the network |
| **Subnet mask** | Tells it who is "local" (talk directly) and who is "remote" (go through the gateway) |
| **Default gateway** | The router to send remote traffic to |
| **DNS server** | Turns `google.com` into an IP address |

**The problem:** picture an office with 500 laptops, phones and printers, where people move around and guests come and go every day. Typing these four values by hand on every device is slow and error-prone (duplicate IPs, typos in the gateway), and nobody can keep up with it.

**The solution:** a server *leases* these values automatically. A device joins, asks "who am I?", and gets everything it needs in under a second.

**Analogy I use:** DHCP works like a hotel front desk. You walk in with no room (no IP) and ask for one (Discover). The desk offers room 211 (Offer), you say "I'll take 211" (Request), and they hand you the key card with the Wi-Fi password and checkout time (Acknowledge: IP, gateway, DNS and lease). When you check out, the room goes back into the pool.

**Key facts:**

- Runs over **UDP**: the server listens on **port 67** and the client on **port 68**.
- The address is a **lease**, not permanent. The client tries to renew at **50%** of the lease (T1) and again at **87.5%** (T2).
- A client that gets no answer gives itself a **169.254.x.x** address (APIPA). If you see 169.254, DHCP failed.

---

## 2. How DORA works

>C: ACK — IP + mask + gateway + DNS + lease



| Step | Message | Sent as | Why |
|---|---|---|---|
| **D** | Discover | Broadcast | The client has no IP and doesn't know where the server is |
| **O** | Offer | Unicast or broadcast | The server proposes an address |
| **R** | Request | **Broadcast** | Tells *every* DHCP server which offer it accepted |
| **A** | Acknowledge | Unicast or broadcast | Makes it official, and the client configures itself |

A detail that stuck with me: the **Request is still a broadcast** even though the client already knows the server. If two DHCP servers both made offers, the broadcast is how the one that wasn't picked finds out.

---

## 3. How one router knows which pool to use

My router has **two pools** and **two LANs**. When a Discover arrives, how does it decide whether the PC gets a 192.168.1.x or a 192.168.2.x address?

**It looks at the interface the Discover arrived on.**

- A Discover arrives on **Gig0/0/0** (IP 192.168.1.1), so the router uses the pool whose `network` statement contains 192.168.1.1, which is `LAN1-POOL`.
- A Discover arrives on **Gig0/0/1** (IP 192.168.2.1), so it uses `LAN2-POOL`.

Three things follow from this, and I tested each one in Section 7:

1. If the interface is **down**, the Discover never arrives.
2. If no pool's `network` matches the **interface subnet**, the router has nothing to offer.
3. The pool hands out exactly what I typed in it, **including my mistakes**.

> **Why I didn't need a relay:** DORA begins with a broadcast, and routers don't forward broadcasts. Here the router *is* the DHCP server and sits directly on both LANs, so the broadcasts reach it. In bigger networks the DHCP server is often a separate machine in a server room, and each gateway is set up with `ip helper-address <server-ip>` to forward the broadcasts to it as unicast. That's my next lab.

---

## 4. Lab topology

<img width="902" height="645" alt="image" src="https://github.com/user-attachments/assets/a4cdcb75-c394-4426-a7b5-c44b81aecff1" />


<!-- Add your Packet Tracer screenshot here -->
<!-- ![Packet Tracer topology](screenshots/topology.png) -->

### Addressing plan

| Network | Router interface | Gateway IP | DHCP hands out | Excluded |
|---|---|---|---|---|
| LAN 1 — 192.168.1.0/24 | Gig0/0/0 | 192.168.1.1 | .11 – .254 | .1 – .10 (gateway and future static devices) |
| LAN 2 — 192.168.2.0/24 | Gig0/0/1 | 192.168.2.1 | .11 – .254 | .1 – .10 |

| Device | Connected to | Gets IP from |
|---|---|---|
| PC0 | Switch0 Fa0/2 | LAN1-POOL |
| PC1 | Switch0 Fa0/3 | LAN1-POOL |
| PC2 | Switch1 Fa0/2 | LAN2-POOL |

The switches need **no configuration** in this lab. Out of the box, every port is in VLAN 1 and simply forwards frames, so each switch is one LAN.

---

## 5. Configuration

### Router — interfaces and DHCP server

```bash
Router> enable
Router# configure terminal
Router(config)# hostname R1
R1(config)# no ip domain-lookup

! ---------- Interfaces (ISR4331 ports are shut down by default) ----------
R1(config)# interface GigabitEthernet0/0/0
R1(config-if)# description LAN1-TO-SWITCH0
R1(config-if)# ip address 192.168.1.1 255.255.255.0
R1(config-if)# no shutdown
R1(config-if)# exit

R1(config)# interface GigabitEthernet0/0/1
R1(config-if)# description LAN2-TO-SWITCH1
R1(config-if)# ip address 192.168.2.1 255.255.255.0
R1(config-if)# no shutdown
R1(config-if)# exit

! ---------- Reserve addresses that must never be handed out ----------
R1(config)# ip dhcp excluded-address 192.168.1.1 192.168.1.10
R1(config)# ip dhcp excluded-address 192.168.2.1 192.168.2.10

! ---------- Pool for LAN 1 ----------
R1(config)# ip dhcp pool LAN1-POOL
R1(dhcp-config)# network 192.168.1.0 255.255.255.0
R1(dhcp-config)# default-router 192.168.1.1
R1(dhcp-config)# dns-server 8.8.8.8
R1(dhcp-config)# exit

! ---------- Pool for LAN 2 ----------
R1(config)# ip dhcp pool LAN2-POOL
R1(dhcp-config)# network 192.168.2.0 255.255.255.0
R1(dhcp-config)# default-router 192.168.2.1
R1(dhcp-config)# dns-server 8.8.8.8
R1(dhcp-config)# exit

R1(config)# end
R1# copy running-config startup-config
```

**Why each line matters:**

| Command | Purpose |
|---|---|
| `no shutdown` | Router interfaces start administratively down. Without this, nothing works |
| `ip dhcp excluded-address` | Keeps the gateway and static devices out of the pool, which prevents duplicate IPs |
| `network` | Defines the range the pool hands out, and is how the router matches the pool to an interface |
| `default-router` | The gateway IP the PCs receive |
| `dns-server` | The DNS server IP the PCs receive |

### PCs

On each PC: **Desktop → IP Configuration → DHCP**. Or from the command prompt:

```
C:\> ipconfig /release
C:\> ipconfig /renew
C:\> ipconfig /all
```

---

## 6. Verification

### On the PCs

```
PC0> ipconfig /all
   IPv4 Address....................: 192.168.1.11
   Subnet Mask.....................: 255.255.255.0
   Default Gateway.................: 192.168.1.1
   DNS Servers.....................: 8.8.8.8

PC2> ipconfig /all
   IPv4 Address....................: 192.168.2.11
   Subnet Mask.....................: 255.255.255.0
   Default Gateway.................: 192.168.2.1
   DNS Servers.....................: 8.8.8.8
```

I check **all four values**, not just the IP. A PC can have a valid IP and a wrong gateway (see [Break #3](#break-3--a-typo-in-the-default-router)).

### End-to-end test

```
PC0> ping 192.168.1.1     ← own gateway
PC0> ping 192.168.2.11    ← PC2 on the other LAN (proves routing works too)
```

### On the router

```
R1# show ip dhcp binding
IP address       Client-ID/Hardware address    Lease expiration    Type
192.168.1.11     <PC0 MAC>                     --                  Automatic
192.168.1.12     <PC1 MAC>                     --                  Automatic
192.168.2.11     <PC2 MAC>                     --                  Automatic
```

<!-- Replace the block above with your own output / screenshot -->

| Command | What it tells me |
|---|---|
| `show ip interface brief` | Are both LAN interfaces up/up with the right IPs? |
| `show ip dhcp binding` | Which PC (by MAC) got which IP |
| `show ip dhcp pool` | Pool range, how many addresses are leased, and whether it's full |
| `show running-config \| section dhcp` | Every DHCP line in one place, which makes typos easy to spot |
| `show ip dhcp conflict` | Addresses the router found already in use |

### Watching DORA live

In Packet Tracer **Simulation mode** I set the event filter to DHCP only, ran `ipconfig /renew` on PC0, and stepped through all four packets. The Discover leaves PC0 with source `0.0.0.0` and destination `255.255.255.255`, and Switch0 floods it out every port. The router answers, and the other PC simply ignores it.

<!-- ![DORA in simulation mode](screenshots/dora-simulation.png) -->

---

## 7. What I broke and how I fixed it

The aim was to practise fault-finding the way I would on a real network: **start from the symptom, collect evidence, and only change something once I know why.**

### Break #1 — Interface left shut down

**What I did:** `shutdown` on `GigabitEthernet0/0/1`. Leaving out `no shutdown` is also the most common mistake on a fresh ISR4331.

**Symptom:** PC0 and PC1 were fine. **PC2 got `169.254.x.x`.**

**Investigation:**

1. `ipconfig` on PC2 showed 169.254.x.x (APIPA), so DHCP failed for this PC.
2. **Only LAN 2 was affected**, so the DHCP service itself was working. The fault had to be something specific to LAN 2.
3. The link light between Router Gig0/0/1 and Switch1 Fa0/1 was **red** in Packet Tracer.
4. On the router:
   ```
   R1# show ip interface brief
   Interface              IP-Address      OK? Method Status                Protocol
   GigabitEthernet0/0/0   192.168.1.1     YES manual up                    up
   GigabitEthernet0/0/1   192.168.2.1     YES manual administratively down down
   ```

**Root cause:** The interface was administratively down, so PC2's Discover never reached the router.

**Fix:**
```bash
R1(config)# interface GigabitEthernet0/0/1
R1(config-if)# no shutdown
```
Then `ipconfig /renew` on PC2, which got 192.168.2.11.

**Lesson:** "One network broken, the other fine" narrows the problem instantly. Check Layer 1 and Layer 2 (link lights, `show ip interface brief`) before you blame DHCP.

---

### Break #2 — Wrong network in the pool

**What I did:** In `LAN2-POOL` I replaced `network 192.168.2.0` with `network 192.168.3.0`, the kind of typo that's easy to make at 2 a.m.

**Symptom:** PC2 got `169.254.x.x` again. This time **the interface was up/up** and the link light was green.

**Investigation:**

1. `show ip interface brief` showed Gig0/0/1 up/up with 192.168.2.1. Layer 1 and 2 were fine.
2. `show ip dhcp binding` had **no 192.168.2.x entries**, so the router never gave out an address on LAN 2.
3. `show ip dhcp pool` and `show running-config | section dhcp`:
   ```
   ip dhcp pool LAN2-POOL
    network 192.168.3.0 255.255.255.0
    default-router 192.168.2.1
   ```
4. The router picks a pool by matching the **receiving interface's subnet** (192.168.2.0/24). No pool covered that subnet, so it had nothing to offer.

**Root cause:** The pool's `network` statement didn't match the interface subnet.

**Fix:**
```bash
R1(config)# ip dhcp pool LAN2-POOL
R1(dhcp-config)# no network 192.168.3.0 255.255.255.0
R1(dhcp-config)# network 192.168.2.0 255.255.255.0
```

**Lesson:** The interface IP and the pool network **must be in the same subnet**. A link that's up and a pool that exists can still add up to no DHCP.

---

### Break #3 — A typo in the default-router

**What I did:** In `LAN1-POOL` I changed `default-router 192.168.1.1` to `default-router 192.168.1.254`.

**Symptom:** This was the sneakiest one. PC0 **got an IP**, could ping PC1 (same LAN), but **couldn't ping PC2** on the other LAN.

**Investigation:**

1. DHCP looked healthy. PC0 had 192.168.1.11, and the binding was in `show ip dhcp binding`.
2. Same-LAN pings worked but remote ones failed, which pointed at the **gateway**. PCs only use the gateway for traffic to other networks.
3. `ipconfig /all` on PC0: `Default Gateway: 192.168.1.254`. Nothing on the network owns that address.
4. `show running-config | section dhcp` confirmed the typo in the pool.

**Root cause:** A wrong `default-router` in the pool. DHCP worked perfectly; it just handed out bad information.

**Fix:**
```bash
R1(config)# ip dhcp pool LAN1-POOL
R1(dhcp-config)# default-router 192.168.1.1
```
Then on PC0 and PC1:
```
C:\> ipconfig /release
C:\> ipconfig /renew
```

**Lesson:** "Got an IP" doesn't mean "DHCP is correct." Also, **fixing the pool doesn't fix PCs that already have a lease**. They keep the old gateway until they renew.

---

### Break #4 — Running out of addresses

**What I did:** Simulated a pool that's too small by changing the LAN 1 exclusion to `ip dhcp excluded-address 192.168.1.1 192.168.1.253`, which leaves only **one** address (.254) to give out.

**Symptom:** PC0 got 192.168.1.254. **PC1 got `169.254.x.x`**, even though it's on the same switch, the same LAN and the same pool.

**Investigation:**

1. Same LAN, one PC working and one not. That rules out the interface and the pool network, so I suspected **capacity**.
2. `show ip dhcp binding` had only one LAN 1 entry: PC0 at .254.
3. `show ip dhcp pool` for LAN1-POOL showed the leased addresses equal to the addresses available. **The pool was full.**
4. `show running-config | section dhcp` showed the oversized exclusion range.

**Root cause:** Pool exhaustion. There were more clients than free addresses.

**Fix:**
```bash
R1(config)# no ip dhcp excluded-address 192.168.1.1 192.168.1.253
R1(config)# ip dhcp excluded-address 192.168.1.1 192.168.1.10
```
Then `ipconfig /renew` on PC1, which got an address.

**Lesson:** In a real office this shows up as "everyone who came in late on Monday has no network." The usual causes are a subnet that's too small, leases that are too long for a high-turnover network (guest Wi-Fi), or stale bindings. The fixes are a bigger subnet, a shorter `lease`, or `clear ip dhcp binding *`.

---

## 8. My troubleshooting method

After these four faults, this is the order I follow for any "no IP" ticket:


    A["PC has 169.254.x.x / no IP"] --> B{"Is the PC set to DHCP?"}
    B -- No --> B1["Switch PC to DHCP"]
    B -- Yes --> C{"Link lights green?<br/>Router interface up/up?<br/>(show ip interface brief)"}
    C -- No --> C1["Fix cable / no shutdown"]
    C -- Yes --> D{"Pool network matches<br/>the interface subnet?<br/>(show run | section dhcp)"}
    D -- No --> D1["Correct the network statement"]
    D -- Yes --> E{"Free addresses left?<br/>(show ip dhcp pool)"}
    E -- No --> E1["Shrink exclusions / bigger subnet / shorter lease"]
    E -- Yes --> F["Renew on PC and check debug / simulation mode"]

    G["PC HAS an IP but something fails"] --> H{"Local ping works,<br/>remote fails?"}
    H -- Yes --> H1["Check default gateway in ipconfig /all → fix default-router"]
    H -- No --> I{"Ping by IP works,<br/>by name fails?"}
    I -- Yes --> I1["Check dns-server in the pool"]


The idea: **follow the packet.** Start at the PC, move towards the router, and confirm each step before suspecting the next.

---

## 9. Symptom → cause cheat-sheet

| Symptom | Most likely cause | Command that proves it |
|---|---|---|
| `169.254.x.x` on one LAN only | Router interface down, or pool network doesn't match | `show ip interface brief`, `show run \| section dhcp` |
| `169.254.x.x` on some PCs in the same LAN | Pool exhausted | `show ip dhcp pool`, `show ip dhcp binding` |
| Got IP, local works, remote fails | Wrong `default-router` | `ipconfig /all` |
| Got IP, ping by IP works, by name fails | Wrong `dns-server` | `ipconfig /all`, `nslookup` |
| Intermittent drop-offs | Duplicate IP (static device inside the DHCP range) | `show ip dhcp conflict` |
| PCs get an unexpected subnet | Rogue DHCP server (e.g. a home router plugged in) | `ipconfig /all` (check the DHCP server field) → prevent with DHCP snooping |

---

## 10. Key takeaways

- **169.254.x.x means the client never got an ACK.** That's where troubleshooting starts.
- **A router picks the pool by the interface the request arrives on**, so the interface subnet and the pool `network` must match.
- **"One LAN broken, the other fine" narrows the search instantly.** Look at what's unique to that LAN.
- **Getting an IP isn't the finish line.** Verify all four parameters.
- **Fixing the pool doesn't fix existing leases.** Clients need to renew.
- **Always exclude** the gateway and static devices before handing out addresses.

---


