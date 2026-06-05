# proj57 在 StarryOS/QEMU 上迁移记录

## 目标

把 `chenlongos/proj57` 的 ACT 模型从 Linux 虚拟机中的 PyTorch/CPU 推理，迁移到 `StarryOS + QEMU + CPU + ONNX Runtime` 环境中运行。

采用的是分阶段路线：

1. 在 Linux/VM 中固定 PyTorch 参考结果
2. 导出确定性 ONNX，并在 Linux 上用 ONNX Runtime 校验
3. 把预处理后的输入带进 StarryOS
4. 先打通 StarryOS 上的文件读取和可执行程序运行
5. 再交叉编译 `riscv64` 版 ONNX Runtime
6. 最后把 ORT C API 接进 `act_infer`

## 环境与关键路径

### Linux VM

- `~/proj57`
  - `proj57_deterministic.onnx`
  - `reference.json`
  - `starry_inputs/`
  - `.venv/`
- `~/StarryOS`
  - `make/disk.img`
  - `act_infer.c`
- `~/onnxruntime`
  - ONNX Runtime 源码目录

### StarryOS

- 架构：`riscv64`
- 目标目录：`/root/proj57`

### 交叉工具链

- C 编译器：`riscv64-linux-musl-gcc`
- C++ 编译器：`riscv64-linux-musl-g++`
- sysroot：`/opt/riscv64-linux-musl-cross/riscv64-linux-musl`
- qemu user：`/usr/bin/qemu-riscv64`

## 下载/安装过的内容与用途

这一节记录为了完成迁移已经下载、安装或准备过的内容，方便后续重建环境或继续开发。

### 1. `proj57` 仓库

- 位置：`~/proj57`
- 来源：`https://github.com/chenlongos/proj57`
- 用途：
  - 提供 `act/` 下的 ACT 模型实现
  - 提供 checkpoint 加载逻辑参考
  - 提供 `infer.ipynb` 和数据预处理流程参考
  - 后续自定义脚本也都放在这个目录下执行

### 2. `proj57` 模型与数据

- 位置：
  - `~/proj57/output/train/model.pt`
  - `~/proj57/output/dataset/`
- 来源：
  - 仓库自带下载流程
  - 模型页参考：`https://huggingface.co/bobodai/proj57_model/tree/main`
- 用途：
  - `model.pt` 是 PyTorch checkpoint
  - `output/dataset/meta/stats.json` 提供 state/action 的归一化和反归一化参数
  - `output/dataset/videos/.../frame_*.jpg` 提供参考输入图像

### 3. Python 虚拟环境 `.venv`

- 位置：`~/proj57/.venv`
- 创建方式：`python3 -m venv .venv`
- 用途：
  - 隔离 PyTorch、ONNX、ONNX Runtime 等 Python 依赖
  - 支持 `freeze_reference.py`
  - 支持 `export_verify_onnx.py`
  - 支持 `prepare_starry_inputs.py`

### 4. Python 侧安装过的关键依赖

已安装过的核心依赖包括：

- `torch`
- `torchvision`
- `onnx`
- `onnxruntime`
- `onnxscript`
- `numpy`
- `pandas`
- `Pillow`
- `pyarrow`
- `jupyter`
- `matplotlib`

用途分别是：

- `torch/torchvision`
  - 跑 PyTorch 模型
  - 做参考推理
  - 导出 ONNX
- `onnx`
  - 校验导出的 ONNX 文件
- `onnxruntime`
  - 在 Linux 上做 ONNX 推理对齐验证
- `onnxscript`
  - 配合当前 PyTorch ONNX 导出器工作
- `numpy/Pillow`
  - 输入预处理和 `.bin` 数据生成
- `pandas/pyarrow/jupyter/matplotlib`
  - 来自仓库原始依赖环境，方便 notebook 流程复现

### 5. 导出的中间产物

这些是迁移过程中已经生成好的关键文件：

- `~/proj57/reference.json`
  - 用途：记录 Linux/PyTorch 参考输出
- `~/proj57/proj57_deterministic.onnx`
  - 用途：StarryOS 侧最终要加载的 ONNX 模型
- `~/proj57/starry_inputs/frame_000000_input.bin`
  - 用途：StarryOS 第一版 image 输入
- `~/proj57/starry_inputs/frame_000227_input.bin`
  - 用途：StarryOS 第一版 image 输入
- `~/proj57/starry_inputs/state.bin`
  - 用途：StarryOS 第一版 state 输入
- `~/proj57/starry_inputs/latent.bin`
  - 用途：StarryOS 第一版 latent 输入

### 6. `StarryOS` 仓库

- 位置：`~/StarryOS`
- 来源：`https://github.com/Starry-OS/StarryOS`
- 用途：
  - 提供 QEMU 启动环境
  - 提供 `make/disk.img`
  - 提供 StarryOS 运行目标系统
  - 后续会承载 `act_infer` 和 ORT 动态库/静态产物

### 7. `make/disk.img`

- 位置：`~/StarryOS/make/disk.img`
- 用途：
  - StarryOS/QEMU 启动时的 rootfs 镜像
  - 已用于拷入：
    - `proj57_deterministic.onnx`
    - `frame_000000_input.bin`
    - `frame_000227_input.bin`
    - `state.bin`
    - `latent.bin`
    - `act_infer`

### 8. `onnxruntime` 源码仓库

- 位置：`~/onnxruntime`
- 来源：`https://github.com/microsoft/onnxruntime.git`
- 用途：
  - 交叉编译 `riscv64` 版 ONNX Runtime
  - 提供后续 `onnxruntime_c_api.h`
  - 目标是产出 `libonnxruntime.so` 或 `libonnxruntime.a`

当前状态：

- 源码已经拉取
- 已修改 `cmake/riscv64.toolchain.cmake`
- 已修改 `onnxruntime/core/common/cpuid_info.cc`
- 还未成功产出最终 runtime 库

### 9. `qemu-user`

- 安装内容：`qemu-user`
- 已确认可执行文件：`/usr/bin/qemu-riscv64`
- 用途：
  - 给 ORT 官方 `--rv64` 构建流程提供交叉编译辅助执行环境
  - 满足 `--riscv_qemu_path=/usr/bin/qemu-riscv64`

### 10. `riscv64-linux-musl` 交叉工具链

- 已确认工具：
  - `riscv64-linux-musl-gcc`
  - `riscv64-linux-musl-g++`
- sysroot：
  - `/opt/riscv64-linux-musl-cross/riscv64-linux-musl`
- 额外确认过：
  - `libatomic.a` 存在

用途：

- 编译 `~/StarryOS/act_infer.c`
- 交叉编译 `riscv64` 版 ONNX Runtime

### 11. `act_infer` 可执行程序

- 源码位置：`~/StarryOS/act_infer.c`
- 编译产物：`~/StarryOS/act_infer`
- StarryOS 侧位置：`/root/proj57/act_infer`
- 用途：
  - 当前阶段：验证 StarryOS 上的可执行程序、参数读取和 `.bin` 文件读取
  - 后续阶段：升级成真正调用 ONNX Runtime C API 的推理程序

## 已完成事项

### 1. 固定了 Linux/PyTorch 参考结果

在 `~/proj57` 中：

- 建立了 Python 虚拟环境 `.venv`
- 安装了 `torch/torchvision/onnx/onnxruntime` 等依赖
- 运行了 `freeze_reference.py`
- 生成了 `reference.json`

固定参考结果如下：

- `frame_000000.jpg`
  - `left_vel=-0.001586`
  - `right_vel=+0.007239`
  - `direction=left`
- `frame_000227.jpg`
  - `left_vel=+0.006118`
  - `right_vel=-0.000455`
  - `direction=right`

这里采用的确定性策略是：

- 不使用随机 latent 采样
- 直接使用 checkpoint 里的 `inference_latent_mu`

### 2. 导出了确定性 ONNX，并完成 Linux/ORT 校验

在 `~/proj57` 中：

- 运行了 `export_verify_onnx.py`
- 导出了 `proj57_deterministic.onnx`
- 在 Linux 上用 ONNX Runtime 校验成功

校验结果：

- 与 `reference.json` 对齐成功
- 数值误差在 `5e-8` 量级
- 左右转方向完全一致

当前约定的输入输出 shape：

- `image`: `[1,1,3,224,224]`
- `state`: `[1,2]`
- `latent`: `[1,32]`
- `action`: `[1,8,3]`

### 3. 生成了 StarryOS 用的预处理输入

在 `~/proj57` 中运行了 `prepare_starry_inputs.py`，生成：

- `starry_inputs/frame_000000_input.bin`
- `starry_inputs/frame_000227_input.bin`
- `starry_inputs/state.bin`
- `starry_inputs/latent.bin`

这些文件的用途是：

- 第一版 StarryOS 程序先不做 JPG 解码和预处理
- 直接读取预处理好的 `float32` 输入
- 先验证 ORT 推理链路

### 4. 已把模型和输入带入 StarryOS

已确认 `make/disk.img` 可挂载并写入。

目前在 StarryOS 的 `/root/proj57` 中已经成功放入：

- `proj57_deterministic.onnx`
- `frame_000000_input.bin`
- `frame_000227_input.bin`
- `state.bin`
- `latent.bin`

### 5. 已打通 StarryOS 上的最小可执行程序

在 `~/StarryOS` 中写了 `act_infer.c` 并交叉编译成功，后续放入了 `/root/proj57/act_infer`。

已经验证过：

- `act_infer` 可以在 StarryOS 上执行
- 可以正确接收命令行参数
- 可以打开 `.bin` 文件
- 可以读取 `float32` 数据

已确认的数据规模：

- `image_count = 150528`
- `state_count = 2`
- `latent_count = 32`

这说明：

- rootfs 拷贝流程没问题
- 程序架构没问题
- StarryOS 文件读写没问题
- `.bin` 输入格式没问题

## 目前修改过或使用过的脚本

### `~/proj57/freeze_reference.py`

作用：

- 固定 PyTorch 参考输出
- 生成 `reference.json`

### `~/proj57/export_verify_onnx.py`

作用：

- 导出 `proj57_deterministic.onnx`
- 用 Linux 上的 ONNX Runtime 做数值校验

### `~/proj57/prepare_starry_inputs.py`

作用：

- 把预处理后的 image/state/latent 写成 `.bin`
- 给 StarryOS 第一版程序直接使用

### `~/StarryOS/act_infer.c`

当前阶段作用：

- 先验证 StarryOS 上程序执行和输入读取
- 后续应升级为真正调用 ORT C API 的推理程序

## 已踩过的主要坑

### 1. 直接把 Python 粘进 shell

一开始把 Python 代码直接粘进了 bash，导致大量 `import not found` 和语法错误。

结论：

- Python 脚本必须保存成 `.py` 文件再运行

### 2. StarryOS/QEMU 控制台不适合做复杂编辑

在 `StarryOS` 里直接折腾源码和长命令非常容易出错。

结论：

- 开发和编译放在 Ubuntu/Linux VM
- 运行和验证放在 StarryOS

### 3. 官方 ORT `riscv64.toolchain.cmake` 默认不适配当前 musl 工具链

官方工具链文件默认使用：

- `riscv64-unknown-linux-gnu-gcc`
- `${RISCV_TOOLCHAIN_ROOT}/sysroot`

但当前可用的是：

- `riscv64-linux-musl-gcc`
- `riscv64-linux-musl-g++`
- `/opt/riscv64-linux-musl-cross/riscv64-linux-musl`

所以已经把 `~/onnxruntime/cmake/riscv64.toolchain.cmake` 改成了 musl 版本。

### 4. ORT 源码中的 `asm/hwprobe.h` 缺失

在 `riscv64` 路径上，`onnxruntime/core/common/cpuid_info.cc` 会直接：

- `#include <asm/hwprobe.h>`

但当前 musl sysroot 里没有这个头文件。

因此已对 `cpuid_info.cc` 做了兼容性修改：

- 如果有 `asm/hwprobe.h`，继续使用
- 如果没有，则将 RISC-V 的 `has_fp16_` 降级为 `false`

### 5. 链接阶段缺少 `libatomic`

曾出现：

- `undefined reference to '__atomic_compare_exchange_1'`

已确认工具链内存在：

- `libatomic.a`

因此后续构建时需要显式带上：

- `-latomic`

### 6. flatbuffers test 代码在 musl 下报 `strtoll_l` 问题

构建后期出现：

- `flatbuffers/util.h`
- `strtoll_l was not declared in this scope`

而报错目标是：

- `onnxruntime_test_all`

这说明问题出在测试目标，而不一定是 runtime 主库本体。

因此下一步必须彻底关闭 unit tests，而不是只用 `--skip_tests`。

## 当前阻塞点

### ONNX Runtime 还没有成功产出库

当前还没有找到：

- `libonnxruntime.so`
- `libonnxruntime.a`

说明 `riscv64` 版 ORT 还未编译完成。

最近一次构建已经推进到：

- 编译/链接主库相关目标
- 但后面又进入了 `onnxruntime_test_all`

说明仅使用 `--skip_tests` 还不够，需要进一步显式关闭测试目标。

## 当前应继续执行的构建命令

建议从这里继续：

```bash
cd ~/onnxruntime
rm -rf build/Linux
./build.sh \
  --config Release \
  --rv64 \
  --riscv_toolchain_root=/opt/riscv64-linux-musl-cross \
  --riscv_qemu_path=/usr/bin/qemu-riscv64 \
  --skip_tests \
  --build_shared_lib \
  --no_kleidiai \
  --no_sve \
  --cmake_extra_defines \
    onnxruntime_BUILD_UNIT_TESTS=OFF \
    onnxruntime_BUILD_BENCHMARKS=OFF \
    CMAKE_C_FLAGS="-pthread -latomic" \
    CMAKE_CXX_FLAGS="-pthread -latomic" \
    CMAKE_EXE_LINKER_FLAGS="-pthread -latomic" \
    CMAKE_SHARED_LINKER_FLAGS="-pthread -latomic" \
  2>&1 | tee ~/ort_rv64_build.log
```

构建结束后先查：

```bash
find ~/onnxruntime -name "libonnxruntime.so" 2>/dev/null
find ~/onnxruntime -name "libonnxruntime.a" 2>/dev/null
```

如果还是失败，再看：

```bash
grep -n "onnxruntime_test_all" ~/ort_rv64_build.log | tail
tail -n 120 ~/ort_rv64_build.log
```

## ORT 编译成功后的下一步

一旦拿到：

- `libonnxruntime.so`
  或
- `libonnxruntime.a`

下一步就是升级 `~/StarryOS/act_infer.c`：

1. 引入 ONNX Runtime C API
2. 加载 `/root/proj57/proj57_deterministic.onnx`
3. 创建 `image/state/latent` 三个输入 tensor
4. 执行一次推理
5. 读取 `action[0][0]`
6. 做反归一化
7. 打印：
   - `left_vel`
   - `right_vel`
   - `gripper`
   - `direction`

## 最终验证标准

StarryOS 上最终只要能看到：

- `frame_000000_input.bin` -> `left`
- `frame_000227_input.bin` -> `right`

就可以认为：

- 模型已在 `StarryOS + QEMU + CPU` 上基本跑通

若数值也接近 Linux 参考，则更好。当前参考值为：

- `frame_000000`
  - `left_vel=-0.001586`
  - `right_vel=+0.007239`
- `frame_000227`
  - `left_vel=+0.006118`
  - `right_vel=-0.000455`

## 当前阶段总结

现在真正剩下的核心问题只有一个：

- **把 `riscv64` 版 ONNX Runtime 编译成功**

其他外围链路基本都已经打通：

- PyTorch 参考结果固定完成
- ONNX 导出与 Linux 校验完成
- StarryOS 输入准备完成
- StarryOS 上文件传输完成
- StarryOS 上最小程序执行与输入读取完成

因此后续开发重点应全部集中在：

- ORT 编译成功
- ORT 接入 `act_infer`
- StarryOS 上输出最终动作方向

## 2026-06-04 补充：最终已跑通

截至 `2026-06-04`，该链路已经在真实 `StarryOS + QEMU` 环境中跑通，不再停留在 Linux/VM 内的用户态验证阶段。

另见：

- `下载与获取记录`：[proj57_starryos_qemu_downloads.md](/f:/Users/LiHongYu/OS-qemu/proj57_starryos_qemu_downloads.md)

最终在 `StarryOS` 的 `/root/proj57` 中直接运行：

```bash
./act_infer proj57_deterministic.onnx frame_000000_input.bin state.bin latent.bin
./act_infer proj57_deterministic.onnx frame_000227_input.bin state.bin latent.bin
```

实际输出结果为：

- `frame_000000_input.bin`
  - `left_vel=-0.001586`
  - `right_vel=+0.007239`
  - `direction=left`
- `frame_000227_input.bin`
  - `left_vel=+0.006118`
  - `right_vel=-0.000454`
  - `direction=right`

这与 Linux 参考结果对齐，说明：

- `riscv64` 版 `ONNX Runtime` 已编译成功
- `act_infer.c` 已成功接入 ORT C API
- `StarryOS` 上模型加载、外部权重读取、输入 tensor 构造、推理执行、后处理和方向判断均已打通

## 本轮对话实际完成的工作

本轮对话中，最终完成了以下闭环：

- 把 `riscv64` 版 `ONNX Runtime` 真正编译出来，产出了 `libonnxruntime.so`
- 在 `act_infer.c` 中接入 ORT C API，完成模型加载和 `image/state/latent` 三输入推理
- 在 Linux VM 中通过 `qemu-riscv64` 先验证了 `act_infer` 的用户态运行链路
- 确认 `C + ORT` 的原始输出与 `Python + ORT` 的原始输出完全一致
- 根据 Python 侧后处理逻辑，把 `left/right` 两个通道的反归一化迁移到 `act_infer.c`
- 识别并补齐了 `StarryOS` 运行所需的动态库和外部权重文件
- 最终在真正启动后的 `StarryOS` 中运行成功，而不只是目录名叫 `StarryOS` 的 Linux 路径中成功

## 本轮用到的关键知识点

### 1. 区分 `qemu-riscv64` 用户态验证和真正的 `StarryOS` 全系统验证

- `~/StarryOS` 只是 Linux VM 中的源码目录
- `qemu-riscv64 ./act_infer ...` 只是 Linux VM 内的 `riscv64` 用户态验证
- 真正目标是启动后的 `StarryOS` 系统中，在 `/root/proj57` 下直接运行 `./act_infer`

这两层验证都很重要：

- 用户态验证适合快速排除 `act_infer`、ORT、动态库、模型输入输出问题
- 全系统验证才代表 `StarryOS + QEMU` 目标环境真正跑通

### 2. ORT 模型存在外部权重文件

`proj57_deterministic.onnx` 不是单文件模型，还依赖：

- `proj57_deterministic.onnx.data`

如果只拷贝 `.onnx` 而漏掉 `.onnx.data`，会在 `CreateSession` 阶段报：

- `External data path does not exist`

因此模型部署时必须把：

- `proj57_deterministic.onnx`
- `proj57_deterministic.onnx.data`

一起放到同一目录。

### 3. `StarryOS` 上动态库搜索路径和运行时依赖

仅把库放到 `/root/proj57` 并设置 `LD_LIBRARY_PATH` 在当前环境下并不总是可靠，最终采用的是把关键共享库直接放到 `/lib` 或确保它们能被默认 loader 找到。

本次实际需要的关键运行时依赖包括：

- `libonnxruntime.so.1`
- `libstdc++.so.6`
- `libgcc_s.so.1`
- `libatomic.so.1`
- `/lib/ld-musl-riscv64.so.1`

### 4. `musl` 交叉编译和动态链接细节

本项目使用的是：

- `riscv64-linux-musl-gcc`
- `riscv64-linux-musl-g++`

因此需要特别注意：

- loader 是 `ld-musl-riscv64.so.1`
- 链接时必须显式带 `-latomic`
- `qemu-riscv64` 的用户态验证需要准备完整的 `musl` 运行时根目录

### 5. ORT 输出与 Python 参考的一致性验证

在用户态验证阶段，先确认：

- `C + ORT` 的 `raw action`
- `Python + ORT` 的 `raw action`

完全一致，再去处理后处理问题。

这一步证明：

- 模型文件没问题
- 输入 tensor 名称和 shape 没问题
- ORT C API 调用链没问题

而方向不一致时，应优先怀疑后处理，而不是怀疑 ORT 本身。

### 6. 动作反归一化来自 Python 侧逻辑

Python 侧确认的后处理公式是：

```python
action_denorm = ((action + 1.0) / 2.0) * (q99 - q01) + q01
```

本次为先完成方向验证，在 `act_infer.c` 中至少对前两个通道完成了反归一化：

- `left_vel`
- `right_vel`

当前 `gripper` 仍输出 `raw` 值，这不影响“左右方向跑通”的验收目标。

## 最终部署到 StarryOS 的最小文件集合

部署到 `StarryOS` 的 `/root/proj57` 时，至少需要：

- `act_infer`
- `proj57_deterministic.onnx`
- `proj57_deterministic.onnx.data`
- `frame_000000_input.bin`
- `frame_000227_input.bin`
- `state.bin`
- `latent.bin`
- `libonnxruntime.so`
- `libonnxruntime.so.1`
- `libonnxruntime.so.1.27.0`
- `libstdc++.so`
- `libstdc++.so.6`
- `libstdc++.so.6.0.29`
- `libgcc_s.so`
- `libgcc_s.so.1`
- `libatomic.so`
- `libatomic.so.1`

并且系统中需要有：

- `/lib/ld-musl-riscv64.so.1`

## 关于“怎么看 QEMU 上用了多少内存”

这个问题要先分清楚你想看的是哪一层的内存：

### 1. 看 `qemu-riscv64` 用户态验证时的宿主机内存

如果是在 Linux VM 中用 `qemu-riscv64` 跑 `act_infer`，最直接的办法是：

```bash
/usr/bin/time -v /usr/bin/qemu-riscv64 \
  -L "$QEMU_LD_PREFIX" \
  -E LD_LIBRARY_PATH=/lib \
  ./act_infer proj57_deterministic.onnx frame_000000_input.bin state.bin latent.bin
```

重点看输出中的：

- `Maximum resident set size`

这表示这次宿主机侧进程的峰值常驻内存。

### 2. 看真正全系统 QEMU 的宿主机内存

如果你启动的是完整 `StarryOS` 虚拟机，想看整个 `qemu-system-*` 进程吃了多少内存，可以在宿主 Linux VM 中执行：

```bash
ps -eo pid,rss,vsz,cmd | grep qemu-system | grep -v grep
```

其中：

- `rss` 是常驻内存，单位通常是 `KB`
- `vsz` 是虚拟地址空间大小

如果想看更详细的总映射，可以再用：

```bash
pmap -x <qemu-system-pid> | tail -n 1
```

### 3. 看 StarryOS 内部可用内存变化

如果关注的是“模型在 guest 内部大概占了多少内存”，可以在 `StarryOS` 中运行前后各看一次总内存或空闲内存。

如果 `StarryOS` 支持 `procfs`，可尝试：

```bash
cat /proc/meminfo
```

在运行 `act_infer` 前后分别记录：

- `MemTotal`
- `MemFree`
- `MemAvailable`

如果支持 `free`，也可以：

```bash
free
```

然后用运行前后的差值，粗略估算 guest 侧因为模型加载和推理额外消耗的内存。

### 4. 当前最推荐的观察方式

如果只是想快速知道“这个模型跑起来大概占了多少内存”，当前最推荐先看两类数字：

- 宿主侧：`/usr/bin/time -v` 的 `Maximum resident set size`
- guest 侧：`StarryOS` 运行前后 `MemFree` 或 `MemAvailable` 的变化

前者适合快速量化 `qemu` 进程实际吃掉的宿主机内存，后者更接近“StarryOS 内看到的模型占用”。
