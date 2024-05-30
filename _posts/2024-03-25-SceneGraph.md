---
title: Scene Graph
---

We've talked before of the idea of expressing the current state of the 3D world by **storing a transform for each object**: each object associated to a spatial location is given its transformation, which goes from *Local space* to *World space*.
Today we introduce the concepts and data structures that make this idea possible, efficient and intuitive.

## Composing transforms

Since the position, orientation and size of an object in the 3D world is determined by the transform associated with it, to change it we simply need to update the transform.
In particular, **transform composition** is a very useful operation in this sense: since transformations represent both states and *changes of state*, if we compose the original transformation $$T_{s}$$ (*$$s$$ as in state*) of a certain object with a transformation $$T_{new}$$ that moves, rotates or scales it by the amount we want we can obtain the new transform $$T'$$ describing the spatial state of the object.

There are, however, **two different ways** to perform this composition: since transform composition is not commutative the resulting transforms $$T_{s} \circ T_{new}$$ and $$T_{new} \circ T_{s}$$ are very different from each other.
To understand this we have to look at the meaning of $$T_{s}$$: this being the transformation that moves the object from Local space to World space means that *if $$T_{new}$$ is applied first it will be applied in Local space, while if it is applied second it will be applied in World space*; this **change in the reference system** will result in two very different final positions.
Let's take, for example, a transformation that moves the object by $$-2$$ units along the $$x$$ axis:

- if done in Local space this will move the object *$$2$$ units to **its** left*;

- if done in World space this will move the object *$$2$$ units west-ward* (to the **world** left);

{% include responsive-image.html path="assets\images\06_OrderOfTransforms.png" alt="Representation of how the order of transformations affects the final position of the object" %}

This difference in results is particularly evident with *scalings*, which when done in world coordinates will also move the object, and *rotations*, where the difference in the axes around which they are defined is especially evident.
However, **both methods of composing transforms are useful**: despite applying the transform in local space first is the most common, global transforms are used by systems like the physics engine which only reason in World space.

## Hierarchical scenes and the Scene Graph

Up until now we've defined a schema where each object in the scene that needs a position, be it a static/animated mesh, a camera, a collider, a spawn point etc., has its own associated transform that moves it from its own Local space to the global World space.
This structure can be imagined as a **tree** where the root is the World and each node represents an object, connected to the root by their transform.

{% include responsive-image.html path="assets\images\06_SceneGraphDepth1.png" alt="A rudimentary scene graph" max-width="350px" %}

Most often, however, we want to define scenes **hierarchically**, that is by defining relationships where objects have *sub-objects* inside them.
For example, let's consider a knight wielding a sword: the sword is not part of the knight mesh but it should move alongside with the knight and remain attached to its hand.
Achieving this effect by modifying the global transforms of the knight and the sword separately is quite tricky and requires us to apply to the sword all transformations that are applied to the knight.
A solution is to instead create a **Scene Graph**, an extension of the aforementioned scene tree where **each node has its own local space** and can have sub-nodes: each of them is connected to the node above by a **transform that expresses the object's position in its parent's space**, meaning one that goes from the local space of the object to the local space of its parent.
The root still represents the World space, that in this context it's also called **Global space**.

{% include responsive-image.html path="assets\images\06_SceneGraph.png" alt="A hierarchical scene graph" max-width="350px" %}

We thus have a kaleidoscopic structure of spaces within spaces, where we distinguish between two types of transforms associated with each node:

- The **local transform** that is stored for the node, going from the node space to it's parent's space.

- The **global transform**, a transform that defines the object's position in the World space: this one isn't actually stored anywhere, but it can instead be easily **obtained by composition** of all the local transforms **from the node to the root**.

This Scene graph approach has two benefits: first, since most often we will want to define the movement of objects with regards to their parent, the object to which they're attached, this structure allows us to only change the local transform and be done with it ($$T_L \leftarrow T_L \circ T_{new}$$).
Second and most importantly, this structure allows to easily define relationships in which one object follows another one's movements: *by changing the transform of a node we automatically apply it to its entire **subtree***.

### Moving in global space

As we've said before, the Scene Graph structure makes it very easy to update the position of a node in Local space: but what if we wanted to **move an object in Global space**?
Let's take into consideration the scene defined by the following Scene Graph:

{% include responsive-image.html path="assets\images\06_GlobalVsLocalTransforms.png" alt="An example of scene graph" max-width="350px" %}

Let's immagine, for example, that we wanted to move node $$L$$ two world units to the east (*world right*) with the following transform $$\color{red}{T} = \{ \text{Scale} = 1, \; \text{Rotation} = \text{Identity}, \; \text{Translation} = (2,0,0) \}$$.
Transform $$\color{red}{T}$$ must be applied in world space and *we can only change the local transform $$T_L$$* of node $$L$$ since we only want to affect that specific node.
How do we find the new local transform $$\color{blue}{T'_L}$$ so that by assigning $$T_L \leftarrow \color{blue}{T'_L}$$ we obtain the wanted result?

Although this operation may seem to work against the structure of the Scene Graph, it's still **easy and efficient to perform**.
Looking at the graph, the global transform of node $$L$$ before the change was:

$$T_B \circ T_E \circ T_L$$

We now want to find $$\color{blue}{T'_L}$$ so that the transform obtained by composing the original global transform we just described with $$T$$, the one expressing the two units move to the right, is equal to the one obtained by using $$\color{blue}{T'_L}$$ and the same $$T_E$$ and $$T_B$$ as before:

$$\color{red}{T} \circ T_B \circ T_E \circ T_L = T_B \circ T_E \circ \color{blue}{T'_L}$$

By multiplying each member by the inverse transforms $$T^{-1}_B$$ and $$T^{-1}_E$$ in the correct order we obtain:

$$\color{blue}{T'_L} = T^{-1}_E \circ T^{-1}_B \circ \color{red}{T} \circ T_B \circ T_E \circ T_L$$

If we isolate and group the components of the previous equation, expressing $$T_{g, p}$$ the *global transform of the parent* and remembering that **the inverse of a composite transform is the composition of the inverses in the opposite order** $${\bf \left((T_A \circ T_B)^{-1} = T_B^{-1} \circ T_A^{-1}\right)}$$ we can see that:

$$\color{blue}{T'_L} = \left( T^{-1}_{g,p} \circ \color{red}{T} \circ T_{g,p} \right) \circ T_L$$

So to obtain the correct transform we have to compose the current transform with the *composition of the global transform of the parent $$T_{g,p}$$, the transform to apply in world space $$\color{red}{T}$$ and the inverse of the global transform of the parent $$T^{-1}_{g,p}$$*.

Similarly, it is easy and efficient to outright **assign a new position in World space** to a node following a similar chain of transformations.
If we want the global transform of the node to become a certain $$\color{red}{T}$$, then we need to find a new $$\color{blue}{T'_L}$$ so that the following is true:

$$
T_B \circ T_E \circ \color{blue}{T'_L} = \color{red}{T} \\
\color{blue}{T'_L} = T^{-1}_E \circ T^{-1}_B \circ \color{red}{T}
$$

### Changing the hierarchy

Let's consider the case of an arrow flying through the air and hitting the shield of a knight: although it moved freely before we now want it to get attached to the shield, following it around.
This effect is easily obtainable through the Scene Graph: we just need to **change the parent of the node** to the node containing the shield, as shown in the picture below.

{% include responsive-image.html path="assets\images\06_ChangingParent.png" alt="A node changing parent in the scene graph" max-width="320px" %}

In the moment we operate this change, however, we don't want the object to change its global positioning (*position, orientation...*): that is, the **global transform must remain constant**.
In the example in the picture this means that we need to update the local transform of node $$L$$ to $$\color{blue}{\color{blue}{T'_L}}$$ so that:

$$
T_B \circ T_E \circ T_L = T_C \circ T_G \circ \color{blue}{\color{blue}{T'_L}} \\
\color{blue}{T'_L} = T_G^{-1} \circ T_C^{-1} \circ T_B \circ T_E \circ T_L
$$

A small note: this mechanism works *only if* no anisotropic scaling is used in the parent nodes: this is why systems like Unity will allow anisotropic scaling only in the leaves of the Scene Graph.

### The camera in the hierarchy

It is often the case that the **camera** used for rendering is also part of the Scene Graph: let's immagine for example an racing game where the camera is meant to follow the player's car.
The global transform associated to the camera is especially important, since its inverse allows objects to be moved from World Space to Camera Space, better known as **View Space**: this is the space the objects are required to be in for the purpose of rendering!
Because of its great importance, the **inverse of the camera's global transform** managed to get a specific name: the **View Transform**.

{% include responsive-image.html path="assets\images\06_Camera.png" alt="The camera as a node of the Scene Graph" max-width="350px" %}

The composition of an object's global transform and the View transform is sometimes called the **Model-View transform**, and is used during rendering to find which objects are visible from the camera's point of view.
It is also sometimes useful to move objects in View space: the procedure is the same as always, involving finding the new local transformation $$T'_L$$ that obtains the required result.

This process of defining nodes in the space defined by another node is very useful in many cases: for example, when computing the distance of a sound receiver from a sound emitter for determining the volume of some music.
We need to be careful, however, since *any 3D computation must use points, vectors and versors defined in the **same space***: for example, physics computations are often done in World space, while lighting is most often performed in Local space or Tangent space.

## Shared subtrees and multi-instancing

It is often the case that the same exact model must be used multiple times inside a scene: in the example above, the same exact wheel model is used four times for each car.
Not only that: if cars are all the same, it could be that the same exact *subtree* may be used multiple times in the scene.
Existing implementations of the Scene Graph data structure allow for a technique where **subtrees can be shared among nodes**: this is the case for Prefabs in Unity and Blueprints in Unreal.

There's also the fact that if a mesh needs to be loaded in memory for each copy of the object it represents this is an obvious waste of RAM space.
Instead, a technique called **multi-instancing** is used: each node of the Scene Graph contains a **reference** to a 3D model in memory, along with their particular transforms and other properties (*eg. different materials*).
This way, different *instances* of the same object can appear at different locations in the scene at the same time but a single 3D mesh is kept in RAM.
