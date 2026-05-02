# openFrameworks Addon Guide

## Standard Addon Directory Structure

```
ofxMyAddon/
├── addon_config.mk          # Build configuration (NOT a Makefile)
├── src/                      # Addon source code
│   ├── ofxMyAddon.h          # Public header
│   └── platform/             # Platform-specific implementations
│       ├── macos/            # macOS-only code (.mm for ObjC++)
│       ├── win/              # Windows-only code
│       └── linux/            # Linux-only code
├── libs/                     # Bundled third-party dependencies
│   └── somelib/
│       ├── include/
│       └── src/
├── testApp/                  # Test/example application
│   ├── addons.make           # Declares: ofxMyAddon
│   ├── Makefile              # Thin wrapper for oF build system
│   └── src/
│       ├── main.cpp
│       └── ofApp.h / ofApp.cpp
├── example-feature/          # Additional examples (optional)
│   ├── addons.make
│   ├── Makefile
│   └── src/
├── README.md
└── LICENSE
```

## addon_config.mk Full Reference

```bash
# Metadata (not used in build, consumed by projectGenerator and oF website)
meta:
    ADDON_NAME = ofxMyAddon
    ADDON_DESCRIPTION = Short description
    ADDON_AUTHOR = Your Name
    ADDON_TAGS = "tag1" "tag2"

# Common settings (applied to all platforms)
common:
    ADDON_INCLUDES = src libs/external/include
    ADDON_SOURCES_EXCLUDE = libs/external/examples/%
    ADDON_SOURCES_EXCLUDE += libs/external/tests/%
    ADDON_DEPENDENCIES = ofxGui ofxOsc

# macOS
osx:
    ADDON_CFLAGS = -DPLATFORM_MACOS=1
    ADDON_CFLAGS += -DSOME_FEATURE=1
    ADDON_FRAMEWORKS = Metal IOSurface Foundation OpenGL
    ADDON_SOURCES_EXCLUDE += src/platform/win/%
    ADDON_SOURCES_EXCLUDE += src/platform/linux/%

# Windows (Visual Studio)
vs:
    ADDON_CFLAGS = /DPLATFORM_WINDOWS=1
    ADDON_CFLAGS += /DSOME_FEATURE=1
    ADDON_LDFLAGS = opengl32.lib d3d11.lib dxgi.lib
    ADDON_SOURCES_EXCLUDE += src/platform/macos/%
    ADDON_SOURCES_EXCLUDE += src/platform/linux/%

# MSYS2/MinGW (alternative Windows toolchain)
msys2:
    ADDON_SOURCES_EXCLUDE += src/platform/macos/%
    ADDON_SOURCES_EXCLUDE += src/platform/linux/%

# Linux x86_64
linux64:
    ADDON_CFLAGS = -DPLATFORM_LINUX=1
    ADDON_LDFLAGS = -lGL -lEGL -ldrm
    ADDON_SOURCES_EXCLUDE += src/platform/macos/%
    ADDON_SOURCES_EXCLUDE += src/platform/win/%

# Linux ARM64
linuxaarch64:
    ADDON_CFLAGS = -DPLATFORM_LINUX=1
    ADDON_LDFLAGS = -lGL -lEGL -ldrm
    ADDON_SOURCES_EXCLUDE += src/platform/macos/%
    ADDON_SOURCES_EXCLUDE += src/platform/win/%

# iOS
ios:
    ADDON_FRAMEWORKS = Metal IOSurface Foundation
    ADDON_SOURCES_EXCLUDE += src/platform/win/%
    ADDON_SOURCES_EXCLUDE += src/platform/linux/%

# Emscripten (WebAssembly)
emscripten:
    ADDON_SOURCES_EXCLUDE += src/platform/macos/%
    ADDON_SOURCES_EXCLUDE += src/platform/win/%
```

## addons.make

Simple text file listing addon names, one per line:

```
ofxMyAddon
ofxGui
ofxOsc
```

This file lives in each **project** directory (test app, example app, or app in `apps/myApps/`). The oF build system reads this to discover which addons to include.

**Common mistake**: Forgetting to create `addons.make` in example projects. The build will succeed for oF core but fail with undefined references for addon types.

## Project Makefile

Every buildable project needs a `Makefile`:

```makefile
ifneq ($(wildcard config.make),)
    include config.make
endif

ifndef OF_ROOT
    OF_ROOT=../../..
endif

include $(OF_ROOT)/libs/openFrameworksCompiled/project/makefileCommon/compile.project.mk
```

The `OF_ROOT` path must point to the oF installation root. `../../..` works when the project is at `addons/{name}/{project}/` (3 levels deep).

## CI with of-actions

Use [2bbb/of-actions](https://github.com/2bbb/of-actions) for automated CI:

### Addon CI

```yaml
# .github/workflows/ci.yml
name: CI
on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  build:
    uses: 2bbb/of-actions/.github/workflows/build-addon.yml@v1
    with:
      of_version: "0.12.1"
      addon_name: "ofxMyAddon"
      test_app: "testApp"
```

### With Windows Preprocessor Defines

```yaml
jobs:
  build:
    uses: 2bbb/of-actions/.github/workflows/build-addon.yml@v1
    with:
      of_version: "0.12.1"
      addon_name: "ofxMyAddon"
      test_app: "testApp"
      preprocessor_defines: "PLATFORM_WINDOWS=1;SOME_FEATURE=1"
```

### App CI

```yaml
jobs:
  build:
    uses: 2bbb/of-actions/.github/workflows/build-app.yml@v1
    with:
      of_version: "0.12.1"
      app_name: "myApp"
```

## Checklist for New Addons

1. `addon_config.mk` with all platform sections (even if empty — future-proof)
2. `src/` with public headers and implementation
3. `testApp/addons.make` listing your addon
4. `testApp/Makefile` with correct `OF_ROOT`
5. `testApp/src/main.cpp` and `ofApp.cpp`
6. `addons.make` in every example project directory
7. Source exclusions for platform-specific code in every platform section
8. Windows: `/D` flags, not `-D`
9. macOS: `.mm` extension for ObjC++ files, pimpl for type hiding
10. Linux: all required `-l` linker flags in `ADDON_LDFLAGS`
