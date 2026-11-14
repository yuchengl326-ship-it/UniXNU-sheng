# macOS 27.3 dual-system build plan (fork)

Scope: build `macos-27.3-26C5035r` through GitHub Actions for a dual-system
device. Android stays on userdata; the macOS root image and Data store live
on a `unixnu` partition carved from the tail of userdata.

## Data seed sizing

The upstream macOS profile sets `data_gib` 210: the Data store fills the
whole 232.4 GiB userdata partition below its 1 GiB log reserve, next to the
16.4 GiB root image (OS, SystemOS, App and their .vroot tables). A
dual-system device keeps Android on userdata, so the Data store must fit the
carved `unixnu` partition instead:

    data_gib = partition GiB - 16.4 (root image) - 1 (log reserve) - ~1 (slack)

This fork sets `data_gib` 30 (provisional, 48 GiB partition target).
Finalize the value after the on-device GPT probe fixes the partition size.

## Workflow change

`.github/workflows/package.yml` gains the `macos-27.3-26C5035r` choice and a
build step that runs `unixnu build userdata` before the seed step: the
userdata build fetches the SoC sources (Context initialisation requires the
cached SoC, see the host check below) and extracts the OS disk image;
`make-macos-data-seed.py` then runs as root with `--gib` from board.toml,
`--skip-setup` and a seeded local administrator (`sheng`). Without the seed,
`runtime.prepare` refuses to build.

`userdata-root.img` and `userdata-root.json` join the run artifacts: the
macOS profile has no UEFI install entry, so the root image is flashed with
`fastboot flash unixnu userdata-root.img` after the partition carve.

## Host check

Command (Windows host, Python 3.13, `-I`):

    python -I -c "import sys; sys.path.insert(0, 'scripts');
    from unixnu.cli import parser; from unixnu.config import Context;
    args = parser().parse_args(['ipsw', '-D', '../UniXNU-sheng', '--os',
    'macos-27.3-26C5035r', 'info', 'OS']); Context(args)"

Result: board.toml and the macOS profile resolve; Context initialisation
fails with "SoC sm8550 is not cached; run unixnu fetch" before any fetch.
That fixes the workflow step order: `unixnu build userdata` (which fetches
the SoC) must run before the `Ipsw.disk_image("OS")` lookup in the seed
step.

## Limits

No hardware run yet: the `unixnu` partition does not exist (on-device probe
pending), the macOS build has not completed, and `data_gib` 30 is
provisional.

## Build runs

Run 43591366749 (2026-11-14) failed in the macos step before any fetch: the
unixnu CLI takes options on the subparser, so `python3 unixnu -D DIR build
userdata` exits with "invalid choice: '../device'". The workflow now runs
`python3 unixnu build userdata -D ../device --os "$OS"`.
