---
title: Introduction
---

Video games have always been the most **resource-intensive** applications running on personal computers: in this course we'll be exploring the **underlying technologies and techniques** that constitute the building blocks of every **3D video game**.
We'll see how under the apparent variety of game engines and frameworks lay the same fundamental algorithms and representations.

The same was true in the 2D era of video games.
Despite the huge number of different platforms and systems, a handful of techniques were so widespread to become industry standards: this included *sprites* for representing objects on screen, *tilesets* and *tilemaps* for easily constructing scenes and compressing tile data, *blitting* for transferring sprite data on screen, *bitmaps* to associate color and semantic information to pixels and so many more.
Where 2D games used sprites and tilemaps, 3D games use models and scenes but the landscape hasn't changed much in terms of the uniformity of technology.
We'll also see how in 3D video game development some techniques are counterintuitively **easier and cheaper than their 2D counterpart**: a prime example is *animation*, where instead of painstakingly drawing each frame by hand on a sprite, in 3D we can just interpolate the position of points on a mesh between two "keyframes".
It's important to specify that here we're talking about 3D technology, regardless of the game type or point-of-view: while some modern games may choose a 2D side view as a stylistic choice or to adhere to a particular sub-genre, if the technology they use is 3D we'll consider them 3D games.

{% include responsive-image.html path="assets/images/01_TechVsNature.png" alt="Examples of the difference between tech and nature" caption="Examples of 2D games being 3D in nature and 3D games being 2D in nature" %}

Nowadays, much of these fundamental techniques are abstracted and offered as black-boxes by **high-level game engines** like Unreal, Unity or Godot.
Nonetheless, understanding what is going on under the hood is key to become a proficient user of these frameworks and a better game developer.

## 3D game engines

Let's then start by discussing what a game engine *actually does*.
Modern high-level game engines, with their ever so increasing array of features, are basically software suites that are responsible for the following aspects of a game:

- Handling of the **3D Scene** and of the objects that populate it;
- **Graphics rendering**, along with lighting, texturing, materials etc.;
- **Physics** simulation and collision detection;
- **Animations**, be them scripted or computed in real-time;
- **Artificial Intelligence**, navigation and behaviours;
- **3D Sound** rendering and mixing;
- **Networking** for multiplayer capabilities;
- **Input** and input devices handling;
- **Scripting**, allowing programmers to create the game's logic;
- **Asset management**, loading and unloading assets to and from memory;
- **Memory management** in general;
- **Localization** support for multiple languages;
- **Events** and timers to allow objects to communicate between one another;
- **GUI** and interfaces to ease the development process;
- ...

We'll only have time to cover the main elements among these, but the sheer scale of tasks game engines have to fulfill (*thus freeing the game developers from having to worry about them*) suggests how powerful these tools really are.
Nowadays it's increasingly more and more rare to see a game developed from scratch and not using Unreal, Unity or another commercial game engine: even AAA studios, which previously developed using their own (usually code-only) game engines are now switching to external ones to streamline the development process and take advantage of the features and optimizations they offer.
In this new landscape the role of the **technical staff**, capable of customizing these engines at a deeper level and create other game tools to help artists and designer bring their vision to life, is becoming fundamental: it is that kind of role we should aim to fill by exploring the techniques commonly used in 3D video games and understand when and how to use them.

## Assets for 3D video games

We've mentioned game artists: they're the people in the development team who are responsible for creating the *contents* of the game.
These pieces of content, better known as **assets**, may belong to many different categories in the case of 3D video games:

- **3D data**: models, textures, materials, UVs, shaders, animations, collision objects etc.;
- **Environments**: scene graphs, levels etc.
- **Audio**: music, sound effects, ambient sounds, voice overs etc.;
- **Video**: cut-scenes, intros etc.;
- **2D Art**: backgrounds, GUI/HUD elements, sprites, fonts, screen splashes, particle effects etc.;
- **Text**: dialogue trees, messages, translations etc.;
- **Logic**: scripts, stats etc.
- ...

In this course we'll focus primarily on 3D data and environments, but each one of this assets needs to be carefully planned and manufactured by an artist to then be put into the game... or does it?

### Procedurality and baking

There is actually an alternative to creating an asset beforehand, storing it on disk and loading it when its needed, and it comes in the form of **procedural generation**: this means *creating the asset at runtime through the use of a procedure*, a function that given some parameters returns the requested asset.
There are of course pros and cons to this approach: while procedural generation allows for the creation of a potentially infinite number of *variations* of an asset, giving the developer great *flexibility* to create the asset most suited for the given situation (*eg. adding the custom player character into a cut-scene*), it requires *computational resources at runtime* to be executed.
On the other hand, an asset constructed offline by artists is usually of *higher quality* and grants them more *artistic control*, with the downside of costing *space on disk* to be stored.
It really is a question of necessity and preference, which must be evaluated on a case-by-case basis.

A middle ground can be found in the concept of **baking**: it consists in the act of *saving for good the result of a procedural generation* to be used as a pre-made asset at runtime.
In doing so we lose space on disk and flexibility, since the parameters of the generation are now frozen in place, but gain time during the execution; also, the procedure that generates the assets is now free of any computational constraint, allowing it to produce assets of higher quality.
The concept of baking is very common in game development, for example in the case of lighting: given a static source of light we can bake its effect on the similarly static objects in the scene, generating the lighting effects on them once and for all.
