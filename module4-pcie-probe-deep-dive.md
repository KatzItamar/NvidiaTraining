# PCIe in Linux: Host Bridge Probe, Enumeration and Driver Probe

A training guide to how Linux finds PCIe hardware and binds drivers to it, from the
host bridge all the way to `mlx5_core`'s `probe_one()`. NVIDIA ConnectX is used as
the running example.

This guide ends where [mlx5 core objects deep dive](module4-mlx5-core-objects-deep-dive.md) begins: at the
first line of `probe_one()`.

**Contents**

1. [The big picture](#1-the-big-picture)
2. [PCIe basics you need](#2-pcie-basics-you-need)
3. [Stage 1: Probing the host bridge](#3-stage-1-probing-the-host-bridge)
4. [Stage 2: Enumeration (scanning the buses)](#4-stage-2-enumeration-scanning-the-buses)
5. [Stage 3: Resource assignment](#5-stage-3-resource-assignment)
6. [Stage 4: Adding devices and matching drivers](#6-stage-4-adding-devices-and-matching-drivers)
7. [Stage 5: The driver probe (`mlx5_core`)](#7-stage-5-the-driver-probe-mlx5_core)
8. [Bonus: SR-IOV VFs are enumerated at runtime](#8-bonus-sr-iov-vfs-are-enumerated-at-runtime)
9. [Full call chain](#9-full-call-chain)
10. [Seeing it on a real system](#10-seeing-it-on-a-real-system)
11. [Cheat sheet](#11-cheat-sheet)

Source files referenced (paths relative to the kernel tree):

| File | What is there |
|---|---|
| `drivers/acpi/pci_root.c` | x86/ACPI host bridge driver: `acpi_pci_root_add()` |
| `arch/x86/pci/acpi.c` | `pci_acpi_scan_root()` (x86) |
| `drivers/pci/controller/pci-host-generic.c` | Device-tree generic ECAM host driver (`gen_pci_driver`) |
| `drivers/pci/controller/pci-host-common.c` | `pci_host_common_probe()`, `pci_host_common_init()` |
| `drivers/pci/probe.c` | Enumeration: `pci_host_probe()`, `pci_scan_*()`, `pci_setup_device()`, `pci_device_add()` |
| `drivers/pci/bus.c` | `pci_bus_add_devices()`, `pci_bus_add_device()` |
| `drivers/pci/pci-driver.c` | `pci_bus_type`, `pci_bus_match()`, `pci_device_probe()`, `__pci_register_driver()` |
| `drivers/pci/iov.c` | SR-IOV: `pci_enable_sriov()`, `pci_iov_add_virtfn()` |
| `drivers/net/ethernet/mellanox/mlx5/core/main.c` | `mlx5_core_pci_table`, `mlx5_core_driver`, `probe_one()`, `mlx5_pci_init()` |

> Code excerpts are simplified. Function names and file locations were checked
> against the tree this guide was written from (v7.4-rc1 era). Example addresses
> and IDs in sample output are illustrative.

---

## 1. The big picture

**Terminology:** in kernel code, "probe" has two meanings. The *host bridge driver
is probed* (stage 1), and later the *endpoint driver is probed* (stage 5). The step
in between, where the kernel discovers what is on the bus, is **enumeration**.

```
 ┌──────────────────────────────────────────────────────────────────────────────────┐
 │ Stage 1  HOST BRIDGE PROBE                                                       │
 │   Firmware (ACPI / Device Tree) describes the PCIe Root Complex.                 │
 │   A host bridge driver binds to it and creates struct pci_host_bridge            │
 │   + the root struct pci_bus (bus 00).                                            │
 ├──────────────────────────────────────────────────────────────────────────────────┤
 │ Stage 2  ENUMERATION  (find out what is plugged in)                              │
 │   The kernel does not know what hardware exists, so it tries every address:      │
 │   on bus 00, for device 0..31 (and its functions 0..7), it reads the first       │
 │   4 bytes of that address's config space (the Vendor ID and Device ID).          │
 │     0xffffffff            -> nothing at this address, try the next one           │
 │     a real ID (15b3:101d) -> a device is there: create a struct pci_dev          │
 │   If the device found is a bridge (for example a root port), there is another    │
 │   bus behind it: give that bus a number, create a struct pci_bus for it, and     │
 │   scan it the same way. Repeat until there are no more bridges.                  │
 ├──────────────────────────────────────────────────────────────────────────────────┤
 │ Stage 3  RESOURCE ASSIGNMENT                                                     │
 │   Size every BAR, assign MMIO / IO addresses, program bridge windows.            │
 ├──────────────────────────────────────────────────────────────────────────────────┤
 │ Stage 4  ADD DEVICES + MATCH DRIVERS                                             │
 │   Each pci_dev is made visible to the driver core.                               │
 │   pci_bus_match() compares vendor/device ID with every PCI driver's id_table.    │
 ├──────────────────────────────────────────────────────────────────────────────────┤
 │ Stage 5  DRIVER PROBE                                                            │
 │   Match found -> pci_device_probe() -> drv->probe(pdev, id)                      │
 │   For a ConnectX: mlx5_core's probe_one(pdev, id).                               │
 └──────────────────────────────────────────────────────────────────────────────────┘
```

---

## 2. PCIe basics you need

### 2.1 Topology

PCIe is a tree of point-to-point links. Software still sees it as PCI buses
connected by bridges.

```
                              CPU / memory
                                   │
                    ┌──────────────┴──────────────┐
                    │   Root Complex (RC)         │   ◄── "host bridge" in Linux
                    │   (struct pci_host_bridge)  │
                    └──────────────┬──────────────┘
          bus 00 (root bus, struct pci_bus)
     ─────────┬──────────────────────┬────────────────────────
              │                      │
     ┌────────┴────────┐    ┌────────┴────────┐
     │ Root Port       │    │ Root Port       │   ◄── each is a PCI-to-PCI bridge
     │ 0000:00:01.0    │    │ 0000:00:02.0    │       (type 1 header), a struct pci_dev
     └────────┬────────┘    └────────┬────────┘
              │ PCIe link            │ PCIe link
        bus 3b (child pci_bus)       bus 5e
              │                      │
     ┌────────┴──────────────────┐   └── NVMe SSD, ...
     │ ConnectX-6 Dx  (endpoint) │
     │  fn 0: 0000:3b:00.0  PF0  │   ◄── each function is its own struct pci_dev
     │  fn 1: 0000:3b:00.1  PF1  │       (type 0 header)
     └───────────────────────────┘
```

A dual-port ConnectX usually appears as **two PCI functions** (one PF per port) on
the same device number. Each gets its own `pci_dev` and its own `mlx5_core` probe.

### 2.2 Addressing: Domain:Bus:Device.Function (BDF)

```
   0000 : 3b : 00 . 0
   ────   ──   ──   ─
    │      │    │   └─ function  0..7    (3 bits)
    │      │    └───── device    0..31   (5 bits)
    │      └────────── bus       0..255  (8 bits)
    └───────────────── domain / segment (one per host bridge hierarchy)
```

In the kernel, device + function are packed into `devfn`:
`PCI_DEVFN(dev, fn)`, `PCI_SLOT(devfn)`, `PCI_FUNC(devfn)`.

### 2.3 Configuration space

Every function has a configuration space: 256 bytes for PCI, 4 KB for PCIe. It is
how software discovers and configures the device **before** any driver exists.

```
 Offset  Type 0 header (endpoint, e.g. ConnectX)
 ──────  ──────────────────────────────────────────────────────────────
 0x00    Vendor ID (0x15b3 = Mellanox)  │ Device ID (0x101d = CX-6 Dx)
 0x04    Command                        │ Status
 0x08    Revision ID │ Class code (0x020000 = Ethernet controller)
 0x0C    Cache line  │ Latency │ Header type (bit7 = multi-function) │ BIST
 0x10    BAR0   ┐
 0x14    BAR1   │  Base Address Registers: where the device's
 0x18    BAR2   │  registers live in MMIO space.
 0x1C    BAR3   │  ConnectX uses BAR0 (64-bit, so BAR0+BAR1)
 0x20    BAR4   │  for its init segment and doorbells.
 0x24    BAR5   ┘
 0x2C    Subsystem Vendor ID │ Subsystem ID
 0x34    Capabilities pointer ─► MSI, MSI-X, PCIe, Power Mgmt, ...
 0x3C    Interrupt line │ Interrupt pin
 0x100+  Extended capabilities (PCIe only): AER, SR-IOV, ATS, ...
```

Type 1 headers (bridges) replace BAR2..5 with **primary / secondary / subordinate
bus numbers** and **memory/IO windows**. That is what lets enumeration walk downwards.

### 2.4 How the CPU reaches config space: ECAM

On modern systems, config space is memory-mapped (ECAM, Enhanced Configuration
Access Mechanism). Firmware tells the kernel the ECAM base (ACPI `MCFG` table, or
DT `reg` property). Then:

```
 address = ECAM_base + (bus << 20) + (device << 15) + (function << 12) + register
```

The host bridge driver provides `struct pci_ops { .read, .write }` that implement
this. Every config access in the kernel (`pci_read_config_dword()` and friends)
ends up in those ops.

### 2.5 The key kernel structures

| Structure | Represents | Created in |
|---|---|---|
| `struct pci_host_bridge` | The Root Complex: config ops, address windows, root bus | Stage 1 |
| `struct pci_bus` | One PCI bus: list of devices, parent bridge, child buses | Stage 1 (root), Stage 2 (children) |
| `struct pci_dev` | One PCI function: IDs, BARs, capabilities, embedded `struct device` | Stage 2 |
| `struct pci_driver` | A driver: `id_table`, `probe`, `remove`, ... | Driver's `module_init` |
| `struct pci_device_id` | One entry of a driver's match table | Static in the driver |

```
 pci_host_bridge
   ├─ ops ──► pci_ops { read, write }   (ECAM accessors)
   ├─ windows  (MMIO / IO / bus-number ranges)
   └─ bus ──► pci_bus (00)
                ├─ devices: list of pci_dev
                │     ├─ 00:01.0 root port ── subordinate ──► pci_bus (3b)
                │     │                                         ├─ devices:
                │     │                                         │   ├─ 3b:00.0 (CX PF0)
                │     │                                         │   └─ 3b:00.1 (CX PF1)
                │     │                                         └─ self ──► 00:01.0
                │     └─ 00:02.0 root port ── subordinate ──► pci_bus (5e) ...
                └─ children: list of child pci_bus
```

---

## 3. Stage 1: Probing the host bridge

The PCI subsystem cannot scan anything until something tells it *where* the Root
Complex is and *how* to reach config space. That comes from firmware, and the code
path depends on the platform.

### 3.1 x86 servers (ACPI)

Firmware describes each root bridge in ACPI as a device with HID `PNP0A08` (PCIe)
or `PNP0A03` (PCI). The ACPI scan handler `pci_root_handler` in
`drivers/acpi/pci_root.c` attaches to it:

```
acpi_pci_root_add(device)                              drivers/acpi/pci_root.c
  ├─ read _SEG (domain), _BBN / _CRS (bus range, MMIO/IO windows)
  ├─ negotiate OS control of PCIe features (_OSC: AER, hotplug, PME, ...)
  ├─ root->bus = pci_acpi_scan_root(root)              arch/x86/pci/acpi.c
  │     └─ acpi_pci_root_create(root, &acpi_pci_root_ops, ...)
  │           ├─ pci_create_root_bus(..., pci_root_ops, ...)
  │           │     ├─ pci_alloc_host_bridge()
  │           │     └─ pci_register_host_bridge()      ─► root pci_bus exists
  │           └─ pci_scan_child_bus(bus)               ─► STAGE 2
  ├─ pci_assign_unassigned_root_bus_resources(bus)     ─► STAGE 3 (if needed)
  └─ pci_bus_add_devices(root->bus)                    ─► STAGE 4/5
```

Config access on x86 uses `pci_root_ops`, which goes to MMCONFIG (ECAM from the
ACPI `MCFG` table) or legacy I/O ports `0xCF8/0xCFC`.

### 3.2 ARM / embedded (Device Tree)

The host bridge is a platform device described in DT, for example:

```dts
pcie@40000000 {
	compatible = "pci-host-ecam-generic";
	device_type = "pci";
	reg = <0x0 0x40000000 0x0 0x10000000>;   /* ECAM window */
	bus-range = <0x00 0xff>;
	ranges = < ... MMIO / IO windows ... >;
};
```

The platform driver whose `of_match_table` contains that `compatible` is probed,
exactly like any other platform driver:

```
gen_pci_driver.probe = pci_host_common_probe()        pci-host-generic.c
  ├─ ops    = of_device_get_match_data()               (which ECAM flavour)
  ├─ bridge = devm_pci_alloc_host_bridge()
  └─ pci_host_common_init(pdev, bridge, ops)           pci-host-common.c
        ├─ cfg = pci_ecam_create(...)                  map ECAM from "reg"
        ├─ bridge->ops = &ops->pci_ops                 config read/write
        └─ pci_host_probe(bridge)                      probe.c
              ├─ pci_scan_root_bus_bridge(bridge)
              │     ├─ pci_register_host_bridge()      ─► root pci_bus exists
              │     └─ pci_scan_child_bus(bus)         ─► STAGE 2
              ├─ pci_assign_unassigned_root_bus_resources(bus)   ─► STAGE 3
              └─ pci_bus_add_devices(bus)              ─► STAGE 4/5
```

Vendor SoC controllers (DesignWare, Tegra, Rockchip, ...) in `drivers/pci/controller/`
follow the same pattern. They first bring up the PCIe link (PHY, clocks, LTSSM), then
call `pci_host_probe()`.

### 3.3 The common point

Both paths converge on the same core steps:

1. Allocate and register a `pci_host_bridge`, which creates the root `pci_bus`.
2. `pci_scan_child_bus()` (enumeration).
3. Assign resources.
4. `pci_bus_add_devices()` (driver matching).

---

## 4. Stage 2: Enumeration (scanning the buses)

### 4.1 In plain words: a worked example

**The problem.** When the host bridge is ready, the kernel knows only that bus 00
exists. It does not know what cards are plugged in, how many buses there are, or
which bus the ConnectX is on.

**The trick.** Every possible device has a fixed address: bus / device / function
(section 2.2). Each address has a config space (section 2.3), and its first 4 bytes
are always the Vendor ID and Device ID. So the kernel reads those 4 bytes at every
possible address, like knocking on every door in a street:

- **No device at that address:** the read fails, and the CPU gets `0xffffffff`
  (all ones). Nobody home, move on.
- **A device is there:** it answers with its IDs, for example `0x101d15b3`
  (vendor `15b3` = Mellanox, device `101d` = ConnectX-6 Dx). The kernel creates a
  `struct pci_dev` for it.

**Why bridges matter.** Some of the devices found are *bridges* (on PCIe, every
root port and switch port is one). A bridge is a door to another street: there is a
whole new bus behind it. The kernel gives that bus a number, creates a
`struct pci_bus` for it, and knocks on every door there too. That is the recursion.

**Two shortcuts** keep the scan short:

- Functions 1..7 are only tried if function 0 exists **and** its header says the
  device is multi-function.
- Behind a PCIe port there is a single link, so only device 0 can exist on that bus.
  The kernel skips devices 1..31 there (`only_one_child()`).

**Trace for the example machine** from section 2.1 (root port IDs are illustrative):

```
 Step  Address    First 4 bytes   What it means                     What the kernel does
 ────  ─────────  ──────────────  ────────────────────────────────  ──────────────────────────────────
 Scan bus 00 (the root bus)
  1    00:00.0    0xffffffff      empty                             next device
  2    00:01.0    0x????8086      Root Port, header type 1 = bridge create pci_dev, note "bridge"
  3    00:02.0    0x????8086      Root Port, bridge                 create pci_dev, note "bridge"
  4    00:03.0    0xffffffff      empty                             next device
  ...  00:1f.0    0xffffffff      empty                             bus 00 done

 Go behind bridge 00:01.0
  5    -          -               bridge needs a bus number         use bus 3b, create pci_bus 3b
                                                                    (firmware usually set it in the
                                                                     bridge's secondary bus register;
                                                                     if not, the kernel assigns one)
 Scan bus 3b
  6    3b:00.0    0x101d15b3      ConnectX-6 Dx, function 0;        create pci_dev 3b:00.0
                                  header says multi-function        → also try functions 1..7
  7    3b:00.1    0x101d15b3      ConnectX-6 Dx, function 1         create pci_dev 3b:00.1
  8    3b:00.2-7  0xffffffff      empty                             -
  9    3b:01.0+   (not read)      PCIe link: only device 0          skip; bus 3b done, no bridges here

 Go behind bridge 00:02.0
 10    -          -               bridge needs a bus number         assign bus 5e, create pci_bus 5e
 11    5e:00.0    0x....144d      NVMe SSD (example)                create pci_dev; bus 5e done

 No bridges left → enumeration finished.
```

**The result** is a tree of kernel objects that mirrors the hardware:

```
 pci_bus 00
   ├─ pci_dev 00:01.0 (root port) ──► pci_bus 3b
   │                                    ├─ pci_dev 3b:00.0  ConnectX PF0
   │                                    └─ pci_dev 3b:00.1  ConnectX PF1
   └─ pci_dev 00:02.0 (root port) ──► pci_bus 5e
                                        └─ pci_dev 5e:00.0  NVMe
```

No driver has run yet. These `pci_dev` objects only say *what* is there. Matching
drivers to them happens in stage 4.

**How a "read" actually happens.** With ECAM (section 2.4), reading the IDs of
`3b:00.0` is just a 32-bit memory read at:

```
 ECAM_base + (0x3b << 20) + (0 << 15) + (0 << 12) + 0x00  =  ECAM_base + 0x3b00000
```

The rest of this section shows the same process in kernel code.

### 4.2 The scan loop

```
pci_scan_child_bus(bus)                                    probe.c
  └─ pci_scan_child_bus_extend(bus, 0)
        │
        ├─ for devnr in 0..31:                             "Go find them, Rover!"
        │     pci_scan_slot(bus, PCI_DEVFN(devnr, 0))
        │       └─ for each function fn (0, then 1..7 if multi-function):
        │            pci_scan_single_device(bus, devfn)
        │              ├─ already known? return it
        │              ├─ dev = pci_scan_device(bus, devfn)
        │              │     ├─ pci_bus_read_dev_vendor_id()   config read @0x00
        │              │     │     0xffffffff / 0 => nothing there, return NULL
        │              │     ├─ dev = pci_alloc_dev(bus)       struct pci_dev
        │              │     ├─ dev->vendor, dev->device   from that dword
        │              │     └─ pci_setup_device(dev)          parse the header
        │              └─ pci_device_add(dev, bus)
        │
        ├─ reserve bus numbers for SR-IOV (pci_iov_bus_range)
        ├─ pcibios_fixup_bus(bus)
        │
        └─ for each bridge found on this bus:
              pci_scan_bridge_extend(bus, dev, max, ...)
                ├─ read/assign secondary + subordinate bus numbers
                ├─ child = pci_add_new_bus(bus, dev, busnr)    struct pci_bus
                └─ pci_scan_child_bus_extend(child, ...)       ◄── RECURSION
```

It is a **depth-first walk**: scan a bus, then descend through each bridge and scan
the bus behind it.

### 4.3 Detecting a device: the Vendor ID read

`pci_bus_generic_read_dev_vendor_id()` reads the first dword of config space:

```c
if (pci_bus_read_config_dword(bus, devfn, PCI_VENDOR_ID, l))
	return false;

/* Some broken boards return 0 or ~0 (PCI_ERROR_RESPONSE) if a slot is empty: */
if (PCI_POSSIBLE_ERROR(*l) || *l == 0x00000000 ||
    *l == 0x0000ffff || *l == 0xffff0000)
	return false;
```

- No device: the read completes as an error (Unsupported Request), which the CPU
  sees as all-ones (`0xffffffff`).
- Device present: low 16 bits = vendor, high 16 bits = device.
- Device not ready yet (Request Retry Status, `0x0001` vendor): the core waits up to
  60 s (`pci_bus_wait_rrs`).

For a ConnectX-6 Dx at `3b:00.0` the read returns `0x101d15b3`:

```
 0x101d15b3
   ├──┘└──┘
   │     └─ vendor = 0x15b3  (Mellanox / NVIDIA networking)
   └─────── device = 0x101d  (ConnectX-6 Dx)
```

### 4.4 Filling in the `pci_dev`: `pci_setup_device()`

| Reads | Stores in `pci_dev` |
|---|---|
| Header type (0x0E) | `hdr_type` (0 = endpoint, 1 = bridge), `multifunction` |
| Class / revision (0x08) | `class` (0x020000 for Ethernet), `revision` |
| Subsystem IDs | `subsystem_vendor`, `subsystem_device` |
| BARs (type 0 only) | `resource[]` via `pci_read_bases()` / `__pci_read_base()` |
| Bridge windows (type 1) | bridge resources |

**BAR sizing** (inside `__pci_read_base()`): save the BAR, write all-ones, read
back, restore. The bits that stay zero give the size:

```
 write 0xffffffff  ->  read 0xfe00000c   (low dword of 64-bit BAR0)
                         └─ bits [3:0] = 1100b: memory, 64-bit, prefetchable
                         └─ ~(0xfe000000) + 1 = 0x02000000 = 32 MB
```

(Values illustrative; the real size depends on the ConnectX model and firmware.)

### 4.5 Registering the `pci_dev`: `pci_device_add()`

```
pci_device_add(dev, bus)                                   probe.c
  ├─ pci_configure_device()          MPS, extended tags, LTR, ...
  ├─ device_initialize(&dev->dev)    embedded struct device
  ├─ set NUMA node, default DMA masks (32-bit)
  ├─ pci_fixup_device(pci_fixup_header, dev)   quirks for broken hardware
  ├─ pci_init_capabilities(dev)      walk capability list:
  │     MSI/MSI-X (disabled), PM, PCIe, AER, SR-IOV (pci_iov_init), ATS, ...
  ├─ list_add_tail(&dev->bus_list, &bus->devices)
  ├─ pci_set_msi_domain(dev)
  └─ device_add(&dev->dev)           now in sysfs: /sys/bus/pci/devices/0000:3b:00.0
                                     (driver binding is NOT allowed yet, see stage 4)
```

At this point the ConnectX exists as a `struct pci_dev`, but **no driver is bound**
and its BARs may not have addresses yet.

---

## 5. Stage 3: Resource assignment

Enumeration only learns BAR **sizes**. Before a driver can use a BAR, it needs an
**address** inside the host bridge's MMIO window, and every bridge on the path must
forward that range.

```
pci_assign_unassigned_root_bus_resources(bus)               setup-bus.c
  ├─ size bridge windows bottom-up
  │     (sum of the BARs behind each bridge, aligned)
  ├─ assign addresses top-down from the host bridge windows
  ├─ write BAR registers in each device
  └─ program bridge memory / prefetch / IO windows
```

On x86 the BIOS usually assigned everything already, and Linux mostly **claims**
those assignments. On many ARM boards Linux assigns them itself
(`PCI_REASSIGN_ALL_BUS` / `preserve_config`).

Example result for the ConnectX:

```
 Root port 00:01.0  prefetchable window  [mem 0x38000000000-0x38003ffffff]
                                                   │
 ConnectX 3b:00.0   BAR0 (64-bit, pref, 32M) ──────┤ 0x38002000000
 ConnectX 3b:00.1   BAR0 (64-bit, pref, 32M) ──────┘ 0x38000000000
```

---

## 6. Stage 4: Adding devices and matching drivers

### 6.1 `pci_bus_add_devices()`

```
pci_bus_add_devices(bus)                                    bus.c
  ├─ for each dev on bus: pci_bus_add_device(dev)
  │     ├─ pci_fixup_device(pci_fixup_final, dev)
  │     ├─ pci_save_state(dev)          saved config, for error recovery
  │     ├─ pm_runtime_enable()
  │     ├─ pci_dev_allow_binding(dev)   drivers may now bind
  │     ├─ device_initial_probe(&dev->dev)   ◄── try to find a driver NOW
  │     └─ pci_dev_assign_added(dev)
  └─ recurse into child buses
```

### 6.2 The driver core and `pci_bus_type`

Every `pci_dev` and every `pci_driver` is on `pci_bus_type` (`pci-driver.c`):

```c
const struct bus_type pci_bus_type = {
	.name  = "pci",
	.match = pci_bus_match,      /* does this driver handle this device? */
	.probe = pci_device_probe,   /* bind them */
	...
};
```

The driver core tries a match in **both** directions. It doesn't matter which side
appears first:

```
  Device appears first                        Driver appears first
  (boot enumeration, hotplug,                 (modprobe mlx5_core after boot)
   SR-IOV VF creation)
  ───────────────────────────                 ──────────────────────────────
  device_initial_probe(dev)                   pci_register_driver(drv)
    └─ for each driver on pci bus:              └─ __pci_register_driver()
                                                     drv->driver.bus = &pci_bus_type
                                                     driver_register()
                                                      └─ driver_attach(drv)
                                                          └─ for each device on pci bus:
           └───────────────────┬────────────────────────────┘
                               ▼
                 pci_bus_match(dev, drv)  → 1 ?
                               ▼
                 driver_probe_device() → really_probe()
                               ▼
                 pci_bus_type.probe = pci_device_probe(dev)
```

With modules there is a third path: when `device_add()` runs, the kernel emits a
**uevent** carrying `MODALIAS=pci:v000015B3d0000101Dsv...`. udev runs
`modprobe` with that alias. The alias table comes from `MODULE_DEVICE_TABLE(pci, ...)`
in the driver, so the right module is loaded automatically, and then the "driver
appears first" path runs.

### 6.3 Matching: `pci_bus_match()`

```
pci_bus_match(dev, drv)                                     pci-driver.c
  ├─ binding disallowed (not added yet)? → 0
  └─ pci_match_device(drv, dev)
        ├─ driver_override set and != this driver? → no match
        ├─ check dynamic IDs (added via /sys/bus/pci/drivers/<drv>/new_id)
        └─ check drv->id_table (static)
              each entry: vendor, device, subvendor, subdevice, class/class_mask
              PCI_ANY_ID acts as a wildcard
```

### 6.4 The ConnectX side: `mlx5_core_pci_table`

From `mlx5/core/main.c` (excerpt):

```c
static const struct pci_device_id mlx5_core_pci_table[] = {
	{ PCI_VDEVICE(MELLANOX, PCI_DEVICE_ID_MELLANOX_CONNECTX4) },
	{ PCI_VDEVICE(MELLANOX, 0x1014),
	  .driver_data = MLX5_PCI_DEV_IS_VF },			/* ConnectX-4 VF */
	...
	{ PCI_VDEVICE(MELLANOX, 0x101b) },			/* ConnectX-6 */
	{ PCI_VDEVICE(MELLANOX, 0x101c),
	  .driver_data = MLX5_PCI_DEV_IS_VF },			/* ConnectX-6 VF */
	{ PCI_VDEVICE(MELLANOX, 0x101d) },			/* ConnectX-6 Dx */
	{ PCI_VDEVICE(MELLANOX, 0x101e),
	  .driver_data = MLX5_PCI_DEV_IS_VF },			/* ConnectX Family mlx5Gen Virtual Function */
	{ PCI_VDEVICE(MELLANOX, 0x101f) },			/* ConnectX-6 LX */
	{ PCI_VDEVICE(MELLANOX, 0x1021) },			/* ConnectX-7 */
	{ PCI_VDEVICE(MELLANOX, 0x1023) },			/* ConnectX-8 */
	...
	{ 0, }
};

MODULE_DEVICE_TABLE(pci, mlx5_core_pci_table);

static struct pci_driver mlx5_core_driver = {
	.name            = KBUILD_MODNAME,          /* "mlx5_core" */
	.id_table        = mlx5_core_pci_table,
	.probe           = probe_one,
	.remove          = remove_one,
	.suspend         = mlx5_suspend,
	.resume          = mlx5_resume,
	.shutdown        = shutdown,
	.err_handler     = &mlx5_err_handler,       /* AER recovery */
	.sriov_configure = mlx5_core_sriov_configure,
	...
};

static int __init mlx5_init(void)
{
	...
	err = pci_register_driver(&mlx5_core_driver);
	...
}
module_init(mlx5_init);
```

`PCI_VDEVICE(MELLANOX, 0x101d)` expands to vendor `0x15b3`, device `0x101d`,
subvendor/subdevice `PCI_ANY_ID`. The `.driver_data` field is how one table marks
VFs. It reaches `probe_one()` as `id->driver_data`, and that is exactly what
`dev->coredev_type` is derived from.

Matching our example device:

```
 pci_dev 0000:3b:00.0                     mlx5_core_pci_table
 ┌───────────────────────┐                ┌──────────────────────────────────┐
 │ vendor           15b3 │ ──── match ──► │ ...                              │
 │ device           101d │                │ { 15b3, 101d, ANY, ANY }  CX-6 Dx│ ◄─ id
 │ subsystem_vendor 15b3 │   (ANY)        │ { 15b3, 101e, ANY, ANY, IS_VF }  │
 │ subsystem_device 0016 │   (ANY)        │ ...                              │
 └───────────────────────┘                └──────────────────────────────────┘
```

---

## 7. Stage 5: The driver probe (`mlx5_core`)

### 7.1 From the PCI core to `probe_one()`

```
pci_device_probe(&pdev->dev)                                pci-driver.c
  ├─ pci_device_can_probe()      VFs: honour sriov_drivers_autoprobe
  ├─ pci_assign_irq(), pcibios_alloc_irq()     legacy INTx
  ├─ pci_dev_get(pdev)
  └─ __pci_device_probe(drv, pdev)
        ├─ id = pci_match_device(drv, pdev)    the matched table entry
        └─ pci_call_probe(drv, pdev, id)
              ├─ run on a CPU of the device's NUMA node (if possible)
              └─ local_pci_probe()
                    ├─ pm_runtime_get_sync()  device in D0
                    ├─ pdev->driver = drv
                    └─ rc = drv->probe(pdev, id)
                                │
                                ▼
                     probe_one(pdev, id)                    mlx5/core/main.c
```

### 7.2 What `mlx5_core` does with the `pci_dev`

At the top of `probe_one()`, the driver allocates its own state
(`devlink` + `mlx5_core_dev`; see [mlx5 core objects deep dive](module4-mlx5-core-objects-deep-dive.md)).
Then `mlx5_pci_init()` does the **PCI-level** bring-up:

```
mlx5_pci_init(dev, pdev, id)                                main.c
  ├─ pci_set_drvdata(pdev, dev)                 pdev → mlx5_core_dev
  ├─ dev->bar_addr = pci_resource_start(pdev, 0)   address assigned in STAGE 3
  ├─ mlx5_pci_enable_device()  → pci_enable_device()
  │                                (turn on memory decoding in Command register)
  ├─ request_bar(pdev)          → pci_request_regions()   claim BARs for this driver
  ├─ pci_set_master(pdev)                       allow the device to do DMA
  ├─ set_dma_caps(pdev)                         64-bit DMA mask
  ├─ pci_enable_atomic_ops_to_root(...)         PCIe AtomicOps, if the path supports it
  └─ dev->iseg = ioremap(dev->bar_addr, sizeof(*dev->iseg))
                                                map BAR0 → the firmware "init segment"
```

From here on, the driver talks to the ConnectX firmware through BAR0 (command
queue, doorbells) and DMA. The rest of the mlx5 story (firmware handshake, EQs,
auxiliary devices for Ethernet / RDMA) continues in `mlx5_init_one()`.

### 7.3 The connection between the two guides

```
 THIS GUIDE                                       MLX5 CORE OBJECTS DEEP DIVE
 ─────────────────────────────────────────        ──────────────────────────────────
 host bridge probe                                 
 → enumeration creates pci_dev (3b:00.0)
 → resources assigned (BAR0 address)
 → pci_bus_match: 15b3:101d ∈ mlx5_core_pci_table
 → pci_device_probe → local_pci_probe
 → probe_one(pdev, id)                    ═══►     devlink + mlx5_core_dev allocated
                                                   dev->pdev = pdev
                                                   dev->coredev_type from id->driver_data
                                                   mlx5_pci_init, mlx5_init_one
                                                   aux devices: eth, rdma, ...
```

---

## 8. Bonus: SR-IOV VFs are enumerated at runtime

VFs are **not** found by the boot-time scan. A PF driver creates them on request,
and they then go through the same stages 2, 4 and 5 as any other device.

```
 echo 4 > /sys/bus/pci/devices/0000:3b:00.0/sriov_numvfs
        │
        ▼
 sriov_numvfs_store()                                     iov.c
   └─ pdev->driver->sriov_configure(pdev, 4)
        = mlx5_core_sriov_configure()                     mlx5/core/sriov.c
            └─ mlx5_sriov_enable()
                  └─ pci_enable_sriov(pdev, 4)            iov.c
                        └─ sriov_enable()
                              ├─ set VF Enable in the SR-IOV capability
                              └─ sriov_add_vfs() → for each VF:
                                    pci_iov_add_virtfn(pf, id)
                                      ├─ virtfn = pci_alloc_dev(); is_virtfn = 1
                                      ├─ devfn = pci_iov_virtfn_devfn(pf, id)
                                      │   (from First VF Offset / VF Stride)
                                      ├─ vendor 15b3, device = VF Device ID (101e)
                                      ├─ BARs from the PF's SR-IOV VF BARs
                                      ├─ pci_device_add(virtfn, bus)   ◄── stage 2 end
                                      └─ pci_bus_add_device(virtfn)    ◄── stage 4
                                            └─ match 15b3:101e
                                                 → .driver_data = MLX5_PCI_DEV_IS_VF
                                                 → probe_one(vf_pdev, id)   stage 5
                                                 → coredev_type = MLX5_COREDEV_VF
```

The same `mlx5_core` driver probes the VFs. `MLX5_PCI_DEV_IS_VF` in the matched
table entry is how it knows.

---

## 9. Full call chain

```
 FIRMWARE describes Root Complex (ACPI PNP0A08  /  DT "pci-host-ecam-generic")
   │
   ▼  STAGE 1: host bridge probe
 acpi_pci_root_add()                    │  pci_host_common_probe()
   pci_acpi_scan_root()                 │    pci_host_common_init()
     acpi_pci_root_create()             │      pci_ecam_create()
       pci_create_root_bus()            │      pci_host_probe()
         pci_register_host_bridge()     │        pci_scan_root_bus_bridge()
                                        │          pci_register_host_bridge()
   │                                    │
   ▼  STAGE 2: enumeration (both paths)
 pci_scan_child_bus(root bus)
   pci_scan_child_bus_extend()
     pci_scan_slot()  × 32 devices
       pci_scan_single_device()  × up to 8 functions
         pci_scan_device()
           pci_bus_read_dev_vendor_id()    ── config read, 0x101d15b3
           pci_alloc_dev()                 ── struct pci_dev
           pci_setup_device()              ── header, class, BAR sizes
         pci_device_add()                  ── capabilities, device_add()
     pci_scan_bridge_extend()  for each bridge
       pci_add_new_bus()                   ── child struct pci_bus
       pci_scan_child_bus_extend(child)    ── recursion
   │
   ▼  STAGE 3: resources
 pci_assign_unassigned_root_bus_resources()
   │
   ▼  STAGE 4: add + match
 pci_bus_add_devices()
   pci_bus_add_device()
     pci_dev_allow_binding()
     device_initial_probe()
       ... driver core ...
         pci_bus_match()  ── 15b3:101d in mlx5_core_pci_table
   │
   ▼  STAGE 5: driver probe
 pci_device_probe()
   __pci_device_probe()
     pci_call_probe()
       local_pci_probe()
         mlx5_core_driver.probe = probe_one(pdev, id)
           mlx5_devlink_alloc(), dev->pdev = pdev, ...
           mlx5_pci_init()  ── enable device, request BARs, bus master, DMA, ioremap BAR0
           mlx5_init_one()  ── firmware, EQs, aux devices (eth, rdma)
```

---

## 10. Seeing it on a real system

> These commands need a machine with the ConnectX physically attached (bare metal,
> or a VM with the device passed through). WSL2 doesn't show the host's real PCIe
> tree: devices passed into WSL arrive through Hyper-V virtual PCI (`pci-hyperv`),
> with a synthetic domain number.

| What | Command |
|---|---|
| Tree view of buses / bridges | `lspci -tv` |
| Find ConnectX functions | `lspci -nn -d 15b3:` |
| Full decoded config space | `sudo lspci -vvv -s 3b:00.0` |
| Raw config space hex | `sudo lspci -xxxx -s 3b:00.0` |
| IDs from sysfs | `cat /sys/bus/pci/devices/0000:3b:00.0/{vendor,device,class}` |
| BAR addresses | `cat /sys/bus/pci/devices/0000:3b:00.0/resource` |
| Which driver is bound | `readlink /sys/bus/pci/devices/0000:3b:00.0/driver` |
| Module alias | `cat /sys/bus/pci/devices/0000:3b:00.0/modalias` |
| Boot-time enumeration log | `dmesg \| grep -E "pci 0000:3b|PCI host bridge|mlx5_core"` |
| Rescan the whole tree | `echo 1 \| sudo tee /sys/bus/pci/rescan` |
| Remove one device | `echo 1 \| sudo tee /sys/bus/pci/devices/0000:3b:00.0/remove` |
| Unbind / bind driver | `echo 0000:3b:00.0 \| sudo tee /sys/bus/pci/drivers/mlx5_core/{unbind,bind}` |

Example output (illustrative):

```
$ lspci -nn -d 15b3:
3b:00.0 Ethernet controller [0200]: Mellanox Technologies MT2892 Family [ConnectX-6 Dx] [15b3:101d]
3b:00.1 Ethernet controller [0200]: Mellanox Technologies MT2892 Family [ConnectX-6 Dx] [15b3:101d]

$ dmesg | grep 3b:00.0
pci 0000:3b:00.0: [15b3:101d] type 00 class 0x020000 PCIe Endpoint     ← stage 2
pci 0000:3b:00.0: BAR 0 [mem 0x38002000000-0x38003ffffff 64bit pref]  ← stage 2/3
pci 0000:3b:00.0: PME# supported from D3cold
mlx5_core 0000:3b:00.0: firmware version: 22.xx.xxxx                   ← stage 5
```

`modalias` shows how module autoload matches:

```
pci:v000015B3d0000101Dsv000015B3sd00000016bc02sc00i00
     └vendor   └device   └subvendor └subdevice └class/subclass/prog-if
```

---

## 11. Cheat sheet

**Who does what**

| Stage | Who | Key function | Output |
|---|---|---|---|
| 1. Host bridge probe | ACPI `pci_root` / platform host driver | `acpi_pci_root_add()`, `pci_host_common_probe()` | `pci_host_bridge`, root `pci_bus` |
| 2. Enumeration | PCI core | `pci_scan_child_bus()` | `pci_dev` per function, `pci_bus` per bridge |
| 3. Resources | PCI core | `pci_assign_unassigned_root_bus_resources()` | BAR addresses, bridge windows |
| 4. Add + match | PCI core + driver core | `pci_bus_add_devices()`, `pci_bus_match()` | (device, driver) pair |
| 5. Driver probe | PCI core → driver | `pci_device_probe()` → `drv->probe()` | `probe_one()` runs |

**ConnectX quick facts**

| Item | Value |
|---|---|
| Vendor ID | `0x15b3` (Mellanox) |
| Example device IDs | CX-6 Dx PF `0x101d`, CX-6 PF `0x101b`, CX-7 `0x1021`, CX-8 `0x1023`, VF `0x101e` |
| Class | `0x020000` (Ethernet) or `0x020700` (InfiniBand), depending on port type |
| PCI driver | `mlx5_core`; `.probe = probe_one` |
| Main BAR | BAR0, 64-bit prefetchable: firmware init segment + doorbells |
| VF marker | `.driver_data = MLX5_PCI_DEV_IS_VF` in `mlx5_core_pci_table` |

**Five things to remember**

1. Firmware (ACPI or DT) tells Linux where the Root Complex is. A host bridge driver
   turns that into a `pci_host_bridge` and the root bus.
2. Enumeration is a depth-first scan: read Vendor ID at every bus/device/function;
   all-ones means empty; bridges lead to child buses.
3. Enumeration learns BAR **sizes**. Resource assignment gives them **addresses**.
4. Drivers bind by matching vendor/device IDs against their `id_table`, whichever
   of device or driver appears first.
5. For a ConnectX, the matched entry of `mlx5_core_pci_table` is passed as `id` to
   `probe_one()`, and that is where [mlx5 core objects deep dive](module4-mlx5-core-objects-deep-dive.md)
   begins.
