# Underlay BGP — Minimal Config Reference
## 3-Leaf / 1-Spine

---

## Device Addressing

| Device | AS | Loopback   | Ethernet0    | Spine4-facing IP |
|--------|----|------------|--------------|------------------|
| Spine4 | 4  | 4.4.4.4/32 | —            | —                |
| Leaf1  | 1  | 1.1.1.1/32 | 1.4.1.1/24   | 1.4.1.4/24       |
| Leaf2  | 2  | 2.2.2.2/32 | 2.4.1.2/24   | 2.4.1.4/24       |
| Leaf3  | 3  | 3.3.3.3/32 | 3.4.1.3/24   | 3.4.1.4/24       |

---

## Step 1 — Split-Unified Routing Mode

> Run on **Leaf1, Leaf2, and Leaf3**. Required before configuring FRR.

```bash
cat > /tmp/routing_mode.json << 'EOF'
{
    "DEVICE_METADATA": {
        "localhost": {
            "docker_routing_config_mode": "split-unified"
        }
    }
}
EOF

sudo config load /tmp/routing_mode.json -y
sudo config save -y
sudo systemctl restart bgp
```

Verify:
```bash
show runningconfiguration all | grep docker_routing_config_mode
# Expected: "docker_routing_config_mode": "split-unified"
```

---

## Step 2 — Interface IPs

Run on **Leaf1**:
```bash
sudo config interface ip add Ethernet0 1.4.1.1/24
sudo config interface ip add Loopback0 1.1.1.1/32
sudo config save -y
```

Run on **Leaf2**:
```bash
sudo config interface ip add Ethernet0 2.4.1.2/24
sudo config interface ip add Loopback0 2.2.2.2/32
sudo config save -y
```

Run on **Leaf3**:
```bash
sudo config interface ip add Ethernet0 3.4.1.3/24
sudo config interface ip add Loopback0 3.3.3.3/32
sudo config save -y
```

Run on **Spine4**:
```bash
sudo config interface ip add Ethernet0 1.4.1.4/24
sudo config interface ip add Ethernet4 2.4.1.4/24
sudo config interface ip add Ethernet8 3.4.1.4/24
sudo config interface ip add Loopback0 4.4.4.4/32
sudo config save -y
```

---

## Step 3 — Underlay BGP

> **FRR defaults to be aware of:**
> - **`bgp ebgp-requires-policy`** is enabled by default in FRR. Every eBGP neighbor must have an explicit inbound and outbound route-map applied, or no routes will be exchanged. This is why `route-map PASS` is applied to all neighbors.
> - **`no bgp default ipv4-unicast`** is set explicitly here. By default in FRR, all neighbors are automatically activated in the IPv4 unicast address-family. Disabling this ensures neighbors are only activated where explicitly configured, giving you full control over which address-families are used per neighbor.

> Enter `vtysh` on each device before pasting.

Run on **Leaf1**:
```
ip prefix-list LOOPBACKS seq 5 permit 1.1.1.1/32

route-map PASS permit 10
exit

route-map ADVERTISE permit 10
 match ip address prefix-list LOOPBACKS
exit

router bgp 1
 bgp router-id 1.1.1.1
 no bgp default ipv4-unicast
 neighbor SPINE peer-group
 neighbor SPINE remote-as 4
 neighbor 1.4.1.4 peer-group SPINE
 !
 address-family ipv4 unicast
  redistribute connected
  neighbor SPINE activate
  neighbor SPINE route-map PASS in
  neighbor SPINE route-map ADVERTISE out
 exit-address-family
exit
```

Run on **Leaf2**:
```
ip prefix-list LOOPBACKS seq 5 permit 2.2.2.2/32

route-map PASS permit 10
exit

route-map ADVERTISE permit 10
 match ip address prefix-list LOOPBACKS
exit

router bgp 2
 bgp router-id 2.2.2.2
 no bgp default ipv4-unicast
 neighbor SPINE peer-group
 neighbor SPINE remote-as 4
 neighbor 2.4.1.4 peer-group SPINE
 !
 address-family ipv4 unicast
  redistribute connected
  neighbor SPINE activate
  neighbor SPINE route-map PASS in
  neighbor SPINE route-map ADVERTISE out
 exit-address-family
exit
```

Run on **Leaf3**:
```
ip prefix-list LOOPBACKS seq 5 permit 3.3.3.3/32

route-map PASS permit 10
exit

route-map ADVERTISE permit 10
 match ip address prefix-list LOOPBACKS
exit

router bgp 3
 bgp router-id 3.3.3.3
 no bgp default ipv4-unicast
 neighbor SPINE peer-group
 neighbor SPINE remote-as 4
 neighbor 3.4.1.4 peer-group SPINE
 !
 address-family ipv4 unicast
  redistribute connected
  neighbor SPINE activate
  neighbor SPINE route-map PASS in
  neighbor SPINE route-map ADVERTISE out
 exit-address-family
exit
```

Run on **Spine4**:
```
route-map PASS permit 10
exit

router bgp 4
 bgp router-id 4.4.4.4
 no bgp default ipv4-unicast
 neighbor LEAF peer-group
 neighbor 1.4.1.1 remote-as 1
 neighbor 1.4.1.1 peer-group LEAF
 neighbor 2.4.1.2 remote-as 2
 neighbor 2.4.1.2 peer-group LEAF
 neighbor 3.4.1.3 remote-as 3
 neighbor 3.4.1.3 peer-group LEAF
 !
 address-family ipv4 unicast
  redistribute connected
  neighbor LEAF activate
  neighbor LEAF route-map PASS in
  neighbor LEAF route-map PASS out
 exit-address-family
exit
```

---

## Quick Verification

Run on **any leaf**:
```bash
# BGP sessions — all neighbors should show Established
vtysh -c 'show bgp ipv4 unicast summary'

# All remote loopbacks should be reachable
ping 2.2.2.2 -c3
ping 3.3.3.3 -c3
ping 4.4.4.4 -c3
```

---

## BGP Logging & Debug

> FRR's `debug bgp` commands produce real-time diagnostic output to `/var/log/frr/frr.log`. Enable only what you need — debug output is very verbose and will fill disk if left on.

### Log Configuration

Enter `vtysh` and confirm logging is directed to a file with useful timestamps:

```
configure terminal
log file /var/log/frr/frr.log
log timestamp precision 3
exit
```

Verify:
```
show logging
```

### Debug BGP Commands

All commands below are run inside `vtysh`.

**Neighbor events** — session state transitions (`Idle → Connect → Established`), hold-timer expirations, TCP resets:
```
debug bgp neighbor-events
```

**Updates** — every UPDATE message (prefixes, path attributes, next-hops):
```
debug bgp updates
```

Scope to a single direction or a specific prefix:
```
debug bgp updates in
debug bgp updates out
debug bgp updates prefix 2.2.2.2/32
```

**Keepalives** — every KEEPALIVE exchanged with peers (useful for hold-timer expiry issues):
```
debug bgp keepalives
```

**Bestpath** — shows why a route was selected or rejected:
```
debug bgp bestpath
debug bgp bestpath 2.2.2.2/32
```

**Zebra** — route installs/withdrawals between BGP and the kernel RIB:
```
debug bgp zebra
debug bgp zebra prefix 2.2.2.2/32
```

**NHT (Next-Hop Tracking)** — next-hop resolution and reachability changes:
```
debug bgp nht
```

**All categories at once** (use with caution):
```
debug bgp
```

**Show what is currently enabled:**
```
show debugging
```

### Viewing Logs

From the SONiC bash shell:
```bash
# Tail the FRR log in real time
sudo tail -f /var/log/frr/frr.log

# Filter for BGP messages
sudo grep "BGP" /var/log/frr/frr.log | tail -50

# Check syslog for bgpd messages
sudo grep "bgpd" /var/log/syslog | tail -20
```

### Disabling Debug

Always turn off debug when done. Disable individual categories or all at once:

```
no debug bgp neighbor-events
no debug bgp updates
no debug bgp keepalives
no debug bgp bestpath
no debug bgp zebra
no debug bgp nht
```

Or disable everything:
```
no debug bgp
```

From bash without entering vtysh:
```bash
vtysh -c 'no debug bgp'
```
