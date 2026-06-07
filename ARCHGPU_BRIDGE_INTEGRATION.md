# ARCHGPU Bridge Integration (Custom Open WebUI Fork)

This document describes the customizations added on top of Open WebUI to
integrate ARCHGPU bridge model management directly inside the existing WebUI.

Related bridge backend repository (runtime/service side):

- [https://github.com/nitty01/archgpu-ollama-bridge](https://github.com/nitty01/archgpu-ollama-bridge)

## What Is Customized

The following custom behavior is added:

- **No separate UI page** for catalogue management
- Model selector now includes:
  - installed models (normal behavior)
  - downloadable models (catalogue-backed)
  - in-place download action
  - live download progress display
- Quick delete action for installed bridge/Ollama models
- Admin `Model Explorer` tab for:
  - full-capacity HF GGUF index (server-side, not client-side scrolling)
  - classic pagination (first/prev/next/last, page size, total count)
  - debounced server-driven search with field filters (`repo:`, `publisher:`, etc.)
  - sortable columns (display name, downloads, fit)
  - metadata (freshness/downloads/likes/publisher/pipeline)
  - quality filters (trusted publishers + thresholds)
  - runtime fit recommendations (download guidance)
  - transfer status strip and pull lifecycle controls (stop/resume/restart/purge/clear)

## Why These Changes Were Made

- Keep the full model lifecycle inside WebUI so users do not need a separate
  bridge management UI.
- Expose richer model metadata and quality signals to improve model selection.
- Surface runtime-fit guidance so users avoid downloading models likely to be
  too heavy for their system profile.
- Make disk cleanup practical with direct delete actions for bridge-managed
  dynamic models.

## Files Changed

- `backend/open_webui/routers/ollama.py`
  - Adds pass-through endpoints:
    - `GET /ollama/api/catalogue` (forwards pagination, search, sort, filter params)
    - `GET /ollama/api/pull/status`
    - `POST /ollama/api/pull/control`
- `src/lib/apis/ollama/index.ts`
  - Adds `getOllamaCatalogue` with `OllamaCatalogueEnvelope` pagination support.
  - Adds `controlOllamaPull` and `getOllamaPullStatus`.
- `src/lib/components/chat/ModelSelector/Selector.svelte`
  - Adds downloadable model section and pull trigger in selector dropdown.
- `src/lib/components/chat/ModelSelector/ModelItem.svelte`
  - Adds quick delete button in model list row.
- `src/lib/components/admin/Settings/ModelExplorer.svelte`
  - Adds full admin explorer for metadata/filter/recommendation workflows.
- `src/lib/components/admin/Settings.svelte`
  - Adds Model Explorer tab and routing.

## Build and Run (Custom Image)

```bash
cd /path/to/open-webui-src
docker build -t openwebui:archgpu-catalogue .
```

Run with your existing bridge stack helper:

```bash
cd /path/to/ARCHGPU_OLLAMA_BRIDGE
RECREATE_OPENWEBUI=1 OPENWEBUI_IMAGE=openwebui:archgpu-catalogue ./scripts/stack.sh up
```

## Usage

1. Open WebUI (`http://127.0.0.1:3000`).
2. Log in as admin.
3. Open **Admin Settings → Model Explorer** for paginated HF catalogue browsing.
4. Use search, filters, and page controls to find models (e.g. `repo:ggml-org/gemma-4`).
5. Open model selector from chat input to download from the **Downloadable** section.
6. Monitor download progress in the selector or Model Explorer transfer status strip.

### Model Explorer pagination

Model Explorer requests one page at a time from the bridge:

- default page size: 25 (configurable in UI)
- server params: `page`, `page_size`, `q`, `sort_by`, `sort_dir`, `installed`, `downloadable`, `capability`, `publisher`, `source`, `fit`
- first HF index build may take minutes; use **Refresh Live Catalogue** to force rebuild

Bridge env vars controlling index size and cache:

- `ARCHGPU_BRIDGE_HF_INDEX_MAX_MODELS` (default `5000`)
- `ARCHGPU_BRIDGE_HF_INDEX_TTL_SECONDS` (default `3600`)
- `ARCHGPU_BRIDGE_HF_INDEX_CACHE_PATH` (default `data/hf_gguf_index.json`)

## Compatibility Disclaimer

This custom fork is tested in a local environment on Ubuntu 26.04 with:

- Intel Arc GPU host setup
- Dockerized Open WebUI
- ARCHGPU bridge service exposing Ollama-compatible endpoints

Behavior on other operating systems, kernels, or Docker/network setups is not guaranteed.
Validate in your own environment before production usage.

## Publish This Fork To Your GitHub

```bash
cd /path/to/open-webui-src
git checkout -b archgpu-bridge-integration
git add backend/open_webui/routers/ollama.py src/lib/apis/ollama/index.ts src/lib/components/chat/ModelSelector/Selector.svelte src/lib/components/chat/ModelSelector/ModelItem.svelte src/lib/components/admin/Settings/ModelExplorer.svelte src/lib/components/admin/Settings.svelte README.md ARCHGPU_BRIDGE_INTEGRATION.md
git commit -m "Add native WebUI model explorer and bridge integration"

# set your own fork repo URL if needed
git remote set-url origin https://github.com/<your-user>/<your-openwebui-fork>.git
git push -u origin archgpu-bridge-integration
```
