# Update Rules

When asked to update rules from the base repository:

## Step 1 — Download and extract
```bash
mkdir -p /tmp/ia-rules-update
curl -L https://github.com/PauloFerreira25/.ia/archive/refs/heads/main.zip \
  -o /tmp/ia-rules-update/rules.zip
unzip /tmp/ia-rules-update/rules.zip -d /tmp/ia-rules-update/
```

## Step 2 — Compare and update
For each `.md` file in the downloaded `.ia/rules/` directory:
- Calculate MD5 hash of both downloaded and local versions
- If hashes differ: compare line by line
- Update or create local files as needed
- Report: which files were modified, added, or kept unchanged
- Never update without user confirmation

## Step 3 — Cleanup
```bash
rm -rf /tmp/ia-rules-update
```

Never:
- Skip file comparison
- Delete local rules without backup
- Update without user confirmation
