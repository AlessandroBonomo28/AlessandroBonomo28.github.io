---
title: "Emiglio modchip, la scheda custom (Parte 5)"
description: Ho progettato un PCB su misura per Emiglio con ESP32, motor driver, ampli I2S, display e ricevitore IR. Più il condensatore che è esploso e tutto quello che ho imparato.
date: 2026-07-30 10:00:00 +0200
categories: [tutorials, electronics]
tags: [tutorial, blog, electronics, IoT, embedded, pcb, esp32]
image:
  path: /assets/img/posts/emiglio-modchip/pcbway-display.jpg
  alt: Emiglio modchip rev 1.0, la scheda custom per Emiglio
toc: true
---

# Un PCB progettato da zero per Emiglio: ESP32, motori, audio, display e IR su una scheda sola

Nelle prime quattro parti Emiglio ha imparato a muoversi col [radiocomando RC](https://alessandrobonomo28.github.io/posts/Trasformare-Emiglio-in-un-AI-assistant/), a [ragionare in locale](https://alessandrobonomo28.github.io/posts/Trasformare-Emiglio-in-un-AI-assistant-2/), a [suonare in bluetooth](https://alessandrobonomo28.github.io/posts/Trasformare-Emiglio-in-un-AI-assistant-3/) e a [camminare sui motori](https://alessandrobonomo28.github.io/posts/Trasformare-Emiglio-in-un-AI-assistant-4/).

Fino ad ora abbiamo tenuto insieme il circuito con un groviglio di cavi jumper e nastro isolante. Ogni volta che apro Emiglio per aggiungere una cosa devo incrociare le dita.

Con una **PCB su misura** è molto più comodo perchè è tutto integrato (embedded) nella scheda.

La scheda si chiama **Emiglio modchip rev 1.0**, progettata in [EasyEDA](https://oshwlab.com/alessandro2001/project_qgcpzeag). I componenti integrati sono: ESP32, USB-C per la programmazione, driver motori, amplificatore audio, display, ricevitore infrarossi e diversi GPIO liberi per collegarci occhi LED o pulsanti di input (o imput citando [numero 5](https://en.wikipedia.org/wiki/Short_Circuit_(1986_film))).

[![Desktop View](/assets/img/posts/emiglio-modchip/board-3d.png)](https://oshwlab.com/alessandro2001/project_qgcpzeag)
_Il render 3D_

Il progetto è pubblico e liberamente consultabile:

**[Emiglio modchip su EasyEDA](https://oshwlab.com/alessandro2001/project_qgcpzeag)**

In questo articolo racconto anche come mi sono accorto dei suoi limiti (uno con una scintilla e un botto) e cosa si può migliorare. Anche le cose che ho sbagliato, perché sono quelle che insegnano di più.

## Nuovo sponsor: PCBWay

Prima di entrare nel merito, una bella notizia:

[![Desktop View](/assets/img/posts/emiglio-modchip/pcbway-logo.jpg)](https://www.pcbway.com/)
_Abbiamo il primo sponsor del blog! PCBWay!_

Da oggi le schede di Emiglio e dei prossimi progetti del blog vengono prodotte da [PCBWay](https://www.pcbway.com/): produzione di PCB e assemblaggio di qualità, tempi rapidi e prezzi onesti anche per il piccolo prototipo da hobbista come questo.

**Il progetto è condiviso su PCBWay, quindi la scheda di Emiglio la puoi ordinare direttamente, assemblata, senza dover caricare niente:**

### 👉 [Ordina qui l'Emiglio modchip](https://www.pcbway.com/project/shareproject/emiglio_modchip_assembly_b18694d1.html)

Se invece ti serve un PCB tuo, **[puoi ordinare qui](https://www.pcbway.com/)**.

Ringraziare con un semplice link mi sembrava poco, quindi ho fatto la cosa che un elettronico fa quando è contento: **ho messo il logo dentro il firmware**. C'è uno sketch dedicato che lo disegna sul display della scheda, con il wordmark che cade dall'alto e rimbalza e lo swoosh arancione che viene "routato" colonna per colonna come una pista su un PCB.

[![Desktop View](/assets/img/posts/emiglio-modchip/pcbway-display.jpg)](/assets/img/posts/emiglio-modchip/pcbway-display.jpg)
_Il logo PCBWay sul display da 1,8 pollici, con la scheda modchip collegata via USB-C e il pannello retro con la serigrafia di Emiglio_

Lo trovi tra [gli sketch del repo](#gli-sketch).

## Cosa c'è sulla scheda

[![Desktop View](/assets/img/posts/emiglio-modchip/schematic.png)](https://oshwlab.com/alessandro2001/project_qgcpzeag)
_Lo schematico completo della rev 1.0, disegnato in EasyEDA. Clicca per aprire il progetto su OSHWLab_

Fisicamente è una scheda a **2 strati, 38,2 × 56,9 mm**, FR-4 da 1,6 mm, solder mask nera e serigrafia bianca. Sul retro c'è la faccia di Emiglio disegnata in serigrafia, il logo *goodman industries* e un saluto a Johnny 5, perché un PCB senza easter egg è solo un pezzo di vetronite.

Lo schematico è diviso in sei blocchi, uno per ogni cosa che Emiglio deve saper fare.

### Il cervello e la programmazione

Il cuore è un **ESP32-WROOM-32E da 16 MB**. Ci sono anche i pulsanti BOOT e RESET fisici, due pulsanti utente e due LED (alimentazione e debug).

### i motori

Per i motori della c'è un **TB6612FNG**, driver a doppio ponte H: due canali, quattro pin di direzione, due di PWM e uno di standby. Rispetto al vecchio L298N è più efficiente e scalda molto meno, perché usa MOSFET invece di transistor bipolari.

Il pin **STBY** ha un pull-down da 10 kΩ verso massa, e questo è un dettaglio che ho messo di proposito: significa che durante il boot e dopo un reset il driver è **disabilitato**, quindi Emiglio non fa scatti mentre l'ESP32 decide chi è. Il firmware lo abilita solo dopo aver messo i pin di direzione a un livello noto e il PWM a zero.

I motori arrivano su un connettore a 4 vie e la batteria su un XH a 2 vie, separati dalla logica.

### L'audio

Per lo speaker bluetooth della Parte 3 c'è un **MAX98357A**, amplificatore audio con ingresso **I2S**: nessun DAC esterno, nessun jack, l'ESP32 gli manda i campioni digitali e lui pilota direttamente l'altoparlante. Tre fili di segnale: bit clock, word select e dati.

Una cosa da sapere prima di ordinarlo: il MAX98357A è **mono**. Non riproduce lo stereo, o somma i due canali o ne butta uno. La resistenza sul suo pin di configurazione decide quale delle due cose fa, e sulla mia scheda è messa per ottenere la somma **(L+R)/2**, così non perdo metà degli strumenti del mix.

### L'interfaccia: display, IR e occhi

Il display è un **TFT da 1,8 pollici, 128x160, controller ST7735S**, su un connettore KF2510 a 8 vie. Ha un interruttore dedicato per l'alimentazione, così posso spegnerlo quando non mi serve senza toccare il firmware.

Il ricevitore infrarossi è un **TSOP4838**: è quello che riceve il telecomando, il metodo di controllo più comodo per fare le prove in casa. E poi ci sono gli **occhi a LED** di Emiglio, che sono la cosa meno utile e più importante di tutta la scheda.

### Link componenti

- [ESP32-WROOM-32E](https://amzn.to/3SuylLD)
- [driver motori TB6612FNG](https://amzn.to/43Z6G8c)
- [amplificatore I2S MAX98357A](https://amzn.to/4oQRt2t)
- [display TFT ST7735 128x160](https://amzn.to/3R2ipQi)
- [ricevitore IR TSOP4838](https://amzn.to/3SOVmZL)
- [batterie NiMH ricaricabili](https://amzn.to/441fVVf)
- PCB prodotto da **[PCBWay](https://www.pcbway.com/)**


### ATTENZIONE AL PACCO BATTERIE con da 12 V ha fatto il botto

Volevo più coppia sui motori, così ho collegato un pacco da **12 V , 1A** al connettore della batteria. Scintilla, botto secco, odore inconfondibile: **C4** (ironico che il nome sia proprio c4), il condensatore da 47 µF sul rail dei motori, si è messo in corto credo.

Guardando i numeri con calma, i margini veri di questa scheda sono questi:

| Componente | Limite |
|---|---|
| Motori di Emiglio | **6 V nominali** |
| TB6612FNG | 15 V massimo assoluto, ma **1,2 A continui per canale** |
| C4 (47 µF, tensione non dichiarata) | probabilmente 6,3–10 V |

Il driver quindi i 12 V li teneva, ma i motori a 12 V assorbono circa il doppio della corrente prevista e sfondano il limite del TB6612.

**La conclusione è che la scheda va bene così com'è, ma va alimentata come si deve:** un pacco da **9 V**, e per precauzione va usato lo sketch con il **duty cycle limitato al 50%**.

Con quel limite la tensione media ai motori resta intorno ai **4,5 V**, ben dentro i 6 V nominali: l'induttanza degli avvolgimenti livella la corrente, quindi il motore lavora come se fosse alimentato a 4,5 V in continua. Il punto fondamentale è che quel limite **esiste solo nel firmware**: non c'è nessun limitatore hardware, quindi a 9 V un bug nel codice diventa un guasto fisico. Potete arrivare fino a 70% di duty cycle, non so non l'ho testato e non sono un elettronico


### il problema sul reset che devo premere a mano

**Quando stacco l'USB e lo riconnetto lo sketch non parte da solo: devo premere il pulsante RESET.** Collego l'alimentazione, i LED si accendono, e il firmware sta lì fermo finché non gli dico io di partire. In laboratorio con la scheda sul tavolo lo fai senza pensarci. Su un robot che dovrebbe accendersi e camminare, no.

Ho due sospettati e non ho ancora deciso quale sia il colpevole:

- **EN sale prima che il 3,3 V sia stabile.** Il condensatore sul pin EN (1 µF con un pull-up da 10 kΩ) ha una costante di tempo di circa 10 ms: se l'alimentazione sale lentamente, l'ESP32 vede EN alto quando non ha ancora una tensione valida, e non si avvia bene. È il caso di manuale del "serve un reset all'accensione".
- **Il circuito di auto-reset tiene basso EN o IO0.** Le linee RTS e DTR del CH340C pilotano EN e IO0 attraverso Q2. Se uno dei due transistor conduce quando non deve, l'ESP32 resta in reset oppure entra in download mode: acceso, vivo, ma senza eseguire lo sketch.

per ora rimane così e mi arrangio

### Il bluetooth che non arrivava lontano

Mi sono dimenticato di togliere il rame sotto l'antenna dell'esp (possono esserci interferenze)

[![Desktop View](/assets/img/posts/emiglio-modchip/pcb-layout.png)](/assets/img/posts/emiglio-modchip/pcb-layout.png)
_sotto l'antenna ci sono 2 strati di rame, andrebbero tolti_

## Gli sketch

Tutto il firmware sta nel repo del progetto:

**[github.com/AlessandroBonomo28/emiglio-modchip](https://github.com/AlessandroBonomo28/emiglio-modchip)**

Ogni cartella è uno sketch Arduino autonomo, pensato per provare un sottosistema alla volta: si flasha uno per volta, così quando qualcosa non va sai esattamente dove guardare.

- **[`test_motori_minimo`](https://github.com/AlessandroBonomo28/emiglio-modchip/tree/main/test_motori_minimo)**  il firmware principale. Guida il TB6612FNG col telecomando IR (indirizzo `0x07`, e ripremere il tasto della manovra in corso fa HALT) oppure da seriale con `w s d a`, `x` per lo stop, `+`/`-` per il duty, `i` per lo stato. PWM a 20 kHz per stare fuori dalla banda udibile, rampa di spunto a 8 step, `STBY` tenuto basso fino a fine inizializzazione e decodifica del reset reason con diagnostica dedicata al brownout  che è la funzione che mi ha fatto capire più cose di tutte. **È lo sketch da usare di default**, per il motivo raccontato [sopra](#attenzione-al-pacco-batterie-con-da-12-v-ha-fatto-il-botto): tutte le scritture PWM passano da un'unica funzione `pwmWrite()` che applica il clamp `DUTY_MAX`, così il limite non si può bucare per distrazione.

- **[`LcdModchip`](https://github.com/AlessandroBonomo28/emiglio-modchip/tree/main/LcdModchip)**  bring-up e taratura del display. Azzera l'intera **GRAM 132×162** (bordi non visibili compresi) subito dopo `initR()`, poi applica `COLSTART=2 / ROWSTART=1` con una sottoclasse che espone il metodo protetto `setColRowStart()`. Senza quegli offset l'ultima riga e l'ultima colonna del pannello mostrano pixel casuali. Con `LCD_BORDER_TEST 1` disegna la cornice di taratura con i quattro pixel d'angolo colorati.

- **[`emiglio_speaker_max98357a`](https://github.com/AlessandroBonomo28/emiglio-modchip/tree/main/emiglio_speaker_max98357a)**  Emiglio come cassa Bluetooth. Sink A2DP con nome `Emiglio_Speaker`, riconnessione automatica e volume gestibile via AVRCP dal telefono. L'I2S è inizializzato a mano coi pin della scheda invece di lasciare che la libreria usi i default, che cambiano tra le versioni. WiFi spento, task audio inchiodato al core 1 e **CPU obbligatoriamente a 240 MHz**: a 80 o 160 MHz la decodifica non sta dietro al flusso audio e il watchdog riavvia il chip, con un sintomo che sembra tutt'altro.

- **[`telecom1`](https://github.com/AlessandroBonomo28/emiglio-modchip/tree/main/telecom1)**  il telecomando che suona. Quattro note (DO/RE/MI/FA) sui tasti direzionali, con i campioni generati a runtime con `sin()` su I2S a 44,1 kHz e un fade di 10 ms in ingresso e uscita per evitare il pop. Dopo ogni nota riempie i buffer DMA di silenzio e ferma il clock, così il silenzio è davvero silenzio.

- **[`pcbway`](https://github.com/AlessandroBonomo28/emiglio-modchip/tree/main/pcbway)**  la schermata per lo sponsor: il logo PCBWay animato sul display, wordmark che cade e rimbalza, swoosh disegnato colonna per colonna e striscia scorrevole in basso col link.

Nel [README](https://github.com/AlessandroBonomo28/emiglio-modchip#dipendenze) c'è anche la tabella completa delle librerie con le versioni verificate e le impostazioni dell'IDE Arduino da usare, compreso lo schema di partizionamento *Huge APP* che serve allo sketch A2DP per far stare lo stack Bluetooth.


**STAY TUNED** per i prossimi tutorial!
