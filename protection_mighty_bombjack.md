---
title: "Atari ST Protection: Mighty Bombjack"
date: "2011-09-01"
categories:
   - "software"
   - "atari st"
   - "reverse engineering"
---

Finally an interesting protection!

The boot sector is executable and simply loads the code into memory plus reading the boot sector (moving the head back to track 0). After loading the code the fun begins (skip further down...)

``` 68k
	TEXT
	BRA.S     L0000

	DC.B      $00,$00,$00,$00,$00,$00,$00,$00
	DC.B      $00,$00,$02,$02,$01,$00,$02,$70
	DC.B      $00,$20,$03,$00,$05,$00,$0A,$00
	DC.B      $01,$00,$00,$00,$00,$00

L0000:MOVE.L    #$230,D0        ;track 56, sector 1
	MOVE.L    #101,D1         ;$CA00 bytes
	MOVEA.L   $432.L,A0       ;_membot = Start of TPA (user memory)
	BSR       LoadSectors
	BNE.S     L0000
	LSL.L     #8,D1
	ADD.L     D1,D1           ;buf += 512 * sectors
	ADDA.L    D1,A0
	MOVEQ     #1,D1           ;$200 bytes
	BSR       LoadSectors     ;Track 1, Sector 1
	MOVE.L    $432.L,-(A7)    ;_membot = Start of TPA (user memory)
	RTS

LoadSectors:
	MOVEM.L   A0-A2/D1-D5,-(A7)
	MOVE.L    D0,D3           ;startsector
	MOVE.L    D1,D4           ;number of sectors
	MOVEQ     #-1,D0
LoadSectorsLoop:
	MOVE.L    D3,D0
	DIVU      #10,D0          ;10 sectors per track
	SWAP      D0
	MOVEQ     #0,D1
	MOVE.W    D0,D1
	ADDQ.W    #1,D1           ;sectno
	SWAP      D0
	ANDI.L    #$FF,D0         ;trackno
	MOVE.L    D0,D5
	MOVEQ     #11,D2
	SUB.L     D1,D2           ;number of sectors
	CMP.L     D2,D4
	BGE.S     LoadSectorsR    ;< remaining sectors
	MOVE.L    D4,D2           ;then read the remaining sectors
LoadSectorsR:
	BSR       Floprd
	BNE.S     LoadSectorsError
	ADD.L     D2,D3           ;sector += read sectors
	SUB.L     D2,D4           ;number of sectors -= read sectors
	LSL.L     #8,D2
	LSL.L     #1,D2           ;buf += read sectors * 512
	ADDA.L    D2,A0
	TST.L     D4              ;remaining sectors left?
	BGT.S     LoadSectorsLoop
	MOVEQ     #0,D0
LoadSectorsError:
	MOVEM.L   (A7)+,A0-A2/D1-D5
	TST.L     D0
	RTS

Floprd:MOVEM.L   A0/D1-D2,-(A7)
	MOVE.W    D2,-(A7)        ;count
	CLR.W     -(A7)           ;sideno
	MOVE.W    D0,-(A7)        ;trackno
	MOVE.W    D1,-(A7)        ;sectno
	CLR.W     -(A7)           ;devno = 'A'
	CLR.L     -(A7)           ;filler
	MOVE.L    A0,-(A7)        ;buf
	MOVE.W    #8,-(A7) 	;FLOPRD
	TRAP      #14             ;Floprd( void *buf, int32_t filler, int16_t devno, int16_t sectno, int16_t trackno, int16_t sideno, int16_t count )
	LEA       20(A7),A7
	TST.L     D0
	MOVEM.L   (A7)+,A0/D1-D2
	RTS

	DS.W      3
	DC.B      'Copylock ST (c)1988-90 Rob Northen Computing, U.K. All Rights Reserved.',0
	DS.W      123
	DC.W      $027F
	END

Ok, the code starts by entering supervisor mode via an illegal instruction, then pushing a trace exception handler onto the stack, which decrypts code on the fly. A *lot* of code. All this code is just there to annoy people trying to trace it by hand. Later it reaches the actual protection, which I'll publish here. It again uses trace to decrypt the next opcode (actually 8 bytes) and encrypting the last one again. As a little bonus it executes a subroutine if the "EXG D7,D7" opcode is executed. Fun, but useless: if you made it that far, this is not doing anything... Another thing: LINE-A, all interrupts and traps have to point to ROM, so does GPI7 and printer code (used to enter the debugger). If not, they point to a memory erase routine and a call into the reset of the ROM.

But now to the actual thing: the protection. 3 Parts:

- Via "read track" the first 450 bytes are read and the first sector address header is searched. It has to be a 512 byte sector on the first side and the first sector has to be equal to "(11 + (track % 5) * -2) % 10". If the routine fails, it returns -1, otherwise the current track. Funny thing: on track 0, sector 1 is just fine and because the boot sector was read at the end of the loading code, that's where the floppy head is located. 3 tries to get it done.
- Via Read Sector it reads sector 5 and sector 6 of track 0. It simply times the time it takes to read the sector and expects sector 6 to be at least >1% slower than sector 5. On the floppy it seems to be more than 3%, so 1% is safe.
- The first 16 bytes of sector 6 have to contain the string "Rob Northen Comp". A checksum over this string plus the following 8 bytes results in the serial number of the disk, which is specific for the application.

Yes, that's it for the protection. It continues with trace decoding and then launches the app. The following code is the heart of the protection, cleaned up and decoded.



018CAE  48E7C0C0                  MOVEM.L   A0-A1/D0-D1,-(A7)
018CB2  43F90000015C              LEA       $15C.L,A1
018CB8  233C00044ED0              MOVE.L    #$44ED0,-(A1)
018CBE  233CFFFA2078              MOVE.L    #$FFFA2078,-(A1)
018CC4  233CFFFF51C8              MOVE.L    #$FFFF51C8,-(A1)
018CCA  233C3FF948E0              MOVE.L    #$3FF948E0,-(A1)
018CD0  233C0000303C              MOVE.L    #$303C,-(A1)
018CD6  233C41F90010              MOVE.L    #$41F90010,-(A1)
018CDC  233C46FC2700              MOVE.L    #$46FC2700,-(A1)

		;erase all memory and call reset
		000140: 46fc 2700                 MOVE      #$2700,SR
		000144: 41f9 0010 0000            LEA       $100000,A0
		00014a: 303c 3ff9                 MOVE.W    #$3FF9,D0
		00014e: 48e0 ffff                 MOVEM.L   D0-D7/A0-A7,-(A0)
		000152: 51c8 fffa                 DBF       D0,*-$4 [$14E]
		000156: 2078 0004                 MOVEA.L   $4.w,A0
		00015a: 4ed0                      JMP       (A0)

018CE2  23C90000042A              MOVE.L    A1,$42A.L
018CE8  23FC3141592600000426      MOVE.L    #$31415926,$426.L

018CF2  41F900000028              LEA       $28.L,A0            ;Line 1010 Emulator (LineA)
018CF8  61000038                  BSR       56(PC)               resetIfNotInROM
018CFC  41F900000060              LEA       $60.L,A0            ;all interrupts and traps!
018D02  6100002E            L0000:BSR       46(PC)               resetIfNotInROM
018D06  B1FC000000C0              CMPA.L    #$C0,A0
018D0C  66F4                      BNE.S     -12(PC)              L0000
018D0E  41F90000013C              LEA       $13C.L,A0           ;GPI7 - Monochrome Detect
018D14  6100001C                  BSR       28(PC)               resetIfNotInROM
018D18  41F90000050A              LEA       $50A.L,A0           ;prv_lst
018D1E  61000012                  BSR       18(PC)               resetIfNotInROM
018D22  41F900000512              LEA       $512.L,A0           ;prv_aux
018D28  61000008                  BSR       8(PC)                resetIfNotInROM
018D2C  4CDF0303                  MOVEM.L   (A7)+,A0-A1/D0-D1
018D30  6018                      BRA.S     24(PC)               L0003

resetIfNotInROM
018D32  2018                      MOVE.L    (A0)+,D0
018D34  6712                      BEQ.S     18(PC)               L0002
018D36  028000FFFFFF              ANDI.L    #$FFFFFF,D0
018D3C  B0BC00400000              CMP.L     #$400000,D0
018D42  6C04                      BGE.S     4(PC)                L0002
018D44  2149FFFC                  MOVE.L    A1,-4(A0)
018D48  4E75                L0002:RTS

018D4A  7000                L0003:MOVEQ     #0,D0
018D4C  7201                      MOVEQ     #1,D1
018D4E  48E73FF0                  MOVEM.L   A0-A3/D2-D7,-(A7)
018D52  7400                      MOVEQ     #0,D2
018D54  E690                      ROXR.L    #3,D0
018D56  E292                      ROXR.L    #1,D2
018D58  E790                      ROXL.L    #3,D0
018D5A  1400                      MOVE.B    D0,D2
018D5C  2601                      MOVE.L    D1,D3
018D5E  7E01                      MOVEQ     #1,D7
018D60  7C00                      MOVEQ     #0,D6
018D62  47FA03A6                  LEA       934(PC),A3           buffer512Bytes
018D66  3F390000043E              MOVE.W    $43E.L,-(A7)
018D6C  50F90000043E              ST        $43E.L
018D72  610000A0            L0004:BSR       160(PC)              doCheckDiskInterleave
018D76  6B0C                      BMI.S     12(PC)               L0005
018D78  61000036                  BSR       54(PC)               checkSectorsReturnMagic
018D7C  4A86                      TST.L     D6                   ;fail? 2nd try!
018D7E  6604                      BNE.S     4(PC)                L0005
018D80  6100002E                  BSR       46(PC)               checkSectorsReturnMagic
018D84  610000A0            L0005:BSR       160(PC)              seekToPreviousTrack
018D88  4A86                      TST.L     D6
018D8A  660E                      BNE.S     14(PC)               L0006
018D8C  4A82                      TST.L     D2
018D8E  6A0A                      BPL.S     10(PC)               L0006
018D90  5202                      ADDQ.B    #1,D2
018D92  02020001                  ANDI.B    #1,D2
018D96  51CFFFDA                  DBF       D7,-38(PC)           L0004
018D9A  6100005E            L0006:BSR       94(PC)               eraseBuffer
018D9E  33DF0000043E              MOVE.W    (A7)+,$43E.L
018DA4  3202                      MOVE.W    D2,D1               ;d1.w = drive no. key disk was found in
018DA6  2006                      MOVE.L    D6,D0               ;d0.l = serial no. 0=key disk not found
018DA8  4CDF0FFC                  MOVEM.L   (A7)+,A0-A3/D2-D7
018DAC  60000560                  BRA       1376(PC)             L002D

checkSectorsReturnMagic:
018DB0  598F                      SUBQ.L    #4,A7
018DB2  303C0005                  MOVE.W    #5,D0               ;Sector 5 in Track 0 (normal sector)
018DB6  610000AC                  BSR       172(PC)              fdcMeasureSectorTiming
018DBA  2E80                      MOVE.L    D0,(A7)
018DBC  303C0006                  MOVE.W    #6,D0               ;Sector 6 in Track 0 (slower sector, ~3%)
018DC0  610000A2                  BSR       162(PC)              fdcMeasureSectorTiming
018DC4  2217                      MOVE.L    (A7),D1
018DC6  9081                      SUB.L     D1,D0               ;delta between 5 and 6
018DC8  6B2C                      BMI.S     44(PC)               L000A
018DCA  C0FC0064                  MULU      #100,D0
018DCE  80C1                      DIVU      D1,D0               ;convert delta into percent
018DD0  B03C0001                  CMP.B     #1,D0               ;1 percent or less is not good enough!
018DD4  6D20                      BLT.S     32(PC)               L000A

018DD6  7000                      MOVEQ     #0,D0
018DD8  7203                      MOVEQ     #3,D1
018DDA  204B                      MOVEA.L   A3,A0
018DDC  9098                L0008:SUB.L     (A0)+,D0            ;checksum over first 16 bytes of sector 6 = "Rob Northen Comp"
018DDE  51C9FFFC                  DBF       D1,-4(PC)            L0008
018DE2  B0BCB34C4FDC              CMP.L     #$B34C4FDC,D0       ;has to be a specific value!
018DE8  660C                      BNE.S     12(PC)               L000A
018DEA  2C00                      MOVE.L    D0,D6
018DEC  7201                      MOVEQ     #1,D1
018DEE  DC98                L0009:ADD.L     (A0)+,D6            ;secret ID over the next 8 bytes
018DF0  4846                      SWAP      D6
018DF2  51C9FFFA                  DBF       D1,-6(PC)            L0009

018DF6  588F                L000A:ADDQ.L    #4,A7
018DF8  4E75                      RTS

eraseBuffer
018DFA  204B                      MOVEA.L   A3,A0
018DFC  323C00FF                  MOVE.W    #$FF,D1
018E00  203C00D4C742              MOVE.L    #$D4C742,D0
018E06  C0FC0011            L000C:MULU      #$11,D0
018E0A  5280                      ADDQ.L    #1,D0
018E0C  30C0                      MOVE.W    D0,(A0)+
018E0E  51C9FFF6                  DBF       D1,-10(PC)           L000C
018E12  4E75                      RTS

doCheckDiskInterleave:
018E14  61000028                  BSR       40(PC)               selectFloppy
018E18  610000D0                  BSR       208(PC)              checkDiskSectorInterleave
018E1C  3A00                      MOVE.W    D0,D5                => current track number
018E1E  6B04                      BMI.S     4(PC)                L000E
018E20  61000234                  BSR       564(PC)              fdcRestore
018E24  4E75                L000E:RTS

seekToPreviousTrack:
018E26  3005                      MOVE.W    D5,D0                current track number set?
018E28  6B0E                      BMI.S     14(PC)               L0010
018E2A  61000210                  BSR       528(PC)              fdcSeekD0
018E2E  4A86                      TST.L     D6
018E30  6722                      BEQ.S     34(PC)               deselectFloppy
018E32  4A03                      TST.B     D3
018E34  671E                      BEQ.S     30(PC)               deselectFloppy
018E36  4E75                      RTS
018E38  61000250            L0010:BSR       592(PC)              fdcFloppyDeselect
018E3C  4E75                      RTS

selectFloppy:
018E3E  1002                      MOVE.B    D2,D0
018E40  5200                      ADDQ.B    #1,D0
018E42  E308                      LSL.B     #1,D0
018E44  0A000007                  EORI.B    #7,D0
018E48  02000007                  ANDI.B    #7,D0
018E4C  7201                      MOVEQ     #1,D1
018E4E  61000244                  BSR       580(PC)              fdcFloppySelect
018E52  4E75                      RTS

deselectFloppy:
018E54  223C00005949              MOVE.L    #$5949,D1
018E5A  103C0007                  MOVE.B    #7,D0
018E5E  61000234                  BSR       564(PC)              fdcFloppySelect
018E62  4E75                      RTS

fdcMeasureSectorTiming:
018E64  48E72000                  MOVEM.L   D2,-(A7)
018E68  33FC008400FF8606          MOVE.W    #$84,$FF8606.L
018E70  6100027C                  BSR       636(PC)              fdcWriteD0
018E74  610000EE                  BSR       238(PC)              fdcDMAReadAddress
018E78  303C0001                  MOVE.W    #1,D0
018E7C  61000270                  BSR       624(PC)              fdcWriteD0
018E80  33FC008000FF8606          MOVE.W    #$80,$FF8606.L
018E88  43F900FF860B              LEA       $FF860B.L,A1
018E8E  7200                      MOVEQ     #0,D1
018E90  240B                      MOVE.L    A3,D2
018E92  2F3C66F64E75              MOVE.L    #$66F64E75,-(A7)
018E98  2F3C00FFFA01              MOVE.L    #$FFFA01,-(A7)
018E9E  2F3C08390005              MOVE.L    #$8390005,-(A7)
018EA4  2F3CB44066F6              MOVE.L    #$B44066F6,-(A7)
018EAA  2F3C01090000              MOVE.L    #$1090000,-(A7)
018EB0  2F3C02005281              MOVE.L    #$2005281,-(A7)
018EB6  2F3C06820000              MOVE.L    #$6820000,-(A7)
018EBC  2F3CB44067F8              MOVE.L    #$B44067F8,-(A7)
018EC2  2F3C01090000              MOVE.L    #$1090000,-(A7)
018EC8  2F3C00FF8604              MOVE.L    #$FF8604,-(A7)
018ECE  2F3C33FC0080              MOVE.L    #$33FC0080,-(A7)
018ED4  2F3C007C0700              MOVE.L    #$7C0700,-(A7)
018EDA  244F                      MOVEA.L   A7,A2
018EDC  CF47                      EXG       D7,D7

A2 =>
		05BC8C  007C0700                  ORI.W     #$700,SR
		05BC90  33FC008000FF8604          MOVE.W    #$80,$FF8604.L
		05BC98  01090000            L0000:MOVEP.W   0(A1),D0
		05BC9C  B440                      CMP.W     D0,D2
		05BC9E  67F8                      BEQ.S     -8(PC)               L0000
		05BCA0  068200000200              ADDI.L    #$200,D2
		05BCA6  5281                L0001:ADDQ.L    #1,D1
		05BCA8  01090000                  MOVEP.W   0(A1),D0
		05BCAC  B440                      CMP.W     D0,D2
		05BCAE  66F6                      BNE.S     -10(PC)              L0001
		05BCB0  0839000500FFFA01    L0002:BTST      #5,$FFFA01.L
		05BCB8  66F6                      BNE.S     -10(PC)              L0002
		05BCBA  4E75                      RTS

018EDE  4FEF0030                  LEA       48(A7),A7
018EE2  2001                      MOVE.L    D1,D0
018EE4  4CDF0004                  MOVEM.L   (A7)+,D2
018EE8  4E75                      RTS

checkDiskSectorInterleave:
018EEA  61000044                  BSR       68(PC)               L0016
018EEE  6B3C                      BMI.S     60(PC)               L0015
018EF0  B03C0002                  CMP.B     #2,D0               ;512 bytes sector?
018EF4  6636                      BNE.S     54(PC)               L0015 ;no => error
018EF6  2200                      MOVE.L    D0,D1
018EF8  E199                      ROL.L     #8,D1
018EFA  0281000000FF              ANDI.L    #$FF,D1             ;track number
018F00  E088                      LSR.L     #8,D0               ;sector number
018F02  82FC0005                  DIVU      #5,D1
018F06  4241                      CLR.W     D1
018F08  4841                      SWAP      D1
018F0A  C2FCFFFE                  MULU      #$FFFE,D1
018F0E  02810000FFFF              ANDI.L    #$FFFF,D1
018F14  0641000B                  ADDI.W    #11,D1
018F18  82FC000A                  DIVU      #10,D1
018F1C  4841                      SWAP      D1                  ;d1 = (11 + (track % 5) * -2) % 10
018F1E  B200                      CMP.B     D0,D1               ;sector number == track
018F20  660A                      BNE.S     10(PC)               L0015
018F22  E088                      LSR.L     #8,D0               ;side
018F24  4A00                      TST.B     D0
018F26  6604                      BNE.S     4(PC)                L0015 ;Side 2? => error
018F28  E088                      LSR.L     #8,D0               ;return the track number
018F2A  4E75                      RTS
018F2C  70FF                L0015:MOVEQ     #-1,D0
018F2E  4E75                      RTS

; read track 0, load 4 bytes of the address block of first sector into D0 (Track,Side,Sector,Length)
; == 0x00000002
018F30  7202                L0016:MOVEQ     #2,D1               ; 3 tries
018F32  61000062            L0017:BSR       98(PC)               fdcReadTrack
018F36  6B2A                      BMI.S     42(PC)               L001B
018F38  204B                      MOVEA.L   A3,A0
018F3A  303C01C1                  MOVE.W    #450-1,D0
018F3E  0C1800FE            L0018:CMPI.B    #$FE,(A0)+
018F42  6614                      BNE.S     20(PC)               L001A
018F44  0C2800A1FFFE              CMPI.B    #$A1,-2(A0)
018F4A  660C                      BNE.S     12(PC)               L001A
018F4C  7203                      MOVEQ     #3,D1
018F4E  E188                L0019:LSL.L     #8,D0
018F50  1018                      MOVE.B    (A0)+,D0
018F52  51C9FFFA                  DBF       D1,-6(PC)            L0019
018F56  4E75                      RTS
018F58  51C8FFE4            L001A:DBF       D0,-28(PC)           L0018
018F5C  51C9FFD4                  DBF       D1,-44(PC)           L0017
018F60  70FF                      MOVEQ     #-1,D0
018F62  4E75                L001B:RTS

fdcDMAReadAddress:
018F64  200B                      MOVE.L    A3,D0
018F66  13C000FF860D              MOVE.B    D0,$FF860D.L
018F6C  E088                      LSR.L     #8,D0
018F6E  13C000FF860B              MOVE.B    D0,$FF860B.L
018F74  E088                      LSR.L     #8,D0
018F76  13C000FF8609              MOVE.B    D0,$FF8609.L
018F7C  33FC009000FF8606          MOVE.W    #$90,$FF8606.L
018F84  33FC019000FF8606          MOVE.W    #$190,$FF8606.L
018F8C  33FC009000FF8606          MOVE.W    #$90,$FF8606.L
018F94  4E75                      RTS

fdcReadTrack:
018F96  48A74000                  MOVEM.W   D1,-(A7)
018F9A  6100FFC8                  BSR       -56(PC)              fdcDMAReadAddress
018F9E  303C001F                  MOVE.W    #$1F,D0
018FA2  6100014A                  BSR       330(PC)              fdcWriteD0
018FA6  33FC008000FF8606          MOVE.W    #$80,$FF8606.L
018FAE  223C00040000              MOVE.L    #$40000,D1
018FB4  43EB01C2                  LEA       450(A3),A1
018FB8  40E7                      MOVE      SR,-(A7)
018FBA  007C0700                  ORI.W     #$700,SR
018FBE  2F3C584F4E75              MOVE.L    #$584F4E75,-(A7)
018FC4  2F3C00FF8604              MOVE.L    #$FF8604,-(A7)
018FCA  2F3C33FC00D0              MOVE.L    #$33FC00D0,-(A7)
018FD0  2F3C00FF8606              MOVE.L    #$FF8606,-(A7)
018FD6  2F3C33FC0080              MOVE.L    #$33FC0080,-(A7)
018FDC  2F3CB3D76ED6              MOVE.L    #$B3D76ED6,-(A7)
018FE2  2F3C860D0003              MOVE.L    #$860D0003,-(A7)
018FE8  2F3C1F7900FF              MOVE.L    #$1F7900FF,-(A7)
018FEE  2F3C860B0002              MOVE.L    #$860B0002,-(A7)
018FF4  2F3C1F7900FF              MOVE.L    #$1F7900FF,-(A7)
018FFA  2F3C86090001              MOVE.L    #$86090001,-(A7)
019000  2F3C1F7900FF              MOVE.L    #$1F7900FF,-(A7)
019006  2F3C53816B1C              MOVE.L    #$53816B1C,-(A7)
01900C  2F3CFA016720              MOVE.L    #$FA016720,-(A7)
019012  2F3C000500FF              MOVE.L    #$500FF,-(A7)
019018  2F3C42A70839              MOVE.L    #$42A70839,-(A7)
01901E  2F3C00FF8604              MOVE.L    #$FF8604,-(A7)
019024  2F3C33FC00E0              MOVE.L    #$33FC00E0,-(A7)
01902A  244F                      MOVEA.L   A7,A2
01902C  CF47                      EXG       D7,D7

A2 =>
		05BC8C  33FC00E000FF8604          MOVE.W    #$E0,$FF8604.L      ; Read Track
		05BC94  42A7                      CLR.L     -(A7)
		05BC96  0839000500FFFA01    L0000:BTST      #5,$FFFA01.L
		05BC9E  6720                      BEQ.S     32(PC)               L0001
		05BCA0  5381                      SUBQ.L    #1,D1
		05BCA2  6B1C                      BMI.S     28(PC)               L0001
		05BCA4  1F7900FF86090001          MOVE.B    $FF8609.L,1(A7)
		05BCAC  1F7900FF860B0002          MOVE.B    $FF860B.L,2(A7)
		05BCB4  1F7900FF860D0003          MOVE.B    $FF860D.L,3(A7)
		05BCBC  B3D7                      CMPA.L    (A7),A1             ; end address reached?
		05BCBE  6ED6                      BGT.S     -42(PC)              L0000
		05BCC0  33FC008000FF8606    L0001:MOVE.W    #$80,$FF8606.L
		05BCC8  33FC00D000FF8604          MOVE.W    #$D0,$FF8604.L      ; Force Interrupt
		05BCD0  584F                      ADDQ.W    #4,A7
		05BCD2  4E75                      RTS

01902E  4FEF0048                  LEA       72(A7),A7
019032  46DF                      MOVE      (A7)+,SR
019034  2001                      MOVE.L    D1,D0
019036  4C9F0002                  MOVEM.W   (A7)+,D1
01903A  4E75                      RTS

fdcSeekD0:
01903C  33FC008600FF8606          MOVE.W    #$86,$FF8606.L
019044  610000A8                  BSR       168(PC)              fdcWriteD0
019048  303C0014                  MOVE.W    #$14,D0
01904C  6100000A                  BSR       10(PC)               fdcCommand
019050  6B02                      BMI.S     2(PC)                L001F
019052  7000                      MOVEQ     #0,D0
019054  4E75                L001F:RTS

fdcRestore:
019056  7004                      MOVEQ     #4,D0

fdcCommand:
019058  B03C0080                  CMP.B     #$80,D0             ;Type I command?
01905C  6404                      BCC.S     4(PC)                L0022
01905E  00000003                  ORI.B     #3,D0               ;=> seek rate = 3ms
019062  33FC008000FF8606    L0022:MOVE.W    #$80,$FF8606.L
01906A  61000082                  BSR       130(PC)              fdcWriteD0
01906E  203C00004000              MOVE.L    #$4000,D0
019074  0839000500FFFA01    L0023:BTST      #5,$FFFA01.L
01907C  6756                      BEQ.S     86(PC)               fdcReadStatus
01907E  5380                      SUBQ.L    #1,D0
019080  66F2                      BNE.S     -14(PC)              L0023
019082  61000038                  BSR       56(PC)               fdcForceInterrupt
019086  70FF                      MOVEQ     #-1,D0
019088  4E75                      RTS


fdcFloppyDeselect:
01908A  223C0000004B              MOVE.L    #$4B,D1
019090  103C0007                  MOVE.B    #7,D0

fdcFloppySelect:
019094  5381                      SUBQ.L    #1,D1
019096  66FC                      BNE.S     -4(PC)               fdcFloppySelect
019098  40E7                      MOVE      SR,-(A7)
01909A  007C0700                  ORI.W     #$700,SR
01909E  13FC000E00FF8800          MOVE.B    #$E,$FF8800.L
0190A6  123900FF8800              MOVE.B    $FF8800.L,D1
0190AC  020100F8                  ANDI.B    #$F8,D1
0190B0  8200                      OR.B      D0,D1
0190B2  13C100FF8802              MOVE.B    D1,$FF8802.L
0190B8  46DF                      MOVE      (A7)+,SR
0190BA  4E75                      RTS

fdcForceInterrupt:
0190BC  33FC008000FF8606          MOVE.W    #$80,$FF8606.L
0190C4  303C00D0                  MOVE.W    #$D0,D0
0190C8  61000024                  BSR       36(PC)               fdcWriteD0
0190CC  303C000F                  MOVE.W    #$F,D0
0190D0  51C8FFFE            L0027:DBF       D0,-2(PC)            L0027

fdcReadStatus
0190D4  33FC008000FF8606          MOVE.W    #$80,$FF8606.L
0190DC  6100001A                  BSR       26(PC)               fdcDelay
0190E0  303900FF8604              MOVE.W    $FF8604.L,D0
0190E6  02800000001F              ANDI.L    #$1F,D0
0190EC  600A                      BRA.S     10(PC)               fdcDelay

fdcWriteD0
0190EE  61000008                  BSR       8(PC)                fdcDelay
0190F2  33C000FF8604              MOVE.W    D0,$FF8604.L

fdcDelay:
0190F8  40E7                      MOVE      SR,-(A7)
0190FA  3F00                      MOVE.W    D0,-(A7)
0190FC  303C0001                  MOVE.W    #1,D0
019100  51C8FFFE            L002B:DBF       D0,-2(PC)            L002B
019104  301F                      MOVE.W    (A7)+,D0
019106  46DF                      MOVE      (A7)+,SR
019108  4E75                      RTS

buffer512Bytes: DS.B 512
```
