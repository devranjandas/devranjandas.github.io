---
layout: default
title: Building a SCADA-Connected Power Supply Controller with a Raspberry Pi and IEC 61499
permalink: /blog-iec61499-power-supply-control/
---


[Home](/) · [Weekend Project Ideas](/weekend-project-ideas/)

*My thoughts on a Saturday morning*

---

## The Idea

What if a Raspberry Pi could sit between three programmable power supplies and a SCADA operator station — reading live voltage and current, running a control loop on the setpoints, and letting an operator adjust everything from a CitectSCADA HMI?

That's the weekend project. The supplies talk **SCPI** (the standard ASCII command language used across test & measurement gear), the controller logic is built in **Eclipse 4diac** (an open-source IEC 61499 implementation), and the SCADA side is **AVEVA Plant SCADA (CitectSCADA)**. Before writing a line of code, it was worth thinking through a few architecture questions that keep coming up whenever "Raspberry Pi" and "industrial control" appear in the same sentence.

## "Should the PID loop run on the Pi, or should I send everything to a PLC?"

This is the question that usually triggers a reflexive "use a real PLC" — and for good reason, in many contexts. But the right answer depends entirely on **how fast your loop actually needs to be**, and that's set by your slowest link.

In this system, that's SCPI. A `MEAS:VOLT?` query over TCP to a bench power supply takes somewhere between 10 and 100 milliseconds round-trip, dominated by the instrument's own command parser and ADC settling — not by the network. No amount of PLC determinism changes that number, because the PLC would still be waiting on the same SCPI response.

What's *actually* happening here is a two-tier control structure that already exists whether we design for it or not:

- **Inner loop (microseconds):** the power supply's own analog/digital regulation hardware, keeping output voltage/current locked to its internal setpoint. This is already real-time and already reliable — it ships that way from the factory.
- **Outer loop (1–10 Hz):** a *supervisory* loop that adjusts the supply's setpoint based on some external measurement or profile — ramping, current-limiting based on temperature, charge-curve following, etc.

The Pi only needs to handle the outer loop, and at 1–10 Hz, a Linux box running 4diac's FORTE runtime is plenty deterministic. Sending data to a PLC and back would add a protocol hop and a sync problem without buying back any speed, because the bottleneck — SCPI — sits *between* the controller and the actuator regardless of which box does the math.

**Where a PLC genuinely earns its keep:** safety interlocks that must survive a Pi crash, sub-10ms loops on real hardware I/O, or certified/hardened deployment requirements. None of those describe "trim three power supply setpoints based on a slow SCPI readback."

The pragmatic safety net, regardless of architecture: configure each supply's **hardware OVP/OCP** directly on the instrument. That protection then exists independent of whatever the Pi is doing — including if it's not doing anything because it crashed.

## "Is there a ready-made PID block, or do I write one from scratch?"

4diac's FORTE ships with `FB_PID` as part of its standard control function block library — ported from the same lineage as the IEC 61131-3 OSCAT control blocks that PLC programmers already know. Drive it from a cyclic event source for a consistent `dT`, and it's a solid starting point. Writing a custom PID FB in Structured Text is also a genuinely good learning exercise (it's about 20 lines), but for getting something working, the standard block is the right default.

## "Pi + Ethernet — is that an industrial red flag?"

Sort of, but not for the reason people usually think. **Ethernet itself is fine** — even industrially. Standard TCP/IP Ethernet running SCPI at 1-10Hz is exactly the right transport for this job. The "industrial Ethernet" protocols you've heard of — **EtherCAT, PROFINET IRT, EtherNet/IP with CIP Sync, TSN** — solve a *different* problem: deterministic communication at sub-millisecond cycle times for things like servo drives and motion control. They modify the MAC layer or add hardware time synchronization to guarantee jitter-free delivery. None of that is relevant when your "field device" round-trips at 50ms over its own SCPI parser.

The actual industrial concern with a bare Raspberry Pi is **the SD card** — power-loss corruption is the classic failure mode — plus the lack of conformal coating, wide-input DC power, and DIN-rail mounting. The fix isn't a different network protocol; it's either a read-only/overlay filesystem and a UPS, or stepping up to an "industrial Pi" carrier board (eMMC storage, DIN rail, isolated I/O) if this ever needs to leave the bench.

## "Then why does test equipment still use SCPI instead of something more 'industrial'?"

Because SCPI is solving a completely different problem than fieldbus protocols are. SCPI is a **command language** — `VOLT 5.0`, `MEAS:CURR?` — standardized so that the same script can drive a Keysight, Rigol, or Tektronix instrument with minimal changes. It can ride over GPIB, USB-TMC, RS-232, or LAN — the transport is interchangeable. The design goal is **interoperability for test engineers writing automation scripts**, not loop-rate determinism.

When an industrial power supply genuinely needs to participate in a fast control loop, it usually offers an **analog 0–10V reference input** for that, with SCPI/Ethernet kept as the "slow lane" for configuration, monitoring, and logging. That's effectively the same two-tier split described above — SCPI persists because the slow lane never went away, even as fast lanes evolved.

## "How do I get this onto SCADA — Modbus or OPC UA?"

This one had a clearer answer than expected: **OPC UA**. Two reasons converged:

1. FORTE's Modbus support is solid as a *client*, but building a reliable Modbus *server* (slave) for SCADA to poll is the harder, less-supported direction.
2. CitectSCADA (current AVEVA Plant SCADA releases) has a **native OPC UA I/O Server driver** — point it at FORTE's OPC UA endpoint, browse the address space, map tags. No gateway needed.

OPC UA also gives structured, typed data — units, timestamps, quality codes — versus a flat Modbus register map, which matters once you're exposing three power supplies' worth of voltage, current, setpoints, and status to an HMI.

One caveat worth flagging for anyone replicating this: OPC UA client support in CitectSCADA depends on version — it showed up in v7.x 2018+ releases. Worth checking before assuming it's there.

For testing the OPC UA server *before* touching the SCADA configuration at all, **UaExpert** (free, Windows) is the standard tool — connect, browse, subscribe, and confirm read/write behavior in isolation. This decouples "is my FORTE server correct" from "is CitectSCADA configured correctly," which is a much easier debugging position to be in.

## "Should SCPI be implemented inside 4diac, or in Python?"

**Python**, and this was the most clear-cut decision of the bunch. IEC 61131-3 Structured Text has no real string formatting, regex, or socket ergonomics comparable to Python — building SCPI command strings and parsing instrument responses (`"+5.000000E+00\r\n"` and friends) is fiddly in ST and trivial with `pyvisa`.

The resulting split:

- **Python microservice** — one per supply (or one process handling all three) — owns the SCPI/PyVISA transport, command formatting, response parsing, and error-queue checking (`SYST:ERR?`).
- **4diac/FORTE** — owns the control logic: PID, sequencing across the three supplies, scaling, and the OPC UA server.
- A simple local IPC mechanism (TCP/JSON or MQTT) connects the two layers on the Pi.

The nice side effect: if a power supply gets swapped for a different vendor with a different SCPI dialect, **only the Python layer changes**. The 61499 application — PID tuning, sequencing, OPC UA tag mapping, SCADA configuration — stays untouched. That's a real maintainability win for what's nominally a "weekend project."

## Where This Lands

The architecture that fell out of these questions is a fairly clean five-layer stack:

```
Power Supplies (SCPI/TCP)
   ↓
Python + PyVISA (device drivers, one per supply)
   ↓  (local IPC)
4diac/FORTE — IEC 61499 (PID, sequencing, scaling)
   ↓
FORTE OPC UA Server
   ↓
CitectSCADA OPC UA I/O Server (+ UaExpert for pre-validation)
```

Nothing here required exotic hardware, real-time Ethernet, or a PLC. The interesting part of the exercise wasn't finding clever technology — it was correctly identifying *which* layer of the stack each requirement (determinism, safety, interoperability, maintainability) actually belongs to, and resisting the urge to over-engineer the layer that doesn't need it (looking at you, EtherCAT).

Next up: building the first per-supply composite function block and getting `FB_PID` talking to a real instrument over SCPI. Updates to follow.

---

*Have questions? Email me: devranjandas@gmail.com*
