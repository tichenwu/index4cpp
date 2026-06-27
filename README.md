# clangd 索引预构建工作流（GitHub Actions）

借助 GitHub Actions 在云端编译大型 C/C++ 项目，**提前生成 clangd 索引**并上传到 Release，
本地下载后即可获得「秒级跳转 / 补全 / 查引用」的体验，无需在自己机器上完整构建这些动辄
几十 GB 的项目。

本目录包含：

- `_template.yml`：通用工作流模板，复制后改 4 处即可适配新项目。
- `build-<项目>-clangd-index.yml`：15 个已填好的项目工作流。
- `clangd/dot-clangd-template.yaml`：单项目 `.clangd` 模板。
- `clangd/config-all-projects.yaml`：一份覆盖全部 15 个项目的 clangd 全局用户配置。

---

## 一、原理说明

### 1. clangd 为什么需要索引

clangd 依赖两样东西才能精确解析 C/C++ 代码：

1. `**compile_commands.json`（编译数据库）**：记录每个源文件的完整编译命令（头文件搜索
  路径、宏定义、标准版本等）。没有它，clangd 只能猜测编译参数，跳转/补全会大面积失效。
2. **索引（index）**：对全工程符号做的预计算，用于「查找定义 / 引用 / 调用层级」等跨文件
  功能。clangd 默认会在本地**后台索引**，但对大项目可能耗时数十分钟并吃满 CPU。

本方案把「**完整构建 + 生成 `compile_commands.json` + 生成索引**」这件重活搬到 GitHub
Actions 的免费 runner 上完成，本地只需消费现成产物。

### 2. 编译数据库怎么来（按构建系统分类）

不同项目生成 `compile_commands.json` 的方式不同，这也是各工作流之间的主要差异：


| 构建系统                 | 生成方式                                       | 代表项目                                           |
| -------------------- | ------------------------------------------ | ---------------------------------------------- |
| Makefile / autotools | `bear -- make`（用 `bear` 拦截编译命令）            | nginx、redis、git、curl、cpython、ffmpeg、postgresql |
| Linux kbuild         | 先 `make` 再 `make compile_commands.json`    | linux                                          |
| CMake                | `-DCMAKE_EXPORT_COMPILE_COMMANDS=ON`（原生支持） | llvm、rocksdb、opencv、protobuf、leveldb、grpc      |
| Meson                | 构建目录原生输出 `compile_commands.json`           | qemu                                           |


> 对 CMake/meson 项目，`compile_commands.json` 在 **build 目录**，所以这些工作流的
> `CDB_DIR` 指向 `/codebase/<项目>/build`。

### 3. 为什么必须统一构建根目录 `/codebase/<项目>`

**clangd 索引里存的是绝对路径。** 如果云端在 `/home/runner/work/...` 构建，本地路径对不上，
索引就失效。因此所有工作流都强制把源码放到固定路径 `/codebase/<项目>` 下构建，
**本地也解压到同一路径**，路径完全对齐后跳转/补全/查引用全部可用。

- 最佳实践：**在 WSL / Linux / macOS 中使用**，本地路径与云端完全一致。
- 原生 Windows：见下文「Windows 注意事项」。

### 4. 两种索引格式：`index_format`

每个工作流都提供 `index_format` 开关：


| 取值           | 产物                                               | 配套用法                          | 说明                                                                          |
| ------------ | ------------------------------------------------ | ----------------------------- | --------------------------------------------------------------------------- |
| `static`（默认） | 单体 `<项目>.idx`                                    | `Index.External.File`         | clangd 稳定支持，开箱即用                                                            |
| `background` | `.cache/clangd/index` 分片（打包为 `*-bgindex.tar.gz`） | 解压到项目根的 `.cache/clangd/index` | 需较新 clangd（`clangd-indexer --format=background`），便于用 `clangd-remap` 跨平台重写路径 |


- 选 `static`：绝大多数场景，路径对齐即可直接用。
- 选 `background`：想在不同机器/不同根路径间搬运索引、用 `clangd-remap` 批量改写路径时更方便。

### 5. 大项目的磁盘扩容（LVM）

linux、llvm、ffmpeg、qemu、postgresql、opencv、protobuf、grpc、rocksdb 等构建产物很大，
GitHub runner 默认磁盘可能不够。这些工作流设置 `NEED_LVM: "true"`，开头会用 LVM 把
`/mnt` 和根分区的空闲空间合并挂到 `/codebase/<项目>`（做法源自 ophub README 5.4 节）。

---

## 二、参考来源

- **ophub/amlogic-s9xxx-armbian**：GitHub Actions 手动触发编译 + 上传 Release + LVM 扩容
的整体范式参考。[https://github.com/ophub/amlogic-s9xxx-armbian/blob/main/documents/README.cn.md](https://github.com/ophub/amlogic-s9xxx-armbian/blob/main/documents/README.cn.md)（1–5 节）
- **clangd 官方文档**
  - 编译数据库：[https://clangd.llvm.org/installation#compile_commandsjson](https://clangd.llvm.org/installation#compile_commandsjson)
  - 项目级 / 全局配置：[https://clangd.llvm.org/config](https://clangd.llvm.org/config)
  - 远程索引（`clangd-index-server`）：[https://clangd.llvm.org/design/remote-index](https://clangd.llvm.org/design/remote-index)
- **clangd-indexer / clangd_indexing_tools** 发行包：
[https://github.com/clangd/clangd/releases](https://github.com/clangd/clangd/releases)
- **Bear（Build EAR，拦截编译命令生成 compile_commands.json）**：
[https://github.com/rizsotto/Bear](https://github.com/rizsotto/Bear)
- **Linux 内核 `compile_commands.json`**：
[https://www.kernel.org/doc/html/latest/dev-tools/clangd.html](https://www.kernel.org/doc/html/latest/dev-tools/clangd.html)
- `**--format=background` 后台分片索引**：LLVM PR #175209。
- **CMake `CMAKE_EXPORT_COMPILE_COMMANDS`**：
[https://cmake.org/cmake/help/latest/variable/CMAKE_EXPORT_COMPILE_COMMANDS.html](https://cmake.org/cmake/help/latest/variable/CMAKE_EXPORT_COMPILE_COMMANDS.html)

---

## 三、使用说明

### 步骤 1：把工作流放进自己的仓库

1. 新建一个仓库（**public / private 均可**；private 同样有免费 Actions 额度，Release 仅自己可见）。
2. 把需要的 `build-<项目>-clangd-index.yml` 复制到仓库的 `.github/workflows/` 目录。
3. 工作流默认 `permissions: contents: write`，使用内置 `GITHUB_TOKEN` 即可创建 Release，
  无需额外配置密钥。

> 多个工作流**可以放在同一个仓库**，互不影响；也可按需拆分。

### 步骤 2：手动触发构建

在仓库 **Actions** 页面选择对应工作流，点击 **Run workflow**，按需填写：

- `ref`：要编译的 commit hash / tag / 分支（如 `v6.12`、`release-7.4`）。
- `clangd_version`：**务必与本地 clangd 主版本号一致**（默认 `19.1.2`）。
- `index_format`：`static`（默认）或 `background`。
- linux 额外参数：`arch`（如 `x86`/`arm64`）、`config_target`（`defconfig`/`allmodconfig`...）。

构建完成后，产物会发布到一个 Release，通常包含：

- `<项目>-<ref>.idx`（static）或 `<项目>-<ref>-bgindex.tar.gz`（background）
- `compile_commands.json`
- `<项目>-src-<ref>.tar.gz`（源码快照）
- `<项目>-generated-<ref>.tar.gz`（CMake/meson 的 build 目录，含生成的头文件，本地解析必需）

### 步骤 3：本地落盘到 `/codebase/<项目>`（推荐 WSL）

以 `static` 为例：

```bash
sudo mkdir -p /codebase/nginx && sudo chown -R "$USER:$USER" /codebase
cd /codebase/nginx

# 1) 源码快照（注意压缩包内带 nginx/ 前缀，按需调整 strip）
tar --strip-components=1 -xzf ~/Downloads/nginx-<ref>.tar.gz

# 2) CMake/meson 项目额外恢复 build/ 与生成头文件
#    tar -xzf ~/Downloads/<项目>-generated-<ref>.tar.gz   # 解压到 /codebase/<项目>

# 3) 索引改名为约定文件名
cp ~/Downloads/nginx-<ref>.idx /codebase/nginx/nginx.idx
```

`background` 格式则把分片解压到项目根：

```bash
cd /codebase/<项目>
tar -xzf ~/Downloads/<项目>-<ref>-bgindex.tar.gz   # 还原 .cache/clangd/index/
```

### 步骤 4：配置 clangd

二选一：

**A. 单项目** —— 用 `clangd/dot-clangd-template.yaml`，复制到项目根改名为 `.clangd`，
把 `<PROJECT>` 替换为项目短名；CMake/meson 记得把 `CompilationDatabase` 改为
`/codebase/<项目>/build`。

**B. 一份全局配置覆盖全部 15 个项目** —— 用 `clangd/config-all-projects.yaml`，放到：

- Linux / WSL：`~/.config/clangd/config.yaml`
- macOS：`~/Library/Preferences/clangd/config.yaml`
- Windows：`%LocalAppData%\clangd\config.yaml`

它用 `If.PathMatch` 为每个 `/codebase/<项目>` 目录单独生效，互不干扰。

### 步骤 5：VSCode 中使用

1. 安装 **clangd 扩展**（`llvm-vs-code-extensions.vscode-clangd`），禁用 Microsoft C/C++
  扩展的 IntelliSense 以免冲突。
2. 让 VSCode 的 clangd 版本与生成索引所用 `clangd_version` 主版本一致。
3. 打开 `/codebase/<项目>` 目录（WSL 用 Remote-WSL 打开），即可享受现成索引。

---

## Windows 注意事项

- **最稳方案：WSL。** 在 WSL 中把项目放到 `/codebase/<项目>`，路径与云端完全对齐，所有
功能开箱即用。
- **原生 Windows**：把配置里的 `/codebase/<项目>` 换成本地盘路径（如 `C:/codebase/<项目>`）。
补全 / 工程内搜索仍可用，但 `static` 索引内部存的是 Linux 绝对路径，**跳转到「索引内符号」
可能失效**。两种补救：
  1. 用 `clangd-index-server`（`--project-root=/codebase/<项目>`）+ 同名 `MountPoint` 做路径映射；
  2. 改用 `background` 格式 + `clangd-remap` 把分片里的路径批量重写为 Windows 路径。
- `clangd.arguments: --path-mappings=...` 是给 **client/server 远程场景**做 URI 映射的，
**不能**把 Windows 上不存在的 Linux 路径「翻译」成可读路径，别用它来解决本地索引路径问题。

---

## 常见报错与排错

以下问题已在工作流文件中修复，这里说明现象与原因，便于自行适配新项目时避坑。

### 1. 下载 clangd 工具链失败 / 「地址无效」

- **现象**：`Download clangd indexing tools` 步骤报下载失败，或后续 `cp` 报
  `No such file or directory`。
- **原因**：① `wget -q` 会把 HTTP 错误静默吞掉；② 解压目录名不能写死——clangd 压缩包
  内部顶层目录命名并不规律，硬拼 `clangd_indexing_tools-<ver>/bin/` 容易对不上。
- **修复**：用 `curl -fL`（HTTP 错误显式失败并打印 URL、自动跟随重定向与重试）下载，
  解压后用 `find ... -name clangd-indexer` **动态定位**二进制，不依赖目录名。

### 2. 生成索引报 `fatal error: 'stddef.h' file not found`

- **现象**：`Generate index` 步骤里几乎每个文件都在 `sys/types.h` 处报找不到 `stddef.h`。
- **原因**：`stddef.h`、`stdarg.h` 等是**编译器自带头文件**，随 clangd 包一起分发，
  clangd-indexer 按「**二进制路径 `/../lib/clang/<版本>/include`**」相对查找
  （见 clangd Discussion #970）。若只把 `bin/*` 拷到 `/usr/local/bin/` 而漏掉 `lib/`，
  就会找不到这些头文件。
- **修复**：**整体复制工具链**、保留 `bin/` 与 `lib/` 的相对结构：
  `sudo cp -rf "${toolroot}/." /usr/local/`。

### 3. 发布 Release 报 `failed to run git: fatal: not a git repository`

- **现象**：`Package + publish to Release` 步骤最后 `gh release create` 失败，报 not a git
  repository（信息可能打印两遍）。
- **原因**：这条信息的 `failed to run git:` 前缀来自 **`gh` CLI**（不是 `git archive`）。
  本工作流为省时间**不执行 `actions/checkout`**，步骤默认工作目录不是 git 仓库，
  `gh` 没有 `--repo` 时会去当前目录读 git remote 推断目标仓库，于是失败。
- **修复**：在 Package 步骤的 `env` 里加 `GH_REPO: ${{ github.repository }}`（`gh` 官方
  识别的环境变量，格式 `owner/repo`），不再依赖本地 git 上下文。

---

## 项目一览


| 项目         | 工作流                                 | 构建系统             | LVM 扩容 | CDB 目录                         |
| ---------- | ----------------------------------- | ---------------- | ------ | ------------------------------ |
| nginx      | `build-nginx-clangd-index.yml`      | autotools + bear | 否      | `/codebase/nginx`              |
| linux      | `build-linux-clangd-index.yml`      | kbuild           | 是      | `/codebase/linux`              |
| llvm       | `build-llvm-clangd-index.yml`       | CMake            | 是      | `/codebase/llvm-project/build` |
| redis      | `build-redis-clangd-index.yml`      | make + bear      | 否      | `/codebase/redis`              |
| git        | `build-git-clangd-index.yml`        | make + bear      | 否      | `/codebase/git`                |
| postgresql | `build-postgresql-clangd-index.yml` | autotools + bear | 是      | `/codebase/postgresql`         |
| cpython    | `build-cpython-clangd-index.yml`    | autotools + bear | 否      | `/codebase/cpython`            |
| ffmpeg     | `build-ffmpeg-clangd-index.yml`     | configure + bear | 是      | `/codebase/ffmpeg`             |
| curl       | `build-curl-clangd-index.yml`       | autotools + bear | 否      | `/codebase/curl`               |
| qemu       | `build-qemu-clangd-index.yml`       | Meson            | 是      | `/codebase/qemu/build`         |
| rocksdb    | `build-rocksdb-clangd-index.yml`    | CMake            | 是      | `/codebase/rocksdb/build`      |
| opencv     | `build-opencv-clangd-index.yml`     | CMake            | 是      | `/codebase/opencv/build`       |
| protobuf   | `build-protobuf-clangd-index.yml`   | CMake + 子模块      | 是      | `/codebase/protobuf/build`     |
| leveldb    | `build-leveldb-clangd-index.yml`    | CMake + 子模块      | 否      | `/codebase/leveldb/build`      |
| grpc       | `build-grpc-clangd-index.yml`       | CMake + 子模块      | 是      | `/codebase/grpc/build`         |


---

## 适配新项目（用 `_template.yml`）

复制 `_template.yml` 为 `build-<项目>-clangd-index.yml`，修改 4 处：

1. `env.PROJECT`：项目短名。
2. `env.SRC_ROOT`：`/codebase/<项目>`。
3. `env.CDB_DIR`：Makefile/autotools → 同 `SRC_ROOT`；CMake/meson → `SRC_ROOT/build`。
4. `env.NEED_LVM`：大项目设 `"true"`。

再补全标注 `>>> ... <<<` 的「依赖安装 / 克隆 / 构建」三段命令即可。

> **Go 项目无需本方案**：`gopls` 会自动、高效地索引，VSCode Go 扩展开箱即用，
> 不需要预生成 clangd 式索引。

