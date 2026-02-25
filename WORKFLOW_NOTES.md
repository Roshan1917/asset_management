# Asset Marketing — Image Catalog Automation (n8n)

## Latest status (as of today)

- **Google Drive**
  - Root folder (“Image Collection”) ID: `1vrNekG1Qr56NK1QjgFO2uocib1KKbnxU`
  - Folder contains **loose images in root** + **multiple subfolders**.
- **Google Sheet**
  - Spreadsheet: **Classification**
  - Spreadsheet ID: `1bSdv2uqqvOqqg8zTGQAjHC4b6P4SS2chNt8iltit6t4`
  - Tabs expected/used: **`Discovery`**, **`Level1`**, **`Level2`**
- **Discovery inventory**
  - Built using workflow: `Works/Build Master Image Collection Inventory from Google Drive (1).json`
  - Output is written/appended into **Classification → `Discovery`** tab.
- **Level 1 + Level 2 enrichment**
  - Templates available in repo:
    - `n8n_level1_sheet_enrichment.json`
    - `n8n_level2_sheet_enrichment.json`
  - These require Azure OpenAI endpoint/key and Google Drive + Google Sheets credentials in n8n.
- **Old (CSV-based) templates (not used for the “one Google Sheet” approach)**
  - `n8n_level1_drive_to_csv.json`
  - `n8n_level2_enrichment_to_csv.json`

---

## Overall approach (high-level)

We run automation in **three phases**:

1. **Discovery**: build the master list of images that exist (inventory).
2. **Level 1**: generate basic metadata + broad classification for each image.
3. **Level 2**: generate detailed domain classification + tags + color classification.

The key concept is using **`image_id` (Drive file ID)** as the unique key so workflows can be **restart/resume-safe** and avoid duplicates.

---

## Flow 1 — Discovery (Master Image Inventory)

**Workflow file:** `Works/Build Master Image Collection Inventory from Google Drive (1).json`

### Story (what this flow achieves)

You want to know **everything that exists** in the Image Collection so downstream enrichment is stable.

This flow:
- lists **loose/root images** directly under “Image Collection”
- lists all **subfolders**
- lists all **images inside each subfolder**
- standardizes every file into a consistent row structure
- writes/appends into **Classification → `Discovery`** tab

### Output (Discovery tab)

One row per image with at least:
- `folder_order`
- `folder_id`
- `image_id`
- `file_name`
- `image_url`
- `mime_type`
- `file_size_bytes`
- `source_path`
- `discovered_at`

### Operational notes
- If new assets are added later, re-run Discovery to refresh/extend the inventory.
- Ensure the `Discovery` tab has a header row so mapping in n8n is clean.

---

## Flow 2 — Level 1 Enrichment (Basic metadata + broad classification)

**Workflow template:** `n8n_level1_sheet_enrichment.json`

### Story (what this flow achieves)

You take the Discovery inventory and convert each image into a searchable catalog entry:
- **Heading / title** (short)
- **Description** (1–2 lines)
- **General classification** (broad bucket)
- carry forward useful technical metadata + source information

### How it runs (conceptually)

- Reads **Classification → `Discovery`** as the “to-do list”.
- Reads **Classification → `Level1`** to learn what’s already processed.
- Builds a queue of **only missing `image_id`s** (skip existing).
- For each queued image:
  - downloads the file from Drive by `image_id`
  - sends it to Azure OpenAI (vision) to generate Level 1 fields
  - appends a row to **`Level1`**

### Why this is safe to stop/restart

Because it always checks `Level1` first, you can:
- stop in the middle
- rerun later
and it will **skip any `image_id` already present** in `Level1`.

### What you must configure in n8n before running

- Google Sheets credential
- Google Drive credential
- Azure:
  - `azureEndpoint`
  - `azureApiKey`
- Make sure the `Level1` tab exists and has a header row.

---

## Flow 3 — Level 2 Enrichment (Detailed domain + tags + colors)

**Workflow template:** `n8n_level2_sheet_enrichment.json`

### Story (what this flow achieves)

You take the Level 1 catalog and enrich it into marketing-friendly indexing:
- **Sub-classification** (detailed domain under the general classification)
- **Tags** (3–5 one-word keywords)
- **Color classification** (dominant colors)

### How it runs (conceptually)

- Reads **Classification → `Level1`** as the input list.
- Reads **Classification → `Level2`** to see what’s already enriched.
- Builds a queue of **only missing `image_id`s** (skip existing).
- For each queued image:
  - downloads the file from Drive by `image_id`
  - calls Azure OpenAI to generate Level 2 fields
  - appends into **`Level2`**

### Why this is safe to stop/restart

Same mechanism as Level 1: it checks `Level2` first and skips existing `image_id`s.

### What you must configure in n8n before running

- Google Sheets credential
- Google Drive credential
- Azure endpoint + key
- Make sure the `Level2` tab exists and has a header row.

---

## Recommended sheet headers (Row 1)

### `Discovery`
`folder_order, folder_id, image_id, file_name, image_url, mime_type, file_size_bytes, source_path, discovered_at`

### `Level1`
`image_id, file_name, heading, description, general_classification, mime_type, file_size_bytes, source_path, folder_id, folder_order, image_url, processed_at`

### `Level2`
`image_id, file_name, heading, description, general_classification, sub_classification, tags, color_classification, mime_type, file_size_bytes, source_path, folder_id, folder_order, image_url, processed_at`

---

## Run order (day-to-day)

1. Run **Discovery** (only when you want to refresh inventory / new images appear).
2. Run **Level 1** (fills `Level1` for any images missing).
3. Run **Level 2** (fills `Level2` for any images missing).

