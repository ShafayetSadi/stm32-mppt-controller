# STM32 MPPT Controller

## System Specifications

### Solar PV Panel

| Parameter                            | Walton 100 W panel |
| ------------------------------------ | -----------------: |
| Maximum power, \(P_{max}\)           |          **100 W** |
| Voltage at maximum power, \(V_{mp}\) |        **21.05 V** |
| Current at maximum power, \(I_{mp}\) |         **4.75 A** |
| Open-circuit voltage, \(V_{oc}\)     |        **24.48 V** |
| Short-circuit current, \(I_{sc}\)    |         **5.17 A** |
| Cell configuration                   | **36 cells (9×4)** |
| Cell type                            |          Mono PERC |
| Cell efficiency                      |                23% |
| Temperature coefficient              |      **-0.34%/°C** |
| Panel dimensions                     |  690 × 780 × 30 mm |
| Weight                               |           ~5.97 kg |
| Rated power tolerance                |                ±3% |

### Battery

| Parameter                 |            Specification |
| ------------------------- | -----------------------: |
| Model                     |       **Walton WBU1218** |
| Type                      |  VRLA / sealed lead-acid |
| Nominal voltage           |                 **12 V** |
| Capacity                  | **18 Ah @ 20-hour rate** |
| Maximum charging current  |                **5.4 A** |
| Cycle-use voltage         |   **14.4–15.0 V @ 25°C** |
| Standby/float voltage     |   **13.5–14.8 V @ 25°C** |
| Walton listed float range |          **14.3–14.8 V** |
| Weight                    |                 ~5.25 kg |

### Power Converter

| Parameter                         |     Design specification |
| --------------------------------- | -----------------------: |
| Converter topology                |       **Buck converter** |
| Input source                      |                 Solar PV |
| Nominal input                     |                    ~21 V |
| Maximum expected PV voltage       |                  24.48 V |
| Design input voltage              |             **~18–25 V** |
| Recommended design voltage rating |                **≥30 V** |
| Output                            |             12 V battery |
| Maximum charging voltage          |     Initially **14.4 V** |
| Maximum charging current          | **5.0 A software limit** |
| Target output power               |   **~70–75 W practical** |
| Power-stage design rating         |                **100 W** |
| Target efficiency                 |                 **≥90%** |

- Maximum regulated battery charging power: ~72 W at 14.4 V and 5 A

### Switching Stage

- Asynchronous buck converter

```text
PV+
 │
 │
MOSFET
 │
 ├──────── Inductor ──────── Battery+
 │
Diode
 │
GND
```

### Switching Frequency

- initial switching frequency $f_{s} = 100 kHz$

### PWM

| Parameter                |                Target |
| ------------------------ | --------------------: |
| PWM frequency            | **100 kHz initially** |
| PWM source               |           STM32 timer |
| Duty-cycle control       |               Digital |
| Duty-cycle range         |               ~0–100% |
| Initial operating region |               ~50–80% |

### Measurement specifications

| Measurement | Purpose                             |
| ----------- | ----------------------------------- |
| \(V_{PV}\)  | Determine PV operating voltage      |
| \(I_{PV}\)  | Determine PV power                  |
| \(V_{BAT}\) | Battery charging control            |
| \(I_{BAT}\) | Current limiting / charging control |
| Temperature | Optional protection                 |

Then,

$$
P_{PV} = V_{PV} \cdot I_{PV}
$$

and,

$$
P_{BAT} = V_{BAT} \cdot I_{BAT}
$$

### ADC Specifications

| Parameter       |                             Target |
| --------------- | ---------------------------------: |
| MCU             |                  **STM32F103C8T6** |
| ADC             |                 Internal STM32 ADC |
| ADC resolution  |                         **12-bit** |
| PV voltage      |                                ADC |
| PV current      |                                ADC |
| Battery voltage |                                ADC |
| Battery current |                                ADC |
| Reference       |                              3.3 V |
| Sampling        | Periodic synchronized measurements |

### MPPT Specifications

- For version 1, Perturb & Observe algorithm.

Conceptually:

```text
Measure Vpv, Ipv
       │
       ▼
 Calculate P
       │
       ▼
Compare with
previous P
       │
┌──────┴──────┐
▼             ▼
P increased    P decreased
│             │
▼             ▼
Continue         Reverse
direction        direction
│             │
└──────┬──────┘
       ▼
  Update PWM
```

### Battery Charging Control

- We will implement constant current (CC) + constant voltage (CV) charging control.

So,

$$
CC \rightarrow CV \rightarrow Float
$$

**CC**
Maximum charging current:

$$
I_{BAT} <= 5.0 A
$$

**CV**
Target battery voltage:

$$
V_{BAT} = 14.4 V
$$

This is the lower end of Walton's 14.4–15.0 V cycle-use range. It is an initial
engineering assumption.

**Float**
We'll eventually choose a value within the manufacturer's stated float/standby
range, but we should label the exact value as our engineering implementation
choice, because Walton hasn't given us enough information to claim a single
exact recommended float setpoint.

### Protection Specifications

| Protection              | Initial target         |
| ----------------------- | ---------------------- |
| PV over-voltage         | Shutdown               |
| PV over-current         | Shutdown/current limit |
| Battery over-voltage    | Stop charging          |
| Battery over-current    | Limit/shutdown         |
| MOSFET over-temperature | Shutdown               |
| Reverse battery         | Protection             |
| Short circuit           | Current limit/shutdown |
| Software fault          | Watchdog               |
| Startup                 | Soft-start             |

## Power Stage Design

### Buck Topology

$$
100 W PV \rightarrow Buck Converter \rightarrow 12 V, 18 Ah Battery
$$

We will use a buck converter because our PV panel operates at a higher voltage
than our battery, so we need to step it down to the battery voltage.

Our asynchronous buck topology will be connected to the PV input and battery
output as shown below.

```text
                         Q1
                    ┌────┤
                    │    │
PV+ ────────┬───────┤    ├─────●────── L ────────┬─── BAT+
            │       └────┘     │                 │
            CIN                SW               COUT
            │                  │                 │
            │                  └───────|<|───────┤
            │                           D1       │
PV− ────────┴────────────────────────────────────┴─── BAT−
```

Where:

- Q1 = main power MOSFET
- D1 = freewheeling diode
- L = output/energy-storage inductor
- CIN = input capacitor
- COUT = output capacitor
- SW = switching node

The battery is connected at the output.

The STM32 controlls the power MOSFET and freewheeling diode to regulate the
output voltage.

$$
D = \frac{T_{ON}}{T}
$$

Where:

- $D$ = duty cycle
- $T_{ON}$ = on-time
- $T$ = period

Ideal buck equation:

$$
V_{OUT} = D V_{IN}
$$

therefore:

$$
D = \frac{V_{OUT}}{V_{IN}}
$$

At the PV maximum-power point

Our panel's nominal MPP is:

$$ V_{IN}=V_{MP}=21.05V $$

Our initial CV target is:

$$ V_{OUT}=14.4V $$

Therefore:

$$ D=\frac{14.4}{21.05} $$ $$ \boxed{D\approx0.684} $$

or:

$$ \boxed{D\approx68.4\%} $$

So when the PV is around its nominal MPP and the battery is at 14.4 V, our ideal
buck would operate around 68% duty cycle.

But the battery voltage is not exactly 14.4 V, so the actual duty cycle will be
slightly different.

### Our preliminary operating envelope

| Parameter                        |    Design value |
| -------------------------------- | --------------: |
| PV nominal MPP voltage           |     **21.05 V** |
| PV nominal MPP current           |      **4.75 A** |
| PV STC open-circuit voltage      |     **24.48 V** |
| Preliminary PV design range      |     **18–25 V** |
| Battery nominal voltage          |        **12 V** |
| Battery normal charging range    |  **~12–14.4 V** |
| Deep-discharge design envelope   |     **~10.5 V** |
| CV target                        |      **14.4 V** |
| Maximum battery charging current |       **5.0 A** |
| Converter design power           | **100 W class** |
| Switching frequency              |     **100 kHz** |
| Approx. duty at 21.05 → 12 V     |       **57.0%** |
| Approx. duty at 21.05 → 14.4 V   |       **68.4%** |
| Approx. maximum duty             |         **80%** |

## Worst-Case Operating Conditions

### Input Voltage Range

The PV operating point changes continuously because of:

- sunlight
- temperature
- MPPT operation
- battery condition
- converter losses

Therefore, we will use a range of:

$$
V_{IN} = 18 - 25 V
$$

### Output Voltage Range

Our battery is nominally 12 V, so the output voltage range will be:

$$
V_{OUT} = 10.5 - 14.4 V
$$

Where:

- $10.5 V$ = conservative deep-discharge design point
- $12-13.x V$ = normal battery operation range
- $14.4 V$ = maximum battery charging voltage, our initial CV target

Again, 10.5 V isn't a recommended battery operating voltage. It's a power-stage
design envelope.

### Duty-cycle range

For an ideal buck:

$$
D = \frac{V_{OUT}}{V_{IN}}
$$

Minimum duty cycle,

$$
D_{min} = \frac{10.5}{25} \\
D_{min} = 42%
$$

Maximum duty cycle,

$$
D_{max} = \frac{14.4}{25} \\
D_{max} = 80%
$$

Therefore, the duty cycle range is $42%$ to $80%$.

### Nominal Duty Cycle

At the panel's MPP, V_in = 21.05 V and battery at CV, V_out = 14.4 V

From the duty cycle equation,

$$
D = \frac{14.4}{21.05} \\
D = 68.4 %
$$

So, our operating point is approximately,

$$
V_{IN} = 21.05 V \\
V_{OUT} = 14.4 V \\
D = 68.4 %
$$

### Output Current

Our firmware limit is,

$$
I_{OUT,max} = 5 A
$$

So, the maximum battery charging power at 14.4V is,

$$
P_{OUT,max} = 14.4 \times 5 A = 72 W
$$

This is our controlled battery power.

### Input Current

The buck converter doesn't create energy. Ignoring losses,

$$
P_{IN} = P_{OUT} \\
V_{IN}I_{IN} = V_{OUT}I_{OUT} \\
I_{IN} = \frac{V_{OUT}}{V_{IN}}I_{OUT}
$$

So, we get,

$$
I_{IN} = \frac{14.4 x 5}{21.05} \\
I_{IN} = 3.42 A
$$

But our panel can provide 4.75 A at MPP, but our battery is limited to 5 A, so
we don't necessarily need to draw the entire 4.75 A at MPP.

## Converter efficiency

Let's assume, $\eta = 90%$, then,

$$
P_{OUT} = P_{IN} \eta \\
P_{IN} = \frac{P_{OUT}}{\eta} \\
P_{IN} = \frac{72 W}{0.9} \\
P_{IN} = 80 W
$$

At 21.05 V,

$$
I_{IN} = \frac{80}{21.05} \\
I_{IN} = 3.82 A
$$

### Components should handle more than 3.8 A

The panel itself can supply $I_{SC} = 5.17A$ and potentially the converter can
experience transient current spikes above its normal operating current.
So, we need to select components accordingly.

Input current capability, $ >= 6 A$
Output current capability, $ >= 5 A$

### Inductor Current

## PWM Control Implementation

## P&O MPPT Control Implementation

## Battery Charging Control

## Protections

## Experimental Results
