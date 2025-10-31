# Rust Port Action Plan for macfanctld

## Overview
This document outlines the action plan for porting `macfanctld` from C to Rust. The Rust port must maintain behavioral compatibility with the C daemon while leveraging Rust's safety features and modern programming patterns.

## Principles
- **Behavioral Compatibility**: Mirror the C daemon's control algorithm, configuration format, logging output, and file paths exactly
- **Safety First**: Leverage Rust's type system and ownership model to eliminate undefined behavior
- **Pragmatic Approach**: Small, incremental changes; prefer clarity over clever abstractions
- **Zero Breaking Changes**: Users should be able to swap binaries without config changes

---

## Phase 1: Project Setup & Dependencies

### 1.1 Git Submodule Setup
The `/rust` directory is a **git submodule** that will become a standalone project. This allows the Rust port to evolve independently while maintaining a reference in the main repository.

```bash
# Initialize the submodule (if not already done)
cd /home/kent/repos/kentwait/macfanctld
git submodule add <rust-repo-url> rust

# Or if creating from scratch:
mkdir -p rust
cd rust
git init
cargo init --name macfanctld
```

### 1.2 Cargo Project Structure (inside `/rust`)
```
rust/                        # Git submodule root
├── .git/                    # Separate git repository
├── Cargo.toml               # Package manifest
├── Cargo.lock               # Dependency lockfile
├── README.md                # Rust-specific documentation
├── LICENSE                  # GPL-3.0 (same as parent)
├── .gitignore               # Rust-specific ignores
├── src/
│   ├── main.rs              # Entry point, daemonization, signal handling
│   ├── config.rs            # Configuration parsing
│   ├── control.rs           # Fan control logic
│   ├── sysfs.rs             # applesmc sysfs interaction
│   └── sensor.rs            # Sensor data structures
├── tests/
│   ├── integration_tests.rs
│   └── config_tests.rs
└── examples/                # Optional: test utilities
    └── read_sensors.rs      # Debug tool to read sensor values
```

### 1.3 Core Dependencies (`rust/Cargo.toml`)
Package metadata for standalone project:

```toml
[package]
name = "macfanctld"
version = "0.7.0"  # Continue versioning from C daemon
authors = ["Mikael Strom <mikael@sesamiq.com>", "Contributors"]
edition = "2021"
rust-version = "1.70"  # MSRV
license = "GPL-3.0-or-later"
description = "Fan control daemon for MacBook on Linux"
repository = "https://github.com/kentwait/macfanctld-rs"  # Submodule repo
keywords = ["macbook", "fan-control", "daemon", "applesmc"]
categories = ["hardware-support", "os::linux-apis"]

[[bin]]
name = "macfanctld"
path = "src/main.rs"

[dependencies]
```

**Dependencies**:
- **`nix`** or **`libc`**: Unix signal handling, daemon functionality
- **`serde`**: Configuration serialization (optional, manual parsing preferred for compatibility)
- **`anyhow`** or **`thiserror`**: Error handling
- **`log`**: Logging facade (output to stdout/file, not syslog)
- **`signal-hook`**: Safe signal handling in async context

### 1.4 Build Configuration (`rust/Cargo.toml` additions)
```toml
[profile.release]
opt-level = 3
lto = true
codegen-units = 1
strip = true  # Smaller binary

[profile.dev]
opt-level = 0

[features]
default = []
debug-daemon = []  # Equivalent to C's -DDEBUG: 20s sleep before daemonization
```

### 1.5 Rust-Specific Files

**`rust/.gitignore`**:
```gitignore
/target
Cargo.lock
**/*.rs.bk
*.pdb
```

**`rust/README.md`** (basic structure):
```markdown
# macfanctld (Rust Port)

Rust implementation of the MacBook fan control daemon for Linux.

## Building
cargo build --release

## Installing
sudo make install  # From parent directory

## Development
See ../RUST_PORT.md for complete porting plan.
```

---

## Phase 2: Core Data Structures

### 2.1 Configuration Module (`config.rs`)
Mirror the C global state but use idiomatic Rust:

```rust
pub struct Config {
    // Temperature thresholds (Celsius)
    pub temp_avg_floor: f32,
    pub temp_avg_ceiling: f32,
    pub temp_tc0p_floor: f32,
    pub temp_tc0p_ceiling: f32,
    pub temp_tg0p_floor: f32,
    pub temp_tg0p_ceiling: f32,
    
    // Fan speed (RPM)
    pub fan_min: u32,
    pub fan_max: u32,  // Fixed at 6200
    
    // Logging
    pub log_level: u8,  // 0, 1, 2
    
    // Excluded sensors (sensor IDs)
    pub exclude: Vec<u32>,
}

impl Config {
    pub const DEFAULT_FAN_MAX: u32 = 6200;
    
    pub fn from_file(path: &str) -> Result<Self>;
    pub fn validate(&self) -> Result<()>;  // Clamp values, check floor < ceiling
}
```

**Key Points**:
- Parse `/etc/macfanctl.conf` line-by-line matching C parser behavior
- Handle `key: value` format with flexible whitespace
- Support both space and comma-separated `exclude` lists
- Warn on ill-formed lines (don't fail hard)
- Apply same clamping logic: `min(max_val, max(min_val, val))`

### 2.2 Sensor Module (`sensor.rs`)
```rust
pub struct Sensor {
    pub id: u32,              // 1-indexed sensor number
    pub excluded: bool,
    pub name: String,         // e.g., "TC0P", "TG0P"
    pub path: PathBuf,        // /sys/.../tempN_input
    pub value: f32,           // Celsius
}

pub struct SensorRegistry {
    sensors: Vec<Sensor>,
    tc0p: Option<usize>,     // Index of TC0P sensor
    tg0p: Option<usize>,     // Index of TG0P sensor
}

impl SensorRegistry {
    pub fn scan(base_path: &Path, exclude: &[u32]) -> Result<Self>;
    pub fn read_values(&mut self) -> Result<()>;
    pub fn average_temp(&self) -> f32;
}
```

### 2.3 Control Module (`control.rs`)
```rust
pub struct FanController {
    pub base_path: PathBuf,
    pub fan_count: u8,
    pub sensors: SensorRegistry,
    pub speed: u32,
    pub control_source: ControlSource,  // AVG, TC0P, TG0P
}

#[derive(Debug, Clone, Copy)]
pub enum ControlSource {
    None,
    Avg,
    TC0P,
    TG0P,
}

impl FanController {
    pub fn new(base_path: PathBuf, exclude: &[u32]) -> Result<Self>;
    pub fn adjust(&mut self, config: &Config) -> Result<()>;
    pub fn calc_fan_speed(&mut self, config: &Config) -> u32;
    pub fn set_fan_speed(&self, speed: u32) -> Result<()>;
}
```

---

## Phase 3: Core Functionality Implementation

### 3.1 Sysfs Interaction (`sysfs.rs`)
Implement safe wrappers for reading/writing sysfs files:

```rust
pub fn find_applesmc() -> Result<PathBuf> {
    // Search /sys/class/hwmon/hwmon*/device/name for "applesmc"
    // Return canonicalized base path
}

pub fn read_sysfs_int(path: &Path) -> Result<i32> {
    // Read file, trim whitespace, parse integer
}

pub fn write_sysfs_int(path: &Path, value: i32) -> Result<()> {
    // Write integer as string to sysfs file
}

pub fn count_fans(base_path: &Path) -> Result<u8> {
    // Check for fan1_min, fan2_min existence
}
```

**Critical**: Match C behavior exactly:
- Read `tempN_input` as millidegrees Celsius → divide by 1000.0
- Write to `fan1_min`, `fan2_min` (if present)
- Set `fan1_manual`, `fan2_manual` to `0` after each update

### 3.2 Fan Control Algorithm (`control.rs::calc_fan_speed`)
Port the exact C logic:

```rust
fn calc_fan_speed(&mut self, config: &Config) -> u32 {
    let fan_window = (config.fan_max - config.fan_min) as f32;
    let mut speed = config.fan_min as f32;
    self.control_source = ControlSource::None;
    
    // 1. Calculate from average temp
    let avg_temp = self.sensors.average_temp();
    let avg_window = config.temp_avg_ceiling - config.temp_avg_floor;
    let normalized = (avg_temp - config.temp_avg_floor) / avg_window;
    let avg_speed = normalized * fan_window;
    if avg_speed > speed {
        speed = avg_speed;
        self.control_source = ControlSource::Avg;
    }
    
    // 2. Calculate from TC0P
    if let Some(idx) = self.sensors.tc0p {
        let tc0p_temp = self.sensors.sensors[idx].value;
        let window = config.temp_tc0p_ceiling - config.temp_tc0p_floor;
        let normalized = (tc0p_temp - config.temp_tc0p_floor) / window;
        let tc0p_speed = normalized * fan_window;
        if tc0p_speed > speed {
            speed = tc0p_speed;
            self.control_source = ControlSource::TC0P;
        }
    }
    
    // 3. Calculate from TG0P (same pattern)
    // ...
    
    // 4. Clamp to [fan_min, fan_max]
    speed.max(config.fan_min as f32).min(config.fan_max as f32) as u32
}
```

### 3.3 Logging (`control.rs::logger`)
**Critical**: Match exact C output format for operational compatibility:

```rust
fn log_status(&self, config: &Config) {
    if config.log_level == 0 {
        return;
    }
    
    let avg = self.sensors.average_temp();
    let mark_avg = if matches!(self.control_source, ControlSource::Avg) { "*" } else { " " };
    
    print!("Speed: {}, {}AVG: {:.1}C", self.speed, mark_avg, avg);
    
    if let Some(idx) = self.sensors.tc0p {
        let mark = if matches!(self.control_source, ControlSource::TC0P) { "*" } else { " " };
        print!(", {}TC0P: {:.1}C", mark, self.sensors.sensors[idx].value);
    }
    
    // ... TG0P ...
    
    if config.log_level > 1 {
        print!(", Sensors: ");
        for sensor in &self.sensors.sensors {
            if !sensor.excluded {
                print!("{}:{:.0} ", sensor.name, sensor.value);
            }
        }
    }
    
    println!();
}
```

---

## Phase 4: Daemonization & Signal Handling

### 4.1 Daemonization (`main.rs`)
Use `nix` crate or manual fork/setsid:

```rust
fn daemonize(log_file: &str) -> Result<()> {
    use nix::unistd::{fork, setsid, ForkResult};
    
    // Check if already a daemon
    if nix::unistd::getppid().as_raw() == 1 {
        return Ok(());
    }
    
    match unsafe { fork() }? {
        ForkResult::Parent { .. } => std::process::exit(0),
        ForkResult::Child => {
            #[cfg(debug_assertions)]
            std::thread::sleep(Duration::from_secs(20));  // DEBUG equivalent
            
            setsid()?;
            nix::sys::stat::umask(Mode::from_bits(0o022).unwrap());
            
            // Redirect stdout/stderr to log file
            let log = OpenOptions::new()
                .create(true)
                .write(true)
                .truncate(true)
                .open(log_file)?;
            nix::unistd::dup2(log.as_raw_fd(), 1)?;  // stdout
            nix::unistd::close(0)?;  // stdin
            
            std::env::set_current_dir("/")?;
            Ok(())
        }
    }
}
```

### 4.2 PID File Locking
```rust
fn create_pid_file(path: &str) -> Result<File> {
    use nix::fcntl::{flock, FlockArg};
    
    let file = OpenOptions::new()
        .read(true)
        .write(true)
        .create(true)
        .mode(0o640)
        .open(path)?;
    
    flock(file.as_raw_fd(), FlockArg::LockExclusiveNonblock)
        .map_err(|_| anyhow!("Another instance is running"))?;
    
    file.set_len(0)?;
    writeln!(&file, "{}", std::process::id())?;
    
    Ok(file)  // Keep file open for lock duration
}
```

### 4.3 Signal Handling
```rust
use signal_hook::consts::signal::*;
use signal_hook::iterator::Signals;

fn setup_signals() -> Result<Signals> {
    let signals = Signals::new(&[SIGHUP, SIGTERM, SIGINT])?;
    Ok(signals)
}

// In main loop:
for signal in signals.pending() {
    match signal {
        SIGHUP => {
            config = Config::from_file(CFG_FILE)?;
            controller.rescan_sensors(&config.exclude)?;
        }
        SIGTERM | SIGINT => {
            running = false;
            break;
        }
        _ => {}
    }
}
```

---

## Phase 5: Main Loop & Integration

### 5.1 Main Function Structure
```rust
const PID_FILE: &str = "/var/run/macfanctld.pid";
const LOG_FILE: &str = "/var/log/macfanctl.log";
const CFG_FILE: &str = "/etc/macfanctl.conf";

fn main() -> Result<()> {
    let args: Vec<String> = std::env::args().collect();
    let foreground = args.contains(&"-f".to_string());
    
    if !foreground {
        daemonize(LOG_FILE)?;
    } else {
        println!("Running in foreground, log to stdout.");
    }
    
    let _pid_file = create_pid_file(PID_FILE)?;
    let signals = setup_signals()?;
    
    let mut config = Config::from_file(CFG_FILE)?;
    let base_path = sysfs::find_applesmc()?;
    let mut controller = FanController::new(base_path, &config.exclude)?;
    
    let mut running = true;
    while running {
        controller.adjust(&config)?;
        controller.log_status(&config);
        
        // Check for signals
        for sig in signals.pending() {
            match sig {
                SIGHUP => {
                    config = Config::from_file(CFG_FILE)?;
                    controller.rescan_sensors(&config.exclude)?;
                }
                SIGTERM | SIGINT => running = false,
                _ => {}
            }
        }
        
        std::thread::sleep(Duration::from_secs(5));
    }
    
    println!("Exiting.");
    std::fs::remove_file(PID_FILE).ok();
    
    Ok(())
}
```

---

## Phase 6: Testing & Validation

### 6.1 Unit Tests
- **Config parsing**: Test all valid/invalid configs, clamping, exclude list parsing
- **Fan calculation**: Test linear interpolation with known inputs
- **Sensor scanning**: Mock sysfs directories with different sensor counts
- **Clamping logic**: Verify `min(fan_max, max(fan_min, calculated))`

### 6.2 Integration Tests
- **Config reload (SIGHUP)**: Verify behavior matches C daemon
- **Sensor exclusion**: Test with various exclude lists
- **Multi-fan systems**: Test fan1 + fan2 control
- **Logging formats**: Compare output byte-for-byte with C version

### 6.3 Hardware Testing Protocol
⚠️ **Critical Safety Warnings**:
1. Always test in foreground mode first (`-f`)
2. Monitor temps with `tail -F /var/log/macfanctl.log`
3. Set conservative limits initially (low ceiling temps)
4. Compare output with C daemon running in parallel
5. Verify fan actually spins up when temps rise

```bash
# Test sequence:
sudo ./target/release/macfanctld-rs -f  # Foreground test
# In another terminal:
tail -F /var/log/macfanctl.log
# Monitor for 5-10 minutes, induce load (stress test)
# Verify fan speeds match expected behavior
```

---

## Phase 7: Build & Packaging

### 7.1 Makefile Integration
Add to parent `Makefile` (in `/home/kent/repos/kentwait/macfanctld/Makefile`):

```makefile
# Rust submodule targets
rust-build:
	cd rust && cargo build --release

rust-install: rust-build
	install -D -m 755 rust/target/release/macfanctld $(SBIN_DIR)/macfanctld-rs
	# Config file still shared with C version
	test -f $(ETC_DIR)/macfanctl.conf || cp macfanctl.conf $(ETC_DIR)

rust-clean:
	cd rust && cargo clean

rust-uninstall:
	rm -f $(SBIN_DIR)/macfanctld-rs

# Update submodule
rust-update:
	git submodule update --remote rust

.PHONY: rust-build rust-install rust-clean rust-uninstall rust-update
```

**Note**: The Rust binary is named `macfanctld` (not `macfanctld-rs`) within the submodule's `target/release/`, but installed as `macfanctld-rs` to avoid conflicts during development.

### 7.2 Debian Packaging

**Option A: Submodule handles its own packaging** (recommended)
Create `rust/debian/` directory within the submodule:

```
rust/
├── debian/
│   ├── control           # Package: macfanctld-rs
│   ├── rules
│   ├── changelog
│   ├── compat
│   ├── copyright
│   └── install
```

**`rust/debian/control`**:
```
Source: macfanctld-rs
Section: admin
Priority: optional
Maintainer: Mikael Strom <mikael@sesamiq.com>
Build-Depends: debhelper (>= 9), cargo, rustc (>= 1.70)
Standards-Version: 4.5.0

Package: macfanctld-rs
Architecture: amd64
Depends: ${shlibs:Depends}, ${misc:Depends}, applesmc-dkms
Conflicts: macfanctld
Replaces: macfanctld
Description: Fan control daemon for MacBook (Rust implementation)
 Modern Rust rewrite of macfanctld with identical behavior.
```

**Option B: Parent repo packages both**
Keep C packaging in `debian/`, add Rust variant in `debian-rs/` in parent repo.

### 7.3 Init Script Compatibility
- Reuse existing `init.d` script (change binary path only)
- Ensure same PID file location
- Support same command-line flags (`-f`)

---

## Phase 8: Documentation & Migration

### 8.1 Documentation Updates

**In submodule (`rust/README.md`)**:
```markdown
# macfanctld - Rust Implementation

Rust port of the MacBook fan control daemon for Linux. This is a behavioral drop-in
replacement for the original C implementation.

## Requirements
- Rust 1.70+ (MSRV)
- Linux kernel with applesmc module (applesmc-dkms)
- Root privileges (writes to /sys/class/hwmon)

## Building
cargo build --release

## Installation
From parent repository:
sudo make rust-install

## Configuration
Uses the same `/etc/macfanctl.conf` as the C version. See parent repo's
`macfanctld.1` man page for configuration details.

## Development
See `../RUST_PORT.md` for complete porting plan and architecture.

## Testing
cargo test
cargo test --test integration_tests

## License
GPL-3.0-or-later (same as original C implementation)
```

**In parent `README`**:
Add section:
```markdown
## Rust Port
A Rust implementation is available in the `rust/` git submodule. It provides
identical behavior with improved safety guarantees. See `RUST_PORT.md` for details.
```

**Man page**: Keep `macfanctld.1` identical (it's implementation-agnostic)

### 8.2 Migration Guide

**For parent repository maintainers**:
```bash
# Add the Rust submodule (one-time setup)
git submodule add https://github.com/<user>/macfanctld-rs rust
git submodule update --init --recursive

# Update submodule to latest
git submodule update --remote rust
git add rust
git commit -m "Update Rust port to latest version"
```

**For end users migrating from C to Rust**:
```markdown
# Migrating from C to Rust

1. Stop the C daemon:
   sudo systemctl stop macfanctld
   # or: sudo /etc/init.d/macfanctld stop

2. Build and install Rust version:
   cd /path/to/macfanctld
   git submodule update --init --recursive
   sudo make rust-install
   
3. Update init script:
   sudo vim /etc/init.d/macfanctld
   # Change: DAEMON=/usr/sbin/macfanctld
   # To:     DAEMON=/usr/sbin/macfanctld-rs
   
4. Start Rust daemon:
   sudo systemctl start macfanctld
   # or: sudo /etc/init.d/macfanctld start
   
5. Verify logs match expected format:
   tail -F /var/log/macfanctl.log
   # Should see: Speed: XXXX, AVG: XX.XC, ...

6. Optional: Remove C version
   sudo make uninstall  # Remove C binary only
```

**Standalone installation (from submodule repo)**:
```bash
git clone https://github.com/<user>/macfanctld-rs
cd macfanctld-rs
cargo build --release
sudo install -m 755 target/release/macfanctld /usr/sbin/macfanctld-rs

# Copy config if not present
sudo install -m 644 ../macfanctl.conf /etc/macfanctl.conf
```

### 8.3 Feature Parity Checklist
- [ ] Parse `/etc/macfanctl.conf` identically
- [ ] Find applesmc in `/sys/class/hwmon`
- [ ] Count fans (1 or 2)
- [ ] Scan and label sensors
- [ ] Exclude sensors by ID
- [ ] Calculate average temp from non-excluded sensors
- [ ] Apply linear interpolation per source (AVG, TC0P, TG0P)
- [ ] Take maximum of three calculated speeds
- [ ] Clamp to `[fan_min, 6200]`
- [ ] Write to `fan1_min`, `fan2_min`
- [ ] Reset `fan1_manual`, `fan2_manual` to 0
- [ ] Log with exact format (speed, temps, control marker)
- [ ] Support log levels 0, 1, 2
- [ ] Daemonize (fork, setsid, redirect stdout)
- [ ] PID file locking at `/var/run/macfanctld.pid`
- [ ] Log to `/var/log/macfanctl.log` (daemon) or stdout (foreground)
- [ ] Handle SIGHUP (reload config + rescan)
- [ ] Handle SIGTERM/SIGINT (clean exit)
- [ ] 5-second control loop cadence
- [ ] `-f` foreground flag

---

## Phase 9: Future Enhancements (Post-Parity)

Only after achieving exact behavioral parity with C:

### 9.1 Rust-Specific Improvements
- **Structured logging**: Use `serde_json` for optional JSON log format (new flag)
- **Systemd integration**: Native Type=notify support
- **Better error messages**: Leverage Rust's error types for diagnostics
- **Configuration validation**: Stronger type checking at parse time

### 9.2 New Features (Breaking Changes)
- **Multiple control policies**: PID control, hysteresis bands
- **Web API**: REST endpoint for metrics (requires new dependency)
- **Dynamic sensor discovery**: Auto-detect TC0P/TG0P equivalents on new hardware
- **Config hot-reload**: Watch `/etc/macfanctl.conf` with inotify

### 9.3 Performance Optimizations
- **Zero-copy sysfs reads**: Use memory-mapped files
- **Batch sensor reads**: Read multiple sensors in parallel
- **Reduced allocations**: Use `&str` instead of `String` where possible

---

## Risk Mitigation

### Hardware Safety
- **Risk**: Incorrect fan control → overheating
- **Mitigation**: Extensive validation against C daemon output, conservative defaults, foreground testing

### Behavior Drift
- **Risk**: Subtle differences in control algorithm
- **Mitigation**: Property-based testing with identical inputs, log diff comparison

### Dependency Bloat
- **Risk**: Large binary size from dependencies
- **Mitigation**: Minimal dependencies, consider `musl` static builds

### Platform Compatibility
- **Risk**: Works on developer's MacBook but not others
- **Mitigation**: Test on multiple MacBook models, document tested hardware

---

## Success Criteria

The Rust port is considered complete when:

1. ✅ Binary drop-in replacement: Swap `macfanctld` → `macfanctld-rs` with zero config changes
2. ✅ Identical logs: Log output matches C version byte-for-byte (whitespace, precision, format)
3. ✅ Same control behavior: Fan speeds converge to same values as C daemon given same temps
4. ✅ Signal handling works: SIGHUP reloads config, SIGTERM exits cleanly
5. ✅ PID locking prevents duplicates
6. ✅ Passes 24-hour stress test without crashes or thermal issues
7. ✅ Builds on stable Rust without warnings
8. ✅ All unit and integration tests pass
9. ✅ Documentation complete and accurate
10. ✅ Packaged as `.deb` with proper dependencies

---

## Timeline Estimate

- **Phase 0** (Git submodule setup): 0.5 days
- **Phase 1-2** (Project setup + data structures): 2-3 days
- **Phase 3** (Core functionality): 4-5 days
- **Phase 4** (Daemonization): 2-3 days
- **Phase 5** (Integration): 1-2 days
- **Phase 6** (Testing): 3-4 days
- **Phase 7** (Packaging): 1-2 days
- **Phase 8** (Documentation): 1-2 days

**Total**: ~15-22 days of focused development

**Submodule-specific considerations**:
- Initial submodule setup adds minimal overhead
- Allows parallel C/Rust development without conflicts
- Can publish Rust version to crates.io independently
- Separate git history for Rust implementation

---

## References

- C source code (canonical behavior reference)
- `macfanctld.1` man page (user-facing contract)
- `/etc/macfanctl.conf` format specification
- applesmc kernel module documentation
- Rust `nix` crate docs (daemonization)
- Rust `signal-hook` docs (signal safety)

---

## Notes

- Keep C daemon as **source of truth** for behavior during porting
- Prioritize **correctness over performance**—this is a 5-second control loop
- **No clever abstractions** until feature parity is achieved
- Test on real hardware early and often
- When in doubt, match C behavior exactly (even quirks)

---

## Phase 0: Git Submodule Setup (Prerequisite)

Before starting Phase 1, set up the Rust submodule structure:

```bash
# From the parent repository root
cd /home/kent/repos/kentwait/macfanctld

# Option A: Link to existing Rust repo (if already created)
git submodule add <rust-repo-url> rust

# Option B: Create new repo for the submodule
mkdir -p ../macfanctld-rs
cd ../macfanctld-rs
git init
cargo init --name macfanctld
git add .
git commit -m "Initial Rust project structure"

# Create remote repo (GitHub, GitLab, etc.)
git remote add origin <new-rust-repo-url>
git push -u origin main

# Back to parent, add as submodule
cd /home/kent/repos/kentwait/macfanctld
git submodule add <new-rust-repo-url> rust
git commit -m "Add Rust implementation as submodule"
```

**Key submodule commands**:
```bash
# Clone parent with submodule
git clone --recursive <parent-repo-url>

# Update existing clone to include submodule
git submodule update --init --recursive

# Pull latest changes in submodule
cd rust
git pull origin main
cd ..
git add rust
git commit -m "Update Rust submodule"

# Work in submodule
cd rust
# ... make changes ...
git add .
git commit -m "Implement feature X"
git push origin main
cd ..
git add rust  # Update parent's submodule reference
git commit -m "Update Rust submodule reference"
```

---

**Last Updated**: October 31, 2025  
**Status**: Planning phase  
**Next Step**: Phase 0 - Set up git submodule, then Phase 1 - Create Cargo project inside `/rust`
