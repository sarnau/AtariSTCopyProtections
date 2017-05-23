---
title: "Atari ST Protection: Maupiti Island"
date: "2011-09-01"
categories:
   - "software"
   - "atari st"
   - "reverse engineering"
---

This game has a custom disk format, which I'll describe here:

The two disks are 80 tracks, two sided.

Track 1, Side 1: 9 Sectors, no file system. The boot sector loads remaining 8 sectors in the track, which contains the "custom BIOS". Interestingly the developer did an off-by-1 error and tries to load 9 sectors starting at sector 2. Therefore the Floprd() call actually fails... But this is not part of the protection.

Track 79, Side 2 on Disk B contains also 9 sectors, which are used to save the current game state.

The rest of these disk doesn't contain __any__ sector. The tracks are read with the read track command and each track has the following format:

* 24 * $4E
* 12 * $00
* 3 * $A1
* $FE
* $07
* track number
* $07
* checksum high byte
* $07
* checksum low byte
* $07
* 5842 data bytes
* $4E repeated till the index mark

The code just searches for two $A1 at the beginning of the track, followed by $FE. It also tests the track number (it has to match the position of the head, bit 7 is set for side 2). The checksum is an unsigned word, which is calculated by adding all 5842 (unsigned) data bytes together. The checksum is also tested.

Because the data bytes can contain any byte $00..$FF it can't be written with a standard FDC, which can't write bytes like $F5 or $F7.

However the FDC has a bug in read track, which tries to sync all the time and certain bytes can trigger the sync, which will destroy the bytes after that. To avoid that a very simple yet efficient method was used: an $07 byte is prefixed any byte that would resync the FDC (Claus Brod wrote a [wonderful article](http://www.clausbrod.de/download/atari/track41.rtf) about this). This is also the reason to have these bytes in the track header!

Well, that is it. Efficient, very large payload in the track and not possible to copy.