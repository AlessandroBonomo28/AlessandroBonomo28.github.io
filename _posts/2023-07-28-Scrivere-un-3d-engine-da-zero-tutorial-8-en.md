---
lang: en
hidden: true
lang_ref: 3d-engine-8
permalink: /en/posts/Scrivere-un-3d-engine-da-zero-tutorial-8/
categories: [tutorials,3Dengine]
tags: [tutorial, 3Dengine, p5js, builtfromscratch]
image:
  path: /assets/img/posts/3dengine/cover-8.jpg
  alt: Importing 3D models in OBJ format in p5.js
---
# Importing 3D Models (OBJ) in p5.js

{% include embed/youtube.html id='YTYGDXST2cA' %}

## What We're Doing

In this final tutorial we take our 3D engine to the next level: we import **real 3D models** in OBJ format! We create a parser that reads .obj files and converts them into triangles renderable by our engine. Now we can load any 3D model!

## The Two Scripts

### objImporter.js - The Parser

This script defines a class that reads and interprets OBJ files.

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

In the constructor:
- We create an input to select files
- We use `FileReader` to read the file as text
- When the file is loaded, we pass it to the parser

### Data Parsing

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

We divide the OBJ file into lines (separated by `\n`) and process each line.

### Vertex Parsing

```javascript
  parseVertex(data) {
    let splitted = data.split(' ');
    this.vertici.push(splitted);
  }
```

A line like `v 1.0 0.5 -0.3` defines a vertex. We split by spaces and save the coordinates in the `vertici` array.

### Face Parsing

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

A line like `f 1 2 3` or `f 1/1/1 2/2/2 3/3/3` defines a face (triangle).
- The numbers refer to the vertex indices (base-1)
- If there is a `/`, we take only the first number (vertex index)
- We retrieve the coordinates from the saved vertices and create a triangle

### Line Identification

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

Each OBJ line starts with a prefix:
- `v` → vertex (x,y,z coordinates)
- `vn` → normal (ignored)
- `vt` → texture coordinate (ignored)
- `f` → face (triangle)

We only process `v` and `f`.

### sketch.js - The Rendering

The main file now uses the importer instead of hardcoded triangles:

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

When `objImporter.importDone` is true, we copy the imported triangles into the `triangles` array and our engine renders them!

### Extra Controls

```javascript
function keyPressed() {
  // ... controlli precedenti ...
  
  if(key === 'h') {
    hideTrianglePoints = !hideTrianglePoints;
    hideTriangleStroke = !hideTriangleStroke;
  }
}
```

By pressing **H** we can hide the points and edges of the triangles to see only the solid faces.

## The OBJ Format

An OBJ file is a plain text format:

```
# Commento
v 0.0 0.0 0.0
v 1.0 0.0 0.0
v 0.5 1.0 0.0
f 1 2 3
```

- `v x y z` → defines a vertex
- `f i1 i2 i3` → defines a triangle using vertex indices
- Indices start from 1 (not 0!)

More complex formats can include:
- `f 1/1/1 2/2/2 3/3/3` → vertex/texture/normal
- Faces with 4+ vertices (quads) → our parser considers them as a single \"triangle\" which will then be handled by clipping

## Complete Pipeline

1. **Upload**: the user selects an .obj file
2. **Parsing**: `ObjImporter` reads vertices and faces
3. **Conversion**: OBJ data becomes an array of triangles
4. **Rendering**: our 3D engine renders the triangles with:
   - Transformations (rotation, scale, translation)
   - Back-face culling
   - Lighting
   - Clipping
   - Perspective projection
   - Painter's algorithm

## Key Concepts

### File Reader API

JavaScript can read local files using `FileReader`:
```javascript
const reader = new FileReader();
reader.onload = e => {
  const data = e.target.result;
  // usa i dati
};
reader.readAsText(file);
```

### Vertex Indexing

Instead of duplicating coordinates, OBJ uses **indices**:
- Vertices are defined once
- Faces reference vertices by index
- Saves space and memory

### Triangulation

Many models have quads (4 vertices). Our parser treats them as longer arrays that the clipping system can then handle.

## Try It Out

1. Go to [editor.p5js.org](https://editor.p5js.org/)
2. Create two files: `sketch.js` and `objImporter.js`
3. Copy the code into both files
4. Download a simple .obj model (e.g. from [here](https://github.com/AlessandroBonomo28/Elegoo-TouchScreen-2.8-GFX-fun/tree/base/p5%20js%20testing/youtube/obj%20files))
5. Press play and select the .obj file
6. Watch your 3D model rendered in 3D!

Controls:
- **WASD** → movement
- **Mouse drag** → look around
- **Space/Shift** → up/down
- **H** → hide/show wireframe
- **T** → show/hide coordinates

### The full code

Here is `objImporter.js`:

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

Here is `sketch.js`:

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
