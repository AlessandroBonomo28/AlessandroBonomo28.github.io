---
lang: en
hidden: true
lang_ref: emiglio-5
permalink: /en/posts/Trasformare-Emiglio-in-un-AI-assistant-5/
title: "Emiglio modchip, the custom board (Part 5)"
description: I designed a custom PCB for Emiglio with ESP32 and our new sponsor PCBway!
date: 2026-07-30 10:00:00 +0200
categories: [tutorials, electronics]
tags: [tutorial, blog, electronics, IoT, embedded, pcb, esp32]
image:
  path: /assets/img/posts/emiglio-modchip/pcbway-display.jpg
  alt: Emiglio modchip rev 1.0, the custom board for Emiglio
toc: true
---

# A custom PCB designed from scratch for Emiglio: ESP32, motors, audio, display and IR on a single board

In the first four parts Emiglio learned to move with the [RC remote control](/en/posts/Trasformare-Emiglio-in-un-AI-assistant/), to [reason locally](/en/posts/Trasformare-Emiglio-in-un-AI-assistant-2/), to [play via bluetooth](/en/posts/Trasformare-Emiglio-in-un-AI-assistant-3/) and to [walk on motors](/en/posts/Trasformare-Emiglio-in-un-AI-assistant-4/).

Up until now we have kept the circuit together with a tangle of jumper wires and electrical tape. Every time I open Emiglio to add something I have to cross my fingers.

With a **custom PCB** it's much more convenient because everything is integrated (embedded) on the board.

The board is called **Emiglio modchip rev 1.0**, designed in [EasyEDA](https://oshwlab.com/alessandro2001/project_qgcpzeag). The integrated components are: ESP32, USB-C for programming, motor drivers, audio amplifier, display, infrared receiver and several free GPIOs to connect LED eyes or input buttons (or imput quoting [number 5](https://en.wikipedia.org/wiki/Short_Circuit_(1986_film))).

[![Desktop View](/assets/img/posts/emiglio-modchip/board-3d.png)](https://oshwlab.com/alessandro2001/project_qgcpzeag)
_The 3D render_

The project is public and freely accessible:

**[Emiglio modchip on EasyEDA](https://oshwlab.com/alessandro2001/project_qgcpzeag)**

In this article I also recount how I noticed its limits (one with a spark and a bang) and what can be improved. Even the things I got wrong, because those are the ones that teach the most.

## New sponsor: PCBWay

Before getting into the details, some good news:

[![Desktop View](/assets/img/posts/emiglio-modchip/pcbway-logo.jpg)](https://www.pcbway.com/)
_We have the first sponsor of the blog! PCBWay!_

From today the boards for Emiglio and the next blog projects are produced by [PCBWay](https://www.pcbway.com/): quality PCB manufacturing and assembly, fast times and fair prices even for the small hobbyist prototype like this one.

**The project is shared on PCBWay, so you can order Emiglio's board directly, assembled, without having to upload anything:**

### 👉 [Order the Emiglio modchip here](https://www.pcbway.com/project/shareproject/emiglio_modchip_assembly_b18694d1.html)

If you need a PCB of your own instead, **[you can order here](https://www.pcbway.com/)**.

Saying thank you with a simple link didn't seem enough, so I did what an electronics guy does when he's happy: **I put the logo inside the firmware**. There is a dedicated sketch that draws it on the board's display, with the wordmark falling from above and bouncing and the orange swoosh being "routed" column by column like a trace on a PCB.

[![Desktop View](/assets/img/posts/emiglio-modchip/pcbway-display.jpg)](/assets/img/posts/emiglio-modchip/pcbway-display.jpg)
_The PCBWay logo on the 1.8-inch display, with the modchip board connected via USB-C and the back panel with Emiglio's silkscreen_

You can find it among [the sketches in the repo](#the-sketches).

## What's on the board

[![Desktop View](/assets/img/posts/emiglio-modchip/schematic.png)](https://oshwlab.com/alessandro2001/project_qgcpzeag)
_The complete schematic of rev 1.0, drawn in EasyEDA. Click to open the project on OSHWLab_

Physically it's a **2-layer board, 38.2 × 56.9 mm**, 1.6 mm FR-4, black solder mask and white silkscreen. On the back there's Emiglio's face drawn in silkscreen, the *goodman industries* logo and a greeting to Johnny 5, because a PCB without an easter egg is just a piece of fiberglass.

The schematic is divided into six blocks, one for each thing Emiglio needs to be able to do.

### The brain and programming

The heart is an **ESP32-WROOM-32E with 16 MB**. There are also physical BOOT and RESET buttons, two user buttons and two LEDs (power and debug).

### The motors

For the motors there's a **TB6612FNG**, a dual H-bridge driver: two channels, four direction pins, two for PWM and one for standby. Compared to the old L298N it is more efficient and heats up much less, because it uses MOSFETs instead of bipolar transistors.

The **STBY** pin has a 10 kΩ pull-down to ground, and this is a detail I put in on purpose: it means that during boot and after a reset the driver is **disabled**, so Emiglio doesn't make sudden movements while the ESP32 decides who it is. The firmware enables it only after having set the direction pins to a known level and the PWM to zero.

The motors go to a 4-pin connector and the battery to a 2-pin XH, separated from the logic.

### The audio

For the bluetooth speaker from Part 3 there's a **MAX98357A**, an audio amplifier with **I2S** input: no external DAC, no jack, the ESP32 sends it the digital samples and it drives the speaker directly. Three signal wires: bit clock, word select and data.

One thing to know before ordering it: the MAX98357A is **mono**. It doesn't reproduce stereo, it either sums the two channels or throws one away. The resistor on its configuration pin decides which of the two things it does, and on my board it's set to get the sum **(L+R)/2**, so I don't lose half of the instruments in the mix.

### The interface: display, IR and eyes

The display is a **1.8-inch TFT, 128x160, ST7735S controller**, on an 8-pin KF2510 connector. It has a dedicated switch for power, so I can turn it off when I don't need it without touching the firmware.

The infrared receiver is a **TSOP4838**: it's what receives the remote control, the most comfortable control method for doing tests at home. And then there are Emiglio's **LED eyes**, which are the least useful and most important thing on the whole board.

### WATCH OUT FOR THE BATTERY PACK with 12 V it went bang

I wanted more torque on the motors, so I connected a **12 V, 1A** pack to the battery connector. A spark, a sharp bang, an unmistakable smell: **C4** (ironic that the name is exactly c4), the 47 µF capacitor on the motor rail, probably shorted.

Looking at the numbers calmly, the real margins of this board are these:

| Component | Limit |
|---|---|
| Emiglio's motors | **6 V nominal** |
| TB6612FNG | 15 V absolute maximum, but **1.2 A continuous per channel** |
| C4 (47 µF, voltage not declared) | probably 6.3–10 V |

So the driver could handle the 12 V, but the motors at 12 V absorb about double the expected current and break the TB6612's limit.

**The conclusion is that the board is fine as it is, but it must be properly powered:** a **9 V** pack, and as a precaution the sketch with the **duty cycle limited to 50%** should be used.

With that limit the average voltage to the motors remains around **4.5 V**, well within the 6 V nominal: the inductance of the windings smooths the current, so the motor works as if it were powered at 4.5 V DC. The fundamental point is that this limit **exists only in the firmware**: there is no hardware limiter, so at 9 V a bug in the code becomes a physical failure. You can go up to 70% duty cycle, I don't know I haven't tested it and I'm not an electronics engineer

### The reset problem that I have to press by hand

**When I unplug the USB and reconnect it the sketch doesn't start on its own: I have to press the RESET button.** I connect the power, the LEDs light up, and the firmware just sits there until I tell it to start. In the lab with the board on the table you do it without thinking. On a robot that's supposed to turn on and walk, no.

I have two suspects and I haven't decided yet which is the culprit:

- **EN goes up before the 3.3 V is stable.** The capacitor on the EN pin (1 µF with a 10 kΩ pull-up) has a time constant of about 10 ms: if the power rises slowly, the ESP32 sees EN high when it doesn't have a valid voltage yet, and it doesn't boot well. It's the textbook case of "needs a reset on power-up".
- **The auto-reset circuit holds EN or IO0 low.** The RTS and DTR lines of the CH340C drive EN and IO0 through Q2. If either transistor conducts when it shouldn't, the ESP32 stays in reset or enters download mode: on, alive, but without running the sketch.

for now it stays like this and I'll make do

### The bluetooth that didn't reach far

I forgot to remove the copper under the esp antenna (there can be interference)

[![Desktop View](/assets/img/posts/emiglio-modchip/pcb-layout.png)](/assets/img/posts/emiglio-modchip/pcb-layout.png)
_under the antenna there are 2 layers of copper, they should be removed_

## The sketches

All the firmware is in the project repo:

**[github.com/AlessandroBonomo28/emiglio-modchip](https://github.com/AlessandroBonomo28/emiglio-modchip)**

Each folder is an autonomous Arduino sketch, designed to test one subsystem at a time: you flash one at a time, so when something goes wrong you know exactly where to look.

- **[`test_motori_minimo`](https://github.com/AlessandroBonomo28/emiglio-modchip/tree/main/test_motori_minimo)** the main firmware. It drives the TB6612FNG with the IR remote (address `0x07`, and repressing the button of the ongoing maneuver causes a HALT) or from serial with `w s d a`, `x` for stop, `+`/`-` for duty, `i` for status. PWM at 20 kHz to stay out of the audible band, starting ramp at 8 steps, `STBY` held low until the end of initialization and decoding of the reset reason with dedicated diagnostics for brownout which is the function that made me understand the most things of all. **It's the sketch to use by default**, for the reason explained [above](#watch-out-for-the-battery-pack-with-12-v-it-went-bang): all PWM writes go through a single function `pwmWrite()` that applies the `DUTY_MAX` clamp, so the limit cannot be broken by distraction.

- **[`LcdModchip`](https://github.com/AlessandroBonomo28/emiglio-modchip/tree/main/LcdModchip)** display bring-up and calibration. It clears the entire **GRAM 132×162** (including non-visible edges) immediately after `initR()`, then applies `COLSTART=2 / ROWSTART=1` with a subclass that exposes the protected method `setColRowStart()`. Without those offsets the last row and the last column of the panel show random pixels. With `LCD_BORDER_TEST 1` it draws the calibration frame with the four colored corner pixels.

- **[`emiglio_speaker_max98357a`](https://github.com/AlessandroBonomo28/emiglio-modchip/tree/main/emiglio_speaker_max98357a)** Emiglio as a Bluetooth speaker. A2DP sink with the name `Emiglio_Speaker`, automatic reconnection and manageable volume via AVRCP from the phone. The I2S is manually initialized with the board's pins instead of letting the library use the defaults, which change between versions. WiFi off, audio task pinned to core 1 and **CPU obligatorily at 240 MHz**: at 80 or 160 MHz the decoding can't keep up with the audio stream and the watchdog restarts the chip, with a symptom that looks like something else entirely.

- **[`telecom1`](https://github.com/AlessandroBonomo28/emiglio-modchip/tree/main/telecom1)** the remote control that plays sounds. Four notes (DO/RE/MI/FA) on the directional keys, with the samples generated at runtime with `sin()` on I2S at 44.1 kHz and a 10 ms fade in and out to avoid popping. After each note it fills the DMA buffers with silence and stops the clock, so the silence is truly silence.

- **[`pcbway`](https://github.com/AlessandroBonomo28/emiglio-modchip/tree/main/pcbway)** the screen for the sponsor: the animated PCBWay logo on the display, wordmark falling and bouncing, swoosh drawn column by column and scrolling banner at the bottom with the link.

In the [README](https://github.com/AlessandroBonomo28/emiglio-modchip#dipendenze) there is also the complete table of libraries with verified versions and the Arduino IDE settings to use, including the *Huge APP* partitioning scheme that the A2DP sketch needs to fit the Bluetooth stack.

**STAY TUNED** for the next tutorials!
