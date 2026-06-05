# claude-skill-moxa-iopac8600

A [Claude skill](https://support.claude.ai/) for C/C++ development on the **Moxa ioPAC 8600-CPU30** RTU controller — API patterns, cross-compile setup, and critical gotchas, all verified from official Moxa sample source code.

---

## What is a Claude Skill?

A Claude skill is a `SKILL.md` file you drop into your Claude project. It gives Claude deep, accurate knowledge of a specific platform or domain that it wouldn't otherwise have reliably in its training data.

For the ioPAC 8600, this matters a lot. Without the skill, Claude has to guess at the `libmoxa_rtu` API and frequently gets things wrong — return-code namespaces, argument order, mandatory call sequences. With the skill loaded, it generates correct, deployable code from the first prompt.

---

## What's in the Skill

The skill covers all 30+ `libmoxa_rtu` subsystems:

| Category | Subsystems |
|---|---|
| Digital I/O | DI, DO, DIO — including counter, frequency, PWM modes |
| Analog I/O | AI, AO, Fast AI (triggered batch capture) |
| Sensors | RTD, Thermocouple |
| Other I/O | Relay |
| Memory | FRAM, SRAM |
| System | LED, RTC, Watchdog, Power supply, Rotary switch, Toggle switch |
| Comms | Modbus TCP master+slave, Modbus RTU master+slave |
| Bus | Serial port, CAN / CANopen |
| Connectivity | GPS, Cellular modem, SMS, AOPC |
| Events | DI events, AI events, DIO events, Hot-plug detection |
| Build | Cross-compile Makefile, GDB debug, deployment |

It also documents the things that trip people up most:

- **Return code namespaces** — each subsystem uses a different success constant (`MODULE_RW_ERR_OK`, `IO_ERR_OK`, `MISC_ERR_OK`, etc.). Getting this wrong silently breaks error handling.
- **DO / Relay read-modify-write** — `Value_Set` writes all 32 bits at once. You must read first or you'll clobber other channels.
- **C++ wrapping** — `libmoxa_rtu` is a C library. In C++ files you must wrap the include: `extern "C" { #include <libmoxa_rtu.h> }` or the linker will fail.
- **Mode-before-use** — `DI_Mode_Set` and `DO_Mode_Set` must be called once before first use.
- **AI event namespace** — AI events return `IO_ERR_OK`, not `MODULE_RW_ERR_OK`. An easy mistake that breaks event handling silently.

---

## How to Use It

**1. Open or create a Claude project**

Skills live inside Claude projects. If you don't have one, create a new project at [claude.ai](https://claude.ai).

**2. Add the skill**

Upload `SKILL.md` to your project's knowledge base. Claude will automatically reference it when you ask ioPAC-related questions.

**3. Start developing**

Ask Claude anything about ioPAC development — reading DI channels, setting up Modbus slave callbacks, configuring Fast AI batch capture, building a Makefile. The skill keeps it accurate.

---

## Platform Quick Reference

| Item | Value |
|---|---|
| Device | Moxa ioPAC 8600-CPU30 |
| CPU | ARM Cortex-A8 (ARMv7) |
| OS | Linux 4.1.15 with PREEMPT_RT real-time patch |
| Compiler | `arm-linux-gnueabihf-g++` (GCC 5.1.1, glibc 2.21) |
| Master include | `#include <libmoxa_rtu.h>` — one header, all APIs |
| Default LAN1 IP | `192.168.127.254` (eth0) |
| Default LAN2 IP | `192.168.126.254` (eth1) |

---

## Related

Looking to set up the full cross-compile toolchain from scratch? See the companion repo:
👉 **[moxa-iopac8600-cross-compile](https://github.com/ai-rafique/moxa-iopac8600-cross-compile)** — toolchain install, sysroot setup, Makefile, and API smoke test.

---

## Disclaimer

API signatures, constants, and behavioural patterns referenced in this skill are derived from Moxa Inc. official SDK documentation and sample source code. Moxa Inc. retains all intellectual property rights over the `libmoxa_rtu` SDK. This skill document is an independent work of curation and is not affiliated with or endorsed by Moxa Inc.
