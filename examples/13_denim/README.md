# Example 13: DENIM (Dynamic Endpoint Network Impairment Model)

DENIM applies configurable impairments: ECN marking, drop, delay to RoCE v2
traffic **before it reaches the RDMA stack**. It lets network
research (e.g. congestion control in the context of RDMA) trigger deterministic, reproducible network events on a two-node setup
with no programmable switch in the path.

The impairment datapath lives in the shell's service layer, in series on the RX
stream between the CMAC and the IP handler, and ahead of the packet sniffer tap
so captures show traffic as the RDMA stack will actually see it. This example
is the **control plane**: a vFPGA exposing the rule table and counters, plus
`denim_ctl` to drive them.

With no rule armed, a DENIM shell is transparent apart from a few cycles of
store-and-forward latency.

## Rule grammar

```
<condition>[, <condition>...] : <effect>[, <effect>...]
```

Conditions are AND-ed and omitted fields are wildcards. The first matching rule
wins and **all** of its effects apply. Effects therefore chain *inside* a rule:
under first-match-wins, two rules with identical conditions would never both fire.

| | |
|---|---|
| Conditions | `qpn N`, `psn N`, `psn A-B`, `psn +K`, `src_ip a.b.c.d`, `dst_ip a.b.c.d`, `opcode 0xNN` |
| Effects | `ecn`, `drop`, `delay <t>` (`ns` or `us`), `count` |

```
qpn 3, psn 50-150   : delay 1us       # emulate queueing over a PSN window
src_ip 10.1.212.62  : ecn             # fake congestion signal per sender
qpn 2, psn +100     : drop            # single loss at a relative PSN
qpn 2               : count           # monitor only, no effect
qpn 4, psn 0-500    : ecn, delay 2us  # two effects on one packet
```

Match values are written as plain host integers, `src_ip 10.1.212.62` is
`0x0A01D43E`. The conversion to network byte order happens in hardware, at
field extraction, so nothing in software byte-swaps.

`psn +K` is relative to the connection's initial PSN.

There are **8 slots**. `count` exists for dry runs.

## Building

**Hardware**, on a build server (e.g. `hacc-build-*`):

```bash
module load vivado/2024.1
mkdir build_hw && cd build_hw
cmake ../examples/13_denim/hw -DFDEV_NAME=u55c
make project && make bitgen
```

**Software**, on the FPGA node itself:

```bash
cd examples/13_denim/sw
mkdir build_$(hostname -s) && cd build_$(hostname -s)
cmake .. && make -j
```

## Using it

```bash
./denim_ctl --status                              # counters and installed rules
./denim_ctl --load ../rules/example.rules         # install a rule set
./denim_ctl --slot 2 --rule "qpn 7 : drop"        # one slot, mid-experiment
./denim_ctl --clear 2                             # disarm one slot
./denim_ctl --clear                               # disarm all
./denim_ctl --global-en 0                         # disarm everything, keep the rules
./denim_ctl --clear-counters
```

`--load` parses the whole file before touching a register, so a typo on the
last line cannot leave half a rule set installed. It then drops `GLOBAL_EN`,
rewrites every slot, and re-enables. Each install is read back and compared against what was asked for.

`--nclk` must match the `NCLK_F` the bitstream was built with (250 MHz by
default), since it converts delay times into clock cycles.

`denim_ctl` checks the hardware's `VERSION` register at startup and refuses to run on a mismatch.

## Reading `--status`

```
slot     matched     overflow    rule
   0       14203            0    src_ip 10.1.212.61 : ecn
   1         101            0    qpn 3, psn 50-150 : delay 1us
```

- **matched**: packets that matched this slot, whatever its effects.
- **overflow**: packets this rule's delay caused to be dropped because the
  FIFO was full.

Overflow drops are counted separately from intentional drops, so `dropped`
rising means the experiment is working and `delay overflows` rising means it
is not.

`marked ECN` counts packets DENIM actually marked, so a packet that arrived
already CE does not increment it.

## Two-node experiment

Node A runs a plain shell (with RDMA enabled, however), node B the DENIM shell, with `09_perf_rdma` driving
traffic A → B. Claims can be validated using three independent witnesses:

1. DENIM's own counters, via `--status`
2. The stack's reaction: retransmissions and other metrics in `coyote_sysfs`,
   plus A's completion times
3. Ground-truth PCAP from the sniffer, which sits downstream of DENIM and so
   records the traffic as modified


## Limitations

IPv4 only, no VLAN tags, fixed 20-byte IPv4 headers, RX path only.
Non-conforming packets pass through unmatched.

Rules are stateless. "Drop the retransmission" or "drop every tenth packet"
would need per-connection state in hardware. Similar patterns are reachable
meanwhile by arming and disarming rules over time and by using PSN (relative) ranges.
