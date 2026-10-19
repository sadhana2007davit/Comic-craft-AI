# ComicCraft - AI Comic Story Creator using Gemini Models

Type a story idea, pick a character, setting, tone and art style, and ComicCraft
writes a five-panel comic, illustrates it and exports it as a PDF.

| Step | File | What happens |
|------|------|--------------|
| 1 | `app/gemini_flash.py` | Gemini 3.5 Flash-Lite plans a 5-panel outline (structured JSON output) |
| 2 | `app/gemini_pro.py` | Gemini 3.5 Flash writes captions, narration and dialogue |
| 3 | `app/image_generator.py` | One illustration per panel (Stable Diffusion or Gemini image model) |
| 4 | `app/layout_builder.py` | Images and story text are matched panel by panel |
| 5 | `app/exporters.py` | The comic is saved as a PDF with FPDF |

## Project structure

```
comiccraft/
├── app/
│   ├── main.py              # FastAPI app, static files, router
│   ├── routes.py            # all routes + the comic pipeline
│   ├── config.py            # settings read from .env
│   ├── gemini_client.py     # Google Gen AI SDK wrapper (Interactions API)
│   ├── gemini_flash.py      # generate_outline()
│   ├── gemini_pro.py        # generate_story()
│   ├── image_generator.py   # generate_image()
│   ├── layout_builder.py    # build_comic_layout()
│   └── exporters.py         # save_pdf()
├── templates/
│   ├── index.html           # story form
│   ├── comic_preview.html   # generated comic + download button
│   └── export_success.html  # confirmation page
├── static/
│   ├── panels/              # generated panel images
│   ├── exports/             # exported comic PDFs
│   ├── fonts/               # DejaVu fonts for the PDF
│   └── images/              # home page background
├── .vscode/launch.json      # press F5 to run in VS Code
├── .env                     # your API keys (never commit this)
├── .env.example
├── requirements.txt         # core dependencies
└── requirements-sd.txt      # optional: local Stable Diffusion
```

## Setup in VS Code

You need **Python 3.10 or newer** and the **Python extension** for VS Code.

1. **Open the folder.** Unzip the project, then in VS Code choose
   *File > Open Folder* and select the `comiccraft` folder (the one that
   contains `app`, `templates` and `requirements.txt`).

2. **Open a terminal.** *Terminal > New Terminal*.

3. **Create and activate a virtual environment.**

   Windows (PowerShell):
   ```powershell
   python -m venv comiccraft-env
   comiccraft-env\Scripts\activate
   ```
   If PowerShell blocks the script, run this once and try again:
   `Set-ExecutionPolicy -Scope CurrentUser RemoteSigned`
uvicorn app.main:app --reload
   macOS / Linux:
   ```bash
   python3 -m venv comiccraft-env
   source comiccraft-env/bin/activate
   ```

   Then press `Ctrl+Shift+P`, choose *Python: Select Interpreter* and pick
   `comiccraft-env`.

4. **Install the dependencies.**
   ```bash
   pip install -r requirements.txt
   ```

5. **Add your Gemini API key.** Create a key at
   <https://aistudio.google.com/apikey>, open `.env` and replace the placeholder:
   ```
   GEMINI_API_KEY=paste_your_key_here
   ```

6. **Run the server.**
   ```bash
   uvicorn app.main:app --reload
   ```
   Or press **F5** in VS Code (the launch configuration is included).

7. **Open the app.** <http://127.0.0.1:8000>
   API documentation: <http://127.0.0.1:8000/docs>

Restart the server whenever you change `.env`.

## Choosing how pictures are made

Set `IMAGE_BACKEND` in `.env`:

| Value | What it does | Cost |
|-------|--------------|------|
| `auto` (default) | Stable Diffusion if it is installed, otherwise the Gemini image model, otherwise title cards | - |
| `stable_diffusion` | Local Stable Diffusion through Hugging Face Diffusers | Free. About 6 GB of downloads. Fast with an NVIDIA GPU, several minutes per image on CPU |
| `gemini` | Gemini image model (`gemini-3.1-flash-image`) | Needs a billing-enabled key, roughly $0.07 per image |
| `placeholder` | Coloured title cards drawn locally | Free, instant |

If an image model fails, that panel gets a title card instead, so you always
receive a complete comic and PDF. The preview page tells you when this happened.

To use Stable Diffusion (as in the original project documentation):

```bash
pip install -r requirements-sd.txt
```

then set `IMAGE_BACKEND=stable_diffusion` in `.env` and restart. The first
comic downloads the model, which takes a while.

## About the Gemini models and the free tier

| Purpose | Setting | Default | Free tier |
|---------|---------|---------|-----------|
| Outline | `GEMINI_FLASH_MODEL` | `gemini-3.5-flash-lite` | Yes |
| Story | `GEMINI_PRO_MODEL` | `gemini-3.5-flash` | Yes |
| Pictures | `GEMINI_IMAGE_MODEL` | `gemini-3.1-flash-image` | No |

The Gemini 3.5 family has two text models and no Pro or image model. So the
fast one (Flash-Lite) plans the outline, the stronger one (Flash) writes the
story, and both run on a free key. If the story model fails, the outline model
writes the story instead.

Pictures cannot come from Gemini 3.5. On a free key use Stable Diffusion or
the title cards; with billing enabled the Gemini image model above works.

Model names change over time. The current list is at
<https://ai.google.dev/gemini-api/docs/models>; to switch, edit the three
settings above in `.env`.

## Testing

**1. Check the configuration.** Open <http://127.0.0.1:8000/health>.
`gemini_api_key_loaded` must be `true`.

**2. Test the pages and the PDF without any AI call.** Set
`COMICCRAFT_MOCK_AI=true` in `.env`, restart, and submit the form. You get a
sample story with title cards in a few seconds. Download the PDF and check
that you land on the "Comic Exported Successfully!" page. Set it back to
`false` afterwards.

**3. Test a single image.**
<http://127.0.0.1:8000/test-image?prompt=a%20brave%20fox&style=Anime>
The response shows the image path and which backend drew it.

**4. Test the full comic in the browser.** Enter
"A brave fox explores an enchanted forest.", character "Free", Forest,
Dramatic, Anime, and choose *Generate Comic*. Expect about a minute with
Gemini images, longer with Stable Diffusion on CPU.

**5. Test the JSON API.** In <http://127.0.0.1:8000/docs>, open
`POST /generate-comic/json`, choose *Try it out* and *Execute*. Or from a terminal:

Windows (PowerShell):
```powershell
Invoke-RestMethod -Uri http://127.0.0.1:8000/generate-comic/json -Method Post -ContentType "application/json" -Body '{"prompt":"A robot learns to paint","character_name":"Bolt","setting":"City","tone":"Funny","style":"Comic Book"}'
```

macOS / Linux:
```bash
curl -X POST http://127.0.0.1:8000/generate-comic/json \
  -H "Content-Type: application/json" \
  -d '{"prompt":"A robot learns to paint","character_name":"Bolt","setting":"City","tone":"Funny","style":"Comic Book"}'
```

## Routes

| Method | Path | Purpose |
|--------|------|---------|
| GET | `/` | Home page with the form |
| POST | `/generate` | Form submission, shows the comic preview |
| POST | `/generate-comic/json` | Same pipeline, JSON in and out |
| GET | `/export-success` | Confirmation page after the download |
| GET | `/test-image` | Generate one image |
| GET | `/health` | Show the active configuration |

## Troubleshooting

| Problem | Fix |
|---------|-----|
| "GEMINI_API_KEY is missing" | Put the key in `.env` (no quotes, no spaces) and restart the server |
| `ModuleNotFoundError: No module named 'app'` | Run `uvicorn` from the `comiccraft` folder, not from inside `app` |
| `ModuleNotFoundError` for fastapi, fpdf or google | The virtual environment is not active, or `pip install -r requirements.txt` was skipped |
| `cannot import name 'genai' from 'google'` | The old `google-generativeai` package is installed. Run `pip uninstall google-generativeai` then `pip install -U google-genai` |
| PDF import error mentioning `fpdf.enums` | The old `fpdf` package is installed. Run `pip uninstall fpdf` then `pip install -U fpdf2` |
| "model not found" or 404 from Gemini | The model was renamed or retired. Update the model names in `.env` |
| 429 / quota errors | Free-tier limit reached. Wait a minute and try again |
| Panels show title cards | No image model was available. See "Choosing how pictures are made" |
| Port 8000 already in use | `uvicorn app.main:app --reload --port 8001` |

## What changed from the original documentation

| Documentation | This project | Why |
|---------------|--------------|-----|
| `google-generativeai` package | `google-genai` with the Interactions API | The old package is discontinued; this is Google's current SDK |
| `gemini-1.5-flash`, `gemini-1.5-pro` | `gemini-3.5-flash-lite`, `gemini-3.5-flash` | The 1.5 models have been shut down; Gemini 3.5 has no Pro model |
| Outline JSON parsed from free text | Structured output with a JSON schema | The model can only answer in the required shape |
| `runwayml/stable-diffusion-v1-5` | `stable-diffusion-v1-5/stable-diffusion-v1-5` | The original repository was removed from Hugging Face; same weights |
| Stable Diffusion only | Stable Diffusion, Gemini images, or title cards | The app runs on any laptop, and a failed image never loses the comic |
| `fpdf` | `fpdf2` | Maintained successor with Unicode support; same `from fpdf import FPDF` |
| Story split on `**Panel` | Split by panel number with a regular expression | A missing or differently formatted heading no longer shifts the text |
