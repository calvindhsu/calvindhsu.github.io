---
layout: page
title: "projects"
comments: false
sharing: false
footer: true
---

# Personal

### Physically Based Path Tracer (Fall 2013)
{% gallery %}
/images/projects/spbr/sponza-day.png
/images/projects/spbr/sponza-night.png
/images/projects/spbr/spheres-close.png
/images/projects/spbr/spheres-front.png
{% endgallery %}
*Scene acknowledgements: Crytek Sponza Atrium scene is modeled by Frank Meinl. Scene data obtained from [Morgan Mcguire](http://graphics.cs.williams.edu/data/meshes.xml)*

A project I started with a friend to do some heavy graphics programming from scratch and later continued on my own. We started by reading [Physically Based Rendering (PBRT)](http://www.pbrt.org) by Matt Pharr and Greg Humphreys, the standard reference for realistic imaging. 

Technologies: C++11, CMake, CUDA.
Platform: Linux

#### Features
**Acceleration Structure: SBVH Split Bounding Volume Hierarchy**

A problem we saw was that our BVH implementation was only accelerating very uniformly tessellated scenes. I.e. like the teapot or the Stanford dragon. However, Sponza atrium and other scenes are more realistic, and it would be good practice to be able to handle those scenes efficiently.

The previous object split BVH took way too long to even get one 1 sample per pixel for a 640x480 image. After SBVH was implemented this pass took on the order of 20s, I could finally see an interactive preview and generate some images!

References: [Stich '09](http://www.nvidia.com/docs/IO/77714/sbvh.pdf)

**CPU/GPU Wavefront Formulation**

As the first step to building a GPU path tracer, I decided to rework my renderer to process many paths in parallel. I was lucky enough to attend a talk by Tero Karras at SIGGRAPH'13 that describe the wavefront formulation for path tracing. This approach seemed to make a lot of sense: extend all paths by 1 ray/bounce at a time and group similar operations together in parallel, resulting in execution coherence. 

As a first step I kept most of my CPU code and only ray traversals GPU accelerated. This was a good bit of work already to transform BVH structures into a gpu-friendly format, and also to be able to unwrap the renderer loop of processing 1 path many bounces, to many paths 1 bounce.

In the end using the gpu even just for ray traversals brought down the render time for the 720p Sponza scene from 9hr single threaded CPU implementation to 2hr wavefront implementation. 

References: 

* [Laine Karras '13](https://research.nvidia.com/publication/megakernels-considered-harmful-wavefront-path-tracing-gpus)
* [Aila Laine '09](http://www.nvidia.com/object/nvidia_research_pub_011.html)

**Efficiency features**

The following were more for own sanity while developing and debugging. Part of the problem in building an offline renderer is that you would waste all this time waiting for your image to come out, when really your camera was positioned incorrectly or your lights weren't bright enough. 

**Progressive preview:** loop over all pixels with 1 sample instead of each pixel with all samples. Measure time between processing batches of samples, if 100ms has elapsed display the current raster to the screen.

**Blender scene format:** I integrated, then contributed to the [Open Asset Importer](http://github.com/assimp). It served my purposes well for allowing me to place camera, lights and repositioning them. However, it's quickly reached it's limits for describing the materials and more complicated lights, that I will likely need to move to using my own exporter or custom importer. Either way, scene set-up should never happen via text file

**Partial scene reloading:** building the SBVH for large scenes can be costly, and isn't necessary if you just move lights around, change brightness, or adjust the camera. I added a button to reload those parts and restart rendering without rebuilding the bvh. This allowed for more minor adjustments.

**Pixel debugging:** If there was a weird pixel in the output, it was easy enough to report the position of the pixel, which could be used to limit the render window from command line while running in a debugger. This allowed me to pinpoint problems without running across all of the other computation.


