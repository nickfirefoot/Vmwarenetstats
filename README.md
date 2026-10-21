# VMware vSphere Network Statistics — management pack for VCF Operations

Generic per-interface network telemetry for vSphere, collected agentlessly through a
single read-only vCenter connection. Independent of any other management pack: no
shared container image, no shared registry path, no version coupling.

## Status — read this first

**Version 1.8.0 carries a real image digest and will collect** — once the GHCR
package is made public.

| | |
|---|---|
| schema | 3 resource kinds, **68 metrics, 36 properties** |
| content | **15 views, 3 dashboards** (2 primary, scoped by cluster) |
| image | `ghcr.io/nickfirefoot/vmwarenetstats@sha256:35515a2b…` — pushed |
| dry run | 82 objects, **82 parent links**, 1,396 metric values, 11 s / 4 hosts |

**One step remains, and it is not in the pak:** the GHCR package is created
*private* by default, and a Cloud Proxy pulls the digest **anonymously**. Until it
is public the adapter installs and silently cannot collect, with an error that
reads like a management-pack fault rather than a permissions one.

> `https://github.com/users/nickfirefoot/packages/container/vmwarenetstats/settings`
> → *Change visibility* → **Public**

**The first collection cycle will look half-empty and that is correct.** Measured:
443 metric values cold, 1,396 warm. Rates need two samples, and this pack emits
*nothing* rather than a zero when it has one. Wait one further cycle before judging
any rate column empty.

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

## Fixed in 1.8.0

An adversarial pre-install review of 1.4.1 found four defects that would have
made the first install misleading. All four are fixed and verified:

| | |
|---|---|
| **Parent relationships were never sent.** The HostSystem parent object was built free-standing and never added to the result, so the payload carried `"relationships": []` while every object's local parent set looked correct. Measured: 80 local parents, **0** links on the wire. Every dashboard selector would have reached nothing. | fixed, verified 80/80 on the wire |
| **Dashboard ids were `uuid4`**, regenerated on every build, so a second install would have stacked duplicate dashboards instead of replacing them. | fixed to `uuid5`; a determinism test now gates it |
| **The cluster selector used `Configuration|Name`**, which is the UI attribute-picker's display path and is neither a property nor a statkey. The real key is `config|name`. The selector drives all three dashboards. | fixed and verified against the live API |
| **`connectFailed24h` emitted a constant `0`** on every vNIC while nothing ever queried the event it claimed to count. | metric removed rather than left declared-and-empty |

## Known limitations

Documented honestly rather than discovered later:

- **Collection does not work in 1.8.0.** See Status above.
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
