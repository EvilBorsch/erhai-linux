# FFC (fast float charge) on the OnePlus Pad 3 without the pack gauge (gauge-id 1)

Branch `eb/ffc-gaugeless`. Written 2026-11-24. **Off by default** (`qcom_battmgr.gaugeless_ffc=0`) and not yet run on
the tablet. Background: `fast-charging-no-gauge.md` (the variant, the guard) and `fast-charging-gaugeless-pps.md` (PD
PPS on this variant, which FFC follows).

## The problem

Measured on the owner's tablet, kernel `eb12`, a PD PPS charger: the PPS session ended at about 90 % with
`battery limit` (the cell had reached the table's 4.49 V), the firmware went on from a fixed 9 V contract and the
charge stalled:

- state of charge 93 %, cell 4484 mV, pack current 43 mA, status Charging, 30 °C;
- `charge_contract`: `buck_phase=1 ffc_step=0 ffc_used=0 buck_fv_mv=-1 buck_fcc_ua=-1 buck_full=0`.

Without the pack gauge the port's buck supervisor (`qcom_battmgr_buck_tick()`, `erhai_buck.h`) does not run;
`qcom_battmgr_buck_guard()` only withholds charging under unsafe conditions and sets nothing (`buck_fv_mv=-1`: the AP
never wrote a float voltage). The firmware charges to its own pre-FFC float voltage and holds the cell there. Nobody
runs the FFC tail and nobody declares the pack full.

## What the stock charger does on this variant

Sources: OnePlusOSS `android_kernel_modules_and_devicetree_oneplus_sm8750`, branch `oneplus/sm8750_b_16.0.0_pad_3`
(16.0.9.400 sync; WSL `~/work-oplus-src/pad3`), `vendor/oplus/kernel/charger/v2`: `oplus_chg_comm.c`,
`oplus_chg_pps.c`, `oplus_chg_wired.c`, `charger_ic/oplus_hal_adsp.c`, `mms/oplus_mms_gauge.c`,
`include/oplus_chg_comm.h`. Values: stock `dtbo.img` of 16.0.10.600, overlay 4 (this unit, `oplus,gauge-id = <1>`;
WSL `~/work-dt/dtbo-04.dts`, node with `oplus_spec,fv-mv`). The overlay without the gauge has the same values as the
one with it (`erhai_buck.h` was written from the latter).

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

`ffc_status` is 0 default, 1 wait, 2 FFC, 3 idle (`enum oplus_chg_ffc_status`).

### Battery authentication

Stock FFC requires the authenticated pack. It is not tested in the FFC code itself, but nothing leads to FFC
without it: `oplus_chg_pps.c` (and VOOC, UFCS) vote `NON_STANDARD_VOTER` on the fast charging disable votable when
`!batt_hmac || !batt_auth`, so no fast session runs and `oplus_comm_switch_ffc()` is never called. The ordinary
profile of such a pack is the non-standard one as well (`non-standard-fv-mv` 3980 mV, `non-standard-ibatmax-ma`
500 mA, `NON_STANDARD_VOTER` in `oplus_chg_wired.c`). On this variant authentication is the 1-wire Maxim chip in the
pack, for which there is no Linux driver.

So FFC needs the same decision the owner took for PPS: **accept the battery profile the firmware selected
(`Erhai_TABLET_12140MAH...`) as the identity check.** `BATT_OPLUS_AUTH` / `BATT_OPLUS_HMAC` stay 0 on this variant
(nothing fakes them); the FFC tail treats its samples as qualified only after `qcom_battmgr_gaugeless_pack()` has
seen that profile, exactly as the pumps are let in for PPS, and only when `gaugeless_ffc` is set. Enabling the
parameter is that decision.

### How the stock system shows 100 %

The percentage Android shows is not the firmware's figure: `oplus_chg_comm.c` smooths it into `ui_soc`
(`oplus,smooth_switch`, `oplus,reserve_chg_soc = <1>`), and once `batt_full` is set the UI figure is driven to 100
(`should_report_full()`, `oplus_comm_ui_soc_update()`). Linux shows the firmware's `BATT_CAPACITY` as it is. The
measured stall (4484 mV, 43 mA) is already past the stock full point (≤ 597 mA above 4440 mV): the cell is nearly
saturated at the pre-FFC voltage and the 93 % is the firmware gauge's figure for that. FFC itself mostly shortens the
constant-voltage tail; whether the firmware's figure reaches 100 % after FFC, the `BATT_BAT_FULL_CURR_SET` hints and
`BATT_SET_BATT_FULL`, has to be measured (test step 1). If it does not, showing 100 % at the AP's full decision is the
stock `ui_soc` behaviour and a separate decision.

## What is implemented

Switch: `qcom_battmgr.gaugeless_ffc` (bool, mode 0644). `/sys/module/qcom_battmgr/parameters/gaugeless_ffc` at
runtime, `qcom_battmgr.gaugeless_ffc=1` on the kernel command line.

- `0` (default): exactly the behaviour without this branch. The PPS engine does not report the end of a gaugeless
  session; the guard runs; the firmware finishes by itself.
- `1`: a gaugeless PPS session that charges its table through (`battery limit` at the table's CV, or `cell taper
  complete`) hands the tail to the battery manager (`erhai_charge_tail_complete()`), which runs the stock tail on
  the buck until the charger is removed. Clearing it at any time ends the tail and gives the charge back.

Code:

- `erhai_buck.h`: unchanged. The tail is `erhai_buck_pump_done()` → `erhai_buck_update()` (wait, full before FFC,
  FFC steps, idle, ordinary float voltage, full, recharge) and `qcom_battmgr_buck_apply()` (output, FV, FCC, the
  `BATT_OPLUS_FULL_PARA` and `BATT_OPLUS_SET_FULL` hints), the same code as with the gauge.
- `qcom_battmgr.c`: `qcom_battmgr_ffc_arm()` (takes the tail over), `qcom_battmgr_ffc_tick()` (samples, cross-checks,
  steps the tail; replaces the guard while armed), `qcom_battmgr_ffc_handback()` (every exit). Hooks: the end-of-session
  event (`qcom_battmgr_tail_owned()`), `qcom_battmgr_buck_tick()`, `qcom_battmgr_buck_release()` (charger removed,
  driver stop, shutdown), firmware restart (`PD_EVENT_SERVICE_RESET`), suspend (`qcom_battmgr_pm_notify()`).
  `qcom_battmgr_guard_withholds()` is the guard's rule, now shared.
- `erhai_pps.c`: `pps_tail_complete()` reports a gaugeless session's end when `erhai_gaugeless_ffc()` says so; new
  operation `cell_mv` (`erhai_pps_cell_mv()`) lends the master SC8547A's battery ADC while the pumps are idle (ADC held
  on as at the start of a PPS session, nothing switches); refused while the engine owns the pumps.
- Also: `qcom_battmgr_buck_apply()` now clamps the FCC it saves for restoring to `BATT_CHG_CTRL_LIM_MAX`, as
  `qcom_battmgr_pd_fcc()` already did; the firmware reports a limit above its ceiling and refuses that value on write.

### Limits and cross-checks

Every sample (once a second while FFC charges, every 4 s otherwise):

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
it runs the PPS table.

Every end (`qcom_battmgr_ffc_handback()`): output off, float voltage back to what the firmware had before FFC if it
read as a float voltage (4400-4500 mV), else the stock normal 4490 mV (which is what the firmware holds the cell at
by itself, measured 4.49-4.51 V); FCC back to the value saved before FFC; hints back to "no FFC, not full"; output on
as the guard would have it; the pump ADC released; the guard resumes. If any write fails the output stays off and
the worker retries. A fault blocks FFC until the charger is replugged (the PPS engine blocks a second session on the
same plug anyway).

### Differences from stock

- Temperature: the battery temperature from the firmware, as everywhere in this driver; stock uses its
  `shell_temp`.
- Only PD PPS leads to FFC: serial SUPERVOOC and UFCS do not run on this variant.
- The AP sends no `fastchg_ing` bit in `BATT_BAT_FULL_CURR_SET` during a PPS session (unchanged from today).
- The extra cross-checks, the ceiling and the 0 °C rule above.

## Test plan (after flashing, each step only with the owner's consent)

Preconditions: owner present and able to pull the cable; the PPS charger and cable of the measurement above;
battery 75-85 % and under 35 °C at the start, so that a PPS session runs and ends at its CV; SSH or the USB console
with

    watch -n1 'cat /sys/class/power_supply/qcom-battmgr-usb/charge_contract | tr " " "\n" | \
      grep -E "^(buck_phase|ffc_step|buck_fv_mv|buck_fcc_ua|buck_enable|buck_full|buck_mv|buck_ma|buck_temp|gaugeless_ffc|ffc_gl|ffc_fw_mv|ffc_pump_mv|ffc_fw_fv|ffc_restore_fv_mv|ffc_gl_end|state|gaugeless_done)="; \
      cd /sys/class/power_supply; cat qcom-battmgr-bat/{status,capacity,voltage_now,current_now,temp}'

and `dmesg -w | grep -E 'gaugeless FFC|erhai_pps|charge output'` in a second session.

`buck_phase`: 0 off, 1 ordinary FV, 2 FFC wait, 3 FFC, 4 FFC idle, 5 full. `ffc_gl`: off, active, ending, fault.

Abort at any time: `echo 0 > /sys/module/qcom_battmgr/parameters/gaugeless_ffc` (the charge goes back to the
firmware within a second) or pull the cable. Abort without waiting for a threshold on: any `gaugeless FFC: ...`
warning, a cell above 4560 mV in either reading, `ffc_fw_mv` and `ffc_pump_mv` more than 100 mV apart, pack current
above 1.8 A at `ffc_step=0` or 1.3 A at `ffc_step=1`, battery above 40 °C, `buck_fv_mv=4630` with `buck_phase` other
than 3.

| Step | Setting | Expect | Duration |
|---|---|---|---|
| 0 | parameter 0 (as flashed) | as `eb12` with the 5 A default: PPS ends with `battery limit`, `gaugeless_done=1`, fixed 9 V, `ffc_gl=off buck_fv_mv=-1` | one PPS end |
| 1 | `echo 1` while PPS is charging (75-88 %) | at the PPS end: `gaugeless FFC: PPS session complete ...` with the firmware's FV; `buck_phase=2` output off 15 s; `buck_phase=3 ffc_step=0 buck_fv_mv=4630 buck_fcc_ua=1500000`, pack about 1.5 A; `ffc_step=1 buck_fcc_ua=1000000` at 4540 mV or under 1300 mA; then `buck_phase=4` (output off about 48 s) or 1; `buck_phase=1 buck_fv_mv=4490`; then `buck_phase=5 buck_full=1`, status Full, charge output off. Record the firmware's `capacity` at each phase change and for 15 min after Full | until Full, at most 60 min |
| 2 | drills at `buck_phase=3`, one per plug: parameter 0; `charge_behaviour` inhibit-charge; cable pulled; `systemctl suspend` | `gaugeless FFC ended (...)`, `buck_fv_mv` back to `ffc_restore_fv_mv`, output on (off for inhibit), within 1 s; suspend refused (EPERM) | 20 min |
| 3 | replug at 95 %+ | no PPS (above 90 %), no FFC, guard as today | 5 min |

After each step save `dmesg` and the samples to `device-logs/` and note the result here.

## Open risks

- The firmware's answers to an AP float voltage of 4630 mV and an AP FCC on this variant are known only from the
  variant with the gauge (the same firmware family). A refused write ends the tail.
- `BATT_VOLT_MAX` read back from the firmware is not known on this variant: if it does not read as a float voltage
  the hand-back writes the stock 4490 mV, which then stays in the firmware until it resets its own value.
- The SC8547A battery ADC with the pumps idle and the buck charging has been read on this unit only around PPS
  sessions; step 1 shows how far it is from the firmware's reading at 1.5 A.
- Whether the firmware's percentage reaches 100 % (see above).
- The pack is not authenticated; the battery profile name is a consistency check, not proof.
- A hung AP leaves 4630 mV and the FFC current in the firmware until the SoC watchdog resets the system (as in the
  stock system); the firmware's own battery overvoltage protection is the last line.
