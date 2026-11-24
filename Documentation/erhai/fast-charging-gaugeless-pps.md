# PD PPS fast charging on the OnePlus Pad 3 without the pack gauge (gauge-id 1)

Branch `eb/fast-charging`. Written 2026-11-23. **On by default at 3000 mA** (branch `eb/fast-charging-on`): steps 0-3
and all fault drills passed on the tablet on 2026-11-23 (see "Results on this tablet"). Steps 4-5 at the stock 5000 mA
need a charger with a 5 A PPS range and a 5 A cable, which the owner does not have yet. **Since kernel `eb14`
(`eb/charging`): 5000 mA by default** (`eb/fast-charging-5a`, owner's decision 2026-11-24, without a separate 5 A
test), and a session that charges its table through hands over to the stock FFC tail and full decision
(`ffc-gaugeless.md`, `gaugeless_ffc` and `gaugeless_full` on by default). Background:
`fast-charging-no-gauge.md` (what the variant is, why fast charging was parked) and `bypass-charging.md` on branch
`eb/bypass-charging` (the firmware protocol).

## Why fast charging was off

The charger drivers of the port (`qcom_battmgr.c`, `erhai_pps.c`, `sc8547a.c`) were written for the variant with
the pack gauge (`bq27z561`, `oplus,gauge-id = 2`). Fast charging there means: the AP negotiates a higher bus
voltage (PD PPS, UFCS or serial SUPERVOOC), switches the two SC8547A charge pumps on, which put about twice the bus
current straight into the cell, and regulates the bus by the cell voltage, current, temperature and state of charge
read from the pack gauge. On the variant without that gauge (`oplus,gauge-id = 1`) those readings do not exist, so
`eb/fixes` kept the pumps unregistered ("parked") and left the charge to the ADSP firmware and its buck charger:
5 V, or a fixed 9 V PD contract with a 2 A input limit, about 18 W.

What was missing to drive the pumps safely, and where each piece comes from now:

| Needed | Stock system on gauge-id 1 | This branch |
|---|---|---|
| cell voltage, pack current | ADSP battery properties, read live while PPS runs (`fg_sm8350_get_battery_mvolts_max()`, `fg_sm8350_get_average_current()` in `charger_ic/oplus_hal_adsp.c` switch from the cached buffer to a property read when `oplus_get_pps_charging()`) | the same properties (`BATT_VOLT_NOW`, `BATT_CURR_NOW`) through `qcom-battmgr-bat`, **checked on every poll against the pumps' own ADCs** |
| temperature, state of charge | ADSP gauge topic | the same properties (`BATT_TEMP`, `BATT_CAPACITY`) |
| second voltage and current measurement | SC8547A ADC (`OPLUS_IC_FUNC_CP_GET_VOUT/IIN`) | SC8547A ADC (`sc8547a_pps_snapshot()`), used for the entry window, the bus servo and the cross-check |
| pack is the stock one | 1-wire Maxim chip in the pack (`oplus,maxim_encryption_gauge_groups`, GPIO 60), `NON_STANDARD_VOTER` disables PPS without it | no driver for the chip; instead the firmware must report its `Erhai_TABLET_12140MAH…` battery profile, and enabling is an explicit decision (module parameter) |
| current and voltage table | `oplus,pps_charge` / `pps_charge_third_strategy` of the device tree | transcribed into `erhai_pps_envelope.h` |
| pump control, PPS requests, watchdog | AP (`oplus_chg_pps.c`, `oplus_virtual_cp`), PPS requests through the ADSP | already in `erhai_pps.c` / `sc8547a.c`, unchanged |

Read from the tablet on 2026-11-22 (read-only, on battery): the firmware's current and voltage readings differ on
every sample taken 0.2 s apart, so they are live, not a slow cache. The unknown notification `0x64` seen at plug-in
is `BC_ADSP_NOTIFY_AP_CP_MOS_DISABLE` of the stock header (`oplus_hal_adsp.h`); the AP has nothing to switch off at
that point.

The stock overlays with and without the gauge have the same charge tables and the same pump nodes; only the source
of the readings and of the pack authentication differs. So what the stock system does on this variant is exactly
"pumps driven by the AP from the firmware's readings".

## Design

Protocol: **PD PPS only**. Any standard PPS charger works; nothing proprietary is needed. Serial SUPERVOOC and UFCS
are kept out on this variant (the battery manager masks both routes and the engine refuses UFCS). They need a
OnePlus charger and put up to 80 W into the pack; the same cross-checks would apply, but only after PPS has been
proven on the tablet, as separate work.

Switch: `qcom_battmgr.gaugeless_pps_ma` (module parameter of the built-in battery manager, mode 0644).

- `0` (default): today's behaviour, pumps parked.
- `N > 0`: PPS allowed, and `N` is the ceiling of the current taken from the charger in mA (clamped to the stock
  5000). The pumps put about `2 N` into the pack. It is read on every pass of the charge worker and every engine
  poll: lowering it lowers the request, `0` ends a running session within one poll (0.5 s) and parks the pumps.
- At runtime: `/sys/module/qcom_battmgr/parameters/gaugeless_pps_ma`; at boot: `qcom_battmgr.gaugeless_pps_ma=N`
  on the kernel command line (the command line lives in the `boot` image, so that is a build decision).

Limits, all from the stock device tree of this variant (`dtbo` 16.0.10.600, overlay 4):

- Table `pps_charge_third_strategy`: bus current by state of charge (ranges ending at 20, 35, 55, 75, 85, 90 %) and
  battery temperature (0, 5, 12, 16, 21, 35, 44, 51 °C). At most 5 A up to 4.20-4.23 V at 16-44 °C, stepping down to
  1 A; the session ends at 4.49 V (4.47 V from 35 °C, 4.13 V from 44 °C). That is lower than the SUPERVOOC table
  the engine uses with the pack gauge (6.1 A, 4.545 V).
- No session starts above 90 % (`pps_charge_strategy_soc`), below 0 °C or from 51 °C; from 44 °C only up to 55 %
  and 4.0 V (`pps_warm_allow_soc`, `pps_warm_allow_vol`).
- From 43 °C the bus current is at most 3 A until the battery is back under 42 °C
  (`pps_strategy_batt_high_temp`, `pps_strategy_high_current`).
- Pack current above 11.4 A for 8 s ends the session (`pps_ibat_over_third`, 55 W class).
- The buck keeps its stock 2 A input limit of a PD source (`pd-iclmax-ma`) while the pumps are not engaged, and the
  AP writes no charge current limit of its own (the firmware's 4.5 A stays).
- Unchanged from the engine: bus voltage at most 9.9 V, 4 A per pump, pump watchdog 5 s, connector thermistors
  (`sc8547_btb_*`), SC8547A fault flags, bypass and charge-limit policy (the pump is released, the buck obeys).
- The guard of `eb/fixes` keeps running: below 0 °C, from 53 °C, on battery overvoltage or without readings it
  withdraws the charge output, which also ends PPS.

Cross-check (`pps_gaugeless_check()`), on every 0.5 s poll with an engaged pump:

1. the firmware's current reading must change (the same microamp value 20 polls in a row means it stopped measuring);
2. the firmware's cell voltage must be within 200 mV of the master pump's battery ADC; the higher of the two runs
   the table;
3. with the buck suspended, the pack current must not exceed twice the bus current measured by the pumps (plus
   10 % and 500 mA of ADC tolerance) and must not be more than 6 A (the largest system draw) below it.

Four consecutive failures of 2 or 3, or the stall of 1, end the session with "battery readings disagree with the
pumps" and no new PPS session starts until the charger is replugged.

Fallback: every exit (limit, failed check, lost reading, failed request, charger capability change, parameter
cleared) releases the pumps, returns the bus to 5 V and hands the port back to the firmware, which negotiates the
fixed 9 V contract as it does today. A session that has charged its table through is not repeated on the same
plug; the firmware finishes the charge.

## What is implemented

- `erhai_pps_envelope.h`: the stock PPS table and thresholds.
- `erhai_pps.c`: the gaugeless mode described above. With the pack gauge nothing changes
  (`erhai_pps_gaugeless_ma()` is negative there).
- `qcom_battmgr.c`: the parameter; `qcom_battmgr_gaugeless_pumps()` un-parks and parks the engine;
  serial/UFCS masked without the gauge; `erhai_pps_gaugeless_ma()`.
- `charge_contract` of `qcom-battmgr-usb` shows `gaugeless_pps_ma=` and `gaugeless_done=` in front of the engine
  state once the pumps are let in.

## Test plan (to run after flashing, each step only with the owner's consent)

Preconditions for every step: owner present and able to pull the cable; battery 20-80 % and under 38 °C at the
start; a PD PPS charger whose PPS range covers 6.7-9.9 V (steps 4-5 need a 5 A PPS range and a 5 A e-marked
cable); SSH or the USB console open with

    watch -n1 'cat /sys/class/power_supply/qcom-battmgr-usb/charge_contract; \
      cd /sys/class/power_supply; cat qcom-battmgr-bat/{voltage_now,current_now,temp,capacity} \
      qcom-battmgr-usb/{voltage_now,current_now}'

and `dmesg -w | grep -E 'erhai_pps|sc8547|battmgr'` in a second session.

Abort at any time: `echo 0 > /sys/module/qcom_battmgr/parameters/gaugeless_pps_ma` (the session ends within 0.5 s,
the charger returns to 5 V and then the fixed 9 V contract), or pull the cable. If the AP hangs, the SC8547A
watchdog stops the pumps after 5 s without a feed. Abort without waiting for a threshold on: any
`erhai_pps: firmware battery readings disagree`, any `SC8547A OVP/OCP`, a hot connector, a cell voltage above
4.50 V, or a reading that stops updating.

| Step | Setting | Expect | Abort if | Duration |
|---|---|---|---|---|
| 0 | parameter 0 (as flashed) | exactly today's behaviour: `state=idle`, fixed 9 V, input up to 2 A. Read the charger's PPS ranges from `/sys/class/typec/port0-partner/usb_power_delivery/source-capabilities/` if the UCSI driver lists them | anything differs from eb7 | 5 min |
| 1 | 1000 (about 2 A into the pack, less than today) | `PPS contract`, then `CP 2:1 ENGAGED`; bus 0.9-1.1 A; firmware pack current near `2 x bus - system`; firmware and pump cell voltage within 100 mV | battery over 40 °C; bus over 1.3 A; pack over 3 A | 3 min, then parameter 0 and check the fall back to 9 V |
| 2 | 2000 (about 4 A, 17 W) | as step 1 at twice the current; slave pump may join | over 40 °C; bus over 2.3 A; pack over 5 A | 5 min |
| 3 | 3000 (about 6 A, 26 W) | charging faster than today; temperature rise under 5 °C | over 42 °C; bus over 3.4 A; pack over 7 A | 10 min |
| 4 | 5000 (stock, about 10 A, 45 W) | current follows the table for the state of charge and temperature | over 43 °C (the engine itself drops to 3 A there); bus over 5.4 A; pack over 11 A | 10 min |
| 5 | 5000, from under 30 % to the end | steps down by cell voltage, exit "cell taper complete" or "battery limit" at 4.47-4.49 V, `gaugeless_done=1`, then the firmware finishes from fixed 9 V and reports Full | as step 4; cell over 4.50 V under the pumps | until Full |
| 6 | fault drills at 2000, one at a time | parameter 0 mid-session; cable pulled mid-session; `charge_behaviour` `inhibit-charge` mid-session (pump released, buck holds); cover closed (suspend) mid-session; replug | pumps still engaged 5 s after any of them; charging not resuming on replug | 15 min |

After each step save `dmesg` and the samples to `device-logs/` and note the result here.

## Open risks

- Up to 2 A from the charger (about 3.4 A into the pack) has run on this tablet, 5 A has not. The engine's PPS and
  pump sequences were validated by the port's author on the variant with the gauge; the ADSP firmware of this unit may answer the PPS properties differently.
- The pack is not authenticated; the battery profile name is a consistency check, not proof.
- The SC8547A battery ADC and its battery overvoltage threshold (`REG_00 = 0x20 | ovp`, device tree default) have
  not been read on this unit; step 1 shows whether the pump's cell voltage agrees with the firmware's.
- On 2026-11-21 this tablet rebooted while on a SUPERVOOC charger under another kernel, after ADSP/battmgr
  timeouts in pstore; the cause is unknown. A lost reading ends the session here and the pump watchdog covers a
  hung AP, but an ADSP crash during a session has not been exercised.
- The port ships the ADSP firmware of 16.0.7.200, the stock system of this unit runs 16.0.10.600.
- A system drawing more than 6 A from the pump output would trip check 3 and fall back to 9 V; that is safe but
  would need the bound revisited.
- With a 3 A cable or charger the gain over today's 18 W is small until the pump engages (bus 3 A at about 9 V,
  27 W).

## Results on this tablet

Kernel `eb9` (`eb/integration` 5d94aa18c), firmware 16.0.10.600, a PD PPS wall charger and its own cable, the
tablet in use with the screen on (system draw about 0.8-1 A). Samples once a second by `tools/tab/fastchg-step.sh`
and `tools/tab/fastchg-drill.sh` of the workspace, logs `device-logs/fastchg-*.txt`.

| Step | Date, state | Result |
|---|---|---|
| 0 | 2026-11-23, 82 %, 28 °C | as `eb7`: `state=idle`, fixed 9 V, input 1.98 A, pack 2.0-2.9 A. The UCSI driver lists no partner capabilities on this unit, so the charger's PPS ranges were not read; the engine reads them itself (contract accepted in steps 1-2). Passed. |
| 1 | 1000 mA, 3 min, 82-83 %, 27-29 °C | PPS contract after 0.2 s, CP entry, `CP 2:1 ENGAGED` (chip vbus 9135 mV, vbat 4342 mV), then `taper` (cell curve) for the whole step: pack 1.08 A on average, 1.28 A at most, cell at most 4383 mV, no `readings disagree`. Parameter 0: back to idle in 0.5 s, the fixed 9 V contract within 2 s (pack 2.9 A). Passed. |
| 2 | 2000 mA, 5 min, 83-85 %, 27-29 °C | engaged after 6 s, `taper`, PPS set 9.67 V: pack 2.98 A on average, 3.43 A at most (fixed 9 V gives 2.88 A on the same tablet at the same moment), cell at most 4483 mV, battery at most 29 °C, no warnings. Fallback as in step 1. Passed. |
| 6 | parameter 0 mid-session, 86 % | request 1500 mA (the table's 85-90 % row), pump bus 1434 mA, pack 2.06 A; parameter 0 → pump off and `state=idle` within one sample (under 1 s), pack back to 2.7 A from the fixed 9 V contract after 1 s. Passed. |
| 6 | `inhibit-charge` mid-session, 87 % | not run: no session started. The fixed 9 V charge had already taken the cell to 4.50 V, above the table's 4.49 V end, so the engine stayed idle (correct). |
| 6 | `inhibit-charge` mid-session, 57 %, kernel tp3 | pump off within 0.9 s, charging off (battery -1..-4 mA, the charger feeds the system, 0.25-0.5 A); `auto` again: fixed 9 V within 8 s, a new PPS session engaged 10 s later (bus 2.0 A, pack 3.2 A). Passed. |
| 6 | suspend mid-session, 58 % | not reachable: with a wall charger online the battery manager holds its charge wakeup source (`erhai_should_stay_awake()`) and `qcom_battmgr_prepare_sleep()` vetoes suspend, so `systemctl suspend` failed with EPERM after 30 ms and the session went on under the AP's supervision. Safe; the PM notifier's PPS stop only matters for a source that does not keep the AP awake. |
| 6 | cable pulled and replugged mid-session, 59 % | pump off 93 ms after the pull (battery -0.85 A); replug: fixed 9 V after 6 s (pack 2.9 A), a new PPS session engaged 20 s later (bus 2.0 A, pack 3.1 A). Passed. |
| 3 | 3000 mA, 10 min, 59-65 %, 30-31 C | the engine asked the charger for 2500 mA (this charger's PPS range or the table's 55-75 % row), pack 4.02 A on average and 4.33 A at most against 2.9 A from fixed 9 V (about 40 % faster), +6 % in 10 min, battery +1 C, no warnings; parameter 0: fixed 9 V again within 2 s. Passed. |

Observations:

- While the pump runs, `qcom-battmgr-usb` `current_now` reads 0: the firmware does not measure the bus with the buck
  suspended. The bus current is `pump_ibus_ma` in `charge_contract` (from the master pump's ADC).
- Above 85 % the stock table allows 1.5 A from the charger; the pumps then put about 2.1 A into the pack, less than
  the fixed 9 V contract gives (2.8 A). The stock system behaves the same (it suspends the buck as well).
- The firmware's own fixed 9 V charging holds the cell at 4.49-4.51 V from about 86 %; the end of a PPS session at
  4.49 V is therefore normal, not a fault.
- The engine logs its `charging`/`taper vbus ...` line on the first 60 polls of a session only (`pps_log_ticks`,
  30 s), then stays quiet until the session ends: in step 2 the lines stop 33 s after the pump engaged.
- Still to do: steps 4-5 at 5000 mA (a charger with a 5 A PPS range and a 5 A cable); until then the default is
  3000 mA.

