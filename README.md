# qemu-lab

A disposable KVM-backed VM sandbox for safely executing untrusted or dangerous student code on Fedora Linux.

---

## The Problem

I teach operating systems. My students write code that calls `fork()`, mmap shared memory, and wrestle with semaphores — exactly the low-level primitives that make an OS tick, and exactly the ones that can make a host machine stop ticking when they go wrong.

After grading a few rounds of the predecessor to my current lab [CECS 326 Lab 2 (Semaphores)](https://github.com/agiacalone/cecs-326-lab-semaphores-revamp) on bare metal, I had been fork-bombed enough times to know that running poorly-tested systemcalls on my laptop directly is a really bad idea. A student's well-intentioned-but-slightly-wrong loop would eat the process table, the machine would crawl, and I would be staring at an unresponsive terminal wondering why I left the field of accounting (I hated it). By the third full reboot during a single grading session, the lesson sunk in.

---

## Why Not Containers?

Containers (Docker, Podman, Distrobox) are the common first answer. They are the wrong answer for this threat model.

Containers share the host kernel. A process inside a container makes the same system calls as a process on the host — the kernel handles them identically. Container isolation is enforced by namespaces and cgroups, which are kernel features. This means:

- A kernel exploit available to a container process is an exploit against the host
- Misconfigured namespace permissions (a common mistake) can allow container escape
- Shared memory and semaphore namespaces, depending on configuration, may bleed across container boundaries
- A fork bomb inside a container can exhaust host PID space if cgroup limits are not carefully set

For general application isolation, containers are a reasonable trade-off. For executing student OS coursework — code that specifically targets low-level kernel interfaces — the shared kernel assumption is the vulnerability.

KVM provides hardware-level isolation. The guest runs on a virtualized CPU with its own kernel, its own memory, and its own IPC namespace. A runaway `fork()` fills the guest's PID table, not the host's. A leaked semaphore is gone when the VM exits. A kernel exploit inside the VM reaches a virtual machine monitor, not bare metal. The guest is entirely disposable.

---

## How It Works

`fedora-sandbox` wraps QEMU/KVM with a minimal workflow designed for repeated, low-friction use:

- **Base image**: A Fedora Cloud qcow2 image downloaded from the official Fedora mirrors and SHA256-verified. It is never written to directly. Development tools and the `sandbox-net` script are baked in at update time.
- **Overlay images**: Each session (sandbox or lax) uses a qcow2 overlay backed by the base image. All writes go to the overlay. In sandbox mode, the overlay is deleted on exit.
- **Cloud-init seeding**: A single NoCloud seed ISO provides credentials, autologin, and first-boot configuration without requiring a network-accessible metadata server.
- **Network control**: Both modes expose a NAT NIC with SSH port forwarding. Both modes start with outbound traffic blocked at boot via `nftables` using the `sandbox-net` script baked into the image. Enable network manually with `sudo sandbox-net enable`.
- **Distrobox awareness**: If the script is invoked from inside a Distrobox container, it automatically re-execs itself on the host via `distrobox-host-exec`. QEMU and KVM run on the Fedora Kinoite host, not inside a container.

---

## Modes

| Mode | Command | Network | Disk | Use case |
|------|---------|---------|------|----------|
| Sandbox (default) | `fedora-sandbox` | Disabled at boot | Disposable overlay, deleted on exit | Grading untrusted submissions |
| Lax | `fedora-sandbox --lax` | Disabled at boot; enable with `sudo sandbox-net enable` | Persistent overlay at `~/.local/share/fedora-sandbox/lax.qcow2` | Development with persistent state |
| Lax reset | `fedora-sandbox --lax --reset` | Disabled at boot; enable with `sudo sandbox-net enable` | Wipes and recreates lax overlay | Starting fresh from the base image |
| Update image | `fedora-sandbox --update-image` | — | Downloads latest Fedora Cloud image | Keeping the base image current |

Both modes have a NAT NIC, but `sandbox-net disable` runs at first boot and drops all new outbound connections via an nftables output chain. Established connections (including any open SSH session) are not affected, so you can copy files in with `scp` even while the network is nominally disabled.

---

## Usage

```bash
# Grade a submission: network disabled, VM gone on exit
fedora-sandbox

# Development or testing: network enabled, state preserved between runs
fedora-sandbox --lax

# Reset lax environment to a clean slate
fedora-sandbox --lax --reset

# Update the base Fedora Cloud image (auto-detects latest version)
fedora-sandbox --update-image

# Update image, then immediately launch lax
fedora-sandbox --update-image --lax
```

**Login:** `fedora` / `sandbox` (auto-login on the serial console)  
**Exit:** `Ctrl-A X`

### Copying files into the VM

SSH port forwarding is active in both modes on port **2222**:

```bash
# Copy a file in
scp -P 2222 submission.c fedora@localhost:~

# Open a shell
ssh -p 2222 fedora@localhost
```

### Controlling network access from inside the VM

```bash
sudo sandbox-net enable    # allow outbound traffic
sudo sandbox-net disable   # block outbound traffic
sudo sandbox-net status    # show current state
```

### Resource overrides

```bash
FEDORA_SANDBOX_CPUS=4 FEDORA_SANDBOX_RAM=4G fedora-sandbox
```

| Variable | Default | Description |
|----------|---------|-------------|
| `FEDORA_SANDBOX_CPUS` | `2` | Number of vCPUs |
| `FEDORA_SANDBOX_RAM` | `2G` | RAM allocation |

The legacy names `SANDBOX_CPUS` and `SANDBOX_RAM` are still accepted as fallbacks.

---

## Dependencies

The following must be available on the host (not inside a container):

| Tool | Purpose |
|------|---------|
| `qemu-system-x86_64` | KVM hypervisor |
| `qemu-img` | Overlay image creation |
| `curl` | Image and checksum download |
| `sha256sum` | Image verification |
| `xorriso` or `genisoimage` | Cloud-init seed ISO generation |
| `virt-customize` | Baking tools and scripts into the base image |

On Fedora Kinoite:

```bash
rpm-ostree install qemu qemu-img xorriso guestfs-tools
```

---

## Storage Layout

All runtime files are stored under `~/.local/share/fedora-sandbox/`:

```
~/.local/share/fedora-sandbox/
├── base.qcow2              # Fedora Cloud base image (never modified directly)
├── lax.qcow2               # Persistent lax overlay (backed by base.qcow2)
├── seed.iso                # Cloud-init NoCloud seed ISO (shared by both modes)
└── sandbox-XXXXXX.qcow2   # Disposable sandbox overlay (deleted on VM exit)
```

The base image is the only file that requires a download. Overlays are created in seconds from it. Deleting `lax.qcow2` resets the lax environment to a clean Fedora install.

After running `--update-image`, any existing `lax.qcow2`, `sandbox-seed.iso`, and `lax-seed.iso` are automatically removed — they are stale against the new base and will be regenerated on next launch.

---

## Security Notes

- Network isolation is enforced inside the VM via `nftables` (the `sandbox-net` script), not at the QEMU level. Both modes start with outbound traffic blocked; the NAT NIC is used for SSH/scp access only.
- The base image is SHA256-verified against the official Fedora checksum file on every download.
- Cloud-init is disabled inside the VM after first boot to prevent re-initialization.
- The guest login password is intentionally weak (`sandbox`). The VM is behind QEMU user-mode NAT and is only reachable via the forwarded SSH port on localhost; the password exists for console and SSH convenience, not security.
- KVM requires hardware virtualization support (`vmx` or `svm` in `/proc/cpuinfo`) and the `kvm` and `kvm_amd`/`kvm_intel` kernel modules loaded on the host.
