# Battery overheat protection in the thermal framework

OnePlus Pad 3 (`erhai`), 2026-11-23. Commits in this tree: "dt-bindings: soc: qcom: pmic-glink: Add the battery
temperature sensor", "power: supply: qcom_battmgr: Register the battery temperature as a thermal sensor", "arm64:
dts: qcom: sm8750-erhai: Protect the battery with a thermal zone" and "arm64: configs: erhai-pad3: Force the power-off
when a thermal shutdown stalls".

Status: build-tested; **not yet run on the tablet** (see "How to verify").

## Why in the kernel

Until now a distribution service, `erhai-battery-guard`, protected the battery: every 10 s it read
`/sys/class/power_supply/qcom-battmgr-bat/temp`, from 49.0 °C lowered `scaling_max_freq` of both CPU clusters and
`max_freq` of the GPU one stock level per 30 s, raised them again below 47.0 °C and ran `systemctl poweroff` after two
readings at 70.0 °C. The kernel itself limited the CPU and GPU only by their junction temperature (95/100/105 °C trips).

A service starts late, can be stopped, killed or left out of an image, and its limits are plain sysfs values that
any other program may overwrite. The protection belongs where the junction protection already is: a thermal zone. It
works from the moment the charger driver probes, cannot be switched off by stopping a program, and combines with the
other zones by the framework's rules.

## The thresholds

They are OnePlus's battery protection for this tablet, from
`odm/etc/temperature_profile/sys_high_temp_protect_OPPO_24976.xml` of firmware 16.0.10.600:

| Stock key | Value | Here |
|---|---|---|
| `HighTemperatureProtectThresholdIn` | 490 | passive trip 49 °C |
| `HighTemperatureProtectThresholdOut` | 470 | hysteresis 2 °C (released below 47 °C) |
| `HighTemperatureProtectShutDown` | 700 | critical trip 70 °C, `critical-action = "shutdown"` |

Android's comfort table, which slows the tablet down much earlier, is deliberately not used (the owner's decision:
full speed as long as it is safe).

## How it works

**Sensor.** `qcom_battmgr` registers the battery temperature as an OF thermal sensor of the `pmic-glink` node
(`#thermal-sensor-cells = <0>`; the battery manager is an auxiliary device that shares that node).
`qcom_battmgr_bat_get_temp()` reads `POWER_SUPPLY_PROP_TEMP` of the battery supply, the same value as
`/sys/class/power_supply/qcom-battmgr-bat/temp` (on this unit from the ADSP firmware, one GLINK request per reading),
in millidegrees.

- No reading (firmware service not up yet or restarting: `-EAGAIN`; any other error; a value outside −40…120 °C)
  gives `THERMAL_TEMP_INVALID`. The thermal core then keeps polling and leaves the cooling as it is. Returning an
  error instead would make the core retry with a growing delay and, after two minutes of failures, disable the zone
  together with its critical trip.
- A reading more than 5 °C away from both the last accepted reading and the previous reading is held back once: the
  pack cannot change that much in one poll, so a single wrong value neither powers the tablet off nor moves the
  cooling, while a real change is passed on with the next reading (5 s later). This replaces the "two readings at
  70 °C" of the service.
- A device tree without a zone for the sensor (an older `vendor_boot`) is fine: nothing is registered, the battery
  and the charger work as before. A zone that fails to register is logged as an error; the charger stays usable.

The power supply core still registers its own trip-less zone `qcom-battmgr-bat` for the same value; it is read only
on demand and does nothing.

**Zone.** `battery-thermal` in `sm8750-erhai.dts`, polled every 5 s (the firmware raises no temperature alarm; the
pack changes by well under a degree in 5 s), governor `step_wise`:

- passive trip 49 °C, hysteresis 2 °C, bound to the cpufreq cooling devices of both clusters and the GPU's devfreq
  cooling device;
- critical trip 70 °C: `hw_protection` shutdown (orderly power-off).

**Cooling states.** Read on the tablet (kernel 7.4.0-rc2-erhai, 22.11.2026). A cooling state *n* is the *n*-th
frequency from the top of the device's table:

| State | CPUs 0-5, kHz | CPUs 6-7, kHz | GPU, MHz |
|---|---|---|---|
| 0 | 3532800 | 4320000 (boost) | 1100 |
| 1 | 3321600 | 4204800 (boost) | 1050 |
| 2 | 3072000 | 4089600 | 967 |
| 3 | 2918400 | 3801600 | 900 |
| 4 | 2745600 | 3513600 | 832 |
| 5 | 2400000 | 3283200 | 734 |
| 6 | 2227200 | 3072000 | 660 |
| 7 | 1996800 | 2841600 | 607 |
| 8 | 1785600 | 2649600 | 525 |
| 9 | 1555200 | 2438400 | 443 |
| 10 | 1363200 | 2246400 | 389 |
| 11 | 1152000 | 1958400 | 342 |
| 12 | 960000 | 1689600 | 222 |
| 13 | 748800 | 1401600 | 160 |
| 14 | 556800 | 1209600 | |
| 15 | 384000 | 1017600 | |

`cpufreq-cpu0` and `cpufreq-cpu6` have 16 states (max_state 15; the big cluster's table includes its two boost
levels), `devfreq-3d00000.gpu` 14 (max_state 13). Every frequency of the stock levels the service used is a state of
these tables: CPUs 0-5 from 3072000 (state 2) to 1152000 (state 11), CPUs 6-7 from 3513600 (state 4) to 1017600
(state 15), the GPU from 1100 (state 0) to 389 MHz (state 10). The cooling maps allow exactly that range: up to
state 11 for CPUs 0-5, the lowest frequency (`THERMAL_NO_LIMIT`, state 15) for CPUs 6-7 and state 10 for the GPU,
the deepest stock level. Below that the critical trip takes over.

`step_wise` moves all three devices together one state per poll, from state 0, while the pack is at or above the
trip and still warming. The stock levels moved the clusters in uneven steps; here the steps are finer and the same
search for the cap at which the pack stops warming happens one table entry at a time. While the pack cools but is
still above 47 °C, one state per poll is given back (the cap stays at least at state 1); once it reads below 47 °C
and falling, the zone's caps are removed at once. A steady reading keeps the cap as it is.

**No fight with the junction zones.** A cooling device bound to several zones takes the deepest state any of them
asks for (`__thermal_cdev_update()` sets the highest target of all instances), so the battery zone and the
`cpu-*`/`gpuss*`/`xo` zones only ever add caps; the lower frequency wins. The cpufreq cooling device sets a
frequency QoS maximum request, which also combines with `scaling_max_freq` (the lower wins).

**Power-off.** The critical trip calls `hw_protection` shutdown: `orderly_poweroff()` runs `/sbin/poweroff`
(`systemctl poweroff` on Ubuntu). The kernel only waits for that program to start, so a power-off that logind refuses
(a "block" shutdown inhibitor) or a shutdown that hangs would leave the tablet running. With
`CONFIG_THERMAL_EMERGENCY_POWEROFF_DELAY_MS=60000` (`erhai-pad3.config`; the defconfig has 0, no fallback) the kernel
powers off by itself 60 s after a critical trip if the system is still up. This applies to every critical trip of
the board (also the 125 °C junction trips). Inhibitors on the tablet on 23.11.2026: only "delay" ones for shutdown
(Unattended Upgrades, Chrome), so today the orderly path would run.

## What it does not do

- No notification to the user. The service could only log; nothing in the system tells the user either way. A
  notification can come from the distribution reading the zone (see the distribution branch).
- No protection while the system is suspended: the zone polls on a freezable workqueue. Then the CPU and GPU are idle;
  the charger firmware keeps its own temperature limits for charging (JEITA).
- Charging current is not limited by this zone. The stock profile has separate charging tables; the fast charging work
  (`fast-charging-no-gauge.md`) has its own temperature limits.

## How to verify

On the tablet, after `boot` (kernel) and `vendor_boot` (device tree) of this branch:

1. Read-only: a zone whose `type` is `battery-thermal`; `trip_point_0_temp` 49000 `passive`, `trip_point_0_hyst`
   2000, `trip_point_1_temp` 70000 `critical`; `cdev*` links to `cpufreq-cpu0`, `cpufreq-cpu6` and
   `devfreq-3d00000.gpu` with `cdev*_trip_point` 0; `temp` equal to `qcom-battmgr-bat/temp` × 100; `policy` step_wise;
   no "battery thermal zone not registered" or "no battery temperature" in `dmesg`.
2. Emulated temperature (`emul_temp`; a real reading at or above the critical trip still wins): rising values from
   49000 cap the cooling states one per 5 s; a falling value below 47000 removes them; writing 0 ends the emulation.
   Never emulate 70000 or more: that powers the tablet off.
3. Real heating with the trip lowered through `trip_point_0_temp` (writable for OF zones, never above 49000), as was
   done for the service at 34 °C.
