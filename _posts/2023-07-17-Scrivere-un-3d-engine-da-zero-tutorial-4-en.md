---
lang: en
hidden: true
lang_ref: 3d-engine-4
permalink: /en/posts/Scrivere-un-3d-engine-da-zero-tutorial-4/
categories: [tutorials,3Dengine]
tags: [tutorial, 3Dengine, p5js, builtfromscratch]
image:
  path: /assets/img/posts/3dengine/cover-4.jpg
  alt: Transformation matrices in p5.js
---
# Transformation Matrices in p5.js

{% include embed/youtube.html id='e8Et4QFrZhc' %}

## What We Do

In this tutorial we complete the 3D cube! We add **rotation**, **scale**, **translation** using matrices, and we connect the vertices with lines to see a solid cube rotating in space.

## The Code Explained

### Vector Functions

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

We calculate the length of a vector and normalize it (bringing it to length 1). It is needed to define rotation axes.

### Rotation Matrices

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

Three matrices to rotate around the X, Y, and Z axes. They use sine and cosine of the angle.

### Arbitrary Axis Rotation

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

This matrix allows rotating around **any axis** in 3D space, not just X, Y, or Z. It uses Rodrigues' rotation formula.

### Matrix Multiplication

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

Multiplies two 4x4 matrices. It is needed to combine multiple transformations into a single matrix.

### The Main Loop

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

We initialize the array for projected points and transformation parameters.

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

We create the scale and translation matrices.

```javascript
    let mat = mat4x4(translationMatrix, scaleMatrix);
    
    let axis = vec3normalize([1, 1, 1]);
    
    vertice = multiplyVectorMatrix(vertice, getRotationMatrixArbitraryAxis(axis, angleSum));
    vertice = multiplyVectorMatrix(vertice, mat);
```

- We combine translation and scale into one matrix
- We define the normalized rotation axis [1,1,1] (diagonal rotation)
- We apply first the rotation, then translation and scale

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

We project, normalize and save all points.

### Drawing the Lines of the Cube

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

We connect the vertices with colored lines:
- Front face (vertices 0-3)
- Back face (vertices 4-7)
- Connections between the two faces
- We skip lines outside the frustum (clipping)

```javascript
  angleSum += deltaTime * Math.PI/5000;
  if(angleSum >= Math.PI*2)
    angleSum = 0;
}
```

We increment the angle every frame to make the cube rotate continuously.

## Key Concepts

### Order of Transformations
The order is important! In the code: **Rotation → Translation → Projection**

If we did Translation → Rotation, the cube would rotate around a different point.

### Matrix Composition
Instead of applying each matrix separately to the vertex, we can multiply the matrices together and apply the result only once. It's more efficient!

### Clipping
The `if(zDepth >= 1)` check prevents drawing lines outside the camera view.

## Try It

Go to [editor.p5js.org](https://editor.p5js.org/) and observe the cube rotating in 3D space!

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
