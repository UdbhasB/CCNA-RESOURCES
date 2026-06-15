# Connecting a Router to a LAN — A Guided Walkthrough

*Based on Packet Tracer 6.4.3.3 (CCNA 1). The goal here is understanding, not just typing commands. Each step explains what you're doing and why it matters.*

---

## 1. The Big Picture

A router's whole job is to move traffic *between* networks. In this lab you have two routers (R1 and R2) joined by a **WAN link** (the serial connection), and each router has two **LANs** hanging off its Ethernet ports.

```
   PC1, PC2 LANs            WAN link              LANs PC3, PC4
   192.168.10.0  ──┐                                 ┌── 10.1.1.0
   192.168.11.0  ──┤ R1 ===== serial (209.165.200.224/30) ===== R2 ├── 10.1.2.0
                   └─ G0/0, G0/1                    S0/0/0          G0/0, G0/1 ─┘
```

In this activity the **serial (WAN) link is already configured and the routing protocol (EIGRP) is already running.** Your actual job is small but fundamental: **bring the Ethernet interfaces to life so the LANs can reach the rest of the network.** Everything else is observing and verifying.

The work breaks into three parts:

1. **Look** at the router's current state (`show` commands).
2. **Configure** the Ethernet interfaces.
3. **Verify** that it all works.

---

## 2. Reading the Addressing Table

Before touching a router, understand the plan you're implementing.

| Device | Interface | IP Address | Role |
|--------|-----------|------------|------|
| R1 | G0/0 | 192.168.10.1 | Gateway for the PC1 LAN |
| R1 | G0/1 | 192.168.11.1 | Gateway for the PC2 LAN |
| R1 | S0/0/0 (DCE) | 209.165.200.225 /30 | WAN link to R2 |
| R2 | G0/0 | 10.1.1.1 | Gateway for the PC3 LAN |
| R2 | G0/1 | 10.1.2.1 | Gateway for the PC4 LAN |
| R2 | S0/0/0 | 209.165.200.226 /30 | WAN link to R1 |

A few things worth noticing:

- **The router interface is the LAN's default gateway.** PC1 (192.168.10.10) is told its gateway is 192.168.10.1 — that's R1's G0/0. Whenever PC1 needs to talk to *anything off its own subnet*, it hands the packet to that interface.
- **The /30 on the serial link** (mask 255.255.255.252) gives only 2 usable addresses — exactly enough for the two ends of a point-to-point link. WAN links don't need more.
- **DCE** on R1's serial port just means R1 supplies the clocking for the link. Not something you configure in this lab, but that's what the label means.

---

## 3. Concepts You Need First

Three ideas make the rest of the lab make sense.

**Command modes.** The Cisco CLI is layered, and the prompt tells you where you are:

| Prompt | Mode | What you can do |
|--------|------|-----------------|
| `R1>` | User EXEC | Basic look-only commands |
| `R1#` | Privileged EXEC | All `show` commands, save config, reboot |
| `R1(config)#` | Global config | Change device-wide settings |
| `R1(config-if)#` | Interface config | Change one interface |

You move *down* with commands (`enable`, `configure terminal`, `interface ...`) and back *up* with `exit` or `end`. In this lab the console password is **cisco** and the privileged EXEC password is **class**.

**Interfaces start shut off.** A brand-new router Ethernet interface is **administratively down** by default — Cisco's safe default so nothing goes live by accident. You'll see this in Part 1, and the single most common beginner mistake is forgetting to turn the interface on with `no shutdown`.

**Connected vs. learned routes.** A router knows about a network in one of two ways:
- **Connected (code `C`)** — the network is plugged directly into one of its own interfaces. The moment you bring up G0/0, R1 *automatically* knows about 192.168.10.0.
- **Learned (code `D` for EIGRP)** — another router advertised the network. R1 learns about R2's LANs this way.

If a destination is in *neither* category, the router has nowhere to send the packet, so **it drops it.** This is why your config matters: an interface that's down means a network nobody can reach.

---

## 4. Part 1 — Look Before You Touch

The point of this part is to read the router's current state. Click a device → **CLI** tab to get a terminal.

### See every interface in detail
```
R1# show interfaces
```
Long output — full statistics for every interface. To narrow it to one:
```
R1# show interfaces serial 0/0/0
```
In that output you can read off the IP (`209.165.200.225/30`) and the bandwidth (`BW 1544 Kbit` — standard T1 speed). The line `Serial0/0/0 is up, line protocol is up` is the phrase you're hunting for: it means the link is alive.

Now look at an Ethernet port:
```
R1# show interfaces gigabitEthernet 0/0
```
Notice the difference: it says **`administratively down`** and there's **no IP address** yet. That's the unconfigured state you're about to fix. (You can still read its burned-in MAC address — `000d.bd6c.7d01` — and its bandwidth, `1000000 Kbit` = 1 Gbps.)

### See everything at a glance
```
R1# show ip interface brief
```
This is the command you'll lean on most — a one-line-per-interface summary of address and status. Run it on both routers and you can answer the survey questions: each router has **2 serial** interfaces; R1 has **6 Ethernet** ports (2 Gigabit + 4 Fast) while R2 has **2**. The Gigabit ports run ten times faster than the Fast Ethernet ones.

### See what the router can reach
```
R1# show ip route
```
Right now R1 shows only **one connected route** (`C 209.165.200.224/30`, the serial link) because that's the only interface currently up. The LANs aren't there yet — that's the gap you close in Part 2.

---

## 5. Part 2 — Configure the Interfaces

This is the heart of the lab. The pattern is identical for every Ethernet interface, so learn it once.

### The four-line pattern
```
R1# configure terminal
R1(config)# interface gigabitethernet 0/0      ← pick the interface
R1(config-if)# ip address 192.168.10.1 255.255.255.0   ← give it the gateway address
R1(config-if)# no shutdown                     ← turn it ON
R1(config-if)# description LAN connection to S1 ← document what it connects to
```

What each line does:
- **`interface gigabitethernet 0/0`** — drops you into config for that one port (prompt changes to `config-if`).
- **`ip address ...`** — assigns the address *and the matching subnet mask.* The mask tells the router which part of the address is "the network," which is how it builds its connected route.
- **`no shutdown`** — the critical one. This flips the port from administratively down to up. Watch for the confirmation messages:
  ```
  %LINK-5-CHANGED: Interface GigabitEthernet0/0, changed state to up
  %LINEPROTO-5-UPDOWN: Line protocol on Interface GigabitEthernet0/0, changed state to up
  ```
  Both lines saying "up" is your success signal.
- **`description ...`** — pure documentation. It doesn't affect traffic, but six months later it tells you (or the next engineer) what's plugged in.

### Confirm it works
```
R1(config-if)# end
R1# ping 192.168.10.10
```
A row of `!` marks means success. (The first `.` is normal — that's just the very first packet waiting on ARP to resolve, so 4/5 success is expected and fine.)

### Repeat for the rest
Do the same four-line pattern for:
- **R1 G0/1** → `192.168.11.1 255.255.255.0`, description "LAN connection to S2"
- **R2 G0/0** → `10.1.1.1 255.255.255.0`
- **R2 G0/1** → `10.1.2.1 255.255.255.0`

### Save your work
The running config lives in RAM and disappears on reboot. Copy it to NVRAM so it survives:
```
R1# copy running-config startup-config
```
(Often typed in shorthand: `copy run start`.) Do this on **both** routers. Forgetting this step is why a correctly configured router can come back from a reboot with nothing configured.

---

## 6. Part 3 — Verify Everything

Configuration without verification is just hoping. Two commands close the loop.

### Check interface status
```
R1# show ip interface brief
```
After your work, **3 interfaces on each router** should read `up / up` (two Ethernet + one serial). One thing this summary *doesn't* show is the **subnet mask** — to see that, use `show running-config`, `show interfaces`, or `show ip protocols`.

### Check the routing table
```
R1# show ip route
```
Now the table should be full. You should see:
- **3 connected routes (`C`)** — the two LANs you just enabled plus the serial link.
- **2 EIGRP routes (`D`)** — the two LANs on the *other* router, learned automatically.

The sanity check: the topology has **5 networks total** (4 LANs + 1 WAN). 3 connected + 2 learned = 5. The numbers matching is proof the router knows the whole network. If they don't match, an interface is still down — go back to Part 2.

### Test end-to-end
The real proof is traffic flowing across the whole topology:
```
PC1> ping PC4        (192.168.10.10 → 10.1.2.10, crosses both routers and the WAN)
R2#  ping 192.168.11.10   (R2 reaching one of R1's LANs)
```
Both should succeed. *(The switches in this lab aren't configured, so you can't ping them — that's expected, not a fault.)*

---

## 7. Quick Command Reference

| Command | What it does |
|---------|--------------|
| `show interfaces [type num]` | Full statistics for all / one interface |
| `show ip interface brief` | One-line status + IP for every interface |
| `show ip route` | The routing table (C = connected, D = EIGRP) |
| `configure terminal` | Enter global config mode |
| `interface g0/0` | Enter config for one interface |
| `ip address <ip> <mask>` | Assign address + mask |
| `no shutdown` | Turn the interface ON |
| `description <text>` | Label the interface |
| `end` / `exit` | Step back up a mode level |
| `copy running-config startup-config` | Save config so it survives reboot |

---

## 8. The Five Things to Remember

1. **A router connects networks** — its interface is each LAN's default gateway.
2. **Interfaces are off by default** — `no shutdown` is mandatory, and the easiest step to forget.
3. **Address + mask together** create the connected route the router needs.
4. **C routes are direct, D routes are learned** — and connected + learned should equal the total number of networks.
5. **Save to NVRAM**, or your work vanishes on the next reboot.

Master this one repeatable interface-configuration pattern and you've learned the foundation that nearly every later router lab builds on.
