# Building the BiDi fork on macOS

The BiDi (Hebrew/Arabic) support lives in the **HarfBuzz shaper** path and is
gated behind the `-Dfribidi` build option. macOS defaults to the CoreText shaper,
which does **not** contain the BiDi logic — so you must select the HarfBuzz shaper.

## Prerequisites

- Zig `0.15.2` (see `build.zig.zon` `minimum_zig_version`)
- Xcode + the Metal toolchain: `xcodebuild -downloadComponent MetalToolchain`
- FriBidi: `brew install fribidi pkg-config`

## Build

```sh
export PKG_CONFIG_PATH="$(brew --prefix)/lib/pkgconfig:$(brew --prefix)/opt/fribidi/lib/pkgconfig"

# 1. Build GhosttyKit.xcframework with BiDi enabled + the HarfBuzz shaper.
zig build -Dfont-backend=coretext_harfbuzz -Dfribidi=true -Demit-macos-app=false

# 2. Build the macOS app. build.nu now passes the FriBidi linker flags
#    (the xcframework is a static lib whose FriBidi symbols are resolved
#    at the final app link).
nu macos/build.nu --configuration ReleaseLocal   # or Debug
```

The app lands in `macos/build/<configuration>/Ghostty.app`.

Verify FriBidi is linked:

```sh
otool -L macos/build/ReleaseLocal/Ghostty.app/Contents/MacOS/*.dylib | grep fribidi
```

## Notes

- `-Dfont-backend=coretext_harfbuzz` keeps CoreText for font **discovery**
  (native macOS font matching) but uses HarfBuzz for **shaping**, which is where
  the RTL detection / glyph reversal lives.
- A plain `zig build` (default CoreText shaper) compiles but renders **no** BiDi.
