---
title: Roll A Tetrahedron
description: A guide appears in front of you. Roll 1d4 for perception.
date: 2023-02-01 12:00:00 -0700
categories: [Tutorial]
tags: [unity,programming,c#]
---

A member of a Discord server I moderate was curious about how to roll a tetrahedron. And that intrigued me. It's such a unique problem.
They wanted something similar to [Tarodev's "Roll-A-Cube" video](https://www.youtube.com/watch?v=V9rVZ9mf0uA).

Searching for a pre-existing solution to see if this had been done before turned up nothing.
So I got to work.
Engineering a solution to a unique problem involves trial and error,
and I've purposefully included any non-working attempts so that you can follow along with the process.

For my initial approach, I'll try modifying the Roll-A-Cube code, which uses the [`RotateAround`](https://docs.unity3d.com/ScriptReference/Transform.RotateAround.html) method.
But before we get to that, we'll need a 3D model to work with.

## 3D Tetrahedron Model

The requirements I set for myself were the following:

- Edge length of 1 unit (a.k.a. "unit tetrahedron").
- Bottom face is level with the XZ-plane (y = 0).\*
- Top vertex is centered directly above the origin.
- Pointy end forward (though this can easily also point right, not a hard-set constraint).
- Precision is important so I can't rely on downloading someone else's model.

\**Note: This was just for easier math; we will offset it later so the center is at the origin.*

With this setup, I did the math (I will leave the fun of trigonometric computations to the reader) and came out with the following 4 vertices:
```
Back-left: (-0.5, 0, -0.288675)  // z = -sqrt(3)/6
Back-right: (0.5, 0, -0.288675)
Forward:    (0, 0, 0.577350)     // z = 1/sqrt(3)
Top:        (0, 0.816497, 0)     // y = sqrt(2/3)
```

All 4 vertices are `1` unit away from each other and `sqrt(3/8)` units away from the center:

```
 Center is at: (0, 0.2041241, 0) // y = sqrt(6)/12
```

If you want to try deriving it for yourself, I also recommend a linear algebra approach by setting up a few system of equations to solve for the unknowns.
Alternatively, you can check out the [Tetrahedron Wikipedia page](https://en.wikipedia.org/wiki/Tetrahedron#Coordinates_for_a_regular_tetrahedron) for examples, and apply your desired scaling and offset.


**Happy rolling!**
