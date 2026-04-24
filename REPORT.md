# eSim 2.5 Installation on Ubuntu 25.x — Bug Report & Fixes

**Author:** Ranjan Mishra  
**Date:** April 2026  
**Environment:** Ubuntu 25.10 (Questing) via GitHub Codespaces  
**Repository:** [https://github.com/LLawliet2028/eSim/tree/master]  
**Branch:** `master` (installer scripts cherry-picked from `installers` branch)

---

## Overview

This report documents the bugs encountered while attempting to install eSim 2.5 on Ubuntu 25.04 and above. Testing was performed on Ubuntu 25.10 ("questing") running inside a GitHub Codespace. A total of **11 bugs** were identified, with **9 bugs fixed**. Each bug is documented with its root cause, error message, and the fix applied.

---

## Environment Setup

- **OS:** Ubuntu 25.10 "Questing" (via GitHub Codespaces with `ubuntu:25.10` base image)
- **Python:** 3.13.7
- **KiCad (native):** 9.0.3
- **Verilator (native):** 5.032
- **Shell:** Bash 5.x

### How to Reproduce

```bash
git clone https://github.com/FOSSEE/eSim.git
cd eSim
git checkout master
# Copy installer scripts from installers branch
git checkout installers -- Ubuntu/
git checkout master
cd /path/to/eSim
./Ubuntu/install-eSim.sh --install
```

---

## Bug 1 — Broken Version Detection in `install-eSim.sh`

### Severity: 🔴 Critical (blocks entire installation)

### Location
`Ubuntu/install-eSim.sh` — `get_ubuntu_version()` function

### Error
```
Detected Ubuntu Version: 
Unsupported Ubuntu version: 25.10 ()
```

### Root Cause
The `FULL_VERSION` variable was set using:
```bash
FULL_VERSION=$(lsb_release -d | grep -oP '\d+\.\d+\.\d+')
```
The `lsb_release -d` command returns a description string like `Ubuntu 25.10`, which does not contain a three-part version number (`X.Y.Z`). The regex `\d+\.\d+\.\d+` finds no match and returns an empty string. Meanwhile, `VERSION_ID` (correctly parsed from `/etc/os-release`) was never used.

### Fix Applied
```bash
# Before
FULL_VERSION=$(lsb_release -d | grep -oP '\d+\.\d+\.\d+')

# After
FULL_VERSION=$VERSION_ID
```

---

## Bug 2 — No Fallback for Ubuntu 25.x in `install-eSim.sh`

### Severity: 🔴 Critical (blocks entire installation)

### Location
`Ubuntu/install-eSim.sh` — `run_version_script()` function

### Error
```
Unsupported Ubuntu version: 25.10 ()
Aborting Installation...
```

### Root Cause
The `case` statement only handled versions `22.04`, `23.04`, and `24.04`. Any other version (including 25.04 and 25.10) hit the `*` wildcard which explicitly called `exit 1`. There was no fallback to the most recent available installer script.

### Fix Applied
```bash
# Before
*)
    echo "Unsupported Ubuntu version: $VERSION_ID ($FULL_VERSION)"
    exit 1
    ;;

# After
*)
    echo "Using fallback installer for newer Ubuntu version: $VERSION_ID"
    SCRIPT="$SCRIPT_DIR/install-eSim-24.04.sh"
    ;;
```

---

## Bug 3 — KiCad 6.0 PPA Does Not Support Ubuntu 25.x

### Severity: 🔴 Critical (blocks KiCad installation)

### Location
`Ubuntu/install-eSim-scripts/install-eSim-24.04.sh` — `installKiCad()` function, lines ~134–180

### Error
```
Err: https://ppa.launchpadcontent.net/kicad/kicad-6.0-releases/ubuntu questing Release
  404  Not Found
E: The repository 'https://ppa.launchpadcontent.net/kicad/kicad-6.0-releases/ubuntu questing Release'
   does not have a Release file.
Error! Kindly resolve above error(s) and try again.
Aborting Installation...
```

### Root Cause
The script's `else` branch (triggered for any Ubuntu version that is not 24.04) assigned `kicad/kicad-6.0-releases` as the PPA. The KiCad 6.0 PPA on Launchpad does not have a release for Ubuntu 25.10 ("questing"), resulting in a 404. Additionally, the version comparison used `bc` for floating-point comparison, but `bc` is not installed by default on Ubuntu 25.10, causing the condition to silently fail.

**Secondary issue:** `bc` not available:
```
/install-eSim-24.04.sh: line 165: bc: command not found
```

### Fix Applied
Added an `elif` branch for Ubuntu ≥ 25 that skips the PPA entirely, since Ubuntu 25.x ships KiCad 9.0.3 natively in its official repositories. Used pure bash integer comparison instead of `bc`:

```bash
# After fix
elif [[ "${ubuntu_version%%.*}" -ge 25 ]]; then
    echo "Ubuntu $ubuntu_version detected. KiCad available natively — skipping PPA."
    kicadppa=""
else
    kicadppa="kicad/kicad-6.0-releases"
fi

# Add PPA only if needed (not required for Ubuntu 25.04+)
if [[ -n "$kicadppa" ]]; then
    if ! grep -q "^deb .*${kicadppa}" /etc/apt/sources.list /etc/apt/sources.list.d/* 2>/dev/null; then
        echo "Adding KiCad PPA to local apt repository: $kicadppa"
        sudo add-apt-repository -y "ppa:$kicadppa"
        sudo apt-get update
    else
        echo "KiCad PPA is already present in sources."
    fi
fi
```

---

## Bug 4 — Missing `install -y` in `apt-get xz-utils`

### Severity: 🟡 Medium (blocks volare/SKY130 installation)

### Location
`Ubuntu/install-eSim-scripts/install-eSim-24.04.sh` — `installDependency()` function, line ~257

### Error
```
E: Invalid operation xz-utils
Error! Kindly resolve above error(s) and try again.
Aborting Installation...
```

### Root Cause
The command was written as:
```bash
sudo apt-get xz-utils
```
This is syntactically incorrect. `apt-get` requires a subcommand (`install`, `remove`, etc.) before the package name.

### Fix Applied
```bash
# Before
sudo apt-get xz-utils

# After
sudo apt-get install -y xz-utils
```

---

## Bug 5 — `library/kicadLibrary.tar.xz` Missing from Repository

### Severity: 🔴 Critical (blocks KiCad library setup)

### Location
`Ubuntu/install-eSim-scripts/install-eSim-24.04.sh` — `copyKicadLibrary()` function, line ~266

### Error
```
tar (child): library/kicadLibrary.tar.xz: Cannot open: No such file or directory
tar: Error is not recoverable: exiting now
Error! Kindly resolve above error(s) and try again.
Aborting Installation...
```

### Root Cause
The `installers` branch only contains installer scripts. The `library/` folder (including `kicadLibrary/`) lives on the `master` branch as part of the eSim source code. The README documents that before distribution, the `kicadLibrary/` folder must be compressed into `kicadLibrary.tar.xz` — but this packaging step is never done automatically.

A developer running the installer from a git clone (rather than from an official release zip) will always hit this error.

### Fix Applied
When running from the `master` branch (which has the `kicadLibrary/` folder), the archive must be created manually:
```bash
cd library
tar -cJf kicadLibrary.tar.xz kicadLibrary/
cd ..
```

**Note:** This is also a documentation/packaging bug — the installer should either check for and create the archive automatically, or the README should clearly warn that the installer must be run from within a packaged release, not a bare git clone.

---

## Bug 6 — `nghdl.zip` Missing from Repository

### Severity: 🔴 Critical (blocks NGHDL installation)

### Location
`Ubuntu/install-eSim-scripts/install-eSim-24.04.sh` — `installNghdl()` function, line ~68

### Error
```
unzip: cannot find or open nghdl.zip, nghdl.zip.zip or nghdl.zip.ZIP.
Error! Kindly resolve above error(s) and try again.
Aborting Installation...
```

### Root Cause
Same packaging issue as Bug 5. The `nghdl/` directory exists in the `master` branch but is never zipped. The installer expects `nghdl.zip` to be present in the working directory.

### Fix Applied
```bash
sudo apt-get install -y zip
zip -r nghdl.zip nghdl/
```

---

## Bug 7 — `install-nghdl.sh` Does Not Accept `--install` Argument

### Severity: 🔴 Critical (NGHDL installation fails silently with wrong steps)

### Location
`Ubuntu/install-eSim-scripts/install-eSim-24.04.sh` — `installNghdl()`, line ~75  
`nghdl/install-nghdl.sh` — main argument parsing block

### Error
```
Unknown argument. Use one of: verilator, nghdl, softlink, config
```

### Root Cause
The eSim installer calls:
```bash
./install-nghdl.sh --install
```
But `nghdl/install-nghdl.sh` does not have an `--install` handler. Its argument parser only accepts individual step names: `verilator`, `nghdl`, `softlink`, `config`. The `--install` argument hits the `*` wildcard and prints the error message.

### Fix Applied
Changed the eSim installer to call each step individually:
```bash
# Before
./install-nghdl.sh --install

# After
./install-nghdl.sh verilator       # Install Verilator
./install-nghdl.sh nghdl           # Install NGHDL/Ngspice
./install-nghdl.sh softlink        # Create softlink
./install-nghdl.sh config          # Create config file
```

---

## Bug 8 — Verilator Installed from Missing Source Tarball

### Severity: 🔴 Critical (Verilator installation fails completely)

### Location
`nghdl/install-nghdl.sh` — `installVerilator()` function

### Error
```
tar: verilator-4.210.tar.xz: Cannot open: No such file or directory
cd: verilator-4.210: No such file or directory
./configure: No such file or directory
sysctl: cannot stat /proc/sys/hw/ncpu: No such file or directory
make: command not found
```

### Root Cause
Multiple issues in the original `installVerilator()` function:
1. `verilator-4.210.tar.xz` source tarball is not present in the repository
2. `sysctl -n hw.ncpu` is a **macOS-only** command for getting CPU core count — on Linux the equivalent is `nproc`
3. `make` and `build-essential` are not installed by default in the Codespace environment
4. The function attempts to compile Verilator 4.210 from source, but Ubuntu 25.10 ships Verilator 5.032 natively via `apt`

### Fix Applied
Replaced the entire source compilation approach with a simple `apt install`:
```bash
function installVerilator
{
    echo "Installing verilator via apt (Ubuntu 25.x native package)..."
    # On Ubuntu 25.04+, verilator is available in official repos.
    # Source compilation from tar.xz is not needed.
    sudo apt-get install -y verilator build-essential
    echo "Verilator installed successfully"
}
```

---

## Bug 9 — SKY130 PDK Installation Fails with Permission Denied

### Severity: 🟡 Medium (SKY130 PDK cannot be installed as non-root)

### Location
`Ubuntu/install-eSim-scripts/install-eSim-24.04.sh` — `installSky130Pdk()` function, line ~95

### Error
```
[Errno 13] Permission denied: '/usr/share/local'
Error! Kindly resolve above error(s) and try again.
Aborting Installation...
```

### Root Cause
The `volare enable` command was called without `sudo` but with `--pdk-root /usr/share/local/`, which requires root privileges to write to. Since `volare` is a Python tool running in a virtualenv as a regular user, it cannot write to `/usr/share/local/`.

### Fix Applied
Used `$HOME/.volare_tmp` as an intermediate download location (writable by the current user), then moved the result to `/usr/share/local/` with `sudo`:

```bash
# Before
volare enable --pdk sky130 --pdk-root /usr/share/local/ 0fe599b2afb6708d281543108caf8310912f54af
sudo mv /usr/share/local/volare/...

# After
mkdir -p $HOME/.volare_tmp
volare enable --pdk sky130 --pdk-root $HOME/.volare_tmp/ 0fe599b2afb6708d281543108caf8310912f54af
sudo mkdir -p /usr/share/local/
sudo mv $HOME/.volare_tmp/volare/.../sky130_fd_pr /usr/share/local/
rm -rf $HOME/.volare_tmp/
```

---

## Bug 10 — Desktop Directory Missing in Headless Environment

### Severity: 🟢 Minor (only affects desktop shortcut creation)

### Location
`Ubuntu/install-eSim-scripts/install-eSim-24.04.sh` — final installation steps

### Error
```
cp: cannot create regular file '/home/vscode/Desktop/': Not a directory
Error! Kindly resolve above error(s) and try again.
Aborting Installation...
```

### Root Cause
The installer attempts to copy `esim.desktop` to `~/Desktop/`, but this directory does not exist in headless environments (servers, Docker containers, GitHub Codespaces).

### Fix Applied
```bash
mkdir -p /home/vscode/Desktop
```

**Recommended permanent fix:** The installer script should check if the Desktop directory exists before copying:
```bash
mkdir -p $HOME/Desktop
cp esim.desktop $HOME/Desktop/
```

---

## Bug 11 — `nghdl-simulator-source.tar.xz` Missing (Unfixed)

### Severity: 🔴 Critical (NGHDL/Ngspice compilation fails)

### Location
`nghdl/install-nghdl.sh` — `installNGHDL()` function

### Error
```
tar: nghdl-simulator-source.tar.xz: Cannot open: No such file or directory
cd: /home/vscode/nghdl-simulator: No such file or directory
../confi.gure: No such file or directory   ← Note: typo in script (confi.gure)
make: No targets specified and no makefile found.
```

### Root Cause
The NGHDL installer expects `nghdl-simulator-source.tar.xz` to be present in the `nghdl/` directory. This file is not present in the repository — it must be downloaded separately. Additionally, there is a **typo** in the script on line 98: `../confi.gure` should be `../configure`.

### Status
**Not fully fixed** — requires either:
1. Downloading the ngspice-ghdl source tarball from the FOSSEE servers, or  
2. Installing `ngspice` via `apt` and using the system installation

### Note on Typo
```bash
# Line 98 in nghdl/install-nghdl.sh - typo present in original source
../confi.gure --enable-xspice ...   # ← Should be ../configure
```

---

## Summary Table

| # | Bug Description | File | Severity | Status |
|---|---|---|---|---|
| 1 | Broken `lsb_release` regex for version detection | `install-eSim.sh` | 🔴 Critical | ✅ Fixed |
| 2 | No fallback for Ubuntu 25.x — hard exits | `install-eSim.sh` | 🔴 Critical | ✅ Fixed |
| 3 | KiCad 6.0 PPA missing for Ubuntu 25.x ("questing") | `install-eSim-24.04.sh` | 🔴 Critical | ✅ Fixed |
| 4 | `apt-get xz-utils` missing `install -y` subcommand | `install-eSim-24.04.sh` | 🟡 Medium | ✅ Fixed |
| 5 | `library/kicadLibrary.tar.xz` not packaged in repo | Missing asset | 🔴 Critical | ✅ Fixed |
| 6 | `nghdl.zip` not packaged in repo | Missing asset | 🔴 Critical | ✅ Fixed |
| 7 | `install-nghdl.sh` does not accept `--install` argument | `install-eSim-24.04.sh` | 🔴 Critical | ✅ Fixed |
| 8 | Verilator compiled from missing source tarball; `sysctl` macOS-only | `nghdl/install-nghdl.sh` | 🔴 Critical | ✅ Fixed |
| 9 | SKY130 `volare enable` permission denied on `/usr/share/local` | `install-eSim-24.04.sh` | 🟡 Medium | ✅ Fixed |
| 10 | `~/Desktop/` directory missing in headless environment | `install-eSim-24.04.sh` | 🟢 Minor | ✅ Fixed |
| 11 | `nghdl-simulator-source.tar.xz` missing + `confi.gure` typo | `nghdl/install-nghdl.sh` | 🔴 Critical | ⚠️ Partial |

---

## Final Installation Result

After applying all fixes, the installer completed successfully:

```
-----------------eSim Installed Successfully-----------------
Type "esim" in Terminal to launch it
or double click on "eSim" icon placed on Desktop
```

---

## Files Modified

| File | Changes |
|---|---|
| `Ubuntu/install-eSim.sh` | Fixed version detection; added Ubuntu 25.x fallback |
| `Ubuntu/install-eSim-scripts/install-eSim-24.04.sh` | KiCad PPA fix; xz-utils fix; NGHDL argument fix; SKY130 permission fix; Desktop dir fix |
| `nghdl/install-nghdl.sh` | Replaced Verilator source compilation with apt install |

---

## References

- [eSim Official Website](https://esim.fossee.in)
- [KiCad Launchpad PPA](https://launchpad.net/~kicad/+archive/ubuntu/kicad-6.0-releases)
- [Volare PDK Manager](https://github.com/efabless/volare)
- [Ubuntu 25.10 Package Archive](http://archive.ubuntu.com/ubuntu/dists/questing/)
- [NGHDL Repository](https://github.com/fossee/nghdl)
