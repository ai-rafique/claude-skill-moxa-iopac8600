# claude-skill-moxa-iopac8600
Claude skill for C/C++ development on the Moxa ioPAC 8600-CPU30 RTU — API patterns, cross-compile setup, and critical gotchas verified from official samples

**Purpose:**

This is a [Claude skill](https://support.claude.ai) — a `SKILL.md` file you drop into your Claude project to give Claude deep, accurate knowledge of the Moxa ioPAC 8600-CPU30 platform. Without it, Claude has to guess at the `libmoxa_rtu` API from partial training data and frequently gets return-code namespaces, argument order, and mandatory call sequences wrong. With the skill loaded, it generates correct, deployable code first time.

**What it covers:** all 30+ `libmoxa_rtu` subsystems — DI/DO/DIO/AI/AO/RTD/TC/Relay, Fast AI, FRAM/SRAM, LED/RTC/Watchdog, Modbus TCP/RTU master+slave, raw SocketCAN and CANopen, serial port, GPS, cellular, SMS, AOPC, events, hot-plug, plus cross-compile Makefile and deployment.

---
