# raylib-kern

`raylib-kern` provides Kern bindings for [raylib](https://www.raylib.com/).
The Craft package name is `raylib`, so applications import it with:

```kern
use raylib;
```

This repository pins raylib 6.0 as a Craft resource and builds the C sources
through Craft instead of requiring a system `libraylib`.

The public API is hand-maintained and organized by raylib domain:

- `raylib`: root prelude for the first-window path, common types, colors, and
  immediate drawing/input helpers.
- `raylib.window`, `raylib.input`, `raylib.drawing`, `raylib.images`,
  `raylib.textures`, `raylib.text`, `raylib.shaders`, `raylib.models`,
  `raylib.audio`, `raylib.files`, and `raylib.automation`: focused modules for
  the complete Kern-facing wrappers in each area.

`src/raw.rn` is generated as the direct C ABI layer. It is package-internal and
only used by the public modules. Resource behavior is exposed on the resource
values themselves: load or generate through the domain module, then call methods
such as `texture.draw_at(...)`, `image.resize(...)`, `sound.play()`, and
`music.update()`.

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

## First Window Pattern

raylib programs usually have four phases. The public `raylib` module keeps that
shape, but uses Kern-style snake_case names:

1. Initialize the window. This also creates raylib's OpenGL context and default
   font.
2. Run a frame loop until `window_should_close()` reports ESC or the window close
   button.
3. Split each frame into an update phase for input/state and a draw phase between
   `begin_drawing()` and `end_drawing()`.
4. Unload resources, then close the window.

```kern
use raylib;

const TITLE = "raylib-kern first window\0";
const MESSAGE = "Hello from Kern + raylib\0";

fn main() i32 {
    raylib.init_window(800, 450, TITLE);
    defer raylib.close_window();

    // `end_drawing()` applies this frame cap while it swaps buffers and polls
    // input events for the next loop iteration.
    raylib.set_target_fps(60);

    while (!raylib.window_should_close()) {
        // Update: read input and change your game/app state here.

        // Draw: one begin/end pair describes a complete frame.
        raylib.begin_drawing();
            raylib.clear_background(raylib.RAYWHITE);
            raylib.draw_text(MESSAGE, 190, 200, 20, raylib.DARKGRAY);
            raylib.draw_fps(10, 10);
        raylib.end_drawing();
    }

    return 0;
}
```

The structure mirrors raylib's upstream beginner examples: `WindowShouldClose`
drives the loop, input is sampled in the update part, and all drawing happens
between `BeginDrawing` and `EndDrawing`.

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
with `base.abi.cstr` before passing it to the public wrapper.

## Input

Input helpers are frame-polled, so call them from the update part of the loop.
Use `_down` for continuous behavior, such as moving a sprite while an arrow key
is held. Use `_pressed` for one-frame actions, such as toggling a menu or
changing color on a mouse click.

```kern
var pos = raylib.vector2(400.0, 225.0);

if (raylib.is_key_down(raylib.Key.RIGHT)) {
    pos.x += 2.0;
}

if (raylib.is_mouse_button_pressed(raylib.MouseButton.LEFT)) {
    pos = raylib.get_mouse_position();
}
```

## Textures

Textures live in GPU memory. Load them only after `init_window()`, because
raylib needs the OpenGL context first, and unload them before `close_window()`.

```kern
const LOGO = "resources/raylib_logo.png\0";

raylib.init_window(800, 450, "textures\0");
defer raylib.close_window();

let logo = raylib.textures.load_texture(LOGO)..&;
defer logo.unload();

while (!raylib.window_should_close()) {
    raylib.begin_drawing();
        raylib.clear_background(raylib.RAYWHITE);
        logo.draw_at(raylib.vector2(40.0, 40.0), raylib.WHITE);
    raylib.end_drawing();
}
```

For CPU-side image editing, use `raylib.images`. A common flow is:

```kern
let image = raylib.images.load_image(LOGO)..&;
defer image.unload();

image.flip_vertical();
image.draw_text("loaded from an image\0", 8, 8, 18, raylib.BLACK);

let texture = image.to_texture();
defer texture.unload();
```

The module path answers where a resource comes from; the resource value answers
what can be done with it. That keeps texture and image code close to Kern's
handle style instead of repeating `raylib.textures.*` or `raylib.images.*` for
every operation.

## Audio

Audio is independent from the window. Initialize it before loading or playing
audio resources, and close it after owned resources are unloaded.

```kern
raylib.audio.init_audio_device();
defer raylib.audio.close_audio_device();

let sound = raylib.audio.load_sound("resources/click.wav\0")..&;
defer sound.unload();

sound.set_volume(0.8);
sound.play();
```

Use `Sound` for short effects. Use `Music` for longer streams, and call
`music.update()` once per frame while the music is playing:

```kern
let music = raylib.audio.load_music_stream("resources/theme.ogg\0")..&;
defer music.unload();

music.play();
while (!raylib.window_should_close()) {
    music.update();
    // Update and draw the frame.
}
```

## Colors

raylib's named colors are exposed as `pub const` values such as
`raylib.RAYWHITE` and `raylib.DARKGRAY`. They rely on Kern's `const`
compile-time semantics and must not lower to linkable global storage.

Color transforms are available as value methods:

```kern
let overlay = raylib.BLUE.alpha(0.5);
let warning = raylib.YELLOW.lerp(raylib.RED, 0.35);
```

## Binding Generation

raylib changes its API regularly. The long-term source of truth for the raw ABI
is raylib's parser output, not a hand-maintained copy of `raylib.h`. Craft builds
and runs the Kern host tool in `tools/raylib-bindgen` against
`tools/rlparser/output/raylib_api.txt` from the pinned raylib resource.

Only `src/raw.rn` is regenerated. The public wrapper modules are hand-maintained
so they can use Kern naming, module boundaries, focused convenience helpers, and
documentation that explains raylib's ownership and frame-loop expectations.

Regenerate the checked-in bindings after updating the pinned raylib resource:

```sh
craft build --project-path tools/raylib-bindgen
tools/raylib-bindgen/.craft/build/dev/target/out/raylib-bindgen-0.1.0/bin/raylib-bindgen raw \
  .craft/resources/raylib-*/raylib/tools/rlparser/output/raylib_api.txt > src/raw.rn
```

## Scope

The generated raw layer targets raylib 6.0's stable C ABI. The public wrapper
covers the representable raylib areas as focused Kern modules. C varargs and
callback ABI shapes are intentionally omitted until Kern has a suitable public
contract for them.

## License

MIT. raylib itself is distributed separately under its own license.
