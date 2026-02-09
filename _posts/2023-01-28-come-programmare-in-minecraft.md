---
categories: [gamedev]
tags: [lua, gamedev, builtfromscratch]
---
# Pong in Minecraft con OpenComputers mod

{% include embed/youtube.html id='9L8DC1-4IbQ' %}

**Pong** scritto in **Lua** usando il mod **OpenComputers** per Minecraft - computer funzionanti con CPU, RAM, scheda grafica e monitor, tutto craftabile e programmabile.

## Il Setup: Computer Veri in un Mondo Virtuale

![Desktop View](/assets/img/posts/luamine/m2.png){: }
_Computer in minecraft_

### Cosa Serve

**OpenComputers mod v1.7.4.153** - Un mod che aggiunge computer FUNZIONANTI in Minecraft. Non computer finti che fanno finta di funzionare. Computer VERI con:
- CPU
- RAM
- Hard disk
- Scheda grafica
- Monitor
- Tastiera
- Lettore di floppy (ovviamente)

Devi **craftare** ogni componente. Vuoi più RAM? Crafta più moduli. Vuoi risoluzione maggiore? Crafta una scheda grafica migliore.

È come costruire un PC, ma con texture pixelate e creeper che esplodono.

## L'Architettura del Progetto

### autorun.lua - Il Menu di Selezione

```lua
local charIntro = '?'
local title = "PONG"
local madeby = "alex"

for i=1,w do
    for j=1,h do
        gpu.set(i,j,charIntro)
    end
end

for j=1,h do
    for i=1,w do
        gpu.set(i,j," ")
    end
end
```

**Intro animata** che riempie lo schermo di '?' e poi li cancella uno per uno. Effetto matrix in Lua, dentro Minecraft.

![Desktop View](/assets/img/posts/luamine/m1.png){: }
_Computer in minecraft_

### Il Sistema di Menu

```lua
local charCursor = '>'
local yMenuDiff = 4
local yCursorMenuDiff = yMenuDiff+2

local msgMenuDiff = "Choose difficulty"
gpu.set(xMenuDiff,yMenuDiff,msg)

msg = "Easy"
gpu.set(xMenuDiff,yMenuDiff+2,msg)
msg = "Medium"
gpu.set(xMenuDiff,yMenuDiff+3,msg)
msg = "Hard"
gpu.set(xMenuDiff,yMenuDiff+4,msg)

gpu.set(xMenuDiff-1,yCursorMenuDiff,charCursor)
```

Menu con cursore ASCII che si muove. Due colonne:
- **Difficoltà**: Easy (racchetta lunga 5), Medium (3), Hard (1)
- **Velocità**: Slow (0.2s), Medium (0.1s), Fast (0.05s)

Tutto disegnato carattere per carattere sulla GPU virtuale di Minecraft.

### Gestione Input con Event Listener

```lua
function keydown(_,_,_,ch)
    if ch == 20 then -- T
        quit = true
    end
    if ch == 31 then -- S
        startPressed = true
    end
    if ch == 203 then -- left key
        computer.beep(beepMenuChange,beepTime)
        if menuSelected == 0 then
            menuSelected = 1
        else
            menuSelected = 0
        end
    end
    -- ... altri controlli
end
event.listen("key_down",keydown)
```

OpenComputers ha un sistema di **eventi asincroni**. Registri un listener e ricevi callback quando qualcuno preme un tasto.

I codici sono numeri (203 = freccia sinistra, 208 = freccia giù). Perché usare nomi leggibili quando puoi memorizzare codici numerici?

![Desktop View](/assets/img/posts/luamine/m3.png){: }
_Computer in minecraft_

### Effetti Sonori con PC Speaker Virtuale

```lua
local beepOptionUp = 195
local beepOptionDown = 220
local beepMenuChange = 391
local beepStart = 440

computer.beep(beepMenuChange,beepTime)
```

Il computer in Minecraft ha un **PC speaker**. Posso generare beep a frequenze diverse:
- 195 Hz per opzione su
- 220 Hz per opzione giù
- 391 Hz per cambio menu
- 440 Hz (La) per start

È audio procedurale degli anni '80, ma in Lua, in Minecraft.

### Animazione del Cursore con Blinking

```lua
local secTraBlink = 0.25
local diffVisible = true
local lastTimeVisibleDiff = 0

function blinkMsgMenuDiff()
    writeMsgMenuSpeed()
    if sleepTimeUntilNow - lastTimeVisibleDiff >= secTraBlink then
        if diffVisible then
            clearMsgMenuDiff()
            diffVisible = false
        else
            writeMsgMenuDiff()
            diffVisible = true
        end
        lastTimeVisibleDiff = sleepTimeUntilNow
    end 
end
```

Il menu selezionato **lampeggia** ogni 0.25 secondi. Classico effetto UI retrò.

Timing manuale: accumulo il tempo di sleep e quando raggiungo 0.25s, cambio visibilità.

## pong.lua - Il Gioco Vero

### Configurazione Dinamica

```lua
function getLenPlayer()
    if argDifficulty == '0' then -- easy
        return 5
    end
    if argDifficulty == '1' then -- medium
        return 3
    end
    if argDifficulty == '2' then -- hard
        return 1
    end
    return 5
end

function getSpeedGame()
    if argSpeed == '0' then -- slow
        return 0.2
    end
    if argSpeed == '1' then -- medium
        return 0.1
    end
    if argSpeed == '2' then -- fast
        return 0.05
    end
    return 0.1
end
```

Gli argomenti dal menu vengono passati come stringhe. Converto le scelte in parametri di gioco:
- Lunghezza racchetta: 5/3/1 caratteri
- Sleep time: 0.2/0.1/0.05 secondi

Hard mode = racchetta di 1 carattere a 0.05s. Buona fortuna.

### Rendering del Campo

```lua
function drawBorder()
    for i= 1,w do
        gpu.set(i,1,borderChar)
        gpu.set(i,h,borderChar)
    end
    for i= 1,h do
        gpu.set(1,i,borderChar)
        gpu.set(w,i,borderChar)
    end
end

function drawPlayer(x,yCenter)
    local halfPlayer = lenPlayer/2 -1
    yCenter = math.max(yCenter,border + halfPlayer+1)
    yCenter = math.min(yCenter,h-border - halfPlayer-1)
    for i= yCenter - halfPlayer,yCenter + halfPlayer+1 do
        gpu.set(x,i,playerChar)
    end
end
```

Tutto ASCII:
- Bordi: `#`
- Giocatori: `|`
- Palla: `O`

La racchetta è centrata sulla Y, con controllo automatico per non uscire dai bordi.

### Sistema di Input Touch

```lua
function touch(_,_,x,y)
    local xIn = x
    local yIn = y
    if xIn < w/2 and yIn - yCenterPlayer1 ~= 0 then
        clearColumnPlayer1()
        local movDir = (yIn - yCenterPlayer1) / math.abs(yIn - yCenterPlayer1)
        yCenterPlayer1 = yCenterPlayer1 + movDir
        drawPlayer1(yCenterPlayer1)
        lastTimeMovPlayer1 = sleepTimeUntilNow
        lastMovPlayer1 = movDir
    elseif yIn - yCenterPlayer2 ~= 0 then
        clearColumnPlayer2()
        local movDir = (yIn - yCenterPlayer2) / math.abs(yIn - yCenterPlayer2)
        yCenterPlayer2 = yCenterPlayer2 + movDir
        drawPlayer2(yCenterPlayer2)
        lastTimeMovPlayer2 = sleepTimeUntilNow
        lastMovPlayer2 = movDir
    end
end
event.listen("touch",touch)
```

Controlli **touch screen**! Sì, il monitor in Minecraft supporta touch.

**SHIFT + RIGHT CLICK** sullo schermo:
- Metà sinistra = muove Player 1
- Metà destra = muove Player 2

Calcolo la direzione del movimento: `(target - current) / abs(target - current)` = ±1

Salvo **quando** e **in che direzione** si è mosso il giocatore, per implementare...

### Fisica Avanzata: Spin sulla Palla

```lua
-- collision player1
if xBall == distWallPlayer and  
    (yBall >= yCenterPlayer1-halfPlayer-1 and yBall <= yCenterPlayer1+halfPlayer+1) then
    bounceBeep()
    xBall = xBall+1
    xSpeed = xSpeed * -1
    
    -- SPIN MECHANISM
    if (sleepTimeUntilNow - lastTimeMovPlayer1) <= 0.2 and lastMovPlayer1 ~= 0 then
        ySpeed = lastMovPlayer1
    end
    
    drawPlayer1(yCenterPlayer1)
end
```

**Meccanica dello spin**: se muovi la racchetta negli ultimi 0.2 secondi prima dell'impatto, la palla acquisisce velocità Y nella direzione del movimento.

È come il "top spin" nel ping pong reale. In Lua. In Minecraft.

### Collision Detection

```lua
local halfPlayer = lenPlayer/2 -1

if xBall == distWallPlayer and  
    (yBall >= yCenterPlayer1-halfPlayer-1 and yBall <= yCenterPlayer1+halfPlayer+1) then
    -- HIT!
end
```

Controllo **pixel-perfect**:
- X della palla coincide con X della racchetta?
- Y della palla è dentro il range della racchetta?

Se sì: bounce, inverti velocità X, applica spin.

### Sistema di Punteggio e Reset

```lua
function resetBall()
    xBall = w/2
    yBall = h/2
    math.randomseed(os.time())
    xSpeed = math.random(-1,1)
    ySpeed = math.random(-1,1)
    if xSpeed*ySpeed == 0 then
        xSpeed = 1
        ySpeed = 1
    end
    drawScores()
    gpu.set(lastXBall,lastYBall," ")
    gpu.set(xBall,yBall,"O")
    os.sleep(resetBallSleepTime)
    resetBallBeep()
end

--right wall
if xBall >= w-border then 
    bounceBeep()
    player1Score = player1Score + 1
    if player1Score >= winScore then
        win = true
    end 
    resetBall()
end
```

Quando la palla esce dal campo:
- Aggiorna punteggio
- Controlla vittoria (primo a 10)
- Reset palla al centro
- Direzione random (ma mai verticale/orizzontale pura)
- Beep e sleep di 1 secondo


![Desktop View](/assets/img/posts/luamine/m4.png){: }
_Computer in minecraft_

### Ottimizzazione Memoria

```lua
if(math.floor(sleepTimeUntilNow)%10 == 0) then
    computer.freeMemory()
end

if(math.floor(sleepTimeUntilNow)%2 == 0) then
    drawPlayer1(yCenterPlayer1)
    drawPlayer2(yCenterPlayer2)
end
```

**Garbage collection manuale** ogni 10 secondi. OpenComputers ha limiti di RAM, devo gestire la memoria.

**Ridisegno periodico** dei giocatori ogni 2 secondi per evitare glitch visivi.

È ottimizzazione low-level in uno script Lua in un gioco.

### Effetti Sonori Procedurali

```lua
local beepBounce1 = 97*2 + 1   -- 195 Hz
local beepBounce2 = 48*2 + 1   -- 97 Hz
local beepScore = 110*2 + 1    -- 221 Hz
local beepResetBall = 123*2 + 1 -- 247 Hz

function bounceBeep()
    local x = math.random(0,1)
    if x == 0 then
        computer.beep(beepBounce1,beepTime)
    else
        computer.beep(beepBounce2,beepTime)
    end
end
```

Ogni bounce ha un beep **randomizzato** tra due frequenze. Varietà audio con 4 byte di codice.

Quando segni: 221 Hz
Quando resetti: 247 Hz

È sound design con PC speaker virtuale.

## Le Sfide Tecniche

### 1. GPU Virtuale Limitata

```lua
local gpu = component.gpu
local w,h = gpu.getResolution()

gpu.set(x, y, character)
gpu.fill(x, y, width, height, " ")
```

L'API grafica è minimale:
- `set(x,y,char)` - imposta un carattere
- `fill(x,y,w,h,char)` - riempie area
- `getResolution()` - ottieni dimensioni schermo

Niente sprite. Niente colori (beh, 16 colori). Solo caratteri ASCII.

### 2. Timing Non Deterministico

```lua
os.sleep(sleepTime)
sleepTimeUntilNow = sleepTimeUntilNow + sleepTime
```

`os.sleep()` non è preciso. Accumulo il tempo manualmente per tracking corretto.

I frame drop sono inevitabili se il server Minecraft lagga. Il gioco rallenta ma resta giocabile.

### 3. Event System Asincrono

```lua
event.listen("touch", touch)
event.listen("key_down", keydown)

-- game loop
while true do
    if quit or win then break end
    -- ... game logic
end

event.ignore("key_down", keydown)
event.ignore("touch", touch)
```

Devo:
- Registrare listener prima del loop
- Controllare flag (`quit`, `win`) nel loop
- Rimuovere listener alla fine

Altrimenti i listener continuano a girare anche dopo che il programma finisce.

### 4. Gestione Componenti Hardware

```lua
component = require("component")
local computer = require("computer")

computer.beep(frequency, duration)
computer.freeMemory()
```

Interagisco con "hardware" virtuale:
- **component.gpu** - scheda grafica
- **computer.beep** - PC speaker
- **computer.freeMemory** - libera RAM

È programmazione embedded, ma per componenti Minecraft.

### 5. Floppy Disk Come Storage

Il codice va su un **floppy virtuale** in Minecraft. Per modificarlo:

1. Estrai il floppy dal computer
2. Copia i file .lua nel floppy (filesystem reale di Minecraft)
3. Reinserisci il floppy
4. Il computer può eseguire i file

È come programmare con floppy veri, ma in texture 16x16.

## Il Workflow di Sviluppo

1. **Scrivi Lua** nel tuo editor preferito
2. **Copia** su floppy virtuale Minecraft
3. **Inserisci** il floppy nel computer in-game
4. **Esegui** il programma
5. **Crash** (ovviamente)
6. **Controlla** i log nel terminale virtuale
7. **Torna allo step 1**

Debug: `print()` su terminale virtuale. Niente debugger, niente breakpoint.

## Le Feature Nascoste

### Auto-run al Boot

Il file si chiama `autorun.lua` - viene eseguito automaticamente quando inserisci il floppy.

Come l'autoexec.bat di MS-DOS, ma in Minecraft.

### Passaggio Argomenti tra Script

```lua
-- autorun.lua
local argDiff = tostring(yCursorMenuDiff - (yMenuDiff+2))
local argSpeed = tostring(yCursorMenuSpeed - (yMenuSpeed+2))
os.execute(path .. "pong" .. " " .. argDiff .. " " .. argSpeed)

-- pong.lua
local argDifficulty, argSpeed = ...
```

Il menu passa le scelte come argomenti command-line al gioco principale.

Sistema di "IPC" (Inter-Process Communication) via filesystem virtuale.

### Memory Management

```lua
computer.freeMemory()
```

Chiamata periodica per evitare **OutOfMemory** in OpenComputers.

I computer hanno RAM limitata (configurabile nel craft). Devo gestirla.

### Cleanup Completo

```lua
event.ignore("key_down", keydown)
event.ignore("touch", touch)
clearScreen()
computer.freeMemory()
```

Alla fine del programma:
- Rimuovi tutti i listener
- Pulisci lo schermo
- Libera memoria

Altrimenti il prossimo programma eredita listener zombie.

## Come Giocare

### Setup

1. **Installa OpenComputers mod** (v1.7.4.153)
2. **Crafta** un computer completo:
   - CPU (Tier 1 basta)
   - RAM (almeno 1x Tier 1)
   - Scheda grafica (Tier 1 per risoluzione base)
   - Hard disk o floppy drive
   - Schermo
   - Tastiera
   - Case
3. **Clona il repository**
4. **Copia** `pong.lua` e `autorun.lua` nel floppy
5. **Inserisci** il floppy nel computer
6. **Accendi** il computer
7. **Gioca**!

### Controlli

Menu:
- **Frecce** - naviga opzioni
- **S** - start
- **T** - quit

Gioco:
- **SHIFT + RIGHT CLICK** sul monitor per muovere le racchette
- Metà sinistra schermo = Player 1
- Metà destra schermo = Player 2
