---
categories: [tutorials,3Dengine]
tags: [tutorial, 3Dengine, p5js, builtfromscratch]
---
# Proiezioni 3D in p5.js

{% include embed/youtube.html id='oKl1viZchsk' %}

## Cosa Facciamo

In questo tutorial creiamo la proiezione 2D di un cubo 3D. Prendiamo 8 punti nello spazio tridimensionale (i vertici di un cubo) e li proiettiamo sullo schermo.

## Il Codice Spiegato

### I Vertici del Cubo

```javascript
let points = [
  [-1, -1, -1], // P1
  [1, -1, -1],  // P2
  [1, 1, -1],   // P3
  [-1, 1, -1],  // P4
  [-1, -1, 1],  // P5
  [1, -1, 1],   // P6
  [1, 1, 1],    // P7
  [-1, 1, 1]    // P8
];
```

Definiamo gli 8 vertici di un cubo centrato nell'origine. Ogni punto ha coordinate [x, y, z].

### Setup

```javascript
function setup() {
  createCanvas(500, 400);
}
```

Creiamo un canvas di 500×400 pixel.

### Il Calcolo della Profondità

```javascript
const zNear = 0.1;
const zFar = 1000;

function computeDepth(z_vertex) {
  let z = z_vertex * -(zNear + zFar)/(zNear - zFar);
  z += (2 * zNear * zFar)/(zNear - zFar);
  return z;
}
```

Questa funzione calcola la profondità del punto usando i piani near e far, utile per il depth clipping (decidere cosa è visibile).

### La Proiezione

```javascript
function draw() {
  background(220);
  
  for(let i = 0; i < points.length; i++) {
    const aspectRatio = height/width;
    
    // Trasliamo il cubo nello spazio
    let translate_x = 1;
    let translate_y = 1.5;
    let translate_z = 5;
    
    let x = (points[i][0] + translate_x) * aspectRatio;
    let y = (points[i][1] + translate_y) * -1;
    let z = points[i][2] + translate_z;
```

Per ogni vertice:
- Calcoliamo l'aspect ratio per mantenere proporzioni corrette
- Trasliamo il cubo nella posizione desiderata (soprattutto in z=5, davanti alla camera)
- Invertiamo y perché lo schermo ha coordinate invertite

```javascript
    let zDepth = computeDepth(z);
    
    // Proiezione prospettica
    if(z != 0) {
      x /= z;
      y /= z;
      zDepth /= z;
    }
```

Dividiamo x e y per z: questa è la **proiezione prospettica**. Gli oggetti più lontani (z maggiore) risultano più piccoli.

```javascript
    // Convertiamo da spazio normalizzato (-1,1) a coordinate schermo
    x = map(x, -1, 1, 0, width);
    y = map(y, -1, 1, 0, height);
    
    // Disegniamo solo se il punto è visibile
    if(zDepth < 1) {
      strokeWeight(5);
      point(x, y);
    }
  }
}
```

Convertiamo le coordinate normalizzate in pixel e disegniamo solo i punti visibili (zDepth < 1).

## Concetti Chiave

### Proiezione Prospettica
Dividere x e y per z crea l'effetto prospettico: oggetti lontani appaiono più piccoli, come nella realtà.

### Trasformazioni
- **Traslazione**: spostiamo il cubo nello spazio 3D
- **Aspect Ratio**: correggiamo la proporzione per canvas non quadrati
- **Mapping**: convertiamo da coordinate 3D a coordinate schermo

### Depth Clipping
Il test `if(zDepth < 1)` determina se un punto è visibile o fuori dal frustum della camera.

## Provalo

Vai su [editor.p5js.org](https://editor.p5js.org/), copia il codice e osserva gli 8 vertici del cubo proiettati sullo schermo!

```javascript
let points = [
  [-1, -1, -1], // P1
  [1, -1, -1], // P2
  [1, 1, -1], // P3
  [-1, 1, -1], // P4
  [-1, -1, 1], // P5
  [1, -1, 1], // P6
  [1, 1, 1], // P7
  [-1, 1, 1] // P8
];


function setup() {
  createCanvas(500, 400);
}

const zNear= 0.1;
const zFar = 1000;

function computeDepth(z_vertex){
  let z = z_vertex * -(zNear + zFar)/(zNear - zFar);
  z += (2 *zNear * zFar)/(zNear - zFar);
  return z;
}

function draw() {
  background(220);
  for(let i = 0; i< points.length; i++){
    const aspectRatio = height/width;
    
    let translate_x = 1;
    let translate_y = 1.5;
    let translate_z =5;
    
    let x = (points[i][0]  + translate_x) * aspectRatio;
    let y = (points[i][1] + translate_y) * -1;
    let z = points[i][2] +translate_z;
    
    let zDepth = computeDepth(z);
    
    if(z!=0){ // normalizzazione -> -1,1
      x/=z;
      y/=z;
      zDepth/=z;
    }
    
    
    
    x = map(x,-1,1,0,width);
    y = map(y,-1,1,0,height);
    
    if(zDepth< 1){
      strokeWeight(5)
      point(x,y)
    }
    
  }
 
  
}
```