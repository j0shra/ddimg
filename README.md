# DDimg-Scripted

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

5. **Imaging profile** — Four ddrescue strategies:

   | # | Profile | Description |
   |---|---------|-------------|
   | 1 | **Normal** | Pass 1: `-n` (no scrape). Pass 2: `-r3` (3 retries). Optional Pass 3: `-d -r3` (direct I/O). Balanced default. |
   | 2 | **Dying-Triage** | Pass 1: `-n -b 1M`. Pass 2: `-n -R -b 1M` (reverse). Large block size, forward then reverse pass. Avoids prolonged retries on a rapidly dying disk. |
   | 3 | **Aggressive** | Pass 1: `-n`. Pass 2: `-r7`. Pass 3: `-d -r7`. More retries for a drive that is degraded but stable. |
   | 4 | **SeverelyFailing-HangProne** | Pass 1 & 2 both use `--min-read-rate=64KiB --skip-size=1MiB,64MiB --timeout=8`. Skips and times out on slow zones to avoid indefinite hangs on a drive that stalls. |

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

---

## Installation

### Option 1 — Clone and install globally

```bash
git clone https://github.com/j0shra/DDimg-Scripted.git
cd DDimg-Scripted
sudo cp ddimg /usr/local/bin/ddimg
sudo chmod +x /usr/local/bin/ddimg
```

### Option 2 — One-liner

```bash
sudo curl -fsSL https://raw.githubusercontent.com/j0shra/DDimg-Scripted/main/ddimg -o /usr/local/bin/ddimg && sudo chmod +x /usr/local/bin/ddimg
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
