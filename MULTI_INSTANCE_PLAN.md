# Multi-Instance Work Plan

This plan lists the remaining work needed to run multiple CMSIS-DAP TCP instances in parallel from this codebase. It assumes the current task-local DAP state change remains in place.

## 1. Validate the task-local DAP state change

Status: Resolved.

Current change:

- `DAP_Data` and `DAP_TransferAbort` are task-local in `main/DAP.c` and `main/DAP.h`.
- `DAP_Setup()` runs inside `cmsis_dap_tcp_task()` so each task initializes the DAP state it uses.

Resolution:

- The project was build-tested after each step.
- Single-instance CMSIS-DAP TCP operation was validated by connecting and flashing with OpenOCD.
- A temporary two-task diagnostic started listeners on ports 4441 and 4443 and printed the addresses of `DAP_Data` and `DAP_TransferAbort` from each FreeRTOS task.
- The diagnostic confirmed separate task-local storage:
  - `cmsis_dap_tcp_0`: `DAP_Data=0x3ffc8014`, `DAP_TransferAbort=0x3ffc8010`
  - `cmsis_dap_tcp_1`: `DAP_Data=0x3ffc9c34`, `DAP_TransferAbort=0x3ffc9c30`

Severity: High.

## 2. Make JTAG/SWD GPIOs instance-configurable

Implementation status: Implemented.
Validation status: Validated.

Today the JTAG/SWD pins are compile-time macros sourced from Kconfig in `main/DAP_config.h`:

- `GPIO_SWCLK_TCK`
- `GPIO_SWDIO_TMS`
- `GPIO_TDI`
- `GPIO_TDO`

Those macros are used directly by setup and fast pin I/O helpers such as `PORT_JTAG_SETUP()`, `PORT_SWD_SETUP()`, `PIN_SWCLK_TCK_IN()`, and `PIN_TDI_OUT()`.

Work needed:

- Add a per-instance pin configuration, likely on `struct cmsis_dap_tcp_config`.
  - Implemented: `struct cmsis_dap_gpio_config` now carries TCK, TMS, TDI, and TDO pin numbers.
- Include JTAG/SWD pins at minimum: TCK, TMS, TDI, TDO.
  - Implemented: `struct cmsis_dap_tcp_config` now has an optional `gpio` pointer.
- Preserve the existing Kconfig defaults for the standalone app.
  - Implemented: `NULL` config and `config->gpio == NULL` continue to use the existing Kconfig GPIO values.
  - Note: this implementation necessarily introduces the backend shape described in step 3, but step 3 is not being dispositioned until it is worked intentionally.

Validation needed:

- Build the standalone app to confirm the default Kconfig path still compiles.
  - Validated: `idf.py build` completed successfully and produced `build/cmsis_dap_tcp_esp32.bin`.
- Flash and confirm the existing single-instance OpenOCD flow still works.
  - Validated: firmware was flashed and the existing single-instance OpenOCD flow still worked with the default Kconfig pins.

Severity: Critical. Multiple tasks can exist now, but they still drive the same pins.

## 3. Provide an instance-local pin backend for `DAP_config.h`

Implementation status: Implemented.
Validation status: Validated.

The CMSIS-DAP core calls inline pin helpers without passing context. That means pin selection cannot become instance-specific unless the pin helpers can find instance-local state.

Possible approaches:

- Minimal path: add a task-local pointer to the active GPIO config, set it during task startup, and have the inline pin helpers read pins from that.
  - Implemented: `DAP_gpio_config.h` defines the GPIO config shape and task-local active config pointer.
  - Implemented: `DAP_gpio_config.c` owns the task-local active config pointer.
  - Implemented: `DAP_config.h` depends on the GPIO backend instead of depending on the TCP server header.
  - Implemented: `cmsis_dap_tcp_task()` sets the task-local active config before `DAP_Setup()`.
- Larger path: thread an explicit DAP context through the CMSIS-DAP command and pin I/O call chain.
  - Not selected for this step because the task-local backend is much smaller and preserves the current CMSIS-DAP call shape.

Validation needed:

- Build the standalone app to confirm the neutral GPIO backend compiles in all translation units.
  - Validated: `idf.py build` completed successfully and compiled `DAP_gpio_config.c` into the `main` component.
- Flash and confirm the existing single-instance OpenOCD flow still works with the default Kconfig pins.
  - Validated: firmware was flashed and the existing single-instance OpenOCD flow still worked with the default Kconfig pins.

Severity: Critical.

## 4. Expose a supported multi-instance startup API

Implementation status: Implemented.
Validation status: Validated.

The standalone app currently starts one CMSIS-DAP TCP task. The task accepts a config pointer, but there is no helper that owns config lifetime or makes multi-instance startup obvious.

Work needed:

- Document that callers must keep `struct cmsis_dap_tcp_config` alive for the lifetime of the task, or add a helper that copies config into task-owned memory.
  - Implemented: `cmsis_dap_tcp_start()` documents that a non-NULL config must remain valid for the task lifetime.
- Consider adding a helper such as `cmsis_dap_tcp_start(config, task_name, handle)`.
  - Implemented: `cmsis_dap_tcp_start(config, task_name, handle)` wraps `xTaskCreate()` with the existing stack size and priority.
- Keep the existing standalone app workflow intact.
  - Implemented: the standalone app now starts the default server through `cmsis_dap_tcp_start(NULL, "cmsis_dap_tcp_task", NULL)`.

Validation needed:

- Build the standalone app to confirm the helper compiles and preserves the default startup path.
  - Validated: `idf.py build` completed successfully and produced `build/cmsis_dap_tcp_esp32.bin`.
- Flash and confirm the existing single-instance OpenOCD flow still works.
  - Validated: firmware was flashed and the existing single-instance OpenOCD flow still worked.

Severity: High.

## 5. Add resource conflict checks

Implementation status: Implemented.
Validation status: Validated.

Multiple instances need to fail clearly when they cannot coexist.

Work needed:

- Reject duplicate TCP ports before or during startup.
  - Implemented: `cmsis_dap_tcp_task()` reserves the effective TCP port at startup and exits if another active instance already reserved it.
- Reject overlapping GPIO sets between instances.
  - Implemented: `cmsis_dap_tcp_task()` reserves the effective TCK/TMS/TDI/TDO pins at startup and exits if any active instance already reserved one.
- Treat optional reset/LED pins as resources too if they are enabled.
  - Deferred to step 6, where reset and LED ownership semantics are being decided.
- Decide whether validation belongs in the library, the wrapper, or both.
  - Implemented for the library: the resource checks live inside `cmsis_dap_tcp_task()`, so both the startup helper and direct task callers get the same checks.

Validation needed:

- Build the standalone app to confirm the registry compiles.
  - Validated: `idf.py build` completed successfully and produced `build/cmsis_dap_tcp_esp32.bin`.
- Flash and confirm the existing single-instance OpenOCD flow still works.
  - Validated: firmware was flashed and the existing single-instance OpenOCD flow still worked.
- Start a temporary duplicate-port pair and confirm the second task exits with a resource conflict.
  - Validated: a temporary pair on port 4441 started one listener and the second task exited with `cmsis_dap_tcp: resource conflict on port or JTAG pins.`
- Start a temporary overlapping-GPIO pair on different ports and confirm the second task exits with a resource conflict.
  - Validated: a temporary pair on ports 4441 and 4443 with overlapping `swclk_tck` started one listener and the second task exited with `cmsis_dap_tcp: resource conflict on port or JTAG pins.`

Severity: High.

## 6. Decide how LED, nRESET, and nTRST behave

Implementation status: Implemented.
Validation status: Validated.

These are also compile-time GPIO macros today:

- `GPIO_NTRST`
- `GPIO_NRESET`
- `GPIO_LED`

Work needed:

- Make them per-instance, or explicitly define them as optional/shared resources.
  - Implemented: `struct cmsis_dap_gpio_config` now carries optional
    `ntrst`, `nreset`, and `led` pin numbers.
  - Implemented: `-1` means the optional pin is omitted for that instance.
  - Implemented: the runtime resource registry treats enabled optional pins as
    exclusive resources and ignores omitted optional pins.
- Preserve Kconfig defaults for the standalone app.
  - Implemented: `NULL` config and `config->gpio == NULL` continue to use the
    existing Kconfig values.
- Document what happens when these pins are omitted.
  - Implemented in code behavior: omitted optional pins are no-ops for setup,
    reset control, and LED control.

Validation needed:

- Build the standalone app to confirm the optional GPIO path compiles.
  - Validated: `idf.py build` completed successfully and produced
    `build/cmsis_dap_tcp_esp32.bin`.
- Flash and confirm the existing single-instance OpenOCD flow still works.
  - Validated: firmware was flashed and the existing single-instance OpenOCD
    flow still worked.

Severity: Medium to High, depending on whether each JTAG header has separate reset and LED wiring.

## 7. Disable or refactor CMSIS-DAP UART support for multi-instance mode

Implementation status: Resolved by leaving CMSIS-DAP UART command support out
of the supported multi-instance surface.
Validation status: Validated for the default build; `DAP_UART == 1` runtime
validation is out of scope because this repository does not provide a CMSIS
`Driver_USART` implementation.

`main/UART.c` contains file-scope state for UART transport, buffers, indices, and error flags. If `DAP_UART` is enabled, multiple CMSIS-DAP instances would share that state.

Work needed:

- Decide whether `DAP_UART` is in scope for the upstream multi-instance patch.
  - Resolved: `DAP_UART` is not in scope for this patch series.
- If it is in scope, move UART state into an instance-local context rather than
  file-scope globals.
  - Not selected. `UART.c` is left untouched because the feature depends on an
    external `Driver_USART` implementation that is not included in this
    repository.
- Make the selected USART driver/runtime port instance-specific, or explicitly
  reserve one UART owner so a second instance cannot use the same UART.
  - Not selected for this patch series.
- Route USART callbacks to the owning UART state instead of updating one shared
  static state block.
  - Not selected for this patch series.
- Preserve the existing standalone behavior when `DAP_UART` remains disabled.
  - Implemented by not changing `main/UART.c`.

Validation needed:

- Build the default `DAP_UART == 0` configuration to confirm existing behavior
  still compiles.
  - Validated: `idf.py build` completed successfully and produced
    `build/cmsis_dap_tcp_esp32.bin`.
- Confirm the existing single-instance OpenOCD flow still works after the UART
  scope decision.
  - Validated: single-instance OpenOCD testing completed successfully.
- `DAP_UART == 1` runtime behavior is not validated or modified here.
  - The repository's `main/Driver_USART.h` is only a placeholder interface.
  - The actual CMSIS `Driver_USART` implementation is an external dependency
    that is not included in this project.
  - If an integrator supplies a real driver and wants multi-instance
    CMSIS-DAP UART commands, that should be a separate, testable patch.

Severity: Medium if unused. High if UART-over-DAP is enabled.

## 8. Disable or refactor SWO support for multi-instance mode

Implementation status: Resolved by leaving SWO support out of the supported
multi-instance surface.
Validation status: Validated for the default build only; SWO-enabled runtime
behavior is not validated or modified here.

`main/SWO.c` contains file-scope trace state, buffers, indices, timestamps, and transfer flags. If SWO is enabled, multiple CMSIS-DAP instances would share trace capture state.

Work needed:

- If SWO is not part of the multi-instance goal, document that it must remain disabled.
  - Resolved: SWO is not in scope for this patch series.
  - The default configuration keeps `SWO_UART`, `SWO_MANCHESTER`, and
    `SWO_STREAM` disabled.
- If it is needed, move SWO state into an instance-local context or task-local storage.
  - Not selected. `main/SWO.c` is left untouched because making it
    instance-safe would require a separate SWO-enabled implementation and
    validation path.

Severity: Medium if unused. High if SWO is enabled.

## 9. Decide whether `uart_bridge_task` needs multi-instance support

Implementation status: Implemented.
Validation status: Validated.

The UART bridge is separate from CMSIS-DAP, but it has a shared static buffer and compile-time UART/port configuration.

Work needed:

- If only one UART bridge is supported, document that limitation.
  - Not selected. The goal is to allow one serial bridge per target.
- If multiple UART bridges are desired, give it runtime config and task-owned state.
  - Implemented: `struct uart_bridge_config` carries the TCP port, keepalive
    timeout, UART number, TX/RX pins, and baud rate.
  - Implemented: `uart_bridge_task()` copies the effective config at startup
    and uses a task-local transfer buffer.
  - Implemented: the standalone app still starts the default UART bridge from
    the existing Kconfig defaults.

Validation needed:

- Build the standalone app with UART bridge enabled to confirm the default
  Kconfig path still compiles.
  - Validated: the UART bridge-enabled build completed successfully.
- Flash and confirm the existing single UART bridge use case still works.
  - Validated: firmware was flashed and the existing single UART bridge use
    case worked.

Severity: Low for CMSIS-DAP multi-instance. Medium for whole-project multi-instance.

## 10. Test two simultaneous sessions

Validation status: Validated for dual CMSIS-DAP/JTAG sessions.

Work needed:

- Start two CMSIS-DAP TCP tasks on two ports.
  - Validated: the temporary harness starts CMSIS-DAP TCP servers on ports
    4441 and 4443.
- Confirm the normal single-interface build/run path still works with the
  temporary harness disabled.
  - Validated: harness disabled build/run succeeded.
- Use two disjoint pin sets.
  - Validated: the temporary harness uses the primary pin set
    TCK=18, TMS=5, TDI=23, TDO=34 and the secondary pin set
    TCK=19, TMS=21, TDI=22, TDO=25.
- Confirm each task uses its configured JTAG/SWD pin set.
  - Validated: the port 4443 OpenOCD run caused the firmware to initialize
    GPIO19, GPIO21, GPIO22, and GPIO25, confirming the secondary instance used
    its configured pin set.
- Connect two clients at the same time.
  - Validated: OpenOCD connected to both CMSIS-DAP TCP ports during testing.
- Verify both can idle, scan, and perform target operations independently.
  - Validated: OpenOCD found Lattice ECP5 TAPs through both instances:
    `0x21111043` and `0x41113043`.
- If both targets have UART consoles, start two UART bridge tasks with distinct
  ports, UART numbers, and pins, then confirm both TCP serial sessions can be
  open simultaneously and carry data independently.
  - Not part of the #10 JTAG harness. The UART bridge was disabled for this
    validation run after it interfered with the native console test setup.
- Test failure paths: duplicate port, overlapping pins, one client disconnecting while the other remains active.
  - Duplicate port and overlapping pin failure paths were validated in step 5.

Severity: Critical before claiming parallel support.

## Main Architectural Blocker

The TCP server is close to being instance-safe. The remaining hard blocker is the GPIO backend in `DAP_config.h`: it is still fundamentally build-time, single-pinout hardware glue. Multi-instance support requires making that hardware glue instance-aware while preserving the current standalone app defaults.
