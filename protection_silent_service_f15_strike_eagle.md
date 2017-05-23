---
title: "Atari ST Protection: Silent Service, F15 Strike Eagle"
date: "2011-09-01"
categories:
   - "software"
   - "atari st"
   - "reverse engineering"
---

This is a more interesting protection! For one it decrypts the track testing code only while it is executed (not really a problem with an Atari ST emulator) and encrypts it afterwards.

It reads track 79 and takes the very __FIRST__ byte from the track and searches for the first position where this byte changes. Usually this is the first GAP before the first sector header and without a SYNC it can be shifted by even 1/2 bit by the FDC. However the code has a lookup table for all 16 possible bit-shift combinations and tests if this first byte plus the following 5 are in this table. This way even without a correct SYNC the protection test still succeeds!

But there is more! It skips sector 1..9 with their headers and data blocks and then stores the sector number of the first sector after these 9 sector (e.g. 0x13). This sector extends over the index mark and has to have random data at the end, because read track stops at the index mark. However up to 496 bytes at the beginning have to be identical. If there are no garbage data at the end of the sector, the protection fails, because the sector didn't extend over the index!

As a last test the sector is read normally via XBIOS Floprd() and this call should return NO error (read sector doesn't stop at the index mark, if it already started reading before the index mark). The whole 512 byte sector should have identical byte in it.

If all this succeeds, the copy protection test was successfull, too.

A very nice protection indeed!

``` 68k
silentServiceProtection:
	move.l  a6,$148d6
	lea     $1491a,a6
	movem.l d1-7/a0-5,-(a6)
	move.l  a6,$148da

	move.w  #$22,-(a7)
	trap    #14                 ;Kbdvbase()
	addq.w  #2,a7
	movea.l d0,a0
	move.l  16(a0),$148ae       ;->kb_mousevec

	move.l  $148ae,-(a7)
	lea     $13cd4,a0
	move.l  a0,-(a7)
	clr.w   -(a7)
	clr.w   -(a7)
	trap    #14                 ;Initmouse(0,...)
	adda.l  #12,a7

	move    sr,d0
	btst    #13,d0
	bne.s   L0102F4
	clr.l   -(a7)
	move.w  #$20,-(a7)
	trap    #1
	addq.w  #6,a7
	move.l  d0,$148ce
	move.l  #-1,$148c2
	bra.s   L0102FA
L0102F4
	clr.l   $148c2

L0102FA
	move.b  #$ff,$43e           ;set flock

	move    sr,-(a7)
	ori.w   #$700,sr
	clr.l   d0
	clr.l   d7
	move    (a7)+,sr

	move.w  #$19,-(a7)
	trap    #1                  ;Dgetdrv()
	addq.w  #2,a7
	cmpi.b  #1,d0
	bne.s   L01032A
	move.w  $4a6,d5
	cmpi.w  #2,d5
	beq.s   L01032A
	clr.l   d0
L01032A
	move.w  d0,$148ac

	addq.b  #1,d0               ;drive = 1/2
	lsl.b   #1,d0               ;drive = 2/4
	or.w    #0,d0               ;side 1
	eori.b  #7,d0
	and.b   #7,d0
	moveq   #2,d4               ;3 retries
	bsr     fdcSelect

	move.w  #$82,$ff8606        ;Track Register
	bsr     fdcReadD0
	move.w  d0,$148b4           ;save track register

cpRetry
	bsr     fdcRestore
	btst    #0,d6
	bne.l   cpFailed

	bsr     fdcSeek79
	btst    #0,d6
	bne.l   cpFailed

	clr.l   d3
	moveq   #2,d5
cpNextTrack:
	bsr     fdcReadTrack
	btst    #0,d6
	bne.l   cpFailed
	bsr     decryptCode

	lea     $1491e,a0       ;track buffer
	move.l  #$1f0,d0
	move.b  (a0)+,d7
cpSearch
	move.b  (a0)+,d6        ;find first changed byte
	cmp.b   d6,d7
	bne.s   cpNext
	dbra    d0,cpSearch
	bra.l   cpFailed
cpNext
	subq.l  #2,a0           ;back to the last byte from the block
	lea     $13c72,a2

	;a possible bit shift mask
		$01,$03,$3F,$0F,$FF,$FF,
		$02,$06,$7C,$1F,$FF,$FF,
		$04,$0C,$F8,$3F,$FF,$FF,
		$08,$19,$F0,$7F,$FF,$FF,
		$10,$33,$E0,$FF,$FF,$FF,
		$20,$67,$C1,$FF,$FF,$FF,
		$40,$CF,$83,$FF,$FF,$FF,
		$80,$81,$9F,$07,$FF,$FF,

		$5E,$5C,$40,$B0,$00,$00,
		$BC,$B8,$81,$60,$00,$00,
		$97,$10,$2C,$00,$00,$00,
		$2F,$2E,$20,$58,$00,$00,
		$E5,$C4,$0B,$00,$00,$00,
		$CB,$88,$16,$00,$00,$00,
		$79,$71,$02,$C0,$00,$00,
		$F2,$E2,$05,$80,$00,$00,

		$55,$55

	move.b  (a0)+,d6        ;byte at the track beginning that was repeated
cpLoop
	move.b  (a2)+,d7        ;find the repeated byte
	cmp.b   #$55,d7
	beq.l   cpFailed
	cmp.b   d6,d7
	beq.s   cpNextB
	addq.w  #5,a2
	bra.s   cpLoop
cpNextB

	moveq   #4,d0
cpLoopB
	move.b  (a0)+,d6        ;the following 5 bytes have to be the same
	cmp.b   (a2)+,d6
	bne.l   cpFailed
	dbra    d0,cpLoopB

	; skip 9 sectors 1..9 in the current track
	clr.b   d0
cpLoopC
	bsr     cpSearchSync    ;search for $A1,$A1
	addq.b  #1,d0           ;sector + 1
	move.b  (a0)+,d6
	cmp.b   #$fe,d6         ;Address Mark Header
	bne.l   cpFailed
	move.b  (a0)+,d6
	cmp.b   #$4f,d6         ;Track 79
	bne.l   cpFailed
	move.b  (a0)+,d6
	cmp.b   #0,d6           ;Side == 0
	bne.l   cpFailed
	move.b  (a0)+,d6
	cmp.b   d0,d6           ;Sector correct?
	bne.l   cpFailed

	bsr     cpSearchSync    ;search for $A1,$A1
	move.b  (a0)+,d6
	cmp.b   #$fb,d6         ;Data Header
	bne.l   cpFailed
	adda.l  #$200,a0        ;Skip Sector
	cmp.b   #9,d0
	bne.s   cpLoopC

	bsr     cpSearchSync    ;search for $A1,$A1
	move.b  (a0)+,d6
	cmp.b   #$fe,d6         ;Addres Mark Header
	bne.s   cpFailed
	move.b  (a0)+,d6
	movea.l #10,a6
	cmp.b   #$4f,d6         ;Track 79
	bne.s   cpFailed
	move.b  (a0)+,d6
	cmp.b   #0,d6           ;Side == 0
	bne.s   cpFailed
	clr.w   d6
	move.b  (a0)+,d6        ;Sector >= 10
	cmp.b   #10,d6
	blt.s   cpFailed
	move.w  d6,$13c6e       ;save found sector number ($0D)

	bsr     cpSearchSync    ;search for $A1,$A1
	move.b  (a0)+,d6
	cmp.b   #$fb,d6         ;Data Header
	bne.s   cpFailed

	clr.w   d0
	clr.w   d6
	move.b  (a0)+,d6
	move.w  d6,$13c70       ;first byte in the sector ($E5)
cpLoopD
	addq.w  #1,d0
	move.b  (a0)+,d6
	cmp.w   $13c70,d6       ;skip same bytes (first byte in the sector ($E5))
	bne.s   cpNextC
	cmp.w   #$1f0,d0        ;too many bytes?
	beq.s   cpFailed
	bra.s   cpLoopD
cpNextC

	cmp.b   #0,d6           ;end of track reached before end of sector?
	beq.s   cpOut

cpFailed
	clr.w   $13c6e          ;save found sector number = illegal

	bsr     encryptCode
	dbra    d5,cpNextTrack
	dbra    d4,cpRetry

	bsr     fdcDone
	move.b  d0,d6
	cmpi.l  #$0,$148c2
	beq.s   L0104A8
	move.l  $148ce,-(a7)
	move.w  #$20,-(a7)
	trap    #1
	addq.w  #6,a7
L0104A8
	bsr     cpCreateReturncode
	bsr     cpRestoreRegister
	rts

cpOut
	bsr     encryptCode
	bsr     fdcDone

	cmpi.l  #$0,$148c2
	beq.s   L0104D4
	move.l  $148ce,-(a7)
	move.w  #$20,-(a7)
	trap    #1
	addq.w  #6,a7

L0104D4
	move.w  #1,-(a7)            ;count
	move.w  #0,-(a7)            ;sideno
	move.w  #$4f,-(a7)          ;trackno
	move.w  $13c6e,-(a7)        ;sectno = saved found sector number ($0D)
	move.w  $148ac,-(a7)        ;devno
	clr.l   -(a7)               ;filler
	lea     $1491e,a0           ;track buffer
	move.l  a0,-(a7)
	move.w  #8,-(a7)
	trap    #14                 ;int16_t Floprd( void *buf, int32_t filler, int16_t devno, int16_t sectno, int16_t trackno, int16_t sideno, int16_t count )
	adda.l  #20,a7
	cmp.l   #0,d0
	beq.s   L010512             ;should be 0! =>
	andi.l  #$ffffff,d0
L010510
	rts

L010512
	ori.l   #$37000000,d0

	;the sector has to have all the same byte
	move.l  #511,d7
	lea     $1491e,a0           ;track buffer
	clr.w   d6
L010526
	move.b  (a0)+,d6
	cmp.w   $13c70,d6           ;first byte in the sector ($E5)
	bne.s   L010510
	dbra    d7,L010526

	ori.l   #$4f,d0
	bsr     cpCreateReturncode
	bsr     cpRestoreRegister
	rts

fdcReadTrack:
	move.l  d7,$148ca
	lea     $1491e,a0           ;track buffer
	move.l  a0,$148d2
	move.b  $148d5,$ff860d
	move.b  $148d4,$ff860b
	move.b  $148d3,$ff8609
	move.l  #$680,d7
fdcReadTrackLoopA
	clr.l   (a0)+
	dbra    d7,fdcReadTrackLoopA
	move.w  #$90,$ff8606
	move.w  #$190,$ff8606
	move.w  #$90,$ff8606
	move.w  #31,d7
	bsr     fdcWriteD7          ;31 DMA sectors (31*512 bytes)
	move.w  #$80,$ff8606
	move.w  #$e4,d7
	bsr     fdcWriteD7          ;Read Track
	move.l  #$40000,d7
	move.b  #255,d6             ;Failed flag
fdcReadTrackLoopB
	btst    #5,$fffa01
	beq.s   fdcReadTrackDone
	subq.l  #1,d7
	bne.s   fdcReadTrackLoopB
	bsr     fdcForceInterrupt
	rts
fdcReadTrackDone
	move.w  #$90,$ff8606
	move.w  $ff8606,d0
	btst    #0,d0               ;DMA ok?
	beq.s   fdcReadTrackFailed
	clr.b   d6                  ;success
fdcReadTrackFailed
	rts

cpRestoreRegister:
	movea.l $148da,a6
	movem.l (a6)+,d1-7/a0-5
	movea.l $148d6,a6
	rts

cpSearchSync:
	move.l  #$1f0,d7
cpSearchSyncLoop
	move.b  (a0)+,d6
	cmp.b   #$a1,d6
	bne.s   cpSearchSyncCheck
	move.b  (a0)+,d6
	cmp.b   #$a1,d6
	bne.s   cpSearchSyncCheck
	rts
cpSearchSyncCheck
	dbra    d7,cpSearchSyncLoop
	suba.l  a6,a6
	addq.w  #4,a7
	bra.l   cpFailed

cpCreateReturncode:             ;$3700004F
	andi.l  #$ff0000ff,d0
	clr.l   d7
	move.w  $13c70,d7           ;first byte in the sector ($E5)
	rol.w   #8,d7
	or.w    $13c6e,d7           ;save found sector number ($0D)
	rol.w   #4,d7
	rol.l   #8,d7
	or.l    d7,d0               ;== $3750DE4F (Silent Service) / $37512E4F (F15 Strike Eagle)
	rts

Calculation after return:
0 == ($0062EB61 ^ $3750DE4F) - $3732352E


fdcDone
	movea.l $148c6,a5
	move.w  $148b4,d7           ;save track register
	bsr     fdcSeekD7
fdcDoneLoop
	move.w  #$80,$ff8606
	bsr     fdcReadD0           ;wait for the motor off
	btst    #7,d0
	bne.s   fdcDoneLoop
	move.b  d2,d0
	move.b  #$0,$43e            ;reset flock
	bsr     fdcSelect           ;deselect floppy

	move.l  $148ae,-(a7)
	lea     $13cd4,a0
	move.l  a0,-(a7)
	move.w  #1,-(a7)
	clr.w   -(a7)
	trap    #14                 ;Initmouse(1,...)
	adda.l  #12,a7
	rts

CryptRetA0Plus_16
	addq.l  #1,a0
	bsr CryptRetA0Plus_10
CryptRetA0Plus_5
	subq.l  #1,a0
	bsr CryptRetA0Plus_5B
CryptRetA0Plus_1
	subq.l  #1,a0
	bsr CryptRetA0Plus_2
CryptRetA0Plus_0
	rts


fdcWriteD7
	bsr.s   fdcDelay
	move.w  d7,$ff8604

fdcDelay
	move    sr,-(a7)
	move.w  d7,-(a7)
	move.w  #32,d7
fdcDelayLoop
	dbra    d7,fdcDelayLoop
	move.w  (a7)+,d7
	move    (a7)+,sr
	rts

encryptCode:
	bsr.s   CryptBlock
	move    $148b2,sr
	rts

decryptCode:
	move    sr,$148b2
	ori.w   #$700,sr
	bsr.s   CryptBlock
	rts

CryptBlock
	move    sr,d7
	lsr.w   #8,d7
	bset    #6,d7
	bclr    #4,d7
	bclr    #3,d7

	lea     $13ce1,a0
	lea     $1038a,a2
	bsr.s   CryptRetA0Plus_16
	addq.l  #1,a0               ;A0 = $13CF2
	moveq   #8,d0
	bsr.s   CryptBlockSub

	lea     $13ce1,a0
	lea     $103a8,a2
	bsr     CryptRetA0Plus_10   ;A0 = $13CF5
	moveq   #34,d0
	bsr.s   CryptBlockSub

	lea     $13ce1,a0
	lea     $103fe,a2
	bsr     CryptRetA0Plus_5B   ;A0 = $13CE6
	moveq   #26,d0
	bsr.s   CryptBlockSub

	addq.l  #5,a0               ;A0 = $13D06
	lea     $10444,a2
	moveq   #4,d0
	bsr.s   CryptBlockSub

	lea     $13ce1,a6
	lea     $10466,a2
	bsr     CryptRetA0Plus_1
	bsr     CryptRetA0Plus_2    ;A0 = $13D0E
	moveq   #4,d0
	bsr.s   CryptBlockSub
	rts

CryptBlockSub
	clr.l   d6
	move.b  (a0)+,d6
	adda.l  d6,a2
	eor.b   d7,(a2)
	dbra    d0,CryptBlockSub
	rts


fdcSelect:
	move    sr,-(a7)
	ori.w   #$700,sr
	move.b  #$e,$ff8800
	move.b  $ff8800,d1
	move.b  d1,d2
	and.b   #$f8,d1
	or.b    d0,d1
	move.b  d1,$ff8802
	move    (a7)+,sr
	rts


fdcReadD0
	bsr     fdcDelay
	move.w  $ff8604,d0
	bra.l   fdcDelay


fdcRestore
	move.w  #$03,d7
	bsr.s   fdcCommandAndWait
fdcRestoreLoop
	subq.l  #1,d6
	beq.s   fdcRestoreTimeout
	btst    #5,$fffa01
	bne.s   fdcRestoreLoop
	clr.l   d6
	move.w  #$80,$ff8606
	bsr.s   fdcReadD0
	btst    #2,d0           ;Track 0?
	bne.s   fdcRestoreSuccess
fdcRestoreTimeout
	bsr.s   fdcForceInterrupt
	moveq   #1,d6
fdcRestoreSuccess
	rts


fdcForceInterrupt
	move.w  #$80,$ff8606
	move.w  #$d0,d7
	bsr     fdcWriteD7
	bsr     fdcDelay
	rts


fdcSeek79
	move.w  #$4f,d7
fdcSeekD7
	move.w  #$86,$ff8606
	bsr     fdcWriteD7
	move.w  #$13,d7
	bsr.s   fdcCommandAndWait
fdcSeek79Loop
	subq.l  #1,d6
	beq.s   fdcSeek79Done
	btst    #5,$fffa01
	bne.s   fdcSeek79Loop
fdcSeek79Done
	clr.l   d6
	rts


fdcCommandAndWait
	move.l  #$40000,d6
	move.w  #$80,$ff8606
	bsr     fdcReadD0
	btst    #7,d0
	bne.s   fdcCommandAndWait2
	move.l  #$60000,d6
fdcCommandAndWait2
	bsr     fdcWriteD7
	rts


CryptRetA0Plus_10
	bsr     CryptRetA0Plus_5
CryptRetA0Plus_5B
	addq.l  #2,a0
	bsr     CryptRetA0Plus_1
CryptRetA0Plus_2
	addq.l  #2,a0
	bsr     CryptRetA0Plus_0
	rts
```
