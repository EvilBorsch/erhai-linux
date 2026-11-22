# Bypass charging on the OnePlus Pad 3 (gauge-id 1)

Measured on the OnePlus Pad 3 (`erhai`, OPD2415, the variant without the external fuel gauge, `oplus,gauge-id = 1`,
ADSP charger firmware from the Ubuntu port 1.0.1) with kernel `7.4.0-rc2-erhai-eb7` on 2026-11-22.

Status: on a USB PD charger (9 V fixed contract) bypass works with the existing driver code, the same way the stock
system does it in its buck mode. On a 5 V computer port (SDP) the firmware cuts the input instead; the cause is not
known and nothing has been changed for it. No driver change in this branch.

## What bypass charging should do

`charge_behaviour` of `qcom-battmgr-bat` offers `auto` and `inhibit-charge` (written by the port's "Bypass Charging"
toggle and by the Denial quick-settings tile). With a charger connected the charger should supply the system while
the battery is neither charged nor discharged, as far as the hardware allows.

## What the driver does

`qcom_battmgr_set_charge_bypass()` sets `charge_bypass`; the charge worker (`qcom_battmgr_charge_limit_update()`)
then holds charging off with `qcom_battmgr_charge_enable(false)`, one ADSP battery-manager request:

    BATTMGR_BAT_PROPERTY_SET (0x31), property BATT_OPLUS_CHG_EN (24), value 0

The input current limit is not touched and the PD/QC contract is kept on purpose (`qcom_battmgr_pd_update()`,
"Bypass charging keeps the 9 V contract"). On gauge-id 1 the guard `qcom_battmgr_buck_guard()` writes the same
property every 4 s with the combined decision (`charge_contract` shows it as `buck_enable`).

## What the stock system does

Source: the OnePlus kernel for this tablet, `OnePlusOSS/android_kernel_modules_and_devicetree_oneplus_sm8750`,
branch `oneplus/sm8750_b_16.0.0_pad_3` ("Synchronize code for OnePlus OPD2415_16.0.9.400(EX01)"), directory
`vendor/oplus/kernel/charger/v2`. The stock device tree overlay of this variant (`dtbo` 16.0.10.600, overlay 4) has
`oplus,plc_charge` (`plc_soc = 90`, buck strategy `buck_simple_strategy`), `oplus,chg_spec_version = 2` and no
`oplus,soccp_support`, so the battery-manager properties are the plain (non-SoCCP) ones with the numbering this
driver uses.

- The ADSP HAL (`charger_ic/oplus_hal_adsp.c`) has two switches:
  - "input suspend", `oplus_chg_8350_input_suspend()`: `USB_INPUT_CURR_LIMIT` (USB property 5) = 0, released with
    `0xFFFFFFFF`; voted as `WIRED_CHARGE_SUSPEND`;
  - "output suspend", `oplus_chg_8350_output_suspend()`: `BATT_CHG_EN` (battery property 24) = 0 / 1; voted as
    `WIRED_CHARGING_DISABLE`.
- What they are for (`oplus_chg_comm.c`, `oplus_enforce_chg_up_limit_result()`): above a user charge limit the input
  is suspended so the battery runs down to it; at the limit charging is only disabled, so the charger holds the
  system. Battery full (`CHG_FULL_VOTER`) uses the output-suspend vote too.
- Bypass ("PLC", `oplus_chg_plc.c`). Buck mode, `oplus_plc_protocol_enable()`: vote `WIRED_CHARGING_DISABLE`
  (`BATT_CHG_EN` = 0) and release `WIRED_CHARGE_SUSPEND` — the same request this driver sends. On top of that the
  simple buck strategy (`simple_strategy_monitor_work()`) turns charging back on whenever the state of charge has
  fallen and off again when it has risen, so the charge stays within about 1 % of where bypass started. Charge-pump
  mode (PPS/UFCS/SVOOC) instead suspends the buck input and regulates the pump's input current on the measured battery
  current towards zero (`step_strategy_check_plc_ibus()`); that needs the pumps, which stay off on gauge-id 1
  (`fast-charging-no-gauge.md`).
- The stock system also reports the bypass state to the firmware: `oplus_chg_adsp_set_plc_status()` writes
  `USB_SET_PLC_STATUS` = **USB property 70** (`BATTMGR_USB_PROPERTY_SET`, 0x33) on every change: 1
  `PLC_STATUS_NOT_ALLOW` (no charger), 2 `PLC_STATUS_DISABLE`, 3 `PLC_STATUS_ENABLE` (values from
  `config/oplus_chg_ic.json`, `plc_enable_status`; 0 is `PLC_STATUS_NOT_SUPPORT`). The number follows from
  `enum usb_property_id` in `charger_ic/oplus_hal_adsp.h`, whose numbering matches the properties this driver already
  uses (`USB_OPLUS_RERUN_AICL` 67, `USB_OPLUS_OTG_AP_ENABLE` 17, `USB_OPLUS_CID_STATUS` 21). Linux never writes it;
  read back on eb7 it is 0.

Firmware values read on eb7: `BATT_CHG_CTRL_LIM` (FCC) 4.5 A, `BATT_CHG_CTRL_LIM_MAX` 4.5 A, `BATT_CHG_EN` 1,
`BATT_VOLT_MAX` 4.63 V; input limit 1.5 A without a charger, 2.0 A on the PD contract.

## Measurements

Scripts and logs are in the workspace (`D:\androidlinux`): `tools/tab/bypass-check.sh`, `bypass-plc-test.sh`,
`bypass-fcc-test.sh`, `bypass-hold-test.sh`, the test modules `tools/tab/bypass-probe/` (built for eb7 only; they
use the driver's exported property bridge or their own pmic_glink client and write the old value back on unload),
logs `device-logs/bypass-*-eb7.txt`. Samples every 5 s; battery current positive while charging; the owner was using
the tablet, so the system load varied.

5 V computer port (SDP, ICL 1.5 A), 94 %, 4.48 V (`bypass-check-eb7.txt`):

| state | status | battery | USB input |
|---|---|---|---|
| `auto` | Charging | +145 .. +283 mA | 0.39 .. 0.40 A at 4.93 .. 4.98 V |
| `inhibit-charge` | Discharging | -350 .. -220 mA | **0 A** at 5.07 V (unloaded) |

USB PD charger, fixed 9 V contract, input limit 2.0 A, 91-92 %, 4.43-4.50 V:

| state | status | battery | USB input | log |
|---|---|---|---|---|
| `auto` | Charging | +1.0 .. +2.3 A | 1.3 .. 2.0 A at 8.6 .. 8.8 V | all |
| `inhibit-charge`, 25 s | Discharging | -131, -49, 0, -3, -1 mA | 0.47 .. 1.24 A at 8.9 V | `bypass-plc-test-eb7.txt` |
| `inhibit-charge`, 2 min | Discharging | -49 .. -145 mA, mostly -50 .. -100 | 0.29 .. 1.05 A at 8.8 .. 9.0 V | `bypass-hold-test-pd-eb7.txt` |
| `inhibit-charge` + PLC status 3, 1 min | Discharging | -51 .. -68 mA | 0.36 .. 1.33 A at 8.9 V | `bypass-plc-test-eb7.txt` |
| `auto` + FCC 0 (`BATT_CHG_CTRL_LIM`), 1 min | Charging | +192 .. +299 mA, rising | 0.55 .. 1.02 A at 8.8 .. 9.0 V | `bypass-fcc-test-eb7.txt` |

In every case `auto` charged again within 5 s and the previous values were written back.

What follows:

- On 9 V PD, `inhibit-charge` already is the stock buck-mode bypass: the charger carries the system (0.3-1.3 A at
  9 V, 3-11 W) and the battery gives 0-145 mA (up to about 0.6 W), against +1-2.3 A while charging and the whole
  load without the input. The battery's small share is what the stock simple strategy corrects by charging again
  after each lost percent.
- PLC status 3 changes nothing measurable on 9 V PD (-51 .. -68 mA lies within the plain range). It is not what
  keeps the input on, so it is not sent. The draft that sent it is kept in the workspace as
  `research/review/bypass-plc-status-NOT-APPLIED.diff`.
- FCC 0 is no pause: the firmware keeps charging at a few hundred mA (the limit reads back as 4.5 A, so the firmware
  apparently treats 0 as "not set" or applies its own floor).
- On the 5 V SDP the same request cuts the input completely. Not known: whether this happens on every 5 V source
  (a 5 V-only wall charger, a PD source that stays at 5 V because it also carries data) or only on a computer port,
  and whether PLC status 3 or anything else keeps the input on there.

## What is left

1. Measure a 5 V source: tablet on a computer port and on a 5 V-only USB-A charger, kernel eb7, the test module
   copied to `/tmp/plc_probe.ko`, then `tools/trun.sh tools/tab/bypass-plc-test.sh` (plain `inhibit-charge` as the
   control, then the same with PLC status 3). If PLC status 3 keeps the input on there, apply the saved diff (it
   mirrors the stock status handling: 1 without a charger, 3 in bypass, 2 otherwise, written again after an ADSP
   restart) and verify on both kinds of source.
2. Decide whether `inhibit-charge` should also hold the charge like the stock strategy (charge again after the
   charge has fallen by 1 %). That makes the battery's net share zero over hours but means short charging periods
   while "inhibit-charge" is set, which the sysfs ABI describes as "do not charge while AC is attached". The same
   hold is already available without a kernel change through `charge_control_end_threshold` /
   `charge_control_start_threshold` (end = present charge, start = end - 1; limits 55-100 % and 50-95 %).
3. While the battery gives even a few mA, `status` reads "Discharging" (`erhai_battery_status()`); with the measured
   -50 .. -100 mA that is accurate.
