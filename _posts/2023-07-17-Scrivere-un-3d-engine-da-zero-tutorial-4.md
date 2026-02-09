---
categories: [tutorials,3Dengine]
tags: [tutorial, 3Dengine, p5js, builtfromscratch]
---
# Matrici di Trasformazione in p5.js

{% include embed/youtube.html id='e8Et4QFrZhc' %}

## Cosa Facciamo

In questo tutorial completiamo il cubo 3D! Aggiungiamo **rotazione**, **scala**, **traslazione** usando matrici, e colleghiamo i vertici con linee per vedere un cubo solido che ruota nello spazio.

## Il Codice Spiegato

### Funzioni Vettoriali

```javascript
function vec3Len(v) {
  return Math.sqrt(v[0]*v[0] + v[1]*v[1] + v[2]*v[2]);
}

function vec3normalize(v) {
  const len = vec3Len(v);
  if(len == 0) return v;
  return [v[0]/len, v[1]/len, v[2]/len];
}
```

Calcoliamo la lunghezza di un vettore e lo normalizziamo (portandolo a lunghezza 1). Serve per definire assi di rotazione.

### Matrici di Rotazione

```javascript
function getRotationMatrixX(angle) {
  let rotationMatrixX = [
    [1, 0, 0, 0],
    [0, cos(angle), sin(angle), 0],
    [0, -sin(angle), cos(angle), 0],
    [0, 0, 0, 1]
  ];
  return rotationMatrixX;
}

function getRotationMatrixY(angle) {
  angle = -angle;
  let rotationMatrixY = [
    [Math.cos(angle), 0, -Math.sin(angle), 0],
    [0, 1, 0, 0],
    [Math.sin(angle), 0, Math.cos(angle), 0],
    [0, 0, 0, 1]
  ];
  return rotationMatrixY;
}

function getRotationMatrixZ(angle) {
  let rotationMatrixZ = [
    [Math.cos(angle), Math.sin(angle), 0, 0],
    [-Math.sin(angle), Math.cos(angle), 0, 0],
    [0, 0, 1, 0],
    [0, 0, 0, 1]
  ];
  return rotationMatrixZ;
}
```

Tre matrici per ruotare attorno agli assi X, Y e Z. Usano seno e coseno dell'angolo.

### Rotazione su Asse Arbitrario

```javascript
function getRotationMatrixArbitraryAxis(a, theta) {
  const mat = [
    [
      a[0]*a[0]*(1-Math.cos(theta))+Math.cos(theta), 
      a[0]*a[1]*(1-Math.cos(theta))+a[2]*Math.sin(theta),
      a[0]*a[2]*(1-Math.cos(theta))-a[1]*Math.sin(theta),
      0
    ],
    // ... altre righe
  ];
  return mat;
}
```

Questa matrice permette di ruotare attorno a **qualsiasi asse** nello spazio 3D, non solo X, Y o Z. Usa la formula di Rodrigues.

### Moltiplicazione tra Matrici

```javascript
function mat4x4(mat1, mat2) {
  const result = [];
  
  for (let i = 0; i < 4; i++) {
    result[i] = [];
    for (let j = 0; j < 4; j++) {
      let sum = 0;
      for (let k = 0; k < 4; k++) {
        sum += mat1[i][k] * mat2[k][j];
      }
      result[i][j] = sum;
    }
  }
  return result;
}
```

Moltiplica due matrici 4×4. Serve per combinare più trasformazioni in una sola matrice.

### Il Loop Principale

```javascript
let angleSum = 0;

function draw() {
  let projected_points = [];
  background(220);
  
  for(let i = 0; i < points.length; i++) {
    let translate_x = 0.5;
    let translate_y = 0;
    let translate_z = 4;
    
    let scale_x = 1;
    let scale_y = 1;
    let scale_z = 1;
    
    let vertice = [...points[i], 1];
```

Inizializziamo l'array per i punti proiettati e i parametri di trasformazione.

```javascript
    const scaleMatrix = [
      [scale_x, 0, 0, 0],
      [0, scale_y, 0, 0],
      [0, 0, scale_z, 0],
      [0, 0, 0, 1]
    ];
    
    const translationMatrix = [
      [1, 0, 0, translate_x],
      [0, 1, 0, translate_y],
      [0, 0, 1, translate_z],
      [0, 0, 0, 1]
    ];
```

Creiamo le matrici di scala e traslazione.

```javascript
    let mat = mat4x4(translationMatrix, scaleMatrix);
    
    let axis = vec3normalize([1, 1, 1]);
    
    vertice = multiplyVectorMatrix(vertice, getRotationMatrixArbitraryAxis(axis, angleSum));
    vertice = multiplyVectorMatrix(vertice, mat);
```

- Combiniamo traslazione e scala in una matrice
- Definiamo l'asse di rotazione [1,1,1] normalizzato (rotazione diagonale)
- Applichiamo prima la rotazione, poi traslazione e scala

```javascript
    let projected = multiplyVectorMatrix(vertice, projectionMatrix);
    
    let x = projected[0];
    let y = projected[1];
    let zDepth = projected[2];
    let z = projected[3];
    
    if(z != 0) {
      x /= z;
      y /= z;
      zDepth /= z;
    }
    
    x = map(x, -1, 1, 0, width);
    y = map(y, -1, 1, 0, height);
    
    if(zDepth < 1) {
      strokeWeight(5);
      point(x, y);
    }
    projected_points.push([x, y, zDepth, z]);
  }
```

Proiettiamo, normalizziamo e salviamo tutti i punti.

### Disegnare le Linee del Cubo

```javascript
  for(let i = 0; i < 4; i++) {
    let j = (i + 1) % 4;
    
    if (projected_points[i][2] >= 1 ||
        projected_points[j][2] >= 1 ||
        projected_points[j + 4][2] >= 1)
      continue;
    
    if(i === 0) stroke('blue');
    if(i === 1) stroke('red');
    if(i === 2) stroke('green');
    if(i === 3) stroke('black');
    
    strokeWeight(1);
    line(projected_points[i][0], projected_points[i][1],
         projected_points[j][0], projected_points[j][1]);
    line(projected_points[i + 4][0], projected_points[i + 4][1],
         projected_points[j + 4][0], projected_points[j + 4][1]);
    line(projected_points[i][0], projected_points[i][1],
         projected_points[i + 4][0], projected_points[i + 4][1]);
  }
```

Colleghiamo i vertici con linee colorate:
- Faccia frontale (vertici 0-3)
- Faccia posteriore (vertici 4-7)
- Connessioni tra le due facce
- Skippiamo le linee fuori dal frustum (clipping)

```javascript
  angleSum += deltaTime * Math.PI/5000;
  if(angleSum >= Math.PI*2)
    angleSum = 0;
}
```

Incrementiamo l'angolo ogni frame per far ruotare il cubo continuamente.

## Concetti Chiave

### Ordine delle Trasformazioni
L'ordine è importante! Nel codice: **Rotazione → Traslazione → Proiezione**

Se facessi Traslazione → Rotazione, il cubo ruoterebbe attorno a un punto diverso.

### Composizione di Matrici
Invece di applicare ogni matrice separatamente al vertice, possiamo moltiplicare le matrici tra loro e applicare il risultato una volta sola. È più efficiente!

### Clipping
Il controllo `if(zDepth >= 1)` evita di disegnare linee fuori dalla visuale della camera.

## Provalo

Vai su [editor.p5js.org](https://editor.p5js.org/) e osserva il cubo che ruota nello spazio 3D!

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

function vec3Len(v){
  return Math.sqrt(v[0]*v[0] + v[1]*v[1] + v[2]*v[2]);
}

function vec3normalize(v){
  const len = vec3Len(v);
  if(len == 0)return v;
  return [v[0]/len,v[1]/len,v[2]/len];
}

function getRotationMatrixArbitraryAxis(a,theta){
  const mat = [
      [
        a[0]*a[0]*(1-Math.cos(theta))+Math.cos(theta), 
        a[0]*a[1]*(1-Math.cos(theta))+a[2]*Math.sin(theta),
        a[0]*a[2]*(1-Math.cos(theta))-a[1]*Math.sin(theta),
        0
      ],
      [
        a[0]*a[1]*(1-Math.cos(theta))-a[2]*Math.sin(theta),
        a[1]*a[1]*(1-Math.cos(theta))+Math.cos(theta),
        a[1]*a[2]*(1-Math.cos(theta))+a[0]*Math.sin(theta),
        0
      ],
      [
        a[0]*a[2]*(1-Math.cos(theta))+a[1]*Math.sin(theta),
        a[1]*a[2]*(1-Math.cos(theta))-a[0]*Math.sin(theta),
        a[2]*a[2]*(1-Math.cos(theta))+Math.cos(theta),
        0
      ],
      [
        0,0,0,1
      ]
    ];
  return mat;
}
function getRotationMatrixY(angle){
  angle = -angle;
  let rotationMatrixY = [
    [Math.cos(angle), 0, -Math.sin(angle), 0],
    [0, 1, 0, 0],
    [Math.sin(angle), 0, Math.cos(angle), 0],
    [0, 0, 0, 1]
  ];
  return rotationMatrixY;
}


function getRotationMatrixX(angle){
  let rotationMatrixX = [
    [1, 0, 0, 0],
    [0, cos(angle), sin(angle), 0],
    [0, -sin(angle), cos(angle), 0],
    [0, 0, 0, 1]
  ];
  return rotationMatrixX;
}
function getRotationMatrixZ(angle){

  let rotationMatrixZ = [
    [Math.cos(angle), Math.sin(angle), 0, 0],
    [-Math.sin(angle), Math.cos(angle), 0, 0],
    [0, 0, 1, 0],
    [0, 0, 0, 1]
  ];
  return rotationMatrixZ;
}

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


function mat4x4(mat1, mat2) {
  const result = [];
  
  for (let i = 0; i < 4; i++) {
    result[i] = [];
    
    for (let j = 0; j < 4; j++) {
      let sum = 0;
      
      for (let k = 0; k < 4; k++) {
        sum += mat1[i][k] * mat2[k][j];
      }
      
      result[i][j] = sum;
    }
  }
  
  return result;
}

let angleSum = 0;
function draw() {
  let projected_points = [];
  background(220);
  for(let i = 0; i< points.length; i++){
    
    let translate_x = 0.5;
    let translate_y = 0;
    let translate_z =4;
    
    let scale_x = 1;
    let scale_y = 1;
    let scale_z =1;
    
    scale_x = scale_y = scale_z =1;
    
    // x,y,z,1
    let vertice = [ ...points[i] ,1]
    
    const scaleMatrix = [
      [scale_x,0,0,0],
      [0,scale_y,0,0],
      [0,0,scale_z,0],
      [0,0,0,1],
    ];
    
    const translationMatrix = [
      [1,0,0,translate_x],
      [0,1,0,translate_y],
      [0,0,1,translate_z],
      [0,0,0,1],
    ];
    
    //vertice = multiplyVectorMatrix(vertice,scaleMatrix);
    //vertice = multiplyVectorMatrix(vertice,translationMatrix);
    let mat = mat4x4(translationMatrix,scaleMatrix);
    
    let axis = vec3normalize([1,1,1]);
    
    vertice = multiplyVectorMatrix(vertice,getRotationMatrixArbitraryAxis(axis,angleSum));
    //vertice = multiplyVectorMatrix(vertice,getRotationMatrixZ(angleSum));
    
    vertice = multiplyVectorMatrix(vertice,mat);
    
    
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
    projected_points.push([x,y,zDepth,z]);
  }
  
  for(let i= 0;i<4;i++){
    let j = (i + 1) % 4;
    
    if (projected_points[i][2] >= 1 ||
        projected_points[j][2] >= 1 ||
        projected_points[j + 4][2] >= 1) // clip lines
      continue;
    
    if(i===0) stroke('blue');
    if(i===1) stroke('red');
    if(i===2) stroke('green');
    if(i===3) stroke('black');
    strokeWeight(1);
    line(projected_points[i][0], projected_points[i][1],
         projected_points[j][0], projected_points[j][1]);
    line(projected_points[i + 4][0], projected_points[i + 4][1],
         projected_points[j + 4][0], projected_points[j + 4][1]);
    line(projected_points[i][0], projected_points[i][1],
         projected_points[i + 4][0], projected_points[i + 4][1]);
    
  }
 
  angleSum += deltaTime * Math.PI/5000;
  if(angleSum >= Math.PI*2 )
    angleSum =0;
}
```