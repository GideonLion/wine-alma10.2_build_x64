# Wine 11.0 — Linux x86-64 binary build

A from-source build of **Wine 11.0** with **new WoW64**: 32-bit Windows programs run without any
32-bit Unix libraries on the host. Built on a distribution that ships no 32-bit packages at all.

Packaged by **GideonLion**. Provided as-is, with no warranty of any kind.

For low-latency audio into JACK/PipeWire, see the companion **wine-asio-dist** package.

---

## 1. Read this first — will it run on your machine?

Two hard requirements. If either fails this build will not work, and there is no fix short of
rebuilding from source (§5).

| Requirement | Why | Check |
|---|---|---|
| **CPU with `x86-64-v3`** (Intel Haswell 2013+, AMD Excavator 2015+) | Compiled with `-march=x86-64-v3`. On an older CPU it dies with **SIGILL / "Illegal instruction"**, instantly and with no useful message. | `grep -o avx2 /proc/cpuinfo \| head -1` → must print `avx2` |
| **glibc 2.38+** | Built against glibc 2.39; binaries reference symbols versioned to `GLIBC_2.38`. | `ldd --version \| head -1` |

Known-good: RHEL 10 and rebuilds (AlmaLinux/Rocky 10), Fedora 39+, Ubuntu 24.04+, Debian 13+.
**Will not run on** RHEL 9, Ubuntu 22.04, or Debian 12 — their glibc is too old.

Wine links only `libc` directly; everything else is dlopen'd at runtime, so missing optional libraries
degrade features rather than preventing startup.

## 2. The archive is split into 4 parts — join it first

The build ships as four pieces, each under 25 MB, so it survives size-limited transports:

```
wine-11.0-x86_64.tar.zst.part01     24 MB
wine-11.0-x86_64.tar.zst.part02     24 MB
wine-11.0-x86_64.tar.zst.part03     24 MB
wine-11.0-x86_64.tar.zst.part04      6 MB
```

**You need all four.** They are raw byte-ranges of one archive, not four independent tarballs — no
single part can be opened on its own, and a missing part is not recoverable from the others.

Check the parts arrived intact, join them in order, then check the joined result:

```sh
sha256sum -c SHA256SUMS.parts
cat wine-11.0-x86_64.tar.zst.part?? > wine-11.0-x86_64.tar.zst
sha256sum -c wine-11.0-x86_64.tar.zst.sha256
```

The `??` glob expands in sorted order, which is why the parts are numbered rather than lettered — the
order is not optional and concatenating them wrongly produces a corrupt archive that fails the second
checksum. If `sha256sum -c` reports **OK** on both lines, you have the exact bytes that were built and
tested here. You can delete the parts at that point.

On Windows or without a shell, any tool that concatenates binary files works —
`copy /b part01+part02+part03+part04 wine-11.0-x86_64.tar.zst` in `cmd.exe`. Do not use a text-mode
copy; it will corrupt the archive.

## 3. Install

The tree is **relocatable** — Wine finds its own modules relative to the `wine` binary, so it runs from
wherever you unpack it. `/opt` is only a convention.

```sh
sudo tar -C /opt -xf wine-11.0-x86_64.tar.zst
/opt/wine-11.0/bin/wine --version        # expect: wine-11.0
```

Extracting needs `zstd` present (`tar` shells out to it). On Debian/Ubuntu that is the `zstd` package;
on RHEL/Fedora it is installed by default.

On SELinux systems (RHEL/Fedora), relabel **only the directory you just created**:

```sh
sudo restorecon -R /opt/wine-11.0
```

There is **no `wine64`**. Under new WoW64 the single `wine` binary handles both architectures; scripts
calling `wine64` will fail. That is upstream behaviour, not a packaging defect.

## 4. Licensing, and how to get the source

Wine is free software under the **GNU Lesser General Public License, version 2.1 or later**. You may
use, copy, modify and redistribute it under those terms.

| File | Contents |
|---|---|
| `COPYING.LIB` | The LGPL-2.1 text, as shipped by the Wine project |
| `LICENSE` | Wine's license statement |
| `AUTHORS` | The Wine project authors, as that statement requires |

**Source offer.** This is an **unmodified** upstream release. The corresponding source is:

> `https://gitlab.winehq.org/wine/wine`, tag **`wine-11.0`**, commit **`db11d0f`**

Anyone may obtain it directly from that repository. No patches were applied to Wine.

## 5. How it was built

```
../src/configure --enable-archs=i386,x86_64 --prefix=/opt/wine-11.0
make
```

Compiler: GCC 14.3.1 (Red Hat 14.3.1-4), targeting `x86-64-v3`. PE modules are built with the MinGW-w64
cross-compilers — that is what makes new WoW64 work. Without them Wine falls back to ELF builtins and
32-bit support does not function.

`--enable-archs=i386,x86_64` is the point of this build, not a detail: it is the only way to keep
32-bit Windows support on a host with no 32-bit Unix libraries.

**Debug information has been stripped** (`--strip-debug`) from every ELF and PE module and from the
import libraries — 1.6 G down to 680 M. Symbol tables and all loadable sections are intact; only DWARF
was removed. If you need a debuggable Wine, rebuild from the tag above.

## 6. Optional features NOT compiled in

`configure` found these missing and silently disabled them. Everything else was built.

| Missing | Consequence |
|---|---|
| FFmpeg (`libavcodec`/`libavformat`/`libavutil`) | No FFmpeg-backed media decoding |
| `libSDL2` | No SDL2 game-controller backend |
| `libpcsclite` | Smart cards unsupported |
| `libcapi20` | ISDN unsupported |
| `libnetapi` | Samba NetAPI unsupported |
| `libinotify` | Not applicable on Linux — Wine uses the kernel's inotify directly |

FFmpeg is the only one likely to matter in practice. If you need it, rebuild with the FFmpeg
development packages present.
