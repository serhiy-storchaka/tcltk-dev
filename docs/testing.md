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

When the version changes (e.g. to 9.2), `make` builds a library with a new name (`libtcl9tk9.2.so`), but a `tktest` linked before still loads the old one. The `make test` targets relink `tktest`; if you run `./tktest` directly, run `make tktest` first and check `ldd tktest`.

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

Use the `~/tcltk/tk-wayland` checkout. Test under more than one compositor: they fail differently. sway doesn't maximize floating windows (wm-state-2.20 and wm-maxsize-2.7 always fail there), GNOME Shell draws no decorations itself (libdecor's title bar makes a window at least 180 pixels wide), and KWin draws the decorations itself. Headless Weston (14.0) can't be used: it offers no `wl_seat`, and without one `glfwInit()` crashes in libdecor's GTK plugin (`gdk_seat_get_keyboard: assertion 'GDK_IS_SEAT (seat)' failed`). Tk checks for a seat first and refuses to start ("couldn't connect to display").

Each compositor gets a private `XDG_RUNTIME_DIR`, `HOME` and D-Bus, as in the X11 examples above:

```
PRIVATE=$(mktemp -d); chmod 700 $PRIVATE
env -u DISPLAY -u WAYLAND_DISPLAY XDG_RUNTIME_DIR=$PRIVATE HOME=$(mktemp -d) \
    dbus-run-session -- sh -c 'COMPOSITOR & sleep 5; WAYLAND_DISPLAY=NAME make test'
```

| Compositor | COMPOSITOR | NAME |
|---|---|---|
| sway | `WLR_BACKENDS=headless WLR_LIBINPUT_NO_DEVICES=1 WLR_RENDERER=pixman LIBGL_ALWAYS_SOFTWARE=1 sway -c minimal.conf` | the socket in `$PRIVATE`, usually `wayland-1` |
| KWin | `QT_QPA_PLATFORM=offscreen kwin_wayland --virtual --width 1280 --height 1024 --socket wl-kwin` | `wl-kwin` |
| GNOME Shell | `gnome-shell --headless --wayland --no-x11 --virtual-monitor 1280x1024 --wayland-display wl-gnome` | `wl-gnome` |

`minimal.conf` for sway:

```
output HEADLESS-1 resolution 1280x1024
for_window [all] floating enable
default_border none
default_floating_border none
```

Display scale:

- sway: `output HEADLESS-1 resolution 2560x1600 scale 1.5` in the config.
- KWin: `--scale` only makes the virtual output larger. Set `"scale"` (and `"mode"`) of the output in `$XDG_CONFIG_HOME/kwinoutputconfig.json` before starting KWin; the file is written on the first start.
- GNOME Shell: call `org.gnome.Mutter.DisplayConfig.ApplyMonitorsConfig` on the private bus with `gdbus`. Only the scales listed by `GetCurrentState` are accepted (no 1.5 at 2560x1600). The property `layout-mode` 2 selects the physical layout mode, in which mutter reports a wrong xdg-output size for scaled outputs.
- GLFW reports the scale rounded up to an integer; the logical size of an output comes from xdg-output.

Several outputs: `WLR_HEADLESS_OUTPUTS=2` or `swaymsg create_output` for sway, `--output-count 2` for KWin, `--virtual-monitor` twice for GNOME Shell.

X11 Tk can be tested under the Xwayland of these compositors (`--xwayland` for KWin, `xwayland enable` for sway). Find its display by comparing `/tmp/.X11-unix` before and after the compositor starts, never by process name, and never use `:0` or another display of your session.

Run tests in your real Wayland session (`WAYLAND_DISPLAY=wayland-0`) only when a test needs it, e.g. a panel, a real fractional scale or several real monitors: windows appear, and your input can change the results.

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
- To decide on a rare failure, repeat the steps of the test in a loop in one process, together with the setup and the tests before it that it depends on. For example, unixWm-6.3 failed 1/30 in full runs, but its steps after a new window and unixWm-6.1 and 6.2 gave 22/900 against 0/900. Repeating only the body of the test showed nothing.
- Installed releases `~/tcltk/<version>/bin/wish*` give quick baselines across versions.
