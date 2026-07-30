---
title: "Emiglio bluetooth speaker (Parte 3) "
date: 2026-06-24 00:00:00 +0000
categories: [tutorials, electronics]
tags: [tutorial, blog, electronics, IoT, embedded]
image:
  path: /assets/img/posts/emiglio/embluetooth.jpg
  alt: Emiglio usato come speaker bluetooth da uno smartphone
---

# Trasformare Emiglio in uno speaker bluetooth

{% include embed/youtube.html id='g96JaZwjHMA' %}

Nella [Parte 2](https://alessandrobonomo28.github.io/posts/Trasformare-Emiglio-in-un-AI-assistant-2/) abbiamo provato a rendere intelligente Emiglio configurando un AI assistant in locale.
Oggi vediamo come trasformare Emiglio in uno speaker bluetooth in modo da poter riprodurre audio facilmente collegandoci con il telefono.

![Desktop View](/assets/img/posts/emiglio/embluetooth.jpg){: width="auto"}
_Vedo emiglio come uno speaker bluetooth_

### I componenti necessari
- [Raspberry pi zero W2](https://amzn.to/3SuylLD)
- [Modulo Respeaker](https://amzn.to/4oQRt2t)

Nel tutorial precedente abbiamo configurato il **Raspberry pi zero W2** e installato i **driver per scheda audio Respeaker**, quindi già si collega in automatico alla nostra rete locale e possiamo collegarci ad esso via **ssh** per programmarlo. Se partite da zero allora dovete prima flashare il sistema operativo Pi OS lite sul raspberry con PI imager e poi installare i drivers.

#### Verifica che microfono e altoparlante funzionino

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
# Come trasformare il Raspberry Pi in una cassa Bluetooth

---

## Prima di tutto: come "viaggia" la musica?

Prima di digitare comandi, capiamo il percorso che fa la musica dal telefono ai coni della ReSpeaker. Useremo uno stack leggerissimo per la CPU del Pi Zero:

1. **BlueZ** è lo stack Bluetooth di Linux, aggancia l'antenna del telefono e riceve lo stream audio grezzo nel protocollo *A2DP* (il protocollo che gli auricolari e le casse usano per la musica in stereo).
2. **BlueALSA** fa il **traduttore**: prende i pacchetti da BlueZ e li trasforma in un normale flusso audio PCM (l'audio "non compresso", il formato che il resto del sistema capisce).
3. **ALSA** è il motore audio di basso livello del Kernel. Riceve il PCM, applica al volo il *resampling* matematico a 48kHz (più sotto spiego perché serve) e lo consegna ai driver della scheda audio.
4. Il processo demone **`bluealsa-aplay`** sta in ascolto perenne su BlueALSA e, appena arrivano dei byte, li spinge fisicamente nell'altoparlante.

![Desktop View](/assets/img/posts/emiglio/bluetooth.png){: width="auto"}
_Stack bluetooth_


Per far funzionare questo stack, dobbiamo però eliminare un programma che altrimenti andrebbe in conflitto con BlueALSA (lo vediamo al Passo 1).

---

## Controlli prima del Passo 1

Questi tre controlli ti fanno risparmiare un pomeriggio di imprecazioni al computer:

* **Stai usando Raspberry Pi OS "Bookworm" (Debian 12) o successivo (es. Trixie)?**
  Digita:
  ```bash
  cat /etc/os-release
  ```
  Alla riga `VERSION_CODENAME` devi leggere **bookworm** oppure **trixie**. Se leggi *bullseye* o *buster*, **fermati**: il pacchetto che installeremo al Passo 2 non esiste nei vecchi repository. Aggiorna prima il sistema operativo, altrimenti non andrai da nessuna parte.

* **Il Bluetooth hardware è acceso?**
  Se in passato avevi disabilitato il BT per risparmiare RAM, riaccendilo:
  ```bash
  sudo raspi-config
  ```
  e vai in *System Options*, *Network at Boot / Bluetooth*, poi accendilo.

---

## Passo 1: Togliere il Bluetooth a PipeWire

**Perché:** sulle immagini *Desktop* di Bookworm il server audio predefinito è PipeWire, che gestisce anche il Bluetooth ed entra quindi in conflitto con BlueALSA. Per risolvere basta rimuovere il **plugin Bluetooth di PipeWire** (`libspa-0.2-bluetooth`): PipeWire smette di occuparsi del Bluetooth (che passa a BlueALSA) ma continua a gestire l'audio locale e USB.

Sulle immagini *Lite* (tipiche di un Pi headless) PipeWire di norma **non è installato**: in quel caso questo passo non serve e puoi saltare al Passo 2.

Prima controlla se il plugin è presente:
```bash
sudo apt update --fix-missing
apt list --installed 2>/dev/null | grep -E "libspa-0.2-bluetooth|pipewire"
```

Se compare `libspa-0.2-bluetooth`, rimuovilo:
```bash
sudo apt remove libspa-0.2-bluetooth -y
```

> *Opzionale:* puoi rimuovere anche `pipewire-pulse` se non usi applicazioni che parlano con l'API PulseAudio (`pactl`). Non è necessario per il Bluetooth.

Se PipeWire è installato, riavvialo per applicare la rimozione del plugin:
```bash
systemctl --user restart pipewire wireplumber
```
*(Eventuali avvisi sono normali: stai riavviando un servizio a cui hai appena tolto un componente.)*

---

## Passo 2: Installare BlueALSA e configurare ALSA

**Perché:** installiamo BlueALSA e gli strumenti per gestire la scheda, poi scriviamo una regola che indica ad ALSA dove inviare il suono.

Installa i pacchetti:
```bash
sudo apt install bluez-alsa-utils alsa-utils -y
```

Ora creiamo il file `~/.asoundrc` (la `~` è la tua cartella personale; il punto iniziale lo rende un file di configurazione nascosto).

Nota hardware: la ReSpeaker ha un oscillatore al quarzo fissato a **48.000 Hz**, quindi la scheda lavora solo a quella frequenza. Se il telefono trasmette un MP3 a 44.100 Hz, senza adattamento la scheda va in errore. Il parametro `type plug` ordina ad ALSA di eseguire il *resampling* a 48.000 Hz in tempo reale e in modo trasparente.

Apri il file:
```bash
nano ~/.asoundrc
```
Cancella l'eventuale contenuto e incolla esattamente questo blocco (in `nano`: incolli, poi `Ctrl+O` e `Invio` per salvare, `Ctrl+X` per uscire):

```text
defaults.bluealsa.interface "hci0"
defaults.bluealsa.device "00:00:00:00:00:00"
defaults.bluealsa.profile "a2dp"

pcm.!default {
    type plug
    slave.pcm "plughw:CARD=seeed2micvoicec,DEV=0"
}
```

> **Significato delle righe:**
> - `hci0` è l'antenna Bluetooth del Pi (quasi sempre `hci0`).
> - `00:00:00:00:00:00` non è un indirizzo reale: è un valore speciale che significa *"qualsiasi dispositivo"*. Lascialo così.
> - `a2dp` è il profilo della musica stereo.
> - Il blocco `pcm.!default` imposta come uscita predefinita la scheda ReSpeaker (`seeed2micvoicec`), con `type plug` per l'adattamento automatico della frequenza.

---

## Passo 3: Verificare il profilo `a2dp-sink`

**Perché:** BlueALSA può lavorare come **sorgente** (`a2dp-source`, invia audio a una cassa) o come **ricevitore** (`a2dp-sink`, riceve audio da un telefono). Per una cassa serve il ruolo **ricevitore**. Sulla maggior parte delle installazioni Bookworm il servizio `bluez-alsa` è già configurato per offrire `a2dp-sink`, quindi spesso questo passo è solo una verifica.

Controlla la riga di avvio del demone:
```bash
systemctl cat bluealsa.service | grep -i exec
```

Aggiungi l'override **solo se** nell'`ExecStart` non c'è `-p a2dp-sink` **e** il telefono si connette per poi disconnettersi subito. In tal caso:
```bash
sudo systemctl edit bluealsa.service
```
Incolla esattamente queste righe nello spazio indicato:
```ini
[Service]
ExecStart=
ExecStart=/usr/bin/bluealsa -p a2dp-sink
```

> **La riga vuota `ExecStart=` non è un errore.** La prima riga azzera il comando predefinito, la seconda imposta quello nuovo. Senza la riga vuota, systemd rifiuta la configurazione. È un comportamento specifico degli override di systemd.

Salva ed esci, poi ricarica e riavvia:
```bash
sudo systemctl daemon-reload
sudo systemctl restart bluealsa
```

> *Se in futuro vuoi che il Pi possa anche inviare musica ad altre casse, usa `-p a2dp-source -p a2dp-sink`. Per una cassa è sufficiente `-p a2dp-sink`.*

---

## Passo 4: Alzare i volumi della scheda

**Perché:** il chip WM8960 della ReSpeaker esce dalla fabbrica con i registri del volume a zero. L'audio risulta in riproduzione ma non si sente nulla finché non alzi questi volumi (operazione una tantum).

1. Apri il mixer della scheda:
   ```bash
   alsamixer -D plughw:CARD=seeed2micvoicec,DEV=0
   ```
2. Con le **frecce destra/sinistra** ti sposti tra i canali, con **su/giù** regoli il livello. Porta **Playback**, **Headphone** e **Speaker** all'**85%**.
   > *I controlli di volume in ALSA seguono una curva logaritmica: sotto il 60% circa il suono è quasi inudibile, e l'ultimo tratto contiene la parte utile. Per questo si imposta un valore alto.*
3. Se sotto un canale leggi `[MM]`, è in mute: premi il tasto `M` per sbloccarlo (diventa `[OO]`).
4. Premi `Esc` per uscire, poi salva le impostazioni nella memoria della scheda (così restano dopo un riavvio):
   ```bash
   sudo alsactl store
   ```

---

## Passo 5: Rendere il Bluetooth sempre visibile

**Perché:** di default BlueZ rende il dispositivo non più individuabile dopo 180 secondi. Per una cassa sempre disponibile conviene disattivare questo timeout.

Apri il file di configurazione:
```bash
sudo nano /etc/bluetooth/main.conf
```
Nel blocco `[General]` trova `DiscoverableTimeout` e impostalo a zero (0 = nessuna scadenza):
```text
DiscoverableTimeout = 0
```
Salva ed esci, poi riavvia il servizio:
```bash
sudo systemctl restart bluetooth
```

---

## Passo 6: Accoppiare il telefono e impostare il trust

**Perché:** per un dispositivo personale non serve uno script che confermi il PIN a ogni connessione. È sufficiente accoppiare una volta e marcare il telefono come *trusted* (fidato): da quel momento la riconnessione è automatica.

1. Entra nella console interattiva del Bluetooth:
   ```bash
   sudo bluetoothctl
   ```
2. Imposta l'agente senza interfaccia (`NoInputNoOutput` indica che il Pi non ha schermo né tastiera per inserire un PIN, quindi accetta l'accoppiamento "Just Works"):
   ```text
   power on
   discoverable on
   pairable on
   agent NoInputNoOutput
   default-agent
   ```
3. Sul telefono: vai nelle impostazioni Bluetooth e, se il Pi compare già, scegli "Dimentica questo dispositivo" (per cancellare un eventuale accoppiamento precedente difettoso). Disattiva e riattiva il Bluetooth, poi tocca il nome del tuo Pi (il suo hostname).
4. Sul Raspberry vedrai scorrere le righe della connessione. Copia il MAC Address del telefono che compare.
   > **Formato del MAC:** va scritto con i due punti, es. `AA:BB:CC:11:22:33`, non tutto attaccato. Un formato errato fa fallire il comando senza messaggi chiari.

   Poi, sempre dentro `bluetoothctl`, segna il telefono come fidato (sostituisci con il MAC reale del tuo dispositivo):
   ```text
   trust AA:BB:CC:11:22:33
   exit
   ```

Da ora in poi, quando quel telefono è nelle vicinanze e ha il Bluetooth attivo, la riconnessione avviene in automatico.

---

## Passo 7: Avvio automatico del player BlueALSA

**Perché:** `bluealsa-aplay` è il processo che instrada l'audio Bluetooth verso la scheda. Per averlo sempre attivo lo configuriamo come servizio systemd, che parte all'avvio e si riavvia in caso di errore.

Crea il file del servizio:
```bash
sudo nano /etc/systemd/system/bt-speaker.service
```

Contenuto:
```ini
[Unit]
Description=Player audio Bluetooth (BlueALSA)
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

> **Note:**
> - `After`/`Requires`: il player non parte prima che Bluetooth e BlueALSA siano pronti.
> - `User=pi`: se il tuo utente non si chiama `pi`, cambia questo campo. L'utente indicato deve essere nel gruppo `audio` (vedi i controlli iniziali).
> - `00:00:00:00:00:00`: di nuovo il valore "qualsiasi dispositivo".
> - `Restart=on-failure`: il servizio si riavvia da solo dopo 5 secondi se si interrompe.

Salva ed esci, poi abilita e avvia il servizio:
```bash
sudo systemctl daemon-reload
sudo systemctl enable --now bt-speaker.service
```

Verifica lo stato:
```bash
sudo systemctl status bt-speaker.service
```
Se vedi `active (running)`, il servizio è attivo.

---

## Verifica finale

Riavvia il Pi e, senza toccare altro, sblocca il telefono e avvia una traccia: l'audio dovrebbe uscire dall'altoparlante della ReSpeaker.

Questa configurazione **convive con un server locale come la Soundboard Flask**: avendo tolto a PipeWire solo la gestione del Bluetooth, PipeWire continua a riprodurre i file `.wav` locali sulla stessa scheda, mentre il Bluetooth passa per BlueALSA. I due percorsi non si contendono più il Bluetooth.

> Nota sulla riproduzione simultanea: l'accesso diretto alla scheda (`plughw`) può essere esclusivo, quindi se vuoi che audio locale e Bluetooth suonino **nello stesso istante** conviene verificarlo sul tuo hardware. In molti casi PipeWire rilascia la scheda quando non riproduce nulla, lasciandola libera per il Bluetooth.

Nella prossima parte vedremo l'integrazione del **Wake Word engine** ("Ehi Emiglio") in esecuzione locale sul Pi Zero.

---
## Risultato finale

Ora potete collegarvi al bluetooth di Emiglio e fargli dire tutte le boiate che vi vengono in mente


{% include embed/youtube.html id='dE9uiRTsfVo' %}


## Risoluzione dei problemi

- **Il telefono non vede la cassa, o si connette e si disconnette subito** → verifica il profilo `a2dp-sink` (Passo 3).
- **Si connette ma non si sente nulla** → controlla i volumi con `alsamixer` (Passo 4): canali non a zero e non in `[MM]`.
- **`Permission denied` sulla scheda audio** → l'utente non è nel gruppo `audio` (controlli iniziali) oppure il campo `User=` del Passo 7 è errato.
- **Funziona nella sessione ma non dopo il riavvio** → controlla che il servizio del Passo 7 sia `enabled` e che `bluealsa-aplay` riparta da solo: `systemctl status bt-speaker.service`.
- **Comportamento incoerente dopo l'installazione** → un `sudo reboot` permette a tutti i servizi di avviarsi nell'ordine corretto.
