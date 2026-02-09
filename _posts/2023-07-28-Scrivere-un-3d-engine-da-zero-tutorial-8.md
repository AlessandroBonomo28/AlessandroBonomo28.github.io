---
categories: [tutorials,3Dengine]
tags: [tutorial, 3Dengine, p5js, builtfromscratch]
---
# Importare Modelli 3D (OBJ) in p5.js

{% include embed/youtube.html id='YTYGDXST2cA' %}

## Cosa Facciamo

In questo tutorial finale portiamo il nostro motore 3D al livello successivo: importiamo **modelli 3D reali** nel formato OBJ! Creiamo un parser che legge file .obj e li converte in triangoli renderizzabili dal nostro engine. Ora possiamo caricare qualsiasi modello 3D!

## I Due Script

### objImporter.js - Il Parser

Questo script definisce una classe che legge e interpreta file OBJ.

```javascript
class ObjImporter {
  constructor() {
    this.importDone = false;
    this.triangoli = [];
    this.vertici = [];
    
    let fileInput = createFileInput(file => {
      const reader = new FileReader();
      reader.onload = e => {
        const data = e.target.result;
        this.importDone = false;
        this.triangoli = [];
        this.vertici = [];
        this.parseData(data);
        this.importDone = true;
        print(this.triangoli.length + " triangles imported.");
      };
      reader.readAsText(file.file);
    });
  }
```

Nel costruttore:
- Creiamo un input per selezionare file
- Usiamo `FileReader` per leggere il file come testo
- Quando il file è caricato, lo passiamo al parser

### Parsing dei Dati

```javascript
  parseData(data) {
    let inputStr = data;
    let charK = '\n';
    let i = 0, j = 0;

    while ((j = inputStr.indexOf(charK, i)) !== -1) {
      this.parseLine(inputStr.substring(i, j));
      i = j + 1;
    }
    this.parseLine(inputStr.substring(i));
  }
```

Dividiamo il file OBJ in righe (separate da `\n`) e processiamo ogni riga.

### Parsing dei Vertici

```javascript
  parseVertex(data) {
    let splitted = data.split(' ');
    this.vertici.push(splitted);
  }
```

Una riga tipo `v 1.0 0.5 -0.3` definisce un vertice. Separiamo per spazi e salviamo le coordinate nell'array `vertici`.

### Parsing delle Facce

```javascript
  parseFace(data) {
    let splitted = data.split(' ');
    let triRead = [];
    
    for(let i = 0; i < splitted.length; i++) {
      if(splitted[i].includes('/')) {
        let slashSplit = splitted[i].split('/');
        triRead = [...triRead, ...this.vertici[slashSplit[0]-1]];
      } else {
        triRead = [...triRead, ...this.vertici[splitted[i]-1]];
      }
    }
    
    this.triangoli.push(triRead);
  }
```

Una riga tipo `f 1 2 3` o `f 1/1/1 2/2/2 3/3/3` definisce una faccia (triangolo).
- I numeri si riferiscono agli indici dei vertici (base-1)
- Se c'è `/`, prendiamo solo il primo numero (indice vertice)
- Recuperiamo le coordinate dai vertici salvati e creiamo un triangolo

### Identificazione delle Righe

```javascript
  parseLine(lineData) {
    if(lineData[0] == 'v' && lineData[1] != 'n' && lineData[1] != 't') {
      this.parseVertex(lineData.substring(2));
    } else if(lineData[0] == 'f') {
      this.parseFace(lineData.substring(2));
    }
  }
}
```

Ogni riga OBJ inizia con un prefisso:
- `v` → vertice (coordinate x,y,z)
- `vn` → normale (ignorata)
- `vt` → texture coordinate (ignorata)
- `f` → faccia (triangolo)

Processiamo solo `v` e `f`.

### sketch.js - Il Rendering

Il file principale ora usa l'importer invece di triangoli hardcoded:

```javascript
let triangles = [];
let objImporter;

function setup() {
  objImporter = new ObjImporter();
  createCanvas(winWidth, winHeight);
}

function draw() {
  let projected_triangles = [];
  background(220);
  
  if(objImporter.importDone) {
    triangles = objImporter.triangoli;
  }
  
  // ... resto del rendering come prima
}
```

Quando `objImporter.importDone` è true, copiamo i triangoli importati nell'array `triangles` e il nostro engine li renderizza!

### Controlli Extra

```javascript
function keyPressed() {
  // ... controlli precedenti ...
  
  if(key === 'h') {
    hideTrianglePoints = !hideTrianglePoints;
    hideTriangleStroke = !hideTriangleStroke;
  }
}
```

Premendo **H** possiamo nascondere i punti e i bordi dei triangoli per vedere solo le facce solide.

## Il Formato OBJ

Un file OBJ è un formato testo semplice:

```
# Commento
v 0.0 0.0 0.0
v 1.0 0.0 0.0
v 0.5 1.0 0.0
f 1 2 3
```

- `v x y z` → definisce un vertice
- `f i1 i2 i3` → definisce un triangolo usando gli indici dei vertici
- Gli indici partono da 1 (non da 0!)

Formati più complessi possono includere:
- `f 1/1/1 2/2/2 3/3/3` → vertice/texture/normale
- Facce con 4+ vertici (quad) → il nostro parser li considera come un solo "triangolo" che verrà poi gestito dal clipping

## Pipeline Completa

1. **Upload**: l'utente seleziona un file .obj
2. **Parsing**: `ObjImporter` legge vertici e facce
3. **Conversione**: i dati OBJ diventano array di triangoli
4. **Rendering**: il nostro engine 3D renderizza i triangoli con:
   - Trasformazioni (rotazione, scala, traslazione)
   - Back-face culling
   - Illuminazione
   - Clipping
   - Proiezione prospettica
   - Painter's algorithm

## Concetti Chiave

### File Reader API

JavaScript può leggere file locali usando `FileReader`:
```javascript
const reader = new FileReader();
reader.onload = e => {
  const data = e.target.result;
  // usa i dati
};
reader.readAsText(file);
```

### Indicizzazione Vertici

Invece di duplicare coordinate, OBJ usa **indici**:
- I vertici sono definiti una volta
- Le facce referenziano i vertici per indice
- Risparmia spazio e memoria

### Triangolazione

Molti modelli hanno quad (4 vertici). Il nostro parser li tratta come array più lunghi che il sistema di clipping può poi gestire.

## Provalo

1. Vai su [editor.p5js.org](https://editor.p5js.org/)
2. Crea due file: `sketch.js` e `objImporter.js`
3. Copia il codice in entrambi i file
4. Scarica un modello .obj semplice (es. da [qui](https://github.com/AlessandroBonomo28/Elegoo-TouchScreen-2.8-GFX-fun/tree/base/p5%20js%20testing/youtube/obj%20files))
5. Premi play e seleziona il file .obj
6. Guarda il tuo modello renderizzato in 3D!

Controlli:
- **WASD** → movimento
- **Mouse drag** → guarda in giro
- **Spazio/Shift** → su/giù
- **H** → nascondi/mostra wireframe
- **T** → mostra/nascondi coordinate

### Il codice intero

Ecco qui `objImporter.js`:

```javascript
class ObjImporter{
  constructor(){
    this.importDone = false;
    this.triangoli = [];
    this.vertici = [];
    let fileInput = createFileInput(file => {
      const reader = new FileReader();
      reader.onload = e => {
        const data = e.target.result;
        this.importDone = false;
        this.triangoli = [];
        this.vertici = [];
        this.parseData(data);
        this.importDone = true;
        print(this.triangoli.length+" triangles imported.");
      };
      reader.readAsText(file.file);
    });
  }

  parseData(data){
    let inputStr = data 
    let charK = '\n';
    let i=0, j = 0;

    while ((j = inputStr.indexOf(charK, i)) !== -1) {
      this.parseLine(inputStr.substring(i, j))
      i = j + 1;
    }
    this.parseLine(inputStr.substring(i))
  }
   parseVertex(data){
    let splitted = data.split(' ')
    this.vertici.push(splitted)
  }
  
  parseFace(data){
    let splitted = data.split(' ')
    let triRead = [];
    for(let i =0;i<splitted.length;i++){
      if(splitted[i].includes('/')){
        let slashSplit = splitted[i].split('/');
        triRead = [...triRead, ...this.vertici[slashSplit[0]-1]];
      } else {
        triRead = [...triRead, ...this.vertici[splitted[i]-1]];
      }
    }
    //print(triRead)
    this.triangoli.push(triRead);
  }
  
  parseLine(lineData){
    if(lineData[0]=='v' && lineData[1]!='n' && lineData[1]!='t'){
      this.parseVertex(lineData.substring(2))
    } else if(lineData[0] == 'f'){
      this.parseFace(lineData.substring(2))
    }
  }
}
```

Ecco qui `sketch.js`:

```javascript
const zNear= 0.1;
const zFar = 1000;
const winWidth = 700;
const winHeight = 400;
const aspectRatio = winHeight/winWidth;

let lightDirection = [0,0,1];

let cameraYaw = 0;
let cameraPitch = 0;
let playerPos = [0,0,0]

let vUp = [0,1,0];
let vRight = [1,0,0];
let vForward = [0,0,1];

let hideTrianglePoints = true;
let hideTriangleStroke = true;
let isTextVisible = false;
let isDragging = false;
let xDrag = 0,yDrag = 0;
let startX,startY;

let triangles = [];
let objImporter;
/*
// clockwise triangle vertex ordering
let triangles = [
  // SOUTH
  
  [0.0, 0.0, 0.0,    0.0, 1.0, 0.0,    1.0, 1.0, 0.0],
  [0.0, 0.0, 0.0,    1.0, 1.0, 0.0,    1.0, 0.0, 0.0],
  

  // EAST
  [1.0, 0.0, 0.0,    1.0, 1.0, 0.0,    1.0, 1.0, 1.0],
  [1.0, 0.0, 0.0,    1.0, 1.0, 1.0,    1.0, 0.0, 1.0],

  // NORTH
  [1.0, 0.0, 1.0,    1.0, 1.0, 1.0,    0.0, 1.0, 1.0],
  [1.0, 0.0, 1.0,    0.0, 1.0, 1.0,    0.0, 0.0, 1.0],

  // WEST
  [0.0, 0.0, 1.0,    0.0, 1.0, 1.0,    0.0, 1.0, 0.0],
  [0.0, 0.0, 1.0,    0.0, 1.0, 0.0,    0.0, 0.0, 0.0],

  // TOP
  [0.0, 1.0, 0.0,    0.0, 1.0, 1.0,    1.0, 1.0, 1.0],
  [0.0, 1.0, 0.0,    1.0, 1.0, 1.0,    1.0, 1.0, 0.0],

  // BOTTOM
  [1.0, 0.0, 1.0,    0.0, 0.0, 1.0,    0.0, 0.0, 0.0],
  [1.0, 0.0, 1.0,    0.0, 0.0, 0.0,    1.0, 0.0, 0.0],
];
*/

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
function crossProduct(v1, v2) {
  const x = v1[1] * v2[2] - v1[2] * v2[1];
  const y = v1[2] * v2[0] - v1[0] * v2[2];
  const z = v1[0] * v2[1] - v1[1] * v2[0];
  return [x, y, z];
}

function sub(v1,v2){
  return [v1[0]-v2[0],v1[1]-v2[1], v1[2]-v2[2]];
}

function dotProduct(v1,v2){
  return v1[0]*v2[0] + v1[1]*v2[1] + v1[2]*v2[2];
}

function getLookAtMatrix(vUp,vRight,vForward,vPos){
  let rotation = [
    [vRight[0], vRight[1], vRight[2], 0],
    [vUp[0], vUp[1], vUp[2], 0],
    [vForward[0], vForward[1], vForward[2], 0],
    [0, 0, 0, 1]
  ];
  let translation = [
    [1, 0, 0, -vPos[0]],
    [0, 1, 0, -vPos[1]],
    [0, 0, 1, -vPos[2]],
    [0, 0, 0, 1],
  ];
  return mat4x4(rotation,translation);
}

function setup() {
  objImporter = new ObjImporter();
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

function isAboveOrOntoPlane(v,planePoint,planeNormal){
  planeNormal = vec3normalize(planeNormal);
  const d1 = dotProduct(v,planeNormal);
  const d2 = dotProduct(planePoint,planeNormal);
  return (d1-d2) >= 0;
}

function interceptPlane(vOrigin,vDirection,planePoint,planeNormal){
  planeNormal = vec3normalize(planeNormal);
  vDirection = vec3normalize(vDirection);
  
  const dot1 = dotProduct(sub(planePoint,vOrigin),planeNormal);
  const dot2 = dotProduct(vDirection,planeNormal);
  
  if(dot1==0 && dot2 ==0) return vOrigin;
  else if(dot1!=0 && dot2 ==0) return 'impossible';
  
  const t = dot1/dot2;
  
  return [vOrigin[0]+vDirection[0]*t,
          vOrigin[1]+vDirection[1]*t,
          vOrigin[2]+vDirection[2]*t];
}

// ritorna array di triangoli risultanti
function clipAgainstPlane(triToClip,planePoint,planeNormalTowardsInside) {
  let insideCount = 0;
  let outsideVertexes = [];
  let insideVertexes = [];
  for(let i=0;i<3;i++) { // foreach vertex
    let x = triToClip[i * 3];
    let y = triToClip[i * 3 + 1];
    let z = triToClip[i * 3 + 2];
    const isInside = isAboveOrOntoPlane([x,y,z],planePoint,
                                 planeNormalTowardsInside);
    if(isInside) {
      insideCount++;
      // ordine di push in array indifferente
      
      //insideVertexes = [...insideVertexes ,[x,y,z]];
      insideVertexes = [[x,y,z],...insideVertexes]; 
    } else {
      // ordine di push in array indifferente
      
      //outsideVertexes = [...outsideVertexes ,[x,y,z]];
      outsideVertexes = [[x,y,z],...outsideVertexes ];
    } 
    
  }
  
  if(insideCount == 0)  return [];
  
  if(insideCount == 3) return [triToClip];
  
  if(insideCount == 1) { // 2 outside
    const dir1 = sub(outsideVertexes[0],insideVertexes[0]);
    const dir2 = sub(outsideVertexes[1],insideVertexes[0]);
    
    const intercept1 = interceptPlane(outsideVertexes[0],dir1,
                                   planePoint,planeNormalTowardsInside);
    const intercept2  = interceptPlane(outsideVertexes[1],dir2,
                                   planePoint,planeNormalTowardsInside);
    // ordinamento dei vertici nell'array indifferente
    const newTri = [...insideVertexes[0],
                    ...intercept1,
                    ...intercept2
    ];
    return [newTri];
  } else { // 1 outside (form quad)
    const dir1 = sub(outsideVertexes[0],insideVertexes[0]);
    const dir2 = sub(outsideVertexes[0],insideVertexes[1]);
    
    const intercept1 = interceptPlane(outsideVertexes[0],dir1,
                                   planePoint,planeNormalTowardsInside);
    const intercept2  = interceptPlane(outsideVertexes[0],dir2,
                                   planePoint,planeNormalTowardsInside);
    
    /*
    anche la seguente configurazione va bene per tri1,tri2:
    
    const tri1 = [...intercept1,
                  ...insideVertexes[0],
                  ...insideVertexes[1]
                  
                  
    ];
    
    const tri2 = [...insideVertexes[1],
                  ...intercept1,
                  ...intercept2 
    ];
    */
    
    // ordinamento dei vertici nell'array indifferente
    const tri1 = [...insideVertexes[0],
                  ...insideVertexes[1],
                  ...intercept2
                  
                  
    ];
    // ordinamento dei vertici nell'array indifferente
    const tri2 = [...insideVertexes[0],
                  ...intercept1,
                  ...intercept2 
    ];
    
    return [tri1,tri2]; // ordine tri1,tri2 in array non ha importanza
  }
}

let angleSum = 0;
function draw() {
  background(220);
  stroke('black');
  while(!objImporter.importDone){
    textSize(20);
    noStroke()
    textAlign(CENTER,CENTER)
    text("Obj file not loaded",
        width/2,height/2);
    stroke('black')
    return;
  }
  triangles = objImporter.triangoli;
  
  let projected_triangles = [];
  
  for(let i= 0; i< triangles.length; i++){
    let triWorldSpace = [];
    for(let j= 0; j< 3; j++){
      let vertice = [triangles[i][j*3], // x
                     triangles[i][(j*3)+1], // y
                     triangles[i][(j*3)+2], // z
                     1]; // w
      
      let translate_x = 1;
      let translate_y = 0;
      let translate_z =2;

      let scale_x = 1;
      let scale_y = 1;
      let scale_z =1;

      scale_x = scale_y = scale_z = 1;

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
      let axisRotation = vec3normalize([1,1,1]);

      let matRotation = 
          //getRotationMatrixArbitraryAxis(axisRotation,angleSum);
      //getRotationMatrixY(angleSum);
      
      mat4x4(getRotationMatrixZ(angleSum),
          mat4x4(getRotationMatrixY(angleSum),getRotationMatrixX(angleSum)
            ));
      
      let matTranslationScaleRotation = mat4x4(mat4x4(translationMatrix,scaleMatrix),matRotation);


      //vertice = multiplyVectorMatrix(vertice,matRotation);
      vertice = multiplyVectorMatrix(vertice,matTranslationScaleRotation);
      
      triWorldSpace = [...triWorldSpace,vertice[0],vertice[1],vertice[2]]
    }
    // [ x,y,z ]
    // [ x,y,z , x,y,z ]
    // [ x,y,z , x,y,z ,x,y,z]
    
    const v1 = [triWorldSpace[0], // x
               triWorldSpace[1], // y
               triWorldSpace[2]]; // z
               
    const v2 = [triWorldSpace[3], // x
               triWorldSpace[4], // y
               triWorldSpace[5]] // z;
    
    const v3 = [triWorldSpace[6], // x
               triWorldSpace[7], // y
               triWorldSpace[8]]; // z
               
    let triNormal = crossProduct(sub(v2,v1),sub(v3,v1));
    triNormal = vec3normalize(triNormal);
    
    const lookAtTriangle = sub(playerPos,triWorldSpace);
    
    const visible = -dotProduct(lookAtTriangle,triNormal);
    const shading = -dotProduct(lightDirection,triNormal);
    if(visible > 0) continue; // triangolo non visibile (Eye > 90°)
    
    let triViewSpace = [];
    for(let j= 0; j< 3; j++){
      let vertice = [triWorldSpace[j*3], // x
                     triWorldSpace[(j*3)+1], // y
                     triWorldSpace[(j*3)+2], // z
                     1]; // w
      
       vertice = multiplyVectorMatrix(vertice,getLookAtMatrix(
          vUp,vRight,vForward,playerPos
        ));
      triViewSpace = [...triViewSpace,vertice[0],vertice[1],vertice[2]]
    }
    
    // Near plane in front of camera 
    // in order to prevent ZDepth >=1 (objects go behind camera)
    const pointNearPlane = [0, 0, 0.01];
    const normalNearPlane = [0,0,1];
    let clippedTriangles = clipAgainstPlane(triViewSpace,pointNearPlane,normalNearPlane);
    
    for(let clippedIndex = 0;clippedIndex<clippedTriangles.length;clippedIndex++){
      
      let triScreenSpace = [];
      for(let j= 0; j< 3; j++) {
        let vertice = [clippedTriangles[clippedIndex][j*3], // x
                       clippedTriangles[clippedIndex][(j*3)+1], // y
                       clippedTriangles[clippedIndex][(j*3)+2], // z
                       1]; // w

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
          if(!hideTrianglePoints) point(x,y);
          if(isTextVisible){
            strokeWeight(0);
            textSize(10);
            textAlign(LEFT, CENTER);
            const off = 10;
            const _x = round(triWorldSpace[j*3] ,1);
            const _y = round(triWorldSpace[j*3+1] ,1);
            const _z = round(triWorldSpace[j*3+2] ,1);
            text("("+_x+","+_y+","+_z+")", x+off,y+off);
          }
        }

        triScreenSpace = [...triScreenSpace,x,y,zDepth];
      }
      triScreenSpace.shading = shading
      projected_triangles.push(triScreenSpace)
      
    }
    
    
    
  }
  
  
  
  function avgZDepth(tri){
    return (tri[2] +tri[5]+tri[8])/3;
  }
  function compareZDepth(a, b) {
    return   avgZDepth(b) - avgZDepth(a); 
  }
  projected_triangles.sort(compareZDepth);
  
  for(let i=0;i<projected_triangles.length;i++){
    const triColor = 255 * max(projected_triangles[i].shading, 0.15);
    stroke('black')
    strokeWeight(1);
    if(hideTriangleStroke)stroke(triColor) 
    fill(triColor);


    // Legenda: [ [point on plane], [normal to plane] ]
    // ordinamento di left,up,right,bottom nell'array indifferente
    const clipPlanes = [
      [ [0,0,0] , [1,0,0] ],       // Left
      [ [0,0,0] , [0,1,0] ],       // Up
      [ [width,0,0] , [-1,0,0] ],  // Right
      [ [0,height,0] , [0,-1,0] ], // Bottom
    ]

    let triQueue = [projected_triangles[i]];
    let newTriangles = 1;

    for(let planeIndex= 0; planeIndex<clipPlanes.length; planeIndex++) {
      while(newTriangles > 0) {
        const pointPlane = clipPlanes[planeIndex][0];
        const normalPlane = clipPlanes[planeIndex][1];
        const tri = triQueue[triQueue.length-1];
        triQueue.length--;
        newTriangles--;
        triQueue = [ ...clipAgainstPlane(tri,pointPlane,normalPlane),...triQueue];
      }
      newTriangles = triQueue.length;
    }

    for(let n=0;n<triQueue.length;n++) {
      triangle(triQueue[n][0],triQueue[n][1],
               triQueue[n][3],triQueue[n][4],
               triQueue[n][6],triQueue[n][7])
    }  
    /* NON PIU' NECESSARIO (risolvo con clip near plane)
    if(projected_triangles[i][2] <1 &&
       projected_triangles[i][5] <1 &&
       projected_triangles[i][8] <1) { // clipping easy zDepths >= 1  
    }
    */
    
  }
   
  fill(0)
  strokeWeight(0);
  textSize(15);
  textAlign(LEFT, CENTER);
  const _yaw = round(cameraYaw * (180/Math.PI) % 360,1);
  const _pitch  = round(cameraPitch * (180/Math.PI) % 360,1);
  text("Position: ("+playerPos.map(x=>round(x,1))+")",5,40);
  text("Rotation: ("+_yaw+"°,"+_pitch+"°,0)",5,20);
  
  if(isDragging)
    updateLook();
  
  angleSum += deltaTime * Math.PI/5000;
  if(angleSum >= Math.PI*2 )
    angleSum =0;
  
  
  renderAxis([1,0,0],'red','X');
  renderAxis([0,1,0],'green','Y');
  renderAxis([0,0,1],'blue','Z');
  
}


let usingRelativeMovement = false;

function keyPressed() {
  const unit = 0.5;
  if(usingRelativeMovement){
    if (key === 'w') { // +z
      playerPos[2] += unit;
    } else if (key === 's') { // -z
      playerPos[2] -= unit;
    } else if (key === 'a') { // -x
      playerPos[0] -= unit;
    } else if (key === 'd') { // +x
      playerPos[0] += unit;
    } 
  } else {
    const vRight = crossProduct(vUp,vForward);
    if (key === 'w') {
      playerPos[0] += vForward[0]*unit;
      playerPos[1] += vForward[1]*unit;
      playerPos[2] += vForward[2]*unit;
    } else if (key === 's') { // -z
      playerPos[0] -= vForward[0]*unit;
      playerPos[1] -= vForward[1]*unit;
      playerPos[2] -= vForward[2]*unit;
    } else if(key === 'a'){
      playerPos[0] -= vRight[0]*unit;
      playerPos[1] -= vRight[1]*unit;
      playerPos[2] -= vRight[2]*unit;
    } else if (key === 'd') { // +x
      playerPos[0] += vRight[0]*unit;
      playerPos[1] += vRight[1]*unit;
      playerPos[2] += vRight[2]*unit;
    }
  }
  
  if(key === ' '){ // space +y
    playerPos[1] +=unit;
  }
  else if(key === 'Shift'){// -y
    playerPos[1]-=unit;
  }
  else if(key === 't'){
    isTextVisible = !isTextVisible;
  }
  else if(key === 'h'){
    hideTrianglePoints = !hideTrianglePoints;
    hideTriangleStroke = !hideTriangleStroke;
  }
}


function mousePressed() {
  isDragging = true;
  startX = mouseX;
  startY = mouseY;
}

function mouseDragged() {
  const speed = 0.05;
  const deltaX = (mouseX - startX) * speed;
  const deltaY = (mouseY - startY) * speed;
  
  xDrag += deltaX;
  yDrag += deltaY;
  
}

function mouseReleased() {
  isDragging = false;
}


function updateLook(){  
  const speed = 0.3;
  cameraYaw = map(xDrag, 0, width, 0, 2*Math.PI) * speed;
  cameraPitch = map(yDrag, 0, width, 0, 2*Math.PI) * speed;
  
  let rotationMat = mat4x4(getRotationMatrixY(cameraYaw), getRotationMatrixX(-cameraPitch));
  
  vForward =  multiplyVectorMatrix([0,0,1],rotationMat);
  vUp = multiplyVectorMatrix([0,1,0],rotationMat);
  
  vRight = crossProduct(vUp,vForward)
  
}

function renderAxis(axis,aColor,aText){
  axis = [...axis,1];
  const viewMatrix = getLookAtMatrix(vUp,vRight,vForward,playerPos);
  const o = [0, 0, 0, 1];
  // Calcola la posizione dell'origine nel sistema di coordinate dello schermo
  let origin = multiplyVectorMatrix(o,viewMatrix );
  origin = multiplyVectorMatrix(origin, projectionMatrix);
  let z = origin[3];
  if(z!=0){
    origin = origin.map(x => x/=z);
  }
  if (origin[2] < 1) {
    origin[0] = map(origin[0], -1, 1, 0, width);
    origin[1] = map(origin[1], -1, 1, 0, height);
    let xEnd = multiplyVectorMatrix(axis, viewMatrix);
    xEnd = multiplyVectorMatrix(xEnd, projectionMatrix);
    z = xEnd[3];
    if(z!=0){
      xEnd = xEnd.map(x => x/=z);
    }
    if (xEnd[2] < 1) {
      xEnd[0] = map(xEnd[0], -1, 1, 0, width);
      xEnd[1] = map(xEnd[1], -1, 1, 0, height);
      stroke(aColor);
      strokeWeight(2);
      line(origin[0], origin[1], xEnd[0], xEnd[1]);
      text(aText, xEnd[0], xEnd[1]);
    }
  }
  
}
```