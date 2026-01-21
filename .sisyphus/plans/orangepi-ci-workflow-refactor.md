# Orange Pi CI Workflow 重构计划

## 📋 项目概述

### 目标
将现有的 `.github/workflows/A733-ci.yml`（基于 OpenWrt 构建的 CI 配置）重构为适用于 Orange Pi Build 项目的 CI/CD 工作流。

### 编译目标
```bash
sudo ./build.sh BOARD=orangepi4pro BRANCH=current RELEASE=jammy \
  BUILD_OPT=image BUILD_DESKTOP=no BUILD_MINIMAL=no KERNEL_CONFIGURE=no
```

## 🔍 技术分析

### 当前工作流问题
1. **构建系统不匹配**：A733-ci.yml 针对 OpenWrt（使用 make），而 Orange Pi 使用 build.sh
2. **依赖安装不适用**：OpenWrt 依赖与 Orange Pi 构建依赖差异大
3. **构建流程不同**：OpenWrt 需要 feeds update/config，Orange Pi 不需要
4. **产物位置不同**：OpenWrt 输出到 `bin/targets/`，Orange Pi 输出到 `output/images/`

### Orange Pi Build 系统特点
1. **需要 sudo 权限**：build.sh 脚本必须以 root 或 sudo 运行
2. **自动依赖安装**：build.sh 会通过 `prepare_host_basic()` 自动安装依赖
3. **构建产物位置**：`output/images/{version}/*.img*` (最终镜像位于版本子目录), `output/debs/` (.deb 包)
4. **源码缓存位置**：`external/cache/sources/` (约5-10GB)
5. **构建时间**：首次构建 2-4 小时，增量编译 30-60 分钟

### GitHub Actions 限制与最佳实践
1. **超时限制**：6 小时（360 分钟）
2. **磁盘空间**：ubuntu-22.04 runner 约 14GB 可用空间
3. **Artifact 限制**：单个文件最大 5GB，总存储 500MB（免费）
4. **最佳实践**：
   - 必须使用 `DEBIAN_FRONTEND=noninteractive` 避免 apt 交互提示
   - 使用 `free-disk-space` action 释放 20-30GB 空间
   - 使用缓存加速重复构建
   - 使用 `timeout-minutes` 设置合理的超时时间

## 📐 详细重构计划

### 阶段 1：创建新的 workflow 文件

#### 1.1 基本配置
```yaml
name: Orange Pi Build CI

# 触发条件：支持 push 和手动触发
on:
  push:
    branches:
      - next  # 只监听 next 分支
  workflow_dispatch:
    inputs:
      board:
        description: 'Orange Pi 板型'
        required: false
        default: 'orangepi4pro'
        type: string

jobs:
  build:
    runs-on: ubuntu-22.04
    timeout-minutes: 360  # 6 小时超时
```

#### 1.2 环境清理（释放磁盘空间）
```yaml
    steps:
      # 检出代码
      - name: 检出代码仓库
        uses: actions/checkout@v4
        with:
          fetch-depth: 0  # 完整历史记录，用于版本标记

      # 释放磁盘空间（关键步骤）
      - name: 释放磁盘空间
        uses: jlumbroso/free-disk-space@main
        with:
          # 保留 Python/Node.js，因为构建脚本可能需要
          android: true
          dotnet: true
          haskell: true
          large-packages: true
          docker-images: true
          swap-storage: true
          tool-cache: false  # 保留构建工具
```

**说明**：
- 释放空间约 20-30GB，为大型固件构建提供足够空间
- `tool-cache: false` 确保 Python、Node.js 等工具可用

### 阶段 2：构建缓存策略

#### 2.1 工具链缓存
```yaml
      # 缓存工具链（加速 5-15 分钟）
      - name: 缓存交叉编译工具链
        uses: actions/cache@v4
        with:
          path: |
            toolchains/
            external/cache/debs/
          key: ${{ runner.os }}-toolchain-orangepi4pro-${{ hashFiles('external/config/boards/orangepi4pro.conf') }}
          restore-keys: |
            ${{ runner.os }}-toolchain-orangepi4pro-
            ${{ runner.os }}-toolchain-
```

#### 2.2 ccache 缓存
```yaml
      # 配置 ccache（加速 50-90%）
      - name: 设置 ccache
        uses: hendrikmuhs/ccache-action@main
        with:
          key: ${{ runner.os }}-ccache-orangepi4pro-${{ github.sha }}
          restore-keys: |
            ${{ runner.os }}-ccache-orangepi4pro-
            ${{ runner.os }}-ccache-
          max-size: 5G
```

### 阶段 3：依赖安装与构建

#### 3.1 安装基础依赖
```yaml
      # 安装编译依赖（build.sh 会自动安装大部分依赖）
      - name: 安装基础依赖
        run: |
          sudo DEBIAN_FRONTEND=noninteractive apt-get update -qq
          sudo DEBIAN_FRONTEND=noninteractive apt-get install -y \
            --no-install-recommends \
            build-essential git wget curl ca-certificates \
            qemu-user-static ccache rsync jq pigz pixz
```

**说明**：
- 使用 `DEBIAN_FRONTEND=noninteractive` 避免交互提示
- `--no-install-recommends` 减少不必要的依赖
- build.sh 的 `prepare_host_basic()` 会自动安装更多依赖

#### 3.2 执行构建
```yaml
      # 编译 Orange Pi 镜像
      - name: 编译 Orange Pi 镜像
        env:
          BOARD: ${{ github.event.inputs.board || 'orangepi4pro' }}
        run: |
          echo "开始编译 Orange Pi ${BOARD}"
          echo "================================"
          
          # 设置 ccache
          export CCACHE_DIR=$HOME/.ccache
          export PATH="/usr/lib/ccache:$PATH"
          
          # 显示初始磁盘空间
          df -h /
          
          # 执行构建（使用 sudo）
          sudo ./build.sh \
            BOARD=${BOARD} \
            BRANCH=current \
            RELEASE=jammy \
            BUILD_OPT=image \
            BUILD_DESKTOP=no \
            BUILD_MINIMAL=no \
            KERNEL_CONFIGURE=no
          
          # 显示构建后磁盘空间
          echo "================================"
          echo "构建完成，磁盘使用情况："
          df -h /
          df -h ./output/images
```

**关键点**：
- 必须使用 `sudo` 运行 build.sh
- 设置 ccache 环境变量加速编译
- 显示磁盘使用情况便于调试

### 阶段 4：构建产物处理

#### 4.1 准备源码压缩包（包含完整源码缓存）
```yaml
      # 准备源码压缩包（用于本地重新编译）
      - name: 准备源码压缩包
        run: |
          mkdir -p ./artifacts
          
          # 创建源码包列表
          echo "准备源码包（包含完整源码缓存）："
          echo "  - 配置文件: external/config/"
          echo "  - 构建脚本: scripts/"
          echo "  - 源码缓存: external/cache/sources/（约 5-10GB）"
          echo "  - 预编译包: external/cache/debs/（约 100MB）"
          echo "  - 板型配置: external/config/boards/orangepi4pro.conf"
          echo ""
          echo "注意：源码包较大（约 5-10GB），但支持完全离线编译"
          
          # 创建源码压缩包（包含完整源码缓存）
          tar -I 'xz -9' -cf ./artifacts/orangepi4pro-sources.tar.xz \
            --exclude='.git' \
            --exclude='output/images' \
            --exclude='output/debug' \
            --exclude='output/.tmp' \
            --exclude='external/cache/.aria2' \
            --exclude='external/cache/rootfs' \
            external/config/ \
            external/cache/sources/ \
            external/cache/debs/ \
            scripts/ \
            build.sh \
            LICENSE \
            README.md
          
          echo ""
          echo "源码包大小："
          du -h ./artifacts/orangepi4pro-sources.tar.xz
```

**说明**：
- 包含完整的 `external/cache/sources/` 源码缓存（约 5-10GB）
- 包含 `external/cache/debs/` 预编译包（约 100MB）
- 支持完全离线编译，无需重新下载源码
- 使用 `xz -9` 高压缩比减小文件大小
- 预期压缩后大小：约 2-4GB（压缩率 60-70%）

#### 4.2 上传镜像文件
```yaml
      # 上传构建镜像
      # 注意：镜像文件位于版本子目录中（如 output/images/Orangepi4pro_1.0.4_ubuntu_jammy_server_linux5.15.147/）
      # 需要使用递归通配符 **/ 来匹配子目录中的文件
      - name: 上传 Orange Pi 镜像
        uses: actions/upload-artifact@v4
        with:
          name: orangepi4pro-image-${{ github.sha }}
          path: |
            output/images/**/*.img*
            output/images/**/*.sha
          retention-days: 30
          compression-level: 0  # 文件已压缩，跳过再次压缩
          if-no-files-found: warn
```

#### 4.3 上传源码压缩包
```yaml
      # 上传源码压缩包（包含完整源码缓存）
      - name: 上传源码压缩包
        uses: actions/upload-artifact@v4
        with:
          name: orangepi4pro-sources-${{ github.sha }}
          path: ./artifacts/orangepi4pro-sources.tar.xz
          retention-days: 30
          compression-level: 0  # xz 已压缩，跳过再次压缩
```

#### 4.4 上传构建日志
```yaml
      # 上传构建日志（便于调试）
      - name: 上传构建日志
        if: always()  # 即使构建失败也上传日志
        uses: actions/upload-artifact@v4
        with:
          name: orangepi4pro-logs-${{ github.sha }}
          path: |
            output/debug/
            *.log
          retention-days: 7
          if-no-files-found: ignore
```

### 阶段 5：构建摘要与通知

#### 5.1 生成构建摘要
```yaml
      # 生成构建摘要
      - name: 生成构建摘要
        if: always()
        run: |
          echo "# Orange Pi 构建摘要" >> $GITHUB_STEP_SUMMARY
          echo "" >> $GITHUB_STEP_SUMMARY
          echo "**板型**: ${{ github.event.inputs.board || 'orangepi4pro' }}" >> $GITHUB_STEP_SUMMARY
          echo "**发行版**: Ubuntu Jammy (22.04)" >> $GITHUB_STEP_SUMMARY
          echo "**内核分支**: current" >> $GITHUB_STEP_SUMMARY
          echo "**构建类型**: 镜像 (无桌面)" >> $GITHUB_STEP_SUMMARY
          echo "" >> $GITHUB_STEP_SUMMARY
          
          if [ -d "output/images" ]; then
            echo "## 构建产物" >> $GITHUB_STEP_SUMMARY
            ls -lh output/images/ >> $GITHUB_STEP_SUMMARY || true
          fi
          
          echo "" >> $GITHUB_STEP_SUMMARY
          echo "## 磁盘使用" >> $GITHUB_STEP_SUMMARY
          df -h / >> $GITHUB_STEP_SUMMARY
```

## 🔧 实施注意事项

### 1. 文件权限问题
**问题**：GitHub Actions 运行在非 root 用户下，但 build.sh 需要 sudo

**解决方案**：
- 在所有需要 root 权限的命令前添加 `sudo`
- 设置正确的文件所有权：`sudo chown -R $USER:$USER .git`
- 使用 `sudo -E` 保留环境变量

### 2. 磁盘空间管理
**问题**：完整的 Orange Pi 构建可能需要 10-15GB 磁盘空间

**解决方案**：
- ✅ 必须在构建前使用 `free-disk-space` action
- ✅ 定期清理临时文件和旧缓存
- ✅ 监控磁盘使用情况：`df -h /`

### 3. 构建时间优化
**问题**：首次构建可能需要 2-4 小时

**解决方案**：
- ✅ 使用 ccache 加速增量构建（50-90% 速度提升）
- ✅ 缓存工具链和依赖包（节省 5-15 分钟）
- ✅ 使用并行编译：`make -j$(nproc)`

### 4. Artifact 上传限制
**问题**：GitHub Actions 单个 artifact 最大 5GB，源码包可能超过此限制

**解决方案**：
- ✅ 使用 `xz -9` 高压缩比（压缩率 60-70%）
- ✅ 如果源码包仍超过 5GB，考虑使用外部存储（S3、云盘）
- ✅ 将大文件分割为多个小 artifacts（镜像、源码、日志）

### 5. 路径匹配问题
**问题**：Orange Pi 构建系统将镜像文件存储在版本子目录中（如 `output/images/Orangepi4pro_1.0.4_ubuntu_jammy_server_linux5.15.147/`），但 GitHub Actions 的默认通配符只匹配直接子目录

**解决方案**：
- ✅ 使用递归通配符 `**/*.img*` 和 `**/*.sha` 匹配任意深度的子目录
- ✅ 在注释中说明镜像文件的实际存储位置
- ✅ 测试路径匹配模式确保正确捕获所有文件

### 5. 构建失败处理
**问题**：构建可能因各种原因失败

**解决方案**：
- ✅ 使用 `if: always()` 确保日志上传
- ✅ 提供详细的构建摘要
- ✅ 记录所有环境变量和构建配置
- ✅ 添加错误处理和重试机制

## 📝 中文注释策略

### 需要添加注释的位置（选项 B - 关键部分）

1. **Workflow 头部**：说明整个工作流的目的
2. **触发条件**：解释为什么同时支持 push 和 workflow_dispatch
3. **关键步骤**：
   - 磁盘空间释放：为什么需要（固件构建需要大量空间）
   - ccache 配置：为什么能加速构建
   - sudo 使用：为什么 build.sh 需要 root 权限
   - 构建参数：每个参数的含义
4. **产物上传**：
   - 区分镜像文件和源码包的用途
   - 源码包包含完整源码缓存（支持离线编译）
5. **错误处理**：关键的安全检查

### 不需要注释的部分
- 标准的 GitHub Actions uses 步骤（如 checkout@v4）
- 简单的 shell 命令（如 ls, cat, echo）
- 明显的配置（如 timeout-minutes: 360）

## ✅ 验证方法

### 1. 功能验证
- [ ] Workflow 成功触发（push 到 next 和手动触发）
- [ ] 构建成功完成（无错误）
- [ ] 镜像文件正确生成（`output/images/*.img*`）
- [ ] 源码压缩包正确生成（`artifacts/*.tar.xz`）
- [ ] 源码包包含 `external/cache/sources/`
- [ ] Artifacts 成功上传

### 2. 性能验证
- [ ] 首次构建时间 < 4 小时
- [ ] 增量构建时间 < 1 小时（启用 ccache 后）
- [ ] 磁盘空间使用合理（< 50GB）
- [ ] Artifact 大小在限制范围内（< 5GB）
- [ ] 源码包支持完全离线编译

### 3. 质量验证
- [ ] 中文注释清晰易懂
- [ ] 错误日志完整可追溯
- [ ] 构建摘要信息丰富
- [ ] Workflow 符合 GitHub Actions 最佳实践

### 4. 本地复现验证
- [ ] 下载的源码包可以解压
- [ ] 使用源码包可以成功重新编译
- [ ] 重新编译的镜像与 CI 构建的一致
- [ ] 支持离线编译（无需网络连接）

## 🚀 实施步骤

### Step 1: 备份原有文件
```bash
cp .github/workflows/A733-ci.yml .github/workflows/A733-ci.yml.backup
```

### Step 2: 创建新的 workflow 文件
创建 `.github/workflows/orangepi-build.yml`，包含上述所有步骤和中文注释

### Step 3: 测试触发
- 推送一个测试 commit 到 `next` 分支
- 手动触发 workflow_dispatch
- 验证两个触发方式都正常工作

### Step 4: 监控首次构建
- 观察构建时间
- 检查磁盘使用情况
- 验证 artifact 上传
- 确认源码包包含完整源码缓存

### Step 5: 测试缓存效果
- 进行第二次构建（触发相同的 workflow）
- 对比首次和增量构建的时间差异
- 验证 ccache 和工具链缓存是否生效

### Step 6: 产物验证
- 下载 artifact
- 验证镜像文件完整性（检查 SHA256）
- 解压源码包并尝试本地编译
- 测试离线编译能力

### Step 7: 优化调整
- 根据实际构建时间调整 timeout（6 小时合理）
- 根据磁盘使用情况优化缓存策略
- 根据反馈调整中文注释
- 如源码包超过 5GB，考虑外部存储方案

## 📊 预期成果

### 构建性能
- **首次构建**：2-4 小时
- **增量构建**（启用缓存）：30-60 分钟
- **磁盘使用**：20-30GB（释放空间后）
- **Artifact 大小**：
  - 镜像文件：500MB - 2GB
  - 源码包：2-4GB（压缩后，包含完整源码缓存）

### 可维护性
- 清晰的中文注释（关键步骤）
- 完整的构建日志和摘要
- 便于调试的错误处理
- 符合最佳实践的结构

### 可扩展性
- 支持添加其他板型（通过 workflow_dispatch 参数）
- 支持添加其他构建选项（如桌面版本）
- 易于集成其他 CI/CD 功能（如自动 Release）

## 🎯 成功标准

1. ✅ Workflow 能够成功编译 orangepi4pro 镜像
2. ✅ 支持自动（push 到 next）和手动（workflow_dispatch）触发
3. ✅ 成功上传镜像文件和源码压缩包（包含完整源码）
4. ✅ 构建时间在合理范围内（< 4 小时）
5. ✅ 磁盘空间管理有效（不超限）
6. ✅ 关键步骤有清晰的中文注释
7. ✅ 源码包可以用于本地重新编译（支持离线编译）
8. ✅ 遵循 GitHub Actions 最佳实践

---

**文档版本**: 1.1（已根据用户反馈更新）  
**创建日期**: 2026-01-21  
**更新日期**: 2026-01-21  
**计划状态**: 待实施
