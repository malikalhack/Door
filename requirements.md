# Door — Requirements & Concept Backlog

**Project:** Door (electronic solenoid-lock access controller)
**Author:** Anton Chernov
**Date:** 07/02/2026
**Version:** 0.1.0 (concept draft)

---

## 0. System Overview

A low-power controller for a door fitted with an electric **solenoid lock**.
The controller stays asleep whenever the door is closed and idle, and wakes
only to service an access interaction, a sensor change, or a danger signal.
Access is granted via an **RFID reader (RC522)** and, optionally, a
**16-key capacitive keypad**. The system is backed by an **uninterruptible
power supply (UPS)** and honours two external **danger signals** (inside /
outside) with hardware-enforced safety priority.

---

## 1. Requirements

### REQ-001 — Solenoid Lock Drive

The controller shall drive a **solenoid-based electric lock** that is
**normally locked** (fail-secure): de-energised = locked, energised = unlocked.

**Given data:**
- Supply: **12 V**, hold current **≈ 150 mA** while energised (unlocked/open-held).
- Near-zero current when locked (de-energised).
- The lock provides an internal **auto-relock delay** of **0 / 3 / 6 s**
  (configurable on the lock, see REQ-006).

**Acceptance criteria:**
- The solenoid is switched by a low-side **MOSFET** (or relay) with a
  **flyback/free-wheeling diode** across the coil (inductive load protection).
- The drive path is in the **switched power domain** (REQ-002), not the
  standby domain.
- Continuous open-hold (150 mA) is accounted for in the UPS energy budget
  (REQ-012).
- The **locked (de-energised) quiescent current** shall be **measured on the
  actual lock** (declared "negligible") and fed into the standby energy budget.

---

### REQ-002 — Two-Domain Power Architecture

The system shall use **two power domains** to minimise standby consumption.

**Acceptance criteria:**
- **Standby domain** (always on): MCU core, RTC/wake logic, wake-up input
  conditioning, and the emergency-unlock circuit (REQ-010). Target quiescent
  current in the single-digit-µA range.
- **Switched domain** (gated on demand): solenoid drive, RFID field (RC522),
  keypad backlight/scan, and any indicator LEDs. Enabled by a load switch
  (P-MOSFET / high-side switch) under MCU control only for the duration of a
  transaction.
- No peripheral in the switched domain shall draw current while the MCU sleeps.

---

### REQ-003 — Ultra-Low-Power Sleep

The MCU shall remain in a deep low-power sleep state whenever the door is
closed and no interaction is in progress.

**Acceptance criteria:**
- Default runtime state is the deepest sleep mode that retains the wake
  sources of REQ-004 (e.g. MSP430 LPM3/LPM4, STM32L STOP).
- The MCU runs only in response to an event; it returns to sleep as soon as
  the state machine (REQ-013) reaches an idle state.
- The controller performs **no periodic busy-work** when the door is closed
  and secure.

---

### REQ-004 — Wake-Up Sources

The controller shall wake from sleep on any of the following events, delivered
as interrupts.

**Acceptance criteria:**
- **Keypad touch** — INT line from the capacitive keypad controller (REQ-008).
- **Danger signals** — inside / outside inputs (REQ-009), highest priority.
- **Sensor change** — door (Hall) or lock limit-switch edge (REQ-005).
- **Presence / access intent** — a low-power trigger (e.g. presence sensor or
  an activation button) that precedes powering up the RFID field, because the
  RC522 cannot wake the MCU on its own. Note: a presence sensor may itself be
  a **significant standby consumer**; a button or the keypad as "first contact"
  is preferred where always-on presence sensing would dominate the energy budget.
- **Power source change** — mains-lost / battery-low from the UPS (REQ-012).
- Optionally a periodic RTC wake for self-test / housekeeping.

---

### REQ-005 — Door & Lock Sensing with Line Supervision

The controller shall read two independent feedback sensors and supervise their
wiring against fault and tamper.

**Given data:**
- **Door sensor:** Hall sensor, provides **NO and NC** contacts.
- **Lock sensor:** lock-closed limit switch, provides **NO and NC** contacts.

**Acceptance criteria:**
- The four sensor states form the base matrix: door {open, closed} × lock
  {open, closed}; this matrix drives the state machine (REQ-013).
- Sensing exploits the NO+NC pair for **line supervision**: a valid line is
  always "one contact made, the other open". Both-made or both-open indicates
  **open circuit, short, or tamper** and raises a fault.
- Supervision may use **end-of-line (EOL) resistors** so each line is read as
  a multi-level analogue value on a single ADC input (fewer pins).
- A "door closed but lock not latched within the expected window" condition
  (see REQ-006) is detectable and reported as an anomaly.

---

### REQ-006 — Auto-Relock Awareness

The firmware shall be aware of the lock's own auto-relock delay and shall not
fight it.

**Acceptance criteria:**
- The configured relock delay (0 / 3 / 6 s) is a known firmware parameter.
- After granting access the firmware treats the delay as the **open window**;
  it does not attempt to re-lock or re-drive the solenoid during that window.
- Latch confirmation is taken from the lock limit switch (REQ-005), not assumed.

---

### REQ-007 — RFID Access

The controller shall grant access via an **RFID-RC522** reader.

**Acceptance criteria:**
- The RC522 is interfaced over **SPI** (SCK/MOSI/MISO/SS + RST); its **IRQ**
  line is used to avoid polling.
- The RC522 lives in the switched power domain and is powered/field-enabled
  only during an access interaction, then returned to power-down.
- A credential store (allowed UIDs) is maintained; management of the list is
  addressed in a later requirement.

---

### REQ-008 — Capacitive Keypad (optional)

The controller may support a **16-key capacitive touch keypad**.

**Acceptance criteria:**
- The keypad uses a touch controller (e.g. **TTP229**, I²C or serial mode)
  exposing an **INT** output usable as a wake source (REQ-004).
- The keypad is optional: the build compiles and the system operates without it.
- Keypad scan/backlight power is in the switched domain.

---

### REQ-009 — Danger Signals (inside / outside)

The controller shall accept two external danger inputs with **asymmetric**,
safety-prioritised behaviour.

**Acceptance criteria:**
- **Inside danger** → **unlock and hold open** (life-safety egress priority).
- **Outside danger** → **block the door** (deny entry).
- Both inputs are **supervised** (normally-closed loop preferred; open = alarm)
  so a cut wire is detected.
- Danger handling has **hardware priority** over normal access logic and over
  firmware state (see REQ-010); firmware must not be able to override the
  inside-danger unlock.

---

### REQ-010 — Hardware Emergency Unlock & Safety Fallback

The system shall provide an unlock path that works **independently of the MCU**.

**Acceptance criteria:**
- A hardware button (and the inside-danger signal) can **energise the solenoid
  directly**, bypassing the MCU, so the lock opens even if the controller is
  hung or dead — as long as power is available.
- **OPEN SAFETY POINT (to resolve):** because the lock is fail-secure, a
  **total power loss leaves it locked**. Inside egress "always" therefore
  requires one of:
  1. a **mechanical override** (thumbturn / handle) independent of electronics, or
  2. the emergency circuit **energises the solenoid from the UPS** on
     inside-danger / manual request.
  The chosen option shall be recorded before hardware design is frozen.
- When the MCU is alive, a valid request (manual, danger, or smart-home,
  REQ-011) unlocks and holds per policy.

---

### REQ-011 — Smart-Home Integration

The controller shall accept an unlock/hold command from a central smart-home
controller.

**Acceptance criteria:**
- A central-block signal requests the door to **unlock and remain held open**
  until released.
- This path is **MCU-mediated** (subject to firmware policy and logging),
  distinct from the hardware emergency path of REQ-010.
- The interface (wired signal vs bus/protocol) is to be defined; a simple
  opto-isolated input is the baseline.

---

### REQ-012 — Uninterruptible Power Supply

The system shall include a UPS so it operates through mains interruptions.

**Acceptance criteria:**
- Battery-backed 12 V supply (candidate chemistry **LiFePO₄** for buffer duty
  and temperature tolerance) with a charge controller and 12 V regulation.
- The **UPS is mounted on the door leaf itself**, so the critical emergency
  loop (button + solenoid + battery, REQ-010) closes on the moving leaf and
  does not depend on power crossing the door hinge.
- **Deep-discharge protection and charge monitoring**: the MCU can distinguish
  mains / battery / battery-low and is woken on transitions (REQ-004).
- A **low-charge policy** shall prioritise reserving energy for emergency
  unlock over non-essential loads (RFID field, backlight, presence sensing).
- The energy budget accounts for **continuous 150 mA open-hold** and worst-case
  hold duration.
- On power failure the system enters a defined **safe state** consistent with
  REQ-009/REQ-010 (inside egress preserved while battery charge remains).

---

### REQ-013 — Event-Driven State Machine

The controller behaviour shall be modelled as an explicit state machine.

**Acceptance criteria:**
- States derive from the door × lock matrix (REQ-005) plus error/fault and
  danger states.
- Transitions are triggered by wake events (REQ-004); the machine returns to a
  sleep-eligible idle state after handling each event.
- Anomalies (unlatched lock, supervised-line fault, tamper) have explicit
  states and reporting.

---

### REQ-014 — Target MCU Candidates

The design shall target a low-power MCU from the following shortlist.

**Acceptance criteria:**
- Candidates:
  - **TI MSP430G2553** — ultra-low-power sleep (LPM3/LPM4), on-hand.
  - **STM32L052C8T6** — Cortex-M0+, STOP mode, on-hand.
  - **STM32L010F4P6** — Cortex-M0+, low cost, available for purchase.
- The **STM32 F-series is excluded** for this project by choice.
- The final selection is justified against the standby-current budget (REQ-002/003),
  peripheral needs (SPI + I²C + ADC + several interrupts), and pin count.

---

### REQ-015 — Wiring to the Moving Door Leaf

Power and signals that must cross to the moving leaf shall use a mechanism
rated for repeated door travel.

**Acceptance criteria:**
- A **flexible door-loop cable** (or equivalent hinge transit) carries mains/
  charging power and any off-leaf signals to the leaf-mounted electronics.
- The **critical emergency loop stays entirely on the leaf** (REQ-010/REQ-012)
  so a severed transit cable cannot disable inside egress while the battery
  holds charge; a transit break is instead detectable as a **mains-lost / no-
  charge** condition (REQ-012).
- Conductor sizing and flex rating account for the door's motion and cycle life.

---

## 2. Open Questions

- **OQ-1 (safety) — DECIDED, residual risk tracked:** A mechanical override is
  currently **not feasible**, so inside egress relies on the **UPS-energised
  solenoid** (REQ-010). "Total power loss" here means **mains lost AND battery
  deep-discharged**, not mere mains absence (the UPS covers mains absence).
  Residual risk is therefore **battery deep discharge**, mitigated by
  deep-discharge protection, charge monitoring, and the low-charge policy
  (REQ-012). Revisit if a mechanical override later becomes possible.
- **OQ-2:** Smart-home interface — simple wired/opto input vs a bus/protocol
  (REQ-011).
- **OQ-3:** RFID wake strategy — dedicated presence sensor vs activation
  button vs keypad-as-first-contact (REQ-004/REQ-007).
- **OQ-4:** Credential management — how the allowed-UID / PIN list is
  provisioned and updated.
- **OQ-5:** Final MCU choice among the three candidates (REQ-014).

---

## 3. Assumptions & Exclusions

- **Lock mechanical failure** (jammed/broken solenoid mechanism) is **out of
  scope** for the electronics: no electrical remedy exists; recovery is by
  mechanical action on the door only.
- Mains absence alone is **not** a hazard — the UPS covers it; only combined
  mains-loss + battery deep-discharge threatens inside egress (see OQ-1).
