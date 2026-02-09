---
categories: [tutorials,3Dengine]
tags: [tutorial, 3Dengine, p5js, builtfromscratch]
---
# Primi Passi con p5.js: Cerchi che Cadono

{% include embed/youtube.html id='zucCXzZ3UCA' %}

## Cos'è p5.js?

p5.js è una libreria JavaScript per creare grafica e animazioni interattive. È perfetta per chi vuole programmare in modo creativo senza troppa complessità. Se vai su [editor.p5js.org](https://editor.p5js.org/) puoi usarla online senza dover scaricare nulla.

## Come Funziona

Ogni progetto p5.js usa due funzioni base:

- **`setup()`** - si esegue una volta all'inizio
- **`draw()`** - si ripete in loop (circa 60 volte al secondo)

## Il Nostro Progetto

In questo tutorial creiamo un'animazione semplice: ogni click del mouse genera un cerchio giallo che cade verso il basso.

### Il Codice Spiegato

```javascript
const raggio = 20;
const fallSpeed = 0.2;
let x = 0, y = 0;
let circles = [];
```

Definiamo il raggio dei cerchi, la velocità di caduta e un array per memorizzare tutti i cerchi creati.

```javascript
function setup() {
  createCanvas(400, 400);
}
```

Creiamo un canvas di 400×400 pixel.

```javascript
function draw() {
  background(20, 190, 20);
  
  x = mouseX;
  y = mouseY;
  
  strokeWeight(5);
  point(x, y);
```

Ogni frame disegniamo uno sfondo verde e un punto che segue il cursore del mouse.

```javascript
  fill(255, 255, 0);
  for(let i = 0; i < circles.length; i++) {
    circle(circles[i][0], circles[i][1], raggio);
    circles[i][1] += deltaTime * fallSpeed;
    
    if(circles[i][1] >= height) {
      circles[i][1] = 0;
    }
  }
}
```

Disegniamo tutti i cerchi gialli, li facciamo cadere e quando escono dal canvas li riportiamo in alto.

```javascript
function mouseClicked() {
  circles.push([x, y]);
}
```

Ogni click crea un nuovo cerchio.

```javascript
function keyPressed() {
  if(key === 'r') {
    circles = [];
  }
}
```

Premendo 'r' cancelliamo tutti i cerchi.

## Provalo Subito

Vai su [editor.p5js.org](https://editor.p5js.org/), copia il codice e premi play

```javascript
const raggio  =20;
const fallSpeed = 0.2;
let x = 0,y = 0;

let circles = [];

function setup() {
  createCanvas(400, 400);
}

function draw() {
  background(20,190,20);
  
  x = mouseX;
  y = mouseY;
  
  strokeWeight(5);
  point(x,y);
  
  strokeWeight(2);
  fill(255,255,0); // r g b 
  for(let i=0; i<circles.length; i++){
    circle(circles[i][0],circles[i][1],raggio);
    circles[i][1] += deltaTime *fallSpeed; 
    if(circles[i][1] >= height){
      circles[i][1] = 0;
    }
  }  
}

function mouseClicked(){
  circles.push([x,y]);
}

function keyPressed(){
  if(key === 'r'){
    circles = []
  }
}
```
