---
title: Particle Systems
---

**Particle effects** are a *subset of visual effects* (**vfx**) meant to represent visual elements that are *not easily described by a surface* or are *extremely dynamic*: some examples include explosions, gusts of wind, clouds, rain showers, swarms of flies, flames and so many more.
They can all be described as a **collection of individual particles** that act independently from other particles: think, for example, of the sparks coming from a fireplace or the single raindrops during a storm.

{% include responsive-image.html path="assets/images/08_Examples.png" alt="Examples of particle effects" max-width="350px" %}

As we can see from just a few examples, these effects often require a certain degree of *randomness* to be credible, so always using the same baked animation to represent them isn't going to work: we instead need a **particle system**, an entirely new game system whose objective is to *dynamically simulate the bunch of individual particles that constitute a particle effect*.
Let's see how they work.

## Simulating particles

The use of the term "particle" may remind us of the particles we used in the physics system, and we wouldn't be completely wrong: the particles of a particle system are a **simplified version of physics particles** with much simpler dynamics and collision handling.
That is because *individual particles are not important* and we're rather interested in the **collective behaviour of all particles**, whose number usually ranges from $$10$$ to $$10^6$$: it's this whole that recreates the visual and behaviour of the wanted effect.
Note that this final effect is not that important either since it's just a cosmetic addition to the game and doesn't usually affect gameplay, so we're free to use whichever approximations we please.

So, what is a particle?
As we've said, a particle represents a tiny speckle of a larger visual effect and it has a simplified **state** associated with it: this includes its *newtonian state*, i.e. its position and velocity, but may also include *orientation and angular velocity* if the case requires it along with any number of *custom fields*, such as color, size etc.
Another important characteristic of a particle is its **lifespan**, meaning the in-game time it has left to live (sometimes called TTL, "time-to-live"), as in most particle systems the **life cycle of a particle** looks something like this:

1. the particle gets **emitted**, i.e. spawned at runtime by an *emitter*;
2. the particle **evolves**, dynamically changing its state as time passes;
3. the particle gets **disposed** after its time-to-live expires, removing it from the simulation.

All of these steps take place according to some predefined criteria tailor-suited to improve the credibility of the effect we're trying to recreate.
Let's now explore the most interesting aspects of this schema.

### Emitters

**Emitters** are game objects whose purpose is, as the name implies, to emit the particles that make up a particle effect.
They can have different **shapes** which determine the *possible initial states of the particles*: for example, a plane emitter may be used for generating snowflakes that fall uniformly over the scene while a sphere emitter could be used to simulate the thrusters on a spaceship.
This shape is just a useful geometrical abstraction that allows us to visualize where particles will be produced and with which probability: common shapes are, as we said, planes and spheres, but also cones, boxes, points etc.

{% include responsive-image.html path="assets/images/08_Emitters.png" alt="Examples of different emitter shapes" max-width="350px" %}

Since these emitters need to have a position in the scene and may need to move inside it, as in the spaceship case above, they are **part of the Scene Graph** and reside in one of its nodes: this means that every emitter has both a local and global transform along with an object space of their own.
This is crucial, since as we will see shortly *the position and orientation of the emitter are the position and orientation of the particle effect*: this way, the spaceship trusters will continue to correctly produce burst flames at the tail of the main object as they move.

But how do these emitters produce particles?
At each frame an emitter creates particles according to a certain set of criterions and parameters:

- in a pseudo-random way with a chosen **probability distribution**;
- at a designated **rate** (*particles/second*), which can change over time;
- with an **initial randomized state**, often obtained by using *noise functions*: the *initial position lies randomly within the emitter's shape* and the initial velocity is also parameterized using a specific probability distribution.
    This state can be either expressed in *world space* or *object space* depending on the situation (*eg. world space for a column of smoke, object space for a rocket engine's blaze*).

{% include responsive-image.html path="assets/images/08_EmitterRatio.png" alt="Example of the production ratio of an emitter" max-width="200px" %}

The process of emitting particles is not always infinite, however, but continues for an established amount of time depending on the effect we want to recreate: it can be very short (*eg. an explosion*), medium (*eg. a blood gush from a wound*), long (*eg. a campfire's smoke*) or even undefined if the effect should be displayed indefinitely or until something happens (*eg. water from a tap*).

To record and track the state of particles created by emitters we can use a number of different data structures, each with their own pros and cons.
A common, choice, however, is to record the state of a single particle in a struct and then use a **circular array** to store references to these records: the round nature of this array means that when the structure is full each newly added value will replace the oldest one, which allows us to put an (often hardwired) *limit on the number of active particles at any given time* (eg. $$5000$$).
We can therefore simply record the indices of the first and last active particles in this array to keep track of which particles are meant to evolve and which ones are already dead.

{% include responsive-image.html path="assets/images/08_CircularArray.png" alt="Visualization of a circular array" max-height="150px" %}

### Evolution of particles

In order to evolve a particle's state we don't need an entire Verlet integration system: instead, we can employ different ***simplified techniques for dynamics*** that each suit certain effects.
In particular:

- **Analytic evolution** (**kinematics**): in the *simplest case* the evolution of the particles' state is decided *a priori* through the definition of some function $$f(t)$$ with no need for a real dynamics simulation.
    Each physics step we simply evaluate this function and update the particle's state accordingly.

    $$state(t) \leftarrow f(t)$$

- **Numeric evolution** (**kinematics**): in this case we actually determine the particle's trajectory at runtime using the current state and the delta time $$dt$$ but *without the use of forces*.
    This allows us to not be limited to "real physics", creating effects that would be impossible if actual forces where involved (*eg. water falling diagonally, puffs of smoke accelerating upward etc*).

    $$state(t + dt) \leftarrow f(state(t), dt)$$

- **Numeric evolution** (**dynamics**): here we give particles a mass and actually *employ forces* to dynamically update their state.
    This allows, for example, particles to repel or attract one another but is by far the *most complex system* of the three.

    $$state(t + dt) \leftarrow f(state(t), state_1(t), state_2(t), \dots, state_n(t), dt)$$

Similarly we can define different, increasingly complex ***techniques for collision detection***: the most simple solution is to allow **no collisions** for particles since this optimizes things and most of the time particle collision doesn't really matter (*eg. smoke going through the roof, doesn't matter if we can't see it*).
If some collisions are needed another common technique is to allow **collisions only with hardwired objects**, like for example the ground plane: the addition of these simple, predefined collisions still keeps the system relatively easy to run and parallelize.
Some times we could want **collisions with static objects**, making particles collide only with static objects using spatial indexing structures to optimize the workload; however, this is quite rare and most times won't be needed.
Even more rare are **collisions with dynamic objects** also: they greatly increase the work required to track particles and should be used only in very specific situations.
Finally, almost never do we allow **collision between particles**: it is a luxury relegated to offline physics simulation and is almost never done in games.

If we allow some form of collision for particles we can then use different ***collision response techniques***: we could just **kill the particle** that collided or **stop it** by putting its velocity to $$0$$.
Sometimes we'll want to have **ad-hoc effects** upon collision: imagine for example a black particle created by an explosion that, after colliding with something, stains the object, stays there for a while and then disappears.
In rare cases we could also perform **full one-way collisions** where we compute the elastic/inelastic/in-between impact but only apply it to the particle and not the object, even if it was dynamic; much more expansive and rare are instead **full two-ways collisions** that affect even the collided object.

### Optimizing with the GPU

As long as we don't use really complex techniques for evolving the particles' state and collision handling, running a particle system is an **extremely parallelizable** task: if simplified dynamics are used *each particle evolves on its own*; moreover, to spawn new particles we don't need to update complex data structures but we can simply *reinitialize an existing particle the new initial state*, which is a most useful approach when using a circular array.
This great degree of parallelization benefits from the use of the **GPU** as the hardware for particle calculations: we can create, evolve and render particles on the GPU itself, to the point that in some cases the particles data never leaves the GPU!

## Rendering particles

Ok, but now that we have a way to create and evolve particles in our scene, how do we render them to screen?
Depending on the effect we want there are many options, the most common one being to **render each particle individually**: this can be done using a simple *rendering primitive* for each particle, for example a point or a segment, a *small 3D model* with very few polygons (maybe textured) or an **impostor** (*or "billboard"*), a small quad centered at the particle's position, usually oriented toward the viewer and textured with 2D images (often animated with multiple frames).
This last imposter option is one of the most commonly used in games because of how easy and convenient it is.
Of course the individual aspect of each particle can be controlled in many ways: we could define the size or transparency of the impostor or decide which frames of animation to use based on specific *parameters* like the remaining time-to-live, the particle's speed, its rotation or any other factor.
Any way this is done, when rendering particles individually the final look is obtained through the *superposition of all particles*.

{% include responsive-image.html path="assets/images/08_RenderingIndividually.png" alt="Example of each particle being rendered individually" max-height="120px" %}

Of course this isn't the only option when rendering particles: instead we could render particles by **fusing them in a single 3D shape** using various techniques.
This isn't so common in games because of how expensive it is and is often instead approximated using *screen-space techniques* which work by temporarily placing a "blob" for each particle in an offscreen buffer and then computing the normals of the blobs union in screen space, finally rendering the created surface: this is ideal for liquid simulations!

{% include responsive-image.html path="assets/images/08_RenderingBlobs.png" alt="Example of rendering of fused particles" max-height="120px" %}

## Designing particle effects

As we've seen throughout this chapter, creating a particle effect involves choosing a great deal of parameters and techniques that are best suited to represent the intended visuals: the complexity of these system creates the need for a specific figure to produce these **assets**, the **effect specialist**, which stands halfway between a developer and an artist.
His role is to design both the *behaviour* of the particle system, that is the shape of the emitter, the emission rate, the evolution functions for particles and so on, and its *looks*, managing the images and animations used for impostors and/or the 3D models that represent particles along with many other aspects.

Fortunately for them there exists many **GUIs** that allow effect specialists to design particle systems *in real time*: one important aspect of these tools is that they allow the artist to immediately see how the changes they make affect the overall effect of the asset, allowing them to try different *spawning*, *evolution* and *rendering parameters* until they find the right ones.
Examples of these tools for authoring particle effects include Houdini (widely used in movies), Cascade (in Unreal), Particle Flows and many others; many systems also provide their built-in editor for particles, like Unity's Shuriken system.
Unfortunately, most of these softwares are incompatible with one another since each uses their own set of parameters, tricks and degrees of customizability: this also results in the **lack of established formats** for particle-effect assets, where Unity uses *.prefab*, Unreal the *cascade* file format and so on.
This makes it difficult for particle systems to be reused or off-sourced across different engines/systems, limiting their use in the one engine they were created in: a few attempts at interoperable formats have been done but they're still not very established.

As a final note, particle effects are *used in offline rendering too*!
A main application of particle systems is in fact 3D animated movies: here the need for simplification is less prevalent and thus more complex and accurate techniques can be used.
In particular, a prerogative of movies when it comes to particle system is the simulation of *hair*, *fur* or *grass* with them: here instead of a particle moving through space we imagine the trajectory of each particle as the shape of the individual hair/blade of grass.
The resulting effect is surprisingly convincing and allows both 3D artists to simplify their models and animators to avoid painstakingly animating each hair.
