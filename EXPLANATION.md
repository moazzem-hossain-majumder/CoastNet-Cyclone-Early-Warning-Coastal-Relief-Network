# CoastNet Project — Full Technical Reference & Progress Log

This document explains **how the entire project was built**, phase by phase — every value used, every command typed, and the reasoning behind each decision. It's written so someone with no prior context on this project (a teammate, a grader, or future-you before a viva) can read it start to finish and understand the whole network.

Group members: Member 1 (ID 24241132), Member 2 (ID 23141013), Member 3 (ID 23201674).

---

## Phase 1 — Addressing Plan (Shared — all members)

### Base network calculation
Per the assignment's formula:
- **Octet 1**: last 4 digits of Member 1's ID (24241132) → 1, 1, 3, 2 → sum = 7 → padded = **07**
- **Octet 2**: last 2 digits of Member 2's ID (23141013) → **13**
- **Octets 3–4**: fixed at 0, mask /16

**Result: base network = `7.13.0.0/16`**

### Subnet sizing (host requirements → mask)
Each unit's "Hosts" figure from the assignment table (not the literal PC count — see Phase 2) was rounded up to the next power of 2 to determine the required mask:

| Unit | Hosts (given) | Required block | Mask |
|---|---|---|---|
| NDCC | 280 | 512 | /23 |
| CGB | 200 | 256 | /24 |
| RLH | 220 | 256 | /24 |
| TRP | 90 | 128 | /25 |
| BMD | 150 | 256 | /24 |
| FRC | 130 | 256 | /24 |
| COI | 40 | 64 | /26 |

Link segments: the 4-router Central Switch needed 8 addresses (/29); each of the 6 point-to-point router links needed 4 addresses (/30).

### VLSM tree (allocated in the assignment's required order: NDCC, CGB, RLH, TRP, BMD, FRC, COI, then links)

```
7.13.0.0/16
|-- 7.13.0.0/23   -> NDCC
|-- 7.13.2.0/24   -> CGB
|-- 7.13.3.0/24   -> RLH
|-- 7.13.4.0/25   -> TRP           (7.13.4.128/25 left unused)
|-- 7.13.5.0/24   -> BMD
|-- 7.13.6.0/24   -> FRC
|-- 7.13.7.0/26   -> COI
|-- 7.13.7.64/29  -> Central Switch (NDCC-CGB-RLH-TRP)
|-- 7.13.7.72/30  -> NDCC <-> RLH link
|-- 7.13.7.76/30  -> NDCC <-> BMD link
|-- 7.13.7.80/30  -> BMD <-> CGB link
|-- 7.13.7.84/30  -> CGB <-> NDCC link
|-- 7.13.7.88/30  -> RLH <-> FRC link
`-- 7.13.7.92/30  -> TRP <-> COI link
```
*(Full binary-split tree with unused branches is in `CoastNet_Final_Documentation.docx`.)*

### IP Address Table (final, devices + links)

| Unit | Router (LAN gateway) | Printer | Email Server | Other Server(s) | DHCP Pool (PCs) |
|---|---|---|---|---|---|
| NDCC | 7.13.0.1 | 7.13.0.2 | 7.13.0.3 | 7.13.0.4 (Web), 7.13.0.5 (DNS) | 7.13.0.10-7.13.1.254 |
| CGB | 7.13.2.1 | 7.13.2.2 | 7.13.2.3 | - | 7.13.2.10-7.13.2.254 |
| RLH | 7.13.3.1 | 7.13.3.2 | 7.13.3.3 | 7.13.3.4 (Web) | 7.13.3.10-7.13.3.254 |
| TRP | 7.13.4.1 | 7.13.4.2 | 7.13.4.3 | - | 7.13.4.10-7.13.4.126 |
| BMD | 7.13.5.1 | 7.13.5.2 | 7.13.5.3 | - | 7.13.5.10-7.13.5.254 |
| FRC | 7.13.6.1 | 7.13.6.2 | 7.13.6.3 | - | 7.13.6.10-7.13.6.254 |
| COI | 7.13.7.1 | 7.13.7.2 | 7.13.7.3 | - | 7.13.7.10-7.13.7.62 |

**Router-to-router link IPs:** Central Switch - NDCC .65, CGB .66, RLH .67, TRP .68. NDCC<->RLH: .73/.74. NDCC<->BMD: .77/.78. BMD<->CGB: .81/.82. CGB<->NDCC: .85/.86. RLH<->FRC: .89/.90. TRP<->COI: .93/.94.

### Worked example — how NDCC got `/23`
NDCC needed 280 host addresses. The nearest power of 2 that's big enough (after adding 2 for the network and broadcast addresses, 282) is 512 = 2^9. So NDCC needs **9 host bits**. Since a full address is 32 bits, the network (locked) portion is 32 - 9 = **23 bits** -> `/23`.

To see what that actually restricts: /23 locks the first 23 bits. Octets 1-2 (`7.13`) are fully locked (16 bits). Octet 3 has 8 bits, and 23 - 16 = 7 of them are locked, leaving only the **last bit** of octet 3 free. A free last bit means octet 3 can only be `0` or `1` in binary (`00000000` or `00000001`), giving two possible values. Combined with all 8 bits of octet 4 being free, that's 2 x 256 = 512 addresses total, spanning `7.13.0.0` through `7.13.1.255`. That's exactly the block reserved for NDCC.

---

## Phase 2 — Topology Build (Shared — all members)

**Decision:** every unit gets a router (Cisco 2911) + LAN switch + 2 PCs + 1 printer + servers, plus one shared switch (SW-CENTRAL) for the 4-router hub. LAN-facing links use Ethernet (onboard GigabitEthernet ports); router-to-router links use **Serial** (added via HWIC-2T modules), following standard Cisco/CCNA convention.

**DCE/DTE assignment** (the DCE side runs `clock rate 64000`):

| Link | DCE side |
|---|---|
| NDCC <-> RLH | NDCC |
| NDCC <-> BMD | NDCC |
| CGB <-> NDCC | NDCC |
| BMD <-> CGB | BMD |
| RLH <-> FRC | RLH |
| TRP <-> COI | TRP |

**Router interface counts:** NDCC=5, CGB=4, RLH=4, TRP=3, BMD=3, FRC=2, COI=2 -- this determined how many HWIC-2T modules each router needed (NDCC: 2 modules; all others: 1 module).

### Worked example — why NDCC needed 2 HWIC-2T modules
A base 2911 router ships with 3 onboard GigabitEthernet ports and no serial ports at all. NDCC's job in the topology touches: its own unit's LAN (1 Ethernet port), the Central Switch (1 Ethernet port), and three separate point-to-point serial links — to RLH, to BMD, and to CGB (3 serial ports). The 2 onboard Ethernet ports needed are already covered by the built-in GigabitEthernet0/0 and 0/1. But 0 serial ports exist until modules are added. One HWIC-2T module supplies exactly 2 serial ports — not enough for NDCC's 3 required serial links — so a **second** HWIC-2T was added, giving 4 serial ports total (one spare). Every other router in the topology needed at most 2 serial links, so a single HWIC-2T module was always enough for them.

---

## Phase 3 — Interface Addressing (CLI commands) (Shared — all members)

Every router got its hostname and interface IPs set via CLI. Pattern used on every router:

```
enable
configure terminal
hostname <NAME>
interface <interface-name>
 ip address <ip> <mask>
 [clock rate 64000]     <- only on the DCE side of a serial link
 no shutdown
 exit
end
copy running-config startup-config
```

**Why:** `no shutdown` is required because router interfaces are administratively off by default. `clock rate` is only needed on the DCE side of a serial link, since it's simulating the "leased line" providing timing -- the DTE side never gets this command. `copy running-config startup-config` saves the config so it survives a reload.

Printers and servers got static IPs via the GUI (Desktop -> IP Configuration), each with:
- IP/mask matching its unit's subnet
- Default gateway = the unit's router LAN IP
- DNS server = `7.13.0.5` (NDCC's central DNS) -- set on **every** device in **every** unit, per the PDF's "every device uses central DNS" requirement

### Worked example — configuring TRP's LAN interface
TRP's LAN gateway IP from the Phase 1 table is `7.13.4.1`, on the `/25` subnet (mask `255.255.255.128`). The actual commands typed were:
```
enable
configure terminal
interface GigabitEthernet0/0
 ip address 7.13.4.1 255.255.255.128
 no shutdown
 exit
end
copy running-config startup-config
```
Verification used `show ip interface brief`, which listed `GigabitEthernet0/0` with IP `7.13.4.1` and status `up/up` — confirming both that the address was accepted and that the interface is actually passing traffic (an interface can show an IP but still be `down/down` if `no shutdown` was forgotten, or if the far-end device isn't ready yet).

---

## Phase 4 — DHCP Configuration (CLI commands) (Member 1 — DHCP; a few static routes added here as fixes also count toward Member 2's Static Routing)

Four different DHCP patterns were required by the PDF.

### NDCC — DHCP server for itself, BMD, and CGB (the Warning Loop)
```
ip dhcp excluded-address 7.13.0.1 7.13.0.9
ip dhcp excluded-address 7.13.5.1 7.13.5.9
ip dhcp excluded-address 7.13.2.1 7.13.2.9
ip dhcp pool NDCC-POOL
 network 7.13.0.0 255.255.254.0
 default-router 7.13.0.1
 dns-server 7.13.0.5
ip dhcp pool BMD-POOL
 network 7.13.5.0 255.255.255.0
 default-router 7.13.5.1
 dns-server 7.13.0.5
ip dhcp pool CGB-POOL
 network 7.13.2.0 255.255.255.0
 default-router 7.13.2.1
 dns-server 7.13.0.5
```
**Why `excluded-address`:** without it, DHCP could hand a PC the same address as a static device (router/printer/server), causing an IP conflict.

### BMD and CGB — DHCP relay to NDCC
```
interface GigabitEthernet0/0
 ip helper-address 7.13.0.1
```
**Why:** DHCP requests are broadcasts, and routers don't forward broadcasts between subnets. `ip helper-address` catches the broadcast on the router facing the PCs and forwards it as a normal routed packet to the real DHCP server.

**Dependency discovered here:** this relay silently failed at first -- `ipconfig /renew` returned APIPA (`169.254.x.x`) addresses on BMD/CGB's PCs. The cause: the relayed packet needs an actual *route* to reach NDCC, and none existed yet (routing hadn't been configured). **Fix:** RIPv2 was configured early on NDCC, BMD, and CGB (normally a Phase 8 task) specifically to unblock this -- see the RIPv2 block in Phase 8/9 below, which was applied here first.

### RLH and TRP — their own local DHCP servers
```
! On RLH:
ip dhcp excluded-address 7.13.3.1 7.13.3.9
ip dhcp pool RLH-POOL
 network 7.13.3.0 255.255.255.0
 default-router 7.13.3.1
 dns-server 7.13.0.5

! On TRP:
ip dhcp excluded-address 7.13.4.1 7.13.4.9
ip dhcp pool TRP-POOL
 network 7.13.4.0 255.255.255.128
 default-router 7.13.4.1
 dns-server 7.13.0.5
```
These worked immediately with no routing dependency, since the server and its PCs share the same router.

### COI — uses TRP's DHCP server (relay)
```
! On TRP (added COI's pool to its own DHCP server):
ip dhcp excluded-address 7.13.7.1 7.13.7.9
ip dhcp pool COI-POOL
 network 7.13.7.0 255.255.255.192
 default-router 7.13.7.1
 dns-server 7.13.0.5

! On COI:
interface GigabitEthernet0/0
 ip helper-address 7.13.4.1
ip route 0.0.0.0 0.0.0.0 Serial0/1/0
```
**Why the default route here:** this is actually the PDF's separately-required "Default Gateway" static route for COI (*"COI can reach the rest of the network only through TRP"*) -- configured here ahead of schedule because the relay needed it to function at all.

### FRC — uses RLH's DHCP server (relay)
```
! On RLH (added FRC's pool):
ip dhcp excluded-address 7.13.6.1 7.13.6.9
ip dhcp pool FRC-POOL
 network 7.13.6.0 255.255.255.0
 default-router 7.13.6.1
 dns-server 7.13.0.5

! On FRC:
interface GigabitEthernet0/0
 ip helper-address 7.13.3.1
ip route 7.13.3.0 255.255.255.0 Serial0/1/0
```
**Why this route is exit-interface style:** the PDF specifically requires FRC's static routes to name the outgoing interface, not a next-hop IP -- this is the first instance of that pattern (more added in Phase 10).

**Second dependency discovered:** COI and FRC's *first* DHCP attempts also failed -- this time because the *reply* had nowhere to go. TRP had no route back to COI's LAN, and RLH had no route back to FRC's LAN. **Fix:**
```
! On TRP:
ip route 7.13.7.0 255.255.255.192 Serial0/1/0
! On RLH:
ip route 7.13.6.0 255.255.255.0 Serial0/1/1
```
This established the general lesson carried through the rest of the project: **DHCP relay (and later, DNS/email/web access) requires a working route in *both* directions** -- toward the server, and back from the server to the client's subnet.

### Worked example — tracing a DHCP request from PC1-BMD to NDCC and back
1. PC1-BMD boots up with no IP and broadcasts a DHCP request onto its local segment (BMD's LAN).
2. BMD's router (`GigabitEthernet0/0`) sees this broadcast. Because of `ip helper-address 7.13.0.1`, it doesn't drop the broadcast — it repackages it as a normal unicast packet addressed to `7.13.0.1` (NDCC).
3. That unicast packet needs an actual route from BMD to `7.13.0.0/23`. Before RIPv2 was configured, BMD had no such route, so the packet was silently dropped — this is exactly why the first attempt returned an APIPA address (`169.254.x.x`).
4. After RIPv2 was added to BMD/CGB/NDCC, BMD learned a route to `7.13.0.0/23` via RIP, and the relayed packet successfully reached NDCC.
5. NDCC's DHCP service checks which pool matches the *original* subnet the request came from (BMD-POOL, since the relay preserved that information) and offers an address like `7.13.5.10`.
6. The offer has to travel **back** to BMD's LAN — this only works because BMD also has a RIP-learned route back to its own subnet (trivially true, it's directly connected) and NDCC has a route back to BMD's subnet (also via RIP, since BMD is in the Warning Loop).
7. PC1-BMD receives the offer and configures itself — confirmed via `ipconfig /renew` returning `7.13.5.10`.

---

## Phase 5 — DNS Configuration (Shared — all members)

No CLI -- configured via DNS-NDCC's GUI (Services -> DNS). DNS service turned on, with these A records:

| Name | IP |
|---|---|
| www.ndcc.relief.gov | 7.13.0.4 |
| www.rlh.relief.gov | 7.13.3.4 |
| mail.ndcc.gov | 7.13.0.3 |
| mail.cgb.gov | 7.13.2.3 |
| mail.rlh.gov | 7.13.3.3 |
| mail.trp.gov | 7.13.4.3 |
| mail.bmd.gov | 7.13.5.3 |
| mail.frc.gov | 7.13.6.3 |
| mail.coi.gov | 7.13.7.3 |
| ndcc.gov | 7.13.0.3 |
| rlh.gov | 7.13.3.3 |

**Why the last two (domain-only) records exist:** Packet Tracer's SMTP implementation resolves the bare domain name (`rlh.gov`), not the `mail.rlh.gov` hostname, when one mail server relays to another. Without these, cross-domain email silently fails even though everything else is correct.

Every static device (Phase 3) and DHCP pool (Phase 4) already pointed to `7.13.0.5` as its DNS server, so no further per-device DNS changes were needed -- this requirement was satisfied structurally by earlier phases.

**Verified:** `ping www.ndcc.relief.gov` from PC1-COI (the farthest device in the network) resolved correctly and got replies -- proving DNS + routing work together end-to-end.

### Worked example — resolving `www.ndcc.relief.gov` from PC1-COI
1. PC1-COI's IP configuration has DNS server `7.13.0.5` set (inherited from TRP's DHCP pool, which specifies `dns-server 7.13.0.5`).
2. When the ping command is typed, PC1-COI first sends a DNS query to `7.13.0.5` asking "what's the address for `www.ndcc.relief.gov`?" — this query itself has to travel COI -> TRP -> (Central Switch) -> NDCC, four network segments, using the routes built in Phases 8-11.
3. DNS-NDCC looks up its A record table, finds `www.ndcc.relief.gov -> 7.13.0.4`, and replies.
4. The reply travels back along the same path (NDCC -> TRP -> COI), and PC1-COI now knows the target IP.
5. *Only now* does the actual ICMP ping begin, sent to `7.13.0.4` — a completely separate exchange from the DNS lookup, following the same multi-hop path.
6. This is why the very first packet in this kind of test often times out (as seen in our actual results) — the DNS lookup itself takes a moment on the first attempt, eating into the ping's timeout window, even though everything is configured correctly.

---

## Phase 6 — Email Configuration (Member 1 — Email)

No CLI -- configured via each Mail server's GUI (Services -> EMAIL) and each PC's GUI (Desktop -> Email).

### NDCC and RLH (cross-domain requirement)
- Mail-NDCC: SMTP+POP3 on, domain `ndcc.gov`, users `officer1`/`officer2` (password `cisco123`)
- Mail-RLH: SMTP+POP3 on, domain `rlh.gov`, users `logistics1`/`logistics2` (password `cisco123`)
- PC clients configured with matching address/server/username/password.

**Issues hit and fixed:**
1. First send attempt gave "SMTP authentication failure" -- root cause was a **stray extra character** in the saved password field on the PC client (9 characters saved instead of 8), found by counting password dots in a screenshot against the known password length. Fixed by clearing and retyping.
2. Verified **both directions**: officer1@ndcc.gov -> logistics1@rlh.gov, and logistics1@rlh.gov -> officer1@ndcc.gov -- both succeeded, satisfying the PDF's two-way requirement.

### CGB, TRP, BMD, FRC, COI (local-only)
Each got: SMTP+POP3 on, its own domain (`cgb.gov`, `trp.gov`, etc.), two generic users (`user1`/`user2`, password `cisco123`), and one PC client configured to send a self-test email.

**Issue hit and fixed (CGB):** first attempt gave "Server not found," despite DNS resolving correctly and the IP/services all being confirmed on. Root cause: the server's **Domain Name field had not actually been committed** -- in Packet Tracer, typing a value into that field isn't enough; the **Set** button must be clicked. Retyping and clicking Set fixed it. This same pitfall was proactively avoided on TRP, BMD, FRC, and COI.

### Worked example — how officer1@ndcc.gov's email actually reached logistics1@rlh.gov
1. PC1-NDCC's email client is configured with outgoing server `mail.ndcc.gov`. When "Send" is clicked, the PC first resolves `mail.ndcc.gov` via DNS (7.13.0.5) to `7.13.0.3` (Mail-NDCC), then connects to that server using SMTP and authenticates as `officer1` with the saved password.
2. Mail-NDCC sees the destination address is `logistics1@rlh.gov` — a *different* domain from its own (`ndcc.gov`) — so it needs to relay this to RLH's mail server rather than deliver it locally.
3. To find RLH's mail server, Mail-NDCC (acting as an SMTP relay) looks up the bare domain `rlh.gov` in DNS — **not** `mail.rlh.gov`. This is the Packet-Tracer-specific quirk that made the extra `rlh.gov -> 7.13.3.3` DNS record necessary; without it, this exact step fails even though the client-side lookup of `mail.ndcc.gov` worked fine.
4. Once resolved, Mail-NDCC connects to `7.13.3.3` (Mail-RLH) over SMTP and hands off the message.
5. Mail-RLH stores it in `logistics1`'s mailbox.
6. When PC1-RLH's email client clicks "Receive," it connects to `mail.rlh.gov` over POP3, authenticates as `logistics1`, and downloads the waiting message.

---

## Phase 7 — Web Servers (Shared — all members)

No CLI -- Services -> HTTP turned on for Web-NDCC and Web-RLH, each with a simple custom `index.html`.

**Verified:** from PC1-COI (farthest device), both `www.ndcc.relief.gov` and `www.rlh.relief.gov` loaded correctly -- confirming DNS, full routing, and both web servers all work together across the entire topology.

### Worked example — what happens when PC1-COI opens `www.rlh.relief.gov`
1. Same DNS lookup pattern as Phase 5: PC1-COI queries `7.13.0.5`, gets back `7.13.3.4` (Web-RLH's IP), taking a round trip through NDCC's DNS server.
2. PC1-COI's browser opens a TCP connection to `7.13.3.4` on port 80 — this connection has to be individually routed COI -> TRP -> (Central Switch) -> RLH, using the exact static routes configured on COI, TRP, and RLH in Phases 4 and 10.
3. Once connected, the browser sends an HTTP GET request; Web-RLH's HTTP service responds with the `index.html` content configured in this phase.
4. The page renders in the browser — proving three separate systems worked together correctly: DNS (name resolution), routing (multi-hop delivery), and the HTTP service itself (actually serving content).

---

## Phase 8/9 — Dynamic Routing & Redistribution (CLI commands) (Member 3 — Dynamic Routing)

### RIPv2 on the Warning Loop (NDCC, BMD, CGB) — identical on all three:
```
router rip
 version 2
 no auto-summary
 network 7.0.0.0
```
**Why `no auto-summary`:** critical, since the project's subnets are all different sizes (VLSM). Without this, RIP tries to summarize everything back into one classful block and breaks the subnetting entirely. **Why `network 7.0.0.0`:** activates RIP on any of that router's interfaces whose IP starts with `7.` -- covers every relevant interface without listing each individually.

*(This was actually configured early, during Phase 4, to unblock DHCP relay -- see the note there.)*

### NDCC — static routes to the rest of the topology + redistribution
```
ip route 7.13.3.0 255.255.255.0 Serial0/1/0
ip route 7.13.4.0 255.255.255.128 7.13.7.68
ip route 7.13.6.0 255.255.255.0 7.13.7.74
ip route 7.13.7.0 255.255.255.192 7.13.7.68
router rip
 redistribute static metric 1
```
**Why:** NDCC needs to know about RLH, TRP, FRC, and COI's networks (some via direct link, some via next-hop through another router). `redistribute static metric 1` then injects these static routes into RIP, so BMD and CGB **automatically** learn about all four networks -- satisfying the PDF's explicit redistribution requirement -- without a single static route being configured on BMD or CGB directly.

**Verified:** `show ip route` on BMD and CGB showed RIP-learned (`R`) routes to every subnet in the topology.

### Worked example — how CGB learned about FRC's network without any config on CGB
1. On NDCC, the static route `ip route 7.13.6.0 255.255.255.0 7.13.7.74` tells NDCC's own routing table "to reach FRC's LAN, send it to `7.13.7.74`" (RLH's serial IP).
2. Separately, `redistribute static metric 1` under `router rip` on NDCC tells RIP: "take every route in my table that came from a static entry, and announce it to my RIP neighbors as if it were a RIP route, starting at hop-count 1."
3. NDCC's RIP process sends this update over the Warning Loop links — CGB (a RIP neighbor) receives an advertisement saying "network `7.13.6.0/24` is reachable via me, NDCC, at metric 1."
4. CGB, having no better information, installs this into its own routing table as an `R` (RIP) route, automatically incrementing the metric it would advertise further (if it had other RIP neighbors needing it).
5. From this point on, if a device on CGB's LAN needs to reach FRC, CGB's router looks up `7.13.6.0/24`, finds the RIP-learned route pointing back toward NDCC, and forwards the packet there — with zero manual configuration on CGB itself.

---

## Phase 10 — Static Routing: RLH, TRP, FRC (CLI commands) (Member 2 — Static Routing)

### RLH — static routes to every external network
```
ip route 7.13.0.0 255.255.254.0 Serial0/1/0      (NDCC - direct link, exit-interface)
ip route 7.13.2.0 255.255.255.0 7.13.7.66         (CGB - via Central Switch)
ip route 7.13.4.0 255.255.255.128 7.13.7.68       (TRP - via Central Switch)
ip route 7.13.5.0 255.255.255.0 7.13.7.65         (BMD - via NDCC)
ip route 7.13.7.0 255.255.255.192 7.13.7.68       (COI - via TRP)
```
(FRC's route was already added in Phase 4.) **Why NDCC's route is deliberately exit-interface style:** this sets up the contrast needed for the recursive backup route in Phase 11 -- the backup needs to be the "recursive" one (next-hop IP), so the primary needed to be the non-recursive one.

### TRP — static routes to every external network
```
ip route 7.13.0.0 255.255.254.0 7.13.7.65   (NDCC - via Central Switch)
ip route 7.13.2.0 255.255.255.0 7.13.7.66   (CGB - via Central Switch)
ip route 7.13.3.0 255.255.255.0 7.13.7.67   (RLH - via Central Switch)
ip route 7.13.5.0 255.255.255.0 7.13.7.65   (BMD - via NDCC)
ip route 7.13.6.0 255.255.255.0 7.13.7.67   (FRC - via RLH)
```
(COI's route was already added in Phase 4.)

**Issue found:** first end-to-end ping test (PC1-RLH -> COI's router) failed with 100% loss even though RLH's route table looked correct -- because **TRP had no route back to RLH's LAN yet**. Adding `ip route 7.13.3.0 255.255.255.0 7.13.7.67` on TRP fixed it -- same "missing return path" pattern as Phase 4.

### FRC — exit-interface static routes to every external network
```
ip route 7.13.0.0 255.255.254.0 Serial0/1/0
ip route 7.13.2.0 255.255.255.0 Serial0/1/0
ip route 7.13.4.0 255.255.255.128 Serial0/1/0
ip route 7.13.5.0 255.255.255.0 Serial0/1/0
ip route 7.13.7.0 255.255.255.192 Serial0/1/0
```
(RLH's route was already added in Phase 4.) **Why exit-interface for all of these:** explicit PDF requirement -- since FRC has only one physical way out anyway, naming the interface directly is used throughout rather than a next-hop IP.

**Verified:** a full 3-hop ping (PC1-FRC -> BMD's router, via RLH->NDCC->BMD) succeeded, confirming the whole mesh works.

### Worked example — tracing the FRC -> COI ping (the longest path in the topology)
1. PC1-FRC pings `7.13.7.1` (COI's router). FRC's router checks its routing table and finds `ip route 7.13.7.0 255.255.255.192 Serial0/1/0` — an exit-interface route, so it sends the packet straight out its only serial link, to RLH.
2. RLH receives it, checks its own table, and finds `ip route 7.13.7.0 255.255.255.192 7.13.7.68` — a next-hop route pointing to TRP's Central Switch IP. RLH forwards the packet out its `GigabitEthernet0/1` (the Central Switch interface).
3. TRP receives it on its own Central Switch interface. Since `7.13.7.0/26` is TRP's *own* directly-connected COI-facing route (`ip route 7.13.7.0 255.255.255.192 Serial0/1/0`), TRP forwards the packet out its serial link to COI.
4. COI receives the ping request on its `GigabitEthernet0/0` (LAN side)... actually on `Serial0/1/0` from TRP, and its router responds directly, since `7.13.7.1` is COI's own interface.
5. The reply retraces the exact same path in reverse: COI -> TRP -> RLH -> FRC — proving every router along a 4-hop chain (FRC, RLH, TRP, COI) had a correctly configured route both forward and backward. The observed `TTL=252` in the real test (starting from a default of 255, minus roughly 3 decrements) is consistent with a multi-hop path.

---

## Phase 11 — Redundancy (CLI commands) (Member 2 — Static Routing)

### RLH's recursive backup route to NDCC (via TRP)
```
ip route 7.13.0.0 255.255.254.0 7.13.7.68 5
```
**Why "recursive":** in Cisco terminology, any static route pointing to a **next-hop IP** (rather than an exit interface) is called "recursive," because the router has to do an extra routing-table lookup to figure out how to actually reach that IP. This contrasts with RLH's *primary* route to NDCC, which uses the exit-interface style. **Why AD 5:** a backup route needs a *higher* administrative distance than the primary's default of 1, so it only activates when the primary is unavailable -- 5 was chosen since the PDF didn't specify an exact value for this particular route (unlike TRP's floating route below).

### TRP's floating static backup route to NDCC (via CGB)
```
ip route 7.13.0.0 255.255.254.0 7.13.7.66 57
```
**AD calculation (per the PDF's formula):** sum of *all* digits of Member 1's ID (24241132) = 2+4+2+4+1+1+3+2 = **19**. x 3 group members = **AD 57**.

### Failure discovered & fixed during RLH's failover test
Shutting down NDCC's `Serial0/1/0` correctly activated RLH's backup route -- but the return ping still failed 100%, because **NDCC's only route back to RLH's LAN was tied to that same now-shutdown interface**, and Cisco automatically removed it. The PDF only asked for a backup on RLH's side, but true bidirectional failover needs one on NDCC's side too. **Fix (an additive assumption within what the PDF permits):**
```
! On NDCC:
ip route 7.13.3.0 255.255.255.0 7.13.7.67 5
```
After this, the failover test succeeded in both directions, and reverted cleanly to primary once the link was restored.

### TRP's floating route test
Since TRP's primary and backup routes both travel over its *same* physical interface (its only link to the network core), the test was done by directly removing the primary route (`no ip route 7.13.0.0 255.255.254.0 7.13.7.65`) to simulate its failure, confirming the AD-57 backup took over immediately -- then restoring it and confirming automatic reversion.

### Worked example — how Administrative Distance decided which route "wins"
RLH had *two* entries for the same destination (`7.13.0.0/23`) sitting in its configuration at once:
```
ip route 7.13.0.0 255.255.254.0 Serial0/1/0        <- AD defaults to 1 (not specified)
ip route 7.13.0.0 255.255.254.0 7.13.7.68 5         <- AD explicitly set to 5
```
Cisco IOS's rule is simple: **lower AD wins**. Think of AD as "how much do I trust this source of information" — a smaller number means more trusted. Since 1 < 5, IOS always installs the `Serial0/1/0` route into the active routing table (the one you see in `show ip route`) and keeps the AD-5 route in reserve, invisible to `show ip route` output, but not deleted.

The moment `Serial0/1/0` actually goes down, IOS removes that specific route from the table (because its next-hop is no longer reachable), which leaves the AD-5 route as the *only* remaining candidate — so it gets installed and becomes visible, with zero manual action needed. When `Serial0/1/0` comes back up, IOS re-adds the AD-1 route, and because 1 < 5 again, it silently displaces the AD-5 route back out of the active table. This automatic "best route wins, but keep the rest as reserves" behavior is the entire mechanism behind both backup routes in this project.

---

## Phase 12 — Testing & Verification (Shared — all members)

- **Extended ping matrix:** 6 test pairs chosen to touch every unit at least twice, including the single longest path in the topology (FRC -> COI, 4 router hops) -- all succeeded (some with a single first-packet timeout, a normal ARP/DNS resolution delay on first contact between two devices, not a fault).
- **RLH recursive backup failover:** demonstrated by shutting down NDCC's `Serial0/1/0`, confirming the backup route activated (`show ip route`), confirming traffic was restored (ping), then restoring the link and confirming automatic reversion to primary.
- **TRP floating static failover:** demonstrated by removing TRP's primary route, confirming the floating route activated, confirming traffic flowed, then restoring the primary and confirming reversion.
- **Cross-domain email (both directions)** and **both websites reachable from the farthest unit (COI)** were already verified in Phases 6-7 and directly satisfy this phase's remaining requirements.

### Worked example — reading a real ping result correctly
One actual test result from this phase: pinging FRC's router from PC1-CGB returned:
```
Request timed out.
Reply from 7.13.6.1: bytes=32 time=8ms TTL=253
Reply from 7.13.6.1: bytes=32 time=3ms TTL=253
Reply from 7.13.6.1: bytes=32 time=20ms TTL=253
Ping statistics: Sent = 4, Received = 3, Lost = 1 (25% loss)
```
How to interpret this correctly: the **first** packet timing out, followed by 3 clean replies, is the signature of a normal first-contact delay (ARP resolution along the path), not a routing fault — if there were a real routing problem, **all 4** packets would fail, not just the first. The `TTL=253` value (starting from a default of 255 or 256 depending on OS, minus roughly 2-3 hops) is consistent with the actual path length (CGB -> Central Switch -> RLH -> FRC, or similar), giving extra confidence the packet actually took a sensible route rather than looping or failing silently. This distinction — "first packet fails, rest succeed" vs. "all packets fail" — was used throughout this phase (and earlier ones) to separate real problems from expected first-contact behavior.

---

## Phase 13 — Deliverables Packaging (Shared — all members)

- `show running-config` captured from all 7 routers and cross-checked against this entire build history -- zero discrepancies found.
- Combined into `CoastNet_Final_Documentation.docx`: group info + base network calc, full VLSM tree, complete IP address table, and all 7 routers' verbatim configurations.
- Remaining: topology screenshot (with network-address notes placed in Packet Tracer) and a work-distribution write-up.

### Worked example — spot-checking one router's real config against the plan
Taking RLH's actual captured `show running-config` output and checking it line by line against what Phases 3, 4, 10, and 11 planned:
- `ip address 7.13.3.1 255.255.255.0` on `GigabitEthernet0/0` — matches the Phase 1 IP table exactly.
- `ip dhcp pool RLH-POOL` with `network 7.13.3.0 255.255.255.0` — matches Phase 4's local DHCP setup.
- `ip route 7.13.0.0 255.255.254.0 Serial0/1/0` — matches Phase 10's exit-interface primary route to NDCC.
- `ip route 7.13.0.0 255.255.254.0 7.13.7.68 5` — matches Phase 11's recursive backup, including the AD value.

Every one of these lines was independently predicted *before* the config was ever pulled, based purely on the addressing plan and routing design — the fact that the actual captured config matched exactly, with no surprises, is what gave confidence to call the network "verified correct" rather than just "verified working right now." This kind of cross-check (plan vs. actual running-config) is good practice any time you're documenting a network for a grade or a handover.

---

## Summary of every "gotcha" encountered (useful for the viva / defense)

1. **DHCP relay requires routing to already exist** -- the relayed packet needs an actual route, both toward the DHCP server and back from it. This dependency surfaced repeatedly and shaped the decision to complete full Routing (Phases 8-11) before DNS/Email/Web.
2. **Static routes tied to a specific interface get removed if that interface goes down** -- this broke RLH's failover test until a backup route was added on NDCC's side too, a gap the PDF's wording didn't explicitly call out but that real bidirectional redundancy requires.
3. **Packet Tracer's "Set" button matters** -- typing a value (like a mail server's domain name) isn't enough; it must be explicitly applied, or the server silently doesn't use it.
4. **Password fields can silently save extra characters** -- always worth retyping cleanly (Ctrl+A, delete, retype) if authentication fails for no visible reason.
5. **First-packet ping timeouts are normal**, not a fault -- caused by ARP/DNS resolution delay on first contact between two devices; a retried ping (or the 2nd-4th packet in the same test) succeeds.

---

## Work Distribution (per PDF requirement: each member configures at least 1 of DHCP/Email/Static Routing/Dynamic Routing)

| Member | Required Category | Phase(s) |
|---|---|---|
| Member 1 | DHCP + Email | Phase 4 (DHCP) + Phase 6 (Email) |
| Member 2 | Static Routing | Phase 10 + Phase 11 (+ NDCC's static routes, part of Phase 4/8) |
| Member 3 | Dynamic Routing | Phase 8/9 (RIPv2 + redistribution) |

Remaining phases (1, 2, 3, 5, 7, 12, 13) are shared/divided freely among all three members.

**A simplified, plain-language viva-prep explanation of each member's section (with analogies and anticipated questions) is in `viva_prep.md`.**
