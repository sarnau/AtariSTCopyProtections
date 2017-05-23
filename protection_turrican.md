---
title: "Atari ST Protection: Turrican"
date: "2011-09-01"
categories:
   - "software"
   - "atari st"
   - "reverse engineering"
---

The boot sector starts quite interesting: copying code to address 8 and executing it there. The format of the disk is different, but not special: sector 1 in the track is 512 bytes long, sector 2-6 are each 1024 bytes long. 5632 bytes per track out of possible 6250 bytes, pretty good capacity on a double sided disk. The trick is that the last 1024 byte sector starts at the end of the track and extends for more than 500 bytes over the index marker. The first sector is packed right after the sector at the beginning. The tracks 75..79 are not used. Strangely on the back side Track 79 contains an empty 512 sector, which does not seem to be part of the protection.

Track 7 to 10 contain the actual protection. The test starts in track 7 and tests the protection, if it fails it continous till track 10 to find a valid protection. If it fails on all 4 tracks, it erases the memory till it crashes...

The usual 6 sectors exist in these tracks, but they also contain sector 0 and 16 in different positions (the position is not tested). They are supposed to be each 1024 bytes long, but they only have an address mark plus the sector header and can not be read without a CRC error. The reason for this is interesting and István Fábián figured out what is going on the the actual disk medium! http://www.atari-forum.com/viewtopic.php?f=47&t=19948&start=25

    Track  7: sector order: 5,3,6, 0,16, 1,4,2
    Track  8: sector order:  0,16, 1,4,2,5,3,6
    Track  9: sector order:  5,3,6, 0,16, 1,4,2
    Track 10: sector order:  0,16, 1,4,2,5,3,6

Sector 0 starts with the following data, as you can see it clearly contains the sector header of sector 16! You can also see four weird bytes: 0x14, 0x14, 0x14, 0x00. These bytes are actually 3 sync markers 0xa1 and another sector header 0xfe, but shifted by one bit and the FDC will re-sync to them when trying to read sector 16. And because data bits and clock bits are interleaved, the re-sync will now actually read the clock bits instead of the data bits! The protection therefore reads the data bits via sector 0 (re-sync is disabled after a 0xFE is found for 1024 bytes plus CRC) and then the clock bits via sector 16.

    0000   00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00    ................
    0010   A1 A1 A1 FE 07 00 10 03 BB 21 4E 4E 4E 4E 4E 4E    .........!NNNNNN
    0020   4E 4E 4E 4E 4E 4E 4E 4E 4E 4E 4E 4E 4E 4E 4E 4E    NNNNNNNNNNNNNNNN
    0030   FF FF FF FF FF FF FF FF FF FF FF FE 14 14 14 00    ................
    0040   FF FF FF FF FF FF FF FF FF FF FF FF FF FF FF FF    ................
    0050   80 00 00 00 00 00 00 00 00 00 00 80 00 00 00 00    ................
    0060   00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00    ................

The DMA read code ignores the first 64 bytes, which is a gap and the re-sync for the sector 16. The following 16 bytes have to contain 0xFF bytes and other 32 bytes have to have more than 204 cleared bits (out of 256 possible bits).

Sector 16 (the clock bits) starts with the following data:

    0000   00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00    ................
    0010   40 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00    @...............
    0020   00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00    ................

The first 16 bytes have to contain 0x00 bytes and other 32 bytes have to have more than 204 cleared bits
(out of 256 possible bits), because of clock drift the FDC probably can randomly read a 1 instead of a 0.


``` 68k
copyProtection:
	MOVEM.L   A0-A6/D0-D7,-(A7)
	MOVE.L    #$100000,D0
L056B:SUBQ.L    #1,D0           ;delay
	BNE.S     L056B
	MOVE      SR,-(A7)
	BTST      #5,(A7)
	BNE.S     L056C
	CLR.L     -(A7)
	MOVE.W    #$20,-(A7) 	;SUPER
	TRAP      #1
	MOVE.L    D0,2(A7)
L056C:MOVE      #$2700,SR

	BSR       clearKeyboard

	LEA       $FFFF8000.W,A4  ;ffff8000
	LEA       1549(A4),A3     ;ffff860d
	LEA       1542(A4),A2     ;ffff8606
	LEA       1540(A4),A1     ;ffff8604
	LEA       31233(A4),A0    ;fffffa01

	MOVEQ     #5,D0
	BSR       selectFloppy

	MOVE.W    #$82,(A2)       ;track register
	BSR       fdcDelay
	MOVE.W    (A1),D0
	ANDI.W    #$FF,D0
	MOVE.W    D0,-(A7)        ;current track

	MOVE.W    #$80,(A2)
	MOVEQ     #$01,D2
	BSR       fdcCommand      ;Restore

	MOVEQ     #7,D7           ;start with track 7
L056D:MOVE.W    D7,D2
	BSR       fdcSeek
	BSR       checkSectors
	TST.W     D6
	BPL.S     L056F
	ADDQ.W    #1,D7
	CMPI.W    #10,D7          ;up to track 10
	BLE.S     L056D

	LEA       $80000,A7
L056E:CLR.L     -(A7)           ;erase everything and crash...
	BRA.S     L056E

L056F:MOVE.W    (A7)+,D2        ;current track
	BSR       fdcSeek
L0570:MOVE.W    (A1),D0         ;wait for motor off
	TST.B     D0
	BMI.S     L0570
	MOVEQ     #7,D0
	BSR       selectFloppy    ;deselect drive
	BSR.S     clearKeyboard
	MOVE.W    (A7)+,D0
	CMPI.W    #$20,D0         ;supervisor mode active?
	BNE.S     L0571
	MOVEA.L   (A7)+,A0
	MOVE.W    (A7)+,D0
	MOVE      A7,USP          ;back to user mode
	MOVEA.L   A0,A7
L0571:MOVE      D0,SR
	MOVEM.L   (A7)+,A0-A6/D0-D7
	BRA       copyProtectionReturn

clearKeyboard
	MOVEQ     #$FF,D0
L0573:BTST      #0,$FFFFFC00.W
	BEQ.S     L0574
	MOVE.B    $FFFFFC02.W,D0
	BRA.S     clearKeyboard
L0574:DBF       D0,L0573
	RTS

checkSectors:
	MOVEQ     #4,D6           ;5 tries
checkSectorsTries:
	MOVEQ     #0,D1           ;sector 0
	MOVEQ     #4,D5           ;skip 5 DMA buffer (one empty at the beginning)
	LEA       -128(A7),A5     ;base address
	BSR       readSector
	MOVEQ     #15,D0
L0577:CMPI.B    #$FF,(A5)+      ;16 0xFF bytes have to be at the beginning of the buffer
	BNE.S     checkSectorsFailed
	DBF       D0,L0577
	BSR       countClearedBits
	CMPI.W    #$CC,D0         ;at least 204 cleared bits have to be in the buffer
	BLT.S     checkSectorsFailed

	MOVEQ     #16,D1          ;sector 16
	MOVEQ     #0,D5           ;skip 1 DMA buffer (which is empty anyway)
	LEA       -64(A7),A5      ;base address
	BSR       readSector
	MOVEQ     #15,D0
L0578:TST.B     (A5)+           ;16 0x00 bytes have to be at the beginning of the buffer
	BNE.S     checkSectorsFailed
	DBF       D0,L0578
	BSR.S     countClearedBits
	CMPI.W    #$CC,D0         ;at least 204 cleared bits have to be in the buffer
	BGE.S     checkSectorsSuccess
checkSectorsFailed:
	DBF       D6,checkSectorsTries
checkSectorsSuccess:
	RTS

;count the number of cleared bits in 32 bytes
countClearedBits:
	MOVEQ     #0,D0
	MOVEQ     #31,D1
L057C:MOVEQ     #7,D2
L057D:BTST      D2,(A5)
	BNE.S     L057E
	ADDQ.L    #1,D0
L057E:DBF       D2,L057D
	ADDQ.L    #1,A5
	DBF       D1,L057C
	RTS

readSector:
	MOVE.L    A5,D0
	MOVE.B    D0,(A3)
	LSR.W     #8,D0
	MOVE.B    D0,1547(A4)     ;set the DMA address
	SWAP      D0
	MOVE.B    D0,1545(A4)
	MOVE.W    #$90,(A2)
	MOVE.W    #$190,(A2)      ;read
	MOVE.W    #$90,(A2)
	MOVEQ     #$10,D2
	BSR       fdcWriteD2      ;16*512 byte
	MOVE.W    #$84,(A2)
	MOVE.W    D1,D2           ;sector number
	BSR       fdcWriteD2
	MOVE.W    #$80,(A2)
	BSR       fdcDelay
	MOVE.W    #$80,(A1)       ;read sector

;5+3 DMA buffer = 128 bytes

L0580:MOVEM.L   (A5),D0-D3      ;read original bytes from the buffer
	SUBQ.W    #1,D5           ;5 DMA buffers
	BMI.S     L0582
	MOVE.W    A5,D4
L0581:CMP.B     (A3),D4         ;DMA low unchanged?
	BEQ.S     L0581           ;wait!
	MOVEM.L   D0-D3,(A5)
	LEA       16(A5),A5
	BRA.S     L0580

L0582:MOVEQ     #2,D1
L0583:MOVE.B    (A3),D0
L0584:CMP.B     (A3),D0         ;wait for DMA low to change 3 times
	BEQ.S     L0584
	DBF       D1,L0583

	MOVE.W    #$50,(A2)
	MOVE.W    #$150,(A2)      ;just weird...
	MOVE.W    #$50,(A2)
	BRA       fdcWait

	RTS

selectFloppy:
	MOVE.B    #$E,2048(A4)
	MOVEQ     #$F8,D1
	AND.B     2048(A4),D1
	OR.B      D0,D1
	MOVE.B    D1,2050(A4)
	RTS

fdcSeek:
	MOVE.W    #$86,(A2)
	BSR.S     fdcWriteD2
	MOVE.W    #$80,(A2)
	MOVEQ     #$11,D2
fdcCommand:
	BSR.S     fdcWriteD2
fdcWait:
	BTST      #5,(A0)
	BNE.S     fdcWait
	RTS

fdcWriteD2:
	BSR.S     fdcDelay
	MOVE.W    D2,(A1)

fdcDelay:
	MOVEQ     #$7F,D3
L058B:DBF       D3,L058B
	RTS

copyProtectionReturn:
	RTS
```
