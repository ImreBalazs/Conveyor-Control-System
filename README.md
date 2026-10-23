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
