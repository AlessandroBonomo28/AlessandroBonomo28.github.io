---
lang: en
hidden: true
lang_ref: 3d-engine-2
permalink: /en/posts/Scrivere-un-3d-engine-da-zero-tutorial-2/
categories: [tutorials,3Dengine]
tags: [tutorial, 3Dengine, p5js, builtfromscratch]
image:
  path: /assets/img/posts/3dengine/cover-2.jpg
  alt: 3D Projections in p5.js
---
# 3D Projections in p5.js

{% include embed/youtube.html id='oKl1viZchsk' %}

## What We Do

In this tutorial we create the 2D projection of a 3D cube. We take 8 points in three-dimensional space (the vertices of a cube) and project them onto the screen.

## The Code Explained

### The Vertices of the Cube

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

We define the 8 vertices of a cube centered at the origin. Each point has [x, y, z] coordinates.

### Setup

```javascript
function setup() {
  createCanvas(500, 400);
}
```

We create a 500x400 pixel canvas.

### Depth Calculation

```javascript
const zNear = 0.1;
const zFar = 1000;

function computeDepth(z_vertex) {
  let z = z_vertex * -(zNear + zFar)/(zNear - zFar);
  z += (2 * zNear * zFar)/(zNear - zFar);
  return z;
}
```

This function calculates the depth of the point using the near and far planes, useful for depth clipping (deciding what is visible).

### The Projection

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

For each vertex:
- We calculate the aspect ratio to keep correct proportions
- We translate the cube to the desired position (especially in z=5, in front of the camera)
- We invert y because the screen has inverted coordinates

```javascript
    let zDepth = computeDepth(z);
    
    // Proiezione prospettica
    if(z != 0) {
      x /= z;
      y /= z;
      zDepth /= z;
    }
```

We divide x and y by z: this is the **perspective projection**. Objects further away (greater z) appear smaller.

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

We convert the normalized coordinates to pixels and draw only the visible points (zDepth < 1).

## Key Concepts

### Perspective Projection
Dividing x and y by z creates the perspective effect: objects further away appear smaller, just like in reality.

### Transformations
- **Translation**: we move the cube in 3D space
- **Aspect Ratio**: we correct the proportion for non-square canvases
- **Mapping**: we convert from 3D coordinates to screen coordinates

### Depth Clipping
The test `if(zDepth < 1)` determines if a point is visible or outside the camera's frustum.

## Try It

Go to [editor.p5js.org](https://editor.p5js.org/), copy the code and observe the 8 vertices of the cube projected on the screen!

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
