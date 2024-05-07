---
title: Spatial transformations
---

As we've seen in the previous chapters, all 3D objects that appear in a video game are essentially made of *points*, *vectors* and *versors*: when the mesh is being built, these components are defined in the so called **Local space** (or *object space*), a reference system that has its origin somewhere in the object (usually at its foot) and uses its own $$x$$, $$y$$ and $$z$$ axes.

{% include responsive-image.html path="assets/images/04_LocalSpace.png" alt="The local space" %}

This creates a problem, however: when we want to place the objects in a **scene** we need to have them share a common reference space in order for them to interact with one another, a **World space** (or *global space*) that represents the current state of the virtual world.
We therefore need a way to move, rotate and scale the individual components of each object in order to place them in the scene.

## Spatial transformations

**Spatial transformations**, more commonly known as **Transforms**, serve exactly this purpose: they are *functions* that transform points in points, vectors in vectors and versors in versors, in a sense maintaining the structure of the object.
The idea is to **associate a Transform to each object** that describes how the mesh is placed in the scene: by applying the function to the points, vectors and versors that compose the object in Local space we obtain their representation in World space, which can be then fed to the rendering engine to create the frames shown to the user.
This way if we want to move or rotate a model we don't need to update the position of all its points one by one: we can instead simply change the transform associated with it, thus gaining a lot in performance.

{% include responsive-image.html path="assets/images/04_LocalToWorld.png" alt="Representation of the passage from local space to world space" max-width="350px" %}

However, transformations are not only used to encode a **state** as detailed before, but they can also encode a **change of state**: instead of using a transformation to place an object in World space, we can also use a transformation to *move* an object, thus updating its state and the transform that represents it.

{% include responsive-image.html path="assets/images/04_TransformsState.png" alt="Representation of transforms as both states and changes of state" max-width="350px" %}

### Affine transformations

Transformations are as general as mathematical functions, so we could ask ourselves if there's a particular class of transformations that we want to use.
There is, actually: in computer graphics and other fields, a particular useful class of transformations is used, called **affine transformations**.
In 3D video games we actually use a sub-set of affine transformations called *similarities*: since these transforms are still affine, we first discuss how general affine transforms work and how they're handled.

Affine transformations are just **arbitrary redefinitions of the reference frame**, which is exactly what we want: they are defined by the new *origin* (a point) and the new *set of 3 axis* (three vectors) to be used.
Points, vectors and versors will thus be transformed simply by reinterpreting their coordinates in the new reference frame: this kind of transformation *preserves parallelism but not angles*.

{% include responsive-image.html path="assets/images/04_AffineTransforms.png" alt="A generic affine transform" max-width="500px" %}

Given this, if we set the origin to be the *position of the object in World space* and orient the axes in that same space to reflect the *rotation of the object*, we can use the resulting affine transformation to move object's components (points, vectors, versors) from Local space to World space.

{% include responsive-image.html path="assets/images/04_LocalToWorldAffine.png" alt="Representation of moving from local space to world space" max-width="500px" %}

## The transform matrix

How do we mathematically represent an affine transformation?
Given the "origin" point $$p$$ and the three "axis" vectors $$\vec{x}$$, $$\vec{y}$$ and $$\vec{z}$$ that define the transform, we can transform a point $$a$$ and a vector $$\vec{v}$$ to their relative point and vector $$a'$$ and $$\vec{v}'$$ in World space by applying the following expressions:

$$
a' = p + a_x \vec{x} + a_y \vec{y} + a_z \vec{z} \\
\vec{v}' = v_x \vec{x} + v_y \vec{y} + v_z \vec{z}
$$

As we can see, the new coordinates are obtained as linear interpolations of the axes and the origin, the latter only appearing in the computation for points.

This whole process can be written concisely through the use of **homogenous coordinates** and the **matrix notation**.
First, we take the cartesian coordinates of the point, vector or versor to be transformed and add a 4th, "affine" coordinate $$w$$ which is either $$1$$ if it's a point or $$0$$ if it's a vector or versor (*unfortunately we cannot distinguish the two*): we thus obtain the so called *homogenous coordinates*.
We then multiply the resulting vector by a **4x4 matrix** that uses the homogenous coordinates of the axes and origin as its columns: this is called the **Transform matrix** and completely describes the transform.

{% include responsive-image.html path="assets/images/04_TransformMatrix.png" alt="Explanation of the transform matrix structure" max-width="200px" %}

Multiplying the 4D vector containing the coordinates in local space with this 4x4 matrix we obtain the transformed coordinates of the point/vector: as we can clearly see in the figure below, the $$w$$ coordinate being either $$1$$ or $$0$$ makes it so that the origin is only added if the object to transform was a point; moreover, this means that points will be transformed in points and vectors/versors in vectors/versors.

{% include responsive-image.html path="assets/images/04_MatrixLerp.png" alt="Visualization of the linear interpolation of the homogenous coordinates of the axes and origin" max-width="400px" %}

The resulting coordinates can also be seen as the *dot product of the corresponding row of the transform matrix with the 4D vector*: the first homogenous coordinate is in fact the result of the sum of the product of the elements of the first row with each respective coordinate of the original vector.

More interestingly, affine transformations can be seen as performing a **combination of geometric transformations** on the points, vectors and versors to which they're applied.
In particular, they apply:

- **Translation** (*only to points*), which is determined by the origin point's position;
- **Rotation**, which is determined by the orientation of the axes;
- **Scaling**, which for each axis is determined by the length of relative the vector (*if the axis is a versor no scaling is applied*) and as such can be *anisotropic* (not uniform);
- **Shearing**, which is determined by the angles between the axes (*if the axes don't form an orthogonal basis the models will be subject to a skewing of sort*).

From a terminology standpoint if only translation and rotation are applied we have an *isometry* or *rigid transform*, which gains its name by the fact that completely rigid objects can only be transformed in this way in the real world.
If we also add scaling we instead have a *similitude*, a transform which can change the size of the object but doesn't affect its shape since angles are left untouched.

## Transforms representation in games

Up until now we've represented transforms as 4x4 matrices: although this is a very elegant and convenient way of representing affine transformations, one that encapsulates the origin point and axes of the new reference frame in a concise way, it is most definitely not the only option!
In fact, using 4x4 matrices is not ideal in games for a variety of reasons; in order to understand why, let's see what games need to with 3D transformations:

- **Store** them in a compact enough data structure;
- **Apply** them efficiently on a large number of objects;
- **Composite** them in order to make objects move, which is done by composing a transformation representing *movement* to the one representing the *positioning* of the object in the scene, thus obtaining the new *positioning* transform (*note: transform composition is not commutative!*);
- **Invert** them in order to revert movement or to go from World space to Local space, which is not only useful to find intersections but also to see how objects are arranged in regards to one another;
- **Interpolate** them for a number or reasons, for example for animation purposes;
- **Design** them, meaning we have to use a data structure that an artist can easily author in order to obtain a desired effect.

We can explore the performance of transform matrices along these axes to see why they're *not a great fit for games*.
First of all, for each matrix we need 16 numbers to store it, which is not great; then we could think that applying a transform matrix to a point/vector/versor is not so bad since it's only a matrix-vector multiplication, but the fact that vectors and versors have the same representation after the transform is not ideal.
Matrices are then hellish to compose, since this entails a 4x4 matrix by 4x4 matrix multiplication for a grand total of approximately 128 scalar operations; moreover, after many compositions we're likely to encounter numerical errors.
Inversion is also not the quickest, but the real problems come with *interpolation*: if we simply interpolate the two matrices component-wise we won't obtain the expected result, and instead the composition of two rigid transforms will no longer be rigid.
Finally, transform matrices are not easy to author: representing rotation and scaling as a basis of three axes is not exactly the most intuitive thing possibile.
For all these reasons, although 4x4 matrices are the go-to representation of transforms in Computer Graphics, **we won't use Transform matrices in games**.

### Transforms as a sum of components

To find the best data structure for transformations in games we instead need to think what we usually need these transforms for: in particular, while *translations* and *rotations* are obviously fundamental in the placement of an object in the scene, *scaling* is usually less useful (*and can usually be done by precomputing the scaled models*), especially when non-isometric; lastly, *shearing* is almost never used in games.
The idea is then to **store the useful components individually** in a data structure that contains:

- **Translation**, which can simply be kept as a **vector**;
- **Rotation**, whose representation we'll cover in later chapters;
- **Scaling**, which since only uniform scaling is used can be kept as a **scalar**.

This division is also conceptually useful since different components apply to different mathematical objects: while *points* are affected by the whole three, *vectors* are unaffected by translations since they don't have a position and *versors* are also unaffected by scalings, since they don't have a length.
As we can see this "composite" representation of transformations also allows to differentiate vectors and versors, which is something matrices wouldn't do!

{% include responsive-image.html path="assets/images/04_EffectOfTransforms.png" alt="Table of how different mathematical objects are affected by the different components of a transform" max-width="150px" %}

The three components of a transformation with this representation usually take different names based on whether the transformation encodes a *state* or a *change of state*: the terms translation, rotation and scale are in fact used mostly for transformations that represent movements.
When instead used to represent a state the components are usually called *position*, *orientation* and *size*.

### Operations on transforms in games

In order to correctly define the composition, inversion and interpolation operations on the data structure that represents transforms as a sum of a translation, a rotation and a (*uniform*) scaling we must first decide how these components are **applied**.
We will adopt the following conventional order:

<p style="text-align:center"><b>Scaling \(\rightarrow\) Rotation \(\rightarrow\) Translation</b></p>

Of course the whole three components are only applied to points, while vectors will lose the translation component and versors will lose the scaling component.
All in all, a typical Transform class may look like:

```c#
class Transform {
    float s;    // Scaling
    Rotation r; // Rotation
    Vector3 t;  // Translation

    Vector3 apply_to_point(Vector3 p) { return r.apply_to(s * p) + t; }

    Vector3 apply_to_vector(Vector3 v) { return r.apply_to(s * v); }

    Vector3 apply_to_versor(Vector n) { return r.apply_to(n); }
}
```

#### Interpolation

Having defined the order of operations we can now start defining the more complex operations we may want to do with transformations, starting from **interpolation**.
This is the most simple of the three, as we simply *interpolate the single components*: both scaling and translation can be subject to the normal linear interpolation, and the rotation's representation will provide its own method of interpolation as well.

```c#
Transform mix_with(Transform b, float k) {
    Transform result;
    result.s = this.s * k + b.s * (1 - k);
    result.r = this.r.mix_with(b.r, k);
    result.t = this.t * k + b.t * (1 - k);
    return result;
}
```

Note that sometimes linear interpolation may not be the best solution: real Transform classes therefore sometimes provide methods for other kinds of interpolation such as normalized interpolation (*nlerp*), spherical interpolation (*slerp*) or geometric interpolation.

#### Inversion

Here things become a little bit more complicated as it's no longer sufficient to simply invert the three components singularly: this is because thanks to our order of operations the inverted scaling needs to apply to the translation, and the same goes for the rotation.

To better explain this concept let's explore the transformations more deeply.
Let's say we have a point $$p$$ and a function $$f()$$ that represents the transformation divided in its components:

$$f(p) = R(s \cdot p) + \vec{t}$$

where $$s$$ represents the scaling factor, $$R$$ the rotation application function and $$\vec{t}$$ the vector of translation.
We now want to find a function $$f^{-1}()$$ that is the inverse of $$f$$, meaning a function in the form:

$$f^{-1}(p) = R'(s' \cdot p) + \vec{t'} \\
\text{where } \quad q = f(p) \; \rightarrow \; p = f^{-1}(q)$$

To do so let's write the original equation in full and find the function that given $$q$$ returns $$p$$:

$$
q = R(s \cdot p) + \vec{t} \\
q - \vec{t} = R(s \cdot p) \\
R^{-1}(q - \vec{t}) = s \cdot p \\
R^{-1}(q - \vec{t})/s = p \\
R^{-1}(q)/s - R^{-1}(\vec{t})/s = p \\
{\bf R^{-1}\left(\frac{1}{s} \cdot q\right) + R^{-1}\left(\frac{1}{s} \cdot -\vec{t}\right) = p}
$$

We've thus obtained a new function composed of a scaling, rotation and translation: while the inverse scaling factor is simply its *inverse* $$\frac{1}{s}$$ and the rotation is the *inverse of the rotation* $$R^{-1}$$, the translation is *the original translation scaled and rotated by the new factors* $$R^{-1}(\frac{1}{s} \cdot -\vec{t})$$.
In code:

```c#
Transform inverse() {
    Transform result;
    result.s = 1.0f / this.s;
    result.r = this.r.inverse();
    result.t = result.r.apply_to(result.s * -this.t);
    return result;
}

```

#### Composition

A similar concept applies to composition, where it *isn't enough to simply compose the single components* but instead we need to operate some more complex calculations for the translation.

Let's say we have two transforms $$f_a(p) = R_a(s_a \cdot p) + \vec{t_a}$$ and $$f_b(p) = R_b(s_b \cdot p) + \vec{t_b}$$.
We now want to find a new transform $$f'(p) = R'(s' \cdot p) + \vec{t'}$$ that is the composition of the two, meaning:

$$f'(p) = f_a(f_b(p))$$

The correct formula for this case is simply found expanding this expression:

$$
f'(p) = f_a(f_b(p)) \\
f'(p) = f_a(R_b(s_b \cdot p) + \vec{t_b}) \\
f'(p) = R_a(s_a \cdot (R_b(s_b \cdot p) + \vec{t_b})) + \vec{t_a} \\
f'(p) = R_a(s_a R_b(s_b \cdot p) + s_a \vec{t_b}) + \vec{t_a} \\
f'(p) = R_a s_a R_b(s_b \cdot p) + R_a s_a \vec{t_b} + \vec{t_a}
$$

Up until now we've only assumed that rotations are linear functions, and as such can be distributed on a sum.
We now assume that the scaling is uniform, in which case we can use commutativity:

$$
f'(p) = R_a R_b (s_a s_b \cdot p) + R_a s_a \vec{t_b} + \vec{t_a} \\
{\bf f'(p) = (R_a R_b)((s_a s_b) \cdot p) + (R_a (s_a \vec{t_b}) + \vec{t_a})}
$$

As we can see, the output rotation is the composition of the two rotations, and the same is true for the scaling (*where composition simply means product*); instead, for translations the composition requires that the first vector is scaled and rotated by the scaling and rotation of the second transform before being summed to its translation.
In code:

```c#
Transform compose_with(Transform b) {
    Transform result;
    result.s = b.s * this.s;
    result.r = b.r.compose_with(this.r);
    result.t = b.r.apply_to(b.s * this.t) + b.t;
    return result;
}
```

### Anisotropic scaling in games

Some popular game engines insist on using non-uniform scaling, or **anisotropic** scaling: in this case the scaling is a vector, not a scalar, where each component represents the scaling along the respective axis.
This is the case for both Unity and Unreal engine.
The problem with this kind of scaling is that the transform class is **no longer closed to combination**, meaning the combination of two valid transforms isn't necessarily a valid transform: Unity solves this problem by not allowing the inversion of transforms, while Unreal computes an approximation of the combination if the scale is not uniform.
In any case it is better to avoid anisotropic scaling where possible and, if strictly needed, *apply it before everything else*.
