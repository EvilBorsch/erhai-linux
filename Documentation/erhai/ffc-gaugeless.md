# FFC, the full decision and the percentage on the OnePlus Pad 3 without the pack gauge (gauge-id 1)

Branches `eb/ffc-gaugeless` (FFC) and `eb/gaugeless-full` (full decision, percentage, defaults), integrated with the
5 A PPS default in `eb/charging` (kernel `eb14`). Written 2026-11-24. **On by default** since `eb14`
(`qcom_battmgr.gaugeless_ffc=1`, `qcom_battmgr.gaugeless_full=1`), owner's decision of 2026-11-24, see "Battery
authentication". Not yet run on the tablet. Background: `fast-charging-no-gauge.md` (the variant, the guard) and
`fast-charging-gaugeless-pps.md` (PD PPS on this variant, which FFC follows).

## The problem

Measured on the owner's tablet, kernel `eb12`, a PD PPS charger: the PPS session ended at about 90 % with
`battery limit` (the cell had reached the table's 4.49 V), the firmware went on from a fixed 9 V contract and the
charge stalled:

- state of charge 93 %, cell 4484 mV, pack current 43 mA, status Charging, 30 °C;
- `charge_contract`: `buck_phase=1 ffc_step=0 ffc_used=0 buck_fv_mv=-1 buck_fcc_ua=-1 buck_full=0`.

Without the pack gauge the port's buck supervisor (`qcom_battmgr_buck_tick()`, `erhai_buck.h`) did not run;
`qcom_battmgr_buck_guard()` only withheld charging under unsafe conditions and set nothing (`buck_fv_mv=-1`: the AP
never wrote a float voltage). The firmware charges to its own pre-FFC float voltage and holds the cell there. Nobody
ran the FFC tail, nobody declared the pack full, and the percentage shown was the firmware's own, which stops short of
100 % (4484 mV at 43 mA is already past the stock full point of 597 mA above 4440 mV).

## What the stock charger does on this variant

Sources: OnePlusOSS `android_kernel_modules_and_devicetree_oneplus_sm8750`, branch `oneplus/sm8750_b_16.0.0_pad_3`
(16.0.9.400 sync; WSL `~/work-oplus-src/pad3`), `vendor/oplus/kernel/charger/v2`: `oplus_chg_comm.c`,
`oplus_chg_pps.c`, `oplus_chg_wired.c`, `charger_ic/oplus_hal_adsp.c`, `mms/oplus_mms_gauge.c`,
`mms/oplus_mms_wired.c`, `hal/oplus_virtual_buck.c`, `include/oplus_chg_comm.h`. Values: stock `dtbo.img` of
16.0.10.600, overlay 4 (this unit, `oplus,gauge-id = <1>`; WSL `~/work-dt/dtbo-04.dts`, node with
`oplus_spec,fv-mv`). The overlay without the gauge has the same values as the one with it (`erhai_buck.h` was written
from the latter).

### When FFC starts

Only at the end of a fast charging session: `oplus_comm_switch_ffc()` is called by `oplus_chg_pps.c`,
`oplus_chg_vooc.c`, `oplus_chg_ufcs.c` and `oplus_chg_wls.c` and nowhere else. For PPS (`oplus_pps_monitor_work`,
around line 3930): when the PPS strategy reports `exit` (its last step is charged through), or the PPS full checks
(`vbat > sw_vth` on the last gear, `vbat > hw_vth`, cool/warm full) vote `CHG_FULL_VOTER`, the session exits with
`switch_to_ffc`: the AP votes the charge output off (`chg_disable_votable`, `FASTCHG_VOTER`) and calls
`oplus_comm_switch_ffc()`. A charge that never had a fast session (plain PD 9 V, QC, 5 V, or a plug-in above the
PPS start limit of 90 %) gets no FFC: the buck charges to the ordinary float voltage and the full check ends it.

### The sequence (`oplus_chg_comm.c`)

| Stage | Stock code | Values (dtbo 4) |
|---|---|---|
| wait | `oplus_comm_switch_ffc()`: `ffc_status = FFC_WAIT`, `ffc_start_work` after `FFC_START_DELAY` | 15 s, charge output off |
| full before FFC | `ffc_start_work`, `full_pre_ffc_judge`: `vbat_mv_max >= full-pre-ffc-mv && soc == 100` → full, no FFC | 4485 mV |
| temperature band | `oplus_comm_get_ffc_temp_region()` on `shell_temp`, `ffc-temp-thr`, FFC only in `PRE_NORMAL`..`NORMAL`; `BATT_TEMP_HYST` 2 °C away from the current region | 12, 21, 35, 44 °C: FFC from 12.0 to under 44.0 °C |
| FFC | `ffc_start_work`: votes `fv_max` = `wired-ffc-fv-mv[step]`, `wired_fcc` = `wired-ffc-fcc-ma[step][region-1]`, output on; `ffc_status = FFC_FAST` | 4630 mV; 1500 mA (step 0), 1000 mA (step 1), the same in all three regions |
| step | `oplus_comm_check_ffc()` every `COUNT_TIMELIMIT` (4 s): `vbat >= wired-ffc-fv-cutoff-mv - dec_vol` counted (`FFC_VOLT_COUNTS` 4, the count is not reset), or `|ibat| < wired-ffc-fcc-cutoff-ma` (`FFC_CURRENT_COUNTS` 2 in a row) → next step | 4540 mV (minus 20 mV from 400 cycles, 40 mV from 800); 1300 mA (step 0), 900 mA (step 1); `wired-ffc-step-max` 2 |
| done | after the last step: FFC votes withdrawn; SOC ≥ 100 → `FFC_DEFAULT`, else output off and `FFC_IDLE` | |
| idle | `FFC_IDLE`: output off for 12 checks, then on, `FFC_DEFAULT` | about 48 s |
| out of band | `FFC_FAST` and the temperature leaves the band: as done | |
| after FFC | ordinary float voltage `fv-mv` and the full checks: `oplus_comm_check_sw_full()` (`vbat > sw-fv-mv` and charge current ≤ `iterm-ma`, more than `sw_check_full_cnt` times; twice as many with no charge current), `check_hw_full` (`fv-mv + hw-fv-inc-mv`), `check_fv_over` (`sw-over-fv-mv`, lowers FV by `sw-over-fv-dec-mv`) | 4490 mV; 4440 mV, 597 mA; 4508 mV; 4500 mV, 8 mV (normal regions) |
| full | `oplus_comm_set_batt_full()`: output off (`CHG_FULL_VOTER`), `COMM_ITEM_CHG_FULL` | |
| recharge | `oplus_comm_check_rechg()`: `wired-vbatdet-mv` | 4390 mV (normal regions) |

The full checks run on every charge, fast session or not, except while a fast session charges
(`oplus_comm_check_sw_full()` returns while `pps_charging`/`vooc_charging`/`ufcs_charging`).

### What the firmware gets from the AP (`charger_ic/oplus_hal_adsp.c`)

`oplus,soccp_support` is not set in the stock tree, so every write is a battery property of the same GLINK protocol
the port uses (`BATTMGR_BAT_PROPERTY_SET`):

| What | Stock function | Property | Port name |
|---|---|---|---|
| float voltage, mV | `oplus_chg_8350_set_fv()` | `POWER_SUPPLY_PROP_VOLTAGE_MAX` = `BATT_VOLT_MAX` (8) | `BATT_VOLT_MAX` |
| charge current, µA | `oplus_chg_8350_set_fcc()` | `POWER_SUPPLY_PROP_CHARGE_CONTROL_LIMIT` = `BATT_CHG_CTRL_LIM` (10) | `BATT_CHG_CTRL_LIM` |
| charge output | `oplus_chg_8350_output_suspend()` | `BATT_CHG_EN` (24) | `BATT_OPLUS_CHG_EN` |
| termination current | `oplus_chg_8350_set_iterm()` | nothing: the body is `#ifdef OPLUS_CHG_UNDEF`, termination is the AP's full check | — |
| gauge hint while charging | `oplus_get_props_from_adsp_by_buffer()` → `oplus_adsp_voocphy_set_full_para_qbg()`: `fcss_status = fastchg_ing \| ffc_status << 1 \| temp_region << 3` (`oplus,batt_full_method_new` is not set) | `BATT_BAT_FULL_CURR_SET` (48) | `BATT_OPLUS_FULL_PARA` |
| full | `oplus_mms_gauge_set_batt_full_work()` → `fg_sm8350_set_battery_full()`: full while the temperature region is `COLD`..`NORMAL_HIGH` | `BATT_SET_BATT_FULL` (64) | `BATT_OPLUS_SET_FULL` |
| percentage before a reboot | `oplus_back_ui_soc_work()` on every change → `oplus_chg_8350_backup_soc()`; read back at start (`oplus_chg_8350_get_shutdown_soc()`); this unit has no `soc_backup_nvmem` | `BATT_RTC_SOC` (31) | `BATT_OPLUS_RTC_SOC` |

`ffc_status` is 0 default, 1 wait, 2 FFC, 3 idle (`enum oplus_chg_ffc_status`).

### The percentage the stock system shows

Android does not show the firmware's figure. `oplus_chg_comm.c` derives `ui_soc` from it (all values the stock
defaults; the devicetree sets only `oplus,smooth_switch`, `oplus,reserve_chg_soc = <1>` and `oplus_spec,batt_full_time
= <300>`):

- `oplus_comm_smooth_to_soc()`: from 55 % the firmware figure is shown one higher, averaged over up to four samples
  near that step (`ui_soc_smooth_table[1]`, `SOC_JUMP_RANGE_VAL`), at most every 4.5 s;
- `oplus_comm_ui_soc_update()`: one step at a time. On a charger: up to the smoothed figure at most every 10 s;
  once the AP has declared full (`sw_full`/`hw_full_by_sw`), below 45 °C and with an authenticated pack, up by one
  every 10 s to 100; down only with less than 50 mA charge current and at the ladder below. Without a charger: never
  up, down towards the smoothed figure one step per 40 s (under 60 %), 60 s (60-94 %), 150 s (95-99 %), 300 s
  (100 %), never below 1 %. Under 3.3 V (3.2 V on a charger) for 20 s (60 s) it goes down to 0 at 15 s a step
  (`oplus_comm_is_under_uv()`). After a suspend it catches up by 1 % per 10 min slept;
- `oplus_comm_report_full()`: 300 s at 100 % on a charger outside a fast session also sets full; below 100 % full is
  cleared unless the AP's full check holds it;
- plug-out (`oplus_comm_plugin_work()`): full cleared; at 95 % and more the next step down waits the full interval;
- status (`oplus_comm_check_battery_status()`): Full once full is set and the percentage is 100 (or at once outside
  the normal temperatures); while it is still climbing, Charging;
- start (the gauge-ready path of `oplus_chg_comm.c`, `shutdown_soc`): the percentage saved in `BATT_RTC_SOC` if
  within 20 of the firmware's, else the smoothed figure, never 0.

### Battery authentication

Stock FFC and the climb to 100 % require the authenticated pack. FFC is not tested against it directly, but nothing
leads to FFC without it: `oplus_chg_pps.c` (and VOOC, UFCS) vote `NON_STANDARD_VOTER` on the fast charging disable
votable when `!batt_hmac || !batt_auth`, so no fast session runs and `oplus_comm_switch_ffc()` is never called; the
ordinary profile of such a pack is the non-standard one (`non-standard-fv-mv` 3980 mV, `non-standard-ibatmax-ma`
500 mA); `oplus_comm_ui_soc_update()` climbs to 100 only with `hmac && authenticate`. On this variant authentication
is the 1-wire Maxim chip in the pack, for which there is no Linux driver.

**Owner's decision of 2026-11-24:** the battery profile the firmware selected for the pack
(`Erhai_TABLET_12140MAH...`) is accepted as the identity check, as for PPS, and FFC, the full decision and the stock
percentage are on by default. `BATT_OPLUS_AUTH` / `BATT_OPLUS_HMAC` stay 0 (nothing fakes them); the AP treats the
pack as the stock one only after `qcom_battmgr_gaugeless_pack()` has seen that profile
(`qcom_battmgr_stock_pack()`, `stock_pack=1` in `charge_contract`). Without it everything stays as before `eb13`:
the guard, the firmware's percentage.

## What is implemented

### Switches

| Parameter | Default | What |
|---|---|---|
| `qcom_battmgr.gaugeless_ffc` | 1 | FFC after a gaugeless PPS session that charged its table through |
| `qcom_battmgr.gaugeless_full` | 1 | the stock full decision on every charge, and the stock percentage |
| `qcom_battmgr.gaugeless_pps_ma` | 5000 (`eb/fast-charging-5a`) | PD PPS, see `fast-charging-gaugeless-pps.md` |

All are mode 0644 (`/sys/module/qcom_battmgr/parameters/`) and read on every pass: clearing `gaugeless_ffc` ends a
running FFC tail, clearing `gaugeless_full` gives the percentage and the full decision back to the firmware within
30 s.

### FFC (`gaugeless_ffc`)

- A gaugeless PPS session that charges its table through (`battery limit` at the table's CV, or `cell taper
  complete`) hands the tail to the battery manager (`erhai_charge_tail_complete()`), which runs the stock tail on the
  buck until the charger is removed.
- `erhai_buck.h`: the tail is `erhai_buck_pump_done()` → `erhai_buck_update()` (wait, full before FFC, FFC steps,
  idle, ordinary float voltage, full, recharge) and `qcom_battmgr_buck_apply()` (output, FV, FCC, the
  `BATT_OPLUS_FULL_PARA` and `BATT_OPLUS_SET_FULL` hints), the same code as with the gauge.
- `qcom_battmgr.c`: `qcom_battmgr_ffc_arm()` (takes the tail over), `qcom_battmgr_ffc_tick()` (samples, cross-checks,
  steps the tail; replaces the guard while armed), `qcom_battmgr_ffc_handback()` (every exit). Hooks: the
  end-of-session event (`qcom_battmgr_tail_owned()`), `qcom_battmgr_buck_tick()`, `qcom_battmgr_buck_release()`
  (charger removed, driver stop, shutdown), firmware restart (`PD_EVENT_SERVICE_RESET`), suspend
  (`qcom_battmgr_pm_notify()`).
- `erhai_pps.c`: `pps_tail_complete()` reports a gaugeless session's end when `erhai_gaugeless_ffc()` says so; new
  operation `cell_mv` (`erhai_pps_cell_mv()`) lends the master SC8547A's battery ADC while the pumps are idle (ADC held
  on as at the start of a PPS session, nothing switches); refused while the engine owns the pumps.
- Also: `qcom_battmgr_buck_apply()` clamps the FCC it saves for restoring to `BATT_CHG_CTRL_LIM_MAX`, as
  `qcom_battmgr_pd_fcc()` already did; the firmware reports a limit above its ceiling and refuses that value on write.

FFC limits and cross-checks, on every sample (once a second while FFC charges, every 4 s otherwise):

| Check | Limit | On failure |
|---|---|---|
| firmware readings (voltage, current, temperature, SOC, cycles) | all readable, plausible | end, fault |
| firmware cell voltage against the SC8547A battery ADC | within 200 mV | 4 failures in a row (an unreadable ADC counts): end, fault |
| cell voltage, the higher of the two | under 4570 mV (stock FFC step 4540, battery OV 4600) | at once: end, fault |
| current reading moves (FFC charging) | not the same µA value | 10 in a row: end, fault |
| pack current (FFC charging) | ≤ FCC + 20 % + 300 mA (2100 mA at 1500, 1500 mA at 1000) | 8 in a row: end, fault |
| temperature | FFC only 12.0 to under 44.0 °C (stock band, hysteresis); as the guard: no charging below 0 °C or from 53 °C, warm rules from 45 °C | stock: leave FFC (idle or ordinary FV) |
| writes (output, FV, FCC) | acknowledged | end, fault |
| pumps | idle (the PPS engine owns nothing) | wait up to 30 s after arming, then end, fault |
| charger firmware restart | — | end, fault |
| `gaugeless_ffc` cleared; inhibit-charge, bypass or a charge limit; charge enable error | — | end |
| charger removed, driver stop, shutdown | — | end |
| suspend | the FFC phases (wait, FFC, idle) veto suspend as with the gauge; a suspend that gets through in the later phases ends the tail first | end (or the suspend is refused if that fails) |

The higher of the firmware's and the pump's cell voltage steers the tail (the step at 4540 mV, the full checks), as
it runs the PPS table. Every end (`qcom_battmgr_ffc_handback()`): output off, float voltage back to what the firmware
had before FFC if it read as a float voltage (4400-4500 mV), else the stock normal 4490 mV (which is what the
firmware holds the cell at by itself, measured 4.49-4.51 V); FCC back to the value saved before FFC; hints back to
"no FFC, not full"; output on as the guard would have it; the pump ADC released; the guard resumes. If any write
fails the output stays off and the worker retries. A fault blocks FFC until the charger is replugged (the PPS engine
blocks a second session on the same plug anyway).

### Full on every charge (`gaugeless_full`)

Outside the FFC tail the guard (`qcom_battmgr_buck_guard()`) now also runs the stock full and recharge checks of
`erhai_buck_update()` on the firmware's readings, every 4 s, on any charger: full when the cell is above 4440 mV
(4420 mV at 35-45 °C, 4080 mV above; minus 20/40 mV from 400/800 cycles) with no more than 597 mA charging for more
than 5 samples in a row (10 with no charge current); recharge when the cell has been at or under 4390 mV (4370 mV
at 35-45 °C, 4290, 4190 in the cool regions, 4030 mV warm) for more than 5 samples. Full takes the charge output off
(the system runs from the charger) and sends the stock gauge hints. Not while the pumps charge, not while a charge
policy holds the output. The decision is attachment-scoped: unplugging clears it.

The firmware keeps its own float voltage outside FFC (the AP writes none, as before), so the stock checks that act on
the AP's float voltage are off there (`erhai_buck.fv_firmware`): `hw full` (above FV + 18 mV) would fire at about
88 % because the firmware holds the cell at 4.49-4.51 V, and `FV over` would lower a float voltage the AP does not
set. Inside the FFC tail, where the AP owns the float voltage, they run as stock.

The guard also sends the stock gauge hints on every charge (`BATT_OPLUS_FULL_PARA` = fast charging bit | temperature
region << 3, `BATT_OPLUS_SET_FULL` = full in the normal regions); a refusal is recorded in `buck_gauge_error` only.

### The stock percentage (`gaugeless_full`)

`erhai_uisoc.h` is the port of the stock logic above, pure functions on jiffies. `qcom_battmgr_uisoc_worker()` runs it
on its own freezable work, with and without a charger, every 10 s or when a step is due: it reads the firmware's
percentage, voltage, current, temperature and `USB_ONLINE`, takes the AP's full decision (`buck_phase=5`) and whether
the pumps run, saves every new value to `BATT_RTC_SOC` and reads it back at start. `capacity` of `qcom-battmgr-bat`
reports it; `status` reports Full once full is set and the percentage is 100, Charging while it climbs there. The
suspend notifier measures the time asleep (boot time) for the catch-up.

Not ported (the devicetree leaves them off, or they do not exist here): low battery control, deep discharge keep 2/3
%, recharge by percentage, decimal display, parallel battery, wireless, the stock charge-up limit. The charge limit
of this kernel (`charge_control_end_threshold`) still compares the firmware's figure.

The firmware's own percentage stays visible: `fw_soc=` in `charge_contract` (with `ui_soc`, `smooth_soc`, `ui_full`,
`ui_shown`, `stock_pack`), a `dev_dbg` line on every change (`echo 'file qcom_battmgr.c +p' >
/sys/kernel/debug/dynamic_debug/control`), and the full/recharge decisions as info lines (`battery full: cell ... mV,
... mA, firmware ...%`).

### Simulation

`tools/testing/erhai/uisoc-sim.c` drives `erhai_uisoc.h` in userspace (`gcc -O2 -Wall -o /tmp/uisoc-sim
tools/testing/erhai/uisoc-sim.c && /tmp/uisoc-sim`, exit status = failed checks). Result on 2026-11-24, all checks
passed:

| Case | What the percentage does |
|---|---|
| charge 80 → 93 % (firmware), then the AP declares full at 4484 mV / 40 mA | follows the firmware + 1 (81 → 94); after full climbs one step per 10 s, 100 % within a minute, stays 100 % and full on the charger |
| unplug at full with the firmware at 93 % | full cleared at once; 99 % at once (the smoothed 94 is under the 95 % hold of stock), then one step per 150 s down to 94 % in about 13 min; never up |
| unplug at full with the firmware at 99 % | holds 100 % (99 + 1); 98 % once the firmware says 97 % |
| discharge 93 → 40 % | firmware + 1 above 55 %, equal below |
| the firmware drops 40 → 30 % at once | one step per 40 s (35 % after 200 s), 30 % after 400 s |
| firmware 0 % at 3.40 V | 1 %, not 0; under 3.3 V 0 % after 10-20 s |
| replug at 95 % (no PPS above 90 %) | stays 95 % until the firmware rises; AP full at firmware 96 % → 100 % within 40 s; a recharge with the firmware at 95 % keeps 100 % and full |
| reboot at 98 % (firmware 93 %) | starts at the saved 98 %; on battery 97 % after 150 s; on the charger stays 98 % until full, then 100 %; a saved value more than 20 away is ignored (starts at the firmware + 1) |
| suspend for an hour at 70 % shown, 60 % firmware | catches up by 6 % at once |

## What the user sees

| Situation | Percentage | Status |
|---|---|---|
| charging, PPS or fixed 9 V | the firmware's + 1 from 55 % (equal below), never down while the charger feeds the pack | Charging |
| the AP declares full (≤ 597 mA above 4440 mV, after FFC or on any charger) | climbs 1 % per 10 s to 100 % | Charging, then Full at 100 % |
| 100 % shown for 300 s with no fast session running, FFC tail included | 100 % | Full while FFC still charges (stock `oplus_comm_report_full()`: `oplus,smooth_switch` lets `report_full_check_ffc()` pass during FFC) |
| left on the charger at full | 100 %; the output stays off, recharge from 4390 mV | Full |
| unplugged at full | 100 % then down at 150-300 s a step towards the firmware's figure (+1) | Discharging |
| discharging | follows the firmware's figure down, never up; 1 % minimum until the cell is under 3.3 V | Discharging |
| replug | from where it was; up only when the firmware's figure is higher or after full | Charging |
| reboot | the saved percentage if within 20 of the firmware's | as above |

## Check list after flashing `eb14` (short)

1. `cat /sys/module/qcom_battmgr/parameters/{gaugeless_ffc,gaugeless_full,gaugeless_pps_ma}` → `Y Y 5000`.
2. `dmesg | grep -E 'pack gauge not fitted|percentage:'` → `percentage: the stock one, N% (firmware M%, saved S%)`,
   with N within 1-2 of M (or S on a reboot).
3. `cat /sys/class/power_supply/qcom-battmgr-usb/charge_contract | tr ' ' '\n' | grep -E
   '^(stock_pack|fw_soc|ui_soc|smooth_soc|ui_full|ui_shown|gaugeless_full|gaugeless_ffc|ffc_gl|buck_phase|buck_full)='`
   → `stock_pack=1 ui_shown=1`; `capacity` of `qcom-battmgr-bat` equals `ui_soc`.
4. On any charger, wait for the end of a charge: `dmesg | grep -E 'battery full|recharge|gaugeless FFC'` shows
   `battery full: cell ... mV, ... mA` (and, after PPS, the FFC lines); `capacity` reaches 100 within about a minute
   after that, `status` Full.
5. Unplug: `status` Discharging, the percentage steps down slowly, never up.
6. Passive FFC check during normal charging from under 85 % on the PPS charger: `journalctl -k | grep 'gaugeless
   FFC'` → `PPS session complete`, `phase=3 step=0 FV=4630 mV FCC=1500000`, `step=1 ... FCC=1000000`, `phase=1` or `4`,
   later `phase=5`. Any `gaugeless FFC: <check>: firmware ... pump vbat ...` warning means a cross-check ended it: note
   it, nothing else to do (the firmware takes over).

To turn it off without a new image: `echo 0 > /sys/module/qcom_battmgr/parameters/gaugeless_ffc` (FFC) or
`gaugeless_full` (percentage and full decision), or `qcom_battmgr.gaugeless_full=0` on the kernel command line.

## Full test plan for FFC (optional, each step only with the owner's consent)

Preconditions: owner present and able to pull the cable; the PPS charger and cable of the measurement above; battery
75-85 % and under 35 °C at the start, so that a PPS session runs and ends at its CV; SSH or the USB console with

    watch -n1 'cat /sys/class/power_supply/qcom-battmgr-usb/charge_contract | tr " " "\n" | \
      grep -E "^(buck_phase|ffc_step|buck_fv_mv|buck_fcc_ua|buck_enable|buck_full|buck_mv|buck_ma|buck_temp|gaugeless_ffc|ffc_gl|ffc_fw_mv|ffc_pump_mv|ffc_fw_fv|ffc_restore_fv_mv|ffc_gl_end|fw_soc|ui_soc|ui_full|state|gaugeless_done)="; \
      cd /sys/class/power_supply; cat qcom-battmgr-bat/{status,capacity,voltage_now,current_now,temp}'

and `dmesg -w | grep -E 'gaugeless FFC|erhai_pps|charge output|battery full|recharge'` in a second session.

`buck_phase`: 0 off, 1 ordinary FV, 2 FFC wait, 3 FFC, 4 FFC idle, 5 full. `ffc_gl`: off, active, ending, fault.

Abort at any time: `echo 0 > /sys/module/qcom_battmgr/parameters/gaugeless_ffc` (the charge goes back to the
firmware within a second) or pull the cable. Abort without waiting for a threshold on: any `gaugeless FFC: ...`
warning, a cell above 4560 mV in either reading, `ffc_fw_mv` and `ffc_pump_mv` more than 100 mV apart, pack current
above 1.8 A at `ffc_step=0` or 1.3 A at `ffc_step=1`, battery above 40 °C, `buck_fv_mv=4630` with `buck_phase` other
than 3.

| Step | Setting | Expect | Duration |
|---|---|---|---|
| 1 | defaults, PPS charging from 75-85 % | at the PPS end: `gaugeless FFC: PPS session complete ...` with the firmware's FV; `buck_phase=2` output off 15 s; `buck_phase=3 ffc_step=0 buck_fv_mv=4630 buck_fcc_ua=1500000`, pack about 1.5 A; `ffc_step=1 buck_fcc_ua=1000000` at 4540 mV or under 1300 mA; then `buck_phase=4` (output off about 48 s) or 1; `buck_phase=1 buck_fv_mv=4490`; then `buck_phase=5 buck_full=1`; `ui_soc` to 100, status Full | until Full, at most 60 min |
| 2 | drills at `buck_phase=3`, one per plug: `gaugeless_ffc` 0; `charge_behaviour` inhibit-charge; cable pulled; `systemctl suspend` | `gaugeless FFC ended (...)`, `buck_fv_mv` back to `ffc_restore_fv_mv`, output on (off for inhibit), within 1 s; suspend refused (EPERM) | 20 min |
| 3 | replug at 95 %+ | no PPS (above 90 %), no FFC; the guard's full check ends the charge (`battery full`), `ui_soc` 100 | until Full |

After each step save `dmesg` and the samples to `device-logs/` and note the result here.

## On the tablet

24.11.2026, kernel eb14, the owner's 100 W charger (PPS 3.3-21 V 5 A), normal use, from 73 % at 30 C:

- PPS started by itself; the request followed the stock table (2500, 2000, 1500, 1000 mA from the charger, up to
  4.3 A into the pack), 30-33 C. Shown percentage 100 % after 40 min, the firmware's own figure 100 % 3 min later.
- After 85 min `erhai_pps: exit: battery limit`, then `gaugeless FFC: PPS session complete, output off for 15 s`,
  `phase=3 step=0 FV=4630 mV FCC=1500000 uA` (cell 4460 mV, 31 C) and, 11 min later at 4540 mV (firmware 4540, pump
  ADC 4535), `step=1 FCC=1000000 uA`. Highest cell reading 4539 mV.
- Status Full 300 s after the FFC start (the row above): stock behaviour, not a fault.
- The cable was pulled 15 min into FFC: `gaugeless FFC ended (charger removed or driver stopped): firmware charge
  profile back, FV 4490 mV, FCC 4500000 uA, output on`.
- Not seen yet: the natural FFC end with `battery full`, the recharge, 5 A (the table gives it only under 55 %).

## Open risks

- The firmware's answers to an AP float voltage of 4630 mV and an AP FCC on this variant are known only from the
  variant with the gauge (the same firmware family). A refused write ends the tail.
- `BATT_VOLT_MAX` read back from the firmware is not known on this variant: if it does not read as a float voltage
  the hand-back writes the stock 4490 mV, which then stays in the firmware until it resets its own value.
- The SC8547A battery ADC with the pumps idle and the buck charging has been read on this unit only around PPS
  sessions; it decides how far it is from the firmware's reading at 1.5 A.
- `BATT_RTC_SOC`, `BATT_BAT_FULL_CURR_SET` and `BATT_SET_BATT_FULL` have never been written on this unit by Linux; the
  stock system writes them. A refused hint or save is only logged. If the firmware's gauge reacts to the full hint,
  its own figure may change (that is what the stock hint is for).
- The pack is not authenticated; the battery profile name is a consistency check, not proof (owner's decision).
- A hung AP leaves 4630 mV and the FFC current in the firmware until the SoC watchdog resets the system (as in the
  stock system); the firmware's own battery overvoltage protection is the last line.
- Full is declared from the firmware's current reading alone outside FFC (no second measurement): a reading stuck
  low would end the charge early (output off, recharge from 4390 mV), never charge more.
