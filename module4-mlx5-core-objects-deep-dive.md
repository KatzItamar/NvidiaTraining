# mlx5 Core Objects: `pdev`, `devlink`, `mlx5_core_dev` and Auxiliary Devices

A training guide to how the mlx5 driver (NVIDIA/Mellanox ConnectX) represents one
PCI function in the kernel, and how the Ethernet and RDMA drivers attach to it.

Read [PCIe deep dive](module4-pcie-probe-deep-dive.md) first: it explains how Linux
finds the ConnectX on the PCIe bus and calls `probe_one()`. This guide starts there.

**Contents**

1. [The big picture](#1-the-big-picture)
2. [Meet the four objects](#2-meet-the-four-objects)
3. [Walkthrough: the start of `probe_one()`](#3-walkthrough-the-start-of-probe_one)
4. [How the objects are linked](#4-how-the-objects-are-linked)
5. [Deep dive: memory layout and fields](#5-deep-dive-memory-layout-and-fields)
6. [Auxiliary devices: how Ethernet and RDMA attach](#6-auxiliary-devices-how-ethernet-and-rdma-attach)
7. [Full lifecycle](#7-full-lifecycle)
8. [Cheat sheet](#8-cheat-sheet)

Source files referenced (paths relative to the kernel tree):

| File | What is there |
|---|---|
| `drivers/net/ethernet/mellanox/mlx5/core/main.c` | `probe_one()`, `remove_one()`, `mlx5_pci_init()`, `mlx5_init_one()` |
| `drivers/net/ethernet/mellanox/mlx5/core/devlink.c` | `mlx5_devlink_alloc()`, `mlx5_devlink_ops` |
| `drivers/net/ethernet/mellanox/mlx5/core/dev.c` | Auxiliary device table, `add_adev()`, `mlx5_register_device()` |
| `drivers/net/ethernet/mellanox/mlx5/core/en_main.c` | `mlx5e_probe()` (Ethernet aux driver) |
| `drivers/infiniband/hw/mlx5/main.c` | `mlx5r_probe()` (RDMA aux driver) |
| `include/linux/pci.h` | `struct pci_dev` |
| `net/devlink/devl_internal.h` | `struct devlink` |
| `include/linux/mlx5/driver.h` | `struct mlx5_core_dev`, `struct mlx5_priv`, `struct mlx5_adev` |

> Field lists in this guide are abridged to what matters for the flow. `...` marks
> omitted members. Always check the headers in your tree for the full definition.

---

## 1. The big picture

One ConnectX PCI function is represented by several kernel objects, each owned by a
different layer:

```
   Layer               Object                        Owned by
 ─────────────────────────────────────────────────────────────────────────────
   Hardware identity   struct pci_dev  (pdev)        PCI core
                         └─ struct device dev        (embedded in pdev)
   Management          struct devlink                devlink core (allocated by mlx5)
   Driver state        struct mlx5_core_dev (mdev)   mlx5_core  (lives inside devlink)
   Functions           auxiliary devices             mlx5_core creates them;
                         .eth  .rdma  .vnet ...      mlx5e / mlx5_ib / mlx5_vdpa bind
   User-facing         net_device, ib_device, ...    netdev / RDMA subsystems
```

From the bottom up:

- The PCI core discovers the function and creates `pdev`.
- `mlx5_core` binds to `pdev` as the **only** PCI driver.
- `mlx5_core` allocates `devlink` + `mlx5_core_dev` together and brings up the hardware.
- `mlx5_core` then publishes **auxiliary devices**, one per function (Ethernet, RDMA, ...).
- Separate drivers bind to those auxiliary devices and create the user-facing
  `net_device` / `ib_device`, all sharing the same `mlx5_core_dev`.

---

## 2. Meet the four objects

| Name | What it is | Layer | Owned by |
|---|---|---|---|
| `struct device` | Generic Linux device-model object: kobject, driver binding, DMA settings, `driver_data` | Driver core | Embedded inside bus-specific structs (here: inside `pci_dev`) |
| `pdev` / `struct pci_dev` | A PCI function: vendor/device IDs, BARs, config space, MSI-X. **Embeds** a `struct device` | PCI bus | PCI core |
| `struct devlink` | Device-wide management handle. What the `devlink` CLI talks to over netlink | Networking | Allocated by the driver via `devlink_alloc()` |
| `struct mlx5_core_dev` | mlx5's per-function state: command queue, capabilities, EQs, eswitch, SR-IOV, ... | mlx5 driver | The driver, as devlink's private area |

### What each one is for

- **`pdev` / `struct device`: hardware identity.** Created by the PCI core during bus
  enumeration. Exists whether or not a driver is bound. Used for PCI config access,
  `pci_enable_device()`, mapping BARs, DMA mapping, IRQ allocation.
- **`devlink`: management identity.** Exposes the device to userspace:
  `devlink dev eswitch set`, `devlink dev reload`, `devlink health`, `devlink dev param`.
  It is independent of any netdev, so one devlink instance can sit behind several
  net devices and RDMA devices.
- **`mlx5_core_dev`: the driver's brain for one PCI function.** Every mlx5
  sub-component takes it as its handle: `mlx5_cmd_exec(dev, ...)`,
  `mlx5_eq_create(dev, ...)`, and so on. Often called `mdev` outside `main.c`.

### Watch out: `dev` is ambiguous

| Where | `dev` means |
|---|---|
| mlx5 core code (`main.c`, `dev.c`, ...) | `struct mlx5_core_dev *` |
| mlx5e / mlx5_ib | usually `mdev` for `mlx5_core_dev`; `dev` may be a private struct |
| Generic kernel code | `struct device *` (or `net_device`) |
| Inside mlx5, the generic device | `dev->device` or `&pdev->dev` |

Always check the type before assuming.

---

## 3. Walkthrough: the start of `probe_one()`

`probe_one()` in `main.c` is the PCI `.probe` callback of `mlx5_core`. Its first lines:

```c
static int probe_one(struct pci_dev *pdev, const struct pci_device_id *id)
{
	struct mlx5_core_dev *dev;
	struct devlink *devlink;
	int err;

	devlink = mlx5_devlink_alloc(&pdev->dev);
	if (!devlink) {
		dev_err(&pdev->dev, "devlink alloc failed\n");
		return -ENOMEM;
	}

	dev = devlink_priv(devlink);
	dev->device = &pdev->dev;
	dev->pdev = pdev;

	dev->coredev_type = id->driver_data & MLX5_PCI_DEV_IS_VF ?
			 MLX5_COREDEV_VF : MLX5_COREDEV_PF;
	...
```

`mlx5_devlink_alloc()` (in `devlink.c`) is a thin wrapper:

```c
struct devlink *mlx5_devlink_alloc(struct device *dev)
{
	return devlink_alloc(&mlx5_devlink_ops, sizeof(struct mlx5_core_dev), dev);
}
```

### Line by line

| Code | Meaning |
|---|---|
| `struct mlx5_core_dev *dev; struct devlink *devlink; int err;` | Declarations. `err` is used later in the function for the return codes of init steps. |
| `mlx5_devlink_alloc(&pdev->dev)` | Allocates `struct devlink` **plus** `sizeof(struct mlx5_core_dev)` bytes of private area, in **one** allocation. Sets `devlink->ops = &mlx5_devlink_ops` and `devlink->dev = &pdev->dev`. |
| `if (!devlink) { ...; return -ENOMEM; }` | Allocation failure. Nothing set up yet, so nothing to unwind. |
| `dev = devlink_priv(devlink)` | Returns `devlink->priv`, the flexible array at the end of `struct devlink`. That memory *is* the `mlx5_core_dev`. |
| `dev->device = &pdev->dev` | Back-pointer to the generic device. Used for logging (`mlx5_core_err()`), the DMA API, and as parent of the auxiliary devices. |
| `dev->pdev = pdev` | Back-pointer to the PCI-typed view of the same device. |
| `dev->coredev_type = ...` | `id` is the matching entry of the driver's `pci_device_id` table. If its `driver_data` has `MLX5_PCI_DEV_IS_VF`, this is an SR-IOV Virtual Function; otherwise a Physical Function. Later code uses this to skip PF-only work. |

### Why allocate it this way?

- devlink exists from the very first line of probe, so later steps can register
  params, health reporters, etc. against it.
- One allocation means one free: `devlink_free()` releases both objects, on the
  error path and on remove.

---

## 4. How the objects are linked

### Overview drawing

```
 PCI core allocates                     mlx5 driver allocates (one kzalloc inside devlink_alloc)
 ─────────────────                      ─────────────────────────────────────────────────────

 struct pci_dev  (pdev)                 ┌──────────────────────────────────────────────┐
 ┌──────────────────────────┐           │ struct devlink            ◄── "devlink"      │
 │ vendor/device, BARs, irq │           │  ├ ops  = &mlx5_devlink_ops                  │
 │ ...                      │           │  ├ dev  ─────────────────────────┐           │
 │ struct device dev  ◄─────┼───────────┼──┼───────────────────────────────┘           │
 │  (generic device, kobj,  │  &pdev->dev  ├ params, ports, health reporters, ...      │
 │   driver_data, ...)      │           │  │                                           │
 └──────────────────────────┘           │  │   (devlink_priv() returns this address)   │
            ▲   ▲                       │  ▼                                           │
            │   │                       │ ┌──────────────────────────────────────────┐ │
            │   │                       │ │ struct mlx5_core_dev  ◄── "dev"          │ │
            │   │                       │ │  (tail / private area, same allocation)  │ │
            │   │  dev->pdev            │ │                                          │ │
            │   └───────────────────────┼─┤  struct pci_dev *pdev                    │ │
            │      dev->device          │ │  struct device  *device                  │ │
            └───────────────────────────┼─┤  enum coredev_type = PF | VF             │ │
                  (&pdev->dev)          │ │  ...                                     │ │
                                        │ └──────────────────────────────────────────┘ │
                                        └──────────────────────────────────────────────┘
                                         one contiguous block; freed by devlink_free()
```

### Compact view

```
pdev (PCI function)
  └─ pdev->dev (struct device)  ◄── devlink->dev
        └─ driver_data ──► mlx5_core_dev ◄── devlink_priv(devlink)
                              ├─ ->pdev   ──► pdev
                              └─ ->device ──► &pdev->dev
```

### Pointer map

| # | From | To | Set by |
|---|---|---|---|
| 1 | `devlink->dev` | `&pdev->dev` | `devlink_alloc()` (3rd arg) |
| 2 | `dev->device` | `&pdev->dev` | `probe_one()` |
| 3 | `dev->pdev` | `pdev` | `probe_one()` |
| 4 | `pdev->dev.driver_data` | `dev` | `pci_set_drvdata()` in `mlx5_pci_init()` |
| 5 | `devlink->ops` | `&mlx5_devlink_ops` | `devlink_alloc()` (1st arg) |
| 6 | `dev` | `devlink->priv` | `devlink_priv()`. Not a stored pointer, just an address computation |

### Navigating between them in code

```c
/* From pdev to everything else (remove, shutdown, PCI error handlers) */
struct mlx5_core_dev *dev = pci_get_drvdata(pdev);       /* pointer 4 */
struct devlink *devlink   = priv_to_devlink(dev);        /* inverse of devlink_priv() */

/* From devlink callbacks (devlink ops receive a struct devlink *) */
struct mlx5_core_dev *dev = devlink_priv(devlink);

/* From mlx5_core_dev to the PCI side */
struct pci_dev *pdev = dev->pdev;
struct device  *d    = dev->device;                      /* == &pdev->dev */
```

### Key points

- `struct pci_dev` is owned by the PCI core. The driver only borrows it.
- `struct devlink` and `struct mlx5_core_dev` are one allocation; they are born and
  die together.
- `pdev` and `&pdev->dev` are the same object seen two ways: PCI-typed and generic.

---

## 5. Deep dive: memory layout and fields

```
╔═══════════════════════════════════════════════════════════════════════════════════════════╗
║  struct pci_dev  (pdev)               allocated by PCI core during bus enumeration        ║
╠═══════════════════════════════════════════════════════════════════════════════════════════╣
║  bus_list, bus, subordinate, slot, devfn                                                  ║
║  vendor, device, subsystem_vendor, subsystem_device, class, revision, hdr_type            ║
║  pcie_cap, msi_cap, msix_cap, aer_cap, ...                                                ║
║  struct resource resource[DEVICE_COUNT_RESOURCE]   ◄── BAR0..5, ROM                       ║
║                      │                                                                    ║
║                      └─ resource[0] start ─► dev->bar_addr (pci_resource_start(pdev, 0))  ║
║                                                                                           ║
║  ┌─────────────────────────────────────────────────────────────────────────────────────┐  ║
║  │ struct device dev   (embedded, "pdev->dev")                                         │  ║
║  │   kobj, parent, bus = &pci_bus_type, driver = mlx5_core driver                      │  ║
║  │   driver_data ───────────────────────────────────────────────┐   ◄─ pci_set_drvdata │  ║
║  │   dma_mask / dma_ops, numa_node, power, ...                  │                      │  ║
║  └──────────────▲───────────────────────────────────────────────┼──────────────────────┘  ║
╚═════════════════╪═══════════════════════════════════════════════╪═════════════════════════╝
                  │                                               │
                  │ (1) devlink->dev                              │ (4) pdev->dev.driver_data
                  │ (2) dev->device                               │     = mlx5_core_dev *
                  │ (3) dev->pdev  ── points to pdev itself ──┐   │
                  │                                           │   │
╔═════════════════╧═══════════════════════════════════════════╧═══╧═════════════════════════╗
║  ONE ALLOCATION from devlink_alloc(&mlx5_devlink_ops, sizeof(struct mlx5_core_dev), dev)  ║
║                                                                                           ║
║  ┌─────────────────────────────────────────────────────────────────────────────────────┐  ║
║  │ struct devlink                                  ◄── "devlink"                       │  ║
║  │   u32 index                          (id in global `devlinks` xarray)               │  ║
║  │   const struct devlink_ops *ops  ───► mlx5_devlink_ops (reload, eswitch, flash...)  │  ║
║  │   struct device *dev             ───► &pdev->dev                                    │  ║
║  │   const struct device_driver *dev_driver                                            │  ║
║  │   struct xarray ports            (devlink_port objects: uplink, VF/SF reps)         │  ║
║  │   struct xarray params           (enable_roce, flow_steering_mode, ...)             │  ║
║  │   struct list_head reporter_list (health reporters: fw, fw_fatal, tx, rx...)        │  ║
║  │   struct list_head region_list, resource_list, trap_list, rate_list, sb_list, ...   │  ║
║  │   struct mutex lock              (instance lock: devl_lock/devl_unlock)             │  ║
║  │   refcount_t refcount, possible_net_t _net, struct rcu_work rwork                   │  ║
║  │   struct devlink_rel *rel, struct xarray nested_rels                                │  ║
║  │                                                                                     │  ║
║  │   char priv[] __aligned(NETDEV_ALIGN)   ◄── flexible array = devlink_priv()         │  ║
║  └──────────────────────────────────────┬──────────────────────────────────────────────┘  ║
║                                         │  priv starts here                               ║
║                                         ▼                                                 ║
║  ┌─────────────────────────────────────────────────────────────────────────────────────┐  ║
║  │ struct mlx5_core_dev                            ◄── "dev" / "mdev"                  │  ║
║  │                                                                                     │  ║
║  │  ── identity / PCI glue (set at the top of probe_one) ────────────────────────────  │  ║
║  │   struct device  *device          = &pdev->dev                                      │  ║
║  │   struct pci_dev *pdev            = pdev                                            │  ║
║  │   enum mlx5_coredev_type coredev_type   = MLX5_COREDEV_PF | MLX5_COREDEV_VF         │  ║
║  │                                                                                     │  ║
║  │  ── PCI / device state ───────────────────────────────────────────────────────────  │  ║
║  │   struct mutex pci_status_mutex;  enum mlx5_pci_status pci_status                   │  ║
║  │   u8 rev_id;  char board_id[]                                                       │  ║
║  │   phys_addr_t bar_addr            = BAR0 physical address                           │  ║
║  │   struct mlx5_init_seg __iomem *iseg   = ioremap of BAR0 (FW init segment)          │  ║
║  │   enum mlx5_device_state state;  unsigned long intf_state                           │  ║
║  │   struct mutex intf_state_mutex                                                     │  ║
║  │   u64 sys_image_guid                                                                │  ║
║  │                                                                                     │  ║
║  │  ── firmware interface ───────────────────────────────────────────────────────────  │  ║
║  │   struct mlx5_cmd cmd             (command queue, mailboxes, DMA pools)             │  ║
║  │   struct mlx5_timeouts *timeouts                                                    │  ║
║  │   struct { hca[MLX5_CAP_NUM]; pcam; mcam; fpga; qcam; embedded_cpu } caps          │  ║
║  │   u32 issi                                                                          │  ║
║  │                                                                                     │  ║
║  │  ── feature objects ──────────────────────────────────────────────────────────────  │  ║
║  │   struct mlx5_clock *clock;  struct mlx5_fw_tracer *tracer                          │  ║
║  │   struct mlx5_vxlan *vxlan;  struct mlx5_geneve *geneve                             │  ║
║  │   struct mlx5_rsc_dump *rsc_dump;  struct mlx5_hwmon *hwmon                         │  ║
║  │   struct mlx5_sd *sd;  struct mlx5_dm *dm;  struct mlx5_st *st                      │  ║
║  │   struct mlx5e_resources mlx5e_res;  struct { reserved_gids; roce_en } roce         │  ║
║  │   struct mlx5_profile profile;  enum mlx5_wc_state wc_state                         │  ║
║  │   struct devlink *shd                                                               │  ║
║  │   [MACsec / FPGA / hv_vhca / data_direct, depending on config]                      │  ║
║  │                                                                                     │  ║
║  │  ── struct mlx5_priv priv  (embedded) ────────────────────────────────────────────  │  ║
║  │  ┌───────────────────────────────────────────────────────────────────────────────┐  │  ║
║  │  │ irq_table, eq_table                       (MSI-X vectors and event queues)    │  │  ║
║  │  │ pg_nb, pg_wq, page_root_xa, free_list,    (firmware page allocation)          │  │  ║
║  │  │   fw_pages, page_counters[]                                                   │  │  ║
║  │  │ struct mlx5_core_health health            (health poll, recovery work)        │  │  ║
║  │  │ struct mlx5_debugfs_entries dbg                                               │  │  ║
║  │  │ alloc_mutex, numa_node, frag_buf_node_pools, db_node_pools                    │  │  ║
║  │  │ struct mlx5_adev **adev  ─────────────────► aux devices (see section 6)       │  │  ║
║  │  │ int adev_idx                              (the ".N" in mlx5_core.eth.N)       │  │  ║
║  │  │ struct mlx5_events *events, *vhca_events                                      │  │  ║
║  │  │ struct mlx5_flow_steering *steering, struct mlx5_mpfs *mpfs                   │  │  ║
║  │  │ struct mlx5_eswitch *eswitch              (switchdev / legacy mode)           │  │  ║
║  │  │ struct mlx5_core_sriov sriov              (VF array, num_vfs)                 │  │  ║
║  │  │ struct mlx5_lag *lag                      (bonding offload)                   │  │  ║
║  │  │ struct mlx5_devcom_dev *devc              (cross-device communication)        │  │  ║
║  │  │ struct mlx5_fw_reset *fw_reset            (reload/reset handling)             │  │  ║
║  │  │ struct mlx5_fc_stats *fc_stats, rl_table, ft_pool, bfregs, bfreg              │  │  ║
║  │  │ u32 flags                                                                     │  │  ║
║  │  │ [SF: vhca_state_nb, sf_dev_table, parent_mdev ... (CONFIG_MLX5_SF)]           │  │  ║
║  │  └───────────────────────────────────────────────────────────────────────────────┘  │  ║
║  └─────────────────────────────────────────────────────────────────────────────────────┘  ║
╚═══════════════════════════════════════════════════════════════════════════════════════════╝
         ▲                                    freed by mlx5_devlink_free() → devlink_free()
         │
   `dev = devlink_priv(devlink)`  ≡  `(void *)devlink->priv`
```

**`bar_addr` and `iseg`: the link to the hardware.** `bar_addr` is the physical
address the PCI core gave the ConnectX's BAR0. `iseg` is the same range mapped into
kernel virtual memory with `ioremap()`. Every register access to the firmware goes
through `iseg` with `ioread32be()`/`iowrite32be()`, which is MMIO. See
[PCIe deep dive, section 2.4](module4-pcie-probe-deep-dive.md#24-mmio-how-the-cpu-talks-to-a-device)
for what MMIO is and how BAR0 got its address.

---

## 6. Auxiliary devices: how Ethernet and RDMA attach

### 6.1 The problem

One ConnectX function must serve several kernel subsystems:

- networking (`net_device`)
- RDMA / InfiniBand (`ib_device`)
- vDPA, DPLL, fwctl, ...

These live in different modules (`mlx5_ib`, `mlx5_vdpa`, ...; the Ethernet part,
`mlx5e`, is built into `mlx5_core`) and should load independently: Ethernet must
work even if the RDMA module is not loaded.

But **only one driver can bind to a PCI device**. That driver is `mlx5_core`.

### 6.2 The solution: the auxiliary bus

The auxiliary bus (`include/linux/auxiliary_bus.h`) is a lightweight virtual bus:

- A **parent driver** (`mlx5_core`) creates child devices on it.
- **Other drivers** bind to those children **by name**, the way a PCI driver binds by
  vendor/device ID.
- A child has no hardware of its own. It represents one *function* of the parent.

### 6.3 The mlx5 wrapper: `struct mlx5_adev`

`include/linux/mlx5/driver.h`:

```c
struct mlx5_adev {
	struct auxiliary_device adev;   /* what the aux bus sees          */
	struct mlx5_core_dev   *mdev;   /* ◄── pointer back to the core    */
	int idx;                        /* protocol index in the table     */
};
```

`mlx5_core_dev->priv.adev[]` is an array of pointers to these wrappers, one slot per
protocol. It is allocated in `mlx5_adev_init()` (called from `mlx5_mdev_init()`).

### 6.4 The protocol table

`dev.c` defines which auxiliary devices exist and when each is supported:

| Protocol | Suffix | Aux device name | Bound by |
|---|---|---|---|
| `MLX5_INTERFACE_PROTOCOL_ETH` | `eth` | `mlx5_core.eth.N` | `mlx5e` (Ethernet) |
| `MLX5_INTERFACE_PROTOCOL_ETH_REP` | `eth-rep` | `mlx5_core.eth-rep.N` | `mlx5e` representors (switchdev) |
| `MLX5_INTERFACE_PROTOCOL_IB` | `rdma` | `mlx5_core.rdma.N` | `mlx5_ib` (RDMA) |
| `MLX5_INTERFACE_PROTOCOL_IB_REP` | `rdma-rep` | `mlx5_core.rdma-rep.N` | `mlx5_ib` representors |
| `MLX5_INTERFACE_PROTOCOL_MPIB` | `multiport` | `mlx5_core.multiport.N` | `mlx5_ib` multiport |
| `MLX5_INTERFACE_PROTOCOL_VNET` | `vnet` | `mlx5_core.vnet.N` | `mlx5_vdpa` |
| `MLX5_INTERFACE_PROTOCOL_DPLL` | `dpll` | `mlx5_core.dpll.N` | mlx5 DPLL driver |
| `MLX5_INTERFACE_PROTOCOL_FWCTL` | `fwctl` | `mlx5_core.fwctl.N` | mlx5 fwctl driver |

Each entry has an `is_supported()` callback (for example `mlx5_eth_supported`,
`mlx5_rdma_supported`), and some also have an `is_enabled()` callback. A device is
only created if the hardware supports it and it is enabled by configuration (for
example via devlink params).

`N` is `dev->priv.adev_idx`, a unique index allocated in `probe_one()` by
`mlx5_adev_idx_alloc()`. All aux devices of one PCI function share the same `N`.
The full name is `MLX5_ADEV_NAME` (`"mlx5_core"`) + `.` + suffix + `.` + `N`.
They are visible in `/sys/bus/auxiliary/devices/`.

### 6.5 Creating an aux device: `add_adev()`

From `dev.c` (simplified):

```c
static struct mlx5_adev *add_adev(struct mlx5_core_dev *dev, int idx)
{
	const char *suffix = mlx5_adev_devices[idx].suffix;
	struct mlx5_adev *madev = kzalloc(...);
	struct auxiliary_device *adev = &madev->adev;

	adev->id          = dev->priv.adev_idx;   /* the ".N"                      */
	adev->name        = suffix;               /* "eth", "rdma", ...            */
	adev->dev.parent  = dev->device;          /* == &pdev->dev                 */
	adev->dev.release = adev_release;
	madev->mdev       = dev;                  /* ◄── the pointer back          */
	madev->idx        = idx;

	auxiliary_device_init(adev);
	auxiliary_device_add(adev);               /* now visible; drivers can bind */
	return madev;
}
```

Note that the aux device's parent is `&pdev->dev`, so in sysfs it appears under the
PCI device.

### 6.6 Binding: how mlx5e and mlx5_ib find `mlx5_core_dev`

Each aux driver declares the names it matches:

```c
/* en_main.c (Ethernet) */
static const struct auxiliary_device_id mlx5e_id_table[] = {
	{ .name = MLX5_ADEV_NAME ".eth", },
	{},
};

/* drivers/infiniband/hw/mlx5/main.c (RDMA) */
static const struct auxiliary_device_id mlx5r_id_table[] = {
	{ .name = MLX5_ADEV_NAME ".rdma", },
	{},
};
```

When the aux bus matches a device to a driver, the driver's probe gets only a
`struct auxiliary_device *`. It recovers the core with `container_of`:

```c
/* mlx5e_probe() in en_main.c; mlx5r_probe() in mlx5_ib does the same */
struct mlx5_adev *edev = container_of(adev, struct mlx5_adev, adev);
struct mlx5_core_dev *mdev = edev->mdev;
```

From `mdev`, every function driver uses the **same** underlying device state:
firmware command interface, HCA capabilities, event queues and IRQs, flow steering,
eswitch. Each then builds its own subsystem object on top: `mlx5e` creates a
`net_device`, `mlx5_ib` registers an `ib_device`.

### 6.7 Drawing: the whole stack

```
                         ┌──────────────────────────────┐
                         │ struct pci_dev (pdev)        │
                         │   struct device dev ◄────────┼──────────────┐ parent of all
                         └──────────────┬───────────────┘              │ aux devices
                                        │ bound to (PCI driver)        │
                                        ▼                              │
                         ┌──────────────────────────────┐              │
                         │ mlx5_core                    │              │
                         │ ┌──────────────────────────┐ │              │
                         │ │ struct devlink           │ │              │
                         │ │ ┌──────────────────────┐ │ │              │
                         │ │ │ mlx5_core_dev (mdev) │ │ │              │
                         │ │ │  priv.adev[] ─┐      │ │ │              │
                         │ │ └───────────────┼──────┘ │ │              │
                         │ └─────────────────┼────────┘ │              │
                         └───────────────────┼──────────┘              │
                                             │                         │
            ┌────────────────────────────────┼───────────────────┐     │
            ▼                                ▼                   ▼     │
 ┌──────────────────────┐   ┌──────────────────────┐   ┌──────────────────────┐
 │ mlx5_adev            │   │ mlx5_adev            │   │ mlx5_adev            │
 │  adev: "eth", id=N   │   │  adev: "rdma", id=N  │   │  adev: "vnet", id=N  │
 │  mdev ──► mdev       │   │  mdev ──► mdev       │   │  mdev ──► mdev       │
 │  idx  = ETH          │   │  idx  = IB           │   │  idx  = VNET         │
 └──────────┬───────────┘   └──────────┬───────────┘   └──────────┬───────────┘
   auxiliary bus: match by name "mlx5_core.<suffix>"              │
            ▼                          ▼                          ▼
 ┌──────────────────────┐   ┌──────────────────────┐   ┌──────────────────────┐
 │ mlx5e  (Ethernet)    │   │ mlx5_ib  (RDMA)      │   │ mlx5_vdpa            │
 │  mlx5e_probe()       │   │  mlx5r_probe()       │   │                      │
 │  container_of → mdev │   │  container_of → mdev │   │  container_of → mdev │
 └──────────┬───────────┘   └──────────┬───────────┘   └──────────┬───────────┘
            ▼                          ▼                          ▼
     struct net_device          struct ib_device           vdpa device
       (eth0, ...)               (mlx5_0, ...)
```

### 6.8 Why this design

- **Modularity.** If `mlx5_ib` is not loaded, the `rdma` aux device just has no driver
  bound. Ethernet is unaffected.
- **Clean ordering.** `mlx5_core` adds the aux devices only after the hardware is up,
  and removes them first on teardown. Function drivers see ordinary probe/remove.
- **Runtime reconfiguration.** Changing the eswitch mode, a devlink reload, or a
  devlink param such as `enable_roce` can add or remove individual aux devices
  without touching the others.
- **Standard mechanism.** It replaces mlx5's older private interface registration
  list (`mlx5_register_interface`) with a generic kernel bus.

---

## 7. Full lifecycle

```
 PCI enumeration ─► pci_dev exists (PCI core)
        │
        ▼  driver bind
 probe_one(pdev, id)                                          main.c
   ├─ mlx5_devlink_alloc()          devlink + mlx5_core_dev born together
   ├─ dev->device / dev->pdev / dev->coredev_type
   ├─ dev->priv.adev_idx = mlx5_adev_idx_alloc()   the ".N" for aux devices
   ├─ mlx5_mdev_init()
   │    └─ mlx5_adev_init()          allocate priv.adev[] (empty)
   ├─ mlx5_pci_init()
   │    ├─ pci_set_drvdata(pdev, dev)   reverse link pdev → mdev
   │    └─ bar_addr = pci_resource_start(pdev, 0), map init segment, ...
   └─ mlx5_init_one()
        ├─ firmware handshake, command interface, caps, EQs, ...
        └─ mlx5_register_device()                             dev.c
             └─ for each protocol where is_supported():
                  add_adev()        ─► aux device "mlx5_core.<suffix>.N"
                                         │
                                         ▼  aux bus matches by name
                                    mlx5e_probe() / mlx5r_probe() / ...
                                         └─ net_device / ib_device created

 ───────────────────────────── device is running ─────────────────────────────

 devlink reload / reset (mlx5_unload_one / mlx5_load_one):
   mlx5_detach_device(dev, suspend)                           dev.c
     for each aux device (in reverse order):
       suspend && driver has .suspend ─► adrv->suspend()   (aux device kept)
       otherwise                       ─► del_adev()        (aux device deleted)
   ... hardware re-init ...
   mlx5_attach_device()
     kept device       ─► adrv->resume()
     deleted / missing ─► add_adev() again (if is_enabled / is_supported)

 remove_one(pdev)                                             main.c
   ├─ mlx5_uninit_one()
   │    └─ mlx5_unregister_device()   delete all aux devices
   │                                  ─► mlx5e/mlx5_ib remove: netdev/ibdev gone
   ├─ mlx5_pci_close()               unmap BAR, disable PCI device
   ├─ mlx5_mdev_uninit()             frees priv.adev[] (mlx5_adev_cleanup)
   ├─ mlx5_adev_idx_free()           release the ".N" index
   └─ mlx5_devlink_free()            devlink + mlx5_core_dev freed together
        │
        ▼  driver unbind
 pci_dev still exists (owned by PCI core, outlives the driver binding)
```

---

## 8. Cheat sheet

**"I have X, I need Y"**

| I have | I need | Code |
|---|---|---|
| `struct pci_dev *pdev` | `mlx5_core_dev` | `pci_get_drvdata(pdev)` |
| `struct mlx5_core_dev *dev` | `pci_dev` | `dev->pdev` |
| `struct mlx5_core_dev *dev` | `struct device` | `dev->device` (== `&pdev->dev`) |
| `struct mlx5_core_dev *dev` | `devlink` | `priv_to_devlink(dev)` |
| `struct devlink *devlink` | `mlx5_core_dev` | `devlink_priv(devlink)` |
| `struct auxiliary_device *adev` | `mlx5_core_dev` | `container_of(adev, struct mlx5_adev, adev)->mdev` |

**Ownership summary**

| Object | Allocated by | Freed by | Lifetime |
|---|---|---|---|
| `pci_dev` | PCI core | PCI core | Longer than the driver binding |
| `devlink` + `mlx5_core_dev` | `probe_one()` via `devlink_alloc()` | `devlink_free()` | Probe → remove |
| `mlx5_adev` | `add_adev()` | `adev_release()` | Register → unregister (may cycle on eswitch mode change / param change) |
| `net_device`, `ib_device` | `mlx5e` / `mlx5_ib` probe | their remove | While bound to the aux device |

**Five things to remember**

1. `mlx5_core` is the only PCI driver. Everything else is an auxiliary driver.
2. `mlx5_core_dev` lives *inside* the devlink allocation (`devlink->priv`).
3. In mlx5 core code, `dev` means `mlx5_core_dev`, not `struct device`.
4. Every aux driver reaches the shared core through `mlx5_adev->mdev`.
5. Aux devices are named `mlx5_core.<suffix>.<adev_idx>` and are parented to `&pdev->dev`.
