---
title: Materials and lighting
---

The term "**material**" is quite ambiguous in the context of 3D video games as it can have two meanings:

- **Material model** (*in Computer Graphics*): a descriptor of *how a point on a surface reacts to light*, with a particular focus on how it reflects it.
    This translates in a set of **parameters** describing the behaviour of the physical substance the object is made of which during rendering will be fed to a (**local**) **lighting equation** to compute the *final color* of the pixels representing the object.

- **Material asset** (*in game engines*): a data structure to be *associated with a mesh object* describing the *material model parameters as varying* (or remaining constant) *over the surface*.
    Authored by a material artist, such a data structure technically encodes the *full status of the rendering engine* when the object is being drawn: in practice it consists of a set of **textures** (*diffuse, specular, normal...*), a set of **shaders** (*vertex and fragment*), a set of **global parameters** (*glossiness, ambient factor...*) and a set of **rendering flags** for a bunch of GPU systems (*back-face culling, rendering ordering...*).

In this chapter we'll explore both meanings, starting from the first and then seeing how the material model is encoded in the material asset.

## Lighting

Before we can start describing material models we need to understand what **lighting** truly is.
Well, lighting is one of the most important tasks of the rendering engine that is tasked with computing the *"final" apparent colors of objects as it appears to the viewer* as the result of their **interaction with light**.
In particular we're interested in **Local lighting**, which means we'll compute lighting **individually for each object** in the scene and we won't care of how light rays bounce around and interact with one another: we'll particularly be interested in the **reflective and refractive** components of light.

{% include responsive-image.html path="assets/images/11_LocalLighting.png" alt="Visualization of the local lighting model" max-height="180px" %}

In order to do this the GPU must evaluate each frame a **Lighting equation**, a function that for each fragment produces its final color in the render.
These models take as input three main components:

- <span style="color:blue"><b>Geometric data</b></span>: data regarding the geometry of the mesh (NTB) and the position of the viewer.
- <span style="color:orange"><b>Illuminant</b></span>: data modelling the Lighting environment, i.e. position and intensity of the lights.
- <span style="color:green"><b>Material parameters</b></span>: data modelling the *physical substance the illuminated object is made of*, which of course impacts on how it reacts to lighting.
    Which specific parameters are used depends on the chosen lighting equation but regardless these values can be stored as either:

    - **global parameters** that are equal for the whole mesh;

    - **per-vertex attributes** in the mesh description;

    - **as texels of a texture sheet**, which provides maximum freedom.

    If a material uses only global parameters it is called a **uniform material** since the whole mesh reacts the same to lighting.
    Instead, if per-vertex or per-texel attributes are used we have a **non-uniform** (*or **spatially varying***) **material** whose behaviour when illuminated depends on the illuminated model: think, for example, of an opaque plastic sticker on a shiny metal bike frame.

{% include responsive-image.html path="assets/images/11_IlluminationSchema.png" alt="Visualization of the data that is fed to the illumination model to compute the colors" max-height="200px" %}

A lot of different lighting models have been proposed: Phong, Lambert, Beckmann, Heidrich-Seidel, Cook-Torrance, Ward, Fresnel and so many more.
The choice of which to use is an important one as all of them offer *different tradeoffs between image quality and computational burden*: consider that this equation will need to be computed by the GPU each frame, for each light, multiple times per pixel.
As a matter of fact, they all feature different levels of:

- *computational complexity*;
- *realism and quality* of the render;
- *number of material parameters* required;
- *variety of lighting environments* that are easily supported;
- *richness of the simulated effect* and physical plausibility.

In this course we'll only have time to explore two (*or three?*) lighting models, so let's buckle up.

### Lambertian model

The most simple illumination model is the **Lambertian model**, which starts from a basic intuition: each object has a *"native" color*, usually called **diffuse color** $$d$$, that is *part of the material* and describes how the substance absorbs and reflects the different wavelengths of light.
This color interacts with the **intensity of the light** $$L$$ to create the color the object should be seen as if it was completely illuminated.
However, the parts of the object that are *oriented towards the light* should look brighter, while the ones on the other side should be darker: to evaluate this orientation we can simply compute the **light direction** $$\hat{L}$$ in each point (*versor from the point to the light*) and confront it with the **normal direction** $$\hat{n}$$ in that same point using a *dot product*, which measures similarity.
This is the resulting equation:

$$(\color{blue}{\hat{n}} \cdot \color{orange}{\hat{L}}) \begin{pmatrix} \color{green}{d_R} \\  \color{green}{d_G} \\ \color{green}{d_B}\end{pmatrix} \otimes \begin{pmatrix} \color{orange}{L_R} \\ \color{orange}{L_G} \\ \color{orange}{L_B}\end{pmatrix}$$

where $$\otimes$$ indicates a component-wise product and each component is color-coded to the type of component of the lighting equation it belongs to: notice that for non-uniform materials the diffuse color component will vary depending on the point on the object we're evaluating the equation for.
If **multiple lights** are in the scene we repeat this computation for each of them and sum the results to create the final color of each pixel (fragment).

{% include responsive-image.html path="assets/images/11_Lambert.png" alt="Example of a render using the lambertian equation" caption="Example of a render using the Lambertian model" max-height="150px" %}

Now, this equation is sort-of **physically based** since that's more or less how things work in real life for refraction, but as we can see in the figure it isn't nearly enough to provide good renders: it's too simple.
As we'll see, however, this equation will be used as a **diffusive component** in more complex ones.

### Phong model

**Phong lighting equation** is one of the most diffused material models in 3D video games, being the historical "OpenGL material" and offering a *great tradeoff between quality and efficiency*.
To understand how this equation operates it is most useful to present the equation and then break it down:

$$
(\color{blue}{\hat{n}} \cdot \color{orange}{\hat{L}}) \begin{pmatrix} \color{green}{d_R} \\  \color{green}{d_G} \\ \color{green}{d_B}\end{pmatrix} \otimes \begin{pmatrix} \color{orange}{L_R} \\ \color{orange}{L_G} \\ \color{orange}{L_B}\end{pmatrix} +
(\color{blue}{\hat{n}} \cdot \hat{H})^{\color{green}{E}} \begin{pmatrix} \color{green}{s_R} \\  \color{green}{s_G} \\ \color{green}{s_B}\end{pmatrix} \otimes \begin{pmatrix} \color{orange}{L_R} \\ \color{orange}{L_G} \\ \color{orange}{L_B}\end{pmatrix} +
\begin{pmatrix} \color{green}{a_R} \\  \color{green}{a_G} \\ \color{green}{a_B}\end{pmatrix} \otimes \begin{pmatrix} \color{orange}{A_R} \\ \color{orange}{A_G} \\ \color{orange}{A_B}\end{pmatrix} +
\begin{pmatrix} \color{green}{e_R} \\  \color{green}{e_G} \\ \color{green}{e_B}\end{pmatrix}
$$

As we can see, the lighting equation is defined as the **sum of 4 components**:

- **Diffuse component**, which represents the native color of the object (*refraction*), is equal to the Lambertian equation and is computed *for each light*;
- **Specular component**, which represents how the object *reflects light* because of its surface and is of course computed *for each light*;
- **Ambient component**, which encapsulates the total effect of global illumination in the scene (*i.e. lighting bouncing from other objects*) and is *added only once* since it doesn't depend on the lights;
- **Emission component**, which represents the light emitted by the object, is rarely used and is *added only once* to adjust the final color of objects that emit light and should therefore be brighter.

{% include responsive-image.html path="assets/images/11_PhongComponents.png" alt="Visualization of the different components of a Phong equation and their sum" max-height="150px" %}

Each of these components has a certain number of **material parameters**: in particular we have a color for each component, so a **diffusive color** (*base color of the object*), a **specular color** (*color of the highlights*), an **ambient color** (*color of the scene*) and an **emission color** (*color of the light emitted by the object*), plus one **specular exponent** $$E$$, a scalar in the range $$[1, 128)$$ that represents the *shininess* of the object and basically regulates the size of the highlights.
Let's now explore more in detail the different components of this equation, save for the diffuse one that we already described for Lambertian models.

#### Ambient term

The basic intuition for the **ambient term** is that *a bit of light will reach each point on the surface from all directions* thanks to light rays bouncing around in the scene.
Since it's coming from all directions, how much light reaches each point isn't thus dependent from surface normals but is instead *proportional to **how exposed** is the point of the surface and **how much light** is around overall* (both constants).

$$\begin{pmatrix} \color{green}{a_R} \\  \color{green}{a_G} \\ \color{green}{a_B}\end{pmatrix} \otimes \begin{pmatrix} \color{orange}{A_R} \\ \color{orange}{A_G} \\ \color{orange}{A_B}\end{pmatrix}$$

Without this ambient lighting things that are not directly lit by a light appear completely black, a very unpleasant and unrealistic effect that can be easily solved with this term.
To compute this component we need the **ambient color** as a material parameter: since most of the time this will simply be a set of different shades of the diffuse color, this ambient color is usually expressed as an **ambient occlusion** factor (AO), a scalar that is then multiplied by the diffuse color to obtain the ambient color.

$$\text{ambient color} = \text{ambient occlusion} \cdot \text{diffuse color}$$

This ambient occlusion factor can be stored as a per-vertex attribute like any other parameter, can be computed on-the-fly with an effect called *SSAO* (*see chapter on rendering*) or it can most commonly be expressed in a texture called an **AO map**: this type of texture sheet is *usually baked* from the model by evaluating how much each point is exposed to the outside (in this case we require an injective UV-map).

{% include responsive-image.html path="assets/images/11_AOmap.png" alt="Example of the baking of an AO map" max-height="200px" %}

#### Specular term

Surely the most interesting component of the Phong equation, the **specular term** simulates the **reflectiveness** of the material by using this basic intuition: when the *normal direction* $$\hat{n}$$ is similar to the the **halfway vector** $$\hat{H}$$ between the *light direction* $$\hat{L}$$ (direction from point to light) and the *view direction* $$\hat{V}$$ (direction from point to viewer) then it means that the light rays that reflect onto the surface are going towards the viewer, so the surface should present an *highlight*.

{% include responsive-image.html path="assets/images/11_SpecularTerm.png" alt="Schema of the specular term and the halfway vector" max-height="120px" %}

The halfway vector can be obtained as the **normalized interpolation** of $$\hat{V}$$ and $$\hat{L}$$ with parameter $$0.5$$:

$$\hat{H} = \text{nlerp}(\color{blue}{\hat{V}}, \color{orange}{\hat{L}}, 0.5) = \frac{\color{blue}{\hat{V}} + \color{orange}{\hat{L}}}{\| \color{blue}{\hat{V}} + \color{orange}{\hat{L}}\|}$$

To measure the similarity between the normal and the halfway vector we can then use the *dot product*, **discarding all negative values**: this is because if the value is negative it means that the angle between the two is $$> 90°$$ and no reflection can be shown.
However, how much light is reflected depends on the **smoothness** of the surface, where really smooth surfaces have extremely small but intense highlights and dull surfaces have bigger but weaker highlights.
To obtain this effect we *exponentiate the dot product* with an exponent that is $$\geq 1$$: this way depending on the exponent we can quickly reduce the factor to $$0$$ unless it was already close to $$1$$, only keeping really good matches.
We obtain the following formula:

$$(\color{blue}{\hat{n}} \cdot \hat{H})^{\color{green}{E}} \begin{pmatrix} \color{green}{s_R} \\  \color{green}{s_G} \\ \color{green}{s_B}\end{pmatrix} \otimes \begin{pmatrix} \color{orange}{L_R} \\ \color{orange}{L_G} \\ \color{orange}{L_B}\end{pmatrix}$$

As we can see, this term uses two different *material parameters*:

- **specular color** $$(s_R, s_G, s_B)$$: this determines the **intensity** and color of the highlight and is sometimes obtained using the diffuse color multiplied by a constant;
- **specular exponent** (*or "glossiness"*): this determines the **size** of the highlights as we've said before.

Sometimes both parameters are stored in the same texture which then takes the name of **specular map** or **glossiness map**: this is a 4 channel texture containing both the RGB specular color and glossiness.

##### <big><u><b>Micro-shape</b></u></big>

The specular term is very much **not physically based** and we can observe that it isn't even energy conserving: this isn't a big issue for us, however, since as long as it works well and looks realistic we're happy.
But which characteristics of real world objects is the *material model* trying to simulate?

We've talked about how materials try to simulate the **substance** an object is made of: we model if it absorbs photons or is transparent and in which proportion based on the light's wavelength, which is what gives the object its color (*it can also depend on electric conductivity, which is why metals are shiny*).
One thing we have't explored yet is how the material model simulates the **micro-shape** of the surface around a point: on a microscopic level, in fact, objects are not perfectly smooth but instead present creases and imperfections that affect how the object reacts to light.
In particular, if an object is **smooth at a microscopic level** then it is **more shiny** since the smoothness means that for the most part light rays bounce perfectly producing small highlights; a rough surface instead produces more dull object as the light scatters everywhere when it hits the surface.
This is why, for example, wet objects appear more shiny: the water goes to fill all the creases and smoothens the surface at a microscopic level.

{% include responsive-image.html path="assets/images/11_Microshape.png" alt="Example of different microscopic surfaces" max-height="180px" %}

Adding this ulterior layer of modelling we're now representing the structure of an object at different levels using different data structures:

- the **macro-structure** of the object is represented by its *low-poly mesh*;
- the **meso-structure** of the object is represented by the *bump map* (normal or displacement);
- the **micro-structure** of the object is represented by its *material*.

#### Emission term

The last component of the equation, the **emission term** tries to model the *light that comes from the object and reaches the camera*: as such it is a plain RGB value that gets added to the final color.
Note that this emitted light **doesn't illuminate other objects** as it isn't a real light: for this reason emittance is rarely used if not for small LED lights that should be visible in the dark.

$$\begin{pmatrix} \color{green}{e_R} \\  \color{green}{e_G} \\ \color{green}{e_B}\end{pmatrix}$$

This emission value can of course be stored in a texture that takes the name of an **emission map**.
Sometimes these maps are used with *HDR values*, i.e. values for the RGB components that are over $$1$$, to obtain a "glow" effect: we'll explore this better in the chapter about rendering.

#### Problems with Phong

The Phong lighting equation has been the standard since the '90s since it's *quick to compute*, *easy to control* by the material artist and years ago it was *hardwired in the graphics APIs* as the only model that was provided.
Because of this its material parameters have been the standard way to define materials for a long time, which had to by picked by hand by an artist: *diffuse color* ("what color is this?"), *specular color* ("how bright are the reflections?"), *specular exponent* ("how concentrated are the reflections?") and *ambient occlusion factor* ("how easy it is for light to reach here?").
These parameters being **not physically justified** makes choosing them really difficult: as a matter of fact there were tables defining these parameters for different substances that were used as reference.
Despite this, however, Phong models make **all materials look the same**, are **not very expressive** and are **too crude** and **unrealistic** for modern day video games (*realistic only if specular term is zero*).

{% include responsive-image.html path="assets/images/11_PhongExamples.png" alt="Examples of different substances being represented with the Phong model" caption="Examples of different substances being represented with the Phong model" max-height="180px" %}

Starting from the year 2000 the lighting equations used in games start to become more complex thanks to an increase in GPU processing power and RAM to store more textures and the advent of programmable shaders that allowed defining different material models.
This adds more terms to the equation and, in turn, **more material parameters**: we have ones for Fresnel effects (*"the more you look at something from the side the more it acts like a mirror"*), anisotropic effects, reflections using environment maps and much more.
Authoring materials thus becomes an **increasingly complex**, ad-hoc task: materials are different to port between objects and engines and it's difficult to guess the right parameters for a mesh, especially if it has to look good in different lighting environments.

### Physically Based Materials

Starting from 2010 onward a new alternative to the increasing mess of Phong-like lighting models has been proposed, one able to widen the **expressiveness** of the material models: **Physically Based Lighting** (**PBL**) and, in turn, **Physically Based Materials** (**PBM**).
The term "physically based" stems from the fact that these models are *more inspired by physical reality*: unlike Phong they don't infringe energy conservation and try to model how things work in the real world, taking *fewer shortcuts and tricks* to model things (*eg. not compressing diffuse color and AO in one texture to increase expressiveness*).
In general, Physically Based Materials and Lighting have the following objectives:

- **increased intuitiveness**: they try to provide material artists with a higher-level material description to ease the material authoring task, making realistic materials easier to design and *reducing the number of parameters* to just a few, each of which is usually standardized in a $$0$$ to $$1$$ range;

- **increased standardization**: they try to make materials more cross-engine and *portable*;

- **increased generality**: they try to accomodate for more lighting effects (*eg. Fresnel effects or anisotropic materials*) and make it easier to obtain *reasonably realistic lighting under a wider range of simulated lighting conditions*, ideally making it so that no combination of parameters looks bad;

- **increased realism and quality**: they try to be *more faithful to real world materials*, which ideally should all be able to be represented with this unified model, also allowing materials to be created from *real world samples*.

There are a number of different physically based lighting equations, each of which asks for a different number and type of **material parameters**.
Generally, however, a good spread of material parameters for a Physically Based Lighting model is the following:

- **Base color**: the RGB color of the object, same as in the traditional lighting models.
- **Specularity**: a scalar between $$0$$ and $$1$$ (*although sometimes it is a color*), it represents the *amount of light bouncing off the surface with reflections* regardless of if they're perfect reflections or scattered all over.
    It is usually pretty high since most objects reflect light, even if in different ways.
- **Metallicity**: a scalar between $$0$$ and $$1$$ indicating *how metallic the object appears*, which models how conductive its surface is since the cloud of electrons around metallic objects makes photons bounce away.
    In theory this should either be $$0$$ or $$1$$ with no in-between, but interpolation has been made possible to increase the expressiveness of the material.

{% include responsive-image.html path="assets/images/11_Metallicity.png" alt="Slider from not metallic to metallic" max-height="50px" %}

- **Roughness**: a scalar between $$0$$ and $$1$$ that models *how rough and ragged the surface of the object is at a microscopic level* (similar to specular exponent but more physically justified).

{% include responsive-image.html path="assets/images/11_Roughness.png" alt="Slider from rough to shiny" max-height="87px" %}

Of course all of these parameters can be baked into textures to provide the object with material details.

## Material assets

We've said at the beginning of this chapter that "materials" are both lighting models and the **assets** in which all that is needed to render an object is stored.
Let's quickly explore this second interpretation.

A material asset is a data structure that describes the **parameters of a material model** and also **how they're stored**, for example as texels in a certain texture, as per-vertex attributes, as global values etc, **the bump maps** to use, although these technically are part of the mesh's shape, and **the algorithms to use in lighting**, which are both the *shader programs* that actually include the lighting equations, the *flag and settings* of the rendering engine to be set, the *number and type of rendering passes* to be done and in which order etc.
The material asset is basically all that is sent to the GPU after the model's geometry to render it and it represents the *entire state of the rendering engine*.

{% include responsive-image.html path="assets/images/11_MaterialAsset.png" alt="Distinction between the type of data sent to the GPU" max-height="150px" %}

Materials are usually stored together with the models in the same mesh file, but some formats like .OBJ instead use a specific *material format*, .MLT in this case.
