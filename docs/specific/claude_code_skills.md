# Installing custom skills in Claude Code on WSL

**Claude Code has no built-in `install-from-zip` command.** Installing a custom skill from an archive like `pbi-squire.zip` is a manual process: you extract the archive into the correct directory, ensure it contains a properly formatted `SKILL.md` file, and restart Claude Code. The official documentation defines two installation scopes — personal (global across all projects) and project-level — each with a distinct filesystem path. A separate plugin system exists for more sophisticated distribution, but for a single skill archive, the standalone directory approach is the simplest and officially documented method.

---

## How skills work under the hood

Every Claude Code skill is a **directory containing a `SKILL.md` file**. The directory name becomes the slash-command name (e.g., a folder called `pbi-squire` creates `/pbi-squire`). The `SKILL.md` file has two parts: YAML frontmatter between `---` markers that controls metadata and invocation behavior, and Markdown body content with the instructions Claude follows.

Claude Code discovers skills at startup from specific filesystem paths. It loads only the `name` and `description` from frontmatter into its system prompt initially. The full body of `SKILL.md` loads only when the skill is triggered — either by the user typing `/skill-name` or by Claude auto-selecting it based on the description matching the current task context.

---

## Step-by-step: installing pbi-squire.zip globally on WSL

This installs the skill so it's available across **all projects** on your machine. On WSL/Ubuntu, the personal skills path is **`~/.claude/skills/`**.

**Step 1 — Create the skills directory if it doesn't exist:**

```bash
mkdir -p ~/.claude/skills/
```

**Step 2 — Extract the archive into the skills directory:**

```bash
unzip pbi-squire.zip -d ~/.claude/skills/
```

After extraction, verify the resulting structure. You need a directory containing a `SKILL.md` file. The expected result is one of these two patterns:

```
# Pattern A: zip extracts directly as a named folder (ideal)
~/.claude/skills/pbi-squire/
├── SKILL.md          # Required
├── reference.md      # Optional supporting docs
└── scripts/          # Optional bundled scripts
    └── analyze.py

# Pattern B: zip extracts files flat (needs wrapping)
~/.claude/skills/
├── SKILL.md          # ← Wrong: sitting at root, not in a subdirectory
└── scripts/
```

**Step 3 — Fix the directory structure if needed.** If the zip extracts files directly without a parent folder, wrap them:

```bash
mkdir -p ~/.claude/skills/pbi-squire
# Move extracted files into the new directory
mv ~/.claude/skills/SKILL.md ~/.claude/skills/pbi-squire/
mv ~/.claude/skills/scripts ~/.claude/skills/pbi-squire/  # if present
# Repeat for any other extracted files
```

If the zip creates a differently named parent folder, simply rename it:

```bash
mv ~/.claude/skills/some-other-name ~/.claude/skills/pbi-squire
```

**Step 4 — Verify the SKILL.md file has valid frontmatter:**

```bash
head -20 ~/.claude/skills/pbi-squire/SKILL.md
```

You should see something like:

```yaml
---
name: pbi-squire
description: Analyzes Power BI files and provides insights on data models, DAX measures, and report structure.
---
```

If the file lacks frontmatter, Claude Code will still load it but will derive the name from the directory name and won't have a description for auto-invocation.

**Step 5 — Set correct permissions:**

```bash
chmod -R 755 ~/.claude/skills/pbi-squire/
```

**Step 6 — Restart Claude Code.** Exit any running session and start a new one. Skills load at session startup.

```bash
# Exit current session (Ctrl+C or /exit), then:
claude
```

**Step 7 — Verify the skill is loaded.** Inside Claude Code, type `/` and look for `pbi-squire` in the autocomplete menu, or type `/help` to see all available commands including skills.

---

## Global vs. project-level installation

The documentation defines a clear two-tier system with distinct paths and behaviors:

| Aspect | Personal / global | Project-level |
|---|---|---|
| **Path** | `~/.claude/skills/<name>/SKILL.md` | `.claude/skills/<name>/SKILL.md` (relative to project root) |
| **Scope** | Available in every project you open | Only available in that specific project |
| **Sharing** | Private to your user account | Shared with team via git |
| **Version control** | Typically not in git | Committed to project repo |
| **Priority** | Higher — overrides project skills with same name | Lower — overridden by personal skills |

**To install per-project instead of globally**, target the project's `.claude/skills/` directory:

```bash
cd /path/to/your/project
mkdir -p .claude/skills/
unzip /path/to/pbi-squire.zip -d .claude/skills/
# Same structure verification as above
git add .claude/skills/pbi-squire/
git commit -m "Add pbi-squire Power BI analysis skill"
```

The full **priority hierarchy** when skills share the same name is: **enterprise managed > personal (`~/.claude/skills/`) > project (`.claude/skills/`)**. Plugin-based skills use namespacing (`plugin-name:skill-name`) and therefore never conflict with standalone skills.

---

## Expected file structure inside a skill archive

The official documentation specifies that a skill is a directory with one required file and several optional components:

```
pbi-squire/                    # Directory name = slash command name
├── SKILL.md                   # REQUIRED — frontmatter + instructions
├── reference.md               # Optional — detailed reference docs
├── examples.md                # Optional — usage examples
├── template.md                # Optional — templates Claude fills in
├── scripts/                   # Optional — executable helper scripts
│   ├── analyze_pbix.py
│   └── validate.sh
├── references/                # Optional — additional docs loaded on demand
│   └── dax-patterns.md
├── assets/                    # Optional — templates, icons, fonts
└── templates/
    └── report-template.txt
```

The `SKILL.md` YAML frontmatter supports these fields:

- **`name`** — Lowercase letters, numbers, hyphens only; max 64 chars; becomes the `/slash-command`; defaults to directory name if omitted
- **`description`** — What the skill does and when to use it; max 1024 chars; critical for Claude's auto-discovery
- **`disable-model-invocation`** — Boolean; when `true`, only the user can trigger it (Claude won't auto-invoke)
- **`user-invocable`** — Boolean; when `false`, only Claude can invoke it (hidden from `/` menu)
- **`allowed-tools`** — Comma-separated tools Claude can use without permission prompts (e.g., `Read, Grep, Glob` or `Bash(python *)`)
- **`context`** — Set to `fork` to run in an isolated subagent context
- **`model`** — Force a specific model (e.g., `claude-3-5-haiku-20241022`)
- **`argument-hint`** — Autocomplete hint shown in slash menu (e.g., `[path-to-pbix]`)
- **`hooks`** — Lifecycle hooks scoped to the skill (`PreToolUse`, `PostToolUse`, `Stop`)

The Markdown body supports **`$ARGUMENTS`** (replaced with text after the slash command), **positional arguments** (`$0`, `$1`, `$2`), **`@<file path>`** for embedding file contents, and **`` !`command` ``** for running shell commands before sending content to Claude.

The documentation recommends keeping `SKILL.md` under **500 lines** and using progressive disclosure: put detailed reference material in separate files that Claude reads only when needed.

---

## CLI commands for skill and plugin management

Claude Code provides no dedicated `skill install` CLI command. Skill management is done through the filesystem for standalone skills, or through the plugin system for distributable skills.

**Standalone skill operations** (filesystem-based):

```bash
# Create personal skill
mkdir -p ~/.claude/skills/my-skill && nano ~/.claude/skills/my-skill/SKILL.md

# Create project skill
mkdir -p .claude/skills/my-skill && nano .claude/skills/my-skill/SKILL.md

# Remove a skill
rm -rf ~/.claude/skills/my-skill        # personal
rm -rf .claude/skills/my-skill           # project

# Load skills from an external directory (session flag)
claude --add-dir /path/to/extra-skills

# Debug skill loading
claude --debug
```

**In-session commands:**

```
/skill-name                  # Invoke a skill
/skill-name arguments        # Invoke with arguments
/help                        # List all available commands including skills
/context                     # Check for excluded-skills warnings
/add-dir /path               # Add directory with skills mid-session
/status                      # Show active settings and configuration layers
```

**Plugin system commands** (for distributable skills packaged as plugins):

```bash
# Terminal CLI
claude plugin install name@marketplace       # Install a plugin
claude plugin uninstall name@marketplace      # Remove a plugin
claude plugin enable name@marketplace         # Enable disabled plugin
claude plugin disable name@marketplace        # Disable without removing
claude plugin update name@marketplace         # Update a plugin
claude plugin validate .                      # Validate plugin/marketplace structure
claude --plugin-dir ./local-plugin            # Load plugin from local dir (dev/test)

# In-session
/plugin                                       # Interactive plugin management UI
/plugin marketplace add owner/repo            # Add a GitHub marketplace
/plugin marketplace add ./local-marketplace   # Add a local marketplace
/plugin marketplace list                      # List configured marketplaces
/plugin install name@marketplace              # Install from within session
```

Installed plugins are cached at **`~/.claude/plugins/cache/`** with metadata in **`~/.claude/plugins/installed_plugins.json`**.

---

## There is no official zip-import feature

The official documentation does not describe a built-in command to install a skill directly from a `.zip` file. The documented `.skill` file format (used in Claude.ai's web interface) is a zip archive with a `.skill` extension — you can upload these through **Settings > Features > Skills** on claude.ai. But in Claude Code (the terminal tool), installation is always filesystem-based: extract the archive, place it in the right directory, and restart.

For more sophisticated distribution (team sharing, versioning, automatic updates), the documentation recommends the **plugin system**: wrap your skill in a plugin directory structure with a `.claude-plugin/plugin.json` manifest, optionally publish it to a marketplace, and have users install it via `/plugin install`. This is the closest equivalent to a managed "install from package" workflow, but it operates on git repositories and local directories — not zip files directly.

The practical workflow for `pbi-squire.zip` on WSL is straightforward: `unzip pbi-squire.zip -d ~/.claude/skills/`, verify the structure, restart Claude Code, and invoke with `/pbi-squire`.