# TwinCAT MC2 Axis Shuttle — Quickstart Example

> **AI-Generated Code — Not Production Ready**
>
> This project was generated with the assistance of an AI tool (Claude, by Anthropic).
> It is intended purely as a **learning and quickstart reference**. Before using any part of
> this code in a real machine or installation, a qualified automation engineer must review and
> validate it against the applicable safety standards (e.g. IEC 62061, ISO 13849), site-specific
> requirements, drive and encoder configuration, and commissioning procedures. No warranty of
> any kind is expressed or implied.

---

This project is a minimal, self-contained TwinCAT 3 example that demonstrates how to perform
**absolute-position motion control** using the Beckhoff **Tc2_MC2** library. It implements a
continuous shuttle (back-and-forth) motion between two configurable positions, driven by a clean
state machine in `MAIN` and an encapsulated reusable function block `FB_AxisControl`.

The project targets any TwinCAT-compatible hardware (x86, x64, ARM) and uses a generic analog
drive with incremental encoder — no vendor-specific drive configuration is required.

---

## What the Sample Does

| Feature | Detail |
|---|---|
| Motion type | Absolute positioning (`MC_MoveAbsolute`) |
| Pattern | Continuous shuttle between position A and position B |
| Axis power | Managed by `MC_Power` with enable/disable handshake |
| Error recovery | `MC_Reset` triggered by operator `bReset` command |
| Operator interface | Three BOOL commands: Start, Stop, Reset |
| PLC cycle time | 10 ms |
| NC cycle time | 20 ms (SAF task) |

---

## Project Structure

```
plc/
├── POUs/
│   ├── MAIN.TcPOU          — Shuttle state machine (entry point)
│   └── FB_AxisControl.TcPOU — Reusable axis wrapper (MC_Power + MC_Reset + MC_MoveAbsolute)
├── DUTs/
│   └── E_AxisState.TcDUT   — Enumeration of state machine states
└── GVLs/
    └── GVL.TcGVL           — Global AXIS_REF (linked to the NC axis in the I/O tree)
```

### MAIN — State Machine

`MAIN` runs a seven-state shuttle loop. All operator commands and motion parameters are
declared directly in `MAIN` so they are visible in the Watch Window or any HMI.

```
eIDLE → ePOWERING_ON → eMOVE_TO_A → eWAIT_MOVE_A
                         ↑                          ↓
                     eWAIT_MOVE_B ← eMOVE_TO_B ←──┘

Any state → eERROR  (on fbAxis.bError)
eERROR    → eIDLE   (on bReset + reset complete)
Any state → eIDLE   (on bStop)
```

| State | Value | Description |
|---|---|---|
| `eIDLE` | 0 | Axis unpowered; waiting for `bStart` rising edge |
| `ePOWERING_ON` | 10 | Power enabled; waiting for `fbAxis.bReady` |
| `eMOVE_TO_A` | 20 | Issues one-cycle move command to position A |
| `eWAIT_MOVE_A` | 30 | Waits for `fbAxis.bMoveDone` |
| `eMOVE_TO_B` | 40 | Issues one-cycle move command to position B |
| `eWAIT_MOVE_B` | 50 | Waits for `fbAxis.bMoveDone`, then loops back |
| `eERROR` | 99 | Powers down; waits for operator reset |

### FB_AxisControl — Axis Wrapper

`FB_AxisControl` hides the three Tc2_MC2 function blocks behind a simple, consistent interface.
MAIN never calls `MC_Power`, `MC_Reset`, or `MC_MoveAbsolute` directly — it only uses
`FB_AxisControl`.

See **[FB_AxisControl.md](Documentation/FB_AxisControl.md)** for the full variable reference and behaviour
description.

### E_AxisState

An `{attribute 'qualified_only'}` enumeration that names each state machine step. Using named
constants instead of magic integers makes the Watch Window and error logs self-documenting.

### GVL.ncAxis

A global `AXIS_REF` variable that must be **linked to the NC axis** in the TwinCAT I/O tree
before the project can be activated. The link is the only hardware-specific step in the project.

---

## Operator Commands (Watch Window / HMI)

| Variable | Type | Action |
|---|---|---|
| `MAIN.bStart` | `BOOL` | Rising edge starts the shuttle sequence |
| `MAIN.bStop` | `BOOL` | `TRUE` immediately powers down and returns to `eIDLE` |
| `MAIN.bReset` | `BOOL` | Rising edge clears an axis error and returns to `eIDLE` |

## Shuttle Parameters

| Variable | Default | Unit | Description |
|---|---|---|---|
| `MAIN.fPosA` | `0.0` | axis unit | First shuttle position |
| `MAIN.fPosB` | `100.0` | axis unit | Second shuttle position |
| `MAIN.fVelocity` | `50.0` | unit/s | Move velocity |
| `MAIN.fAcceleration` | `500.0` | unit/s² | Acceleration ramp |
| `MAIN.fDeceleration` | `500.0` | unit/s² | Deceleration ramp |
| `MAIN.fJerk` | `0.0` | unit/s³ | Jerk limit (0 = disabled) |

All parameters can be changed online while the PLC is running. The new values take effect on
the next move command.

---

## Further Reading

- [FB_AxisControl.md](Documentation/FB_AxisControl.md) — Detailed reference for the `FB_AxisControl`
  function block: every input, output, and internal behaviour.
- [quickstart.md](Documentation/quickstart.md) — Step-by-step instructions for adding `FB_AxisControl` to
  your own project and linking it to an NC axis.

---

## Requirements

| Component | Minimum version |
|---|---|
| TwinCAT 3 XAE | 3.1.4026 |
| Tc2_MC2 library | 3.3.72.0 |
| Tc2_Standard library | 3.4.7.0 |
| Tc2_System library | 3.10.2.0 |
| Hardware | Any TwinCAT-compatible controller (x86 / x64 / ARM) |
