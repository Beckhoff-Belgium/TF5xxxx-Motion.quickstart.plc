# FB_AxisControl — Function Block Reference

> **AI-Generated — Not Production Ready:** This document and the accompanying code were
> produced with AI assistance. Review by a qualified engineer is required before any
> industrial or safety-related use.

`FB_AxisControl` is a thin, reusable wrapper around the three most common Beckhoff **Tc2_MC2**
motion-control function blocks:

| Internal FB | Purpose |
|---|---|
| `MC_Power` | Servo enable / disable |
| `MC_Reset` | Clear axis error |
| `MC_MoveAbsolute` | Commanded absolute positioning |

The wrapper presents a single, consistent interface so application code (e.g. `MAIN`) never
needs to instantiate or call these blocks directly. All three FBs share one `AXIS_REF` and are
called every PLC cycle.

---

## Declaration

```pascal
FUNCTION_BLOCK FB_AxisControl
```

The block is instantiated once per physical axis:

```pascal
fbAxis : FB_AxisControl;
```

---

## VAR_IN_OUT

| Name | Type | Description |
|---|---|---|
| `stAxis` | `AXIS_REF` | NC axis reference. Must be linked to the NC axis object in the I/O tree (e.g. via `GVL.ncAxis`). Passed by reference so the MC function blocks can update the structure in place. |

`AXIS_REF` is a Beckhoff-defined structure. It carries both the PLC→NC command channel
(`PlcToNc`) and the NC→PLC status channel (`NcToPlc`). Linking this variable in the I/O tree
is the only hardware-coupling step required.

---

## VAR_INPUT — Power Control

| Name | Type | Default | Description |
|---|---|---|---|
| `bEnable` | `BOOL` | `FALSE` | `TRUE` = servo on. Drive continuously at `MC_Power`. |
| `bEnablePositive` | `BOOL` | `TRUE` | Allow motion in the positive direction. |
| `bEnableNegative` | `BOOL` | `TRUE` | Allow motion in the negative direction. |

`bEnablePositive` and `bEnableNegative` default to `TRUE` inside the FB declaration so the
caller only needs to drive `bEnable`.

---

## VAR_INPUT — Reset

| Name | Type | Description |
|---|---|---|
| `bReset` | `BOOL` | Rising edge triggers `MC_Reset`. Clears axis error and re-arms the axis. |

`MC_Reset` is edge-triggered internally. The caller must hold `bReset` `TRUE` for at least one
PLC cycle and then release it; or pulse it for exactly one cycle.

---

## VAR_INPUT — Absolute Move

| Name | Type | Unit | Description |
|---|---|---|---|
| `bMoveAbsolute` | `BOOL` | — | Rising edge starts an absolute move. Must be a one-cycle pulse; the state machine in `MAIN` guarantees this. |
| `fTargetPosition` | `LREAL` | axis unit | Commanded target position. |
| `fVelocity` | `LREAL` | unit/s | Move velocity (must be > 0). |
| `fAcceleration` | `LREAL` | unit/s² | Acceleration ramp. |
| `fDeceleration` | `LREAL` | unit/s² | Deceleration ramp. |
| `fJerk` | `LREAL` | unit/s³ | Jerk limit. `0` disables jerk limiting (trapezoidal profile). |

`MC_MoveAbsolute` uses `BufferMode := MC_Aborting`, so a new move command immediately cancels
any move currently in progress.

---

## VAR_OUTPUT — Axis Status

| Name | Type | Description |
|---|---|---|
| `bPowered` | `BOOL` | `MC_Power.Status` — drive is energised and accepting commands. |
| `bReady` | `BOOL` | `bPowered AND NOT bError` — safe to issue motion commands. |
| `bBusy` | `BOOL` | `bResetBusy OR bMoveBusy` — at least one motion FB is active. |

`bReady` is the recommended gate before issuing any move command. Check it in your state
machine after enabling the axis.

---

## VAR_OUTPUT — MC_Reset Feedback

| Name | Type | Description |
|---|---|---|
| `bResetDone` | `BOOL` | Reset completed successfully. |
| `bResetBusy` | `BOOL` | Reset in progress. |
| `bResetError` | `BOOL` | `MC_Reset` reported an error. |
| `nResetErrorId` | `UDINT` | ADS error ID from `MC_Reset`. `0` = no error. |

---

## VAR_OUTPUT — MC_MoveAbsolute Feedback

| Name | Type | Description |
|---|---|---|
| `bMoveDone` | `BOOL` | Target position reached. Cleared on the next move command. |
| `bMoveBusy` | `BOOL` | Move in progress. |
| `bMoveActive` | `BOOL` | Move command is currently active on the NC axis. |
| `bMoveError` | `BOOL` | `MC_MoveAbsolute` reported an error. |
| `nMoveErrorId` | `UDINT` | ADS error ID from `MC_MoveAbsolute`. `0` = no error. |

---

## VAR_OUTPUT — Aggregated Error

| Name | Type | Description |
|---|---|---|
| `bError` | `BOOL` | `TRUE` if any internal FB (`MC_Power`, `MC_Reset`, or `MC_MoveAbsolute`) is in error. |
| `nErrorId` | `UDINT` | First non-zero error ID among the three FBs. Priority: `MC_Power` > `MC_Reset` > `MC_MoveAbsolute`. |

Monitor `bError` in your state machine to detect faults. When `bError` is `TRUE`, issue
`bReset` and wait for `bResetDone` before resuming motion.

---

## VAR_OUTPUT — Live Axis Data

| Name | Type | Unit | Source |
|---|---|---|---|
| `fActPosition` | `LREAL` | axis unit | `stAxis.NcToPlc.ActPos` |
| `fActVelocity` | `LREAL` | unit/s | `stAxis.NcToPlc.ActVelo` |

These are raw values read from the NC→PLC status structure every cycle. They reflect the
encoder-measured position and velocity regardless of whether a move command is active.

---

## Internal Behaviour

### Execution Order

Each PLC cycle the FB performs the following steps in order:

1. **`stAxis.ReadStatus()`** — refresh the `NcToPlc` structure from the NC task.
2. **`fbMcPower()`** — continuously drive servo enable.
3. **`fbMcReset()`** — pass `bReset` through; `MC_Reset` detects its own rising edge.
4. **`fbMcMoveAbsolute()`** — pass `bMoveAbsolute` through; `MC_MoveAbsolute` detects its own rising edge.
5. Aggregate outputs (`bPowered`, `bReady`, `bBusy`, `bError`, `nErrorId`, `fActPosition`, `fActVelocity`).

### MC_Power Configuration

```pascal
fbMcPower(
    Axis            := stAxis,
    Enable          := bEnable,
    Enable_Positive := bEnablePositive,
    Enable_Negative := bEnableNegative,
    Override        := LREAL#100.0,       // full velocity override
    BufferMode      := MC_Aborting
);
```

`Override` is fixed at 100 %. If you need dynamic override, expose it as an additional input.

### MC_MoveAbsolute Configuration

```pascal
fbMcMoveAbsolute(
    Axis         := stAxis,
    Execute      := bMoveAbsolute,        // rising-edge triggered
    Position     := fTargetPosition,
    Velocity     := fVelocity,
    Acceleration := fAcceleration,
    Deceleration := fDeceleration,
    Jerk         := fJerk,
    BufferMode   := MC_Aborting
);
```

`MC_Aborting` means a new `Execute` rising edge immediately overrides any in-flight move.

### Error ID Priority

```pascal
nErrorId := SEL(fbMcPower.ErrorID <> 0,
              SEL(nResetErrorId <> 0,
                nMoveErrorId,
                nResetErrorId),
              fbMcPower.ErrorID);
```

The first non-zero error ID wins. This gives a single `nErrorId` output that always points to
the highest-priority fault.

---

## Usage Pattern

```pascal
// Declare
fbAxis : FB_AxisControl;

// Wire the global AXIS_REF
fbAxis.stAxis REF= GVL.ncAxis;   // or pass as VAR_IN_OUT in the FB call

// Call every cycle (pass all inputs as named parameters)
fbAxis(
    stAxis           := GVL.ncAxis,
    bEnable          := bEnableAxis,
    bReset           := bResetCmd,
    bMoveAbsolute    := bMoveCmd,
    fTargetPosition  := fTarget,
    fVelocity        := 50.0,
    fAcceleration    := 500.0,
    fDeceleration    := 500.0,
    fJerk            := 0.0
);

// Read outputs
IF fbAxis.bError THEN
    // handle fault
END_IF
IF fbAxis.bMoveDone THEN
    // move finished
END_IF
```

For a complete working example see `MAIN.TcPOU` and the step-by-step guide in
[quickstart.md](quickstart.md).

---

## Extending the Block

Because `FB_AxisControl` is a thin wrapper, common extensions are straightforward:

| Extension | How |
|---|---|
| Homing | Add `fbMcHome : MC_Home` and expose `bHome` / `bHomeDone` outputs |
| Jogging | Add `fbMcJog : MC_Jog` and expose `bJogForward` / `bJogBackward` inputs |
| Velocity override | Expose `fOverride : LREAL` input; pass to `MC_Power.Override` |
| Halt | Add `fbMcHalt : MC_Halt` for a controlled deceleration stop |
| Relative move | Add `fbMcMoveRelative : MC_MoveRelative` alongside the absolute block |
