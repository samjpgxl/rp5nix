# rp5nix changes

Branch `rp5-optimizations` on top of upstream ROCKNIX `next` at `9fd07e6`.
It targets the Retroid Pocket 5 (SM8250). Each change is limited to SM8250, and
most are limited further to the RP5. The two changes to shared ROCKNIX files
add hook points only; devices that don't use those hooks behave as before.

Nothing here has been run on hardware. Under each change, **Verified** lists
only what could be checked without a device.

## 1. Suspend path (as found)

| Stage | Where | Notes |
|---|---|---|
| Power key | `pm8941_pwrkey` (PMIC PON) → logind `HandlePowerKey=suspend` (`packages/sysutils/systemd/package.mk`) | `input_sense` also calls `rocknix-fake-suspend`, which exits when hardware suspend is enabled. |
| Sleep mode | `sleep.conf` `SuspendState=mem standby freeze` | SM8250 firmware has no PSCI SYSTEM_SUSPEND, so `mem` is s2idle. No SM8250 `030-suspend_mode` quirk exists, so this default applies. `suspendmode off` switches to `rocknix-fake-suspend` (idle-sleep). |
| Hooks | `/usr/lib/systemd/system-sleep/sleep` (`packages/sysutils/sleep/sources/sleep.sh`) | Freezes `games.slice`, rfkill-blocks Wi-Fi, stops the headphone, input, bluetooth and powerstate services, then runs `quirks/platforms/SM8250/sleep.d` (LEDs off, powersave governors). |
| Kernel, Wi-Fi | ath11k (QCA6390, PCIe) | With the interface down, 7.2 ath11k uses its default PM policy, so the chip is powered down and the device is put in D3hot. `qcom-pcie` then sends PME_Turn_Off, enters L2 and drops its interconnect votes (`dw_pcie_suspend_noirq`). The Wi-Fi rfkill in sleep.sh is therefore what lets PCIe power off. |
| Kernel, gamepad | `retroid` serdev driver (0008, 0013) | **Had no PM callbacks.** The MCU stayed powered and kept streaming over UART16 through every sleep. |
| Kernel, speakers | wsa881x amps | They only power down through runtime PM. **A udev rule meant for the MANGMI boards also pinned the RP5 amps on**, which kept both amps enabled at all times, suspend included. |
| Kernel, other | pwm-fan, edt-ft5x06, DSI panel, GPU | Have suspend handling (fan off, touch powered off, panel unprepared, GPU runtime-suspended). |
| Depth | `qcom,rpmh-stats` node in sm8250.dtsi → `/sys/kernel/debug/qcom_stats/{aosd,cxsd,ddr,adsp,cdsp,slpi,...}` | Shows whether the SoC actually reached CX collapse and AOSS sleep. |

Kernel config already had `SUSPEND`, `PM_SLEEP`, `ARM_PSCI_CPUIDLE_DOMAIN` (OSI
cluster idle), `QCOM_STATS`, `QCOM_AOSS_QMP` and `QCOM_RPMH`. What was missing
was `PM_DEBUG`.

### Research limits
- The container's network policy blocks kernel.org, GitHub's web UI, the
  postmarketOS wiki and the GitHub search tools outside this repo. Kernel
  sources were read from `raw.githubusercontent.com/torvalds/linux/v7.2` (the
  pinned version) and from a sparse clone of tag v7.2.
- Web search found no ROCKNIX issue that pins down the RP5 drain. The root
  causes above come from reading the code, not from upstream reports.

## 2. Changes

### 2.1 Kernel: `CONFIG_PM_DEBUG=y` (+ `PM_SLEEP_DEBUG`)
- File: `projects/ROCKNIX/devices/SM8250/linux/linux.aarch64.conf`
- Why: this provides `/sys/power/pm_wakeup_irq`, `pm_debug_messages` and
  `pm_test`, which the diagnostics use. The cost is negligible.
- Verified: the symbols and their dependencies exist in v7.2 `kernel/power/Kconfig`.
  `DPM_WATCHDOG` stays off because it needs `EXPERT`.
- Revert: set `# CONFIG_PM_DEBUG is not set`.

### 2.2 Kernel: power off the gamepad MCU during suspend
- File: `projects/ROCKNIX/devices/SM8250/patches/linux/0020-retroid-gamepad-power-off-mcu-in-suspend.patch`
  (applies after 0008 and 0013; 0008 is unchanged)
- Why: the MCU ran for the whole sleep. Suspend now drives reset low and
  enable low. Resume reruns the exact power-up and init sequence from probe,
  in a work item so resume isn't blocked for about 0.8 s. The input device
  stays registered, so emulators keep their controller.
- Verified: the full 7.2 + `mainline` + SM8250 patch series applies to v7.2,
  and `retroid.o` builds with `W=1` and no warnings (x86 host build; the same
  code is used on arm64). **Untested on hardware**: that holding the MCU in
  reset actually saves power, and that it comes back every time.
- Disable at runtime: `echo N > /sys/module/retroid/parameters/suspend_power_off`.
  To make that persistent, add `options retroid suspend_power_off=0` to
  `/storage/.config/modprobe.d/retroid.conf`. Revert: delete the 0020 patch.

### 2.3 udev: let RP5 speaker amps runtime-suspend
- File: `projects/ROCKNIX/devices/SM8250/filesystem/usr/lib/udev/rules.d/99-wsa881x-nosleep.rules`
- Why: the rule matched every wsa881x by part ID. Its own comment says it was
  meant for the boards where both amps share one powerdown GPIO. The RP5 has
  two separate lines (tlmm 129/127). The rule now requires `MANGMI` in the DT
  model (Pocket Max and Air Y Pro both share line 127).
- Verified: `grep -qs MANGMI` was checked against NUL-terminated model
  strings. **Untested on hardware**: speakers after boot, after idle and
  after resume. A silent speaker after a long idle would point to this change.
- Revert: drop the `PROGRAM=` match.

### 2.4 Suspend hooks, per-sleep log, `rp5-sleepcheck`
- Files:
  - `projects/ROCKNIX/devices/SM8250/filesystem/usr/bin/rp5-sleep`
  - `projects/ROCKNIX/devices/SM8250/filesystem/usr/bin/rp5-sleepcheck`
  - `projects/ROCKNIX/packages/hardware/quirks/platforms/SM8250/sleep.d/pre/090-rp5-sleep`
  - `projects/ROCKNIX/packages/hardware/quirks/platforms/SM8250/sleep.d/post/000-rp5-sleep`
- What the pre hook does, with the post hook undoing it:
  - rfkill-blocks Bluetooth (stopping `bluetooth.service` does not power off the QCA6390 BT side).
  - RP5 only: disables every wakeup source except the power key, the RTC
    (needed for timed tests), alarmtimer and, by default, charger/Type-C. If
    no enabled power key source is found, it skips filtering so the device
    can still be woken.
  - Snapshots counters. After resume it appends one line to
    `/storage/.config/rp5nix/sleep.log`:
    `slept=… bat=A->B% rate=…%/h ok fail aosd=n/s cxsd=n/s ddr=… adsp=… wake=[source irqN(name)]`.
- `rp5-sleepcheck` covers the test side; see §4.
- Verified: shellcheck is clean. The log math, the wakeup filter and restore,
  and the sleepcheck PASS and FAIL paths were run against a fake sysfs tree.
  **Untested on hardware.**
- Settings (`set_setting <key> 0` disables):
  - `rp5.sleep.bluetooth_off`
  - `rp5.sleep.wakeup_filter`
  - `rp5.sleep.wake_on_charger` (0 means only the power key wakes the device)
- Revert: delete the two `sleep.d` files.

### 2.5 Idle-sleep fallback (`suspendmode off`)
- Files:
  - `projects/ROCKNIX/packages/rocknix/sources/scripts/rocknix-fake-suspend` (shared, additive)
  - `projects/ROCKNIX/packages/hardware/quirks/platforms/SM8250/fake-sleep.d/{pre,post}/090-rp5-idle`
  - `projects/ROCKNIX/packages/hardware/quirks/platforms/SM8250/032-idle_sleep`
- Why: ROCKNIX already has an idle-sleep. On the power key it blanks the
  screen, SIGSTOPs games, parks CPUs 1-7, sets powersave governors and turns
  off the LEDs, and after 15 min it shuts down (unless charging). It now also
  runs `fake-sleep.d` hooks, the same layout `sleep.d` uses. SM8250 uses them
  to rfkill Wi-Fi and Bluetooth and restore them on wake, and defaults
  `system.suspend.dpms=1` so the panel and DSI link are actually powered off
  through sway DPMS instead of just being set to brightness 0.
- Enable it with `suspendmode off`. Return to real suspend with
  `suspendmode mem`. Related settings: `rp5.idle.radios_off`,
  `system.suspend.dpms`, `system.suspend.park_cores`,
  `system.shutdown_delay` (seconds).
- Verified: `bash -n`, shellcheck on the new hooks. **Untested on hardware.**

### 2.6 schedutil as the SM8250 default CPU governor
- File: `projects/ROCKNIX/packages/hardware/quirks/platforms/SM8250/010-governors`
- Why: sm8250.dtsi gives every core `capacity-dmips-mhz` and
  `dynamic-power-coefficient`, and `ENERGY_MODEL=y`. Under schedutil, EAS
  places light work on the A55s at low OPPs. ondemand ignores the energy
  model. SM8750 already defaults to schedutil. Existing installs still on the
  old `ondemand` default are switched once (flag `rp5nix.governor.migrated`);
  a governor you set later is kept. Per-game governors in ES are unaffected.
- Verified: shellcheck; the governor is built in (`CPU_FREQ_GOV_SCHEDUTIL=y`).
  **Untested**: menu and idle power, performance.
- Revert: `set_setting system.cpugovernor ondemand`.

### 2.7 RP5 battery design capacity 3850 → 5000 mAh
- File: `projects/ROCKNIX/devices/SM8250/linux/dts/qcom/sm8250-retroidpocket-rp5.dts`
  (the RP5 Visionox DTS includes it; Flip2 and Mini are unchanged)
- Why: the RP5 cell is 5000 mAh. The PM8150B charger patch (0011) sets the
  fast-charge current to half the design capacity when no limit is given, so
  this raises it from 1.925 A to 2.5 A (0.5C). It also corrects
  `charge_full_design`. The capacity % comes from the fuel gauge and is unchanged.
- Verified: both RP5 DTBs were built against v7.2 with the SM8250 patches, and
  decompiled diffs against the old DTBs show this property as the only change.
  **Untested**: charge rate and temperature.
- Revert: remove the `&battery` block.

### 2.8 SM8250-only CI build
- File: `.github/workflows/rp5nix-build-sm8250.yml`
- Why: upstream's `Build` workflow builds and pushes its build container to
  the fork's GHCR, then pulls it without logging in. That needs package
  write access and a public package on a fork. The wrapper reuses
  `build-device.yml` for SM8250 only, with upstream's public
  `ghcr.io/rocknix/rocknix-build:latest` (this fork's Dockerfile is unchanged).
- Verified: YAML parses and the inputs match `build-device.yml`. **Not run.**

### Not changed
- Branding / version string: skipped so the updater and image names stay
  compatible with upstream.
- Wi-Fi power save: `CFG80211_DEFAULT_PS=y` already applies.
- Fan curve: kernel thermal already drives the fan with sane trips, and pwm-fan
  turns it off in suspend.

## 3. Build and flash

**CI (recommended):**
1. Push a tag from the branch:
   `git tag rp5nix-$(date +%Y%m%d) rp5-optimizations && git push origin --tags`
   (or merge the workflow file to the default branch and use Actions →
   "Build SM8250 (rp5nix)" → Run workflow).
2. The fork has no ccache, so the first run takes several hours.
3. Download the `ROCKNIX-image-SM8250-<date>` (`.img.gz`) and
   `ROCKNIX-update-SM8250-<date>` (`.tar`) artifacts.

**Local:** `make docker-SM8250` (needs Docker, 100+ GB free disk and many hours).
The output goes to `target/`.

**Install:**
- Existing ROCKNIX install: copy the `.tar` to `/storage/.update/` and reboot.
- Fresh install: write the `.img.gz` to the SD card with balenaEtcher or
  `gunzip -c … | dd of=/dev/sdX bs=4M`.
- To go back: install an upstream nightly `.tar` the same way.

## 4. On-device test plan

Run these over SSH as root (`ssh root@<ip>`; the default password is `rocknix` unless changed in ES).

1. **Sanity:**
   - `cat /sys/power/mem_sleep` should show `[s2idle]`.
   - `ls /sys/kernel/debug/qcom_stats` should list `aosd cxsd ddr …`.
   - `cat /sys/module/retroid/parameters/suspend_power_off` should print `Y`.
2. **Timed suspend.** Wi-Fi drops while asleep, so detach the test:
   ```
   nohup rp5-sleepcheck -t 120 -n 3 >/dev/null 2>&1 &
   # wait ~8 min, reconnect
   cat /storage/rp5-sleepcheck.txt
   ```
   - PASS needs every cycle to suspend, the SoC to reach **AOSS sleep**, and
     the gamepad and sound card to be present afterwards.
   - `CX collapse … but AOSS never reached` or `never reached CX collapse`
     means a device still holds a vote. Bisect with the toggles in step 5.
   - `woke after Ns` lists the source/IRQ. With `wake_on_charger`, plugging
     in USB counts.
3. **Overnight drain:**
   1. Charge to 80-100%, unplug and suspend with the power key for 8+ hours.
   2. Wake it and run `rp5-sleepcheck --log`. The last line has
      `rate=…%/h` and the aosd/cxsd seconds.
   3. Healthy s2idle on this SoC should come in well under 1%/h, and
      `aosd` seconds should be close to `slept`. Repeat once with Wi-Fi
      enabled and once with it disabled.
4. **Resume functions.** After each wake, check:
   - picture appears within about 2 s
   - buttons, sticks and triggers work after about 1 s (the MCU re-inits after resume)
   - rumble works
   - speakers and headphones work (including after 10+ min of idle first)
   - Wi-Fi reconnects
   - BT controller reconnects
   - LEDs restore
   - the fan spins under load
   - Also suspend in-game and in ES.
5. **A/B toggles** to find which change helps or hurts. Change one at a time, then repeat step 2:
   - `echo N > /sys/module/retroid/parameters/suspend_power_off`
   - `set_setting rp5.sleep.wakeup_filter 0`
   - `set_setting rp5.sleep.bluetooth_off 0`
   - stop a DSP for a test: `echo stop > /sys/class/remoteproc/remoteprocN/state`
     (check `name` first; cdsp and slpi are unused by ROCKNIX)
6. **Idle-sleep fallback:**
   1. `suspendmode off`, then press power.
   2. Screen and radios should go off. Press power again to come back.
   3. `suspendmode mem` restores real suspend.
7. **Charging:** with the charger plugged in and the screen off,
   `cat /sys/class/power_supply/*/current_now` should show about 2.5 A
   charge (negative) at low SoC. Watch `temp`.

## 5. Known risks
- **Gamepad MCU re-init** (2.2): if the MCU needs a longer start than probe
  allows, controls may be dead after resume until reboot. Turn the parameter
  off and report it.
- **WSA amps runtime PM on RP5** (2.3): if the speakers stay silent after
  idle, that indicates the RP5 also needs the pin. Revert the rule change.
- **Wakeup filter** (2.4): it could disable a wakeup the RP5 needs, such as
  a USB host device. It never removes the power key.
- **Charge current** (2.7): slightly more heat while charging.
- **schedutil** (2.6): may add frame-time jitter in some emulators versus a
  fixed max. Per-game "performance" still overrides it.

## 6. Open questions (need the device)
1. Does the RP5 reach AOSS sleep in s2idle at all with Wi-Fi rfkilled? If
   not, which counter stays flat (cxsd vs aosd vs ddr)?
2. Does holding the gamepad MCU in reset measurably cut sleep drain, and does
   it always come back?
3. Real overnight %/h before and after (stock upstream nightly vs this branch).
4. Do the CDSP or SLPI (unused) block or slow AOSS sleep? If so, disabling
   them in the RP5 DTS is the next step.
5. Is any wake reason other than the power key and RTC showing up in `sleep.log`?
