# example — 通过 Conan 包消费 `food_volume_measure`

把 `food_volume_measure` 当作**外部依赖**消费的最小工程，用来验证安装出来的包
（头文件 + 静态库）在**桌面 x86_64** 与 **RK3562（aarch64）** 两套目标上都能正常
构建、链接和运行。

- `example` — 功能演示：跑一次测量并打印完整诊断（含逐连通块明细）
- `bench` — 性能基准：`run()` 端到端耗时 + CPU 时间 + 进程峰值内存

两者都是纯消费者：只 include 安装出来的公开头
（`types.hpp` / `measurement.hpp` / `log.hpp`），
不依赖任何 `src/` 下的私有实现。

## 前置：把库装进本地 Conan 缓存

```bash
# 在 food_volume_measure 仓库根目录
conan create . -pr:b=default -pr:h=default -s build_type=Release --build=missing
```

`example/conanfile.py` 依赖 `food_volume_measure/0.1.0`，**profile 必须与编库时一致**
（桌面 gcc-13 / RK3562 gcc-14 + armv8，均 Release）。

## 构建与运行（桌面 x86_64）

```bash
cd example
conan install . -pr:b=default -pr:h=../profiles/gcc13_release \
  -of build/desktop --build=missing

cmake -S . -B build/desktop \
  -DCMAKE_TOOLCHAIN_FILE=build/desktop/conan_toolchain.cmake \
  -DCMAKE_BUILD_TYPE=Release
cmake --build build/desktop -j

./build/desktop/example ../py/test_data/base.pcd ../py/test_data/food/3.pcd
./build/desktop/bench   ../py/test_data/base.pcd ../py/test_data/food/3.pcd 5
```

> 测试点云在 `../py/test_data/`：`base.pcd` 为空炉基线，`food/1.pcd` … `food/11.pcd`
> 为食材帧。真值见 `../py/test_data/ground_truth.json`。

## 交叉编译与板端部署（RK3562 / aarch64）

完整流程（工具链准备、profile 说明、依赖链规模、验证点、常见问题）见
**[`cross-compile-rk3562.md`](cross-compile-rk3562.md)**。核心命令：

```bash
cd example
conan install . -pr:b=default -pr:h=../profiles/rk3562_aarch64 \
  -of build/armv8 --build=missing -c tools.build:jobs=4

cmake -S . -B build/armv8 \
  -DCMAKE_TOOLCHAIN_FILE=build/armv8/conan_toolchain.cmake \
  -DCMAKE_BUILD_TYPE=Release
cmake --build build/armv8 -j4

scp build/armv8/example build/armv8/bench root@<板子IP>:/root/
scp ../py/test_data/base.pcd root@<板子IP>:/root/
scp ../py/test_data/food/3.pcd root@<板子IP>:/root/
ssh root@<板子IP> '/root/bench /root/base.pcd /root/3.pcd 5'
```

> 前置条件：`arm-toolchain/14.3.rel1` 必须先在本地 Conan 缓存中建好，
> 且 PCL 的 `with_libusb` / `with_pcap` / `with_qt` / `with_opengl` 必须关闭
> （已写在 `../profiles/rk3562_aarch64` 里）——细节见上述文档。

## 用法

```
example <baseline.pcd> <food.pcd> [config.json] [--dump-middle[=DIR]]
bench   <baseline.pcd> <food.pcd> [runs] [config.json]
```

- `config.json` 可选，对应 `MeasurementConfig`；示例见 `../config/measurement_config.example.json`
- `--dump-middle[=DIR]` 让库把各阶段中间点云写到 `DIR`（默认 `middle_data/cpp`），
  便于用 Open3D / CloudCompare 逐阶段排查。这会顺序写出 6 个文件：

  | 文件 | 阶段 |
  |------|------|
  | `0_input_food.pcd` | 原始输入食材帧 |
  | `1_downsampled.pcd` | 体素降采样后 |
  | `2_remaining.pcd` | 移除背景平面后 |
  | `3_baseline_surface.pcd` | 空炉基线表面（由基线栅格重建） |
  | `4_food_components.pcd` | 选中的食材连通块（合并） |
  | `4_food_component_label<NN>.pcd` | 每个连通块单独一个，`NN` 为连通块标签 |
  | `5_top_surface.pcd` | 顶表面（体积积分所用） |

## 目录说明

```
example/
├── CMakeLists.txt           # 两个可执行目标
├── conanfile.py             # 声明 food_volume_measure 依赖
├── main.cpp                 # 功能演示
├── bench.cpp                # 端到端性能基准
├── README.md                # 本文件
└── cross-compile-rk3562.md  # RK3562 交叉编译与板端部署完整流程
```

`build/` 及其下所有产物均为生成物，不入版本管理。

```cpp
measurer.set_save_middle_cloud(true).set_middle_cloud_dir("middle_data/cpp");
```

`run()` 成功后写出各阶段点云；**每个选中的连通块还会单独写出一个 PCD**，
多食材场景可以逐个连通域排查。
