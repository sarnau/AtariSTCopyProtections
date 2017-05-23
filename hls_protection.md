---
title: "Atari ST Protection: HLS"
date: "2011-09-01"
categories:
   - "software"
   - "atari st"
   - "reverse engineering"
---

This was used in many games, like:
- Roadrunner
- OutRun from Sega
- Music Construction Set
- Metrocross from Epyx
- Skyfox
- The Bards Tale
- Pinball Wizard

It varies with the checksum in $24.w/$26.w (the trace vector) and for Skyfox with the sector numbers that are tested.

Track 79 has 12 sectors in it. 4 are part of the actual protection: Sector 9 and $F7, $F5 and $F6, which are impossible to write into the sector header. The 3 "weird" sectors are semi-defect. The header of sector $F7 is written right before sector 9. After 9, comes 5 (which is empty, filled with 0xE6) and then 0xF5 (also empty with 0xE6 and no read error!), 0xF6 (which continues over the index mark). Sector $F7 has a CRC error, because it contains sector 9. And Sector $F6 has a CRC error, because it continues over the index mark and therefore has garbage at the end.

The protection tests the following things:
Sector 1-9 are read and should return no read-error. They also can't contain the following bytes $A1,$A1,$A1,$FE or $FB (which is a sector header or data header with sync) and they also have to be 512 bytes long. These tests avoid that somebody creates a different - copyable - track 79 with shorter sectors, etc.

The other 3 sectors either return no error or a CRC error (error code -4). The software calculates a checksum over these 3 sectors plus sector 9 as well as a checksum (based on the seed in $24.w/$26.w) over the protection code. This checksum is then used to decrypt the main application. If any of the above tests fail, the memory is cleared of the code and the app is terminated.

Interestingly the protection can be defeated with a simple Floprd() patch which returns the right buffer for the 4 sectors...

``` 68k
L0000 DC.W $000,$F7
    DC.W $400,$F5
    DC.W $600,$f6
```

for Skyfox for values are:

``` 68k
L0000 DC.W $000,$43
    DC.W $400,$21
    DC.W $600,$09
```

``` 68k
05BC94  48790001FFFC        L0001:PEA       $1FFFC
05BC9A  3F3C0020                  MOVE.W    #$20,-(A7)
05BC9E  4E41                      TRAP      #1
05BCA0  61000FA0                  BSR       4000(PC)             L0025
05BCA4  4240                      CLR.W     D0
05BCA6  6100                      BSR       ...
05BCAA  7000                SuperOn: MOVEQ     #0,D0
05BCAA  2F00                SuperOff:MOVE.L    D0,-(A7)
05BCAC  3F3C0020                  MOVE.W    #$20,-(A7)
05BCB0  4E41                      TRAP      #1
05BCB2  5C8F                      ADDQ.L    #6,A7
05BCB4  4E75                      RTS

05BCB6  204F                      MOVEA.L   A7,A0
05BCB8  48E7FF7E                  MOVEM.L   A1-A6/D0-D7,-(A7)
05BCBC  2C48                      MOVEA.L   A0,A6
05BCBE  61E8                      BSR.S     -24(PC)              SuperOn
05BCC0  40C6                      MOVE      SR,D6
05BCC2  2E00                      MOVE.L    D0,D7
05BCC4  007C0700                  ORI.W     #$700,SR
05BCC8  43FAFFBE                  LEA       -66(PC),A1           L0000
05BCCC  45FA000C                  LEA       12(PC),A2            decryptBlock
05BCD0  323C0105                  MOVE.W    #$105,D1
05BCD4  3019                decryptBlockLoop:MOVE.W    (A1)+,D0
05BCD6  B15A                      EOR.W     D0,(A2)+
05BCD8  51C9FFFA                  DBF       D1,-6(PC)            decryptBlockLoop
05BCDC                      decryptBlock EQU       *-2
05BCDC  46C6                      MOVE      D6,SR
05BCDE  2C6E0004                  MOVEA.L   4(A6),A6
05BCE2  2A6E0018                  MOVEA.L   24(A6),A5
05BCE6  91C8                      SUBA.L    A0,A0
05BCE8  317C40650024              MOVE.W    #$4065,36(A0)       ;$24
05BCEE  317C32070026              MOVE.W    #$3207,38(A0)       ;$26
05BCF4  2007                      MOVE.L    D7,D0
05BCF6  61B2                      BSR.S     -78(PC)              SuperOff

05BCF8  7800                      MOVEQ     #0,D4               ;drive = A
05BCFA  61000080            L0006:BSR       128(PC)              flush9Buffers

05BCFE  7401                      MOVEQ     #1,D2               ;start at setor 1
05BD00  7609                      MOVEQ     #9,D3               ;9 sectors
05BD02  204D                      MOVEA.L   A5,A0               ;$1200 bytes
05BD04  6100008C                  BSR       140(PC)              readSectors
05BD08  6666                      BNE.S     102(PC)              L000D => read error

;check that all 9 sectors don't contain 0xA1,0xA1,0xA1 + sync header
;_and_ that all were 512 bytes long.
05BD0A  204D                      MOVEA.L   A5,A0               ;sector buffer start
05BD0C  7408                      MOVEQ     #8,D2               ;9 sectors
05BD0E  303CA1A1                  MOVE.W    #$A1A1,D0
05BD12  323C00FF            L0007:MOVE.W    #$FF,D1
05BD16  B058                L0008:CMP.W     (A0)+,D0            ;search for first sync inside the sector
05BD18  57C9FFFC                  DBEQ      D1,-4(PC)            L0008
05BD1C  6624                      BNE.S     36(PC)               L000A
05BD1E  16280001                  MOVE.B    1(A0),D3            ;0xFE = address header
05BD22  B010                      CMP.B     (A0),D0             ;another sync?
05BD24  6708                      BEQ.S     8(PC)                L0009
05BD26  1610                      MOVE.B    (A0),D3             ;address header
05BD28  B029FFFD                  CMP.B     -3(A1),D0           ;sync before?
05BD2C  66E8                      BNE.S     -24(PC)              L0008 => continue searching
05BD2E  B63C00FE            L0009:CMP.B     #$FE,D3             ;address header found?
05BD32  670001A0                  BEQ       416(PC)              eraseAndTerminate
05BD36  B63C00FB                  CMP.B     #$FB,D3             ;data header found?
05BD3A  67000198                  BEQ       408(PC)              eraseAndTerminate
05BD3E  5341                      SUBQ.W    #1,D1
05BD40  6AD4                      BPL.S     -44(PC)              L0008
05BD42  0CA8484C5320FFFC    L000A:CMPI.L    #$484C5320,-4(A0)   ;last long in the sector is "HLS "?
05BD4A  57CAFFC6                  DBEQ      D2,-58(PC)           L0007
05BD4E  67000184                  BEQ       388(PC)              eraseAndTerminate

The header of sector 0xF7 is written right before sector 0x09.
After 0x09, comes 0x05 (which is empty, filled with 0xE6) and then 0xF5 (also empty with 0xE6 and no read error!), 0xF6 (which continues over the index mark)

TrackLength:0x0018d0, fuzzySectorLength:0x000000, SectorCount:12 Flags:0x0001 TrackBytes:0x187f Track:0x4f
SectorOffset:0x0000, PosTiming:  592, ReadTiming:    0 [Address Block: Track:0x4f Side:0x00 Sector:0x01 SectorSize:0x02 CRC:0x701d=OK] FDCStatus:0x00 Flags:0x00
SectorOffset:0x0200, PosTiming: 5248, ReadTiming:    0 [Address Block: Track:0x4f Side:0x00 Sector:0x06 SectorSize:0x02 CRC:0xe98a=OK] FDCStatus:0x00 Flags:0x00
SectorOffset:0x0400, PosTiming: 9904, ReadTiming:    0 [Address Block: Track:0x4f Side:0x00 Sector:0x02 SectorSize:0x02 CRC:0x254e=OK] FDCStatus:0x00 Flags:0x00
SectorOffset:0x0600, PosTiming:14561, ReadTiming:    0 [Address Block: Track:0x4f Side:0x00 Sector:0x07 SectorSize:0x02 CRC:0xdabb=OK] FDCStatus:0x00 Flags:0x00
SectorOffset:0x0800, PosTiming:19217, ReadTiming:    0 [Address Block: Track:0x4f Side:0x00 Sector:0x03 SectorSize:0x02 CRC:0x167f=OK] FDCStatus:0x00 Flags:0x00
SectorOffset:0x0a00, PosTiming:23873, ReadTiming:    0 [Address Block: Track:0x4f Side:0x00 Sector:0x08 SectorSize:0x02 CRC:0xca85=OK] FDCStatus:0x00 Flags:0x00
SectorOffset:0x0c00, PosTiming:28530, ReadTiming:    0 [Address Block: Track:0x4f Side:0x00 Sector:0x04 SectorSize:0x02 CRC:0x8fe8=OK] FDCStatus:0x00 Flags:0x00
SectorOffset:0x0e00, PosTiming:33186, ReadTiming:    0 [Address Block: Track:0x4f Side:0x00 Sector:0xf7 SectorSize:0x02 CRC:0xc97a=OK] FDCStatus:0x08 Flags:0x00
SectorOffset:0x1000, PosTiming:33904, ReadTiming:    0 [Address Block: Track:0x4f Side:0x00 Sector:0x09 SectorSize:0x02 CRC:0xf9b4=OK] FDCStatus:0x00 Flags:0x00
SectorOffset:0x1200, PosTiming:38560, ReadTiming:    0 [Address Block: Track:0x4f Side:0x00 Sector:0x05 SectorSize:0x02 CRC:0xbcd9=OK] FDCStatus:0x00 Flags:0x00
SectorOffset:0x1400, PosTiming:43216, ReadTiming:    0 [Address Block: Track:0x4f Side:0x00 Sector:0xf5 SectorSize:0x02 CRC:0xaf18=OK] FDCStatus:0x00 Flags:0x00
SectorOffset:0x1600, PosTiming:47873, ReadTiming:    0 [Address Block: Track:0x4f Side:0x00 Sector:0xf6 SectorSize:0x02 CRC:0xfa4b=OK] FDCStatus:0x08 Flags:0x00

;now read the 3 other sectors
05BD52  47FAFF34                  LEA       -204(PC),A3          L0000
05BD56  7E02                      MOVEQ     #2,D7
05BD58  301B                L000B:MOVE.W    (A3)+,D0            ;buffer offset
05BD5A  341B                      MOVE.W    (A3)+,D2            ;sector number
05BD5C  41F50000                  LEA       0(A5,D0.W),A0
05BD60  7601                      MOVEQ     #1,D3               ;1 sector
05BD62  612E                      BSR.S     46(PC)               readSectors
05BD64  6704                      BEQ.S     4(PC)                L000C
05BD66  B07CFFFC                  CMP.W     #-4,D0              ;CRC error
05BD6A  56CFFFEC            L000C:DBNE      D7,-20(PC)           L000B
05BD6E  6742                      BEQ.S     66(PC)               L0011 => protection OK!
05BD70  5244                L000D:ADDQ.W    #1,D4               ;drive + 1
05BD72  B87C0002                  CMP.W     #2,D4               ;drive = B?
05BD76  6D82                      BLT.S     -126(PC)             L0006
05BD78  6000015A                  BRA       346(PC)              eraseAndTerminate

05BD7C  204D                flush9Buffers:MOVEA.L   A5,A0
05BD7E  7208                      MOVEQ     #8,D1
05BD80  203C484C5320              MOVE.L    #$484C5320,D0       ;(#"HLS ")
05BD86  D0FC01FC            L000F:ADDA.W    #512-sizeof(long),A0
05BD8A  20C0                      MOVE.L    D0,(A0)+
05BD8C  51C9FFF8                  DBF       D1,-8(PC)            L000F
05BD90  4E75                      RTS

05BD92  704F                readSectors:MOVEQ     #$4F,D0        ;track = 79
05BD94  7200                      MOVEQ     #0,D1               ;side = 0
05BD96  3F03                      MOVE.W    D3,-(A7)            ;int16_t count
05BD98  3F01                      MOVE.W    D1,-(A7)            ;int16_t sideno
05BD9A  3F00                      MOVE.W    D0,-(A7)            ;int16_t trackno
05BD9C  3F02                      MOVE.W    D2,-(A7)            ;int16_t sectno
05BD9E  3F04                      MOVE.W    D4,-(A7)            ;int16_t devno
05BDA0  42A7                      CLR.L     -(A7)               ;int32_t filler
05BDA2  2F08                      MOVE.L    A0,-(A7)            ;void *buf
05BDA4  3F3C0008                  MOVE.W    #8,-(A7)
05BDA8  4E4E                      TRAP      #14
05BDAA  DEFC0014                  ADDA.W    #20,A7
05BDAE  4A80                      TST.L     D0
05BDB0  4E75                      RTS


05BDB2  41ED1000            L0011:LEA       $1000(A5),A0
05BDB6  43ED0200                  LEA       $200(A5),A1
05BDBA  303C007F                  MOVE.W    #$7F,D0
05BDBE  22D8                L0012:MOVE.L    (A0)+,(A1)+         ;Copy Sector 9 onto Sector 2
05BDC0  51C8FFFC                  DBF       D0,-4(PC)            L0012

Buffer after the reading:
A5: 0x0000..0x01FF Sector 0xF7
    0x0200..0x03FF Sector 9     (was sector 2)
    0x0400..0x05FF Sector 0xF5
    0x0600..0x07FF Sector 0xF6
    ;the following sectors are not used for the checksum:
    0x0800..0x09FF Sector 5
    0x0A00..0x0BFF Sector 6
    0x0C00..0x0DFF Sector 7
    0x0E00..0x0FFF Sector 8
    0x1000..0x11FF Sector 9

05BDC4  6100FEE2                  BSR       -286(PC)             SuperOn
05BDC8  2F00                      MOVE.L    D0,-(A7)

05BDCA  43F9FFFFEBAC              LEA       $FFFFEBAC.L,A1
05BDD0  204D                      MOVEA.L   A5,A0
05BDD2  3A3C0349                  MOVE.W    #$349,D5            ;842 words (3 sectors and 148/$94 bytes of the last ~60 bytes before index impulse triggers)
05BDD6  30291478                  MOVE.W    5240(A1),D0         ;$24 = $4065
05BDDA  5345                      SUBQ.W    #1,D5
05BDDC  3218                L0013:MOVE.W    (A0)+,D1
05BDDE  780F                      MOVEQ     #15,D4              ;16 bits per word
05BDE0  7400                L0014:MOVEQ     #0,D2
05BDE2  E349                      LSL.W     #1,D1
05BDE4  E252                      ROXR.W    #1,D2
05BDE6  B540                      EOR.W     D2,D0
05BDE8  E348                      LSL.W     #1,D0
05BDEA  6406                      BCC.S     6(PC)                L0015
05BDEC  3629147A                  MOVE.W    5242(A1),D3          ;$26 = $3207
05BDF0  B740                      EOR.W     D3,D0
05BDF2  51CCFFEC            L0015:DBF       D4,-20(PC)           L0014
05BDF6  51CDFFE4                  DBF       D5,-28(PC)           L0013
;checksum in D0

;Calculate the checksum over the protection code
05BDFA  41FAFE8C                  LEA       -372(PC),A0          L0000
05BDFE  3A3A00FE                  MOVE.W    254(PC),D5           L0024
05BE02  3340147A                  MOVE.W    D0,5242(A1)          ;$26 = contains last checksum
05BE06  3003                      MOVE.W    D3,D0
05BE08  5345                      SUBQ.W    #1,D5
05BE0A  3218                L0016:MOVE.W    (A0)+,D1
05BE0C  780F                      MOVEQ     #15,D4
05BE0E  7400                L0017:MOVEQ     #0,D2
05BE10  E349                      LSL.W     #1,D1
05BE12  E252                      ROXR.W    #1,D2
05BE14  B540                      EOR.W     D2,D0
05BE16  E348                      LSL.W     #1,D0
05BE18  6406                      BCC.S     6(PC)                L0018
05BE1A  3629147A                  MOVE.W    5242(A1),D3          ;$26 = $3207
05BE1E  B740                      EOR.W     D3,D0
05BE20  51CCFFEC            L0018:DBF       D4,-20(PC)           L0017
05BE24  51CDFFE4                  DBF       D5,-28(PC)           L0016
05BE28  3600                      MOVE.W    D0,D3                ;new checksum in D3


;copy code fragment to our function parameter
05BE2A  41FAFE68                  LEA       -408(PC),A0          L0001
05BE2E  226E0008                  MOVEA.L   8(A6),A1
05BE32  7209                      MOVEQ     #9,D1
05BE34  32D8                L0019:MOVE.W    (A0)+,(A1)+
05BE36  51C9FFFC                  DBF       D1,-4(PC)            L0019

;decrypt code with the checksum in D3
05BE3A  243C000045D2              MOVE.L    #$45D2,D2
05BE40  2802                      MOVE.L    D2,D4
05BE42  4844                      SWAP      D4
05BE44  023C0000                  ANDI.B    #0,CCR
05BE48  600C                      BRA.S     12(PC)               L001B
05BE4A  3019                L001A:MOVE.W    (A1)+,D0
05BE4C  40C1                      MOVE      SR,D1
05BE4E  B340                      EOR.W     D1,D0
05BE50  B740                      EOR.W     D3,D0
05BE52  B151                      EOR.W     D0,(A1)
05BE54  E253                      ROXR.W    #1,D3
05BE56  51CAFFF2            L001B:DBF       D2,-14(PC)           L001A
05BE5A  51CCFFEE                  DBF       D4,-18(PC)           L001A

05BE5E  201F                      MOVE.L    (A7)+,D0
05BE60  6100FE48                  BSR       -440(PC)             SuperOff

;relocate binary
05BE64  2A3C00000630              MOVE.L    #$630,D5
05BE6A  9BAE0014                  SUB.L     D5,20(A6)
05BE6E  9BAE0018                  SUB.L     D5,24(A6)
05BE72  203C00000000              MOVE.L    #0,D0
05BE78  2D40001C                  MOVE.L    D0,28(A6)
05BE7C  206E0008                  MOVEA.L   8(A6),A0
05BE80  226E0018                  MOVEA.L   24(A6),A1
05BE84  2008                      MOVE.L    A0,D0
05BE86  D1D9                      ADDA.L    (A1)+,A0
05BE88  B1C0                      CMPA.L    D0,A0
05BE8A  671C                      BEQ.S     28(PC)               L001F
05BE8C  7200                      MOVEQ     #0,D1
05BE8E  D190                L001C:ADD.L     D0,(A0)
05BE90  1219                L001D:MOVE.B    (A1)+,D1
05BE92  B27C0002                  CMP.W     #2,D1
05BE96  6504                      BCS.S     4(PC)                L001E
05BE98  D0C1                      ADDA.W    D1,A0
05BE9A  60F2                      BRA.S     -14(PC)              L001C
05BE9C  B27C0000            L001E:CMP.W     #0,D1
05BEA0  6706                      BEQ.S     6(PC)                L001F
05BEA2  D0FC00FE                  ADDA.W    #254,A0
05BEA6  60E8                      BRA.S     -24(PC)              L001D

05BEA8  43FA002A            L001F:LEA       42(PC),A1            eraseAndTerminate
05BEAC  244F                      MOVEA.L   A7,A2
05BEAE  7206                      MOVEQ     #6,D1
05BEB0  3F21                L0020:MOVE.W    -(A1),-(A7)
05BEB2  51C9FFFC                  DBF       D1,-4(PC)            L0020
05BEB6  DABC00001208              ADD.L     #$1208,D5
05BEBC  E48D                      LSR.L     #2,D5
05BEBE  2040                      MOVEA.L   D0,A0               ;A0 = 0xCEBA
05BEC0  226E0018                  MOVEA.L   24(A6),A1
05BEC4  4ED7                      JMP       (A7)

05BEC6  4299                L0021:CLR.L     (A1)+
05BEC8  51CDFFFC                  DBF       D5,-4(PC)            L0021
05BECC  2E4A                      MOVEA.L   A2,A7
05BECE  4CDF7EFF                  MOVEM.L   (A7)+,A1-A6/D0-D7
05BED2  4ED0                      JMP       (A0)

05BED4  323C0095            eraseAndTerminate:MOVE.W    #$95,D1
05BED8  43FAFDAE                  LEA       -594(PC),A1          L0000
05BEDC  4299                L0023:CLR.L     (A1)+
05BEDE  51C9FFFC                  DBF       D1,-4(PC)            L0023
05BEE2  42A7                      CLR.L     -(A7)
05BEE4  4E41                      TRAP      #1
05BEE6  23F974105542465951CA      MOVE.L    $74105542.L,$465951CA.L
05BEF0  FFFC                      DC.B      $FF,$FC
05BEF2  43FAFD94                  LEA       -620(PC),A1          L0000
05BEF6  32BC0000                  MOVE.W    #0,(A1)
05BEFA  6000FDD8                  BRA       -552(PC)             decryptBlockLoop
05BEFE  013BF7F7F7F7F7F7    L0024:DC.B      $01,$3B,$F7,$F7,$F7,$F7,$F7,$F7
05BF06  F7F7F7F7F7F7F7F7          DC.B      $F7,$F7,$F7,$F7,$F7,$F7,$F7,$F7
05BF0E  F7F7F7F7F7F7F7F7          DC.B      $F7,$F7,$F7,$F7,$F7,$F7,$F7,$F7
05BF16  F7F7F7F7F7F7F7F7          DC.B      $F7,$F7,$F7,$F7,$F7,$F7,$F7,$F7
05BF1E  0000000000000000          DC.B      $00,$00,$00,$00,$00,$00,$00,$00
05BF26  00000000A1A1A1FE          DC.B      $00,$00,$00,$00,$A1,$A1,$A1,$FE
05BF2E  4F000902F9B4F7F7          DC.B      $4F,$00,$09,$02,$F9,$B4,$F7,$F7
05BF36  F7F7F7F7F7F7F7F7          DC.B      $F7,$F7,$F7,$F7,$F7,$F7,$F7,$F7
05BF3E  F7F7F7F7F7F7F7F7          DC.B      $F7,$F7,$F7,$F7,$F7,$F7,$F7,$F7
05BF46  F7F7F7F700000000          DC.B      $F7,$F7,$F7,$F7,$00,$00,$00,$00
05BF4E  0000000000000000          DC.B      $00,$00,$00,$00,$00,$00,$00,$00
05BF56  A1A1A1FBF0F1F2F3          DC.B      $A1,$A1,$A1,$FB,$F0,$F1,$F2,$F3
05BF5E  F4F5F6F7F8F9FAFB          DC.B      $F4,$F5,$F6,$F7,$F8,$F9,$FA,$FB
05BF66  FCFDFEFF484C5320          DC.B      $FC,$FD,$FE,$FF,$48,$4C,$53,$20
05BF6E  4455504C49434154          DC.B      $44,$55,$50,$4C,$49,$43,$41,$54
05BF76  494F4EF7F7F7F7F7          DC.B      $49,$4F,$4E,$F7,$F7,$F7,$F7,$F7
05BF7E  F7F7F7F7F7F7F7F7          DC.B      $F7,$F7,$F7,$F7,$F7,$F7,$F7,$F7
05BF86  F7F7F7F7F7F7F7F7          DC.B      $F7,$F7,$F7,$F7,$F7,$F7,$F7,$F7
05BF8E  F7F7F7F7F7F7F7F7          DC.B      $F7,$F7,$F7,$F7,$F7,$F7,$F7,$F7
05BF96  F7F7F7F7F7F7F7F7          DC.B      $F7,$F7,$F7,$F7,$F7,$F7,$F7,$F7
05BF9E  F7F7F7F7F7F7F7F7          DC.B      $F7,$F7,$F7,$F7,$F7,$F7,$F7,$F7
05BFA6  F7F7F7F7F7F7F7F7          DC.B      $F7,$F7,$F7,$F7,$F7,$F7,$F7,$F7
05BFAE  F7F7F7F7F7F7F7F7          DC.B      $F7,$F7,$F7,$F7,$F7,$F7,$F7,$F7
05BFB6  F7F7F7F7F7F7F7F7          DC.B      $F7,$F7,$F7,$F7,$F7,$F7,$F7,$F7
05BFBE  F7F7F7F7F7F7F7F7          DC.B      $F7,$F7,$F7,$F7,$F7,$F7,$F7,$F7
05BFC6  F7F7F7F7F7F7F7F7          DC.B      $F7,$F7,$F7,$F7,$F7,$F7,$F7,$F7
05BFCE  F7F7F7F7F7F7F7F7          DC.B      $F7,$F7,$F7,$F7,$F7,$F7,$F7,$F7
05BFD6  F7F7F7F7F7F7F7F7          DC.B      $F7,$F7,$F7,$F7,$F7,$F7,$F7,$F7
05BFDE  F7F7F7F7F7F7F7F7          DC.B      $F7,$F7,$F7,$F7,$F7,$F7,$F7,$F7
05BFE6  F7F7F7F7F7F7F7F7          DC.B      $F7,$F7,$F7,$F7,$F7,$F7,$F7,$F7
05BFEE  F7F7F7F7F7F7F7F7          DC.B      $F7,$F7,$F7,$F7,$F7,$F7,$F7,$F7
05BFF6  F7F7F7F7F7F7F7F7          DC.B      $F7,$F7,$F7,$F7,$F7,$F7,$F7,$F7
05BFFE  F7F7F7F7F7F7F7F7          DC.B      $F7,$F7,$F7,$F7,$F7,$F7,$F7,$F7
05C006  F7F7F7F7F7F7F7F7          DC.B      $F7,$F7,$F7,$F7,$F7,$F7,$F7,$F7
05C00E  F7F7F7F7F7F7F7F7          DC.B      $F7,$F7,$F7,$F7,$F7,$F7,$F7,$F7
05C016  F7F7F7F7F7F7F7F7          DC.B      $F7,$F7,$F7,$F7,$F7,$F7,$F7,$F7
05C01E  F7F7F7F7F7F7F7F7          DC.B      $F7,$F7,$F7,$F7,$F7,$F7,$F7,$F7
05C026  F7F7F7F7F7F7F7F7          DC.B      $F7,$F7,$F7,$F7,$F7,$F7,$F7,$F7
05C02E  F7F7F7F7F7F7F7F7          DC.B      $F7,$F7,$F7,$F7,$F7,$F7,$F7,$F7
05C036  F7F7F7F7F7F7F7F7          DC.B      $F7,$F7,$F7,$F7,$F7,$F7,$F7,$F7
05C03E  F7F7F7F7F7F7F7F7          DC.B      $F7,$F7,$F7,$F7,$F7,$F7,$F7,$F7
05C046  F7F7F7F7F7F7F7F7          DC.B      $F7,$F7,$F7,$F7,$F7,$F7,$F7,$F7
05C04E  F7F7F7F7F7F7F7F7          DC.B      $F7,$F7,$F7,$F7,$F7,$F7,$F7,$F7
05C056  F7F7F7F7F7F7F7F7          DC.B      $F7,$F7,$F7,$F7,$F7,$F7,$F7,$F7
05C05E  F7F7F7F7F7F7F7F7          DC.B      $F7,$F7,$F7,$F7,$F7,$F7,$F7,$F7
05C066  F7F7F7F7F7F7F7F7          DC.B      $F7,$F7,$F7,$F7,$F7,$F7,$F7,$F7
05C06E  F7F7F7F7F7F7F7F7          DC.B      $F7,$F7,$F7,$F7,$F7,$F7,$F7,$F7
05C076  F7F7F7F7F7F7F7F7          DC.B      $F7,$F7,$F7,$F7,$F7,$F7,$F7,$F7
05C07E  F7F7F7F7F7F7F7F7          DC.B      $F7,$F7,$F7,$F7,$F7,$F7,$F7,$F7
05C086  F7F7F7F7F7F7F7F7          DC.B      $F7,$F7,$F7,$F7,$F7,$F7,$F7,$F7
05C08E  F7F7F7F7F7F7F7F7          DC.B      $F7,$F7,$F7,$F7,$F7,$F7,$F7,$F7
05C096  F7F7F7F7F7F7F7F7          DC.B      $F7,$F7,$F7,$F7,$F7,$F7,$F7,$F7
05C09E  F7F7F7F7F7F7F7F7          DC.B      $F7,$F7,$F7,$F7,$F7,$F7,$F7,$F7
05C0A6  F7F7F7F7F7F7F7F7          DC.B      $F7,$F7,$F7,$F7,$F7,$F7,$F7,$F7
05C0AE  F7F7F7F7F7F7F7F7          DC.B      $F7,$F7,$F7,$F7,$F7,$F7,$F7,$F7
05C0B6  F7F7F7F7F7F7F7F7          DC.B      $F7,$F7,$F7,$F7,$F7,$F7,$F7,$F7
05C0BE  F7F7F7F7F7F7F7F7          DC.B      $F7,$F7,$F7,$F7,$F7,$F7,$F7,$F7
05C0C6  F7F7F7F7F7F7F7F7          DC.B      $F7,$F7,$F7,$F7,$F7,$F7,$F7,$F7
05C0CE  F7F7F7F7F7F7F7F7          DC.B      $F7,$F7,$F7,$F7,$F7,$F7,$F7,$F7
05C0D6  F7F7F7F7F7F7F7F7          DC.B      $F7,$F7,$F7,$F7,$F7,$F7,$F7,$F7
05C0DE  F7F7F7F7F7F7F7F7          DC.B      $F7,$F7,$F7,$F7,$F7,$F7,$F7,$F7
05C0E6  F7F7F7F7F7F7F7F7          DC.B      $F7,$F7,$F7,$F7,$F7,$F7,$F7,$F7
05C0EE  F7F7F7F7F7F7F7F7          DC.B      $F7,$F7,$F7,$F7,$F7,$F7,$F7,$F7
05C0F6  F7F7F7F7F7F7F7F7          DC.B      $F7,$F7,$F7,$F7,$F7,$F7,$F7,$F7
05C0FE  F7F79EFC24F42A3C          DC.B      $F7,$F7,$9E,$FC,$24,$F4,$2A,$3C
05C106  1208E2BDBBEE227A          DC.B      $12,$08,$E2,$BD,$BB,$EE,$22,$7A
05C10E  9BB64ECF62A551CD          DC.B      $9B,$B6,$4E,$CF,$62,$A5,$51,$CD
05C116  FFFC030A4CC35E91          DC.B      $FF,$FC,$03,$0A,$4C,$C3,$5E,$91
05C11E  4ED81052008D63F2          DC.B      $4E,$D8,$10,$52,$00,$8D,$63,$F2
05C126  2C77F35936D58DFC          DC.B      $2C,$77,$F3,$59,$36,$D5,$8D,$FC
05C12E  93375C5823F97410          DC.B      $93,$37,$5C,$58,$23,$F9,$74,$10
05C136  5542465951CAFFFC          DC.B      $55,$42,$46,$59,$51,$CA,$FF,$FC
05C13E  43FAFD9432BC0000          DC.B      $43,$FA,$FD,$94,$32,$BC,$00,$00
05C146  6000FDD8013B0000          DC.B      $60,$00,$FD,$D8,$01,$3B,$00,$00
05C14E  00100000013FE6E6          DC.B      $00,$10,$00,$00,$01,$3F,$E6,$E6
05C156  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05C15E  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05C166  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05C16E  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05C176  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05C17E  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05C186  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05C18E  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05C196  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05C19E  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05C1A6  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05C1AE  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05C1B6  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05C1BE  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05C1C6  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05C1CE  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05C1D6  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05C1DE  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05C1E6  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05C1EE  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05C1F6  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05C1FE  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05C206  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05C20E  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05C216  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05C21E  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05C226  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05C22E  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05C236  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05C23E  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05C246  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05C24E  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05C256  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05C25E  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05C266  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05C26E  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05C276  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05C27E  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05C286  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05C28E  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05C296  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05C29E  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05C2A6  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05C2AE  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05C2B6  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05C2BE  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05C2C6  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05C2CE  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05C2D6  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05C2DE  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05C2E6  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05C2EE  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05C2F6  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05C2FE  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05C306  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05C30E  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05C316  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05C31E  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05C326  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05C32E  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05C336  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05C33E  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05C346  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05C34E  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05C356  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05C35E  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05C366  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05C36E  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05C376  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05C37E  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05C386  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05C38E  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05C396  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05C39E  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05C3A6  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05C3AE  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05C3B6  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05C3BE  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05C3C6  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05C3CE  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05C3D6  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05C3DE  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05C3E6  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05C3EE  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05C3F6  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05C3FE  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05C406  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05C40E  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05C416  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05C41E  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05C426  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05C42E  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05C436  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05C43E  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05C446  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05C44E  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05C456  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05C45E  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05C466  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05C46E  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05C476  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05C47E  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05C486  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05C48E  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05C496  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05C49E  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05C4A6  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05C4AE  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05C4B6  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05C4BE  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05C4C6  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05C4CE  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05C4D6  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05C4DE  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05C4E6  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05C4EE  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05C4F6  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05C4FE  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05C506  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05C50E  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05C516  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05C51E  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05C526  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05C52E  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05C536  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05C53E  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05C546  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05C54E  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05C556  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05C55E  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05C566  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05C56E  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05C576  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05C57E  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05C586  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05C58E  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05C596  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05C59E  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05C5A6  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05C5AE  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05C5B6  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05C5BE  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05C5C6  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05C5CE  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05C5D6  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05C5DE  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05C5E6  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05C5EE  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05C5F6  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05C5FE  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05C606  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05C60E  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05C616  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05C61E  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05C626  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05C62E  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05C636  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05C63E  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05C646  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05C64E  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05C656  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05C65E  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05C666  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05C66E  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05C676  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05C67E  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05C686  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05C68E  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05C696  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05C69E  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05C6A6  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05C6AE  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05C6B6  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05C6BE  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05C6C6  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05C6CE  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05C6D6  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05C6DE  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05C6E6  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05C6EE  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05C6F6  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05C6FE  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05C706  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05C70E  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05C716  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05C71E  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05C726  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05C72E  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05C736  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05C73E  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05C746  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05C74E  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05C756  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05C75E  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05C766  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05C76E  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05C776  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05C77E  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05C786  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05C78E  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05C796  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05C79E  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05C7A6  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05C7AE  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05C7B6  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05C7BE  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05C7C6  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05C7CE  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05C7D6  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05C7DE  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05C7E6  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05C7EE  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05C7F6  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05C7FE  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05C806  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05C80E  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05C816  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05C81E  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05C826  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05C82E  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05C836  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05C83E  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05C846  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05C84E  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05C856  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05C85E  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05C866  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05C86E  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05C876  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05C87E  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05C886  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05C88E  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05C896  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05C89E  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05C8A6  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05C8AE  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05C8B6  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05C8BE  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05C8C6  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05C8CE  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05C8D6  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05C8DE  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05C8E6  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05C8EE  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05C8F6  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05C8FE  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05C906  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05C90E  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05C916  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05C91E  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05C926  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05C92E  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05C936  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05C93E  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05C946  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05C94E  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05C956  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05C95E  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05C966  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05C96E  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05C976  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05C97E  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05C986  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05C98E  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05C996  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05C99E  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05C9A6  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05C9AE  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05C9B6  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05C9BE  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05C9C6  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05C9CE  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05C9D6  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05C9DE  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05C9E6  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05C9EE  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05C9F6  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05C9FE  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05CA06  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05CA0E  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05CA16  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05CA1E  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05CA26  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05CA2E  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05CA36  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05CA3E  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05CA46  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05CA4E  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05CA56  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05CA5E  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05CA66  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05CA6E  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05CA76  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05CA7E  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05CA86  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05CA8E  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05CA96  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05CA9E  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05CAA6  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05CAAE  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05CAB6  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05CABE  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05CAC6  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05CACE  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05CAD6  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05CADE  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05CAE6  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05CAEE  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05CAF6  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05CAFE  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05CB06  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05CB0E  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05CB16  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05CB1E  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05CB26  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05CB2E  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05CB36  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05CB3E  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05CB46  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05CB4E  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05CB56  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05CB5E  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05CB66  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05CB6E  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05CB76  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05CB7E  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05CB86  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05CB8E  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05CB96  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05CB9E  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05CBA6  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05CBAE  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05CBB6  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05CBBE  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05CBC6  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05CBCE  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05CBD6  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05CBDE  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05CBE6  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05CBEE  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05CBF6  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05CBFE  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05CC06  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05CC0E  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05CC16  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05CC1E  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05CC26  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05CC2E  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05CC36  E6E6E6E6E6E6E6E6          DC.B      $E6,$E6,$E6,$E6,$E6,$E6,$E6,$E6
05CC3E  E6E6E6E6                  DC.B      $E6,$E6,$E6,$E6
```
