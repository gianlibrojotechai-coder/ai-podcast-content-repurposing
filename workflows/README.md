# Workflow JSON

Version-controlled exports of the two n8n workflows in this project.

| File | Workflow | ID |
| --- | --- | --- |
| `podcast-pipeline.json` | Podcast to Content Pipeline | `<main-workflow-id>` |
| `error-handler.json` | Podcast Pipeline Error Handler | `<error-workflow-id>` |

## Exporting

**From the n8n UI** (simplest):

1. Open the workflow
2. `⋯` menu (top right) → **Download**
3. Save into this folder with the filename from the table above

**From the CLI**, if you have shell access to the n8n container:

```bash
docker exec -it <n8n-container> n8n export:workflow \
  --id=<main-workflow-id> \
  --output=/tmp/podcast-pipeline.json
docker cp <n8n-container>:/tmp/podcast-pipeline.json ./
```

## Importing into another instance

```bash
docker cp podcast-pipeline.json <n8n-container>:/tmp/
docker exec -it <n8n-container> n8n import:workflow --input=/tmp/podcast-pipeline.json
```

## Before you commit an export

n8n exports reference credentials **by ID and name**, not by value — no API keys are written to the JSON. Confirm anyway:

```bash
grep -iE '"(apiKey|accessToken|password|token)"\s*:\s*"[^"]+"' *.json
```

That should return nothing. If it returns a match, the export included resolved credential data — do not commit it.

## What does not survive an export

- **Credentials themselves.** The importing instance needs its own credentials created with matching names, then re-attached to the nodes.
- **The Error Workflow setting.** Re-select `Podcast Pipeline Error Handler` under the main workflow's Settings after import.
- **Active state.** Imported workflows arrive inactive. Intentional — activate deliberately.
