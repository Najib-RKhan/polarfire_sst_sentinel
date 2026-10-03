# System idea and the software boundary

This is the working map for the embedded and Linux side. It follows engineering spec SS-001 rev 0.2 (2 Oct 2026). Numbers below are **targets or proposals**. None of them have been measured on hardware yet.

Team split in the spec: **PE** owns the power stage and shares the FPGA work. **ES** owns embedded software and shares the FPGA work. If you are taking the software seat, sections 12–16 of the spec (recorder, cores, registers, event files, dashboard) are your contract. Several FPGA tasks are still assigned to ES as well; those are called out at the end so they do not disappear.

## What the project is trying to prove

AI racks are moving toward 800 V DC. The switches in those converters are SiC MOSFETs. A real short circuit can destroy one in a few microseconds. A healthy GPU load step can look similar for a moment: current jumps, then it is supposed to keep running. A threshold on current alone cannot tell those apart. Trip too early and you drop a rack of compute. Trip too late and you lose the switch. Trip with no record and nobody knows whether it is safe to turn back on.

SST Sentinel does **not** build an 800 V converter or a solid-state transformer. It builds a **48 V laboratory stand-in**: one SiC half-bridge, two current measurements in different places, and a record of what the protection logic saw.

The experiment is: at the same fault-catch rate as a well-tuned fixed threshold, can extra context (how fast the current rose, which switch was on, whether the two current paths agree) let more real load steps ride through, while every unclear case still shuts down.

Three outcomes are reported separately. They are not the same thing:

| Decision | What it means | What the hardware does |
| --- | --- | --- |
| Benign / ride-through | Evidence fits a tested safe envelope | Switching continues, for a limited time, under the emergency ceiling |
| Fault | Evidence supports a real abnormal path | Shut down |
| Unresolved | Stale, missing, contradictory, or the deadline expired | Shut down anyway |

An unresolved shutdown on a healthy load step counts as a nuisance trip. It is a safe choice and a wrong diagnosis. The dashboard has to show that difference. Safe ambiguity is not presented as "we found the fault."

## The two paths, and where software enters

Everything fast happens in analog hardware and in the FPGA fabric (the programmable-logic half of the chip). The processors never sit on the path that turns the gates off.

```text
48 V half-bridge
    │
    ▼
Analog front end
    │
    ├── 6 comparators (3 per current path) ──► FPGA fabric
    │         warning levels from a DAC              │
    │         emergency level from a fixed reference │  decision in < 1 µs target
    │                                                │  emergency trip < 200 ns target
    │                                                ▼
    │                                         gate inhibit / PWM
    │                                         (external latch still wins if the FPGA is dead)
    │
    └── 4-channel ADC at 1 MSPS ──────────► fabric sample ring (on-chip RAM)
              I_dc, I_load, V_bus, V_sw              │
                                                     │ one frozen event, DMA
                                                     ▼
                                              reserved DDR slot
                                                     │
                              bare-metal core ───────┤  only software allowed to
                              (configure, arm, IRQ)  │  change safety state
                                                     ▼
                                              Linux on the other cores
                                              archive to eMMC, Ethernet dashboard
```

**Transition 1 — pins, not software.** Comparator outputs, ADC serial data, PWM, and the inhibit line are FPGA pins on the Icicle Kit's 40-pin header (J26). Linux never samples these. A dead processor, a stalled network stack, or a full disk must leave this path alone. That is requirement SYS-005.

**Transition 2 — fabric to bare-metal.** This is the first software boundary.

- Control and status registers on an APB bus through FIC3. Bare-metal reads and writes them.
- An interrupt when an event is captured. Servicing that interrupt moves records. It does not make the trip decision.
- DMA (FIC2) copies a finished waveform from on-chip RAM into a reserved region of the 2 GB LPDDR4. The CPU does not copy those bytes one by one.

**Transition 3 — bare-metal to Linux.** Linux asks for configuration changes over a versioned mailbox (Microchip IHC, inter-hart communication). Bare-metal checks the request and is the only core that writes safety registers and the DAC. Linux reads a separate read-only status view and consumes DDR slots whose descriptor says the record is complete. Ethernet and the web dashboard sit on this side of the line.

There is also a slow I2C bus to the DAC that sets warning thresholds. That bus is configuration. It is never part of a microsecond decision. Bare-metal owns it.

## Who is allowed to do what

Priority is fixed. A lower layer cannot override a higher one.

| Priority | Who | Allowed to |
| --- | --- | --- |
| 1 | External trip, device fault, rail failure, external watchdog | Force gates off even if every processor is dead |
| 2 | FPGA sticky emergency, or an unsafe / expired decision | Revoke permission to switch and latch it |
| 3 | FPGA contextual decision, inside tested limits | Keep PWM running, or request a shutdown profile that has been validated |
| 4 | Bare-metal on U54_1 | Disarm, configure, request arm. Cannot override a live fault |
| 5 | Linux, dashboard, test PC | Ask and observe. No safety-register ownership |

"Arm" means switching is allowed only after rails, clocks, references, self-test, and configuration have passed. Power-up is inhibited. Thresholds change only while disarmed. A trip stays latched until an explicit clear, and clear does not arm and does not erase the event record. There is no automatic restart in the required prototype.

The external emergency comparator uses its own fixed reference so a software bug or a DAC failure cannot raise the "shut down now" level.

## The chip, from the software seat

PolarFire SoC on the Icicle Kit is one part with two worlds:

- **Fabric.** Custom logic plus Microchip IP (clocks, RAM, FIFOs, DMA, buses). Protection clock target is 200 MHz, so one tick is 5 ns. If timing does not close, the clock slows down and every deadline in the software-visible ABI has to be recomputed.
- **MSS (microprocessor subsystem).** Five 64-bit RISC-V cores, DDR, Ethernet, storage.

| Core | Spec name | Software |
| --- | --- | --- |
| Monitor | E51 | Microchip HSS (Hart Software Services). Boots the chip and loads the other cores |
| Application | U54_1 | Bare-metal. Safety configuration, DAC, arm/disarm, event IRQ, DMA control, calibration |
| Application | U54_2, U54_3, U54_4 | Linux. Archive, network, dashboard, campaign I/O |

This split is AMP (asymmetric multiprocessing): different cores, different software, one chip. The Linux hart mask must leave U54_1 out. HSS is what actually enforces that at boot. The exact HSS payload, memory map, and peripheral ownership are still an open decision (O13). Bringing up "Linux on the Icicle" from the reference design is the first software milestone, and then that reference design gets edited so our pins and our core split still boot.

Two heartbeats exist, and they are different:

- Fabric heartbeat toward an **external** watchdog. Candidate period 10 µs, timeout 100 µs. This is hardware health. A stopped FPGA clock inhibits switching.
- Bare-metal heartbeat into a fabric register. Candidate period 10 ms, timeout 100 ms. If the safety core wedges, hardware revokes permission. This is not the external watchdog.

Linux failing to boot must leave the converter inhibited, with bare-metal still able to report why.

## What crosses into software

The fabric keeps a ring of recent samples in on-chip RAM and exports **events**, not a live firehose.

Proposed capture:

- 4 channels, 16-bit, 1 MSPS, common sample index: `I_dc`, `I_load`, `V_bus`, `V_sw`
- Raw frame is 8 bytes, little endian: `I_dc`, `I_load`, `V_bus`, `V_sw` as ADC codes
- Continuous raw rate is 8 MB/s, but that stream stays in the fabric ring
- Ring is 128 KiB, about 16,384 frames, about 16 ms
- One event is 8 ms before the trigger and 8 ms after: 8,000 + 8,000 frames = 128,000 raw bytes
- Engineering units (milliamps, millivolts) are applied from calibration metadata. The file keeps the raw codes

While the ring is frozen for export, a second event cannot get a full waveform. The campaign runner has to wait until capture is ready again. A missed waveform still leaves a sticky cause, a timestamp, and a loss counter. Silence is a bug.

DDR queue (target integration, after local capture works):

- 64 MiB carveout, hidden from Linux's general allocator
- Candidate slot size 160 KiB, on the order of 400 slots after headers
- Descriptor states: `FREE → WRITING → COMPLETE → READING → FREE`, plus an error state
- `COMPLETE` is published only after the payload and checksum are finished
- A full queue drops the new waveform and keeps the events already complete
- DDR is volatile. Power loss drops anything not yet on eMMC. eMMC shares its interface with the SD slot, so do not assume both are available at once

**Cache.** FIC2 writes DDR behind the CPU caches. The recorder region has to be mapped uncached, or software has to do explicit cache maintenance, or Linux will display a stale or torn waveform. This is a classic AMP bug and it will look like "the FPGA sent garbage."

**64-bit values.** Timestamps, event IDs, and frame counters are split across two 32-bit registers. Use the snapshot request/ack pair (`SNAPSHOT_REQ` / `SNAPSHOT_ACK` plus the latched words). A plain double read can tear.

## Register contract you will program against

Offsets below are a **proposed** APB block. The physical base address is open and must not collide with the reference design. Registers are 32-bit, little endian. Writes that change behavior are allowed while disarmed. Linux's view of this block is read-only status. File permissions are not the isolation mechanism; PMP/MPU and the boot configuration are. APB itself does not say which core issued a transaction, so isolation has to be proven in the memory map and then tested by trying to bypass it.

Commands are codes, not a bit that turns the gate on:

| Code | Command | Effect |
| --- | --- | --- |
| `0xA5C30001` | ARM | Accepted only from READY, with a valid config, healthy sources, and capture ready for the selected mode |
| `0xA5C30002` | DISARM | Revokes permission to switch |
| `0xA5C30003` | CLEAR_TRIP | Does not arm, does not erase the record, does not clear a fault that is still asserted |
| `0xA5C30004` | SELF_TEST | Power path stays inhibited |

Configuration is a shadow set plus `CONFIG_COMMIT`. Success bumps `APPLIED_REVISION`. Rejection leaves the live config untouched. "Bare-metal accepted the mailbox message" is not "the fabric is running the new thresholds." DAC settling is part of the apply result. Return the applied revision.

The protection state machine, as software sees it: `RESET_INHIBIT → DISARMED → SELF_TEST → READY → ARMED_NORMAL`, then `OBSERVE` while a deadline runs, then either `GUARDED_CONTINUE` or `SHUTDOWN` / `FAULT_LATCHED`. Software may request arm from READY. Software may disarm. Software may start the clear sequence only when permission is already low and sources are healthy.

Status worth surfacing on day one: `STATE`, `TRIP_CAUSE` (first cause kept, later causes accumulated), `DECISION_STATUS`, `DECISION_LATENCY_TICKS`, `HEALTH_STATUS`, `RECORDER_STATE`, `RECORDER_FLAGS`, `CAPTURE_LOSS_COUNT`, `DDR_FREE_SLOTS`, `APPLIED_REVISION`, `CALIBRATION_ID`, build IDs. Interrupt ack does not clear the protection latch.

A separate indexed block will hold calibration coefficients and rule tables. Its layout is open (O14). Do not invent a second config path beside commit.

## Event file

One versioned binary waveform plus a JSON manifest (exact header packing is open, O14). Fields the viewer and the campaign log both need:

- Identifiers: `event_id`, and campaign/shot IDs that are **ground truth**. Those labels must never be fed back into the decision logic as features.
- Versions: RTL build, firmware build, software build, rule revision, calibration id, envelope revision. A decision that cannot name these is not reproducible.
- Time: trigger tick, decision tick, inhibit tick, first aperture tick, sample period (candidate 200 ticks at 5 ns = 1 µs).
- Outcome: first cause, accumulated causes, decision class, requested action, applied action. Inference and what the pins actually did are separate fields.
- Quality: feature validity and age, capture flags (short prehistory, gap, truncation, overflow), transport errors, loss count. Unknown backup-protection status is stored as unknown.
- Payload: raw channel order, scale and offset, actual pre/post counts, checksum.

Parser limits reject oversized headers before allocating. An incomplete file must not show up as a finished waveform. The checksum catches accidents, not an attacker.

## Linux and the dashboard

| Piece | Job |
| --- | --- |
| Bare-metal service | Configure while disarmed, apply a checked revision, self-test, arm/disarm, service IRQs, run the queue |
| Linux broker | Take `COMPLETE` records, write the archive, report health and loss counters. No safety ownership |
| Web dashboard | Event list, waveform, markers for trigger / decision / inhibit, class, action, reason, validity, build and calibration IDs, completeness flags |
| Status API | Health, queue, builds. Units and schema version on every response |
| Command API | Disarm, configure, self-test, arm, forwarded to bare-metal. States are accepted, applied, or rejected |
| Campaign runner | Run an approved shot list, check ready/health, interlock the shot, attach ground truth and the scope file, enforce cooldown, keep every outcome including failures |
| UART console | Health and configuration when the network or the browser is down |

Rules that are easy to violate from a normal web app:

- Page load, reconnect, or a disconnected browser does not arm, disarm, or change a threshold.
- A safety-changing request is an explicit operator action with state checks.
- The UI says whether a number is a design target or a measurement, and whether a trace is recorded or an illustration.
- Gaps are drawn as gaps. Unresolved decisions are visually obvious.
- Polling and streaming are rate-limited so logging cannot starve the bare-metal control service.
- Apply, reject, and the reason go to an audit log tied to the revision.

The host PC is a viewer and a place to keep scope files. It is not a fourth protection layer.

## What you have to know about the hardware without designing it

You do not need to write Verilog to use the record correctly. You do need these facts, because they show up as flags and as wrong-looking waveforms.

- **Two currents.** `I_dc` is on the supply side. `I_load` is on the load side. During freewheel (both switches off, current in the body diode) one of them can read near zero while the other still carries current. A mismatch is information. It is not automatically a sensor failure.
- **`V_sw` is the switching node.** It slams between 0 V and the bus every cycle. It is not a ground-referenced voltage a normal probe or amp can touch. The record's scale/offset for this channel will look odd. That is expected.
- **Comparators make the fast decision. The ADC explains it afterward.** The 1 µs decision does not wait for a fresh multi-sample derivative. ADC slope in the file is a microsecond-scale feature. Comparator slope is the nanosecond-scale one, stored as an interval with a validity flag. Software may convert that interval to A/µs for display. The fabric compares intervals with multiplies so it does not need a divider on the critical path.
- **Emergency always beats a "looks benign" result.** If both fire, the record should show emergency as the applied action.
- **Blanking and recovery.** Right after a switching edge, or while an amplifier is recovering from saturation, samples can be marked invalid. Those samples stay in the file. They do not get interpolated into "measured" evidence.
- **Shutdown is not the end of the story.** Current and voltage overshoot continue after the logic trips (`V = L di/dt`). "Fault-to-safe-current" is a separate, longer number from the <200 ns logic path. The dashboard should not label decision latency as "the current was gone."
- **J26 pinmux can break Linux.** The Icicle reference design already routes UART, QSPI, and I2C through some of these header pins. Reclaiming them for comparators and ADC lanes has to leave eMMC, Ethernet, and the console working, or the software seat has no machine to boot. That check is IO-002 and IO-003, owned by ES, and it happens before any custom board is ordered.

Candidate parts (they can change): TLV3601/3602 comparators, MCP4728 DAC, ADS9224R ADCs, OPA356 amplifier, ADuM110N isolator, a Microchip 700 V SiC MOSFET run at 48 V. Treat names in the glossary as candidates until the BOM is frozen.

## Build order that matches the spec

Local capture in on-chip RAM comes before DDR export. DDR export comes before "Linux shows a waveform." Optional features (soft turn-off, a replay DAC, oscillation diagnosis) wait until the required prototype works.

| When | Software-side work |
| --- | --- |
| Oct–Nov 2026, before the kit | Boot the exact reference release you will fork. Plan hart mask, DDR regions, and which reference pins you must not steal. Start a golden model for calibration math so later RTL and Python agree |
| 30 Nov – 18 Dec 2026 | Loopback on the real header. Capture logic. Bare-metal / AMP skeleton. Gates still forced off |
| 19 Dec 2026 – 17 Jan 2027 | Recorder integrated with real ADC samples. Rules still bounded |
| 18 Jan – 7 Feb 2027 | DDR queue, Linux archive, small dashboard. This is the connected-viewer milestone |
| 8 Feb – 7 Mar 2027 | Campaign automation, outcome accounting, software stress (stall Linux, fill the queue, pull the network) |
| 8–27 Mar 2027 | Feature freeze, clean build, README, disclosures, video |

Gate G5 in the spec is the software acceptance gate: raw-channel time order, pre/post capture, DDR ownership, archive completeness, the bare-metal/Linux split, and stress tests. G7 is the contest package.

## Open decisions that block software

Do not silently pick these. Each one needs a recorded close (date, reviewer, build, calibration id).

| ID | Decision | Why it blocks you |
| --- | --- | --- |
| O13 | AMP, peripherals, PMP/MPU | Which core runs Linux, which devices it can see, and whether a root process can poke safety registers |
| O14 | DDR addresses, slot layout, event ABI | The file format, the checksum, and the parser tests |
| O12 | 200 MHz and resource map | Tick-to-nanosecond conversion and every latency field |
| O09 | ADC protocol and reset | Frame association, gap flags, what "invalid pipeline" means in the file |
| O04 | Exact kit revision and pin constraints | Whether Ethernet, eMMC, and UART survive the pinmux |
| O02 | Portal deadline and file names | Contest admin, shared with PE |

## Spec items still assigned to ES that are not "just Linux"

The spec's ES seat also owns a large part of the fabric-facing logic: ADC serial mode, recorder state machine, DMA bounds, the golden fixed-point model, and cocotb (or equivalent) tests of the register and mailbox behavior. PE shares FPGA integration. If your scope is bare-metal plus Linux only, agree that split explicitly with PE before the December milestone, and keep a single versioned integration branch so pin constraints and the device tree cannot drift apart.

The tests that will fall on this side either way:

- Mailbox rejects a bad range, a bad state, and any attempt to arm by writing around bare-metal (V-S12).
- Shadow commit applies atomically and only while disarmed (V-S06).
- DMA stays inside the carveout and does not publish `COMPLETE` early (V-S09).
- 64-bit snapshots do not tear (V-S11).
- Linux stress and a pulled cable leave thresholds, arming, and deadlines unchanged (SW-003, SW-004, K09, K13).

## Source documents

These notes compress SS-001 rev 0.2 and the contest rules. The spec also says to fix the earlier architecture diagram's 24-pin vs 26-pin wording before treating that diagram as current.
