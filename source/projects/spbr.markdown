---
layout: page
title: "Physically Based Path Tracer"
comments: true
sharing: true
footer: true
---

### Physically Based Path Tracer
{% gallery %}
/images/projects/spbr/sponza-day.png: Crytek Sponza Environment + Directional light
/images/projects/spbr/sponza-night.png: Crytek Sponza 18 point lights, 1 directional moon-light
/images/projects/spbr/spheres-close.png: Cornell Box 1 point light
/images/projects/spbr/spheres-front.png: Cornell Box 1 point light
{% endgallery %}
*Scene acknowledgements: Crytek Sponza Atrium scene is modeled by Frank Meinl. Scene data obtained from [Morgan Mcguire](http://graphics.cs.williams.edu/data/meshes.xml)*

A project I started with a friend to do some heavy graphics programming from scratch and later continued on my own. We started by reading and implenting core features of [Physically Based Rendering (PBRT)](http://www.pbrt.org) by Matt Pharr and Greg Humphreys.

#### Stats

* Time: 2.5 months (09/13-11/13)
* Technologies: C++11, CUDA, CMake
* Platform: Linux
* Team-Size: 2 => 1 

#### Features
**Acceleration Structure: SBVH Split Bounding Volume Hierarchy**

A problem we saw was that our BVH implementation was only accelerating very uniformly tessellated scenes. I.e. like the teapot or the Stanford dragon. However, Sponza atrium and other scenes are more realistic, and it would be good practice to be able to handle those scenes efficiently.

The previous object split BVH took way too long to even get one 1 sample per pixel for a 640x480 image. After SBVH was implemented this pass took on the order of 20s, I could finally see an interactive preview and generate some images!

References: [Stich '09](http://www.nvidia.com/docs/IO/77714/sbvh.pdf)

**CPU/GPU Wavefront Formulation**

As the first step to building a GPU path tracer, I decided to rework my renderer to process many paths in parallel. I was lucky enough to attend a talk by Tero Karras at SIGGRAPH'13 that describe the wavefront formulation for path tracing. This approach seemed to make a lot of sense: extend all paths by 1 ray/bounce at a time and group similar operations together in parallel, resulting in execution coherence. 

As a first step I kept most of my CPU code and only ray traversals GPU accelerated. This was a good bit of work already to transform BVH structures into a GPU-friendly format, and also to be able to unwrap the renderer loop of processing 1 path many bounces, to many paths 1 bounce.

In the end using the GPU even just for ray traversals brought down the render time for the 720p Sponza scene from 9hr single threaded CPU implementation to 2hr wavefront implementation. 

References: 

* [Laine Karras '13](https://research.nvidia.com/publication/megakernels-considered-harmful-wavefront-path-tracing-GPUs)
* [Aila Laine '09](http://www.nvidia.com/object/nvidia_research_pub_011.html)

**Efficiency features**

The following were more for own sanity while developing and debugging. Part of the problem in building an offline renderer is that you would waste all this time waiting for your image to come out, when really your camera was positioned incorrectly or your lights weren't bright enough. 

**Progressive preview:** loop over all pixels with 1 sample instead of each pixel with all samples. Measure time between processing batches of samples, if 100ms has elapsed display the current raster to the screen.

**Blender scene format:** I integrated, then contributed to the [Open Asset Importer](http://github.com/assimp). It served my purposes well for allowing me to place camera, lights and repositioning them. However, it's quickly reached it's limits for describing the materials and more complicated lights, that I will likely need to move to using my own exporter or custom importer. Either way, scene set-up should never happen via text file

**Partial scene reloading:** building the SBVH for large scenes can be costly, and isn't necessary if you just move lights around, change brightness, or adjust the camera. I added a button to reload those parts and restart rendering without rebuilding the BVH. This allowed for more minor adjustments.

**Pixel debugging:** If there was a weird pixel in the output, it was easy enough to report the position of the pixel, which could be used to limit the render window from command line while running in a debugger. This allowed me to pinpoint problems without running across all of the other computation.

