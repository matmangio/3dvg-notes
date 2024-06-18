---
title: Collision handling
---

**Collision handling** constitutes the other main objective of the physics engine along with dynamics: divided in **collision detection**, i.e. *detecting when objects collide*, and **collision response**, i.e. *updating the dynamics as a result of collision*, collision handling is one of the most computationally intensive duties in video games.
Let's consider for example a scene with $$n$$ objects: this would mean having to check for a total of $$n^2$$ collisions each physics step, an inane amount even for small non-trivial values of $$n$$.

The first optimization we can do to ease this process is to categorize objects in *two types*:

- **static objects** that don't move, are usually part of the background/setting and affect other objects but aren't affected by them in return (*eg. walls, trees, mountains...*);
- **non-static objects** (also called *dynamic/movable objects*) that can move around the scene.

This categorization allows us to distinguish *two types of collisions*: **one-way collisions**, where a non-static object collides with a static object (*eg. a ball on a wall*), and **two-ways collisions**, where instead two non-static objects collide (*eg. arrow against knight*).
Finally, **two static object cannot collide**: this isn't just an optimization that aims to reduce the number of collision checks we need to perform, but a feature that makes the construction of levels easier (*eg. walls can compenetrate each other...*).

{% include responsive-image.html path="assets/images/07_TypesOfCollisions.png" alt="Table of the different types of collisions" max-width="150px" %}

This simple distinction in the types of objects greatly reduces the amount of needed computation: if we had 50% static objects and 50% movable objects then 1/4 of the potential static-to-static collisions would cease to exist, 2/3 of the actual collisions would be one-way, which are easier to handle and less computationally impacting, and only 1/3 would be two-ways.

Apart from this simple optimization, collision handling is a much more complex topic.
Let's now start by discussing what the physics engine should do when a collision happens and work our way backwards to understand how collisions are detected in the first place.

## Collision response

How do we respond to a collision?
Apart from the ad-hoc gameplay effects which are regulated by scripts (*eg. enemy loses HP because we hit it with an arrow*), **collision response** can be divided in a series of sub-tasks that we now explore in the following paragraphs: some of these will need to be applied each step, like for example *enforcing non-penetration*, others only on the step where collisions occur, like *impacts*, and others with entirely different conditions, like *frictions* that need to be applied starting from the second consecutive step of collision.

### Enforcing non-penetration

To maintain a certain plausibility in our physics simulation we must make sure that rigid objects **do not compenetrate**.
But what do we do if in a certain physics step the trajectories of two objects would place them one inside the other?
We have two options:

- revert back to the **last valid position**, which is very easy to do but isn't at all ideal;
- project the object to the **closest valid position**, i.e. the last valid position it would have reached.

{% include responsive-image.html path="assets/images/07_Compenetration.png" alt="Visualization of the closest valid position approach" max-width="320px" %}

As we can see the second option is the most plausible one, making the objects move up to a state where they touch.
Using Position Based Dynamics this behaviour emerges by the introduction of a new *positional constraint* in the form of an inequality ($$\leq$$): in particular we define the planes that constitute the faces of the objects and ask that all other particles are on the "right" semi-space defined by the plane.
This approach has the advantage of automatically update the velocity to reflect the impact: however, usually that is not enough, as if we want to make the object bounce on one another we will need to do so by **adding impulses** to simulate the impact.
If rigidbodies are instead simulated in another way we can use different techniques and heuristics to detect compenetration and react to it.

How do we enforce this non-penetration, however?
It depends on whether the collision is two-ways or one-way: if the former we *displace both objects minimizing the sum of the squared displacements weighted by the mass*, while if the latter is the case we simply *displace only the movable object by the minimal amount* (which is theoretically equal to the first case where the mass of the static object is $$\infty$$).

### Applying frictions

Another interesting form of collision response is **contact frictions**: these are attritions that develop between objects as an effect of one sliding onto the other and as such they need to be applied only on **prolonged contact**, meaning we start applying them when two objects collide on two consecutive physics steps.
These attritions act on the component of the velocity that is *parallel to the contact plane* and can be either implemented with the introduction of appropriate **forces** or with **velocity damping**: if the former is the case then the force is *coplanar to the surface of contact*, *opposite to the current velocity* projected on that same plane and with *magnitude proportional to the speed of the object*.
Note that to calculate the velocity projection on the surface of contact we will need the normal of the collision plane.

### Impacts and impulses

By far the most common form of collision response is **impacts**, *sudden changes in velocity* that we'll need to simulate through **impulses**.
When resolving an impact what we'll need to do is then to *determine the new velocities $$\vec{v}_{new}$$* or, equivalently, *determine the impulses* $$\vec{i} = \left(\vec{v}_{new} - \vec{v}_{old}\right) \cdot m$$: we'll do whichever is easier in a certain context.
That's right: although **all impacts preserve total momentum** $$m \cdot \vec{v}$$ no matter what, which is only possible by applying *opposite impulses to the two objects* since impulses are instantaneous changes in momentum ($$\vec{i}_a = -\vec{i}_b$$), we can have different types of impacts based on the material and surface properties of the two colliding objects:

- **elastic impacts**, where no energy is lost;
- **inelastic impacts**, where some energy is lost and after the two objects move at the same speed.

Some clarifications are needed on this point: first, the energy loss we refer to has to do with the two objects getting damaged and both heat and sound being produced on impact.
Secondly, the classification of impact is not binary, but rather a *spectrum of **bounciness***: based on the properties of the two objects the impact will behave more or less like one of the two elastic/inelastic extremes.
Note that this bounciness has nothing to do with the real world, it is simply a useful abstraction for the properties of different objects under impacts in games: we're only aiming at plausibility here.

{% include responsive-image.html path="assets/images/07_Bounciness.png" alt="Different types of objects colliding and their bounciness" max-width="320px" %}

If the two colliding objects don't share the same bounciness value, game engines often compute a formula to "mix" the two values together: this can be either the *average* of the two, the *maximum* value or the *minimum* value.
The choice depends on the game engine and is sometimes exposed to the designers, which can then manually regulate how the impacts between objects work.
That said, let's now see how to solve the impact in the two extreme cases, the completely elastic and completely inelastic impacts: what we'll do in the intermediate impacts is solve both and **interpolate the resulting velocities** of these two cases *using the bounciness value as the weight for the interpolation*.

#### Inelastic impacts

Let's start by the simpler case, the one for completely **inelastic impacts**: in this case we assume that after the impact *the two objects share the same velocity*, as if the inelastic impact temporarily glued them together (they can and most probably will drift apart in subsequent steps); we also assumed that the *total momentum of the system is kept constant*.
So if we call $$\vec{v}_a$$ and $$\vec{v}_b$$ the velocities of the two objects before the impact, $$m_a$$ and $$m_b$$ their masses and $$\vec{w}$$ their shared velocity after the impact we want:

$$
m_a\vec{v}_a + m_b\vec{v}_b = m_a\color{blue}{\vec{w}} + m_b\color{blue}{\vec{w}} \\
m_a\vec{v}_a + m_b\vec{v}_b = (m_a + m_b)\color{blue}{\vec{w}} \\
\color{blue}{\vec{w}} = \frac{m_a\vec{v}_a + m_b\vec{v}_b}{m_a + m_b}
$$

As we can see, to solve the inelastic case we don't need to compute any additional information about the impact, we only need the velocities and masses of the two particles.
Notice how *if the masses are equal* then the new velocity is just the vector average of the pre-impact velocities.

#### Elastic impacts

The completely **elastic impact** is more difficult to solve: for it we assume that *both the momentum and the total energy are kept constant* after the impact.
With energy we intend the **kinetic energy**, which for a particle of mass $$m$$ and velocity $$\vec{v}$$ is computed as:

$$\frac{1}{2}m\|\vec{v}\|^2$$

These two assumptions are not enough for 3D impacts, however, since if we put the two equations in system by calling $$m_a$$ and $$m_b$$ the masses of the two objects, $$\vec{v}_a$$ and $$\vec{v}_b$$ their velocities before the impact and $$\vec{w}_a = \vec{w}_b$$ their velocities after the impact we would get a system of 4 equations in 6 variables (the $$xyz$$ components of the velocity vectors after the impact):

$$
\begin{cases}
m_a\vec{v}_a + m_b\vec{v}_b = m_a\color{blue}{\vec{w}_a} + m_b\color{blue}{\vec{w}_b} \\
\frac{1}{2}m_a\|\vec{v}_a\|^2 + \frac{1}{2}m_b\|\vec{v}_b\|^2 = \frac{1}{2}m_a\|\color{blue}{\vec{w}_a}\|^2 + \frac{1}{2}m_b\|\color{blue}{\vec{w}_b}\|^2
\end{cases}
$$

We instead need to operate an additional assumption.
To find this new assumption we first solve the problem in a simplified environment: *say we are in 1D*, then necessarily *all velocities lie on the same line* and therefore are only defined by a scalar rather than a vector.
In this simple case, if we call $$i_a$$ and $$i_b$$ the impulses to which the particles are subject (scalars), we can solve the system:

$$
\begin{cases}
w_a = v_a + \frac{\color{blue}{i_a}}{m_a} \\
w_b = v_b + \frac{\color{blue}{i_b}}{m_b} \\
\color{blue}{i_b} = -\color{blue}{i_a} \\
\frac{1}{2}m_a v_a^2 + \frac{1}{2}m_b v_b ^2 = \frac{1}{2}m_a w_a^2 + \frac{1}{2}m_b w_b^2
\end{cases}
$$

Substituting in the last equation we get:

$$
\begin{align*}
\frac{1}{2}m_a v_a^2 + \frac{1}{2}m_b v_b ^2 &= \frac{1}{2}m_a \left(v_a + \frac{\color{blue}{i_a}}{m_a}\right)^2 + \frac{1}{2}m_b \left(v_b + \frac{\color{blue}{i_b}}{m_b}\right)^2 \\
m_a v_a^2 + m_b v_b^2 &= m_a v_a^2 + \frac{\color{blue}{i_a}^2}{m_a} + 2v_a \color{blue}{i_a} + m_b v_b^2 + \frac{\color{blue}{i_b}^2}{m_b} + 2v_b \color{blue}{i_b} \\
0 &= \frac{\color{blue}{i_a}^2}{m_a} + 2v_a \color{blue}{i_a} + \frac{(-\color{blue}{i_a})^2}{m_b} + 2v_b (-\color{blue}{i_a}) \\
0 &= \color{blue}{i_a}^2 \frac{m_a + m_b}{m_a m_b} + \color{blue}{i_a}2(v_a - v_b) \\
0 &= \color{blue}{i_a} \left( \color{blue}{i_a} \frac{m_a + m_b}{m_a m_b} + 2(v_a - v_b) \right)
\end{align*}
$$

We therefore have two possible solutions:

$$
\color{blue}{i_a} = 0 \quad \vee \quad \color{blue}{i_a} = \frac{2m_a m_b}{m_a + m_b}(v_b - v_a)
$$

The first solution indicates the state before the impact, which of course satisfies all of our assumptions, while the second one the state after the impact: we will use this one to apply the impact to the particles.
Notice how *if the masses of the two particles are equal* they will simply *swap velocities*.

How do we use this simple solution in 3D?
We do it by introducing a new assumption on the impact, that is that there exists an **impact plane** and the **impulses are orthogonal to this plane**, of which we only need to know the *normal $$\hat{n}$$*.
Using this assumption we can then solve the elastic impact in 3D as:

1. Find the *scalar velocities* $$v_a, v_b$$ by *projecting the velocity vectors onto $$\hat{n}$$ using the dot product*;

    $$v_a = \vec{v}_a \cdot \hat{n}$$

2. Find the *scalar impulses* $$i_a, i_b$$ using the 1D case;

    $$i_a = \frac{2m_a m_b}{m_a + m_b}(v_b - v_a)$$

3. Extract the *vector impulses* $$\vec{i}_a, \vec{i}_b$$ multiplying the scalar ones by the normal $$\hat{n}$$;

    $$\vec{i}_a = i_a \hat{n}$$

4. Apply the impulses to the find the new *vector velocities* $$\vec{w}_a, \vec{w}_b$$;

    $$\vec{w}_a = \vec{v}_a + \frac{i_a}{m_a}$$

So as we can see the impact only affects the component of the velocity that lies orthogonal to the impact plane, which is plausible enough and will work within the simulation.
Notice how we used this decomposition of the velocity vector along the orthogonal and parallel direction of an impact plane for both impacts and frictions, so we now offer a recap of how these components are calculated:

$$
\vec{v}_n = (\vec{v} \cdot \hat{n})\hat{n} \\
\vec{v}_p = \vec{v} - \vec{v}_n = \vec{v}_n \times \vec{v} \times \vec{v}_n
$$

{% include responsive-image.html path="assets/images/07_Scomposition.png" alt="Decomposition of the velocity vector along an impact plane" max-width="320px" %}

#### Impacts with static objects

What happens with impacts when we have a **one-way** collision, meaning one of the two objects is **static**?
If we consider the static object to have infinite mass ($$m = \infty$$) and null velocity ($$\vec{v} = \vec{0}$$), then the following behaviour emerges:

- for completely **inelastic impacts** the *movable object just **stops*** ($$\vec{w} = \vec{0}$$);
- for completely **elastic impacts** the *component of the velocity of the movable object that is orthogonal to the impact plane just **flips sign*** ($$\vec{w}_a = \vec{w}_p - \vec{w}_n$$).

{% include responsive-image.html path="assets/images/07_ImpactsWithStatic.png" alt="Representation of one-way impacts" max-width="320px" %}

#### Impacts between rigidbodies

Up until now we've discussed only of impacts between particles: how do things change when it comes to **rigidbodies**?
These types of objects add *angular velocities* to the mix, for which we need to do exactly the same things we did for velocities: we always *preserve angular momentum*, need to *make angular velocities match for inelastic impacts* and *preserve kinetic angular energy for elastic impacts*, using bounciness to interpolate between the two extremes exactly like before.
Notice how all of this is emerging behaviour when using PBD to simulate rigidbodies, so we don't need to change anything; when instead we explicitly simulate rigidbodies we need to also compute post-impact angular velocities.

## Collision detection

We've discussed how to react in case of collision, but how do we **detect** if any collision occurred in a certain physics frame?
We need to find out **if any two objects overlap** and if so we also need to compute the so-called **collision data**, that is the *hit positions* and the *normal of the collision plane*: this data will then be passed to the collision response mechanisms for use.

{% include responsive-image.html path="assets/images/07_CollisionFlow.png" alt="Flowchart of collision detection" max-width="220px" %}

As we've said before, the main concern of collision detection is one of **efficiency**: we absolutely do not want to test $$n^2$$ possible collisions, since this would swamp the entire physics engine.
It can also be empirically demonstrated that *almost 100% of the times almost 100% of the object pairs **don't collide***: collisions are quite rare, so we would like a system that is able to efficiently perform **early rejection**, that is rapidly discard pairs of objects that don't collide.
This optimization for the "no collision" case is twofold, in that it involves two different types of optimizations:

- how to efficiently **test for collisions between two objects**;
- how to **avoid the quadratic explosion of collision tests**.

In the following paragraphs we first explore efficient collision tests and then how to avoid testing most pairs of objects, only performing collision tests when absolutely necessary.

### Collision tests

To test if two objects collide we first need two data structures that describe the **shape** of the objects, otherwise we cannot know if they actually compenetrate.
However, the 3D meshes that graphically represent the objects are far too complex for this: testing a collision between two of them would mean testing if each one of their vertices is inside the other shape, which especially for high-res models is unfeasible.
We instead introduce **geometric proxies**, *simple shapes or collections of simple shapes that act as crude approximations of the meshes for the purpose of collision detection*: this way, for example, a knight can be represented by a simple set of cylinders that are far easier to test for collisions.

{% include responsive-image.html path="assets/images/07_GeometricProxies.png" alt="A very complex mesh beside its geometric proxy" max-width="250px" %}

Geometric proxies can then be categorized based on their use as:

- **Bounding volume**: an *upper bound on the total extension of the object*, whose mesh must be **all contained inside**.
    These types of proxies are used for *conservative* collision tests to see if any part of the two objects *could* overlap: if the bounding volumes don't collide then the objects can't collide.
    Most of the time, a good enough bounding volume for a mesh can be *automatically computed* with an **algorithm**: it may not be the smallest possible or the most optimal but it will still allow for significant optimizations.

- **Collider** (*or "hit-box", or "collision proxy"*): an *approximation of the object's spatial extension* that doesn't necessarily contain the whole mesh.
    These are used to represent the actual approximation of the mesh that we want to use for collision and are involved in *more accurate* collision tests: if two colliders overlap then we do have a collision and we need to calculate the collision data.
    Because they need to approximate the mesh these are far more complex to automatically construct, especially if we want a simpler, more efficient proxy and a good approximation: for this reason colliders are often *hand-crafted by digital artists* and are thus **assets** in all regards.

{% include responsive-image.html path="assets/images/07_BVandCollider.png" alt="A visual representation of the difference between bounding volume and collider" max-width="250px" %}

So often what games do is use a **simple bounding volume with inside a more complex collider** to represent the object: this allows us to test for collision first on the bounding volume, which is usually easier, and then on the more complicated collider that actually approximates the mesh.

{% include responsive-image.html path="assets/images/07_EarlyRejection.png" alt="Flowchart for using the BV for early rejection" max-width="300px" %}

Note that geometric proxies aren't used only for collision detection, quite the contrary: *most any task in the game engine except rendering sees the object as its geometric proxy*.
This of course includes the physics engine, which uses it to calculate the collision data and to compute the barycenter and moment-of-inertia matrix for rigidbodies, but also the rendering optimizer (view frustum and occlusion culling use the bounding volumes), the AI system (colliders used for visibility and navigation), the sound engine (3D sound absorption) and many more other systems.

#### Types of geometric proxies

We now explore the **types of geometric proxies** most common in modern 3D video games: not every game engine will implement all of them since for every pair of proxies we will need an *algorithm implementation* to test their intersection; the number of algorithm to provide is thus *quadratic* in the number of available proxies, so some selection must be done.
Speaking of which, how do we choose which proxies to include?
A number of parameters need to be taken into consideration:

- The **workload needed to create them** either by artists or by automatic algorithms;
- The **space needed to store them** in RAM;
- How they behave with **transformations**, i.e. if they're easy to update when the mesh is transformed;
- How efficient **intersection tests** between them and other proxies, points and rays are and how easy it is to compute the relative **collision data**;
- How good the geometric **approximation** is: this can mean different things depending on which use the geometric proxy is needed for.
    In particular:

    - for *bounding volumes* we care how **tight** the proxy is;
    - for *colliders* we care how **close** the proxy is to the actual mesh;

Given these metrics for evaluating geometric proxies let's now delve into the different types of geometric proxies used in games, starting from the simpler ones and gradually turning to the most complex:

- **Sphere**: one of the simplest proxy shapes, the sphere is exceptionally easy to compute as a boundary (*provided we don't care to get the optimal one*), very small to store as it's just a center and radius for a total of 4 scalars, trivial to test for collisions with other shapes (*we just find the closest point and test its distance with the center against the radius*) and easy to transform.
    The problem with spheres is that they **don't provide very good approximations of most shapes**: they can easily model a head, but what about a castle, a dragon or a sword?

{% include responsive-image.html path="assets/images/07_GPsphere.png" alt="Sphere geometric proxy" max-width="100px" %}

- **Capsule**: resembling a cylinder with two half-spheres attached at its ends, the capsule is really a *generalization of a sphere* where instead of being the collection of points with distance from a point (center) $$\leq$$ than the radius, it is the collection of points with distance from a *segment* $$\leq$$ than the radius.
    Stored as simply the end points of the segment and the radius for a total of 7 scalars, which is still very little, the capsule has the same useful geometric properties of the sphere and is similarly easy to test, but is also capable of representing a **wider range of shapes** fairly well.
    That's why it is a very common choice in games, which often use it to represent characters.

{% include responsive-image.html path="assets/images/07_GPcapsule.png" alt="Capsule geometric proxy" max-width="100px" %}

- **Half-space**: this trivial geometric proxy constituted of a simple infinite plane that cuts the space in half is surprisingly useful when trying to model flat terrains, walls or invisible "force fields".
    The plane can be represented either as a random point on it and the normal of the plane or as the same *normal and the distance of the plane from the origin* (*which is just the dot product of the normal and the point*), thus cutting the total size to only 4 scalars: notice that in this case the impact normal is given and doesn't need to be computed.
    Half-spaces are easy to test with and transform, but unfortunately they are **only useful in specific situations**.

{% include responsive-image.html path="assets/images/07_GPhalfspace.png" alt="Half-space geometric proxy" max-width="100px" %}

- **Axis-Aligned Bounding Box** (**AABB**): typically used for *bounding volumes* rather than colliders, as the name implies, axis-aligned bounding boxes are simply 3D boxes that have their faces aligned with the $$xyz$$ axes.
    They *contain the whole mesh* and since they're aligned with the axes can just be represented by the cartesian product of **3 intervals**, a minimum and a maximum on each axis for a total of 6 scalars (*which are computed simply by finding the minimum and maximum coordinates of the vertices on each one of the axes*):

    $$[min_x, max_x] \times [min_y, max_y] \times [min_z, max_z]$$

    The resulting boxes are very easy to store, compute optimally and test against points and other axis-aligned bounding boxes.
    The problem comes with transformations: while scaling and translating work fine, **when the object is rotated the AABB will need to grow or shrink** and in general will need to be re-computed completely.
    For this reason this proxy is typically used for static objects.

{% include responsive-image.html path="assets/images/07_GPaabb.png" alt="AABB geometric proxy" max-width="150px" %}

- **Oriented Bounding Box** (**OBB**): having seen the limitations of AABBs, oriented bounding boxes ease the transform process by representing simple boxes that can have an **orientation** in space.
    They're stored as an axis-aligned bounding box and a rotation defining the box's orientation: this way this *generalized bounding box* can be freely transformed and doesn't have any problem with rotation.
    The introduction of the orientation make intersection tests a little bit more difficult but still relatively easy: to test with a point, for example, we can simply rotate the point using the inverse rotation to transport it in the boxes "local space" where it is axis aligned and then confront its coordinates with the intervals defining the box.

{% include responsive-image.html path="assets/images/07_GPobb.png" alt="OBB geometric proxy" max-width="150px" %}

- **Polytope** (or **Convex Polyhedron**): similarly to how in 2D we can define a convex polygon as the intersection of a number of half-planes equal to its number of sides, in 3D we can define a polytope as the **intersection of a number of half-spaces** equal to its number of faces.
    Stored exactly as a collection of planes defining half-spaces, each one of them requiring 4 scalars, polytopes are feasible geometric proxies as long as the face count remains reasonable: on the other hand, they are very **flexible** and can offer **great approximations** for a large number of objects; testing intersections with them is also relatively easy, since a point is inside the polytope if and only if it is inside all the half-spaces that define it.
    The only problem with this proxy is that, because of they way it's defined, it is totally **incapable of representing concave objects**.

{% include responsive-image.html path="assets/images/07_GPpolytope.png" alt="Polytope geometric proxy" max-width="100px" %}

- **Polyhedron**: the most general of all geometric proxies, polyhedrons are basically **extremely simple meshes** that can represent any kind of objects, concave ones included.
    Mainly used for *colliders* since they're far too accurate for bounding volumes, generic polyhedrons offer the **best approximations** but are the **most expensive** to store and perform intersection tests with: to check for collisions we in fact need specific algorithms and data structures (*eg. BSP-trees, see later*).
    These "special meshes" are usually hand-designed by artists to create the best approximation with the smallest number of polygons, but there exists a lot of automatic algorithms that try to simplify complex meshes (like the ones used for rendering) into polyhedrons.

{% include responsive-image.html path="assets/images/07_GPpolyhedron.png" alt="Polyhedron geometric proxy" max-width="120px" %}

- **Composite proxies**: not a real proxy type in and on itself, composite proxies are **collections of other types of proxies** that are used a single proxy to better approximate the underlying object.
    They are very expressive, as for example the union of different convex proxies can be concave, and they're both still quite efficient to store and easy to test: a collision happens if the other objects collides with *any* sub-proxy.
    Unfortunately composite proxies are *difficult to construct automatically*.

{% include responsive-image.html path="assets/images/07_GPcomposite.png" alt="Composite geometric proxy" max-width="90px" %}

#### BSP-trees for polyhedrons

We've talked about polyhedrons and identified them as the most accurate and expensive colliders, citing the fact that we would need an appropriate data structure to store them.
We now explore one such data structure, the **Binary Spatial Partition tree** or **BSP-tree** for short: this is a *binary tree* where each node represents a certain portion of space and its children represent a *partition of that space by an arbitrary plane*.
The root represents the whole space and *for all internal nodes we record the plane that splits the parent's space* as a normal and a distance (4 scalars), while the leaf nodes are just one bit: they either represent that the half-space identified by the transition that led to them is outside the proxy or inside the proxy.
An image will surely clear things up.

{% include responsive-image.html path="assets/images/07_BspTree.png" alt="Representation of a 2D BSP-tree that encodes a polygon" max-width="400px" %}

Such trees are precomputed for the polyhedrons we want to represent and, when constructed well, can serve as *"decision trees" to test if a point is inside the proxy*: we simply traverse the tree starting from the root and asking ourselves at every node if the point is in front or behind the plane.
Of course many BSP-trees can describe the same polyhedron, so when constructing we would try to find the best one, i.e. the one where the path to each possible leaf is the shortest.

How do we test for intersections of BSP-trees with other objects?
Things become a little more complex in this case, but generally what we can do is follow the tree structure and test with the planes representing each node: this is of course a little expensive, but some object can really benefit from the great approximations that polyhedrons offer.
When testing for intersection between BSP-trees, however, things become really quite more difficult: we first need a *preprocessing step* to find all edges of both polyhedra and a second step where we actually *test all edges of one polyhedron against the other* and viceversa.
Since the preprocessing can take up quite some time it is usually done once and then the edges are stored in an appropriate data structure.

It is now important to stress that *meshes used for geometric proxies are very different from the ones used for rendering*!
For once they're much more *lower res* ($$\leq 10^2$$ faces), have *no attributes* (no color, UV mapping etc), can be made of *generic polygons* and not just triangles, are necessarily *water tight* (they don't have holes) and are stored using *different data structures*: in particular, we use a polytope (set of bounding planes) if the object is convex and a BSP-tree if it's concave.

### Collision detection strategies

A question may arise: when do we perform collision detection?
Intuitively we may set it as a part of the physics step and we would be right: in particular we need to check for collision **before applying dynamics** to the objects, since both the forces/impulses and positional constraints (if we're using PBD) will need to be influenced by collision response.

{% include responsive-image.html path="assets/images/07_CollisionDetection.png" alt="Flowchart for collision detection" max-width="350px" %}

Now, there are actually **two strategies** for how to perform collision detection that have to do with *which states* of the system are checked for collisions.
Let's explore them in more detail:

- **Static Collision Detection** (*or **Discrete**, or "a posteriori"*): with this system we *only check for collision at discrete time steps after each physics step*, only testing the final positions of objects.
    This system is very **simple and quick** but it also has many problems: first, *non-penetration constraints will be violated* for a single physics step since we have to wait for the next one to patch them in collision response, which is not always easy.
    More importantly, however, the system doesn't check for collisions that happened *in-between steps*, creating an effect called **tunneling** where an object can pass through another if its velocity allowed it to reach the other side in a single step.
    This is most common when *$$dt$$ is too large*, the object's *speed is too large* or the *objects are too thin* and it actually is quite a problem: when one of these conditions happen it may be best to not consider the objects particles and use a totally different system for physics.

{% include responsive-image.html path="assets/images/07_StaticCD.png" alt="Static Collision Detection example" max-width="450px" %}

- **Dynamic Collision Detection** (*or **Continuous**, or "a priori"*): this **more accurate** strategy tries to fix the problems of discrete collision detection by testing for collision in *all intermediate states*, testing the moving objects while they move to always keep them in a safe state.
    This has the added bonus of not requiring the search for a safe state after penetration since we never allow penetrations to happen, stopping the objects in contact, but is of course much more **resource consuming**: for one-way collision we basically have to check the collisions between the static object and the **volume swept** by the moving object between its starting and final position (see figure).
    This is only doable for very simple geometric proxies like points or spheres, and it isn't at all practical with any other proxy: note that even with these simple ones we should use continuous collision detection only when it's absolutely needed because of how expensive it is.
    Moreover, for two-ways collisions things become much more difficult, since even if the volumes swept by the two objects collide they may have missed each other: we need to find a value for $$t$$ where the two proxies actually compenetrate.

{% include responsive-image.html path="assets/images/07_DynamicCD.png" alt="Dynamic Collision Detection example" max-width="200px" %}

As we can see from these descriptions, although Static collision detection is not at all perfect most games use it as the default because of how quicker and easier it is with respects to the other option.
Dynamic collision detection is instead more accurate but its cost makes it so that it's only used when necessary, for example for fast moving objects.

### Reducing the number of tests

So far we've seen how to detect if two certain objects collide using geometric proxies and a certain detection strategy.
The problem now is that *we don't want to test every pair of objects* in the scene, it would be too much computation: early rejection with the bounding volumes is costly in and on itself, so we don't want to even that for most pairs!
The idea is then to have a "**broad phase**" of collision detection that identifies **which pairs need testing**, meaning the pairs of objects that could potentially collide: this means significant optimization since most pairs of objects are so far away from each other that they couldn't ever collide, so we only need to test for collision the ones that are suspiciously close.
This process must of course be strictly **conservative**: we cannot ever discard two objects that do collide, while it's ok to test objects that in the end didn't collide.
There are three main classes of solutions for this problem: the ones based on *sorting the objects*, the one based on *dividing the space in portions* and the ones based on *hierarchies of bounding volumes*.
Let's now explore each of them.

#### Sorting-based algorithms

The basic idea of sorting-based algorithms is to *order the objects* based on some criteria and then find the ones that are close to each other and therefore could collide.
The most famous of these algorithms is the **Sweep and Prune** (**SAP**) method, which we now explore.

To work, the SAP algorithm first needs to quickly compute an **AABB for each object** counting its current translation and rotation: this is very easy provided that we have, for example, the transformed bounding sphere of the object (*the AABB simply found by adding or subtracting the radius from the center coordinates*).
If AABB can be easily constructed for each object we can then perform the algorithm, which works as follows:

1. **Bound**: quickly find the AABB for each collider;
2. **Sort**: considering the longest axis of the scene, sort the start and end values on that axis of all AABBs obtaining a sorted array of all starting and ending points;
3. **Sweep and Prune**: follow the sorted array from start to finish, adding each object whose interval's starting value we find to the *active set* and testing it against all other objects in the active set by *checking if their intervals also overlap on the other two axes* ("pruning"): if they do we need to check the two objects for intersection, otherwise we don't.
    Each time we find the ending value of an object's interval we remove it for the active set: we won't even test it against all following objects!

{% include responsive-image.html path="assets/images/07_SAP.png" alt="Visualization of the Sweep and Prune algorithm" max-width="350px" %}

Following this algorithm we can quickly detect which AABBs overlap on all three axes and therefore which pairs of objects could potentially collide: notice how pairs whose AABBs don't overlap on the main axis cost *zero* to test, while the ones that overlap on the main axis but not on the other ones still cost *very little* to check.
The main question is then how to *efficiently sort* values on the main axis: using an optimal sorting algorithm we can get a bound of $$O(n\log{n})$$, but we can achieve better results if each frame we just **adjust the sorting used in the previous frame**.
Since frames are very frequent the sorting won't probably change much between two consecutive ones so we can use an *incremental sorting algorithm* like quicksort or bubblesort to get a better bound of $$O(n)$$ in the average case.

This algorithm can of course still break when some objects are extremely large (*eg. a cave*) or when the objects are all very packed, going back to needing a quadratic number of tests: this isn't very common however and it won't happen in most scenes, especially if we choose the longest axis as the main one.

#### Spatial indexing structures

Spatial indexing structures follow a completely different philosophy: these are **data structures** that accelerate queries of the kind *"given this object's 3D position, **which objects are near it** (if any)?"*
There are many examples of structures of this type, but they all need to optimize the following two tasks:

- **Construction/update**: these data structures need to be easy to construct and *relatively easy to update*.
    In fact, while static objects can be placed in the structure as a pre-processing step, movable objects may need to update their position inside it after each physics step.
- **Access**: once the data structure has been updated it needs to be *extremely fast to search* since we will need to ask for the neighbours of each and every object in order to find which ones may collide with it.
    Hopefully the usage of the data structure should happen in *constant time* ($$O(1)$$).

Let's now define and explore four of these spatial indexing structures, explaining their pros and cons:

- **Regular Grid** (*or **lattice***): this data structure divides the entire space in a **3D grid of cells** all of the same size, where *each cell contains a list of pointers to all objects that are within it* or partially within it.
    The construction of the data structure follows a *"scatter" approach*, where for each object we find all the cells it touches and add a pointer to it to those cells; updating the structure is similarly pretty simple.
    Given the position of a point we then have a **simple indexing function** that returns the cell where that 3D position resides: we can therefore perform queries using a *"gather" approach*, returning the cell occupied by the point and all adjacent ones, which are the cells containing objects that may collide with the considered position.
    Given this structure we can therefore *test for collisions only the pairs of objects that **share a cell***.

    This data structure has some obvious problems.
    For starters, choosing the appropriate **cell size** is extremely difficult: if cells are too small then we occupy too much memory, whereas if cells are too large we have too many objects sharing the same cells, which kills the structure's efficiency.
    Moreover, even if cell size is chose aptly, the **memory occupancy** is still very large and is in fact *cubic with the resolution*: this is also a great waste of memory, since many cells will be empty most of the time.
    This is why **hash tables** are often used to balance efficiency with storage costs: in that case "collisions" on the hash table mean the possibility of collisions in physics.

{% include responsive-image.html path="assets/images/07_Lattice.png" alt="Visualization of the regular grid system" max-width="350px" %}

- **kD-tree**: to try and fix the problem of mostly empty cells of regular grids, kD-trees divide the space in a **hierarchical structure of sub-spaces** where each node's children are a partition of the parent and the leaves contain references to the objects contained in their respective sub-spaces.
    The process for creating and updating a kD-tree goes something like this: first we create the root node, which represents the whole space; we then **split the space along one axis** ($$x$$, $$y$$ or $$z$$), creating two subspaces which are child nodes of the root, and we continue to recursively subdivide them until *each leaf node contains (or partially contains) $$k$$ objects or a maximum depth of $$D$$ has been reached*.
    Objects are now neatly organized in a hierarchical structure that we can follow from root to leaf to uncover which objects are close to a certain 3D position or a certain object.

    There of course *many ways to implement the splits* that happen at each node in the tree: we could for example record along which axis each node splits or always keep the same order ($$x$$, then $$y$$, then $$z$$) to save memory.
    When performing the split we could also always divide the space in the middle or optimize the split point using appropriate algorithm.
    Each one of these choices, along with the parametrization of $$k$$ (usually $$1$$) and $$D$$, greatly impact on the structure's performance and must be chosen wisely, tailoring them to the scene we're representing.

{% include responsive-image.html path="assets/images/07_kDtree.png" alt="Visualization of the kD-tree" max-width="350px" %}

- **Octree** (and **Quad-tree**): structures quite similar to kD-trees, octrees and their 2D counterparts, quad-trees, recursively subdivide the space in smaller portions by **always splitting halfway across all dimensions at once** and continuing subdividing until the leaf nodes contain few enough objects or a maximum depth is reached.
    The resulting structure is a tree with *branching factor 8* (or 4 in the case of quad-trees) for each node, which visually represents a **irregular grid** where the resolution is greater in the portions of the space that contain the most objects: this way we keep the simple properties of a grid structure while avoiding the excessive memory usage.

{% include responsive-image.html path="assets/images/07_Octree.png" alt="Visualization of the world divided by a quad-tree" max-width="150px" %}

- **BSP-tree**: the same BSP-tree structure we described before for defining polyhedrons can be used as a spatial indexing structure when we remember it was just a collection of splits across planes.
    In particular, this structure greatly resembles a kD-tree: the root is the whole space, child nodes are partitions of the parent and the tree is *binary*, the only difference being that each node is split by an **arbitrary plane** stored in the node as the $$(\hat{n}_x, \hat{n}_y, \hat{n}_z, k)$$ tuple.
    The use of arbitrary planes allows us to split the scene more efficiently, for example going for a 50%-50% object split at each node or leaving *exactly one* object at the leaves: this keeps the tree more shallow and **optimizes the queries** but makes **construction and update much more difficult**.
    Because of these properties BSP-trees are usually only used for completely static scenes or to organize static objects in dynamic scenes, allowing us to build the tree once as a pre-processing step and never touch it again.

{% include responsive-image.html path="assets/images/07_BspTreeIndexing.png" alt="Visualization of the BSP-tree structure for spatial indexing" max-width="350px" %}

#### Bounding Volume Hierarchies

The spatial indexing structures detailed in the previous paragraph all had a bit of a problem: they required the construction and update of a completely new, often hierarchical data structure.
What if we instead used the hierarchy of objects defined by the **Scene Graph** in lieu of a spatially derived one?

**Bounding Volume Hierarchies** (**BVH**) employ exactly this principle to create an indexing structure that is *easy to construct and update*: the idea is to take the Scene Graph and **associate a bounding volume with each node** so that it encapsulates all the objects in its subtree.
This can be seen as a **bottom-up construction** rather than a top-down approach: first we find the bounding volumes for the single objects, then we progressively merge them based on the scene hierarchy until we reach the root, which will be associated to a bounding volume containing all the objects in the scene.
If the geometric proxies used for the bounding volumes are simple enough (eg. spheres) and we don't pretend to always find the optimal ones, this construction process that starts from the leaves and goes up to the root is *extremely efficient* and allows for quick updates to the structure (which need to be performed each frame).

{% include responsive-image.html path="assets/images/07_BVH.png" alt="Visualization of a Bounding Volume Hierarchy" max-width="350px" %}

But how do we use these bounding volume hierarchies to reduce the number of collision tests?
Easy, we operate a **top-down search** when we want to test for collision: we start by testing for intersections with the root's bounding volume; if we don't intersect with it we stop, otherwise we continue by *recursively testing all children of the root and further exploring only the ones we intersect with* in a similar fashion.
Notice how, differently from BSP-trees and other structures, using BVH we may need to explore multiple children of a node before finding the path from root to node we're actually looking for: this, along with the possibility of an excessive tree depth, may make BVH *not extremely efficient to access*.
Despite this, BVHs are extremely common and used to optimize many video game tasks, from culling to raytracing.

#### Summary

To summarize what we've said about the strategies that try and reduce the number of collision tests:

- **regular grids** offer the best access time ($$O(1)$$) and allow for parallel construction, but they very much lack on the storage front: they're either huge or require the extra cost of hashing;
- **kD-trees** and **octrees** are much more compact in ram than regular grids but they are more complex, slower to access and can't be constructed in parallel;
- **BSP-trees** offer a great performance during access while still being relatively small in size, but they're very much more complex to construct and update than any of the previous structures: for this reason they are a good candidate for the **broad phase of static parts of the scene**;
- **SAP** can take a while to construct the first time ($$O(n\log{n})$$) but they're much quicker to update during subsequent steps and can totally prune certain collision detections: for this reason they're good fit for the **broad phase of dynamic parts of the scene**;
- **BVHs** offer a fast construction and update since they take advantage of the already existing structure of the scene graph but they aren't the most efficient to access: for this reason they can be used in the **intermediate phase of dynamic parts of the scene**, an ulterior phase that happens after the broad one and tries to further diminish the number of collision tests needed.

### Physics and the GPU

In this chapter and the last one we've seen how the physics engine works: while doing so we've tried to squeeze every inch of optimization we could from our algorithms, since efficiency is extremely important in the context of physics simulations.
From this point of view the possibility of **parallelization on the GPU** seem quite appetizing as they would allow us to perform more complex computations in the same delta time.
Let's now see how parallelizable each of the tasks the physics engine needs to perform is:

- starting from **Dynamics**, computing forces and updating the position and velocity of objects (either particles or rigidbodies) is an **extremely parallelizable** task: the *structures used are very simple* and the *workflow is fixed* for each object, which means each one can be analyzed in parallel in the GPU.

- when we talk of **Constraint enforcement** things are quite similar to dynamics: although the *structures can be more complex*, the *workflow for each object is still fixed* and it doesn't usually care for other objects if we disregard collision constraints for the moment.
    This is still a **highly parallelizable** tasks if we take the right precautions.

- **Collision detection** crushes our dreams of parallelization: this task uses *non-trivial data structures and algorithms* and has a *hugely variable workflow* based on whether objects actually collide or not.
    As such it is **difficult to parallelize** well and must be mainly executed in the CPU: the problem is that *its outcomes affect the other two tasks*, slowing them down due to the need of CPU-GPU communication and the update in GPU structures that is quite inefficient in most architectures.

As we can see, collision detection ruins the possibility of parallelization of physics tasks on the GPU, affirming itself as one of the most complex and expensive duties of the physics engine.
