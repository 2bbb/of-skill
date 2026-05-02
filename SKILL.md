---
name: of-skill
description: openFrameworks conventions, gotchas, addon patterns, and build system behavior. Use when writing, reviewing, or debugging any openFrameworks code — addons, apps, or CI.
license: MIT
---

# openFrameworks Skill

C++ toolkit for creative coding. oF is **not** a framework — it's a set of libraries with a consistent API wrapped around OpenGL, audio, video, and input.

**Docs**: https://openframeworks.cc
**Version**: 0.12.1+ (C++17). Use v0.12.1+ unless explicitly told otherwise.
**Platform**: macOS (primary dev), Linux, Windows

## Reference Files

- **`api-conventions.md`** — Coding style, naming, class patterns, oF-specific conventions
- **`gotchas.md`** — Build system behavior, addon_config.mk traps, platform differences, bad know-how
- **`addon-guide.md`** — How to structure, configure, and build an oF addon (addon_config.mk, addons.make, directory layout, CI)

## 30-Second Summary

- **oF is header-heavy with implicit conventions.** No strict ABI, no namespace (everything in global scope with `of` prefix).
- **The build system is NOT standard make.** It uses `compile.project.mk` which parses `addons.make` → reads each addon's `addon_config.mk` → applies platform-specific sections.
- **`addons.make` and `addon_config.mk` are unrelated files serving different purposes.** `addons.make` = which addons to use (in project dir). `addon_config.mk` = how an addon builds (in addon root).
- **camelCase everywhere.** oF uses camelCase for everything: class names (`ofTexture`), methods (`draw()`), variables (`width`), files (`ofApp.cpp`).
- **`ofApp.cpp` is the entry point.** Single-file implementation preferred until refactoring is warranted.

## Naming Convention

| Element | Style | Example |
|---------|-------|---------|
| Classes | camelCase, `of` prefix | `ofTexture`, `ofxOsc` |
| Methods | camelCase | `setup()`, `draw()`, `getWidth()` |
| Variables | camelCase | `frameNum`, `windowWidth` |
| Files | camelCase | `ofApp.cpp`, `ofMain.cpp` |
| Addon prefix | `ofx` | `ofxOsc`, `ofxGui` |
| Member variables | camelCase, no prefix | `width` not `m_width` |


