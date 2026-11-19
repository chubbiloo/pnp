---
title: "Pick and Place Machine"
github: "https://github.com/chubbiloo/pnp"
description: "5-axis pick and place machine, capable of picking and placing surface mounted parts on a PCB.\n"
created_at: "2026-10-06"
total_time: "37h"
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

# November 1, 2026: Board architecture changes
<!-- fabricate:entry 1847 -->

I realized that my board was becoming very complex. If I made a mistake when designing the board, I would have to respin the whole board. If I fried the board, everything would have to be replaced. So, I decided to split my design into three PCBs: a power distribution board, a pneumatics board and a board for the Pico and stepper motors. I added connectors on the Pico board for the unused I/O for a possible 4th board for the feeders.

I made separate nets for the grounds and power rails of each board. To protect against ESD, I added the PS5915-S26, a TVS diode array which will be placed in series with each I/O line close to the connectors and clamp the voltage through them. Since the Pico's 3V3 pin is for output only, I replaced the 3.3V regulator with a 5V one so that the Pico is powered thru VSYS and its internal regulator supplies 3.3V to the rest of the circuit. This also lets me add a schottky diode to protect the Pico from reverse polarity.

![5V regulator](https://fabricate.hackclub-assets.com/58b7d514fb4f53e36c3562ea2ecb1d2609a20de9e5c7c163e401755ab338d67a/image-png/image.png)
![24V distribution with fuses for each board](https://fabricate.hackclub-assets.com/5fb0e4a7fd02245086508b696e496784ab42d6086e3a2222633e107430a931a7/image-png/Screenshot%202026-11-02%20100957.png)
![Updated Pico wiring with ESD protection](https://fabricate.hackclub-assets.com/681868591ef0e755ac9b29e7bb6a50cde65aea55686990aacb32047648adc071/image-png/Screenshot%202026-11-02%20101019.png)

**Total time spent: 3h**

# November 2, 2026: Schematic part 4
<!-- fabricate:entry 3036 -->

To control each vacuum pump, two circuits are needed, one to switch on/off power to the pump itself and another to the solenoid. I realized that since both are just loads, I could use the same design for both circuits. I took some inspiration from the [Starfish](https://blog.thea.codes/starfish-a-control-board-with-the-rp2040/#qwiic-sch:D403) PNP board in my design. I am using a N-channel MOSFET (2N7002) and a P-channel MOSFET (AO7401A), as well as a flyback diode across the terminals of the load. When the N-FET's gate is pulled low, the P-FET's gate voltage is the same as its source voltage, so no current can flow through it. When the N-FET is pulled high, a second path to ground is opened. The two 10k resistors divide the P-FET's input voltage to 12V, allowing current to flow through from the P-FET's source to drain.

Each pump needs a pressure sensor to detect whether a part has been picked up. I chose the 2SMPP-02, which outputs an analog signals. Since the resolution of the Pico's ADC is insufficient, an external ADC will be added to the pneumatics board. Lastly, the input current to the pressure sensor must be limited, so I used the LM334 current limiting IC along with a resistor to do this.

![](https://fabricate.hackclub-assets.com/e3e979ca53d4880590d14602f3ecc676f00f7ca92f53d3a821a91ca605fc12f4/image-png/Screenshot%202026-11-15%20020413.png)
![](https://fabricate.hackclub-assets.com/54d716176127be69b9f2b78c8abf2326da3688bef47609d7133f5d547068c644/image-png/Screenshot%202026-11-19%20123821.png)
![](https://fabricate.hackclub-assets.com/dde145355555c1e2e830e3507adeb41cb149a1b3a7d8cb3cabd7e17c57ee092c/image-png/Screenshot%202026-11-19%20123835.png)

**Total time spent: 3h 30m**

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

# November 18, 2026: PCB routing part 5
<!-- fabricate:entry 3471 -->

Routed the rest of the pneumatics board, including the pressure sensors, the ADC and the ESD protection ICs. Added connectors and traces to distribute power to the gate driver circuits, which were sized accordingly.

![](https://fabricate.hackclub-assets.com/17cd4456091cbf863d92b716f5945a67113f19cd52a6bb579742740844e9e39e/image-png/image.png)

**Total time spent: 2h 30m**

# November 18, 2026 (entry 2): PCB routing part 6
<!-- fabricate:entry 3476 -->

Routed the power board. Due to the high fuse current of 7.5A, I used pours instead of traces for the main power rail. I realized that my shunt resistor and ferrite beads were too small to carry enough current, so I selected larger parts, a 2512 for the resistor and 1612 for the bead. I found that I had missed a decimal place when calculating the shunt's resistance, it should be 0.006 not 0.062 ohms.

![](https://fabricate.hackclub-assets.com/63d3d17b50d91a15904251c816eff6d08f414556b9b02659bf8d029c14e1f506/image-png/Screenshot%202026-11-15%20013216.png)

**Total time spent: 2h**

# November 19, 2026: Widened traces and ran DRC
<!-- fabricate:entry 3474 -->

When routing the pneumatics and power boards, I used an online calculator to find how wide my traces needed to be, based on the trip current for the board's fuse. I realized that the +24V trace on the Pico board was too narrow, so I widened it by shifting the stepper motor drivers downwards to make more space for the trace, and tapering the traces at the first few pads as it was too wide.

After that, I ran DRC and corrected the errors. Most errors were interference errors caused by footprints being too close to each other or traces. There is a clearance error regarding the LTC4368CMS which can't be fixed as its pads are closer together than the KiCad default clearance (0.15mm vs 0.2mm). JLCPCB's minimum clearance is 0.15mm, so this should still be manufacturable.

![](https://fabricate.hackclub-assets.com/28899e5c3e5131ee046c1c4f889ee1ed6e2b1f1704f6095633cc99069f8cdfad/image-png/image.png)

**Total time spent: 1h 30m**

# November 19, 2026 (entry 2): Started frame CAD
<!-- fabricate:entry 3478 -->

Decided to use 2040 extrusion for the frame. Imported a profile for the extrusion, and cadded frame. Using a variable studio to keep the frame parametric. Also imported some other parts (stepper, brackets, GT2 pulley and parametric linear rails).

![](https://fabricate.hackclub-assets.com/fc6bdbc7f49606e20756882f8df21f32e8709cd773b16fdef686896974f879f6/image-png/image.png)
![](https://fabricate.hackclub-assets.com/2cb9d6a71b72e3906017e9781c5a691b46adf162f0f56cb7a72ae31805ad6b6d/image-png/image.png)
![](https://fabricate.hackclub-assets.com/8c80da68729e983c2b3980fbe8802fbda67208e144917d97676897e32ac889a9/image-png/image.png)

**Total time spent: 1h 30m**



