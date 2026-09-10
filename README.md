# Fold Depth

A foldable phone's defocus, in CSS. Two display halves on a real 3D hinge; as
the hinge opens the blur resolves, reaching exactly zero when the panel lies
flat.

**[Live demo](https://pwmikolajek.github.io/fold-depth/)** · drag either rim up
or down to work the hinge, or scrub the controls. The page generates the CSS for
whatever state you leave it in.

## The idea

The blur is not a property of the open display. It belongs to the hinge. Each
half is tilted out of the focal plane by

```
φ = 90° − θ/2          θ = the angle between the halves
```

so a point `x` from the hinge sits at depth `x·sin(φ)`, and its circle of
confusion grows with that depth. One angle, one factor:

```
σ ∝ sin(φ)            ( = cos(θ/2) )
```

`sin(φ)` does double duty — it is both how far the surface has travelled off the
focal plane and what that costs in sharpness. At θ=180° it is exactly 0, so the
display is bit-for-bit sharp. CSS can evaluate the trigonometry, so the whole
effect takes a single input:

```css
@property --hinge { syntax: "<angle>";  inherits: true; initial-value: 180deg }
@property --phi   { syntax: "<angle>";  inherits: true; initial-value: 0deg   }
@property --depth { syntax: "<number>"; inherits: true; initial-value: 0      }

.book {
  --hinge: 90deg;
  --phi:   calc(90deg - var(--hinge) / 2);
  --depth: sin(var(--phi));

  transform-style: preserve-3d;
  transform: translateZ(calc(-25em * sin(var(--phi))));
}
.leaf[data-side="l"] { transform-origin: 100% 50%; transform: rotateY(var(--phi))            }
.leaf[data-side="r"] { transform-origin: 0    50%; transform: rotateY(calc(-1 * var(--phi))) }
```

The book is pulled back by `25em·sin(φ)` so the fold pivots about its own centre
of mass instead of lunging at the viewer.

## Building a variable-radius blur

A Gaussian blur has exactly one radius. There is no `blur(0px 40px)`, and
masking a single blurred copy only cross-fades between two fixed radii — you can
see the seam. So the falloff is assembled from a stack of six `backdrop-filter`
veils over each half. Each blurs ~1.8× harder than the one beneath it and each
is masked to *begin* further out. Every veil filters the accumulated backdrop
below it, so the radii compound in quadrature (σ = √Σσᵢ²) and the ramp is
continuous.

Two numbers describe a veil, both measured outward from the hinge. `1em` is 1%
of the display width, so the whole effect scales with the panel:

| veil | `--r` (radius) | `--A` (core ends) | `--B` (full strength) |
|-----:|---------------:|------------------:|----------------------:|
| 1 | 0.28em |  8em | 21em |
| 2 | 0.50em | 14em | 27em |
| 3 | 0.90em | 20em | 33em |
| 4 | 1.60em | 26em | 39em |
| 5 | 2.90em | 32em | 45em |
| 6 | 5.20em | 38em | 50em |

One "field function" turns `--A`/`--B` into a mask: a linear gradient bends the
surface like a cylinder, a radial one like a dome.

## The Chrome trap worth knowing

**Chrome silently drops a `backdrop-filter` element's mask when the container it
sits in is clipped to an _asymmetric_ rounded rect.** The blur then applies to
the whole box, with no error and no warning.

A folded half wants exactly that shape — rounded on its two outer corners,
square where it meets the hinge. Writing it the obvious way

```css
/* both of these break every veil mask inside */
.face { border-radius: 3.4em 0 0 3.4em; overflow: hidden }
.face { clip-path: inset(0 round 3.4em 0 0 3.4em) }
```

leaves you with a uniformly blurred panel. A *uniform* radius is fine, so the
fix is to keep one and crop the difference:

```css
.crop { position: absolute; inset: 0; overflow: hidden }   /* plain rectangle */
.face { border-radius: var(--R); overflow: hidden }        /* uniform */
[data-side="l"] .face { left: 0;  right: calc(-1 * var(--R)) }
[data-side="r"] .face { right: 0; left:  calc(-1 * var(--R)) }
```

Each half overhangs the hinge by one radius, so its inner corners are rounded
outside the visible area and `.crop` shears that side off square. The mask stops
are offset by `--R` to compensate.

Two other things that look like the same bug but are not: an ancestor `filter`
is harmless (this build uses one on each leaf to carry the key light), and so is
backdrop content that overflows the veil's own box.

One more, at the other end of the range. At exactly 180° every fold transform is
identity and the blur radii are all zero, but Chrome still composites the twelve
no-op `blur(0px)` layers and the two abutting leaves, and the antialiased edges
leave a hairline down the hinge — visible at 180° and at no other angle. Since a
flat foldable is not two panels, the demo stops drawing it as two: below half a
degree of tilt the veils, the crease, the lighting filters and the transforms are
all dropped, and one leaf is stretched to carry the whole display.

```css
.stage.flat .veil, .stage.flat .crease { display: none }
.stage.flat .leaf { filter: none; transform: none }
.stage.flat [data-side="l"] .crop { right: -100% }
.stage.flat [data-side="r"] .crop { display: none }
```


## Credits

Photograph by [George Cox](https://unsplash.com/photos/NFpvXe7sSJM) via
Unsplash, embedded in the page as a data URI so `index.html` stands alone.

Built by Paweł Mikołajek · [humanmade.com](https://humanmade.com)
