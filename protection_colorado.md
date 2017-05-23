---
title: "Atari ST Protection: Colorado"
date: "2011-09-01"
categories:
   - "software"
   - "atari st"
   - "reverse engineering"
---

Another simple one: it reads track 0 and searches between byte $18 and $97 (128 bytes) for $A1,$FE,$??,$??,$01, which is the sector header of sector 1. I don't really see the value of this protection?