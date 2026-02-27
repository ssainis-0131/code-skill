# Security Review — code-skill (Cadence SKILL VS Code Extension)

**Review Date:** February 26, 2026
**Extension Version:** 0.5.0 (package.json) / Changelog references up to 0.10.0
**Publisher:** herbertagosto
**Repository:** https://github.com/herbertagosto/code-skill.git
**License:** BSD 2-Clause

---

## 1. Executive Summary

This extension provides language support (syntax highlighting, snippets, themes, and language configuration) for the Cadence SKILL scripting language in VS Code. The repository contains **no executable code** (no `main` or `browser` entry point in `package.json`), meaning all contributions are declarative (TextMate grammars, JSON configurations, snippets, themes). This significantly limits the active threat surface compared to extensions that run JavaScript/TypeScript code.

However, several areas of concern were identified across supply chain integrity, regex-based denial-of-service, information disclosure, external resource trust, and a critical discrepancy between advertised features and shipped code.

**Overall Risk Rating: LOW-MODERATE**

---

## 2. Threat Surface Analysis

### 2.1 CRITICAL — Feature/Code Discrepancy (Missing Executable Code)

| Property | Detail |
|---|---|
| **Severity** | CRITICAL |
| **Category** | Supply Chain Integrity / Trust |
| **Affected Files** | `package.json`, `README.md`, `CHANGELOG.md` |

**Finding:**
The `README.md` and `CHANGELOG.md` advertise the following features that **require an executable entry point** (`main` field in `package.json`):

- Go to Definition (`F12`)
- Document Symbol
- Code Completion (IntelliSense)
- Hover
- Documentation Comment generation
- Diagnostics
- Configuration settings (`skill.completion.*`, `skill.diagnostic.*`)

However, `package.json` has **no `main` or `browser` field**, and **no JavaScript or TypeScript source files exist** in the repository. The `contributes` key in `package.json` only declares languages, grammars, snippets, and themes — none of the programmatic features.

**Security Implications:**

1. **The published VSIX on the VS Code Marketplace may contain code not present in this repository.** If the publisher compiles and packages code that is not committed to the public repo, users cannot audit what actually runs on their machine.
2. **Alternatively**, these features may not work at all from this repo, meaning the README is misleading, which is a trust concern.
3. A consumer building this extension from source will get a **different extension** than what is on the Marketplace.

**Recommendation:**
- Obtain and audit the actual `.vsix` package from the Marketplace.
- Compare its contents against this repository to identify any bundled JavaScript/TypeScript that is not source-available.
- Do not trust this repository as a complete representation of the published extension without verification.

---

### 2.2 HIGH — Missing `.vscodeignore` File

| Property | Detail |
|---|---|
| **Severity** | HIGH |
| **Category** | Information Disclosure / Packaging |
| **Affected Files** | Root directory (missing `.vscodeignore`) |

**Finding:**
There is no `.vscodeignore` file in the repository. When the extension is packaged with `vsce package`, **all files not covered by `.gitignore` are included in the VSIX**. The `.gitignore` only excludes `.vscode/`.

This means the following files would be included in a published package:

- `domain-knowledge/background-information.md` — Contains detailed internal analysis, including references to Intel internal usage, licensing analysis, and full specification details.
- `SUPPORT.md`, `CHANGELOG.md`, `LICENSE.md` — Generally fine, but should be intentional.
- `resources/images/` — All GIF/PNG demo assets (~potentially large).
- Any future files added to the root.

**Security Implications:**
- Internal organizational documents (like `background-information.md`) could be unintentionally shipped.
- The `background-information.md` file explicitly references "internal usage at Intel" — publishing this in a VSIX would leak internal organizational context.

**Recommendation:**
- Create a `.vscodeignore` file that excludes at minimum: `domain-knowledge/`, `.gitignore`, and any files not required at runtime.
- Audit what is currently in the published VSIX on the Marketplace.

---

### 2.3 MEDIUM — Regular Expression Denial of Service (ReDoS) in Grammar & Configuration

| Property | Detail |
|---|---|
| **Severity** | MEDIUM |
| **Category** | Availability / Denial of Service |
| **Affected Files** | `syntaxes/skill.tmLanguage`, `skill.configuration.json` |

**Finding:**
Several regex patterns used for tokenization and editor behavior contain constructs that could lead to excessive backtracking on crafted input:

#### 2.3.1 — `wordPattern` in `skill.configuration.json`

```regex
(-?\d*\.\d\w*)|([^\`\~\!\@\#\%\^\&\*\(\)\-\=\+\[\{\]\}\\\|\;\:\'\"\,\.\<\>\/\?\s]+)
```

The second alternative uses a **negated character class with `+` quantifier**. While generally safe, the breadth of the character class means very long tokens made of non-excluded characters could cause slow matching depending on VS Code's regex engine behavior. This regex is evaluated on **every cursor movement and word selection**.

#### 2.3.2 — `increaseIndentPattern` in `skill.configuration.json`

```regex
^((?!\/\/).)*(\\{[^}"'`]*|\\([^)"'`]*|\\[[^\\]"'`]*)$
```

This pattern uses a **negative lookahead `(?!\/\/)` inside a `.*` quantifier**, which creates nested quantifier-like behavior. On lines with many characters that *almost* match the lookahead, this can cause significant backtracking.

#### 2.3.3 — Monolithic keyword regex in `skill.tmLanguage` (line 48)

The keyword pattern is a single `\b(...|...|...)\b` with **hundreds of alternatives**. While alternation within word boundaries is generally safe from catastrophic backtracking, this regex is:
- Extremely long (thousands of characters)
- Evaluated against every word-boundary in every opened SKILL file
- Could cause noticeable latency on very large files

#### 2.3.4 — Number matching regex in `skill.tmLanguage`

```regex
\b((([0-9]+\.?[0-9]*)|(\.[0-9]+))((e|E)(\+|-)?[0-9]+)?)\b
```

Contains nested groups with alternation and optional quantifiers. While bounded by `\b`, adversarial content like long sequences of digits could cause slowdowns.

#### 2.3.5 — Function definition regex in `skill.tmLanguage`

```regex
(\b(?i:(defun|defmacro|defstruct|defclass|defmethod|defconstant|defvar|defgeneric))\b)(\s+)((\w|\-|\!|\?)*) 
```

The final group `(\w|\-|\!|\?)*` uses alternation inside a `*` quantifier. This is a classic ReDoS pattern — if a string contains a long sequence of characters that individually match some alternatives but the overall match fails, the engine backtracks exponentially.

**Recommendation:**
- Replace `(\w|\-|\!|\?)*` with a character class: `[\w\-!?]*` — semantically identical but immune to backtracking.
- Consider simplifying the `increaseIndentPattern` or adding guards.
- Test all regex patterns against a ReDoS analysis tool (e.g., `recheck`, `safe-regex`).

---

### 2.4 MEDIUM — External Resource Loading and Third-Party Trust

| Property | Detail |
|---|---|
| **Severity** | MEDIUM |
| **Category** | Data Exfiltration / Trust Boundary |
| **Affected Files** | `README.md` |

**Finding:**
The `README.md` references multiple external URLs that are loaded when the file is rendered (in VS Code, on GitHub, or on the VS Code Marketplace):

#### GitHub-hosted images (8 instances):
```
https://github.com/herbertagosto/code-skill/blob/main/resources/images/*.png?raw=true
https://github.com/herbertagosto/code-skill/blob/main/resources/images/*.gif?raw=true
```

#### Third-party CDN (Buy Me a Coffee):
```html
<a href="https://www.buymeacoffee.com/hagosto" target="_blank">
<img src="https://cdn.buymeacoffee.com/buttons/v2/default-blue.png" ...>
```

**Security Implications:**

1. **IP Address Disclosure:** Loading images from `github.com` and `cdn.buymeacoffee.com` sends the viewer's IP address to these servers. When the README is viewed within VS Code or on the Marketplace, this happens automatically.

2. **Account Compromise Risk:** If the `herbertagosto` GitHub account is compromised, the image URLs could be redirected to serve:
   - Tracking pixels with unique identifiers
   - Exploit payloads targeting image parser vulnerabilities
   - Misleading content (social engineering)

3. **Third-Party Tracking:** The Buy Me a Coffee link and CDN image enable cross-site tracking of users who view or click the README.

4. **Contact Email Exposure:** The README contains `http://herbertagosto@gmail.com` — this is formatted as a URL (using `http://` scheme) rather than `mailto:`, which is unusual and could trigger unexpected browser behavior.

**Recommendation:**
- For internal/enterprise use, consider forking the repo and replacing external image references with locally bundled copies.
- Block or review outbound connections to `cdn.buymeacoffee.com` if the README is rendered in internal tools.
- Note the malformed email URL (`http://` instead of `mailto:`).

---

### 2.5 LOW — Information Disclosure via Snippets

| Property | Detail |
|---|---|
| **Severity** | LOW |
| **Category** | Information Disclosure |
| **Affected Files** | `snippets/skill.json` |

**Finding:**

#### 2.5.1 — Author Name Hardcoded
The "Author" snippet outputs the hardcoded string `"Herbert Agosto"`:

```json
"Author": {
    "prefix":"author",
    "body":["Herbert Agosto"],
    "description": "Author"
}
```

If a user triggers this snippet inadvertently, the extension author's name is inserted into their source file. This is non-harmful but could:
- Cause confusion in code attribution.
- Be committed to internal repos, creating misleading authorship records.

#### 2.5.2 — Header Snippet Exposes Metadata
The "Header" snippet uses VS Code variables:

```json
"body":[
    "/*\t$TM_FILENAME",
    "",
    "\t${2:author}",
    "\t${3:company}",
    ...
]
```

The `$TM_FILENAME` variable automatically inserts the current filename. The `author` and `company` placeholders prompt the user for input — if filled with real values, this sensitive metadata is embedded in source files.

**Recommendation:**
- For internal use, consider removing or customizing the "Author" snippet.
- Train users to be aware that the "header" snippet prompts for company name, which will be written to file.

---

### 2.6 LOW — Version Inconsistency

| Property | Detail |
|---|---|
| **Severity** | LOW |
| **Category** | Supply Chain Integrity |
| **Affected Files** | `package.json`, `CHANGELOG.md`, `domain-knowledge/background-information.md` |

**Finding:**
Version numbers are inconsistent across the repository:

- `package.json` declares version `0.5.0`
- `CHANGELOG.md` documents versions up to `0.10.0`
- `domain-knowledge/background-information.md` references version `0.5.0`

This mismatch means:
- The `package.json` in this repo does **not** correspond to the latest published version.
- The repository may not reflect the actual state of the published extension.
- Building from source would produce version `0.5.0`, while the Marketplace may have `0.10.0`.

**Recommendation:**
- Always verify the VSIX from the Marketplace against a known-good state.
- If forking, update `package.json` to the correct version.

---

### 2.7 LOW — Bundled Copies of VS Code Default Themes

| Property | Detail |
|---|---|
| **Severity** | LOW |
| **Category** | Supply Chain / Maintenance |
| **Affected Files** | `themes/dark_modern.json`, `themes/dark_plus.json`, `themes/dark_vs.json` |

**Finding:**
The extension bundles what appear to be copies of VS Code's built-in dark themes (`Dark Modern`, `Dark+`, `Dark Visual Studio`). The custom `skill_dark_modern.json` theme uses `"include": "./dark_modern.json"` to inherit from these copied files.

**Security Implications:**
- These copies will **not receive updates** when VS Code updates its built-in themes.
- If VS Code patches a theme-related security issue, this extension would still ship the old version.
- The provenance of these copies is not documented — there is no indication of which VS Code version they were extracted from.

**Recommendation:**
- Document the VS Code version these themes were sourced from.
- Periodically check for updates to the upstream theme files.

---

### 2.8 INFO — Broad Folding Markers in TextMate Grammar

| Property | Detail |
|---|---|
| **Severity** | INFO |
| **Category** | Availability |
| **Affected Files** | `syntaxes/skill.tmLanguage` |

**Finding:**
The `foldingStartMarker` is `\(` and `foldingStopMarker` is `\)`. In SKILL, which is a Lisp-like language, every expression uses parentheses. This means:
- Every `(` in a file triggers folding computation.
- On large SKILL files (thousands of lines), this could degrade editor performance.

**Recommendation:**
- This is expected for a Lisp-like language but should be noted for performance testing with large files.

---

## 3. Data Exfiltration Risk Assessment

### 3.1 No Outbound Network Calls in Extension Code

Since the extension has **no executable code** in this repository (no `main` entry point, no JS/TS files), there are no programmatic network calls. This means:

- **No telemetry** is collected by the extension code in this repo.
- **No data is sent** to external servers by the extension code in this repo.
- **No authentication tokens** are requested or stored.

**CAVEAT:** This assessment applies only to the code in this repository. If the published VSIX contains additional JavaScript code not present here (see Section 2.1), that code could perform arbitrary network operations.

### 3.2 Passive Data Exfiltration Vectors

| Vector | Risk | Detail |
|---|---|---|
| README image loading | LOW | Loads images from GitHub and buymeacoffee CDN — leaks viewer IP |
| Snippet metadata | LOW | Header snippet prompts for company/author names that are written to files |
| File extension association | NONE | `.il`, `.ils`, `.ocn` associations are standard for SKILL |
| Theme/grammar processing | NONE | Declarative; no data leaves the machine |

### 3.3 Indirect Exfiltration via Marketplace

When installed from the VS Code Marketplace:
- VS Code may send extension usage telemetry to Microsoft (controlled by VS Code settings, not this extension).
- The Marketplace page will log visitor IP addresses.
- Extension update checks are made periodically by VS Code.

These are standard VS Code behaviors, not specific to this extension.

---

## 4. Logging Risk Assessment

### 4.1 No Extension-Level Logging

The extension contains no executable code in this repository and therefore:
- **Produces no log files.**
- **Writes no data to disk** beyond what VS Code does natively.
- **Has no logging configuration.**

### 4.2 VS Code Platform Logging

VS Code itself may log the following information related to this extension:
- Extension activation events.
- Errors in grammar/snippet parsing.
- File paths of opened `.il`, `.ils`, `.ocn` files (visible in VS Code's recent files, window titles).
- Diagnostic messages (if the programmatic features exist in the published VSIX).

### 4.3 Potential Logging Risks if Hidden Code Exists

If the published VSIX contains JavaScript code not present in this repo (see Section 2.1), that code could potentially:
- Log file contents processed for code completion, hover, diagnostics.
- Log file paths, symbol names, and function signatures.
- Write logs to VS Code's output channel or extension storage.

**This cannot be assessed without examining the actual published VSIX.**

---

## 5. Recommendations Summary

| # | Priority | Action |
|---|---|---|
| 1 | **CRITICAL** | Obtain and decompile the published VSIX to verify it matches this repository. Audit any JavaScript code found in the VSIX that is not in this repo. |
| 2 | **HIGH** | Create a `.vscodeignore` file to prevent `domain-knowledge/`, `.gitignore`, and non-essential files from shipping in the VSIX. |
| 3 | **MEDIUM** | Fix the ReDoS-prone regex `(\w\|\-\|\!\|\?)*` in `skill.tmLanguage` — replace with character class `[\w\-!?]*`. |
| 4 | **MEDIUM** | Test all regex patterns in `skill.tmLanguage` and `skill.configuration.json` with a ReDoS analysis tool. |
| 5 | **MEDIUM** | For internal/enterprise deployment, fork the repo and replace external image URLs with bundled assets. |
| 6 | **LOW** | Remove or customize the hardcoded "Author" snippet for internal use. |
| 7 | **LOW** | Reconcile the version number in `package.json` (0.5.0) with the CHANGELOG (0.10.0). |
| 8 | **LOW** | Document the provenance of bundled VS Code theme files (`dark_modern.json`, `dark_plus.json`, `dark_vs.json`). |
| 9 | **LOW** | Fix the malformed contact URL in `README.md` (uses `http://` instead of `mailto:`). |

---

## 6. Files Reviewed

| File | Type | Risk Findings |
|---|---|---|
| `package.json` | Extension manifest | Missing `main` entry; version mismatch |
| `skill.configuration.json` | Language config | ReDoS-prone regex patterns |
| `syntaxes/skill.tmLanguage` | TextMate grammar | ReDoS-prone regex; broad folding markers |
| `snippets/skill.json` | Code snippets | Hardcoded author name; metadata prompts |
| `themes/skill_dark_modern.json` | Theme | Includes bundled VS Code themes |
| `themes/skill-dark.tmTheme` | Theme | No issues |
| `themes/dark_modern.json` | Bundled VS Code theme | Provenance undocumented |
| `themes/dark_plus.json` | Bundled VS Code theme | Provenance undocumented |
| `themes/dark_vs.json` | Bundled VS Code theme | Provenance undocumented |
| `README.md` | Documentation | External URLs; malformed email link |
| `CHANGELOG.md` | Documentation | Version mismatch with package.json |
| `LICENSE.md` | License | BSD 2-Clause — no issues |
| `SUPPORT.md` | Documentation | No issues |
| `domain-knowledge/background-information.md` | Internal docs | Contains org-specific content; should not ship in VSIX |
| `.gitignore` | Git config | Only excludes `.vscode/`; no `.vscodeignore` |
| `resources/icons/main.png` | Image asset | No issues |
| `resources/images/*.gif, *.png` | Image assets | No issues (binary, not analyzed for steganography) |

---

## 7. Scope & Limitations

- This review covers **only the source code in this repository**.
- The published VSIX on the VS Code Marketplace was **not** examined. The actual published extension may contain additional code.
- Binary assets (images) were not analyzed for steganographic content.
- No dynamic analysis (runtime behavior testing) was performed.
- The extension's interaction with other installed extensions was not assessed.
