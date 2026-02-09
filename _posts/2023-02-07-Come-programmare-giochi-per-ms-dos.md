---
categories: [gamedev]
tags: [assembly, gamedev, builtfromscratch]
---
# Scrivere Pong in Assembly x86 e lo giocarlo su MS-DOS Emulato

In vista del Lucca comics 2023, un bel giorno, ho deciso di e scrivere **Pong** in **Assembly x86** per **MS-DOS**. Perché? non me lo ricordo ma ricordo che lo feci nel Flixbus mentre ci andavo.

Ho caricato il sorgente in questo [repository](https://github.com/AlessandroBonomo28/MS-DOS-Pong)

{% include embed/youtube.html id='Rijj1_BilIo' %}

## Il Setup:

Per giocare a questo Pong servono:
- Un emulatore MS-DOS
- Un floppy virtuale (sì, FLOPPY)
- Emu8086 per compilare il codice Assembly
- WinImage per iniettare il .com nel floppy virtuale

![Desktop View](/assets/img/posts/pongasm/p1.png){: }
_Pong in MSDOS emulato su virtualBox_

## Come Giocare

1. Clona il repository git
2. Setup MS-DOS (https://github.com/AlessandroBonomo28/MS-DOS-setup)
3. Carica il floppy1 nella macchina virtuale MS-DOS
4. Digita `A:` per montare il floppy
5. Digita `pong` per eseguire il file .com

## Il Codice Assembly

### La Schermata "How to Play"

```
MACRO HowToPlay
    mov charColor, 0001b
    mov charToWrite, 80  ; P
    mov xChar, 05h
    mov yChar, 05h
    call WriteChar
```

Ho creato una macro che disegna pixel per pixel le istruzioni su schermo. Ogni carattere richiede impostare:
- Il colore (4 bit!)
- Il carattere da scrivere (codice ASCII)
- Le coordinate X e Y
- Chiamare la funzione WriteChar

Per scrivere "P1" e le frecce servono 20+ righe di Assembly. In Python sarebbe stato `print("P1 ↑ W")`. Ma dov'è il divertimento?

![Desktop View](/assets/img/posts/pongasm/p3.png){: }
_Pong in MSDOS emulato su virtualBox_

### Il Game Loop Principale

```
loop:   
    MOV AH,2Ch
    INT 21h         ; get sys time		
    CMP DL,curTime  	
    JE loop
    MOV curTime,DL  ; update time
```

Non c'è un `while(true)` comodo. Devi:
1. Leggere il timer di sistema con un interrupt
2. Confrontarlo con il tempo precedente
3. Saltare indietro se non è cambiato
4. Aggiornare manualmente la variabile del tempo

Ogni frame di gioco è una danza di registri e salti condizionali.

### Disegnare un Rettangolo (420 Righe per un Cubo)

```
DrawRect PROC 
    mov ax,0           ; init counter primo for 
loop1:   
    mov cx,0           ; init counter for annidato
loop2:   
    mov bx,cx
    mov dx,xDraw
    add cx,dx
    mov dx,yDraw
    add dx,ax
    
    push ax
    push bx            ; salva i counter
    push dx
    
    mov ah, 0ch
    mov dl, colorDraw
    mov al, dl
    
    pop dx
    
    int 10h            ; set pixel
```

Non esiste `drawRect()`. Devi:
1. Creare un doppio loop annidato
2. Salvare i registri nello stack (perché verranno sovrascritti)
3. Calcolare manualmente ogni coordinata pixel
4. Chiamare l'interrupt video per OGNI SINGOLO PIXEL
5. Ripristinare i registri dallo stack

Un rettangolo 6x100 pixel = 600 chiamate a interrupt. Per ogni frame. Benvenuti nell'ottimizzazione anni '80.

### Gestione Input: Un Incubo di Interrupt

```
MOV AH,01h
INT 16H             ; interrupt check input

jne press 
jmp nokeys  

press:  
    cmp al,73h      ; s pressed
    je dwnkey1
    jmp next1 
    
dwnkey1:  
    mov dx,yPlayer1
    add dx,heightPlayer
    mov bx,yMax
    sub bx,playerStep
    cmp dx,bx       ; se al prossimo step esci dal muro top
    jb incyp1
```

Per leggere un tasto:
1. Interrupt per controllare se c'è input
2. Confronto manuale del codice ASCII
3. Serie di salti condizionali per ogni tasto
4. Calcoli manuali per evitare che il giocatore esca dallo schermo
5. Flush manuale del buffer di input

In C++ sarebbe `if (key == 's') player.y += speed;`. Qui sono 30 righe.

### Fisica della Palla: Geometria Analitica in Assembly

```
; check ball_bottom hit left wall
mov ax,xBall
cmp ax,0000h
je p1loss 

mov dx, xOffPlayer
cmp ax,dx           ; if xBall < xOffPlayer
jb cansub1

mov dx,xOffPlayer
add dx,widthPlayer

cmp ax,dx           ; if xBall >= xOffPlayer+widthPlayer
jae cansub1 

mov bl,xDirBall
cmp bl,1b           ; se xDir = -1
jne cansub1  

mov dx,yPlayer1
mov cx,yBall
cmp cx,dx           ; if xBall >= yPlayer1
jae condp1
jmp cansub1

condp1: 
    mov bx,heightPlayer
    add dx,bx
    cmp cx,dx
    jb hitp1
    jmp cansub1
                   
hitp1:  
    call Beep 
    mov xDirBall, 0b  ; set xDirBall = 0 (positive direction)
```

Questo codice fa UNA cosa: controlla se la palla ha colpito il giocatore 1.

Devi manualmente:
- Controllare se la palla ha toccato il muro sinistro
- Verificare se è nella zona della racchetta
- Controllare se la direzione è quella giusta
- Fare collision detection pixel-perfect
- Invertire la direzione

Tutto questo ripetuto per:
- Giocatore 1 (sinistra)
- Giocatore 2 (destra)
- Muro superiore
- Muro inferiore

Sono circa 200 righe di Assembly solo per la fisica della palla.

### Il Sistema di Coordinate VGA

```
; resolution of int 12h is 640x480 
; range x: [0-639]; range y: [0-479]

xMax DW 027Fh      ; = 639
yMax DW 01DFh      ; = 479 

xBall DW 01F4h
yBall DW 0190h
```

Lavoriamo in **modalità video VGA 12h**:
- Risoluzione: 640x480 pixel
- Colori: 16 (sì, SEDICI colori)
- Ogni pixel va impostato manualmente via interrupt
- Niente double buffering, niente VSync, niente antialiasing

Quando aggiorni lo schermo, vedi ogni pixel accendersi in sequenza. È bellissimo in modo nostalgico e orribile in modo pratico.

### Gestione dello Stato: Variabili Globali Everywhere

```
xBall DW 01F4h
yBall DW 0190h 
xBallold DW 01F4h
yBallold DW 0190h

ballWidth DW 0006h
xDirBall DB 1b      ; 1b = -dir, 0b = +dir
yDirBall DB 0b      ; 1b = -dir, 0b = +dir
ballColor DB 1010b

ballStep DW 0006h

yPlayer1 DW 00BEh
yPlayer2 DW 00BEh
yPlayer1old DW 00B0h
yPlayer2old DW 00B0h

widthPlayer DW 0006h
heightPlayer DW 0064h
xOffPlayer DW 00012h
colorPlayer DB 1100b 

playerStep DW 000Ah
curTime DB 00h
```

Non c'è OOP. Non ci sono struct. Non ci sono classi.

Solo variabili globali. 20+ variabili globali per tenere traccia di:
- Posizione attuale della palla
- Posizione vecchia della palla (per cancellarla)
- Direzione X e Y
- Dimensioni
- Colori (4 bit ciascuno)
- Velocità
- Posizioni dei giocatori
- Timer

Ogni variabile è definita manualmente in esadecimale. `DW` = Define Word (16 bit), `DB` = Define Byte (8 bit).

### Il Beep: L'Audio del Futuro (1981)

```
PROC Beep
    xor ax,ax  
    xor dx,dx 
    mov ah,2
    mov dl,7
    int 21h
    RET
ENDP
```

Effetti sonori? Musica di sottofondo? No.

Hai un BEEP. Un singolo beep del PC speaker. Codice ASCII 7 (BEL). È tutto ciò che hai.

Quando la palla colpisce qualcosa: BEEP.
Quando qualcuno perde: silenzio imbarazzante.

È minimalismo forzato.

### Ottimizzazione: Redraw Intelligente

```
mov ax,xBall
mov bx,xOffPlayer
add bx,widthPlayer
add bx,ballStep
cmp ax,bx
jb drawp1           ; se ball vicino a p1 disegna p1

mov ax,yPlayer1
cmp ax,yPlayer1old
je nodrawp1         ; skip se non si è mosso
```

Un'ottimizzazione cruciale: **non ridisegnare i giocatori se non si sono mossi** e **non ridisegnare i giocatori se la palla è lontana**.

Perché? Perché disegnare un rettangolo 6x100 pixel richiede 600 chiamate a interrupt. A 30 FPS, sono 18.000 interrupt al secondo SOLO per un giocatore.

Quindi controllo:
1. La palla è vicina al giocatore? (entro ballStep pixel)
2. Il giocatore si è mosso dall'ultimo frame?

Se entrambe le risposte sono "no", skippo il ridisegno. Risparmio del 70% delle chiamate grafiche.

Questa è ottimizzazione old-school: contare i cicli di CPU.

## Le Sfide Tecniche

### 1. Niente Librerie

Niente SDL, niente OpenGL, niente DirectX. Solo:
- Interrupt BIOS per video (`INT 10h`)
- Interrupt DOS per input (`INT 16h`)
- Interrupt DOS per timer (`INT 21h`)

Ogni feature richiede la conoscenza profonda degli interrupt del BIOS.

### 2. Gestione Manuale della Memoria

```
push ax
push bx
; ... usa i registri
pop bx
pop ax
```

Solo 4 registri general-purpose (AX, BX, CX, DX). Se chiami una funzione, devi salvare manualmente i registri nello stack e ripristinarli dopo.

### 3. Timing Basato su Hardware

```
MOV AH,2Ch
INT 21h             ; get sys time
CMP DL,curTime
JE loop             ; loop se stesso centesimo di secondo
```

Il game loop è legato al timer hardware del DOS. Ogni tick è circa 1/100 di secondo.

Non c'è `deltaTime`. Non c'è VSync. Il gioco gira alla velocità del clock del PC.

## Il Workflow di Sviluppo (Assurdo)

1. **Scrivi il codice** in emu8086
2. **Compila** in un file .COM
3. **Apri WinImage** (software del 1993!)
4. **Inietta il .COM nel floppy virtuale**
5. **Monta il floppy in MS-DOS**
6. **Digita A:** per accedere al floppy
7. **Esegui il programma**
8. **Crash**
9. **Torna allo step 1**

Ogni iterazione richiede 9 step. In Unity premi Play. Qui premi 9 cose diverse.

## Il Risultato Finale

Un Pong funzionante che:
- Gira su MS-DOS (o emulatore)
- Supporta 2 giocatori (W/S e I/K)
- Ha collision detection perfetta
- Mostra chi ha vinto
- Emette un soddisfacente BEEP

Tutto in **meno di 700 righe di Assembly puro**.

## Come Provarlo

1. Clona: `git clone [repository]`
2. Setup MS-DOS: https://github.com/AlessandroBonomo28/MS-DOS-setup
3. Monta il floppy1 nella VM
4. `A:`
5. `pong`

Oppure, se vuoi compilare il tuo Assembly:

1. Installa emu8086: https://github.com/AlessandroBonomo28/emu8086
2. Installa WinImage: https://winimage.com/
3. Scrivi il codice in emu8086
4. Compila in .COM
5. Usa WinImage per iniettare il .COM nel floppy
6. Monta il floppy in MS-DOS
7. `A:`
8. `dir` per vedere il contenuto
9. Esegui il .COM

## Conclusione

Scrivere Pong in Assembly x86 è come scalare l'Everest quando potresti prendere l'elicottero.

È tecnicamente inutile, praticamente masochista, ma volevo farlo per capire quanto fosse difficile.

![Desktop View](/assets/img/posts/pongasm/p2.png){: }
_Pong in MSDOS emulato su virtualBox_

