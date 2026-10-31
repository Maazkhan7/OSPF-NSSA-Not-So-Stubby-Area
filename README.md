<div align="center">

# 🚧 OSPF NSSA — Not-So-Stubby Area

### Type-7 LSAs · ABR Translation to Type-5 · N2 / E2 Routes · Default Route into the Area

![Cisco](https://img.shields.io/badge/Cisco-Packet%20Tracer-1BA0D7?style=for-the-badge&logo=cisco&logoColor=white)
![Protocol](https://img.shields.io/badge/Protocol-OSPF-blue?style=for-the-badge)
![Area](https://img.shields.io/badge/Area%201-NSSA-orange?style=for-the-badge)
![LSA](https://img.shields.io/badge/LSA-Type%207%20→%20Type%205-purple?style=for-the-badge)
![Routers](https://img.shields.io/badge/Routers-3%20×%201941-yellow?style=for-the-badge)
![Level](https://img.shields.io/badge/Level-CCNA%20%2F%20CCNP-red?style=for-the-badge)
![Status](https://img.shields.io/badge/Status-Completed-success?style=for-the-badge)

**An area that blocks external LSAs… yet still allows its own ASBR**

</div>

---

## 📖 Table of Contents

[Overview](#-project-overview) · [Lab Snapshot](#-lab-snapshot) · [LSA Journey](#-the-lsa-journey) · [Topology](#️-network-topology) · [Configuration](#️-configuration) · [Verification](#-verification--stage-by-stage) · [Area Types](#-ospf-area-types-compared) · [NSSA vs Stub](#-nssa-vs-stub) · [Metrics](#-route-codes--metrics) · [Troubleshooting](#-troubleshooting-guide) · [Advanced](#-advanced-topics) · [Takeaways](#-key-takeaways) · [Challenges](#-challenges)

---

## 📌 Project Overview

A **stub area** shrinks routing tables by blocking external (Type-5) LSAs, but it has one big limitation: **it cannot contain an ASBR**. So what if a branch inside that area needs to connect an external source — for example static routes or another routing protocol?

That is exactly why **NSSA (Not-So-Stubby Area)** exists. An NSSA:

- 🚫 **Blocks Type-5 LSAs** from entering the area
- ✅ **Allows an ASBR inside the area**, which advertises its external routes as **Type-7 LSAs**
- 🔄 Lets the **ABR translate Type-7 → Type-5** so the rest of the network sees those routes

This lab builds that exact scenario with three Cisco 1941 routers:

| Router | OSPF Role | External routes it injects |
|:------:|:---------:|:---------------------------|
| **Router0** (RID 10.1.1.1) | **ASBR inside Area 1 (NSSA)** | `100.100.50.0` · `100.100.60.0` · `100.100.70.0` → **Type-7** |
| **Router1** (RID 20.1.1.1) | **ABR** between Area 1 and Area 0 | *(translates Type-7 → Type-5)* |
| **Router2** (RID 30.1.1.1) | **ASBR in the backbone** | `100.100.100.0` · `100.100.150.0` · `100.100.200.0` → **Type-5** |

> 🎯 **Result:** Router2 sees Router0's networks as `O E2` (translated by the ABR), Router1 sees both `O N2` and `O E2`, and Router0 — inside the NSSA — receives just a **default route** (`O*IA 0.0.0.0/0`).

---

## 📸 Lab Snapshot

| Item | Value |
|:-----|:------|
| 🧰 Tool | Cisco Packet Tracer |
| 🖧 Routers | 3 × Cisco **1941** (Router0, Router1, Router2) |
| 🔀 Switches / PCs | 2 × 2950-24 / 6 × PC-PT |
| 🧭 Protocol | **OSPF** (process 1) |
| 📍 Areas | **Area 1 = NSSA** (Router0 side) · **Area 0** (backbone, Router2 side) |
| 🏷️ Router IDs | 10.1.1.1 · 20.1.1.1 · 30.1.1.1 |
| 📦 External sources | 3 static routes on Router0 · 3 static routes on Router2 (all → `Null0`) |
| 🔑 LSAs verified | **Type-7** (Area 1) · **Type-5** (translated + original) |
| 🧪 Route codes seen | `O N2` · `O E2` · `O IA` · `O*IA` default |

---

## 🛰️ The LSA Journey

```
 ──────────────────────  Router0's networks  (inside the NSSA)  ──────────────────────
 Router0 (ASBR)                 Router1 (ABR)                          Router2
 ┌──────────────┐  Type-7 LSA   ┌──────────────┐  Type-5 LSA  ┌──────────────┐
 │ static routes│ ────────────► │ translates   │ ───────────► │ O E2 routes  │
 │ 100.100.50/  │   (Area 1)    │ 7  ➜  5      │   (Area 0)   │ 100.100.50/  │
 │ 60/70.0      │               │              │              │ 60/70.0      │
 └──────────────┘               └──────────────┘              └──────────────┘

 ──────────────────────  Router2's networks  (backbone side)  ───────────────────────
 Router0 (NSSA)                  Router1 (ABR)                         Router2 (ASBR)
 ┌──────────────┐  ✖ Type-5     ┌──────────────┐  Type-5 LSA  ┌──────────────┐
 │ default route│ ◄─ blocked ── │ BLOCKS Type-5│ ◄─────────── │ static routes│
 │ O*IA 0.0.0.0 │   Type-3      │ sends default│              │ 100.100.100/ │
 │ via 1.1.1.2  │   default     │ into Area 1  │              │ 150/200.0    │
 └──────────────┘               └──────────────┘              └──────────────┘
```

---

## 🎯 Objectives

- ✅ Configure an OSPF **NSSA** with an **ASBR inside it**
- ✅ See **Type-7 LSAs** in the NSSA database
- ✅ See the **ABR translate Type-7 into Type-5**
- ✅ Recognize **`O N2`** and **`O E2`** routes in the routing table
- ✅ Prove that the NSSA **blocks external routes** and receives a **default route**
- ✅ Understand the difference between **Stub** and **NSSA**
- ✅ Read the OSPF **database** by LSA type and advertising router

---

## 🗺️ Network Topology

```
          Area 1  —  NSSA  🟧                                     Area 0  —  Backbone  🟪
 ┌──────────────────────────────────────────────┬─────────────────────────────────────────────┐
 │   L1 10.1.1.1/32          L2 20.1.1.1/32     │                          L3 30.1.1.1/32     │
 │        │                       │             │                               │             │
 │  ┌──────────┐ g0/0       g0/0 ┌──────────┐ g0/1                      g0/1 ┌──────────┐    │
 │  │ Router0  ├─────────────────┤ Router1  ├──────────────────────────────────┤ Router2  │    │
 │  │  ASBR    │  .1  1.1.1.0/24 .2│  ABR    │.1      2.1.1.0/24            .2│  ASBR    │    │
 │  └────┬─────┘                 └──────────┘                                  └────┬─────┘    │
 │       │ g0/1 192.168.1.1                                         g0/0 192.168.2.1│          │
 │   Switch0                                                                    Switch2        │
 │  PC1 PC0 PC2   192.168.1.0/24                                      PC6 PC7 PC8  192.168.2.0/24
 │                                              │                                              │
 │ ip route 100.100.50.0 255.255.255.0 null0    │  ip route 100.100.100.0 255.255.255.0 null0  │
 │ ip route 100.100.60.0 255.255.255.0 null0    │  ip route 100.100.150.0 255.255.255.0 null0  │
 │ ip route 100.100.70.0 255.255.255.0 null0    │  ip route 100.100.200.0 255.255.255.0 null0  │
 └──────────────────────────────────────────────┴─────────────────────────────────────────────┘
        LSA TYPE-7 (inside NSSA)                          LSA TYPE-5 (rest of the network)
```

> 📷 Add your screenshot: `![Topology](images/topology.png)`

> 📝 **Naming note:** the Packet Tracer icons are *Router0 / Router1 / Router2*; the CLI prompts in the screenshots show **R1 / R2 / R3**.

---

## 📋 IP Addressing Table

| Device | Interface | IP Address | OSPF Area |
|:------:|:---------:|:----------:|:---------:|
| **Router0** | Gi0/0 | 1.1.1.1/24 | Area 1 |
| **Router0** | Gi0/1 | 192.168.1.1/24 (LAN 1) | Area 1 |
| **Router0** | Loopback1 | 10.1.1.1/32 | Area 1 |
| **Router1** | Gi0/0 | 1.1.1.2/24 | Area 1 |
| **Router1** | Loopback1 | 20.1.1.1/32 | Area 1 |
| **Router1** | Gi0/1 | 2.1.1.1/24 | **Area 0** |
| **Router2** | Gi0/1 | 2.1.1.2/24 | Area 0 |
| **Router2** | Gi0/0 | 192.168.2.1/24 (LAN 2) | Area 0 |
| **Router2** | Loopback1 | 30.1.1.1/32 | Area 0 |

---

## ⚙️ Configuration

> ⚠️ The OSPF configuration itself is not visible in the screenshots. The commands below are the standard configuration that **produces exactly these outputs** — compare them with your own running-config.

### 🔹 Router0 — ASBR inside the NSSA

```bash
ip route 100.100.50.0 255.255.255.0 null0
ip route 100.100.60.0 255.255.255.0 null0
ip route 100.100.70.0 255.255.255.0 null0

router ospf 1
 router-id 10.1.1.1
 area 1 nssa
 network 1.1.1.0 0.0.0.255 area 1
 network 192.168.1.0 0.0.0.255 area 1
 network 10.1.1.1 0.0.0.0 area 1
 redistribute static subnets
```

### 🔹 Router1 — ABR

```bash
router ospf 1
 router-id 20.1.1.1
 area 1 nssa no-summary           ◄── gives Area 1 only a default route (see Stage 4)
 network 1.1.1.0 0.0.0.255 area 1
 network 20.1.1.1 0.0.0.0 area 1
 network 2.1.1.0 0.0.0.255 area 0
```

### 🔹 Router2 — ASBR in the backbone

```bash
ip route 100.100.100.0 255.255.255.0 null0
ip route 100.100.150.0 255.255.255.0 null0
ip route 100.100.200.0 255.255.255.0 null0

router ospf 1
 router-id 30.1.1.1
 network 2.1.1.0 0.0.0.255 area 0
 network 192.168.2.0 0.0.0.255 area 0
 network 30.1.1.1 0.0.0.0 area 0
 redistribute static subnets
```

> 🔑 `area 1 nssa` must be configured on **every router in Area 1** (Router0 and Router1). `no-summary` is added only on the **ABR**.

---

## ✅ Verification — Stage by Stage

### 🟠 Stage 1: Type-7 LSAs Inside the NSSA (Router0's Database)

```
R1# show ip ospf database
            OSPF Router with ID (10.1.1.1) (Process ID 1)

                Router Link States (Area 1)
Link ID         ADV Router      Age     Seq#         Checksum Link count
20.1.1.1        20.1.1.1        1078    0x80000006   0x00ab43 2
10.1.1.1        10.1.1.1        985     0x80000007   0x002f61 3

                Net Link States (Area 1)
1.1.1.2         20.1.1.1        1471    0x80000001   0x0084b7

                Summary Net Link States (Area 1)
0.0.0.0         10.1.1.1        72      0x80000001   0x00242d       ◄── default route LSAs (Type-3)
0.0.0.0         20.1.1.1        51      0x80000005   0x00c181

                Type-7 AS External Link States (Area 1)               ◄── 🔥 NSSA external LSAs
Link ID         ADV Router      Age     Seq#         Checksum Tag
100.100.50.0    10.1.1.1        72      0x80000004   0x00415b 0
100.100.60.0    10.1.1.1        72      0x80000005   0x00d0c0 0
100.100.70.0    10.1.1.1        72      0x80000006   0x006026 0

                Type-5 AS External Link States
100.100.100.0   30.1.1.1        1192    0x80000001   0x0087ea 0
100.100.150.0   30.1.1.1        1192    0x80000001   0x005fe0 0
100.100.200.0   30.1.1.1        1192    0x80000001   0x0037d6 0
```

🧠 **What this shows**

- Router0 generates **Type-7 LSAs** for its three static networks, flooded **inside Area 1 only**
- The advertising router is **10.1.1.1** — Router0 itself (the ASBR in the NSSA)
- *Simulator note:* Packet Tracer still lists the three Type-5 LSAs from 30.1.1.1 in Router0's database, but Router0's routing table (Stage 4) has **no routes for them** — on real IOS, Type-5 LSAs aren't flooded into an NSSA at all

---

### 🔵 Stage 2: The ABR Translates Type-7 → Type-5 (Router1's Database)

```
R2# show ip ospf database
                Summary Net Link States (Area 1)
0.0.0.0         10.1.1.1        166     0x80000001   0x00242d
0.0.0.0         20.1.1.1        145     0x80000005   0x00c181

                Type-7 AS External Link States (Area 1)
100.100.50.0    10.1.1.1        166     0x80000004   0x00415b 0     ◄── from Router0
100.100.60.0    10.1.1.1        166     0x80000005   0x00d0c0 0
100.100.70.0    10.1.1.1        166     0x80000006   0x006026 0

                Type-5 AS External Link States
100.100.100.0   30.1.1.1        1286    0x80000001   0x0087ea 0     ◄── Router2's own externals
100.100.150.0   30.1.1.1        1286    0x80000001   0x005fe0 0
100.100.200.0   30.1.1.1        1286    0x80000001   0x0037d6 0
100.100.50.0    20.1.1.1        1079    0x80000001   0x009f02 0     ◄── 🔥 TRANSLATED by Router1 (ADV = 20.1.1.1)
100.100.60.0    20.1.1.1        1079    0x80000002   0x002f67 0
100.100.70.0    20.1.1.1        1079    0x80000003   0x00becc 0
```

🧠 **What this shows:** for each Type-7 LSA, Router1 (**ABR**) originates a matching **Type-5 LSA with itself (20.1.1.1) as the advertising router** — that is how Router0's networks reach the backbone.

---

### 🟣 Stage 3: Routing Tables of the ABR and the Backbone Router

**Router1 (ABR) — sees both the N2 and E2 routes**

```
R2# show ip route
C       1.1.1.0/24 is directly connected, GigabitEthernet0/0
C       2.1.1.0/24 is directly connected, GigabitEthernet0/1
O       10.1.1.1/32 [110/2] via 1.1.1.1, 00:21:40, GigabitEthernet0/0
C       20.1.1.1/32 is directly connected, Loopback1
O       30.1.1.1/32 [110/2] via 2.1.1.2, 00:25:23, GigabitEthernet0/1
O N2    100.100.50.0/24 [110/20] via 1.1.1.1, 00:20:07, GigabitEthernet0/0   ◄── 🟧 from the NSSA (Type-7)
O N2    100.100.60.0/24 [110/20] via 1.1.1.1, 00:20:07, GigabitEthernet0/0
O N2    100.100.70.0/24 [110/20] via 1.1.1.1, 00:20:07, GigabitEthernet0/0
O E2    100.100.100.0/24 [110/20] via 2.1.1.2, 00:23:34, GigabitEthernet0/1  ◄── 🟪 from the backbone (Type-5)
O E2    100.100.150.0/24 [110/20] via 2.1.1.2, 00:23:34, GigabitEthernet0/1
O E2    100.100.200.0/24 [110/20] via 2.1.1.2, 00:23:34, GigabitEthernet0/1
O       192.168.1.0/24 [110/2] via 1.1.1.1, 00:21:40, GigabitEthernet0/0
O       192.168.2.0/24 [110/2] via 2.1.1.2, 00:25:23, GigabitEthernet0/1
```

**Router2 (backbone ASBR) — sees Router0's networks as E2**

```
R3# show ip route
O IA    1.1.1.0/24 [110/2] via 2.1.1.1, 00:24:55, GigabitEthernet0/1       ◄── inter-area (Area 1)
O IA    10.1.1.1/32 [110/3] via 2.1.1.1, 00:21:07, GigabitEthernet0/1
O IA    20.1.1.1/32 [110/2] via 2.1.1.1, 00:24:55, GigabitEthernet0/1
O E2    100.100.50.0/24 [110/20] via 2.1.1.1, 00:19:39, GigabitEthernet0/1 ◄── 🔥 Router0's networks, translated
O E2    100.100.60.0/24 [110/20] via 2.1.1.1, 00:19:39, GigabitEthernet0/1
O E2    100.100.70.0/24 [110/20] via 2.1.1.1, 00:19:39, GigabitEthernet0/1
S       100.100.100.0/24 is directly connected, Null0                       ◄── its own static routes
S       100.100.150.0/24 is directly connected, Null0
S       100.100.200.0/24 is directly connected, Null0
O IA    192.168.1.0/24 [110/3] via 2.1.1.1, 00:21:07, GigabitEthernet0/1
```

🧠 **What this shows**

| Router | Sees Router0's 100.100.50/60/70.0 as | Why |
|:-------|:------------------------------------:|:----|
| Router0 | `S` (its own static → Null0) | It is the origin |
| Router1 (ABR) | **`O N2`** | Still a Type-7 LSA (NSSA) |
| Router2 | **`O E2`** | Translated to Type-5 by the ABR |

---

### 🟢 Stage 4: Router0 (Inside the NSSA) — Only a Default Route ⭐

```
R1# show ip route
Gateway of last resort is 1.1.1.2 to network 0.0.0.0

C       1.1.1.0/24 is directly connected, GigabitEthernet0/0
C       10.1.1.1/32 is directly connected, Loopback1
O       20.1.1.1/32 [110/2] via 1.1.1.2, 00:22:52, GigabitEthernet0/0
S       100.100.50.0/24 is directly connected, Null0
S       100.100.60.0/24 is directly connected, Null0
S       100.100.70.0/24 is directly connected, Null0
C       192.168.1.0/24 is directly connected, GigabitEthernet0/1
O*IA    0.0.0.0/0 [110/2] via 1.1.1.2, 00:05:46, GigabitEthernet0/0      ◄── 🔥 DEFAULT ROUTE from the ABR
```

🧠 **What this shows**

- ❌ **No** routes to `2.1.1.0/24`, `30.1.1.1/32`, `192.168.2.0/24` or `100.100.100/150/200.0` — the NSSA keeps the routing table tiny
- ✅ A single **`O*IA 0.0.0.0/0`** points to the ABR (`1.1.1.2`), so Router0 can still reach everything else
- This "no specific inter-area routes, only a default" behavior is what **`area 1 nssa no-summary`** (a *totally NSSA*) produces — a plain NSSA would still show `O IA` routes
- The default route is **younger (5:46)** than the other OSPF routes (22+ min), consistent with `no-summary` being applied in a later step

---

### 🧾 Evidence Summary

| # | Screenshot | What it proves |
|:-:|:-----------|:---------------|
| 1 | Topology | Area 1 (NSSA) vs Area 0, ASBR/ABR roles, static routes, LSA Type-7 / Type-5 labels |
| 2 | Router0 database | **Type-7** LSAs for 100.100.50/60/70.0 originated by 10.1.1.1 |
| 3 | Router1 database | ABR creates **Type-5** LSAs (ADV 20.1.1.1) for Router0's networks |
| 4 | Router2 routing table | Router0's networks appear as **`O E2`**; Area 1 routes as **`O IA`** |
| 5 | Router1 routing table | Both **`O N2`** (NSSA) and **`O E2`** (backbone) routes present |
| 6 | Router0 routing table | Only a **default route** `O*IA 0.0.0.0/0` — external routes blocked |

### 📷 Add These Screenshots to Complete the Lab

```bash
show ip ospf                                  # on Router0/Router1: "Area 1 ... It is a NSSA area"
show ip ospf database nssa-external           # Type-7 detail (P-bit, forwarding address)
show ip ospf database external                # Type-5 detail on Router2
show running-config | section ospf            # exact NSSA / no-summary lines
ping 30.1.1.1 source loopback1                # from Router0 — works through the default route
```

---

## 🧱 OSPF Area Types Compared

| Area Type | Type-1/2 | Type-3 | Type-4/5 | Type-7 | Default Route | ASBR allowed? |
|:----------|:--------:|:------:|:--------:|:------:|:-------------:|:-------------:|
| **Normal** | ✅ | ✅ | ✅ | ❌ | No | ✅ |
| **Stub** | ✅ | ✅ | ❌ | ❌ | ✅ auto (Type-3) | ❌ |
| **Totally Stubby** | ✅ | ❌ (only default) | ❌ | ❌ | ✅ (Type-3) | ❌ |
| **NSSA** | ✅ | ✅ | ❌ | ✅ | ❌ optional | ✅ |
| **Totally NSSA** *(this lab)* | ✅ | ❌ (only default) | ❌ | ✅ | ✅ (Type-3) | ✅ |

---

## ⚔️ NSSA vs Stub

| | 🟦 Stub Area | 🟧 NSSA |
|:--|:------------:|:-------:|
| Blocks Type-5 LSAs | ✅ | ✅ |
| Can contain an ASBR | ❌ **No** | ✅ **Yes** |
| External routes from inside the area | Not possible | Sent as **Type-7** |
| Route code for those externals | — | **`N1` / `N2`** (inside) → **`E1` / `E2`** (after translation) |
| Translation at the ABR | — | **Type-7 → Type-5** |
| Config command | `area 1 stub` | `area 1 nssa` |

> 💡 *"Not-So-Stubby"* = a stub area that is allowed to have **a little external connectivity**.

---

## 🔢 Route Codes & Metrics

| Code | Meaning | Where seen in this lab |
|:----:|:--------|:-----------------------|
| `O` | Intra-area OSPF route | Router1 → 10.1.1.1/32 |
| `O IA` | Inter-area route | Router2 → 192.168.1.0/24 |
| `O*IA` | **Default route** from an ABR (Type-3) | Router0 → 0.0.0.0/0 |
| `O N2` | **NSSA external type 2** | Router1 → 100.100.50.0/24 |
| `O E2` | **External type 2** | Router2 → 100.100.50.0/24 |

**Why `[110/20]`?** Redistributed routes get a default **seed metric of 20**. With **type 2** (the default), that value stays **constant** across the network — internal cost is *not* added.

| Type | Metric behavior |
|:----:|:----------------|
| **E2 / N2** *(default)* | Fixed at the seed value (20) everywhere |
| **E1 / N1** | Seed value **+ internal OSPF cost** to the ASBR |

---

## 🧰 Troubleshooting Guide

| Symptom | Likely Cause | Fix |
|:--------|:-------------|:----|
| Neighbor won't form across the NSSA boundary | **Area type mismatch** (one side `nssa`, other normal) | Configure `area 1 nssa` on **all** routers in Area 1 |
| No Type-7 LSAs appear | Missing `redistribute static subnets` on the NSSA ASBR | Add it under `router ospf 1` |
| Router0's networks missing on Router2 | ABR not translating (area misconfig / P-bit) | Check `show ip ospf database nssa-external` on the ABR |
| NSSA router has no route to the backbone | No default route | Use `area 1 nssa no-summary` or `default-information-originate` |
| Still seeing `O IA` routes inside the NSSA | `no-summary` not configured on the ABR | Add `no-summary` on the **ABR only** |
| Externals show `E2` not `N2` on the ABR's neighbors | Normal — Type-7 is **translated** at the ABR | Expected behavior |
| Redistributed static routes missing | Missing `subnets` keyword | `redistribute static subnets` |

---

## 🚀 Advanced Topics

### 🔹 Default route into a plain NSSA

A plain NSSA doesn't get a default route automatically. Ask for one:

```bash
router ospf 1
 area 1 nssa default-information-originate
```
The default then arrives as **`O*N2 0.0.0.0/0`** (a Type-7 default) instead of `O*IA`.

### 🔹 Control the translation (several ABRs)

When an NSSA has **more than one ABR**, only **one** (the one with the **highest router ID**) translates Type-7 to Type-5. You can influence it with:

```bash
area 1 nssa translate type7 always
```

### 🔹 Stop a Type-7 route from being translated

Setting `no-redistribution` on the NSSA's ABR lets **only** the area's own externals be advertised without leaking others:

```bash
area 1 nssa no-redistribution
```

### 🔹 Type-1 external metric

```bash
redistribute static subnets metric 50 metric-type 1
```
Gives `N1`/`E1` routes whose metric grows with the internal path cost.

---

## 🎓 Key Takeaways

1. 🔸 **NSSA = stub area + ASBR allowed**
2. 🔸 An NSSA **blocks Type-5** but uses **Type-7** for its own external routes
3. 🔸 The **ABR translates Type-7 → Type-5** (ADV router becomes the ABR: 20.1.1.1)
4. 🔸 Inside the NSSA, external routes are **`N2`**; elsewhere the same routes are **`E2`**
5. 🔸 `area 1 nssa` must match on **every router** in the area
6. 🔸 `no-summary` on the **ABR** turns the area into a *totally NSSA* with a single **`O*IA`** default route
7. 🔸 Redistributed routes use a **seed metric of 20** and **type 2** by default (`[110/20]`)
8. 🔸 Verify with the **database** (`show ip ospf database`) **and** the **routing table**
9. 🔸 The routing table of an NSSA router stays **tiny** — a major scalability win

---

## 🧪 Challenges

1. Remove `no-summary` — which routes reappear on Router0 (`O IA`)? Is the default route still there?
2. Change to a **stub** area (`area 1 stub`) — what happens to Router0's static route redistribution?
3. Switch to `metric-type 1` and compare `N1`/`E1` metrics
4. Add a second ABR and find out **which** one translates (hint: highest router ID)
5. Use `show ip ospf database nssa-external` and explain the **forwarding address** and **P-bit**
6. Add a default route with `default-information-originate` instead of `no-summary`

---

## 📁 Repository Structure

```
📦 OSPF-NSSA
 ┣ 📜 README.md
 ┣ 📂 images
 ┃ ┣ 🖼️ topology.png
 ┃ ┣ 🖼️ 01-router0-database-type7.png
 ┃ ┣ 🖼️ 02-router1-database-type5-translation.png
 ┃ ┣ 🖼️ 03-router2-routing-table.png
 ┃ ┣ 🖼️ 04-router1-routing-table-n2-e2.png
 ┃ ┗ 🖼️ 05-router0-default-route.png
 ┗ 📂 lab-file
   ┗ 📦 OSPF-NSSA.pkt
```

---

## 🔗 Related Labs

| Topic | Lab |
|:------|:----|
| OSPF | **OSPF NSSA** ◄── *you are here* |
| EIGRP | EIGRP Stub Routing — the EIGRP take on "limit what a spoke advertises" |
| EIGRP | EIGRP Route Summarization — shrinking routing tables with summary routes |
| EIGRP | EIGRP Redistribution — external routes in EIGRP (`D EX`) vs OSPF (`E2`/`N2`) |

`OSPF` · `NSSA` · `LSA Type-7` · `LSA Type-5` · `ABR` · `ASBR` · `Redistribution` · `Area Types` · `Default Route`

---

## 👨‍💻 Author

**Maaz Khan**
CCNA-Certified Network & NOC Engineer

[![LinkedIn](https://img.shields.io/badge/LinkedIn-maazkhanms-0A66C2?style=for-the-badge&logo=linkedin&logoColor=white)](https://www.linkedin.com/in/maazkhanms)
[![GitHub](https://img.shields.io/badge/GitHub-maazkhanms-181717?style=for-the-badge&logo=github&logoColor=white)](https://github.com/maazkhanms)

---

<div align="center">

⭐ **If this lab helped you, please star the repository!** ⭐

*Part of my ongoing Cisco Packet Tracer lab series — routing, switching, security & more.*

</div>
