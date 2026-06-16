---
name: translate-project-i18n-java
description: Translates the natural-language identifiers in a Java project from one human language to another (e.g. German to English) while keeping the project compilable and runnable — renaming classes, methods, fields, parameters, variables, packages, enum/constant names, comments, and string literals, and updating associated .properties, .json, and .xml files. ONLY use this skill when the user explicitly invokes it by name (e.g. "use the java-i18n-rename skill"). Do NOT trigger this skill automatically based on the topic alone — even if the user asks to translate or rename identifiers in Java code, do not use this skill unless they have asked for it by name. Parameters: source language (Sprache 1), target language (Sprache 2).
---

# Java Identifier Translation

## Purpose
Translate the human-readable identifiers of a Java codebase from a **source human language** to a **target human language** while guaranteeing the project still compiles and runs identically. This is a refactoring/rename task, NOT a programming-language conversion.

## Required parameters
Before starting, confirm both:
- `SOURCE_LANG` (Sprache 1): the human language identifiers are currently written in.
- `TARGET_LANG` (Sprache 2): the human language to translate identifiers into.

If either is missing, ask before proceeding.

## What to translate
- Class, interface, enum, record, and annotation names
- Method and constructor names
- Field, parameter, and local variable names
- Enum constants and named constants
- Package name segments that carry meaning (renames the directory too)
- Comments (line, block, Javadoc)
- User-facing string literals where appropriate

## Naming conventions
Use idiomatic Java casing for the target-language names, regardless of the old casing: `PascalCase` for types, `camelCase` for methods/fields/vars, `UPPER_SNAKE_CASE` for constants, `lowercase` for packages.

## Already in target language
- Determine the human language of each identifier first. Identifiers (or word parts of compound identifiers) already in TARGET_LANG are left exactly as they are — do not re-translate, re-spell, or "normalize" them.
- For compound identifiers mixing both languages, translate only the SOURCE_LANG parts and keep the TARGET_LANG parts unchanged.
- Mark such identifiers in the glossary as "already in target language — unchanged".
- When unsure whether a word is SOURCE_LANG or TARGET_LANG (cognates, ambiguous abbreviations, technical terms identical in both), leave it unchanged and note it rather than risk a wrong translation.

## Config and data files
Treat these as first-class translation targets, not just places to patch references:
- **.properties** (resource bundles, `application.properties`): translate human-readable key segments and values; keep interpolation placeholders (`{0}`, `%s`, `${...}`) intact.
- **.json** (i18n bundles, config): translate natural-language keys/values; NEVER touch structural keys that code or tooling depends on by string.
- **.xml** (Spring config, `pom.xml`, Android resources, `web.xml`): translate human-readable element/attribute values and comments; do NOT rename schema-defined element/attribute names.

**Critical coupling rule:** if you rename a code identifier that is also referenced by string in a config/data file (Spring bean ids, Jackson `@JsonProperty`, JPA `@Column`, resource-bundle keys, `@Value("${...}")` keys), update the code AND every config occurrence in the same batch, or leave both unchanged. A mismatch silently breaks the app at runtime.

## What must NEVER change
- Java keywords, standard-library names, and third-party API symbols (`String`, `List`, `getClass`, framework annotations, etc.).
- `@Override` members: renaming requires renaming the declaration in the supertype/interface too, or it breaks.
- Names that are semantically load-bearing for a framework: JavaBean getters/setters bound by reflection, JPA/Jackson field↔column/JSON mappings, Spring bean names, `serialVersionUID`, `main`, JUnit/TestNG lifecycle methods. When unsure, preserve the name, or keep the original wire name via mapping (e.g. `@JsonProperty("originalName")`, `@Column(name="...")`).
- Anything referenced reflectively or by string: `Class.forName`, `getMethod("...")`, resource-bundle keys, config keys — update the string too or do not rename.
- The external contract: public APIs consumed outside this project are renamed only if the user confirms downstream consumers are in scope.

## Procedure
1. **Read this SKILL.md fully, then inventory the project.** Map the source tree, build file (`pom.xml` / `build.gradle`), and config/data files (.properties/.json/.xml). Confirm how to build (`mvn compile`, `gradle build`, or `javac`).
2. **Establish a baseline.** Compile as-is and, if tests exist, run them. If it does not compile before translation, report and stop — you cannot verify correctness otherwise.
3. **Build a translation glossary.** For each unique SOURCE_LANG identifier, propose its TARGET_LANG equivalent using the conventions above. Skip identifiers already in TARGET_LANG. Keep the mapping consistent — the same source word always maps to the same target word across code AND config/data files.
4. **Rename safely, symbol-by-symbol — never blind find-replace.** Textual replace corrupts substrings and library names. Prefer scope-aware renaming: rename a declaration together with all its references; anchor any regex on word boundaries and verify each hit. When a class is renamed, rename its file; when a package is renamed, move the directory and fix all imports.
5. **Update config and data files in lockstep** per the rules above.
6. **Recompile after each meaningful batch.** Fix every error before continuing. Do not let errors accumulate.
7. **Final verification.** Run a clean build and any tests. Results must match the baseline.

## Output
- The translated, compiling project in the outputs directory.
- A glossary mapping every SOURCE_LANG → TARGET_LANG identifier (including those left unchanged because already in TARGET_LANG), covering code and config/data files, so the rename is auditable and reversible.
- A short note listing identifiers deliberately left unchanged and why (already in target language, framework constraint, external contract, etc.).

## Guardrails
- Behavior must be equivalent at runtime. Translation changes names, never logic.
- If a rename would break the build or a framework/string binding and there is no safe equivalent, leave it unchanged and record it rather than guessing.
- Preserve formatting, license headers, placeholders, and unrelated content untouched.