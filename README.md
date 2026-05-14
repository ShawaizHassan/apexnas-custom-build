# ApexNAS — Custom NAS Build from XigmaNAS Source

> White-label NAS operating system compiled from source on FreeBSD 14.4.  
> Based on [XigmaNAS](https://sourceforge.net/projects/xigmanas/) — modified, recompiled, and deployed as a custom-branded distribution.

---

## Overview

This project documents the complete process of taking the XigmaNAS open-source NAS operating system, applying custom branding at source level, compiling the entire system from scratch on FreeBSD 14.4, and producing a bootable LiveCD ISO.

The result is **ApexNAS** — a fully functional NAS OS with custom identity, compiled from source, verified in a VirtualBox VM.

**Build output:**
```
ApexNAS-x64-LiveCD-14.3.0.5.10606.iso     559 MB
ApexNAS-x64-embedded-14.3.0.5.10606.img.xz  222 MB
```

---

## Build Environment

| Component | Details |
|-----------|---------|
| Host OS | Windows 11 |
| Hypervisor | Oracle VirtualBox |
| Build VM OS | FreeBSD 14.4-RELEASE-p3 |
| Build VM RAM | 4 GB |
| Build VM Disk | 50 GB |
| Source | XigmaNAS SVN trunk (revision 10606) |
| FreeBSD Source | 14.4-RELEASE (src.txz) |

> **Note:** The official XigmaNAS build guide targets FreeBSD 11.2. This build was done on FreeBSD 14.4, which required resolving several compatibility issues documented below.

---

## What Was Modified

### Branding (Source-Level)
All changes were applied before compilation so they are baked into the final ISO.

| File | Change |
|------|--------|
| `svn/www/images/login_logo.png` | Replaced XigmaNAS logo with ApexNAS logo |
| `svn/www/favicon.ico` | Replaced with custom ApexNAS favicon |
| `svn/www/*.php` (100+ files) | Replaced all `XigmaNAS` → `ApexNAS` strings via `sed` |
| `svn/www/css/*.php` | Updated brand color and name references |

```sh
# How string replacement was done across all PHP files
grep -rl "XigmaNAS" /usr/local/xigmanas/svn/www --include="*.php" \
  | xargs sed -i '' 's/XigmaNAS/ApexNAS/g'
```

### Build System Fixes

| File | Change | Reason |
|------|--------|--------|
| `svn/build/functions.inc` | Increased `XIGMANAS_MDLOCAL_MINI_SIZE` from 52 → 150 | Filesystem full during image creation |
| `svn/build/functions.inc` | Commented out `make clean` and cookie deletion | Preserve build state across restarts |
| `svn/build/ports/samba422/Makefile` | Commented out `FAM_USES` and `FAM_CONFIGURE_WITH` | `fam` framework removed in FreeBSD 14.4 |
| `svn/build/ports/*/pkg-state` | Set to `HIDE` for dead/irrelevant ports | arcconf, ataidle, ipmitool, sas2ircu, sas3ircu, tw_cli, vbox |

---

## Build Process

The build uses an interactive compile menu (`make.sh`) with these steps executed in order:

```
2  → Create Filesystem Structure
3  → Build/Install Kernel        (prebuild/patches skipped — incompatible with FreeBSD 14.4)
4  → Build World
5  → Import Base-Ports
6  → Build Ports                 (most complex phase — see Issues section)
7  → Build Bootloader
8  → Add Necessary Libraries     (add_libs only)
9  → Modify File Permissions
13 → Create LiveCD ISO
```

### Pre-requisite before Step 13
The ISO creation script expects `mdlocal-mini.xz` to already exist:
```sh
xz -0kv /usr/local/xigmanas/mdlocal-mini
```
The `-0` flag reduces compression time from ~80 minutes to under 2 minutes.

---

## Issues Encountered & Resolved

### 1. Kernel Patches Incompatible with FreeBSD 14.4
Patches written for FreeBSD 11.x fail against 14.4 source.  
**Fix:** Disabled the `prebuild` option in Step 3. Built kernel directly from unpatched 14.4 source.

### 2. Slow VM Network / Dead Download URLs
VM network throttled to ~11 KB/s in China. Several port upstream URLs permanently dead (arcconf, ataidle).  
**Fix:** Downloaded all distfiles on Windows host, transferred via SCP. Dead ports suppressed via `pkg-state HIDE`.

### 3. Truncated Distfiles
Files downloaded via PowerShell `Invoke-WebRequest` were silently truncated.  
**Fix:** Wrote a verification script comparing actual file sizes against `distinfo` SIZE entries. Re-downloaded failures using `curl.exe --ssl-no-revoke`.

```sh
# Verification script
for port in /usr/local/xigmanas/svn/build/ports/*/; do
  distinfo="${port}distinfo"
  [ ! -f "$distinfo" ] && continue
  grep "^SIZE" "$distinfo" | while read -r line; do
    file=$(echo "$line" | sed 's/SIZE (\(.*\)) = .*/\1/')
    expected=$(echo "$line" | awk '{print $4}')
    actual=$(stat -f%z "/usr/ports/distfiles/$file" 2>/dev/null)
    [ "$actual" = "$expected" ] && echo "OK: $file" || echo "BAD: $file"
  done
done
```

### 4. Build Cookie Deletion on Restart
Every restart recompiled all ports from scratch.  
**Fix:** Commented out `make clean` (line 1920) and cookie deletion (line 1948) in `functions.inc`.

### 5. Samba — `USES=fam` Unknown
`fam` (File Alteration Monitor) was removed from FreeBSD 14.4 ports.  
**Fix:**
```sh
sed -i '' '368s/FAM_USES=/#FAM_USES=/' svn/build/ports/samba422/Makefile
sed -i '' '369s/FAM_CONFIGURE_WITH=/#FAM_CONFIGURE_WITH=/' svn/build/ports/samba422/Makefile
```

### 6. Rust Build Failure (Samba Dependency Chain)
Samba's dependency chain triggered a full Rust 1.92.0 source compilation which failed at the bootstrap stage.  
**Fix:** Pre-installed Rust via pkg, bypassing source compilation entirely:
```sh
pkg install -y rust
```

### 7. mdlocal Filesystem Full
`XIGMANAS_MDLOCAL_MINI_SIZE` set to 52 MB — insufficient for compiled ports.  
**Fix:** Increased to 150 in `functions.inc` (both definition locations on lines 91 and 97).

### 8. mdlocal-mini.xz Missing at ISO Creation
Step 13 expects the compressed memory filesystem to already exist.  
**Fix:** Manually compress before running Step 13:
```sh
xz -0kv /usr/local/xigmanas/mdlocal-mini
```

### 9. Perl / autoreconf Version Mismatch
Ports expected `perl5.42.0` and `autoreconf2.72` but system had newer patch versions.  
**Fix:** Created symlinks:
```sh
ln -s /usr/local/bin/perl5.42.2 /usr/local/bin/perl5.42.0
ln -sf /usr/local/bin/autoreconf2.73 /usr/local/bin/autoreconf2.72
```

---

## Final Verification

| Check | Result |
|-------|--------|
| ISO size | 559 MB |
| Binaries in `/usr/local/bin` | 116 |
| Binaries in `/usr/local/sbin` | 23 |
| WebGUI accessible | ✓ http://127.0.0.1:5050 |
| Custom logo visible | ✓ |
| Custom favicon visible | ✓ |
| Version string | 14.3.0.5 - Jalanto (revision 10606) |
| Compile date | Tuesday May 05 15:06:16 UTC 2026 |
| Platform OS | FreeBSD 14.4-RELEASE-p3 |
| Samba (smbd) | ✓ present |
| lighttpd | ✓ serving WebGUI |
| PHP | ✓ executing |

---

## Repository Contents

```
svn/build/functions.inc          — Build system: mdlocal size fix, cookie preservation
svn/build/create-all.sh          — Port selection list
svn/build/make.sh                — Interactive compile menu
svn/build/kernel-patches/        — Kernel patches (not applied on FreeBSD 14.4)
svn/build/ports/samba422/Makefile — FAM fix
svn/build/ports/lighttpd/Makefile
svn/build/ports/rrdtool/Makefile
svn/build/ports/python311/Makefile
```

> Large artifacts (rootfs, ISO, distfiles, ports work directories) are excluded via `.gitignore`.

---

## Related

- [XigmaNAS Project](https://sourceforge.net/projects/xigmanas/)
- [FreeBSD 14.4 Release](https://www.freebsd.org/releases/14.4R/)
- Full technical report available on request.
