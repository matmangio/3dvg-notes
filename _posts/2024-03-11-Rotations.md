---
title: Representing rotations
---

In defining transforms as a sum of three components we left the representation of **rotations** to be discussed: while uniform scalings can be represented by simple scalars and translation by three-dimensional vectors, for rotations things are quite more complex.
In this chapter we explore the different possible representations for rotations and confront them on the basis of the following attributes:

- **Storage space** required;
- **Application** efficiency;
- **Composition** efficiency;
- **Inversion** efficiency;
- **Interpolation** efficiency;
- **Design** and assignment ease.

The chosen representation will need to be efficient on most of these parameters, be invertible and composable and also allow us to express both *orientations* (state) and *rotations* (changes of state).

## Rotations in 2D

Let's start by discussing how rotations can be expressed when only **two dimensions** are considered.
The answer in this case is trivial: we simply need a *scalar* to indicate the angle used to rotate the object, where by convention we can decide that positive numbers will be rotations in a counter-clockwise direction and negative numbers will instead rotate the object in a clockwise direction.

{% include responsive-image.html path="assets/images/05_2dRotations.png" alt="Representation of 2D rotations" max-width="150px" %}

By our parameters this is the perfect representation: is extremely cheap to store as it's only one scalar, very fast to apply using the following trigonometric formula:

$$r_\alpha\begin{pmatrix}x \cr y\end{pmatrix} = \begin{pmatrix} \cos(\alpha)x - \sin(\alpha)y \cr \sin(\alpha)x + \cos(\alpha)y\end{pmatrix}$$

while still being easy to invert by simply flipping the sign, easy to cumulate since we can simply sum the two angles and quite convenient to design for an artist.

A problem arises, however, when we try to linearly *interpolate* rotations defined this way.
As we can see in the figure below, because of the circular space in which angles reside and the following fact that *each angle is represented by two numbers*, one positive and one negative, there are actually *two valid interpolation values* for each couple of angles: in the example both the angle $$5°$$ and the opposite angle $$185°$$ are acceptable linear interpolations.

{% include responsive-image.html path="assets/images/05_ShortestPath.png" alt="Representation of the shortest path problem" max-width="400px" %}

In this case the solution is expressed by the so-called **shortest-path principle**: when choosing between the equivalent $$\alpha$$ and $$\alpha+360°$$ representations we will choose the one **most similar** to the other angle and use that for the linear interpolation.
With this method we're sure the interpolation will look the most "natural", since we will move along the shortest path between the two values.

2D rotations also have a very nice way of moving from an angle value to the versor that represents it on the trigonometric circumference: we simply use the *cosine* and *sine* of the angle as components.

$$\hat{v} = \begin{pmatrix}\cos(\alpha) \cr \sin(\alpha) \end{pmatrix}$$

{% include responsive-image.html path="assets/images/05_Atan2.png" alt="The atan2 situation" max-width="150px" %}

To do the inverse we can instead use the **atan2** function, a function that takes in input the sine and cosine of the angle and returns its value.
In this case the sine is $$\hat{v}_y$$ and the cosine $$\hat{v}_x$$, so we can write:

$$\alpha = \text{atan2}(\hat{v}_y, \hat{v}_x)$$

Note that since the arctangent is calculated by dividing the value of the sine for the one of the cosine this functions works even if $$\vec{v}$$ is not a versor, as by dividing one component with the other it performs an implicit normalization.
Therefore we can also write:

$$\alpha = \text{atan2}(\vec{v}_y, \vec{v}_x)$$

## Rotations in 3D

To start discussing the different representations of rotations in 3D let's begin by asking ourselves *how many dimensions does the rotation space have in 3D*?
This will give us a **lower bound** on the number of values needed to describe a rotation.
Let's immagine we want to describe the orientation of a camera: we would need a versor describing the direction it's looking at, which can be represented with two scalars (*the third one follows by knowing that the norm is unitary*), plus a scalar for the rotation angle around that axis.
As we can see, rotations have **3 degrees of freedom** (*DOF*), so we'll need at least three numbers to define a rotation.
When using rotations along with translations the total number of degrees of freedom is 6, which grows to 7 if we also consider scaling.

### 3x3 matrix

Let's start from the representation we already know: orthonormal **3x3 matrices** with determinant $$1$$, obtained as the left-top sub-matrix of a purely rotational transform (*no translation or scaling*).
Although this representation has the benefit of **containing the transformed versors explicitly as columns** and of being **easily applicable**, as we can simply multiply it for the point/vector/versor to obtain the rotated counterpart, it has many problems on the other fronts.
Let's evaluate it with respects to our metrics:

- *Storage space*: being composed of 9 scalars, as opposed to the minimum of 3 we demonstrated before, this representation is quite **wasteful in RAM**.

- *Application*: it is very **easy to apply** and especially in GPU, which is optimized for matrix algebra.

- *Composition*: it is **relatively easy to compose** as it is simply the multiplication of two 3x3 matrices, requiring a total of 27 multiplications.
    That is in theory: in practice, however, approximation errors can break the process and make it so that $$R_{tot} = R_0 \cdot R_1$$ is no longer a rotation, especially after a long sequence of compositions.

- *Inversion*: it is **easily invertibile** since by being orthonormal its inverse is simply its **transposed**, meaning $$R^{-1} = R^T$$.
    This not only allows for easy inversion since it just requires us to swap the position of three pairs of elements, but also that *the rows of the rotation matrix encode the representation of the world $$\vec{x}$$, $$\vec{y}$$ and $$\vec{z}$$ axis in local coordinates*.

- *Interpolation*: it is **very difficult to interpolate**, as linear interpolation between the single columns introduces shear and scale since the resulting columns are no longer versors.
    There is a solution to this: spherical interpolation can maintain the unitarity of the column versors; this, however, is quite an extreme solution as *slerp* is very complex and not viable in real-time situations with large scenes.

- *Design*: although they contain the local axes in world coordinates, 3x3 matrices are definitely **not intuitive** to design from an artist's perspective.

The serious problems with inversion, storage and composition make 3x3 matrices a quite weak representation of rotations and one that is not used in practice very often.

### Euler angles

The **euler angles** representation stems from Euler's theorem, which states that *all 3D rotations can be expressed as a **rotation around the $$\vec{x}$$ axis followed by a rotation around the $$\vec{y}$$ axis followed by a rotation around the $$\vec{z}$$ axis*** (the specific order is chosen arbitrarily but must be kept the same).
If we choose to represent rotations with this solution we only need to store **3 angles of rotation**, since the axes are already known: in aeronautical terms, these angles are respectively called *pitch* (upward-downward, usually $$\vec{x}$$), *yaw* (left-right, usually $$\vec{y}$$) and *roll* (wings parallel-perpendicular to the ground, $$\vec{z}$$).

{% include responsive-image.html path="assets/images/05_PitchRollYaw.png" alt="The representation of the roll, pitch and yaw angles" max-width="400px" %}

Thanks to these components this is an **extremely intuitive** representation, and if the angles are properly *bounded* we also have the nice property that each roll, pitch and yaw triplet corresponds to one and only one rotation.
In particular, roll and yaw can be restricted to $$[0, 360]$$ in order to avoid periodicity, while the pitch is usually restricted to $$[-90, 90]$$ since values beyond those can be expressed by changing the other two angles.
Despite these bounds, however, a problem called **gimbal lock** rears its ugly head: this is the event in which the first rotation makes the other two axes coincide, *losing one degree of freedom* as in this state rotation around one of the axes is no longer possible.
This eventuality cannot be avoided, now matter which order is chosen: a rotation of $$90$$° of the first axis will always result in gimbal lock.

{% include responsive-image.html path="assets/images/05_GimbalLock.png" alt="Gimbal lock representation" max-width="200px" %}

Let's now look at this representation through the lens of our metrics and see how it performs:

- *Storage space*: only requiring 3 scalars to be stored, this representation is the **best storage-wise**.

- *Application*: it is **not the most efficient**, since it requires 3 rotations in succession; moreover, it requires the computation of trigonometric functions, something that is computationally quite hard.

- *Composition*: one could think that composition involves simply summing the angles of the two rotations, but this isn't the case as the bound on the angles make it so that much more attention is needed, **limiting the efficiency of the composition**.

- *Inversion*: for the same reason as the composition, simply inverting the angles' sign wouldn't work.
    Instead, a **complex algorithm** involving trigonometric functions is needed to account for the order of rotations being of great importance.

- *Interpolation*: although linearly interpolating the three angles *somewhat* works if we also use the **shortest path approach** to account for the $$\alpha = \alpha + 360k$$ equivalence, the results usually aren't satisfactory.
    This is because the bounds on the pitch mean that to go from two specular orientations where the pitch is the same we wouldn't simply progressively change the pitch but we would instead rotate along the other two axes.

- *Design*: this is absolutely the **most intuitive** representation of rotations, allowing artists to author orientations precisely.

Because of how intuitive they are, euler angles are often exposed to artists to design and program, but their terrible mathematical properties mean that internally the rotations are converted to another representation.
This is the case for the Unity game engine, which allows users to manipulate euler angles but uses *quaternions* internally.

### Axis and angle

Physicists employ another different representation of rotations, the so-called **axis & angle** model: with this, each rotation is portrayed with the **axis** around which it takes place and the **angle** of rotation; note that in this case the axis is supposed to pass through the origin, but for more general representations we can combine it with translations.
With this representation we only need to represent a versor, the axis, and a scalar, the angle: by convention *positive values for the angle mean counterclockwise rotations* when seeing the axis from the point.
The two components sum up to a total of **4 scalars**, which is only slightly worse than the minimum of 3 storage-wise.

{% include responsive-image.html path="assets/images/05_AxisAngle.png" alt="Axis & angle representation" max-width="120px" %}

While inversion with this method is very easy, as we can just *invert either the axis or the angle*, what happens if we invert both?
As we can easily guess, by inverting both the axis and the angle we obtain the same exact rotation, meaning there are **two equivalent representations for each rotation**: $$(\hat{a}, \alpha)$$ and $$(-\hat{a}, -\alpha)$$; this is valid for all rotations except for *identity*, which has *infinite representations* where $$\alpha=0$$ and the axis is any axis.
Apart from this last particular case, having two equivalent representations for each rotation is a bit of an inconvenience as it requires interpolation to use the **shortest-path algorithm** and complicates testing for equality between rotations.

However, these problems can be solved using a **variant** of this representation called the **axis *times* angle** representation: instead of storing a versor and a scalar we store only a **vector** whose *direction represents the axis of rotation and whose norm represents the angle of rotation*.

$$\hat{a} = \frac{\vec{v}}{\|\vec{v}\|} \qquad \alpha = \|\vec{v}\|$$

Using this alternative representation we not only reduce the storage required for each rotation to a single vector (3 scalars), but also solve the problem of equivalent representations: here, in fact, inverting both the angle and the axis creates the same exact vector, since $$\vec{v} = (\alpha)(\hat{a}) = (-\alpha)(-\hat{a})$$.
Moreover, even the identity has one and only one representation, the degenerate vector $$\vec{0}$$.

Having solved this issue we can now evaluate this representation with our metrics:

- *Storage space*: since this representation requires only 3 or 4 numbers it is very **compact**, especially with the axis times angle variant.
    However, to properly distinguish between different axes of rotation we need **high precision** with how the numbers are represented.

- *Application*: it is **very difficult** to apply the rotations in this form.
    Some options include translating to a 3x3 matrix/quaternion or using the *Rodrigues rotation formula*: however, all of these methods require trigonometric functions and are therefore very computationally involved.

- *Composition*: it isn't at all easy or immediate to compose rotations in this form, since this requires a **very complex algorithm** for finding the new axis and angle of rotation.

- *Inversion*: inversion is **exceptionally cheap**, we can simply flip the sign of one component (or the whole vector in the axis times angle case).

- *Interpolation*: although when we don't use the axis times angle representation the interpolation requires much care in applying the *shortest-path method for both axis and angle*, along with the renormalizing of the axis with *nlerp or slerp* and the *handling of degenerate cases* such as opposite axes, this representation gives the **best results** when interpolating, usually producing the most "intuitive" intermediate rotations.

- *Design*: it is **very hard** to see which axis to use for the rotation, especially in the case of very complex movements like in animations where the hand of an artist is most needed.

### Quaternions

We now introduce **quaternions**, the representation of rotations that is most common in 3D video games: we'll see how with reasonable store requirements we obtain a method of representing rotations that is easy to apply, compose, interpolate and invert.
Before we can do this, however, we need to recall some algebraic concepts relating to *complex numbers*.

#### Complex numbers

**Complex numbers**, indicated as $$\mathbb{C}$$, are a class of numbers based on the arbitrary assumption that there exists an **imaginary number** $${\bf \color{blue}i}$$ with the following property: $${\bf \color{blue}i^2 = -1}$$. This class extends real numbers and was created to have a class of numbers that is *closed to any exponentiation*, even roots included:

$$
\begin{align*}
c \in \mathbb{C} &\rightarrow c^a \in \mathbb{C}, \quad & \forall a \in \mathbb{R} \\
\text{in particular: } c \in \mathbb{C} &\rightarrow \sqrt[\leftroot{-2}\uproot{2}2k]{c} \in \mathbb{C}, \quad & \forall k \in \mathbb{R}
\end{align*}
$$

Usually complex numbers are expressed as the sum $$a + b\color{blue}i$$ of a **real part** $$a$$ and an **imaginary part** $$b\color{blue}i$$, where $$a,b \in \mathbb{R}$$.
This representation makes the algebra of complex numbers easily understandable: they are summed component-wise and multiplied similarly, and we define the **conjugate** of a complex number $$c$$ as the complex number $$\bar{c} = a - b\color{blue}i$$ that has the *same real part but opposite imaginary part*.
Having defined this object we can also define the inversion operation by dividing the conjugate by the squared *magnitude* of the complex number, which is $$\|c\| = \sqrt{a^2 + b^2}$$.

$$
\begin{align*}
(a + b\color{blue}i) + (c + d \color{blue}i) &= (a + c) + (b + d)\color{blue}i \\
(a + b\color{blue}i) \cdot (c + d\color{blue}i) &= (ac - bd) + (ad + bc)\color{blue}i \\
(a + b\color{blue}i)^{-1} &= \frac{(a - b\color{blue}i)}{a^2 + b^2} = \frac{\bar{c}}{\|c\|^2}
\end{align*}
$$

What's interesting to us is the **geometric interpretation** of complex numbers: they can be represented as points/vectors on a plane where one axis represents the real part and the other the imaginary part.

{% include responsive-image.html path="assets/images/05_ComplexNumbers.png" %}

With this interpretation then the sum of complex numbers becomes the sum of vectors in this space, while the conjugate is nothing more than the same vector reflected along the real axis.
What's more revolutionary, however, is the new interpretation of **products**: these generate vectors whose *magnitude is the product of magnitudes* and whose *angle with the real axis is the sum of the angles with the real axis of the two original vectors*.
Therefore, **multiplying a vector with a *unitary complex number* $$r$$ with $$\|r\| = 1$$ will rotate the vector around the origin**: we've found a really convenient way to represent 2D rotations!
Moreover, the inverse rotation is simply the conjugate $$\bar{r}$$ since $$r^{-1} = \frac{\bar{r}}{\|r\|^2} = \bar{r}$$.

#### Defining quaternions

The idea of **quaternions** is to extend the realm of complex numbers so that it can easily and efficiently represent 3D rotations as well.
Hamilton proposed the class of quaternions $$\mathbb{H}$$ as an extension of complex numbers where **two more imaginary numbers** are introduced: $$j$$ and $$k$$.
Both of these numbers have the property that $$j^2 = k^2 = -1$$ but they are distinguished from one another and from $$i$$: moreover, the products between them follow very specific rules and are **non-commutative**.
As a rule of thumb, when multiplying two imaginary numbers following the $$ijk$$ order we get the third one: if we instead don't follow the order we get the opposite of the third one.

$$
i^2 = j^2 = k^2 = -1 \\
ij = k \qquad ji = -k \\
jk = i \qquad kj = -i \\
ki = j \qquad ik = -j
$$

With these new imaginary numbers we can now define each quaternion $$q \in \mathbb{H}$$ as the sum of **three different imaginary parts**, each with its own real coefficient, and a **real part**:

$$q = ai + bj + ck + d$$

sometimes also represented as the tuple of the *vector of imaginary coefficients* and the *real coefficient* or as a *4D vector of the coefficients*:

$$
\begin{align*}
q &= (a, b, c, d) \\
q &= (\vec{v}, d)
\end{align*}
$$

As we can expect, quaternion algebra follows the same rules as the one for complex numbers, with the only exception that now we have to be careful of the *order of factors in a product* when dealing with the imaginary parts: it follows that **quaternion product is non-commutative**.
We can also define the **conjugate** of a quaternion, which is simply the same with the sign of all imaginary parts flipped:

$$
\bar{q} = -ai -bj -ck + d
$$

In the same way as we did for complex numbers we can define the **magnitude** of a quaternion as the square root of the sum of squared coefficients: we'll say the quaternion is **unitary** if its magnitude is $$1$$.

$$\|q\| = \sqrt{a^2 + b^2 + c^2 + d^2}$$

Having defined this we can define inversion at last, that like with complex numbers is the quaternion resulting by dividing the conjugate by the squared magnitude: note that this means that **for unitary quaternions the inverse is the conjugate**.

$$
\begin{align*}
q^{-1} &= \frac{\bar{q}}{\|q\|^2} \\
\|q\| = 1 \; &\rightarrow \; q^{-1} = \bar{q}
\end{align*}
$$

#### Geometric interpretation of quaternions

Quaternions can represent both **3D point/vectors/versors** and **rotations**: it all depends on the value of their coordinates.
This becomes abundantly clear when describing quaternions as the tuple $$(\vec{v}, d)$$:

- if $${\bf d=0}$$ then the quaternion represents **3D point/vector/versor of coordinates** $${\bf \vec{v}}$$;

- if $${\bf \|q\| = 1}$$ then the quaternion represents a **rotation**;

- if neither is true, then the quaternion represents neither.

Basically, points in the $$ijk$$ space represent 3D points/vectors/versors, while rotations reside in a 4-dimensional space.
Although representing 3D objects this way can be seen as unnecessary since we have a perfectly good representation for them in the 3D cartesian space, the advantage comes when we talk about **applying rotations**.
As a matter of fact, if $$p \in \mathbb{H}$$ is a point/vector/versor and $$q \in \mathbb{H}$$ is a rotation, then it can be demonstrated that:

$${\bf q p \bar{q} \; \text{ is the rotated point/vector/versor}}$$

If we represent $$q = (\vec{w}, a)$$ and $$p = (\vec{v}, 0)$$ it can be easily demonstrate that the formula for the rotated point/vector/versor is the following:

$$
{\bf qp\bar{q} = (a^2\vec{v} + 2a (\vec{w}\times\vec{v}) + (\vec{v} \cdot \vec{w})\vec{w} - \vec{w} \times \vec{v} \times \vec{w}, \quad 0)}
$$

Moreover, using this result along with associativity we can demonstrate that **rotation composition is just quaternion multiplication**.
If we try and rotate a point $$p$$ first by rotation $$q_1$$ and then by $$q_0$$:

$$
q_0 \cdot (q_1 \cdot p \cdot \bar{q_1}) \cdot \bar{q_0} \\
= \\
(q_0 \cdot q_1) \cdot p \cdot (\bar{q_1} \cdot \bar{q_0}) \\
= \\
(q_0 \cdot q_1) \cdot p \cdot \overline{(q_0 \cdot q_1)}
$$

So the composition of rotation $$q_1$$ followed by rotation $$q_0$$ is the quaternion $$q_0 \cdot q_1$$: notice how, similarly to transforms, the **order of application is the inverse of the order in which they're written**.
Similarly as before we can then demonstrate the form of a quaternion multiplication between quaternions $$q = (\vec{w}, h)$$ and $$p = (\vec{v}, d)$$ is the following:

$$
{\bf qp = (d\vec{w} + h\vec{v} + \vec{w} \times \vec{v}, \; hd - \vec{w} \cdot \vec{v})}
$$

As we can see the presence of the cross product $$\vec{w} \times \vec{v}$$ makes quaternion multiplication **non-commutative** without making it anti-commutative: the only case in which order doesn't matter is when $$\vec{w} \times \vec{v} = \vec{0}$$, which happens only if the two vectors are colinear (the same or opposite), and as such the order in which we apply them doesn't matter.

But how are rotations **constructed** as quaternions?
The formula is actually very simple: a rotation of $${\bf \alpha}$$ **angles around axis** $${\bf \hat{a}}$$ is simply represented by the following quaternion:

$$
\begin{align*}
q &= \left(\sin\left(\frac{\alpha}{2}\right)\hat{a}, \; \cos\left(\frac{\alpha}{2}\right)\right) \\
&= \sin\left(\frac{\alpha}{2}\right)\hat{a}_x i + \sin\left(\frac{\alpha}{2}\right)\hat{a}_y j + \sin\left(\frac{\alpha}{2}\right)\hat{a}_z k + \cos\left(\frac{\alpha}{2}\right)
\end{align*}
$$

It can be easily verified that $$\|q\| = 1$$.
The reason half of the angle of rotation is used is quite complex, but can be summarized with the intuition that since application involves multiplying by both $$q$$ and $$\bar{q}$$ each of the two takes care of half the rotation.
This formula also confirms our postulate stating that the **inverse rotation is the conjugate quaternion**.
As a matter of fact, if we try to rotate around the same axis $$\hat{a}$$ by the opposite angle $$-\alpha$$ we obtain, thanks to the parity of sine and cosine:

$$
q^{-1} = \left( \sin\left(-\frac{\alpha}{2}\right) \hat{a}, \; \cos\left(-\frac{\alpha}{2}\right) \right) = \left(-\sin\left(\frac{\alpha}{2}\right)\hat{a}, \; \cos\left(\frac{\alpha}{2}\right)\right) = \bar{q}
$$

What happens if we instead **flip both axis and angle**?
Ideally we would like to obtain the **same quaternion** since this represents the same rotation, and in fact this is the case:

$$q' = \left( \sin\left( \frac{-\alpha}{2} \right) (-\hat{a}), \; \cos\left( \frac{-\alpha}{2} \right) \right) = \left( \sin\left( \frac{\alpha}{2} \right) \hat{a}, \; \cos\left( \frac{\alpha}{2} \right) \right) = q$$

We instead have problems when we look at the rotation $$q''$$ defined by axis $$\hat{a}$$ and angle $$\alpha'' = \alpha + 2\pi$$, i.e. the rotation obtained by adding an **odd number of full 360° rotations**.
This rotation obtains the same result as $$q = (\vec{v}, \alpha)$$, so we would also like these two to have the same representation: unfortunately, this is not the case.
Instead, the rotation $$q'' = (\vec{v}, \alpha + 2\pi)$$ is the **opposite quaternion**:

$$
q'' = \left( \sin\left( \frac{\alpha}{2} + \pi \right) \hat{a}, \; \cos\left( \frac{\alpha}{2} + \pi \right) \right) = \left( -\sin\left( \frac{\alpha}{2} \right) \hat{a}, \; -\cos\left( \frac{\alpha}{2} \right) \right) = -q
$$

In conclusion, quaternions $$\bf{q}$$ **and** $${\bf -q}$$ **encode the same rotation**: quaternions are a *double covering* of the set of rotations, where *each rotation is represented by **exactly two** quaternions, no more no less*.
As we'll see in the next paragraph, this creates some small problems with interpolation.
Moreover, this creates an interesting effect: we can now see that not only the zero rotation $$(\vec{0}, 1)$$ has conjugate equal to itself, but also *every quaternion without real part* since in that case $$\bar{q} = (-\vec{v},0) = -q = q$$.

#### Interpolating quaternions

Interpolating rotations as quaternions yields **good results** but it requires a bit of caution because of the two different representations of each rotation: to avoid artefacts we need to use the **shortest-path** method, interpolating the two representations that are closer to one another.
But how do we decide which of the two representations is "closer"?

The technique is rather simple.
Given a certain rotation $$p$$, we need to decide to interpolate it either with $$q$$ or with $$-q$$: to decide which of the two we can simply treat the quaternions as 4D vectors and calculate the **dot product** between $$p$$ and $$q$$ and then $$p$$ and $$-q$$.
If you recall, in fact, in case of unitary vectors like these ones the dot product was a measure of similarity: since both $$q$$ and $$-q$$ will yield the same absolute value, we choose the one whose dot product was **positive**.
In summary:

- If $$\text{dot}(p,q) \geq 0$$, interpolate with $$q$$;
- Otherwise, interpolate with $$-q$$.

The only other caveat of quaternion interpolation is that at first *the result may not be unitary*: for this we need **re-normalization** with *nlerp* in case of linear interpolation, or we can simply use spherical interpolation (*slerp*) and avoid this problem altogether.

#### Evaluating quaternions

Having discussed quaternions and their usage extensively we can now evaluate them using our metrics:

- *Storage space*: quaternions are **compact** to store, only requiring a total of 4 scalars to be recorded in memory (almost the minimum).
    However, they do require quite **high precision** for these numbers since most of them will be usually around $$0$$.

- *Application*: quaternions are **fast to apply**, not requiring the use of trigonometric functions but only vector operations (dot and cross product).

- *Composition*: being just quaternion multiplication composition works the same as application for quaternions, meaning they're very **easy to compose** as well.
    Moreover, it is *easy to enforce that the result stays a rotation*: we can simply just **renormalize** the result!

- *Inversion*: inverting a quaternion is **trivial** since the inverse is just the conjugate.

- *Interpolation*: although it requires a consideration of the shortest path, quaternion interpolation **behaves very well** when using *nlerp*, and behaves even better with *slerp* if we can afford it.

- *Design*: despite all their nice properties, quaternions are **terrible to design** as they're extremely unintuitive.
    This is why most often an euler angle representation is used when designing which is then transformed to a quaternion.

Because they behave so well and are so simple from many points of view, **quaternions are the solution of choice for 3D rotations in video games**.

### Summary

Quaternions being the best solution for internal calculations of rotations in video games doesn't mean, however, that other representations are completely useless: as we can see in the figure below, each has its strengths and weaknesses and each is therefore used in specific situations.

{% include responsive-image.html path="assets/images/05_RepresentationsTable.png" %}

It can be useful to know how to pass from one representation to the other: some of these translations are trivial (*axis & angle $$\leftrightarrow$$ quaternions, euler angles $$\rightarrow$$ 3x3 matrix*), others are a bit more involved (*euler angles $$\rightarrow$$ quaternions, quaternions $$\rightarrow$$ 3x3 matrix, axis & angle $$\leftrightarrow$$ 3x3 matrix*) and others are totally impractical, in which case it is often best to pass through an intermediate representation.

## Roto-translations

Up until now we've assumed that the rotation and translation components of a transformation are stored separately: we've seen before why this is a good idea generally, but sometimes it may be useful to store the *combination of a rotation and a translation* in a single data structure.
These kinds of transformations are called **roto-translations** or **rigid transformations**, where the reference to rigidity comes from the fact that since no scaling is allowed the object doesn't deform during the transformation. \\
There are, in fact, mathematical representations that are able to *store both translations and rotations jointly*: one we've already seen, that is a *4x4 matrix* (*or 3x4, since the last row is always the same*), and another that we're going to briefly explore now, *dual quaternions*.

**Dual quaternions** offer a better mathematical abstraction to model roto-translations: not only do they store these transformations in a single object, but they also offer way *better interpolation* than 4x4 matrices could ever achieve; for this reason they're sometimes employed in *animations*.
These new mathematical objects are based on a new fantasy assumption, the existence of a number $$\color{blue}{\epsilon}$$ so that:

$$\color{blue}{\epsilon} \neq 0, \quad \color{blue}{\epsilon}^2 = 0$$

A dual quaternion is then defined by the *sum of a **primal quaternion** $$p$$ representing the rotational part and a **dual quaternion** $$\color{blue}{\epsilon}q$$ representing the translational part*:

$${\bf p + \color{blue}{\epsilon}q} = ai + bj + ck + d + e\color{blue}{\epsilon}i + f\color{blue}{\epsilon}j + g\color{blue}{\epsilon}k + h\color{blue}{\epsilon}$$

A dual quaternion can either represent points/vectors/versors or a roto-translation:

- if $$p=1$$ and $$h = 0$$, then it represents a point/vector/versor of coordinates $$(e,f,g)$$;

- if $$\|p\| = 1$$ and $$\text{dot}(p, q) = 0$$, then it represents a roto-translation where $$p$$ is the rotational part and $$q$$ the translational part;

- if neither is true, then the quaternion represents neither.

To apply dual quaternions to a point/vector/versor we simply use the same formula we've seen for ordinary quaternions, where the conjugate of a dual quaternion is defined as $$\bar{b} = p - \color{blue}{\epsilon}q$$:

$$a' = b \cdot a \cdot \bar{b}$$
