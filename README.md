# static-gdb-mips64

Statically-linked [GDB](https://www.gnu.org/software/gdb/) that runs **natively on
MIPS64 Linux machines** — single portable binaries with no shared-library
dependencies, for copying onto routers, CPE, embedded boards, or any MIPS64 box
where you can't install packages.

Built for **both endiannesses** and **two variants** each:

| Binary | ~Size | Description |
|---|---|---|
| `gdb-<ver>-mips64-static-nopython` | 15 MB | Big-endian, TUI, no Python scripting |
| `gdb-<ver>-mips64-static-python` | 22 MB | Big-endian + embedded CPython 3.12 |
| `gdb-<ver>-mips64el-static-nopython` | 15 MB | Little-endian, TUI, no Python scripting |
| `gdb-<ver>-mips64el-static-python` | 22 MB | Little-endian + embedded CPython 3.12 |

Target profile: **n64 ABI, MIPS64r2 baseline, glibc 2.40** (needs a reasonably
modern kernel on the device). Check your device with `readelf -h /bin/sh |
grep -E 'ELF|flags'` — `ELF64` + MSB/LSB tells you the ABI and endianness.

## Quick start

Requirements: Ubuntu/Debian-ish **x86_64** host (the build cross-compiles), with
`build-essential`, `curl`, and:

```bash
sudo apt-get install -y \
    libgmp-dev libmpfr-dev libexpat1-dev zlib1g-dev libncurses-dev texinfo \
    gcc-14-mips64-linux-gnuabi64 g++-14-mips64-linux-gnuabi64 \
    libc6-dev-mips64-cross binutils-mips64-linux-gnuabi64 \
    gcc-14-mips64el-linux-gnuabi64 g++-14-mips64el-linux-gnuabi64 \
    libc6-dev-mips64el-cross binutils-mips64el-linux-gnuabi64 \
    qemu-user-static
```

Then:

```bash
./build.sh
```

Binaries land in `dist/`, full install trees under `install/<triplet>-<variant>/`.
A full build takes roughly 30–45 minutes on a 20-core machine. The script is
re-runnable: sources, host Python, cross libraries, and cross Pythons are
skipped when already present.

### Options

```bash
GDB_VERSION=16.3     PYTHON_VERSION=3.12.7    # upstream versions
TRIPLETS="mips64-linux-gnuabi64 mips64el-linux-gnuabi64"  # subset to build
VARIANTS="nopython python"                    # subset to build
CROSS_GCC_VER=14    # cross gcc package series on the host
JOBS=$(nproc)       # parallel jobs
BASE=/path/to/dir   # where to place src/deps/build/install/dist (default: repo dir)
```

## Deploying to a device

```bash
scp dist/gdb-16.3-mips64el-static-nopython root@device:/usr/local/bin/gdb
```

For the **python** variants also copy the data-directory (its path is baked in
at build time; keep the relative layout or pass `--data-directory`):

```bash
scp -r install/mips64el-python/share root@device:/usr/local/
```

## Verifying on an x86_64 host (qemu)

The build script sanity-checks every binary under qemu-user. You can also run a
full end-to-end debug session on your workstation:

```bash
# target program under qemu's gdbserver stub
qemu-mips64el-static -g 12345 ./my-mips64el-binary &
qemu-mips64el-static ./dist/gdb-16.3-mips64el-static-nopython \
    -batch -ex 'file ./my-mips64el-binary' -ex 'target remote :12345' \
    -ex 'break main' -ex continue -ex 'info registers pc'
```

## Caveats

- **Thread debugging is limited**: a glibc-static executable cannot `dlopen`
  `libthread_db` (same limitation as any statically linked GDB).
- **Python variant**: extension modules relying on `dlopen` cannot load
  (`import ctypes` fails); pure-Python GDB scripts work. pwndbg/gef will not
  work with a static build.
- Native target only: these debug **n64** userland on the matching endianness.
  To cross-debug MIPS64 from x86_64 instead, a `--enable-targets=all` x86_64
  build is a better fit.
- Static glibc 2.40 binaries need a kernel of roughly 3.2 or newer on the device.

## How it works / notable gotchas

1. **Cross toolchain**: Ubuntu's `gcc-14-<triplet>` + `libc6-dev-*-cross`
   packages (static glibc archives included). GDB's support libraries
   (GMP/MPFR — hard requirements of GDB 16 — plus expat, zlib, ncurses) are not
   packaged for cross use, so they are cross-built statically from source.
2. **libtool swallows `-static`**: GDB's final link runs through libtool, which
   consumes a plain `-static` flag and silently produces a dynamic binary. The
   build relinks with `make -C gdb LDFLAGS="-all-static"`.
3. **qemu-user as the Python bridge**: GDB's configure must *execute* the
   Python it links against. The cross-built Python (static libpython, dynamic
   glibc) is wrapped in a `qemu-<arch>-static -L /usr/<triplet>` script passed
   to `--with-python=` — configure runs the target interpreter under emulation
   and link-tests the real cross `libpython3.12.a`.
4. **CPython 3.12 cross quirks**: `--with-build-python=<host python>` is
   required; `ac_cv_buggy_getaddrinfo=no` (a runtime test false-fails when
   cross-compiling); out-of-tree builds require a pristine source tree, which
   is why every Python build here (host and cross) is out-of-tree.
5. **ncurses widec**: cross ncurses installs only the widec flavor
   (`libncursesw.a`), while readline/TUI look for `-lncurses`/`-ltinfo`; the
   build provides symlink aliases (ABI-compatible for readline's use).

Build layout (all gitignored): `src/`, `deps/` (host + cross Pythons and
libraries), `build-*/`, `install/`, `dist/`.

## CI

`.gitlab-ci.yml` builds all four variants on push and exposes `dist/` as job
artifacts (an `ubuntu:24.04` image installing its own toolchains and qemu in
`before_script`).

## Publishing this repo

After creating the project on GitLab:

```bash
git remote add origin git@gitlab.com:<user>/static-gdb-mips64.git
git push -u origin main
```

Prefer attaching `dist/` binaries to a GitLab **Release** over committing them.

## License

GDB is GPLv3; statically-linked GDB binaries are derivative works, so if you
distribute the binaries you must offer the corresponding source (this repo plus
the pinned upstream versions is a reasonable way to do that). No license is
claimed over these build files — add one of your choosing before publishing.
