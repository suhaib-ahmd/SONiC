# Automating SONiC with Ansible & gNMI

---

## Introduction

SONiC runs on Debian Linux and exposes multiple configuration interfaces: a Python Click-based CLI, JSON file loading, configuration patches, gNMI, and REST APIs. This flexibility makes SONiC a natural fit for network automation.

In this lab you will explore two automation approaches:

1. **Ansible** — using built-in Linux modules and the `cisco.sonic` Ansible collection to push configuration changes across multiple SONiC switches.
2. **gNMI** — using the gNMI protocol with the `gnmic` CLI tool (running directly on a SONiC device) to query and modify device state through SONiC's database, native YANG, and OpenConfig models.

---

## Lab Setup

You are working from a **Linux container** that has SSH connectivity to all devices in your pod. SSH aliases are pre-configured so you can connect by name:

```bash
ssh leaf1
ssh leaf2
ssh leaf3
ssh spine4
```

The container has `ansible` and `python3` pre-installed.

### Device Addressing

| Device | SSH Alias | Management IP    | Loopback   |
|--------|-----------|------------------|------------|
| Leaf1  | leaf1     | 192.168.122.11   | 1.1.1.1/32 |
| Leaf2  | leaf2     | 192.168.122.12   | 2.2.2.2/32 |
| Leaf3  | leaf3     | 192.168.122.13   | 3.3.3.3/32 |
| Spine4 | spine4    | 192.168.122.14   | 4.4.4.4/32 |

### Credentials

| Field    | Value    |
|----------|----------|
| Username | admin    |
| Password | password |

### gNMI Access

Part 2 covers gNMI on SONiC. The gNMI commands in this guide use `gnmic`, which is installed directly on **Leaf1** for ease of access. The trainer will demonstrate these commands live. The commands and expected outputs are included in this guide as reference.

---

## Part 1: Ansible Automation

### Background

Ansible is an open-source automation tool that uses SSH to execute tasks on remote hosts. Since SONiC runs on Linux, it can leverage the full library of Ansible built-in modules (`shell`, `copy`, `template`, etc.) in addition to purpose-built network modules.

SONiC's configuration workflow gives us three natural approaches for Ansible automation:

| Approach | Ansible Module | SONiC Mechanism |
|----------|---------------|-----------------|
| CLI commands via shell | `ansible.builtin.shell` | `config vlan add 100`, `config interface ip add`, etc. |
| cisco.sonic collection | `cisco.sonic.sonic_config` | Direct CLI over `network_cli` connection |

We will work through both of these using VLAN configuration as a running example.

---

### 1.1 Ansible Inventory

Before writing playbooks, set up the inventory file that tells Ansible how to reach each device.

**Step 1 — Create the project directory:**

```bash
mkdir -p ~/sonic-automation
cd ~/sonic-automation
```

**Step 2 — Create the inventory file:**

Create `~/sonic-automation/inventory.yaml`:

```yaml
leaf_switches:
  hosts:
    leaf1:
  vars:
    ansible_user: admin
    ansible_ssh_pass: password
    ansible_become_password: password
```

**Step 3 — Test connectivity:**

```bash
ansible all -i inventory.yaml -m ping
```

Expected output:
```
leaf1 | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
```

---

### 1.2 Method 1 — SONiC CLI via `ansible.builtin.shell`

The simplest approach: run SONiC CLI commands directly through an SSH shell. This is the same as if you SSH'd into the device and typed the commands yourself.

**Step 1 — Create the playbook:**

Create `~/sonic-automation/vlan_cli.yml`:

```yaml
---
- name: Configure VLANs using SONiC CLI
  gather_facts: false
  hosts: leaf_switches
  tasks:
    - name: Create VLAN 100
      ansible.builtin.shell: config vlan add 100
      become: true

    - name: Create VLAN 200
      ansible.builtin.shell: config vlan add 200
      become: true

    - name: Add Ethernet40 to VLAN 100 as untagged
      ansible.builtin.shell: config vlan member add 100 -u Ethernet40
      become: true

    - name: Add Ethernet44 to VLAN 200 as untagged
      ansible.builtin.shell: config vlan member add 200 -u Ethernet44
      become: true

    - name: Save configuration
      ansible.builtin.shell: config save -y
      become: true
```

**Step 2 — Run the playbook:**

```bash
ansible-playbook -i inventory.yaml vlan_cli.yml
```

Expected output:
```
PLAY [Configure VLANs using SONiC CLI] ****************************************

TASK [Create VLAN 100] ********************************************************
changed: [leaf1]

TASK [Create VLAN 200] ********************************************************
changed: [leaf1]

...

PLAY RECAP ********************************************************************
leaf1    : ok=5    changed=5    unreachable=0    failed=0
```

**Step 3 — Verify on any leaf:**

```bash
ssh leaf1
show vlan brief
```

Expected output:
```
+-----------+--------------+------------+----------------+-------------+
| VLAN ID   | IP Address   | Ports      | Port Tagging   | Proxy ARP   |
+===========+==============+============+================+=============+
| 100       |              | Ethernet40 | untagged       | disabled    |
+-----------+--------------+------------+----------------+-------------+
| 200       |              | Ethernet44 | untagged       | disabled    |
+-----------+--------------+------------+----------------+-------------+
```

> **Pros:** Simple, familiar, no extra modules needed.
> **Cons:** No idempotency — running the playbook again may produce errors if VLANs already exist. No structured error handling.

---

### 1.3 Method 2 — The `cisco.sonic` Ansible Collection

The `cisco.sonic` collection provides purpose-built modules for SONiC that support structured error handling, conditional execution (`wait_for`), and atomic rollback (`all_or_none`).

Since the VLANs are already configured from the previous playbook, we will use the collection to **verify** the existing configuration and **configure FRR routing** — tasks that highlight the collection's unique strengths.

The `cisco.sonic` modules use `network_cli` (SSH-based CLI sessions), so we need to update our inventory. Update `~/sonic-automation/inventory.yaml`:

```yaml
leaf_switches:
  hosts:
    leaf1:
  vars:
    ansible_user: admin
    ansible_ssh_pass: password
    ansible_become_password: password
    ansible_connection: network_cli
    ansible_network_os: cisco.sonic.sonic
```

**Step 1 — Install the collection (if not already installed):**

```bash
ansible-galaxy collection install cisco.sonic
```

**Step 2 — Create the playbook:**

Create `~/sonic-automation/verify_and_configure.yml`:

```yaml
---
- name: Verify VLANs and configure BGP
  hosts: leaf_switches
  gather_facts: false
  connection: network_cli
  tasks:
    - name: Verify VLANs 100 and 200 exist
      cisco.sonic.sonic_command:
        commands:
          - "show vlan brief"
        wait_for:
          - result[0] contains 100
          - result[0] contains 200
        match: all
        retries: 3
        interval: 5
      register: vlan_output

    - name: Display VLAN status
      ansible.builtin.debug:
        msg: "{{ vlan_output.stdout_lines }}"

    - name: Show BGP summary
      cisco.sonic.sonic_frr_command:
        commands:
          - "show bgp summary"
      register: bgp_output

    - name: Display BGP status
      ansible.builtin.debug:
        msg: "{{ bgp_output.stdout_lines }}"

    - name: Create a prefix-list in FRR
      cisco.sonic.sonic_frr_config:
        lines:
          - "ip prefix-list DENY_DEFAULT seq 10 deny 0.0.0.0/0"
          - "ip prefix-list DENY_DEFAULT seq 20 permit 0.0.0.0/0 le 32"
```

**Step 3 — Run the playbook:**

```bash
ansible-playbook -i inventory.yaml verify_and_configure.yml
```

#### Key Features of `cisco.sonic`

| Feature | Description |
|---------|-------------|
| `sonic_command` | Run SONiC show commands with conditional checks |
| `sonic_config` | Push SONiC CLI config with `all_or_none` rollback |
| `sonic_frr_command` | Run FRR show commands (`show bgp summary`, `show route`, etc.) |
| `sonic_frr_config` | Push FRR config with hierarchical `parents` context |
| `wait_for` | Polls command output until a condition is met or retries are exhausted |
| `match: any` / `match: all` | Requires any or all `wait_for` conditions to pass |
| `all_or_none: 'none'` | Rolls back all commands if any command fails |
| `parents` | Specifies the parent context for hierarchical configs (e.g., `router bgp 65000`) |

---

### 1.4 Ansible Summary

| Method | Best For | Idempotent? | Error Handling |
|--------|----------|-------------|----------------|
| `ansible.builtin.shell` | Quick tasks, prototyping | No | Manual (check `rc`) |
| `cisco.sonic` collection | Production automation, CI/CD pipelines | Via `wait_for` | `all_or_none`, `wait_for`, `retries` |

---

## Part 2: gNMI Automation

### Background

gNMI (gRPC Network Management Interface) is a protocol for network device configuration and telemetry based on gRPC. SONiC supports gNMI through the `gnmi` container.

SONiC's gNMI server supports three types of data models:

| Model Type | Path Style | Example |
|-----------|------------|---------|
| **SONiC DB** (sonic-db) | `TABLE/KEY` targeting a specific database | `--path PORT/Ethernet0 --target CONFIG_DB` |
| **SONiC Native YANG** | `sonic-module:container/list[key=value]` | `sonic-port:sonic-port/PORT/PORT_LIST[name=Ethernet0]` |
| **OpenConfig** | `openconfig-module:container/list[key=value]` | `openconfig-interfaces:interfaces/interface[name=Ethernet0]` |

> **Note:** The gNMI commands in this section will be **demonstrated by the trainer** using `gnmic` installed directly on Leaf1. The commands and expected outputs are provided here as reference so you can follow along and understand the gNMI workflow.

---

### 2.1 Enabling gNMI on SONiC

Before using gNMI, the gNMI server must be configured on the device. SONiC's gNMI server is managed through the `GNMI` table in CONFIG_DB.

**Step 1 — SSH into Leaf1:**

```bash
ssh leaf1
```

**Step 2 — Create the gNMI configuration file:**

In our lab we will use non-TLS (insecure) mode. Create a JSON file that configures the gNMI server without client authentication:

```bash
cat > /tmp/gnmi.json << 'EOF'
{
    "GNMI": {
        "certs": {},
        "gnmi": {
            "client_auth": "false",
            "log_level": "2",
            "port": "50051",
            "save_on_set": "false"
        }
    }
}
EOF
```

> **Note:** In a production environment, you would configure TLS certificates by populating the `certs` section with paths to `ca_crt`, `server_crt`, and `server_key`, and setting `client_auth` to `"true"`. Our lab uses insecure mode to simplify the exercises.

**Step 3 — Load the configuration into CONFIG_DB:**

```bash
sudo config load /tmp/gnmi.json -y
```

Expected output:
```
Running command: /usr/local/bin/sonic-cfggen -j /tmp/gnmi.json --write-to-db
```

**Step 4 — Restart the gnmi container to apply the changes:**

```bash
sudo systemctl restart gnmi
```

**Step 5 — Verify the gnmi container is running:**

```bash
docker ps -f name=gnmi
```

You should see the `gnmi` container with status `Up`.

**Step 6 — Verify the gNMI port is listening:**

Use `ss` to confirm the gNMI server is listening on port 50051:

```bash
sudo ss -tlnp | grep 50051
```

Expected output:
```
LISTEN  0  4096  *:50051  *:*  users:(("telemetry",pid=xxxx,fd=xx))
```

This confirms the gNMI server process is bound and listening on port 50051.

**Step 7 — Save the configuration so it persists across reboots:**

```bash
sudo config save -y
```

> **In our lab**, gNMI has already been enabled on Leaf1. The above walkthrough is provided so you understand how gNMI is configured on a SONiC device.

---

### 2.2 gNMI Capabilities

The capabilities RPC tells you which YANG models the device supports, the gNMI version, and the supported encodings.

```bash
gnmic -a localhost:50051 \
  -u admin -p password \
  --insecure \
  capabilities
```

Expected output:
```
gNMI version: 0.7.0
supported models:
  - openconfig-acl, OpenConfig working group, 1.0.2
  - openconfig-system, OpenConfig working group,
  - openconfig-platform, OpenConfig working group,
  - openconfig-network-instance, OpenConfig working group,
  - openconfig-routing-policy, OpenConfig working group,
  - openconfig-interfaces, OpenConfig working group,
  - openconfig-mclag, OpenConfig working group,
  - openconfig-lldp, OpenConfig working group, 1.0.2
  - openconfig-lldp-ext, SONiC, 0.1.0
  - ietf-yang-library, IETF NETCONF (Network Configuration) Working Group, 2016-06-21
  - sonic-db, SONiC, 0.1.0
supported encodings:
  - JSON
  - JSON_IETF
  - PROTO
```

Notice the `sonic-db` model at the bottom — this is a special SONiC model that lets you query Redis databases directly using their native schema.

---

### 2.3 SONiC DB GET — Querying CONFIG_DB Directly

The `sonic-db` model allows you to query any SONiC Redis database (CONFIG_DB, APPL_DB, STATE_DB, etc.) using the database's native key structure. This is equivalent to running `redis-cli` commands but over gNMI.

**GET a port entry from CONFIG_DB:**

```bash
gnmic -a localhost:50051 \
  -u admin -p password \
  --insecure \
  get \
  --path PORT/Ethernet0 \
  --target CONFIG_DB
```

Expected output:
```json
{
  "notification": [
    {
      "timestamp": 1731590130064395032,
      "update": [
        {
          "path": "PORT/Ethernet0",
          "val": {
            "admin_status": "up",
            "alias": "etp0",
            "index": "0",
            "lanes": "2304,2305,2306,2307,2308,2309,2310,2311",
            "mtu": "9100",
            "speed": "400000"
          }
        }
      ]
    }
  ]
}
```

This returns exactly what you would see by running `redis-cli -n 4` and issuing `HGETALL PORT|Ethernet0` — but through a standard gNMI interface.

**GET VLAN configuration:**

```bash
gnmic -a localhost:50051 \
  -u admin -p password \
  --insecure \
  get \
  --path VLAN \
  --target CONFIG_DB
```

**GET from other databases:**

You can target any SONiC database by changing `--target`:

```bash
gnmic -a localhost:50051 \
  -u admin -p password \
  --insecure \
  get \
  --path PORT_TABLE/Ethernet0 \
  --target APPL_DB
```

> **Key takeaway:** The `--target` flag selects which Redis database to query. The `--path` follows the Redis key structure: `TABLE/KEY` or just `TABLE` to get all entries.

---

### 2.4 SONiC Native YANG GET

SONiC also publishes native YANG models that map to its configuration database schema. These provide a more structured, standards-compliant interface compared to raw database access.

The native YANG models are maintained at: https://github.com/sonic-net/sonic-buildimage/tree/master/src/sonic-yang-models/yang-models

**GET a port using native YANG:**

```bash
gnmic -a localhost:50051 \
  -u admin -p password \
  --insecure \
  get \
  --path sonic-port:sonic-port/PORT/PORT_LIST[name=Ethernet0]
```

Expected output:
```json
[
  {
    "source": "localhost:50051",
    "timestamp": 1764723311188651240,
    "time": "2025-12-02T16:55:11.18865124-08:00",
    "updates": [
      {
        "Path": "sonic-port:sonic-port/PORT/PORT_LIST[name=Ethernet0]",
        "values": {
          "sonic-port:sonic-port/PORT/PORT_LIST": {
            "sonic-port:PORT_LIST": [
              {
                "admin_status": "up",
                "alias": "etp0",
                "index": 0,
                "lanes": "2304,2305,2306,2307,2308,2309,2310,2311",
                "mtu": 9100,
                "name": "Ethernet0",
                "speed": 400000,
                "subport": 0
              }
            ]
          }
        }
      }
    ]
  }
]
```

Notice the difference from the sonic-db response: the native YANG model returns typed values (integers for `mtu`, `speed`, `index`) rather than strings, and uses a structured path with module prefixes.

---

### 2.5 OpenConfig GET

OpenConfig models provide a vendor-neutral view of device configuration and state. SONiC translates between its internal CONFIG_DB/STATE_DB and the OpenConfig schema.

**GET an interface using OpenConfig:**

```bash
gnmic -a localhost:50051 \
  -u admin -p password \
  --insecure \
  get \
  --path openconfig-interfaces:interfaces/interface[name=Ethernet0]
```

Expected output (abbreviated):
```json
[
  {
    "source": "localhost:50051",
    "updates": [
      {
        "Path": "openconfig-interfaces:interfaces/interface[name=Ethernet0]",
        "values": {
          "openconfig-interfaces:interfaces/interface": {
            "openconfig-interfaces:interface": [
              {
                "config": {
                  "enabled": true,
                  "mtu": 9100,
                  "name": "Ethernet0",
                  "type": "iana-if-type:ethernetCsmacd"
                },
                "name": "Ethernet0",
                "openconfig-if-ethernet:ethernet": {
                  "config": {
                    "port-speed": "openconfig-if-ethernet:SPEED_400GB"
                  },
                  "state": {
                    "port-speed": "openconfig-if-ethernet:SPEED_400GB"
                  }
                },
                "state": {
                  "admin-status": "UP",
                  "counters": {
                    "in-octets": "1115724",
                    "in-pkts": "8521",
                    "out-octets": "1076622",
                    "out-pkts": "6304"
                  },
                  "enabled": true,
                  "mtu": 9100,
                  "name": "Ethernet0",
                  "oper-status": "UP",
                  "type": "iana-if-type:ethernetCsmacd"
                }
              }
            ]
          }
        }
      }
    ]
  }
]
```

OpenConfig responses include both `config` (desired state) and `state` (actual operational state including counters). This is richer than the sonic-db response, which only returns CONFIG_DB contents.

---

### 2.6 Comparing the Three gNMI Models

All three approaches query the same device — the difference is in the data model used:

| Aspect | SONiC DB (`sonic-db`) | SONiC Native YANG | OpenConfig |
|--------|----------------------|-------------------|------------|
| **Path style** | `TABLE/KEY --target DB` | `sonic-module:path/LIST[key=val]` | `oc-module:path/list[key=val]` |
| **Schema source** | Redis DB schema | SONiC YANG models | OpenConfig YANG models |
| **Data types** | All strings (Redis) | Typed (int, string, enum) | Typed with OC enumerations |
| **Config vs State** | Depends on target DB | Config only (native models) | Both config and state |
| **Vendor neutral** | No (SONiC-specific) | No (SONiC-specific) | Yes |
| **Best for** | Direct DB queries, debugging | SONiC-native automation | Multi-vendor automation |

---

### 2.7 gNMI SET — Making Configuration Changes

gNMI is not just for reading — you can also push configuration changes using the SET RPC. Below is an example that changes the MTU on an interface.

**Update MTU via sonic-db:**

```bash
gnmic -a localhost:50051 \
  -u admin -p password \
  --insecure \
  set \
  --update-path PORT/Ethernet0 \
  --update-value '{"mtu": "1500"}' \
  --target CONFIG_DB
```

**Update MTU via OpenConfig:**

```bash
gnmic -a localhost:50051 \
  -u admin -p password \
  --insecure \
  set \
  --update-path openconfig-interfaces:interfaces/interface[name=Ethernet0]/config/mtu \
  --update-value 1500
```

> **Important:** gNMI SET writes directly to CONFIG_DB. The change takes effect immediately but is not automatically saved to `/etc/sonic/config_db.json`. To persist, run `sudo config save -y` on the device.

---

## Summary

This lab covered two automation approaches for SONiC:

**Ansible** provides two methods of increasing sophistication:

| Method | Module | Mechanism |
|--------|--------|-----------|
| Shell commands | `ansible.builtin.shell` | Runs SONiC CLI commands via SSH |
| cisco.sonic collection | `cisco.sonic.sonic_config` | Structured CLI over network_cli with error handling |

**gNMI** provides programmatic access through three data models:

| Model | Target Audience | Key Advantage |
|-------|----------------|---------------|
| SONiC DB (`sonic-db`) | SONiC operators, debugging | Direct database access, familiar Redis schema |
| SONiC Native YANG | SONiC automation engineers | Typed data, YANG validation |
| OpenConfig | Multi-vendor environments | Vendor-neutral, config + state separation |

Both Ansible and gNMI can be combined: use Ansible for orchestrating multi-device workflows and gNMI for granular, real-time device interactions.

---
