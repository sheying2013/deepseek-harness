# DeepSeek Harness Fork 自动同步与 macOS x86_64 云编译流水线

本项目为 `sheying2013/deepseek-harness`（fork 自 `deepseek-ai/deepseek-harness`）配置了双 GitHub Actions 工作流体系，实现：
1. **上游代码与标签自动同步**：定时检测并合并上游 `master` 与 tags，发现新提交时自动调度编译；
2. **macOS x86_64 自动化云编译与发布**：在 GitHub 官方 `macos-15-intel` 真实 x64 环境上构建未签名/未公证本地试用包，执行四重完整性验收，并自动上传 Artifact 与创建 GitHub Release。

---

## 一、工作流概览

| 工作流文件 | 显示名称 | 运行环境 | 默认触发方式 | 核心职责 |
| :--- | :--- | :--- | :--- | :--- |
| `fork-sync-upstream.yml` | `Sync Upstream & Trigger Build` | `ubuntu-latest` | 每 6 小时定时 / 手动 `workflow_dispatch` / `repository_dispatch` | 同步上游代码与标签；检测到更新时触发编译 |
| `fork-macos-x64.yml` | `Build macOS x64 Desktop App` | `macos-15-intel` | 手动 `workflow_dispatch` / 工作流调用 `workflow_call` / 上游同步后自动调度 | 自动化构建、四重验收、Artifact 保存与 Release 发布 |

---

## 二、工作流详细说明

### 1. `fork-sync-upstream.yml`（上游同步与调度）

- **工作机制**：
  1. 通过 `actions/checkout@v4` 拉取 fork 仓库的 `master` 分支完整历史（`fetch-depth: 0`）。
  2. 配置上游远程地址 `https://github.com/deepseek-ai/deepseek-harness.git` 并拉取 `upstream/master`。
  3. 执行 `git merge-base --is-ancestor "$UPSTREAM_SHA" HEAD` 比较分支状态：
     - 若本地已包含上游最新提交，标记 `changed=false`；
     - 若上游存在新提交，执行 `git merge --no-edit "$UPSTREAM_SHA"` 并推送至 fork 的 `master`。
  4. 同步上游的所有 Git Tags（`git fetch upstream --tags` && `git push origin --tags`）。
  5. 若有代码更新（或手动触发勾选了 `force_build`），显式调用 GitHub CLI：
     ```bash
     gh workflow run fork-macos-x64.yml --repo "$GITHUB_REPOSITORY" --ref master
     ```
- **关键设计：同步触发链**：
  GitHub Actions 安全机制明确规定：由默认 `GITHUB_TOKEN` 推送产生的 `push` 事件**不会**触发仓库中的其他 Workflow（防止自动化循环风暴）。因此，同步工作流在检测到代码变动后，通过具备 `actions: write` 权限的 `GITHUB_TOKEN` 显式调用 `gh workflow run` 触发编译工作流，确保链路打通。

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

### 1. 手动触发上游同步与构建
- **Web UI 界面操作**：
  1. 进入仓库页面，点击顶部导航栏的 **Actions**；
  2. 在左侧工作流列表中选择 **`Sync Upstream & Trigger Build`**；
  3. 点击右侧 **Run workflow** 下拉按钮；
  4. 可选择是否勾选 `Force trigger macOS x64 build even if no upstream changes`（即使上游无更新也强制构建）；
  5. 点击绿色 **Run workflow** 按钮启动。
- **GitHub CLI 命令行操作**：
  ```bash
  # 普通同步（仅在发现上游更新时触发编译）
  gh workflow run fork-sync-upstream.yml --ref master

  # 强制同步并立即触发云编译
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

### 1. 修改定时同步频率（Cron）
在 `.github/workflows/fork-sync-upstream.yml` 中修改 `schedule.cron` 字段：
```yaml
on:
  schedule:
    # 示例 1：默认每 6 小时运行一次（第 23 分钟）
    - cron: '23 */6 * * *'

    # 示例 2：每 2 小时运行一次
    # - cron: '0 */2 * * *'

    # 示例 3：每天凌晨 3 点运行一次 (UTC 时间)
    # - cron: '0 3 * * *'
```

### 2. `master` 分支持有的文件（fork 专属，不会被上游覆盖）

fork 的 `master` 只在上游代码之外多出以下文件，上游永远不会碰它们，因此**自动同步不会产生合并冲突**：

| 路径 | 作用 |
| :--- | :--- |
| `.github/workflows/fork-sync-upstream.yml` | 上游同步工作流 |
| `.github/workflows/fork-macos-x64.yml` | macOS x64 云编译工作流 |
| `.github/fork/unsigned-macos-x64.patch` | 唯一一处源码改动：放开上游「macOS 不允许 `--unsigned`」的限制 |
| `.github/fork/README.md` | 本说明 |

**为什么用补丁文件而不是直接改源码？** 上游官方打包流程强制要求 Apple Developer ID 签名与公证（`forceCodeSigning: true`），fork 里没有证书，所以必须放开未签名模式。把改动存成补丁、由编译工作流在构建前 `git apply`，可以让 `master` 始终等于「上游 + CI 文件」，同步永远是快进/无冲突合并。

#### 补丁失效怎么办（唯一的已知失效点）
如果上游某次更新改动了 `apps/desktop/scripts/` 下这四个文件的相关代码，补丁可能应用失败，此时**同步仍然成功**，但云编译会在 `Apply fork unsigned-macOS patch` 步骤报错中止（并提示 reject/上下文不符），不会有半成品产物。

修复方式（本地一条命令即可重新生成补丁）：

```bash
git clone https://github.com/sheying2013/deepseek-harness && cd deepseek-harness
git remote add upstream https://github.com/deepseek-ai/deepseek-harness.git
git fetch upstream master && git checkout -b tmp upstream/master
# 手工按 .github/fork/unsigned-macos-x64.patch 的内容改这四个文件：
#   apps/desktop/scripts/package-target.ts
#   apps/desktop/scripts/prepare-dsh.ts
#   apps/desktop/scripts/electron-builder-config.mjs
#   apps/desktop/scripts/smoke-packaged-runtime.ts
git diff > .github/fork/unsigned-macos-x64.patch   # 覆盖旧补丁
git commit -am "ci: refresh unsigned-macos-x64 patch" && git push origin HEAD:master
```

编译工作流里还有一步 `Verify fork patch markers`，会对四处关键改动做语义断言（例如 `prepare-dsh.ts` 必须含 `DSH_DESKTOP_UNSIGNED !== '1'`），即使 `patch(1)` 带 fuzz 应用成功也必须通过断言，杜绝「补丁错位但构建照跑」的情况。

### 3. 首次启用（fork 的 Actions 需要手动放行一次）

fork 出来的仓库，GitHub 默认不会自动运行继承来的上游工作流。首次需要：

1. 打开 `https://github.com/sheying2013/deepseek-harness/actions`；
2. 若顶部出现 “Workflows aren't being run on this forked repository” 横幅，点击 **I understand my workflows, go ahead and enable them**；
3. 之后可在 Actions 页面手动运行一次 `Sync Upstream & Trigger Build` 验证链路。

另外，GitHub 会在**仓库连续 60 天没有任何活动**时自动停用 `schedule` 定时触发；届时手动跑一次任一工作流即可恢复。

### 4. 上游仓库的其它工作流

fork 里同时存在上游自带的 21 个工作流（`ci.yml`、`release.yml` 等）。本方案不修改、不禁用它们；如果你不想让它们消耗额度或产生噪音通知，可以在 Actions 页面里逐个 Disable workflow。
