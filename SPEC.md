# The `.pitlog` file format

**Version 1.1 – draft.** Until 1.0 is final, details may still change.

A `.pitlog` file records one session of a PitLogger: vehicle data read via OBD-II and vehicle
profiles (manufacturer requests, see [PitLogger/vehicle-profiles](https://github.com/PitLogger/vehicle-profiles)), GPS positions,
trouble codes and events such as markers set by the driver. This document describes the format
completely, so that anyone can write a reader without other sources.

## 1. Design goals

1. **Self-describing.** A file names and explains its own channels. A reader does not need to know
   in advance which values a vehicle or an add-on module provides.
2. **Robust against power loss.** In a car the supply can disappear at any moment. Data is written
   in blocks with a checksum; a half-written block at the end is detected and dropped, everything
   before it stays valid.
3. **Raw data included.** OBD values are stored both converted and as the raw bytes received.
   A wrong conversion formula can be fixed later in the converter, and values the logger could
   not decode are not lost.
4. **Extensible.** Unknown record types and unknown trailing fields are skipped. Old readers keep
   working with newer files of the same major version.
5. **Simple.** Writable by a microcontroller with little memory, readable in a few hundred lines
   in any language.

## 2. Conventions

| Item | Rule |
|---|---|
| Byte order | Little-endian, unless stated otherwise |
| `u8` `u16` `u32` | Unsigned integers with 1, 2, 4 bytes |
| `i32` `i64` | Signed integers (two's complement) |
| `f32` | IEEE 754 single precision |
| `str` | `u8` length followed by that many bytes of UTF-8, no terminator (max. 255 bytes) |
| `t` | `u32` milliseconds since the **session origin** (see 7.1) |
| Offsets | Counted from the start of the structure, in bytes |

## 3. File layout

```
+-------------+---------+---------+-----+---------+
| File header | Block 0 | Block 1 | ... | Block n |
+-------------+---------+---------+-----+---------+
```

A file consists of a 16-byte file header followed by any number of blocks. There is no index and
no footer: a file can be read up to wherever writing stopped.

### 3.1 File header (16 bytes)

| Offset | Type | Field | Value |
|---|---|---|---|
| 0 | 8 bytes | magic | `89 50 49 54 4C 4F 47 0A` (`\x89PITLOG\n`) |
| 8 | `u8` | version_major | `1` |
| 9 | `u8` | version_minor | `1` (`0` in files of firmware before vehicle profiles) |
| 10 | `u16` | flags | `0` (reserved, readers ignore unknown bits) |
| 12 | `u32` | session_id | random number, identical in every block of this file |

The first magic byte is not ASCII and the last one is a line feed, so files damaged by text-mode
transfers are recognised immediately (same idea as PNG).

### 3.2 Block

| Offset | Type | Field | Meaning |
|---|---|---|---|
| 0 | 4 bytes | block_magic | `50 4C 42 4B` (`PLBK`) |
| 4 | `u32` | session_id | same as in the file header |
| 8 | `u32` | sequence | 0 for the first block, +1 for each following block |
| 12 | `u32` | payload_length | number of payload bytes (max. 1 048 576) |
| 16 | bytes | payload | records, see 4 |
| 16 + payload_length | `u32` | crc32 | CRC-32 over bytes 0 … 15 + payload_length (header and payload) |

**CRC-32** is the common variant used by zlib, PNG and Ethernet: polynomial `0x04C11DB7`
(reflected `0xEDB88320`), initial value `0xFFFFFFFF`, final XOR `0xFFFFFFFF`.
Check value: CRC-32 of the ASCII string `123456789` is `0xCBF43926`.

A record never spans two blocks.

## 4. Records

The payload of a block is a sequence of records:

| Offset | Type | Field |
|---|---|---|
| 0 | `u8` | type |
| 1 | `u16` | body_length |
| 3 | bytes | body |

**Extensibility rules**

- A reader **skips** records of unknown type using `body_length`.
- If `body_length` is larger than the fields a reader knows, the reader **ignores the extra
  bytes**. Minor versions may only append fields at the end of a body.
- If `body_length` is smaller than the fields of the current version, the missing fields are
  treated as absent. Writers of version 1.0 always write all fields listed here.

### 4.1 Overview

| Type | Name | Content |
|---|---|---|
| `0x01` | SESSION_INFO | device, firmware, why and how the recording started |
| `0x02` | TIME_SYNC | link between `t` and real time (UTC) |
| `0x03` | VEHICLE | VIN, protocol, control unit, supported PIDs |
| `0x10` | CHANNEL_DEF | definition of a measurement channel |
| `0x11` | SAMPLE | one measured value of a channel |
| `0x12` | RAW_OBD | raw OBD-II response bytes |
| `0x13` | RAW_DIAG | raw answer to a diagnostic request (since 1.1) |
| `0x20` | GPS_FIX | position, speed, course, quality |
| `0x30` | DTC | trouble codes of one kind at one point in time |
| `0x40` | EVENT | start, stop, marker, connection changes … |
| `0xE0`–`0xEF` | – | reserved for add-on modules (to be specified with the module SDK) |
| `0xF0`–`0xFF` | – | experimental, never used in released firmware |

All other values are reserved for future versions.

### 4.2 `0x01` SESSION_INFO

Exactly once, as the first record of block 0.

| Type | Field | Meaning |
|---|---|---|
| `u32` | t | normally 0 |
| 6 bytes | device_id | MAC address of the logger |
| `str` | firmware | e.g. `PitLogger 2026.10.1` |
| `u8` | start_source | who started the recording, see 5.1 |
| `u32` | preroll_ms | length of pre-trigger data contained in the file (0 = none) |

### 4.3 `0x02` TIME_SYNC

As soon as the logger knows the real time, e.g. from GPS. May occur several times (to correct
drift); the most recent one before a given `t` applies.

| Type | Field | Meaning |
|---|---|---|
| `u32` | t | session time of the measurement |
| `i64` | unix_ms | UTC milliseconds since 1970-01-01 at that moment |
| `u8` | source | 1 = GPS, 2 = phone/browser, 3 = entered manually |

Real time of any record: `unix_ms + (t_record − t)`. Without TIME_SYNC only relative times are known.

### 4.4 `0x03` VEHICLE

| Type | Field | Meaning |
|---|---|---|
| `u32` | t | |
| `str` | vin | vehicle identification number, empty if not available |
| `str` | protocol | OBD protocol as reported by the adapter |
| `str` | ecu_name | Mode 09 PID 0A, empty if not available |
| `str` | calibration_id | Mode 09 PID 04, empty if not available |
| `u8` | obd_standard | raw value of Mode 01 PID 1C (SAE J1979), 0 = unknown |
| `u8` | fuel_type | raw value of Mode 01 PID 51 (SAE J1979), 0 = unknown |
| 32 bytes | pids_mode01 | bitmap of supported Mode 01 PIDs: bit 7 of byte 0 = PID 0x00, bit 0 of byte 31 = PID 0xFF |

Manufacturer, model and model year are deliberately not stored: they are derived from the VIN by
the converter, which can improve that decoding over time.

### 4.5 `0x10` CHANNEL_DEF

Defines a channel. Must appear **before the first SAMPLE** of that channel (in the same or an
earlier block). Channel numbers are assigned per file, starting at 1.

| Type | Field | Meaning |
|---|---|---|
| `u16` | channel | channel number within this file |
| `u8` | source | origin of the value, see 5.3 |
| `u16` | source_id | depends on source, see 5.3 |
| `u8` | value_type | 1 = `f32` (only value in 1.0) |
| `str` | name | English display name, e.g. `Engine speed` – see note below |
| `str` | unit | e.g. `1/min`, `km/h`, `°C` |
| `u8` | decimals | sensible number of decimals for display |
| `f32` | min | typical range for scaling charts |
| `f32` | max | |
| `str` | key | since 1.1: stable identifier – OBD-II: the PID in hex (`0C`), vehicle profile: the signal `id` (`I30_STEER_ANGLE`), add-on module: `M`, the last four hex digits of the module's MAC address, `.` and the value's number (`MBD3C.0`) |
| `str` | metric | since 1.1: standard meaning such as `speed`, `engineSpeed`, `stateOfCharge`; empty if none |
| `u8` | label_count | since 1.1: number of display texts that follow |
| (`i32`, `str`) × label_count | labels | since 1.1: display text for a raw value, e.g. `0` → `No`, `1` → `Yes` for two-state values or enumerations; the SAMPLE then holds the raw value |

Channel names and units are always **English**, independent of the language the logger's user
interface is set to. Readers that want to show other languages translate by `source` and
`source_id` (e.g. OBD-II `0x010C`), not by the name, and fall back to the stored name.

### 4.6 `0x11` SAMPLE

| Type | Field | Meaning |
|---|---|---|
| `u32` | t | time of measurement |
| `u16` | channel | channel number |
| `f32` | value | converted value in the channel's unit |

### 4.7 `0x12` RAW_OBD

Raw response to a request. Written in addition to the SAMPLE for the same `t` – or alone, if the
logger has no conversion for that PID.

| Type | Field | Meaning |
|---|---|---|
| `u32` | t | time of measurement |
| `u8` | mode | requested service, e.g. `0x01` |
| `u8` | pid | requested PID |
| bytes | data | data bytes of the response **without** the response service and PID bytes (rest of the body) |

Example: the adapter answers `41 0C 0C B2` → `mode = 0x01`, `pid = 0x0C`, `data = 0C B2`.

### 4.7a `0x13` RAW_DIAG (since 1.1)

Raw answer to a diagnostic request of a vehicle profile (e.g. ReadDataByIdentifier `22`). Written
once per answer, together with the SAMPLEs of the signals decoded from it.

| Type | Field | Meaning |
|---|---|---|
| `u32` | t | time of the answer |
| `u32` | tx | CAN ID the request was sent to |
| `u32` | rx | CAN ID of the answer, 0 = not known |
| `u8` | request_len | number of request bytes |
| bytes | request | the request, service first, e.g. `22 01 04` |
| bytes | answer | the complete answer, service first, e.g. `62 01 04 …` (rest of the body) |

### 4.8 `0x20` GPS_FIX

| Type | Field | Unit |
|---|---|---|
| `u32` | t | |
| `i32` | latitude | 1e-7 degrees (WGS84) |
| `i32` | longitude | 1e-7 degrees |
| `i32` | altitude | centimetres above mean sea level |
| `u16` | speed | 0.01 km/h |
| `u16` | course | 0.01 degrees, 0 = north, clockwise |
| `u8` | satellites | number used for the fix |
| `u8` | fix_type | 0 = no fix, 2 = 2D, 3 = 3D |
| `u16` | hdop | 0.01 |

### 4.9 `0x30` DTC

The trouble codes of one kind as read at time `t`. Written at session start and whenever the
set changes. An empty list means "read, no codes".

| Type | Field | Meaning |
|---|---|---|
| `u32` | t | |
| `u8` | kind | 1 = stored (Mode 03), 2 = pending (Mode 07), 3 = permanent (Mode 0A) |
| `u8` | mil | malfunction indicator lamp: 0 = off, 1 = on, 255 = unknown |
| `u8` | count | number of codes |
| `count` × 2 bytes | codes | each code as the two bytes sent by the vehicle, **big-endian** |

The codes keep the byte order of the OBD response so they can be copied unchanged.
Decoding: bits 15–14 → letter (`P`, `C`, `B`, `U`), bits 13–12 → first digit, then three hex
digits. `01 33` → `P0133`, `C1 23` → `U0123`.

### 4.10 `0x40` EVENT

| Type | Field | Meaning |
|---|---|---|
| `u32` | t | |
| `u16` | event | see 5.2 |
| `u8` | source | who caused it, see 5.1 |
| `str` | text | optional detail, e.g. the rule that fired; may be empty |

## 5. Enumerations

### 5.1 Sources (`start_source`, EVENT `source`)

| Value | Source |
|---|---|
| 0 | system / unknown |
| 1 | web interface |
| 2 | smartphone app |
| 3 | automatic rule |
| 4 | add-on module (e.g. button) |
| 5 | automatically at power-up |

### 5.2 Events

| Value | Event |
|---|---|
| 1 | recording started (moment of the trigger; earlier data is pre-roll) |
| 2 | recording stopped |
| 3 | marker set by the user |
| 4 | OBD adapter connected |
| 5 | OBD adapter disconnected |
| 6 | vehicle stopped answering |
| 7 | session statistics reset |
| 8 | supply voltage lost, file closed early |

### 5.3 Channel sources

| `source` | Meaning | `source_id` |
|---|---|---|
| 1 | OBD-II | `mode << 8 \| pid`, e.g. `0x010C` for engine speed |
| 2 | GPS | 1 = speed, 2 = altitude, 3 = course (values also in GPS_FIX; for charts and rules) |
| 3 | add-on module | the value's number within its module; the module and value are identified by `key` (since 1.1) |
| 4 | PitLogger itself | 1 = supply voltage, 2 = internal temperature |
| 5 | vehicle profile | 0; the signal is identified by `key` (since 1.1) |

## 6. Reading

1. Check the magic. If `version_major` is greater than supported, stop with a clear message.
2. Read blocks one after another. A block is valid if its magic and `session_id` match,
   `payload_length` ≤ 1 048 576, the block is complete and the CRC is correct.
3. At the **first invalid block** reading stops; all records before are valid. If the file
   simply ends inside a block, this is the normal result of a power loss and not an error.
4. *Optional recovery:* a reader may search the rest of the file for the next `PLBK` with the
   same `session_id` and continue there, reporting the gap (detectable from `sequence`).
5. Within a block, process records in order; skip unknown types.

## 7. Writing

### 7.1 Session origin and pre-roll

`t = 0` is the time of the **oldest data in the file**. If the logger keeps a pre-roll buffer
(e.g. the last 30 s before a rule fired), the buffered data starts at `t = 0`, `preroll_ms` in
SESSION_INFO gives its length and an EVENT 1 marks the trigger. `t` therefore never becomes
negative. Sessions longer than 49 days (`u32` overflow) must be split into several files.

### 7.2 Recommendations

- Write a block every **5 seconds** or earlier when the buffer reaches **32 KiB**; keep blocks
  below **64 KiB**. After each block, flush so the file system updates the file size.
- Block 0 starts with SESSION_INFO, followed by VEHICLE, the CHANNEL_DEFs known so far and the
  current DTC lists. Channels discovered later are defined in the block where they first occur.
- Store a SAMPLE together with the RAW_OBD it was converted from.
- File names: `NNNNN.pitlog` with a running number. Date and time come from TIME_SYNC, so the
  name does not depend on a clock.

### 7.3 Size

SAMPLE (13 bytes) plus RAW_OBD (about 11 bytes) take about 24 bytes per measurement. At around
9 values per second that is roughly **0.75 MB per hour**, plus about 0.1 MB per hour for GPS at
1 Hz.

## 8. Example

[`examples/example.pitlog`](examples/example.pitlog) (424 bytes) contains a session with two
blocks: SESSION_INFO, VEHICLE, two CHANNEL_DEFs, stored trouble codes `P0133` and `P0301`,
samples with raw data, a TIME_SYNC, a GPS fix and two events.

Beginning of the file:

```
0000  89 50 49 54 4c 4f 47 0a 01 00 00 00 34 12 ed 5e   file header: magic, v1.0, flags 0,
                                                         session_id 0x5EED1234
0010  50 4c 42 4b 34 12 ed 5e 00 00 00 00 10 01 00 00   block 0: PLBK, session_id, seq 0,
                                                         payload 272 bytes
0020  01 1f 00 00 00 00 00 cc ba 97 0a 1c 54 0f 50 69   SESSION_INFO (31 bytes): t 0,
0030  74 4c 6f 67 67 65 72 20 30 2e 31 2e 30 01 30 75   device cc:ba:97:0a:1c:54, "PitLogger 0.1.0",
                                                         source 1 (web), pre-roll 30000 ms …
```

A SAMPLE and the matching RAW_OBD (engine speed 812.5 1/min at t = 1000 ms on channel 1):

```
11 0a 00  e8 03 00 00  01 00  00 20 4b 44     SAMPLE: t 1000, channel 1, f32 812.5
12 08 00  e8 03 00 00  01 0c  0c b2           RAW_OBD: t 1000, mode 01, PID 0C, data 0C B2
```

## 9. Version history

| Version | Changes |
|---|---|
| 1.0 (draft) | first version; channel names in English, manufacturer and model derived from the VIN by readers |
| 1.1 (draft) | vehicle profiles: channel source 5, CHANNEL_DEF fields `key`, `metric`, `labels`; record RAW_DIAG; values of add-on modules (channel source 3, `key`). Readers of 1.0 skip the new fields and records |
