# proj57 ACT 模型在 StarryOS/QEMU 上的推理部署说明

## 一、项目概述

本项目完成了 `chenlongos/proj57` 赛题中 ACT 模型在 `StarryOS + QEMU` 环境下的 CPU 推理部署。原始模型为 PyTorch checkpoint，本项目将其导出为确定性 ONNX 模型，并在 StarryOS 中通过 ONNX Runtime C API 完成模型加载、输入构造、推理执行、动作后处理和左右转方向判断。

本项目对应赛题任务三：

| 项目 | 内容 |
|------|------|
| 目标平台 | QEMU 中运行的 StarryOS |
| 目标架构 | riscv64 |
| 推理设备 | CPU |
| 推理后端 | ONNX Runtime |
| 推理程序语言 | C |
| 交叉工具链 | riscv64-linux-musl |
| 模型格式 | ONNX |

最终验证结果为：

| 输入 | left_vel | right_vel | direction |
|------|----------|-----------|-----------|
| `frame_000000_input.bin` | `-0.001586` | `+0.007239` | `left` |
| `frame_000227_input.bin` | `+0.006118` | `-0.000454` | `right` |

上述输出与 Linux/PyTorch 参考结果方向一致，数值也基本对齐。

## 二、代码与文件说明

### 1. StarryOS 侧推理程序

核心推理代码：

```text
src/act_infer.c
```

对应原始开发路径：

```text
~/StarryOS/act_infer.c
```

该程序使用 C 语言编写，通过 ONNX Runtime C API 完成以下工作：

- 读取 `proj57_deterministic.onnx`
- 读取预处理后的 `image/state/latent` 三个 float32 输入文件
- 创建 ONNX Runtime 输入 tensor
- 执行 ACT 模型推理
- 获取输出 `action`
- 对 `left_vel` 和 `right_vel` 做反归一化
- 输出左右轮速度和方向判断结果

实际输入输出 shape：

| 名称 | Shape | 说明 |
|------|-------|------|
| `image` | `[1,1,3,224,224]` | 预处理后的图像输入，多出的 `1` 为相机维度 |
| `state` | `[1,2]` | 机器人状态 |
| `latent` | `[1,32]` | 固定 CVAE latent |
| `action` | `[1,8,3]` | 输出动作 chunk |

后处理公式：

```c
action_denorm = ((action + 1.0f) * 0.5f) * (q99 - q01) + q01;
```

方向判断逻辑：

```c
direction = left_vel < right_vel ? "left" : "right";
```

### 2. Python 辅助脚本

这些脚本在 Linux VM 中运行，用于模型导出和输入准备，不直接在 StarryOS 中运行。

| 文件 | 作用 |
|------|------|
| `scripts/freeze_reference.py` | 固定 PyTorch 参考输出，生成 `reference.json` |
| `scripts/export_verify_onnx.py` | 导出确定性 ONNX，并用 Linux ONNX Runtime 校验 |
| `scripts/prepare_starry_inputs.py` | 生成 StarryOS 使用的 float32 `.bin` 输入 |

### 3. 模型与输入文件

最终部署到 StarryOS 的 `/root/proj57` 目录时，至少需要：

```text
act_infer
proj57_deterministic.onnx
proj57_deterministic.onnx.data
frame_000000_input.bin
frame_000227_input.bin
state.bin
latent.bin
```

其中：

- `proj57_deterministic.onnx` 是导出的 ONNX 模型文件
- `proj57_deterministic.onnx.data` 是 ONNX 外部权重文件，必须与 `.onnx` 放在同一目录
- `frame_000000_input.bin` 和 `frame_000227_input.bin` 是离线预处理后的图像输入
- `state.bin` 是归一化后的状态输入
- `latent.bin` 是固定 latent 输入

### 4. 运行时动态库

`act_infer` 是 riscv64 动态链接程序，直接依赖：

```text
libonnxruntime.so.1
libatomic.so.1
libc.so
```

StarryOS 中实际部署时还需要确保以下运行时库可被 loader 找到：

```text
libonnxruntime.so
libonnxruntime.so.1
libonnxruntime.so.1.27.0
libstdc++.so
libstdc++.so.6
libstdc++.so.6.0.29
libgcc_s.so
libgcc_s.so.1
libatomic.so
libatomic.so.1
ld-musl-riscv64.so.1
```

运行时库主要来自：

```text
/opt/riscv64-linux-musl-cross/riscv64-linux-musl/lib/
~/onnxruntime/build/Linux/Release/
```

## 三、部署环境

### 1. Linux VM

Linux VM 负责开发、导出、交叉编译和镜像写入。

关键目录：

```text
~/proj57
~/StarryOS
~/onnxruntime
```

Python 侧依赖：

```text
torch
torchvision
onnx
onnxruntime
onnxscript
numpy
pandas
Pillow
pyarrow
jupyter
matplotlib
```

### 2. StarryOS/QEMU

StarryOS 运行环境：

```text
架构：riscv64
目标目录：/root/proj57
推理后端：ONNX Runtime CPU
```

### 3. 交叉编译工具链

使用的工具链：

```text
riscv64-linux-musl-gcc
riscv64-linux-musl-g++
```

sysroot：

```text
/opt/riscv64-linux-musl-cross/riscv64-linux-musl
```

qemu user：

```text
/usr/bin/qemu-riscv64
```

## 四、模型转换与验证流程

### 1. 固定 PyTorch 参考结果

在 Linux VM 中运行：

```bash
cd ~/proj57
python freeze_reference.py
```

该步骤生成 `reference.json`，用于记录 PyTorch/CPU 的参考输出。

参考结果：

```text
frame_000000.jpg:
  left_vel=-0.001586
  right_vel=+0.007239
  direction=left

frame_000227.jpg:
  left_vel=+0.006118
  right_vel=-0.000455
  direction=right
```

这里采用确定性策略：

- 不使用随机 latent 采样
- 直接使用 checkpoint 中的 `inference_latent_mu`

### 2. 导出确定性 ONNX

在 Linux VM 中运行：

```bash
cd ~/proj57
python export_verify_onnx.py
```

该步骤完成：

- 加载 PyTorch checkpoint
- 导出 `proj57_deterministic.onnx`
- 使用 Linux ONNX Runtime 做数值校验

Linux ONNX Runtime 校验结果：

```text
与 reference.json 对齐成功
数值误差约为 5e-8
左右转方向完全一致
```

### 3. 准备 StarryOS 输入

在 Linux VM 中运行：

```bash
cd ~/proj57
python prepare_starry_inputs.py
```

该步骤生成：

```text
starry_inputs/frame_000000_input.bin
starry_inputs/frame_000227_input.bin
starry_inputs/state.bin
starry_inputs/latent.bin
```

当前实现采用离线预处理方案：

- Linux VM 中完成图像 resize、ToTensor、Normalize
- Linux VM 中完成 state 归一化和 latent 固定
- StarryOS 中直接读取 float32 `.bin` 输入

这样可以先把模型推理链路跑通，避免在 StarryOS 中额外引入 JPEG 解码和图像预处理依赖。

## 五、ONNX Runtime 交叉编译

ONNX Runtime 源码路径：

```text
~/onnxruntime
```

构建目标：

```text
riscv64 Linux musl
shared library
CPU backend
```

构建过程中做过的关键适配：

- 将官方 `riscv64.toolchain.cmake` 调整为当前 `riscv64-linux-musl` 工具链
- 处理 musl sysroot 中缺少 `asm/hwprobe.h` 的问题
- 链接时显式加入 `-latomic`
- 关闭测试目标，避免 musl 下 test 代码中的 `strtoll_l` 问题
- 将部分 FetchContent 远程依赖改成本地缓存 zip，避免 GitHub/Codeload 下载失败

最终产物：

```text
~/onnxruntime/build/Linux/Release/libonnxruntime.so
~/onnxruntime/build/Linux/Release/libonnxruntime.so.1
~/onnxruntime/build/Linux/Release/libonnxruntime.so.1.27.0
```

已确认 `libonnxruntime.so` 可被 `act_infer` 在 StarryOS 中调用。

## 六、StarryOS 侧运行方式

将以下文件部署到 StarryOS 的 `/root/proj57`：

```text
act_infer
proj57_deterministic.onnx
proj57_deterministic.onnx.data
frame_000000_input.bin
frame_000227_input.bin
state.bin
latent.bin
libonnxruntime.so*
必要 musl 运行时库
```

在 StarryOS 中运行：

```bash
cd /root/proj57
./act_infer proj57_deterministic.onnx frame_000000_input.bin state.bin latent.bin
./act_infer proj57_deterministic.onnx frame_000227_input.bin state.bin latent.bin
```

实际输出：

```text
frame_000000_input.bin:
  left_vel=-0.001586
  right_vel=+0.007239
  direction=left

frame_000227_input.bin:
  left_vel=+0.006118
  right_vel=-0.000454
  direction=right
```

这说明：

- StarryOS 可以执行 `riscv64` 版 `act_infer`
- ONNX Runtime 可以成功加载模型
- ONNX 外部权重文件可以被正确读取
- `image/state/latent` 三输入 tensor 构造正确
- ACT 模型推理输出与 Linux 参考方向一致
- 动作后处理和方向判断已经完成

## 七、文件大小与资源情况

已确认文件大小：

| 文件 | 大小 |
|------|------|
| `act_infer` | `13K` |
| `act_infer.c` | `5.2K` |
| `libonnxruntime.so` | `23M` |
| `rootfs-riscv64.img` | `1.0G` |

推理耗时和内存占用需要在最终提交前实测补充。可使用以下命令采集 Linux VM 中 `qemu-riscv64` 用户态验证数据：

```bash
cd ~/StarryOS
/usr/bin/time -v /usr/bin/qemu-riscv64 \
  -L /opt/riscv64-linux-musl-cross/riscv64-linux-musl \
  -E "LD_LIBRARY_PATH=$PWD:/opt/riscv64-linux-musl-cross/riscv64-linux-musl/lib" \
  ./act_infer proj57_deterministic.onnx frame_000000_input.bin state.bin latent.bin
```

重点记录：

```text
Elapsed wall clock time
Maximum resident set size
```

如果统计完整 StarryOS/QEMU 进程的宿主机内存，可以在 Linux VM 中运行：

```bash
ps -eo pid,rss,vsz,cmd | grep qemu-system | grep -v grep
```

如果 StarryOS 内支持 `/proc/meminfo`，也可以在运行前后记录：

```bash
cat /proc/meminfo
```

## 八、提交文件清单

建议 GitHub 仓库中提交源码和文档：

```text
src/act_infer.c
scripts/freeze_reference.py
scripts/export_verify_onnx.py
scripts/prepare_starry_inputs.py
docs/proj57_starryos_qemu_progress.md
docs/proj57_starryos_qemu_downloads.md
README.md 或本说明文档
```

建议通过 GitHub Release 提交大文件和运行产物：

```text
act_infer
proj57_deterministic.onnx
proj57_deterministic.onnx.data
frame_000000_input.bin
frame_000227_input.bin
state.bin
latent.bin
libonnxruntime.so
libonnxruntime.so.1
libonnxruntime.so.1.27.0
必要运行时库
```

不建议直接提交到普通 Git 仓库的大文件：

```text
rootfs-riscv64.img
*.onnx.data
*.so*
*.bin
```

## 九、AI 使用说明

本项目开发过程中使用了 AI 辅助，包括：

- 帮助梳理从 PyTorch 到 ONNX 再到 StarryOS/QEMU 的迁移路线
- 辅助编写和修改 `freeze_reference.py`、`export_verify_onnx.py`、`prepare_starry_inputs.py`
- 辅助分析 ONNX Runtime 交叉编译错误和运行时动态库问题
- 辅助编写 `act_infer.c` 中的 ONNX Runtime C API 调用流程
- 辅助整理部署记录、依赖记录和参赛文档

人工完成和确认的部分包括：

- 在 Linux VM 中实际下载、安装、编译和运行相关工具
- 在 StarryOS/QEMU 中实际部署和执行 `act_infer`
- 根据终端输出确认推理结果
- 识别最终需要部署的模型、输入、动态库和外部权重文件
- 对最终提交内容进行检查和取舍

本项目没有直接复制他人的完整参赛实现。原始模型结构、数据格式、checkpoint 和 Notebook 参考流程来自赛题仓库 `chenlongos/proj57`，StarryOS 运行环境来自 StarryOS 项目，ONNX Runtime 来自 Microsoft ONNX Runtime 开源项目。

## 十、当前实现的局限与后续优化

当前实现已经完成任务三的基础目标，即在 `StarryOS + QEMU + CPU` 中完成 ACT 推理并输出正确方向。后续可以继续优化：

- 将图像预处理从 Linux 离线阶段迁移到 StarryOS 侧
- 补充完整的 `gripper_target` 反归一化输出
- 对 ONNX Runtime 或模型进行裁剪，降低运行库体积
- 统计并优化推理耗时和内存占用
- 增加更多输入样本，验证长时间运行稳定性

