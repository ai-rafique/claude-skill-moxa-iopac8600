---
name: moxa-iopac8600-cpp
description: >
  Expert skill for developing, structuring, cross-compiling, and deploying C/C++
  applications for the Moxa ioPAC 8600-CPU30 series programmable RTU controller
  (ARM Cortex-A8, Linux 4.1.15 PREEMPT_RT, arm-linux-gnueabihf toolchain).
  Use this skill whenever the user mentions ioPAC, Moxa RTU, moxa controller,
  ARM cross-compilation, iopac programming, RTU C++ development, Modbus on ioPAC,
  DI DO AI AO DIO relay RTD TC fast AI FRAM SRAM LED RTC watchdog rotary switch
  toggle switch power supply system info hot-plug, I/O module programming, serial
  port, CAN, GPS, SMS, cellular, AOPC, events, or any task involving
  compiling/deploying to the ioPAC 8600 platform. Also triggers for Makefile
  generation, project scaffolding, GDB debugging, firmware upgrade, or
  library/API usage on this hardware.
---

# Moxa ioPAC 8600 CPU30 — Complete C/C++ Development Skill

All API patterns verified directly from official Moxa sample source code.

---

## Platform Quick Reference

| Item | Value |
|---|---|
| CPU | ARM Cortex-A8 (ARMv7) |
| OS | Linux 4.1.15 with PREEMPT_RT real-time patch |
| Compiler prefix | `arm-linux-gnueabihf-` (gcc 5.1.1, glibc 2.21) |
| **Master include** | `#include <libmoxa_rtu.h>` — one header, ALL APIs |
| Moxa lib path | `/usr/local/arm-linux/lib/RTU/` |
| Include path | `/usr/local/arm-linux/include/` |
| Default LAN1 IP | 192.168.127.254/24 (eth0) |
| Default LAN2 IP | 192.168.126.254/24 (eth1) |

---

## 1. Universal Rules

```c
#include <libmoxa_rtu.h>                        // C
```
```cpp
extern "C" { #include <libmoxa_rtu.h> }         // C++ — always wrap
```

**Slot numbering:** slot 0 = built-in CPU module; slots 1–9 = expansion modules left-to-right.

**Discover slots at runtime:**
```c
UINT32 slotMax = 0;
MX_RTU_Total_Slots_Get(&slotMax);
```

**Poll interval:** `usleep(200000)` (200ms) — used in all official Moxa samples.

**Check every return code.** All functions return 0 on success. Print slot/channel/rc on failure.

---

## 2. Return Code Namespaces

Different subsystems use different success constants — this matters:

| Subsystem | Success code |
|---|---|
| All I/O modules (DI/DO/AI/AO/RTD/TC/Relay) | `MODULE_RW_ERR_OK` (0) |
| FRAM, SRAM, System info, events (non-AI) | `IO_ERR_OK` (0) |
| LED, RTC, Watchdog, Power, Switches | `MISC_ERR_OK` (0) |
| AI events specifically | `IO_ERR_OK` (0) |
| Modbus master | `MODBUS_MASTER_ERR_OK` (0) |
| Modbus slave | `MODBUS_SLAVE_ERR_OK` (0) |
| Serial | `SERIAL_ERR_OK` (0) |
| CAN | `CAN_ERR_OK` (0) |
| AOPC | `AOPC_ERR_OK` (0) |
| Cellular modem | `MODEM_ERR_OK` (0) |
| SMS | `SMS_ERR_OK` (0) |
| Modbus callbacks | `RETURN_OK` (0) |

---

## 3. DI — Digital Input

### Basic read (bitmask)
```c
UINT32 diValue = 0;
struct Timestamp ts;

// Third arg is timestamp — pass NULL if not needed
UINT32 rc = MX_RTU_Module_DI_Value_Get(diSlot, &diValue, NULL);
// bit N = channel N state (0=OFF, 1=ON)
int ch5 = (diValue >> 5) & 1;
```

### Configuration (call once before polling)
```c
UINT8  modes[16], triggers[16];
UINT32 filters[16], counters[16];

for (int i = 0; i < 16; i++) {
    modes[i]    = DI_MODE_DI;              // DI_MODE_COUNTER / DI_MODE_FREQUENCY
    filters[i]  = 1000;                    // units of 100µs
    triggers[i] = DI_EVENT_TOGGLE_L2H;    // L2H / H2L / BOTH
    counters[i] = 0;
}
// args: slot, startChannel, count, array
MX_RTU_Module_DI_Mode_Set(slot, 0, 16, modes);
MX_RTU_Module_DI_Filter_Set(slot, 0, 16, filters);
MX_RTU_Module_DI_Counter_Trigger_Set(slot, 0, 16, triggers);
MX_RTU_Module_DI_Counter_Value_Set(slot, 0, 16, counters);   // reset counters
MX_RTU_Module_DI_Counter_Start_Set(slot, 0xFFFF);            // channel bitmask
```

### Counter read
```c
UINT32 counters[16] = {0};
MX_RTU_Module_DI_Counter_Value_Get(slot, 0, 16, counters, NULL);
```

### Frequency mode
```c
UINT8  mode = DI_MODE_FREQUENCY;
UINT32 measTime = 1000;   // measurement window in ms

MX_RTU_Module_DI_Mode_Set(slot, 0, 1, &mode);
MX_RTU_Module_DI_Filter_Set(slot, 0, 1, &filter);
MX_RTU_Module_DI_Frequency_Measurement_Time_Set(slot, 0, 1, &measTime);
MX_RTU_Module_DI_Frequency_Start_Set(slot, 0xFFFF);  // channel bitmask
float freq[1];
MX_RTU_Module_DI_Frequency_Get(slot, 0, 1, freq);    // Hz
```

### DI mode constants
| Constant | Meaning |
|---|---|
| `DI_MODE_DI` | Standard digital input |
| `DI_MODE_COUNTER` | Pulse counter |
| `DI_MODE_FREQUENCY` | Frequency measurement |
| `DI_EVENT_TOGGLE_L2H` | Low→high edge |
| `DI_EVENT_TOGGLE_H2L` | High→low edge |
| `DI_EVENT_TOGGLE_BOTH` | Any edge |

---

## 4. DO — Digital Output

### Mode setup (call once before first write)
```c
UINT8 modes[16];
for (int i = 0; i < 16; i++) modes[i] = DO_MODE_DO;
MX_RTU_Module_DO_Mode_Set(slot, 0, 16, modes);
```

### Read-modify-write — MANDATORY for all DO writes
`DO_Value_Set` writes the **entire 32-bit bitmask** at once. Always read first.
```c
UINT32 doVal = 0;
MX_RTU_Module_DO_Value_Get(slot, &doVal);
doVal |=  (1 << ch);   // turn channel ON
doVal &= ~(1 << ch);   // turn channel OFF
MX_RTU_Module_DO_Value_Set(slot, doVal);
```

### PWM output
```c
UINT8  m[1] = { DO_MODE_PWM };
float  f[1] = { 10.0f };    // Hz
float  d[1] = { 50.0f };    // duty cycle %
UINT32 c[1] = { 0 };        // pulse count (0 = infinite)
UINT32 flags = 1 << ch;     // channel bitmask

MX_RTU_Module_DO_Mode_Set(slot, ch, 1, m);
MX_RTU_Module_DO_PWM_Config_Set(slot, ch, 1, f, d);
MX_RTU_Module_DO_PWM_Count_Set(slot, ch, 1, c);
MX_RTU_Module_DO_PWM_Start_Set(slot, flags);
MX_RTU_Module_DO_PWM_Stop_Set(slot, flags);
```

### DO mode constants
| Constant | Meaning |
|---|---|
| `DO_MODE_DO` | Standard digital output |
| `DO_MODE_PWM` | PWM output |

---

## 5. DIO — Bidirectional Digital I/O (built-in, typically slot 0)

```c
// Set direction: 1=DO, 0=DI per bit (e.g. 0xF0 = ch0-3 DI, ch4-7 DO)
MX_RTU_Module_DIO_Map_Set(slot, 0xF0);

UINT8 chMode[8];
for (int i = 0; i < 4; i++) chMode[i] = DI_MODE_DI;
MX_RTU_Module_DIO_DI_Mode_Set(slot, 0, 4, chMode);   // ch0-3 as DI

for (int i = 0; i < 4; i++) chMode[i] = DO_MODE_DO;
MX_RTU_Module_DIO_DO_Mode_Set(slot, 4, 4, chMode);   // ch4-7 as DO

// Read DI bitmask
UINT32 diVal = 0;
struct Timestamp ts;
MX_RTU_Module_DIO_DI_Value_Get(slot, &diVal, &ts);

// Write DO bitmask (same RMW rules as DO)
UINT32 doVal = (diVal & 0x0F) << 4;
MX_RTU_Module_DIO_DO_Value_Set(slot, doVal);

// DIO filter (for DI-direction channels)
MX_RTU_Module_DIO_DI_Filter_Set(slot, ch, 1, &filter);
```

---

## 6. AI — Analog Input

```c
// Check burnout before reading
UINT8 status = 0;
MX_RTU_Module_AI_Status_Get(slot, ch, 1, &status);
if (status == AI_STATUS_BURNOUT) { /* handle open circuit */ }

// Read engineering value (V or mA)
float val = 0.0f;
MX_RTU_Module_AI_Eng_Value_Get(slot, ch, 1, &val, NULL);

// Configuration
UINT8 range = 0;
MX_RTU_Module_AI_Range_Get(slot, ch, 1, &range);   // AI_RANGE_4_20mA / AI_RANGE_0_10V
float burnout = 2.0f;
MX_RTU_Module_AI_Burnout_Value_Set(slot, ch, 1, &burnout);
```

| Constant | Meaning |
|---|---|
| `AI_RANGE_4_20mA` | 4–20 mA current loop |
| `AI_RANGE_0_10V` | 0–10 V voltage |
| `AI_STATUS_BURNOUT` | Open circuit detected |

---

## 7. Fast AI — High-Speed Triggered Batch Capture

```c
UINT8  range = AI_RANGE_4_20mA;
UINT32 rate = 500;           // samples/sec (10–5000)
UINT32 fore = 1, back = 1;  // pre/post-trigger seconds (0–3)
UINT32 flags = 1 << ch;
UINT32 requiredBufSize = 0;

MX_RTU_Module_Fast_AI_Range_Get(slot, 0, 1, &range);
MX_RTU_Module_Fast_AI_Sampling_Rate_Set(slot, rate);
MX_RTU_Module_Fast_AI_Buf_Overflow_Reset(slot, flags);
MX_RTU_Module_Fast_AI_Buf_Reset(slot);
MX_RTU_Module_Fast_AI_Burnout_Value_Set(slot, ch, 1, &burnout);

// Real-time single-channel read (also works while capturing)
float live = 0.0f;
MX_RTU_Module_Fast_AI_Eng_Value_Get(slot, ch, 1, &live, NULL);

// Trigger batch capture (retry if NOT_READY)
do {
    rc = MX_RTU_Module_Fast_AI_Trigger_Set(slot, flags, fore, back, &requiredBufSize);
} while (rc == MODULE_RW_ERR_IO_FAST_AI_NOT_READY);

// Optional: stop capture early
struct Timestamp stopTime;
MX_RTU_Module_Fast_AI_Trigger_Stop_Set(slot, flags, &stopTime);

// Read batch (retry if BUF_EMPTY)
UINT8 *buf = malloc(requiredBufSize);
struct Timestamp aiTime;
do {
    rc = MX_RTU_Module_Fast_AI_Batch_Data_Get(slot, ch, fore, back, buf, &aiTime);
} while (rc == MODULE_RW_ERR_IO_FAST_AI_BUF_EMPTY);

// Convert raw 16-bit → engineering units
for (int i = 0; i < requiredBufSize; i += 2) {
    UINT16 raw; float eng;
    memcpy(&raw, &buf[i], 2);
    MX_RTU_AIO_Raw_to_Eng(range, 16, raw, &eng);
}
free(buf);
```

| Code | Meaning |
|---|---|
| `MODULE_RW_ERR_IO_FAST_AI_NOT_READY` | Module busy — retry trigger |
| `MODULE_RW_ERR_IO_FAST_AI_BUF_EMPTY` | Buffer not ready — retry get |

---

## 8. AO — Analog Output

```c
UINT8 en[1]  = { 1 };               // AO_CHANNEL_ENABLE = 1
UINT8 rng[1] = { AO_RANGE_0_10V }; // or AO_RANGE_4_20mA
float val[1] = { 6.0f };            // output value in V or mA

MX_RTU_Module_AO_Enable_Set(slot, ch, 1, en);
MX_RTU_Module_AO_Range_Set(slot, ch, 1, rng);
MX_RTU_Module_AO_Eng_Value_Set(slot, ch, 1, val);
```

---

## 9. RTD — Resistance Temperature Detector

```c
UINT8 rtdType = RTD_TYPE_PT100;
MX_RTU_Module_RTD_Type_Set(slot, ch, 1, &rtdType);

UINT8 status = 0;
MX_RTU_Module_RTD_Burnout_Status_Get(slot, ch, 1, &status);
if (status == BURNOUT_STATUS_BURNOUT) { /* handle */ }

float val = 0.0f;
MX_RTU_Module_RTD_Eng_Value_Get(slot, ch, 1, &val, NULL);  // °C
```

---

## 10. TC — Thermocouple

```c
UINT8 tcType = TC_TYPE_J;
MX_RTU_Module_TC_Type_Set(slot, ch, 1, &tcType);

UINT8 status = 0;
MX_RTU_Module_TC_Burnout_Status_Get(slot, ch, 1, &status);
if (status == BURNOUT_STATUS_BURNOUT) { /* handle */ }

float val = 0.0f;
MX_RTU_Module_TC_Eng_Value_Get(slot, ch, 1, &val, NULL);  // °C

// Ice-point calibration (writes non-volatile offset)
float knownTemp = 0.0f;
MX_RTU_Module_TC_Calibration_Set(slot, ch, 1, &knownTemp);
```

| Constant | Sensor |
|---|---|
| `TC_TYPE_J/K/T/E/R/S` | Standard thermocouple types |
| `RTD_TYPE_PT100/PT1000` | RTD types |
| `BURNOUT_STATUS_BURNOUT` | Open circuit |

---

## 11. Relay

```c
UINT8  modes[6];
UINT32 arrTotal[6], arrCurrent[6], u32Val = 0;

for (int i = 0; i < 6; i++) modes[i] = RELAY_MODE_RELAY;
MX_RTU_Module_Relay_Mode_Set(slot, 0, 6, modes);
MX_RTU_Module_Relay_Mode_Get(slot, 0, 6, modes);        // verify

// Lifetime statistics
MX_RTU_Module_Relay_Total_Count_Get(slot, 0, 6, arrTotal);
MX_RTU_Module_Relay_Current_Count_Get(slot, 0, 6, arrCurrent);

// Read/write state (bitmask — same RMW rules as DO)
MX_RTU_Module_Relay_Value_Get(slot, &u32Val);
u32Val |= (1 << ch);                         // energise
MX_RTU_Module_Relay_Value_Set(slot, u32Val);
```

---

## 12. Event System

Events queue timestamped data when a condition is met — no tight polling needed.

### DI Event
```c
int handle = -1;
MX_RTU_Module_DI_Filter_Set(slot, ch, 1, &filter);  // configure first
MX_RTU_DI_Event_Reset();
MX_RTU_DI_Event_Register(slot, ch, DI_EVENT_TOGGLE_BOTH, &handle);

while (1) {
    UINT32 count = 0;
    MX_RTU_DI_Event_Count(handle, &count);           // MODULE_RW_ERR_OK
    while (count--) {
        UINT32 status = 0; struct Timestamp ts;
        MX_RTU_DI_Event_Get(handle, &status, &ts);
        printf("%d/%d/%d %02d:%02d:%02d.%d\t%d\n",
               ts.year, ts.mon, ts.day,
               ts.hour, ts.min, ts.sec, ts.msec, status);
    }
}
MX_RTU_DI_Event_Unregister(handle);
MX_RTU_DI_Event_Clear(handle);   // call on error mid-drain
```

### AI Event
> **AI event functions return `IO_ERR_OK`, not `MODULE_RW_ERR_OK`.**

```c
int handle = -1;
float level = 4.0f;
UINT32 condition = AI_TC_RTD_EVENT_GREATER;  // _SMALLER / _EQUAL

MX_RTU_AI_Event_Reset();
MX_RTU_AI_Event_Register(slot, ch, level, condition, &handle);

while (1) {
    UINT32 count = 0;
    MX_RTU_AI_Event_Count(handle, &count);   // returns IO_ERR_OK
    while (count--) {
        float fVal = 0; struct Timestamp ts;
        MX_RTU_AI_Event_Get(handle, &fVal, &ts);
    }
}
MX_RTU_AI_Event_Unregister(handle);
MX_RTU_AI_Event_Clear(handle);
```

| Constant | Trigger |
|---|---|
| `AI_TC_RTD_EVENT_GREATER` | value > level |
| `AI_TC_RTD_EVENT_SMALLER` | value < level |
| `AI_TC_RTD_EVENT_EQUAL` | value == level |

### DIO DI Event (separate namespace)
```c
MX_RTU_Module_DIO_DI_Filter_Set(slot, ch, 1, &filter);
MX_RTU_DIO_DI_Event_Reset();
MX_RTU_DIO_DI_Event_Register(slot, ch, eventTrigger, &handle);
// Poll: MX_RTU_DIO_DI_Event_Count / Get / Clear / Unregister
```

---

## 13. FRAM — Ferroelectric RAM (battery-backed, power-fail safe)

```c
// Address range: FRAM_START_ADDRESS to FRAM_END_ADDRESS
UINT32 addr = 0x0;
float  data = 0.0f;

// Read (returns IO_ERR_OK)
MX_RTU_FRAM_Read(addr, sizeof(data), (UINT8*)&data);

// Write
float newVal = 3.14f;
MX_RTU_FRAM_Write(addr, sizeof(newVal), (UINT8*)&newVal);
```

**FRAM survives power loss.** Use it for persisting counters, last-known values, or
config data that must survive a cold restart. Prefer FRAM over SRAM for critical data
as FRAM has no battery to replace.

---

## 14. SRAM — Battery-Backed Static RAM

```c
// Address range: SRAM_START_ADDRESS to SRAM_END_ADDRESS
UINT32 addr = 0x0;
float  data = 0.0f;

MX_RTU_SRAM_Read(addr, sizeof(data), (UINT8*)&data);   // IO_ERR_OK
MX_RTU_SRAM_Write(addr, sizeof(data), (UINT8*)&data);  // IO_ERR_OK
```

**SRAM requires battery backup.** Contents are lost when the battery dies.
Use for temporary state that should survive a soft reboot but not long-term storage.

---

## 15. LED

```c
// LED1 and LED2 each support RED, GREEN, DARK
// Returns MISC_ERR_OK

MX_RTU_LED1_Set(LED_RED);
MX_RTU_LED1_Set(LED_GREEN);
MX_RTU_LED1_Set(LED_DARK);    // off

UINT8 state = 0;
MX_RTU_LED1_Get(&state);      // read current state

MX_RTU_LED2_Set(LED_RED);
MX_RTU_LED2_Set(LED_GREEN);
MX_RTU_LED2_Set(LED_DARK);
MX_RTU_LED2_Get(&state);
```

| Constant | Meaning |
|---|---|
| `LED_RED` | Red |
| `LED_GREEN` | Green |
| `LED_DARK` | Off |

---

## 16. RTC — Real-Time Clock

```c
#include <time.h>
struct rtc_time timeRTC;

// Set RTC from system time
time_t timep; time(&timep);
struct tm *p = localtime(&timep);
memset(&timeRTC, 0, sizeof(timeRTC));
timeRTC.tm_sec  = p->tm_sec;
timeRTC.tm_min  = p->tm_min;
timeRTC.tm_hour = p->tm_hour;
timeRTC.tm_mday = p->tm_mday;
timeRTC.tm_mon  = 1 + p->tm_mon;      // 1-based
timeRTC.tm_year = 1900 + p->tm_year;  // full year
MX_RTU_RTC_Set(&timeRTC);             // MISC_ERR_OK

// Read RTC
memset(&timeRTC, 0, sizeof(timeRTC));
MX_RTU_RTC_Get(&timeRTC);
printf("%d/%d/%d %d:%d:%d\n",
       timeRTC.tm_year, timeRTC.tm_mon, timeRTC.tm_mday,
       timeRTC.tm_hour, timeRTC.tm_min, timeRTC.tm_sec);

// Shell equivalents (also valid from code via system()):
// system("hwclock -r");   // read hardware RTC
// system("hwclock -s");   // sync system time from RTC
// system("hwclock -w");   // sync RTC from system time
```

---

## 17. System Time (POSIX — no Moxa wrapper needed)

```c
#include <time.h>
#include <sys/time.h>

// Read
time_t timep; time(&timep);
struct tm *p;
p = gmtime(&timep);      // UTC
p = localtime(&timep);   // local (set TZ env var: export TZ=CST-8)
printf("%d/%d/%d %d:%d:%d\n",
       1900+p->tm_year, 1+p->tm_mon, p->tm_mday,
       p->tm_hour, p->tm_min, p->tm_sec);

// High-resolution read
struct timeval tv;
gettimeofday(&tv, NULL);   // tv.tv_sec, tv.tv_usec

// Set system time
tv.tv_sec -= 3600;         // example: subtract 1 hour
settimeofday(&tv, NULL);
```

---

## 18. Software Watchdog

```c
struct swtd_setting setting;
UINT32 ackTime = 10 * 1000;  // 10 seconds (range: SOFTWARE_WATCHDOG_MIN_TIME to MAX_TIME)

// Enable watchdog — system reboots if Ack not called within ackTime ms
MX_RTU_SWTD_Enable(ackTime);   // MISC_ERR_OK

// Read current settings
MX_RTU_SWTD_Get_Setting(&setting);
// setting.enable, setting.time

// Critical section — must call Ack at least every ackTime ms
while (1) {
    // ... do work ...
    MX_RTU_SWTD_Ack();   // pet the watchdog — MISC_ERR_OK
}
```

**Use the software watchdog** in any production application to auto-recover from hangs.
If `MX_RTU_SWTD_Ack()` is not called within `ackTime` ms, the system reboots.

---

## 19. Power Supply Monitoring

```c
UINT8 state = 0;

// state == 1: connected, state == 0: disconnected
MX_RTU_Dual_Power1_Get(&state);   // MISC_ERR_OK
if (state == 0) { /* Power1 lost — raise alarm */ }

MX_RTU_Dual_Power2_Get(&state);
if (state == 0) { /* Power2 lost — raise alarm */ }
```

Combine with SMS to send power-fail alerts:
```c
if (state == 0) {
    MX_RTU_SMS_Send_GSM_7bits_Default_Alphabet(
        phoneNum, pinCode, "Power1 disconnect", 17);
}
```

---

## 20. Rotary Switch

```c
UINT32 mode = 0;
MX_RTU_Rotary_Switch_Mode_Get(&mode);   // MISC_ERR_OK
// mode = 0–9 matching the physical switch position

// Example: position 5 → DO on, position 6 → DO off
if (mode == 5)      MX_RTU_Module_DO_Value_Set(doSlot, 0x0001);
else if (mode == 6) MX_RTU_Module_DO_Value_Set(doSlot, 0x0000);
```

---

## 21. Toggle Switch

```c
UINT8 state = 0;
MX_RTU_Toggle_Switch_Get(&state);   // MISC_ERR_OK
// state == 1: position 1 (ON side)
// state == 0: position 2 (OFF side)

UINT32 doVal = 0;
MX_RTU_Module_DO_Value_Get(doSlot, &doVal);
if (state == 1) doVal |=  (1 << ch);   // ON
else            doVal &= ~(1 << ch);   // OFF
MX_RTU_Module_DO_Value_Set(doSlot, doVal);
```

---

## 22. System Information

```c
struct Module_Info moduleInfo;
memset(&moduleInfo, 0, sizeof(moduleInfo));
MX_RTU_Module_Info_Get(slot, &moduleInfo);   // IO_ERR_OK
// moduleInfo.slot, .vendor_id, .product_id, .serial_number
// moduleInfo.hw_version, .fw_version
// moduleInfo.io_info.di_channels / do_channels / dio_channels
//   ai_channels / fast_ai_channels / ao_channels / tc_channels / rtd_channels

UINT32 totalSlot = 0;
MX_RTU_Total_Slots_Get(&totalSlot);   // IO_ERR_OK

UINT32 slotInserted = 0;
MX_RTU_Slot_Inserted_Get(&slotInserted);   // bitmask: bit N = slot N present

UINT32 etherType = 0;
MX_RTU_Ethernet_Adapter_Type_Get(&etherType);
// etherType: ETHERNET_ADAPTER_RJ45 or ETHERNET_ADAPTER_M12

// Hot-plug detection (bitmask — bit N = slot N event)
UINT32 connectState = 0, disconnectState = 0;
MX_RTU_System_Hot_Plug_Connect_Get(&connectState);
MX_RTU_System_Hot_Plug_Disconnect_Get(&disconnectState);

// Version numbers (no return code — direct return value)
UINT32 apiVer  = MX_RTU_API_Version_Get();
UINT32 apiDate = MX_RTU_API_BuildDate_Get();   // e.g. 0x0d03010e = 2013.03.01-14:00
UINT32 sysVer  = MX_RTU_System_Version_Get();
UINT32 sysDate = MX_RTU_System_BuildDate_Get();
```

---

## 23. Timestamp Structure

```c
struct Timestamp { int year, mon, day, hour, min, sec, msec; };

printf("%d/%d/%d %02d:%02d:%02d.%03d\n",
       ts.year, ts.mon, ts.day,
       ts.hour, ts.min, ts.sec, ts.msec);
```

Pass `NULL` as the timestamp argument when you don't need it.

---

## 24. Multi-Channel Array Convention

Most config and read functions take `(slot, startChannel, count, array)`.

```c
// Read 3 AI channels starting at channel 2:
float vals[3];
MX_RTU_Module_AI_Eng_Value_Get(slot, 2, 3, vals, NULL);

// Configure 16 DI channels from channel 0:
UINT8 modes[16];
for (int i = 0; i < 16; i++) modes[i] = DI_MODE_DI;
MX_RTU_Module_DI_Mode_Set(slot, 0, 16, modes);
```

---

## 25. Serial Port

```c
MX_RTU_SerialOpen(slot, port);    // port = PORT1 (=0)
MX_RTU_SerialSetMode(slot, port, RS232_MODE);
MX_RTU_SerialSetSpeed(slot, port, BAUD_RATE_115200);
MX_RTU_SerialSetParam(slot, port, SERIAL_PARITY_NONE,
                      SERIAL_DATA_BITS_8, SERIAL_STOP_BIT_1);
MX_RTU_SerialFlowControl(slot, port, HW_FLOW_CONTROL);
INT32 fd = 0;
MX_RTU_FindFD(slot, port, &fd);   // get underlying fd for select()
MX_RTU_SerialClose(slot, port);

char buf[512]; UINT32 rb=0, wb=0, q=0;
MX_RTU_SerialNonBlockRead(slot, port, sizeof(buf), buf, &rb);
MX_RTU_SerialDataInOutputQueue(slot, port, &q);
MX_RTU_SerialWrite(slot, port, rb, buf, &wb);
MX_RTU_SerialBlockRead(slot, port, sizeof(buf), buf, &rb);  // blocking
```

---

## 26. Modbus TCP

### Master
```c
UINT32 handle=0; UINT8 exCode=0;
MX_RTU_Modbus_Master_Init();
MX_RTU_Modbus_Tcp_Master_Open("192.168.1.100", 502, 1000, &handle);
MX_RTU_Modbus_Tcp_Master_Ioctl(handle, 1, 1000);  // unitID, timeout

float val=0;
MX_RTU_Modbus_Tcp_Master_Read_Input_Regs(handle, 0x0800, 2, (UINT16*)&val, &exCode);
UINT16 reg=0;
MX_RTU_Modbus_Tcp_Master_Read_Holding_Regs(handle, 0x0810, 1, &reg, &exCode);
MX_RTU_Modbus_Tcp_Master_Write_Holding_Regs(handle, 0x0810, 1, &reg, &exCode);
UINT8 coil=0;
MX_RTU_Modbus_Tcp_Master_Read_Coils(handle, 0x0001, 1, &coil, &exCode);
MX_RTU_Modbus_Tcp_Master_Read_Discrete_Inputs(handle, 0x0002, 1, &coil, &exCode);

MX_RTU_Modbus_Tcp_Master_Close(handle);
MX_RTU_Modbus_Master_Uninit();
```

### Slave (callbacks + lifecycle)
```c
// Callbacks must handle big-endian byte order
static int getReg(UINT8 *pData, UINT16 nth, void *pUserData) {
    UINT16 data = ((MY_MAP*)pUserData)->reg;
    pData[nth*2] = (data>>8)&0xFF;  pData[nth*2+1] = data&0xFF;
    return RETURN_OK;
}
static int setReg(UINT8 *pData, UINT16 nth, void *pUserData) {
    ((MY_MAP*)pUserData)->reg = MAKE_WORD(pData[nth*2], pData[nth*2+1]);
    return RETURN_OK;
}
// Groups: MODBUS_INPUT_REGISTER, MODBUS_HOLDING_REGISTER, MODBUS_COIL, MODBUS_INPUT_COIL

UINT32 handle=0;
MX_RTU_Modbus_Tcp_Slave_Init();
MX_RTU_Modbus_Tcp_Slave_Register(502, mapSize, 1000, &handle);
for (int i=0; i<mapSize; i++)
    MX_RTU_Modbus_Tcp_Slave_Add_Entry(handle, map[i].group, map[i].addr,
        &map[i], map[i].pfnRead, map[i].pfnWrite);
MX_RTU_Modbus_Tcp_Slave_Start(handle);
// Cleanup: Stop → Unregister → Uninit
```

---

## 27. Modbus RTU (Serial)

```c
TTY_PARAM p={0};
p.baudrate=BAUD_RATE_115200; p.databits=SERIAL_DATA_BITS_8;
p.flowCtrl=HW_FLOW_CONTROL;  p.mode=RS232_MODE;
p.parity=SERIAL_PARITY_NONE; p.stopbit=SERIAL_STOP_BIT_1;

// Master
MX_RTU_Modbus_Master_Init();
MX_RTU_Modbus_Rtu_Master_Open(slot, port, &p);
MX_RTU_Modbus_Rtu_Master_Read_Input_Regs(slot, port,
    unitID, addr, 2, (UINT16*)&fVal, timeoutMs, &exCode);
MX_RTU_Modbus_Rtu_Master_Close(slot, port);
MX_RTU_Modbus_Master_Uninit();

// Slave — same entry/callback pattern as TCP slave
MX_RTU_Modbus_Rtu_Slave_Init();
MX_RTU_Modbus_Rtu_Slave_Register(slot, PORT1, DEVICE_ID, MAP_SIZE, &p, &handle);
MX_RTU_Modbus_Rtu_Slave_Add_Entry(handle, group, addr, userData, pfnRead, pfnWrite);
MX_RTU_Modbus_Rtu_Slave_Start(handle);
// Cleanup: Stop → Unregister → Uninit
```

---

## 28. CAN / CANopen

```c
int handle=0;
MX_RTU_CanSetBaudrate(slot, port, 125000);
MX_RTU_CanOpen(slot, port, &handle);
MX_RTU_CanNMTSetState(handle, slaveID, CAN_NODE_START);
MX_RTU_CanNMTHeartbeat(handle, slaveID, producerMs, consumerMs);
MX_RTU_CanCyclicSYNCSend(handle, 0x80, timerMs, syncTimes);
UINT8 wData[CAN_MAX_PAYLOAD_LEN]={0xF0};
MX_RTU_CanCyclicPDOSend(handle, pdoNum, slaveID, wData, 1, timerMs, pdoTimes);
UINT8 rData[CAN_MAX_PAYLOAD_LEN]; UINT32 rLen=0, abortCode=0;
MX_RTU_CanSDORead(handle, slaveID, 0x6200, subIdx, rData, &rLen, &abortCode);
CAN_NMT_ERROR_STATUS nmt; CAN_NMT_NODE_GUARDING_STATE guard;
MX_RTU_CanGetNMTError(handle, slaveID, &nmt, &guard);
MX_RTU_CanNMTSetState(handle, slaveID, CAN_NODE_STOP);
MX_RTU_CanClose(handle);
```

---

## 29. GPS / Cellular / SMS / AOPC

### GPS
```c
GPS_DATA gpsData;
MX_RTU_GPS_Start(0);   // 0=passive, 1=active antenna
MX_RTU_GPS_Get_Info(&gpsData);
// .fix, .lat, .lon, .satInUse, .satInView, .time.year/mon/day/hour/min/sec
MX_RTU_GPS_Stop();
```

### Cellular
```c
CheckInfo ac={.autoCheckEnable=1,.pingIntervalS=5,.pingMaxFail=3};
sprintf((char*)ac.pingHostname,"www.google.com");
MX_RTU_Cellular_Modem_Init(pinCode, MODEM_BAND_PH8_AUTO);
MX_RTU_Cellular_Net_Start(apn, user, pass, &ac);
UINT8 ip[16]; MX_RTU_Cellular_Net_IP_Address(ip);
UINT8 rssi;   MX_RTU_Cellular_Modem_RSSI(&rssi);
INT8 imei[16]; MX_RTU_Cellular_Modem_IMEI(imei);
MX_RTU_Cellular_Net_Stop();
```

### SMS
```c
// ASCII — check SMS_ERR_OK
MX_RTU_SMS_Send_GSM_7bits_Default_Alphabet(phoneNum, pinCode, msg, strlen(msg));
// Unicode
INT8 ucs[]={0x00,0x53,0x00,0x4D,0x00,0x53};
MX_RTU_SMS_Send_Ucs2(phoneNum, pinCode, ucs, sizeof(ucs));
```

### AOPC
```c
UINT32 h=0;
MX_RTU_AOPC_Init();
MX_RTU_AOPC_Connect(devName, heartBeatS, ip, port, timeoutMs, &h);
MX_RTU_AOPC_DelAllTag(h, timeoutMs);
TAG tag={0};
sprintf((char*)tag.strTagName,"do-%d-%d",slot,ch);
tag.tagValueType=TAG_TYPE_BOOL;  // TAG_TYPE_FLOAT for AI
tag.tagAccessRight=TAG_ACC_READ_WRITE;
tag.tagCallBack=myWriteCb;       // NULL for read-only
tag.tagQuality=TAG_QUALITY_GOOD;
tag.tagValue=(void*)&myVar;
MX_RTU_AOPC_AddTag(h, &tag, NULL, timeoutMs);
MX_RTU_AOPC_UpdateValue(h, tag.strTagName, (void*)&myVar, &ts, timeoutMs);
// On AOPC_ERR_SOCKET: MX_RTU_AOPC_Reconnect(h, timeoutMs);
MX_RTU_AOPC_DelTag(h, tag.strTagName, timeoutMs);
MX_RTU_AOPC_Disconnect(h);
MX_RTU_AOPC_Uninit();
```

---

## 30. Build System

### Linker flags (never omit any)
```makefile
MOXA_ROOT := /usr/local/arm-linux
LIBDIRS   := -L$(MOXA_ROOT)/lib -L$(MOXA_ROOT)/lib/RTU
MOXA_LIBS := -lmoxa_rtu -lrtu_common -ltag -lmxml -lpthread
RPATH     := -Wl,-rpath,/lib/RTU/
LDFLAGS   := $(LIBDIRS) $(MOXA_LIBS) $(RPATH) -Wl,--allow-shlib-undefined
```

### Standard Makefile
```makefile
CROSS   := arm-linux-gnueabihf-
CXX     := $(CROSS)g++
STRIP   := $(CROSS)strip
INC     := -I$(MOXA_ROOT)/include -I$(MOXA_ROOT)/include/RTU -Iinclude
SRCS    := $(wildcard src/*.cpp)
OBJS    := $(patsubst src/%.cpp,build/%.o,$(SRCS))
TARGET  := my_app
CXXFLAGS_REL := -Wall -Wextra -O2 -std=c++11 $(INC)
CXXFLAGS_DBG := -Wall -Wextra -O0 -ggdb -std=c++11 $(INC)

all: release debug

release: $(OBJS) | build
	$(CXX) $(CXXFLAGS_REL) -o $(TARGET) $^ $(LDFLAGS)
	$(STRIP) $(TARGET)

debug: $(SRCS) | build
	$(CXX) $(CXXFLAGS_DBG) -o $(TARGET)-debug $^ $(LDFLAGS)

build/%.o: src/%.cpp | build
	$(CXX) $(CXXFLAGS_REL) -c $< -o $@

build:
	mkdir -p build

clean:
	rm -rf build $(TARGET) $(TARGET)-debug
```

---

## 31. Critical Rules

1. **`sudo` always** — all Moxa library calls require root.
2. **DO/Relay read-modify-write** — `Value_Set` writes all 32 bits; read first.
3. **Set mode before first use** — `DI_Mode_Set` and `DO_Mode_Set` must be called once.
4. **Single `#include <libmoxa_rtu.h>`** — never include sub-headers.
5. **C++ files: `extern "C" { ... }`** — library is C, not C++.
6. **`make` from project root only** — cross-compiler path is relative.
7. **Persistent storage:** use `/home/moxa/` or `/var/`; OverlayFS loses other writes on reboot.
8. **Log to SD (`/mnt/sd`) not eMMC** — continuous writes shorten flash life.
9. **FRAM > SRAM** for critical data — FRAM needs no battery; SRAM battery can die.
10. **Watchdog in production** — always pet `MX_RTU_SWTD_Ack()` inside your main loop.
11. **AI events: `IO_ERR_OK`** — not `MODULE_RW_ERR_OK`; different namespace.
12. **`usleep(200000)`** — 200ms poll interval used in all official Moxa samples.
13. **`MAKE_WORD(high, low)`** — use in Modbus callbacks; Modbus is big-endian.
14. **Version functions return value directly** — `MX_RTU_API_Version_Get()` has no rc arg.

---

## 32. System Commands

| Command | Purpose |
|---|---|
| `sudo kversion -a` | Full RTU/API/HW/slot version info |
| `setdef` | Factory reset + reboot |
| `upgradehfm <file.hfm>` | Firmware upgrade |
| `hwclock -r` | Read hardware RTC |
| `hwclock -s` | Sync system time from RTC |
| `hwclock -w` | Sync RTC from system time |
| `systemctl start vsftpd.service` | FTP server (for deploy) |
| `df -Th` / `free -ht` | Storage / RAM |
| `sudo journalctl -u <svc>` | Service logs |
