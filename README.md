# ProtoNova – Slide Generator (LLM → JSON → PowerPoint)

This project turns a short prompt into a **PowerPoint deck** by asking an LLM (DeepSeek via an OpenAI‑compatible API) to create a **JSON slide specification**, validating it, and then rendering slides with **python‑pptx**.

> TL;DR flow: **Prompt → (LLM) JSON spec → (jsonschema) Validate → (python‑pptx) Render .pptx**

---

## Table of contents
- [What the code does](#what-the-code-does)
- [Key files](#key-files)
- [Dependencies](#dependencies)
- [Environment variables](#environment-variables)
- [How it works (step by step)](#how-it-works-step-by-step)
- [Code tour (function by function)](#code-tour-function-by-function)
- [Slide schema & types](#slide-schema--types)
- [Run it](#run-it)
- [Troubleshooting](#troubleshooting)
- [Roadmap / ideas](#roadmap--ideas)

---

## What the code does

1. **Parses CLI arguments** like `--prompt`, `--out`, `--model`.
2. **Calls DeepSeek** with a strict system prompt asking for **JSON only** that matches a schema (`SLIDE_SPEC_SCHEMA`).
3. **Extracts** the JSON block from the LLM response (`extract_json_like`) and **parses** it.
4. **Validates** the JSON against a schema using **jsonschema**.
5. **Renders** a PowerPoint deck (`.pptx`) using **python-pptx** according to slide `type` (title, bullet, image, two_column).
6. **Saves** the deck to the `--out` path (default: `output.pptx`).

---

## Key files

- `GeneratePPT.ipynb` — a Jupyter Notebook where you can run the pipeline interactively.
- `script.py` (or equivalent) — Python source that implements the CLI and all helpers (shown in the snippets below).
- Generated output: `output.pptx` (or your chosen name).

> In your current code, the main logic is embedded in Python cells/functions. This README documents those functions & parts explicitly.

---

## Dependencies

- `python-pptx` — creates and edits PowerPoint files.
- `Pillow` (PIL) — used for placeholder images/text rendering if you add `generate_placeholder_image`.
- `jsonschema` — validates the LLM’s JSON.
- `openai` — OpenAI-compatible SDK used to call **DeepSeek** (`base_url` is set to `https://api.deepseek.com`).
- (Optional) `python-dotenv` — load `.env` for local development.

Install (example):
```bash
pip install python-pptx pillow jsonschema openai python-dotenv
```

---

## Environment variables

- `DEEPSEEK_API_KEY` — **required** for the DeepSeek call.
- (Optional) `OPENAI_API_KEY` — if you later add image generation with OpenAI images.

You can set variables in your shell, a `.env` file (with `python-dotenv`), or directly inside a Notebook cell:
```python
import os
os.environ["DEEPSEEK_API_KEY"] = "sk-..."
```

---

## How it works (step by step)

1. **Collect input**  
   From CLI (`--prompt`) or interactive prompt if missing.

2. **Call the LLM** (`call_deepseek_llm`)  
   Sends a **system prompt** that strictly instructs the model to output **only JSON** that matches the schema (title + slides).

3. **Extract JSON** (`extract_json_like`)  
   Pulls the first balanced `{...}` block from the model’s text (also strips code fences).

4. **Validate JSON** (`jsonschema.validate`)  
   Ensures the structure matches `SLIDE_SPEC_SCHEMA`. If invalid, prints errors and dumps the received JSON for debugging.

5. **Render PPTX** (`render_pptx_from_spec`)  
   Creates a new `Presentation()` and loops over `slides`. For each slide:
   - Choose a layout.
   - Add title/subtitle/bullets/images (depending on type).
   - Save the finished deck.

---

## Code tour (function by function)

### 1) `SLIDE_SPEC_SCHEMA` (global)
Defines the expected JSON that the LLM must return:
```json
{
  "title": "string",
  "slides": [
    {
      "type": "title_slide | bullet_slide | image_slide | two_column",
      "title": "string",
      "subtitle": "string?",
      "bullets": ["string", "..."]?,
      "image_prompt": "string?",
      "notes": "string?"
    }
  ]
}
```
- Guarantees the presence of `title` and `slides` at the top level.
- Enforces `type` and `title` on each slide.

### 2) `call_deepseek_llm(user_prompt, model="deepseek-chat", temperature=0.2)`
- Initializes an OpenAI client **pointing to DeepSeek** using `DEEPSEEK_API_KEY` and a `base_url` of `https://api.deepseek.com`.
- Builds messages:
  - **system**: “you are a slide authoring assistant… return JSON only…”
  - **user**: whatever `--prompt` you supplied.
- Calls `client.chat.completions.create(...)` to get a response.
- Extracts JSON via `extract_json_like` and returns `dict`.

### 3) `extract_json_like(s: str) -> str`
- Strips code fences if present.
- Finds the first `{ ... }` block using `find/rfind` and returns it.
- Throws a clear error if it can’t find a JSON object.

> Why this matters: LLMs sometimes add text around JSON; this function isolates the JSON segment.

### 4) `render_pptx_from_spec(spec: Dict[str, Any], out_path="output.pptx")`
- Creates a basic `Presentation()` (default template).
- Loops over `spec["slides"]` and branches by `type`:
  - **`title_slide`**
    - Uses layout 0 (title) if available.
    - Sets `title` and optional `subtitle`.
  - **`bullet_slide`**
    - Uses layout 1 (title + content) if available.
    - Puts the slide title and renders bullet points into the body placeholder.
    - **Note:** current code references `generate_placeholder_image` when `image_prompt` is present and places an image on the right—this function must exist in your code; otherwise it will error. If you haven’t added it yet, either:
      - implement it, or
      - remove the image block to avoid exceptions.
  - **`two_column`**
    - Adds a left textbox (bullets go there). Right column is **not implemented** in the current code.
  - **`image_slide`**
    - Adds a title and an optional caption area at the bottom.
    - **Important:** the current code **does not actually insert an image** for `image_slide`. If you want a real image here, you need to add the logic (see roadmap below).
- Saves the deck to `out_path` and prints a done message.

### 5) `main()`
- Parses arguments: `--prompt`, `--out`, `--model`.
- If `--prompt` is empty, asks interactively.
- Calls `call_deepseek_llm` → `spec`.
- Validates `spec` against `SLIDE_SPEC_SCHEMA` (warns on failure; still tries to render).
- Calls `render_pptx_from_spec(spec, out_path=args.out)`.

> **Edge cases to be aware of:**
> - If the LLM call fails, `spec` can be `None`. The renderer will then fail (because it expects a dict) unless guarded.
> - If `generate_placeholder_image` is not implemented but `image_prompt` exists, the `bullet_slide` branch will raise an error.

---

## Slide schema & types

### `title_slide`
- Fields used: `title`, `subtitle?`

### `bullet_slide`
- Fields used: `title`, `bullets?`, `image_prompt?`
- Body text goes into the content placeholder. If `image_prompt` is present, the sample code tries to add a **placeholder** image on the right (requires you to define `generate_placeholder_image`).

### `image_slide`
- Fields used: `title`, `bullets?` (as caption)
- **Currently no image is added**. You’ll want to extend this to display a hero image using `image_prompt` (see  roadmap).

### `two_column`
- Fields used: `title`, `bullets?`
- Only the **left** column is implemented in the current code.

---

## Run it

### CLI (script form)
```bash
python script.py --prompt "Explain diffusion models to product managers" --out diffusion.pptx
```

### Jupyter Notebook
- Ensure `DEEPSEEK_API_KEY` is set (via `os.environ[...]` or a `.env`).
- Run cells in order: define helpers → run the `main()` cell or call functions directly.

---

## Troubleshooting

- **`openai package not available`**: `pip install openai`.
- **`DEEPSEEK_API_KEY 未设置`**: set the environment variable (shell, .env, or Notebook cell).
- **`无法从模型输出中提取 JSON`**: the model returned non‑JSON; inspect the raw output, adjust the prompt, or make `extract_json_like` stricter.
- **`jsonschema` validation fails**: check that slides have mandatory `type` and `title`, and fields are of correct types.
- **`generate_placeholder_image` not defined**: implement it or remove the image block in `bullet_slide` to avoid exceptions.
- **PowerPoint layout mismatch**: default layout indices depend on template. If you use a custom template, verify `slide_layouts[...]` indices.

---

## Roadmap / ideas

- **Robust defaults and guards**: sanitize `spec`, hard‑limit to 10 slides, bail out early if `spec is None`.
- **Real images**: accept URLs/paths or generate images from text prompts; add image logic for `image_slide`.
- **Two columns**: implement the right column and an optional `right_bullets` field.
- **Speaker notes**: write `notes` into `slide.notes_slide.notes_text_frame`.
- **Templates & branding**: allow a `--template` to load a branded PowerPoint theme.
- **Better JSON extraction**: fence‑aware, balanced‑brace parsing with validity check.
- **Retries & logging**: backoff on API failures, structured logs.
- **Unit tests** for JSON extraction, schema validation, and slide rendering.

---

## Example slide spec (what the LLM returns)

```json
{
  "title": "Intro to Retrieval-Augmented Generation",
  "slides": [
    { "type": "title_slide", "title": "Retrieval-Augmented Generation", "subtitle": "Why, when, how" },
    { "type": "bullet_slide", "title": "Motivation", "bullets": ["Reduce hallucinations", "Ground answers in data", "Enable citations"], "image_prompt": "knowledge base icon" },
    { "type": "two_column", "title": "System Components", "bullets": ["Indexer", "Retriever", "Generator"] },
    { "type": "image_slide", "title": "RAG Architecture", "bullets": ["High-level view of pipeline"] }
  ]
}
```


GeneratePPT.ipynb: 用户输入一段提示词，PPT agent根据提示词内容输出一个纯文字PPT。（后续会加入生成配图、指定模板等功能）
