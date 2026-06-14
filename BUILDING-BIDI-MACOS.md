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

## Apple Silicon: build arm64-only

`ReleaseLocal`/`Release` build a **universal** (arm64 + x86_64) binary, but
Homebrew's `libfribidi` on Apple Silicon is **arm64-only**, so the x86_64 slice
fails to link. Build arm64-only:

```sh
env -i HOME="$HOME" PATH=/usr/bin:/bin:/usr/sbin:/sbin \
  xcodebuild -project macos/Ghostty.xcodeproj -scheme Ghostty \
  -configuration ReleaseLocal SYMROOT="$PWD/macos/build" \
  ARCHS=arm64 ONLY_ACTIVE_ARCH=YES \
  'OTHER_LDFLAGS=$(inherited) -L/opt/homebrew/lib -lfribidi' build
```

(`Debug` is arm64 native by default, so `nu macos/build.nu` works unchanged there.)
A universal build needs an x86_64 `libfribidi` too (e.g. a universal/`x86_64` brew).

## Notes

- `-Dfont-backend=coretext_harfbuzz` keeps CoreText for font **discovery**
  (native macOS font matching) but uses HarfBuzz for **shaping**, which is where
  the RTL detection / glyph reversal lives.
- A plain `zig build` (default CoreText shaper) compiles but renders **no** BiDi.
