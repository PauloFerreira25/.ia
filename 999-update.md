# Update .ia

When asked to update from the base repository:

## Step 1 — Download and extract
```bash
mkdir -p /tmp/ia-update
curl -L https://github.com/PauloFerreira25/.ia/archive/refs/heads/main.zip \
  -o /tmp/ia-update/repo.zip
unzip /tmp/ia-update/repo.zip -d /tmp/ia-update/
```

## Step 2 — Overwrite .ia content
Copy the entire `.ia/` directory from the downloaded repo into the project root, **overwriting** all existing files:

```bash
cp -r /tmp/ia-update/.ia-main/.ia/. /path/to/project/.ia/
```

- Overwrite all files from the base repo (rules, architecture, playbooks, root files)
- Preserve any project-specific files (numbered 30+) that exist only locally
- Never delete local-only files — only update or add files present in the base repo

## Step 3 — Cleanup
```bash
rm -rf /tmp/ia-update
```

## Step 4 — Report
List which files were added, updated, or kept unchanged. Never apply changes without user confirmation.
