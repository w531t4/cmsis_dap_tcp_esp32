# TODO

This list tracks upstream follow-up work that remains after the validated
multi-instance CMSIS-DAP TCP/JTAG and standalone UART bridge work.

## 1. Make LED polarity instance-configurable

Status: Resolved.

Why it matters:

- `struct cmsis_dap_gpio_config` can already carry a per-instance LED pin.
- `CONFIG_ESP_DAP_LED_ACTIVE_HIGH` previously controlled polarity globally at
  compile time.
- Multiple instances can use separate LED pins and separate LED polarities once
  callers populate the runtime GPIO config.

Implementation:

- [x] Add an LED polarity field to `struct cmsis_dap_gpio_config`.
  - Resolved by adding `led_active_high`.
- [x] Preserve the existing Kconfig polarity as the default for `NULL` config.
  - Resolved in `get_effective_gpio_config()` by copying
    `CONFIG_ESP_DAP_LED_ACTIVE_HIGH` into `led_active_high` for the default
    standalone path.
- [x] Update `LED_CONNECTED_OUT()` to use the active task's runtime polarity.
  - Resolved by routing the output level through `GPIO_LED_ACTIVE_HIGH`.
- [x] Update LED setup/off state to use the active task's runtime polarity.
  - Resolved by routing the setup-time inactive level through
    `GPIO_LED_ACTIVE_HIGH`.

Validation:

- [x] Build default standalone configuration.
  - Validated: `idf.py build` completed successfully.
- [x] Validate existing single-instance OpenOCD flow still works.
  - Validated: existing single-instance OpenOCD flow completed successfully.
- [x] Validate one instance can use a runtime-configured LED pin.
  - Validated: single-instance runtime LED pin test completed successfully.
- [x] Validate one instance can use runtime LED polarity by confirming active-high
  and active-low settings produce opposite GPIO levels for the same logical LED
  state.
  - Validated: active-high and active-low runtime polarity settings produced
    the expected opposite GPIO levels for the same logical LED state.
- [x] Validate two instances can use separate LED pins.
  - Validated: two runtime-configured instances used separate LED pins successfully.
- [x] If hardware is available, validate active-high and active-low LEDs can coexist.
  - Validated: active-high and active-low LED polarity settings can coexist across instances.

## 2. Decide whether CMSIS-DAP UART command support is in scope

Status: Open, currently out of scope.

Why it matters:

- This is not the standalone TCP UART bridge.
- `main/UART.c` implements CMSIS-DAP UART commands and still uses file-scope
  state, buffers, indices, and error flags.
- The repository only includes `Driver_USART.h`; it does not include a concrete
  CMSIS USART driver implementation to validate this feature.

Possible resolution:

- Keep `DAP_UART` disabled and document that CMSIS-DAP UART commands are not
  multi-instance-safe.
- Or, if a real USART driver is added, move UART command state into an
  instance-local context and route callbacks to the owning instance.

Validation needed:

- Build default `DAP_UART == 0` configuration.
- If enabling this feature later, add a real `Driver_USART` implementation or
  fixture before refactoring.
- Validate one instance first, then validate two instances.

## 3. Decide whether SWO support is in scope

Status: Open, currently out of scope.

Why it matters:

- `main/SWO.c` still uses file-scope trace state, buffers, indices, timestamp
  state, and transfer flags.
- SWO is disabled in the current configuration and has not been runtime-tested.

Possible resolution:

- Keep SWO disabled and document that SWO is not multi-instance-safe.
- Or move SWO trace state into an instance-local context and add runtime tests
  using hardware that can exercise SWO.

Validation needed:

- Build default SWO-disabled configuration.
- If enabling SWO later, validate single-instance SWO before attempting
  multi-instance behavior.

## 4. Consider per-instance UART bridge serial framing

Status: Implemented, validation pending.

Why it matters:

- The standalone TCP UART bridge now accepts runtime config for port, keepalive
  timeout, UART number, TX/RX pins, and baud rate.
- Data bits, parity, and stop bits are still selected by compile-time Kconfig.
- Multiple UART bridge instances can coexist today when they use the same
  framing, for example `8N1`.

Implementation:

- [x] Extend `struct uart_bridge_config` with data bits, parity, and stop bits.
  - Implemented by adding `data_bits`, `parity`, and `stop_bits`.
- [x] Preserve Kconfig values as defaults for `NULL` config.
  - Implemented in `default_uart_bridge_config`.
- [x] Configure the ESP-IDF UART driver from the runtime config.
  - Implemented by passing the runtime values into `uart_config_t`.

Validation needed:

- [x] Build default standalone configuration.
  - Validated: default standalone configuration builds successfully.
- [x] Validate existing UART bridge behavior.
  - Validated: existing UART bridge behavior works successfully.
- [x] Validate two UART bridge instances with the same framing.
  - Validated: two UART bridge instances with the same framing work successfully.
- [ ] If hardware is available, validate two UART bridge instances with different
  framing.
  - Not validated: suitable hardware/test setup is not available.

## 5. Review global capability toggles

Status: Resolved by review; no code change in this section.

The remaining globals below do not block the validated multi-instance JTAG,
LED, and standalone UART bridge behavior.

Why it matters:

- Several capabilities remain compile-time global: JTAG support, SWD support,
  nTRST support, nRESET support, LED support, packet size, and timing constants.
- Some of these are acceptable as firmware-wide capabilities.
- Others may become limiting if future users need truly heterogeneous target
  configurations.

Intentionally global:

- [x] Protocol support toggles:
  - `CONFIG_ESP_DAP_JTAG_SUPPORTED`
  - `CONFIG_ESP_DAP_SWD_SUPPORTED`
  - These describe which protocol engines are compiled into the firmware image.
    Keeping them firmware-wide is reasonable as long as builds enable the
    superset needed by deployed instances.
- [x] Feature availability toggles:
  - `CONFIG_ESP_DAP_JTAG_NTRST_SUPPORTED`
  - `CONFIG_ESP_DAP_NRESET_SUPPORTED`
  - `CONFIG_ESP_DAP_LED_SUPPORTED`
  - These describe whether support code is compiled in. The instance-specific
    reset pins, LED pin, and LED polarity are already carried by runtime config.
- [x] Protocol buffer sizing:
  - `CONFIG_ESP_DAP_TCP_MAX_PKT_SIZE`
  - This controls protocol buffer sizing. Making it per-instance would add
    allocation and lifetime complexity without a demonstrated
    heterogeneous-target use case.
- [x] TCP keepalive support:
  - `CONFIG_ESP_DAP_TCP_USE_KEEPALIVE`
  - This controls whether TCP keepalive support is compiled into the firmware.
    Instances can still opt out with `disable_keepalive` and can provide their
    own keepalive timeout when support is compiled in.
- [x] Standalone UART bridge feature gate:
  - `CONFIG_ESP_UART_BRIDGE_ENABLED`
  - This controls whether the standalone app compiles and starts the UART
    bridge feature. Runtime `uart_bridge_config` handles the per-instance
    UART bridge details once the feature is compiled in.

Remaining global but needs to be addressed:

- [x] Low-level timing constants:
  - Tracked separately in item 6.

Global but left alone:

- [x] CMSIS-DAP UART command support:
  - Tracked separately in item 2 because `UART.c` has file-scope state beyond a
    simple capability toggle.
- [x] SWO support:
  - Tracked separately in item 3 because `SWO.c` has file-scope trace state
    beyond a simple capability toggle.

## 6. Leftovers

Status: Resolved.

Implementation:

- [x] Add runtime `io_port_write_cycles` config with Kconfig fallback.
- [x] Add runtime `delay_slow_cycles` config with Kconfig fallback.
- [x] Route `IO_PORT_WRITE_CYCLES` through the active instance config.
- [x] Route `DELAY_SLOW_CYCLES` through the active instance config.

Validation needed:

- [x] Build default standalone configuration.
  - Validated: default standalone configuration builds successfully.
- [x] Validate existing OpenOCD flow still works with Kconfig timing defaults.
  - Validated: existing OpenOCD flow works with Kconfig timing defaults.
- [x] Validate existing multi-instance flow still works when runtime timing
  config is populated with Kconfig-equivalent values.
  - Validated: existing multi-instance flow works with runtime timing fields set
    to the Kconfig timing defaults.
- [x] Validate one instance with runtime timing values that differ from Kconfig.
  - Validated: one instance works with runtime timing values that differ from
    the Kconfig defaults.
