# Model Comparison Playground

A small benchmarking script that sends the same prompt set to multiple model types — a frontier API model, a reasoning model, a local open-weight model, and a multimodal (vision) call — and validates every response through Pydantic before comparing quality, latency, and cost.

## What this covers

| Requirement | Where it's implemented |
|---|---|
| Frontier API model | Leg 1 — `gemini-3.8-flash` via Gemini's OpenAI-compatible endpoint |
| Reasoning model | Leg 2 — `gemini-3.6-flash` |
| Local model via Ollama | Leg 3 — calls Ollama's REST API directly at `localhost:11434` |
| Multimodal (image + PDF field extraction) | Leg 4 — describes an image and extracts fields from a PDF page rendered to an image via PyMuPDF |
| Pydantic-validated structured output | Every leg validates its response against a `pydantic.BaseModel` schema (`ExplanatoryOutput`, `DocumentExtractionOutput`, `ToolFinalOutput`) |
| Native tool + manual tool-call loop | Leg 5 — a `get_weather` function tool, with the tool-call loop (model requests tool → we run it → we send the result back → model gives final answer) handled by hand |
| Prompt caching / cost & latency logging | Leg 6 — repeated calls against a long system prompt, with cost and latency logged per call (see caveat below) |
| Comparison table | Printed at the end of the script from data collected in `benchmark_results` |

## Setup

**1. Install dependencies**
```bash
pip install openai pydantic python-dotenv httpx pymupdf
```

**2. Get a Gemini API key**
Create one at [Google AI Studio](https://aistudio.google.com/apikey).

**3. Create a `.env` file** in the project root:
```dotenv
Gemini_API_Key=your_actual_key_here
GEMINI_MODEL_FRONTIER=gemini-3.8-flash
GEMINI_MODEL_REASONING=gemini-3.6-flash
MODEL_OLLAMA=my-local-model:latest
OLLAMA_URL=http://localhost:11434
```
Never commit this file — add `.env` to `.gitignore`. If a key is ever accidentally shared or committed, rotate it immediately from AI Studio.

**4. Set up Ollama**
```bash
ollama pull llama3.2:1b        # or whichever small model you prefer
ollama serve                   # make sure the daemon is running
```
If you're using a custom tag (like `my-local-model:latest`), confirm it's actually present with:
```bash
ollama list
```
`MODEL_OLLAMA` in `.env` must match one of these names **exactly** — including the tag after the colon.

**5. (Optional) Add test files**
Drop a `Sample.jpg` in the project root if you want Leg 4 to describe a real photo. A synthetic invoice PDF page is always generated automatically via PyMuPDF as a second test case, so this leg works even without a real image.

## Running it

```bash
python model_comparison.py
```

The script pauses 15 seconds between legs to stay under Gemini's free-tier per-minute rate limit. A full run takes roughly 1.5–2 minutes.

## Notes and known limitations

- **Free-tier daily quota:** Gemini's free tier caps each model at 20 requests/day. If you rerun the script many times in one day against the same model, later legs will fail with a `429 RESOURCE_EXHAUSTED` error. This is a hard daily cap, not something retries can fix — either wait for the reset (midnight Pacific) or enable billing on the project.
- **Prompt caching leg is really a latency-variance test.** Gemini's OpenAI-compatible endpoint doesn't expose explicit `cache_control` parameters or `cached_tokens` usage telemetry the way Anthropic's API does, so Leg 6 measures repeated-call latency spread instead of a true cold-cache-vs-warm-cache comparison. This is noted directly in the script's own printed output.
- **Local model schema compliance:** smaller local models are less reliable at following a strict JSON schema than the hosted API models. `ExplanatoryOutput` includes a `field_validator` that normalizes a common local-model quirk (returning `key_points` as objects instead of plain strings) instead of hard-failing, so this difference in compliance is captured as a finding rather than hidden as a bug.
- **Gemini tool-calling and `thought_signature`:** Gemini 3.x attaches an encrypted reasoning signature to function-call responses. The tool loop (Leg 5) forwards the model's full raw response back into the conversation (`assistant_msg.model_dump(exclude_none=True)`) rather than manually reconstructing it, since hand-picking only some fields silently drops this signature and causes the next call to fail.

## Sample results

| Category | Provider | Model | Quality | Latency | Cost | Status | Best Use |
|---|---|---|---|---|---|---|---|
| Frontier | Google | gemini-3.8-flash | High | 36.19s | $0.00057 | Validated | High-accuracy structured parsing |
| Reasoning | Google | gemini-3.6-flash | Very High | 22.37s | $0.00047 | Validated | Complex logical constraints |
| Local Open-Weights | Ollama | my-local-model:latest | Moderate | 24.63s | $0.00000 | Validated | Zero API cost, air-gapped/privacy-sensitive use |
| Multimodal | Google | gemini-3.8-flash | High | 6.60s | $0.00238 | Validated | OCR & document layout parsing |
| Tooling Loop | Google | gemini-3.8-flash | High | 10.08s | $0.00032 | Validated | Multi-step tool integration |

*(Latencies above include occasional 503-retry overhead from Gemini's free tier being under load; expect faster typical-case numbers on a paid tier or off-peak.)*

**When to pick each:**
- **Frontier/reasoning API models** — best when you need high accuracy and can't self-host; pick the reasoning-tier model specifically for tasks needing multi-step logical consistency, and the general frontier model for straightforward structured generation at lower per-token reasoning cost.
- **Local model (Ollama)** — best for zero marginal cost, offline/air-gapped environments, or strict data-privacy requirements; trade-off is lower schema compliance and generally lower output quality at small parameter counts.
- **Multimodal calls** — necessary whenever the input is an image or scanned document rather than plain text; cost is noticeably higher per call due to image token overhead.
- **Tool-calling loop** — appropriate whenever the model needs to fetch live/external data it doesn't already know, rather than trying to get it to hallucinate plausible-sounding facts.
