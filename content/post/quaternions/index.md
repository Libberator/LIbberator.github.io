---
title: Your Guide to Quaternions
description: Do quaternions have your head spinning? What are they? How do you use them?
date: 2023-07-04 12:00:00-0700
slug: "quaternions"
categories:
    - Blogging
tags: 
    - math
    - c#
    - unity
image: https://hugelolcdn.com/hugewoah.com/i/7950.gif
math: true
---

> <center><i>Here as he walked by on the 16th of October 1843 Sir William Rowan Hamilton<br>in a flash of genius discovered the fundamental formula for quaternion multiplication:<br><code>i<sup>2</sup> = j<sup>2</sup> = k<sup>2</sup> = ijk = -1</code></i><br>—Plaque on Broom Bridge, Dublin</center>

## Intro

So you want to learn about quaternions? Well, you've come to the right place. I'll try my best to simplify it for you. This is written assuming that you know the following:
- Vectors and/or Vector math fundamentals,
- the difference between "global" and "local", or the term "relative to"

I've split this article into four main parts:
1. **[Intuition](#intuition)**. How to think about them
2. **[Properties of Quaternions](#properties-of-quaternions)**
3. **[Unity Methods](#unity-methods)** and alternative rotational options
4. Deeper dive into the **[Math](#math)** behind them

### Terminology

You probably know already that a quaternion represents a "rotation". Great! (*Technically*, they can be used for other things like scale, but not for our purposes. I digress...) However, the term "rotation" is a bit overloaded, so to clarify the terminology I'll be splitting it into two distinct terms (Note: these are not "official" or universally standardized terms):
- <u>Orientation</u>: the **state** or **pose** of an object. How the object is ~~positioned~~/oriented in space.
- <u>Rotation</u>: the **action** or **verb** of rotating/turning/spinning an object. Specifically denotes the transformation or struct that will be *applied* to another **rotation *or* orientation**.

<details><summary><b>Example Terminology Usage</b></summary>
Consider an object initially facing East. "Facing East" represents an orientation. To make the object face South, we need to apply a rotation. If we say "turn 90° about the Y axis", this describes a rotation without explicitly mentioning the resulting orientation. It's like giving driving directions to a computer.
</details><br>

When you see or hear "multiplying" (in the context of quaternions), it's easier to instead think of "applying" - not in the sense of "setting" but rather "modifying by". Multiplying a quaternion with a vector means you're **applying** the rotation *to* the vector. If you've ever heard someone say *"multiplying quaternions together is like adding them"*, **unlearn that**. We don't "add" quaternions in the traditional sense. Instead, one rotation gets *applied* to the other. It's better in the long-run to think about them properly rather than lazily.

Also worth mentioning...
- I will be using Unity's coordinate system (X = right, Y = up, Z = forward), which is a Left-Handed coordinate system (i.e. satisfies Left-Hand Rule). If you don't know what that means, look it up. In short: positive angles are CCW rotations when looking in the same direction of the axis you're rotating about.

## Intuition

### Frame of Reference

Are you looking to perform a *relative* rotation or a *global* one? Just like how a vector can represent either a global or local position, a quaternion struct can be global or local - it matters how you set them up and use them. Without any context, you can consider all quaternions to be global because you have nothing else to reference the rotation on. However, if you set up a quaternion using a relative value and apply it to the same object, that makes it a relative rotation. For instance, using `transform.forward` would make the rotation relative to the object's `Transform`, while `Vector3.forward` represents a global rotation.

It's generally not a good idea to try to share relative rotations with another object, especially if they started with a different orientation.

Don't confuse `transform.rotation` and `transform.localRotation`. Know the difference.<br>
Are you calculating a global orientation relative to the global XYZ coordinates (e.g. wanting to make your character face a target), then use the global-facing `transform.rotation`. If you only care about how it rotates with respect to its parent (e.g. Animations, door swinging on a hinge), then use `transform.localRotation`.

Bonus Tip: In Unity, toggle between "Local" and "Global" to see which axis is which on the Gizmo. Extra useful if you imported a model and the axes are set up differently.

### Axis and Angles

Look around yourself at all the things that do any kind of rotating. And I want you to think in terms of **Axis** and **Angles**. That's a singular axis: a vector in \*any\* direction of your choice. This axis will be what gets rotated *around* by a certain amount of angular units. I'll be using degrees, but radians are valid too.

Leaning back in your chair? Nah. You're rotating the chair negative degrees about its `right` axis (or positive degrees about the `left` axis).

Looking down and to your left? Nah. Your neck is oriented some positive degrees around `Vector(1, -1, 0)`, assuming Z is your forward. Might be tough to wrap your head around.

Grab a pineapple. :pineapple: Grab a pen. :pen: Make a Pineapple Pen. :pineapple: :pen: That pen is the axis, at whatever arbitrary vector you stuck it in at. Spin it to see how it rotates.

Why think in terms of axis and angles? Because that's more in line with how quaternions actually rotate. If you have a start and end orientation and want to interpolate between them, the shortest path is achieved with this kind of visualization. It's like drawing two dots on a basketball and connecting it with the shortest line. Look at the path the dot needs to travel, and rotate it with an axis that's perpendicular to that.

### Why Euler Angles Suck

*Disclaimer: These reasons don't really apply to standard 2D games, because you usually only rotate thing about a single Z axis. So it's typically fine to use Euler Angles there.*

<details><summary>What are Euler Angles?</summary>
It's a combination of 3 numbers representing the amount of angles rotated about each axes (X, Y, and Z), with a total range of 360° each. This is what you're used to using and seeing and what's displayed in Unity's Inspector for a Transform's "Rotation".
</details>

&ensp;<b>#1. You can't always trust the numbers you see in the Inspector</b>

Those numbers next to "Rotation" aren't even its `transform.rotation`, but rather the Transform's **local orientation** (i.e. relative to its parent). The Inspector lies to you, so don't be fooled.

Despite displaying as Euler angles, they're actually a Quaternion behind the scenes. If you rotate around multiple axes, like for some 3D games, don't use its Euler angles as a reference to base your logic off of. This is because the internal conversion that Unity does between Quaternions and Euler angles can have some of the angles "jumping around", giving you **unpredictable** and **unreliable** behavior.

&ensp;<b>#2. Euler Angles are subject to Gimbal Lock</b>

"What even is that?" you might be thinking. Let me Google that for you: "Gimbal Lock happens when two of the rotational axes align, causing a loss of one degree of freedom." Okay, that wasn't very helpful.

But it's fairly easy to demonstrate with an actual example, so let's do a "show, don't tell" approach here. Put the following script on a GameObject:

```csharp
public class GimbalLockExample : MonoBehaviour
{
    [SerializeField] private float _speed = 50f;
    private float _pitch, _yaw, _roll;

    private void Update()
    {
        var pitchInput = Input.GetAxisRaw("Vertical"); // W/S
        var yawInput = Input.GetAxisRaw("Horizontal"); // A/D
        var rollInput = (Input.GetKey(KeyCode.Q) ? 1f : 0f) + 
                        (Input.GetKey(KeyCode.E) ? -1f : 0f); // Q/E

        _pitch += pitchInput * _speed * Time.deltaTime;
        _yaw += yawInput * _speed * Time.deltaTime;
        _roll += rollInput * _speed * Time.deltaTime;

        transform.rotation = Quaternion.Euler(_pitch, _yaw, _roll);
    }
}
```
Press 'S' until you've pitched upwards 90° ("inverted controls" are just for this demo; I'm not a monster) so that the local forward vector aligns with the global Y-axis - this is when the red and blue rings align in the video below (at 0:10). Then you'll see that any adjustment to Yaw (A/D) or Roll (Q/E) will be **indistinguishable** from one another; they both behave like Roll. That's a loss of a degree of freedom (a.k.a. "gimbal lock").

{{<video src="gimbal-lock.webm" autoplay="true" loop="true">}}

<center><u><b><a href="../../g/gimbal-lock" target="_blank">Play this Gimbal Lock demo for yourself!</a></b></u></center>

&ensp;<b>#3. Interpolation sucks</b>

At the end of the **[Axis and Angles](#axis-and-angles)** section, I pointed out that quaternions will interpolate along the shortest path. If you interpolate with Euler angles, well... it's not a straight path. It will be more like a soft "S". Or it will just glitch out the rotation entirely, because Lerping one axis from 359° to 0° *doesn't* move just 1° like you'd hope it would.\*

*\*This example assumes you're manually handling the Lerping of Euler angles, and not using built-in Quaternion methods.*

&ensp;<b>#4. Dealing with the wrap-around point sucks</b>

Having to add extra code to handle the transition between 0° and 360° (or -180° and 180°) to complete the circle is tedious. This is a situational game-dependent opinion, and sometimes unavoidable, but it's still a pain. Clamping how far up and down you can look in an FPS is fine. But managing Eulers for handling Yaw in a Top-Down/3rd Person game feels wrong to me.

## Properties of Quaternions

### Unit Quaternions (Normalized)

Unit quaternions have a \|\|magnitude\|\| of 1. This means if you square all four components and add them together, it should equal 1. Techincally should square-root it too, but $\sqrt{1} = 1$, so I'm cutting corners. Within Unity, quaternions get normalized by default. There does exist a `Normalize` method in case you're manually entering quaternion values and it, for some reason, doesn't auto-normalize (it can happen). But usually you shouldn't have to worry about that. Non-Unit Quaternions are more complicated things that I'm just not going to get into.

### Identity ("1")

Identity just means "no change". It's like multiplying regular numbers by `1`, or adding `0`. When you set an object's orientation to `Quaternion.identity`, it goes to the default orientation, which represents an absence of any rotations applied (i.e. 0° on all Euler Angles).

<code>Q<sub>I</sub> = new Quaternion(0, 0, 0, 1);</code>

### Invertability (Ctrl + Z)

The Inverse of a quaternion `Q` is denoted as <code>Q<sup>-1</sup></code>. Any rotation that is applied can be undone. A quaternion will cancel out with its Inverse, resulting in the Identity quaternion. Thinking in terms of *Axis and Angles*, you can view it as negating the axis to point in the opposite direction, while keeping the same rotational angle. Another way to think about it is just negating the angle to rotate in the opposite direction. Mathematically speaking:

<code>Q<sup>-1</sup> = new Quaternion(-Q.x, -Q.y, -Q.z, Q.w);</code>

<code>Q * Q<sup>-1</sup> = Q<sup>-1</sup> * Q = Q<sub>I</sub></code> (Identity)

Note: There also exists a quaternion Conjugate, <code>Q<sup>*</sup></code>, where the only difference is that Conjugate keeps the original magnitude and the Inverse is just the Conjugate divided by the square-magnitude. But since we're working with Unit quaternions, the Inverse and Conjugate will be the same for our purposes, so I purposefully omitted any division operation for the Inverse.

### Associativity (Parentheses order doesn't matter)

Given quaternions <code>Q<sub>1</sub></code>, <code>Q<sub>2</sub></code>, and <code>Q<sub>3</sub></code>, associativity states that:

<code>(Q<sub>1</sub> * Q<sub>2</sub>) * Q<sub>3</sub> = Q<sub>1</sub> * (Q<sub>2</sub> * Q<sub>3</sub>)</code>

Same applies when multiplying with a vector `V`:

<code>(Q<sub>1</sub> * Q<sub>2</sub>) * V = Q<sub>1</sub> * (Q<sub>2</sub> * V)</code>

### Non-Commutativity (Multiplcation order matters)

Given quaternions <code>Q<sub>1</sub></code> and <code>Q<sub>2</sub></code>, non-commutativity states that:

<code>Q<sub>1</sub> * Q<sub>2</sub></code> is **not always** the same as <code>Q<sub>2</sub> * Q<sub>1</sub></code>

There are *some cases* where they will just happen to be equal. Like if at least one of them is the Identity quaternion, if one is the Inverse of the other, or some other exceptions involving symmetry.

When multiplying quaternions together, the best way to think about it is from **right-to-left** (I know, it seems backwards). Given <code>Q<sub>1</sub> * Q<sub>2</sub> * V</code>: start with the Vector direction, imagine rotating it *first* by <code>Q<sub>2</sub></code>, *then* rotate that resulting vector by <code>Q<sub>1</sub></code>. Order of operations is important to visualize the rotations. Alternatively, if you prefer left-to-right, you could try to visualize the result of <code>Q<sub>1</sub> * Q<sub>2</sub></code> first, but I find that's generally harder.

### A "Complete" Rotation is 720°, Not Just 360°

{{<video src="quaternion-spin.webm" autoplay="true" loop="true">}}

By "complete", I'm referring to an orientation returning to its starting *state*. If you track one face of the cube in the video (or gif at the top), you'll see how this works where there are two different "states" even when the face of the cube is oriented the same way.

This is how it works in real life, too: Electrons and other matter particles in quantum mechanics have this property called "spin". These two similar-but-different states are what physicists are referring to when they say "spin up" or "spin down".

## Unity Methods

### Quaternion Creation

- [Euler](https://docs.unity3d.com/ScriptReference/Quaternion.Euler.html): Rotation that rotates z° around the `z` axis, x° around the `x` axis, and y° around the `y` axis. **In that order**
    - Can access these values with `transform.eulerAngles` - not to be confused with `transform.rotation`
    - If you're typically setting two of the three values to `0°`, you can accomplish the same thing with AngleAxis instead
- [AngleAxis](https://docs.unity3d.com/ScriptReference/Quaternion.AngleAxis.html): Are you thinking in terms of axis and angles yet?
    - You can create most of the same quaternions you'd probably make with Euler with this instead
    - This method is ***FASTER***, too! With a simple test of 100,000 calls every frame, AngleAxis outperformed Euler with 30-50% more FPS. Ditch using Euler; it's a more expensive operation
- [LookRotation](https://docs.unity3d.com/ScriptReference/Quaternion.LookRotation.html): Get an orientation by providing a `Vector3 forward` and a `Vector3 upwards`. The upwards vector (default: `Vector3.up`) is used as a "hint" to determine how to orient the Roll
    - Usually you'll get a forward direction by doing `target.position - transform.position`
    - Make sure your forward vector won't directly align with the upward vector. Provide a new upward hint if that will be a problem
    - <details><summary>Example Analogy</summary>You know how dogs do that cute little head-tilt when they're confused? Their eyes are still locked onto their target (you, probably after saying "treat" or "walk" to pique their interest) looking "forward". But their head's "up" reference is changing, no longer aligned with <code>Vector3.up</code></details>
- [FromToRotation](https://docs.unity3d.com/ScriptReference/Quaternion.FromToRotation.html): This gets the difference in orientation between two direction vectors; the *rotation* that you would need to apply to `fromDirection` which will result in the `toDirection`.
    - In other words, this solves for <code>Q<sub>FromTo</sub></code> in the equation <code>Q<sub>FromTo</sub> * V<sub>From</sub> = V<sub>To</sub></code>
    - This method was vital for my Roll-a-Tetrahedron solution
- [Inverse](https://docs.unity3d.com/ScriptReference/Quaternion.Inverse.html): Gets the ~~evil twin~~ opposite rotation of a quaternion.
    - Example: if you wanted to get <code>Q<sub>ToFrom</sub></code>, you *could* call FromToRotation again swapping parameters or you could just do <code>Q<sub>ToFrom</sub> = Quaternion.Inverse(Q<sub>FromTo</sub>);</code>
- The [Quaternion Constructor](https://docs.unity3d.com/ScriptReference/Quaternion-ctor.html): this takes in 4 floats `x`,`y`,`z`, and `w`. You usually won't use this. If you are using this, it's for one of the following reasons:
    - deserialization
    - optimizations (e.g. swizzling via extension methods)
    - you understand quaternions better than most
    - you copied code you found somewhere

### Quaternion Interpolation

When interpolating between a start and end orientation, Quaternions take the shortest path (this means that angles more than 180° apart aren't a thing). Try to avoid having your start and end 180° apart (pointing in opposite directions), otherwise the path between them will not be well-defined.

#### Time- or Percent-Based Methods
- [Lerp](https://docs.unity3d.com/ScriptReference/Quaternion.Lerp.html): Linear interpolation between [0-1]
- [Slerp](https://docs.unity3d.com/ScriptReference/Quaternion.Slerp.html): Spherical interpolation between [0-1]
- [LerpUnclamped](https://docs.unity3d.com/ScriptReference/Quaternion.LerpUnclamped.html) & [SlerpUnclamped](https://docs.unity3d.com/ScriptReference/Quaternion.SlerpUnclamped.html): can also extrapolate beyond 0 and 1

<details><summary>What's the difference between Lerp and Slerp? Aren't they both spherical since it involves things that rotate?</summary>
Yeah, not much difference tbh. Unlike the differences between Vector3's Lerp and Slerp, these all follow the same path but just have an ever-so-slightly different easing/timing along the path. The docs say that "[Lerp] is faster than Slerp but looks worse if the rotations are far apart." But it's hard to see a difference</details>

<details><summary>Do LerpUnclamped and SlerpUnclamped behave the same?</summary>
<b>No</b>. Because of how the math works with extrapolating and quaternions getting auto-normalized, LerpUnclamped "fizzles out" at certain values outside the 0-1 range, like diminishing returns. Unsatisfying.<br>In my tests, if the angle between the start and end orientations are less than ~36° apart, then SlerpUnclamped <i>also</i> behaves like LerpUnclamped. However, at larger starting angles, SlerpUnclamped shines and will properly extrapolate.<br>For example, if you wanted to extrapolate the Minute Hand on a Clock, your start and end orientations should be at least 6 minutes apart, but less than 30 minutes so that <i>forward</i> in time doesn't go the wrong way
</details>

#### Speed-Based Method
- [RotateTowards](https://docs.unity3d.com/ScriptReference/Quaternion.RotateTowards.html): Get an orientation that is the `from` quaternion rotated by a `float maxDegreesDelta` angle towards the `to` quaternion without overshooting
    - If you use a `float rotationSpeed` multiplied by `Time.deltaTime`, it will rotate at that **rate** in degrees per second, no matter the difference between `from` and `to`

### Transform Options for Rotating

Quaternions by theirself are just a math construct - they have no idea what a Transform is. With the Transform class, you have access to more contextual utility, like methods that know the difference between global and local.

- [Transform.Rotate](https://docs.unity3d.com/ScriptReference/Transform.Rotate.html): This method has many overlaod options
    - Primarily relies on Euler Angles or an Axis and Angle combo
    - Can specify whether it's a global (`Space.World`) or local (`Space.Self`) rotation (default)
- [Transform.RotateAround](https://docs.unity3d.com/ScriptReference/Transform.RotateAround.html): Like a satellite orbiting around a planet or like a hinge joint, this method **also affects the position**
    - Uses an Axis and Angle and a world position `Vector3 point` as a pivot or anchor that the axis passes through
- [Transform.LookAt](https://docs.unity3d.com/ScriptReference/Transform.LookAt.html): Immediately snap a transform's forward to a `Transform target` or `Vector3 worldPosition`
    - Similar to `Quaternion.LookRotation`, this also can take a `Vector3 worldUp` hint
    - For 2D games, since this defaults to using the transform's forward, you'll either want to adjust the `worldUp` to have it orient towards a target (leaving the `target` vector directly forward of your character) OR change how your sprite is parented/oriented
- Applying a `Vector3 direction` directly to `transform.`[`forward`](https://docs.unity3d.com/ScriptReference/Transform-forward.html)/[`up`](https://docs.unity3d.com/ScriptReference/Transform-up.html)/[`right`](https://docs.unity3d.com/ScriptReference/Transform-right.html)
    - This approach doesn't give control over where the other axes choose to align, so use cautiously or only in very simple cases
- Applying a `Quaternion rotation` directly to [`transform.rotation`](https://docs.unity3d.com/ScriptReference/Transform-rotation.html) or [`transform.localRotation`](https://docs.unity3d.com/ScriptReference/Transform-localRotation.html)
    - Tip: If you have `someRotation` you want to *apply* to your transform's orientation, **don't do** `transform.rotation *= someRotation;`! Instead, do `transform.rotation = someRotation * transform.rotation;`. Order matters

### Quaternion Methods You'll Never or Rarely Use

- [Normalize](https://docs.unity3d.com/ScriptReference/Quaternion.Normalize.html): Set the magnitude of a quaternion to 1, keeping its orientation. Unity does this by default in most cases
- [Dot](https://docs.unity3d.com/ScriptReference/Quaternion.Dot.html): Returns the Dot Product between two quaternions, a value between -1 and +1 as a measure of "alignment". Easier to use Vectors
- [Angle](https://docs.unity3d.com/ScriptReference/Quaternion.Angle.html): Returns the angle in degrees between two quaternions. Easier to use Vectors
- [ToAngleAxis](https://docs.unity3d.com/ScriptReference/Quaternion.ToAngleAxis.html): Gets an Angle and Axis from a quaternion. Usually you're using that kind of info to create quaternions, not the other way around
    - Fun fact: the axis that gets returned from `Quaternion.identity` is `Vector3.right`, or (1, 0, 0)

## Math

Quaternions consist of four numbers: `x`, `y`, `z`, and `w`\*, which are all values between -1 and +1. But that still leaves us with questions:
- Should we try to visualize it geometrically?
- What do those numbers mean?
- How does multiplying with them work?

\**Note: I'll be placing `w` at the end to match with Unity; other sources may put the `w` at the beginning.*

### Geometric Interpretation

It can be hard for people to visualize a 4-dimensional struct. Imagine a coordinate system that consists of 4 axes. By definition, all four axes are perpendicular to each other. Oh, and three of the four axes are imaginary.

That's not very helpful or intuitive.

Instead, it's easier to imagine a quaternion in two parts: a Vector3 (using `x`, `y`, and `z`) with a certain amount of *spin* or *twist* about itself (the `w` component). This goes back to thinking in terms of Axis and Angle. The trickier part comes when trying to interpret how the `w` value relates to an actual angular value.

### Imaginary Numbers

The magical rotational properties stem from the usage of imaginary numbers: $i = \sqrt{-1}$

Brief refresher on the Complex Plane, $\mathbb{C}$: it's a 2D grid, where the horizontal axis consists of real numbers, $\mathbb{R}$, and the vertical axis consists of imaginary numbers, $\mathbb{I}$. A point in the grid is called a complex number, written as `a + bi`. Any time you multiply a complex number by `i`, it's like rotating that point 90° counter-clockwise about the origin. If you do that 4 times, you're back to where you started: `1` → `i` → `-1` → `-i` → `1`

Let's extend the Complex Plane by adding two more imaginary dimensions: `j` and `k`. Both `j` and `k` also have the rotational superpowers that come from $\sqrt{-1}$. All four axes (`1`, `i`, `j`, and `k`) are orthogonal to one another, forming a "basis" in our 4D space. We can represent a quaternion in the form:

$$ x\mathbb{i} + y\mathbb{j} + z\mathbb{k} + w $$

It's worth pointing out that the *real axis* where `w` occupies has a unit value of `1`. It's just omitted in the formula for convenience: $... + w*1$. This will be useful info later when we're [multiplying quaternions](#qq-multiplication).

To make sense of how multiplication with these new axes works, we need to add a few special rules. And remember: the order of multiplication **matters**. The value on the left is being *applied* to the one on the right:
- `i * j = k`,&ensp; `j * i = -k`
- `j * k = i`,&ensp; `k * j = -i`
- `k * i = j`,&ensp; `i * k = -j`<br>
[Click here for an interactive visualization of these rules](https://upload.wikimedia.org/wikipedia/commons/0/04/Cayley_Q8_quaternion_multiplication_graph.svg)

Now we can make sense of what Sir Hamilton etched into stone (see quote at top of this article):
- <code>i<sup>2</sup> = j<sup>2</sup> = k<sup>2</sup> = ijk = -1</code>

For that `ijk` part, whether you do the multiplication like `(i * j) * k` or `i * (j * k)`, you can use the rules to replace what's in the parentheses and you'll get `(k) * k = -1` or `i * (i) = -1`, respectively. Yay associativity!

### The Real Part: w

Since `w` is a value between -1 and +1, how can we map that from/to an angle of rotation?

Answer: $ w = \cos{(\theta / 2)} $, and $ \theta = 2 * \arccos{w} $

When `w` is 1, `x`, `y`, and `z` will be 0 due to normalization constraints. This is the same as `Quaternion.identity`, where the angle of rotation is 0°.

Although a `w` value of 0 may seem insignificant, it actually represents the **most extreme rotation**: 180°.

What about when `w` is -1? The other values will be forced to be 0 just as before because normalization, but the angle is actually 360°.
<details><summary>Is this last case the same as <code>Quaternion.identity</code>?</summary>
<b>No.</b> When applied to a 3D model, it will <i>look</i> the exact same, the <code>Angle</code> difference between this and Identity will be 0°, but an equality check will return false. Remember: <a href="#a-complete-rotation-is-720-not-just-360">A "Complete" Rotation is 720°, Not Just 360°</a>. If you applied this <i>twice</i>, then it will be equal to the Identity. In a way, you could consider this to be the <a href="https://en.wikipedia.org/wiki/Root_of_unity" target="_blank">2<sup>nd</sup> root</a> of the Identity.</details>

### From Axis & Angle to Quaternion

This is to demonstrate how to do `var q = Quaternion.AngleAxis(angle, axis);` if you didn't have access to that Unity method

```csharp
// 1. First define your axis and your angle
var axis = new Vector3(4f, 20f, 69f);
var angle = 100f;

// 2. Get the w from the angle. Remember to convert to radians
var w = Mathf.Cos(angle * Mathf.Deg2Rad / 2f);

// 3. Get the downscaling factor that we'll apply to the vector
var downscale = Mathf.Sin(angle * Mathf.Deg2Rad / 2f);

// 4. Downscaling the vector ensures all 4 values combined will be normalized
var xyz = axis.normalized * downscale;

// 5. Make your Quaternion
var q = new Quaternion(xyz.x, xyz.y, xyz.z, w);
```

### From Quaternion to Axis & Angle

This is to demonstrate how to do <code>q.ToAngleAxis(out var angle, out var axis);</code> if you didn't have access to that Unity method

```csharp
// 1. Assuming we already have a quaternion 'q' defined, get the angle in degrees
var angle = 2f * Mathf.Rad2Deg * Mathf.Acos(q.w);

// 2. Extract the vector portion. Normalize it just because
var axis = new Vector3(q.x, q.y, q.z).normalized;

// That's it. Return those or assign to out parameter
```

### Q*Q Multiplication

If you know the basics of multiplication, addition, and subtraction, you can perform quaternion multiplication!

Since we know the form of a quaternion, and we're only plugging in the values for `x`, `y`, `z`, and `w`, we can set up a multiplication table for the axes and use the special rules from earlier to simplify the math:

|     | `i` | `j` | `k` | `1` |
| --- | :-: | :-: | :-: | :-: |
| `i` | -1  |  k  | -j  |  i  |
| `j` | -k  | -1  |  i  |  j  |
| `k` |  j  | -i  | -1  |  k  |
| `1` |  i  |  j  |  k  |  1  |

For <code>Q<sub>A</sub> * Q<sub>B</sub></code>, <code>Q<sub>A</sub></code> would be along the left column, and <code>Q<sub>B</sub></code> would be along the top row. If you swap them, you'll just have some minus signs in the wrong spots.

<b>Step-by-Step Example</b>

In this example, imagine we're looking at a computer monitor slightly to our right at 30° and want to turn our neck left by 120°.<br>
<code>Q<sub>A</sub> = 0i - 0.866j + 0k + 0.5</code> ← rotation same as `Quaternion.AngleAxis(-120f, Vector3.up)`<br>
<code>Q<sub>B</sub> = 0i + 0.2588j + 0k + 0.9659</code> ← orientation same as `Quaternion.AngleAxis(30, Vector3.up)`<br>
Use a matrix table to perform <code>Q<sub>A</sub> * Q<sub>B</sub></code>

|     | `0 i` | `0.2588 j` | `0 k` | `0.9659` |
| --- | :-: | :-: | :-: | :-: |
| `0 i` | 0  |  0  | 0  |  0  |
| `-0.866 j` | 0  | 0.22414  |  0  |  -0.8365 j  |
| `0 k` |  0  | 0  | 0  |  0  |
| `0.5` |  0  |  0.1294 j  |  0  |  0.48296  |

Combinining like terms gives us: <code>Q<sub>B</sub>' = 0i - 0.7071j + 0k + 0.7071</code>

Converting that [quaternion to an Angle and Axis](#from-quaternion-to-axis--angle) tells us it's a 90° rotation about the negative Y-axis. Using the left-hand-rule, this orientation represents looking directly left. We did it! That wasn't too bad.

### Q*V Multiplication

I know I wrote `Q * V`, but *actually* we have to do <code>Q * V * Q<sup>-1</sup></code>, *sandwiching* the vector between the quaterion and its inverse\*. Yummy. This is to ensure the `w` value gets properly canceled out so that we're left with a Vector in the end and not actually a quaternion.

\*<i>Note: In Unity, you can't perform `V * Q` in that order. This math section is how to solve it **on paper**, and it's more-or-less what happens behind the scenes when you do a `Q * V` operation in Unity.</i>

Not the best analogy, but here's one way to think about it the sandwiching: Imagine wringing out a wet towel. Both hands start facing the same direction as each other, palms down gripping the towel. One hand twists the towel in one direction 180° (`Q`). The other hand twists it 180° in the opposite direction (<code>Q<sup>-1</sup></code>). Both hands end up still facing the same way as each other, but the towel (`V`) ends up twisted (rotated) and slightly less soaked.

So how do we multiply something that lives in 3D space by something that's 4D? They live in very different spaces - real world vs complex imaginary land. **Here's the trick**:

1. We *pretend* our vector is actually 4D like a quaternion, setting the `w` value to 0. We don't do any normalzing to it. We'll call it <code>V<sub>4D</sub></code>
2. We do the same quaternion multiplication as before but just **twice**. It doesn't matter which pair you multiply first:  <code>(Q * V<sub>4D</sub>) * Q<sup>-1</sup></code> = <code>Q * (V<sub>4D</sub> * Q<sup>-1</sup>)</code>
3. After all the multiplication math dust settles, if we did it right, its `w` will end up as 0 and so we drop that real axis. We also drop the imaginary labels `i`, `j`, and `k` from it to convert it back to a regular Vector, now rotated

**Longer Step-by-Step Example**

In this example, let's start with a `(7, 7, 0)` vector and try to rotate that counter-clockwise by 45°. Relatively simple 2D example that you could solve using the [Rotation Matrix](https://en.wikipedia.org/wiki/Rotation_matrix), but let's demonstrate what it looks like with quaternions<br>
`V = 7x + 7y + 0z`<br>
`Q = 0i + 0j + 0.38268k + 0.92388` ← rotation same as `Quaternion.AngleAxis(45f, Vector3.forward)`<br>
<code>Q<sup>-1</sup> = 0i + 0j - 0.38268k + 0.92388</code> ← Inverse to sandwich with<br>
Let's pretend that V is a quaternion. I'm choosing to perform <code>V<sub>4D</sub> * Q<sup>-1</sup></code> first

|     | `0 i` | `0 j` | `-0.38268 k` | `0.92388` |
| --- | :-: | :-: | :-: | :-: |
| `7 i` | 0  |  0  | 2.67876 j  |  6.46716 i  |
| `7 j` | 0  | 0  |  -2.67876 i  |  6.46716 j  |
| `0 k` |  0  | 0  | 0  |  0  |
| `0` |  0  |  0  |  0  |  0  |

Combinining like terms gives us <code>V<sub>4D</sub>' = 3.7884i + 9.14592j + 0k + 0</code>

Half-way there. Now we multiply <code>Q * V<sub>4D</sub>'</code>

|     | `3.7884 i` | `9.14592 j` | `0 k` | `0` |
| --- | :-: | :-: | :-: | :-: |
| `0 i` | 0  |  0  | 0  |  0  |
| `0 j` | 0  | 0  |  0  |  0  |
| `0.38268 k` |  1.45 j  | -3.5 i  | 0  |  0  |
| `0.92388` |  3.5 i  |  8.45 j  |  0  |  0  |

Simplifying one last time gives us <code>V<sub>4D</sub>' = 0i + 9.9j + 0k + 0</code>

Converting the final result back into 3D world space gives us a vector of `(0, 9.9, 0)`. Awesome!

## Conclusion

Visualizing the rotations and getting a solid point of reference (local, global, relative to something else?) can be daunting. But now you know the essentials and to think about them in terms of and **axis** and an **angle**, so go out there and practice with them! Think about what sort of scenarios would be suitable for different methods. Don't multiply them in the wrong order. Level up beyond the pedestrian Euler angles. Being able to use Quaternions is an invaluable tool to have as a gamedev. And they're pretty nifty too :thumbsup:

## Bonus Material and Resources

### FAQ

<details><summary>I want to interpolate between a start and end orientation, but with the <b>long/outer</b> path. How can I achieve this?</summary>
As with anything program-related, there are lots of potential solutions. Here is one approach using <code>SlerpUnclamped</code>:
<pre><code>public Quaternion SlerpLongPath(Quaternion from, Quaternion to, float t)
{
    var angle = Quaternion.Angle(from, to); // angle along short path
    if (angle == 0) return from; // avoid divide-by-zero
    float adjustedT = t * (angle - 360f) / angle; // remaps t from [0,1] to [0,-N]
    return Quaternion.SlerpUnclamped(from, to, adjustedT);
}
</code></pre>
<details><summary>Pop Quiz: In what situation will the above code <b>not</b> give desired results?</summary>
Answer: When the Angle between <code>from</code> and <code>to</code> is less than ~36°. See <a href="#quaternion-interpolation">Quaternion Interpolation</a> for details</details>
If that's going to be a problem, here is a different approach that uses a mid-point that we flip around:
<pre><code>public Quaternion SlerpLongPath(Quaternion from, Quaternion to, float t)
{
    var midRot = Quaternion.Slerp(from, to, 0.5f); // orientation along short path
    midRot.ToAngleAxis(out var midAngle, out var midAxis); // niche use for this method
    var midRotLong = Quaternion.AngleAxis(midAngle + 180f, midAxis); // opposite dir of short path
    if (t < 0.5f) return Quaternion.Slerp(from, midRotLong, 2 * t);
    return Quaternion.Slerp(midRotLong, to, 2 * t - 1);
}
</code></pre>
With this approach, if <code>from</code> and <code>to</code> are identical or exactly 180° apart, the long path is not strongly defined so results may vary
</details>

### Videos

- [*Visualizing quaternions*](https://www.youtube.com/watch?v=d4EgbgTm0Bg) (31:50) - 3Blue1Brown, part 1
- [*Quaternions and 3d rotation*](https://www.youtube.com/watch?v=zjMuIxRvygQ) (5:58) - 3Blue1Brown, part 2
- [*How quaternions produce 3D rotation*](https://www.youtube.com/watch?v=jTgdKoQv738) (11:34) - PenguinMaths

### Reading

- [*How Quaternions Produce 3D Rotations*](https://penguinmaths.blogspot.com/2019/06/how-quaternions-produce-3d-rotation.html) - PenguinMaths
- [*Rotation and orientation in Unity*](https://docs.unity3d.com/Manual/QuaternionAndEulerRotationsInUnity.html) - Unity Docs. Short 'n sweet
- [*Quaternion*](https://en.wikipedia.org/wiki/Quaternion) - Wikipedia

### Further Reading

- [*Let's remove Quaternions from every 3D Engine*](https://marctenbosch.com/quaternions/) - Marc ten Bosch. Interesting read. Suggests Rotors
- [*Conversion between quaternions and Euler angles*](https://en.wikipedia.org/wiki/Conversion_between_quaternions_and_Euler_angles) - Wikipedia. Did not cover this. Heavy math
