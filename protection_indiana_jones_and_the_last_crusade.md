---
title: "Atari ST Protection: Indiana Jones and the Last Crusade"
date: "2011-09-01"
categories:
   - "software"
   - "atari st"
   - "reverse engineering"
---

Another protection that measures the length of track 79, which has to be smaller than 6020 byte. The copy station created them bit a lower bitrate, on a standard floppy these tracks can not be written.

``` 68k
0001885a: 6100 0068                      BSR      fdcSelect
0001885e: 6100 0082                      BSR      fdcRestore

00018862: 303c 004e                      MOVE.W   #$4E,D0
00018866: 6100 0092                      BSR      fdcSeek
0001886a: 6100 0132                      BSR      fdcReadTrack
0001886e: 6100 010c                      BSR      dmaReadByteOffset
00018872: 23c0 0001 89f6                 MOVE.L   D0,$189F6         ;0x186d bytes long (unused?)

00018878: 303c 004f                      MOVE.W   #$4F,D0
0001887c: 6100 007c                      BSR      fdcSeek
00018880: 6100 011c                      BSR      fdcReadTrack
00018884: 6100 0020                      BSR      fdcDone
00018888: 6100 00f2                      BSR      dmaReadByteOffset ;0x175d bytes long
0001888c: b0bc 0000 1784                 CMP.L    #$1784,D0         ;6020 bytes
00018892: 6e00 000a                      BGT      *+$C [$1889E]
00018896: 203c 0000 0000                 MOVE.L   #0,D0             ;success
0001889c: 4e75                           RTS
0001889e: 203c 0000 0001                 MOVE.L   #1,D0             ;fail
000188a4: 4e75                           RTS


fdcDone
000188a6: 33fc 0080 ffff 8606            MOVE.W   #$80,$FFFF8606
000188ae: 3039 ffff 8604                 MOVE.W   $FFFF8604,D0
000188b4: 0800 0000                      BTST     #0,D0
000188b8: 6600 fff4                      BNE      *-$A [$188AE]
000188bc: 7007                           MOVEQ    #7,D0
000188be: 6100 0006                      BSR      *+$8 [$188C6]
000188c2: 4e75                           RTS

fdcSelect
000188c4: 7005                           MOVEQ    #5,D0
000188c6: 13fc 000e ffff 8800            MOVE.B   #$E,$FFFF8800
000188ce: 1239 ffff 8800                 MOVE.B   $FFFF8800,D1
000188d4: 0201 00f8                      ANDI.B   #$F8,D1
000188d8: 8200                           OR.B     D0,D1
000188da: 13c1 ffff 8802                 MOVE.B   D1,$FFFF8802
000188e0: 4e75                           RTS

fdcRestore
000188e2: 33fc 0080 ffff 8606            MOVE.W   #$80,$FFFF8606
000188ea: 6100 0056                      BSR      fdcDelay
000188ee: 33fc 0000 ffff 8604            MOVE.W   #$00,$FFFF8604
000188f6: 6000 005c                      BRA      fdcWait

fdcSeek
000188fa: 33fc 0086 ffff 8606            MOVE.W   #$86,$FFFF8606
00018902: 6100 003e                      BSR      fdcDelay
00018906: 33c0 ffff 8604                 MOVE.W   D0,$FFFF8604
0001890c: 6100 0046                      BSR      fdcWait
00018910: 33fc 0080 ffff 8606            MOVE.W   #$80,$FFFF8606
00018918: 6100 0028                      BSR      fdcDelay
0001891c: 33fc 0015 ffff 8604            MOVE.W   #$15,$FFFF8604
00018924: 6100 002e                      BSR      fdcWait
00018928: 4e75                           RTS

0001892a: 33fc 0080 ffff 8606            MOVE.W   #$80,$FFFF8606
00018932: 6100 000e                      BSR      fdcDelay
00018936: 33fc 0050 ffff 8604            MOVE.W   #$50,$FFFF8604
0001893e: 6000 0014                      BRA      fdcWait

fdcDelay
00018942: 40e7                           MOVE     SR,-(A7)
00018944: 3f07                           MOVE.W   D7,-(A7)
00018946: 3e3c 0020                      MOVE.W   #$20,D7
0001894a: 51cf fffe                      DBF      D7,*-$0 [$1894A]
0001894e: 3e1f                           MOVE.W   (A7)+,D7
00018950: 46df                           MOVE     (A7)+,SR
00018952: 4e75                           RTS

fdcWait
00018954: 0839 0005 ffff fa01            BTST     #5,$FFFFFA01
0001895c: 6600 fff6                      BNE      fdcWait
00018960: 4e75                           RTS

dmaSetAddress
00018962: 2008                           MOVE.L   A0,D0
00018964: 13c0 ffff 860d                 MOVE.B   D0,$FFFF860D
0001896a: e088                           LSR.L    #8,D0
0001896c: 13c0 ffff 860b                 MOVE.B   D0,$FFFF860B
00018972: e088                           LSR.L    #8,D0
00018974: 13c0 ffff 8609                 MOVE.B   D0,$FFFF8609
0001897a: 4e75                           RTS

dmaReadByteOffset
0001897c: 223c 0001 89fa                 MOVE.L   #$189FA,D1
00018982: 4280                           CLR.L    D0
00018984: 1039 00ff 8609                 MOVE.B   $FF8609,D0
0001898a: e188                           LSL.L    #8,D0
0001898c: 1039 00ff 860b                 MOVE.B   $FF860B,D0
00018992: e188                           LSL.L    #8,D0
00018994: 1039 00ff 860d                 MOVE.B   $FF860D,D0
0001899a: 9081                           SUB.L    D1,D0
0001899c: 4e75                           RTS

fdcReadTrack
0001899e: 303c 2710                      MOVE.W   #10000,D0
000189a2: 51c8 fffe                      DBF      D0,*-$0 [$189A2]
000189a6: 41f9 0001 89fa                 LEA      $189FA,A0
000189ac: 6100 ffb4                      BSR      dmaSetAddress
000189b0: 33fc 0090 ffff 8606            MOVE.W   #$90,$FFFF8606
000189b8: 33fc 0190 ffff 8606            MOVE.W   #$190,$FFFF8606
000189c0: 33fc 0090 ffff 8606            MOVE.W   #$90,$FFFF8606
000189c8: 6100 ff78                      BSR      fdcDelay
000189cc: 33fc 001f ffff 8604            MOVE.W   #$1F,$FFFF8604
000189d4: 6100 ff6c                      BSR      fdcDelay
000189d8: 33fc 0080 ffff 8606            MOVE.W   #$80,$FFFF8606
000189e0: 6100 ff60                      BSR      fdcDelay
000189e4: 33fc 00e4 ffff 8604            MOVE.W   #$E4,$FFFF8604
000189ec: 6100 ff54                      BSR      fdcDelay
000189f0: 6100 ff62                      BSR      fdcWait
000189f4: 4e75                           RTS

000189F6: DC.L 0
000189FA: DS.B buffersize
```
