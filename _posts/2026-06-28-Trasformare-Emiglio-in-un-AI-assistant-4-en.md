---
lang: en
hidden: true
lang_ref: emiglio-4
permalink: /en/posts/Trasformare-Emiglio-in-un-AI-assistant-4/
title: "Emiglio robot with tank tracks (Part 4)"
date: 2026-06-24 00:00:00 +0000
categories: [tutorials, electronics]
tags: [tutorial, blog, electronics, IoT, embedded]
image:
  path: /assets/img/posts/emiglio/cingoli.png
  alt: The tank tracks bought for Emiglio
---

# Increasing the robot's stability with tracks and a swivel caster wheel

{% include embed/youtube.html id='g96JaZwjHMA' %}

## Emiglio's old wheels broke

After weeks of honorable service, the wheels of our [Emiglio robot from the first tutorial](/en/posts/Trasformare-Emiglio-in-un-AI-assistant/) stopped working... the plastic carriage couldn't handle the repeated stress of the motors driven by a 12V voltage and at a certain point it gave way. In the gif below you can see an offended Emiglio with its wheels taken off.

![Desktop View](/assets/img/posts/emiglio/ruoteandate.gif){: width="auto"}
_Emiglio K.O. with the broken old wheels_

## Emiglio is better (with tracks)
To permanently solve this I decided to buy some **badass tank tracks** for Emiglio in order to make it stable and ready to face any kind of terrain. Searching online my eye fell on these:

![Desktop View](/assets/img/posts/emiglio/cingoli.png){: width="auto"}
_Tracks on amazon_

### Component links

- [TRACKS on amazon](https://amzn.to/441fVVf)
- [Raspberry pi zero W2](https://amzn.to/3SuylLD)
- [Respeaker Module](https://amzn.to/4oQRt2t)
- [Raspberry pi 4 B (better than Pi 1 B)](https://amzn.to/3R2ipQi)
- [12V rechargeable battery](https://amzn.to/3SOVmZL)
- [TB6612FNG motor driver](https://amzn.to/43Z6G8c)

### The TS100 motors and the new wiring

![Desktop View](/assets/img/posts/emiglio/tsmotor.png){: width="500px"}
_TS100 motors of the tracked chassis_

The new motors of the tracked chassis come with a 6-wire connector because they include an encoder (a sensor to measure the wheel's rotation). The modification is simple because Emiglio's electronic circuit remains the same as the [tutorial part 1](/en/posts/Trasformare-Emiglio-in-un-AI-assistant/). I just need to change the cables going to the motors, we can drive them like completely normal DC motors. Here are the practical steps:

- Disconnect Emiglio's old motors from the motor driver outputs (Motor A and Motor B terminals).

- Isolate the useless wires: On the new 6-wire motors, ignore the 4 wires dedicated to the sensor (phase A, phase B, sensor VCC, and sensor GND). You can cut them or secure them with electrical tape.

- Connect the power: Take the only two remaining wires, namely the power ones marked as M+ and M-. Connect the M+ and M- of the right track to the Motor A terminals on the driver, and repeat the operation for the left track on the Motor B terminals.

> Note: to mount the new tracks I also had to print an intermediate piece that connected Emiglio's body to the chassis. Here it is, it goes mounted under Emiglio's feet:

![Desktop View](/assets/img/posts/emiglio/supportoint.png){: width="400px"}
_Internal support to connect the chassis to Emiglio_

## Useful resources for mounting the new tracks
- [tracks assembly manual](https://sposmart.com/#/Robot/FrameChassis/TS_Series/TS100/TStank)
- [official page link](https://sposmart.com/#/)

## The stability problem
Immediately after mounting the tracks on Emiglio, I take the remote control and start making it spin in place to test the lateral movement, which works correctly

![Desktop View](/assets/img/posts/emiglio/turn2.gif){: width="auto"}
_Emiglio spinning in place with the tracks_

Then I try to move it back and forth but it **pops a wheelie and falls to the ground**, so I decide to hang it with a cable from the ceiling and try to figure out how to solve it:

![Desktop View](/assets/img/posts/emiglio/impenna.gif){: width="auto"}
_Emiglio popping a wheelie safely hanging from the cable_

### The attempt to solve it with code

Initially, in a state of despair, I try to solve it with code (and I succeed but later I discover that on inclined surfaces it flips over anyway). Here is the **anti-flip** code portion:

```smooth-tank.py
# --- CONFIGURAZIONE PIN ---
PIN_X, PIN_Y = 17, 18  # Ingressi RC

STANDBY = 24

# MOTORE A (Sinistro)
AIN1, AIN2 = 4, 25     # PIN_UP e PIN_DOWN
PWMA = 8

# MOTORE B (Destro)
BIN1, BIN2 = 7, 27     # PIN_LEFT e PIN_RIGHT
PWMB = 11

# --- CONFIGURAZIONE FLUIDITA' (SMOOTH DOPPIO) ---
SMOOTH_TIME_FB_MS = 450  # Millisecondi per Avanti/Indietro
SMOOTH_TIME_LR_MS = 200  # Millisecondi per Destra/Sinistra (più reattivo)
LOOP_DELAY = 0.05        # Tempo di ciclo del while (50ms)
```
Basically I added a gradual acceleration instead of an instantaneous acceleration. Here is the result

![Desktop View](/assets/img/posts/emiglio/graduale.gif){: width="450px"}
_Emiglio accelerating gradually_

### The introduction of the rear caster wheel

After the code modification Emiglio was quite stable and no longer fell but I wanted to permanently solve the problem. I was suggested many ways to solve it and among all the solutions, reasoning about it, the one that convinced me the most was the rear swivel caster wheel because it's non-intrusive, simple, and it works.

Here is the model in Fusion360, it has shock absorbers to prevent impacts from damaging it:

![Desktop View](/assets/img/posts/emiglio/ruotino.jpg){: width="350px"}
_Rear caster wheel and 3D design_

- [In this folder you can find all the STL files](https://github.com/AlessandroBonomo28/AlessandroBonomo28.github.io/tree/main/assets/3dfiles/emiglio/)
## The final result

![Desktop View](/assets/img/posts/emiglio/mov.gif){: width="400px"}
_Rear caster wheel and 3D design_

Emiglio is now as stable as a tank and ready to go anywhere!
**STAY TUNED** for the next tutorials!

{% include embed/youtube.html id='zc8rK_VowUU' %}

### Future developments
I have so many ideas, for the future I'm working on several things:
- Full duplex AI that responds in real time
- Remote control and synchronization with Metaquest VR
- Emiglio modchip (embedded board)
