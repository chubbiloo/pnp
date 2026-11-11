---
title: "Pick and Place Machine"
github: "https://github.com/chubbiloo/pnp"
description: "5-axis pick and place machine, capable of picking and placing surface mounted parts on a PCB.\n"
created_at: "2026-10-06"
total_time: "26h"
---

# October 6, 2026: Selected components for mainboard
<!-- fabricate:entry 168 -->

Although I have done some brainstorming already, the first work I log will be on the custom mainboard. From my research, I learned that I need 5-6 stepper motors for the PnP:
- 2 on the X-axis
- 1 on Y
- 1 on Z to raise and plunge the toolhead(s) 
- 1 per toolhead to rotate the part

The parts are picked up using vacuum pump(s), but we will worry about that later.

I chose to use TMC2209 drivers due to my previous experience with them and a Pi Pico, which will be plugged into headers on the PCB instead of integrated directly. The motors are powered directly from a 24V PSU but it must be stepped down to 5V to power the Pico. For that, I chose a LDO. I read the LDO and TMC2209 datasheets to figure out what decoupling/bulk caps are needed. 

Lastly, I found parts for the reverse polarity and back emf protection circuits (more on that later).

![](https://fabricate.hackclub-assets.com/641b7c6f86d94152d30847304873691f710521ee39370987eaa9ac3939856161/image.png)

**Total time spent: 5h**

# October 13, 2026: Schematic part 1 + TMC2209 mald
<!-- fabricate:entry 368 -->

I started the schematic by reading the TMC2209's datasheets and connecting the STEP/DIR pins to the Pi Pico. I decided that I wanted to control the TMCs from the RP2040 using the UART interface. This has several benefits, most importantly the ability to set the current limit in software instead of turning the potentiometer on the board. 

Four TMCs can be connected to one UART bus by pulling the MS1 and MS2 pins up or down to change the 2-bit UART address. If UART is disabled, MS1 and MS2 control the microstepping rate of the driver. Enabling UART saves 2 I/O pins per TMC as the MS1 and MS2 pins can now be fixed (and the microstepping rate set through UART). This eliminates an I/O expander which I thought I needed before (1st image).

![](https://fabricate.hackclub-assets.com/3453fd3b1c283c0c837fc0831a6c4b50474f0daf6056b9641a315685ffb3d14c/image-png/Screenshot%202026-10-13%20000528.png)
![](https://fabricate.hackclub-assets.com/d6f3501c6a5db9c893317da139beceed0162d6c347b5a5f10fd7f62752d96bc2/image-png/image.png)

**Total time spent: 4h**

# October 13, 2026 (entry 2): Schematic part 2 + LDO replacement
<!-- fabricate:entry 370 -->

I was about to route the LDO when I decided to replace it with a buck converter. I need to step 24V down to 3.3V to power the Pi Pico. Since the input current of a linear regulator is roughly equal to the output current, the LDO is approximately (3.3V/24V)*100% = 13.75% efficient. The power which the LDO dissipates as heat (more than 6x what it outputs) could overheat and damage it. A buck converter is >80% efficient.

I decided to buy a buck converter instead of designing my own, which would have overcomplicated this project and caused issues downstream if I didn't design it properly.

I found buck converter boards on Amazon and Aliexpress, but the listings did not specify the spacing between the pins, which I would need to mount the buck onto the main PCB using headers. Instead, I found a 24-3.3V buck converter in a smaller package on DigiKey (MSR7805CW) and routed it.

![](https://fabricate.hackclub-assets.com/2928f44638920b997314a90a4ed1b0b3a205b9713b864354dea0ed7d62b63194/image-png/Screenshot%202026-10-13%20003736.png)
![](https://fabricate.hackclub-assets.com/e71e740b177b9c4d22370e7e7568e8ffc312d2f4b06d486ff1a09636d3367617/image-png/Screenshot%202026-10-13%20003741.png)
![](https://fabricate.hackclub-assets.com/ebfc25fc4ad26662e15000bbb570d079962c43da48c5605ff974ca5c3e72896f/image-png/Screenshot%202026-10-13%20003849.png)

**Total time spent: 2h**

# October 13, 2026 (entry 3): Schematic part 3
<!-- fabricate:entry 371 -->

The PCB has a circuit which protects it from overvoltage, overcurrent and reverse polarity. It uses two N-channel MOSFETs and is controlled by an IC ("UV/OV and Reverse Protection 
Controller with Bidirectional Circuit Breaker"). As I have not taken circuits and do not fully understand how the controller works, I copied the example circuit from the datasheet. I changed the value of the sense resistor to reduce the current limit from 10A to 8.06A.

In addition to the circuit, I also added a 7.5A automotive fuse and TVS diodes to protect the TMC2209 drivers from flyback voltage spikes which could occur if the motor is stalled or if power is cut while the motor is still running.

![](https://fabricate.hackclub-assets.com/318df4715239e82048dc9d87d64700f3f8ef3cdbe84788ad65e0805102cd1fa9/image-png/Screenshot%202026-10-13%20011745.png)
![](https://fabricate.hackclub-assets.com/50c20b466250f44cec943d7b4f027a111352ea035e53d4b93411c84301470b47/image-png/Screenshot%202026-10-13%20011830.png)
![](https://fabricate.hackclub-assets.com/36ea443b81d2092855c4d07336a12a218cb969fe9a7991448f4f470a04ab0977/image-png/Screenshot%202026-10-13%20012211.png)

**Total time spent: 3h**

# November 2, 2026: Board architecture changes
<!-- fabricate:entry 1847 -->

I realized that my board was becoming very complex. If I made a mistake when designing the board, I would have to respin the whole board. If I fried the board, everything would have to be replaced. So, I decided to split my design into three PCBs: a power distribution board, a pneumatics board and a board for the Pico and stepper motors. I added connectors on the Pico board for the unused I/O for a possible 4th board for the feeders.

I made separate nets for the grounds and power rails of each board. To protect against ESD, I added the PS5915-S26, a TVS diode array which will be placed in series with each I/O line close to the connectors and clamp the voltage through them. Since the Pico's 3V3 pin is for output only, I replaced the 3.3V regulator with a 5V one so that the Pico is powered thru VSYS and its internal regulator supplies 3.3V to the rest of the circuit. This also lets me add a schottky diode to protect the Pico from reverse polarity.

![5V regulator](https://fabricate.hackclub-assets.com/58b7d514fb4f53e36c3562ea2ecb1d2609a20de9e5c7c163e401755ab338d67a/image-png/image.png)
![24V distribution with fuses for each board](https://fabricate.hackclub-assets.com/5fb0e4a7fd02245086508b696e496784ab42d6086e3a2222633e107430a931a7/image-png/Screenshot%202026-11-02%20100957.png)
![Updated Pico wiring with ESD protection](https://fabricate.hackclub-assets.com/681868591ef0e755ac9b29e7bb6a50cde65aea55686990aacb32047648adc071/image-png/Screenshot%202026-11-02%20101019.png)

**Total time spent: 3h**

# November 2, 2026 (entry 2): PCB routing part 1
<!-- fabricate:entry 1848 -->

I started by routing the TMC2209s on the Pico board, going for a rectangular board shape with the drivers on the right and bottom sides. I kept the capacitors, TVS diodes and connectors close to the TMCs, and widened the 24V rail to handle up to 5A of current. I changed the mapping of some STEP/DIR/UART pins on the Pico to minimize trace crossing.

![TMC2209 routing](https://fabricate.hackclub-assets.com/8b1e99b59fac3bd85426892349d74fa1f8338472e757fc1ae80cb957ffac48a3/image-png/Screenshot%202026-10-25%20194943.png)

**Total time spent: 2h 30m**

# November 11, 2026: PCB routing part 2
<!-- fabricate:entry 2602 -->

I placed the second 24V input (the one that goes into the 5V regulator) on the left side, above the Pico so that both 24V inputs are close together. The 5V regulator splits into two branches: one powering the Pico and one leading to an external connector. I tried to keep these two paths as straight as possible but they do "backtrack" a bit due to limited space.

Afterwards, I placed the remaining connectors (most of which go to pneumatics board) on the top edge of the board, and connected the ESD protection ICs to them. 


![](https://fabricate.hackclub-assets.com/f74d17538f0422e3656de348a32daadf473c27f796bbf894f38e17c1916786ca/image-png/Screenshot%202026-10-28%20114534.png)
![](https://fabricate.hackclub-assets.com/db5e8179b6740d53ef97ea053c511991b4f36d42685e658b09ad68ed8317622b/image-png/Screenshot%202026-10-26%20005122.png)

**Total time spent: 3h**

# November 11, 2026 (entry 2): PCB routing part 3
<!-- fabricate:entry 2603 -->

I added resistors in series with the `SENSE_SCL`, `SENSE_SDA` lines and the lines reserved for Board 4 for additional ESD protection. I adjusted the wiring of the ESD ICs to make sure that the traces pass through the pads of the IC instead of branching off to them. Then, I expanded the size of the board to fit 6 M2 mounting holes, not tied to ground as the electronics will probably be enclosed inside a 3D printed case. I added stitching vias with a spacing of 5 mm using the array tool and added a ground pour to both sides.   The Pico board is pretty much done now!

![](https://fabricate.hackclub-assets.com/8ed859008d844f1fae4eb57880e9f52cfbab17e67bb605670560820a9ae205ea/image-png/image.png)
![](https://fabricate.hackclub-assets.com/14ba731ebffb45d031fa451c663025657336a278afdcd5dcd5c395e84fde0591/image-png/Screenshot%202026-11-11%20010052.png)

**Total time spent: 2h**

# November 11, 2026 (entry 3): PCB routing part 4
<!-- fabricate:entry 2646 -->

Now, it is time to start working on the pneumatics board. This board has 4 of the same gate driver circuit, which is used to switch on/off the 2 vacuum pumps and 2 solenoids. I routed one circuit, making sure to widen the power-carrying trace and replicated it 3 times.

![](https://fabricate.hackclub-assets.com/80c18432bde949ece805bd8c5b440ff1d0bb07df649b4872c77304c8e26ddfd5/image-png/image.png)

**Total time spent: 1h 30m**
