# food3d_im_test

这是 [food3d_im](https://github.com/HeT-FTI/food3d_im) 的独立消费者工程。
它通过 Conan 安装 `food3d_im/0.1.0`，再用安装后的公开头文件和 CMake target
编译示例，不直接编译主库的 `src/`。本仓库用于检查包能否被下游工程使用，并在点云数据
可用时运行测量和性能基准。

目前提供两套 **Linux** 构建配置：桌面 x86_64（GCC 13）和 RK3562 aarch64（GCC 12.3
交叉编译）。Windows 和 macOS 没有对应的 Conan profile；`bench.cpp` 使用
`sys/resource.h`，其中峰值内存的换算按 Linux 的单位处理。

## 测试范围

| 位置 | 验证内容 | 运行位置 |
|------|----------|----------|
| 主库的 `test_package/` | `conan create` 后的包消费检查；主库开启 `trigger_tests` 时运行完整 GTest/CTest | 本机构建时在主机运行；交叉构建时跳过执行 |
| 本仓库 `example/` | 用安装的 Conan 包编译、链接 `example` 与 `bench`，分别检查测量结果和 `run()` 耗时 | 桌面或 RK3562 板端 |
| 本仓库 `py/` | Python 绑定的手工冒烟测试、点云导出及算法实验 | 与扩展模块 ABI 匹配的桌面 Python 环境 |

`example/conanfile.py` 固定依赖 `food3d_im/0.1.0`。库包和消费者必须使用
相同的 **host profile** 与 Release 配置；RK3562 的 profile 还固定了 PCL 的选项。

## 开始前

- 安装 Conan 2、CMake 3.28+，并准备与所选 profile 匹配的编译器。
- 将 `food3d_im` 与本仓库放在同一父目录。下方命令以这个目录为起点。
- 准备一组使用同一坐标系采集的空盘和食材 PCD。建议分别放在
  `py/test_data/base.pcd` 与 `py/test_data/food/3.pcd`。
  `*.pcd` 被 `.gitignore` 排除，**克隆仓库后不会自动获得点云数据**；
  `py/test_data/ground_truth.json` 仅保存数值真值，不能代替输入 PCD。
- RK3562 构建前，按 [交叉编译指南](example/cross-compile-rk3562.md) 将
  `arm-toolchain/12.3.rel1` 放进本地 Conan 缓存，并确认板端运行的是 aarch64 Linux。

### 配置文件兼容性

`example/main.cpp` 和 `example/bench.cpp` 已使用主库当前的
`set_middle_cloud_dir()` 接口。两份旧 JSON 示例仍需更新：

`config/measurement_config.example.json` 和 `example/param.json` 仍含已删除的字段
（例如 `save_middle_cloud`、`cluster_eps_m`）。当前主库会拒绝未知 JSON 键；
验证基本构建与运行时，先不要向 `example` 或 `bench` 传这些配置文件。

## Linux x86_64：编译并运行

先在主库目录用本仓库的 `profiles/gcc13_release` 创建 Conan 包：

```bash
# 从两个仓库的共同父目录执行
cd food3d_im
conan create . -pr:b=default -pr:h=../food3d_im_test/profiles/gcc13_release \
    --build=missing -c tools.build:jobs=4
```

然后在本仓库编译消费者：

```bash
cd ../food3d_im_test/example
conan install . -pr:b=default -pr:h=../profiles/gcc13_release \
    -of build/desktop --build=missing
cmake -S . -B build/desktop \
    -DCMAKE_TOOLCHAIN_FILE=build/desktop/conan_toolchain.cmake \
    -DCMAKE_BUILD_TYPE=Release
cmake --build build/desktop -j4
```

有 PCD 数据后，运行功能示例和五次测量的基准：

```bash
./build/desktop/example ../py/test_data/base.pcd ../py/test_data/food/3.pcd
./build/desktop/bench ../py/test_data/base.pcd ../py/test_data/food/3.pcd 5
```

`example` 打印状态、总体积和逐连通块诊断；只有状态为 `kSuccess` 时才使用体积数值。
`bench` 在计时前读入点云，报告 `run()` 的墙钟/CPU 时间与进程峰值内存。
当前源码在 `run()` 中启用了中间点云保存，因此计时包含输出 PCD 的 I/O，不是纯算法
耗时。可将 `food/3.pcd` 的输出与
[ground_truth.json](py/test_data/ground_truth.json) 中对应的 `gt_volume_cm3` 比较。

## RK3562 aarch64：交叉编译与板端运行

`profiles/rk3562_aarch64` 使用 `os=Linux`、`arch=armv8`、GCC 12 和
`arm-toolchain/12.3.rel1`。它通过 `include(pcl-minimal)` 关闭 PCL 的 libusb、pcap、
Qt、OpenGL 选项；桌面 profile `gcc13_release` 也引入同一个文件，因此两边选项不会
漂移，命令行无需再写 `-o pcl/*:...`。
**创建库包和安装示例依赖时都使用这个 profile**，否则 Conan 可能需要另一套包。

```bash
# 同样从两个仓库的共同父目录执行
cd food3d_im
conan create . -pr:b=default -pr:h=../food3d_im_test/profiles/rk3562_aarch64 \
    --build=missing -c tools.build:jobs=4

cd ../food3d_im_test/example
conan install . -pr:b=default -pr:h=../profiles/rk3562_aarch64 \
    -of build/armv8 --build=missing -c tools.build:jobs=4
cmake -S . -B build/armv8 \
    -DCMAKE_TOOLCHAIN_FILE=build/armv8/conan_toolchain.cmake \
    -DCMAKE_BUILD_TYPE=Release
cmake --build build/armv8 -j4
file build/armv8/example build/armv8/bench
```

`file` 应显示 aarch64 ELF。交叉构建只在主机上完成编译、链接检查；可执行文件需要
拷贝到 RK3562 上运行（把 `BOARD_IP` 换成板子的实际地址）：

```bash
scp build/armv8/example build/armv8/bench root@BOARD_IP:/root/
scp ../py/test_data/base.pcd root@BOARD_IP:/root/
scp ../py/test_data/food/3.pcd root@BOARD_IP:/root/
ssh root@BOARD_IP '/root/example /root/base.pcd /root/3.pcd'
ssh root@BOARD_IP '/root/bench /root/base.pcd /root/3.pcd 5'
```

检查测量状态、体积与桌面结果，并比较耗时。不同架构的浮点结果可能有小幅差异。
工具链创建、板端运行库与常见错误见
[交叉编译指南](example/cross-compile-rk3562.md)。

## Python 绑定与中间点云

Python 是手工验证路径，不参与上面的 C++ 跨平台构建。主库启用
`enable_python_bindings` 时，可在与扩展模块 ABI 匹配的 Python 环境中，从本仓库
根目录部署已创建的桌面包：

```bash
conan install --requires=food3d_im/0.1.0 \
    -pr:b=default -pr:h=profiles/gcc13_release \
    -of pylib --deployer=direct_deploy -nr
```

`py/vm_module.py` 从 `pylib/direct_deploy/food3d_im/lib/` 加载
`food3d_im_python`。当前 `py/test_pybind_module.py` 和
`py/dump_intermediate_pcd.py` 的默认输入文件名与仓库中的
`base.pcd` / `food/<case>.pcd` 约定不一致，且部分调用仍使用已删除的
`set_save_middle_cloud()`。运行这些脚本前需同步脚本接口和输入路径。

新接口只需设置输出目录即可打开中间点云导出：

```cpp
measurer.set_middle_cloud_dir("middle_data/cpp");
```

成功测量且对应阶段有点云时，库会写出以下文件；选中的每个连通块另有
`4_food_component_label<NN>.pcd`：

| 文件 | 内容 |
|------|------|
| `0_input_food.pcd` | 原始食材点云 |
| `1_downsampled.pcd` | 降采样结果 |
| `2_remaining.pcd` | 移除背景平面后的点云 |
| `3_baseline_surface.pcd` | 空盘基线表面 |
| `4_food_components.pcd` | 合并后的选中食材连通块 |
| `5_top_surface.pcd` | 补洞前的实测顶表面 |
| `6_hole_filled_surface.pcd` | 包含通过检查的补洞栅格的顶表面 |

## 目录

| 路径 | 用途 |
|------|------|
| `profiles/` | 桌面和 RK3562 的 Conan host profiles；`pcl-minimal` 是两者共用的 PCL 选项 |
| `example/` | C++ 消费者、性能基准及板端部署说明 |
| `py/` | Python 绑定测试、点云导出和算法实验 |
| `py/test_data/ground_truth.json` | 样本真值；PCD 输入文件需自行提供 |
| `config/` | 配置示例；当前版本需先删去主库不再接受的字段 |
| `scripts/` | 文档配图生成脚本 |

`example/build/`、`pylib/`、`middle_data/` 等构建和运行产物不入库。
