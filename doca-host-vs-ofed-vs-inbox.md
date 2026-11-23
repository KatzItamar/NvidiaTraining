# DOCA-Host vs MLNX_OFED vs inbox drivers

Part of [Module 5: DOCA and DOCA-Host](README.md#module-5-doca-and-doca-host).

A customer server with a ConnectX NIC or a BlueField DPU can run mlx5 from one of three sources. The source decides which code is running, who supports it and how it gets fixed. Customers and support engineers often mix these up, so this page sets out how they differ.

| Name people use | What it really is |
| --- | --- |
| **Inbox** (in-box, in-tree, distro driver) | The mlx5 that ships inside the distro's own kernel and the distro's own rdma-core. Nothing extra is installed |
| **MLNX_OFED** ("MOFED", "OFED") | NVIDIA's older all-in-one driver bundle. It was installed with the `mlnxofedinstall` script. It is now in LTS only |
| **MLNX_EN** | A smaller Ethernet-only version of MLNX_OFED. It is no longer supported. NVIDIA tells those users to switch to inbox |
| **DOCA-Host** | NVIDIA's current host package, installed with `apt` or `dnf`. It comes in **profiles** |
| **DOCA-OFED** | The `doca-ofed` profile of DOCA-Host. It replaces MLNX_OFED one for one, with the same drivers and tools and nothing else from DOCA |

> Sources: NVIDIA's *MLNX_OFED to DOCA-OFED Transition Guide*, *DOCA Profiles* and *DOCA-Host Installation and Upgrade* pages (DOCA 3.x docs, checked Nov 2026). Items marked **(confirm)** are not in the public docs. Ask your team about them.

---

## 1. The big picture: same code, three ways to deliver it

The upstream Linux kernel, rdma-core and DPDK are the **single source**. Every mlx5 feature starts there (see [Module 4](README.md#module-4-mlx5-upstream-clone-read-and-ask-for-changes)). The three options differ only in how that code reaches the customer's server and how old or new it is when it gets there.

```
                 upstream (kernel.org, rdma-core, DPDK)
                    |                           |
        distro vendor backports        NVIDIA backports (mlnx-ofa_kernel + compat)
        into its own kernel            into one source tree that builds on many kernels
                    |                           |
                 INBOX                   MLNX_OFED  ──(2024: frozen, LTS)──►  DOCA-Host
   (RHEL / Ubuntu / SLES kernel +                                      (doca-ofed, doca-roce,
    distro rdma-core packages)                                          doca-networking, doca-all ...)
```

| | Inbox | MLNX_OFED (legacy) | DOCA-Host |
| --- | --- | --- | --- |
| **Who builds and ships it** | The distro vendor (Red Hat, Canonical, SUSE) | NVIDIA | NVIDIA |
| **Who supports it first** | The distro vendor, with NVIDIA as their partner | NVIDIA | NVIDIA |
| **Kernel driver source** | Upstream mlx5 as of the distro's kernel, plus fixes and features the vendor backports | `mlnx-ofa_kernel`, a backport of recent upstream mlx5 and RDMA core | `mlnx-ofa_kernel`, the same idea and a newer version |
| **How new is the driver** | As old as the distro kernel (RHEL 8 is based on 4.18, with many backports) | Close to upstream when released | Close to upstream when released |
| **Replaces kernel modules?** | No, these are the kernel's own modules | Yes: `mlx5_core`, `mlx5_ib`, `ib_core` and the rest of the RDMA stack | Yes, the same (for profiles that include the kernel part) |
| **User space** | Distro `rdma-core`, `perftest`, `iproute2`, `ethtool` | NVIDIA builds of rdma-core (`...mlnx` versions), MFT, `mlnx-tools` | The same, plus optional DOCA libraries, OVS-DOCA, MLNX-DPDK |
| **Install method** | Nothing to do: `dnf install rdma-core` and so on | `./mlnxofedinstall` script from an ISO or tarball | Standard repo and package manager: `apt install doca-ofed` |
| **Kernel modules built** | By the distro, signed by the distro | Prebuilt per kernel, or rebuilt with `mlnx_add_kernel_support.sh` | **DKMS**, built on the customer's host and signed with a locally made key |
| **Secure Boot** | Works out of the box (vendor-signed) | NVIDIA-signed prebuilt modules | You must enroll the DKMS MOK key (`mokutil --import /var/lib/dkms/mok.pub`) |
| **Survives a kernel update** | Yes, the driver *is* the kernel | No. Reinstall or rebuild for the new kernel | Yes. DKMS rebuilds automatically |
| **New hardware support** | Only when the distro picks up the new kernel code | Ended. No new features after the last release | Yes. All new features go here. **BlueField-3 needs DOCA-Host**, not MLNX_OFED |
| **Status** | Always there | LTS only. **Support ends October 2027** | Current product |

---

## 2. MLNX_OFED → DOCA-Host: what changed

### 2.1 Timeline

| Date | Event |
| --- | --- |
| Oct 2024 | Last standalone MLNX_OFED release (24.10 LTS). MLNX_OFED and MLNX_EN move to LTS-only, with 3 years of critical and security fixes |
| Jan 2025 | All new features go into DOCA-OFED / DOCA-Host only |
| Oct 2027 | MLNX_OFED end of life: no more support or updates from NVIDIA |

### 2.2 What stays the same

- The **kernel drivers**: DOCA-OFED's kernel part is still the `mlnx-ofa_kernel` package, the same code base that MLNX_OFED used.
- The **user-space stack and tools**: rdma-core (NVIDIA build), perftest, `mlnx-tools`, MFT (`mst`, `mlxconfig`, `mlxfwmanager`), `ofed_info`, and the `openibd` service.
- The **supported OSes and apps**: the transition guide states DOCA-OFED supports what MLNX_OFED supported.

### 2.3 What changes

| Topic | MLNX_OFED | DOCA-Host |
| --- | --- | --- |
| Install | One big script: `./mlnxofedinstall --add-kernel-support ...` | Add the DOCA repo (local repo `.deb`/`.rpm` or online repo), then `apt`/`dnf`/`zypper install <profile>` |
| Choosing what you get | Script flags (`--basic`, `--dpdk`, `--upstream-libs` ...) | **Profiles** (meta-packages), see section 3 |
| Kernel modules | Prebuilt for each supported kernel. Unsupported kernels needed `mlnx_add_kernel_support.sh` | Built by **DKMS** on the host. Needs `kernel-devel`/headers matching `uname -r` and the same gcc |
| Custom kernels | `mlnx_add_kernel_support.sh` | DKMS does it automatically. `doca-extra` → `/opt/mellanox/doca/tools/doca-kernel-support` can still build a `doca-kernel-repo` package for a whole fleet |
| Module signing | NVIDIA-signed prebuilt modules | Not NVIDIA-signed. DKMS signs with a local key, which you enroll in MOK |
| Minor OS repos | Per minor release (RHEL 9.3, 9.4, ...) | One repo per **major** release (RHEL 9) |
| Upgrade | Uninstall and reinstall | `apt install` / `dnf upgrade <profile>` from DOCA 2.6.0 on. From 2.5.x and older you must remove everything first |
| Version check | `ofed_info -s` | `ofed_info -s` still works. Also `/opt/mellanox/doca/tools/doca-info` |
| Firmware updater | Bundled | Separate package: `mlnx-fw-updater` |
| Lightweight Ethernet only | MLNX_EN | Not offered. Use **inbox**, or the `doca-host-basic` / `doca-roce` profile |

### 2.4 Migration steps (what a customer does)

```bash
# 1. Remove MLNX_OFED (the uninstall script ships with it)
sudo /usr/sbin/ofed_uninstall.sh --force

# 2. Add the DOCA-Host repo for the distro (downloaded from developer.nvidia.com/doca-downloads)
sudo dpkg -i doca-host_<ver>_<distro>_amd64.deb && sudo apt-get update      # Ubuntu/Debian
# sudo rpm -Uvh doca-host-<ver>.<distro>.rpm && sudo dnf makecache           # RHEL-like

# 3. Install the profile that matches MLNX_OFED
sudo apt install -y doca-ofed mlnx-fw-updater

# 4. Load the new drivers (or reboot) and check
sudo /etc/init.d/openibd restart
ofed_info -s
modinfo mlx5_core | grep -E '^(filename|version)'
```

---

## 3. DOCA-Host profiles

A profile is a meta-package. You install one and it pulls in a set of packages.

| Profile | For | Includes | Kernel part? |
| --- | --- | --- | --- |
| `doca-all` | Everything, recommended for BlueField hosts | MLNX_OFED set + DOCA Core and all DOCA libraries + MLNX-DPDK + OVS-DOCA + DOCA Flow | Yes |
| `doca-networking` | Networking without the full DOCA SDK, recommended for ConnectX | MLNX_OFED set + DOCA Core + DOCA-PCC + OVS-DOCA + DOCA Flow + DOCA Perftest | Yes |
| `doca-ofed` | **The MLNX_OFED replacement** | MLNX_OFED drivers and tools only | Yes |
| `doca-roce` | RoCE only, no InfiniBand extras | rdma-core, ofed-scripts, mlnx-tools, `mlnx-ofa_kernel`, perftest | Yes |
| `doca-host-basic` | Lightweight Ethernet | rdma-core, mlnx-tools, `mlnx-ofa_kernel`, perftest, mlnx-ethtool, mlnx-iproute2 (no MFT or fw-updater) | Yes |

Add-ons: `doca-libvma`, `doca-libxlio` (socket acceleration), `hpcx` (MPI/UCX/SHARP/NCCL plugins), `mlnx-nvme-dkms`, `mlnx-nfsrdma-dkms` (storage over RDMA built against the DOCA kernel stack).

**DOCA user space over inbox kernel drivers.** DKMS applies to the four profiles with a kernel part. NVIDIA's docs say that "user-space-only metapackages are not affected", so you can run DOCA libraries on top of the distro's own mlx5. Which DOCA versions support which inbox kernels is in the release notes' support matrix **(confirm the exact package names with your team)**. This is the middle ground many cloud and Kubernetes customers want: no out-of-tree kernel modules, but still DOCA user space.

---

## 4. How DOCA-Host replaces the inbox driver on a running system

This is the part customers find surprising. When you install a DOCA-Host profile that includes the kernel part, **the whole RDMA and mlx5 kernel stack is replaced**. That includes `mlx5_core` and `mlx5_ib`, and also `ib_core`, `ib_uverbs`, `rdma_cm`, `ib_ipoib` and any other module that depends on them.

### 4.1 Why it must replace `ib_core` too

Kernel modules link to each other by exported symbols and by **struct layouts**. `mlx5_ib` from a newer code base expects a newer `ib_core` (new fields in `struct ib_device`, new verbs ops). Mixing a new `mlx5_ib` with the distro's older `ib_core` would crash or fail to load. So `mlnx-ofa_kernel` ships its **own copy of the RDMA core subsystem** together with the drivers. As a result, any other out-of-tree module that uses RDMA (Lustre, BeeGFS, GPFS, a vendor's storage driver, `nvidia-peermem`) **must be rebuilt against the DOCA-Host headers** and not against the distro's. NVIDIA ships its own `mlnx-nvme` and `mlnx-nfsrdma` for the same reason.

### 4.2 How the kernel picks the DOCA modules and not the inbox ones

```
/lib/modules/$(uname -r)/
├── kernel/drivers/net/ethernet/mellanox/mlx5/core/mlx5_core.ko   ← inbox (still on disk)
└── updates/dkms/mlx5_core.ko                                      ← DOCA-Host (DKMS build)
```

`depmod` ranks the `updates/` directory above `kernel/`, so `modprobe mlx5_core` loads the DOCA copy. The inbox file stays on disk. Remove the DOCA packages, run `depmod`, and the inbox driver comes back. The `openibd` service (`/etc/init.d/openibd`) loads and unloads the stack in the right order. The initramfs must be rebuilt (`dracut -f` / `update-initramfs -u`) so that early boot does not load the inbox copy first.

### 4.3 How to tell which one is running

| Check | Inbox | DOCA-Host / MLNX_OFED |
| --- | --- | --- |
| `modinfo mlx5_core \| grep filename` | `.../kernel/drivers/net/ethernet/mellanox/...` | `.../updates/dkms/...` (or `.../extra/mlnx-ofa_kernel/...` in older MLNX_OFED) |
| `modinfo mlx5_core \| grep ^version` | No version line, or the kernel version | An OFED version such as `24.10-...` |
| `ethtool -i <iface>` → `version:` | The kernel version (e.g. `5.14.0-427...`) | The OFED version |
| `lsmod \| grep mlx_compat` | Nothing | `mlx_compat` is loaded (see section 5) |
| `ofed_info -s` | Command not found | `MLNX_OFED_LINUX-24.10-...` (DOCA-Host reports the OFED base it contains) |
| `rpm -q rdma-core` / `dpkg -l rdma-core` | Distro version (e.g. `48.0-1.el9`) | NVIDIA version with `mlnx` in it (e.g. `2407mlnx52`) |
| `dkms status` | No mlnx entries | `mlnx-ofed-kernel/...: installed` |

---

## 5. The compat layer: how one source tree builds on many kernels

This is the engineering core of DOCA-Host, and most of your backport team's work is here.

### 5.1 The problem

Upstream mlx5 is written for **one** kernel: the current mainline. It freely uses the newest kernel APIs. A function that changed its arguments in 6.8, a new `page_pool` helper, a new `devlink` op, a renamed `netdev` field. Upstream does not care about older kernels. When an internal API changes, the person who changes it fixes every caller in the tree in the same patch series.

DOCA-Host must take that same driver and build it on **dozens of kernels**: RHEL 8 (4.18), RHEL 9 (5.14), SLES 15, Ubuntu 20.04/22.04/24.04 (5.4 … 6.8), Debian, Oracle UEK, and also any custom kernel a customer builds with DKMS. Each one is missing different APIs. Enterprise kernels make it worse because Red Hat or SUSE backport *some* new APIs into an old version number, so "kernel ≥ 5.15" says nothing reliable about what exists.

### 5.2 The solution: test, don't guess

`mlnx-ofa_kernel` copies an idea from the kernel's own [backports project](https://backports.docs.kernel.org/documentation/compat.html): **do not check version numbers. Probe for every feature.**

1. **`configure` probes the target kernel.** Before building, a large set of autoconf checks (under `compat/config/`) each try to compile a tiny test program against the target kernel's headers. Example: "does `struct net_device_ops` have `ndo_get_stats64`?", "does `devlink_register()` take one or two arguments?". Each check that compiles defines a macro like `HAVE_DEVLINK_REGISTER_GET_1_PARAM` in a generated `config.h`. There are thousands of these.
2. **The driver code is guarded with `#ifdef HAVE_...`.** In the `mlnx-ofa_kernel` copy of mlx5, the places where kernels differ look like this (simplified):

   ```c
   #ifdef HAVE_DEVLINK_REGISTER_GET_1_PARAM
       devlink_register(devlink);
   #else
       devlink_register(devlink, dev->device);
   #endif
   ```

   Older generations kept these as separate patch files under `backports/`, applied at `configure` time (`--without-backport-patches` skipped them). Newer trees mostly carry the `#ifdef`s inline in the source **(confirm which style your release uses)**.
3. **Missing helpers are filled in by compat headers and the `mlx_compat` module.** When an old kernel lacks a small helper the driver calls, `include/linux/*.h` overrides in the tree provide an inline version, or `compat/*.c` implements it. Those `.c` files build into **`mlx_compat.ko`**, which loads before everything else in the stack. That is why `lsmod` shows `mlx_compat` only on DOCA-Host and MLNX_OFED systems.
4. **The result:** one source tree, one release, built by DKMS on the customer's exact kernel, with only the code paths that kernel can support.

### 5.3 What this means in practice

| Effect | Why it matters to you |
| --- | --- |
| Every upstream patch that touches a changed API needs compat work before it lands in DOCA-Host | The backport is not a `git cherry-pick`. Effort grows with the distance from upstream and with the number of supported kernels |
| Each new distro kernel (RHEL 9.N, a new Ubuntu HWE) can break the build | New `HAVE_` checks are needed. This is a constant stream of work, driven by distro calendars and not by NVIDIA |
| A feature can be "in DOCA-Host" but silently compiled out on an old kernel | Feature support differs per OS. Ask for the per-OS feature matrix |
| Bugs can live in the compat code itself (wrong `#ifdef`, a bad helper) | A bug that reproduces on DOCA-Host but not upstream (or not inbox) is often a compat bug. Triage should ask this first |
| DKMS build failures on customer machines | Usually missing headers, a gcc mismatch, or a kernel newer than the release supports. These are among the most common support tickets |
| Out-of-tree modules must build against DOCA's RDMA headers | Partners (storage, GPU) need clear guidance. This often comes back to your team |

---

## 6. Inbox in depth: what customers get without NVIDIA's package

### 6.1 How mlx5 reaches the inbox driver

1. NVIDIA engineers merge mlx5 code **upstream** (Module 4).
2. A distro picks a kernel base (RHEL 9 = 5.14, Ubuntu 24.04 = 6.8, SLES 15 SP6 = 6.4).
3. For each minor release, the distro vendor **backports** selected upstream mlx5 and RDMA patches into its kernel. They do this by hand with their own tools, not with `mlnx-ofa_kernel`. NVIDIA works with the vendors on which features and fixes to pull **(confirm: ask who owns the "inbox / distro partner" relationship in your org)**.
4. The distro also ships its own `rdma-core`, which contains `libmlx5`, and its own DPDK packages.

### 6.2 Inbox vs DOCA-Host: the trade-off

| | Inbox wins | DOCA-Host wins |
| --- | --- | --- |
| Support and certification | One vendor to call, and the stack is certified with the OS | NVIDIA supports the whole stack directly, so there is one owner for driver bugs |
| Operations | No extra packages. Survives kernel updates. Secure Boot just works. No DKMS on production hosts | DKMS handles kernel updates, but it adds build tools on every host |
| Features | Only what the distro backported, which can lag by months or years | The latest mlx5, RDMA, steering, offloads, new NICs (ConnectX-8, BlueField-3), DOCA services |
| Interop with other modules | Every other driver is built against the same kernel | Out-of-tree RDMA users must be rebuilt against DOCA headers |
| Tools | `ethtool`, `devlink`, `rdma`, `mstflint` (open source) | Also MFT (`mlxconfig`, `mlxlink`, `mlxfwmanager`), `mlnx_qos`, `show_gids`, `ofed_info` and others |

**Typical customer choices**

- **Hyperscalers and Kubernetes clusters** prefer inbox (or inbox kernel with DOCA user space). They don't want out-of-tree modules.
- **AI and HPC clusters** (DGX, SuperPOD, InfiniBand fabrics) use DOCA-Host (`doca-ofed` or `doca-all`). They need the newest RDMA features, SHARP, GPUDirect and HPC-X. In Kubernetes this is often the NVIDIA **Network Operator**, which installs the DOCA-Host driver as a container (DOCA-OFED driver container).
- **Enterprise on RHEL/SLES** often start inbox and move to DOCA-Host when they need a fix or feature the distro hasn't backported yet.

### 6.3 The key questions for your team

- [ ] For the current DOCA-Host release, which **upstream kernel** is `mlnx-ofa_kernel` based on, and how far behind mainline is it?
- [ ] How many patches are **DOCA-Host only** (not upstream)? How many are compat-only, and how many are real features or fixes not yet upstream?
- [ ] Which distro kernels are supported in this release, and who adds the next RHEL/Ubuntu kernel when it comes out?
- [ ] How do we test compat across kernels (the CI matrix)? How long does a full matrix run take?
- [ ] For each important feature: is it in upstream, inbox RHEL/Ubuntu/SLES, and DOCA-Host? (Keep one table per feature: **upstream version / inbox distro versions / DOCA-Host version**.)
- [ ] What share of support tickets are DKMS or install problems, and what share are compat bugs?
- [ ] Who talks to Red Hat, Canonical and SUSE about inbox backports, and how do we push a fix into inbox?
- [ ] How many customers are still on MLNX_OFED, and what is the plan to move them before **October 2027**?

---

## 7. Lab: see all three on one machine

On a lab server with a ConnectX NIC (a RHEL 9 or Ubuntu 22.04/24.04 VM with a VF passed through also works):

```bash
# A. Inbox: before installing anything from NVIDIA
modinfo mlx5_core | grep -E '^(filename|version|vermagic)'
ethtool -i <iface>
rpm -q rdma-core || dpkg -l rdma-core
lsmod | grep -E 'mlx5|ib_core|mlx_compat'

# B. DOCA-Host: install the MLNX_OFED-equivalent profile
sudo apt install -y doca-ofed       # after adding the DOCA repo, see section 2.4
sudo /etc/init.d/openibd restart
dkms status
modinfo mlx5_core | grep -E '^(filename|version)'   # now under updates/dkms
lsmod | grep mlx_compat                              # now present
ofed_info -s

# C. Back to inbox
sudo apt remove --purge -y doca-ofed && sudo /usr/sbin/ofed_uninstall.sh --force
sudo depmod -a && sudo update-initramfs -u && sudo reboot
```

Write down, for each step, the driver path, the version, and whether `mlx_compat` is loaded. Then get the `mlnx-ofa_kernel` source (DOCA source packages, or `/usr/src/mlnx-ofa_kernel-*/` after a DKMS install). Run `grep -rc "#ifdef HAVE_" drivers/net/ethernet/mellanox/mlx5/core | sort -t: -k2 -n | tail`. This shows which files carry the most compat work.

---

## 8. Self-check

1. A customer on RHEL 9.4 says "the mlx5 driver" has a bug. What three commands tell you whether they run inbox or DOCA-Host?
2. Why does installing `doca-ofed` replace `ib_core`, not just `mlx5_core`?
3. Why does `mlnx-ofa_kernel` test for features (`HAVE_...`) instead of checking `LINUX_VERSION_CODE`?
4. A storage partner's RDMA module stops loading after the customer installs DOCA-Host. What is the likely cause, and what is the fix?
5. A customer wants the smallest NVIDIA-supported Ethernet stack, which used to be MLNX_EN. What are their two options now?
6. When does MLNX_OFED support end, and which device family already requires DOCA-Host?

## Read

- [MLNX_OFED to DOCA-OFED Transition Guide](https://docs.nvidia.com/doca/sdk/nvidia-mlnx-ofed-to-doca-ofed-transition-guide.pdf)
- [DOCA Profiles](https://docs.nvidia.com/doca/sdk/nvidia+doca+profiles/index.html)
- [DOCA-Host Installation and Upgrade](https://docs.nvidia.com/doca/sdk/doca-host-installation-and-upgrade/index.html) and the DKMS management guide linked from it
- [Adapter Software (MLNX_OFED / MLNX_EN LTS status)](https://docs.nvidia.com/networking/software/adapter-software/index.html)
- [Kernel backports project: the compat idea](https://backports.docs.kernel.org/documentation/compat.html)
