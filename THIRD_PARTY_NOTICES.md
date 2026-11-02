# Third-party notices

## libmp3lame (LAME) 3.100 — LGPL-2.1

- Source: https://lame.sourceforge.io/ , release 3.100 (2017-10-11)
- Vendored at `third_party/lame` (encoder subset only; see
  `third_party/lame/README.vendor.md` for the exact file list)
- License: GNU Library General Public License, version 2 (LGPL-2.1) —
  see `third_party/lame/COPYING` and `third_party/lame/LICENSE`
- Copyright: LAME project contributors (Mark Taylor, Robert Hegemann,
  Alexander Leidinger and others)
- Use here: FLAC → MP3 320 kbps CBR transcoding. `ncm_jni/build.rs` compiles the
  encoder sources into `libmp3lame.a` and links them **statically** into
  `lib/arm64-v8a/libncm_jni.so`. No LAME source file was modified; the only
  added file is a hand-written `third_party/lame/config.h` equivalent to the one
  LAME's `configure` produces for Android/bionic.

Because the library is statically linked, redistributing the APK requires
satisfying LGPL-2.1 for libmp3lame: the LAME source (and the build script that
produces the object code) must be made available, and recipients must be able
to relink the application against a modified libmp3lame. This repository ships
the unmodified sources under `third_party/lame` together with the build recipe
in `ncm_jni/build.rs` for that purpose. The app itself does not modify LAME and
does not expose the encoder as a separate library.

## ncmc_lib

- Source: https://github.com/kingwingfly/ncmc
- Vendored at `third_party/ncmc_lib`, commit `d8351309e5401af53d344a25168c8f68a2eba368`
- License: MIT (per upstream Cargo metadata; upstream repository does not include a separate LICENSE file at the tested commit)
- Copyright: Louis / upstream contributors

The Android JNI wrapper (`ncm_jni`) adds container sanity checks, full audio
decode validation, FLAC → MP3 transcoding (via LAME) and JNI bindings; it does
not change ncmc_lib's decryption algorithm.

## Other Rust crates

Dependencies are resolved from crates.io at build time. See `ncm_jni/Cargo.lock`
for the exact versions and their respective licenses.
