---
lang: en
hidden: true
lang_ref: emiglio-6
permalink: /en/posts/Trasformare-Emiglio-in-un-AI-assistant-6/
title: "Emiglio sees, listens and speaks in real time (Part 6)"
description: Emiglio streams video and audio from the Raspberry Pi to MiniCPM-o 4.5 in full duplex, and the conversation no longer freezes after a few minutes.
date: 2026-09-16 10:00:00 +0200
categories: [tutorials, electronics]
tags: [tutorial, blog, electronics, IoT, embedded, AI, raspberry, streaming]
image:
  path: /assets/img/posts/emiglio-stream/robot-demo.png
  alt: Emiglio sends video and audio to the PC, MiniCPM-o responds with its voice
toc: true
---

# A full duplex AI for Emiglio: streaming from the Raspberry Pi to MiniCPM-o 4.5

In the first five parts Emiglio learned to move with the [RC remote control](/en/posts/Trasformare-Emiglio-in-un-AI-assistant/), to [reason locally](/en/posts/Trasformare-Emiglio-in-un-AI-assistant-2/), to [play via bluetooth](/en/posts/Trasformare-Emiglio-in-un-AI-assistant-3/), to [walk on tracks](/en/posts/Trasformare-Emiglio-in-un-AI-assistant-4/) and received [a board all of its own](/en/posts/Trasformare-Emiglio-in-un-AI-assistant-5/).

In part 4, among the next developments, I had written *"Full duplex AI that responds in real time"*. Here it is.

Until now Emiglio spoke in turns: I speak, it listens, then it responds, and in the meantime it sees nothing. With a **full duplex** model instead the AI sees, listens and speaks **at the same time**, like in a video call: it can interrupt you, comment on what it has in front of it and decide on its own when it's time to open its mouth.

The project is divided into two repos:

- **[emiglio-stream](https://github.com/AlessandroBonomo28/emiglio-stream)**: the part that runs on Emiglio. The Raspberry Pi Zero 2 W takes the video from the webcam and the audio from the ReSpeaker microphone, sends them to the PC, and plays the voice that comes back on the speaker.
- **[MiniCPM-o-Demo-kvpurge](https://github.com/AlessandroBonomo28/MiniCPM-o-Demo-kvpurge)**: the brain. It's a fork of the official demo of **MiniCPM-o 4.5**, an omnimodal model with 9 billion parameters, running locally on an **RTX 5090**. I modified it because the original demo, after a few minutes of conversation, would slow down until it became unusable.

The basic idea is simple: the PC must see Emiglio as a **normal webcam and a normal microphone**. This way MiniCPM-o (or Discord, or whatever else) doesn't even know there's a robot on the other side.

## Component links

- [Raspberry pi zero W2](https://amzn.to/3SuylLD)
- [Respeaker Module](https://amzn.to/4oQRt2t)
- Any USB webcam (I use a Trust at 640x480)
- A PC with an NVIDIA GPU with at least 28 GB of VRAM to run the model

## How streaming works

The Pi Zero 2 W is small but it has one thing I need: the **hardware H.264 encoder**. Compressing video in software on that processor would be impossible, but with the encoder it handles 30 fps at 640x480 without problems and the CPU remains mostly free. The bitrate is at **600 kbit/s**, lower than what the quality would require: I lowered it on purpose to leave breathing room for the 2.4 GHz Wi-Fi, for the reason I explain [later](#the-pi-zeros-wi-fi-and-the-growing-delay).

The flow is this: the webcam spits out raw frames, the encoder compresses them into H.264 and sends them via UDP; in parallel the ReSpeaker microphone is encoded into Opus and sent the same way. Everything arrives at **MediaMTX**, a very lightweight streaming server that runs on the Pi itself and acts as a switchboard: anyone who wants to see Emiglio connects there.

And "anyone" really means anyone. From the browser you open `https://ronaldo.local:8889/emiglio` (yes, the Pi is called ronaldo) and you see Emiglio in WebRTC with very low latency. From OBS or ffmpeg you hook up via RTSP. The PC instead takes it with a Python script that turns it into two virtual devices: a **virtual camera** (thanks to the OBS driver) and a **virtual microphone** (thanks to VB-Cable). At that point in any program I choose "OBS Virtual Camera" and "CABLE Output" and I'm using Emiglio's eyes and ears.

The return works the other way around. MediaMTX has a second channel, `voice`, where the PC publishes the audio it wants to output from Emiglio's speaker. On the Pi it is received by a small player written in Python, `voice-player.py` (launched by `play-voice.sh`): ffmpeg decodes the stream, a queue acts as a buffer and `aplay` plays it on the ReSpeaker. GStreamer is now only used for capturing the webcam and microphone, (and if you have latency problems [we'll talk about it below](#the-pi-zeros-wi-fi-and-the-growing-delay)). The output chain is therefore this:

```
MiniCPM-o (audio) → Python bridge on Windows → MediaMTX (channel "voice") → voice-player → ReSpeaker
```

The PC can publish in two ways: from the browser, with a small page that captures the microphone, or with the bridge that captures Windows' audio output directly. The second is the one I use with MiniCPM-o: the model speaks, Windows hears it, the bridge sends it and the voice comes out of Emiglio.

To install everything on the Pi you just need:

```bash
git clone https://github.com/AlessandroBonomo28/emiglio-stream.git
cd emiglio-stream/pi && sudo ./install.sh
```

The script installs MediaMTX as a service, configures the streams and generates the HTTPS certificates. On the PC you need FFmpeg, OBS (only for the virtual camera driver), VB-Cable and the Python dependencies, all explained in the README. Then you launch the bridge:

```bash
python pc/bridge.py --return "Cuffie (Oculus" --aec
```

`--return` is the Windows audio device on which MiniCPM speaks: the bridge captures it in loopback and sends it to Emiglio's speaker. `--aec` turns on the anti-echo, which if you don't specify it remains off. There are also `--gate duck`, which instead of zeroing the microphone attenuates it by 30 dB, and `--gate-hold 2.5` to lengthen the closing.

One thing I got out of the way early is fragility: **everything reconnects on its own**. If I restart MediaMTX or the network drops, video, audio and return come back up in a few seconds without me touching anything. I tested it by restarting the services and killing the processes by hand, which is the only serious way to verify it.

## MiniCPM-o 4.5 and the problem of the few minutes

**MiniCPM-o 4.5** is an omnimodal open source model: it sees video, listens to audio and responds with voice, and it does so in full duplex mode. Despite the 9 billion parameters (small, by today's standards) on visual benchmarks it's above GPT-4o. To run it you need about 21 GB of VRAM, so it's not Pi stuff: it runs on the PC with the 5090, inside **WSL** with the repo cloned directly (the official demo proposes Docker, but a container in the middle would only have added latency), and it is used from a web page where you select the camera, the microphone and you go.

With the official demo the first test was exciting: Emiglio commenting on the room, responding while I talk to it, noticing when I show it an object. Then, after three or four minutes, it started to slow down. Responses arrived later and later, until the session died.

The reason is the **KV cache**. A model of this type, for everything it sees and hears, keeps in memory a compressed representation (the "keys" and "values" of attention) that it needs to remember the context. In a normal chat the context grows only when you type. In full duplex instead the model swallows audio and video **continuously**, even when no one is talking, and the cache inflates every second. When it reaches the context limit the model freezes. The official demo knew this, and in fact solved it by forcibly closing the session after 5 minutes with video and 10 without. Which for a robot you want to chat with is a bit sad.

### The "kvpurge" fork

The main modification of the fork is a **sliding window on the KV cache**: when the cache exceeds a threshold (4000 tokens by default) it is pruned by cutting the oldest tokens until it drops to 3500. The model loses memory of things that happened a few minutes before, but maintains the recent context, and above all it no longer slows down. In the logs each pruning appears as `✂ KV pruned`, so you can see how often it happens.

Once this was solved, the 5-minute timeout no longer made sense, so I made it **optional**: if you don't set the `REALTIME_MAX_DURATION_S` variable the session goes on until you close it. I also fixed an annoying bug where the speaker chosen on the page was ignored in the first session, and with a virtual device configuration like mine this meant that the voice came out of the PC speakers instead of Emiglio.

## The Pi Zero's Wi-Fi and the growing delay

There was a second slowdown, which had nothing to do with the model. The two symptoms look similar from the outside — "after a few minutes it goes slow" — but they are two different bugs, and for a while I blamed the KV cache even when it wasn't its fault.

The Wi-Fi on the Pi Zero 2 occasionally stalls, even for more than a second. On TCP a lost packet stops everything until it is retransmitted, and when it restarts the data arrives all together, in a burst. The problem is not the burst: it's what you do with it.

The first version played it in its entirety. But the audio goes in real time, so that second of backlog never went away: it stayed in the queue forever. Every hiccup added a piece of permanent delay, and after half an hour Emiglio responded with seconds of lag. The second version did the opposite, throwing away the backlog: delay solved, but the voice came out in pieces. GStreamer's jitter buffer on the Pi also behaved like this, discarding packets that arrived "late" — and that's the reason why the voice playback ended up in a player written by me.

The solution is in the middle, and it's a single rule: **speech is never discarded**. During a stall Emiglio pauses mid-sentence and then resumes exactly from where it was; the accumulated delay is recovered by skipping only the **silences** between one word and another, which in speech are many and nobody notices. The same logic runs in both directions: on the PC for Emiglio's microphone, on the Pi for its voice.

Two traps that made me waste time and that are worth knowing if you redo the project:

- The **Wi-Fi power save** on the Pi Zero 2 W is active by default and produces periodic stalls. `install.sh` now turns it off, permanently.
- The **"zombie player"**: `gst-launch` with the `-e` option, if the connection drops, stays hung waiting for an end of stream that never arrives, and meanwhile keeps the sound card busy. From then on every new player fails with "device busy": the stream arrives, the model responds, and the speaker is silent. It was the most subtle bug of all.

## The AI that talked to itself and the AEC compromise

I discovered the most amusing problem at the first complete test. Emiglio says a sentence, the speaker plays it, the microphone (which is three centimeters from the speaker) hears it, sends it to the model, and the model replies to itself. Emiglio did a one-minute monologue without anyone speaking to it.

The ideal solution would have been a **hardware AEC** (Acoustic Echo Cancellation): a dedicated chip that knows exactly what is coming out of the speaker and subtracts it from the microphone in real time, like phones and conference rooms do. The ReSpeaker 2-Mics doesn't have it, so I tried to do it in software on the Pi, with the echo-cancel module of PipeWire and the WebRTC engine.

The interesting thing is that there was computing power: it ran around **23% of a core** and in the lab it canceled 21-27 dB of echo. It failed for another reason. The Seeed driver puts **59 dB of gain** on the microphone, which with the speaker a few centimeters away saturates: a distorted echo cannot be subtracted, because it no longer resembles what came out of the speaker. And WebRTC's AGC works **after** the canceller, so it raises exactly the residual echo just knocked down (measured: from 21 dB of cancellation to 11). The result in real use was that the voice arrived too low or disturbed and MiniCPM didn't respond at all. In short, I tried it and it didn't hold up to real use.

So I opted for a much rougher anti-echo, and I put it **in the bridge on the PC** (`pc/bridge.py`, flag `--aec`), which is the only point that sees both directions: when it is sending the AI's voice to Emiglio, it zeros Emiglio's microphone towards the model, and keeps it closed for a second and a half after the last word, the time the echo takes to go around the network. Nothing extra runs on the Pi, and that is exactly the advantage: zero calculation on the Pi Zero 2.

The compromise is evident: this in a sense **"kills" the full duplex** and makes it return to turns, because while Emiglio speaks you cannot interrupt it. With a hardware AEC it could have been preserved, and in fact the AEC is deactivatable: without the flag the full duplex is complete, but with the speaker so close to the microphone the monologue is guaranteed.

That said, even in a half-duplex version it remains much better than the cascading pipeline of the previous parts. Not only because there is vision: MiniCPM-o 4.5 is a **state-of-the-art end-to-end model**, and you can feel the difference in every response. I talk about it in the section below.

## Cascaded vs end-to-end models

Up to part 2 Emiglio worked in **cascade**: a speech-to-text model transcribes what I say, the text passes to an LLM that writes the response, and a text-to-speech reads it. Three models in a row, each waiting for the previous one.

It works, but it has structural limits. The first is **latency**: each stage adds its own time, and until the transcription is finished the LLM cannot even start. The second is that between one stage and another **information is lost**: the transcription is only text, so the LLM doesn't know if I spoke whispering, laughing or angry, and the TTS in turn reads a text without knowing the context of the conversation. The result is a correct but flat voice, which always sounds the same whatever is happening. The third is that a cascade **cannot listen while speaking**: it is turn-based by design.

An **end-to-end** model like MiniCPM-o 4.5 is a single network that takes raw audio and video as input and directly produces audio as output. There is no transcription in the middle: the model hears the voice with all its nuances and sees them reflected in the response, with a much more fluid **emotional generation**, with pauses, intonation and rhythm changes that a cascade cannot produce. Under the hood audio and video are broken down into small temporal blocks and inserted into the context of the model as a single sequence, interspersed with the tokens of the response it is generating: this is how it manages to work in **streaming**, a little piece at a time, instead of waiting for all the input to finish. And it's also the reason why the KV cache grew continuously: the price of a model that always listens is that it must always remember.

![Desktop View](/assets/img/posts/emiglio-stream/minicpm-architecture.webp){: width="auto"}
_MiniCPM-o architecture: video and audio continuously enter the encoders, and the model emits `[silent]` tokens until it decides to speak_

The diagram above gives a good idea. At the bottom flow the two incoming streams, broken down second by second; in the middle the model consumes them and produces text tokens that immediately become voice tokens, decoded into audio while the sentence is still in progress. And in the moments when it has nothing to say it emits `[silent]` tokens: it continues to watch and listen, but stays quiet. This is what allows it to decide **on its own** when to intervene, instead of waiting for someone to press a button.

On benchmarks it's a **SOTA** model on both the visual and vocal parts, and it does so with 9 billion parameters, few enough to fit on a single consumer GPU. For a robot that has to stay at home, without cloud, it's exactly the right category.

## The other hassles

Three things worth knowing if you want to replicate the setup.

The **ReSpeaker speaker is only one** and ALSA opens it exclusively: as long as the player holds it for the AI voice, the bluetooth speaker from part 3 cannot use it. Either one or the other, and for now you choose by hand.

The **HTTPS certificates** are mandatory. The browser does not let you access the microphone from an HTTP page, so MediaMTX must run in HTTPS with a self-signed certificate, and the first time you have to accept the security warning. It's not elegant but it works.

And a curiosity that made me lose half an hour: the **speaker of the ReSpeaker 2-Mics is on the right channel**. If you send it a mono audio, it hooks to the left channel and you hear absolutely nothing, with everything else seeming to work perfectly.

## Bonus: the robot voice

Since the PC is publishing the voice towards Emiglio anyway, I added a page with some effects made with Tone.js: pitch, robot, walkie-talkie, distortion, chorus. A model that speaks with a perfect human voice from inside a 90s toy robot is a bit weird. With a pinch of distortion and lowered pitch, Emiglio finally sounds like Emiglio.

## The result

Emiglio now sees what's in front of it, listens and responds in real time, without cloud and without time limits. The code is all in the two repos linked above, with the READMEs explaining every detail I skipped here.

**STAY TUNED** for the next tutorials!

### Future developments

- Remote control and synchronization with Metaquest VR
- Let the AI drive the tracks directly
- **A true full duplex**, which requires hardware with silicon AEC: a ReSpeaker Lite or an XVF3800 with XMOS chip, or a banal USB conference speakerphone, with the speaker driven directly by that board
