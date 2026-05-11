# Skills do Mau

Personal collection of AI agent skills. Each skill follows the [Agent Skills Specification](https://agentskills.io/specification).

## Ground Rules for maintaining this repository

The following are ground rules for maintaining this repository (and not for the skills themselves):

### Committing changes

**Always ask before committing changes to git.** Never run `git commit` without explicit user confirmation

### Commit Messages

Follow this format for all commits:

```
<Summary of the changes>

- <Change 1>: <Reasoning>
- <Change 2>: <Reasoning>
- ...
```

The summary line should describe the intent in plain language. Each bullet should state what changed and why, not just what was done.

**Example:**

```
Update omarchy-check-updates to assess Intel Xe impact

- Add lsmod check for xe/i915 modules: ensures we only flag relevant updates
- Include /proc/cmdline inspection: detects if panel replay override is active
- Add limine/cmdline config search: matches release notes against boot config
```

## Structure

A skill is a directory containing, at minimum, a `SKILL.md` file:

```
skill-name/
├── SKILL.md          # Required: metadata + instructions
├── scripts/          # Optional: executable code
├── references/       # Optional: documentation
├── assets/           # Optional: templates, resources
└── ...               # Any additional files or directories
```

## `SKILL.md` Format

The `SKILL.md` file must contain YAML frontmatter followed by Markdown content.

### Frontmatter

| Field           | Required | Constraints                                                                                                       |
| --------------- | -------- | ----------------------------------------------------------------------------------------------------------------- |
| `name`          | Yes      | 1–64 chars. Lowercase letters, numbers, hyphens only. No leading/trailing hyphen. No consecutive hyphens. Must match directory name. |
| `description`   | Yes      | 1–1024 chars. Describes what the skill does **and** when to use it. Include keywords to help agents match tasks. |
| `license`       | No       | License name or reference to a bundled license file. Keep it short.                                               |
| `compatibility` | No       | 1–500 chars if provided. Only include if your skill has specific environment requirements. Most skills do not need this. |
| `metadata`      | No       | Map of string keys to string values. Use reasonably unique key names to avoid conflicts.                          |
| `allowed-tools` | No       | Space-separated string of pre-approved tools. (Experimental — support varies by agent.)                           |

**Minimal example:**

```markdown
---
name: skill-name
description: A description of what this skill does and when to use it.
---
```

**Example with optional fields:**

```markdown
---
name: pdf-processing
description: Extract PDF text, fill forms, merge files. Use when handling PDFs.
license: Apache-2.0
metadata:
  author: mau
  version: "1.0"
---
```

#### `name` field constraints

- Must be 1–64 characters.
- May only contain lowercase alphanumeric characters (`a-z`, `0-9`) and hyphens (`-`).
- Must not start or end with a hyphen.
- Must not contain consecutive hyphens (`--`).
- Must match the parent directory name.

**Valid:** `pdf-processing`, `data-analysis`, `code-review`  
**Invalid:** `PDF-Processing` (uppercase), `-pdf` (leading hyphen), `pdf--processing` (consecutive hyphens)

#### `description` field guidance

- Should describe **both** what the skill does **and** when to use it.
- Should include specific keywords that help agents identify relevant tasks.

**Good:** `Extracts text and tables from PDF files, fills PDF forms, and merges multiple PDFs. Use when working with PDF documents or when the user mentions PDFs, forms, or document extraction.`  
**Poor:** `Helps with PDFs.`

#### `compatibility` field examples

```yaml
compatibility: Designed for Claude Code (or similar products)
```

```yaml
compatibility: Requires git, docker, jq, and access to the internet
```

### Body Content

The Markdown body after the frontmatter contains the skill instructions. There are no format restrictions — write whatever helps agents perform the task effectively.

The agent will load this entire file once it's decided to activate a skill. **Consider splitting longer `SKILL.md` content into referenced files.**

Recommended sections:

- **Step-by-step instructions**
- **Examples** — inputs and outputs
- **Common edge cases**

Optional but useful sections:

- **When To Use** — bullet list of triggers that should activate this skill.
- **Safety Rules** — if the skill touches system state, explicitly list what NOT to do.
- **Workflow** — step-by-step commands or logic. Prefer copy-pasteable shell snippets.
- **Report Format** — if the skill produces output, describe the expected structure.

## Optional Directories

### `scripts/`

Contains executable code that agents can run. Scripts should:

- Be self-contained or clearly document dependencies.
- Include helpful error messages.
- Handle edge cases gracefully.

Supported languages depend on the agent implementation. Common options include Python, Bash, and JavaScript.

### `references/`

Contains additional documentation that agents can read when needed:

- `REFERENCE.md` — detailed technical reference.
- `FORMS.md` — form templates or structured data formats.
- Domain-specific files (`finance.md`, `legal.md`, etc.).

Keep individual reference files focused. Agents load these on demand, so smaller files mean less use of context.

### `assets/`

Contains static resources:

- Templates (document templates, configuration templates).
- Images (diagrams, examples).
- Data files (lookup tables, schemas).

## Progressive Disclosure

Agents load skills *progressively*, pulling in more detail only as a task calls for it. Skills should be structured to take advantage of this:

1. **Metadata** (~100 tokens): The `name` and `description` fields are loaded at startup for all skills.
2. **Instructions** (< 5000 tokens recommended): The full `SKILL.md` body is loaded when the skill is activated.
3. **Resources** (as needed): Files (e.g. those in `scripts/`, `references/`, or `assets/`) are loaded only when required.

Keep your main `SKILL.md` under 500 lines. Move detailed reference material to separate files.

## File References

When referencing other files in your skill, use relative paths from the skill root:

```markdown
See [the reference guide](references/REFERENCE.md) for details.

Run the extraction script:
scripts/extract.py
```

Keep file references one level deep from `SKILL.md`. Avoid deeply nested reference chains.

## Guidelines

- Keep skills **read-only** unless the user explicitly asks for changes.
- Prefer `gh`, `rg`, `jq`, and standard POSIX tools. Avoid adding new dependencies.
- Do not duplicate system-wide `AGENTS.md` instructions; focus on skill-specific behavior.
- Use concise language. This is reference material, not a user guide.

## Validation

Use the [skills-ref](https://github.com/agentskills/agentskills/tree/main/skills-ref) reference library to validate your skills:

```bash
npx skills-ref validate ./skill-name
```

This checks that your `SKILL.md` frontmatter is valid and follows all naming conventions.
