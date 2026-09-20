---
lang: en
hidden: true
lang_ref: msdos-pong
permalink: /en/posts/Come-programmare-giochi-per-ms-dos/
categories: [gamedev]
tags: [assembly, gamedev, builtfromscratch]
image:
  path: /assets/img/posts/pongasm/p2.png
  alt: Pong written in x86 Assembly running on emulated MS-DOS
---
# Writing Pong in x86 Assembly and playing it on Emulated MS-DOS

In anticipation of Lucca Comics 2023, one fine day, I decided to write **Pong** in **x86 Assembly** for **MS-DOS**. Why? I don't remember, but I remember I did it on the Flixbus while going there.

I uploaded the source in this [repository](https://github.com/AlessandroBonomo28/MS-DOS-Pong)

{% include embed/youtube.html id='Rijj1_BilIo' %}

## The Setup:

To play this Pong you need:
- An MS-DOS emulator
- A virtual floppy (yes, FLOPPY)
- Emu8086 to compile the Assembly code
- WinImage to inject the .com into the virtual floppy

![Desktop View](/assets/img/posts/pongasm/p1.png){: }
_Pong in emulated MSDOS on virtualBox_

## How to Play

1. Clone the git repository
2. MS-DOS Setup (https://github.com/AlessandroBonomo28/MS-DOS-setup)
3. Load floppy1 into the MS-DOS virtual machine
4. Type `A:` to mount the floppy
5. Type `pong` to execute the .com file

## The Assembly Code

### The "How to Play" Screen

```
MACRO HowToPlay
    mov charColor, 0001b
    mov charToWrite, 80  ; P
    mov xChar, 05h
    mov yChar, 05h
    call WriteChar
```

I created a macro that draws the on-screen instructions pixel by pixel. Each character requires setting:
- The color (4 bits!)
- The character to write (ASCII code)
- The X and Y coordinates
- Calling the WriteChar function

To write "P1" and the arrows takes 20+ lines of Assembly. In Python it would have been `print("P1 ↑ W")`. But where's the fun in that?

![Desktop View](/assets/img/posts/pongasm/p3.png){: }
_Pong in emulated MSDOS on virtualBox_

### The Main Game Loop

```
loop:   
    MOV AH,2Ch
    INT 21h         ; get sys time		
    CMP DL,curTime  	
    JE loop
    MOV curTime,DL  ; update time
```

There is no convenient `while(true)`. You have to:
1. Read the system timer with an interrupt
2. Compare it with the previous time
3. Jump back if it hasn't changed
4. Manually update the time variable

Every frame of the game is a dance of registers and conditional jumps.

### Drawing a Rectangle (420 Lines for a Cube)

```
DrawRect PROC 
    mov ax,0           ; init first for counter 
loop1:   
    mov cx,0           ; init nested for counter
loop2:   
    mov bx,cx
    mov dx,xDraw
    add cx,dx
    mov dx,yDraw
    add dx,ax
    
    push ax
    push bx            ; save counters
    push dx
    
    mov ah, 0ch
    mov dl, colorDraw
    mov al, dl
    
    pop dx
    
    int 10h            ; set pixel
```

There is no `drawRect()`. You have to:
1. Create a double nested loop
2. Save the registers on the stack (because they will be overwritten)
3. Manually calculate every pixel coordinate
4. Call the video interrupt for EVERY SINGLE PIXEL
5. Restore the registers from the stack

A 6x100 pixel rectangle = 600 interrupt calls. For every frame. Welcome to 80s optimization.

### Input Management: A Nightmare of Interrupts

```
MOV AH,01h
INT 16H             ; check input interrupt
    
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
    cmp dx,bx       ; if next step goes out of top wall
    jb incyp1
```

To read a key:
1. Interrupt to check if there is input
2. Manual comparison of the ASCII code
3. Series of conditional jumps for each key
4. Manual calculations to prevent the player from going off screen
5. Manual flush of the input buffer

In C++ it would be `if (key == 's') player.y += speed;`. Here it's 30 lines.

### Ball Physics: Analytical Geometry in Assembly

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
cmp bl,1b           ; if xDir = -1
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

This code does ONE thing: it checks if the ball hit player 1.

You have to manually:
- Check if the ball touched the left wall
- Verify if it's in the paddle's zone
- Check if the direction is right
- Do pixel-perfect collision detection
- Invert the direction

All this repeated for:
- Player 1 (left)
- Player 2 (right)
- Top wall
- Bottom wall

It's about 200 lines of Assembly just for the ball's physics.

### The VGA Coordinate System

```
; resolution of int 12h is 640x480 
; range x: [0-639]; range y: [0-479]
    
xMax DW 027Fh      ; = 639
yMax DW 01DFh      ; = 479 
    
xBall DW 01F4h
yBall DW 0190h
```

We work in **VGA video mode 12h**:
- Resolution: 640x480 pixels
- Colors: 16 (yes, SIXTEEN colors)
- Every pixel must be set manually via interrupt
- No double buffering, no VSync, no antialiasing

When you update the screen, you see every pixel light up in sequence. It's beautifully nostalgic and practically horrible.

### State Management: Global Variables Everywhere

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

There is no OOP. There are no structs. There are no classes.

Only global variables. 20+ global variables to keep track of:
- Current ball position
- Old ball position (to erase it)
- X and Y direction
- Dimensions
- Colors (4 bits each)
- Speed
- Player positions
- Timer

Every variable is defined manually in hexadecimal. `DW` = Define Word (16 bits), `DB` = Define Byte (8 bits).

### The Beep: The Audio of the Future (1981)

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

Sound effects? Background music? No.

You have a BEEP. A single beep from the PC speaker. ASCII code 7 (BEL). It's all you have.

When the ball hits something: BEEP.
When someone loses: awkward silence.

It's forced minimalism.

### Optimization: Smart Redraw

```
mov ax,xBall
mov bx,xOffPlayer
add bx,widthPlayer
add bx,ballStep
cmp ax,bx
jb drawp1           ; if ball close to p1 draw p1
    
mov ax,yPlayer1
cmp ax,yPlayer1old
je nodrawp1         ; skip if not moved
```

A crucial optimization: **don't redraw players if they haven't moved** and **don't redraw players if the ball is far away**.

Why? Because drawing a 6x100 pixel rectangle takes 600 interrupt calls. At 30 FPS, that's 18,000 interrupts per second ONLY for one player.

So I check:
1. Is the ball close to the player? (within ballStep pixels)
2. Has the player moved since the last frame?

If both answers are "no", I skip the redraw. Saves 70% of graphical calls.

This is old-school optimization: counting CPU cycles.

## The Technical Challenges

### 1. No Libraries

No SDL, no OpenGL, no DirectX. Just:
- BIOS interrupt for video (`INT 10h`)
- DOS interrupt for input (`INT 16h`)
- DOS interrupt for timer (`INT 21h`)

Every feature requires deep knowledge of BIOS interrupts.

### 2. Manual Memory Management

```
push ax
push bx
; ... use registers
pop bx
pop ax
```

Only 4 general-purpose registers (AX, BX, CX, DX). If you call a function, you have to manually save the registers on the stack and restore them afterwards.

### 3. Hardware-Based Timing

```
MOV AH,2Ch
INT 21h             ; get sys time
CMP DL,curTime
JE loop             ; loop if same hundredth of a second
```

The game loop is tied to the DOS hardware timer. Every tick is about 1/100 of a second.

There's no `deltaTime`. There's no VSync. The game runs at the speed of the PC clock.

## The Development Workflow (Absurd)

1. **Write the code** in emu8086
2. **Compile** into a .COM file
3. **Open WinImage** (software from 1993!)
4. **Inject the .COM into the virtual floppy**
5. **Mount the floppy in MS-DOS**
6. **Type A:** to access the floppy
7. **Run the program**
8. **Crash**
9. **Go back to step 1**

Every iteration requires 9 steps. In Unity you press Play. Here you press 9 different things.

## The Final Result

A working Pong that:
- Runs on MS-DOS (or emulator)
- Supports 2 players (W/S and I/K)
- Has perfect collision detection
- Shows who won
- Emits a satisfying BEEP

All in **less than 700 lines of pure Assembly**.

## How to Try It

1. Clone: `git clone [repository]`
2. MS-DOS Setup: https://github.com/AlessandroBonomo28/MS-DOS-setup
3. Mount floppy1 in the VM
4. `A:`
5. `pong`

Or, if you want to compile your own Assembly:

1. Install emu8086: https://github.com/AlessandroBonomo28/emu8086
2. Install WinImage: https://winimage.com/
3. Write the code in emu8086
4. Compile to .COM
5. Use WinImage to inject the .COM into the floppy
6. Mount the floppy in MS-DOS
7. `A:`
8. `dir` to see the content
9. Run the .COM

## Conclusion

Writing Pong in x86 Assembly is like climbing Everest when you could take a helicopter.

It's technically useless, practically masochistic, but I wanted to do it to understand how hard it was.

![Desktop View](/assets/img/posts/pongasm/p2.png){: }
_Pong in emulated MSDOS on virtualBox_
