---
title: "Atari ST Protection: Gauntlet II"
date: "2011-09-01"
categories:
   - "software"
   - "atari st"
   - "reverse engineering"
---

Another track length protection. Track 78 has to be at least 100 bytes shorter than Track 79. On my disk Track 78 was 5988 bytes, while 79 was 6285 bytes. The content of the track is not important for the protection, just the length. Therefore the track was recorded with ~240kbit/s instead of the normal 250kbit/s.

``` 68k
00B5D2  0000                L0403:DC.W      $0000   ;Restore Command
00B5D4  0010                L0404:DC.W      $0010   ;Seek Command
00B5D6  0030                L0405:DC.W      $0030   ;Step Command
00B5D8  0050                L0406:DC.W      $0050   ;Step-in Command
00B5DA  0070                L0407:DC.W      $0070   ;Step-out Command
00B5DC  0090                L0408:DC.W      $0090   ;Read multiple Sectors
00B5DE  00D0                L0409:DC.W      $00D0   ;Force Interrupt

fdcDoMoveTrack0
00B5E0  303C002C            L040A:MOVE.W    #$2C,D0
00B5E4  323C0002                  MOVE.W    #2,D1
00B5E8  4EBA03EC                  JSR       1004(PC)             fdcDoAction(fdcDoSelectFloppy)
00B5EC  303C0000                  MOVE.W    #0,D0
00B5F0  4EBA03E4                  JSR       996(PC)              fdcDoAction(fdcDoRestore)
00B5F4  303C0028                  MOVE.W    #$28,D0
00B5F8  7200                      MOVEQ     #0,D1
00B5FA  4EBA03DA                  JSR       986(PC)              fdcDoAction(fdcDoWriteTrackNo)
00B5FE  303C002C                  MOVE.W    #$2C,D0
00B602  323C0000                  MOVE.W    #0,D1
00B606  4EBA03CE                  JSR       974(PC)              fdcDoAction(fdcDoSelectFloppy)
00B60A  4E75                      RTS

...

fdcDoAction:
00B9D6  4BFA0304            L0422:LEA       772(PC),A5           L0440
00B9DA  3B400000                  MOVE.W    D0,0(A5)             L0440
00B9DE  426D000E                  CLR.W     14(A5)               L0446
00B9E2  42790000BCF8              CLR.W     $0000BCF8            L0449
00B9E8  0C40002C                  CMPI.W    #$2C,D0
00B9EC  620E                      BHI.S     14(PC)               L0423
00B9EE  49FA030E                  LEA       782(PC),A4           L044E
00B9F2  28740000                  MOVEA.L   0(A4,D0.W),A4
00B9F6  4E94                      JSR       (A4)
00B9F8  302D000A                  MOVE.W    10(A5),D0            L0444
00B9FC  4E75                L0423:RTS

fdcDoRestore:
00B9FE  33FC008000FF8606    L0424:MOVE.W    #$80,$FF8606.L
00BA06  3E3AFBCA                  MOVE.W    -1078(PC),D7         L0403 = Restore Command
00BA0A  4EBA01A8                  JSR       424(PC)              fdcWriteD7
00BA0E  4EBA01C4                  JSR       452(PC)              fdcWaitDone
00BA12  4E75                      RTS

fdcDoSeekD1
00BA14  33FC008600FF8606    L0425:MOVE.W    #$86,$FF8606.L
00BA1C  3E01                      MOVE.W    D1,D7
00BA1E  4EBA0194                  JSR       404(PC)              fdcWriteD7
00BA22  33FC008000FF8606          MOVE.W    #$80,$FF8606.L
00BA2A  3E3AFBA8                  MOVE.W    -1112(PC),D7         L0404 = Seek Command
00BA2E  4EBA0184                  JSR       388(PC)              fdcWriteD7
00BA32  4EBA01A0                  JSR       416(PC)              fdcWaitDone
00BA36  4E75                      RTS

fdcDoStep
00BA38  33FC008000FF8606    L0426:MOVE.W    #$80,$FF8606.L
00BA40  3E3AFB94                  MOVE.W    -1132(PC),D7         L0405 = Step Command
00BA44  4EBA016E                  JSR       366(PC)              fdcWriteD7
00BA48  4EBA018A                  JSR       394(PC)              fdcWaitDone
00BA4C  4E75                      RTS

fdcDoStepIn
00BA4E  33FC008000FF8606    L0427:MOVE.W    #$80,$FF8606.L
00BA56  3E3AFB80                  MOVE.W    -1152(PC),D7         L0406 = Step-in Command
00BA5A  4EBA0158                  JSR       344(PC)              fdcWriteD7
00BA5E  4EBA0174                  JSR       372(PC)              fdcWaitDone
00BA62  4E75                      RTS

fdcDoStepOut
00BA64  33FC008000FF8606    L0428:MOVE.W    #$80,$FF8606.L
00BA6C  3E3AFB6C                  MOVE.W    -1172(PC),D7         L0407 = Step-out Command
00BA70  4EBA0142                  JSR       322(PC)              fdcWriteD7
00BA74  4EBA015E                  JSR       350(PC)              fdcWaitDone
00BA78  4E75                      RTS

fdcDoForceInterrupt
00BA7A  3E3AFB62            L0429:MOVE.W    -1182(PC),D7         L0409 = Force Interrupt
00BA7E  4EBA0134                  JSR       308(PC)              fdcWriteD7
00BA82  3E3C3E80                  MOVE.W    #$3E80,D7
00BA86  51CFFFFE            L042A:DBF       D7,-2(PC)            L042A
00BA8A  4E75                      RTS

fdcDoReadSectors
00BA8C  2E0B                L042B:MOVE.L    A3,D7
00BA8E  4EBA00C6                  JSR       198(PC)              fdcDoSetAddress
00BA92  33FC00010000BCF8          MOVE.W    #1,$0000BCF8         L0449
00BA9A  33FC009000FF8606          MOVE.W    #$90,$FF8606.L
00BAA2  33FC019000FF8606          MOVE.W    #$190,$FF8606.L
00BAAA  33FC009000FF8606          MOVE.W    #$90,$FF8606.L
00BAB2  3E3C000C                  MOVE.W    #$C,D7
00BAB6  4EBA00FC                  JSR       252(PC)              fdcWriteD7
00BABA  33FC008400FF8606          MOVE.W    #$84,$FF8606.L
00BAC2  3E01                      MOVE.W    D1,D7
00BAC4  4EBA00EE                  JSR       238(PC)              fdcWriteD7
00BAC8  33FC008000FF8606          MOVE.W    #$80,$FF8606.L
00BAD0  3E390000B5DC              MOVE.W    $0000B5DC,D7         L0408 = Read multiple Sectors
00BAD6  4EBA00DC                  JSR       220(PC)              fdcWriteD7
00BADA  4EBA00F8                  JSR       248(PC)              fdcWaitDone
00BADE  4EBA00A0                  JSR       160(PC)              fdcDoReadAddress
00BAE2  4EBA0040                  JSR       64(PC)               fdcDoReadStatus
00BAE6  4E75                      RTS

fdcDoReadSectorNo
00BAE8  33FC008400FF8606    L042C:MOVE.W    #$84,$FF8606.L
00BAF0  4EBA00D2                  JSR       210(PC)              fdcReadD0
00BAF4  024000FF                  ANDI.W    #$FF,D0
00BAF8  3B400006                  MOVE.W    D0,6(A5)             L0442
00BAFC  33FC008000FF8606          MOVE.W    #$80,$FF8606.L
00BB04  4E75                      RTS

fdcDoReadTrackNo
00BB06  33FC008200FF8606    L042D:MOVE.W    #$82,$FF8606.L
00BB0E  4EBA00B4                  JSR       180(PC)              fdcReadD0
00BB12  024000FF                  ANDI.W    #$FF,D0
00BB16  3B400004                  MOVE.W    D0,4(A5)             L0441
00BB1A  33FC008000FF8606          MOVE.W    #$80,$FF8606.L
00BB22  4E75                      RTS

fdcDoReadStatus
00BB24  33FC008000FF8606    L042E:MOVE.W    #$80,$FF8606.L
00BB2C  4EBA0096                  JSR       150(PC)              fdcReadD0
00BB30  024000FF                  ANDI.W    #$FF,D0
00BB34  3B40000A                  MOVE.W    D0,10(A5)            L0444
00BB38  4E75                      RTS

fdcDoWriteTrackNo
00BB3A  33FC008200FF8606    L042F:MOVE.W    #$82,$FF8606.L
00BB42  3E01                      MOVE.W    D1,D7
00BB44  024700FF                  ANDI.W    #$FF,D7
00BB48  4EBA006A                  JSR       106(PC)              fdcWriteD7
00BB4C  33FC008000FF8606          MOVE.W    #$80,$FF8606.L
00BB54  4E75                      RTS

fdcDoSetAddress
00BB56  2B470010            L0430:MOVE.L    D7,16(A5)            L0447
00BB5A  13C700FF860D              MOVE.B    D7,$FF860D.L
00BB60  E08F                      LSR.L     #8,D7
00BB62  13C700FF860B              MOVE.B    D7,$FF860B.L
00BB68  E08F                      LSR.L     #8,D7
00BB6A  13C700FF8609              MOVE.B    D7,$FF8609.L
00BB70  2E2D0010                  MOVE.L    16(A5),D7            L0447
00BB74  7000                      MOVEQ     #0,D0
00BB76  3002                      MOVE.W    D2,D0
00BB78  DE80                      ADD.L     D0,D7
00BB7A  2B470014                  MOVE.L    D7,20(A5)            L0448
00BB7E  4E75                      RTS

fdcDoReadAddress
00BB80  303900FF8606        L0431:MOVE.W    $FF8606.L,D0
00BB86  02400007                  ANDI.W    #7,D0
00BB8A  3B40000C                  MOVE.W    D0,12(A5)            L0445
00BB8E  7200                      MOVEQ     #0,D1
00BB90  123900FF8609              MOVE.B    $FF8609.L,D1
00BB96  E189                      LSL.L     #8,D1
00BB98  123900FF860B              MOVE.B    $FF860B.L,D1
00BB9E  E189                      LSL.L     #8,D1
00BBA0  123900FF860D              MOVE.B    $FF860D.L,D1
00BBA6  2B410014                  MOVE.L    D1,20(A5)            L0448
00BBAA  92AD0010                  SUB.L     16(A5),D1            L0447
00BBAE  3B410008                  MOVE.W    D1,8(A5)             L0443
00BBB2  4E75                      RTS

fdcWriteD7
00BBB4  4EBA00BE            L0432:JSR       190(PC)              fdcDelay32
00BBB8  33C700FF8604              MOVE.W    D7,$FF8604.L
00BBBE  4EBA00B4                  JSR       180(PC)              fdcDelay32
00BBC2  4E75                      RTS

fdcReadD0
00BBC4  4EBA00AE            L0433:JSR       174(PC)              fdcDelay32
00BBC8  303900FF8604              MOVE.W    $FF8604.L,D0
00BBCE  4EBA00A4                  JSR       164(PC)              fdcDelay32
00BBD2  4E75                      RTS

fdcWaitDone
00BBD4  303C0360            L0434:MOVE.W    #$360,D0
00BBD8  51C8FFFE            L0435:DBF       D0,-2(PC)            L0435
00BBDC  203C00035000              MOVE.L    #$35000,D0
00BBE2  0839000500FFFA01    L0436:BTST      #5,$FFFA01.L
00BBEA  67000064                  BEQ       100(PC)              L0438
00BBEE  5380                      SUBQ.L    #1,D0
00BBF0  67000044                  BEQ       68(PC)               L0437
00BBF4  4A790000BCF8              TST.W     $0000BCF8            L0449
00BBFA  67E6                      BEQ.S     -26(PC)              L0436
00BBFC  13F900FF86090000BCFB      MOVE.B    $FF8609.L,$0000BCFB  L044A+1
00BC06  13F900FF860B0000BCFC      MOVE.B    $FF860B.L,$0000BCFC  L044A+2
00BC10  13F900FF860D0000BCFD      MOVE.B    $FF860D.L,$0000BCFD  L044A+3
00BC1A  2E390000BCFA              MOVE.L    $0000BCFA,D7         L044A
00BC20  BEAD0014                  CMP.L     20(A5),D7            L0448
00BC24  6D00FFBC                  BLT       -68(PC)              L0436
00BC28  4EBAFE50                  JSR       -432(PC)             fdcDoForceInterrupt
00BC2C  42790000BCF8              CLR.W     $0000BCF8            L0449
00BC32  6000001C                  BRA       28(PC)               L0438
00BC36  303900FF8604        L0437:MOVE.W    $FF8604.L,D0
00BC3C  024000FF                  ANDI.W    #$FF,D0
00BC40  3B40000C                  MOVE.W    D0,12(A5)            L0445
00BC44  4EBAFE34                  JSR       -460(PC)             fdcDoForceInterrupt
00BC48  3B7C0001000E              MOVE.W    #1,14(A5)            L0446
00BC4E  4E75                      RTS
00BC50  303900FF8604        L0438:MOVE.W    $FF8604.L,D0
00BC56  024000FF                  ANDI.W    #$FF,D0
00BC5A  3B40000A                  MOVE.W    D0,10(A5)            L0444
00BC5E  4E75                      RTS

fdcWaitForMotorOff
00BC60  33FC008000FF8606    L0439:MOVE.W    #$80,$FF8606.L
00BC68  4EBAFF5A            L043A:JSR       -166(PC)             fdcReadD0
00BC6C  08000007                  BTST      #7,D0
00BC70  66F6                      BNE.S     -10(PC)              L043A
00BC72  4E75                      RTS

fdcDelay32
00BC74  40E7                L043B:MOVE      SR,-(A7)
00BC76  3F00                      MOVE.W    D0,-(A7)
00BC78  303C0020                  MOVE.W    #$20,D0
00BC7C  51C8FFFE            L043C:DBF       D0,-2(PC)            L043C
00BC80  301F                      MOVE.W    (A7)+,D0
00BC82  46DF                      MOVE      (A7)+,SR
00BC84  4E75                      RTS

fdcDoSelectFloppy:
00BC86  4DF900FF8800        L043D:LEA       $FF8800.L,A6
00BC8C  303C07FF                  MOVE.W    #$7FF,D0
00BC90  018E0000                  MOVEP.W   D0,0(A6)
00BC94  7E00                      MOVEQ     #0,D7
00BC96  3E01                      MOVE.W    D1,D7
00BC98  6604                      BNE.S     4(PC)                L043E
00BC9A  4EBAFFC4                  JSR       -60(PC)              fdcWaitForMotorOff
00BC9E  0A070007            L043E:EORI.B    #7,D7
00BCA2  02070007                  ANDI.B    #7,D7
00BCA6  40E7                      MOVE      SR,-(A7)
00BCA8  007C0700                  ORI.W     #$700,SR
00BCAC  1CBC000E                  MOVE.B    #$E,(A6)
00BCB0  1016                      MOVE.B    (A6),D0
00BCB2  020000F8                  ANDI.B    #$F8,D0
00BCB6  8E00                      OR.B      D0,D7
00BCB8  1D470002                  MOVE.B    D7,2(A6)
00BCBC  46DF                      MOVE      (A7)+,SR
00BCBE  4A41                      TST.W     D1
00BCC0  6618                      BNE.S     24(PC)               L043F
00BCC2  303C073F                  MOVE.W    #$73F,D0
00BCC6  018E0000                  MOVEP.W   D0,0(A6)
00BCCA  13FC000300FFFC00          MOVE.B    #3,$FFFC00.L
00BCD2  13FC009600FFFC00          MOVE.B    #$96,$FFFC00.L
00BCDA  4E75                L043F:RTS

00BCDC  002C                L0440:DC.W      $002C
00BCDE  0000                      DC.W      $0000
00BCE0  0000                L0441:DC.W      $0000
00BCE2  0000                L0442:DC.W      $0000
00BCE4  0A00                L0443:DC.W      $0A00
00BCE6  00A4                L0444:DC.W      $00A4
00BCE8  0003                L0445:DC.W      $0003
00BCEA  0000                L0446:DC.W      $0000
00BCEC  00050CFC            L0447:DC.L      $00050CFC
00BCF0  000516FC            L0448:DC.L      $000516FC
00BCF4  00000000                  DC.L      $00000000
00BCF8  0000                L0449:DC.W      $0000
00BCFA  000516FC            L044A:DC.L      $000516FC
00BCFE  0000B9FE            L044E:DC.L      $0000B9FE            00 fdcDoRestore
00BD02  0000BA14                  DC.L      $0000BA14            04 fdcDoSeekD1
00BD06  0000BA38                  DC.L      $0000BA38            08 fdcDoStep
00BD0A  0000BA4E                  DC.L      $0000BA4E            0C fdcDoStepIn
00BD0E  0000BA64                  DC.L      $0000BA64            10 fdcDoStepOut
00BD12  0000BA8C                  DC.L      $0000BA8C            14 fdcDoReadSectors
00BD16  0000BA7A                  DC.L      $0000BA7A            18 fdcDoForceInterrupt
00BD1A  0000BB06                  DC.L      $0000BB06            1C fdcDoReadTrackNo
00BD1E  0000BAE8                  DC.L      $0000BAE8            20 fdcDoReadSectorNo
00BD22  0000BB24                  DC.L      $0000BB24            24 fdcDoReadStatus
00BD26  0000BB3A                  DC.L      $0000BB3A            28 fdcDoWriteTrackNo
00BD2A  0000BC86                  DC.L      $0000BC86            2C fdcDoSelectFloppy
00BD2E  1234                      DC.W      $1234

copyProtectionCheck
00BD30  4EB90000B5E0        L044F:JSR       $0000B5E0            fdcDoMoveTrack0
00BD36  303C002C                  MOVE.W    #$2C,D0
00BD3A  323C0002                  MOVE.W    #2,D1
00BD3E  4EB90000B9D6              JSR       $0000B9D6            fdcDoAction(fdcDoSelectFloppy)
00BD44  303C1F40                  MOVE.W    #$1F40,D0
00BD48  51C8FFFE            L0450:DBF       D0,-2(PC)            L0450
00BD4C  303C0004                  MOVE.W    #4,D0
00BD50  323C004E                  MOVE.W    #$4E,D1
00BD54  4EB90000B9D6              JSR       $0000B9D6            fdcDoAction(fdcDoSeekD1)
00BD5A  4EB90000BDAE              JSR       $0000BDAE            fdcReadTrack
00BD60  4EB90000BE28              JSR       $0000BE28            dmaReadAddress
00BD66  23C00000BE5E              MOVE.L    D0,$0000BE5E         L045A
00BD6C  303C1F40                  MOVE.W    #$1F40,D0
00BD70  51C8FFFE            L0451:DBF       D0,-2(PC)            L0451
00BD74  303C000C                  MOVE.W    #$C,D0
00BD78  4EB90000B9D6              JSR       $0000B9D6            fdcDoAction(fdcDoStepIn)
00BD7E  4EB90000BDAE              JSR       $0000BDAE            fdcReadTrack
00BD84  4EB90000BE28              JSR       $0000BE28            dmaReadAddress
00BD8A  91B90000BE5E              SUB.L     D0,$0000BE5E         L045A
00BD90  303C1F40                  MOVE.W    #$1F40,D0
00BD94  51C8FFFE            L0452:DBF       D0,-2(PC)            L0452
00BD98  4EB90000B5E0              JSR       $0000B5E0            fdcDoMoveTrack0
00BD9E  0CB9000000640000BE5E      CMPI.L    #100,$0000BE5E       L045A
00BDA8  6D00FF86                  BLT       -122(PC)             L044F
00BDAC  4E75                      RTS

fdcReadTrack
00BDAE  303C2710            L0453:MOVE.W    #$2710,D0
00BDB2  51C8FFFE            L0454:DBF       D0,-2(PC)            L0454
00BDB6  41F90001457C              LEA       $1457C,A0
00BDBC  4EB90000BE42              JSR       $0000BE42            dmaWriteAddress
00BDC2  33FC009000FF8606          MOVE.W    #$90,$FF8606.L
00BDCA  33FC019000FF8606          MOVE.W    #$190,$FF8606.L
00BDD2  33FC009000FF8606          MOVE.W    #$90,$FF8606.L
00BDDA  4EB90000BE16              JSR       $0000BE16            fdcDelay
00BDE0  33FC001F00FF8604          MOVE.W    #$1F,$FF8604.L
00BDE8  4EB90000BE16              JSR       $0000BE16            fdcDelay
00BDEE  33FC008000FF8606          MOVE.W    #$80,$FF8606.L
00BDF6  4EB90000BE16              JSR       $0000BE16            fdcDelay
00BDFC  33FC00E400FF8604          MOVE.W    #$E4,$FF8604.L
00BE04  4EB90000BE16              JSR       $0000BE16            fdcDelay
00BE0A  0839000500FFFA01    L0455:BTST      #5,$FFFA01.L
00BE12  66F6                      BNE.S     -10(PC)              L0455
00BE14  4E75                      RTS

fdcDelay
00BE16  40E7                L0456:MOVE      SR,-(A7)
00BE18  3F07                      MOVE.W    D7,-(A7)
00BE1A  3E3C0020                  MOVE.W    #$20,D7
00BE1E  51CFFFFE            L0457:DBF       D7,-2(PC)            L0457
00BE22  3E1F                      MOVE.W    (A7)+,D7
00BE24  46DF                      MOVE      (A7)+,SR
00BE26  4E75                      RTS

dmaReadAddress
00BE28  4280                L0458:CLR.L     D0
00BE2A  103900FF8609              MOVE.B    $FF8609.L,D0
00BE30  E188                      LSL.L     #8,D0
00BE32  103900FF860B              MOVE.B    $FF860B.L,D0
00BE38  E188                      LSL.L     #8,D0
00BE3A  103900FF860D              MOVE.B    $FF860D.L,D0
00BE40  4E75                      RTS

dmaWriteAddress
00BE42  2008                L0459:MOVE.L    A0,D0
00BE44  13C000FF860D              MOVE.B    D0,$FF860D.L
00BE4A  E088                      LSR.L     #8,D0
00BE4C  13C000FF860B              MOVE.B    D0,$FF860B.L
00BE52  E088                      LSR.L     #8,D0
00BE54  13C000FF8609              MOVE.B    D0,$FF8609.L
00BE5A  4E75                      RTS

00BE5C  5678                      DC.W      $5678
00BE5E  00000120            L045A:DC.L      $00000120
```
