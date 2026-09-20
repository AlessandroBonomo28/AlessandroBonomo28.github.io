---
lang: en
hidden: true
lang_ref: booking-desk
permalink: /en/posts/MiniCPM-o-booking-desk/
title: "A full duplex AI receptionist that books appointments (and doesn't invent anything)"
description: I built a voice booking desk with MiniCPM-o 4.5 that listens and speaks at the same time, reads a screen as context and writes to the database only when the client says yes.
date: 2026-09-17 10:00:00 +0200
categories: [tutorials, AI]
tags: [tutorial, blog, AI, LLM, full-duplex, voice, agent]
image:
  path: /assets/img/posts/booking-desk/state-machine.jpg
  alt: The state machine of the booking desk, from IDLE to DONE
toc: true
---

# A full duplex voice agent that takes bookings seriously

In [part 6 of Emiglio](/en/posts/Trasformare-Emiglio-in-un-AI-assistant-6/) I had put MiniCPM-o 4.5 inside a robot to chat. Chatting is fun, but a model that speaks in real time becomes truly interesting when you give it a **job**: answering a studio's phone, taking an appointment, canceling it, and most importantly not making mistakes.

The project is called **MiniCPM-o Booking Desk** and it's a voice receptionist that manages an appointment calendar in full duplex: it listens and speaks at the same time, you can interrupt it, and it never writes a booking to the database until you have explicitly said yes.

**[github.com/AlessandroBonomo28/Minicpm-o-booking-desk](https://github.com/AlessandroBonomo28/Minicpm-o-booking-desk)**

Below is a recorded three-minute session, without cuts, with a booking, a cancellation and some real interruptions:

{% include embed/youtube.html id='Yx80VoA8Vw4' %}

## The problem: a voice model is not a database

A full duplex model like MiniCPM-o is great at sounding like a person, but it has two fatal flaws for a receptionist job. The first is that **it doesn't have reliable memory**: after a few minutes the KV cache needs to be pruned (I talked about it in part 6) and the model quietly forgets that it had promised you Thursday at 10. The second is that **it invents**: if you ask it "is Tuesday free?" it answers yes with the same confidence it would answer no, because it has no calendar in front of it, it's just generating plausible words.

If you let the model decide what to write to the database, sooner or later you end up with a ghost appointment. And a client showing up at the studio for an appointment that doesn't exist is the fastest way to get the project trashed.

The solution I adopted is to cleanly separate the two things: **the model speaks, the code decides.** The model has no tools to write to the database. All writes go through a deterministic state machine in Python that validates every piece of data and acts only upon explicit client consent.

## How it works

![Desktop View](/assets/img/posts/booking-desk/architettura.png){: width="auto"}
_The complete architecture: the client speaks to the model, but only the state machine writes to the database_

The system runs on three parallel channels that all arrive at the model.

The **audio channel** is the most obvious: the client's microphone at 16 kHz enters, the model's voice at 24 kHz exits. There is no synthetic voice injected in the middle: the model decides when to speak, with its natural pause detection mechanism.

The **visual channel** is the trick that holds everything together. MiniCPM-o sees, so instead of trying to make it "remember" the booking state I have it **look at a screen**. It's the *operator screen*: two or three sentences addressed to the client, like *"BOOK: APRIL 8? DOES THAT WORK?"*, updated every second by the state machine. The model reads them as if it were a clerk with a sticky note in front of them. It's not an instruction, it's **perceivable state**: it can't be forgotten because it's always there, in the last frame.

![Desktop View](/assets/img/posts/booking-desk/operator-screen.png){: width="auto"}
_On the left the screen the model sees, on the right the session log with the machine state and the KV cache size_

The **control channel** are two tokens, `force_listen` which already existed in the model and `force_speak` which I added, with which the code can force the turn once a second. They are needed when the state machine has something urgent to say, for example a correction.

To understand where these tokens fit in you have to look at how MiniCPM-o works under the hood:

![Desktop View](/assets/img/posts/booking-desk/minicpm-fullduplex.webp){: width="auto"}
_How the model handles full duplex: video and audio enter in one-second blocks, and for each block the model decides whether to stay quiet (`[silent]`) or generate words_

Video and audio are broken down into one-second blocks and inserted into the context as a single sequence. For each block the model chooses: either it emits a `[silent]` token, and therefore continues only to listen, or it emits text, which a decoder immediately transforms into voice. It is exactly in this choice that `force_speak` and `force_listen` are inserted: the code does not put words into the model's mouth, it just tells it *now it's your turn* or *now stay quiet*.

### The chain for every sentence of the client

When the client finishes speaking, these things happen in order:

1. A **VAD** in the browser hears 300 ms of silence and closes the sentence.
2. **Whisper** (large-v3-turbo) transcribes it. Yes, there is a transcription on the side, but it is not in the voice path: it is only for the code, the model continues to hear the real audio.
3. A small LLM, the **extractor**, reads the transcription and the session state and spits out **only one of four events**: `set` (the client said some data: month, day, hour), `yes`, `no`, `cancel`. Nothing more. It cannot invent actions.
4. The **state machine** in `gateway.py` receives the event, validates the data, checks the calendar, and updates the operator screen. It's the diagram on the cover: it starts from `IDLE`, goes to `COLLECTING` while collecting month, day and hour, then to `CONFIRM`, and only a `yes` from there leads to `DONE` and writes to the database. A "yes" said at any other time writes nothing.
5. **MiniCPM-o** in the meantime has heard the client and is reading the new screen, so it replies with voice.
6. A second LLM, the **judge**, listens to what the model has said and compares it with the real state of the database. If the model has stated a false thing ("perfect, booked!" when it isn't), has gone out of context or is repeating the same sentence, the judge queues a correction prompt, and at the next second the model corrects itself.

Point 6 is the one I like the most. The model is never perfect, so instead of trying to make it perfect I put someone to check it in real time. In the test sessions the judge intervenes three or four times per session, and is right every time.

### The memory that does not expire

As in part 6, the model's KV cache has a **sliding window** (from 4000 to 3500 tokens), but here with a difference: the system prompt is always preserved at the head of the window, so the model never loses its role. And since all the state that matters is on the screen, and not in the model's memory, pruning the cache breaks nothing.

## The numbers

In the reference sessions, from 2 to 4 minutes with multiple operations each, I had **zero incorrect writes on the database**. The latency of the extractor is between 1 and 1.4 seconds, that of the judge under 1.2, and forcing a turn opening costs about 0.6 seconds. There are also regression tests in the repo: 40/40 on date normalization, 76/76 on the state machine replay, 58/58 on the extractor.

Everything runs on **a single GPU**: an RTX 5090 with about 29 GB of VRAM occupied between MiniCPM-o and Whisper. Extractor and judge call a lightweight cloud model (Gemini Flash Lite), but there is a local fallback with Qwen3-1.7B if you want to stay completely offline, with a few points less in accuracy.

## The limits

I would be dishonest not to list them.

- **One-second granularity.** Turn decisions and judge corrections arrive at the next second, so a wrong sentence can start before being corrected.
- **English only**, for now.
- **Needs a headset.** There is no echo cancellation between the model's voice and the microphone, same problem as Emiglio.
- Extractor and judge are cloud by default.
- On very short turns, after a filler "uhm", sometimes the model inserts a few phonemes that are not English.

There is also a more general consideration: newer models like DuplexOmni are bringing real-time reasoning directly into the audio path, and at that point a part of this orchestration could become superfluous. But the structure, with a model that speaks, a screen that is the source of truth and a code that is the only one to write, should transfer exactly to a stronger foundation.

## To try it out

You need a 32 GB GPU, Python 3.10 with PyTorch CUDA, the MiniCPM-o 4.5 weights from Hugging Face and an API key for the extractor. Then:

```bash
bash tools/run_desk.sh
```

and you open `https://localhost:8006/static/hud/hud.html`, where there is the operator screen and the view on the bookings database. The details, configuration and benchmarks are all in the [README](https://github.com/AlessandroBonomo28/Minicpm-o-booking-desk).

The project is built on top of the official OpenBMB demo and the MiniCPM-o 4.5 model, with OpenAI's Whisper for transcription. The logic of the desk, the operator screen, the judge and the modifications to the model are mine.

**STAY TUNED** for the next tutorials!
