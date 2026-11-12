---
title: "ADR-0028: Complete Kernel Network Protocol Stripping (Elimination of PF_INET and PF_INET6)"
status: Accepted
date: 2026-11-12
tags:
  - adr
  - architecture
  - security
  - kernel
  - networking
  - air-gap
  - yocto
  - rpi
deciders:
  - Workspace System Architect
  - Security Software Lead
  - Embedded Linux Engineer
references:
  - "[[Architecture/Decisions/ADR-0025 - Complete U-Boot Network Subsystem Deactivation in Air-Gapped Appliance|ADR-0025: Complete U-Boot Network Subsystem Deactivation]]"
  - "[[Architecture/07 - Air-Gapped RPi HSM OS Architecture|07 - Air-Gapped RPi HSM OS Architecture]]"
  - "[[Specification/HSM Yocto OS Hardening, Clean Console, U-Boot Network Deactivation, and gphsm CLI Alignment|Specification: HSM Yocto OS Hardening]]"
  - "[[00 - Index|Vault Index]]"
---

# ADR-0028: Complete Kernel Network Protocol Stripping (Elimination of PF_INET and PF_INET6)

## Context & Problem Statement

The **air-gapped Raspberry Pi HSM OS** ([`hsm-yocto-offgrid-rpi`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/hsm-yocto-offgrid-rpi)) enforces strict isolation across physical and software boundaries:
1. On-board Wi-Fi and Bluetooth are disabled in the Device Tree and hardware configuration.
2. In [[Architecture/Decisions/ADR-0025 - Complete U-Boot Network Subsystem Deactivation in Air-Gapped Appliance|ADR-0025]], all network drivers, commands (`dhcp`, `ping`, `tftpboot`), and socket subsystems were completely compiled out of U-Boot (`no-net.cfg`).
3. User-space previously deployed `nftables` with a 100% total-drop packet filter rule.

However, kernel security auditing revealed that while physical network interface drivers (Ethernet, Wi-Fi, Bluetooth) were unselected, the core Linux kernel defconfig still compiled in:
- IPv4 protocol suite (`CONFIG_INET=y`)
- IPv6 protocol suite (`CONFIG_IPV6=y`)
- Netfilter and packet filtering infrastructure (`CONFIG_NETFILTER=y`, `CONFIG_NF_TABLES=y`, `CONFIG_NFT_*`)
- IP routing tables, socket buffers (`sk_buff`), connection tracking, and packet classification logic.

In an ultra-hardened cryptographic appliance designed exclusively for offline key generation, PKI signing, and HSM administration, these dormant protocol stacks present latent attack surface and unnecessary overhead:
- **Dead Code Surface**: Complex protocol parsers (TCP state machines, UDP demux, ICMP handling, IPv6 extension headers, ARP) remain compiled into the kernel image.
- **Physical Bus Exploits**: If a malicious USB Ethernet or cellular modem dongle is inserted, the kernel IP layer would attempt to bind and parse packet streams.
- **Firewall Redundancy**: Running `nftables` in user space to block network packets on a system with zero legitimate IP network capability is an architectural anti-pattern and wastes CPU/RAM during boot.

Conversely, the Linux kernel socket infrastructure (`CONFIG_NET=y`) and local UNIX domain sockets (`CONFIG_UNIX=y`, `AF_UNIX`) cannot be stripped, as they are mandatory for system init (`systemd`), hardware enumeration (`udev`), journal logging (`systemd-journald`), and inter-process communication.

---

## Decision Drivers

- **Zero IP Invariant**: Complete elimination of IP networking protocols (`PF_INET`, `PF_INET6`) from the kernel binary image.
- **Dead Code & Vulnerability Elimination**: Eliminate entire vulnerability classes related to TCP/IP and packet processing flaws.
- **IPC Preservation**: Retain local UNIX domain sockets (`AF_UNIX`) and base socket plumbing (`CONFIG_NET=y`) required by `systemd` and `udev`.
- **User-Space Pruning**: Remove redundant firewall user-space utilities (`nftables`, `nftables-rules`) once IP stacks cease to exist.
- **Deterministic Kernel Behavior**: Guarantee that any user-space call to `socket(AF_INET, ...)` or `socket(AF_INET6, ...)` unconditionally fails with `EAFNOSUPPORT` at the syscall entrypoint.

---

## Considered Options

### Option 1: Keep IP Stacks and Rely on Netfilter Firewall (Status Quo)
- **Description**: Maintain `CONFIG_INET=y` and `CONFIG_IPV6=y`, relying on `nftables` to drop all ingress, egress, and forwarded packets.
- **Pros**: Standard desktop/server Linux paradigm.
- **Cons**: Retains over 100,000 lines of kernel networking code; retains memory overhead for routing tables; leaves packet parsers vulnerable to USB network dongle attacks; requires maintaining user-space `nftables` rules and systemd service units.

### Option 2: Disable Netfilter Firewall but Retain IP Stacks
- **Description**: Remove `nftables` user-space and unselect `CONFIG_NETFILTER`, but keep `CONFIG_INET=y`.
- **Pros**: Reduces user-space image size and removes firewall execution overhead.
- **Cons**: Leaves the system vulnerable if a USB network adapter is attached, as the IP stack would be fully functional without even a firewall to drop incoming traffic.

### Option 3: Complete Kernel IP Protocol Stripping (`# CONFIG_INET is not set`) & Retain UNIX Sockets (Selected)
- **Description**: Unset `CONFIG_INET`, `CONFIG_IPV6`, and `CONFIG_NETFILTER` in the kernel defconfig fragment `hardened.cfg`. Retain `CONFIG_NET=y` and `CONFIG_UNIX=y`. Remove `nftables` from image packaging.
- **Pros**:
  - Eliminates `PF_INET` and `PF_INET6` socket families entirely; `socket(AF_INET, ...)` returns `EAFNOSUPPORT`.
  - Kernel cannot parse IP packets under any circumstance, even if a physical network interface were connected.
  - Decreases kernel image size and boot memory footprint.
  - Eliminates the need for user-space firewall services (`nftables`), simplifying the image and accelerating boot.
  - `systemd`, `udev`, and local D-Bus IPC remain fully functional over `AF_UNIX`.
- **Cons**: None. The appliance is strictly air-gapped and requires zero IP communication.

---

## Decision Outcome

**Accepted Option 3**. We completely disable IPv4, IPv6, and Netfilter packet filtering in the Linux kernel defconfig while preserving UNIX domain socket IPC, and purge `nftables` from user-space image recipes.

### Configuration Specifications

1. **Kernel Defconfig Fragment (`meta-hsm-os/recipes-kernel/linux/files/hardened.cfg`)**:
   ```config
   # Disallow IP protocols (IPv4 and IPv6)
   # CONFIG_INET is not set
   # CONFIG_IPV6 is not set

   # Disallow Netfilter / Packet Filtering (obsolete without IP stacks)
   # CONFIG_NETFILTER is not set
   # CONFIG_NF_TABLES is not set
   # CONFIG_NF_TABLES_INET is not set
   # CONFIG_NF_TABLES_NETDEV is not set
   # CONFIG_NFT_REJECT is not set
   # CONFIG_NFT_REJECT_INET is not set
   # CONFIG_NFT_REJECT_IPV4 is not set
   # CONFIG_NFT_REJECT_IPV6 is not set

   # Retain Local UNIX Domain Sockets & Base Socket Infrastructure for IPC (systemd, udev)
   CONFIG_NET=y
   CONFIG_UNIX=y
   ```

2. **Base Image Manifest (`meta-hsm-os/recipes-core/images/hsm-image-base.bb`)**:
   - Remove `nftables` and `nftables-rules` from `IMAGE_INSTALL`.
   - Prune the obsolete recipe directory `meta-hsm-os/recipes-connectivity/nftables/`.

---

## Consequences

### Positive
- **Architectural Air-Gap**: The OS has no conceptual ability to process IP packets. Remote network attacks against IP/TCP/UDP stacks are physically and logically impossible.
- **Attack Surface Minimization**: All kernel IP routing, packet classification, and socket buffer demultiplexing code is omitted from compilation.
- **Boot Performance**: Kernel initialization skips network protocol registration; user space boots faster without `nftables.service`.
- **Resource Footprint**: Smaller kernel footprint and reduced RAM usage.

### Negative / Neutral
- **No IP Diagnostic Tools**: Utilities like `ping`, `curl`, or `ip` cannot function and are not provided. This is desired behavior for an offgrid HSM appliance.
- **Local Loopback (127.0.0.1) Absent**: Any software requiring local TCP loopback (`localhost:PORT`) cannot run; applications must communicate via UNIX domain sockets or character devices (`/dev/tty.hsm*`). `gphsm` and `libgphsm` communicate directly over serial CDC-ACM ports, which is completely unaffected.

---

## Verification & Test Assertions

1. **Kernel Config Validation**:
   - `zcat /proc/config.gz | grep "CONFIG_INET="` &rarr; `# CONFIG_INET is not set`.
   - `zcat /proc/config.gz | grep "CONFIG_UNIX="` &rarr; `CONFIG_UNIX=y`.
   - `zcat /proc/config.gz | grep "CONFIG_NET="` &rarr; `CONFIG_NET=y`.
2. **Syscall Verification**:
   - Attempting to invoke `python3 -c "import socket; socket.socket(socket.AF_INET, socket.SOCK_STREAM)"` raises `OSError: [Errno 97] Address family not supported by protocol` (`EAFNOSUPPORT`).
3. **Image Cleanliness**:
   - `which nft` returns exit code 1 (`not found`).
   - `/etc/nftables.conf` is absent from rootfs.
