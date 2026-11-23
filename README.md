# APB UVM Verification Environment

A class-based **SystemVerilog UVM** verification environment built to validate an **AMBA APB (Advanced Peripheral Bus)** interface, developed and simulated in **QuestaSim**.

This repository showcases a modular, reusable UVM testbench featuring constrained-random stimulus generation, protocol-aware bus monitoring, a shadow-memory reference scoreboard, functional and cross coverage, SystemVerilog Assertions (SVA), wait-state flow control, and slave error (`PSLVERR`) verification[cite: 3, 4, 5, 6, 7, 9].

---

## 1. Project Overview

The goal of this project is to verify APB read and write transfers using a structured UVM verification architecture[cite: 4, 6]. Rather than connecting to a synthesizable RTL peripheral, the environment interacts with an **APB Slave BFM (Bus Functional Model)** on the device side.

### Key Verification Capabilities
* **Bus Transfers:** Directed and randomized APB `READ` and `WRITE` transactions.
* **Protocol Phasing:** Cycle-accurate `IDLE`, `SETUP`, and `ACCESS` state transitions.
* **Wait-State Handling:** Dynamic slave wait-state insertion and handshaking via `PREADY`[cite: 4, 5, 9].
* **Error Signaling:** Out-of-bounds address detection and `PSLVERR` response checking[cite: 7, 9].
* **Address Validation:** Legal word-aligned register accesses vs. unmapped/invalid addresses[cite: 6, 7, 9].
* **Data Integrity:** Automated read-after-write verification against a golden reference memory.
* **Sign-Off Metrics:** Constrained-random testing, functional coverage closure, and concurrent APB protocol assertions[cite: 3, 5, 8].

---

## 2. Verification Architecture

```text
                         +----------------------+
                         |    apb_base_test     |
                         +----------+-----------+
                                    |
                                    v
                         +----------------------+
                         |       apb_env        |
                         +----------+-----------+
                                    |
                 +------------------+------------------+
                 |                                     |
                 v                                     v
        +-------------------+                 +-------------------+
        |     apb_agent     |                 |  apb_scoreboard   |
        +---------+---------+                 +---------+---------+
                  |                                     ^
          +-------+-------+                             |
          |       |       |                             |
          v       v       v                             |
    apb_sqr  apb_drv   apb_mon -------------------------+
          |       |       |                             |
          |       |       +--------------------+        |
          |       |                            |        |
          |       v                            v        |
          |   +--------+                 +--------------+
          |   | apb_if |                 | apb_coverage |
          |   +---+----+                 +--------------+
          |       |
          |       v
          | +------------+
          +-| apb_slave  |
            |    _bfm    |
            +------------+
Transaction Data Flow
Sequence (apb_smoke_seq / apb_random_seq / apb_error_seq)
   |
   v
Sequencer (apb_sequencer)
   |
   v
Master Driver (apb_master_drv)
   |
   v
APB Interface (apb_if + SVA)
   |
   v
APB Slave BFM (apb_slave_bfm)
   |
   v
Passive Monitor (apb_monitor)
   |
   +--------> Scoreboard (apb_scoreboard)
   |
   +--------> Functional Coverage (apb_coverage)

3. UVM Testbench ComponentsAPB Sequence Item (apb_rw.sv)Defines the transaction-level representation (apb_rw) of an APB operation:   Stimulus Fields: Address (addr), write data (data), and operation type (apb_cmd — READ or WRITE).   Response Fields: Captured read data (rdata) and slave error flag (slverr).   Constraints: Restricts randomized transactions to 4-byte aligned addresses within the legal register map (0x0000_0000–0x0000_003C) and applies a weighted read/write distribution (60% WRITE / 40% READ).   APB Sequencer (apb_driver_seq_mon.sv)Implemented as apb_sequencer, routing directed and constrained-random apb_rw items to the driver via TLM ports.   APB Master Driver (apb_driver_seq_mon.sv)Implemented as apb_master_drv, translating apb_rw objects into pin-level activity through vif.master_cb:   Initializes bus signals in the IDLE state (PSEL = 0, PENABLE = 0).   Drives address, direction, and write data during the SETUP phase (PSEL = 1, PENABLE = 0).   Asserts PENABLE = 1 to enter the ACCESS phase.   Stalls until PREADY is sampled high.   Captures PRDATA and PSLVERR before returning the bus to IDLE.   APB Passive Monitor (apb_driver_seq_mon.sv)Implemented as apb_monitor, observing vif.monitor_cb and reconstructing completed transfers. A transfer is sampled strictly when:   

PSEL = 1
PENABLE = 1
PREADY = 1
This ensures that transactions are collected after the APB transfer actually completes, including transfers containing wait states.

The monitor sends transactions through a UVM analysis port to:

Scoreboard
Functional coverage
APB Agent
Implemented in apb_agent_env_config.sv.

The agent contains:

Sequencer
Driver
Monitor
The virtual interface is passed to the driver and monitor using uvm_config_db.

UVM Environment
Implemented in apb_agent_env_config.sv.

The environment instantiates:

APB agent
Scoreboard
Coverage collector
The monitor's analysis port is connected to both the scoreboard and coverage subscriber.

4. APB Slave BFM
File: apb_slave_bfm.sv

The project uses an APB Slave BFM instead of a synthesizable APB slave RTL design.

The BFM provides:

Register-like memory behavior
Read/write storage
Variable wait-state insertion
PREADY generation
PSLVERR generation
Invalid-address detection
Valid address range
The verification model treats the following addresses as valid:

0x00000000 - 0x0000003C
with 4-byte alignment.

Therefore, the valid register locations are:

0x00, 0x04, 0x08, 0x0C, ... , 0x38, 0x3C
Invalid addresses generate an APB error response through:

PSLVERR = 1
5. APB Wait-State Verification
The slave BFM can introduce up to two wait states.

During a wait state:

PSEL     = 1
PENABLE  = 1
PREADY   = 0
The driver waits until PREADY becomes high before completing the transaction.

The monitor also waits for the completed transfer before publishing the transaction.

This verifies that the environment correctly handles APB transfers requiring multiple ACCESS cycles.

6. Scoreboard and Reference Model
File: apb_scoreboard.sv

The scoreboard contains an independent reference memory model.

For write transactions:

APB Write
   |
   v
Reference Memory Update
For read transactions:

APB Read
   |
   v
Expected Data from Reference Model
   |
   v
Compare with Actual PRDATA
The scoreboard also checks PSLVERR.

For valid accesses:

PSLVERR = 0
For invalid accesses:

PSLVERR = 1
This provides transaction-level checking independent of the slave BFM's internal implementation.

7. Functional Coverage
File: apb_coverage.sv

The functional coverage model tracks important verification scenarios.

Command coverage
READ
WRITE
Address coverage
The defined valid register addresses are covered individually, along with an invalid-address category.

Error coverage
NO_ERROR
ERROR
Cross coverage
The project also crosses:

Command × Error
The regression achieved:

100% of the defined functional coverage model
This represents coverage of the bins defined in this project's coverage model; it does not mean that every possible APB behavior has been exhaustively verified.

8. Protocol Assertions
File: apb_if.sv

The APB interface contains assertions for basic protocol behavior.

PENABLE requires PSEL
PENABLE -> PSEL
This checks that PENABLE is not asserted without an active APB select.

SETUP to ACCESS transition
A SETUP cycle:

PSEL = 1
PENABLE = 0
must be followed by an ACCESS cycle:

PSEL = 1
PENABLE = 1
Control stability during wait states
When an APB transfer is waiting for PREADY, important control information such as address, read/write direction, and write data must remain stable.

9. Verification Sequences
File: apb_sequences.sv

The project includes multiple verification scenarios.

Smoke Sequence
apb_smoke_seq

Performs directed write/readback operations using several valid addresses.

Example:

WRITE 0x00 -> AAAA1111
READ  0x00 -> AAAA1111

WRITE 0x04 -> BBBB2222
READ  0x04 -> BBBB2222
Constrained-Random Sequence
apb_random_seq

Generates 50 randomized APB transactions.

The sequence exercises:

Random addresses within the legal range
Aligned addresses
Random read/write operations
Different data values
Error Sequence
apb_error_seq

Explicitly accesses invalid addresses to verify error handling.

Example:

WRITE 0x100
READ  0x104
Expected behavior:

PSLVERR = 1
10. Testbench Files
File	Description
testbench.sv	Top-level simulation module
apb_if.sv	APB interface, clocking blocks and assertions
apb_rw.sv	APB UVM sequence item
apb_driver_seq_mon.sv	Driver, sequencer and monitor
apb_agent_env_config.sv	Configuration, agent and environment
apb_scoreboard.sv	Reference model and transaction checking
apb_coverage.sv	Functional coverage
apb_sequences.sv	Directed and constrained-random sequences
apb_test.sv	UVM test
apb_slave_bfm.sv	APB slave bus functional model
run.do	QuestaSim compilation and simulation script
11. Simulation Flow
The complete testbench can be compiled and simulated using the provided QuestaSim script.

Run from QuestaSim:

do run.do
The script:

Cleans the previous work library
Creates a new work library
Compiles the SystemVerilog testbench
Starts the UVM test
Adds APB signals to the waveform
Runs the simulation to completion
12. Regression Results
The final known-good regression exercised:

Smoke transactions          : 10
Constrained-random          : 50
Error transactions          : Included
--------------------------------
Total monitored             : 60
Scoreboard PASS             : 60
Scoreboard FAIL             : 0
Functional coverage         : 100%
UVM errors                  : 0
UVM fatals                  : 0
Wait states were exercised during the simulation, and invalid accesses generated the expected PSLVERR response.

The single UVM warning observed in the regression corresponds to the intentionally generated APB error response for an invalid transaction and is not a scoreboard failure.

13. Tools and Technologies
SystemVerilog
UVM
QuestaSim / ModelSim
Constrained-Random Verification
Functional Coverage
SystemVerilog Assertions
Reference Model / Scoreboard
APB Protocol
14. Verification Concepts Demonstrated
This project demonstrates practical knowledge of:

UVM testbench architecture
Sequence items
Sequences
Sequencers
Drivers
Monitors
Agents
Environments
Virtual interfaces
uvm_config_db
UVM factory mechanism
Analysis ports
Scoreboards
Reference models
Constrained randomization
Functional coverage
Cross coverage
SystemVerilog assertions
APB protocol phases
Wait-state handling
Error-response verification
QuestaSim simulation and debugging
15. Project Limitation
This implementation uses an APB Slave BFM as the device-side model rather than a synthesizable APB slave RTL DUT.

Therefore, the project primarily demonstrates the construction and operation of an APB UVM verification environment.

For a production-level DUT verification project, the next step would be to replace the BFM with an actual APB slave RTL design and verify its implementation against the APB protocol and functional requirements.

16. Future Enhancements
Potential extensions include:

Replace the slave BFM with an actual APB RTL DUT
Add reset verification
Add dedicated smoke, random, error and wait-state tests
Add back-to-back APB transfer testing
Add boundary-address testing
Add additional protocol assertions
Add coverage-driven test refinement
Add multiple APB slave/register regions
Add UVM configuration objects for BFM parameters
Add regression automation
Integrate the project with CI where simulator licensing permits
17. Repository Purpose
This repository is intended as a Design Verification portfolio project demonstrating the development of a structured SystemVerilog/UVM environment for a standard on-chip bus protocol.

The focus is on showing how stimulus generation, protocol driving, monitoring, checking, coverage, and assertions work together to verify a hardware interface.
