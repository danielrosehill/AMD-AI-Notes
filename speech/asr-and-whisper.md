# ASR And Whisper

GPU acceleration is very useful for running local speech-to-text ASR models.

## The ROCm/Whisper Compatibility Challenge

OpenAI's Whisper and most Whisper forks are built on PyTorch with CUDA dependencies. While PyTorch does support ROCm, the Whisper ecosystem has several pain points for AMD users:

### Why Native ROCm is Problematic

1. **CTranslate2 ROCm Support**: The popular `faster-whisper` library uses CTranslate2, which has limited/experimental ROCm support. Building from source with ROCm can be painful.

2. **PyTorch ROCm Wheels**: While official PyTorch ROCm wheels exist, many Whisper frontends and wrappers don't account for AMD GPU detection properly.

3. **GFX Version Mismatches**: Even when ROCm works, you often need `HSA_OVERRIDE_GFX_VERSION` environment variable tricks (e.g., `HSA_OVERRIDE_GFX_VERSION=11.0.0` for gfx1101 GPUs).

4. **Docker Complexity**: ROCm Docker images exist but require specific device mappings (`--device=/dev/kfd --device=/dev/dri`) and group permissions.

## The Vulkan Solution

I've had the best success—or rather suffered the least complication—using **Vulkan** for GPU acceleration with CTranslate2 model weights.

### Why Vulkan Works Better

- **Hardware Agnostic**: Vulkan is a cross-platform graphics API that works on AMD, NVIDIA, and Intel GPUs
- **No ROCm Stack Required**: Avoids the entire ROCm installation and compatibility dance
- **CTranslate2 Native Support**: CTranslate2 has solid Vulkan backend support
- **Simpler Setup**: Just need Vulkan drivers (usually pre-installed on Linux with Mesa)

### Model Format Requirements

When using Vulkan acceleration, ensure your model weights are in **CTranslate2 format**. Many Hugging Face models offer CTranslate2 conversions, or you can convert them yourself.

## Validated Software

I have used this approach to run both stock Whisper as well as my own work-in-progress fine-tunes.

For those on Linux, I can validate:

- **[DSNote](https://github.com/AnotherStranger/dsnote)** - Highly recommended! Works great with Vulkan backend
- **[Speech Note](https://github.com/AnotherStranger/dsnote)** - KDE-native option

## Alternative Approaches

### Whisper.cpp

[Whisper.cpp](https://github.com/ggerganov/whisper.cpp) offers another path:
- Pure C/C++ implementation
- Has experimental ROCm/HIP support via `hipBLAS`
- Can also use Vulkan via kompute
- Lighter weight than Python-based solutions

### Docker with ROCm

If you must use ROCm-based Whisper:

```bash
# Example Docker run with ROCm
docker run -it --rm \
  --device=/dev/kfd \
  --device=/dev/dri \
  --group-add video \
  --group-add render \
  -e HSA_OVERRIDE_GFX_VERSION=11.0.0 \
  rocm/pytorch:latest
```

But honestly, Vulkan is less hassle for ASR workloads.
