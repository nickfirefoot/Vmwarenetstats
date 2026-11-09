# VMware vSphere Network Statistics — management pack for VCF Operations

Per-interface network telemetry for vSphere, collected agentlessly through a single
read-only vCenter connection: physical uplinks, VMkernel interfaces (NSX TEPs
included) and VM vNICs. Independent of any other management pack: no shared
container image, no shared registry path, no version coupling.

## Current release: 2.0.3

`paks/VsphereNetStats_2.0.3.pak` ·
image `ghcr.io/nickfirefoot/vmwarenetstats@sha256:562c38a9aefe0cbf66775be7111b6b1ca1574d580587e39dcbe6a548cf8734c1` (public)

Validated on Aria Operations 8.18.7 and VCF Operations 9.1.1.

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

- **Rapid Network Physical NICs** — loss, wire errors, ring and posture, VMkernel by role, and vCenter's own counters for contrast
- **Rapid Network VM NICs** — loss by vNIC, adapters and networks, by role, throughput
- **Rapid Network VMkernel** — loss, throughput, and a dedicated **TEP panel** for NSX overlay performance

Each lists one row per host × interface or VM × vNIC for the selected cluster.

## Requirements

- Aria Operations 8.18 or VCF Operations 9.x, with a **Cloud Proxy** that can run
  container adapters and pull from `ghcr.io` **and** `pkg-containers.githubusercontent.com`
  (every image layer redirects there)
- One vCenter reachable on 443 with a **read-only** account
- No ESXi SSH, no host agent, no guest agent. Only `get` / `list` / `stats` esxcli
  verbs are invoked, enforced in code
- No Operations account to create: the adapter writes with the credential Operations
  injects into its instance

## Installing

1. **9.x only:** Admin UI → Administrator Settings → Security Settings →
   *Activate Unsigned PAK Installation*. (8.18 asks you to confirm instead.)
2. Operations: **Administration → Integrations → Repository → ADD**, select the `.pak`
3. Tick **both** *Install the PAK file even if it is already installed* and
   *Ignore the PAK file signature checking*; confirm the unsigned prompt; accept the EULA
4. Add an account: vCenter host, read-only user and password, the Cloud Proxy as
   collector; accept the vCenter certificate; **Start Collecting**

The first push lands within one collection cycle. Rates need two samples, so rate
and percentage columns fill from the second cycle (about five minutes later).
A blank cell is never a zero: where a driver refuses a read, the pack emits nothing.

## Upgrading from earlier versions

Install over the top. Versions before 2.0.3 created their own uplink / VMkernel /
vNIC objects; those stop updating and keep their history. The custom groups those
versions shipped are removed (a group cannot select a single interface on a host).

## Known limits

- VMs Operations does not model (vSphere Pods; on 8.18 also Supervisor control-plane
  VMs) have nowhere to receive vNIC data and are skipped.
- Pushed values cannot be deleted from Operations. When an interface disappears its
  metrics age out with retention; its last properties remain. `lastUpdated` tells
  you whether the rest is current.
- Air-gapped sites must mirror the image into a reachable registry; a container-free
  build is in progress.

## License

Apache-2.0 — see `LICENSE`.
