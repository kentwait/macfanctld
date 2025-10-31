## AI working instructions for macfanctld

Focus: This is a tiny C daemon that controls MacBook fans on Linux via the applesmc sysfs. Be pragmatic: small, readable diffs; keep behavior and file paths stable unless explicitly changing them.

- Big picture
  - Entry point: `macfanctl.c` — daemonization, signal handling, main loop.
  - Control logic: `control.c`/`control.h` — discovers applesmc under `/sys/class/hwmon`, reads sensors, computes fan speed, writes to `fan*_min` and resets `fan*_manual`.
  - Config: `config.c`/`config.h` — parses `/etc/macfanctl.conf` and exposes globals (temps, fan_min, log_level, exclude[]).
  - Docs/init: `macfanctld.1` (man page), `init.d` (SysV init script), sample `macfanctl.conf`.
  - Flow: read_cfg → find_applesmc → scan_sensors → loop { adjust(); logger(); if SIGHUP then re-read cfg+rescan } every 5s.
  - Rust port: `rust/` — an updated Rust reimplementation is in progress. Treat the C daemon as canonical for behavior, paths, and log format; mirror config keys and control math when working in Rust.


- Build/run workflow (Linux)
  - Build: `make` (see `Makefile`: `gcc -Wall macfanctl.c control.c config.c -o macfanctld`).
  - Foreground run (for debugging): `sudo ./macfanctld -f` (writes logs to stdout); daemon run writes `/var/log/macfanctl.log` and PID to `/var/run/macfanctld.pid`.
  - Reload config without restart: `kill -HUP $(cat /var/run/macfanctld.pid)`.
  - Install (copies to system paths): `sudo make install` → `/usr/sbin/macfanctld`, `/etc/macfanctl.conf`. Removal: `make uninstall`.
  - Dependency: applesmc-dkms must be present; writing to sysfs requires root.

- Configuration conventions (see `macfanctl.conf` and `macfanctld.1`)
  - Lines are `key: value`. Keys: `fan_min` (<=6200), `temp_*_{floor,ceiling}` for AVG/TC0P/TG0P, `log_level` (0–2), `exclude` (space- or comma-separated sensor numbers like `exclude: 1 7`).
  - Parser trims whitespace, warns on ill-formed lines, clamps to valid ranges. Defaults live in `config.c` and `config.h` (`fan_max` fixed at 6200).

- Control algorithm (specifics that matter when modifying)
  - `scan_sensors()` locates applesmc base path, counts fans, builds sensor list, marks excluded sensors, and caches pointers to TC0P/TG0P by label.
  - `read_sensors()` loads values from `tempN_input` (milli-Celsius → float Celsius).
  - `calc_fan()` linearly scales each source from its floor→ceiling into `[fan_min, fan_max]`, then takes the maximum of AVG, TC0P, TG0P; finally clamps to `[fan_min, 6200]` using `min/max` macros in `config.h`.
  - `set_fan()` writes speed to `fan1_min` (and `fan2_min` if present) and sets `fan*_manual` to `0`. Logging via `logger()` depends on `log_level` and marks which source is controlling with `*`.

- Signals, files, and paths (don’t move lightly)
  - Paths are hardcoded in `macfanctl.c`: PID `/var/run/macfanctld.pid`, log `/var/log/macfanctl.log`, cfg `/etc/macfanctl.conf`.
  - Only flag: `-f` (foreground). `SIGHUP` triggers config reload and sensor rescan; `SIGTERM`/`SIGINT` exit cleanly.

- Project patterns and tips
  - Keep output as `printf` to stdout/file (no syslog). Respect the 5s loop cadence and existing logging format (see man page examples) to avoid breaking operational habits.
  - When adding inputs/policies, mirror the existing pattern: add config keys in `config.c`, expose in `config.h`, hook into `calc_fan()` and reflect in `logger()` with clear labeling.
  - Debugging: compile with `-DDEBUG` to get a 20s post-fork sleep and permissive `umask(0)` before daemonization (attach a debugger). Example: `make CFLAGS="-Wall -DDEBUG"`.
  - Platform assumptions: Linux + applesmc; SysV init script provided (`init.d`). If introducing systemd, keep SysV compatible paths or update man page and packaging (`debian/`).

- Quick references
  - Main loop: `macfanctl.c: main()`.
  - Fan math: `control.c: calc_fan()`; IO to sysfs: `control.c: set_fan()`.
  - Config parsing: `config.c: read_cfg()`, `read_param()`, `read_exclude_list()`.

Caution: Mis-setting fan control on real hardware can cause overheating or noise. Test changes in foreground mode with conservative limits first and monitor temps (`tail -F /var/log/macfanctl.log`).