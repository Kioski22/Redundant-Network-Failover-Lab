# Redundant Network and Failover Lab (OSPF + BFD)

A four-router lab in GNS3 that measures how fast a network recovers when a cable or a whole router fails, and how much BFD (Bidirectional Forwarding Detection) improves that recovery.

**Author:** Randy (GitHub: Kioski22)
**Platform:** GNS3, four VyOS routers (2026.10.31-0016-rolling), two Alpine Linux hosts

## Summary

- Built a diamond topology with a primary path (R1-R2-R4) and a backup path (R1-R3-R4), running OSPF area 0.
- **Without BFD:** a pulled cable is detected in about 3 seconds, but a stopped router takes about **34 seconds** (33 lost pings) because OSPF waits for its dead timer.
- **With BFD (300 ms x 3 = 900 ms detection):** stopping R2 cost **0 to 1 lost pings** across three runs.
- Solved three troubleshooting tickets (wrong BFD peer IP, BFD not linked to OSPF, BFD timers too aggressive for a virtual lab).

## Topology

```
          [Alpine-Host-2]  192.168.1.10/24
                 |
          192.168.1.0/24  (R1 eth2)
                 |
               [ R1 ]
         eth0 /      \ eth1
   10.0.0.0/30        10.0.0.4/30
         /                \
     [ R2 ]              [ R3 ]
         \ eth1        eth1 /
   10.0.0.8/30        10.0.0.12/30
           \              /
        eth0 \          / eth1
               [ R4 ]
                 |
          192.168.4.0/24  (R4 eth2)
                 |
          [Alpine-Host-1]  192.168.4.10/24
```

| Link | Subnet | Side A | Side B | OSPF cost |
|---|---|---|---|---|
| R1-R2 (primary) | 10.0.0.0/30 | R1 eth0 = 10.0.0.1 | R2 eth0 = 10.0.0.2 | 10 |
| R2-R4 (primary) | 10.0.0.8/30 | R2 eth1 = 10.0.0.9 | R4 eth0 = 10.0.0.10 | 10 |
| R1-R3 (backup) | 10.0.0.4/30 | R1 eth1 = 10.0.0.5 | R3 eth0 = 10.0.0.6 | 20 |
| R3-R4 (backup) | 10.0.0.12/30 | R3 eth1 = 10.0.0.13 | R4 eth1 = 10.0.0.14 | 20 |

- Loopbacks (also the OSPF router IDs): R1 1.1.1.1, R2 2.2.2.2, R3 3.3.3.3, R4 4.4.4.4.
- All router links use OSPF network type point-to-point. The host-facing interfaces (R1 eth2, R4 eth2) are passive.
- The route to 192.168.4.0/24 costs **21 via R2** (primary) and **41 via R3** (backup).

## How the tests were measured

- A continuous ping between the two hosts, with a timestamp added to each line:
  `ping <host-ip> | while read l; do echo "$(date +%T) $l"; done`
- **Outage time** = first reply after the gap minus last reply before the gap, using only the host's own timestamps.
- **Lost pings** = missing sequence numbers in the gap.
- Limits: ping sends one packet per second, so times are accurate to about 1 second. Host and router clocks differ by a few seconds, so a router log time must never be subtracted from a ping time.

## Failure tests without BFD (Phase 3)

| # | Test | Detection | Outage | Path after | Failback |
|---|---|---|---|---|---|
| 1 | Suspend R1-R2 link | Instant (link down) | about 3 s (2 pings lost) | R1-R3-R4 | see Test 2 |
| 2 | Resume R1-R2 link | n/a | 0 s | R1-R2-R4 | about 10 s |
| 3 | Stop router R2 | OSPF dead timer | **about 34 s (33 pings lost)** | R1-R3-R4 | not tested |
| 4 | Suspend R3-R4 (backup link) | Instant (link down) | 0 s | R1-R2-R4 (unchanged) | not tested |
| 5 | Suspend R1-R2 and R3-R4 | Instant (link down) | Until repaired | None (hosts unreachable) | about 7 s |

**Key finding:** a failed cable is noticed immediately, because GNS3 raises a "NIC Link is Down" event on the router and OSPF drops the neighbor at once (`KillNbr`). A stopped router sends no such event, so R1 waits for the OSPF dead timer (`InactivityTimer`), which cost about 34 seconds.

Other observations:
- Test 4: users saw nothing, but redundancy was gone. A broken backup is invisible to users, so it has to be monitored.
- Test 5: with both paths down, R1 kept only 3.3.3.3 in its OSPF table, and traceroute from Host-1 returned `!N` (network unreachable) at the first hop.

## Phase 4: BFD

**Goal:** make OSPF notice a dead neighbor in under a second instead of waiting for the dead timer.

**What BFD does:** two routers exchange tiny packets every few hundred milliseconds. If several in a row are missed, BFD declares the neighbor down and tells OSPF immediately. BFD only detects failures; OSPF still decides the routes.

**Configuration pattern** (applied on both ends of all four router links):

```
set protocols bfd peer <neighbor-ip> source address <my-ip>
set protocols bfd peer <neighbor-ip> interval receive 300
set protocols bfd peer <neighbor-ip> interval transmit 300
set protocols bfd peer <neighbor-ip> interval multiplier 3
set protocols ospf interface <ifname> bfd
```

Detection time = 300 ms x 3 = **900 ms**. Three parts have to be in place: the BFD session, the matching session on the other router, and the `ospf interface ... bfd` line that makes OSPF listen.

**Verification:**

```
show bfd peers
show ip ospf neighbor
show configuration commands | match ospf
```

**Result:**

| | Lost pings when R2 is stopped |
|---|---|
| Before BFD | 33 (about 34 s) |
| With BFD (3 runs) | 0, 0 and 1 (about 1 s or less) |

After stopping R2, R1 removed it from its OSPF neighbor table within seconds instead of holding it as Full until the dead timer ran out.

## Troubleshooting tickets

| Ticket | Symptom | Root cause | Fix |
|---|---|---|---|
| INC-001 (low) | One BFD peer on R4 down, Remote ID 0. OSPF neighbors still Full. | R4 had a BFD peer pointing at 10.0.0.12 (the network address of the /30) instead of R3's 10.0.0.13. | Deleted the wrong peer and added the correct one. |
| INC-002 (medium) | All BFD sessions up, but failover relied on OSPF timers. | R1 had no `set protocols ospf interface eth0 bfd`, so OSPF ignored BFD on the R1-R2 link. | Added the line. |
| INC-003 (high) | R3-R4 BFD session cycling down, init, up. The OSPF neighbor kept dropping (`InactivityTimer`). | BFD timers set to 20 ms x 2 (40 ms detection), too tight for virtual routers. The log showed `bfdd` CPU starvation with timers firing 11 to 47 ms late. | Set both ends back to 300 ms x 3. Session and OSPF stayed up. |

Notes on each ticket:
- **INC-001:** a healthy OSPF neighbor table does not mean BFD is healthy. Without the BFD peer, failover on that link would silently fall back to the 40-second dead timer. Read `show bfd peers` for Status and Remote ID, and compare the config on both ends.
- **INC-002:** the ping test barely changed (2 lost before the fix, 1 after). A likely reason is that R4 had BFD toward R2, noticed R2 die, and OSPF passed that news to R1 through R3. The conclusive check is R1's own neighbor table right after stopping R2.
- **INC-003:** while a session is not up, `show bfd peers` shows default 1000 ms remote timers and a 2000 ms detection timeout. The real clue is the local timers: receive 20 ms, transmit 20 ms, multiplier 2. VyOS only accepts a multiplier of 2 or higher.

## What I learned

- Link failures and router failures are detected by different mechanisms. Link-down is instant, router-down depends on timers.
- BFD turns a 34-second router failure into roughly a one-second event, but only when BFD is configured on both ends and linked to OSPF.
- Faster timers are not always better. Virtual routers share a laptop CPU, so 300 ms x 3 is a safe value here, while hardware routers can run much lower.
- Monitor BFD sessions as well as OSPF neighbors, and monitor backup paths, because both can fail without users noticing.
- Measure carefully: use one clock, state the measurement resolution, and prove a fix with a direct check, not only with pings.

## Skills demonstrated

OSPF design and cost tuning, BFD configuration, failover testing and measurement, log analysis (`monitor log`), systematic troubleshooting, technical documentation.