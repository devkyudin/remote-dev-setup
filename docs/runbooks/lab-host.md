# Runbook: LAB Host

> How to choose, order and (from L0b) bootstrap the LAB host: the bare-metal machine that runs Proxmox for dev, CI, agents and the platform (see [architecture → LAB Host](../architecture.md#lab-host)).
>
> This runbook is **provider-agnostic**. Steps that differ by hosting provider are declared as named **slots**. Each provider adapter in [`providers/`](../../providers/README.md) answers them in `providers/<name>/lab-host.md`.
>
> Last updated: 2026-10-10.

## Contents

- [Selection](#selection): set a budget, filter, rank, measure latency, choose, order and accept the host.
- [Bootstrap](#bootstrap): skeleton of the provider slots and generic steps, written in L0b.

## Selection

You will end this section with a host ordered and checked, and the decision written down. Plan for one session of research plus the provider's delivery time.

### 1. Set a budget

Write down:

- the monthly range you are willing to pay (for example "50–100 per month");
- whether that range **includes VAT**. Providers show prices with or without VAT, depending on the page and your account type. Mixing the two makes offers look cheaper or more expensive than they are;
- whether you would pay a **setup fee** or **prepay** a longer term to get a lower price.

*Why:* without a fixed budget, every comparison drifts towards "a bit more for a bit better". A budget that states its VAT basis lets you compare offers directly.

To compare offers with different fees, use the cost of the first year:

```
first-year cost = setup fee + 12 x monthly price
```

### 2. Apply the hard filters

A hard filter is pass/fail. A candidate that fails **any** of them is dropped, however cheap it is. Collect the candidates from the providers you are considering, then drop every offer that fails:

| # | Filter | Passes when | Why |
|---|---|---|---|
| F1 | Hardware baseline | x86-64, at least 32 GB RAM, at least 8 CPU cores, at least 512 GB usable on **two NVMe devices** that you can mirror | One disk failure must not lose the host; HDD-only storage makes builds and Docker disk-bound. |
| F2 | Out-of-band access | A rescue system, KVM or IPMI console that works without the host's own OS or network | You will lock yourself out with a firewall or SSH mistake at least once; the console is how you get back in. |
| F3 | Hypervisor installable | Proxmox VE can be installed, from a provider image or from the official installer | Offers limited to the provider's own managed OS images cannot run the LAB. |
| F4 | VM networking with one public IP | VMs can reach the internet through **NAT** behind the host's single public IP without breaking the provider's network policy | Extra public IPs cost money and are not needed. Some providers block or flag traffic from MAC addresses they did not assign, so VMs must stay behind NAT and never appear on the provider network with their own MAC. |
| F5 | Traffic | Unmetered, or an included volume large enough for CI image pulls, platform telemetry and restore drills | Metered traffic turns every `docker pull` into a cost. |
| F6 | Independence from PROD | A different provider from the PROD host, or at least a separate account | See [step 5](#5-check-independence-from-the-prod-provider). |

You can **raise** a filter for your own workload, but not lower it. For example, if you plan to run an IDE backend and CI from day one, 32 GB RAM is tight and you might filter at 64 GB. Write down any raised filter together with the decision.

*Tip:* the hardware filters (F1) are on the offer page. F2–F5 are usually in the provider's documentation or FAQ; search it for "rescue", "IPMI", "KVM", "Proxmox", "MAC address" and "traffic".

### 3. Rank the remaining candidates

Sort the candidates that passed every filter by these criteria, **in this order**. A later criterion only matters when the earlier ones are close.

1. **Price**, using the first-year cost from step 1.
2. **Single-thread CPU performance.** Compare the CPU models on a public benchmark that publishes single-thread scores. *Why:* a remote IDE backend and incremental builds mostly wait on one core, so a newer CPU feels faster than more older cores.
3. **ECC memory.** True ECC protects against silent memory corruption. *Why it is not higher:* LAB is "ok to break", and important data (PROD backups) does not depend on it.
4. **RAM headroom** above your filter. More agent slots and staging environments later.
5. **Latency** from where you work, measured as in step 4. It is a tie-breaker, not a filter.

### 4. Measure latency

Latency decides how responsive a remote IDE feels. Measure it rather than guessing from a map.

1. For each candidate datacenter, find the provider's **test host**: a looking-glass or speed-test address that the provider publishes for this purpose. Do not measure against a random customer server.
2. From the machine and network you usually work on, run:

   ```bash
   ping -c 100 <test-host>
   ```

   ```bash
   mtr --report --report-cycles 100 <test-host>
   ```

   `ping` prints `min/avg/max/stddev` (on Linux, `mdev`) at the end. `avg` is the latency, and `stddev`/`mdev` is a good proxy for jitter. `mtr` shows where along the route loss or delay appears.
3. Repeat at **several times of day**: at least a working-hours run, an evening peak run and a quiet run (night or early morning).
4. Record the results in a table:

   | Datacenter | Time of day | Avg (ms) | Jitter (ms) | Loss |
   |---|---|---|---|---|
   | `<dc-1>` | morning | | | |
   | `<dc-1>` | evening | | | |
   | `<dc-2>` | morning | | | |

**Rules of thumb** for round-trip latency to a remote IDE. They are rough guides; jitter and packet loss hurt more than a stable average.

| Average RTT | Full remote IDE (backend on the host, thin client locally) | Editor over SSH (local UI, remote files and terminal) |
|---|---|---|
| under 30 ms | feels local | feels local |
| 30–60 ms | comfortable | comfortable |
| 60–100 ms | usable, typing lag is noticeable | comfortable |
| over 100 ms | frustrating | usable |

If all candidates land in the same band, latency cannot separate them; go back to price and hardware.

### 5. Check independence from the PROD provider

Look at where the PROD host runs.

- **Preferred:** the LAB host is at a **different provider**. An account suspension, a billing dispute or a provider-wide outage then cannot take down both.
- **Acceptable:** if the best candidate is at PROD's provider, take it only under a **separate account**, and write down why in the decision.

*Why:* the [architecture](../architecture.md#core-principles) keeps PROD independent of LAB. A shared account is a shared point of failure, even when the two hosts never talk to each other.

### 6. Choose a billing term

Providers usually offer some of:

- **month-to-month**, often with a setup fee;
- **prepaid terms** (for example 12 months), often without a setup fee or with a discount.

For your **first** LAB host, prefer a short term even if it costs a little more. *Why:* the bootstrap (L0b) may reveal a problem (an installer that will not mirror the disks, a console that does not work as documented) that makes switching worthwhile. Prepay once the host has proven itself.

### 7. Record the decision in the provider adapter

Write the decision into the `order` slot of `providers/<name>/lab-host.md` (create the adapter if it does not exist; see [`providers/README.md`](../../providers/README.md)):

- the offers at this provider that passed the hard filters, with their trade-offs;
- the billing options;
- what to re-check in the order flow;
- the offer you chose and why, including any raised filter from step 2.

Every price and stock statement gets the **date it was checked** and a **source link**. Do **not** record deployment values: no IPs, hostnames, account IDs or order numbers. They belong in the password manager.

### 8. Order and accept the host

1. Open the provider's order flow and re-check price, CPU, RAM, disk configuration (two NVMe devices), port speed and stock in the datacenter you want. If anything differs from the adapter, update the adapter (with the new date) first.
2. Place the order. Store the provider account credentials and any server access details in the password manager (bootstrap tier, see [Secrets by Tier](../architecture.md#secrets-by-tier)).
3. When the host is delivered, check it against **every hard filter** in the provider's control panel: CPU, RAM, two NVMe devices, traffic, and the out-of-band options.
4. If something does not match, raise it with the provider **before** any bootstrap work. Bootstrap does not start until it is resolved.

## Bootstrap

> **Skeleton.** The slots below are declared so adapters can fill them. The steps themselves are written in L0b (`lab-host-bootstrap`), on a real host.

### Slots

Each slot has a stable name and the question it must answer. The answers live in `providers/<name>/lab-host.md` under a heading with the slot's exact name. To find the adapter for your provider, see the index in [`providers/README.md`](../../providers/README.md).

| Slot | Question the adapter must answer | Used in |
|---|---|---|
| `order` | Which offers at this provider pass the hard filters, what do they cost, which billing terms exist, and what must be re-checked in the order flow? | [Selection, steps 7–8](#7-record-the-decision-in-the-provider-adapter) |
| `oob` | How do I reach a console or rescue system when the host is unreachable over the network, and how do I test that access before I need it? | Bootstrap step 1 |
| `install` | How do I install Proxmox VE on this host with the two NVMe devices mirrored: a provider template, or the official installer through the console? | Bootstrap step 2 |
| `network` | How is the host's public interface configured (IPv4 and IPv6), and how do VMs reach the internet through NAT without violating the provider's network policy, such as MAC filtering? | Bootstrap step 3 |
| `edge-firewall` | Does the provider offer a network firewall in front of the host, what can it filter, and how should it be configured alongside the host firewall? | Bootstrap step 6 |
| `recovery` | How do I boot a rescue system, reach the host's disks, and repair a broken boot, network or firewall configuration? How is a failed disk replaced? | Bootstrap step 7 |
| `private-network` | Can a second node join this host over a provider private network, or must it connect over the overlay network? | Bootstrap step 8 (L9+) |

### Steps

#### 1. Verify out-of-band access

Provider-specific: slot **`oob`** (see [`providers/README.md`](../../providers/README.md)). Test the console **before** you change anything on the host. *To be written in L0b.*

#### 2. Install Proxmox VE with mirrored NVMe

Provider-specific: slot **`install`** (see [`providers/README.md`](../../providers/README.md)). *To be written in L0b.*

#### 3. Configure host networking and NAT for VMs

Provider-specific: slot **`network`** (see [`providers/README.md`](../../providers/README.md)). *To be written in L0b.*

#### 4. Join the overlay network

Not provider-specific. *To be written in L0b.*

#### 5. Lock down the Proxmox web UI and SSH

Not provider-specific: make the web UI and API reachable only over the overlay network. *To be written in L0b.*

#### 6. Configure the edge firewall

Provider-specific: slot **`edge-firewall`** (see [`providers/README.md`](../../providers/README.md)). *To be written in L0b.*

#### 7. Practice recovery

Provider-specific: slot **`recovery`** (see [`providers/README.md`](../../providers/README.md)). Break something on purpose and repair it through the console. *To be written in L0b.*

#### 8. Private network for a second node

Provider-specific: slot **`private-network`** (see [`providers/README.md`](../../providers/README.md)). *To be written when a second node is added (L9+).*
