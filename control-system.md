# Control System Development

This note records the balance- and movement-control work I contributed to the Flip robot. It focuses on the engineering progression, including approaches that were tested and later replaced.

## Tilt estimation

The MPU6050 accelerometer gives a long-term reference to gravity but is disturbed by vibration and longitudinal acceleration. Its gyroscope gives a responsive angular-rate measurement but accumulates bias when integrated. The controller combined them with a complementary filter:

```text
tilt[n] = (1 - C) * accelerometer_tilt
        + C * (tilt[n - 1] + gyro_rate * dt)
```

Values of `C` from 0.85 to 0.99 were tested. A value of **0.95** gave a useful compromise between responsiveness and long-term drift. The loop interval was later measured dynamically rather than assumed constant so that network-related timing variations had less effect on the estimate.

## Sensor and setpoint calibration

Stationary gyro measurements were averaged to obtain per-axis bias offsets. Subtracting these offsets reduced drift and improved derivative damping, because the controller used gyroscope rate directly.

The MPU6050 was not perfectly aligned with the robot body. A rotation-matrix method was tested to transform sensor readings into the robot frame, with the most useful correction occurring around the pitch axis. The balance setpoint also differed from the geometric upright angle because the centre of mass was offset.

Calibration evolved through three approaches:

1. placing the robot on a stand and assuming the upright angle was 90 degrees;
2. holding the robot manually at its apparent balance point; and
3. averaging the pitch measurement over 8 seconds while the robot attempted to self-balance.

The last approach represented the operating condition most accurately. Averaging reduced the effect of the remaining oscillation, which was approximately 2 Hz.

## Inner balance controller

The first controller mapped angle error directly to wheel velocity with proportional gain. Increasing the gain alone did not allow the robot to recover because the configured stepper acceleration limit prevented the wheels from moving under the centre of mass quickly enough.

Increasing the limit from **200 rad/s² to 1000 rad/s²** allowed a proportional gain of approximately **24** to drive the system past the equilibrium point, confirming that the controller had sufficient authority. The response was still underdamped.

A derivative term was added to reduce overshoot. It was first calculated from successive angle errors, then replaced by the MPU6050 gyroscope rate. The direct rate measurement responded earlier and avoided the noise amplification associated with numerical differentiation.

The controller output was changed from direct velocity to target acceleration. Each loop integrated this acceleration to update the wheel-velocity command:

```text
integrated_velocity += target_acceleration * dt
```

This produced smoother commands and better control of the wheel-speed change.

## Velocity damping

After the robot could balance, a low-frequency oscillation remained. Mechanical checks and tightening reduced some vibration. Increasing derivative gain further reduced the slow rocking, but introduced high-frequency motor vibration, heating, and an increase in measured current from about 1 A to approximately 1.3-1.5 A.

I therefore added damping based on the integrated velocity command:

```text
target_acceleration = Kp * angle_error
                    + Kd * gyro_rate
                    - Kv * integrated_velocity
```

This behaved like artificial friction. It opposed excessive accumulated wheel velocity without making the controller overly sensitive to small, high-frequency angle changes.

## Dynamic movement experiments

### Position and moving-target control

Position was estimated from wheel rotation and radius. A direct position-to-tilt controller could start movement smoothly but often built excessive speed and overshot its target. Reaching the target position did not guarantee zero velocity: the robot could become upright and continue rolling.

A moving-target strategy was tested for manual control. While a movement command was active, the target position was continually placed ahead of the robot. Releasing the command moved the target to its current position. This felt more natural, but the absence of true wheel encoders limited accurate stopping.

### State-based gain scheduling

Static balancing and movement initially appeared to require different controller gains. A state-machine experiment separated operation into:

- **idle**, with stronger damping and the velocity loop disabled;
- **driving**, with the velocity outer loop enabled and reduced inner-loop damping;
- **braking**, with a zero velocity target and reverse tilt used to remove momentum; and
- **turning**, handled as a separate differential-steering mode.

Fixed braking durations were unreliable because the robot did not always reach maximum velocity before a command was released. A velocity-threshold exit reduced drift, but aggressive braking and abrupt gain changes caused vibration and falls.

Later testing showed that loose wheels had contributed significantly to the apparent controller instability. After the mechanical fault was corrected, the team returned to a simpler continuously active velocity outer loop, which avoided disruptive enable/disable transitions.

## Limitations

- Stepper position and speed estimates were not equivalent to independent encoder feedback; missed steps and wheel slip remained unobservable.
- Position control needed explicit velocity damping to stop reliably.
- Gain scheduling introduced discontinuities unless transitions were smoothed.
- Network and telemetry work had to be kept from disturbing the real-time control loop.

