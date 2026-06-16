---
name: translate-project-i18n-python
description: Translates the natural-language identifiers in a Python project from one human language to another (e.g. German to English) while keeping the project importable and runnable — renaming classes, functions, methods, variables, parameters, attributes, modules, constants, comments, and docstrings, and updating associated .properties, .json, .yaml, .ini, .toml, and .env files. ONLY use this skill when the user explicitly invokes it by name (e.g. "use the python-i18n-rename skill"). Do NOT trigger this skill automatically based on the topic alone — even if the user asks to translate or rename identifiers in Python code, do not use this skill unless they have asked for it by name. Parameters: source language (Sprache 1), target language (Sprache 2).
---

# Python Identifier Translation

## Purpose
Translate the human-readable identifiers of a Python codebase from a **source human language** to a **target human language** while guaranteeing the project still imports/parses and runs identically. This is a refactoring/rename task, NOT a programming-language conversion.

## Required parameters
Before starting, confirm both:
- `SOURCE_LANG` (Sprache 1): the human language identifiers are currently written in.
- `TARGET_LANG` (Sprache 2): the human language to translate identifiers into.

If either is missing, ask before proceeding.

## What to translate
- Class names
- Function and method names
- Variable, parameter, attribute, and property names
- Named constants and enum members
- Module and package names (renames the .py file / directory too)
- Comments (line, block)
- Docstrings (module, class, function)
- User-facing string literals where appropriate

## Naming conventions
Use idiomatic Python casing for the target-language names, regardless of the old casing: `PascalCase` for classes, `snake_case` for functions/methods/variables/attributes, `UPPER_SNAKE_CASE` for constants, `lower_snake_case` for modules and packages.

## Already in target language
- Determine the human language of each identifier first. Identifiers (or word parts of compound identifiers) already in TARGET_LANG are left exactly as they are — do not re-translate, re-spell, or "normalize" them.
- For compound identifiers mixing both languages, translate only the SOURCE_LANG parts and keep the TARGET_LANG parts unchanged.
- Mark such identifiers in the glossary as "already in target language — unchanged".
- When unsure whether a word is SOURCE_LANG or TARGET_LANG (cognates, ambiguous abbreviations, technical terms identical in both), leave it unchanged and note it rather than risk a wrong translation.

## Config and data files
Treat these as first-class translation targets, not just places to patch references:
- **.json** (config, i18n bundles): translate natural-language keys/values; NEVER touch structural keys that code or tooling depends on by string.
- **.yaml/.yml, .ini, .cfg, .toml** (`pyproject.toml`, `setup.cfg`, app config): translate human-readable keys/values and comments; do NOT rename keys that are part of a tool/library schema (e.g. `[tool.poetry]`, `dependencies`, entry-point names).
- **.properties / .env**: translate meaning-bearing keys/values; keep placeholders (`{0}`, `%s`, `${...}`, `{name}`) intact; never rename env var names that code reads via `os.environ[...]` unless you update those reads too.

**Critical coupling rule:** if you rename a code identifier that is also referenced by string in a config/data file (env var names, Pydantic/dataclass field names bound to serialized data, config keys read via `config["..."]`, Django settings, entry-point targets), update the code AND every config occurrence in the same batch, or leave both unchanged. A mismatch silently breaks the app at runtime.

## What must NEVER change
- Python keywords, built-ins, and standard-library / third-party API names (`print`, `len`, `self`, `list`, decorators like `@property`, `@staticmethod`, framework decorators, etc.).
- Dunder names: `__init__`, `__str__`, `__repr__`, `__call__`, `__enter__`, etc. — never rename these.
- Overridden / interface methods: renaming requires renaming the declaration in the base class or protocol too, or it breaks. Methods dictated by a framework (Django `save`, `get_queryset`; Flask/FastAPI route handlers; pytest `test_*` and fixtures; `__post_init__`) keep their required names.
- Names that are semantically load-bearing for a framework or serialization: Pydantic/dataclass field names mapped to JSON/DB columns (use `alias=` / `Field(alias=...)` to keep the wire name if you rename the attribute), SQLAlchemy column/table names, Django model field names tied to migrations, dict keys that mirror an external API.
- Anything referenced reflectively or by string: `getattr`/`setattr`, `globals()[...]`, dynamic imports (`importlib`), `**kwargs` keys consumed by name, string-based config keys — update the string too or do not rename.
- The external contract: public APIs (anything importable by other projects, names in `__all__`) are renamed only if the user confirms downstream consumers are in scope.

## Procedure
1. **Read this SKILL.md fully, then inventory the project.** Map the source tree, packaging/build files (`pyproject.toml`, `setup.py`/`setup.cfg`, `requirements.txt`), and config/data files (.json/.yaml/.ini/.toml/.env). Confirm how to run/verify (`python -m py_compile`, `pytest`, `python -m <module>`).
2. **Establish a baseline.** Byte-compile (`python -m compileall`) and, if tests exist, run them. If it does not parse/run before translation, report and stop — you cannot verify correctness otherwise.
3. **Build a translation glossary.** For each unique SOURCE_LANG identifier, propose its TARGET_LANG equivalent using the conventions above. Skip identifiers already in TARGET_LANG. Keep the mapping consistent — the same source word always maps to the same target word across code AND config/data files.
4. **Rename safely, symbol-by-symbol — never blind find-replace.** Textual replace corrupts substrings and library names. Prefer scope-aware renaming: rename a definition together with all its references; anchor any regex on word boundaries and verify each hit. Watch Python-specific hazards: keyword arguments at call sites must follow a parameter rename; `**kwargs` consumed by name; string keys mirroring attribute names. When a module is renamed, rename its .py file and fix every `import` / `from ... import`.
5. **Update config and data files in lockstep** per the rules above.
6. **Re-verify after each meaningful batch.** Re-compile and fix every error before continuing. Because Python is dynamically typed, also re-run tests where available — many breakages won't surface at compile time.
7. **Final verification.** Run a clean byte-compile and the test suite. Results must match the baseline.

## Output
- The translated, working project in the outputs directory.
- A glossary mapping every SOURCE_LANG → TARGET_LANG identifier (including those left unchanged because already in TARGET_LANG), covering code and config/data files, so the rename is auditable and reversible.
- A short note listing identifiers deliberately left unchanged and why (already in target language, framework constraint, external contract, etc.).

## Guardrails
- Behavior must be equivalent at runtime. Translation changes names, never logic.
- Python's dynamic nature means renames can break silently (string-based lookups, kwargs, serialization). When a safe rename isn't possible, leave it unchanged and record it rather than guessing.
- Preserve formatting, license headers, placeholders, and unrelated content untouched.