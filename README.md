# Raspberry Pi 5 NVMe freeze / SSH "Connection closed by … port 22": Host Memory Buffer fix

A Raspberry Pi 5 booting from a DRAM-less NVMe SSD can become unresponsive after a few hours: SSH is refused with `Connection closed by <ip> port 22` while the device still answers pings. The cause is the default 32 MiB Host Memory Buffer (HMB) cap, and the fix is one line in `cmdline.txt`.

## Symptoms

- The Pi runs normally, then hangs after several hours (~10–12 h).
- SSH fails with `Connection closed by <ip> port 22`, or with `ssh -v`, `kex_exchange_identification: Connection closed by remote host`.
- The device still responds to ping, and closed TCP ports return "connection refused" immediately. The network stack is up, but the system is unresponsive.
- `journalctl` stops logging at a single timestamp until reboot.
- `vcgencmd get_throttled` returns `0x0` (not undervoltage).
- Only a power cycle or forced reboot recovers it.

## Cause

DRAM-less NVMe drives have no onboard cache and instead use a Host Memory Buffer (HMB), a region of system RAM. Such a drive reports a minimum HMB size (64 MiB in our case). The Pi 5 caps HMB at 32 MiB by default. When a drive's minimum exceeds the cap, the kernel disables HMB entirely rather than allocating a smaller buffer, so the drive operates with no cache and eventually stalls under sustained I/O.

The kernel log shows:

```
nvme nvme0: min host memory (64 MiB) above limit (32 MiB).
```

with no `allocated … host memory buffer` line.

## Confirm

```bash
# Minimum HMB; divide by 256 for MiB. A value above 32 MiB indicates the issue.
sudo nvme id-ctrl /dev/nvme0 | grep hmmin

# Absence of an "allocated ... host memory buffer" line means HMB is disabled.
sudo dmesg | grep -i 'host mem'
```

## Fix

`cmdline.txt` is a single line; append the parameter without adding a newline.

```bash
# Pi OS Bookworm and later: /boot/firmware/cmdline.txt   (older: /boot/cmdline.txt; and in case of our NTP server with custom kernel (i.e., https://github.com/by/RT-Kernel): /boot/firmware/NTP/cmdline.txt)
sudo sed -i 's/$/ nvme.max_host_mem_size_mb=128/' /boot/firmware/cmdline.txt
sudo reboot
```

Verify after reboot:

```bash
sudo dmesg | grep -i 'host mem'
# expected: nvme nvme0: allocated 64 MiB host memory buffer (2 segments).
```

`128` raises the ceiling; the drive uses only what it requests (64 MiB), so `=64` is equivalent. `cmdline.txt` is not overwritten by kernel or firmware updates.

## Is it a fault of Pi or SSD?

Per the NVMe specification a drive must remain functional with no HMB, so one that fails without it is non-compliant. The 32 MiB default only exposes the problem; this parameter is a practical workaround. Reporting it to the SSD vendor is also worthwhile.

## Hardware

- Raspberry Pi 5
- Corsair MP600 GS (DRAM-less, Phison PS5021-E21 controller)
- Pimoroni NVMe Base (passive M.2 carrier)

But any DRAM-less drive reporting an HMB minimum above 32 MiB is likely affected.

## Background

The 32 MiB default is set in the Raspberry Pi kernel device tree (commit `26e6d69`). Related discussion: [raspberrypi/linux#7445](https://github.com/raspberrypi/linux/issues/7445).
