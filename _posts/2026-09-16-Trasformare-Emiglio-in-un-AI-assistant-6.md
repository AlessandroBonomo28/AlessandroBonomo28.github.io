---
title: "Emiglio vede, ascolta e parla in tempo reale (Parte 6)"
description: Emiglio trasmette video e audio dal Raspberry Pi a MiniCPM-o 4.5 in full duplex, e la conversazione non si pianta più dopo pochi minuti.
date: 2026-09-16 10:00:00 +0200
categories: [tutorials, electronics]
tags: [tutorial, blog, electronics, IoT, embedded, AI, raspberry, streaming]
image:
  path: /assets/img/posts/emiglio-stream/robot-demo.png
  alt: Emiglio manda video e audio al PC, MiniCPM-o risponde con la voce
toc: true
---

# Un'IA full duplex per Emiglio: streaming dal Raspberry Pi a MiniCPM-o 4.5

Nelle prime cinque parti Emiglio ha imparato a muoversi col [radiocomando RC](https://alessandrobonomo28.github.io/posts/Trasformare-Emiglio-in-un-AI-assistant/), a [ragionare in locale](https://alessandrobonomo28.github.io/posts/Trasformare-Emiglio-in-un-AI-assistant-2/), a [suonare in bluetooth](https://alessandrobonomo28.github.io/posts/Trasformare-Emiglio-in-un-AI-assistant-3/), a [camminare sui cingoli](https://alessandrobonomo28.github.io/posts/Trasformare-Emiglio-in-un-AI-assistant-4/) e ha ricevuto una [scheda tutta sua](https://alessandrobonomo28.github.io/posts/Trasformare-Emiglio-in-un-AI-assistant-5/).

Nella parte 4, tra i prossimi sviluppi, avevo scritto *"IA full duplex che risponde in real time"*. Eccola.

Finora Emiglio parlava a turni: io parlo, lui ascolta, poi risponde, e nel frattempo non vede niente. Con un modello **full duplex** invece l'IA vede, ascolta e parla **nello stesso momento**, come in una videochiamata: può interromperti, commentare quello che ha davanti e decidere da sola quando è il momento di aprire bocca.

Il progetto è diviso in due repo:

- **[emiglio-stream](https://github.com/AlessandroBonomo28/emiglio-stream)**: la parte che gira su Emiglio. Il Raspberry Pi Zero 2 W prende il video dalla webcam e l'audio dal microfono del ReSpeaker, li manda al PC, e riproduce sull'altoparlante la voce che torna indietro.
- **[MiniCPM-o-Demo-kvpurge](https://github.com/AlessandroBonomo28/MiniCPM-o-Demo-kvpurge)**: il cervello. È un fork della demo ufficiale di **MiniCPM-o 4.5**, un modello omnimodale da 9 miliardi di parametri, che gira in locale su una **RTX 5090**. L'ho modificato perché la demo originale, dopo qualche minuto di conversazione, rallentava fino a diventare inutilizzabile.

L'idea di fondo è semplice: il PC deve vedere Emiglio come una **normale webcam e un normale microfono**. Così MiniCPM-o (o Discord, o qualunque altra cosa) non sa nemmeno che dall'altra parte c'è un robot.

## Link componenti

- [Raspberry pi zero W2](https://amzn.to/3SuylLD)
- [Modulo Respeaker](https://amzn.to/4oQRt2t)
- Una webcam USB qualsiasi (io uso una Trust da 640x480)
- Un PC con GPU NVIDIA con almeno 28 GB di VRAM per far girare il modello

## Come funziona lo streaming

Il Pi Zero 2 W è piccolo ma ha una cosa che mi serve: l'**encoder H.264 hardware**. Comprimere video in software su quel processore sarebbe impossibile, con l'encoder invece i 30 fps a 640x480 li regge senza problemi e la CPU resta quasi libera.

Il flusso è questo: la webcam sputa fotogrammi grezzi, l'encoder li comprime in H.264 e li spedisce via UDP; in parallelo il microfono del ReSpeaker viene codificato in Opus e mandato per la stessa strada. Tutto arriva a **MediaMTX**, un server di streaming leggerissimo che gira sul Pi stesso e fa da centralino: chiunque vuole vedere Emiglio si collega lì.

E "chiunque" vuol dire davvero chiunque. Dal browser si apre `https://ronaldo.local:8889/emiglio` (sì, il Pi si chiama ronaldo) e si vede Emiglio in WebRTC con latenza bassissima. Da OBS o ffmpeg ci si aggancia via RTSP. Il PC invece se lo prende con uno script Python che lo trasforma in due dispositivi virtuali: una **camera virtuale** (grazie al driver di OBS) e un **microfono virtuale** (grazie a VB-Cable). A quel punto in qualsiasi programma scelgo "OBS Virtual Camera" e "CABLE Output" e sto usando gli occhi e le orecchie di Emiglio.

Il ritorno funziona al contrario. MediaMTX ha un secondo canale, `voice`, dove il PC pubblica l'audio che vuole far uscire dall'altoparlante di Emiglio. Sul Pi c'è GStreamer in ascolto che prende quel flusso e lo manda al ReSpeaker. Il PC può pubblicare in due modi: dal browser, con una paginetta che cattura il microfono, oppure con uno script che cattura direttamente l'uscita audio di Windows. Il secondo è quello che uso con MiniCPM-o: il modello parla, Windows lo sente, lo script lo spedisce e la voce esce da Emiglio.

Per installare tutto sul Pi basta:

```bash
git clone https://github.com/AlessandroBonomo28/emiglio-stream.git
cd emiglio-stream/pi && sudo ./install.sh
```

Lo script installa MediaMTX come servizio, configura i flussi e genera i certificati HTTPS. Sul PC servono FFmpeg, OBS (solo per il driver della camera virtuale), VB-Cable e le dipendenze Python, tutto spiegato nel README.

## MiniCPM-o 4.5 e il problema dei pochi minuti

**MiniCPM-o 4.5** è un modello open source omnimodale: vede video, ascolta audio e risponde con voce, e lo fa in modalità full duplex. Nonostante i 9 miliardi di parametri (piccolo, per gli standard di oggi) sui benchmark visivi sta sopra GPT-4o. Per farlo girare servono circa 21 GB di VRAM, quindi non è roba da Pi: gira sul PC con la 5090, dentro **WSL** con il repo clonato direttamente (la demo ufficiale propone Docker, ma un container in mezzo avrebbe solo aggiunto latenza), e si usa da una pagina web dove si seleziona la camera, il microfono e si parte.

Con la demo ufficiale il primo test era entusiasmante: Emiglio che commenta la stanza, che risponde mentre gli parlo, che si accorge quando gli mostro un oggetto. Poi, dopo tre o quattro minuti, cominciava a rallentare. Le risposte arrivavano sempre più tardi, finché la sessione moriva.

Il motivo è la **KV cache**. Un modello di questo tipo, per ogni cosa che vede e sente, si tiene in memoria una rappresentazione compressa (le "chiavi" e i "valori" dell'attenzione) che gli serve per ricordare il contesto. In una chat normale il contesto cresce solo quando scrivi. In full duplex invece il modello ingoia audio e video **in continuo**, anche quando nessuno parla, e la cache si gonfia a ogni secondo. Quando arriva al limite del contesto il modello si impianta. La demo ufficiale lo sapeva, e infatti risolveva chiudendo la sessione a forza dopo 5 minuti con video e 10 senza. Che per un robot con cui vuoi chiacchierare è un po' triste.

### Il fork "kvpurge"

La modifica principale del fork è una **finestra scorrevole sulla KV cache**: quando la cache supera una soglia (4000 token di default) viene potata tagliando i token più vecchi fino a scendere a 3500. Il modello perde la memoria delle cose successe qualche minuto prima, ma mantiene il contesto recente, e soprattutto non rallenta più. Nei log ogni potatura compare come `✂ KV pruned`, così si vede quanto spesso succede.

Una volta risolto questo, il timeout di 5 minuti non aveva più senso, quindi l'ho reso **opzionale**: se non si imposta la variabile `REALTIME_MAX_DURATION_S` la sessione va avanti finché non la chiudi tu. Ho anche sistemato un bug fastidioso per cui l'altoparlante scelto nella pagina veniva ignorato alla prima sessione, e con una configurazione a dispositivi virtuali come la mia questo voleva dire che la voce usciva dalle casse del PC invece che da Emiglio.

## L'IA che parlava da sola e il compromesso AEC

Il problema più divertente l'ho scoperto al primo test completo. Emiglio dice una frase, l'altoparlante la riproduce, il microfono (che sta a tre centimetri dall'altoparlante) la sente, la spedisce al modello, e il modello risponde a sé stesso. Emiglio ha fatto un monologo di un minuto senza che nessuno gli parlasse.

La soluzione ideale sarebbe stata un **AEC hardware** (Acoustic Echo Cancellation): un chip dedicato che conosce esattamente cosa sta uscendo dall'altoparlante e lo sottrae dal microfono in tempo reale, come fanno i telefoni e le sale conferenza. Il ReSpeaker 2-Mics però non ce l'ha, e farlo in software sul Pi Zero, con l'audio che passa da mezza rete, è fuori portata.

Così ho optato per un **AEC lato Pi** molto più grezzo: quando arriva audio dal PC, il microfono di Emiglio viene attenuato, e resta attenuato per un po' dopo che la voce finisce. In pratica Emiglio smette di ascoltare mentre parla. Si attiva con il flag `--aec` e si può regolare quanto tenere chiuso il microfono e quanto attenuarlo.

Il compromesso è evidente: questo in un certo senso **"killa" il full duplex** e lo fa tornare a turni, perché mentre Emiglio parla non puoi interromperlo. Con un AEC hardware si sarebbe potuto preservare, e infatti l'AEC è disattivabile: senza il flag il full duplex è completo, ma con l'altoparlante così vicino al microfono il monologo è garantito.

Detto questo, anche in versione mezza-duplex resta molto meglio della pipeline a cascata delle parti precedenti. Non solo perché c'è la vision: MiniCPM-o 4.5 è un modello **end-to-end allo stato dell'arte**, e la differenza si sente in ogni risposta. Ne parlo nella sezione qui sotto.

## Modelli a cascata vs end-to-end

Fino alla parte 2 Emiglio funzionava a **cascata**: un modello di speech-to-text trascrive quello che dico, il testo passa a un LLM che scrive la risposta, e un text-to-speech la legge. Tre modelli in fila, ognuno che aspetta il precedente.

Funziona, ma ha dei limiti strutturali. Il primo è la **latenza**: ogni stadio aggiunge il suo tempo, e finché la trascrizione non è finita l'LLM non può nemmeno iniziare. Il secondo è che tra uno stadio e l'altro **si perde informazione**: la trascrizione è solo testo, quindi l'LLM non sa se ho parlato sussurrando, ridendo o arrabbiato, e il TTS a sua volta legge un testo senza sapere il contesto della conversazione. Il risultato è una voce corretta ma piatta, che suona sempre uguale qualunque cosa stia succedendo. Il terzo è che una cascata **non può ascoltare mentre parla**: è a turni per costruzione.

Un modello **end-to-end** come MiniCPM-o 4.5 è una rete sola che prende in ingresso audio e video grezzi e produce direttamente audio in uscita. Non c'è nessuna trascrizione nel mezzo: il modello sente la voce con tutte le sue sfumature e le vede riflesse nella risposta, con una **generazione emotiva** molto più fluida, con pause, intonazione e cambi di ritmo che una cascata non può produrre. Sotto il cofano audio e video vengono spezzettati in piccoli blocchi temporali e infilati nel contesto del modello come una sequenza unica, intervallati ai token della risposta che sta generando: è così che riesce a lavorare in **streaming**, un pezzetto alla volta, invece di aspettare che finisca tutto l'input. Ed è anche il motivo per cui la KV cache cresceva senza sosta: il prezzo di un modello che ascolta sempre è che deve ricordare sempre.

![Desktop View](/assets/img/posts/emiglio-stream/minicpm-architecture.webp){: width="auto"}
_L'architettura di MiniCPM-o: video e audio entrano in continuo negli encoder, e il modello emette token `[silent]` finché non decide di parlare_

Lo schema qui sopra rende bene l'idea. In basso scorrono i due flussi in ingresso, spezzettati secondo per secondo; in mezzo il modello li consuma e produce token di testo che diventano subito token vocali, decodificati in audio mentre la frase è ancora in corso. E nei momenti in cui non ha niente da dire emette dei token `[silent]`: continua a guardare e ad ascoltare, ma sta zitto. È questo che gli permette di decidere **da solo** quando intervenire, invece di aspettare che qualcuno prema un pulsante.

Sui benchmark è un modello **SOTA** sia sulla parte visiva che su quella vocale, e lo fa con 9 miliardi di parametri, abbastanza pochi da stare su una singola GPU consumer. Per un robot che deve stare in casa, senza cloud, è esattamente la categoria giusta.

## Le altre grane

Due cose che vale la pena sapere se volete replicare il setup.

L'**altoparlante del ReSpeaker è uno solo** e ALSA lo apre in esclusiva: finché GStreamer lo tiene per la voce dell'IA, la cassa bluetooth della parte 3 non può usarlo. O l'uno o l'altro, e per ora si sceglie a mano.

I **certificati HTTPS** sono obbligatori. Il browser non lascia accedere al microfono da una pagina in HTTP, quindi MediaMTX deve girare in HTTPS con un certificato autofirmato, e la prima volta bisogna accettare l'avviso di sicurezza. Non è elegante ma funziona.

## Bonus: la voce da robot

Già che il PC pubblica la voce verso Emiglio, ho aggiunto una pagina con qualche effetto fatto con Tone.js: pitch, robot, walkie-talkie, distorsione, chorus. Un modello che parla con voce umana perfetta da dentro un robot giocattolo degli anni '90 fa un po' strano. Con un pizzico di distorsione e il pitch abbassato, Emiglio suona finalmente come Emiglio.

## Il risultato

Emiglio ora vede quello che ha davanti, ascolta e risponde in tempo reale, senza cloud e senza limiti di tempo. Il codice è tutto nei due repo linkati sopra, con i README che spiegano ogni dettaglio che qui ho saltato.

**STAY TUNED** per i prossimi tutorial!

### Prossimi sviluppi

- Controllo remoto e sincronizzazione con Metaquest VR
- Far guidare i cingoli direttamente all'IA
