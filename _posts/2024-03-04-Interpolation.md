---
title: Vector interpolation
---

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

## Linear interpolation

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

## Normalized interpolation

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

## Spherical interpolation

To fix the problem with normalized interpolation not being smooth when transitioning we define a new operation called **spherical interpolation**: conceptually, it creates an interpolated versor by interpolating the **angle** between the two starting ones.
This way we can create smooth transitions since we force a *constant angular velocity* for the interpolated versor.

$$\hat{d_2} = \frac{\sin((1 - t)\alpha)}{\sin(\alpha)} \hat{d_0} + \frac{\sin(t\alpha)}{\sin(\alpha)} \hat{d_1}$$

{% include responsive-image.html path="assets/images/02_Slerp.png" alt="Representation of the spherical interpolation" %}

As we can see from the formula, this operation is significantly **more complex** than both linear and normalized interpolation, requiring us to know the angle between the vectors/versors and to calculate a number of trigonometric functions.
Also, if $$t=0.5$$ the results are the same as normalized interpolation, and if the two versors are close enough the difference is not really noticeable: based on this we can decide if the added complexity is actually worth it on a case-by-case basis.
