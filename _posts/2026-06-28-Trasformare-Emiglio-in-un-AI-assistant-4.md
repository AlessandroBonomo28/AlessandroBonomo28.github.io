---
title: "Emiglio robot con cingoli da carro armato (Parte 4) "
date: 2026-06-24 00:00:00 +0000
categories: [tutorials, electronics]
tags: [tutorial, blog, electronics, IoT, embedded]
---

# Aumentare la stabilità del robot con dei cingoli e una ruota da carrello girevole

## Le vecchie ruote di Emgilio si sono rotte

Dopo settimane di onorato servizio, le ruote del nostro [Emiglio robot del primo tutorial](https://alessandrobonomo28.github.io/posts/Trasformare-Emiglio-in-un-AI-assistant/) hanno smesso di funzionare... il carrello in plastica non ha retto il ripetuto stress dei motori spinti da una tensione di 12V e ad un certo punto ha ceduto. Nella gif qui di seguito si può osservare Emiglio offeso con le ruote smontate.

![Desktop View](/assets/img/posts/emiglio/ruoteandate.gif){: width="auto"}
_Emiglio K.O. con le ruote vecchie rotte_

## Emiglio è meglio (con i cingoli)
Per risolvere definitvamente ho deciso di comprare dei **cazzuttissimi cingoli** da carro armato per Emiglio in modo da renderlo stabile e pronto ad affrontare qualsiasi tipo di terreno. Cercando online il mio occhio è caduto su questi:

![Desktop View](/assets/img/posts/emiglio/cingoli.png){: width="auto"}
_Cingoli su amazon_

### Link componenti

- [CINGOLI su amazon](https://amzn.to/441fVVf)
- [Raspberry pi zero W2](https://amzn.to/3SuylLD)
- [Modulo Respeaker](https://amzn.to/4oQRt2t)
- [Raspberry pi 4 B (meglio del Pi 1 B)](https://amzn.to/3R2ipQi)
- [Batteria 12V ricaricabile](https://amzn.to/3SOVmZL)
- [driver motori TB6612FNG](https://amzn.to/43Z6G8c)

### I motori TS100 e il nuovo cablaggio

![Desktop View](/assets/img/posts/emiglio/tsmotor.png){: width="500px"}
_Motori TS100 del telaio cingolato_

I nuovi motori del telaio cingolato si presentano con un connettore a 6 fili perché includono un encoder (un sensore per misurare i giri della ruota). La modifica è semplice perchè il circuito elettronico di Emiglio rimane lo stesso del [tutorial parte 1](https://alessandrobonomo28.github.io/posts/Trasformare-Emiglio-in-un-AI-assistant/). Devo solo cambiare i cavi che vanno ai motori, possiamo pilotarli come dei normalissimi motori DC. Ecco i passaggi pratici:

- Scollegate i vecchi motori dell'Emiglio dalle uscite del motor driver (i morsetti del Motor A e Motor B).

- Isolate i fili inutili: Sui nuovi motori a 6 fili, ignorate i 4 cavetti dedicati al sensore (fase A, fase B, VCC sensore e GND sensore). Potete tagliarli o bloccarli con del nastro isolante.

- Collegate l'alimentazione: Prendete gli unici due fili rimanenti, ovvero quelli di potenza contrassegnati come M+ e M-. Collegate l'M+ e l'M- del cingolo destro ai morsetti del Motor A sul driver, e ripetete l'operazione per il cingolo sinistro sui morsetti del Motor B.

> Nota: per montare i cingoli nuovi ho dovuto anche stampare un pezzo intermedio che collegasse la scocca di Emiglio al telaio. Eccolo, va montato sotto i piedi di Emiglio:

![Desktop View](/assets/img/posts/emiglio/supportoint.png){: width="400px"}
_Supporto interno per collegare il telaio a Emiglio_

## Risorse utili per il montaggio dei cingoli nuovi
- [manuale montaggio cingoli](https://sposmart.com/#/Robot/FrameChassis/TS_Series/TS100/TStank)
- [link pagina ufficiale](https://sposmart.com/#/)

## Il problema di stabilità
Subito dopo aver montato i cingoli su Emiglio, prendo il radiocomando e inizio a farlo girare su sè stesso per testare il movimento laterale, che funziona correttamente

![Desktop View](/assets/img/posts/emiglio/turn2.gif){: width="auto"}
_Emiglio gira su sè stesso con i cingoli_

Poi provo a muoverlo avanti e indietro ma **impenna e cade a terra**, quindi decido di appenderlo con un cavo al soffitto e cerco di capire come risolvere:

![Desktop View](/assets/img/posts/emiglio/impenna.gif){: width="auto"}
_Emiglio impenna in sicurezza appeso al cavo_

### Il tentativo di risolvere con il codice

Inizialmente, in preda allo sconforto, cerco di risolvere con il codice (e ci riesco ma successivamente scopro che su superfici inclinate si accappotta comunque). Ecco la porzione di codice **anti-accappottamento**:

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
In sostanza ho aggiunto un'accelerazione graduale invece di un'accelerazione istantanea. Ecco il risultato

![Desktop View](/assets/img/posts/emiglio/graduale.gif){: width="450px"}
_Emiglio che accelera gradualmente_

### L'introduzione della ruota da carrello posteriore

Dopo la modifica al codice Emiglio era abbastanza stabile e non cadeva più ma volevo risolvere definitivamente il problema. Mi hanno suggerito di risolvere in molti modi e tra tutte le soluzioni, ragionando, quella che mi ha convinto di più è stato il ruotino da carrello girevole posteriore perchè non è invadente, è semplice e funziona.

Ecco il modello in Fusion360, ha degli ammortizzatori per evitare che gli urti lo danneggino:

![Desktop View](/assets/img/posts/emiglio/ruotino.jpg){: width="350px"}
_Ruotino posteriore e progetto 3D_

## Il risultato finale

![Desktop View](/assets/img/posts/emiglio/mov.gif){: width="400px"}
_Ruotino posteriore e progetto 3D_

Emgilio ora è stabile come un carro armato e pronto ad andare ovunque!
**STAY TUNED** per i prossimi tutorial!

{% include embed/youtube.html id='zc8rK_VowUU' %}

### Prossimi sviluppi
Ho tantissime idee, per il futuro sto lavorando a diverse cose:
- IA full duplex che risponde in real time
- Controllo remoto e sincronizzazione con Metaquest VR
- Pistola ad acqua su Emiglio

