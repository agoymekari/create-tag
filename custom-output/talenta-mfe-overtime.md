# Custom output — talenta-mfe-overtime

After the tag is created and pushed (Step 6), also gather and append:

- **MFE name** — read the `name` field from this repo's `package.json` at its root
  (currently `@talenta/mf-overtime`).
- **Commit** — the full commit hash the tag was created on (already known from this run).
- **Tag** — the tag version just created (already known from this run).

Present it as:

```
MFE: <name>
Commit: <full-commit-hash>
Tag: <tag-name>
```
