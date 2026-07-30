---
title: "Emiglio robot con radiocomando RC (Parte 1)"
date: 2026-04-10 00:00:00 +0000
categories: [tutorials, electronics]
tags: [tutorial, blog, electronics, IoT, embedded]
image:
  path: /assets/img/posts/emiglio/radiocommm.png
  alt: Emiglio smontato con il radiocomando RC e l'elettronica di controllo
---

{% include embed/youtube.html id='oEHIA1CD1nk' %}

### La storia
Un giorno, mentre navigo su un sito di usato, mi imbatto in **un vecchio Emiglio a 20€**. Ho sempre desiderato avere un robot "jarvis" personale come in Iron Man e, senza pensarci un attimo, scrivo al venditore per acquistarlo. Il venditore mi dice che non spedisce, quindi mi metto in macchina e parto per recuperare Emiglio.

![Desktop View](/assets/img/posts/emiglio/usato.png){: width="300"}
_L'annuncio online_

Emiglio è messo un pò maluccio:

- manca manca il radiocomando
- manca il vassoio
- lo zaino e la testa sono danneggiati

Ci sarà un pò di lavoro da fare per rimetterlo a posto ma vale tutti i 20€ spesi.

### Il restauro

Ho deciso di iniziare dalla parte estetica. Ho preso le misure dei pezzi danneggiati di Emiglio e li ho ristampati con la stampante 3D. Ho dovuto anche aprire Emiglio, smontare i motori DC e sostituire le rondelle di plastica dato che erano danneggiate e non facevano girare le ruote del carrello.  

[In questa cartella trovate tutti i files STL](https://github.com/AlessandroBonomo28/AlessandroBonomo28.github.io/tree/main/assets/3dfiles/emiglio/) dopo aver stampato i pezzi e averli assemblati, questo è il risultato finale:

![Desktop View](/assets/img/posts/emiglio/img2.png){: width="500"}
_Emiglio come nuovo_

![Desktop View](/assets/img/posts/emiglio/img1.png){: width="500"}
_Vista CAD_

Ora che Emiglio ha un aspetto migliore e la meccanica degli ingranaggi delle ruote funziona e permette corretamente ai motori DC di far girare le ruote possiamo passare alla fase di radiocomando.
### Il radiocomando per aerei

![Desktop View](/assets/img/posts/emiglio/radiocommm.png){: width="500"}
_Radiocomando e circuito saldato su PCB_

Dato che non sono in possesso del radiocomando originale mi sono arrangiato con un radiocomando 2.4Ghz per aerei. Questo tipo di radiocomando trasmette i comandi tramite segnali PWM (Pulse Width Modulation): ogni canale invia un impulso con durata variabile tra circa 1000µs e 2000µs, dove il valore centrale (~1500µs) corrisponde al punto neutro dello stick.
Il ricevitore radio viene collegato direttamente ai pin GPIO del **Raspberry Pi 1 B**. Per leggere questi segnali con precisione ho usato la libreria pigpio, che permette di misurare la durata degli impulsi in modo hardware-accurato senza dipendere dai tempi del sistema operativo. Il [driver TB6612FNG](https://amzn.to/43Z6G8c) si occupa di pilotare i 4 motori DC ricevendo i segnali di direzione e PWM dai GPIO del Raspberry tramite lo script `tank.py`, che implementa una logica di movimento a carri armati: le ruote dei due lati vengono gestite indipendentemente per permettere le svolte.

![Desktop View](/assets/img/posts/emiglio/schema.jpg){: width="auto"}
_Schema dei collegamenti_

Tutto [il codice è opensource su github qui](https://github.com/AlessandroBonomo28/emiglio-controller)

### Lista componenti:
- [Raspberry pi 4 B (meglio del Pi 1 B)](https://amzn.to/3R2ipQi)
- [Batteria 12V ricaricabile](https://amzn.to/3SOVmZL)
- [driver motori TB6612FNG](https://amzn.to/43Z6G8c)

## Risultato finale

{% include embed/youtube.html id='I8c7Y01DfZw' %}

I motori ora sono alimentati da una batteria ricaricabile a 12V e il Raspberry da una powerbank, entrambi comodamente alloggiati nello zaino di Emiglio.
Ho fatto anche dei fori con il Dremel sul retro di Emiglio per far uscire un Power LED, l'antenna radio, i cavi della batteria, l'interruttore della powerbank e il pulsante di spegnimento logico del Raspberry — necessario perché spegnere brutalmente l'alimentazione corrompe la SD card. I servizi `tank.py` e `btn.py` si avviano automaticamente al boot tramite systemd, così Emiglio è operativo non appena acceso, senza bisogno di un monitor o di connettersi in SSH.

### Parte 2

Nella parte 2 vediamo come usare un raspberry pi W2 e un modulo respeaker per interagire vocalmente con Emiglio. [Clicca qui per leggere la parte 2](https://alessandrobonomo28.github.io/posts/Trasformare-Emiglio-in-un-AI-assistant-2/)
