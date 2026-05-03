---
title: "Emiglio AI assistant locale (Parte 2)"
date: 2026-04-30 00:00:00 +0000
categories: [tutorials, electronics]
tags: [tutorial, blog, electronics, IoT, embedded]
---

In questo articolo spiego come ho costruito un assistente vocale completamente self-hosted basato su [Home Assistant](https://www.home-assistant.io/), Wyoming Satellite su un **Raspberry Pi Zero W2** con microfono ReSpeaker 2-Mic, e un **server home assistant** con **Ollama** + **llama3-groq-tool-use** capace di fare ricerche web in tempo reale tramite **SearXNG**.

![Desktop View](/assets/img/posts/emiglio/pizero.png){: width="auto"}
_Speaker Emiglio collegato a raspberry pi zero W2 + Scheda Respeaker_

{% include embed/youtube.html id='zcHoOtzRp1E' %}

## Link utili

- [Wyoming Satellite - Repository ufficiale](https://github.com/rhasspy/wyoming-satellite/tree/master)
- [Tutorial ufficiale ReSpeaker 2-Mic](https://github.com/rhasspy/wyoming-satellite/blob/master/docs/tutorial_2mic.md)
- [HACS - Download e troubleshooting](https://hacs.xyz/docs/use/download/download/#troubleshooting)
- [Extended OpenAI Conversation (HACS)](https://github.com/jekalmin/extended_openai_conversation)
- [Documentazione ufficiale scheda Respeaker2 Mics Pi HAT](https://wiki.seeedstudio.com/Respeaker_2_Mics_Pi_HAT/)

{% include embed/youtube.html id='XvbVePuP7NY' %}

> **ATTENZIONE** ho seguito esattamente gli step illustrati in questo tutorial youtube di NetworkChuck e posso dirvi che se provate a replicarli con un raspberry pi zero W 2 avrete problemi. In questo articolo vi spiego come risolverli e tutti gli step funzionanti da eseguire ad oggi 2026/05/01. Probabilmente il tutorial di Chuck funziona senza problemi su raspberry pi 3/4/5 ma **NON** su raspberry pi zero W2.
{: .prompt-danger }

## Architettura generale

Il sistema è composto da due macchine:

1. **Raspberry Pi Zero W2** — satellite vocale con microfono ReSpeaker 2-Mic, gira `wyoming-satellite` e ascolta la wake word
2. **Server** con almeno **8 GB di VRAM** — gira Home Assistant, Whisper (STT), Piper (TTS), Ollama e SearXNG, tutti in Docker

![Desktop View](/assets/img/posts/emiglio/gem.png){: width="auto"}
_Immagine generata con Gemini_

## Parte 1: Server — Docker Compose

Sul server, crea una cartella di progetto e un file `docker-compose.yml` con questo contenuto:

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

  # 2. Faster Whisper (Speech-to-Text in Italiano)
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

  # 3. Piper (Text-to-Speech in Italiano)
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

  # 4. SearXNG (motore di ricerca self-hosted)
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

Avvia tutto con:

```bash
docker compose up -d
```

### Installare Ollama sul server

Ollama deve girare con la porta aperta sulla rete locale, in modo che Home Assistant possa raggiungerlo:

```bash
# Installa Ollama
curl -fsSL https://ollama.com/install.sh | sh

# Scarica il modello
ollama pull llama3-groq-tool-use

# Apri la porta sul firewall
sudo ufw allow 11434
```

Assicurati che il server abbia almeno **8 GB di VRAM** disponibile per far girare il modello in modo fluido.

### SearXNG e configuration.yaml

SearXNG è già incluso nel docker-compose e gira sulla porta `8888`. Una volta avviato, aggiungi questa configurazione al file `configuration.yaml` di Home Assistant per esporre il comando di ricerca:

```yaml
rest_command:
  searxng_search:
    url: "http://192.168.1.83:8888/search?q={{ query }}&format=json"
    method: GET
    response_variable: results
```

Sostituisci `192.168.1.83` con l'IP del tuo server.

## Parte 2: Raspberry Pi Zero W2 — Wyoming Satellite con ReSpeaker 2-Mic

**Importante:** durante il setup del Pi, scegli **`pi`** come nome utente. I path nei file di servizio dipendono da `/home/pi` e usare un nome diverso causa problemi.

### 2.1 Dipendenze di sistema

```bash
sudo apt install git python3-venv python3-pip -y
```

### 2.2 Clona il repository

```bash
git clone https://github.com/rhasspy/wyoming-satellite.git
cd wyoming-satellite
```

### 2.3 Aumenta la swap (necessario su Pi Zero W2)

Il Pi Zero ha poca RAM, serve più swap per compilare i driver:

```bash
sudo apt install dphys-swapfile -y
sudo nano /etc/dphys-swapfile
```

Imposta:

```
CONF_SWAPSIZE=1024
```

Poi applica:

```bash
sudo dphys-swapfile setup
sudo dphys-swapfile swapon
```

### 2.4 Installa i driver ReSpeaker 2-Mic

Prima installa i kernel headers e le dipendenze manualmente:

```bash
sudo apt install linux-headers-rpi-v8 dkms i2c-tools libasound2-plugins alsa-utils -y
```

Poi apri lo script di installazione dei driver e **commenta** il blocco che tenta di rilevare automaticamente la versione kernel e scaricare i driver, perché su kernel recenti non funziona:

```bash
nano etc/install-respeaker-drivers.sh
```

Commenta queste righe (aggiungi `#` davanti):

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

Ora clona il fork aggiornato dei driver e installali:

```bash
cd ~
git clone -b v6.12 https://github.com/HinTak/seeed-voicecard
cd seeed-voicecard
sudo ./install.sh
sudo reboot
```

### 2.5 Ambiente Python

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

Se il file `pyproject.toml` contiene `zeroconf==0.88.0`, cambialo in `zeroconf>=0.133.0` prima di installare.

### 2.6 Verifica microfono e altoparlante

```bash
arecord -L
aplay -L
```

Dovresti vedere tra i dispositivi:

```
plughw:CARD=seeed2micvoicec,DEV=0
    seeed-2mic-voicecard, bcm2835-i2s-wm8960-hifi wm8960-hifi-0
    Hardware device with all software conversions
```

Registra 5 secondi e riascolta per verificare che funzioni:

```bash
arecord -D plughw:CARD=seeed2micvoicec,DEV=0 -r 16000 -c 1 -f S16_LE -t wav -d 5 test.wav
aplay -D plughw:CARD=seeed2micvoicec,DEV=0 test.wav
```

### 2.7 Test manuale del satellite

```bash
cd ~/wyoming-satellite
script/run \
  --debug \
  --name 'my satellite' \
  --uri 'tcp://0.0.0.0:10700' \
  --mic-command 'arecord -D plughw:CARD=seeed2micvoicec,DEV=0 -r 16000 -c 1 -f S16_LE -t raw' \
  --snd-command 'aplay -D plughw:CARD=seeed2micvoicec,DEV=0 -r 22050 -c 1 -f S16_LE -t raw'
```

### 2.8 Aggiungi il satellite a Home Assistant

Su Home Assistant vai in **Impostazioni → Dispositivi e servizi → Aggiungi integrazione → Wyoming Protocol**, inserisci l'IP del Pi e la porta `10700`. Home Assistant dovrebbe scoprire e aggiungere il satellite automaticamente.

### 2.9 Service systemd per avvio automatico

```bash
sudo systemctl edit --force --full wyoming-satellite.service
```

Incolla:

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

I parametri `--mic-auto-gain 5` e `--mic-noise-suppression 2` sono necessari per evitare che il satellite rimanga bloccato in ascolto infinito dopo la wake word. Dall'interfaccia di Home Assistant imposta anche: **mic audio = 5**, **noise suppression = medio**, **terminazione rilevamento vocale = aggressivo**.

### 2.10 Wake word con OpenWakeWord

```bash
sudo apt-get install --no-install-recommends libopenblas-dev -y
cd ~
git clone https://github.com/rhasspy/wyoming-openwakeword.git
cd wyoming-openwakeword
script/setup
```

OpenWakeWord include già alcuni modelli pronti all'uso. Le principali wake word disponibili di default sono `ok_nabu`, `hey_jarvis` e `alexa`. Per usarne una, modifica il service di openwakeword aggiungendo `--preload-model 'hey_jarvis'`, e il service del satellite aggiungendo `--wake-word-name 'hey_jarvis'`. Poi:

```bash
sudo systemctl daemon-reload
sudo systemctl restart wyoming-satellite.service
```

La prima volta l'avvio è lento perché il modello viene caricato in memoria.

#### Wake word personalizzata

Se nessuna delle parole incluse ti soddisfa, puoi usare un modello `.tflite` custom da questo repository: [home-assistant-wakewords-collection](https://github.com/fwartner/home-assistant-wakewords-collection).

Il satellite da solo non sa leggere i file `.tflite`: è openWakeWord che li gestisce, quindi il file va dato in pasto a lui.

**1. Scarica il modello**

```bash
mkdir -p ~/wyoming-satellite/custom_models
cd ~/wyoming-satellite/custom_models

# Esempio con il modello "jarvis"
wget https://github.com/fwartner/home-assistant-wakewords-collection/raw/main/trained_models/jarvis.tflite
```

**2. Configura openWakeWord per leggere la cartella custom**

Modifica il service di openwakeword e aggiungi `--custom-model-dir`:

```bash
sudo systemctl edit --force --full wyoming-openwakeword.service
```

Nella riga `ExecStart` aggiungi il parametro:

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

**3. Aggiorna il service del satellite**

Il nome da usare in `--wake-word-name` è il nome del file `.tflite` senza estensione:

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

### 2.11 LED ReSpeaker 2-Mic

```bash
cd ~/wyoming-satellite
.venv/bin/pip install 'pixel-ring'
sudo apt-get install python3-spidev python3-gpiozero -y
```

Crea il service per i LED:

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

## Parte 3: Agente AI con Extended OpenAI Conversation e SearXNG

Questa è la parte che trasforma il satellite da semplice comando vocale a un vero assistente AI con ricerca web in tempo reale.

### 3.1 Installa HACS e Extended OpenAI Conversation

Installa prima [HACS](https://hacs.xyz/docs/use/download/download/#troubleshooting) su Home Assistant, poi tramite HACS installa l'integrazione [Extended OpenAI Conversation](https://github.com/jekalmin/extended_openai_conversation).

Questa integrazione permette di collegare Home Assistant a qualsiasi endpoint compatibile OpenAI — incluso Ollama — e di definire **function calling** personalizzato, ad esempio per chiamare SearXNG.

### 3.2 Configurazione dell'agente

Il modello usato è `llama3-groq-tool-use` tramite Ollama, scelto perché supporta nativamente il function calling.

| Parametro | Valore |
|---|---|
| Modello | `llama3-groq-tool-use` |
| Max token | 1200 |
| Top P | 1 |
| Temperature | 0.05 |
| Max tool call per conversazione | 1 |
| Soglia contesto | 13000 |

La temperature bassa (0.05) serve a rendere le risposte più deterministiche e ad evitare che il modello improvvisi invece di chiamare il tool di ricerca quando dovrebbe.

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

### 3.3 Definizione della funzione web_search

Nella configurazione di Extended OpenAI Conversation, definisci la funzione `web_search`, [copiate lo snippet di codice da questo gist link](https://gist.github.com/AlessandroBonomo28/123ccc53aaa051358000ab73c43aa8f6)

Quando l'agente non conosce la risposta o viene interrogato su eventi recenti, richiama automaticamente SearXNG, ottiene i risultati e li include nella risposta vocale.

### Extra Button SHUTDOWN automatico su modulo Respeaker

Non spegnere mai il Pi togliendo la corrente brutalmente: corrompe la SD card.

Aggiungi un pulsante hardware sul GPIO 17 configurando

```bash
sudo nano /etc/firmware/config.txt
```

Aggiungi in fondo:

```
[all]
enable_uart=1
dtoverlay=i2s-mmap
dtparam=i2s=on
dtoverlay=gpio-shutdown,gpio_pin=17,active_low=1,gpio_pull=up
```

In alternativa, spegni sempre in modo sicuro via SSH:

```bash
ssh pi@<hostname>.local
sudo halt
# aspetta che i LED si spengano, poi togli la corrente
```

**questo step è importante** perchè prima o poi ci capiterà di dover spegnere il nostro raspberry e per non staccare brutalmente la corrente e corrompere i file della SD abbiamo bisogno di uno script automatico che spegne logicamente il raspberry quando premiamo il pulsante. Possiamo capire che il raspberry si sta spegnendo osservando il suo LED verde.

### Conclusione

Abbiamo configurato un'AI assistant locale con websearch che possiamo interrogare localmente e vocalmente con il nostro raspberry pi zero W2. **STAY TUNED** per altri tutorial
