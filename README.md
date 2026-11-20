# Agave XDP: a practical guide by NIC

Field notes for Solana validator operators enabling XDP retransmit in Agave / Jito-Agave.
The official docs cover the flags. This guide covers what breaks on real hardware and how to fix it.

> **Status:** living document. Each NIC section says where its data comes from.
> **Verified** means tested on a mainnet validator by the maintainer or a contributor who sent `ethtool` output.
> **Reported** means collected from operator discussions and not yet independently verified.
> Please send your own results; see [Contributing](#contributing).

Maintained by [Dragon Trust](https://dragontrust.pro) validator · License: CC BY 4.0

---

## Contents

1. [TL;DR](#tldr)
2. [Before you start: collect facts](#before-you-start-collect-facts)
3. [NIC compatibility table](#nic-compatibility-table)
4. [NIC notes](#nic-notes)
5. [Ring sizes and the EINVAL error](#ring-sizes-and-the-einval-error)
6. [Making NIC settings survive a reboot](#making-nic-settings-survive-a-reboot)
7. [CPU cores, SMT and NUMA](#cpu-cores-smt-and-numa)
8. [Hugepages](#hugepages)
9. [Is XDP actually working? Counters to watch](#is-xdp-actually-working-counters-to-watch)
10. [Error → cause → fix](#error--cause--fix)
11. [Contributing](#contributing)

---

## TL;DR

- **Start in copy mode.** Leave out `--xdp-zero-copy`. Add it only after copy mode has run cleanly and only if your driver is known to support it. On Broadcom `bnxt_en`, copy mode is enough.
- **Set the NIC rings to a power of two** (for example `ethtool -G <if> rx 1024 tx 1024`). Otherwise the AF_XDP socket setup can fail with `EINVAL`.
- **Make ring and channel settings persistent.** `ethtool` changes are lost on reboot, and the validator will then fail to start XDP after the next restart.
- **Give XDP its own physical core** and keep everything else off it, including the core's SMT sibling.
- After the switch, **watch `ethtool -S`** for `xsk`/`xdp`/`alloc` errors, not just the validator log.

---

## Before you start: collect facts

Run these commands. They only read state and do not affect a running node.

```bash
IF=enp2s0f1np1            # your public interface
uname -r
ethtool -i $IF            # driver + firmware
ethtool -g $IF            # ring sizes: max and current
ethtool -l $IF            # channels (queues)
ethtool -S $IF | grep -iE 'xsk|xdp|alloc|drop|err' | grep -v ': 0$'
lscpu | grep -iE 'model name|numa|^CPU\(s\)'
cat /sys/class/net/$IF/device/numa_node
grep -i huge /proc/meminfo
```

Save the output. You will compare against it after enabling XDP, and it is exactly what a contribution to this guide needs.

---

## NIC compatibility table

| Driver | Typical NICs | Copy mode | Zero-copy | Notes | Source |
|---|---|---|---|---|---|
| `bnxt_en` | Broadcom BCM57502 / 57508 (NetXtreme-E) | ✅ works | ❌ `EOPNOTSUPP` on kernel 6.8 | Rings must be a power of two (1024/1024 tested) | **Verified** (Dragon Trust, mainnet, Jito-Agave 4.4.1) |
| `i40e` | Intel X710 / XL710 | ✅ | ✅ reported working | `rx_xsk_buff_alloc_err` grows → reduce channels | Reported |
| `ixgbe` | Intel X540 / X550 / 82599 | ✅ | ❌ network freeze with zero-copy on kernel 7.0 | No known fix yet; stay in copy mode | Reported |
| `ice` | Intel E810 | ✅ reported | reported working | Needs more data | Reported |
| `mlx5_core` | NVIDIA/Mellanox ConnectX-5/6/7 | ✅ reported | reported working | Needs more data | Reported |

Do you have one of these cards, or a card that is not listed? Please send your output.

---

## NIC notes

### Broadcom `bnxt_en` (BCM57502). Verified

Test system:

| | |
|---|---|
| CPU | AMD Ryzen 9 9950X (16C/32T), single NUMA node |
| Kernel | 6.8 series (Ubuntu 24.04) |
| Firmware | 231.0.154.0 |
| Ring maximums | RX 8191 / TX 2047 |
| Ring setting used | **RX 1024 / TX 1024** |
| Channels | 8 combined (maximum 128) |
| Hugepages | 0 reserved |
| Mode | copy mode, one dedicated XDP core |
| Client | Jito-Agave 4.4.1 |
| Uptime at time of writing | 2+ weeks without XDP errors |

Validator flags used (other flags omitted):

```
--xdp-cpu-cores 1
```

What happened:

1. With the default rings the validator panicked at startup:
   ```
   WARN  agave_xdp::tx_loop] huge page alloc failed, falling back to regular page size
   ERROR agave_xdp::tx_loop] failed to create AF_XDP TX socket for queue QueueId(0) on CPU 1:
         setsockopt(SOL_XDP, ring=6, size=1022) failed: Invalid argument (os error 22)
   panicked at validator/src/commands/run/execute.rs:410:30: failed to create xdp transmitter
   ```
   Setting rings to 1024/1024 fixed it. See [Ring sizes](#ring-sizes-and-the-einval-error).
2. With `--xdp-zero-copy` the validator panicked with a different error:
   ```
   bind(AF_XDP, ifindex=3, queue=0, flags=0xc) failed: Operation not supported (os error 95)
   ```
   `flags=0xc` means `XDP_ZEROCOPY | XDP_USE_NEED_WAKEUP`. `EOPNOTSUPP` means the driver refuses an AF_XDP zero-copy bind, so zero-copy is not available on `bnxt_en` with this kernel. Remove the flag. Copy mode runs stably, and the `ethtool -S` XDP/XSK error counters stay at zero.
3. `rx_l4_csum_errors` is non-zero: 2688 out of about 48.8 billion received packets, roughly 1 in 18 million. This counter is not related to XDP. It counts incoming packets whose UDP/TCP checksum failed hardware validation, which is normal background noise on a public internet interface. It is only a concern if it grows quickly or follows your own traffic.

### Intel `i40e` (X710 / XL710). Reported

- Symptom: `rx_xsk_buff_alloc_err` in `ethtool -S` keeps growing.
- Reported fix: reduce the number of combined channels, for example `ethtool -L <if> combined 4`. The AF_XDP fill ring then has to feed fewer queues.

### Intel `ixgbe` (X540). Reported

- With `--xdp-zero-copy` on kernel 7.0 (Ubuntu 26.04), the whole interface freezes and the node drops off the network.
- No fix known as of November 2026. Use copy mode.

---

## Ring sizes and the EINVAL error

AF_XDP rings (fill, completion, RX, TX) **must be a power of two**. The error

```
setsockopt(SOL_XDP, ring=6, size=1022) EINVAL
```

refers to socket option 6, `XDP_UMEM_COMPLETION_RING`. The size Agave requested was 1022, which is not a power of two.
Agave appears to size its XDP rings from the NIC's current ring size. A NIC left at a vendor default that is not a power of two therefore produces an invalid request.
The failure happens at startup and the validator panics (`failed to create xdp transmitter`). It cannot run in a broken state.

Fix:

```bash
ethtool -G <if> rx 1024 tx 1024
ethtool -g <if>     # confirm "Current hardware settings"
```

Notes:

- Stay within the "Pre-set maximums" from `ethtool -g`. On `bnxt_en` TX tops out at 2047, so the largest power of two for TX is 1024.
- On most drivers, changing rings briefly resets the link. Do it before the validator starts, not while it is voting.
- Bigger is not automatically better. 1024/1024 is a proven value. Change it only if you see drops.

---

## Making NIC settings survive a reboot

`ethtool -G` and `ethtool -L` are lost on reboot, and netplan cannot set rings.
The simplest reliable option is a oneshot systemd unit that runs before the validator. A ready-made file is in [`systemd/xdp-nic-tuning.service`](systemd/xdp-nic-tuning.service).

```bash
sudo cp systemd/xdp-nic-tuning.service /etc/systemd/system/
sudo sed -i 's/enp2s0f1np1/<your-if>/g' /etc/systemd/system/xdp-nic-tuning.service
sudo systemctl daemon-reload
sudo systemctl enable xdp-nic-tuning.service
# test without rebooting (resets the link briefly, so only do this while the validator is stopped)
sudo systemctl start xdp-nic-tuning.service && ethtool -g <your-if>
```

The unit is ordered `Before=solana.service`. If your validator unit has a different name, change that line.

---

## CPU cores, SMT and NUMA

`--xdp-cpu-cores` pins the XDP retransmit thread to the listed core(s). That core should do nothing else.

1. **Find the SMT sibling** of the core you chose:
   ```bash
   cat /sys/devices/system/cpu/cpu1/topology/thread_siblings_list
   # e.g. "1,17" on Ryzen: core 17 shares the same physical core
   ```
   Keep busy threads off the sibling as well. Otherwise the XDP thread effectively gets half a core.
2. **Do not combine `isolcpus` with Agave's own pinning blindly.** Operators report that `isolcpus` breaks PoH core pinning. If you isolate cores, check that the PoH and XDP threads actually land where you expect (`ps -L -o tid,psr,comm -p $(pgrep -f agave-validator)`).
3. **Keep the XDP core, the PoH core and the NIC IRQs apart.** Check where the NIC queue interrupts go:
   ```bash
   grep <your-if> /proc/interrupts
   ```
4. **NUMA** matters only on multi-socket or multi-NUMA systems (dual EPYC/Xeon, or EPYC with NPS>1). Pick an XDP core on the same node as the NIC:
   ```bash
   cat /sys/class/net/<if>/device/numa_node
   lscpu | grep NUMA
   ```
   A single-socket Ryzen has one node, so you can skip this step.

---

## Hugepages

Operators report the startup warning `huge page alloc failed` and ask how many hugepages XDP needs.

The full message is `huge page alloc failed, falling back to regular page size`. It is a **WARN**, not an error: Agave uses regular pages instead.
What we observed: on `bnxt_en` in copy mode, XDP has run stably for weeks with **`HugePages_Total: 0`**. When the validator crashes right after this line, the real cause is the next ERROR line, not hugepages. Reserve hugepages only if you can measure a benefit, and report your numbers.

---

## Is XDP actually working? Counters to watch

After enabling XDP:

Agave (4.4.x) uses AF_XDP for the **TX (retransmit) path**. `ip -d link show <if>` may therefore show no XDP program, and that is expected. The reliable sign is this startup line in the validator log (the file passed to `--log`, not necessarily journald):

```
INFO  agave_xdp::tx_loop] starting xdp loop on enp2s0f1np1 queue QueueId(0) cpu 1
```

with no `ERROR agave_xdp` line after it. If XDP setup fails, the validator panics, so a running node with that line means XDP is active.

```bash
# 1. startup line in the validator log
grep -E 'agave_xdp' /path/to/validator.log | tail

# 2. driver counters: these should stay flat
watch -n 5 "ethtool -S <if> | grep -iE 'xsk|xdp|alloc_err|drop' | grep -v ': 0$'"
```

| Counter (name varies by driver) | Meaning | Action |
|---|---|---|
| `*xsk_buff_alloc_err*` | The AF_XDP fill ring ran dry | Reduce channels; check XDP core load |
| `*xdp_drop*`, `*xdp_tx_err*` | Packets dropped on the XDP path | Check ring sizes and the zero-copy setting |
| `rx_*_discards`, `rx_drops` | The NIC dropped packets before software saw them | Increase RX ring; check IRQ/core layout |
| `rx_l4_csum_errors` | Bad checksums from the internet | Usually ignore; compare to total packets |

---

## Error → cause → fix

| Error / symptom | Likely cause | Fix | Source |
|---|---|---|---|
| `setsockopt(SOL_XDP, ring=6, size=1022) EINVAL` | NIC ring size is not a power of two, so the derived AF_XDP ring size is invalid | `ethtool -G <if> rx 1024 tx 1024`, then make it persistent | Verified (bnxt_en) |
| `bind(AF_XDP, ..., flags=0xc) failed: Operation not supported (os error 95)` | Zero-copy requested (`--xdp-zero-copy`) but the driver does not support AF_XDP zero-copy | Remove `--xdp-zero-copy` | Verified (bnxt_en, kernel 6.8) |
| XDP worked, then broke after a reboot | `ethtool` settings were not persistent | [systemd unit](#making-nic-settings-survive-a-reboot) | Verified |
| `rx_xsk_buff_alloc_err` growing (i40e) | Too many queues for the fill ring | `ethtool -L <if> combined 4` (or fewer) | Reported |
| Whole NIC freezes with zero-copy (ixgbe X540, kernel 7.0) | Driver bug | Copy mode; no fix known yet | Reported |
| `WARN huge page alloc failed, falling back to regular page size` | No hugepages reserved | Harmless; look at the next ERROR line instead | Verified (bnxt_en) |
| `ip -d link` shows no XDP program | Agave uses AF_XDP for TX only | Expected; check the `starting xdp loop` log line | Verified |
| PoH pinning ignored after adding `isolcpus` | Conflict between `isolcpus` and Agave thread pinning | Drop `isolcpus` or check thread placement | Reported |

---

## Contributing

Open an issue or PR with:

1. The output of the commands in [Before you start](#before-you-start-collect-facts). Remove IPs if you like.
2. Your Agave / Jito-Agave version.
3. Whether you run copy mode or zero-copy, and your `--xdp-*` flags.
4. What failed and what fixed it, with exact error text.

Use [`NIC_REPORT_TEMPLATE.md`](NIC_REPORT_TEMPLATE.md). Verified reports move a NIC from **Reported** to **Verified** in the table.

Verified findings will also be proposed to the network-compatibility list at solanahcl.org.
