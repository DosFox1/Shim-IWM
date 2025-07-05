
![telegram-cloud-photo-size-4-5947352454880412968-y](https://github.com/user-attachments/assets/80b29d71-7c9d-4a58-96a5-6ca927554258)

# What is the Shim IWM?

A simple electronic "shim" to replace an Integrated Woz Machine (the apple floppy controller) in a Macintosh. It contains just enough hardware to pass the initial boot check, and boot into Mac OS via SCSI. This board is also referred to as the "SCHWIM", as I cannot say no to a bad pun.
This setup has been tested on a Macintosh Plus, Macintosh SE FDHD (which expects a SWIM), as well as a Macintosh LC ii. It is likely this will work in any Macintosh that requires an IWM or SWIM.

If you need to repair a Macintosh that has a bad IWM, but you are okay with losing Floppy support - then this is the board for you!

Many thanks to Matt Evans (https://github.com/evansm7) for providing help with this project - specifically the data to be put on the databus, and initially testing the modification that led to this board in QEMU as well as Pico-mac.
Many thanks to Max1zzz (https://github.com/max234252) for testing the SCHWIM in an SE FDHD, as well as in a Macintosh LC ii. 

# What this is not

This is in no way a full replacement for the IWM. This is only meant to get the system to get past checking for the floppy controller, and then to boot from a different boot medium - such as ROM, SCSI or IDE. 
It will **NOT:**
1) give you full floppy support
2) work in an Apple ii
3) give you HD20 emulation support
4) force you to eat toast
5) gain sentience and try to take over the world (probably)

# How does this even work? 

First of all, I want to say that I HATE THAT THIS SOLUTION WORKS. 
It is so so so dumb and backwards. If you check internal Apple documents of the era, you will literally find references to engineers saying they don't want to touch the IWM or the associated driver. 

https://bitsavers.org/pdf/apple/mac/prototypes/1985_YACC/YACC_Description.pdf
http://bitsavers.informatik.uni-stuttgart.de/pdf/apple/mac/prototypes/1985_Big_Mac/BIGMAC_Bringup.pdf

Safe to say the IWM and the associated driver are a bit of a headache. 
So how does this all work??

Effectively, the circuitry works by using an OR gate to monitor the state of Address 0 (also referred to as A1 in some schematics, it's also an enable pin - just to annoy you - basically it's pin 3), as well as the /DEV pin (the enable pin for the IWM itself - Pin 10). When both are low (active low pins), they enable the 74LS244 octal buffer to put the data 0xDF (1101111 - D7->D0) onto the data bus, which has been pulled high and low via eight 4.7K resistors. 

On boot, the Macintosh checks for the presence of the IWM by reading the Status register. 
The status register contains a single byte of data, with each bit representing the status of a setting in the IWM. These are as follows: 

<pre>
0 - Latch mode (set in async mode, 1 = async mode) - Macintosh is async, so set to 1
1 - synchronous handshake protocol (1 = async mode) - Macintosh is async, so set to 1 again
2 - 1 second onboard timer enable (1 = disabled, 0 = keep motor on for 1 second) - timer disabled, set to 1
3 - slow mode/fast mode (bit cell time - 1 = 2us, 0 = 4us) - bit sel is 2us in a Macintosh 
4 - clock - 7mhz/8mhz (clcok input, 1 - 8mhz, 0 = 7mhz) - Macintosh clock is 8mhz, set to 1
5 - /ENBL1 or /ENBL2 is currently active (0 = active, 1 = inactive) - IWM is enabled, set to 0
6 - MZ (reserved for future compatability - should be read as 0, but 1 apparently works) - set to 1, could be 0?
7 - Sense input (1 = sense is high, 0 = sense is low. Could be either, but 1 works) - set to 1, but could be 0?
</pre>

So from  D7 to D0, the data should read:
<pre>
0 - 1
1 - 1
2 - 1
3 - 1
4 - 1
5 - 0
6 - 1
7 - 1
</pre>


Which in turn, represented in hex, is 0xDF, or 1101111 in binary.

Once the status register is read, the mode register is read straight after. D0 to D4 are the same as the mode register. The mode register contains: 

<pre>
0 - Latch mode (set in async mode, 1 = async mode) - Macintosh is async, so set to 1
1 - synchronous handshake protocol (1 = async mode) - Macintosh is async, so set to 1 again
2 - 1 second onboard timer enable (1 = disabled, 0 = keep motor on for 1 second) - timer disabled, set to 1
3 - slow mode/fast mode (bit cell time - 1 = 2us, 0 = 4us) - bit sel is 2us in a Macintosh 
4 - clock - 7mhz/8mhz (clcok input, 1 - 8mhz, 0 = 7mhz) - Macintosh clock is 8mhz, set to 1
5 - test mode (1 = test mode, 0 = normal operation) - set to normal operation - 0
6 - MZ-reset (reserved for future compatability - should be read as 0, but 1 apparently works) - set to 1, could be 0?
7 - Reserved for Future expansion, so could be either, but 1 works - set to 1, but could be 0?
</pre>

So again, seting the data to 1101111 works for the mode register as well. I might be wrong about that explanation, but hey this implementation works!
After all that, the system does not check the IWM again - or at least when it does, it see that the sense line is still high, so it believes that no disk has been inserted. 

It basically just sits there, reading a newspaper saying "keep going, you're all good. No problems here", and the Macintosh effectively goes "sounds good!" and keeps booting. 

It is so mind numblingy simple, I almost hate it!

# Construction

Assembling the board is relatively simple, and can be assembled by anyone with decent soldering abilities. The PCB routing was done in about fifteen minutes on my lunch break, so it is pretty dire. This board is currently at VDEV1,and I'm happy to report that they do work! Please note, to make the best electrical connection from the IWM socket to the SCHWIM - it's ideal to add a turned pin socket between the SCHWIM board and the logic board.

The BOM is as follows, all parts are generic garden variety components:

<pre>
Part     Value          Package          Library       Position (mm)         Orientation

C1       100n 50V       CAP_5.08MM_PITCH Capacitors TH (9.8 6.9)             R180
C2       100n 50V       CAP_5.08MM_PITCH Capacitors TH (16.9 21.7)           R180
R1       4k7            RESISTORTHRUHOLE Resistors TH  (13.5 20.6)           R180
R2       4k7            RESISTORTHRUHOLE Resistors TH  (13.5 24.1)           R180
R3       4k7            RESISTORTHRUHOLE Resistors TH  (13.5 27.6)           R180
R4       4k7            RESISTORTHRUHOLE Resistors TH  (13.5 31.1)           R180
R5       4k7            RESISTORTHRUHOLE Resistors TH  (13.5 34.6)           R180
R6       4k7            RESISTORTHRUHOLE Resistors TH  (32 34.1)             R180
R7       4k7            RESISTORTHRUHOLE Resistors TH  (32 37.1)             R180
R8       4k7            RESISTORTHRUHOLE Resistors TH  (32 31.1)             R180
CN1      IWM            Turn Pin Header  Headers       (20 9.6)              R0
U1       74LS32N        DIL14            74xx-us       (27.88 24.19)         R0
U2       74LS244N       DIL20            74xx-eu       (26.3 9.6)            R0
</pre>

The board should be able to fit over the ROMs in a Macintosh Plus, it may be necessary to add another DIP28 socket between the board and the IWM socket. 

I never got around to designing an SMD version of the SCHWIM - but the wonderful HKZ did!
See here for the SMD version:
https://github.com/hkzlab/schwim_repacked
