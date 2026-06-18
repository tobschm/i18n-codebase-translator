---
name: translate-project-i18n-javascript
description: Translates the natural-language identifiers in a JavaScript or TypeScript project from one human language to another (e.g. German to English) while keeping the project building and running — renaming classes, functions, methods, variables, parameters, properties, modules, constants, comments, and JSDoc, and updating associated .json, .yaml, .env, and config files. ONLY use this skill when the user explicitly invokes it by name (e.g. "use the javascript-i18n-rename skill"). Do NOT trigger this skill automatically based on the topic alone — even if the user asks to translate or rename identifiers in JavaScript code, do not use this skill unless they have asked for it by name. Parameters: source language (Language 1), target language (Language 2).
---

# JavaScript Identifier Translation

## Purpose
Translate the human-readable identifiers of a JavaScript/TypeScript codebase from a **source human language** to a **target human language** while guaranteeing the project still builds/parses and runs identically. This is a refactoring/rename task, NOT a programming-language conversion.

## Required parameters
Before starting, confirm both:
- `SOURCE_LANG` (Language 1): the human language identifiers are currently written in.
- `TARGET_LANG` (Language 2): the human language to translate identifiers into.

If either is missing, ask before proceeding.

## What to translate
- Class names (incl. React component names)
- Function, method, and arrow-function variable names
- Variable, parameter, and object-property names
- Named constants and enum members (TS enums)
- Module / file names (renames the file too)
- TypeScript type aliases, interfaces, generics
- Comments (line, block) and JSDoc
- User-facing string literals where appropriate

## Scope: what to ignore
Only translate files that are part of the project's own source. NEVER read, rename, or modify:
- Dependency / build-output directories: `node_modules/`, `dist/`, `build/`, `.next/`, `.nuxt/`, `out/`, `coverage/`, `.cache/`, and any vendored or bundled libraries.
- Everything matched by the project's `.gitignore` (and any nested `.gitignore` files) — treat those paths as off-limits. If a `.gitignore` exists, parse it first and exclude all matching files/directories before doing anything else.
- Version-control and tooling metadata: `.git/`, `.idea/`, etc.
- Lockfiles (`package-lock.json`, `yarn.lock`, `pnpm-lock.yaml`) — never edit these.
If unsure whether a directory is project source or a dependency, exclude it and mention it rather than risk editing third-party code.

## Naming conventions
Use idiomatic JS/TS casing for the target-language names, regardless of the old casing: `PascalCase` for classes / React components / types / interfaces, `camelCase` for functions/methods/variables/properties, `UPPER_SNAKE_CASE` for constants.

## Already in target language
- Determine the human language of each identifier first. Identifiers (or word parts of compound identifiers) already in TARGET_LANG are left exactly as they are — do not re-translate, re-spell, or "normalize" them.
- For compound identifiers mixing both languages, translate only the SOURCE_LANG parts and keep the TARGET_LANG parts unchanged.
- Mark such identifiers in the glossary as "already in target language — unchanged".
- When unsure whether a word is SOURCE_LANG or TARGET_LANG (cognates, ambiguous abbreviations, technical terms identical in both), leave it unchanged and note it rather than risk a wrong translation.

## Config and data files
Treat these as first-class translation targets, not just places to patch references:
- **.json** (i18n bundles, app config): translate natural-language keys/values; NEVER touch `package.json` contract fields (`name`, `version`, `scripts`, `dependencies`, `main`, `exports`, `bin`) or structural keys other code/tooling depends on by string.
- **.yaml/.yml, .toml, .ini**: translate human-readable keys/values and comments; do NOT rename keys that are part of a tool schema (CI configs, `tsconfig` options, linter rules).
- **.env**: translate meaning-bearing values; never rename env var names read via `process.env.X` / `import.meta.env.X` unless you update those reads too. Keep placeholders (`{0}`, `%s`, `${...}`, `{{name}}`) intact.

**Critical coupling rule:** if you rename a code identifier that is also referenced by string in a config/data file (env var names, JSON config keys read via `config["..."]`, i18n message keys referenced by `t("...")`, route names, DI tokens, exported entry names in `package.json`), update the code AND every config occurrence in the same batch, or leave both unchanged. A mismatch silently breaks the app at runtime.

## What must NEVER change
- JS/TS keywords, built-ins, and standard/third-party API names (`Object.keys`, `Array.map`, `Promise`, `console`, `JSON`, framework APIs, etc.).
- Reserved/lifecycle method names: `constructor`, `toString`, `then`, React lifecycle (`render`, `componentDidMount`), React hooks rules (a custom hook must still start with `use`), framework conventions (Next.js `getServerSideProps`, Vue `setup`, Express middleware signature).
- Names tied to the runtime contract: object-property names serialized to/from JSON or an external API, GraphQL field names, DOM/HTML attribute names, CSS class names referenced from stylesheets, React prop names consumed by other components or passed through `...spread`.
- Anything referenced reflectively or by string: bracket access `obj["name"]`, dynamic `import()`, `eval`, template-literal-built identifiers, string keys in reducers/action types — update the string too or do not rename.
- The external contract: anything `export`ed and consumed by other packages, public API surface — renamed only if the user confirms downstream consumers are in scope. (Renaming a named export means updating every `import` of it.)
- Any names and words that are not in the source language. E.g. if the source language (Language 1) german should be translated to english (Language 2), don't translate any spanish words

## Procedure
0. **Load the ignore list.** Read `.claude-plugin/ignored-words.txt` if it exists. Treat every non-empty, non-comment line (lines not starting with `#`) as a protected word that must never be translated, regardless of language. Add these words to the glossary upfront, marked as "ignored by config — unchanged".
1. **Read this SKILL.md fully, then inventory the project.** Map the source tree, `package.json`, build/tooling config (`tsconfig.json`, bundler config, `.eslintrc`), and config/data files (.json/.yaml/.env). Confirm how to build/verify (`npm install`, `npm run build`, `tsc --noEmit`, `npm test`, lint).
2. **Establish a baseline.** Build/typecheck and, if tests exist, run them. If it does not build before translation, report and stop — you cannot verify correctness otherwise.
3. **Build a translation glossary.** For each unique SOURCE_LANG identifier, propose its TARGET_LANG equivalent using the conventions above. Skip identifiers already in TARGET_LANG. Keep the mapping consistent — the same source word always maps to the same target word across code AND config/data files.
4. **Rename safely, symbol-by-symbol — never blind find-replace.** Textual replace corrupts substrings and library names. Prefer scope-aware renaming: rename a definition together with all its references; anchor any regex on word boundaries and verify each hit. Watch JS-specific hazards: object shorthand (`{ foo }` expands to `{ foo: foo }` — renaming the var changes the key unless you keep `{ newName: foo }`); destructuring renames (`const { foo: bar } = x`); object-property keys that double as JSON wire names; bracket vs dot access; JSX prop names. When a module is renamed, rename its file and fix every `import` / `require` / dynamic `import()`.
5. **Update config and data files in lockstep** per the rules above.
6. **Re-verify after each meaningful batch.** Re-build / re-typecheck and fix every error before continuing. JS/TS is partly dynamic — also run tests where available, since string-based and runtime breakages won't surface at build time.
7. **Final verification.** Run a clean build/typecheck and the test suite. Results must match the baseline.

## Output
- The translated, working project in the outputs directory.
- A glossary mapping every SOURCE_LANG → TARGET_LANG identifier (including those left unchanged because already in TARGET_LANG), covering code and config/data files, so the rename is auditable and reversible.
- A short note listing identifiers deliberately left unchanged and why (already in target language, framework constraint, external contract, etc.).

## Guardrails
- Behavior must be equivalent at runtime. Translation changes names, never logic.
- JS/TS dynamism means renames can break silently (object shorthand, string keys, JSON serialization, dynamic access). When a safe rename isn't possible, leave it unchanged and record it rather than guessing.
- Preserve formatting, license headers, placeholders, and unrelated content untouched.