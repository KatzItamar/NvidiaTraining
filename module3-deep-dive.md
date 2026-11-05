# Module 3 deep dive: from the app to the firmware

This page expands [Module 3](README.md#module-3-how-mlx5-doca-host-and-doca-fit-together) with a one-week study plan, an explanation of the setup commands, one call traced through every layer, and a map of the main interfaces between the layers. All commands run in Ubuntu (WSL). Type `wsl` first if your prompt starts with `PS C:\`. Every function name, file and opcode below was checked against the upstream sources.

## Study plan (about one week, one hour a day)

**Day 1: Read and draw.** Read Module 3. Then close it and draw the four layers from memory: DOCA apps and SDK; user space (rdma-core `libmlx5` and the DPDK mlx5 driver); kernel (`mlx5_core` and `mlx5_ib`); firmware and hardware. Add DOCA-Host as the box that packages the middle two layers for customers. Compare with the diagram and fix what you missed.

**Day 2: Get the source for each layer.** See [Getting the source, explained](#getting-the-source-explained).

**Day 3: Follow one call through all the layers.** See [One call, end to end](#one-call-end-to-end-dpdk-creates-a-send-queue).

**Day 4: DOCA-Host and DOCA (reading).**

- Read the DOCA-Host install guide and profiles at [docs.nvidia.com/doca](https://docs.nvidia.com/doca/sdk/index.html). Write down which profile (`doca-ofed`, `doca-roce`, `doca-networking`, `doca-all`) your main customers use.
- Learn this fact: the kernel part of DOCA-Host (`mlnx-ofa_kernel`) is a backport of the upstream mlx5 you cloned on Day 2. It isn't a separate driver.
- Skim the DOCA SDK library list (Flow, DMA, Comch, RDMA, GPUNetIO) and note which ones sit on top of the DPDK mlx5 driver or rdma-core.
- In WSL, `lsmod | grep mlx5` shows nothing because a laptop has no ConnectX NIC. To see loaded drivers and run `ofed_info -s`, you need a lab server.

**Day 5: Talk to your team.** Book 30 minutes with a senior engineer and ask:

1. Walk me through one recent feature across firmware, kernel, user space and DOCA. Which patches went where, and when?
2. Which of our changes are upstream, and which live only in DOCA-Host?
3. Which kernel merge window is our next feature targeting?

**Self-check.** You're done when you can answer without notes:

- What is the difference between `mlx5_core` and `mlx5_ib`?
- Where does `libmlx5` live, and how do you ask for a change there? (A GitHub PR to rdma-core.)
- Why does DOCA-Host exist if the drivers are already in the upstream kernel?
- If a feature misses a kernel merge window, how long does it wait? (About 9 to 10 weeks.)
- What does "upstream first" save the company?

## Getting the source, explained

**1. Make a home for source code.**

```bash
mkdir -p ~/src && cd ~/src
```

`~` is your home folder in Ubuntu. `mkdir -p` creates the folder and doesn't complain if it already exists. `&&` runs the next command only if the first one worked. `cd` moves you into the folder.

**2. Clone only the mlx5 part of the Linux kernel.**

```bash
git clone --depth 1 --filter=blob:none --sparse https://github.com/torvalds/linux.git
```

The full kernel is about 5 GB with more than 1.3 million commits. Three options shrink the download:

| Option | What it does | Analogy |
| --- | --- | --- |
| `--depth 1` | Only the latest commit, no history | A photo of today, not the whole album |
| `--filter=blob:none` | The list of files, but contents only when needed | A library catalog, not the books |
| `--sparse` | Only top-level files on disk, no folders yet | An empty bookshelf, ready to fill |

```bash
cd ~/src/linux && git sparse-checkout set drivers/net/ethernet/mellanox/mlx5 drivers/infiniband/hw/mlx5 include/linux/mlx5 && cd ~/src
```

`git sparse-checkout set` tells Git which folders you want on disk:

- `drivers/net/ethernet/mellanox/mlx5`: **mlx5_core**, the Ethernet part (netdev, eswitch, SR-IOV, devlink)
- `drivers/infiniband/hw/mlx5`: **mlx5_ib**, the RDMA and InfiniBand part
- `include/linux/mlx5`: shared headers, including `mlx5_ifc.h`, which defines every command the driver can send to the NIC firmware

To also read the generic RDMA layer used in the trace below: `git sparse-checkout add drivers/infiniband/core`.

**3. Clone rdma-core (user-space RDMA).**

```bash
git clone --depth 1 https://github.com/linux-rdma/rdma-core.git
```

The mlx5 part, **libmlx5**, is in `providers/mlx5/`. RDMA apps (MPI, NCCL, storage) use it to talk to the NIC.

**4. Clone DPDK.**

```bash
git clone --depth 1 https://github.com/DPDK/dpdk.git
```

DPDK is a framework for very fast packet processing in user space. The NVIDIA drivers are in `drivers/net/mlx5/` (the PMD, which handles packets) and `drivers/common/mlx5/` (shared code, including the calls into rdma-core).

**5. Measure how much code your team owns.**

```bash
cd ~/src && for d in linux/drivers/net/ethernet/mellanox/mlx5 linux/drivers/infiniband/hw/mlx5 rdma-core/providers/mlx5 dpdk/drivers/net/mlx5 dpdk/drivers/common/mlx5; do printf "%8d  %s\n" $(find $d -name '*.[ch]' -print0 | xargs -0 cat | wc -l) $d; done
```

This prints the number of lines of C code in each mlx5 folder. Together they come to several hundred thousand lines, spread over **three separate communities** (kernel, rdma-core, DPDK), each with its own maintainers, reviews and release schedule.

## One call, end to end: DPDK creates a send queue

This traces how a DPDK app creates a send queue (SQ) on the NIC, through every layer.

```
DPDK app (testpmd, OVS-DPDK, DOCA Flow)
   |
   v  1  DPDK      drivers/common/mlx5/mlx5_devx_cmds.c  mlx5_devx_cmd_create_sq()
   |                builds the firmware command CREATE_SQ in memory
   v  2  DPDK      drivers/common/mlx5/linux/mlx5_glue.c  mlx5_glue_devx_obj_create()
   ======== project boundary: DPDK -> rdma-core ========
   v  3  rdma-core providers/mlx5/verbs.c  mlx5dv_devx_obj_create()
   |                return dvops->devx_obj_create(...)
   v  4  rdma-core providers/mlx5/verbs.c  _mlx5dv_devx_obj_create()
   v  5  rdma-core libibverbs/cmd_ioctl.c  ioctl(cmd_fd, RDMA_VERBS_IOCTL)
   ======== user space -> kernel (system call) ========
   v  6  kernel    drivers/infiniband/core (ib_uverbs) routes to mlx5_ib
   v  7  kernel    drivers/infiniband/hw/mlx5/devx.c  DEVX_OBJ_CREATE handler
   ======== module boundary: mlx5_ib -> mlx5_core ========
   v  8  kernel    mlx5/core/cmd.c  mlx5_cmd_do()
   v  9  kernel    mlx5/core/cmd.c  iowrite32be(..., &dev->iseg->cmd_dbell)
   ======== driver -> hardware (PCIe) ========
   v  10 firmware  runs opcode 0x904 CREATE_SQ (mlx5_ifc.h), returns the queue number
   ^  the answer travels back up the same path
```

1. **DPDK builds the command.** `mlx5_devx_cmd_create_sq()` fills a buffer field by field with `MLX5_SET(sqc, sq_ctx, cqn, ...)`. That buffer is the firmware command, in the exact binary layout the NIC expects. This technique is called **DevX**: user space writes the hardware command and asks the kernel only to deliver it.
2. **The glue layer.** `mlx5_glue->devx_obj_create(...)` calls rdma-core. The glue table lets the same driver run on Linux and Windows.
3. **The ops table.** `mlx5dv_devx_obj_create()` looks up `dvops`, a table of function pointers, and calls `dvops->devx_obj_create`. rdma-core fills it in `mlx5_set_dv_ctx_ops()`. There are two back ends: the normal path through the kernel (`_mlx5dv_devx_obj_create`) and a VFIO path that talks to the NIC directly (`vfio_devx_obj_create` in `mlx5_vfio.c`). If neither exists, the call returns `EOPNOTSUPP`.
4. **Pack it for the kernel.** `_mlx5dv_devx_obj_create` wraps the command into an ioctl message with three attributes: `CMD_IN` (the command), `CMD_OUT` (room for the answer) and `HANDLE` (an ID to destroy the object later).
5. **Cross into the kernel.** `execute_ioctl()` calls the `ioctl()` system call on `/dev/infiniband/uverbsN`.
6. **RDMA core routing.** The kernel's generic RDMA layer (`ib_uverbs`) sees driver mlx5, object DEVX_OBJ, method CREATE, and calls the mlx5 handler.
7. **mlx5_ib checks and forwards.** In `devx.c` the kernel refuses anything that isn't a create command, stamps the command with the process's **uid** so firmware isolates this app's objects (the security boundary), and calls `mlx5_cmd_do()`.
8. **mlx5_core sends it.** `mlx5_cmd_do()` is the single entry point for every firmware command: Ethernet, RDMA, devlink and DevX.
9. **Doorbell.** The command goes into a slot in memory shared with the NIC, then a write to a hardware register tells the NIC "command waiting in slot N".
10. **Firmware** runs `CREATE_SQ`, creates the queue and writes the result. `cmd_status_err()` turns firmware errors into Linux error codes, and DPDK reads the queue number with `MLX5_GET(create_sq_out, out, sqn)`.

**What this means for you:**

- **One feature, four codebases.** A new firmware object needs a layout in `mlx5_ifc.h`, maybe an allow-list change in `devx.c`, maybe a helper in rdma-core, and the caller in DPDK: four reviews and four release schedules.
- **Why DevX matters.** Many new hardware features reach DPDK and DOCA without a kernel change. That's faster, but it depends on the customer's kernel being new enough.
- **Where bugs show up.** A firmware error often appears in DPDK logs as "create SQ failed". This map is how the team works out which layer is at fault.

## The main interfaces between the layers

The driver talks to the NIC in only three ways:

| Channel | Direction | Used for | How it works |
| --- | --- | --- | --- |
| Command interface | Driver to firmware | Configuration and teardown (slow path) | The driver writes a command (opcode from `mlx5_ifc.h`) into a mailbox, rings the command doorbell, firmware replies |
| Queues and doorbells | Driver and hardware | RX/TX packets (fast path) | The driver writes descriptors (WQEs) into rings and rings a doorbell. The NIC writes completions (CQEs). No firmware command per packet |
| Event queues (EQ) | NIC to driver | Interrupts, link changes, errors, page requests | The NIC writes an event and raises an interrupt |

There are two app paths: the **kernel netdev** path (sockets, `ip`, `ethtool`) and the **DPDK / DOCA** path (user space, DevX through rdma-core). Read each table top to bottom.

### 1. NIC configuration

**Driver load** (boot or `modprobe mlx5_core`, no app involved):

| Layer | What happens |
| --- | --- |
| Kernel PCI | Finds the ConnectX device and calls mlx5 probe |
| mlx5_core | `mlx5_load` and `mlx5_function_setup` (`main.c`), `mlx5_query_hca_caps` (`fw.c`), `mlx5_satisfy_startup_pages` (`pagealloc.c`), `mlx5_cmd_init_hca` |
| Firmware | `ENABLE_HCA 0x104`, `QUERY_HCA_CAP 0x100`, `SET_HCA_CAP 0x109`, `MANAGE_PAGES 0x108` (gives host memory to the NIC), `INIT_HCA 0x102`, `CREATE_EQ 0x301` |

**Bring a port up and create queues:**

| Layer | Kernel netdev path | DPDK / DOCA path |
| --- | --- | --- |
| App | `ip link set eth0 up`, `ip link set eth0 mtu 9000`, `ethtool -L eth0 combined 8` | `rte_eth_dev_configure()`, `rte_eth_rx_queue_setup()`, `rte_eth_tx_queue_setup()`, `rte_eth_dev_start()` |
| API | netlink (rtnetlink, ethtool-netlink) to the kernel net core | Direct call into the PMD through `eth_dev_ops` |
| User-space driver | none | `mlx5_dev_configure`, `mlx5_rx_queue_setup`, `mlx5_tx_queue_setup`, `mlx5_dev_start`, then `mlx5_rxq_devx_obj_new` / `mlx5_txq_devx_obj_new` (`mlx5_devx.c`), then `mlx5_devx_cq_create` / `mlx5_devx_sq_create` (`common/mlx5`) |
| rdma-core | none | `mlx5dv_devx_umem_reg` (register queue memory), `mlx5dv_devx_obj_create`, then `ioctl()` |
| Kernel entry | `net_device_ops`: `.ndo_open = mlx5e_open`, `.ndo_change_mtu = mlx5e_change_nic_mtu`. `ethtool_ops`: `.set_channels = mlx5e_set_channels` | `ib_uverbs`, then the mlx5_ib `devx.c` handler |
| Kernel mlx5 driver | `mlx5e_open`, `mlx5e_open_channels`, `mlx5e_create_sq` / RQ / CQ, then `mlx5_core_create_sq`, `mlx5_core_create_cq` (`transobj.c`, `cq.c`), `mlx5_set_port_mtu` (`port.c`) | mlx5_ib passes the command to `mlx5_cmd_do` |
| Command layer | `mlx5_cmd_exec`, `mlx5_cmd_do`, doorbell (`cmd.c`) | Same |
| Firmware | `CREATE_CQ 0x400`, `CREATE_RQ 0x908`, `CREATE_SQ 0x904`, `CREATE_TIS 0x912`, `CREATE_TIR 0x900`, `CREATE_RQT 0x916`, then `MODIFY_RQ 0x909` / `MODIFY_SQ 0x905` (reset to ready). MTU through `ACCESS_REG 0x805` | Same opcodes, built by DPDK itself |

### 2. RX / TX packets (fast path)

The most important point: **there is no firmware command here.** Packets move through memory rings and doorbells only.

**TX (send):**

| Layer | Kernel netdev path | DPDK / DOCA path |
| --- | --- | --- |
| App | `send()` / `write()` on a socket | `rte_eth_tx_burst(port, queue, pkts, n)` |
| Stack | TCP/IP, then `dev_queue_xmit` | none (no kernel) |
| Driver | `.ndo_start_xmit = mlx5e_xmit` (`en_tx.c`) writes a WQE into the SQ ring | PMD TX burst functions (`mlx5_tx_*.c`, generated with `MLX5_TXOFF_DECL`) write WQEs into the SQ ring |
| Doorbell | `mlx5e_notify_hw` (`en/txrx.h`): MMIO write to the UAR page | Same doorbell, written directly from user space (the UAR page is mapped into the app) |
| Hardware | NIC reads the WQE and packet by DMA, sends it, writes a CQE | Same |
| Completion | Interrupt, then NAPI `mlx5e_napi_poll`, then `mlx5e_poll_tx_cq` frees the buffer | The PMD polls the CQ on the next burst, no interrupt |

**RX (receive):**

| Layer | Kernel netdev path | DPDK / DOCA path |
| --- | --- | --- |
| Ahead of time | Driver fills the RQ with empty buffers and rings the doorbell | Same, in the PMD |
| Hardware | Packet arrives, steering picks the RQ, DMA into a buffer, CQE written | Same |
| Driver | Interrupt (EQ), `mlx5e_napi_poll` (`en_txrx.c`), `mlx5e_poll_rx_cq` (`en_rx.c`) builds the skb, `mlx5e_cq_arm` re-arms the CQ | `rte_eth_rx_burst`, then `mlx5_rx_burst` / `mlx5_rx_burst_vec` polls the CQ |
| App | Stack (GRO, IP, TCP), socket, `recv()` | Gets `rte_mbuf` packets directly |
| Firmware | none | none |

**Why DPDK is fast:** after setup, the kernel and firmware are out of the loop. No system calls, no interrupts, no copies.

### 3. Deconfiguration (teardown)

| Layer | Kernel netdev path | DPDK / DOCA path |
| --- | --- | --- |
| App | `ip link set eth0 down`, `rmmod mlx5_core` | `rte_flow_destroy()`, `rte_eth_dev_stop()`, `rte_eth_dev_close()` |
| User-space driver | none | `mlx5_flow_destroy`, `mlx5_dev_stop`, `mlx5_dev_close`, then `mlx5dv_devx_obj_destroy` and `ioctl()` |
| Kernel | `.ndo_stop = mlx5e_close`, `mlx5e_close_channels`, `mlx5e_destroy_sq`, `mlx5_core_destroy_sq`, `mlx5_core_destroy_cq` | mlx5_ib `devx.c` sends a destroy command it built at create time (`devx_obj_build_destroy_cmd`) |
| Driver unload | `mlx5_unload`, `mlx5_function_teardown`, `mlx5_cmd_teardown_hca` | none |
| Firmware | `DESTROY_SQ 0x906`, `DESTROY_RQ 0x90a`, `DESTROY_CQ 0x401`, `DESTROY_EQ 0x302`, then `TEARDOWN_HCA 0x103`, `MANAGE_PAGES` (reclaim), `DISABLE_HCA 0x105` | Same destroy opcodes |

**Manager point:** if a DPDK app crashes, the kernel still cleans up. Its file handle closes and mlx5_ib runs the saved destroy commands. Leaks here are a classic source of bugs, so ask about tests for it.

### 4. Others

**Statistics and link:**

| App | Driver | Firmware |
| --- | --- | --- |
| `ethtool -S eth0` | `.get_ethtool_stats = mlx5e_get_ethtool_stats`, `mlx5_core_query_vport_counter` (`vport.c`) | `QUERY_VPORT_COUNTER 0x770`, port counters through `ACCESS_REG 0x805` |
| `ethtool eth0` (speed, link) | `.get_link_ksettings = mlx5e_get_link_ksettings` | `ACCESS_REG` (port registers such as PTYS) |
| `rte_eth_stats_get()` | `mlx5_stats_get` in the PMD, mostly software counters | none, or through DevX |

**Flow steering and offloads** (which packet goes to which queue, drop, encap):

| App | Driver | Firmware |
| --- | --- | --- |
| `ethtool -N`, `tc flower ... skip_sw`, OVS | mlx5_core flow steering (`fs_core.c`, `fs_cmd.c`) | `CREATE_FLOW_TABLE 0x930`, `SET_FLOW_TABLE_ENTRY 0x936`, `DELETE_FLOW_TABLE_ENTRY 0x938`, `ALLOC_FLOW_COUNTER 0x939` |
| `rte_flow_create()` / DOCA Flow | `mlx5_flow_create` (`mlx5_flow.c`), then DevX, or hardware steering (HWS), which writes rules straight to NIC memory | Same opcodes, or none for HWS rule inserts |

**Device management (devlink):**

| App | Kernel driver (`devlink.c`) | Firmware |
| --- | --- | --- |
| `devlink dev eswitch set pci/... mode switchdev` | `mlx5_devlink_eswitch_mode_set` | vport and eswitch commands, `MODIFY_NIC_VPORT_CONTEXT 0x755` |
| `devlink dev info` | `mlx5_devlink_info_get` | `QUERY_HCA_CAP`, `ACCESS_REG` (firmware versions) |
| `devlink dev flash` | `mlx5_devlink_flash_update` | Flash update through `ACCESS_REG` |
| `devlink dev reload` | `mlx5_devlink_reload_down` / `_up` | Full teardown, then load again |

**RDMA (verbs apps: MPI, NCCL, storage):**

| App | rdma-core (`libmlx5`) | Kernel (mlx5_ib) | Firmware |
| --- | --- | --- | --- |
| `ibv_open_device`, `ibv_alloc_pd` | `mlx5_alloc_pd`, then `ioctl` | `mlx5_ib_alloc_pd` | `ALLOC_PD 0x800` |
| `ibv_reg_mr` | `ioctl` | Pins memory, creates an MKEY | `CREATE_MKEY 0x200` |
| `ibv_create_cq`, `ibv_create_qp`, `ibv_modify_qp` | `mlx5_create_qp` and others, then `ioctl` | `mlx5_ib_create_qp` and others | `CREATE_CQ 0x400`, `CREATE_QP 0x500`, `MODIFY_QP` (RST, INIT, RTR, RTS) |
| `ibv_post_send`, `ibv_poll_cq` | Writes a WQE and rings the doorbell from user space | none | none (fast path, like DPDK) |

**Events (NIC to driver).** The NIC writes to an EQ (created with `CREATE_EQ 0x301`) and mlx5_core handles it: link up/down changes the netdev carrier; a firmware request for memory triggers `MANAGE_PAGES`; a command completion wakes the waiting `mlx5_cmd_do`; errors feed the devlink health reporters.

**Firmware tools (outside the driver).** `mlxconfig`, `mlxfwmanager` and `mstflint` talk to firmware directly through PCI. They set persistent options (for example, SR-IOV and the number of VFs) that apply after a reboot or firmware reset.

## Summary

**Setup and teardown use firmware commands (opcodes). Packets use rings and doorbells, with no firmware. Events come back to the driver through EQs.** The kernel path enters through netlink and `net_device_ops`. The DPDK and DOCA path enters through rdma-core DevX `ioctl()` for setup, then bypasses the kernel completely for packets.

To see which firmware commands a driver file sends:

```bash
cd ~/src/linux && grep -rn "MLX5_CMD_OP_" drivers/net/ethernet/mellanox/mlx5/core/transobj.c | head -20
```
