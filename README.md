# raylib-kern

`raylib-kern` provides Kern bindings for [raylib](https://www.raylib.com/).
The Craft package name is `raylib`, so applications import it with:

```kern
use raylib;
```

This repository pins raylib 6.0 as a Craft resource and builds the C sources
through Craft instead of requiring a system `libraylib`.

The public shape is:

- `raylib.raw`: direct C ABI declarations using raylib's C names.
- `raylib`: Kern-style type aliases, constants, color constructors, and small helpers.

## Requirements

Craft fetches raylib from Git and compiles the required C modules from the
resource tree. Platform system libraries are still linked by the build script.
On Linux that means OpenGL and X11 development libraries must be present for
examples or applications to link.

On Fedora:

```sh
sudo dnf install mesa-libGL-devel libX11-devel libXrandr-devel libXinerama-devel libXi-devel libXcursor-devel
```

## Quick Start

```toml
[dependencies]
raylib = { git = "https://github.com/softfault/raylib-kern.git" }
```

Local development can use a path dependency:

```toml
[dependencies]
raylib = { path = "../raylib-kern" }
```

Run the included example after platform development libraries are installed:

```sh
craft run --example basic_window
```

## Strings

raylib's C API expects zero-terminated `const char *` strings. Recent Kern
versions type string literals as `[]u8` byte slices, so the ergonomic wrapper
functions accept `[]u8` directly. The binding still does not allocate or append
the C terminator for you: pass a literal with `\0` when calling helpers such as
`init_window`, `draw_text`, `measure_text`, or `load_texture`.

```kern
const TITLE = "raylib-kern basic window\0";
const MESSAGE = "Hello from Kern + raylib\0";

raylib.init_window(800, 450, TITLE);
raylib.draw_text(MESSAGE, 190, 200, 20, raylib.DARKGRAY);
```

For runtime text that is not already terminated, allocate a temporary C string
with `base.abi.cstr` or call `raylib.raw.*` directly with your own pointer.

## Colors

raylib's named colors are exposed as `pub const` values such as
`raylib.RAYWHITE` and `raylib.DARKGRAY`. They rely on Kern's `const`
compile-time semantics and must not lower to linkable global storage.

## Binding Generation

raylib changes its API regularly. The long-term source of truth for this package
is raylib's parser output, not a hand-maintained copy of `raylib.h`. Craft builds
and runs the Kern host tool in `tools/raylib-bindgen` against
`tools/rlparser/output/raylib_api.txt` from the pinned raylib resource.

Regenerate the checked-in bindings after updating the pinned raylib resource:

```sh
craft build --project-path tools/raylib-bindgen
tools/raylib-bindgen/.craft/build/dev/target/out/raylib-bindgen-0.1.0/bin/raylib-bindgen raw \
  .craft/resources/raylib-*/raylib/tools/rlparser/output/raylib_api.txt > src/raw.rn
tools/raylib-bindgen/.craft/build/dev/target/out/raylib-bindgen-0.1.0/bin/raylib-bindgen public \
  .craft/resources/raylib-*/raylib/tools/rlparser/output/raylib_api.txt > src/lib.rn
```

## Scope

The first binding pass targets raylib 6.0's stable C ABI and covers core,
shapes, textures, text, models, and audio. Variadic C helpers are deliberately
not wrapped as Kern varargs until the language has a clear FFI story for them.

## License

MIT. raylib itself is distributed separately under its own license.
