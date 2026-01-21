# GitHub Actions Workflow 离线编译修复方案（方案 C：使用系统工具链）

## 📋 方案概述

**核心策略**: 使用 Ubuntu 22.04 系统预装的交叉编译工具链，不依赖外部下载的工具链。

### 选择此方案的理由

1. **Ubuntu 22.04 预装工具链已满足要求**
   - `gcc-arm-linux-gnueabihf` 版本约 9.x（满足 U-Boot 要求 > 6.0）
   - `gcc-aarch64-linux-gnu` 版本约 12.x（满足 Kernel 要求 > 10.0）

2. **解决 GitHub Actions 缓存失败问题**
   - 避免 `toolchains/` 目录不存在的警告
   - 简化缓存逻辑

3. **减少源码包大小**
   - 不打包 toolchains/，节省 500MB-1GB
   - 下载速度更快

4. **简化维护**
   - 使用系统工具链，版本更新方便
   - 无需管理外部工具链下载和缓存

---

## 🎯 需要修改的文件

**唯一文件**: `.github/workflows/orangepi-build.yml`

---

## 📝 详细修改步骤

### Step 1: 移除 toolchains/ 缓存配置

**位置**: Line 65-74

#### 修改前的代码

```yaml
      # 缓存交叉编译工具链和预编译包
      # 说明：此缓存可以节省 5-15 分钟的构建时间
      # 工具链和预编译包不会经常变化，可以长时间缓存
      - name: 缓存交叉编译工具链
        uses: actions/cache@v4
        with:
          path: |
            toolchains/           # ⚠️ 移除此行
            external/cache/debs/  # 预编译的 Debian 包
          key: ${{ runner.os }}-toolchain-orangepi4pro-${{ hashFiles('external/config/boards/orangepi4pro.conf') }}
          restore-keys: |
            ${{ runner.os }}-toolchain-orangepi4pro-
            ${{ runner.os }}-toolchain-
```

#### 修改后的代码

```yaml
      # 缓存预编译包
      # 说明：使用 Ubuntu 22.04 系统预装的交叉编译工具链
      # - gcc-arm-linux-gnueabihf (用于 U-Boot)
      # - gcc-aarch64-linux-gnu (用于 Kernel)
      # 仅缓存预编译的 Debian 包以加速构建
      - name: 缓存预编译包
        uses: actions/cache@v4
        with:
          path: external/cache/debs/  # 只缓存预编译包
          key: ${{ runner.os }}-debs-orangepi4pro-${{ hashFiles('external/config/boards/orangepi4pro.conf') }}
          restore-keys: |
            ${{ runner.os }}-debs-orangepi4pro-
            ${{ runner.os }}-debs-
```

---

### Step 2: 更新 "准备源码压缩包" 步骤

**位置**: Line 169-248

#### 修改前的代码

```yaml
      # 准备源码压缩包（包含完整源码缓存）
      # 说明：
      # - 此压缩包包含完整的构建系统 + 源码缓存 + 预编译包
      # - 下载后可以在本地完全离线编译，无需重新下载源码
      # - 源码缓存 external/cache/sources/ 约 5-10GB
      # - 压缩后预计 2-4GB（压缩率 60-70%）
      - name: 准备源码压缩包
        run: |
          echo "========================================"
          echo "准备源码压缩包"
          echo "========================================"
          echo ""
          echo "包含内容："
          echo "  - 配置文件: external/config/"
          echo "  - 构建脚本: scripts/"
          echo "  - 源码缓存: external/cache/sources/（约 5-10GB）"
          echo "  - 预编译包: external/cache/debs/（约 100MB）"
          echo "  - 主构建脚本: build.sh"
          echo "  - 文档文件: LICENSE, README.md"
          echo ""
          echo "说明：此源码包支持完全离线编译"

          # 创建 artifacts 目录
          mkdir -p ./artifacts
          echo ""
          echo ""

          # 创建源码压缩包（xz 高压缩比）
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
          echo "=== 源码包创建完成 ==="
          du -h ./artifacts/orangepi4pro-sources.tar.xz
```

#### 修改后的代码

```yaml
      # 准备离线源码包（包含完整编译环境）
      # 说明：
      # - 此压缩包包含完整的构建系统 + 实际编译位置的源码
      # - u-boot/ 和 kernel/ 是 fetch_from_repo() 克隆源码的实际位置
      # - toolchains/ 不包含（使用系统工具链）
      # - 保留部分 .git 目录以避免 fetch_from_repo() 的 offline=false bug
      # - 下载后可以在本地完全离线编译，无需网络访问
      # - 预计总大小 3-5GB（压缩后 1.5-3GB，不包含 toolchains）
      - name: 准备离线源码包
        run: |
          echo "========================================"
          echo "准备离线源码包"
          echo "========================================"
          echo ""
          echo "包含内容："
          echo "  - 配置文件: external/config/"
          echo "  - 构建脚本: scripts/"
          echo "  - U-Boot 源码: u-boot/ (实际编译位置，约 2-3GB)"
          echo "  - Kernel 源码: kernel/ (实际编译位置，约 1-2GB)"
          echo "  - 其他源码缓存: external/cache/sources/（约 500MB-1GB）"
          echo "  - 预编译包: external/cache/debs/（约 100MB）"
          echo "  - 用户配置: userpatches/（可选）"
          echo "  - Git 信息: .git/ (部分，约 100-200MB)"
          echo "  - 主构建脚本: build.sh"
          echo "  - 文档文件: LICENSE, README.md"
          echo ""
          echo "❌ 不包含: toolchains/ (使用系统工具链)"
          echo ""
          echo "说明：此源码包支持完全离线编译，无需网络访问"
          echo "       需要本地系统已安装交叉编译工具链"

          # 创建 artifacts 目录
          mkdir -p ./artifacts
          
          # 检查关键目录是否存在
          echo ""
          echo "=== 验证关键目录 ==="
          if [ -d "u-boot" ]; then
            echo "✓ U-Boot 源码目录存在 ($(find u-boot -type f | wc -l) 个文件)"
          else
            echo "✗ 警告: u-boot/ 目录不存在或为空"
          fi
          
          if [ -d "kernel" ]; then
            echo "✓ Kernel 源码目录存在 ($(find kernel -type f | wc -l) 个文件)"
          else
            echo "✗ 警告: kernel/ 目录不存在或为空"
          fi
          
          if [ -d "external/cache/sources" ]; then
            echo "✓ 其他源码缓存存在 ($(find external/cache/sources -type f 2>/dev/null | wc -l) 个文件)"
          else
            echo "✗ 警告: external/cache/sources/ 目录不存在或为空"
          fi
          
          echo ""
          echo "正在创建压缩包..."
          tar -I 'xz -9' -cf ./artifacts/orangepi4pro-sources.tar.xz \
            --exclude='.git/objects' \
            --exclude='.git/logs' \
            --exclude='.git/refs/remotes' \
            --exclude='.ccache' \
            --exclude='.tmp' \
            --exclude='output' \
            --exclude='*.o' \
            --exclude='*.a' \
            --exclude='*.ko' \
            --exclude='*.mod' \
            --exclude='*.order' \
            --exclude='*.symvers' \
            --exclude='System.map' \
            --exclude='vmlinux' \
            external/config/ \
            external/cache/sources/ \
            external/cache/debs/ \
            scripts/ \
            u-boot/ \
            kernel/ \
            userpatches/ \
            build.sh \
            LICENSE \
            README.md

          echo ""
          echo "=== 源码包创建完成 ==="
          ls -lh ./artifacts/orangepi4pro-sources.tar.xz

      # 创建离线编译 README
      # 说明：包含详细的离线编译说明和工具链安装指南
      - name: 创建离线编译说明
        run: |
          echo "========================================"
          echo "创建离线编译说明文档"
          echo "========================================"
          
          cat > ./artifacts/OFFLINE_BUILD_README.md << 'EOF'
      # Orange Pi 离线编译完整指南

      ## 📋 概述

      本指南介绍如何在无网络环境下使用 Orange Pi Build 系统编译镜像。

      ## ⚠️ 前置要求

      ### 安装系统交叉编译工具链

      **重要**: 本源码包**不包含**交叉编译工具链。请手动安装。

      #### Ubuntu/Debian

      ```bash
      # 更新包列表
      sudo apt update

      # 安装交叉编译工具链
      sudo apt install -y gcc-arm-linux-gnueabihf gcc-aarch64-linux-gnu
      ```

      #### 验证工具链

      ```bash
      # 检查 ARMv7 工具链（用于 U-Boot）
      arm-linux-gnueabihf-gcc --version

      # 检查 ARMv8 工具链（用于 Kernel）
      aarch64-linux-gnu-gcc --version
      ```

      **期望输出**:
      - `arm-linux-gnueabihf-gcc` 版本 > 6.0 (Ubuntu 22.04 通常为 9.x 或更高)
      - `aarch64-linux-gnu-gcc` 版本 > 10.0 (Ubuntu 22.04 通常为 12.x 或更高)

      #### 其他发行版

      请参考对应发行版的包管理器安装交叉编译工具链：
      - Fedora: `sudo dnf install arm-linux-gnueabihf-gcc-cs aarch64-linux-gnu-gcc-cs`
      - Arch: `sudo pacman -S arm-linux-gnueabihf-gcc aarch64-linux-gnu-gcc`

      ## 📁 离线编译步骤

      ### Step 1: 下载并解压源码包

      ```bash
      # 从 GitHub Actions Artifacts 下载源码包
      # 下载链接格式: https://github.com/<owner>/<repo>/actions/runs/<run-id>

      # 解压源码包
      tar -xf orangepi4pro-sources.tar.xz

      # 进入构建目录（解压后就是项目根目录）
      cd /path/to/orangepi-build
      ```

      ### Step 2: 验证源码包完整性

      ```bash
      # 检查关键目录
      ls -la u-boot/ kernel/ scripts/ external/config/

      # 应该看到:
      # u-boot/v2018.05-sun60iw2/Makefile
      # kernel/orange-pi-5.15-sun60iw2/Makefile
      # scripts/general.sh
      # external/config/boards/orangepi4pro.conf

      # 检查其他源码
      ls external/cache/sources/

      # 应该看到:
      # orangepi-config/
      # firmware/
      # wiringOP/
      ```

      ### Step 3: 验证系统工具链

      ```bash
      # 验证 ARMv7 工具链
      if command -v arm-linux-gnueabihf-gcc &> /dev/null; then
        VERSION=$(arm-linux-gnueabihf-gcc --version | awk '{print $3}')
        echo "✓ ARMv7 工具链: $VERSION"
      else
        echo "✗ ARMv7 工具链未安装"
        echo "  运行: sudo apt install -y gcc-arm-linux-gnueabihf"
        exit 1
      fi

      # 验证 ARMv8 工具链
      if command -v aarch64-linux-gnu-gcc &> /dev/null; then
        VERSION=$(aarch64-linux-gnu-gcc --version | awk '{print $3}')
        echo "✓ ARMv8 工具链: $VERSION"
      else
        echo "✗ ARMv8 工具链未安装"
        echo "  运行: sudo apt install -y gcc-aarch64-linux-gnu"
        exit 1
      fi
      ```

      ### Step 4: 执行离线编译

      **方法 1: 直接传递参数**

      ```bash
      sudo ./build.sh \
        BOARD=orangepi4pro \
        BRANCH=current \
        RELEASE=jammy \
        BUILD_OPT=image \
        BUILD_DESKTOP=no \
        BUILD_MINIMAL=no \
        KERNEL_CONFIGURE=no \
        OFFLINE_WORK=yes \
        IGNORE_UPDATES=yes
      ```

      **方法 2: 使用配置文件**

      ```bash
      # 创建配置文件
      cat > userpatches/config-offline.conf << 'CONF'
      # Orange Pi Build 离线编译配置
      BOARD=orangepi4pro
      BRANCH=current
      RELEASE=jammy
      BUILD_OPT=image
      BUILD_DESKTOP=no
      BUILD_MINIMAL=no
      KERNEL_CONFIGURE=no
      OFFLINE_WORK=yes
      IGNORE_UPDATES=yes
      CONF

      # 使用配置文件编译
      sudo ./build.sh userpatches/config-offline.conf
      ```

      ### Step 5: 查看编译结果

      ```bash
      # 编译成功的镜像文件
      ls -lh output/images/Orangepi4pro_*.img*

      # 应该看到类似输出:
      # -rw-r--r-- 1 user user 1.2G Jan 21 10:00 Orangepi4pro_1.0.4_ubuntu_jammy_server_linux5.15.147.img
      # -rw-r--r-- 1 user user   65 Jan 21 10:00 Orangepi4pro_1.0.4_ubuntu_jammy_server_linux5.15.147.img.sha
      # -rw-r--r-- 1 user user 420M Jan 21 10:00 Orangepi4pro_1.0.4_ubuntu_jammy_server_linux5.15.147.img.xz
      ```

      ## 🔧 目录结构

      解压后的目录结构：

      ```
      orangepi-build/
      ├── u-boot/                        # ⭐ U-Boot 源码（实际编译位置）
      │   └── v2018.05-sun60iw2/
      │       ├── .git/
      │       ├── Makefile
      │       ├── arch/
      │       ├── drivers/
      │       └── ...
      ├── kernel/                        # ⭐ Kernel 源码（实际编译位置）
      │   └── orange-pi-5.15-sun60iw2/
      │       ├── .git/
      │       ├── Makefile
      │       ├── arch/
      │       ├── drivers/
      │       └── ...
      ├── external/
      │   ├── config/                   # 板型/芯片族配置
      │   │   ├── boards/orangepi4pro.conf
      │   │   ├── sources/families/sun60iw2.conf
      │   │   └── kernel/
      │   └── cache/
      │       ├── sources/               # 其他源码（orangepi-config, firmware 等）
      │       └── debs/                # 预编译包缓存
      ├── scripts/                       # 构建脚本
      │   ├── general.sh
      │   ├── main.sh
      │   ├── compilation.sh
      │   └── ...
      ├── userpatches/                   # 用户自定义配置
      ├── build.sh                       # 主构建脚本
      ├── LICENSE
      ├── README.md
      └── output/                        # 构建产物（编译后生成）
          └── images/
              └── Orangepi4pro_*.img
      ```

      **注意**: `toolchains/` 目录不包含（使用系统工具链）。

      ## ❓ 常见问题

      ### Q1: 提示 "Could not find required toolchain"

      **原因**: 系统未安装交叉编译工具链。

      **解决**:
      ```bash
      sudo apt install -y gcc-arm-linux-gnueabihf gcc-aarch64-linux-gnu
      ```

      ### Q2: 提示 "gnutls_handshake() failed"

      **原因**: 未正确启用离线模式。

      **解决**: 确保设置了 `OFFLINE_WORK=yes`:
      ```bash
      sudo ./build.sh ... OFFLINE_WORK=yes
      ```

      ### Q3: 提示 "No sources found in offline mode"

      **原因**: 源码目录为空或不存在。

      **解决**: 
      - 检查 `u-boot/` 和 `kernel/` 目录是否存在
      - 重新解压源码包
      - 确认下载了完整的源码包

      ### Q4: 工具链版本不满足要求

      **原因**: Ubuntu 22.04 预装的工具链版本应该满足要求，但如果使用旧版本可能不满足。

      **检查工具链版本**:
      ```bash
      arm-linux-gnueabihf-gcc --version
      aarch64-linux-gnu-gcc --version
      ```

      **要求**:
      - `arm-linux-gnueabihf-gcc` > 6.0
      - `aarch64-linux-gnu-gcc` > 10.0

      **解决**:
      ```bash
      # Ubuntu 22.04 预装版本通常满足要求
      # 如果版本过低，可以升级
      sudo apt install -y gcc-arm-linux-gnueabihf gcc-aarch64-linux-gnu
      ```

      ### Q5: 编译过程中尝试访问网络

      **原因**: 可能是 fetch_from_repo() 的 bug，即使 OFFLINE_WORK=yes 也尝试访问网络。

      **临时解决**:
      - 确保源码目录包含 `.git/` 目录（源码包已包含）
      - 设置 `IGNORE_UPDATES=yes` 跳过更新检查

      ### Q6: 编译失败，提示找不到文件

      **原因**: 源码包不完整或解压出错。

      **解决**:
      ```bash
      # 重新下载并解压源码包
      rm -rf orangepi-build/
      tar -xf orangepi4pro-sources.tar.xz

      # 验证完整性
      ls -la u-boot/v2018.05-sun60iw2/Makefile
      ls -la kernel/orange-pi-5.15-sun60iw2/Makefile
      ```

      ## 📊 预期结果

      ### 离线编译成功输出示例

      ```
      * You are working offline.
      * Sources, time and host will not be checked

      Verifying offline source package...
      ✓ 配置文件目录存在 (包含 234 个文件)
      ✓ 构建脚本目录存在 (包含 45 个文件)
      ✓ U-Boot 源码目录存在 (包含 15234 个文件)
      ✓ Kernel 源码目录存在 (包含 48521 个文件)
      ✓ 其他源码缓存存在 (包含 1234 个文件)

      [ o.k. ] Checking git sources [ /home/user/orangepi-build/u-boot v2018.05-sun60iw2 ]
      [ o.k. ] Offline mode - Using existing sources
      [ o.k. ] Compiling u-boot
      [ .... ] U-Boot compilation complete

      [ o.k. ] Checking git sources [ /home/user/orangepi-build/kernel orange-pi-5.15-sun60iw2 ]
      [ o.k. ] Offline mode - Using existing sources
      [ o.k. ] Compiling kernel
      [ .... ] Kernel compilation complete

      [ o.k. ] Image creation complete
      -rw-r--r-- 1 user user 1.2G Orangepi4pro_1.0.4_ubuntu_jammy_server_linux5.15.147.img

      ✅ 离线编译成功！
      ```

      ## 📚 参考资料

      - Orange Pi 官方文档: http://www.orangepi.cn
      - Orange Pi 英文文档: http://www.orangepi.org
      - Ubuntu 交叉编译工具链: https://wiki.ubuntu.com/ArmCrossToolchains
      - ARM 工具链: https://developer.arm.com/tools-and-software/open-source-software/developer-tools/gnu-toolchain/gnu-a

      ## 📝 更新日志

      ### v1.0 (2026-01-21)
      - 初始版本
      - 使用系统工具链方案（方案 C）
      - 包含完整的离线编译指南
      EOF

      # 将 README 添加到源码包
      tar -I 'xz -9' -rf ./artifacts/orangepi4pro-sources.tar.xz \
        --transform='s|artifacts/||' \
        ./artifacts/OFFLINE_BUILD_README.md

      echo ""
      echo "✓ 离线编译说明文档已创建并添加到源码包"
```

---

### Step 3: 添加 "验证离线源码包" 步骤

**位置**: 在 Line 248 之后插入

```yaml
      # 验证离线源码包内容
      # 说明：检查压缩包中是否包含所有必需的目录
      - name: 验证离线源码包
        run: |
          echo "========================================"
          echo "验证离线源码包内容"
          echo "========================================"
          echo ""
          
          # 检查压缩包内容
          echo "=== 压缩包文件列表（部分）==="
          tar -tf ./artifacts/orangepi4pro-sources.tar.xz | grep -E '^(u-boot/|kernel/|scripts/|external/config/)' | head -20
          
          echo ""
          echo "=== 源码包统计 ==="
          echo "U-Boot 文件数: $(tar -tf ./artifacts/orangepi4pro-sources.tar.xz | grep '^u-boot/' | wc -l)"
          echo "Kernel 文件数: $(tar -tf ./artifacts/orangepi4pro-sources.tar.xz | grep '^kernel/' | wc -l)"
          echo "脚本文件数: $(tar -tf ./artifacts/orangepi4pro-sources.tar.xz | grep '^scripts/' | wc -l)"
          echo "配置文件数: $(tar -tf ./artifacts/orangepi4pro-sources.tar.xz | grep '^external/config/' | wc -l)"
          echo "其他源码: $(tar -tf ./artifacts/orangepi4pro-sources.tar.xz | grep '^external/cache/sources/' | wc -l)"
          echo "预编译包: $(tar -tf ./artifacts/orangepi4pro-sources.tar.xz | grep '^external/cache/debs/' | wc -l)"
          
          echo ""
          echo "=== 关键检查 ==="
          
          # 检查是否包含 U-Boot 源码
          if tar -tf ./artifacts/orangepi4pro-sources.tar.xz | grep -q '^u-boot/v2018.05-sun60iw2/Makefile$'; then
            echo "✓ 包含 U-Boot 源码 (v2018.05-sun60iw2)"
          else
            echo "✗ 错误: U-Boot 源码缺失或不完整"
            exit 1
          fi
          
          # 检查是否包含 Kernel 源码
          if tar -tf ./artifacts/orangepi4pro-sources.tar.xz | grep -q '^kernel/orange-pi-5.15-sun60iw2/Makefile$'; then
            echo "✓ 包含 Kernel 源码 (orange-pi-5.15-sun60iw2)"
          else
            echo "✗ 错误: Kernel 源码缺失或不完整"
            exit 1
          fi
          
          # 检查是否包含构建脚本
          if tar -tf ./artifacts/orangepi4pro-sources.tar.xz | grep -q '^scripts/general.sh$'; then
            echo "✓ 包含构建脚本"
          else
            echo "✗ 错误: 构建脚本缺失"
            exit 1
          fi
          
          # 检查是否包含离线编译说明
          if tar -tf ./artifacts/orangepi4pro-sources.tar.xz | grep -q 'OFFLINE_BUILD_README.md'; then
            echo "✓ 包含离线编译说明文档"
          else
            echo "⚠ 警告: 离线编译说明文档缺失"
          fi
          
          echo ""
          echo "✅ 源码包验证通过！"
          echo ""
          echo "源码包大小："
          ls -lh ./artifacts/orangepi4pro-sources.tar.xz
```

---

### Step 4: 更新 "生成构建摘要" 步骤

**位置**: Line 321-332（及后续）

#### 修改前的代码

```yaml
          echo "### 如何使用源码包进行离线编译" >> $GITHUB_STEP_SUMMARY
          echo "" >> $GITHUB_STEP_SUMMARY
          echo "#### 1. 下载并解压源码包" >> $GITHUB_STEP_SUMMARY
          echo "\`\`\`bash" >> $GITHUB_STEP_SUMMARY
          echo "# 下载源码包（从 GitHub Actions Artifacts）" >> $GITHUB_STEP_SUMMARY
          echo "wget https://github.com/your-org/orangepi-build/actions/artifacts/..." >> $GITHUB_STEP_SUMMARY
          echo "" >> $GITHUB_STEP_SUMMARY
          echo "# 解压源码包" >> $GITHUB_STEP_SUMMARY
          echo "tar -xf orangepi4pro-sources.tar.xz" >> $GITHUB_STEP_SUMMARY
          echo "" >> $GITHUB_STEP_SUMMARY
          echo "# 2. 进入构建目录" >> $GITHUB_STEP_SUMMARY
          echo "cd external/orangepi-build" >> $GITHUB_STEP_SUMMARY
          echo "" >> $GITHUB_STEP_SUMMARY
          echo "# 3. 执行编译（无需网络，完全离线）" >> $GITHUB_STEP_SUMMARY
          echo "sudo ./build.sh BOARD=orangepi4pro BRANCH=current RELEASE=jammy \\" >> $GITHUB_STEP_SUMMARY
          echo "  BUILD_OPT=image BUILD_DESKTOP=no BUILD_MINIMAL=no KERNEL_CONFIGURE=no" >> $GITHUB_STEP_SUMMARY
          echo "\`\`\`" >> $GITHUB_STEP_SUMMARY
```

#### 修改后的代码

```yaml
          echo "### 如何使用源码包进行离线编译" >> $GITHUB_STEP_SUMMARY
          echo "" >> $GITHUB_STEP_SUMMARY
          echo "#### ⚠️ 前置要求：安装系统工具链" >> $GITHUB_STEP_SUMMARY
          echo "" >> $GITHUB_STEP_SUMMARY
          echo "本源码包**不包含**交叉编译工具链。" >> $GITHUB_STEP_SUMMARY
          echo "请确保本地系统已安装以下工具链：" >> $GITHUB_STEP_SUMMARY
          echo "" >> $GITHUB_STEP_SUMMARY
          echo "\`\`\`bash" >> $GITHUB_STEP_SUMMARY
          echo "# Ubuntu/Debian" >> $GITHUB_STEP_SUMMARY
          echo "sudo apt update" >> $GITHUB_STEP_SUMMARY
          echo "sudo apt install -y gcc-arm-linux-gnueabihf gcc-aarch64-linux-gnu" >> $GITHUB_STEP_SUMMARY
          echo "" >> $GITHUB_STEP_SUMMARY
          echo "# 验证工具链" >> $GITHUB_STEP_SUMMARY
          echo "arm-linux-gnueabihf-gcc --version" >> $GITHUB_STEP_SUMMARY
          echo "aarch64-linux-gnu-gcc --version" >> $GITHUB_STEP_SUMMARY
          echo "\`\`\`" >> $GITHUB_STEP_SUMMARY
          echo "" >> $GITHUB_STEP_SUMMARY
          echo "**工具链要求**：" >> $GITHUB_STEP_SUMMARY
          echo "- \`arm-linux-gnueabihf-gcc\` 版本 > 6.0 (用于 U-Boot)" >> $GITHUB_STEP_SUMMARY
          echo "- \`aarch64-linux-gnu-gcc\` 版本 > 10.0 (用于 Kernel)" >> $GITHUB_STEP_SUMMARY
          echo "" >> $GITHUB_STEP_SUMMARY
          echo "**Ubuntu 22.04 预装版本**:" >> $GITHUB_STEP_SUMMARY
          echo "- \`arm-linux-gnueabihf-gcc\` ≈ 9.x (满足要求)" >> $GITHUB_STEP_SUMMARY
          echo "- \`aarch64-linux-gnu-gcc\` ≈ 12.x (满足要求)" >> $GITHUB_STEP_SUMMARY
          echo "" >> $GITHUB_STEP_SUMMARY
          echo "#### 1. 下载并解压源码包" >> $GITHUB_STEP_SUMMARY
          echo "" >> $GITHUB_STEP_SUMMARY
          echo "\`\`\`bash" >> $GITHUB_STEP_SUMMARY
          echo "# 从 GitHub Actions Artifacts 下载源码包" >> $GITHUB_STEP_SUMMARY
          echo "wget [下载链接]" >> $GITHUB_STEP_SUMMARY
          echo "" >> $GITHUB_STEP_SUMMARY
          echo "# 解压源码包" >> $GITHUB_STEP_SUMMARY
          echo "tar -xf orangepi4pro-sources.tar.xz" >> $GITHUB_STEP_SUMMARY
          echo "" >> $GITHUB_STEP_SUMMARY
          echo "# 解压后目录结构：" >> $GITHUB_STEP_SUMMARY
          echo "# orangepi-build/" >> $GITHUB_STEP_SUMMARY
          echo "#   ├── u-boot/              (U-Boot 源码)" >> $GITHUB_STEP_SUMMARY
          echo "#   ├── kernel/               (Kernel 源码)" >> $GITHUB_STEP_SUMMARY
          echo "#   ├── scripts/              (构建脚本)" >> $GITHUB_STEP_SUMMARY
          echo "#   ├── external/" >> $GITHUB_STEP_SUMMARY
          echo "#   ├── build.sh" >> $GITHUB_STEP_SUMMARY
          echo "#   └── OFFLINE_BUILD_README.md  (离线编译说明)" >> $GITHUB_STEP_SUMMARY
          echo "\`\`\`" >> $GITHUB_STEP_SUMMARY
          echo "" >> $GITHUB_STEP_SUMMARY
          echo "#### 2. 验证源码包完整性（可选）" >> $GITHUB_STEP_SUMMARY
          echo "" >> $GITHUB_STEP_SUMMARY
          echo "\`\`\`bash" >> $GITHUB_STEP_SUMMARY
          echo "# 检查关键目录" >> $GITHUB_STEP_SUMMARY
          echo "ls -la u-boot/ kernel/ scripts/" >> $GITHUB_STEP_SUMMARY
          echo "" >> $GITHUB_STEP_SUMMARY
          echo "# 验证 U-Boot 源码" >> $GITHUB_STEP_SUMMARY
          echo "ls u-boot/v2018.05-sun60iw2/Makefile" >> $GITHUB_STEP_SUMMARY
          echo "" >> $GITHUB_STEP_SUMMARY
          echo "# 验证 Kernel 源码" >> $GITHUB_STEP_SUMMARY
          echo "ls kernel/orange-pi-5.15-sun60iw2/Makefile" >> $GITHUB_STEP_SUMMARY
          echo "" >> $GITHUB_STEP_SUMMARY
          echo "# 应该看到：" >> $GITHUB_STEP_SUMMARY
          echo "# u-boot/v2018.05-sun60iw2/Makefile" >> $GITHUB_STEP_SUMMARY
          echo "# kernel/orange-pi-5.15-sun60iw2/Makefile" >> $GITHUB_STEP_SUMMARY
          echo "\`\`\`" >> $GITHUB_STEP_SUMMARY
          echo "" >> $GITHUB_STEP_SUMMARY
          echo "#### 3. 验证系统工具链（必需）" >> $GITHUB_STEP_SUMMARY
          echo "" >> $GITHUB_STEP_SUMMARY
          echo "\`\`\`bash" >> $GITHUB_STEP_SUMMARY
          echo "# 检查 ARMv7 工具链（用于 U-Boot）" >> $GITHUB_STEP_SUMMARY
          echo "arm-linux-gnueabihf-gcc --version" >> $GITHUB_STEP_SUMMARY
          echo "" >> $GITHUB_STEP_SUMMARY
          echo "# 检查 ARMv8 工具链（用于 Kernel）" >> $GITHUB_STEP_SUMMARY
          echo "aarch64-linux-gnu-gcc --version" >> $GITHUB_STEP_SUMMARY
          echo "" >> $GITHUB_STEP_SUMMARY
          echo "# 如果未安装，运行：" >> $GITHUB_STEP_SUMMARY
          echo "# sudo apt install -y gcc-arm-linux-gnueabihf gcc-aarch64-linux-gnu" >> $GITHUB_STEP_SUMMARY
          echo "\`\`\`" >> $GITHUB_STEP_SUMMARY
          echo "" >> $GITHUB_STEP_SUMMARY
          echo "#### 4. 执行离线编译" >> $GITHUB_STEP_SUMMARY
          echo "" >> $GITHUB_STEP_SUMMARY
          echo "\`\`\`bash" >> $GITHUB_STEP_SUMMARY
          echo "# 关键：必须设置 OFFLINE_WORK=yes 以禁用网络访问" >> $GITHUB_STEP_SUMMARY
          echo "sudo ./build.sh BOARD=orangepi4pro BRANCH=current RELEASE=jammy \\" >> $GITHUB_STEP_SUMMARY
          echo "  BUILD_OPT=image BUILD_DESKTOP=no BUILD_MINIMAL=no \\" >> $GITHUB_STEP_SUMMARY
          echo "  KERNEL_CONFIGURE=no OFFLINE_WORK=yes IGNORE_UPDATES=yes" >> $GITHUB_STEP_SUMMARY
          echo "" >> $GITHUB_STEP_SUMMARY
          echo "# 或者创建配置文件" >> $GITHUB_STEP_SUMMARY
          echo "cat > userpatches/config-offline.conf << EOF" >> $GITHUB_STEP_SUMMARY
          echo "BOARD=orangepi4pro" >> $GITHUB_STEP_SUMMARY
          echo "BRANCH=current" >> $GITHUB_STEP_SUMMARY
          echo "RELEASE=jammy" >> $GITHUB_STEP_SUMMARY
          echo "BUILD_OPT=image" >> $GITHUB_STEP_SUMMARY
          echo "BUILD_DESKTOP=no" >> $GITHUB_STEP_SUMMARY
          echo "OFFLINE_WORK=yes" >> $GITHUB_STEP_SUMMARY
          echo "IGNORE_UPDATES=yes" >> $GITHUB_STEP_SUMMARY
          echo "EOF" >> $GITHUB_STEP_SUMMARY
          echo "" >> $GITHUB_STEP_SUMMARY
          echo "# 使用配置文件编译" >> $GITHUB_STEP_SUMMARY
          echo "sudo ./build.sh userpatches/config-offline.conf" >> $GITHUB_STEP_SUMMARY
          echo "\`\`\`" >> $GITHUB_STEP_SUMMARY
          echo "" >> $GITHUB_STEP_SUMMARY
          echo "#### 5. 查看编译结果" >> $GITHUB_STEP_SUMMARY
          echo "" >> $GITHUB_STEP_SUMMARY
          echo "\`\`\`bash" >> $GITHUB_STEP_SUMMARY
          echo "# 编译成功的镜像文件" >> $GITHUB_STEP_SUMMARY
          echo "ls -lh output/images/Orangepi4pro_*.img*" >> $GITHUB_STEP_SUMMARY
          echo "\`\`\`" >> $GITHUB_STEP_SUMMARY
          echo "" >> $GITHUB_STEP_SUMMARY
          echo "**注意事项**：" >> $GITHUB_STEP_SUMMARY
          echo "- ⚠️ 必须先安装系统交叉编译工具链（见前置要求）" >> $GITHUB_STEP_SUMMARY
          echo "- 必须设置 \`OFFLINE_WORK=yes\` 以禁用所有网络访问" >> $GITHUB_STEP_SUMMARY
          echo "- 建议同时设置 \`IGNORE_UPDATES=yes\` 以避免更新检查" >> $GITHUB_STEP_SUMMARY
          echo "- 源码包包含了部分 git 信息（.git/HEAD, .git/refs/），避免了 fetch_from_repo() 的 offline=false bug" >> $GITHUB_STEP_SUMMARY
          echo "- 如果仍有网络访问尝试，请检查源码目录是否包含有效的 .git 目录" >> $GITHUB_STEP_SUMMARY
          echo "- 详细说明请参考源码包中的 \`OFFLINE_BUILD_README.md\`" >> $GITHUB_STEP_SUMMARY
```

---

### Step 5: 更新 "上传源码压缩包" 步骤说明

**位置**: Line 226-235

#### 修改前的代码

```yaml
      # 上传源码压缩包
      # 说明：此压缩包包含完整源码缓存，用于本地离线编译
      - name: 上传源码压缩包
        uses: actions/upload-artifact@v4
        with:
          name: orangepi4pro-sources-${{ github.sha }}
          path: ./artifacts/orangepi4pro-sources.tar.xz
          retention-days: 30
          compression-level: 0  # xz 已压缩，跳过再次压缩
          if-no-files-found: error
```

#### 修改后的代码

```yaml
      # 上传源码压缩包
      # 说明：此压缩包包含完整的离线编译环境（不包含 toolchains，使用系统工具链）
      - name: 上传源码压缩包
        uses: actions/upload-artifact@v4
        with:
          name: orangepi4pro-sources-${{ github.sha }}
          path: ./artifacts/orangepi4pro-sources.tar.xz
          retention-days: 30
          compression-level: 0  # xz 已压缩，跳过再次压缩
          if-no-files-found: error
```

---

### Step 6: 更新构建摘要中的源码包信息

**位置**: Line 289-296

#### 修改前的代码

```yaml
          # 源码包信息
          if [ -f "./artifacts/orangepi4pro-sources.tar.xz" ]; then
            echo "## 源码包" >> $GITHUB_STEP_SUMMARY
            echo "- **文件名**: \`orangepi4pro-sources.tar.xz\`" >> $GITHUB_STEP_SUMMARY
            echo "- **大小**: \`\`\`text" >> $GITHUB_STEP_SUMMARY
            du -h ./artifacts/orangepi4pro-sources.tar.xz >> $GITHUB_STEP_SUMMARY
            echo "\`\`\`" >> $GITHUB_STEP_SUMMARY
            echo "- **包含**: 完整源码缓存（支持离线编译）" >> $GITHUB_STEP_SUMMARY
          fi
```

#### 修改后的代码

```yaml
          # 源码包信息
          if [ -f "./artifacts/orangepi4pro-sources.tar.xz" ]; then
            echo "## 源码包" >> $GITHUB_STEP_SUMMARY
            echo "- **文件名**: \`orangepi4pro-sources.tar.xz\`" >> $GITHUB_STEP_SUMMARY
            echo "- **大小**: \`\`\`text" >> $GITHUB_STEP_SUMMARY
            du -h ./artifacts/orangepi4pro-sources.tar.xz >> $GITHUB_STEP_SUMMARY
            echo "\`\`\`" >> $GITHUB_STEP_SUMMARY
            echo "- **包含**:" >> $GITHUB_STEP_SUMMARY
            echo "  - U-Boot 源码 (u-boot/)" >> $GITHUB_STEP_SUMMARY
            echo "  - Kernel 源码 (kernel/)" >> $GITHUB_STEP_SUMMARY
            echo "  - 其他源码缓存 (external/cache/sources/)" >> $GITHUB_STEP_SUMMARY
            echo "  - 构建脚本 (scripts/)" >> $GITHUB_STEP_SUMMARY
            echo "  - 离线编译说明 (OFFLINE_BUILD_README.md)" >> $GITHUB_STEP_SUMMARY
            echo "- **不包含**: toolchains/ (使用系统工具链)" >> $GITHUB_STEP_SUMMARY
            echo "- **用途**: 支持完全离线编译（需安装系统工具链）" >> $GITHUB_STEP_SUMMARY
          fi
```

---

## 📊 修改前后对比

### 缓存策略

| 项目 | 修改前 | 修改后 |
|-----|-------|-------|
| 缓存路径 | `toolchains/`, `external/cache/debs/` | `external/cache/debs/` |
| 缓存 key | `toolchain-orangepi4pro-` | `debs-orangepi4pro-` |
| 工具链来源 | 外部下载 | 系统预装 |

### 源码包内容

| 目录 | 修改前 | 修改后 |
|-----|-------|-------|
| `u-boot/` | ❌ 不包含 | ✅ 包含 |
| `kernel/` | ❌ 不包含 | ✅ 包含 |
| `toolchains/` | ❌ 不包含 | ❌ 不包含（使用系统工具链） |
| `scripts/` | ✅ 包含 | ✅ 包含 |
| `external/config/` | ✅ 包含 | ✅ 包含 |
| `external/cache/sources/` | ✅ 包含 | ✅ 包含 |
| `external/cache/debs/` | ✅ 包含 | ✅ 包含 |
| `userpatches/` | ❌ 不包含 | ✅ 包含 |
| `OFFLINE_BUILD_README.md` | ❌ 不包含 | ✅ 包含 |

### 源码包大小

| 版本 | 大小估算 |
|-----|---------|
| 修改前（假设完整） | 2-4GB |
| 修改后（不包含 toolchains） | 1.5-3GB |
| 减小 | 500MB-1GB |

---

## ✅ 实施检查清单

### 实施前检查

- [ ] 已阅读完整工作计划
- [ ] 理解方案 C 的优势和限制
- [ ] 确认本地系统已安装交叉编译工具链
- [ ] 了解离线编译需要安装系统工具链

### 实施步骤

- [ ] 备份当前的 workflow 文件
  ```bash
  cp .github/workflows/orangepi-build.yml .github/workflows/orangepi-build.yml.backup
  ```

- [ ] **Step 1**: 移除 toolchains/ 缓存 (Line 65-74)
  - [ ] 移除 `toolchains/` 路径
  - [ ] 更新缓存 key 为 `debs-orangepi4pro`
  - [ ] 更新 restore-keys
  - [ ] 更新说明注释

- [ ] **Step 2**: 更新打包逻辑 (Line 169-248)
  - [ ] 添加目录验证代码
  - [ ] 更新打包列表（移除 toolchains/）
  - [ ] 添加编译产物排除规则
  - [ ] 创建 OFFLINE_BUILD_README.md 并添加到包中
  - [ ] 更新说明注释

- [ ] **Step 3**: 添加验证步骤 (Line 248 之后)
  - [ ] 创建新的 step: "验证离线源码包"
  - [ ] 检查压缩包内容
  - [ ] 验证关键目录
  - [ ] 显示统计信息

- [ ] **Step 4**: 更新使用说明 (Line 321+)
  - [ ] 添加工具链安装前置要求
  - [ ] 更新解压后目录结构说明
  - [ ] 添加工具链验证步骤
  - [ ] 更新编译命令（保持 OFFLINE_WORK=yes）
  - [ ] 添加注意事项

- [ ] **Step 5**: 更新上传说明 (Line 226-235)
  - [ ] 更新注释说明不包含 toolchains

- [ ] **Step 6**: 更新源码包信息 (Line 289-296)
  - [ ] 更新包含内容列表
  - [ ] 添加 "不包含: toolchains/"
  - [ ] 更新用途说明

### 实施后测试

- [ ] 提交修改到 next 分支
  ```bash
  git add .github/workflows/orangepi-build.yml
  git commit -m "fix: 改进离线源码包，使用系统工具链（方案 C）

  - 移除 toolchains/ 缓存，避免目录不存在警告
  - 不打包 toolchains/，减少源码包大小（节省 500MB-1GB）
  - 添加 u-boot/ 和 kernel/ 到源码包（实际编译位置）
  - 保留部分 .git 目录以避免 offline=false bug
  - 创建 OFFLINE_BUILD_README.md，包含详细离线编译说明
  - 更新 GitHub Actions 摘要，添加工具链安装前置要求"
  git push origin next
  ```

- [ ] 等待 CI 执行完成

- [ ] 下载生成的源码包

- [ ] 验证源码包内容
  - [ ] 包含 `u-boot/v2018.05-sun60iw2/Makefile`
  - [ ] 包含 `kernel/orange-pi-5.15-sun60iw2/Makefile`
  - [ ] 包含 `OFFLINE_BUILD_README.md`
  - [ ] **不包含** `toolchains/` 目录
  - [ ] 验证 README 内容正确

- [ ] 本地测试离线编译
  - [ ] 解压源码包到新目录
  - [ ] 安装系统工具链:
    ```bash
    sudo apt install -y gcc-arm-linux-gnueabihf gcc-aarch64-linux-gnu
    ```
  - [ ] 验证工具链版本
  - [ ] 设置 `OFFLINE_WORK=yes`
  - [ ] 执行编译
  - [ ] 验证编译成功
  - [ ] 检查生成的镜像文件

---

## 📚 参考资料

### 相关文件

- `.github/workflows/orangepi-build.yml` - GitHub Actions workflow 配置
- `external/config/sources/arm64.conf` - ARM64 架构配置
- `external/config/sources/families/sun60iw2.conf` - Orange Pi 4Pro 配置
- `scripts/main.sh` - 主构建流程 (调用 fetch_from_repo)
- `scripts/general.sh` - fetch_from_repo() 函数实现
- `scripts/compilation.sh` - find_toolchain() 函数实现

### Ubuntu 交叉编译工具链

- 官方文档: https://wiki.ubuntu.com/ArmCrossToolchains
- ARM 工具链: https://developer.arm.com/tools-and-software/open-source-software/developer-tools/gnu-toolchain/gnu-a

---

## ✅ 预期最终效果

### GitHub Actions 构建摘要示例

```
## 构建配置
| 参数 | 值 |
|------|-----|
| 板型 | orangepi4pro |
| 发行版 | Ubuntu Jammy (22.04) |
| 内核分支 | current |
| 构建类型 | 镜像 (无桌面) |
| 触发方式 | push |

## 构建产物
-rw-r--r-- 1 runner docker 1.2G Jan 21 10:00 Orangepi4pro_1.0.4_ubuntu_jammy_server_linux5.15.147.img
-rw-r--r-- 1 runner docker   65 Jan 21 10:00 Orangepi4pro_1.0.4_ubuntu_jammy_server_linux5.15.147.img.sha

## 源码包
- **文件名**: `orangepi4pro-sources.tar.xz`
- **大小**: 2.8GB
- **包含**:
  - U-Boot 源码 (u-boot/)
  - Kernel 源码 (kernel/)
  - 其他源码缓存 (external/cache/sources/)
  - 构建脚本 (scripts/)
  - 离线编译说明 (OFFLINE_BUILD_README.md)
- **不包含**: toolchains/ (使用系统工具链)
- **用途**: 支持完全离线编译（需安装系统工具链）

## 离线编译前置要求
⚠️ 请确保本地系统已安装交叉编译工具链：
```bash
sudo apt install -y gcc-arm-linux-gnueabihf gcc-aarch64-linux-gnu
```

## 磁盘使用
Filesystem      Size  Used Avail Use% Mounted on
/dev/root        89G   40G   49G  45% /

## 构建状态
✅ 构建成功！
```

### 本地离线编译输出示例

```bash
$ # 1. 安装系统工具链
$ sudo apt install -y gcc-arm-linux-gnueabihf gcc-aarch64-linux-gnu
Reading package lists... Done
...
Setting up gcc-aarch64-linux-gnu (12.3.0-1ubuntu1~22.04) ...

$ # 2. 验证工具链
$ arm-linux-gnueabihf-gcc --version
arm-linux-gnueabihf-gcc (Ubuntu 9.4.0-1ubuntu1~20.04.1) 9.4.0
✓ ARMv7 工具链: 9.4.0

$ aarch64-linux-gnu-gcc --version
aarch64-linux-gnu-gcc (Ubuntu 12.3.0-1ubuntu1~22.04) 12.3.0
✓ ARMv8 工具链: 12.3.0

$ # 3. 执行离线编译
$ sudo ./build.sh BOARD=orangepi4pro OFFLINE_WORK=yes IGNORE_UPDATES=yes

* You are working offline.
* Sources, time and host will not be checked

[ o.k. ] Checking git sources [ /home/user/orangepi-build/u-boot v2018.05-sun60iw2 ]
[ o.k. ] Offline mode - Using existing sources
[ o.k. ] Compiling u-boot
[ .... ] U-Boot compilation complete

[ o.k. ] Checking git sources [ /home/user/orangepi-build/kernel orange-pi-5.15-sun60iw2 ]
[ o.k. ] Offline mode - Using existing sources
[ o.k. ] Compiling kernel
[ .... ] Kernel compilation complete

[ o.k. ] Image creation complete
-rw-r--r-- 1 user user 1.2G Orangepi4pro_1.0.4_ubuntu_jammy_server_linux5.15.147.img

✅ 离线编译成功！
```

---

## 🎯 方案优势总结

### ✅ 解决 GitHub Actions 缓存失败问题

- 不再依赖 `toolchains/` 缓存
- 避免 "Path Validation Error" 警告
- 简化缓存逻辑

### ✅ 减小源码包大小

- 不打包 toolchains/，节省 500MB-1GB
- 下载速度更快
- 降低 GitHub Actions 存储

### ✅ 简化维护

- 使用系统工具链，版本更新方便
- 无需管理外部工具链下载和缓存
- 兼容性好（Ubuntu 22.04+）

### ✅ 提供完整文档

- 详细的离线编译说明
- 工具链安装指南
- 常见问题解答

---

## ⚠️ 注意事项

### 1. 离线编译需要手动安装工具链

用户在离线环境编译前，必须先安装系统交叉编译工具链：
```bash
sudo apt install -y gcc-arm-linux-gnueabihf gcc-aarch64-linux-gnu
```

### 2. 系统工具链版本必须满足要求

- `arm-linux-gnueabihf-gcc` > 6.0
- `aarch64-linux-gnu-gcc` > 10.0

Ubuntu 22.04 预装版本应该满足要求。

### 3. 其他发行版的工具链安装

非 Ubuntu/Debian 系统需要参考对应发行版的包管理器安装工具链。

### 4. 源码包大小限制

如果源码包压缩后超过 2GB（GitHub Actions 免费版限制），可能需要：
- 降低压缩级别 (xz -6 而非 -9)
- 分割为多个 artifact
- 或使用 GitHub Release（无大小限制）

根据估算，方案 C 的源码包应该在 1.5-3GB 之间，应该不会超过 2GB 限制。

---

**计划状态**: ✅ 准备就绪  
**唯一修改文件**: `.github/workflows/orangepi-build.yml`  
**预计实施时间**: 20-30 分钟  
**风险等级**: 低（仅修改 CI workflow，不涉及核心构建逻辑）  
**下一步**: 运行 `/start-work` 开始实施
