---
title: "Atari ST Protection: The Sentry or The Sentinel"
date: "2011-09-01"
categories:
   - "software"
   - "atari st"
   - "reverse engineering"
---

This protection is in track 20, it searches for $13,$07,$F7,$19,$87,$F7 right AFTER the CRC of sector 9, especially the $F7 is nasty, because the FDC normally writes a two byte CRC when receiving a $F7 with the write track command. Because of a bad implementation these bytes could be written __inside__ sector 9 and it would still work...


``` 68k
001644  605C                      BRA.S     92(PC)               L0000

001646  4E4E4E4E4E4E6969          DC.B      $4E,$4E,$4E,$4E,$4E,$4E,$69,$69
00164E  6900020201000270          DC.B      $69,$00,$02,$02,$01,$00,$02,$70
001656  00D002F805000900          DC.B      $00,$D0,$02,$F8,$05,$00,$09,$00
00165E  01000000                  DC.B      $01,$00,$00,$00

001662                            DC.B 'Copyright (c) 1987 Michael K. Glover/Icom Simulations (U.K.) Ltd'

0016A2  780A                L0000:MOVEQ     #$A,D4
0016A4  2444                      MOVEA.L   D4,A2
0016A6  D4C4                      ADDA.W    D4,A2
0016A8  42AA0010                  CLR.L     16(A2)            ;$24
0016AC  42AA0030                  CLR.L     48(A2)            ;$44
0016B0  4EBA0136                  JSR       310(PC)              loadRegisters

0016B4  1145FFFE                  MOVE.B    D5,-2(A0)           ;$FF8800 = $0E
0016B8  E05D                      ROR.W     #8,D5
0016BA  1085                      MOVE.B    D5,(A0)             ;$FF8802 = $25

0016BC  32BC0080                  MOVE.W    #$80,(A1)           ;$FF8606 = $80
0016C0  4EBA0148                  JSR       328(PC)              clrTrace
0016C4  337C0003FFFE              MOVE.W    #3,-2(A1)           ;$FF8604 = $03   Restore
0016CA  4EBA012C                  JSR       300(PC)              waitFDC

0016CE  2C0E                      MOVE.L    A6,D6               ;buffer address = $30000
0016D0  17460002                  MOVE.B    D6,2(A3)            ;$FF860D = low
0016D4  E05E                      ROR.W     #8,D6
0016D6  1686                      MOVE.B    D6,(A3)             ;$FF860B = mid
0016D8  4846                      SWAP      D6
0016DA  1746FFFE                  MOVE.B    D6,-2(A3)           ;$FF8609 = high
0016DE  3282                      MOVE.W    D2,(A1)             ;$FF8606 = $90
0016E0  8441                      OR.W      D1,D2
0016E2  3282                      MOVE.W    D2,(A1)             ;$FF8606 = $190 DMA Read
0016E4  9441                      SUB.W     D1,D2
0016E6  3282                      MOVE.W    D2,(A1)             ;$FF8606 = $90
0016E8  4EBA0120                  JSR       288(PC)              clrTrace
0016EC  3343FFFE                  MOVE.W    D3,-2(A1)           ;$FF8604 = $1F 31 DMA Sectors
0016F0  4EBA0118                  JSR       280(PC)              clrTrace
0016F4  3284                      MOVE.W    D4,(A1)             ;$FF8606 = $80
0016F6  4EBA0112                  JSR       274(PC)              clrTrace
0016FA  3347FFFE                  MOVE.W    D7,-2(A1)           ;$FF8604 = $E0 Read Track
0016FE  4EBA00F8                  JSR       248(PC)              waitFDC

001702  32BC0086                  MOVE.W    #$86,(A1)           ;$FF8606 = $86
001706  4EBA0102                  JSR       258(PC)              clrTrace
00170A  337C0014FFFE              MOVE.W    #$14,-2(A1)         ;$FF8604 = $14
001710  4EBA00F8                  JSR       248(PC)              clrTrace
001714  3284                      MOVE.W    D4,(A1)             ;$FF8606 = $80
001716  4EBA00F2                  JSR       242(PC)              clrTrace
00171A  337C0013FFFE              MOVE.W    #$13,-2(A1)         ;$FF8606 = $13 Seek Track 20
001720  4EBA00D6                  JSR       214(PC)              waitFDC

001724  224E                      MOVEA.L   A6,A1
001726  D2FC125A                  ADDA.W    #$125A,A1           ;offset before header of sector 9 in track 0
00172A  3C3C0500                  MOVE.W    #$500,D6            ;search to the end of the track
00172E  7E05                L0001:MOVEQ     #5,D7
001730  41FA00F8                  LEA       248(PC),A0           L000F
										$13,$07,$F7,$19,$87,$F7 ;these bytes are directly after the CRC of sector 9.
																;funny thing: you could copy them INTO sector 9 and
																;the protection would still work.

001734  B308                L0002:CMPM.B    (A0)+,(A1)+
001736  6606                      BNE.S     6(PC)                L0003
001738  51CFFFFA                  DBF       D7,-6(PC)            L0002
00173C  600A                      BRA.S     10(PC)               L0004
00173E  51CEFFEE            L0003:DBF       D6,-18(PC)           L0001
001742  DECB                      ADDA.W    A3,A7               ;kill stackpointer (it will become negative => double bus fault)
001744  4EBA00C4                  JSR       196(PC)              clrTrace

001748  41FA0058            L0004:LEA       88(PC),A0            L0005
00174C  227C00054C00              MOVEA.L   #$54C00,A1
001752  2E3C0000A400              MOVE.L    #$A400,D7
001758  4EBA0062                  JSR       98(PC)               loadFile

00175C  41FA0047                  LEA       71(PC),A0            L0006
001760  227C00048000              MOVEA.L   #$48000,A1
001766  2E3C00007D00              MOVE.L    #$7D00,D7
00176C  4EBA004E                  JSR       78(PC)               loadFile

001770  41FA003E                  LEA       62(PC),A0            L0007
001774  227C00070000              MOVEA.L   #$70000,A1
00177A  2E3C00007D00              MOVE.L    #$7D00,D7
001780  4EBA003A                  JSR       58(PC)               loadFile

001784  42A7                      CLR.L     -(A7)
001786  3F3C0020                  MOVE.W    #$20,-(A7)
00178A  4E41                      TRAP      #1
00178C  5C8F                      ADDQ.L    #6,A7

00178E  13FC007B0005C452          MOVE.B    #$7B,$5C452
001796  11FC00801876              MOVE.B    #$80,$1876.W
00179C  4EF900058000              JMP       $58000

0017A2  534D00              L0005:DC.B      'SM',0
0017A5  5446494E414C2E4249  L0006:DC.B      'TFINAL.BIN',0
0017B0  534B59424F582E42    L0007:DC.B      'SKYBOX.BIN',0

0017BC  3F3C0000            loadFile:MOVE.W    #0,-(A7)
0017C0  2F08                      MOVE.L    A0,-(A7)
0017C2  3F3C003D                  MOVE.W    #$3D,-(A7)
0017C6                      L0009 EQU       *-2
0017C6  4E41                      TRAP      #1
0017C8  508F                      ADDQ.L    #8,A7
0017CA  3C00                      MOVE.W    D0,D6
0017CC  2F09                      MOVE.L    A1,-(A7)
0017CE  2F07                      MOVE.L    D7,-(A7)
0017D0  3F06                      MOVE.W    D6,-(A7)
0017D2  3F3C003F                  MOVE.W    #$3F,-(A7)
0017D6  4E41                      TRAP      #1
0017D8  DEFC000C                  ADDA.W    #$C,A7
0017DC  3F06                      MOVE.W    D6,-(A7)
0017DE  3F3C003E                  MOVE.W    #$3E,-(A7)
0017E2  4E41                      TRAP      #1
0017E4  588F                      ADDQ.L    #4,A7
0017E6  4E75                      RTS

0017E8  E744                loadRegisters:ASL.W     #3,D4
0017EA  4CBB3FFE40D6              MOVEM.W   -42(PC,D4.W),A0-A5/D1-D7 L0009      ;001814
0017F0  4DF900030000              LEA       $30000,A6
0017F6  4E75                      RTS

0017F8  2C3C0007A120        waitFDC:MOVE.L    #500000,D6
0017FE  08120005            L000C:BTST      #5,(A2)
001802  6704                      BEQ.S     4(PC)                L000D
001804  5386                      SUBQ.L    #1,D6
001806  66F6                      BNE.S     -10(PC)              L000C
001808  4E75                L000D:RTS

00180A  42B80024            clrTrace:CLR.L     $24.W
00180E  42B8002C                  CLR.L     $2C.W
001812  4E75                      RTS

001814  01000090001F0080          DC.W      $0100,$0090,$001F,$0080     ; D1,D2,D3,D4
00181C  250E230E00E08802          DC.W      $250E,$230E,$00E0,$8802     ; D5,D7,D7,A0
001824  8606FA01860B              DC.W      $8606,$FA01,$860B           ; A1,A2,A3

00182A  1307F71987F7FFA0    L000F:DC.B      $13,$07,$F7,$19,$87,$F7,$FF,$A0 ;A4,A5
001832  00000000CA4B0000          DC.B      $00,$00,$00,$00,$CA,$4B,$00,$00
00183A  0000000000000000          DC.B      $00,$00,$00,$00,$00,$00,$00,$00
001842  0000000000000000          DC.B      $00,$00,$00,$00,$00,$00,$00,$00
00184A  0000000000000000          DC.B      $00,$00,$00,$00,$00,$00,$00,$00
001852  0000000000000000          DC.B      $00,$00,$00,$00,$00,$00,$00,$00
```
