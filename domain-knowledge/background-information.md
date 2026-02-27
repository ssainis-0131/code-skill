# Cadence SKILL for VS Code - Extension Overview

## Introduction
This Visual Studio Code extension provides comprehensive language support for **Cadence SKILL**, a scripting language used primarily with Cadence design tools (like Virtuoso). The extension aims to modernize the development experience for SKILL developers by bringing standard IDE features like syntax highlighting, code completion, and snippets to VS Code.

## Detailed Feature Set

### 1. Language Support & Association
- **File Extensions:** Automatically activates for `.il`, `.ils`, and `.ocn` files.
- **Language ID:** `skill`

### 2. Syntax Highlighting
- Provides tokenization for SKILL syntax, ensuring code is readable and visually structured.
- Differentiates between keywords, functions, strings, comments, and other language constructs.

### 3. Integrated Themes
The extension comes with custom color themes optimized for SKILL code:
- **Skill Dark Modern** (Standard VS Code Dark Modern style adapted for SKILL)
- **Skill Dark** (Classic Dark theme)

### 4. Code Snippets
A robust library of snippets accelerates coding by providing boilerplate for common structures. Triggers include:
- **Structure:** `defun`, `procedure`, `let`, `lambda`
- **Control Flow:** `if`, `ifelse` (or `else`), `when`, `unless`, `case`
- **Loops:** `foreach`, `for`
- **List Manipulation:** `setof`, `mapcar` (via `lambda` or `mapcarlambda`)
- **Utilities:** `header` (file header), `region` (folding regions), `index`, `author`

### 5. Intelligent Editing Features
- **Code Completion (IntelliSense):**
  - Suggests keywords and functions.
  - Configurable to include specific libraries like "Allegro SKILL" or "Framework II SKILL".
  - Supports user-defined custom keywords.
  - Word-based suggestions support.
- **Go to Definition (`F12`):** Quickly navigate to where a function or variable is defined.
- **Hover Information:** Hovering over symbols displays relevant information.
- **Document Symbols:** Supports outlining the file structure (e.g., in the Outline view or Breadcrumbs).
- **Diagnostics:** Basic error checking, currently focused on `defun` and `procedure` syntax errors.
- **Documentation Generation:** Typing `doc` before a function generates a documentation comment block automatically. This documentation is then accessible via Hover and Code Completion.

### 6. Editor Configuration
- **Comments:** Supports line comments (`;`) and block comments (`/* ... */`).
- **Bracket Matching:** Auto-closing and surrounding support for `()`, `[]`, `{}`, `""`, `''`.
- **Folding:** 
  - Supports standard block folding.
  - **Custom Regions:** specially supports `;region` and `;endregion` markers for custom code folding similar to C# regions.
- **Indentation:** specific indentation rules defined for SKILL constraints.

## Configuration Options
Users can customize the extension behavior via VS Code settings:
- `skill.completion.wordBasedSuggestions`: Toggle word-based completion.
- `skill.completion.enableAllegroSkillFunctions`: Enable suggestions for Allegro PCB tools.
- `skill.completion.enableFrameworkIISkillFunctions`: Enable suggestions for Framework II.
- `skill.completion.userCustomKeywords`: Add a pipe-separated list of custom keywords to the completion list.
- `skill.completion.maxNumberOfCompletions`: Limit the number of suggestions shown.
- `skill.diagnostic.maxNumberOfProblems`: Limit the number of reported errors.

---

# Software Specification

## 1. Product Name
**Skill** (Extension for Visual Studio Code)

## 2. Publisher
**herbertagosto**

## 3. Version
**0.5.0** (Current)

## 4. Target Ecosystem
- **Host Application:** Visual Studio Code (Engine `^1.0.0`)
- **Target Language:** Cadence SKILL
- **Associated File Types:** `.il`, `.ils`, `.ocn`

## 5. Functional Requirements

### 5.1 Syntax Highlighting
- **Grammar Source:** `syntaxes/skill.tmLanguage`
- **Scope Name:** `source.skill`
- **Requirement:** Must correctly tokenize SKILL language specific constructs (S-expressions, function calls, keywords).

### 5.2 Snippets
- **Source:** `snippets/skill.json`
- **Requirement:** Must provide tab-completion templates for the following keys:
  - `header`, `region`, `defun`, `procedure`, `let`, `when`, `unless`, `case`, `if`, `ifelse/else`, `foreach`, `for`, `setof/linq`, `mapcarlambda/lambda`, `index`.

### 5.3 Language Configuration
- **Source:** `skill.configuration.json`
- **Comments:** Line (`;`), Block (`/* */`)
- **Brackets:** `()`, `[]`, `{}`
- **Auto-Closing:** Enabled for brackets and quotes.
- **Folding:** Marker-based folding for `;region` / `;endregion`.

### 5.4 UI Contributions
- **Themes:**
  - Skill Dark Modern (`themes/skill_dark_modern.json`)
  - Skill Dark (`themes/skill-dark.tmTheme`)

### 5.5 Settings (Configuration schema)
| Key | Type | Description |
| :--- | :--- | :--- |
| `skill.completion.wordBasedSuggestions` | Boolean | Enable/Disable word-based suggestions. |
| `skill.completion.enableAllegroSkillFunctions` | Boolean | Include Allegro function signatures. |
| `skill.completion.enableFrameworkIISkillFunctions` | Boolean | Include Framework II function signatures. |
| `skill.completion.userCustomKeywords` | String | User-defined keywords separated by `|`. |
| `skill.completion.maxNumberOfCompletions` | Number | Max items in suggestion list. |
| `skill.diagnostic.maxNumberOfProblems` | Number | Max diagnostic issues to report. |

## 6. Development & Repository Info
- **Repository Type:** Git
- **URL:** https://github.com/herbertagosto/code-skill.git
- **License:** BSD 2-Clause (See `LICENSE.md`)
- **Change Log:** See `CHANGELOG.md` for version history.

## 7. Licensing & Usage Rights (Analysis)
Based on the **BSD 2-Clause License** included in the repository:
- **Commercial Use:** Allowed. You can use this extension for commercial purposes, including internal usage at Intel.
- **Modifications:** Allowed. Creating a fork for customization is permitted.
- **Attribution Requirement:** You must retain the original copyright notice (`Copyright (c) 2016, herbertagosto`), the list of conditions, and the disclaimer in any redistributions (both source code and binary forms).
