---
title: "Atari ST Protection: Speedball"
date: "2011-09-01"
categories:
   - "software"
   - "atari st"
   - "reverse engineering"
---

SpeedBall protection is almost identical in the code to Garfield. Track 79 has to be shorter than normal (e.g. 5979 bytes, about 5% shorter than normal)

``` 68k
02BDA8  48E77F7E            L0434:MOVEM.L   A1-A6/D1-D7,-(A7)
02BDAC  23C80002C1D0              MOVE.L    A0,$0002C1D0         L0465
02BDB2  13FC00000002C1D4          MOVE.B    #0,$0002C1D4         L0466
02BDBA  263C00000001              MOVE.L    #1,D3
02BDC0  23C30002C1D6        L0435:MOVE.L    D3,$0002C1D6         L0467
02BDC6  4EB90002BE38              JSR       EnterSuper
02BDCC  4EB90002C19E              JSR       InitProtBuffer
02BDD2  50F90000043E              ST        $43E.L
02BDD8  4EB90002BE90              JSR       YMPortDeselect
02BDDE  4EB90002BECE              JSR       FDCForceInterrupt
02BDE4  4EB90002BEF2              JSR       FDCSeekTrack79
02BDEA  4EB90002BF48              JSR       FDCReadTrack
02BDF0  4EB90002BF48              JSR       FDCReadTrack
02BDF6  4EB90002BECE              JSR       FDCForceInterrupt
02BDFC  4EB90002C050              JSR       YMDeselectFloppy
02BE02  51F90000043E              SF        $43E.L
02BE08  4EB90002C06A              JSR       ExitSuper
02BE0E  4EB90002C14E              JSR       CheckProtBuffer
02BE14  10390002C1BE              MOVE.B    $0002C1BE,D0         L045C
02BE1A  B03C0000                  CMP.B     #0,D0
02BE1E  67000012                  BEQ       18(PC)               L0436
02BE22  26390002C1D6              MOVE.L    $0002C1D6,D3         L0467
02BE28  51CBFF96                  DBF       D3,-106(PC)          L0435
02BE2C  4EF900025290              JMP       $25290
02BE32  4CDF7EFE            L0436:MOVEM.L   (A7)+,A1-A6/D1-D7
02BE36  4E75                      RTS

EnterSuper
02BE38  2F3C00000001              MOVE.L    #1,-(A7)
02BE3E  3F3C0020                  MOVE.W    #$20,-(A7)
02BE42  4E41                      TRAP      #1
02BE44  DFFC00000006              ADDA.L    #6,A7
02BE4A  4A40                      TST.W     D0
02BE4C  6600002C                  BNE       44(PC)               L0438
02BE50  42A7                      CLR.L     -(A7)
02BE52  3F3C0020                  MOVE.W    #$20,-(A7)
02BE56  4E41                      TRAP      #1
02BE58  DFFC00000006              ADDA.L    #6,A7
02BE5E  23C00002C1C0              MOVE.L    D0,$0002C1C0         L045D
02BE64  12390002C1D4              MOVE.B    $0002C1D4,D1         L0466
02BE6A  4A01                      TST.B     D1
02BE6C  66000020                  BNE       32(PC)               L0439
02BE70  13FC00020002C1D4          MOVE.B    #2,$0002C1D4         L0466
02BE78  4E75                      RTS
02BE7A  12390002C1D4        L0438:MOVE.B    $0002C1D4,D1         L0466
02BE80  4A01                      TST.B     D1
02BE82  6600000A                  BNE       10(PC)               L0439
02BE86  13FC00010002C1D4          MOVE.B    #1,$0002C1D4         L0466
02BE8E  4E75                L0439:RTS

YMPortDeselect
02BE90  4EB90002BE38              JSR       EnterSuper
02BE96  303C0000                  MOVE.W    #0,D0
02BE9A  5200                      ADDQ.B    #1,D0
02BE9C  E308                      LSL.B     #1,D0
02BE9E  00400000                  ORI.W     #0,D0
02BEA2  0A000007                  EORI.B    #7,D0
02BEA6  02000007                  ANDI.B    #7,D0
02BEAA  40E7                L043B:MOVE      SR,-(A7)
02BEAC  007C0700                  ORI.W     #$700,SR
02BEB0  13FC000E00FF8800          MOVE.B    #$E,$FF8800.L
02BEB8  123900FF8800              MOVE.B    $FF8800.L,D1
02BEBE  020100F8                  ANDI.B    #$F8,D1
02BEC2  8200                      OR.B      D0,D1
02BEC4  13C100FF8802              MOVE.B    D1,$FF8802.L
02BECA  46DF                      MOVE      (A7)+,SR
02BECC  4E75                      RTS

FDCForceInterrupt
02BECE  4EB90002BE38              JSR       EnterSuper
02BED4  33FC008000FF8606          MOVE.W    #$80,$FF8606.L
02BEDC  3C3C00D0                  MOVE.W    #$D0,D6
02BEE0  4EB90002C0B2              JSR       FDCWriteReg
02BEE6  3E3C0028                  MOVE.W    #$28,D7
02BEEA  4EB90002C0AC              JSR       DelayD7
02BEF0  4E75                      RTS

FDCSeekTrack79
02BEF2  4EB90002BE38              JSR       EnterSuper
02BEF8  4EB90002C0F0              JSR       $0002C0F0            L044E
02BEFE  33FC008600FF8606          MOVE.W    #$86,$FF8606.L
02BF06  3C3C004F                  MOVE.W    #$4F,D6
02BF0A  4EB90002C0B2              JSR       FDCWriteReg
02BF10  33FC008000FF8606          MOVE.W    #$80,$FF8606.L
02BF18  3C3C001B                  MOVE.W    #$1B,D6
02BF1C  4EB90002C0B2              JSR       FDCWriteReg
02BF22  2E3C00060000              MOVE.L    #$60000,D7
02BF28  5387                L043E:SUBQ.L    #1,D7
02BF2A  67000010                  BEQ       16(PC)               L043F
02BF2E  0839000500FFFA01          BTST      #5,$FFFA01.L
02BF36  6600FFF0                  BNE       -16(PC)              L043E
02BF3A  4E75                      RTS
02BF3C  3F3CFFF9            L043F:MOVE.W    #$FFF9,-(A7)
02BF40  4EB90002C14C              JSR       $0002C14C            L0452
02BF46  4E75                      RTS

FDCReadTrack
02BF48  4EB90002BE38              JSR       EnterSuper
02BF4E  42B90002C1C8              CLR.L     $0002C1C8            L045F
02BF54  40F90002C1B8              MOVE      SR,$0002C1B8         L0459
02BF5A  46FC2700                  MOVE      #$2700,SR
02BF5E  33FC009000FF8606          MOVE.W    #$90,$FF8606.L
02BF66  33FC019000FF8606          MOVE.W    #$190,$FF8606.L
02BF6E  33FC009000FF8606          MOVE.W    #$90,$FF8606.L
02BF76  3C3C0016                  MOVE.W    #$16,D6
02BF7A  343C0200                  MOVE.W    #$200,D2
02BF7E  C4C6                      MULU      D6,D2
02BF80  33C20002C1BA              MOVE.W    D2,$0002C1BA         L045A
02BF86  26390002C1D0              MOVE.L    $0002C1D0,D3         L0465
02BF8C  D483                      ADD.L     D3,D2
02BF8E  23C20002C1C4              MOVE.L    D2,$0002C1C4         L045E
02BF94  4EB90002C0B2              JSR       FDCWriteReg
02BF9A  20390002C1D0              MOVE.L    $0002C1D0,D0         L0465
02BFA0  13C000FF860D              MOVE.B    D0,$FF860D.L
02BFA6  E088                      LSR.L     #8,D0
02BFA8  13C000FF860B              MOVE.B    D0,$FF860B.L
02BFAE  E088                      LSR.L     #8,D0
02BFB0  13C000FF8609              MOVE.B    D0,$FF8609.L
02BFB6  33FC008000FF8606          MOVE.W    #$80,$FF8606.L
02BFBE  3C3C00E8                  MOVE.W    #$E8,D6
02BFC2  4EB90002C0B2              JSR       FDCWriteReg
02BFC8  2E3C00050000              MOVE.L    #$50000,D7
02BFCE  2A790002C1C4              MOVEA.L   $0002C1C4,A5         L045E
02BFD4  303C0200                  MOVE.W    #$200,D0
02BFD8  51C8FFFE            L0441:DBF       D0,-2(PC)            L0441
02BFDC  0839000500FFFA01    L0442:BTST      #5,$FFFA01.L
02BFE4  67000030                  BEQ       48(PC)               L0443
02BFE8  5387                      SUBQ.L    #1,D7
02BFEA  67000060                  BEQ       96(PC)               L0446
02BFEE  13F900FF86090002C1C9      MOVE.B    $FF8609.L,$0002C1C9  L0460
02BFF8  13F900FF860B0002C1CA      MOVE.B    $FF860B.L,$0002C1CA  L0461
02C002  13F900FF860D0002C1CB      MOVE.B    $FF860D.L,$0002C1CB  L0462
02C00C  BBF90002C1C8              CMPA.L    $0002C1C8,A5         L045F
02C012  6E00FFC8                  BGT       -56(PC)              L0442
02C016  33FC009000FF8606    L0443:MOVE.W    #$90,$FF8606.L
02C01E  3A3900FF8606              MOVE.W    $FF8606.L,D5
02C024  33C50002C1BC              MOVE.W    D5,$0002C1BC         L045B
02C02A  08050000                  BTST      #0,D5
02C02E  67000018                  BEQ       24(PC)               L0445
02C032  33FC008000FF8606          MOVE.W    #$80,$FF8606.L
02C03A  4EB90002C12E              JSR       $0002C12E            FDCReadRegB
02C040  46F90002C1B8        L0444:MOVE      $0002C1B8,SR         L0459
02C046  4E75                      RTS
02C048  6000FFF6            L0445:BRA       -10(PC)              L0444
02C04C  6000FFF2            L0446:BRA       -14(PC)              L0444

YMDeselectFloppy
02C050  4EB90002BE38              JSR       EnterSuper
02C056  33FC008000FF8606          MOVE.W    #$80,$FF8606.L
02C05E  103C0007                  MOVE.B    #7,D0
02C062  4EB90002BEAA              JSR       $0002BEAA            L043B
02C068  4E75                      RTS

ExitSuper
02C06A  10390002C1D4              MOVE.B    $0002C1D4,D0         L0466
02C070  5300                      SUBQ.B    #1,D0
02C072  4A00                      TST.B     D0
02C074  6700002C                  BEQ       44(PC)               L0449
02C078  2F3C00000001              MOVE.L    #1,-(A7)
02C07E  3F3C0020                  MOVE.W    #$20,-(A7)
02C082  4E41                      TRAP      #1
02C084  DFFC00000006              ADDA.L    #6,A7
02C08A  4A40                      TST.W     D0
02C08C  67000014                  BEQ       20(PC)               L0449
02C090  2F390002C1C0              MOVE.L    $0002C1C0,-(A7)      L045D
02C096  3F3C0020                  MOVE.W    #$20,-(A7)
02C09A  4E41                      TRAP      #1
02C09C  DFFC00000006              ADDA.L    #6,A7
02C0A2  33FC4E7100029B08    L0449:MOVE.W    #$4E71,$29B08       ;(#"Nq")
02C0AA  4E75                      RTS

DelayD7
02C0AC  51CFFFFE                  DBF       D7,-2(PC)            DelayD7
02C0B0  4E75                      RTS

FDCWriteReg
02C0B2  4EB90002BE38              JSR       EnterSuper
02C0B8  4EB90002C0DE              JSR       fdcDelay
02C0BE  33C600FF8604              MOVE.W    D6,$FF8604.L
02C0C4  4EB90002C0DE              JSR       fdcDelay
02C0CA  4E75                      RTS

FDCReadReg
02C0CC  4EB90002BE38              JSR       EnterSuper
02C0D2  363900FF8604              MOVE.W    $FF8604.L,D3
02C0D8  4EB90002C0DE              JSR       fdcDelay

fdcDelay
02C0DE  40E7                      MOVE      SR,-(A7)
02C0E0  3F07                      MOVE.W    D7,-(A7)
02C0E2  3E3C0028                  MOVE.W    #$28,D7             ;(#"(")
02C0E6  51CFFFFE            L044D:DBF       D7,-2(PC)            L044D
02C0EA  3E1F                      MOVE.W    (A7)+,D7
02C0EC  46DF                      MOVE      (A7)+,SR
02C0EE  4E75                      RTS

02C0F0  3C390002C1CC        L044E:MOVE.W    $0002C1CC,D6         L0463
02C0F6  02460003                  ANDI.W    #3,D6
02C0FA  2E3C00050000              MOVE.L    #$50000,D7
02C100  33FC008000FF8606          MOVE.W    #$80,$FF8606.L
02C108  4EB90002C0B2              JSR       FDCWriteReg
02C10E  5387                L044F:SUBQ.L    #1,D7
02C110  67000010                  BEQ       16(PC)               L0450
02C114  0839000500FFFA01          BTST      #5,$FFFA01.L
02C11C  6600FFF0                  BNE       -16(PC)              L044F
02C120  4E75                      RTS
02C122  3F3CFFF9            L0450:MOVE.W    #$FFF9,-(A7)
02C126  4EB90002C14C              JSR       $0002C14C            L0452
02C12C  4E75                      RTS

FDCReadRegB
02C12E  4EB90002BE38              JSR       EnterSuper
02C134  4EB90002C0DE              JSR       fdcDelay
02C13A  33F900FF86040002C1CE      MOVE.W    $FF8604.L,$0002C1CE  L0464
02C144  4EB90002C0DE              JSR       fdcDelay
02C14A  4E75                      RTS

02C14C  4E75                L0452:RTS

CheckProtBuffer
02C14E  33FC4E750002BDA8          MOVE.W    #$4E75,$0002BDA8    ;(#"Nu") L0434
02C156  22390002C1D0              MOVE.L    $0002C1D0,D1         L0465
02C15C  068100002EE0              ADDI.L    #6000*2,D1
02C162  2041                      MOVEA.L   D1,A0
02C164  263C00000DAC              MOVE.L    #3500,D3
02C16A  243C00000000              MOVE.L    #0,D2
02C170  3A20                      MOVE.W    -(A0),D5
02C172  3020                L0454:MOVE.W    -(A0),D0
02C174  BA40                      CMP.W     D0,D5
02C176  660A                      BNE.S     10(PC)               L0455
02C178  5482                      ADDQ.L    #2,D2
02C17A  51CBFFF6                  DBF       D3,-10(PC)           L0454
02C17E  60000014                  BRA       20(PC)               L0456
02C182  04820000170C        L0455:SUBI.L    #5900,D2
02C188  6B0A                      BMI.S     10(PC)               L0456
02C18A  13FC00000002C1BE          MOVE.B    #0,$0002C1BE         L045C
02C192  4E75                      RTS
02C194  13FC00010002C1BE    L0456:MOVE.B    #1,$0002C1BE         L045C
02C19C  4E75                      RTS

InitProtBuffer
02C19E  243C0000176F              MOVE.L    #5999,D2
02C1A4  20790002C1D0              MOVEA.L   $0002C1D0,A0         L0465
02C1AA  2639000004BA              MOVE.L    $4BA.L,D3
02C1B0  30C3                L0458:MOVE.W    D3,(A0)+
02C1B2  51CAFFFC                  DBF       D2,-4(PC)            L0458
02C1B6  4E75                      RTS

02C1B8  2304                L0459:DC.W      $2304
02C1BA  2C00                L045A:DC.W      $2C00
02C1BC  0000                L045B:DC.W      $0000
02C1BE  0000                L045C:DC.W      $0000
02C1C0  00000000            L045D:DC.L      $00000000
02C1C4  00017EFA            L045E:DC.L      $00017EFA
02C1C8  00                  L045F:DC.B      $00
02C1C9  00                  L0460:DC.B      $00
02C1CA  00                  L0461:DC.B      $00
02C1CB  00                  L0462:DC.B      $00
02C1CC  0003                L0463:DC.W      $0003
02C1CE  0000                L0464:DC.W      $0000
02C1D0  000152FA            L0465:DC.L      $000152FA
02C1D4  0100                L0466:DC.B      $01,$00
02C1D6  00000001            L0467:DC.L      $00000001
```
