+++
date = '2025-11-23T13:42:25+01:00'
draft = false
title = 'Retro Recovery'
weight = 1
+++

![XXX](/images/act2/markdevito-avatar.gif)

## Objective

Difficulty: 2/5

Join Mark in the retro shop. Analyze his disk image for a blast from the retro past and recover some classic treasures.

## Mark DeVito mission statement

> I am an avid collector of things of the past. I love old technology. I love how it connects us to the past. Some remind me of my dad, who was an engineer on the Apollo 11 mission and helped develop the rendezvous and altimeter radar on the space craft.
>
> If you ever get into collecting things like vintage computers, here is a tip. Never forget to remove the RIFA capacitors from vintage computer power supplies when restoring a system. If not they can pop and fill the room with nasty smoke.
>
> I love vintage computing, it’s the very core of where and when it all began. I still enjoy writing programs in BASIC and have started re-learning Apple II assembly language. I started writing code in 1982 on a Commodore CBM.
> 
> Sometimes it is the people no one can imagine anything of who do the things no one can imagine. - Alan Turing
> 
> You never forget your first 8-bit system.
> 
> ----
> 
> There's something in the works. Come see me later and we'll talk.
> 
> ----
> 
> While Kevin and I were cleaning up the Retro Store, we found this FAT12 floppy disk image, must have been under this arcade machine for years. These disks were the heart of machines like the Commodore 64. I am so glad you can still mount them on a modern PC.
> 
> When I was a kid we shared warez by hiding things as deleted files.
> 
> I remember writing programs in BASIC. So much fun! My favorite was Star Trek.
> 
> The beauty of file systems is that 'deleted' doesn't always mean gone forever.
> 
> Ready to dive into some digital archaeology and see what secrets this old disk is hiding?
> 
> Go to Items in your badge, download the floppy disk image, and see what you can find!

## Solution

### Running file utility

```bash
file floppy.img
```

Output

```bash
floppy.img: DOS/MBR boot sector, code offset 0x3c+2, OEM-ID "mkfs.fat", root entries 224, sectors 2880 (volumes <=32 MB), sectors/FAT 9, sectors/track 18, reserved 0x1, serial number 0x9c01e8ae, unlabeled, FAT (12 bit), followed by FAT
```

### Running binwalk

```bash
binwalk floppy.img
```

Output

```bash
DECIMAL       HEXADECIMAL     DESCRIPTION
--------------------------------------------------------------------------------
133776        0x20A90         Copyright string: "Copyright Microsoft Corporation 1982-1988."
143535        0x230AF         Copyright string: "Copyright (C) Microsoft Corp 1982-1987. All rights reserved."
222225        0x36411         Copyright string: "Copyright Microsoft Corporation, 1982-1988."
258196        0x3F094         Copyright string: "Copyright (C) Microsoft Corp 1983-1988.  %s."
327297        0x4FE81         Copyright string: "Copyright (C) Microsoft Corp 1983-1988.  %s."
343826        0x53F12         Copyright string: "Copyright (C) Microsoft Corp 1983-1988.  All rights reserved."
614815        0x9619F         Copyright string: "Copyright Microsoft Corporation, 1985-1988."
```

### Running fstat

```bash
fsstat floppy.img
```

Output:

```bash
FILE SYSTEM INFORMATION
--------------------------------------------
File System Type: FAT12

OEM Name: mkfs.fat
Volume ID: 0x9c01e8ae
Volume Label (Boot Sector): NO NAME
Volume Label (Root Directory):
File System Type Label: FAT12

Sectors before file system: 0

File System Layout (in sectors)
Total Range: 0 - 2879
* Reserved: 0 - 0
** Boot Sector: 0
* FAT 0: 1 - 9
* FAT 1: 10 - 18
* Data Area: 19 - 2879
** Root Directory: 19 - 32
** Cluster Area: 33 - 2879

METADATA INFORMATION
--------------------------------------------
Range: 2 - 45782
Root Directory: 2

CONTENT INFORMATION
--------------------------------------------
Sector Size: 512
Cluster Size: 512
Total Cluster Range: 2 - 2848

FAT CONTENTS (in sectors)
--------------------------------------------
93-93 (1) -> EOF
94-284 (191) -> EOF
285-436 (152) -> EOF
437-506 (70) -> EOF
507-642 (136) -> EOF
643-671 (29) -> EOF
672-690 (19) -> EOF
691-1235 (545) -> EOF
1236-1236 (1) -> EOF
```

### Listing normal files

```bash
fls -r floppy.img
```

Output:

```bash
r/r * 6:        all_i-want_for_christmas.bas
d/d 8:  qb45
+ r/r 1189:     BC.EXE
+ r/r 1190:     BRUN45.EXE
+ r/r 1191:     LIB.EXE
+ r/r 1192:     LINK.EXE
+ r/r 1193:     MOUSE.COM
+ r/r 1196:     PACKING.LST.txt
+ r/r 1197:     QB.EXE
+ r/r 1198:     QB.INI
r/r * 10:       .all_i-want_f
v/v 45779:      $MBR
v/v 45780:      $FAT1
v/v 45781:      $FAT2
V/V 45782:      $OrphanFiles
```

### Listing deleted files

```bash
fls -rd floppy.img
```

Output

```bash
r/r * 6:        all_i-want_for_christmas.bas
r/r * 10:       .all_i-want_f
```

### Extracting files

```bash
icat floppy.img 6 > all_i-want_for_christmas.bas
icat floppy.img 10 > all_i-want_f
```

### Inspecting file

```bash
strings all_i-want_f
```

This file only contained this, which wasnt much interesting:

```bash
b0nano 7.2
mark
arcade
all_i-want_for_christmas.bas
```

However this revealed something interesting: 

```bash
strings all_i-want_for_christmas.bas
```

Output:

![Solution](/images/act2/act2-retro-recovery-1.png)

It apperas that this file is actually a Star Trek game. On line `211` we find a Base64 encoded string: `bWVycnkgY2hyaXN0bWFzIHRvIGFsbCBhbmQgdG8gYWxsIGEgZ29vZCBuaWdodAo=` 

This string decodes to (the solution) `merry christmas to all and to all a good night`

## Mark DeVito mission debrief

> Excellent work! You've successfully recovered that deleted file and decoded the hidden message.
>
> Sometimes the old ways are the best ways. Vintage file systems never truly forget what they've seen. Play some Star Trek... it actually works.