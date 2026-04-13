# esp32-idf-sqlite3 (Thebys fork)

SQLite3 library for ESP-IDF with VFS correctness fixes and modernized SQLite.

Forked from [siara-cc/esp32-idf-sqlite3](https://github.com/siara-cc/esp32-idf-sqlite3).

## Changes from upstream

### SQLite upgrade: 3.25.2 → 3.46.0

The upstream repository ships SQLite 3.25.2 (September 2018). This fork upgrades the amalgamation to **SQLite 3.46.0** (June 2024), bringing 6 years of bug fixes, security patches, and performance improvements. Based on [PR #30](https://github.com/siara-cc/esp32-idf-sqlite3/pull/30) by fedepell.

Flash impact: ~+90 KB. RAM impact: ~+2.5 KB.

### VFS layer fixes (`esp32.c`)

The ESP32 VFS bridge (`esp32.c`) had several correctness issues, some of which caused **silent data loss** on modern ESP-IDF configurations. All fixes are in a single file and do not touch the SQLite amalgamation.

#### Critical: fix silent data loss when `stat()` is unavailable

**Affected configurations:** ESP-IDF with `CONFIG_VFS_SUPPORT_DIR=n` (the ESPHome default since October 2025).

When `CONFIG_VFS_SUPPORT_DIR` is disabled, `stat()` returns `ENOSYS`. The original `esp32_Access()` used `stat()` to check file existence — it always reported "file not found." This caused `esp32_Open()` to open existing databases with `fopen("w+")`, **truncating them to zero bytes on every open**.

**Fix:** `esp32_Access()` now uses `fopen("r")` + `fclose()` to probe file existence. This works regardless of `CONFIG_VFS_SUPPORT_DIR`.

#### Fix missing zero-fill on short reads

The SQLite VFS contract requires `xRead()` to zero-fill the unread portion of the buffer when returning `SQLITE_IOERR_SHORT_READ`. The original implementation did not do this. Uninitialized buffer data can cause database corruption or unpredictable behavior.

#### Fix `esp32_Open` VFS contract violations

- `memset(file, 0, ...)` is now called **before** the NULL path check, ensuring `pMethods` is always NULL on error return (SQLite VFS contract requirement).
- `*outflags` is now set on successful open.
- The `SQLITE_OPEN_CREATE` flag is respected: file creation (`fopen("w+")`) only happens when the caller explicitly requests it.

#### Implement `esp32_Truncate` with `ftruncate()`

Previously a no-op (`return SQLITE_OK`). Now calls `ftruncate()` when available (`CONFIG_VFS_SUPPORT_DIR=y` or non-ESP32 hosts). Falls back to no-op when `ftruncate()` is unavailable, matching the original behavior. Memory-backed journals (fd == NULL) are handled with an early return.

#### Fix `esp32_FileControl` return value

Changed from `return SQLITE_OK` to `return SQLITE_NOTFOUND`. The original `SQLITE_OK` is actively harmful: several opcodes (e.g. `FCNTL_LOCKSTATE`, `FCNTL_VFSNAME`, `FCNTL_POWERSAFE_OVERWRITE`) cause SQLite to read from the `arg` pointer after a successful return — but since the ESP32 VFS never writes to `arg`, SQLite reads uninitialized data. Other opcodes (`FCNTL_OVERWRITE`, `FCNTL_SIZE_HINT`) can cause SQLite to skip safety steps it would otherwise perform. `SQLITE_NOTFOUND` tells SQLite to use its default fallback behavior, which is the correct and safe response for unimplemented opcodes. This matches all reference VFS implementations (unix, windows).

#### Remove dead code and unused variables

Removed unused `file` variables in stub functions, dead `nRead >= 0` branch (size_t is always ≥ 0), and commented-out stat code in `esp32_FullPathname`.

### Build system updates (`CMakeLists.txt`)

- `PRIV_REQUIRES`: replaced `spi_flash` with `esp_system esp_rom` for ESP-IDF 5.x compatibility.
- `SPI_FLASH_SEC_SIZE` is now defined inline (hardware constant, avoids build dependency on `spi_flash` component).
- Added `__has_include(<esp_random.h>)` guard for ESP-IDF 5.x where `esp_random()` moved from `esp_system.h` to `esp_random.h`.

## Known limitations

- **VACUUM** works correctly in single-task mode but causes FreeRTOS scheduler crashes in multi-task environments (e.g., when an HTTP server task coexists). The root cause appears to be memory layout corruption during VACUUM's intensive malloc/free pattern. VACUUM is not recommended on ESP32 with this library.
- **Threading:** SQLite is compiled with `SQLITE_THREADSAFE=0`. All access to the same `sqlite3*` handle must be serialized externally (e.g., with a mutex) when multiple FreeRTOS tasks are involved.

## Installation

Use as an ESP-IDF component:

```
git submodule add https://github.com/Thebys/esp32-idf-sqlite3 components/esp32-idf-sqlite3
```

Or with ESPHome `external_components` via `add_idf_component()` pointing to a local path.

## Upstream

Original library by Arundale Ramanathan (siara-cc). See [upstream repository](https://github.com/siara-cc/esp32-idf-sqlite3) for examples and general documentation.
