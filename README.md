# VMware vSphere Network Statistics — management pack for VCF Operations

Generic per-interface network telemetry for vSphere, collected agentlessly through a
single read-only vCenter connection. Independent of any other management pack: no
shared container image, no shared registry path, no version coupling.

## Status — read this first

**Version 1.4.1 is a schema and content release. It does not collect yet.**

The pak is built with a placeholder image digest, so:

| Installs and works | Does not work |
|---|---|
| Schema — 3 resource kinds, 46 metrics, 32 properties | Collection |
| 3 dashboards, 11 views | Any data in the panels |

Every panel will be empty until a real adapter image is published and the pak is
rebuilt against its digest. That is deliberate: install and image pull are separate
phases in Operations, so this release proves the schema and screens land correctly
before the container path is settled.

## What it collects (once collecting)

| Kind | Identifiers | Subject |
|---|---|---|
| `NetUplink` | host, vmnic | physical uplinks — driver error breakdown, ring config, link state |
| `NetVmkernel` | host, vmk | VMkernel ports, per netstack — drops, filter drops, MTU coherence |
| `NetVnic` | vm_uuid, vnic_key | virtual NICs — the inbound drop counters Operations declares but never fills |

The design target is **grey-state** NIC failure: interfaces that are up, passing
traffic, and quietly losing frames. The headline counter is `rxMissedRate`
(ring overrun), which in the reference lab reads 24,225 lifetime on one uplink while
every built-in vCenter error counter for the same uplink reads zero.

## Requirements

- VCF Operations 9.x (`vcops_minimum_version` 8.10.0)
- One vCenter, reachable on 443, with a **read-only** account
- No ESXi SSH, no host agent, no guest agent

Only `get` / `list` / `stats` esxcli verbs are ever invoked, enforced in code rather
than by convention.

## Installing

1. In Operations: **Administration → Integrations → Repository → ADD**
2. Select the `.pak` from `paks/`
3. Tick **both** boxes:
   - *Install the PAK file even if it is already installed*
   - *Ignore the PAK file signature checking*

This pak is **unsigned** — Broadcom signing is not available here, so the second box
is mandatory. You are told up front rather than discovering it mid-install.

If a panel shows *"Widget is not configured"* with an hourglass, the content import is
still running. Wait for it; nothing has failed.

## Known limitations

Documented honestly rather than discovered later:

- **Collection does not work in 1.4.1.** See Status above.
- **Mellanox ring configuration is unavailable.** `nmlx5_core` does not expose ring
  size through the VMkernel Sysinfo Interface, so `esxcli network nic ring current get`
  returns *Not supported*. Broadcom's own `nicinfo.sh` uses the same command and hits
  the same wall. Reported as "not reported by nmlx5_core", never as a zero.
- **Vendor private statistics are unreachable.** They exist, as
  `localcli --plugin-dir /usr/lib/vmware/esxcli/int networkinternal nic privstats get`,
  but that plugin directory is not registered with hostd and `localcli` bypasses hostd
  entirely. No agentless path reaches it.
- **vmxnet3 ring depth has no API.** A guest receive-buffer overrun is detectable at the
  port drop counter; the ring depth that caused it is `vsish`-only.
- **Colour bands are not carried in the view files.** Set them in the UI per column.

## Licence

Apache 2.0 — see `LICENSE`.
