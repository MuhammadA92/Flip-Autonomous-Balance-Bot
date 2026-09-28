# Power Monitoring Hardware

This note describes the battery and current-monitoring subsystem I designed for Flip.

## Measurement requirements

The robot used two 7.2 V NiMH battery packs in series, giving a nominal 14.4 V supply. Three quantities were monitored:

1. total battery voltage;
2. motor-rail current and power; and
3. 5 V electronics-rail current and power.

The supplied ESP32 breakout board included a MCP3208 ADC with a 4.096 V reference. Each analogue interface therefore had to use a large portion of the available range for resolution while remaining safely below the limit.

## Final integrated signal path

I connected the battery divider and both current-sensing circuits to three channels of the **MCP3208**. The ESP32 read them sequentially over SPI at approximately 100 Hz and sent the readings to the Raspberry Pi as telemetry. In the web client, we used my measured calibration equations to reconstruct battery voltage, current, and power for display. The 5 V shunt measured logic/peripheral load; the other high-side shunt measured the motor/battery supply path.

The circuit sweeps below show how I checked the analogue conversion over its intended ranges. The battery reading measured voltage; I did not validate a full remaining-runtime estimator.

## Battery voltage

A 100 kÎ© / 22 kÎ© potential divider reduced a 15 V battery input to approximately 2.7 V. This provided a simple, safe battery-voltage measurement that could be converted back to supply voltage in software.

## High-side current sensing

The supplied power PCB already contained high-side shunt resistors:

- **0.1 Î©** on the motor rail; and
- **0.01 Î©** on the 5 V rail.

The useful signal was the small voltage difference between the two sides of each shunt. Both nodes, however, sat close to their rail voltage and could not be connected directly to the ADC.

Each current-sensing interface therefore used:

1. matched potential dividers to reduce both high-side voltages;
2. a differential amplifier to reject their common voltage and amplify the difference;
3. a non-inverting amplifier for adjustable second-stage gain;
4. output filtering to reduce high-frequency noise; and
5. a 10 kÎ© series protection resistor before the ADC.

The MCP6022 was selected because its two rail-to-rail op-amps could implement both gain stages from the available 5 V supply.

## Motor rail

Bench testing showed approximately:

- 1 A during static balancing;
- 1.5 A when the lifted robot was tilted and the wheels accelerated; and
- brief peaks around 1.7 A when wheel motion was resisted.

The design range was therefore set to 2 A. Across the 0.1 Î© shunt this corresponds to 0.2 V. The potential divider and two amplifier stages scaled this signal towards, but below, the 4.096 V ADC limit.

LTspice DC sweeps confirmed a linear response over the design range before physical construction.

## 5 V rail

The 5 V rail measured approximately 5.2 V and supplied the ESP32, Raspberry Pi, and peripherals. The initial 1.5 A design range was extended to the converter's 3 A capability. This required reducing the amplifier gain so the output would not saturate at the maximum 0.03 V shunt drop.

The final LTspice sweep again showed a linear output across most of the ADC range.

## Construction and calibration

I prototyped both circuits on breadboard and then transferred them to perfboard. When the intended 2 MÎ© resistors were unavailable, I measured the available 1.8 MÎ©, 10%-tolerance parts and selected closely matched pairs to preserve differential-amplifier accuracy.

I used two bench supplies to simulate the voltages at either side of each shunt. I measured the input differences with a multimeter and recorded the circuit outputs across a sweep. The maximum outputs differed slightly from simulation, so I adjusted the second-stage gains before final assembly.

The resulting calibration equations were:

```text
Motor rail: Vout = 16.936 * delta_Vin + 0.2617    (RÂ² = 0.9994)
5 V rail:   Vout = 97.936 * delta_Vin + 0.0179    (RÂ² = 0.9981)
```

The non-zero intercepts were handled in software when converting ADC readings into shunt voltage. Current and power were then calculated as:

```text
current = shunt_voltage / shunt_resistance
power   = current * rail_voltage
```

The practical sweeps confirmed that both circuits were sufficiently linear and remained within the ADC input range across their intended operating regions.
