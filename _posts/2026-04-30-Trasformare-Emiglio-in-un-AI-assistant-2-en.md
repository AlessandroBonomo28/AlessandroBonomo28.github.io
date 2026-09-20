---
lang: en
hidden: true
lang_ref: emiglio-2
permalink: /en/posts/Trasformare-Emiglio-in-un-AI-assistant-2/
title: "Emiglio local AI assistant (Part 2)"
date: 2026-04-30 00:00:00 +0000
categories: [tutorials, electronics]
tags: [tutorial, blog, electronics, IoT, embedded]
image:
  path: /assets/img/posts/emiglio/gem.png
  alt: "General system architecture: Raspberry Pi Zero W2 voice satellite and server with Home Assistant, Whisper, Piper, Ollama and SearXNG"
---

In this article I explain how I built a completely self-hosted voice assistant based on [Home Assistant](https://www.home-assistant.io/), Wyoming Satellite on a **Raspberry Pi Zero W2** with ReSpeaker 2-Mic microphone, and a **home assistant server** with **Ollama** + **llama3-groq-tool-use** capable of doing web searches in real time via **SearXNG**.

![Desktop View](/assets/img/posts/emiglio/pizero.png){: width="auto"}
_Emiglio speaker connected to raspberry pi zero W2 + Respeaker Board_

{% include embed/youtube.html id='zcHoOtzRp1E' %}

### Parts list
- [Raspberry pi zero W2](https://amzn.to/3SuylLD)
- [Respeaker module](https://amzn.to/4oQRt2t)
- [Raspberry pi 4 B (better than Pi 1 B)](https://amzn.to/3R2ipQi)
- [12V rechargeable battery](https://amzn.to/3SOVmZL)
- [TB6612FNG motor driver](https://amzn.to/43Z6G8c)

{% include embed/youtube.html id='XvbVePuP7NY' %}

> **WARNING** I followed exactly the steps illustrated in this youtube tutorial by NetworkChuck and I can tell you that if you try to replicate them with a raspberry pi zero W 2 you will have problems. In this article I'll explain how to solve them and all the working steps to perform as of today 2026/05/01. Probably Chuck's tutorial works without problems on raspberry pi 3/4/5 but **NOT** on raspberry pi zero W2.
{: .prompt-danger }

## General architecture

The system is composed of two machines:

1. **Raspberry Pi Zero W2** — voice satellite with ReSpeaker 2-Mic microphone, runs `wyoming-satellite` and listens for the wake word
2. **Server** with at least **8 GB of VRAM** — runs Home Assistant, Whisper (STT), Piper (TTS), Ollama and SearXNG, all in Docker

![Desktop View](/assets/img/posts/emiglio/gem.png){: width="auto"}
_Image generated with Gemini_

## Part 1: Server — Docker Compose

On the server, create a project folder and a `docker-compose.yml` file with this content:

```yaml
services:
  # 1. Home Assistant Core
  homeassistant:
    container_name: homeassistant
    image: "ghcr.io/home-assistant/home-assistant:stable"
    volumes:
      - ./config:/config
      - /etc/localtime:/etc/localtime:ro
    restart: unless-stopped
    privileged: true
    network_mode: host

  # 2. Faster Whisper (Speech-to-Text in Italian)
  whisper:
    container_name: wyoming-whisper
    image: rhasspy/wyoming-whisper
    command: --model base --language it
    volumes:
      - ./whisper_data:/data
    environment:
      - TZ=Europe/Rome
    restart: unless-stopped
    ports:
      - "10300:10300"

  # 3. Piper (Text-to-Speech in Italian)
  piper:
    container_name: wyoming-piper
    image: rhasspy/wyoming-piper
    command: --voice it_IT-riccardo-x_low
    volumes:
      - ./piper_data:/data
    environment:
      - TZ=Europe/Rome
    restart: unless-stopped
    ports:
      - "10200:10200"

  # 4. SearXNG (self-hosted search engine)
  searxng:
    image: docker.io/searxng/searxng:latest
    container_name: searxng
    restart: unless-stopped
    ports:
      - "8888:8080"
    volumes:
      - ./config:/etc/searxng
      - ./data:/var/cache/searxng
```

Start everything with:

```bash
docker compose up -d
```

### Install Ollama on the server

Ollama must run with the port open on the local network, so that Home Assistant can reach it:

```bash
# Install Ollama
curl -fsSL https://ollama.com/install.sh | sh

# Download the model
ollama pull llama3-groq-tool-use

# Open the port on the firewall
sudo ufw allow 11434
```

Make sure the server has at least **8 GB of VRAM** available to run the model smoothly.

### SearXNG and configuration.yaml

SearXNG is already included in the docker-compose and runs on port `8888`. Once started, add this configuration to Home Assistant's `configuration.yaml` file to expose the search command:

```yaml
rest_command:
  searxng_search:
    url: "http://192.168.1.83:8888/search?q={{ query }}&format=json"
    method: GET
    response_variable: results
```

Replace `192.168.1.83` with your server's IP.

## Part 2: Raspberry Pi Zero W2 — Wyoming Satellite with ReSpeaker 2-Mic

**Important:** during the Pi setup, choose **`pi`** as the username. The paths in the service files depend on `/home/pi` and using a different name causes problems.

### 2.1 System dependencies

```bash
sudo apt install git python3-venv python3-pip -y
```

### 2.2 Clone the repository

```bash
git clone https://github.com/rhasspy/wyoming-satellite.git
cd wyoming-satellite
```

### 2.3 Increase swap (necessary on Pi Zero W2)

The Pi Zero has little RAM, it needs more swap to compile the drivers:

```bash
sudo apt install dphys-swapfile -y
sudo nano /etc/dphys-swapfile
```

Set:

```
CONF_SWAPSIZE=1024
```

Then apply:

```bash
sudo dphys-swapfile setup
sudo dphys-swapfile swapon
```

### 2.4 Install the ReSpeaker 2-Mic drivers

First install the kernel headers and dependencies manually:

```bash
sudo apt install linux-headers-rpi-v8 dkms i2c-tools libasound2-plugins alsa-utils -y
```

Then open the driver installation script and **comment** the block that attempts to automatically detect the kernel version and download the drivers, because it doesn't work on recent kernels:

```bash
nano etc/install-respeaker-drivers.sh
```

Comment these lines (add `#` in front):

```bash
#kernel_formatted="$(uname -r | cut -f1,2 -d.)"
#driver_url_status="$(curl -ILs https://github.com/HinTak/seeed-voicecard/archive/refs/heads/v$kernel_formatted.tar.gz | tac | grep -o "^HTTP.*" | cut -f 2 -d' ' | head -1)"
#if  [ ! "$driver_url_status" = 200 ]; then
#echo "Could not find driver for kernel $kernel_formatted"
#exit 1
#fi
#apt-get update
#apt-get install --no-install-recommends --yes \
#    curl raspberrypi-kernel-headers dkms i2c-tools libasound2-plugins alsa-utils
```

Now clone the updated driver fork and install them:

```bash
cd ~
git clone -b v6.12 https://github.com/HinTak/seeed-voicecard
cd seeed-voicecard
sudo ./install.sh
sudo reboot
```

### 2.5 Python Environment

```bash
sudo apt install python3-numpy python3-zeroconf python3-setuptools python3-pip -y

cd ~/wyoming-satellite
rm -rf .venv
python -m venv .venv --system-site-packages

.venv/bin/pip install --upgrade pip
.venv/bin/pip install wyoming==1.5.4
.venv/bin/pip install "zeroconf>=0.133.0"
.venv/bin/pip install "pyring-buffer>=1,<2"
.venv/bin/pip install --no-deps -e .
.venv/bin/pip install --no-deps pysilero-vad==1.0.0 webrtc-noise-gain==1.2.3
```

If the `pyproject.toml` file contains `zeroconf==0.88.0`, change it to `zeroconf>=0.133.0` before installing.

### 2.6 Verify microphone and speaker

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

### 2.7 Satellite manual test

```bash
cd ~/wyoming-satellite
script/run \
  --debug \
  --name 'my satellite' \
  --uri 'tcp://0.0.0.0:10700' \
  --mic-command 'arecord -D plughw:CARD=seeed2micvoicec,DEV=0 -r 16000 -c 1 -f S16_LE -t raw' \
  --snd-command 'aplay -D plughw:CARD=seeed2micvoicec,DEV=0 -r 22050 -c 1 -f S16_LE -t raw'
```

### 2.8 Add the satellite to Home Assistant

On Home Assistant go to **Settings → Devices & Services → Add Integration → Wyoming Protocol**, enter the Pi's IP and port `10700`. Home Assistant should automatically discover and add the satellite.

### 2.9 systemd service for automatic startup

```bash
sudo systemctl edit --force --full wyoming-satellite.service
```

Paste:

```ini
[Unit]
Description=Wyoming Satellite
Wants=network-online.target
After=network-online.target

[Service]
Type=simple
ExecStart=/home/pi/wyoming-satellite/script/run \
  --name 'my satellite' \
  --uri 'tcp://0.0.0.0:10700' \
  --mic-command 'arecord -D plughw:CARD=seeed2micvoicec,DEV=0 -r 16000 -c 1 -f S16_LE -t raw' \
  --snd-command 'aplay -D plughw:CARD=seeed2micvoicec,DEV=0 -r 22050 -c 1 -f S16_LE -t raw' \
  --mic-auto-gain 5 \
  --mic-noise-suppression 2
WorkingDirectory=/home/pi/wyoming-satellite
Restart=always
RestartSec=1

[Install]
WantedBy=default.target
```

```bash
sudo systemctl enable --now wyoming-satellite.service
```

The `--mic-auto-gain 5` and `--mic-noise-suppression 2` parameters are necessary to prevent the satellite from getting stuck in infinite listening after the wake word. From the Home Assistant interface also set: **mic audio = 5**, **noise suppression = medium**, **voice detection termination = aggressive**.

### 2.10 Wake word with OpenWakeWord

```bash
sudo apt-get install --no-install-recommends libopenblas-dev -y
cd ~
git clone https://github.com/rhasspy/wyoming-openwakeword.git
cd wyoming-openwakeword
script/setup
```

OpenWakeWord already includes some ready-to-use models. The main default available wake words are `ok_nabu`, `hey_jarvis` and `alexa`. To use one, edit the openwakeword service by adding `--preload-model 'hey_jarvis'`, and the satellite service by adding `--wake-word-name 'hey_jarvis'`. Then:

```bash
sudo systemctl daemon-reload
sudo systemctl restart wyoming-satellite.service
```

The first time the startup is slow because the model is loaded into memory.

#### Custom wake word

If none of the included words satisfy you, you can use a custom `.tflite` model from this repository: [home-assistant-wakewords-collection](https://github.com/fwartner/home-assistant-wakewords-collection).

The satellite alone does not know how to read `.tflite` files: it is openWakeWord that manages them, so the file must be fed to it.

**1. Download the model**

```bash
mkdir -p ~/wyoming-satellite/custom_models
cd ~/wyoming-satellite/custom_models

# Example with the "jarvis" model
wget https://github.com/fwartner/home-assistant-wakewords-collection/raw/main/trained_models/jarvis.tflite
```

**2. Configure openWakeWord to read the custom folder**

Edit the openwakeword service and add `--custom-model-dir`:

```bash
sudo systemctl edit --force --full wyoming-openwakeword.service
```

In the `ExecStart` line add the parameter:

```ini
ExecStart=/home/pi/wyoming-openwakeword/script/run \
  --uri 'tcp://127.0.0.1:10400' \
  --custom-model-dir /home/pi/wyoming-satellite/custom_models \
  --preload-model 'jarvis'
```

```bash
sudo systemctl daemon-reload
sudo systemctl restart wyoming-openwakeword.service
```

**3. Update the satellite service**

The name to use in `--wake-word-name` is the name of the `.tflite` file without extension:

```ini
ExecStart=/home/pi/wyoming-satellite/script/run \
  --name 'my satellite' \
  --uri 'tcp://0.0.0.0:10700' \
  --mic-command 'arecord -D plughw:CARD=seeed2micvoicec,DEV=0 -r 16000 -c 1 -f S16_LE -t raw' \
  --snd-command 'aplay -D plughw:CARD=seeed2micvoicec,DEV=0 -r 22050 -c 1 -f S16_LE -t raw' \
  --mic-auto-gain 5 \
  --mic-noise-suppression 2 \
  --wake-uri 'tcp://127.0.0.1:10400' \
  --wake-word-name 'jarvis'
```

```bash
sudo systemctl daemon-reload
sudo systemctl restart wyoming-satellite.service
```

### 2.11 ReSpeaker 2-Mic LEDs

```bash
cd ~/wyoming-satellite
.venv/bin/pip install 'pixel-ring'
sudo apt-get install python3-spidev python3-gpiozero -y
```

Create the service for the LEDs:

```bash
sudo systemctl edit --force --full 2mic-leds.service
```

```ini
[Unit]
Description=2Mic LEDs

[Service]
Type=simple
ExecStart=/home/pi/wyoming-satellite/.venv/bin/python /home/pi/wyoming-satellite/examples/2mic_service.py --uri 'tcp://127.0.0.1:10500'
WorkingDirectory=/home/pi/wyoming-satellite/examples
Restart=always
RestartSec=1

[Install]
WantedBy=default.target
```

```bash
sudo systemctl enable --now 2mic-leds.service
```

## Part 3: AI Agent with Extended OpenAI Conversation and SearXNG

This is the part that turns the satellite from a simple voice command into a real AI assistant with real-time web search.

### 3.1 Install HACS and Extended OpenAI Conversation

First install [HACS](https://hacs.xyz/docs/use/download/download/#troubleshooting) on Home Assistant, then via HACS install the [Extended OpenAI Conversation](https://github.com/jekalmin/extended_openai_conversation) integration.

This integration allows you to connect Home Assistant to any OpenAI compatible endpoint — including Ollama — and define custom **function calling**, for example to call SearXNG.

### 3.2 Agent configuration

The model used is `llama3-groq-tool-use` via Ollama, chosen because it natively supports function calling.

| Parameter | Value |
|---|---|
| Model | `llama3-groq-tool-use` |
| Max token | 1200 |
| Top P | 1 |
| Temperature | 0.05 |
| Max tool call per conversation | 1 |
| Context threshold | 13000 |

The low temperature (0.05) is used to make the answers more deterministic and avoid the model improvising instead of calling the search tool when it should.

**System prompt:**

```
If the user asks for facts, news, real-world information, or current events, YOU MUST CALL 'web_search'.
DO NOT write sentences. DO NOT say "I will search".
OUTPUT ONLY THE JSON:
{
  "function_call": {
    "name": "web_search",
    "arguments": "{\"query\": \"DOMANDA_QUI\"}"
  }
}
Play along with the user, be also sarcastic. NEVER say "I am an AI", "I don't have opinions", or "I cannot form subjective thoughts".
ANSWER ONLY IN ITALIAN.
```

### 3.3 Definition of the web_search function

In the Extended OpenAI Conversation configuration, define the `web_search` function, [copy the code snippet from this gist link](https://gist.github.com/AlessandroBonomo28/123ccc53aaa051358000ab73c43aa8f6)

When the agent doesn't know the answer or is queried on recent events, it automatically calls SearXNG, gets the results and includes them in the voice answer.

### Extra automatic SHUTDOWN Button on Respeaker module

Never turn off the Pi by brutally removing the power: it corrupts the SD card.

Add a hardware button on GPIO 17 by configuring

```bash
sudo nano /etc/firmware/config.txt
```

Add at the bottom:

```
[all]
enable_uart=1
dtoverlay=i2s-mmap
dtparam=i2s=on
dtoverlay=gpio-shutdown,gpio_pin=17,active_low=1,gpio_pull=up
```

Alternatively, always turn off safely via SSH:

```bash
ssh pi@<hostname>.local
sudo halt
# wait for the LEDs to turn off, then remove power
```

**this step is important** because sooner or later we will have to turn off our raspberry and to avoid brutally unplugging the power and corrupting the SD files we need an automatic script that logically turns off the raspberry when we press the button. We can understand that the raspberry is shutting down by observing its green LED.

### Conclusion

We have configured a local AI assistant with websearch that we can query locally and vocally with our raspberry pi zero W2. **STAY TUNED** for more tutorials

## Useful links

- [Wyoming Satellite - Official repository](https://github.com/rhasspy/wyoming-satellite/tree/master)
- [Official ReSpeaker 2-Mic tutorial](https://github.com/rhasspy/wyoming-satellite/blob/master/docs/tutorial_2mic.md)
- [HACS - Download and troubleshooting](https://hacs.xyz/docs/use/download/download/#troubleshooting)
- [Extended OpenAI Conversation (HACS)](https://github.com/jekalmin/extended_openai_conversation)
- [Official Respeaker2 Mics Pi HAT board documentation](https://wiki.seeedstudio.com/Respeaker_2_Mics_Pi_HAT/)

### Part 3

In part 3 we see how to configure Emiglio as a bluetooth speaker to connect directly with the phone and play audio (without writing code). [Click here to read part 3](https://alessandrobonomo28.github.io/en/posts/Trasformare-Emiglio-in-un-AI-assistant-3/)
