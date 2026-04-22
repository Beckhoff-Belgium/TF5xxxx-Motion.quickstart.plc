# Quickstart — Using FB_AxisControl in Your Project

> **AI-Generated — Not Production Ready:** This guide and the accompanying code were
> produced with AI assistance. Review by a qualified engineer is required before any
> industrial or safety-related use.

This guide walks you through adding `FB_AxisControl` to a new or existing TwinCAT 3 PLC
project and connecting it to an NC axis. It assumes TwinCAT 3 XAE 3.1.4026 or later.

---

## Prerequisites

- TwinCAT 3 XAE installed
- A TwinCAT project with at least one PLC project
- The `Tc2_MC2` library added to your PLC project (see Step 1)
- An NC task present in the system configuration (see Step 2)

---

## Step 1 — Add the Tc2_MC2 Library

`FB_AxisControl` depends on `Tc2_MC2`. If it is not already referenced:

1. In the **Solution Explorer**, expand your PLC project and open **References**.
2. Right-click **References** → **Add library…**
3. Search for **Tc2_MC2** and select the latest available version (≥ 3.3.72.0).
4. Click **OK**.

Repeat for `Tc2_Standard` and `Tc2_System` if they are missing.

---

## Step 2 — Add an NC Task

An NC task provides the real-time motion control layer that the PLC commands.

1. In the **Solution Explorer**, right-click the TwinCAT system node (e.g. **SimpleNC**).
2. Select **Add** → **NC/PTP NCI Configuration**.
3. A node named **NC-Task 1 SAF** appears under the system. This is the NC task.

> **Cycle times:** The NC SAF task default is 2 ms. Set it to match your requirements.  
> The PLC task cycle time must be a multiple of the NC task cycle time.

---

## Step 3 — Add an NC Axis

1. Expand the **NC-Task 1 SAF** node.
2. Right-click **Axes** → **Add New Item…** → select **Continuous Axis**.
3. A new axis (e.g. **Axis 1**) appears. Note its **Name** and **ID**.

Configure the axis in its property pages:
- **Settings** tab: set encoder resolution, axis type (rotary / linear), unit.
- **Enc** tab: select encoder type and link to hardware I/O.
- **Drive** tab: select drive type and link to hardware I/O.
- **Controller** tab: tune P/I gains once hardware is connected.

For simulation (no hardware), leave **Enc** and **Drive** unlinked — the axis runs in
simulation mode automatically.

---

## Step 4 — Declare a Global AXIS_REF

The `AXIS_REF` type is a Beckhoff structure that bridges the PLC and the NC task. Declare one
global instance per physical axis.

In your **GVL** (Global Variable List):

```pascal
VAR_GLOBAL
    {attribute 'TcLinkTo' := '/^NC-Task 1 SAF^Axes^Axis 1'}
    ncAxis : AXIS_REF;
END_VAR
```

> The `TcLinkTo` attribute is an alternative to the drag-and-drop link in Step 5. You can use
> either method — do not use both at the same time.

If you prefer the drag-and-drop approach, declare the variable without the attribute:

```pascal
VAR_GLOBAL
    ncAxis : AXIS_REF;
END_VAR
```

---

## Step 5 — Link the AXIS_REF to the NC Axis

This step creates the data connection between `GVL.ncAxis` and **Axis 1** in the NC task.

### Method A — Drag-and-Drop (recommended for beginners)

1. In the **Solution Explorer**, expand **NC-Task 1 SAF** → **Axes** → **Axis 1**.
2. Click on **Axis 1** to open its properties.
3. Open the **PLC** tab inside the axis properties.
4. Click **Link to PLC…**
5. Browse to your GVL and select `ncAxis`.
6. Click **OK**.

A green link icon confirms the connection.

### Method B — I/O Tree Drag-and-Drop

1. Open the **I/O** view in the TwinCAT XAE left panel.
2. Locate the NC task inputs/outputs under **I/O → Devices → NC-Task 1 SAF → …**.
3. Drag `Axis 1 (PlcToNc)` onto `GVL.ncAxis` in the PLC I/O mapping view, and repeat for
   `Axis 1 (NcToPlc)`.

---

## Step 6 — Copy FB_AxisControl into Your Project

1. Copy `FB_AxisControl.TcPOU` from this sample into your project's `POUs` folder on disk.
2. In TwinCAT XAE, right-click your **POUs** folder in the Solution Explorer → **Add** →
   **Existing Item…** → select `FB_AxisControl.TcPOU`.
3. Also copy `E_AxisState.TcDUT` if you want to use the same state machine pattern.

Alternatively, declare `FB_AxisControl` directly by creating a new Function Block POU named
`FB_AxisControl` and pasting the variable and implementation sections from the source file.

---

## Step 7 — Instantiate and Call FB_AxisControl

In your program (e.g. `MAIN`), declare one instance of `FB_AxisControl` per axis:

```pascal
PROGRAM MAIN
VAR
    fbAxis   : FB_AxisControl;

    bEnable  : BOOL;
    bReset   : BOOL;
    bMove    : BOOL;
    fTarget  : LREAL := 100.0;
END_VAR
```

Call the function block **once per PLC cycle** and pass `GVL.ncAxis` as the `stAxis`
`VAR_IN_OUT` parameter:

```pascal
fbAxis(
    stAxis          := GVL.ncAxis,
    bEnable         := bEnable,
    bEnablePositive := TRUE,
    bEnableNegative := TRUE,
    bReset          := bReset,
    bMoveAbsolute   := bMove,
    fTargetPosition := fTarget,
    fVelocity       := 50.0,
    fAcceleration   := 500.0,
    fDeceleration   := 500.0,
    fJerk           := 0.0
);
```

> **Important:** Because `stAxis` is `VAR_IN_OUT`, the actual variable name (`GVL.ncAxis`)
> must appear inside the call parentheses, not assigned beforehand.

---

## Step 8 — Implement a Basic Sequence

Below is the minimum sequence to power on the axis and move to a position:

```pascal
// 1. Enable axis
bEnable := TRUE;

// 2. Wait until ready (servo on, no error)
IF fbAxis.bReady THEN
    // 3. Issue a single-cycle move pulse
    bMove   := TRUE;
    fTarget := 250.0;
END_IF

// Clear the move pulse next cycle
IF bMove AND fbAxis.bMoveBusy THEN
    bMove := FALSE;
END_IF

// 4. Detect completion
IF fbAxis.bMoveDone THEN
    // axis has reached fTarget
END_IF

// 5. Handle errors
IF fbAxis.bError THEN
    bEnable := FALSE;     // power down
    bReset  := TRUE;      // request reset
END_IF
IF fbAxis.bResetDone THEN
    bReset := FALSE;
END_IF
```

For a complete, production-quality example using a formal state machine, see `MAIN.TcPOU` in
this project. The state machine pattern is described in [readme.md](readme.md).

---

## Step 9 — Activate the Configuration

1. Press **Ctrl + Shift + F4** (or click **Build** → **Activate Configuration**) to compile
   and download the configuration to the controller.
2. Confirm the restart dialog if prompted.
3. Switch TwinCAT to **Run Mode** (green play button in the toolbar, or **TwinCAT** →
   **Restart TwinCAT (Run Mode)**).

The PLC program starts automatically if **Auto Start** is configured, or press **F5** in the
PLC project to start it manually.

---

## Step 10 — Test in the Watch Window

1. Open **Online** → **Create Watch Window** (or press **Ctrl + F7**).
2. Add the following variables to observe the axis:

| Variable | What to watch |
|---|---|
| `MAIN.fbAxis.bPowered` | Servo energised |
| `MAIN.fbAxis.bReady` | Ready for commands |
| `MAIN.fbAxis.fActPosition` | Current position |
| `MAIN.fbAxis.fActVelocity` | Current velocity |
| `MAIN.fbAxis.bMoveDone` | Last move complete |
| `MAIN.fbAxis.bError` | Any fault active |
| `MAIN.fbAxis.nErrorId` | Fault ADS error code |

3. Set `MAIN.bEnable := TRUE` to power on the axis.
4. Wait for `MAIN.fbAxis.bReady = TRUE`.
5. Set `MAIN.fTarget` to the desired position.
6. Pulse `MAIN.bMove := TRUE` for one scan, then release.
7. Observe `fActPosition` track toward the target and `bMoveDone` go `TRUE` on arrival.

---

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| `bPowered` never goes `TRUE` | `AXIS_REF` not linked or NC not in Run Mode | Verify Step 5; check NC task state |
| `bError = TRUE` immediately | Drive fault or axis configuration error | Check `nErrorId` against Beckhoff error code list; inspect NC axis state in the System Manager |
| `bMoveDone` never goes `TRUE` | Move aborted by a following error or limit switch | Check axis limits and following-error tolerance in the NC axis **Settings** tab |
| `nErrorId = 0x4490` | Velocity or acceleration exceeds axis parameter limits | Reduce `fVelocity` / `fAcceleration` inputs |
| Position units unexpected | Encoder resolution or unit scaling wrong | Revisit **Enc** tab, set **Encoder Increments** and **Scale Factor** correctly |

For ADS error code lookup, see the Beckhoff Information System:
`https://infosys.beckhoff.com` → search **ADS Return Codes**.

---

## Next Steps

- Study `MAIN.TcPOU` to see how the seven-state shuttle machine uses `FB_AxisControl`.
- Consult [FB_AxisControl.md](FB_AxisControl.md) for the full variable reference and extension
  guidance (homing, jogging, override, etc.).
- Return to [readme.md](../readme.md) for the full project overview.
- For multi-axis applications, declare one `FB_AxisControl` instance and one `AXIS_REF` per
  physical axis and follow this guide for each.
