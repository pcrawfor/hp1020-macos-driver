# HP LaserJet 1020 on Apple Silicon macOS 26.5

## Installation, Security Review, Troubleshooting, and Working Solution

**Date:** November 14, 2026
**Computer:** Apple Silicon Mac Studio
**Operating system:** macOS 26.5
**Printer:** HP LaserJet 1020 (USB)
**Printer serial:** `XXXXXXX`
**Status:** Successfully installed and printing

## 1. Summary

The HP LaserJet 1020 is a legacy USB laser printer that requires the host computer to upload firmware after each printer power cycle.

Modern macOS versions no longer provide straightforward native driver support for this printer.

We initially tried a third-party Gutenprint/foo2zjs package, but print jobs stalled with USB communication timeouts. We removed that package and switched to a native Rust implementation.

The successful solution was:

1. Use the open-source Rust-based HP 1020 driver.
2. Review its source code and dependencies for security concerns.
3. Install Rust using Homebrew.
4. Update a dependency with a security advisory.
5. Compile and test the driver locally for Apple Silicon.
6. Verify direct USB communication.
7. Download, verify, and upload the printer firmware.
8. Modify the installer to build ARM64-only binaries.
9. Build and install a local macOS installer package.
10. Configure the printer through CUPS and successfully print a test page.

**Final result:** The HP LaserJet 1020 works on the Mac Studio using native ARM64 binaries, without Rosetta.

---

## 2. Initial attempt: Apple Silicon Printer Drivers

### Repository

https://github.com/theshiver/apple-silicon-printer-drivers

**Package:** `AppleSilicon-Printer-Drivers-2.0.2.pkg`

This project bundles Gutenprint, foo2zjs, and other legacy printer drivers.

### Security checks

Before installation, we verified the installer:

```bash
spctl -a -vv -t install AppleSilicon-Printer-Drivers-2.0.2.pkg

pkgutil --check-signature AppleSilicon-Printer-Drivers-2.0.2.pkg
```

Results included:

- Apple-notarized installer
- Valid Developer ID Installer signature
- Developer: Can Cetin
- Team ID: `S2RH54MMT3`

We also expanded the package and reviewed the installation scripts and bundled components.

### Installation result

The package successfully installed and created a printer queue:

```text
Hewlett_Packard_HP_LaserJet_1020
```

Its CUPS configuration identified the printer as:

```text
HP LaserJet 1020 - Apple Silicon
```

The filter used was:

```text
/Library/Printers/Gutenprint/libexec/rastertofoo-foo2zjs
```

### Failure

Jobs stalled at:

```text
Sending data to printer
```

The CUPS logs repeatedly reported:

```text
rastertofoo: firmware state -1, nothing to send
Got USB transaction timeout during read
```

We also attempted to send firmware directly through the raw print queue:

```bash
lp -d Hewlett_Packard_HP_LaserJet_1020 -o raw \
  /Library/Printers/Gutenprint/share/foo2zjs/firmware/sihp1020.dl
```

This also stalled with USB transaction timeouts.

**Conclusion:** The printer was detected, but communication through the existing CUPS USB backend was unsuccessful.

---

## 3. Removing the failed driver

We used the project's uninstall script, which removed the Gutenprint components, associated printer queues, and package receipt.

After uninstalling, verification showed:

```bash
ls -ld /Library/Printers/Gutenprint
```

Output:

```text
No such file or directory
```

And:

```bash
lpstat -t
```

Output included:

```text
scheduler is running
no system default destination
lpstat: No destinations added.
```

The previous installation was successfully removed.

---

## 4. Selecting the replacement driver

### Chosen repository

https://github.com/UtkarshChandel/hp1020-macos-driver

**Project:** Native macOS CUPS driver for HP LaserJet 1018/1020/1020 Plus.

**Version:** `1.0.0`

**Repository commit reviewed:**

```text
ab77b01db5ed3e9baf46e353fb8928ab1b518a08
```

### Why this implementation

Unlike the previous package, this driver includes its own native Rust USB backend.

Its architecture is:

```text
macOS application
       |
       v
macOS CUPS printing system
       |
       v
rastertozjs (Rust print filter)
       |
       v
hp1020 (Rust USB backend)
       |
       v
USB / IOKit
       |
       v
HP LaserJet 1020
```

The implementation contains two main executables:

| Component | Purpose |
|---|---|
| `rastertozjs` | Converts CUPS raster output into HP ZjStream print data |
| `hp1020` | Communicates directly with the printer over USB and uploads firmware |

It uses `nusb` for USB access through macOS IOKit.

This provides a different communication path from the previous driver.

---

## 5. Security review

Before installation, we inspected the repository's implementation and packaging approach.

### Positive findings

- Rust source code is publicly available.
- Small dependency footprint.
- No obvious telemetry or credential-access behavior in the reviewed implementation.
- No background service installed.
- Firmware is validated against a hardcoded SHA-256 checksum.
- Binaries can be compiled locally rather than downloaded from an unverified release.
- CUPS integration uses conventional printer filters and a custom backend.

### Remaining security considerations

- The USB backend runs with elevated privileges under CUPS.
- The raster parser has input-validation and resource-limit concerns that were identified but not patched.
- The installation script downloads and extracts firmware with root privileges.
- The project has limited independent scrutiny.
- The driver is not distributed as an Apple-notarized package.

We chose to compile from source and review its dependencies rather than trust a prebuilt release binary.

**Important:** A clean dependency audit and successful local build do not constitute a complete security audit.

---

## 6. Installing Rust

Homebrew was used to install Rust:

```bash
brew install rust
```

Verify:

```bash
rustc --version
cargo --version
```

Installed versions:

```text
rustc 1.100.0 (Homebrew)
cargo 1.100.0 (Homebrew)
```

Homebrew supplies the native Apple Silicon Rust toolchain.

For this installation, a separate `rustup` installation was unnecessary.

---

## 7. Obtaining the driver

```bash
cd ~/Downloads

git clone https://github.com/UtkarshChandel/hp1020-macos-driver.git

cd hp1020-macos-driver
```

Verify the checked-out commit:

```bash
git rev-parse HEAD
```

The version used for this installation was:

```text
ab77b01db5ed3e9baf46e353fb8928ab1b518a08
```

For a reproducible repeat of this process, explicitly check out that revision:

```bash
git checkout ab77b01db5ed3e9baf46e353fb8928ab1b518a08
```

Note that the original working directory was subsequently modified locally to fix dependency and packaging issues.

---

## 8. Rust dependencies

The project's `Cargo.toml` declared:

```toml
[package]
name = "hp1020"
version = "1.0.0"
edition = "2021"

[dependencies]
libc = "0.2.191"
nusb = "0.2.5"
```

The lockfile resolved:

```text
libc 0.2.191
nusb 0.2.8
```

### Inspect dependencies

```bash
cargo metadata --locked --format-version 1 --no-deps
```

Then:

```bash
cargo tree --locked --target aarch64-apple-darwin
```

The Apple Silicon dependency tree contained:

```text
hp1020
├── libc
└── nusb
    ├── core-foundation
    ├── core-foundation-sys
    ├── futures-core
    ├── io-kit-sys
    │   └── mach2
    ├── log
    └── slab
```

The application and its ARM64 dependencies formed a relatively small dependency tree.

---

## 9. Dependency security audit

Install the RustSec advisory scanner:

```bash
brew install cargo-audit
```

Run:

```bash
cargo audit
```

### Initial result

The audit reported an unsoundness advisory:

```text
Crate: rustix
Version: 1.1.5
Advisory: RUSTSEC-2026-0486
```

The issue involved extended-attribute functions and zero-length buffers.

`rustix` was not part of the macOS ARM64 dependency tree, but we updated it anyway.

### Fix

```bash
cargo update -p rustix --precise 1.1.6
```

Recheck:

```bash
cargo audit
```

### Final result

```text
Scanning Cargo.lock for vulnerabilities (35 crate dependencies)
```

No advisories were reported.

**Result:** Dependency audit passed.

The lockfile was modified locally. Keep the updated `Cargo.lock` when repeating the build.

---

## 10. Compiling and testing

Run the project's tests:

```bash
cargo test --locked
```

Results:

```text
running 1 test
test matches_foo2zjs ... ok

test result: ok. 1 passed; 0 failed
```

The test compares the Rust output against expected foo2zjs-compatible output.

Build optimized binaries:

```bash
cargo build --release --locked
```

The build completed successfully.

### Verify CPU architecture

```bash
file target/release/hp1020 target/release/rastertozjs
```

Output:

```text
target/release/hp1020:
Mach-O 64-bit executable arm64

target/release/rastertozjs:
Mach-O 64-bit executable arm64
```

### Inspect dynamic library dependencies

```bash
otool -L target/release/hp1020

otool -L target/release/rastertozjs
```

The USB backend linked to Apple's system libraries, including:

- IOKit
- CoreFoundation
- libiconv
- libSystem

The print filter linked to libiconv and libSystem.

No unexpected third-party dynamic libraries were found.

---

## 11. Testing USB communication before installation

We tested the locally compiled executable directly:

```bash
target/release/hp1020 status
```

### Initial issue

One attempt produced:

```text
hp1020: transfer was cancelled
```

After another attempt, communication succeeded.

### Successful output

```text
HP LaserJet 1020 serial XXXXXXX

device id:
MFG:Hewlett-Packard;
MDL:HP LaserJet 1020;
CMD:ACL;
CLS:PRINTER;
DES:HP LaserJet 1020;

firmware: not loaded
```

This established that the new Rust backend could communicate with the printer directly.

It also confirmed that the printer needed firmware.

---

## 12. Firmware download and verification

The HP LaserJet 1020 does not retain its operating firmware when powered off.

The firmware must therefore be uploaded again after a power cycle.

The repository provides:

```text
packaging/fetch-firmware.sh
```

This script downloads firmware from OpenPrinting's HP HPLIP plugin repository.

For the initial test, we ran the script without root privileges and used a user-owned directory.

### Download firmware

```bash
mkdir -p ~/Downloads/hp1020-firmware

bash packaging/fetch-firmware.sh \
  ~/Downloads/hp1020-firmware
```

The script downloaded firmware for the HP 1018 and HP 1020 and checked their SHA-256 hashes.

### Verify the HP 1020 firmware

```bash
shasum -a 256 \
  ~/Downloads/hp1020-firmware/hp_laserjet_1020.fw
```

Expected SHA-256:

```text
9a6d03c858d9cf64ba86fdbe6cf0beec1297d2e45e606162b19f3857efae4ff4
```

The checksum is defined in the project's firmware-fetching script.

### Upload firmware

With the printer powered on and connected:

```bash
target/release/hp1020 load-firmware \
  ~/Downloads/hp1020-firmware/hp_laserjet_1020.fw
```

### Successful result

```text
DEBUG: hp1020: device id: MFG:Hewlett-Packard;MDL:HP LaserJet 1020;...

STATE: +connecting-to-device

INFO: hp1020: Loading printer firmware
~/Downloads/hp1020-firmware/hp_laserjet_1020.fw
(128999 bytes)

INFO: hp1020: Waiting for the printer to start its firmware

INFO: hp1020: Firmware running (20080222)

STATE: -connecting-to-device
```

Verify again:

```bash
target/release/hp1020 status
```

Output:

```text
HP LaserJet 1020 serial XXXXXXX

device id:
MFG:Hewlett-Packard;
MDL:HP LaserJet 1020;
CMD:ACL;
CLS:PRINTER;
DES:HP LaserJet 1020;
FWVER:20080222;

firmware: running (20080222)
```

**This was the key breakthrough.** The new backend could both communicate with the printer and successfully upload its firmware.

---

## 13. Installer compatibility problem

The repository provides an installation wrapper:

```bash
./install.sh
```

The wrapper invokes:

```bash
packaging/build-pkg.sh
```

The original packaging script builds universal macOS binaries containing both:

- ARM64 (Apple Silicon)
- x86_64 (Intel)

Our Rust installation from Homebrew only had the native ARM64 target available.

### Original failure

```text
error[E0463]: can't find crate for `core`

the `x86_64-apple-darwin` target may not be installed
```

Because the printer only needed to work on this Apple Silicon machine, we modified the packaging script to build ARM64-only binaries.

---

## 14. Modifying the installer for ARM64

Make a backup:

```bash
cp packaging/build-pkg.sh packaging/build-pkg.sh.bak
```

Edit:

```bash
nano packaging/build-pkg.sh
```

### Change 1: Compile only the native architecture

Original:

```bash
for t in aarch64-apple-darwin x86_64-apple-darwin; do
    cargo build --release --quiet --target "$t"
done
```

Replacement:

```bash
cargo build --release --quiet
```

This builds native ARM64 binaries using the installed Homebrew Rust toolchain.

### Change 2: Remove the universal binary assembly

Original:

```bash
for bin in rastertozjs hp1020; do
    lipo -create -output "$WORK/$bin" \
        "target/aarch64-apple-darwin/release/$bin" \
        "target/x86_64-apple-darwin/release/$bin"
done
```

Replacement:

```bash
for bin in rastertozjs hp1020; do
    cp "target/release/$bin" "$WORK/$bin"
done
```

The `lipo` command is unnecessary when building for only one architecture.

**Result:** The package contains native ARM64 executables instead of universal executables.

The package metadata may still advertise both architectures; that does not change the actual executable architecture. The resulting package is intended for Apple Silicon only.

---

## 15. Installing the driver

With the ARM64 packaging changes in place:

```bash
./install.sh
```

The script:

1. Compiled the Rust project.
2. Built a macOS installer package.
3. Invoked macOS Installer with administrator privileges.
4. Installed the CUPS filter and USB backend.
5. Downloaded and installed printer firmware.
6. Created the HP LaserJet 1020 printer queue.

### Installation output

```text
==> Building the installer package...

==> Installing dist/HP-LaserJet-1020-Driver-1.0.0.pkg
(needs your password)...

installer: Package name is HP LaserJet 1020 Driver 1.0.0

installer: Installing at base path /

installer: The install was successful.

Firmware installed in /Library/Printers/hp1020/firmware

lpadmin: Printer drivers are deprecated and will stop
working in a future version of CUPS.

queue HP_LaserJet_1020 ->
hp1020://HP/LaserJet%201020?serial=XXXXXXX

Done.
```

The driver-deprecation message is a CUPS warning rather than an installation error.

### Installed components

The package installs its components at locations including:

```text
/usr/libexec/cups/backend/hp1020

/usr/libexec/cups/filter/rastertozjs

/Library/Printers/hp1020/

/Library/Printers/hp1020/firmware/

/Library/Printers/PPDs/Contents/Resources/HP LaserJet 1020 Native.ppd
```

The package's installer uses elevated privileges and its post-install script downloads the firmware. We had reviewed these operations, but did not remove the privileged firmware download step from the installer before running it.

---

## 16. Verify the installed printer

Check the CUPS configuration:

```bash
lpstat -t
```

Successful output:

```text
scheduler is running

no system default destination

device for HP_LaserJet_1020:
hp1020://HP/LaserJet%201020?serial=XXXXXXX

HP_LaserJet_1020 accepting requests

printer HP_LaserJet_1020 is idle. enabled
```

This confirms the new USB backend has been registered and the queue is active.

### Test printing

```bash
echo "HP LaserJet 1020 test print" | \
  lp -d HP_LaserJet_1020
```

**Result: Test page printed successfully.**

The printer is now usable from macOS applications through the normal Print dialog.

### Optional: Set default printer

```bash
lpoptions -d HP_LaserJet_1020
```

The original installation did not automatically set a system default printer.

---

## 17. Useful maintenance commands

### Check printer status

```bash
lpstat -t
```

### Check firmware state directly

From the repository directory:

```bash
target/release/hp1020 status
```

### Upload firmware manually

```bash
target/release/hp1020 load-firmware \
  ~/Downloads/hp1020-firmware/hp_laserjet_1020.fw
```

For normal printing, the installed USB backend should handle firmware loading automatically when needed. We confirmed manual firmware loading, but did not separately test automatic loading after a printer power cycle.

### Print a test page

```bash
echo "HP LaserJet 1020 test" | \
  lp -d HP_LaserJet_1020
```

### Inspect completed and queued jobs

```bash
lpstat -W all -o HP_LaserJet_1020
```

### Inspect CUPS errors

```bash
sudo tail -n 100 /var/log/cups/error_log
```

### Check installed package information

```bash
pkgutil --pkg-info io.github.utkarshchandel.hp1020-driver
```

### Remove printer queue

```bash
sudo lpadmin -x HP_LaserJet_1020
```

This removes the configured printer queue, not the installed driver files.

For complete removal, review the project's `packaging/uninstall.command` script before executing it with elevated privileges.

---

## 18. Security conclusions

### What we verified

- Inspected the Rust source and installation scripts.
- Confirmed the project has a small dependency tree.
- Updated a dependency affected by a RustSec advisory.
- Confirmed `cargo audit` reported no advisories after the update.
- Compiled the executables locally.
- Confirmed they were native ARM64 executables.
- Checked dynamically linked libraries.
- Ran the project's test suite successfully.
- Verified the printer's firmware checksum.
- Tested direct USB communication without administrator privileges.
- Successfully uploaded firmware without modifying CUPS.
- Built a local installer package from source.

### What remains unverified

- Comprehensive fuzz testing and security testing of the raster parser.
- Least-privilege execution of the CUPS backend.
- Independent audit of all third-party Rust dependencies.
- Automatic firmware reloading after a printer power cycle.
- Compatibility with future macOS and CUPS versions.

### Overall assessment

The installation is preferable to blindly installing an unverified third-party binary package because the relevant driver was compiled locally from reviewed source and its dependencies were audited.

However, the package still installs a privileged USB backend and a raster filter that processes print-job data. It should be treated as third-party system software rather than fully trusted Apple software.

---

## 19. Quick installation reference for another Apple Silicon Mac

This summarizes the successful process. It assumes macOS, Apple Silicon, Homebrew, and a compatible HP LaserJet 1020 printer.

### Install tools

```bash
brew install rust cargo-audit
```

### Clone and pin repository

```bash
git clone \
  https://github.com/UtkarshChandel/hp1020-macos-driver.git

cd hp1020-macos-driver

git checkout ab77b01db5ed3e9baf46e353fb8928ab1b518a08
```

### Update affected dependency

```bash
cargo update -p rustix --precise 1.1.6
```

### Audit and test

```bash
cargo audit

cargo test --locked

cargo build --release --locked
```

### Modify packaging

Edit `packaging/build-pkg.sh` with the two ARM64-only changes documented in section 14.

### Check printer communication

```bash
target/release/hp1020 status
```

### Verify firmware and test upload

```bash
mkdir -p ~/Downloads/hp1020-firmware

bash packaging/fetch-firmware.sh \
  ~/Downloads/hp1020-firmware

shasum -a 256 \
  ~/Downloads/hp1020-firmware/hp_laserjet_1020.fw

target/release/hp1020 load-firmware \
  ~/Downloads/hp1020-firmware/hp_laserjet_1020.fw
```

### Install

After reviewing the privileged installation steps:

```bash
./install.sh
```

### Verify

```bash
lpstat -t

echo "HP LaserJet 1020 test print" | \
  lp -d HP_LaserJet_1020
```

---

## 20. Final result and key takeaway

**Working configuration:**

| Component | Value |
|---|---|
| Computer | Apple Silicon Mac Studio |
| macOS | 26.5 |
| Printer | HP LaserJet 1020 |
| Driver | hp1020-macos-driver 1.0.0 |
| Source revision | `ab77b01db5ed3e9baf46e353fb8928ab1b518a08` |
| Architecture | ARM64 |
| USB implementation | Rust `nusb` / Apple IOKit |
| Print format | HP ZjStream |
| Firmware version | `20080222` |
| Queue | `HP_LaserJet_1020` |
| Printer URI | `hp1020://HP/LaserJet%201020?serial=XXXXXXX` |
| Rosetta required | No |
| Printing | **Confirmed working** |

The key was switching from the legacy foo2zjs/CUPS USB communication path to a native Rust USB backend.

The replacement driver successfully identified the printer, uploaded its firmware, and completed a print job.

The resulting installation uses locally compiled Apple Silicon binaries and works through macOS's standard CUPS printing system.