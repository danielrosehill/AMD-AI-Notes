# Conda Or Docker (Or UV Or Venv!)

This is a question that comes up frequently for those setting up AI/ML environments on AMD hardware. Here's my take after spending way too much time with both.

## TL;DR

For most AMD/ROCm users doing local AI work: **use Docker**. It's AMD's recommended approach, and once you understand Docker layering, the storage concerns that drove many of us to Conda largely disappear.

That said, Conda isn't bad—it's genuinely excellent for scientific computing and research environments. But for the "I just want to run Stable Diffusion locally" crowd, Docker is simpler.

## When to Use Docker

Docker is the better choice when:

- **You want AMD's officially supported path.** AMD publishes and maintains ROCm Docker images. This means less troubleshooting.
- **You want isolation without complexity.** Containers are self-contained. If something breaks, nuke the container and start fresh.
- **You're running multiple AI tools.** ComfyUI, InvokeAI, text-generation-webui—each can live in its own container without conflicts.
- **You want reproducibility.** Share a Dockerfile or compose file, and someone else can replicate your exact environment.

## When to Use Conda

Conda still makes sense when:

- **You're doing scientific/research computing** where precise package versions and academic reproducibility matter.
- **You need to mix packages from conda-forge with pip packages** in complex ways.
- **You're developing Python libraries** and need to test across multiple Python versions easily.
- **You're on a system where Docker isn't available** (some HPC clusters, restricted environments).

## Why Not Both?

Here's my possibly controversial take: **for most people, there's no good reason to use both Conda and Docker together**.

The typical argument for combining them is "Conda inside Docker for environment management." But this adds complexity without much benefit. Docker already provides isolation. Adding Conda inside Docker means:

- Larger images
- More moving parts to debug
- Conda's solver running inside a container (slow)
- Two layers of environment management to understand

If you're using Docker, just use pip/venv inside the container. If you're using Conda, you probably don't need Docker's isolation.

The exception might be if you're building images for a team that's standardized on Conda, but even then, I'd question whether that complexity is necessary.

## The Docker Layering Revelation

This is the thing that took me **far too long** to understand, and it completely changed how I think about Docker for AI workloads.

### The Problem I Thought I Had

When I first looked at Docker for AI, my reaction was: "Great, so I need to download a 15GB PyTorch+ROCm image for every single project? My disk is going to explode."

This concern drove me to Conda initially—the idea of having one PyTorch installation shared across environments seemed much more sensible.

### What I Didn't Understand

Docker images are built in **layers**, and layers are **shared** across images.

Here's what that means practically:

```
rocm/pytorch:latest (base image)
├── Ubuntu base layer (~100MB)
├── ROCm libraries layer (~2GB)
├── PyTorch layer (~8GB)
└── Your customizations (~few MB)
```

When you build a new image **based on** `rocm/pytorch:latest`, Docker doesn't copy those base layers. It reuses them. So if you have:

- Image A: rocm/pytorch + ComfyUI dependencies
- Image B: rocm/pytorch + InvokeAI dependencies
- Image C: rocm/pytorch + text-generation-webui dependencies

You're not storing 3x the PyTorch+ROCm stack. You're storing it **once**, plus the small delta for each project.

### How to Take Advantage of This

1. **Pick a base image and stick with it.** For AMD/ROCm, `rocm/pytorch` is the obvious choice.

2. **Build your project images FROM that base:**
   ```dockerfile
   FROM rocm/pytorch:latest

   # Your project-specific stuff
   RUN pip install comfyui-dependencies
   ```

3. **Don't use different base images for different projects** unless you have a good reason. Every different base image is a new set of layers to store.

4. **Use `docker system df` to see actual disk usage.** You'll see that shared layers aren't counted multiple times.

### Practical Example

```bash
# Pull the base image once
docker pull rocm/pytorch:rocm6.3_ubuntu22.04_py3.10_pytorch_release_2.4.0

# Check size
docker images
# Shows ~15GB for the base image

# Build a project image
docker build -t my-comfyui -f Dockerfile.comfyui .
# This adds maybe 500MB of project-specific packages

# Build another project image
docker build -t my-invokeai -f Dockerfile.invokeai .
# This adds maybe 800MB of different packages

# Total disk usage is NOT 15GB + 15GB + 15GB
# It's closer to 15GB + 500MB + 800MB
```

### Why This Matters for AMD Users

The PyTorch+ROCm stack is **large**. The ROCm libraries alone are several gigabytes. If you thought (like I did) that Docker meant duplicating this for every project, the storage requirements seemed insane.

Understanding layering means you can have dozens of AI tool containers while only storing the base stack once. This is actually *more* storage-efficient than Conda in many cases, because Conda environments don't share packages between environments by default.

## What About UV/Venv?

For lightweight projects that don't need the full PyTorch+ROCm stack, `uv` with standard venvs is fast and simple. Great for:

- API wrapper projects (OpenAI, Anthropic clients)
- Data processing scripts
- Anything that doesn't need GPU acceleration

But for local AI inference on AMD, you're going to want either Docker or Conda for managing the ROCm+PyTorch stack.

## My Recommended Setup

1. **Install ROCm on host** (see [host-rocm.md](host-rocm.md))
2. **Pull the official rocm/pytorch Docker image** as your base
3. **Build project-specific images FROM that base** for each AI tool you want to run
4. **Use uv/venv** for lightweight Python projects that don't need GPU

This gives you the isolation benefits of containers, AMD's officially supported PyTorch path, and efficient storage through layer sharing.

## A Note on ROCm: Host AND Docker (Not Either/Or)

This seems to confuse people, so let me address it directly: **having ROCm installed on both your host system AND inside Docker containers is completely normal and expected**. It's not redundant or wasteful.

Here's why this makes sense:

**ROCm on host** is necessary for:
- The kernel drivers that let your GPU communicate with the system
- Direct GPU access for non-containerized applications
- Tools like `rocm-smi` for monitoring your GPU
- Running InvokeAI, ComfyUI, or other tools directly on your system without containers

**ROCm inside Docker** is necessary for:
- The userspace libraries that applications link against
- Container isolation (the container can't just reach out and use host libraries)
- Reproducibility (the container carries its own known-good ROCm version)

The host ROCm provides the **kernel-level drivers** that Docker containers access via `--device=/dev/kfd --device=/dev/dri`. The in-container ROCm provides the **userspace libraries** that your AI applications actually link against.

Think of it like this: the host ROCm makes the GPU *visible* to containers, while the container's ROCm makes the GPU *usable* by the application running inside.

This is also why you'll see recommendations to ensure your host ROCm version and container ROCm version are compatible—they're working together, not duplicating each other.
