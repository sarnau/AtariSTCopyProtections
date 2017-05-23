---
title: "Atari ST Protection: Arkanoid/CopylockST"
date: "2011-09-01"
categories:
   - "software"
   - "atari st"
   - "reverse engineering"
---

This is an earlier version of CopylockST from Rob Northen from 29/09/88, it doesn't use timing variations but three tests:

- Track 79: has to be shorter than 6027 bytes, that is not possible to be written with a standard drive.
- Track 0:  The GAP 3a of Sector 1 (after the address mark) has to be exactly 32 bytes long   and followed by 0x56,0x1E. Easily to miss this one.
- Track 0:  Byte 8 and 9 of Sector 1 has to contain 0xF5 and 0xF7 (not really a "protection")

``` 68k
copyProtectionBaseAddress:
000188e4: 243c 0000 0200                 MOVE.L   #$200,D2
000188ea: 206f 0004                      MOVEA.L  (4,A7),A0
000188ee: 2628 000c                      MOVE.L   (12,A0),D3
000188f2: 0483 0000 0044                 SUBI.L   #$44,D3
000188f8: 41fa ffea                      LEA      copyProtectionBaseAddress(PC),A0
000188fc: 43fa 002a                      LEA      startOfDecodedBlock(PC),A1
00018900: 4280                           CLR.L    D0
00018902: 4281                           CLR.L    D1
00018904: 3a30 0000                      MOVE.W   (0,A0,D0.W),D5
00018908: bb71 1000                      EOR.W    D5,(0,A1,D1.W)
0001890c: 3a31 1000                      MOVE.W   (0,A1,D1.W),D5
00018910: 5481                           ADDQ.L   #2,D1
00018912: b243                           CMP.W    D3,D1
00018914: 6600 fff2                      BNE      *-$C [$18908]
00018918: 5480                           ADDQ.L   #2,D0
0001891a: b07c 0044                      CMP.W    #$44,D0
0001891e: 6700 fff8                      BEQ      *-$6 [$18918]
00018922: b042                           CMP.W    D2,D0
00018924: 6600 ffdc                      BNE      *-$22 [$18902]

startOfDecodedBlock:
00018928: 2a6f 0004                      MOVEA.L  (4,A7),A5
0001892c: 206d 0010                      MOVEA.L  ($10,A5),A0
00018930: 43fa 0948                      LEA      ($948,PC) [$1927A],A1
00018934: 2288                           MOVE.L   A0,(A1)

00018936: 6100 0454                      BSR      disableMFPInterrupts
0001893a: 6100 02ea                      BSR      checkFloppyProtection
0001893e: 6100 0036                      BSR      readArkanoidProFile
00018942: 6100 0084                      BSR      decompressArkanoidProFile
00018946: 6100 0158                      BSR      checksumCompare
0001894a: 6100 0180                      BSR      relocateApplication
0001894e: 6100 01d8                      BSR      getApplicationSizes
00018952: 6100 0206                      BSR      runApplication
; the following code is unused, because runApplication never returns anyway

00018956: 6100 0174                      BSR      relocateApplication
0001895a: 6100 01cc                      BSR      getApplicationSizes
0001895e: 6100 042c                      BSR      disableMFPInterrupts
00018962: 6100 0012                      BSR      readArkanoidProFile
00018966: 6100 0060                      BSR      decompressArkanoidProFile
0001896a: 6100 0134                      BSR      checksumCompare
0001896e: 6100 02b6                      BSR      checkFloppyProtection
00018972: 6000 01e6                      BRA      runApplication


readArkanoidProFile:
00018976: 4267                           CLR.W    -(A7)
00018978: 487a 090a                      PEA      ($90A,PC) [$19284]
0001897c: 3f3c 003d                      MOVE.W   #$3D,-(A7)
00018980: 4e41                           TRAP     #1                        ;Fopen ("arkanoid.PRO", 0)
00018982: 508f                           ADDQ.L   #8,A7
00018984: 4a80                           TST.L    D0
00018986: 6b00 0038                      BMI      *+$3A [$189C0]
0001898a: 3f00                           MOVE.W   D0,-(A7)
0001898c: 2f3a 08ec                      MOVE.L   ($8EC,PC) [$1927A],-(A7)  ;buffer address
00018990: 2f3a 08e0                      MOVE.L   ($8E0,PC) [$19272],-(A7)  ;number of bytes
00018994: 3f00                           MOVE.W   D0,-(A7)
00018996: 3f3c 003f                      MOVE.W   #$3F,-(A7)
0001899a: 4e41                           TRAP     #1                        ;Fread ( int16_t handle, int32_t count, void *buf )
0001899c: dffc 0000 000c                 ADDA.L   #$C,A7
000189a2: 4a80                           TST.L    D0
000189a4: 6b00 001a                      BMI      *+$1C [$189C0]
000189a8: b0ba 08c8                      CMP.L    ($8C8,PC) [$19272],D0     ;all bytes read?
000189ac: 6600 0012                      BNE      *+$14 [$189C0]
000189b0: 3f3c 003e                      MOVE.W   #$3E,-(A7)
000189b4: 4e41                           TRAP     #1                        ;Fclose ( int16_t handle )
000189b6: 588f                           ADDQ.L   #4,A7
000189b8: 4a80                           TST.L    D0
000189ba: 6b00 0004                      BMI      *+$6 [$189C0]
000189be: 4e75                           RTS
000189c0: 41fa 08ea                      LEA      ($8EA,PC) [$192AC],A0     ;"[ESC]lDisc error loading file"
000189c4: 6000 01f4                      BRA      fatalError

decompressArkanoidProFile:
000189c8: 41fa ff1a                      LEA      copyProtectionBaseAddress,A0
000189cc: 203c 0000 0972                 MOVE.L   #$972,D0
000189d2: 6100 00ea                      BSR      longChecksumD6
000189d6: 41fa 092e                      LEA      ($92E,PC) [$19306],A0
000189da: 227a 089e                      MOVEA.L  ($89E,PC) [$1927A],A1     ;arkanoid pro file buffer
000189de: 323c 0001                      MOVE.W   #1,D1
000189e2: 4240                           CLR.W    D0
000189e4: 4a70 0000                      TST.W    (0,A0,D0.W)
000189e8: 6700 0032                      BEQ      *+$34 [$18A1C]
000189ec: b270 0000                      CMP.W    (0,A0,D0.W),D1
000189f0: 6700 0008                      BEQ      *+$A [$189FA]
000189f4: 5840                           ADDQ.W   #4,D0
000189f6: 6000 ffec                      BRA      *-$12 [$189E4]
000189fa: 3f30 0002                      MOVE.W   (2,A0,D0.W),-(A7)
000189fe: e448                           LSR.W    #2,D0
00018a00: 0640 0001                      ADDI.W   #1,D0
00018a04: 6100 0018                      BSR      *+$1A [$18A1E]
00018a08: 301f                           MOVE.W   (A7)+,D0
00018a0a: 343a 0876                      MOVE.W   ($876,PC) [$19282],D2     ;copy protection checksum
00018a0e: bd42                           EOR.W    D6,D2
00018a10: b540                           EOR.W    D2,D0
00018a12: 6100 005c                      BSR      *+$5E [$18A70]
00018a16: 5281                           ADDQ.L   #1,D1
00018a18: 6000 ffc8                      BRA      *-$36 [$189E2]
00018a1c: 4e75                           RTS

00018a1e: 48e7 c000                      MOVEM.L  D0-D1,-(A7)
00018a22: b240                           CMP.W    D0,D1
00018a24: 6700 0044                      BEQ      *+$46 [$18A6A]
00018a28: 0440 0001                      SUBI.W   #1,D0
00018a2c: 0441 0001                      SUBI.W   #1,D1
00018a30: e548                           LSL.W    #2,D0
00018a32: e549                           LSL.W    #2,D1
00018a34: 2430 0000                      MOVE.L   (0,A0,D0.W),D2
00018a38: 21b0 1000 0000                 MOVE.L   (0,A0,D1.W),(0,A0,D0.W)
00018a3e: 2182 1000                      MOVE.L   D2,(0,A0,D1.W)
00018a42: 0280 0000 ffff                 ANDI.L   #$FFFF,D0
00018a48: 0281 0000 ffff                 ANDI.L   #$FFFF,D1
00018a4e: ef88                           LSL.L    #7,D0
00018a50: ef89                           LSL.L    #7,D1
00018a52: 2449                           MOVEA.L  A1,A2
00018a54: 2649                           MOVEA.L  A1,A3
00018a56: d5c0                           ADDA.L   D0,A2
00018a58: d7c1                           ADDA.L   D1,A3
00018a5a: 203c 0000 007f                 MOVE.L   #$7F,D0
00018a60: 2212                           MOVE.L   (A2),D1
00018a62: 24d3                           MOVE.L   (A3),(A2)+
00018a64: 26c1                           MOVE.L   D1,(A3)+
00018a66: 51c8 fff8                      DBF      D0,*-$6 [$18A60]
00018a6a: 4cdf 0003                      MOVEM.L  (A7)+,D0-D1
00018a6e: 4e75                           RTS

00018a70: 48e7 4080                      MOVEM.L  D1/A0,-(A7)
00018a74: 0281 0000 ffff                 ANDI.L   #$FFFF,D1
00018a7a: 5381                           SUBQ.L   #1,D1
00018a7c: e189                           LSL.L    #8,D1
00018a7e: e389                           LSL.L    #1,D1
00018a80: 207a 07f8                      MOVEA.L  ($7F8,PC) [$1927A],A0
00018a84: d1c1                           ADDA.L   D1,A0
00018a86: 223c 0000 00ff                 MOVE.L   #$FF,D1
00018a8c: 4a58                           TST.W    (A0)+
00018a8e: 6700 0006                      BEQ      *+$8 [$18A96]
00018a92: b168 fffe                      EOR.W    D0,(-2,A0)
00018a96: 51c9 fff4                      DBF      D1,*-$A [$18A8C]
00018a9a: 4cdf 0102                      MOVEM.L  (A7)+,D1/A0
00018a9e: 4e75                           RTS

checksumCompare:
00018aa0: 207a 07d8                      MOVEA.L  ($7D8,PC) [$1927A],A0
00018aa4: 203a 07cc                      MOVE.L   ($7CC,PC) [$19272],D0
00018aa8: 6100 0014                      BSR      longChecksumD6
00018aac: bcba 07d0                      CMP.L    ($7D0,PC) [$1927E],D6
00018ab0: 6700 000a                      BEQ      *+$C [$18ABC]
00018ab4: 41fa 0826                      LEA      ($826,PC) [$192DC],A0     ;"[ESC]lData has been corrupted"
00018ab8: 6000 0100                      BRA      fatalError
00018abc: 4e75                           RTS

longChecksumD6:
00018abe: e488                           LSR.L    #2,D0
00018ac0: 4286                           CLR.L    D6
00018ac2: dc98                           ADD.L    (A0)+,D6
00018ac4: 5380                           SUBQ.L   #1,D0
00018ac6: 6600 fffa                      BNE      *-$4 [$18AC2]
00018aca: 4e75                           RTS

relocateApplication:
00018acc: 207a 07ac                      MOVEA.L  ($7AC,PC) [$1927A],A0
00018ad0: 2028 0002                      MOVE.L   (2,A0),D0             ;text size
00018ad4: d0a8 0006                      ADD.L    (6,A0),D0             ;data size
00018ad8: 43fa 079c                      LEA      ($79C,PC) [$19276],A1 ;application text+data size
00018adc: 2280                           MOVE.L   D0,(A1)
00018ade: 4a68 001a                      TST.W    ($1A,A0)              ;reloc info
00018ae2: 6600 0042                      BNE      *+$44 [$18B26]
00018ae6: 2248                           MOVEA.L  A0,A1
00018ae8: d3fc 0000 001c                 ADDA.L   #28,A1
00018aee: 2449                           MOVEA.L  A1,A2
00018af0: d5e8 0002                      ADDA.L   (2,A0),A2             ;text size
00018af4: d5e8 0006                      ADDA.L   (6,A0),A2             ;data size
00018af8: d5e8 000e                      ADDA.L   ($E,A0),A2            ;symbol table size
00018afc: 4281                           CLR.L    D1
00018afe: 41fa fde4                      LEA      copyProtectionBaseAddress,A0
00018b02: 2408                           MOVE.L   A0,D2
00018b04: 201a                           MOVE.L   (A2)+,D0
00018b06: d5b1 0800                      ADD.L    D2,(0,A1,D0.L)
00018b0a: 121a                           MOVE.B   (A2)+,D1
00018b0c: 4a01                           TST.B    D1
00018b0e: 6700 0016                      BEQ      *+$18 [$18B26]
00018b12: d081                           ADD.L    D1,D0
00018b14: b23c 0001                      CMP.B    #1,D1
00018b18: 6600 ffec                      BNE      *-$12 [$18B06]
00018b1c: 0680 0000 00fd                 ADDI.L   #254-1,D0
00018b22: 6000 ffe6                      BRA      *-$18 [$18B0A]
00018b26: 4e75                           RTS

getApplicationSizes:
00018b28: 206f 0008                      MOVEA.L  (8,A7),A0
00018b2c: 43fa fdb6                      LEA      copyProtectionBaseAddress,A1
00018b30: 2009                           MOVE.L   A1,D0
00018b32: 227a 0746                      MOVEA.L  ($746,PC) [$1927A],A1
00018b36: 2169 0002 000c                 MOVE.L   (2,A1),(12,A0)        ;text size
00018b3c: d0a8 000c                      ADD.L    (12,A0),D0
00018b40: 2140 0010                      MOVE.L   D0,(16,A0)
00018b44: 2169 0006 0014                 MOVE.L   (6,A1),(20,A0)        ;data size
00018b4a: d0a8 0014                      ADD.L    (20,A0),D0
00018b4e: 2140 0018                      MOVE.L   D0,(24,A0)
00018b52: 2169 000a 001c                 MOVE.L   (10,A1),(28,A0)       ;bss size
00018b58: 4e75                           RTS

runApplication:
00018b5a: 588f                           ADDQ.L   #4,A7
00018b5c: 206f 0004                      MOVEA.L  (4,A7),A0
00018b60: 2468 0018                      MOVEA.L  (24,A0),A2            ;bss address
00018b64: 2228 001c                      MOVE.L   (28,A0),D1            ;bss size
00018b68: 264a                           MOVEA.L  A2,A3
00018b6a: d7c1                           ADDA.L   D1,A3                 ;bss end address
00018b6c: 207a 070c                      MOVEA.L  ($70C,PC) [$1927A],A0
00018b70: d1fa 0700                      ADDA.L   ($700,PC) [$19272],A0
00018b74: b1cb                           CMPA.L   A3,A0
00018b76: 6d00 0004                      BLT      *+$6 [$18B7C]
00018b7a: 2648                           MOVEA.L  A0,A3
00018b7c: 203c 0000 000b                 MOVE.L   #$B,D0
00018b82: 41fa 0036                      LEA      fatalError,A0
00018b86: 43fa fd5c                      LEA      copyProtectionBaseAddress,A1
00018b8a: 2f09                           MOVE.L   A1,-(A7)
00018b8c: 3f20                           MOVE.W   -(A0),-(A7)
00018b8e: 51c8 fffc                      DBF      D0,*-$2 [$18B8C]
00018b92: 207a 06e6                      MOVEA.L  ($6E6,PC) [$1927A],A0
00018b96: d1fc 0000 001c                 ADDA.L   #28,A0                ;text + data code
00018b9c: 203a 06d8                      MOVE.L   ($6D8,PC) [$19276],D0 ;application text+data size
00018ba0: 4ed7                           JMP      (A7)
00018ba2: 22d8                           MOVE.L   (A0)+,(A1)+
00018ba4: 5980                           SUBQ.L   #4,D0
00018ba6: 6a00 fffa                      BPL      *-$4 [$18BA2]
00018baa: 429a                           CLR.L    (A2)+
00018bac: b7ca                           CMPA.L   A2,A3
00018bae: 6e00 fffa                      BGT      *-$4 [$18BAA]
00018bb2: dffc 0000 0018                 ADDA.L   #$18,A7
00018bb8: 4e75                           RTS

fatalError:
00018bba: 227a 06be                      MOVEA.L  ($6BE,PC) [$1927A],A1
00018bbe: 12d8                           MOVE.B   (A0)+,(A1)+
00018bc0: 6600 fffc                      BNE      *-$2 [$18BBE]
00018bc4: 5389                           SUBQ.L   #1,A1
00018bc6: 41fa 072e                      LEA      ($72E,PC) [$192F6],A0 ;" - Press a key"
00018bca: 12d8                           MOVE.B   (A0)+,(A1)+
00018bcc: 6600 fffc                      BNE      *-$2 [$18BCA]
00018bd0: 2009                           MOVE.L   A1,D0
00018bd2: 0800 0000                      BTST     #0,D0
00018bd6: 6700 0004                      BEQ      *+$6 [$18BDC]
00018bda: 5289                           ADDQ.L   #1,A1
00018bdc: 3f21                           MOVE.W   -(A1),-(A7)
00018bde: b3fa 069a                      CMPA.L   ($69A,PC) [$1927A],A1
00018be2: 6600 fff8                      BNE      *-$6 [$18BDC]

; copy fail text onto the stack and jump into this routine, it erases the copy protection routine
; and prints a fail text.
00018be6: 2c4f                           MOVEA.L  A7,A6
00018be8: 203c 0000 000d                 MOVE.L   #13,D0
00018bee: 41fa 0036                      LEA      ($36,PC) [$18C26],A0
00018bf2: 3f20                           MOVE.W   -(A0),-(A7)
00018bf4: 51c8 fffc                      DBF      D0,*-$2 [$18BF2]
00018bf8: 41fa fcea                      LEA      copyProtectionBaseAddress,A0
00018bfc: 227a 067c                      MOVEA.L  ($67C,PC) [$1927A],A1
00018c00: 93c8                           SUBA.L   A0,A1
00018c02: 203a 066e                      MOVE.L   ($66E,PC) [$19272],D0
00018c06: d089                           ADD.L    A1,D0
00018c08: 4ed7                           JMP      (A7)

00018c0a: 4218                           CLR.B    (A0)+
00018c0c: 51c8 fffc                      DBF      D0,*-$2 [$18C0A]
00018c10: 2f0e                           MOVE.L   A6,-(A7)
00018c12: 3f3c 0009                      MOVE.W   #9,-(A7)
00018c16: 4e41                           TRAP     #1
00018c18: 5c8f                           ADDQ.L   #6,A7
00018c1a: 3f3c 0001                      MOVE.W   #1,-(A7)
00018c1e: 4e41                           TRAP     #1
00018c20: 548f                           ADDQ.L   #2,A7
00018c22: 4267                           CLR.W    -(A7)
00018c24: 4e41                           TRAP     #1


checkFloppyProtection:
00018c26: 6100 05b4                      BSR      GEMDOS_Dgetdrv
00018c2a: 41fa 062a                      LEA      ($62A,PC) [$19256],A0
00018c2e: 3140 0000                      MOVE.W   D0,(0,A0)                 ;floppy drive
00018c32: b07c 0002                      CMP.W    #2,D0
00018c36: 6d00 001a                      BLT      *+$1C [$18C52]
00018c3a: 317c 0000 0000                 MOVE.W   #0,(0,A0)                 ;floppy drive = A
00018c40: 6100 0022                      BSR      checkProtectionDrive
00018c44: 6700 001c                      BEQ      *+$1E [$18C62]
00018c48: 41fa 060c                      LEA      ($60C,PC) [$19256],A0
00018c4c: 317c 0001 0000                 MOVE.W   #1,(0,A0)                 ;floppy drive = B
00018c52: 6100 0010                      BSR      checkProtectionDrive
00018c56: 6700 000a                      BEQ      *+$C [$18C62]
00018c5a: 41fa 066a                      LEA      ($66A,PC) [$192C6],A0     ;"[ESC]lThis disc is a copy"
00018c5e: 6000 ff5a                      BRA      fatalError
00018c62: 4e75                           RTS

checkProtectionDrive:
00018c64: 41fa 05f0                      LEA      ($5F0,PC) [$19256],A0
00018c68: 217a 0610 0002                 MOVE.L   ($610,PC) [$1927A],(2,A0) [$19258] ;FDC Buffer
00018c6e: 317c 00e0 0008                 MOVE.W   #$E0,(8,A0) [$1925E]      ;Read Track
00018c74: 41fa 05e0                      LEA      ($5E0,PC) [$19256],A0
00018c78: 317c 004f 000a                 MOVE.W   #$4F,(10,A0) [$19260]     ;Track 79
00018c7e: 6100 0144                      BSR      scheduleFDCCommand
00018c82: 243a 05e6                      MOVE.L   ($5E6,PC) [$1926A],D2     ;end address of the DMA
00018c86: 94ba 05d0                      SUB.L    ($5D0,PC) [$19258],D2     ;- start address of the DMA = length of the track
00018c8a: 6100 00d4                      BSR      testTrack79
00018c8e: 43fa 05f2                      LEA      ($5F2,PC) [$19282],A1     ;copy protection checksum
00018c92: 3282                           MOVE.W   D2,(A1)                   ;track length (filtered by valid values, should be $1746)
00018c94: 383c 0002                      MOVE.W   #2,D4                     ;try 3 times!
00018c98: 41fa 05bc                      LEA      ($5BC,PC) [$19256],A0
00018c9c: 317c 0000 000a                 MOVE.W   #0,(10,A0) [$19260]       ;Track 0
00018ca2: 6100 0120                      BSR      scheduleFDCCommand
00018ca6: 6100 0084                      BSR      testTrack0
00018caa: 57cc ffec                      DBEQ     D4,*-$12 [$18C98]

00018cae: 40e7                           MOVE     SR,-(A7)
00018cb0: 3f3c 0001                      MOVE.W   #1,-(A7)                  ;count
00018cb4: 4267                           CLR.W    -(A7)                     ;side
00018cb6: 4267                           CLR.W    -(A7)                     ;track
00018cb8: 3f3c 0001                      MOVE.W   #1,-(A7)                  ;sector
00018cbc: 3f3a 0598                      MOVE.W   ($598,PC) [$19256],-(A7)  ;devno = floppy drive
00018cc0: 42a7                           CLR.L    -(A7)                     ;filler
00018cc2: 2f3a 05b6                      MOVE.L   ($5B6,PC) [$1927A],-(A7)  ;FDC Buffer
00018cc6: 3f3c 0008                      MOVE.W   #8,-(A7)
00018cca: 4e4e                           TRAP     #$E
00018ccc: dffc 0000 0014                 ADDA.L   #$14,A7
00018cd2: 44df                           MOVE     (A7)+,CCR
00018cd4: 6600 000c                      BNE      *+$E [$18CE2]
00018cd8: 207a 05a0                      MOVEA.L  ($5A0,PC) [$1927A],A0
00018cdc: 0c68 f5f7 0008                 CMPI.W   #$F5F7,(8,A0)             ;Byte 8 and 9 in the boot sector should contain these bytes
00018ce2: 4e75                           RTS

track0SearchSector1:
00018ce4: 323a 057a                      MOVE.W   ($57A,PC) [$19260],D1     ;track number
00018ce8: 207a 0590                      MOVEA.L  ($590,PC) [$1927A],A0     ;buffer
00018cec: 4280                           CLR.L    D0
00018cee: 0c30 00fe 0000                 CMPI.B   #$FE,(0,A0,D0.W)          ;ID Address Mark
00018cf4: 6600 0028                      BNE      *+$2A [$18D1E]
00018cf8: b230 0001                      CMP.B    (1,A0,D0.W),D1            ;Track Number
00018cfc: 6600 0020                      BNE      *+$22 [$18D1E]
00018d00: 0c30 0000 0002                 CMPI.B   #0,(2,A0,D0.W)            ;Side = 0
00018d06: 6600 0016                      BNE      *+$18 [$18D1E]
00018d0a: 0c30 0001 0003                 CMPI.B   #1,(3,A0,D0.W)            ;Sector = 1
00018d10: 6600 000c                      BNE      *+$E [$18D1E]
00018d14: 0c30 0002 0004                 CMPI.B   #2,(4,A0,D0.W)            ;Sector Length = 2 (512 bytes)
00018d1a: 6700 000e                      BEQ      *+$10 [$18D2A]
00018d1e: 5280                           ADDQ.L   #1,D0
00018d20: b07c 1900                      CMP.W    #$1900,D0
00018d24: 6600 ffc8                      BNE      *-$36 [$18CEE]
00018d28: 7001                           MOVEQ    #1,D0
00018d2a: 4e75                           RTS

testTrack0:
00018d2c: 6100 ffb6                      BSR      track0SearchSector1       ;D0 should point the the Address Mark of the first sector
00018d30: 662c                           BNE.S    *+$2E [$18D5E]
00018d32: 323c ffff                      MOVE.W   #-1,D1
00018d36: 5280                           ADDQ.L   #1,D0
00018d38: 5281                           ADDQ.L   #1,D1
00018d3a: 0c30 004e 0006                 CMPI.B   #$4E,(6,A0,D0.W)          ;skip GAP 3a
00018d40: 6700 fff4                      BEQ      *-$A [$18D36]
00018d44: b27c 0020                      CMP.W    #32,D1                    ;should be exactly 32 bytes long! (normal is 22)
00018d48: 6600 0014                      BNE      *+$16 [$18D5E]
00018d4c: 1230 0006                      MOVE.B   (6,A0,D0.W),D1
00018d50: e149                           LSL.W    #8,D1                     ;$561E follow right after that
00018d52: 1230 0007                      MOVE.B   (7,A0,D0.W),D1
00018d56: 43fa 052a                      LEA      ($52A,PC) [$19282],A1     ;copy protection checksum
00018d5a: d351                           ADD.W    D1,(A1)                   ;$561e + $1746 == $6d64 ('md')
00018d5c: 7000                           MOVEQ    #0,D0
00018d5e: 4e75                           RTS

testTrack79:    // D2 should be 0x1738, should return $1746 in D2
00018d60: 41fa 0018                      LEA      ($18,PC) [$18D7A],A0
00018d64: b458                           CMP.W    (A0)+,D2
00018d66: 6cfc                           BGE.S    *-$2 [$18D64]
00018d68: 3220                           MOVE.W   -(A0),D1          ;max
00018d6a: 9260                           SUB.W    -(A0),D1          ;min
00018d6c: e249                           LSR.W    #1,D1
00018d6e: d250                           ADD.W    (A0),D1           ;val = (max - min) / 2 + min
00018d70: b441                           CMP.W    D1,D2             ;closer to previous value?
00018d72: 6d02                           BLT.S    *+$4 [$18D76]     ;then return min
00018d74: 4a58                           TST.W    (A0)+
00018d76: 3410                           MOVE.W   (A0),D2           ;else return max
00018d78: 4e75                           RTS
00018D7A: DC.W $0000,$1746,$17d1,$186a,$190a,$19b2,$1a64,$1b20,$7fff


disableMFPInterrupts:
00018d8c: 7800                           MOVEQ    #0,D4
00018d8e: 41fa 0016                      LEA      ($16,PC) [$18DA6],A0
00018d92: 3030 4000                      MOVE.W   (0,A0,D4.W),D0
00018d96: 6700 000c                      BEQ      *+$E [$18DA4]
00018d9a: 6100 001c                      BSR      *+$1E [$18DB8]
00018d9e: 5484                           ADDQ.L   #2,D4
00018da0: 6000 ffec                      BRA      *-$12 [$18D8E]
00018da4: 4e75                           RTS
00018DA6: DC.W 1,2,4,9,10,11,12,14,0
00018db8: 3f00                           MOVE.W   D0,-(A7)
00018dba: 3f3c 001a                      MOVE.W   #$1A,-(A7)
00018dbe: 4e4e                           TRAP     #$E                   ;Jdisint(number)
00018dc0: 588f                           ADDQ.L   #4,A7
00018dc2: 4e75                           RTS



scheduleFDCCommand:
00018dc4: 48e7 ff7e                      MOVEM.L  D0-D7/A1-A6,-(A7)
00018dc8: 43fa 049a                      LEA      ($49A,PC) [$19264],A1
00018dcc: 6100 0420                      BSR      GEMDOS_SuperOn
00018dd0: 50f9 0000 043e                 ST.B     $43E
00018dd6: 6100 037e                      BSR      select
00018dda: 6100 005a                      BSR      *+$5C [$18E36]
00018dde: 6700 0036                      BEQ      *+$38 [$18E16]
00018de2: 4a71 5000                      TST.W    (0,A1,D5.W)       ;track position for drive valid?
00018de6: 6a00 001c                      BPL      *+$1E [$18E04]    ;=> yes
00018dea: 6100 028e                      BSR      fdcRestore
00018dee: 6700 0014                      BEQ      *+$16 [$18E04]
00018df2: 7c0a                           MOVEQ    #$0A,D6           ;Restore with no spin-up, but with verify
00018df4: 6100 025a                      BSR      executeFDCCommandD6
00018df8: 6600 001c                      BNE      *+$1E [$18E16]
00018dfc: 6100 027c                      BSR      fdcRestore
00018e00: 6600 0014                      BNE      *+$16 [$18E16]

00018e04: 6100 0246                      BSR      executeFDCcommand
00018e08: 0c68 0069 0008                 CMPI.W   #$69,(8,A0) [$1925E]
00018e0e: 6700 0006                      BEQ      *+$8 [$18E16]
00018e12: 6100 007a                      BSR      *+$7C [$18E8E]
00018e16: 6100 032e                      BSR      deselect
00018e1a: 4279 0000 043e                 CLR.W    $43E
00018e20: 6100 0402                      BSR      GEMDOS_SuperOff
00018e24: 3028 0006                      MOVE.W   (6,A0) [$1925C],D0
00018e28: e348                           LSL.W    #1,D0
00018e2a: 31a9 0004 000a                 MOVE.W   (4,A1),(10,A0,D0.W) [$19260]
00018e30: 4cdf 7eff                      MOVEM.L  (A7)+,D0-D7/A1-A6
00018e34: 4e75                           RTS

00018e36: 0c68 007a 0008                 CMPI.W   #$7A,(8,A0) [$1925E]
00018e3c: 6700 000e                      BEQ      *+$10 [$18E4C]
00018e40: 0c68 007d 0008                 CMPI.W   #$7D,(8,A0) [$1925E]
00018e46: 6700 0018                      BEQ      *+$1A [$18E60]
00018e4a: 4e75                           RTS

00018e4c: 6100 0026                      BSR      *+$28 [$18E74]
00018e50: 6600 0008                      BNE      *+$A [$18E5A]
00018e54: 33a8 000c 0000                 MOVE.W   (12,A0),(0,A1,D0.W)
00018e5a: 003c 0004                      ORI      #$4,CCR
00018e5e: 4e75                           RTS

00018e60: 6100 0012                      BSR      *+$14 [$18E74]
00018e64: 6600 0008                      BNE      *+$A [$18E6E]
00018e68: 3171 0000 000c                 MOVE.W   (0,A1,D0.W),(12,A0)
00018e6e: 003c 0004                      ORI      #$4,CCR
00018e72: 4e75                           RTS

00018e74: 303c 0000                      MOVE.W   #0,D0
00018e78: 0c68 0012 000a                 CMPI.W   #$12,(10,A0) [$19260]      ; Drive A
00018e7e: 6700 000c                      BEQ      *+$E [$18E8C]
00018e82: 303c 0002                      MOVE.W   #2,D0
00018e86: 0c68 001a 000a                 CMPI.W   #$1A,(10,A0) [$19260]      ; Drive B
00018e8c: 4e75                           RTS

00018e8e: 0c68 00e0 0008                 CMPI.W   #$E0,(8,A0) [$1925E]       ;read track
00018e94: 6700 008e                      BEQ      *+$90 [$18F24]
00018e98: 0c68 00f0 0008                 CMPI.W   #$F0,(8,A0) [$1925E]       ;write track
00018e9e: 6700 0018                      BEQ      *+$1A [$18EB8]
00018ea2: 0c68 0053 0008                 CMPI.W   #$53,(8,A0) [$1925E]       ;write sector
00018ea8: 6700 0088                      BEQ      *+$8A [$18F32]
00018eac: 0c68 005d 0008                 CMPI.W   #$5D,(8,A0) [$1925E]       ;???
00018eb2: 6700 00f4                      BEQ      *+$F6 [$18FA8]
00018eb6: 4e75                           RTS

00018eb8: 3c3c 00f0                      MOVE.W   #$F0,D6
00018ebc: 6100 0174                      BSR      dmaWriteMode
00018ec0: 33fc 001f 00ff 8604            MOVE.W   #$1F,$FF8604
00018ec8: 33fc 0180 00ff 8606            MOVE.W   #$180,$FF8606
00018ed0: 6100 0228                      BSR      wrfdc
00018ed4: 2e3c 0004 0000                 MOVE.L   #$40000,D7
00018eda: 0839 0005 00ff fa01            BTST     #5,$FFFA01
00018ee2: 6700 0014                      BEQ      *+$16 [$18EF8]
00018ee6: 5387                           SUBQ.L   #1,D7
00018ee8: 6600 fff0                      BNE      *-$E [$18EDA]
00018eec: 6100 0220                      BSR      forceInterrupt
00018ef0: 7e01                           MOVEQ    #1,D7
00018ef2: 3347 0004                      MOVE.W   D7,(4,A1)
00018ef6: 4e75                           RTS
00018ef8: 33fc 0190 00ff 8606            MOVE.W   #$190,$FF8606
00018f00: 3039 00ff 8606                 MOVE.W   $FF8606,D0
00018f06: 0800 0000                      BTST     #0,D0
00018f0a: 6700 ffe4                      BEQ      *-$1A [$18EF0]
00018f0e: 33fc 0180 00ff 8606            MOVE.W   #$180,$FF8606
00018f16: 6100 020e                      BSR      readfdc
00018f1a: 0247 0044                      ANDI.W   #$44,D7
00018f1e: 3347 0004                      MOVE.W   D7,(4,A1)
00018f22: 4e75                           RTS


00018f24: 3c3c 00e0                      MOVE.W   #$E0,D6           ;read track
00018f28: 247c 0000 2c00                 MOVEA.L  #$2C00,A2         ;maximum number of bytes
00018f2e: 6000 001a                      BRA      *+$1C [$18F4A]

00018f32: 33fc 0084 00ff 8606            MOVE.W   #$84,$FF8606      ;FDC Sector Register
00018f3a: 3c28 000c                      MOVE.W   (12,A0) [$19262],D6 ;sector number
00018f3e: 6100 01ba                      BSR      wrfdc
00018f42: 3c3c 0090                      MOVE.W   #$90,D6           ;write sector
00018f46: 3468 000e                      MOVEA.W  (14,A0) [$19264],A2 ;number of bytes


00018f4a: 6100 00cc                      BSR      dmaReadMode
00018f4e: 33fc 0016 00ff 8604            MOVE.W   #$16,$FF8604
00018f56: d5e8 0002                      ADDA.L   (2,A0) [$19258],A2 ;end address of the buffer
00018f5a: 33fc 0080 00ff 8606            MOVE.W   #$80,$FF8606
00018f62: 6100 0196                      BSR      wrfdc
00018f66: 2e3c 0004 0000                 MOVE.L   #$40000,D7
00018f6c: 137c 0000 0006                 MOVE.B   #0,(6,A1)

00018f72: 0839 0005 00ff fa01            BTST     #5,$FFFA01        ;wait for the FDC to complete
00018f7a: 6700 008a                      BEQ      fdcDone
00018f7e: 1379 00ff 8609 0007            MOVE.B   $FF8609,(7,A1) [$1926A+1]
00018f86: 1379 00ff 860b 0008            MOVE.B   $FF860B,(8,A1) [$1926A+2]    ;or the end address being reached
00018f8e: 1379 00ff 860d 0009            MOVE.B   $FF860D,(9,A1) [$1926A+3]
00018f96: b5e9 0006                      CMPA.L   (6,A1) [$1926A],A2
00018f9a: 6f00 0066                      BLE      *+$68 [$19002]
00018f9e: 5387                           SUBQ.L   #1,D7             ;or a timeout
00018fa0: 6600 ffd0                      BNE      *-$2E [$18F72]
00018fa4: 6700 005c                      BEQ      fdcTimeout

00018fa8: 33fc 0080 00ff 8606            MOVE.W   #$80,$FF8606
00018fb0: 6100 0182                      BSR      waitl
00018fb4: 3e39 00ff 8604                 MOVE.W   $FF8604,D7
00018fba: 0247 0002                      ANDI.W   #2,D7
00018fbe: 6700 fff4                      BEQ      *-$A [$18FB4]
00018fc2: 6100 0054                      BSR      dmaReadMode
00018fc6: 33fc 0001 00ff 8604            MOVE.W   #1,$FF8604
00018fce: 33fc 0080 00ff 8606            MOVE.W   #$80,$FF8606
00018fd6: 3828 000c                      MOVE.W   (12,A0) [$19262],D4
00018fda: 3c3c 00c0                      MOVE.W   #$C0,D6           ;Read Address
00018fde: 2e3c 0004 0000                 MOVE.L   #$40000,D7
00018fe4: 6100 0114                      BSR      wrfdc
00018fe8: 0839 0005 00ff fa01            BTST     #5,$FFFA01
00018ff0: 6700 000c                      BEQ      *+$E [$18FFE]
00018ff4: 5387                           SUBQ.L   #1,D7
00018ff6: 6700 000a                      BEQ      *+$C [$19002]
00018ffa: 6000 ffec                      BRA      *-$12 [$18FE8]
00018ffe: 51cc ffda                      DBF      D4,*-$24 [$18FDA]

fdcTimeout:
00019002: 6100 010a                      BSR      forceInterrupt

fdcDone:
00019006: 33fc 0080 00ff 8606            MOVE.W   #$80,$FF8606
0001900e: 6100 0116                      BSR      readfdc
00019012: 3347 0004                      MOVE.W   D7,(4,A1)
00019016: 4e75                           RTS

dmaReadMode:
00019018: 33fc 0090 00ff 8606            MOVE.W   #$90,$FF8606
00019020: 33fc 0190 00ff 8606            MOVE.W   #$190,$FF8606
00019028: 33fc 0090 00ff 8606            MOVE.W   #$90,$FF8606
00019030: 4e75                           RTS

dmaWriteMode:
00019032: 33fc 0190 00ff 8606            MOVE.W   #$190,$FF8606
0001903a: 33fc 0090 00ff 8606            MOVE.W   #$90,$FF8606
00019042: 33fc 0190 00ff 8606            MOVE.W   #$190,$FF8606
0001904a: 4e75                           RTS

executeFDCcommand:
0001904c: 3c28 000a                      MOVE.W   (10,A0) [$19260],D6
executeFDCCommandD6:
00019050: 4a46                           TST.W    D6
00019052: 6700 0026                      BEQ      fdcRestore
00019056: 33fc 0086 00ff 8606            MOVE.W   #$86,$FF8606
0001905e: 6100 009a                      BSR      wrfdc
00019062: 3c3c 0010                      MOVE.W   #$10,D6
00019066: 6100 0038                      BSR      doFDCCommand
0001906a: 6600 000c                      BNE      *+$E [$19078]
0001906e: 3c28 000a                      MOVE.W   (10,A0) [$19260],D6
00019072: 3386 5000                      MOVE.W   D6,(0,A1,D5.W)    ;store track position for drive
00019076: 4246                           CLR.W    D6
00019078: 4e75                           RTS

fdcRestore:
0001907a: 4246                           CLR.W    D6
0001907c: 6100 0022                      BSR      doFDCCommand
00019080: 0807 0002                      BTST     #2,D7             ;Track 0 indicator
00019084: 0a3c 0004                      EORI     #$4,CCR
00019088: 6600 0014                      BNE      *+$16 [$1909E]
0001908c: 33fc 0082 00ff 8606            MOVE.W   #$82,$FF8606
00019094: 7c00                           MOVEQ    #0,D6
00019096: 6100 0062                      BSR      wrfdc             ;Set Track Register to 0
0001909a: 4271 5000                      CLR.W    (0,A1,D5.W)       ;reset track position for drive
0001909e: 4e75                           RTS

doFDCCommand:
000190a0: 3039 0000 0440                 MOVE.W   $440,D0           ;seekrate (0 - 6ms, 1 - 12ms, 2 - 2ms, 3 - 3ms (default))
000190a6: 0200 0003                      ANDI.B   #3,D0
000190aa: 8c00                           OR.B     D0,D6
000190ac: 2e3c 0004 0000                 MOVE.L   #$40000,D7
000190b2: 33fc 0080 00ff 8606            MOVE.W   #$80,$FF8606      ;FDC Status
000190ba: 6100 006a                      BSR      readfdc
000190be: 0800 0007                      BTST     #7,D0             ;Motor already on?
000190c2: 6600 0008                      BNE      *+$A [$190CC]
000190c6: 2e3c 0006 0000                 MOVE.L   #$60000,D7        ;no => longer timeout
000190cc: 6100 002c                      BSR      wrfdc
000190d0: 5387                           SUBQ.L   #1,D7
000190d2: 6700 001e                      BEQ      *+$20 [$190F2]
000190d6: 0839 0005 00ff fa01            BTST     #5,$FFFA01
000190de: 6600 fff0                      BNE      *-$E [$190D0]
000190e2: 33fc 0080 00ff 8606            MOVE.W   #$80,$FF8606
000190ea: 6100 003a                      BSR      readfdc
000190ee: 4246                           CLR.W    D6
000190f0: 4e75                           RTS
000190f2: 6100 001a                      BSR      forceInterrupt
000190f6: 7c01                           MOVEQ    #1,D6
000190f8: 4e75                           RTS

wrfdc:
000190fa: 6100 0038                      BSR      waitl
000190fe: 33c6 00ff 8604                 MOVE.W   D6,$FF8604
00019104: 6000 002e                      BRA      waitl

delayD7:
00019108: 51cf fffe                      DBF      D7,delayD7
0001910c: 4e75                           RTS

forceInterrupt:
0001910e: 33fc 0080 00ff 8606            MOVE.W   #$80,$FF8606
00019116: 3c3c 00d0                      MOVE.W   #$D0,D6
0001911a: 6100 ffde                      BSR      wrfdc
0001911e: 3e3c 000f                      MOVE.W   #15,D7
00019122: 6100 ffe4                      BSR      delayD7

readfdc:
00019126: 6100 000c                      BSR      waitl
0001912a: 3e39 00ff 8604                 MOVE.W   $FF8604,D7
00019130: 0247 007f                      ANDI.W   #$7F,D7

waitl:
00019134: 40e7                           MOVE     SR,-(A7)
00019136: 3f07                           MOVE.W   D7,-(A7)
00019138: 3e3c 0020                      MOVE.W   #$20,D7
0001913c: 51cf fffe                      DBF      D7,*-$0 [$1913C]
00019140: 3e1f                           MOVE.W   (A7)+,D7
00019142: 46df                           MOVE     (A7)+,SR
00019144: 4e75                           RTS

deselect:
00019146: 3e3c 3a98                      MOVE.W   #15000,D7
0001914a: 6100 ffbc                      BSR      delayD7
0001914e: 103c 0007                      MOVE.B   #7,D0
00019152: 6000 0064                      BRA      *+$66 [$191B8]

select:
00019156: 3028 0000                      MOVE.W   (0,A0),D0
0001915a: 6a00 0006                      BPL      *+$8 [$19162]
0001915e: 6100 007c                      BSR      GEMDOS_Dgetdrv
00019162: 0240 0003                      ANDI.W   #3,D0
00019166: 1a00                           MOVE.B   D0,D5
00019168: 0200 0001                      ANDI.B   #1,D0
0001916c: 5200                           ADDQ.B   #1,D0
0001916e: e308                           LSL.B    #1,D0
00019170: 0805 0001                      BTST     #1,D5
00019174: 6700 0006                      BEQ      *+$8 [$1917C]
00019178: 0000 0001                      ORI.B    #1,D0
0001917c: 0a00 0007                      EORI.B   #7,D0
00019180: 0200 0007                      ANDI.B   #7,D0
00019184: 6100 0032                      BSR      *+$34 [$191B8]
00019188: 13e8 0005 00ff 860d            MOVE.B   (5,A0),$FF860D
00019190: 13e8 0004 00ff 860b            MOVE.B   (4,A0),$FF860B
00019198: 13e8 0003 00ff 8609            MOVE.B   (3,A0),$FF8609
000191a0: 0245 0001                      ANDI.W   #1,D5
000191a4: e34d                           LSL.W    #1,D5
000191a6: 3c31 5000                      MOVE.W   (0,A1,D5.W),D6    ;track position for drive
000191aa: 33fc 0082 00ff 8606            MOVE.W   #$82,$FF8606      ;set track register
000191b2: 6100 ff46                      BSR      wrfdc
000191b6: 4e75                           RTS

000191b8: 40e7                           MOVE     SR,-(A7)
000191ba: 007c 0700                      ORI      #$700,SR
000191be: 13fc 000e 00ff 8800            MOVE.B   #$E,$FF8800
000191c6: 1239 00ff 8800                 MOVE.B   $FF8800,D1
000191cc: 0201 00f8                      ANDI.B   #$F8,D1
000191d0: 8200                           OR.B     D0,D1
000191d2: 13c1 00ff 8802                 MOVE.B   D1,$FF8802
000191d8: 46df                           MOVE     (A7)+,SR
000191da: 4e75                           RTS

GEMDOS_Dgetdrv:
000191dc: 48e7 7ffe                      MOVEM.L  D1-D7/A0-A6,-(A7)
000191e0: 3f3c 0019                      MOVE.W   #$19,-(A7)
000191e4: 4e41                           TRAP     #1
000191e6: 548f                           ADDQ.L   #2,A7
000191e8: 4cdf 7ffe                      MOVEM.L  (A7)+,D1-D7/A0-A6
000191ec: 4e75                           RTS

GEMDOS_SuperOn:
000191ee: 48e7 00c0                      MOVEM.L  A0-A1,-(A7)
000191f2: 2f3c 0000 0001                 MOVE.L   #1,-(A7)
000191f8: 3f3c 0020                      MOVE.W   #$20,-(A7)
000191fc: 4e41                           TRAP     #1
000191fe: dffc 0000 0006                 ADDA.L   #6,A7
00019204: 4a40                           TST.W    D0
00019206: 6600 0016                      BNE      *+$18 [$1921E]
0001920a: 42a7                           CLR.L    -(A7)
0001920c: 3f3c 0020                      MOVE.W   #$20,-(A7)
00019210: 4e41                           TRAP     #1
00019212: dffc 0000 0006                 ADDA.L   #6,A7
00019218: 41fa 0054                      LEA      ($54,PC) [$1926E],A0
0001921c: 2080                           MOVE.L   D0,(A0)
0001921e: 4cdf 0300                      MOVEM.L  (A7)+,A0-A1
00019222: 4e75                           RTS

GEMDOS_SuperOff:
00019224: 48e7 00c0                      MOVEM.L  A0-A1,-(A7)
00019228: 2f3c 0000 0001                 MOVE.L   #1,-(A7)
0001922e: 3f3c 0020                      MOVE.W   #$20,-(A7)
00019232: 4e41                           TRAP     #1
00019234: dffc 0000 0006                 ADDA.L   #6,A7
0001923a: 4a40                           TST.W    D0
0001923c: 6700 0012                      BEQ      *+$14 [$19250]
00019240: 2f3a 002c                      MOVE.L   ($2C,PC) [$1926E],-(A7)
00019244: 3f3c 0020                      MOVE.W   #$20,-(A7)
00019248: 4e41                           TRAP     #1
0001924a: dffc 0000 0006                 ADDA.L   #6,A7
00019250: 4cdf 0300                      MOVEM.L  (A7)+,A0-A1
00019254: 4e75                           RTS

BEFORE:
A0:
0 $19256: DC.W $ffff
2 $19258: DC.L $ffffffff
6 $1925C: DC.W $0001
8 $1925E: DC.W $00e0
10 $19260: DC.W $0000
12 $19262: DC.W $0000

A1:
$19264: DC.W $ffff,$ffff
$19268: DC.W $ffff
$1926A: DC.L $00000000
$1926E: DC.L $00000000
$19272: DC.L $00000200
$19276: DC.L $0
$1927A: DC.L $0001fd88
$1927E: DC.L $695cb2a0
$19282: DC.W $0000


$19284: DC.B "arkanoid.PRO",0
$192AC: DC.B "\x1blDisc error loading file",0
$192C6: DC.B "\x1blThis disc is a copy",0
$192DC: DC.B "\x1blData has been corrupted",0
$192F6: DC.B " - Press a key",0,0

$19306: DC.W $0001,$e1f0,$0000
$1930C: DS.B ...            ;buffer
```
