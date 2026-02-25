## Asset Marketing — Image catalog workflows (n8n)

This repo contains n8n workflows + notes used to build and enrich an image catalog from a shared Google Drive folder into a single Google Sheet.

### Source / destination
- **Drive root folder (Image Collection)**: `1vrNekG1Qr56NK1QjgFO2uocib1KKbnxU`
- **Google Sheet (Classification)**: `1bSdv2uqqvOqqg8zTGQAjHC4b6P4SS2chNt8iltit6t4`
- **Tabs used**: `Discovery`, `Level1`, `Level2`

### Workflows
- **Discovery (inventory)**: `Works/Build Master Image Collection Inventory from Google Drive (1).json`
  - Populates `Discovery` tab with one row per image (root + subfolders).
- **Level 1 enrichment**: `n8n_level1_sheet_enrichment.json`
  - Reads `Discovery`, skips already-processed `image_id`s in `Level1`, enriches missing rows via Azure OpenAI, appends to `Level1`.
- **Level 2 enrichment**: `n8n_level2_sheet_enrichment.json`
  - Reads `Level1`, skips already-processed `image_id`s in `Level2`, enriches missing rows via Azure OpenAI, appends to `Level2`.

### Notes
See `WORKFLOW_NOTES.md` for the latest status and the “story” for each flow.

### Secrets
Do not commit API keys. Set Azure endpoint/key inside n8n credentials or the workflow’s config nodes.

