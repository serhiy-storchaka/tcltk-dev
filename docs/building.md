# Building Tcl/Tk

Tcl is built first, then Tk against Tcl's *build* directory (`--with-tcl=.../tcl/unix`), so Tk can be tested without installing anything.

## Linux (unix/)

```
cd ~/tcltk/tcl9.1/unix
./configure --prefix=$HOME/tcltk/9.1
make -j8

cd ~/tcltk/tk/unix
./configure --prefix=$HOME/tcltk/9.1 --with-tcl=$HOME/tcltk/tcl9.1/unix --enable-bidi
make -j8
make tktest            # test executable, also built by "make test"
make install           # only when an installed copy is needed
```

An example set of trees (one checkout per branch, see the [Fossil cheat sheet](fossil-cheatsheet.md)) and their configure options:

| Tree | configure options |
| --- | --- |
| `tcl9.1/unix` | `--prefix=~/tcltk/9.1` |
| `tk/unix` (main) | `--prefix=~/tcltk/9.1 --with-tcl=~/tcltk/tcl9.1/unix --enable-bidi` |
| `tcl9.0/unix` | `--prefix=~/tcltk/9.0` |
| `tk9.0/unix` | `--prefix=~/tcltk/9.0 --with-tcl=~/tcltk/tcl9.0/unix --enable-bidi` |
| `tk-wayland/unix` | `--prefix=~/tcltk/tk-wayland/inst --with-tcl=~/tcltk/tcl9.1/unix --enable-wayland --enable-symbols` |
| `tk-wayland/build-x11` | out-of-tree X11 build of the same checkout, `--enable-bidi --enable-symbols` |

`./configure --help` lists every option. The ones that matter most:

| Option | Effect |
| --- | --- |
| `--enable-symbols` | debug build (`-g`, no optimisation); `=mem` adds Tcl memory debugging, `=all` both |
| `--disable-shared` | static libraries |
| `--enable-bidi` | Tk: bidirectional text support |
| `--enable-wayland` | Tk: Wayland backend (wayland branch only) |
| `--disable-accessibility` | Tk 9.1: build without the ATK bridge (default is on) |
| `--disable-xft` | Tk: core X fonts instead of Xft |
| `--disable-zipfs` | no zipfs-attached script library |

### Out-of-tree builds

You can configure from any directory, so one checkout can have several builds:

```
mkdir ~/tcltk/tk-wayland/build-x11 && cd $_
../unix/configure --with-tcl=$HOME/tcltk/tcl9.1/unix --enable-bidi --enable-symbols
make -j8
```

### Regenerating configure

Edit `unix/configure.ac` or `tcl.m4`, then regenerate with autoconf **2.73**, the version the committed `configure` scripts are made with. If your system has another version, install 2.73 separately (for example under `~/tcltk/autoconf-2.73`).

```
cd ~/tcltk/tk/unix && ~/tcltk/autoconf-2.73/bin/autoconf
fossil diff configure        # only the intended block should change
```

Commit `configure` together with `configure.ac`.

### Other generated files

```
make genstubs         # after editing generic/*.decls (needs tclsh)
make checkstubs       # compare stub table with exported symbols
make html             # HTML man pages (Tcl and Tk together)
make dist             # source tarball
```

### Rebuilding cleanly

```
make clean            # objects and binaries
make distclean        # also Makefile, config.status: configure again afterwards
fossil extras         # what is left untracked
```

If the clock skews (VMs), `Makefile` can keep regenerating itself. Fix the clock, don't touch timestamps.

## Windows (win/, MSVC)

These examples use a Windows 11 VM reachable by ssh as `winvm`, with sources under `%USERPROFILE%\tcltk`. Use MSVC 2022 Build Tools with `nmake -f makefile.vc` from a `vcvarsall x64` environment.

```
call "C:\Program Files (x86)\Microsoft Visual Studio\2022\BuildTools\VC\Auxiliary\Build\vcvarsall.bat" x64
cd /D %USERPROFILE%\tcltk\tcl91\win
nmake -f makefile.vc MACHINE=AMD64 release
cd /D %USERPROFILE%\tcltk\tk91\win
nmake -f makefile.vc MACHINE=AMD64 TCLDIR=%USERPROFILE%\tcltk\tcl91 release tktest
```

Output goes to `win\Release_AMD64_VC1944\` (`wish91.exe`, `tktest91.exe`; `wish86t.exe` for 8.6).

| makefile.vc variable | Values |
| --- | --- |
| `OPTS` | `symbols` (debug), `pdbs`, `static`, `nomsvcrt`, `nothreads`, `noembed`, `thrdalloc`, `unchecked`, `profile`; comma-separated. `msvcrt` is the default. 8.6 builds use `OPTS=threads` |
| `STATS` | `memdbg`, `compdbg` |
| `CHECKS` | `nodep`, `fullwarn`, `64bit` |
| `TCLDIR` | Tcl source tree (Tk only) |
| `INSTALLDIR` | target of `nmake install` |

Points to remember:

- Kill stray `wish`/`tktest` processes before rebuilding, or linking fails with LNK1104.
- For Tcl, run `release` and `install` as **separate** nmake calls. `makefile.vc` checks for an installed tclsh at parse time.
- GUI programs must run in the interactive desktop session, not over ssh. One way is a scheduled task that runs an uploaded `.cmd` file in the logged-in session.
- A scheduled task starts in `System32`. Every `.cmd` must `cd /d` to a writable directory first.
- Shut the VM down with `ssh winvm "shutdown /s /t 0"` if it ignores the ACPI power button.

## Windows (win/, MSYS2 and MinGW)

`win/configure` builds with gcc from MSYS2 (UCRT64 environment) or with a MinGW cross-compiler on Linux:

```
# MSYS2 UCRT64 shell
cd tcl91/win && ./configure --enable-64bit && make -j8
cd ../../tk91/win && ./configure --enable-64bit --with-tcl=../../tcl91/win && make -j8 && make tktest

# cross-compiling on Linux
../win/configure --host=x86_64-w64-mingw32 --enable-64bit ...
```

## Other Unix systems (VMs)

FreeBSD, NetBSD, OpenBSD, DragonFly, OpenIndiana and openSUSE VMs have X11 dev libraries, HarfBuzz, Xvfb and DejaVu fonts installed. Start one with `VBoxManage startvm NAME --type headless`. Build quirks:

- **OpenIndiana:** use `gmake`. Put `/usr/gnu/bin` first on PATH, because native `tar` rejects the pax headers in the Fossil tarball and native `find` lacks `-empty`.
- **NetBSD:** the root fs is small, so build in `/tmp`.
- **DragonFly:** the login shell is tcsh, so pipe commands through `sh`.

## Old releases

Old releases can be installed side by side under `~/tcltk/<version>`, with the sources in `~/tcltk/rel/<version>/{tcl,tk}`. gcc 15 defaults to C23, so old sources need:

```
CC="gcc -std=gnu17 -fpermissive -U_FORTIFY_SOURCE -D_FORTIFY_SOURCE=0" ./configure ...
```

Per-version source fixes (Tk 8.5.0–8.5.3 X11 init, MSVC 2022 header clashes) are in [old-versions.md](old-versions.md).
