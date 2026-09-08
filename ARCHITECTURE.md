# Architecture

## System overview

```
                    ┌───────────────────────────────┐
   USB keyboard ───▶│ CH9350  USB-HID → UART bridge │
   / mouse          └──────────────┬────────────────┘
                                   │ USART1, byte-at-a-time IT
                                   ▼
        ┌──────────────────────────────────────────────────┐
        │  STM32F405RGT6   ·  F405_V3.0_Red board          │
        │                                                  │
        │   USART1 RX ISR ──▶ RxBuf ring ──▶ command frame │
        │                                       │          │
        │                            ┌──────────┴────────┐ │
        │                            ▼                   ▼ │
        │                  TIM3_CH2 PWM (PA7)      USART2 tx│
        │                  DIR      GPIO (PA4)     @9600    │
        │                  TIM3 update ISR                  │
        └────────────────────┬──────────────────────┬───────┘
                             │                      │
                             ▼                      ▼
                  ┌────────────────────┐  ┌────────────────────────┐
                  │ Stepper driver     │  │ Lobot LSC 24-ch servo  │
                  │ NEMA-17 + screw    │  │ controller  (0x55 hdr) │
                  └─────────┬──────────┘  └───────────┬────────────┘
                            │                         │
                            ▼                         ▼
                     Linear stage             3 × LX-16A bus servos
                     (X travel)               + SG90 gripper
                                                      │
                                                      ▼
                                                  Ink brush
```

Four actuated degrees of freedom reach the paper: the linear stage sweeps the arm
base along one axis, and the 3R chain places and orients the brush tip within the
plane it can reach from there. The gripper is a separate, non-kinematic axis used
to hold and release the brush.

## Control path

### Command intake — `RxBuf` + USART1

`HAL_UART_RxCpltCallback` in `Core/Src/main.c` runs per received byte. Each byte is
pushed into `stRev`, a 57-byte `stRingBuf` ring (`MDK-ARM/RxBuf/RxBuf.c`). Once
`RxCnt` reaches 9 the handler drains 14 bytes into `RxBuffer` — the CH9350 report
length — and raises `Receive_Need_Analysis` for the consumer to parse.

`RxBuf.h` declares what a parsed report yields: `flag_click` for the mouse buttons
and `X_move` / `Y_move` / `scroll_move` deltas, plus `SUM_*` accumulators. A USB
mouse is therefore the teach pendant: cursor deltas jog the axes and clicks latch
brush-down and stroke boundaries.

The callback also carries recovery code: if the UART is left in a non-`READY`
state it aborts and re-arms the interrupt, counting the event in `Fluat_CNT`. This
guards against the RX overrun that the CH9350 bridge can cause when a HID device
bursts, which otherwise silently stops the receive chain.

The CH9350 is put into streaming mode with the fixed initialisation sequence
`CH9350_start_Sending[]` (`57 AB 12 …`) declared in `main.c`.

### Linear axis — stepper

`dianji_kongzhi(fangxiang, sudu, zhuoqi)` in `MDK-ARM/motor/bujin.c` is the whole
driver:

- `fangxiang` (direction) → `PA4` level, `up` / `down` in `bujin.h`
- `sudu` (speed) → `TIM3` auto-reload, so a smaller reload means a faster step train
- duty is set on `TIM3_CH2` (`PA7`); a compare of 0 stops the axis

Travel is counted, not measured — there is no encoder on this axis.
`HAL_TIM_PeriodElapsedCallback` increments `ITjishu` once per PWM period, and at
640000 counts it zeroes the compare register to stop the motor. At the documented
3200 pulses per revolution this is the open-loop distance limit; homing is manual.

CubeMX sets `TIM3` to prescaler `80-1` and period `1600`.

### Arm joints — LSC serial protocol

`MDK-ARM/motor/LobotServoController.c` speaks the Lobot LSC frame format on
USART2 at 9600 baud: header `0x55 0x55`, length, command, payload.

| Command | Code | Use |
|---|---|---|
| `CMD_SERVO_MOVE` | `0x03` | move one or more servos to a position over a given time |
| `CMD_ACTION_GROUP_RUN` | `0x06` | replay a stroke recorded on the controller |
| `CMD_ACTION_GROUP_STOP` | `0x07` | abort the running group |
| `CMD_ACTION_GROUP_SPEED` | `0x0B` | scale playback speed |
| `CMD_GET_BATTERY_VOLTAGE` | `0x0F` | read pack voltage |

`moveServos(Num, Time, ...)` is the variadic entry point; `moveServosByArray()`
takes a `LobotServo[]` of `{ID, Position}` pairs. Because the LSC board owns the
interpolation, the STM32 sends targets and durations rather than a servo update
loop — timing precision for a stroke comes from the `Time` argument.

Action groups matter for calligraphy: a stroke can be taught once on the LSC and
replayed by index, so the STM32 sequences strokes instead of streaming setpoints.

### Scheduling

FreeRTOS is initialised through CMSIS-RTOS v2 (`Core/Src/freertos.c`) with a
single `defaultTask` at `osPriorityNormal`, 512-byte stack. The real work is
interrupt-driven — UART RX and TIM3 update — with the task loop left as the place
for sequencing. Motion is therefore paced by hardware timers, not by the
scheduler.

## Module map

| Path | Responsibility |
|---|---|
| `Core/Src/main.c` | clock tree, peripheral init, TIM3 and USART1 ISRs, CH9350 startup |
| `Core/Src/freertos.c` | CMSIS-RTOS v2 objects and the default task |
| `Core/Src/tim.c`, `usart.c`, `gpio.c` | CubeMX peripheral configuration |
| `MDK-ARM/motor/bujin.{c,h}` | stepper direction / speed / revolution counting |
| `MDK-ARM/motor/LobotServoController.{c,h}` | LSC servo-controller protocol |
| `MDK-ARM/motor/my_lib.h` | shared externs across the hand-written modules |
| `MDK-ARM/RxBuf/RxBuf.{c,h}` | ring buffer, `WriteOneByte` / `ReadOneByte` |
| `MDK-ARM/bool/bool.h` | boolean type for the C89-style sources |

## Electrical layout

The `F405_V3.0_Red` board carries the F405, the SWD header, the stepper
direction/step outputs and the two UART headers. A separate buck stage drops the
pack voltage for the stepper driver and the logic rails. The LSC servo controller
and the servo pack are powered independently of the logic rail — bus servos draw
enough current under stall that sharing the rail browns out the MCU.

Wire routing was constrained by the mechanics rather than the other way round: the
electronics tray (`Model/电路部分/`) fixes the controller, the 24-channel servo
board and the battery box outside the arm's swept volume, so cabling to the moving
links stays short.

## Known limitations

- The linear axis is open loop. Lost steps accumulate over a long stroke sequence
  and there is no home switch to recover from.
- The USART1 frame logic is off by five. `RxCnt` is compared against 9 but 14 bytes
  are drained (`main.c:207-211`), and `RxCnt` is then pinned at 9 rather than
  reset — so every byte after the ninth triggers another 14-byte read from a ring
  that has not received that much. It works in practice only because the CH9350
  streams continuously and the ring is 57 bytes deep.
- `Core/Src/main.c` declares `uint8_t Rxchar` twice in the same translation unit
  (`main.c:39` and `main.c:42`); it compiles as a tentative definition but is
  redundant.
- `RxBuf.h` still carries `#define HUART huart3` copied from the source project;
  this board routes the bridge to `huart1`.
- Stroke geometry lives in LSC action groups, not in this repository, so the
  written characters are not reproducible from source alone.
