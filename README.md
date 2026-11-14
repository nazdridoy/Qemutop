# qemu-top

`htop`-style live resource monitor for QEMU/KVM virtual machines.

Discovers all running QEMU VMs directly from `/proc` — **no libvirt, no virsh, no configuration required**.

Works with VMs launched by libvirt/virt-manager, GNS3, or any bare `qemu-system-*` command.

---

## Installation

```bash
# Copy to user bin directory
cp qemu-top ~/.local/bin/qemu-top
chmod +x ~/.local/bin/qemu-top

# Ensure ~/.local/bin is in your PATH (add to ~/.bashrc or ~/.zshrc if needed)
export PATH="$HOME/.local/bin:$PATH"
```

**Requirements:** Python 3.10+, standard library only.

---

## Usage

```bash
# Live TUI (default)
qemu-top

# Set refresh interval to 1 second
qemu-top -i 1
qemu-top --interval 1

# Print one snapshot table and exit
qemu-top --once
qemu-top --once --sort memory

# Disable color (useful over serial console or dumb terminals)
qemu-top --no-color

# Sort by a specific column at startup
qemu-top --sort cpu         # default
qemu-top --sort ram
qemu-top --sort read
qemu-top --sort write
qemu-top --sort netrx
qemu-top --sort nettx
qemu-top --sort name
qemu-top --sort pid
qemu-top --sort uptime

# Run self-tests
qemu-top --self-test

# Version
qemu-top --version
```

---

## TUI Keyboard Controls

| Key | Action |
|---|---|
| `q` / `Ctrl-C` | Quit |
| `f` | Force immediate refresh |
| `+` / `-` | Increase / decrease refresh interval |
| `↑` / `↓` | Navigate VM list |
| `Enter` / `d` | Open VM detail view |
| `l` | Show full QEMU command line |
| `Esc` | Return to main view |
| `c` | Sort by CPU |
| `m` | Sort by RAM |
| `r` | Sort by Disk Read |
| `w` | Sort by Disk Write |
| `n` | Sort by Net RX |
| `t` | Sort by Net TX |
| `N` | Sort by Name |
| `p` | Sort by PID |
| `u` | Sort by Uptime |

Pressing a sort key twice **reverses** the sort direction.

---

## Example TUI Output

```
 qemu-top v1.0.0   Host: myhost              VMs: 4  refresh: 2.0s
 CPU: 31.4% / 16 cores    RAM: 11.8 / 31.2 GiB

NAME               PID      CPU        RAM       DISK R       DISK W       NET RX       NET TX   UPTIME
win11▼            4217   348.2%   6.44 GiB   12.4 MiB/s    2.1 MiB/s    1.8 MiB/s  420.0 KiB/s   3h12m
ubuntu-server     4382    62.1%   2.19 GiB    1.2 MiB/s  340.0 KiB/s  620.0 KiB/s  110.0 KiB/s  1d04h
router            4511    11.0%  512.00 KiB  120.0 KiB/s   20.0 KiB/s    2.4 MiB/s    1.1 MiB/s   8h21m
gns3-router       4620   184.2%   3.91 GiB    8.7 MiB/s    1.4 MiB/s   14.2 MiB/s    5.2 MiB/s   5h04m
TOTAL (4 VMs)             605.5%  13.06 GiB   22.4 MiB/s    3.9 MiB/s   18.9 MiB/s    6.8 MiB/s

 q:Quit  ↑↓:Select  Enter/d:Detail  l:CmdLine  c/m/r/w/n/t/N/p/u:Sort  +/-:Interval  f:Refresh
```

---

## How Metrics Are Collected

### VM Discovery
Scans `/proc` for all processes whose `/proc/<pid>/comm` or `argv[0]` matches `qemu-system-*`. Handles `qemu-system-x86_64`, `qemu-system-aarch64`, `qemu-system-arm`, `qemu-system-riscv64`, and any other architecture variant. Race conditions and permission errors are silently ignored.

### VM Name
Resolved from the QEMU `-name` argument:
- `-name win11` → `win11`
- `-name guest=ubuntu,debug-threads=on` → `ubuntu`
- No `-name` argument → `qemu-<PID>`

### CPU
Read from `/proc/<pid>/stat` fields `utime` + `stime` (jiffies). CPU% is calculated as a **delta between two consecutive samples** divided by the elapsed wall-clock time.

Display style matches `top`: **one fully utilized CPU core = 100%**. A VM using 4 full cores will show ~400%. The first sample always shows `--` (no artificial data).

### RAM
Read from `/proc/<pid>/status` as `VmRSS` (Resident Set Size in KiB). Displayed as MiB or GiB.

> **Important:** Host RSS is the memory consumed by the QEMU *process* on the host. It is **not** the same as memory used inside the guest OS. Guest memory is largely backed by anonymous mmap regions; if KSM or memory ballooning is active, host RSS may differ significantly from the guest's configured RAM.

### Disk I/O
Read from `/proc/<pid>/io`:
- `read_bytes`: bytes read from storage by this QEMU process
- `write_bytes`: bytes written to storage by this QEMU process

Rates are calculated between samples (bytes/s).

> **Note:** `/proc/<pid>/io` requires that you own the process **or** have `CAP_SYS_PTRACE`. If permission is denied, columns show `N/A`. Running as root or the same user as the VM removes this restriction.

> **Important:** This is **host-side I/O attributed to the QEMU process** — it includes disk image I/O but may also include other I/O (e.g., logging). It does **not** represent the guest filesystem's I/O.

### Network I/O
This is the most complex metric. The tool uses a layered best-effort approach:

1. **Cmdline parsing:** Looks for `-netdev tap,ifname=<if>` or `-net tap,ifname=<if>` in the QEMU command line. This is the most reliable method.
2. **File descriptor scan:** Walks `/proc/<pid>/fd/` for symlinks to `/dev/net/tun`, then identifies tap/vnet interfaces in `/sys/class/net/` that belong to QEMU.
3. **N/A fallback:** If no interface can be mapped, the column shows `N/A` rather than reporting incorrect host-wide traffic.

Once interfaces are identified, rates are read from:
```
/sys/class/net/<interface>/statistics/rx_bytes
/sys/class/net/<interface>/statistics/tx_bytes
```
and differenced between samples.

> **Note:** Method 2 (fd scan) may match *any* tap interface on the host, not just the one belonging to a specific VM, when multiple VMs share a host without explicit `ifname=` in their cmdlines. In that case, `N/A` is safer.

### Uptime
Process start time is read from `/proc/<pid>/stat` field 22 (`starttime`, in jiffies since boot). Host boot time is read from `/proc/uptime`. QEMU process uptime = host uptime − (starttime / CLK_TCK).

> **Note:** This is **QEMU process uptime**, not the guest OS uptime.

---

## Known Limitations

| Limitation | Notes |
|---|---|
| Disk I/O requires process ownership | `/proc/<pid>/io` returns `PermissionError` if you don't own the process. Run as root or the VM owner. |
| Network attribution is best-effort | Only reliable when QEMU is started with explicit `-netdev tap,ifname=...`. VMs using user-mode networking or unnamed tap devices will show `N/A`. |
| CPU is host-side QEMU process CPU | vCPU-level breakdown requires QMP (not in v1). |
| RAM is host-side RSS | Not equal to guest memory. |
| No Windows/macOS support | Linux only (requires `/proc` and `/sys`). |
| First sample has no rates | CPU%, disk rate, and net rate show `--`/`N/A` on the very first refresh since deltas require two samples. |

---

## Running QEMU VMs for Best Compatibility

For best interface attribution in `qemu-top`, launch VMs with explicit `ifname`:

```bash
# Create a tap interface
sudo ip tuntap add dev tap0 mode tap
sudo ip link set tap0 up

# Launch QEMU with explicit ifname
qemu-system-x86_64 \
  -name myvm \
  -m 2048 \
  -hda disk.qcow2 \
  -netdev tap,id=net0,ifname=tap0,script=no,downscript=no \
  -device virtio-net-pci,netdev=net0
```

`qemu-top` will then correctly attribute `tap0` traffic to `myvm`.

---

## Adding QMP Support (Future)

The architecture is designed for easy QMP extension. To add QMP support:

1. Create a `QMPCollector` class that:
   - Auto-discovers QMP sockets by scanning `/proc/<pid>/fd/` for Unix domain sockets matching common patterns (`/run/qemu-*`, `/var/lib/libvirt/qemu/*.sock`, etc.)
   - Connects with a short timeout and never blocks the main loop
   - Falls back gracefully if no socket is found

2. Add a `QMPMetrics` dataclass alongside `VMMetrics` with fields like:
   - `vcpu_count`, `vcpu_usage` (per-vCPU breakdown)
   - `balloon_current`, `balloon_maximum` (guest memory)
   - `block_stats` (per-device I/O from guest perspective)
   - `run_state` (`running`, `paused`, `saved`, etc.)

3. In `MetricsEngine.update()`, merge QMP data into `VMMetrics` when available — keeping the `/proc`-only path as the primary/fallback.

4. Add a QMP detail section to the TUI detail view.

This keeps the `/proc` monitoring core completely independent of QMP.

---

## Self-Test

```bash
qemu-top --self-test
```

Tests:
- VM name resolution from cmdline
- Uptime formatting
- Memory formatting
- Bytes/rate formatting
- CPU delta calculation
- `/proc/<pid>/stat` parsing
- `/proc/<pid>/status` parsing
- `/proc/<pid>/io` parsing
- Network interface name parsing from cmdline
- QEMU process discovery regex
- Host memory reading (`/proc/meminfo`)
- Host uptime reading (`/proc/uptime`)
