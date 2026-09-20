---
lang: en
hidden: true
lang_ref: 3d-engine-3
permalink: /en/posts/Scrivere-un-3D-engine-da-zero-tutorial-3/
categories: [tutorials,3Dengine]
tags: [tutorial, 3Dengine, p5js, builtfromscratch]
image:
  path: /assets/img/posts/3dengine/cover-3.jpg
  alt: Projection matrices in p5.js
---
# Projection Matrices in p5.js

{% include embed/youtube.html id='Ngx1xuyGa_w' %}

## What We Do

In this tutorial we use **matrices** to do 3D projection. Instead of manually calculating each transformation, we use a projection matrix that does all the work at once.

## The Code Explained

### Parameters and Vertices

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

We define the camera parameters and the 8 vertices of the cube.

### The Projection Matrix

```javascript
let projectionMatrix = [
  [aspectRatio, 0, 0, 0],
  [0, -1, 0, 0],
  [0, 0, -(zFar + zNear)/(zNear - zFar), (2*zFar*zNear)/(zNear - zFar)],
  [0, 0, 1, 0]
];
```

This 4x4 matrix contains **all** the transformations:
- **Row 1**: handles x and the aspect ratio
- **Row 2**: inverts y (from -1 to 1)
- **Row 3**: calculates depth for depth clipping
- **Row 4**: copies z into the fourth component for perspective division

### Matrix-Vector Multiplication

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

This function multiplies a vector (3D vertex) by a 4x4 matrix. It's the core of the transformation: it takes a 3D point and transforms it applying all the operations of the matrix.

### The Rendering Loop

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

For each vertex:
- We transform it into homogeneous coordinates by adding 1 as the fourth component
- We apply the translation manually

```javascript
    // Applichiamo la matrice di proiezione
    let projected = multiplyVectorMatrix(vertice, projectionMatrix);
    
    let x = projected[0];
    let y = projected[1];
    let zDepth = projected[2];
    let z = projected[3];
```

The matrix-vector multiplication produces the projected point with 4 components.

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

We divide by z (perspective division), convert to pixels and draw.

## Why Matrices?

### Advantages
- **A single operation**: all transformations at once
- **Composable**: you can multiply matrices together to combine them
- **Standard**: this is how OpenGL, WebGL and all 3D engines work

### Homogeneous Coordinates
We use 4 components [x, y, z, w] instead of 3 because:
- The fourth component (w=1) allows representing translations with matrices
- After the projection, w contains z for the perspective division

## The Complete Flow

1. 3D Vertex → [x, y, z, 1]
2. Translation → moves the cube in front of the camera
3. Projection matrix → transforms everything at once
4. Perspective division → divide x, y, z by w
5. Mapping → convert to screen coordinates
6. Draw if visible

## Try It

Go to [editor.p5js.org](https://editor.p5js.org/) and see how a single matrix-vector multiplication does all the work!

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
