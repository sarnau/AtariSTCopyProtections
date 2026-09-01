---
title: "Atari ST Protection: Indiana Jones and the Last Crusade"
date: "2011-09-01"
categories:
   - "software"
   - "atari st"
   - "reverse engineering"
---

Another protection that measures the length of track 79, which has to be smaller than 6020 byte. The copy station created them bit a lower bitrate, on a standard floppy these tracks can not be written.

The game is `AUTO\INDY.PRG` on disk 1 (Tiertex/U.S. Gold, 1989) - not to be confused with the
LucasArts SCUMM adventure of the same name, which is the one with the red filter and the printed
translation table. This action game has no manual protection at all: the only user visible strings
in the whole 59494 byte executable are the credits, the high score table, `PLEASE ENTER YOUR NAME`
and `PLACE DISK 2 INTO DRIVE A AND PRESS FIRE`.

`checkProtection` runs from the title screen sequence, *after* the game has asked you to swap in
disk 2, and `fdcSelect` hard-selects drive A. So the track that actually gets verified is the one on
**disk 2** - which is why both disks carry the same protected track 79.

Track 78 is read first and its length is stored, but never looked at again - a head settle/spin-up
warm-up, or a decoy.

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

## Where it is called from

``` 68k
titleScreen
0001af04: 41f9 0000 2d04                 LEA      $FBBE,A0          ;"GRAPHICS\TITLE.PC1"
0001af0a: 43f9 0005 4000                 LEA      $54000,A1
0001af10: 203c 0000 7d2c                 MOVE.L   #$7D2C,D0
0001af16: 4eb9 0000 281c                 JSR      loadFile

0001af1c: 41f9 0000 2d17                 LEA      $FBD1,A0          ;"GRAPHICS\INTRO.PC1"
0001af22: 43f9 0005 bfde                 LEA      $5BFDE,A1
0001af28: 203c 0000 7d2c                 MOVE.L   #$7D2C,D0
0001af2e: 4eb9 0000 281c                 JSR      loadFile

0001af34: 4eb9 0000 b9a0                 JSR      checkProtection
0001af3a: 4a40                           TST.W    D0
0001af3c: 6700 000c                      BEQ      titleScreenGo     ;original: leave the vector alone
0001af40: 23fc 0000 e090 0000 948c       MOVE.L   #$1AF4A,$16346    ;copy: poison the chain

titleScreenGo
0001af4a: 33fc 0001 0000 e018            MOVE.W   #1,$1AED2
```

## What happens on a copy

Nothing - and that is the point. On an original, `checkProtection` returns 0 and the routine simply
does nothing at all. On a copy it returns 1 and a single long word is written into the middle of the
Timer B raster interrupt below, at `$16346`. That long word is the operand of the `JMP` that ends the
routine. It is deliberately *not* in the relocation table - the `$1F000` in the file is only a placeholder.
At runtime that long word holds the previously installed vector, so the handler chains onwards
(`$FC06C0` was there under TOS 1.04). Overwriting it with `titleScreenGo` sends the interrupt into
the tail of the title screen code, which is not an interrupt handler and ends in an `RTS`. The check
itself passes silently; the machine dies later, with a bus error, a long way away from it.

``` 68k
installTimerB
000162f8: 13fc 0000 00ff fa1b            MOVE.B   #0,$FFFFFA1B
00016300: 23f9 0000 93ae 0000 0120       MOVE.L   $16268,$120       ;Timer B vector
0001630a: 48e7 80c0                      MOVEM.L  D0/A0-A1,-(A7)
0001630e: 3039 0000 93ac                 MOVE.W   $16266,D0
waitRaster
00016314: 13c0 00ff fa21                 MOVE.B   D0,$FFFFFA21      ;raster line to split on
0001631a: b039 00ff fa21                 CMP.B    $FFFFFA21,D0
00016320: 6600 fff2                      BNE      waitRaster
00016324: 13fc 0008 00ff fa1b            MOVE.B   #8,$FFFFFA1B
0001632c: 2079 0000 209c                 MOVEA.L  $EF56,A0          ;palette pointer
00016332: 43f9 00ff 8240                 LEA      $FFFF8240,A1
00016338: 700f                           MOVEQ    #15,D0
setPalette
0001633a: 32d8                           MOVE.W   (A0)+,(A1)+
0001633c: 51c8 fffc                      DBF      D0,setPalette
00016340: 4cdf 0301                      MOVEM.L  (A7)+,D0/A0-A1
00016344: 4ef9 0001 f000                 JMP      $0001F000         ;not relocated! overwritten at startup
0001634a: 4e73                           RTE
```

## Verified in Hatari

Breakpoint on the `CMP.L` at `$1888C`, with the game booted from disk 1 and disk 2 swapped into
drive A:

| disk 2 in drive A                        | D0            | result                                |
| ---------------------------------------- | ------------- | ------------------------------------- |
| original `.STX` (track 79 = 5982 byte)    | `$1750` = 5968 | <= 6020, returns 0, game runs         |
| plain `.ST` sector copy (standard track)  | `$1870` = 6256 | > 6020, returns 1, vector poisoned -> bus error |

A trace of the whole boot-to-gameplay run (`--trace fdc`) contains exactly two type III `READ TRACK`
commands, for tracks `$4E` and `$4F`, and nothing else out of the ordinary - no `READ ADDRESS`, and
no repeated reads of the weak sector that Pasti flags on track 79, sector 6.

## Unrelated: the IEHOVA cheat

Sitting right next to the protection call is a cheat that is checked on the title screen. Type
**I E H O V A** - the Latin spelling from the Grail floor puzzle - and `$EF70` is set to $FF.
With the cheat active, `I` skips a level, `RETURN` restarts the game and `ESC` reboots the machine.
(A test of F2 at `$F2A4` branches to the instruction that follows it anyway, so that one was
neutered before release.)

``` 68k
iehovaKeys
0001b02a: 1712 2318 2f1e                 DC.B     $17,$12,$23,$18,$2F,$1E; "IEHOVA"

checkIehova
0001b032: 4dfa fff6                      LEA      iehovaKeys(PC),A6
0001b036: 47f9 0000 299e                 LEA      keyMatrix,A3      ;keyboard matrix, $FF = key down
...
0001b0d8: 33fc 00ff 0000 20b6            MOVE.W   #$FF,$EF70        ;all six typed in order: cheat on
0001b0e0: 33fc 0333 00ff 8240            MOVE.W   #$333,$FF8240     ;flash the background

cheatKeys
0000f296: 4a79 0000 20b6                 TST.W    $EF70             ;cheat active?
0000f29c: 6700 ff58                      BEQ      normalKeys
0000f2a0: 41fa 05b6                      LEA      keyMatrix(PC),A0
...
0000f2c8: 4a28 0017                      TST.B    23(A0)            ;"I" - skip a level
0000f2cc: 6700 0010                      BEQ      $F2DE
0000f2d0: 5279 0000 20d8                 ADDQ.W   #1,$EF92
0000f2d6: 6100 67b2                      BSR      nextLevel
0000f2da: 6000 ff1a                      BRA      normalKeys
0000f2de: 4a28 001c                      TST.B    28(A0)            ;RETURN - restart the game
0000f2e2: 6600 fd32                      BNE      $F016
0000f2e6: 4a28 0001                      TST.B    1(A0)             ;ESC - reboot the machine
```
