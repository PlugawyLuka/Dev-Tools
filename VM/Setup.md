# VM resource guidance — recommended allocations

Purpose
- Suggested VM resource allocations for common guest OS targets when running on a modern high‑spec Linux desktop host. Adjust to available host resources and workload.

Notes
- These are starting recommendations. For heavy workloads (compilation, ML, multiple services) increase CPU/RAM/disk. Use SSD-backed storage and virtio drivers for best results.

1) Windows 11 (guest)
- CPU: 4 cores (2 vCPUs minimum; 4 recommended for responsive UI)
- RAM: 8–16 GB (8 GB minimum for basic use; 16 GB for comfortable multitasking)
- Disk: 80–120 GB (SSD recommended)
- Graphics: virtio‑gpu or passthrough for GPU if available (for heavy GUI/workloads)
- Network: bridged or NAT per need
- Notes: Install virtio drivers (if using KVM) and VMware/VirtualBox guest tools for better integration.

2) Ubuntu Desktop (22.04 / 24.04)
- CPU: 2–4 cores (2 min, 4 recommended)
- RAM: 4–8 GB (4 GB minimum for desktop; 8 GB for comfortable multitasking)
- Disk: 30–60 GB (SSD recommended)
- Notes: Use 2D acceleration, virtio drivers and enable cloud-init for cloud images.

3) Ubuntu Server / headless Linux
- CPU: 1–2 cores (depending on services)
- RAM: 1–4 GB (1 GB minimum for very light services; 2–4 GB typical)
- Disk: 10–40 GB (depends on logs/data)
- Notes: Keep minimal install for server workloads.

4) Raspberry Pi 3 (emulated guest) — QEMU emulation (functional testing only)
- CPU: emulate 1–2 ARM cores (host will emulate — slow)
- RAM: 512 MB – 1 GB (Pi 3 hardware: 1 GB max)
- Disk: 8–32 GB image
- Notes: Emulation is CPU-heavy and slow on x86 hosts. Use for functional testing only; performance results do not reflect real hardware.

5) Raspberry Pi 4 (emulated guest) — QEMU emulation (functional testing only)
- CPU: emulate 2–4 ARM cores
- RAM: 2–8 GB (Pi 4 variants up to 8 GB)
- Disk: 16–64 GB
- Notes: Same caveat — emulation heavy. For performance-sensitive tests, use a real Pi 4.

6) Raspberry Pi 5 (emulation / cross-compile testing)
- CPU: emulate 4‑cores (host emulation)
- RAM: 4–8 GB
- Disk: 16–128 GB
- Notes: Pi 5 is new and full hardware parity is not guaranteed under QEMU; use real hardware for final tests.

Performance & tuning tips
- For best guest disk/network performance use virtio drivers and SSD storage.
- Use paravirtualised NICs and disk interfaces (virtio) in KVM.
- If you plan to run many VMs, consider NUMA/cpu pinning and cgroups to limit resource contention.
- For GUI guests, increase vCPU and RAM and consider GPU passthrough (advanced).

Backup & snapshots
- Keep regular snapshots and external backups of critical VM images.
- Use QCOW2 for snapshot-friendly storage, raw for maximum throughput.

Final note
- Emulated Pi images are valuable for development and CI, but always validate performance and I/O on the actual Raspberry Pi model you intend to deploy to.
