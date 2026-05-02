# openFrameworks Gotchas

## §1. addons.make ≠ addon_config.mk — Completely Different Files

**The most common confusion.** These two files have similar names but serve entirely different purposes:

**`addons.make`** (lives in project directory — `testApp/`, `example-*/`, `apps/myApps/myApp/`):
- Plain text, one addon name per line
- Tells the build system WHICH addons this project depends on
- Parsed by `compile.project.mk` at build time
- Has NOTHING to do with GNU Make syntax despite the `.make` extension

```
ofxYourAddon
ofxGui
ofxOsc
```

**`addon_config.mk`** (lives in addon root — `addons/ofxYourAddon/`):
- Tells the build system HOW to build this addon
- Has platform-specific sections with compiler flags, include paths, linker flags, source exclusions
- Parsed by the oF build system, NOT by GNU Make directly
- Uses a custom key-value format, NOT standard Makefile syntax

**Both files use `.mk` / `.make` extensions but are NOT standard Makefiles.** They are parsed by oF's custom build system (`compile.project.mk`).

## §2. The Build System is NOT Standard Make

oF's build system lives in `libs/openFrameworksCompiled/project/makefileCommon/compile.project.mk`. It:

1. Reads `addons.make` from the project directory
2. For each addon listed, reads `addons/{name}/addon_config.mk`
3. Applies the platform-specific section matching the current OS
4. Generates compiler/linker commands dynamically

A project's `Makefile` is just a thin wrapper:

```makefile
ifndef OF_ROOT
    OF_ROOT=../../..
endif
include $(OF_ROOT)/libs/openFrameworksCompiled/project/makefileCommon/compile.project.mk
```

**OF_ROOT must be correct** or nothing works. Default is `../../..` (three levels up from the project directory, which puts you at the oF root when the project is in `addons/{name}/{test_app}/`).

## §3. addon_config.mk Platform Keys

Valid section names:

| Key | Platform |
|-----|----------|
| `osx` | macOS |
| `vs` | Visual Studio (Windows) |
| `msys2` | MSYS2/MinGW (Windows) |
| `linux64` | Linux x86_64 |
| `linux` | Generic Linux |
| `linuxarmv6l` | ARMv6 (Raspberry Pi) |
| `linuxarmv7l` | ARMv7 |
| `linuxaarch64` | ARM64 |
| `emscripten` | WebAssembly |
| `ios` | iOS |
| `android` | Android |

**Gotcha**: The key is `osx`, not `macos`. Despite Apple renaming the OS, oF still uses `osx` in addon_config.mk.

**Gotcha**: The key is `vs`, not `windows` or `visualstudio`.

## §4. addon_config.mk Variables

```bash
meta:
    ADDON_NAME = ofxYourAddon
    ADDON_DESCRIPTION = Description here
    ADDON_AUTHOR = Author
    ADDON_TAGS = "tag1 tag2"

common:                          # Applied to ALL platforms
    ADDON_INCLUDES = src libs/external/include
    ADDON_SOURCES_EXCLUDE = libs/external/examples/%
    ADDON_DEPENDENCIES = ofxGui   # Other addons this needs

osx:
    ADDON_CFLAGS = -DDEFINE=1
    ADDON_FRAMEWORKS = Metal IOSurface Foundation OpenGL
    ADDON_SOURCES_EXCLUDE += src/platform/win/%
    ADDON_SOURCES_EXCLUDE += src/platform/linux/%

vs:
    ADDON_CFLAGS = /DDEFINE=1    # Windows uses /D, not -D
    ADDON_LDFLAGS = opengl32.lib d3d11.lib
    ADDON_SOURCES_EXCLUDE += src/platform/macos/%
    ADDON_SOURCES_EXCLUDE += src/platform/linux/%

linux64:
    ADDON_CFLAGS = -DDEFINE=1
    ADDON_LDFLAGS = -lGL -lEGL
    ADDON_SOURCES_EXCLUDE += src/platform/macos/%
    ADDON_SOURCES_EXCLUDE += src/platform/win/%
```

**Key variables:**
- `ADDON_INCLUDES` — Additional include paths (relative to addon root)
- `ADDON_SOURCES_EXCLUDE` — Glob patterns to exclude from compilation (use `%` as wildcard, not `*`)
- `ADDON_CFLAGS` — Compiler flags (`-D` on Unix, `/D` on Windows)
- `ADDON_LDFLAGS` — Linker flags (`-lfoo` on Unix, `foo.lib` on Windows)
- `ADDON_FRAMEWORKS` — macOS/iOS system frameworks
- `ADDON_DEPENDENCIES` — Other oF addons this depends on

**Critical**: Use `%` for wildcards in `ADDON_SOURCES_EXCLUDE`, not `*`. The oF build system uses its own glob syntax.

**Critical**: Windows `/D` vs Unix `-D`. Getting this wrong causes silent build failures.

## §5. Source Exclusion is Mandatory for Multi-Platform Addons

If your addon has platform-specific source files (e.g. Metal code in `.mm` files), you MUST exclude them on other platforms:

```bash
osx:
    ADDON_SOURCES_EXCLUDE += src/platform/win/%
    ADDON_SOURCES_EXCLUDE += src/platform/linux/%
    ADDON_SOURCES_EXCLUDE += libs/nozzle/src/backends/d3d11/%
    ADDON_SOURCES_EXCLUDE += libs/nozzle/src/backends/linux/%
```

Without exclusions, the build system tries to compile all source files, including platform-incompatible ones. This produces inscrutable compilation errors because the build system globs everything under `src/` and `libs/`.

## §6. Objective-C++ Only in .mm Files

oF addons on macOS often need Objective-C++ for Metal, IOSurface, etc.:

- `.mm` files can contain ObjC++
- `.cpp` files must be pure C++
- Headers (`.h`, `.hpp`) must be pure C++ — no ObjC types

Use the pimpl pattern to hide ObjC types:

```cpp
// ofxAddon.h (pure C++)
class ofxAddon {
public:
    ofxAddon();
    ~ofxAddon();
    void setup();
private:
    class Impl;
    std::unique_ptr<Impl> impl;
};

// ofxAddon.mm (ObjC++)
#include "ofxAddon.h"
#import <Metal/Metal.h>

class ofxAddon::Impl {
    id<MTLDevice> device;  // ObjC type hidden here
};

ofxAddon::ofxAddon() : impl(std::make_unique<Impl>()) {}
ofxAddon::~ofxAddon() = default;
void ofxAddon::setup() { /* use impl->device */ }
```

## §7. projectGenerator Path on Windows

The projectGenerator is NOT at the obvious location:

```
of_root\projectGenerator\resources\app\app\projectGenerator.exe
```

NOT at `of_root\projectGenerator\projectGenerator.exe` or `of_root\projectGenerator-vs\projectGenerator.exe`.

Usage:
```powershell
$pg = "of_root\projectGenerator\resources\app\app\projectGenerator.exe"
& $pg --ofPath="of_root" "of_root\addons\ofxMyAddon\testApp"
```

## §8. macOS .app Bundle Path

Built macOS apps are at `bin/{AppName}.app/Contents/MacOS/{AppName}`, not `bin/{AppName}`:

```bash
./bin/testApp.app/Contents/MacOS/testApp
```

## §9. Linux Needs xvfb-run for Headless Execution

On CI (no display server), oF apps need `xvfb-run`:

```bash
xvfb-run ./bin/testApp
```

This applies to any oF app that creates a window, even if it immediately exits.

## §10. oF Release Download URLs are Inconsistent

oF release asset naming changes between versions. Do NOT hardcode URLs:

| Version | macOS | Linux | Windows |
|---------|-------|-------|---------|
| 0.12.1 | `.tar.gz` | `linux64_gcc6` | `vs_release` + `vs_64_release` |
| 0.12.0 | `.zip` | `linux64gcc6` | `vs_release` |
| 0.11.2 | `.zip` | `linux64gcc6` | `vs2017_release` |

Use `gh release view` to resolve asset names dynamically (see `addon-guide.md`).

## §11. addons.make Must Exist in Every Buildable Project

Each project directory that you want to build must have its own `addons.make`. Even if it's just:

```
ofxMyAddon
```

Without this file, the build system doesn't know which addons to include and will fail with "undefined reference" errors for addon types.

## §12. Directory.Build.props for Windows Preprocessor Defines

oF's `addon_config.mk` handles `/D` defines for makefile builds but the Windows projectGenerator does NOT always propagate custom defines to `.vcxproj`. The workaround is to inject a `Directory.Build.props` file:

```xml
<Project>
  <ItemDefinitionGroup>
    <ClCompile>
      <PreprocessorDefinitions>MY_DEFINE=1;%(PreprocessorDefinitions)</PreprocessorDefinitions>
    </ClCompile>
  </ItemDefinitionGroup>
</Project>
```

Place this in the project directory next to the `.vcxproj`.

## §13. oF Caches Compiled Libraries

The compiled oF libraries live in `libs/openFrameworksCompiled/`. This directory should be cached in CI (it takes 5-10 minutes to compile from scratch). Cache key should include the oF version.

## §14. C++17 on MSVC Has No Designated Initializers

MSVC C++17 does not support designated initializers (` .member = value`). All struct initialization must use explicit assignment. This matters when writing cross-platform addon code.
