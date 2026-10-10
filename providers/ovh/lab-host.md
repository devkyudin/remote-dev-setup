# OVHcloud Adapter: LAB Host

> Fills the slots of the generic runbook [`docs/runbooks/lab-host.md`](../../docs/runbooks/lab-host.md) for OVHcloud dedicated servers. For the slot convention, see [`providers/README.md`](../README.md).
>
> The reference deployment uses OVHcloud for **LAB**.
>
> Last updated: 2026-10-10.

Prices below are in EUR per month **excluding VAT**, as shown on the Irish (`en-ie`) site. Prices and stock change often: every figure carries the date it was checked, and you should re-check it in the order flow before paying.

## `order`

*Which offers at this provider pass the hard filters, what do they cost, which billing terms exist, and what must be re-checked in the order flow?*

### Offers that pass the hard filters

The reference deployment raised the RAM filter (F1) to **64 GB**, because 32 GB is tight from day one with an IDE backend and CI. The offers below pass that raised filter and the rest of F1–F6. All three include an out-of-band console and rescue mode, and allow a Proxmox VE install (see the unverified notes under [`oob`](#oob) and [`install`](#install)).

| Offer | CPU | RAM | Disks | Port | Price | Trade-offs |
|---|---|---|---|---|---|---|
| **Rise-S** ([source](https://www.ovhcloud.com/en-ie/bare-metal/rise/rise-s/)) | AMD Ryzen 7 9700X (Zen 5) | 64 GB DDR5, on-die ECC only | 2x512 GB NVMe | 1 Gbps, unmetered | 64.99 (checked 2026-10-10) | Best single-thread CPU in budget. No true ECC. No vRack (private network) on this model. |
| **RISE-1** ([source](https://www.ovhcloud.com/en-ie/bare-metal/rise/rise-1/)) | Intel Xeon-E 2386G | 64 GB ECC | 2x512 GB NVMe | 1 Gbps, unmetered | about 64.99 (checked 2026-10-10) | True ECC and vRack, but a noticeably slower CPU. With 2x960 GB NVMe (or RISE-2) it costs 85–93 (checked 2026-10-10), at or over budget including VAT. |
| **SYS-GAME-2** ([source](https://eco.ovhcloud.com/en-ie/)) | AMD Ryzen 7 3800X (Zen 2, 2019) | 64 GB ECC | 2x960 GB NVMe | 500 Mbps | 64.99 (checked 2026-10-10) | Same price, more disk and true ECC, but a CPU three generations older and half the port speed. |

All prices: OVHcloud catalog and product pages, checked 2026-10-10 (overview: [Rise range](https://www.ovhcloud.com/en-ie/bare-metal/rise/), [dedicated server prices](https://www.ovhcloud.com/en-ie/bare-metal/prices/)). Several providers raised dedicated server prices in 2026 because of RAM and NVMe costs. During research (2026-10-10) OVHcloud stated that the Rise range was not affected by its H2 2026 increase; re-check on the [prices page](https://www.ovhcloud.com/en-ie/bare-metal/prices/).

### Billing options

Checked 2026-10-10 for Rise-S, on the [Rise-S page](https://www.ovhcloud.com/en-ie/bare-metal/rise/rise-s/) and in the order flow:

| Term | Setup fee | Monthly | First-year cost |
|---|---|---|---|
| Month-to-month | one month's rent (64.99) | 64.99 | about 845 |
| 12 months prepaid | none | about 5% less | about 741 |

At 2026-10-10 prices, the prepay saves about 100 EUR in the first year but locks in the choice. See runbook step 6 for why the first host should prefer month-to-month.

### What to re-check in the order flow

- **Price** for the term you pick, and whether the page shows it with or without VAT (at 20% VAT, Rise-S was about 78 on 2026-10-10).
- **Stock per datacenter.** On 2026-10-10 Rise-S was in stock only in two datacenters in France ([source](https://www.ovhcloud.com/en-ie/bare-metal/rise/rise-s/), availability selector). Any EU datacenter is fine for the reference deployment, since latency differences between them were within noise.
- **Disk configuration:** two NVMe devices, not a single larger one. Some configurators offer storage upgrades that change the layout.
- **Port speed** and traffic: 1 Gbps unmetered on Rise.
- **Fallback** if Rise-S is out of stock: RISE-1 with 2x512 GB NVMe at about the same price.

### Reference deployment's choice

**Rise-S**, month-to-month (decided 2026-10-10). Why:

- the best single-thread CPU within budget, which matters most for a remote IDE backend and .NET builds;
- IPMI/KVM included, and rescue mode available;
- a Proxmox VE 9 install template exists ([OS list](https://www.ovhcloud.com/en-ie/bare-metal/os/));
- 1 Gbps unmetered traffic;
- a different provider from PROD, which removes the shared failure domain entirely.

Accepted trade-offs:

- no true ECC (DDR5 on-die ECC only): acceptable for a LAB that is "ok to break";
- no vRack: a second node (L9+) joins over the overlay network, or a different model is chosen for it;
- 512 GB usable: enough for the dev VM, CI caches, the platform with bounded retention, and VM templates.

Latency from the owner's usual location to all candidate EU datacenters (this provider and others) averaged 64–76 ms with similar jitter (measured 2026-10-10), so latency did not separate the candidates.

Order and delivery notes (2026-10-10): ordered month-to-month; the delivered host was checked in the control panel against every hard filter (CPU, RAM, two NVMe devices, traffic, IPMI/KVM and rescue mode) and passed.

## `oob`

**Not yet covered: L0b.**

*Unverified research notes (2026-10-10, [OVHcloud dedicated server docs](https://help.ovhcloud.com/csm/en-ie-documentation-bare-metal-cloud-dedicated-servers?id=kb_browse_cat)):*

- Rescue mode is available on all dedicated server ranges.
- IPMI/KVM is included on the Rise range.

## `install`

**Not yet covered: L0b.**

*Unverified research notes (2026-10-10, [OS list](https://www.ovhcloud.com/en-ie/bare-metal/os/)):*

- An official Proxmox VE 9 install template exists.
- Reports conflict on whether the template's RAID1 option gives a ZFS mirror or md-RAID with LVM.
- Fallback: install the official Proxmox VE ISO through the IPMI console.

## `network`

**Not yet covered: L0b.**

*Unverified research notes (2026-10-10, [OVHcloud dedicated server docs](https://help.ovhcloud.com/csm/en-ie-documentation-bare-metal-cloud-dedicated-servers?id=kb_browse_cat)):*

- An additional IPv4 costs 1.99 EUR/month (checked 2026-10-10) and needs a virtual MAC per IP when bridged.
- NAT behind the main IP needs neither an additional IP nor a virtual MAC.
- A /64 IPv6 block is included.

## `edge-firewall`

**Not yet covered: L0b.**

*Unverified research notes (2026-10-10, [OVHcloud dedicated server docs](https://help.ovhcloud.com/csm/en-ie-documentation-bare-metal-cloud-dedicated-servers?id=kb_browse_cat)):*

- A stateless network firewall, IPv4 only, up to 20 rules per IP.
- Anti-DDoS is always on.

## `recovery`

**Not yet covered: L0b.**

*Unverified research notes:* no notes beyond rescue mode and IPMI/KVM, see [`oob`](#oob).

## `private-network`

**Not yet covered: L9+.**

*Unverified research notes (2026-10-10, [vRack](https://www.ovhcloud.com/en-ie/network/vrack/)):*

- vRack (private network) is available on most Rise models, but not on Rise-S.
