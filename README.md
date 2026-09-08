# Calligraphy Writing Robot

A desktop robot that writes Chinese calligraphy with a real ink brush. A 3R serial
arm carries the brush; a lead-screw linear stage carries the arm, adding the travel
needed to cross a sheet of A4 paper. Both are driven by an STM32F405 board designed
and fabricated for this project.

Built for the *Mini Robot Design and Production Practice* course at the School of
Mechanical and Electrical Engineering, University of Electronic Science and
Technology of China (UESTC), 2023.

Demo video: [`Robotics Project Showreel`](https://github.com/pengyu-jing/Robotics-Project-Showreel)
→ `A calligraphy robot-Domostration Video.mp4`

## At a glance

| | |
|---|---|
| **Kinematics** | 3R serial arm mounted on a 1-DOF linear stage (4 actuated axes + gripper) |
| **MCU** | STM32F405RGT6 @ 160 MHz (HSI × PLL), custom `F405_V3.0_Red` board |
| **RTOS** | FreeRTOS through CMSIS-RTOS v2 |
| **Arm joints** | LX-16A serial bus servos, commanded through a Lobot LSC 24-channel servo controller on USART2 @ 9600 baud |
| **Gripper** | SG90 servo driving a spur-gear pinch pair |
| **Linear stage** | NEMA-17 (42 mm) stepper, 220 mm lead screw, LMK8UU / U linear bearings, SHF8 shaft supports |
| **Stepper drive** | TIM3_CH2 PWM step train on PA7, direction on PA4, 3200 pulses per revolution |
| **Operator input** | USB mouse through a CH9350 USB-HID-to-UART bridge on USART1 — cursor deltas jog the axes, clicks mark strokes |
| **Mechanical** | SolidWorks; most parts 3D printed, the rest stock or machined in-house |
| **Electrical** | Altium Designer; on-board buck stage for the stepper and logic rails |

## Repository layout

```
Code/                          STM32CubeMX + Keil MDK project
  maomaojqr.ioc                CubeMX configuration (F405RGT6, FreeRTOS, TIM3, USART1/2)
  Core/                        CubeMX-generated application core
    Src/main.c                 peripheral bring-up, TIM3 overflow ISR, USART1 RX ISR
    Src/freertos.c             CMSIS-RTOS v2 setup (defaultTask)
  Drivers/                     STM32F4xx HAL + CMSIS (vendor, unmodified)
  Middlewares/Third_Party/     FreeRTOS kernel (vendor, unmodified)
  MDK-ARM/                     Keil uVision project and hand-written modules
    maomaojqr.uvprojx          build target
    motor/bujin.{c,h}          stepper step/direction driver (方向, 速度, 圈数)
    motor/LobotServoController.{c,h}   LSC serial-servo protocol (0x55 framing)
    motor/my_lib.h             shared externs
    RxBuf/RxBuf.{c,h}          byte ring buffer for the USART1 stream
    bool/bool.h                boolean shim for C89-style sources
    maomaojqr/                 build output (.axf, .hex)

Hardware/
  F405_V3.0_Red/               Altium project: schematic, PCB, footprint library
  F405_V3.0_Red.pdf            schematic export

Model/                         SolidWorks CAD
  毛笔字机器人.SLDASM           top-level assembly (calligraphy robot)
  机械臂/                       3R arm: links, LX-16A mounts, bearings, gripper, brush, A4 sheet
  滑台/                         linear stage: lead screw, nut blocks, bearings, motor plate
  电路部分/                     electronics tray: controller, servo board, battery box
  草图/                         layout sketches and joint studies

Documents/                     course handouts (Chinese-language content)
Mini-Robot-Design-Report.docx  project report (Chinese-language content)
```

## Building and flashing

1. Open `Code/MDK-ARM/maomaojqr.uvprojx` in Keil MDK-ARM (Arm Compiler 5/6).
   The hand-written modules under `MDK-ARM/motor`, `MDK-ARM/RxBuf` and
   `MDK-ARM/bool` are already on the include path.
2. Build, then flash over SWD (PA13/PA14) with an ST-LINK.
   A prebuilt image is available at `Code/MDK-ARM/maomaojqr/maomaojqr.hex`.
3. To change the pin map or clock tree, edit `Code/maomaojqr.ioc` in STM32CubeMX
   and regenerate. Regeneration preserves the `USER CODE` regions in
   `Core/Src/main.c` and `Core/Src/freertos.c`.

Sources under `MDK-ARM/motor` are GBK-encoded and contain Chinese comments. Set
your editor to GBK (or convert with `iconv -f GBK -t UTF-8`) to read them.

## Design notes

The arm is a 3R planar chain: three LX-16A bus servos in series, each riding on a
deep-groove ball bearing so the servo horn carries torque but not the radial load.
The brush is fixed to the last link; ink loading and paper changes are manual.

The linear stage exists because a 3R arm with links short enough to stay stiff
cannot span an A4 sheet. Sliding the whole arm along the lead screw turns a small,
rigid workspace into a wide one, at the cost of one more axis to coordinate.

See [ARCHITECTURE.md](ARCHITECTURE.md) for the control-path breakdown.

## Related repositories

- [`Robocon2023-Elephant-Robot`](https://github.com/pengyu-jing/Robocon2023-Elephant-Robot) — ABU Robocon 2023 competition robot
- [`Seesaw-Balancing-Robot`](https://github.com/pengyu-jing/Seesaw-Balancing-Robot) — seesaw balancing robot
- [`Chiyu-A60-Spray-Drone`](https://github.com/pengyu-jing/Chiyu-A60-Spray-Drone) — agricultural spraying UAV
- [`Robotics-Project-Showreel`](https://github.com/pengyu-jing/Robotics-Project-Showreel) — demonstration videos

## Credits

Zhang Chuming, Jing Pengyu, Zhu Xiaorui. Supervisor: Mao Xiangyu.
UESTC, School of Mechanical and Electrical Engineering, Robot Engineering.

`Drivers/` and `Middlewares/` are ST and FreeRTOS vendor code, redistributed under
their original licenses. `LobotServoController.{c,h}` derives from the Lobot LSC
reference example.
