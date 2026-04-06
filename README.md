# ---

> OpenClaw AI Agent Skill

---
name: file-editor
description: Safe, reliable file editing with validation and debugging. Use when editing code, configs, or markup (HTML, JSON, Python, JS). Prevents common mistakes like wrong parameter names, exact-text mismatches, and unintended changes. Includes validation, search verification, and rollback strategies.
---

# File Editor Skill

## Quick Checklist Before Every Edit

- [ ] **Read first** — always `read` the file before editing
- [ ] **Find the exact text** — search with `grep` for the literal string you want to match
- [ ] **Check for minification** — if the file is minified (no newlines), reformat it first
- [ ] **Verify context** — show 2-3 lines above/below to confirm you're in the right place
- [ ] **Use correct parameter** — `new_string` not `newText`, `old_string` not `oldText`
- [ ] **Test after edit** — validate syntax or run a quick check
- [ ] **Rollback ready** — know what to do if it breaks

## Common Edit Failures & Fixes

### 1. **"Could not find the exact text"**

**Cause:** Whitespace mismatch, different line endings, or minified code

**Fix:**
```bash
# Search for what you're trying to replace
grep -n "your_search_string" /path/to/file

# If no match, search smaller pieces
grep -n "const AGENTS" /path/to/file

# Check for hidden characters
cat -A /path/to/file | grep "your_string"

# If minified, reformat first
node -e "console.log(require('fs').readFileSync('/path/to/file.json', 'utf8'))" | jq . > /tmp/formatted.json
```

### 2. **Wrong Parameter Name**

**Wrong:**
- `newText` ❌
- `new_string` (when tool expects `newText`) ❌
- `oldText` (when tool expects `old_string`) ❌

**Right:**
- Use tool's spec: check `edit` parameters
- Current correct params: `new_string`, `old_string` (both accepted)

### 3. **Minified or Single-Line Files**

**Problem:** Can't match multiline strings in minified code

**Solution:**
```bash
# Format the file first
cd ~/council-room/frontend
node -e "
const fs = require('fs');
const code = fs.readFileSync('index.html', 'utf8');
// For HTML: just prettify with better formatting
const formatted = code.replace(/></g, '>\n<');
fs.writeFileSync('index.html', formatted);
"

# Now you can edit more easily
```

### 4. **Can't Find Exact Match in Minified HTML**

**Problem:** Frontend bundles are minified; `edit` can't find the string

**Strategy:**
- Use `grep -o` to find the exact substring
- Extract the context around it
- Match a smaller, unique segment that includes whitespace exactly
- If that fails, use `write` to replace the whole file after reading it

## Safe Editing Workflow

### Step 1: Read & Search

```bash
# Read the file
read /path/to/file

# Search for what you need to change
grep -n "SEARCH_STRING" /path/to/file | head -5

# Get context
grep -B 2 -A 2 "SEARCH_STRING" /path/to/file
```

### Step 2: Copy Exact Text

When using `edit`, copy the EXACT text including whitespace:

```bash
# Get exact line(s) to match
sed -n '100,110p' /path/to/file

# Copy this output directly into old_string parameter
```

### Step 3: Edit (Small Changes)

Use `edit` for surgical, single-location changes:

```
old_string: [EXACT TEXT FROM sed/grep]
new_string: [YOUR REPLACEMENT]
```

### Step 4: Edit (Multiple Locations or Complex Changes)

If `edit` fails, use `read` + `write`:

```bash
# Read the file
read /path/to/file

# In your mind/notes, make the change

# Write the modified version back
write /path/to/file [ENTIRE MODIFIED CONTENT]
```

### Step 5: Validate

```bash
# Syntax check
node -c /path/to/file.js        # For JS/Node
python3 -m py_compile /path/to/file.py  # For Python
jq . /path/to/file.json         # For JSON

# Run a quick test
curl -s http://127.0.0.1:3000/health
```

## When to Use What

| Task | Tool | Why |
|------|------|-----|
| Change one string in middle of file | `edit` | Surgical, safe |
| Change multiple locations | `read` + `write` | Easier than multiple edits |
| Minified file & need multi-line match | `read` + `write` | `edit` can't handle minified |
| Append/append to file | `edit` or `write` | Depends on context |
| Create new file | `write` | Direct creation |
| Update config file | `edit` | Usually single-location |

## Debugging Edit Failures

### Edit Failed: Parameter Name Wrong?

```bash
# Check tool spec
# edit tool accepts: file_path, old_string, new_string
# NOT: oldText, newText
```

### Edit Failed: Whitespace?

```bash
# Get exact bytes
od -c /path/to/file | grep "SEARCH" | head -1

# Look for hidden tabs, extra spaces
cat -A /path/to/file | grep "SEARCH"
```

### Edit Failed: Found Multiple Matches?

```bash
# Count occurrences
grep -c "exact_string" /path/to/file

# If > 1, make your old_string more unique by adding context
grep -B 1 -A 1 "exact_string" /path/to/file | head -5
```

### Frontend Edits Failing?

For minified HTML/JS files:

1. Check if it's actually minified:
   ```bash
   grep -o ".\{100,\}" /path/to/file.html | head -1
   # If no newlines in output: it's minified
   ```

2. Try to find a unique sub-string:
   ```bash
   grep -n "const AGENTS" /path/to/file.html
   ```

3. If still failing, use `read` + `write` to replace the entire file:
   ```bash
   read /path/to/file.html    # Get full content
   # Modify in-memory
   write /path/to/file.html   # Write back
   ```

## Post-Edit Checklist

- [ ] Syntax valid? (node -c, python3 -m py_compile, etc.)
- [ ] Service restarted if needed? (pkill, npm start, etc.)
- [ ] URLs/endpoints still working? (curl, browser test)
- [ ] No unintended changes? (grep for edge cases)

## When Things Break

**"Can't find exact text"**
→ Use `grep` to verify the string exists
→ Copy-paste the exact output from grep into old_string
→ If that fails, switch to `read` + `write`

**"Edit worked but service broke"**
→ Check syntax: `node -c file.js`
→ Check logs: `tail -50 logfile.log`
→ Rollback: Use version control or revert the change

**"Minified file won't edit"**
→ Read the whole file
→ Modify in-memory (you have the full content)
→ Write it back as a single write operation

## Key Insight

**Never trust `edit` on minified code.** If a file has no newlines or very long lines (>500 chars), use `read` + `write` instead. It's faster and more reliable.

## Installation

```bash
cp -r file-editor/ ~/.openclaw/workspace/skills/file-editor/
```

## License

MIT © [Sentra Technology](https://github.com/Icattj)
