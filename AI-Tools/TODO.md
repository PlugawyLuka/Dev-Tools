# LLM — TODO & step-by-step tutorials

Purpose
- Checklist and practical tutorial steps for evaluating and deploying local LLMs / runtimes. Organised per model/toolkit category found in PROSPECTS.md.

Quick checklist (global)
- [ ] Choose target use case (chat / Q&A / command assistant / structured tasks).
- [ ] Pick 1–2 model families to evaluate (recommend: LLaMA-7B via llama.cpp, and Flan-T5-small).
- [ ] Prepare development host (desktop) for conversion & testing.
- [ ] Prepare target Pi (Pi 4 with 8GB or Pi 5 recommended); ensure power & cooling.
- [ ] Acquire storage (fast SD or SSD) and a backup plan for model files.
- [ ] Test inference speed on desktop first; then move to Pi with quantized models.
- [ ] If Pi too slow, prepare hybrid plan (Pi as I/O, desktop as model server).

Necessary things (hardware & software)
- Hardware
  - Desktop with good RAM/CPU/GPU for model conversion and experimentation.
  - Raspberry Pi 4 (8GB) or Pi 5 for target deployment.
  - Fast storage (SSD recommended), reliable power supply.
- Software / tooling
  - Python 3.10+/pip, Hugging Face transformers (for small models), model conversion scripts, llama.cpp/ggml binaries, quantization scripts.
  - Tools: ffmpeg (if handling audio), ssh, systemd (for service), monitoring (htop).
- Files & accounts
  - Hugging Face account (some models require accepting license).
  - Local model storage area (do not commit model weights to git).

Step-by-step tutorials (high-level, no runnable commands)

A) LLaMA-family (7B) via llama.cpp — recommended Pi path
1. Decide exact model (LLaMA2-7B or community Alpaca/Vicuna 7B).
2. On desktop: download official weights from the model provider (follow license & model card).
3. Convert weights into GGML/llama.cpp format using the official conversion tools (available in the llama.cpp repo).
4. Quantize the converted model to 4-bit (or other supported quant) to reduce memory footprint.
5. Test inference on desktop with llama.cpp executable and measure latency / quality.
6. Transfer quantized GGML file to Pi storage (SSD if possible).
7. Install llama.cpp on Pi (ARM build or prebuilt binary); run inference and measure response times.
8. If performance acceptable, create a small wrapper service (REST or socket) and systemd unit for auto-start.
9. Log usage locally and limit accepted commands (safety whitelist).

Notes & tips
- Keep the unquantized model only on desktop; keep Pi copies quantized to save space.
- Use low batch sizes / small context windows to reduce RAM usage.
- Test privacy & access patterns: Pi should not auto-upload data.

B) Flan-T5 (small) — lightweight instruction-following path (easier)
1. Choose Flan-T5-small or other tiny Flan variant from Hugging Face.
2. On desktop and Pi: use standard transformer pipeline (small models load quickly).
3. Prototype the NLU workflow: input → model.generate → post-process.
4. Measure latency on Pi; small Flan models typically run adequately on Pi 4/5.
5. Wrap the model with a small service and systemd unit for deployment.

Notes & tips
- Flan-T5 is encoder‑decoder; fine for Q&A, summarization, and structured responses.
- It is easier to use the Hugging Face pipeline for small Flan models than to convert.

C) GPT-J-6B / GPT-Neo — evaluation path (desktop-first)
1. Download GPT-J-6B weights and test on desktop in half precision (fp16) if possible.
2. Try model offloading / quantization experiments; measure resource use.
3. If acceptable, experiment with smaller quantized formats, but expect heavy compute and memory.
4. Deploy only on desktop/server; Pi inference not recommended.

D) General deployment & ops (common steps)
1. Create a reproducible environment on desktop (requirements, virtualenv).
2. Document precise model versions and conversion commands (do not store weights).
3. Create an API wrapper for serving the model (simple REST or socket) and a minimal client on Pi if hybrid.
4. Create systemd service files and logging strategy (rotate logs).
5. Implement usage quotas / resource guards to avoid OOM.
6. Add monitoring (simple CPU/memory alerts) and a health-check endpoint.

Testing & evaluation checklist
- Functional: sample prompts with expected outputs, record pass/fail.
- Performance: latency (ms–s), CPU usage, memory usage.
- Robustness: test larger prompts and concurrent requests (if required).
- Privacy: ensure no unintentional network egress; test with firewall or isolated network.

Security & license notes
- Respect each model’s license (some require acceptance on Hugging Face).
- Do not commit model weights to the repo; store locally or on attached storage.
- If you ever expose a model via API, enforce authentication and local-only binding by default.

Next steps (recommendation)
- 1) Start with two experiments: (a) LLaMA-7B quantized via llama.cpp (desktop → Pi test), (b) Flan-T5-small native pipeline on Pi.
- 2) Record timings and memory usage for both on desktop and Pi.
- 3) Decide on hybrid vs fully local based on those metrics.

If you want, I can:
- Provide a concise test checklist for the Pi (models to try, sample prompts, metrics to capture), or
- Draft the exact file templates (service unit examples, API wrappers) as pseudo-code (no runnable scripts) for your repo.
Which do you want next?