# Orange Pi CI Workflow 实施指南

## 📋 计划总结

### ✅ 已完成的工作

1. **深入研究项目结构**
   - 分析了 Orange Pi Build 系统的构建流程
   - 理解了 build.sh 的工作原理和依赖关系
   - 探索了构建产物的位置和结构

2. **制定详细重构计划**
   - 创建了完整的重构计划文档
   - 定义了所有技术决策和实施细节
   - 考虑了 GitHub Actions 的限制和最佳实践

3. **创建 workflow 文件**
   - 编写了完整的 YAML 配置文件
   - 添加了详细的中文注释（关键步骤）
   - 实现了所有需求功能

### 📁 文件位置

- **详细计划**: `.sisyphus/plans/orangepi-ci-workflow-refactor.md`
- **Workflow 文件**: `.sisyphus/workflows/orangepi-build.yml`
- **本指南**: `.sisyphus/plans/IMPLEMENTATION_GUIDE.md`

## 🎯 实施步骤

### Step 1: 备份原有 workflow
```bash
cp .github/workflows/A733-ci.yml .github/workflows/A733-ci.yml.backup
```

### Step 2: 应用新的 workflow
```bash
# 复制新的 workflow 文件到正确位置
cp .sisyphus/workflows/orangepi-build.yml .github/workflows/orangepi-build.yml

# 查看文件内容（可选）
cat .github/workflows/orangepi-build.yml
```

### Step 3: 提交并推送到 GitHub
```bash
# 添加文件到 Git
git add .github/workflows/orangepi-build.yml
git add .sisyphus/

# 提交更改
git commit -m "feat: 添加 Orange Pi Build CI workflow

- 支持自动触发（push 到 next 分支）
- 支持手动触发（workflow_dispatch）
- 集成磁盘空间释放（20-30GB）
- 配置构建缓存（toolchain + ccache）
- 上传镜像文件和源码压缩包（包含完整源码缓存）
- 添加详细的中文注释
- 生成构建摘要便于查看"

# 推送到 GitHub
git push origin next
```

### Step 4: 验证 workflow 触发
推送到 `next` 分支后，workflow 会自动触发。

#### 查看构建状态：
1. 访问 GitHub 仓库的 "Actions" 标签页
2. 查看最新的 workflow run
3. 点击进入查看详细日志

#### 预期构建时间：
- **首次构建**: 2-4 小时
- **增量构建**（启用缓存后）: 30-60 分钟

### Step 5: 测试手动触发（可选）
如果需要测试手动触发功能：

1. 访问 GitHub 仓库的 "Actions" 标签页
2. 选择 "Orange Pi Build CI" workflow
3. 点击 "Run workflow" 按钮
4. 选择分支和板型（可选）
5. 点击 "Run workflow" 确认

### Step 6: 下载构建产物
构建成功后，可以在 workflow run 页面底部找到 "Artifacts" 部分。

#### 可用产物：
1. **orangepi4pro-image-{sha}.zip**
   - 包含: `output/images/*.img*` 和 `output/images/*.sha`
   - 用途: 直接烧录到 SD 卡
   
2. **orangepi4pro-sources-{sha}.zip**
   - 包含: 完整源码压缩包（包含 `external/cache/sources/`）
   - 用途: 本地离线编译
   - 大小: 约 2-4GB（压缩后）

3. **orangepi4pro-logs-{sha}.zip**
   - 包含: `output/debug/` 和所有 `*.log` 文件
   - 用途: 调试构建问题

### Step 7: 本地编译测试（可选）
使用下载的源码包进行本地编译：

```bash
# 1. 下载并解压源码包
unzip orangepi4pro-sources-{sha}.zip
tar -xf orangepi4pro-sources.tar.xz

# 2. 进入构建目录
# 注意：解压后的目录结构是 external/config/ 等
# 需要调整路径

# 3. 执行编译（离线模式）
sudo ./build.sh BOARD=orangepi4pro BRANCH=current RELEASE=jammy \
  BUILD_OPT=image BUILD_DESKTOP=no BUILD_MINIMAL=no KERNEL_CONFIGURE=no \
  OFFLINE_WORK=yes
```

## 🔍 验证检查清单

### 功能验证
- [ ] Workflow 成功触发（push 到 next 分支）
- [ ] 构建成功完成（无错误）
- [ ] 镜像文件正确生成（`output/images/*.img*`）
- [ ] 源码压缩包正确生成（包含 `external/cache/sources/`）
- [ ] 所有 artifacts 成功上传

### 性能验证
- [ ] 首次构建时间 < 4 小时
- [ ] 增量构建时间 < 1 小时（启用 ccache 后）
- [ ] 磁盘空间使用合理（< 50GB）
- [ ] Artifact 大小在限制范围内（< 5GB）

### 质量验证
- [ ] 中文注释清晰易懂
- [ ] 构建摘要信息丰富
- [ ] 错误日志完整可追溯
- [ ] Workflow 符合 GitHub Actions 最佳实践

## ⚠️ 常见问题

### Q1: 构建失败，提示磁盘空间不足
**解决方案**：
- 检查 `free-disk-space` step 是否成功执行
- 查看 "检查磁盘空间" step 的输出
- 确认释放空间后至少有 20GB 可用空间

### Q2: 源码包超过 5GB 限制
**解决方案**：
- 如果源码包超过 5GB，GitHub Actions 会拒绝上传
- 考虑以下选项：
  - 使用外部存储（如 S3、云盘）上传大文件
  - 修改源码包内容，排除某些大型源码目录
  - 分割源码包为多个小文件

### Q3: 构建超时（6 小时内未完成）
**解决方案**：
- 检查是否为首次构建（首次需要下载源码，时间较长）
- 查看 "编译 Orange Pi 镜像" step 的日志
- 确认网络连接正常（首次构建需要下载大量源码）

### Q4: ccache 缓存未生效
**解决方案**：
- 检查 "设置 ccache" step 是否成功
- 查看编译日志中是否有 ccache 命令
- 确认 `CCACHE_DIR` 和 `PATH` 环境变量设置正确

### Q5: 手动触发 workflow 时找不到板型配置
**解决方案**：
- 确认板型名称正确（如 `orangepi4pro`）
- 检查 `external/config/boards/` 目录下是否存在对应的 `.conf` 文件
- 板型名称应与配置文件名匹配（不含 `.conf` 后缀）

## 📊 预期结果

### 构建产物
```bash
output/images/
├── OrangePi4Pro_1.0.4_Ubuntu_jammy_current_linux6.1.50.img
├── OrangePi4Pro_1.0.4_Ubuntu_jammy_current_linux6.1.50.img.sha
└── OrangePi4Pro_1.0.4_Ubuntu_jammy_current_linux6.1.50.img.xz

artifacts/
└── orangepi4pro-sources.tar.xz  # 约 2-4GB
```

### GitHub Actions Artifacts
1. **orangepi4pro-image-{sha}.zip** - 镜像文件（500MB-2GB）
2. **orangepi4pro-sources-{sha}.zip** - 源码包（2-4GB）
3. **orangepi4pro-logs-{sha}.zip** - 构建日志（<100MB）

### 构建摘要示例
```
# Orange Pi 构建摘要

## 构建配置
| 参数 | 值 |
|------|-----|
| 板型 | orangepi4pro |
| 发行版 | Ubuntu Jammy (22.04) |
| 内核分支 | current |
| 构建类型 | 镜像 (无桌面) |
| 触发方式 | push |

## 构建产物
-rw-r--r-- 1 runner docker 1.2G Jan 21 10:00 OrangePi4Pro_1.0.4_Ubuntu_jammy_current_linux6.1.50.img
-rw-r--r-- 1 runner docker   65 Jan 21 10:00 OrangePi4Pro_1.0.4_Ubuntu_jammy_current_linux6.1.50.img.sha
-rw-r--r-- 1 runner docker 420M Jan 21 10:00 OrangePi4Pro_1.0.4_Ubuntu_jammy_current_linux6.1.50.img.xz

## 源码包
- **文件名**: `orangepi4pro-sources.tar.xz`
- **大小**: 2.8GB
- **包含**: 完整源码缓存（支持离线编译）

## 磁盘使用
Filesystem      Size  Used Avail Use% Mounted on
/dev/root        89G   45G   44G  51% /

## 构建状态
✅ 构建成功！
```

## 🚀 下一步优化

### 如果一切正常工作，可以考虑：

1. **添加更多板型支持**
   - 修改 workflow_dispatch 的 inputs，添加更多板型选项
   - 或改为 `type: choice` 提供板型列表

2. **支持其他构建选项**
   - 添加 `BUILD_DESKTOP` 参数（桌面版本）
   - 添加 `KERNEL_CONFIGURE` 参数（自定义内核配置）

3. **集成自动 Release**
   - 当 push tag 时自动创建 GitHub Release
   - 将镜像和源码包作为 Release assets 上传

4. **优化缓存策略**
   - 根据实际构建时间调整 ccache 缓存大小
   - 添加更多路径到缓存（如 rootfs 缓存）

5. **添加通知功能**
   - 构建成功/失败时发送通知（Email、Slack、Discord）
   - 集成 Slack 或 Telegram bot

## 📞 需要帮助？

如果遇到任何问题：

1. **查看详细日志**
   - 在 GitHub Actions 页面查看每个 step 的日志
   - 下载 `orangepi4pro-logs-{sha}.zip` artifact

2. **检查配置**
   - 确认 `.github/workflows/orangepi-build.yml` 文件正确
   - 验证 `external/config/boards/orangepi4pro.conf` 存在

3. **参考文档**
   - 阅读完整计划：`.sisyphus/plans/orangepi-ci-workflow-refactor.md`
   - 查看原始 workflow：`.github/workflows/A733-ci.yml.backup`

4. **提交 Issue**
   - 如果发现问题，记录详细的错误信息和日志
   - 附上 workflow run 的链接

---

**准备状态**: ✅ 准备就绪  
**下一步**: 运行 `/start-work` 开始实施
