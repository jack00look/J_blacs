# BLACS Workflow: From H5 Path to Experiment Execution

This document explains what happens when a `h5_filepath` is sent to the `ExperimentServer`.

## 1. Reception and Initial Processing
The `ExperimentServer` (defined in `blacs/__main__.py`) is a ZMQ server that listens for incoming filepaths.

- **`handler(h5_filepath)`**: Receives the message and calls `process()`.
- **`process(h5_filepath)`**: 
    - Converts the path to a local representation using `labscript_utils.shared_drive.path_to_local()`.
    - Hands off the path to the queue: `app.queue.process_request(h5_filepath)`.

## 2. Queue Admission: `QueueManager.process_request`
Located in `blacs/experiment_queue.py`, this method acts as a "gatekeeper".

1.  **Connection Table Validation**: It reads the connection table from the H5 file and compares it to the current BLACS configuration (the "active" connection table). If they don't match (e.g., a channel was renamed or added), the shot is rejected with a descriptive error.
2.  **Duplicate/Rerun Detection**: 
    - If the file has already been run (contains a `/data` group) or is already in the queue, BLACS creates a **fresh copy** (e.g., `filename_rep00001.h5`).
    - This "cleaned" copy contains only the necessary configuration (globals, connection table, device instructions) and removes any results from previous runs.
3.  **Model Insertion**: The path is added to `self._model` (a `QStandardItemModel`). This updates the **Queue** tab in the BLACS UI.

## 3. The Execution Loop: `QueueManager.manage`
Process orchestration happens in a dedicated background thread.

1.  **Getting the Next Shot**: It calls `get_next_file()`, which performs `self._model.takeRow(0)`.
2.  **Orchestrating `transition_to_buffered`**: 
    - BLACS extracts the list of devices used in the shot from `hdf5_file['devices']`.
    - It iterates through these devices, calling `tab.transition_to_buffered(h5file)` on each.
    - **Parallelism & Ordering**: Devices are transitioned in groups according to their `start_order` (defined in the connection table/labscript). This is critical for devices that depend on one another.
3.  **The Master Pseudoclock Trigger**: 
    - Once **all** devices have finished transitioning (confirmed via an internal queue), the master pseudoclock is triggered via `start_run()`.
4.  **Waiting for Science**: The thread waits for the master pseudoclock to signal `'done'`.
5.  **Post-Experiment Cleanup**:
    - **Save Data**: Front panel settings and a `/data` group are written to the H5 file.
    - **Transition to Manual**: All tabs return to manual/static mode.
    - **Analysis Submission**: The file is sent to `lyse`.

---

## Deep Dive: `transition_to_buffered`

This method is defined in `DeviceTab` (`blacs/device_base_class.py`). Its primary roles are:

1.  **Mode Switch**: It changes the tab mode to `MODE_TRANSITION_TO_BUFFERED`, which typically disables manual controls in the UI to prevent user interference.
2.  **Worker Handoff**: It calls `_transition_to_buffered` on the device's **Worker process**.
3.  **Hardware Programming**: The worker process reads the instructions from the H5 file and programs the physical hardware buffer.
4.  **Final Values**: The method returns "final values" (the state of the hardware at the very end of the shot) so the BLACS GUI can update its sliders/indicators accurately after the shot finishes.

## Deep Dive: The Master Pseudoclock

In labscript, the **Master Pseudoclock** is the device at the root of the clock hierarchy. In BLACS:

- **Identity**: It is identified by the `ConnectionTable` during startup.
- **Responsibility**: It is the only device whose `start_run()` method is called by the `QueueManager`.
- **Function**: While other devices "arm" themselves during `transition_to_buffered`, the master pseudoclock is the one that actually "fires" the start signal (hardware trigger or software command) that initiates the synced execution of all devices.
- **Wait Mechanism**: It is also responsible for monitoring the experiment's progress and notifying BLACS when the sequence has finished (usually by waiting for a hardware "done" signal or calculating the duration).

---

## Applied Example: `connectiontable.py`

In your [connectiontable.py](file:///home/jacklook/Documents/Projects/labscript-suite/J_blacs/connectiontable.py), we see a typical labscript definition:

```python
FPGA('fpga0')
DigitalBoard(name='ttl1', parent=fpga0, address=1)
DigitalOut(name='breakpoint_main_table', parent_device=ttl1, connection=0)
...
DDS9958(name='dds41', parent=fpga0, address=41, ...)
DDS(name='ddsa', parent_device=dds41, connection=0)
...
fpga0.start()
fpga0.stop(1)
```

### In the BLACS Workflow:

1.  **Master Pseudoclock**: `fpga0` is the root of the tree. BLACS will identify the tab corresponding to `fpga0` as the **Master Pseudoclock**.
2.  **Clock Hierarchy**: Since `ttl1` and `dds41` are children of `fpga0`, they depend on it for their timing signal.
3.  **Transitioning**: When a shot is run:
    - `dds41` and `ttl1` (and its children like `breakpoint_main_table`) will be programmed during their respective `transition_to_buffered` calls.
    - Because `fpga0` is the master, its `start_run()` will be the final trigger that allows `dds41` and `ttl1` to begin executing their buffered instructions.
4.  **Validation**: If you were to change `DigitalOut(name='breakpoint_main_table', ...)` to a different name in this file and re-compile, BLACS's `process_request` would catch the mismatch and reject the shot until the BLACS connection table is also updated.

---

## Why `NotImplementedError` in `DeviceTab`?

You noticed that in `blacs/device_base_class.py`, `start_run` raises a `NotImplementedError`. This is intentional:

- **`DeviceTab` is a Base Class**: It defines the *interface* that all BLACS tabs must follow, but it doesn't know how to talk to your specific hardware (NI Card, FPGA, Arduino, etc.).
- **Hardware-Specific Implementation**: Every pseudoclock must **override** this method to provide the actual trigger logic.

### Example: `FPGATab` Implementation
In your [GFIB/fpga.py](file:///home/jacklook/Documents/Projects/labscript-suite/J_blacs/GFIB/fpga.py), the `FPGATab` (which inherits from `DeviceTab`) provides the real implementation:

```python
class FPGATab(DeviceTab):
    ...
    @define_state(MODE_BUFFERED, True)
    def start_run(self, notify_queue):
        # 1. Start a monitor to check when the experiment is finished
        self.statemachine_timeout_add(100, self.status_monitor, notify_queue)
        
        # 2. Tell the worker process to trigger the hardware
        yield(self.queue_work(self.primary_worker, 'start_run'))
```

And in the corresponding `FPGAWorker`:
```python
class FPGAWorker(Worker):
    ...
    def start_run(self):
        # The actual software/hardware trigger
        self.fpga.execute() 
        time.sleep(0.2)
```

In summary: `DeviceTab` provides the "template", and files like `GFIB/fpga.py` provide the "action".

---

## Tracing the FPGA Execution Flow

Based on [GFIB/fpga.py](file:///home/jacklook/Documents/Projects/labscript-suite/J_blacs/GFIB/fpga.py), here is how the FPGA "works" from compilation to execution:

### 1. Compilation ([labscript-side](file:///home/jacklook/Documents/Projects/labscript-suite/J_blacs/GFIB/fpga.py#L205-L335))
When you compile your experiment script, the `FPGA.generate_code` method translates your commands into a specific **8-byte packet** format that the hardware understands:

| Byte Offset | Content | Description |
| :--- | :--- | :--- |
| **0** | Flags | `0x10` indicates a WAIT instruction. |
| **1-4** | Ticks | A 32-bit integer representing the wait time until the *next* instruction. |
| **5** | Address | Which child board (Digital, Analog, DDS) should receive the data. |
| **6-7** | Data | The 16-bit payload (e.g., bitmask for TTLs or voltage for AOs). |

### 2. Transition (Programming)
In the `FPGAWorker.transition_to_buffered` method:
1.  **Read H5**: The worker extracts the instruction table from the HDF5 file.
2.  **ZeroRPC Program**: It sends these bytes to the physical FPGA server (at `192.168.1.151:4753`) using `self.fpga.program(program)`.
3.  **Handoff**: The FPGA server stores this program in a local buffer (likely a FIFO in memory).

---

## The Bottom of the Stack: Direct Hardware Interaction

The last mile of execution happens in the [fpga.py library](file:///home/jacklook/Documents/Projects/labscript-suite/J_blacs/fpga.py), which uses `pylibftdi` to communicate over USB.

### 1. The Low-Level Protocol
The hardware understands a set of 1-byte control opcodes:
- `0x02` (LOAD): Prepare to receive 10 bytes of instruction data.
- `0x08` (STATUS): Return 6 bytes describing the internal state.
- `0x0b` (RUN): Begin processing the loaded buffer.
- `0x0a` (STOP): Halt execution immediately.

### 2. Instruction Wrapping
While Labscript generates **8-byte packets**, the library wraps them into **11-byte hardware commands** for sequential loading:

```text
[ LOAD (1 byte) ] + [ Index (2 bytes) ] + [ Labscript Packet (8 bytes) ]
```

### 3. The "Valid Program" Check
Before running, the library calls `LOAD_DONE (0x04)`. The FPGA then performs internal checks (e.g., verifying that the FIFO won't underflow at the requested clock speed). If the FPGA returns a `valid program` bit in its status response, the experiment is cleared to start.

### 4. FTDI Communication
The commands are pushed to the hardware using `self.write(program)`. Since the library inherits from `pylibftdi.Device`, this translates directly to USB bulk transfers to the FTDI chip on the FPGA board.

---

## Full Summary: The Journey of an Instruction

1.  **Compilation**: You write `ttl1.high(t=1)`. Labscript creates an 8-byte packet saying `[0x00, Ticks(1s), Address(ttl1), 0x01]`.
2.  **Queue**: You submit the H5. BLACS ensures your hardware matches the script.
3.  **Transition**: `FPGAWorker` reads the packets and sends them via **ZeroRPC** to the FPGA Server.
4.  **Programming**: `FPGAsServer` uses the `fpga` library to prefix each packet with `LOAD + Index` and blasts them over **USB** to the FTDI chip.
5.  **Execution**: You click "Run". BLACS calls `execute()`, the FPGA server sends `0x0b` over USB, and the hardware begins toggling pins in real-time.
