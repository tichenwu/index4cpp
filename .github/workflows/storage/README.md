# 分布式存储方向 — 学习项目与 clangd 索引工作流

本目录收录**分布式存储系统岗**常见开源项目的阅读清单，以及其中 **C/C++ 项目**对应的 clangd 预构建工作流。

> **存储引擎打底**（LevelDB、RocksDB）的工作流在上一级目录：[`../build-leveldb-clangd-index.yml`](../build-leveldb-clangd-index.yml)、[`../build-rocksdb-clangd-index.yml`](../build-rocksdb-clangd-index.yml)。

---

## 一、分层关系：引擎与分布式系统

分布式存储不是「和 LSM 引擎二选一」，而是**上下两层相辅相成**：

```
客户端 ── S3 / POSIX / 块设备 ──► Ceph / MinIO / JuiceFS   ← 分布式对象/文件（系统层）
                                        │
                                        ▼
                         RocksDB / 自研 LSM / 文件系统      ← 本机存储引擎（引擎层）
                                        │
                                        ▼
                              磁盘 / SSD / 网络块设备
```

| 层次 | 解决什么问题 | 代表项目 |
|------|--------------|----------|
| **引擎层** | 单机如何把 KV/对象可靠、高效地写到盘上（WAL、SST、Compaction） | LevelDB、RocksDB |
| **共识层** | 多副本如何选主、复制日志、保证一致性 | braft、etcd |
| **系统层** | 多机如何对外提供对象/块/文件服务（分片、副本、元数据、协议） | Ceph、MinIO、JuiceFS |
| **加分项** | 分布式 KV 全栈、列存、垂直文件系统等 | TiKV、FoundationDB、Kudu、GlusterFS |

**学习顺序**：先引擎 → 再共识 → 再系统。跳过引擎直接啃 Ceph，容易看不懂 BlueStore/RocksDB 等落盘路径。

---

## 二、推荐学习路径（按顺序执行）

以下各阶段**按编号顺序推进**；阶段 3 在两条路线中**二选一**深啃，不要并行通读多个大系统。

### 阶段 0：前置（约 1–2 周）

**目标**：能跟读 C/C++ 源码，会用 clangd 索引或 `compile_commands.json` 跳转。

**需要具备**：
- C/C++ 基础、指针与内存模型
- 基本 Linux：进程、文件 I/O、`epoll` 概念
- 网络基础：TCP、HTTP 概念即可

**工具准备**：
- 源码解压到 `/codebase/<项目>`（与 Actions 构建路径一致）
- 配置 clangd：见本文「五、工作流一览」与 `clangd/config-storage-projects.yaml`
- 用法细节见上级 [`../README.md`](../README.md)

**完成标准**：能在 IDE 里对一个开源 C++ 项目做「跳转到定义 / 查引用」。

---

### 阶段 1：LSM 存储引擎打底（约 4–8 周）

**目标**：理解「数据如何从 `Put` 落到磁盘」，掌握 LSM 读写路径与 Compaction。

#### 1.1 LevelDB（入门，约 2–3 周）

| 项 | 说明 |
|----|------|
| 仓库 | [google/leveldb](https://github.com/google/leveldb) |
| workflow | [`../build-leveldb-clangd-index.yml`](../build-leveldb-clangd-index.yml) |
| CDB 目录 | `/codebase/leveldb/build` |

**建议阅读顺序**：
1. `db/db_impl.cc` — 对外 API 与读写主路径
2. `db/memtable.cc`、`db/version_set.cc` — MemTable 与版本管理
3. `table/` — SST 格式与迭代
4. `db/log_*` — WAL

**完成标准**：能口述一次 `Put` 的路径（MemTable → WAL → Flush → SST → Compaction），并解释写放大/读放大。

#### 1.2 RocksDB（进阶，约 4–6 周）

| 项 | 说明 |
|----|------|
| 仓库 | [facebook/rocksdb](https://github.com/facebook/rocksdb) |
| workflow | [`../build-rocksdb-clangd-index.yml`](../build-rocksdb-clangd-index.yml) |
| CDB 目录 | `/codebase/rocksdb/build` |

**建议阅读顺序**：
1. 在 LevelDB 基础上对照读 `db/db_impl.cc`
2. `db/column_family.cc` — Column Family
3. `db/flush_job.cc`、`db/compaction/` — Flush 与 Compaction 策略
4. `table/block_based/` — Block Cache、Filter

**完成标准**：能讲清 Compaction 策略差异（leveled vs universal）、WAL 与崩溃恢复、Column Family 用途；知道 TiKV/Ceph BlueStore 等为何选用 RocksDB。

**常见坑**：不要试图全仓通读；以「写路径 + 读路径 + Compaction」三条线为主。

---

### 阶段 2：分布式一致性（约 3–4 周）

**目标**：理解选主、日志复制、多数派，为读分布式元数据/副本逻辑打基础。

#### 2.1 braft（C++ 源码，主线路）

| 项 | 说明 |
|----|------|
| 仓库 | [baidu/braft](https://github.com/baidu/braft) |
| workflow | [`build-braft-clangd-index.yml`](build-braft-clangd-index.yml) |
| CDB 目录 | `/codebase/braft/build` |

**建议阅读顺序**：
1. Raft 论文（或可视化 Raft）
2. `src/braft/raft.cpp` — 状态机与角色转换
3. `src/braft/log.cpp`、`src/braft/replicator.cpp` — 日志与复制

**完成标准**：能画 Raft 选主与日志复制流程，说明与存储系统元数据集群的关系。

#### 2.2 etcd（概念补充，约 1 周）

| 项 | 说明 |
|----|------|
| 仓库 | [etcd-io/etcd](https://github.com/etcd-io/etcd) |
| 工具 | Go 项目，用 **gopls**，无 clangd workflow |

**学什么**：租约、Watch、线性一致读；作为「协调服务」产品形态对照 braft 库。不必深啃 Go 源码，除非 JD 明确要求。

---

### 阶段 3：分布式存储系统（二选一深啃，约 8–16 周）

**目标**：能讲清一个完整分布式存储产品的架构，并跟读一条主路径源码。

不要同时深啃 Ceph 与 MinIO+JuiceFS；选与目标 JD 更贴近的一条。

#### 路线 A：统一存储 — Ceph

| 项 | 说明 |
|----|------|
| 仓库 | [ceph/ceph](https://github.com/ceph/ceph) |
| workflow | [`build-ceph-clangd-index.yml`](build-ceph-clangd-index.yml) |
| CDB 目录 | `/codebase/ceph/build` |

**架构要点**：RADOS（对象）→ RBD（块）/ RGW（对象网关）/ CephFS（文件）；OSD、MON、MGR 角色。

**建议阅读顺序**（只选一条子线深入）：
1. **RADOS / OSD**（最通用）：`src/osd/`、`src/objclass/`
2. **RGW**（对象存储岗）：`src/rgw/`
3. **CephFS**（文件存储岗）：`src/mds/`

**完成标准**：能画 Ceph 集群组件图，说明对象如何映射到 OSD、副本/纠删码如何工作。

**常见坑**：全仓极大；**不要**从 RGW 和 OSD 同时入手。先定子系统再读。

#### 路线 B：对象 + POSIX — MinIO + JuiceFS

| 项 | 说明 |
|----|------|
| MinIO | [minio/minio](https://github.com/minio/minio) — S3 兼容对象存储 |
| JuiceFS | [juicedata/juicefs](https://github.com/juicedata/juicefs) — 元数据 + 对象后端的 POSIX FS |
| 工具 | 均为 **Go**，用 **gopls**；无本目录 clangd workflow |

**建议阅读顺序**：
1. MinIO：S3 API 处理、纠删码/副本、对象布局（先文档 + 主路径，再源码）
2. JuiceFS：元数据引擎（Redis/TiKV 等）、与 MinIO/S3 如何配合挂载

**完成标准**：能说明「对象存储 vs POSIX 文件系统」差异，以及 JuiceFS 如何把对象后端暴露成文件路径。

**可选对照**：[SeaweedFS](https://github.com/seaweedfs/seaweedfs)（小文件/对象，Go）— 体量小于 Ceph，可作路线 B 的轻量替代阅读。

---

### 阶段 4：加分项（按兴趣与 JD，勿并行深啃）

完成阶段 1–3 后再选 **0–2 项** 扩展，避免「清单收集式」学习。

| 项目 | 语言 | 类型 | workflow | 何时选 |
|------|------|------|----------|--------|
| [TiKV](https://github.com/tikv/tikv) | Rust | 分布式 KV（RocksDB） | —（rust-analyzer） | JD 偏 NewSQL / 分布式 KV |
| [FoundationDB](https://github.com/apple/foundationdb) | C++ | 分布式事务 KV | —（构建链复杂，建议官方 Docker） | JD 偏分布式事务 |
| [GlusterFS](https://github.com/gluster/glusterfs) | C | 分布式文件系统 | `build-glusterfs-clangd-index.yml` | JD 偏 Gluster 生态 |
| [MooseFS](https://github.com/moosefs/moosefs) | C | 分布式文件系统 | `build-moosefs-clangd-index.yml` | 文件存储，体量小于 Ceph |
| [Apache Kudu](https://github.com/apache/kudu) | C++ | 列存 + 分布式表 | `build-kudu-clangd-index.yml` | JD 偏 HTAP / 列存 |
| [HDFS](https://github.com/apache/hadoop) | Java | 大数据文件系统 | — | 大数据存储岗 |
| [Lustre](https://github.com/lustre/lustre) | C | HPC 并行 FS | — | HPC / 超算存储岗 |
| [WiredTiger](https://github.com/wiredtiger/wiredtiger) | C | 文档库引擎 | — | 补 B 树 / 缓存页，对照 LSM |

---

## 三、6 个月学习计划（业余 2–3 小时/天）

面向「分布式存储研发」岗位的默认可执行节奏；若已有基础可压缩阶段 0–1。

| 月份 | 阶段 | 内容 | 产出 |
|------|------|------|------|
| 第 1 月 | 0 + 1.1 | 前置 + LevelDB 通读 | 能讲 LSM 骨架；本地配好 leveldb index |
| 第 2 月 | 1.2 | RocksDB 主路径 | 能讲 Compaction / WAL / CF；rocksdb index 可用 |
| 第 3 月 | 2 | braft + etcd 概念 | Raft 流程图；能关联元数据集群 |
| 第 4–5 月 | 3 | Ceph **或** MinIO+JuiceFS 选一 | 架构图 + 一条子系统源码笔记 |
| 第 6 月 | 4 | TiKV / Kudu / Gluster 等择一 | 与主线对比的面试话术 |

**每周建议节奏**：
- 3 天跟源码（配合 clangd index）
- 1 天画架构/数据流图
- 1 天对照论文或官方文档（Bigtable、Dynamo、Raft 等）

**论文加餐**（不必全读）：[rxin/db-readings](https://github.com/rxin/db-readings)、Google Bigtable、Amazon Dynamo、Raft 论文。

---

## 四、项目总表（按学习顺序索引）

完整清单，便于检索；**学习时请回到第二节按阶段推进**，不要按表格从上到下扫。

### 4.1 引擎层（阶段 1）

| 项目 | 语言 | 类型 | workflow | 优先级 |
|------|------|------|----------|--------|
| [LevelDB](https://github.com/google/leveldb) | C++ | LSM 入门 | `../build-leveldb-clangd-index.yml` | ★★★★★ |
| [RocksDB](https://github.com/facebook/rocksdb) | C++ | LSM 工业实现 | `../build-rocksdb-clangd-index.yml` | ★★★★★ |
| [WiredTiger](https://github.com/wiredtiger/wiredtiger) | C | B 树引擎 | — | ★★★ |

### 4.2 共识层（阶段 2）

| 项目 | 语言 | 类型 | workflow | 优先级 |
|------|------|------|----------|--------|
| [braft](https://github.com/baidu/braft) | C++ | Raft 库 | `build-braft-clangd-index.yml` | ★★★★ |
| [etcd](https://github.com/etcd-io/etcd) | Go | 协调服务 | —（gopls） | ★★★★ |
| [Apache Ratis](https://github.com/apache/ratis) | Java | Raft 实现 | — | ★★★ |

### 4.3 系统层（阶段 3）

| 项目 | 语言 | 类型 | workflow | 优先级 |
|------|------|------|----------|--------|
| [Ceph](https://github.com/ceph/ceph) | C++ | 对象/块/文件统一 | `build-ceph-clangd-index.yml` | ★★★★★ |
| [MinIO](https://github.com/minio/minio) | Go | S3 对象存储 | —（gopls） | ★★★★★ |
| [JuiceFS](https://github.com/juicedata/juicefs) | Go | POSIX + 对象后端 | —（gopls） | ★★★★ |
| [SeaweedFS](https://github.com/seaweedfs/seaweedfs) | Go | 小文件/对象 | —（gopls） | ★★★★ |
| [GlusterFS](https://github.com/gluster/glusterfs) | C | 分布式文件系统 | `build-glusterfs-clangd-index.yml` | ★★★ |
| [MooseFS](https://github.com/moosefs/moosefs) | C | 分布式文件系统 | `build-moosefs-clangd-index.yml` | ★★★ |
| [HDFS](https://github.com/apache/hadoop) | Java | 大数据 FS | — | ★★★ |
| [Lustre](https://github.com/lustre/lustre) | C | HPC 并行 FS | — | ★★ |

### 4.4 加分项（阶段 4）

| 项目 | 语言 | 类型 | workflow | 优先级 |
|------|------|------|----------|--------|
| [TiKV](https://github.com/tikv/tikv) | Rust | 分布式 KV | —（rust-analyzer） | ★★★★★ |
| [FoundationDB](https://github.com/apple/foundationdb) | C++ | 分布式事务 KV | — | ★★★★ |
| [Apache Kudu](https://github.com/apache/kudu) | C++ | 列存 + 分布式表 | `build-kudu-clangd-index.yml` | ★★★ |

---

## 五、本目录工作流一览

本目录内的 C/C++ 项目 clangd 预构建工作流；LevelDB/RocksDB 见上级目录。

| 项目 | 工作流 | 构建系统 | LVM | CDB 目录 | 默认 ref | 备注 |
|------|--------|----------|-----|----------|----------|------|
| ceph | `build-ceph-clangd-index.yml` | cmake + do_cmake.sh | 是 | `/codebase/ceph/build` | `reef` | 构建耗时长，子模块多 |
| braft | `build-braft-clangd-index.yml` | cmake（先编译安装 brpc） | 否 | `/codebase/braft/build` | `master` | brpc 是系统依赖，不是 git 子模块 |
| glusterfs | `build-glusterfs-clangd-index.yml` | autotools + bear | 否 | `/codebase/glusterfs` | `v11.2` | tag 是 `v11.2`，没有 `v11` |
| moosefs | `build-moosefs-clangd-index.yml` | autotools + bear | 否 | `/codebase/moosefs` | `master` | |
| kudu | `build-kudu-clangd-index.yml` | cmake | 是 | `/codebase/kudu/build` | `master` | |

**上级目录（引擎打底）**：

| 项目 | 工作流 | CDB 目录 |
|------|--------|----------|
| leveldb | `../build-leveldb-clangd-index.yml` | `/codebase/leveldb/build` |
| rocksdb | `../build-rocksdb-clangd-index.yml` | `/codebase/rocksdb/build` |

clangd 配置片段见 [`clangd/config-storage-projects.yaml`](clangd/config-storage-projects.yaml)（含 leveldb/rocksdb），可合并进 `~/.config/clangd/config.yaml`。

**本地消费步骤**（与上级 README 一致）：
1. 从 Release 下载 `*.idx`、`compile_commands.json`、源码快照（及 CMake 项目的 `*-generated-*.tar.gz`）
2. 解压到 `/codebase/<项目>`，索引重命名为 `<项目>.idx`
3. 重启 clangd

---

## 六、子目录 workflow 如何执行？

**可以，但有两个前提：**

1. **必须放到 `.github/workflows/` 下才会被 GitHub Actions 识别。**  
   本仓库的 `workflows/` 只是模板目录；使用时请复制到目标仓库，例如：
   ```bash
   mkdir -p .github/workflows/storage
   cp workflows/storage/build-ceph-clangd-index.yml .github/workflows/storage/
   ```
2. **`.github/workflows/` 的子目录是被支持的**（如 `.github/workflows/storage/build-ceph-clangd-index.yml`），在 Actions 页面会正常出现，可手动触发。

仅放在工作区 `workflows/storage/` 而未复制到 `.github/workflows/` 时，**不会自动执行**。

其余用法（Release 产物、`/codebase/<项目>` 路径约定、`index_format` 开关、排错）见上级 [`../README.md`](../README.md)。
