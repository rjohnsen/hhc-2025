+++
date = '2025-11-30T11:48:44+01:00'
draft = false
title = 'On the Wire'
weight = 6
+++

![Evan Booth](/images/act3/evanbooth-avatar.gif)

## Objective

Difficulty: 4/5

Help Evan next to city hall hack this gnome and retrieve the temperature value reported by the I²C device at address 0x3C. The temperature data is XOR-encrypted, so you’ll need to work through each communication stage to uncover the necessary keys. Start with the unencrypted data being transmitted over the 1-wire protocol.

## Evan Booth mission statement

> Hey, I'm Evan!
> 
> I like to build things.
> 
> All sorts of things.
> 
> If you aren't failing on some front, consider adjusting your difficulty settings.
> 
> Come find me later - I might need a hand with something.
>
> ---- 
>
> So here's the deal - there are some seriously bizarre signals floating around this area.
>
> Not your typical radio chatter or WiFi noise, but something... different.
> 
> I've been trying to make sense of the patterns, but it's like trying to build a robot hand out of a coffee maker - you need the right approach.
> 
> Think you can help me decode whatever weirdness is being transmitted out there?
> 
> You know what happens to electronics in extreme cold? They fail. All my builds, all my robots, all my weird coffee-maker contraptions—frozen solid. We can't let Frosty turn this place into a permanent deep freeze.

## Solution

The endpoints where extracted by viesing for _WS_ in Network tab:

| Protocol | Endpoint name | Endpoint URL |
| -------- | ------------- | ------------ |
| 1-Wire   | dq            | wss://signals.holidayhackchallenge.com/wire/dq   |
| SPI      | mosi          | wss://signals.holidayhackchallenge.com/wire/mosi |
| SPI      | sck           | wss://signals.holidayhackchallenge.com/wire/sck  |
| I2C      | sda           | wss://signals.holidayhackchallenge.com/wire/sda  |
| I2C      | scl           | wss://signals.holidayhackchallenge.com/wire/scl  |

### 1-Wire

```python
#!/usr/bin/env python3
"""
SANS Holiday Hack Challenge 2025 – "On the Wire" (1-Wire / dq)

What this script does
---------------------
1) Connects to the dq WebSocket endpoint.
2) Records one complete transmission (from marker "reset" to marker "stop").
3) Saves the raw capture to dq_capture.json for reproducibility in a write-up.
4) Decodes 1-Wire data by measuring LOW pulse widths (timing-based decoding).
5) Prints decoded bytes as HEX + ASCII and extracts the XOR key if present.

Why timing matters
------------------
1-Wire does NOT send bits as a simple sequence of 0/1 voltage levels.
Instead, each bit is encoded by how long the line stays LOW during a time slot.

Important implementation detail for this challenge feed:
- The 'presence' marker is on the FALLING edge of the presence pulse (v=0).
  If you start decoding immediately at that record, you will accidentally treat
  the presence pulse as a data bit and the entire decode will be shifted.
  Therefore, we start decoding AFTER the presence pulse ends (next v==1).

Expected outcome
----------------
The decoded ASCII should include an instruction ending with:
"... XOR key: icy"
"""

import json
import websocket
from collections import Counter

# -----------------------------
# Capture control
# -----------------------------

START_MARKER = "reset"
STOP_MARKER = "stop"

recording = False
recording_done = False
recorded = []  # list of dict frames


def on_message(ws, message: str):
    """Capture frames between START_MARKER and STOP_MARKER."""
    global recording, recording_done, recorded

    if recording_done:
        return

    msg = json.loads(message)
    marker = msg.get("marker")

    # Start recording at reset
    if not recording and marker == START_MARKER:
        recording = True
        recorded = [msg]  # include reset frame
        print("START recording (reset)")
        return

    # Ignore everything until we see the start marker
    if not recording:
        return

    # Recording phase
    recorded.append(msg)

    # Stop recording at stop marker
    if marker == STOP_MARKER:
        print(f"STOP recording (stop). Events captured: {len(recorded)}")
        recording = False
        recording_done = True
        ws.keep_running = False
        ws.close()


def on_error(ws, error):
    print("WebSocket error:", error)


def on_close(ws, close_status_code, close_msg):
    print("Connection Closed", close_status_code, close_msg)


# -----------------------------
# 1-Wire decoding helpers
# -----------------------------

def collapse_levels(frames):
    """
    Collapse consecutive identical levels to obtain a transition list.

    Input : list of frames dicts with keys t, v
    Output: list of (t, v) tuples where v flips each entry
    """
    frames = sorted(frames, key=lambda x: x["t"])
    out = []
    last_v = None
    for f in frames:
        t = f["t"]
        v = int(f["v"])
        if last_v is None or v != last_v:
            out.append((t, v))
            last_v = v
    return out


def extract_low_pulse_widths(transitions):
    """
    Extract LOW pulse widths.

    A LOW pulse is a segment where v==0 followed by v==1.
    Width = t_rise - t_fall
    """
    widths = []
    for i in range(len(transitions) - 1):
        t0, v0 = transitions[i]
        t1, v1 = transitions[i + 1]
        if v0 == 0 and v1 == 1:
            widths.append(t1 - t0)
    return widths


def infer_threshold(widths):
    """
    Infer a threshold separating short and long LOW pulses.

    This challenge feed typically has two dominant widths (e.g. 6 and 60).
    We take the two most common widths and set the threshold midway.
    """
    counts = Counter(widths)
    common = [w for (w, _n) in counts.most_common(5)]
    if len(common) < 2:
        raise ValueError("Not enough distinct pulse widths to infer threshold.")
    w_short = min(common[0], common[1])
    w_long = max(common[0], common[1])
    threshold = (w_short + w_long) / 2.0
    return threshold, w_short, w_long, counts


def bits_to_bytes_lsb(bits):
    """
    1-Wire transmits bits LSB-first within each byte.

    bit0 is sent first, then bit1, ..., bit7.
    """
    out = []
    cur = 0
    pos = 0
    for b in bits:
        cur |= (b & 1) << pos
        pos += 1
        if pos == 8:
            out.append(cur)
            cur = 0
            pos = 0
    return bytes(out)


def hexdump_and_ascii(data: bytes):
    """Return a (hex_string, ascii_string) pair."""
    hx = " ".join(f"{b:02x}" for b in data)
    asc = "".join(chr(b) if 0x20 <= b <= 0x7E else "." for b in data)
    return hx, asc


def find_decode_start_index(frames):
    """
    Start decoding AFTER the presence pulse ends.

    The feed marks:
      presence marker at the FALLING edge (v=0)
    So we must move to the next frame where v==1 after that timestamp,
    otherwise the presence pulse width (e.g. 150) will be misread as a data bit.

    If no presence marker exists, fall back to "reset" and start after it returns high.
    """
    frames = sorted(frames, key=lambda x: x["t"])

    def start_after_marker(marker_name: str):
        for i, f in enumerate(frames):
            if f.get("marker") == marker_name:
                t_mark = f["t"]
                # Find next rising level (v==1) AFTER this marker timestamp
                for j in range(i + 1, len(frames)):
                    if frames[j]["t"] > t_mark and int(frames[j]["v"]) == 1:
                        return j
                return i
        return None

    idx = start_after_marker("presence")
    if idx is not None:
        return idx

    idx = start_after_marker("reset")
    if idx is not None:
        return idx

    return 0


def decode_1wire(frames):
    """
    Full pipeline:
    - Determine correct start index (after presence pulse ends)
    - Collapse levels to transitions
    - Extract LOW widths
    - Infer threshold separating short vs long widths
    - Map short->1, long->0 (standard 1-wire convention)
    - Pack bits LSB-first into bytes
    """
    start_idx = find_decode_start_index(frames)
    subset = frames[start_idx:]

    transitions = collapse_levels(subset)
    widths = extract_low_pulse_widths(transitions)

    threshold, w_short, w_long, counts = infer_threshold(widths)

    # Standard mapping for this challenge stream:
    #   short low pulse => bit 1
    #   long  low pulse => bit 0
    bits = [1 if w < threshold else 0 for w in widths]

    decoded = bits_to_bytes_lsb(bits)

    return decoded, {
        "start_idx": start_idx,
        "counts": counts,
        "threshold": threshold,
        "w_short": w_short,
        "w_long": w_long,
        "num_widths": len(widths),
        "num_bits": len(bits),
        "num_bytes": len(decoded),
    }


def extract_printable(data: bytes) -> str:
    """Extract printable ASCII characters for quick human inspection."""
    return "".join(chr(b) for b in data if 0x20 <= b <= 0x7E)


# -----------------------------
# Main
# -----------------------------

def main():
    endpoint = "wss://signals.holidayhackchallenge.com/wire/dq"
    ws = websocket.WebSocketApp(
        endpoint,
        on_message=on_message,
        on_error=on_error,
        on_close=on_close,
    )

    ws.run_forever()

    # Save raw capture for your write-up
    with open("dq_capture.json", "w", encoding="utf-8") as out:
        json.dump(recorded, out, indent=2)

    decoded, meta = decode_1wire(recorded)

    print("\n--- LOW pulse width histogram (top 10) ---")
    for w, n in meta["counts"].most_common(10):
        print(f"width={w:>6}  count={n}")

    print("\n--- Decode parameters ---")
    print(f"decode start index: {meta['start_idx']}")
    print(f"short width: {meta['w_short']}")
    print(f"long  width: {meta['w_long']}")
    print(f"threshold : {meta['threshold']:.2f} (short<thr => bit=1)")
    print(f"bits      : {meta['num_bits']}")
    print(f"bytes     : {meta['num_bytes']}")

    hx, asc = hexdump_and_ascii(decoded)
    print("\n--- Decoded payload ---")
    print("HEX  :", hx)
    print("ASCII:", asc)

    printable = extract_printable(decoded)
    print("\n--- Printable only ---")
    print(printable)

    # Optional: try to pull a key after "key:"
    if "key:" in printable.lower():
        # Split on last occurrence of ':' to be tolerant of other colons earlier
        candidate = printable.split(":")[-1].strip()
        print("\n--- Candidate XOR key ---")
        print(candidate)


if __name__ == "__main__":
    main()
```

This script outputted: 

![1-Wire XOR](/images/act3/act3-onthewire-1.png)

The XOR key is:

```text
icy
```

### SPI

```python
#!/usr/bin/env python3
"""
SANS Holiday Hack Challenge 2025 – "On the Wire" (SPI stage, LIVE)
==================================================================

Goal
----
Capture the LIVE SPI signals from the WebSocket "wires" and decode the byte stream.
Then XOR-decrypt it using the key recovered from the 1-Wire stage (key: "icy").

Why brute force is necessary here
---------------------------------
SPI decoding depends on:
  - Which clock edge you sample on (rising vs falling) -> CPHA/Mode issues
  - Bit order within each byte (MSB-first vs LSB-first)
  - Byte alignment (you might start sampling 1–7 bits too early/late)

If any of these are wrong, XOR output may look like printable garbage
("text-like", but no real words). The fastest way to recover is to brute force:
  - edge ∈ {rising, falling}
  - bit_order ∈ {msb, lsb}
  - bit_shift ∈ {0..7}  (re-align byte boundaries)

This script:
  1) Connects to /wire/sck and /wire/mosi
  2) Captures one broadcast loop (detected by MOSI timestamp wrap-around t dropping)
  3) Decodes using the brute-force matrix above
  4) Ranks candidates by a simple plaintext-likeness score
  5) Prints top candidates and full hexdump for the best candidate

Reproducibility
---------------
- The script saves the raw samples to spi_sck_live.json and spi_mosi_live.json.
"""

from __future__ import annotations

import json
import threading
import time
from collections import defaultdict
from dataclasses import dataclass
from pathlib import Path
from typing import List, Optional, Tuple

import websocket


# -----------------------------
# Endpoints
# -----------------------------
SCK_ENDPOINT  = "wss://signals.holidayhackchallenge.com/wire/sck"
MOSI_ENDPOINT = "wss://signals.holidayhackchallenge.com/wire/mosi"

# XOR key from 1-Wire stage
XOR_KEY = b"icy"

# -----------------------------
# Capture stop conditions
# -----------------------------
MAX_CAPTURE_SECONDS = 20.0   # safety cap
POST_WRAP_GRACE_MS = 150     # let a little extra data arrive after wrap detected


# -----------------------------
# Data structures
# -----------------------------
@dataclass(frozen=True)
class Event:
    t: int
    v: int
    marker: Optional[str] = None


lock = threading.Lock()

wire_frames = defaultdict(list)  # wire_frames["sck"] = [Event...], ["mosi"] = [Event...]

done = False
wrap_detected = False
wrap_wallclock_time: Optional[float] = None

mosi_last_t: Optional[int] = None
mosi_saw_advance = False

last_any_message_time = time.time()


# -----------------------------
# Waveform helpers
# -----------------------------
def collapse_levels(events: List[Event]) -> List[Tuple[int, int]]:
    """
    Collapse consecutive identical v values to obtain transitions (t, v).
    """
    events = sorted(events, key=lambda e: e.t)
    out: List[Tuple[int, int]] = []
    last_v: Optional[int] = None
    for e in events:
        if last_v is None or e.v != last_v:
            out.append((e.t, e.v))
            last_v = e.v
    return out


def extract_edges(clock_transitions: List[Tuple[int, int]], edge: str) -> List[int]:
    """
    Extract timestamps of clock edges (rising or falling) from collapsed transitions.
    """
    times: List[int] = []
    for i in range(1, len(clock_transitions)):
        t_prev, v_prev = clock_transitions[i - 1]
        t_now, v_now   = clock_transitions[i]
        if edge == "rising" and v_prev == 0 and v_now == 1:
            times.append(t_now)
        elif edge == "falling" and v_prev == 1 and v_now == 0:
            times.append(t_now)
    return times


def sample_level_at(transitions: List[Tuple[int, int]], t_edge: int) -> int:
    """
    Return the line level at time t_edge given transitions [(t, v), ...].
    Linear scan is fine for the challenge capture sizes.
    """
    if not transitions:
        return 0
    cur = transitions[0][1]
    for t, v in transitions:
        if t > t_edge:
            break
        cur = v
    return cur


def bits_to_bytes(bits: List[int], bit_order: str) -> bytes:
    """
    Pack bits into bytes.
      - msb: b = (b<<1) | bit
      - lsb: b |= bit << position
    """
    out = bytearray()
    n = len(bits) - (len(bits) % 8)

    if bit_order == "msb":
        for i in range(0, n, 8):
            b = 0
            for bit in bits[i:i+8]:
                b = (b << 1) | (bit & 1)
            out.append(b)

    elif bit_order == "lsb":
        for i in range(0, n, 8):
            b = 0
            for pos, bit in enumerate(bits[i:i+8]):
                b |= (bit & 1) << pos
            out.append(b)
    else:
        raise ValueError("bit_order must be 'msb' or 'lsb'")

    return bytes(out)


def xor_repeat(data: bytes, key: bytes) -> bytes:
    """
    XOR with a repeating key (same operation for encrypt/decrypt).
    """
    return bytes(b ^ key[i % len(key)] for i, b in enumerate(data))


def hexdump_ascii(data: bytes, width: int = 16) -> str:
    """
    Hex + ASCII dump for write-ups.
    """
    def p(x: int) -> str:
        return chr(x) if 0x20 <= x <= 0x7E else "."
    lines = []
    for i in range(0, len(data), width):
        chunk = data[i:i+width]
        hexpart = " ".join(f"{b:02x}" for b in chunk)
        asciipart = "".join(p(b) for b in chunk)
        lines.append(f"{i:04x}  {hexpart:<{width*3}}  {asciipart}")
    return "\n".join(lines)


# -----------------------------
# Brute-force SPI decode scoring
# -----------------------------
def shift_bits(bits: List[int], shift: int) -> List[int]:
    """
    Drop the first 'shift' bits to realign byte boundaries.
    shift is 0..7.
    """
    if shift <= 0:
        return bits
    if shift >= len(bits):
        return []
    return bits[shift:]


def score_plaintext(data: bytes) -> float:
    """
    Heuristic scoring:
    - printable ratio
    - presence of common tokens that appear in HHC hint payloads
    """
    if not data:
        return -1.0

    printable = sum(
        1 for x in data
        if 0x20 <= x <= 0x7E or x in (0x0A, 0x0D, 0x09)
    )
    ratio = printable / len(data)

    s = "".join(chr(x) if 0x20 <= x <= 0x7E else " " for x in data).lower()

    tokens = [
        "key", "xor", "addr", "address", "i2c", "target", "device",
        "use", "decrypt", "payload", "{", "}", ":", "=", "[", "]"
    ]
    hits = sum(1 for t in tokens if t in s)

    # printable ratio dominates; token hits add small bonus
    return ratio + (0.15 * hits)


def try_all_spi_decodes(sck_events: List[Event], mosi_events: List[Event], xor_key: bytes):
    """
    Try combinations:
      - edge: rising/falling
      - bit_order: msb/lsb
      - bit_shift: 0..7

    Returns list of tuples:
      (score, edge, bit_order, shift, raw_bytes, decrypted_bytes)
    sorted by descending score.
    """
    sck_trans = collapse_levels(sck_events)
    mosi_trans = collapse_levels(mosi_events)

    candidates = []
    for edge in ("rising", "falling"):
        edge_times = extract_edges(sck_trans, edge)
        bits = [sample_level_at(mosi_trans, t) for t in edge_times]

        for bit_order in ("msb", "lsb"):
            for shift in range(0, 8):
                bits2 = shift_bits(bits, shift)
                raw = bits_to_bytes(bits2, bit_order)
                dec = xor_repeat(raw, xor_key)
                sc = score_plaintext(dec)
                candidates.append((sc, edge, bit_order, shift, raw, dec))

    candidates.sort(key=lambda x: x[0], reverse=True)
    return candidates


# -----------------------------
# WebSocket callbacks
# -----------------------------
def on_message_sck(ws, message: str):
    global last_any_message_time
    obj = json.loads(message)
    if not isinstance(obj, dict):
        return
    if obj.get("line") != "sck":
        return
    if "t" not in obj or "v" not in obj:
        return

    e = Event(t=int(obj["t"]), v=int(obj["v"]), marker=obj.get("marker"))
    with lock:
        last_any_message_time = time.time()
        wire_frames["sck"].append(e)


def on_message_mosi(ws, message: str):
    global done, wrap_detected, wrap_wallclock_time
    global mosi_last_t, mosi_saw_advance, last_any_message_time

    obj = json.loads(message)
    if not isinstance(obj, dict):
        return
    if obj.get("line") != "mosi":
        return
    if "t" not in obj or "v" not in obj:
        return

    t = int(obj["t"])
    v = int(obj["v"])
    e = Event(t=t, v=v, marker=obj.get("marker"))

    with lock:
        last_any_message_time = time.time()
        wire_frames["mosi"].append(e)

        if mosi_last_t is None:
            mosi_last_t = t
            return

        if t > mosi_last_t:
            mosi_saw_advance = True

        # loop boundary: time drops after we have seen time advance
        if mosi_saw_advance and t < mosi_last_t and not wrap_detected:
            wrap_detected = True
            wrap_wallclock_time = time.time()

        mosi_last_t = t


def on_error(ws, error):
    print("WebSocket error:", error)


def on_close(ws, close_status_code, close_msg):
    # Normal when we stop capture
    pass


# -----------------------------
# Supervisor
# -----------------------------
def supervisor(ws_sck: websocket.WebSocketApp, ws_mosi: websocket.WebSocketApp):
    global done

    started = time.time()
    while True:
        time.sleep(0.05)

        with lock:
            now = time.time()

            # hard cap
            if now - started > MAX_CAPTURE_SECONDS:
                done = True

            # if loop wrap detected, wait a brief grace window then stop
            if wrap_detected and wrap_wallclock_time is not None:
                if (now - wrap_wallclock_time) * 1000.0 >= POST_WRAP_GRACE_MS:
                    done = True

            if done:
                break

    try:
        ws_sck.close()
    except Exception:
        pass
    try:
        ws_mosi.close()
    except Exception:
        pass


def slice_one_loop_by_t_reset(events: List[Event]) -> List[Event]:
    """
    Slice exactly one loop based on the first detected wrap (t decreases).
    Uses the event ordering by t. Works well for the HHC looped streams.
    """
    if not events:
        return events

    ev = sorted(events, key=lambda e: e.t)
    # find first t==0 (common) or a wrap point; prefer t==0 if present
    first0 = next((i for i, e in enumerate(ev) if e.t == 0), None)
    if first0 is None:
        # fallback: find first drop
        for i in range(1, len(ev)):
            if ev[i].t < ev[i-1].t:
                return ev[:i]
        return ev

    # find next t==0 after time advanced
    advanced = False
    for j in range(first0 + 1, len(ev)):
        if ev[j].t != 0:
            advanced = True
            continue
        if advanced and ev[j].t == 0:
            return ev[first0:j]

    return ev[first0:]


# -----------------------------
# Main
# -----------------------------
def main():
    global done

    ws_sck = websocket.WebSocketApp(
        "wss://signals.holidayhackchallenge.com/wire/sck",
        on_message=on_message_sck,
        on_error=on_error,
        on_close=on_close,
    )
    ws_mosi = websocket.WebSocketApp(
        "wss://signals.holidayhackchallenge.com/wire/mosi",
        on_message=on_message_mosi,
        on_error=on_error,
        on_close=on_close,
    )

    t_sck = threading.Thread(target=ws_sck.run_forever, daemon=True)
    t_mosi = threading.Thread(target=ws_mosi.run_forever, daemon=True)
    t_sck.start()
    t_mosi.start()

    sup = threading.Thread(target=supervisor, args=(ws_sck, ws_mosi), daemon=True)
    sup.start()
    sup.join()

    with lock:
        sck_all = list(wire_frames["sck"])
        mosi_all = list(wire_frames["mosi"])

    # Save raw capture for write-up reproducibility
    Path("spi_sck_live.json").write_text(
        "\n".join(json.dumps({"t": e.t, "v": e.v, "marker": e.marker}) for e in sck_all),
        encoding="utf-8",
    )
    Path("spi_mosi_live.json").write_text(
        "\n".join(json.dumps({"t": e.t, "v": e.v, "marker": e.marker}) for e in mosi_all),
        encoding="utf-8",
    )

    if not sck_all or not mosi_all:
        raise SystemExit("Capture failed: missing SCK or MOSI data.")

    # Slice to one loop using MOSI's t-reset behavior
    mosi_loop = slice_one_loop_by_t_reset(mosi_all)

    # Align SCK window roughly to MOSI loop time range
    t_min = min(e.t for e in mosi_loop)
    t_max = max(e.t for e in mosi_loop)
    sck_loop = [e for e in sck_all if t_min <= e.t <= t_max]

    print("\n--- Capture summary ---")
    print(f"MOSI events (all) : {len(mosi_all)}")
    print(f"SCK  events (all) : {len(sck_all)}")
    print(f"MOSI events (loop): {len(mosi_loop)}  t=[{t_min}..{t_max}]")
    print(f"SCK  events (win) : {len(sck_loop)}  t=[{t_min}..{t_max}]")

    # Brute-force decode matrix
    cands = try_all_spi_decodes(sck_loop, mosi_loop, xor_key=XOR_KEY)

    print("\n--- Top SPI decode candidates ---")
    for sc, edge, order, shift, _raw, dec in cands[:8]:
        printable = "".join(chr(b) for b in dec if 0x20 <= b <= 0x7E)
        print(f"\nscore={sc:.3f} edge={edge:7} order={order:3} shift={shift}  bytes={len(dec)}")
        print(printable[:240])

    # Print full dump for the best candidate
    best = cands[0]
    sc, edge, order, shift, raw, dec = best

    print("\n=== BEST CANDIDATE (full dump) ===")
    print(f"score={sc:.3f} edge={edge} bit_order={order} bit_shift={shift}")
    print("\n--- RAW ---")
    print(hexdump_ascii(raw))
    print("\n--- DECRYPTED (XOR key='icy') ---")
    print(hexdump_ascii(dec))

    printable_full = "".join(chr(b) for b in dec if 0x20 <= b <= 0x7E)
    print("\n--- Decrypted printable-only (full) ---")
    print(printable_full)


if __name__ == "__main__":
    main()
```

This script outputted: 

![SPI output](/images/act3/act3-onthewire-2.png)

Decryptred text is:

```text
read and decrypt the I2C bus data using the XOR key: bananza. the temperature sensor address is 0x3
```

### I2C

```python
#!/usr/bin/env python3
"""
SANS HHC 2025 – On the Wire – I2C (LIVE, final)

Known facts:
- Target I²C device (7-bit) = 0x3C
- XOR key = "bananza"
- Expected answer format = xx.xx

Observed bus behavior in your run:
- No READ transactions in this segment (R=0, W=2), so decode WRITE payloads.
- Correct framing is "Mode A" (no ACK bits present in collected address-bit/data-bit stream).
- Decrypted payload for 0x3C contains ASCII temperature (e.g., "32.84").

What this script does:
1) Capture one broadcast loop (timestamp wrap) from SDA/SCL
2) Parse I²C transactions using start/stop markers
3) For last transaction to 0x3C:
   - Convert bits->bytes using Mode A (pack every 8 bits)
   - XOR-decrypt with "bananza"
   - If decrypted text matches r"^\\d{2}\\.\\d{2}$", print SUBMIT: xx.xx
   - Otherwise print diagnostics and (optional) fallback interpretations
"""

import json
import re
import threading
import time
from dataclasses import dataclass
from typing import List, Optional

import websocket


SDA_ENDPOINT = "wss://signals.holidayhackchallenge.com/wire/sda"
SCL_ENDPOINT = "wss://signals.holidayhackchallenge.com/wire/scl"

TARGET_ADDRESS = 0x3C
XOR_KEY = b"bananza"

MAX_CAPTURE_SECONDS = 25.0
POST_WRAP_GRACE_MS = 200

RE_ANSWER = re.compile(r"^\d{2}\.\d{2}$")


@dataclass
class BitEvent:
    t: int
    v: int
    marker: Optional[str]


@dataclass
class TxBits:
    address: int
    rw: int
    bits_after_addr: List[int]


lock = threading.Lock()
events: List[BitEvent] = []

wrap_detected = False
wrap_wallclock_time: Optional[float] = None
last_t: Optional[int] = None
saw_advance = False


def xor_decrypt(data: bytes, key: bytes) -> bytes:
    return bytes(b ^ key[i % len(key)] for i, b in enumerate(data))


def assemble_byte_msb(bits8: List[int]) -> int:
    b = 0
    for bit in bits8:
        b = (b << 1) | (bit & 1)
    return b


def bits_to_bytes_mode_a(bits: List[int]) -> bytes:
    """
    Mode A: ACK bits are NOT present in the collected bitstream.
    Pack sequential 8-bit groups.
    """
    out = bytearray()
    n = len(bits) - (len(bits) % 8)
    for i in range(0, n, 8):
        out.append(assemble_byte_msb(bits[i:i+8]))
    return bytes(out)


def parse_i2c_transactions(events_in: List[BitEvent]) -> List[TxBits]:
    """
    START ... STOP framing.
    Collect only address-bit and data-bit values (0/1).
    The first 8 bits after START are the address byte (MSB-first).
    """
    txs: List[TxBits] = []
    in_tx = False
    bits: List[int] = []

    for e in events_in:
        if e.marker == "start":
            in_tx = True
            bits = []
            continue

        if e.marker == "stop":
            if in_tx and len(bits) >= 8:
                addr_byte = assemble_byte_msb(bits[:8])
                address = addr_byte >> 1
                rw = addr_byte & 1
                txs.append(TxBits(address=address, rw=rw, bits_after_addr=bits[8:]))
            in_tx = False
            bits = []
            continue

        if in_tx and e.marker in ("address-bit", "data-bit"):
            bits.append(e.v)

    return txs


def on_message_sda(ws, message: str):
    global wrap_detected, wrap_wallclock_time, last_t, saw_advance
    obj = json.loads(message)
    if not isinstance(obj, dict) or obj.get("line") != "sda":
        return

    t = int(obj.get("t", 0))
    v = int(obj.get("v", 0))
    marker = obj.get("marker")

    with lock:
        events.append(BitEvent(t=t, v=v, marker=marker))

        if last_t is not None:
            if t > last_t:
                saw_advance = True
            elif saw_advance and t < last_t and not wrap_detected:
                wrap_detected = True
                wrap_wallclock_time = time.time()
        last_t = t


def on_message_scl(ws, message: str):
    # Not used; kept to mirror physical bus capture.
    pass


def supervisor(ws_sda, ws_scl):
    start = time.time()
    while True:
        time.sleep(0.05)
        now = time.time()

        if now - start > MAX_CAPTURE_SECONDS:
            break
        if wrap_detected and wrap_wallclock_time is not None:
            if (now - wrap_wallclock_time) * 1000 >= POST_WRAP_GRACE_MS:
                break

    ws_sda.close()
    ws_scl.close()


def main():
    ws_sda = websocket.WebSocketApp(SDA_ENDPOINT, on_message=on_message_sda)
    ws_scl = websocket.WebSocketApp(SCL_ENDPOINT, on_message=on_message_scl)

    t1 = threading.Thread(target=ws_sda.run_forever, daemon=True)
    t2 = threading.Thread(target=ws_scl.run_forever, daemon=True)
    t1.start()
    t2.start()

    supervisor(ws_sda, ws_scl)

    with lock:
        captured = list(events)

    txs = parse_i2c_transactions(captured)
    target = [tx for tx in txs if tx.address == TARGET_ADDRESS]

    if not target:
        print(f"No transactions seen for 0x{TARGET_ADDRESS:02x}")
        return

    chosen = target[-1]  # value write is typically last
    enc = bits_to_bytes_mode_a(chosen.bits_after_addr)
    dec = xor_decrypt(enc, XOR_KEY)

    # Try ASCII first (this challenge returns ASCII temperature)
    try:
        text = dec.decode("ascii", errors="strict")
    except UnicodeDecodeError:
        text = ""

    text = text.strip()

    print(f"Target 0x{TARGET_ADDRESS:02x} transactions: {len(target)} (using last)")
    print(f"Chosen TX rw={chosen.rw} bits_after_addr={len(chosen.bits_after_addr)}")
    print(f"Decrypted ASCII: {repr(text)}")
    print(f"Decrypted HEX  : {dec.hex()}")

    if RE_ANSWER.fullmatch(text):
        print(f"\nSUBMIT: {text}")
        return

    # If not a direct xx.xx string, provide guidance for manual inspection.
    print("\nDecrypted output did not match xx.xx. Inspect the ASCII/HEX above.")
    print("If the value is embedded, search the ASCII for a pattern like \\d\\d\\.\\d\\d.")


if __name__ == "__main__":
    main()

```


This script outputted: 

![I2C output](/images/act3/act3-onthewire-3.png)

The value I was looking for is:

```text
32.84
```

## Evan Booth mission debrief

> Nice work! You cracked that signal encoding like a pro.
>
> Turns out the weirdness had a method to it after all - just like most of my builds!

## Hints

### On Rails

1. Connect to the captured wire files or endpoints for the relevant wires.
2. Collect all frames for the transmission (buffer until inactivity or loop boundary).
3. Identify protocol from wire names (e.g., dq → 1-Wire; mosi/sck → SPI; sda/scl → I²C).
4. Decode the raw signal:

* Pulse-width protocols: locate falling→rising transitions and measure low-pulse width.
* Clocked protocols: detect clock edges and sample the data line at the specified sampling phase.

5. Assemble bits into bytes taking the correct bit order (LSB vs MSB).
6. Convert bytes to text (printable ASCII or hex as appropriate).
7. Extract information from the decoded output — it contains the XOR key or other hints for the next stage.

1. Repeat Stage 1 decoding to recover raw bytes (they will appear random).
2. Apply XOR decryption using the key obtained from the previous stage.
3. Inspect decrypted output for next-stage keys or target device information.

* Multiple 7-bit device addresses share the same SDA/SCL lines.
* START condition: SDA falls while SCL is high. STOP: SDA rises while SCL is high.
* First byte of a transaction = (7-bit address << 1) | R/W. Extract address with address = first_byte >> 1.
* Identify and decode every device’s transactions; decrypt only the target device’s payload.
* Print bytes in hex and as ASCII (if printable) — hex patterns reveal structure.
* Check printable ASCII range (0x20–0x7E) to spot valid text.
* Verify endianness: swapping LSB/MSB will quickly break readable text.
* For XOR keys, test short candidate keys and look for common English words.
* If you connect mid-broadcast, wait for the next loop or detect a reset/loop marker before decoding.
* Buffering heuristic: treat the stream complete after a short inactivity window (e.g., 500 ms) or after a full broadcast loop.
* Sort frames by timestamp per wire and collapse consecutive identical levels before decoding to align with the physical waveform.

### Protocols

**Key concept - Clock vs. Data signals:**

* Some protocols have separate clock and data lines (like SPI and I2C)
* For clocked protocols, you need to sample the data line at specific moments defined by the clock
* The clock signal tells you when to read the data signal

**For 1-Wire (no separate clock):**

* Information is encoded in pulse widths (how long the signal stays low or high)
* Different pulse widths represent different bit values
* Look for patterns in the timing between transitions

**For SPI and I2C:**

* Identify which line is the clock (SCL for I2C, SCK for SPI)
* Data is typically valid/stable when the clock is in a specific state (high or low)
* You need to detect clock edges (transitions) and sample data at those moments

**Technical approach:**

* Sort frames by timestamp
* Detect rising edges (0→1) and falling edges (1→0) on the clock line
* Sample the data line's value at each clock edge

### Bits and Bytes

_Critical detail - Bit ordering varies by protocol:_

**MSB-first (Most Significant Bit first):**

* SPI and I2C typically send the highest bit (bit 7) first
* When assembling bytes: byte = (byte << 1) | bit_value
* Start with an empty byte, shift left, add the new bit

**LSB-first (Least Significant Bit first):**

* 1-Wire and UART send the lowest bit (bit 0) first
* When assembling bytes: byte |= bit_value << bit_position
* Build the byte from bit 0 to bit 7

**I2C specific considerations**

* Every 9th bit is an ACK (acknowledgment) bit - ignore these when decoding data
* The first byte in each transaction is the device address (7 bits) plus a R/W bit
* You may need to filter for specific device addresses

**Converting bytes to text:**

String.fromCharCode(byte_value)  // Converts byte to ASCII character

### Garbage?

If your decoded data looks like gibberish:

* The data may be encrypted with XOR cipher
* XOR is a simple encryption: encrypted_byte XOR key_byte = plaintext_byte
* The same operation both encrypts and decrypts: plaintext XOR key = encrypted, encrypted XOR key = plaintext

**How XOR cipher works:**

```text
function xorDecrypt(encrypted, key) {
  let result = "";
  for (let i = 0; i < encrypted.length; i++) {
    const encryptedChar = encrypted.charCodeAt(i);
    const keyChar = key.charCodeAt(i % key.length);  // Key repeats
    result += String.fromCharCode(encryptedChar ^ keyChar);
  }
  return result;
}
```

**Key characteristics:**

* The key is typically short and repeats for the length of the message
* You need the correct key to decrypt (look for keys in previous stage messages)
* If you see readable words mixed with garbage, you might have the wrong key or bit order

**Testing your decryption:**

* Encrypted data will have random-looking byte values
* Decrypted data should be readable ASCII text
* Try different keys from messages you've already decoded

### Structure

What you're dealing with:

* You have access to WebSocket endpoints that stream digital signal data
* Each endpoint represents a physical wire in a hardware communication system
* The data comes as JSON frames with three properties: line (wire name), t (timestamp), and v (value: 0 or 1)
* The server continuously broadcasts signal data in a loop - you can connect at any time
* This is a multi-stage challenge where solving one stage reveals information needed for the next

**Where to start:**

* Connect to a WebSocket endpoint and observe the data format
* The server automatically sends data every few seconds - just wait and collect
* Look for documentation on the protocol types mentioned (1-Wire, SPI, I2C)
* Consider that hardware protocols encode information in the timing and sequence of signal transitions, not just the values themselves
* Consider capturing the WebSocket frames to a file so you can work offline
