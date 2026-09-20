---
lang: en
hidden: true
lang_ref: steam-game
permalink: /en/posts/come-non-pubblicare-un-gioco-su-steam/
categories: [gamedev]
tags: [gamedev,games, builtfromscratch]
image:
  path: /assets/img/posts/dung/1.png
  alt: Dungeon Island, the title screen
---
# How NOT to publish a successful game on Steam

![Desktop View](/assets/img/posts/dung/1.png){: }
_Dungeon Island on Steam_

I started programming [Dungeon Island](https://store.steampowered.com/app/1355450/Dungeon_Island/) at the beginning of the COVID quarantine, in January 2020. As of October 28, 2021, 471 days have passed since the release on Steam and the game has finally reached version 1.0.0.

**The problem?** I did everything myself, focusing obsessively on programming and completely ignoring what players actually care about: the visual aspect and the gameplay.

![Desktop View](/assets/img/posts/dung/2.png){: }
_Dungeon Island on Steam_

## The technical obsession that nobody appreciated

I focused on complex systems that only I cared about

### Game Systems Implemented

**Procedural world generation** using Perlin noise - a sophisticated mathematical system to create natural terrains, mountains, caves, and bodies of water. I spent weeks perfecting the noise parameters and the object placement algorithms.

**Dynamic lighting system** - one of the most difficult systems to implement, with realistic lighting effects that change in real time. Technically impressive, but who notices if the game looks like it was made in Paint?

**Monster pathfinding** - complex algorithms to calculate the optimal path of enemies through obstacles and different terrains. Works perfectly. Nobody cares.

**Steam achievement synchronization** - I studied the Steam API, implemented the objective and statistics tracking system. Hours of work for a feature that 90% of players don't even notice.

**Save/load system** - managing huge amounts of data: positions and states of all objects, monsters, player character, seed for procedural generation. A masterpiece of software architecture that no one will ever see.

### Other Technical Systems (that only I cared about)

- **Day-night cycle** with monster spawning based on light levels
- **Inventory system** with complex object state management
- **Item throwing system** with physical calculations for weight, movement, and damage
- **Crafting and furnace** with recipe and combination database
- **Palm and bamboo cane farming** with environmental growth system
- **Player animation system** with skeleton and interactions with objects, weapons, and armor
- **Combat system** with damage, dodging, and critical hits
- **Health, sleep, and hunger management** that keeps track of the player's status
- **Minimap** with efficient real-time rendering
- **Respawn system** that tracks status and position after death
- **Audio management** for sound effects and music
- **Final boss** with unique mechanics and AI behavior
- **Dialog box** for interacting with signs
- **Adaptive tiling** with animations for sea and caves

{% include embed/youtube.html id='FZAc6fcgKag' %}

## The structural problems

### 1. Zero Artistic Skill

I'm not good with graphics. The game looks like a prototype made by a programmer (because it is). I ignored this problem thinking that "the technique speaks for itself". Spoiler: it doesn't.

### 2. Beginner Marketing

I made a trailer with **CapCut** using **default fonts**. Yes, you read that right. I spent months implementing complex systems and then made a trailer in half an hour with the same effort as a holiday video.

No noteworthy presskit. No launch strategy. No community building. Just: "Here's the game, buy it".

### 3. The Illusion of "If it's technically good, it will sell"

I convinced myself that technical excellence was enough. That players would appreciate the perfect pathfinding, the balanced procedural generation, the dynamic lighting system.

The reality? Players look at the screenshots, see the amateurish graphics, and move on. They never get to find out how well programmed the game is.

### 4. The killing blow

The thing that doomed the game to death was the player movement and control system, which was too complex and which I didn't test enough, taking for granted that it was comfortable and that the player should make the effort to learn it.

![Desktop View](/assets/img/posts/dung/4.jpg){: }
_Dungeon Island on Steam_

## What I should have done

**Collaborate** - Find an artist. Pay a professional for the trailer. Hire someone for marketing.

**Invest in marketing as much as in programming** - If I spent 6 months programming, I should have spent just as much time (or budget) on marketing.

**Understand my audience** - Players don't buy a game for its pathfinding. They buy it because it looks beautiful, it looks fun, and they've heard about it.

**Make a professional trailer** - Not with CapCut and default fonts. A trailer is the first impression, often the only one.

**Focus on user experience** - I myself admitted that "the UX aspect of the game didn't receive much attention" and the controls were difficult and counter-intuitive. This was a fatal mistake.

## The Technical Challenges (that nobody saw)

Ironically, the hardest problems I solved are the ones players take for granted:

**Procedural generation with Perlin noise** - Creating natural patterns requires precise mathematical tuning. I implemented complex algorithms to place objects and monsters in the generated terrain. Visual result? "Meh, looks like a mobile game".

**Save system** - Managing huge amounts of data, saving the complete state of the world, managing file systems. Works perfectly. Nobody notices until it goes wrong.

![Desktop View](/assets/img/posts/dung/3.jpg){: }
_Dungeon Island on Steam_

## Conclusion

Dungeon Island remains my pride as a programmer. I implemented complex systems, solved difficult problems. But as a commercial product? A disaster.

It was a total disaster but the result of sincere work and not of a cold AI

![Desktop View](/assets/img/posts/dung/sprites.png){: }
_Hand-drawn animations and sprites_
