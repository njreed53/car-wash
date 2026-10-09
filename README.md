# Automatic Car Wash

A PLC simulation of an automatic car wash built in RSLogix 500 and tested with Emulate 500. Uses state machine sequencing, safety interlocks, timers/counters, and analog signal handling.

**Important:** This is a simulation. All I/O uses internal files so RSLogix Emulate works without real hardware.

[Full Ladder Logic Program](./docs/car-wash.pdf)
---

## Setup

RSLogix 500 (v17+) for programming Emulate 500 for running the simulation Language: Ladder Logic

Program is organized into three routines called from MAIN using JSR:

- IO – digital and analog I/O mapping
- CONTROLS – state machine and sequencer logic
- ALARMS – fault detection and safety monitoring

---

## I/O Mapping

[IO Ladder](./docs/IO.pdf)

### Digital (Binary File B9:0)

| Address | Signal                     | Type   | Notes                         |
| ------- | -------------------------- | ------ | ----------------------------- |
| B9:0/0  | Start Button               | Input  | Starts the cycle              |
| B9:0/1  | Stop Button                | Input  | Stops current operation       |
| B9:0/2  | E-Stop                     | Input  | Shuts everything down         |
| B9:0/3  | Front Gate Sensor          | Input  | Vehicle at entry              |
| B9:0/4  | Rear PhotoEye              | Input  | Tracks vehicle through tunnel |
| B9:0/5  | Water Pressure Fault Reset | Input  | Clears pressure fault         |
| B9:0/6  | Motor Temp Fault Reset     | Input  | Clears temp fault             |
| B9:0/7  | Emergency Reset            | Input  | Clears all faults             |
| B9:0/8  | Water Pump                 | Output | Water system                  |
| B9:0/9  | Soap Dispenser             | Output | Chemical injection            |
| B9:0/10 | Dryer                      | Output | Air blowers                   |
| B9:0/11 | Conveyor                   | Output | Moves vehicle                 |
| B9:0/12 | Pressure Fault             | Input  | Low pressure warning          |
| B9:0/13 | Temperature Fault          | Input  | Motor overheating             |
| B9:0/14 | E-Stop Fault               | Input  | E-stop triggered              |

### Analog (Integer Files)

|Address|Parameter|Range|
|---|---|---|
|N10:0|Water Pressure|0–2000 psi|
|N10:1|Motor Temperature|0–300°F|

---

## Operation States

|State|Timer|Active Outputs|Transitions To|
|---|---|---|---|
|IDLE|—|Nothing|ENTERING (on Start)|
|ENTERING|Gate delay (~2 sec)|Front gate closed|WASH (after delay)|
|WASH|15 sec|Spray valve, pump 1|SOAP|
|SOAP|5 sec|Soap valve, brush motor|RINSE|
|RINSE|15 sec + conveyor timer|Water valve, pump 2|DRY|
|DRY|Dry timer|Blower fans|EXIT|
|EXIT|Exit timer|Rear gate open, conveyor|IDLE + car counter increment|
|FAULT|—|Front gate open|IDLE (on Emergency Reset)|

---

## Controls Ladder Logic

[Controls Ladder](./docs/controls.pdf)

Each rung handles one state transition:

|Rung|State|Condition|Action|
|---|---|---|---|
|0|IDLE|Start button pressed|Move to ENTERING state|
|1|ENTERING|Rear photoeye activated + stabilization delay|LATCH gate closed, wait ~2 sec, then move to WASH|
|2|WASH|15-second timer done|Move to SOAP state|
|3|SOAP|5-second timer done|Move to RINSE state|
|4|RINSE|15-second + conveyor timer done|Move to DRY state|
|5|DRY|Dry timer done|Move to EXIT state|
|6|EXIT|Exit timer done|Unlatch gate open, increment car counter, return to IDLE|
|7|FAULT|Any fault active|Unlatch gate open, wait for Emergency Reset|

Gate control uses OTL/OTU (Latch/Unlatch) instructions for persistent state:

- LATCH on in ENTERING after rear photoeye triggers
- UNLATCH on in EXIT and FAULT states

---

## Alarms Ladder Logic

[Alarms Ladder](./docs/alarms.pdf)

Three fault conditions monitored:

1. Low water pressure (below minimum threshold)
2. Motor temperature too high (above max threshold)
3. E-Stop pressed

Each fault uses motor interlocking circuitry so the condition latches and runs continuously until its specific reset button is pressed. All faults trigger a MOVE instruction that sets the system to FAULT state.

---

## Safety Features

- E-Stop cuts power to all outputs immediately
- Brushes won't start if conveyor is running
- Motor over-temperature disables operation
- Low water pressure stops the pumps
- Faults require manual reset before continuing
- Front gate latched closed during all wash cycles
