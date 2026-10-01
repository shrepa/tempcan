# STM32F446 CAN Loopback Telemetry

Bare-metal STM32 firmware that reads the MCU's internal temperature sensor and
sends it as a CAN frame once per second, using the bxCAN peripheral in
**loopback mode**, so it runs on a bare Nucleo board with no external CAN
transceiver or second node.

## What it does

1. Samples the internal temperature sensor (ADC1) in interrupt mode, restarting
   the conversion from the completion callback.
2. Converts the raw reading to °C using the STM32F446 datasheet constants
   (V25 = 0.76 V, avg. slope = 2.5 mV/°C).
3. Every 1 s, puts the temperature in a CAN frame and queues it for transmission.
4. Receives the frame back through CAN RX FIFO0 (via interrupt) because the
   peripheral is in loopback mode.

## Hardware

| Item | Detail |
|---|---|
| Board | Nucleo-F446RE (STM32F446RETx, LQFP64) |
| Clock | HSI → PLL, 84 MHz SYSCLK, APB1 = 42 MHz |
| Debug | On-board ST-LINK (SWD) |

No other hardware is required: loopback mode makes the firmware self-contained, which makes it easy to bring up and test.

### Pins

| Signal | Pin |
|---|---|
| CAN1_RX | PA11 (pull-up) |
| CAN1_TX | PA12 |
| USART2_TX / RX | PA2 / PA3 (ST-LINK virtual COM, 115200 8N1) |
| LD2 (green LED) | PA5 |
| B1 (user button) | PC13 |

## CAN configuration

| Parameter | Value |
|---|---|
| Mode | CAN_MODE_LOOPBACK |
| Bit rate | 500 kbit/s (prescaler 4, BS1 = 16 TQ, BS2 = 4 TQ, 21 TQ per bit) |
| Frame | Standard ID 0x123, data frame, DLC 8 |
| Payload | byte 0 = temperature in °C (integer), byte 7 = 0xFF, others 0 |
| Rx filter | Bank 0, 32-bit ID/mask, accept all → FIFO0 |
| Rx handling | Interrupt (HAL_CAN_RxFifo0MsgPendingCallback) |

In loopback mode the bxCAN feeds its own transmitted frames back to its receiver and ignores the
RX pin, so the peripheral works with nothing connected to the CAN pins.


## Build and run

1. Install [STM32CubeIDE](https://www.st.com/en/development-tools/stm32cubeide.html).
2. **File → Import → Existing Projects into Workspace** and select this folder.
3. Build the Debug configuration.
4. Connect the Nucleo over USB and click **Debug**.

### Checking that it works

There's no UART output; verify in the debugger. Add these to **Live Expressions**:

- temperature_in_c: should read roughly room/die temperature (die is usually a
  few °C above ambient).
- TxData[0] and RxData[0]: should match once per second.
- RxHeader.StdId: should be 0x123.

## Known limitations

- Temperature is truncated to a single uint8_t, so sub-degree precision is
  lost and negative values wrap.
- The TX loop busy-waits for a free mailbox.
- Received frames are stored but not yet acted on.
- The analog watchdog is enabled with thresholds of 0, which raises needless
  watchdog interrupts. Safe to disable.
- Internal temperature sensor accuracy is only about ±1.5 °C (uncalibrated) and
  it measures the die, not the environment.
