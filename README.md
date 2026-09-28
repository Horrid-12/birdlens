# BirdLens 🐦

**BirdLens** is a small web app for identifying birds from photos and then showing useful information about the identified species.

It combines the **OSEA / DIB-10K** bird recognition pipeline with data from **eBird**, **iNaturalist**, and **Wikipedia** to turn a single image into something a little more useful than just a species name.

> **Note:** BirdLens is still a work in progress. AI identifications can be wrong, so results should be independently verified.

## What it does

- Upload a bird photo by clicking or dragging it into the upload area.
- Detect whether a bird is actually present in the image.
- Classify the detected bird using the OSEA model.
- Show the top candidate species and confidence.
- Return an explicit result when:
  - no bird is detected, or
  - a bird is detected but the model is not confident enough to identify the species.
- Show additional species information from external sources.
- Display recent sightings for the selected region.
- Keep a client-side history of the last 12 identifications, including thumbnails. This history lives only in the page and is cleared on reload.
- Show notable birds recently reported in the selected region.
- Support several Indian regions, an All India option, and a few international ones (United States, New York, United Kingdom, Australia).
- Handle JPG, PNG and WEBP uploads. HEIC/HEIF passes validation but needs an extra decoder to actually be identified — see [Image handling](#image-handling).
- Limit uploads to 20 MB.

## How the identification works

BirdLens currently uses a two-stage OSEA pipeline:

```text
Image
  ↓
Bird detection
  ↓
Crop detected bird
  ↓
DIB-10K / OSEA classification
  ↓
Confidence decision
  ↓
┌───────────────────────┬─────────────────────────┐
│ Bird not detected     │ Bird detected           │
│                       │ ↓                       │
│ "Not a bird"          │ Confident?              │
│                       │ ├─ Yes → identification │
│                       │ └─ No  → unable to ID   │
└───────────────────────┴─────────────────────────┘
```

The confidence decision is intentionally kept in its own module rather than being scattered through the application. It considers the detector confidence, the classifier's top-1 score, and the gap between the top two predictions.

The current defaults are:

- Detector confidence: **0.60**
- Minimum classifier top-1 probability: **0.05**
- Minimum top-1/top-2 margin: **0.03**

These values are configurable and should be recalibrated against a larger labelled test set before treating them as final.

There is one more threshold on the frontend: results scoring below **15%** are replaced with an "unable to identify" panel instead of the full results view. So a request can pass the server-side check and still be presented to the user as unidentified.

## Data sources

After a successful identification, BirdLens can query:

- **eBird** for taxonomy and recent sightings
- **iNaturalist** for observation counts and conservation information
- **Wikipedia** for a short species summary

The `/identify` response also includes a Wikipedia thumbnail URL, though the current interface only displays the summary text and the link back to the article.

The nearby sightings section is also powered by eBird's regional observations.

## Tech stack

### Backend
- Python
- Flask
- Gunicorn (WSGI server, optional for local runs)
- ONNX Runtime
- NumPy
- Pillow
- Requests

### AI
- OSEA bird identification pipeline
- Quantized **ResNet34** classifier
- Quantized **SSD MobileNet** bird detector
- DIB-10K species labels

### Frontend
- HTML
- CSS
- Vanilla JavaScript

There is deliberately no frontend framework or build step. The current frontend is served directly by Flask.

## Project structure

A simplified version of the project looks like this:

```text
BirdLens/
├── static/
│   ├── index.html
│   └── favicon.png
├── models/
│   ├── bird_model.onnx
│   ├── ssd_mobilenet.onnx
│   └── bird_info.json
├── app.py
├── osea_model.py
├── osea_confidence.py
├── requirements.txt
├── Procfile
├── .gitignore
└── README.md
```

The model assets are downloaded automatically into `models/` the first time OSEA is loaded if they are not already present. They are also committed to the repository, so a fresh clone already has them and no download occurs.

## Running locally

### 1. Clone the repository

```bash
git clone <your-repository-url>
cd BirdLens
```

### 2. Create a virtual environment

Windows:

```bash
python -m venv venv
venv\Scripts\activate
```

macOS / Linux:

```bash
python3 -m venv venv
source venv/bin/activate
```

### 3. Install dependencies

```bash
pip install -r requirements.txt
```

### 4. Configure the eBird API key

BirdLens uses the eBird API for taxonomy and sighting data.

Set your API key as an environment variable rather than committing it to the repository. There is no built-in fallback, so the app still starts without one, but identification results arrive with no taxonomy, no recent sightings, and an empty nearby section.

Windows PowerShell:

```powershell
$env:EBIRD_API_KEY="your-api-key"
```

Windows CMD:

```cmd
set EBIRD_API_KEY=your-api-key
```

macOS / Linux:

```bash
export EBIRD_API_KEY="your-api-key"
```

### 5. Start BirdLens

```bash
python app.py
```

To enable Flask debug mode (auto-reload, debugger), set the `FLASK_DEBUG` environment variable:

```bash
FLASK_DEBUG=1 python app.py
```

The application runs on:

```text
http://127.0.0.1:5000
```

On its first startup, OSEA downloads the required model assets if they are not already present.

## Deployment

The Flask application can also be loaded by a WSGI server such as Gunicorn.

The `Procfile` already wires this up:

```text
web: gunicorn app:app
```

Which is the same as running:

```bash
gunicorn app:app
```

Note that `app.py` loads the OSEA models at import time, so every Gunicorn worker loads its own copy. Keep the worker count low.

For temporary remote testing, a tunnel such as Cloudflare Tunnel can be placed in front of the local Flask server. This is useful for sharing the app with friends without deploying the application permanently.

## API endpoints

### `GET /`

Serves the BirdLens web interface.

### `POST /identify`

Accepts an uploaded image and returns the identification result.

Form data:

```text
image   uploaded image file
region  eBird region code, e.g. IN-MH
```

A region that does not match a country or country-state pattern is not rejected. It silently falls back to `IN-MH`, so an invalid region returns results for Maharashtra rather than an error.

The response includes the identification state, predictions, confidence information, taxonomy, recent sightings, and available external information.

### `GET /nearby`

Returns notable recent sightings for the selected region.

Example:

```text
/nearby?region=IN-MH
```

## Image handling

Uploads are accepted for these MIME types:

- `image/jpeg`
- `image/png`
- `image/webp`
- `image/heic`
- `image/heif`

Note that HEIC and HEIF are accepted by the server but not decodable — see the [caveat below](#heic--heif-caveat).

The server enforces a **20 MB upload limit** and rejects requests whose MIME type is not on the list above with a `415`.

The file extension is not used to reject anything. An unrecognised extension is quietly rewritten to `.jpg` before the upload is saved to a temporary file, so the MIME type is what actually decides whether a file is accepted.

Temporary uploaded files are removed after classification.

## A note on memory usage

OSEA's detector and classifier are loaded **once when the Flask process starts** and reused for subsequent requests. The application also includes lightweight process-memory logging through `psutil` when it is installed.

This is important because loading a fresh model for every image would be extremely expensive in both memory and startup time.

Memory usage can still temporarily spike during inference, particularly with large images, but the model sessions are intended to remain shared within the running process.

## Current limitations

BirdLens is still experimental, so there are a few things to keep in mind:

- Species identification is not guaranteed to be correct.
- Very similar species can be difficult to separate.
- Image quality, framing, lighting and obstructions can affect detection.
- HEIC/HEIF uploads are accepted but cannot be decoded, so they never identify. Use JPG, PNG or WEBP.
- The identification history is in-memory only and is lost when the page is reloaded.
- External API availability can affect taxonomy and sighting information.
- Without an `EBIRD_API_KEY` set, all eBird calls fail, so there is no taxonomy, no sightings and no nearby section.
- OSEA currently runs on the CPU.
- The model is loaded once per server process/worker. Running multiple Gunicorn workers therefore means each worker has its own model instance.
- The confidence thresholds are practical defaults, not a formal calibration.

## Credits

BirdLens currently builds on the following projects and data sources:

- **OSEA / DIB-10K** for bird detection and classification
- **eBird** for taxonomy and observations
- **iNaturalist** for biodiversity data
- **Wikipedia** for species summaries

The OSEA model assets used by this project come from the official `osea_mobile` release assets.

## Why BirdLens?

This started as a fairly simple idea: upload a bird photo and find out what it is.

The project has gradually turned into a little more than that, with the goal of making the result useful even after the identification itself. Instead of stopping at a species name, BirdLens tries to answer the next few questions too:

**What bird is this? Where is it usually seen? What do we know about it? And have people been seeing it nearby?**

---

**BirdLens** · AI-powered bird identification  
*Currently in development.*
