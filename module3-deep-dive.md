# Module 3 deep dive: from the app to the firmware

This page builds up in four steps. Read it in order. Each part assumes only the parts before it.

1. [Use cases](#part-1-use-cases-what-customers-do-with-the-nic): what customers do with an NVIDIA NIC
2. [The basics](#part-2-the-basics): how an NVIDIA NIC works, and what RDMA is
3. [End to end](#part-3-end-to-end-configure-send-receive): configure, send and receive, from the app down to the firmware, with the API at each layer, plus [NCCL](#35-nccl-the-biggest-rdma-customer), the AI library that drives door 3
4. [Deep dive](#part-4-deep-dive): one call traced line by line, and the full interface tables

At the end: [hands-on lab and study plan](#part-5-hands-on-lab-and-study-plan).

---

## Part 1: Use cases: what customers do with the NIC

Your drivers serve very different customers. Each use case enters the NIC through a different door, and a different part of your code carries it.

| Use case | Example customers | How the app talks to the NIC | Your code that carries it |
| --- | --- | --- | --- |
| **Ordinary server networking** | Web servers, databases, Kubernetes pods | Normal sockets (`send`, `recv`) through the Linux kernel | `mlx5_core` (Ethernet part, called `mlx5e`) |
| **Cloud and virtualization** | Cloud providers, VMs, OpenStack | One NIC split into many virtual NICs (SR-IOV VFs), virtual switch offloaded to the NIC (OVS, `tc`) | `mlx5_core` eswitch, SR-IOV, devlink |
| **High-rate packet processing** | Firewalls, load balancers, 5G core, telecom | DPDK: the app drives the NIC directly from user space | DPDK mlx5 driver (PMD) + rdma-core |
| **AI training and HPC** | GPU clusters, NCCL, MPI | RDMA verbs: GPUs and servers read and write each other's memory | rdma-core `libmlx5` + `mlx5_ib` |
| **Storage** | NVMe over Fabrics, NFS over RDMA, SMB Direct | RDMA, used by storage code inside the kernel | `mlx5_ib` |
| **DPU apps on BlueField** | Security, storage and network services running on the NIC's own Arm cores | DOCA libraries (DOCA Flow, DMA, Comch, ...) | DOCA, which uses the DPDK driver and rdma-core underneath |

So there are really **three doors** into the NIC:

| Door | Who uses it | Goes through the kernel for packets? |
| --- | --- | --- |
| **Kernel sockets** | Almost every normal app | Yes, every packet |
| **DPDK** | Packet-processing apps, DOCA Flow | No. Kernel only for setup |
| **RDMA verbs** | AI, HPC, storage | No. Kernel only for setup |

Keep these three doors in mind. Part 3 walks through each one.

---

## Part 2: The basics

### 2.1 How an NVIDIA NIC works

A NIC (network interface card) moves data between the server's memory and the network cable. NVIDIA makes three families:

- **ConnectX**: the NIC itself (for example ConnectX-7). Ethernet or InfiniBand, up to 400 Gb/s per port and beyond.
- **BlueField DPU**: a ConnectX plus Arm CPU cores and memory on the same card. It runs its own Linux, so services can run on the card instead of the server.
- **SuperNIC**: a BlueField tuned for AI clusters.

All of them use the same driver family: **mlx5**.

Inside the card there are four main parts:

```
                    SERVER (host)
   +------------------------------------------------+
   |  CPU / GPU          Host memory                |
   |                     +-------------------------+|
   |                     | Rings (queues):         ||
   |                     |  SQ = send queue        ||
   |                     |  RQ = receive queue     ||
   |                     |  CQ = completion queue  ||
   |                     | Packet buffers          ||
   |                     +-------------------------+|
   +------------------------|-----------------------+
                            | PCIe (DMA reads/writes + doorbells)
   +------------------------|-----------------------+
   |  NVIDIA NIC            v                       |
   |  [1] PCIe + DMA engines: copy data to/from     |
   |      host memory by themselves                 |
   |  [2] Firmware: small processors that run       |
   |      setup commands from the driver            |
   |  [3] Packet pipeline (hardware): parse, steer, |
   |      checksum, encrypt, RDMA transport, switch |
   |  [4] Ports: to the Ethernet or InfiniBand wire |
   +------------------------------------------------+
```

The key idea: **the driver and the NIC talk through memory.** The driver doesn't hand each packet to the NIC. Instead:

1. The driver creates **rings** in host memory, with a firmware command (once, at setup).
2. To send, the driver writes a small descriptor (a **WQE**, "work queue entry") into the send ring, then writes to a NIC register called a **doorbell**: "there is new work".
3. The NIC's DMA engine reads the descriptor and the packet from host memory by itself and sends it.
4. When done, the NIC writes a **CQE** ("completion queue entry") into the completion ring: "packet sent".
5. Receive is the mirror image: the driver puts empty buffers in the receive ring ahead of time. The NIC writes arriving packets into them and adds a CQE.

So there are two very different speeds of work:

| | Slow path (control) | Fast path (data) |
| --- | --- | --- |
| What | Create and destroy rings, set MTU, add steering rules, read counters | Send and receive packets |
| How | **Firmware command**: the driver writes a command to a mailbox, rings the command doorbell, firmware runs it and answers | **Rings + doorbells**: no firmware involved |
| How often | Rarely (setup, changes) | Millions of times per second |

There is also a third channel, from the NIC back to the driver: **events** (link went down, error, "command finished"). The NIC writes them into an event queue (EQ) and raises an interrupt.

Other things the hardware does on its own (offloads): checksums, splitting large sends into packets (TSO), spreading traffic over CPU cores (RSS), VXLAN tunnels, encryption (IPsec, TLS, MACsec), and a full virtual switch for VMs (eswitch).

### 2.2 What is RDMA

**RDMA (Remote Direct Memory Access)** lets an app on one server read or write memory on another server directly, through the NICs, **without the remote CPU or the kernel touching the data**.

Analogy: normal networking is like posting a letter. The post office (kernel) on each side receives, sorts and copies it. RDMA is like having a key to a shelf in the other building: you place the box on the shelf yourself, and the other side finds it there.

Why customers want it:

- **Low latency:** around 1 to 2 microseconds instead of tens of microseconds.
- **No CPU cost:** the NICs do the transfer, so CPUs and GPUs keep computing.
- **Zero copy:** data goes straight from app memory to app memory. With **GPUDirect RDMA**, it goes straight from GPU memory to GPU memory. That is why AI clusters depend on it.

RDMA runs over two kinds of network:

- **InfiniBand**: a separate network type, common in AI and HPC clusters.
- **RoCE** ("rocky", RDMA over Converged Ethernet): RDMA inside normal Ethernet/UDP packets (UDP port 4791).

The main RDMA objects (you will hear these words every day):

| Object | What it is |
| --- | --- |
| **PD** (protection domain) | A security group: objects in one PD can work together |
| **MR** (memory region) | A piece of app memory registered with the NIC, so the NIC may read or write it. Gives a local key (lkey) and a remote key (rkey) |
| **QP** (queue pair) | A connection endpoint: one send queue + one receive queue. Like a socket |
| **CQ** (completion queue) | Where the NIC reports "work done" |
| **Verbs** | The API for all of this: `ibv_reg_mr`, `ibv_create_qp`, `ibv_post_send`, `ibv_poll_cq`, ... |

Main operations: **RDMA WRITE** (put data into remote memory), **RDMA READ** (fetch data from remote memory), and **SEND/RECEIVE** (a message the other side has posted a buffer for).

### 2.3 The layers, from app to wire

```
  App               web server       DPDK app / DOCA      AI / MPI / storage
                    (sockets)        (rte_eth_*, doca_*)  (ibv_* verbs)
  ---------------------------------------------------------------------------
  User-space lib    libc             DPDK mlx5 driver     rdma-core libmlx5
                                     + rdma-core (setup)
  ---------------------------------------------------------------------------
  Kernel            TCP/IP stack     ib_uverbs + mlx5_ib (setup only)
                    mlx5_core (mlx5e)
                    mlx5_core command interface (all firmware commands)
  ---------------------------------------------------------------------------
  NIC               firmware (commands)  +  hardware (rings, DMA, pipeline)
```

The two most important facts on this page:

1. **Every setup action, from any door, ends as a firmware command sent by `mlx5_core`.**
2. **For DPDK and RDMA, the kernel is not on the packet path at all.** The app writes descriptors and rings doorbells itself.

---

## Part 3: End to end: configure, send, receive

Each flow below is one row per layer, from top to bottom. The **API** column is the interface the layer above uses to call the layer below.

### 3.1 Door 1: kernel sockets (example: a web server)

**Configure (bring the port up):**

| Step | Layer | API / what happens |
| --- | --- | --- |
| 1 | Admin | `ip link set eth0 up` |
| 2 | User space to kernel | **netlink** message (the kernel's configuration API) |
| 3 | Kernel net core | calls the driver through **`net_device_ops`** (`.ndo_open`) |
| 4 | mlx5 driver | `mlx5e_open`: allocates rings in host memory |
| 5 | Driver to firmware | **firmware commands**: `CREATE_CQ`, `CREATE_RQ`, `CREATE_SQ`, then "move to ready" |
| 6 | NIC | rings are live, link comes up, an **event** tells the driver "link up" |

**Send a packet:**

| Step | Layer | API / what happens |
| --- | --- | --- |
| 1 | App | `send(socket, data)` |
| 2 | App to kernel | **system call** |
| 3 | Kernel TCP/IP | builds TCP/IP headers, picks the NIC |
| 4 | Kernel to driver | **`net_device_ops`** (`.ndo_start_xmit` = `mlx5e_xmit`) |
| 5 | mlx5 driver | writes a WQE into the send ring, rings the **doorbell** |
| 6 | NIC hardware | DMA-reads the packet, adds checksums, sends it on the wire, writes a CQE |
| 7 | Driver | on interrupt, reads the CQE and frees the buffer |

**Receive a packet:**

| Step | Layer | API / what happens |
| --- | --- | --- |
| 1 | NIC hardware | packet arrives; steering picks a receive ring; DMA writes it into a buffer the driver posted earlier; writes a CQE; raises an **interrupt** |
| 2 | mlx5 driver | **NAPI** poll (`mlx5e_napi_poll`) reads the CQE and wraps the data as a kernel packet (skb) |
| 3 | Kernel TCP/IP | checks it, finds the socket |
| 4 | App | `recv(socket)` returns the data |

Firmware is used only in "configure". Every packet goes through the kernel.

### 3.2 Door 2: DPDK (example: a firewall)

**Configure:**

| Step | Layer | API / what happens |
| --- | --- | --- |
| 1 | App | `rte_eth_dev_configure()`, `rte_eth_rx_queue_setup()`, `rte_eth_tx_queue_setup()`, `rte_eth_dev_start()` (DPDK **ethdev API**) |
| 2 | DPDK mlx5 driver | allocates rings in the app's own memory, builds the firmware commands itself (**DevX**) |
| 3 | rdma-core | `mlx5dv_devx_obj_create()` (**mlx5 direct verbs API**) |
| 4 | User space to kernel | **`ioctl()`** on `/dev/infiniband/uverbsN` |
| 5 | Kernel mlx5_ib, then mlx5_core | security check, then sends the command |
| 6 | Firmware | `CREATE_CQ`, `CREATE_RQ`, `CREATE_SQ`, same commands as door 1 |

**Send and receive:**

| Step | Layer | API / what happens |
| --- | --- | --- |
| 1 | App | `rte_eth_tx_burst(port, queue, packets, n)` / `rte_eth_rx_burst(...)` |
| 2 | DPDK mlx5 driver | writes WQEs and rings the doorbell **directly from user space** / reads CQEs from memory |
| 3 | NIC hardware | same as door 1 |
| — | Kernel, firmware | **not involved** |

No system calls, no interrupts (the app polls), no copies. This is why one CPU core can handle tens of millions of packets per second.

### 3.3 Door 3: RDMA verbs (example: two GPU servers training a model)

**Configure (once per connection):**

| Step | App calls (**verbs API**) | What happens below |
| --- | --- | --- |
| 1 | `ibv_open_device()` | opens `/dev/infiniband/uverbsN` |
| 2 | `ibv_alloc_pd()` | rdma-core `libmlx5` → `ioctl()` → `mlx5_ib` → firmware `ALLOC_PD` |
| 3 | `ibv_reg_mr(buffer)` | kernel pins the memory so it can't move; firmware `CREATE_MKEY` gives the lkey and rkey |
| 4 | `ibv_create_cq()`, `ibv_create_qp()` | firmware `CREATE_CQ`, `CREATE_QP`. The rings live in app memory |
| 5 | exchange QP number, address and rkey with the other server | usually with **librdmacm** (`rdma_connect`) or a normal socket |
| 6 | `ibv_modify_qp()` (RESET → INIT → RTR → RTS) | firmware `MODIFY_QP`: the connection is ready |

**Send (RDMA WRITE to the other server's memory):**

| Step | Layer | API / what happens |
| --- | --- | --- |
| 1 | App | `ibv_post_send(qp, RDMA_WRITE, local buffer, remote address, rkey)` |
| 2 | rdma-core `libmlx5` | writes a WQE into the send ring, rings the **doorbell from user space** |
| 3 | Local NIC | DMA-reads the buffer, sends RoCE or InfiniBand packets, handles retransmits in hardware |
| 4 | Remote NIC | checks the rkey, **writes the data straight into the remote app's memory**. The remote CPU and kernel do nothing |
| 5 | Local NIC → app | writes a CQE; the app sees it with `ibv_poll_cq()` |

For SEND/RECEIVE instead of WRITE, the remote app first posts a buffer with `ibv_post_recv()` and later finds a CQE saying "a message arrived".

Storage protocols (NVMe-oF, NFS over RDMA) use the same verbs, but from inside the kernel: they call `mlx5_ib` directly, with no rdma-core.

### 3.4 What the three doors share

| | Kernel sockets | DPDK | RDMA verbs |
| --- | --- | --- | --- |
| Setup API | netlink, `ethtool`, `ip` | `rte_eth_*` | `ibv_*` |
| Setup path | kernel → `mlx5_core` | rdma-core → `ioctl` → `mlx5_ib` → `mlx5_core` | rdma-core → `ioctl` → `mlx5_ib` → `mlx5_core` |
| Firmware commands | yes, at setup | yes, at setup | yes, at setup |
| Packet path | app → kernel → driver → NIC | app + driver → NIC | app + libmlx5 → NIC |
| Kernel per packet | yes | no | no |
| Typical users | everyone | firewalls, telecom, DOCA Flow | AI, HPC, storage |

### 3.5 NCCL: the biggest RDMA customer

**What it is.** NCCL (NVIDIA Collective Communications Library, said "nickel") is the library PyTorch, TensorFlow and JAX call when many GPUs must work as one. Its main operations are called *collectives*:

| Collective | What it does | Typical use in AI |
| --- | --- | --- |
| AllReduce | Every GPU adds its numbers and all get the sum | Combining gradients every training step |
| AllGather | Every GPU gets every other GPU's piece | Model sharding |
| ReduceScatter | Sum, then each GPU keeps one slice | Sharded training |
| Send / Recv | Point to point | Pipeline parallelism |

Inside one server, NCCL moves data over NVLink or PCIe. **Between servers, it uses the network: your NIC and your drivers.**

**Where it sits.** NCCL enters through door 3, RDMA verbs (3.3). Nothing in the data path is NCCL-specific: NCCL is simply the biggest and most demanding user of `libmlx5` and `mlx5_ib`.

```
 PyTorch            torch.distributed, backend="nccl"
    |  ncclAllReduce(...)
 NCCL               libnccl.so: rings and trees, proxy thread
    |  net plugin API: listen, connect, accept, regMr, isend, irecv, test
 NCCL IB transport  built in (src/transport/net_ib/), or a plugin (e.g. SHARP)
    |  ibv_reg_mr, ibv_create_qp, ibv_post_send (RDMA WRITE) ...   <- door 3
 rdma-core          libibverbs + libmlx5: writes WQEs, rings doorbells
    |  ioctl (setup only)
 mlx5_ib -> mlx5_core -> firmware commands (CREATE_QP 0x500, CREATE_MKEY 0x200 ...)
    |
 ConnectX / BlueField / SuperNIC -> InfiniBand or RoCE -> the other server
```

**Configure (once, when training starts):**

| Step | NCCL does | API it calls | Ends up in |
| --- | --- | --- | --- |
| 1 | Finds GPUs, NICs and the PCIe / NVLink layout | sysfs, `ibv_get_device_list`, `ibv_query_port` | `mlx5_ib` query paths |
| 2 | Picks the NIC closest to each GPU | its own topology logic | nothing below (pure software) |
| 3 | Opens the NIC, creates a PD and CQs | `ibv_open_device`, `ibv_alloc_pd`, `ibv_create_cq` | firmware ALLOC_PD 0x800, CREATE_CQ 0x400 |
| 4 | **Registers GPU memory** so the NIC can reach it | `ibv_reg_dmabuf_mr`, `mlx5dv_reg_dmabuf_mr`, or `ibv_reg_mr` | `mlx5_ib` → firmware CREATE_MKEY 0x200. GPU pages arrive through **dma-buf** or the **nvidia-peermem** module |
| 5 | Creates QPs and connects them to the peer (QP numbers are swapped over a TCP socket) | `ibv_create_qp`, `ibv_modify_qp` (INIT → RTR → RTS) | firmware CREATE_QP 0x500 and the QP state-change commands |

**Send and receive (every training step, millions of times):**

| Step | What happens | Firmware? |
| --- | --- | --- |
| 1 | PyTorch calls `ncclAllReduce`. NCCL's GPU kernel and its CPU proxy thread cut the data into chunks | no |
| 2 | The receiver tells the sender "write here" with a small RDMA WRITE (the *clear to send*) | no |
| 3 | The proxy thread posts the chunk as RDMA WRITEs straight from GPU memory, the last one as RDMA WRITE WITH IMMEDIATE, via `ibv_post_send` | no: `libmlx5` writes WQEs and rings the doorbell |
| 4 | The NIC reads GPU memory over PCIe (**GPUDirect RDMA**) and sends it | no: hardware |
| 5 | The remote NIC writes straight into the remote GPU's memory. The immediate value creates a CQE, so the receiver knows the chunk is complete | no |
| 6 | Both sides call `ibv_poll_cq` to collect completions. The receiver may do a small RDMA READ ("flush") so the GPU sees all the data | no |

This is the "fast path, no firmware" rule from 2.1, at enormous scale. (These steps match `src/transport/net_ib/p2p.cc` in NCCL v2.32.)

**NVIDIA pieces around NCCL that touch your area:**

| Piece | What it is | Link to your team |
| --- | --- | --- |
| GPUDirect RDMA | The NIC reads and writes GPU memory directly | Needs **dma-buf** (upstream, supported by `mlx5_ib`) or **nvidia-peermem** (older peer-memory interface) |
| GPU-initiated networking | Newer NCCL lets the GPU itself write WQEs and ring the doorbell, with no CPU proxy | Built on **DOCA GPUNetIO** (NCCL ships it under `src/transport/net_ib/gdaki/`). A direct DOCA ↔ mlx5 link |
| SHARP | Quantum InfiniBand switches do the AllReduce math inside the network | NCCL SHARP plugin ([nccl-rdma-sharp-plugins](https://github.com/Mellanox/nccl-rdma-sharp-plugins)), CollNet |
| Spectrum-X / SuperNIC | Ethernet (RoCE) built for AI: adaptive routing, congestion control | Firmware and mlx5 features, shipped with DOCA-Host |
| RoCE congestion control | DCQCN, ECN and PFC keep RoCE close to lossless | Set through mlx5 firmware and tools (`mlnx_qos`, devlink parameters) |
| UCX | Another communication library, used by MPI | Also built on `libmlx5` |

**What you will hear from NCCL users:**

- **"NCCL is slow."** Usual causes: the wrong NIC for the GPU, GPUDirect RDMA not active, or PCIe settings (ACS, relaxed ordering). First step: run with `NCCL_DEBUG=INFO`, which prints the NIC and transport chosen and whether GPUDirect (GDR) is on.
- **"transport retry counter exceeded"** (`IBV_WC_RETRY_EXC_ERR` in the CQE). Packets were lost or the peer died. Usually the fabric or congestion, sometimes driver or firmware.
- **Memory registration failures.** dma-buf not supported, nvidia-peermem not loaded, or a locked-memory limit (`ulimit -l`).
- **Settings to recognize:** `NCCL_IB_HCA` (which NICs), `NCCL_SOCKET_IFNAME` (which interface for setup traffic), `NCCL_IB_GID_INDEX` (RoCE addressing), `NCCL_NET_GDR_LEVEL`, `NCCL_IB_QPS_PER_CONNECTION`, `NCCL_IB_ADAPTIVE_ROUTING`, `NCCL_IB_PCI_RELAXED_ORDERING`, `NCCL_IB_TIMEOUT`, `NCCL_IB_RETRY_CNT`.
- **The standard benchmark:** `all_reduce_perf` from [nccl-tests](https://github.com/NVIDIA/nccl-tests). It is the first thing customers run, and the first number they quote.

---

## Part 4: Deep dive

Now the same flows with the real function names and firmware opcodes, checked against the upstream sources (Linux kernel, rdma-core, DPDK). File paths are relative to each project.

### 4.1 One call traced line by line: DPDK creates a send queue

This is step 2 to 6 of "Door 2: configure", in full detail.

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

### 4.2 NIC configuration in detail

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

### 4.2a MSI-X: how the NIC interrupts the CPU

**What it is.** The NIC must tell the CPU "something happened" (a packet arrived, a send finished, the link went down). It does this with an **interrupt**. MSI-X is the PCIe way: the NIC does not use a wire. It does an ordinary **memory write** to a special address, and the CPU's interrupt controller turns that write into an interrupt. One device can have **many vectors** (a vector is one numbered interrupt, with its own address and data value). This is why a modern NIC can spread work over many CPU cores: **one vector per CPU core's queue**.

| Old way | MSI-X |
| --- | --- |
| Shared wire (INTx), one interrupt for the whole device | Up to 2048 vectors per PCIe function |
| Every core fights over one handler | Each queue gets its own vector, pinned to its own core |

**The key idea: three tables, three owners**

| Piece | Owner | Job |
| --- | --- | --- |
| MSI-X table in the NIC (address + data per vector) | Kernel PCI/IRQ core programs it | Tells the NIC where to write for each vector |
| IRQ (Linux interrupt number, `virq`) | Kernel | What shows in `/proc/interrupts` |
| EQ (event queue, a DMA ring in host memory) | mlx5_core | The NIC writes the event details here. The MSI-X write only says "look at your EQ" |

So an interrupt is two steps: (1) the NIC writes an **event entry (EQE)** into the EQ in host memory by DMA, (2) it then sends the **MSI-X write** to wake a CPU. The handler reads the EQ to find out what happened. This is why the interrupt itself carries no data.

**Where it happens in the code** (all in `drivers/net/ethernet/mellanox/mlx5/core/`):

| Step | When | Function (file) | What it does |
| --- | --- | --- | --- |
| 1 | `mlx5_load` in probe | `mlx5_irq_table_create` (`pci_irq.c`) | Decides how many vectors: `num_ports × online CPUs + 1`, capped by the device's EQ limit and by `pci_msix_vec_count()` (the size of the NIC's MSI-X table, read from PCI config space) |
| 2 | same | `pci_alloc_irq_vectors(..., PCI_IRQ_MSIX)` | Asks the kernel PCI core to enable MSI-X. If the platform supports *dynamic* MSI-X (`pci_msix_can_alloc_dyn`), only vector 0 is allocated now; the rest are added later, on demand |
| 3 | same | `irq_pools_init` (`pci_irq.c`) | Splits the vectors into **pools**: one for the PF's queues, one for SFs (sub-functions) |
| 4 | `mlx5_eq_table_create` (`eq.c`) | `mlx5_irq_alloc` (`pci_irq.c`) | For each vector: gets a `virq` (`pci_irq_vector` or `pci_msix_alloc_irq_at`), then calls **`request_irq(virq, irq_int_handler, ...)`** and sets CPU affinity (`irq_set_affinity_and_hint`) |
| 5 | same | `create_map_eq` (`eq.c`) | Sends firmware **`CREATE_EQ 0x301`** with `eqc.intr = vecidx`. This links the EQ to the vector. Also gives the EQ's DMA pages (`pas[]`) |
| 6 | runtime | `irq_int_handler` (`pci_irq.c`) | Tiny: just calls `atomic_notifier_call_chain` for that vector |
| 7 | runtime | `mlx5_eq_comp_int` (completion) or `mlx5_eq_async_int` (async), in `eq.c` | Reads EQEs from the EQ while the ownership bit says "new". For a completion, it finds the CQ by number and calls `cq->comp()` |
| 8 | runtime | `mlx5e_completion_event` (`en_txrx.c`) | Ethernet: calls `napi_schedule()`. The interrupt hands over to NAPI polling, so heavy traffic doesn't cause one interrupt per packet |

**Two kinds of EQ, many vectors:**

- **Async EQ** (vector 0 / control IRQ): link events, firmware requests for pages, command completions, errors, fatal health events. One per device.
- **Completion EQs**: one per CPU (up to the limit). Each serves many CQs. For Ethernet each channel's CQ is tied to one comp EQ. For RDMA, `mlx5_ib` creates CQs with `ibv_create_cq` and picks a completion vector (`comp_vector`); NCCL and other apps usually poll and don't wait for the interrupt.

**Affinity and why a manager cares:** vectors are pinned to cores (`irq_affinity.c`, `mlx5_irq_affinity_request`) so a queue's work stays on a core close to the NIC (same NUMA node). Wrong affinity (for example, `irqbalance` moving interrupts to the wrong socket) is a classic cause of "throughput is lower than expected".

**Related settings:**

- `ethtool -L eth0 combined N`: number of channels (queues, each using one comp EQ and one vector).
- `ethtool -C eth0 ...`: interrupt moderation (wait a little before interrupting, to batch). Also adaptive moderation, DIM (`en_dim.c`).
- `cat /proc/interrupts | grep mlx5`: one line per vector, with a counter per CPU. Names look like `mlx5_async0@pci:0000:03:00.0` (the control vector) and `mlx5_comp3@pci:0000:03:00.0`. If the device only has one vector, it is called `mlx5_combined0`.
- `cat /proc/irq/<N>/smp_affinity_list`: where a vector is pinned.
- `lspci -vv -s 03:00.0 | grep -i msi-x`: how many vectors the NIC offers and whether MSI-X is enabled.

**What goes wrong (what you will hear):**

| Report | Likely cause |
| --- | --- |
| `Failed to alloc IRQs` / probe fails | Not enough vectors (VM with few, or BIOS/platform limit), `pci=nomsi` on the kernel command line |
| One core at 100% in softirq, others idle | Affinity: all queues on one core, or too few channels |
| High latency, low CPU use | Interrupt moderation too aggressive |
| Counters in `/proc/interrupts` not rising | Wrong IRQ mapping, interrupt remapping or hypervisor problem, or the app is polling (normal for DPDK and RDMA polling) |
| A VF has fewer vectors than the PF | The PF assigns MSI-X per VF (SR-IOV), check the VF's limits |

**DPDK and RDMA polling.** DPDK poll-mode drivers normally turn interrupts **off** and poll the CQ in a loop. That is why they are fast and why they use 100% of a core. The MSI-X path is mostly used for link events and optional Rx-interrupt mode. RDMA apps can arm a CQ and wait on a completion channel (`ibv_req_notify_cq`) when they prefer sleeping to polling.

**Exercise:**
1. On a machine with a ConnectX card, run `grep mlx5 /proc/interrupts`. Count the vectors and match them to `ethtool -l`.
2. In the source, follow one completion: `mlx5_eq_comp_int` → `cq->comp` → `mlx5e_completion_event` → `napi_schedule`.
3. Find the line where `CREATE_EQ` sets `intr` to the vector index.

### 4.2b From an RDMA queue to the CPU: the full interrupt path (ARM GIC and x86)

This section follows **one completion** from the moment the NIC finishes a send to the moment a CPU runs code. It adds the part 4.2a left out: the queues, and the interrupt controller.

**The queue objects (all are rings in host memory, filled by DMA)**

| Object | Created by | Direction | Job |
| --- | --- | --- | --- |
| SQ (send queue) | `ibv_create_qp` | App writes WQEs, NIC reads | Work to send |
| RQ (receive queue) | `ibv_create_qp` | App writes WQEs, NIC reads | Buffers to receive into |
| CQ (completion queue) | `ibv_create_cq(ctx, n, ctx, channel, **comp_vector**)` | NIC writes CQEs, app reads | "This WQE is done" (good or error) |
| EQ (event queue) | mlx5_core, at probe (`mlx5_eq_table_create`) | NIC writes EQEs, driver reads | "CQ number X has a new CQE", and async events |
| MSI-X vector | PCI core + mlx5_core | NIC writes one word to an address | "Wake a CPU" |

A QP has an SQ and an RQ. Each QP is tied to a CQ (or two). **Each CQ is tied to one EQ, and each EQ is tied to one MSI-X vector.** The chain is: **CQ → EQ → vector → IRQ → CPU**.

**Where each link is made in the code**

| Link | Where | How |
| --- | --- | --- |
| CQ → EQ | `mlx5_ib_create_cq` (`drivers/infiniband/hw/mlx5/cq.c`) | Reads `attr->comp_vector`, calls `mlx5_comp_eqn_get(mdev, vector, &eqn)`, then `MLX5_SET(cqc, cqc, c_eqn_or_apu_element, eqn)` inside firmware `CREATE_CQ 0x400`. Ethernet does the same in `en_main.c` (`mlx5e_create_cq`) |
| EQ → vector | `create_map_eq` (`eq.c`) | `MLX5_SET(eqc, eqc, intr, vecidx)` inside `CREATE_EQ 0x301` |
| vector → IRQ number | `mlx5_irq_alloc` (`pci_irq.c`) | `pci_irq_vector()` or `pci_msix_alloc_irq_at()` gives a Linux `virq`; `request_irq()` installs the handler |
| IRQ → CPU | same, plus `irq_affinity.c` | `irq_set_affinity_and_hint()` picks the CPU mask |
| Vector → address+data in the NIC | **kernel PCI/MSI core**, not mlx5 | Writes the vector's *address* and *data* into the NIC's **MSI-X table** (inside a BAR). mlx5 never writes these itself |

`comp_vector` is only a small number (0 to `num_comp_vectors - 1`). The application chooses it. That choice decides **which EQ, vector and CPU** will be woken for that CQ. Good RDMA apps spread CQs over vectors and pin their threads to match.

**What the NIC does when a send finishes (the step by step)**

1. The NIC finishes the WQE on the wire (for RC, after the ACK).
2. It DMA-writes a **CQE** into the CQ ring in host memory.
3. If the CQ is **armed** (the app called `ibv_req_notify_cq`, or the Ethernet driver re-armed it after NAPI), the NIC also DMA-writes an **EQE** (type COMP, carrying the CQ number) into the EQ ring. If the CQ is *not* armed (a polling app like NCCL or DPDK), this step and the next ones **do not happen**. The app just reads the CQE.
4. The NIC does the **MSI-X write**: a PCIe Memory Write TLP to the *address* stored in its MSI-X table entry for this vector, with the *data* value from the same entry. This is an ordinary posted write on PCIe. The NIC does not know about CPUs or the GIC.
5. The interrupt controller receives that write and turns it into an interrupt for one CPU (details below).
6. The CPU runs the kernel interrupt entry and calls `irq_int_handler`, then `mlx5_eq_comp_int`: it reads EQEs, finds the CQ (`mlx5_eq_cq_get`), and calls `cq->comp()`.
7. For Ethernet, `mlx5e_completion_event` calls `napi_schedule()`. For RDMA user space, `mlx5_ib_cq_comp` signals the completion channel, so a thread blocked in `ibv_get_cq_event` wakes up. The app then calls `ibv_poll_cq`.

**What the interrupt controller does with the write**

The address in step 4 is what makes the difference between platforms:

| | ARM64 with GICv3 + ITS (BlueField, Grace, most Arm servers) | x86 (Intel / AMD) |
| --- | --- | --- |
| Address the NIC writes to | The ITS register **`GITS_TRANSLATER`** (a memory-mapped register of the ITS block). The kernel sets it in `its_irq_compose_msi_msg` (`drivers/irqchip/irq-gic-v3-its.c`: address = `its->phys_base + GITS_TRANSLATER`) | The local APIC message address window, `0xFEExxxxx`. The destination CPU's **APIC ID** is encoded in the address bits |
| Data the NIC writes | An **EventID** (`its_get_event_id`) | An interrupt **vector number** |
| Who identifies the sender | The ITS uses the PCIe **DeviceID** (the NIC's bus/device/function, also called the requester ID or "stream ID") that arrives with the write | With an IOMMU interrupt remapping unit, the remapping table checks the sender and translates the message |
| Translation | ITS looks up (DeviceID, EventID) in its **Device Table** and **Interrupt Translation Table (ITT)**, and finds an **LPI** number (a message-based interrupt) and a target **collection** (which maps to a CPU / redistributor) | Message gives CPU + vector directly (or via remapping table) |
| Delivery | The ITS sends the LPI to the target CPU's **Redistributor**, which signals the CPU interface. The CPU takes an IRQ exception; the kernel reads the interrupt ID (`ICC_IAR1_EL1`), maps it to the Linux `virq`, calls the handler, and finishes with an end-of-interrupt write | The local APIC raises the vector on that CPU; the IDT entry runs; the kernel maps vector → `virq` |
| Moving to another CPU (`smp_affinity`) | The kernel sends an ITS command (`MOVI` / `MOVC`) to retarget that (DeviceID, EventID) to another collection. **The NIC is not touched** | The kernel rewrites the MSI-X table entry (new address/data) in the NIC |

The important points:

- **The NIC only ever does a memory write.** It has no idea whether the CPU is x86 or ARM.
- On ARM the NIC's MSI-X table keeps the same address (`GITS_TRANSLATER`) and a different *EventID* per vector. The ITS tables, programmed by the kernel, decide the CPU.
- **The NIC's PCIe function and the ITS must agree on the DeviceID.** If a BlueField/ARM system, a VM, or an IOMMU setup gives a wrong DeviceID mapping, the interrupts never arrive. That looks like "traffic works, but completions never fire".
- In a VM or with SR-IOV VFs, the hypervisor (and on ARM, the GICv4 direct injection feature) may sit in between. VFs have their own MSI-X table and their own DeviceID.

**BlueField note.** A BlueField DPU has two worlds: the **host** (x86 or ARM) sees the DPU as a PCIe NIC with its own MSI-X table; the **DPU's own Arm cores** see the on-chip ConnectX as a device on their internal PCIe bus, so the same path runs there through the DPU's own GIC/ITS. Both run `mlx5_core`, each with its own EQs and vectors.

**Where to look on a live machine**

| Question | Command |
| --- | --- |
| Which vectors does the NIC have? | `lspci -vv -s <BDF> \| grep -i msi-x` |
| Which Linux IRQ is which queue? | `grep mlx5 /proc/interrupts` |
| Where is it pinned? | `cat /proc/irq/<N>/smp_affinity_list`, `cat /proc/irq/<N>/effective_affinity_list` |
| Which controller handles it? | `cat /proc/interrupts` (the column before the name shows `ITS-MSI` / `GICv3` on ARM, `PCI-MSIX` / `IR-PCI-MSIX` on x86). `ls /sys/kernel/irq/<N>/` and `cat /sys/kernel/irq/<N>/chip_name` also show it |
| Is the NIC under an IOMMU? | `ls /sys/kernel/iommu_groups`, `dmesg \| grep -i -E 'iommu\|ITS\|GIC'` |
| Which CQ uses which vector? | `rdma res show cq` (shows the CQ and its poll context); the verbs app chose `comp_vector` |
| Are interrupts really firing? | Watch the counters: `watch -n1 'grep mlx5_comp /proc/interrupts'` while running a test |

**Failure patterns (what you will hear)**

| Report | Meaning |
| --- | --- |
| Completions never wake a sleeping app, but polling works | Interrupt path broken: MSI-X disabled, DeviceID/ITS or IOMMU issue, or the CQ was never armed |
| All `mlx5_comp` interrupts land on CPU0 | `smp_affinity` wrong or `irqbalance` fighting the driver. Check `effective_affinity_list` |
| Throughput drops on ARM, not on x86 | Often IRQ affinity vs NUMA, or ITS collection mapping, or cache-coherent DMA settings. Compare both |
| `Failed to alloc IRQs` at probe | Not enough MSI-X vectors / LPIs available to this DeviceID |
| Latency spikes with RDMA completion channels | Interrupt moderation or CPU idle states (waking a deep-sleep core is slow) |

**Exercise**

1. On any Linux machine, run `grep -E 'mlx5|PCI-MSI' /proc/interrupts | head`. Say which interrupt chip name appears and what it tells you about the platform.
2. Read `its_irq_compose_msi_msg` (`git show HEAD:drivers/irqchip/irq-gic-v3-its.c` in your clone, since it is not in the sparse checkout). Answer: what two values does it fill in?
3. In `drivers/infiniband/hw/mlx5/cq.c`, find where `comp_vector` becomes an `eqn`.
4. Draw the chain CQ → EQ → vector → ITS/APIC → CPU from memory, with the memory write and the DMA writes marked in different colours.

### 4.3 Send and receive in detail

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

**RDMA:**

| App | rdma-core (`libmlx5`) | Kernel (mlx5_ib) | Firmware |
| --- | --- | --- | --- |
| `ibv_open_device`, `ibv_alloc_pd` | `mlx5_alloc_pd`, then `ioctl` | `mlx5_ib_alloc_pd` | `ALLOC_PD 0x800` |
| `ibv_reg_mr` | `ioctl` | Pins memory, creates an MKEY | `CREATE_MKEY 0x200` |
| `ibv_create_cq`, `ibv_create_qp`, `ibv_modify_qp` | `mlx5_create_qp` and others, then `ioctl` | `mlx5_ib_create_qp` and others | `CREATE_CQ 0x400`, `CREATE_QP 0x500`, `MODIFY_QP` (RST, INIT, RTR, RTS) |
| `ibv_post_send`, `ibv_poll_cq` | Writes a WQE and rings the doorbell from user space | none | none (fast path, like DPDK) |

### 4.4 Deconfiguration (teardown) in detail

| Layer | Kernel netdev path | DPDK / DOCA path |
| --- | --- | --- |
| App | `ip link set eth0 down`, `rmmod mlx5_core` | `rte_flow_destroy()`, `rte_eth_dev_stop()`, `rte_eth_dev_close()` |
| User-space driver | none | `mlx5_flow_destroy`, `mlx5_dev_stop`, `mlx5_dev_close`, then `mlx5dv_devx_obj_destroy` and `ioctl()` |
| Kernel | `.ndo_stop = mlx5e_close`, `mlx5e_close_channels`, `mlx5e_destroy_sq`, `mlx5_core_destroy_sq`, `mlx5_core_destroy_cq` | mlx5_ib `devx.c` sends a destroy command it built at create time (`devx_obj_build_destroy_cmd`) |
| Driver unload | `mlx5_unload`, `mlx5_function_teardown`, `mlx5_cmd_teardown_hca` | none |
| Firmware | `DESTROY_SQ 0x906`, `DESTROY_RQ 0x90a`, `DESTROY_CQ 0x401`, `DESTROY_EQ 0x302`, then `TEARDOWN_HCA 0x103`, `MANAGE_PAGES` (reclaim), `DISABLE_HCA 0x105` | Same destroy opcodes |

**Manager point:** if a DPDK app crashes, the kernel still cleans up. Its file handle closes and mlx5_ib runs the saved destroy commands. Leaks here are a classic source of bugs, so ask about tests for it.

### 4.5 Other interfaces

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

**Events (NIC to driver).** The NIC writes to an EQ (created with `CREATE_EQ 0x301`) and mlx5_core handles it: link up/down changes the netdev carrier; a firmware request for memory triggers `MANAGE_PAGES`; a command completion wakes the waiting `mlx5_cmd_do`; errors feed the devlink health reporters.

**Firmware tools (outside the driver).** `mlxconfig`, `mlxfwmanager` and `mstflint` talk to firmware directly through PCI. They set persistent options (for example, SR-IOV and the number of VFs) that apply after a reboot or firmware reset.

---

## Part 5: Hands-on lab and study plan

### 5.1 Get the source for each layer

All commands run in Ubuntu (WSL). Type `wsl` first if your prompt starts with `PS C:\`.

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
cd ~/src/linux && git sparse-checkout set drivers/net/ethernet/mellanox/mlx5 drivers/infiniband/hw/mlx5 drivers/infiniband/core include/linux/mlx5 && cd ~/src
```

`git sparse-checkout set` tells Git which folders you want on disk:

- `drivers/net/ethernet/mellanox/mlx5`: **mlx5_core**, the Ethernet part (netdev, eswitch, SR-IOV, devlink)
- `drivers/infiniband/hw/mlx5`: **mlx5_ib**, the RDMA and InfiniBand part
- `drivers/infiniband/core`: the generic RDMA layer (`ib_uverbs`), used in the trace in 4.1
- `include/linux/mlx5`: shared headers, including `mlx5_ifc.h`, which defines every command the driver can send to the NIC firmware

**3. Clone rdma-core (user-space RDMA).**

```bash
git clone --depth 1 https://github.com/linux-rdma/rdma-core.git
```

The mlx5 part, **libmlx5**, is in `providers/mlx5/`.

**4. Clone DPDK.**

```bash
git clone --depth 1 https://github.com/DPDK/dpdk.git
```

The NVIDIA drivers are in `drivers/net/mlx5/` (the PMD, which handles packets) and `drivers/common/mlx5/` (shared code, including the calls into rdma-core).

**5. Measure how much code your team owns.**

```bash
cd ~/src && for d in linux/drivers/net/ethernet/mellanox/mlx5 linux/drivers/infiniband/hw/mlx5 rdma-core/providers/mlx5 dpdk/drivers/net/mlx5 dpdk/drivers/common/mlx5; do printf "%8d  %s\n" $(find $d -name '*.[ch]' -print0 | xargs -0 cat | wc -l) $d; done
```

This prints the lines of C code in each mlx5 folder: several hundred thousand in total, spread over **three separate communities** (kernel, rdma-core, DPDK), each with its own maintainers, reviews and release schedule.

**6. Follow the trace from 4.1 yourself.**

```bash
cd ~/src && grep -rn "mlx5dv_devx_obj_create" dpdk/drivers/common/mlx5 | head -5
```

```bash
cd ~/src/rdma-core && sed -n '6290,6345p' providers/mlx5/verbs.c
```

```bash
cd ~/src/linux && grep -n "MLX5_CMD_OP_CREATE_SQ\b" include/linux/mlx5/mlx5_ifc.h
```

**7. See where NCCL meets your code.**

```bash
cd ~/src && git clone --depth 1 https://github.com/NVIDIA/nccl.git
```

```bash
cd ~/src/nccl && grep -n "wrap_ibv_post_send\|IBV_WR_RDMA" src/transport/net_ib/p2p.cc
```

```bash
cd ~/src/nccl && grep -n "reg_dmabuf_mr\|ibv_reg_mr" src/transport/net_ib/reg.cc
```

The first grep shows NCCL posting RDMA WRITEs: that is the `ibv_post_send` of 3.3. The second shows NCCL registering GPU memory. To go one layer down, find `mlx5_post_send` in `~/src/rdma-core/providers/mlx5/qp.c`: that is the code that writes the WQE, and its helper `post_send_db` rings the doorbell. The plugin API that NCCL uses toward any network is in `src/include/plugin/nccl_net.h`.

### 5.2 Study plan (about one week, one hour a day)

- **Day 1:** Read Parts 1 and 2. Then draw the NIC (rings, doorbell, DMA, firmware) and the three doors from memory.
- **Day 2:** Read Part 3. For each door, say out loud which API each layer uses and where firmware is involved.
- **Day 3:** Do lab 5.1 and follow the trace in 4.1 in the real source. Finish with step 7: follow one NCCL `ibv_post_send` down into `libmlx5`.
- **Day 4:** DOCA-Host and DOCA. Read the install guide and profiles at [docs.nvidia.com/doca](https://docs.nvidia.com/doca/sdk/index.html). Note which profile (`doca-ofed`, `doca-roce`, `doca-networking`, `doca-all`) your customers use. Remember: the DOCA-Host kernel part (`mlnx-ofa_kernel`) is a backport of the upstream mlx5 you cloned.
- **Day 5:** Book 30 minutes with a senior engineer and ask: (1) walk me through one recent feature across firmware, kernel, user space and DOCA; (2) which of our changes are upstream, and which live only in DOCA-Host; (3) which kernel merge window does our next feature target?

### 5.3 Self-check

You're done when you can answer without notes:

1. Name the three doors into the NIC and one customer type for each.
2. What is a doorbell, and what is a CQE?
3. Which work uses firmware commands, and which doesn't?
4. What is RDMA, and why do AI clusters need it?
5. What are a QP, an MR and an rkey?
6. Why is DPDK faster than kernel sockets?
7. What is the difference between `mlx5_core` and `mlx5_ib`?
8. Why can one feature need patches in four projects?
9. Which door does NCCL use, and which firmware commands does it trigger at setup? Which ones per packet?
10. What is GPUDirect RDMA, and what two ways can the NIC get access to GPU memory?
11. What is MSI-X, and why does the NIC write an EQ entry *and* an interrupt?
12. What is the relation between a vector, an IRQ number, an EQ and a CQ?
13. A customer has one core at 100% softirq. Which three things do you check first?
14. Trace one RDMA completion from the NIC to the CPU: which queues are written by DMA, and which step is the MSI-X write?
15. What does the NIC write on ARM (GICv3 + ITS) versus x86, and who decides which CPU runs the handler?
16. What does `comp_vector` in `ibv_create_cq` choose?
