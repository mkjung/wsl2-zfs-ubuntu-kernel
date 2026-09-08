# wsl2-zfs-ubuntu-kernel

Notes on running **Ubuntu on WSL2 with OpenZFS as the primary filesystem**, kept
public so the environment can be cited from upstream issues and pull requests.
Two things live here:

1. **The current environment** (below) — what "tested on Windows 11 / WSL2 /
   Ubuntu 26.04 LTS / ZFS" means concretely, including the kernel build.
2. **A kernel panic and its fix** — in July 2026, on the then-current WSL kernel
   6.6.123.2, the whole distro died whenever memory compaction ran while a
   process (OpenAI Codex CLI, as it happened) was writing to ZFS. The write-up,
   the semantic backport of the upstream fix, and how the current kernel makes
   the patch unnecessary are in [`docs/`](docs/) and [`patches/`](patches/).

Nothing here contains credentials, pool layouts, or host-specific paths; the
private working notes this was distilled from are not published.

## Current environment (2026-09)

| item | value |
|---|---|
| Host | Windows 11, WSL 2.7.10.0 (WSLg 1.0.73.2) |
| Distro | Ubuntu 26.04 LTS (`systemd=true`, `appendWindowsPath=false`) |
| Kernel | `6.18.35.2-microsoft-standard-WSL2+` — built from the stock tag `linux-msft-wsl-6.18.35.2` with Microsoft's own config, no source or config change by hand. Build record: [mkjung/WSL2-Linux-Kernel @ `mksols/linux-msft-wsl-6.18.35.2-zfs-build`](https://github.com/mkjung/WSL2-Linux-Kernel/tree/mksols/linux-msft-wsl-6.18.35.2-zfs-build/mksols/6.18.35.2) |
| OpenZFS | 2.4.1 (`zfs-dkms 2.4.1-1ubuntu5.1`), DKMS-built against that exact kernel release; `zfs.ko`/`spl.ko` vermagic match `uname -r` |
| Pool | single pool on a dynamically expanding VHDX attached with `wsl --mount --vhd --bare`; `ashift=12`, `autotrim=on`, `compression=zstd`, `atime=off` |
| Datasets | native encryption `aes-256-gcm` under one encryption root; the home directory and every service directory are datasets under it |
| Block cloning | pool feature `block_cloning` active, `zfs_bclone_enabled=1`, `zfs_bclone_wait_dirty=1` — so `ioctl(FICLONE)` / `cp --reflink` shares blocks on encrypted datasets too |
| Root / `/tmp` | ext4 (WSL system VHDX) / tmpfs — neither supports reflink, which matters for tests that use `os.tmpdir()` |
| `.wslconfig` | `memory=20GB`, `processors=8`, `swap=4GB`, `networkingMode=nat`, `guiApplications=false`, `[experimental] autoMemoryReclaim=disabled`, `kernel=` pointing at the custom `bzImage` |
| Toolchain | gcc 15.2, binutils 2.46, pahole 1.31, Node 24 |

Why a custom kernel at all: OpenZFS modules must be built against the exact
kernel release and symbol versions of the running kernel, and Microsoft ships no
matching modules. So the kernel is built from Microsoft's tag with Microsoft's
config, then OpenZFS is DKMS-built against that tree. The full command sequence
is in the build record linked above; the short form:

```bash
git clone --branch linux-msft-wsl-6.18.35.2 --depth 1 \
  https://github.com/microsoft/WSL2-Linux-Kernel.git WSL2-Linux-Kernel-6.18.35.2
cd WSL2-Linux-Kernel-6.18.35.2
make -j"$(nproc)" KCONFIG_CONFIG=Microsoft/config-wsl
KERNEL_RELEASE="$(make -s kernelrelease)"
make INSTALL_MOD_PATH="$PWD/modules" modules_install
sudo dkms build   --force -j "$(nproc)" -m zfs -v 2.4.1 -k "$KERNEL_RELEASE" --kernelsourcedir "$PWD"
sudo dkms install --force -m zfs -v 2.4.1 -k "$KERNEL_RELEASE" --kernelsourcedir "$PWD" \
  --installtree "$PWD/modules/lib/modules" --no-depmod
```

## The July 2026 panic, in one paragraph

On WSL kernel 6.6.123.2 with OpenZFS 2.2.2 (Ubuntu 24.04), WSL's `mini_init`
periodically writes to `/proc/sys/vm/compact_memory`. Linux 6.6's
`fallback_migrate_folio()` handled a dirty folio of a filesystem without a
`migrate_folio` callback — OpenZFS 2.2 is one — by calling the filesystem's
`->writepage()`. ZFS completes that writeback asynchronously, the migration
retried while the folio was still under writeback, and
`migrate_folio_extra()` hit `BUG_ON(folio_test_writeback(src))`
(`kernel BUG at mm/migrate.c:662`). The VM panicked, WSL restarted the distro,
and Windows Terminal showed only `[process exited with code 1]` — which looked
like the foreground program (Codex) crashing, but was the kernel.

The fix was a semantic backport of upstream Linux commit
[`7ee3647243e5` "migrate: Remove call to ->writepage"](https://github.com/torvalds/linux/commit/7ee3647243e5c4a9d74d4c7ec621eac75c6d37ea)
to 6.6.123.2: drop the migration-only `writeout()` helper and return `-EBUSY`
for dirty folios. That patch is
[`patches/0001-migrate-remove-writepage-from-fallback-wsl-6.6.123.2.patch`](patches/0001-migrate-remove-writepage-from-fallback-wsl-6.6.123.2.patch).
The current kernel (6.18.35.2) already carries the upstream change —
`fallback_migrate_folio()` there returns `-EBUSY` for dirty folios and warns
when a filesystem lacks `migrate_folio` — so no patch is applied today, and the
post-build check is simply:

```bash
rg -n -A8 'static int fallback_migrate_folio' mm/migrate.c   # expect: if (folio_test_dirty(src)) return -EBUSY;
```

Full write-up: [`docs/2026-07-wsl-6.6-migrate-writepage-panic.md`](docs/2026-07-wsl-6.6-migrate-writepage-panic.md).

## Also in this repository

- [`examples/zfs-modprobe.conf`](examples/zfs-modprobe.conf) — ARC limits as a
  modprobe file. Inline comments (`options zfs zfs_arc_max=… # 2 GiB`) are passed
  to the module as parameters and log `unknown parameter '#'`; keep comments on
  their own line.

## License

The kernel patch under `patches/` is derived from the Linux kernel and is
**GPL-2.0-only**. Everything else in this repository is MIT (see `LICENSE`).
