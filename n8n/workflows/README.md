# n8n workflow exports

To add a workflow: in n8n, open it, then go to ⋯ → **Download**. Save the file here as `<name>.json`.

Before committing, search the file for `apiKey`, `token`, `password` and `Bearer`. n8n exports only credential *names and IDs*, never the secrets themselves, but double-check anyway.
