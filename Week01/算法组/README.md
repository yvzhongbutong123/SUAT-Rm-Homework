# CMake 自学作业（精简版）

> **用法**：按单元顺序做，每单元先读“讲解”，再动手，最后回答思考题。遇到报错先看编译命令和错误信息。
> **总时长**：约 2～3 小时。项目：`mycalc`（库 `calc` + 可执行 `mycalc`）。

---

## 单元 0：环境准备（10 分钟）

**目标**：确认工具链齐全。

```bash
cmake --version      # 需 ≥ 3.23
g++ --version        # 或 clang++ / cl
ninja --version      # 或 make --version
```

**理解**：CMake 不是编译器，它读 `CMakeLists.txt`，为当前平台生成 Makefile / build.ninja，再由 Make/Ninja 真正编译。

**验收**
- [ ] 三条命令都有输出，CMake ≥ 3.23
- [ ] 报告记录：系统、CMake 版本、编译器版本、构建工具

---

## 单元 1：跑通最小项目（30 分钟）

**目标**：理解“配置 + 构建”两阶段，认识七条核心命令。

### 讲解

CMake 永远是两步：

```
配置：cmake -S . -B build     # 读 CMakeLists.txt，生成构建文件到 build/
构建：cmake --build build     # 调用 Make/Ninja 真正编译链接
```

现代 CMake 的核心是 **Target（目标）**：一个可执行文件或一个库。`target_*` 命令都是给某个目标设属性。

### 动手

建目录 `mycalc/`：

```
mycalc/
├── CMakeLists.txt
├── include/calc.h
└── src/calc.cpp
└── src/main.cpp
```

`include/calc.h`
```cpp
#pragma once
int add(int a, int b);
int sub(int a, int b);
```

`src/calc.cpp`
```cpp
#include "calc.h"
int add(int a, int b) { return a + b; }
int sub(int a, int b) { return a - b; }
```

`src/main.cpp`
```cpp
#include "calc.h"
#include <iostream>
int main() {
    std::cout << add(1, 2) << "\n" << sub(5, 3) << "\n";
    return 0;
}
```

`CMakeLists.txt`
```cmake
cmake_minimum_required(VERSION 3.23)
project(mycalc VERSION 0.1.0 LANGUAGES CXX)

add_library(calc STATIC src/calc.cpp)
target_include_directories(calc PUBLIC include)
target_compile_features(calc PUBLIC cxx_std_17)

add_executable(mycalc src/main.cpp)
target_link_libraries(mycalc PRIVATE calc)
```

执行：
```bash
cmake -S . -B build
cmake --build build
./build/mycalc
```

输出：
```
3
2
```

### 七条命令一句话

| 命令 | 作用 |
| :--- | :--- |
| `cmake_minimum_required` | 声明最低版本，**必须第一行** |
| `project` | 项目名 + 启用语言，探测编译器 |
| `add_library(名 类型 源)` | 创建库目标（STATIC/SHARED/INTERFACE） |
| `add_executable(名 源)` | 创建可执行目标 |
| `target_include_directories(目标 作用域 目录)` | 给目标加头文件搜索路径 |
| `target_compile_features(目标 作用域 cxx_std_17)` | 给目标设 C++ 标准 |
| `target_link_libraries(目标 作用域 依赖)` | 让目标链接别的库 |

### 思考题
1. 只跑 `cmake --build build` 不先配置会怎样？为什么？
2. `-S .` 和 `-B build` 各代表什么？

**验收**
- [ ] 构建运行成功
- [ ] 能用自己的话解释七条命令
- [ ] 思考题已答

---

## 单元 2：PUBLIC / PRIVATE / INTERFACE（40 分钟）

**目标**：彻底理解属性传播，能通过编译命令判断。

### 讲解

设 **A 链接 B**（A 依赖 B）。给 B 设属性时：

| 关键字 | 给 B 自己用？ | 传给 A？ | 口诀 |
| :--- | :---: | :---: | :--- |
| `PRIVATE` | ✅ | ❌ | 只给自己 |
| `INTERFACE` | ❌ | ✅ | 只给别人 |
| `PUBLIC` | ✅ | ✅ | 给自己也给别人 |

**看编译命令**：在 `CMakeLists.txt` 顶部临时加 `set(CMAKE_VERBOSE_MAKEFILE ON)`，重新构建，观察 `calc.cpp` 和 `main.cpp` 的编译命令里**有没有 `-I.../include`**。有就是拿到了路径，没有就是没拿到。

### 动手：三个实验

每个实验先**预测**结果，再执行。

**实验 A：改成 `PRIVATE`**
```cmake
target_include_directories(calc PRIVATE include)
```
重新配置构建。预期：`calc.cpp` 编过，`main.cpp` 报 `calc.h: No such file or directory`。

**实验 B：改成 `INTERFACE`**
```cmake
target_include_directories(calc INTERFACE include)
```
预期：`main.cpp` 编过，`calc.cpp` 报错。

**实验 C：改回 `PUBLIC`**，确认恢复。

### 观察表（填报告）

| 作用域 | `calc.cpp` 命令有 `-I`？ | `main.cpp` 命令有 `-I`？ | 谁失败 |
| :--- | :---: | :---: | :--- |
| PUBLIC | | | |
| PRIVATE | | | |
| INTERFACE | | | |

### 思考题
1. 为什么这里必须用 `PUBLIC`？`PRIVATE` 和 `INTERFACE` 各挂在哪一步？
2. `target_link_libraries(mycalc PRIVATE calc)` 为什么 `PRIVATE` 就够？（提示：谁还会链接 `mycalc`？）

**验收**
- [ ] 两个错误都亲手复现
- [ ] 观察表与理论一致
- [ ] 项目已恢复 `PUBLIC`
- [ ] 思考题已答

---

## 单元 3：C++ 标准的传播（20 分钟）

**目标**：理解 `target_compile_features` 的传播。

### 动手

把 `main.cpp` 改成用 C++17 的 `std::optional`：

```cpp
#include "calc.h"
#include <iostream>
#include <optional>

int main() {
    std::optional<int> x = 42;
    std::cout << add(1, 2) << "\n" << sub(5, 3) << "\n";
    if (x) std::cout << *x << "\n";
    return 0;
}
```

1. 保持 `PUBLIC cxx_std_17`，构建成功，看编译命令里有 `-std=c++17`；
2. 改成 `PRIVATE`，重新构建。若你的编译器默认已 ≥ C++17 不报错，临时在文件顶部加：
   ```cmake
   set(CMAKE_CXX_STANDARD 14)
   set(CMAKE_CXX_STANDARD_REQUIRED ON)
   ```
   再观察 `mycalc` 是否编译失败（实验完删掉这两行）；
3. 改回 `PUBLIC`。

### 思考题
1. C++ 标准为什么该用 `PUBLIC` 传播给使用者？
2. 全局 `set(CMAKE_CXX_STANDARD 17)` 和 `target_compile_features` 各适合什么场景？
3. `cxx_std_17` 是“至少 C++17”还是“恰好 C++17”？

**验收**
- [ ] 观察过 `PUBLIC` 时 `main.cpp` 命令里有 `-std=c++17`
- [ ] 项目已恢复 `PUBLIC`
- [ ] 思考题已答

---

## 单元 4：静态库与动态库（20 分钟）

**目标**：观察 `STATIC` → `SHARED` 的产物变化。

### 动手

1. 当前 `STATIC`，构建后 `ls build`，记录产物（Linux：`libcalc.a`，Win：`calc.lib`）；
2. 改成 `add_library(calc SHARED src/calc.cpp)`，重新配置构建，记录产物（Linux：`libcalc.so`，Win：`calc.dll` + `calc.lib`）；
3. 检查运行时依赖：
   ```bash
   ldd ./build/mycalc | grep calc    # Linux
   otool -L ./build/mycalc | grep calc   # macOS
   ```
   静态时无输出，动态时有 `libcalc.so`；
4. 改回 `STATIC`。

### 思考题
1. 静态链接和动态链接各有什么优缺点？
2. 只把 `mycalc` 拷到别的机器，`SHARED` 时会怎样？为什么？

**验收**
- [ ] 两种类型产物都记录
- [ ] 用 `ldd` 验证过依赖差异
- [ ] 已改回 `STATIC`

---

## 单元 5：CMakePresets.json（30 分钟）

**目标**：把配置参数固化到文件，命令行和 IDE 都能用。

### 讲解

Presets 解决“每次敲一堆参数、团队不一致”的问题。`cmake --preset debug` 展开后等价于：

```bash
cmake -S . -B build/debug -G Ninja -DCMAKE_BUILD_TYPE=Debug -DCMAKE_EXPORT_COMPILE_COMMANDS=ON
```

**Preset 只是参数集的别名。**

### 动手

在 `mycalc/` 根目录建 `CMakePresets.json`：

```json
{
  "version": 6,
  "cmakeMinimumRequired": { "major": 3, "minor": 23, "patch": 0 },
  "configurePresets": [
    {
      "name": "debug",
      "displayName": "Debug (Ninja)",
      "generator": "Ninja",
      "binaryDir": "${sourceDir}/build/debug",
      "cacheVariables": {
        "CMAKE_BUILD_TYPE": "Debug",
        "CMAKE_EXPORT_COMPILE_COMMANDS": "ON"
      }
    },
    {
      "name": "release",
      "displayName": "Release (Ninja)",
      "inherits": "debug",
      "binaryDir": "${sourceDir}/build/release",
      "cacheVariables": { "CMAKE_BUILD_TYPE": "Release" }
    }
  ],
  "buildPresets": [
    { "name": "debug",   "configurePreset": "debug" },
    { "name": "release", "configurePreset": "release" }
  ]
}
```

执行：
```bash
rm -rf build
cmake --list-presets
cmake --preset debug
cmake --build --preset debug
./build/debug/mycalc
```

再走一遍 release，确认 `build/release/` 也有产物。

### 关键字段

| 字段 | 作用 |
| :--- | :--- |
| `name` | 唯一标识，`--preset <name>` 引用 |
| `generator` | `Ninja` 或 `Unix Makefiles` |
| `binaryDir` | 构建目录，`${sourceDir}` 是源码根 |
| `cacheVariables` | 等价 `-D<k>=<v>` |
| `inherits` | 继承另一个预设，只写差异 |
| `buildPresets[].configurePreset` | 指定基于哪个配置预设构建 |

### 思考题
1. `inherits` 省了什么重复？再加一个 `asan` 预设你会继承谁、覆盖什么？
2. `binaryDir` 为什么写 `${sourceDir}/build/debug` 而不是 `build/debug`？

**验收**
- [ ] `--list-presets` 能看到 debug/release
- [ ] 两条流程都跑通，两个目录各有产物
- [ ] 能写出 `cmake --preset debug` 展开后的完整命令

---

## 单元 6：多目录项目（20 分钟）

**目标**：用 `add_subdirectory` 把库拆到子目录。

### 动手

目标结构：

```
mycalc/
├── CMakeLists.txt
├── CMakePresets.json
├── include/calc.h
├── lib/
│   ├── CMakeLists.txt
│   └── calc.cpp
└── src/main.cpp
```

把 `src/calc.cpp` 移到 `lib/calc.cpp`。

**顶层 `CMakeLists.txt`**
```cmake
cmake_minimum_required(VERSION 3.23)
project(mycalc VERSION 0.1.0 LANGUAGES CXX)

add_subdirectory(lib)

add_executable(mycalc src/main.cpp)
target_link_libraries(mycalc PRIVATE calc)
```

**`lib/CMakeLists.txt`**
```cmake
add_library(calc STATIC calc.cpp)
target_include_directories(calc PUBLIC ${CMAKE_SOURCE_DIR}/include)
target_compile_features(calc PUBLIC cxx_std_17)
```

重新构建运行。

### 讲解：两个变量

| 变量 | 含义 |
| :--- | :--- |
| `CMAKE_SOURCE_DIR` | 整个项目的源码根 |
| `CMAKE_CURRENT_SOURCE_DIR` | 当前 `CMakeLists.txt` 所在目录 |

在 `lib/` 里，前者是 `mycalc/`，后者是 `mycalc/lib/`。`include/` 在根，所以用 `CMAKE_SOURCE_DIR`。

### 思考题
1. 子目录为什么不需要再写 `project()`？
2. 如果写成 `CMAKE_CURRENT_SOURCE_DIR/include` 会怎样？为什么？
3. `add_subdirectory(lib)` 为什么必须在 `target_link_libraries(mycalc PRIVATE calc)` 之前？

**验收**
- [ ] 重构后正常构建运行
- [ ] 能说清两个变量的区别
- [ ] 思考题已答

---

## 单元 7（挑战，选做）：INTERFACE 库（20 分钟）

**目标**：理解“无源文件、纯接口”的库，以及链接传播链。

### 动手

改写 `lib/CMakeLists.txt`：

```cmake
add_library(calc_headers INTERFACE)
target_include_directories(calc_headers INTERFACE ${CMAKE_SOURCE_DIR}/include)
target_compile_features(calc_headers INTERFACE cxx_std_17)

add_library(calc STATIC calc.cpp)
target_link_libraries(calc PUBLIC calc_headers)
```

重新构建，确认结果不变。

### 讲解

传播链：

```
calc_headers --(INTERFACE 属性)--> calc --(PUBLIC 链接)--> mycalc
```

`INTERFACE` 库自身不编译任何东西，所以所有属性都用 `INTERFACE` 关键字。

### 思考题
1. `INTERFACE` 库为什么不能有源文件？
2. 把 `target_link_libraries(calc PUBLIC calc_headers)` 改成 `PRIVATE`，`mycalc` 还能拿到 `include` 和 C++17 吗？预测后验证。

**验收**
- [ ] 重构后正常构建
- [ ] 能手写出传播链
- [ ] 思考题 2 亲手验证

---

## 附录 A：思考题参考要点

> 先自己写，再对照。不必逐字一致，概念对即可。

**单元 1**
- Q1：会报错，`build/` 里还没有构建文件。`cmake --build` 只是转发器，不负责生成构建文件。
- Q2：`-S` 源码目录，`-B` 构建目录。推荐是因为源码外构建、一条命令、适合 CI。

**单元 2**
- Q1：`PRIVATE` 时 `mycalc` 拿不到 `-I`，编译 `main.cpp` 失败；`INTERFACE` 时 `calc` 拿不到，编译 `calc.cpp` 失败；只有 `PUBLIC` 两者都拿到。
- Q2：`mycalc` 是可执行文件，没有下游，不需要传播。

**单元 3**
- Q1：库的公开头文件可能用到标准特性，使用者以低标准编译会出错，所以“接口要求的最低标准”应传播。
- Q2：全局适合小项目统一标准；按目标适合多标准并存。
- Q3：至少 C++17。编译命令里可能看到更高值。

**单元 4**
- Q1：静态部署简单、文件大、更新需重链；动态共享、可独立更新、部署需带库文件。
- Q2：`SHARED` 会因找不到 `libcalc.so` 启动失败。

**单元 5**
- Q1：`release` 继承 `debug` 的 `generator`，只覆盖 `binaryDir` 和 `CMAKE_BUILD_TYPE`。`asan` 可继承 `debug`，覆盖 `cacheVariables` 加 sanitizer 选项。
- Q2：`${sourceDir}` 保证解析到绝对路径，不受当前工作目录影响。

**单元 6**
- Q1：`project` 作用于整个项目，子目录通过 `add_subdirectory` 共享顶层上下文，重复写会重置变量。
- Q2：会解析成 `mycalc/lib/include`，目录不存在，找不到头文件。
- Q3：CMake 按顺序解析，链接一个尚未定义的目标会报错。

**单元 7**
- Q1：`INTERFACE` 库定义就是“无编译产物的目标”，传源文件会报错。
- Q2：`calc` 自己仍拿到（它链接了 `calc_headers`），但不再传给 `mycalc`，`mycalc` 编译失败。

---

## 附录 B：常见错误速查

| 现象 | 原因 |
| :--- | :--- |
| `calc.h: No such file or directory` | `target_include_directories` 缺失/写错/作用域错 |
| `undefined reference to add(int, int)` | 忘了 `target_link_libraries` 或目标名错 |
| 改 `CMakeLists.txt` 不生效 | 没重新配置 |
| `Could not find preset` | `name` 拼错，或不在根目录执行 |
| `Generator Ninja not found` | 没装 Ninja，改 `Unix Makefiles` |
| `INTERFACE` 库报 “no SOURCES” | 给 INTERFACE 库传了源文件 |

---

## 附录 C：提交自检

**报告建议内容**
1. 环境信息
2. 每单元的观察表 + 思考题答案
3. 至少 1 条踩坑记录（现象 / 原因 / 解决）
4. 一段总结：你对“基于目标的现代 CMake + 作用域 + Presets”的理解

**交前检查**
- [ ] 删除所有 `build/`
- [ ] 从零能跑通：`cmake --preset debug && cmake --build --preset debug && ./build/debug/mycalc`
- [ ] 没有全局 `include_directories` / `link_libraries`
- [ ] 思考题全部作答
- [ ] 踩坑记录 ≥ 1 条

