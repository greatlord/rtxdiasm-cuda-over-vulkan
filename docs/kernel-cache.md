# Kernel cache

ACuda caches compiled kernels (and known compile failures) between runs so a
launch does not re-lift and re-compile every entry in a fatbin.  There are two
independent settings: *where* the cache lives, and *which namespace* is used
inside it.

## Folder

| environment | cache directory |
|---|---|
| `ACUDA_CACHE_DIR=<dir>` | `<dir>` |
| unset (Windows) | `%LOCALAPPDATA%\acuda\kernel-cache` |
| unset (POSIX) | `$XDG_CACHE_HOME/acuda/kernel-cache`, `$HOME/.cache/acuda/kernel-cache`, or `/tmp/acuda-kernel-cache` |

`ACUDA_NO_CACHE=1` disables both reads and writes.

## Tag (namespace)

The tag is folded into every cache key — both the `.akb` success blobs and the
`.failed` markers — so changing it retires everything cached under the old
namespace without deleting any files.

| situation | tag |
|---|---|
| `ACUDA_CACHE_TAG=<value>` | `<value>` |
| unset | `acuda-kernel-cache-<PE link timestamp of nvcuda.dll>` |

The default is deterministic, not random and not a fixed string:

* repeated runs of one build reuse the cache (fast path);
* a rebuilt driver gets a new timestamp and therefore starts cold.

### Do not use a random tag for normal runs

A per-run random tag makes every launch cold: the driver re-lifts every kernel
and re-attempts every `.failed` entry, which is the slow path.  Use a
random/specific tag only as a deliberate one-off probe:

```powershell
$env:ACUDA_CACHE_TAG = "probe-$(Get-Date -Format yyyyMMddHHmmss)"
```

### Caveat: lifter-only rebuilds

The default stamp tracks the **driver** (`nvcuda.dll`) build.  If only the C#
lifter (`rtxdiasm_native.dll`) changes and the driver is not rebuilt, the tag
does not change.  Set `ACUDA_CACHE_TAG` explicitly when testing lifter-only
changes, or fold the helper DLL's size/mtime into the stamp as a follow-up.

## Negative cache

Deterministic compile failures (unsupported SASS, missing types, ...) are
recorded as `<hash>.failed` and skipped on subsequent runs.  Transient
failures (OOM, resource exhaustion) are never cached, so they are retried on
the next run.

Inspect or clear the default cache:

```powershell
$cache = "$env:LOCALAPPDATA\acuda\kernel-cache"
Get-ChildItem $cache -Filter *.failed | Measure-Object
Remove-Item "$cache\*" -Force   # full reset
```

## Quick reference

| goal | setting |
|---|---|
| cold cache without deleting files | `ACUDA_CACHE_TAG=<new value>` |
| cold cache, default folder | delete `%LOCALAPPDATA%\acuda\kernel-cache\*` |
| disable cache entirely | `ACUDA_NO_CACHE=1` |
| relocate cache | `ACUDA_CACHE_DIR=<dir>` |
