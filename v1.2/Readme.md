This is the initial prototype release.

Prototype boards WERE produced and tested with this release, and did work. 

Known issues:
- C11 obstructs the USB port on the NanoPi
- JP1 should have been a solder jumper, not a physical header
- C14/C20 silkscreens are swapped
- TOT doesn't function. R11 should connect from 3V3 to C9, and U1-6/7 
  should connect to junction of R11/C9 (field change required)
- AN0 max input with R5=10k/R6=3.74k is 13.2VDC. Change R6 to 3.00k 
  for 15.6VDC max.
- Radio MAY transmit for up to TOT value when device is powered on, due 
  to un-initialized GPIO pins 
