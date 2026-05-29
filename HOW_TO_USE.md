# 如何推送到 GitHub 并使用工作流

## 目录
1. [前置准备](#1-前置准备)
2. [在 GitHub 创建仓库](#2-在-github-创建仓库)
3. [初始化并推送代码](#3-初始化并推送代码)
4. [工作流触发方式](#4-工作流触发方式)
5. [查看构建结果](#5-查看构建结果)
6. [下载产物](#6-下载产物)
7. [发布 Release](#7-发布-release)
8. [常见问题](#8-常见问题)

---

## 1. 前置准备

### 安装 Git
```bash
# 确认 Git 已安装
git --version
# 输出类似：git version 2.x.x
```

### 配置 Git 用户信息（首次使用）
```bash
git config --global user.name  "你的名字"
git config --global user.email "你的邮箱@example.com"
```

### 登录 GitHub
推荐使用 **Personal Access Token（PAT）** 代替密码：
1. 打开 https://github.com/settings/tokens
2. 点击 **Generate new token (classic)**
3. 勾选 `repo`（完整仓库权限）
4. 生成后复制 Token，**只显示一次，妥善保存**

---

## 2. 在 GitHub 创建仓库

1. 打开 https://github.com/new
2. 填写仓库名，例如 `onnxruntime-aarch64-build`
3. 选择 **Public** 或 **Private**（工作流两种都支持）
4. **不要勾选** "Add a README file"（本地已有文件，避免冲突）
5. 点击 **Create repository**

创建成功后，页面会显示仓库的远端地址，格式为：
```
https://github.com/你的用户名/onnxruntime-aarch64-build.git
```

---

## 3. 初始化并推送代码

在 Git Bash 中，进入工作目录（工作流文件所在的目录）：

```bash
# 进入项目根目录（.github 目录所在的地方）
cd /c/Users/weishian/WorkBuddy/2026-05-29-10-50-03

# 初始化 Git 仓库
git init

# 将所有文件加入暂存区
git add .

# 提交第一个版本
git commit -m "feat: add GitHub Actions workflow for cross-compiling ONNXRuntime 1.15 (AArch64)"

# 将默认分支重命名为 main（与工作流中 branches 保持一致）
git branch -M main

# 添加远端仓库地址（替换为你自己的地址）
git remote add origin https://github.com/你的用户名/onnxruntime-aarch64-build.git

# 推送到 GitHub
git push -u origin main
```

推送时会弹出认证窗口，用户名填 GitHub 账号，密码填第 1 步生成的 **PAT Token**。

---

## 4. 工作流触发方式

工作流文件位于：`.github/workflows/build-onnxruntime-aarch64.yml`

共支持 **4 种触发方式**：

### 方式一：推送到 main/master 分支（自动触发）
```bash
# 修改任意文件后提交并推送，工作流自动启动
git add .
git commit -m "chore: trigger build"
git push origin main
```
> 推送成功后，打开 GitHub → 仓库页面 → **Actions** 标签，即可看到正在运行的任务。

---

### 方式二：提交 Pull Request（自动触发）
向 `main` 或 `master` 分支提交 PR 时，工作流自动运行，可用于 CI 验证。

---

### 方式三：手动触发（workflow_dispatch）
不需要改代码，直接在 GitHub 网页上手动触发：

1. 打开仓库页面，点击顶部 **Actions** 标签
2. 左侧找到 **Cross-Compile ONNXRuntime 1.15 for AArch64**
3. 点击右侧 **Run workflow** 按钮
4. 选择分支（默认 `main`），点击绿色 **Run workflow**

![手动触发示意](https://docs.github.com/assets/cb-87234/mw-1440/images/help/actions/workflow-dispatch-button.webp)

---

### 方式四：打 Tag 触发（自动构建 + 发布 Release）
```bash
# 打一个版本标签
git tag v1.0.0

# 推送标签到 GitHub（触发工作流 + 自动发布 Release）
git push origin v1.0.0
```
> Tag 以 `v` 开头即可触发，例如 `v1.0.0`、`v2.3.1-beta`。

---

## 5. 查看构建结果

1. GitHub 仓库页面 → **Actions** 标签
2. 点击最新一次 workflow run（绿色 ✅ 成功 / 红色 ❌ 失败 / 黄色 🟡 运行中）
3. 点击 **build** job，展开各步骤查看日志

重点关注的步骤日志：
| 步骤 | 关注内容 |
|------|----------|
| Add toolchain to PATH | `aarch64-linux-gnu-gcc --version` 输出，确认工具链加载成功 |
| Configure ONNX Runtime | CMake 配置是否报错 |
| Build ONNX Runtime | 编译输出，Error/Warning |
| Verify output binary architecture | 是否输出 `✅ Architecture check passed: AArch64` |

---

## 6. 下载产物

构建成功后，产物（`.tar.gz`）上传为 GitHub Artifact，保留 7 天。

**下载步骤：**
1. Actions → 点击成功的 workflow run
2. 页面底部找到 **Artifacts** 区域
3. 点击 `onnxruntime-v1.15.1-aarch64-linux-gnu` 下载压缩包

**产物目录结构：**
```
onnxruntime-aarch64-install/
├── include/
│   └── onnxruntime/
│       ├── onnxruntime_c_api.h       # C API 头文件（嵌入式常用）
│       ├── onnxruntime_cxx_api.h     # C++ API 头文件
│       └── ...
└── lib/
    ├── libonnxruntime.so             # 主库（符号链接）
    ├── libonnxruntime.so.1.15.1      # 带版本号的实际文件
    └── cmake/
        └── onnxruntime/              # CMake find_package 支持文件
```

---

## 7. 发布 Release

打 Tag 后工作流会自动通过 `softprops/action-gh-release` 创建 Release 并附加产物文件。

手动操作也可以：
1. GitHub 仓库页面 → **Releases** → **Create a new release**
2. 选择已有 Tag 或新建 Tag
3. 填写版本说明，上传 `.tar.gz` 文件
4. 点击 **Publish release**

---

## 8. 常见问题

### Q1：push 报错 `failed to push: remote rejected`
```bash
# 原因：远端有文件（README）本地没有，先拉取再推
git pull origin main --rebase
git push origin main
```

### Q2：Actions 报错 `Permission denied` 或 `403`
- 检查 PAT Token 是否有 `repo` 权限
- 检查仓库是否设置了 Actions 权限：Settings → Actions → General → Allow all actions

### Q3：工具链下载失败（网络超时）
- Linaro 官网有时不稳定，可以将工具链 tar.xz 手动上传到仓库的 Release 或自建 CDN
- 修改工作流中的 `TOOLCHAIN_URL` 指向自己的下载地址

### Q4：编译时间太长
- 首次构建约 **40~60 分钟**（包含克隆源码 + 编译）
- 第二次起由于缓存工具链和源码，约 **25~35 分钟**
- GitHub Actions 免费账户每月有 **2000 分钟** 额度（公开仓库无限制）

### Q5：如何修改 ONNX Runtime 版本
编辑工作流文件顶部的环境变量：
```yaml
env:
  ONNX_VERSION: "v1.15.1"   # 改为目标版本，如 v1.16.3
```
修改后提交推送即可触发新的构建。
