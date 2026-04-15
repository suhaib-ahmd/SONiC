# Introduction to SONiC & SONiC Configuration
## Lab 1 — 3-Leaf / 1-Spine

---

## Introduction

This lab introduces the SONiC (Software for Open Networking in the Cloud) network operating system. You will learn how to verify platform and software health, manage SONiC images, configure basic networking primitives (hostname, users, interfaces, loopbacks, VLANs), and interact with the underlying REDIS configuration database. By the end of this lab you will have a solid foundation for subsequent labs that build eBGP underlay and VXLAN EVPN overlay fabrics.

**SONiC version:** Community SONiC (FRR-based routing stack)


## Task 1 — System Verification

> **Goal:** Confirm that SONiC is running correctly on all devices, identify the platform and software version, and verify the health of core services.

### 1.1 Platform and SONiC Software Verification

Run on **Leaf1, Leaf2, Leaf3, and Spine4**:

```bash
show version
```

Expected output (example from Leaf1):
```
SONiC Software Version: SONiC.202405cz.2.2.2
SONiC OS Version: 12
Distribution: Debian 12.13
Kernel: 6.1.0-11-2-amd64
Build commit: b0e9485bd
Build date: Thu Mar  5 08:32:57 UTC 2026
Built by: sonicci@sonic-rtp-1-lnx.cisco.com

Platform: x86_64-8102_64h_o-r0
HwSKU: Cisco-8102-C64
ASIC: cisco-8000
ASIC Count: 1
Serial Number: CSNRH5LKMQG
Model Number: 8102-64H-O
Hardware Revision: 0.20
Uptime: 00:12:56 up 1 day, 18:08,  1 user,  load average: 2.03, 2.66, 2.62
Date: Wed 15 Apr 2026 00:12:56

Docker images:
REPOSITORY                    TAG              IMAGE ID       SIZE
docker-orchagent              202405cz.2.2.2   182a7e9af168   385MB
docker-orchagent              latest           182a7e9af168   385MB
docker-sflow                  202405cz.2.2.2   c725b317a099   373MB
docker-sflow                  latest           c725b317a099   373MB
docker-fpm-frr                202405cz.2.2.2   86b8f513c3bf   404MB
docker-fpm-frr                latest           86b8f513c3bf   404MB
docker-nat                    202405cz.2.2.2   cf051a6c5ea4   375MB
docker-nat                    latest           cf051a6c5ea4   375MB
docker-platform-monitor       202405cz.2.2.2   aaed09790a23   471MB
docker-platform-monitor       latest           aaed09790a23   471MB
docker-macsec                 latest           236ddb6086c7   375MB
docker-dhcp-relay             latest           87a1e577fc5d   374MB
docker-snmp                   202405cz.2.2.2   a96f91740909   386MB
docker-snmp                   latest           a96f91740909   386MB
docker-stp                    202405cz.2.2.2   fb2eb46d8f25   676MB
docker-stp                    latest           fb2eb46d8f25   676MB
docker-sonic-mgmt-framework   202405cz.2.2.2   d50ca6b35d61   437MB
docker-sonic-mgmt-framework   latest           d50ca6b35d61   437MB
docker-teamd                  202405cz.2.2.2   94a5864322fc   372MB
docker-teamd                  latest           94a5864322fc   372MB
docker-router-advertiser      202405cz.2.2.2   8a005ebf2bc7   342MB
docker-router-advertiser      latest           8a005ebf2bc7   342MB
docker-apm                    202405cz.2.2.2   75b8681bd27f   351MB
docker-apm                    latest           75b8681bd27f   351MB
docker-lldp                   202405cz.2.2.2   0be80aa06605   388MB
docker-lldp                   latest           0be80aa06605   388MB
docker-mux                    202405cz.2.2.2   0310d97a17fd   394MB
docker-mux                    latest           0310d97a17fd   394MB
docker-database               202405cz.2.2.2   8e5e3e655964   350MB
docker-database               latest           8e5e3e655964   350MB
docker-sonic-gnmi             202405cz.2.2.2   7687a7935566   458MB
docker-sonic-gnmi             latest           7687a7935566   458MB
docker-eventd                 202405cz.2.2.2   c071b52d2b73   342MB
docker-eventd                 latest           c071b52d2b73   342MB
docker-ipxeserver-cisco       202405cz.2.2.2   3dcf5dbeaded   353MB
docker-ipxeserver-cisco       latest           3dcf5dbeaded   353MB
docker-syncd-cisco            202405cz.2.2.2   a779101f92fb   1.11GB
docker-syncd-cisco            latest           a779101f92fb   1.11GB
docker-gbsyncd-cisco          202405cz.2.2.2   d423df275eee   393MB
docker-gbsyncd-cisco          latest           d423df275eee   393MB
```

Key fields to note:
- **Platform** and **HwSKU** — identify the hardware
- **ASIC** — the forwarding chip vendor (e.g., barefoot, memory, memory)
- **SONiC Software Version** — the running image version

```bash
show platform summary
```
```bash
Platform: x86_64-8102_64h_o-r0
HwSKU: Cisco-8102-C64
ASIC: cisco-8000
ASIC Count: 1
Serial Number: CSNRH5LKMQG
Model Number: 8102-64H-O
Hardware Revision: 0.20
```
This provides a concise view of the platform name, HwSKU, and ASIC type.

### 1.2 Container Verification

SONiC runs its subsystems as Docker containers. Each container is responsible for a specific function (BGP, LLDP, SNMP, teamd, etc.).

Run on **Leaf1, Leaf2, Leaf3, and Spine4**:

```bash
docker ps
```

Expected output shows containers like `bgp`, `teamd`, `lldp`, `snmp`, `swss`, `syncd`, `database`, `pmon`, etc. All should show `Up` status.


## Task 2 — Image Management

> **Goal:** Understand how to view installed SONiC images, install new images, switch between them, and use different reboot strategies.

### 2.1 Image Verification

Run on **any device**:

```bash
sudo sonic-installer list
```

Expected output:
```
Current: SONiC-OS-202405cz.2.2.2
Next: SONiC-OS-202405cz.2.2.2
Available:
SONiC-OS-202405cz.2.2.2
```

- **Current** — the image the system booted from
- **Next** — the image that will be used on next reboot
- **Available** — all installed images (SONiC supports dual image: two images can be installed simultaneously)

### 2.2 SONiC Upgrade and Downgrade

To install a new image (upgrade or downgrade):

```bash
sudo sonic-installer install <image-url-or-path>
```

After installation, the new image is set as `Next` automatically. To switch back to the previous image without installing:

```bash
sudo sonic-installer set-default <image-name>
```

To remove an unused image:
```bash
sudo sonic-installer remove <image-name>
```

### 2.3 Reboot Mechanisms

SONiC supports several reboot strategies with different trade-offs:

| Reboot Type | Downtime | Description |
|-------------|----------|-------------|
| `sudo reboot` | Full | Cold reboot — all services restart, full convergence required |
| `sudo fast-reboot` | ~30s | Dataplane stays up briefly; control plane restarts |
| `sudo warm-reboot` | Minimal | Dataplane and most state preserved; hitless for forwarding |
| `sudo express-reboot` * | Sub-second | NPU state is saved and restored via DMA; control plane follows warm-reboot flow |

> \* **Platform dependency:** `warm-reboot`, `fast-reboot`, and `express-reboot` support depends on platform and ASIC. Not all platforms support all reboot types. Express Boot currently requires Cisco 8000 series hardware with SDK-level support. Check your platform release notes before relying on these features.


### 2.4 Reference Commands

| Command | Description |
|---------|-------------|
| `sonic-installer list` | Show installed images and boot selection |
| `sonic-installer install <url>` | Install a new SONiC image |
| `sonic-installer set-default <name>` | Set the next-boot image |
| `sonic-installer remove <name>` | Remove an installed image |
| `sudo reboot` | Cold reboot |
| `sudo fast-reboot` | Fast reboot (brief dataplane hit) |
| `sudo warm-reboot` | Warm reboot (minimal disruption) |
| `sudo express-reboot` | Express reboot (sub-second disruption) * |
| `show boot` | Show next-boot image |

---

## Task 3 — Basic SONiC Configuration

> **Goal:** Configure fundamental device settings on Leaf1 — users, interface IPs, loopback addresses, VLANs, and more.
>
> All commands in Task 3 are run on **Leaf1** only.

### 3.1 Configuring Users

SONiC uses standard Linux user management. The default user is `admin`.

```bash
sudo useradd -m -s /bin/bash labuser
sudo passwd labuser
```

To add the user to the sudo group:
```bash
sudo usermod -aG sudo labuser
```

Verify:
```bash
cat /etc/passwd | grep labuser
```

### 3.2 Configuring Interface IPv4

```bash
sudo config interface ip add Ethernet0 1.4.1.1/24
```

Verify:
```bash
show ip interfaces
```

```bash
Interface    Master    IPv4 address/mask    Admin/Oper    BGP Neighbor    Neighbor IP
-----------  --------  -------------------  ------------  --------------  -------------
Ethernet0              1.4.1.1/24           up/up         N/A             N/A
Loopback0              1.1.1.1/32           up/up         N/A             N/A
docker0                240.127.1.1/24       up/down       N/A             N/A
eth0                   192.168.122.46/24    up/up         N/A             N/A
eth4                   192.168.123.105/24   up/up         N/A             N/A
lo                     127.0.0.1/16         up/up         N/A             N/A
```

Expected: Ethernet0 shows `1.4.1.1/24` with `up/up` status.

### 3.3 Configuring Loopback Interface

```bash
sudo config interface ip add Loopback0 1.1.1.1/32
```

Verify:
```bash
show ip interfaces | grep Loopback
```

```bash
Loopback0              1.1.1.1/32           up/up         N/A             N/A
```

> Loopback interfaces are used as router-IDs and VTEP source addresses in subsequent labs. They are always up and do not depend on physical link state.

### 3.4 Configuring VLANs

> **Note:** Ethernet16 and Ethernet20 are used here for VLAN exercises. These ports may not have physical links connected, so they will show `oper down`. The configuration is still accepted and can be verified in `show vlan brief` and CONFIG_DB — you do not need an active link to configure VLANs.

#### Access Port (Untagged)

Create VLAN 10 and add Ethernet16 as an access port:

```bash
sudo config vlan add 10
sudo config vlan member add -u 10 Ethernet16
```

The `-u` flag adds the port as an untagged (access) member. Frames entering this port are placed into VLAN 10 without any 802.1Q tag.

#### Trunk Port (Tagged)

A trunk port carries multiple VLANs as tagged (802.1Q) traffic. Create VLAN 20 and add Ethernet20 as a trunk carrying both VLANs. You can combine tagged and untagged on the same port — the untagged VLAN becomes the native VLAN:

```bash
sudo config vlan add 20
sudo config vlan member add -u 10 Ethernet20    # native VLAN (untagged)
sudo config vlan member add 20 Ethernet20        # tagged
```

#### Verify

```bash
show vlan brief
```

Expected:
```
+-----------+--------------+------------+----------------+-------------+--------------------------+-----------------------+
|   VLAN ID | IP Address   | Ports      | Port Tagging   | Proxy ARP   | Static Anycast Gateway   | DHCP Helper Address   |
+===========+==============+============+================+=============+==========================+=======================+
|        10 |              | Ethernet16 | untagged       | disabled    | disabled                 |                       |
|           |              | Ethernet20 | untagged       |             |                          |                       |
+-----------+--------------+------------+----------------+-------------+--------------------------+-----------------------+
|        20 |              | Ethernet20 | tagged         | disabled    | disabled                 |                       |
+-----------+--------------+------------+----------------+-------------+--------------------------+-----------------------+
```

#### Cleanup

To remove a VLAN member:
```bash
sudo config vlan member del 20 Ethernet20
```

To delete a VLAN (all members must be removed first):
```bash
sudo config vlan del 20
```

### 3.5 VLAN SVI (Switched Virtual Interface)

An SVI gives a VLAN a Layer 3 IP address, enabling the switch to route traffic in and out of that VLAN. This is the foundation for inter-VLAN routing and is also used for Static Anycast Gateways in the EVPN lab.

```bash
sudo config interface ip add Vlan10 10.10.10.1/24
```

Verify:
```bash
show ip interfaces
```

```bash
Interface    Master    IPv4 address/mask    Admin/Oper    BGP Neighbor    Neighbor IP
-----------  --------  -------------------  ------------  --------------  -------------
Ethernet0              1.4.1.1/24           up/up         N/A             N/A
Loopback0              1.1.1.1/32           up/up         N/A             N/A
Vlan10                 10.10.10.1/24        up/up         N/A             N/A
docker0                240.127.1.1/24       up/down       N/A             N/A
eth0                   192.168.122.46/24    up/up         N/A             N/A
eth4                   192.168.123.105/24   up/up         N/A             N/A
lo                     127.0.0.1/16         up/up         N/A             N/A
```
Expected: `Vlan10` appears with `10.10.10.1/24` and `up/up` status.

Hosts connected to access ports on Vlan10 can now use the SVI IP as their default gateway.

To remove an SVI IP:
```bash
sudo config interface ip remove Vlan10 10.10.10.1/24
```

### 3.6 PortChannel (LACP)

A PortChannel bundles multiple physical links into a single logical interface using LACP (Link Aggregation Control Protocol). This provides link redundancy and increased bandwidth. In the EVPN lab, PortChannels are also used for multihoming with a shared ESI.

```bash
sudo config portchannel add PortChannel1
sudo config portchannel member add PortChannel1 Ethernet4
```

Verify:
```bash
show interfaces portchannel
```

Expected:
```
Flags: A - active, I - inactive, Up - up, Dw - Down, N/A - not available,
       S - selected, D - deselected, * - not synced
  No.  Team Dev      Protocol     Ports
-----  ------------  -----------  ---------------
    1  PortChannel1  LACP(A)(Up)  Ethernet4(S)
```

`LACP(A)(Up)` means LACP is active and the bundle is up. `(S)` on the member means it is selected and forwarding.

> **Note:** In a later lab (EVPN VXLAN Multihoming), we will bring up LACP between Leaf1, Leaf2, and Host12 using a shared Ethernet Segment. At that point the PortChannel will come up with active members on both leaves.

### 3.7 Configuring MTU

The default MTU on SONiC interfaces is 9100. Here we set Ethernet16 to 1500 to demonstrate the command:

```bash
sudo config interface mtu Ethernet16 1500
```

Verify:
```bash
show interfaces status | grep -w Ethernet16
```

```bash
  Ethernet16  2312,2313,2314,2315     100G   1500    N/A     etp4         trunk      up       up  QSFP28 or later         N/A
```

> When changing MTU on fabric links, ensure both ends match. Mismatched MTU can cause silent packet drops for large frames.

### 3.8 VRF Configuration

A VRF (Virtual Routing and Forwarding) creates an isolated routing table, allowing overlapping IP addresses and separate forwarding domains on the same device. VRFs are used extensively in the EVPN lab for L3VNI routing.

> **Note:** Ethernet24 and Ethernet28 are used here for VRF exercises. These ports are not used elsewhere in the lab.

#### Create VRFs

```bash
sudo config vrf add VrfBlue
sudo config vrf add VrfRed
```

Verify:
```bash
show vrf
```

```bash
VRF      Interfaces
-------  ------------
VrfBlue
VrfRed
```

#### Bind Interfaces to a VRF

```bash
sudo config interface vrf bind Ethernet24 VrfBlue
sudo config interface vrf bind Ethernet28 VrfRed
```

Once bound, you can assign IPs within the VRF:
```bash
sudo config interface ip add Ethernet24 172.16.1.1/24
sudo config interface ip add Ethernet28 192.168.50.1/24
```

Verify:
```bash
show ip interfaces
```

```bash
Interface    Master    IPv4 address/mask    Admin/Oper    BGP Neighbor    Neighbor IP
-----------  --------  -------------------  ------------  --------------  -------------
Ethernet0              1.4.1.1/24           up/up         N/A             N/A
Ethernet24   VrfBlue   172.16.1.1/24        up/up         N/A             N/A
Ethernet28   VrfRed    192.168.50.1/24      up/up         N/A             N/A
Loopback0              1.1.1.1/32           up/up         N/A             N/A
docker0                240.127.1.1/24       up/down       N/A             N/A
eth0                   192.168.122.46/24    up/up         N/A             N/A
eth4                   192.168.123.105/24   up/up         N/A             N/A
lo                     127.0.0.1/16         up/up         N/A             N/A
```

Ethernet24 and Ethernet28 will appear with their IPs and VRF assignment.

#### Unbind an Interface

```bash
sudo config interface ip remove Ethernet28 192.168.50.1/24
sudo config interface vrf unbind Ethernet28
```

> You must remove all IP addresses from an interface before unbinding it from a VRF.

#### Delete a VRF

```bash
sudo config vrf del VrfRed
```

> All interfaces must be unbound from a VRF before it can be deleted.

VrfBlue remains configured with Ethernet24 for use in the static routes section below.

### 3.9 Static Routes

Static routes provide reachability to destinations not directly connected. They are useful for out-of-band management paths or stub networks.

Add a static route to a remote subnet via Ethernet24's next-hop:
```bash
sudo config route add prefix 172.30.0.0/16 nexthop 172.16.1.254
```

To add a static route within a VRF:
```bash
sudo config route add prefix vrf VrfBlue 172.31.0.0/16 nexthop 172.16.1.254
```

To remove a static route:
```bash
sudo config route del prefix 172.30.0.0/16 nexthop 172.16.1.254
```

> In subsequent labs, BGP and EVPN replace static routes for fabric and overlay reachability.

### 3.10 NTP Configuration

Synchronized time across all devices is critical for correlating logs, troubleshooting, and certificate validation.

```bash
sudo config ntp add 10.100.0.1
sudo config ntp add 10.100.0.2
```

To remove an NTP server:
```bash
sudo config ntp del 10.100.0.2
```

### 3.11 Syslog Configuration

Remote syslog sends device logs to a centralized server for monitoring, alerting, and long-term storage.

```bash
sudo config syslog add 10.100.0.10
```

To add a second syslog server for redundancy:
```bash
sudo config syslog add 10.100.0.11
```

To remove a syslog server:
```bash
sudo config syslog del 10.100.0.10
```


Save all configuration on **all devices**:
```bash
sudo config save -y
```

---

## Task 4 — Configuration Management

> **Goal:** Understand SONiC's configuration lifecycle — how running config relates to startup config, how to save and load config files, and how to query the REDIS database directly.

### 4.1 Running vs Startup Configuration

SONiC maintains two configuration states:

| Config | Location | Description |
|--------|----------|-------------|
| Running | REDIS (CONFIG_DB) | The active, in-memory configuration |
| Startup | `/etc/sonic/config_db.json` | The on-disk config loaded at boot |

- **`config save`** writes the running config from REDIS to `/etc/sonic/config_db.json`
- **`config load`** merges a JSON file into the running config (REDIS)
- **`config reload`** replaces the running config with the startup config from disk

### 4.2 REDIS Database Queries

SONiC stores all configuration and state in REDIS databases. The primary ones are:

| DB ID | Name | Purpose |
|-------|------|---------|
| 0 | APPL_DB | Application state (routes, neighbors) |
| 1 | ASIC_DB | ASIC/SAI object programming |
| 4 | CONFIG_DB | Running configuration |
| 6 | STATE_DB | Operational state |

#### Viewing the full running config as JSON

Run on **any device**:
```bash
show runningconfiguration all
```

#### Querying CONFIG_DB directly

Run on **any device**:
```bash
sonic-db-cli CONFIG_DB keys '*'
```

This lists all keys in the configuration database.

To query a specific table (e.g., all interface configurations):
```bash
sonic-db-cli CONFIG_DB keys 'INTERFACE*'
```

To get all fields of a specific key:
```bash
sonic-db-cli CONFIG_DB hgetall 'DEVICE_METADATA|localhost'
```

Expected output (key-value pairs):
```
hostname
Leaf1
docker_routing_config_mode
unified
type
LeafRouter
...
```

#### Querying interface IP configuration

```bash
sonic-db-cli CONFIG_DB keys 'INTERFACE*'
```

Example output:
```
INTERFACE|Ethernet0|1.4.1.1/24
INTERFACE|Loopback0|1.1.1.1/32
INTERFACE|Ethernet0
INTERFACE|Loopback0
```

#### Querying VLAN configuration

```bash
sonic-db-cli CONFIG_DB keys 'VLAN*'
```

```bash
sonic-db-cli CONFIG_DB hgetall 'VLAN|Vlan10'
```

#### Querying STATE_DB for operational data

```bash
sonic-db-cli STATE_DB keys '*'
```

To check interface operational state:
```bash
sonic-db-cli APPL_DB hgetall 'PORT_TABLE:Ethernet0'
```

### 4.3 Config Save, Load, and Reload

#### Save running config to startup

Run on **any device**:
```bash
sudo config save -y
```

This writes the current REDIS CONFIG_DB to `/etc/sonic/config_db.json`.

#### Load a config file into running config

```bash
sudo config load /path/to/config.json -y
```

This merges the JSON file into the running CONFIG_DB. Existing keys not in the file are preserved.

#### Reload startup config (replace running config)

```bash
sudo config reload -y
```

> **Caution:** `config reload` replaces the entire running config with the startup file. Any unsaved changes will be lost. Services may restart.

#### View the startup config file directly

```bash
cat /etc/sonic/config_db.json | python3 -m json.tool | head -50
```

#### Compare running vs startup

A quick way to check for unsaved changes:
```bash
show runningconfiguration all > /tmp/running.json
diff /etc/sonic/config_db.json /tmp/running.json
```

---

## End of Lab 1

Lab 1 is complete. You now have:

- Verified the SONiC platform, software version, and service health
- Understood image management and reboot strategies
- Configured users, interface IPs, loopbacks, and VLANs
- Configured MTU, VLAN trunks, SVIs, and PortChannels
- Added static routes, NTP servers, and remote syslog
- Learned how to save, load, and reload configuration
- Queried the REDIS database directly for configuration and state

Proceed to Lab 2 (Underlay BGP) to configure eBGP routing between the leaves and spine.
