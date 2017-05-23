---
title: "Atari ST Protection: Garfield"
date: "2011-09-01"
categories:
   - "software"
   - "atari st"
   - "reverse engineering"
---

Track 79: shorter than 6100 bytes (and longer than 4996 bytes), that is not possible to be written with a standard drive. Pretty standard.

``` 68k
; allocate screen buffer
0x0d452: 0x2f3c 0x0000 0x9000                       MOVE.L   #0x9000,-(A7)
0x0d458: 0x3f3c 0x0048                              MOVE.W   #0x48,-(A7)
0x0d45c: 0x4e41                                     TRAP     #0x1
0x0d45e: 0x6700 0xfa8e                              BEQ      *-0x570 [0xCEEE]
0x0d462: 0x5c8f                                     ADDQ.L   #6,A7
0x0d464: 0x2d40 0x0142                              MOVE.L   D0,(0x142,A6)

...

; check copy protection with allocated screen buffer as track buffer.
; This buffer is reused right after this call.
0x0d4d2: 0x206e 0x0142                              MOVEA.L  (0x142,A6),A0
0x0d4d6: 0x6100 0x4430                              BSR      *+0x4432 [0x11908]
0x0d4da: 0x4a00                                     TST.B    D0
0x0d4dc: 0x6704                                     BEQ.S    *+0x6 [0xD4E2]

; protection failed, jump to beginning of init routine, but because of supervisor
; mode it will crash with a bus error initializing the IKBD...
0x0d4de: 0x6000 0xfa0e                              BRA      *-0x5F0 [0xCEEE]

...


----------------------------------------------------------------------------------------------------

0x11908: 0x48e7 0x7f7e                              MOVEM.L  D1-D7/A1-A6,-(A7)
0x1190c: 0x23c8 0x0001 0x1d1a                       MOVE.L   A0,0x11D1A         ;track buffer
0x11912: 0x13fc 0x0000 0x0001 0x1d1e                MOVE.B   #0,0x11D1E         ;supervisor mode flag
0x1191a: 0x263c 0x0000 0x0001                       MOVE.L   #1,D3
0x11920: 0x23c3 0x0001 0x1d20                       MOVE.L   D3,0x11D20         ;retry counter
0x11926: 0x4eb9 0x0001 0x1992                       JSR      EnterSuper
0x1192c: 0x4eb9 0x0001 0x1ce8                       JSR      InitProtBuffer
0x11932: 0x50f9 0x0000 0x043e                       ST .B    0x43E
0x11938: 0x4eb9 0x0001 0x19ea                       JSR      YMPortDeselect
0x1193e: 0x4eb9 0x0001 0x1a28                       JSR      FDCForceInterrupt
0x11944: 0x4eb9 0x0001 0x1a4c                       JSR      FDCSeekTrack79
0x1194a: 0x4eb9 0x0001 0x1aa2                       JSR      FDCReadTrack
0x11950: 0x4eb9 0x0001 0x1aa2                       JSR      FDCReadTrack
0x11956: 0x4eb9 0x0001 0x1a28                       JSR      FDCForceInterrupt
0x1195c: 0x4eb9 0x0001 0x1baa                       JSR      YMDeselectFloppy
0x11962: 0x51f9 0x0000 0x043e                       SF .B    0x43E
0x11968: 0x4eb9 0x0001 0x1bc4                       JSR      ExitSuper
0x1196e: 0x4eb9 0x0001 0x1ca0                       JSR      CheckProtBuffer
0x11974: 0x1039 0x0001 0x1d08                       MOVE.B   0x11D08,D0         ;failed flag
0x1197a: 0xb03c 0x0000                              CMP.B    #0,D0
0x1197e: 0x6700 0x000c                              BEQ      *+0xE [0x1198C]
0x11982: 0x2639 0x0001 0x1d20                       MOVE.L   0x11D20,D3         ;retry counter
0x11988: 0x51cb 0xff96                              DBF      D3,*-0x68 [0x11920]
0x1198c: 0x4cdf 0x7efe                              MOVEM.L  (A7)+,D1-D7/A1-A6
0x11990: 0x4e75                                     RTS

EnterSuper
0x11992: 0x2f3c 0x0000 0x0001                       MOVE.L   #1,-(A7)
0x11998: 0x3f3c 0x0020                              MOVE.W   #0x20,-(A7)
0x1199c: 0x4e41                                     TRAP     #1
0x1199e: 0xdffc 0x0000 0x0006                       ADDA.L   #6,A7
0x119a4: 0x4a40                                     TST.W    D0
0x119a6: 0x6600 0x002c                              BNE      *+0x2E [0x119D4]
0x119aa: 0x42a7                                     CLR.L    -(A7)
0x119ac: 0x3f3c 0x0020                              MOVE.W   #0x20,-(A7)
0x119b0: 0x4e41                                     TRAP     #1
0x119b2: 0xdffc 0x0000 0x0006                       ADDA.L   #6,A7
0x119b8: 0x23c0 0x0001 0x1d0a                       MOVE.L   D0,0x11D0A         ;USP stack pointer
0x119be: 0x1239 0x0001 0x1d1e                       MOVE.B   0x11D1E,D1         ;supervisor mode flag
0x119c4: 0x4a01                                     TST.B    D1
0x119c6: 0x6600 0x0020                              BNE      *+0x22 [0x119E8]
0x119ca: 0x13fc 0x0002 0x0001 0x1d1e                MOVE.B   #2,0x11D1E         ;supervisor mode flag
0x119d2: 0x4e75                                     RTS
0x119d4: 0x1239 0x0001 0x1d1e                       MOVE.B   0x11D1E,D1         ;supervisor mode flag
0x119da: 0x4a01                                     TST.B    D1
0x119dc: 0x6600 0x000a                              BNE      *+0xC [0x119E8]
0x119e0: 0x13fc 0x0001 0x0001 0x1d1e                MOVE.B   #1,0x11D1E         ;supervisor mode flag
0x119e8: 0x4e75                                     RTS

YMPortDeselect:
0x119ea: 0x4eb9 0x0001 0x1992                       JSR      EnterSuper
0x119f0: 0x303c 0x0000                              MOVE.W   #0,D0
0x119f4: 0x5200                                     ADDQ.B   #1,D0
0x119f6: 0xe308                                     LSL.B    #1,D0
0x119f8: 0x0040 0x0000                              ORI.W    #0,D0
0x119fc: 0x0a00 0x0007                              EORI.B   #7,D0
0x11a00: 0x0200 0x0007                              ANDI.B   #7,D0

YMPortSelect:
0x11a04: 0x40e7                                     MOVE     SR,-(A7)
0x11a06: 0x007c 0x0700                              ORI      #0x700,SR
0x11a0a: 0x13fc 0x000e 0x00ff 0x8800                MOVE.B   #0xE,0xFF8800
0x11a12: 0x1239 0x00ff 0x8800                       MOVE.B   0xFF8800,D1
0x11a18: 0x0201 0x00f8                              ANDI.B   #0xF8,D1
0x11a1c: 0x8200                                     OR.B     D0,D1
0x11a1e: 0x13c1 0x00ff 0x8802                       MOVE.B   D1,0xFF8802
0x11a24: 0x46df                                     MOVE     (A7)+,SR
0x11a26: 0x4e75                                     RTS

FDCForceInterrupt:
0x11a28: 0x4eb9 0x0001 0x1992                       JSR      EnterSuper
0x11a2e: 0x33fc 0x0080 0x00ff 0x8606                MOVE.W   #0x80,0xFF8606
0x11a36: 0x3c3c 0x00d0                              MOVE.W   #0xD0,D6
0x11a3a: 0x4eb9 0x0001 0x1c04                       JSR      FDCWriteReg
0x11a40: 0x3e3c 0x0028                              MOVE.W   #40,D7
0x11a44: 0x4eb9 0x0001 0x1bfe                       JSR      DelayD7
0x11a4a: 0x4e75                                     RTS

FDCSeekTrack79:
0x11a4c: 0x4eb9 0x0001 0x1992                       JSR      EnterSuper
0x11a52: 0x4eb9 0x0001 0x1c42                       JSR      0x11C42
0x11a58: 0x33fc 0x0086 0x00ff 0x8606                MOVE.W   #0x86,0xFF8606
0x11a60: 0x3c3c 0x004f                              MOVE.W   #0x4F,D6
0x11a64: 0x4eb9 0x0001 0x1c04                       JSR      FDCWriteReg
0x11a6a: 0x33fc 0x0080 0x00ff 0x8606                MOVE.W   #0x80,0xFF8606
0x11a72: 0x3c3c 0x001b                              MOVE.W   #0x1B,D6
0x11a76: 0x4eb9 0x0001 0x1c04                       JSR      FDCWriteReg
0x11a7c: 0x2e3c 0x0006 0x0000                       MOVE.L   #0x60000,D7
0x11a82: 0x5387                                     SUBQ.L   #1,D7
0x11a84: 0x6700 0x0010                              BEQ      *+0x12 [0x11A96]
0x11a88: 0x0839 0x0005 0x00ff 0xfa01                BTST     #5,0xFFFA01
0x11a90: 0x6600 0xfff0                              BNE      *-0xE [0x11A82]
0x11a94: 0x4e75                                     RTS
0x11a96: 0x3f3c 0xfff9                              MOVE.W   #0xFFF9,-(A7)
0x11a9a: 0x4eb9 0x0001 0x1c9e                       JSR      0x11C9E
0x11aa0: 0x4e75                                     RTS

FDCReadTrack:
0x11aa2: 0x4eb9 0x0001 0x1992                       JSR      EnterSuper
0x11aa8: 0x42b9 0x0001 0x1d12                       CLR.L    0x11D12        ;current DMA address pointer
0x11aae: 0x40f9 0x0001 0x1d02                       MOVE     SR,0x11D02     ;saved status register
0x11ab4: 0x46fc 0x2700                              MOVE     #0x2700,SR
0x11ab8: 0x33fc 0x0090 0x00ff 0x8606                MOVE.W   #0x90,0xFF8606
0x11ac0: 0x33fc 0x0190 0x00ff 0x8606                MOVE.W   #0x190,0xFF8606
0x11ac8: 0x33fc 0x0090 0x00ff 0x8606                MOVE.W   #0x90,0xFF8606
0x11ad0: 0x3c3c 0x0016                              MOVE.W   #22,D6         ;22 Sectors
0x11ad4: 0x343c 0x0200                              MOVE.W   #0x200,D2
0x11ad8: 0xc4c6                                     MULU.W   D6,D2
0x11ada: 0x33c2 0x0001 0x1d04                       MOVE.W   D2,0x11D04     ;number of bytes to read
0x11ae0: 0x2639 0x0001 0x1d1a                       MOVE.L   0x11D1A,D3     ;track buffer
0x11ae6: 0xd483                                     ADD.L    D3,D2
0x11ae8: 0x23c2 0x0001 0x1d0e                       MOVE.L   D2,0x11D0E     ;calculated DMA end address
0x11aee: 0x4eb9 0x0001 0x1c04                       JSR      FDCWriteReg    ;write sector count

0x11af4: 0x2039 0x0001 0x1d1a                       MOVE.L   0x11D1A,D0     ;track buffer
0x11afa: 0x13c0 0x00ff 0x860d                       MOVE.B   D0,0xFF860D
0x11b00: 0xe088                                     LSR.L    #8,D0
0x11b02: 0x13c0 0x00ff 0x860b                       MOVE.B   D0,0xFF860B    ;DMA base address to buffer start
0x11b08: 0xe088                                     LSR.L    #8,D0
0x11b0a: 0x13c0 0x00ff 0x8609                       MOVE.B   D0,0xFF8609

0x11b10: 0x33fc 0x0080 0x00ff 0x8606                MOVE.W   #0x80,0xFF8606 ;FDC Command Register
0x11b18: 0x3c3c 0x00e8                              MOVE.W   #0xE8,D6
0x11b1c: 0x4eb9 0x0001 0x1c04                       JSR      FDCWriteReg    ;Read Track

0x11b22: 0x2e3c 0x0005 0x0000                       MOVE.L   #0x50000,D7
0x11b28: 0x2a79 0x0001 0x1d0e                       MOVEA.L  0x11D0E,A5     ;calculated DMA end address
0x11b2e: 0x303c 0x0200                              MOVE.W   #0x200,D0
0x11b32: 0x51c8 0xfffe                              DBF      D0,*-0x0 [0x11B32] ;little delay
0x11b36: 0x0839 0x0005 0x00ff 0xfa01                BTST     #5,0xFFFA01    ;FDC done?
0x11b3e: 0x6700 0x0030                              BEQ      *+0x32 [0x11B70]
0x11b42: 0x5387                                     SUBQ.L   #1,D7          ;timeout?
0x11b44: 0x6700 0x0060                              BEQ      *+0x62 [0x11BA6]
0x11b48: 0x13f9 0x00ff 0x8609 0x0001 0x1d13         MOVE.B   0xFF8609,0x11D12+1
0x11b52: 0x13f9 0x00ff 0x860b 0x0001 0x1d14         MOVE.B   0xFF860B,0x11D12+2
0x11b5c: 0x13f9 0x00ff 0x860d 0x0001 0x1d15         MOVE.B   0xFF860D,0x11D12+3
0x11b66: 0xbbf9 0x0001 0x1d12                       CMPA.L   0x11D12,A5     ;check current DMA address
0x11b6c: 0x6e00 0xffc8                              BGT      *-0x36 [0x11B36]

0x11b70: 0x33fc 0x0090 0x00ff 0x8606                MOVE.W   #0x90,0xFF8606
0x11b78: 0x3a39 0x00ff 0x8606                       MOVE.W   0xFF8606,D5    ;FDC status
0x11b7e: 0x33c5 0x0001 0x1d06                       MOVE.W   D5,0x11D06     ;FDC status
0x11b84: 0x0805 0x0000                              BTST     #0,D5          ;Still Busy?
0x11b88: 0x6700 0x0018                              BEQ      *+0x1A [0x11BA2]
0x11b8c: 0x33fc 0x0080 0x00ff 0x8606                MOVE.W   #0x80,0xFF8606
0x11b94: 0x4eb9 0x0001 0x1c80                       JSR      FDCReadRegB
0x11b9a: 0x46f9 0x0001 0x1d02                       MOVE     0x11D02,SR     ;saved status register
0x11ba0: 0x4e75                                     RTS
0x11ba2: 0x6000 0xfff6                              BRA      *-0x8 [0x11B9A]
0x11ba6: 0x6000 0xfff2                              BRA      *-0xC [0x11B9A]

YMDeselectFloppy:
0x11baa: 0x4eb9 0x0001 0x1992                       JSR      EnterSuper
0x11bb0: 0x33fc 0x0080 0x00ff 0x8606                MOVE.W   #0x80,0xFF8606
0x11bb8: 0x103c 0x0007                              MOVE.B   #7,D0
0x11bbc: 0x4eb9 0x0001 0x1a04                       JSR      YMPortSelect
0x11bc2: 0x4e75                                     RTS

ExitSuper:
0x11bc4: 0x1039 0x0001 0x1d1e                       MOVE.B   0x11D1E,D0             ;supervisor mode flag
0x11bca: 0x5300                                     SUBQ.B   #1,D0
0x11bcc: 0x4a00                                     TST.B    D0
0x11bce: 0x6700 0x002c                              BEQ      *+0x2E [0x11BFC]
0x11bd2: 0x2f3c 0x0000 0x0001                       MOVE.L   #1,-(A7)
0x11bd8: 0x3f3c 0x0020                              MOVE.W   #0x20,-(A7)
0x11bdc: 0x4e41                                     TRAP     #1
0x11bde: 0xdffc 0x0000 0x0006                       ADDA.L   #6,A7
0x11be4: 0x4a40                                     TST.W    D0
0x11be6: 0x6700 0x0014                              BEQ      *+0x16 [0x11BFC]
0x11bea: 0x2f39 0x0001 0x1d0a                       MOVE.L   0x11D0A,-(A7)         ;USP stack pointer
0x11bf0: 0x3f3c 0x0020                              MOVE.W   #0x20,-(A7)
0x11bf4: 0x4e41                                     TRAP     #1
0x11bf6: 0xdffc 0x0000 0x0006                       ADDA.L   #6,A7
0x11bfc: 0x4e75                                     RTS

DelayD7:
0x11bfe: 0x51cf 0xfffe                              DBF      D7,*-0x0 [0x11BFE]
0x11c02: 0x4e75                                     RTS

FDCWriteReg:
0x11c04: 0x4eb9 0x0001 0x1992                       JSR      EnterSuper
0x11c0a: 0x4eb9 0x0001 0x1c30                       JSR      fdcDelay
0x11c10: 0x33c6 0x00ff 0x8604                       MOVE.W   D6,0xFF8604
0x11c16: 0x4eb9 0x0001 0x1c30                       JSR      fdcDelay
0x11c1c: 0x4e75                                     RTS

FDCReadReg:
0x11c1e: 0x4eb9 0x0001 0x1992                       JSR      EnterSuper
0x11c24: 0x3639 0x00ff 0x8604                       MOVE.W   0xFF8604,D3
0x11c2a: 0x4eb9 0x0001 0x1c30                       JSR      fdcDelay

fdcDelay
0x11c30: 0x40e7                                     MOVE     SR,-(A7)
0x11c32: 0x3f07                                     MOVE.W   D7,-(A7)
0x11c34: 0x3e3c 0x0028                              MOVE.W   #40,D7
0x11c38: 0x51cf 0xfffe                              DBF      D7,*-0x0 [0x11C38]
0x11c3c: 0x3e1f                                     MOVE.W   (A7)+,D7
0x11c3e: 0x46df                                     MOVE     (A7)+,SR
0x11c40: 0x4e75                                     RTS

0x11c42: 0x3c39 0x0001 0x1d16                       MOVE.W   0x11D16,D6
0x11c48: 0x0246 0x0003                              ANDI.W   #3,D6
0x11c4c: 0x2e3c 0x0005 0x0000                       MOVE.L   #0x50000,D7
0x11c52: 0x33fc 0x0080 0x00ff 0x8606                MOVE.W   #0x80,0xFF8606
0x11c5a: 0x4eb9 0x0001 0x1c04                       JSR      FDCWriteReg
0x11c60: 0x5387                                     SUBQ.L   #1,D7
0x11c62: 0x6700 0x0010                              BEQ      *+0x12 [0x11C74]
0x11c66: 0x0839 0x0005 0x00ff 0xfa01                BTST     #0x5,0xFFFA01
0x11c6e: 0x6600 0xfff0                              BNE      *-0xE [0x11C60]
0x11c72: 0x4e75                                     RTS
0x11c74: 0x3f3c 0xfff9                              MOVE.W   #0xFFF9,-(A7)
0x11c78: 0x4eb9 0x0001 0x1c9e                       JSR      0x11C9E
0x11c7e: 0x4e75                                     RTS                         ;crash because of 0xFFF9 on the stack

FDCReadRegB:
0x11c80: 0x4eb9 0x0001 0x1992                       JSR      EnterSuper
0x11c86: 0x4eb9 0x0001 0x1c30                       JSR      fdcDelay
0x11c8c: 0x33f9 0x00ff 0x8604 0x0001 0x1d18         MOVE.W   0xFF8604,0x11D18   ;FDC register
0x11c96: 0x4eb9 0x0001 0x1c30                       JSR      fdcDelay
0x11c9c: 0x4e75                                     RTS

0x11c9e: 0x4e75                                     RTS


CheckProtBuffer:
0x11ca0: 0x2239 0x0001 0x1d1a                       MOVE.L   0x11D1A,D1     ;track buffer
0x11ca6: 0x0681 0x0000 0x2ee0                       ADDI.L   #6000*2,D1     ;6000 word buffer
0x11cac: 0x2041                                     MOVEA.L  D1,A0
0x11cae: 0x263c 0x0000 0x0dac                       MOVE.L   #3500,D3       ;compare up to 3501 words
0x11cb4: 0x243c 0x0000 0x0000                       MOVE.L   #0,D2          ;byte counter of unused bytes in the track buffer
0x11cba: 0x3a20                                     MOVE.W   -(A0),D5
0x11cbc: 0x3020                                     MOVE.W   -(A0),D0
0x11cbe: 0xba40                                     CMP.W    D0,D5
0x11cc0: 0x660a                                     BNE.S    *+0xC [0x11CCC]
0x11cc2: 0x5482                                     ADDQ.L   #2,D2
0x11cc4: 0x51cb 0xfff6                              DBF      D3,*-0x8 [0x11CBC]
0x11cc8: 0x6000 0x0014                              BRA      *+0x16 [0x11CDE]   ;track too short (shorter than 4996 bytes) => failed
0x11ccc: 0x0482 0x0000 0x170c                       SUBI.L   #5900,D2       ;track length larger than (12000-5900) = 6100 bytes? => fail!
0x11cd2: 0x6b0a                                     BMI.S    *+0xC [0x11CDE]
0x11cd4: 0x13fc 0x0000 0x0001 0x1d08                MOVE.B   #0,0x11D08     ;failed flag = not failed
0x11cdc: 0x4e75                                     RTS
0x11cde: 0x13fc 0x0001 0x0001 0x1d08                MOVE.B   #1,0x11D08     ;failed flag = failed protection
0x11ce6: 0x4e75                                     RTS

InitProtBuffer:
0x11ce8: 0x243c 0x0000 0x176f                       MOVE.L   #5999,D2
0x11cee: 0x2079 0x0001 0x1d1a                       MOVEA.L  0x11D1A,A0     ;track buffer
0x11cf4: 0x2639 0x0000 0x04ba                       MOVE.L   0x4BA.w,D3     ; _hz_200
0x11cfa: 0x30c3                                     MOVE.W   D3,(A0)+       ;fill track buffer with "random" filler
0x11cfc: 0x51ca 0xfffc                              DBF      D2,*-0x2 [0x11CFA]
0x11d00: 0x4e75                                     RTS

0x11D02: 0000           ;saved status register
0x11D04: 0000           ;number of track bytes to read
0x11D06: 0000           ;FDC status
0x11D08: 0000           ;failed flag (0 = protection OK, 1 = protection failed)
0x11D0A: 00000000       ;USP stack pointer
0x11D0E: 00000000       ;DMA end address
0x11D12: 00000000       ;current DMA address pointer
0x11D16: 0003           ;floppy port selection bits
0x11D18: 0000           ;FDC register
0x11D1A: 00000000       ;track buffer
0x11D1E: 0000           ;supervisor mode flag
0x11D20: 00000000       ;retry counter
```
