# Water Pump Control System - Firmware Logic V1.2

## 1. Overview

This module controls a **drainage water pump** based on water level sensor inputs (**S1** and **S2**). It uses a **Finite State Machine (FSM)** architecture to ensure the pump operates reliably, handles potential clogs ("Special Mode"), and performs self-maintenance ("Autorun").

### Key Hardware Abstractions:
- **S1 (Lower Sensor)**: Detects if water is present at a low level.
- **S2 (Higher Sensor)**: Detects if water has reached a critical high level (triggering the pump).
- **Pump Output**: Controlled via GPIO to turn the motor **ON** or **OFF**.


## 2. State Machine Description

The system operates in one of the following states (`g_wpState`):

### A. Standard Idle (`WP_SM_STD_IDLE`)
**Behavior**: The system waits here when the tank is empty.

**Transitions**:
- If **S2 == 1** (High Water): Transition to Standard Pump.
- If **24 Hours** pass with no activity: Transition to Autorun.

### B. Standard Pump (`WP_SM_STD_PUMP`)
**Behavior**: The pump runs to drain the water.

**Stop Conditions**:
- **Timeout**: The pump runs for **45 seconds** (Safety limit).
- **S1 Logic**: If S1 goes **HIGH** (water detected) and then drops **LOW** (water gone) for **5 continuous seconds**.

**Post-Pump Decision** (`CheckPostPump`):
- If **S1=0, S2=0** (Empty): Return to Standard Idle (Success).
- If **S1=1** (Partial Drain): Move to Special Idle (Possible clog/slow drain).
- If **S2=1** (Still Full): Critical Error (Pump failed to move water).

### C. Special Idle ("Retry Mode" - `WP_SM_SPEC_IDLE`)
**Purpose**: If the tank didn't drain completely (S1 is still high), we enter this mode to wait and retry.

**Safety Logic**: Every time we enter this state, a counter (`g_S2StuckCounter`) increments.
- If **Counter < Max**: Retry pumping (Move to Special Pump).
- If **Counter >= Max**: Error **0x202A** (Stuck/Clogged).

### D. Special Pump (`WP_SM_SPEC_PUMP`)
**Behavior**: Similar to Standard Pump, but with a longer timeout (**60 seconds**) to force water out.

**Transition**: After stopping, it always returns to Standard Idle to re-evaluate the sensors from scratch.

### E. Autorun (`WP_SM_AUTORUN`)
**Purpose**: Runs the pump for **30 seconds** every **24 hours** to prevent seizing/sediment buildup.

**Logic**: Once finished, it checks the sensors using the standard `CheckPostPump` logic to ensure no water was accidentally left behind.


## 3. Potential Error Scenarios

*(To be filled in by the developer based on testing data)*

### Error Case 1: [Placeholder Name]
- **Description**: [TBD]
- **System Reaction**: [TBD]

### Error Case 2: [Placeholder Name]
- **Description**: [TBD]
- **System Reaction**: [TBD]

### Error Case 3: [Placeholder Name]
- **Description**: [TBD]
- **System Reaction**: [TBD]


## 4. Logic Flow Diagram

The following diagram illustrates the state transitions and decision logic.

```mermaid
stateDiagram-v2
    [*] --> TEST_RUN: Test Mode
    [*] --> STD_IDLE

    %% --- Standard Cycle ---
    state "Standard Idle" as STD_IDLE
    state "Standard PUMP (45s)" as STD_PUMP

    state "TEST RUN (10s)" as TEST_RUN

    STD_IDLE --> STD_PUMP: S2 == 1 <br/>(Water Detected)
    
    STD_PUMP --> Check_S2_Std: Timeout
    STD_PUMP --> Check_S2_Std: S1 == 0 -> run 5s pump 
    
    state Check_S2_Std <<choice>>
    Check_S2_Std --> STD_IDLE: S2 == 0 && S1 == 0 <br/>(Reset counter!)
    Check_S2_Std --> SPEC_IDLE: S1 == 1 <br/>(Water Level in between S1 and S2)
    Check_S2_Std --> ERROR: "S2 == 1 <br/>(Not dumping at all, immediately Error)"

    %% --- Special / Retry Cycle ---
    state "Special Idle<br/>(Counter++)" as SPEC_IDLE
    state "Special PUMP (60s)" as SPEC_PUMP
    state "ERROR (0x202A)" as ERROR

    SPEC_IDLE --> ERROR: Counter >= 10
    SPEC_IDLE --> SPEC_PUMP: Counter < 10 (Retry)
    note right of SPEC_IDLE
      Increments Counter
      on entry
    end note

    SPEC_PUMP --> STD_IDLE: Timeout
    SPEC_PUMP --> STD_IDLE: S1 == 0 -> run 5s pump 

    %% --- AutoRun (24h) ---
    state "Autorun (30s)" as AUTORUN
    STD_IDLE --> AUTORUN: 24h No Activity
    AUTORUN --> Check_S2_Std: Go check S2

```
