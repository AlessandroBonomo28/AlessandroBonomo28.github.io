---
categories: [gamedev]
tags: [gamedev,games, builtfromscratch]
---
# Come NON pubblicare un gioco di successo su Steam

![Desktop View](/assets/img/posts/dung/1.png){: }
_Dungeon Island su Steam_

Ho iniziato a programmare [Dungeon Island](https://store.steampowered.com/app/1355450/Dungeon_Island/) all'inizio della quarantena COVID, nel gennaio 2020. Al 28 ottobre 2021, sono passati 471 giorni dal rilascio su Steam e il gioco ha finalmente raggiunto la versione 1.0.0.

**Il problema?** Ho fatto tutto da solo, concentrandomi ossessivamente sulla programmazione e ignorando completamente quello che interessava davvero ai giocatori: l'aspetto visivo e il gameplay.

![Desktop View](/assets/img/posts/dung/2.png){: }
_Dungeon Island su Steam_

## L'ossessione tecnica che nessuno ha apprezzato

Mi sono concentrato su sistemi complessi che interessavano solo a me

### Sistemi di Gioco Implementati

**Generazione procedurale del mondo** usando Perlin noise - un sistema matematico sofisticato per creare terreni naturali, montagne, grotte e corpi d'acqua. Ho passato settimane a perfezionare i parametri del noise e gli algoritmi di posizionamento degli oggetti.

**Sistema di illuminazione dinamica** - uno dei sistemi più difficili da implementare, con effetti di luce realistici che cambiano in tempo reale. Tecnicamente impressionante, ma chi se ne accorge se il gioco sembra fatto in Paint?

**Pathfinding per i mostri** - algoritmi complessi per calcolare il percorso ottimale dei nemici attraverso ostacoli e terreni diversi. Funziona perfettamente. A nessuno importa.

**Sincronizzazione achievement con Steam** - ho studiato le API di Steam, implementato il sistema di tracking degli obiettivi e delle statistiche. Ore di lavoro per una feature che il 90% dei giocatori nemmeno nota.

**Sistema di salvataggio/caricamento** - gestione di enormi quantità di dati: posizioni e stati di tutti gli oggetti, mostri, personaggio giocatore, seed per la generazione procedurale. Un capolavoro di architettura software che nessuno vedrà mai.

### Altri Sistemi Tecnici (che interessavano solo a me)

- **Ciclo giorno-notte** con spawn dei mostri basato sui livelli di luce
- **Sistema di inventario** con gestione complessa dello stato degli oggetti
- **Sistema di lancio oggetti** con calcoli fisici per peso, movimento e danno
- **Crafting e fornace** con database di ricette e combinazioni
- **Coltivazione di palme e canne di bambù** con sistema di crescita ambientale
- **Sistema di animazione del giocatore** con scheletro e interazioni con oggetti, armi e armature
- **Sistema di combattimento** con danni, schivate e colpi critici
- **Gestione salute, sonno e fame** che tiene traccia dello stato del giocatore
- **Minimappa** con rendering efficiente in tempo reale
- **Sistema di respawn** che traccia stato e posizione dopo la morte
- **Gestione audio** per effetti sonori e musica
- **Boss finale** con meccaniche uniche e comportamento AI
- **Dialog box** per interazione con i cartelli
- **Tiling adattivo** con animazioni per mare e grotte

{% include embed/youtube.html id='FZAc6fcgKag' %}

## I problemi strutturali

### 1. Zero Competenza Artistica

Non sono bravo con la grafica. Il gioco sembra un prototipo fatto da un programmatore (perché lo è). Ho ignorato questo problema pensando che "la tecnica parla da sola". Spoiler: non lo fa.

### 2. Marketing da Principiante

Ho fatto un trailer con **CapCut** usando i **font di default**. Sì, avete letto bene. Ho passato mesi a implementare sistemi complessi e poi ho fatto un trailer in mezz'ora con lo stesso impegno di un video delle vacanze.

Niente presskit degno di nota. Niente strategia di lancio. Niente community building. Solo: "Ecco il gioco, compratelo".

### 3. L'Illusione del "Se è buono tecnicamente, si venderà"

Mi sono convinto che l'eccellenza tecnica fosse sufficiente. Che i giocatori avrebbero apprezzato il pathfinding perfetto, la generazione procedurale bilanciata, il sistema di illuminazione dinamica.

La realtà? I giocatori guardano gli screenshot, vedono la grafica amatoriale, e vanno avanti. Non arrivano mai a scoprire quanto è ben programmato il gioco.

### 4. Il colpo di grazia

La cosa che ha condannato a morte il gioco è stato il sistema di movimento e controlli del player troppo complesso e che non ho testato a sufficienza dando per scontato che fosse comodo e che dovesse essere il player a sforzarsi di impararlo.

![Desktop View](/assets/img/posts/dung/4.jpg){: }
_Dungeon Island su Steam_

## Cosa avrei dovuto fare

**Collaborare** - Trovare un artista. Pagare un professionista per il trailer. Assumere qualcuno per il marketing.

**Investire nel marketing quanto nella programmazione** - Se ho passato 6 mesi a programmare, avrei dovuto passare altrettanto tempo (o budget) nel marketing.

**Capire il mio pubblico** - I giocatori non comprano un gioco per il suo pathfinding. Lo comprano perché sembra bello, sembra divertente, e ne hanno sentito parlare.

**Fare un trailer professionale** - Non con CapCut e font di default. Un trailer è la prima impressione, spesso l'unica.

**Focus sull'esperienza utente** - Ho ammesso io stesso che "l'aspetto UX del gioco non ha ricevuto molta attenzione" e i comandi erano difficili e non intuitivi. Questo è stato un errore fatale.

## Le Sfide Tecniche (che nessuno ha visto)

Ironia della sorte, i problemi più difficili che ho risolto sono quelli che i giocatori danno per scontati:

**Generazione procedurale con Perlin noise** - Creare pattern naturali richiede tuning matematico preciso. Ho implementato algoritmi complessi per piazzare oggetti e mostri nel terreno generato. Risultato visivo? "Meh, sembra un gioco mobile".

**Sistema di salvataggio** - Gestire enormi quantità di dati, salvare lo stato completo del mondo, gestire i file system. Funziona perfettamente. Nessuno lo nota finché non va storto.

![Desktop View](/assets/img/posts/dung/3.jpg){: }
_Dungeon Island su Steam_

## Conclusione

Dungeon Island rimane il mio orgoglio come programmatore. Ho implementato sistemi complessi, risolto problemi difficili. Ma come prodotto commerciale? Un disastro.

E' stato un disastro totale ma frutto di un lavoro sincero e non di una fredda AI

