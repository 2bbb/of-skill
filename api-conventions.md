# openFrameworks API Conventions

## Core Pattern

oF uses a consistent lifecycle across most objects:

```cpp
class MyAddon {
public:
    void setup();    // Initialize once
    void update();   // Called every frame before draw
    void draw();     // Called every frame
    void exit();     // Cleanup on shutdown
    
    bool isSetup();  // Check initialization state
};
```

## ofApp Structure

`ofApp.cpp` is the main application file. Single-file implementation is preferred initially.

```cpp
#include "ofMain.h"
#include "ofxYourAddon.h"

class ofApp : public ofBaseApp {
    void setup() override;
    void update() override;
    void draw() override;
    void exit() override;
    
    void keyPressed(int key) override;
    void keyReleased(int key) override;
    void mouseMoved(int x, int y) override;
    void mouseDragged(int x, int y, int button) override;
    void mousePressed(int x, int y, int button) override;
    void mouseReleased(int x, int y, int button) override;
    void windowResized(int w, int h) override;
};
```

## Memory Management

- oF uses raw pointers internally. No smart pointers in the public API.
- Textures, FBOs, and other GPU resources are managed via `allocate()` / `clear()` pattern, not RAII.
- Always call `allocate()` before first use. Check with `isAllocated()`.
- `ofTexture`, `ofFbo`, `ofSoundPlayer` all follow allocate/use/clear lifecycle.

```cpp
ofTexture tex;
tex.allocate(width, height, GL_RGBA);
// ... use tex
tex.clear();
```

## OpenGL in oF

- oF wraps OpenGL but doesn't hide it. You can mix raw GL calls with oF calls.
- **Coordinate systems**: oF uses ARB (arbitrary) textures by default — pixel coordinates. Standard GL textures use normalized [0,1] coordinates. Check which mode is active.
- **Vertex shaders**: When writing GLSL, `modelViewProjectionMatrix` is provided by oF. Use it unless you have a reason not to.
- **GL versions**: oF 0.12 uses OpenGL 2.1 core on macOS (deprecated by Apple but functional). Programmable pipeline is available via `ofGLProgrammableRenderer`.

## Logging

```cpp
ofLogNotice("MyClass::methodName") << "message";
ofLogError("MyClass::methodName") << "error: " << code;
ofLogWarning("MyClass::methodName") << "warning";
```

Always use the format `ofLogNotice("{ClassName}::{methodName}")` for easy filtering.

## Common oF Types

| Type | Purpose |
|------|---------|
| `ofTexture` | GPU texture wrapper |
| `ofFbo` | Framebuffer object |
| `ofPixels` | CPU-side pixel data |
| `ofImage` | Image (combines ofPixels + ofTexture) |
| `ofShader` | GLSL shader program |
| `ofVboMesh` | Vertex buffer mesh |
| `ofCamera` / `ofEasyCam` | Camera |
| `ofSoundPlayer` | Audio playback |
| `ofTrueTypeFont` | Font rendering |

## File Splitting

When splitting `ofApp.cpp` into multiple files:

1. **macOS**: New `.cpp` files must be added to the Xcode project manually. Build will silently ignore files not in the project.
2. **Linux/Makefile**: The makefile system automatically globs `src/*.cpp` — new files are picked up automatically.
3. **Windows**: The projectGenerator regenerates `.vcxproj` from `addons.make` + `addon_config.mk`. New source files in addon `src/` are auto-included.

**Always confirm with the user before splitting files** — the build system behavior varies by platform.
