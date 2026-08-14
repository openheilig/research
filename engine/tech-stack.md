# The Linux port's tech stack

Measured from the binary, not from documentation.

`install/sacred` is a stripped 32-bit i386 ELF with **615 dynamic imports**.

## What LGP swapped in for the Windows middleware

| Windows original | Linux replacement |
|---|---|
| Miles Sound System 6 | `libopenal.so.1` + `libvorbisfile` / `libvorbis` / `libogg` |
| Bink video | `libavcodec.so` / `libavformat.so` / `libavutil.so` (**unversioned**) |
| TINCAT2 networking | `libgrapple.so` |
| DirectX 8 | `libSDL-1.2.so.0` + `libX11` / `libXext` / `libXi` |
| Win32 launcher UI | `libgtk-1.2` / `libgdk-1.2` / `libglib-1.2` / `libgmodule-1.2` |
| — | `libcrypto`/`libssl` 0.9.8, `libfreetype.so.6`, `libjpeg.so.62`, `libxml2.so.2`, `libz.so.1`, `libasound.so.2` |

The **unversioned `libav*.so` plus OpenSSL 0.9.8 are the fragile set** — they
must be the exact 2005/2010-era builds, which is why they are bundled rather
than resolved from the system.

Import families: libc 343 · libstdc++ 90 · SDL 47 · OpenAL 22 · pthread 16 ·
vorbis/ogg 8 · Xlib 7 · zlib 2.

## Class information is intact

> **The "no RTTI in the Linux binary" claim is false.** The *symbol table* is
> stripped; the Itanium-ABI RTTI is fully intact. `tools/vtables.py` recovers
> **318 classes, 317 with vtable and base**. The earlier claim came from an
> extractor that looked for MSVC-style RTTI and found none, which is a fact
> about the extractor.
>
> The 2001 `armalion.exe` likewise retains full MSVC RTTI — 127 auto-named
> vftables, with `__RTDynamicCast` called against `cObject` / `cCreature`
> type descriptors inline. Class reconstruction requires no Windows detour on
> either binary.

## Confirmed engine addresses

German Underworld executable, image base `0x400000`:

| VA | What |
|---|---|
| `0x008485E2` | engine malloc |
| `0x0066F1C0` | debug print |
| `0x00553080` | UI create-by-name |
| `0x00816BD0` | `CreateMutexA("SACRED_INSTANCE")` |
| `0x0084C704` | entry point |
| `0x00856A8A` | `balance.bin` loader hook |

Linux binary, script interpreter: opcode jump table `0x086f4298`, dispatcher
`0x0826e8ce`, argument-tag table `0x086f3fbc`. See
[../formats/script-bytecode.md](../formats/script-bytecode.md).

## String table

`global.res` is an `ID → {ger, eng, rus}` table, XOR-obfuscated per-WORD with
key `0x45AD`. Loader at VA `0x0080DBF0`.

---
Provenance: measured from `install/sacred` with `tools/vtables.py` and
`ehframe_funcs.py`; import families counted from the ELF directly.

