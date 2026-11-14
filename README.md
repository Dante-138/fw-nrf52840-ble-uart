# fw-nrf52840-ble-uart

BLE to UART bridge for the nRF52840 DK using the Nordic UART Service (NUS). Zephyr RTOS, nRF Connect SDK.

## What it does

Two-way text link between a serial terminal on a PC and a phone running nRF Connect:

```
PC terminal <--UART--> nRF52840 DK <--BLE--> Phone (nRF Connect)
```

- Text typed in the terminal is sent to the phone as a NUS notification when Enter is pressed.
- Text written by the phone to the NUS RX characteristic appears in the terminal.
- The device advertises as `nRF52840_BLE_UART`.

LEDs: LED1 blinks while the firmware runs, LED2 is on while a BLE central is connected.

Design:

- UART to BLE: UART RX uses the async API with DMA. Characters accumulate until `\r` or `\n`, then go through a `K_FIFO` to `ble_write_thread`, which calls `bt_nus_send()`. The thread exists because `bt_nus_send()` can block.
- BLE to UART: `bt_receive_cb()` runs in thread context and calls `uart_tx()` directly.

| Thread | Role | Stack | Priority |
|---|---|---|---|
| main | init, blinks LED1 | default | 0 |
| ble_write_thread | reads the FIFO, sends over BLE | 1024 B | 7 |

Limitations:

- No local echo in the terminal.
- Messages are sent only on Enter.
- About 20 characters per message at the default MTU; longer messages are fragmented.

## Hardware

| Item | Detail |
|---|---|
| Board | nRF52840 DK (PCA10056) |
| MCU | nRF52840, Cortex-M4F, 64 MHz, 1 MB flash, 256 KB RAM |
| UART | J-Link VCOM over the on-board USB, 115200 8N1, no flow control |
| Phone | nRF Connect app (Android or iOS) |

## Build

Requires nRF Connect SDK v3.x (tested with v3.0.2). Use a project path without spaces.

```bash
west build -b nrf52840dk/nrf52840 --no-sysbuild --pristine
```

`--no-sysbuild` is needed on SDK v3.x for a standalone project.

Relevant `prj.conf` options: `CONFIG_BT_PERIPHERAL`, `CONFIG_BT_NUS`, `CONFIG_UART_ASYNC_API`, `CONFIG_DK_LIBRARY`. Logging goes to RTT (`CONFIG_UART_CONSOLE=n`) so the UART carries only NUS data.

## Flash

```bash
west flash --runner jlink
```

## Test

1. Open a serial terminal on the DK's COM port: 115200 baud, 8 data bits, 1 stop bit, no parity, no flow control.
2. In nRF Connect on the phone, scan and connect to `nRF52840_BLE_UART`. LED2 turns on.
3. Open the Nordic UART Service and enable notifications on the TX characteristic (`6E400003-B5A3-F393-E0A9-E50E24DCCA9E`).
4. Type text plus Enter in the terminal: it shows up on the TX characteristic.
5. Write text to the RX characteristic (`6E400002-B5A3-F393-E0A9-E50E24DCCA9E`): it shows up in the terminal.

If the connection fails, clear the old bond on the phone. `nrfjprog --eraseall` followed by a new flash also clears the board's bonding data.

## License

MIT, see [LICENSE](LICENSE).
