---
categories: [tutorials,3Dengine]
tags: [tutorial, 3Dengine, p5js, builtfromscratch]
---
# Matrici di Proiezione in p5.js

{% include embed/youtube.html id='Ngx1xuyGa_w' %}

## Cosa Facciamo

In questo tutorial usiamo le **matrici** per fare la proiezione 3D. Invece di calcolare manualmente ogni trasformazione, usiamo una matrice di proiezione che fa tutto il lavoro in un colpo solo.

## Il Codice Spiegato

### Parametri e Vertici

```javascript
const zNear = 0.1;
const zFar = 1000;
const winWidth = 500;
const winHeight = 400;
const aspectRatio = winHeight/winWidth;

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

Definiamo i parametri della camera e gli 8 vertici del cubo.

### La Matrice di Proiezione

```javascript
let projectionMatrix = [
  [aspectRatio, 0, 0, 0],
  [0, -1, 0, 0],
  [0, 0, -(zFar + zNear)/(zNear - zFar), (2*zFar*zNear)/(zNear - zFar)],
  [0, 0, 1, 0]
];
```

Questa matrice 4×4 contiene **tutte** le trasformazioni:
- **Riga 1**: gestisce x e l'aspect ratio
- **Riga 2**: inverte y (da -1 a 1)
- **Riga 3**: calcola la profondità per il depth clipping
- **Riga 4**: copia z nella quarta componente per la divisione prospettica

### Moltiplicazione Matrice-Vettore

```javascript
function multiplyVectorMatrix(vector, matrix) {
  const result = [];
  
  for (let i = 0; i < matrix.length; i++) {
    let sum = 0;
    
    for (let j = 0; j < vector.length; j++) {
      sum += matrix[i][j] * vector[j];
    }
    
    result[i] = sum;
  }
  return result;
}
```

Questa funzione moltiplica un vettore (vertice 3D) per una matrice 4×4. È il cuore della trasformazione: prende un punto 3D e lo trasforma applicando tutte le operazioni della matrice.

### Il Loop di Rendering

```javascript
function draw() {
  background(220);
  
  for(let i = 0; i < points.length; i++) {
    let translate_x = 1;
    let translate_y = 1.5;
    let translate_z = 5;
    
    // Creiamo un vettore omogeneo [x, y, z, 1]
    let vertice = [...points[i], 1];
    
    vertice[0] += translate_x;
    vertice[1] += translate_y;
    vertice[2] += translate_z;
```

Per ogni vertice:
- Lo trasformiamo in coordinate omogenee aggiungendo 1 come quarta componente
- Applichiamo la traslazione manualmente

```javascript
    // Applichiamo la matrice di proiezione
    let projected = multiplyVectorMatrix(vertice, projectionMatrix);
    
    let x = projected[0];
    let y = projected[1];
    let zDepth = projected[2];
    let z = projected[3];
```

La moltiplicazione matrice-vettore produce il punto proiettato con 4 componenti.

```javascript
    // Divisione prospettica
    if(z != 0) {
      x /= z;
      y /= z;
      zDepth /= z;
    }
    
    // Convertiamo a coordinate schermo
    x = map(x, -1, 1, 0, width);
    y = map(y, -1, 1, 0, height);
    
    // Disegniamo solo se visibile
    if(zDepth < 1) {
      strokeWeight(5);
      point(x, y);
    }
  }
}
```

Dividiamo per z (divisione prospettica), convertiamo in pixel e disegniamo.

## Perché le Matrici?

### Vantaggi
- **Una sola operazione**: tutte le trasformazioni in un colpo
- **Componibili**: puoi moltiplicare matrici tra loro per combinarle
- **Standard**: è così che funzionano OpenGL, WebGL e tutti i motori 3D

### Coordinate Omogenee
Usiamo 4 componenti [x, y, z, w] invece di 3 perché:
- La quarta componente (w=1) permette di rappresentare traslazioni con matrici
- Dopo la proiezione, w contiene z per la divisione prospettica

## Il Flusso Completo

1. Vertice 3D → [x, y, z, 1]
2. Traslazione → sposta il cubo davanti alla camera
3. Matrice di proiezione → trasforma tutto in un colpo
4. Divisione prospettica → dividi x, y, z per w
5. Mapping → converti in coordinate schermo
6. Disegna se visibile

## Provalo

Vai su [editor.p5js.org](https://editor.p5js.org/) e osserva come una singola moltiplicazione matrice-vettore fa tutto il lavoro!

```javascript
const zNear= 0.1;
const zFar = 1000;
const winWidth = 500;
const winHeight = 400;
const aspectRatio = winHeight/winWidth;

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

let projectionMatrix = [
  [aspectRatio, 0, 0, 0],
  [0, -1, 0, 0],
  [0, 0, -(zFar + zNear)/(zNear - zFar), (2*zFar*zNear)/(zNear -   zFar)],
  [0, 0,1 , 0]
];

function multiplyVectorMatrix(vector,matrix) {
  const result = [];
  
  for (let i = 0; i < matrix.length; i++) {
    let sum = 0;
    
    for (let j = 0; j < vector.length; j++) {
      sum += matrix[i][j] * vector[j];
    }
    
    result[i] = sum;
  }
  return result;
}

function setup() {
  createCanvas(winWidth, winHeight);
}




function draw() {
  background(220);
  for(let i = 0; i< points.length; i++){
    
    let translate_x = 1;
    let translate_y = 1.5;
    let translate_z =5;
    
    // x,y,z,1
    let vertice = [ ...points[i] ,1]
    
    vertice[0]+=translate_x;
    vertice[1]+=translate_y;
    vertice[2]+=translate_z;
    
    let projected = multiplyVectorMatrix(vertice,projectionMatrix);
    
    let x = projected[0];
    let y = projected[1];
    let zDepth = projected[2];
    let z = projected[3];
    
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