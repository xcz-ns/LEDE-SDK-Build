# LEDE SDK 插件编译仓库

通过 GitHub Actions 手动触发，用 [xcz-ns/LEDE](https://github.com/xcz-ns/LEDE) 发布的 OpenWrt SDK，编译指定插件源码仓库中的插件（.ipk），产物以工作流制品（Artifacts）形式交付，不发布 Release。

## 使用方式

1. 将本仓库推到 GitHub（公开即可，无需任何配置）；
2. 打开 **Actions** 页 → 选择 **SDK 编译插件** → 点 **Run workflow**；
3. 填写参数并运行，等待约 10~30 分钟（取决于插件依赖）；
4. 运行结束后在本次运行结果页的 **Artifacts** 区下载 `.ipk` 文件。

## 输入参数（仅 3 项）

| 参数 | 必填 | 说明 |
|---|---|---|
| firmware_repo | 是 | **插件源码仓库**地址（不是 OpenWrt 源码），支持 `https://github.com/owner/repo` 或 `git@github.com:owner/repo.git`，可以是**包含大量插件和依赖库的聚合仓库**（如 [kenzok8/small-package](https://github.com/kenzok8/small-package)） |
| architecture | 是 | 目标架构，三选一：`LEDE_Cudy` / `LEDE_R3S` / `LEDE_x86` |
| plugin_name | 是 | 要编译的插件包目录名（含 Makefile 的目录名），**支持多个，多个插件之间用空格连接**，如 `luci-app-openclash luci-app-passwall2` |

其余一切自动处理：

- SDK 固定取自 `https://github.com/xcz-ns/LEDE`，无需填写；
- SDK 版本自动选择所选架构的最新 release 标签，无需填写；
- 编译前自动更新并安装全部 feeds（含插件仓库注册的 custom feed），无需配置。

## 架构与 SDK 对应关系（自动匹配）

工作流通过 GitHub API 按标签后缀（`LEDE_Cudy` / `LEDE_R3S` / `LEDE_x86`）找到该架构最新的 release，并在其资产中自动挑选 `openwrt-sdk-*.tar.xz`，无需手动填文件名。当前实际对应：

| 架构 | SDK 资产 |
|---|---|
| LEDE_x86 | `openwrt-sdk-x86-64_gcc-13.3.0_musl.Linux-x86_64.tar.xz` |
| LEDE_Cudy | `openwrt-sdk-mediatek-filogic_gcc-13.3.0_musl.Linux-x86_64.tar.xz` |
| LEDE_R3S | `openwrt-sdk-rockchip-armv8_gcc-13.3.0_musl.Linux-x86_64.tar.xz` |

> 若某个 release 恰好没带 SDK 资产，工作流会自动回退到该架构更早的标签，直到找到带 SDK 的版本。

## luci feed 分支（已固定为 LEDE master）

- xcz-ns/LEDE 构建固件（及 SDK）时，其 `custom.sh` 会把 feeds 里的 luci 源切到 `coolsnowwolf/luci.git;master`，因此 **SDK 自带的 luci 就是 LEDE master 分支**，不是 openwrt-25.x；
- 工作流在编译前还会对 `feeds.conf.default` 再执行一次同样的 sed 固定（`src-git luci https://github.com/coolsnowwolf/luci.git;master`），双重保险，确保 luci 相关插件（luci-app-*）始终基于与固件一致的 master 分支编译。

## 插件与依赖的处理方式（重要）

- 克隆插件源码仓库后，**整个仓库**会被注册为名为 `custom` 的 feed（`src-link`），仓库里的**所有插件和依赖库包**都会对 SDK 构建系统可见；
- 因此像 [kenzok8/small-package](https://github.com/kenzok8/small-package) 这种"很多插件 + 自带依赖库"的仓库可以直接使用：目标插件依赖的同仓库包（如 `sing-box`、`chinadns-ng`、`libcron` 等）会被自动解析并先行编译；
- **只编译你指定的插件**：`make package/<插件名>/compile` 只构建目标插件及其依赖闭包，不会把仓库里所有插件都编译一遍；
- 与 SDK 核心或其他 feed 重名的包（如仓库里的 `base-files`、`dnsmasq` 等）会被 OpenWrt 的 feeds 机制**自动跳过**，不会覆盖 SDK 自带版本；
- 插件在 SDK 中的定位顺序：`custom feed` → 其他 feed（含 luci 的 `applications/` 子目录，递归查找）→ 核心 `package/` 目录。任一插件找不到（或该目录没有 Makefile）会导致任务报错并列出未找到的插件名。
- **与 SDK 内置/其他 feed 重名的插件**：SDK 的 luci feed（coolsnowwolf/luci master）本身已自带大量 `luci-app-*`（如 `luci-app-lucky`）。若目标插件在插件仓库（custom feed）里存在但与其他 feed 重名被跳过，工作流会自动删除 SDK 其他 feed 中的同名项并重装 custom，**优先编译插件仓库中的版本**（参考 helloworld 删官方冲突包的做法）；若插件仓库里也没有该插件，则按 `custom → 其他 feed → 核心 package/` 顺序查找后编译。若插件名找不到，工作流会提示 SDK 中名字相近的包目录供你核对。

## 产物保留策略

- 一次运行编译的所有插件 `.ipk` 合并为一个工作流制品（Artifacts）下载，**不**发布 Release；
- 制品 `retention-days: 7`，即保留最近 7 天，超过 7 天由 GitHub 自动删除，无需额外清理脚本。

## 与成熟方案（fw876/helloworld）对齐的优化

本工作流参考了 [fw876/helloworld](https://github.com/fw876/helloworld) 的 `release-packages.yml` 的成熟做法：

- **GitHub API 全程带 `GITHUB_TOKEN` 认证**：避免匿名调用触发限流（403 / curl exit 22），SDK 标签与资产解析稳定；
- **`feeds install -a -p custom` 优先单独安装插件仓库的包**：`feeds update -a`（失败自动重试）后**先**单独安装 custom feed，再全量 `install -a`（与已装 custom 包重名的会自动跳过，custom 版本保留），确保插件仓库的包一定被安装且版本优先；
- **重名插件自动删冲突包**：若插件在 custom feed 中存在、但与其他 feed（如 luci）重名被跳过，自动删除 SDK 内其他 feed 的同名项并重装 custom——**填了 firmware_repo 就优先编译该仓库的版本**（与 helloworld 删官方 xray-core 的思路一致）；
- **`make download` 预下载全部依赖源码**：编译前一次性拉齐依赖，配合 `dl/` 缓存（按 SDK 标签区分），避免编译阶段反复下载；
- **编译失败自动 `-j1 V=s` 单线程重试**：先并行快编，失败后单线程输出详细日志定位。

## 日志排查指引（每个步骤都会输出关键执行细节）

工作流每个步骤都会在日志中打印详细的执行内容，方便对照排查：

| 步骤 | 日志中会看到的关键输出 |
|---|---|
| 1 解析 SDK 标签 | 输入参数、API 认证状态、匹配到的全部标签、最终选中的 `SDK_TAG` / `SDK_ASSET` |
| 2 下载并解压 SDK | 下载地址/大小、SDK 顶层结构、**SDK 自带 feeds 配置（修改前）** |
| 4 克隆插件仓库 | 克隆 HEAD、firmware 条目数、**修改后 feeds.conf.default 完整内容**（含 `src-link custom` 行、luci master 固定结果） |
| 5 更新并安装 feeds | update/install 是否成功（失败会打 `::error::`）、`feeds/` 目录内容、**luci feed 克隆状态**、**custom feed 安装后的包列表**、**install 后 package/feeds/ 结构**、专项检查 `luci-app-lucky` 是否存在 |
| 6 make defconfig | defconfig 成功与否、`.config` 生成情况 |
| 7 make download | 预下载成功与否、`dl/` 大小（部分失败不中断） |
| 8 编译插件 | 拆分后的插件列表、**重名检测与删除过程**、**每个候选路径的查找过程**（存在/不存在/find 结果）、命中路径的 `PKG_NAME`、编译失败重试 |
| 10 收集产物 | 收集的 `.ipk` 数量与文件列表 |

**"未找到插件"时先看两个地方**：
1. 步骤5 日志里的 `feeds install 后 package/feeds/ 结构`——确认你的插件是否真的被 install（若 feeds install 失败会有 `::error::`）；
2. 步骤8 日志里的候选路径查找过程——确认插件实际所在的位置（可能是 `package/feeds/luci/applications/` 这类三级路径，或 SDK 核心 `package/` 下）。

## 常见问题

- **编译聚合仓库中的单个插件**：`firmware_repo` 填仓库地址，`plugin_name` 填目标插件目录名即可（如 `luci-app-openclash`），同仓库依赖会自动带上。
- **一次编译多个插件**：`plugin_name` 用空格分隔即可，如 `luci-app-openclash luci-app-passwall2`（逗号也兼容）；任一插件名错误会导致任务失败并列出未找到的插件名。
- **插件依赖 luci/packages 源**：工作流会自动 `feeds update/install`（luci 固定为 LEDE master 分支）；feeds 更新失败不会中断编译，但可能因缺依赖而编译失败，请查看日志。
- **SDK 解压后报 `menuconfig` / `Error opening terminal: unknown.`**：SDK 包本身不带 `.config`，工作流已在编译前自动执行 `make defconfig`（非交互）并内置了对 `base-files <-> busybox-selinux` kconfig 递归依赖的兜底补丁，正常无需干预。
- **固件仓库是私有的**：默认按公开仓库处理（克隆不需要 token）。私有仓库需在仓库 Settings → Secrets 中添加 `GH_PAT`（具有读权限的 Personal Access Token），并在工作流中改用 `https://x-access-token:${GH_PAT}@github.com/...` 克隆。
- **GitHub API 限流**：工作流对 GitHub API 的调用全程使用自动注入的 `GITHUB_TOKEN` 认证（权限为 `contents: read`，仅读 Release 信息，安全无密钥），配额充足，不会触发匿名限流（403 / curl exit 22）。
- **为什么编译时会先下载 linux-firmware（几百 MB）而不是直接编插件**：`make package/<名>/compile` 会先构建该包**全部依赖闭包**（日志里 `luci-compat` 等依赖包会先编译），linux-firmware 属于依赖/默认启用集合的一部分，是正常前置准备，插件本体在依赖就绪后编译。为免每次重复下载，工作流已对 SDK 的 `dl/` 目录启用 GitHub Actions 缓存（按 SDK 标签区分），同一标签第二次起直接命中缓存，不再重复下载。
