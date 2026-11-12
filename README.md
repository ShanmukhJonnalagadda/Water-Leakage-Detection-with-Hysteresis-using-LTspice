# Water Leakage Detection with Hysteresis Using LTspice

An analog water-leak alarm designed and simulated in **LTspice**. The circuit uses a modeled resistive water sensor, an **LT1011 comparator**, and a **2N3904 transistor** to drive an LED. A second design adds **positive feedback (47 kΩ)** to create hysteresis and reduce unwanted switching when the sensor signal is noisy.

> **Scope:** Simulation and circuit analysis. The water sensor is represented by an electrical model; this is not yet a tested physical water-detection device.

## Project overview

A conductive water probe can exhibit high resistance when dry and lower resistance when bridged by water. A voltage divider translates that resistance change into a voltage for a comparator. When the sensed voltage drops below a threshold, the comparator drives a transistor and LED alarm.

The project explores two questions:

1. Can the circuit distinguish a simulated dry condition from a wet condition and activate an alarm?
2. Can comparator hysteresis prevent repeated alarm toggling when the sensor signal fluctuates near the detection threshold?

## Repository files

| File | Purpose |
| --- | --- |
| [`Water_Leak_Detection.asc`](Water_Leak_Detection.asc) | Resistive sensor model with a time-dependent dry-to-wet transition at 10 ms. |
| [`Water_Leak_Detection_Hysteresis.asc`](Water_Leak_Detection_Hysteresis.asc) | Comparator and alarm stage with a 47 kΩ positive-feedback resistor; currently configured for a triangular-input test. |
| `.gitignore` | Excludes large/generated LTspice outputs (`.raw`, `.log`, `.net`, etc.). |

## Circuit design

### 1. Water-leak detector

The main schematic uses the following nominal values:

| Component | Value / model | Function |
| --- | --- | --- |
| V1 | 5 V DC | Supply |
| R1 | 820 kΩ | Sensor divider pull-up |
| Rw | `R=if(time<10m,1Meg,10k)` | Simulated water-sensor resistance: dry to wet at 10 ms |
| R2 | 100 kΩ | Series resistance below Rw; keeps the modeled wet input above ground |
| R3, R4 | 10 kΩ each | Approximately 2.5 V reference divider |
| U2 | LT1011 | Comparator |
| R5 | 10 kΩ | Comparator output pull-up |
| R7 | 10 kΩ | Transistor base drive resistance |
| Q1 | 2N3904 | LED switching transistor |
| R6 | 1 kΩ | LED current limiting |
| D1 | QTLP690C | LED indicator model |

The sensor voltage for this topology is approximately

\[
V_{\mathrm{sense}}=5\,\mathrm{V}\,\frac{R_w+100\,\mathrm{k\Omega}}{820\,\mathrm{k\Omega}+R_w+100\,\mathrm{k\Omega}}.
\]

With the chosen *illustrative* sensor resistances:

| Condition | Rw | Calculated Vsense | Alarm behavior observed in simulations |
| --- | ---: | ---: | --- |
| Dry | 1 MΩ | ≈2.865 V | LED off |
| Wet | 10 kΩ | ≈0.591 V | LED on (approximately 3.2 mA) |

The sensed voltage is compared against a nominal 2.5 V reference. In the schematic the time-varying resistor changes at **10 ms**, providing a repeatable test event.

### 2. Hysteresis and noise immunity

The second schematic substitutes a controlled input source (**V2**) for the resistive probe, allowing the comparator's switching thresholds to be measured independently. It adds:

- **R8 = 47 kΩ** between the comparator output and the non-inverting reference input (positive feedback).
- A 5 V supply and two 10 kΩ reference resistors.
- The same LT1011 → 2N3904 → LED output principle.

With feedback, the switching threshold depends on the comparator's current output state. From the LTspice tests discussed for this design, the approximate switching points were:

| Input movement | LED transition | Approximate Vsense |
| --- | --- | ---: |
| Decreasing | Off → on | 2.275 V |
| Increasing | On → off | 2.531 V |

Thus the **estimated hysteresis width** is

\[
\Delta V_H\approx2.531-2.275=0.256\,\mathrm{V}\quad(256\,\mathrm{mV}).
\]

These values are approximate results inferred from the simulation waveforms, not calibrated physical measurements. The actual comparator output is loaded by the transistor drive network, so the thresholds should be checked in each simulation setup.

## Simulation experiments

### Experiment A — Dry and wet operating points

Run `.op` with Rw set to `1Meg` and then `10k` (or compare the steady states in a transient simulation). Inspect `V(v_sense)`, the reference voltage, and `I(D1)`.

**Expected:** the LED stays off with high sensor resistance and conducts when the resistance falls.

### Experiment B — Automatic leak event

Open [`Water_Leak_Detection.asc`](Water_Leak_Detection.asc). Its resistor model changes from **1 MΩ** to **10 kΩ** at **10 ms**. Run the transient analysis and inspect:

- `V(v_sense)` — sensor-divider output;
- the 2.5 V reference node;
- `I(D1)` — alarm LED current.

**Observed behavior in the project tests:** sensor voltage fell from approximately **2.865 V** to **0.591 V** and LED current rose from essentially zero to approximately **3.2 mA**.

### Experiment C — Thresholds with and without hysteresis

Open [`Water_Leak_Detection_Hysteresis.asc`](Water_Leak_Detection_Hysteresis.asc). Its V2 source is currently configured as:

```spice
PWL(0 3 10ms 2 20ms 3)
```

Run:

```spice
.tran 0 20m 0 1u
```

The input falls linearly from **3 V to 2 V** and rises back to **3 V**. Compare the comparator with **R8 = 47 kΩ** to a copy with R8 removed. The circuit without feedback switches near one 2.5 V level; the feedback version changes state at different rising and falling thresholds.

### Experiment D — Noise immunity

In separate copies of the comparator test schematic, replace V2's triangle-wave value with:

```spice
SINE(2.5 0.02 1k)
```

This is a **1 kHz, 40 mV peak-to-peak** signal (2.48–2.52 V). Run the same transient analysis with and without R8, keeping all other conditions the same.

**Observed behavior in the project comparison:** the no-hysteresis version repeatedly switched its LED, while the hysteresis version maintained its initial LED-off state under the same noise input. This demonstrates improvement for the *tested signal and initial state*, not immunity to every possible disturbance.

## Results summary

| Test | Key result |
| --- | --- |
| Dry vs wet | Sensor resistance controls comparator and LED alarm state |
| Leak event | Alarm activates after modeled wet transition at 10 ms |
| Hysteresis | Approximately 2.275 V turn-on and 2.531 V turn-off thresholds |
| Noise test | Repeated comparator toggling suppressed for the specified 1 kHz, ±20 mV input |

## How to reproduce

1. Install [LTspice](https://www.analog.com/en/resources/design-tools-and-calculators/ltspice-simulator.html).
2. Clone or download this repository.
3. Open either `.asc` file in LTspice. Both use standard LTspice component models, including the **LT1011**.
4. Click **Run**, then plot `V(v_sense)` and `I(D1)`. Label additional nodes if desired to make the reference and output traces easier to find.
5. For the noise comparison, create a copy of the hysteresis schematic and remove R8 from the copy only. Change V2 in **both** copies to the same sine-wave expression and compare results.

## Screenshots

Simulation screenshots are not yet checked into this repository. Recommended figures to add in a `results/` directory:

- `dry_wet_operating_points.png`
- `leak_event_transient.png`
- `hysteresis_triangle_comparison.png`
- `noise_immunity_comparison.png`

Once added, embed the images here using Markdown (`![Description](results/filename.png)`).

## Engineering limitations and next steps

- **Water-sensor model:** 1 MΩ / 10 kΩ are assumed resistances, not measurements from a particular probe. Real resistance depends on water conductivity, electrode geometry, contamination, and corrosion.
- **Comparator operating limits:** The initial 680 kΩ divider allowed a wet-state voltage near 0.073 V, outside the LT1011's specified common-mode range on a single 5 V supply. The updated 820 kΩ / 100 kΩ network targets approximately 0.591 V wet and 2.865 V dry; hardware use still requires datasheet checks and validation.
- **Switching behavior:** Investigate worst-case component tolerances, supply variation, slow sensor transitions, startup state, and higher-amplitude noise.
- **Physical prototype:** Validate with real electrodes, an appropriate power supply, and safe test conditions before using the circuit for property protection. This simulation is not a certified water-alarm product.

## Tools and skills

**LTspice**, analog voltage dividers, LT1011 comparator modeling, positive feedback/hysteresis, 2N3904 transistor switching, DC operating-point analysis, transient simulation, waveform interpretation, and circuit verification.

## Reference

- [Analog Devices — LTspice](https://www.analog.com/en/resources/design-tools-and-calculators/ltspice-simulator.html)
- [Analog Devices — LT1011 product information](https://www.analog.com/en/products/lt1011.html)

