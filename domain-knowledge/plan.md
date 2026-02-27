# Remediation Plan — code-skill Security Findings

**Created:** February 26, 2026
**Source:** [security-review.md](security-review.md)
**Status:** Not Started

---

## Phase 1 — Critical & High Priority (Do First)

### Task 1.1 — Audit Published VSIX Against Repository

> **Fixes:** §2.1 CRITICAL — Feature/Code Discrepancy

**Goal:** Determine whether the published extension contains JavaScript code not present in this repo.

**Steps:**

1. Download the published VSIX from the VS Code Marketplace:
   ```powershell
   # Option A: From command line (requires vsce or direct URL)
   # The VSIX can be downloaded from:
   # https://marketplace.visualstudio.com/items?itemName=herbertagosto.skill
   # Click the "Download Extension" link on the version history tab
   ```

2. Rename the `.vsix` to `.zip` and extract:
   ```powershell
   Rename-Item herbertagosto.skill-*.vsix herbertagosto.skill.zip
   Expand-Archive herbertagosto.skill.zip -DestinationPath vsix-contents
   ```

3. Compare the extracted contents against this repo:
   ```powershell
   # List all files in the VSIX
   Get-ChildItem -Recurse vsix-contents | Select-Object FullName
   
   # Look specifically for JS/TS files
   Get-ChildItem -Recurse vsix-contents -Include *.js,*.ts,*.mjs,*.cjs
   ```

4. If JS files are found:
   - Audit each file for network calls (`fetch`, `http`, `https`, `request`, `XMLHttpRequest`, `WebSocket`).
   - Audit for file system operations (`fs.`, `readFile`, `writeFile`).
   - Audit for telemetry/analytics patterns.
   - Audit for `child_process`, `exec`, `spawn` calls.
   - Check for obfuscated or minified code that resists review.

5. Document findings and decide:
   - **If clean:** Note in security review that VSIX was audited and is consistent.
   - **If suspicious code found:** Do not install; fork from repo source only.

**Acceptance Criteria:** A written record of what the VSIX contains vs. this repo.

---

### Task 1.2 — Create `.vscodeignore` File

> **Fixes:** §2.2 HIGH — Missing `.vscodeignore` File

**Goal:** Prevent internal documents and non-essential files from being packaged in the VSIX.

**Steps:**

1. Create `.vscodeignore` in the repository root with the following content:

   ```
   # Development and internal files
   .vscode/
   .gitignore
   domain-knowledge/
   
   # Documentation not needed at runtime
   CHANGELOG.md
   SUPPORT.md
   
   # Source images used only in README (served from GitHub URLs)
   resources/images/
   ```

2. Verify the resulting VSIX only contains required files by running:
   ```powershell
   # Install vsce if not already available
   npm install -g @vscode/vsce
   
   # List what would be packaged
   vsce ls
   ```

3. Confirm the following are **included**:
   - `package.json`
   - `README.md`
   - `LICENSE.md`
   - `skill.configuration.json`
   - `syntaxes/skill.tmLanguage`
   - `snippets/skill.json`
   - `themes/*.json` and `themes/*.tmTheme`
   - `resources/icons/main.png`

4. Confirm the following are **excluded**:
   - `domain-knowledge/` (contains `background-information.md` with Intel references)
   - `.gitignore`
   - `resources/images/` (demo screenshots, not needed at runtime)

**File to create:**

```
FilePath: .vscodeignore
```

```
.vscode/
.gitignore
domain-knowledge/
SUPPORT.md
resources/images/
```

**Acceptance Criteria:** `vsce ls` output shows no files from `domain-knowledge/` or `resources/images/`.

---

## Phase 2 — Medium Priority (Do Next)

### Task 2.1 — Fix ReDoS-Prone Regex in Function Definition Pattern

> **Fixes:** §2.3.5 — ReDoS in `skill.tmLanguage`

**Goal:** Eliminate catastrophic backtracking in the function name capture group.

**File:** `syntaxes/skill.tmLanguage`, line 54

**Current (vulnerable):**
```regex
(\b(?i:(defun|defmacro|defstruct|defclass|defmethod|defconstant|defvar|defgeneric))\b)(\s+)((\w|\-|\!|\?)*)
```

**Fixed:**
```regex
(\b(?i:(defun|defmacro|defstruct|defclass|defmethod|defconstant|defvar|defgeneric))\b)(\s+)([\w\-!?]*)
```

**Change:** Replace `((\w|\-|\!|\?)*)` with `([\w\-!?]*)`.

**Why:** Alternation inside a repeated group `(a|b|c)*` causes the regex engine to try every permutation on failure. A character class `[abc]*` is matched in a single pass with no backtracking.

**Note:** The capture group number for the function name changes from group 4 to group 4 (still the same, since we removed the inner group 5 but the `captures` dict only references groups 2 and 4). Verify that group 4 still captures the function name correctly after the change. The inner group `(\w|\-|\!|\?)` was group 5 (uncaptured in the grammar), and removing it shifts nothing that the grammar references.

**Acceptance Criteria:** Extension still highlights function names after `defun`, `procedure`, etc. No catastrophic backtracking on adversarial input.

---

### Task 2.2 — Audit and Harden Remaining Regex Patterns

> **Fixes:** §2.3.1–2.3.4 — Other ReDoS concerns

**Goal:** Confirm all regex patterns are safe or fix as needed.

**Steps:**

1. Extract all regex patterns from `skill.tmLanguage` and `skill.configuration.json`.

2. Test each with a ReDoS analysis tool:
   ```powershell
   # Option A: Use npm package
   npm install -g recheck
   
   # Test individual patterns
   recheck "^((?!\/\/).)*(\\{[^}\"'``]*|\\([^)\"'``]*|\\[[^\\]\"'``]*)$"
   ```

3. For any pattern flagged as vulnerable, apply one of:
   - Convert alternation-in-repetition to character classes.
   - Add possessive quantifiers or atomic groups (if supported by the regex engine).
   - Simplify the pattern.

4. Patterns to test:
   | Source File | Pattern Description | Concern Level |
   |---|---|---|
   | `skill.configuration.json` | `wordPattern` | Low |
   | `skill.configuration.json` | `increaseIndentPattern` | Medium |
   | `skill.tmLanguage` | Keyword alternation (line 48) | Low |
   | `skill.tmLanguage` | Number matching | Low |
   | `skill.tmLanguage` | Function definition (line 54) | **Fixed in Task 2.1** |

**Acceptance Criteria:** All regex patterns pass ReDoS analysis without warnings.

---

### Task 2.3 — Replace External Image URLs in README with Relative Paths

> **Fixes:** §2.4 — External Resource Loading

**Goal:** Eliminate passive IP disclosure from external image loading when the README is viewed.

**File:** `README.md`

**Steps:**

1. Replace all absolute GitHub image URLs with relative paths:

   | Current | Replacement |
   |---|---|
   | `https://github.com/herbertagosto/code-skill/blob/main/resources/images/syntax-dark.png?raw=true` | `resources/images/syntax-dark.png` |
   | `https://github.com/herbertagosto/code-skill/blob/main/resources/images/goto-definition.gif?raw=true` | `resources/images/goto-definition.gif` |
   | `https://github.com/herbertagosto/code-skill/blob/main/resources/images/document-symbol.gif?raw=true` | `resources/images/document-symbol.gif` |
   | `https://github.com/herbertagosto/code-skill/blob/main/resources/images/code-completion.gif?raw=true` | `resources/images/code-completion.gif` |
   | `https://github.com/herbertagosto/code-skill/blob/main/resources/images/hover.gif?raw=true` | `resources/images/hover.gif` |
   | `https://github.com/herbertagosto/code-skill/blob/main/resources/images/document-comment.gif?raw=true` | `resources/images/document-comment.gif` |
   | `https://github.com/herbertagosto/code-skill/blob/main/resources/images/diagnostic.gif?raw=true` | `resources/images/diagnostic.gif` |
   | `https://github.com/herbertagosto/code-skill/blob/main/resources/images/syntax-light.png?raw=true` | `resources/images/syntax-light.png` |

   **Note:** Relative paths work on both GitHub and the VS Code Marketplace. The Marketplace renders relative paths using the VSIX contents, so if images are excluded via `.vscodeignore`, they won't display on the Marketplace page. Decide whether to:
   - **(A)** Keep images in the VSIX (remove `resources/images/` from `.vscodeignore`) — larger package, but images show on Marketplace.
   - **(B)** Keep external URLs only for the Marketplace-facing README — accept the IP disclosure tradeoff.
   - **(C)** Use relative paths AND include images — recommended balance.

   **Recommended:** Option (C). Use relative paths and remove the `resources/images/` line from `.vscodeignore`.

2. Remove the Buy Me a Coffee external image and link, or replace with a text-only link:

   **Current:**
   ```html
   <a href="https://www.buymeacoffee.com/hagosto" target="_blank"><img src="https://cdn.buymeacoffee.com/buttons/v2/default-blue.png" ...></a>
   ```

   **Replacement (text-only):**
   ```markdown
   [Support this project on Buy Me a Coffee](https://www.buymeacoffee.com/hagosto)
   ```

3. Fix the malformed email URL:

   **Current:** `[Reach me](http://herbertagosto@gmail.com)`
   **Fixed:** `[Reach me](mailto:herbertagosto@gmail.com)`

**Acceptance Criteria:** No external `<img src=...>` tags remain. All images use relative paths. Email uses `mailto:` scheme.

---

## Phase 3 — Low Priority (Do When Convenient)

### Task 3.1 — Remove or Customize the Author Snippet

> **Fixes:** §2.5.1 — Hardcoded Author Name

**File:** `snippets/skill.json`

**Option A — Remove the snippet entirely:**

Delete the `"Author"` entry from `snippets/skill.json`.

**Option B — Make it use a VS Code variable (preferred):**

Replace:
```json
"Author": {
    "prefix":"author",
    "body":["Herbert Agosto"],
    "description": "Author"
}
```

With:
```json
"Author": {
    "prefix":"author",
    "body":["${1:Your Name}"],
    "description": "Insert author name"
}
```

This turns it into a placeholder the user fills in, rather than injecting a fixed name.

**Acceptance Criteria:** Triggering the `author` snippet does not output a hardcoded name.

---

### Task 3.2 — Reconcile Version Numbers

> **Fixes:** §2.6 — Version Inconsistency

**Steps:**

1. Determine the correct current version. Based on `CHANGELOG.md`, the latest release is `0.10.0`.

2. Update `package.json`:
   ```json
   "version": "0.10.0"
   ```

3. Update `domain-knowledge/background-information.md` version reference from `0.5.0` to `0.10.0`.

**Acceptance Criteria:** All files reference the same version.

---

### Task 3.3 — Document Bundled Theme Provenance

> **Fixes:** §2.7 — Bundled VS Code Themes

**Steps:**

1. Determine which VS Code version the theme files were sourced from. Check git blame or the VS Code GitHub repo at:
   - https://github.com/microsoft/vscode/tree/main/extensions/theme-defaults/themes

2. Add a comment header or a separate `themes/README.md` noting:
   ```markdown
   # Bundled Theme Sources
   
   The following files are copies of VS Code's default themes, used as base
   themes for the custom Skill themes via the `include` mechanism.
   
   | File | Source | VS Code Version |
   |---|---|---|
   | dark_modern.json | microsoft/vscode theme-defaults | vX.Y.Z |
   | dark_plus.json | microsoft/vscode theme-defaults | vX.Y.Z |
   | dark_vs.json | microsoft/vscode theme-defaults | vX.Y.Z |
   
   Last updated: YYYY-MM-DD
   ```

**Acceptance Criteria:** Provenance is documented; update schedule is noted.

---

## Phase 4 — Ongoing / Process

### Task 4.1 — Establish VSIX Verification Process

For any future installs or updates of this extension:

1. Download the VSIX.
2. Extract and diff against the known repo state.
3. Audit any new JavaScript files.
4. Only deploy after verification.

### Task 4.2 — Monitor Upstream Repository

- Watch the [herbertagosto/code-skill](https://github.com/herbertagosto/code-skill) repository for new releases.
- Re-run security review on major version bumps.
- Check for new `main`/`browser` entry points added to `package.json`.

### Task 4.3 — Periodic ReDoS Re-Testing

- Re-test regex patterns after any grammar updates.
- Include ReDoS testing in any CI pipeline if the extension is forked.

---

## Implementation Summary

| Task | Phase | Effort | Files Modified |
|---|---|---|---|
| 1.1 Audit VSIX | 1 | 1-2 hours | None (investigation) |
| 1.2 Create `.vscodeignore` | 1 | 15 min | New: `.vscodeignore` |
| 2.1 Fix ReDoS regex | 2 | 15 min | `syntaxes/skill.tmLanguage` |
| 2.2 Audit all regex | 2 | 1 hour | `syntaxes/skill.tmLanguage`, `skill.configuration.json` |
| 2.3 Fix README URLs | 2 | 30 min | `README.md` |
| 3.1 Fix author snippet | 3 | 5 min | `snippets/skill.json` |
| 3.2 Fix version numbers | 3 | 10 min | `package.json`, `domain-knowledge/background-information.md` |
| 3.3 Document themes | 3 | 30 min | New: `themes/README.md` |
| 4.1-4.3 Ongoing process | 4 | Recurring | N/A |

**Total estimated one-time effort:** ~4 hours
