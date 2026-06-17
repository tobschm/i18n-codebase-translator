# i18n Codebase Translator

If you have ever worked on a project where the codebase was written in a non-English human language, you know the pain. Identifiers mix German (or French, Spanish, …) domain vocabulary with unavoidable English keywords, making the code feel schizophrenic — and onboarding developers from other countries becomes a real challenge.

This Claude Code plugin translates the **human-language layer** of an entire codebase from one language to another with a single skill invocation. It is a refactoring operation, not a programming-language conversion: logic, behavior, and runtime semantics stay identical.

## Supported languages

| Programming language | Skill |
|---|---|
| Java | `translate-project-i18n-java` |
| JavaScript / TypeScript | `translate-project-i18n-javascript` |
| Python | `translate-project-i18n-python` |

Beyond source files, associated config and data files are treated as first-class targets: `.properties`, `.json`, `.xml`, `.yaml`, `.env`, and more.

## What gets translated

- Class, interface, function, method, and variable names
- Parameters, constants, and enum members
- Module / package / file names (the files are renamed too)
- Comments, docstrings, and JSDoc
- User-facing string literals
- Human-readable keys and values in config files

Idiomatic casing for the target language is applied automatically (`PascalCase`, `camelCase`, `snake_case`, `UPPER_SNAKE_CASE` — whichever fits the context).

## What is never touched

- Language keywords, standard-library names, and third-party API symbols
- Framework-reserved names (lifecycle hooks, annotations, decorators, etc.)
- Names bound by string in config files, unless the code reference is updated in the same batch
- Everything matched by `.gitignore` — dependencies, build outputs, and lock files are off-limits

## Usage

Invoke the skill for your project's primary language and provide the source and target human language:

```
/translate-project-i18n-java   German → English
/translate-project-i18n-javascript   French → English
/translate-project-i18n-python   Spanish → English
```

The skill will:
1. Establish a clean build/test baseline before touching anything
2. Build a translation glossary so every rename is consistent and auditable
3. Rename symbols safely — no blind find-and-replace; renames are scope-aware
4. Update code and config files in lockstep to avoid silent runtime mismatches
5. Recompile / re-run tests after each batch and again at the end
6. Produce a full glossary mapping every source identifier to its translated equivalent, including identifiers left unchanged and why

## Output

The translated project plus a glossary file documenting every rename decision. The glossary makes the transformation fully auditable and reversible.

## Author

Tobias Schmelzle
