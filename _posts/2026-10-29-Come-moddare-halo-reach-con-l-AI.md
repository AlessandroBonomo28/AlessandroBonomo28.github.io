---
title: "Come moddare Halo Reach con l'AI"
description: "Dalla mia mod per Jorge a cinque passi per usare HaloModSkill con Codex e l'Editing Kit di Halo: Reach."
date: 2026-10-29 10:00:00 +0100
categories: [gaming, modding]
tags: [Halo, Halo Reach, modding, Jorge]
image:
  path: /assets/img/posts/jorge-il-buon-finale-stellare/cover.png
  alt: copertina della mod Jorge the good ending
---

## La mod

Ora che l'AI ci permette di fare qualsiasi cosa ho deciso di provare a moddare il mio gioco preferito: Halo Reach. La mod si chiama **Jorge the good ending** e sono tre missioni che immaginano un finale alternativo per Jorge, uno dei personaggi a cui sono più affezionato del NOBLE Team.

Nel gioco originale Jorge si sacrifica per distruggere la nave Covenant con una bomba slipspace. La mia mod parte da quella storia e inventa un finale alternativo (no spolier). L'unico spoiler è che nella mod è possibile guidare un Seraph.

### Trailer della Mod:

<!-- Video provvisorio ripreso da un vecchio articolo: sostituire con il gameplay della mod. -->
{% include embed/youtube.html id='_3KXZ9kKqHo' %}


## Come moddare

Per lavorare alle missioni ho creato anche [HaloModSkill](https://github.com/AlessandroBonomo28/HaloModSkill), un repository di skill riutilizzabili per l'**Halo: Reach MCC Editing Kit**. Una guida la preparazione dell'ambiente, l'altra segue il lavoro sulle missioni: scenari, script, compilazione, test e pacchettizzazione. Nel repository trovate gli strumenti e le istruzioni che uso per sviluppare la mod; le tre missioni sono il progetto di gioco che ne è nato.

Ecco come iniziare con **Codex** come agente AI, in cinque passi:

1. **Prepara gli strumenti ufficiali.** Su Steam installa *Halo: The Master Chief Collection* e verifica di aver scaricato anche il contenuto di **Halo: Reach**. Nella Libreria attiva il filtro **Strumenti**, cerca [**Halo: Reach Mod Tools – MCC**](https://store.steampowered.com/app/1695793/Halo_Reach_Mod_Tools__MCC/) e premi **Installa**: è l'Editing Kit ufficiale (HREK), con programmi come Sapien per modificare le missioni e `tool_fast` per costruire le mappe. Aprilo almeno una volta da Steam per completare l'eventuale installazione dei componenti necessari, poi usa **Gestisci → Sfoglia i file locali** per trovare la sua cartella; non dare per scontato che sia nella stessa cartella del gioco. Se vuoi creare il pacchetto della mod o pubblicarlo sul Workshop, installa separatamente [**Halo MCC Mod Uploader**](https://store.steampowered.com/app/1695795), chiamato **Excession** nella documentazione ufficiale, e avvia anche questo una volta da Steam.
2. **Aggiungi le skill al progetto.** Scarica [HaloModSkill](https://github.com/AlessandroBonomo28/HaloModSkill) e copia le cartelle complete `skills/halo-reach-setup` e `skills/halo-reach-modding` dentro `<tuo-progetto>/.agents/skills/`. Conserva anche le sottocartelle `references` e `scripts`: le skill le usano durante il lavoro. La [guida per Codex](https://github.com/AlessandroBonomo28/HaloModSkill/blob/main/docs/codex.md) mostra la struttura esatta.
3. **Fai controllare l'ambiente.** Apri una nuova chat Codex nel progetto e chiedi: `Usa $halo-reach-setup per verificare la mia installazione HREK.` Codex controlla dove sono gli strumenti e ti segnala ciò che manca prima di toccare la missione.
4. **Descrivi la modifica.** Indica a Codex lo scenario su cui vuoi lavorare e l'effetto desiderato, poi chiedi: `Usa $halo-reach-modding per modificare questa missione e verificare la build.` La skill distingue le modifiche agli script da quelle ai soli tag e guida la compilazione con Sapien e `tool_fast` quando serve.
5. **Prova e prepara la mod.** Avvia una **nuova missione** in MCC con l'opzione ufficiale che disabilita l'anti-cheat, controlla il risultato nel gioco e correggi eventuali problemi. Quando funziona, usa Excession per creare il pacchetto e ripeti il test sulla versione installata.
