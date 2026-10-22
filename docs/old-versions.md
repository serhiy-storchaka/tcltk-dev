# Building Old Tcl/Tk Releases

Old sources don't compile with current toolchains as-is. This page lists the flags and per-version fixes that build every release from 8.5.0 to 8.6.18 (and Tk 8.6.9.1 and 8.6.11.1) with gcc 15 on Linux and MSVC 2022 on Windows, plus a few notes for 8.4.

## Layout used here

| What | Where |
| --- | --- |
| Linux installs | `~/tcltk/<version>/bin/{tclsh,wish}8.X` |
| Linux sources | `~/tcltk/rel/<version>/{tcl,tk}` |
| Windows installs | `%USERPROFILE%\rel\<version>\bin\{tclsh8Xt,wish8Xt}.exe` |
| Windows build scripts | `%USERPROFILE%\rel\` (described below) |

Checking that Tk really initialised: wish keeps running after "Application initialization failed", so ask for a command that only exists once the Tk library loaded:

```
echo 'puts [package provide Tk]/[llength [info commands ::tk::AmpWidget]]; exit' > /tmp/tkcheck.tcl
DISPLAY=:99 timeout 20 ~/tcltk/8.5.4/bin/wish8.5 /tmp/tkcheck.tcl
```

`8.5.4/1` means Tk initialised. Run it on Xvfb, never on `:0`. Pass a script file: piping into wish's stdin can hang with a window open.

## Getting the sources

Release tags are `core-X-Y-Z`, for example `core-8-5-0` or `core-8-6-9-1`. Export them from the local repositories:

```
v=8.5.4; t=core-$(echo $v | tr . -)
mkdir -p ~/tcltk/rel/$v && cd ~/tcltk/rel/$v
fossil tarball $t tcl.tar.gz -R ~/tcltk/tcl.fossil --name tcl && tar xzf tcl.tar.gz
fossil tarball $t tk.tar.gz  -R ~/tcltk/tk.fossil  --name tk  && tar xzf tk.tar.gz
```

## Linux (gcc 15)

gcc 15 defaults to C23, so old sources need an older standard and relaxed errors:

```
CC="gcc -std=gnu17 -fpermissive -U_FORTIFY_SOURCE -D_FORTIFY_SOURCE=0"
cd ~/tcltk/rel/$v/tcl/unix
./configure --prefix=$HOME/tcltk/$v CC="$CC" && make -j8 && make install
cd ~/tcltk/rel/$v/tk/unix
./configure --prefix=$HOME/tcltk/$v --with-tcl=$HOME/tcltk/rel/$v/tcl/unix CC="$CC" && make -j8 && make install
```

What each flag fixes:

| Flag | Without it |
| --- | --- |
| `-std=gnu17` | configure fails with "C compiler cannot create executables" (implicit `int` in its tests); `tkMain.c` clashes with glibc's `_Generic` `strrchr` |
| `-fpermissive` | the same old-C errors in the sources |
| `-U_FORTIFY_SOURCE -D_FORTIFY_SOURCE=0` | 8.5.0 aborts at run time in a fortify check |

### Tk 8.5.0–8.5.3: cannot initialise on current X

wish stops with "no event type or button # or keysym" at `bind Listbox <MouseWheel>` ([Bug 2010422](https://core.tcl-lang.org/tk/tktview/2010422), fixed in 8.5.4). The virtual event numbers collide with newer X event types. Backport the 8.5.4 fix:

- `generic/tk.h`: start `VirtualEvent`, `ActivateNotify`, … at `MappingNotify + 1` instead of fixed numbers.
- `generic/tkEvent.c`: split `realEventMasks` and `virtualEventMasks` like 8.5.4.

## Windows (MSVC 2022)

This uses MSVC 2022 Build Tools with Windows SDK 10.0.26100 (UCRT), and a few local helper scripts in `%USERPROFILE%\rel`:

| Script | Role |
| --- | --- |
| `versions.txt` | releases to build, one per line (8.6.18 … 8.5.0, including 8.6.9.1 and 8.6.11.1) |
| `driver.ps1` | runs `build-rel.cmd` for every version, 4 in parallel; writes `logs\ALL.done` |
| `build-rel.cmd V TCLTAG TKTAG` | builds and installs one release into `rel\V`; skips it if `bin\tclsh*.exe` and `bin\wish*.exe` exist |
| `oldtcl-fixes.ps1`, `oldtk-fixes.ps1` | apply the source fixes below with regexes; each one is a no-op on releases that don't need it |
| `wishcheck.ps1` + `wishcheck.tcl` | start every `wish*.exe` and append `V tk=… amp=…` (or `V TIMEOUT`) to `logs\wishcheck.txt` |

For a Tk-only release such as 8.6.9.1, `driver.ps1` uses the Tcl tag of the first three parts (`core-8-6-9`) and the Tk tag of the full version (`core-8-6-9-1`). Logs go to `logs\<V>.log`, and the result (`OK`, `FAILED`, `skipped`) goes to `logs\<V>.status`.

Run the scripts in the interactive desktop session (see [testing.md](testing.md)), so that `wishcheck` can open windows:

```
powershell -NoProfile -ExecutionPolicy Bypass -File %USERPROFILE%\rel\driver.ps1
:: or a single release:
%USERPROFILE%\rel\build-rel.cmd 8.6.10 core-8-6-10 core-8-6-10
```

What `build-rel.cmd` does for one release:

```
call vcvarsall.bat x64
fossil tarball -R %USERPROFILE%\tcltk\tcl.fossil --name tcl TCLTAG tcl.tgz   :: into rel\src\V
fossil tarball -R %USERPROFILE%\tcltk\tk.fossil  --name tk  TKTAG  tk.tgz
tar xzf ...;  oldtcl-fixes.ps1 rel\src\V\tcl
cd tcl\win
nmake -f makefile.vc MACHINE=AMD64 OPTS=threads INSTALLDIR=rel\V release
nmake -f makefile.vc MACHINE=AMD64 OPTS=threads INSTALLDIR=rel\V install
oldtk-fixes.ps1 rel\src\V\tk
cd ..\..\tk\win
nmake -f makefile.vc MACHINE=AMD64 OPTS=threads ^
      "OPTIMIZATIONS=-O2 -GS -GL -DOEMRESOURCE -DHAVE_VSSYM32_H -FIwinsock2.h" ^
      TCLDIR=rel\src\V\tcl INSTALLDIR=rel\V release install
:: on success rel\src\V is deleted
```

Run Tcl's `release` and `install` as **separate** nmake calls. `makefile.vc` tests `!if exist($(TCLSH))` when it parses the file, so in a single call the install step thinks tclsh is missing. Delete `rel\V\bin` to force a rebuild.

### Per-version fixes

All of these are applied automatically except `IX86=IX86` for Tk 8.4. 8.4 isn't in `versions.txt`, so its builds were done by hand.

| Version | Problem | Fix |
| --- | --- | --- |
| Tk < 8.6.10 | X11's `None` macro breaks `IMAGE_POLICY_ENTRY` in `winnt.h` | `"OPTIMIZATIONS=-O2 -GS -GL -DOEMRESOURCE -DHAVE_VSSYM32_H -FIwinsock2.h"` on the nmake line (`build-rel.cmd` passes it for every version) |
| Tk < 8.6.8 | the SDK defines `SEARCH_ALL` | rename the `SEARCH_*` enum in `tkText.c` to `TK_TEXT_SEARCH_*` |
| Tk 8.5 | `tmschema.h` no longer exists | `ttkWinXPTheme.c`: include `vssym32.h` |
| Tcl < 8.6.3 | `timezone` is not declared | `tclWinTime.c`: `timezone` → `_timezone` (edit the source; a `-D` breaks `struct timeb`) |
| Tcl ≤ 8.5.10 | C2099 "initializer is not a constant" | drop `#define hypot _hypot` from `tclWinPort.h` |
| Tcl ≤ 8.5.8 | `_stati64` mismatch; duplicate errno values | `tcl.h`: `typedef struct _stat64 Tcl_StatBuf`; `tclPosixStr.c`: guard `ENOLINK`/`ENOPROTOOPT` against `ESOCKTNOSUPPORT`/`EPFNOSUPPORT` (as in 8.5.9) |
| Tk 8.4 | the manifest says X86 → 0xC000007B at start | add `IX86=IX86` to the nmake line |

## Using old builds

- For a bisection baseline, run the same script with each `~/tcltk/<version>/bin/wish*`, at least 10 times per version for test failures and at least 40 for races. Report counts.
- To check whether a release contains a fix, use ancestry, not dates or the patchlevel on a branch: `fossil merge-base FIX core-8-6-17` prints FIX itself if the release contains it.
- On Windows, run the binaries in the interactive desktop session (see [testing.md](testing.md)), from a writable directory.
