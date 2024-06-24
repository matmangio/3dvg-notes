---
title: Animations
---

**Animations** are a fundamental part of modern 3D video games: they make the world more believable and allow the player to immerse herself in the experience.
Although technically physics based movement is a procedural animation, in this chapter we'll talk of asset animations that are produced by artists.

As a first categorization we can divide animations based on how they treat the object they animate:

- **As a rigidbody**: this is the case of rigid objects or object made of rigid sub-parts.
    What we can do here is just move and rotate the object around by **animating its scene transforms** each frame to obtain the wanted effect: immagine, for example, the animation of a door opening where we just rotate it over time.
    This type of animations offer us a total of **6 degrees of freedom**, 3 for the change in position and 3 for the change in rotation (*assuming objects cannot scale, otherwise 7*).

{% include responsive-image.html path="assets/images/12_Rigidbodies.png" alt="Examples of animations that treat objects as rigidbodies" max-height="120px" %}

- **As a free-form**: here we consider the object to be *fully deformable* and able to change its shape however we want.
    We can therefore apply any **generic transformation** to the object to shape it however we want over multiple frames: think, for example, of a pufferfish puffing itself or an umbrella opening.
    By being able to arbitrarily shape the model by changing the position of its vertices we have **3 degrees of freedom per-vertex**, making these the most expressive animations.

{% include responsive-image.html path="assets/images/12_FreeForm.png" alt="Examples of free-form animations" max-height="150px" %}

- **As an articulated body**: instead of treating the model as completely malleable here we give it a bit more structure by imposing an **internal skeleton** made of joints that constraint the movement of body parts.
    Through a process called **skinning** areas of the mesh are assigned to bones in this skeleton and move with them, so we only have to *animate the skeleton* each frame to obtain the movement.
    Most video game characters are animated like this since articulated bodies offer a great tradeoff between expressiveness and efficiency, offering **3 degrees of freedom per-joint** in the skeleton (*usually we have ~25 bones, so ~50-100 DOF*).

{% include responsive-image.html path="assets/images/12_ArticulatedBodies.png" alt="Examples of characters having articulated bodies" max-height="120px" %}

As always, animations in each category can be either **authored by an artist** or **produced procedurally** with some algorithm that is usually based on physics laws: while the former approach grants *more artistic control* but is in turn *more complex* and creates *less flexible animations* that occupy RAM space since they're asset, the latter creates animations that are both *very realistic* and automatically *adapt to the environment* but are also *harder to control* in order obtain the exact desired effect and consume GPU computational power.
To better understand the difference imagine a very touching death animation as opposed to a simple ragdoll: while the first is surely more curated and able to evoke emotions, the second adapts to the scene and may for example make the body roll down the stairs. \\
The two approaches totally exclusive, however, and many modern video games *mix authored and procedural animations* to have the best of both worlds: this usually takes the form of **primary** authored animations and **secondary** procedural ones.
Going back to our death animation example, we could have the alive character be governed by authored animations and have her play an asset death animation before going into ragdoll, from which point onward the body is entirely procedurally animated.

Given this distinction between the three different types of animations and the authored vs procedural approaches we can now create a **matrix of possible animation types** where we expose the relative techniques used for each combination in 3D video games:

{% include responsive-image.html path="assets/images/12_AnimationTypes.png" alt="Matrix of possible animation types" max-height="250px" %}

Before we can start exploring some of these animation types in more detail, however, we first need to explain a *technique used by **all** asset animations*: **keyframing** and interpolation.

## Keyframes

As we know animations have to span multiple rendering frames to give the illusion of smooth transitions between states to the player: when authoring an asset animation, however, it would really be too much to ask of the animators to produce a different state for the object for each individual frame of the animation.
What we do instead is to *only store in the asset a **subset of frames*** called **keyframes**, each of which is *associated with a **time***: all other frames of the animation, called **in-betweens**, are instead *automatically **interpolated** starting from these keyframes* using time as a parameter.

{% include responsive-image.html path="assets/images/12_Keyframes.png" alt="A set of keyframes arranged on timeline" max-height="75px" %}

This not only **saves artists work** as we've said, but it also **saves RAM space** since now only a subset of frames need to be recorded in the animation asset: moreover, the animation can now automatically be **very smooth** and avoid *temporal aliasing* altogether even when played in slow motion!
This technique of course implies the ability to *interpolate keyframes*, so we'll have to make sure that the data structures representing the state in our animation types are easily interpolated.

A few things of note when using keyframing are the following: first, *keyframe distribution can be **adaptive***, meaning we don't need to keep a set amount of time between keyframes but we can instead create more only where they're needed.
Moreover, *times associated to keyframes are chosen arbitrarily*: we don't have, for example, to set them to the time of a specific video frame since all frames shown on screen will be created as in-betweens on demand during rendering anyway.
Notice also how we haven't said anything about the **interpolation schema**: this is because each technique will use their own and will also be able to use different schemas between different keyframes to create better and more suited transitions (*better interpolation schemas will need fewer keyframes*).
Lastly, keyframing opens an entirely *new way of editing animations*: we can start with a pair of keyframes for the start and end states, pick a time $$t_i$$ between them, bake the in-between as a keyframe and edit it to achieve the wanted result.
This way we only change things when they're needed, leaving the rest to interpolation.

Speaking of which, how does the **interpolation** work in general?
The most simple way for the interpolation to work is the following: given a set of keyframes $$A,B,C,D$$ distributed at the times $$t_A, t_B, t_C, t_D$$ respectively, we observe *between which frames the current time $$t$$ rests*.
If for example $$t_B \leq t \leq t_C$$ we then need to interpolate frames $$B$$ and $$C$$ using the following weights:

$$w_B = \frac{t - t_C}{t_B - t_C} \qquad w_C = (1 - w_B) = \frac{t - t_B}{t_C - t_B}$$

This provides a simple **linear interpolation** between the two keyframes.
Sometimes, however, this isn't what we want: we could want, for example, for the transition to go fast in the beginning and then slow down or viceversa, or maybe to bounce between the two states.
Fortunately all of these effects don't need to be created by hand but can instead be achieved simply by using a **transition function**, a function that *takes as input the base linear weights and produces **new weights*** for the two keyframes:

$$w_B = \color{green}{f}\left(\frac{t - t_C}{t_B - t_C}\right) \qquad w_C = (1 - w_B)$$

There exists a lot of transition functions, some of which are showed in the figure below.
Notice how in the highlighted spots of some of these functions we have weights outside the range $$[0,1]$$: we're therefore producing **extrapolations** by exaggerating one of the two keyframes, a choice that can be quite expressive but needs to be managed carefully.

{% include responsive-image.html path="assets/images/12_TransitionFunctions.png" alt="Examples of different transition functions" max-height="150px" %}

## Blend shapes

Also known with a bunch of different names that include *Morph targets*, *Shape keys*, *Face morphs*, *Vertex animations* and many more, **Blend shapes** are a sort of *3D counterpart to old 2D games animations*: in those games objects were represented by sprites and animations where done simply by changing the sprite at specific times.
The intuition is then to translate this approach to 3D by representing animations as a **sequence of meshes** since meshes are how we represent objects in 3D.

Following this idea **Blend shapes** are an extension of the indexed mesh data structure that represents **a mesh with several associated geometries**, i.e. a series of meshes (*shapes*) that have *different geometries* but *share the same connectivity, textures, UV-map and attributes* except for the ones depending on the geometry itself like normal and tangent directions.
Also known as **morph targets**, these alternate geometries can be expressed in two equivalent ways:

- **absolute encoding**: each morph target is a shape that stores its *vertex positions*;
- **relative encoding**: we have a *base shape* that expresses the starting vertex positions and each morph target stores a *vectorial difference for each vertex* that creates the new shape.

{% include responsive-image.html path="assets/images/12_BlendShapesData.png" alt="Visualization of the blend shapes data structure" max-height="250px" %}

As we can see, all morph targets will always share the same *mesh connectivity* and *surface topology*, which means we cannot change the mesh resolution or break the surface apart.
Moreover all *attributes* are kept the same apart from positions and normals, which means the object cannot change color or *textures*: if such a change is needed then it's better to animate the textures directly.

### Blending shapes

To animate blend shapes we simply **use morph targets as keyframes** and interpolate between them by *interpolating the geometries* along temporal sequences.
How this interpolation actually happens, however, depends on the way the morph targets are recorded:

- When using **absolute encoding** the morph targets are intended to be used as *alternatives* to one another: to interpolate between them we simply use the *normal rules for keyframe interpolation* making sure to have a unitary sum of weights.

    $$\sum{w_i} = 1$$

- When using **relative encoding** the shapes are intended to be *superimposed* onto a base shape with various degrees of strength.
    In order to allow for the maximum expressiveness in this case we don't require the weights of the superimpositions to sum up to one:

    $$\text{base} + \sum{w_i\text{shape}_i}$$

    This however is the equivalent of using absolute mode where the weight of the base mesh is adjusted so that all weights sum up to 1 in the end (*if negative we're implicitly extrapolating*):

    $$w_{base} = 1 - \sum{w_i}$$

{% include responsive-image.html path="assets/images/12_BlendExample.png" alt="Example of blend shapes being interpolated" max-height="100px" %}

Nothing stops us from interpolating *more than two shapes at a time*: in that case *relative encoding* is usually preferred since we would just be interpolating the displacements onto the base mesh.

### Blend shapes use cases

What are the most common use cases of blend shapes?
An extremely prominent one are **facial expressions**, a type of animation that tries to mimic realistic movements of the face of characters (*sometimes in tandem with skeletal animations*), since they benefit very much from the ability to transition smoothly between one expression and the other.
Usually expressed using relative encoding starting from a base neutral pose, facial expressions usually take also advantage of the possibility of *blending more than one shape*: we could for example have different shapes for the eyebrow movements and the mouth movement (*"face space"*) and combine them to create realistic expressions.

{% include responsive-image.html path="assets/images/12_BlendFacialExp.png" alt="Example of facial expressions over a mesh" max-height="150px" %}

Apart from **generic deformations** like the pufferfish example we cited at the start of the chapter, blend shapes are also extremely used to provide a set of **variations** for a given class of objects: think, for example, of different versions of the same RPG outfit that are adapted based on the height of the character's species.
**Character creation editors** that allow players to freely modify their avatar also fall under this category, although at the end of the process they usually bake the resulting character mesh to use more efficiently during the game.

{% include responsive-image.html path="assets/images/12_BlendVariants.png" alt="Example of variants of the same outfit" caption="Example of the same outfit being shaped to match with the character species" max-height="150px" %}

### Authoring blend shapes

How are blend shapes created?
The process is usually quite simple: first, *a base mesh is constructed* along with its UV-mapping, texturing, setting of vertex attributes etc.
Now that we have this connectivity and attributes set in stone we can then *re-edit* the mesh to create each keyframe through low-poly modelling, sculpting, subdivision surfaces and any other 3D modelling technique we could want.

{% include responsive-image.html path="assets/images/12_BlendAuthoring.png" alt="Example of a mesh being created and then re-edited to create keyframes" max-height="100px" %}

There are quite a few mesh interchange formats that support blend-shapes: we have Khronos's *.GLTF* which uses relative encoding, Collada's *.DAE*, Autodesk's *.FBX* and even old formats such as *.MD5* created by Valve.
An alternative to these formats would be to just store the meshes separately while making sure to keep connectivity consistent: this can be tricky, as both vertex and face ordering must be the same.

### Pros and cons of blend shapes

Despite all of this, Blend shapes aren't too used in games apart from the specific use cases we detailed above.
Although they're extremely **easy to use** since interpolation is just a matter of defining weights and are very **flexible** and expressive, the need to replicate the geometry of the mesh a large number of times means that they're **very expensive to store** and they cost a lot in RAM (*although it's not as bad as old sprite animations*).
Moreover, it is only the creation process that is very flexible and with a huge number of degrees of freedom: blend shapes offer very **little freedom when used** since the shapes from which and to which to blend have been predefined and cannot be changed at runtime.

However, blend shapes are pretty much an open field of research with many challenges to tackle: issues have to do with *capturing* movement and translating it into blend shapes, *compressing* the data structure to reduce its storage cost, *streaming* of morph targets at runtime and *LODing*, which is more complex than with simple meshes since the same connectivity must fit all shapes well.

## Kinematic animations

Let's now explore a totally different way of defining animations which is much easier to define than blend shapes and achieves great results with just a few keyframes: we're talking about **kinematic animations**, a type of animation that treats objects as perfectly rigid bodies.

It all starts from the *Scene Graph*: given a hierarchical structure for the scene a simple way to animate it is to **store local transformations as keyframes** for each moving part, using the same keyframing and interpolating technique seen before.
In particular, in-betweens are created by **interpolating local transformations** of the single objects and are applied by *cumulating the interpolated local transforms to obtain global transforms* for each object: this makes kinematic animations very expressive since animating a parent node will also affect all of its children.

{% include responsive-image.html path="assets/images/12_AnimatedSceneGraph.png" alt="Example of a Scene Graph animated with kinematic animations" max-height="250px" %}

This type of transformation is **very cheap** since transforms are really compact to store (*~8 scalars*) and most of the time we'll only need to store rotations as translation and scaling are usually constant across keyframes, especially for nodes that are children of other objects.
Despite being this efficient storage-wise, kinematic animations are also **surprisingly expressive**.

## Skeletal animations

The schema of animated Scene Graphs used for kinematic animations suggests an idea for **animating complex character models**: we could *split up the character's body in parts* called "bones" and organize them in a hierarchy in the Scene Graph, using local transformations to animate each bone individually.

{% include responsive-image.html path="assets/images/12_RobotGraph.png" alt="Example of a Scene Graph hierarchy representing a character" max-height="200px" %}

This would however only work for very simple objects, as when the number of bones increases a bit things become *difficult to manage*: the need to have *individual meshes for each bone* makes the structure *inefficient in terms of draw calls and storage*, along with being a nightmare to author since each body part would have to be designed separately.
Not only that, but the results look *robotic* since each body part is completely rigid and moves independently from nearby regions, which is not realistic.

The idea of **skeletal animations** is instead to use only **one mesh** for the whole character in tandem with a **skeleton** (or **rig**), a bone structure that will be linked to the mesh through a process called **skinning**: we'll basically *tell each vertex which bone (or bones) to follow* through per-vertex attributes so that by changing the skeleton's pose we automatically modify the mesh to assume the same pose.

{% include responsive-image.html path="assets/images/12_SkeletonExample.png" alt="Example of a rig used in skeletal animations" max-height="120px" %}

This approach has many advantages: first of all it only requires the use of a single mesh and a single draw call for the character, which greatly optimizes GPU performance.
Moreover, it creates **orthogonality between models and animations**: as long as they share the *same skeleton*, every skinned model can run with any animation and any animation can be applicable to any model.
This vastly reduces both the artists workload and the VRAM occupancy of the system: if we have 500 animations and 500 models we can store only 1000 things in GPU RAM instead of 500 $$\times$$ 500.

### Rigging and poses

In order to understand how skeletal animations work we first need to define what the **skeleton**, also known as **rig**, is.
Basically a skeleton is a **tree of bones**, each bone being a **vectorial space** used to express pieces of an animated model: we can have bones for the forearm, calf, individual fingers etc.
Think of a skeleton as the equivalent of a Scene Graph but for one single model, where *the root's space is by definition the model's Local Space*: this root bone is usually located around the *pelvis* or in a **virtual "basis" bone** placed in the origin of the mesh.
An usual number of bones for a character in modern video game starts from ~22-24, is optimal around 40 and very rarely goes over 100.

{% include responsive-image.html path="assets/images/12_SkeletonHierarchy.png" alt="Example of a skeleton's hierarchy of bones" max-height="220px" %}

As a data structure, a skeleton also contains a **rest pose**: this is a special "starting" pose that represents the one in which the mesh was modelled and usually features the character up-straight with its arms open to form a "T shape", which is why this is often called *T-pose* or *T-stance*.
In general, a **pose** is a data structure that contains **a local transform for each bone** $$i$$, each of which are transforms that go from the bone's space to their father's space: usually these transformations *mostly feature rotations* as bones are assumed to be fixed length and therefore their *relative translations are defined only once and for all in the skeleton's rest pose*.
In a pose each bone thus has both a *Local Transform* and a **Global Transform** that is obtained by combining the local transforms of bones up to the root and moves the bone in *"Character Space"* (just an alternative name for Object Space).

Now, because the mesh's vertices are expressed in Character Space and we've said that by definition this is equivalent to the skeleton's root's space in rest pose, to obtain the **Final Transform of a bone** that can be used to *move all vertices assigned to it to their position in a pose* we need to combine the bone's *Global transform in the pose* with the *inverse of the bone's Global transform in rest space*.

{% include responsive-image.html path="assets/images/12_RestToPose.png" alt="Visualization of the Final Transform of a bone" caption="In this case, the final transform from rest pose to pose X is the following:" max-height="260px" %}

$$T_{final} = P_2 \circ P_7 \circ P_8 \circ (R_2 \circ R_7 \circ R_8)^{-1} = P_2 \circ P_7 \circ P_8 \circ R_8^{-1} \circ R_7^{-1} \circ R_2^{-1}$$

Once we have Final Transforms for each bone the rest pose becomes useless during rendering as to obtain the actual pose we simply need to **transform vertices with the Final Transforms of the current pose**.
This aspects reflects on how poses are differently kept as a data structure in CPU and in GPU:

- in **CPU RAM** poses are **arrays of Local Transforms**, one per bone, defined for a given skeleton;

- in **GPU RAM** poses are **arrays of Final Transforms**, one per bone, defined for a given skeleton and **precomputed** before sending the data to the GPU using the method above.

{% include responsive-image.html path="assets/images/12_PoseDataStructure.png" alt="Visualization of the pose data structure in CPU and GPU RAM" max-height="200px" %}

With this in mind we can finally see that to animate a character with a given skeleton all we need is to *use poses as keyframes*.
**Skeletal animations** are thus simply **arrays of poses**, each of which is assigned a *timestamp $$dt$$* often defined as a *delta from the starting time of the animation* $$t_0$$: to make matters easier this type of animation can be *looped*, meaning the last keyframe will be interpolated with the first to restart the animation cycle (*eg. walking animation*).
This data structure has a storage cost equal to:

$$(\text{num. keyframes}) \times (\text{pose size}) = (\text{num. keyframes}) \times (\text{num. bones}) \times (\text{transform size})$$

In-between poses are simply obtained by **linearly interpolating the Local transforms of each bone**: this **works surprisingly well**, with very different keyframes being blended with good results, far better than the ones obtained with blend shapes for example.
This allows keyframes to be *very far apart*, optimizing the performance of the animation assets: a decent walk cycle can be obtained with just 4 keyframes for example and an attack animation may require just 2!

{% include responsive-image.html path="assets/images/12_SkeletonKeyframeInterpolation.png" alt="Example of the very good keyframe interpolation of skeletal animations" max-height="130px" %}

Animations can then also use some tricks to optimize storage even more: they can for example **omit a bone's transform if it doesn't change** in a certain pose, using an interpolation of the ones from the previous and next keyframes instead.
Another optimization is **allow translations only in the root bone** while all other local transforms only feature rotations (and scalings): if translation is needed a *global translation factor $$t_0$$ is added to keyframes* and applied only to the root bone.

{% include responsive-image.html path="assets/images/12_SkeletalTranslation.png" alt="Example of a skeletal animation using a global translation" max-height="200px" %}

### Skinning and animating

Up until now we've implicitly assumed each vertex of the base mesh follows *one bone*'s movement by having its *index stored as a per-vertex attribute*: this, however, doesn't allow natural movements.

The idea of **skinning** is to instead create *articulated deformable objects* by **linking each vertex to multiple bones**: also called *"blend" skinning*, this technique will then transform each vertex using an **interpolation of the Final Transforms of the linked bones** which is done using *weights for each bone* that are defined as **per-vertex attributes** of the mesh.
The addition of this multiple indices and weights transforms the mesh into a *skinned mesh*, a data structure that is heavier to store by a factor of $$(\text{bone index} + \text{weight}) \times N_{max}$$, where $$N_{max}$$ is the max number of bones vertices can be linked to.

{% include responsive-image.html path="assets/images/12_SkinnedMesh.png" alt="A mesh's skeleton and skinned mesh side-by-side" max-height="400px" %}

Usually each vertex can be linked to a number of bones *between 1 and 4*: 1 gives us the non-blended skinning effect we had earlier, 2 looks quite cheap and is commonly used for mobile games, *4 is the standard* and achieves top quality (more than 4 is almost never used in games).
But why put a cap on the number of links?
The reasons are two: first, it **reduces performance cost** by putting a bound on the number of interpolated transforms and by allowing the GPU to *avoid control instructions* meant to evaluate the number of bones a vertex is linked to, which are very much unoptimized for the hardware; instead, all vertices will interpolate between $$N_max$$ bones and if less are needed we simply put the extra weights to $$0$$.
Secondly, having a maximum number of links **reduces VRAM storage cost**: moreover, having arrays of fixed length optimizes the performance of GPU memory.
Note that when importing a mesh with skinning some game engines will allow us to *lower* $$N_{max}$$ as a preprocessing step.

So, **how does skinning actually work in the GPU**?
The task of real-time skinning is to take a *skinned mesh* and a **target pose** (the interpolated pose produced as an in-between in the CPU) and produce a **transform** $$T$$ **for each vertex** in order to "create" the deformed model to show on screen: in reality, the deformed model is *never really stored in VRAM* or anywhere really, the vertices' coordinates are simply changed at some point of the rendering pipeline to give the illusion of the model being deformed when in reality the only things stored in VRAM are the mesh in rest pose and the target pose.

{% include responsive-image.html path="assets/images/12_RealTimeSkinning.png" alt="Visualization of real time skinning" max-height="210px" %}

As we said before, in order to produce this transform $$T$$ need to transform the position and normal of vertices to put them in the current pose what we need to do is to **interpolate the Final Transforms of multiple bones** using the appropriate weights.
Because these are final transforms and not local ones it can be seen that representing them as always as the sum of a translation, a rotation and a scaling produces very ugly results: instead, the game engine has to pick one of **two possible solutions** that have been seen to work.
They are the following:

- **Linear Blend Skinning** (**LBS**): here the final transforms are expressed as **4x4 matrices** and are interpolated using normal **linear matrix interpolation**.
    This approach works because standard lerp makes it so that when applying the total transform $$T$$ to a point we can interpret the final position $$p_P$$ as the *interpolation of per-bone transformed points*, i.e. the positions the point $$p_R$$ would have had it followed only one bone.
    So, if we call $$w_i$$ the weight of the $$i$$-th bone and $$T[b_i]$$ its final transform:

    $$
    {\bf T = \sum_{i = 0}^{N_{max}-1}{w_i T[b_i]}} \\
    p_{P} = T p_R = \left( \sum_{i = 0}^{N_{max}-1}{w_i T[b_i]} \right)p_R = \sum_{i = 0}^{N_{max}-1}{w_i (T[b_i] p_R)}
    $$

    This basic approach isn't horrible but suffers from the so-called **"candy wrapper effect"**, where due to the limitations of 4x4 matrix interpolation (*interpolating rotations doesn't produce a rotation and introduces shears*) it tends to *shrink* the mesh around joints.

{% include responsive-image.html path="assets/images/12_CandyWrapper.png" alt="Example of the candy wrapper effect" max-height="100px" %}

- **Dual Quaternion Skinning** (**DQS**): here the final transforms are expressed as **dual quaternions** and mixed using **dual quaternion interpolation**.
    This works because, differently from the sum of translations, rotations and scalings which had to pick an order of operations (S, R, T), dual quaternions completely avoid the problem by *rotating and translating simultaneously*.
    Thus:

    $$
    {\bf T = \text{mix}(w_0, T[b_0], \dots)} \\
    p_P = (\text{mix}(w_0, T[b_0], \dots))p_R
    $$

    This type of interpolation is **more expensive than LBS**, costing around 50% more floating point operations per-vertex and it's not as well established, so algorithms aren't as optimized.
    However, this techniques **avoids the "candy wrapper effect"** and could actually have the opposite effect of *enlarging the volumes*.
    Another thing of note is that dual quaternions **don't allow scalings**, not even uniform ones, so if scaling during animations is needed LBS is the safest choice.

Both techniques are used in practice and use exactly the same assets (*skinned mesh, skeleton, animations as a sequence of poses*), so they can be easily swapped for one another if needed.

### Combining animations

We've seen that in skeletal animations *poses can be easily blended* as long as they share the same skeleton: this not only means that we can effectively use poses as keyframes, but also opens the possibility of **combining different animations** altogether in real time during the game's execution!
In particular, there are two ways of combining animations: **transitions** between two animations and **compositing** (*or **layering***) different poses.
Let's briefly explore both approaches.

The most common way of combining animations is creating **transitions** between different animation cycles through the use of some transition function.
Imagine, for example, a character that's walking and must suddenly start running: we could have a walk cycle and a run cycle, but how do we bridge between the two to avoid a stark contrast.
We could create a bridging animation, but this would take time and a lot of work: what we can instead do cheaply is **interpolate between the current poses of both animations** using some **transition function** with an **offset** and use the resulting pose for rendering.

{% include responsive-image.html path="assets/images/12_SkeletalTransitions.png" alt="Example of transition between different animation cycles" max-height="250px" %}

The *quality of the transition can vary* depending on *how similar the two poses are* and on the choice of the offset, the transition **speed** (*number of frames before fully transitioning*) and the type of the *transition function*.
Usually good results are found with trial and error and many game engines offer animation authoring systems that speed up this trial process (*eg. Unity*).

The second way of combining animations is **compositing poses**: the idea is to **pick local transforms from different poses** and use them together on different bones with the option of interpolating some of them.
What we obtain is a **layering** of the two poses: we could, for example, layer the lower part of a running pose with the upper part of a sword slashing pose to obtain a pose where the character is both running and attacking with his blade.

{% include responsive-image.html path="assets/images/12_SkeletalLayering.png" alt="Example of the layering of different poses" max-height="250px" %}

### Skeletal animation assets

To recap, skeletal animations require the use of three different types of assets: **skeletons**, which are trees of bones, **skinned meshes**, i.e. meshes where vertices are linked to bones of a skeleton, and **skeletal animations** themselves, sequences of keyframe poses that define a local transform per bone.
There are many interchange formats for all tree, including *.SMD* by Valve, *.FBX* by Autodesk and *.BVH* by Biovision.

Now, as always, animation assets begin their life on disk, are transported onto central RAM and then find their way to the GPU being uploaded as a single *animation GPU object* after some preprocessing.

{% include responsive-image.html path="assets/images/12_SkeletalAssets.png" alt="Visualization of the life cycle of animation assets" max-height="200px" %}

As always considerations on the different cost of memory in the three storage units (*disk, RAM and VRAM*) applies, so we will have a lot of animation files and skeletons on disk, only the ones needed by the application in RAM and in VRAM we'll store only the assets that are currently needed for rendering.

Now, the most interesting of these assets on disk are **animation files**: these are basically *tables of different poses*, where each pose is a column containing the Local transform for each bone.
Each of these poses has an associated *timestamp* for use as keyframes, which can be expressed as either an int or a generic scalar and is usually defined in terms of *time passed form the start of the animation*.
As we've said before not all poses will have a local transform for every single bone in order to save space: the missing ones will be derived by the interpolation of the previous and next keyframes.

{% include responsive-image.html path="assets/images/12_SkeletalAnimationAssets.png" alt="Inner table structure of an animation asset" max-height="180px" %}

Lastly, some animation files can contain multiple animations all defined onto the same rest pose: in that case the different animation cycles are usually identified using index intervals.

### Creating skeletal animations

How are skeletal animations created?
The creation of an animation using this techniques requires *two fundamental steps* that must be taken when creating the 3D mesh to animate:

- **Rigging**: this refers to the **creation of the skeleton** data structure and the corresponding *rest pose* that is the duty of a *rigger*.
    A skeleton can be created for a single mesh or for a group of similar meshes (*eg. humanoid characters*), in which case we talk of a **shared rig**.
    Moreover, when constructing the skeleton we must also *define controls for the animator* to easily create poses with it: these include **constraints** that don't allow certain movements, like elbows bending backwards.

- **Skinning**: this process involves **painting the weights of bones** over different parts of the mesh, thus creating the links between vertices and bones that are used to animate the model.
    This task is the duty of a specific type of digital artist called a *skinner*.

Once this is done and we have a skeleton and a skinned mesh at our disposal we can start actually **animating the rig**.
As usual, this can be done either procedurally or by hand by a digital artist: let's explore these philosophies starting with an introduction on *inverse kinematics*, a "tool" used by both.

#### Inverse kinematics

**Inverse kinematics** is a field of computation very used both in robotics and in modern day video games: as opposed to *"forward" kinematics*, which starting from a number of local transforms computes where an object would end up, inverse kinematics asks the question "*if I need the object to be in position $$p$$, which set of local transforms would achieve this given this skeleton and these constraints*?"
As we can clearly see this is a much more difficult problem and as a matter of fact it could have any number of solutions, from $$0$$ or $$1$$ to even infinite valid solutions!

{% include responsive-image.html path="assets/images/12_InverseKinematics.png" alt="Example of an inverse kinematics problem" caption="Example of an inverse kinematic problem: given the starting and ending points (a, d), the length of each bone (k, h, j) and the constraints on the joints, find the highlighted angles (i.e. the relative rotations of the bones)." max-height="180px" %}

Fortunately usually in games we use inverse kinematics in very simple settings, for example ones only considering two bones.
Nonetheless we can face some **ambiguity** as multiple solutions may satisfy our constraints and therefore we need to choose the one that looks more natural: to do so we can *disambiguate using additional constraints*, for example adding **attractor points** that favour the solution with the minimum distance from them.

{% include responsive-image.html path="assets/images/12_AttractorPoints.png" alt="Example of an inverse kinematics problem where an attractor point is used to select the most natural solution" max-height="110px" %}

Now, what are inverse kinematics used for in video games?
A lot of things!
As a matter of fact they can be used both during *preprocessing* and in *real time*: during preprocessing they can **help animators** create poses by simple *drag-and-drop* mechanisms instead of manually creating the local transformations for bones.
Examples of real-time uses then include **procedural animations** for positioning the character's feet on ground, the movement of the character's hand over an object to grab it or the positioning of the hands over a weapon (*eg. dual wielding a sword*).
It can also help with **animation retargeting**, i.e. making the system autocorrect for small errors in bone lengths or in interpolated frames, and is often also used to *make attacks "connect"* with their target.

#### Ragdolls

**Procedural skeletal animations** are ones where we *let the physics system take control of bones* and are called **ragdolls**.
This can be done in a number of different ways, but some of the most common ones include using an actual Verlet simulation system with PDB constraints: after *adding a collider to most bones*, though not necessarily all, and *creating equidistance constraints* to replicate the skeleton structure plus some *joint constraints* that bound the angles between bones we can let the physics simulation take care of the object.
A humanoid character controlled this way won't probably stay on its feet, which is why this system is most used for **unconscious characters**.

Dead bodies rolling around isn't the only use case of ragdolls, however: instead we could let physics simulation take care of only a few **secondary bones** while the others are still controlled by a standard authored animation.
Often these secondary "bones" aren't bones at all: they could be, for example, *hair* or pieces of *cloth* that benefit much from being physically simulated.

#### Keyframe editing and mocap

If we instead decide to have our animation authored by a *digital animator* we have two possible routes we could pursue: manual **keyframe editing** and **motion capture**.
Let's see them one by one.

**Keyframe editing** is the task of manually posing the character in each pose/keyframe along a *timeline* performed by a *digital animator* with a 3D animation software.
Usually starting from the initial and final pose, the animator adds keyframes in between using a **timebar** to choose their timestamp and *editing the transition functions* between poses to obtain the wanted result.
To edit each pose the artist uses a **rig**, a set of GUI controls that ease the task of creating local transforms for the keyframes: this consists of the skeleton, rest pose included, of the set of constraints created by the rigger and of inverse kinematics mechanisms that allow the artist to drag parts of the mesh around with the appropriate attractors and automatically see the consequent changes in the pose (*eg. moving the hand twists the elbow*).

{% include responsive-image.html path="assets/images/12_KeyframeEditing.png" alt="Screenshot of a 3D animation software" max-height="150px" %}

The other alternative way of creating animations is through **motion capture**, also know as **mocap**.
This is a *very costly* technique which requires heavy setup in which a **real actor** is strapped with sensors and asked to perform the movements we want to see in the animation: high frequency equipment will *capture the movement* and translate it animation form.
The obtained animation usually requires *substantial postprocessing* to be cleaned up, during which **keyframes are extracted** removing the in-betweens to keep the final animation file reasonably compact.

{% include responsive-image.html path="assets/images/12_Mocap.png" alt="Photo of a footballer performing motion capture for a sports game" max-height="150px" %}

#### Machine Learning

In recent years there have been significant advancements towards a new way of creating skeleton animations by employing the power of **machine learning**: preliminary results show models learning over multiple generations how to contract and expand muscles in order to walk forward or run.

{% include responsive-image.html path="assets/images/12_MachineLearning.png" alt="Screenshots from papers about machine learning for authoring animations" max-height="100px" %}

This is the future since it can shine where normal skeletal animations struggle: for example, transitions between different animation cycles can be quite awkward when using normal animations because these transitions have to happen suddenly as a result of player inputs; however, if properly trained, machine learning techniques can predict these patterns of input and prepare the skeleton for a graceful transition.

{% include responsive-image.html path="assets/images/12_SkeletalLifeCycle.png" alt="Visualization of the complete life cycle of skeletal animations" caption="Visualization of the complete life cycle of skeletal animations" max-height="350px" %}

### More about skeletal animations

Let's conclude the paragraph about skeletal animations by exploring some more aspects of this amazing animation technique, starting from the fact that skeletons can be just considered **subtrees of the Scene Graph**, the only difference being that the local transforms of its nodes are the ones dictated by the current pose.
Using skeletons as part of the scene graph has many possible uses, some of which include **attaching a camera to a bone** as to create a third-person/first-person view, and using **per-bone collision proxies** for more accurate collision detection and handling.

{% include responsive-image.html path="assets/images/12_SkeletalHitboxes.png" alt="Example of a skeleton tree where each bone has its own collision proxy" max-height="250px" %}

Another interesting aspect of skeletal animations we haven't touched on yet is the various *preprocessing tasks* that can take place with them.
Some of them include:

- **Keyframe sparsification**: the idea is to take an animation and *reduce the number of its keyframes* to optimize the data structure.
    This is done by tentatively trying to remove each keyframe $$P_i$$, computing the in-between at the same timestamp and confront the two: if the difference is less than a certain threshold than the keyframe can be removed.

- **Animation retargeting**: given an animation designed for a certain skeleton here we want to *create a similar animation on a different skeleton*.
    This task is relatively easy if the only difference between the two skeletons is the length of the bones, quite hard otherwise.

- **Automatic generation from a blend shape**: given a blend shape animation we want to *derive a skeleton, a skinned mesh and an animation* automatically.
    This is quite a difficult task while the opposite operation is instead pretty easy (*just another form of baking*).

Lastly it's interesting to note that *both blend shapes and skeletal animations work well with **$$\mu$$-meshes***: as a matter of fact they can animate the base coarse mesh and have the displacement applied on top, with the displacement direction correctly adjusted based on the current frame of animation.

### Limitations of skeletal animations

While the bar for 3D video game quality has gone up in recent years, skeletal animations and skinning techniques have remained the same for over 10 years and their inability to keep up with the times is starting to show.
Among the most notable limitations of skeletal animations we have **crude transitions** between animations, which can look robotic, the stark **contrast between authored animations and ragdolling**, the lack of **physical justifications** for animations and the fact that **inverse kinematics are not realistic** 100% of the times and may look awkward.

Not only this, but **skinning** is becoming quite outdated too: despite allowing non-rigid transformations of the mesh, the deformations created by following multiple bones are sometimes **unrealistic**.
For example, they don't account for a number of different aspects, including:

- **dynamic effects** like a belly jiggling up and down during a run (can use physically simulated bones);
- **collisions** and contact surfaces;
- **volume preservations**, like for example muscle bulging;
- **speed deformations** (although *velocity skinning* tries to simulate them);

Good skinning and additional bone placements can somewhat mitigate these problems and it appears that DQS interpolation may work better (arguably).
Still, another problem is the fact that **interpolated/layered animations may look simplistic** and unaware of the broader context (which they are).

## Combining animation types

What to do to fix the limitations of skeletal animations?
One idea is to enrich the expressiveness of animations by **using skeletal animations and blend shapes together**!
We can combine both types of animation by *skinning the base morph target*, which represents the rest pose, and then use both types of animation: in particular, the *rest pose is created by blending morph targets* and then animated to the current pose using skinning.
For example, if we use relative encoding for blend shapes with LBS we have:

$$p_P = \left( \sum_{i = 0}^{N_{max}-1}{w_i T[b_i]} \right)(p_R + \color{blue}{\vec{d}})$$

where $$\color{blue}{\vec{d}}$$ is the displacement encoding the morph target that is applied to the rest pose before skinning.
This is very convenient as a *blend shape can be merged with a skinned mesh* to create a data structure that has both multiple geometries and a set of weight for each vertex, the same for all morph targets.

{% include responsive-image.html path="assets/images/12_SkeletalPlusBlend.png" alt="Visualization of the data structure combining blend shapes and skeletal animations" max-height="250px" %}

The combination of blend shapes and skeletal animations can be used to achieve a number of different results: we can combine *broad skeletal animations* with facial expressions or other *subtle effects created using blend shapes* (*eg. cheeks puffing, breathing...*).
But that's not all, as we can also harness the power of **blend shapes correctives**: these are subtle corrections operated with the use of blend shapes to fix the effects that simple skinning could not simulate by pre-compensating for them.

{% include responsive-image.html path="assets/images/12_BlendCorrectives.png" alt="Example of blend shapes correctives" max-height="120px" %}

This isn't a perfect solution, however, as it *breaks orthogonality* between meshes and animations; moreover, blend shapes are expensive to use!
Recent developments suggest the use of **neural skinning**, a ML-based technique that by observing a large number of accurate physical simulations of the object through the lens of the skeleton (*we feed it local transforms since they're more learnable*) can infer and apply the deformations that normal skinning cannot create.

## Choosing animation type

Now that we've described multiple ways of representing animations in depth, **which of them should we choose** for our game objects depending on the situation?
Let's try and categorize:

- With **kinematic animations** *each moving part must be its own mesh* since we're basically animating a subtree of the Scene Graph: this same fact makes ***rendering very simple*** as we can just use the standard procedures; moreover, the ability to *instantiate only once repeating objects* makes kinematic animations **very compact** in RAM.
    However, having distinct meshes for each part means we have to issue **multiple draw calls** for one single object and the usage of simple affine transformations makes **deformations impossible** as only rigid movements are allowed (*eg. no cloth*).

- **Skeletal animations** solves the problem of multiple draw calls by using only **one mesh** with its relative *skeleton and skinning*: however, that single draw call is **more computationally expensive** in GPU because of the need to *interpolate between bones with either LBS or DQS*.
    This type of animation, however, **allows for deformations** and even makes it possible to have *multiple animations playing at once* thanks to **layering**.

- Lastly, **blend shapes** need a *single draw call* too as they use **one mesh** with multiple geometries, which also allow for *multiple animations to play at once*: the addition of a large number of vertex positions, however, makes blend shapes the **heaviest representation** storage-wise.
    Moreover, the *linear interpolation used to blend between morph targets* makes animations **look bad** quite often.

In recent years, however, there have been proposed methods capable of *automatizing the choice of the type of animation to use*.
We're talking about **Geometry caches**, a new technology proposed by Crytech that can be used as a black box: the structure automatically *choses the type of animation to use dynamically*, with the possibility of using a mixture of more than one.
Baked, compressed and optimized for streaming, they flatten the skeletons reducing them to only two levels: the root and everything else.

{% include responsive-image.html path="assets/images/12_GeometryCaches.png" alt="Example of the same object being represented in different ways by a geometry cache" max-height="220px" %}
