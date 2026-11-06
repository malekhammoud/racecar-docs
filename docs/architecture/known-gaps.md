# Implementation notes

The architecture pages describe the intended behavior and the normal driving
path. This page records the places where the source in this revision differs:
inactive checks, missing initialization, and behaviors that can surprise a
driver, tester or reviewer.

This is a short list of significant findings, not a complete code audit.

## Front controller

- **The HVIL branch is inactive.** `hvil_interrupt` is a local `false` value,
  so the governor's software shutdown branch can never fire. The physical
  shutdown circuit is independent and still removes power.
- **Missing inverter CAN can crash the running state.** In `RUNNING`, the
  governor dereferences `Inv1_ActualValues2` and `Inv2_ActualValues2` without
  checking whether the optional values are present. The motor layer sets
  `NoInv1Can` / `NoInv2Can`, but this display path does not guard the
  dereference.
- **The governor stops checking the accumulator after precharge.**
  `WAIT_START_MOTOR` and `STARTING_MOTORS` do not re-check an accumulator
  fault. A fault that appears after `STARTING_HV` may not move the governor to
  `ERROR`.
- **Low state of charge does not block precharge.** The low-SOC check is
  commented out while the BMS SOC estimate is being validated.
- **The HSD overcurrent alert never fires.** `HSD::Update_10Hz()` writes the
  member `_channels` array, but `HasOverCurrent()` reads a separate file-scope
  `channels` array that is never written. Current readings are also not sent
  over CAN.
- **A welded contactor can stall startup silently.** `STARTUP_ENSURE_OPEN`
  waits for every feedback to read open and has no fault branch. A contactor
  that is already closed when HV starts keeps the accumulator waiting forever.
- **Some waits have no timeout.** Missing contactor feedback stalls the
  accumulator, and a missing `LvStatus` stalls the motor handoff. The 500 ms
  pack-voltage timeout is the main exception.
- **Torque assistance is limited.** Torque vectoring is disabled by default and
  the traction-control scale factor is fixed at `1`, so neither steering-based
  vectoring nor slip control is active.

## LV controller

- **The application does not boot as written.** `main()` never calls
  `Initialize()`, so clocks, GPIO, ADC, CAN and timers are not configured
  before the application objects are used.
- **The scheduler does not advance time.** `scheduler::run(GetTick())` captures
  one timestamp and never refreshes it inside the loop. Registered tasks
  cannot run repeatedly on a real schedule.
- **The motor-controller precharge step does not wait.** The LV power governor
  advances out of `PWRUP_MOTOR_CONTROLLER_PRECHARGING` while the motor
  controller is **not** `ON`, so it continues during or before precharge
  instead of waiting for completion.
- **Constructor arguments are swapped.** `Motor_Controller` expects
  `(precharge, main, switch)`, but the application constructs it with
  `(main, precharge, switch)`. The two enable lines are therefore controlled
  in the wrong order.
- **The fan PWM starts too early.** The global `Fans` object calls
  `PWMOutput::Start()` during construction, before `MX_TIM2_Init()` runs.
- **DCDC control is disabled.** The DCDC state exists, but the control calls
  are commented out and the module is not used by the current sequence.
- **The fan enable outputs are forced low.** `UpdateEnabled()` always writes
  `false` to both fan enable pins to avoid blowing a fuse; only the PWM duty is
  updated.

## Dashboard

- **There is no normal path into `ERROR`.** No screen transition selects it,
  so the error screen and its hold-ENTER shutdown action are not reachable in
  the ordinary driver flow.
- **Two fault bits are ignored.** The dashboard never reads
  `DashCommand.Errored` or `DashCommand.HvShutdown`.
- **One running-screen guard checks the wrong message.** The current arc uses
  the `BmsBroadcast` optional to decide whether to read `Pack_State`. If a BMS
  temperature frame has arrived but a pack frame has not, it dereferences an
  empty optional.
- **The brake threshold is tiny.** The motor-start screen uses 0.20 percentage
  points, not 20%. Raw values 0 and 1 fail; 2 and above pass. See
  [Dashboard](dashboard.md).
- **The selected profile does nothing.** The FC stores the dashboard's profile
  but never reads it back, so all five profiles currently behave the same.
- **Some displayed values are misleading.** The speedometer is labeled `km/h`
  but receives a value in `mph` without converting it. The HV voltage label can
  exceed 255 V incorrectly because the FC truncates the float pack-voltage
  reading to 8 bits before sending it.
- **No message-age check.** A disconnected ECU leaves its last values on the
  running screen. The `FC: OK` label is constant and the green CAN indicator
  does not monitor traffic.
- **The running screen has no shutdown action.** The only ordinary way back to
  profile selection is the front controller's global `Reset`.

## TMS

- **Fan start is never called.** `FanController::Start()` would start the PWM
  timer and set the initial duty, but the application only calls `Update()`.
  `MX_TIM4_Init()` configures the timer without starting the output, so the
  periodic duty writes alone may not produce an active fan signal.
- **`TMSValues` overflows.** Each field is
  `static_cast<uint8_t>(temperature × 50)`. Above about 5 °C the value exceeds
  255 and wraps, so this diagnostic frame is not a reliable Celsius reading.
- **Temperature filtering starts at zero.** The 20-sample moving average is
  zero-filled and always divides by 20, so the first samples read far too low.
  A steady 40 °C input needs twenty updates to read 40 °C.
- **There is no sensor fault detection.** Open and short circuits are not
  distinguished, and readings outside the lookup table clamp to its endpoints.
- **Both debug LEDs toggle.** Red is not a fault-only indicator on this board.

## CAN and shared code

- **Reception is a latest-value mailbox.** `GetRx...()` returns the last frame
  of that type with no age check. A sender that stops transmitting still looks
  valid. Only the accumulator adds a timeout, and only for pack voltage.
- **Generated packers do not clamp.** Values are rounded and cast to the DBC
  width, not limited to the signal's declared minimum and maximum.
- **State numbers have gaps.** The FC governor state value `1` is unused;
  `WAIT_DRIVER_SELECT` is `2`. Do not assume states are contiguous.

## Tooling

- **The static-analysis helper looks in the wrong directories.**
  `scripts/static_analysis/static_analysis.sh` and the CI step check `src/` and
  `include/`, but the production projects use `Src/` and `Inc/`. On a
  case-sensitive filesystem they can skip every production project. The local
  helper also only looks at staged changes.
- **CAN interface names are not consistent.** The monitor scripts use `can1`
  for vehicle and `can0` for powertrain; the DAQ setup uses `can0` for vehicle
  and `can1` for powertrain. Verify the wiring or udev rules before capturing.
- **The DAQ deployment is ambiguous.** `scripts/daq/setup_pi.sh` installs the
  cloud logger, while `scripts/daq/setup.sh` installs local InfluxDB and
  Grafana. Confirm which one the Pi actually uses.
- **CAN Flash pin mapping is marked unverified.** `scripts/canflash/constants.py`
  carries a warning that the Raspberry Pi GPIO pins in the table are not
  correct yet.
- **CAN Flash is only wired up on the TMS.** The web interface lists the front
  controller, LV controller and TMS, but the front controller's
  `CheckCanFlash()` call is commented out and the LV controller defines
  `check_can_flash()` without calling it. The TMS application does call its
  handler.
- **The `fc_common` test is not run by CI.** `lib/fc_common/test/` contains a
  vehicle-dynamics test, but the library has no `platformio.ini` or `native`
  environment, so the test runner skips it.
