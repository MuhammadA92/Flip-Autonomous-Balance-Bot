# Flip Autonomous Balance Bot

A two-wheel balancing robot developed for the Imperial College London second-year Electronics Design Project in summer 2026. The group combined balancing, remote control, sensor-driven navigation, video, and power telemetry in one demonstrator.

This repository documents my individual engineering contribution to **Flip**, a five-person group project. My work focused on balance control, motion-control experiments, IMU calibration, and the analogue hardware and software used to monitor the robot's batteries and power consumption.

> **Muhammad Abubakar** - control systems, remote movement, and power monitoring  
> The complete team implementation and development history are available in [Josiah Tse's Flip repository](https://github.com/FlyawayNutria/Flip).

## Project brief

The robot was required to:

- balance with its centre of gravity above the wheel axle;
- move manually in two dimensions through a remote interface;
- demonstrate autonomous behaviour using onboard sensors or imaging;
- report battery status and subsystem power consumption; and
- integrate an ESP32, Raspberry Pi, MPU6050 IMU, stepper motors, sensor electronics, and a web interface.

The original university specification and starter platform are documented in the supervisor's [Balance Robot technical guide](https://github.com/edstott/EE2Project/tree/main/balance-robot).

## Final integrated system

- **Operator interface:** A Raspberry Pi hosted the web dashboard for manual control, mode selection, live video, and telemetry. It exchanged commands and telemetry with the ESP32 over USB serial/UART at 115200 baud.
- **Balancing and movement:** The ESP32 ran the approximately 100 Hz inner balance loop using the MPU6050 tilt estimate. A continuously active **PI velocity outer loop** adjusted the target tilt for forward/backward movement. The inner tilt controller produced a motor acceleration command, which was integrated into the wheel-speed command. A separate PD yaw controller added a differential steering offset for turns.
- **Velocity feedback:** The control software used the stepper library's speed estimate. There were **no physical wheel encoders**, so wheel slip and missed steps were not directly measured.
- **Autonomous functions:** The team adapted IR line-following logic for continuous movement on the balancing robot and implemented ToF-based obstacle/maze-navigation logic. The report describes integration and tests, but gives its clearest quantitative final result for upright balance; it does not establish a measured end-to-end success rate for every autonomous mode.
- **Power telemetry:** Battery voltage and the high-side current-sense signals for the 5 V and motor/battery rails were conditioned for the MCP3208 ADC. The ESP32 read three ADC channels over SPI; the Pi received analogue readings in telemetry, and the web client applied the calibration equations to display voltage, current, and power.

The sections below explain my contribution and the experiments that led to this design. In particular, the position controller and the idle/driving/braking state machine were **development experiments**, not the final movement controller.

## My contribution

### Balance and motion control

- Co-developed the robot's static and dynamic balancing system and remote movement control.
- Developed and tuned the inner balance controller, progressing from proportional velocity control to gyro-damped PD control with acceleration-based motor commands.
- Identified stepper acceleration as an early limiting factor and increased the configured limit from **200 rad/s² to 1000 rad/s²**.
- Used the MPU6050 gyroscope rate directly for derivative damping, avoiding the delay and noise amplification of numerical differentiation.
- Tested a velocity-damping term that reduced low-frequency oscillation during static-balance tuning without the motor heating and current increase caused by excessive derivative gain. It was removed during later movement-controller tuning.
- Investigated position control, wheel odometry, moving-target control, tilt-pulse movement, and state-based gain scheduling.
- Developed and tested idle, driving, and braking states to study drift, stopping behaviour, and controller transitions; the team ultimately used a continuously active PI velocity outer loop instead.

### IMU and balance-point calibration

- Compensated for stationary gyroscope bias on each axis before sensor fusion and control calculations.
- Aligned the MPU6050 pitch measurement with the robot's wheel axis using a rotation-matrix approach.
- Refined the balance setpoint to account for sensor alignment and the robot's offset centre of mass.
- Improved the final calibration by averaging the pitch estimate for **8 seconds** while the robot attempted to self-balance, reducing the effect of its approximately **2 Hz** oscillation.

### Battery and power monitoring

- Designed monitoring for battery voltage, motor-rail current, and 5 V electronics-rail current.
- Used the existing high-side shunts on the power PCB: **0.1 Ω for the motors** and **0.01 Ω for the 5 V rail**.
- Designed potential-divider, differential-amplifier, and non-inverting gain stages around an **MCP6022 rail-to-rail dual op-amp**.
- Simulated both sensing circuits in LTspice and selected gains that used most of the **0-4.096 V ADC range** without exceeding it.
- Built and tested the circuits on breadboard, selected closely matched resistors, adjusted component values from measured results, and soldered the final perfboard implementation.
- Produced software calibration equations that converted ADC voltage into shunt voltage, current, and power.

## Control architecture

<img width="1940" height="525" alt="Figure 2.4: Flip control-loop block diagram" src="https://github.com/user-attachments/assets/c392782c-0629-4441-a408-6fd4e55275c4" />

*Control-loop block diagram from the group report, Figure 2.4.* It depicts the final cascaded velocity and tilt control concept, with a separate yaw controller mixed into the motor command. The MPU6050 supplies tilt and yaw-rate feedback.

The report labels the velocity feedback “wheel encoders.” The implementation used the stepper library's speed estimate (a virtual encoder), not physical wheel encoders.

During static-balance development, a tested controller included velocity damping:

```text
target_acceleration = Kp * angle_error
                    + Kd * gyro_rate
                    - Kv * integrated_velocity
```

The acceleration command was integrated each control iteration to obtain the target wheel velocity. The `Kv` term helped with static oscillation, but this exact expression should not be read as the final integrated controller: the later PI velocity outer loop was kept active and `Kv` was removed during movement tuning.

More detail is available in [Control system development](control-system.md).

## Power-monitoring architecture

| Measurement | Interface | Purpose |
|---|---|---|
| Battery voltage | Potential divider | Estimate remaining battery level safely |
| Motor current | 0.1 Ω shunt + differential and gain stages | Measure motor demand during balancing and movement |
| 5 V rail current | 0.01 Ω shunt + differential and gain stages | Measure the ESP32, Raspberry Pi, and peripheral load |

The practical calibration sweeps remained highly linear:

| Circuit | Measured calibration | R² |
|---|---:|---:|
| Motor rail | `Vout = 16.936 * ΔVin + 0.2617` | 0.9994 |
| 5 V rail | `Vout = 97.936 * ΔVin + 0.0179` | 0.9981 |

More detail is available in [Power-monitoring hardware](power-monitoring.md).

## Selected engineering results

- Achieved repeatable static balancing using complementary-filtered IMU feedback.
- Reduced balance oscillation while avoiding the increased motor current caused by excessive derivative gain.
- Enabled remote forward/backward motion while maintaining balance.
- Developed power-monitoring circuits whose outputs remained within the ADC range and exhibited near-linear measured responses.
- Integrated balancing, remote control, telemetry, and line-following behaviour in the team demonstrator. The report documents additional obstacle and maze-navigation work without a complete measured end-to-end evaluation of those modes.

## What I learned

- Controller tuning cannot compensate for every mechanical defect. Loose wheels and compliance can look like a control-software problem.
- A robot can be upright while still translating; reliable stopping requires velocity feedback as well as angle control.
- Cascaded control loops require clear bandwidth separation and dependable feedback signals.
- High-side current measurement demands careful common-mode handling, resistor matching, protection, simulation, and calibration.
- Quantitative tests made it possible to distinguish control improvements from changes that merely looked smoother.

## Project ownership

Flip was a collaborative university project completed by 5 undergraduate students. This repository is a personal portfolio record of Muhammad's work; it does not claim sole ownership of the full robot or the team codebase.

For access to the full source tree and details of the wider team implementation, please contact the owner of the [original Flip repository](https://github.com/FlyawayNutria/Flip).
