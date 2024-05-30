---
title: Vector algebra
---

Every 3D video game needs to handle the placing of objects, items and characters in a 3D environment: to do so, it will need a way to manage the position of these elements and their movement in the scene.
It needs, in short, a representation of **points** (positions), **vectors** (movements) and **versors** (directions).

Before seeing how these mathematical elements are handled in 3D video games, though, we need to recall how they are defined, how they differ from one another and how they behave from a purely algebraical standpoint.

## Definitions

**Points** are positions in the 3D space: given a certain cartesian coordinate system, they are identified by their **coordinates** on the $$x$$, $$y$$ and $$z$$ axes.
Usually represented by uppercase letters, they can be described by listing their coordinates in the $$x$$, $$y$$, $$z$$ order:

$$P = \begin{pmatrix}4.2 \cr \frac{5}{6} \cr 6\end{pmatrix}$$

**Vectors**, on the other hand, represent movements in the 3D space: given a cartesian coordinate system, they are described by **how much they move objects** on the $$x$$, $$y$$ and $$z$$ axes respectively.
Unlike points, vectors *don't have a position* in the 3D space and can thus be visually represented wherever; identified by a lowercase letter with an arrow on it, they too can be laid out by listing their components (*sometimes referred to as $$v_x$$, $$v_y$$ and $$v_z$$ respectively*):

$$\vec{v} = \begin{pmatrix}7.8 \cr 5 \cr \sqrt{2} \end{pmatrix}$$

As we can see the representation of points and vectors is almost identical: this is because they're both basically groups of three real numbers; this can be formalized by saying they are a part of the **vector space** $$\bf{\mathbb{R}^3}$$.
Vector algebra deals primarily with elements from this set, so to differentiate singular real numbers from vectors and points we will call the former **scalars**.

$$
k \in \mathbb{R} \longrightarrow k \text{ is a scalar} \\
v \in \mathbb{R}^3 \longrightarrow v \text{ is either a vector or a point}
$$

Since points and vectors share the same representation we may begin to think they are somewhat correlated, and so it is.
In particular, since vectors represent movements and points positions, given two points $$A$$ and $$B$$ we can define the **vector $$\vec{v}$$ that moves $$A$$ in $$B$$ as the difference of $$B$$ and $$A$$** (*the one being subtracted is the starting point, aka the tail of the vector*):

$$
\vec{v} = B - A \\
\begin{pmatrix}3 \cr 2 \cr 0\end{pmatrix} - \begin{pmatrix}1 \cr 1 \cr 0\end{pmatrix} = \begin{pmatrix}3 - 1 \cr 2 - 1 \cr 0 - 0\end{pmatrix} = \begin{pmatrix}2 \cr 1 \cr 0\end{pmatrix}
$$

{% include responsive-image.html path="assets/images/01_Vectors-points.png" alt="Representation of a vector being the difference of two points" %}

It follows that given a point $$A$$ and a vector $$\vec{v}$$ we can find the point $$B$$ that is obtained by moving $$A$$ by the movement represented by $$\vec{v}$$ simply by **adding the vector to the point**:

$$B = A + \vec{v}$$

These sums and differences of points and vectors have the same properties of sum and differences of scalars: they are cumulative and associative.
We'll see that most vector operations follow the same rules of their scalar counterparts, with a few notable exceptions.

## Scaling

Let's start with the easiest operation: **scaling**.
This means taking a vector and multiplying it by a scalar, and has the effect of modifying the vector's length among all axes:

$$k\vec{v} = \begin{pmatrix}kv_x \cr kv_x \cr kv_x\end{pmatrix}$$

{% include responsive-image.html path="assets/images/01_Scaling.png" alt="Representation of a vector being scaled" %}

As we can see in the figure, scaling by a negative factor *inverts the vector direction*: if the scaling factor is exactly $$-1$$ we get the so called **opposite** vector.
Instead, scaling a vector by a factor of $$0$$ turns it in the **degenerate vector** $$\vec{0}$$: this is a vector that has no direction nor length and doesn't actually represent a movement.

## Sum and difference

Vector **sum** represents the **concatenation of the movements represented by the two vectors**: the result is a vector describing the total movement an object takes when following the two vectors consecutively.
It is obtained by summing the individual components and is visually obtained by positioning the tail of the second vector on the point of the first one and connecting the tail of the first to the point of the second (*aka the parallelogram rule*):

$$\vec{v} + \vec{w} = \begin{pmatrix}v_x + w_x \cr v_y + w_y \cr v_z + w_z \end{pmatrix}$$

{% include responsive-image.html path="assets/images/01_Sum.png" alt="Representation of vector sum and the parallelogram rule" %}

Composed with scaling, we can then understand the **difference** of two vectors as the **sum of the first one with the opposite of the second**.
There is, however, another geometric interpretation: turns out that the difference of two vectors is *the vector connecting their ends when their tails are in the same point*, where the tail of the difference is on the point of the vector being subtracted.

$$\vec{v} - \vec{w} = \begin{pmatrix}v_x - w_x \cr v_y - w_y \cr v_z - w_z \end{pmatrix}$$

{% include responsive-image.html path="assets/images/01_Difference.png" alt="Representation of vector difference" %}

As already said before, vector sum and difference follow the same rules as scalar sum and difference, in that the sum is commutative and associative while the difference is only associative.

## Norm

Due to representing movements, vectors have an innate *length* better know as their **norm**: indicated by double pipe symbols around the name of the vector, it represents how much an object would travel in space if moved by the aforementioned vector.
Applying Pythagoras' theorem it can be easily demonstrated that this measure is obtained by squaring the components of the vector, adding them together and calculating the square root of the result:

$$||\vec{v}|| = \sqrt{ {v_x}^2 + {v_y}^2 + {v_z}^2 }$$

Given this formula, the norm is always a **non-negative scalar**: this appears apt for a measure of distance.
Speaking of, the norm gives us an easy way to compute the distance between two points, as it is only the norm of the vector represented by their difference:

$$\text{distance between A and B} = ||B - A||$$

We can also note that the norm of a scaled vector is the product of the scale by the vector norm, since scaling has the effect of modifying the vector's length.

$$||k\vec{v}|| = k \cdot ||\vec{v}||$$

### Versors

Having introduced the concept of norm we can define **versors**: they can be imagined as vectors of norm $$1$$ that are used to represent **directions** rather than movements.
Identified by a lowercase letter with a cap on it, they cannot be scaled since they have no length for scaling:

$$||\hat{d}|| = 1$$

Given a vector we can always extract its direction as a versor by scaling it by the inverse of its norm: we call this operation **normalization**.

$$\frac{\vec{v}}{||\vec{v}||} = \hat{d}$$

Of course normalization can fail if the length of the original vector was $$0$$: this is, however, an acceptable edge case since the direction of the degenerate vector $$\vec{0}$$ isn't really defined.

## Dot product

Let's now introduce the first operation that's specific to vectors and versors: the **dot product**.
Sometimes called *inner product* and represented by a dot, the dot product is an operation which returns a **scalar** obtained by multiplying the components of the two vectors one by one and summing the results:

$$\vec{v} \cdot \vec{w} = v_x w_x + v_y w_y + v_z w_z$$

This operation is sometimes indicated with the symbol $$\left<v, w\right>$$, and can be also expressed in terms of matrix multiplication as $$\vec{v}^T \vec{w}$$.
We can also note that by the aforementioned formula, the dot product of a vector with itself resembles an awful lot the formula for the vector norm: this is a great observation, and in fact *the dot product of a vector with itself is the square of its norm*.

$$\vec{v} \cdot \vec{v} = {||\vec{v}||}^2$$

More interestingly, though, the dot product can be demonstrated to be equal to *the product of the norms of the two vectors and the cosine of the angle $$\alpha$$ between them*:

$$\bf{\vec{v} \cdot \vec{w} = ||\vec{v}|| \cdot ||\vec{w}|| \cdot \cos(\alpha)}$$

{% include responsive-image.html path="assets/images/01_DotProduct.png" alt="Representation of the angle between two vectors used for the dot product" %}

Due to the nature of the cosine, this alternative formula gives us so many interesting interpretations and observations about the dot product:

- Assuming neither vector is degenerate, **a dot product of $$\bf{0}$$ means the two vectors/versors are orthogonal**.
    On the other hand, a *positive value of the dot product represents an acute angle between the vectors/versors*, while a *negative value an obtuse angle*.

    $$\vec{v} \cdot \vec{w} = 0 \longleftrightarrow \vec{v} \perp \vec{w}$$

- Given two **versors**, the dot product is the **cosine of the angle between them** since their norm is $$1$$.
    This makes it also a measure of **similarity** between the two directions, with values spanning from $$-1$$ (*opposite directions*) to $$1$$ (*same direction*).

    $$
    \hat{v} \cdot \hat{w} = \cos(\alpha) \\
    \hat{v} \cdot \hat{w} \in [-1, 1]
    $$

- The dot product of a **vector with a versor** is the **projection of the vector on the direction of the versor**: this is due to the trigonometric definition of cosine as the ratio of the length of the hypotenuse and the length of the adjacent cathetus.

    $$\vec{v} \cdot \hat{d} = ||\vec{v}|| \cdot \cos(\alpha) = v_d$$

{% include responsive-image.html path="assets/images/01_Projection.png" alt="Representation of the projection of a vector along the axis represented by a versor" %}

As we could have guessed from its definition, the dot product is a **commutative** and **associative** operation.
This characteristics also allow us to demonstrate a result regarding the *norm of the sum of two vectors*, which is not the sum of the norms:

$$
||\vec{v} + \vec{w}||^2 = (\vec{v} + \vec{w}) \cdot (\vec{v} + \vec{w}) = \vec{v} \cdot \vec{v} + \vec{v} \cdot \vec{w} + \vec{w} \cdot \vec{v} + \vec{w} \cdot \vec{w} =  \bf{||\vec{v}||^2 + ||\vec{w}||^2 + 2 (\vec{v} \cdot \vec{w})}
$$

The dot product is also **distributive** with regards to the sum of vectors:

$$(\vec{v} + \vec{w}) \cdot \vec{u} = \vec{v} \cdot \vec{u} + \vec{w} \cdot \vec{u}$$

## Cross product

Following the dot product we now introduce a new kind of product between vectors that instead of resulting in a scalar returns a **vector** instead: we're talking about the **cross product**.
Indicated by the $$\times$$ symbol, the cross product is only defined in 3D: this is because it produces a vector that is guaranteed to be **orthogonal to the plane containing the factors**, meaning it is **orthogonal to both vectors**.

$$\vec{u} = \vec{v} \times \vec{w} \longrightarrow \vec{u} \perp \vec{v} \; \text{ and } \; \vec{u} \perp \vec{w}$$

More specifically, the cross product is defined as such:

$$\vec{v} \times \vec{w} = \begin{pmatrix}v_y w_z - v_z w_y \cr v_z w_x - v_x w_z \cr v_x w_y - v_y w_x\end{pmatrix}$$

To help remember this formula we can imagine putting the components of the two vectors in a matrix: now, each entry of the cross product is produced by calculating *the determinant of the matrix obtained by removing the corresponding line* from the one we just built, making sure that for the $$y$$ coordinate we start from the $$v_z$$ component.

$$
\begin{bmatrix}v_x & w_x \cr v_y & w_y \cr v_z & w_z \end{bmatrix} \longrightarrow u_x = D\begin{bmatrix} v_y & w_y \cr v_z & w_z\end{bmatrix}, u_y = D\begin{bmatrix} v_z & w_z \cr v_x & w_x\end{bmatrix}, u_z = D\begin{bmatrix}v_x & w_x \cr v_y & w_y \end{bmatrix}
$$

Similarly to the dot product, though, there are simplified ways to find the direction and length of the resulting vector.
Starting with the latter, it can be demonstrated that:

$$\bf{||\vec{v} \times \vec{w}|| = ||\vec{v}|| \cdot ||\vec{w}|| \cdot \sin(\alpha)}$$

where $$\alpha$$ is as usual the angle between the two vectors; in order to prevent the norm of the vector from being negative (which wouldn't make any sense), this angle is defined to be $$\alpha \in [0°, 180°]$$ and in case it is larger we simply take the opposite angle.
However, this alternative formula gives much insight in the possible interpretation of the norm of the cross product between two vectors:

- The norm of the cross product of **two versors** is exactly the **sine of the angle** between them.
    Note that the resulting vector is *usually not a versor*, since the sine has value 1 only in $$90°$$: this means the resulting vector will be a versor if and only if the starting versors were orthogonal.
    In any other case, if we want to obtain a versor orthogonal to two other ones, for example when calculating the normal of a triangle in 3D, we need to *normalize the result*:

    $$\hat{n} = \frac{\hat{d} \times \hat{c}}{||\hat{d} \times \hat{c}||} = \frac{\hat{d} \times \hat{c}}{\sin(\alpha)}$$

- Following the previous point, the norm of the cross product of two **versors** can be taken as a measure of **orthogonality** (or *collinearity*, if you wish): the value ranges from $$0$$ when the two versors are collinear (either the same or opposite) to $$1$$ when they are orthogonal.
    As a test of collinearity it works also for vectors: note that two collinear vector will have a **degenerate vector** as their cross product, which makes a lot of sense since there is an infinite number of vectors that are orthogonal to two vectors on the same line.

    $$\hat{d} \times \hat{c} \in [0, 1]$$

    Note how, in that sense, the cross product of a vector with itself is *always the degenerate vector*: a vector is always perfectly collinear with itself, so the sine of the angle always results to be zero.

    $$\vec{v} \times \vec{v} = \vec{0} \qquad  \forall \, \vec{v} \in \mathbb{R}^3$$

- Analyzing things from a geometric perspective, the norm of the cross product between two vectors is also equal to the **area of the parallelogram** created by pointing both vectors in the same point; alternative, it is *double the area of the triangle* created by the same vectors.
    This is because multiplying one of the vector's norm by the sine of the angle $$\alpha$$ returns the height of the parallelogram, while the other vector's norm serves as the base:

{% include responsive-image.html path="assets/images/01_CrossParallelogram.png" alt="Representation of the parallelogram's area" max-width="220px"%}

Another thing that is left to define is, however, the **direction** of the cross product: if it is true that the resulting vector is orthogonal to both factors and to the plane they share, we still need to understand if it is pointed "above" or "below" that plane.
To better understand this we can use the **handedness** of our reference cartesian frame, which is a way to interpret how the three orthonormal axes that compose it are positioned in relation to one another: in particular, they can be either *left-handed*, meaning the positive direction for the $$x$$ is left of the other two axes, or *right-handed*, meaning the positive direction for the $$x$$ is right from the other axes (*the most common*).
The naming structure comes from the fact that, as seen in the figure below, the placing of the axes can be obtained by sticking out the first three fingers of the left or right hand respectively.

{% include responsive-image.html path="assets/images/01_Handedness.png" alt="Representation of the handedness rule" caption="Left-handed frame on the left, right-handed frame on the right" max-width="500px"%}

To account for this difference in interpretation, the cross product is defined to follow the same rule for positioning so that $$\hat{x} \times \hat{y} = \hat{z}$$ regardless of the handedness of the reference frame.
So the direction of the cross product can be understood as such: taking the *hand indicated by the handedness of the frame* and *pointing the vectors with the thumb and index fingers*, the cross product has the **same direction as the middle finger**.

This rule, however, makes the cross product not only *not commutative*, but instead **anti-commutative**: inverting the order of the vectors yields the **opposite vector**.
This is because changing the order in which each finger is pointed requires a hand rotation, resulting in the direction being the exact opposite.

$$\bf{\vec{v} \times \vec{w}} = -(\vec{w} \times \vec{v})$$

Despite this, some nice properties still apply: the cross product is **associative with scalars** and **distributive with the sum**, although we need to be very careful with the ordering of operands.
The same care is required for a sort of associative-ness with the dot product, where some operands can be shifted around a bit: this is also sometimes referred to as the **triple product** and gives the volume of the parallelepiped created by the three vectors or, alternatively, the determinant of the $$3 \times 3$$ matrix created with their components.

$$
k(\vec{a} \times \vec{b}) = (k\vec{a}) \times \vec{b} \\
(\vec{a} + \vec{b}) \times \vec{c} = \vec{a} \times \vec{c} + \vec{b} \times \vec{c} \\
(\vec{a} \times \vec{b}) \cdot \vec{c} = (\vec{c} \times \vec{a}) \cdot \vec{b}
$$

## Vector interpolation

In every algebra where we define both *scaling elements by a real factor* and the *sum of elements* we can create the concept of **interpolation**, essentially mixing two or more elements in order to create a new one that displays a mix of their characteristics.
Given two elements $$x$$ and $$y$$, we define a **linear combination** between the two any element obtained by scaling the two and summing the results:

$$ax + by \qquad a,b \in \mathbb{R}$$

Linear combinations are not interpolations though: a **linear interpolation** is a linear combination where the scaling factors are a *partition of unity*, meaning **their sum is** $$\bf{1}$$.
In this case the scaling factors take the name of **weights**, since they define how each element is considered when mixing the characteristics to create the new one.

$$ax + by \qquad a,b \in \mathbb{R}, \; a + b = 1$$

In particular, we define two cases:

- we have a proper **interpolation** if the weights are between $$0$$ and $$1$$: this is the most common interpretation of interpolation and returns an object that is the mix of the two starting ones.

    $$ax + bx \qquad a,b \in [0,1], \; a + b = 1$$

    In case of only two elements the interpolation is usually defined with only **one parameter** $$\bf{t}$$ that represents the *percentage weight of the second element in the mix*: obviously the weight of the first elements follows.

    $$(1-t)x + ty \qquad t \in [0,1]$$

- if instead the weights are not between $$0$$ and $$1$$ but still sum to $$1$$ we have an **extrapolation**: this operation, sometimes used in computer graphics or other fields, returns an element that is not a mix of the two starting ones but instead sits on the line that connects them.

    $$ax + bx \qquad a,b \notin [0,1], \; a + b = 1$$

There are other non-linear types of interpolation, as we'll see, but for now let's explore how to interpolate vectors, versors and... points?

### Linear interpolation

Linearly interpolating **vectors** is very easy, since they have both the scale and sum operations and so all the math in the previous paragraph stands: being so diffused and useful, the operation is also known as *mix*, *blend* or **lerp** (*Linear intERPolation*).
In particular, when linearly interpolating vectors we can see that geometrically we obtain a vector whose *tail sits on the tails of the other two and whose arrow is on the segment connecting the arrows of the other two*:

{% include responsive-image.html path="assets/images/02_LinearInterpolation.png" alt="Representation of the linear interpolation of two vectors" %}

Although scaling and sum is not defined for points, we would like to have a way to easily find a point on the segment between two starting points.
To achieve this there's a useful trick that allows us to **linearly interpolate points**: instead of interpolating the actual points, we sum the vector defining the movement of the first one to the second one, scaled by a factor $$t$$, to the first point.
This way we obtain a point on the segment that is as near to the first point as $$t$$ is near to $$0$$:

$$C = A + t(B - A)$$

{% include responsive-image.html path="assets/images/02_PointInterpolation.png" alt="Representation of the linear interpolation of two points" %}

To ease operations, though, we sometimes just pretend to linearly interpolate points as if scaling and sum were defined for them (*we scale their components and sum them one by one*):

$$C = (1-t)A + tB$$

Points also provide an intuitive visual representation for **linear extrapolation**: as seen in the image below, extrapolating means finding a point on the line passing between the two starting points, but not one that is on the segment between them.

{% include responsive-image.html path="assets/images/02_PointExtrapolation.png" alt="Representation of the linear extrapolation of two points" %}

### Normalized interpolation

When it comes to interpolating **versors**, though, things get a little more difficult.
Since scaling a versor returns a vector, we cannot be sure that interpolating two versors will result in a versor: in fact, it can be demonstrated that this is not the case, since the norm of the resulting vector will be **less than** $$\bf{1}$$.

$$\vec{b} = (1-t)\hat{d} + t\hat{c} \; \longrightarrow \; ||\vec{b}|| < 1$$

{% include responsive-image.html path="assets/images/02_VersorInterp.png" alt="Representation of the linear interpolation of two versors" %}

To obtain a proper versor we thus need to **normalize** the result: interpolation followed by normalization is sometimes called **normalized interpolation**, or **nlerp**.
Instead of vectors whose arrow is on the segment between the two arrows, this operation returns a vector (*or versor in this case*) with its arrow on the arc of circumference between the two arrows.

$$\hat{b} = \frac{(1-t)\hat{d} + t\hat{c}}{||(1-t)\hat{d} + t\hat{c}||}$$

This operation has some flaws though: for example, it breaks when the two *versors are opposite*, since in that case the interpolated vector is the degenerate vector and we would thus be performing a division by $$0$$.
Not only that: if we wish to **smoothly interpolate** between two directions, for example, we see that performing subsequent normalized interpolations doesn't yield the expected result.
Instead, we would see the versor accelerating at the start and end of the transition and slowing down towards the middle: this is because we're interpolating on the segment between the arrows of the original versors and then projecting on the arc of circumference.

{% include responsive-image.html path="assets/images/02_NlerpProblem.png" alt="Representation of the problem of normalized interpolation" %}

### Spherical interpolation

To fix the problem with normalized interpolation not being smooth when transitioning we define a new operation called **spherical interpolation**: conceptually, it creates an interpolated versor by interpolating the **angle** between the two starting ones.
This way we can create smooth transitions since we force a *constant angular velocity* for the interpolated versor.

$$\hat{d_2} = \frac{\sin((1 - t)\alpha)}{\sin(\alpha)} \hat{d_0} + \frac{\sin(t\alpha)}{\sin(\alpha)} \hat{d_1}$$

{% include responsive-image.html path="assets/images/02_Slerp.png" alt="Representation of the spherical interpolation" %}

As we can see from the formula, this operation is significantly **more complex** than both linear and normalized interpolation, requiring us to know the angle between the vectors/versors and to calculate a number of trigonometric functions.
Also, if $$t=0.5$$ the results are the same as normalized interpolation, and if the two versors are close enough the difference is not really noticeable: based on this we can decide if the added complexity is actually worth it on a case-by-case basis.
