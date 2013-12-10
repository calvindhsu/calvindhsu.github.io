---
layout: post
title: "CUDA vs OpenCL Development Experience on an NVIDIA GPU"
date: 2013-12-06 14:01:48 -0800
comments: true
categories: 
- GPGPU
- Debugging
- OpenCL
- CUDA
- Tools
---

I had to make a choice as to what GPGPU API to use for my GPU path tracer. I read sources like [CUDA Vs OpenCL: Which should I use](http://wiki.tiker.net/CudaVsOpenCL) and decided initially that OpenCL would be the right choice: support an open standard which would see wider and wider adoption and gain experience with the API. However, I found along the way, the path of least resistance brought me to CUDA.

Big caveat here: I approached this choice as something to use for my personal, learning project with no need to ship a commercial product.

### What's inside my machine?

I started out basing my choice off of the GPU I had already, since I wasn't ready to shell out money for a new card. Obviously if you have an AMD card there's no choice (also good luck if you want to develop on Linux, the drivers [are not well regarded by the community](https://wiki.archlinux.org/index.php/AMD_Catalyst)).

So in my case I did have a choice with my 2 year old NVIDIA Geforce 560Ti (Fermi). This post documents experience with the tool support, build system, and language features.

<!-- more -->

## Tools

GPU tool support has always been vendor specific i.e. Nsight vs CodeXL (previously gDebugger, AMD purchased Graphic Remedy), and FX Composer vs RenderMonkey. For GPGPU computing there's the additional factor that CUDA is NVIDIA's proprietary technology, and this appears to motivate some uneven support.

### NVIDIA Nsight (Eclipse Edition v5.5.0): CUDA only [Official Site](http://www.nvidia.com/object/nsight.html)

NVIDIA Nsight doesn't have any support for OpenCL development. I've only tested out the Eclipse edition at the time of this writing, and it lacks, well, everything:

* No OpenCL syntax highlighting
* Inability to profile OpenCL calls / kernel performance
* Inability to debug OpenCL kernels

This was a pretty big disappointment, I was really looking forward to at least seeing some performance traces for my program. At least that would help reveal at a glance where the time is being spent (i.e. memory transfers, compute, CPU).

**Workaround: Lack of Syntax Highlighting**

Because OpenCL's C is similar, Google reveals that a lot of people end up designating cl files as C files, then creating a dummy header file that is included in .cl code, something like:

```c cl_ignores.h
#define __kernel
#define __global
#define __local
#define __constant
```

**Warning: CUDA kernel debugging requires 2 GPUs or SM3.5**

One unfortunate discovery was finding out that kernel debugging requires a 2nd GPU or a SM3.5+ GPU of which there are currently a handful of models. See [this demonstration](http://www.youtube.com/watch?v=nKKLqc2TgsI).

### AMD CodeXL [Official Site](http://developer.amd.com/tools-and-sdks/heterogeneous-computing/codexl/)

I evaluated CodeXL for use with NVIDIA hardware on a Linux machine and it *does* log and report OpenCL calls and errors, but it is not able to debug or profile.

### CUDA only: printf support

With an inability to use a debugger, one would naturally rely on print to log debug information. However, NVIDIA drivers only support OpenCL 1.1, so printf isn't supported. This support is enabled via vendor extensions, but I cannot find such an OpenCL extension for NVIDIA GPUs.

Funny enough (NVIDIA's priorities are not hidden), CUDA has this function built-in, and it works on the same hardware.

**Aside: Why is it important to have printf for GPU development?**

Some might argue: don't waste your time, printf isn't useful for debugging, you should write tests or examine that value of output when read back on the CPU. 

I believe some of these opinions have to do with the noise of too many inputs. After all if you put in 1 printf into your kernel but your kernels runs 1k threads in parallel you will get so much extra data (Warning: don't do this with 1 GPU since you'll freeze up the driver!).

I argue that it's still useful provided you are able to limit your input sizes. This was a process that I used in my own development process: see a bad pixel, limit rendering to that 1 pixel, use printfs to output debug information to determine where GPU code was failing.

Combine that with the fact that GPU programs are becoming increasingly complicated. Aila & Laine's purportedly state of the art [GPU ray traversal kernels](https://code.google.com/p/understanding-the-efficiency-of-ray-traversal-on-gpus/) involves several nested while loops, a stack 32 frames deep, compact data structures with precomputed values. It would be extremely tedious to debug without having some sort of intermediate output besides ray hit / not-hit.

### Build support: CUDA compiles kernel code

This not only makes you avoid the fuss of dealing with loading text from file and giving your errors at build time, but it makes it easier to share header files between host and device code. This is useful for declaring types that need to be passed between host and device in a shared header file.

**CUDA and C++11 with CMake** 

In my project I encountered some complications with incorporating CUDA and C++11 code. The problem is that CUDA's host compiler is older, and won't compile the latest GCC 4.8 headers that are on my system. The solution is to separate CUDA code into a self contained library, and then link it with the main binary.

Recent versions of CMake already includes a FindCUDA.cmake library written by James Bigler. By default this will propagate your host compiler flags to the NVIDIA C++ compiler. If you're using `-std=c++11` then you might get an error like this:

    /opt/cuda/bin/nvcc /home/calvin/Projects/spbr/spbr/cudakernels/src/bvh_intersect.cu -c -o /home/calvin/Projects/spbr-build2/spbr/cudakernels/CMakeFiles/cudakernels.dir/src/./cudakernels_generated_bvh_intersect.cu.o -ccbin /usr/bin/cc -m64 -D__CL_ENABLE_EXCEPTIONS -Xcompiler ,\"-std=c++11\",\"-g\",\"-Wall\",\"-g\" --gpu-architecture sm_21 -DNVCC -I/opt/cuda/include -I/opt/cuda/include
    /usr/lib/gcc/x86_64-unknown-linux-gnu/4.8.2/include/stddef.h(432): error: identifier "nullptr" is undefined

    /usr/lib/gcc/x86_64-unknown-linux-gnu/4.8.2/include/stddef.h(432): error: expected a ";"

    /usr/include/c++/4.8.2/x86_64-unknown-linux-gnu/bits/c++config.h(190): error: expected a ";"

    /usr/include/c++/4.8.2/exception(63): error: expected a ";"

    /usr/include/c++/4.8.2/exception(68): error: expected a ";"

    /usr/include/c++/4.8.2/exception(76): error: expected a ";"

    /usr/include/c++/4.8.2/exception(83): error: expected a ";"

    /usr/include/c++/4.8.2/exception(93): error: expected a "{"

This is caused by the NVIDIA compiler (nvcc) getting c++11 flags and being able to compile the standard headers. To fix, add the following line to your `CMakeLists.txt`

```cmake Getting CUDA to compile with C++11 in CMake```
set(CUDA_PROPAGATE_HOST_FLAGS OFF)
```

**OpenCL Semi-workaround 1: Includes by prepending source strings** 

You can get around lack of include support by prepending header files to your OpenCL source in your host code. But this also messy since then you need a build mechanism to include shared header files as text.

**OpenCL Semi-Workaround 2: Auto-convert cl source files to cpp string**

The guys who work on [Luxrender GPU](http://www.luxrender.net) cleverly made their build system take \*.cl files and insert them as strings in .cpp files. This nicety removes the pain of resource managing and loading files from source. Here's a CMake function from their build set-up:

```cmake KernelPreprocess.cmake http://www.luxrender.net
FUNCTION(PreprocessOCLKernel NAMESPACE KERNEL SRC DST)
	MESSAGE(STATUS "Preprocessing OpenCL kernel: " ${SRC} " => " ${DST} )

	IF(WIN32)
		add_custom_command(
			OUTPUT ${DST}
			COMMAND echo "#include <string>" > ${DST}
			COMMAND echo namespace ${NAMESPACE} "{ namespace ocl {" >> ${DST}
			COMMAND echo std::string KernelSource_PathOCL_${KERNEL} = >> ${DST}
# TODO: this code need to be update in order to replace " char with \" (i.e. sed 's/"/\\"/g')
			COMMAND for /F \"usebackq tokens=*\" %%a in (${SRC}) do echo \"%%a\\n\" >> ${DST}
			COMMAND echo "; } }" >> ${DST}
			MAIN_DEPENDENCY ${SRC}
		)
	ELSE(WIN32)
		add_custom_command(
			OUTPUT ${DST}
			COMMAND echo \"\#include <string>\" > ${DST}
			COMMAND echo namespace ${NAMESPACE} \"{ namespace ocl {\" >> ${DST}
			COMMAND echo "std::string KernelSource_${KERNEL} = " >> ${DST}
			COMMAND cat ${SRC} | sed 's/\\\\/\\\\\\\\/g' | sed 's/\"/\\\\\"/g' | awk '{ printf \(\"\\"%s\\\\n\\"\\n\", $$0\) }' >> ${DST}
			COMMAND echo "\; } }" >> ${DST}
			MAIN_DEPENDENCY ${SRC}
		)
	ENDIF(WIN32)
ENDFUNCTION(PreprocessOCLKernel)
```

## CUDA-only Language Features

Perhaps more important than tools are the capabilities of the language. Here are some notable ones that I encountered.

### Warp vote functions

I found certain functions that were relied on in research (well, NVIDIA's graphics research group no less) did not have OpenCL analogues. 

* \_\_ballot(): Used in wavefront rendering to coalesce atomic increments to a queue count
* \_\_any(): Used by speculative ray traversal to help with execution coherence
* \_\_all()

[See documentation here](http://docs.nvidia.com/cuda/cuda-c-programming-guide/index.html#warp-vote-functions)

It's not to say that these instructions cannot be implemented in OpenCL (they can with comparison to an atomic counter), and I'm also not clear on how much performance benefit these functions bring, but it was something I encountered when trying to implement recent papers.

### Kernel C++ support

You can write kernel traversal code in structs and classes. You can use some simple function templating, compare:

```c++ CUDA Swap
template <typename T>
__host__ __device__ void swap(T &x, T &y)
{
    T temp = x;
    x = y;
    y = temp;
}
```

For OpenCL you'd have to write a swap function per type...tedious:

```c OpenCL Swap
void swap(int *x, int *y)
{
    int temp = *x;
    *x = *y;
    *y = temp;
}
```

### Clean code with Thrust

A benefit of the C++ support in CUDA is the thrust library. The idea and usage is pretty neat: use STL types to encapsulate information on where the data is stored, and provide functions as a way of operating on that data in a device/host agnostic way.

Here's an example of doing GPU programming in a STL, template friendly like manner. 

```c++ Thrust Kernel Functor
struct BVHTraverseFunctor
{
    const CudaBVHNode *nodes;
    const CudaTriangle *leaves;

    BVHTraverseFunctor(const CudaBVHNode *nodes_,
                       const CudaTriangle *leaves_) :
        nodes(nodes_),
        leaves(leaves_)
    {}

    /**
     * @return (t, u, v, index) intersection
     */
    __host__ __device__ CudaIntersection operator()(CudaRay ray)  /* ((origin, tmin), (dest, tmax))  */
    {
        // Perform ray traversal
        ...
        return intersection;
    }
}
```

Using `thrust::device_vector` and `thrust::host_vector` allows for simpler and more maintainable code than managing raw buffer objects using `clCreateBuffer` or `cudaMalloc`. Assigning to a device vector from a host vector triggers a copy from host to device.

```c++ Host Code
class CudaTracer
{
private:
    thrust::device_vector<CudaBVHNode> _gpu_bvh_nodes;
    thrust::device_vector<CudaTriangle> _gpu_leaves;

    thrust::device_vector<CudaRay> _gpu_rays;
    thrust::device_vector<CudaIntersection> _gpu_hits;

    ...
};

void CudaTracer::EnqueueRays(const thrust::host_vector<CudaRay> &ray_buffer,
                             thrust::host_vector<CudaIntersection> &hit_buffer)
{
    // Upload input to device
    _gpu_rays = ray_buffer;

    // Resize output array
    _gpu_hits.resize(_gpu_rays.size());

    // Initialize functor with device pointers
    BVHTraverseFunctor traverse_functor(raw_pointer_cast(&_gpu_bvh_nodes[0]),
                                        raw_pointer_cast(&_gpu_leaves[0]));

    // Execute kernel via thrust
    thrust::transform(_gpu_rays.begin(),
                      _gpu_rays.end(),
                      _gpu_hits.begin(),
                      traverse_functor);

    // Copy results back to host
    hit_buffer = _gpu_hits;
}
```

## Conclusion

If you have NVIDIA hardware, and you're looking to have a smooth, more well supported development experience, and you're not concerned about deploying/distributing to users who don't have NVIDIA hardware, use CUDA! In the meantime, I'm hoping to see that OpenCL receive more support from NVIDIA.
