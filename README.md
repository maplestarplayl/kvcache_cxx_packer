# KVCache C++ Packer

这个项目提供了一个自动化的构建系统，用于编译和打包 KVCache 项目所需的所有 C++ 依赖库。

## 📦 包含的库

- **etcd-cpp-apiv3** - etcd C++ API 客户端
- **gflags** - Google 命令行标志库
- **glog** - Google 日志库
- **jsoncpp** - JSON 解析库
- **rdma-core** - RDMA 核心库
- **yalantinglibs** - 高性能 C++ 库集合

## 🏗️ 构建方式

### 本地构建

```bash
# 直接构建（需要 Ubuntu 20.04 环境）
python3 pack.py

# 使用容器构建（推荐）- 自动检测架构
python3 pack_in_container.py

# 指定架构构建
python3 pack_in_container.py --arch amd64    # 构建 AMD64 版本
python3 pack_in_container.py --arch arm64    # 构建 ARM64 版本

# 指定系统和架构
python3 pack_in_container.py --system-name ubuntu22.04 --arch arm64
```

### GitHub Actions 自动构建

#### 1. 测试构建

使用 `test-build.yml` workflow 进行手动测试：

1. 访问 GitHub repository 的 Actions 页面
2. 选择 "Test Build" workflow
3. 点击 "Run workflow"
4. 选择目标架构（amd64 或 arm64）
5. 点击 "Run workflow" 按钮

#### 2. 发布版本

使用 `build-and-release.yml` workflow 自动构建和发布：

1. 创建版本标签：
```bash
git tag v1.0.0
git push origin v1.0.0
```

2. GitHub Actions 将自动：
   - 为 amd64 和 arm64 架构构建包
   - 支持多个系统：Ubuntu 20.04、Ubuntu 22.04、ManyLinux 2014
   - 创建 `output_{system}_{arch}.tar.gz` 文件（例如：`output_ubuntu20.04_amd64.tar.gz`、`output_manylinux_2014_arm64.tar.gz`）
   - 生成 SHA256 校验和
   - 创建 GitHub Release 并上传所有架构的文件

## 📋 输出结果

构建完成后，输出目录包含：

- `output_{system}_{arch}.tar.gz` - 编译好的库文件包（例如：`output_ubuntu20.04_arm64.tar.gz`）
- `output_{system}_{arch}.tar.gz.sha256` - SHA256 校验和
- `build_summary.txt` - 构建摘要
- `build_report.json` - 详细构建报告

## 🚀 使用方法

1. 下载对应架构和系统的包：
```bash
# AMD64 架构示例
wget https://github.com/AI-Infra-Team/kvcache_cxx_packer/releases/download/v1.0.0/output_ubuntu20.04_amd64.tar.gz
wget https://github.com/AI-Infra-Team/kvcache_cxx_packer/releases/download/v1.0.0/output_ubuntu20.04_amd64.tar.gz.sha256
sha256sum -c output_ubuntu20.04_amd64.tar.gz.sha256

# ARM64 架构示例
wget https://github.com/AI-Infra-Team/kvcache_cxx_packer/releases/download/v1.0.0/output_ubuntu20.04_arm64.tar.gz
wget https://github.com/AI-Infra-Team/kvcache_cxx_packer/releases/download/v1.0.0/output_ubuntu20.04_arm64.tar.gz.sha256
sha256sum -c output_ubuntu20.04_arm64.tar.gz.sha256
```

2. 解压并使用：
```bash
# 解压到指定目录
mkdir -p /opt/kvcache-deps
tar -xzf output_ubuntu20.04_amd64.tar.gz -C /opt/kvcache-deps
# 或 ARM64: tar -xzf output_ubuntu20.04_arm64.tar.gz -C /opt/kvcache-deps
```

3. 在 CMake 项目中使用：
```cmake
# 设置依赖路径
set(CMAKE_PREFIX_PATH "/opt/kvcache-deps" ${CMAKE_PREFIX_PATH})

# 查找并链接库
find_package(gflags REQUIRED)
find_package(glog REQUIRED)
find_package(PkgConfig REQUIRED)
pkg_check_modules(JSONCPP jsoncpp)

target_link_libraries(your_target
    gflags::gflags
    glog::glog
    ${JSONCPP_LIBRARIES}
)
```

## 🔧 配置说明

### 包配置

所有包的配置都在 `pack.py` 中的 `PACKS` 字典中定义：

```python
PACKS = {
    "https://github.com/AI-Infra-Team/glog": {
        "branch": "v0.6.0",
        "c++": 17,
        "dependencies": ["gflags"],
        "build_type": "Release",
        "define": [
            ["WITH_GFLAGS", "ON"],
            ["BUILD_SHARED_LIBS", "OFF"],
        ],
    },
    # ... 其他包配置
}
```

### APT 依赖

系统依赖包列表在 `pack.py` 中的 `APT` 数组中定义。

## 🐳 Docker 支持

`pack_in_container.py` 脚本会：

1. 创建基于 Ubuntu 20.04 的 Docker 镜像
2. 安装所有必需的 APT 包
3. 在容器中执行构建
4. 将结果挂载到主机目录

## 🏗️ 架构支持

### 支持的架构

| 架构 | 支持状态 | 说明 |
|------|---------|------|
| **AMD64 (x86_64)** | ✅ 原生支持 | 在 AMD64 主机上原生构建 |
| **ARM64 (aarch64)** | ✅ 完整支持 | 通过 Docker QEMU 模拟或 ARM64 主机 |
| **ARM (armv7)** | ⚠️ 实验性 | 部分支持，需手动测试 |

### 支持的系统

| 系统 | AMD64 | ARM64 | 镜像 |
|------|-------|-------|------|
| Ubuntu 20.04 | ✅ | ✅ | `ubuntu:20.04` |
| Ubuntu 22.04 | ✅ | ✅ | `ubuntu:22.04` |
| ManyLinux 2014 | ✅ | ✅ | `dockcross/manylinux2014-x64` / `dockcross/manylinux2014-aarch64` |

### GitHub Actions 多架构构建

GitHub Actions 工作流会自动为以下组合构建：
- Ubuntu 20.04 (AMD64 + ARM64)
- Ubuntu 22.04 (AMD64 + ARM64)
- ManyLinux 2014 (AMD64 + ARM64)

总共生成 **6 个构建产物**，每个都包含完整的库文件包。

### 技术实现

- **QEMU 模拟**: 使用 `docker/setup-qemu-action` 启用跨架构构建
- **Docker Buildx**: 提供多平台构建支持
- **平台参数**: 自动为 Docker 添加 `--platform linux/arm64` 等参数
- **架构检测**: 自动检测主机架构或通过 `--arch` 参数指定

### 性能说明

- **原生构建** (在对应架构主机上): 最快
- **QEMU 模拟** (在 AMD64 上模拟 ARM64): 较慢（约 2-5 倍时间）
- **建议**: 生产环境推荐使用原生 ARM64 runner 或预构建的包

## 🤝 贡献

1. Fork 这个项目
2. 创建特性分支：`git checkout -b feature/your-feature`
3. 提交更改：`git commit -am 'Add some feature'`
4. 推送分支：`git push origin feature/your-feature`
5. 创建 Pull Request

## 📄 许可证

本项目采用 MIT 许可证 - 查看 LICENSE 文件了解详情。

## 🔗 相关链接

- [etcd-cpp-apiv3](https://github.com/AI-Infra-Team/etcd-cpp-apiv3)
- [gflags](https://github.com/AI-Infra-Team/gflags)
- [glog](https://github.com/AI-Infra-Team/glog)
- [jsoncpp](https://github.com/AI-Infra-Team/jsoncpp)
- [rdma-core](https://github.com/AI-Infra-Team/rdma-core)
- [yalantinglibs](https://github.com/AI-Infra-Team/yalantinglibs) 