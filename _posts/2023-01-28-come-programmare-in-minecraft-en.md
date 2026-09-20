---
lang: en
hidden: true
lang_ref: minecraft-pong
permalink: /en/posts/come-programmare-in-minecraft/
categories: [gamedev]
tags: [lua, gamedev, builtfromscratch]
image:
  path: /assets/img/posts/luamine/m1.png
  alt: Pong running on an OpenComputers computer inside Minecraft
---
# Pong in Minecraft with OpenComputers mod

{% include embed/youtube.html id='9L8DC1-4IbQ' %}

**Pong** written in **Lua** using the **OpenComputers** mod for Minecraft - working computers with CPU, RAM, graphics card, and monitor, all craftable and programmable.

## The Setup: Real Computers in a Virtual World

![Desktop View](/assets/img/posts/luamine/m2.png){: }
_Computers in minecraft_

### What You Need

**OpenComputers mod v1.7.4.153** - A mod that adds WORKING computers in Minecraft. Not fake computers that pretend to work. REAL computers with:
- CPU
- RAM
- Hard disk
- Graphics card
- Monitor
- Keyboard
- Floppy drive (obviously)

You have to **craft** every component. Want more RAM? Craft more modules. Want higher resolution? Craft a better graphics card.

It's like building a PC, but with pixelated textures and exploding creepers.

## Project Architecture

### autorun.lua - The Selection Menu

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

**Animated intro** that fills the screen with '?' and then clears them one by one. Matrix effect in Lua, inside Minecraft.

![Desktop View](/assets/img/posts/luamine/m1.png){: }
_Computers in minecraft_

### The Menu System

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

Menu with an ASCII cursor that moves. Two columns:
- **Difficulty**: Easy (paddle length 5), Medium (3), Hard (1)
- **Speed**: Slow (0.2s), Medium (0.1s), Fast (0.05s)

Everything drawn character by character on the virtual GPU of Minecraft.

### Input Management with Event Listeners

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
    -- ... other controls
end
event.listen("key_down",keydown)
```

OpenComputers has an **asynchronous event system**. You register a listener and receive callbacks when someone presses a key.

The codes are numbers (203 = left arrow, 208 = down arrow). Why use readable names when you can memorize numeric codes?

![Desktop View](/assets/img/posts/luamine/m3.png){: }
_Computers in minecraft_

### Sound Effects with Virtual PC Speaker

```lua
local beepOptionUp = 195
local beepOptionDown = 220
local beepMenuChange = 391
local beepStart = 440

computer.beep(beepMenuChange,beepTime)
```

The computer in Minecraft has a **PC speaker**. I can generate beeps at different frequencies:
- 195 Hz for option up
- 220 Hz for option down
- 391 Hz for menu change
- 440 Hz (A) for start

It's 80s procedural audio, but in Lua, in Minecraft.

### Cursor Animation with Blinking

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

The selected menu **blinks** every 0.25 seconds. Classic retro UI effect.

Manual timing: I accumulate sleep time and when I reach 0.25s, I toggle visibility.

## pong.lua - The Actual Game

### Dynamic Configuration

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

The arguments from the menu are passed as strings. I convert the choices into game parameters:
- Paddle length: 5/3/1 characters
- Sleep time: 0.2/0.1/0.05 seconds

Hard mode = paddle of 1 character at 0.05s. Good luck.

### Field Rendering

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

All ASCII:
- Borders: `#`
- Players: `|`
- Ball: `O`

The paddle is centered on Y, with automatic control to not go out of bounds.

### Touch Input System

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

**Touch screen** controls! Yes, the monitor in Minecraft supports touch.

**SHIFT + RIGHT CLICK** on the screen:
- Left half = moves Player 1
- Right half = moves Player 2

I calculate the direction of the movement: `(target - current) / abs(target - current)` = ±1

I save **when** and **in which direction** the player moved, to implement...

### Advanced Physics: Spin on the Ball

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

**Spin mechanic**: if you move the paddle in the last 0.2 seconds before the impact, the ball acquires Y velocity in the direction of the movement.

It's like "top spin" in real ping pong. In Lua. In Minecraft.

### Collision Detection

```lua
local halfPlayer = lenPlayer/2 -1

if xBall == distWallPlayer and  
    (yBall >= yCenterPlayer1-halfPlayer-1 and yBall <= yCenterPlayer1+halfPlayer+1) then
    -- HIT!
end
```

**Pixel-perfect** check:
- Does the X of the ball match the X of the paddle?
- Is the Y of the ball inside the paddle's range?

If yes: bounce, invert X velocity, apply spin.

### Scoring System and Reset

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

When the ball goes out of bounds:
- Update score
- Check victory (first to 10)
- Reset ball to center
- Random direction (but never pure vertical/horizontal)
- Beep and sleep for 1 second

![Desktop View](/assets/img/posts/luamine/m4.png){: }
_Computers in minecraft_

### Memory Optimization

```lua
if(math.floor(sleepTimeUntilNow)%10 == 0) then
    computer.freeMemory()
end

if(math.floor(sleepTimeUntilNow)%2 == 0) then
    drawPlayer1(yCenterPlayer1)
    drawPlayer2(yCenterPlayer2)
end
```

**Manual garbage collection** every 10 seconds. OpenComputers has RAM limits, I have to manage memory.

**Periodic redrawing** of the players every 2 seconds to avoid visual glitches.

It's low-level optimization in a Lua script inside a game.

### Procedural Sound Effects

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

Every bounce has a **randomized** beep between two frequencies. Audio variety with 4 bytes of code.

When you score: 221 Hz
When you reset: 247 Hz

It's sound design with virtual PC speaker.

## The Technical Challenges

### 1. Limited Virtual GPU

```lua
local gpu = component.gpu
local w,h = gpu.getResolution()

gpu.set(x, y, character)
gpu.fill(x, y, width, height, " ")
```

The graphics API is minimal:
- `set(x,y,char)` - sets a character
- `fill(x,y,w,h,char)` - fills area
- `getResolution()` - gets screen dimensions

No sprites. No colors (well, 16 colors). Just ASCII characters.

### 2. Non-Deterministic Timing

```lua
os.sleep(sleepTime)
sleepTimeUntilNow = sleepTimeUntilNow + sleepTime
```

`os.sleep()` is not precise. I accumulate the time manually for correct tracking.

Frame drops are inevitable if the Minecraft server lags. The game slows down but remains playable.

### 3. Asynchronous Event System

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

I have to:
- Register listeners before the loop
- Check flags (`quit`, `win`) in the loop
- Remove listeners at the end

Otherwise the listeners keep running even after the program finishes.

### 4. Hardware Components Management

```lua
component = require("component")
local computer = require("computer")

computer.beep(frequency, duration)
computer.freeMemory()
```

I interact with virtual "hardware":
- **component.gpu** - graphics card
- **computer.beep** - PC speaker
- **computer.freeMemory** - frees RAM

It's embedded programming, but for Minecraft components.

### 5. Floppy Disk as Storage

The code goes onto a **virtual floppy** in Minecraft. To modify it:

1. Eject the floppy from the computer
2. Copy the .lua files into the floppy (real Minecraft filesystem)
3. Reinsert the floppy
4. The computer can run the files

It's like programming with real floppies, but in 16x16 textures.

## The Development Workflow

1. **Write Lua** in your favorite editor
2. **Copy** to virtual Minecraft floppy
3. **Insert** the floppy into the in-game computer
4. **Run** the program
5. **Crash** (obviously)
6. **Check** the logs in the virtual terminal
7. **Go back to step 1**

Debug: `print()` on virtual terminal. No debugger, no breakpoints.

## Hidden Features

### Auto-run on Boot

The file is called `autorun.lua` - it's automatically executed when you insert the floppy.

Like MS-DOS autoexec.bat, but in Minecraft.

### Passing Arguments Between Scripts

```lua
-- autorun.lua
local argDiff = tostring(yCursorMenuDiff - (yMenuDiff+2))
local argSpeed = tostring(yCursorMenuSpeed - (yMenuSpeed+2))
os.execute(path .. "pong" .. " " .. argDiff .. " " .. argSpeed)

-- pong.lua
local argDifficulty, argSpeed = ...
```

The menu passes choices as command-line arguments to the main game.

"IPC" (Inter-Process Communication) system via virtual filesystem.

### Memory Management

```lua
computer.freeMemory()
```

Periodic call to avoid **OutOfMemory** in OpenComputers.

Computers have limited RAM (configurable in crafting). I must manage it.

### Complete Cleanup

```lua
event.ignore("key_down", keydown)
event.ignore("touch", touch)
clearScreen()
computer.freeMemory()
```

At the end of the program:
- Remove all listeners
- Clear the screen
- Free memory

Otherwise the next program inherits zombie listeners.

## How to Play

### Setup

1. **Install OpenComputers mod** (v1.7.4.153)
2. **Craft** a complete computer:
   - CPU (Tier 1 is enough)
   - RAM (at least 1x Tier 1)
   - Graphics card (Tier 1 for base resolution)
   - Hard disk or floppy drive
   - Screen
   - Keyboard
   - Case
3. **Clone the repository**
4. **Copy** `pong.lua` and `autorun.lua` into the floppy
5. **Insert** the floppy into the computer
6. **Turn on** the computer
7. **Play**!

### Controls

Menu:
- **Arrows** - navigate options
- **S** - start
- **T** - quit

Game:
- **SHIFT + RIGHT CLICK** on the monitor to move paddles
- Left half screen = Player 1
- Right half screen = Player 2
