# Obstacle Avoiding Vehicle — TM4C123GH6PM

Bare-metal embedded C project implementing an autonomous obstacle-avoidance vehicle on the Tiva TM4C123GH6PM microcontroller (ARM Cortex-M4F, 50 MHz).

An HC-SR04 ultrasonic sensor mounted on a servo scans five directions. A finite state machine selects a clear path and commands two DC motors via an L298N H-bridge driver. A Bluetooth module enables wireless on/off control and sensor queries. Status is shown on a 16×2 LCD.

---

## Hardware

| Component | Part | Interface |
|---|---|---|
| Microcontroller | TM4C123GH6PM (Tiva C LaunchPad) | — |
| Motor driver | L298N dual H-bridge | GPIO + PWM |
| Ultrasonic sensor | HC-SR04 | GPIO + Timer input capture |
| Servo | SG90 (or equivalent) | Bit-bang PWM |
| LCD | HD44780-compatible 16×2 | 4-bit parallel |
| Bluetooth | HC-05 | UART5 |
| Temperature sensor | TMP36 | ADC |

---

## Pin Assignments

| Pin | Function |
|---|---|
| PA2 / PA3 | Motor A direction (IN1/IN2) |
| PA4 | Servo PWM (bit-bang, 50 Hz) |
| PA5 / PA6 | Motor B direction (IN3/IN4) |
| PB2 | Ultrasonic echo — T3CCP0 (Timer3A input capture) |
| PB3 | Ultrasonic trigger (GPIO output) |
| PB4 | Motor A PWM — M0PWM2 |
| PB5 | Motor B PWM — M0PWM3 |
| PC4–PC7 | LCD D4–D7 |
| PD0 / PD1 / PD2 | LCD RS / RW / E |
| PD3 | TMP36 analog input (AIN4, ADC0) |
| PE4 / PE5 | UART5 RX / TX (Bluetooth) |
| PF1 / PF2 / PF3 | Red / Blue / Green LEDs |

> **Note:** PC0–PC3 are JTAG/SWD pins and must not be configured.

---

## Firmware Architecture

```
src/
  main.c          Initialization sequence and main loop
  fsm.c           Obstacle avoidance FSM
  port_init.c     GPIO configuration for all ports
  timers.c        Delay timers (Timer1A/2A) and 1s tick (Timer4A)
  ultrasonic.c    HC-SR04 trigger/echo driver (Timer3A input capture)
  servo.c         Bit-bang servo PWM driver
  motors.c        L298N direction + M0PWM duty cycle control
  lcd.c           HD44780 4-bit mode driver
  bluetooth.c     UART5 driver + command ISR
  temperature.c   TMP36 ADC driver

include/
  *.h             Public API headers for each module
```

### Obstacle Avoidance FSM

Scan order (per project spec):

```
SCAN_FRONT (90°) → SCAN_RIGHT45 (45°) → SCAN_LEFT45 (135°)
                 → SCAN_RIGHT90 (0°)  → SCAN_LEFT90 (180°)
```

If any direction has clearance > 10 cm, the vehicle moves in that direction and restarts the scan from the front. If all five directions are blocked, the vehicle performs a 180° spin in place and rescans.

### Interrupts

| Interrupt | Priority | Purpose |
|---|---|---|
| UART5 (IRQ 61) | 1 — HIGH | Bluetooth command handler |
| Timer4A (IRQ 70) | 2 — LOW | 1-second heartbeat tick |

### Timers

| Timer | Mode | Use |
|---|---|---|
| Timer1A | 32-bit periodic | 1 ms polling delay |
| Timer2A | 32-bit periodic | 1 µs polling delay |
| Timer3A | 16-bit input capture | Ultrasonic echo measurement |
| Timer4A | 32-bit periodic interrupt | 1-second tick / green LED heartbeat |

---

## Bluetooth Commands

Connect a serial terminal (9600 baud, 8N1) to the HC-05 module:

| Command | Action |
|---|---|
| `O` | Toggle vehicle ON / OFF |
| `D` | Query front distance (cm) |
| `T` | Query temperature (°C) |

---

## Build & Flash

1. Open the project in **Keil uVision 5**
2. Target device: `TM4C123GH6PM`
3. Add all `.c` files from `src/` and `include/` path to the project
4. Include the TivaWare `tm4c123gh6pm.h` header (from TI TivaWare or the Keil device pack)
5. Build → Flash via the on-board ICDI debugger

---

## Notes

- All register access is bare-metal (no TivaWare DriverLib)
- System clock: 50 MHz via PLL
- Servo pulse range: 500 µs (0°) to 2350 µs (180°), tuned for SG90
- Ultrasonic divisor (2500) was determined empirically on hardware
- `delay_ms()` / `delay_us()` are blocking — not suitable for RTOS use
