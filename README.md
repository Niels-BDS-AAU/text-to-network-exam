# From Text to Network with LLMs  
### Danish EU Parliamentary Debates

This repo contains **my** solution to the module assignment *“From Text to Network with LLMs”*.


Short version:  
I take **EU Parliament speeches**, use a **Large Language Model (LLM)** to pull out **topics**, and then build **networks** that show how those topics hang together across speeches and speakers.

---

## 1. Goal & Research Question

**What am I trying to do?**

Turn messy debate speeches into structured data, and then into networks that I can actually analyze.

**Research question**

> How can a Large Language Model be used to extract speaker–topic relations from Danish EU debate speeches and, with network analysis, reveal which themes are central and how they are connected?

Or in pipeline form:

> **raw text → structured triples → networks → insights**

---

## 2. Dataset

**Source**

- Hugging Face: [`RJuro/eu_debates`](https://huggingface.co/datasets/RJuro/eu_debates)  
  EU Parliament debate speeches (many languages, many years).

**What I actually use**

- Only **Danish** speeches (`intervention_language == "DA"`).
- A selected year range (set with `YEAR_MIN` / `YEAR_MAX` in the notebook).
- Columns:
  - `speaker` (speaker_name)
  - `party` (speaker_party)
  - `year`
  - `speech_id` (based on date)
  - `text` (prefers `translated_text` if available)
- Sampling:
  - At most `SAMPLE_N` speeches.
  - Each speech is truncated to ~1800 characters before sending it to the LLM.

**Why this subset?**

- Small enough that LLM calls are realistic.
- Still diverse: multiple years, parties and speakers.
- One language (Danish) makes extraction and topic-normalisation easier.

---

## 3. Pipeline Overview

Everything lives in:

- `OPGAVE_Text_to_Network_Gemini.ipynb`

High-level steps:

1. Load and filter the EU debates (Danish + year range + sample).
2. Send each speech to an LLM (Gemini) and extract:
   - `speaker`, `party`,
   - `topics` (1–3 per speech),
   - simple triples `(speaker, mentions_topic, topic)`,
   - metadata (`speech_id`, `year`).
3. Save the extracted triples to CSV.
4. Clean and normalize the topic labels.
5. Do basic descriptive stats and plots (top topics, speakers, parties).
6. Build:
   - a **speaker–topic bipartite graph**, and
   - one or more **topic–topic networks**.
7. Calculate centrality measures and visualise the strongest connections.

---

## 4. LLM Extraction

### 4.1 Model

- LLM: **Google Gemini** (via `google.generativeai`).
- Model name is taken from the `LLM_MODEL` environment variable.
- Main settings:
  - `temperature = 0` (keep it stable),
  - `response_mime_type = "application/json"` (so we get JSON back directly).

### 4.2 Schema-based prompt

For each speech I ask the model to return **strict JSON** with this structure:

```json
{
  "speaker": "str",
  "party": "str|null",
  "topics": ["list[str] (1–3 short Danish nouns, lowercase)"],
  "triples": [
    {"subject": "<speaker>", "predicate": "mentions_topic", "object": "<topic>"}
  ],
  "meta": {"speech_id": "str", "year": 2022}
}
