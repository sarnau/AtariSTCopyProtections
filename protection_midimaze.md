---
title: "Atari ST Protection: Midi-Maze"
date: "2011-09-01"
categories:
   - "software"
   - "atari st"
   - "reverse engineering"
---

This is the actual call into the protection check:

``` 68k
	JSR       init_aes_window
	TST.W     D0
	BGE       aes_ok
	MOVEQ     #-1,D0
	BRA       init_screen_return
aes_ok:JSR       check_copy_protection
	MOVE.W    D0,protection_flag

	CLR.W     (A7)
	MOVE.L    #str_midimaze_d8a,-(A7)
	MOVE.W    #$3D,-(A7)                  ; Fopen("MIDIMAZE.D8A", 0)
	JSR       _gemdos
```

It is tested later and disables the multi-player game:

``` 68k
check protection in a different place later
	TST.W     protection_flag
	BNE       protOK
	MOVE.W    #-2,machines_online
	MOVE.W    #-2,all_players
protOK:
```

The protection of Midi Maze is quite simple, but effective: Track 0, Sector 0 on Side 0 is read twice and while the first 6 bytes have to contain '100004', the rest of the sector has to change between these two reads. This is done via "weak" or "fuzzy" bits on the disk. Interestingly a patch of the Floprd() function could defeat the protection easily, however because there is no further checksums or encryption replacing the beginning of `check_copy_protection` with `ST D0; RTS` will be more efficient. This was also exactly the patch that was done on a Karstadt Public-Domain disk with an official version of MIDIMAZE on it!

``` 68k
check_copy_protection:
	MOVEM.L   A0-A6/D1-D7,-(A7)
	MOVEQ     #2,D6       ;test drive A and B
	MOVEQ     #2,D5       ;read the sector twice

	MOVE.W    #$19,-(A7) 	;DGETDRV
	TRAP      #1
	ADDQ.W    #2,A7
	MOVE.W    D0,D7
	CMPI.W    #2,D7
	BLT.S     check_copy_protection_rd_loop
	CLR.W     D7          ;if loaded from harddisk, search drive A

check_copy_protection_rd_loop:

	MOVE.W    #1,-(A7)    ;1 sector
	CLR.W     -(A7)       ;Side 0
	CLR.W     -(A7)       ;Track 0
	CLR.W     -(A7)       ;Sector 0
	MOVE.W    D7,-(A7)    ;drive
	CLR.L     -(A7)       ;reserved
	PEA       check_copy_protection_buffer
	MOVE.W    #8,-(A7) 	;FLOPRD
	TRAP      #14
	ADDA.W    #20,A7
	TST.W     D0			;E_OK
	BEQ.S     check_copy_protection_verify
	CMP.W     #-4,D0		;E_CRC
	BEQ.S     check_copy_protection_verify

check_copy_protection_rd_retry:
	SUBQ.W    #1,D6		;out of tries?
	BEQ.S     check_copy_protection_fail	;=> protection fail
	MOVEQ     #2,D5       ;read the sector twice (again)

	; try the potentially other floppy drive
	ADDQ.W    #1,D7       ;try reading drive B
	CMPI.W    #2,D7		;reached C?
	BNE.S     check_copy_protection_rd_loop
	CLR.W     D7          ;then try drive A
	BRA.S     check_copy_protection_rd_loop

check_copy_protection_fail:
	CLR.W     D0          ;fail
	BRA.S     check_copy_protection_return

	DC.B      'RMP  V1.00  31-July-86'

check_copy_protection_verify:
	CLR.W     D0

	; first: compare the first 6 bytes for identity
	LEA       check_copy_protection_buffer,A0
	LEA       check_copy_protection_magic,A1
	CMPM.L    (A0)+,(A1)+
	BNE.S     check_copy_protection_rd_retry
	CMPM.W    (A0)+,(A1)+
	BNE.S     check_copy_protection_rd_retry

	; count the number of set bits in the remaining 506 bytes
	MOVE.W    #505,D1
check_copy_protection_byteloop:
	MOVEQ     #7,D2
	MOVE.B    (A0)+,D3
check_copy_protection_bitloop:
	ASR.W     #1,D3
	BCC.S     check_copy_protection_bitloop2
	ADDQ.W    #1,D0
check_copy_protection_bitloop2:
	DBF       D2,check_copy_protection_bitloop
	DBF       D1,check_copy_protection_byteloop

	SUBQ.W    #1,D5       ;already read twice?
	BEQ.S     check_copy_protection_check     ;yes! => compare the bitcounter

	MOVE.W    D0,D4       ; save the number of bits
	BRA       check_copy_protection_rd_loop	;read sector again

check_copy_protection_check:
	SUB.W     D4,D0       ;same number of bits set?
	BEQ.S     check_copy_protection_rd_retry ;=> that is a fail!

	ST        D0          ;bitcounter change => success
check_copy_protection_return:
	MOVEM.L   (A7)+,A0-A6/D1-D7
	RTS


check_copy_protection_buffer:
	DCB.B     512,0

	ALIGN 4
check_copy_protection_magic:
	DC.B      '100004'
```
