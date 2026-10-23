[🇭🇺 Magyar nyelvű leírásért kattints ide](./README_HU.md)

## 1. Project Overview

In this project, I implemented the complete automation of an industrial conveyor and sorting system. The system consists of an electrically driven conveyor belt and a pneumatic pusher cylinder, with processes controlled by optical sensors.
The purpose of the system is the automatic size-based sorting and counting of boxes arriving on the conveyor:
- **Size-based sorting:** The system distinguishes between low and tall boxes. 
  * When a **tall box** arrives, the height sensor triggers a signal, and the pneumatic pusher arm diverts it off the main conveyor line into a collection area.
  * **Low boxes** are allowed to pass straight through along the main line.
- **Part counting & safety shutdown:** An optical counter sensor registers the boxes passing through the process. Once the counter reaches **10 units**, the system automatically stops the cycle.
- **Direction reversal:** The conveyor motor offers flexible control – it can be started in both **forward** and **reverse** directions.

During development, the electrical schematics (EPLAN Electric P8), PLC control program (CODESYS V3.5), embedded HMI interface (CODESYS WebVisu), and digital twin simulation (Factory I/O) were fully designed and implemented, interconnected via an OPC UA communication layer.

### Control Architecture (Three-Mode Concept)

System operation is based on three distinct operating modes, governed by the position of the mode selector switch and logical conditions defined in the PLC program:

#### 1. HAND (Manual Mode)
- **Purpose:** Manual testing, commissioning, and maintenance of individual actuators.
- **Direction reversal and interlocking:** Moving the conveyor forward/reverse is executed via relay latching circuits. Hardware and software electrical interlocking (normally closed contacts of `-K1` and `-K2` contactors, alongside CODESYS `RS` bistable blocks) guarantees that both direction outputs can never activate simultaneously.

#### 2. AUTO (Automatic Mode)
- **Purpose:** Cyclic, fully automated process control requiring no human intervention.
- **Operation:** The main PLC program conditionally invokes the `PRG_Automata` POU block. Process control relies on signals from optical sensors:
  * **Presence and height detection:** Determines the size category of incoming boxes on the conveyor.
  * **Active sorting:** Actuates the pneumatic pusher cylinder (`-Y1`) with precise timing for selected boxes.
  * **Counting:** Registers the number of passed and sorted items using incremental logic.

#### 3. OFF / Safety (Off and Safety State)
- **Forced State Reset:** In the OFF position of the selector switch, the PLC cycle forcibly clears all active outputs and resets internal latches, preventing unwanted stuck states during the next startup.
- **Safety interlocking:** When the Emergency Stop (`E-Stop`) circuit is opened, the system immediately de-energizes the actuators and halts all drives.

## 2. Electrical Schematics (EPLAN Electric P8)

The complete electrical schematics for the system were designed in **EPLAN Electric P8**. A key highlight is Page 4, which illustrates the hardware control and protection logic for Manual (HAND) mode, as well as mode selection:

- **Relay Latching Circuits:** Pressing the pushbuttons energizes the corresponding direction contactor (`-K1` for forward, `-K2` for reverse), which latches itself via its auxiliary contact. This ensures that motor power is maintained even after the button is released.
- **Electrical Cross-Interlocking:** To prevent phase-to-phase short circuits, normally closed (NC) contacts of each contactor are cross-connected into the coil control circuit of the opposing contactor. While `-K1` is energized, its NC contact interrupts the circuit of the `-K2` coil (and vice versa), making simultaneous activation of both rotation directions physically impossible.
- **Series Stop Control:** Pressing the `-S4` STOP button, wired in series at the head of the control circuit, breaks power, immediately unlatching the circuit and halting the motor.
- **Switching to Automatic (AUTO) Mode:** Using the selector switch, the system can be switched to automatic mode at any time. In this state, direct relay control is superseded by the PLC, which governs all operations via the CODESYS control program based on optical sensor signals.

<img width="1239" height="779" alt="{646A23E2-210F-464D-AC1D-F51A5C04ACA7}" src="https://github.com/user-attachments/assets/70e57e85-13bf-4055-b3ba-1fc0c533e337" />
<p> <a href="./conveyorControlSystemElectricalDiagram.pdf"><b>View Complete Electrical Schematics (PDF)</b></a></p>

---
### 3. PLC Control Program (CODESYS V3.5)

The entire control software was implemented in the **CODESYS V3.5** environment using Ladder Diagram (**LD**). The program architecture is built upon **two separate POUs (Program Organization Units)**:

- **`S2LD` (Main Program):** The main execution cycle responsible for mode selection (HAND / AUTO / OFF), software interlocks, manual motion control, and conditional invocation of the automatic subroutine.
- **`PRG_Automata` (Automatic Subroutine):** Contains the automated process logic, including optical sensor signal conditioning, pneumatic pusher cylinder timing, and part counting.

Below, the control logic is detailed network by network.

#### 3.1. `S2LD` Main Program (Mode Selection and Safety Logic)

The Ladder Diagram of the main program (Networks 1–5) manages global system states, manual motion control, and mode switching.

<img width="1564" height="640" alt="{7DCDBEDC-3AAE-4514-8973-1807DF6A95EB}" src="https://github.com/user-attachments/assets/d36fd252-ef03-4974-8409-d704580a8e9c" />
<img width="1565" height="334" alt="{B010B7BD-DE19-4976-A164-4478670E05CF}" src="https://github.com/user-attachments/assets/6644dbfa-78e1-41fa-872e-1406ac24838a" />

*`S2LD` Ladder Diagram networks (Networks 1–5)*

- **Network 1 – Automatic Call:** Enables and calls the `PRG_Automata` subroutine when the `autoMode` signal is active.
- **Network 2–3 – Manual Control & Cross-Interlocking:** Manual forward/reverse actuation is controlled via `RS` bistable latches. Software cross-interlocking against short circuits is provided by normally closed (NC) contacts of opposing directions. Pressing the STOP (`stopPB`) or Emergency Stop (`emergencyStop`) pushbuttons instantly resets (`RESET1`) the latches.
- **Network 4 – Status Indication:** As long as the conveyor is moving neither forward nor reverse, the `stopLight` indicator lamp remains active.
- **Network 5 – OFF / Safety Reset:** In the OFF state (when neither `autoMode` nor `handMode` is active), the PLC forcibly resets (`Reset coils`) all drive and actuator outputs (`forwardDirection`, `reverseDirection`, `pusher`), preventing lingering active states.

#### 3.2. `PRG_Automata` (Automatic and Sorting Logic)

The `PRG_Automata` subroutine is responsible for supervising automatic operation, driving indicator lamps, executing pneumatic sorting of tall boxes, and counting low boxes.

**Variable Declaration (`PRG_Automata`):**
```pascal
PROGRAM PRG_Automata
VAR
    // Operator controls and pushbuttons
    forwardPB               : BOOL;             // Forward start pushbutton
    stopPB                  : BOOL;             // Stop pushbutton
    reversePB               : BOOL;             // Reverse start pushbutton
    emergencyStop           : BOOL;             // Emergency Stop (E-Stop)
    counterReset            : BOOL;             // Counter reset pushbutton

    // Sensors and limit switches
    presenceSensor          : BOOL;             // Entry presence sensor
    heightSensor            : BOOL;             // Height sensor (tall box detection)
    counterSensor           : BOOL;             // Exit counter sensor
    pusherFrontLimit        : BOOL;             // Pusher cylinder extended limit switch

    // Internal latches, timers, and counters
    forwardBlock            : RS;               // Forward direction RS latch
    reverseBlock            : RS;               // Reverse direction RS latch
    rsTimer                 : RS;               // Ejection timing trigger latch
    rsPusher                : RS;               // Pusher cylinder RS latch
    pusherTimer             : TON;              // Ejection delay timer (400 ms)
    smallBoxCounter         : Standard.CTU;     // Low box counter
    counterCurrentValue     : WORD;             // Counter current value
    counterOut              : BOOL;             // Counter preset reached flag
    timerSetter             : RS;               // Timer auxiliary latch
    stopSetter              : RS;               // Stop auxiliary latch

    // Indicators and status lamps
    counterOutLamp          : BOOL;             // Counter preset reached lamp
    isPowerOKLamp           : BOOL;             // Power OK status lamp
    safetyCircuitFailedLamp : BOOL;             // Safety circuit fault lamp
END_VAR

VAR_INPUT
    // Input parameters / Controlled output states
    forwardDirection        : BOOL;             // Forward drive state
    forwardLight            : BOOL;             // Forward indicator lamp state
    reverseDirection        : BOOL;             // Reverse drive state
    reverseLight            : BOOL;             // Reverse indicator lamp state
    pusher                  : BOOL;             // Pneumatic pusher cylinder output
    stopLight               : BOOL;             // Stopped status indicator lamp
END_VAR

<img width="1356" height="607" alt="{13A9DADC-BD7A-4D6C-947B-14AEB7D7572E}" src="https://github.com/user-attachments/assets/9a1e6a5f-52fc-4adf-bf65-a27366d5da54" />
<img width="1345" height="743" alt="{458321D3-1F0B-4443-96A6-392F97A3B0C1}" src="https://github.com/user-attachments/assets/6571fbd5-442f-4f76-9743-2f1dea18d587" />
<img width="1342" height="218" alt="{BB383D67-F807-4B72-9AD6-8515D6A87BB8}" src="https://github.com/user-attachments/assets/89372c71-ab19-4d08-abc1-2cd1d25da61e" />

---
##### Network 1: Power Supply Status (`isPowerOKLamp`)
- **Operation:** Upon presence of control power, the PLC directly energizes the `isPowerOKLamp` output, indicating active power status on the panel.

---

##### Network 2: Forward Motion and Automatic Stop (`forwardBlock`)
- **SET (Start):** The forward `RS` latch is set when the `forwardPB` pushbutton is pressed and reverse motion is inactive (`NOT reverseDirection`).
- **RESET1 (Stop):** Forward conveyor motion stops immediately if:
  * The STOP button is pressed (`stopPB`), **OR**
  * Emergency Stop is triggered (`emergencyStop`), **OR**
  * The part counter reaches its preset limit (`counterOut`).
- **Output:** Energizes the drive (`forwardDirection`) and the control panel indicator lamp (`forwardLight`).

---

##### Network 3: Stopped Status Indication (`stopLight`)
- **Operation:** As long as neither forward (`NOT forwardLight`) nor reverse (`NOT reverseLight`) direction is active, the PLC energizes the `stopLight` indicator lamp.

---

##### Network 4: Safety Circuit Fault Indication (`safetyCircuitFailedLamp`)
- **Operation:** Pressing the Emergency Stop pushbutton or opening the safety circuit (`NOT emergencyStop`) triggers the `safetyCircuitFailedLamp` fault indicator lamp.

---

##### Network 5: Reverse Motion and Automatic Stop (`reverseBlock`)
- **SET (Start):** The reverse `RS` latch is set via the `reversePB` pushbutton provided forward motion is inactive (`NOT forwardDirection`).
- **RESET1 (Stop):** Identical to Network 2 – pressing STOP (`stopPB`), Emergency Stop (`emergencyStop`), or reaching the counter limit (`counterOut`) halts the drive instantly.
- **Output:** Energizes reverse drive (`reverseDirection`) and the corresponding indicator lamp (`reverseLight`).

---

##### Network 6: Pneumatic Pusher Cylinder Control and Timing (`pusher`)
- **Detection:** When the conveyor moves forward (`forwardDirection`), and the entry presence sensor (`presenceSensor`) combined with the height sensor (`heightSensor`) detects a **tall box**, the `rsTimer` latch is activated.
- **Delay (TON):** The `pusherTimer` initiates a **400 ms** (`T#400MS`) delay, allowing the box to align precisely in front of the pusher arm.
- **Ejection:** After the 400 ms elapsed time, the `rsPusher` latch energizes the pneumatic cylinder (`pusher`), diverting the box off the main conveyor line.
- **Home Position:** As soon as the cylinder reaches its extended limit switch (`pusherFrontLimit`) or `emergencyStop` is triggered, both latches are reset (`RESET1`), retracting the cylinder.

---

##### Network 7: Low Box Counting (`smallBoxCounter`)
- **Counting (CU):** The `CTU` counter registers only boxes detected by the exit sensor (`counterSensor`) while the height sensor remains inactive (`NOT heightSensor`) during forward movement (`NOT reverseDirection`).
- **Preset Value (PV):** The preset limit is **10 units** (`PV := 10`).
- **Cycle Complete (Q):** Upon passage of the 10th low box, the `counterOut` output transitions to a high state:
  * Turns on the target-reached indicator lamp (`counterOutLamp`).
  * Automatically halts conveyor motion in Networks 2 and 5.
- **Reset (RESET):** The current counter value (`counterCurrentValue`) can be reset at any time using the `counterReset` pushbutton.
