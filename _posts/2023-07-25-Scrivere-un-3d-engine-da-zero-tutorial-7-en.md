---
lang: en
hidden: true
lang_ref: 3d-engine-7
permalink: /en/posts/Scrivere-un-3d-engine-da-zero-tutorial-7/
categories: [tutorials,3Dengine]
tags: [tutorial, 3Dengine, p5js, builtfromscratch]
image:
  path: /assets/img/posts/3dengine/cover-7.jpg
  alt: Triangle clipping in p5.js
---
# Triangle Clipping in p5.js

{% include embed/youtube.html id='4SXWeJYTCu0' %}

## What We're Doing

In this tutorial we solve one of the most complex problems in 3D graphics: **clipping**! When a triangle partially exits the screen or crosses the camera's near plane, we must \"cut\" it and create new triangles. Without clipping, triangles deform or disappear incorrectly.

## The Code Explained

### Basic Geometric Functions

```javascript
function isAboveOrOntoPlane(v, planePoint, planeNormal) {
  planeNormal = vec3normalize(planeNormal);
  const d1 = dotProduct(v, planeNormal);
  const d2 = dotProduct(planePoint, planeNormal);
  return (d1 - d2) >= 0;
}
```

Checks if a point is above (or on) a plane. A plane is defined by:
- A point on the plane (`planePoint`)
- The normal (perpendicular) to the plane (`planeNormal`)

Uses the dot product to calculate the signed distance from the plane.

### Plane Intersection

```javascript
function interceptPlane(vOrigin, vDirection, planePoint, planeNormal) {
  planeNormal = vec3normalize(planeNormal);
  vDirection = vec3normalize(vDirection);
  
  const dot1 = dotProduct(sub(planePoint, vOrigin), planeNormal);
  const dot2 = dotProduct(vDirection, planeNormal);
  
  if(dot1 == 0 && dot2 == 0) return vOrigin;
  else if(dot1 != 0 && dot2 == 0) return 'impossible';
  
  const t = dot1 / dot2;
  
  return [vOrigin[0] + vDirection[0] * t,
          vOrigin[1] + vDirection[1] * t,
          vOrigin[2] + vDirection[2] * t];
}
```

Calculates where a line (from `vOrigin` in the direction `vDirection`) intersects a plane. Uses the parametric line equation: `P = vOrigin + t * vDirection`.

### Clipping a Triangle

```javascript
function clipAgainstPlane(triToClip, planePoint, planeNormalTowardsInside) {
  let insideCount = 0;
  let outsideVertexes = [];
  let insideVertexes = [];
  
  // Classifica ogni vertice: dentro o fuori?
  for(let i = 0; i < 3; i++) {
    let x = triToClip[i * 3];
    let y = triToClip[i * 3 + 1];
    let z = triToClip[i * 3 + 2];
    
    const isInside = isAboveOrOntoPlane([x,y,z], planePoint, planeNormalTowardsInside);
    
    if(isInside) {
      insideCount++;
      insideVertexes = [[x,y,z], ...insideVertexes];
    } else {
      outsideVertexes = [[x,y,z], ...outsideVertexes];
    }
  }
```

First we classify each vertex of the triangle: is it inside or outside the plane??

#### Case 1: All Outside (insideCount = 0)

```javascript
  if(insideCount == 0) return [];
```

The triangle is completely outside → we remove it.

#### Case 2: All Inside (insideCount = 3)

```javascript
  if(insideCount == 3) return [triToClip];
```

The triangle is completely inside → we keep it as it is.

#### Case 3: One Vertex Inside (insideCount = 1)

```javascript
  if(insideCount == 1) {
    const dir1 = sub(outsideVertexes[0], insideVertexes[0]);
    const dir2 = sub(outsideVertexes[1], insideVertexes[0]);
    
    const intercept1 = interceptPlane(outsideVertexes[0], dir1, planePoint, planeNormalTowardsInside);
    const intercept2 = interceptPlane(outsideVertexes[1], dir2, planePoint, planeNormalTowardsInside);
    
    const newTri = [...insideVertexes[0], ...intercept1, ...intercept2];
    return [newTri];
  }
```

Two vertices outside, one inside → we create a **smaller triangle**:
- We keep the inner vertex
- We find where the two sides intersect the plane
- We create a new triangle with these three points

#### Case 4: Two Vertices Inside (insideCount = 2)

```javascript
  else {
    const dir1 = sub(outsideVertexes[0], insideVertexes[0]);
    const dir2 = sub(outsideVertexes[0], insideVertexes[1]);
    
    const intercept1 = interceptPlane(outsideVertexes[0], dir1, planePoint, planeNormalTowardsInside);
    const intercept2 = interceptPlane(outsideVertexes[0], dir2, planePoint, planeNormalTowardsInside);
    
    const tri1 = [...insideVertexes[0], ...insideVertexes[1], ...intercept1];
    const tri2 = [...intercept1, ...insideVertexes[1], ...intercept2];
    
    return [tri1, tri2];
  }
}
```

One vertex outside, two inside → we create a **quad (quadrilateral)** which we then divide into **two triangles**:
- We find where the sides intersect the plane
- We create two triangles that cover the area inside the plane

### Near Plane Clipping

```javascript
const pointNearPlane = [0, 0, 0.01];
const normalNearPlane = [0, 0, 1];
let clippedTriangles = clipAgainstPlane(triViewSpace, pointNearPlane, normalNearPlane);
```

After the view transformation, we clip against the camera's **near plane** (z = 0.01). This prevents triangles behind the camera from causing projection issues.

### Screen Space Clipping

```javascript
const clipPlanes = [
  [[0,0,0], [1,0,0]],         // Left
  [[0,0,0], [0,1,0]],         // Up
  [[width,0,0], [-1,0,0]],    // Right
  [[0,height,0], [0,-1,0]],   // Bottom
];

let triQueue = [projected_triangles[i]];
let newTriangles = 1;

for(let planeIndex = 0; planeIndex < clipPlanes.length; planeIndex++) {
  while(newTriangles > 0) {
    const pointPlane = clipPlanes[planeIndex][0];
    const normalPlane = clipPlanes[planeIndex][1];
    const tri = triQueue[triQueue.length-1];
    triQueue.length--;
    newTriangles--;
    
    triQueue = [...clipAgainstPlane(tri, pointPlane, normalPlane), ...triQueue];
  }
  newTriangles = triQueue.length;
}
```

After projection, we clip against the **4 screen edges**:
- **Left**: x = 0
- **Right**: x = width
- **Top**: y = 0
- **Bottom**: y = height

For each plane:
1. We take all triangles in the queue
2. We clip them one by one
3. The resulting triangles are put back in the queue
4. We move to the next plane

A triangle can be clipped multiple times, creating up to 8-9 triangles!

```javascript
for(let n = 0; n < triQueue.length; n++) {
  triangle(triQueue[n][0], triQueue[n][1],
           triQueue[n][3], triQueue[n][4],
           triQueue[n][6], triQueue[n][7]);
}
```

We draw all the triangles resulting from clipping.

## Key Concepts

### Why is Clipping Needed?

Without clipping:
- Triangles behind the camera project poorly (division by negative z!)
- Triangles outside the screen waste resources
- Visual artifacts when a triangle is partially visible

### Sutherland-Hodgman Algorithm

The algorithm used for screen-space clipping. Clips a polygon against one plane at a time, creating new vertices where necessary.

### Complete Clipping Pipeline

1. **View Space**: clipping against the near plane (z = 0.01)
2. **Projection**: perspective transformation
3. **Screen Space**: clipping against the 4 screen edges

### Clipping Cases

Each plane can generate:
- **0 triangles**: all outside
- **1 triangle**: all inside OR only 1 vertex inside
- **2 triangles**: 2 vertices inside (forms a quad)

## Try It Out

Go to [editor.p5js.org](https://editor.p5js.org/) and copy and paste the code:

- Get very close to the cube (press W)
- Watch how the triangles are cut correctly when they exit the screen
- Without clipping, you would see strange artifacts!

```javascript
const zNear= 0.1;
const zFar = 1000;
const winWidth = 400;
const winHeight = 400;
const aspectRatio = winHeight/winWidth;

let lightDirection = [0,0,1];

let cameraYaw = 0;
let cameraPitch = 0;
let playerPos = [0,0,0]

let vUp = [0,1,0];
let vRight = [1,0,0];
let vForward = [0,0,1];

let isTextVisible = true;
let isDragging = false;
let xDrag = 0,yDrag = 0;
let startX,startY;


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
  
  let projected_triangles = [];
  background(220);
  stroke('black');
  
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

      scale_x = scale_y = scale_z = 4;

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
          getRotationMatrixArbitraryAxis(axisRotation,angleSum);
      //getRotationMatrixY(angleSum);
      /*
      mat4x4(getRotationMatrixZ(angleSum),
          mat4x4(getRotationMatrixY(angleSum),getRotationMatrixX(angleSum)
            ));
      */
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
          point(x,y)
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
    //stroke(triColor) 
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
