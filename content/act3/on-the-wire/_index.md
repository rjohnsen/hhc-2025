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

#### 1-Wire

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

### 

## Evan Booth mission debrief

> 

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
