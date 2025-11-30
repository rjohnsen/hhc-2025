+++
date = '2025-11-23T13:43:29+01:00'
draft = false
title = 'Going in Reverse'
weight = 7
+++

![Kevin McFarland](/images/act2/kevinmcfarland-avatar.gif)

## Objective

Difficulty: 2/5

Kevin in the Retro Store needs help rewinding tech and going in reverse. Extract the flag and enter it here.

## Kevin McFarland mission statement

> Hello, I'm Kevin (though past friends have referred to me as 'Heavy K'). If you ever hear any one say that philosophy is a useless college degree, don't believe them; it's not. I've arrived where I am at because of it. It just made the path more interesting.
> 
> I have more hobbies than I can keep up with, including Amateur Astronomy, Shortwave Radio, and retro-gaming. Things like backyard observances of distant galaxies, non-internet involved, around the world communications, and those who program for the Atari 2600 still invoke degrees of awe for me.
> 
> One of the most influential books I've read is "Godel, Escher, and Bach" by Douglas Hofstadter. I'm also a bit of a Tolkien fanatic.
> 
> My wife and my daughter are everything; without them, I surely would still be kicking rusty tin cans down the lonely highways of my past.
> 
> ----
> 
> I don't have the details yet, but something's coming up. Check back with me.
> 
> ----
> 
> You know, there's something beautifully nostalgic about stumbling across old computing artifacts. Just last week, I was sorting through some boxes in my garage and came across a collection of 5.25" floppies from my college days - mostly containing terrible attempts at programming assignments and a few games I'd copied from friends.
> 
> Finding an old Commodore 64 disk with a mysterious BASIC program on it? That's like discovering a digital time capsule. The C64 was an incredible machine for its time - 64KB of RAM seemed like an ocean of possibility back then. I spent countless hours as a kid typing in program listings from Compute! magazine, usually making at least a dozen typos along the way.
> 
> The thing about BASIC programs from that era is they were often written by clever programmers who knew how to hide things in plain sight. Sometimes the most interesting discoveries come from reading the code itself rather than watching it execute. It's like being a digital archaeologist - you're not just looking at what the program does, you're understanding how the programmer thought.
> 
> Take your time with this one. Those old-school programmers had to be creative within such tight constraints. You'll know the flag by the Christmas phrase that pays.

## Solving the Challenge

The challenge provides a short BASIC program supposedly acting as a "Commodore 64 security system." The goal is to understand how the password check works and recover the hidden flag. The relevant code is:

```basic
10 REM *** COMMODORE 64 SECURITY SYSTEM ***
20 ENC_PASS$ = "D13URKBT"
30 ENC_FLAG$ = "DSA|auhts*wkfi=dhjwubtthut+dhhkfis+hnkz" ' old "DSA|qnisf`bX_huXariz"
40 INPUT "ENTER PASSWORD: "; PASS$
50 IF LEN(PASS$) <> LEN(ENC_PASS$) THEN GOTO 90
60 FOR I = 1 TO LEN(PASS$)
70 IF CHR$(ASC(MID$(PASS$,I,1)) XOR 7) <> MID$(ENC_PASS$,I,1) THEN GOTO 90
80 NEXT I
85 FLAG$ = "" : FOR I = 1 TO LEN(ENC_FLAG$) : FLAG$ = FLAG$ + CHR$(ASC(MID$(ENC_FLAG$,I,1)) XOR 7) : NEXT I : PRINT FLAG$
90 PRINT "ACCESS DENIED"
100 END
```
---

### Understanding the Password Check

The critical lines are 60–70. For each character in the user-supplied password:

```
Take the ASCII value
XOR it with 7
Convert it back to a character
Compare it to the same index in ENC_PASS$
```

In other words:

```
ENC_PASS$(i) = CHR( ASC(real_password(i)) XOR 7 )
```

Since XOR is reversible:

```
real_password(i) = CHR( ASC(ENC_PASS$(i)) XOR 7 )
```

The program stores:

```
ENC_PASS$ = "D13URKBT"
```

Applying XOR 7 to each character:

| Encoded | ASCII | XOR 7       | Decoded |
| ------- | ----- | ----------- | ------- |
| D       | 68    | 68 ⊕ 7 = 67 | C       |
| 1       | 49    | 49 ⊕ 7 = 54 | 6       |
| 3       | 51    | 51 ⊕ 7 = 52 | 4       |
| U       | 85    | 85 ⊕ 7 = 82 | R       |
| R       | 82    | 82 ⊕ 7 = 85 | U       |
| K       | 75    | 75 ⊕ 7 = 68 | L       |
| B       | 66    | 66 ⊕ 7 = 69 | E       |
| T       | 84    | 84 ⊕ 7 = 83 | S       |

So the correct password is:

```
C64RULES
```

Entering anything else sends execution to line 90 → "ACCESS DENIED".

---

### Decoding the Hidden Flag

If—and only if—the password loop completes without failing, line 85 executes:

```basic
FLAG$ = "" 
FOR I = 1 TO LEN(ENC_FLAG$)
    FLAG$ = FLAG$ + CHR$(ASC(MID$(ENC_FLAG$,I,1)) XOR 7)
NEXT I
PRINT FLAG$
```

This uses the same XOR-7 trick as the password check.

The encoded flag is:

```
DSA|auhts*wkfi=dhjwubtthut+dhhkfis+hnkz
```

Decoding each character with XOR 7 yields:

```
CTF{frost-plan:compressors,coolant,oil}
```

This is the actual secret the program hides.

---

### The Logic Bug

Even when the correct password is entered and the flag is printed, the code falls through to:

```basic
90 PRINT "ACCESS DENIED"
```

The intended system probably meant to branch differently, but the bug doesn’t matter for extraction—the flag is still printed correctly before the failure message.

---

### Final Result

**Password:**

```
C64RULES
```

**Flag:**

```
CTF{frost-plan:compressors,coolant,oil}
```

## Kevin McFarland mission debrief

> Excellent work! You've just demonstrated one of the most valuable skills in cybersecurity - the ability to think like the original programmer and unravel their logic without needing to execute a single line of code.


