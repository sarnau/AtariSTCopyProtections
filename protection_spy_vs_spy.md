---
title: "Atari ST Protection: Spy Vs Spy from Databyte"
date: "2011-09-01"
categories:
   - "software"
   - "atari st"
   - "reverse engineering"
---

Another simple, yet efficient protection. Sector 9 in Track 79 is read twice, the EOR checksum has to change, done via "weak" or "fuzzy" bits in the sector. Again: easily defeated with a Floprd() patch.

``` 68k
move.w #$9,d0                                    ; 00CEC0: 303C 0009
bsr .l +$28 {$00CEEE}                            ; 00CEC4: 6100 0028
move.b d0,$cf2c                                  ; 00CEC8: 13C0 0000 CF2C
move.w #$9,d0                                    ; 00CECE: 303C 0009
bsr .l +$1a {$00CEEE}                            ; 00CED2: 6100 001A
move.b $cf2c,d1                                  ; 00CED6: 1239 0000 CF2C
cmp.b d0,d1                                      ; 00CEDC: B200
beq .l +$8 {$00CEE8}                             ; 00CEDE: 6700 0008
move.b #$0,d0                                    ; 00CEE2: 103C 0000
rts                                              ; 00CEE6: 4E75
move.b #$1,d0                                    ; 00CEE8: 103C 0001
rts                                              ; 00CEEC: 4E75
move.w #$1,-(a7)                                 ; 00CEEE: 3F3C 0001
move.w #$0,-(a7)                                 ; 00CEF2: 3F3C 0000
move.w #$4f,-(a7)                                ; 00CEF6: 3F3C 004F
move.w d0,-(a7)                                  ; 00CEFA: 3F00
move.w #$0,-(a7)                                 ; 00CEFC: 3F3C 0000
clr.l -(a7)                                      ; 00CF00: 42A7
move.l #$cf2e,-(a7)                              ; 00CF02: 2F3C 0000 CF2E
move.w #$8,-(a7)                                 ; 00CF08: 3F3C 0008
trap #14                                         ; 00CF0C: 4E4E
adda.l #$14,a7                                   ; 00CF0E: DFFC 0000 0014
lea $cf2e,a0                                     ; 00CF14: 41F9 0000 CF2E
move.w #$10,d1                                   ; 00CF1A: 323C 0010
move.b #$0,d0                                    ; 00CF1E: 103C 0000
move.b (a0)+,d2                                  ; 00CF22: 1418
eor.b d2,d0                                      ; 00CF24: B500
dbra d1,-6 {$00CF22}                             ; 00CF26: 51C9 FFFA
rts                                              ; 00CF2A: 4E75
```
