---
lang: en
hidden: true
lang_ref: 3d-engine-1
permalink: /en/posts/Scrivere-un-3D-engine-da-zero-tutorial-1/
categories: [tutorials,3Dengine]
tags: [tutorial, 3Dengine, p5js, builtfromscratch]
image:
  path: /assets/img/posts/3dengine/cover-1.jpg
  alt: Introduction to p5.js
---
# First Steps with p5.js: Falling Circles

{% include embed/youtube.html id='zucCXzZ3UCA' %}

## What is p5.js?

p5.js is a JavaScript library for creating interactive graphics and animations. It is perfect for those who want to code creatively without too much complexity. If you go to [editor.p5js.org](https://editor.p5js.org/) you can use it online without having to download anything.

## How it Works

Each p5.js project uses two basic functions:

- **`setup()`** - runs once at the beginning
- **`draw()`** - repeats in a loop (about 60 times per second)

## Our Project

In this tutorial we create a simple animation: each mouse click generates a yellow circle that falls downwards.

### The Code Explained

```javascript
const raggio = 20;
const fallSpeed = 0.2;
let x = 0, y = 0;
let circles = [];
```

We define the radius of the circles, the falling speed and an array to store all created circles.

```javascript
function setup() {
  createCanvas(400, 400);
}
```

We create a 400x400 pixel canvas.

```javascript
function draw() {
  background(20, 190, 20);
  
  x = mouseX;
  y = mouseY;
  
  strokeWeight(5);
  point(x, y);
```

Every frame we draw a green background and a point that follows the mouse cursor.

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

We draw all the yellow circles, we make them fall and when they go out of the canvas we bring them back to the top.

```javascript
function mouseClicked() {
  circles.push([x, y]);
}
```

Each click creates a new circle.

```javascript
function keyPressed() {
  if(key === 'r') {
    circles = [];
  }
}
```

Pressing 'r' we delete all circles.

## Try It Now

Go to [editor.p5js.org](https://editor.p5js.org/), copy the code and hit play

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
