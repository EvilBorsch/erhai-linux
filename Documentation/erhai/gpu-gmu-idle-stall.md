# Adreno 830: one-second GPU stall after a wake without rendering

Found and fixed on the OnePlus Pad 3 (`erhai`, SM8750, Adreno 830, GMU firmware 5.1.25) on 2026-11-22. The same code
is in mainline (`drivers/gpu/drm/msm/adreno/a6xx_gmu.c`, compared with `msm-next` that day), so this is material for
an upstream report. Commits in this tree: `57acf1b89` (report) and `a31555b50` (fix).

## What happened

Starting Firefox took 2 to 4 s, and while it started nothing on the screen moved. `perf record -a` over a start showed
55 to 65 % of all CPU samples in a kworker inside `a6xx_gmu_wait_for_idle()`.

The path:

1. A client opens the render node. Mesa (freedreno and turnip) asks for `MSM_PARAM_TIMESTAMP` while it creates its
   screen. `adreno_get_param()` resumed the GPU with `pm_runtime_resume_and_get()`, read the counter and dropped the
   reference; it did not run `msm_gpu_hw_init()`, which only a submit did.
2. If the client does not draw within the autosuspend delay (66 ms), the GPU suspends with `needs_hw_init` still set.
   `a6xx_gmu_shutdown()` then does its "dummy" `GMU_OOB_GPU_SET` handshake and calls `a6xx_gmu_wait_for_idle()`.
3. With IFPC (`ADRENO_QUIRK_IFPC`, idle level 3) the wait wants the GMU to report power state 3 with GX off. In this
   state the firmware stays at power state 0 with GX on. `spin_until()` busy-loops for `ADRENO_IDLE_TIMEOUT` (1 s) with
   `gmu->lock` held, then the code silently takes the `force_off` path.
4. Any submit in that second blocks in `pm_runtime_get_sync()` behind the suspend: the compositor freezes.

Firefox makes it certain: its GL probe (`/usr/lib/firefox/gfxtest glx -f 1 -w`) opens the device in several helper
processes and never draws. Any GL program that starts while the GPU is asleep and is slow to draw its first frame
can hit it.

## Evidence

- Reproducer: `gfxtest glx -f 1 -w` as the session user; `perf record -a -F 999 -- sleep 4` around it: 93 % of
  samples in `a6xx_gmu_wait_for_idle` (kernel `eb3`).
- Not triggered by: `gfxtest vulkan`, `gfxtest vaapi`, `gfxtest v4l2`, starting Chromium or Nautilus, an idle screen,
  touches (0 samples each).
- With the report of `57acf1b89` (kernel `eb5`), each run of the reproducer logged:

      adreno_gmu 3d6c000.gmu: [drm:a6xx_gmu_stop] *ERROR* GMU did not reach idle level 3 before slumber:
      power state 0, GX on, pwr_clk_status 0x0, cx_busy 0x30000000, hw_init not done

- With the fix (kernel `eb6`): three runs of the reproducer, no report, 0 samples in the function; a Firefox start by
  a tap on its icon, no report. Suspend to idle and resume, Droidloom (Android rendering on the GPU) and GL after
  resume work; no GPU errors in the kernel log.

## The fix here

`a31555b50` makes the timestamp query bring the GPU up the way a submit does: take `gpu->lock`, resume, run
`msm_gpu_hw_init()`, read, put. The suspend that follows is then the ordinary, tested one. The debugfs state dump
already uses the same sequence.

## What is still open (for upstream)

- Why the GMU does not enter IFPC after the dummy handshake when the GPU was never initialized is not known. The
  fix avoids the state; it does not make `a6xx_gmu_shutdown()` handle it. Other wakes without `hw_init` remain
  (`MSM_PARAM_SYSPROF` with value 2 resumes the GPU the same way) and would still spin for a second; the report from
  `57acf1b89` makes that visible.
- Whether other IFPC GPUs (A7xx with the quirk, other A8xx) behave the same was not tested; only this tablet was.
- A busy loop of a second with a mutex held is harsh for a wait that can time out; a sleeping poll would at least
  not burn a core.

## Tools

`tools/tab/gmu-spin-cmd.sh`, `perf-launch.sh`, `perf-tap.sh` in the workspace (`D:\androidlinux\tools\tab`); logs
`device-logs/perf-launch-firefox.txt`, `gmu-spin-*.txt`, `eb5-gmu-report.txt`, `eb6-gmu-fix.txt`.
