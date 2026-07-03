+++ 
draft = false
date = 2026-07-01T16:35:26+01:00
title = "AceX - A minimal ACES 2.0 Fitted Curve."
description = "A single-function, LUT-free tonemapper that fits the real ACES 2.0 SDR tonescale and adds a cheap hue-preserving path to white. Drop-in GLSL and HLSL."
slug = "acex-tonemapper"
authors = []
tags = ["shaders", "tonemapping", "color-science", "aces", "glsl", "hlsl"]
categories = [Tone Mapping Operator]
externalLink = ""
series = []
+++

[AceX Colour Sweep](/images/Sweep.png)

```glsl
float AceXCurve(float x){
    float f = 1.0471038 * pow(max(0.0, x) / (x + 0.9198583), 1.15);
    return f * f / (f + 0.04);
}

vec3 AceX(vec3 x){
    float Y  = max(x.r, max(x.g, x.b));
    float Yt = AceXCurve(Y);
    vec3  c  = x * (Yt / max(Y, 1e-5));

    float nJ = clamp(Yt, 0.0, 1.0);
    c = mix(c, vec3(Yt), nJ * nJ * 0.85);

    return clamp(c, 0.0, 1.0);
}
```

[AceX Fitted Curve](/images/acex_fit_light.png)

If you have ever pasted the infamous Narkowicz `ACESFilm` fit into your post-processing shaders, you already know the appeal: one function, no lookup tables, and a filmic roll-off that stops your highlights from screaming. That fit approximates **ACES 1.0**, which is now several years behind the times. ACES 2.0 shipped a completely rebuilt output transform with a nicer tonescale and, crucially, hue handling that no longer skews your blues to cyan and your reds to orange on the way to white.

The catch is that ACES 2.0 is not a one-liner. It converts into a perceptual colour appearance model, tonemaps a lightness correlate, runs a hue-dependent chroma compression, then gamut-maps against per-hue lookup tables. Wonderful for a colour pipeline, far too heavy for a lightweight post pass.

**AceX** is the pragmatic middle ground. It takes the *actual* ACES 2.0 SDR tonescale, fitted to a compact closed form, and pairs it with a cheap hue-preserving path to white. One curve, one operator, no textures.

## Be clear about what this is

AceX is not ACES 2.0. It is an homage that gets you most of the way for a fraction of the cost. The **tonescale is exact**: it is the ACES 2.0 Daniele Evo curve specialised to the 100-nit SDR case, and 18% grey lands on 10 nits just as the real transform intends. The **path to white is an approximation**: instead of the real chroma compression in appearance space, AceX preserves hue by holding RGB ratios and desaturates toward the tonemapped achromatic value in the highlights. Neutrals and muted material match the real thing closely. Very saturated highlights will differ, because the real hue trajectories come from tables AceX does not carry.

If you need a bit-exact ACES 2.0, bake a 3D LUT from the reference transform. If you want something that looks right, runs everywhere, and fits in a code block, read on.

## The curve

Here is the whole tonescale. Three lines.

```glsl
float AceXCurve(float x){
    float f = 1.0471038 * pow(max(0.0, x) / (x + 0.9198583), 1.15);
    return f * f / (f + 0.04);
}
```

Those constants are not tuned by eye. They fall directly out of the ACES 2.0 tonescale initialisation for a 100-nit peak, collapsed to numbers. The fit tracks the reference curve to within about one part in a thousand in display code value across the entire range, and `AceXCurve(0.18)` returns `0.10` on the nose.

A nice thing to notice: this is a Reinhard relative. The inner term `(x / (x + s))^g` is a generalised Reinhard, and the outer `f² / (f + t)` is a second rational that adds the shadow toe. On a linear axis it looks like Reinhard; the S-curve only appears when you plot against stops. That is a property of the axis, not a different function.

## The operator

The curve is a 1D function. To tonemap colour you need to decide *what* to run through it and *how* to bring the colour back. AceX runs the achromatic peak channel through the curve, rescales the pixel to preserve hue, then rolls the highlights to white.

```glsl
vec3 AceX(vec3 x){

    // 1. tonescale on the achromatic peak channel
    float Y  = max(x.r, max(x.g, x.b));
    float Yt = AceXCurve(Y);
    vec3  c  = x * (Yt / max(Y, 1e-5));

    // 2. hue-preserving path to white
    float nJ = clamp(Yt, 0.0, 1.0);
    c = mix(c, vec3(Yt), nJ * nJ * 0.85);

    return clamp(c, 0.0, 1.0);
}
```

That is the entire operator. It returns **display-linear** values in `[0, 1]`.

## Where to apply it

AceX works on **scene-linear** light. Not log, not display-encoded. This trips people up because the plot is easiest to read on a logarithmic axis, but that log is only a way to draw the graph. The shader wants radiometric linear values where 0.18 is mid grey, roughly 1.0 is diffuse white, and highlights run above 1.0.

```
1. sample HDR linear texture
2. AceX(rgb)          // runs in linear, returns display-linear 0..1
3. display encode     // sRGB OETF, or pow(x, 1.0/2.2) as a quick stand-in
```

Feed it a log signal and the curve will read that log shape as if it were linear light and crush everything. If your footage is log-encoded, decode to linear first.

## Scope and caveats

- **SDR, 100-nit.** The constants are specialised to a 100-nit peak. HDR targets (1000 or 4000 nit) shift the tonescale and the desaturation, and need a re-fit.
- **Approximate path to white.** Saturated highlights will not match a full ACES 2.0 render. Neutrals and midtones will be very close.
- **Tune the roll-off.** The `0.85` in the path-to-white `mix` controls how far highlights desaturate. Push it to `1.0` for full neutral highlights, drop it to keep more colour.

## Credits and licence

The tonescale is the ACES 2.0 Daniele Evo curve from the [ACES project](https://github.com/aces-aswf/aces-core); AceX only fits it and wraps it. The per-channel form follows the lineage of Krzysztof Narkowicz's ACES 1.0 fit and Stephen Hill's matrix variant. The AceX code above is free to use, modify, and ship in anything, commercial or not. No attribution required, though a link back is always appreciated.