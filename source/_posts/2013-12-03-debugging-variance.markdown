---
layout: post
title: "Debugging Variance: Mistakes and Lessons"
date: 2013-12-03 17:40
comments: true
categories: 
 - Rendering
 - Debugging
 - Development
---

My first entry will be dedicated to humbly pointing out some of the fun bugs I've run into while working on my own path tracer. I didn't find many entries on steps used to debug variance, so I thought I'd share my experiences here.

### The problem

Consider the below output of the Cornell Box Scene:
{% img images/posts/2013-12-03/noisy.png Noise and Artifacts %}

Some questions I kept asking myself:

1. This is rendered with 1024 samples per pixel. Why is it so noisy?
1. This scene is simple and has dirac delta BRDFs (mirror), and plain old lambert. Why is it so noisy?
1. While interesting, where did that "flower pedal" light pattern on the ground come from?

### Hypotheses and suspicions

Initially I thought that the variance was caused by a bug in the integrator,
that I wasn't computing the pdf value correctly and properly weighting the
sample. Or it was a problem with the material evaluation and I wasn't computing
the bounce direction correctly. 

Then I began to suspect the concentric uniform disk sampling code. At the start
of the project my partner and I thought it best to implement some of the
low-level routines ourselves to foster our understanding. We thought we could
write a simpler routine that would be clearer to us instead of borrowing the
code that was readily available. However, we hadn't tested it's correctness...
*it must be right* was our flaw.

We ignored it, and proceeded to generate many images. They all appeared too
noisy for the amount of samples we were taking. Being inexperienced, I thought,
maybe this was some efficiency issue, if we take more samples the noise will go
away.  However, published results kept showing cleaner images for the same
scene in <= the # of samples per pixel. 

### Debugging with pictures

The best way to debug random sampling is with a picture, not with some routine that
measures the distribution. The human visual system is *superior at recognizing patterns*... given that you
feed a picture without too much information, after all the final image wasn't
helpful in pinpointing the problem.

Using alpha blending helped me to see if certain points overlapped more than
others. If the distribution was uniform, the color should be even.

``` c++ Plot the points with 10% alpha blending
void PlotPoint(int raster_width, int raster_height, const Point2 &input, vector<Spectrum> &pixels)
{
    int plot_width = raster_width / 2;
    int plot_height = raster_height / 2;

    Point2 p = input;
    p.u = 0.5f * (p.u + 1.f);
    p.v = 0.5f * (p.v + 1.f);

    int x = roundf(plot_width * p.u + 0.5f * (raster_width - plot_width));
    int y = roundf(plot_height * p.v + 0.5f * (raster_height - plot_height));

    if (x < 0 || y < 0 || x >= raster_width || y >= raster_height)
    {
        cout << format("%f, %f") % p.u % p.v << endl;
        return;
    }

    Spectrum &target = pixels[x + raster_height * y];

    Spectrum color(1.f, 0.f, 0.f);
    float alpha = 0.1f;

    target = alpha * color + (1.f - alpha) * target;
}
```

Taking a lot of samples is necessary to see a pattern (esp at 512x512 pixels), but too many and the image with coagulate into a solid blob

``` c++ Take 1M samples
    const int kNumSamples = 1 << 20;
    for (int i = 0; i < kNumSamples; ++i)
    {
        PlotPoint(kWidth, kHeight, ConcentricSampleDisk(rng.RandomFloat(), rng.RandomFloat()), pixels);
    }
```

### Realization

{% img /images/posts/2013-12-03/sampling_broken.png Broken %}

Wow, so happy that I discovered this. All the samples are clumping around this flower petal pattern, and it's the same as our light pattern in the picture.

`ConcentricSampleDisk` should take 2 floats ranging from [0, 1] and return them uniformly distributed in a 2D disk. This is based off of [Peter Shirley's original unit disk mapping](http://psgraphics.blogspot.com/2011/01/improved-code-for-concentric-map.html), which takes 4 triangle quadrants of a unit square to map to 45 degree slices of the unit disk. Our mapping was motivated by ideas similar to the simpler code posted by Shirley (divide into 2 quadrants), except that we decided to use tau as % half-lengths instead.

``` c++ Incorrect sampling
inline Point2 ConcentricSampleDisk(const float u1, const float u2)
{
    const float x = 2 * u1 - 1;
    const float y = 2 * u2 - 1;

    if (x == 0.f && y == 0.f)
        return Point2(0.f, 0.f);

    float tau;
    const float r_x = fabs(x);
    const float r_y = fabs(y);
    float radius;
    if (r_x > r_y)
    {
        const float tau_start = x >= 0.f ? -0.25f : 0.75f;
        tau = u2 * 0.5f + tau_start;
        radius = r_x;
    }
    else
    {
        const float tau_start = y >= 0.f ? 0.25f : 1.25f;
        tau = u1 * 0.5f + tau_start;
        radius = r_y;
    }
    const float theta = tau * M_PI;
    return Point2(radius * cos(theta), radius * sin(theta);
}
```

In lines 13, 19 we incorrectly redid this mapping as pick which sample is the
radius, which determines the tau_start, then use the other sample as the basis for the angle, theta.
However you'll notice there's an unforeseen dependency here, take the first
case for instance. If r_x is greater than r_y, then abs(u1) > abs(u2) and then
the range of values for u2 are strictly less than the abs(length). For this to work
u2 needs to be able to range from [0, 1], which is impossible given the if
condition.

### Solution

The solution is to divide the two numbers. I hadn't realized it until this point, but this is more intuitive and probably is the motivation of the original code: the slope of the triangle maps directly to the angle. So obvious I feel dumb.

``` c++ Fixed code 
        tau = y/x * 0.5f + tau_start;
        tau = x/y * 0.5f + tau_start;
```

The new sampling code produces a uniformly distributed result.

{%img /images/posts/2013-12-03/sampling_correct.png Uniform! %}

Which subsequently removed the noise and light artifacts

{%img /images/posts/2013-12-03/correct.png Smooth! %}

### Takeaways

1. Take the time to write the visualization code for sampling routines. It took so little time and I wish I had done it sooner.
1. Reference images/scenes are a must for determining correctness. If I didn't have an idea of what an image with 1024 samples per pixel should look like, I would have a lot harder time convincing myself something was wrong.
1. Caution required before I implement someone else's technique, doing it better or making it clearer is not trivial. The context and motivation for each operation is often missing.
