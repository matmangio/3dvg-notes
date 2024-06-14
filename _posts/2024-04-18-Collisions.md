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
