# Clangd index compilation and configuration
_Exported on 6/27/2026 at 12:02:46 GMT+8 from Cursor (3.6.31)_

---

**User**

阅读 https://github.com/ophub/amlogic-s9xxx-armbian/blob/main/documents/README.cn.md  1-5节，想要借助该功能，fork nginx仓库代码，实现手动编译nginx源码的clangd-index数据，编译时可以指定需要编译的hash版本，上传到release，便于下载后作为本地clangd索引的功能，给出详细的操作步骤；并给出windows本地vscode配置clangd的方法及可用的clangd配置

---

**Cursor**

先阅读你上传的文档和目标仓库的 README，了解其 GitHub Actions 的工作方式。

我已读完 README 1–5 节，理解了它的核心套路：**Fork 仓库 → 配 workflow 权限 → Actions 里手动 `Run workflow`（可填参数）→ 在 runner 上编译 → 用 `upload-to-release` 把产物传到 Release → 本地下载使用**。

在为 nginx 设计 clangd 索引方案前，我需要先确认一个关键技术点：clangd 索引的可移植性（尤其 Linux 构建、Windows 使用时的路径问题）。让我快速核实。

关键技术点已核实，但我需要确认 clangd 配置里 `Index.External` + `MountPoint` 的语义（这是解决「Linux 构建、Windows 使用」路径不匹配的核心）。

我已读完 README 1–5 节并核实了 clangd 索引的关键技术细节。下面给你一套完整可落地的方案。

先说结论：ophub 的套路本质是「**Fork 仓库 → 给 Actions 写权限 → `workflow_dispatch` 手动触发并填参数 → runner 上编译 → 把产物 `upload-to-release` → 本地下载**」。我们把"编译 Armbian 镜像"换成"编译 nginx 源码 + 生成 clangd 索引"即可，参数从"设备型号"换成"nginx 的 hash/版本"。

但 clangd 索引有一个**绕不开的前提**，必须先讲清楚，否则下载下来用不了。

---

## 0. 关键前提：clangd 索引里存的是绝对路径

经核实（clangd 官方设计文档 + 社区 issue）：

- clangd 的两种索引（`.cache/clangd/index` 后台分片、`clangd-indexer` 生成的静态 `.idx`）**内部都存源码文件的绝对路径**（`file://` URI）。
- 想跨机器/跨平台直接用，路径必须能对得上。`Index.External.File` 模式下的 `MountPoint` **只用于过滤"索引对哪些文件生效"，并不会改写索引里已存的路径**（见 clangd issue #2100）。真正能"去掉构建机前缀、换成本地前缀"的只有 `clangd-index-server --project-root` + `MountPoint` 这条路。
- 还有个新工具 `clangd-remap`（2025/2026 才合入 llvm-project），能离线重写后台分片里的路径，但稳定版 clangd 里基本还没有。

而 nginx 是 **Unix 代码**（大量 `src/os/unix`、`unistd.h` 等），在原生 Windows 上让 clangd 去解析源码本身就会满屏报错。所以：

> **强烈建议在 Windows 上用 WSL 来消费这个索引**：把源码放到与构建时一致的绝对路径，路径完全对得上，跳转/补全/查引用全部可用，零路径映射烦恼。
> 原生 Windows VSCode 也能用，但主要适合"全局符号搜索/补全"，本地解析 unix 专属文件会不完美。两种配置我都给你。

为此，workflow 里我把构建根目录**固定为 `/opt/ngx`**，并把源码快照一并打进 Release，方便你本地放到同一路径做映射。

---

## 1. 注册账号 / 准备仓库（对应 README 第 1、3 节）

1. 有 GitHub 账号即可（无需额外 secret，`GITHUB_TOKEN` 由 Actions 自动注入，对应 README 第 2 节）。
2. 你不需要 fork nginx 本身。**新建一个你自己的仓库**（例如 `nginx-clangd-index`）用来放 workflow——nginx 源码在 Actions 里实时 `git clone`，这样可以任意指定 hash。
   - 如果你坚持要"fork"，fork `nginx/nginx` 也行，但把下面的 workflow 放进 fork 后的 `.github/workflows/` 即可。
3. 进入该仓库 → `Settings` → `Actions` → `General` → `Workflow permissions` → 勾选 **Read and write permissions** 并保存（对应 README 第 3 节，这是能创建 Release 的前提）。

---

## 2. 添加 workflow 文件（对应 README 第 4、5、6 节）

在仓库新建 `.github/workflows/build-nginx-clangd-index.yml`：

```yaml
name: Build nginx clangd index

on:
  workflow_dispatch:
    inputs:
      nginx_ref:
        description: "要编译的 nginx 版本：commit hash / tag / 分支（如 release-1.27.4 或 40 位 hash）"
        required: true
        default: "master"
      configure_args:
        description: "传给 nginx ./auto/configure 的额外参数（决定索引覆盖哪些模块）"
        required: false
        default: "--with-http_ssl_module --with-http_v2_module --with-stream --with-threads --with-file-aio"
      clangd_version:
        description: "clangd / indexing tools 版本（务必与你本地 clangd 主版本一致）"
        required: false
        default: "19.1.2"

permissions:
  contents: write          # 允许创建 Release、上传产物（对应 README 第 6 节）

env:
  NGX_ROOT: /opt/ngx       # 固定构建根目录，本地映射时作为 project-root / MountPoint

jobs:
  build-index:
    runs-on: ubuntu-22.04
    steps:
      - name: Install build deps
        run: |
          sudo apt-get update
          sudo apt-get install -y --no-install-recommends \
            build-essential git wget unzip ca-certificates \
            libpcre2-dev zlib1g-dev libssl-dev bear

      - name: Download clangd indexing tools (clangd-indexer / clangd-index-server)
        run: |
          ver="${{ inputs.clangd_version }}"
          cd /tmp
          wget -q "https://github.com/clangd/clangd/releases/download/${ver}/clangd_indexing_tools-linux-${ver}.zip"
          unzip -q "clangd_indexing_tools-linux-${ver}.zip"
          sudo cp -f "clangd_indexing_tools-${ver}/bin/"* /usr/local/bin/
          clangd-indexer --version || true

      - name: Clone nginx @ ${{ inputs.nginx_ref }}
        run: |
          sudo mkdir -p "${NGX_ROOT}"
          sudo chown -R "$USER:$USER" "${NGX_ROOT}"
          git clone https://github.com/nginx/nginx.git "${NGX_ROOT}"
          cd "${NGX_ROOT}"
          git checkout --force "${{ inputs.nginx_ref }}"
          echo "RESOLVED_SHA=$(git rev-parse --short HEAD)" >> "${GITHUB_ENV}"

      - name: Configure & build (capture compile_commands.json via bear)
        run: |
          cd "${NGX_ROOT}"
          ./auto/configure ${{ inputs.configure_args }}
          bear -- make -j"$(nproc)"
          test -s compile_commands.json   # 为空就说明没真正调用编译器

      - name: Generate clangd static index
        run: |
          cd "${NGX_ROOT}"
          clangd-indexer --executor=all-TUs compile_commands.json > /tmp/nginx.idx
          ls -lh /tmp/nginx.idx

      - name: Package artifacts
        run: |
          out=/tmp/out; mkdir -p "${out}"
          ref="${{ inputs.nginx_ref }}"; safe="${ref//\//_}"
          cp /tmp/nginx.idx                      "${out}/nginx-${safe}.idx"
          cp "${NGX_ROOT}/compile_commands.json" "${out}/compile_commands.json"
          # 源码快照：本地解压到 ${NGX_ROOT} 即可与索引路径完全对齐
          git -C "${NGX_ROOT}" archive --format=tar.gz --prefix="ngx/" \
              -o "${out}/nginx-src-${safe}.tar.gz" HEAD
          # 现成 .clangd 模板
          cat > "${out}/dot-clangd-example.yaml" <<'EOF'
          CompileFlags:
            CompilationDatabase: /opt/ngx
          Index:
            External:
              File: /opt/ngx/nginx.idx
              MountPoint: /opt/ngx
          EOF
          echo "BUILD_SAFE_REF=${safe}" >> "${GITHUB_ENV}"
          ls -lh "${out}"

      - name: Publish to Release（对应 README 第 6 节 upload-to-release）
        env:
          GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        run: |
          tag="nginx-${BUILD_SAFE_REF}"
          gh release create "${tag}" /tmp/out/* \
            --title "nginx clangd index (${{ inputs.nginx_ref }} @ ${RESOLVED_SHA})" \
            --notes "clangd 静态索引 for nginx ${{ inputs.nginx_ref }}。构建 project-root=${NGX_ROOT}，clangd 工具版本=${{ inputs.clangd_version }}。" \
          || gh release upload "${tag}" /tmp/out/* --clobber
```

要点说明：
- **指定 hash**：触发时填 `nginx_ref`，可以是 40 位 commit hash、tag（如 `release-1.27.4`）或分支名，`git checkout` 直接切过去——这就是你要的"编译时指定 hash 版本"。
- **为什么用 `bear`**：nginx 用 `make`（不是 cmake），没有原生的 `compile_commands.json`。`bear -- make` 拦截编译命令生成它（核实过：configure 阶段只做特性探测、不属于工程文件，所以只需对 `make` 用 bear；且要从干净状态构建，否则数据库为空）。
- **为什么主打静态 `.idx`**：`clangd-indexer` 一条命令即可在无界面环境批量生成整个工程索引，比"跑后台索引再抓 `.cache`"稳得多。
- **clangd 版本要对齐**：索引格式与 clangd 主版本绑定，`clangd_version` 必须和你本地 VSCode 用的 clangd 主版本一致，否则可能加载失败。

---

## 3. 手动触发编译（对应 README 第 5.1 节）

仓库顶部 `Actions` → 左侧选 `Build nginx clangd index` → 右侧 `Run workflow` → 填入：
- `nginx_ref`：例如 `release-1.27.4`
- `configure_args`：按需调整（决定索引覆盖哪些模块）
- `clangd_version`：与本地一致

→ 点 `Run workflow` 开始。几分钟后在仓库右侧 **Releases** 里会出现 `nginx-release-1.27.4` 这个 Release，含：
- `nginx-<ref>.idx`（核心索引）
- `compile_commands.json`
- `nginx-src-<ref>.tar.gz`（源码快照）
- `dot-clangd-example.yaml`（配置模板）

> 想定时编译（对应 README 第 5.2 节）可加 `schedule: - cron: '0 17 * * *'`，但通常索引按需触发即可。

---

## 4. 本地下载

从 Release 页面下载 `.idx`、`compile_commands.json` 和源码快照三件套。

---

## 5. Windows 本地 VSCode 配置 clangd

### 5.1 安装（两种环境通用）
1. VSCode 装扩展 **clangd**（`llvm-vs-code-extensions.vscode-clangd`）。
2. **禁用** 微软 C/C++ 扩展的 IntelliSense（否则和 clangd 抢，二选一）：设置里 `C_Cpp.intelliSenseEngine` 设为 `disabled`。
3. clangd 二进制：让 clangd 扩展自动下载，或手动装 LLVM。**主版本要和 workflow 里的 `clangd_version` 一致。**

### 5.2 方式 A（强烈推荐）：在 WSL 里用，路径完全对齐

这是 nginx 这种 Unix 工程体验最好的方式，跳转/补全/查引用全部正常：

```bash
# 在 WSL 中
sudo mkdir -p /opt/ngx && sudo chown -R "$USER:$USER" /opt/ngx
tar -xzf nginx-src-release-1.27.4.tar.gz -C /tmp
cp -r /tmp/ngx/* /opt/ngx/          # 源码放到与构建时相同的 /opt/ngx
cp nginx-release-1.27.4.idx /opt/ngx/nginx.idx
cp compile_commands.json /opt/ngx/
```

在 `/opt/ngx/.clangd` 写入（路径与索引完全一致，无需任何映射）：

```yaml
CompileFlags:
  CompilationDatabase: /opt/ngx
Index:
  External:
    File: /opt/ngx/nginx.idx
    MountPoint: /opt/ngx
  Background: Skip          # 已有完整静态索引，关掉后台索引省 CPU
```

VSCode 用 **WSL 远程** 打开 `/opt/ngx` 即可。

### 5.3 方式 B：原生 Windows + 本地 index-server（导航可用）

原生 Windows 路径和索引里的 Linux 路径对不上，用 `clangd-index-server` 做映射（**纯本地、无需联网**，监听 `127.0.0.1:50051`）：

```powershell
# 从 Release 的 indexing tools 里拿 clangd-index-server.exe（Windows 版同版本）
# --project-root 去掉构建机前缀 /opt/ngx
clangd-index-server.exe C:\clangd\nginx-release-1.27.4.idx --project-root=/opt/ngx
```

把源码快照解压到本地，例如 `C:\src\nginx`，然后在**用户配置** `%LocalAppData%\clangd\config.yaml` 写：

```yaml
If:
  PathMatch: C:/src/nginx/.*
Index:
  External:
    Server: localhost:50051
    MountPoint: C:/src/nginx/
```

`--project-root` 剥离 `/opt/ngx`、`MountPoint` 补上 `C:/src/nginx` → 跳转能正确落到本地文件。
注意：remote-index 需要 clangd 编译时开启该能力；用 clangd 官方 release 自带的 indexing tools 通常一致，若 `Server` 不生效就改用方式 A 或 C。

### 5.4 方式 C：原生 Windows + 直接加载 `.idx`（最简单，够用）

只想要**全局符号补全 / 查找引用**、能接受"跳到索引内符号时可能指向 Linux 路径"的话，最省事。把源码解压到 `C:\src\nginx`，`C:\src\nginx\.clangd`：

```yaml
Index:
  External:
    File: C:/clangd/nginx-release-1.27.4.idx
    MountPoint: C:/src/nginx/
  Background: Skip
```

实际体验：你**本地打开的文件**由 clangd 现场解析，跳转准确；静态索引主要补全全局符号和"查引用"。对索引内、本地没打开的符号做跳转时，可能落到 `/opt/ngx/...` 而失效。补全/悬停/符号搜索正常。
（进阶：若用到了带 `clangd-remap` 的新版工具，可 `clangd-remap --path-mappings=/opt/ngx=C:/src/nginx C:\path\to\.cache\clangd\index` 重写后原生可用，但需后台分片格式而非静态 idx，且工具较新。）

---

## 小结与建议

| 环境 | 推荐方式 | 导航 | 补全/查引用 | 复杂度 |
|---|---|---|---|---|
| Windows + WSL | 方式 A（路径对齐） | ✅完全可用 | ✅ | 低 |
| 原生 Windows | 方式 B（本地 index-server） | ✅基本可用 | ✅ | 中 |
| 原生 Windows | 方式 C（直接 File） | ⚠️部分 | ✅ | 最低 |

- 整体照搬了 ophub 的「fork→权限→`workflow_dispatch` 填参→编译→`gh release` 上传→下载」流程，把参数换成 nginx 的 hash/版本。
- nginx 是 Unix 工程，**首选 WSL（方式 A）**，体验最完整且无路径烦恼。
- 务必让 **workflow 的 `clangd_version` 与本地 clangd 主版本一致**。
- 上面的 workflow 和配置都是给你贴到 GitHub 仓库 / 本地用的，我没有改动你当前工作区的文件。需要的话我可以直接在你工作区里生成这个 yml 和配置模板文件。

需要我把 workflow 文件和 `.clangd` 模板直接写到你本地某个目录里吗？

---

**User**

不fork nginx，创建自己的仓库的方案，仓库必须是public属性的吗？private行不行

---

**Cursor**

完全可以用 **private（私有）仓库**，不强制 public。两者功能上都能跑通这套流程，区别在于额度和下载方式：

## 能不能用 private：能

| 维度 | public 公开仓库 | private 私有仓库 |
|---|---|---|
| GitHub Actions | **免费、无时长配额**（单次最长 6h，README 里说的就是这个） | 免费额度有限：GitHub Free 每月 **2000 分钟** + **500MB** 存储（超了要付费或等下月重置） |
| `workflow_dispatch` 手动触发 | ✅ | ✅ |
| 创建 Release / 上传产物 | ✅ | ✅ |
| 下载 Release 产物 | 任何人**匿名**直接下 | **必须带 token 鉴权**才能下 |
| `GITHUB_TOKEN` 写权限 | ✅ | ✅ |

结论：**private 行**，nginx 索引这种构建（几分钟一次）远用不完 2000 分钟。唯一要注意的是**下载产物需要鉴权**。

## private 仓库下载产物的两种方式

公开仓库可以直接 `wget <release-url>`，私有仓库不行，需要带 token。生成一个 **PAT（Personal Access Token）**，勾选 `repo` 权限（或 fine-grained token 给该仓库 `Contents: Read`）。

方式一：用 `gh` CLI（最简单，自动鉴权）
```bash
gh auth login                       # 登录一次
gh release download nginx-release-1.27.4 \
  --repo your-name/nginx-clangd-index \
  --dir ./download
```

方式二：用 API + token（需要拿 asset 的内部 id，不能直接用浏览器链接）
```bash
# 列出资产 id
curl -s -H "Authorization: Bearer $GH_TOKEN" \
  "https://api.github.com/repos/your-name/nginx-clangd-index/releases/tags/nginx-release-1.27.4"

# 按 asset id 下载（注意 Accept 头）
curl -L -H "Authorization: Bearer $GH_TOKEN" \
  -H "Accept: application/octet-stream" \
  "https://api.github.com/repos/your-name/nginx-clangd-index/releases/assets/<ASSET_ID>" \
  -o nginx.idx
```

## 给你的建议

- **个人自用、不在意是否公开** → 用 **public** 最省心：Actions 不限时长、下载零鉴权，配置最简单。nginx 源码本身就是开源的，索引产物也没有敏感信息。
- **不想暴露仓库 / 公司内部用** → 用 **private** 完全可以，只需多一步 `gh release download` 鉴权下载即可，其余 workflow 一字不改。

你之前那份 workflow 在 public / private 下都能直接用，不需要改动。需要我把 `private 自动鉴权下载` 也写成一个小脚本给你吗？

---

**User**

针对linux源码，也按照相同的需求生成对应的脚本和说明

---

**Cursor**

思路和 nginx 完全一致（Fork/建仓 → Actions 写权限 → `workflow_dispatch` 填 hash → 构建生成 `compile_commands.json` → `clangd-indexer` 出 `.idx` → 上传 Release → 本地下载用）。但 Linux 内核有几处关键差异必须针对性处理：

1. **不需要 bear**：内核自带生成器，构建后 `make compile_commands.json` 即可（依赖编译时产生的 `.cmd` 文件，所以**必须先 build**）。
2. **arch + config 决定索引覆盖面**：`compile_commands.json` 只含被编译的 TU。默认用 `defconfig`，想全覆盖用 `allmodconfig`（体积大很多）。
3. **盘可能不够**：内核源码+构建产物+索引可达数 GB～十几 GB。`allmodconfig` 时要用 README 5.4 的 LVM 扩容技巧，我已内置该步骤。
4. **用 clang 构建（`LLVM=1`）**：让 `compile_commands.json` 里的编译参数与 clangd 解析一致，减少误报。
5. **本地解析需要生成头文件**（`autoconf.h`、`include/generated/*` 等只在 configure/build 后才有），所以产物里我额外打包了这些生成文件，本地无需重建即可解析。
6. 内核是纯 Unix 代码，**强烈建议 WSL 消费**（路径对齐，体验最好），原理同 nginx 那答里讲过的索引绝对路径问题。

---

## 1. 准备仓库（同 nginx）

新建一个仓库（如 `linux-clangd-index`，public/private 均可，private 下载产物需 `gh release download` 鉴权）→ `Settings` → `Actions` → `General` → 勾选 **Read and write permissions**。

---

## 2. workflow 文件

新建 `.github/workflows/build-linux-clangd-index.yml`：

```yaml
name: Build linux-kernel clangd index

on:
  workflow_dispatch:
    inputs:
      linux_ref:
        description: "要编译的内核版本：tag / commit hash / 分支（如 v6.12 或 40 位 hash）"
        required: true
        default: "v6.12"
      arch:
        description: "目标架构 ARCH（x86 / arm64 / riscv ...）"
        required: false
        default: "x86"
      config_target:
        description: "内核配置目标（defconfig / allmodconfig / tinyconfig ...）。allmodconfig 覆盖最全但很大"
        required: false
        default: "defconfig"
      clangd_version:
        description: "clangd / indexing tools 版本（务必与本地 clangd 主版本一致）"
        required: false
        default: "19.1.2"

permissions:
  contents: write

env:
  LINUX_ROOT: /opt/linux        # 固定构建根目录，本地映射时作为 project-root / MountPoint

jobs:
  build-index:
    runs-on: ubuntu-22.04
    steps:
      - name: Expand build disk (LVM，对应 README 5.4，allmodconfig 必需)
        run: |
          mnt_size=$(expr $(df -h /mnt | tail -1 | awk '{print $4}' | sed 's/[[:alpha:]]//g;s/\..*//') - 1)
          root_size=$(expr $(df -h /   | tail -1 | awk '{print $4}' | sed 's/[[:alpha:]]//g;s/\..*//') - 4)
          sudo truncate -s "${mnt_size}"G /mnt/mnt.img
          sudo truncate -s "${root_size}"G /root.img
          sudo losetup /dev/loop6 /mnt/mnt.img
          sudo losetup /dev/loop7 /root.img
          sudo pvcreate /dev/loop6 /dev/loop7
          sudo vgcreate github /dev/loop6 /dev/loop7
          sudo lvcreate -n runner -l 100%FREE github
          sudo mkfs.xfs /dev/github/runner
          sudo mkdir -p "${LINUX_ROOT}"
          sudo mount /dev/github/runner "${LINUX_ROOT}"
          sudo chown -R "$USER:$USER" "${LINUX_ROOT}"
          df -Th

      - name: Install kernel build deps + clang
        run: |
          sudo apt-get update
          sudo apt-get install -y --no-install-recommends \
            git wget unzip ca-certificates \
            build-essential bc bison flex libelf-dev libssl-dev libncurses-dev \
            clang lld llvm dwarves python3

      - name: Download clangd indexing tools
        run: |
          ver="${{ inputs.clangd_version }}"
          cd /tmp
          wget -q "https://github.com/clangd/clangd/releases/download/${ver}/clangd_indexing_tools-linux-${ver}.zip"
          unzip -q "clangd_indexing_tools-linux-${ver}.zip"
          sudo cp -f "clangd_indexing_tools-${ver}/bin/"* /usr/local/bin/
          clangd-indexer --version || true

      - name: Clone linux @ ${{ inputs.linux_ref }}
        run: |
          # 浅克隆指定 ref，省时省盘
          git clone --depth 1 --branch "${{ inputs.linux_ref }}" \
            https://github.com/torvalds/linux.git "${LINUX_ROOT}" 2>/dev/null \
          || { git clone --filter=blob:none https://github.com/torvalds/linux.git "${LINUX_ROOT}" \
               && git -C "${LINUX_ROOT}" checkout --force "${{ inputs.linux_ref }}"; }
          echo "RESOLVED_SHA=$(git -C "${LINUX_ROOT}" rev-parse --short HEAD)" >> "${GITHUB_ENV}"

      - name: Configure & build with clang (生成 .cmd 文件)
        run: |
          cd "${LINUX_ROOT}"
          export ARCH="${{ inputs.arch }}" LLVM=1
          make ${config_target}
          # 关掉 BTF 以免 pahole 版本问题导致构建失败
          ./scripts/config --disable DEBUG_INFO_BTF || true
          make olddefconfig
          make -j"$(nproc)"
        env:
          config_target: ${{ inputs.config_target }}

      - name: Generate compile_commands.json + 静态索引
        run: |
          cd "${LINUX_ROOT}"
          make ARCH="${{ inputs.arch }}" LLVM=1 compile_commands.json
          test -s compile_commands.json
          clangd-indexer --executor=all-TUs compile_commands.json > /tmp/linux.idx
          ls -lh compile_commands.json /tmp/linux.idx

      - name: Package artifacts
        run: |
          out=/tmp/out; mkdir -p "${out}"
          ref="${{ inputs.linux_ref }}"; arch="${{ inputs.arch }}"; safe="${ref//\//_}"
          cp /tmp/linux.idx                        "${out}/linux-${safe}-${arch}.idx"
          cp "${LINUX_ROOT}/compile_commands.json" "${out}/compile_commands.json"
          # 源码快照
          git -C "${LINUX_ROOT}" archive --format=tar.gz --prefix="linux/" \
              -o "${out}/linux-src-${safe}.tar.gz" HEAD
          # 生成头文件快照：本地解析必需（autoconf.h / include/generated 等）
          cd "${LINUX_ROOT}"
          tar -czf "${out}/linux-generated-${safe}-${arch}.tar.gz" \
              .config include/generated include/config \
              "arch/${arch}/include/generated" 2>/dev/null || true
          cat > "${out}/dot-clangd-example.yaml" <<'EOF'
          CompileFlags:
            CompilationDatabase: /opt/linux
          Index:
            External:
              File: /opt/linux/linux.idx
              MountPoint: /opt/linux
            Background: Skip
          EOF
          echo "BUILD_SAFE_REF=${safe}" >> "${GITHUB_ENV}"
          echo "BUILD_ARCH=${arch}" >> "${GITHUB_ENV}"
          ls -lh "${out}"

      - name: Publish to Release
        env:
          GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        run: |
          tag="linux-${BUILD_SAFE_REF}-${BUILD_ARCH}"
          gh release create "${tag}" /tmp/out/* \
            --title "linux clangd index (${{ inputs.linux_ref }} / ${{ inputs.arch }} @ ${RESOLVED_SHA})" \
            --notes "clangd 静态索引 for linux ${{ inputs.linux_ref }} (ARCH=${{ inputs.arch }}, ${{ inputs.config_target }})。构建 project-root=${LINUX_ROOT}，clangd 工具版本=${{ inputs.clangd_version }}。" \
          || gh release upload "${tag}" /tmp/out/* --clobber
```

关键说明：
- **必须先 `make` 再 `make compile_commands.json`**：生成器靠每个 `.o` 旁边的 `.cmd` 文件还原编译命令，没 build 就是空数据库。
- **`LLVM=1` 用 clang 构建**：和 clangd 解析器对齐，避免 gcc 专属 flag 报错。
- **`--disable DEBUG_INFO_BTF`**：defconfig 默认开 BTF，需要特定 pahole 版本，关掉更稳。
- **`allmodconfig` 才需要 LVM 扩容**；`defconfig` 在默认盘也够，但保留该步骤无害。
- 多了一个 **`linux-generated-*.tar.gz`**：本地解析内核源码离不开这些生成头文件。

---

## 3. 手动触发

`Actions` → `Build linux-kernel clangd index` → `Run workflow`，填：
- `linux_ref`：如 `v6.12`
- `arch`：如 `x86` / `arm64`
- `config_target`：`defconfig`（快）或 `allmodconfig`（全但慢且大）

完成后 Release `linux-v6.12-x86` 含：`.idx`、`compile_commands.json`、源码快照、生成头文件快照、配置模板。

---

## 4. 下载（同 nginx）

- public 仓库：直接下载。
- private 仓库：`gh release download linux-v6.12-x86 --repo your-name/linux-clangd-index --dir ./download`。

---

## 5. 本地 VSCode + clangd 配置

### 推荐：WSL 中路径对齐使用（导航/补全全可用）

```bash
sudo mkdir -p /opt/linux && sudo chown -R "$USER:$USER" /opt/linux
tar -xzf linux-src-v6.12.tar.gz -C /tmp && cp -r /tmp/linux/* /opt/linux/
# 关键：解压生成头文件，否则解析报错（缺 autoconf.h 等）
tar -xzf linux-generated-v6.12-x86.tar.gz -C /opt/linux/
cp linux-v6.12-x86.idx /opt/linux/linux.idx
cp compile_commands.json /opt/linux/
```

`/opt/linux/.clangd`（路径与索引一致，无需映射）：

```yaml
CompileFlags:
  CompilationDatabase: /opt/linux
Index:
  External:
    File: /opt/linux/linux.idx
    MountPoint: /opt/linux
  Background: Skip
```

VSCode 用 WSL 远程打开 `/opt/linux` 即可。

### 原生 Windows

和 nginx 那答的方式 B / C 完全一样：
- **方式 B（导航可用）**：本机跑 `clangd-index-server.exe linux-v6.12-x86.idx --project-root=/opt/linux`，用户配置 `Index.External.Server: localhost:50051` + `MountPoint: C:/src/linux/`。
- **方式 C（最简单）**：`.clangd` 直接 `Index.External.File` 指向 `.idx`，全局补全/查引用可用，跳转到索引内符号可能落到 Linux 路径而失效。

> 内核比 nginx 更"纯 Unix"，原生 Windows 上本地解析报错会更多，**首选 WSL**。

---

需要我把 nginx + Linux 两个 workflow 一并写进你工作区（比如 `/home/tichen/cursor` 下的某个目录）方便你直接推到 GitHub 吗？

---

**User**

相同的需求，适配到llvm项目

---

**Cursor**

同样的套路，LLVM 用 **CMake + Ninja**，比 nginx/内核更省事——`compile_commands.json` 由 CMake 原生生成（`-DCMAKE_EXPORT_COMPILE_COMMANDS=ON`），不用 bear、不用内核那套生成器。但有 LLVM 特有的两点必须处理：

1. **必须 build 一次**：LLVM 大量用 TableGen 生成 `.inc`/`.def` 头文件（如 `Attributes.inc`），不构建这些头文件就不存在，`clangd-indexer` 解析每个 TU 时会报错。所以要 `ninja` 一遍（至少把生成头文件产出来）。
2. **体积巨大**：全量构建几十 GB。务必用 README 5.4 的 LVM 扩容，并**限制 `LLVM_ENABLE_PROJECTS` 和 `LLVM_TARGETS_TO_BUILD`** 控制时间/体积；用 `clang + lld` 加速。

路径可移植性问题与前面一致，结论照旧：**首选 WSL 路径对齐消费**。

---

## 1. 准备仓库（同前）

新建仓库（如 `llvm-clangd-index`，public/private 均可）→ `Settings` → `Actions` → `General` → **Read and write permissions**。

---

## 2. workflow 文件

新建 `.github/workflows/build-llvm-clangd-index.yml`：

```yaml
name: Build llvm-project clangd index

on:
  workflow_dispatch:
    inputs:
      llvm_ref:
        description: "要编译的版本：tag / commit hash / 分支（如 llvmorg-19.1.7 或 main 或 40 位 hash）"
        required: true
        default: "llvmorg-19.1.7"
      llvm_projects:
        description: "要索引的子项目 LLVM_ENABLE_PROJECTS（如 clang / 'clang;lld' / 'clang;clang-tools-extra'）"
        required: false
        default: "clang"
      llvm_targets:
        description: "构建的后端 LLVM_TARGETS_TO_BUILD（X86 / AArch64 / 'X86;AArch64' / all）"
        required: false
        default: "X86"
      clangd_version:
        description: "clangd / indexing tools 版本（务必与本地 clangd 主版本一致）"
        required: false
        default: "19.1.2"

permissions:
  contents: write

env:
  LLVM_ROOT: /opt/llvm-project       # 固定根目录，本地映射时作为 project-root / MountPoint

jobs:
  build-index:
    runs-on: ubuntu-22.04
    steps:
      - name: Expand build disk (LVM，对应 README 5.4，LLVM 必需)
        run: |
          mnt_size=$(expr $(df -h /mnt | tail -1 | awk '{print $4}' | sed 's/[[:alpha:]]//g;s/\..*//') - 1)
          root_size=$(expr $(df -h /   | tail -1 | awk '{print $4}' | sed 's/[[:alpha:]]//g;s/\..*//') - 4)
          sudo truncate -s "${mnt_size}"G /mnt/mnt.img
          sudo truncate -s "${root_size}"G /root.img
          sudo losetup /dev/loop6 /mnt/mnt.img
          sudo losetup /dev/loop7 /root.img
          sudo pvcreate /dev/loop6 /dev/loop7
          sudo vgcreate github /dev/loop6 /dev/loop7
          sudo lvcreate -n runner -l 100%FREE github
          sudo mkfs.xfs /dev/github/runner
          sudo mkdir -p "${LLVM_ROOT}"
          sudo mount /dev/github/runner "${LLVM_ROOT}"
          sudo chown -R "$USER:$USER" "${LLVM_ROOT}"
          df -Th

      - name: Install build deps (cmake / ninja / clang / lld)
        run: |
          sudo apt-get update
          sudo apt-get install -y --no-install-recommends \
            git wget unzip ca-certificates \
            cmake ninja-build clang lld llvm python3

      - name: Download clangd indexing tools
        run: |
          ver="${{ inputs.clangd_version }}"
          cd /tmp
          wget -q "https://github.com/clangd/clangd/releases/download/${ver}/clangd_indexing_tools-linux-${ver}.zip"
          unzip -q "clangd_indexing_tools-linux-${ver}.zip"
          sudo cp -f "clangd_indexing_tools-${ver}/bin/"* /usr/local/bin/
          clangd-indexer --version || true

      - name: Clone llvm-project @ ${{ inputs.llvm_ref }}
        run: |
          git clone --depth 1 --branch "${{ inputs.llvm_ref }}" \
            https://github.com/llvm/llvm-project.git "${LLVM_ROOT}" 2>/dev/null \
          || { git clone --filter=blob:none https://github.com/llvm/llvm-project.git "${LLVM_ROOT}" \
               && git -C "${LLVM_ROOT}" checkout --force "${{ inputs.llvm_ref }}"; }
          echo "RESOLVED_SHA=$(git -C "${LLVM_ROOT}" rev-parse --short HEAD)" >> "${GITHUB_ENV}"

      - name: Configure (CMake 原生导出 compile_commands.json)
        run: |
          cd "${LLVM_ROOT}"
          cmake -G Ninja -S llvm -B build \
            -DCMAKE_BUILD_TYPE=Release \
            -DLLVM_ENABLE_ASSERTIONS=OFF \
            -DCMAKE_EXPORT_COMPILE_COMMANDS=ON \
            -DLLVM_ENABLE_PROJECTS="${{ inputs.llvm_projects }}" \
            -DLLVM_TARGETS_TO_BUILD="${{ inputs.llvm_targets }}" \
            -DCMAKE_C_COMPILER=clang -DCMAKE_CXX_COMPILER=clang++ \
            -DLLVM_USE_LINKER=lld

      - name: Build (产出 TableGen 生成头文件，索引解析必需)
        run: |
          cd "${LLVM_ROOT}"
          # 完整 ninja 最稳；想省时可只构建生成头文件相关目标
          ninja -C build

      - name: Generate static index
        run: |
          cd "${LLVM_ROOT}"
          test -s build/compile_commands.json
          clangd-indexer --executor=all-TUs build/compile_commands.json > /tmp/llvm.idx
          ls -lh build/compile_commands.json /tmp/llvm.idx

      - name: Package artifacts
        run: |
          out=/tmp/out; mkdir -p "${out}"
          ref="${{ inputs.llvm_ref }}"; safe="${ref//\//_}"
          cp /tmp/llvm.idx                          "${out}/llvm-${safe}.idx"
          cp "${LLVM_ROOT}/build/compile_commands.json" "${out}/compile_commands.json"
          # 源码快照
          git -C "${LLVM_ROOT}" archive --format=tar.gz --prefix="llvm-project/" \
              -o "${out}/llvm-src-${safe}.tar.gz" HEAD
          # 生成产物快照（保留生成头文件 + cmake，剔除巨大的目标/库文件）
          tar -czf "${out}/llvm-generated-${safe}.tar.gz" \
              --exclude='*.o' --exclude='*.a' --exclude='*.so*' \
              --exclude='*.obj' --exclude='*.dylib' --exclude='bin' --exclude='lib' \
              -C "${LLVM_ROOT}" build
          cat > "${out}/dot-clangd-example.yaml" <<'EOF'
          CompileFlags:
            CompilationDatabase: /opt/llvm-project/build
          Index:
            External:
              File: /opt/llvm-project/llvm.idx
              MountPoint: /opt/llvm-project
            Background: Skip
          EOF
          echo "BUILD_SAFE_REF=${safe}" >> "${GITHUB_ENV}"
          ls -lh "${out}"

      - name: Publish to Release
        env:
          GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        run: |
          tag="llvm-${BUILD_SAFE_REF}"
          gh release create "${tag}" /tmp/out/* \
            --title "llvm clangd index (${{ inputs.llvm_ref }} @ ${RESOLVED_SHA})" \
            --notes "clangd 静态索引 for llvm-project ${{ inputs.llvm_ref }} (projects=${{ inputs.llvm_projects }}, targets=${{ inputs.llvm_targets }})。构建 project-root=${LLVM_ROOT}，clangd 工具版本=${{ inputs.clangd_version }}。" \
          || gh release upload "${tag}" /tmp/out/* --clobber
```

关键说明：
- **CMake 配置阶段就写出 `build/compile_commands.json`**，但里面引用的 TableGen 生成头文件要 `ninja` 后才存在，所以索引前必须构建。
- **用 `clang + lld` 构建**，和 clangd 解析一致且更快。
- **`llvm_projects` / `llvm_targets` 控制规模**：默认 `clang` + `X86` 已经不小；`all` 目标 + 多项目会显著拉长时间和体积，必要时配合 LVM 扩容。
- 产物里的 `llvm-generated-*.tar.gz` 剔除了 `.o/.a/.so` 和 `bin/lib`，只留生成头文件和 cmake 信息——本地解析必需，但体积可控。

---

## 3. 手动触发

`Actions` → `Build llvm-project clangd index` → `Run workflow`：
- `llvm_ref`：如 `llvmorg-19.1.7`
- `llvm_projects`：如 `clang` 或 `clang;lld`
- `llvm_targets`：如 `X86`

完成后 Release `llvm-llvmorg-19.1.7` 含：`.idx`、`compile_commands.json`、源码快照、生成产物快照、配置模板。

---

## 4. 下载（同前）

- public：直接下载。
- private：`gh release download llvm-llvmorg-19.1.7 --repo your-name/llvm-clangd-index --dir ./download`。

---

## 5. 本地 VSCode + clangd 配置

### 推荐：WSL 路径对齐（导航/补全全可用）

```bash
sudo mkdir -p /opt/llvm-project && sudo chown -R "$USER:$USER" /opt/llvm-project
tar -xzf llvm-src-llvmorg-19.1.7.tar.gz -C /tmp
cp -r /tmp/llvm-project/* /opt/llvm-project/
# 关键：恢复 build 目录里的生成头文件，否则解析报错
tar -xzf llvm-generated-llvmorg-19.1.7.tar.gz -C /opt/llvm-project/
cp llvm-llvmorg-19.1.7.idx /opt/llvm-project/llvm.idx
# compile_commands.json 已在 build/ 内（随 generated 包恢复）
```

`/opt/llvm-project/.clangd`（路径完全对齐，无需映射）：

```yaml
CompileFlags:
  CompilationDatabase: /opt/llvm-project/build
Index:
  External:
    File: /opt/llvm-project/llvm.idx
    MountPoint: /opt/llvm-project
  Background: Skip
```

VSCode 用 WSL 远程打开 `/opt/llvm-project`。

### 原生 Windows

与前两答的方式 B / C 一致：
- **方式 B（导航可用）**：`clangd-index-server.exe llvm-*.idx --project-root=/opt/llvm-project`，用户配置 `Index.External.Server: localhost:50051` + `MountPoint: C:/src/llvm-project/`。
- **方式 C（最简单）**：`.clangd` 直接 `Index.External.File` 指向 `.idx`，全局补全/查引用可用，跳转到索引内符号可能落到 Linux 路径而失效。

---

三个项目（nginx / linux / llvm）的差异汇总：

| 项目 | 生成 compile_commands 方式 | 是否需先 build | 规模 / 是否需 LVM 扩容 |
|---|---|---|---|
| nginx | `bear -- make` | 是（bear 拦截） | 小，不需要 |
| linux | `make compile_commands.json` | 是（先 `make` 出 `.cmd`） | 中大，allmodconfig 需扩容 |
| llvm | CMake `-DCMAKE_EXPORT_COMPILE_COMMANDS=ON` | 是（先 `ninja` 出 TableGen 头） | 大，建议扩容 + 限制 projects/targets |

需要我把这三个 workflow + 各自 `.clangd` 模板一并写到你工作区目录里，方便直接推到 GitHub 吗？

---

**User**

https://github.com/v2fly/v2ray-core  项目是用go写的，是否也有类似的LSP server工具？这个代码量是否有必要提前生成索引，还是使用vscode的go插件就够用了？如果提前生成索引应当使用什么工具，如何操作呢

---

**Cursor**

先直接回答三个问题，再给具体操作。

## 1. Go 有没有类似 clangd 的 LSP？有，就是 `gopls`

- Go 的官方 LSP server 是 **`gopls`**（由 Go 团队维护），地位等同于 C/C++ 的 clangd。
- **VSCode 的 Go 插件（`golang.go`）底层用的就是 gopls**——你装了 Go 插件，其实就已经在用这个"索引/LSP 引擎"了，二者不是两个东西。

所以对 [v2fly/v2ray-core](https://github.com/v2fly/v2ray-core) 这种纯 Go 项目，你不需要再找别的工具，VSCode Go 插件 = gopls 就是全部。

## 2. 这个代码量需不需要提前生成索引？不需要

结论：**直接用 VSCode Go 插件（gopls）就够了，无需、也没必要走 clangd 那套"Actions 预生成 .idx → 上传 Release → 下载"流程。**

原因在于 Go 和 C/C++ 的工具链模型完全不同：

| 维度 | C/C++ + clangd | Go + gopls |
|---|---|---|
| 编译模型 | 按翻译单元、头文件反复展开，全量后台索引可达**数小时** | 按 package 编译，类型信息加载极快 |
| 构建缓存 | 需自己生成 `compile_commands.json` | Go 自带**内容寻址**的 `GOCACHE`，gopls 直接复用 |
| 索引方式 | 后台分片 / 静态 `.idx`，可离线预生成 | gopls **打开即增量加载并缓存**，无独立可分发的索引格式 |
| 是否有"预生成可下载索引"的标准做法 | 有（clangd-indexer） | **没有**，官方不提供可跨机分发的索引产物 |

v2ray-core 属于**中等规模**的 Go 仓库（约 7000 commits、几十万行级，纯 Go，且 `*.pb.go`、`errors.generated.go` 等生成文件已提交进仓库）。gopls 首次打开通常几十秒内就完成加载，之后走缓存基本秒级。这跟 clangd 给 LLVM/内核做几小时全量索引完全不是一个量级——所以**对 Go 项目预生成索引这件事，既无标准工具，也没有收益**。

## 3. 如果还是想"提前准备"，正确的做法是预热缓存（而不是生成索引）

gopls 没有 `.idx` 可上传 Release，但你可以**预热 Go 的两个缓存**，让 gopls 首次打开更快。这才是 Go 世界的等价物：

```bash
# 1. 预拉取所有依赖到模块缓存（GOMODCACHE，默认 ~/go/pkg/mod）
git clone https://github.com/v2fly/v2ray-core
cd v2ray-core
go mod download

# 2. 预热构建缓存（GOCACHE，默认 ~/.cache/go-build），gopls 会复用
go build ./...
go vet ./...     # 可选，进一步覆盖更多包
```

做完这两步，gopls 打开项目时大部分类型信息已就绪，加载非常快。

> 如果是在 CI 里追求构建/检查提速，可以用 `actions/cache` 缓存 `GOMODCACHE` 和 `GOCACHE`——但这是给 CI 用的，不是给编辑器分发索引用的，和 clangd 那套场景不同。

## VSCode 上的配置（这就是全部所需）

1. 安装官方 **Go 扩展**（`golang.go`）。
2. 首次会提示安装工具，点 “Install All”，或命令面板执行 **`Go: Install/Update Tools`**，确保装上 `gopls`。
3. 装一个不太旧的 Go（建议与项目 `go.mod` 的版本匹配或更新）。
4. 用 VSCode **打开 v2ray-core 根目录**（含 `go.mod` 的目录），gopls 自动识别为一个 module 并开始加载。

可选的 `settings.json` 优化（大仓库体验更好）：

```json
{
  "go.useLanguageServer": true,
  "gopls": {
    "ui.semanticTokens": true,
    "build.directoryFilters": ["-node_modules", "-release"],
    "ui.completion.usePlaceholders": true
  }
}
```

- `build.directoryFilters` 可排除无需分析的目录，减少负担。
- 内存吃紧时 gopls 会自适应，一般无需手动调。

---

一句话总结：**Go 不用搞 clangd 那套预生成索引；v2ray-core 直接用 VSCode Go 插件（gopls）即可，顶多事先 `go mod download` + `go build ./...` 预热缓存让首次打开更快。** 那套 Actions 生成索引上传 Release 的方案只对 C/C++（nginx/linux/llvm）这类索引昂贵的项目才值得做。

---

**User**

我上面的三个workflow文件是否可以放到同一个仓库，还是推荐分开不同的仓库？

---

**Cursor**

可以放同一个仓库，而且对你这种个人用途**推荐放一个仓库**。三个 workflow 互相独立，不会冲突。下面说清楚原因和注意点。

## 为什么放一个仓库没问题

1. **workflow 互不干扰**：`.github/workflows/` 下可以放任意多个 `.yml`，每个都是独立的 `workflow_dispatch`，在 Actions 页面各自一行、单独触发，互不影响。
2. **Release 标签不会撞车**：你三个 workflow 的 tag 已经天然带前缀——`nginx-*`、`linux-*`、`llvm-*`，所以即使在同一个仓库的 Releases 里也不会冲突，还能按前缀排序、一眼区分。
3. **权限只配一次**：`Read and write permissions`、下载用的 PAT（private 时）都只需弄一套。

## 单仓库 vs 多仓库对比

| 维度 | 同一个仓库（推荐） | 拆成多个仓库 |
|---|---|---|
| 管理成本 | 一处维护，一套权限/token | 每个仓库都要重复配置 |
| Actions 额度（private） | **共享**月度分钟数 | 每个仓库独立，但总额度仍按账号算，没本质区别 |
| Release 列表 | 三类混在一起，靠 tag 前缀区分 | 各仓库独立、更清爽 |
| 访问控制 | 要么全 public 要么全 private | 可单独设某个 public、某个 private |
| 产物存储 | 同一仓库累积 | 分散到各仓库 |

## 什么时候才建议拆开

- 想让**不同项目走不同可见性**（比如 nginx 索引公开、内部项目私有）。
- 想让某个超大索引（llvm allmodconfig 之类）的存储/下载**单独隔离**。
- 多人协作、需要**按项目分别授权**。

如果没有这些诉求，**一个仓库 + 三个 workflow** 最省心。

## 放同仓库时的两个小建议

1. 文件名保持清晰：`build-nginx-clangd-index.yml`、`build-linux-clangd-index.yml`、`build-llvm-clangd-index.yml`，Actions 列表里就是三个独立条目。
2. 三者都用了 `LINUX_ROOT=/opt/...` 这类 env，但都是**各自 job 内的环境变量**，不共享、不冲突，无需改动。

> 进阶（可选）：如果你想"一个入口选项目"，也可以合并成**单个 workflow**，加一个 `project` 选择输入（nginx/linux/llvm）再分支执行——但三个独立文件其实更直观、更好维护，我更推荐保持现状放同一个仓库即可。

需要的话我可以把三个 workflow 文件直接生成到你工作区的 `.github/workflows/` 下，方便你一次性推到同一个仓库。

---

**User**

除了上述项目，推荐5-10个其他的大型经典的项目用于学习，分别生成对应的workflow文件和相关配置说明

---

**Cursor**

这一系列项目的关注点是 **C/C++ 这种"索引昂贵、值得预生成"** 的经典大仓（Go 那类不需要，前面已说明）。下面挑了 **8 个**覆盖不同构建系统、风格各异的经典项目，方便你系统学习。

因为 8 个 workflow 95% 内容是重复的，只在「依赖安装」「构建命令」「compile_commands.json 位置」三处不同，所以我给你**一个通用模板 + 每个项目的差异块**，比贴 8 份近乎一样的 80 行 YAML 实用得多——把差异块填进模板，存成 `build-<项目>-clangd-index.yml` 即可。

## 推荐项目一览

| 项目 | 语言 | 构建系统 | 规模 | 学习价值 |
|---|---|---|---|---|
| Redis | C | Makefile | 中小 | 事件循环、数据结构、单线程模型，代码极干净 |
| Git | C | Makefile | 中 | 版本控制内核、对象模型 |
| PostgreSQL | C | autotools | 大 | 经典 DBMS：MVCC、查询优化、存储 |
| CPython | C | autotools | 中大 | 解释器/字节码/GC/对象模型 |
| FFmpeg | C | 自定义 configure | 大 | 多媒体编解码、性能优化 |
| curl | C | autotools | 中小 | 网络协议栈、跨平台抽象 |
| QEMU | C | meson | 大 | 虚拟化/模拟器、设备模型 |
| RocksDB | C++ | CMake | 中大 | LSM-Tree 存储引擎，现代 C++ |

> 加餐（同 CMake 套路，按需照搬 RocksDB 块）：OpenCV、protobuf、LevelDB、gRPC。

---

## 通用模板（`build-<项目>-clangd-index.yml`）

```yaml
name: Build PROJECT clangd index

on:
  workflow_dispatch:
    inputs:
      ref:
        description: "commit hash / tag / 分支"
        required: true
        default: "DEFAULT_REF"          # ← 替换
      clangd_version:
        description: "clangd / indexing tools 版本（与本地 clangd 主版本一致）"
        required: false
        default: "19.1.2"

permissions:
  contents: write

env:
  SRC_ROOT: /opt/src
  CDB_DIR: /opt/src                      # ← bear/autotools 用 /opt/src；CMake/meson 改成 /opt/src/build
  NEED_LVM: "false"                      # ← 大项目(ffmpeg/qemu/postgres/opencv)设为 true

jobs:
  build-index:
    runs-on: ubuntu-22.04
    steps:
      - name: Expand disk via LVM (大项目用，对应 README 5.4)
        if: env.NEED_LVM == 'true'
        run: |
          mnt_size=$(expr $(df -h /mnt | tail -1 | awk '{print $4}' | sed 's/[[:alpha:]]//g;s/\..*//') - 1)
          root_size=$(expr $(df -h /   | tail -1 | awk '{print $4}' | sed 's/[[:alpha:]]//g;s/\..*//') - 4)
          sudo truncate -s "${mnt_size}"G /mnt/mnt.img; sudo truncate -s "${root_size}"G /root.img
          sudo losetup /dev/loop6 /mnt/mnt.img; sudo losetup /dev/loop7 /root.img
          sudo pvcreate /dev/loop6 /dev/loop7; sudo vgcreate github /dev/loop6 /dev/loop7
          sudo lvcreate -n runner -l 100%FREE github; sudo mkfs.xfs /dev/github/runner
          sudo mkdir -p "${SRC_ROOT}"; sudo mount /dev/github/runner "${SRC_ROOT}"
          sudo chown -R "$USER:$USER" "${SRC_ROOT}"; df -Th

      - name: Install deps
        run: |
          sudo apt-get update
          # >>> PROJECT DEPS（替换为对应项目的依赖行） <<<
          sudo apt-get install -y --no-install-recommends \
            git wget unzip ca-certificates PROJECT_DEPS

      - name: Download clangd indexing tools
        run: |
          ver="${{ inputs.clangd_version }}"; cd /tmp
          wget -q "https://github.com/clangd/clangd/releases/download/${ver}/clangd_indexing_tools-linux-${ver}.zip"
          unzip -q "clangd_indexing_tools-linux-${ver}.zip"
          sudo cp -f "clangd_indexing_tools-${ver}/bin/"* /usr/local/bin/

      - name: Clone PROJECT @ ${{ inputs.ref }}
        run: |
          sudo mkdir -p "${SRC_ROOT}"; sudo chown -R "$USER:$USER" "${SRC_ROOT}"
          git clone --filter=blob:none REPO_URL "${SRC_ROOT}"     # ← 替换 REPO_URL
          git -C "${SRC_ROOT}" checkout --force "${{ inputs.ref }}"
          # 若项目用子模块：git -C "${SRC_ROOT}" submodule update --init --recursive --depth 1
          echo "RESOLVED_SHA=$(git -C "${SRC_ROOT}" rev-parse --short HEAD)" >> "${GITHUB_ENV}"

      - name: Build & export compile_commands.json
        run: |
          cd "${SRC_ROOT}"
          # >>> PROJECT BUILD BLOCK（替换为对应项目的构建命令） <<<
          test -s "${CDB_DIR}/compile_commands.json"

      - name: Generate static index
        run: |
          clangd-indexer --executor=all-TUs "${CDB_DIR}/compile_commands.json" > /tmp/proj.idx
          ls -lh /tmp/proj.idx

      - name: Package + Release
        env:
          GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        run: |
          out=/tmp/out; mkdir -p "${out}"
          ref="${{ inputs.ref }}"; safe="${ref//\//_}"; proj="PROJECT"   # ← 替换 PROJECT
          cp /tmp/proj.idx                "${out}/${proj}-${safe}.idx"
          cp "${CDB_DIR}/compile_commands.json" "${out}/compile_commands.json"
          git -C "${SRC_ROOT}" archive --format=tar.gz --prefix="${proj}/" -o "${out}/${proj}-src-${safe}.tar.gz" HEAD
          # 生成头文件快照（autotools 的 config.h / CMake·meson 的 build 头文件，本地解析必需）
          if [ "${CDB_DIR}" != "${SRC_ROOT}" ]; then
            tar -czf "${out}/${proj}-generated-${safe}.tar.gz" --exclude='*.o' --exclude='*.a' \
                --exclude='*.so*' --exclude='*.obj' -C "${SRC_ROOT}" "$(basename "${CDB_DIR}")"
          fi
          gh release create "${proj}-${safe}" /tmp/out/* \
            --title "${proj} clangd index (${ref} @ ${RESOLVED_SHA})" \
            --notes "clangd 静态索引 for ${proj} ${ref}。project-root=${SRC_ROOT}，clangd 工具=${{ inputs.clangd_version }}。" \
          || gh release upload "${proj}-${safe}" /tmp/out/* --clobber
```

填三处即可：`PROJECT_DEPS`、`REPO_URL` + `DEFAULT_REF`、`PROJECT BUILD BLOCK`（必要时改 `CDB_DIR` / `NEED_LVM` / 子模块）。

---

## 各项目差异块

### 1. Redis（C / Makefile）
- `REPO_URL` `https://github.com/redis/redis.git`，`DEFAULT_REF` `8.0.0`
- DEPS：`bear build-essential pkg-config`
- CDB_DIR：`/opt/src`，NEED_LVM：false
- BUILD：
```bash
bear -- make -j"$(nproc)"
```
> 自带 jemalloc/lua/hiredis，无需额外依赖。最适合入门精读。

### 2. Git（C / Makefile）
- `REPO_URL` `https://github.com/git/git.git`，`DEFAULT_REF` `v2.45.0`
- DEPS：`bear build-essential gettext zlib1g-dev libssl-dev libcurl4-openssl-dev libexpat1-dev`
- CDB_DIR：`/opt/src`，NEED_LVM：false
- BUILD：
```bash
bear -- make -j"$(nproc)"
```

### 3. PostgreSQL（C / autotools）
- `REPO_URL` `https://github.com/postgres/postgres.git`，`DEFAULT_REF` `REL_17_2`
- DEPS：`bear build-essential bison flex libreadline-dev zlib1g-dev libicu-dev pkg-config`
- CDB_DIR：`/opt/src`，NEED_LVM：true
- BUILD：
```bash
./configure
bear -- make -j"$(nproc)"
```

### 4. CPython（C / autotools）
- `REPO_URL` `https://github.com/python/cpython.git`，`DEFAULT_REF` `v3.13.1`
- DEPS：`bear build-essential pkg-config libssl-dev zlib1g-dev libffi-dev libbz2-dev libreadline-dev libsqlite3-dev liblzma-dev`
- CDB_DIR：`/opt/src`，NEED_LVM：false
- BUILD：
```bash
./configure --with-pydebug
bear -- make -j"$(nproc)"
```
> 会生成 `pyconfig.h`，本地解析需要（在源码树内，随源码快照已包含）。

### 5. FFmpeg（C / 自定义 configure）
- `REPO_URL` `https://github.com/FFmpeg/FFmpeg.git`，`DEFAULT_REF` `n7.1`
- DEPS：`bear build-essential nasm yasm pkg-config`
- CDB_DIR：`/opt/src`，NEED_LVM：true
- BUILD：
```bash
./configure
bear -- make -j"$(nproc)"
```
> configure 会生成 `config.h`/`config_components.h` 等，已在源码树内。

### 6. curl（C / autotools）
- `REPO_URL` `https://github.com/curl/curl.git`，`DEFAULT_REF` `curl-8_11_0`
- DEPS：`bear build-essential autoconf automake libtool pkg-config libssl-dev`
- CDB_DIR：`/opt/src`，NEED_LVM：false
- BUILD：
```bash
autoreconf -fi
./configure --with-openssl
bear -- make -j"$(nproc)"
```

### 7. QEMU（C / meson —— 原生导出 CDB）
- `REPO_URL` `https://github.com/qemu/qemu.git`，`DEFAULT_REF` `v9.2.0`
- DEPS：`build-essential ninja-build pkg-config python3 python3-venv libglib2.0-dev libpixman-1-dev flex bison`（**不需要 bear**）
- **CDB_DIR：`/opt/src/build`**，NEED_LVM：true
- BUILD：
```bash
./configure --target-list=x86_64-softmmu --enable-debug
make -j"$(nproc)"        # meson 会在 build/ 写出 compile_commands.json
```
> 限定 `--target-list` 控制体积。`build/` 里的生成头文件会被 generated 包打进 Release。

### 8. RocksDB（C++ / CMake —— 原生导出 CDB）
- `REPO_URL` `https://github.com/facebook/rocksdb.git`，`DEFAULT_REF` `v9.7.4`
- DEPS：`build-essential cmake ninja-build libgflags-dev libsnappy-dev zlib1g-dev libbz2-dev liblz4-dev libzstd-dev`（**不需要 bear**）
- **CDB_DIR：`/opt/src/build`**，NEED_LVM：true
- BUILD：
```bash
cmake -G Ninja -S . -B build \
  -DCMAKE_BUILD_TYPE=Release -DCMAKE_EXPORT_COMPILE_COMMANDS=ON \
  -DWITH_GFLAGS=ON -DWITH_SNAPPY=ON -DWITH_LZ4=ON -DWITH_ZSTD=ON -DWITH_BZ2=ON
cmake --build build -j"$(nproc)"
```
> 加餐项目（OpenCV / protobuf）同此套路：CMake + `-DCMAKE_EXPORT_COMPILE_COMMANDS=ON`，protobuf 记得 `submodule update --init`（abseil）。

---

## 本地消费（所有项目一致）

和前面 nginx/linux/llvm 完全相同，**首选 WSL 路径对齐**：

```bash
sudo mkdir -p /opt/src && sudo chown -R "$USER:$USER" /opt/src
tar -xzf <proj>-src-<ref>.tar.gz -C /tmp && cp -r /tmp/<proj>/* /opt/src/
# 仅 CMake/meson 项目需要：恢复 build 目录里的生成头文件
[ -f <proj>-generated-<ref>.tar.gz ] && tar -xzf <proj>-generated-<ref>.tar.gz -C /opt/src/
cp <proj>-<ref>.idx /opt/src/proj.idx
cp compile_commands.json /opt/src/        # CMake/meson 项目其 CDB 在 build/，随 generated 包恢复
```

`/opt/src/.clangd`：

```yaml
CompileFlags:
  CompilationDatabase: /opt/src          # CMake/meson 项目改成 /opt/src/build
Index:
  External:
    File: /opt/src/proj.idx
    MountPoint: /opt/src
  Background: Skip
```

原生 Windows 仍是方式 B（本地 `clangd-index-server --project-root=/opt/src` + `MountPoint`）或方式 C（直接 `Index.External.File`，补全可用、跳转可能失效），细节同前。

---

构建系统与 CDB 生成方式的对应关系（记住这个就能套任意 C/C++ 项目）：

| 构建系统 | 生成 compile_commands.json | CDB 位置 | 代表项目 |
|---|---|---|---|
| Makefile | `bear -- make` | 源码根 | Redis、Git |
| autotools | `./configure && bear -- make` | 源码根 | PostgreSQL、CPython、curl |
| 自定义 configure | `./configure && bear -- make` | 源码根 | FFmpeg、nginx |
| 内核 Kbuild | `make compile_commands.json`（先 build） | 源码根 | linux |
| CMake | `-DCMAKE_EXPORT_COMPILE_COMMANDS=ON` | `build/` | RocksDB、LLVM、OpenCV |
| meson | 原生自动导出 | `build/` | QEMU |

需要我把这个**通用模板 + 8 个填好的 workflow 文件 + `.clangd` 模板**直接生成到你工作区的 `.github/workflows/` 下吗？这样你可以连同 nginx/linux/llvm 一起推到同一个仓库。

---

**User**

将通用模板 + 8个填好的workflow文件 + nginx/linux/llvm + OpenCV、protobuf、LevelDB、gRPC对应的workflow文件 + .clangd模板 + 包含这些项目的.clangd配置 写入我工作区的workflows下，其中/opt/前缀统一替换为/codebase/

---

**Cursor**

明白。我会把通用模板、15 个填好的 workflow（nginx/linux/llvm + redis/git/postgresql/cpython/ffmpeg/curl/qemu/rocksdb + opencv/protobuf/leveldb/grpc）、`.clangd` 模板，以及一份包含全部项目的合并版 `config.yaml` 写到 `workflows/` 下，并把所有 `/opt/` 前缀替换为 `/codebase/`。

为了让"合并版多项目配置"能路径对齐、跳转可用，我让每个项目都构建/消费在各自独立目录 `/codebase/<项目名>` 下（仍在 `/codebase/` 前缀内）。

先确认工作区目录，然后开始写入。

开始写入文件。先创建目录并写入通用模板与三个原始项目（nginx/linux/llvm）。

现在写 bear/make/autotools 类项目（redis、git、postgresql、cpython、ffmpeg、curl）。

现在写 meson/CMake 类项目（qemu、rocksdb、opencv、protobuf、leveldb、grpc），它们的 CDB 在 `build/` 且需要打包生成头文件。

现在写 `.clangd` 单项目模板与包含所有项目的合并版 `config.yaml`。

全部写完。做一下 YAML 语法校验，确保没有缩进/格式错误。

全部写入完成，18 个文件 YAML 语法校验通过，且所有 `/opt/` 前缀已统一为 `/codebase/`。

## 已生成文件（`/home/tichen/cursor/workflows/`）

**通用模板（1）**
- `_template.yml` — 标注了 4 处 env + 3 处命令的替换说明，套任意 C/C++ 项目

**15 个填好的 workflow**

| 文件 | 构建方式 | CDB 位置 | LVM | 默认 ref |
|---|---|---|---|---|
| `build-nginx-clangd-index.yml` | configure+bear | 根 | 否 | master |
| `build-linux-clangd-index.yml` | kbuild | 根 | 是 | v6.12 |
| `build-llvm-clangd-index.yml` | cmake | build | 是 | llvmorg-19.1.7 |
| `build-redis-clangd-index.yml` | make+bear | 根 | 否 | 8.0.0 |
| `build-git-clangd-index.yml` | make+bear | 根 | 否 | v2.45.0 |
| `build-postgresql-clangd-index.yml` | autotools+bear | 根 | 是 | REL_17_2 |
| `build-cpython-clangd-index.yml` | autotools+bear | 根 | 否 | v3.13.1 |
| `build-ffmpeg-clangd-index.yml` | configure+bear | 根 | 是 | n7.1 |
| `build-curl-clangd-index.yml` | autotools+bear | 根 | 否 | curl-8_11_0 |
| `build-qemu-clangd-index.yml` | meson | build | 是 | v9.2.0 |
| `build-rocksdb-clangd-index.yml` | cmake | build | 是 | v9.7.4 |
| `build-opencv-clangd-index.yml` | cmake | build | 是 | 4.10.0 |
| `build-protobuf-clangd-index.yml` | cmake+子模块 | build | 是 | v29.3 |
| `build-leveldb-clangd-index.yml` | cmake+子模块 | build | 否 | 1.23 |
| `build-grpc-clangd-index.yml` | cmake+子模块 | build | 是 | v1.68.0 |

**clangd 配置（2）**
- `clangd/dot-clangd-template.yaml` — 单项目 `.clangd` 模板
- `clangd/config-all-projects.yaml` — 一份全局 `config.yaml` 覆盖全部 15 个项目（每段 `If.PathMatch` 限定 `/codebase/<项目>`）

## 使用要点

1. 推到 GitHub 时，把这些 `.yml` 放到仓库的 **`.github/workflows/`** 下（GitHub 只识别该目录）。`_template.yml` 仅作参考，可不推。
2. 每个项目构建/消费路径统一为 `/codebase/<项目>`，所以合并版配置在 **WSL 路径对齐**时跳转/补全全可用。
3. 部分默认 ref（如 protobuf `v29.3`、grpc `v1.68.0`）请按需在触发时改成实际存在的 tag。
4. 子模块项目（protobuf/leveldb/grpc）源码包用 `tar`（含子模块、排除 `.git`/`build`），其余用 `git archive`。

需要我顺手把它们也复制一份到 `.github/workflows/` 目录，或生成一个简短的使用 README 吗？

---

**User**

"clangd.arguments": [
    "--enable-config",
    "--clang-tidy",
    "--path-mappings=/workspace=/tmp/ci_workspace"
    ]
可以完成路径转换，windows中是否也可以通过mappings=C:/codebase/nginx=/codebase/nginx"完成解析映射

---

**Cursor**

好问题，但答案是：**在原生 Windows 上，`--path-mappings=C:/codebase/nginx=/codebase/nginx` 这个方向行不通**。原因在于这个 flag 的真实语义和你期望的"重写索引内路径"不是一回事。

## `--path-mappings` 到底做什么

它解决的是 **client（编辑器）↔ server（clangd 进程所在磁盘）** 两套文件系统的 URI 映射，格式是 `<client路径>=<server路径>`，**双向生效**：

- 编辑器发来 didOpen（client 路径）→ clangd 按 `client→server` 改写，**然后用 server 路径去磁盘读文件**；
- clangd 返回跳转/引用结果（server 路径）→ 按 `server→client` 改写后发回编辑器。

它的设计场景是：**clangd 真的跑在文件位于 server 路径的地方**（典型是 clangd 在 Docker/远程容器里，文件挂在 `/workspace`，编辑器在宿主机）。你引用的 `/workspace=/tmp/ci_workspace` 正是这种 CI/容器场景。

**关键点：映射后的 server 路径会被 clangd 当作真实磁盘路径去做 I/O。**

## 为什么 Windows 上你的写法会坏

你写 `--path-mappings=C:/codebase/nginx=/codebase/nginx`（client=`C:/...`，server=`/codebase/...`），结果是：

1. 你在 VSCode 打开 `C:/codebase/nginx/src/core/nginx.c` → clangd 按 client→server 改成 `/codebase/nginx/src/core/nginx.c`；
2. clangd 于是去 **Windows 磁盘**上读 `/codebase/nginx/...` —— 这个路径在 Windows 上根本不存在；
3. 当前文件正文还能靠 LSP didOpen 的内存 buffer，但 **头文件 include、`compile_commands.json` 定位全部按 `/codebase/nginx` 去找 → 全部失败**，解析直接崩。

也就是说：`--path-mappings` **不是"把索引里的路径重映射到本地另一个路径"的工具**，它会强制 clangd 以 server 路径访问磁盘。原生 Windows 上磁盘里只有 `C:/codebase/nginx`，没有 `/codebase/nginx`，所以这条路不通。它也不会单独改写静态 `.idx` 内部存的路径。

## 什么情况下它确实有用

只有当 **clangd 真的跑在文件位于 `/codebase/nginx` 的环境**（WSL / 容器 / 远程 Linux），而编辑器用另一套路径视图时，`--path-mappings` 才发挥作用。而这种情况下——

> 如果你直接用 **VSCode Remote-WSL / Dev Container**，扩展宿主本来就跑在 WSL 里，路径已经是 `/codebase/nginx`，**和索引完全一致，连 mapping 都不需要**。

## 给你的正确选项（按推荐度）

| 方案 | 适用 | 跳转 | 说明 |
|---|---|---|---|
| **WSL 路径对齐**（`/codebase/nginx`，无需任何 mapping） | WSL | ✅ 完整 | 源码解压到 `/codebase/nginx`，索引路径天然一致，最省心 |
| **clangd-index-server** `--project-root=/codebase/nginx` + `MountPoint: C:/codebase/nginx` | 原生 Windows | ✅ 完整 | 本机起服务（localhost，不联网），`--project-root` 去前缀、`MountPoint` 补本地前缀，这才是官方支持的"索引路径重定位" |
| **clangd-remap** 重写分片 | 原生 Windows | ✅ 完整 | 把 `/codebase/nginx` 改写成 `C:/codebase/nginx`，但只作用于**后台 index 分片**，不是静态 `.idx`，且工具较新 |
| `Index.External.File` 直接加载 | 原生 Windows | ⚠️ 部分 | 补全/搜索可用；`MountPoint` 只做过滤不改写路径，跳转到索引内符号会指向 `/codebase/...` 而失效 |

一句话：你想要的"路径转换"在原生 Windows 上要用 **`clangd-index-server --project-root` + `MountPoint`**（或 WSL 直接对齐），而不是 `--path-mappings`。`--path-mappings` 是给"clangd 跑在远端/容器"用的，套到原生 Windows 反而会让 clangd 去找不存在的 `/codebase` 路径。

---

**User**

github上是否已经有类似流程的代码仓库了

---

**Cursor**

让我搜一下 GitHub 上是否已有类似"CI 预生成 clangd 索引并分发"的现成仓库。

有，而且**官方就有一个几乎一模一样的参考实现**。你设计的这套流程本质上就是 clangd 官方"远程索引"方案的简化版。

## 最接近的现成仓库

### 1. `clangd/llvm-remote-index`（官方，最值得参考）
[clangd/llvm-remote-index](https://github.com/clangd/llvm-remote-index) —— 这就是你这套流程的"生产级"原版：
- 用 **GitHub Actions 每天定时**构建 llvm-project 的单体索引（`-DLLVM_ENABLE_PROJECTS="all"`）；
- 跑 `clangd-indexer --executor=all-TUs build/compile_commands.json > llvm.idx`；
- 打包成 zip **发布为 GitHub Release**；
- 索引服务器（GCE 上）定期拉取最新 Release 的索引对外提供。

它的 [`.github/workflows/index.yaml`](https://github.com/clangd/llvm-remote-index/blob/master/.github/workflows/index.yaml) 跟你写的 llvm workflow 几乎逐条对应（checkout → apt 装依赖 → cmake 配置导出 CDB → clangd-indexer → 打包 → create release → upload asset）。差别只是：它最终是给 `clangd-index-server` 用的远程索引，而你是直接下载 `.idx` 本地用。

对应的在线服务和说明见 [LLVM remote index service](https://sam-mccall.github.io/clangd-www/llvm-remote-index.html) 和官方 [Using a remote index](https://clangd.llvm.org/guides/remote-index.html)。

### 2. `kirillbobyrev/indexing-tools`
clangd 维护者最早在 Actions 上做 LLVM 索引的实验仓库（上面那个的前身），见 [clangd issue #479 的说明](https://github.com/clangd/clangd/issues/479)。

### 3. Chromium 的远程索引
Google 为 Chromium 跑了同类服务（`linux.clangd-index.chromium.org:5900` 等），也是 Actions/CI 定期 `clangd-indexer` + 服务器分发。

## 两个对你方案有用的"官方结论"

1. **官方 clangd release 自带 remote-index 支持**：搜索结果确认"The clangd releases on GitHub include remote index support, but official LLVM releases do not"。所以前面讲的"原生 Windows 用 `clangd-index-server --project-root` + `MountPoint`"这条路，用官方 clangd 二进制就能跑，不用自己编译。

2. **新能力：clangd-indexer 现在能直接产出"后台索引分片"**（[clangd issue #587](https://github.com/clangd/clangd/issues/587) + [llvm PR #175209](https://github.com/llvm/llvm-project/pull/175209)）：
   ```bash
   clangd-indexer --format=background --executor=all-TUs compile_commands.json
   ```
   它会直接写出 `.cache/clangd/index/*.idx` 分片（而非单体 `.dex/.idx`）。意义在于：分片格式可配合新的 `clangd-remap` 做路径重写，原生 Windows 跳转问题更好解。如果你本地 clangd 版本够新，可以在 workflow 里改用这个模式，省去 index-server。

## 这对你意味着什么

- **没必要从零造轮子**：流程是经过官方验证的成熟模式，`clangd/llvm-remote-index` 可以直接当模板对照（尤其是 release 上传那段，它用的是 `actions/create-release` + `actions/upload-release-asset`，你用的 `gh release create` 更简洁）。
- 但**"为多个项目 + 指定 hash + 直接下载本地用（而非架服务器）"这种打包形态，公开仓库里并不常见**——大多数现成方案要么只针对 LLVM/Chromium 单个项目，要么走 remote-index-server。你写的这一套多项目、可选 hash、产物自带源码/生成头文件快照、面向本地直接消费的组合，算是把官方模式做了实用化扩展，目前没看到完全等价的现成仓库。

要不要我把 workflow 里的 `clangd-indexer` 调用改成可选 `--format=background` 模式（并相应调整本地消费方式），让原生 Windows 的跳转更顺？

---

**User**

好，调整好之后为workflows生成README.md，包含原理说明，参考来源和使用说明

---

**Cursor**

先确认一处的精确空白，避免替换不匹配，然后批量调整所有 workflow 支持 `--format=background` 切换。

我把所有 workflow 增加一个 `index_format` 开关（`static` 默认 / `background`），并相应调整索引生成与打包逻辑。由于改动涉及每个文件的三处且有对齐空白，我直接用完整重写方式确保准确。先重写模板与三个主项目。

模板已更新。现在确认各文件中待替换行的精确内容后批量修改 15 个项目文件。

精确内容已确认（统一 15 空格，linux 带 `-${arch}`）。现在批量为 15 个文件添加 `index_format` 开关并改造索引/打包逻辑。先处理 nginx、redis、git、postgresql、cpython。

---

**User**

继续上个任务

---

**Cursor**

继续。处理 redis、git、postgresql、cpython、ffmpeg、curl。

Now the CMake/meson group: llvm, qemu, rocksdb, opencv, protobuf, leveldb, grpc.

Now the linux file which has special arch handling. Let me read it.

All 15 workflow files are now updated. Let me verify no stale lines remain.

所有 16 个文件（模板 + 15 个项目）都已加入 `index_format`。现在生成 README。先看一下目录结构和已有配置文件。

Now I have enough context. Let me write the README.

现在写 README。

已完成上个任务的全部剩余工作。

## 完成内容

**1. 索引格式开关（`index_format`）同步到全部文件**

除了之前已改好的 `_template.yml` 和 nginx，本次补齐了剩余 14 个项目工作流（redis、git、postgresql、cpython、ffmpeg、curl、llvm、qemu、rocksdb、opencv、protobuf、leveldb、grpc、linux），每个文件统一做了 3 处修改：

- 新增 `index_format` 输入项（`static` / `background`，默认 `static`）
- 索引生成步骤改为条件分支：`background` 走 `clangd-indexer --format=background` 并打包 `.cache/clangd/index`，`static` 走原单体 `.idx`
- 打包步骤按格式拷贝对应产物（`linux` 额外保留了 `-${arch}` 后缀）

校验结果：16 个文件均含 `index_format`，无残留的旧 `Generate static index` 步骤或旧拷贝行。

**2. 新建 `workflows/README.md`**

包含三大块：

- **原理说明**：clangd 为何需要 `compile_commands.json` + 索引、四类构建系统生成编译数据库的方式对照表、为何强制 `/codebase/<项目>` 统一根路径、`static` vs `background` 两种格式、大项目 LVM 扩容。
- **参考来源**：ophub README、clangd 官方文档、Bear、内核 clangd 文档、`--format=background`（LLVM PR #175209）、CMake 导出编译库等链接。
- **使用说明**：从放置工作流 → 手动触发 → 本地落盘 → 配置 `.clangd`/全局 config → VSCode 使用的完整步骤，外加 Windows 注意事项、15 个项目一览表、以及用模板适配新项目的方法。

整个 `workflows/` 目录的工作流改造与文档现已齐备。

