# proj57 在 StarryOS/QEMU 上迁移：下载与获取记录

## 目的

这份文档只记录本轮迁移过程中“额外获取过什么东西”，包括：

- 直接下载的压缩包
- `git clone` 得到的源码
- 通过包管理器安装的工具
- 为了 ORT 构建而本地缓存的依赖

它不重复记录原本就已经在项目里的源码或模型逻辑，只聚焦“后来为了跑通而拿到手的东西”。

## 一、通过系统包管理器安装

### 1. `protobuf-compiler`

用途：

- 提供宿主机上的 `protoc`
- 用于避免 ORT 构建阶段继续下载 `protoc_binary`

安装方式：

```bash
sudo apt-get update
sudo apt-get install -y protobuf-compiler
```

安装后验证：

```bash
which protoc
protoc --version
```

实际结果：

- 路径：`/usr/bin/protoc`
- 版本：`libprotoc 3.21.12`

最终是否参与成功链路：

- `是`

## 二、手动下载的压缩包

### 1. `abseil-cpp-20250814.0.zip`

来源：

- `https://codeload.github.com/abseil/abseil-cpp/zip/refs/tags/20250814.0`

本地保存路径：

- `~/ort-deps-cache/abseil-cpp-20250814.0.zip`

用途：

- 作为 ORT 构建依赖 `abseil_cpp` 的本地缓存
- 避免 `FetchContent` 在 GitHub/Codeload 上反复失败

校验值：

- `a9eb1d648cbca4d4d788737e971a6a7a63726b07`

最终是否参与成功链路：

- `是`

### 2. `re2-2024-07-02.zip`

来源：

- `https://codeload.github.com/google/re2/zip/refs/tags/2024-07-02`

本地保存路径：

- `~/ort-deps-cache/re2-2024-07-02.zip`

用途：

- 作为 ORT 构建依赖 `re2` 的本地缓存

校验值：

- `646e1728269cde7fcef990bf4a8e87b047882e88`

最终是否参与成功链路：

- `是`

### 3. `protoc-27.2-linux-x86_64.zip`

来源：

- `https://github.com/protocolbuffers/protobuf/releases/download/v27.2/protoc-27.2-linux-x86_64.zip`

本地保存路径：

- `~/protoc-27.2-linux-x86_64.zip`

解压目标：

- `~/.local/bin/protoc`
- `~/.local/include/google/protobuf/*`

用途：

- 作为尝试性的宿主机 `protoc` 方案

实际结果：

- 成功下载并解压
- `protoc --version` 为 `libprotoc 27.2`

最终是否参与成功链路：

- `否`

原因：

- 最终成功链路采用的是系统安装的 `/usr/bin/protoc`，版本 `3.21.12`
- 这样更接近 ORT 当前依赖的 `protobuf v21.12`

## 三、通过 `git clone` 获取并本地归档的源码

### 1. `protobuf-v21.12-src`

获取方式：

```bash
git clone --depth 1 --branch v21.12 https://github.com/protocolbuffers/protobuf.git protobuf-v21.12-src
```

用途：

- 因为直接下载 `protobuf v21.12` zip 失败
- 通过 `git clone` 拉源码，再用 `git archive` 本地打包供 ORT 使用

本地打包结果：

- `~/ort-deps-cache/protobuf-v21.12.zip`

生成方式：

```bash
git -C ~/protobuf-v21.12-src archive --format=zip \
  --output="$HOME/ort-deps-cache/protobuf-v21.12.zip" \
  --prefix=protobuf-21.12/ \
  HEAD
```

生成后校验值：

- `517b6ad6689794e45c8cda6073df0e948c35d359`

最终是否参与成功链路：

- `是`

### 2. `mp11-boost-1.82.0-src`

获取方式：

```bash
git clone --depth 1 --branch boost-1.82.0 https://github.com/boostorg/mp11.git mp11-boost-1.82.0-src
```

用途：

- 为 ORT 依赖 `mp11` 提供本地 zip 缓存

后续处理：

- 用 `git archive` 打成 `~/ort-deps-cache/mp11-boost-1.82.0.zip`

最终是否参与成功链路：

- `是`

### 3. `date-v3.0.1-src`

获取方式：

```bash
git clone --depth 1 --branch v3.0.1 https://github.com/HowardHinnant/date.git date-v3.0.1-src
```

用途：

- 为 ORT 依赖 `date` 提供本地 zip 缓存

后续处理：

- 用 `git archive` 打成 `~/ort-deps-cache/date-v3.0.1.zip`

最终是否参与成功链路：

- `是`

## 四、为了 ORT 构建本地缓存过的依赖

这些文件的共同用途都是：

- 避免 ORT 的 `FetchContent` 继续访问 GitHub/Codeload
- 改成从本地绝对路径读取依赖

### 缓存目录

- `~/ort-deps-cache`

### 实际用到过的缓存文件

- `abseil-cpp-20250814.0.zip`
- `re2-2024-07-02.zip`
- `protobuf-v21.12.zip`
- `mp11-boost-1.82.0.zip`
- `date-v3.0.1.zip`

### 配套动作

通过修改：

- `~/onnxruntime/cmake/deps.txt`

把依赖条目从远程 URL 改成本地路径，例如：

- `abseil_cpp;...` -> `~/ort-deps-cache/abseil-cpp-20250814.0.zip`
- `re2;...` -> `~/ort-deps-cache/re2-2024-07-02.zip`
- `protobuf;...` -> `~/ort-deps-cache/protobuf-v21.12.zip`
- `mp11;...` -> `~/ort-deps-cache/mp11-boost-1.82.0.zip`
- `date;...` -> `~/ort-deps-cache/date-v3.0.1.zip`

最终是否参与成功链路：

- `是`

## 五、运行时从工具链侧带入的共享库

这些不一定是“现下载”的，但在本轮迁移里是后来额外整理、拷贝并部署到 `StarryOS` 的运行时依赖，因此这里也一并记录。

来源目录：

- `/opt/riscv64-linux-musl-cross/riscv64-linux-musl/lib/`

实际带入并部署的主要文件：

- `libstdc++.so`
- `libstdc++.so.6`
- `libstdc++.so.6.0.29`
- `libgcc_s.so`
- `libgcc_s.so.1`
- `libatomic.so`
- `libatomic.so.1`
- `libatomic.so.1.2.0`
- `ld-musl-riscv64.so.1`
- `libc.so`

用途：

- 支撑 `act_infer` 与 `libonnxruntime.so` 在 `StarryOS` 中动态加载并运行

最终是否参与成功链路：

- `是`

## 六、模型部署时额外必须带上的文件

### 1. `proj57_deterministic.onnx.data`

这个文件不是“后来下载”的，但它是在最终部署到 `StarryOS` 阶段额外识别出来、并确认必须和模型一起带上的关键文件，因此单独记一下。

用途：

- `proj57_deterministic.onnx` 的外部权重数据文件

现象：

- 如果漏掉，会在 `CreateSession` 阶段报：
  - `External data path does not exist`

最终是否参与成功链路：

- `是`

## 七、最终哪些东西真正参与了成功部署

最终真正参与“编译成功 + StarryOS 跑通”的获取项有：

- `protobuf-compiler` 提供的 `/usr/bin/protoc`
- `abseil-cpp-20250814.0.zip`
- `re2-2024-07-02.zip`
- `protobuf-v21.12-src` 及其归档 `protobuf-v21.12.zip`
- `mp11-boost-1.82.0-src` 及其归档 zip
- `date-v3.0.1-src` 及其归档 zip
- `riscv64-musl` 工具链中的 `libstdc++/libgcc_s/libatomic/ld-musl`
- `proj57_deterministic.onnx.data`

最终没有进入成功链路、但中途尝试过的获取项主要是：

- `protoc-27.2-linux-x86_64.zip`

## 八、经验总结

从这次过程看，后续类似迁移任务里，凡是遇到：

- `FetchContent` 访问 GitHub 不稳定
- `codeload.github.com` 握手失败
- release asset 下载失败

都可以优先采用下面两种思路：

1. 宿主机已有工具的，优先用宿主机工具替代远程下载
   - 例如：`--path_to_protoc_exe /usr/bin/protoc`
2. 远程 zip 不稳定的，优先改为：
   - `git clone --depth 1 --branch <tag>`
   - `git archive --format=zip`
   - 再把 `deps.txt` 改成本地绝对路径

这样通常比反复重试网络下载更稳。
