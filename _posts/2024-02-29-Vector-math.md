---
title: Points, vectors and versors
---

Every 3D video game needs to handle the placing of objects, items and characters in a 3D environment: to do so, it will need a way to manage the position of these elements and their movement in the scene.
It needs, in short, a representation of **points** (positions), **vectors** (movements) and **versors** (directions).

Before seeing how these mathematical elements are handled in 3D video games, though, we need to recall how they are defined, how they differ from one another and how they behave from a purely algebraical standpoint.

## Vector algebra

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

### Scaling

Let's start with the easiest operation: **scaling**.
This means taking a vector and multiplying it by a scalar, and has the effect of modifying the vector's length among all axes:

$$k\vec{v} = \begin{pmatrix}kv_x \cr kv_x \cr kv_x\end{pmatrix}$$

{% include responsive-image.html path="assets/images/01_Scaling.png" alt="Representation of a vector being scaled" %}

As we can see in the figure, scaling by a negative factor *inverts the vector direction*: if the scaling factor is exactly $$-1$$ we get the so called **opposite** vector.
Instead, scaling a vector by a factor of $$0$$ turns it in the **degenerate vector** $$\vec{0}$$: this is a vector that has no direction nor length and doesn't actually represent a movement.

### Sum and difference

Vector **sum** represents the **concatenation of the movements represented by the two vectors**: the result is a vector describing the total movement an object takes when following the two vectors consecutively.
It is obtained by summing the individual components and is visually obtained by positioning the tail of the second vector on the point of the first one and connecting the tail of the first to the point of the second (*aka the parallelogram rule*):

$$\vec{v} + \vec{w} = \begin{pmatrix}v_x + w_x \cr v_y + w_y \cr v_z + w_z \end{pmatrix}$$

{% include responsive-image.html path="assets/images/01_Sum.png" alt="Representation of vector sum and the parallelogram rule" %}

Composed with scaling, we can then understand the **difference** of two vectors as the **sum of the first one with the opposite of the second**.
There is, however, another geometric interpretation: turns out that the difference of two vectors is *the vector connecting their ends when their tails are in the same point*, where the tail of the difference is on the point of the vector being subtracted.

$$\vec{v} - \vec{w} = \begin{pmatrix}v_x - w_x \cr v_y - w_y \cr v_z - w_z \end{pmatrix}$$

{% include responsive-image.html path="assets/images/01_Difference.png" alt="Representation of vector difference" %}

As already said before, vector sum and difference follow the same rules as scalar sum and difference, in that the sum is commutative and associative while the difference is only associative.

### Norm

Due to representing movements, vectors have an innate *length* better know as their **norm**: indicated by double pipe symbols around the name of the vector, it represents how much an object would travel in space if moved by the aforementioned vector.
Applying Pythagoras' theorem it can be easily demonstrated that this measure is obtained by squaring the components of the vector, adding them together and calculating the square root of the result:

$$||\vec{v}|| = \sqrt{ {v_x}^2 + {v_y}^2 + {v_z}^2 }$$

Given this formula, the norm is always a **non-negative scalar**: this appears apt for a measure of distance.
Speaking of, the norm gives us an easy way to compute the distance between two points, as it is only the norm of the vector represented by their difference:

$$\text{distance between A and B} = ||B - A||$$

We can also note that the norm of a scaled vector is the product of the scale by the vector norm, since scaling has the effect of modifying the vector's length.

$$||k\vec{v}|| = k \cdot ||\vec{v}||$$

#### Versors

Having introduced the concept of norm we can define **versors**: they can be imagined as vectors of norm $$1$$ that are used to represent **directions** rather than movements.
Identified by a lowercase letter with a cap on it, they cannot be scaled since they have no length for scaling:

$$||\hat{d}|| = 1$$

Given a vector we can always extract its direction as a versor by scaling it by the inverse of its norm: we call this operation **normalization**.

$$\frac{\vec{v}}{||\vec{v}||} = \hat{d}$$

Of course normalization can fail if the length of the original vector was $$0$$: this is, however, an acceptable edge case since the direction of the degenerate vector $$\vec{0}$$ isn't really defined.

### Dot product

Let's now introduce the first operation that's specific to vectors and versors: the **dot product**.
Sometimes called *inner product* and represented by a dot, the dot product is an operation which returns a **scalar** obtained by multiplying the components of the two vectors one by one and summing the results:

$$\vec{v} \cdot \vec{w} = v_x w_x + v_y w_y + v_z w_z$$

This operation is sometimes indicated with the symbol $$\left<v, w\right>$$, and can be also expressed in terms of matrix multiplication as $$\vec{v}^T \vec{w}$$.
We can also note that by the aforementioned formula, the dot product of a vector with itself resembles an awful lot the formula for the vector norm: this is a great observation, and in fact *the dot product of a vector with itself is the square of its norm*.

$$\vec{v} \cdot \vec{v} = {||\vec{v}||}^2$$

More interestingly, though, the dot product can be demonstrated to be equal to *the product of the norms of the two vectors and the cosign of the angle $$\alpha$$ between them*:

$$\vec{v} \cdot \vec{w} = ||\vec{v}|| \cdot ||\vec{w}|| \cdot \cos(\alpha)$$

{% include responsive-image.html path="assets/images/01_DotProduct.png" alt="Representation of the angle between two vectors used for the dot product" %}

Due to the nature of the cosign, this alternative formula gives us so many interesting interpretations and observations about the dot product:

- Assuming neither vector is degenerate, **a dot product of $$\bf{0}$$ means the two vectors/versors are orthogonal**.
    On the other hand, a *positive value of the dot product represents an acute angle between the vectors/versors*, while a *negative value an obtuse angle*.

    $$\vec{v} \cdot \vec{w} = 0 \longleftrightarrow \vec{v} \perp \vec{w}$$

- Given two **versors**, the dot product is the **cosign of the angle between them** since their norm is $$1$$.
    This makes it also a measure of **similarity** between the two directions, with values spanning from $$-1$$ (*opposite directions*) to $$1$$ (*same direction*).

    $$
    \hat{v} \cdot \hat{w} = \cos(\alpha) \\
    \hat{v} \cdot \hat{w} \in [-1, 1]
    $$

- The dot product of a **vector with a versor** is the **projection of the vector on the direction of the versor**: this is due to the trigonometric definition of cosign as the ratio of the length of the hypotenuse and the length of the adjacent cathetus.

    $$\vec{v} \cdot \hat{d} = ||\vec{v}|| \cdot \cos(\alpha) = v_d$$

{% include responsive-image.html path="assets/images/01_Projection.png" alt="Representation of the projection of a vector along the axis represented by a versor" %}

As we could have guessed from its definition, the dot product is a **commutative** and **associative** operation.
This characteristics also allow us to demonstrate a result regarding the *norm of the sum of two vectors*, which is not the sum of the norms:

$$
||\vec{v} + \vec{w}||^2 = (\vec{v} + \vec{w}) \cdot (\vec{v} + \vec{w}) = \vec{v} \cdot \vec{v} + \vec{v} \cdot \vec{w} + \vec{w} \cdot \vec{v} + \vec{w} \cdot \vec{w} =  \bf{||\vec{v}||^2 + ||\vec{w}|| + 2 (\vec{v} \cdot \vec{w})}
$$

## Interpolation

### Normalized interpolation

### Spherical interpolation

## Cross product
