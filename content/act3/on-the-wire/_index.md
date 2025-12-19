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

![Solution](/images/act1/visual-network-thinger-1.png)

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
