---
lang: en
hidden: true
lang_ref: emiglio-1
permalink: /en/posts/Trasformare-Emiglio-in-un-AI-assistant/
title: "Emiglio robot with RC remote control (Part 1)"
date: 2026-04-10 00:00:00 +0000
categories: [tutorials, electronics]
tags: [tutorial, blog, electronics, IoT, embedded]
image:
  path: /assets/img/posts/emiglio/radiocommm.png
  alt: Emiglio disassembled with the RC remote control and control electronics
---

{% include embed/youtube.html id='oEHIA1CD1nk' %}

### The story
One day, while browsing a used items website, I came across **an old Emiglio for 20€**. I've always wanted to have a personal "jarvis" robot like in Iron Man and, without a second thought, I wrote to the seller to buy it. The seller told me they don't ship, so I got in my car and set off to retrieve Emiglio.

![Desktop View](/assets/img/posts/emiglio/usato.png){: width="300"}
_The online ad_

Emiglio is in pretty bad shape:

- missing the remote control
- missing the tray
- the backpack and head are damaged

There will be some work to do to fix him up but he's worth all the 20€ spent.

### The restoration

I decided to start with the aesthetic part. I took the measurements of Emiglio's damaged parts and reprinted them with a 3D printer. I also had to open Emiglio, disassemble the DC motors and replace the plastic washers since they were damaged and didn't let the cart wheels turn.  

[In this folder you can find all the STL files](https://github.com/AlessandroBonomo28/AlessandroBonomo28.github.io/tree/main/assets/3dfiles/emiglio/) after printing the parts and assembling them, this is the final result:

![Desktop View](/assets/img/posts/emiglio/img2.png){: width="500"}
_Emiglio as good as new_

![Desktop View](/assets/img/posts/emiglio/img1.png){: width="500"}
_CAD view_

Now that Emiglio looks better and the mechanics of the wheel gears work and correctly allow the DC motors to turn the wheels, we can move on to the remote control phase.
### The airplane remote control

![Desktop View](/assets/img/posts/emiglio/radiocommm.png){: width="500"}
_Remote control and circuit soldered on PCB_

Since I don't have the original remote control, I made do with a 2.4Ghz airplane remote control. This type of remote control transmits commands via PWM (Pulse Width Modulation) signals: each channel sends a pulse with a variable duration between about 1000µs and 2000µs, where the central value (~1500µs) corresponds to the neutral point of the stick.
The radio receiver is connected directly to the GPIO pins of the **Raspberry Pi 1 B**. To read these signals with precision I used the pigpio library, which allows to measure the duration of the pulses in a hardware-accurate way without depending on the operating system timings. The [TB6612FNG driver](https://amzn.to/43Z6G8c) takes care of driving the 4 DC motors by receiving the direction and PWM signals from the Raspberry's GPIOs via the `tank.py` script, which implements a tank-like movement logic: the wheels on the two sides are managed independently to allow turning.

![Desktop View](/assets/img/posts/emiglio/schema.jpg){: width="auto"}
_Wiring diagram_

All [the code is opensource on github here](https://github.com/AlessandroBonomo28/emiglio-controller)

### Parts list:
- [Raspberry pi 4 B (better than Pi 1 B)](https://amzn.to/3R2ipQi)
- [12V rechargeable battery](https://amzn.to/3SOVmZL)
- [TB6612FNG motor driver](https://amzn.to/43Z6G8c)

## Final result

{% include embed/youtube.html id='I8c7Y01DfZw' %}

The motors are now powered by a 12V rechargeable battery and the Raspberry by a powerbank, both conveniently housed in Emiglio's backpack.
I also drilled some holes with a Dremel on the back of Emiglio to let out a Power LED, the radio antenna, the battery cables, the powerbank switch and the logical shutdown button of the Raspberry — necessary because brutally turning off the power corrupts the SD card. The `tank.py` and `btn.py` services start automatically at boot via systemd, so Emiglio is operational as soon as he is turned on, without needing a monitor or to connect via SSH.

### Part 2

In part 2 we see how to use a raspberry pi W2 and a respeaker module to interact vocally with Emiglio. [Click here to read part 2](https://alessandrobonomo28.github.io/en/posts/Trasformare-Emiglio-in-un-AI-assistant-2/)
