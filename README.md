# ddimg

An interactive Bash wrapper around `ddrescue` that guides you through imaging a failing or healthy drive with minimal manual command construction.

---

## How It Works

`ddimg` walks you through every decision interactively:

1. **Drive selection** — Displays all detected block devices via `lsblk`, then prompts for a source drive letter (the disk to image) and a destination drive letter (where the `.img` will be stored). It assumes `/dev/sdX` for the source and `/dev/sdY1` (partition 1) for the destination.

2. **Source target** — Choose to image the whole source disk or a specific partition on it. Imaging a partition saves space; imaging the whole disk is safer for full forensic recovery.

3. **Mount management** — Automatically mounts the destination partition at `/mnt/backupdrive`. If the mountpoint or destination partition is already mounted, it offers to unmount cleanly before proceeding. If the wrong device ends up mounted, it errors out rather than writing to the wrong place.

4. **Recovery folder** — Four naming options for where the image and mapfile land on the destination:
   - `recovery` (auto-increments to `recovery-1`, `recovery-2`, … if it already exists)
   - Timestamped: `recovery-YYYY-MM-DD_HH-MM`
   - Custom name (auto-increments if it already exists)
   - Resume/re-attempt an existing folder — scans for folders containing a prior `image.img` or `recovery.map`, or lets you type a path manually. No auto-increment; reuses whatever is already there.

5. **Imaging profile** — Five ddrescue strategies:

   | # | Profile | Description |
   |---|---------|-------------|
   | 1 | **Normal** | Pass 1: `-n` (no scrape). Pass 2: `-r3` (3 retries). Optional Pass 3: `-d -r3` (direct I/O). Balanced default. |
   | 2 | **Dying-Triage** | Pass 1: `-n -b 1M`. Pass 2: `-n -R -b 1M` (reverse). Large block size, forward then reverse pass. Avoids prolonged retries on a rapidly dying disk. |
   | 3 | **Aggressive** | Pass 1: `-n`. Pass 2: `-r7`. Pass 3: `-d -r7`. More retries for a drive that is degraded but stable. |
   | 4 | **SeverelyFailing-HangProne** | Pass 1 & 2 both use `--min-read-rate=64KiB --skip-size=1MiB,64MiB --timeout=8`. Skips and times out on slow zones to avoid indefinite hangs on a drive that stalls. |
   | 5 | **DeadHead-Auto** | Pass 1: `-n --min-read-rate=0 --timeout=5m`. Pass 2: same, reversed (`-R`). Optional Pass 3: `-d -r1 --timeout=10m`. Requires **ddrescue 1.30+**. See below. |

### Profile 5 — DeadHead-Auto

Profile 5 targets a drive with a **dead or dying head**, and it works by specifying *less* than profile 4, not more.

ddrescue 1.30 reordered its algorithm into five phases — copying, trimming, **sweeping**, scraping, retrying. When the copying phase hits a read error it jumps forward by the skip size to get off the bad area fast, which leaves unread holes marked `non-tried`. Sweeping is the phase that goes back and reads those holes with skipping disabled. In 1.30 it was promoted out of the copying phase and moved to *after* trimming, on the reasoning that trimming inward from known-good edges has a much higher hit rate, and it sharpens the boundaries of the bad region before sweeping commits to reading it.

For a drive with one dead head out of four, ~75% of the data is still perfectly readable, but the dead head's sectors are interleaved across the platter in a repeating pattern. Older versions kept re-hitting that pattern — upstream measured ~3.78 million read errors on a 1 TB disk. With 1.30's ordering and larger default skip size, the same recovery takes about **283 read errors**.

What the profile does and does not set:

- **No `--skip-size`.** 1.30 defaults to `infile_size / 32768` initial with a max of 1% of the disk, so it scales with the drive. Profile 4's fixed `1MiB,64MiB` cap is too small to escape a dead head's territory efficiently on a large disk.
- **`--min-read-rate=0`** means *auto* — recalculated every second as `average_rate / 10` — rather than assuming a fixed 64 KiB/s.
- **`-n` is kept** in passes 1–2. That is `--no-scrape`; sector-by-sector scraping is wasted effort on a head that cannot read. It does **not** disable sweeping — that would be `-N` / `--no-sweep`, which this profile deliberately avoids.
- **Longer timeouts** (`5m` / `10m`) than profile 4's `--timeout=8`. Eight seconds is a triage tripwire; five minutes lets a slow-but-alive drive keep making progress. The tradeoff is that profile 5 will sit on a genuinely hung drive longer before bailing out.

Selecting profile 5 on a system with an older ddrescue exits immediately with a clear message rather than failing partway through an image. The check probes the binary `sudo` will actually invoke, since the passes run under `sudo` and that can resolve differently from your interactive `PATH`.

> **Note on `-N`:** ddrescue 1.30 reassigned the short flag `-N` from `--no-trim` to `--no-sweep`. If you adapt any recipe written for 1.29 or earlier, use the long form `--no-trim` to get the old behavior.

6. **Execution** — Runs each ddrescue pass, prompting between passes so you can stop if recovery looks good enough. All passes write to the same `image.img` and `recovery.map` so resuming works seamlessly.

7. **Notes file** — Appends a timestamped log of the run (profile, source, destination, planned commands) to `notes.txt` in the recovery folder. Resume runs append rather than overwrite, keeping a full history.

8. **Optional unmount** — Offers to unmount the destination partition when all passes are done.

---

## Requirements

The following commands must be present on your system:

```
ddrescue  lsblk  mount  umount  mountpoint  findmnt  date  find  sort  awk  sed  df  blockdev
```

On Debian/Ubuntu:

```bash
sudo apt install gddrescue util-linux
```

On Arch:

```bash
sudo pacman -S ddrescue util-linux
```

On Fedora/RHEL:

```bash
sudo dnf install ddrescue util-linux
```

### ddrescue version

Profiles 1–4 work on any reasonably recent ddrescue. **Profile 5 requires ddrescue 1.30 or newer.**

Check what you have:

```bash
ddrescue --version
```

Most distributions lag well behind upstream — Ubuntu 24.04, for example, ships 1.27 with no backport. If your package manager cannot get you to 1.30, build it from source:

```bash
sudo apt install lzip build-essential          # lzip is needed for GNU .tar.lz archives
wget https://ftp.gnu.org/gnu/ddrescue/ddrescue-1.30.tar.lz
tar --lzip -xf ddrescue-1.30.tar.lz
cd ddrescue-1.30
./configure && make && make check
sudo make install                              # installs to /usr/local
```

This installs to `/usr/local/bin/ddrescue` and leaves your distribution's copy at `/usr/bin/ddrescue` untouched, so your package manager stays consistent and you keep a fallback. `/usr/local/bin` precedes `/usr/bin` on the default `PATH` and in sudo's `secure_path` on Debian/Ubuntu, so the newer build is what runs. Verify with `which -a ddrescue`.

Mapfiles are compatible in both directions between 1.27 and 1.30, so a rescue started on one version can be resumed on the other.

To verify the download first (recommended — GNU signs its releases):

```bash
wget https://ftp.gnu.org/gnu/ddrescue/ddrescue-1.30.tar.lz.sig
gpg --keyserver keyserver.ubuntu.com --recv-keys 8FE99503132D7742
gpg --verify ddrescue-1.30.tar.lz.sig ddrescue-1.30.tar.lz
```

---

## Installation

### Option 1 — Clone and install globally

```bash
git clone https://github.com/j0shra/ddimg.git
cd ddimg
sudo cp ddimg /usr/local/bin/ddimg
sudo chmod +x /usr/local/bin/ddimg
```

### Option 2 — One-liner

```bash
sudo curl -fsSL https://raw.githubusercontent.com/j0shra/ddimg/main/ddimg -o /usr/local/bin/ddimg && sudo chmod +x /usr/local/bin/ddimg
```

---

## Making It Globally Executable

The script lives at `/usr/local/bin/ddimg` after installation. That directory is on the default `PATH` for most Linux distributions, so you can run it from anywhere as:

```bash
sudo ddimg
```

> `sudo` is required because ddrescue reads raw block devices and mounts filesystems.

If `/usr/local/bin` is not on your `PATH`, add this to your `~/.bashrc` or `~/.zshrc`:

```bash
export PATH="$PATH:/usr/local/bin"
```

Then reload your shell:

```bash
source ~/.bashrc   # or source ~/.zshrc
```

---

## Usage

Just run:

```bash
sudo ddimg
```

The script is fully interactive — no flags or arguments needed. Follow the prompts.

---

## Output Files

All output lands in a recovery folder on the destination partition (`/mnt/backupdrive/<folder-name>/`):

| File | Description |
|------|-------------|
| `image.img` | The raw disk or partition image |
| `recovery.map` | ddrescue mapfile — tracks which sectors were recovered |
| `notes.txt` | Timestamped log of each run against this folder |

---

## License

MIT
