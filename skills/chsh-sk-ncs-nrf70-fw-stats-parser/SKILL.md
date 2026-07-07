---
name: chsh-sk-ncs-nrf70-fw-stats-parser
description: Use when parsing an nRF70 FW stats `.bin` blob, or interpreting rpu_sys_fw_stats (PHY/LMAC/UMAC counters) to diagnose Wi-Fi disconnects — reason 34/low-ACK, reason 6, RPU lockups, link quality, or channel congestion on nRF7002.
---

# chsh-sk-ncs-nrf70-fw-stats-parser

How to parse an nRF70 `rpu_sys_fw_stats` binary blob and read the counters to
diagnose Wi-Fi link problems (disconnect reason codes, RPU lockups, congestion).

The stats are the **device-side ground truth** the supplicant log can't give you:
RSSI, PHY CRC error rates, TX timeouts, and the RPU hardware-lockup counter.

---

## Parse the blob

```bash
HDR=/opt/nordic/ncs/v3.3.0/modules/lib/nrf_wifi/fw_if/umac_if/inc/fw/host_rpu_sys_if.h
python3 nordic-wifi-memfault/script/nrf70_fw_stats_parser.py "$HDR" <stats>.bin
```

- The parser reads struct layouts **from the header**, so the header must match
  the firmware that produced the blob. Use the `nrf_wifi` module header above
  (the **nrf70** path). Do **not** use the `nrf/drivers/wifi/nrf71/...` header —
  that's a different chip and the offsets differ.
- Blob is little-endian, packed. The script hard-codes `<` endianness.
- Blob ships as a Memfault custom data recording / attached `.bin` in the device
  log folder (e.g. `<serial>_nrf70-fw-stats_<date>.bin`).

> **Counters are cumulative since the last boot/RPU reset, not per-event.** A
> blob captured hours after a disconnect cannot prove the disconnect's cause by
> itself. To pin timing, capture stats again right after the *next* event and
> diff the counters that moved (esp. `rpu_hw_lockup_count`, `reset_cmd_cnt`).

---

## Fields that matter for disconnect diagnosis

Ignore most counters. These are the ones that decide a diagnosis:

### PHY (`rpu_phy_stats`) — link quality & channel
| Field | Reads as |
|-------|----------|
| `rssi_avg` (dBm) | **Rules signal in/out.** > −60 = strong; a disconnect at −48 is **not** range/weak-signal. < −75 = marginal link. |
| `ofdm_crc32_fail_cnt` vs `ofdm_crc32_pass_cnt` | High fail % on the high-rate (OFDM/11g/n) path = **noisy/congested channel** (interference, co-channel contention). |
| `dsss_crc32_fail_cnt` vs `dsss_crc32_pass_cnt` | Low-rate 11b path. If DSSS is clean but OFDM fails a lot → classic congestion signature (robust rates survive, fast rates don't). |

### LMAC (`rpu_lmac_stats`) — TX/RX health & lockups
| Field | Reads as |
|-------|----------|
| `rpu_hw_lockup_count` / `rpu_hw_lockup_recovery_done` | **Smoking gun for device-side stalls.** Each lockup = a window where the radio answered nobody → AP sees missed ACKs → can trigger reason-34 low-ACK disassoc. Non-zero is always worth escalating. |
| `tx_timeout` vs `tx_pkt_cnt` | TX attempts that timed out at LMAC. High ratio = device repeatedly couldn't get frames out (medium contention / no response). |
| `reset_cmd_cnt` / `reset_complete_event_cnt` | RPU recovery resets. Usually pairs with a lockup. |
| `rx_mpdu_crc_fail_cnt` vs `rx_mpdu_crc_success_cnt` | Aggregate downlink corruption — corroborates the PHY OFDM picture. |

### UMAC TX/RX (`umac_tx_dbg_params` / `umac_rx_dbg_params`) — who deauth'd
| Field | Reads as |
|-------|----------|
| `tx_packet_deauth_count` / `tx_packet_disassoc_count` | Frames the **device sent**. Both 0 → device did **not** initiate the disconnect. |
| `rx_packet_deauth_count` / `rx_packet_disassoc_count` | Deauth/disassoc frames the **device received** → **AP-initiated**. Count should roughly match the number of reported disconnects. |

> **Do NOT trust the "UMAC control path stats" section** (the `cmd_*` /
> `event_*` block at the end of the parser output). Its struct alignment is
> wrong in the parser — values like `cmd_set_wiphy: 3440128` or a giant
> `CURR_STATE` are artifacts, not real counts. PHY / LMAC / UMAC-TX / UMAC-RX
> sections parse correctly; the control-path block does not.

---

## Disconnect reason codes (IEEE 802.11) seen on this platform

| Reason | Name | Meaning & where to look |
|--------|------|-------------------------|
| 6 | `CLASS2_FRAME_FROM_NONAUTH_STA` | AP got a data frame from a STA it thinks isn't associated. AP-side: its STA table was cleared (AP reboot, radio restart, or aging timer) while supplicant still believed it was COMPLETED. Device only learns on next TX. |
| 34 | `DISASSOC_LOW_ACK` | AP retransmitted downlink frames and got **no ACKs**, hit its retry limit, kicked the STA. Device-side or RF cause. Check: `rssi_avg` (rule out range), OFDM CRC fail % + `tx_timeout` (congestion), and `rpu_hw_lockup_count` (radio stalled). |
| 1 | `UNSPECIFIED` | Often the local supplicant's own teardown line that follows the real AP reason — read the `CTRL-EVENT-DISCONNECTED reason=` line, not the `net_event_mgmt` "Reason: Unspecified" line. |

`freq=` in the supplicant log gives the channel: 2412→ch1 … 2462→ch11 (2.4 GHz).

---

## Worked diagnosis (reason 34, F4CE3600230A, nRF7002DK, 2026-06-30)

Customer reported repeated AP-side disconnects in a busy lab. Reason 6 once,
then reason 34 twice. Stats blob showed:

- `rssi_avg = −48` → strong link, **range ruled out**.
- OFDM CRC fail **10.7%** (2.91M/27.2M), DSSS fail **0.32%** → **congested 2.4 GHz
  ch 11** (busy lab, interference).
- `tx_timeout = 3464` of `tx_pkt_cnt = 13437` → heavy medium contention.
- **`rpu_hw_lockup_count = 1`, `rpu_hw_lockup_recovery_done = 1`** → the RPU
  locked up once and self-recovered — a window of total radio silence that lines
  up exactly with the reason-34 low-ACK mechanism.
- `tx_packet_deauth/disassoc = 0`, `rx_packet_deauth = 2` → **AP-initiated** both
  times (device sent none, received two).

**Conclusion:** two overlapping causes — congested channel (intermittent missed
ACKs) **and** an RPU lockup (hard stall). Next steps: re-capture stats after the
next event to see if `rpu_hw_lockup_count` increments at the disconnect (proves
lockup is the trigger → escalate to dev team); move AP to a quiet channel to test
the congestion hypothesis; sniffer at the AP to see the unACK'd retries.

See wiki `concepts/wifi-debugging-patterns.md` (reason-34 pattern) for the
narrative version.

---

## Gotchas
- **Wrong header → garbage offsets.** Using the nrf71 header (or a mismatched
  firmware version) silently produces plausible-looking but wrong numbers. Always
  use the `nrf_wifi` module `host_rpu_sys_if.h` matching the running firmware.
- **Control-path section is misaligned** — see the warning box above. Don't quote
  `cmd_*` values to anyone.
- **Cumulative counters ≠ event proof.** A single post-hoc blob shows accumulated
  state, not what happened at the disconnect instant. One lockup count + one
  disconnect is *consistent with*, not *proof of*, lockup-as-cause — say so, and
  re-capture to confirm.

## Self-Update Policy

At the **end of each conversation**, review what was discovered and check
whether any facts in this skill are new, corrected, or outdated (e.g. new fields
worth interpreting, parser alignment fixes, additional reason codes seen, or
header-path changes across NCS versions).

If updates are warranted:
1. Collect all proposed changes with a brief rationale for each.
2. Present a summary to the user and ask for approval using `AskQuestion`.
3. Apply approved updates to this file immediately.

Do **not** modify this skill mid-conversation unless the user explicitly asks.
