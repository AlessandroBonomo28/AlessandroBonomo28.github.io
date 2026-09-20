---
lang: en
hidden: true
lang_ref: emiglio-3
permalink: /en/posts/Trasformare-Emiglio-in-un-AI-assistant-3/
title: "Emiglio bluetooth speaker (Part 3) "
date: 2026-06-24 00:00:00 +0000
categories: [tutorials, electronics]
tags: [tutorial, blog, electronics, IoT, embedded]
image:
  path: /assets/img/posts/emiglio/embluetooth.jpg
  alt: Emiglio used as a bluetooth speaker by a smartphone
---

# Transform Emiglio into a bluetooth speaker

{% include embed/youtube.html id='g96JaZwjHMA' %}

In [Part 2](/en/posts/Trasformare-Emiglio-in-un-AI-assistant-2/) we tried to make Emiglio smart by configuring a local AI assistant.
Today we see how to transform Emiglio into a bluetooth speaker so you can easily play audio by connecting with your phone.

![Desktop View](/assets/img/posts/emiglio/embluetooth.jpg){: width="auto"}
_I see emiglio as a bluetooth speaker_

### Required parts
- [Raspberry pi zero W2](https://amzn.to/3SuylLD)
- [Respeaker module](https://amzn.to/4oQRt2t)

In the previous tutorial we configured the **Raspberry pi zero W2** and installed the **Respeaker sound card drivers**, so it already connects automatically to our local network and we can connect to it via **ssh** to program it. If you are starting from scratch then you must first flash the Pi OS lite operating system on the raspberry with PI imager and then install the drivers.

#### Verify that microphone and speaker work

```bash
arecord -L
aplay -L
```

You should see among the devices:

```
plughw:CARD=seeed2micvoicec,DEV=0
    seeed-2mic-voicecard, bcm2835-i2s-wm8960-hifi wm8960-hifi-0
    Hardware device with all software conversions
```

Record 5 seconds and play it back to verify that it works:

```bash
arecord -D plughw:CARD=seeed2micvoicec,DEV=0 -r 16000 -c 1 -f S16_LE -t wav -d 5 test.wav
aplay -D plughw:CARD=seeed2micvoicec,DEV=0 test.wav
```
# How to turn the Raspberry Pi into a Bluetooth speaker

---

## First of all: how does the music "travel"?

Before typing commands, let's understand the path the music takes from the phone to the ReSpeaker cones. We will use a very lightweight stack for the Pi Zero's CPU:

1. **BlueZ** is the Linux Bluetooth stack, it hooks the phone's antenna and receives the raw audio stream in the *A2DP* protocol (the protocol that earphones and speakers use for stereo music).
2. **BlueALSA** acts as a **translator**: it takes the packets from BlueZ and transforms them into a normal PCM audio stream ("uncompressed" audio, the format the rest of the system understands).
3. **ALSA** is the Kernel's low-level audio engine. It receives the PCM, applies mathematical *resampling* to 48kHz on the fly (I explain why this is needed below) and delivers it to the sound card drivers.
4. The **`bluealsa-aplay`** daemon process listens perpetually on BlueALSA and, as soon as bytes arrive, it physically pushes them into the speaker.

![Desktop View](/assets/img/posts/emiglio/bluetooth.png){: width="auto"}
_Bluetooth stack_


To make this stack work, however, we have to eliminate a program that would otherwise conflict with BlueALSA (we see it in Step 1).

---

## Checks before Step 1

These three checks save you an afternoon of cursing at the computer:

* **Are you using Raspberry Pi OS "Bookworm" (Debian 12) or later (e.g. Trixie)?**
  Type:
  ```bash
  cat /etc/os-release
  ```
  On the `VERSION_CODENAME` line you must read **bookworm** or **trixie**. If you read *bullseye* or *buster*, **stop**: the package we will install in Step 2 doesn't exist in the old repositories. Update the operating system first, otherwise you won't get anywhere.

* **Is the hardware Bluetooth turned on?**
  If in the past you had disabled BT to save RAM, turn it back on:
  ```bash
  sudo raspi-config
  ```
  and go to *System Options*, *Network at Boot / Bluetooth*, then turn it on.

---

## Step 1: Remove Bluetooth from PipeWire

**Why:** on Bookworm *Desktop* images the default audio server is PipeWire, which also manages Bluetooth and therefore conflicts with BlueALSA. To solve this just remove the **PipeWire Bluetooth plugin** (`libspa-0.2-bluetooth`): PipeWire stops dealing with Bluetooth (which passes to BlueALSA) but continues to manage local and USB audio.

On *Lite* images (typical of a headless Pi) PipeWire is usually **not installed**: in that case this step is not needed and you can skip to Step 2.

First check if the plugin is present:
```bash
sudo apt update --fix-missing
apt list --installed 2>/dev/null | grep -E "libspa-0.2-bluetooth|pipewire"
```

If `libspa-0.2-bluetooth` appears, remove it:
```bash
sudo apt remove libspa-0.2-bluetooth -y
```

> *Optional:* you can also remove `pipewire-pulse` if you don't use applications that talk to the PulseAudio API (`pactl`). It is not necessary for Bluetooth.

If PipeWire is installed, restart it to apply the plugin removal:
```bash
systemctl --user restart pipewire wireplumber
```
*(Any warnings are normal: you are restarting a service from which you have just removed a component.)*

---

## Step 2: Install BlueALSA and configure ALSA

**Why:** we install BlueALSA and the tools to manage the board, then we write a rule that tells ALSA where to send the sound.

Install the packages:
```bash
sudo apt install bluez-alsa-utils alsa-utils -y
```

Now let's create the `~/.asoundrc` file (the `~` is your personal folder; the initial dot makes it a hidden configuration file).

Hardware note: the ReSpeaker has a quartz oscillator fixed at **48,000 Hz**, so the board only works at that frequency. If the phone transmits an MP3 at 44,100 Hz, without adaptation the board goes into error. The `type plug` parameter orders ALSA to perform *resampling* at 48,000 Hz in real time and transparently.

Open the file:
```bash
nano ~/.asoundrc
```
Delete any content and paste exactly this block (in `nano`: you paste, then `Ctrl+O` and `Enter` to save, `Ctrl+X` to exit):

```text
defaults.bluealsa.interface "hci0"
defaults.bluealsa.device "00:00:00:00:00:00"
defaults.bluealsa.profile "a2dp"

pcm.!default {
    type plug
    slave.pcm "plughw:CARD=seeed2micvoicec,DEV=0"
}
```

> **Meaning of the lines:**
> - `hci0` is the Pi's Bluetooth antenna (almost always `hci0`).
> - `00:00:00:00:00:00` is not a real address: it is a special value that means *"any device"*. Leave it like this.
> - `a2dp` is the stereo music profile.
> - The `pcm.!default` block sets the ReSpeaker board (`seeed2micvoicec`) as the default output, with `type plug` for automatic frequency adaptation.

---

## Step 3: Verify the `a2dp-sink` profile

**Why:** BlueALSA can work as a **source** (`a2dp-source`, sends audio to a speaker) or as a **sink** (`a2dp-sink`, receives audio from a phone). For a speaker the **sink** role is needed. On most Bookworm installations the `bluez-alsa` service is already configured to offer `a2dp-sink`, so often this step is just a verification.

Check the daemon startup line:
```bash
systemctl cat bluealsa.service | grep -i exec
```

Add the override **only if** `-p a2dp-sink` is missing in the `ExecStart` **and** the phone connects and then immediately disconnects. In that case:
```bash
sudo systemctl edit bluealsa.service
```
Paste exactly these lines in the space indicated:
```ini
[Service]
ExecStart=
ExecStart=/usr/bin/bluealsa -p a2dp-sink
```

> **The empty `ExecStart=` line is not an error.** The first line resets the default command, the second sets the new one. Without the empty line, systemd rejects the configuration. This is a specific behavior of systemd overrides.

Save and exit, then reload and restart:
```bash
sudo systemctl daemon-reload
sudo systemctl restart bluealsa
```

> *If in the future you want the Pi to also be able to send music to other speakers, use `-p a2dp-source -p a2dp-sink`. For a speaker `-p a2dp-sink` is sufficient.*

---

## Step 4: Raise the board volumes

**Why:** the ReSpeaker's WM8960 chip comes out of the factory with volume registers at zero. The audio results as playing but you hear nothing until you raise these volumes (one-time operation).

1. Open the board mixer:
   ```bash
   alsamixer -D plughw:CARD=seeed2micvoicec,DEV=0
   ```
2. With the **right/left arrows** you move between the channels, with **up/down** you adjust the level. Bring **Playback**, **Headphone** and **Speaker** to **85%**.
   > *Volume controls in ALSA follow a logarithmic curve: below about 60% the sound is almost inaudible, and the last stretch contains the useful part. For this reason a high value is set.*
3. If under a channel you read `[MM]`, it's on mute: press the `M` key to unlock it (it becomes `[OO]`).
4. Press `Esc` to exit, then save the settings in the board's memory (so they remain after a reboot):
   ```bash
   sudo alsactl store
   ```

---

## Step 5: Make Bluetooth always visible

**Why:** by default BlueZ makes the device no longer discoverable after 180 seconds. For an always available speaker it's better to disable this timeout.

Open the configuration file:
```bash
sudo nano /etc/bluetooth/main.conf
```
In the `[General]` block find `DiscoverableTimeout` and set it to zero (0 = no expiration):
```text
DiscoverableTimeout = 0
```
Save and exit, then restart the service:
```bash
sudo systemctl restart bluetooth
```

---

## Step 6: Pair the phone and set trust

**Why:** for a personal device a script that confirms the PIN at every connection is not needed. It is sufficient to pair once and mark the phone as *trusted*: from that moment reconnection is automatic.

1. Enter the Bluetooth interactive console:
   ```bash
   sudo bluetoothctl
   ```
2. Set the agent without interface (`NoInputNoOutput` indicates that the Pi has neither screen nor keyboard to enter a PIN, so it accepts "Just Works" pairing):
   ```text
   power on
   discoverable on
   pairable on
   agent NoInputNoOutput
   default-agent
   ```
3. On the phone: go to Bluetooth settings and, if the Pi already appears, choose "Forget this device" (to delete a possible previous faulty pairing). Disable and re-enable Bluetooth, then tap the name of your Pi (its hostname).
4. On the Raspberry you will see the connection lines scroll by. Copy the phone's MAC Address that appears.
   > **MAC Format:** must be written with colons, e.g. `AA:BB:CC:11:22:33`, not all attached. A wrong format makes the command fail without clear messages.

   Then, still inside `bluetoothctl`, mark the phone as trusted (replace with the real MAC of your device):
   ```text
   trust AA:BB:CC:11:22:33
   exit
   ```

From now on, when that phone is nearby and has Bluetooth active, reconnection happens automatically.

---

## Step 7: Autostart of the BlueALSA player

**Why:** `bluealsa-aplay` is the process that routes Bluetooth audio to the board. To have it always active we configure it as a systemd service, which starts at boot and restarts in case of error.

Create the service file:
```bash
sudo nano /etc/systemd/system/bt-speaker.service
```

Content:
```ini
[Unit]
Description=Bluetooth Audio Player (BlueALSA)
After=bluetooth.service bluealsa.service
Requires=bluetooth.service bluealsa.service

[Service]
Type=simple
User=pi
ExecStart=/usr/bin/bluealsa-aplay --profile-a2dp 00:00:00:00:00:00
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

> **Notes:**
> - `After`/`Requires`: the player doesn't start before Bluetooth and BlueALSA are ready.
> - `User=pi`: if your user is not called `pi`, change this field. The indicated user must be in the `audio` group (see the initial checks).
> - `00:00:00:00:00:00`: again the "any device" value.
> - `Restart=on-failure`: the service restarts on its own after 5 seconds if it stops.

Save and exit, then enable and start the service:
```bash
sudo systemctl daemon-reload
sudo systemctl enable --now bt-speaker.service
```

Verify the state:
```bash
sudo systemctl status bt-speaker.service
```
If you see `active (running)`, the service is active.

---

## Final verification

Restart the Pi and, without touching anything else, unlock the phone and start a track: the audio should come out of the ReSpeaker speaker.

This configuration **coexists with a local server like the Flask Soundboard**: having removed only Bluetooth management from PipeWire, PipeWire continues to play local `.wav` files on the same board, while Bluetooth passes through BlueALSA. The two paths no longer compete for Bluetooth.

> Note on simultaneous playback: direct access to the board (`plughw`) can be exclusive, so if you want local audio and Bluetooth to play **at the same instant** you should verify it on your hardware. In many cases PipeWire releases the board when it's not playing anything, leaving it free for Bluetooth.

In the next part we will see the integration of the **Wake Word engine** ("Hey Emiglio") running locally on the Pi Zero.

---
## Final result

Now you can connect to Emiglio's bluetooth and make him say all the nonsense that comes to your mind


{% include embed/youtube.html id='dE9uiRTsfVo' %}


## Troubleshooting

- **The phone doesn't see the speaker, or connects and disconnects immediately** → verify the `a2dp-sink` profile (Step 3).
- **It connects but nothing is heard** → check the volumes with `alsamixer` (Step 4): channels not at zero and not in `[MM]`.
- **`Permission denied` on the sound card** → the user is not in the `audio` group (initial checks) or the `User=` field of Step 7 is wrong.
- **It works in the session but not after a reboot** → check that the service in Step 7 is `enabled` and that `bluealsa-aplay` restarts on its own: `systemctl status bt-speaker.service`.
- **Inconsistent behavior after installation** → a `sudo reboot` allows all services to start in the correct order.
