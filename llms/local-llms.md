# Local LLMs On AMD GPU

For running local large language models on AMD GPUs:

- **Ollama** (on host) works pretty seamlessly in my experience
- The same thing can be said approximately for running Ollama in Docker

I use/have used Msty, Jan, and LM Studio.

More than the AMD support, the main annoyance has actually been the lack of intercompatibility between model formats which often means needing to duplicate weights.

## My Setup

My "go-to" for conversational use of local AI tools on AMD is **LM Studio**.

My more typical use case for local large language models, however, is in batch text edits done via script. For this use case I've used both LM Studio's local API function and Ollama.

## Vision Models

For vision models on my 12 GB VRAM system I've found that **Qwen 2.5 VL (8B)** works really well.
