# Testing Tcl/Tk

Tk tests run on a private Xvfb display. Tk 9.1 also needs a private D-Bus. Never run them on the real display `:0`: your own mouse and keyboard make tests flaky, and the ATK bridge attaches to the live desktop session.

## Running the suites

Run these from the build directory (`unix/` or an out-of-tree build). The `make` targets set `LD_LIBRARY_PATH`, `TCL_LIBRARY` and `TK_LIBRARY`, so they test the build, not an installed copy.

```
make test                                  # Tcl: all tests + bundled packages; Tk: test-classic + test-ttk
make test-classic TESTFLAGS="-file font.test"
make test-ttk TESTFLAGS="-file treeview.test"
make shell SCRIPT=repro.tcl                # wish/tclsh from the build tree
make runtest                               # interactive tktest/tcltest
make gdb                                   # wish under gdb (Tcl: also gdb-test)
make valgrind TESTFLAGS="-file text.test"  # Tk: tktest under memcheck
make ro-test                               # Tcl: suite with an unwritable cwd
```

Tk's `tests/all.tcl` and `tests/ttk/all.tcl` default to `-singleproc 1`, all test files in one process. Options in `TESTFLAGS` are applied on top of that default, so `TESTFLAGS="-singleproc 0"` runs each file in its own process.

## tcltest options (TESTFLAGS)

| Option | Meaning |
| --- | --- |
| `-file "a.test b*.test"` | only these files (glob) |
| `-notfile "x.test"` | skip files |
| `-match "font-1.* font-2.3"` | only matching test names |
| `-skip "font-4.*"` | skip test names |
| `-constraints "knownBug"` | turn constraints on |
| `-limitconstraints 1` | run only tests that have the `-constraints` given |
| `-verbose bpstel` | b = body of failures, p = passes, s = skips, t = start, e = error codes, l = line |
| `-singleproc 1` | all files in one process (Tk's default; needed for gdb/valgrind). State left by one file can affect the next |
| `-singleproc 0` | one process per test file (Tcl's default) |
| `-tmpdir DIR` | scratch dir (default `[pwd]`) |
| `-debug 1..3` | tcltest internals |

Run one test:

```
make test-classic TESTFLAGS="-file entry.test -match entry-3.4 -verbose bpe"
```

Running `./tktest` directly skips the environment the `make` targets set, so it may load an installed Tcl or Tk library instead of the build.

## Linux: Xvfb wrapper

```
Xvfb :99 -screen 0 1280x1024x24 -nolisten tcp & echo $! > /tmp/xvfb.pid
export DISPLAY=:99
make test TESTFLAGS="-file wm.test"
kill $(cat /tmp/xvfb.pid)
```

- Put `DISPLAY=:NN` (or `unset DISPLAY`) on **every** command that may start wish or tktest, including quick existence checks. Use `ls`/`test -x` to check that a binary exists rather than running it.
- Kill only your own Xvfb, by PID. Never use `pkill -f Xvfb`.
- `TK_ALT_DISPLAY=:98` (a second Xvfb) enables the `altDisplay` tests.

### Tk 9.1 with ATK: private D-Bus

```
env -u WAYLAND_DISPLAY DISPLAY=:99 HOME=$(mktemp -d) XDG_RUNTIME_DIR=$(mktemp -d) \
    dbus-run-session -- make test
ps -eo lstart,args | grep at-spi     # no new registryd in your session afterwards
```

Clear `DISPLAY`/`WAYLAND_DISPLAY` *before* `dbus-run-session`. Otherwise services it activates connect to the real session and outlive the test. To avoid the issue entirely, build with `--disable-accessibility`.

### Window managers

The `wm`/`unixWm` tests (`withWindowManager`) need a window manager on the display; without one, many of them fail.

| WM | Use |
| --- | --- |
| icewm | preferred: places windows by itself, no D-Bus |
| fvwm, wmaker, i3, dwm, openbox | safe to start directly on Xvfb |
| twm, ctwm, vtwm | only with a private `HOME` whose `.twmrc` has `RandomPlacement` and `NoGrabServer`, else they wait for a click and the run hangs |
| kwin_x11, mutter, xfwm4, metacity | only as `dbus-run-session -- env HOME=$(mktemp -d) XDG_CONFIG_HOME=... wm`: otherwise they take over the live session's shortcuts and rewrite `~/.config` |

```
DISPLAY=:99 icewm & wm=$!
DISPLAY=:99 make test TESTFLAGS="-file wm.test unixWm.test"
kill $wm
```

### Keyboard layouts and dead keys

Xvfb and Xephyr may silently ignore `setxkbmap`/`xmodmap`. Use Xorg with the dummy driver instead. For examples, use a Ukrainian layout and pick letters that have no Latin look-alike (q/й, not C/С).

```
Xorg :78 -config dummy.conf -configdir EMPTYDIR -noreset -nolisten tcp \
     -logfile xorg.log -novtswitch -sharevts &
DISPLAY=:78 setxkbmap us,ua
```

`dummy.conf` needs `Driver "dummy"` and `AutoAddDevices false`. Never inject XTEST input into an Xwayland display, because that opens a remote-control prompt in your session.

### Wayland branch

Use the `~/tcltk/tk-wayland` checkout. Headless sway works; headless Weston crashes GLFW in `glfwInit()`.

`minimal.conf` for sway:

```
output HEADLESS-1 resolution 1280x1024
for_window [all] floating enable
default_border none
default_floating_border none
```

```
PRIVATE=$(mktemp -d); chmod 700 $PRIVATE
env -u DISPLAY -u WAYLAND_DISPLAY XDG_RUNTIME_DIR=$PRIVATE \
  WLR_BACKENDS=headless WLR_LIBINPUT_NO_DEVICES=1 WLR_RENDERER=pixman \
  LIBGL_ALWAYS_SOFTWARE=1 sway -c minimal.conf &
sleep 2; ls $PRIVATE                       # the socket name, usually wayland-1
WAYLAND_DISPLAY=wayland-1 XDG_RUNTIME_DIR=$PRIVATE make test
```

## Windows VM

The test binaries are `tktest91.exe` and `tktest86t.exe`. Run them in the interactive desktop session (for example through a scheduled task that runs an uploaded `.cmd` file), not over ssh:

```
:: env91.cmd
set R=%USERPROFILE%\tcltk
set PATH=%R%\tcl91\win\Release_AMD64_VC1944;%R%\tk91\win\Release_AMD64_VC1944;%PATH%
set TCL_LIBRARY=%R%\tcl91\library
set TK_LIBRARY=%R%\tk91\library
set T=%R%\tk91\win\Release_AMD64_VC1944\tktest91.exe
set TK=%R%\tk91
set O=%USERPROFILE%\tmp
cd /d %O%
```

```
:: test.cmd
call %USERPROFILE%\tmp\env91.cmd
%T% %TK%\tests\all.tcl -file menubut.test > %O%\result.txt 2>&1
```

```
scp winvm:tmp/result.txt .
```

Starting in `System32` makes tests fail with "permission denied" and hangs `winDialog-5.11`. That comes from the environment, not from Tk. Take screenshots with PowerShell `CopyFromScreen` in the session, not `VBoxManage screenshotpng`, which misses windows.

## macOS and CI

macOS can't be tested locally; GitHub Actions is the only macOS check. In Tk, tag the tip with `core-NAME`, push, and look at https://github.com/tcltk/tk/actions after the nightly sync. Cancel the tag once the run has started. Tag only cross-platform changes, because runs cost CI minutes. In Tcl, branches normally wait for a core developer instead.

## Flaky failures and bisecting

- Run each version the same number of times before calling it "good". Use at least 10 runs for a test failure and at least 40 for races or crashes. Report counts ("fails 3/10"), not "passes".
- Classify crashes by stack (`coredumpctl`) before blaming a change.
- Run a test file alone and in the full suite. Some failures depend on test order or on state left behind (fonts, focus, grabs).
- Installed releases `~/tcltk/<version>/bin/wish*` give quick baselines across versions.
