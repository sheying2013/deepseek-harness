# DeepSeek Harness Fork 自动同步与 macOS x86_64 云编译流水线

本项目为 `sheying2013/deepseek-harness`（fork 自 `deepseek-ai/deepseek-harness`）配置了双 GitHub Actions 工作流体系，实现：
1. **上游代码、标签与 Release 自动同步**：每 10 分钟轮询上游 `master`、tags 和 DSH Releases（含 prerelease）；
2. **每个新上游 DSH Release 自动云编译**：在 `macos-15-intel` 上按发布日期排队构建 x86_64 试用包，验收通过后自动上传 Artifact 并创建 fork Release。

---

## 一、工作流概览

| 工作流文件 | 显示名称 | 运行环境 | 默认触发方式 | 核心职责 |
| :--- | :--- | :--- | :--- | :--- |
| `fork-sync-upstream.yml` | `Sync Upstream & Trigger Build` | `ubuntu-latest` | 每 10 分钟 / 手动 `workflow_dispatch` / `repository_dispatch` | 同步 master/tags，轮询上游 DSH Releases，并排队调用 x64 构建 |
| `fork-macos-x64.yml` | `Build macOS x64 Desktop App` | `macos-15-intel` | 手动 `workflow_dispatch` / 可复用 `workflow_call` | 构建指定上游 Release tag、四重验收、Artifact 与 fork Release 发布 |

---

## 二、工作流详细说明

### 1. `fork-sync-upstream.yml`（上游同步与 Release 构建队列）

- 每 **10 分钟**运行一次；GitHub 不会把上游仓库的 `release` 事件直接转发给 fork，因此用轮询发现新发布，通常 10 分钟内启动。
- 合并上游 `master` 并镜像上游 tags。准备构建前会确认 release tag 已同步到 fork。
- 查询上游所有已发布的 `dsh-v*` Release（包括 prerelease），以 fork 中**最新一个已成功生成 DMG 的版本**为检查点，只选择其后的第一个发布版本。
- 通过 `workflow_call` 直接运行下方 macOS 构建工作流；每个版本串行构建。构建成功并创建 DMG Release 后，下一次轮询才会推进到下一版本；失败的版本会留在队列头部并自动重试。
- 若历史 fork 中没有可识别的成功 DMG 检查点，工作流会 fail closed，不猜测、不一次性补构建全部旧版本。
- 手动运行可勾选 `force_build`，强制重建上游最新的 `dsh-v*` 发布版本。

### 2. `fork-macos-x64.yml`（macOS x86_64 云编译与打包）

- **工作机制**：
  1. **环境准备**：使用 `macos-15-intel` 真实 x86_64 硬件 Runner，Node.js 24，使用 `pnpm/action-setup@v4` 安装项目指定版本 `pnpm@11.7.0`，并通过 `actions/setup-node@v4` 启用 pnpm 缓存加速。
  2. **配置注入**：自动生成打包必需的 `apps/desktop/.env.macos` 模板配置（未签名模式无需真实 Apple 开发者证书与公证凭证）。
  3. **版本号解析**：读取 `apps/desktop/package.json` 的产品版本，通过官方脚本逻辑自动生成符合规范的唯一构建版本号。
  4. **预检与打包**：
     - 运行 `pnpm run package:mac:x64 -- --unsigned --check` 校验工具链配置；
     - 运行 `pnpm run package:mac:x64 -- --unsigned --build-version "$BUILD_VERSION"` 完成桌面端打包，产出已 ad-hoc 签名的 `DeepSeek Harness.app` 与 `.dmg` 安装镜像。
  5. **四重严格验收**（任一失败立即中止并报错）：
     - `hdiutil verify "<dmg>"`：校验 DMG 磁盘镜像结构完整性；
     - `lipo -archs "<main_bin>"`：校验主程序包含 `x86_64` 架构；
     - `file "<node_bin>"`：遍历检查内置 Node 运行时均为 `x86_64` 架构；
     - `codesign --verify --deep --strict "<app>"`：严格验证签名链结构有效性。
  6. **产物分发**：
     - 使用 `actions/upload-artifact@v4` 保存 DMG 产物（保留 30 天）；
     - 若 `create_release` 为 `true`，通过 `gh release create` 创建以 `macos-x64-v<buildVersion>` 为 Tag 的 Pre-release，并附带未签名安装风险与使用提示。

- **关键设计：版本号唯一性**：
  构建版本号默认格式为：`<产品主版本>.<UTC年月日>.<GITHUB_RUN_NUMBER>`（如 `0.2.0-rc.2.20260930.12`）。
  该格式严格兼容官方 `desktop-build-version.mjs` 中的 `validateDesktopBuildVersion` 语义校验规则，同时引入每次运行单调递增的 `GITHUB_RUN_NUMBER`，确保即使同一天多次重跑、重复触发，版本号也具备绝对唯一性，不会发生 Release 标签冲突或产物覆盖。

---

## 三、操作指引

### 1. 手动触发上游同步与 Release 构建
- **Web UI 界面操作**：
  1. 进入仓库页面，点击顶部导航栏的 **Actions**；
  2. 在左侧工作流列表中选择 **`Sync Upstream & Trigger Build`**；
  3. 点击右侧 **Run workflow** 下拉按钮；
  4. 通常不勾选 `force_build`；勾选后会强制重建上游最新的 DSH Release；
  5. 点击绿色 **Run workflow** 按钮启动。
- **GitHub CLI 命令行操作**：
  ```bash
  # 立即检查上游；若有新 Release，自动构建队列中的下一个版本
  gh workflow run fork-sync-upstream.yml --ref master

  # 强制重建上游最新 DSH Release
  gh workflow run fork-sync-upstream.yml --ref master -f force_build=true
  ```

### 2. 单独手动触发 macOS x64 云编译
- **Web UI 界面操作**：
  1. 进入 **Actions** -> 选择 **`Build macOS x64 Desktop App`**；
  2. 点击 **Run workflow**；
  3. 参数项：
     - `Git ref`: 默认为 `master`，亦可指定特定 Tag、分支或 SHA；
     - `Custom build version`: 留空则全自动生成；若指定须遵循官方规范（如 `0.2.0-rc.2.20260930.9`）；
     - `Create a GitHub Release`: 默认为勾选（自动创建 Release）；
  4. 点击 **Run workflow** 启动。
- **GitHub CLI 命令行操作**：
  ```bash
  gh workflow run fork-macos-x64.yml --ref master
  ```

---

## 四、产物获取位置

构建完成后，产物可在以下两个位置获取：
1. **GitHub Releases**（推荐）：
   - 进入仓库首页右侧的 **Releases** 列表；
   - 找到对应版本标签（例如 `macos-x64-v0.2.0-rc.2.20260930.1`）；
   - 在 Assets 列表中直接下载 `.dmg` 文件。
2. **Actions 运行记录 Artifacts**：
   - 进入对应的 Workflow 运行详情页；
   - 滚动至页面最底部 **Artifacts** 区域下载 ZIP 包，解压后即为 `.dmg` 文件。

---

## 五、macOS Intel 本地运行指引（绕过 Gatekeeper）

本云编译流程产出的产物为**未签名 / 未公证**的测试试用包。直接在 macOS 上打开可能会出现：
> 「“DeepSeek Harness.app”已损坏，无法打开。你应该将它移到废纸篓。」或「无法打开，因为 Apple 无法检查其是否包含恶意软件。」

### 解决方法（二选一）：

#### 方法 1：终端执行解除隔离命令（推荐）
将应用拖入 `/Applications` 后，在终端中执行以下命令清除隔离属性：
```bash
xattr -dr com.apple.quarantine "/Applications/DeepSeek Harness.app"
```

#### 方法 2：访达右键绕过
1. 打开访达（Finder），进入「应用程序」目录；
2. 找到 `DeepSeek Harness.app`，**按住 Control 键或右键点击**；
3. 在弹出菜单中选择「**打开**」；
4. 在弹出的系统安全警告窗口中，点击「**打开**」即可正常运行。

---

## 六、进阶配置与常见问题

### 1. 修改上游 Release 检查频率（Cron）
默认每 10 分钟轮询一次，以便在上游发布后尽快开始构建。可在 `.github/workflows/fork-sync-upstream.yml` 中修改 `schedule.cron` 字段：
```yaml
on:
  schedule:
    # 默认每 10 分钟轮询上游 DSH Releases（含 prerelease）
    - cron: '*/10 * * * *'

    # 示例：每小时运行一次
    # - cron: '0 * * * *'
```

### 2. `master` 分支持有的文件（fork 专属，不会被上游覆盖）

fork 的 `master` 只在上游代码之外多出以下文件，上游永远不会碰它们，因此**自动同步不会产生合并冲突**：

| 路径 | 作用 |
| :--- | :--- |
| `.github/workflows/fork-sync-upstream.yml` | 上游同步工作流 |
| `.github/workflows/fork-macos-x64.yml` | macOS x64 云编译工作流 |
| `.github/fork/README.md` | 本说明 |

**不需要改上游源码。** 自 `v0.2.1-alpha.2` 起，上游已经原生支持 macOS 未签名构建（`--unsigned`）：

- `electron-builder-config.mjs`：`identity: unsigned ? '-' : ...`、`notarize: !unsigned`、`target: unsigned ? ['dmg'] : ['dmg','zip']`、产物写入 `unsigned-artifacts/`；
- `package-target.ts`：`--unsigned` 已允许 macOS 目标，产物目录切到 `unsigned-artifacts`；
- `prepare-dsh.ts`：`DSH_DESKTOP_UNSIGNED=1` 时跳过原生 runtime 签名；
- `smoke-packaged-runtime.ts`：未签名产物走 `unsigned-artifacts` 路径。

因此 fork 里**不再需要任何源码补丁**，`master` 始终等于「上游 + 少量 CI 文件」，上游同步永远是快进/无冲突合并。

#### 关于 ad-hoc 重签（fork 独有的一步）

上游的未签名模式用 `identity: '-'` 让 electron-builder 做 ad-hoc 签名，但打包过程会改写 `Contents/Resources/app.asar.unpacked/dsh` 与 `runtime/primary-runtime`，这些路径在 `signIgnore` 名单里不会被重新封装，导致 `codesign --verify --deep --strict` 可能失败。

所以编译工作流在打包之后、验收之前增加了一步 `Ad-hoc re-sign app bundle and rebuild DMG`：

1. `/usr/bin/codesign --force --deep --sign - "<app>"` —— 对整个 App bundle 做深度 ad-hoc 重签；
2. `/usr/bin/codesign --verify --deep --strict --verbose=2 "<app>"` —— 立即自证签名链有效；
3. `/usr/bin/hdiutil create -volname "DeepSeek Harness" -srcfolder "<app>" -format UDZO -ov "<dmg>"` —— 用重签后的 App 重建 DMG（这一步只在 CI 里做，不碰上游源码）。

以后上游如果再次改动未签名相关逻辑，只会影响这一条 CI 步骤，不会再出现「补丁错位导致编译中止」。

### 3. 首次启用（fork 的 Actions 需要手动放行一次）

fork 出来的仓库，GitHub 默认不会自动运行继承来的上游工作流。首次需要：

1. 打开 `https://github.com/sheying2013/deepseek-harness/actions`；
2. 若顶部出现 “Workflows aren't being run on this forked repository” 横幅，点击 **I understand my workflows, go ahead and enable them**；
3. 之后可在 Actions 页面手动运行一次 `Sync Upstream & Trigger Build` 验证链路。

另外，GitHub 会在**仓库连续 60 天没有任何活动**时自动停用 `schedule` 定时触发；届时手动跑一次任一工作流即可恢复。

### 4. 可选但推荐：`FORK_SYNC_TOKEN`（让「同步 tag / 上游改 CI 文件」也能通过）

GitHub 对默认的 `GITHUB_TOKEN` 有一条硬限制：**不允许创建或更新包含 `.github/workflows/**` 变更的 ref**。实测现象：

```text
! [remote rejected] dsh-v0.1.0-rc.7 -> dsh-v0.1.0-rc.7
  (refusing to allow a GitHub App to create or update workflow
   `.github/workflows/build-exe-for-python-sdk.yml` without `workflows` permission)
```

影响面：
- **tag 镜像**：只有不修改 workflow 文件的 tag 能推成功，其余会被拒绝（工作流会打 `::warning::`，不影响 master 同步）；
- **master 同步**：绝大多数提交没问题；但只要上游那次更新**改动了 `.github/workflows/` 下的文件**，`GITHUB_TOKEN` 推送就会被拒，同步工作流会明确报错并给出修复指引。

修复方式（一次性，30 秒）：

1. 生成 token：
   - 经典 PAT：勾选 `repo` + `workflow`；或
   - 细粒度 PAT：仓库选 `sheying2013/deepseek-harness`，权限给 **Contents: Read and write** 与 **Workflows: Read and write**；
2. 仓库 `Settings → Secrets and variables → Actions → New repository secret`，名称填 **`FORK_SYNC_TOKEN`**，值填上面生成的 token；
3. 无需改代码——上游同步工作流已使用 `secrets.FORK_SYNC_TOKEN || secrets.GITHUB_TOKEN`，加了 secret 就自动切换。该 PAT 也用于下游 reusable workflow 构建及创建 fork Release。

> 不配置这个 secret，GitHub 默认 `GITHUB_TOKEN` 可能无法推送包含 `.github/workflows/**` 变更的提交/tag，或触发下游 Actions；自动链路就可能失败。

### 5. 上游仓库的其它工作流

fork 里同时存在上游自带的 21 个工作流（`ci.yml`、`release.yml`、`Sandbox` 等）。它们是给上游自己的基础设施用的，在这个 fork 里只会重复跑、报错并刷通知，因此**已经把除下面两个之外的全部上游工作流设为 disabled**：

- ✅ 保持启用：`Sync Upstream & Trigger Build`（`fork-sync-upstream.yml`）
- ⚙️ 由同步工作流通过 `workflow_call` 调用：`Build macOS x64 Desktop App`（`fork-macos-x64.yml`）；不要手动 Disable，否则手动 `workflow_dispatch` 将不可用
- ⏸️ 已停用：其余 19 个（`CI`、`CI master`、`Release (dsh)`、`Release (vendor)`、`Sandbox`、`E2E ...`、`Issue policy` 等）

需要恢复任意一个：`Actions → 左侧选中该工作流 → 右上角 ⋯ → Enable workflow`；或者用命令行：

```bash
gh workflow list --all --repo sheying2013/deepseek-harness
gh workflow enable "<workflow name>" --repo sheying2013/deepseek-harness
```
