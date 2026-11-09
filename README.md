# Cod3r

Cod3r is a local coding assistant for [Ollama](https://ollama.com). It is a custom persona on top of an existing code model: a tuned `Modelfile` with a system prompt that makes it a Python-first mentor for someone learning cybersecurity on both Windows and Linux. It runs fully offline on your own hardware.

There is no fine-tuning or new weights here. Cod3r is a base model plus sampling parameters plus a system prompt, so building it takes seconds once the base model is downloaded.

## What it does

- Defaults to Python 3.10+ and writes complete, runnable code with type hints, docstrings, and PEP 8 style.
- Prefers the standard library (`argparse`, `pathlib`, `logging`) and adds timeouts and error handling to network code.
- Explains the "why", and answers in a fixed shape: short summary, code, "How it works", "How to run it", then optional next steps.
- Supports defensive and educational security work (log analysis, hardening, CTFs, home labs, forensics). It reminds you to test only systems you own or are authorized to test, and declines to write malware or evasion tooling.

## Base model

| | |
|---|---|
| Base | [`qwen2.5-coder:14b`](https://ollama.com/library/qwen2.5-coder) (about 9 GB, 4-bit quantized) |
| Context window | `num_ctx 16384` |
| Sampling | `temperature 0.2`, `top_p 0.9`, `top_k 40`, `repeat_penalty 1.1` |

Low temperature keeps code output consistent. See [`Modelfile`](Modelfile) for the full system prompt.

## Build and run

Requires [Ollama](https://ollama.com/download).

```bash
# 1. Pull the base model
ollama pull qwen2.5-coder:14b

# 2. Build Cod3r from the Modelfile (run from the repo root)
ollama create cod3r -f Modelfile

# 3. Chat
ollama run cod3r
```

Other useful commands:

```bash
ollama list                 # confirm cod3r is installed
ollama show cod3r           # inspect parameters
ollama show cod3r --modelfile
ollama rm cod3r             # remove it
ollama run cod3r "Write a CLI that hashes a file with SHA-256"
```

After editing the `Modelfile`, re-run `ollama create cod3r -f Modelfile` to rebuild.

## Repository layout

```text
Modelfile        # model definition: base, parameters, system prompt
knowledge/
  python/        # cheat sheets: argparse, pathlib, logging, subprocess, socket, regex
  security/      # (planned)
  windows-linux/ # (planned)
  templates/     # (planned)
  notes/         # (planned)
```

The `knowledge/` folder holds reference material written for Cod3r and for you. The Python cheat sheets each have short examples that have been run on Python 3.10+, plus a "common mistakes" table. Ollama does not read this folder automatically. Paste a sheet into a chat, or pass it in the prompt:

```bash
ollama run cod3r "$(cat knowledge/python/subprocess.md) Using the notes above, write a safe ping wrapper."
```

## Tested on

| Component | Spec |
|---|---|
| CPU | AMD Ryzen 7 7800X3D (8 cores) |
| RAM | 64 GB (61 GiB usable) |
| GPU | NVIDIA GeForce RTX 4080 SUPER, 16 GB VRAM |
| Driver | NVIDIA 550.163.01 (CUDA 12.4) |
| OS | Kali GNU/Linux Rolling |
| Ollama | 0.40.15 |

The 14B 4-bit model is about 9 GB, so it fits in 16 GB of VRAM with room for the 16K context. With less VRAM, Ollama will offload layers to the CPU and run slower. For smaller GPUs, change the `FROM` line to `qwen2.5-coder:7b` (about 4.7 GB).

## License

See [LICENSE](LICENSE).
