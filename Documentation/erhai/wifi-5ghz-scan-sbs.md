# WCN7860: no 5 GHz networks in a scan while disconnected (SBS)

Found on the OnePlus Pad 3 (`erhai`, WCN7860 hw2.0, firmware `WLAN.GNG.1.0-04065`, driver `ath12k`) on 2026-11-22.
The driver code involved is the same in mainline (v7.4-rc3 and `ath-next` of that day), so this is material for an
upstream report. Commit in this tree: "wifi: ath12k: Keep the firmware from using SBS on the OnePlus Pad 3".

Status: the cause is measured on the tablet, the fix is **not yet verified on the tablet** (see "How to verify").

## What happened

While the tablet is not connected to any network, a full scan (NetworkManager through wpa_supplicant, or `iw scan`)
finds the 2.4 GHz BSSes and no 5 GHz BSS on channels 36-64. Connected to a 2.4 GHz network, every scan finds them.
Joining a 5 GHz network works once its BSS is in the scan cache. So a tablet that is offline never offers the 5 GHz
networks.

## What was measured

Kernel `7.4.0-rc2-erhai-eb6` has no ath12k debug options, so the measurements use what is always there: the
`cfg80211:cfg80211_inform_bss_frame`, `mac80211:drv_hw_scan` and `mac80211:api_scan_completed` tracepoints (through
`perf`, which gives the frame bytes: beacon or probe response, receive frequency, time since the request) and the
survey data the firmware reports per scanned channel (`iw dev wlp1s0 survey dump`: dwell time and busy time). An AP
at -37 dBm sits on channel 36 (5180 MHz), a weaker one on channel 44. Scripts and logs are in the workspace:
`tools/tab/wifi5-trace.sh`, `wifi5-order.sh`, `wifi5-split.sh`, `wifi5-boundary.sh`, `wifi5-verify.sh` and
`device-logs/wifi5-*.txt`.

Disconnected, active scan, each case at least twice with the same result:

| Channels in the request | BSS on 5180 found | Survey of 5180: dwell / busy, ms |
|---|---|---|
| 5180 alone; 5180-5240 | yes | 50 / 1-3 |
| 5180 + 5220, + 5260, + 5320 | yes | 50 / 1-4 |
| 5180 + all 6 GHz; 5180-5240 + 5955 | yes | 49-50 / 1-3 |
| all 2.4 GHz + 5180-5240; 2412 + 5180 in either order | yes | 51-52 / 1 |
| **5180 + 5660, + 5700, + 5745, + 5825** | **no** | **41-42 / 0** |
| 5220 + 5745 (the BSS on 5220) | no | 42 / 0 |
| all 5 GHz; 5 GHz without the DFS channels | no | 42 / 0 |
| all 5 GHz + all 6 GHz | no | 42 / 0 |
| full list in any order (2.4, 5, 6 GHz) | no, 47 scans of 47 | 46-47 / 0 |
| 5180 + 5220 + 5745, passive | no | 142 / 0 |

- In the failing scans nothing at all arrives from the lower channels: no probe response and no beacon. The upper
  channels do receive (a BSS on 5745 showed up in several of them).
- The failing scans are shorter than the sum of their parts: all 5 GHz channels take 0.93 s, not 1.5 s; 5180 + 5220 +
  5745 passive takes 0.41 s, not 0.45 s. The request is split in two that run at the same time.
- With 2.4 GHz and one 5 GHz channel the two also run at the same time (frames from 2412 and 5180 arrive within the
  same 50 ms), and that works. Two MACs on 2.4 and 5 GHz (DBS) are fine; two MACs on lower and upper 5 GHz are not.
- Connected to 2.4 GHz, one MAC stays with the connection and the other scans all of 5 and 6 GHz in turn: 5180 is the
  first channel, 45 ms after the request, and the BSS answers.
- The border lies between 5320 and 5660 MHz (the channels in between are disabled in the regulatory domain used).

Ruled out on the way:

- regulatory state (channels enabled, no "no IR", country applied while disconnected) - `device-logs/wifi5-reg-1450.txt`;
- the probe request itself: the same request with only lower channels gets probe responses addressed to the
  interface;
- 11d scans: they are off after `iw reg set` (`regdom_set_by_user`), and the failure stays;
- the board data file: `bdwlan.elf` in use is byte for byte the one of the stock firmware 16.0.10.600;
- a missed upstream fix: `wmi.c` equals mainline, the differences in `mac.c` do not touch scanning.

## Cause

The chip has two MACs. Besides one MAC per band (DBS) the firmware can put the lower part of the 5 GHz band on one
MAC and the upper part with 6 GHz on the other (SBS), if the board's RF front end has the filters for it. This board
does not: the stock Android configuration has `enable_sbs=0` in every `WCNSS_qcom_cfg.ini` (`odm/vendor/etc/wifi/`,
`vendor/etc/wifi/peach*/`), in the OnePlus file under the comment "the hardware ... not support sbs mode, this diff
is created by hardware team".

The firmware is not told by the board data. The vendor driver (qcacld-3.0) tells it: the firmware reports its initial
"dual MAC" configuration in the service ready ext event (`default_conc_scan_config_bits`, `default_fw_config_bits`),
the driver copies the bits it wants, takes the "async SBS" bit (bit 26 of the firmware mode configuration) only if
`enable_sbs` is set (`policy_mgr_init_sbs_fw_config()`), and sends the result with `WMI_PDEV_SET_MAC_CONFIG_CMDID`.

`ath12k` parses the two default words and never sends that command. The firmware keeps what it started with and,
when no vdev holds a MAC, scans the two 5 GHz parts in parallel. The MAC that gets the lower part hears nothing.

## The fix

`ath12k_core_disable_sbs()` in `core.c`, called from `ath12k_core_start()` after the hardware mode is set: on boards
in `ath12k_core_no_sbs_table` (machine compatible `oneplus,erhai` with WCN7860) send
`WMI_PDEV_SET_MAC_CONFIG_CMDID` with the firmware's default words, bit 26 of the firmware mode word cleared, wait for
`WMI_PDEV_SET_MAC_CONFIG_RESP_EVENTID` and warn if the firmware refuses or does not answer. On those boards the
driver also stops treating lower and upper 5 GHz as a pair for simultaneous MLO links (`support_sbs`), as the vendor
driver does when the bit is off.

The kernel log shows what was sent, once per firmware start:

```
ath12k_wifi7_pci 0000:01:00.0: no SBS on this board: fw mode config 0x........ -> 0x........, scan config 0x........
```

If bit 26 (`0x04000000`) is not set in the first number, the firmware did not start with async SBS allowed and this
explanation is wrong.

## How to verify

`tools/tab/wifi5-verify.sh` (workspace): prints the log line above, then scans while disconnected with the requests
that fail on `eb6` and judges each by the BSS on channel 36. Control, kernel `eb6`:
`device-logs/wifi5-verify-eb6-control-1615.txt`, 10 of 13 scans fail. Expected with the fix: 13 of 13 pass and the
survey of 5180 shows a dwell of about 50 ms with a busy time above 0 in every scan. The duration of the full scan is
not a criterion: the connected full scans of the control run, which do find the BSS, took 3.41 s, the same as the
failing disconnected ones.

The fix and `eb6` differ in nothing else that touches Wi-Fi, so the control is the `eb6` run above. If the log line
shows bit 26 set and cleared and the scans still fail, the firmware ignores the bit for scans or the cause is
another one; the next things to try are the "async DBS scan" bit of the scan configuration word (bit 28) and the DBS
policy bits of the scan request (`scan_ctrl_flags_ext`, "force non-DBS").

## Open

- Where the property belongs. A table of machines in the driver works with the device tree that is flashed today; a
  property of the Wi-Fi node in the board DT would be the upstream form.
- Whether other WCN7850/WCN7860 boards without SBS filters show the same. The symptom to look for: `iw dev X scan
  freq 5180 5745` while disconnected finds nothing on 5180.
- `ath12k_wmi_send_scan_start_cmd()` never writes `scan_priority` into the command (it sets `arg->scan_priority` from
  the 11d state and drops it), so every scan runs with priority 0 (very low). Same in mainline. Not changed.
- The phy's country changed from RU to US after connecting (seen 2026-11-22, `wifi5-reg-1450.txt`). Not looked at.
