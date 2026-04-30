# 📦 Sample — QuickstartMotion

---

## 🧠 Remarks and limitations

- This is an **AI-generated sample/template project** intended purely as a learning and quickstart reference. Before using any part of this code in a real machine or installation, a qualified automation engineer must review and validate it against the applicable safety standards (e.g. IEC 62061, ISO 13849), site-specific requirements, drive and encoder configuration, and commissioning procedures. No warranty of any kind is expressed or implied.
- Minimum TwinCAT version: **3.1.4026**
- Requires the **Tc2_MC2** library (ships with TwinCAT, no manual install needed)
- The NC axis object (`GVL.ncAxis`) must be linked to an actual NC axis in the I/O tree before activating the project — this is the only hardware-specific step.
- The project uses a generic analog drive with incremental encoder — no vendor-specific drive configuration is required.

---

## 🔧 Functionality Overview

### `FB_AxisControl`

Wraps the three core Tc2_MC2 motion function blocks (`MC_Power`, `MC_Reset`, `MC_MoveAbsolute`) for a single NC axis behind a simple, cycle-called interface.

- Internally calls all three MC function blocks every PLC cycle; operator commands are triggered via rising-edge inputs.
- Aggregates error status from all three FBs into a single `bError` / `nErrorId` output.
- Exposes live axis position and velocity from the NcToPlc structure.

**Inputs**

| I/O | Name | Type | Default | Description |
|-----|------|------|---------|-------------|
| VAR_IN_OUT | stAxis | AXIS_REF | — | NC axis reference (link to I/O tree) |
| VAR_INPUT | bEnable | BOOL | — | TRUE = servo on |
| VAR_INPUT | bEnablePositive | BOOL | — | Allow motion in positive direction |
| VAR_INPUT | bEnableNegative | BOOL | — | Allow motion in negative direction |
| VAR_INPUT | bReset | BOOL | — | Rising edge clears axis error |
| VAR_INPUT | bMoveAbsolute | BOOL | — | Rising edge starts absolute move |
| VAR_INPUT | fTargetPosition | LREAL | — | Target position [axis unit] |
| VAR_INPUT | fVelocity | LREAL | — | Move velocity [axis unit/s] |
| VAR_INPUT | fAcceleration | LREAL | — | Acceleration [axis unit/s²] |
| VAR_INPUT | fDeceleration | LREAL | — | Deceleration [axis unit/s²] |
| VAR_INPUT | fJerk | LREAL | — | Jerk limit [axis unit/s³], 0 = disabled |

**Outputs**

| I/O | Name | Type | Default | Description |
|-----|------|------|---------|-------------|
| VAR_OUTPUT | bPowered | BOOL | — | Axis servo is on |
| VAR_OUTPUT | bReady | BOOL | — | Powered and no error |
| VAR_OUTPUT | bBusy | BOOL | — | Any motion command is active |
| VAR_OUTPUT | bError | BOOL | — | Any FB has an active error |
| VAR_OUTPUT | nErrorId | UDINT | — | First non-zero error ID |
| VAR_OUTPUT | fActPosition | LREAL | — | Actual position [axis unit] |
| VAR_OUTPUT | fActVelocity | LREAL | — | Actual velocity [axis unit/s] |

### `MAIN`

Program that implements a continuous shuttle (back-and-forth) between two configurable positions using `FB_AxisControl`.

- Uses a state machine driven by `E_AxisState` (7 states: IDLE → POWERING_ON → MOVE_TO_A ↔ MOVE_TO_B, with ERROR handling).
- Operator commands: `bStart` (rising edge), `bStop` (immediate power-down), `bReset` (clear error).
- Shuttle parameters (`fPosA`, `fPosB`, `fVelocity`, `fAcceleration`, `fDeceleration`, `fJerk`) can be changed online.
- A global stop and error guard run after the state machine to catch errors and stop requests from any active state.

### `E_AxisState`

Qualified-only enumeration naming each state machine step. Values are spaced by 10 for readability (0, 10, 20, 30, 40, 50, 99).

### `GVL`

Single global variable `ncAxis : AXIS_REF` — must be linked to the NC axis in the TwinCAT I/O tree.

| Task | Cycle | Priority | Runs |
|------|-------|----------|------|
| PlcTask | 10 ms | 20 | MAIN |

---

## 🧪 Example

<!-- Minimal shuttle instantiation — already wired in MAIN -->
```iecst
PROGRAM MAIN
VAR
    fbAxis          : FB_AxisControl;
    bStart          : BOOL;
    bStop           : BOOL;
    bReset          : BOOL;
    fPosA           : LREAL := 0.0;
    fPosB           : LREAL := 100.0;
    fVelocity       : LREAL := 50.0;
    fAcceleration   : LREAL := 500.0;
    fDeceleration   : LREAL := 500.0;
    fJerk           : LREAL := 0.0;
    eState          : E_AxisState := E_AxisState.eIDLE;
    bEnableAxis     : BOOL;
    bMoveCmd        : BOOL;
    fMoveTarget     : LREAL;
END_VAR
```

```iecst
// Call FB_AxisControl at the end of every PLC cycle
fbAxis(
    stAxis          := GVL.ncAxis,
    bEnable         := bEnableAxis,
    bEnablePositive := bEnableAxis,
    bEnableNegative := bEnableAxis,
    bReset          := bReset,
    bMoveAbsolute   := bMoveCmd,
    fTargetPosition := fMoveTarget,
    fVelocity       := fVelocity,
    fAcceleration   := fAcceleration,
    fDeceleration   := fDeceleration,
    fJerk           := fJerk
);
```

To adapt this sample for your own device, link `GVL.ncAxis` to your NC axis in the I/O tree, adjust `fPosA`/`fPosB` to match your travel range, and set velocity/acceleration values appropriate for your drive and mechanics.

---

## 🧠 Notes

- `FB_AxisControl` must be called every PLC cycle — place the call as the last statement in `MAIN` so that all state-machine decisions are reflected before forwarding to the MC function blocks.
- `bMoveCmd` is asserted for exactly one cycle to generate the rising edge that `MC_MoveAbsolute` requires; it is cleared in the following wait state.
- The axis is considered ready (`bReady`) only when `MC_Power.Status` is TRUE and no error is active across any of the three MC FBs.
- All shuttle parameters can be changed online while the PLC is running — new values take effect on the next move command.

---

## 🔢 Additional information

**Required libraries**

| Library | Vendor | Purpose |
|---------|--------|---------|
| Tc2_MC2 | Beckhoff Automation GmbH | Motion control function blocks (MC_Power, MC_Reset, MC_MoveAbsolute) |
| Tc2_Standard | Beckhoff Automation GmbH | Standard PLC library |
| Tc2_System | Beckhoff Automation GmbH | System library |

**Supported platforms**: x64

**Minimum TwinCAT version**: 3.1.4026

**License**: [LICENSE.md](LICENSE.md)
