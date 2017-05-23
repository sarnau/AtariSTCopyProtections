---
title: "Atari ST Protection: Brataccas"
date: "2011-09-01"
categories:
   - "software"
   - "atari st"
   - "reverse engineering"
---

The protection is also simple: track 81, sector 20 has to contain at least 71 times 'JAKE' at the beginning. It then reads sector 1 in track 5, which has to have a CRC error in the data block (the FDC status has bit 3 set, but not bit 4).

Interesting, but not as interesting as the code in the boot sector. Michael K. Glover had some fun with it:


This is the executed code:

``` 68k
$FFFF8800 = $0E, $FFFF8802 = $05    ;Select Disk A, Side 0
$FFFF8606 = $80, $FFFF8604 = $03    ;CMD Restore
WAIT BTST #0,$FA01                  ;wait for completion
$FFFF8606 = $86, $FFFF8604 = $0A    ;DATA = 10
$FFFF8606 = $80, $FFFF8604 = $10    ;CMD Seek to track 10
WAIT BTST #0,$FA01                  ;wait for completion
$FFFF8606 = $84, $FFFF8604 = $0A    ;SECTOR = 10
$FFFF8606 = $82, $FFFF8604 = $45    ;TRACK = $45
$FFFF8609 = $00, $FFFF860B = $04, $FFFF860D = $80,  ;DMA ADDRESS = $000480
$FFFF8606 = $90, $FFFF8606 = $190, $FFFF8606 = $90  ;DMA read
$FFFF8604 = $01                     ;1 sector (512 bytes)
$FFFF8606 = $80, $FFFF8604 = $80    ;CMD Read Sector
WAIT BTST #0,$FA01                  ;wait for completion
JMP (A6) => $480


	TEXT
$0000 BRA.S     L0000
$0002 DC.B      'M.Glover',$00
$000B DC.B      $00,$02, $02, $01,$00, $02, $70,$00, $D0,$02, $F8, $05,$00, $09,$00, $01,$00, $00,$00
$001E DC.B      $00,$00, $00,$00, $00,$00, $00,$00, $00,$00, $00,$00
	DC.B      $00,$04, $00,$00, $00,$00,$80,$00
	DC.B      'TOS     IMG',$00

L0000:MOVE      #$2700,SR
	BRA       L000E

	DC.B      'This beauty by Michael K. Glover.(JAKE)'
	DC.B      'I dont know about "NEUTER BOOTER",but you need real balls to check this out!!!',0

	//        D0    D4    D5    D6    D7    A1    A2    A3    A4    A5
L0001:DC.W      $0000,$0001,$0001,$0000,$001A,$8800,$8608,$FA01,$8609,$000F

L0002:
$00CC EXT.W     D0
$00CE MOVEM.W   L0001(PC,D0.W),A1-A5/D4-D7/D0
$00D4 MOVEM.L   L0005(PC,D7.W),A6-A7            ; L0007
$00DA LEA       L0007+3(PC,A5.L),A5

L0003:
$00DE MOVE.B    (A5)+,D6
$00E0 MOVE.B    D6,(A1)+            ;select floppy and side
$00E2 ADDQ.L    #1,A1
$00E4 DBF       D5,L0003

L0004:
$00E8 JSR       L000C(PC,D7.W)  ;L000D
$00EC JSR       L0004(PC,D7.W)  ;L0008
$00F0 JSR       L000B(PC,D7.W)  ;L000D
$00F4 JSR       L0006(PC,D7.W)  ;L0008
$00F8 JSR       L0009(PC,D7.W)  ;L000D

;set DMA address
$00FC MOVE.W    (A5)+,D2
$00FE MOVE.B    D2,(A4)+
$0100 ADDQ.L    #1,A4
L0005:
$0102 DBF       D7,$00FC

;$90,$190,$90 => FFFF8606
$0106 ADDQ.L    #2,A0
$0108 MOVE.W    (A5)+,(A0)
$010A DBF       D4,$0108
$010E MOVEQ     #1,D0
$0110 JSR       L0010(PC,D7.W)  ;$0167 + D7 (-1) => $0166 = L000D
L0006 EQU $0112
$0114 JSR       L0003(PC,D7.W)  ;$00DE + $46 => $0124 = L0008
$0118 BRA       L000F

	//        A6    A7
L0007:
$011C DC.W      $0000,$0480
$0120 DC.W      $0007,$0000

L0008:
$0124 BTST      D6,(A3)
$0126 BNE.S     L0008
$0128 MOVE.W    (A5)+,D7
L0009:
$012A MOVE.W    (A5)+,D0
L000A:
$012C RTS

L000X:
$012E DC.B      $0E,$05
$0130 DC.W      $0080,$0003,$003C   ;=> L0008
$0136 DC.W      $0028,$0001         ;=> L000D, D0=1

$013A DC.W      $0086,$000A
L000B:
$013E DC.W      $0080,$0010,$0012   ;L0008

	DC.W      $003C,$0001         ;L000D
	DC.W      $0084,$000A
L000C:DC.W      $0082,$0045,$0002
	DC.W      $0000,$0004,$0080
	DC.W      $0090,$0190,$0090
	DC.W      $0001,$0080,$0080,$0046

L000D:
$0166 MOVEA.L   A2,A0
L0010 EQU       *-1
	MOVE.W    (A5)+,-(A0)
	BSR.S     L000A
	MOVE.W    (A5)+,-(A0)
	DBF       D0,L000D
	MOVE.W    (A5)+,D7
	RTS

	DC.B      'So there it is. What a little beauty Eh?'

L000E:MOVE      SR,D0
	ORI.W     #0,D0
	ORI.W     #6,D1
	ORI.W     #8,D1
	ORI.W     #0,D1
	BRA       L0002

	DC.B      'The code in this game is by Dave,Jake and Phil!!'

L000F:JMP       (A6)

	; checksum
	DC.B      $90,$EE,$00,$00,$00,$00,$00,$00
	DCB.W     9,0
```
