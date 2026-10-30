# LVD

## Overview

This Software Building Block demonstrates the Renesas FSP Low Voltage Detection (LVD) HAL. The supplied reference design uses LVD0 to reset the MCU when the input voltage falls below its configured threshold and LVD1 to generate a maskable interrupt when the input voltage rises above its configured threshold.

This document is limited to:

- FPB-RA0L1
- FPB-RA2E3
- FPB-RA4E2
- FPB-RA6E2
- FPB-RA8E1

For all five boards, the supplied README states that terminal log output is not supported by this example. LED1 indicates an LVD0 reset and LED2 toggles when LVD1 generates an interrupt.

## 1. What use case does this SBB demonstrate?

The SBB demonstrates two voltage-monitoring behaviors:

1. **Low-voltage reset with LVD0.** When the input voltage drops below the configured LVD0 threshold, LVD0 resets the MCU. After the input voltage is raised above the LVD0 threshold and the board powers up, the application checks the LVD0 reset flag and turns on LED1 when that reset is detected.
2. **Rising-voltage notification with LVD1.** LVD1 generates a maskable interrupt when the input voltage rises from below to above the configured LVD1 threshold. The callback sets a software flag. The reference application detects the flag, toggles LED2, clears the LVD1 status, and clears the software flag.

The README specifically notes that LVD1 does not generate another interrupt merely because voltage that is already above the threshold is increased further.

## 2. What does the SBB core code do?

The reusable LVD implementation is contained in `lvd_ep.c` and `lvd_ep.h`.

It:

- Checks whether the previous MCU reset was caused by LVD0.
- Handles the different LVD0 reset-status register path used for FPB-RA0L1 versus the other relevant targets.
- Clears the LVD0 reset flag after reading it on targets where software clearing is required.
- Opens the FSP LVD1 instance using `R_LVD_Open()`.
- Clears LVD1 status using `R_LVD_StatusClear()`.
- Closes the LVD1 instance using `R_LVD_Close()`.
- Implements `lvd1_callback()` and sets `g_lvd1_status` when the callback reports that voltage is above the threshold.

LVD0 is not opened by `lvd_init()`. The application source states that LVD0 is enabled through the OFS1 register configuration, with the voltage-detection circuit and voltage level configured in the FSP BSP properties.

## 3. What is demo/reference application code versus reusable SBB code?

### Reusable SBB code

- `lvd_ep.c`
- `lvd_ep.h`

These files provide the reusable LVD0 reset-detection and LVD1 driver/callback interface.

### Demo/reference application code

- `lvd_example_app.c`
- `lvd_example_app.h`

The reference application:

- Initializes the UART console before attempting application prints.
- Retrieves the FSP version and contains the example banner/guide text.
- Verifies that the BSP reports on-board LEDs.
- Calls `lvd0_reset()` and turns LED1 on when an LVD0 reset is detected.
- Opens LVD1 using `lvd_init()`.
- Monitors `g_lvd1_status`.
- Toggles LED2 when an LVD1 threshold event has occurred.
- Calls `lvd_status_clear()` after handling the event.
- Provides application-level error handling and breakpoint behavior.

`lvd_example_app_run()` is the single reference-application entry point called by `hal_entry.c` according to the source comments.

## 4. Which public functions should I look at first?

### `bool lvd0_reset(void)`

Checks whether the previous MCU reset was caused by the LVD0 monitor. It returns `true` when the LVD0 reset condition is found and `false` otherwise.

### `fsp_err_t lvd_init(void)`

Opens the generated LVD1 instance using `g_lvd_ctrl` and `g_lvd_cfg`.

### `fsp_err_t lvd_status_clear(void)`

Clears the LVD1 status through the FSP LVD API.

### `void lvd_deinit(void)`

Closes the LVD1 instance if it is open.

### `void lvd1_callback(lvd_callback_args_t * p_args)`

This callback is implemented in `lvd_ep.c`. When `current_state` is `LVD_CURRENT_STATE_ABOVE_THRESHOLD`, it sets `g_lvd1_status` to notify the foreground application.

### `void lvd_example_app_run(void)`

Review this function to understand how the reference application connects LVD0 reset detection, LVD1 interrupt handling, LED indication, and application error handling.

## 5. What configuration assumptions were made for the reference boards?

### Common hardware

The README requires:

- One supported Renesas RA board.
- One variable DC power supply.
- Jumper wires.
- An external capacitor if the power source produces unwanted voltage spikes.
- An external SEGGER J-Link for boards other than FPB-RA4E2 and FPB-RA6E2.

The DC supply positive output is connected to an on-board VCC pin and its negative output to an on-board VSS/GND pin.

### Board-specific thresholds and debug connection

| Board | External debug/configuration | LVD1 rising threshold | LVD0 falling threshold |
|---|---|---:|---:|
| FPB-RA0L1 | Place J9 on pins 2-3 for external debug; connect external J-Link to J13 | 2.91 V | 2.62 V |
| FPB-RA2E3 | Connect external J-Link to CN4 | 2.9 V | 2.51 V |
| FPB-RA4E2 | README does not require the external J-Link listed for most other boards | 2.99 V | 2.8 V |
| FPB-RA6E2 | README does not require the external J-Link listed for most other boards | 2.99 V | 2.8 V |
| FPB-RA8E1 | Connect external J-Link to J13 | 3.08 V | 2.85 V |

### LVD0 configuration

The supplied README states that LVD0 is configured through OFS1, or OFS1_SEC for FPB-RA4E2 and FPB-RA6E2, using the BSP properties in the FSP Configurator.

The application source also states that the LVD0 circuit and detection voltage can be configured under the OFS1 register settings in `configuration.xml` -> BSP -> Properties.

### Output behavior on these five boards

For FPB-RA0L1, FPB-RA2E3, FPB-RA4E2, FPB-RA6E2, and FPB-RA8E1, the README explicitly states:

- Terminal log output is not supported by this example.
- LED1 turns on when an LVD0 reset is detected.
- LED2 toggles when LVD1 generates an interrupt.

### FPB-RA2E3 reset sequence

For FPB-RA2E3, the README additionally instructs the user to program using the external debugger, disconnect debug mode, press the on-board reset button, decrease the input voltage below 1.7 V (the Power-on Reset voltage identified by the README), and then raise the input voltage to 3.3 V before exercising the voltage-changing behavior.

## 6. What must I change for my board, pins, clocks, timing, memory, task priorities, or middleware setup?

The supplied files document the reference LVD configuration but do not provide numeric memory requirements, RTOS task priorities, or timing-performance requirements.

### Voltage thresholds and LVD configuration

Change the FSP/BSP LVD settings to match the required voltage-monitoring policy of the target design. LVD0 is configured through OFS1/OFS1_SEC in this example. LVD1 is represented by the generated `g_lvd_ctrl` and `g_lvd_cfg` objects used by `R_LVD_Open()`.

Changing thresholds requires corresponding review of the expected input-voltage transitions used to demonstrate LVD0 reset and LVD1 interrupt behavior.

### Power connections

The reference test actively varies the board supply using a variable DC supply connected to board VCC and VSS/GND. Any target setup must account for how the monitored supply is applied and varied. The README recommends an external capacitor on specified Pmod power pins if the supply creates unwanted spikes.

### LEDs

The reference application uses the first two BSP LED entries. LED1 is used for LVD0 reset indication and LED2 for LVD1 interrupt indication. A target without the same LED resources must adapt or remove that reference-application indication.

### Generated LVD instance and callback

The core source expects `g_lvd_ctrl` and `g_lvd_cfg`. LVD1 event handling also depends on `lvd1_callback()` being associated with the configured LVD instance.

### Clocks, timing, memory, and task priorities

The attached files do not specify a clock frequency, response-time benchmark, memory budget, or RTOS task priorities for this SBB. The source uses an LVD callback and a foreground loop rather than showing an RTOS task implementation.

### Terminal/console

Although the reference application contains UART/stdio calls, the README explicitly states that terminal log output is not supported for the five boards covered by this document. Customer-observable behavior for these boards should therefore be based on LED1 and LED2 as documented in the README.

## 7. What dependencies does the SBB have on FSP, middleware, drivers, services, or hardware resources?

### FSP dependencies

The core includes `hal_data.h` and uses:

- `R_LVD_Open()`
- `R_LVD_StatusClear()`
- `R_LVD_Close()`
- `lvd_callback_args_t`
- `LVD_CURRENT_STATE_ABOVE_THRESHOLD`
- MCU system reset-status registers

The reference application additionally uses:

- `R_FSP_VersionGet()`
- `R_IOPORT_PinRead()`
- `R_IOPORT_PinWrite()`
- `g_ioport_ctrl`
- `g_bsp_leds`

### FSP-generated/BSP configuration

The implementation expects:

- `g_lvd_ctrl`
- `g_lvd_cfg`
- An LVD1 configuration that uses `lvd1_callback()`.
- The LVD0 OFS1/OFS1_SEC configuration described in the README.
- BSP LED definitions if the supplied visual indication is retained.

### Demo service dependency

`lvd_example_app.c` includes `console/uart_console.h` and calls UART-console/stdio functions. However, the supplied README states that terminal log output is not supported for all five boards in this document.

### Hardware resources

The demonstrated setup uses a supported RA board, variable DC power supply, LED1/LED2, jumper wiring, and the board-specific external-debug arrangement documented above. The README also recommends an external capacitor if the power supply produces unwanted spikes.

## 8. What behavior and performance were demonstrated or measured?

### Demonstrated behavior

The supplied material documents these threshold behaviors:

| Board | LVD0 reset demonstrated below | LVD1 interrupt demonstrated when rising above |
|---|---:|---:|
| FPB-RA0L1 | 2.62 V | 2.91 V |
| FPB-RA2E3 | 2.51 V | 2.9 V |
| FPB-RA4E2 | 2.8 V | 2.99 V |
| FPB-RA6E2 | 2.8 V | 2.99 V |
| FPB-RA8E1 | 2.85 V | 3.08 V |

The example also demonstrates:

- Detection of a previous LVD0 reset after power is restored above the LVD0 threshold.
- LED1 indication of detected LVD0 reset.
- A maskable LVD1 interrupt on a below-to-above threshold transition.
- LED2 toggling after the foreground application observes the LVD1 event.
- LVD1 status clearing after the event is handled.

### Performance data

The supplied README and source do not report interrupt latency, voltage-detection latency, CPU utilization, flash/RAM footprint, power consumption, or other benchmark measurements. The threshold voltages above are configured/demonstrated operating points documented by the README, not performance measurements.

## 9. What files are safe to modify and what files are generated, preset, or demo-only?

### Reusable SBB files

- `lvd_ep.c`
- `lvd_ep.h`

These files contain LVD0 reset checking, the LVD1 FSP wrapper, the status flag, and the LVD1 callback.

### Demo/reference application files

- `lvd_example_app.c`
- `lvd_example_app.h`

These files contain the demonstration sequencing, LED handling, application messages/guidance, and application-level error behavior.

### Generated/preset configuration

The core uses `hal_data.h` and generated LVD objects. The README specifically identifies LVD0 configuration in the BSP OFS1/OFS1_SEC settings in the FSP Configurator. The attached file set does not enumerate every generated project file, so this document does not label individual generated files beyond the configuration dependencies explicitly visible in the supplied material.

## 10. What happens if I modify the source and later want to adopt a newer SBB release?

The supplied README and source do not define an automated SBB upgrade or migration mechanism.

As integration guidance, keep target-specific customization separate from the reusable LVD interface where possible, and track changes to `lvd_ep.c` and `lvd_ep.h` in source control. Before adopting a newer SBB release, compare locally modified core files with the new release and review changes to the public APIs, reset-status handling, LVD1 callback, generated-instance references, and OFS-related assumptions before replacing local code.

Board voltage thresholds and OFS configuration also need to remain aligned with the intended target configuration when integrating a later release.

## Reference Application Flow

```text
hal_entry()
  -> lvd_example_app_run()
       -> check that on-board LEDs exist
       -> lvd0_reset()
       |    -> if LVD0 reset detected: LED1 ON
       |
       -> lvd_init()
       |
       -> foreground loop
            -> lvd1_callback() sets g_lvd1_status
            -> application observes g_lvd1_status
            -> toggle LED2
            -> lvd_status_clear()
            -> clear g_lvd1_status
```

## Integration Checklist

- [ ] Use the LVD0/LVD1 voltage thresholds specified for the selected reference board or configure intentional target-specific values in FSP.
- [ ] Configure LVD0 through the applicable OFS1/OFS1_SEC BSP settings.
- [ ] Verify `g_lvd_ctrl` and `g_lvd_cfg` correspond to the intended LVD1 configuration.
- [ ] Verify `lvd1_callback()` is associated with the LVD1 instance.
- [ ] Apply the documented external-debug connection for FPB-RA0L1, FPB-RA2E3, or FPB-RA8E1 when following the README setup.
- [ ] Account for the special FPB-RA2E3 voltage/reset sequence documented by the README.
- [ ] Use a variable DC supply connected to VCC and VSS/GND as described by the reference setup.
- [ ] Add external capacitance if the supply introduces unwanted voltage spikes, as recommended by the README.
- [ ] Provide LED1 and LED2 indication or adapt the reference layer for different target indicators.
- [ ] Do not depend on terminal logging for the five boards covered by this document; the README specifies LED-based event indication for them.
