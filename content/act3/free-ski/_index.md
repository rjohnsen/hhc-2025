+++
date = '2025-12-23T15:59:59+01:00'
draft = false
title = 'Free Ski'
weight = 7
+++

![Goose Olivia](/images/act3/goose-space.png)

## Objective


| Difficulty | Description |
| ---------- | ----------- |
| 5/5 | Go to the retro store and help Goose Olivia ski down the mountain and collect all five treasure chests to reveal the hidden flag in this classic SkiFree-inspired challenge. |

## Goose Olivia mission statement

> "HONK! Well hello there! Fancy meeting you here in the Dosis Neighborhood.
> 
> You know, it's the strangest thing... I used to just waddle around the Geese Islands going 'BONK' all day long. Random noises, no thoughts, just vibes. But then something changed, and now here I am—speaking, thinking, wondering how I even got here!
> 
> ----
> 
> HONK! You know what happens to geese in a permanent winter? We can't migrate! And trust me, being stuck in one place forever isn't natural—even for someone who just discovered they can think and talk. Frosty needs to chill out... wait, that's exactly the problem!
> 
> ----
>
> This game looks simple enough, doesn't it? Almost too simple. But between you and me... it seems nearly impossible to win fair and square.
> 
> My advice? If you ain't cheatin', you ain't tryin'. wink
> 
> Now get out there and show that mountain who's boss!

## Solution

This objective shifts from pure technical analysis to pattern recognition and perseverance within a constrained game environment. Although presented as a simple arcade-style challenge, success required careful observation of movement patterns, obstacle behavior, and reward placement rather than brute-force attempts. The exercise mirrors real-world problem solving where progress depends on adapting strategy, recognizing implicit rules, and maintaining focus through repeated failure.

The entire objective was solved using Kali Linux (WSL), using Python 3.13.

### Extract Python Game

First I had to smuggle out the compile Python files from the .exe file. The hints for this objective already had given away that I was dealing with Python due to the tool references. First I had to set up the extractor: 

```bash
git clone https://github.com/extremecoders-re/pyinstxtractor.git
python3 pyinstxtractor/pyinstxtractor.py FreeSki.exe
```

This tool extracted all __pyc__ (compiled Python files) and placed them in a folder called __FreeSki.exe\_extracted__. Once this was done, I moved on to decompilation step. 

### Decompiling using Decompyle++

The hints stated to use Pycdc. In order to use it I had to fetch it from Github and compile it: 

```bash
git clone https://github.com/zrax/pycdc.git
cd pycdc
cmake .
make
```

The decompilation was ... interesting. At first I tried to use __pycdc__, but it did not decompile the __FreeSki.pyc__ file completeley. So I decided to try the ASM route instead: 

```bash
pycdc/pycdas
pycdc/pycdas FreeSki.exe_extracted/FreeSki.pyc > FreeSki.asm
```

This is a snippet from the __FreeSki.asm__ file:

![ASM snippet](/images/act3/act3-freeski-1.png)

### Finding the solution

From here I uploaded the __FreeSki.asm__ to ChatGPT and asked what this file did - and I got this response:

![ChatGPT explaining the meaning of the ASM](/images/act3/act3-freeski-2.png)


Then I asked ChatGPT to make a "one-click-resolver":

```python
#!/usr/bin/env python3
import random
import binascii

MOUNTAIN_WIDTH = 1000

MOUNTAINS = [
    ("Mount Snow",    3586,  b"\x90\x00\x1d\xbc\x17b\xed6S\"\xb0<Y\xd6\xce\x169\xae\xe9|\xe2Gs\xb7\xfdy\xcf5\x98"),
    ("Aspen",        11211,  b"U\xd7%x\xbfvj!\xfe\x9d\xb9\xc2\xd1k\x02y\x17\x9dK\x98\xf1\x92\x0f!\xf1\\\xa0\x1b\x0f"),
    ("Whistler",      7156,  b"\x1cN\x13\x1a\x97\xd4\xb2!\xf9\xf6\xd4#\xee\xebh\xecs.\x08M!hr9?\xde\x0c\x86\x02"),
    ("Mount Baker",  10781,  b"\xac\xf9#\xf4T\xf1%h\xbe3FI+h\r\x01V\xee\xc2C\x13\xf3\x97ef\xac\xe3z\x96"),
    ("Mount Norquay", 6998,  b"\x0c\x1c\xad!\xc6,\xec0\x0b+\"\x9f@.\xc8\x13\xadb\x86\xea{\xfeS\xe0S\x85\x90\x03q"),
    ("Mount Erciyes",12848,  b"n\xad\xb4l^I\xdb\xe1\xd0\x7f\x92\x92\x96\x1bq\xca`PvWg\x85\xb21^\x93F\x1a\xee"),
    ("Dragonmount",  16282,  b"Z\xf9\xdf\x7f_\x02\xd8\x89\x12\xd2\x11p\xb6\x96\x19\x05x))v\xc3\xecv\xf4\xe2\\\x9a\xbe\xb5"),
]

def treasure_locations(name: str, height: int) -> dict[int, int]:
    """
    Reimplements Mountain.GetTreasureLocations():
      random.seed(crc32(name))
      prev_height = height
      prev_horiz = 0
      repeat 5:
        e_delta = randint(200,800)
        h_delta = randint(int(0-e_delta/4), int(e_delta/4))
        locations[prev_height - e_delta] = prev_horiz + h_delta
        prev_height -= e_delta
        prev_horiz += h_delta
    """
    locations: dict[int, int] = {}
    random.seed(binascii.crc32(name.encode("utf-8")))
    prev_height = height
    prev_horiz = 0
    for _ in range(5):
        e_delta = random.randint(200, 800)
        h_delta = random.randint(int(0 - e_delta / 4), int(e_delta / 4))
        row = prev_height - e_delta
        horiz = prev_horiz + h_delta
        locations[row] = horiz
        prev_height = row
        prev_horiz = horiz
    return locations

def treasure_values_in_encounter_order(name: str, height: int) -> list[int]:
    """
    In-game you ski "down" (elevation decreases), so you encounter higher treasure rows first.
    The generation loop already creates them in encounter order, but dict order is an implementation detail,
    so we sort by elevation descending to match gameplay.
    """
    locs = treasure_locations(name, height)
    vals: list[int] = []
    for row in sorted(locs.keys(), reverse=True):
        x_mod = locs[row] % MOUNTAIN_WIDTH
        vals.append(row * MOUNTAIN_WIDTH + x_mod)
    return vals

def decode_flag(encoded_flag: bytes, treasure_vals: list[int]) -> str:
    product = 0
    for t in treasure_vals:
        product = (product << 8) ^ t

    random.seed(product)
    out_chars = []
    for b in encoded_flag:
        r = random.randint(0, 255)
        out_chars.append(chr(b ^ r))
    return "".join(out_chars)

def main():
    for name, height, enc in MOUNTAINS:
        tvals = treasure_values_in_encounter_order(name, height)
        flag = decode_flag(enc, tvals)
        print(f"{name}:")
        print(f"  treasure_vals: {tvals}")
        print(f"  Flag: {flag}")
        print()

if __name__ == "__main__":
    main()
```

Oddly enough, ChatGPT usually takes a nosedive into forbidden territory and refuses to do CTF work. But this time it didn't complain and made a script that actually worked: 

![Output from one click resolver](/images/act3/act3-freeski-3.png)

Passphrase is: __frosty_yet_predictably_random__

Perceived simplicity often masks complexity that only reveals itself through patience and pattern awareness.

## Goose Olivia closing words

After solving, Olivia says:

> Looks like you found your own way down that mountain... and maybe took a few shortcuts along the way. No judgment here—sometimes the clever path IS the right path. Now I'm one step closer to figuring out my own mystery. Thanks for the company, friend!

## Hints

1) Have you ever used [PyInstaller Extractor](https://github.com/extremecoders-re/pyinstxtractor)?
2) Many Python decompilers don't understand Python 3.13, but [Decompyle++](https://github.com/zrax/pycdc) does!