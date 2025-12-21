# 容器化技术体系

## 文档导读

本文档系统性地介绍容器化技术体系，从底层原理到生产实践，涵盖以下核心内容：

| 章节 | 内容 | 适合读者 |
|------|------|----------|
| **1. VMWare 和容器化区别** | 虚拟化 vs 容器化底层原理对比 | 所有人 |
| **2. Kubernetes 架构与核心概念** | Pod 生命周期、流量链路、调度器、控制器 | 所有人 |
| **3. 容器运行时架构** | containerd、CRI-O、runc 架构详解 | 开发/运维 |
| **4. Kubernetes 核心组件** | etcd、API Server、kubelet、kube-proxy | 开发/运维 |
| **5. 容器网络架构** | CNI、Calico、Cilium、AWS VPC CNI | 运维/架构师 |
| **6. 容器存储架构** | CSI、StorageClass、PV/PVC | 运维/架构师 |
| **7. Kubernetes 安全架构** | RBAC、PSA、NetworkPolicy、Secret 管理 | 安全/运维 |
| **8. Service Mesh 架构** | Istio、流量管理、mTLS | 架构师 |
| **9. 可观测性架构** | Prometheus、日志、追踪 | 运维/SRE |
| **10. GitOps 与 CI/CD** | ArgoCD、Flux、渐进式交付 | DevOps |
| **11. Kubernetes 工作负载** | Pod、Deployment、StatefulSet | 开发/运维 |
| **12. Kubernetes 调度约束** | Affinity、Taints、topologySpread | 运维/架构师 |
| **13. Kubernetes 弹性伸缩** | HPA、VPA、Karpenter、KEDA | 运维/架构师 |
| **14. Kubernetes 升级策略** | 滚动更新、蓝绿、灰度、集群升级 | 运维/SRE |
| **15. Kubernetes 生产实践** | 集群规划、故障排查、性能调优、高可用 | 运维/SRE |
| **16. 容器编排系统对比** | K8S vs Docker Swarm vs Nomad vs ECS | 架构师 |

---

## 1. VMWare 和容器化区别

**本章节导读：**

本章节从底层技术原理深入对比操作系统、VMWare 虚拟化和容器化技术，包含：
1. 快速对比 - 适合快速了解
2. Deep Dive 技术原理 - 适合深入理解底层机制

### 1.1 技术底层实现来源

**虚拟化技术的底层实现：**

- **KVM (Kernel-based Virtual Machine)** - Linux 内核虚拟化模块（2007年合并到 Linux 2.6.20）
  - 将 Linux 内核转变为 Hypervisor（Type-1.5 混合型，直接利用 Linux 内核作为 Hypervisor）
  - 依赖 CPU 硬件虚拟化扩展（Intel VT-x / AMD-V）
  - 每个虚拟机是一个 Linux 进程，由 QEMU 管理
  - 提供**硬件级完全隔离**（独立内核、独立操作系统）
  - 应用：AWS EC2、OpenStack、Proxmox、QEMU/KVM
  
  > **注：** KVM 的分类存在争议。传统上 Type-2 Hypervisor 运行在宿主 OS 之上（如 VirtualBox），而 KVM 作为内核模块直接将 Linux 内核转变为 Hypervisor，因此也被称为 Type-1.5 或混合型 Hypervisor。

- **VMware ESXi** - 专有 Hypervisor（Type-1 裸金属虚拟化）
  - 直接运行在物理硬件上，不依赖宿主操作系统
  - 使用硬件辅助虚拟化（Intel VT-x / AMD-V）
  - 提供企业级虚拟化管理和高级特性

**容器化技术的底层实现：**

- **Linux 内核原生技术**（最底层，逐步加入内核）
  - **Namespace（命名空间）** - 提供进程级隔离（PID、NET、MNT、UTS、IPC、User、Cgroup）（2002年起）
  - **Cgroups（控制组）** - 提供资源限制（CPU、Memory、Disk I/O、Network、PIDs）（2007-2008年加入）
  - **Union File System** - 提供分层存储（AUFS、OverlayFS、Btrfs）

- **LXC (Linux Containers)** - 2008年，第一个完整的容器化实现
  - 封装了 Namespace + Cgroups + rootfs
  - 提供用户空间工具和 API
  - **是容器化技术的先驱和基础**

- **Docker** - 2013年，容器化技术的普及者
  - 最初基于 LXC（Docker 0.9 之前）
  - 2014年用 libcontainer 替代 LXC，直接调用 Linux 内核技术
  - 2015年演进为 runC（OCI 标准实现）
  - 提供镜像标准、分发机制、编排能力

**技术演进时间线：**
```
虚拟化：1999 VMware Workstation → 2004 VMware ESXi → 2007 KVM 合并到 Linux 内核
容器化：2007 Cgroups 加入内核 → 2008 LXC 诞生 → 2013 Docker 诞生 → 2014 libcontainer → 2015 runC (OCI)
```

**核心区别：**
- **虚拟化（KVM/VMware）** = 硬件级隔离 + 独立内核 + 完整操作系统
- **容器化（LXC/Docker）** = 内核级隔离 + 共享内核 + 进程隔离

---

### 1.2 快速对比（概览）

- **架构和虚拟化方式**
  - VMware 是一种虚拟化技术，使用 hypervisor（如 VMware ESXi）在物理硬件上创建和管理虚拟机（VM）。每个虚拟机都运行自己的操作系统实例，并且包含所有必要的驱动程序和库。虚拟机是**完全隔离的环境**，**每个 VM 都有自己的操作系统**。底层依赖 **KVM（Linux）或 VMware ESXi 的 Hypervisor + CPU 硬件虚拟化**（Intel VT-x / AMD-V）。
  - 容器是一种**轻量级的虚拟化技术**，依赖于宿主操作系统的内核。容器共享宿主操作系统，但每个容器都有自己的文件系统、库和运行时环境。容器启动速度快，资源消耗低，因为它们不需要完整的操作系统，只需包含运行应用程序所需的依赖项。底层依赖 **Linux 内核的 Namespace + Cgroups**，由 **LXC（2008）奠定基础，Docker（2013）普及应用**。
- **资源利用效率**
  - VMWare 启动时间长（30-60 秒），每个虚拟机都需要分配相对较多的资源，包括 CPU、内存和存储。这导致**较高的资源开销**，因为每个 VM 都需要完整的操作系统（通常 1-2GB 内存起步）。
  - 容器启动极快（<1 秒，毫秒级），容器**共享宿主机的内核**，因此它们在资源利用上更为高效（通常 10-100MB 内存）。多个容器可以在同一宿主机上运行，密度是 VM 的 10-20 倍。
- **安全与隔离**
  - 虚拟机提供更强的安全性和隔离性（**硬件级隔离**），因为每个 VM 都有独立的操作系统和内核。攻击者需要突破 Guest OS 和 Hypervisor 两层才能影响其他 VM，逃逸风险极低。
  - 容器之间的隔离相对较弱（**内核级隔离**），因为它们共享同一个操作系统内核。如果一个容器中的应用程序存在内核漏洞或配置不当，可能会逃逸到宿主机或影响其他容器。需要通过 Seccomp、AppArmor、SELinux 等机制加固。
- **文件系统隔离**
  - VMware hypervisor 管理硬件资源的分配与访问，拥有一个统一的 VMFS 文件系统（该文件系统支持并发访问，文件锁定，保证隔离），虚拟磁盘的读写都需要进过 VMkernel 处理，VMkernel 确保虚拟机不能超越磁盘边界，通过锁机制和权限管理避免磁盘文件被多个虚拟机同时修改，防止数据冲突与损坏。
  - 容器化的文件系统主要依赖 Linux 内核的 Mount Namespace(命名空间技术）和 UnionFS（联合文件系统），利用 rootfs 和 chroot 将容器的根目录挂载到 UnionFS，使容器进程只能访问自己的根文件系统。
- **可移植性**
  - VMWare 虚拟机包含完整的操作系统，体积大（通常 GB 级别），迁移和分发较慢。虚拟机镜像格式（VMDK、OVF）在不同平台间兼容性有限。
  - 容器镜像体积小（通常 MB 级别），分层存储可复用，迁移和分发快。容器镜像格式（OCI 标准）跨平台兼容性好，"一次构建，到处运行"。
- **运维管理**
  - VMWare 需要管理完整的操作系统（补丁、更新、驱动），运维复杂度高。扩缩容慢（需要启动完整 OS）。
  - 容器只需管理应用及其依赖，运维简单。扩缩容快（秒级），适合微服务和 CI/CD。Kubernetes 等编排工具提供自动化管理。
- 适用场景
  - VMWare 适合：多租户 SaaS（强隔离）、传统单体应用、需要不同操作系统（Windows + Linux）、遗留系统迁移、强安全要求场景。
  - 容器适合：微服务架构、云原生应用、CI/CD 流水线、DevOps 敏捷开发、高密度部署、快速扩缩容场景。
- **容器隔离机制（核心）**
  - **Namespace（命名空间）：负责"看到什么"（视图隔离）**
    - PID Namespace：容器内进程只能看到自己的进程树（PID 1 是容器主进程）
    - Network Namespace：容器有独立的网络栈（IP、端口、路由表、防火墙）
    - Mount Namespace：容器有独立的文件系统挂载点（看不到宿主机其他目录）
    - UTS Namespace：容器有独立的主机名和域名
    - IPC Namespace：容器有独立的进程间通信资源（消息队列、共享内存）
    - User Namespace：容器内 root 用户映射到宿主机普通用户（安全加固）
  - **Union File System（联合文件系统）：负责"文件系统隔离"（分层存储）**
    - **OverlayFS**（现代 Docker 默认）：将多个目录层叠挂载成一个统一视图
      - Lower Layer（只读层）：基础镜像层（如 Ubuntu、Nginx），多个容器共享
      - Upper Layer（读写层）：容器运行时的修改层，每个容器独立
      - Merged Layer（合并层）：容器内看到的完整文件系统
    - **其他实现**：AUFS、Btrfs、ZFS、Device Mapper
    - **rootfs + chroot**：将容器根目录切换到 Union FS，进程只能访问容器内文件系统
    - **写时复制（Copy-on-Write）**：修改文件时才从只读层复制到读写层，节省空间
    - **镜像分层复用**：多个容器共享相同的基础镜像层，只有差异部分独立存储
  - **Cgroups（控制组）：负责"能用多少"（资源限制）**
    - CPU：限制 CPU 使用率和核心数（如最多使用 2 核，50% CPU）
    - Memory：限制内存使用上限（如最多 512MB，超出则 OOM Kill）
    - Disk I/O：限制磁盘读写速度（如读 10MB/s，写 5MB/s）
    - Network：限制网络带宽（如最多 100Mbps）
    - PIDs：限制进程数量（如最多 100 个进程，防止 fork 炸弹）
  - **Docker 资源限制示例**
    ```bash
    # 限制 CPU 和内存
    docker run -d \
      --cpus="1.5" \              # 最多使用 1.5 个 CPU 核心
      --memory="512m" \           # 最多使用 512MB 内存
      --memory-swap="1g" \        # 内存 + Swap 总共 1GB
      nginx
    
    # 限制磁盘 I/O
    docker run -d \
      --device-read-bps /dev/sda:10mb \   # 读速度限制 10MB/s
      --device-write-bps /dev/sda:5mb \   # 写速度限制 5MB/s
      nginx
    
    # 限制进程数
    docker run -d \
      --pids-limit 100 \          # 最多 100 个进程
      nginx
    
    # 查看容器资源使用情况
    docker stats <container_id>
    ```

#### 1.2.1 Linux Namespace 8 种类型详解

| Namespace | 隔离内容 | 系统调用标志 | 内核版本 | 容器使用 | 说明 |
|-----------|----------|--------------|----------|----------|------|
| **Mount (mnt)** | 文件系统挂载点 | CLONE_NEWNS | 2.4.19 | ✅ 必须 | 最早的 Namespace |
| **UTS** | 主机名和域名 | CLONE_NEWUTS | 2.6.19 | ✅ 必须 | Unix Time-sharing System |
| **IPC** | 进程间通信 | CLONE_NEWIPC | 2.6.19 | ✅ 必须 | 消息队列、信号量、共享内存 |
| **PID** | 进程 ID | CLONE_NEWPID | 2.6.24 | ✅ 必须 | 容器内 PID 从 1 开始 |
| **Network** | 网络设备、IP、端口 | CLONE_NEWNET | 2.6.29 | ✅ 必须 | 独立网络栈 |
| **User** | 用户和组 ID | CLONE_NEWUSER | 3.8 | ⚠️ 可选 | 容器内 root 映射到宿主机普通用户 |
| **Cgroup** | Cgroup 根目录 | CLONE_NEWCGROUP | 4.6 | ⚠️ 可选 | 隔离 cgroup 视图 |
| **Time** | 系统时间 | CLONE_NEWTIME | 5.6 | 🔴 较少 | 容器内可设置不同时间 |

**Namespace 操作命令：**

```bash
# 查看进程的 Namespace
ls -la /proc/$$/ns/

# 创建新的 Network Namespace
ip netns add test-ns
ip netns exec test-ns ip addr

# 使用 unshare 创建隔离环境
unshare --mount --uts --ipc --net --pid --fork /bin/bash

# 使用 nsenter 进入已有 Namespace
nsenter -t <pid> -n -m -u -i -p /bin/bash
```

#### 1.2.2 Cgroups v1 vs v2 对比

| 维度 | Cgroups v1 | Cgroups v2 |
|------|------------|------------|
| **层级结构** | 多层级（每个控制器独立） | 单一统一层级 |
| **挂载点** | /sys/fs/cgroup/<controller> | /sys/fs/cgroup |
| **进程归属** | 可属于不同层级的不同 cgroup | 只能属于一个 cgroup |
| **资源分配** | 各控制器独立配置 | 统一接口配置 |
| **内存控制** | memory.limit_in_bytes | memory.max |
| **CPU 控制** | cpu.cfs_quota_us | cpu.max |
| **压力监控** | 🔴 不支持 | ✅ PSI（Pressure Stall Information） |
| **K8S 支持** | ✅ 默认 | ✅ 1.25+ 推荐 |
| **Docker 支持** | ✅ 默认 | ✅ 20.10+ |

**Cgroups v2 配置示例：**

```bash
# 检查系统使用的 cgroups 版本
mount | grep cgroup

# Cgroups v2 资源限制
echo "500M" > /sys/fs/cgroup/mygroup/memory.max
echo "100000 100000" > /sys/fs/cgroup/mygroup/cpu.max  # 100ms/100ms = 100% CPU

# 查看 PSI（压力信息）
cat /sys/fs/cgroup/mygroup/cpu.pressure
cat /sys/fs/cgroup/mygroup/memory.pressure
```

---

### 1.3 Deep Dive：技术原理深度对比

#### 1.3.1 虚拟化层级架构对比

```
物理机（Bare Metal）：
┌─────────────────────────────────────┐
│         应用程序（Application）       │
├─────────────────────────────────────┤
│      操作系统（OS Kernel）            │
├─────────────────────────────────────┤
│      物理硬件（CPU/内存/磁盘/网卡）     │
└─────────────────────────────────────┘

VMWare 虚拟化（Type-1 Hypervisor）：
┌─────────────────────────────────────┐
│  VM 1       │  VM 2      │  VM 3    │
│  ┌───────┐  │  ┌───────┐ │  ┌──────┐│
│  │ App   │  │  │ App   │ │  │App   ││
│  ├───────┤  │  ├───────┤ │  ├──────┤│
│  │ Guest │  │  │ Guest │ │  │Guest ││
│  │ OS    │  │  │ OS    │ │  │OS    ││
│  └───────┘  │  └───────┘ │  └──────┘│
├─────────────────────────────────────┤
│    Hypervisor（VMware ESXi）         │
│    • 硬件虚拟化                       │
│    • CPU 虚拟化（VT-x/AMD-V）         │
│    • 内存虚拟化（EPT/NPT）             │
│    • I/O 虚拟化（SR-IOV）             │
├─────────────────────────────────────┤
│      物理硬件（CPU/内存/磁盘/网卡）     │
└─────────────────────────────────────┘

容器化（Container）：
┌─────────────────────────────────────┐
│ 容器1      │ 容器2     │ 容器3        │
│ ┌────────┐│ ┌────────┐│ ┌────────┐  │
│ │ App    ││ │ App    ││ │ App    │  │
│ │ Libs   ││ │ Libs   ││ │ Libs   │  │
│ └────────┘│ └────────┘│ └────────┘  │
├─────────────────────────────────────┤
│   容器运行时（Docker/containerd）     │
│   • Namespace 隔离                  │
│   • Cgroups 资源限制                 │
│   • UnionFS 文件系统                 │
├─────────────────────────────────────┤
│      宿主操作系统（Host OS Kernel）    │
├─────────────────────────────────────┤
│      物理硬件（CPU/内存/磁盘/网卡）     │
└─────────────────────────────────────┘

关键差异：
• VMWare：每个 VM 有完整的 Guest OS（内核 + 用户空间）
• 容器：共享 Host OS 内核，只有独立的用户空间
```

#### 1.3.2 CPU 虚拟化机制

| 技术         | CPU 虚拟化方式                       | 指令执行                                                               | 性能开销  | 隔离级别  |
| ---------- | ------------------------------- | ------------------------------------------------------------------ | ----- | ----- |
| **物理机**    | 直接执行                            | CPU 直接执行                                                           | 0%    | 无隔离   |
| **VMWare** | 硬件辅助虚拟化<br>（Intel VT-x / AMD-V + EPT） | • Guest OS 特权指令陷入 Hypervisor<br>• Hypervisor 模拟执行<br>• 返回 Guest OS | 2-5% | 硬件级隔离 |
| **容器**     | 进程隔离<br>（Linux Namespace）       | • 直接在 Host OS 内核执行<br>• 系统调用直接处理<br>• 无需模拟                         | <1%   | 内核级隔离 |

> **注：** 现代硬件辅助虚拟化（VT-x/AMD-V + EPT/NPT）性能开销已降至 2-5%，早期无硬件辅助时为 5-10%。
**CPU 虚拟化详解：**

```
VMWare CPU 虚拟化（硬件辅助）：

1. 特权指令陷入（Trap）：
   Guest OS 执行特权指令（如修改页表）
   ↓
   CPU 触发 VM Exit（陷入 Hypervisor）
   ↓
   Hypervisor 模拟执行指令
   ↓
   CPU 触发 VM Entry（返回 Guest OS）

2. 性能优化：
   • Intel VT-x：硬件支持 VM Exit/Entry
   • EPT（Extended Page Table）：硬件支持内存虚拟化
   • VPID（Virtual Processor ID）：减少 TLB 刷新

容器 CPU 虚拟化（进程隔离）：

1. 直接执行：
   容器进程执行指令
   ↓
   直接在 Host OS 内核执行（无陷入）
   ↓
   返回容器进程

2. 系统调用：
   容器进程发起系统调用
   ↓
   Host OS 内核处理（检查 Namespace）
   ↓
   返回容器进程

3. 无需硬件虚拟化：
   • 不需要 VT-x/AMD-V
   • 不需要 VM Exit/Entry
   • 性能接近物理机
```

#### 1.3.3 内存虚拟化机制

| 技术         | 内存虚拟化方式                       | 地址转换             | 内存开销                 | 隔离方式       |
| ---------- | ----------------------------- | ---------------- | -------------------- | ---------- |
| **物理机**    | 虚拟地址 → 物理地址                   | MMU（1 次转换）       | 0%                   | 进程隔离       |
| **VMWare** | 虚拟地址 → Guest 物理地址 → Host 物理地址 | MMU + EPT（2 次转换） | 支持内存过量分配<br>（Memory Overcommitment） | 硬件级隔离      |
| **容器**     | 虚拟地址 → 物理地址                   | MMU（1 次转换）       | 按需分配<br>（如 100MB）    | Cgroups 限制 |

> **注：** 现代 VMware 支持内存过量分配（Memory Overcommitment）和透明页共享（TPS），不一定需要为每个 VM 预留固定内存。
**内存虚拟化详解：**

```
VMWare 内存虚拟化（两级地址转换）：

Guest 虚拟地址（GVA）
    ↓ Guest OS 页表
Guest 物理地址（GPA）
    ↓ EPT（Extended Page Table，Hypervisor 管理）
Host 物理地址（HPA）

问题：
• 两次地址转换，性能开销
• 每个 VM 需要预留内存（即使未使用）
• 内存碎片化

优化技术：
• EPT（硬件支持两级转换）
• 内存气球（Balloon）：回收 VM 未使用的内存
• 内存共享（Transparent Page Sharing）：相同页面只存储一份

容器内存虚拟化（单级地址转换）：

容器虚拟地址（VA）
    ↓ Host OS 页表
Host 物理地址（PA）

优势：
• 单次地址转换，性能接近物理机
• 按需分配内存（不需要预留）
• Cgroups 限制内存使用上限

Cgroups 内存限制：
echo "512M" > /sys/fs/cgroup/memory/container1/memory.limit_in_bytes
```

#### 1.3.4 文件系统隔离机制

| 技术 | 文件系统 | 隔离机制 | 存储开销 | 快照能力 |
|------|---------|---------|---------|---------|
| **VMWare** | VMFS（虚拟机文件系统） | • 每个 VM 独立虚拟磁盘（VMDK）<br>• VMkernel 锁机制<br>• 硬件级隔离 | 每 VM 预分配磁盘<br>（如 40GB） | ✅ 支持（快照） |
| **容器** | UnionFS（联合文件系统） | • Mount Namespace<br>• rootfs + chroot<br>• 分层存储（Layer） | 共享基础镜像<br>（如 100MB） | ✅ 支持（镜像层） |
**文件系统详解：**

```
VMWare 文件系统（VMFS）：

物理磁盘
    ↓
VMFS 文件系统（Hypervisor 管理）
    ↓
虚拟磁盘文件（VMDK）
    ├─ VM1.vmdk（40GB，预分配）
    ├─ VM2.vmdk（40GB，预分配）
    └─ VM3.vmdk（40GB，预分配）
    ↓
Guest OS 文件系统（ext4/NTFS）

特点：
• 每个 VM 有独立的虚拟磁盘文件
• 预分配或动态增长
• 文件锁定机制（防止多 VM 同时访问）
• 快照通过 COW（Copy-On-Write）实现

隔离机制：
• VMkernel 检查磁盘边界
• 文件锁（防止并发修改）
• 权限管理（VM 只能访问自己的 VMDK）

容器文件系统（UnionFS）：

物理磁盘
    ↓
Host OS 文件系统（ext4/xfs）
    ↓
容器镜像（分层存储）
    ├─ Layer 1: 基础镜像（Ubuntu，100MB）← 只读，共享
    ├─ Layer 2: 应用依赖（Python，50MB）← 只读，共享
    ├─ Layer 3: 应用代码（10MB）        ← 只读，共享
    └─ Layer 4: 容器可写层（1MB）       ← 可写，独立
    ↓
容器视图（通过 Mount Namespace）

特点：
• 分层存储，只读层共享
• 只有最上层可写（COW）
• 启动时挂载所有层（UnionFS）
• 容器删除后可写层丢失

隔离机制：
• Mount Namespace（文件系统隔离）
• chroot（根目录隔离）
• rootfs（容器根文件系统）

存储效率对比：
VMWare：
  VM1: 40GB（独立）
  VM2: 40GB（独立）
  VM3: 40GB（独立）
  总计: 120GB

容器：
  基础镜像: 100MB（共享）
  应用层: 50MB（共享）
  容器1 可写层: 1MB
  容器2 可写层: 1MB
  容器3 可写层: 1MB
  总计: 153MB（节省 99%）
```

#### 1.3.5 网络虚拟化机制

| 技术 | 网络虚拟化方式 | 网络栈 | 性能开销 | 隔离方式 |
|------|--------------|--------|---------|---------|
| **VMWare** | 虚拟网卡（vNIC） | 每个 VM 完整的网络栈 | 5-10% | 硬件级隔离 |
| **容器** | veth pair + bridge | 共享 Host OS 网络栈 | <1% | Network Namespace |

#### 1.3.6 进程隔离机制

| 技术 | 进程隔离方式 | 进程树 | PID 空间 | 资源限制 |
|------|------------|--------|---------|---------|
| **VMWare** | 硬件虚拟化 | 每个 VM 独立进程树 | 每个 VM 独立 PID 空间 | Hypervisor 分配 |
| **容器** | Namespace 隔离 | 容器内独立进程树 | PID Namespace 隔离 | Cgroups 限制 |

**进程隔离详解：**

```
VMWare 进程隔离：

Host OS 视角：
  PID 1234: vmware-vmx（VM1 进程）
  PID 1235: vmware-vmx（VM2 进程）

VM1 内部视角：
  PID 1: systemd（init 进程）
  PID 100: nginx

特点：
• 每个 VM 是 Host OS 的一个进程
• VM 内部有完整的进程树（从 PID 1 开始）
• 硬件级隔离（Guest OS 无法看到 Host OS 进程）

容器进程隔离：

Host OS 视角：
  PID 1: systemd
  PID 3000: nginx（容器1）
  PID 3001: mysql（容器2）

容器1 内部视角（PID Namespace）：
  PID 1: nginx（实际是 Host OS 的 PID 3000）

特点：
• 容器进程是 Host OS 的普通进程
• PID Namespace 隔离（容器内看到的 PID 1）
• Host OS 可以看到所有容器进程（ps aux）
• 内核级隔离（共享内核）
```

#### 1.3.7 启动流程对比

```
VMWare 启动流程（30-60 秒）：

1. Hypervisor 分配资源（5 秒）
   • 分配 CPU、内存、磁盘
   • 创建虚拟硬件

2. 加载 Guest OS（10-20 秒）
   • BIOS/UEFI 初始化
   • 加载 Bootloader
   • 加载内核

3. Guest OS 启动（10-30 秒）
   • 初始化内核
   • 加载驱动程序
   • 启动系统服务（systemd）

4. 应用启动（5-10 秒）

总计：30-60 秒

容器启动流程（<1 秒）：

1. 创建 Namespace（<10ms）
   • PID/Network/Mount/UTS/IPC Namespace

2. 设置 Cgroups（<10ms）
   • CPU/内存/I/O 限制

3. 挂载文件系统（<100ms）
   • 挂载镜像层（UnionFS）

4. 启动应用进程（<1 秒）
   • fork + exec 启动应用

总计：<1 秒（快 30-60 倍）
```

#### 1.3.8 资源开销对比

```
场景：运行 10 个 Nginx 实例

VMWare 方案：
每个 VM:
• Guest OS: 1GB 内存
• Nginx: 50MB 内存
• 磁盘: 10GB
• 启动时间: 30 秒

总资源：
• 内存: 10.5GB（10 × 1.05GB）
• 磁盘: 100GB（10 × 10GB）
• 启动时间: 5 分钟

容器方案：
共享：
• 基础镜像: 100MB
• Nginx 镜像: 50MB

每个容器:
• 内存: 50MB

总资源：
• 内存: 500MB（10 × 50MB）
• 磁盘: 150MB（基础镜像共享）
• 启动时间: 10 秒

节省：
• 内存: 95% 节省（10.5GB → 500MB）
• 磁盘: 99% 节省（100GB → 150MB）
• 启动时间: 97% 节省（5 分钟 → 10 秒）
```

#### 1.3.9 安全隔离对比

| 维度 | VMWare | 容器 | 说明 |
|------|--------|------|------|
| **隔离级别** | 硬件级隔离 | 内核级隔离 | VMWare 更强 |
| **内核隔离** | ✅ 每个 VM 独立内核 | 🔴 共享 Host OS 内核 | VM 内核漏洞不影响其他 VM |
| **逃逸风险** | 极低（需要 Hypervisor 漏洞） | 中等（可通过安全加固降低） | 现代容器安全技术可显著降低风险 |
| **多租户** | ✅ 适合（强隔离） | ⚠️ 需要额外安全措施 | VM 更适合多租户 |
| **攻击面** | 小（Hypervisor） | 大（共享内核） | 容器攻击面更大 |

> **容器安全加固技术：** gVisor（用户态内核）、Kata Containers（轻量级 VM）、Seccomp（系统调用过滤）、AppArmor/SELinux（强制访问控制）可显著降低容器逃逸风险。

**安全隔离详解：**

```
VMWare 安全边界：
攻击者需要突破：
1. Guest OS 内核
2. Hypervisor
才能影响其他 VM

容器安全边界：
攻击者只需突破：
1. Namespace 隔离
2. Cgroups 限制
就可能影响其他容器或 Host OS

容器安全加固：
• Seccomp（限制系统调用）
• AppArmor/SELinux（强制访问控制）
• User Namespace（用户隔离）
• Capabilities（权限细分）
• 只读根文件系统
```

#### 1.3.10 技术选型决策表

| 场景 | 推荐技术 | 理由 |
|------|---------|------|
| **多租户 SaaS 平台** | VMWare | 强隔离，安全性高 |
| **微服务架构** | 容器 | 轻量级，快速部署 |
| **传统单体应用** | VMWare | 兼容性好，无需改造 |
| **CI/CD 流水线** | 容器 | 快速启动，资源高效 |
| **需要不同 OS** | VMWare | 可以运行 Windows + Linux |
| **高密度部署** | 容器 | 资源利用率高（10-20 倍） |
| **强安全要求** | VMWare | 硬件级隔离 |
| **云原生应用** | 容器 | Kubernetes 生态 |
| **遗留系统** | VMWare | 无需修改应用 |
| **DevOps 敏捷开发** | 容器 | 快速迭代，环境一致性 |

#### 1.3.11 性能对比总结

| 维度         | 物理机  | VMWare   | 容器          | 容器 vs VMWare |
| ---------- | ---- | -------- | ----------- | ------------ |
| **CPU 性能** | 100% | 95-98%   | 98-99%      | 容器快 1-4%     |
| **内存性能**   | 100% | 90-95%   | 95-98%      | 容器快 3-8%    |
| **磁盘 I/O** | 100% | 85-95%   | 90-95%      | 容器快 0-10%   |
| **网络 I/O** | 100% | 90-95%   | 95-98%      | 容器快 3-8%    |
| **启动时间**   | 0 秒  | 30-60 秒  | <1 秒        | 容器快 30-60 倍  |
| **内存开销**   | 0    | 1-2GB/VM | 10-100MB/容器 | 容器省 90-99%   |
| **密度**     | 1    | 5-10/服务器 | 50-100/服务器  | 容器高 10-20 倍  |

> **注：** 现代 VMware 配合 NVMe + SR-IOV 直通场景下，磁盘 I/O 可达 95%+；CPU 性能在 VT-x + EPT 加持下可达 95-98%。

---

## 2. Kubernetes 架构与核心概念

> **注：** 本章节整合了 Kubernetes 架构概述、Pod 生命周期、流量链路、调度器等核心内容。

https://devopscube.com/kubernetes-architecture-explained/

![K8S](https://devopscube.com/content/images/2025/03/02-k8s-architecture-1.gif)

![K8S](https://substackcdn.com/image/fetch/$s_!Oyvm!,w_1456,c_limit,f_webp,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2F4876c777-96bb-4563-bc98-60561dbc4442_1493x1600.png)

![EKS](https://docs.aws.amazon.com/zh_cn/eks/latest/userguide/images/k8sinaction.png)

![K8S-Architecture](https://kubernetes.io/images/docs/kubernetes-cluster-architecture.svg)

**Namespace 和 Service 是 K8S 集群级别的概念**

![Namespace-Service-Pod](https://media.licdn.com/dms/image/v2/C5612AQHh6Pn98FZVhQ/article-cover_image-shrink_720_1280/article-cover_image-shrink_720_1280/0/1590431596346?e=2147483647&v=beta&t=Nyz-fV-x_6tTVMTGRh_o_yUyrG9K2Ge9DmcjjHwUbJ8)

![Node-Overview](https://kubernetes.io/docs/tutorials/kubernetes-basics/public/images/module_03_nodes.svg)

### 2.1 Kind 资源类型与 API 对象

> Kubernetes 资源定义的核心字段：apiVersion、kind、metadata、spec

#### 2.1.1 YAML 资源定义结构

K8S 资源通过 YAML 文件声明，包含四个顶层字段：

```yaml
apiVersion: apps/v1          # API 版本（组/版本）
kind: Deployment             # 资源类型
metadata:                    # 元数据（名称、标签、命名空间）
  name: my-app
spec:                        # 期望状态（用户定义）
  replicas: 3
```

#### 2.1.2 kind 字段详解

`kind` 是 YAML 中用于标识资源类型的顶层字段，如 Pod、Deployment、Service 等。

**kind 的作用：**
- 与 `apiVersion` 结合，确定资源的 GroupVersionKind（GVK）
- API Server 根据 GVK 路由到正确的处理程序
- 控制器根据 kind 监听和处理对应类型的资源

**kind 与 REST API 的映射：**

| kind | REST API 路径 | 说明 |
|------|--------------|------|
| Pod | `/api/v1/namespaces/{ns}/pods` | 核心 API 组 |
| Deployment | `/apis/apps/v1/namespaces/{ns}/deployments` | apps API 组 |
| Service | `/api/v1/namespaces/{ns}/services` | 核心 API 组 |

**kind 机制：**
- **声明式 API 解析**：当用户提交 YAML 文件时，API Server 先读取 `kind` 和 `apiVersion` 来确定资源的组（Group）、版本（Version）、类型（Kind），然后验证 schema 并存储到 etcd
- **类型注册与扩展**：K8S 通过 CRD（Custom Resource Definition）动态注册新的 kind，确保 kind 唯一且符合命名规范，允许扩展系统而不破坏核心 API
- **控制器调谐**：对应 kind 的控制器（如 Deployment Controller）监视该资源的当前状态与期望状态（spec），通过 Reconcile 循环调整实际状态

**kind 原理：**
- **RESTful 基础**：kind 是 K8S REST API 路径的组成部分，例如 `/api/v1/namespaces/default/pods` 中的 `pods` 对应 `kind: Pod`，基于 HTTP 方法（POST/GET/PUT/DELETE）与资源 URI 的映射实现 CRUD 操作
- **版本控制**：kind 与 apiVersion 结合支持多版本共存，例如 Deployment 有 `apps/v1` 和 `extensions/v1beta1` 两个版本，API Server 根据 GVK 路由到正确的处理程序
- **状态一致性**：依赖 Informer 机制，控制器通过 Watch API 监听 kind 资源的变更事件，计算 diff 并执行动作，确保集群向声明状态收敛

#### 2.1.3 spec 字段详解

`spec` 是资源的规约（Specification）字段，描述用户期望的资源状态（Desired State），如 Deployment 的副本数或 Pod 的容器镜像。K8S 控制平面通过持续调谐确保实际状态匹配 spec。

**spec vs status：**

| 字段 | 维护者 | 内容 | 示例 |
|------|--------|------|------|
| **spec** | 用户定义 | 期望状态 | `replicas: 3` |
| **status** | 系统维护 | 实际状态 | `availableReplicas: 2` |

**spec 机制：**
- **声明式配置驱动**：用户通过 `kubectl apply` 或 API 调用提交 YAML，API Server 验证 spec 的 schema 并存储到 etcd，然后相关控制器读取 spec，计算期望与实际的差异（diff），并执行动作
- **状态收敛循环**：控制器使用 Reconcile 机制周期性检查 spec（如 `replicas: 3`），若 status 不匹配（如当前 Pod 数量是 2），则创建新 Pod 或删除多余 Pod，确保系统向 spec 收敛
- **验证与默认值**：API Server 在处理 spec 时应用默认值和验证规则，例如 Deployment 的 `spec.selector` 必须匹配 `template.metadata.labels`，否则拒绝请求

**spec 原理：**
- **期望状态抽象**：spec 基于 K8S 的声明式模型，原理是"用户声明意图，系统自动实现"，spec 定义资源的理想配置（如端口、资源限制），而非命令式指令
- **与 API 版本绑定**：spec 嵌入 apiVersion 和 kind 定义的资源中，支持版本演进，通过 GVK 路由到特定处理器，确保向后兼容
- **控制器协作**：依赖 Informer 和 Watch API，控制器监听 spec 变更事件，结合 etcd 的原子更新实现一致性

**spec 在 K8S 架构中的位置：**
- **YAML 结构位置**：spec 位于资源对象的顶层，与 metadata 和 status 并列
- **架构位置**：处于 API 对象的核心层，桥接用户意图与控制平面组件（Scheduler、Controller Manager），例如 Pod spec 中定义的容器配置影响 kubelet 的实际执行

**声明式 API 工作流程：**
1. 用户提交 YAML（spec）
2. API Server 验证 schema → 存储到 etcd
3. 控制器 Watch 到变更 → 读取 spec
4. 计算 diff（spec vs 实际状态）
5. 执行动作（创建/更新/删除资源）
6. 更新 status 字段

### 2.2 Pod 生命周期

Pod 在其生命周期只会被调度一次。将 Pod 分配到特定节点过程称为绑定，而选择使用哪个节点的过程称为调度。

**容器状态流转：** Waiting → Running → Terminated

**Pod 重启策略（restartPolicy）：**

| 策略 | 说明 | 适用场景 |
|------|------|----------|
| **Always**（默认） | 容器退出后总是重启 | 长期运行的服务（Web、API） |
| **OnFailure** | 仅在容器异常退出（exit code ≠ 0）时重启 | 批处理任务（Job） |
| **Never** | 容器退出后不重启 | 一次性任务 |

![Pod-Lifecycle](https://cdn-images-1.medium.com/max/1500/1*WDJmiyarVfcsDp6X1-lLFQ.png)

### 2.3 Service 与流量模型

#### 2.3.1 南北流量 vs 东西流量

| 流量类型 | 方向 | 负责组件 | 说明 |
|----------|------|----------|------|
| **南北流量** | 集群外部 ↔ 集群内部 | AWS ELB（ALB/NLB） | 用户请求进入集群 |
| **东西流量** | 集群内部 Pod ↔ Pod | kube-proxy | 服务间调用 |

#### 2.3.2 Service 的本质

**Service 是什么？**
- 从网络角度：一个路由表配置，告诉 kube-proxy 如何转发流量
- 从 K8S 角度：一个 API 对象，存储在 etcd 中
- 从开发者角度：一个稳定的服务访问端点（DNS 名称 + ClusterIP）

**K8S Service vs 业务 Service：**

| 概念 | 关注点 | 示例 |
|------|--------|------|
| **K8S Service** | 网络访问（如何访问） | `payment-service`（ClusterIP/NodePort） |
| **业务 Service** | 业务逻辑（运行什么） | `payment-app`（Deployment/StatefulSet） |

**K8S Service 的 4 种类型：**

```
┌─────────────────────────────────────────────────────────────────────────┐
│  K8S Service 类型（type 字段）                                            │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  1. ClusterIP（默认）                                                    │
│     ├── 只在集群内部可访问                                                 │
│     ├── 分配一个虚拟 IP（ClusterIP）                                       │
│     └── 用途：服务间内部调用                                               │
│                                                                         │
│  2. NodePort                                                            │
│     ├── 在 ClusterIP 基础上，额外在每个节点开放一个端口（30000-32767）        │
│     ├── 外部可通过 NodeIP:NodePort 访问                                   │
│     └── 用途：开发测试、简单场景                                            │
│                                                                         │
│  3. LoadBalancer                                                        │
│     ├── 在 NodePort 基础上，额外创建云厂商的负载均衡器                        │
│     ├── 外部通过 LB 的公网 IP 访问                                         │
│     ├── K8S 只定义抽象，云厂商 Controller 负责创建实际 LB                    │
│     └── 用途：生产环境对外暴露服务                                          │
│                                                                         │
│  4. ExternalName                                                        │
│     ├── 不创建 ClusterIP，只返回 CNAME 记录                                │
│     ├── 将 Service 名称映射到外部 DNS 名称                                 │
│     └── 用途：访问集群外部服务（如 RDS、外部 API）                           │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

**类型之间的包含关系：**

```
┌─────────────────────────────────────────────────────────────────────────┐
│                                                                         │
│  LoadBalancer  ⊃  NodePort  ⊃  ClusterIP                                │
│                                                                         │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │  LoadBalancer                                                   │    │
│  │  ┌─────────────────────────────────────────────────────────┐    │    │
│  │  │  NodePort                                               │    │    │
│  │  │  ┌─────────────────────────────────────────────────┐    │    │    │
│  │  │  │  ClusterIP（虚拟 IP + kube-proxy 规则）           │    │    │    │
│  │  │  └─────────────────────────────────────────────────┘    │    │    │
│  │  │  + 每个节点开放端口 30000-32767                            │    │   │
│  │  └─────────────────────────────────────────────────────────┘    │    │
│  │  + 云厂商创建外部 LB（NLB/ALB/GLB）                                │    │
│  └─────────────────────────────────────────────────────────────────┘    │
│                                                                         │
│  ExternalName 是独立的，不包含上述任何类型                                   │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

> 💡 **关键理解**：`LoadBalancer` 是 K8S 定义的抽象概念，K8S 本身不创建负载均衡器。
> 实际创建 LB 的是云厂商的 Controller（如 AWS Load Balancer Controller）。
> 不同云厂商会创建不同的 LB：AWS 创建 NLB/ALB，GCP 创建 GLB，Azure 创建 Azure LB。

#### 2.3.3 对外暴露服务的方式

**服务暴露层级架构：**

```
┌─────────────────────────────────────────────────────────────────┐
│                      对外暴露服务的三个层级                         │
├─────────────────────────────────────────────────────────────────┤
│  第 1 层：K8S 原生方式（必选）                                      │
│  ├── NodePort（直接暴露节点端口）                                  │
│  ├── LoadBalancer（云厂商 LB，L4）                                │
│  └── Ingress（L7 路由，支持路径/域名）                              │
├─────────────────────────────────────────────────────────────────┤
│  第 2 层：API Gateway（可选，对外 API 需要认证计费时）                │
│  └── AWS API Gateway / Kong / Apigee                            │
│      - 功能：认证授权、限流熔断、API 版本管理、请求转换                │
│      - 流量：Client → API GW → K8S (NLB/ALB) → Pod               │
├─────────────────────────────────────────────────────────────────┤
│  第 3 层：Service Mesh（可选，复杂微服务/零信任网络时）               │
│  └── Istio / Linkerd                                            │
│      - 功能：流量管理、可观测性、安全（mTLS）、故障注入                 │
│      - 流量：Client → Istio Gateway → Sidecar (Envoy) → Pod      │
└─────────────────────────────────────────────────────────────────┘
```

**⚠️ 重要说明：** 三层不是必须全部使用。大多数场景只需要第 1 层（K8S 原生方式）即可满足需求。从简单开始，按需增加复杂度。

**第 1 层：K8S 原生方式**

| 方式                                   | 流量路径                               | 使用的组件                        | 模式            |
| ------------------------------------ | ---------------------------------- | ---------------------------- | ------------- |
| NodePort                             | Client → Node:30000-32767 → Pod    | 无需 LB                        | -             |
| LoadBalancer (默认)                    | Client → NLB → Node:NodePort → Pod | Kubernetes Cloud Controller  | Instance Mode |
| **LoadBalancer (AWS LB Controller)** | Client → NLB → Pod IP              | AWS Load Balancer Controller | IP Mode (可选)  |
| Ingress (Instance Mode)              | Client → ALB → Node:NodePort → Pod | AWS Load Balancer Controller | Instance Mode |
| Ingress (IP Mode)                    | Client → ALB → Pod IP              | AWS Load Balancer Controller | IP Mode       |

**LoadBalancer vs Ingress 核心区别：**

| 维度 | LoadBalancer (Service) | Ingress |
|------|------------------------|---------|
| **创建的 AWS 资源** | NLB（L4） | ALB（L7） |
| **LB 与 Service 关系** | 1 个 Service = 1 个 NLB | 多个 Service 共享 1 个 ALB |
| **路由能力** | 无（只能转发到一个 Service） | 有（路径/域名路由到多个 Service） |
| **成本** | 高（每个服务一个 NLB） | 低（多服务共享） |
| **适用场景** | 单服务暴露、TCP/UDP | 多服务暴露、HTTP/HTTPS |

**💡 底层原理：为什么 LoadBalancer 用 NLB，Ingress 用 ALB？**

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        OSI 网络层级与 LB 类型对应                          │
├─────────────────────────────────────────────────────────────────────────┤
│  L7 应用层  │  HTTP/HTTPS  │  能看到: URL路径、域名、Header、Cookie         │
│            │              │  → 需要 ALB（Application Load Balancer）     │
├─────────────────────────────────────────────────────────────────────────┤
│  L4 传输层  │  TCP/UDP     │  只能看到: IP地址、端口号                      │
│            │              │  → 需要 NLB（Network Load Balancer）         │
└─────────────────────────────────────────────────────────────────────────┘
```

- **Ingress 设计目标**：处理 HTTP/HTTPS 流量，需要理解 URL 路径、域名、Header 等 L7 信息来做路由决策，所以只能用 **ALB**
- **LoadBalancer Service 设计目标**：处理 TCP/UDP 流量，只需要知道 IP 和端口进行转发，所以用 **NLB**

> ⚠️ **注意**：LoadBalancer Service 默认会创建 CLB（Classic Load Balancer），AWS 推荐使用 NLB，需要添加注解：
> `service.beta.kubernetes.io/aws-load-balancer-type: "nlb"`

**🔧 Ingress 与 AWS Load Balancer Controller 的关系**

```
┌─────────────────────────────────────────────────────────────────────────┐
│  Ingress 工作原理                                                        │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  1. 安装 AWS Load Balancer Controller（必须先安装）                        │
│     helm install aws-load-balancer-controller ...                       │
│                                                                         │
│  2. 创建 Ingress 资源（只是配置声明，不会自动创建 ALB）                       │
│     kind: Ingress                                                       │
│     spec:                                                               │
│       ingressClassName: alb  ← 告诉 K8S 由哪个 Controller 处理            │
│                                                                         │
│  3. AWS LB Controller 监听到 Ingress 资源，自动创建 ALB                    │
│                                                                         │
│  ┌──────────────┐    监听      ┌──────────────┐    创建     ┌─────┐      │ 
│  │   Ingress    │  ───────→   │  AWS LB      │  ──────→    │ ALB │      │
│  │  (K8S 对象)   │             │  Controller  │            │     │       │
│  └──────────────┘             └──────────────┘            └─────┘       │
│       配置声明                      实现者                   实际资源       │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

**关键点：**
- `Ingress` 本身只是 K8S API 对象（配置声明），不会自动创建任何 AWS 资源
- 必须先安装 `AWS Load Balancer Controller`，它会监听 Ingress 资源并创建对应的 ALB
- `ingressClassName: alb` 告诉 K8S 这个 Ingress 由 AWS LB Controller 处理
- 同一个 Controller 也处理 `type: LoadBalancer` 的 Service，但会创建 NLB 而不是 ALB

**图示对比：**

```
┌─────────────────────────────────────────────────────────────────┐
│  LoadBalancer Service（每个 Service 一个 NLB）                    │
│                                                                 │
│  NLB-1 ──→ user-service ──→ user Pods                           │
│  NLB-2 ──→ order-service ──→ order Pods                         │
│  NLB-3 ──→ payment-service ──→ payment Pods                     │
│                                                                 │
│  3 个服务 = 3 个 NLB = 3 倍成本                                   │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│  Ingress（多个 Service 共享一个 ALB）                              │
│                                                                 │
│         ┌──→ /users    ──→ user-service ──→ user Pods           │
│  ALB ───┼──→ /orders   ──→ order-service ──→ order Pods         │
│         └──→ /payments ──→ payment-service ──→ payment Pods     │
│                                                                 │
│  3 个服务 = 1 个 ALB = 节省成本 + L7 路由能力                       │
└─────────────────────────────────────────────────────────────────┘
```

**选择建议：**

| 场景 | 推荐方式 |
|------|----------|
| 单个服务对外暴露（TCP/UDP） | LoadBalancer (NLB) |
| 多个 HTTP 服务对外暴露 | Ingress (ALB) |
| 需要路径/域名路由 | Ingress (ALB) |
| 需要 gRPC、WebSocket | Ingress (ALB) 或 NLB |
| 追求最低延迟（跳过 NodePort） | IP 模式（LoadBalancer 或 Ingress） |

**第 2 层：API Gateway**

| 方式 | 流量路径 | 核心功能 | 适用场景 |
|------|----------|----------|----------|
| AWS API Gateway | Client → API GW → NLB/ALB → Pod | 认证（JWT/OAuth）、限流、API Key、请求转换 | 对外 API、多租户 SaaS、合作伙伴 API |

**API Gateway vs LoadBalancer/Ingress 对比：**

| 维度           | LoadBalancer/Ingress | API Gateway         |
| ------------ | -------------------- | ------------------- |
| **层级**       | L4/L7 负载均衡           | L7 + 应用层            |
| **核心职责**     | 流量分发                 | API 管理 + 流量治理       |
| **认证授权**     | 🔴 不支持               | ✅ OAuth、JWT、API Key |
| **限流熔断**     | 🔴 不支持               | ✅ Rate Limiting     |
| **请求转换**     | 🔴 不支持               | ✅ 请求重写、响应映射         |
| **API 版本管理** | 🔴 不支持               | ✅ v1/v2 路由          |
| **计费模式**     | 按 LCU/流量             | 按请求数                |

**第 3 层：Service Mesh**

| 方式 | 流量路径 | 核心功能 | 适用场景 |
|------|----------|----------|----------|
| Istio Gateway | Client → Istio GW → Sidecar → Pod | 流量管理、mTLS、可观测性、故障注入 | 微服务架构、零信任网络 |

**Service Mesh 与其他方式的区别：**
- **Sidecar 模式**：每个 Pod 注入 Envoy 代理，所有流量经过 Sidecar
- **不仅是入口**：Service Mesh 管理的是服务间（东西向）+ 入口（南北向）的全部流量
- **详见第 8 章 Service Mesh 架构**

#### 2.3.4 网关类型与选型

**网关类型定位：**

```
┌─────────────────────────────────────────────────────────────────┐
│                       网关类型与定位                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌─────────────────┐                                            │
│  │  云厂商 API 网关  │  AWS API Gateway / Kong / Apigee           │
│  │  (外部流量入口)   │  - 面向外部客户端（Web/Mobile/第三方）         │
│  └────────┬────────┘  - 认证计费、限流、API 版本管理                │
│           │                                                     │
│           ▼                                                     │
│  ┌─────────────────┐                                            │
│  │  K8S 流量入口    │  NLB / ALB / Ingress                       │
│  │  (负载均衡)      │  - 流量分发到 K8S 集群                        │
│  └────────┬────────┘  - L4/L7 路由                               │
│           │                                                     │
│           ▼                                                     │
│  ┌─────────────────┐                                            │
│  │  业务网关        │  Spring Cloud Gateway / Zuul               │
│  │  (应用层网关)    │  - 面向内部微服务                             │
│  └────────┬────────┘  - 业务路由、过滤器、熔断、灰度                 │
│           │                                                     │
│           ▼                                                     │
│  ┌─────────────────┐                                            │
│  │  服务网格网关     │  Istio Gateway                             │
│  │  (基础设施网关)   │  - 服务间通信治理                            │
│  └─────────────────┘  - mTLS、流量管理、可观测性                   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**网关类型对比：**

| 网关类型 | 代表产品 | 部署位置 | 核心职责 | 面向对象 |
|----------|----------|----------|----------|----------|
| **云厂商 API 网关** | AWS API Gateway | K8S 外部（云服务） | API 管理、认证计费 | 外部客户端 |
| **K8S 入口** | ALB/NLB/Ingress | K8S 边界 | 流量分发 | 进入集群的流量 |
| **业务网关** | Spring Cloud Gateway | K8S 内部（Pod） | 业务路由、过滤器 | 内部微服务 |
| **服务网格** | Istio Gateway | K8S 内部（Sidecar） | 服务间治理 | 所有服务间流量 |

**常见架构模式：**

| 模式 | 流量路径 | 适用场景 | Path 配置位置 |
|------|----------|----------|---------------|
| **模式 1：传统 Spring Cloud** | ALB → Spring Cloud Gateway → Pods | 已有 Spring Cloud 技术栈 | Gateway application.yml |
| **模式 2：云原生 Istio** | ALB → Istio Gateway → Sidecar → Pods | 多语言微服务、零信任网络 | Istio VirtualService |
| **模式 3：混合架构** | ALB → Istio GW → Spring Cloud GW → Pods | 渐进式迁移 | 两处都有（需明确职责） |
| **模式 4：对外 API** | AWS API GW → NLB → Pods | 对外 API 需独立管理 | API Gateway 控制台 |

**网关选型决策：**

| 问题 | 选择建议 |
|------|----------|
| 团队只熟悉 Java？ | Spring Cloud Gateway |
| 多语言微服务（Java + Go + Python）？ | Istio |
| 需要对外 API 计费/配额管理？ | AWS API Gateway |
| 需要 mTLS 零信任网络？ | Istio |
| 已有 Spring Cloud 全家桶？ | 保留 Spring Cloud Gateway |
| 追求简单，服务数量少？ | 只用 ALB + Ingress |

**⚠️ 避免过度设计：** 不建议同时使用 AWS API Gateway + Spring Cloud Gateway + Istio Gateway，会造成流量路径过长、配置分散、功能重叠。

#### 2.3.5 API Path 管理

**核心问题：** 当有多层网关时，同一个请求路径可能在多个地方配置，容易混乱。

```
Client 请求: GET /api/v1/users/123

问题：这个路径在哪里配置？谁负责路由？
- ALB Ingress 规则？
- AWS API Gateway？
- Spring Cloud Gateway？
- Istio VirtualService？
```

**API Path 管理原则：**

| 原则 | 说明 |
|------|------|
| **单一职责** | 业务路由只在一个地方配置（网关或 Istio），其他层只做透传 |
| **透传优先** | 非路由层尽量透传路径，不做路径修改 |
| **路径重写要谨慎** | 如果必须重写，只在一个地方做，并文档化 |
| **版本在路径中** | 推荐 `/api/v1/users` 而不是 Header 版本控制 |

**场景 1：ALB + Spring Cloud Gateway（推荐）**

```
┌─────────────────────────────────────────────────────────────────┐
│  Client: GET https://api.example.com/api/v1/users/123           │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│  ALB (Ingress)                                                  │
│  规则：/* → spring-cloud-gateway Service                         │
│  职责：SSL 终止 + 透传所有路径（不做细粒度路由）                       │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│  Spring Cloud Gateway（业务路由在这里）                            │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │  application.yml:                                         │  │
│  │  - /api/v1/users/**    → lb://user-service                │  │
│  │  - /api/v1/orders/**   → lb://order-service               │  │
│  │  - /api/v1/payments/** → lb://payment-service             │  │
│  └───────────────────────────────────────────────────────────┘  │
│  职责：业务路由 + 过滤器 + 熔断 + 灰度                               │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│  user-service (Pod)                                             │
│  接收路径: /api/v1/users/123（原始路径透传）                        │
│  或者: /users/123（如果网关配置了 StripPrefix=2）                   │
└─────────────────────────────────────────────────────────────────┘
```

**Path 配置位置：Spring Cloud Gateway 的 application.yml**

**场景 2：ALB + Istio（无 Spring Cloud Gateway）**

```
┌─────────────────────────────────────────────────────────────────┐
│  Client: GET https://api.example.com/api/v1/users/123           │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│  ALB (Ingress)                                                  │
│  规则：/* → istio-ingressgateway Service                         │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│  Istio Gateway + VirtualService（业务路由在这里）                  │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │  VirtualService:                                          │  │
│  │  - match: uri.prefix="/api/v1/users"                      │  │
│  │    route: destination.host=user-service                   │  │
│  │  - match: uri.prefix="/api/v1/orders"                     │  │
│  │    route: destination.host=order-service                  │  │
│  └───────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│  user-service (Pod + Sidecar)                                   │
└─────────────────────────────────────────────────────────────────┘
```

**Path 配置位置：Istio VirtualService（K8S CRD）**

**场景 3：AWS API Gateway + NLB（对外 API）**

```
┌─────────────────────────────────────────────────────────────────┐
│  外部 Client: GET https://api.example.com/v1/users/123           │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│  AWS API Gateway（API 路由在这里）                                │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │  资源配置:                                                  │  │
│  │  - GET  /v1/users/{id}  → 集成: VPC Link → NLB             │  │
│  │  - POST /v1/users       → 集成: VPC Link → NLB             │  │
│  │  - GET  /v1/orders/{id} → 集成: VPC Link → NLB             │  │
│  └───────────────────────────────────────────────────────────┘  │
│  职责：认证、限流、API Key、请求转换                                 │
│  可选：路径重写 /v1/users → /api/v1/users                         │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│  NLB → K8S Service → Pod（透传，不做路由）                         │
└─────────────────────────────────────────────────────────────────┘
```

**Path 配置位置：AWS API Gateway 控制台或 OpenAPI Spec**

**推荐架构总结：**

| 场景 | 推荐架构 | Path 配置位置 |
|------|----------|---------------|
| **已有 Spring Cloud 技术栈** | ALB → Spring Cloud Gateway → Pods | Gateway application.yml |
| **多语言微服务** | ALB → Istio Gateway → Pods | Istio VirtualService |
| **对外 API 需独立管理** | AWS API GW → NLB → Pods | API Gateway 控制台 |
| **简单场景** | ALB Ingress → Pods | Ingress 规则 |

**Ingress Target Type 对比：**

| 模式 | Target Group 注册内容 | 流量路径 | 配置方式 |
|------|----------------------|----------|----------|
| **Instance**（默认） | EC2 + NodePort | ALB → EC2:NodePort → kube-proxy → Pod | 默认 |
| **IP** | Pod IP | ALB → Pod IP（跳过 NodePort） | `alb.ingress.kubernetes.io/target-type: ip` |

**Ingress 的本质：**
- Ingress 是一个 API 配置对象（存储在 etcd）
- Ingress Controller（如 ALB Controller、Nginx Controller）监听 Ingress 资源变化
- Ingress Controller 根据配置创建/更新实际的负载均衡器

### 2.4 流量链路详解

> 终端用户请求 ALB -> EKS Pod 完整链路

1. **准备阶段**
   - 创建 Ingress 资源 -> API Server 存储到 etcd -> Ingress Controller 监听 API Server -> Ingress Controller 读取 Ingress 配置 -> 创建 ALB，创建 Target Group，创建监听器，创建路由规则 -> 等待 ALB 就绪 -> 更新 Ingress 状态
   - **Target Group 注册方式（两种模式）：**
     - **Instance 模式（默认）**：注册 EC2 实例 + NodePort（例如：10.0.1.20:30080）
       - Service 会自动分配一个 NodePort（范围：30000-32767）
       - ALB 流量 → EC2 NodePort → iptables/IPVS → Pod
     - **IP 模式**：直接注册 Pod IP（例如：10.0.1.25:8080）
       - 需要在 Ingress 注解中指定：`alb.ingress.kubernetes.io/target-type: ip`
       - ALB 流量 → Pod IP（跳过 NodePort 和 kube-proxy 规则）
   - Ingress Controller 将节点或 Pod IP 注册到 Target Group 以后，节点上的 kube-proxy 会监听 Service 的变化，然后 kube-proxy 会在节点上配置 **iptables/IPVS** 规则
     - **IPVS 是一个专门负责负载均衡的模块，比 iptables 更高效。**
       - **iptables 模式：**
         - 数据包延迟：~0.1-0.2ms
         - 原因：全程内核态
         - 吞吐量： 5-10 Gbps
       - **IPVS 模式：**
         - 数据包延迟：~0.05-0.1ms
         - 原因：内核态 + 优化的数据结构
         - 吞吐量：10-40 Gbps

2. **DNS 解析**
   
   2.1 End-User 作为 Client 访问：example.com/users
   2.2 如果 DNS 没有缓存一步步查：浏览器查询本地 DNS 缓存 -> 递归 DNS 服务器 -> Route53 权威DNS（其他 DNS服务器）
   2.3 最终返回：api.example.com → abc123.us-east-1.elb.amazonaws.com，abc123.us-east-1.elb.amazonaws.com → 54.123.45.67 (ALB IP) 
   2.4 Client 获取 ALB IP
      - 关键点：
         - 域名通过 CNAME 指向 ALB 的 DNS 名称
         - ALB 的 DNS 名称解析为3个 IP（跨 AZ 的 ALB 节点，3个AZ返回3个 IP，一般默认选用第一个 IP）具体策略由客户端自己实现（选择第一个为常见的，随机选择，轮训，基于延迟 RTT）

3. **建立 TCP 连接 ALB**
   
   3.1 与 ALB 三次握手，并确定路由规则 /users -> target-group-users（EKS 的 Worker Node），在 ALB 监听器一侧，进行解密并终止 SSL/TLS。ALB 检查请求头（Host，Path），ALB 根据 Ingress 规则决定路由

4. **ALB 到 EKS 的 Node 连接**
   
   4.1 ALB 内网 IP 与 Node 所在的 EC2 建立 TCP 链接，例如：EC2 作为 Target Group 注册的实例为：**10.0.1.20:30080**
   4.2 流量进入 EC2 通过 ENI，到达该节点的 30080 端口

5. **节点网络栈（Linux 内核网络栈）负责将数据包的目标改为 Pod IP -- 这一部分是绕过了用户态，直接在内核态执行的详细过程**
   
   5.1 数据包到达 EC2 节点后，进入 Linux 内核网络栈
   5.2 数据包进入 PREROUTING 链
   5.3 **匹配 iptables NAT 规则（由 kube-proxy 配置），iptables 规则定义了如何将原目标地址改为最终的 Pod 目标地址**
   5.4 查找规则：**目标端口 30080 -> Service** user-service （user） 
   5.5 匹配 kube-proxy 预先生成的 iptables 规则（规则中已包含后端 Pod 列表：Pod1、Pod2、Pod3）
   5.6 其中 Pod2 在当前 EC2 节点，**Pod2 IP 为：10.0.1.25:8080**
   5.7 负载均衡选择（取决于 externalTrafficPolicy）：
      - **Cluster 模式（默认）**：随机选择任意 Pod（可能跨节点），做 SNAT 将源 IP 改为节点 IP，确保响应能原路返回
      - **Local 模式**：仅选择本地节点的 Pod，不做 SNAT，保留源 IP（ALB IP），但可能负载不均
      - 本例假设使用 **Local 模式**，选择本地 Pod2
   5.8 **DNAT（目标地址转换）：**原目标 IP：10.0.1.20:30080 改为 Pod2 IP 为：10.0.1.25:8080，**此时数据包的目标地址已经改为 Pod2 IP**
   5.9 路由决策： 10.0.1.25 在本地节点转发到 Pod

6. **CNI 网络转发到 Pod**

   6.1 现在数据包状态（Local 模式下）：源 IP 10.0.0.100（ALB）→ 目标 IP 10.0.1.25（Pod2），目标端口 8080
      - 注：如果是 Cluster 模式且 Pod 在其他节点，源 IP 会被 SNAT 为当前节点 IP
   6.2 路由表查询，找到 veth pair（虚拟网络设备对）
      - 数据包从节点网络栈路由到 vethXXX 接口 → 通过 veth pair → 从 Pod 侧 eth0 接口进入 Pod
      - 数据包进入 Pod2 的网络命名空间，到达 Pod2 的 eth0 网络接口

7. **Pod 内部应用处理**
   
   7.1 容器 user-app 接收 HTTP 请求，该请求内有 Client IP（X-Forwarded-For 字段携带）
   7.2 解析 HTTP 请求，路由到处理函数
   7.3 **流量原路返回**
      - Pod 生成响应数据包：源 IP 10.0.1.25:8080 → 目标 IP 10.0.0.100（ALB）
      - 数据包从 Pod eth0 → veth pair → 节点网络栈
      - **conntrack 连接跟踪处理**：
        - 入站时做了 DNAT（目标地址：NodePort → Pod IP），conntrack 记录了这个转换
        - 出站时 conntrack 自动做**反向 DNAT**（源地址：Pod IP → NodePort），使 ALB 能正确接收响应
        - **Local 模式**：入站时未做 SNAT，出站时只需反向 DNAT，响应包直接返回 ALB
        - **Cluster 模式**：入站时做了 SNAT（源地址：ALB IP → 节点 IP），出站时 conntrack 自动做反向 SNAT（目标地址：节点 IP → ALB IP）
      - 数据包通过节点 ENI → VPC 网络 → ALB
      - ALB 将响应返回给客户端（通过之前建立的 TCP 连接）

#### 2.4.1 完整流程图

```
┌──────────────────────────────────────────────────────────┐
│                        终端用户                           │
│              https://api.example.com/users               │
└────────────────────────────┬─────────────────────────────┘
                             │ 1. DNS 解析
                             │    api.example.com
                             │    → abc123.elb.amazonaws.com
                             │    → 54.123.45.67
                             ▼
┌──────────────────────────────────────────────────────────┐
│              AWS Application Load Balancer               │
│  ┌────────────────────────────────────────────────────┐  │
│  │  2. SSL/TLS 终止（解密 HTTPS）                       │  │
│  └────────────────────────────────────────────────────┘  │
│  ┌────────────────────────────────────────────────────┐  │
│  │  3. 路由规则匹配                                     │  │
│  │     Host: api.example.com                          │  │
│  │     Path: /users                                   │  │
│  │     → Target Group: tg-user-service                │  │
│  └────────────────────────────────────────────────────┘  │
│  ┌────────────────────────────────────────────────────┐  │
│  │  4. 负载均衡选择目标                                  │  │
│  │     节点 1: 10.0.1.10:30080                         │  │
│  │     节点 2: 10.0.1.20:30080 ← 选中                   │  │
│  │     节点 3: 10.0.1.30:30080                         │  │
│  └────────────────────────────────────────────────────┘  │
└────────────────────────────┬─────────────────────────────┘
                             │ 5. 转发到节点 NodePort
                             │    目标: 10.0.1.20:30080
                             ▼
┌──────────────────────────────────────────────────────────┐
│                  节点 2 (EC2 实例)                        │
│                  IP: 10.0.1.20                           │
│  ┌────────────────────────────────────────────────────┐  │
│  │  6. 数据包到达 NodePort 30080                        │  │
│  └────────────────────────────────────────────────────┘  │
│                            ↓                             │
│  ┌────────────────────────────────────────────────────┐  │
│  │  7. Linux 内核网络栈处理                             │  │
│  │                                                    │  │
│  │   数据包到达网卡 → 内核网络栈                          │  │
│  │      ↓                                             │  │
│  │   Linux 内核的 netfilter 拦截流量                    │  │
│  │      ↓                                             │  │
│  │  iptables NAT 规则（kube-proxy 配置）:               │  │
│  │      ↓                                             │  │
│  │  PREROUTING                                        │  │
│  │      ↓                                             │  │
│  │  KUBE-SERVICES                                     │  │
│  │      ↓                                             │  │
│  │  KUBE-NODEPORTS (匹配端口 30080)                    │  │
│  │      ↓                                             │  │
│  │  KUBE-SVC-USER-SERVICE                             │  │
│  │      ↓                                             │  │
│  │  负载均衡选择 Pod:                                   │  │
│  │  ├─ Pod-1: 10.0.1.5:8080 (节点 1)                   │  │
│  │  ├─ Pod-2: 10.0.1.25:8080 (节点 2) ← 选中           │  │
│  │  └─ Pod-3: 10.0.1.35:8080 (节点 3)                  │  │
│  │      ↓                                             │  │
│  │  KUBE-SEP-POD2                                     │  │
│  │      ↓                                             │  │
│  │  DNAT: 10.0.1.20:30080 → 10.0.1.25:8080            │  │
│  └────────────────────────────────────────────────────┘  │
│                             ↓                            │
│  ┌────────────────────────────────────────────────────┐  │
│  │  8. 路由到 Pod                                      │  │
│  │     查找路由表: 10.0.1.25 → veth12345678             │  │
│  └────────────────────────────────────────────────────┘  │
│                             ↓                            │
│  ┌────────────────────────────────────────────────────┐  │
│  │  9. veth pair 转发                                  │  │
│  │     节点侧: veth12345678                            │  │
│  │         ↕                                          │  │
│  │     Pod 侧: eth0                                   │  │
│  └────────────────────────────────────────────────────┘  │
│                             ↓                            │
│  ┌────────────────────────────────────────────────────┐  │
│  │  Pod-2 (网络命名空间)                                │  │
│  │  IP: 10.0.1.25                                     │  │
│  │  监听: 0.0.0.0:8080                                 │  │
│  │  ┌──────────────────────────────────────────────┐  │  │
│  │  │  10. 应用容器接收请求                           │  │  │
│  │  │      GET /users HTTP/1.1                     │  │  │
│  │  │      Host: api.example.com                   │  │  │
│  │  └──────────────────────────────────────────────┘  │  │
│  │                     ↓                              │  │
│  │  ┌──────────────────────────────────────────────┐  │  │
│  │  │  11. 应用处理业务逻辑                           │  │  │
│  │  │      - 查询数据库                              │  │  │
│  │  │      - 生成响应                               │  │  │
│  │  └──────────────────────────────────────────────┘  │  │
│  │                     ↓                              │  │
│  │  ┌──────────────────────────────────────────────┐  │  │
│  │  │  12. 返回响应                                 │  │  │
│  │  │      HTTP 200 OK                             │  │  │
│  │  │      {"users": [...]}                        │  │  │
│  │  └──────────────────────────────────────────────┘  │  │
│  └────────────────────────────────────────────────────┘  │
│                             ↓                            │
│  ┌────────────────────────────────────────────────────┐  │
│  │  13. 响应返回（原路返回）                             │  │
│  │      veth pair → 节点网络栈                         │  │
│  │      conntrack 自动做反向 DNAT                      │  │
│  │      源地址: 10.0.1.25:8080 → 10.0.1.20:30080       │  │
│  └────────────────────────────────────────────────────┘  │
└────────────────────────────┬─────────────────────────────┘
                             │ 14. 返回给 ALB
                             ▼
┌──────────────────────────────────────────────────────────┐
│                      AWS ALB                             │
│  ┌────────────────────────────────────────────────────┐  │
│  │  15. 加密响应（HTTPS）                               │  │
│  └────────────────────────────────────────────────────┘  │
└────────────────────────────┬─────────────────────────────┘
                             │ 16. 返回给客户端
                             ▼
```

---

### 2.5 服务发现与 DNS

> Pod 内部之间如何使用 CoreDNS 与其他 Pod 进行流量转发，业务调用

**假设场景：** users 和 payment 是两个微服务，分别部署为 Pod，users 需要调用 payment 的 API。

1. users 业务代码通过 `http://payment-service/api/balance/{user_id}` 请求 payment 业务，执行支付
2. users Pod 发起 DNS 查询：
   - Pod 内 DNS resolver 读取 `/etc/resolv.conf`，根据 `search default.svc.cluster.local svc.cluster.local cluster.local` 补全域名
   - 补全后的 FQDN：`payment-service.default.svc.cluster.local`
   - DNS 查询发送到 CoreDNS Pod（可能在任意节点），CoreDNS 从内存缓存返回 payment Service 的 ClusterIP
3. users Pod 在应用层与 payment-service:80 建立连接，传输层创建 TCP socket，网络层构建 IP 数据包
4. 数据包离开 users Pod，通过 veth pair 进入 Node 网络栈（Linux Kernel）：
   - 经过 iptables/IPVS 规则，DNAT 将目标地址从 ClusterIP 改为 payment Pod IP
   - **同节点**：直接通过 veth pair 转发到 payment Pod
   - **跨节点**：通过 CNI 网络（Overlay/Underlay）转发到目标节点，再到 payment Pod
5. 数据包进入 payment Pod 网络栈，流程和在 users Pod 流程一致。**详细信息参考上一个完整流程图和描述。**

```
┌─────────────────────────────────────────────────────────┐
│                    Control Plane                        │
│  ┌────────────┐         ┌──────────────────────────┐    │
│  │    etcd    │         │ EndpointSlice Controller │    │
│  └────────────┘         └──────────────────────────┘    │
│        ↑                           │                    │
│        │      ┌────────────────────┘                    │
│        │      ↓ 写入 EndpointSlice                       │
│  ┌─────┴──────────┐                                     │
│  │   API Server   │                                     │
│  └────────────────┘                                     │
└─────────────────────────────────────────────────────────┘
               ↑ Watch (Service + EndpointSlice)
    ┌──────────┼──────────┬──────────┐
    ↓          ↓          ↓          ↓
┌─────────────────────────────────────────────────────────┐
│                      Worker Node                        │
│  ┌──────────────┐  ┌──────────────┐                     │
│  │ kube-proxy   │  │   CoreDNS    │ ← 可能在任意节点      │
│  │ 配置规则      │  │   DNS 解析    │                     │
│  └──────────────┘  └──────────────┘                     │
│  ┌──────────────┐  ┌──────────────┐                     │
│  │  users Pod   │  │ payment Pod  │                     │
│  └──────────────┘  └──────────────┘                     │
│  users Pod 通过 CoreDNS 找到 Service ClusterIP，          │
│  通过 kube-proxy 配置的规则转发到 payment Pod。             │
└─────────────────────────────────────────────────────────┘
```

总结：
- **CoreDNS** 负责"找到" Service（DNS 解析 Service 名称 → ClusterIP）
- **kube-proxy** 负责"转发到" Pod（配置 iptables 规则，DNAT ClusterIP → Pod IP）
  - **kube-proxy 提前配置好规则 → 数据包经过时，iptables/IPVS 自动处理**
  - **✅ kube-proxy 是"配置者"，不是"执行者"**
- **kube-proxy 与 API Server 交互**，读取 Service 和 EndpointSlice
- **EndpointSlice Controller 在 Control Plane**，负责维护 Pod 和 Service 的映射关系

---

#### 2.5.1 拓扑感知路由

> 如何避免跨 AZ 流量

避免跨 AZ 流量需要同时从 **调度分布**，**服务转发**，**入站负载均衡**，**存储选址** 四个层面入手。

**核心思路：**

1. 启用**拓扑感知路由/提示**，合理使用 **internalTrafficPolicy** 和 **externalTrafficPolicy**，保证每个 AZ 都有就绪副本可就近服务
2. 在**负载均衡侧关闭跨区流量分发**（cross-zone load balancing）
3. 通过**调度约束**配合**卷的同 AZ 绑定**，实现全链路同 AZ 流量的优先或限定

**目标与原则：**

1. **目标：** 三种流量场景都优先或仅在同一 AZ 内完成转发，避免跨区延迟和数据传输费用
   - **Pod -> Pod**：直接通信（不经过 Service）
   - **Pod -> Service**：集群内服务调用
   - **外部 -> Service**：入站流量（通过 NLB/ALB）
2. **原则：有副本才能就近路由。** 必须先保证每个 AZ 都有足够的副本与端点，再用拓扑感知路由让 kube-proxy 或负载均衡选择本 AZ 目标

---

##### 2.4.1.1 Pod -> Service（集群内服务调用）

**场景：** Pod 作为客户端通过 Service 访问后端 Pod
**机制：** 拓扑感知路由（Topology Aware Routing）通过 EndpointSlice hints 让客户端所在节点的 kube-proxy 优先选择同 AZ 的后端 Pod
**工作原理：**
```
客户端 Pod (us-east-1a)
    ↓ 访问 backend-service:80
客户端所在节点的 kube-proxy (us-east-1a)
    ↓ 读取 EndpointSlice hints
    ↓ 发现 hint 指向 us-east-1a 的端点
iptables/IPVS 规则（优先选择同 AZ）
    ↓
后端 Pod (us-east-1a) ← 同 AZ，避免跨区
```
**配置步骤：**
1. **启用 Service 拓扑感知模式：**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: backend-service
  annotations:
    service.kubernetes.io/topology-mode: Auto  # 启用拓扑感知
spec:
  selector:
    app: backend
  ports:
  - port: 80
    targetPort: 8080
```
2. **EndpointSlice Controller 自动生成 hints：**
在 EKS 中，kube-controller-manager 的 EndpointSlice Controller 会自动为端点打上 hints：
```yaml
apiVersion: discovery.k8s.io/v1
kind: EndpointSlice
metadata:
  name: backend-service-abc123
  labels:
    kubernetes.io/service-name: backend-service
addressType: IPv4
endpoints:
  # 端点 1：Pod 在 us-east-1a
- addresses:
  - "10.0.1.10"
  conditions:
    ready: true
  nodeName: node-1
  zone: us-east-1a          # Pod 所在的 AZ
  hints:                    # ← 这就是 hint（提示）
    forZones:
    - name: "us-east-1a"    # 提示：这个 Pod 适合 us-east-1a 的客户端

  # 端点 2：Pod 在 us-east-1b
- addresses:
  - "10.0.2.20"
  conditions:
    ready: true
  nodeName: node-4
  zone: us-east-1b
  hints:
    forZones:
    - name: "us-east-1b"    # 提示：这个 Pod 适合 us-east-1b 的客户端

  # 端点 3：Pod 在 us-east-1c
- addresses:
  - "10.0.3.30"
  conditions:
    ready: true
  nodeName: node-7
  zone: us-east-1c
  hints:
    forZones:
    - name: "us-east-1c"    # 提示：这个 Pod 适合 us-east-1c 的客户端

ports:
- port: 8080
  protocol: TCP
```
**EndpointSlice Controller 位置：**
```
kube-controller-manager（K8s 控制平面组件）
│
├─ Deployment Controller
├─ ReplicaSet Controller
├─ Service Controller
├─ Endpoint Controller（传统）
├─ EndpointSlice Controller  ← 这里（内置，自动生成 hints）
├─ Node Controller
└─ ... 其他控制器
```
3. **kube-proxy 根据 hints 调整转发规则：**
iptables 模式示例：
```bash
# kube-proxy 根据 hints 调整概率权重
Chain KUBE-SVC-BACKEND (1 references)
# 同 AZ 端点权重更高
KUBE-SEP-POD-1A  statistic mode random probability 0.80  # us-east-1a (本地)
KUBE-SEP-POD-1B  statistic mode random probability 0.10  # us-east-1b (备用)
KUBE-SEP-POD-1C  statistic mode random probability 0.10  # us-east-1c (备用)
```
IPVS 模式示例：
```bash
# IPVS 通过权重实现拓扑感知
TCP  10.96.100.50:80 wrr
  -> 10.0.1.10:8080    Masq    8      # us-east-1a (高权重)
  -> 10.0.2.20:8080    Masq    1      # us-east-1b (低权重)
  -> 10.0.3.30:8080    Masq    1      # us-east-1c (低权重)
```
4. **极端场景：仅本节点转发（internalTrafficPolicy: Local）**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: backend-service
spec:
  internalTrafficPolicy: Local  # 只转发到本节点的 Pod
  selector:
    app: backend
  ports:
  - port: 80
    targetPort: 8080
```
**注意：** 如果本节点没有健康 Pod，请求会失败。需配合 DaemonSet 或确保每个节点都有副本。

---

##### 2.4.1.2 Pod -> Pod（直接通信）

**场景：** Pod 通过 Pod IP 直接通信（不经过 Service）
**问题：** CNI 网络默认会跨 AZ 路由，无法自动感知拓扑
**解决方案：**

**方案 1：通过 Service 访问（最佳实践，优先推荐）**

始终通过 Service 访问，让 kube-proxy 自动处理拓扑路由，无需应用层感知 AZ。

```yaml
apiVersion: v1
kind: Service
metadata:
  name: backend-service
  annotations:
    service.kubernetes.io/topology-mode: Auto
spec:
  selector:
    app: backend
  ports:
  - port: 80
    targetPort: 8080
```

```python
# 应用代码（无需感知 AZ 和 Pod IP）
import requests

# 直接使用 Service 名称，Kubernetes 自动处理同 AZ 路由
response = requests.get("http://backend-service/api")
```

**优势：**
- 简单：无需应用代码改动
- 自动：kube-proxy 自动选择同 AZ 的 Pod
- 解耦：应用不感知基础设施细节
- 可靠：自动负载均衡和健康检查

---

**以下方案仅在特殊场景使用：**
- 必须连接特定 Pod（如 StatefulSet 的 `mysql-0`）
- 应用自己实现服务发现（如 Consul、Eureka）
- P2P 通信场景（如分布式数据库集群内部通信）
- gRPC 客户端负载均衡

**方案 2：使用 Headless Service + 拓扑感知**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: backend-headless
  annotations:
    service.kubernetes.io/topology-mode: Auto
spec:
  clusterIP: None  # Headless Service
  selector:
    app: backend
  ports:
  - port: 8080
```

通过 DNS 查询时，EndpointSlice hints 会影响返回的 Pod IP 顺序（同 AZ 的 IP 优先返回）。

**方案 3a：Downward API（获取自己的 AZ）**

适合：应用需要知道自己的位置，配合其他服务发现机制使用

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp
spec:
  containers:
  - name: app
    image: myapp:latest
    env:
    - name: MY_POD_AZ
      valueFrom:
        fieldRef:
          fieldPath: metadata.labels['topology.kubernetes.io/zone']
    - name: MY_POD_IP
      valueFrom:
        fieldRef:
          fieldPath: status.podIP
```

**方案 3b：Kubernetes API（获取目标 Pod 的 AZ）**

适合：应用需要动态查询和选择目标 Pod

```yaml
# 需要 RBAC 权限
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader
  namespace: default
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: pod-reader-binding
  namespace: default
subjects:
- kind: ServiceAccount
  name: default
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

应用代码示例：
```go
// 直接从 Kubernetes API 获取目标 Pod 的 AZ 信息
import (
    metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
    "k8s.io/client-go/kubernetes"
    "k8s.io/client-go/rest"
)

config, _ := rest.InClusterConfig()
clientset, _ := kubernetes.NewForConfig(config)

pods, _ := clientset.CoreV1().Pods("default").List(ctx, metav1.ListOptions{
    LabelSelector: "app=backend",
})

// 直接读取目标 Pod 的 zone 标签
for _, pod := range pods.Items {
    targetAZ := pod.Labels["topology.kubernetes.io/zone"]
    // 选择同 AZ 的 Pod
}
```

**方案 3c：组合使用（特殊场景推荐）**

Downward API 获取自己的 AZ + Kubernetes API 获取目标 Pod 列表

```python
import os
from kubernetes import client, config

# 1. 从 Downward API 获取自己的 AZ（轻量级，无需 API 调用）
MY_AZ = os.getenv("MY_POD_AZ")

# 2. 从 Kubernetes API 获取目标 Pod 列表
config.load_incluster_config()
v1 = client.CoreV1Api()
pods = v1.list_namespaced_pod(
    namespace="default",
    label_selector="app=backend"
)

# 3. 优先选择同 AZ 的 Pod
same_az_pods = [
    p for p in pods.items 
    if p.metadata.labels.get("topology.kubernetes.io/zone") == MY_AZ
    and p.status.phase == "Running"
]

target_pod = same_az_pods[0] if same_az_pods else pods.items[0]
target_ip = target_pod.status.pod_ip
```

**注意：** 以上方案均适用于生产环境，不是测试专用。选择哪个方案取决于应用的服务发现机制：
- 方案 3a：应用已有外部服务发现（如 Consul、Eureka），只需知道自己的 AZ
- 方案 3b：应用完全依赖 Kubernetes API 进行服务发现
- 方案 3c：平衡性能和灵活性，减少 API 调用次数


##### 2.4.1.3 外部 -> Service（入站流量）

**场景：** 通过 ALB/NLB 访问 EKS Service

**方案 1：NLB + externalTrafficPolicy: Local（推荐）**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: frontend
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-type: "nlb"
    service.beta.kubernetes.io/aws-load-balancer-scheme: "internet-facing"
    # 关键：禁用跨区负载均衡
    service.beta.kubernetes.io/aws-load-balancer-cross-zone-load-balancing-enabled: "false"
spec:
  type: LoadBalancer
  externalTrafficPolicy: Local  # 只转发到本节点 Pod
  selector:
    app: frontend
  ports:
  - port: 80
    targetPort: 8080
```

**流程：**
```
用户请求
  ↓
NLB (us-east-1a)
  ↓ cross-zone-load-balancing: false
只选择 us-east-1a 的目标组节点
  ↓
节点 (us-east-1a)
  ↓ externalTrafficPolicy: Local
只转发到本节点的 Pod (us-east-1a)
  ↓
Pod (us-east-1a) ← 全程同 AZ
```

**优势：**
- 保留客户端源 IP
- 避免跨 AZ 流量
- 降低延迟

**注意：** 需确保每个 AZ 的节点都有健康 Pod，否则健康检查失败。

**方案 2：ALB Ingress + Target Type IP**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: frontend
  annotations:
    service.kubernetes.io/topology-mode: Auto  # 拓扑感知
spec:
  type: ClusterIP  # 不是 LoadBalancer
  selector:
    app: frontend
  ports:
  - port: 80
    targetPort: 8080
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: frontend-ingress
  annotations:
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip  # 直接注册 Pod IP
    # ALB 自动按 AZ 分组目标
spec:
  ingressClassName: alb
  rules:
  - host: example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: frontend
            port:
              number: 80
```

**ALB 行为：**
- AWS Load Balancer Controller 自动将 Pod IP 按 AZ 分组注册到 ALB Target Group
- ALB 默认优先将来自某 AZ 的请求路由到同 AZ 的目标（如果有健康目标）
- 集群内部通过 topology hints 保持同 AZ

**方案 3：NLB + Topology Aware Routing（最佳性能）**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: frontend
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-type: "nlb"
    service.beta.kubernetes.io/aws-load-balancer-nlb-target-type: "ip"  # IP 模式
    service.beta.kubernetes.io/aws-load-balancer-cross-zone-load-balancing-enabled: "false"
    service.kubernetes.io/topology-mode: Auto  # 拓扑感知
spec:
  type: LoadBalancer
  selector:
    app: frontend
  ports:
  - port: 80
    targetPort: 8080
```

**优势：**
- NLB 按 AZ 分组注册 Pod IP
- 关闭跨区负载均衡
- 集群内部通过 topology hints 保持同 AZ
- 端到端同 AZ 闭环

---

##### 2.4.1.4 调度分布（副本分区就近服务）

**前提：** 拓扑感知路由的前提是每个 AZ 都有就绪副本

**配置 Pod Topology Spread Constraints：**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
spec:
  replicas: 6
  selector:
    matchLabels:
      app: backend
  template:
    metadata:
      labels:
        app: backend
    spec:
      topologySpreadConstraints:
      - maxSkew: 1  # 各 AZ 副本数差异不超过 1
        topologyKey: topology.kubernetes.io/zone  # 按 AZ 分布
        whenUnsatisfiable: DoNotSchedule  # 无法满足时不调度
        labelSelector:
          matchLabels:
            app: backend
      containers:
      - name: backend
        image: backend:latest
```

**效果：** 6 个副本会均匀分布在 3 个 AZ（每个 AZ 2 个副本）

**结合 podAntiAffinity 避免单点故障：**

```yaml
spec:
  affinity:
    podAntiAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchLabels:
            app: backend
        topologyKey: kubernetes.io/hostname  # 不同节点
  topologySpreadConstraints:
  - maxSkew: 1
    topologyKey: topology.kubernetes.io/zone  # 不同 AZ
    whenUnsatisfiable: DoNotSchedule
    labelSelector:
      matchLabels:
        app: backend
```

---

##### 2.4.1.5 有状态和存储（PV/PVC）

**问题：** EBS 卷只能挂载到同 AZ 的节点，跨 AZ 会导致调度失败

**解决方案：**

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: ebs-sc
provisioner: ebs.csi.aws.com
volumeBindingMode: WaitForFirstConsumer  # 延迟绑定，等 Pod 调度后再创建卷
parameters:
  type: gp3
  encrypted: "true"
allowVolumeExpansion: true
```

**工作原理：**
1. Pod 先调度到某个节点（如 us-east-1a）
2. PVC 绑定时，在 Pod 所在 AZ 创建 EBS 卷
3. 避免卷/节点分属不同 AZ

**StatefulSet 配合 nodeSelector 固定 AZ：**

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: database
spec:
  replicas: 3
  selector:
    matchLabels:
      app: database
  template:
    metadata:
      labels:
        app: database
    spec:
      nodeSelector:
        topology.kubernetes.io/zone: us-east-1a  # 固定在 us-east-1a
      containers:
      - name: db
        image: postgres:14
        volumeMounts:
        - name: data
          mountPath: /var/lib/postgresql/data
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      storageClassName: ebs-sc
      resources:
        requests:
          storage: 100Gi
```

---

##### 2.4.1.6 自愈与扩缩容（副本/节点供给）

**确保每个 AZ 都有节点和副本：**

**Karpenter NodePool 配置：**

```yaml
apiVersion: karpenter.sh/v1beta1
kind: NodePool
metadata:
  name: default
spec:
  template:
    spec:
      requirements:
      - key: topology.kubernetes.io/zone
        operator: In
        values:
        - us-east-1a
        - us-east-1b
        - us-east-1c
      - key: karpenter.sh/capacity-type
        operator: In
        values:
        - on-demand
      nodeClassRef:
        name: default
  limits:
    cpu: 1000
  disruption:
    consolidationPolicy: WhenUnderutilized
```

**配合 HPA 和拓扑约束：**

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: backend-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: backend
  minReplicas: 6  # 至少 6 个副本（每个 AZ 2 个）
  maxReplicas: 30
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
```

**确保扩容时保持 AZ 均衡：** Deployment 的 topologySpreadConstraints 会自动生效，HPA 扩容的新副本会均匀分布到各 AZ。

---

##### 2.4.1.7 三种流量场景对比

| 流量类型               | 机制                 | 关键配置                                                                     | 是否需要 hints |
| ------------------ | ------------------ | ------------------------------------------------------------------------ | ---------- |
| **Pod -> Pod**     | CNI 直接路由           | ① Headless Service（`clusterIP: None`）② StatefulSet 固定 DNS ③ 应用层服务发现（Kafka/ES） | 可选         |
| **Pod -> Service** | kube-proxy + hints | `topology-mode: Auto`（拓扑感知路由时必需）                                        | 拓扑感知时必需   |
| **外部 -> Service（NLB）**  | NLB + 拓扑       | `cross-zone: false` + `externalTrafficPolicy: Local`                     | 推荐         |
| **外部 -> Service（ALB）**  | ALB + IP Mode   | Ingress 注解 `target-type: ip`（跳过 NodePort，直达 Pod）                        | 推荐         |

> 💡 **说明**：
> - **Pod -> Pod（CNI 直接路由）** 的典型场景：
>   - Headless Service（`clusterIP: None`）：DNS 直接返回 Pod IP，应用直连 Pod
>   - StatefulSet：每个 Pod 有固定 DNS 名称（如 `mysql-0.mysql-headless`），直接访问特定 Pod
>   - 应用层服务发现：应用自己维护 Pod IP 列表（如 Kafka、Elasticsearch 集群内部通信）
> - **Pod -> Service** 的 hints 是指拓扑感知路由（Topology Aware Routing），用于优先将流量路由到同 AZ 的 Pod，减少跨 AZ 流量费用
> - **NLB** 通过 `externalTrafficPolicy: Local` 保持客户端源 IP，流量只转发到本节点 Pod
> - **ALB** 通过 `target-type: ip` 直接将流量发送到 Pod IP，跳过 NodePort

**最佳实践：**
1. 所有 Service 启用 `topology-mode: Auto`
2. LoadBalancer Service 使用 `externalTrafficPolicy: Local` + `cross-zone: false`
3. 使用 topologySpreadConstraints 确保每个 AZ 都有副本
4. 存储使用 `WaitForFirstConsumer` 延迟绑定
5. Karpenter/HPA 配合拓扑约束实现自动扩缩容
---

### 2.6 调度器架构

> kube-scheduler 调度器

**kube-scheduler 是 Kubernetes 控制平面的核心组件**，负责为新创建的 Pod 选择最合适的节点。调度器通过 Watch API 监听未调度的 Pod，应用一系列过滤和打分算法，最终将 Pod 绑定到最优节点。

**调度器核心职责：**
1. **监听未调度 Pod**：Watch API Server，获取 `spec.nodeName` 为空的 Pod
2. **过滤节点**：排除不满足条件的节点（资源不足、Taint 不匹配等）
3. **打分排序**：对剩余节点打分，选择最高分节点
4. **绑定 Pod**：更新 Pod 的 `spec.nodeName`，通知 kubelet 拉起容器

**调度器架构图：**

```
┌─────────────────────────────────────────────────────────────────┐
│                      kube-scheduler                             │
│                                                                 │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │              Scheduling Queue（调度队列）                   │  │
│  │  • activeQ：待调度 Pod 队列                                 │  │
│  │  • backoffQ：调度失败重试队列                                │  │
│  │  • unschedulableQ：不可调度 Pod 队列                        │  │
│  └───────────────────────────────────────────────────────────┘  │
│                            ↓                                    │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │              Scheduling Framework（调度框架）               │  │
│  │                                                           │  │
│  │  ┌─────────────────────────────────────────────────────┐  │  │
│  │  │  PreFilter 插件（预过滤）                             │  │  │
│  │  │  • 预处理 Pod 信息                                    │  │  │
│  │  │  • 检查集群状态                                       │  │  │
│  │  └─────────────────────────────────────────────────────┘  │  │
│  │                            ↓                              │  │
│  │  ┌─────────────────────────────────────────────────────┐  │  │
│  │  │  Filter 插件（过滤阶段）                               │  │  │
│  │  │  • NodeResourcesFit：资源检查                         │  │  │
│  │  │  • NodeName：节点名称匹配                              │  │  │
│  │  │  • TaintToleration：污点容忍检查                       │  │  │
│  │  │  • NodeAffinity：节点亲和性检查                        │  │  │
│  │  │  • PodTopologySpread：拓扑分布检查                     │  │  │
│  │  │  • VolumeBinding：存储卷绑定检查                       │  │  │
│  │  └─────────────────────────────────────────────────────┘  │  │
│  │                            ↓                              │  │
│  │  ┌─────────────────────────────────────────────────────┐  │  │
│  │  │  PostFilter 插件（后过滤）                            │  │  │
│  │  │  • 抢占逻辑（Preemption）                             │  │  │
│  │  │  • 如果无可用节点，尝试驱逐低优先级 Pod                  │  │  │
│  │  └─────────────────────────────────────────────────────┘  │  │
│  │                            ↓                              │  │
│  │  ┌─────────────────────────────────────────────────────┐  │  │
│  │  │  Score 插件（打分阶段）                                │  │  │
│  │  │  • NodeResourcesBalancedAllocation：资源均衡打分       │  │  │
│  │  │  • ImageLocality：镜像本地性打分                       │  │  │
│  │  │  • InterPodAffinity：Pod 亲和性打分                   │  │  │
│  │  │  • NodeAffinity：节点亲和性打分                        │  │  │
│  │  │  • TaintToleration：污点容忍打分                       │  │  │
│  │  └─────────────────────────────────────────────────────┘  │  │
│  │                            ↓                              │  │
│  │  ┌─────────────────────────────────────────────────────┐  │  │
│  │  │  NormalizeScore 插件（分数归一化）                     │  │  │
│  │  │  • 将各插件分数归一化到 0-100                          │  │  │
│  │  └─────────────────────────────────────────────────────┘  │  │
│  │                            ↓                              │  │
│  │  ┌─────────────────────────────────────────────────────┐  │  │
│  │  │  Reserve 插件（资源预留）                              │  │  │
│  │  │  • 在绑定前预留资源                                    │  │  │
│  │  └─────────────────────────────────────────────────────┘  │  │
│  │                            ↓                              │  │
│  │  ┌─────────────────────────────────────────────────────┐  │  │
│  │  │  Permit 插件（许可检查）                               │  │  │
│  │  │  • 最后的许可检查                                     │  │  │
│  │  └─────────────────────────────────────────────────────┘  │  │
│  │                            ↓                              │  │
│  │  ┌─────────────────────────────────────────────────────┐  │  │
│  │  │  Bind 插件（绑定阶段）                                 │  │  │
│  │  │  • 更新 Pod.spec.nodeName                            │  │  │
│  │  │  • 通知 API Server                                   │  │  │
│  │  └─────────────────────────────────────────────────────┘  │  │
│  └───────────────────────────────────────────────────────────┘  │
│                                                                 │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │              Scheduler Cache（调度缓存）                    │  │
│  │  • 缓存节点信息                                             │  │
│  │  • 缓存 Pod 信息                                           │  │
│  │  • 提高调度性能                                             │  │
│  └───────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

**调度器架构说明：**

kube-scheduler 采用**插件化架构（Scheduling Framework）**，将调度过程分解为多个扩展点，每个扩展点可以注册多个插件。这种设计使调度器具有高度可扩展性，用户可以自定义插件实现特定调度策略。

**核心组件详解：**

1. **Scheduling Queue（调度队列）**
   - **activeQ**：存放待调度的 Pod，按优先级排序，高优先级 Pod 优先调度
   - **backoffQ**：存放调度失败需要重试的 Pod，采用指数退避策略（1s → 2s → 4s → ... → 10s 上限）
   - **unschedulableQ**：存放当前无法调度的 Pod，等待集群状态变化（如新节点加入、Pod 删除释放资源）后移回 activeQ

2. **Scheduling Framework（调度框架）扩展点**
   - **PreFilter**：预处理阶段，检查 Pod 是否可调度（如 PVC 是否存在），计算后续阶段需要的数据
   - **Filter**：过滤阶段，并行检查所有节点，排除不满足条件的节点（硬性约束）
   - **PostFilter**：后过滤阶段，当 Filter 无可用节点时触发，执行抢占逻辑（Preemption）
   - **PreScore**：预打分阶段，为 Score 阶段准备数据
   - **Score**：打分阶段，对通过 Filter 的节点打分（软性偏好），每个插件返回 0-100 分
   - **NormalizeScore**：分数归一化，将各插件分数按权重汇总
   - **Reserve**：资源预留阶段，在 Scheduler Cache 中预留资源，防止并发调度冲突
   - **Permit**：许可阶段，可以延迟绑定（如等待 Gang Scheduling 的其他 Pod）
   - **PreBind**：预绑定阶段，执行绑定前的准备工作（如创建 PV）
   - **Bind**：绑定阶段，更新 Pod 的 `spec.nodeName`，写入 API Server
   - **PostBind**：后绑定阶段，执行绑定后的清理工作

3. **Scheduler Cache（调度缓存）**
   - 缓存节点资源信息、标签、Taints，避免每次调度都查询 API Server
   - 缓存已调度 Pod 的资源占用，用于计算节点剩余可分配资源
   - 维护 Assumed Pods（已选择节点但尚未绑定的 Pod），防止资源超额分配
   - 通过 Informer 机制与 API Server 保持同步

**调度器与其他组件的交互：**

```
┌─────────────────────────────────────────────────────────────────┐
│                      API Server / etcd                          │
└─────────────────────────────────────────────────────────────────┘
                            ↑ ↓
                    Watch Pod / Update Pod
                            ↑ ↓
┌─────────────────────────────────────────────────────────────────┐
│                      kube-scheduler                             │
│  • Watch 未调度 Pod（spec.nodeName 为空）                         │
│  • 执行 Filter + Score                                          │
│  • 更新 Pod.spec.nodeName = "node-1"                            │
└─────────────────────────────────────────────────────────────────┘
                            ↓
                    Pod 绑定到节点
                            ↓
┌─────────────────────────────────────────────────────────────────┐
│                      kubelet (node-1)                           │
│  • Watch 绑定到本节点的 Pod                                       │
│  • 拉起容器                                                      │
│  • 上报 Pod 状态                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**调度器交互流程说明：**
1. **kube-scheduler Watch Pod**：通过 Informer 机制监听 API Server，获取 `spec.nodeName` 为空的未调度 Pod
2. **执行调度算法**：对 Pod 执行 Filter（过滤不满足条件的节点）和 Score（对剩余节点打分）
3. **绑定 Pod 到节点**：选择最高分节点，更新 Pod 的 `spec.nodeName` 字段，写回 API Server
4. **kubelet 拉起容器**：目标节点的 kubelet Watch 到绑定到本节点的 Pod，调用容器运行时拉起容器
5. **状态上报**：kubelet 持续上报 Pod 状态到 API Server，用户可通过 `kubectl get pod` 查看

---

#### 2.6.1 调度流程详解

**完整的 Pod 调度流程（从创建到运行）：**

```
┌─────────────────────────────────────────────────────────────────┐
│  1. 用户创建 Pod                                                 │
│     kubectl apply -f pod.yaml                                   │
└────────────────────────────┬────────────────────────────────────┘
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│  2. API Server 接收请求                                          │
│     • 验证 Pod 定义                                              │
│     • 写入 etcd                                                  │
│     • Pod.spec.nodeName = ""（未调度）                            │
│     • Pod.status.phase = "Pending"                              │
└────────────────────────────┬────────────────────────────────────┘
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│  3. kube-scheduler Watch 到未调度 Pod                            │
│     • Informer 监听 Pod 创建事件                                 │
│     • 过滤条件：spec.nodeName == ""                              │
│     • Pod 进入调度队列（activeQ）                                 │
└────────────────────────────┬────────────────────────────────────┘
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│  4. PreFilter 阶段（预过滤）                                      │
│     • 检查 Pod 的 PVC 是否存在                                    │
│     • 检查 Pod 的 ServiceAccount 是否存在                         │
│     • 预计算 Pod 亲和性/反亲和性                                   │
│     • 如果预过滤失败 → Pod 进入 unschedulableQ                     │
└────────────────────────────┬────────────────────────────────────┘
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│  5. Filter 阶段（过滤节点）                                        │
│     获取所有节点列表（从 Scheduler Cache）                          │
│                                                                 │
│     对每个节点执行过滤插件：                                        │
│     ┌─────────────────────────────────────────────────────┐     │
│     │ NodeResourcesFit 插件                                │     │
│     │ • 检查节点 CPU/内存是否满足 Pod 请求                    │     │
│     │ • 计算：节点可用资源 >= Pod requests                   │     │
│     │ • 不满足 → 排除该节点                                  │     │
│     └─────────────────────────────────────────────────────┘     │
│     ┌─────────────────────────────────────────────────────┐     │
│     │ NodeName 插件                                        │     │
│     │ • 如果 Pod 指定了 nodeName                            │     │
│     │ • 只保留该节点，排除其他所有节点                         │     │
│     └─────────────────────────────────────────────────────┘     │
│     ┌─────────────────────────────────────────────────────┐     │
│     │ TaintToleration 插件                                 │     │
│     │ • 检查节点 Taints 与 Pod Tolerations 是否匹配          │     │
│     │ • 未匹配 NoSchedule → 排除该节点                       │     │
│     └─────────────────────────────────────────────────────┘     │
│     ┌─────────────────────────────────────────────────────┐     │
│     │ NodeAffinity 插件                                    │    │
│     │ • 检查节点标签是否满足 Pod 的 nodeAffinity              │    │
│     │ • required 不满足 → 排除该节点                         │    │
│     └─────────────────────────────────────────────────────┘    │
│     ┌─────────────────────────────────────────────────────┐    │
│     │ PodTopologySpread 插件                               │    │
│     │ • 检查 Pod 在拓扑域的分布是否满足约束                    │    │
│     │ • 超过 maxSkew → 排除该节点                           │    │
│     └─────────────────────────────────────────────────────┘    │
│     ┌─────────────────────────────────────────────────────┐    │
│     │ VolumeBinding 插件                                   │    │
│     │ • 检查 PVC 是否能绑定到该节点的 AZ                      │    │
│     │ • EBS 卷跨 AZ → 排除该节点                            │    │
│     └─────────────────────────────────────────────────────┘    │
│                                                                │
│     过滤结果：                                                   │
│     • 可调度节点列表：[node-1, node-2, node-3]                    │
│     • 如果列表为空 → PostFilter（抢占逻辑）                        │
└────────────────────────────┬───────────────────────────────────┘
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│  6. Score 阶段（给节点打分）                                       │
│     对每个可调度节点执行打分插件：                                   │
│                                                                 │
│     ┌─────────────────────────────────────────────────────┐     │
│     │ NodeResourcesBalancedAllocation 插件                 │     │
│     │ • 计算节点资源使用均衡度                                │     │
│     │ • CPU 和内存使用越均衡，分数越高                         │     │
│     │ • 分数范围：0-100                                     │     │
│     └─────────────────────────────────────────────────────┘     │
│     ┌─────────────────────────────────────────────────────┐     │
│     │ ImageLocality 插件                                   │    │
│     │ • 检查节点是否已有 Pod 需要的镜像                       │     │
│     │ • 有镜像 → 分数更高（减少拉取时间）                      │    │
│     └─────────────────────────────────────────────────────┘    │
│     ┌─────────────────────────────────────────────────────┐    │
│     │ InterPodAffinity 插件                                │    │
│     │ • 根据 Pod 亲和性/反亲和性打分                          │    │
│     │ • 满足 preferred 亲和性 → 加分                        │    │
│     └─────────────────────────────────────────────────────┘    │
│     ┌─────────────────────────────────────────────────────┐    │
│     │ NodeAffinity 插件                                    │    │
│     │ • 根据 preferred nodeAffinity 打分                    │    │
│     │ • 满足 preferred 条件 → 加权重分                       │    │
│     └─────────────────────────────────────────────────────┘    │
│     ┌─────────────────────────────────────────────────────┐    │
│     │ TaintToleration 插件                                 │    │
│     │ • PreferNoSchedule Taint 降低分数                    │    │
│     │ • 有 Toleration → 不降分                             │    │
│     └─────────────────────────────────────────────────────┘    │
│                                                                │
│     打分结果示例：                                               │
│     • node-1: 85 分                                            │
│     • node-2: 92 分 ← 最高分                                    │
│     • node-3: 78 分                                            │
└────────────────────────────┬───────────────────────────────────┘
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│  7. 选择最高分节点                                                │
│     • 选择 node-2（92 分）                                       │
│     • 如果有多个节点同分 → 随机选择一个                              │
└────────────────────────────┬────────────────────────────────────┘
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│  8. Reserve 阶段（资源预留）                                       │
│     • 在调度缓存中预留资源                                          │
│     • 防止并发调度冲突                                             │
└────────────────────────────┬────────────────────────────────────┘
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│  9. Permit 阶段（许可检查）                                        │
│     • 最后的许可检查                                               │
│     • 可以延迟绑定（等待其他条件）                                   │
└────────────────────────────┬────────────────────────────────────┘
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│  10. Bind 阶段（绑定 Pod 到节点）                                  │
│      • 调用 API Server 更新 Pod                                  │
│      • Pod.spec.nodeName = "node-2"                             │
│      • 绑定成功 → Pod 从调度队列移除                                │
└────────────────────────────┬────────────────────────────────────┘
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│  11. kubelet 监听到 Pod 绑定                                      │
│      • node-2 上的 kubelet Watch 到 Pod                          │
│      • 拉取镜像                                                  │
│      • 创建容器                                                  │
│      • 启动容器                                                  │
└────────────────────────────┬────────────────────────────────────┘
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│  12. Pod 运行                                                    │
│      • Pod.status.phase = "Running"                             │
│      • kubelet 持续上报 Pod 状态到 API Server                     │
└─────────────────────────────────────────────────────────────────┘
```

---

#### 2.6.2 调度失败处理

**当 Filter 阶段没有可用节点时：**

```
┌─────────────────────────────────────────────────────────────────┐
│  Filter 阶段结果：可调度节点列表为空                                 │
└────────────────────────────┬────────────────────────────────────┘
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│  PostFilter 阶段（抢占逻辑）                                       │
│                                                                 │
│  1. 检查 Pod 是否有 PriorityClass                                 │
│     • 如果没有 → 调度失败，Pod 进入 unschedulableQ                  │
│                                                                 │
│  2. 如果有 PriorityClass：                                        │
│     • 查找节点上优先级更低的 Pod                                    │
│     • 计算驱逐这些 Pod 后是否能调度当前 Pod                          │
│                                                                 │
│  3. 选择抢占方案：                                                │
│     • 驱逐最少数量的低优先级 Pod                                    │
│     • 驱逐总优先级最低的 Pod 组合                                   │
│                                                                 │
│  4. 执行抢占：                                                    │
│     • 标记被抢占的 Pod 为 Terminating                              │
│     • 等待被抢占的 Pod 终止                                        │
│     • 重新调度当前 Pod                                            │
└────────────────────────────┬────────────────────────────────────┘
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│  如果抢占也失败：                                                  │
│  • Pod 进入 unschedulableQ                                       │
│  • Pod.status.conditions 添加调度失败原因                          │
│  • 等待集群状态变化后重试                                           │
│                                                                 │
│  常见失败原因：                                                   │
│  • Insufficient cpu（CPU 不足）                                  │
│  • Insufficient memory（内存不足）                                │
│  • 0/3 nodes are available: 3 node(s) had taint                 │
│  • 0/3 nodes are available: 3 node(s) didn't match affinity     │
└─────────────────────────────────────────────────────────────────┘
```

---

#### 2.6.3 调度器性能优化

**1. 调度缓存（Scheduler Cache）**

调度器维护一个内存缓存，避免每次调度都查询 API Server：

```
┌────────────────────────────────────────────────────────────────┐
│                    Scheduler Cache                             │
│                                                                │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  Node Cache（节点缓存）                                    │  │
│  │  • 节点资源信息（CPU、内存、GPU）                            │  │
│  │  • 节点标签和 Taints                                      │  │
│  │  • 节点状态（Ready、NotReady）                             │  │
│  │  • 更新机制：Watch API Server                              │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  Pod Cache（Pod 缓存）                                    │  │
│  │  • 已调度 Pod 的资源占用                                   │  │
│  │  │  • 用于计算节点剩余资源                                  │  │
│  │  • Pod 亲和性/反亲和性信息                                  │  │
│  │  • 更新机制：Watch API Server                              │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  Assumed Pods（假定 Pod）                                 │  │
│  │  • 已选择节点但尚未绑定的 Pod                                │  │
│  │  • 防止并发调度冲突                                         │  │
│  │  • 预留资源，避免超额分配                                    │  │
│  └──────────────────────────────────────────────────────────┘  │
└────────────────────────────────────────────────────────────────┘
```

**缓存更新流程：**
```
API Server 事件
  ↓
Informer 监听
  ↓
更新 Scheduler Cache
  ↓
调度器使用缓存数据（无需查询 API Server）
```

**性能提升**：
- 避免每次调度都查询 API Server
- 减少 API Server 压力
- 调度延迟从 ~100ms 降低到 ~10ms

---

**2. 调度循环与绑定循环**

调度器的工作分为两个阶段：

```
┌─────────────────────────────────────────────────────────────────┐
│                    Scheduling Queue                             │
│  activeQ: [Pod1, Pod2, Pod3, Pod4, Pod5, ...]                   │
└────────────────────────────┬────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│              Scheduling Cycle（调度循环 - 串行）                   │
│                                                                 │
│  Pod1 → Filter → Score → 选择节点 → Reserve                      │
│                                      │                          │
│  Pod2 等待 Pod1 完成后再开始            │                          │
└────────────────────────────┬─────────┼──────────────────────────┘
                             │         │
                             ▼         ▼
┌─────────────────────────────────────────────────────────────────┐
│              Binding Cycle（绑定循环 - 可并发）                    │
│                                                                 │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐              │
│  │ Bind Pod1   │  │ Bind Pod2   │  │ Bind Pod3   │              │
│  │ → node-1    │  │ → node-2    │  │ → node-3    │              │
│  └─────────────┘  └─────────────┘  └─────────────┘              │
└─────────────────────────────────────────────────────────────────┘
```

**并发控制**：
- **调度循环（Scheduling Cycle）串行执行**：一次只处理一个 Pod 的 Filter + Score
- **绑定循环（Binding Cycle）可并发执行**：多个 Pod 可同时绑定到不同节点
- 通过 Assumed Pods 机制防止资源冲突（在绑定完成前预留资源）

**性能提升**：
- 绑定阶段并发减少整体调度延迟
- 适合批量 Pod 创建场景（如 Job、CronJob）

---

**3. 调度队列优先级**

调度队列支持优先级调度：

```
┌─────────────────────────────────────────────────────────────────┐
│                    Scheduling Queue                             │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │  activeQ（活跃队列）                                        │  │
│  │  按 PriorityClass 排序：                                   │  │
│  │  ┌─────────────────────────────────────────────────────┐  │  │
│  │  │ 高优先级 Pod（PriorityClass: system-cluster-critical) │  │  │
│  │  │ • kube-system 组件                                   │  │  │
│  │  │ • 优先调度                                           │  │  │
│  │  └─────────────────────────────────────────────────────┘  │  │
│  │  ┌─────────────────────────────────────────────────────┐  │  │
│  │  │ 中优先级 Pod（PriorityClass: high-priority）          │  │  │
│  │  │ • 生产业务 Pod                                       │  │  │
│  │  └─────────────────────────────────────────────────────┘  │  │
│  │  ┌─────────────────────────────────────────────────────┐  │  │
│  │  │ 低优先级 Pod（PriorityClass: low-priority）           │  │  │
│  │  │ • 测试/开发 Pod                                      │  │  │
│  │  │ • 可被抢占                                           │  │  │
│  │  └─────────────────────────────────────────────────────┘  │  │
│  └───────────────────────────────────────────────────────────┘  │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │  backoffQ（退避队列）                                       │  │
│  │  • 调度失败的 Pod                                           │  │
│  │  • 指数退避重试（1s, 2s, 4s, 8s, ...）                       │  │
│  └───────────────────────────────────────────────────────────┘  │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │  unschedulableQ（不可调度队列）                              │  │
│  │  • 长期无法调度的 Pod                                       │  │
│  │  • 等待集群状态变化（如节点扩容）                              │  │
│  └───────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

**PriorityClass 示例：**
```yaml
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: high-priority
value: 1000000  # 优先级值，越大越高
globalDefault: false
description: "高优先级业务 Pod"
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: critical-pod
spec:
  priorityClassName: high-priority  # 使用高优先级
  containers:
  - name: app
    image: myapp:latest
```

---

**4. 调度器底层机制详解**

**Filter 插件工作原理：**

```
┌─────────────────────────────────────────────────────────────────┐
│  Filter 插件执行流程（以 NodeResourcesFit 为例）                    │
│                                                                 │
│  输入：                                                          │
│  • Pod: requests.cpu=2, requests.memory=4Gi                     │
│  • Node: allocatable.cpu=8, allocatable.memory=16Gi             │
│  • 已调度 Pod 占用: cpu=3, memory=6Gi                             │
│                                                                 │
│  计算过程：                                                      │
│  1. 计算节点剩余资源：                                            │
│     剩余 CPU = 8 - 3 = 5                                        │
│     剩余内存 = 16 - 6 = 10Gi                                     │
│                                                                 │
│  2. 检查是否满足 Pod 请求：                                        │
│     5 >= 2 ✅ CPU 满足                                          │
│     10Gi >= 4Gi ✅ 内存满足                                      │
│                                                                 │
│  3. 返回结果：                                                    │
│     • Success（节点通过过滤）                                      │
│                                                                 │
│  如果不满足：                                                     │
│     • Unschedulable（节点被排除）                                 │
│     • 原因：Insufficient cpu 或 Insufficient memory              │
└─────────────────────────────────────────────────────────────────┘
```

**Score 插件工作原理：**

```
┌─────────────────────────────────────────────────────────────────┐
│  Score 插件执行流程（以 NodeResourcesBalancedAllocation 为例）      │
│                                                                 │
│  目标：选择资源使用最均衡的节点                                      │
│                                                                 │
│  节点 1：                                                        │
│  • CPU 使用率：60%（3/5）                                         │
│  • 内存使用率：40%（6/15）                                         │
│  • 均衡度：|60% - 40%| = 20%                                      │
│  • 分数：100 - 20 = 80 分                                         │
│                                                                 │
│  节点 2：                                                        │
│  • CPU 使用率：50%（4/8）                                         │
│  • 内存使用率：50%（8/16）                                         │
│  • 均衡度：|50% - 50%| = 0%                                      │
│  • 分数：100 - 0 = 100 分 ← 最均衡，分数最高                        │
│                                                                 │
│  节点 3：                                                        │
│  • CPU 使用率：80%（8/10）                                        │
│  • 内存使用率：30%（6/20）                                         │
│  • 均衡度：|80% - 30%| = 50%                                     │
│  • 分数：100 - 50 = 50 分                                        │
│                                                                 │
│  最终选择：节点 2（100 分）                                        │
└─────────────────────────────────────────────────────────────────┘
```

**多插件分数聚合：**

```
节点 1 的总分计算：
  NodeResourcesBalancedAllocation: 80 分 × 权重 1 = 80
  ImageLocality: 50 分 × 权重 1 = 50
  InterPodAffinity: 70 分 × 权重 2 = 140
  ────────────────────────────────────────
  总分 = (80 + 50 + 140) / (1 + 1 + 2) = 67.5 分

节点 2 的总分计算：
  NodeResourcesBalancedAllocation: 100 分 × 权重 1 = 100
  ImageLocality: 100 分 × 权重 1 = 100
  InterPodAffinity: 80 分 × 权重 2 = 160
  ────────────────────────────────────────
  总分 = (100 + 100 + 160) / (1 + 1 + 2) = 90 分 ← 最高

最终选择：节点 2
```

---

**5. 调度器配置与调优**

**调度器配置文件示例：**

```yaml
apiVersion: kubescheduler.config.k8s.io/v1
kind: KubeSchedulerConfiguration
profiles:
- schedulerName: default-scheduler
  plugins:
    # 启用/禁用插件
    filter:
      enabled:
      - name: NodeResourcesFit
      - name: TaintToleration
      - name: NodeAffinity
      disabled:
      - name: "*"  # 禁用其他所有插件
    score:
      enabled:
      - name: NodeResourcesBalancedAllocation
        weight: 1
      - name: ImageLocality
        weight: 1
      - name: InterPodAffinity
        weight: 2  # 更高权重
  # 并发配置
  percentageOfNodesToScore: 50  # 只对 50% 节点打分（大集群优化）
```

**性能调优建议：**

| 场景 | 配置 | 说明 |
|------|------|------|
| **大集群（1000+ 节点）** | `percentageOfNodesToScore: 30` | 只对 30% 节点打分，提升速度 |
| **高并发调度** | 增加 Worker 数量 | 默认 16，可增加到 32-64 |
| **资源紧张** | 启用抢占 | 允许高优先级 Pod 驱逐低优先级 Pod |
| **特定硬件** | 自定义 Filter 插件 | 如 GPU 型号检查 |

---

### 2.7 调度约束

> K8S 核心调度约束条件

#### 2.7.1 调度约束概述

K8S 中的**亲和性（Affinity），反亲和性（Anti-Affinity），污染和容忍（Taints and Tolerations）概念属于 Pod 调度约束（Pod Scheduling Constraints）领域**，这是 K8S 调度（Scheduling）系统的一部分，用于控制 Pod 如何被分配到节点上，确保资源利用优化，高可用，故障隔离。这些约束条件帮助管理员根据标签（Labels），拓扑（Topology）或节点状态实现精细调度，避免 Pod 过度集中或调度到不合适的节点。

**K8S 核心调度约束条件一共有以下 4 种，这 4 种约束不是独立的，可组合使用（如：Affinity + Taints）**

- **节点亲和性（Node Affinity）**：Pod 偏好或强制调度到特定节点
  - Node Affinity 使用 requiredDuringSchedulingIgnoredDuringExecution 否定条件，在子级的 nodeSelectorTerms: operator 中通过 NotIn，DoesNotExist 操作符实现节点反亲和性
  - RequiredDuringSchedulingIgnoredDuringExecution 属于硬约束条件，过滤时必须匹配
  - PreferredDuringSchedulingIgnoredDuringExecution 属于软约束条件，打分时加权
- **Pod 亲和性（Pod Affinity）**：Pod 偏好或与特定 Pod 相邻的节点（基于拓扑，如同一 AZ）
- **Pod 反亲和性（Pod Anti-Affinity）**：Pod 避免调度到与特定 Pod 相邻的节点（高可用场景）
- **污点与容忍（Taints and Tolerations）**：节点标记 “排斥” 某些 Pod，Pod 通过容忍（Toleration）绕过。K8S 中的污点（Taints）和容忍（Tolerations）用于控制 Pod 不能/不宜调度到节点上，从而实现节点隔离，资源专用和高可用。
  - Taints 像节点的 “排斥标签”，默认阻止不匹配的 Pod 调度到该节点
  - Tolerations 像 Pod “豁免权”，允许 Pod 忽略特定 Taints 并调度到带污点的节点。只有 Pod 的 “Tolerations” 匹配节点的 Taints 时，Pod 才能调度成功，否则 Pod 只能 Pending。

**为什么 Affinity 不需要 Effect？**

| 维度   | Taints/Tolerations | Affinity/Anti-Affinity  |
| ---- | ------------------ | ----------------------- |
| 控制对象 | 节点主动"拒绝"           | Pod 主动"选择"              |
| 强度表达 | Effect 有 3 种：<br />• NoSchedule（硬约束）<br />• PreferNoSchedule（软约束）<br />• NoExecute（驱逐） | 有 2 种：<br />• required（硬约束）<br />• preferred（软约束） |
| 驱逐能力 | 有（NoExecute）       | 无（只影响调度）                |
| 实现位置 | 调度器 + 控制器          | 仅调度器                    |

#### 2.7.2 Taints 和 Tolerations

**Effect 类型对比：**

| Effect           | 阻止新 Pod | 驱逐已有 Pod | 适合升级场景 | 说明                                                                                                                                          |
| ---------------- | ------- | -------- | ------ | ------------------------------------------------------------------------------------------------------------------------------------------- |
| NoSchedule       | ✅       | 🔴       | 🔴     | 只能阻止新 Pod，已有 Pod 继续运行，无法清空节点，<br>使用类似 **GPU 专用节点，Master 节点隔离，渐进维护**<br>给 GPU 节点打污点，只允许需要 GPU 的 Pod 调度，<br>只有带 Toleration 的 Pod 才能调度到 GPU 节点 |
| PreferNoSchedule | 尽量      | 🔴       | 🔴     | 软约束，不可靠                                                                                                                                     |
| NoExecute        | ✅       | ✅        | ✅      | 能驱逐 Pod，**适合升级**                                                                                                                            |

**Taints 和 Tolerations 定义：**

- **Taints**：应用于节点（Node）的属性，由 key-value-effect 三元组组成（key=dedicated，value=gpu，effect=NoSchedule）
  - 标记 Node 不欢迎某些 Pod，防止普通 Pod 占用 gpu 资源或临时节点。Taints 不影响已运行的 Pod（除 NoExecute 效果），仅影响新调度
  - 位置：Node 对象的 .spec.taints 数组（API Server 持久化，kubectl taint 更新 etcd）
- **Tolerations**：应用于 Pod 的属性，定义 Pod 可以容忍的 Taintes 列表（key: "dedicated", operatpr: "Equal", value: "gpu", effect: "NoSchedule"）
  - Tolerations 为了让 Pod 绕过节点的 Taints，实现精确调度。只有精确或通过匹配的才能部署到 Taints 节点，解锁该 Taints 节点。
  - 位置：Pod 对象的 .spec.tolerations 数组（Pod 创建时注入）
- **协同作用**：Taints 和 Tolerations 类似于 “门和钥匙”，Taints 加锁，Pod 需要持有钥匙（Toleration 能够匹配的定义）才能进入。

**Taints 和 Tolerations 工作机制：**

Taints 和 Tolerations 的机制由 **kube-scheduler** 在调度框架中处理，**结合 API Server 存储和事件 Watch 实现**。

**核心是 “匹配过滤” 逻辑：** 调度时，kube-scheduler 检查节点所有 Taints 与 Pod Tolerations 的**交集**，未匹配 Taints 会触发排除或低优先级。

**Taints 和 Toleration 交互机制细节**

1. 存储位置：Taints 在 Node 对象数组由 API Server 存储在 etcd，Tolerations 在 Pod 对象数组，创建时注入。
2. **调度阶段检查（Filter/Predicate）**
   - kube-scheduler Watch 到新 Pod（未绑定节点），获取所有 Node
   - 对于每个节点：遍历节点 Taints 列表，对每个 Taint 检查 Pod Tolerations 是否有匹配项
      - 匹配规则
         - operator=“Equal”，key 和 value 必须相等
         - operator=“Exists”，仅 key 存在即可，忽略 value
         - effect 必须匹配
      - 如果节点所有 Taints 都有对应的 Tolerations，节点通过过滤，可调度
      - 如果未匹配 Taint，根据 Effect 处理
3. **Effect 类型的影响机制，Taint 必须配置 Effect，Toleration 是可选配置**
   - **NoSchedule**：硬约束，过滤阶段直接排除节点（未匹配的 Pod 不能调度）。常见专用节点（node-role.kubernetes.io/master:NoSchedule）
   - **PreferNoSchedule**：软约束，过滤阶段不排除，打分阶段（匹配无 Taint 节点优先 100 分，未匹配低至 1 分）机制：Priority Plugins 中加权，允许 Pod 偶尔调度到带 Taint 节点，但偏好避免。
   - **NoExecute**：调度 + 驱逐约束，同时阻止新 Pod 调度和驱逐已运行的 Pod。如果无匹配 Toleration，Eviction Controller（在 Controller Manager）立即驱逐不匹配的 Pod。可选 tolerationSeconds：延迟驱逐
   - **Effect 对比表：**

| Effect           | 阻止新 Pod | 驱逐已有 Pod | 强制性 | 典型用途       |
| ---------------- | ------- | -------- | --- | ---------- |
| NoSchedule       | ✅       | 🔴       | 硬约束 | 节点预留、分阶段维护 |
| NoExecute        | ✅       | ✅        | 最强  | 紧急维护、节点下线  |
| PreferNoSchedule | 尽量      | 🔴       | 软约束 | 优化建议、成本控制  |

4. **整体流程**
   - **Pod 创建 -> kube-scheduler 拉取 Node/Pod 数据 -> Filter：逐节点检查 Taints -> Tolerations 匹配 -> 通过的过滤节点进入 Score（资源/亲和性打分）-> Bind 到最高分数节点 -> kubelet 拉起 Pod**
   - **多 Taints 处理（AND 机制，不是 OR 机制）**：节点有多个 Taints，Pod 必须全部匹配
   - **执行时忽略：**Taints/Tolerations 只在调度时检查，运行中节点不会自动迁移 Pod（NoExecute 除外）

4. **边缘机制**
   - 自动 Taints：节点内存压力时，NodeController 添加 “node.kubernetes.io/memory-pressure:NoSchedule”
   - 性能：大集群中，Taints 增加过滤开销，但 scheduler 缓存优化（Informer机制）
   - 移除 Taint 后：新 Pod 可调度，但已驱逐的 Pod 需手动调度。

5. 示例场景：
   - 专用 GPU 节点
   - Master 节点隔离：默认 Taint，防止工作 Pod 调度到控制平面
   - 临时节点：加 “temporary:NoExecute”，运行 Pod 5min 自动驱逐，回收资源

```
┌─────────────────────────────────────────────────────────┐
│              Kubernetes API Server                      │
│  ┌───────────────────────────────────────────────────┐  │
│  │  Taint Object (包含 Effect 字段)                   │  │
│  └───────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────┘
                         │
        ┌────────────────┼────────────────┐
        │                                 │
        ▼                                 ▼
┌───────────────────┐      ┌──────────────────────────┐
│    Scheduler      │      │   Node Lifecycle         │
│                   │      │   Controller             │
│  ┌─────────────┐  │      │  ┌────────────────────┐  │
│  │   Filter    │  │      │  │  TaintManager      │  │
│  │   Plugin    │  │      │  └────────────────────┘  │
│  └─────────────┘  │      │           │              │
│        │          │      │           ▼              │
│        ▼          │      │  处理 NoExecute           │
│  检查             │       │  驱逐 Pod                │
│  NoSchedule      │       └──────────────────────────┘
│  检查             │
│  NoExecute        │
│        │          │
│  ┌─────────────┐  │
│  │   Score     │  │
│  │   Plugin    │  │
│  └─────────────┘  │
│        │          │
│        ▼          │
│  处理              │
│  PreferNoSchedule │
│  (降低节点分数)     │
└───────────────────┘
```

#### 2.7.3 调度底层原理与机制

**K8S 调度过程由 kube-scheduler 驱动**，由 Pod 创建时，scheduler 从 API Server 获取 Pod 和节点信息（API Server 从 etcd 获取），应用这些约束进行过滤（Predicate/Filter）和打分 （Priority/Score），最终选择最佳节点并绑定（Bind）Pod。原理基于标签匹配（Label Matching）和拓扑感知（Topology-Aware）。

- **过滤阶段**：排除不满足硬约束的节点（如必须亲和或无容忍污点），如果无节点，Pod 保持 Pending
- **打分阶段**：对剩余节点打分（0-100 分），优先级插件计算 Affinity 偏好，资源利用率等，选最高分的节点。
- **机制核心**：所有约束通过 YAML 配置注入 Pod spec 或 Node spec，API Server 存储（etcd），scheduler 通过 Watch 事件或周期扫描应用。拓扑基于节点标签（如 kubernetes.io/hostname，topology.kubernetes.io/zone）实现 **”相邻“** 判断。
- **驱逐机制**：Taints 可结合 Eviction（驱逐）API，导致已调度的 Pod 被移除（NoExecute 效果）

**核心约束条件在 K8S 中的位置，这些约束条件分布在 K8S 的 API 中**

- **Pod spec 字段，位置： Pod 定义下的 .spec 中。API Server 持久化存储**
  - 亲和性（affinity.nodeAffinity / podAffinity / podAntiAffinity）
  - 容忍度（tolerations）
  - 节点选择器（nodeSelector）
- **Node spec 字段，位置：Node 对象的 .spec.taints 下，管理员通过 kubectl taint 添加**
  - 污点（taints）
  - Topology 标签在 Node 的 .metadata.labels 中
- **整理框架位置，属于调度框架（Scheduling Framework），在 kube-scheduler 的插件链中（Filter Plugins 和 Priority Plugins）**

**kube-scheduler 是唯一负责这些约束并部署 Pod 的核心组件**

- **职责**：监听未调度 Pod（Unscheduled Pods），应用所用约束过滤/打分，选择节点后执行 Bind 操作（更新 Pod 的 .spec.nodeName）。如果自定义调度器（Custom Scheduler）可部分接管，但默认仍使用 kube-scheduler。
  
  - **NodeSelector 节点选择器在 kube-scheduler 调度器中的交互过程**
    - kube-scheduler 通过 Watch 监听到新的 Pod（未绑定节点）时，会读取 Pod.spec.nodeSelector 配置。该阶段是 NodeSelector 与 kube-scheduler 交互
    - 在 Filter 过滤阶段，kube-scheduler 会遍历所有节点，检查节点标签是否匹配 nodeSelector，匹配节点会进入 Score 打分阶段，不匹配的直接排除。
    - 在绑定阶段，更新 Pod.spec.nodeName，kubelet 负责拉起 Pod

- **部署流程**：Pod 创建 -> API Server 存储 -> kube-scheduler Watch 到 Pod -> 应用约束条件 -> Bind 到 Node -> kubelet（节点代理） 拉起容器

- 其他组件辅助：API Server 存储配置，kubelet 报告节点状态（影响过滤），Controller Manager 可添加，移除 Taints 

**约束类型对比：**

| 约束类型                             | 领域/位置                                           | 底层原理                      | 机制（过滤/打分）                                                        | 适用场景                   | 优缺点                                   |
|:-------------------------------- |:----------------------------------------------- |:------------------------- |:---------------------------------------------------------------- |:---------------------- |:------------------------------------- |
| **节点亲和性 (Node Affinity)**        | 调度约束；Pod spec.nodeAffinity                      | 基于 Node labels 匹配         | 过滤：Required 必须匹配标签排除节点；打分：Preferred 加权 (e.g., 权重 1-100)          | 硬件/地域优化 (e.g., 高性能节点)  | 优点：简单精确；缺点：忽略执行期变化，不支持 Pod 间关系。       |
| **Pod 亲和性 (Pod Affinity)**       | 调度约束；Pod spec.affinity.podAffinity              | 基于其他 Pod labels + 拓扑域匹配   | 过滤：Required 检查拓扑内匹配 Pod 存在；打分：Preferred 基于匹配 Pod 数加分             | 服务局部性 (e.g., 前后端共置)    | 优点：支持拓扑感知；缺点：计算密集，集群大时查询慢。            |
| **Pod 反亲和性 (Pod Anti-Affinity)** | 调度约束；Pod spec.affinity.podAntiAffinity          | 基于其他 Pod labels + 拓扑域反向匹配 | 过滤：Required 排除有匹配 Pod 的拓扑；打分：Preferred 匹配少的分高                    | 高可用 (e.g., 副本分布)       | 优点：防单点故障；缺点：硬约束下可能无节点可用 (Pending 多)。  |
| **污点与容忍 (Taints/Tolerations)**   | 调度+驱逐约束；Node spec.taints & Pod spec.tolerations | 节点排斥 + Pod 容忍匹配           | 过滤：未匹配 Taint 排除 (NoSchedule) 或低分 (PreferNoSchedule)；NoExecute 驱逐 | 专用/隔离节点 (e.g., GPU 专用) | 优点：支持驱逐，灵活；缺点：需手动管理 Taint，易误驱逐运行 Pod。 |

#### 2.7.4 调度约束组合方案

**组合方案对比：**

| #   | 组合方案                                                          | 核心目标            | 硬约束/软约束 | 典型场景                                                            | 关键区别                                  | 优先级   |
| --- | ------------------------------------------------------------- | --------------- | ------- | --------------------------------------------------------------- | ------------------------------------- | ----- |
| 1   | Node Affinity +<br />Taints/Tolerations                       | 资源独占 + 类型匹配     | 硬约束     | GPU 节点、高性能计算、专用硬件                                               | Taint 阻止其他 Pod，Affinity 确保正确节点类型      | ⭐⭐⭐⭐⭐ |
| 2   | Pod Anti-Affinity + <br />Node Affinity                       | 高可用 + 节点筛选      | 硬约束     | 跨 AZ 部署、生产环境 Web 服务<br />**跨 AZ 需要使用 topologyKey**              | Anti-Affinity 保证分散，NodeAffinity 限定节点池 | ⭐⭐⭐⭐⭐ |
| 3   | Pod Affinity + <br />Pod Anti-Affinity                        | 就近部署 + 自身分散     | 混合约束    | 微服务 + 缓存、API + Redis<br /> **Affinity 靠近缓存层，AntiAffinity 自身分散** | 对外亲和（靠近依赖），对内反亲和（自身分散）                | ⭐⭐⭐⭐  |
| 4   | Pod Anti-Affinity +<br /> Taints/Tolerations                  | 独占 + 高可用        | 硬约束     | 关键业务（支付、订单）、数据库                                                 | Taint 隔离节点，Anti-Affinity 保证副本分散       | ⭐⭐⭐⭐  |
| 5   | Node Affinity + <br />Pod Affinity + <br />Taints/Tolerations | 三层约束（节点+Pod+隔离） | 硬约束     | 数据库主从、有状态服务                                                     | 同时控制节点类型、Pod 位置关系、节点隔离                | ⭐⭐⭐   |
| 6   | 全组合（4 种机制）                                                    | 多维度精细控制         | 混合约束    | 多租户 SaaS、企业级平台                                                  | 最复杂，覆盖所有调度维度                          | ⭐⭐⭐   |
| 7   | 软约束组合（Preferred）                                              | 尽力优化，可妥协        | 软约束     | 批处理、非关键工作负载                                                     | 资源不足时可降级，不会调度失败                       | ⭐⭐    |
| 8   | Taints/Tolerations 单独                                         | 节点隔离/驱逐         | 硬约束     | 节点维护、系统组件部署                                                     | 只控制"能否调度"，不关心"调度到哪"                   | ⭐⭐⭐   |

• 80% 场景用方案 1、2、3 即可
• 关键业务加方案 4
• 复杂平台才需要方案 5、6
• 方案 7 通常作为其他方案的软约束补充
• 方案 8 是运维工具，非业务调度

**组合决策树**

需要节点隔离？
├─ 是 → 需要 Pod 分散？
│      ├─ 是 → 方案 4（Anti-Affinity + Taints）
│      └─ 否 → 需要特定节点类型？
│             ├─ 是 → 方案 1（Node Affinity + Taints）
│             └─ 否 → 方案 8（仅 Taints）
│
└─ 否 → 需要高可用？
       ├─ 是 → 需要限定节点池？
       │      ├─ 是 → 方案 2（Anti-Affinity + Node Affinity）
       │      └─ 否 → 仅 Pod Anti-Affinity
       │
       └─ 否 → 需要 Pod 就近？
              ├─ 是 → 方案 3（Affinity + Anti-Affinity）
              └─ 否 → 方案 7（软约束）或无约束

### 2.8 控制器管理器

> Kube Controller Manager

Controller Manager 作为集群内部的管理控制中心，负责集群内的 Node、Pod 副本、服务端点（EndpointSlice）、命名空间（Namespace）、服务账号（ServiceAccount）、资源定额（ResourceQuota）的管理。当某个 Node 意外宕机时，Controller Manager 会及时发现并执行自动化修复流程，确保集群始终处于预期的工作状态。

```
┌─────────────────────────────────────────────────────────────────────┐
│                    kube-controller-manager 架构                      │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │                   kube-controller-manager                   │    │
│  │                    （单进程，多控制器）                        │    │
│  └─────────────────────────────────────────────────────────────┘    │
│                              │                                      │
│       ┌──────────────────────┼──────────────────────┐               │
│       │                      │                      │               │
│       ▼                      ▼                      ▼               │
│  ┌─────────────┐      ┌─────────────┐      ┌─────────────┐          │
│  │ Deployment  │      │ ReplicaSet  │      │    Node     │          │
│  │ Controller  │      │ Controller  │      │ Controller  │          │
│  └─────────────┘      └─────────────┘      └─────────────┘          │
│       │                      │                      │               │
│       ▼                      ▼                      ▼               │
│  ┌─────────────┐      ┌─────────────┐      ┌─────────────┐          │
│  │   Service   │      │EndpointSlice│      │ Namespace   │          │
│  │ Controller  │      │ Controller  │      │ Controller  │          │
│  └─────────────┘      └─────────────┘      └─────────────┘          │
│       │                      │                      │               │
│       ▼                      ▼                      ▼               │
│  ┌─────────────┐      ┌─────────────┐      ┌─────────────┐          │
│  │    Job      │      │  CronJob    │      │ResourceQuota│          │
│  │ Controller  │      │ Controller  │      │ Controller  │          │
│  └─────────────┘      └─────────────┘      └─────────────┘          │
│       │                      │                      │               │
│       ▼                      ▼                      ▼               │
│  ┌─────────────┐      ┌─────────────┐      ┌─────────────┐          │
│  │StatefulSet  │      │ DaemonSet   │      │   PV/PVC    │          │
│  │ Controller  │      │ Controller  │      │ Controller  │          │
│  └─────────────┘      └─────────────┘      └─────────────┘          │
│                                                                     │
│  所有控制器共享：                                                      │
│  • Informer 机制（List/Watch）                                       │
│  • Workqueue（去重、限速、延迟）                                       │
│  • Leader Election（多副本选主）                                      │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

**核心控制器职责：**

| 控制器 | 职责 | 监听资源 | 管理资源 |
|--------|------|----------|----------|
| Deployment | 管理无状态应用的声明式更新 | Deployment | ReplicaSet |
| ReplicaSet | 维护 Pod 副本数量 | ReplicaSet | Pod |
| StatefulSet | 管理有状态应用 | StatefulSet | Pod, PVC |
| DaemonSet | 确保每个节点运行一个 Pod | DaemonSet | Pod |
| Job | 管理一次性任务 | Job | Pod |
| CronJob | 管理定时任务 | CronJob | Job |
| Node | 管理节点生命周期 | Node | - |
| EndpointSlice | 维护 Service 后端 Pod 列表 | Service, Pod | EndpointSlice |
| Namespace | 管理命名空间生命周期 | Namespace | 级联删除 |
| ResourceQuota | 限制命名空间资源使用 | ResourceQuota | - |
| ServiceAccount | 管理服务账号 | ServiceAccount | Secret |
| PV/PVC | 管理持久卷绑定 | PV, PVC | - |

#### 2.8.1 控制器模式

##### 2.8.1.1 控制器模式概述

**控制器模式三个协作组件：Informer，Reconcile Loop，Leader Election**

**事件采集与缓存（Informer）-> 事件入队/触发（Workqueue）-> 业务收敛（Reconcile Loop）**，在多副本部署时用 Leader Election 保证只有一份控制器实例执行主动收敛，避免重复与冲突。Informer 提供变更事件和高效只读缓存。Reconcile Loop 实现声明式 “期望状态 -> 实际状态” 的收敛。

- **组件职责和链接关系**
  - **Informer（事件采集与缓存）：**
    - 职责：使用 List/Watch 持续订阅资源时间，把对象快照存储在本地 Indexer（只读缓存），并在事件回调中将 “受影响对象的 key ” 入队到控制器的工作队列（Workqueue）
    - 作用：把 API Server 的变更流转成 “可重放的 Key 队列 + 本地缓存”，降低对 API 的直接压力并防止重复拉取，提高一致性读与性能
  - **Reconcile Loop（协调循环）**
    - 职责：控制器从队列取出 Key，请求缓存/客户端读取最新对象状态，对比期望与实际，执行创建/跟新/删除等操作，直至收敛，要求幂等，快速返回，可选择 requeue 或定制重排
    - 作用：落地声明式模型，把 “资源的 spec 所表达的期望状态” 不断推进为 “集群实际状态”，并在丢事件/乱序时依然通过 “level-based 收敛” 达成正确性。
  - **Leader Election（选主）**
    - 职责：控制器副本数 > 1 时，为避免竞态，采用基于租约（Lease/ConfigMap/Endpoints）的选主，只有 Leader 执行主动 Reconcile，Follower 保持只读监控或待命；Leader 失效由其他副本接管。
    - 作用：保证高可用下的 “单写者” 语义，避免重复创建/删除外部资源，维持幂等一致性
- **整体拓扑，三者解决了一致性，性能，高可用**
  - **Informer** 启动 List+Watch，维护本地缓存（Indexer）并将事件对象的 key 入队到 Workqueue
    - Indexer 和 Workqueue 将海量的 Watch 事件转为 “本地读 + 批处理”，降低 API Server 压力
    - level-based 设计保证了即使事件丢失/乱序，Reconcile 仍能基于缓存做幂等收敛
  - Controller 的 **Reconcile** 消费 Workqueue 的 Key，使用缓存客户端 Get/List 读取对象与相关依赖对象，计算差异并调用 K8S API 或外部 API 实施变更，如需重试/延后，返回 Requeue 或 RequeueAfter
  - 若部署多副本，**Leader Election** 确保只能获得 Lease 的 Leader 执行上述主动收敛逻辑；Leader 变更时，新 Leader 继续消费队列并执行 Reconcile，保持连续性。
    - Leader Election 实现多副本选主与自动切换，确保了高可用
- **典型交互流程**
  - 用户提交/修改对象到 **API Server**，数据写入 etcd，形成 **“期望状态”**
  - 控制器的 **Informer 发现变更**并将对象 Key **入队到 Workqueue 队列**，**Reconcile 读取缓存/对象**，计算差异并创建/修改子对象（如 Deployment 控制器创建/缩放 ReplicaSet/Pod）
  - **kube-scheduler 调度器**为未绑定的 Pod 选择节点并写入绑定，**kubelet** 在目标节点拉起容器并上报状态，控制器再次感知到实际状态变化并持续调谐直到达成收敛

**控制器模式是 Kubernetes 实现 “声明式状态收敛” 的通用机制**，即被内置控制器采用（打包在 kube-controller-manager 内），也被各类独立控制器/Operator（如 Ingress/云负载均衡/存储/自研控制器）采用。通过 Watch API 与 API Server/etcd 交互来驱动集群状态走向期望值。不是单一组件本身，而是一套运行与控制面与外部控制器的通用模式，**核心交互对象：kube-apiserver，etcd，kube-scheduler，kubelet 及底层云接口。

**控制器使用位置**

- 内置控制器：副本类（ReplicaSet/Deployment），批处理（Job/CronJob），节点（Node），服务发现（EndpointSlice），账号（ServiceAccount），逻辑上各自独立，实际作为进程集合运行在 kube-controller-manager 中
- 独立控制器/Operator：通过 Custom Resource Definiation 扩展或接管内置资源的控制环境，如：Ingress Controller，云控制器（cloud-controller-manger），各类数据库/消息中间间 Operator，弹性与负载均衡控制器等。

**分别属于哪些组件**

- 内置控制器属于控制屏幕组件 kube-controller-manager（单进程运行多种控制器），统筹对内置资源的调协与垃圾回收等任务
- 第三方或自研控制器作为独立 Deployment 运行在集群上（或控制平面外部），但统一遵循相同的控制器模式（Informer + Workqueue + Reconcile），对 API Server 进行声明式调谐。

**与哪些组件交互**

- kube-apiserver/etcd：控制器通过 List/Watch 订阅对象变更，从 API Server 读取或写回对象，所有状态持久化到 etcd
- kube-scheduler：控制器创建/缩放子对象（如 Pod），调度决策由调度器执行决定绑定（bind）到节点，而这分工互补。
- kubelet：控制器促成对象变化（如新 Pod），kubelet 在节点落实容器声明周期并汇报状态，形成闭环
- 云接口：云控制器或负载均衡控制器期望状态调用云 API（创建 LB/硬盘/弹性IP），并回写 K8S 对象以反应云侧实际资源

##### 2.8.1.2 控制器核心组件

**Informer 工作原理**

```
┌─────────────────────────────────────────────────────┐
│                  API Server (etcd)                  │
└─────────────────────────────────────────────────────┘
                        ↓
              ┌─────────┴─────────┐
              │   Watch API       │ ← 长连接（HTTP/2）
              └─────────┬─────────┘
                        ↓
┌─────────────────────────────────────────────────────┐
│                    Reflector                        │
│  • List：初始化时获取全量资源                           │
│  • Watch：持续监听资源变化（Added/Modified/Deleted）    │
│  • 处理 ResourceVersion 实现断点续传                   │
└─────────────────────────────────────────────────────┘
                        ↓
┌─────────────────────────────────────────────────────┐
│                  DeltaFIFO 队列                      │
│  • 存储资源变更事件（Delta）                           │
│  • FIFO 保证事件顺序                                  │
│  • 去重：相同对象的多次变更合并                          │
└─────────────────────────────────────────────────────┘
                        ↓
              ┌─────────┴─────────┐
              │   Indexer         │
              │  （本地缓存）       │
              └─────────┬─────────┘
                        ↓
┌─────────────────────────────────────────────────────┐
│              Event Handler（用户代码）                │
│  • OnAdd：资源创建                                    │
│  • OnUpdate：资源更新                                 │
│  • OnDelete：资源删除                                 │
└─────────────────────────────────────────────────────┘
                        ↓
┌─────────────────────────────────────────────────────┐
│                  WorkQueue                          │
│  • 存储待处理的资源 Key（namespace/name）              │
│  • 支持延迟、限速、去重                                │
└─────────────────────────────────────────────────────┘
```

**Workqueue（工作队列）**

Workqueue 是控制器模式的核心组件，用于解耦事件接收和处理：

```
┌─────────────────────────────────────────────────────────┐
│                   Workqueue 工作原理                     │
│                                                         │
│  Informer 事件 ──▶ Add to Queue ──▶ Worker 处理          │
│                                                         │
│  特性：                                                  │
│  • 去重：相同 Key 只保留一个                               │
│  • 限速：失败重试有退避策略                                 │
│  • 延迟：支持延迟入队                                      │
│  • 并发：多 Worker 并行处理                                │
└─────────────────────────────────────────────────────────┘
```

**Reconcile Loop 调谐循环工作流程**

```
┌─────────────────────────────────────────────────────┐
│  1. 从 WorkQueue 获取资源 Key                         │
└─────────────────────────────────────────────────────┘
                        ↓
┌─────────────────────────────────────────────────────┐
│  2. 从 Indexer（本地缓存）读取资源对象                  │
│     • 如果不存在 → 可能已删除，执行清理                  │
└─────────────────────────────────────────────────────┘
                        ↓
┌─────────────────────────────────────────────────────┐
│  3. 读取期望状态（Spec）                               │
│     • Deployment.spec.replicas = 3                  │
└─────────────────────────────────────────────────────┘
                        ↓
┌─────────────────────────────────────────────────────┐
│  4. 查询当前状态（Status）                             │
│     • 查询关联的 ReplicaSet                           │
│     • 统计当前 Pod 数量 = 2                           │
└─────────────────────────────────────────────────────┘
                        ↓
┌─────────────────────────────────────────────────────┐
│  5. 计算差异（Diff）                                  │
│     • 期望 3 个，当前 2 个 → 需要创建 1 个              │
└─────────────────────────────────────────────────────┘
                        ↓
┌─────────────────────────────────────────────────────┐
│  6. 执行操作（Reconcile）                             │
│     • 调用 API Server 创建新 Pod                      │
└─────────────────────────────────────────────────────┘
                        ↓
┌─────────────────────────────────────────────────────┐
│  7. 更新状态（Status）                                │
│     • Deployment.status.replicas = 3                │
│     • Deployment.status.readyReplicas = 2           │
└─────────────────────────────────────────────────────┘
                        ↓
┌─────────────────────────────────────────────────────┐
│  8. 重新入队（如果需要）                               │
│     • 如果状态未收敛，延迟后重新处理                     │
└─────────────────────────────────────────────────────┘
```

**Leader Election 实现方式**

基于 etcd Lease 的实现

```
┌─────────────────────────────────────────────────────────┐
│              etcd（存储 Lease 对象）                      │
│  Key: /leases/kube-controller-manager                   │
│  Value: {                                               │
│    "holderIdentity": "controller-1",                    │
│    "leaseDurationSeconds": 15,                          │
│    "acquireTime": "2025-11-02T12:00:00Z",               │
│    "renewTime": "2025-11-02T12:00:10Z"                  │
│  }                                                      │
└─────────────────────────────────────────────────────────┘
                         ↑
        ┌────────────────┼────────────────┐
        │                │                │
┌───────────────┐ ┌───────────────┐ ┌───────────────┐
│ Controller 1  │ │ Controller 2  │ │ Controller 3  │
│   (Leader)    │ │  (Candidate)  │ │  (Candidate)  │
└───────────────┘ └───────────────┘ └───────────────┘
```

##### 2.8.1.3 内置控制器

###### 2.8.1.3.1 Deployment Controller

Deployment Controller 是最常用的工作负载控制器，负责管理无状态应用的声明式更新：

```
┌─────────────────────────────────────────────────────────────────────┐
│                   Deployment Controller 工作流程                     │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  1. 监听 Deployment 对象变化                                          │
│       │                                                             │
│       ▼                                                             │
│  2. 读取 spec.template（Pod 模板）和 spec.replicas                    │
│       │                                                             │
│       ▼                                                             │
│  3. 计算 Pod Template Hash（用于区分不同版本）                          │
│       │                                                             │
│       ▼                                                             │
│  4. 创建/更新 ReplicaSet：                                            │
│     • 新版本：创建新 ReplicaSet，逐步扩容                               │
│     • 旧版本：逐步缩容旧 ReplicaSet                                    │
│       │                                                             │
│       ▼                                                             │
│  5. 根据 strategy 执行滚动更新：                                       │
│     • RollingUpdate：maxSurge + maxUnavailable 控制节奏               │
│     • Recreate：先删除所有旧 Pod，再创建新 Pod                          │
│       │                                                             │
│       ▼                                                             │
│  6. 更新 status（replicas, readyReplicas, conditions）               │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

Deployment 与 ReplicaSet 的关系：
- Deployment 不直接管理 Pod，而是通过 ReplicaSet 间接管理
- 每次 Pod Template 变化会创建新的 ReplicaSet（保留历史版本用于回滚）
- `spec.revisionHistoryLimit` 控制保留的历史 ReplicaSet 数量（默认 10）
- 回滚操作：`kubectl rollout undo deployment/xxx --to-revision=N`

###### 2.8.1.3.2 ReplicaSet Controller

ReplicaSet Controller 替代已废弃的 Replication Controller，是 Deployment 的底层实现，负责维护 Pod 副本数量：

```
┌─────────────────────────────────────────────────────────────────────┐
│                    ReplicaSet Controller 工作流程                    │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  1. 监听 ReplicaSet 对象变化                                          │
│       │                                                             │
│       ▼                                                             │
│  2. 读取 spec.replicas（期望副本数）                                   │
│       │                                                             │
│       ▼                                                             │
│  3. 通过 selector 查询匹配的 Pod 数量（实际副本数）                      │
│       │                                                             │
│       ▼                                                             │
│  4. 计算差异：                                                       │
│     • 实际 < 期望 → 创建新 Pod                                        │
│     • 实际 > 期望 → 删除多余 Pod（按策略选择）                           │
│       │                                                             │
│       ▼                                                             │
│  5. 更新 status.replicas 和 status.readyReplicas                     │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

- ReplicaSet 通过 kube-apiserver 管理 Pod 生命周期
- 当 Pod 的 `RestartPolicy=Always` 时，ReplicaSet 会自动重建失败的 Pod
- Pod Template 变化不会影响已创建的 Pod（需要通过 Deployment 触发滚动更新）
- 删除 ReplicaSet 默认会级联删除其管理的 Pod（可通过 `--cascade=orphan` 保留）

###### 2.8.1.3.3 StatefulSet Controller

StatefulSet Controller 负责管理有状态应用，提供稳定的网络标识和持久存储：

```
┌─────────────────────────────────────────────────────────────────────┐
│                   StatefulSet Controller 工作流程                    │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  1. 监听 StatefulSet 对象变化                                         │
│       │                                                             │
│       ▼                                                             │
│  2. 按序号顺序创建 Pod（pod-0, pod-1, pod-2...）                       │
│     • 必须等待前一个 Pod Running + Ready 才创建下一个                   │
│       │                                                             │
│       ▼                                                             │
│  3. 为每个 Pod 创建对应的 PVC（如果定义了 volumeClaimTemplates）         │
│     • PVC 命名：{volumeClaimTemplate.name}-{pod.name}                │
│       │                                                             │
│       ▼                                                             │
│  4. 更新策略：                                                        │
│     • RollingUpdate：从最大序号开始逆序更新                            │
│     • OnDelete：手动删除 Pod 后才更新                                  │
│     • partition：只更新序号 >= partition 的 Pod                       │
│       │                                                             │
│       ▼                                                             │
│  5. 缩容时按逆序删除 Pod（保留 PVC，除非设置级联删除）                    │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

StatefulSet 核心特性：
- 稳定网络标识：Pod 名称固定（{statefulset}-{ordinal}），配合 Headless Service 提供稳定 DNS
- 稳定存储：每个 Pod 绑定独立 PVC，Pod 重建后自动挂载原 PVC
- 有序部署/扩缩容：按序号顺序操作，保证依赖关系
- 有序滚动更新：逆序更新，支持 partition 分批更新

###### 2.8.1.3.4 DaemonSet Controller

DaemonSet Controller 确保每个（或指定）节点上运行一个 Pod 副本：

```
┌─────────────────────────────────────────────────────────────────────┐
│                    DaemonSet Controller 工作流程                     │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  1. 监听 DaemonSet 和 Node 对象变化                                   │
│       │                                                             │
│       ▼                                                             │
│  2. 遍历所有节点，判断是否应该运行 Pod：                                 │
│     • nodeSelector / nodeAffinity 匹配                              │
│     • tolerations 是否容忍节点 Taints                                │
│       │                                                             │
│       ▼                                                             │
│  3. 对比期望与实际：                                                   │
│     • 节点应该有 Pod 但没有 → 创建 Pod                                 │
│     • 节点不应该有 Pod 但有 → 删除 Pod                                 │
│       │                                                             │
│       ▼                                                             │
│  4. 更新策略：                                                        │
│     • RollingUpdate：逐节点滚动更新（maxUnavailable 控制）             │
│     • OnDelete：手动删除 Pod 后才更新                                  │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

DaemonSet 典型使用场景：
- 日志收集：fluentd、filebeat（每个节点收集日志）
- 监控代理：node-exporter、datadog-agent（每个节点采集指标）
- 网络插件：calico-node、aws-node（每个节点运行 CNI）
- 存储插件：csi-node-driver（每个节点运行 CSI 驱动）

###### 2.8.1.3.5 Job Controller

Job Controller 负责管理一次性批处理任务：

```
┌─────────────────────────────────────────────────────────────────────┐
│                      Job Controller 工作流程                         │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  1. 监听 Job 对象变化                                                 │
│       │                                                             │
│       ▼                                                             │
│  2. 读取 spec 配置：                                                  │
│     • completions：需要成功完成的 Pod 数量                             │
│     • parallelism：并行运行的 Pod 数量                                │
│     • backoffLimit：失败重试次数                                      │
│       │                                                             │
│       ▼                                                             │
│  3. 创建 Pod 执行任务：                                                │
│     • Pod 成功（exit 0）→ 计入 succeeded                              │
│     • Pod 失败 → 重试直到 backoffLimit                                │
│       │                                                             │
│       ▼                                                             │
│  4. 判断完成条件：                                                     │
│     • succeeded >= completions → Job 完成                            │
│     • 失败次数 > backoffLimit → Job 失败                              │
│       │                                                             │
│       ▼                                                             │
│  5. 更新 status（active, succeeded, failed, conditions）             │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

Job 完成模式（completionMode）：
- NonIndexed（默认）：任意 Pod 成功即计数，适合无序任务
- Indexed（K8S 1.21+）：每个 Pod 有唯一索引，适合分片处理

###### 2.8.1.3.6 CronJob Controller

CronJob Controller 负责按 Cron 表达式定时创建 Job：

```
┌─────────────────────────────────────────────────────────────────────┐
│                    CronJob Controller 工作流程                       │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  1. 监听 CronJob 对象                                                 │
│       │                                                             │
│       ▼                                                             │
│  2. 解析 spec.schedule（Cron 表达式）                                  │
│     • 格式：分 时 日 月 周（如 "0 2 * * *" 每天凌晨 2 点）              │
│       │                                                             │
│       ▼                                                             │
│  3. 到达调度时间时创建 Job：                                           │
│     • Job 名称：{cronjob}-{timestamp}                                │
│     • 继承 CronJob 的 jobTemplate                                    │
│       │                                                             │
│       ▼                                                             │
│  4. 并发策略（concurrencyPolicy）：                                    │
│     • Allow：允许并发运行多个 Job                                      │
│     • Forbid：跳过本次调度（如果上次未完成）                            │
│     • Replace：取消正在运行的 Job，创建新 Job                          │
│       │                                                             │
│       ▼                                                             │
│  5. 历史清理：                                                        │
│     • successfulJobsHistoryLimit：保留成功 Job 数量（默认 3）          │
│     • failedJobsHistoryLimit：保留失败 Job 数量（默认 1）              │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

CronJob 注意事项：
- `startingDeadlineSeconds`：错过调度的容忍时间，超过则跳过
- 时区：K8S 1.27+ 支持 `spec.timeZone` 指定时区

###### 2.8.1.3.7 EndpointSlice Controller

EndpointSlice Controller（K8S 1.21+ 默认）替代 Endpoints Controller，解决了大规模集群下 Endpoints 对象过大的问题：

```
┌─────────────────────────────────────────────────────────────────────┐
│                 Endpoints vs EndpointSlice 对比                      │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Endpoints（旧）：                                                    │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │ 一个 Service 对应一个 Endpoints 对象                           │    │
│  │ 所有 Pod IP 存储在同一个对象中                                  │    │
│  │ 问题：1000+ Pod 时对象过大，更新开销高                           │    │
│  └─────────────────────────────────────────────────────────────┘    │
│                                                                     │
│  EndpointSlice（新）：                                                │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │ 一个 Service 对应多个 EndpointSlice 对象                       │    │
│  │ 每个 Slice 最多 100 个端点（可配置）                            │    │
│  │ 优势：增量更新，减少 etcd 和网络开销                             │    │
│  └─────────────────────────────────────────────────────────────┘    │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

EndpointSlice Controller 工作流程：
- 监听 Pod 创建/删除/更新事件
- 监听 Service 的 selector 变化
- 计算哪些 Pod 属于哪个 Service
- 将 Pod IP 分片存储到多个 EndpointSlice 对象
- kube-proxy 监听 EndpointSlice 更新 iptables/IPVS 规则

###### 2.8.1.3.8 Node Controller

Node Controller 负责管理节点生命周期和健康状态：

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Node Controller 工作机制                          │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  1. 节点注册：                                                       │
│     kubelet 启动 → 向 API Server 注册 Node 对象                       │
│                                                                    │
│  2. 心跳监控：                                                       │
│     kubelet 定期更新 Node.status（默认 10s）                          │
│     Node Controller 检查心跳超时（默认 40s）                           │
│                                                                     │
│  3. 状态转换：                                                        │
│     ┌─────────┐    心跳正常     ┌─────────┐                          │
│     │  Ready  │ ◄────────────► │ Unknown │                          │
│     └─────────┘    心跳超时     └─────────┘                          │
│                                      │                              │
│                              超时 5 分钟                             │
│                                      ▼                              │
│                               ┌─────────────┐                       │
│                               │ 添加 Taint   │                       │
│                               │ 驱逐 Pod     │                       │
│                               └─────────────┘                       │
│                                                                     │
│  4. 关键参数：                                                        │
│     --node-monitor-period=5s      # 检查节点状态间隔                   │
│     --node-monitor-grace-period=40s # 心跳超时阈值                    │
│     --pod-eviction-timeout=5m     # Pod 驱逐等待时间                  │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

注意：K8S 1.26+ 废弃了 `--pod-eviction-timeout` 参数，改用 Taint-based Eviction 机制。节点不可达时自动添加 `node.kubernetes.io/not-ready` 或 `node.kubernetes.io/unreachable` Taint，Pod 通过 `tolerationSeconds` 控制驱逐等待时间（默认 300s）。

###### 2.8.1.3.9 ResourceQuota Controller

ResourceQuota Controller 负责在 Namespace 级别限制资源使用：

| 资源类型 | 限制项 | 示例 |
|----------|--------|------|
| **计算资源** | CPU、Memory 总量 | `requests.cpu: 10`, `limits.memory: 20Gi` |
| **对象数量** | Pod、Service、Secret 等数量 | `pods: 100`, `services: 20` |
| **存储资源** | PVC 数量和总容量 | `persistentvolumeclaims: 10`, `requests.storage: 100Gi` |

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: compute-quota
  namespace: production
spec:
  hard:
    requests.cpu: "20"
    requests.memory: 40Gi
    limits.cpu: "40"
    limits.memory: 80Gi
    pods: "50"
    services: "10"
    secrets: "20"
    configmaps: "20"
```

注意：ResourceQuota 是 Namespace 级别的限制，Pod/Container 级别的限制使用 LimitRange。

###### 2.8.1.3.10 Namespace Controller

Namespace Controller 负责管理命名空间生命周期，特别是级联删除：

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Namespace 删除流程                                │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  1. 用户执行 kubectl delete namespace xxx                            │
│       │                                                             │
│       ▼                                                             │
│  2. API Server 设置 DeletionTimestamp                                │
│     Namespace 状态变为 "Terminating"                                 │
│       │                                                             │
│       ▼                                                             │
│  3. Namespace Controller 开始级联删除：                               │
│     • Pod、Deployment、ReplicaSet                                   │
│     • Service、EndpointSlice                                        │
│     • ConfigMap、Secret                                             │
│     • ServiceAccount、Role、RoleBinding                             │
│     • PVC（根据 reclaimPolicy）                                      │
│     • NetworkPolicy、ResourceQuota、LimitRange                      │
│       │                                                             │
│       ▼                                                             │
│  4. 所有资源删除完成后，移除 Namespace Finalizer                        │
│       │                                                             │
│       ▼                                                             │
│  5. Namespace 对象从 etcd 中删除                                      │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

常见问题：Namespace 卡在 "Terminating" 状态，通常是因为某些资源有 Finalizer 未清理。

###### 2.8.1.3.11 ServiceAccount Controller

ServiceAccount Controller 负责管理服务账号的自动创建和 Token 管理：

```
┌─────────────────────────────────────────────────────────────────────┐
│                 ServiceAccount Controller 工作流程                   │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  1. 监听 Namespace 创建事件                                           │
│       │                                                             │
│       ▼                                                             │
│  2. 自动创建 default ServiceAccount                                   │
│     • 每个 Namespace 都有一个 default SA                              │
│       │                                                             │
│       ▼                                                             │
│  3. Token 管理（K8S 1.24+ 变化）：                                     │
│     • 旧版本：自动创建永久 Secret Token                                │
│     • 新版本：使用 TokenRequest API 生成短期 Token                     │
│       │                                                             │
│       ▼                                                             │
│  4. Pod 挂载 ServiceAccount：                                         │
│     • 自动挂载 Token 到 /var/run/secrets/kubernetes.io/serviceaccount │
│     • 可通过 automountServiceAccountToken: false 禁用                 │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

ServiceAccount 与 RBAC 集成：
- ServiceAccount 是 Pod 在集群内的身份标识
- 通过 RoleBinding/ClusterRoleBinding 绑定权限
- 最小权限原则：为每个应用创建专用 SA，避免使用 default

###### 2.8.1.3.12 PersistentVolume Controller

PersistentVolume Controller 负责管理 PV 和 PVC 的绑定生命周期：

```
┌─────────────────────────────────────────────────────────────────────┐
│                  PV/PVC 绑定流程                                     │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  1. 用户创建 PVC（声明存储需求）                                        │
│       │                                                             │
│       ▼                                                             │
│  2. PV Controller 查找匹配的 PV：                                      │
│     • 容量 >= PVC 请求                                               │
│     • accessModes 匹配                                               │
│     • storageClassName 匹配                                          │
│     • 状态为 Available                                               │
│       │                                                             │
│       ├─ 找到匹配 PV → 绑定（PV.status = Bound）                       │
│       │                                                             │
│       └─ 未找到 + 有 StorageClass → 触发动态供应                       │
│            │                                                        │
│            ▼                                                        │
│          CSI Driver 创建实际存储卷                                    │
│            │                                                        │
│            ▼                                                        │
│          创建 PV 并绑定到 PVC                                         │
│                                                                     │
│  3. PVC 删除时根据 reclaimPolicy 处理 PV：                             │
│     • Retain：保留 PV 和数据，需手动清理                               │
│     • Delete：删除 PV 和底层存储                                      │
│     • Recycle（已废弃）：清空数据后重新可用                             │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

PV 状态流转：Available → Bound → Released → (Available/Failed)

###### 2.8.1.3.13 Service Controller

Service Controller（运行在 cloud-controller-manager）负责管理 LoadBalancer 类型 Service 与云平台的集成：

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Service Controller 工作流程                       │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  1. 监听 Service 对象变化                                             │
│       │                                                             │
│       ▼                                                             │
│  2. 检查 Service.spec.type == LoadBalancer ?                         │
│     ├─ 否 → 跳过                                                     │
│     └─ 是 → 继续                                                     │
│       │                                                             │
│       ▼                                                             │
│  3. 调用云平台 API：                                                  │
│     • 创建：调用 EnsureLoadBalancer() 创建 LB                         │
│     • 更新：调用 UpdateLoadBalancer() 更新后端                         │
│     • 删除：调用 EnsureLoadBalancerDeleted() 删除 LB                  │
│       │                                                             │
│       ▼                                                             │
│  4. 回写 Service.status.loadBalancer.ingress                         │
│     记录云平台分配的 LB IP/Hostname                                    │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

注意：K8S 1.6+ 将 Service Controller 从 kube-controller-manager 移到 cloud-controller-manager，实现云平台解耦。

#### 2.8.2 升级策略（概述）

> 详细内容请参考第 14 章「Kubernetes 升级策略」

**Kubernetes 原生策略：**

| 策略 | 说明 | 适用场景 |
|------|------|----------|
| RollingUpdate | 滚动更新（默认） | 99% 场景 |
| Recreate | 先删后建 | 不兼容、单例 |
| OnDelete | 手动控制 | StatefulSet/DaemonSet |

**高级策略（需要工具）：**

| 策略 | 说明 | 工具 |
|------|------|------|
| Blue-Green | 两套环境切换 | Service selector / Ingress |
| Canary | 渐进式验证 | Argo Rollouts / Flagger |
| A/B Testing | 基于用户路由 | Istio / Ingress |

**推荐组合：**
- 生产环境：RollingUpdate + PDB
- 关键业务：RollingUpdate + Canary（Argo Rollouts）
- 开发环境：Recreate（快速重启）
- 有状态应用：StatefulSet RollingUpdate + partition

#### 2.8.3 弹性伸缩（概述）

> 详细内容请参考第 13 章「Kubernetes 弹性伸缩」

**伸缩层级：**

| 层级 | 组件 | 说明 |
|------|------|------|
| **Pod 级别** | HPA | 基于 CPU/Memory 水平扩展 |
| | VPA | 垂直扩展（调整 requests/limits） |
| | KEDA | 事件驱动扩展（SQS/Kafka 等） |
| **节点级别** | Cluster Autoscaler | 基于 ASG 扩展节点 |
| | Karpenter | JIT 动态选择最优实例 |

**CA vs Karpenter 快速对比：**

| 维度 | Cluster Autoscaler | Karpenter |
|------|-------------------|-----------|
| 扩容速度 | 1-3 分钟 | < 1 分钟 |
| 节点选择 | 预定义 ASG | 动态选择最优 |
| Spot 支持 | 需要单独 ASG | 原生混合支持 |
| 成本优化 | 有限 | bin-packing + Consolidation |

**ConfigMap（配置下发）**

- 定义：键值或文件型配置对象，可通过环境变量、挂载卷或命令行参数注入到 Pod。
- 特点：命名空间级、默认体积上限（常见 1MiB 等实现限制）、可选 immutable: true（只读、跳过 watch、提升性能与安全）。
- 更新语义：卷挂载会刷新文件内容（需应用自己 reload），通过环境变量注入的需滚动更新 Pod 生效；建议与 Deployment 滚动更新搭配。

#### 2.8.4 健康检查探针

K8S 提供三种探针来检测容器健康状态：

| 探针类型 | 作用 | 失败后果 | 典型场景 |
|----------|------|----------|----------|
| **livenessProbe** | 检测容器是否存活 | 重启容器 | 检测死锁、无响应 |
| **readinessProbe** | 检测容器是否就绪 | 从 Service 摘除 | 启动预热、依赖检查 |
| **startupProbe** | 检测容器是否启动完成 | 重启容器 | 慢启动应用 |

**探针配置示例：**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app
spec:
  containers:
  - name: app
    image: myapp:v1
    livenessProbe:
      httpGet:
        path: /healthz
        port: 8080
      initialDelaySeconds: 15    # 首次检查延迟
      periodSeconds: 10          # 检查间隔
      timeoutSeconds: 3          # 超时时间
      failureThreshold: 3        # 失败阈值
    readinessProbe:
      httpGet:
        path: /ready
        port: 8080
      initialDelaySeconds: 5
      periodSeconds: 5
      successThreshold: 1        # 成功阈值
    startupProbe:
      httpGet:
        path: /healthz
        port: 8080
      failureThreshold: 30       # 允许 30 次失败
      periodSeconds: 10          # 最长等待 300s 启动
```

**探针检测方式：**

| 方式          | 说明                      | 适用场景          | 版本要求 |
| ----------- | ----------------------- | ------------- | ------- |
| `httpGet`   | HTTP GET 请求，2xx/3xx 为成功 | Web 应用        | - |
| `tcpSocket` | TCP 连接检测                | 数据库、非 HTTP 服务 | - |
| `exec`      | 执行命令，返回 0 为成功           | 自定义检测逻辑       | - |
| `grpc`      | gRPC 健康检查协议（需实现 grpc.health.v1.Health）| gRPC 服务       | K8S 1.24+ GA |

**探针执行顺序与相互关系：**

```
┌─────────────────────────────────────────────────────────────────────┐
│                      探针执行时序                                     │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  容器启动                                                            │
│      │                                                              │
│      ▼                                                              │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │ startupProbe 阶段（如果配置）                                  │    │
│  │ • liveness 和 readiness 探针被阻塞，不执行                     │    │
│  │ • 成功：进入正常探针阶段                                        │    │
│  │ • 失败超过 failureThreshold：重启容器                          │    │
│  └─────────────────────────────────────────────────────────────┘    │
│      │ startupProbe 成功                                            │
│      ▼                                                              │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │ 正常探针阶段（并行执行）                                         │    │
│  │                                                              │    │
│  │  livenessProbe ──────────────────────────────────────────    │    │
│  │  │ 失败 → kubelet 重启容器（根据 restartPolicy）               │    │
│  │                                                              │    │
│  │  readinessProbe ─────────────────────────────────────────    │    │
│  │  │ 失败 → 从 Service Endpoints 摘除（不重启）                  │    │
│  │  │ 恢复 → 重新加入 Service Endpoints                          │    │
│  └─────────────────────────────────────────────────────────────┘    │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

**探针配置参数详解：**

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `initialDelaySeconds` | 0 | 容器启动后首次探测延迟（秒） |
| `periodSeconds` | 10 | 探测间隔（秒） |
| `timeoutSeconds` | 1 | 探测超时时间（秒） |
| `successThreshold` | 1 | 连续成功次数才算成功（liveness/startup 必须为 1） |
| `failureThreshold` | 3 | 连续失败次数才算失败 |
| `terminationGracePeriodSeconds` | 30 | 探针失败触发重启时的优雅终止时间（可覆盖 Pod 级别设置） |

**探针配置最佳实践：**

1. **startupProbe 用于慢启动应用**
   - 避免设置过长的 `initialDelaySeconds`，改用 startupProbe
   - 计算公式：最大启动时间 = `failureThreshold × periodSeconds`

2. **liveness 和 readiness 应该检测不同端点**
   - liveness：检测进程是否存活（如 `/healthz`，轻量级）
   - readiness：检测是否可以处理请求（如 `/ready`，可包含依赖检查）

3. **避免 liveness 探针检测外部依赖**
   - 错误：liveness 检测数据库连接 → 数据库故障导致所有 Pod 重启
   - 正确：liveness 只检测进程本身，外部依赖放在 readiness

4. **合理设置超时和阈值**
   - `timeoutSeconds` 应小于 `periodSeconds`
   - 生产环境建议 `failureThreshold >= 3`，避免瞬时抖动触发重启

**常见配置错误：**

| 错误 | 后果 | 正确做法 |
|------|------|----------|
| liveness 检测外部依赖 | 级联故障，大规模重启 | 外部依赖放 readiness |
| initialDelaySeconds 过短 | 应用未启动就被杀 | 使用 startupProbe |
| timeoutSeconds 过短 | GC 暂停导致误杀 | 根据应用特性调整 |
| liveness = readiness 同端点 | 无法区分"不健康"和"暂时繁忙" | 分开设计端点 |

#### 2.8.5 调度约束（概述）

> 详细内容请参考第 12 章「Kubernetes 调度约束」

| 调度机制 | 作用 | 详见章节 |
|----------|------|----------|
| nodeSelector | 简单节点选择 | 12.1 |
| Node Affinity | 灵活节点亲和性 | 12.2 |
| Pod Affinity/Anti-Affinity | Pod 位置控制 | 12.3 |
| Taints & Tolerations | 节点排斥机制 | 12.4 |
| topologySpreadConstraints | 拓扑分布约束 | 12.5 |

#### 2.8.6 K8S 权限管理（概述）

> 详细内容请参考第 7 章「Kubernetes 安全架构」

1. 基于 RBAC 管理用户权限，只有 allow，没有 deny。可以通过 Role binding 向用户/组授于角色权限。
2. 按照 namespace 分配权限，尽量不使用通配符，定期审查/更新权限，不使用 admin-cluster 权限

##### 2.8.6.1 IAM 与 K8S 集成

1. 创建/更新 aws-auth 的 ConfigMap，添加 IAM Role/User 到 mapRoles/mapUsers
2. EKS v1.23 引入了 Cluster Access Manager 使用访问条目直接链接到 IAM 实例，主要通过 Access Entries 和 Access Policies

##### 2.8.6.2 Pod 级别 IAM 权限配置

1. 通过 kubectl apply -f eks-pod-identity.yaml 方式，或 Helm 方式安装一个 EKS Pod Identity 的 DaemonSet 程序的 Agent 为 Pod 提供 IAM 角色凭证。
2. 创建一个 AssumeRole 信任策略的 Role
3. 创建一个 kind:ServiceAccount 服务账号
4. aws eks create-pod-identity-association 指定 service account name, Role 的 ARN
5. aws eks list-pod-identity-association 验证

**ServiceAccount（集群内身份）**

- 定义：命名空间级账号，绑定 RBAC 权限并注入到 Pod（投影 Token + CA 证书 + API Server 地址），供工作负载安全访问 K8S API 或与外部身份系统对接。
- 作用：最小权限访问控制（Pod → API Server），与 Role/ClusterRole 及绑定配合；可设置 automountServiceAccountToken=false 阻止默认挂载。
- 云集成：在 EKS 中可结合 Pod 级身份（如 IRSA）把 ServiceAccount 映射到云 IAM 角色，以最小权限访问云 API。

#### 2.8.7 Storage（概述）

> 详细内容请参考第 6 章「容器存储架构」

![K8s Storage class, PV, and PVC. Kubernetes provides several mechanisms… |  by Park Sehun | Medium](https://miro.medium.com/v2/resize:fit:1400/0*jpP3h6TGRIq272j7.png)

- **PV** Persistent Volumes 持久卷，PV 是对 K8S 存储资源的抽象
- **PVC** Persistent Volume Claims 持久卷声明，PVC 是 Pod 对存储资源的一个申请，主要是存储空间申请，访问模式等。Pod 通过 PVC 向  PV 申请资源。
- Storage Classes 存储类，和 PVC 不同，**Storage Class 是动态模式**，不需要像 PVC 一样静态模式需要提前申请。Storage Class 根据 PVC 的需求动态创建合适的 PV 资源，从而实现存储卷的按需创建。
- CSI Drivers CSI 驱动程序

Authentication and Authorization（身份验证与授权）

- ServiceAccount 为 Pod 运行的进程提供身份，如果创建 Pod 没有指定，分配同一 namespace 默认服务账户

#### 2.8.8 Pod 中断预算（PDB）

Pod 中断预算（Pod Disruption Budget，PDB） 是 Kubernetes 中一个重要的特性，旨在确保在自愿中断期间（如节点维护、应用更新等），至少有一定数量的 Pods 处于可用状态。

- **PDB 不保护非计划性中断（硬件故障、直接删除 Pod）**

#### 2.8.9 Headless Service 与 Readiness Probe 对比
##### 2.8.9.1 核心概念区别

| 维度 | Headless Service | Readiness Probe |
|:-----|:-----------------|:----------------|
| 层级 | Service 级别（服务发现） | Pod 级别（健康检查） |
| 核心功能 | 返回 Pod IP/DNS 列表，无 ClusterIP | 决定 Pod 是否加入 EndpointSlice |
| 负载均衡 | 无，直接暴露 Pod IP | 间接支持，通过 EndpointSlice 过滤后由 Service 均衡 |

##### 2.8.9.2 StatefulSet 与 Deployment 的 Readiness 差异

| 维度     | StatefulSet Readiness                                                                     | 普通就绪探针（Deployment 等）                                        |
| :----- | :---------------------------------------------------------------------------------------- | :---------------------------------------------------------- |
| 作用范围   | 结合 Pod 稳定身份（ordinal/FQDN），<br>影响 Headless Service 的 SRV/端点有序暴露，<br>确保下线时按序摘除（如 web-2 先排空） | 仅过滤 EndpointSlice 中的 Pod 端点，<br>无身份绑定，Pod 可互换，<br>摘除后随机重路由  |
| 与下线关联  | 支持零丢包有序滚更：<br>readiness 降级时端点移除，<br>但有序 Pod 替换推进，宽限期内保留在途连接                               | 通用流量隔离：<br>降级即摘端点，支持并行滚更，<br>但无序，可能并发影响多 Pod                |
| 服务发现协同 | 与 Headless SRV 联动，<br>仅返回就绪 Pod 的稳定 DNS（如 pod-1.svc），<br>优先重连健康实例                         | 通过 ClusterIP Service 端点过滤，<br>支持负载均衡，<br>但无稳定身份，<br>客户端感知随机 |
| 适用场景   | 有状态游戏服/数据库：<br>确保主从拓扑下就绪 Pod 暴露，迁移时有序转移长连接                                                | 无状态 Web/API：<br>快速隔离故障 Pod，滚动更新时最小化中断，无需身份管理                |
| 配置影响   | partition/minReadySeconds 依赖 readiness 推进更新，<br>有序等待就绪再暴露端点                               | 仅影响 ReplicaSet 的可用副本计数，<br>无序扩展/缩容                          |

- Headless Service 使用场景：
  - 有状态 Pod 间通信：节点需要直接连接特别 Pod IP 而非负载均衡，Headless 通过 DNS 返回所有 Pod 记录实现有序访问
  - 客户端选择性连接：客户端需要解析服务明获取 Pod 的 FQDN（如：pod-0.game.svc）进行直连或重连，避免虚拟 IP 随机路由
  - 服务发现与 SRV 记录：结合命名端口生成 SRV 记录，便于应用如：HDFS/ES 动态协调访问目标 Pod，而非依赖 kube-proxy
- Readiness Probe 使用场景：
  - 流量健康隔离：滚动更新或节点维护时，Pod 进入 drain 模式后，readiness 失败，从 EndpointSlice 移除，避免新包路由到不稳定示例，实现零丢包
  - 动态可用性检查：长连接应用中，readiness 通过 HTTP/TCP 探针验证 Pod 是否处理流量，若崩溃或负载高则快速摘除，保障在途连接继续但拒绝新会话
  - 启动/恢复验证：Pod 启动后等待 readiness true 再加入端点集，适用于有初始化依赖的应用

| 维度               | Headless Service                       | Readiness Probe<br /> (Pod 是否准备好提供服务的标签，初始延迟之前的就绪状态默认 Failure，如果容器不提供就绪探针，默认 Success) |
|:---------------- |:-------------------------------------- |:------------------------------------------------------------------------------------- |
| 核心功能             | 服务发现与直连（返回 Pod IP/DNS 列表，无 Cluster IP） | 健康检查与流量过滤（决定 Pod 是否加入端点集）                                                             |
| 适用类型             | 有状态应用、Pod 间协商（如 StatefulSet 集群）        | 任意工作负载、动态路由（如 Deployment/StatefulSet 更新）                                              |
| 负载均衡             | 无，直接暴露 Pod，避免随机路由                      | 间接支持，通过 EndpointSlice 过滤后由 Service 均衡                                                 |
| 场景示例             | 游戏实例间同步、数据库主从发现                        | 零丢包下线、故障隔离                                                                            |
| 与 StatefulSet 关联 | 提供稳定 Pod 身份与 SRV 发现                    | 确保有序 Pod 只在就绪时暴露，协同下线                                                                 |

#### 2.8.10 Deployment 与 StatefulSet 对比

| 维度    | 无状态应用（Deployment）   | 有状态应用（StatefulSet） |
|:----- |:------------------- |:------------------ |
| 状态持久性 | 不在 Pod 内保留跨请求状态     | 需要跨请求/重启保持状态       |
| 实例身份  | 实例可互换，无固定身份         | 每个实例具稳定有序身份        |
| 网络标识  | 依赖 Service 负载均衡即可   | 稳定 DNS/有序命名便于点对点访问 |
| 数据存储  | 存于外部系统，独立于 Pod 生命周期 | 每副本独立 PVC，重调度不丢数据  |
| 扩缩容   | 并行、弹性强、简单           | 需按序扩缩，涉及数据复制/重配置   |
| 升级顺序  | 可并行滚动，无顺序依赖         | **有序滚动，确保一致性**     |
| 常用控制器 | Deployment          | StatefulSet        |
| 典型场景  | Web/API 前端、无会话计算    | 数据库、消息队列、分布式存储     |

| 维度     | Deployment                        | StatefulSet                         |
|:------ |:--------------------------------- |:----------------------------------- |
| 控制链路   | **Deployment → ReplicaSet → Pod** | **StatefulSet → Pod（+ 每副本 PVC）**    |
| 实例身份   | 副本可互换，无固定身份                       | 每副本有稳定有序身份（ordinal）                 |
| 网络标识   | 借助 Service 负载均衡即可                 | 需 Headless Service 以稳定 DNS/hostname |
| 存储     | 可用，但不保证“每副本专属+身份稳定”               | volumeClaimTemplates 为每副本创建 PVC     |
| 扩缩容/更新 | 并行滚更，弹性强                          | 多为有序滚更，强调一致性与安全                     |
| 适用场景   | 无状态服务                             | 有状态服务与拓扑敏感服务                        |

PersistentVolumeClaim（PVC）持久卷声明

从上边表格可以看到，Deployment 并不直接创建 Pod，而是**创建/管理 ReplicSet**，由 ReplicaSet 创建 Pod，这使得 “滚动发布/回滚” 可以通过快速切换/罗嗦不同版本的 ReplicaSet 实现

---

##### 2.8.10.1 StatefulSet Pod 崩溃与长连接处理

**场景**：StatefulSet 部署的游戏应用，某个 Pod 崩溃时，该 Pod 上的长连接用户会断开。

结论：当 StatefulSet 的某个 Pod 突发崩溃时，该 Pod 上的长连接会被**中断**。无法做到 “无断线无感知切换”。可行做法是用 “优雅终止” 将计划性变更的断线降到最短，并为 “非计划性崩溃” 设计快速重连和状态恢复，同时用 PDB 等机制降低并发中断概率，必要时引入游戏服务编排，如 Agones 来管理长连型对局生命周期。

**两种中断类型：**

- 计划性中断：如滚动升级，节点维护，手动删除，可以通过优雅终止让 Pod 停止接受新链接，保留宽限期限完成清理并断开，尽量不影响会话。
- 非计划性中断：如进程崩溃，OOM，节点故障，来不及优雅收尾，长连接会立即断开，必须依赖客户端快速重连和服务端状态恢复策略兜底。

**计划性 “优雅终止方案”**

- 后端应用层监听 K8S 的  SIGTERM 信号，并在应用内获取该信号，停止接受新链接，通知会话结束，保存必要状态，再有序关闭连接，利用 preStop 的钩子出发这一收尾逻辑。
- 设置 terminationGracePeriodSeconds（默认 30s）足够覆盖最长对局/收尾时间，确保 preStop 与清理能够在宽限时间完成，否则会被强制 SIGSKILL 终止。
- 使用 SeriviceMesh 或 SideCar，当收到终止信号时拒绝新请求，有助于在终止窗口内完成长连接的自然结束。

**非计划性崩溃的兜底方案**

- 设计客户端自动重连到可用示例，并在服务端提供快速重建对局/会话的机制（例如从外部状态恢复）
- 将关键会话信息与持久状态放在进程外部，例如存储在数据库

**降低 “同时掉线” 的集群级别保护**

- 为 StatefulSet 选用 **PodDistributionBudget（如：minAvailable:2）**，限制滚动升级/节点升级维护这类 “自愿中断” 在任意时刻仅影响少量样本，避免三副本里同时少于2个可用

**可落地配置与流程要点**

- Pod 模板中添加 preStop 并设置 terminationGracePeriodSeconds 时间。
- 配合服务网格/ SideCar 的优雅终止能力，在终止期间拒绝新链接，包括当前剩余链接退出
- 创建 PDB，设置 **minAvailable** 或 **maxUnavailable**，防止一次性驱逐过多

**在 StatefulSet 中优雅转移长连接用户到其他 Pod**

- 原则和思路：
  - 摘除新流量 + 等待老连接自然结束 + 引导重连
  - Pod 设置为 Terminating 后应从 Service 的可用端点摘除，不在接收新链接，同时允许存量连接在宽限期完成
  - 客户端重连逻辑和会话恢复逻辑，服务端提供快速恢复
- K8S 层配置：
  - 启用终止端点与优雅关停：设置 preStop 钩子和足够的 terminationGracePeriodSeconds 时间
  - 后端应用层接收 SIGTERM 后停止接收新连接并开始排空老连接，直到宽限期结束
  - 利用 Endpoints/EndpointSlice 的 “终止” 语义： Pod 进入终止态时，端点会被标记为非就绪带有 Terminating/Serving 条件，负载分发将停止把新连接导向到 Pod，实现 “只出不进”
  - Service 侧保持会话亲和：对长链接/会话型流量可设置 sessionAffinity: ClientIP，并按需配置超时，保证稳定粘性直至开始排空
  - 使用 PDB 控制同时中断规模：为 StatefulSet 设置 PodDistributionBudget，限制滚动更新/节点维护的最小可用副本
  - 分阶段更新与序控：StatefulSet 默认有序滚更，可通过 rollingUpdate.partition 分批更新，配合 minReadySeconds 让每次替换就绪稳定后再更新下一个
- 应用层改造：
  - SIGTERM 在后端业务代码逻辑中处理，客户端快速重连
  - 会话等信息持久化到数据库等外部服务

#### 故障排查与调试

**Pod 常见故障排查表：**

| 状态 | 常见原因 | 排查命令 | 解决方向 |
|------|----------|----------|----------|
| **CrashLoopBackOff** | 应用启动失败（配置错误、依赖缺失）| `kubectl describe pod <pod>` | 检查应用配置和依赖 |
| | 健康检查失败 | `kubectl logs <pod> --previous` | 调整探针配置或修复应用 |
| | OOMKilled（内存不足） | `kubectl top pod <pod>` | 增加内存 limits |
| **ImagePullBackOff** | 镜像不存在或 tag 错误 | `kubectl describe pod <pod>` | 确认镜像名称和 tag |
| | 镜像仓库认证失败 | `kubectl get secret <secret> -o yaml` | 检查 imagePullSecrets |
| | 网络问题 | `docker pull <image>` | 检查网络和防火墙 |
| **Pending** | 资源不足（CPU/内存） | `kubectl describe pod <pod>` | 扩容节点或调整 requests |
| | Taint/Toleration 不匹配 | `kubectl top nodes` | 添加 tolerations |
| | PVC 绑定失败 | `kubectl get pvc` | 检查 StorageClass 和 PV |
| | 节点选择器不匹配 | `kubectl get nodes --show-labels` | 调整 nodeSelector |
| **Evicted** | 节点资源压力（磁盘/内存/PID） | `kubectl describe node <node>` | 清理节点资源或扩容 |
| | 超过 ResourceQuota | `kubectl describe quota -n <ns>` | 调整配额或清理资源 |
| **Terminating** | Finalizer 未清理 | `kubectl get pod <pod> -o yaml` | 检查并移除 Finalizer |
| | preStop Hook 超时 | `kubectl describe pod <pod>` | 调整 terminationGracePeriodSeconds |

**调试工具：**

| 工具 | 用途 | 示例命令 |
|------|------|----------|
| `kubectl describe` | 查看资源详情和事件 | `kubectl describe pod <pod>` |
| `kubectl logs` | 查看容器日志 | `kubectl logs <pod> -c <container> --previous` |
| `kubectl exec` | 进入容器执行命令 | `kubectl exec -it <pod> -- /bin/sh` |
| `kubectl debug` | 调试 Pod（临时容器） | `kubectl debug <pod> -it --image=busybox` |
| `kubectl port-forward` | 端口转发 | `kubectl port-forward <pod> 8080:80` |
| `kubectl top` | 查看资源使用 | `kubectl top pod <pod>` |
| `kubectl get events` | 查看集群事件 | `kubectl get events --sort-by='.lastTimestamp'` |

**Metrics Server 部署：**

```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```

**关键监控指标：**

| 层级 | 指标 | 说明 |
|------|------|------|
| **节点** | CPU、内存、磁盘、网络 | 节点整体资源使用情况 |
| **Pod** | CPU、内存、重启次数 | Pod 资源消耗和稳定性 |
| **容器** | 资源使用率、OOM 次数 | 容器级别的资源监控 |

### 2.9 Informer 机制深度解析
#### 2.9.1 Informer 架构图

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Informer 完整架构                                 │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ┌─────────────┐                                                    │
│  │  API Server │                                                    │
│  │   (etcd)    │                                                    │
│  └──────┬──────┘                                                    │
│         │                                                           │
│         │ List + Watch                                              │
│         ▼                                                           │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │                      Reflector                              │    │
│  │  • ListAndWatch: 首次 List 全量 + 持续 Watch 增量              │    │
│  │  • ResourceVersion: 断点续传，避免重复                         │    │
│  │  • Backoff: 失败重试机制                                      │    │
│  └──────────────────────────┬──────────────────────────────────┘    │
│                             │                                       │
│                             │ Delta (Add/Update/Delete)             │
│                             ▼                                       │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │                      DeltaFIFO                              │    │
│  │  • FIFO 队列: 保证事件顺序                                     │    │
│  │  • Delta 类型: Added, Updated, Deleted, Replaced, Sync       │    │
│  │  • 去重: 相同 Key 的事件合并                                   │    │
│  └──────────────────────────┬──────────────────────────────────┘    │
│                             │                                       │
│                             │ Pop                                   │
│                             ▼                                       │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │                      Indexer (Store)                        │    │
│  │  • 本地缓存: 存储对象完整状态                                   │    │
│  │  • 索引: 支持多维度查询（Namespace、Label 等）                  │    │
│  │  • 线程安全: 读写锁保护                                        │    │
│  └──────────────────────────┬──────────────────────────────────┘    │
│                             │                                       │
│                             │ Event Handler                         │
│                             ▼                                       │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │                   ResourceEventHandler                      │    │
│  │  • OnAdd: 对象创建                                           │    │
│  │  • OnUpdate: 对象更新                                        │    │
│  │  • OnDelete: 对象删除                                        │    │
│  └──────────────────────────┬──────────────────────────────────┘    │
│                             │                                       │
│                             │ Enqueue Key                           │
│                             ▼                                       │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │                      Workqueue                              │    │
│  │  • 去重: 相同 Key 只入队一次                                   │    │
│  │  • 限速: 指数退避重试                                          │    │
│  │  • 延迟: 支持延迟入队                                          │    │
│  └──────────────────────────┬──────────────────────────────────┘    │
│                             │                                       │
│                             │ Get Key                               │
│                             ▼                                       │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │                    Reconcile Loop                           │    │
│  │  • 从 Workqueue 获取 Key                                     │    │
│  │  • 从 Indexer 读取对象                                        │    │
│  │  • 计算期望状态与实际状态差异                                   │    │
│  │  • 执行调谐操作                                               │    │
│  └─────────────────────────────────────────────────────────────┘    │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

#### 2.9.2 Informer 组件对比表

| 组件 | 职责 | 数据结构 | 线程安全 | 关键方法 |
|------|------|----------|----------|----------|
| **Reflector** | List+Watch API Server | - | ✅ | ListAndWatch(), Run() |
| **DeltaFIFO** | 事件队列 | FIFO + Map | ✅ | Add(), Pop(), Replace() |
| **Indexer** | 本地缓存 | ThreadSafeStore | ✅ | Add(), Get(), Index() |
| **Workqueue** | 任务队列 | Set + Queue | ✅ | Add(), Get(), Done() |
| **SharedInformer** | 多控制器共享 | Informer + Listeners | ✅ | AddEventHandler() |

#### 2.9.3 Informer 参数调优表

| 参数 | 默认值 | 调优建议 | 性能影响 | 说明 |
|------|--------|----------|----------|------|
| **ResyncPeriod** | 0（不重同步） | 30s-5m | 一致性 vs 性能 | 定期全量同步，防止事件丢失 |
| **WatchCacheSize** | 100 | 1000-10000 | 内存 vs Watch 性能 | API Server 端 Watch 缓存大小 |
| **QPS** | 5 | 20-100 | API Server 压力 | 客户端请求限速 |
| **Burst** | 10 | 50-200 | 突发请求能力 | 突发请求数量 |
| **WorkerCount** | 1 | 2-10 | 并发处理能力 | Reconcile 并发 Worker 数 |
| **MaxRetries** | 5 | 3-10 | 错误恢复 | 失败重试次数 |
| **RateLimiter.BaseDelay** | 5ms | 1ms-100ms | 重试延迟 | 指数退避基础延迟 |
| **RateLimiter.MaxDelay** | 1000s | 60s-300s | 最大重试间隔 | 指数退避最大延迟 |

#### 2.9.4 Informer 问题定位表

| 问题现象 | 可能原因 | 排查命令 | 解决方案 |
|----------|----------|----------|----------|
| **控制器启动慢** | List 全量数据过多 | 监控 List 耗时 | 增加 API Server 资源、分页 List |
| **事件延迟高** | Workqueue 积压 | 监控队列长度 | 增加 Worker 数、优化 Reconcile |
| **内存占用高** | Indexer 缓存过大 | `kubectl top pod` | 减少 Watch 资源类型、增加内存 |
| **Watch 断开重连** | 网络不稳定 | 检查网络日志 | 优化网络、增加重试 |
| **重复处理** | ResyncPeriod 过短 | 检查 Reconcile 日志 | 增加 ResyncPeriod |
| **API Server 压力大** | QPS 过高 | API Server 指标 | 降低 QPS、使用 SharedInformer |
| **对象状态不一致** | 缓存未更新 | 对比 API Server 数据 | 检查 Reflector 状态 |
| **控制器 OOM** | 大量对象缓存 | 内存分析 | 限制 Watch 范围、增加内存 |

#### 2.9.5 SharedInformer vs Informer

```
┌─────────────────────────────────────────────────────────────────────┐
│              SharedInformer 共享机制                                 │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  普通 Informer（每个控制器独立 Watch）：                                 │
│                                                                     │
│  ┌──────────┐     ┌──────────┐     ┌──────────┐                     │
│  │Controller│     │Controller│     │Controller│                     │
│  │    A     │     │    B     │     │    C     │                     │
│  └────┬─────┘     └────┬─────┘     └────┬─────┘                     │
│       │                │                │                           │
│       │ Watch          │ Watch          │ Watch                     │
│       ▼                ▼                ▼                           │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │                      API Server                             │    │
│  │              3 个独立 Watch 连接（资源浪费）                    │    │
│  └─────────────────────────────────────────────────────────────┘    │
│                                                                     │
│  SharedInformer（共享 Watch）：                                       │
│                                                                     │
│  ┌──────────┐     ┌──────────┐     ┌──────────┐                     │
│  │Controller│     │Controller│     │Controller│                     │
│  │    A     │     │    B     │     │    C     │                     │
│  └────┬─────┘     └────┬─────┘     └────┬─────┘                     │
│       │                │                │                           │
│       │ Handler        │ Handler        │ Handler                   │
│       ▼                ▼                ▼                           │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │                   SharedInformer                            │    │
│  │              1 个 Watch + 多个 EventHandler                  │    │
│  └──────────────────────────┬──────────────────────────────────┘    │
│                             │                                       │
│                             │ 1 个 Watch                            │
│                             ▼                                       │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │                      API Server                             │    │
│  │              资源节省 66%                                    │    │
│  └─────────────────────────────────────────────────────────────┘    │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

| 维度 | Informer | SharedInformer |
|------|----------|----------------|
| **Watch 连接** | 每个控制器独立 | 共享一个连接 |
| **内存占用** | 高（重复缓存） | 低（共享缓存） |
| **API Server 压力** | 高 | 低 |
| **适用场景** | 单控制器 | 多控制器共享资源 |
| **使用方式** | NewInformer() | SharedInformerFactory |

---

## 3. 容器运行时架构
### 3.1 容器运行时演进历史

```
┌─────────────────────────────────────────────────────────┐
│                容器运行时演进时间线                        │
│                                                         │
│  2013 ──▶ 2015 ──▶ 2016 ──▶ 2017 ──▶ 2020 ──▶ 2022      │
│    │        │        │        │        │        │       │
│  Docker   runC    containerd  CRI-O   Docker   K8S      │
│  诞生     OCI标准  从Docker    专为K8S   弃用     1.24     │
│           实现     拆分出来     设计      警告    移除      │
│                                               dockershim│
└─────────────────────────────────────────────────────────┘
```

### 3.2 OCI 标准规范

| 规范 | 说明 | 定义内容 |
|------|------|---------|
| **Runtime Spec** | 运行时规范 | 容器生命周期、配置格式 |
| **Image Spec** | 镜像规范 | 镜像格式、分层结构 |
| **Distribution Spec** | 分发规范 | 镜像仓库 API |

### 3.3 containerd 架构详解

```
┌─────────────────────────────────────────────────────────┐
│                  containerd 架构                         │
│                                                         │
│  ┌─────────────────────────────────────────────────┐    │
│  │                 containerd                      │    │
│  │  ┌──────────┐ ┌──────────┐ ┌──────────┐         │    │
│  │  │ Content  │ │ Snapshot │ │  Image   │         │    │
│  │  │  Store   │ │  Store   │ │  Store   │         │    │
│  │  └──────────┘ └──────────┘ └──────────┘         │    │
│  │  ┌──────────┐ ┌──────────┐ ┌──────────┐         │    │
│  │  │Container │ │  Task    │ │  Event   │         │    │
│  │  │ Service  │ │ Service  │ │ Service  │         │    │
│  │  └──────────┘ └──────────┘ └──────────┘         │    │
│  └─────────────────────────────────────────────────┘    │
│                         │                               │
│                         ▼                               │
│  ┌─────────────────────────────────────────────────┐    │
│  │              containerd-shim                    │    │
│  │         （每个容器一个 shim 进程）                 │    │
│  └─────────────────────────────────────────────────┘    │
│                         │                               │
│                         ▼                               │
│  ┌─────────────────────────────────────────────────┐    │
│  │                    runc                         │    │
│  │            （OCI 运行时实现）                     │    │
│  └─────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────┘
```

**containerd 核心组件说明：**

| 组件 | 职责 | 说明 |
|------|------|------|
| **Content Store** | 内容寻址存储 | 按 SHA256 哈希存储镜像层 blob，支持去重 |
| **Snapshot Store** | 文件系统快照 | 管理容器的 rootfs，支持 OverlayFS/DevMapper |
| **Image Store** | 镜像元数据 | 存储镜像 manifest、config、层信息 |
| **Container Service** | 容器元数据 | 管理容器配置（不是运行状态） |
| **Task Service** | 容器进程管理 | 管理运行中的容器（启动、停止、exec） |
| **Event Service** | 事件发布订阅 | 容器生命周期事件通知 |

**containerd-shim 的作用：**
- **解耦生命周期**：shim 作为容器进程的父进程，containerd 重启不影响运行中的容器
- **保持 stdio**：shim 保持容器的 stdin/stdout/stderr 连接
- **报告退出状态**：容器退出后，shim 向 containerd 报告退出码
- **每容器一个**：每个容器有独立的 shim 进程，隔离故障影响

**runc 的作用：**
- **OCI 标准实现**：按照 OCI Runtime Spec 创建容器
- **创建容器进程**：调用 clone() 创建新进程，配置 Namespace 和 Cgroups
- **执行后退出**：runc 创建容器后退出，shim 接管容器进程

### 3.4 容器运行时对比表

| 运行时 | 类型 | 特点 | 适用场景 |
|--------|------|------|---------|
| **containerd** | 高级运行时 | Docker 拆分出来，功能完整 | 生产环境首选 |
| **CRI-O** | 高级运行时 | 专为 K8S 设计，轻量 | OpenShift 默认 |
| **runc** | 低级运行时 | OCI 标准实现 | 默认容器执行器 |
| **gVisor** | 低级运行时 | 用户态内核，增强隔离 | 多租户/安全敏感 |
| **Kata Containers** | 低级运行时 | 轻量级 VM，硬件隔离 | 强隔离需求 |

### 3.5 低级运行时对比

```
┌─────────────────────────────────────────────────────────┐
│                 低级运行时隔离对比                         │
│                                                         │
│  runc（默认）：                                            │
│  ┌─────────┐                                            │
│  │ 容器进程 │ ──▶ 系统调用 ──▶ Host 内核                   │
│  └─────────┘     （直接调用）                             │
│                                                         │
│  gVisor：                                               │
│  ┌─────────┐                                            │
│  │ 容器进程 │ ──▶ Sentry ──▶ 有限系统调用 ──▶ Host 内核     │
│  └─────────┘   （用户态内核）                             │
│                                                         │
│  Kata Containers：                                      │
│  ┌─────────┐                                            │
│  │ 容器进程 │ ──▶ Guest 内核 ──▶ Hypervisor ──▶ Host      │
│  └─────────┘   （轻量级 VM）                              │
└─────────────────────────────────────────────────────────┘
```

**低级运行时隔离原理详解：**

| 运行时 | 隔离机制 | 系统调用处理 | 安全边界 |
|--------|----------|--------------|----------|
| **runc** | Namespace + Cgroups | 直接调用 Host 内核 | 内核级隔离（共享内核） |
| **gVisor** | 用户态内核（Sentry） | Sentry 拦截并模拟 | 应用级隔离（减少内核攻击面） |
| **Kata** | 轻量级 VM（QEMU/Firecracker） | Guest 内核处理 | 硬件级隔离（独立内核） |

**选型建议：**
- **runc**：默认选择，性能最优，适合信任的工作负载
- **gVisor**：多租户场景，需要增强隔离但不想牺牲太多性能
- **Kata**：强安全要求，可接受 VM 级别的启动延迟和资源开销

| 运行时 | 隔离级别 | 性能开销 | 兼容性 |
|--------|---------|---------|--------|
| runc | 内核级 | ~0% | 100% |
| gVisor | 用户态内核 | 5-30% | 90%+ |
| Kata | 硬件级 | 10-20% | 95%+ |

### 3.6 containerd 详细架构

```
┌─────────────────────────────────────────────────────────────────────┐
│                    containerd 6 层架构详解                            │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Layer 6: gRPC API（对外接口）                                        │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │  • containers.v1    → 容器 CRUD                             │    │
│  │  • images.v1        → 镜像管理                               │    │
│  │  • tasks.v1         → 任务（运行中容器）管理                   │    │
│  │  • namespaces.v1    → 命名空间隔离                           │    │
│  │  • content.v1       → 内容存储                               │    │
│  │  • snapshots.v1     → 快照管理                               │    │
│  └─────────────────────────────────────────────────────────────┘    │
│                              │                                      │
│  Layer 5: Services（服务层）                                         │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │  • Container Service  → 容器元数据管理                        │    │
│  │  • Image Service      → 镜像拉取、解压                        │    │
│  │  • Snapshot Service   → 文件系统快照                          │    │
│  │  • Task Service       → 容器进程管理                          │    │
│  │  • Event Service      → 事件发布订阅                          │    │
│  └─────────────────────────────────────────────────────────────┘    │
│                              │                                      │
│  Layer 4: Metadata Store（元数据存储）                                │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │  • BoltDB 嵌入式数据库                                        │    │
│  │  • 存储容器、镜像、快照元数据                                   │    │
│  │  • 路径: /var/lib/containerd/io.containerd.metadata.v1.bolt │    │
│  └─────────────────────────────────────────────────────────────┘    │
│                              │                                      │
│  Layer 3: Content Store（内容存储）                                   │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │  • Content-addressable 存储（按 SHA256 寻址）                  │    │
│  │  • 存储镜像层 blob                                            │    │
│  │  • 路径: /var/lib/containerd/io.containerd.content.v1.content│    │
│  └─────────────────────────────────────────────────────────────┘    │
│                              │                                      │
│  Layer 2: Snapshotter（快照器）                                       │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │  • overlayfs（默认，推荐）                                    │    │
│  │  • native（简单复制）                                         │    │
│  │  • devmapper（块设备）                                        │    │
│  │  • 路径: /var/lib/containerd/io.containerd.snapshotter.v1.*  │    │
│  └─────────────────────────────────────────────────────────────┘    │
│                              │                                      │
│  Layer 1: Runtime（运行时层）                                         │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │  containerd-shim-runc-v2                                    │    │
│  │  ├─ 每个容器一个 shim 进程                                     │    │
│  │  ├─ 解耦 containerd 和容器生命周期                             │    │
│  │  └─ containerd 重启不影响运行中容器                            │    │
│  │                         │                                   │    │
│  │                         ▼                                   │    │
│  │  runc（OCI 运行时）                                          │    │
│  │  └─ 创建容器进程（clone + namespace + cgroups）               │    │
│  └─────────────────────────────────────────────────────────────┘    │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

**containerd 6 层架构说明：**

| 层级 | 名称 | 职责 | 关键技术 |
|------|------|------|----------|
| **Layer 6** | gRPC API | 对外提供服务接口 | gRPC、Protobuf |
| **Layer 5** | Services | 业务逻辑实现 | 容器/镜像/任务管理 |
| **Layer 4** | Metadata Store | 元数据持久化 | BoltDB 嵌入式数据库 |
| **Layer 3** | Content Store | 镜像层存储 | Content-addressable（SHA256） |
| **Layer 2** | Snapshotter | 容器文件系统 | OverlayFS/DevMapper |
| **Layer 1** | Runtime | 容器进程管理 | shim + runc |

**为什么需要 containerd-shim？**

```
问题：如果 containerd 直接管理容器进程
┌─────────────┐     ┌─────────────┐
│ containerd  │────▶│  容器进程    │
└─────────────┘     └─────────────┘
      │
      ▼
containerd 重启 → 容器进程成为孤儿进程 → 容器异常

解决：引入 shim 作为中间层
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│ containerd  │────▶│    shim     │────▶│  容器进程    │
└─────────────┘     └─────────────┘     └─────────────┘
      │                   │
      ▼                   ▼
containerd 重启    shim 继续运行 → 容器不受影响
```

### 3.7 CRI-O 架构详解

```
┌─────────────────────────────────────────────────────────────────────┐
│                    CRI-O 架构                                        │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │                      kubelet                                │    │
│  │                         │                                   │    │
│  │                    CRI gRPC 调用                             │    │
│  │                         │                                   │    │
│  └─────────────────────────┼───────────────────────────────────┘    │
│                            ▼                                        │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │                      CRI-O                                  │    │
│  │  ┌──────────────────────────────────────────────────────┐   │    │
│  │  │  CRI 实现                                            │   │    │
│  │  │  ├─ RuntimeService  → 容器生命周期管理                 │   │    │
│  │  │  └─ ImageService    → 镜像管理                        │   │    │
│  │  └──────────────────────────────────────────────────────┘   │    │
│  │  ┌──────────────────────────────────────────────────────┐   │    │
│  │  │  核心组件                                             │   │    │
│  │  │  ├─ containers/image → 镜像拉取                       │   │    │
│  │  │  ├─ containers/storage → 存储管理                     │   │    │
│  │  │  └─ CNI              → 网络配置                       │   │    │
│  │  └──────────────────────────────────────────────────────┘   │    │
│  └─────────────────────────────────────────────────────────────┘    │
│                            │                                        │
│                            ▼                                        │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │                    conmon（容器监控器）                       │    │
│  │  ├─ 每个容器一个 conmon 进程                                  │    │
│  │  ├─ 处理容器日志                                              │    │
│  │  └─ 监控容器退出状态                                           │    │
│  └─────────────────────────────────────────────────────────────┘    │
│                            │                                        │
│                            ▼                                        │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │                    OCI Runtime（runc/crun）                  │    │
│  └─────────────────────────────────────────────────────────────┘    │
│                                                                     │
│  CRI-O vs containerd 对比：                                          │
│  • CRI-O: 专为 K8S 设计，更轻量，无额外功能                              │
│  • containerd: 功能更丰富，支持非 K8S 场景                              │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

**CRI-O 架构核心组件说明：**

CRI-O 是 Red Hat 主导开发的轻量级容器运行时，专为 Kubernetes 设计。与 containerd 不同，CRI-O 不追求通用性，而是专注于做好一件事：为 Kubernetes 提供最小化的 CRI 实现。

| 组件 | 职责 | 与 containerd 对比 |
|------|------|-------------------|
| **CRI-O 守护进程** | 实现 CRI 接口，处理 kubelet 请求 | 类似 containerd，但更轻量 |
| **containers/image** | 镜像拉取和管理 | 独立库，非 containerd 内置 |
| **containers/storage** | 存储管理（OverlayFS） | 独立库，支持多种存储驱动 |
| **conmon** | 容器监控器，每容器一个进程 | 类似 containerd-shim 的作用 |
| **OCI Runtime** | 实际创建容器（runc/crun） | 与 containerd 相同 |

**CRI-O 设计哲学：**

1. **最小化原则**：只实现 Kubernetes 需要的功能，不支持 `docker build`、`docker-compose` 等
2. **稳定性优先**：版本号与 Kubernetes 版本对齐（如 CRI-O 1.28 对应 K8S 1.28）
3. **安全性**：默认启用 SELinux、Seccomp，支持 User Namespace

**conmon vs containerd-shim 对比：**

| 维度 | conmon（CRI-O） | containerd-shim |
|------|-----------------|-----------------|
| **进程模型** | 每容器一个 conmon | 每容器一个 shim |
| **主要职责** | 日志收集、退出状态监控 | 容器生命周期管理、stdio 转发 |
| **与主进程关系** | CRI-O 重启后 conmon 继续运行 | containerd 重启后 shim 继续运行 |
| **实现语言** | C 语言（更轻量） | Go 语言 |

### 3.8 容器运行时详细对比

| 维度 | Docker | containerd | CRI-O | Podman |
|------|--------|------------|-------|--------|
| **定位** | 完整容器平台 | 容器运行时 | K8S 专用运行时 | 无守护进程 |
| **守护进程** | dockerd | containerd | crio | 无 |
| **CRI 支持** | 通过 cri-dockerd | 原生支持 | 原生支持 | 通过 podman-cri |
| **镜像构建** | ✅ docker build | 🔴 不支持 | 🔴 不支持 | ✅ podman build |
| **Compose** | ✅ docker-compose | 🔴 不支持 | 🔴 不支持 | ✅ podman-compose |
| **Rootless** | ⚠️ 实验性 | ⚠️ 实验性 | ⚠️ 实验性 | ✅ 原生支持 |
| **资源占用** | 高（~100MB） | 中（~50MB） | 低（~30MB） | 最低 |
| **K8S 推荐** | 🔴 已弃用 | ✅ 推荐 | ✅ 推荐 | ⚠️ 边缘场景 |
| **CLI 工具** | docker | ctr/crictl | crictl | podman |
| **日志驱动** | 多种 | 有限 | 有限 | 多种 |

### 3.9 containerd 参数调优表

| 参数 | 默认值 | 调优建议 | 性能影响 | 说明 |
|------|--------|----------|----------|------|
| `max_concurrent_downloads` | 3 | 8-10 | 镜像拉取速度 | 并发下载镜像层数 |
| `max_concurrent_uploads` | 3 | 8-10 | 镜像推送速度 | 并发上传镜像层数 |
| `snapshotter` | overlayfs | overlayfs | 性能最优 | 快照器类型 |
| `default_runtime_name` | runc | runc | - | 默认运行时 |
| `discard_unpacked_layers` | false | true | 节省磁盘 | 解压后删除原始层 |
| `oom_score` | 0 | -999 | 稳定性 | OOM 优先级 |

**containerd 配置示例（/etc/containerd/config.toml）：**

```toml
version = 2

[plugins."io.containerd.grpc.v1.cri"]
  sandbox_image = "registry.k8s.io/pause:3.9"
  max_concurrent_downloads = 10
  max_concurrent_uploads = 10

[plugins."io.containerd.grpc.v1.cri".containerd]
  snapshotter = "overlayfs"
  default_runtime_name = "runc"

[plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc]
  runtime_type = "io.containerd.runc.v2"

[plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
  SystemdCgroup = true

[plugins."io.containerd.grpc.v1.cri".registry]
  config_path = "/etc/containerd/certs.d"
```

### 3.10 容器运行时问题定位表

| 问题 | 症状 | 原因 | 排查命令 | 解决方案 |
|------|------|------|----------|----------|
| 容器创建失败 | CrashLoopBackOff | OCI 运行时错误 | `crictl logs <container-id>` | 检查 runc 日志 |
| 镜像拉取慢 | ImagePullBackOff | 并发下载数低 | `crictl images` | 增加 max_concurrent_downloads |
| 容器启动慢 | Pod 启动延迟 | Snapshotter 性能 | `ctr snapshot ls` | 使用 overlayfs |
| containerd 崩溃 | 节点 NotReady | 内存不足 | `journalctl -u containerd` | 增加内存或调整 oom_score |
| 镜像空间不足 | 磁盘满 | 未清理旧镜像 | `crictl rmi --prune` | 配置镜像 GC |
| CRI 连接失败 | kubelet 报错 | Socket 路径错误 | `ls /run/containerd/containerd.sock` | 检查 containerd 状态 |
 
### 3.11 常用运行时命令

```bash
# ===== containerd 命令（ctr）=====
ctr images ls                          # 列出镜像
ctr containers ls                      # 列出容器
ctr tasks ls                           # 列出运行中任务
ctr namespaces ls                      # 列出命名空间

# ===== CRI 命令（crictl）=====
crictl images                          # 列出镜像
crictl ps -a                           # 列出所有容器
crictl pods                            # 列出 Pod
crictl logs <container-id>             # 查看容器日志
crictl exec -it <container-id> sh      # 进入容器
crictl stats                           # 容器资源统计
crictl rmi --prune                     # 清理未使用镜像

# ===== 调试命令 =====
journalctl -u containerd -f            # 查看 containerd 日志
systemctl status containerd            # 检查服务状态
containerd config dump                 # 导出当前配置
```

---

## 4. Kubernetes 核心组件详解
### 4.1 etcd 架构详解

etcd 是 Kubernetes 的核心存储组件，所有集群状态数据都存储在 etcd 中。

**etcd 核心特性：**
- 分布式键值存储
- 强一致性（基于 Raft 协议）
- 支持 Watch 机制
- 支持 MVCC（多版本并发控制）

#### 4.1.1 Raft 共识算法

```
┌─────────────────────────────────────────────────────────┐
│                    Raft 集群状态                         │
│                                                         │
│  ┌──────────┐    ┌──────────┐    ┌──────────┐           │
│  │  Leader  │    │ Follower │    │ Follower │           │
│  │  etcd-0  │───▶│  etcd-1  │    │  etcd-2  │           │
│  └──────────┘    └──────────┘    └──────────┘           │
│       │               ▲               ▲                 │
│       └───────────────┴───────────────┘                 │
│              日志复制（Log Replication）                  │
└─────────────────────────────────────────────────────────┘
```

**Raft 三种角色：**
- **Leader**：处理所有客户端请求，复制日志到 Follower
- **Follower**：被动接收 Leader 的日志复制
- **Candidate**：Leader 选举时的临时状态


**Leader 选举流程：**
1. Follower 在 election timeout 内未收到 Leader 心跳
2. Follower 转为 Candidate，发起投票请求
3. 获得多数票（N/2+1）后成为 Leader
4. Leader 定期发送心跳维持领导地位

**Raft 日志复制详细流程（Log Replication）：**

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Raft 日志复制完整流程                               │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  1. 客户端写入请求                                                    │
│     Client ──PUT /registry/pods/nginx──▶ Leader                     │
│                                                                     │
│  2. Leader 追加日志条目                                               │
│     ┌─────────────────────────────────────────────────────────┐     │
│     │ Log Entry:                                              │     │
│     │   Index: 101                                            │     │
│     │   Term: 5                                               │     │
│     │   Command: PUT /registry/pods/nginx {data}              │     │
│     └─────────────────────────────────────────────────────────┘     │
│                                                                     │
│  3. Leader 发送 AppendEntries RPC 到所有 Follower                     │
│     ┌─────────────────────────────────────────────────────────┐     │
│     │ AppendEntries RPC:                                      │     │
│     │   term: 5              (Leader 当前任期)                 │     │
│     │   leaderId: etcd-0                                      │     │
│     │   prevLogIndex: 100    (前一条日志索引)                   │     │
│     │   prevLogTerm: 5       (前一条日志任期)                   │     │
│     │   entries: [{101, 5, PUT...}]  (新日志条目)              │     │
│     │   leaderCommit: 99     (Leader 已提交索引)               │     │
│     └─────────────────────────────────────────────────────────┘     │
│                                                                     │
│  4. Follower 验证并追加日志                                           │
│     ┌─────────────────────────────────────────────────────────┐     │
│     │ Follower 检查:                                          │     │
│     │   ① term >= currentTerm ?                              │     │
│     │   ② log[prevLogIndex].term == prevLogTerm ?            │     │
│     │      (Log Matching Property - 日志匹配属性)               │     │
│     │   如果 ② 失败: 返回 false，Leader 回退 prevLogIndex       │     │
│     │   如果 ② 成功: 追加 entries，更新 commitIndex             │     │
│     └─────────────────────────────────────────────────────────┘     │
│                                                                     │
│  5. Leader 收到多数派确认后提交                                        │
│     ┌─────────────────────────────────────────────────────────┐     │
│     │ 提交条件（Commit）:                                       │     │
│     │   ① 日志已复制到多数派节点（N/2+1）                         │     │
│     │   ② 日志的 term == Leader 当前 term                      │     │
│     │   满足后: commitIndex = 101                              │     │
│     └─────────────────────────────────────────────────────────┘     │
│                                                                     │
│  6. 应用到状态机                                                      │
│     Leader: apply log[101] to BoltDB                                │
│     Follower: 下次 AppendEntries 时通过 leaderCommit 得知提交          │
│               然后 apply log[101] to BoltDB                          │
│                                                                     │
│  7. 返回客户端                                                        │
│     Leader ──200 OK──▶ Client                                       │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

**Raft 安全性保证（Safety Properties）：**

| 属性 | 说明 | 保证机制 |
|------|------|----------|
| **Election Safety** | 每个 term 最多一个 Leader | 多数派投票 + term 单调递增 |
| **Leader Append-Only** | Leader 不会覆盖或删除日志 | 只追加新日志 |
| **Log Matching** | 相同 index+term 的日志内容相同 | AppendEntries 的 prevLogIndex/prevLogTerm 检查 |
| **Leader Completeness** | 已提交的日志一定在新 Leader 中 | 投票时检查候选人日志是否足够新 |
| **State Machine Safety** | 所有节点在相同 index 应用相同命令 | 只应用已提交的日志 |

**日志冲突处理：**
```
场景：Follower 日志与 Leader 不一致

Leader 日志:  [1,1] [2,1] [3,2] [4,2] [5,3]
Follower 日志: [1,1] [2,1] [3,2] [4,3]  ← index 4 的 term 不同

处理流程：
1. Leader 发送 AppendEntries(prevLogIndex=4, prevLogTerm=2)
2. Follower 检查: log[4].term=3 ≠ prevLogTerm=2，返回 false
3. Leader 回退: prevLogIndex=3
4. Leader 发送 AppendEntries(prevLogIndex=3, prevLogTerm=2)
5. Follower 检查: log[3].term=2 == prevLogTerm=2，成功
6. Follower 删除 index 4 及之后的日志，追加 Leader 的日志
7. 最终一致: [1,1] [2,1] [3,2] [4,2] [5,3]
```

#### 4.1.2 etcd 数据模型

```
┌─────────────────────────────────────────────────────────┐
│                    etcd 数据模型                         │
│                                                         │
│  Key: /registry/pods/default/nginx-pod                  │
│  Value: {Pod 对象的 JSON/Protobuf 序列化数据}             │
│  Revision: 12345 (全局递增版本号)                         │
│  ModRevision: 12340 (最后修改版本)                        │
│  CreateRevision: 12300 (创建版本)                        │
└─────────────────────────────────────────────────────────┘
```

**K8S 资源在 etcd 中的存储路径：**

| 资源类型 | etcd Key 路径 |
|---------|---------------|
| Pod | `/registry/pods/{namespace}/{name}` |
| Service | `/registry/services/specs/{namespace}/{name}` |
| Deployment | `/registry/deployments/{namespace}/{name}` |
| ConfigMap | `/registry/configmaps/{namespace}/{name}` |
| Secret | `/registry/secrets/{namespace}/{name}` |

**BoltDB 存储引擎详解：**

etcd 使用 BoltDB 作为底层存储引擎，BoltDB 是一个纯 Go 实现的嵌入式 KV 数据库。

```
┌─────────────────────────────────────────────────────────────────────┐
│                    BoltDB 存储结构                                   │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  BoltDB 文件结构（db 文件）:                                           │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │ Meta Page 0    │ Meta Page 1    │ Freelist Page │ Data Pages│    │
│  │ (元数据页)      │ (元数据页备份)    │ (空闲页列表)   │ (B+Tree)  │    │
│  └─────────────────────────────────────────────────────────────┘    │
│                                                                     │
│  B+Tree 结构:                                                        │
│                    ┌─────────────┐                                  │
│                    │ Branch Page │ (内部节点)                        │
│                    │ [key1|key2] │                                  │
│                    └──────┬──────┘                                  │
│              ┌───────────┼───────────┐                              │
│              ▼           ▼           ▼                              │
│         ┌────────┐  ┌────────┐  ┌────────┐                          │
│         │ Leaf   │  │ Leaf   │  │ Leaf   │ (叶子节点)                │
│         │ Page   │  │ Page   │  │ Page   │                          │
│         │[k:v]   │  │[k:v]   │  │[k:v]   │                          │
│         └────────┘  └────────┘  └────────┘                          │
│                                                                     │
│  etcd 使用两个 Bucket:                                               │
│  • key bucket: 存储 revision → {key, value, create_rev, mod_rev}    │
│  • meta bucket: 存储元数据（如 consistent_index）                     │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

**MVCC 版本链实现：**

```
┌─────────────────────────────────────────────────────────────────────┐
│                    etcd MVCC 版本链                                  │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Revision 结构:                                                      │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │ revision = {main: int64, sub: int64}                        │    │
│  │   • main: 全局递增，每次事务 +1                                │    │
│  │   • sub: 事务内操作序号，从 0 开始                              │    │
│  │   例: revision{main:100, sub:0} 表示第 100 次事务的第 1 个操作. │    │
│  └─────────────────────────────────────────────────────────────┘    │
│                                                                     │
│  keyIndex 结构（内存索引）:                                            │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │ key: "/registry/pods/default/nginx"                         │    │
│  │ generations:                                                │    │
│  │  [0]: {created: rev{90,0}, modified: [rev{90,0}, rev{95,0}]}│    │
│  │  [1]: {created: rev{100,0}, modified: [rev{100,0}]}  ← 当前  │    │
│  │                                                             │    │
│  │ generation 切换时机:                                         │    │
│  │   • 删除 key 后重新创建 → 新 generation                        │    │
│  │   • Compaction 清理旧 generation                             │    │
│  └─────────────────────────────────────────────────────────────┘    │
│                                                                     │
│  版本链查询流程:                                                       │
│  1. 查询 key="/registry/pods/default/nginx" at revision=95           │
│  2. 在 keyIndex 中找到 key，定位到 generation[0]                       │
│  3. 在 generation[0].modified 中二分查找 <= 95 的最大 revision         │
│  4. 找到 rev{95,0}，从 BoltDB 读取对应 value                           │
│                                                                     │
│  Watch 如何利用 MVCC:                                                │
│  • Watch 从指定 revision 开始                                         │
│  • 遍历 revision 之后的所有变更                                        │
│  • 支持历史回放（只要未被 Compaction 清理）                              │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

**Compaction（压缩）机制：**

| 操作 | 说明 | 影响 |
|------|------|------|
| **Compact** | 删除指定 revision 之前的历史版本 | 释放 BoltDB 空间（标记删除） |
| **Defrag** | 整理 BoltDB 文件，回收空间 | 实际释放磁盘空间 |

```bash
# 获取当前 revision
rev=$(etcdctl endpoint status --write-out=json | jq -r '.[0].Status.header.revision')

# 压缩到指定 revision（保留最近 1000 个版本）
etcdctl compact $((rev - 1000))

# 碎片整理（需要在每个节点执行）
etcdctl defrag --cluster
```

#### 4.1.3 Watch 机制

```
┌─────────────────────────────────────────────────────────┐
│                    Watch 工作流程                        │
│                                                         │
│  API Server                                             │
│      │                                                  │
│      │ Watch /registry/pods/*                           │
│      ▼                                                  │
│  ┌──────────┐                                           │
│  │   etcd   │ ──▶ 事件流（Event Stream）                  │
│  └──────────┘                                           │
│      │                                                  │
│      │ PUT /registry/pods/default/nginx                 │
│      ▼                                                  │
│  生成 WatchEvent:                                        │
│  • Type: PUT/DELETE                                     │
│  • Key: /registry/pods/default/nginx                    │
│  • Value: {新的 Pod 数据}                                │
│  • PrevValue: {旧的 Pod 数据}                            │
│  • ModRevision: 12346                                   │
└─────────────────────────────────────────────────────────┘
```

**Watch 特性：**
- 增量同步：只推送变更事件
- 断点续传：通过 Revision 恢复 Watch
- 历史回放：可从指定 Revision 开始 Watch

#### 4.1.4 分布式协调系统横向对比

| 维度 | etcd | ZooKeeper | Consul |
|------|------|-----------|--------|
| **共识协议** | Raft | ZAB (类 Paxos) | Raft |
| **数据模型** | 扁平 KV + MVCC | 层级树 + 临时节点 | KV + 服务目录 |
| **Watch 机制** | 基于 Revision 的增量 Watch | 基于 Watcher 的一次性通知 | 长轮询 Blocking Query |
| **一致性** | 线性一致性 | 顺序一致性 | 可选（默认最终一致） |
| **性能（写）** | ~10K ops/s | ~10K ops/s | ~5K ops/s |
| **性能（读）** | ~50K ops/s | ~100K ops/s | ~20K ops/s |
| **K8S 集成** | ✅ 原生支持 | 🔴 需要适配 | 🔴 需要适配 |
| **运维复杂度** | 中 | 高 | 低 |
| **多数据中心** | 🔴 不支持 | 🔴 不支持 | ✅ 原生支持 |
| **服务发现** | 🔴 需要额外实现 | 🔴 需要额外实现 | ✅ 原生支持 |
| **健康检查** | 🔴 不支持 | 🔴 不支持 | ✅ 原生支持 |

**选型建议：**
- **Kubernetes 集群**：etcd（原生支持，无需额外适配）
- **传统 Java 生态**：ZooKeeper（Kafka、Hadoop 等依赖）
- **服务发现 + 配置中心**：Consul（功能全面，多数据中心）

#### 4.1.5 etcd 参数调优表

| 参数 | 默认值 | 调优建议 | 性能影响 | 说明 |
|------|--------|----------|----------|------|
| `--quota-backend-bytes` | 2GB | 8GB（大集群） | 存储容量 | 超过后 etcd 只读，需要 compact |
| `--snapshot-count` | 100000 | 50000（SSD） | 恢复速度 | 快照频率，值越小恢复越快 |
| `--heartbeat-interval` | 100ms | 100-500ms | 选举稳定性 | 心跳间隔，网络延迟高时增大 |
| `--election-timeout` | 1000ms | 1000-5000ms | 选举速度 | 选举超时，建议 10x heartbeat |
| `--max-request-bytes` | 1.5MB | 10MB（大对象） | 请求大小 | 单请求最大字节数 |
| `--auto-compaction-retention` | 0（禁用） | 1h | 存储空间 | 自动压缩保留时间 |
| `--auto-compaction-mode` | periodic | revision | 压缩模式 | periodic 按时间，revision 按版本 |
| `--max-txn-ops` | 128 | 256（批量操作） | 事务大小 | 单事务最大操作数 |
| `--max-watcher-per-key` | 8192 | 16384（大集群） | Watch 数量 | 单 Key 最大 Watcher 数 |

**磁盘 I/O 优化建议：**
```bash
# 检查 etcd 磁盘延迟（应 < 10ms）
etcdctl endpoint status --write-out=table

# 推荐使用 SSD，IOPS >= 3000
# AWS 推荐：gp3 或 io1/io2
# 避免使用网络存储（EFS、NFS）
```

#### 4.1.6 etcd 问题定位表

| 问题 | 症状 | 原因 | 排查命令 | 解决方案 |
|------|------|------|----------|----------|
| **Leader 频繁切换** | 日志显示 "elected leader" | 网络延迟 > election-timeout | `etcdctl endpoint status` | 增加 election-timeout |
| **写入延迟高** | 写入 > 100ms | 磁盘 I/O 慢 | `etcdctl endpoint status --write-out=table` | 使用 SSD，检查 fsync |
| **存储空间不足** | "mvcc: database space exceeded" | 未配置自动压缩 | `etcdctl endpoint status` | 设置 auto-compaction |
| **Watch 延迟** | 事件通知慢 | WatchCache 过小 | `etcdctl watch --rev=<rev>` | 增加 watch-cache-size |
| **集群不健康** | "cluster is unhealthy" | 节点故障或网络分区 | `etcdctl endpoint health` | 检查节点状态和网络 |
| **内存使用高** | OOM 或内存持续增长 | 历史版本过多 | `etcdctl endpoint status` | 执行 compact + defrag |
| **快照恢复慢** | 集群启动慢 | 快照过大 | 检查快照文件大小 | 减小 snapshot-count |
| **请求被拒绝** | "too many requests" | QPS 超限 | 检查客户端请求频率 | 增加 etcd 副本或优化客户端 |

**常用排查命令：**
```bash
# 检查集群健康状态
etcdctl endpoint health --cluster

# 检查集群状态详情
etcdctl endpoint status --cluster --write-out=table

# 检查 Leader
etcdctl endpoint status --write-out=table | grep -i leader

# 查看 etcd 指标
curl -s http://localhost:2379/metrics | grep etcd_

# 检查磁盘使用
etcdctl endpoint status --write-out=table | awk '{print $6}'

# 执行压缩（释放空间）
rev=$(etcdctl endpoint status --write-out=json | jq -r '.[] | .Status.header.revision')
etcdctl compact $rev

# 执行碎片整理
etcdctl defrag --cluster

# 查看 Key 数量
etcdctl get / --prefix --keys-only | wc -l
```

#### 4.1.7 etcd 集群规模规划

| 集群规模 | etcd 节点数 | 推荐配置 | 磁盘 IOPS | 说明 |
|----------|------------|----------|-----------|------|
| **小型**（< 100 节点） | 3 | 2 vCPU, 8GB RAM | 1000+ | 开发/测试环境 |
| **中型**（100-500 节点） | 3-5 | 4 vCPU, 16GB RAM | 3000+ | 生产环境 |
| **大型**（500-1000 节点） | 5 | 8 vCPU, 32GB RAM | 8000+ | 大规模生产 |
| **超大型**（> 1000 节点） | 5 | 16 vCPU, 64GB RAM | 16000+ | 需要分片或联邦 |

**注意事项：**
- etcd 节点数必须为奇数（3 或 5），保证 Raft 多数派
- 超过 5 个节点会增加写入延迟，不建议
- 跨 AZ 部署时注意网络延迟（建议 < 10ms）


### 4.2 API Server 架构详解

API Server 是 K8S 控制平面的核心入口，所有组件都通过 API Server 交互。

```
┌─────────────────────────────────────────────────────────┐
│                   API Server 请求处理流程                 │
│                                                         │
│  客户端请求 ──▶ 认证 ──▶ 授权 ──▶ 准入控制 ──▶ etcd         │
│                                                         │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌────────┐   │
│  │ 认证      │  │ 授权     │  │ 准入控制   │  │ 持久化  │   │
│  │ AuthN    │─▶│ AuthZ    │─▶│ Admission│─▶│ etcd   │   │
│  └──────────┘  └──────────┘  └──────────┘  └────────┘   │
│                                                         │
│  认证方式：      授权方式：      准入控制器：                 │
│  • X509证书     • RBAC        • Mutating                 │
│  • Token       • ABAC        • Validating               │
│  • OIDC        • Node        • ResourceQuota            │
│  • Webhook     • Webhook     • LimitRanger              │
└─────────────────────────────────────────────────────────┘
```

#### 4.2.1 认证机制（Authentication）

| 认证方式 | 说明 | 适用场景 |
|---------|------|---------|
| X509 客户端证书 | 基于 TLS 证书的双向认证 | 集群组件、kubectl |
| Bearer Token | ServiceAccount Token | Pod 内访问 API |
| OIDC | OpenID Connect 集成 | 企业 SSO 集成 |
| Webhook | 外部认证服务 | 自定义认证逻辑 |

#### 4.2.2 授权机制（Authorization）

**RBAC 核心概念：**
```
┌─────────────────────────────────────────────────────────┐
│                    RBAC 模型                             │
│                                                         │
│  User/Group/ServiceAccount                              │
│           │                                             │
│           │ RoleBinding / ClusterRoleBinding            │
│           ▼                                             │
│  Role / ClusterRole                                     │
│           │                                             │
│           │ rules: [apiGroups, resources, verbs]        │
│           ▼                                             │
│  Resources (pods, services, deployments...)             │
└─────────────────────────────────────────────────────────┘
```

| 资源 | 作用域 | 说明 |
|------|--------|------|
| Role | Namespace | 命名空间级别权限 |
| ClusterRole | Cluster | 集群级别权限 |
| RoleBinding | Namespace | 绑定 Role 到用户 |
| ClusterRoleBinding | Cluster | 绑定 ClusterRole 到用户 |

**RBAC 工作机制详解：**

RBAC（Role-Based Access Control）是 Kubernetes 推荐的授权模式，通过"角色"和"绑定"两层抽象实现权限管理。理解 RBAC 的关键是理解"谁（Subject）可以对什么资源（Resource）执行什么操作（Verb）"。

**RBAC 四种资源的关系：**

| 资源类型 | 定义内容 | 作用范围 | 使用场景 |
|----------|----------|----------|----------|
| **Role** | 权限规则（apiGroups + resources + verbs） | 单个 Namespace | 应用团队权限隔离 |
| **ClusterRole** | 权限规则 | 整个集群 | 集群管理员、跨 NS 资源访问 |
| **RoleBinding** | Subject → Role 的映射 | 单个 Namespace | 将 Role 授予用户/组/SA |
| **ClusterRoleBinding** | Subject → ClusterRole 的映射 | 整个集群 | 将 ClusterRole 授予用户 |

**权限规则（rules）的三要素：**

```yaml
rules:
- apiGroups: [""]           # 核心 API 组（Pod、Service、ConfigMap）
  resources: ["pods"]       # 资源类型
  verbs: ["get", "list"]    # 允许的操作
- apiGroups: ["apps"]       # apps API 组（Deployment、StatefulSet）
  resources: ["deployments"]
  verbs: ["get", "list", "create", "update", "delete"]
```

**常用 Verbs 说明：**

| Verb | 对应 HTTP 方法 | 说明 |
|------|---------------|------|
| get | GET（单个资源） | 获取单个资源详情 |
| list | GET（资源列表） | 列出资源 |
| watch | GET（长连接） | 监听资源变化 |
| create | POST | 创建资源 |
| update | PUT | 更新整个资源 |
| patch | PATCH | 部分更新资源 |
| delete | DELETE | 删除单个资源 |
| deletecollection | DELETE（集合） | 批量删除资源 |

**特殊 Verbs：**
- `*`：表示所有操作权限
- `impersonate`：允许模拟其他用户/组/ServiceAccount

**为什么需要 ClusterRole + RoleBinding 组合？**

这是一个常见的最佳实践：定义一个 ClusterRole（如 `pod-reader`），然后在不同 Namespace 中通过 RoleBinding 复用，避免在每个 Namespace 中重复定义相同的 Role。

#### 4.2.3 准入控制（Admission Control）

**两类准入控制器：**
- **Mutating**：可修改请求对象（如注入 Sidecar）
- **Validating**：只验证不修改（如检查资源配额）

**执行顺序：** Mutating → Validating → 持久化到 etcd

#### 4.2.4 API Server 请求处理 6 阶段流程

```
┌─────────────────────────────────────────────────────────────────────┐
│                 API Server 请求处理流程（6 阶段）                      │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Client Request (kubectl / SDK / Controller)                        │
│         │                                                           │
│         ▼                                                           │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │ 1. Authentication（认证）                                    │    │
│  │    ├─ X509 Client Cert（证书认证）                           │    │
│  │    ├─ Bearer Token（ServiceAccount Token）                  │    │
│  │    ├─ OIDC Token（企业 SSO）                                 │    │
│  │    └─ Webhook Token（自定义认证）                             │    │
│  │    结果：识别用户身份 (User, Groups)                           │    │
│  └─────────────────────────────────────────────────────────────┘    │
│         │ 认证失败 → 401 Unauthorized                                │
│         ▼                                                           │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │ 2. Authorization（授权）                                     │    │
│  │    ├─ RBAC（推荐，基于角色）                                   │    │
│  │    ├─ Node（kubelet 专用）                                   │    │
│  │    ├─ Webhook（外部授权服务）                                 │    │
│  │    └─ ABAC（已弃用）                                         │    │
│  │    结果：判断用户是否有权限执行操作                              │    │
│  └─────────────────────────────────────────────────────────────┘    │
│         │ 授权失败 → 403 Forbidden                                   │
│         ▼                                                           │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │ 3. Mutating Admission（变更准入）                             │    │
│  │    ├─ 内置：DefaultStorageClass, ServiceAccount              │    │
│  │    └─ Webhook：注入 Sidecar, 添加标签, 设置默认值               │    │
│  │    结果：修改请求对象                                          │    │
│  └─────────────────────────────────────────────────────────────┘    │
│         │                                                           │
│         ▼                                                           │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │ 4. Schema Validation（模式验证）                              │    │
│  │    └─ 验证对象是否符合 API 定义（字段类型、必填项）                │    │
│  └─────────────────────────────────────────────────────────────┘    │
│         │ 验证失败 → 400 Bad Request                                 │
│         ▼                                                           │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │ 5. Validating Admission（验证准入）                           │    │
│  │    ├─ 内置：LimitRanger, ResourceQuota, PodSecurity          │    │
│  │    └─ Webhook：OPA/Gatekeeper, Kyverno, 自定义策略            │    │
│  │    结果：拒绝不符合策略的请求                                   │    │
│  └─────────────────────────────────────────────────────────────┘    │
│         │ 验证失败 → 403 Forbidden                                   │
│         ▼                                                           │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │ 6. Persist to etcd（持久化）                                  │    │
│  │    └─ 写入 etcd 存储，返回成功响应                              │    │
│  └─────────────────────────────────────────────────────────────┘    │
│         │                                                           │
│         ▼                                                           │
│  Response → Client (201 Created / 200 OK)                           │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

#### 4.2.5 认证方式横向对比

| 认证方式 | 适用场景 | 安全性 | 配置复杂度 | Token 有效期 | 说明 |
|----------|----------|--------|------------|--------------|------|
| **X509 Client Cert** | 管理员、CI/CD | 高 | 中 | 证书有效期 | 需要管理证书生命周期 |
| **ServiceAccount Token** | Pod 内应用 | 中 | 低 | 可配置（默认 1h） | K8S 自动管理，推荐 Bound Token |
| **OIDC Token** | 企业 SSO 集成 | 高 | 高 | IdP 配置 | 集成 Okta、Azure AD、Keycloak |
| **Webhook Token** | 自定义认证 | 可变 | 高 | 自定义 | 需要开发认证服务 |
| **Bootstrap Token** | 节点加入集群 | 低 | 低 | 24h | 仅用于 kubeadm join |

**认证配置示例（kubeconfig）：**
```yaml
apiVersion: v1
kind: Config
clusters:
- cluster:
    certificate-authority-data: <CA_CERT>
    server: https://api.example.com:6443
  name: my-cluster
users:
- name: admin
  user:
    client-certificate-data: <CLIENT_CERT>  # X509 认证
    client-key-data: <CLIENT_KEY>
- name: developer
  user:
    token: <BEARER_TOKEN>  # Token 认证
contexts:
- context:
    cluster: my-cluster
    user: admin
  name: admin@my-cluster
```

#### 4.2.6 API Server 参数调优表

| 参数 | 默认值 | 调优建议 | 性能影响 | 说明 |
|------|--------|----------|----------|------|
| `--max-requests-inflight` | 400 | 800（大集群） | 并发读请求 | 非 mutating 请求并发数 |
| `--max-mutating-requests-inflight` | 200 | 400（大集群） | 并发写请求 | mutating 请求并发数 |
| `--request-timeout` | 60s | 60s | 请求超时 | 单请求最大处理时间 |
| `--watch-cache-sizes` | 默认 | 增大热点资源 | Watch 性能 | 格式：resource#size |
| `--default-watch-cache-size` | 100 | 500（大集群） | Watch 缓存 | 默认 Watch 缓存大小 |
| `--etcd-compaction-interval` | 5m | 5m | etcd 压缩 | 自动压缩间隔 |
| `--audit-log-maxage` | 0 | 30 | 审计日志 | 日志保留天数 |
| `--audit-log-maxsize` | 0 | 100 | 审计日志 | 单文件最大 MB |

#### 4.2.7 API Server 问题定位表

| 问题 | 症状 | 原因 | 排查命令 | 解决方案 |
|------|------|------|----------|----------|
| **认证失败** | 401 Unauthorized | Token 过期或证书无效 | `kubectl auth whoami` | 刷新 Token 或更新证书 |
| **授权失败** | 403 Forbidden | RBAC 权限不足 | `kubectl auth can-i <verb> <resource>` | 检查 Role/RoleBinding |
| **请求超时** | 504 Gateway Timeout | API Server 过载 | `kubectl get --raw='/healthz?verbose'` | 增加副本或优化请求 |
| **etcd 连接失败** | 500 Internal Error | etcd 不可用 | `etcdctl endpoint health` | 检查 etcd 集群状态 |
| **准入拒绝** | 403 Forbidden (admission) | Webhook 拒绝 | `kubectl describe pod` | 检查准入控制器日志 |
| **API 限流** | 429 Too Many Requests | 请求超过限制 | 检查客户端 QPS | 增加 max-requests-inflight |
| **Watch 断开** | 资源状态不同步 | 网络问题或超时 | 检查客户端日志 | 增加 watch-cache-sizes |

**常用排查命令：**
```bash
# 检查 API Server 健康状态
kubectl get --raw='/healthz?verbose'

# 检查 API Server 指标
kubectl get --raw='/metrics' | grep apiserver_request

# 检查当前用户权限
kubectl auth can-i --list

# 检查特定操作权限
kubectl auth can-i create pods -n default

# 调试请求（显示详细信息）
kubectl get pods -v=8

# 检查 API Server 日志
kubectl logs -n kube-system kube-apiserver-<node>
```

#### 4.2.8 请求处理链底层机制

**认证链（Authentication Chain）执行逻辑：**

```
┌─────────────────────────────────────────────────────────────────────┐
│                    认证链执行流程                                    │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  请求到达 API Server                                                 │
│       │                                                              │
│       ▼                                                              │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │ 认证器链（按顺序执行，任一成功即通过）                        │    │
│  │                                                              │    │
│  │  1. X509 Client Cert Authenticator                          │    │
│  │     └─ 检查 TLS 证书的 CN/O 字段                            │    │
│  │     └─ 成功: User=CN, Groups=O                              │    │
│  │                    │                                        │    │
│  │                    ▼ 失败则继续                              │    │
│  │  2. Bearer Token Authenticator                              │    │
│  │     └─ 解析 Authorization: Bearer <token>                   │    │
│  │     └─ 验证 ServiceAccount Token（JWT 签名验证）            │    │
│  │                    │                                        │    │
│  │                    ▼ 失败则继续                              │    │
│  │  3. OIDC Authenticator                                      │    │
│  │     └─ 验证 OIDC ID Token（JWT 签名 + issuer + audience）   │    │
│  │                    │                                        │    │
│  │                    ▼ 失败则继续                              │    │
│  │  4. Webhook Token Authenticator                             │    │
│  │     └─ 调用外部认证服务                                     │    │
│  │                    │                                        │    │
│  │                    ▼ 全部失败                                │    │
│  │  5. Anonymous Authenticator（如果启用）                     │    │
│  │     └─ User=system:anonymous, Groups=[system:unauthenticated]│    │
│  └─────────────────────────────────────────────────────────────┘    │
│                                                                      │
│  认证结果: UserInfo{Username, UID, Groups, Extra}                   │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

**ServiceAccount Token 验证流程：**

```
JWT Token 结构:
┌─────────────────────────────────────────────────────────────────────┐
│ Header: {"alg": "RS256", "kid": "key-id"}                           │
│ Payload: {                                                          │
│   "iss": "https://kubernetes.default.svc",                         │
│   "sub": "system:serviceaccount:default:my-sa",                    │
│   "aud": ["https://kubernetes.default.svc"],                       │
│   "exp": 1702742400,                                                │
│   "iat": 1702656000,                                                │
│   "kubernetes.io/serviceaccount/namespace": "default",             │
│   "kubernetes.io/serviceaccount/name": "my-sa"                     │
│ }                                                                   │
│ Signature: RS256(Header.Payload, PrivateKey)                       │
└─────────────────────────────────────────────────────────────────────┘

验证步骤:
1. 解析 JWT Header，获取 kid（Key ID）
2. 从 API Server 的 --service-account-key-file 获取公钥
3. 验证签名: RS256.verify(Header.Payload, Signature, PublicKey)
4. 检查 exp（过期时间）> 当前时间
5. 检查 aud（audience）是否匹配
6. 提取 UserInfo: User=sub, Groups=[system:serviceaccounts, system:serviceaccounts:default]
```

**RBAC 规则匹配算法：**

```
┌─────────────────────────────────────────────────────────────────────┐
│                    RBAC 授权决策流程                                 │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  输入: User, Groups, Verb, Resource, Namespace, Name                │
│                                                                      │
│  1. 查找所有绑定到该 User/Groups 的 RoleBinding/ClusterRoleBinding  │
│     ┌─────────────────────────────────────────────────────────┐     │
│     │ SELECT * FROM RoleBindings                              │     │
│     │ WHERE subjects CONTAINS (User OR Groups)                │     │
│     │ AND namespace = request.namespace (for RoleBinding)     │     │
│     └─────────────────────────────────────────────────────────┘     │
│                                                                      │
│  2. 获取绑定的 Role/ClusterRole 的 rules                            │
│     ┌─────────────────────────────────────────────────────────┐     │
│     │ rules:                                                  │     │
│     │ - apiGroups: [""]                                       │     │
│     │   resources: ["pods"]                                   │     │
│     │   verbs: ["get", "list", "watch"]                       │     │
│     │ - apiGroups: ["apps"]                                   │     │
│     │   resources: ["deployments"]                            │     │
│     │   verbs: ["*"]                                          │     │
│     └─────────────────────────────────────────────────────────┘     │
│                                                                      │
│  3. 规则匹配（任一规则匹配即允许）                                   │
│     ┌─────────────────────────────────────────────────────────┐     │
│     │ for rule in rules:                                      │     │
│     │   if rule.apiGroups CONTAINS request.apiGroup           │     │
│     │      AND rule.resources CONTAINS request.resource       │     │
│     │      AND rule.verbs CONTAINS request.verb               │     │
│     │      AND (rule.resourceNames is empty                   │     │
│     │           OR rule.resourceNames CONTAINS request.name): │     │
│     │     return ALLOW                                        │     │
│     │ return DENY                                             │     │
│     └─────────────────────────────────────────────────────────┘     │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

**WatchCache 数据结构：**

```
┌─────────────────────────────────────────────────────────────────────┐
│                    WatchCache 架构                                   │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  WatchCache 作用:                                                    │
│  • 缓存 etcd 数据，减少 etcd 压力                                   │
│  • 支持 Watch 的断点续传（通过 resourceVersion）                    │
│  • 提供一致性读取                                                   │
│                                                                      │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │                    WatchCache 结构                           │    │
│  │                                                              │    │
│  │  store: map[key]*storeElement                               │    │
│  │    └─ 当前所有对象的最新版本                                 │    │
│  │                                                              │    │
│  │  cache: cyclic buffer (环形缓冲区)                          │    │
│  │    └─ 最近 N 个事件（默认 100 个）                          │    │
│  │    └─ 每个事件: {Type, Object, PrevObject, ResourceVersion} │    │
│  │                                                              │    │
│  │  resourceVersion: int64                                     │    │
│  │    └─ 当前缓存的最新版本号                                  │    │
│  │                                                              │    │
│  │  bookmarkAfterResourceVersion: int64                        │    │
│  │    └─ 用于 Bookmark 事件                                    │    │
│  └─────────────────────────────────────────────────────────────┘    │
│                                                                      │
│  Watch 断点续传流程:                                                 │
│  1. 客户端发起 Watch(resourceVersion=12345)                         │
│  2. WatchCache 检查 12345 是否在 cache 环形缓冲区内                 │
│  3. 如果在: 从 cache 中回放 12345 之后的事件                        │
│  4. 如果不在: 返回 410 Gone，客户端需要重新 List                    │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

**Admission Webhook 执行机制：**

| 阶段 | Webhook 类型 | 执行方式 | 失败策略 | 说明 |
|------|-------------|----------|----------|------|
| **Mutating** | MutatingAdmissionWebhook | 串行执行 | Fail/Ignore | 可修改对象，按 order 排序 |
| **Validating** | ValidatingAdmissionWebhook | 并行执行 | Fail/Ignore | 只验证不修改 |

```yaml
# Webhook 配置示例
apiVersion: admissionregistration.k8s.io/v1
kind: MutatingWebhookConfiguration
metadata:
  name: sidecar-injector
webhooks:
- name: sidecar.example.com
  clientConfig:
    service:
      name: sidecar-injector
      namespace: kube-system
      path: "/inject"
    caBundle: <CA_BUNDLE>
  rules:
  - operations: ["CREATE"]
    apiGroups: [""]
    apiVersions: ["v1"]
    resources: ["pods"]
  failurePolicy: Fail      # Fail: 拒绝请求, Ignore: 忽略错误
  timeoutSeconds: 10       # Webhook 超时时间
  reinvocationPolicy: Never # 是否重新调用
```


### 4.3 kubelet 架构详解

kubelet 是运行在每个节点上的代理，负责管理 Pod 生命周期。

```
┌─────────────────────────────────────────────────────────┐
│                    kubelet 架构                          │
│                                                         │
│  ┌──────────────────────────────────────────────────┐   │
│  │                   kubelet                        │   │
│  │  ┌────────────┐  ┌────────────┐  ┌────────────┐  │   │
│  │  │ SyncLoop   │  │ PLEG       │  │ProbeManager│  │   │
│  │  │ Pod 同步    │  │ Pod 生命周期│  │ 健康检查    │  │   │
│  │  └────────────┘  └────────────┘  └────────────┘  │   │
│  │  ┌────────────┐  ┌────────────┐  ┌────────────┐  │   │
│  │  │ cAdvisor   │  │VolumeManager│ │ ImageManager│ │   │
│  │  │ 资源监控    │  │ 存储管理     │  │ 镜像管理    │  │   │
│  │  └────────────┘  └────────────┘  └────────────┘  │   │
│  └──────────────────────────────────────────────────┘   │
│           │              │              │               │
│           ▼              ▼              ▼               │
│  ┌────────────┐  ┌────────────┐  ┌────────────┐         │
│  │    CRI     │  │    CNI     │  │    CSI     │         │
│  │ 容器运行时   │  │ 网络插件    │  │ 存储插件    │         │
│  └────────────┘  └────────────┘  └────────────┘         │
└─────────────────────────────────────────────────────────┘
```

**PLEG（Pod Lifecycle Event Generator）工作机制：**

PLEG 是 kubelet 的核心组件，负责将容器运行时的状态变化转换为 Pod 生命周期事件。

```
┌─────────────────────────────────────────────────────────────────────┐
│                    PLEG 工作机制详解                                  │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  PLEG 的作用:                                                        │
│  • 定期调用 CRI 获取所有容器状态                                        │
│  • 对比上次状态，生成 Pod 生命周期事件                                   │
│  • 将事件发送到 SyncLoop 触发 Pod 同步                                 │
│                                                                     │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │                    Generic PLEG（轮询模式）                   │    │
│  │                                                             │    │
│  │  每 1 秒执行一次 relist:                                      │    │
│  │  1. 调用 CRI ListPodSandbox() 获取所有 Pod                    │    │
│  │  2. 调用 CRI ListContainers() 获取所有容器                     │    │
│  │  3. 对比上次状态，生成事件:                                     │    │
│  │     • ContainerStarted: 容器启动                             │    │
│  │     • ContainerDied: 容器退出                                │    │
│  │     • ContainerRemoved: 容器被删除                           │    │
│  │     • ContainerChanged: 容器状态变化                          │    │
│  │  4. 将事件发送到 plegCh 通道                                   │    │
│  │                                                             │    │
│  │  关键参数:                                                   │    │
│  │  • relistPeriod: 1s（relist 周期）                           │    │
│  │  • relistThreshold: 3m（relist 超时阈值，超过则不健康）         │    │
│  └─────────────────────────────────────────────────────────────┘    │
│                                                                     │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │                    Evented PLEG（事件驱动模式）                │    │
│  │                    K8S 1.26+ Alpha，1.27+ Beta              │    │
│  │                                                             │    │
│  │  工作方式:                                                   │    │
│  │  1. 订阅 CRI 的容器事件流（GetContainerEvents）                │    │
│  │  2. 实时接收容器状态变化事件                                    │    │
│  │  3. 无需轮询，延迟更低                                         │    │
│  │                                                             │    │
│  │  优势:                                                       │    │
│  │  • 延迟: 毫秒级（vs Generic PLEG 的秒级）                       │    │
│  │  • CPU: 降低 ~10%（无需频繁 relist）                           │    │
│  │  • 适合大规模节点（100+ Pod）                                  │    │
│  │                                                             │    │
│  │  启用方式:                                                   │    │
│  │  kubelet --feature-gates=EventedPLEG=true                   │    │
│  └─────────────────────────────────────────────────────────────┘    │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

**SyncLoop 主循环：**

```
┌─────────────────────────────────────────────────────────────────────┐
│                    kubelet SyncLoop 事件源                           │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  SyncLoop 监听多个事件源:                                             │
│                                                                     │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐                  │
│  │  configCh   │  │   plegCh    │  │   syncCh    │                  │
│  │ API Server  │  │    PLEG     │  │  定时同步    │                   │
│  │ Pod 变更     │  │ 容器事件     │  │  每 1 秒     │                  │
│  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘                  │
│         │                │                │                         │
│         └────────────────┼────────────────┘                         │
│                          ▼                                          │
│                   ┌─────────────┐                                   │
│                   │  SyncLoop   │                                   │
│                   │  主循环      │                                   │
│                   └──────┬──────┘                                   │
│                          │                                          │
│                          ▼                                          │
│                   ┌─────────────┐                                   │
│                   │ syncPod()   │                                   │
│                   │ 同步 Pod     │                                   │
│                   └─────────────┘                                   │
│                                                                     │
│  其他事件源:                                                          │
│  • housekeepingCh: 清理任务（每 2 秒）                                 │
│  • livenessManager: 存活探针失败事件                                   │
│  • readinessManager: 就绪探针失败事件                                  │
│  • startupManager: 启动探针失败事件                                    │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

**PLEG 不健康的排查：**

| 症状 | 原因 | 排查命令 | 解决方案 |
|------|------|----------|----------|
| 节点 NotReady | PLEG relist 超时 | `kubectl describe node` 查看 Conditions | 检查容器运行时 |
| "PLEG is not healthy" | relist 耗时 > 3 分钟 | `journalctl -u kubelet \| grep PLEG` | 减少节点 Pod 数量 |
| Pod 状态不更新 | PLEG 事件丢失 | `crictl ps` 对比实际状态 | 重启 kubelet |

```bash
# 检查 PLEG 健康状态
curl -s http://localhost:10248/healthz/pleg

# 查看 PLEG 相关日志
journalctl -u kubelet | grep -i pleg

# 查看 relist 耗时指标
curl -s http://localhost:10255/metrics | grep pleg_relist_duration
```

**Pod 状态机（Pod Phase 转换）：**

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Pod Phase 状态转换                                │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ┌─────────┐                                                        │
│  │ Pending │ ← Pod 已创建，等待调度或容器创建                           │
│  └────┬────┘                                                        │
│       │ 所有容器创建成功                                               │
│       ▼                                                             │
│  ┌─────────┐                                                        │
│  │ Running │ ← 至少一个容器正在运行                                    │
│  └────┬────┘                                                        │
│       │                                                             │
│       ├──────────────────┬──────────────────┐                       │
│       │                  │                  │                       │
│       ▼                  ▼                  ▼                       │
│  ┌───────────┐    ┌───────────┐    ┌───────────┐                    │
│  │ Succeeded │    │  Failed   │    │  Unknown  │                    │
│  │ 所有容器   │    │ 至少一个    │    │ 无法获取   │                    │
│  │ 成功退出   │    │ 容器失败    │    │ Pod 状态  │                    │
│  └───────────┘    └───────────┘    └───────────┘                    │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

| Pod Phase | 条件 | 说明 |
|-----------|------|------|
| **Pending** | Pod 已被 API Server 接受 | 等待调度、镜像拉取、容器创建 |
| **Running** | Pod 已绑定到节点，至少一个容器运行中 | 包括容器正在启动或重启 |
| **Succeeded** | 所有容器成功终止（exit 0），不会重启 | Job 完成的正常状态 |
| **Failed** | 所有容器终止，至少一个失败（exit ≠ 0） | 需要检查容器日志 |
| **Unknown** | 无法获取 Pod 状态 | 通常是节点通信问题 |

**Container State 状态转换：**

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Container State 状态转换                          │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ┌─────────┐    创建容器      ┌─────────┐                            │
│  │ Waiting │ ──────────────▶ │ Running │                            │
│  │ 等待中   │                 │ 运行中   │                            │
│  └────┬────┘                 └────┬────┘                            │
│       │                           │                                 │
│       │ 镜像拉取失败                │ 容器退出                         │
│       │ 配置错误                   │ OOM Kill                        │
│       ▼                           ▼                                 │
│  ┌─────────────┐           ┌──────────────┐                         │
│  │ Waiting     │           │ Terminated   │                         │
│  │ (错误原因)   │           │ (退出码/信号)  │                         │
│  │ ImagePull   │           │ exit 0/1/137 │                         │
│  │ BackOff     │           └──────┬───────┘                         │
│  └─────────────┘                  │                                 │
│                                   │ restartPolicy                   │
│                                   ▼                                 │
│                            ┌─────────────┐                          │
│                            │ Waiting     │                          │
│                            │ (重启等待)   │                          │
│                            │ CrashLoop   │                          │
│                            │ BackOff     │                          │
│                            └─────────────┘                          │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

| Container State | Reason | 说明 |
|-----------------|--------|------|
| **Waiting** | ContainerCreating | 容器正在创建 |
| **Waiting** | ImagePullBackOff | 镜像拉取失败，等待重试 |
| **Waiting** | CrashLoopBackOff | 容器反复崩溃，等待重启 |
| **Running** | - | 容器正在运行 |
| **Terminated** | Completed (exit 0) | 容器正常退出 |
| **Terminated** | Error (exit 1) | 容器异常退出 |
| **Terminated** | OOMKilled (exit 137) | 内存超限被杀 |

#### 4.3.1 CRI 接口（Container Runtime Interface）

```
┌─────────────────────────────────────────────────────────┐
│                    CRI 接口                              │
│                                                         │
│  kubelet ──▶ CRI gRPC ──▶ 容器运行时                      │
│                                                         │
│  RuntimeService:                                        │
│  • RunPodSandbox()    - 创建 Pod 沙箱                    │
│  • CreateContainer()  - 创建容器                         │
│  • StartContainer()   - 启动容器                         │
│  • StopContainer()    - 停止容器                         │
│  • RemoveContainer()  - 删除容器                         │
│                                                         │
│  ImageService:                                          │
│  • PullImage()        - 拉取镜像                         │
│  • ListImages()       - 列出镜像                         │
│  • RemoveImage()      - 删除镜像                         │
└─────────────────────────────────────────────────────────┘
```

**主流 CRI 实现：**

| 运行时 | 说明 | 适用场景 |
|--------|------|---------|
| containerd | Docker 拆分出的运行时 | 生产环境首选 |
| CRI-O | 专为 K8S 设计 | OpenShift 默认 |
| Docker（已弃用） | 通过 dockershim | K8S 1.24 前 |

**Pod 创建的 CRI 调用序列：**

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Pod 创建 CRI 调用流程                              │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  kubelet SyncPod()                                                  │
│       │                                                             │
│       ▼                                                             │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │ 1. RunPodSandbox()                                          │    │
│  │    • 创建 Pod 的网络命名空间                                   │    │
│  │    • 调用 CNI 插件配置网络                                     │    │
│  │    • 创建 pause 容器（infra container）                       │    │
│  │    • 返回 PodSandboxId                                       │    │
│  └─────────────────────────────────────────────────────────────┘    │
│       │                                                             │
│       ▼                                                             │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │ 2. PullImage()（如果镜像不存在）                               │    │
│  │    • 检查本地镜像缓存                                          │    │
│  │    • 从 Registry 拉取镜像                                     │    │
│  │    • 验证镜像签名（如果启用）                                   │    │
│  └─────────────────────────────────────────────────────────────┘    │
│       │                                                             │
│       ▼                                                             │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │ 3. CreateContainer()                                        │    │
│  │    • 创建容器配置（环境变量、挂载、资源限制）                      │    │
│  │    • 创建容器文件系统（OverlayFS）                              │    │
│  │    • 返回 ContainerId                                        │    │
│  └─────────────────────────────────────────────────────────────┘    │
│       │                                                             │
│       ▼                                                             │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │ 4. StartContainer()                                         │    │
│  │    • 启动容器进程                                             │    │
│  │    • 执行 postStart Hook（如果定义）                           │    │
│  │    • 容器进入 Running 状态                                    │    │
│  └─────────────────────────────────────────────────────────────┘    │
│       │                                                             │
│       ▼                                                             │
│  Pod Running                                                        │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

**containerd vs CRI-O 架构对比：**

```
┌─────────────────────────────────────────────────────────────────────┐
│                    containerd 架构                                   │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  kubelet                                                            │
│     │                                                               │
│     │ CRI gRPC                                                      │
│     ▼                                                               │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │ containerd                                                  │    │
│  │   ├─ CRI Plugin（内置）                                      │    │
│  │   ├─ Image Service                                          │    │
│  │   ├─ Container Service                                      │    │
│  │   └─ Snapshot Service（OverlayFS）                          │    │
│  └─────────────────────────────────────────────────────────────┘    │
│     │                                                               │
│     │ containerd-shim-runc-v2                                       │
│     ▼                                                               │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │ runc（OCI 运行时）                                            │    │
│  │   └─ 创建容器进程（clone + exec）                              │    │
│  └─────────────────────────────────────────────────────────────┘    │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│                    CRI-O 架构                                        │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  kubelet                                                            │
│     │                                                               │
│     │ CRI gRPC                                                      │
│     ▼                                                               │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │ CRI-O                                                       │    │
│  │   ├─ CRI Server                                             │    │
│  │   ├─ Image Server（containers/image）                        │    │
│  │   └─ Storage（containers/storage）                           │    │
│  └─────────────────────────────────────────────────────────────┘    │
│     │                                                               │
│     │ conmon（Container Monitor）                                    │
│     ▼                                                               │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │ runc / crun（OCI 运行时）                                     │    │
│  │   └─ 创建容器进程                                             │    │
│  └─────────────────────────────────────────────────────────────┘    │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

| 维度 | containerd | CRI-O |
|------|------------|-------|
| **设计目标** | 通用容器运行时 | 专为 K8S 设计 |
| **镜像格式** | Docker/OCI | OCI |
| **进程监控** | containerd-shim | conmon |
| **默认 OCI 运行时** | runc | runc/crun |
| **内存占用** | ~50MB | ~30MB |
| **启动延迟** | ~100ms | ~80ms |
| **生态系统** | Docker、K8S、Podman | K8S、OpenShift |
| **推荐场景** | 通用、EKS/GKE/AKS | OpenShift、安全敏感 |

#### 4.3.2 探针类型横向对比

| 探针类型 | 作用 | 失败后果 | 默认行为 | 适用场景 |
|----------|------|----------|----------|----------|
| **livenessProbe** | 检测容器是否存活 | 重启容器 | 无探针则认为存活 | 检测死锁、无响应、内存泄漏 |
| **readinessProbe** | 检测容器是否就绪 | 从 Service Endpoints 移除 | 无探针则认为就绪 | 检测依赖服务、预热完成 |
| **startupProbe** | 检测容器是否启动完成 | 重启容器 | 无探针则立即检测 liveness | 慢启动应用（Java、大型应用） |

**探针配置示例：**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app
spec:
  containers:
  - name: app
    image: myapp:v1
    ports:
    - containerPort: 8080
    # 启动探针（慢启动应用）
    startupProbe:
      httpGet:
        path: /health/startup
        port: 8080
      failureThreshold: 30      # 最多失败 30 次
      periodSeconds: 10         # 每 10 秒检测一次
      # 总启动时间：30 × 10 = 300 秒
    # 存活探针
    livenessProbe:
      httpGet:
        path: /health/live
        port: 8080
      initialDelaySeconds: 0    # startupProbe 成功后立即开始
      periodSeconds: 10
      timeoutSeconds: 5
      failureThreshold: 3       # 连续 3 次失败则重启
    # 就绪探针
    readinessProbe:
      httpGet:
        path: /health/ready
        port: 8080
      initialDelaySeconds: 0
      periodSeconds: 5
      timeoutSeconds: 3
      failureThreshold: 3       # 连续 3 次失败则从 Service 移除
```

**探针检测方式：**

| 方式 | 说明 | 适用场景 |
|------|------|---------|
| **httpGet** | HTTP GET 请求，2xx/3xx 为成功 | Web 应用 |
| **tcpSocket** | TCP 连接检测 | 数据库、非 HTTP 服务 |
| **exec** | 执行命令，exit 0 为成功 | 自定义检测逻辑 |
| **grpc** | gRPC 健康检查协议 | gRPC 服务 |

#### 4.3.3 kubelet 参数调优表

| 参数 | 默认值 | 调优建议 | 性能影响 | 说明 |
|------|--------|----------|----------|------|
| `--max-pods` | 110 | 250（大节点） | Pod 密度 | 单节点最大 Pod 数 |
| `--kube-api-qps` | 5 | 50（大集群） | API 请求速率 | 与 API Server 通信 QPS |
| `--kube-api-burst` | 10 | 100（大集群） | API 请求突发 | 突发请求数 |
| `--sync-frequency` | 1m | 30s（快速同步） | 同步频率 | Pod 状态同步间隔 |
| `--eviction-hard` | memory.available<100Mi | memory.available<500Mi | 驱逐阈值 | 内存不足时驱逐 Pod |
| `--eviction-soft` | 无 | memory.available<1Gi | 软驱逐阈值 | 配合 eviction-soft-grace-period |
| `--image-gc-high-threshold` | 85% | 80% | 镜像清理 | 磁盘使用率阈值 |
| `--image-gc-low-threshold` | 80% | 70% | 镜像清理 | 清理后目标使用率 |
| `--serialize-image-pulls` | true | false（高带宽） | 镜像拉取 | 是否串行拉取镜像 |
| `--registry-qps` | 5 | 20（大规模部署） | 镜像拉取 | 镜像仓库请求 QPS |
| `--pods-per-core` | 0（无限制） | 10 | Pod 密度 | 每核心最大 Pod 数 |

#### 4.3.4 kubelet 问题定位表

| 问题 | 症状 | 原因 | 排查命令 | 解决方案 |
|------|------|------|----------|----------|
| **Pod 卡在 Pending** | Pod 无法调度 | 资源不足或约束不满足 | `kubectl describe pod` | 检查 Events 和调度约束 |
| **Pod 卡在 ContainerCreating** | 容器无法创建 | 镜像拉取失败或 CNI 问题 | `kubectl describe pod` | 检查镜像和网络插件 |
| **CrashLoopBackOff** | 容器反复重启 | 应用错误或配置问题 | `kubectl logs --previous` | 检查应用日志 |
| **OOMKilled** | 容器被 OOM 杀死 | 内存超限 | `kubectl describe pod` | 增加 memory limits |
| **Evicted** | Pod 被驱逐 | 节点资源压力 | `kubectl describe pod` | 检查节点资源 |
| **节点 NotReady** | 节点不可用 | kubelet 问题或网络问题 | `systemctl status kubelet` | 检查 kubelet 日志 |
| **镜像拉取失败** | ImagePullBackOff | 镜像不存在或认证失败 | `kubectl describe pod` | 检查镜像名和 imagePullSecrets |
| **探针失败** | Pod 不断重启或从 Service 移除 | 探针配置不当 | `kubectl describe pod` | 调整探针参数 |

**常用排查命令：**
```bash
# 检查 kubelet 状态
systemctl status kubelet

# 查看 kubelet 日志
journalctl -u kubelet -f

# 检查节点状态
kubectl describe node <node-name>

# 检查节点资源使用
kubectl top node <node-name>

# 检查 Pod 详情
kubectl describe pod <pod-name>

# 查看 Pod 日志
kubectl logs <pod-name> -c <container-name>

# 查看上次崩溃日志
kubectl logs <pod-name> --previous

# 进入容器调试
kubectl exec -it <pod-name> -- /bin/sh

# 检查容器运行时
crictl ps
crictl logs <container-id>
```


### 4.4 kube-proxy 详解

kube-proxy 负责实现 Service 的网络代理和负载均衡。

#### 4.4.1 三种代理模式对比

| 模式 | 实现方式 | 性能 | 适用场景 |
|------|---------|------|---------|
| **userspace** | 用户态代理 | 最差 | 已弃用 |
| **iptables** | 内核 iptables 规则 | 中等 | 中小集群（<1000 Service） |
| **IPVS** | 内核 IPVS 模块 | 最佳 | 大规模集群 |

#### 4.4.2 iptables 模式

```
┌─────────────────────────────────────────────────────────┐
│                 iptables 模式工作原理                     │
│                                                         │
│  数据包 ──▶ PREROUTING ──▶ KUBE-SERVICES                 │
│                              │                          │
│                              ▼                          │
│                    KUBE-SVC-XXX (Service)               │
│                              │                          │
│              ┌───────────────┼───────────────┐          │
│              ▼               ▼               ▼          │
│         KUBE-SEP-1      KUBE-SEP-2      KUBE-SEP-3      │
│         (Pod 1)         (Pod 2)         (Pod 3)         │
│         概率 33%        概率 33%        概率 33%          │
└─────────────────────────────────────────────────────────┘
```

**iptables 模式特点：**
- 规则数量 = O(Services × Endpoints)
- 规则更新需要全量刷新
- 大规模集群性能下降明显

**iptables 规则链详解：**

```
┌─────────────────────────────────────────────────────────────────────┐
│                 iptables 规则链完整流程                               │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  数据包进入节点                                                       │
│       │                                                             │
│       ▼                                                             │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │ PREROUTING 链（nat 表）                                      │    │
│  │   -j KUBE-SERVICES  ← 跳转到 K8S Service 处理链               │    │
│  └─────────────────────────────────────────────────────────────┘    │
│       │                                                             │
│       ▼                                                             │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │ KUBE-SERVICES 链                                            │    │
│  │   作用: 匹配 Service ClusterIP，跳转到对应 Service 链          │    │
│  │                                                             │    │
│  │   规则示例:                                                  │    │
│  │   -d 10.96.0.10/32 -p tcp --dport 53 -j KUBE-SVC-DNS        │    │
│  │   -d 10.96.100.50/32 -p tcp --dport 80 -j KUBE-SVC-NGINX    │    │
│  │   ...                                                       │    │
│  │   -j KUBE-NODEPORTS  ← 最后匹配 NodePort                     │    │
│  └─────────────────────────────────────────────────────────────┘    │
│       │                                                             │
│       ▼                                                             │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │ KUBE-SVC-NGINX 链（以 nginx Service 为例）                    │    │
│  │   作用: 负载均衡，随机选择后端 Pod                              │    │
│  │                                                             │    │
│  │   规则示例（3 个 Pod 副本）:                                   │    │
│  │   -m statistic --mode random --probability 0.33333          │    │
│  │       -j KUBE-SEP-POD1                                      │    │
│  │   -m statistic --mode random --probability 0.50000          │    │
│  │       -j KUBE-SEP-POD2                                      │    │
│  │   -j KUBE-SEP-POD3  ← 最后一个无需概率                         │    │
│  │                                                             │    │
│  │   概率计算: 1/3=0.333, 1/2=0.5, 1/1=1.0                      │    │
│  └─────────────────────────────────────────────────────────────┘    │
│       │                                                             │
│       ▼                                                             │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │ KUBE-SEP-POD1 链（Service Endpoint - Pod 1）                 │    │
│  │   作用: DNAT 到具体 Pod IP                                    │    │
│  │                                                             │    │
│  │   规则:                                                      │    │
│  │   -p tcp -j DNAT --to-destination 10.0.1.10:8080            │    │
│  │                                                             │    │
│  │   DNAT 效果:                                                 │    │
│  │   原目标: 10.96.100.50:80 (Service ClusterIP)                │    │
│  │   新目标: 10.0.1.10:8080 (Pod IP)                            │    │
│  └─────────────────────────────────────────────────────────────┘    │
│       │                                                             │
│       ▼                                                             │
│  数据包路由到 Pod                                                     │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

**实际 iptables 规则查看：**

```bash
# 查看 KUBE-SERVICES 链（所有 Service 入口）
iptables -t nat -L KUBE-SERVICES -n --line-numbers

# 查看特定 Service 的规则链
iptables -t nat -L KUBE-SVC-XXXXXXXXXXXXXXXX -n --line-numbers

# 查看 Endpoint 规则（DNAT 到 Pod）
iptables -t nat -L KUBE-SEP-XXXXXXXXXXXXXXXX -n --line-numbers

# 统计规则数量
iptables -t nat -L | wc -l
```

**NodePort 和 LoadBalancer 的规则链：**

```
NodePort 流量路径:
PREROUTING → KUBE-SERVICES → KUBE-NODEPORTS → KUBE-SVC-XXX → KUBE-SEP-XXX

LoadBalancer 流量路径（externalTrafficPolicy: Cluster）:
PREROUTING → KUBE-SERVICES → KUBE-EXT-XXX → KUBE-SVC-XXX → KUBE-SEP-XXX

LoadBalancer 流量路径（externalTrafficPolicy: Local）:
PREROUTING → KUBE-SERVICES → KUBE-EXT-XXX → KUBE-SVL-XXX → KUBE-SEP-XXX（仅本地 Pod）
```

**conntrack 连接跟踪：**

```
┌─────────────────────────────────────────────────────────────────────┐
│                    conntrack 连接跟踪表                               │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  conntrack 表结构:                                                   │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │ 协议  │ 源IP:端口      │ 目标IP:端口     │ 状态        │ 超时   │    │
│  ├─────────────────────────────────────────────────────────────┤    │
│  │ tcp │ 10.0.0.5:45678 │ 10.96.100.50:80 │ ESTABLISHED │ 120s │    │
│  │     │ 回复: 10.0.1.10:8080 → 10.0.0.5:45678                  │    │
│  └─────────────────────────────────────────────────────────────┘    │
│                                                                     │
│  conntrack 作用:                                                     │
│  • 记录 DNAT 前后的地址映射                                            │
│  • 响应包自动做反向 NAT（SNAT）                                        │
│  • 保证同一连接的包发往同一 Pod                                         │
│                                                                     │
│  常见问题:                                                           │
│  • conntrack 表满: 新连接被丢弃，日志显示 "nf_conntrack: table full"    │
│  • 解决: 增加 net.netfilter.nf_conntrack_max                         │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

```bash
# 查看 conntrack 表
conntrack -L

# 查看 conntrack 统计
conntrack -S

# 查看 conntrack 表大小配置
sysctl net.netfilter.nf_conntrack_max

# 调整 conntrack 表大小
sysctl -w net.netfilter.nf_conntrack_max=262144
```

#### 4.4.3 IPVS 模式

```
┌─────────────────────────────────────────────────────────┐
│                  IPVS 模式工作原理                        │
│                                                         │
│  数据包 ──▶ IPVS Virtual Server (Service ClusterIP)      │
│                              │                          │
│                    负载均衡算法选择                        │
│              ┌───────────────┼───────────────┐          │
│              ▼               ▼               ▼          │
│         Real Server 1   Real Server 2   Real Server 3   │
│         (Pod 1)         (Pod 2)         (Pod 3)         │
└─────────────────────────────────────────────────────────┘
```

**IPVS 负载均衡算法：**

| 算法 | 说明 |
|------|------|
| rr | 轮询（Round Robin） |
| lc | 最少连接（Least Connection） |
| dh | 目标地址哈希 |
| sh | 源地址哈希 |
| sed | 最短期望延迟 |
| nq | 无队列调度 |

**IPVS vs iptables 性能对比：**

| 指标 | iptables | IPVS |
|------|----------|------|
| 规则查找 | O(n) 线性 | O(1) 哈希 |
| 规则更新 | 全量刷新 | 增量更新 |
| 1000 Service 延迟 | ~5ms | ~0.5ms |
| 10000 Service 延迟 | ~50ms | ~1ms |

**IPVS 内核模块详解：**

```
┌─────────────────────────────────────────────────────────────────────┐
│                    IPVS 内核数据结构                                  │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ip_vs_service（虚拟服务）:                                           │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │ struct ip_vs_service {                                      │    │
│  │   u16 protocol;        // TCP/UDP/SCTP                      │    │
│  │   __be32 addr;         // Service ClusterIP (10.96.0.10)    │    │
│  │   __be16 port;         // Service Port (80)                 │    │
│  │   char *sched_name;    // 调度算法 (rr/lc/sh)                │    │
│  │   struct list_head destinations;  // 后端 Pod 列表           │    │
│  │ }                                                           │    │
│  └─────────────────────────────────────────────────────────────┘    │
│                                                                     │
│  ip_vs_dest（真实服务器/Pod）:                                        │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │ struct ip_vs_dest {                                         │    │
│  │   __be32 addr;         // Pod IP (10.0.1.10)                │    │
│  │   __be16 port;         // Pod Port (8080)                   │    │
│  │   int weight;          // 权重                               │    │
│  │   atomic_t activeconns; // 活跃连接数                         │    │
│  │   atomic_t inactconns;  // 非活跃连接数                       │    │
│  │ }                                                           │    │
│  └─────────────────────────────────────────────────────────────┘    │
│                                                                     │
│  ip_vs_conn（连接跟踪）:                                              │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │ struct ip_vs_conn {                                         │    │
│  │   __be32 caddr;        // 客户端 IP                          │    │
│  │   __be32 vaddr;        // Service ClusterIP                 │    │
│  │   __be32 daddr;        // Pod IP                            │    │
│  │   __be16 cport, vport, dport;  // 端口                      │    │
│  │   struct ip_vs_dest *dest;     // 目标 Pod                  │    │
│  │ }                                                           │    │
│  └─────────────────────────────────────────────────────────────┘    │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

**IPVS 连接调度流程：**

```
┌─────────────────────────────────────────────────────────────────────┐
│                    IPVS 数据包处理流程                                │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  1. 数据包到达（目标: 10.96.0.10:80）                                  │
│       │                                                             │
│       ▼                                                             │
│  2. IPVS 查找 ip_vs_service 哈希表                                    │
│     key = hash(protocol, vaddr, vport)                              │
│     O(1) 时间复杂度                                                   │
│       │                                                             │
│       ▼                                                             │
│  3. 检查是否有已存在的连接（ip_vs_conn）                                │
│     ├─ 有: 直接使用已有连接的目标 Pod                                   │
│     └─ 无: 调用调度算法选择新的 Pod                                     │
│       │                                                             │
│       ▼                                                             │
│  4. 调度算法选择后端 Pod                                              │
│     ┌─────────────────────────────────────────────────────────┐     │
│     │ rr:  轮询，依次选择                                       │     │
│     │ lc:  选择 activeconns 最少的 Pod                          │    │
│     │ sh:  hash(源IP) % Pod数量，保证同源到同Pod                 │     │
│     │ sed: 选择 (activeconns+1)/weight 最小的 Pod               │     │
│     └─────────────────────────────────────────────────────────┘     │
│       │                                                             │
│       ▼                                                             │
│  5. 创建 ip_vs_conn 连接记录                                          │
│       │                                                             │
│       ▼                                                             │
│  6. DNAT: 修改目标地址为 Pod IP                                       │
│     10.96.0.10:80 → 10.0.1.10:8080                                  │
│       │                                                             │
│       ▼                                                             │
│  7. 转发到 Pod                                                       │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

**IPVS 常用命令：**

```bash
# 查看所有虚拟服务
ipvsadm -Ln

# 查看特定服务的后端
ipvsadm -Ln -t 10.96.0.10:80

# 查看连接统计
ipvsadm -Ln --stats

# 查看连接速率
ipvsadm -Ln --rate

# 查看当前连接
ipvsadm -Lnc

# 检查 IPVS 模块是否加载
lsmod | grep ip_vs
```

#### 4.4.4 eBPF 替代方案（Cilium）

```
┌─────────────────────────────────────────────────────────────────────┐
│                    eBPF 数据路径（Cilium）                            │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  传统路径（iptables/IPVS）：                                           │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │ 数据包 → 网卡 → 内核网络栈 → iptables/IPVS → 路由 → Pod         │    │
│  │                    ↑                                        │    │
│  │              多次内核态切换                                   │    │
│  └─────────────────────────────────────────────────────────────┘    │
│                                                                     │
│  eBPF 路径（Cilium）：                                                 │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │ 数据包 → 网卡 → XDP/TC eBPF → 直接转发 → Pod                   │    │
│  │                ↑                                            │    │
│  │         绕过 iptables/conntrack                              │    │
│  └─────────────────────────────────────────────────────────────┘    │
│                                                                     │
│  eBPF 挂载点：                                                       │
│  • XDP（eXpress Data Path）：网卡驱动层，最早处理                        │
│  • TC（Traffic Control）：流量控制层                                   │
│  • Socket：套接字层，应用感知                                          │
│  • cgroup：容器组级别控制                                             │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

**iptables vs IPVS vs eBPF 详细对比：**

| 维度 | iptables | IPVS | eBPF (Cilium) |
|------|----------|------|---------------|
| **规则复杂度** | O(n) 线性匹配 | O(1) 哈希查找 | O(1) 哈希查找 |
| **Service 数量** | < 1000 | < 10000 | 无限制 |
| **负载均衡算法** | 随机 | rr/lc/sh/sed/nq | rr/random/maglev |
| **连接跟踪** | conntrack（必须） | conntrack（必须） | 可选（可绕过） |
| **性能（延迟）** | 高（规则多时） | 低 | 最低 |
| **性能（吞吐）** | 中 | 高 | 最高（可达 40Gbps+） |
| **内核要求** | 3.10+ | 4.0+ | 4.9+（推荐 5.4+） |
| **配置复杂度** | 低 | 中 | 高 |
| **可观测性** | 差 | 中 | 优秀（Hubble） |
| **NetworkPolicy** | 需要 CNI 支持 | 需要 CNI 支持 | 原生支持 + L7 策略 |
| **Service Mesh** | 🔴 不支持 | 🔴 不支持 | ✅ 支持（无 Sidecar） |
| **K8S 默认** | ✅ 是 | 🔴 否 | 🔴 否 |

**Cilium 替代 kube-proxy 配置：**
```yaml
# Cilium Helm values
kubeProxyReplacement: strict  # 完全替代 kube-proxy
k8sServiceHost: <API_SERVER_IP>
k8sServicePort: 6443
bpf:
  masquerade: true
  hostRouting: true
loadBalancer:
  algorithm: maglev  # 一致性哈希
  mode: dsr          # Direct Server Return（高性能）
hubble:
  enabled: true      # 可观测性
  relay:
    enabled: true
  ui:
    enabled: true
```

#### 4.4.5 kube-proxy 参数调优表

| 参数 | 默认值 | 调优建议 | 性能影响 | 说明 |
|------|--------|----------|----------|------|
| `--proxy-mode` | iptables | ipvs（大集群） | 性能模式 | 代理模式选择 |
| `--iptables-sync-period` | 30s | 60s（大集群） | iptables 同步 | 规则同步间隔 |
| `--iptables-min-sync-period` | 1s | 5s | 最小同步间隔 | 防止频繁同步 |
| `--ipvs-sync-period` | 30s | 30s | IPVS 同步 | IPVS 规则同步间隔 |
| `--ipvs-scheduler` | rr | lc（长连接） | 负载均衡 | IPVS 调度算法 |
| `--conntrack-max-per-core` | 32768 | 65536（高并发） | 连接跟踪 | 每核心最大连接数 |
| `--conntrack-min` | 131072 | 262144 | 连接跟踪 | 最小连接跟踪数 |

#### 4.4.6 kube-proxy 问题定位表

| 问题 | 症状 | 原因 | 排查命令 | 解决方案 |
|------|------|------|----------|----------|
| **Service 不通** | curl ClusterIP 超时 | Endpoint 为空 | `kubectl get endpoints` | 检查 Pod 标签和 Service selector |
| **负载不均衡** | 流量集中到少数 Pod | iptables 随机算法 | `iptables -t nat -L -n` | 切换到 IPVS 模式 |
| **性能下降** | 延迟增加 | iptables 规则过多 | `iptables -t nat -L -n \| wc -l` | 切换到 IPVS 或 eBPF |
| **conntrack 表满** | 连接失败 | 高并发场景 | `conntrack -C` | 增加 nf_conntrack_max |
| **NodePort 不通** | 外部访问失败 | 防火墙规则 | `iptables -L -n` | 检查安全组/iptables |
| **IPVS 规则丢失** | Service 间歇性不通 | IPVS 同步问题 | `ipvsadm -Ln` | 重启 kube-proxy |
| **externalTrafficPolicy 问题** | 源 IP 丢失 | SNAT 导致 | `kubectl get svc -o yaml` | 设置 externalTrafficPolicy: Local |

**常用排查命令：**
```bash
# 检查 kube-proxy 模式
kubectl get cm kube-proxy -n kube-system -o yaml | grep mode

# 检查 iptables 规则
iptables -t nat -L KUBE-SERVICES -n
iptables -t nat -L -n | grep <service-name>

# 检查 IPVS 规则
ipvsadm -Ln
ipvsadm -Ln -t <cluster-ip>:<port>

# 检查 conntrack 表
conntrack -L | wc -l
conntrack -C

# 检查 Endpoints
kubectl get endpoints <service-name>

# 检查 EndpointSlice
kubectl get endpointslice -l kubernetes.io/service-name=<service-name>

# 检查 kube-proxy 日志
kubectl logs -n kube-system -l k8s-app=kube-proxy

# 测试 Service 连通性
kubectl run debug --image=busybox -it --rm -- wget -qO- http://<service-name>:<port>
```

#### 4.4.7 kube-proxy 模式选型决策

```
┌─────────────────────────────────────────────────────────────────────┐
│                    kube-proxy 模式选型决策树                          │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Service 数量 > 1000？                                               │
│       │                                                             │
│       ├─── 是 ──────────────────────────────────────────────────┐   │
│       │                                                         │   │
│       │    需要高级可观测性或 L7 策略？                             │   │
│       │         │                                               │   │
│       │         ├─── 是 → Cilium eBPF（推荐）                     │   │
│       │         │         • 最高性能                             │   │
│       │         │         • Hubble 可观测性                      │   │
│       │         │         • L7 NetworkPolicy                    │   │
│       │         │                                               │   │
│       │         └─── 否 → IPVS 模式                              │   │
│       │                   • 高性能                               │   │
│       │                   • 多种负载均衡算法                      │   │
│       │                   • 配置简单                             │   │
│       │                                                         │   │
│       └─── 否 ──────────────────────────────────────────────────┐   │
│                                                                 │   │
│            需要简单配置？                                         │   │
│                 │                                               │   │
│                 ├─── 是 → iptables 模式（默认）                   │   │
│                 │         • 配置简单                             │   │
│                 │         • 兼容性好                             │   │
│                 │                                               │   │
│                 └─── 否 → 根据需求选择                            │   │
│                           • 需要 DSR → Cilium                    │   │
│                           • 需要 Maglev → Cilium                 │   │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```


---

## 5. 容器网络架构
### 5.1 CNI 规范详解

CNI（Container Network Interface）是容器网络的标准接口。

```
┌─────────────────────────────────────────────────────────┐
│                    CNI 工作流程                          │
│                                                         │
│  kubelet ──▶ CRI ──▶ CNI 插件                           │
│                        │                                │
│              ┌─────────┴─────────┐                      │
│              ▼                   ▼                      │
│         ADD 操作            DEL 操作                     │
│         (创建网络)          (删除网络)                    │
│              │                   │                      │
│              ▼                   ▼                      │
│         分配 IP              回收 IP                     │
│         配置路由             清理路由                     │
│         设置 veth            删除 veth                   │
└─────────────────────────────────────────────────────────┘
```

### 5.2 主流 CNI 插件对比

| CNI 插件          | 网络模式          | 数据平面          | NetworkPolicy | 适用场景    |
| --------------- | ------------- | ------------- | ------------- | ------- |
| **Calico**      | BGP/VXLAN     | iptables/eBPF | ✅ 原生支持        | 通用场景    |
| **Cilium**      | VXLAN/Native  | eBPF          | ✅ 增强支持        | 高性能/可观测 |
| **Flannel**     | VXLAN/host-gw | iptables      | 🔴 不支持        | 简单场景    |
| **AWS VPC CNI** | Native VPC    | iptables      | ✅ 安全组         | EKS 专用  |

### 5.3 CNI 插件详细架构
#### 5.3.1 AWS VPC CNI 架构

```
┌─────────────────────────────────────────────────────────┐
│                  AWS VPC CNI 架构                        │
│                                                          │
│  ┌──────────────────────────────────────────────────┐   │
│  │                   EC2 节点                        │   │
│  │  ┌────────────┐  ┌────────────┐  ┌────────────┐  │   │
│  │  │   Pod 1    │  │   Pod 2    │  │   Pod 3    │  │   │
│  │  │ 10.0.1.10  │  │ 10.0.1.11  │  │ 10.0.1.12  │  │   │
│  │  └────────────┘  └────────────┘  └────────────┘  │   │
│  │        │              │              │           │   │
│  │        └──────────────┼──────────────┘           │   │
│  │                       ▼                          │   │
│  │              ┌────────────────┐                  │   │
│  │              │  ENI (弹性网卡) │                  │   │
│  │              │  Secondary IPs │                  │   │
│  │              └────────────────┘                  │   │
│  └──────────────────────────────────────────────────┘   │
│                          │                              │
│                          ▼                              │
│                    AWS VPC 网络                          │
│              （Pod IP 直接路由，无 Overlay）               │
└─────────────────────────────────────────────────────────┘
```

**AWS VPC CNI 特点：**
- Pod 使用 VPC 原生 IP，无需 Overlay
- 每个 ENI 可分配多个 Secondary IP
- 支持安全组直接绑定到 Pod
- IP 地址受 ENI 数量和实例类型限制

#### 5.3.2 CNI 插件详细对比

| 维度 | Calico | Cilium | Flannel | AWS VPC CNI | Weave |
|------|--------|--------|---------|-------------|-------|
| **网络模式** | BGP/IPIP/VXLAN | VXLAN/Native | VXLAN/host-gw | Native VPC | VXLAN |
| **数据平面** | iptables/eBPF | eBPF | iptables | iptables | iptables |
| **NetworkPolicy** | ✅ 完整支持 | ✅ 完整 + L7 | 🔴 不支持 | ⚠️ 需要 Calico | ✅ 支持 |
| **性能** | 高 | 最高 | 中 | 高 | 中 |
| **可观测性** | 中 | 优秀（Hubble） | 差 | 中 | 中 |
| **多集群** | ✅ 支持 | ✅ 支持 | 🔴 不支持 | 🔴 不支持 | ✅ 支持 |
| **Service Mesh** | 🔴 不支持 | ✅ 支持 | 🔴 不支持 | 🔴 不支持 | 🔴 不支持 |
| **学习曲线** | 中 | 高 | 低 | 低 | 低 |
| **AWS 集成** | ⚠️ 需要配置 | ⚠️ 需要配置 | ⚠️ 需要配置 | ✅ 原生 | ⚠️ 需要配置 |
| **IP 管理** | Calico IPAM | Cilium IPAM | host-local | VPC IPAM | Weave IPAM |
| **加密** | WireGuard | WireGuard/IPsec | 🔴 不支持 | 🔴 不支持 | ✅ 支持 |
| **适用场景** | 通用 | 高性能/安全 | 简单场景 | AWS EKS | 简单场景 |

#### 5.3.3 网络模型对比（Overlay vs Underlay）

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Overlay 网络（VXLAN）                             │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Pod A (10.244.1.10)                    Pod B (10.244.2.20)         │
│       │                                       │                     │
│       ▼                                       ▼                     │
│  ┌─────────────┐                        ┌─────────────┐             │
│  │   veth      │                        │   veth      │             │
│  └─────────────┘                        └─────────────┘             │
│       │                                       │                     │
│       ▼                                       ▼                     │
│  ┌─────────────┐                        ┌─────────────┐             │
│  │ VXLAN 封装   │                        │ VXLAN 解封  │             │
│  │ 外层 IP:     │                        │ 外层 IP:    │             │
│  │ 192.168.1.1 │ ──── 物理网络 ──────▶    │ 192.168.1.2 │            │
│  │ 内层 IP:     │                        │ 内层 IP:    │             │
│  │ 10.244.1.10 │                        │ 10.244.2.20 │             │
│  └─────────────┘                        └─────────────┘             │
│                                                                     │
│  优点：不依赖底层网络，跨子网通信                                        │
│  缺点：封装开销（~50 字节），MTU 问题，性能损耗 5-10%                      │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│                    Underlay 网络（BGP/Native）                       │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Pod A (10.244.1.10)                    Pod B (10.244.2.20)         │
│       │                                       │                     │
│       ▼                                       ▼                     │
│  ┌─────────────┐                        ┌─────────────┐             │
│  │   veth      │                        │   veth      │             │
│  └─────────────┘                        └─────────────┘             │
│       │                                       │                     │
│       ▼                                       ▼                     │
│  ┌─────────────┐    BGP 路由宣告        ┌─────────────┐              │
│  │ Node 1      │ ◀──────────────────▶  │ Node 2      │              │
│  │ 路由表:      │                       │ 路由表:      │              │
│  │ 10.244.2.0  │                       │ 10.244.1.0  │              │
│  │ via Node 2  │                       │ via Node 1  │              │
│  └─────────────┘                       └─────────────┘              │
│                                                                     │
│  优点：无封装开销，性能最优，原生路由                                     │
│  缺点：需要网络设备支持 BGP，或同子网部署                                 │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

| 维度 | Overlay (VXLAN) | Underlay (BGP) | Native VPC |
|------|-----------------|----------------|------------|
| **封装开销** | ~50 字节 | 无 | 无 |
| **MTU 影响** | 需要调整（1450） | 无影响（1500） | 无影响 |
| **性能损耗** | 5-10% | < 1% | < 1% |
| **跨子网** | ✅ 支持 | 需要 BGP 支持 | ✅ VPC 路由 |
| **网络要求** | 低 | 高（BGP） | AWS VPC |
| **调试难度** | 高（封装） | 中 | 低 |

**VXLAN 封装原理详解：**

```
┌─────────────────────────────────────────────────────────────────────┐
│                    VXLAN 数据包封装结构                               │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  原始数据包（Pod A → Pod B）:                                         │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │ Ethernet │ IP Header       │ TCP/UDP │ Payload              │    │
│  │ 14 bytes │ Src: 10.244.1.10│         │                      │    │
│  │          │ Dst: 10.244.2.20│         │                      │    │
│  └─────────────────────────────────────────────────────────────┘    │
│                                                                     │
│  VXLAN 封装后:                                                       │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │ Outer    │ Outer IP        │ UDP     │ VXLAN  │ 原始数据包    │    │
│  │ Ethernet │ Src: 192.168.1.1│ Dst:4789│ Header │             │    │
│  │ 14 bytes │ Dst: 192.168.1.2│ 8 bytes │ 8 bytes│             │    │
│  └─────────────────────────────────────────────────────────────┘    │
│                                                                     │
│  VXLAN Header 结构（8 字节）:                                         │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │ Flags (8 bits) │ Reserved (24 bits) │ VNI (24 bits) │ Res   │    │
│  │ 0x08           │ 0x000000           │ 例: 42        │ 0x00  │    │
│  └─────────────────────────────────────────────────────────────┘    │
│                                                                     │
│  VNI（VXLAN Network Identifier）:                                    │
│  • 24 位，支持 16M 个虚拟网络                                          │
│  • 类似 VLAN ID，但规模更大                                            │
│  • 每个 K8S 集群通常使用一个 VNI                                        │
│                                                                     │
│  封装开销: 14(Outer Eth) + 20(Outer IP) + 8(UDP) + 8(VXLAN) = 50字节  │
│  MTU 影响: 1500 - 50 = 1450（需要调整 Pod MTU）                        │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

**VTEP（VXLAN Tunnel Endpoint）工作机制：**

```
┌─────────────────────────────────────────────────────────────────────┐
│                    VTEP 工作流程                                     │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Node 1 (VTEP: 192.168.1.1)              Node 2 (VTEP: 192.168.1.2) │
│  ┌─────────────────────────┐            ┌─────────────────────────┐ │
│  │ Pod A: 10.244.1.10      │            │ Pod B: 10.244.2.20      │ │
│  │         │               │            │         ▲               │ │
│  │         ▼               │            │         │               │ │
│  │ ┌─────────────────────┐ │            │ ┌─────────────────────┐ │ │
│  │ │ vxlan0 (VTEP)       │ │            │ │ vxlan0 (VTEP)       │ │ │
│  │ │ 1. 查 FDB 表         │ │            │ │ 4. 解封装            │ │ │
│  │ │ 2. 封装 VXLAN        │ │            │ │ 5. 转发到 Pod B      │ │ │
│  │ └─────────────────────┘ │            │ └─────────────────────┘ │ │
│  └───────────│─────────────┘            └───────────│─────────────┘ │
│              │                                      │               │
│              └──────────── 物理网络 ─────────────────┘               │
│                        3. UDP:4789                                  │
│                                                                     │
│  FDB（Forwarding Database）表:                                       │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │ MAC Address        │ VTEP IP      │ VNI  │ 说明             │    │
│  │ aa:bb:cc:dd:ee:01  │ 192.168.1.1  │ 42   │ Pod A 的 MAC     │    │
│  │ aa:bb:cc:dd:ee:02  │ 192.168.1.2  │ 42   │ Pod B 的 MAC     │    │
│  └─────────────────────────────────────────────────────────────┘    │
│                                                                     │
│  FDB 学习方式:                                                       │
│  • 静态配置: CNI 插件在创建 Pod 时写入                                  │
│  • 动态学习: 通过 ARP/NDP 或控制平面同步                                │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

**BGP 路由原理（Calico）：**

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Calico BGP 路由模式                               │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Node 1                                  Node 2                     │
│  Pod CIDR: 10.244.1.0/24                Pod CIDR: 10.244.2.0/24     │
│  ┌─────────────────────────┐            ┌─────────────────────────┐ │
│  │ BIRD (BGP Daemon)       │◀── BGP ──▶ │ BIRD (BGP Daemon)       │ │
│  │ 宣告: 10.244.1.0/24     │   Peering   │ 宣告: 10.244.2.0/24     │ │
│  └─────────────────────────┘            └─────────────────────────┘ │
│              │                                      │               │
│              ▼                                      ▼               │
│  路由表:                                 路由表:                      │
│  10.244.1.0/24 dev cali*                10.244.1.0/24 via Node1     │
│  10.244.2.0/24 via Node2                10.244.2.0/24 dev cali*     │
│                                                                     │
│  BGP 模式选择:                                                       │
│  • Full Mesh: 所有节点互联（< 100 节点）                               │
│  • Route Reflector: 大规模集群，减少 BGP 连接数                         │
│  • 与 ToR 交换机 Peering: 数据中心部署                                  │
│                                                                     │
│  IPIP 模式（跨子网）:                                                  │
│  • 当节点不在同一 L2 网络时启用                                         │
│  • IP-in-IP 封装（20 字节开销，比 VXLAN 少）                            │
│  • CrossSubnet 模式: 同子网直接路由，跨子网 IPIP                        │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

#### 5.3.4 AWS VPC CNI 详细配置

**ENI 和 IP 限制（常见实例类型）：**

| 实例类型 | 最大 ENI | 每 ENI IP 数 | 最大 Pod 数 | 说明 |
|----------|----------|--------------|-------------|------|
| t3.small | 3 | 4 | 11 | (3×4)-1 |
| t3.medium | 3 | 6 | 17 | (3×6)-1 |
| m5.large | 3 | 10 | 29 | (3×10)-1 |
| m5.xlarge | 4 | 15 | 58 | (4×15)-2 |
| m5.2xlarge | 4 | 15 | 58 | (4×15)-2 |
| m5.4xlarge | 8 | 30 | 234 | (8×30)-6 |

**VPC CNI 环境变量配置：**
```yaml
# aws-node DaemonSet 配置
env:
- name: WARM_ENI_TARGET
  value: "1"              # 预热 ENI 数量
- name: WARM_IP_TARGET
  value: "5"              # 预热 IP 数量
- name: MINIMUM_IP_TARGET
  value: "2"              # 最小 IP 数量
- name: ENABLE_PREFIX_DELEGATION
  value: "true"           # 启用前缀委派（增加 IP 容量）
- name: WARM_PREFIX_TARGET
  value: "1"              # 预热前缀数量
- name: AWS_VPC_K8S_CNI_EXTERNALSNAT
  value: "false"          # 是否使用外部 SNAT
- name: AWS_VPC_ENI_MTU
  value: "9001"           # Jumbo Frame MTU
```

**前缀委派模式（Prefix Delegation）：**
```
┌─────────────────────────────────────────────────────────────────────┐
│                    前缀委派 vs 传统模式                                │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  传统模式（Secondary IP）：                                            │
│  • 每个 ENI 分配多个 Secondary IP                                     │
│  • m5.large: 3 ENI × 10 IP = 29 Pod                                 │
│                                                                     │
│  前缀委派模式（/28 前缀）：                                             │
│  • 每个 ENI 分配 /28 前缀（16 个 IP）                                  │
│  • m5.large: 3 ENI × 9 前缀 × 16 IP = 432 Pod                        │
│  • 大幅提升 Pod 密度                                                  │
│                                                                     │
│  启用条件：                                                           │
│  • Nitro 实例（m5、c5、r5 等）                                        │
│  • VPC CNI 1.9.0+                                                   │
│  • 子网有足够的 /28 CIDR 块                                           │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

#### 5.3.5 容器网络问题定位表

| 问题 | 症状 | 原因 | 排查命令 | 解决方案 |
|------|------|------|----------|----------|
| **Pod 无法获取 IP** | ContainerCreating | CNI 插件失败或 IP 耗尽 | `kubectl describe pod` | 检查 CNI 日志，扩展子网 |
| **Pod 间网络不通** | ping 失败 | NetworkPolicy 或路由问题 | `kubectl exec -- ping` | 检查 NetworkPolicy |
| **Service 不通** | curl ClusterIP 超时 | kube-proxy 或 Endpoints 问题 | `kubectl get endpoints` | 检查 iptables/IPVS 规则 |
| **DNS 解析失败** | nslookup 失败 | CoreDNS 问题 | `kubectl logs -n kube-system coredns` | 检查 CoreDNS 配置 |
| **跨节点不通** | 跨节点 Pod 不通 | VXLAN/BGP 配置问题 | `tcpdump -i any` | 检查网络插件配置 |
| **IP 耗尽** | 新 Pod 无法调度 | 子网 IP 不足 | `aws ec2 describe-subnets` | 扩展子网 CIDR |
| **MTU 问题** | 大包丢失 | VXLAN 封装导致 | `ping -s 1400` | 调整 MTU 配置 |
| **安全组问题** | 特定端口不通 | 安全组规则缺失 | `aws ec2 describe-security-groups` | 添加安全组规则 |

**常用排查命令：**
```bash
# 检查 CNI 插件状态
kubectl get pods -n kube-system -l k8s-app=aws-node

# 查看 CNI 日志
kubectl logs -n kube-system -l k8s-app=aws-node

# 检查节点 ENI 分配
aws ec2 describe-network-interfaces --filters "Name=attachment.instance-id,Values=<instance-id>"

# 检查 Pod 网络命名空间
kubectl exec <pod> -- ip addr
kubectl exec <pod> -- ip route

# 测试 Pod 间连通性
kubectl exec <pod-a> -- ping <pod-b-ip>
kubectl exec <pod-a> -- curl <service-name>:<port>

# 检查 DNS 解析
kubectl exec <pod> -- nslookup kubernetes.default

# 抓包分析
kubectl exec <pod> -- tcpdump -i eth0 -nn

# 检查 NetworkPolicy
kubectl get networkpolicy -A
kubectl describe networkpolicy <policy-name>
```

### 5.4 NetworkPolicy 详解

**NetworkPolicy 示例（默认拒绝 + 白名单）：**
```yaml
# 默认拒绝所有入站流量
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-ingress
  namespace: production
spec:
  podSelector: {}  # 选择所有 Pod
  policyTypes:
  - Ingress
---
# 允许特定流量
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend-to-backend
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: frontend
    ports:
    - protocol: TCP
      port: 8080
```

---

## 6. 容器存储架构
### 6.1 CSI 规范详解

CSI（Container Storage Interface）是容器存储的标准接口。

```
┌─────────────────────────────────────────────────────────┐
│                    CSI 架构                              │
│                                                         │
│  ┌─────────────────────┐    ┌─────────────────────┐     │
│  │  CSI Controller     │    │  CSI Node           │     │
│  │  (Deployment)       │    │  (DaemonSet)        │     │
│  │                     │    │                     │     │
│  │  • CreateVolume     │    │  • NodeStageVolume  │     │
│  │  • DeleteVolume     │    │  • NodePublishVolume│     │
│  │  • ControllerPublish│    │  • NodeUnpublish    │     │
│  └─────────────────────┘    └─────────────────────┘     │
│           │                          │                  │
│           └──────────┬───────────────┘                  │
│                      ▼                                  │
│              云存储 API (EBS/EFS/...)                    │
└─────────────────────────────────────────────────────────┘
```

**CSI 架构核心组件说明：**

CSI（Container Storage Interface）将存储供应商的实现与 Kubernetes 核心代码解耦，使存储插件可以独立开发和部署。CSI 架构分为两个核心组件：Controller 和 Node。

| 组件 | 部署方式 | 职责 | 关键操作 |
|------|----------|------|----------|
| **CSI Controller** | Deployment（通常 1-2 副本） | 卷的生命周期管理 | CreateVolume、DeleteVolume、ControllerPublish |
| **CSI Node** | DaemonSet（每节点一个） | 卷的挂载和卸载 | NodeStageVolume、NodePublishVolume |

**CSI 操作流程详解：**

1. **CreateVolume**：Controller 调用云 API 创建存储卷（如 EBS Volume）
2. **ControllerPublish**：Controller 将卷附加到节点（如 EBS Attach）
3. **NodeStageVolume**：Node 将卷格式化并挂载到全局目录（如 `/var/lib/kubelet/plugins/`）
4. **NodePublishVolume**：Node 将卷 bind mount 到 Pod 目录

**为什么需要 Stage 和 Publish 两步？**

Stage 是将卷挂载到节点的全局目录，Publish 是将卷挂载到 Pod 的目录。这种设计支持一个卷被多个 Pod 共享（如 ReadWriteMany 模式），Stage 只执行一次，Publish 为每个 Pod 执行一次。

### 6.2 StorageClass 详解

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: ebs-gp3
provisioner: ebs.csi.aws.com
parameters:
  type: gp3
  iops: "3000"
  throughput: "125"
volumeBindingMode: WaitForFirstConsumer  # 延迟绑定
reclaimPolicy: Delete
allowVolumeExpansion: true
```


**volumeBindingMode 对比：**

| 模式 | 说明 | 适用场景 |
|------|------|---------|
| Immediate | PVC 创建时立即绑定 PV | 无拓扑约束 |
| WaitForFirstConsumer | Pod 调度后再绑定 | EBS 等有 AZ 限制的存储 |

**PV 访问模式（accessModes）详解：**

| 访问模式 | 缩写 | 说明 | 典型存储 |
|----------|------|------|----------|
| ReadWriteOnce | RWO | 单节点读写 | EBS、本地磁盘 |
| ReadOnlyMany | ROX | 多节点只读 | NFS、EFS |
| ReadWriteMany | RWX | 多节点读写 | NFS、EFS、CephFS |
| ReadWriteOncePod | RWOP | 单 Pod 读写（K8S 1.22+） | 需要独占访问的场景 |

注意：访问模式是 PV 的能力声明，不是强制约束。实际行为取决于存储后端的支持。

**PV 回收策略（reclaimPolicy）详解：**

| 策略 | 说明 | PVC 删除后 PV 状态 | 数据处理 | 适用场景 |
|------|------|-------------------|----------|----------|
| Retain | 保留 | Released（需手动清理） | 保留数据 | 重要数据、需要手动迁移 |
| Delete | 删除 | 删除 PV 和底层存储 | 删除数据 | 临时数据、动态供应 |
| Recycle | 回收（已废弃） | Available | 清空数据（rm -rf） | 不推荐使用 |

```
PV 状态流转：
Available → Bound → Released → (Available/Failed/删除)
                        │
                        ├─ Retain：需手动清理 claimRef 后变为 Available
                        └─ Delete：自动删除 PV 和底层存储
```

### 6.3 CSI 组件详解

```
┌─────────────────────────────────────────────────────────────────────┐
│                    CSI 完整架构与组件                                 │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │                    Kubernetes 控制平面                       │    │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐          │    │
│  │  │ PV Controller│  │ PVC Controller│ │ Attach/Detach│       │    │
│  │  │             │  │              │  │ Controller   │        │    │
│  │  └─────────────┘  └─────────────┘  └─────────────┘          │    │
│  └─────────────────────────────────────────────────────────────┘    │
│                              │                                      │
│                              ▼                                      │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │              CSI Controller（Deployment）                    │    │
│  │  ┌─────────────────────────────────────────────────────────┐     │
│  │  │  Sidecar 容器                                            │     │
│  │  │  ├─ external-provisioner  → CreateVolume/DeleteVolume    │    │    
│  │  │  ├─ external-attacher     → ControllerPublish/Unpublish  │    │    
│  │  │  ├─ external-snapshotter  → CreateSnapshot/DeleteSnapshot│    │    
│  │  │  └─ external-resizer      → ControllerExpandVolume       │    │    
│  │  └─────────────────────────────────────────────────────────┘     │    
│  │  ┌─────────────────────────────────────────────────────┐    │    │
│  │  │  CSI Driver 容器                                     │    │    │
│  │  │  └─ 实现 CSI Controller Service                      │    │    │
│  │  └─────────────────────────────────────────────────────┘    │    │
│  └─────────────────────────────────────────────────────────────┘    │
│                              │                                      │
│                              ▼                                      │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │              CSI Node（DaemonSet，每个节点一个）               │    │
│  │  ┌─────────────────────────────────────────────────────┐    │    │
│  │  │  Sidecar 容器                                        │    │    │
│  │  │  └─ node-driver-registrar → 向 kubelet 注册驱动       │    │    │
│  │  └─────────────────────────────────────────────────────┘    │    │
│  │  ┌─────────────────────────────────────────────────────┐    │    │
│  │  │  CSI Driver 容器                                     │    │    │
│  │  │  └─ 实现 CSI Node Service                            │    │    │
│  │  │     ├─ NodeStageVolume   → 挂载到 staging 目录        │    │    │
│  │  │     ├─ NodePublishVolume → 挂载到 Pod 目录            │    │    │
│  │  │     └─ NodeGetInfo       → 返回节点拓扑信息            │    │    │
│  │  └─────────────────────────────────────────────────────┘    │    │
│  └─────────────────────────────────────────────────────────────┘    │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### 6.4 CSI 卷生命周期

```
┌─────────────────────────────────────────────────────────────────────┐
│                    CSI 卷完整生命周期                                 │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  1. 创建 PVC                                                         │
│     │                                                               │
│     ▼                                                               │
│  2. external-provisioner 监听到 PVC                                  │
│     │                                                               │
│     ▼                                                               │
│  3. 调用 CSI Driver CreateVolume()                                   │
│     │  → 在云平台创建存储卷（如 EBS）                                   │
│     ▼                                                               │
│  4. 创建 PV 对象，绑定 PVC                                            │
│     │                                                               │
│     ▼                                                               │
│  5. Pod 调度到节点                                                    │
│     │                                                               │
│     ▼                                                               │
│  6. external-attacher 调用 ControllerPublishVolume()                 │
│     │  → 将卷附加到节点（如 EBS Attach）                               │
│     ▼                                                               │
│  7. kubelet 调用 NodeStageVolume()                                   │
│     │  → 格式化并挂载到 staging 目录                                   │
│     ▼                                                               │
│  8. kubelet 调用 NodePublishVolume()                                 │
│     │  → bind mount 到 Pod 目录                                      │
│     ▼                                                               │
│  9. Pod 使用存储卷                                                    │
│                                                                     │
│  删除流程（逆向）：                                                     │
│  NodeUnpublishVolume → NodeUnstageVolume →                          │
│  ControllerUnpublishVolume → DeleteVolume                           │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### 6.5 主流 CSI 驱动对比

| 维度 | AWS EBS CSI | AWS EFS CSI | Ceph RBD CSI | Longhorn |
|------|-------------|-------------|--------------|----------|
| **存储类型** | 块存储 | 文件存储 | 块存储 | 块存储 |
| **访问模式** | RWO | RWX | RWO/RWX | RWO |
| **动态供应** | ✅ | ✅ | ✅ | ✅ |
| **快照** | ✅ | 🔴 | ✅ | ✅ |
| **扩容** | ✅ | ✅ | ✅ | ✅ |
| **加密** | ✅ KMS | ✅ 传输加密 | ✅ | ✅ |
| **性能** | 高（gp3: 16K IOPS） | 中 | 高 | 中 |
| **多 AZ** | 🔴 单 AZ | ✅ 多 AZ | ✅ | ✅ |
| **适用场景** | 数据库、有状态应用 | 共享存储、CMS | 自建存储 | 边缘/开发 |

### 6.6 卷快照（VolumeSnapshot）

VolumeSnapshot 是 CSI 提供的快照功能，用于数据备份和恢复。

**快照架构：**

```
┌─────────────────────────────────────────────────────────────────────┐
│                    VolumeSnapshot 架构                               │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  VolumeSnapshotClass       VolumeSnapshot     VolumeSnapshotContent │
│  （定义快照策略）              （用户创建）             （实际快照）       │
│        │                           │                        │       │
│        │                           │                        │       │
│        └───────────────────────────┼────────────────────────┘       │
│                                    │                                │
│                                    ▼                                │
│                          CSI Driver CreateSnapshot()                │
│                                    │                                │
│                                    ▼                                │
│                          云平台快照（如 EBS Snapshot）                 │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

**VolumeSnapshotClass 配置：**

```yaml
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshotClass
metadata:
  name: ebs-snapshot-class
driver: ebs.csi.aws.com
deletionPolicy: Delete  # Delete 或 Retain
```

**创建快照：**

```yaml
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshot
metadata:
  name: mysql-snapshot
spec:
  volumeSnapshotClassName: ebs-snapshot-class
  source:
    persistentVolumeClaimName: mysql-data  # 源 PVC
```

**从快照恢复（创建新 PVC）：**

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mysql-data-restored
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: ebs-gp3
  resources:
    requests:
      storage: 100Gi
  dataSource:
    name: mysql-snapshot
    kind: VolumeSnapshot
    apiGroup: snapshot.storage.k8s.io
```

### 6.7 临时卷（Ephemeral Volume）

临时卷的生命周期与 Pod 绑定，Pod 删除时卷也会被删除。

| 卷类型 | 说明 | 数据持久性 | 典型场景 |
|--------|------|------------|----------|
| emptyDir | 空目录，Pod 内容器共享 | Pod 删除即丢失 | 缓存、临时文件、Sidecar 共享 |
| configMap | 挂载 ConfigMap 数据 | 只读配置 | 应用配置文件 |
| secret | 挂载 Secret 数据 | 只读敏感数据 | 证书、密码 |
| downwardAPI | 暴露 Pod 元数据 | 只读 | 获取 Pod 名称、标签等 |
| projected | 组合多种卷源 | 只读 | 同时挂载 configMap + secret |

**emptyDir 配置示例：**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-with-cache
spec:
  containers:
  - name: app
    image: myapp:v1
    volumeMounts:
    - name: cache-volume
      mountPath: /cache
  - name: sidecar
    image: sidecar:v1
    volumeMounts:
    - name: cache-volume
      mountPath: /shared
  volumes:
  - name: cache-volume
    emptyDir:
      medium: Memory    # 使用内存（tmpfs），默认使用磁盘
      sizeLimit: 1Gi    # 大小限制
```

### 6.8 AWS EBS CSI 配置示例

```yaml
# StorageClass 配置
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: ebs-gp3-encrypted
provisioner: ebs.csi.aws.com
parameters:
  type: gp3
  iops: "3000"           # 基准 IOPS
  throughput: "125"      # 吞吐量 MB/s
  encrypted: "true"
  kmsKeyId: "arn:aws:kms:us-east-1:123456789:key/xxx"
volumeBindingMode: WaitForFirstConsumer
reclaimPolicy: Delete
allowVolumeExpansion: true
---
# PVC 配置
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mysql-data
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: ebs-gp3-encrypted
  resources:
    requests:
      storage: 100Gi
```

### 6.9 存储类型对比

| 类型 | 访问模式 | 性能特点 | 适用场景 | K8S 支持 |
|------|----------|----------|----------|----------|
| **块存储** | RWO | 低延迟、高 IOPS | 数据库、有状态应用 | PV/PVC |
| **文件存储** | RWX | 共享访问、中等性能 | CMS、共享配置 | PV/PVC |
| **对象存储** | N/A | 高吞吐、高延迟 | 备份、静态资源 | S3 API |

### 6.10 存储问题定位表

| 问题 | 症状 | 原因 | 排查命令 | 解决方案 |
|------|------|------|----------|----------|
| PVC Pending | PVC 一直 Pending | StorageClass 不存在 | `kubectl get sc` | 创建 StorageClass |
| PVC Pending | PVC 一直 Pending | CSI Driver 未安装 | `kubectl get pods -n kube-system` | 安装 CSI Driver |
| Pod 挂载失败 | Pod ContainerCreating | 卷未 Attach | `kubectl describe pod` | 检查 CSI Node 日志 |
| 卷扩容失败 | resize 不生效 | allowVolumeExpansion=false | `kubectl get sc -o yaml` | 修改 StorageClass |
| 跨 AZ 挂载失败 | Pod 调度失败 | EBS 单 AZ 限制 | `kubectl describe pv` | 使用 WaitForFirstConsumer |
| 性能不足 | I/O 延迟高 | IOPS/吞吐量不足 | `iostat -x 1` | 升级存储类型或 IOPS |

### 6.11 存储选型决策树

```
┌─────────────────────────────────────────────────────────────────────┐
│                    存储选型决策树                                     │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  是否需要多 Pod 同时读写？                                             │
│       │                                                             │
│       ├─── 是 ──────────────────────────────────────────────────┐   │
│       │                                                        │   │
│       │    → 文件存储（EFS、CephFS、NFS）                         │   │
│       │    → 访问模式：ReadWriteMany (RWX)                       │   │
│       │                                                        │   │
│       └─── 否 ──────────────────────────────────────────────────┐   │
│                                                                 │   │
│            是否需要高 IOPS/低延迟？                                │   │
│                 │                                               │   │
│                 ├─── 是 → 块存储（EBS gp3/io2、Ceph RBD）         │   │
│                 │        → 访问模式：ReadWriteOnce (RWO)         │   │
│                 │                                               │   │
│                 └─── 否 → 根据成本选择                            │   │
│                          ├─ 低成本 → EBS gp3                     │   │
│                          └─ 高性能 → EBS io2                     │   │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 7. Kubernetes 安全架构
### 7.1 安全模型概述（4C 模型）

```
┌─────────────────────────────────────────────────────────┐
│                    4C 安全模型                           │
│                                                         │
│  ┌─────────────────────────────────────────────────┐    │
│  │  Cloud（云平台安全）                              │    │
│  │  • IAM 权限、VPC 网络、安全组                      │    │
│  │  ┌─────────────────────────────────────────┐    │    │
│  │  │  Cluster（集群安全）                      │    │    │
│  │  │  • RBAC、NetworkPolicy、PSA              │    │    │
│  │  │  ┌─────────────────────────────────┐    │    │    │
│  │  │  │  Container（容器安全）            │    │    │    │
│  │  │  │  • 镜像扫描、Seccomp、AppArmor    │    │    │    │
│  │  │  │  ┌─────────────────────────┐    │    │    │    │
│  │  │  │  │  Code（代码安全）         │    │    │    │    │
│  │  │  │  │  • 依赖扫描、SAST         │    │    │    │    │
│  │  │  │  └─────────────────────────┘    │    │    │    │
│  │  │  └─────────────────────────────────┘    │    │    │
│  │  └─────────────────────────────────────────┘    │    │
│  └─────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────┘
```

**4C 安全模型详解：**

4C 模型是 Kubernetes 安全的分层防御框架，从外到内分为四层：Cloud → Cluster → Container → Code。每一层都需要独立的安全措施，形成纵深防御。

| 层级 | 安全范围 | 关键措施 | 责任方 |
|------|----------|----------|--------|
| **Cloud** | 云基础设施 | IAM 最小权限、VPC 隔离、安全组、加密 | 云平台/基础设施团队 |
| **Cluster** | K8S 集群 | RBAC、NetworkPolicy、PSA、审计日志 | 平台/SRE 团队 |
| **Container** | 容器运行时 | 镜像扫描、非 root 运行、Seccomp、只读文件系统 | 平台/安全团队 |
| **Code** | 应用代码 | 依赖扫描、SAST/DAST、Secret 管理 | 开发团队 |

**为什么需要分层防御？**

单一层级的安全措施无法提供完整保护。例如：
- 即使代码安全，容器镜像可能包含漏洞
- 即使容器安全，集群 RBAC 配置错误可能导致越权
- 即使集群安全，云 IAM 权限过大可能被利用

**各层级典型安全配置：**

| 层级 | 配置示例 |
|------|----------|
| Cloud | `aws:iam:policy` 最小权限、VPC 私有子网、Security Group 白名单 |
| Cluster | `NetworkPolicy` 默认拒绝、`PSA: restricted`、启用审计日志 |
| Container | `securityContext.runAsNonRoot: true`、`readOnlyRootFilesystem: true` |
| Code | Trivy 镜像扫描、Snyk 依赖扫描、External Secrets Operator |


### 7.2 Pod 安全标准（PSA）

K8S 1.25+ 使用 Pod Security Admission 替代 PodSecurityPolicy。

| 级别 | 说明 | 限制 |
|------|------|------|
| **privileged** | 无限制 | 允许所有特权操作 |
| **baseline** | 最小限制 | 禁止已知特权升级 |
| **restricted** | 严格限制 | 强制最佳安全实践 |

**配置示例：**
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/warn: restricted
    pod-security.kubernetes.io/audit: restricted
```

### 7.3 RBAC 详解

**RBAC 核心资源对比：**

| 资源 | 作用域 | 绑定资源 | 适用场景 |
|------|--------|----------|----------|
| **Role** | Namespace | RoleBinding | 命名空间内权限 |
| **ClusterRole** | Cluster | ClusterRoleBinding | 集群级权限 |
| **ClusterRole** | Cluster | RoleBinding | 跨命名空间复用角色 |

**RBAC 最佳实践示例：**
```yaml
# 最小权限原则：只读 Pod 权限
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: production
  name: pod-reader
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list", "watch"]  # 只读，无 create/delete
- apiGroups: [""]
  resources: ["pods/log"]
  verbs: ["get"]  # 允许查看日志
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-pods
  namespace: production
subjects:
- kind: ServiceAccount
  name: monitoring-sa
  namespace: production
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

**常用 RBAC 权限模板：**

| 角色 | 权限 | 适用场景 |
|------|------|---------|
| **只读用户** | get, list, watch | 监控、审计 |
| **开发者** | get, list, watch, create, update, delete（限定资源） | 开发环境 |
| **命名空间管理员** | 命名空间内所有权限 | 团队管理 |
| **集群管理员** | cluster-admin | 运维管理 |

### 7.4 运行时安全

| 技术 | 作用 | 说明 | 性能影响 |
|------|------|------|----------|
| **Seccomp** | 系统调用过滤 | 限制容器可用的系统调用 | 低 |
| **AppArmor** | 强制访问控制 | 限制文件/网络访问 | 低 |
| **SELinux** | 强制访问控制 | 基于标签的访问控制 | 中 |
| **gVisor** | 用户态内核 | 拦截系统调用，增强隔离 | 高（20-50%） |
| **Kata Containers** | 轻量级 VM | 每个容器运行在独立 VM | 高（启动慢） |

**Seccomp 配置示例：**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secure-pod
spec:
  securityContext:
    seccompProfile:
      type: RuntimeDefault  # 使用运行时默认配置
  containers:
  - name: app
    image: myapp:v1
    securityContext:
      allowPrivilegeEscalation: false
      readOnlyRootFilesystem: true
      runAsNonRoot: true
      runAsUser: 1000
      capabilities:
        drop:
        - ALL
```

### 7.5 Secret 管理

**K8S Secret vs 外部 Secret 管理对比：**

| 维度 | K8S Secret | External Secrets (ESO) | HashiCorp Vault |
|------|------------|------------------------|-----------------|
| **存储位置** | etcd（Base64） | 外部（AWS SM/SSM） | Vault 服务器 |
| **加密** | 可选（etcd 加密） | 外部服务加密 | 强加密 |
| **轮换** | 手动 | 自动同步 | 自动轮换 |
| **审计** | K8S 审计日志 | 外部服务审计 | 完整审计 |
| **复杂度** | 低 | 中 | 高 |
| **适用场景** | 简单场景 | AWS 环境 | 企业级 |

**External Secrets Operator 配置示例：**
```yaml
# SecretStore（连接 AWS Secrets Manager）
apiVersion: external-secrets.io/v1beta1
kind: SecretStore
metadata:
  name: aws-secrets-manager
  namespace: production
spec:
  provider:
    aws:
      service: SecretsManager
      region: us-east-1
      auth:
        jwt:
          serviceAccountRef:
            name: external-secrets-sa
---
# ExternalSecret（同步外部 Secret）
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: db-credentials
  namespace: production
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: aws-secrets-manager
    kind: SecretStore
  target:
    name: db-credentials
    creationPolicy: Owner
  data:
  - secretKey: username
    remoteRef:
      key: production/database
      property: username
  - secretKey: password
    remoteRef:
      key: production/database
      property: password
```

### 7.6 安全问题定位表

| 问题 | 症状 | 原因 | 排查命令 | 解决方案 |
|------|------|------|----------|----------|
| **权限不足** | 403 Forbidden | RBAC 配置错误 | `kubectl auth can-i <verb> <resource>` | 检查 Role/RoleBinding |
| **Pod 被 PSA 拒绝** | admission webhook denied | 违反 Pod 安全标准 | `kubectl describe pod` | 调整 securityContext |
| **Secret 泄露** | 敏感信息暴露 | Secret 未加密或权限过大 | 审计日志 | 使用 External Secrets |
| **镜像漏洞** | 安全扫描告警 | 基础镜像过旧 | `trivy image <image>` | 更新基础镜像 |
| **容器逃逸风险** | 安全审计告警 | 特权容器或 hostPath | `kubectl get pods -o yaml` | 禁用特权模式 |
| **网络暴露** | 未授权访问 | NetworkPolicy 缺失 | `kubectl get networkpolicy` | 配置 NetworkPolicy |
| **ServiceAccount 滥用** | 权限过大 | 使用默认 SA | `kubectl get sa` | 创建专用 SA |

**安全检查命令：**
```bash
# 检查当前用户权限
kubectl auth can-i --list

# 检查特定操作权限
kubectl auth can-i create pods -n production

# 检查 Pod 安全配置
kubectl get pods -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.securityContext}{"\n"}{end}'

# 检查特权容器
kubectl get pods -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.containers[*].securityContext.privileged}{"\n"}{end}'

# 检查 hostPath 挂载
kubectl get pods -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.volumes[*].hostPath}{"\n"}{end}'

# 扫描镜像漏洞
trivy image --severity HIGH,CRITICAL <image>

# 检查 RBAC 配置
kubectl get rolebindings,clusterrolebindings -A -o wide
```


---

## 8. Service Mesh 架构
### 8.1 Service Mesh 概述

Service Mesh 是微服务架构中的基础设施层，处理服务间通信。

```
┌─────────────────────────────────────────────────────────┐
│                Service Mesh 架构                         │
│                                                         │
│  ┌─────────────────────────────────────────────────┐    │
│  │              控制平面（Control Plane）            │    │
│  │  ┌──────────┐ ┌──────────┐ ┌──────────┐         │    │
│  │  │  Pilot   │ │  Citadel │ │  Galley  │         │    │
│  │  │ 流量管理  │ │ 证书管理   │ │ 配置管理  │         │    │
│  │  └──────────┘ └──────────┘ └──────────┘         │    │
│  └─────────────────────────────────────────────────┘    │
│                         │ 配置下发                       │
│                         ▼                               │
│  ┌─────────────────────────────────────────────────┐    │
│  │              数据平面（Data Plane）               │    │
│  │  ┌─────────────────┐  ┌─────────────────┐       │    │
│  │  │     Pod A       │  │     Pod B       │       │    │
│  │  │ ┌─────┐ ┌─────┐ │  │ ┌─────┐ ┌─────┐ │       │    │
│  │  │ │ App │ │Envoy│ │──│ │Envoy│ │ App │ │       │    │
│  │  │ └─────┘ └─────┘ │  │ └─────┘ └─────┘ │       │    │
│  │  └─────────────────┘  └─────────────────┘       │    │
│  └─────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────┘
```

**Service Mesh 架构核心组件说明：**

Service Mesh 将服务间通信的复杂性从应用代码中剥离，下沉到基础设施层。架构分为控制平面和数据平面两部分。

| 组件 | 所属平面 | 职责 | Istio 对应组件 |
|------|----------|------|----------------|
| **Pilot** | 控制平面 | 服务发现、流量管理规则下发 | istiod（1.5+ 合并） |
| **Citadel** | 控制平面 | 证书签发、mTLS 密钥管理 | istiod（1.5+ 合并） |
| **Galley** | 控制平面 | 配置验证和分发 | istiod（1.5+ 合并） |
| **Envoy Sidecar** | 数据平面 | 代理所有进出流量、执行策略 | istio-proxy |

**注意**：Istio 1.5+ 版本将 Pilot、Citadel、Galley 合并为单一的 `istiod` 组件，简化了部署和运维。上图展示的是逻辑架构，实际部署中只有 istiod 一个控制平面 Pod。

**数据平面工作原理：**

每个 Pod 中注入一个 Envoy Sidecar 代理，所有进出 Pod 的流量都经过 Sidecar。Sidecar 负责：
1. **流量拦截**：通过 iptables 规则拦截所有进出流量
2. **负载均衡**：根据 Pilot 下发的规则进行智能路由
3. **mTLS 加密**：自动加密服务间通信
4. **遥测采集**：自动采集请求指标、日志、追踪数据

**为什么需要 Service Mesh？**

| 传统方式 | Service Mesh |
|----------|--------------|
| 每个服务实现重试、熔断、限流 | 统一在 Sidecar 实现，应用无感知 |
| 服务间通信明文传输 | 自动 mTLS 加密 |
| 需要修改代码实现金丝雀发布 | 通过配置实现流量切分 |
| 分散的监控埋点 | 统一的可观测性数据采集 |

### 8.2 Istio 核心功能

| 功能 | 说明 | 实现方式 |
|------|------|---------|
| **流量管理** | 路由、负载均衡、熔断 | VirtualService, DestinationRule |
| **安全** | mTLS、授权策略 | PeerAuthentication, AuthorizationPolicy |
| **可观测性** | 指标、日志、追踪 | Envoy 自动采集 |


### 8.3 流量管理示例

**VirtualService（路由规则）：**
```yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: reviews
spec:
  hosts:
  - reviews
  http:
  - match:
    - headers:
        end-user:
          exact: jason
    route:
    - destination:
        host: reviews
        subset: v2
  - route:
    - destination:
        host: reviews
        subset: v1
```

**DestinationRule（目标规则）：**
```yaml
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: reviews
spec:
  host: reviews
  trafficPolicy:
    connectionPool:
      tcp:
        maxConnections: 100
  subsets:
  - name: v1
    labels:
      version: v1
  - name: v2
    labels:
      version: v2
```

### 8.4 Service Mesh 对比

| 维度 | Istio | Linkerd | AWS App Mesh | Cilium Service Mesh |
|------|-------|---------|--------------|---------------------|
| **数据平面** | Envoy | linkerd2-proxy | Envoy | eBPF（无 Sidecar） |
| **控制平面** | istiod | linkerd-control-plane | App Mesh Controller | Cilium Agent |
| **复杂度** | 高 | 低 | 中 | 中 |
| **资源开销** | 高（Sidecar） | 低 | 中 | 最低（无 Sidecar） |
| **功能丰富度** | 最全 | 核心功能 | AWS 集成 | 网络+安全 |
| **mTLS** | ✅ 自动 | ✅ 自动 | ✅ 支持 | ✅ 支持 |
| **流量管理** | 强大 | 基础 | 中等 | 基础 |
| **可观测性** | 优秀 | 良好 | CloudWatch 集成 | Hubble |
| **学习曲线** | 陡峭 | 平缓 | 中等 | 中等 |
| **适用场景** | 复杂微服务 | 简单场景 | AWS 原生 | 高性能场景 |

### 8.5 Service Mesh 选型决策

| 场景 | 推荐方案 | 理由 |
|------|----------|------|
| **复杂流量管理** | Istio | 功能最全面，支持复杂路由规则 |
| **简单 mTLS 需求** | Linkerd | 轻量级，资源开销低 |
| **AWS 原生应用** | App Mesh | 与 AWS 服务深度集成 |
| **高性能要求** | Cilium | eBPF 无 Sidecar，性能最优 |
| **已有 Cilium CNI** | Cilium Service Mesh | 统一网络和服务网格 |

### 8.6 Istio 常见问题定位

| 问题 | 症状 | 原因 | 解决方案 |
|------|------|------|----------|
| **Sidecar 注入失败** | Pod 无 Envoy 容器 | Namespace 未标记 | 添加 `istio-injection=enabled` 标签 |
| **mTLS 连接失败** | 503 错误 | PeerAuthentication 配置错误 | 检查 mTLS 模式配置 |
| **流量路由不生效** | 请求未按规则路由 | VirtualService 配置错误 | 检查 host 和 match 规则 |
| **延迟增加** | P99 延迟上升 | Envoy 资源不足 | 增加 Sidecar 资源限制 |
| **配置不同步** | 规则不生效 | istiod 问题 | 检查 istiod 日志和状态 |

**常用排查命令：**
```bash
# 检查 Istio 组件状态
kubectl get pods -n istio-system

# 检查 Sidecar 注入状态
kubectl get pods -o jsonpath='{.items[*].spec.containers[*].name}' | tr ' ' '\n' | grep istio-proxy

# 检查 Envoy 配置
istioctl proxy-config cluster <pod-name>
istioctl proxy-config route <pod-name>
istioctl proxy-config listener <pod-name>

# 分析配置问题
istioctl analyze

# 检查 mTLS 状态
istioctl authn tls-check <pod-name>
```


---

## 9. 可观测性架构
### 9.1 可观测性三大支柱

```
┌─────────────────────────────────────────────────────────┐
│                可观测性三大支柱                            │
│                                                         │
│  ┌──────────────┐ ┌──────────────┐ ┌──────────────┐     │
│  │   Metrics    │ │    Logs      │ │   Traces     │     │
│  │    指标       │ │    日志      │ │    追踪       │     │
│  │              │ │              │ │              │     │
│  │ • CPU/内存    │ │ • 应用日志    │ │ • 请求链路    │     │
│  │ • 请求延迟    │ │ • 系统日志    │ │ • 跨服务调用   │     │
│  │ • 错误率      │ │ • 审计日志    │ │ • 性能瓶颈    │     │
│  │              │ │              │ │              │     │
│  │ Prometheus   │ │ EFK/Loki     │ │ Jaeger/X-Ray │     │
│  └──────────────┘ └──────────────┘ └──────────────┘     │
└─────────────────────────────────────────────────────────┘
```

### 9.2 Prometheus 架构

```
┌─────────────────────────────────────────────────────────┐
│                 Prometheus 架构                          │
│                                                         │
│  ┌──────────────────────────────────────────────────┐   │
│  │                  Prometheus Server               │   │
│  │  ┌──────────┐ ┌──────────┐ ┌──────────┐          │   │
│  │  │ Retrieval│ │  TSDB    │ │ HTTP API │          │   │
│  │  │ 数据采集  │ │ 时序存储   │ │ 查询接口  │          │   │
│  │  └──────────┘ └──────────┘ └──────────┘          │   │
│  └──────────────────────────────────────────────────┘   │
│         ↑                              ↓                │
│    Pull 模式                      PromQL 查询            │
│         ↑                              ↓                │
│  ┌──────────────┐              ┌──────────────┐         │
│  │   Targets    │              │   Grafana    │         │
│  │ • Pod        │              │   可视化      │         │
│  │ • Node       │              └──────────────┘         │
│  │ • Service    │              ┌──────────────┐         │
│  └──────────────┘              │ Alertmanager │         │
│                                │   告警管理    │         │
│                                └──────────────┘         │
└─────────────────────────────────────────────────────────┘
```

**Prometheus 架构核心组件说明：**

Prometheus 是 CNCF 毕业项目，是 Kubernetes 生态中最流行的监控系统。其核心设计是 Pull 模式采集和时序数据库存储。

| 组件 | 职责 | 关键特性 |
|------|------|----------|
| **Retrieval** | 从 Targets 拉取指标 | 支持服务发现（K8S、Consul、DNS） |
| **TSDB** | 时序数据存储 | 高效压缩、本地存储、支持远程写入 |
| **HTTP API** | 提供查询接口 | PromQL 查询语言 |
| **Alertmanager** | 告警管理 | 分组、抑制、静默、路由 |
| **Grafana** | 可视化 | Dashboard、告警、数据源聚合 |
| **Pushgateway** | 接收推送指标 | 用于短生命周期 Job（批处理任务） |

**为什么 Prometheus 使用 Pull 模式？**

| Pull 模式优势 | 说明 |
|--------------|------|
| **简化配置** | Target 只需暴露 `/metrics` 端点，无需配置推送目标 |
| **健康检查** | 拉取失败即表示 Target 不健康 |
| **避免过载** | Prometheus 控制采集频率，不会被 Target 推送压垮 |
| **易于调试** | 可以直接访问 Target 的 `/metrics` 端点查看原始数据 |

**Pull 模式的例外 - Pushgateway：**

对于短生命周期的批处理任务（如 CronJob），任务可能在 Prometheus 下次拉取前就已结束。这种情况下使用 Pushgateway：任务将指标推送到 Pushgateway，Prometheus 从 Pushgateway 拉取。

**Kubernetes 服务发现机制：**

Prometheus 通过 Kubernetes API 自动发现需要监控的 Targets：
- `kubernetes_sd_configs` 配置服务发现
- 支持发现 Pod、Service、Node、Endpoints、Ingress
- 通过 `relabel_configs` 过滤和重命名标签


### 9.3 K8S 指标采集配置

**ServiceMonitor（Prometheus Operator）：**
```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: app-monitor
spec:
  selector:
    matchLabels:
      app: myapp
  endpoints:
  - port: metrics
    interval: 30s
    path: /metrics
```

### 9.4 日志架构

| 方案 | 组件 | 特点 |
|------|------|------|
| **EFK** | Elasticsearch + Fluentd + Kibana | 功能全面，资源消耗大 |
| **ELK** | Elasticsearch + Logstash + Kibana | 传统方案 |
| **PLG** | Promtail + Loki + Grafana | 轻量级，与 Prometheus 集成 |
| **CloudWatch** | Fluent Bit + CloudWatch Logs | AWS 原生 |

### 9.5 分布式追踪

| 方案 | 协议 | 特点 |
|------|------|------|
| **Jaeger** | OpenTracing/OpenTelemetry | CNCF 项目，功能全面 |
| **Zipkin** | B3 | 轻量级 |
| **AWS X-Ray** | X-Ray SDK | AWS 原生集成 |
| **OpenTelemetry** | OTLP | 统一标准，推荐 |

### 9.6 可观测性方案对比

| 维度 | Prometheus + Grafana | Datadog | AWS CloudWatch | New Relic |
|------|---------------------|---------|----------------|-----------|
| **部署方式** | 自建 | SaaS | AWS 托管 | SaaS |
| **指标采集** | Pull 模式 | Agent | Agent/API | Agent |
| **日志支持** | 需要 Loki | 原生支持 | 原生支持 | 原生支持 |
| **追踪支持** | 需要 Jaeger | 原生支持 | X-Ray 集成 | 原生支持 |
| **告警** | Alertmanager | 原生支持 | CloudWatch Alarms | 原生支持 |
| **成本** | 低（自建） | 高 | 中 | 高 |
| **学习曲线** | 中 | 低 | 低 | 低 |
| **K8S 集成** | 优秀 | 优秀 | 良好 | 优秀 |
| **适用场景** | 成本敏感 | 企业级 | AWS 原生 | 企业级 |

### 9.7 常用 PromQL 查询

```promql
# CPU 使用率（按 Pod）
sum(rate(container_cpu_usage_seconds_total{namespace="production"}[5m])) by (pod)

# 内存使用率（按 Pod）
sum(container_memory_working_set_bytes{namespace="production"}) by (pod) / 
sum(container_spec_memory_limit_bytes{namespace="production"}) by (pod) * 100

# 请求延迟 P99
histogram_quantile(0.99, sum(rate(http_request_duration_seconds_bucket[5m])) by (le, service))

# 错误率
sum(rate(http_requests_total{status=~"5.."}[5m])) / sum(rate(http_requests_total[5m])) * 100

# Pod 重启次数
sum(kube_pod_container_status_restarts_total{namespace="production"}) by (pod)

# 节点 CPU 使用率
100 - (avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)

# 节点内存使用率
(1 - node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes) * 100
```

### 9.8 可观测性问题定位表

| 问题 | 症状 | 排查方向 | 工具/命令 |
|------|------|----------|----------|
| **高延迟** | P99 延迟增加 | 检查资源使用、依赖服务 | Prometheus + Jaeger |
| **高错误率** | 5xx 错误增加 | 检查应用日志、依赖服务 | Loki/EFK + 告警 |
| **Pod 重启** | 重启次数增加 | 检查 OOM、探针失败 | `kubectl describe pod` |
| **资源不足** | CPU/内存告警 | 检查资源使用趋势 | Grafana 仪表板 |
| **网络问题** | 连接超时 | 检查 DNS、Service | `kubectl exec -- curl` |
| **存储问题** | I/O 延迟高 | 检查 PV 性能 | `kubectl top pods` |


---

## 10. GitOps 与 CI/CD
### 10.1 GitOps 原则

| 原则 | 说明 |
|------|------|
| **声明式** | 系统状态以声明式配置存储 |
| **版本控制** | Git 作为唯一真实来源 |
| **自动同步** | 自动将 Git 状态同步到集群 |
| **持续调谐** | 持续检测并修复漂移 |

### 10.2 ArgoCD 架构

```
┌─────────────────────────────────────────────────────────┐
│                   ArgoCD 架构                            │
│                                                         │
│  ┌──────────────────────────────────────────────────┐   │
│  │                  ArgoCD Server                   │   │
│  │  ┌──────────┐ ┌──────────┐ ┌───────────┐         │   │
│  │  │ API      │ │ Repo     │ │Application│         │   │
│  │  │ Server   │ │ Server   │ │ Controller│         │   │
│  │  └──────────┘ └──────────┘ └───────────┘         │   │
│  └──────────────────────────────────────────────────┘   │
│         ↑                              ↓                │
│    Git Repo                      K8S Cluster            │
│    (期望状态)                    (实际状态)                │
│                                                         │
│  同步流程：                                               │
│  1. 监听 Git 仓库变更                                     │
│  2. 比较期望状态与实际状态                                  │
│  3. 自动/手动同步差异                                      │
│  4. 持续监控漂移                                          │
└─────────────────────────────────────────────────────────┘
```

**ArgoCD 架构核心组件说明：**

ArgoCD 是 CNCF 毕业的 GitOps 持续交付工具，实现了"Git 作为唯一真实来源"的理念。其核心是将 Git 仓库中的声明式配置自动同步到 Kubernetes 集群。

| 组件 | 职责 | 关键功能 |
|------|------|----------|
| **API Server** | 提供 Web UI 和 CLI 接口 | 用户交互、RBAC、SSO 集成 |
| **Repo Server** | 管理 Git 仓库连接 | 克隆仓库、渲染 Helm/Kustomize |
| **Application Controller** | 核心调谐循环 | 比较期望/实际状态、执行同步 |
| **Redis** | 缓存层 | 缓存 Git 仓库状态、应用状态 |
| **Dex**（可选） | 身份认证 | SSO 集成（OIDC、LDAP、SAML） |

**GitOps 同步流程详解：**

1. **检测变更**：Application Controller 定期（默认 3 分钟）检查 Git 仓库
2. **渲染配置**：Repo Server 将 Helm Chart/Kustomize 渲染为原始 YAML
3. **状态比较**：比较 Git 中的期望状态与集群中的实际状态
4. **同步执行**：根据同步策略（自动/手动）应用差异
5. **漂移检测**：持续监控集群状态，检测手动修改导致的漂移

**为什么选择 GitOps？**

| 传统 CI/CD | GitOps（ArgoCD） |
|------------|------------------|
| CI 工具直接 `kubectl apply` | Git 提交触发同步 |
| 难以追踪"谁在什么时候改了什么" | Git 历史即审计日志 |
| 集群状态可能与配置不一致 | 持续调谐保证一致性 |
| 回滚需要重新部署 | `git revert` 即可回滚 |

**Application 配置示例：**
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: myapp
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/org/repo.git
    targetRevision: HEAD
    path: k8s/overlays/prod
  destination:
    server: https://kubernetes.default.svc
    namespace: production
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```


### 10.3 ArgoCD vs Flux 对比

| 维度 | ArgoCD | Flux |
|------|--------|------|
| UI | ✅ 内置 Web UI | ❌ 需要额外部署 |
| 多集群 | ✅ 原生支持 | ✅ 支持 |
| Helm 支持 | ✅ 原生 | ✅ HelmRelease CRD |
| Kustomize | ✅ 原生 | ✅ Kustomization CRD |
| RBAC | ✅ 细粒度 | 依赖 K8S RBAC |
| 资源消耗 | 较高 | 较低 |
| 学习曲线 | 中等 | 较低 |

### 10.4 渐进式交付（Argo Rollouts）

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: myapp
spec:
  replicas: 10
  strategy:
    canary:
      steps:
      - setWeight: 10      # 10% 流量到新版本
      - pause: {duration: 5m}
      - setWeight: 30
      - pause: {duration: 5m}
      - setWeight: 50
      - pause: {duration: 5m}
      - setWeight: 100
      analysis:
        templates:
        - templateName: success-rate
        startingStep: 1
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
      - name: app
        image: myapp:v2
```

### 10.5 CI/CD 工具对比

| 维度 | Jenkins | GitHub Actions | GitLab CI | ArgoCD |
|------|---------|----------------|-----------|--------|
| **类型** | CI/CD | CI/CD | CI/CD | CD (GitOps) |
| **部署方式** | 自建 | SaaS | 自建/SaaS | 自建 |
| **配置方式** | Jenkinsfile | YAML | YAML | CRD |
| **K8S 集成** | 插件 | 原生 | 原生 | 原生 |
| **学习曲线** | 高 | 低 | 中 | 中 |
| **扩展性** | 插件丰富 | Marketplace | 内置 | Webhook |
| **成本** | 自建成本 | 免费额度 | 免费额度 | 开源免费 |
| **适用场景** | 企业级 | 开源项目 | GitLab 用户 | K8S 原生 |

### 10.6 GitOps 最佳实践

**目录结构示例：**
```
├── apps/
│   ├── base/                    # 基础配置
│   │   ├── deployment.yaml
│   │   ├── service.yaml
│   │   └── kustomization.yaml
│   └── overlays/
│       ├── dev/                 # 开发环境
│       │   ├── kustomization.yaml
│       │   └── patch.yaml
│       ├── staging/             # 预发环境
│       │   ├── kustomization.yaml
│       │   └── patch.yaml
│       └── prod/                # 生产环境
│           ├── kustomization.yaml
│           └── patch.yaml
├── infrastructure/              # 基础设施配置
│   ├── cert-manager/
│   ├── ingress-nginx/
│   └── monitoring/
└── argocd/                      # ArgoCD 配置
    ├── projects/
    └── applications/
```

### 10.7 GitOps 问题定位表

| 问题 | 症状 | 原因 | 解决方案 |
|------|------|------|----------|
| **同步失败** | OutOfSync 状态 | Git 配置错误或权限问题 | 检查 Git 仓库访问权限 |
| **资源漂移** | 手动修改被覆盖 | selfHeal 启用 | 通过 Git 提交修改 |
| **Helm 渲染失败** | 应用部署失败 | values 文件错误 | 本地 helm template 验证 |
| **镜像拉取失败** | ImagePullBackOff | 镜像不存在或认证失败 | 检查镜像名和 imagePullSecrets |
| **健康检查失败** | Degraded 状态 | 应用启动失败 | 检查 Pod 日志和事件 |

**常用排查命令：**
```bash
# 检查 ArgoCD 应用状态
argocd app list
argocd app get <app-name>

# 查看同步历史
argocd app history <app-name>

# 手动同步
argocd app sync <app-name>

# 查看差异
argocd app diff <app-name>

# 回滚到上一版本
argocd app rollback <app-name>

# 检查 ArgoCD 日志
kubectl logs -n argocd -l app.kubernetes.io/name=argocd-application-controller
```


---

## 11. Kubernetes 工作负载
### 11.1 Pod 生命周期状态机

```
┌─────────────────────────────────────────────────────────┐
│                     Pod 状态机                           │
│                                                         │
│                    ┌──────────┐                         │
│                    │ Pending  │                         │
│                    └────┬─────┘                         │
│                         │ 调度成功 + 镜像拉取完成          │
│                         ▼                               │
│                    ┌──────────┐                         │
│          ┌─────────│ Running  │─────────┐               │
│          │         └──────────┘         │               │
│          │              │               │               │
│    容器正常退出     容器异常退出       容器被终止             │
│    (exit 0)       (exit != 0)     (SIGTERM)             │
│          │              │               │               │
│          ▼              ▼               ▼               │
│    ┌──────────┐  ┌──────────┐    ┌──────────┐           │
│    │Succeeded │  │  Failed  │    │  Failed  │           │
│    └──────────┘  └──────────┘    └──────────┘           │
│                                                         │
│    ┌──────────┐                                         │
│    │ Unknown  │ ← 节点失联，状态未知                       │
│    └──────────┘                                         │
└─────────────────────────────────────────────────────────┘
```

**Pod 状态说明：**

| 状态 | 说明 |
|------|------|
| **Pending** | Pod 已创建，等待调度或镜像拉取 |
| **Running** | Pod 已绑定节点，至少一个容器运行中 |
| **Succeeded** | 所有容器正常退出（exit 0） |
| **Failed** | 至少一个容器异常退出 |
| **Unknown** | 无法获取 Pod 状态（通常是节点失联） |

### 11.2 工作负载类型对比

| 工作负载 | 用途 | Pod 标识 | 存储 | 扩缩容 | 适用场景 |
|----------|------|----------|------|--------|----------|
| **Deployment** | 无状态应用 | 随机名称 | 共享 | 任意顺序 | Web 服务、API |
| **StatefulSet** | 有状态应用 | 有序名称（-0, -1, -2） | 独立 PVC | 有序扩缩 | 数据库、消息队列 |
| **DaemonSet** | 每节点一个 | 节点名称后缀 | 可选 | 随节点 | 日志采集、监控 |
| **Job** | 一次性任务 | 随机名称 | 可选 | N/A | 批处理、数据迁移 |
| **CronJob** | 定时任务 | 随机名称 | 可选 | N/A | 定时备份、报表 |
| **ReplicaSet** | 副本管理 | 随机名称 | 共享 | 任意顺序 | 被 Deployment 管理 |

### 11.3 Deployment 详解

**Deployment 更新策略：**

| 策略 | 说明 | 参数 |
|------|------|------|
| **RollingUpdate** | 滚动更新（默认） | maxSurge, maxUnavailable |
| **Recreate** | 先删除再创建 | 无 |

**Deployment 配置示例：**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1          # 最多多创建 1 个 Pod
      maxUnavailable: 0    # 不允许不可用 Pod
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
      - name: web
        image: nginx:1.21
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            cpu: 500m
            memory: 512Mi
        readinessProbe:
          httpGet:
            path: /health
            port: 80
          initialDelaySeconds: 5
          periodSeconds: 10
```

### 11.4 StatefulSet 详解

**StatefulSet 特性：**
- 稳定的网络标识（pod-0, pod-1, pod-2）
- 稳定的存储（每个 Pod 独立 PVC）
- 有序部署和扩缩容
- 有序滚动更新

**StatefulSet 配置示例：**
```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql
spec:
  serviceName: mysql-headless  # Headless Service
  replicas: 3
  podManagementPolicy: OrderedReady  # 有序启动
  updateStrategy:
    type: RollingUpdate
    rollingUpdate:
      partition: 0  # 从 partition 开始更新
  selector:
    matchLabels:
      app: mysql
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
      - name: mysql
        image: mysql:8.0
        volumeMounts:
        - name: data
          mountPath: /var/lib/mysql
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      storageClassName: ebs-gp3
      resources:
        requests:
          storage: 100Gi
```

### 11.5 工作负载问题定位表

| 问题 | 症状 | 原因 | 解决方案 |
|------|------|------|----------|
| **Deployment 卡住** | 更新进度停滞 | Pod 无法就绪 | 检查 Pod 日志和事件 |
| **StatefulSet 启动慢** | Pod 依次启动 | OrderedReady 策略 | 使用 Parallel 策略（如适用） |
| **DaemonSet 未调度** | 部分节点无 Pod | Taint 或资源不足 | 添加 Tolerations |
| **Job 失败** | backoffLimit 达到 | 任务执行失败 | 检查 Pod 日志 |
| **CronJob 未触发** | 任务未执行 | schedule 语法错误 | 验证 cron 表达式 |


---

## 12. Kubernetes 调度约束
### 12.1 nodeSelector（最简单的节点选择）

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: gpu-pod
spec:
  nodeSelector:
    accelerator: nvidia-tesla-v100  # 必须精确匹配节点标签
  containers:
  - name: cuda-container
    image: nvidia/cuda:11.0-base
```

**nodeSelector vs Node Affinity：**

| 维度   | nodeSelector | Node Affinity                           |
| ---- | ------------ | --------------------------------------- |
| 语法   | 简单键值对        | 复杂表达式                                   |
| 操作符  | 只支持 =        | In, NotIn, Exists, DoesNotExist, Gt, Lt |
| 软约束  | 🔴 不支持       | ✅ preferred 支持                          |
| 适用场景 | 简单场景         | 复杂调度需求                                  |

### 12.2 Node Affinity（节点亲和性）

Node Affinity 提供比 nodeSelector 更灵活的节点选择能力，支持软约束和复杂表达式。

**Node Affinity 类型：**

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Node Affinity 类型对比                            │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  requiredDuringSchedulingIgnoredDuringExecution（硬约束）            │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │ • 必须满足，否则 Pod 不会被调度                                 │    │
│  │ • 类似 nodeSelector，但支持更复杂的表达式                       │    │
│  │ • IgnoredDuringExecution: 运行中的 Pod 不受影响               │    │
│  └─────────────────────────────────────────────────────────────┘    │
│                                                                     │
│  preferredDuringSchedulingIgnoredDuringExecution（软约束）            │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │ • 优先满足，但不强制                                           │    │
│  │ • 通过 weight（1-100）设置优先级                               │    │
│  │ • 调度器会尽量满足，但资源不足时可忽略                            │    │
│  └─────────────────────────────────────────────────────────────┘    │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

**Node Affinity 操作符：**

| 操作符 | 说明 | 示例 |
|--------|------|------|
| In | 值在列表中 | `values: [v100, a100]` |
| NotIn | 值不在列表中 | `values: [spot]` |
| Exists | 标签存在（不检查值） | `key: gpu` |
| DoesNotExist | 标签不存在 | `key: spot` |
| Gt | 值大于（数值比较） | `values: ["4"]` |
| Lt | 值小于（数值比较） | `values: ["8"]` |

**完整示例：**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: node-affinity-demo
spec:
  affinity:
    nodeAffinity:
      # 硬约束：必须是 amd64 架构
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: kubernetes.io/arch
            operator: In
            values: ["amd64"]
          - key: node.kubernetes.io/instance-type
            operator: In
            values: ["m5.xlarge", "m5.2xlarge", "m6i.xlarge"]
      # 软约束：优先选择 us-east-1a
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 80
        preference:
          matchExpressions:
          - key: topology.kubernetes.io/zone
            operator: In
            values: ["us-east-1a"]
      - weight: 20
        preference:
          matchExpressions:
          - key: topology.kubernetes.io/zone
            operator: In
            values: ["us-east-1b"]
  containers:
  - name: app
    image: nginx
```

**调度器评分计算：**

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Node Affinity 评分计算                            │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  假设有 3 个节点：                                                    │
│  • Node-A: zone=us-east-1a, arch=amd64, type=m5.xlarge              │
│  • Node-B: zone=us-east-1b, arch=amd64, type=m5.2xlarge             │
│  • Node-C: zone=us-east-1c, arch=arm64, type=m6g.xlarge             │
│                                                                     │
│  评分过程：                                                          │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │ 1. 硬约束过滤：                                               │    │
│  │    Node-A: ✅ 通过（amd64 + m5.xlarge）                      │    │
│  │    Node-B: ✅ 通过（amd64 + m5.2xlarge）                     │    │
│  │    Node-C: ❌ 过滤（arm64 不匹配）                            │    │
│  │                                                             │    │
│  │ 2. 软约束评分：                                               │    │
│  │    Node-A: 80 分（匹配 us-east-1a，weight=80）                │    │
│  │    Node-B: 20 分（匹配 us-east-1b，weight=20）                │    │
│  │                                                             │    │
│  │ 3. 最终选择：Node-A（得分最高）                                 │    │
│  └─────────────────────────────────────────────────────────────┘    │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### 12.3 Pod Affinity/Anti-Affinity（Pod 亲和性）

Pod Affinity 用于将 Pod 调度到与指定 Pod 相同拓扑域的节点，Anti-Affinity 则相反。

**核心概念：**

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Pod Affinity 核心概念                             │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  topologyKey（拓扑域）：                                               │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │ • kubernetes.io/hostname     → 同一节点                      │    │
│  │ • topology.kubernetes.io/zone → 同一可用区                   │    │
│  │ • topology.kubernetes.io/region → 同一区域                   │    │
│  │ • 自定义标签（如 rack、room）                                  │    │
│  └─────────────────────────────────────────────────────────────┘    │
│                                                                     │
│  Pod Affinity vs Anti-Affinity：                                    │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │ Affinity:     调度到【有】匹配 Pod 的拓扑域                     │    │
│  │ Anti-Affinity: 调度到【没有】匹配 Pod 的拓扑域                  │    │
│  └─────────────────────────────────────────────────────────────┘    │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

**典型场景：**

| 场景 | 类型 | topologyKey | 说明 |
|------|------|-------------|------|
| Web 与 Cache 同节点 | Affinity | hostname | 减少网络延迟 |
| 副本跨 AZ 分布 | Anti-Affinity | zone | 高可用 |
| 副本跨节点分布 | Anti-Affinity | hostname | 避免单点故障 |
| 同机房部署 | Affinity | rack | 低延迟通信 |

**完整示例 - Web 与 Redis 同节点部署：**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-server
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      affinity:
        # Pod Affinity: 与 Redis 同节点
        podAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchLabels:
                app: redis
            topologyKey: kubernetes.io/hostname
        # Pod Anti-Affinity: Web 副本分散到不同节点
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchLabels:
                  app: web
              topologyKey: kubernetes.io/hostname
      containers:
      - name: web
        image: nginx
```

**调度流程图：**

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Pod Affinity 调度流程                             │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  集群状态：                                                           │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐               │
│  │   Node-1     │  │   Node-2     │  │   Node-3     │               │
│  │  zone: a     │  │  zone: a     │  │  zone: b     │               │
│  │              │  │              │  │              │               │
│  │ [Redis-1]    │  │ [Redis-2]    │  │              │               │
│  │              │  │              │  │              │               │
│  └──────────────┘  └──────────────┘  └──────────────┘               │
│                                                                     │
│  调度 Web Pod（要求与 Redis 同节点）：                                   │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │ 1. 查找所有运行 app=redis 的 Pod                              │    │
│  │ 2. 获取这些 Pod 所在节点的 topologyKey 值                      │    │
│  │    → Node-1, Node-2（有 Redis）                              │    │
│  │ 3. 只考虑这些节点作为候选                                       │    │
│  │ 4. Node-3 被排除（没有 Redis）                                │    │
│  └─────────────────────────────────────────────────────────────┘    │
│                                                                     │
│  结果：                                                              │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐               │
│  │   Node-1     │  │   Node-2     │  │   Node-3     │               │
│  │ [Redis-1]    │  │ [Redis-2]    │  │              │               │
│  │ [Web-1] ✅   │  │ [Web-2] ✅   │  │ ❌ 不调度    │               │
│  └──────────────┘  └──────────────┘  └──────────────┘               │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

**性能注意事项：**

| 注意点 | 说明 | 建议 |
|--------|------|------|
| 计算开销 | Pod Affinity 需要遍历所有 Pod | 大集群慎用硬约束 |
| 调度延迟 | 复杂规则增加调度时间 | 限制 labelSelector 范围 |
| 死锁风险 | 互相依赖可能导致无法调度 | 避免循环依赖 |

### 12.4 Taints 与 Tolerations（污点与容忍）

Taints 作用于节点，阻止 Pod 调度；Tolerations 作用于 Pod，允许调度到有污点的节点。

**工作机制：**

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Taints & Tolerations 工作机制                     │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Taint（污点）- 作用于 Node：                                          │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │ kubectl taint nodes node1 key=value:effect                  │    │
│  │                                                             │    │
│  │ 组成部分：                                                    │    │
│  │ • key:   污点键（如 dedicated、gpu）                          │    │
│  │ • value: 污点值（如 database、nvidia）                        │    │
│  │ • effect: 效果（NoSchedule/PreferNoSchedule/NoExecute）      │    │
│  └─────────────────────────────────────────────────────────────┘    │
│                                                                     │
│  Toleration（容忍）- 作用于 Pod：                                      │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │ tolerations:                                                │    │
│  │ - key: "key"                                                │    │
│  │   operator: "Equal"  # 或 "Exists"                          │    │
│  │   value: "value"                                            │    │
│  │   effect: "NoSchedule"                                      │    │
│  └─────────────────────────────────────────────────────────────┘    │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

**Effect 类型详解：**

| Effect           | 调度阶段    | 运行阶段  | 说明                      |
| ---------------- | ------- | ----- | ----------------------- |
| NoSchedule       | 🔴 禁止调度 | ✅ 不影响 | 新 Pod 不会调度，已运行 Pod 不受影响 |
| PreferNoSchedule | ⚠️ 尽量避免 | ✅ 不影响 | 软约束，资源不足时仍可调度           |
| NoExecute        | 🔴 禁止调度 | 🔴 驱逐 | 新 Pod 不调度，已运行 Pod 被驱逐   |

**Operator 匹配规则：**

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Toleration 匹配规则                               │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Equal 操作符（精确匹配）：                                             │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │ Taint:      dedicated=database:NoSchedule                   │    │
│  │ Toleration: key=dedicated, value=database, effect=NoSchedule│    │
│  │ 结果:       ✅ 匹配（key + value + effect 都相同）             │    │
│  └─────────────────────────────────────────────────────────────┘    │
│                                                                     │
│  Exists 操作符（只匹配 key）：                                          │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │ Taint:      dedicated=database:NoSchedule                    │   │
│  │ Toleration: key=dedicated, operator=Exists, effect=NoSchedule│   │
│  │ 结果:       ✅ 匹配（只要 key 存在，不检查 value）                │   │
│  └──────────────────────────────────────────────────────────────┘   │
│                                                                     │
│  空 key + Exists（容忍所有污点）：                                      │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │ Toleration: operator=Exists                                 │    │
│  │ 结果:       ✅ 匹配所有污点（通常用于 DaemonSet）                │    │
│  └─────────────────────────────────────────────────────────────┘    │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

**典型使用场景：**

```yaml
# 场景 1: 专用节点（数据库专用）
# 给节点打污点
# kubectl taint nodes db-node-1 dedicated=database:NoSchedule

apiVersion: v1
kind: Pod
metadata:
  name: mysql
spec:
  tolerations:
  - key: "dedicated"
    operator: "Equal"
    value: "database"
    effect: "NoSchedule"
  nodeSelector:
    dedicated: database  # 配合 nodeSelector 确保调度到专用节点
  containers:
  - name: mysql
    image: mysql:8.0
---
# 场景 2: GPU 节点
# kubectl taint nodes gpu-node-1 nvidia.com/gpu=present:NoSchedule

apiVersion: v1
kind: Pod
metadata:
  name: ml-training
spec:
  tolerations:
  - key: "nvidia.com/gpu"
    operator: "Exists"
    effect: "NoSchedule"
  containers:
  - name: training
    image: tensorflow/tensorflow:latest-gpu
    resources:
      limits:
        nvidia.com/gpu: 1
---
# 场景 3: DaemonSet 容忍所有污点（如日志收集）
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluentd
spec:
  selector:
    matchLabels:
      name: fluentd
  template:
    metadata:
      labels:
        name: fluentd
    spec:
      tolerations:
      - operator: "Exists"  # 容忍所有污点，确保每个节点都运行
      containers:
      - name: fluentd
        image: fluent/fluentd
```

**NoExecute 与 tolerationSeconds：**

```yaml
# NoExecute 会驱逐已运行的 Pod
# tolerationSeconds 可以设置驱逐前的等待时间

apiVersion: v1
kind: Pod
metadata:
  name: critical-app
spec:
  tolerations:
  - key: "node.kubernetes.io/not-ready"
    operator: "Exists"
    effect: "NoExecute"
    tolerationSeconds: 300  # 节点 NotReady 后等待 5 分钟再驱逐
  - key: "node.kubernetes.io/unreachable"
    operator: "Exists"
    effect: "NoExecute"
    tolerationSeconds: 300
  containers:
  - name: app
    image: nginx
```

**内置污点（Kubernetes 自动添加）：**

| 污点 | 触发条件 | 默认 tolerationSeconds |
|------|----------|------------------------|
| node.kubernetes.io/not-ready | 节点 NotReady | 300s |
| node.kubernetes.io/unreachable | 节点不可达 | 300s |
| node.kubernetes.io/memory-pressure | 内存压力 | - |
| node.kubernetes.io/disk-pressure | 磁盘压力 | - |
| node.kubernetes.io/pid-pressure | PID 压力 | - |
| node.kubernetes.io/network-unavailable | 网络不可用 | - |
| node.kubernetes.io/unschedulable | 节点被 cordon | - |

**常用命令：**

```bash
# 添加污点
kubectl taint nodes node1 dedicated=database:NoSchedule

# 查看节点污点
kubectl describe node node1 | grep Taints

# 删除污点（在 key 后加 -）
kubectl taint nodes node1 dedicated=database:NoSchedule-

# 删除某个 key 的所有污点
kubectl taint nodes node1 dedicated-
```

### 12.5 topologySpreadConstraints（拓扑分布约束）

确保 Pod 在拓扑域（AZ、节点）间均匀分布：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
spec:
  replicas: 6
  template:
    spec:
      topologySpreadConstraints:
      - maxSkew: 1                              # 最大不均衡度
        topologyKey: topology.kubernetes.io/zone # 按 AZ 分布
        whenUnsatisfiable: DoNotSchedule        # 硬约束
        labelSelector:
          matchLabels:
            app: web
      - maxSkew: 1
        topologyKey: kubernetes.io/hostname     # 按节点分布
        whenUnsatisfiable: ScheduleAnyway       # 软约束
        labelSelector:
          matchLabels:
            app: web
```


**topologySpreadConstraints 参数说明：**

| 参数 | 说明 |
|------|------|
| **maxSkew** | 允许的最大不均衡度（各拓扑域 Pod 数差值） |
| **topologyKey** | 拓扑域标签（zone/hostname/region） |
| **whenUnsatisfiable** | DoNotSchedule（硬约束）/ ScheduleAnyway（软约束） |
| **labelSelector** | 计算分布时考虑的 Pod 标签 |

**分布效果示例（6 副本，3 AZ，maxSkew=1）：**
```
us-east-1a: 2 个 Pod
us-east-1b: 2 个 Pod
us-east-1c: 2 个 Pod
```

### 12.6 调度约束组合最佳实践

| 场景 | 推荐组合 |
|------|---------|
| **高可用部署** | topologySpreadConstraints（跨 AZ）+ podAntiAffinity（跨节点） |
| **GPU 专用节点** | nodeSelector/nodeAffinity + Taints/Tolerations |
| **数据本地性** | podAffinity（与数据库同 AZ） |
| **成本优化** | nodeAffinity（Spot 节点）+ topologySpreadConstraints |

### 12.7 调度约束横向对比表

| 维度       | nodeSelector | Node Affinity         | Pod Affinity    | Taints/Tolerations  | topologySpread   |
| -------- | ------------ | --------------------- | --------------- | ------------------- | ---------------- |
| **作用对象** | 节点           | 节点                    | Pod             | 节点                  | 拓扑域              |
| **约束方向** | Pod → Node   | Pod → Node            | Pod → Pod       | Node → Pod          | Pod 分布           |
| **硬约束**  | ✅ 仅硬约束       | ✅ required            | ✅ required      | ✅ NoSchedule        | ✅ DoNotSchedule  |
| **软约束**  | 🔴 不支持       | ✅ preferred           | ✅ preferred     | ⚠️ PreferNoSchedule | ✅ ScheduleAnyway |
| **操作符**  | = 精确匹配       | In/NotIn/Exists/Gt/Lt | In/NotIn/Exists | Equal/Exists        | -                |
| **权重支持** | 🔴           | ✅ 1-100               | ✅ 1-100         | 🔴                  | 🔴               |
| **拓扑感知** | 🔴           | 🔴                    | ✅ topologyKey   | 🔴                  | ✅ topologyKey    |
| **典型场景** | 简单节点选择       | 复杂节点选择                | 共置/反亲和          | 专用节点/驱逐             | 跨 AZ 分布          |
| **复杂度**  | 低            | 中                     | 高               | 中                   | 中                |
| **性能影响** | 低            | 中                     | 高（O(n²)）        | 低                   | 中                |

### 12.8 调度约束问题定位表

| 问题现象 | 可能原因 | 排查命令 | 解决方案 |
|----------|----------|----------|----------|
| **Pod 一直 Pending** | 没有满足约束的节点 | `kubectl describe pod <pod>` 查看 Events | 检查节点标签、放宽约束条件 |
| **Pod 分布不均** | topologySpread maxSkew 设置过大 | `kubectl get pod -o wide` 查看分布 | 减小 maxSkew 值 |
| **Pod 无法调度到特定节点** | 节点有 Taint 但 Pod 无 Toleration | `kubectl describe node <node>` 查看 Taints | 添加对应的 Toleration |
| **Pod Affinity 不生效** | labelSelector 不匹配 | `kubectl get pod --show-labels` | 检查标签选择器 |
| **调度延迟高** | Pod Affinity 计算复杂度高 | 监控调度器延迟指标 | 减少 Affinity 规则、使用 topologySpread |
| **节点资源不均** | 缺少 topologySpread 约束 | `kubectl top nodes` | 添加 topologySpreadConstraints |
| **Spot 节点 Pod 被驱逐** | 缺少 PDB 保护 | `kubectl get pdb` | 配置 PodDisruptionBudget |
| **跨 AZ 延迟高** | Pod 与依赖服务不在同 AZ | `kubectl get pod -o wide` | 使用 Pod Affinity 共置 |

### 12.9 调度约束决策流程图

```
┌─────────────────────────────────────────────────────────────────────┐
│                    调度约束选型决策树                                  │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  需要什么类型的约束？                                                  │
│       │                                                             │
│       ├─── 节点选择 ─────────────────────────────────────────────┐   │
│       │                                                         │   │
│       │    是否需要复杂条件？                                      │   │
│       │         │                                               │   │
│       │         ├─── 否 → nodeSelector（简单键值匹配）             │   │
│       │         │                                               │   │
│       │         └─── 是 → Node Affinity                         │   │
│       │                   • required: 硬约束                     │   │
│       │                   • preferred: 软约束 + 权重              │   │
│       │                                                         │   │
│       ├─── Pod 关系 ─────────────────────────────────────────────┐   │
│       │                                                          │  │
│       │    Pod 之间需要什么关系？                                   │  │
│       │         │                                                │  │
│       │         ├─── 共置（同节点/同 AZ）→ Pod Affinity             │  │
│       │         │                                                │  │
│       │         └─── 分散（不同节点/不同 AZ）→ Pod Anti-Affinity    │  │
│       │                                                          │  │
│       ├─── 节点专用 ─────────────────────────────────────────────┐   │
│       │                                                         │   │
│       │    节点是否需要专用？                                      │   │
│       │         │                                               │   │
│       │         ├─── 是 → Taints + Tolerations                  │   │
│       │         │         • NoSchedule: 禁止调度                 │   │
│       │         │         • NoExecute: 驱逐现有 Pod              │   │
│       │         │                                               │   │
│       │         └─── 否 → 使用 Node Affinity                     │   │
│       │                                                         │   │
│       └─── 均匀分布 ─────────────────────────────────────────────┐   │
│                                                                     │
│            需要跨拓扑域均匀分布？                                       │
│                 │                                                   │
│                 └─── 是 → topologySpreadConstraints                 │
│                           • 跨 AZ: topology.kubernetes.io/zone      │
│                           • 跨节点: kubernetes.io/hostname           │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### 12.10 调度约束参数调优表

| 参数 | 默认值 | 调优建议 | 性能影响 | 说明 |
|------|--------|----------|----------|------|
| **topologySpread.maxSkew** | 1 | 1-3 | 调度灵活性 | 值越大分布越不均匀 |
| **affinity.weight** | - | 1-100 | 调度优先级 | 多个 preferred 规则时的权重 |
| **podAntiAffinity.topologyKey** | - | hostname/zone | 隔离粒度 | hostname 更严格但资源利用率低 |
| **toleration.tolerationSeconds** | - | 300-600 | 驱逐延迟 | NoExecute 时 Pod 存活时间 |
| **scheduler.percentageOfNodesToScore** | 50% | 10-50% | 调度延迟 | 大集群可降低以提升性能 |

---

## 13. Kubernetes 弹性伸缩
### 13.1 弹性伸缩概述

Kubernetes 弹性伸缩分为两个层面：
- **Pod 级别伸缩**：HPA（水平）、VPA（垂直）、KEDA（事件驱动）
- **节点级别伸缩**：Cluster Autoscaler、Karpenter

### 13.2 Cluster Autoscaler 架构详解

```
┌─────────────────────────────────────────────────────────┐
│              Cluster Autoscaler 工作流程                 │
│                                                         │
│  1. 监听 Pending Pod                                    │
│     kube-scheduler 标记 Pod 为 Unschedulable            │
│                    ↓                                   │
│  2. 计算所需节点                                         │
│     CA 检查 ASG 配置，计算需要多少节点                     │
│                    ↓                                   │
│  3. 扩容请求                                            │
│     CA 调用 ASG API 增加 DesiredCapacity                │
│                    ↓                                   │
│  4. ASG 启动 EC2                                        │
│     等待 EC2 启动 + 加入集群（1-3 分钟）                   │
│                    ↓                                   │
│  5. Pod 调度                                            │
│     kube-scheduler 将 Pending Pod 调度到新节点            │
└─────────────────────────────────────────────────────────┘
```

**CA 缩容逻辑：**
- 节点利用率 < 50%（可配置）持续 10 分钟
- 节点上所有 Pod 可以迁移到其他节点
- 无 PDB 阻止驱逐


### 13.3 Karpenter 架构详解

```
┌─────────────────────────────────────────────────────────┐
│                 Karpenter 工作流程                       │
│                                                         │
│  1. 监听 Pending Pod                                    │
│     Karpenter 直接 Watch API Server                     │
│                    ↓                                    │
│  2. 计算最优实例                                         │
│     根据 Pod 需求选择最合适的实例类型                       │
│     （考虑 CPU/内存/GPU/价格/可用性）                      │
│                    ↓                                   │
│  3. 直接调用 EC2 Fleet API                              │
│     跳过 ASG，直接创建 EC2 实例                           │
│     （支持 Spot + On-Demand 混合）                       │
│                    ↓                                   │
│  4. 快速启动（< 1 分钟）                                  │
│     EC2 启动 + 加入集群                                   │
│                    ↓                                    │
│  5. Pod 调度                                             │
│     kube-scheduler 将 Pending Pod 调度到新节点            │
└─────────────────────────────────────────────────────────┘
```

### 13.4 Cluster Autoscaler vs Karpenter 核心对比

| 维度 | Cluster Autoscaler | Karpenter |
|------|-------------------|-----------|
| **扩容速度** | 1-3 分钟 | < 1 分钟 |
| **节点选择** | 预定义 ASG 实例类型 | 动态选择最优实例 |
| **Spot 支持** | 需要单独 ASG | 原生支持混合 |
| **成本优化** | 有限 | bin-packing + Consolidation |
| **缩容智能** | 基于利用率阈值 | Drift + Consolidation |
| **配置复杂度** | 需要管理多个 ASG | NodePool 声明式配置 |


**Karpenter NodePool 配置示例：**
```yaml
apiVersion: karpenter.sh/v1beta1
kind: NodePool
metadata:
  name: default
spec:
  template:
    spec:
      requirements:
      - key: kubernetes.io/arch
        operator: In
        values: ["amd64", "arm64"]
      - key: karpenter.sh/capacity-type
        operator: In
        values: ["spot", "on-demand"]
      - key: topology.kubernetes.io/zone
        operator: In
        values: ["us-east-1a", "us-east-1b", "us-east-1c"]
      nodeClassRef:
        name: default
  limits:
    cpu: 1000
    memory: 1000Gi
  disruption:
    consolidationPolicy: WhenUnderutilized
    consolidateAfter: 30s
```

**选型建议：**
| 场景 | 推荐 | 原因 |
|------|------|------|
| 新集群 | Karpenter | 更快、更智能、成本更优 |
| 已有 ASG 管理 | CA | 迁移成本低 |
| 需要 Spot 混合 | Karpenter | 原生支持 |
| 简单场景 | CA | 配置简单 |
| 成本敏感 | Karpenter | bin-packing + Consolidation |

### 13.5 HPA（Horizontal Pod Autoscaler）详解

**HPA 工作原理：**
```
┌─────────────────────────────────────────────────────────┐
│                    HPA 工作流程                          │
│                                                         │
│  1. Metrics Server 采集 Pod 指标                         │
│     CPU/内存使用率、自定义指标                              │
│                    ↓                                    │
│  2. HPA Controller 计算期望副本数                         │
│     期望副本 = ceil(当前副本 × 当前指标 / 目标指标)          │
│                    ↓                                    │
│  3. 更新 Deployment replicas                            │
│     受 minReplicas 和 maxReplicas 限制                   │
│                    ↓                                    │
│  4. Deployment Controller 创建/删除 Pod                  │
└─────────────────────────────────────────────────────────┘
```

**HPA 配置示例：**
```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: web-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: web
  minReplicas: 2
  maxReplicas: 20
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70  # CPU 使用率 70% 触发扩容
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80  # 内存使用率 80% 触发扩容
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300  # 缩容稳定窗口
      policies:
      - type: Percent
        value: 10
        periodSeconds: 60  # 每分钟最多缩容 10%
    scaleUp:
      stabilizationWindowSeconds: 0
      policies:
      - type: Percent
        value: 100
        periodSeconds: 15  # 每 15 秒最多扩容 100%
```

### 13.6 VPA（Vertical Pod Autoscaler）详解

**VPA 工作模式：**

| 模式 | 说明 | 适用场景 |
|------|------|----------|
| **Off** | 只推荐，不自动调整 | 观察阶段 |
| **Initial** | 只在 Pod 创建时设置 | 避免重启 |
| **Auto** | 自动调整（会重启 Pod） | 完全自动化 |

**VPA 配置示例：**
```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: web-vpa
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: web
  updatePolicy:
    updateMode: Auto  # 自动调整
  resourcePolicy:
    containerPolicies:
    - containerName: web
      minAllowed:
        cpu: 100m
        memory: 128Mi
      maxAllowed:
        cpu: 4
        memory: 8Gi
      controlledResources: ["cpu", "memory"]
```

### 13.7 KEDA（Kubernetes Event-driven Autoscaling）

**KEDA 支持的触发器：**

| 触发器 | 说明 | 适用场景 |
|--------|------|----------|
| **AWS SQS** | SQS 队列深度 | 消息处理 |
| **Kafka** | Kafka 消费者 Lag | 流处理 |
| **Prometheus** | 自定义指标 | 通用场景 |
| **Cron** | 定时扩缩容 | 预测性扩容 |
| **HTTP** | HTTP 请求数 | Web 应用 |

**KEDA 配置示例（SQS 触发）：**
```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: sqs-scaler
spec:
  scaleTargetRef:
    name: worker
  minReplicaCount: 0   # 可以缩容到 0
  maxReplicaCount: 100
  triggers:
  - type: aws-sqs-queue
    metadata:
      queueURL: https://sqs.us-east-1.amazonaws.com/123456789/my-queue
      queueLength: "5"  # 每 5 条消息一个 Pod
      awsRegion: us-east-1
    authenticationRef:
      name: keda-aws-credentials
```

### 13.8 弹性伸缩方案对比

| 维度         | HPA        | VPA    | KEDA   | CA/Karpenter |
| ---------- | ---------- | ------ | ------ | ------------ |
| **伸缩维度**   | Pod 数量     | Pod 资源 | Pod 数量 | 节点数量         |
| **触发条件**   | CPU/内存/自定义 | 资源使用   | 事件驱动   | Pending Pod  |
| **缩容到 0**  | 🔴 不支持     | N/A    | ✅ 支持   | ✅ 支持         |
| **重启 Pod** | 🔴 不需要     | ✅ 需要   | 🔴 不需要 | N/A          |
| **适用场景**   | 通用         | 资源优化   | 事件驱动   | 节点扩缩         |

### 13.9 弹性伸缩问题定位表

| 问题 | 症状 | 原因 | 解决方案 |
|------|------|------|----------|
| **HPA 不扩容** | 副本数不变 | Metrics Server 未部署 | 部署 Metrics Server |
| **HPA 扩容慢** | 扩容延迟 | stabilizationWindow 过长 | 调整 behavior 配置 |
| **VPA 不生效** | 资源未调整 | updateMode 为 Off | 设置为 Auto 或 Initial |
| **KEDA 不触发** | 副本为 0 | 触发器配置错误 | 检查触发器认证和配置 |
| **CA 扩容慢** | 节点启动慢 | ASG 启动时间 | 使用 Karpenter |
| **节点不缩容** | 节点空闲 | PDB 阻止驱逐 | 检查 PDB 配置 |


---

## 14. Kubernetes 升级策略
### 14.1 部署策略概述

| 策略 | 说明 | 停机时间 | 回滚速度 | 资源开销 |
|------|------|---------|---------|---------|
| **滚动更新** | 逐步替换旧 Pod | 无 | 中等 | 低 |
| **蓝绿部署** | 两套环境切换 | 无 | 快 | 高（2x） |
| **灰度发布** | 按比例分流 | 无 | 快 | 中等 |
| **金丝雀发布** | 小流量验证 | 无 | 快 | 低 |
| **重建部署** | 先删后建 | 有 | 慢 | 低 |

### 14.2 滚动更新（Rolling Update）

**Deployment 滚动更新配置：**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  replicas: 10
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 25%        # 最多超出期望副本数的比例
      maxUnavailable: 25%  # 最多不可用副本数的比例
  template:
    spec:
      containers:
      - name: app
        image: myapp:v2
        readinessProbe:    # 就绪探针（必须配置）
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 5
```

**滚动更新流程：**
```
┌─────────────────────────────────────────────────────────┐
│              滚动更新流程（10 副本）                      │
│                                                          │
│  初始状态: v1 v1 v1 v1 v1 v1 v1 v1 v1 v1                │
│                    ↓                                    │
│  Step 1:    v1 v1 v1 v1 v1 v1 v1 v1 v2 v2 (maxSurge)   │
│                    ↓                                    │
│  Step 2:    v1 v1 v1 v1 v1 v1 v2 v2 v2 v2              │
│                    ↓                                    │
│  Step 3:    v1 v1 v1 v1 v2 v2 v2 v2 v2 v2              │
│                    ↓                                    │
│  Step 4:    v1 v1 v2 v2 v2 v2 v2 v2 v2 v2              │
│                    ↓                                    │
│  完成:      v2 v2 v2 v2 v2 v2 v2 v2 v2 v2              │
└─────────────────────────────────────────────────────────┘
```

**关键参数说明：**
| 参数 | 说明 | 推荐值 |
|------|------|--------|
| **maxSurge** | 滚动更新时最多创建的额外 Pod 数 | 25% |
| **maxUnavailable** | 滚动更新时最多不可用的 Pod 数 | 25% |
| **minReadySeconds** | Pod 就绪后等待时间 | 10-30s |
| **progressDeadlineSeconds** | 更新超时时间 | 600s |

### 14.3 蓝绿部署（Blue-Green Deployment）

```
┌─────────────────────────────────────────────────────────┐
│                   蓝绿部署架构                            │
│                                                         │
│                    ┌──────────┐                         │
│                    │  Service │                         │
│                    │ selector │                         │
│                    └────┬─────┘                         │
│                         │                               │
│           ┌─────────────┴─────────────┐                 │
│           │                           │                 │
│           ▼                           ▼                 │
│    ┌─────────────┐            ┌─────────────┐           │
│    │  Blue (v1)  │            │ Green (v2)  │           │
│    │  当前生产    │            │  新版本      │           │
│    │  version=v1 │            │  version=v2 │           │ 
│    └─────────────┘            └─────────────┘           │
│                                                         │
│  切换方式：修改 Service selector                          │
│  selector: version=v1  →  selector: version=v2          │
└─────────────────────────────────────────────────────────┘
```

**蓝绿部署配置示例：**
```yaml
# Blue Deployment (当前版本)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-blue
spec:
  replicas: 10
  selector:
    matchLabels:
      app: myapp
      version: v1
  template:
    metadata:
      labels:
        app: myapp
        version: v1
    spec:
      containers:
      - name: app
        image: myapp:v1
---
# Green Deployment (新版本)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-green
spec:
  replicas: 10
  selector:
    matchLabels:
      app: myapp
      version: v2
  template:
    metadata:
      labels:
        app: myapp
        version: v2
    spec:
      containers:
      - name: app
        image: myapp:v2
---
# Service (切换流量)
apiVersion: v1
kind: Service
metadata:
  name: myapp
spec:
  selector:
    app: myapp
    version: v1  # 切换到 v2 即完成蓝绿部署
  ports:
  - port: 80
    targetPort: 8080
```

### 14.4 灰度发布（Canary Release）

**使用 Ingress 实现灰度发布：**
```yaml
# 主 Ingress (90% 流量)
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myapp-main
spec:
  rules:
  - host: myapp.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: myapp-stable
            port:
              number: 80
---
# Canary Ingress (10% 流量)
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myapp-canary
  annotations:
    nginx.ingress.kubernetes.io/canary: "true"
    nginx.ingress.kubernetes.io/canary-weight: "10"
spec:
  rules:
  - host: myapp.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: myapp-canary
            port:
              number: 80
```

### 14.5 Argo Rollouts 渐进式交付

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: myapp
spec:
  replicas: 10
  strategy:
    canary:
      steps:
      - setWeight: 5       # 5% 流量
      - pause: {duration: 2m}
      - setWeight: 20      # 20% 流量
      - pause: {duration: 5m}
      - setWeight: 50      # 50% 流量
      - pause: {duration: 10m}
      - setWeight: 100     # 100% 流量
      analysis:
        templates:
        - templateName: success-rate
        startingStep: 1    # 从第一步开始分析
      trafficRouting:
        nginx:
          stableIngress: myapp-stable
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
      - name: app
        image: myapp:v2
---
# 分析模板
apiVersion: argoproj.io/v1alpha1
kind: AnalysisTemplate
metadata:
  name: success-rate
spec:
  metrics:
  - name: success-rate
    interval: 1m
    successCondition: result[0] >= 0.95
    provider:
      prometheus:
        address: http://prometheus:9090
        query: |
          sum(rate(http_requests_total{app="myapp",status=~"2.."}[5m])) /
          sum(rate(http_requests_total{app="myapp"}[5m]))
```

### 14.6 集群升级流程

**EKS 集群升级步骤：**
```
┌─────────────────────────────────────────────────────────┐
│                 EKS 集群升级流程                          │
│                                                         │
│  1. 升级前准备                                           │
│     • 检查版本兼容性（kubectl、插件、应用）                  │
│     • 备份 etcd 数据                                     │
│     • 检查 PDB（Pod Disruption Budget）                  │
│                    ↓                                    │
│  2. 升级控制平面                                          │
│     • AWS 自动升级 EKS 控制平面                            │
│     • 通常 10-15 分钟                                    │
│                    ↓                                    │
│  3. 升级数据平面（节点组）                                 │
│     • 方式 1：滚动更新节点组                               │
│     • 方式 2：蓝绿节点组切换                               │
│                    ↓                                    │
│  4. 升级插件                                             │
│     • CoreDNS、kube-proxy、VPC CNI                      │
│     • AWS Load Balancer Controller                      │
│                    ↓                                    │
│  5. 验证                                                │
│     • 检查节点状态                                        │
│     • 检查 Pod 状态                                      │
│     • 运行 E2E 测试                                      │
└─────────────────────────────────────────────────────────┘
```

**升级命令示例：**
```bash
# 1. 检查当前版本
kubectl version --short
aws eks describe-cluster --name my-cluster --query cluster.version

# 2. 升级控制平面
aws eks update-cluster-version --name my-cluster --kubernetes-version 1.29

# 3. 等待升级完成
aws eks wait cluster-active --name my-cluster

# 4. 升级节点组
aws eks update-nodegroup-version \
  --cluster-name my-cluster \
  --nodegroup-name my-nodegroup \
  --kubernetes-version 1.29

# 5. 升级插件
aws eks update-addon --cluster-name my-cluster --addon-name coredns
aws eks update-addon --cluster-name my-cluster --addon-name kube-proxy
aws eks update-addon --cluster-name my-cluster --addon-name vpc-cni
```

### 14.7 升级策略对比

| 维度 | 滚动更新 | 蓝绿部署 | 灰度发布 |
|------|---------|---------|---------|
| **复杂度** | 低 | 中 | 高 |
| **资源开销** | 低 | 高（2x） | 中 |
| **回滚速度** | 慢 | 快（秒级） | 快 |
| **风险控制** | 中 | 高 | 最高 |
| **适用场景** | 常规更新 | 重大版本 | 高风险变更 |
| **流量控制** | 无 | 全量切换 | 精细控制 |

### 14.8 升级策略横向对比表（扩展）

| 维度               | 滚动更新   | 蓝绿部署    | 灰度发布        | Argo Rollouts        | Flagger              |
| ---------------- | ------ | ------- | ----------- | -------------------- | -------------------- |
| **实现方式**         | K8S 原生 | 手动/脚本   | Ingress 注解  | CRD + Controller     | CRD + Controller     |
| **流量分割**         | 🔴 不支持 | 🔴 全量切换 | ✅ 权重/Header | ✅ 权重/Header/镜像       | ✅ 权重/Header          |
| **自动回滚**         | ⚠️ 手动  | ⚠️ 手动   | ⚠️ 手动       | ✅ 基于指标               | ✅ 基于指标               |
| **指标分析**         | 🔴     | 🔴      | 🔴          | ✅ Prometheus/Datadog | ✅ Prometheus/Datadog |
| **A/B 测试**       | 🔴     | 🔴      | ⚠️ 有限       | ✅ 完整支持               | ✅ 完整支持               |
| **流量镜像**         | 🔴     | 🔴      | 🔴          | ✅ 支持                 | ✅ 支持                 |
| **Service Mesh** | 不需要    | 不需要     | 不需要         | 可选（增强）               | 需要 Istio/Linkerd     |
| **学习曲线**         | 低      | 低       | 中           | 中                    | 高                    |
| **生产推荐**         | 常规更新   | 数据库迁移   | 前端应用        | 微服务                  | Service Mesh 环境      |

### 14.9 升级参数调优表

| 参数 | 默认值 | 调优建议 | 性能影响 | 说明 |
|------|--------|----------|----------|------|
| **maxSurge** | 25% | 10-50% | 更新速度 | 值越大更新越快，资源消耗越多 |
| **maxUnavailable** | 25% | 0-25% | 可用性 | 0 表示始终保持全部副本可用 |
| **minReadySeconds** | 0 | 10-60s | 稳定性 | Pod 就绪后等待时间，防止快速失败 |
| **progressDeadlineSeconds** | 600s | 300-1200s | 超时控制 | 更新超时时间 |
| **revisionHistoryLimit** | 10 | 3-10 | 存储空间 | 保留的 ReplicaSet 历史版本数 |
| **terminationGracePeriodSeconds** | 30s | 30-120s | 优雅关闭 | Pod 终止前的等待时间 |
| **PDB.minAvailable** | - | 50-80% | 可用性 | 升级期间最少可用 Pod 数 |
| **PDB.maxUnavailable** | - | 1-20% | 更新速度 | 升级期间最多不可用 Pod 数 |

### 14.10 升级问题定位表

| 问题现象 | 可能原因 | 排查命令 | 解决方案 |
|----------|----------|----------|----------|
| **更新卡住不动** | 新 Pod 无法就绪 | `kubectl rollout status deployment/<name>` | 检查 readinessProbe、镜像拉取 |
| **更新超时** | progressDeadlineSeconds 过短 | `kubectl describe deployment/<name>` | 增加超时时间或检查 Pod 问题 |
| **回滚失败** | 历史版本被清理 | `kubectl rollout history deployment/<name>` | 增加 revisionHistoryLimit |
| **流量丢失** | 缺少 readinessProbe | `kubectl get endpoints <service>` | 配置就绪探针 |
| **连接中断** | terminationGracePeriod 过短 | 检查应用日志 | 增加优雅关闭时间 |
| **灰度流量不准** | Ingress 配置错误 | `kubectl describe ingress` | 检查 canary 注解 |
| **Argo Rollouts 分析失败** | Prometheus 查询错误 | `kubectl describe analysisrun` | 检查 PromQL 语法 |
| **节点升级 Pod 被驱逐** | 缺少 PDB | `kubectl get pdb` | 配置 PodDisruptionBudget |
| **升级后性能下降** | 资源配置不当 | `kubectl top pods` | 检查 requests/limits |
| **升级后功能异常** | 配置不兼容 | 检查 ConfigMap/Secret | 验证配置版本兼容性 |

### 14.11 升级策略决策流程图

```
┌─────────────────────────────────────────────────────────────────────┐
│                    升级策略选型决策树                                  │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  变更风险级别？                                                       │
│       │                                                             │
│       ├─── 低风险（配置变更、小版本更新）                                │
│       │         │                                                   │
│       │         └─── 滚动更新（Rolling Update）                       │
│       │               • 简单、资源开销低                              │
│       │               • 配置 readinessProbe + PDB                   │
│       │                                                            │
│       ├─── 中风险（功能变更、API 变更）                                │
│       │         │                                                  │
│       │         └─── 是否需要快速回滚？                               │
│       │                   │                                        │
│       │                   ├─── 是 → 蓝绿部署                         │
│       │                   │         • 秒级回滚                       │
│       │                   │         • 资源开销 2x                    │
│       │                   │                                        │
│       │                   └─── 否 → 灰度发布                         │
│       │                             • 按比例分流验证                  │
│       │                             • 逐步扩大流量                   │
│       │                                                            │
│       └─── 高风险（重大版本、架构变更）                                 │
│                 │                                                  │
│                 └─── 是否有 Service Mesh？                           │
│                           │                                         │
│                           ├─── 是 → Flagger + Istio                 │
│                           │         • 自动金丝雀分析                  │
│                           │         • 基于指标自动回滚                 │
│                           │                                         │
│                           └─── 否 → Argo Rollouts                   │
│                                     • 渐进式交付                      │
│                                     • 支持多种流量管理                 │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### 14.12 升级最佳实践清单

| 阶段 | 检查项 | 说明 |
|------|--------|------|
| **升级前** | ☐ 版本兼容性检查 | kubectl、插件、应用 API 兼容性 |
| | ☐ 备份 etcd | 防止升级失败数据丢失 |
| | ☐ 配置 PDB | 保证升级期间服务可用性 |
| | ☐ 检查资源余量 | 确保有足够资源创建新 Pod |
| | ☐ 通知相关团队 | 升级窗口协调 |
| **升级中** | ☐ 监控关键指标 | 错误率、延迟、资源使用 |
| | ☐ 观察 Pod 状态 | 确保新 Pod 正常启动 |
| | ☐ 检查日志 | 发现潜在问题 |
| **升级后** | ☐ 功能验证 | 核心功能测试 |
| | ☐ 性能验证 | 对比升级前后性能 |
| | ☐ 清理旧资源 | 删除旧版本 Deployment/ReplicaSet |
| | ☐ 更新文档 | 记录升级过程和问题 |

---

## 15. Kubernetes 生产实践
### 15.1 集群规模规划
#### 15.1.1 控制平面规模规划

| 集群规模 | 节点数 | API Server | etcd | Controller Manager | 说明 |
|----------|--------|------------|------|---------------------|------|
| **小型** | < 100 | 2 副本 | 3 节点 | 2 副本 | 开发/测试 |
| **中型** | 100-500 | 3 副本 | 3 节点 | 2 副本 | 生产环境 |
| **大型** | 500-1000 | 3-5 副本 | 5 节点 | 2 副本 | 大规模生产 |
| **超大型** | > 1000 | 5+ 副本 | 5 节点 | 2 副本 | 需要分片 |

**EKS 托管集群规模限制：**

| 资源 | 默认限制 | 可调整 |
|------|----------|--------|
| 节点数 | 1000 | 可申请提升 |
| Pod 数/节点 | 110-250 | 取决于实例类型 |
| Service 数 | 10000 | 可申请提升 |
| ConfigMap 数 | 无限制 | - |
| Secret 数 | 无限制 | - |

### 15.2 故障排查手册
#### 15.2.1 Pod 故障排查决策树

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Pod 故障排查决策树                                 │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Pod 状态异常                                                        │
│       │                                                             │
│       ▼                                                             │
│  kubectl describe pod <pod-name>                                    │
│  kubectl get events --field-selector involvedObject.name=<pod>      │
│       │                                                             │
│       ├─── Pending ───────────────────────────────────────────┐     │
│       │                                                       │     │
│       │    ├─ "Insufficient cpu/memory"                       │     │
│       │    │   → 检查节点资源: kubectl top nodes                │     │
│       │    │   → 解决: 扩容节点或调整 requests                   │     │
│       │    │                                                  │     │
│       │    ├─ "no nodes available to schedule"                │     │
│       │    │   → 检查 Taints: kubectl describe node           │     │
│       │    │   → 解决: 添加 Tolerations 或移除 Taint            │     │
│       │    │                                                  │     │
│       │    ├─ "didn't match Pod's node affinity"              │     │
│       │    │   → 检查 nodeSelector/affinity                   │     │
│       │    │   → 解决: 调整调度约束或添加节点标签                  │     │
│       │    │                                                  │     │
│       │    └─ "persistentvolumeclaim not found"               │     │
│       │        → 检查 PVC: kubectl get pvc                    │     │
│       │        → 解决: 创建 PVC 或修复 StorageClass             │     │
│       │                                                       │     │
│       ├─── ContainerCreating ─────────────────────────────────┐     │
│       │                                                       │     │
│       │    ├─ "failed to pull image"                          │     │
│       │    │   → 检查镜像名称和 tag                             │     │
│       │    │   → 检查 imagePullSecrets                        │     │
│       │    │                                                  │     │
│       │    └─ "network plugin is not ready"                   │     │
│       │        → 检查 CNI: kubectl get pods -n kube-system     │     │
│       │        → 解决: 重启 CNI 插件                            │     │
│       │                                                       │     │
│       ├─── CrashLoopBackOff ──────────────────────────────────┐     │
│       │                                                       │     │
│       │    → kubectl logs <pod-name> --previous               │     │
│       │    │                                                  │     │
│       │    ├─ 应用错误 → 检查应用日志，修复代码                    │     │
│       │    ├─ 配置错误 → 检查 ConfigMap/Secret                  │     │
│       │    └─ 资源不足 → 增加 memory limits                     │     │
│       │                                                       │     │
│       ├─── OOMKilled ─────────────────────────────────────────┐     │
│       │                                                       │     │
│       │    → kubectl describe pod | grep -A5 "Last State"     │     │
│       │    → 解决: 增加 memory limits 或优化应用内存使用          │     │
│       │                                                       │     │
│       └─── Evicted ───────────────────────────────────────────┐     │
│                                                               │     │
│            → kubectl describe pod | grep "eviction"           │     │
│            → 原因: 节点资源压力（磁盘/内存）                       │     │
│            → 解决: 清理节点或调整 eviction 阈值                   │     │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

#### 15.2.2 常用故障排查命令集

```bash
# ===== Pod 排查 =====
kubectl get pods -o wide                          # 查看 Pod 状态和节点
kubectl describe pod <pod-name>                   # 查看 Pod 详情和事件
kubectl logs <pod-name> -c <container>            # 查看容器日志
kubectl logs <pod-name> --previous                # 查看上次崩溃日志
kubectl exec -it <pod-name> -- /bin/sh            # 进入容器调试
kubectl get events --sort-by='.lastTimestamp'     # 按时间排序事件

# ===== 节点排查 =====
kubectl get nodes -o wide                         # 查看节点状态
kubectl describe node <node-name>                 # 查看节点详情
kubectl top nodes                                 # 查看节点资源使用
kubectl get pods -A -o wide --field-selector spec.nodeName=<node>  # 节点上的 Pod

# ===== 网络排查 =====
kubectl get svc,endpoints                         # 查看 Service 和 Endpoints
kubectl run debug --image=busybox -it --rm -- sh  # 启动调试 Pod
nslookup kubernetes.default                       # 测试 DNS
curl -v http://<service-name>:<port>              # 测试 Service 连通性

# ===== 存储排查 =====
kubectl get pv,pvc                                # 查看存储卷状态
kubectl describe pvc <pvc-name>                   # 查看 PVC 详情

# ===== 资源使用 =====
kubectl top pods --sort-by=memory                 # 按内存排序 Pod
kubectl top pods --sort-by=cpu                    # 按 CPU 排序 Pod
```

### 15.3 性能调优参数汇总

| 组件 | 参数 | 默认值 | 调优建议 | 影响 |
|------|------|--------|----------|------|
| **etcd** | --quota-backend-bytes | 2GB | 8GB | 存储容量 |
| **etcd** | --snapshot-count | 100000 | 50000 | 恢复速度 |
| **etcd** | --auto-compaction-retention | 0 | 1h | 存储空间 |
| **API Server** | --max-requests-inflight | 400 | 800 | 并发读请求 |
| **API Server** | --max-mutating-requests-inflight | 200 | 400 | 并发写请求 |
| **API Server** | --watch-cache-sizes | 默认 | 增大热点资源 | Watch 性能 |
| **Controller** | --kube-api-qps | 20 | 100 | API 请求速率 |
| **Controller** | --kube-api-burst | 30 | 200 | API 突发请求 |
| **Controller** | --concurrent-deployment-syncs | 5 | 10 | Deployment 并发 |
| **Scheduler** | --kube-api-qps | 50 | 100 | API 请求速率 |
| **Scheduler** | --percentageOfNodesToScore | 50% | 30%（大集群） | 调度性能 |
| **kubelet** | --max-pods | 110 | 250 | Pod 密度 |
| **kubelet** | --kube-api-qps | 5 | 50 | API 请求速率 |
| **kubelet** | --serialize-image-pulls | true | false | 镜像拉取并发 |
| **kube-proxy** | --iptables-sync-period | 30s | 60s | iptables 同步 |
| **kube-proxy** | mode | iptables | ipvs | 大规模 Service |

### 15.4 高可用架构

```
┌─────────────────────────────────────────────────────────────────────┐
│                    EKS 高可用架构                                     │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │                    AWS 托管控制平面                           │    │
│  │  ┌───────────┐  ┌───────────┐  ┌───────────┐                │    │
│  │  │ API Server│  │ API Server│  │ API Server│                │    │
│  │  │  (AZ-a)   │  │  (AZ-b)   │  │  (AZ-c)   │                │    │
│  │  └───────────┘  └───────────┘  └───────────┘                │    │
│  │       │              │              │                       │    │
│  │       └──────────────┼──────────────┘                       │    │
│  │                      ▼                                      │    │
│  │              ┌──────────────┐                               │    │
│  │              │   etcd 集群   │                               │    │
│  │              │  (3 节点)     │                               │    │
│  │              └──────────────┘                               │    │
│  └─────────────────────────────────────────────────────────────┘    │
│                              │                                      │
│                              ▼                                      │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │                    数据平面（Worker Nodes）                   │    │
│  │                                                             │    │
│  │  AZ-a                AZ-b                AZ-c               │    │
│  │  ┌──────────┐       ┌──────────┐       ┌──────────┐         │    │
│  │  │ Node 1   │       │ Node 3   │       │ Node 5   │         │    │
│  │  │ Node 2   │       │ Node 4   │       │ Node 6   │         │    │
│  │  └──────────┘       └──────────┘       └──────────┘         │    │
│  │                                                             │    │
│  │  topologySpreadConstraints 确保 Pod 跨 AZ 分布                │    │
│  │  PodDisruptionBudget 确保升级时最小可用副本                     │    │
│  └─────────────────────────────────────────────────────────────┘    │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

**高可用配置示例：**
```yaml
# PodDisruptionBudget（确保最小可用副本）
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: myapp-pdb
spec:
  minAvailable: 2  # 或 maxUnavailable: 1
  selector:
    matchLabels:
      app: myapp
---
# Deployment（跨 AZ 分布）
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  replicas: 6
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      topologySpreadConstraints:
      - maxSkew: 1
        topologyKey: topology.kubernetes.io/zone
        whenUnsatisfiable: DoNotSchedule
        labelSelector:
          matchLabels:
            app: myapp
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchLabels:
                  app: myapp
              topologyKey: kubernetes.io/hostname
      containers:
      - name: app
        image: myapp:v1
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            cpu: 500m
            memory: 512Mi
```

### 15.5 成本优化最佳实践
#### 15.5.1 成本优化策略对比表

| 策略 | 节省比例 | 实施复杂度 | 风险 | 适用场景 |
|------|----------|------------|------|----------|
| **Spot 实例** | 60-90% | 中 | 中断风险 | 无状态、批处理 |
| **Reserved 实例** | 30-60% | 低 | 锁定期 | 稳定负载 |
| **Savings Plans** | 30-60% | 低 | 锁定期 | 混合负载 |
| **右调资源** | 20-40% | 中 | 性能影响 | 所有场景 |
| **Karpenter** | 20-30% | 中 | 学习曲线 | 动态负载 |
| **VPA** | 10-30% | 中 | 重启 Pod | 资源波动大 |
| **集群整合** | 30-50% | 高 | 隔离性 | 多环境 |
| **节点自动缩容** | 20-40% | 低 | 扩容延迟 | 波动负载 |

#### 15.5.2 Spot 实例最佳实践

```yaml
# Karpenter NodePool 配置 Spot 实例
apiVersion: karpenter.sh/v1beta1
kind: NodePool
metadata:
  name: spot-pool
spec:
  template:
    spec:
      requirements:
      - key: karpenter.sh/capacity-type
        operator: In
        values: ["spot"]
      - key: kubernetes.io/arch
        operator: In
        values: ["amd64"]
      - key: node.kubernetes.io/instance-type
        operator: In
        values: ["m5.large", "m5.xlarge", "m5a.large", "m5a.xlarge"]  # 多实例类型
  limits:
    cpu: 1000
    memory: 1000Gi
  disruption:
    consolidationPolicy: WhenUnderutilized
    consolidateAfter: 30s
---
# Deployment 配置 Spot 容忍
apiVersion: apps/v1
kind: Deployment
metadata:
  name: spot-workload
spec:
  replicas: 10
  template:
    spec:
      tolerations:
      - key: "karpenter.sh/capacity-type"
        operator: "Equal"
        value: "spot"
        effect: "NoSchedule"
      topologySpreadConstraints:
      - maxSkew: 1
        topologyKey: topology.kubernetes.io/zone
        whenUnsatisfiable: ScheduleAnyway
        labelSelector:
          matchLabels:
            app: spot-workload
```

#### 15.5.3 资源右调（Right-sizing）

```bash
# 使用 VPA 推荐器获取资源建议
kubectl get vpa -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.recommendation.containerRecommendations[*].target}{"\n"}{end}'

# 使用 kubectl top 分析实际使用
kubectl top pods --sort-by=cpu
kubectl top pods --sort-by=memory

# 使用 Prometheus 查询历史使用率
# CPU 使用率
sum(rate(container_cpu_usage_seconds_total{namespace="default"}[5m])) by (pod) / 
sum(kube_pod_container_resource_requests{resource="cpu",namespace="default"}) by (pod)

# 内存使用率
sum(container_memory_working_set_bytes{namespace="default"}) by (pod) / 
sum(kube_pod_container_resource_requests{resource="memory",namespace="default"}) by (pod)
```

#### 15.5.4 成本优化检查清单

| 检查项 | 说明 | 工具 |
|--------|------|------|
| ☐ 资源 requests 是否合理 | 避免过度申请 | VPA、kubectl top |
| ☐ 是否使用 Spot 实例 | 无状态负载优先 Spot | Karpenter |
| ☐ 是否配置 HPA | 根据负载自动扩缩 | HPA |
| ☐ 是否配置节点自动缩容 | 空闲节点自动回收 | CA/Karpenter |
| ☐ 是否有闲置资源 | 定期清理未使用资源 | kubectl get all |
| ☐ 存储是否优化 | 使用合适的存储类型 | gp3 vs gp2 |
| ☐ 网络是否优化 | 减少跨 AZ 流量 | Pod Affinity |
| ☐ 是否使用 Reserved/Savings Plans | 稳定负载预留 | AWS Cost Explorer |

### 15.6 安全加固最佳实践
#### 15.6.1 安全加固检查清单

| 层级 | 检查项 | 说明 | 优先级 |
|------|--------|------|--------|
| **集群** | ☐ API Server 访问控制 | 限制 API Server 网络访问 | 高 |
| | ☐ etcd 加密 | 启用 etcd 数据加密 | 高 |
| | ☐ 审计日志 | 启用 API 审计日志 | 高 |
| | ☐ 版本更新 | 及时更新 K8S 版本 | 中 |
| **节点** | ☐ 节点加固 | CIS Benchmark 合规 | 高 |
| | ☐ 节点访问控制 | 禁用 SSH、使用 SSM | 高 |
| | ☐ 节点镜像更新 | 定期更新 AMI | 中 |
| **网络** | ☐ NetworkPolicy | 默认拒绝 + 白名单 | 高 |
| | ☐ Pod 安全组 | 使用 Security Group for Pods | 中 |
| | ☐ 加密传输 | 启用 mTLS | 中 |
| **Pod** | ☐ PSA 策略 | 启用 Restricted 策略 | 高 |
| | ☐ 非 root 运行 | runAsNonRoot: true | 高 |
| | ☐ 只读文件系统 | readOnlyRootFilesystem: true | 中 |
| | ☐ 禁用特权 | privileged: false | 高 |
| **镜像** | ☐ 镜像扫描 | 使用 Trivy/ECR 扫描 | 高 |
| | ☐ 镜像签名 | 使用 Cosign 签名验证 | 中 |
| | ☐ 基础镜像 | 使用最小化基础镜像 | 中 |
| **Secret** | ☐ 外部 Secret 管理 | 使用 Secrets Manager | 高 |
| | ☐ Secret 加密 | 启用 KMS 加密 | 高 |
| | ☐ Secret 轮换 | 定期轮换 Secret | 中 |

#### 15.6.2 Pod 安全配置示例

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secure-pod
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    runAsGroup: 1000
    fsGroup: 1000
    seccompProfile:
      type: RuntimeDefault
  containers:
  - name: app
    image: myapp:v1
    securityContext:
      allowPrivilegeEscalation: false
      readOnlyRootFilesystem: true
      capabilities:
        drop:
        - ALL
    resources:
      limits:
        cpu: 500m
        memory: 512Mi
      requests:
        cpu: 100m
        memory: 128Mi
    volumeMounts:
    - name: tmp
      mountPath: /tmp
  volumes:
  - name: tmp
    emptyDir: {}
  serviceAccountName: minimal-sa
  automountServiceAccountToken: false
```

#### 15.6.3 NetworkPolicy 最佳实践

```yaml
# 默认拒绝所有入站流量
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-ingress
  namespace: production
spec:
  podSelector: {}
  policyTypes:
  - Ingress
---
# 默认拒绝所有出站流量
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-egress
  namespace: production
spec:
  podSelector: {}
  policyTypes:
  - Egress
---
# 允许特定流量
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-app-traffic
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: frontend
    ports:
    - protocol: TCP
      port: 8080
  egress:
  - to:
    - podSelector:
        matchLabels:
          app: database
    ports:
    - protocol: TCP
      port: 5432
  - to:  # 允许 DNS
    - namespaceSelector: {}
      podSelector:
        matchLabels:
          k8s-app: kube-dns
    ports:
    - protocol: UDP
      port: 53
```

### 15.7 生产实践问题定位表

| 问题类型 | 问题现象 | 可能原因 | 排查方法 | 解决方案 |
|----------|----------|----------|----------|----------|
| **性能** | API Server 响应慢 | 请求过多 | 监控 API Server 指标 | 增加副本、限流 |
| | etcd 延迟高 | 磁盘 I/O 慢 | etcdctl endpoint status | 使用 SSD、增加 IOPS |
| | 调度延迟高 | 节点过多 | 监控调度器指标 | 调整 percentageOfNodesToScore |
| | Pod 启动慢 | 镜像拉取慢 | 检查镜像大小 | 使用镜像缓存、减小镜像 |
| **稳定性** | 节点 NotReady | kubelet 问题 | kubectl describe node | 检查 kubelet 日志 |
| | Pod 频繁重启 | OOM/应用错误 | kubectl logs --previous | 增加内存/修复应用 |
| | 服务不可用 | Endpoint 为空 | kubectl get endpoints | 检查 Pod 就绪状态 |
| **成本** | 资源利用率低 | 过度申请 | kubectl top | 使用 VPA 调整 |
| | 节点空闲 | 缩容不及时 | 检查 CA 日志 | 调整缩容阈值 |
| **安全** | 未授权访问 | RBAC 配置错误 | kubectl auth can-i | 审查 RBAC 配置 |
| | Secret 泄露 | 日志/配置暴露 | 审计日志 | 使用外部 Secret 管理 |

---

## 16. 容器编排系统对比
### 16.1 编排系统概述

| 编排系统 | 定位 | 开发者 | 开源 | 适用场景 |
|----------|------|--------|------|----------|
| **Kubernetes** | 通用容器编排平台 | CNCF | ✅ | 复杂微服务、多云 |
| **Docker Swarm** | Docker 原生编排 | Docker | ✅ | 简单场景、Docker 生态 |
| **HashiCorp Nomad** | 多工作负载编排 | HashiCorp | ✅ | 混合工作负载 |
| **AWS ECS** | AWS 托管编排 | AWS | 🔴 | AWS 原生应用 |
| **Apache Mesos** | 数据中心操作系统 | Apache | ✅ | 大规模数据处理 |

### 16.2 多维度对比表

| 维度 | Kubernetes | Docker Swarm | Nomad | AWS ECS |
|------|------------|--------------|-------|---------|
| **架构复杂度** | 高 | 低 | 中 | 低（托管） |
| **学习曲线** | 陡峭 | 平缓 | 中等 | 平缓 |
| **扩展性** | 极强（CRD/Operator） | 弱 | 强（插件） | 中（AWS 集成） |
| **调度能力** | 强（多维度约束） | 中 | 强（bin-packing） | 中 |
| **服务发现** | 内置（CoreDNS） | 内置 | 需要 Consul | 内置（Cloud Map） |
| **负载均衡** | Service/Ingress | 内置 | 需要外部 | ALB/NLB 集成 |
| **存储支持** | CSI（丰富） | Volume Driver | CSI | EBS/EFS |
| **网络模型** | CNI（丰富） | Overlay | CNI | awsvpc |
| **多租户** | Namespace | 弱 | Namespace | 账户隔离 |
| **社区生态** | 最大 | 萎缩 | 增长中 | AWS 生态 |
| **运维成本** | 高（自建）/低（托管） | 低 | 中 | 低（托管） |
| **高可用** | 原生支持 | 原生支持 | 原生支持 | AWS 托管 |
| **滚动更新** | 原生支持 | 原生支持 | 原生支持 | 原生支持 |
| **自动扩缩容** | HPA/VPA/CA | 有限 | 需要外部 | Application Auto Scaling |

### 16.3 选型决策树

```
┌─────────────────────────────────────────────────────────────────────┐
│                    容器编排系统选型决策树                               │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  是否使用 AWS？                                                       │
│       │                                                             │
│       ├─── 是 ──────────────────────────────────────────────────┐   │
│       │                                                         │   │
│       │    是否需要复杂编排能力？                                   │   │
│       │         │                                               │   │
│       │         ├─── 是 → EKS（托管 K8S）                         │   │
│       │         │         • 完整 K8S 功能                        │   │
│       │         │         • 丰富生态系统                          │   │
│       │         │         • 多云可移植                            │   │
│       │         │                                               │   │
│       │         └─── 否 → ECS（简单、低成本）                      │   │
│       │                   • 简单易用                             │   │
│       │                   • 与 AWS 深度集成                      │   │
│       │                   • 无需管理控制平面                      │   │
│       │                                                         │   │
│       └─── 否 ──────────────────────────────────────────────────┐   │
│                                                                 │   │
│            是否需要多云/混合云？                                   │   │
│                 │                                               │   │
│                 ├─── 是 → Kubernetes                            │   │
│                 │         • 跨云可移植                           │   │
│                 │         • 标准化 API                           │   │
│                 │         • 丰富生态                             │   │
│                 │                                               │   │
│                 └─── 否 ─────────────────────────────────────   │   │
│                                                                 │   │
│                      是否需要混合工作负载（容器+VM+批处理）？         │   │
│                           │                                     │   │
│                           ├─── 是 → Nomad                       │   │
│                           │         • 支持多种工作负载            │   │
│                           │         • 简单架构                   │   │
│                           │         • HashiCorp 生态            │   │
│                           │                                     │   │
│                           └─── 否 → Kubernetes 或 Docker Swarm  │   │
│                                     • K8S: 复杂场景              │   │
│                                     • Swarm: 简单场景            │   │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### 16.4 选型建议总结

| 场景 | 推荐方案 | 理由 |
|------|----------|------|
| **AWS 原生应用** | ECS + Fargate | 简单、低成本、无需管理 |
| **复杂微服务** | EKS 或自建 K8S | 功能全面、生态丰富 |
| **多云部署** | Kubernetes | 跨云可移植、标准化 |
| **简单容器化** | Docker Swarm | 学习成本低、快速上手 |
| **混合工作负载** | Nomad | 支持容器+VM+批处理 |
| **大数据处理** | Kubernetes + Spark Operator | 统一平台、资源共享 |
| **边缘计算** | K3s 或 MicroK8s | 轻量级、资源占用低 |


---

## 附录：Kubernetes 常用命令速查表
### 集群管理命令

```bash
# ===== 集群信息 =====
kubectl cluster-info                              # 集群信息
kubectl get nodes -o wide                         # 节点列表
kubectl describe node <node-name>                 # 节点详情
kubectl top nodes                                 # 节点资源使用
kubectl get componentstatuses                     # 组件状态（已弃用）
kubectl get --raw='/healthz?verbose'              # API Server 健康检查

# ===== 命名空间 =====
kubectl get namespaces                            # 命名空间列表
kubectl create namespace <name>                   # 创建命名空间
kubectl delete namespace <name>                   # 删除命名空间
kubectl config set-context --current --namespace=<name>  # 切换默认命名空间

# ===== 上下文管理 =====
kubectl config get-contexts                       # 上下文列表
kubectl config current-context                    # 当前上下文
kubectl config use-context <context-name>         # 切换上下文
```

### Pod 管理命令

```bash
# ===== Pod 查看 =====
kubectl get pods                                  # Pod 列表
kubectl get pods -o wide                          # Pod 详细信息（含节点）
kubectl get pods -A                               # 所有命名空间 Pod
kubectl get pods --show-labels                    # 显示标签
kubectl get pods -l app=nginx                     # 按标签筛选
kubectl get pods --field-selector status.phase=Running  # 按状态筛选
kubectl describe pod <pod-name>                   # Pod 详情
kubectl get pod <pod-name> -o yaml                # Pod YAML

# ===== Pod 日志 =====
kubectl logs <pod-name>                           # 查看日志
kubectl logs <pod-name> -c <container>            # 指定容器日志
kubectl logs <pod-name> --previous                # 上次崩溃日志
kubectl logs <pod-name> -f                        # 实时日志
kubectl logs <pod-name> --tail=100                # 最后 100 行
kubectl logs -l app=nginx --all-containers        # 按标签查看所有容器日志

# ===== Pod 调试 =====
kubectl exec -it <pod-name> -- /bin/sh            # 进入容器
kubectl exec -it <pod-name> -c <container> -- /bin/sh  # 进入指定容器
kubectl port-forward <pod-name> 8080:80           # 端口转发
kubectl cp <pod-name>:/path/to/file ./local-file  # 复制文件
kubectl debug <pod-name> -it --image=busybox      # 调试 Pod

# ===== Pod 管理 =====
kubectl delete pod <pod-name>                     # 删除 Pod
kubectl delete pod <pod-name> --force --grace-period=0  # 强制删除
kubectl run debug --image=busybox -it --rm -- sh  # 临时调试 Pod
```

### Deployment 管理命令

```bash
# ===== Deployment 查看 =====
kubectl get deployments                           # Deployment 列表
kubectl describe deployment <name>                # Deployment 详情
kubectl get deployment <name> -o yaml             # Deployment YAML

# ===== Deployment 操作 =====
kubectl create deployment nginx --image=nginx     # 创建 Deployment
kubectl scale deployment <name> --replicas=5      # 扩缩容
kubectl set image deployment/<name> <container>=<image>  # 更新镜像
kubectl rollout status deployment/<name>          # 查看更新状态
kubectl rollout history deployment/<name>         # 查看更新历史
kubectl rollout undo deployment/<name>            # 回滚到上一版本
kubectl rollout undo deployment/<name> --to-revision=2  # 回滚到指定版本
kubectl rollout restart deployment/<name>         # 重启 Deployment
kubectl rollout pause deployment/<name>           # 暂停更新
kubectl rollout resume deployment/<name>          # 恢复更新
```

### Service 和网络命令

```bash
# ===== Service 管理 =====
kubectl get services                              # Service 列表
kubectl get svc -o wide                           # Service 详细信息
kubectl describe service <name>                   # Service 详情
kubectl get endpoints                             # Endpoints 列表
kubectl expose deployment <name> --port=80 --type=ClusterIP  # 创建 Service

# ===== 网络调试 =====
kubectl run debug --image=busybox -it --rm -- sh  # 启动调试 Pod
nslookup kubernetes.default                       # DNS 测试
wget -qO- http://<service-name>:<port>            # HTTP 测试
nc -zv <service-name> <port>                      # 端口测试

# ===== Ingress 管理 =====
kubectl get ingress                               # Ingress 列表
kubectl describe ingress <name>                   # Ingress 详情
```

### 存储管理命令

```bash
# ===== PV/PVC 管理 =====
kubectl get pv                                    # PV 列表
kubectl get pvc                                   # PVC 列表
kubectl describe pv <name>                        # PV 详情
kubectl describe pvc <name>                       # PVC 详情

# ===== StorageClass =====
kubectl get storageclass                          # StorageClass 列表
kubectl describe storageclass <name>              # StorageClass 详情
```

### 配置管理命令

```bash
# ===== ConfigMap =====
kubectl get configmaps                            # ConfigMap 列表
kubectl describe configmap <name>                 # ConfigMap 详情
kubectl create configmap <name> --from-file=<path>  # 从文件创建
kubectl create configmap <name> --from-literal=key=value  # 从字面量创建

# ===== Secret =====
kubectl get secrets                               # Secret 列表
kubectl describe secret <name>                    # Secret 详情
kubectl create secret generic <name> --from-literal=key=value  # 创建 Secret
kubectl get secret <name> -o jsonpath='{.data.key}' | base64 -d  # 解码 Secret
```

### RBAC 管理命令

```bash
# ===== 权限检查 =====
kubectl auth can-i create pods                    # 检查当前用户权限
kubectl auth can-i create pods --as=<user>        # 检查指定用户权限
kubectl auth can-i --list                         # 列出所有权限

# ===== RBAC 资源 =====
kubectl get roles                                 # Role 列表
kubectl get clusterroles                          # ClusterRole 列表
kubectl get rolebindings                          # RoleBinding 列表
kubectl get clusterrolebindings                   # ClusterRoleBinding 列表
kubectl describe role <name>                      # Role 详情
```

### 事件和监控命令

```bash
# ===== 事件 =====
kubectl get events                                # 事件列表
kubectl get events --sort-by='.lastTimestamp'     # 按时间排序
kubectl get events --field-selector type=Warning  # 警告事件
kubectl get events -w                             # 实时监控事件

# ===== 资源使用 =====
kubectl top nodes                                 # 节点资源使用
kubectl top pods                                  # Pod 资源使用
kubectl top pods --sort-by=cpu                    # 按 CPU 排序
kubectl top pods --sort-by=memory                 # 按内存排序
```

### 高级命令

```bash
# ===== 资源导出 =====
kubectl get deployment <name> -o yaml > deployment.yaml  # 导出 YAML
kubectl get all -o yaml > all-resources.yaml      # 导出所有资源

# ===== 批量操作 =====
kubectl delete pods --all                         # 删除所有 Pod
kubectl delete pods -l app=nginx                  # 按标签删除
kubectl get pods -o name | xargs kubectl delete   # 批量删除

# ===== 资源申请 =====
kubectl apply -f <file.yaml>                      # 应用配置
kubectl apply -f <directory>/                     # 应用目录下所有配置
kubectl apply -k <kustomization-dir>/             # 应用 Kustomize 配置
kubectl diff -f <file.yaml>                       # 查看差异

# ===== 标签和注解 =====
kubectl label pods <pod-name> env=prod            # 添加标签
kubectl label pods <pod-name> env-                # 删除标签
kubectl annotate pods <pod-name> description="test"  # 添加注解

# ===== 污点和容忍 =====
kubectl taint nodes <node-name> key=value:NoSchedule  # 添加污点
kubectl taint nodes <node-name> key-              # 删除污点
```

---

## 附录：EKS 常用命令速查表
### 集群管理

```bash
# ===== 集群操作 =====
aws eks list-clusters                             # 列出集群
aws eks describe-cluster --name <cluster-name>    # 集群详情
aws eks update-kubeconfig --name <cluster-name>   # 更新 kubeconfig

# ===== 集群升级 =====
aws eks update-cluster-version --name <cluster-name> --kubernetes-version 1.29
aws eks wait cluster-active --name <cluster-name>

# ===== 节点组 =====
aws eks list-nodegroups --cluster-name <cluster-name>
aws eks describe-nodegroup --cluster-name <cluster-name> --nodegroup-name <name>
aws eks update-nodegroup-version --cluster-name <cluster-name> --nodegroup-name <name>

# ===== 插件管理 =====
aws eks list-addons --cluster-name <cluster-name>
aws eks describe-addon --cluster-name <cluster-name> --addon-name <name>
aws eks update-addon --cluster-name <cluster-name> --addon-name <name>
```

### eksctl 命令

```bash
# ===== 集群创建 =====
eksctl create cluster --name <name> --region <region> --nodegroup-name <ng-name>
eksctl create cluster -f cluster.yaml             # 从配置文件创建

# ===== 节点组管理 =====
eksctl create nodegroup --cluster <cluster-name> --name <ng-name>
eksctl delete nodegroup --cluster <cluster-name> --name <ng-name>
eksctl scale nodegroup --cluster <cluster-name> --name <ng-name> --nodes 5

# ===== IAM 集成 =====
eksctl create iamserviceaccount --cluster <cluster-name> --name <sa-name> --namespace <ns> --attach-policy-arn <arn>
eksctl utils associate-iam-oidc-provider --cluster <cluster-name> --approve
```

---

## 附录：Kubernetes 资源配置最佳实践
### Pod 资源配置模板

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: best-practice-pod
  labels:
    app: myapp
    version: v1
    environment: production
spec:
  # 安全上下文
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    runAsGroup: 1000
    fsGroup: 1000
    seccompProfile:
      type: RuntimeDefault
  
  # 服务账户
  serviceAccountName: myapp-sa
  automountServiceAccountToken: false
  
  # 调度约束
  topologySpreadConstraints:
  - maxSkew: 1
    topologyKey: topology.kubernetes.io/zone
    whenUnsatisfiable: DoNotSchedule
    labelSelector:
      matchLabels:
        app: myapp
  
  affinity:
    podAntiAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        podAffinityTerm:
          labelSelector:
            matchLabels:
              app: myapp
          topologyKey: kubernetes.io/hostname
  
  containers:
  - name: app
    image: myapp:v1
    imagePullPolicy: IfNotPresent
    
    # 端口配置
    ports:
    - name: http
      containerPort: 8080
      protocol: TCP

    # 资源配置
    resources:
      requests:
        cpu: 100m
        memory: 128Mi
      limits:
        cpu: 500m
        memory: 512Mi
    
    # 容器安全上下文
    securityContext:
      allowPrivilegeEscalation: false
      readOnlyRootFilesystem: true
      capabilities:
        drop:
        - ALL
    
    # 健康检查
    livenessProbe:
      httpGet:
        path: /healthz
        port: 8080
      initialDelaySeconds: 15
      periodSeconds: 10
      timeoutSeconds: 5
      failureThreshold: 3
    
    readinessProbe:
      httpGet:
        path: /ready
        port: 8080
      initialDelaySeconds: 5
      periodSeconds: 5
      timeoutSeconds: 3
      failureThreshold: 3
    
    startupProbe:
      httpGet:
        path: /healthz
        port: 8080
      initialDelaySeconds: 0
      periodSeconds: 10
      timeoutSeconds: 5
      failureThreshold: 30
    
    # 环境变量
    env:
    - name: POD_NAME
      valueFrom:
        fieldRef:
          fieldPath: metadata.name
    - name: POD_NAMESPACE
      valueFrom:
        fieldRef:
          fieldPath: metadata.namespace
    - name: CONFIG_VALUE
      valueFrom:
        configMapKeyRef:
          name: myapp-config
          key: config-key
    - name: SECRET_VALUE
      valueFrom:
        secretKeyRef:
          name: myapp-secret
          key: secret-key
    
    # 卷挂载
    volumeMounts:
    - name: config
      mountPath: /etc/config
      readOnly: true
    - name: tmp
      mountPath: /tmp
    - name: data
      mountPath: /data
  
  # 卷定义
  volumes:
  - name: config
    configMap:
      name: myapp-config
  - name: tmp
    emptyDir: {}
  - name: data
    persistentVolumeClaim:
      claimName: myapp-data
  
  # 优雅终止
  terminationGracePeriodSeconds: 30
  
  # DNS 配置
  dnsPolicy: ClusterFirst
  dnsConfig:
    options:
    - name: ndots
      value: "2"
```

### Deployment 最佳实践模板

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
  labels:
    app: myapp
spec:
  replicas: 3
  revisionHistoryLimit: 5
  selector:
    matchLabels:
      app: myapp
  
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 25%
      maxUnavailable: 25%
  
  minReadySeconds: 10
  progressDeadlineSeconds: 600
  
  template:
    metadata:
      labels:
        app: myapp
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "8080"
        prometheus.io/path: "/metrics"
    spec:
      # ... Pod spec（参考上面的 Pod 模板）
```

### Service 最佳实践模板

```yaml
apiVersion: v1
kind: Service
metadata:
  name: myapp
  labels:
    app: myapp
spec:
  type: ClusterIP
  selector:
    app: myapp
  ports:
  - name: http
    port: 80
    targetPort: 8080
    protocol: TCP
  sessionAffinity: None
```

### HPA 最佳实践模板

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: myapp-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: myapp
  minReplicas: 2
  maxReplicas: 20
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
      - type: Percent
        value: 10
        periodSeconds: 60
    scaleUp:
      stabilizationWindowSeconds: 0
      policies:
      - type: Percent
        value: 100
        periodSeconds: 15
      - type: Pods
        value: 4
        periodSeconds: 15
      selectPolicy: Max
```

### PDB 最佳实践模板

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: myapp-pdb
spec:
  minAvailable: 2  # 或使用 maxUnavailable: 1
  selector:
    matchLabels:
      app: myapp
```

### NetworkPolicy 最佳实践模板

```yaml
# 默认拒绝所有入站流量
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-ingress
spec:
  podSelector: {}
  policyTypes:
  - Ingress
---
# 允许特定流量
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-app-traffic
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: frontend
    - namespaceSelector:
        matchLabels:
          name: monitoring
    ports:
    - protocol: TCP
      port: 8080
  egress:
  - to:
    - podSelector:
        matchLabels:
          app: database
    ports:
    - protocol: TCP
      port: 5432
  - to:  # 允许 DNS
    - namespaceSelector: {}
      podSelector:
        matchLabels:
          k8s-app: kube-dns
    ports:
    - protocol: UDP
      port: 53
```

---

## 附录：容器镜像最佳实践
### Dockerfile 最佳实践

```dockerfile
# 使用多阶段构建
FROM golang:1.21-alpine AS builder

# 设置工作目录
WORKDIR /app

# 复制依赖文件（利用缓存）
COPY go.mod go.sum ./
RUN go mod download

# 复制源代码
COPY . .

# 构建二进制文件
RUN CGO_ENABLED=0 GOOS=linux go build -a -installsuffix cgo -o main .

# 使用最小化基础镜像
FROM gcr.io/distroless/static:nonroot

# 复制二进制文件
COPY --from=builder /app/main /

# 使用非 root 用户
USER nonroot:nonroot

# 暴露端口
EXPOSE 8080

# 启动命令
ENTRYPOINT ["/main"]
```

### 镜像安全扫描

```bash
# 使用 Trivy 扫描镜像
trivy image myapp:v1

# 扫描高危漏洞
trivy image --severity HIGH,CRITICAL myapp:v1

# 生成 SBOM
trivy image --format spdx-json -o sbom.json myapp:v1
```

### 镜像大小优化

| 基础镜像 | 大小 | 适用场景 |
|----------|------|----------|
| scratch | 0 MB | 静态编译二进制 |
| distroless | 2-20 MB | Go、Java、Python |
| alpine | 5 MB | 需要 shell 调试 |
| debian-slim | 80 MB | 需要完整工具链 |
| ubuntu | 70 MB | 开发环境 |

---

## 附录：Kubernetes 监控指标参考
### 核心指标（Metrics Server）

| 指标 | 说明 | 单位 |
|------|------|------|
| cpu/usage | CPU 使用量 | 核心数 |
| memory/usage | 内存使用量 | 字节 |
| memory/working_set | 工作集内存 | 字节 |

### Prometheus 常用指标
#### 容器指标

| 指标 | 说明 | 类型 |
|------|------|------|
| container_cpu_usage_seconds_total | CPU 使用时间 | Counter |
| container_memory_working_set_bytes | 工作集内存 | Gauge |
| container_memory_usage_bytes | 内存使用量 | Gauge |
| container_network_receive_bytes_total | 网络接收字节 | Counter |
| container_network_transmit_bytes_total | 网络发送字节 | Counter |
| container_fs_reads_bytes_total | 文件系统读取 | Counter |
| container_fs_writes_bytes_total | 文件系统写入 | Counter |

#### Kubernetes 指标

| 指标 | 说明 | 类型 |
|------|------|------|
| kube_pod_status_phase | Pod 状态 | Gauge |
| kube_pod_container_status_restarts_total | 容器重启次数 | Counter |
| kube_deployment_status_replicas | Deployment 副本数 | Gauge |
| kube_deployment_status_replicas_available | 可用副本数 | Gauge |
| kube_node_status_condition | 节点状态 | Gauge |
| kube_node_status_allocatable | 节点可分配资源 | Gauge |

### 告警规则示例

```yaml
groups:
- name: kubernetes-alerts
  rules:
  # Pod 重启告警
  - alert: PodRestartingTooMuch
    expr: increase(kube_pod_container_status_restarts_total[1h]) > 5
    for: 10m
    labels:
      severity: warning
    annotations:
      summary: "Pod {{ $labels.pod }} 重启过于频繁"
      description: "Pod {{ $labels.pod }} 在过去 1 小时内重启了 {{ $value }} 次"
  
  # CPU 使用率告警
  - alert: HighCPUUsage
    expr: |
      sum(rate(container_cpu_usage_seconds_total{container!=""}[5m])) by (pod, namespace) /
      sum(kube_pod_container_resource_limits{resource="cpu"}) by (pod, namespace) > 0.9
    for: 15m
    labels:
      severity: warning
    annotations:
      summary: "Pod {{ $labels.pod }} CPU 使用率过高"
      description: "Pod {{ $labels.pod }} CPU 使用率超过 90%"
  
  # 内存使用率告警
  - alert: HighMemoryUsage
    expr: |
      sum(container_memory_working_set_bytes{container!=""}) by (pod, namespace) /
      sum(kube_pod_container_resource_limits{resource="memory"}) by (pod, namespace) > 0.9
    for: 15m
    labels:
      severity: warning
    annotations:
      summary: "Pod {{ $labels.pod }} 内存使用率过高"
      description: "Pod {{ $labels.pod }} 内存使用率超过 90%"
  
  # 节点不可用告警
  - alert: NodeNotReady
    expr: kube_node_status_condition{condition="Ready",status="true"} == 0
    for: 5m
    labels:
      severity: critical
    annotations:
      summary: "节点 {{ $labels.node }} 不可用"
      description: "节点 {{ $labels.node }} 已经 NotReady 超过 5 分钟"
```


---

## 附录：Kubernetes 网络深度解析
### Pod 网络通信模型

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Kubernetes 网络通信模型                            │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  1. 同 Pod 内容器通信（localhost）                                     │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │  Pod                                                        │    │
│  │  ┌──────────┐    localhost:port    ┌──────────┐             │    │
│  │  │Container │ ◀──────────────────▶ │Container │             │    │
│  │  │    A     │                      │    B     │             │    │
│  │  └──────────┘                      └──────────┘             │    │
│  │       │                                 │                   │    │
│  │       └──────────── eth0 ───────────────┘                   │    │
│  │                   (共享网络命名空间)                           │    │
│  └─────────────────────────────────────────────────────────────┘    │
│                                                                     │
│  2. 同节点 Pod 间通信（veth pair + bridge）                            │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │  Node                                                       │    │
│  │  ┌──────────┐                       ┌──────────┐            │    │
│  │  │  Pod A   │                       │  Pod B   │            │    │
│  │  │ 10.0.1.2 │                       │ 10.0.1.3 │            │    │
│  │  └────┬─────┘                       └────┬─────┘            │    │
│  │       │ veth                             │ veth             │    │
│  │       └──────────┬───────────────────────┘                  │    │
│  │                  │                                          │    │
│  │           ┌──────┴──────┐                                   │    │
│  │           │   cni0      │  (Linux Bridge)                   │    │
│  │           │  10.0.1.1   │                                   │    │
│  │           └─────────────┘                                   │    │
│  └─────────────────────────────────────────────────────────────┘    │
│                                                                     │
│  3. 跨节点 Pod 通信（Overlay/Underlay）                               │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │                                                              │   │
│  │  Node 1                              Node 2                  │   │
│  │  ┌──────────┐                       ┌──────────┐             │   │
│  │  │  Pod A   │                       │  Pod B   │             │   │
│  │  │ 10.0.1.2 │                       │ 10.0.2.3 │             │   │
│  │  └────┬─────┘                       └────┬─────┘             │   │
│  │       │                                  │                   │   │
│  │  ┌────┴────┐                        ┌────┴────┐              │   │
│  │  │  cni0   │                        │  cni0   │              │   │
│  │  │10.0.1.1 │                        │10.0.2.1 │              │   │
│  │  └────┬────┘                        └────┬────┘              │   │
│  │       │                                  │                   │   │
│  │  ┌────┴────┐    VXLAN/BGP/VPC       ┌────┴────┐              │   │
│  │  │  eth0   │ ◀──────────────────▶   │  eth0   │              │   │
│  │  │192.168.1│                        │192.168.2│              │   │
│  │  └─────────┘                        └─────────┘              │   │
│  │                                                              │   │
│  └──────────────────────────────────────────────────────────────┘   │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Service 实现原理

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Service 实现原理（iptables 模式）                  │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Client Pod                                                         │
│       │                                                             │
│       │ 访问 ClusterIP:Port (10.96.0.1:80)                           │
│       ▼                                                             │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │  iptables 规则链                                             │    │
│  │                                                             │    │
│  │  PREROUTING → KUBE-SERVICES                                 │    │
│  │       │                                                     │    │
│  │       ▼                                                     │    │
│  │  KUBE-SVC-XXX (Service 规则)                                 │    │
│  │       │                                                     │    │
│  │       │ 随机选择后端 (概率负载均衡)                             │    │
│  │       ├─── 33% ──▶ KUBE-SEP-AAA (Pod 1: 10.0.1.2:8080)      │    │
│  │       ├─── 33% ──▶ KUBE-SEP-BBB (Pod 2: 10.0.1.3:8080)      │    │
│  │       └─── 34% ──▶ KUBE-SEP-CCC (Pod 3: 10.0.2.2:8080)      │    │
│  │                                                             │    │
│  │  KUBE-SEP-XXX (Endpoint 规则)                                │    │
│  │       │                                                     │    │
│  │       │ DNAT: 10.96.0.1:80 → 10.0.1.2:8080                  │    │
│  │       ▼                                                     │    │
│  │  POSTROUTING → KUBE-POSTROUTING                             │    │
│  │       │                                                     │    │
│  │       │ SNAT (如果需要)                                      │    │
│  │       ▼                                                     │    │
│  └─────────────────────────────────────────────────────────────┘    │
│                                                                     │
│  Backend Pod (10.0.1.2:8080)                                        │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### IPVS vs iptables 详细对比

| 维度 | iptables | IPVS |
|------|----------|------|
| **数据结构** | 链表（O(n)） | 哈希表（O(1)） |
| **规则数量** | 每个 Service 多条规则 | 每个 Service 一条规则 |
| **性能（1000 Service）** | 延迟增加明显 | 延迟稳定 |
| **负载均衡算法** | 随机 | rr/lc/dh/sh/sed/nq |
| **连接跟踪** | conntrack | IPVS 内置 |
| **会话保持** | 有限支持 | 完整支持 |
| **健康检查** | 不支持 | 支持 |
| **调试难度** | 中等 | 较高 |
| **K8S 默认** | ✅ 默认 | 需要配置 |

**启用 IPVS 模式：**
```yaml
# kube-proxy ConfigMap
apiVersion: v1
kind: ConfigMap
metadata:
  name: kube-proxy
  namespace: kube-system
data:
  config.conf: |
    mode: "ipvs"
    ipvs:
      scheduler: "rr"  # 轮询算法
      strictARP: true
```


### DNS 解析流程

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Kubernetes DNS 解析流程                           │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Pod 内应用                                                          │
│       │                                                             │
│       │ 解析 myservice.default.svc.cluster.local                     │
│       ▼                                                             │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │  /etc/resolv.conf                                           │    │
│  │  nameserver 10.96.0.10  (CoreDNS ClusterIP)                 │    │
│  │  search default.svc.cluster.local svc.cluster.local         │    │
│  │  options ndots:5                                            │    │
│  └─────────────────────────────────────────────────────────────┘    │
│       │                                                             │
│       │ DNS 查询                                                     │
│       ▼                                                             │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │  CoreDNS (kube-system)                                      │    │
│  │                                                             │    │
│  │  Corefile 配置：                                             │    │
│  │  .:53 {                                                     │    │
│  │      kubernetes cluster.local in-addr.arpa ip6.arpa {       │    │
│  │          pods insecure                                      │    │
│  │          fallthrough in-addr.arpa ip6.arpa                  │    │
│  │      }                                                      │    │
│  │      forward . /etc/resolv.conf                             │    │
│  │      cache 30                                               │    │
│  │  }                                                          │    │
│  └─────────────────────────────────────────────────────────────┘    │
│       │                                                             │
│       │ 查询 API Server 获取 Service/Endpoints                       │
│       ▼                                                             │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │  返回 ClusterIP: 10.96.0.100                                 │    │
│  └─────────────────────────────────────────────────────────────┘    │
│                                                                     │
│  DNS 记录类型：                                                      │
│  • A 记录: myservice.default.svc.cluster.local → ClusterIP          │
│  • SRV 记录: _http._tcp.myservice.default.svc.cluster.local         │
│  • Headless Service: 返回所有 Pod IP                                 │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### ndots 配置影响

| ndots 值 | 查询 "myservice" | 查询次数 | 说明 |
|----------|------------------|----------|------|
| 5（默认） | 先搜索 search 域 | 最多 6 次 | 适合内部服务 |
| 2 | 先搜索 search 域 | 最多 3 次 | 平衡内外部 |
| 1 | 直接查询 | 1 次 | 适合外部域名 |

**优化 DNS 配置：**
```yaml
spec:
  dnsConfig:
    options:
    - name: ndots
      value: "2"
    - name: single-request-reopen
```

---

## 附录：Kubernetes 存储深度解析
### CSI 架构详解

```
┌─────────────────────────────────────────────────────────────────────┐
│                    CSI 架构详解                                      │
├─────────────────────────────────────────────────────────────────────┤
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │                    Kubernetes 控制平面                       │    │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐       │    │
│  │  │ PV Controller│  │ Attach/Detach│  │ Volume       │       │    │
│  │  │              │  │ Controller   │  │ Scheduler    │       │    │
│  │  └──────────────┘  └──────────────┘  └──────────────┘       │    │
│  └─────────────────────────────────────────────────────────────┘    │
│                              │ CSI gRPC                             │
│                              ▼                                      │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │                    CSI Controller Plugin                    │    │
│  │                    (Deployment)                             │    │
│  │  ┌──────────────────────────────────────────────────────┐   │    │
│  │  │  Sidecar Containers                                  │   │    │
│  │  │  ┌────────────────┐  ┌────────────────┐              │   │    │
│  │  │  │ external-      │  │ external-      │              │   │    │
│  │  │  │ provisioner    │  │ attacher       │              │   │    │
│  │  │  │ (CreateVolume) │  │ (Attach/Detach)│              │   │    │
│  │  │  └────────────────┘  └────────────────┘              │   │    │
│  │  │  ┌────────────────┐  ┌────────────────┐              │   │    │
│  │  │  │ external-      │  │ external-      │              │   │    │
│  │  │  │ snapshotter    │  │ resizer        │              │   │    │
│  │  │  │ (Snapshot)     │  │ (Expand)       │              │   │    │
│  │  │  └────────────────┘  └────────────────┘              │   │    │
│  │  └──────────────────────────────────────────────────────┘   │    │
│  │                              │ CSI gRPC                     │    │
│  │                              ▼                              │    │
│  │  ┌──────────────────────────────────────────────────────┐   │    │
│  │  │  CSI Driver Container                                │   │    │
│  │  │  (实现 Controller Service)                            │   │    │
│  │  └──────────────────────────────────────────────────────┘   │    │
│  └─────────────────────────────────────────────────────────────┘    │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │                    CSI Node Plugin                          │    │
│  │                    (DaemonSet - 每个节点)                    │    │
│  │  ┌──────────────────────────────────────────────────────┐   │    │
│  │  │  Sidecar Container                                   │   │    │
│  │  │  ┌────────────────────────────────────────────────┐  │   │    │
│  │  │  │ node-driver-registrar                          │  │   │    │
│  │  │  │ (注册 CSI Driver 到 kubelet)                    │  │   │    │
│  │  │  └────────────────────────────────────────────────┘  │   │    │
│  │  └──────────────────────────────────────────────────────┘   │    │
│  │                              │ CSI gRPC                     │    │
│  │                              ▼                              │    │
│  │  ┌──────────────────────────────────────────────────────┐   │    │
│  │  │  CSI Driver Container                                │   │    │
│  │  │  (实现 Node Service)                                  │   │    │
│  │  │  • NodeStageVolume (挂载到 staging 目录)               │   │    │
│  │  │  • NodePublishVolume (挂载到 Pod 目录)                 │   │    │
│  │  └──────────────────────────────────────────────────────┘   │    │
│  └─────────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────────┘
```


### 卷生命周期

```
┌─────────────────────────────────────────────────────────────────────┐
│                    卷生命周期                                         │
├─────────────────────────────────────────────────────────────────────┤
│  1. Provisioning（供应）                                             │
│     ┌─────────────────────────────────────────────────────────┐     │
│     │  PVC 创建 → external-provisioner 监听                    │     │
│     │           → CSI CreateVolume                            │     │
│     │           → 云平台创建卷（如 EBS）                        │     │
│     │           → 创建 PV 并绑定 PVC                           │     │
│     └─────────────────────────────────────────────────────────┘     │
│                              │                                      │
│                              ▼                                      │
│  2. Attaching（挂载到节点）                                           │
│     ┌─────────────────────────────────────────────────────────┐     │
│     │  Pod 调度到节点 → Attach/Detach Controller 监听           │     │
│     │                → external-attacher 调用                  │     │
│     │                → CSI ControllerPublishVolume            │     │
│     │                → 云平台挂载卷到 EC2                       │     │
│     │                → 创建 VolumeAttachment 对象              │     │
│     └─────────────────────────────────────────────────────────┘     │
│                              │                                      │
│                              ▼                                      │
│  3. Staging（挂载到 staging 目录）                                    │
│     ┌─────────────────────────────────────────────────────────┐     │
│     │  kubelet 调用 CSI NodeStageVolume                        │     │
│     │  → 格式化卷（如果需要）                                    │     │
│     │  → 挂载到 /var/lib/kubelet/plugins/kubernetes.io/csi/    │     │
│     │     pv/<pv-name>/globalmount                            │     │
│     └─────────────────────────────────────────────────────────┘     │
│                              │                                      │
│                              ▼                                      │
│  4. Publishing（挂载到 Pod 目录）                                     │
│     ┌─────────────────────────────────────────────────────────┐     │
│     │  kubelet 调用 CSI NodePublishVolume                      │     │
│     │  → bind mount 到 Pod 目录                                │     │
│     │  → /var/lib/kubelet/pods/<pod-uid>/volumes/             │     │
│     │     kubernetes.io~csi/<pv-name>/mount                   │     │
│     └─────────────────────────────────────────────────────────┘     │
│                              │                                      │
│                              ▼                                      │
│  5. Pod 使用卷                                                       │
│                              │                                      │
│                              ▼                                      │
│  6. Unpublishing（卸载 Pod 目录）                                     │
│     ┌─────────────────────────────────────────────────────────┐     │
│     │  Pod 删除 → kubelet 调用 CSI NodeUnpublishVolume         │     │
│     └─────────────────────────────────────────────────────────┘     │
│                              │                                      │
│                              ▼                                      │
│  7. Unstaging（卸载 staging 目录）                                    │
│     ┌─────────────────────────────────────────────────────────┐     │
│     │  kubelet 调用 CSI NodeUnstageVolume                      │     │
│     └─────────────────────────────────────────────────────────┘     │
│                              │                                      │
│                              ▼                                      │
│  8. Detaching（从节点卸载）                                           │
│     ┌─────────────────────────────────────────────────────────┐     │
│     │  external-attacher 调用 CSI ControllerUnpublishVolume    │     │
│     │  → 云平台从 EC2 卸载卷                                     │     │
│     └─────────────────────────────────────────────────────────┘     │
│                              │                                      │
│                              ▼                                      │
│  9. Deleting（删除卷）- 如果 reclaimPolicy=Delete                      │
│     ┌─────────────────────────────────────────────────────────┐     │
│     │  PVC 删除 → external-provisioner 调用 CSI DeleteVolume   │     │
│     │           → 云平台删除卷                                  │     │
│     └─────────────────────────────────────────────────────────┘     │
└─────────────────────────────────────────────────────────────────────┘
```

### 存储性能对比

| 存储类型 | IOPS | 吞吐量 | 延迟 | 适用场景 |
|----------|------|--------|------|----------|
| **EBS gp3** | 3000-16000 | 125-1000 MB/s | 1-2ms | 通用工作负载 |
| **EBS io2** | 64000 | 1000 MB/s | <1ms | 高性能数据库 |
| **EBS st1** | 500 | 500 MB/s | 5-10ms | 大数据、日志 |
| **EFS** | 弹性 | 弹性 | 5-10ms | 共享存储 |
| **Local NVMe** | 100000+ | 2000+ MB/s | <0.1ms | 缓存、临时数据 |

---

## 附录：Kubernetes 安全深度解析
### 攻击面分析

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Kubernetes 攻击面                                 │
├─────────────────────────────────────────────────────────────────────┤
│  外部攻击面：                                                         │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │  • API Server 暴露（未授权访问）                               │    │
│  │  • Ingress/LoadBalancer 漏洞                                 │    │
│  │  • 容器镜像漏洞                                               │    │
│  │  • 供应链攻击（恶意镜像）                                       │    │
│  └─────────────────────────────────────────────────────────────┘    │
│  内部攻击面：                                                         │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │  • 容器逃逸（内核漏洞、配置错误）                                │    │
│  │  • 横向移动（Pod 间网络访问）                                   │    │
│  │  • 权限提升（RBAC 配置错误）                                   │    │
│  │  • Secret 泄露（etcd 未加密、日志暴露）                         │    │
│  │  • ServiceAccount Token 滥用                                 │    │
│  └─────────────────────────────────────────────────────────────┘    │
│  防护措施：                                                          │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │  • API Server：私有网络、RBAC、审计日志                         │    │
│  │  • 网络：NetworkPolicy、Service Mesh mTLS                    │    │
│  │  • Pod：PSA、Seccomp、AppArmor、非 root 运行                  │    │
│  │  • 镜像：扫描、签名、最小化基础镜像                              │    │
│  │  • Secret：外部管理（Vault、Secrets Manager）                 │    │
│  │  • 运行时：Falco、Sysdig 异常检测                              │    │
│  └─────────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────────┘
```

### 安全配置检查清单

| 类别 | 检查项 | 风险等级 | 修复建议 |
|------|--------|----------|----------|
| **API Server** | 匿名访问是否禁用 | 高 | --anonymous-auth=false |
| | 是否启用审计日志 | 高 | 配置 audit-policy |
| | 是否限制网络访问 | 高 | 使用私有 VPC |
| **etcd** | 是否启用加密 | 高 | 配置 encryption-provider |
| | 是否限制访问 | 高 | 仅允许 API Server 访问 |
| **RBAC** | 是否使用最小权限 | 高 | 避免 cluster-admin |
| | ServiceAccount 是否隔离 | 中 | 每个应用独立 SA |
| **Pod** | 是否以 root 运行 | 高 | runAsNonRoot: true |
| | 是否允许特权 | 高 | privileged: false |
| | 是否只读文件系统 | 中 | readOnlyRootFilesystem: true |
| **网络** | 是否有 NetworkPolicy | 高 | 默认拒绝 + 白名单 |
| | 是否启用 mTLS | 中 | 使用 Service Mesh |
| **镜像** | 是否扫描漏洞 | 高 | CI/CD 集成 Trivy |
| | 是否使用签名 | 中 | Cosign + 准入策略 |
| **Secret** | 是否使用外部管理 | 高 | Secrets Manager/Vault |
| | 是否启用 KMS 加密 | 高 | 配置 KMS provider |


---

## 附录：Kubernetes 调度深度解析
### 调度器架构

```
┌─────────────────────────────────────────────────────────────────────┐
│                    kube-scheduler 架构                               │
├─────────────────────────────────────────────────────────────────────┤
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │                    Scheduling Queue                         │    │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────────┐               │    │
│  │  │ activeQ  │  │backoffQ  │  │unschedulableQ│               │    │
│  │  │(待调度)   │  │(退避队列) │  │  (不可调度)    │               │    │
│  │  └──────────┘  └──────────┘  └──────────────┘               │    │
│  └─────────────────────────────────────────────────────────────┘    │
│                              │ Pop Pod                              │
│                              ▼                                      │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │                    Scheduling Cycle                         │    │
│  │  1. PreFilter  → 预处理，计算 Pod 需求                        │    │
│  │  2. Filter     → 过滤不满足条件的节点                          │    │
│  │  3. PostFilter → 过滤后处理（抢占）                            │    │
│  │  4. PreScore   → 评分预处理                                   │    │
│  │  5. Score      → 对可用节点评分                               │    │
│  │  6. NormalizeScore → 归一化分数                              │    │
│  │  7. Reserve    → 预留资源                                    │    │
│  │  8. Permit     → 等待/批准/拒绝                              │    │
│  │  9. PreBind    → 绑定前处理                                  │    │
│  │  10. Bind      → 绑定 Pod 到节点                             │    │
│  │  11. PostBind  → 绑定后处理                                  │    │
│  └─────────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────────┘
```

### Scheduling Framework 扩展点

| 扩展点 | 阶段 | 说明 | 示例插件 |
|--------|------|------|----------|
| **PreEnqueue** | 入队前 | 决定 Pod 是否入队 | - |
| **PreFilter** | 过滤前 | 预处理，计算 Pod 需求 | NodeResourcesFit |
| **Filter** | 过滤 | 过滤不满足条件的节点 | NodeAffinity, TaintToleration |
| **PostFilter** | 过滤后 | 处理无可用节点情况 | DefaultPreemption |
| **PreScore** | 评分前 | 评分预处理 | InterPodAffinity |
| **Score** | 评分 | 对可用节点评分 | NodeResourcesBalancedAllocation |
| **NormalizeScore** | 归一化 | 归一化分数到 0-100 | - |
| **Reserve** | 预留 | 预留资源 | VolumeBinding |
| **Permit** | 许可 | 等待/批准/拒绝 | - |
| **PreBind** | 绑定前 | 绑定前处理 | VolumeBinding |
| **Bind** | 绑定 | 绑定 Pod 到节点 | DefaultBinder |
| **PostBind** | 绑定后 | 绑定后处理 | - |


### 调度算法详解
#### Filter 阶段插件

| 插件                    | 功能     | 过滤条件                           |
| --------------------- | ------ | ------------------------------ |
| **NodeResourcesFit**  | 资源检查   | 节点可用资源 >= Pod 请求               |
| **NodeName**          | 节点名称   | spec.nodeName 匹配               |
| **NodePorts**         | 端口检查   | hostPort 不冲突                   |
| **NodeAffinity**      | 节点亲和   | 满足 nodeAffinity 规则             |
| **TaintToleration**   | 污点容忍   | Pod 容忍节点 Taint                 |
| **NodeUnschedulable** | 可调度性   | 节点未标记为不可调度                     |
| **PodTopologySpread** | 拓扑分布   | 满足 topologySpreadConstraints   |
| **InterPodAffinity**  | Pod 亲和 | 满足 podAffinity/podAntiAffinity |
| **VolumeBinding**     | 卷绑定    | PVC 可绑定到节点                     |
| **VolumeZone**        | 卷区域    | 卷和节点在同一区域                      |

#### Score 阶段插件

| 插件 | 功能 | 评分逻辑 |
|------|------|----------|
| **NodeResourcesBalancedAllocation** | 资源均衡 | CPU/内存使用率接近的节点得分高 |
| **NodeResourcesLeastAllocated** | 最少分配 | 资源使用率低的节点得分高 |
| **NodeResourcesMostAllocated** | 最多分配 | 资源使用率高的节点得分高（bin-packing） |
| **NodeAffinity** | 节点亲和 | 满足 preferred 规则的节点得分高 |
| **InterPodAffinity** | Pod 亲和 | 满足 preferred 规则的节点得分高 |
| **TaintToleration** | 污点容忍 | 容忍更多 Taint 的节点得分高 |
| **ImageLocality** | 镜像本地性 | 已有镜像的节点得分高 |
| **PodTopologySpread** | 拓扑分布 | 分布更均匀的节点得分高 |

### 调度性能优化

| 参数 | 默认值 | 调优建议 | 说明 |
|------|--------|----------|------|
| **percentageOfNodesToScore** | 50% | 10-30%（大集群） | 评分节点比例 |
| **parallelism** | 16 | 32（大集群） | 并行调度数 |
| **podInitialBackoffSeconds** | 1s | 1s | 初始退避时间 |
| **podMaxBackoffSeconds** | 10s | 10s | 最大退避时间 |

---

## 附录：Kubernetes 控制器深度解析
### 控制器模式

```
┌─────────────────────────────────────────────────────────────────────┐
│                    控制器模式（Reconciliation Loop）                  │
├─────────────────────────────────────────────────────────────────────┤
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │                    期望状态（Desired State）                  │    │
│  │                    存储在 etcd 中                            │    │
│  │                    例如：replicas: 3                         │    │
│  └─────────────────────────────────────────────────────────────┘    │
│                              │ 比较                                  │
│                              ▼                                      │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │                    Reconcile Loop                           │    │
│  │  while true:                                                │    │
│  │      desired = getDesiredState()                            │    │
│  │      actual = getActualState()                              │    │
│  │      if desired != actual:                                  │    │
│  │          makeChanges(desired, actual)                       │    │
│  │      sleep(interval)                                        │    │
│  └─────────────────────────────────────────────────────────────┘    │
│                              │ 调整                                  │
│                              ▼                                      │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │                    实际状态（Actual State）                   │    │
│  │                    集群中的实际资源                           │    │
│  │                    例如：当前 Pod 数量                        │    │
│  └─────────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────────┘
```


### Deployment Controller 工作流程

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Deployment Controller 工作流程                    │
├─────────────────────────────────────────────────────────────────────┤
│  用户创建/更新 Deployment                                             │
│       │                                                             │
│       ▼                                                             │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │  Deployment Controller                                      │    │
│  │  1. 监听 Deployment 变更（Informer）                          │    │
│  │  2. 计算期望的 ReplicaSet                                    │    │
│  │  3. 创建/更新 ReplicaSet                                     │    │
│  │  4. 管理滚动更新策略                                          │    │
│  │  5. 更新 Deployment 状态                                     │    │
│  └─────────────────────────────────────────────────────────────┘    │
│       │ 创建/更新 ReplicaSet                                         │
│       ▼                                                             │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │  ReplicaSet Controller                                      │    │
│  │  1. 监听 ReplicaSet 变更（Informer）                          │    │
│  │  2. 计算期望的 Pod 数量                                       │    │
│  │  3. 创建/删除 Pod 以匹配期望数量                               │    │
│  │  4. 更新 ReplicaSet 状态                                     │    │
│  └─────────────────────────────────────────────────────────────┘    │
│       │ 创建 Pod                                                    │
│       ▼                                                             │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │  Scheduler                                                  │    │
│  │  1. 监听未调度的 Pod（Informer）                              │    │
│  │  2. 选择合适的节点                                            │    │
│  │  3. 绑定 Pod 到节点                                          │    │
│  └─────────────────────────────────────────────────────────────┘    │
│       │ 绑定到节点                                                   │
│       ▼                                                             │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │  kubelet                                                    │    │
│  │  1. 监听分配到本节点的 Pod（Informer）                          │    │
│  │  2. 调用 CRI 创建容器                                         │    │
│  │  3. 配置网络（CNI）和存储（CSI）                                │    │
│  │  4. 运行健康检查                                              │    │
│  │  5. 上报 Pod 状态                                            │    │
│  └─────────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────────┘
```

### 控制器并发参数

| 控制器 | 参数 | 默认值 | 说明 |
|--------|------|--------|------|
| **Deployment** | --concurrent-deployment-syncs | 5 | 并发同步数 |
| **ReplicaSet** | --concurrent-replicaset-syncs | 5 | 并发同步数 |
| **StatefulSet** | --concurrent-statefulset-syncs | 5 | 并发同步数 |
| **Endpoint** | --concurrent-endpoint-syncs | 5 | 并发同步数 |
| **Service** | --concurrent-service-syncs | 1 | 并发同步数 |
| **Namespace** | --concurrent-namespace-syncs | 10 | 并发同步数 |
| **GC** | --concurrent-gc-syncs | 20 | 并发同步数 |

---

## 附录：EKS 架构深度解析
### EKS 控制平面架构

```
┌─────────────────────────────────────────────────────────────────────┐
│                    EKS 控制平面架构                                   │
├─────────────────────────────────────────────────────────────────────┤
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │                    AWS 托管控制平面                           │    │
│  │                    (跨 3 个 AZ 高可用)                        │    │
│  │  ┌──────────────────────────────────────────────────────┐   │    │
│  │  │  API Server (多副本)                                  │   │    │
│  │  │  • 私有 VPC 端点                                      │   │    │
│  │  │  • NLB 负载均衡                                       │   │    │
│  │  │  • 自动扩缩容                                         │   │    │
│  │  └──────────────────────────────────────────────────────┘   │    │
│  │                              │                              │    │
│  │  ┌──────────────────────────────────────────────────────┐   │    │
│  │  │  etcd 集群 (3 节点)                                   │   │    │
│  │  │  • 加密存储                                           │   │    │
│  │  │  • 自动备份                                           │   │    │
│  │  │  • 跨 AZ 复制                                         │   │    │
│  │  └──────────────────────────────────────────────────────┘   │    │
│  │                              │                              │    │
│  │  ┌──────────────────────────────────────────────────────┐   │    │
│  │  │  Controller Manager + Scheduler                      │   │    │
│  │  │  • 高可用部署                                          │   │    │
│  │  │  • Leader Election                                   │   │    │
│  │  └──────────────────────────────────────────────────────┘   │    │
│  └─────────────────────────────────────────────────────────────┘    │
│                              │ ENI (跨账户)                          │
│                              ▼                                      │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │                    客户 VPC                                  │    │
│  │  ┌──────────────────────────────────────────────────────┐   │    │
│  │  │  Worker Nodes (EC2/Fargate)                          │   │    │
│  │  │  • kubelet                                           │   │    │
│  │  │  • kube-proxy                                        │   │    │
│  │  │  • VPC CNI                                           │   │    │
│  │  │  • 应用 Pod                                           │   │    │
│  │  └──────────────────────────────────────────────────────┘   │    │
│  └─────────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────────┘
```


### EKS 网络架构

```
┌─────────────────────────────────────────────────────────────────────┐
│                    EKS 网络架构                                      │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │                    VPC (10.0.0.0/16)                        │    │
│  │                                                             │    │
│  │  ┌──────────────────────────────────────────────────────┐   │    │
│  │  │  Public Subnets (10.0.0.0/24, 10.0.1.0/24)           │   │    │
│  │  │  • NAT Gateway                                       │   │    │
│  │  │  • ALB/NLB                                           │   │    │
│  │  │  • Bastion Host (可选)                                │   │    │
│  │  └──────────────────────────────────────────────────────┘   │    │
│  │                              │                              │    │
│  │  ┌──────────────────────────────────────────────────────┐   │    │
│  │  │  Private Subnets (10.0.10.0/24, 10.0.11.0/24)        │   │    │
│  │  │  • Worker Nodes                                      │   │    │
│  │  │  • Pod IP (VPC CNI)                                  │   │    │
│  │  │  • VPC Endpoints                                     │   │    │
│  │  └──────────────────────────────────────────────────────┘   │    │
│  │                                                             │    │
│  │  VPC Endpoints (私有访问 AWS 服务):                           │    │
│  │  • com.amazonaws.region.ec2                                 │    │
│  │  • com.amazonaws.region.ecr.api                             │    │
│  │  • com.amazonaws.region.ecr.dkr                             │    │
│  │  • com.amazonaws.region.s3                                  │    │
│  │  • com.amazonaws.region.sts                                 │    │
│  │  • com.amazonaws.region.logs                                │    │
│  │  • com.amazonaws.region.elasticloadbalancing                │    │
│  └─────────────────────────────────────────────────────────────┘    │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### EKS 与自建 K8S 对比

| 维度 | EKS | 自建 K8S |
|------|-----|----------|
| **控制平面管理** | AWS 托管 | 自行管理 |
| **控制平面成本** | $0.10/小时 | EC2 成本 |
| **高可用** | 自动（跨 3 AZ） | 需要配置 |
| **升级** | 一键升级 | 手动升级 |
| **etcd 备份** | 自动 | 需要配置 |
| **安全补丁** | AWS 管理 | 自行管理 |
| **IAM 集成** | 原生支持 | 需要配置 |
| **VPC CNI** | 原生支持 | 需要安装 |
| **负载均衡** | ALB/NLB 集成 | 需要配置 |
| **日志/监控** | CloudWatch 集成 | 需要配置 |
| **灵活性** | 受限 | 完全控制 |
| **学习曲线** | 低 | 高 |

### EKS 成本优化

| 策略 | 节省比例 | 说明 |
|------|----------|------|
| **Spot 实例** | 60-90% | 无状态工作负载 |
| **Savings Plans** | 30-60% | 稳定工作负载 |
| **Graviton 实例** | 20-40% | ARM 架构 |
| **Fargate Spot** | 70% | 无状态 Fargate |
| **Karpenter** | 20-30% | 智能节点管理 |
| **VPA** | 10-30% | 资源优化 |

---

## 附录：容器化迁移指南
### 迁移评估清单

| 评估维度 | 检查项 | 说明 |
|----------|--------|------|
| **应用架构** | 是否无状态 | 无状态应用更易容器化 |
| | 是否有外部依赖 | 数据库、缓存、消息队列 |
| | 是否有本地存储 | 需要迁移到 PV/PVC |
| | 是否有定时任务 | 使用 CronJob |
| **配置管理** | 配置是否外部化 | 使用 ConfigMap/Secret |
| | 是否有硬编码 | 需要参数化 |
| | 是否有环境差异 | 使用 Kustomize/Helm |
| **日志和监控** | 日志是否输出到 stdout | 容器最佳实践 |
| | 是否有健康检查端点 | 配置探针 |
| | 是否有指标端点 | Prometheus 采集 |
| **网络** | 是否有固定 IP 需求 | 使用 Service |
| | 是否有端口冲突 | 容器端口隔离 |
| | 是否有特殊网络需求 | NetworkPolicy |
| **安全** | 是否需要 root 权限 | 尽量避免 |
| | 是否有敏感信息 | 使用 Secret |
| | 是否有合规要求 | PSA/NetworkPolicy |

### 迁移步骤

```
┌─────────────────────────────────────────────────────────────────────┐
│                    容器化迁移步骤                                     │
├─────────────────────────────────────────────────────────────────────┤
│  1. 评估阶段                                                         │
│     ├─ 应用架构分析                                                   │
│     ├─ 依赖关系梳理                                                   │
│     ├─ 资源需求评估                                                   │
│     └─ 迁移风险评估                                                   │
│                              │                                      │
│                              ▼                                      │
│  2. 准备阶段                                                         │
│     ├─ 编写 Dockerfile                                               │
│     ├─ 构建容器镜像                                                   │
│     ├─ 镜像安全扫描                                                   │ 
│     └─ 本地测试验证                                                   │
│                              │                                      │
│                              ▼                                      │
│  3. 配置阶段                                                         │
│     ├─ 编写 K8S 资源清单                                             │
│     ├─ 配置 ConfigMap/Secret                                        │
│     ├─ 配置 PV/PVC（如需要）                                         │
│     └─ 配置 Service/Ingress                                         │
│                              │                                     │
│                              ▼                                     │
│  4. 部署阶段                                                        │
│     ├─ 部署到开发环境                                                │
│     ├─ 功能测试                                                     │
│     ├─ 性能测试                                                      │
│     └─ 安全测试                                                      │
│                              │                                      │
│                              ▼                                      │
│  5. 上线阶段                                                         │
│     ├─ 部署到预发环境                                                 │
│     ├─ 灰度发布                                                      │
│     ├─ 监控告警配置                                                   │
│     └─ 全量上线                                                      │
│                              │                                      │
│                              ▼                                      │
│  6. 优化阶段                                                         │
│     ├─ 资源调优（VPA）                                               │
│     ├─ 弹性伸缩（HPA）                                               │
│     ├─ 成本优化                                                      │
│     └─ 持续改进                                                      │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```


---

## 附录：Kubernetes 故障排查手册
### 故障排查流程图

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Kubernetes 故障排查流程                            │
├─────────────────────────────────────────────────────────────────────┤
│  问题发现                                                            │
│       │                                                             │
│       ▼                                                             │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │  1. 确定问题范围                                              │    │
│  │     • 单个 Pod？多个 Pod？整个集群？                            │    │
│  │     • 特定命名空间？特定节点？                                  │    │
│  │     • 何时开始？是否有变更？                                   │    │
│  └─────────────────────────────────────────────────────────────┘    │
│       │                                                             │
│       ▼                                                             │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │  2. 收集信息                                                 │    │
│  │     • kubectl get pods -o wide                              │    │
│  │     • kubectl describe pod <pod-name>                       │    │
│  │     • kubectl logs <pod-name>                               │    │
│  │     • kubectl get events --sort-by='.lastTimestamp'         │    │
│  └─────────────────────────────────────────────────────────────┘    │
│       │                                                             │
│       ▼                                                             │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │  3. 分析问题类型                                              │    │
│  │     ├─ Pod 问题 → 见 Pod 故障排查                             │    │
│  │     ├─ 网络问题 → 见网络故障排查                               │    │
│  │     ├─ 存储问题 → 见存储故障排查                               │    │
│  │     ├─ 节点问题 → 见节点故障排查                               │    │
│  │     └─ 控制平面问题 → 见控制平面故障排查                         │    │
│  └─────────────────────────────────────────────────────────────┘    │
│       │                                                             │
│       ▼                                                             │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │  4. 实施修复                                                 │    │
│  │     • 应用修复方案                                            │    │
│  │     • 验证修复效果                                            │    │
│  │     • 记录问题和解决方案                                       │    │
│  └─────────────────────────────────────────────────────────────┘    │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Pod 故障排查

| 状态 | 可能原因 | 排查命令 | 解决方案 |
|------|----------|----------|----------|
| **Pending** | 资源不足 | `kubectl describe pod` | 扩容节点或调整 requests |
| | Taint 不匹配 | `kubectl describe node` | 添加 Tolerations |
| | PVC 未绑定 | `kubectl get pvc` | 检查 StorageClass |
| | 节点选择器不匹配 | `kubectl get nodes --show-labels` | 调整 nodeSelector |
| **ContainerCreating** | 镜像拉取失败 | `kubectl describe pod` | 检查镜像名和认证 |
| | CNI 问题 | `kubectl logs -n kube-system <cni-pod>` | 重启 CNI 插件 |
| | 卷挂载失败 | `kubectl describe pod` | 检查 CSI 驱动 |
| **CrashLoopBackOff** | 应用错误 | `kubectl logs --previous` | 检查应用日志 |
| | 配置错误 | `kubectl describe pod` | 检查 ConfigMap/Secret |
| | 资源不足 | `kubectl top pod` | 增加 limits |
| **OOMKilled** | 内存不足 | `kubectl describe pod` | 增加 memory limits |
| | 内存泄漏 | 应用监控 | 修复应用 |
| **Evicted** | 节点资源压力 | `kubectl describe node` | 清理节点或扩容 |
| | 超过配额 | `kubectl describe resourcequota` | 调整配额 |
| **ImagePullBackOff** | 镜像不存在 | `kubectl describe pod` | 检查镜像名 |
| | 认证失败 | `kubectl get secret` | 检查 imagePullSecrets |
| | 网络问题 | 节点网络测试 | 检查网络连通性 |


### 网络故障排查

| 问题 | 症状 | 排查步骤 | 解决方案 |
|------|------|----------|----------|
| **DNS 解析失败** | nslookup 失败 | 1. 检查 CoreDNS Pod 状态<br>2. 检查 /etc/resolv.conf<br>3. 测试 DNS 服务 | 重启 CoreDNS 或检查配置 |
| **Service 不通** | curl ClusterIP 超时 | 1. 检查 Endpoints<br>2. 检查 kube-proxy<br>3. 检查 iptables/IPVS | 检查 Pod 就绪状态 |
| **Pod 间不通** | ping 失败 | 1. 检查 NetworkPolicy<br>2. 检查 CNI 插件<br>3. 检查节点路由 | 调整 NetworkPolicy |
| **外部访问不通** | Ingress 502/504 | 1. 检查 Ingress Controller<br>2. 检查后端 Service<br>3. 检查 ALB 配置 | 检查健康检查配置 |
| **跨节点不通** | 跨节点 Pod 不通 | 1. 检查 VXLAN/BGP<br>2. 检查安全组<br>3. 检查 VPC 路由 | 检查网络插件配置 |

**网络排查命令：**
```bash
# DNS 测试
kubectl run debug --image=busybox -it --rm -- nslookup kubernetes.default

# Service 连通性测试
kubectl run debug --image=busybox -it --rm -- wget -qO- http://<service>:<port>

# 检查 Endpoints
kubectl get endpoints <service-name>

# 检查 iptables 规则
kubectl exec -n kube-system <kube-proxy-pod> -- iptables -t nat -L KUBE-SERVICES

# 检查 IPVS 规则
kubectl exec -n kube-system <kube-proxy-pod> -- ipvsadm -Ln

# 抓包分析
kubectl exec -it <pod-name> -- tcpdump -i any -nn port 80
```

### 存储故障排查

| 问题 | 症状 | 排查步骤 | 解决方案 |
|------|------|----------|----------|
| **PVC Pending** | PVC 一直 Pending | 1. 检查 StorageClass<br>2. 检查 CSI 驱动<br>3. 检查配额 | 创建 StorageClass 或安装 CSI |
| **挂载失败** | Pod ContainerCreating | 1. 检查 CSI Node 日志<br>2. 检查卷状态<br>3. 检查节点权限 | 检查 IAM 权限 |
| **扩容失败** | resize 不生效 | 1. 检查 allowVolumeExpansion<br>2. 检查 CSI 支持<br>3. 检查文件系统 | 修改 StorageClass |
| **性能问题** | I/O 延迟高 | 1. 检查 IOPS 配置<br>2. 检查卷类型<br>3. 检查应用 I/O 模式 | 升级存储类型 |
| **跨 AZ 失败** | Pod 调度失败 | 1. 检查 PV 区域<br>2. 检查 volumeBindingMode | 使用 WaitForFirstConsumer |

**存储排查命令：**
```bash
# 检查 PV/PVC 状态
kubectl get pv,pvc

# 检查 PVC 详情
kubectl describe pvc <pvc-name>

# 检查 CSI 驱动
kubectl get pods -n kube-system -l app=ebs-csi-controller

# 检查 CSI 日志
kubectl logs -n kube-system -l app=ebs-csi-controller -c csi-provisioner

# 检查卷挂载
kubectl exec -it <pod-name> -- df -h
kubectl exec -it <pod-name> -- mount | grep <volume-path>
```

### 节点故障排查

| 问题 | 症状 | 排查步骤 | 解决方案 |
|------|------|----------|----------|
| **NotReady** | 节点状态 NotReady | 1. 检查 kubelet 状态<br>2. 检查节点资源<br>3. 检查网络连通性 | 重启 kubelet 或节点 |
| **资源压力** | MemoryPressure/DiskPressure | 1. 检查资源使用<br>2. 检查 Pod 分布<br>3. 检查 eviction 阈值 | 清理资源或扩容 |
| **Pod 驱逐** | Pod 被驱逐 | 1. 检查节点状态<br>2. 检查 PDB<br>3. 检查驱逐原因 | 调整 eviction 阈值 |
| **调度失败** | Pod 无法调度 | 1. 检查 Taint<br>2. 检查资源<br>3. 检查标签 | 调整调度约束 |

**节点排查命令：**
```bash
# 检查节点状态
kubectl get nodes -o wide
kubectl describe node <node-name>

# 检查节点资源
kubectl top nodes

# 检查节点上的 Pod
kubectl get pods -A -o wide --field-selector spec.nodeName=<node-name>

# 检查 kubelet 日志（SSH 到节点）
journalctl -u kubelet -f

# 检查节点条件
kubectl get node <node-name> -o jsonpath='{.status.conditions[*].type}'
```


### 控制平面故障排查

| 组件 | 问题 | 排查步骤 | 解决方案 |
|------|------|----------|----------|
| **API Server** | 响应慢/超时 | 1. 检查 API Server 日志<br>2. 检查 etcd 延迟<br>3. 检查请求量 | 增加副本或限流 |
| **etcd** | 延迟高 | 1. 检查磁盘 I/O<br>2. 检查网络延迟<br>3. 检查数据大小 | 使用 SSD 或压缩 |
| **Scheduler** | 调度延迟 | 1. 检查调度器日志<br>2. 检查节点数量<br>3. 检查调度约束 | 调整 percentageOfNodesToScore |
| **Controller** | 资源不同步 | 1. 检查控制器日志<br>2. 检查 Informer 状态<br>3. 检查 API Server 连接 | 重启控制器 |

**控制平面排查命令（EKS）：**
```bash
# 检查 EKS 集群状态
aws eks describe-cluster --name <cluster-name>

# 检查 API Server 健康
kubectl get --raw='/healthz?verbose'

# 检查组件状态
kubectl get componentstatuses  # 已弃用，但仍可用

# 检查 API Server 指标
kubectl get --raw='/metrics' | grep apiserver_request

# 检查 etcd 健康（自建集群）
etcdctl endpoint health
etcdctl endpoint status
```

---

## 附录：Kubernetes 性能调优指南
### 性能调优检查清单

| 层级 | 检查项 | 默认值 | 调优建议 | 影响 |
|------|--------|--------|----------|------|
| **etcd** | quota-backend-bytes | 2GB | 8GB | 存储容量 |
| | snapshot-count | 100000 | 50000 | 恢复速度 |
| | auto-compaction-retention | 0 | 1h | 存储空间 |
| **API Server** | max-requests-inflight | 400 | 800 | 并发读 |
| | max-mutating-requests-inflight | 200 | 400 | 并发写 |
| | watch-cache-sizes | 默认 | 增大热点 | Watch 性能 |
| **Controller** | kube-api-qps | 20 | 100 | API 请求 |
| | kube-api-burst | 30 | 200 | 突发请求 |
| | concurrent-*-syncs | 5 | 10-20 | 并发同步 |
| **Scheduler** | kube-api-qps | 50 | 100 | API 请求 |
| | percentageOfNodesToScore | 50% | 10-30% | 调度性能 |
| **kubelet** | max-pods | 110 | 250 | Pod 密度 |
| | kube-api-qps | 5 | 50 | API 请求 |
| | serialize-image-pulls | true | false | 镜像拉取 |
| **kube-proxy** | iptables-sync-period | 30s | 60s | 规则同步 |
| | mode | iptables | ipvs | 大规模 Service |

### 应用性能优化

| 优化项 | 说明 | 配置示例 |
|--------|------|----------|
| **资源配置** | 合理设置 requests/limits | requests: 100m/128Mi |
| **探针配置** | 避免过于频繁的探针 | periodSeconds: 10 |
| **镜像优化** | 使用小镜像，减少启动时间 | distroless/alpine |
| **预热** | 使用 startupProbe 预热 | failureThreshold: 30 |
| **连接池** | 复用连接，减少开销 | 应用配置 |
| **缓存** | 使用本地缓存减少网络调用 | Redis/Memcached |
| **异步处理** | 使用消息队列异步处理 | SQS/Kafka |

### 网络性能优化

| 优化项 | 说明 | 配置 |
|--------|------|------|
| **CNI 选择** | 选择高性能 CNI | Cilium eBPF |
| **kube-proxy 模式** | 大规模使用 IPVS | mode: ipvs |
| **MTU 配置** | 启用 Jumbo Frame | MTU: 9001 |
| **DNS 优化** | 减少 DNS 查询 | ndots: 2 |
| **Service 拓扑** | 优先本地流量 | topologyKeys |
| **连接复用** | 启用 HTTP/2 | 应用配置 |

---

## 附录：Kubernetes 最佳实践总结
### 设计原则

| 原则 | 说明 | 实践 |
|------|------|------|
| **无状态优先** | 应用设计为无状态 | 状态外部化到数据库/缓存 |
| **声明式配置** | 使用声明式 API | YAML 配置，GitOps |
| **不可变基础设施** | 不修改运行中的容器 | 重新部署而非修补 |
| **最小权限** | 只授予必要权限 | RBAC、PSA、NetworkPolicy |
| **故障隔离** | 隔离故障影响范围 | Namespace、PDB、多 AZ |
| **可观测性** | 全面的监控和日志 | Prometheus、Loki、Jaeger |
| **自动化** | 自动化运维操作 | GitOps、HPA、CA |

### 资源管理最佳实践

| 实践 | 说明 | 配置 |
|------|------|------|
| **设置 requests** | 保证资源预留 | 必须设置 |
| **设置 limits** | 防止资源滥用 | 建议设置 |
| **使用 LimitRange** | 设置默认值 | 命名空间级别 |
| **使用 ResourceQuota** | 限制总量 | 命名空间级别 |
| **使用 PDB** | 保证可用性 | 生产环境必须 |
| **使用 HPA** | 自动扩缩容 | 根据负载 |
| **使用 VPA** | 资源优化 | 推荐模式 |

### 安全最佳实践

| 实践 | 说明 | 配置 |
|------|------|------|
| **非 root 运行** | 降低权限 | runAsNonRoot: true |
| **只读文件系统** | 防止篡改 | readOnlyRootFilesystem: true |
| **禁用特权** | 防止逃逸 | privileged: false |
| **使用 PSA** | 强制安全策略 | restricted 级别 |
| **使用 NetworkPolicy** | 网络隔离 | 默认拒绝 |
| **外部 Secret 管理** | 安全存储 | Secrets Manager |
| **镜像扫描** | 漏洞检测 | CI/CD 集成 |
| **审计日志** | 安全审计 | API Server 审计 |

### 高可用最佳实践

| 实践 | 说明 | 配置 |
|------|------|------|
| **多副本部署** | 避免单点故障 | replicas >= 2 |
| **跨 AZ 分布** | 容灾 | topologySpreadConstraints |
| **反亲和** | 分散部署 | podAntiAffinity |
| **PDB 保护** | 升级保护 | minAvailable |
| **健康检查** | 自动恢复 | liveness/readiness |
| **优雅终止** | 无损下线 | terminationGracePeriod |
| **滚动更新** | 无停机更新 | maxSurge/maxUnavailable |


---

## 附录：Kubernetes 面试题精选
### 基础概念

| 问题 | 答案要点 |
|------|----------|
| **什么是 Pod？** | K8S 最小调度单元，包含一个或多个容器，共享网络和存储 |
| **Pod 和容器的区别？** | Pod 是逻辑主机，容器是进程；Pod 内容器共享 localhost |
| **什么是 Service？** | 服务发现和负载均衡抽象，提供稳定的访问入口 |
| **ClusterIP vs NodePort vs LoadBalancer？** | ClusterIP 集群内访问；NodePort 节点端口暴露；LoadBalancer 云负载均衡 |
| **什么是 Ingress？** | 七层流量入口，支持路径路由、TLS 终止 |
| **Deployment vs StatefulSet？** | Deployment 无状态；StatefulSet 有状态，有序部署，稳定网络标识 |
| **什么是 DaemonSet？** | 确保每个节点运行一个 Pod，用于日志、监控等 |
| **什么是 ConfigMap 和 Secret？** | ConfigMap 存储配置；Secret 存储敏感信息（Base64 编码） |
| **什么是 PV 和 PVC？** | PV 是存储资源；PVC 是存储请求，解耦存储和使用 |
| **什么是 Namespace？** | 资源隔离机制，逻辑分组 |

### 架构原理

| 问题 | 答案要点 |
|------|----------|
| **K8S 架构组件？** | 控制平面：API Server、etcd、Scheduler、Controller Manager；数据平面：kubelet、kube-proxy |
| **API Server 的作用？** | 集群入口，认证授权准入，etcd 唯一入口 |
| **etcd 的作用？** | 分布式 KV 存储，存储集群状态，Raft 共识 |
| **Scheduler 如何工作？** | 过滤（Filter）→ 评分（Score）→ 绑定（Bind） |
| **Controller 模式？** | Informer 监听 → Workqueue 入队 → Reconcile 调谐 |
| **kubelet 的作用？** | 节点代理，管理 Pod 生命周期，调用 CRI/CNI/CSI |
| **kube-proxy 的作用？** | Service 实现，iptables/IPVS 规则管理 |
| **什么是 CRI/CNI/CSI？** | 容器运行时/网络/存储接口标准 |
| **什么是 Informer？** | List+Watch 机制，本地缓存，事件通知 |
| **什么是 Admission Controller？** | 准入控制，Mutating 修改请求，Validating 验证请求 |

### 网络相关

| 问题 | 答案要点 |
|------|----------|
| **K8S 网络模型？** | 每个 Pod 有独立 IP，Pod 间可直接通信，无 NAT |
| **Service 如何实现？** | kube-proxy 通过 iptables/IPVS 实现 DNAT |
| **iptables vs IPVS？** | iptables O(n) 链表；IPVS O(1) 哈希表，大规模更优 |
| **什么是 CNI？** | 容器网络接口，定义网络插件标准 |
| **Calico vs Cilium？** | Calico BGP/VXLAN；Cilium eBPF，性能更优 |
| **什么是 NetworkPolicy？** | 网络策略，控制 Pod 间流量 |
| **DNS 如何工作？** | CoreDNS 提供服务发现，解析 Service 名称 |
| **什么是 Headless Service？** | ClusterIP=None，直接返回 Pod IP |
| **Ingress 如何工作？** | Ingress Controller 监听 Ingress 资源，配置负载均衡 |
| **什么是 Service Mesh？** | 服务间通信基础设施，Sidecar 代理 |

### 存储相关

| 问题 | 答案要点 |
|------|----------|
| **什么是 CSI？** | 容器存储接口，定义存储插件标准 |
| **PV 的访问模式？** | RWO（单节点读写）、ROX（多节点只读）、RWX（多节点读写） |
| **什么是 StorageClass？** | 动态供应存储，定义存储类型和参数 |
| **什么是 volumeBindingMode？** | Immediate 立即绑定；WaitForFirstConsumer 延迟绑定 |
| **什么是 reclaimPolicy？** | Delete 删除 PVC 时删除 PV；Retain 保留 |
| **StatefulSet 如何使用存储？** | volumeClaimTemplates 为每个 Pod 创建独立 PVC |
| **什么是 emptyDir？** | 临时存储，Pod 删除时清除 |
| **什么是 hostPath？** | 挂载节点目录，有安全风险 |
| **EBS vs EFS？** | EBS 块存储 RWO；EFS 文件存储 RWX |
| **如何扩容 PVC？** | 修改 PVC 大小，需要 allowVolumeExpansion=true |


### 调度相关

| 问题 | 答案要点 |
|------|----------|
| **调度流程？** | 过滤（Filter）→ 评分（Score）→ 绑定（Bind） |
| **nodeSelector vs nodeAffinity？** | nodeSelector 简单匹配；nodeAffinity 支持复杂表达式和软约束 |
| **什么是 Taint 和 Toleration？** | Taint 标记节点；Toleration 允许 Pod 调度到有 Taint 的节点 |
| **什么是 podAffinity？** | Pod 亲和，将 Pod 调度到有特定 Pod 的节点 |
| **什么是 topologySpreadConstraints？** | 拓扑分布约束，确保 Pod 跨拓扑域均匀分布 |
| **什么是抢占（Preemption）？** | 高优先级 Pod 驱逐低优先级 Pod |
| **什么是 PriorityClass？** | 定义 Pod 优先级 |
| **调度失败怎么排查？** | kubectl describe pod 查看 Events |
| **如何实现 bin-packing？** | 使用 MostAllocated 评分策略 |
| **什么是 Descheduler？** | 重新平衡 Pod 分布 |

### 安全相关

| 问题 | 答案要点 |
|------|----------|
| **什么是 RBAC？** | 基于角色的访问控制，Role/ClusterRole + Binding |
| **什么是 ServiceAccount？** | Pod 身份，用于 API 认证 |
| **什么是 PSA？** | Pod 安全准入，替代 PSP，三级策略 |
| **如何限制容器权限？** | securityContext：runAsNonRoot、readOnlyRootFilesystem |
| **什么是 NetworkPolicy？** | 网络策略，控制 Pod 间流量 |
| **Secret 如何加密？** | etcd 加密，KMS 集成 |
| **什么是 Seccomp？** | 系统调用过滤 |
| **什么是 AppArmor？** | 强制访问控制 |
| **如何扫描镜像漏洞？** | Trivy、Clair、ECR 扫描 |
| **什么是 OPA/Gatekeeper？** | 策略引擎，自定义准入策略 |

### 运维相关

| 问题 | 答案要点 |
|------|----------|
| **什么是 HPA？** | 水平 Pod 自动扩缩容，基于指标 |
| **什么是 VPA？** | 垂直 Pod 自动扩缩容，调整资源配置 |
| **什么是 Cluster Autoscaler？** | 集群自动扩缩容，调整节点数量 |
| **什么是 Karpenter？** | 新一代节点自动供应，更快更智能 |
| **滚动更新如何工作？** | 逐步替换 Pod，maxSurge/maxUnavailable 控制 |
| **什么是蓝绿部署？** | 两套环境，切换 Service selector |
| **什么是灰度发布？** | 按比例分流，逐步验证 |
| **什么是 PDB？** | Pod 中断预算，保证最小可用副本 |
| **如何排查 Pod 问题？** | describe、logs、events、exec |
| **如何升级集群？** | 控制平面 → 数据平面 → 插件 |

### 高级话题

| 问题 | 答案要点 |
|------|----------|
| **什么是 Operator？** | 自定义控制器，管理复杂应用 |
| **什么是 CRD？** | 自定义资源定义，扩展 K8S API |
| **什么是 Helm？** | K8S 包管理器，模板化部署 |
| **什么是 Kustomize？** | 配置管理，overlay 模式 |
| **什么是 GitOps？** | Git 作为唯一真实来源，自动同步 |
| **什么是 ArgoCD？** | GitOps 工具，声明式 CD |
| **什么是 Service Mesh？** | 服务间通信基础设施 |
| **什么是 Istio？** | Service Mesh 实现，Envoy Sidecar |
| **什么是 eBPF？** | 内核可编程，高性能网络 |
| **什么是 Cilium？** | eBPF 网络插件，高性能 |

---

## 附录：云原生生态系
### AWS 容器服务对比

| 服务 | 类型 | 控制平面 | 数据平面 | 适用场景 |
|------|------|----------|----------|----------|
| **EKS** | K8S 托管 | AWS 托管 | EC2/Fargate | 复杂微服务 |
| **ECS** | AWS 原生 | AWS 托管 | EC2/Fargate | 简单容器化 |
| **Fargate** | Serverless | AWS 托管 | AWS 托管 | 无需管理节点 |
| **App Runner** | PaaS | AWS 托管 | AWS 托管 | 快速部署 |
| **Lambda** | FaaS | AWS 托管 | AWS 托管 | 事件驱动 |

### 容器编排选型指南

| 场景 | 推荐方案 | 理由 |
|------|----------|------|
| **AWS 原生，简单应用** | ECS + Fargate | 简单、低成本 |
| **AWS 原生，复杂微服务** | EKS | K8S 生态、灵活 |
| **多云/混合云** | Kubernetes | 跨云可移植 |
| **边缘计算** | K3s | 轻量级 |
| **开发测试** | minikube/kind | 本地环境 |
| **大数据** | K8S + Spark Operator | 统一平台 |
| **AI/ML** | K8S + Kubeflow | GPU 调度 |


---

## 附录：Kubernetes API 参考
### 核心 API 组

| API 组 | 版本 | 资源 | 说明 |
|--------|------|------|------|
| **core (v1)** | v1 | Pod, Service, ConfigMap, Secret, PV, PVC, Namespace, Node | 核心资源 |
| **apps** | v1 | Deployment, StatefulSet, DaemonSet, ReplicaSet | 工作负载 |
| **batch** | v1 | Job, CronJob | 批处理 |
| **networking.k8s.io** | v1 | Ingress, NetworkPolicy, IngressClass | 网络 |
| **storage.k8s.io** | v1 | StorageClass, VolumeAttachment, CSIDriver | 存储 |
| **rbac.authorization.k8s.io** | v1 | Role, ClusterRole, RoleBinding, ClusterRoleBinding | RBAC |
| **autoscaling** | v2 | HorizontalPodAutoscaler | 自动扩缩 |
| **policy** | v1 | PodDisruptionBudget | 策略 |
| **admissionregistration.k8s.io** | v1 | MutatingWebhookConfiguration, ValidatingWebhookConfiguration | 准入控制 |
| **apiextensions.k8s.io** | v1 | CustomResourceDefinition | CRD |

### 常用 API 操作

| 操作 | HTTP 方法 | 说明 |
|------|-----------|------|
| **Create** | POST | 创建资源 |
| **Get** | GET | 获取单个资源 |
| **List** | GET | 获取资源列表 |
| **Watch** | GET (streaming) | 监听资源变更 |
| **Update** | PUT | 更新整个资源 |
| **Patch** | PATCH | 部分更新资源 |
| **Delete** | DELETE | 删除资源 |

### API 版本演进

| 版本 | 稳定性 | 说明 |
|------|--------|------|
| **v1** | 稳定 | 生产就绪，不会删除 |
| **v1beta1** | Beta | 功能完整，可能有变更 |
| **v1alpha1** | Alpha | 实验性，可能删除 |

### kubectl 输出格式

| 格式 | 说明 | 示例 |
|------|------|------|
| **-o wide** | 扩展输出 | `kubectl get pods -o wide` |
| **-o yaml** | YAML 格式 | `kubectl get pod <name> -o yaml` |
| **-o json** | JSON 格式 | `kubectl get pod <name> -o json` |
| **-o jsonpath** | JSONPath 提取 | `kubectl get pod <name> -o jsonpath='{.status.phase}'` |
| **-o custom-columns** | 自定义列 | `kubectl get pods -o custom-columns=NAME:.metadata.name,STATUS:.status.phase` |
| **-o name** | 仅名称 | `kubectl get pods -o name` |

### JSONPath 常用表达式

```bash
# 获取所有 Pod 名称
kubectl get pods -o jsonpath='{.items[*].metadata.name}'

# 获取所有 Pod IP
kubectl get pods -o jsonpath='{.items[*].status.podIP}'

# 获取特定标签的 Pod
kubectl get pods -o jsonpath='{.items[?(@.metadata.labels.app=="nginx")].metadata.name}'

# 获取所有容器镜像
kubectl get pods -o jsonpath='{.items[*].spec.containers[*].image}'

# 获取节点 IP
kubectl get nodes -o jsonpath='{.items[*].status.addresses[?(@.type=="InternalIP")].address}'

# 格式化输出
kubectl get pods -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.phase}{"\n"}{end}'
```

---

## 附录：Helm 使用指南
### Helm 基本概念

| 概念 | 说明 |
|------|------|
| **Chart** | Helm 包，包含 K8S 资源模板 |
| **Release** | Chart 的一次部署实例 |
| **Repository** | Chart 仓库 |
| **Values** | Chart 配置参数 |
| **Template** | K8S 资源模板 |

### Helm 常用命令

```bash
# ===== 仓库管理 =====
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update
helm repo list
helm search repo nginx

# ===== Chart 管理 =====
helm show chart bitnami/nginx
helm show values bitnami/nginx
helm pull bitnami/nginx

# ===== 安装/升级 =====
helm install my-nginx bitnami/nginx
helm install my-nginx bitnami/nginx -f values.yaml
helm install my-nginx bitnami/nginx --set replicaCount=3
helm upgrade my-nginx bitnami/nginx
helm upgrade --install my-nginx bitnami/nginx

# ===== 查看/管理 =====
helm list
helm status my-nginx
helm history my-nginx
helm get values my-nginx
helm get manifest my-nginx

# ===== 回滚/删除 =====
helm rollback my-nginx 1
helm uninstall my-nginx

# ===== 调试 =====
helm template my-nginx bitnami/nginx
helm install my-nginx bitnami/nginx --dry-run --debug
helm lint ./my-chart
```

### Chart 目录结构

```
my-chart/
├── Chart.yaml          # Chart 元数据
├── values.yaml         # 默认配置值
├── charts/             # 依赖 Chart
├── templates/          # K8S 资源模板
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── ingress.yaml
│   ├── configmap.yaml
│   ├── secret.yaml
│   ├── _helpers.tpl    # 模板辅助函数
│   └── NOTES.txt       # 安装说明
└── .helmignore         # 忽略文件
```

### Helm 模板语法

```yaml
# values.yaml
replicaCount: 3
image:
  repository: nginx
  tag: "1.21"
  pullPolicy: IfNotPresent

# templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "my-chart.fullname" . }}
  labels:
    {{- include "my-chart.labels" . | nindent 4 }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      {{- include "my-chart.selectorLabels" . | nindent 6 }}
  template:
    metadata:
      labels:
        {{- include "my-chart.selectorLabels" . | nindent 8 }}
    spec:
      containers:
      - name: {{ .Chart.Name }}
        image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
        imagePullPolicy: {{ .Values.image.pullPolicy }}
        {{- if .Values.resources }}
        resources:
          {{- toYaml .Values.resources | nindent 10 }}
        {{- end }}
```

---

## 附录：Kustomize 使用指南
### Kustomize 基本概念

| 概念 | 说明 |
|------|------|
| **Base** | 基础配置 |
| **Overlay** | 环境特定配置 |
| **Patch** | 配置补丁 |
| **Generator** | 生成 ConfigMap/Secret |
| **Transformer** | 转换资源 |

### Kustomize 目录结构

```
├── base/
│   ├── kustomization.yaml
│   ├── deployment.yaml
│   ├── service.yaml
│   └── configmap.yaml
└── overlays/
    ├── dev/
    │   ├── kustomization.yaml
    │   └── patch.yaml
    ├── staging/
    │   ├── kustomization.yaml
    │   └── patch.yaml
    └── prod/
        ├── kustomization.yaml
        └── patch.yaml
```

### kustomization.yaml 示例

```yaml
# base/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
- deployment.yaml
- service.yaml
- configmap.yaml

commonLabels:
  app: myapp

# overlays/prod/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
- ../../base

namespace: production

namePrefix: prod-

commonLabels:
  environment: production

replicas:
- name: myapp
  count: 5

images:
- name: myapp
  newTag: v2.0.0

patches:
- path: patch.yaml
  target:
    kind: Deployment
    name: myapp

configMapGenerator:
- name: myapp-config
  literals:
  - LOG_LEVEL=info
  - ENV=production

secretGenerator:
- name: myapp-secret
  literals:
  - DB_PASSWORD=secret123
```

### Kustomize 常用命令

```bash
# 构建配置
kubectl kustomize overlays/prod

# 应用配置
kubectl apply -k overlays/prod

# 查看差异
kubectl diff -k overlays/prod

# 删除配置
kubectl delete -k overlays/prod
```


---

## 附录：Kubernetes 日志管理
### 日志架构

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Kubernetes 日志架构                               │
├─────────────────────────────────────────────────────────────────────┤
│  应用日志                                                            │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │  容器 stdout/stderr → /var/log/containers/*.log              │    │
│  │                    → /var/log/pods/<namespace>_<pod>_<uid>/ │    │
│  └─────────────────────────────────────────────────────────────┘    │
│                              │                                      │
│                              ▼                                      │
│  日志采集                                                            │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │  DaemonSet 部署日志采集器                                     │    │
│  │  ├─ Fluent Bit（轻量级，推荐）                                │    │
│  │  ├─ Fluentd（功能丰富）                                       │    │
│  │  └─ Filebeat（Elastic 生态）                                 │    │
│  └─────────────────────────────────────────────────────────────┘    │
│                              │                                      │
│                              ▼                                      │
│  日志存储                                                            │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │  ├─ Elasticsearch（EFK 栈）                                  │    │
│  │  ├─ Loki（Grafana 生态，轻量级）                              │    │
│  │  ├─ CloudWatch Logs（AWS 原生）                              │    │
│  │  └─ S3（长期存储）                                            │    │
│  └─────────────────────────────────────────────────────────────┘    │
│                              │                                      │
│                              ▼                                      │
│  日志查询                                                            │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │  ├─ Kibana（Elasticsearch）                                  │    │
│  │  ├─ Grafana（Loki）                                          │    │
│  │  └─ CloudWatch Logs Insights（AWS）                          │    │
│  └─────────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────────┘
```

### 日志方案对比

| 方案 | 采集器 | 存储 | 查询 | 成本 | 适用场景 |
|------|--------|------|------|------|----------|
| **EFK** | Fluentd | Elasticsearch | Kibana | 高 | 大规模、复杂查询 |
| **PLG** | Promtail | Loki | Grafana | 低 | 中小规模、成本敏感 |
| **CloudWatch** | Fluent Bit | CloudWatch | Insights | 中 | AWS 原生 |
| **Datadog** | Agent | Datadog | Datadog | 高 | 企业级 |

### Fluent Bit 配置示例

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: fluent-bit-config
  namespace: logging
data:
  fluent-bit.conf: |
    [SERVICE]
        Flush         1
        Log_Level     info
        Daemon        off
        Parsers_File  parsers.conf

    [INPUT]
        Name              tail
        Tag               kube.*
        Path              /var/log/containers/*.log
        Parser            docker
        DB                /var/log/flb_kube.db
        Mem_Buf_Limit     5MB
        Skip_Long_Lines   On
        Refresh_Interval  10

    [FILTER]
        Name                kubernetes
        Match               kube.*
        Kube_URL            https://kubernetes.default.svc:443
        Kube_CA_File        /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
        Kube_Token_File     /var/run/secrets/kubernetes.io/serviceaccount/token
        Kube_Tag_Prefix     kube.var.log.containers.
        Merge_Log           On
        Keep_Log            Off
        K8S-Logging.Parser  On
        K8S-Logging.Exclude On

    [OUTPUT]
        Name            cloudwatch_logs
        Match           *
        region          us-east-1
        log_group_name  /aws/eks/cluster-name/containers
        log_stream_prefix from-fluent-bit-
        auto_create_group true
```

### 日志最佳实践

| 实践 | 说明 |
|------|------|
| **结构化日志** | 使用 JSON 格式，便于解析 |
| **日志级别** | 合理使用 DEBUG/INFO/WARN/ERROR |
| **上下文信息** | 包含 traceId、requestId |
| **避免敏感信息** | 不记录密码、Token |
| **日志轮转** | 配置日志大小和保留策略 |
| **集中存储** | 统一日志存储和查询 |
| **告警集成** | 基于日志触发告警 |

---

## 附录：Kubernetes 备份与恢复
### 备份策略

| 备份对象 | 工具 | 频率 | 说明 |
|----------|------|------|------|
| **etcd** | etcdctl snapshot | 每小时 | 集群状态 |
| **K8S 资源** | Velero | 每天 | 资源定义 |
| **PV 数据** | Velero + CSI | 每天 | 持久化数据 |
| **配置** | Git | 实时 | GitOps |

### etcd 备份

```bash
# 创建快照
ETCDCTL_API=3 etcdctl snapshot save /backup/etcd-snapshot.db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key

# 验证快照
ETCDCTL_API=3 etcdctl snapshot status /backup/etcd-snapshot.db

# 恢复快照
ETCDCTL_API=3 etcdctl snapshot restore /backup/etcd-snapshot.db \
  --data-dir=/var/lib/etcd-restore
```

### Velero 备份

```bash
# 安装 Velero
velero install \
  --provider aws \
  --plugins velero/velero-plugin-for-aws:v1.7.0 \
  --bucket velero-backup \
  --backup-location-config region=us-east-1 \
  --snapshot-location-config region=us-east-1 \
  --secret-file ./credentials-velero

# 创建备份
velero backup create my-backup --include-namespaces production

# 定时备份
velero schedule create daily-backup --schedule="0 2 * * *" --include-namespaces production

# 查看备份
velero backup get
velero backup describe my-backup

# 恢复备份
velero restore create --from-backup my-backup

# 查看恢复
velero restore get
velero restore describe my-restore
```

### 灾难恢复流程

```
┌─────────────────────────────────────────────────────────────────────┐
│                    灾难恢复流程                                       │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  1. 评估灾难范围                                                      │
│     ├─ 单个 Pod/Deployment？→ 重新部署                                │
│     ├─ 单个节点？→ 驱逐 Pod，修复/替换节点                              │
│     ├─ 控制平面？→ 恢复 etcd                                          │
│     └─ 整个集群？→ 重建集群 + 恢复数据                                  │
│                              │                                      │
│                              ▼                                      │
│  2. 恢复控制平面                                                      │
│     ├─ 恢复 etcd 快照                                                │
│     ├─ 启动 API Server                                              │
│     ├─ 启动 Controller Manager                                      │
│     └─ 启动 Scheduler                                               │
│                              │                                      │
│                              ▼                                      │
│  3. 恢复数据平面                                                      │
│     ├─ 加入 Worker 节点                                              │
│     ├─ 恢复 CNI 插件                                                 │
│     └─ 恢复 CSI 驱动                                                 │
│                              │                                      │
│                              ▼                                      │
│  4. 恢复应用                                                         │
│     ├─ 使用 Velero 恢复资源                                           │
│     ├─ 恢复 PV 数据                                                  │
│     └─ 验证应用状态                                                   │
│                              │                                      │
│                              ▼                                      │
│  5. 验证恢复                                                         │
│     ├─ 检查所有 Pod 状态                                              │
│     ├─ 检查 Service 连通性                                            │
│     ├─ 运行 E2E 测试                                                 │
│     └─ 通知相关团队                                                   │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 附录：Kubernetes 多集群管理
### 多集群架构

```
┌─────────────────────────────────────────────────────────────────────┐
│                    多集群架构模式                                     │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  模式 1：独立集群                                                     │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐                           │
│  │ Cluster A│  │ Cluster B│  │ Cluster C│                           │
│  │  (Dev)   │  │ (Staging)│  │  (Prod)  │                           │
│  └──────────┘  └──────────┘  └──────────┘                           │
│  • 完全隔离                                                          │
│  • 独立管理                                                          │
│  • 适合环境隔离                                                       │
│                                                                     │
│  模式 2：联邦集群                                                     │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │                    Federation Control Plane                 │    │
│  └─────────────────────────────────────────────────────────────┘    │
│       │              │              │                               │
│       ▼              ▼              ▼                               │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐                           │
│  │ Cluster A│  │ Cluster B│  │ Cluster C│                           │
│  │ (Region1)│  │ (Region2)│  │ (Region3)│                           │
│  └──────────┘  └──────────┘  └──────────┘                           │
│  • 统一管理                                                          │
│  • 跨集群调度                                                        │
│  • 适合多区域部署                                                     │
│                                                                     │
│  模式 3：Hub-Spoke                                                   │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │                    Hub Cluster (管理集群)                    │    │
│  │                    ArgoCD / Rancher / ACM                   │    │
│  └─────────────────────────────────────────────────────────────┘    │
│       │              │              │                               │
│       ▼              ▼              ▼                               │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐                           │
│  │ Spoke 1  │  │ Spoke 2  │  │ Spoke 3  │                           │
│  │(工作集群) │  │(工作集群)  │  │(工作集群) │                           │
│  └──────────┘  └──────────┘  └──────────┘                           │
│  • 集中管理                                                          │
│  • GitOps 部署                                                       │
│  • 适合大规模管理                                                     │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### 多集群工具对比

| 工具 | 类型 | 特点 | 适用场景 |
|------|------|------|----------|
| **ArgoCD** | GitOps | 声明式部署，多集群支持 | GitOps 部署 |
| **Rancher** | 管理平台 | 统一 UI，多集群管理 | 企业级管理 |
| **Cluster API** | 集群生命周期 | 声明式集群管理 | 集群自动化 |
| **Liqo** | 多集群网络 | 跨集群 Pod 调度 | 资源共享 |
| **Submariner** | 多集群网络 | 跨集群网络连接 | 跨集群通信 |
| **Karmada** | 联邦调度 | 跨集群工作负载调度 | 多集群调度 |

### kubeconfig 多集群配置

```yaml
apiVersion: v1
kind: Config
clusters:
- cluster:
    certificate-authority-data: <CA_DATA>
    server: https://cluster-a.example.com
  name: cluster-a
- cluster:
    certificate-authority-data: <CA_DATA>
    server: https://cluster-b.example.com
  name: cluster-b
contexts:
- context:
    cluster: cluster-a
    user: admin-a
  name: cluster-a-context
- context:
    cluster: cluster-b
    user: admin-b
  name: cluster-b-context
current-context: cluster-a-context
users:
- name: admin-a
  user:
    token: <TOKEN_A>
- name: admin-b
  user:
    token: <TOKEN_B>
```

```bash
# 切换集群
kubectl config use-context cluster-b-context

# 指定集群执行命令
kubectl --context=cluster-a-context get pods
kubectl --context=cluster-b-context get pods

# 查看所有上下文
kubectl config get-contexts
```


---

## 附录：Kubernetes Operator 开发指南
### Operator 模式概述

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Operator 模式架构                                 │
├─────────────────────────────────────────────────────────────────────┤
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │                    Custom Resource (CR)                     │    │
│  │                    用户声明期望状态                            │    │
│  └─────────────────────────────────────────────────────────────┘    │
│                              │                                      │
│                              ▼                                      │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │                    Operator Controller                      │    │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐          │    │
│  │  │   Watch     │  │  Reconcile  │  │   Update    │          │    │
│  │  │   CR 变化    │──│  调谐逻辑    │──│   状态       │          │    │
│  │  └─────────────┘  └─────────────┘  └─────────────┘          │    │
│  └─────────────────────────────────────────────────────────────┘    │
│                              │                                      │
│                              ▼                                      │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │                    Managed Resources                        │    │
│  │  Deployment │ Service │ ConfigMap │ Secret │ PVC            │    │
│  └─────────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────────┘
```


### Operator 开发框架对比

| 框架 | 语言 | 特点 | 学习曲线 | 适用场景 |
|------|------|------|----------|----------|
| **Kubebuilder** | Go | 官方推荐，功能完整 | 中等 | 生产级 Operator |
| **Operator SDK** | Go/Ansible/Helm | 多语言支持，集成 OLM | 中等 | 企业级 Operator |
| **Kopf** | Python | 简单易用，快速开发 | 低 | 快速原型 |
| **KUDO** | YAML | 声明式，无需编码 | 低 | 简单有状态应用 |
| **Metacontroller** | 任意语言 | Webhook 模式 | 低 | 多语言团队 |
| **shell-operator** | Shell/Python | 脚本化开发 | 低 | 运维自动化 |

### Kubebuilder 项目结构

```
my-operator/
├── api/
│   └── v1/
│       ├── myapp_types.go      # CRD 类型定义
│       ├── groupversion_info.go
│       └── zz_generated.deepcopy.go
├── config/
│   ├── crd/                    # CRD YAML
│   ├── manager/                # Deployment YAML
│   ├── rbac/                   # RBAC 配置
│   └── samples/                # CR 示例
├── controllers/
│   └── myapp_controller.go     # 控制器逻辑
├── main.go                     # 入口文件
├── Dockerfile
├── Makefile
└── go.mod
```


### CRD 类型定义示例

```go
// api/v1/myapp_types.go
package v1

import (
    metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
)

// MyAppSpec 定义期望状态
type MyAppSpec struct {
    // 副本数
    // +kubebuilder:validation:Minimum=1
    // +kubebuilder:validation:Maximum=10
    Replicas int32 `json:"replicas"`
    
    // 镜像
    // +kubebuilder:validation:Required
    Image string `json:"image"`
    
    // 资源配置
    // +optional
    Resources ResourceSpec `json:"resources,omitempty"`
}

// MyAppStatus 定义观察到的状态
type MyAppStatus struct {
    // 当前副本数
    ReadyReplicas int32 `json:"readyReplicas"`
    
    // 条件列表
    Conditions []metav1.Condition `json:"conditions,omitempty"`
    
    // 阶段
    // +kubebuilder:validation:Enum=Pending;Running;Failed
    Phase string `json:"phase,omitempty"`
}

// +kubebuilder:object:root=true
// +kubebuilder:subresource:status
// +kubebuilder:printcolumn:name="Replicas",type=integer,JSONPath=`.spec.replicas`
// +kubebuilder:printcolumn:name="Ready",type=integer,JSONPath=`.status.readyReplicas`
// +kubebuilder:printcolumn:name="Phase",type=string,JSONPath=`.status.phase`
type MyApp struct {
    metav1.TypeMeta   `json:",inline"`
    metav1.ObjectMeta `json:"metadata,omitempty"`
    Spec   MyAppSpec   `json:"spec,omitempty"`
    Status MyAppStatus `json:"status,omitempty"`
}
```


### Reconcile 控制器逻辑示例

```go
// controllers/myapp_controller.go
func (r *MyAppReconciler) Reconcile(ctx context.Context, 
    req ctrl.Request) (ctrl.Result, error) {
    
    log := log.FromContext(ctx)
    
    // 1. 获取 CR
    myapp := &myappv1.MyApp{}
    if err := r.Get(ctx, req.NamespacedName, myapp); err != nil {
        if errors.IsNotFound(err) {
            return ctrl.Result{}, nil
        }
        return ctrl.Result{}, err
    }
    
    // 2. 创建或更新 Deployment
    deployment := r.deploymentForMyApp(myapp)
    if err := controllerutil.SetControllerReference(
        myapp, deployment, r.Scheme); err != nil {
        return ctrl.Result{}, err
    }
    
    found := &appsv1.Deployment{}
    err := r.Get(ctx, types.NamespacedName{
        Name: deployment.Name, Namespace: deployment.Namespace}, found)
    
    if err != nil && errors.IsNotFound(err) {
        log.Info("Creating Deployment", "name", deployment.Name)
        if err = r.Create(ctx, deployment); err != nil {
            return ctrl.Result{}, err
        }
        return ctrl.Result{Requeue: true}, nil
    }
    
    // 3. 更新状态
    myapp.Status.ReadyReplicas = found.Status.ReadyReplicas
    myapp.Status.Phase = "Running"
    if err := r.Status().Update(ctx, myapp); err != nil {
        return ctrl.Result{}, err
    }
    
    return ctrl.Result{RequeueAfter: time.Minute}, nil
}
```


### Operator 开发最佳实践

| 实践 | 说明 | 示例 |
|------|------|------|
| **幂等性** | Reconcile 必须幂等 | 使用 CreateOrUpdate |
| **Owner Reference** | 设置资源所有者 | SetControllerReference |
| **Finalizer** | 清理外部资源 | 添加 finalizer 字段 |
| **Status Subresource** | 分离 spec 和 status | +kubebuilder:subresource:status |
| **Validation** | 使用 webhook 验证 | ValidatingWebhook |
| **Rate Limiting** | 避免 API 过载 | 使用 workqueue |
| **Metrics** | 暴露 Prometheus 指标 | controller-runtime metrics |
| **Leader Election** | 高可用部署 | 启用 leader election |

### Operator 问题定位表

| 问题 | 原因 | 解决方案 |
|------|------|----------|
| CR 创建后无响应 | Controller 未启动 | 检查 Deployment 日志 |
| Reconcile 循环 | 状态更新触发 Watch | 使用 Generation 判断 |
| 资源未清理 | 缺少 Owner Reference | 设置 SetControllerReference |
| 权限不足 | RBAC 配置错误 | 检查 ClusterRole |
| Webhook 超时 | 证书配置错误 | 检查 cert-manager |
| 内存泄漏 | Informer 缓存过大 | 配置 LabelSelector |

---

## 附录：GitOps 实践指南
### GitOps 架构


```
┌─────────────────────────────────────────────────────────────────────┐
│                    GitOps 工作流架构                                 │
├─────────────────────────────────────────────────────────────────────┤
│  ┌──────────────┐    Push     ┌──────────────┐                      │
│  │   Developer  │────────────▶│  Git Repo    │                      │
│  │   (代码变更)  │             │  (单一真相源) │                       │
│  └──────────────┘             └──────────────┘                      │
│                                      │                              │
│                                      │ Watch/Poll                   │
│                                      ▼                              │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │                    GitOps Controller                        │    │
│  │              (ArgoCD / Flux / Jenkins X)                    │    │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐          │    │
│  │  │   Detect    │  │   Compare   │  │   Sync      │          │    │
│  │  │   变更检测   │──│   差异对比    │──│   同步部署   │          │    │
│  │  └─────────────┘  └─────────────┘  └─────────────┘          │    │
│  └─────────────────────────────────────────────────────────────┘    │
│                              │                                      │
│                              ▼                                      │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │                    Kubernetes Cluster                       │    │
│  │              (实际运行状态 = Git 期望状态)                     │    │
│  └─────────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────────┘
```

### GitOps 工具对比

| 工具 | 架构 | 特点 | 多集群 | UI | 适用场景 |
|------|------|------|--------|-----|----------|
| **ArgoCD** | Pull | 声明式，强大 UI | ✅ | ✅ 优秀 | 企业级 |
| **Flux v2** | Pull | 轻量，GitOps Toolkit | ✅ | ⚠️ 基础 | 云原生 |
| **Jenkins X** | Push/Pull | CI/CD 集成 | ✅ | ✅ | DevOps |
| **Rancher Fleet** | Pull | 大规模集群 | ✅ | ✅ | 边缘计算 |
| **Weave GitOps** | Pull | Flux 增强 | ✅ | ✅ | 企业级 |


### ArgoCD 核心概念

```yaml
# ArgoCD Application 示例
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: my-app
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/org/repo.git
    targetRevision: HEAD
    path: k8s/overlays/production
  destination:
    server: https://kubernetes.default.svc
    namespace: production
  syncPolicy:
    automated:
      prune: true           # 删除 Git 中不存在的资源
      selfHeal: true        # 自动修复漂移
    syncOptions:
      - CreateNamespace=true
    retry:
      limit: 5
      backoff:
        duration: 5s
        factor: 2
        maxDuration: 3m
```

### Git 仓库结构最佳实践

```
gitops-repo/
├── apps/                       # 应用定义
│   ├── base/                   # 基础配置
│   │   ├── deployment.yaml
│   │   ├── service.yaml
│   │   └── kustomization.yaml
│   └── overlays/               # 环境覆盖
│       ├── dev/
│       ├── staging/
│       └── production/
├── infrastructure/             # 基础设施
│   ├── cert-manager/
│   ├── ingress-nginx/
│   └── monitoring/
├── clusters/                   # 集群配置
│   ├── dev-cluster/
│   ├── staging-cluster/
│   └── prod-cluster/
└── projects/                   # ArgoCD Projects
    └── apps.yaml
```


### GitOps 最佳实践

| 实践 | 说明 | 示例 |
|------|------|------|
| **单一真相源** | Git 是唯一配置来源 | 禁止 kubectl apply |
| **声明式配置** | 使用 YAML 描述期望状态 | Kustomize/Helm |
| **环境分离** | 不同环境使用不同分支/目录 | overlays/prod |
| **密钥管理** | 不在 Git 存储明文密钥 | Sealed Secrets |
| **变更审计** | 所有变更通过 PR | Git history |
| **自动同步** | 启用自动同步和自愈 | syncPolicy.automated |
| **渐进式交付** | 结合 Argo Rollouts | Canary/BlueGreen |

### GitOps 问题定位表

| 问题 | 原因 | 解决方案 |
|------|------|----------|
| 同步失败 | YAML 语法错误 | 检查 Git 提交 |
| 资源漂移 | 手动修改集群 | 启用 selfHeal |
| 权限不足 | ServiceAccount 权限 | 检查 RBAC |
| 密钥泄露 | 明文存储 Secret | 使用 Sealed Secrets |
| 同步慢 | 仓库过大 | 拆分仓库 |
| Webhook 失败 | 网络不通 | 检查防火墙 |

---

## 附录：容器化成本优化深度指南
### 成本优化架构


```
┌─────────────────────────────────────────────────────────────────────┐
│                    容器化成本优化层级                                  │
├─────────────────────────────────────────────────────────────────────┤
│  Layer 1: 基础设施层                                                 │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │  • 实例类型选择 (Graviton/Spot/Reserved)                      │    │
│  │  • 节点自动伸缩 (Karpenter/CA)                                │    │
│  │  • 多 AZ 优化                                                │    │
│  └─────────────────────────────────────────────────────────────┘    │
│                              │                                      │
│  Layer 2: 集群层                                                    │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │  • 资源配额管理 (ResourceQuota)                               │    │
│  │  • 命名空间隔离                                               │    │
│  │  • 节点池优化                                                 │    │
│  └─────────────────────────────────────────────────────────────┘    │
│                              │                                      │
│  Layer 3: 工作负载层                                                  │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │  • 资源请求/限制优化                                           │    │
│  │  • HPA/VPA 自动伸缩                                          │    │
│  │  • Pod 优先级和抢占                                           │    │
│  └─────────────────────────────────────────────────────────────┘    │
│                              │                                      │
│  Layer 4: 应用层                                                     │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │  • 镜像大小优化                                               │    │
│  │  • 应用性能调优                                               │    │
│  │  • 缓存策略                                                  │    │
│  └─────────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────────┘
```


### 成本优化策略对比

| 策略 | 节省比例 | 实施难度 | 风险 | 适用场景 |
|------|----------|----------|------|----------|
| **Spot 实例** | 60-90% | 中 | 中断风险 | 无状态、批处理 |
| **Reserved 实例** | 30-60% | 低 | 锁定风险 | 稳定基线负载 |
| **Graviton (ARM)** | 20-40% | 中 | 兼容性 | 新应用 |
| **资源优化** | 20-50% | 中 | 性能影响 | 所有应用 |
| **Karpenter** | 15-30% | 中 | 学习曲线 | 动态负载 |
| **节点整合** | 10-30% | 低 | 可用性 | 低利用率集群 |

### 资源优化参数表

| 参数 | 默认值 | 优化建议 | 影响 |
|------|--------|----------|------|
| **requests.cpu** | 无 | 基于 P95 使用量 | 调度决策 |
| **requests.memory** | 无 | 基于 P99 使用量 | 调度决策 |
| **limits.cpu** | 无 | requests 的 2-4 倍 | 限流 |
| **limits.memory** | 无 | requests 的 1.2-1.5 倍 | OOM |
| **HPA minReplicas** | 1 | 基于 SLA 要求 | 可用性 |
| **HPA maxReplicas** | 无 | 基于预算限制 | 成本上限 |
| **VPA updateMode** | Off | Auto (谨慎) | 自动调整 |


### 成本监控指标

```yaml
# Prometheus 成本相关告警规则
groups:
- name: cost-optimization
  rules:
  # CPU 请求过高告警
  - alert: CPURequestOverProvisioned
    expr: |
      (
        sum(kube_pod_container_resource_requests{resource="cpu"}) by (pod)
        /
        sum(rate(container_cpu_usage_seconds_total[5m])) by (pod)
      ) > 3
    for: 24h
    labels:
      severity: warning
    annotations:
      summary: "Pod {{ $labels.pod }} CPU 请求过高"
      
  # 内存请求过高告警
  - alert: MemoryRequestOverProvisioned
    expr: |
      (
        sum(kube_pod_container_resource_requests{resource="memory"}) by (pod)
        /
        sum(container_memory_working_set_bytes) by (pod)
      ) > 2
    for: 24h
    labels:
      severity: warning
      
  # 节点利用率过低告警
  - alert: NodeUnderutilized
    expr: |
      (
        sum(rate(container_cpu_usage_seconds_total[5m])) by (node)
        /
        sum(kube_node_status_allocatable{resource="cpu"}) by (node)
      ) < 0.3
    for: 6h
    labels:
      severity: info
```


### 成本优化工具对比

| 工具 | 类型 | 功能 | 定价 | 适用场景 |
|------|------|------|------|----------|
| **Kubecost** | 成本分析 | 成本分配、优化建议 | 免费/商业 | 成本可见性 |
| **CAST AI** | 自动优化 | 自动调整、Spot 管理 | 按节省付费 | 自动化优化 |
| **Spot.io** | Spot 管理 | Spot 实例管理 | 商业 | Spot 优化 |
| **Goldilocks** | VPA 建议 | 资源推荐 | 开源 | 资源优化 |
| **Kube-resource-report** | 报告 | 资源使用报告 | 开源 | 可见性 |
| **AWS Cost Explorer** | 云成本 | AWS 成本分析 | 免费 | AWS 用户 |

### 成本优化决策流程

```
┌─────────────────────────────────────────────────────────────────────┐
│                    成本优化决策流程                                   │
├─────────────────────────────────────────────────────────────────────┤
│  1. 成本可见性                                                        │
│     ├─ 部署 Kubecost                                                 │
│     ├─ 配置成本分配标签                                                │
│     └─ 建立成本基线                                                   │
│                              │                                      │
│                              ▼                                      │
│  2. 识别优化机会                                                      │
│     ├─ 分析资源利用率                                                 │
│     ├─ 识别过度配置                                                   │
│     └─ 评估 Spot 适用性                                               │
│                              │                                      │
│                              ▼                                      │
│  3. 实施优化                                                         │
│     ├─ 调整资源请求/限制                                              │
│     ├─ 配置 HPA/VPA                                                 │
│     ├─ 迁移到 Spot/Graviton                                         │
│     └─ 优化 Karpenter 配置                                           │
│                              │                                      │
│                              ▼                                      │
│  4. 持续监控                                                         │
│     ├─ 设置成本告警                                                   │
│     ├─ 定期审查报告                                                   │
│     └─ 迭代优化                                                      │
└─────────────────────────────────────────────────────────────────────┘
```


### 成本优化问题定位表

| 问题 | 原因 | 解决方案 |
|------|------|----------|
| 成本持续上升 | 资源过度配置 | 使用 VPA 推荐值 |
| Spot 中断频繁 | 实例类型单一 | 配置多种实例类型 |
| 节点利用率低 | 调度不均衡 | 使用 Karpenter |
| 成本分配不清 | 缺少标签 | 强制资源标签 |
| Reserved 浪费 | 预估不准 | 使用 Savings Plans |
| 跨 AZ 流量费 | 架构问题 | 拓扑感知路由 |

---

## 附录：Service Mesh 深度解析
### Service Mesh 架构演进

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Service Mesh 架构演进                             │
├─────────────────────────────────────────────────────────────────────┤
│  阶段 1: 库模式 (Library)                                            │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │  App + Netflix OSS (Hystrix, Ribbon, Eureka)                 │   │
│  │  • 语言绑定                                                   │   │
│  │  • 侵入式                                                     │   │
│  │  • 升级困难                                                   │   │
│  └──────────────────────────────────────────────────────────────┘   │
│                              │                                      │
│                              ▼                                      │
│  阶段 2: Sidecar 模式                                                │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │  App ←→ Envoy Sidecar ←→ Control Plane                       │   │
│  │  • 语言无关                                                   │   │
│  │  • 非侵入式                                                   │   │
│  │  • 资源开销                                                   │   │
│  └──────────────────────────────────────────────────────────────┘   │
│                              │                                      │
│                              ▼                                      │
│  阶段 3: Ambient Mesh (无 Sidecar)                                   │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │  App ←→ ztunnel (节点级) ←→ waypoint (可选 L7)                 │   │
│  │  • 更低资源开销                                                │   │
│  │  • 简化运维                                                   │   │
│  │  • 渐进式采用                                                  │   │
│  └──────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────┘
```


### Service Mesh 产品对比

| 特性             | Istio  | Linkerd        | Cilium SM | AWS App Mesh |
| -------------- | ------ | -------------- | --------- | ------------ |
| **数据平面**       | Envoy  | linkerd2-proxy | eBPF      | Envoy        |
| **控制平面**       | istiod | control plane  | Cilium    | App Mesh     |
| **资源开销**       | 高      | 低              | 最低        | 中            |
| **功能丰富度**      | 最高     | 中等             | 中等        | 中等           |
| **学习曲线**       | 陡峭     | 平缓             | 中等        | 中等           |
| **mTLS**       | ✅      | ✅              | ✅         | ✅            |
| **流量管理**       | ✅ 强大   | ✅ 基础           | ✅ 基础      | ✅ 中等         |
| **可观测性**       | ✅ 强大   | ✅ 良好           | ✅ Hubble  | ✅ X-Ray      |
| **多集群**        | ✅      | ✅              | ✅         | ✅            |
| **Ambient 模式** | ✅      | 🔴             | N/A       | 🔴           |

### Istio 核心资源

```yaml
# VirtualService - 流量路由
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: reviews-route
spec:
  hosts:
  - reviews
  http:
  - match:
    - headers:
        end-user:
          exact: jason
    route:
    - destination:
        host: reviews
        subset: v2
  - route:
    - destination:
        host: reviews
        subset: v1
      weight: 90
    - destination:
        host: reviews
        subset: v2
      weight: 10
```


```yaml
# DestinationRule - 流量策略
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: reviews-destination
spec:
  host: reviews
  trafficPolicy:
    connectionPool:
      tcp:
        maxConnections: 100
      http:
        h2UpgradePolicy: UPGRADE
        http1MaxPendingRequests: 100
    outlierDetection:
      consecutive5xxErrors: 5
      interval: 30s
      baseEjectionTime: 30s
  subsets:
  - name: v1
    labels:
      version: v1
  - name: v2
    labels:
      version: v2
    trafficPolicy:
      loadBalancer:
        simple: ROUND_ROBIN
```

### Service Mesh 参数调优表

| 参数 | 默认值 | 调优建议 | 影响 |
|------|--------|----------|------|
| **Envoy CPU** | 100m | 根据 QPS 调整 | 性能 |
| **Envoy Memory** | 128Mi | 根据连接数调整 | 稳定性 |
| **连接池大小** | 1024 | 根据并发调整 | 吞吐量 |
| **熔断阈值** | 5 | 根据 SLA 调整 | 可用性 |
| **重试次数** | 2 | 根据幂等性调整 | 延迟 |
| **超时时间** | 15s | 根据 P99 调整 | 用户体验 |


### Service Mesh 问题定位表

| 问题 | 原因 | 解决方案 |
|------|------|----------|
| 延迟增加 | Sidecar 开销 | 优化 Envoy 配置 |
| 503 错误 | 熔断触发 | 调整 outlierDetection |
| mTLS 失败 | 证书过期 | 检查 citadel/istiod |
| 流量不生效 | VirtualService 配置 | 检查 hosts 匹配 |
| 内存泄漏 | Envoy 配置 | 限制连接池大小 |
| 启动慢 | Sidecar 注入 | 使用 holdApplicationUntilProxyStarts |

---

## 附录：Kubernetes 证书管理
### 证书架构

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Kubernetes 证书体系                               │
├─────────────────────────────────────────────────────────────────────┤
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │                    Root CA (集群 CA)                         │    │
│  │                    /etc/kubernetes/pki/ca.crt               │    │
│  └─────────────────────────────────────────────────────────────┘    │
│       │              │              │              │                │
│       ▼              ▼              ▼              ▼                │
│  ┌──────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐                │
│  │API Server│  │ etcd    │  │ kubelet │  │ Front   │                │
│  │  证书     │  │  证书   │  │  证书    │  │ Proxy   │                │
│  └──────────┘  └─────────┘  └─────────┘  └─────────┘                │
│                                                                     │
│  证书用途：                                                           │
│  • API Server: 服务端证书 + 客户端证书 (访问 etcd/kubelet)              │
│  • etcd: 服务端证书 + peer 证书 + 客户端证书                            │
│  • kubelet: 服务端证书 + 客户端证书 (访问 API Server)                   │
│  • Front Proxy: 用于 API 聚合                                        │
└─────────────────────────────────────────────────────────────────────┘
```


### cert-manager 架构

```yaml
# cert-manager Issuer 示例
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: admin@example.com
    privateKeySecretRef:
      name: letsencrypt-prod-key
    solvers:
    - http01:
        ingress:
          class: nginx
    - dns01:
        route53:
          region: us-east-1
          hostedZoneID: Z1234567890

---
# Certificate 示例
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: example-com-tls
  namespace: default
spec:
  secretName: example-com-tls
  issuerRef:
    name: letsencrypt-prod
    kind: ClusterIssuer
  dnsNames:
  - example.com
  - "*.example.com"
  duration: 2160h    # 90 天
  renewBefore: 360h  # 15 天前续期
```

### 证书管理最佳实践

| 实践 | 说明 | 工具 |
|------|------|------|
| **自动续期** | 避免证书过期 | cert-manager |
| **短期证书** | 减少泄露风险 | 90 天或更短 |
| **证书轮换** | 定期更新 | kubeadm certs renew |
| **监控告警** | 过期前告警 | Prometheus |
| **密钥保护** | 安全存储私钥 | Vault/KMS |
| **审计日志** | 记录证书操作 | 审计策略 |


### 证书问题定位表

| 问题 | 原因 | 解决方案 |
|------|------|----------|
| x509: certificate expired | 证书过期 | kubeadm certs renew |
| x509: unknown authority | CA 不信任 | 检查 CA 证书链 |
| TLS handshake timeout | 网络/证书问题 | 检查端口和证书 |
| certificate signed by unknown authority | 证书不匹配 | 重新签发证书 |
| cert-manager 续期失败 | DNS/HTTP 验证失败 | 检查 solver 配置 |

---

## 附录：Kubernetes 网关 API
### Gateway API 架构

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Gateway API 资源模型                              │
├─────────────────────────────────────────────────────────────────────┤
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │                    GatewayClass                             │    │
│  │                    (集群级别，定义网关类型)                    │    │
│  │                    例如: nginx, istio, aws-alb              │    │
│  └─────────────────────────────────────────────────────────────┘    │
│                              │                                      │
│                              ▼                                      │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │                    Gateway                                  │    │
│  │                    (命名空间级别，定义监听器)                   │    │
│  │                    Listeners: HTTP/HTTPS/TCP/UDP            │    │
│  └─────────────────────────────────────────────────────────────┘    │
│                              │                                      │
│                              ▼                                      │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │                    HTTPRoute / TCPRoute / etc.              │    │
│  │                    (命名空间级别，定义路由规则)                 │    │
│  │                    路径匹配、Header 匹配、权重分配              │    │
│  └─────────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────────┘
```


### Gateway API vs Ingress 对比

| 特性            | Ingress    | Gateway API             |
| ------------- | ---------- | ----------------------- |
| **API 成熟度**   | Stable     | GA (v1.0)               |
| **角色分离**      | 🔴 单一资源    | ✅ 基础设施/应用分离             |
| **协议支持**      | HTTP/HTTPS | HTTP/HTTPS/TCP/UDP/gRPC |
| **Header 匹配** | 注解依赖       | ✅ 原生支持                  |
| **流量分割**      | 注解依赖       | ✅ 原生支持                  |
| **跨命名空间**     | 🔴         | ✅ ReferenceGrant        |
| **可移植性**      | 低 (注解差异)   | 高 (标准 API)              |

### Gateway API 示例

```yaml
# GatewayClass
apiVersion: gateway.networking.k8s.io/v1
kind: GatewayClass
metadata:
  name: aws-alb
spec:
  controllerName: eks.amazonaws.com/alb

---
# Gateway
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: main-gateway
  namespace: gateway-system
spec:
  gatewayClassName: aws-alb
  listeners:
  - name: http
    port: 80
    protocol: HTTP
    allowedRoutes:
      namespaces:
        from: All
  - name: https
    port: 443
    protocol: HTTPS
    tls:
      mode: Terminate
      certificateRefs:
      - name: example-com-tls
```


```yaml
# HTTPRoute
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: app-route
  namespace: production
spec:
  parentRefs:
  - name: main-gateway
    namespace: gateway-system
  hostnames:
  - "app.example.com"
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /api
    - headers:
      - name: X-Version
        value: v2
    backendRefs:
    - name: api-service-v2
      port: 80
      weight: 90
    - name: api-service-v3
      port: 80
      weight: 10
  - matches:
    - path:
        type: PathPrefix
        value: /
    backendRefs:
    - name: frontend-service
      port: 80
```

---

## 附录：Kubernetes 准入控制深度解析
### 准入控制流程

```
┌─────────────────────────────────────────────────────────────────────┐
│                    API Server 准入控制流程                            │
├─────────────────────────────────────────────────────────────────────┤
│  请求 ──▶ 认证 ──▶ 授权 ──▶ 准入控制 ──▶ 持久化                         │
│                              │                                      │
│                              ▼                                      │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │                    Mutating Admission                       │    │
│  │  ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐         │    │
│  │  │ Default │  │ Sidecar │  │ Label   │  │ Custom  │         │    │
│  │  │ Values  │  │ Inject  │  │ Inject  │  │ Webhook │         │    │
│  │  └─────────┘  └─────────┘  └─────────┘  └─────────┘         │    │
│  └─────────────────────────────────────────────────────────────┘    │
│                              │                                      │
│                              ▼                                      │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │                    Validating Admission                     │    │
│  │  ┌─────────┐  ┌───────────┐  ┌─────────┐  ┌─────────┐       │    │
│  │  │ PSA     │  │ OPA/      │  │ Kyverno │  │ Custom  │       │    │
│  │  │ Check   │  │ Gatekeeper│  │ Policy  │  │ Webhook │       │    │
│  │  └─────────┘  └───────────┘  └─────────┘  └─────────┘       │    │
│  └─────────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────────┘
```


### 策略引擎对比

| 特性             | OPA/Gatekeeper | Kyverno | Kubewarden  |
| -------------- | -------------- | ------- | ----------- |
| **策略语言**       | Rego           | YAML    | WebAssembly |
| **学习曲线**       | 陡峭             | 平缓      | 中等          |
| **Mutating**   | ✅              | ✅       | ✅           |
| **Validating** | ✅              | ✅       | ✅           |
| **生成资源**       | 🔴             | ✅       | 🔴          |
| **审计模式**       | ✅              | ✅       | ✅           |
| **策略库**        | 丰富             | 丰富      | 增长中         |
| **性能**         | 良好             | 良好      | 优秀          |

### Kyverno 策略示例

```yaml
# 强制要求资源限制
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: require-resource-limits
spec:
  validationFailureAction: Enforce
  rules:
  - name: require-limits
    match:
      any:
      - resources:
          kinds:
          - Pod
    validate:
      message: "CPU and memory limits are required"
      pattern:
        spec:
          containers:
          - resources:
              limits:
                memory: "?*"
                cpu: "?*"

---
# 自动添加标签
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: add-labels
spec:
  rules:
  - name: add-team-label
    match:
      any:
      - resources:
          kinds:
          - Pod
    mutate:
      patchStrategicMerge:
        metadata:
          labels:
            managed-by: kyverno
```


### 准入控制问题定位表

| 问题 | 原因 | 解决方案 |
|------|------|----------|
| 资源创建被拒绝 | 策略验证失败 | 检查策略规则 |
| Webhook 超时 | 网络/服务问题 | 检查 Webhook 服务 |
| 策略不生效 | 匹配规则错误 | 检查 match 配置 |
| 性能下降 | Webhook 延迟 | 优化策略或增加副本 |
| 证书错误 | Webhook 证书过期 | 更新证书 |

---

## 附录：Kubernetes 资源优先级与抢占
### 优先级与抢占架构

```
┌─────────────────────────────────────────────────────────────────────┐
│                    优先级与抢占机制                                   │
├─────────────────────────────────────────────────────────────────────┤
│  PriorityClass 定义                                                 │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │  system-cluster-critical: 2000000000                        │    │
│  │  system-node-critical:    2000001000                        │    │
│  │  high-priority:           1000000                           │    │
│  │  default:                 0                                 │    │
│  │  low-priority:            -1000                             │    │
│  └─────────────────────────────────────────────────────────────┘    │
│                              │                                      │
│  抢占流程                                                            │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │  1. 高优先级 Pod 无法调度                                      │    │
│  │  2. 调度器查找可抢占的低优先级 Pod                              │    │
│  │  3. 选择最小影响的抢占方案                                      │    │
│  │  4. 驱逐低优先级 Pod                                          │    │
│  │  5. 调度高优先级 Pod                                          │    │
│  └─────────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────────┘
```


### PriorityClass 配置示例

```yaml
# 高优先级 PriorityClass
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: high-priority
value: 1000000
globalDefault: false
preemptionPolicy: PreemptLowerPriority
description: "用于关键业务应用"

---
# 低优先级 PriorityClass (可被抢占)
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: low-priority
value: -1000
globalDefault: false
preemptionPolicy: Never  # 不抢占其他 Pod
description: "用于批处理任务"

---
# 使用 PriorityClass 的 Pod
apiVersion: v1
kind: Pod
metadata:
  name: critical-app
spec:
  priorityClassName: high-priority
  containers:
  - name: app
    image: myapp:latest
```

### 优先级最佳实践

| 实践 | 说明 | 示例 |
|------|------|------|
| **分层优先级** | 定义清晰的优先级层级 | 系统 > 关键业务 > 普通 > 批处理 |
| **避免过高值** | 不要接近系统保留值 | < 1000000000 |
| **配合 PDB** | 保护关键应用 | minAvailable: 1 |
| **监控抢占** | 跟踪抢占事件 | Prometheus 指标 |
| **测试验证** | 验证抢占行为 | 混沌工程 |


---

## 附录：Kubernetes 垂直 Pod 自动伸缩 (VPA) 深度解析
### VPA 架构

```
┌─────────────────────────────────────────────────────────────────────┐
│                    VPA 组件架构                                      │
├─────────────────────────────────────────────────────────────────────┤
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │                    VPA Recommender                          │    │
│  │  • 收集历史资源使用数据                                        │    │
│  │  • 计算推荐的 requests/limits                                │    │
│  │  • 基于 P95/P99 百分位数                                      │    │
│  └─────────────────────────────────────────────────────────────┘    │
│                              │                                      │
│                              ▼                                      │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │                    VPA Updater                              │    │
│  │  • 监控 Pod 资源配置                                          │    │
│  │  • 驱逐需要更新的 Pod                                         │    │
│  │  • 触发 Pod 重建                                             │    │
│  └─────────────────────────────────────────────────────────────┘    │
│                              │                                      │
│                              ▼                                      │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │                    VPA Admission Controller                 │    │
│  │  • 拦截 Pod 创建请求                                          │    │
│  │  • 注入推荐的资源配置                                          │    │
│  │  • Mutating Webhook                                         │    │
│  └─────────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────────┘
```


### VPA 配置示例

```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: my-app-vpa
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: my-app
  updatePolicy:
    updateMode: "Auto"  # Off, Initial, Recreate, Auto
  resourcePolicy:
    containerPolicies:
    - containerName: "*"
      minAllowed:
        cpu: 100m
        memory: 128Mi
      maxAllowed:
        cpu: 4
        memory: 8Gi
      controlledResources: ["cpu", "memory"]
      controlledValues: RequestsAndLimits
```

### VPA 更新模式对比

| 模式 | 行为 | 适用场景 |
|------|------|----------|
| **Off** | 仅生成推荐，不更新 | 观察阶段 |
| **Initial** | 仅在 Pod 创建时应用 | 保守策略 |
| **Recreate** | 驱逐并重建 Pod | 可接受中断 |
| **Auto** | 自动选择最佳方式 | 生产环境 |

### VPA vs HPA 对比

| 特性 | VPA | HPA |
|------|-----|-----|
| **伸缩维度** | 垂直 (资源) | 水平 (副本) |
| **触发条件** | 资源使用率 | 指标阈值 |
| **中断影响** | 需要重建 Pod | 无中断 |
| **适用场景** | 单副本/有状态 | 无状态应用 |
| **组合使用** | ⚠️ 需谨慎 | ✅ 推荐 |


### VPA 问题定位表

| 问题 | 原因 | 解决方案 |
|------|------|----------|
| 推荐值不准确 | 数据不足 | 等待更多历史数据 |
| Pod 频繁重建 | 阈值过敏感 | 调整 minChange |
| 与 HPA 冲突 | 同时调整 CPU | VPA 仅管理 memory |
| OOM 仍然发生 | maxAllowed 过低 | 提高上限 |
| 推荐值过高 | 峰值影响 | 调整百分位数 |

---

## 附录：Kubernetes Pod 中断预算 (PDB) 深度解析

### PDB 工作原理

```
┌─────────────────────────────────────────────────────────────────────┐
│                    PDB 保护机制                                      │
├─────────────────────────────────────────────────────────────────────┤
│  自愿中断 (受 PDB 保护)                                               │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │  • kubectl drain (节点维护)                                  │    │
│  │  • kubectl delete pod (手动删除)                             │    │
│  │  • Deployment 滚动更新                                       │    │
│  │  • Cluster Autoscaler 缩容                                   │    │
│  │  • VPA 重建 Pod                                              │    │
│  └─────────────────────────────────────────────────────────────┘    │
│                                                                     │
│  非自愿中断 (不受 PDB 保护)                                            │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │  • 节点故障                                                  │    │
│  │  • OOM Killer                                               │    │
│  │  • 内核崩溃                                                  │    │
│  │  • 硬件故障                                                  │    │
│  └─────────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────────┘
```


### PDB 配置示例

```yaml
# 基于最小可用数
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: app-pdb
spec:
  minAvailable: 2
  selector:
    matchLabels:
      app: my-app

---
# 基于最大不可用数
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: app-pdb-percentage
spec:
  maxUnavailable: 25%
  selector:
    matchLabels:
      app: my-app

---
# 不健康 Pod 阈值 (K8s 1.27+)
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: app-pdb-unhealthy
spec:
  minAvailable: 2
  unhealthyPodEvictionPolicy: AlwaysAllow
  selector:
    matchLabels:
      app: my-app
```

### PDB 最佳实践

| 实践 | 说明 | 示例 |
|------|------|------|
| **设置合理阈值** | 平衡可用性和运维 | minAvailable: N-1 |
| **使用百分比** | 适应副本数变化 | maxUnavailable: 25% |
| **配合反亲和** | 分散 Pod 分布 | podAntiAffinity |
| **监控 PDB 状态** | 跟踪中断预算 | kubectl get pdb |
| **处理不健康 Pod** | 避免阻塞运维 | unhealthyPodEvictionPolicy |


### PDB 问题定位表

| 问题 | 原因 | 解决方案 |
|------|------|----------|
| drain 卡住 | PDB 阻止驱逐 | 检查 PDB 配置 |
| 滚动更新慢 | minAvailable 过高 | 降低阈值 |
| 节点无法缩容 | PDB 保护 | 调整 PDB 或手动迁移 |
| 不健康 Pod 阻塞 | 默认策略 | 使用 AlwaysAllow |

---

## 附录：Kubernetes 资源配额与限制范围
### 资源管理层级

```
┌─────────────────────────────────────────────────────────────────────┐
│                    资源管理层级                                       │
├─────────────────────────────────────────────────────────────────────┤
│  集群级别                                                            │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │  • 节点资源总量                                               │    │
│  │  • 集群配额 (多租户)                                          │    │
│  └─────────────────────────────────────────────────────────────┘    │
│                              │                                      │
│  命名空间级别                                                         │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │  ResourceQuota                                              │    │
│  │  • 限制命名空间资源总量                                        │    │
│  │  • CPU/Memory/Storage/对象数量                               │    │
│  └─────────────────────────────────────────────────────────────┘    │
│                              │                                      │
│  容器级别                                                            │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │  LimitRange                                                 │    │
│  │  • 限制单个容器资源范围                                        │    │
│  │  • 设置默认值                                                 │    │
│  └─────────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────────┘
```


### ResourceQuota 配置示例

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: team-quota
  namespace: team-a
spec:
  hard:
    # 计算资源
    requests.cpu: "20"
    requests.memory: 40Gi
    limits.cpu: "40"
    limits.memory: 80Gi
    # 存储资源
    requests.storage: 100Gi
    persistentvolumeclaims: "10"
    # 对象数量
    pods: "50"
    services: "20"
    secrets: "100"
    configmaps: "100"
    # 特定 StorageClass
    gold.storageclass.storage.k8s.io/requests.storage: 50Gi
```

### LimitRange 配置示例

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: container-limits
  namespace: team-a
spec:
  limits:
  - type: Container
    default:
      cpu: 500m
      memory: 512Mi
    defaultRequest:
      cpu: 100m
      memory: 128Mi
    min:
      cpu: 50m
      memory: 64Mi
    max:
      cpu: 4
      memory: 8Gi
  - type: PersistentVolumeClaim
    min:
      storage: 1Gi
    max:
      storage: 100Gi
```


### 资源配额问题定位表

| 问题 | 原因 | 解决方案 |
|------|------|----------|
| Pod 创建失败 | 超出配额 | 增加配额或释放资源 |
| 缺少 requests | LimitRange 未设置默认值 | 配置 defaultRequest |
| 配额计算不准 | 包含 Terminating Pod | 等待清理完成 |
| 存储配额超限 | PVC 未释放 | 删除未使用 PVC |

---

## 附录：Kubernetes 节点管理

### 节点生命周期

```
┌─────────────────────────────────────────────────────────────────────┐
│                    节点生命周期管理                                  │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  节点加入                                                           │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │  1. kubelet 启动并注册到 API Server                          │    │
│  │  2. Node Controller 验证节点                                 │    │
│  │  3. 节点状态变为 Ready                                       │    │
│  │  4. 调度器开始调度 Pod                                       │    │
│  └─────────────────────────────────────────────────────────────┘    │
│                              │                                       │
│  节点维护                                                           │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │  1. kubectl cordon (标记不可调度)                            │    │
│  │  2. kubectl drain (驱逐 Pod)                                 │    │
│  │  3. 执行维护操作                                             │    │
│  │  4. kubectl uncordon (恢复调度)                              │    │
│  └─────────────────────────────────────────────────────────────┘    │
│                              │                                       │
│  节点移除                                                           │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │  1. kubectl drain --delete-emptydir-data                     │    │
│  │  2. kubectl delete node                                      │    │
│  │  3. 清理节点资源                                             │    │
│  └─────────────────────────────────────────────────────────────┘    │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```


### 节点管理命令

```bash
# 查看节点状态
kubectl get nodes -o wide
kubectl describe node <node-name>

# 标记节点不可调度
kubectl cordon <node-name>

# 驱逐节点上的 Pod
kubectl drain <node-name> \
  --ignore-daemonsets \
  --delete-emptydir-data \
  --force \
  --grace-period=30

# 恢复节点调度
kubectl uncordon <node-name>

# 添加节点标签
kubectl label nodes <node-name> node-type=compute

# 添加节点污点
kubectl taint nodes <node-name> dedicated=special:NoSchedule

# 查看节点资源使用
kubectl top nodes

# 查看节点事件
kubectl get events --field-selector involvedObject.kind=Node
```

### 节点状态条件

| 条件 | 含义 | 影响 |
|------|------|------|
| **Ready** | kubelet 健康 | 可调度 |
| **MemoryPressure** | 内存不足 | 驱逐 Pod |
| **DiskPressure** | 磁盘不足 | 驱逐 Pod |
| **PIDPressure** | PID 不足 | 驱逐 Pod |
| **NetworkUnavailable** | 网络未配置 | 不可调度 |


### 节点问题定位表

| 问题 | 原因 | 解决方案 |
|------|------|----------|
| NotReady | kubelet 异常 | 检查 kubelet 日志 |
| MemoryPressure | 内存不足 | 扩容或驱逐 Pod |
| DiskPressure | 磁盘满 | 清理镜像/日志 |
| drain 卡住 | PDB 阻止 | 检查 PDB 配置 |
| 节点无法加入 | 证书/网络问题 | 检查 kubeadm join |

---

## 附录：容器镜像优化指南

### 镜像分层优化

```
┌─────────────────────────────────────────────────────────────────────┐
│                    镜像分层最佳实践                                  │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  优化前                          优化后                             │
│  ┌─────────────────────┐        ┌─────────────────────┐            │
│  │ Layer 5: App Code   │        │ Layer 3: App Code   │            │
│  │ (变化频繁)          │        │ (变化频繁)          │            │
│  ├─────────────────────┤        ├─────────────────────┤            │
│  │ Layer 4: Dependencies│       │ Layer 2: Dependencies│            │
│  │ (偶尔变化)          │        │ (偶尔变化)          │            │
│  ├─────────────────────┤        ├─────────────────────┤            │
│  │ Layer 3: OS Packages│        │ Layer 1: Distroless │            │
│  │ (很少变化)          │        │ (最小基础镜像)      │            │
│  ├─────────────────────┤        └─────────────────────┘            │
│  │ Layer 2: Base Image │                                           │
│  ├─────────────────────┤        大小: 50MB                         │
│  │ Layer 1: OS         │        层数: 3                            │
│  └─────────────────────┘                                           │
│  大小: 500MB                                                        │
│  层数: 5                                                            │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```


### 多阶段构建示例

```dockerfile
# 阶段 1: 构建
FROM golang:1.21-alpine AS builder
WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -ldflags="-s -w" -o /app/main

# 阶段 2: 运行
FROM gcr.io/distroless/static:nonroot
COPY --from=builder /app/main /
USER nonroot:nonroot
ENTRYPOINT ["/main"]
```

### 基础镜像对比

| 镜像 | 大小 | 安全性 | Shell | 适用场景 |
|------|------|--------|-------|----------|
| **ubuntu** | ~77MB | 中 | ✅ | 开发调试 |
| **alpine** | ~5MB | 高 | ✅ | 通用生产 |
| **distroless** | ~2MB | 最高 | ❌ | 安全敏感 |
| **scratch** | 0MB | 最高 | ❌ | 静态二进制 |
| **chainguard** | ~2MB | 最高 | ❌ | 企业安全 |

### 镜像优化检查清单

| 检查项 | 说明 | 工具 |
|--------|------|------|
| **多阶段构建** | 分离构建和运行环境 | Dockerfile |
| **最小基础镜像** | 使用 alpine/distroless | - |
| **合并 RUN 指令** | 减少层数 | Dockerfile |
| **清理缓存** | 删除包管理器缓存 | rm -rf /var/cache |
| **使用 .dockerignore** | 排除不需要的文件 | .dockerignore |
| **固定版本** | 避免 latest 标签 | 版本号 |
| **漏洞扫描** | 检查安全漏洞 | Trivy/Snyk |


### 镜像问题定位表

| 问题 | 原因 | 解决方案 |
|------|------|----------|
| 镜像过大 | 包含构建工具 | 使用多阶段构建 |
| 拉取慢 | 镜像层过多 | 合并 RUN 指令 |
| 漏洞多 | 基础镜像过时 | 更新基础镜像 |
| 构建慢 | 缓存失效 | 优化 COPY 顺序 |
| 权限问题 | root 用户运行 | 使用非 root 用户 |

---

## 附录：Kubernetes 日志最佳实践

### 日志架构

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Kubernetes 日志架构                               │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  应用层                                                             │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │  App → stdout/stderr → Container Runtime → Node Log File    │    │
│  │                        /var/log/containers/*.log            │    │
│  └─────────────────────────────────────────────────────────────┘    │
│                              │                                       │
│  采集层                                                             │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │  DaemonSet: Fluent Bit / Fluentd / Vector                   │    │
│  │  • 读取节点日志文件                                          │    │
│  │  • 解析和转换                                                │    │
│  │  • 添加 K8S 元数据                                          │    │
│  └─────────────────────────────────────────────────────────────┘    │
│                              │                                       │
│  存储层                                                             │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │  Elasticsearch / Loki / CloudWatch Logs / S3                │    │
│  └─────────────────────────────────────────────────────────────┘    │
│                              │                                       │
│  查询层                                                             │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │  Kibana / Grafana / CloudWatch Insights                     │    │
│  └─────────────────────────────────────────────────────────────┘    │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```


### 日志采集器对比

| 特性 | Fluent Bit | Fluentd | Vector |
|------|------------|---------|--------|
| **语言** | C | Ruby | Rust |
| **内存占用** | ~5MB | ~40MB | ~10MB |
| **性能** | 高 | 中 | 最高 |
| **插件生态** | 中 | 丰富 | 增长中 |
| **配置复杂度** | 低 | 中 | 中 |
| **适用场景** | 边缘/资源受限 | 复杂处理 | 高性能 |

### 结构化日志示例

```json
{
  "timestamp": "2024-01-15T10:30:00.000Z",
  "level": "INFO",
  "service": "order-service",
  "trace_id": "abc123",
  "span_id": "def456",
  "message": "Order created successfully",
  "order_id": "ORD-12345",
  "user_id": "USR-67890",
  "duration_ms": 150
}
```

### 日志最佳实践

| 实践 | 说明 | 示例 |
|------|------|------|
| **结构化日志** | 使用 JSON 格式 | {"level":"INFO"} |
| **统一时间格式** | ISO 8601 | 2024-01-15T10:30:00Z |
| **包含追踪 ID** | 关联分布式追踪 | trace_id, span_id |
| **日志级别** | 合理使用级别 | DEBUG/INFO/WARN/ERROR |
| **避免敏感信息** | 脱敏处理 | 不记录密码/Token |
| **日志轮转** | 防止磁盘满 | logrotate 配置 |


### 日志问题定位表

| 问题 | 原因 | 解决方案 |
|------|------|----------|
| 日志丢失 | 采集器资源不足 | 增加资源或缓冲 |
| 延迟高 | 网络/存储瓶颈 | 优化批量发送 |
| 磁盘满 | 日志轮转未配置 | 配置 logrotate |
| 解析失败 | 日志格式不一致 | 统一日志格式 |
| 成本高 | 日志量过大 | 采样或过滤 |

---

## 附录：Kubernetes 事件管理

### 事件架构

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Kubernetes 事件系统                               │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  事件来源                                                           │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │  kubelet │ kube-scheduler │ controllers │ operators        │    │
│  └─────────────────────────────────────────────────────────────┘    │
│                              │                                       │
│                              ▼                                       │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │                    API Server                                │    │
│  │                    (事件存储在 etcd)                         │    │
│  │                    默认保留 1 小时                           │    │
│  └─────────────────────────────────────────────────────────────┘    │
│                              │                                       │
│                              ▼                                       │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │  事件导出器                                                  │    │
│  │  kubernetes-event-exporter / eventrouter                    │    │
│  │  → Elasticsearch / Loki / CloudWatch / Slack               │    │
│  └─────────────────────────────────────────────────────────────┘    │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```


### 常见事件类型

| 事件 | 类型 | 含义 | 处理建议 |
|------|------|------|----------|
| **Scheduled** | Normal | Pod 已调度 | 正常 |
| **Pulled** | Normal | 镜像已拉取 | 正常 |
| **Created** | Normal | 容器已创建 | 正常 |
| **Started** | Normal | 容器已启动 | 正常 |
| **FailedScheduling** | Warning | 调度失败 | 检查资源/亲和性 |
| **FailedMount** | Warning | 挂载失败 | 检查 PV/Secret |
| **BackOff** | Warning | 重启退避 | 检查应用日志 |
| **Unhealthy** | Warning | 探针失败 | 检查探针配置 |
| **OOMKilled** | Warning | 内存不足 | 增加内存限制 |
| **Evicted** | Warning | Pod 被驱逐 | 检查节点资源 |

### 事件查询命令

```bash
# 查看所有事件
kubectl get events --sort-by='.lastTimestamp'

# 查看特定命名空间事件
kubectl get events -n production

# 查看 Warning 事件
kubectl get events --field-selector type=Warning

# 查看特定 Pod 事件
kubectl get events --field-selector involvedObject.name=my-pod

# 持续监控事件
kubectl get events -w

# 查看事件详情
kubectl describe event <event-name>
```


### 事件问题定位表

| 问题 | 原因 | 解决方案 |
|------|------|----------|
| 事件丢失 | 默认只保留 1 小时 | 部署事件导出器 |
| 事件过多 | 频繁重启/调度 | 解决根本问题 |
| 无法查询历史 | etcd 清理 | 导出到外部存储 |

---

## 附录：Kubernetes 版本升级检查清单
### 升级前检查

| 检查项 | 命令/操作 | 说明 |
|--------|----------|------|
| **备份 etcd** | etcdctl snapshot save | 必须 |
| **检查 API 废弃** | kubectl deprecations | 检查废弃 API |
| **检查 PDB** | kubectl get pdb | 确保可驱逐 |
| **检查节点状态** | kubectl get nodes | 所有节点 Ready |
| **检查 Pod 状态** | kubectl get pods -A | 无异常 Pod |
| **检查存储** | kubectl get pv,pvc | 存储正常 |
| **检查证书** | kubeadm certs check-expiration | 证书有效 |
| **阅读 Release Notes** | K8S 官方文档 | 了解变更 |

### 升级后验证

| 验证项 | 命令/操作 | 说明 |
|--------|----------|------|
| **集群版本** | kubectl version | 确认版本 |
| **节点版本** | kubectl get nodes | 所有节点升级 |
| **核心组件** | kubectl get pods -n kube-system | 组件正常 |
| **应用状态** | kubectl get pods -A | 应用正常 |
| **网络连通** | 测试 Service/Ingress | 网络正常 |
| **存储访问** | 测试 PVC 读写 | 存储正常 |
| **监控告警** | 检查 Prometheus | 无异常告警 |

---
