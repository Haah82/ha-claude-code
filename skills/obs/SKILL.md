---
name: obs
description: Save a markdown article to the Haah82/obsidian_haah GitHub repository at a specified
  path. Use this skill when the user types `/obs {path} {filename.md}` and has a markdown article
  ready to be saved. Handles both create (new file) and update (existing file).
origin: ha-claude-code
---

# Obsidian GitHub Save (`/obs`)

Skill for publishing markdown articles directly to `github.com/Haah82/obsidian_haah` via the GitHub MCP tools. Supports creating new files and updating existing ones, with automatic SHA lookup to prevent overwrite conflicts.

## When to Use

- User types `/obs {path} {filename.md}` with a markdown article in context.
- Saving a newly written article (book review, essay, analysis) to the Obsidian vault.
- Updating an existing note in the vault with revised content.
- Any time the user wants to persist a markdown document to the `obsidian_haah` repository.

**Trigger pattern:** `/obs "2. Track/2026 May" Ha-noi-tinh-hoa.md`

## Parameters

| Parameter | Description | Example |
|-----------|-------------|---------|
| `path` | Directory path inside the repo (no leading `/`) | `2. Track/2026 May` |
| `filename.md` | Target filename with `.md` extension | `Ha-noi-tinh-hoa.md` |

The full file path in the repo will be: `{path}/{filename.md}`

## Core Concepts

- **Repo target**: Always `Haah82/obsidian_haah`, branch `main`
- **SHA requirement**: GitHub API requires the current blob SHA when updating an existing file. This skill fetches it automatically.
- **Content to save**: The most recent markdown article written or discussed in the current conversation.
- **Obsidian format**: Articles must have YAML frontmatter, a `# Title`, and follow the vault's formatting conventions.

## How It Works

### Step 1: Parse Command

Extract `path` and `filename` from the user's `/obs` invocation:
- `/obs "2. Track/2026 May" Ha-noi-tinh-hoa.md` → path = `2. Track/2026 May`, file = `Ha-noi-tinh-hoa.md`
- Full repo path = `2. Track/2026 May/Ha-noi-tinh-hoa.md`

### Step 2: Check if File Already Exists

Call `mcp__github__get_file_contents` to check whether the file exists:

```
owner: haah82
repo: obsidian_haah
path: {path}/{filename.md}
ref: main
```

- **If file exists**: extract the `sha` field from the response — required for the update call.
- **If file does not exist** (error returned): proceed with `sha` = `null` (omit the field).

### Step 3: Identify the Article Content

Use the markdown article that was most recently written or discussed in the conversation. If the conversation contains multiple articles, confirm with the user which one to save.

Verify the content has:
- YAML frontmatter (between `---` delimiters)
- A `# Title` heading
- Proper Obsidian formatting (no unnecessary `---` separators between sections)

### Step 4: Create or Update the File

Call `mcp__github__create_or_update_file`:

```
owner: haah82
repo: obsidian_haah
path: {path}/{filename.md}
branch: main
message: "docs: save {filename} to {path}"
content: {base64-encoded article content}
sha: {sha from Step 2, or omit if new file}
```

### Step 5: Confirm Success

Report to the user:
- Full GitHub URL of the saved file
- Commit SHA
- Whether it was a create or update operation

## Example Workflow

**User command:** `/obs "2. Track/2026 May" Ha-noi-tinh-hoa.md`

**Assistant actions:**
1. Parse: path = `2. Track/2026 May`, file = `Ha-noi-tinh-hoa.md`
2. Check existence → file not found → no SHA needed
3. Take the Hanoi housing article from conversation
4. Call `mcp__github__create_or_update_file` → success
5. Report: "Saved to `2. Track/2026 May/Ha-noi-tinh-hoa.md` on `main` — commit `abc1234`"

**Update scenario:** If the file already exists:
1. `mcp__github__get_file_contents` returns `sha: "47fb905cbc30ae..."`
2. Pass this SHA in the update call
3. File is updated without conflict

## Obsidian Format Checklist

Before saving, verify the article follows vault conventions:

- [ ] YAML frontmatter with `aliases`, `created`, `tags`, `category`, `author`, `source`
- [ ] `# Title` as the first heading after YAML
- [ ] `*Italic subtitle*` on the next line
- [ ] `---` separator only after subtitle, after opening paragraph, and after Mục Lục
- [ ] Numbered sections `## N. Section Name`
- [ ] Mục Lục with `[[#Section Name]]` links
- [ ] Callout blocks using lowercase Obsidian syntax: `> [!info]`, `> [!note]`, etc.
- [ ] No `---` between regular sections or subsections

## Error Handling

| Situation | Action |
|-----------|--------|
| File exists, no SHA passed | API returns 422 — retry with SHA from `get_file_contents` |
| Path contains spaces | URL-encode when needed; the MCP tool handles encoding |
| No article in context | Ask user to provide or paste the article content |
| Permission denied | Verify repo is `haah82/obsidian_haah` (not ha-claude-code) |
| Branch not `main` | Default to `main`; ask user if they want a different branch |

## Related Commands

- `/ha-obs-format` — format an article to Obsidian conventions before saving
- `/ha-vn-pmp` — generate Vietnamese public investment project documents
- `/obs {path} {file.md}` — this skill

## Related Skills

- `obsidian-formatting`
- `github-file-ops`
- `markdown-writer`
