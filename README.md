# VMware vSphere Network Statistics — management pack for VCF Operations

Per-interface network telemetry for vSphere, collected agentlessly through a single
read-only vCenter connection: physical uplinks, VMkernel interfaces (NSX TEPs
included) and VM vNICs. Independent of any other management pack.

## Current release: 3.0.0

`paks/VsphereNetStats_3.0.0.pak` (build 3.0.0.6) ·
sha256 `aa177f734c2fc9093a5ba8ef1890f971ec02db5aa6f188555749c664658e9b18`

Validated on Aria Operations 8.18.7 and VCF Operations 9.1.1.

**3.0.0 runs inside the collector itself.** There is no container image, no
registry and no outbound internet access to arrange: the pak is the whole
product, so it installs on air-gapped sites as is. It collects the same
metrics and properties as 2.0.3, under the same keys.

## Where the data lands

The pack writes its values onto **vCenter's own objects** in Operations, under the
prefix `pcktpshr`, so selecting a vCenter cluster scopes every dashboard natively:

| vCenter object | keys |
|---|---|
| HostSystem | `pcktpshr\|Uplink:<vmnic>\|<metric>` · `pcktpshr\|VMkernel:<vmk>\|<metric>` · `pcktpshr\|TEP:<vmk>\|<metric>` (NSX tunnel endpoints) |
| VirtualMachine | `pcktpshr\|vNIC:<device key>\|<metric>` |
| both | `pcktpshr\|Collector\|lastUpdated` — when the pack last wrote there |

The pack sends **no relationships** and writes **no key outside `pcktpshr`**, so
vCenter's own metrics, properties and structure are never modified.

## What it collects

| Interface | Highlights |
|---|---|
| Physical uplink | rx-missed / ring overrun, CRC, frame, length, FIFO, over, carrier and abort errors; drops and loss ratios; ring size vs maximum; link state, flaps, negotiated vs capable speed; driver, firmware, pause and coalescing posture |
| VMkernel interface | per-netstack drops, firewall-filter drops / errors / queue, MTU vs its uplink, ECN and congestion control, role (vSAN, vMotion, TEP, …) |
| VM vNIC | rx/tx drops and loss ratios, broadcast/multicast/unicast rates, adapter type and class, disconnected, bandwidth limit, buffer settings |

The design target is **grey-state** NIC failure: interfaces that are up, passing
traffic, and quietly losing frames. The headline counter is ring overrun
(`rxMissedRate`), which vCenter's built-in error counters do not report.

## Dashboards

All three are in the **vSphere Network Statistics** folder:

- **Rapid Network Physical NICs** — loss, wire errors, ring and posture, VMkernel by role, and vCenter's own counters for contrast
- **Rapid Network VM NICs** — loss by vNIC, adapters and networks, by role, throughput
- **Rapid Network VMkernel** — loss, throughput, and a dedicated **TEP panel** for NSX overlay performance

Each lists one row per host × interface or VM × vNIC for the selected cluster.
The loss columns show each interface's worst value of the last 7 days, so a
problem stays visible at the top after it has passed.

## Requirements

- Aria Operations 8.18 or VCF Operations 9.x, with a **Cloud Proxy** (or collector) that can reach vCenter on 443
- One vCenter account with **read-only** access
- No ESXi SSH, no host agent, no guest agent. Only `get` / `list` / `stats` esxcli
  verbs are invoked, enforced in code
- No Operations account to create: the adapter writes with the credential Operations
  gives its own instance

## Installing

1. Operations: **Administration → Integrations → Repository → ADD**, select the `.pak`
2. Tick **both** *Install the PAK file even if it is already installed* and
   *Ignore the PAK file signature checking*; confirm the unsigned prompt; accept the EULA.
   On 9.x, if the Repository refuses an unsigned pak, first enable
   Admin UI → Administrator Settings → Security Settings → *Activate Unsigned PAK Installation*.
3. Add an account of type **vSphere Network Statistics**:

   | field | value |
   |---|---|
   | vCenter Server (FQDN) | the name on vCenter's certificate |
   | Port | 443 |
   | Credential | the read-only vCenter user and password |
   | Allow Insecure TLS | leave `false`; accept vCenter's certificate when prompted |
   | Collect VMkernel interfaces | `1` (yes) |
   | Collect virtual NICs | `all` |
   | Collector | the Cloud Proxy that reaches vCenter |

4. Accept the vCenter certificate, save, and make sure the account shows
   **collecting**. If it shows stopped after the certificate step, choose
   *Start Collecting*.

The first push lands within one collection cycle. Rates need two samples, so rate
and percentage columns fill from the second cycle (about five minutes later).
A blank cell is never a zero: where a driver refuses a read, the pack emits nothing.

**Right after an install the dashboards can show "The widget is not configured"
with an hourglass on every widget for up to 15 minutes** while Operations binds
them to their views. Nothing needs doing; reload after that.

## Upgrading from 2.x

3.0.0 is a new adapter, not an in-place upgrade of the 2.x container adapter:

1. Remove the 2.x account and uninstall the 2.x pack
2. Install 3.0.0 and add its account as above

Do not run 2.x and 3.0.0 against the same vCenter: both write the same keys.
Because the keys are unchanged, history continues on the same host and VM
objects across the switch.

## Scale

Measured on a lab vCenter (3 hosts, 12 uplinks, 15 VMkernel interfaces,
45 vNICs): a cycle takes **1–3 s** with about 160 vCenter calls and under
0.4 MB on the wire. Up to 4 hosts are collected at a time, and slow-changing
reads (ring size, driver and firmware, pause, coalescing, netstack settings)
rotate over 3 cycles.

Projected, not measured: 400 hosts at about 55–80 s per cycle, inside a
5-minute collection interval. The multi-host rate and the Operations side
have not been measured beyond the lab.

## Known limits

- VMs Operations does not model (vSphere Pods; on 8.18 also Supervisor control-plane
  VMs) have nowhere to receive vNIC data and are skipped. VM objects Operations
  still lists after the VM was deleted from vCenter receive nothing.
- Pushed values cannot be deleted from Operations. When an interface disappears its
  metrics age out with retention; its last properties remain. `lastUpdated` tells
  you whether the rest is current.
- A rotated property (ring size, driver, firmware, pause, coalescing, ECN) can be
  up to 10 minutes older than `lastUpdated` says.

## Earlier releases

`paks/VsphereNetStats_2.0.3.pak` and earlier remain for sites on 2.x. They need
the container image `ghcr.io/nickfirefoot/vmwarenetstats` and a Cloud Proxy that
can pull from `ghcr.io` and `pkg-containers.githubusercontent.com`.

## License

Apache-2.0 — see `LICENSE`.
