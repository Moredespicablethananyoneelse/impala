
******************************
Apache Impala 通过一套高度自动化的 CMake + Python + Shell 脚本 流程，将 C++ 源码编译为 LLVM IR（Intermediate Representation），再将其嵌入到最终的二进制中，实现 运行时 JIT（Just-In-Time）编译。下面结合你提供的 LlvmCodeGen::CreateFromMemory 函数和 CMake 脚本，系统性地解释整个流程。

🧩 一、整体目标

Impala 的核心性能优化依赖于 动态代码生成（Codegen）。为了高效实现：
将通用算子（如 HashJoin、Aggregation、Expression Evaluation）预先编译成 LLVM IR，运行时根据查询需求加载并 JIT 成机器码。

这样既避免了每次从 C++ 源码编译的开销，又能利用 LLVM 强大的优化能力。

🔧 二、关键组件与文件

组件 说明
------ ------
impala-ir.cc 唯一 IR 源文件，包含所有可 Codegen 的函数（用 IMPALA_UDF_EXPORT 标记）
gen_ir_descriptions.py 扫描 impala-ir.cc，生成函数名列表（impala-ir-functions.h）供 C++ 调用
file2array.sh 将 .bc（LLVM bitcode）文件转换为 C 数组（如 unsigned char impala_llvm_o2_ir[] = {...}）
COMPILE_TO_IR_C_ARRAY() CMake 宏，封装“编译 → 优化 → 转数组”全过程
LlvmCodeGen::CreateFromMemory() 运行时从这些 C 数组加载 IR 并初始化 JIT 环境

🏗️ 三、编译流程详解（基于 CMakeLists.txt）
Step 1️⃣：定义输出路径和变量

cmake
set(IR_O1_C_FILE $ENV{IMPALA_HOME}/be/generated-sources/impala-ir/impala-ir-o1.cc)
... O2, Os, legacy-avx ...
所有生成的 C 数组文件放在 be/generated-sources/impala-ir/；
文件名体现优化级别和 CPU 特性。

Step 2️⃣：注册自定义构建命令（COMPILE_TO_IR_C_ARRAY）

这是一个 CMake 函数宏，接收：
输出 C 文件路径（如 impala-ir-o2.cc）
C 数组变量名（如 impala_llvm_o2_ir）
额外 Clang 编译标志（如 -O2 -mavx2）
内部执行两步：
✅ 2.1 编译 C++ → LLVM Bitcode（.bc）

cmake
COMMAND ${LLVM_CLANG_EXECUTABLE} ${CLANG_IR_CXX_FLAGS} -O2 -mavx2
${CLANG_INCLUDE_FLAGS} impala-ir.cc -o tmp.bc
COMMAND ${LLVM_OPT_EXECUTABLE} < tmp.bc > optimized.bc
使用 Clang 直接生成 LLVM IR（bitcode），而非汇编或目标文件；
-mavx2：启用 AVX2 指令集（x86_64 默认）；
-mavx：用于 legacy 模式；
opt 工具应用 LLVM 优化（对应 -O1/-O2/-Os）；
输入是 单一文件 impala-ir.cc，其中包含所有可 codegen 函数。
💡 为什么用 opt 而不是靠 Clang？
因为 Impala 需要精确控制优化级别，并可能在后续做自定义 Pass。
✅ 2.2 Bitcode → C 数组（嵌入二进制）

cmake
COMMAND file2array.sh -n -v impala_llvm_o2_ir optimized.bc > impala-ir-o2.cc.tmp
COMMAND mv ... .cc
file2array.sh 是一个 shell 脚本，本质是：
bash
xxd -i optimized.bc sed 's/unsigned char . =/const unsigned char VARNAME[] =/'
生成的 impala-ir-o2.cc 内容类似：
cpp
const unsigned char impala_llvm_o2_ir[] = {
0x42, 0x43, 0xc0, 0xde, ... // LLVM bitcode magic + content
};
const size_t impala_llvm_o2_ir_len = 12345;

Step 3️⃣：调用宏生成四套 IR 变体

cmake
COMPILE_TO_IR_C_ARRAY(${IR_O1_C_FILE} impala_llvm_o1_ir -O1 ${PLATFORM_SPECIFIC_FLAGS})
COMPILE_TO_IR_C_ARRAY(${IR_O2_C_FILE} impala_llvm_o2_ir -O2 ${PLATFORM_SPECIFIC_FLAGS})
COMPILE_TO_IR_C_ARRAY(${IR_Os_C_FILE} impala_llvm_os_ir -Os ${PLATFORM_SPECIFIC_FLAGS})
COMPILE_TO_IR_C_ARRAY(${LEGACY_AVX_IR_C_FILE} impala_legacy_avx_llvm_ir -O1 ${LEGACY_AVX_SPECIFIC_FLAGS})
三套优化级别：O1（平衡）、O2（性能优先）、Os（代码尺寸优先）；
一套 legacy AVX：仅当 CPU 不支持 AVX2 但启用 --enable_legacy_avx_support 时使用；
所有 .cc 文件被加入 CodeGen 库，链接进 libexecbackend.so 或主程序。

Step 4️⃣：生成函数符号元数据（gen_ir_descriptions.py）

cmake
add_custom_command(
OUTPUT impala-ir-names.h impala-ir-functions.h
COMMAND ./gen_ir_descriptions.py
)
该脚本解析 impala-ir.cc 中所有用 IMPALA_UDF_EXPORT 标记的函数；
生成头文件，例如：
cpp
// impala-ir-functions.h
#define IR_FUNCTIONS \
X(Add, "Add") \
X(Multiply, "Multiply") \
...
C++ 代码通过这个宏遍历所有 IR 函数，用于：
JIT 时查找函数地址；
构建调用图（Call Graph）；
符号去重与缓存。

⚙️ 四、运行时加载（LlvmCodeGen::CreateFromMemory）

现在回到你最初的问题：运行时如何使用这些 IR？
1. 根据启动参数选择 IR

cpp
if (FLAGS_llvm_ir_opt == "O2") {
module_ir = StringRef(impala_llvm_o2_ir, impala_llvm_o2_ir_len);
}
直接引用编译时生成的全局数组；
零文件 I/O，零动态分配，极致高效。
2. 根据 CPU 特性覆盖选择

cpp
#if __x86_64__
if (!AVX2 && FLAGS_enable_legacy_avx_support && AVX) {
module_ir = StringRef(impala_legacy_avx_llvm_ir, ...);
}
#endif
实现 指令集自适应，确保兼容性与性能兼顾。
3. 加载为 LLVM Module

cpp
MemoryBuffer::getMemBuffer(module_ir, "", false);
getOwningLazyBitcodeModule(...); // lazy load
使用 Lazy Loading：函数只有在被调用时才解析 IR，节省内存；
后续可通过 ExecutionEngine::getFunctionAddress("Add") 获取 JIT 后的函数指针。

🧠 五、设计优势总结

优势 说明
------ ------
启动快 IR 内嵌二进制，无需读取外部文件
性能高 预优化 IR + JIT，接近手写汇编
兼容性强 支持 AVX2 / AVX / ARM 多平台
调试友好 可通过 --dump_ir 导出 IR 供分析
安全可控 禁用未测试的 CPU 特性（如 AVX-512）
模块化 IR 与主逻辑解耦，易于维护

📌 六、补充：impala-ir.cc 长什么样？

虽然你没提供，但典型结构如下：

cpp
// be/src/codegen/impala-ir.cc
#include "impala-ir/impala-ir-functions.h"

// 所有函数必须用此宏标记，供 gen_ir_descriptions.py 识别
#define IMPALA_UDF_EXPORT extern "C" __attribute__((visibility("default")))

IMPALA_UDF_EXPORT int32_t Add(int32_t a, int32_t b) {
return a + b;
}

IMPALA_UDF_EXPORT void Memcpy(void dst, const void* src, int64_t n) {
memcpy(dst, src, n);
}
// ... 更多算子

✅ 结论

Apache Impala 通过 “预编译 IR → 嵌入二进制 → 运行时按需 JIT” 的模式，实现了高性能、可移植、易维护的动态代码生成系统。整个流程由 CMake 自动化驱动，结合 Python 和 Shell 脚本完成元数据提取和格式转换，最终在 LlvmCodeGen 中无缝集成，是现代大数据系统中 编译技术与工程实践结合的典范。
*************************************************************88
请解释：

```cmake

set(THIRDPARTY_LLVM_SRC_DIR "${CMAKE_SOURCE_DIR}/be/src/thirdparty/llvm")
```
这段代码位于 Apache Impala 的 `be/src/codegen/CMakeLists.txt` 文件中，是一个非常简单的变量定义语句：

```cmake
set(THIRDPARTY_LLVM_SRC_DIR "${CMAKE_SOURCE_DIR}/be/src/thirdparty/llvm")
```

### 详细解释

#### 1. 变量的作用
它定义了一个 CMake 变量 `THIRDPARTY_LLVM_SRC_DIR`，其值指向 Impala 项目源码树中一个特定的目录：

```
<Impala 项目根目录>/be/src/thirdparty/llvm/
```

这个目录存放了 **Impala 从 LLVM 项目中复制并稍作修改的一部分源码**。

#### 2. 为什么 Impala 需要自己的 LLVM 源码拷贝？

Impala 的运行时代码生成（runtime codegen）高度依赖 LLVM（尤其是 ORC JIT 引擎和 IR 优化部分），但它**不直接使用系统安装的 LLVM 库**，而是：

- 在自己的 toolchain 中**预编译并打包了完整的 LLVM 库**（libLLVM*.so）。
- 同时在源码树中**保留了一小部分 LLVM 的原始源码**（位于 `be/src/thirdparty/llvm/`）。

具体来说，这个目录目前主要包含一个文件：
```
SectionMemoryManager.cpp
```

这个文件是 LLVM 的一个组件，实现了一个自定义的内存管理器，用于 JIT 分配可执行内存段。

#### 3. 为什么不直接用 LLVM 官方的版本？

原因有几个：
- **兼容性**：Impala 需要特定版本的 LLVM 行为（早期使用 LLVM 5.0，后来升级到更高版本），直接使用系统 LLVM 可能版本不匹配。
- **定制修改**：Impala 对 `SectionMemoryManager` 可能做了轻微修改或打补丁，以适应自己的 JIT 需求（例如内存对齐、权限设置等）。
- **自包含**：Impala 的 toolchain 追求**自包含、可移植**，避免依赖主机系统的 LLVM 开发包。

#### 4. 这行代码在后续的使用

在同一个 CMakeLists.txt 中，后面有：
```cmake
add_library(CodeGen
  ...
  ${THIRDPARTY_LLVM_SRC_DIR}/SectionMemoryManager.cpp
  ...
)
```

这行代码把 Impala 自己维护的 `SectionMemoryManager.cpp` **直接编译进 CodeGen 库**，而不是链接外部 LLVM 库中的版本。

这样可以确保：
- 使用的是 Impala 测试和验证过的实现。
- 避免与系统 LLVM 版本冲突。

### 总结

这段代码的核心目的：

> **定义一个路径变量，指向 Impala 项目中自行维护的一小部分 LLVM 源码**（主要是 `SectionMemoryManager.cpp`）。  
> 这样 Impala 可以在完全控制的情况下编译并使用这个关键组件，  
> 保证运行时代码生成的稳定性、可移植性和兼容性。

这是大型高性能系统（如 Impala）在深度依赖复杂第三方库（如 LLVM）时的常见做法：**不完全信任外部版本，而是保留关键部分的控制权**。
*************************************************************
请解释：

```cmake

# where to put generated libraries
set(LIBRARY_OUTPUT_PATH "${BUILD_OUTPUT_ROOT_DIRECTORY}/codegen")

# where to put generated binaries
set(EXECUTABLE_OUTPUT_PATH "${BUILD_OUTPUT_ROOT_DIRECTORY}/codegen")
```
这段代码位于 Apache Impala 的 `be/src/codegen/CMakeLists.txt` 文件中，是 CMake 中非常常见的输出路径设置语句。

```cmake
# where to put generated libraries
set(LIBRARY_OUTPUT_PATH "${BUILD_OUTPUT_ROOT_DIRECTORY}/codegen")

# where to put generated binaries
set(EXECUTABLE_OUTPUT_PATH "${BUILD_OUTPUT_ROOT_DIRECTORY}/codegen")
```

### 详细解释

#### 1. 两个变量的含义（CMake 旧版变量）

- `LIBRARY_OUTPUT_PATH`  
  控制**所有库目标**（`add_library` 生成的 .a 或 .so）输出到的目录。

- `EXECUTABLE_OUTPUT_PATH`  
  控制**所有可执行目标**（`add_executable` 生成的二进制程序）输出到的目录。

这两个是 CMake **较早版本**（2.8 左右）提供的全局变量，后来被更精细的 `CMAKE_LIBRARY_OUTPUT_DIRECTORY` 和 `CMAKE_RUNTIME_OUTPUT_DIRECTORY` 取代，但许多老项目（包括 Impala）仍在使用。

#### 2. 路径值解析
```cmake
"${BUILD_OUTPUT_ROOT_DIRECTORY}/codegen"
```

- `${BUILD_OUTPUT_ROOT_DIRECTORY}` 是顶层 CMakeLists.txt 中根据构建类型（Debug/Release）定义的变量：
  - Debug 类构建 → `be/build/debug/`
  - Release 构建 → `be/build/release/`
- 因此最终路径是：
  - Debug 构建：`be/build/debug/codegen/`
  - Release 构建：`be/build/release/codegen/`

#### 3. 为什么要把 CodeGen 模块的输出单独放到 codegen/ 子目录？

Impala 的构建系统会把每个模块的产物放到**独立的子目录**，好处包括：

- **组织清晰**：一眼就能看出哪些文件属于哪个模块。
  - `codegen/` 下只有 CodeGen 库和可能的测试程序。
  - `runtime/` 下只有 Runtime 相关产物。
- **避免冲突**：不同模块可能生成同名文件（如测试程序 `test`），放到不同子目录避免覆盖。
- **并行构建友好**：CMake 可以更高效地跟踪依赖和增量构建。

#### 4. 实际效果

在这个 CMakeLists.txt 中：
- `add_library(CodeGen ...)` 生成的 `libCodeGen.a`（或 .so）会输出到：
  ```
  be/build/debug/codegen/libCodeGen.a
  ```
- 如果有 `add_executable`（本文件中没有，但测试可能有），可执行文件也会放到同一个 `codegen/` 目录。

#### 5. 现代 CMake 的更好实践（参考）

当前 CMake 推荐**按目标设置**，而不是全局变量：
```cmake
set_target_properties(CodeGen PROPERTIES
  LIBRARY_OUTPUT_DIRECTORY "${BUILD_OUTPUT_ROOT_DIRECTORY}/codegen"
  ARCHIVE_OUTPUT_DIRECTORY "${BUILD_OUTPUT_ROOT_DIRECTORY}/codegen"  # 静态库
  RUNTIME_OUTPUT_DIRECTORY "${BUILD_OUTPUT_ROOT_DIRECTORY}/codegen"  # 可执行文件
)
```

但 Impala 作为历史悠久的项目，继续使用旧的全局变量方式是完全合理的（稳定、简单）。

### 总结

这两行代码的核心目的：

> **把 CodeGen 模块生成的所有库和可执行文件统一输出到构建目录下的 `codegen/` 子目录**（debug 或 release 根据构建类型）。  
> 实现模块化构建产物的组织隔离，提高构建系统的清晰度、可维护性和并行效率。

这是 Impala 构建系统中常见的模块输出路径管理模式，每个主要模块（如 runtime、exec、exprs 等）都会在自己的 CMakeLists.txt 中类似设置自己的输出子目录。

******************************************************************
请解释：

```cmake

# Generated C files for IR
set(IR_O1_C_FILE $ENV{IMPALA_HOME}/be/generated-sources/impala-ir/impala-ir-o1.cc)
set(IR_O2_C_FILE $ENV{IMPALA_HOME}/be/generated-sources/impala-ir/impala-ir-o2.cc)
set(IR_Os_C_FILE $ENV{IMPALA_HOME}/be/generated-sources/impala-ir/impala-ir-os.cc)
set(LEGACY_AVX_IR_C_FILE $ENV{IMPALA_HOME}/be/generated-sources/impala-ir/impala-ir-legacy-avx.cc)
```
这段代码位于 Apache Impala 的 `be/src/codegen/CMakeLists.txt` 文件中，作用是**定义四个预生成的 C 源文件路径**，这些文件的内容实际上是 **LLVM IR（中间表示）数据被转换为 C 语言数组** 的形式，最终会被静态编译进 CodeGen 库，供运行时 JIT 使用。

```cmake
# Generated C files for IR
set(IR_O1_C_FILE $ENV{IMPALA_HOME}/be/generated-sources/impala-ir/impala-ir-o1.cc)
set(IR_O2_C_FILE $ENV{IMPALA_HOME}/be/generated-sources/impala-ir/impala-ir-o2.cc)
set(IR_Os_C_FILE $ENV{IMPALA_HOME}/be/generated-sources/impala-ir/impala-ir-os.cc)
set(LEGACY_AVX_IR_C_FILE $ENV{IMPALA_HOME}/be/generated-sources/impala-ir/impala-ir-legacy-avx.cc)
```

### 详细解释

#### 1. 路径结构说明
- `$ENV{IMPALA_HOME}`：环境变量，指向 Impala 项目源码根目录（通常是你克隆仓库的路径）。
- 最终路径是：
  ```
  <Impala 项目根目录>/be/generated-sources/impala-ir/impala-ir-o1.cc
  ```
  （以及 o2、os、legacy-avx 三个同级文件）

这些文件**不是手写的**，而是**构建过程中自动生成的**（通过前面提到的 `COMPILE_TO_IR_C_ARRAY` 函数和 `file2array.sh` 脚本）。

#### 2. 四个文件的含义（对应不同优化级别和平台）

Impala 的运行时代码生成支持**多版本预优化 IR**，运行时根据启动参数或硬件能力选择加载最合适的版本：

| 文件名                        | 对应优化级别 / 特性               | 含义 |
|-------------------------------|-----------------------------------|------|
| `impala-ir-o1.cc`             | `-O1`                             | 轻量优化（适合快速启动、调试） |
| `impala-ir-o2.cc`             | `-O2`（默认推荐）                 | 平衡优化（性能与编译时间最佳折中） |
| `impala-ir-os.cc`             | `-Os`（Optimize for size）        | 最小化代码体积（适合内存敏感场景） |
| `impala-ir-legacy-avx.cc`     | `-O1` + 仅支持 AVX（不含 AVX2）   | 兼容老 CPU（不支持 AVX2 的机器，如某些老 Xeon） |

运行时可以通过启动参数控制加载哪个版本，例如：
```
impalad --codegen_level=2   # 使用 O2 版本
```

#### 3. 文件内容是什么样的？

这些 `.cc` 文件的内容**不是普通 C++ 代码**，而是大致长这样（简化示例）：

```cpp
// impala-ir-o2.cc（自动生成）
const unsigned char impala_llvm_o2_ir[] = {
  0x42, 0x43, 0xc0, 0xde,  // LLVM bitcode 的二进制数据开头
  // ... 成千上万字节的 bitcode 数据 ...
  0x00, 0x00, 0x00, 0x00
};
const unsigned int impala_llvm_o2_ir_len = 1234567;  // 数据长度
```

- 这些数组就是 **LLVM bitcode（.bc 文件）** 的完整二进制内容。
- 通过 `file2array.sh` 脚本把 `.bc` 文件转成 C 数组。
- 编译后，这些数组成为静态常量，嵌入到 `libCodeGen.a` 中。

#### 4. 为什么要把 IR 转成 C 数组嵌入二进制？

直接嵌入 `.bc` 文件运行时读取也可以，但 Impala 选择嵌入到代码中的原因：

- **自包含**：impalad 是一个单一可执行文件，无需额外数据文件。
- **启动更快**：无需运行时打开文件读取 IR，直接从内存加载。
- **部署简单**：生产环境拷贝一个 impalad 就够了。

#### 5. 生成流程回顾

这些文件是构建时自动生成的：
1. 用 Clang 把 `impala-ir.cc` 编译成不同优化级别的 `.bc` bitcode。
2. 用 `file2array.sh` 把 `.bc` 转成 C 数组 → 生成这四个 `.cc` 文件。
3. 这四个 `.cc` 文件被编译进 `CodeGen` 库。
4. 运行时根据参数选择加载对应数组 → 交给 LLVM JIT。

### 总结

这段代码的核心目的：

> **定义四个预生成的 C 源文件路径**，这些文件包含不同优化级别（O1、O2、Os）和遗留平台支持的 **LLVM IR bitcode 数据（以 C 数组形式嵌入）**。

它们是 Impala 运行时代码生成系统的“**预编译内核模板**”：
- 构建时生成并嵌入二进制。
- 运行时动态选择最优版本加载。
- 实现“**多版本预优化 + 运行时 JIT 定制**”的高性能策略。

这四个文件是 Impala 查询执行极致性能的重要基石之一。


**********************************************************************
请解释：

```cmake

add_library(CodeGen
  codegen-anyval.cc
  codegen-anyval-read-write-info.cc
  codegen-callgraph.cc
  codegen-symbol-emitter.cc
  codegen-util.cc
  llvm-codegen.cc
  llvm-codegen-cache.cc
  llvm-codegen-object-cache.cc
  instruction-counter.cc
  ${THIRDPARTY_LLVM_SRC_DIR}/SectionMemoryManager.cpp
  ${IR_O1_C_FILE}
  ${IR_O2_C_FILE}
  ${IR_Os_C_FILE}
  ${LEGACY_AVX_IR_C_FILE}
)
```
这段代码位于 Apache Impala 的 `be/src/codegen/CMakeLists.txt` 文件中，是**创建 CodeGen 库**的核心语句。CodeGen 是 Impala 运行时代码生成（runtime code generation）的核心模块，负责将查询中的热点表达式动态编译成高效的机器码。

```cmake
add_library(CodeGen
  codegen-anyval.cc
  codegen-anyval-read-write-info.cc
  codegen-callgraph.cc
  codegen-symbol-emitter.cc
  codegen-util.cc
  llvm-codegen.cc
  llvm-codegen-cache.cc
  llvm-codegen-object-cache.cc
  instruction-counter.cc
  ${THIRDPARTY_LLVM_SRC_DIR}/SectionMemoryManager.cpp
  ${IR_O1_C_FILE}
  ${IR_O2_C_FILE}
  ${IR_Os_C_FILE}
  ${LEGACY_AVX_IR_C_FILE}
)
```

### 详细解释每个源文件的作用

#### 1. 核心实现文件（手写 C++ 源码）

| 文件名                              | 主要职责                                                                 |
|-------------------------------------|--------------------------------------------------------------------------|
| `codegen-anyval.cc`                 | AnyVal（通用值类型）的代码生成支持（Impala 内部统一值表示）             |
| `codegen-anyval-read-write-info.cc` | AnyVal 的读写信息生成（优化内存访问）                                    |
| `codegen-callgraph.cc`              | 构建和分析代码生成调用图（决定内联、优化顺序）                           |
| `codegen-symbol-emitter.cc`         | 生成符号（全局变量、函数名等）在 JIT 代码中的引用                        |
| `codegen-util.cc`                   | 代码生成通用工具函数                                                     |
| `llvm-codegen.cc`                   | **核心**：主 LLVM 代码生成器，负责把 Impala IR 转成 LLVM IR 并优化       |
| `llvm-codegen-cache.cc`             | 代码生成缓存（避免重复生成相同表达式）                                   |
| `llvm-codegen-object-cache.cc`      | JIT 对象缓存（缓存已编译的机器码）                                       |
| `instruction-counter.cc`            | 指令计数器（统计生成的机器码指令数，用于调试和优化决策）                 |

#### 2. 来自第三方 LLVM 的源码
```cmake
${THIRDPARTY_LLVM_SRC_DIR}/SectionMemoryManager.cpp
```

- 路径指向 `be/src/thirdparty/llvm/SectionMemoryManager.cpp`
- 这是 Impala **从 LLVM 官方源码复制并可能稍作修改**的一个文件。
- 作用：自定义的 JIT 内存管理器，负责为生成的机器码分配可执行内存段（RX 权限）。
- 为什么不直接用 toolchain 中的 LLVM 库？
  - 需要特定行为或补丁。
  - 确保与 Impala 的 JIT 需求完全匹配。

#### 3. 预生成的 IR C 文件（关键！）
```cmake
${IR_O1_C_FILE}
${IR_O2_C_FILE}
${IR_Os_C_FILE}
${LEGACY_AVX_IR_C_FILE}
```

- 这四个变量指向**构建时自动生成的 .cc 文件**（路径在 `be/generated-sources/impala-ir/`）。
- 每个文件的内容是一个**巨大的 C 数组**，数组数据就是对应优化级别的 LLVM bitcode（.bc 文件）的二进制内容。
- 例如 `impala-ir-o2.cc` 大致长这样：
  ```cpp
  const unsigned char impala_llvm_o2_ir[] = { /* 成千上万字节的 bitcode 数据 */ };
  const unsigned int impala_llvm_o2_ir_len = 1234567;
  ```
- 这些数组在运行时被加载到内存，交给 LLVM ORC JIT 引擎进一步优化和生成机器码。

**为什么有四个版本？**
- `-O1`：轻量优化，启动快
- `-O2`：平衡优化（默认推荐）
- `-Os`：最小化代码大小（内存敏感场景）
- `legacy-avx`：仅支持 AVX（不含 AVX2），兼容老 CPU

运行时通过启动参数（如 `--codegen_level=2`）选择加载哪个版本。

### 最终生成的库

`add_library(CodeGen ...)` 会生成：
- 静态构建：`libCodeGen.a`
- 动态构建：`libCodeGen.so`

这个库会被链接到 impalad 和测试程序中，是查询执行性能的核心引擎。

### 总结

这段代码的核心目的：

> **创建 Impala 的运行时代码生成核心库 `CodeGen`**，包含：
> - 所有手写的代码生成逻辑（LLVM 集成、缓存、调用图等）。
> - Impala 定制的 LLVM 内存管理器。
> - **四个预生成的 LLVM IR bitcode 数据（以 C 数组形式嵌入）**，对应不同优化级别和平台支持。

这些嵌入的 IR 是 Impala 查询“**预编译内核**”：
- 构建时生成并静态嵌入。
- 运行时根据参数加载最优版本，再进行最终 JIT 定制。
- 实现极致查询性能的关键技术。

没有这个库，Impala 的向量化执行和表达式计算就无法达到当前的高性能水平。

***************************************************************
请解释：add_dependencies(CodeGen gen-deps)
### `add_dependencies(CodeGen gen-deps)` 解释

这段代码是 CMake 的一个标准命令，用于在构建目标 `CodeGen`（Impala 的运行时代码生成库）时，**显式声明它依赖于另一个自定义目标 `gen-deps`**。这确保了在编译 `CodeGen` 之前，`gen-deps` 必须先完成构建，从而避免环境或依赖准备不全导致的编译失败。

#### 命令语法
- `add_dependencies(target dependency)`：让 `target`（这里是 `CodeGen`）依赖 `dependency`（这里是 `gen-deps`）。
- 如果 `gen-deps` 尚未构建，CMake 会自动先执行它。

#### `gen-deps` 目标是什么？
从 Apache Impala 的源代码（`CMakeLists.txt`）中可以看出，`gen-deps` 是一个**顶层自定义目标**（custom target），定义在项目根目录的 `CMakeLists.txt` 中：

```cmake
add_custom_target(gen-deps ALL DEPENDS thrift-deps proto-deps fb-deps
  kudu-util-proto-deps kudu-rpc-proto-deps kudu-security-proto-deps gen_ir_descriptions)
```

- **作用**：这是一个“生成依赖”的聚合目标，它确保**所有预生成的文件和代码**在后端构建开始前准备好。
- 它依赖以下子目标：
  - `thrift-deps`：Thrift 接口代码生成（从 .thrift 文件生成 C++/Java 代码）。
  - `proto-deps`：Protobuf 代码生成（从 .proto 文件生成 C++ 存根）。
  - `fb-deps`：FlatBuffers 代码生成（用于高效序列化）。
  - `kudu-*`：Kudu 集成相关的 Protobuf 生成。
  - `gen_ir_descriptions`：IR 元数据生成（用于 codegen 反射）。

这些都是**代码生成步骤**（code generation），不产生二进制，但必须先运行，因为后续模块（如 CodeGen）会包含生成的头文件和源文件。

- `ALL` 关键字：`gen-deps` 本身也会加入默认构建（`make` 时自动运行），确保整个项目从零开始构建时一切就绪。

#### 为什么 CodeGen 需要依赖 gen-deps？
CodeGen 模块高度依赖这些生成的文件：
- Thrift 生成的 RPC 接口（`ImpalaInternalService.h` 等）。
- Protobuf 生成的协议存根。
- FlatBuffers 生成的序列化代码。
- IR 描述元数据（函数名、签名）。

如果不加这个依赖：
- 构建时可能顺序混乱：CodeGen 先编译，但生成文件还没准备好 → 头文件未找到、链接失败。
- 加了依赖后：CMake 保证 `gen-deps` 先运行 → 生成所有必要文件 → CodeGen 安全编译。

#### 实际构建流程
运行 `make` 或 `./buildall.sh` 时：
1. CMake 解析依赖图，发现 `CodeGen` 依赖 `gen-deps`。
2. 先执行 `gen-deps`（运行 Thrift/Protobuf/FlatBuffers 生成器）。
3. 生成的文件（如 `gen-cpp/ImpalaInternalService.h`）就位。
4. 然后编译 `CodeGen` 的源码（`llvm-codegen.cc` 等），成功包含生成文件。

#### 总结
`add_dependencies(CodeGen gen-deps)` 的核心目的：

> **强制 CodeGen 库的构建等待所有代码生成依赖（Thrift、Protobuf、FlatBuffers 等）完成**，  
> 确保运行时代码生成模块能正确使用预生成的接口和元数据文件。

这是 Impala 构建系统中典型的**防御性依赖管理**，防止模块间顺序错误导致的构建失败，尤其在有大量自动生成代码的项目中非常必要。

**************************************************************************

我没有看懂请解释：
```cmake
# output cross compile to ir metadata
set(IR_DESC_GEN_OUTPUT
  $ENV{IMPALA_HOME}/be/generated-sources/impala-ir/impala-ir-names.h
  $ENV{IMPALA_HOME}/be/generated-sources/impala-ir/impala-ir-functions.h
)
```各个参数的含义，这个命令执行完成后结果是什么
这段 CMake 代码的作用是**定义一个变量 `IR_DESC_GEN_OUTPUT`**，该变量的值是一个**包含两个头文件路径的列表**。这两个头文件是构建过程中**自动生成的**，用于存放 Impala 运行时代码生成（runtime codegen）的 LLVM IR 元数据。

```cmake
# output cross compile to ir metadata
set(IR_DESC_GEN_OUTPUT
  $ENV{IMPALA_HOME}/be/generated-sources/impala-ir/impala-ir-names.h
  $ENV{IMPALA_HOME}/be/generated-sources/impala-ir/impala-ir-functions.h
)
```

### 逐部分解释

1. **`set(IR_DESC_GEN_OUTPUT ...)`**  
   - `set` 是 CMake 的基本命令，用于定义一个变量。
   - 变量名：`IR_DESC_GEN_OUTPUT`
   - 变量类型：**列表（list）**，包含两个路径字符串（CMake 会自动把多行内容当作列表处理）。

2. **`$ENV{IMPALA_HOME}`**  
   - `$ENV{...}` 是 CMake 读取**环境变量**的语法。
   - `IMPALA_HOME` 是 Impala 项目根目录的路径（通常由 `./buildall.sh` 或 `impala-config.sh` 设置）。
   - 例如，如果你的 Impala 源码在 `/home/user/impala`，那么 `$ENV{IMPALA_HOME}` 就是 `/home/user/impala`。

3. **完整路径展开示例**  
   假设 `IMPALA_HOME=/home/user/impala`，那么变量内容实际是：

   ```
   /home/user/impala/be/generated-sources/impala-ir/impala-ir-names.h
   /home/user/impala/be/generated-sources/impala-ir/impala-ir-functions.h
   ```

   这两个文件位于源码树的 `be/generated-sources/impala-ir/` 目录下。

### 这两个头文件的内容和用途

这两个文件**不是手动写的**，而是**构建时由脚本自动生成**的（后面会调用 `gen_ir_descriptions.py`）：

| 文件名                          | 生成后大致内容（简化示例）                                      | 主要用途 |
|---------------------------------|------------------------------------------------------------------|----------|
| `impala-ir-names.h`             | ```cpp
| `impala-ir-functions.h`         | ```cpp<br>struct IrFunctionDesc {<br>  const char* name;<br>  int num_params;<br>  // 可能还有签名、ID 等<br>};<br>extern IrFunctionDesc kIrFunctions[];<br>...``` | 保存函数的**详细元数据**（如参数个数、类型、ID 映射），用于运行时反射、参数检查等 |

### 执行这行命令后的结果

执行 `set(...)` 这一行后：
- CMake 内存中多了一个变量 `IR_DESC_GEN_OUTPUT`。
- 这个变量的值是一个包含**两个文件路径**的列表。
- **文件本身还不存在**（此时只是路径）。
- 后续 CMake 代码会用这个变量作为 `add_custom_command` 的 `OUTPUT`，告诉 CMake：
  - 这两个文件是由 `gen_ir_descriptions.py` 脚本生成的。
  - 如果源码（`impala-ir.cc`）改动了，就重新运行脚本生成最新版本。

### 后续实际生成流程（简要）

稍后的代码：
```cmake
add_custom_command(
  OUTPUT ${IR_DESC_GEN_OUTPUT}   # ← 这里就用到了这个变量
  COMMAND ./gen_ir_descriptions.py
  ...
)
add_custom_target(gen_ir_descriptions ALL DEPENDS ${IR_DESC_GEN_OUTPUT})
```

构建时（`make`）会：
1. 运行 `gen_ir_descriptions.py` 脚本。
2. 脚本解析 `impala-ir.cc` 中的函数定义。
3. 生成上述两个头文件到指定路径。
4. 这些头文件被 `CodeGen` 库或其他模块 `#include`，用于运行时打印函数名等。

### 总结

这行代码本身只做了**一件事**：

> 定义了一个 CMake 列表变量 `IR_DESC_GEN_OUTPUT`，内容是两个将来要生成的头文件的完整路径。

它的意义在于：
- 为后续的自动生成规则提供输出文件列表。
- 确保构建时生成最新的 IR 元数据头文件。
- 让 Impala 的代码生成函数在运行时有可读的名称和信息（极大提升调试体验）。

执行完这行后，**没有文件被创建**，只是准备好了路径；真正的文件生成发生在后面的 `add_custom_command` 执行时（即实际构建时）。
*********************************************************************
请解释：

```cmake

# output cross compile to ir metadata
set(IR_DESC_GEN_OUTPUT
  $ENV{IMPALA_HOME}/be/generated-sources/impala-ir/impala-ir-names.h
  $ENV{IMPALA_HOME}/be/generated-sources/impala-ir/impala-ir-functions.h
)
```
这段代码位于 Apache Impala 的 `be/src/codegen/CMakeLists.txt` 文件中，作用是**定义两个自动生成的头文件路径**，这两个头文件包含运行时代码生成（runtime codegen）所需的 **LLVM IR 元数据**（metadata），供调试、反射和性能分析使用。

```cmake
# output cross compile to ir metadata
set(IR_DESC_GEN_OUTPUT
  $ENV{IMPALA_HOME}/be/generated-sources/impala-ir/impala-ir-names.h
  $ENV{IMPALA_HOME}/be/generated-sources/impala-ir/impala-ir-functions.h
)
```

### 详细解释

#### 1. 两个头文件的路径和含义
- `$ENV{IMPALA_HOME}`：环境变量，指向 Impala 项目源码根目录。
- 最终路径：
  ```
  <Impala 项目根目录>/be/generated-sources/impala-ir/impala-ir-names.h
  <Impala 项目根目录>/be/generated-sources/impala-ir/impala-ir-functions.h
  ```

这两个文件**不是手写的**，而是**构建时由 Python 脚本自动生成**的（后面会看到 `gen_ir_descriptions.py`）。

#### 2. 两个头文件的内容和作用

| 文件名                          | 主要内容                                                                 | 作用 |
|---------------------------------|--------------------------------------------------------------------------|------|
| `impala-ir-names.h`             | 所有预生成 IR 函数/全局变量的**字符串名称常量**（如 `"scalar_fn_add"`） | 运行时打印生成的函数名、调试日志、性能分析工具显示可读名称 |
| `impala-ir-functions.h`         | IR 函数的**签名、参数类型、返回值类型**等元数据（可能包括函数指针或 ID 映射） | 运行时反射：根据字符串名找到对应 IR 函数、参数检查、类型安全 |

这些元数据让 Impala 在使用预生成的 LLVM IR 模板时，仍能保持**可读性和调试友好性**：
- 例如，当一个查询使用了代码生成的加法函数，日志中可以打印 `"Executing codegen function: scalar_fn_add"` 而不是一串无意义的地址。
- 性能分析工具（如 `--display_codegen`）可以显示人类可读的函数名。

#### 3. 生成流程（后续代码会用到这个变量）

稍后在同一个 CMakeLists.txt 中，有：
```cmake
add_custom_command(
  OUTPUT ${IR_DESC_GEN_OUTPUT}
  COMMAND ./gen_ir_descriptions.py
  DEPENDS ./gen_ir_descriptions.py
  COMMENT "Generating ir cross compile metadata."
  VERBATIM
)
add_custom_target(gen_ir_descriptions ALL DEPENDS ${IR_DESC_GEN_OUTPUT})
```

- `IR_DESC_GEN_OUTPUT` 作为 `add_custom_command` 的 `OUTPUT`，告诉 CMake 这两个头文件是由 `gen_ir_descriptions.py` 脚本生成的。
- 脚本会解析 `impala-ir.cc`（核心 IR 源码）中的函数定义，提取名称和签名，生成这两个头文件。
- `ALL` 确保每次构建都重新生成最新元数据（源码改了，名称可能变）。

#### 4. 为什么需要这些元数据？

Impala 的运行时代码生成采用**预生成 IR 模板 + 运行时 JIT 定制**的策略：
- 核心模板（如标量函数、哈希函数）在构建时预编译成 bitcode 并嵌入二进制。
- 运行时加载这些 bitcode 进行最终优化和链接。
- 如果没有元数据，生成的函数只是匿名 LLVM 函数，调试时只能看到 `??` 或地址。
- 有了 `impala-ir-names.h` 和 `impala-ir-functions.h`，Impala 可以在运行时：
  - 打印清晰的函数名
  - 实现函数注册表
  - 支持 `--display_codegen` 参数输出可读的 IR 信息

### 总结

这段代码的核心目的：

> **定义两个自动生成的头文件路径**，它们包含预生成 LLVM IR 函数的**名称和签名元数据**：
> - `impala-ir-names.h`：函数名字符串常量
> - `impala-ir-functions.h`：函数签名等详细信息

这些元数据是 Impala 运行时代码生成系统**调试友好**和**可维护**的关键：
- 让生成的机器码函数有“人类可读的名字”。
- 支持日志、性能分析、调试输出显示清晰信息。
- 构建时由 `gen_ir_descriptions.py` 自动生成，确保与源码同步。

没有它们，Impala 的 codegen 虽然性能依旧极高，但调试起来会像“黑盒”一样困难。
************************************************************************

请详细解释下：

```cmake
add_custom_command(
  OUTPUT ${IR_DESC_GEN_OUTPUT}
  COMMAND ./gen_ir_descriptions.py
  DEPENDS ./gen_ir_descriptions.py
  COMMENT "Generating ir cross compile metadata."
  VERBATIM
)
```
这段 CMake 代码的核心作用是：**告诉 CMake 如何生成两个重要的头文件**（`impala-ir-names.h` 和 `impala-ir-functions.h`），这两个头文件包含 Impala 预生成的 LLVM IR 函数的**元数据**（名称、签名等），用于运行时调试、日志打印和反射。

```cmake
add_custom_command(
  OUTPUT ${IR_DESC_GEN_OUTPUT}
  COMMAND ./gen_ir_descriptions.py
  DEPENDS ./gen_ir_descriptions.py
  COMMENT "Generating ir cross compile metadata."
  VERBATIM
)
```

下面逐个参数详细解释：

### 1. `OUTPUT ${IR_DESC_GEN_OUTPUT}`
- **含义**：指定这条自定义命令的**输出文件**。
- `${IR_DESC_GEN_OUTPUT}` 是前面定义的变量，内容是两个头文件的完整路径：
  ```
  /path/to/impala/be/generated-sources/impala-ir/impala-ir-names.h
  /path/to/impala/be/generated-sources/impala-ir/impala-ir-functions.h
  ```
- **作用**：
  - CMake 会监控这两个文件是否存在或是否过时。
  - 如果这两个文件**不存在**，或者它们的依赖有变化，CMake 就会重新执行这条命令来生成它们。

### 2. `COMMAND ./gen_ir_descriptions.py`
- **含义**：当需要生成输出文件时，CMake 要执行的具体命令。
- **执行的内容**：
  - 在当前目录（`be/src/codegen/`）下运行 Python 脚本 `gen_ir_descriptions.py`。
  - 这个脚本会：
    - 解析核心 IR 源码文件 `impala-ir.cc`
    - 提取所有函数的名称、参数个数、返回值类型等信息
    - 生成上述两个头文件（`impala-ir-names.h` 和 `impala-ir-functions.h`）

### 3. `DEPENDS ./gen_ir_descriptions.py`
- **含义**：声明这条命令依赖于哪些文件。
- **具体依赖**：
  - 脚本本身 `gen_ir_descriptions.py`
- **作用**：
  - 如果这个 Python 脚本文件被修改了（比如修复 bug 或添加新功能），CMake 会认为输出文件可能过时，从而重新运行命令生成最新的头文件。
  - （注意：实际依赖还应该包括 `impala-ir.cc`，但这里没有显式列出，可能在其他地方处理或依赖传递）

### 4. `COMMENT "Generating ir cross compile metadata."`
- **含义**：在构建时显示的进度提示信息。
- **效果**：
  - 当 `make` 执行到这条命令时，终端会显示：
    ```
    [ 12%] Generating ir cross compile metadata.
    ```
  - 让开发者知道当前正在生成 IR 元数据，而不是卡住。

### 5. `VERBATIM`
- **含义**：告诉 CMake **严格按照原样传递命令**，不要进行额外的 shell 转义或变量替换。
- **为什么需要**：
  - 确保 `./gen_ir_descriptions.py` 这个命令被完整、正确地传递给 shell 执行。
  - 避免 CMake 对路径或参数进行不必要的处理（尤其在 Windows 或复杂路径下）。

### 执行后的实际结果

当你运行 `make` 时，如果这两个头文件不存在或过时，CMake 会：

1. 切换到 `be/src/codegen/` 目录
2. 执行命令：`./gen_ir_descriptions.py`
3. 脚本运行后，在以下路径生成两个头文件：
   ```
   be/generated-sources/impala-ir/impala-ir-names.h
   be/generated-sources/impala-ir/impala-ir-functions.h
   ```
4. 生成的内容示例（简化）：
   ```cpp
   // impala-ir-names.h
   constexpr const char* kScalarAddName = "scalar_add";
   constexpr const char* kHash32Name = "hash_32";
   // ...
   
   // impala-ir-functions.h
   struct IrFunctionInfo {
     const char* name;
     int num_args;
     // 其他元数据...
   };
   extern const IrFunctionInfo kIrFunctions[];
   ```

这些头文件会被其他代码 `#include`，用于：
- 打印人类可读的函数名（日志、调试）
- 运行时根据字符串查找对应 IR 函数
- 性能分析工具显示函数名

### 总结：这条命令的整体意义

> **定义一个自动生成规则**：  
> 当 `impala-ir-names.h` 和 `impala-ir-functions.h` 不存在或过时，就运行 `gen_ir_descriptions.py` 脚本来生成它们。

它是 Impala 运行时代码生成系统**调试友好性**的关键一环：
- 让预生成的 LLVM IR 函数拥有“名字”和“元数据”
- 避免生成的机器码函数在日志中显示为 `??` 或地址
- 支持 `--display_codegen` 等调试功能显示清晰信息

没有这条规则，IR 元数据就无法自动生成，调试 codegen 相关问题会非常困难。
*************************************************************************
请解释：

```cmake

add_custom_target(gen_ir_descriptions ALL DEPENDS ${IR_DESC_GEN_OUTPUT})
```
这段代码的作用是：**创建一个名为 `gen_ir_descriptions` 的自定义构建目标（custom target）**，并把它加入到**默认构建流程**中，确保每次执行 `make`（不带参数）时，都会自动检查并（如果需要）重新生成 Impala 运行时代码生成的两个 IR 元数据头文件。

```cmake
add_custom_target(gen_ir_descriptions ALL DEPENDS ${IR_DESC_GEN_OUTPUT})
```

### 逐参数详细解释

1. **`add_custom_target(gen_ir_descriptions ...)`**
   - 创建一个**自定义目标**，名字叫 `gen_ir_descriptions`。
   - 自定义目标本身不直接生成可执行文件或库，而是用来**组织一系列命令**（这里是通过前面的 `add_custom_command` 定义的）。

2. **`ALL` 关键字（最关键！）**
   - 把这个目标加入到 CMake 的**默认构建目标（default build target）**中。
   - 效果：
     - 运行 `make`（不指定任何目标）时，会**自动执行**这个目标。
     - 如果不加 `ALL`，这个目标就变成“可选的”，必须手动输入 `make gen_ir_descriptions` 才会运行。

   **为什么一定要加 `ALL`？**  
   这两个头文件（`impala-ir-names.h` 和 `impala-ir-functions.h`）是 CodeGen 模块正常工作的**必需依赖**：
   - 它们提供预生成 IR 函数的可读名称和签名。
   - 如果不生成它们，后续编译会失败（头文件找不到）或运行时调试信息缺失。
   - 加 `ALL` 确保开发者**不管用什么方式构建**（直接 `make`、`./buildall.sh` 等），这些关键文件都会自动、及时生成。

3. **`DEPENDS ${IR_DESC_GEN_OUTPUT}`**
   - 声明这个自定义目标**依赖于前面 `add_custom_command` 生成的两个输出文件**：
     ```
     impala-ir-names.h
     impala-ir-functions.h
     ```
   - 实际效果：
     - CMake 会自动把前面的 `add_custom_command`（运行 `gen_ir_descriptions.py` 脚本）关联到这个目标。
     - 当这两个头文件不存在、过时，或依赖的脚本/源码变化时，执行 `make gen_ir_descriptions`（或普通 `make`）就会重新运行脚本生成最新版本。

### 完整执行流程总结

结合前面的 `add_custom_command`，整个过程是：

1. `add_custom_command` 定义了**如何生成**两个头文件（运行 Python 脚本）。
2. `add_custom_target(... ALL DEPENDS ...)` 创建了一个**构建目标**来包装这个生成过程。
3. 加了 `ALL` → 这个目标成为默认构建的一部分。
4. 结果：
   - 每次运行 `make` 或 `./buildall.sh` 时：
     - CMake 检查两个头文件是否需要更新。
     - 如果需要 → 自动运行 `gen_ir_descriptions.py` 重新生成。
     - 生成后继续后续编译。

### 实际使用示例

```bash
# 普通构建（最常见）
./buildall.sh          # 或 make
→ 自动运行 gen_ir_descriptions（因为 ALL），生成最新头文件

# 只想强制重新生成 IR 元数据
make gen_ir_descriptions
→ 强制运行脚本，更新 impala-ir-names.h 和 impala-ir-functions.h
```

### 总结：这行代码的核心意义

> **创建一个必须在每次默认构建中自动执行的目标**，  
> 负责生成运行时代码生成所需的两个关键元数据头文件（函数名 + 签名信息），  
> 确保：
> - 开发者无需手动干预，这些文件始终是最新的。
> - 构建失败风险最小化（避免“头文件找不到”）。
> - 运行时调试信息完整（函数有可读名称）。

没有这行（特别是没有 `ALL`），开发者很容易忘记生成这些头文件，导致构建诡异失败或调试困难。  
加了 `ALL`，整个过程对开发者完全透明——这就是优秀构建系统的体现。

****************************************************************************
请解释：

```cmake


set(IR_INPUT_FILES impala-ir.cc)
```
这段代码位于 Apache Impala 的 `be/src/codegen/CMakeLists.txt` 文件中，作用是**定义一个 CMake 变量 `IR_INPUT_FILES`**，它的值是一个源文件列表（这里只有一个文件：`impala-ir.cc`）。

```cmake
set(IR_INPUT_FILES impala-ir.cc)
```

### 详细解释

#### 1. 变量含义
- `set(IR_INPUT_FILES impala-ir.cc)`  
  把变量 `IR_INPUT_FILES` 设置为一个**包含单个文件路径的列表**：`impala-ir.cc`。
- 这个文件位于当前目录（`be/src/codegen/`），完整路径是 `be/src/codegen/impala-ir.cc`。

#### 2. `impala-ir.cc` 是什么文件？为什么这么重要？

`impala-ir.cc` 是 Impala 运行时代码生成（runtime codegen）的**核心模板源码**，可以说是 Impala 查询性能的“秘密武器”之一。

它是一个**手写的 C++ 文件**，但内容非常特殊：
- 里面定义了一系列**关键的热点函数**，如：
  - 标量表达式计算（加减乘除、字符串操作等）
  - 哈希函数（Hash Join、Aggregation 用）
  - 聚合函数中间状态处理
  - 字符串处理、日期处理等
- 这些函数被设计成**高度可代码生成的模板**：使用大量宏、模板和内联汇编，便于 LLVM 优化和向量化。

**关键点**：这个文件**不会直接编译进最终二进制**，而是作为**输入**，在构建时被多次交叉编译成不同优化级别的 LLVM IR（中间表示），然后嵌入到 `CodeGen` 库中。

#### 3. 这个变量在后续的使用

稍后在同一个 CMakeLists.txt 中，有一个函数 `COMPILE_TO_IR_C_ARRAY`，它会多次调用来生成不同版本的 IR：

```cmake
COMPILE_TO_IR_C_ARRAY(${IR_O1_C_FILE} impala_llvm_o1_ir -O1 ${PLATFORM_SPECIFIC_FLAGS})
COMPILE_TO_IR_C_ARRAY(${IR_O2_C_FILE} impala_llvm_o2_ir -O2 ${PLATFORM_SPECIFIC_FLAGS})
COMPILE_TO_IR_C_ARRAY(${IR_Os_C_FILE} impala_llvm_os_ir -Os ${PLATFORM_SPECIFIC_FLAGS})
COMPILE_TO_IR_C_ARRAY(${LEGACY_AVX_IR_C_FILE} impala_legacy_avx_llvm_ir -O1 ${LEGACY_AVX_SPECIFIC_FLAGS})
```

在这个函数内部，会使用 `${IR_INPUT_FILES}` 作为编译输入：
```cmake
COMMAND ${LLVM_CLANG_EXECUTABLE} ... ${IR_INPUT_FILES} -o ...
```

意思是：用 Clang 把 `impala-ir.cc` 编译成 bitcode（.bc 文件），再进一步处理成 C 数组嵌入。

#### 4. 为什么用变量而不是直接写文件名？

- **可维护性**：如果将来需要添加更多输入文件（例如拆分成多个 .cc），只需修改这个变量。
- **清晰性**：明确表达“这些是 IR 生成的输入文件”。
- **一致性**：所有调用都使用同一个变量，避免硬编码出错。

### 执行这行命令后的结果

执行 `set(IR_INPUT_FILES impala-ir.cc)` 后：
- CMake 内存中多了一个变量 `IR_INPUT_FILES`。
- 值是一个列表：`impala-ir.cc`（当前目录下的文件）。
- **文件本身不受影响**（它早就存在，是手写源码）。
- 后续所有用到 `${IR_INPUT_FILES}` 的地方，都会替换成 `impala-ir.cc`。

### 总结

这行代码的核心目的：

> **定义一个变量 `IR_INPUT_FILES`，明确指定运行时代码生成的“核心模板源码”是 `impala-ir.cc`**。

它是整个预生成 IR 流程的起点：
- `impala-ir.cc` →（多次编译）→ 不同优化级别的 bitcode → 转成 C 数组 → 嵌入二进制
- 运行时根据参数加载最优版本 → JIT 定制 → 生成最终机器码

没有这行（或写错文件名），所有预生成 IR 都无法正确编译，Impala 的高性能代码生成就瘫痪了。  
这行看似简单，却是 Impala 极致性能的“源头”。
*******************************************************8
请解释：

```cmake

function(COMPILE_TO_IR_C_ARRAY IR_C_FILE VARNAME)
  # Run the clang compiler to generate IR. Then run the LLVM opt tool to apply specific
  # optimisations. We need to compile to IR several times for different optimization settings
  # and legacy AVX support. Additional args (ARGN) are passed to clang as compiler flags.
  get_filename_component(BASE_NAME ${IR_C_FILE} NAME_WE)
  set(IR_OUTPUT_FILE "${LLVM_IR_OUTPUT_DIRECTORY}/${BASE_NAME}.bc")
  set(IR_TMP_OUTPUT_FILE "${LLVM_IR_OUTPUT_DIRECTORY}/${BASE_NAME}-tmp.bc")
  add_custom_command(
    OUTPUT ${IR_OUTPUT_FILE}
    COMMAND ${LLVM_CLANG_EXECUTABLE} ${CLANG_IR_CXX_FLAGS} ${ARGN}
            ${CLANG_INCLUDE_FLAGS} ${IR_INPUT_FILES} -o ${IR_TMP_OUTPUT_FILE}
    COMMAND ${LLVM_OPT_EXECUTABLE} ${LLVM_OPT_IR_FLAGS} < ${IR_TMP_OUTPUT_FILE} > ${IR_OUTPUT_FILE}
    COMMAND rm ${IR_TMP_OUTPUT_FILE}
    DEPENDS ExecIr ExecAvroIr ExprsGeoIr ExecKuduIr ExprsIr RuntimeIr UdfIr UtilIr ${IR_INPUT_FILES}
  )

  # Convert LLVM bytecode to C array.
  set(IR_TMP_C_FILE ${IR_C_FILE}.tmp)
  add_custom_command(
    OUTPUT ${IR_C_FILE}
    COMMAND $ENV{IMPALA_HOME}/bin/file2array.sh -n -v ${VARNAME} ${IR_OUTPUT_FILE} > ${IR_TMP_C_FILE}
    COMMAND mv ${IR_TMP_C_FILE} ${IR_C_FILE}
    DEPENDS $ENV{IMPALA_HOME}/bin/file2array.sh
    DEPENDS ${IR_OUTPUT_FILE}
  )
endfunction(COMPILE_TO_IR_C_ARRAY)
```

这个 CMake 函数 `COMPILE_TO_IR_C_ARRAY` 是 Apache Impala **运行时代码生成（runtime codegen）** 的核心自动化工具。它的作用是：

> **将一份关键的 C++ 源码（`impala-ir.cc`）交叉编译成 LLVM bitcode（.bc 文件），再将 bitcode 转换为 C 语言数组，最终生成一个可以直接编译进 Impala 二进制的 `.cc` 文件**。

这样做的最终目的是：在 impalad 启动时就把大量高性能的预优化 IR 模板**静态嵌入**到程序中，运行时只需要加载这些数组、交给 LLVM JIT 做最后的定制化优化即可，大幅提升查询性能。

下面逐行、逐部分详细解释这个函数。

```cmake
function(COMPILE_TO_IR_C_ARRAY IR_C_FILE VARNAME)
```
### 函数参数
- `IR_C_FILE`：**最终生成的 C 文件路径**（例如 `impala-ir-o2.cc`）。
- `VARNAME`：生成的 C 数组的**变量名**（例如 `impala_llvm_o2_ir`）。
- `${ARGN}`：额外传递给 Clang 的编译选项（例如 `-O2`、`-mavx2` 等），这是 CMake 的特殊变量，表示“剩余的所有参数”。

### 第一步：准备文件名
```cmake
  get_filename_component(BASE_NAME ${IR_C_FILE} NAME_WE)
  set(IR_OUTPUT_FILE "${LLVM_IR_OUTPUT_DIRECTORY}/${BASE_NAME}.bc")
  set(IR_TMP_OUTPUT_FILE "${LLVM_IR_OUTPUT_DIRECTORY}/${BASE_NAME}-tmp.bc")
```
- 从 `IR_C_FILE`（如 `.../impala-ir-o2.cc`）提取**不带扩展名的基础名** → `impala-ir-o2`
- 定义两个 bitcode 文件路径（都在 `llvm-ir/` 目录下）：
  - 临时文件：`impala-ir-o2-tmp.bc`
  - 最终文件：`impala-ir-o2.bc`

### 第二步：生成 LLVM bitcode（.bc 文件）
```cmake
  add_custom_command(
    OUTPUT ${IR_OUTPUT_FILE}
    COMMAND ${LLVM_CLANG_EXECUTABLE} ${CLANG_IR_CXX_FLAGS} ${ARGN}
            ${CLANG_INCLUDE_FLAGS} ${IR_INPUT_FILES} -o ${IR_TMP_OUTPUT_FILE}
    COMMAND ${LLVM_OPT_EXECUTABLE} ${LLVM_OPT_IR_FLAGS} < ${IR_TMP_OUTPUT_FILE} > ${IR_OUTPUT_FILE}
    COMMAND rm ${IR_TMP_OUTPUT_FILE}
    DEPENDS ExecIr ExecAvroIr ExprsGeoIr ExecKuduIr ExprsIr RuntimeIr UdfIr UtilIr ${IR_INPUT_FILES}
  )
```
这是最关键的生成步骤，分三条命令执行：

1. **用 Clang 交叉编译成 bitcode**
   ```
   clang++ [预定义的IR编译选项] [用户传入的优化选项，如 -O2] [所有include路径] impala-ir.cc -o impala-ir-o2-tmp.bc
   ```
   - 生成一个**临时的 bitcode 文件**（带 `-tmp` 后缀）。

2. **用 LLVM opt 工具进一步优化**
   ```
   opt [预定义的优化选项] < impala-ir-o2-tmp.bc > impala-ir-o2.bc
   ```
   - 对临时 bitcode 再做一次优化（例如内联、循环向量化等），输出最终的 `.bc` 文件。

3. **删除临时文件**
   ```
   rm impala-ir-o2-tmp.bc
   ```

**DEPENDS** 部分列出了所有依赖：
- 很多 `ExecIr`、`ExprsIr` 等目标（这些是其他模块的 IR 生成目标，确保它们先完成）
- `${IR_INPUT_FILES}` 即 `impala-ir.cc`（源码改了就重新生成）

### 第三步：将 bitcode 转换为 C 数组
```cmake
  # Convert LLVM bytecode to C array.
  set(IR_TMP_C_FILE ${IR_C_FILE}.tmp)
  add_custom_command(
    OUTPUT ${IR_C_FILE}
    COMMAND $ENV{IMPALA_HOME}/bin/file2array.sh -n -v ${VARNAME} ${IR_OUTPUT_FILE} > ${IR_TMP_C_FILE}
    COMMAND mv ${IR_TMP_C_FILE} ${IR_C_FILE}
    DEPENDS $ENV{IMPALA_HOME}/bin/file2array.sh
    DEPENDS ${IR_OUTPUT_FILE}
  )
```
- 使用 Impala 自带的脚本 `file2array.sh`：
  - `-n`：输出 const unsigned char 数组
  - `-v ${VARNAME}`：指定数组变量名（如 `impala_llvm_o2_ir`）
  - 把 `.bc` 二进制内容转成 C 数组，写入临时文件 `impala-ir-o2.cc.tmp`
- 再把临时文件重命名为最终目标文件 `impala-ir-o2.cc`

最终生成的 `.cc` 文件内容大致是：
```cpp
const unsigned char impala_llvm_o2_ir[] = {
  0x42, 0x43, 0xc0, 0xde, /* ... 成千上万字节的 bitcode 数据 ... */
};
const unsigned int impala_llvm_o2_ir_len = 1234567;
```

### 函数调用示例（实际使用）

在同一个 CMakeLists.txt 中有四次调用：
```cmake
COMPILE_TO_IR_C_ARRAY(${IR_O1_C_FILE} impala_llvm_o1_ir -O1 ${PLATFORM_SPECIFIC_FLAGS})
COMPILE_TO_IR_C_ARRAY(${IR_O2_C_FILE} impala_llvm_o2_ir -O2 ${PLATFORM_SPECIFIC_FLAGS})
COMPILE_TO_IR_C_ARRAY(${IR_Os_C_FILE} impala_llvm_os_ir -Os ${PLATFORM_SPECIFIC_FLAGS})
COMPILE_TO_IR_C_ARRAY(${LEGACY_AVX_IR_C_FILE} impala_legacy_avx_llvm_ir -O1 ${LEGACY_AVX_SPECIFIC_FLAGS})
```

生成四个不同版本的预优化 IR，运行时 Impala 会根据启动参数（如 `--codegen_level=2`）或 CPU 能力选择加载哪个。

### 总结：这个函数的整体意义

`COMPILE_TO_IR_C_ARRAY` 是 Impala **高性能查询执行** 的基石之一，它完成了：

1. 用 Clang + opt 将核心模板代码 `impala-ir.cc` 编译成**高度优化的 LLVM bitcode**（不同优化级别）。
2. 将 bitcode 转换为**C 数组**，生成一个普通的 `.cc` 文件。
3. 这个 `.cc` 文件被直接编译进 `libCodeGen`，实现**IR 静态嵌入**。

优势：
- 启动快（无需运行时加载外部文件）
- 自包含（部署只需一个 impalad 二进制）
- 多版本支持（O1/O2/Os + 老 CPU 兼容）

没有这个函数，Impala 的运行时代码生成就无法实现当前水平的极致性能和调试友好性。


**************************************************************************
```shellscript


这个是apache impala根目录bin/file2array.sh的内容：
shellscript#!/bin/bash

# Copyright 2015 Cloudera Inc.
# Copyright 2013 University of Chicago and Argonne National Laboratory
#
# Licensed under the Apache License, Version 2.0 (the "License");
# you may not use this file except in compliance with the License.
# You may obtain a copy of the License at
#
#     http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License

# Convert file to a C array with provided name.
#
# Also produce length variable of type size_t with _len suffix.
#
# We add an extra null byte on the end of the array, which is not
# included in the length, to allow it to be used as a string if needed.
#
set -e

arrname=""
modifiers=""
null_terminate=0

usage () {
  echo "Usage: $0 [options] [input file] " >&2
  echo "Options:" >&2
  echo "  -v <c array variable name>" >&2
  echo "     Name of C variable in output file. Must be provided." >&2
  echo "  -m <array variable modifiers>" >&2
  echo "     Modifiers for C variable in output file. Default is const." >&2
  echo "  -n" >&2
  echo "     Add a trailing null byte, not included in length." >&2
  exit 1
}

while getopts "m:nv:" opt; do
  case $opt in
    m)
      if [[ $modifiers != "" ]]; then
        echo "-m specified twice" >&2
        usage
      fi
      modifiers=$OPTARG
      ;;
    n)
      null_terminate=1
      ;;
    v)
      if [[ $arrname != "" ]]; then
        echo "-v specified twice" >&2
        usage
      fi
      arrname=$OPTARG
      ;;
    \?)
      echo "Invalid option: -$OPTARG" >&2
      usage
  esac
done
shift $((OPTIND - 1))

infile=$1
if [[ $# > 1 ]]; then
  echo "Too many remaining arguments: $@" >&2
  usage
fi

if [ -z "$arrname" ]; then
  echo "-v not provided or empty." >&2
  usage
fi

if [ -z "$modifiers" ]; then
  # Default is const with global linking visibility
  modifiers="const"
fi

echo "#include <stddef.h>" # For size_t
echo
# Preceding extern declaration guarantees external linkage in C++
echo "extern $modifiers unsigned char $arrname[];";
echo "extern $modifiers size_t ${arrname}_len;"
echo
echo "$modifiers unsigned char $arrname[] = {"
xxd -i < $infile
if [ ${null_terminate} = 1 ]; then
  echo ", 0x0"
fi
echo "};"

echo -n "$modifiers size_t ${arrname}_len = "
if [ ${null_terminate} = 1 ]; then
  echo "sizeof(${arrname}) - 1;"
else
  echo "sizeof(${arrname});"
fi
```

请解释：

```cmake
  # Convert LLVM bytecode to C array.
  set(IR_TMP_C_FILE ${IR_C_FILE}.tmp)
  add_custom_command(
    OUTPUT ${IR_C_FILE}
    COMMAND $ENV{IMPALA_HOME}/bin/file2array.sh -n -v ${VARNAME} ${IR_OUTPUT_FILE} > ${IR_TMP_C_FILE}
    COMMAND mv ${IR_TMP_C_FILE} ${IR_C_FILE}
    DEPENDS $ENV{IMPALA_HOME}/bin/file2array.sh
    DEPENDS ${IR_OUTPUT_FILE}
  )
endfunction(COMPILE_TO_IR_C_ARRAY)
```
这段 CMake 代码是 `COMPILE_TO_IR_C_ARRAY` 函数的**最后一步**，它的作用是：

> **将前面生成的 LLVM bitcode 文件（`.bc` 二进制文件）转换为一个 C 语言源文件（`.cc`），其中 bitcode 数据以静态数组的形式嵌入**，最终这个 `.cc` 文件会被直接编译进 Impala 的 `CodeGen` 库，实现“预生成 IR 静态嵌入二进制”的目标。

```cmake
  # Convert LLVM bytecode to C array.
  set(IR_TMP_C_FILE ${IR_C_FILE}.tmp)
  add_custom_command(
    OUTPUT ${IR_C_FILE}
    COMMAND $ENV{IMPALA_HOME}/bin/file2array.sh -n -v ${VARNAME} ${IR_OUTPUT_FILE} > ${IR_TMP_C_FILE}
    COMMAND mv ${IR_TMP_C_FILE} ${IR_C_FILE}
    DEPENDS $ENV{IMPALA_HOME}/bin/file2array.sh
    DEPENDS ${IR_OUTPUT_FILE}
  )
```

### 逐行详细解释

#### 1. `set(IR_TMP_C_FILE ${IR_C_FILE}.tmp)`
- 定义一个**临时文件名**：在最终目标文件（如 `impala-ir-o2.cc`）后面加 `.tmp` 后缀。
- 目的：先把脚本输出写入临时文件，避免在生成过程中如果失败导致最终文件被破坏（半成品）。

#### 2. `add_custom_command(...)`
这是一个自定义构建命令，告诉 CMake 如何生成最终的 `${IR_C_FILE}`（如 `impala-ir-o2.cc`）。

- **`OUTPUT ${IR_C_FILE}`**  
  指定这条命令的输出是最终的 `.cc` 文件。如果这个文件不存在或过时，CMake 就会执行下面的命令。

- **`COMMAND ...`（两条命令）**
  1. **第一条：调用 `file2array.sh` 脚本**
     ```bash
     $ENV{IMPALA_HOME}/bin/file2array.sh -n -v ${VARNAME} ${IR_OUTPUT_FILE} > ${IR_TMP_C_FILE}
     ```
     - `$ENV{IMPALA_HOME}/bin/file2array.sh`：调用你提供的那个 Bash 脚本。
     - `-n`：**添加一个额外的结尾 null 字节**（`\0`），方便如果需要时当作字符串使用（虽然 IR 是二进制，但加 null 更安全）。
     - `-v ${VARNAME}`：指定生成的 C 数组变量名（例如 `impala_llvm_o2_ir`）。
     - `${IR_OUTPUT_FILE}`：输入的 `.bc` 文件（例如 `impala-ir-o2.bc`）。
     - `> ${IR_TMP_C_FILE}`：把脚本输出重定向到临时文件。

     **这个脚本会做什么？**  
     根据你提供的 `file2array.sh` 内容，它会：
     - 用 `xxd -i` 把二进制文件转成十六进制数组形式。
     - 输出类似这样的 C 代码（简化示例）：
       ```cpp
       #include <stddef.h>

       extern const unsigned char impala_llvm_o2_ir[];
       extern const size_t impala_llvm_o2_ir_len;

       const unsigned char impala_llvm_o2_ir[] = {
         0x42, 0x43, 0xc0, 0xde, 0x00, 0x00, 0x00, 0x08, /* ... 成千上万字节 ... */
         , 0x00  // -n 选项添加的额外 null 字节
       };

       const size_t impala_llvm_o2_ir_len = 1234567;  // 不包括结尾的 null
       ```

  2. **第二条：移动临时文件到最终位置**
     ```bash
     mv ${IR_TMP_C_FILE} ${IR_C_FILE}
     ```
     - 把临时文件重命名为最终目标文件（如 `impala-ir-o2.cc`）。
     - 这样 CMake 认为输出文件已生成，构建继续。

- **`DEPENDS ...`**
  - 依赖 `file2array.sh` 脚本本身：如果脚本改了，就重新生成。
  - 依赖 `${IR_OUTPUT_FILE}`（即 `.bc` 文件）：这是前一步 `opt` 生成的，如果 bitcode 变了，就重新转成 C 数组。

### 实际生成的文件示例（最终结果）

构建完成后，你会在 `be/generated-sources/impala-ir/` 目录看到：

```cpp
// impala-ir-o2.cc（自动生成，大约几 MB）
#include <stddef.h>

extern const unsigned char impala_llvm_o2_ir[];
extern const size_t impala_llvm_o2_ir_len;

const unsigned char impala_llvm_o2_ir[] = {
  0x42, 0x43, 0xc0, 0xde, /* ... LLVM bitcode 的完整二进制内容 ... */
  , 0x00  // 额外 null 字节
};

const size_t impala_llvm_o2_ir_len = 2345678;  // 实际 bitcode 长度，不包括结尾 null
```

这个文件会被当作普通 C++ 源文件编译进 `libCodeGen`，最终嵌入 impalad 二进制。

### 运行时如何使用？

Impala 的 `llvm-codegen.cc` 中会有类似代码：
```cpp
if (codegen_level == 2) {
  LoadIrFromMemory(impala_llvm_o2_ir, impala_llvm_o2_ir_len);
}
```

直接从内存加载预优化 IR，开始 JIT 流程。

### 总结：这段代码的核心目的

> **将 LLVM bitcode（`.bc`）安全、可靠地转换为一个可编译的 C 数组源文件（`.cc`）**：
> - 使用临时文件避免生成失败导致损坏。
> - 添加 null 结尾增强兼容性。
> - 生成人类可读的变量名和长度常量。
> - 确保依赖正确（脚本或 bitcode 变化时自动重生）。

这是 Impala 实现“**预生成多版本优化 IR + 运行时 JIT 定制**”高性能策略的**最后一步**，也是整个 codegen 流水线的收尾。没有它，IR 就无法嵌入二进制，Impala 的查询性能会大幅下降。

*************************************************************
请解释：

```cmake

if (NOT CMAKE_SYSTEM_PROCESSOR STREQUAL "aarch64")
  set(PLATFORM_SPECIFIC_FLAGS "-mavx2")
  set(LEGACY_AVX_SPECIFIC_FLAGS "-mavx")
endif()
```
这段 CMake 代码的作用是：**根据当前构建目标的 CPU 架构（处理器类型），决定是否向编译器添加 x86 特定的 SIMD 指令集标志（AVX 和 AVX2）**，以便在支持这些指令的 CPU 上生成更高效的向量化代码。

```cmake
if (NOT CMAKE_SYSTEM_PROCESSOR STREQUAL "aarch64")
  set(PLATFORM_SPECIFIC_FLAGS "-mavx2")
  set(LEGACY_AVX_SPECIFIC_FLAGS "-mavx")
endif()
```

### 逐行详细解释

#### 1. `if (NOT CMAKE_SYSTEM_PROCESSOR STREQUAL "aarch64")`
- `CMAKE_SYSTEM_PROCESSOR` 是 CMake 自动检测的变量，表示**当前构建目标的处理器架构**。
  - 在 x86_64 机器上，通常是 `"x86_64"` 或 `"amd64"`
  - 在 ARM64 机器上，是 `"aarch64"`
- `NOT ... STREQUAL "aarch64"` 的意思是：
  - **如果不是 ARM64 架构**（即 x86_64），就执行里面的代码。
  - **如果是 ARM64**，则跳过，什么都不设置。

#### 2. `set(PLATFORM_SPECIFIC_FLAGS "-mavx2")`
- 为变量 `PLATFORM_SPECIFIC_FLAGS` 设置值 `"-mavx2"`
- `-mavx2` 是 GCC/Clang 的编译选项，含义是：
  - **启用 AVX2（Advanced Vector Extensions 2）指令集**
  - 允许编译器生成使用 256 位宽 YMM 寄存器的向量化指令
  - 可以显著提升浮点运算、整数运算、内存操作的性能（尤其是 Impala 的向量批量处理）

#### 3. `set(LEGACY_AVX_SPECIFIC_FLAGS "-mavx")`
- 为变量 `LEGACY_AVX_SPECIFIC_FLAGS` 设置值 `"-mavx"`
- `-mavx` 是更老的指令集标志：
  - 只启用 **AVX（第一代）**，不支持 AVX2 的新指令（如 256 位整数运算、Gather 等）
  - 用于兼容较老的 x86 CPU（例如某些 2011–2013 年的 Intel Sandy Bridge / Ivy Bridge）

### 为什么这样设计？

Impala 的运行时代码生成（runtime codegen）会**预编译核心计算内核**（如加法、哈希、聚合等），并嵌入到二进制中。为了最大化性能：

- 在 **现代 x86_64 CPU** 上（2013 年后的 Intel Haswell 及以后、AMD Zen 及以后），默认使用 **AVX2**（`-mavx2`），获得最佳性能。
- 为兼容**老旧 x86_64 CPU**（不支持 AVX2），额外生成一个只用 AVX 的版本（`-mavx`），称为 “legacy AVX” 版本。
- **ARM64（aarch64）** 架构完全不同：
  - 不支持 x86 的 AVX/AVX2 指令
  - 有自己的 SIMD 指令集（如 NEON、SVE）
  - 强行加 `-mavx2` 会导致编译错误
  - 因此在 ARM64 上**不添加任何 x86 特定标志**，让编译器使用默认的 ARM 优化

### 实际使用位置

这两个变量在后面的函数调用中被使用：

```cmake
COMPILE_TO_IR_C_ARRAY(... -O2 ${PLATFORM_SPECIFIC_FLAGS})      # 主版本：用 AVX2
COMPILE_TO_IR_C_ARRAY(... -O1 ${LEGACY_AVX_SPECIFIC_FLAGS})   # 遗留版本：只用 AVX
```

生成两个版本的预优化 IR：
- 主流版本：支持 AVX2 的高性能内核
- 遗留版本：兼容老 CPU 的保守内核

运行时 Impala 会检测 CPU 能力，自动选择最优版本加载。

### 总结

这段代码的核心目的：

> **只在 x86_64 架构上启用 AVX/AVX2 指令集优化**：
> - 主流版本使用 `-mavx2`（高性能）
> - 遗留版本使用 `-mavx`（兼容老 CPU）
> - 在 ARM64（aarch64）上不添加任何 x86 指令标志，避免编译错误

这是 Impala 在跨平台（x86_64 vs ARM64）构建中**性能与兼容性平衡**的典型工程实践：
- 在支持的平台上榨取最大性能
- 在不支持的平台上安全降级
- 不影响 ARM 服务器的构建和运行


******************************************************************
请解释：

```cmake

# At runtime impala will pick the file to load based on a startup parameter.
COMPILE_TO_IR_C_ARRAY(${IR_O1_C_FILE} impala_llvm_o1_ir -O1 ${PLATFORM_SPECIFIC_FLAGS})
COMPILE_TO_IR_C_ARRAY(${IR_O2_C_FILE} impala_llvm_o2_ir -O2 ${PLATFORM_SPECIFIC_FLAGS})
COMPILE_TO_IR_C_ARRAY(${IR_Os_C_FILE} impala_llvm_os_ir -Os ${PLATFORM_SPECIFIC_FLAGS})
COMPILE_TO_IR_C_ARRAY(${LEGACY_AVX_IR_C_FILE} impala_legacy_avx_llvm_ir -O1 ${LEGACY_AVX_SPECIFIC_FLAGS})
```‘
这段代码是 Apache Impala **运行时代码生成（runtime codegen）** 的关键构建步骤。它调用前面定义的 `COMPILE_TO_IR_C_ARRAY` 函数，**生成四个不同优化版本的预编译 LLVM IR 模板**，这些模板会被静态嵌入到 Impala 二进制中，供运行时根据启动参数动态选择加载。

```cmake
# At runtime impala will pick the file to load based on a startup parameter.
COMPILE_TO_IR_C_ARRAY(${IR_O1_C_FILE} impala_llvm_o1_ir -O1 ${PLATFORM_SPECIFIC_FLAGS})
COMPILE_TO_IR_C_ARRAY(${IR_O2_C_FILE} impala_llvm_o2_ir -O2 ${PLATFORM_SPECIFIC_FLAGS})
COMPILE_TO_IR_C_ARRAY(${IR_Os_C_FILE} impala_llvm_os_ir -Os ${PLATFORM_SPECIFIC_FLAGS})
COMPILE_TO_IR_C_ARRAY(${LEGACY_AVX_IR_C_FILE} impala_legacy_avx_llvm_ir -O1 ${LEGACY_AVX_SPECIFIC_FLAGS})
```

### 逐行详细解释

每行调用 `COMPILE_TO_IR_C_ARRAY` 函数，参数含义如下：

| 调用行 | 生成的目标文件 (`${IR_XX_C_FILE}`) | 数组变量名 | 优化级别 | 额外标志 (`${PLATFORM_SPECIFIC_FLAGS}` 或 `${LEGACY_AVX_SPECIFIC_FLAGS}`) | 用途 |
|--------|------------------------------------|------------|----------|-----------------------------------------------------------------------|------|
| 第1行 | `impala-ir-o1.cc`                  | `impala_llvm_o1_ir` | `-O1`   | `-mavx2`（x86_64） 或 无（ARM64）                                     | **轻量优化版本**：编译快、启动快、代码体积适中，适合调试或低负载场景 |
| 第2行 | `impala-ir-o2.cc`                  | `impala_llvm_o2_ir` | `-O2`   | `-mavx2`（x86_64） 或 无（ARM64）                                     | **平衡优化版本**（默认推荐）：性能与编译时间最佳折中，几乎所有生产环境使用 |
| 第3行 | `impala-ir-os.cc`                  | `impala_llvm_os_ir` | `-Os`   | `-mavx2`（x86_64） 或 无（ARM64）                                     | **最小化体积版本**：代码体积最小，适合内存极度受限的场景（如容器） |
| 第4行 | `impala-ir-legacy-avx.cc`          | `impala_legacy_avx_llvm_ir` | `-O1`   | `-mavx`（仅第一代 AVX，不含 AVX2）                                    | **兼容老 CPU 版本**：用于不支持 AVX2 的老旧 x86 CPU（2011–2013 年左右的机器） |

### 运行时如何选择？

Impala 启动时通过参数 `--codegen_level`（或内部 CPU 能力检测）决定加载哪个版本：

| 参数值 | 加载的数组变量 | 对应版本 | 典型场景 |
|--------|----------------|----------|----------|
| 0      | 无（禁用 codegen） | -        | 调试、排查 codegen 问题 |
| 1      | `impala_llvm_o1_ir` | -O1     | 快速启动、调试优先 |
| 2      | `impala_llvm_o2_ir` | -O2     | **生产默认**，性能最佳 |
| 3      | `impala_llvm_os_ir` | -Os     | 内存敏感环境 |
| 遗留 CPU 检测失败 | `impala_legacy_avx_llvm_ir` | -O1 + AVX | 老服务器兼容 |

如果 CPU 不支持 AVX2，Impala 会自动回退到 legacy 版本。

### 生成的文件位置和内容

构建完成后，你会在以下路径看到四个自动生成的 `.cc` 文件：

```
be/generated-sources/impala-ir/
├── impala-ir-o1.cc
├── impala-ir-o2.cc
├── impala-ir-os.cc
└── impala-ir-legacy-avx.cc
```

每个文件内容都是一个巨大的 C 数组（几 MB），例如 `impala-ir-o2.cc`：
```cpp
const unsigned char impala_llvm_o2_ir[] = { /* 优化后的 LLVM bitcode 二进制数据 */ };
const size_t impala_llvm_o2_ir_len = /* 数据长度 */;
```

这些文件被直接编译进 `libCodeGen`，实现**零外部依赖的 IR 加载**。

### 总结：这段代码的核心意义

> **为不同运行时场景预生成四个优化级别的 LLVM IR 模板**：
> - **O1**：快速、轻量
> - **O2**：性能最佳（生产默认）
> - **Os**：最小体积
> - **Legacy AVX**：兼容老 CPU

通过启动参数动态选择加载，实现：
- **极致性能**（现代 CPU 用 AVX2 + O2）
- **广泛兼容性**（老 CPU 也能运行）
- **灵活调试**（低优化级别便于分析）

这是 Impala 查询执行“**接近原生 C++ 性能**”的底层秘密之一：**多版本预优化 IR + 运行时智能选择**。  
没有这四行，Impala 的 codegen 将失去大部分灵活性和兼容性。


******************************************************************
请解释：

```cmake

if (BUILD_WITH_NO_TESTS)
  return()
endif()
```
这段 CMake 代码的作用是：**如果用户明确禁用了测试构建，就立即退出当前 CMakeLists.txt 文件的处理**，不再执行后面的所有代码。

```cmake
if (BUILD_WITH_NO_TESTS)
  return()
endif()
```

### 详细解释

#### 1. `BUILD_WITH_NO_TESTS` 变量的来源
这个变量在 Impala 的构建系统中是通过命令行参数控制的：

- 在 `./buildall.sh` 脚本中，如果你使用 **`-notests`** 参数，就会设置：
  ```bash
  BUILD_TESTS=0
  ```
  然后在 `generate_cmake_files` 函数中传入：
  ```cmake
  -DBUILD_WITH_NO_TESTS=ON
  ```

- 因此，当你运行：
  ```bash
  ./buildall.sh -notests
  ```
  时，CMake 配置阶段就会定义 `BUILD_WITH_NO_TESTS=ON`。

#### 2. `if (BUILD_WITH_NO_TESTS) ... endif()`
- 这是一个条件判断：
  - 如果 `BUILD_WITH_NO_TESTS` 为 `ON`（即用户禁用了测试），就执行里面的代码。
  - 否则跳过，继续执行后面的内容。

#### 3. `return()` 的作用
- `return()` 是 CMake 中的控制流命令。
- 在 `CMakeLists.txt` 文件中执行 `return()` 会**立即停止当前文件的处理**，后面的所有代码都不再执行，直接返回到父级 CMakeLists.txt。

#### 4. 这段代码出现在哪里？为什么需要它？

这段代码出现在 `be/src/codegen/CMakeLists.txt` 的**测试部分之前**：

```cmake
COMPILE_TO_IR_C_ARRAY(...)  # 生成各种 IR 版本（必须构建）

if (BUILD_WITH_NO_TESTS)
  return()
endif()

# 下面是测试相关的代码
add_library(CodeGenTests STATIC ...)
ADD_BE_LSAN_TEST(...)
ADD_UNIFIED_BE_LSAN_TEST(...)
```

**目的**：
- CodeGen 模块的**核心功能**（生成预编译 IR、嵌入到 libCodeGen）是 Impala 运行必须的，即使不构建测试也要生成。
- 但 CodeGen 模块还有一些**纯测试相关的目标**（CodeGenTests 库、llvm-codegen-test 等），这些在生产包或快速构建时不需要。
- 当用户用 `-notests` 时：
  - 希望**跳过所有测试相关的构建**（节省时间和磁盘空间）。
  - 但**不能跳过 CodeGen 的核心部分**（否则 impalad 无法运行）。
- 所以用 `return()` 精准控制：核心部分正常构建，测试部分直接跳过。

### 实际效果对比

| 构建命令                  | BUILD_WITH_NO_TESTS | 效果 |
|---------------------------|---------------------|------|
| `./buildall.sh`           | OFF（默认）         | 构建 CodeGen 核心 + 所有测试 |
| `./buildall.sh -notests`  | ON                  | 构建 CodeGen 核心，**跳过所有测试相关目标** |
| `./buildall.sh -skiptests`| OFF                 | 构建所有（包括测试），但运行时跳过测试执行 |

### 总结

这段代码的核心目的：

> **在用户明确禁止构建测试（`-notests`）时，立即退出当前 CMakeLists.txt 的后续处理**，  
> 从而**跳过所有测试相关的库和目标**（如 CodeGenTests、llvm-codegen-test 等），  
> 但**保留 CodeGen 模块的核心功能**（预生成 IR 嵌入），确保 impalad 仍能正常构建和运行。

这是 Impala 构建系统中常见的优化技巧：**区分“核心运行时必需”和“纯测试可选”**，让用户可以用 `-notests` 快速构建最小化二进制（节省时间和磁盘）。

**********************************************************************
请解释：

```cmake


add_library(CodeGenTests STATIC
  instruction-counter-test.cc
)
```
这段代码的作用是：**在 CodeGen 模块中创建一个专用于测试的静态库**，名字叫 `CodeGenTests`，它只包含一个测试源文件 `instruction-counter-test.cc`。

```cmake
add_library(CodeGenTests STATIC
  instruction-counter-test.cc
)
```

### 详细解释

#### 1. `add_library(CodeGenTests STATIC ... )`
- `add_library` 是 CMake 的核心命令，用于定义一个库目标。
- **目标名**：`CodeGenTests`
- **类型**：`STATIC`（静态库）
  - 生成 `libCodeGenTests.a`
  - 与 `SHARED`（动态库 .so）相对，这里选择静态是因为测试库不需要独立运行，只会被链接到测试可执行文件中。

#### 2. 源文件：`instruction-counter-test.cc`
- 这个文件是 **CodeGen 模块的专用单元测试**，专门测试 `instruction-counter.cc` 中的功能。
- `instruction-counter.cc` 的作用是：
  - 在 Impala 运行时代码生成（codegen）过程中，**统计生成的 LLVM IR 或机器码包含多少条指令**。
  - 用于性能分析、调试、以及防止生成的代码膨胀（instruction bloat）。
- 测试文件会：
  - 构造一些简单的表达式或函数
  - 触发 codegen
  - 检查生成的指令数量是否在预期范围内
  - 验证计数器是否准确

#### 3. 为什么单独创建一个测试库？

Impala 的后端测试系统采用**统一测试（unified backend tests）**模式：
- 所有后端测试代码最终链接到一个大测试可执行文件 `unifiedbetests` 中运行。
- 为了模块化管理，每个模块（如 CodeGen、Runtime、Exec 等）可以定义自己的**测试专用静态库**（如 `CodeGenTests`）。
- 在顶层 CMake 中，这些测试库会被统一收集并链接进 `unifiedbetests`。

好处：
- **隔离性**：测试代码不污染生产库 `CodeGen`
- **增量构建**：只改测试代码时，只重新编译这个小库
- **清晰性**：一眼看出哪些是测试目标

#### 4. 后续会发生什么？

稍后在顶层 `be/CMakeLists.txt` 中：
- `CodeGenTests` 会被加入测试专用库列表（如 `UNIFIED_TEST_LIBS`）
- 最终链接进统一测试程序
- 并通过 `ADD_UNIFIED_BE_LSAN_TEST(instruction-counter-test ...)` 注册为一个测试子集

运行测试时，你会看到类似输出：
```
[ RUN      ] InstructionCounterTest.SomeTestCase
[       OK ] InstructionCounterTest.SomeTestCase
```

### 总结

这段代码的核心目的：

> **为 CodeGen 模块创建一个独立的静态测试库 `CodeGenTests`**，  
> 只包含 `instruction-counter-test.cc` 一个测试文件，  
> 用于验证运行时代码生成过程中**指令计数器**的正确性。

它体现了 Impala 构建系统的优良实践：
- 生产代码和测试代码严格分离
- 测试库小而专注，便于维护和快速编译
- 为统一测试框架提供模块化输入

如果你运行 `./buildall.sh -notests`，由于前面有 `if (BUILD_WITH_NO_TESTS) return()`，这段代码根本不会执行，`CodeGenTests` 就不会被构建（节省时间和空间）。


*******************************************************************
请解释：

```cmake

)
add_dependencies(CodeGenTests gen-deps)
```
这段代码的作用是：**让测试专用静态库 `CodeGenTests` 的构建依赖于 `gen-deps` 这个自定义目标**。

```cmake
add_dependencies(CodeGenTests gen-deps)
```

### 详细解释

#### 1. `add_dependencies(target dependency)`
- CMake 命令，用于**显式声明构建依赖关系**。
- 含义：构建 `target`（这里是 `CodeGenTests`）之前，**必须先完成** `dependency`（这里是 `gen-deps`）。

#### 2. `gen-deps` 是什么？（回顾）
`gen-deps` 是 Impala 项目顶层 CMake 中定义的一个**聚合自定义目标**：

```cmake
add_custom_target(gen-deps ALL DEPENDS thrift-deps proto-deps fb-deps ... gen_ir_descriptions)
```

它负责确保所有**代码生成步骤**在后端构建开始前完成，包括：
- Thrift 接口生成（.thrift → C++/Java 代码）
- Protobuf 生成
- FlatBuffers 生成
- **IR 元数据生成**（`gen_ir_descriptions`：生成 `impala-ir-names.h` 和 `impala-ir-functions.h`）

#### 3. 为什么 `CodeGenTests` 需要依赖 `gen-deps`？

`CodeGenTests` 中的测试文件 `instruction-counter-test.cc` 很可能：
- 使用了某些 Thrift 生成的头文件（例如 RPC 接口）
- 或者包含了 IR 元数据头文件（`impala-ir-names.h` 等）用于验证函数名
- 或者依赖其他生成的代码

如果不加这个依赖，可能出现：
- 构建时 `instruction-counter-test.cc` 编译失败（头文件未找到）
- 即使编译通过，测试运行时行为异常（因为使用了过时的生成文件）

加了 `add_dependencies(CodeGenTests gen-deps)` 后：
- CMake 保证：先运行所有代码生成脚本（Thrift、Protobuf、IR 元数据等）
- 生成的文件就位后，再编译 `CodeGenTests`
- 测试代码能正确包含和使用最新的生成内容

#### 4. 实际构建顺序效果

运行 `make` 或 `./buildall.sh` 时：
1. 先执行 `gen-deps`（运行各种生成脚本）
2. 生成头文件（如 `impala-ir-names.h`）就位
3. 然后才编译 `instruction-counter-test.cc` → 生成 `libCodeGenTests.a`

#### 5. 与前面的 `add_dependencies(CodeGen gen-deps)` 的区别

- `CodeGen`（生产库）也依赖 `gen-deps`：因为它需要生成的 Thrift 接口和 IR 元数据来正常工作。
- `CodeGenTests`（测试库）同样依赖：因为测试代码也需要这些生成的文件来编译和验证行为。

两者都加依赖是**防御性编程**的体现，确保无论构建生产代码还是测试代码，环境都完整。

### 总结

这行代码的核心目的：

> **强制 CodeGen 模块的测试库 `CodeGenTests` 在构建前等待所有代码生成依赖（Thrift、Protobuf、IR 元数据等）完成**，  
> 避免因生成文件缺失或过时导致的测试编译失败或行为异常。

它是 Impala 构建系统中常见的**依赖管理实践**，尤其在有大量自动生成代码的项目中至关重要。  
和生产库 `CodeGen` 一样，测试库也必须在“生成环境就绪”后才能安全构建。

************************************************************************
请解释：

```cmake

# Run the clang compiler to generate BC for llvm-codegen-test
add_custom_target(test-loop.bc
  COMMAND ${LLVM_CLANG_EXECUTABLE} ${CLANG_IR_CXX_FLAGS} -O2 ${CLANG_INCLUDE_FLAGS} ${CMAKE_SOURCE_DIR}/testdata/llvm/test-loop.cc -o ${CMAKE_SOURCE_DIR}/llvm-ir/test-loop.bc
  SOURCES ${CMAKE_SOURCE_DIR}/testdata/llvm/test-loop.cc
)
```

这段 CMake 代码的作用是：**为 CodeGen 模块的测试创建一个自定义目标**，它使用 Clang 将一个专门的测试 C++ 源文件（`test-loop.cc`）交叉编译成 LLVM bitcode（`.bc` 文件），这个 bitcode 文件用于单元测试中验证 LLVM 代码生成的优化行为。

```cmake
# Run the clang compiler to generate BC for llvm-codegen-test
add_custom_target(test-loop.bc
  COMMAND ${LLVM_CLANG_EXECUTABLE} ${CLANG_IR_CXX_FLAGS} -O2 ${CLANG_INCLUDE_FLAGS} ${CMAKE_SOURCE_DIR}/testdata/llvm/test-loop.cc -o ${CMAKE_SOURCE_DIR}/llvm-ir/test-loop.bc
  SOURCES ${CMAKE_SOURCE_DIR}/testdata/llvm/test-loop.cc
)
```

### 逐参数详细解释

#### 1. `add_custom_target(test-loop.bc ...)`
- `add_custom_target`：创建**自定义构建目标**（custom target），名字是 `test-loop.bc`。
- 这个目标**不生成可执行文件**，而是执行指定的命令来生成一个 `.bc`（LLVM bitcode）文件。
- 目标名 `test-loop.bc` 暗示了它的输出文件就是 `test-loop.bc`。

#### 2. `COMMAND ...`（核心执行命令）
```bash
${LLVM_CLANG_EXECUTABLE} ${CLANG_IR_CXX_FLAGS} -O2 ${CLANG_INCLUDE_FLAGS} ${CMAKE_SOURCE_DIR}/testdata/llvm/test-loop.cc -o ${CMAKE_SOURCE_DIR}/llvm-ir/test-loop.bc
```

这是一个完整的 Clang 编译命令，分解如下：

- **`${LLVM_CLANG_EXECUTABLE}`**：toolchain 中的 Clang++ 可执行文件路径（如 `/opt/impala-toolchain/bin/clang++`）。

- **`${CLANG_IR_CXX_FLAGS}`**：前面定义的 IR 交叉编译专用选项，包括：
  - `-emit-llvm`：输出 LLVM IR（bitcode）而不是机器码
  - `-DIR_COMPILE`：Impala 自定义宏，移除 IR 中不必要的代码
  - `-DBOOST_NO_EXCEPTIONS`：禁用异常（JIT 代码中不能抛异常）
  - 其他兼容性宏和警告抑制

- **`-O2`**：**中等优化级别**，让生成的 bitcode 包含一些实际优化效果，便于测试验证。

- **`${CLANG_INCLUDE_FLAGS}`**：所有头文件搜索路径（Impala 源码、第三方库如 Boost、LLVM 等）。

- **输入文件**：`${CMAKE_SOURCE_DIR}/testdata/llvm/test-loop.cc`
  - `${CMAKE_SOURCE_DIR}`：项目根目录（如 `/home/user/impala`）
  - 完整路径：`/home/user/impala/testdata/llvm/test-loop.cc`
  - 这个文件是**测试专用**的简单 C++ 源码，专门设计成一个循环结构，用于测试 LLVM 的循环优化（如循环展开、向量化等）。

- **`-o ${CMAKE_SOURCE_DIR}/llvm-ir/test-loop.bc`**：
  - 输出文件：`/home/user/impala/llvm-ir/test-loop.bc`
  - 注意输出到**项目根目录下的 llvm-ir/**，而不是构建目录，便于测试脚本直接访问。

#### 3. `SOURCES ${CMAKE_SOURCE_DIR}/testdata/llvm/test-loop.cc`
- 声明这个目标**依赖于**源文件 `test-loop.cc`。
- 作用：
  - 如果源文件被修改，CMake 会重新执行命令生成新的 `.bc`。
  - 让 IDE（如 CLion、VS Code）能正确识别这个目标的源文件，提供语法高亮、跳转等功能。

### 这个目标的用途：为 `llvm-codegen-test` 提供测试输入

从注释可以看出：
> # Run the clang compiler to generate BC for llvm-codegen-test

这个 `.bc` 文件是 **CodeGen 模块的单元测试 `llvm-codegen-test`** 的输入数据：

- 测试流程：
  1. 构建时生成 `test-loop.bc`（一个预优化的循环 bitcode）。
  2. 测试程序加载这个 bitcode 到 LLVM 引擎。
  3. 运行各种优化通行证（passes），验证：
     - 优化是否正确（循环展开、指令计数等）
     - 生成的机器码性能是否符合预期
     - 没有崩溃或未定义行为

- `test-loop.cc` 源码大概长这样（简化）：
  ```cpp
  int sum = 0;
  for (int i = 0; i < N; i++) {
    sum += i * 2;  // 简单循环，易于优化验证
  }
  return sum;
  ```

- 编译成 bitcode 后，测试可以反复加载、优化、执行这个循环，检查 LLVM 行为。

### 为什么用 add_custom_target 而不是 add_executable？

- 这个目标**不生成可执行文件**，只生成一个 `.bc` 数据文件。
- `add_custom_target` 适合这种“运行命令生成文件”的场景。
- 后续测试目标（如 `llvm-codegen-test`）会依赖这个目标：
  ```cmake
  add_dependencies(llvm-codegen-test test-loop.bc)
  ```

### 实际执行效果

运行 `make test-loop.bc` 或普通 `make` 时：
1. CMake 执行 Clang 命令，生成 `llvm-ir/test-loop.bc`
2. 如果源文件 `test-loop.cc` 改动，自动重新生成
3. 生成的 `.bc` 文件是二进制（用 `llvm-dis test-loop.bc` 可以反汇编查看）

### 总结

这段代码的核心目的：

> **创建一个构建目标 `test-loop.bc`**，  
> 在构建时用 Clang 将测试专用循环源码 `test-loop.cc` 交叉编译成 LLVM bitcode，  
> 为 CodeGen 模块的单元测试 `llvm-codegen-test` 提供**预优化的测试输入数据**，  
> 便于验证 LLVM 优化通行证的正确性和性能。

这是 Impala 测试系统的一种高级设计：**预生成测试输入**，确保测试的确定性和可重复性，尤其适合验证复杂优化器（如 LLVM JIT）行为。


***************************************************

```cmake
# Run the clang compiler to generate BC for llvm-codegen-test
add_custom_target(test-loop.bc
  COMMAND ${LLVM_CLANG_EXECUTABLE} ${CLANG_IR_CXX_FLAGS} -O2 ${CLANG_INCLUDE_FLAGS} ${CMAKE_SOURCE_DIR}/testdata/llvm/test-loop.cc -o ${CMAKE_SOURCE_DIR}/llvm-ir/test-loop.bc
  SOURCES ${CMAKE_SOURCE_DIR}/testdata/llvm/test-loop.cc
)
其中用了SOURCES关键字，和DEPENDS有什么区别？
```
在 CMake 的 `add_custom_target` 中，**`SOURCES` 和 `DEPENDS` 都是用来声明依赖关系的**，但它们的**语义、作用范围和使用场景有明显区别**。

下面以你提供的这段代码为例，详细对比说明：

```cmake
add_custom_target(test-loop.bc
  COMMAND ${LLVM_CLANG_EXECUTABLE} ... -o .../llvm-ir/test-loop.bc
  SOURCES ${CMAKE_SOURCE_DIR}/testdata/llvm/test-loop.cc
)
```

### 1. `SOURCES` 的作用（你代码中用的这个）

- **语义**：告诉 CMake 这个自定义目标“逻辑上依赖”这些源文件。
- **实际效果**：
  - **不会自动触发重新构建**：即使 `test-loop.cc` 修改了，运行 `make test-loop.bc` 时**不一定**会重新执行 `COMMAND`（取决于 CMake 版本和生成器）。
  - **主要用于 IDE 支持**：让支持 CMake 的 IDE（如 CLion、VS Code、Qt Creator）能够：
    - 正确显示这个目标关联的源文件
    - 提供语法高亮、跳转到定义、代码补全等功能
    - 在项目树中把源文件和目标关联起来
  - **不影响构建依赖图**：CMake 的依赖分析器通常不会把 `SOURCES` 当作真正的构建依赖。

**结论**：`SOURCES` 更像是一个**“提示性”依赖**，主要改善开发体验，对实际构建行为影响很小或没有。

### 2. `DEPENDS` 的作用（更推荐用于这种场景）

- **语义**：明确告诉 CMake 这个目标**真正依赖**这些文件。
- **实际效果**：
  - 如果列在 `DEPENDS` 中的文件修改了，CMake 会**自动认为目标过时**，重新执行 `COMMAND`。
  - 完全参与 CMake 的依赖分析和增量构建。
  - 是 CMake 官方推荐的、可靠的方式来表达自定义目标的输入依赖。

**如果改用 `DEPENDS`**，代码应该是：

```cmake
add_custom_target(test-loop.bc
  COMMAND ${LLVM_CLANG_EXECUTABLE} ${CLANG_IR_CXX_FLAGS} -O2 ${CLANG_INCLUDE_FLAGS}
          ${CMAKE_SOURCE_DIR}/testdata/llvm/test-loop.cc
          -o ${CMAKE_SOURCE_DIR}/llvm-ir/test-loop.bc
  DEPENDS ${CMAKE_SOURCE_DIR}/testdata/llvm/test-loop.cc   # ← 关键在这里
)
```

这样：
- 修改 `test-loop.cc` → 运行 `make` → 自动重新生成 `test-loop.bc`
- 完全符合增量构建原则

### 3. 总结对比表

| 特性                     | `SOURCES`                          | `DEPENDS`                              |
|--------------------------|------------------------------------|----------------------------------------|
| 是否触发重新构建         | 不一定（多数情况下不触发）         | 一定会触发                             |
| 是否参与依赖分析         | 否                                 | 是                                     |
| 主要用途                 | 改善 IDE 显示和导航                | 实现正确的增量构建                     |
| 对构建行为的影响         | 很小或无                           | 直接影响                               |
| 官方推荐度               | 低（仅用于源文件关联）             | 高（自定义目标依赖的标准方式）         |
| 在你的场景中是否合适     | 不太合适（希望源码改了自动重生）   | **强烈推荐**                           |

### 4. Impala 项目中的实际情况

Impala 的这个 `CMakeLists.txt` 使用 `SOURCES` 而不是 `DEPENDS`，可能有以下原因：
- 历史遗留（早期 CMake 版本行为不同）
- 实际测试中依赖关系通过其他方式保证（比如测试目标依赖这个 custom target）
- 更注重 IDE 友好性

但从现代 CMake 最佳实践来看，**应该使用 `DEPENDS`** 来确保修改 `test-loop.cc` 时能自动重新生成 bitcode。

### 推荐修改（更正确）

```cmake
add_custom_target(test-loop.bc
  COMMAND ${LLVM_CLANG_EXECUTABLE} ${CLANG_IR_CXX_FLAGS} -O2 ${CLANG_INCLUDE_FLAGS}
          ${CMAKE_SOURCE_DIR}/testdata/llvm/test-loop.cc
          -o ${CMAKE_SOURCE_DIR}/llvm-ir/test-loop.bc
  DEPENDS ${CMAKE_SOURCE_DIR}/testdata/llvm/test-loop.cc   # 推荐用这个
  COMMENT "Generating test-loop.bc for llvm-codegen-test"
)
```

这样就能实现：**源码修改 → 自动重新生成测试 bitcode → 测试始终使用最新版本**。

*********************************************************************
```cmake
# Run the clang compiler to generate BC for llvm-codegen-test
add_custom_target(test-loop.bc
  COMMAND ${LLVM_CLANG_EXECUTABLE} ${CLANG_IR_CXX_FLAGS} -O2 ${CLANG_INCLUDE_FLAGS} ${CMAKE_SOURCE_DIR}/testdata/llvm/test-loop.cc -o ${CMAKE_SOURCE_DIR}/llvm-ir/test-loop.bc
  SOURCES ${CMAKE_SOURCE_DIR}/testdata/llvm/test-loop.cc
)
其中用了SOURCES关键字，和DEPENDS有什么区别？
```
请继续解释：

```cmake

# Provide "unoptimized" output. Clang O0 adds attributes to everything which prevents us
# doing optimizations in the test, so use O1 instead.
add_custom_target(test-opt.bc
  COMMAND ${LLVM_CLANG_EXECUTABLE} ${CLANG_IR_CXX_FLAGS} -O1 ${CLANG_INCLUDE_FLAGS} ${CMAKE_SOURCE_DIR}/testdata/llvm/test-opt.cc -o ${CMAKE_SOURCE_DIR}/llvm-ir/test-opt.bc
  SOURCES ${CMAKE_SOURCE_DIR}/testdata/llvm/test-opt.cc
)
```

这段 CMake 代码的作用是：**为 CodeGen 模块的单元测试创建一个自定义构建目标 `test-opt.bc`**，它使用 Clang 将一个专门的测试源文件 `test-opt.cc` 交叉编译成 LLVM bitcode（`.bc` 文件），并特别说明了为什么使用 `-O1` 而不是 `-O0`。

```cmake
# Provide "unoptimized" output. Clang O0 adds attributes to everything which prevents us
# doing optimizations in the test, so use O1 instead.
add_custom_target(test-opt.bc
  COMMAND ${LLVM_CLANG_EXECUTABLE} ${CLANG_IR_CXX_FLAGS} -O1 ${CLANG_INCLUDE_FLAGS} ${CMAKE_SOURCE_DIR}/testdata/llvm/test-opt.cc -o ${CMAKE_SOURCE_DIR}/llvm-ir/test-opt.bc
  SOURCES ${CMAKE_SOURCE_DIR}/testdata/llvm/test-opt.cc
)
```

### 逐部分解释

#### 1. 注释的含义
```cmake
# Provide "unoptimized" output. Clang O0 adds attributes to everything which prevents us
# doing optimizations in the test, so use O1 instead.
```
- 作者想生成一个**“几乎未优化”**的 bitcode 文件，用于测试 LLVM 优化器的行为。
- 理想情况下应该用 `-O0`（完全不优化），这样生成的 IR 更接近原始源码，便于验证优化通行证是否正确工作。
- 但 Clang 在 `-O0` 时会给**几乎所有指令**添加调试属性（`!dbg` 元数据）和一些其他属性，这些属性会**干扰后续优化测试**：
  - 某些优化通行证（如内联、循环展开）在有大量调试元数据时可能被禁用或行为异常。
  - 测试结果不准确（无法区分是优化器的问题还是调试属性导致的）。
- 因此折中选择了 **`-O1`**：
  - 开启最轻量的优化（常量折叠、简单指令合并等）
  - 不会添加太多干扰调试属性的元数据
  - 仍然保留大部分原始结构，便于测试优化效果

#### 2. `add_custom_target(test-opt.bc ... )`
- 创建一个名为 `test-opt.bc` 的自定义目标。
- 这个目标**不生成可执行文件**，而是执行命令生成一个 bitcode 文件 `llvm-ir/test-opt.bc`。

#### 3. `COMMAND` 部分（实际执行的编译命令）
```bash
${LLVM_CLANG_EXECUTABLE} ${CLANG_IR_CXX_FLAGS} -O1 ${CLANG_INCLUDE_FLAGS} ${CMAKE_SOURCE_DIR}/testdata/llvm/test-opt.cc -o ${CMAKE_SOURCE_DIR}/llvm-ir/test-opt.bc
```

- **输入文件**：`${CMAKE_SOURCE_DIR}/testdata/llvm/test-opt.cc`
  - 项目根目录下的 `testdata/llvm/test-opt.cc`
  - 这是一个**专门为测试设计的简单 C++ 文件**，很可能包含一些循环、条件分支、函数调用等结构，方便测试 LLVM 的优化通行证（如循环向量化、内联、指令重排等）。

- **编译选项**：
  - `${LLVM_CLANG_EXECUTABLE}`：toolchain 中的 Clang++。
  - `${CLANG_IR_CXX_FLAGS}`：IR 交叉编译专用选项（`-emit-llvm`、`-DIR_COMPILE` 等）。
  - `-O1`：轻量优化（避免 -O0 的属性干扰）。
  - `${CLANG_INCLUDE_FLAGS}`：所有头文件搜索路径。
  - `-o .../llvm-ir/test-opt.bc`：输出到项目根目录的 `llvm-ir/` 下（与 `test-loop.bc` 同目录）。

#### 4. `SOURCES` 参数
```cmake
SOURCES ${CMAKE_SOURCE_DIR}/testdata/llvm/test-opt.cc
```
- 声明这个目标**依赖于**源文件 `test-opt.cc`。
- 作用：
  - 如果源文件修改，CMake 会标记目标为过时，重新执行命令。
  - 让支持 CMake 的 IDE 能正确识别源文件，提供语法高亮、跳转等功能。
- **注意**：这里用 `SOURCES` 而不是 `DEPENDS`（前面解释过区别），`SOURCES` 更偏向 IDE 支持，`DEPENDS` 更强调构建依赖。实际中用 `DEPENDS` 会更可靠。

#### 5. 这个目标的用途（与测试的关系）
从注释可以看出：
> # Run the clang compiler to generate BC for llvm-codegen-test

这个 `test-opt.bc` 是专门为测试目标 `llvm-codegen-test` 准备的输入数据：

- 测试程序 `llvm-codegen-test` 会：
  1. 加载这个预生成的 bitcode 文件。
  2. 运行 LLVM 的各种优化通行证。
  3. 检查优化后的 IR 是否符合预期（指令数、结构等）。
- 使用 `-O1` 而不是 `-O0` 是为了让生成的 bitcode **既有一定优化痕迹**，又**不会因为调试属性干扰测试**。

### 实际构建效果

运行 `make test-opt.bc` 或普通 `make` 时：
1. CMake 检查 `test-opt.cc` 是否修改。
2. 如果需要，执行 Clang 命令生成 `llvm-ir/test-opt.bc`。
3. 生成的 `.bc` 文件是二进制 bitcode，后续测试会用 `llvm-dis` 反汇编查看，或直接加载到 LLVM 引擎进行优化测试。

### 总结

这段代码的核心目的：

> **创建一个构建目标 `test-opt.bc`**，  
> 在构建时用 Clang 将测试专用源码 `test-opt.cc` 交叉编译成 LLVM bitcode，  
> 并**故意使用 `-O1` 而不是 `-O0`**，以避免 Clang -O0 模式添加过多调试属性干扰后续优化测试。

这个 bitcode 是 `llvm-codegen-test` 的**预生成测试输入**，用于验证 CodeGen 模块中 LLVM 优化通行证的正确性和效果。  
这是 Impala 测试系统的一种高级设计：**预生成确定性输入**，确保测试结果可重复、可验证。
**************************************************************************
请解释：

```cmake

# Exception to unified be tests: custom main initializes LLVM
ADD_BE_LSAN_TEST(llvm-codegen-test)
add_dependencies(llvm-codegen-test test-loop.bc test-opt.bc)
ADD_BE_LSAN_TEST(llvm-codegen-cache-test LlvmCodegenCacheTest.*)

ADD_UNIFIED_BE_LSAN_TEST(instruction-counter-test InstructionCounterTest.*)
```

这段代码位于 `be/src/codegen/CMakeLists.txt` 的最后部分，是 **CodeGen 模块测试目标的注册**。它为 CodeGen 模块添加了三个单元测试，并处理了其中一个测试的特殊情况。

```cmake
# Exception to unified be tests: custom main initializes LLVM
ADD_BE_LSAN_TEST(llvm-codegen-test)
add_dependencies(llvm-codegen-test test-loop.bc test-opt.bc)
ADD_BE_LSAN_TEST(llvm-codegen-cache-test LlvmCodegenCacheTest.*)

ADD_UNIFIED_BE_LSAN_TEST(instruction-counter-test InstructionCounterTest.*)
```

### 逐行详细解释

#### 1. `ADD_BE_LSAN_TEST(llvm-codegen-test)`
- 调用 Impala 自定义的宏 `ADD_BE_LSAN_TEST`（定义在顶层 `be/CMakeLists.txt`）。
- 这个宏的作用是：
  - 使用 `ADD_BE_TEST` 添加一个**独立的后端测试**（standalone test），即生成一个**单独的可执行文件**（而不是加入统一测试程序）。
  - 同时调用 `ENABLE_LSAN_FOR_TEST` 启用 **LeakSanitizer**（内存泄漏检测）。

- **为什么是“Exception”（例外）？**  
  注释明确说明：
  > # Exception to unified be tests: custom main initializes LLVM

  Impala 的后端测试主流方式是**统一测试**（unified backend tests）：所有测试用例合并到一个大可执行文件 `unifiedbetests` 中运行，通过 `--gtest_filter` 选择子集。

  但 `llvm-codegen-test` 是一个**例外**，原因：
  - 这个测试需要**手动初始化 LLVM 环境**（调用 `LLVMInitializeNativeTarget()` 等）。
  - 如果把它加入统一测试框架，统一框架的 `main()` 函数会先运行，LLVM 可能已经被初始化，导致测试无法控制初始化顺序或测试特定初始化行为。
  - 因此必须用**自定义的 `main()` 函数**，只能作为独立测试（standalone）。

#### 2. `add_dependencies(llvm-codegen-test test-loop.bc test-opt.bc)`
- 让测试目标 `llvm-codegen-test` **依赖于前面两个自定义目标**：
  - `test-loop.bc`（前面生成的循环测试 bitcode）
  - `test-opt.bc`（前面生成的轻优化测试 bitcode）

- **效果**：
  - 构建 `llvm-codegen-test` 前，必须先完成这两个 `.bc` 文件的生成。
  - 如果 `test-loop.cc` 或 `test-opt.cc` 修改了，`.bc` 会重新生成，测试可执行文件也会重新链接/构建。

#### 3. `ADD_BE_LSAN_TEST(llvm-codegen-cache-test LlvmCodegenCacheTest.*)`
- 同样添加一个**独立测试** `llvm-codegen-cache-test`。
- 第二个参数 `LlvmCodegenCacheTest.*` 是 **gtest_filter 模式**（虽然这里是独立测试，但宏内部可能仍会用到）。
- 这个测试验证 CodeGen 的**缓存机制**（避免重复生成相同表达式）。
- 也是独立测试，可能因为需要特殊初始化或隔离环境。

#### 4. `ADD_UNIFIED_BE_LSAN_TEST(instruction-counter-test InstructionCounterTest.*)`
- 调用另一个宏 `ADD_UNIFIED_BE_LSAN_TEST`。
- 这个宏的作用：
  - 使用 `ADD_UNIFIED_BE_TEST` 将测试加入**统一测试框架**（unified backend tests）。
  - 生成一个 wrapper 脚本，运行统一测试程序并用 `--gtest_filter=InstructionCounterTest.*` 只执行这个测试的用例。
  - 同时启用 LeakSanitizer。

- `instruction-counter-test` 是**正常加入统一测试**的例子（前面已经创建了 `CodeGenTests` 静态库包含其源码）。

### 总结：这四行代码注册了三个测试

| 测试名                        | 类型               | 是否启用 LSAN | 特殊说明                                                                 |
|-------------------------------|--------------------|---------------|--------------------------------------------------------------------------|
| `llvm-codegen-test`           | 独立测试（standalone） | 是            | **例外**：需要自定义 `main()` 初始化 LLVM，不能加入统一测试             |
| `llvm-codegen-cache-test`     | 独立测试           | 是            | 可能也需要特殊环境                                                       |
| `instruction-counter-test`    | 统一测试（unified）| 是            | 正常加入统一测试框架，验证指令计数器功能                                 |

### 为什么 `llvm-codegen-test` 是例外？

LLVM 初始化是**全局单例**的行为：
- 只能调用一次某些初始化函数（如 `LLVMInitializeNativeTarget()`）。
- 统一测试框架的 `main()` 会先初始化 LLVM。
- 如果测试需要：
  - 测试初始化失败的情况
  - 测试多次初始化
  - 控制初始化顺序
- 就必须自己写 `main()`，无法复用统一框架的 `main()`。

### 实际运行效果

- 构建完成后：
  - `llvm-codegen-test` 和 `llvm-codegen-cache-test` 是两个独立的可执行文件。
  - `instruction-counter-test` 是一个脚本，运行 `unifiedbetests --gtest_filter=InstructionCounterTest.*`。
- 运行 `./buildall.sh` 或 `make be-test` 时，所有三个测试都会构建并可运行。

这段代码体现了 Impala 测试系统的**灵活性**：主流用高效的统一测试，但为特殊需求保留独立测试的能力。


********************************************************
[llvm-dev] External function resolution: MCJIT vs ORC JIT
已查看 749 次
订阅 
Larry Gritz via llvm-dev的个人资料照片
Larry Gritz via llvm-dev
2016年5月18日 03:13:59
收件人 llvm...@lists.llvm.org
When using ORC JIT, I'm having trouble with external function resolution (that is, of a function defined in the app, with C linkage).
I add a declaration for the function to my IR, and when I use MCJIT, it finds it and all is well, But when I use ORC JIT (I *think* correctly, at least it closely matches what I see in the tutorial), I get an LLVM error, "Program used external function 'sqr' which could not be resolved."

(Excuse my coming to the ORC party late, I've been stuck on an older LLVM for my DSL, and am only now jumping forward several versions at once, with some growing pains.)

I've boiled it down to the minimal example below. When I build with orc=0, thus using MCJIT, it works fine and I get the expected output. When I build with orc=1, it fails as I described.

I'm having this trouble with LLVM 3.7 and 3.8, on both Linux and OSX.

I figure I'm probably just getting some part of the magic incantation wrong. Can anybody help a poor guy out and spot the error of my ways?

-----

#include <llvm/ADT/STLExtras.h>
#include <llvm/Bitcode/ReaderWriter.h>
#include <llvm/ExecutionEngine/ExecutionEngine.h>
#include <llvm/ExecutionEngine/Orc/CompileUtils.h>
#include <llvm/ExecutionEngine/RuntimeDyld.h>
#include <llvm/ExecutionEngine/Orc/IRCompileLayer.h>
#include <llvm/ExecutionEngine/Orc/LambdaResolver.h>
#include <llvm/ExecutionEngine/Orc/ObjectLinkingLayer.h>
#include <llvm/IR/DataLayout.h>
#include <llvm/IR/IRBuilder.h>
#include <llvm/IR/LegacyPassManager.h>
#include <llvm/IR/LLVMContext.h>
#include <llvm/IR/Mangler.h>
#include <llvm/IR/Module.h>
#include <llvm/IR/Verifier.h>
#include <llvm/Support/TargetSelect.h>
#include <llvm/Target/TargetMachine.h>
#include <llvm/Transforms/IPO.h>
#include <llvm/Transforms/IPO/PassManagerBuilder.h>
#include <llvm/Transforms/Scalar.h>
#include <llvm/Transforms/Utils/UnifyFunctionExitNodes.h>

template <typename T>
inline std::vector<T> singletonSet (T t)
{
std::vector<T> Vec;
Vec.push_back(std::move(t));
return Vec;
}


///
/// THIS is the function I want my IR to call
///
extern "C" {
float sqr (float x) { return x*x; }
}


void
simple ()
{
llvm::InitializeAllTargets();
llvm::InitializeAllTargetMCs();
llvm::InitializeAllAsmPrinters();
llvm::InitializeAllAsmParsers();
llvm::LLVMContext Context;
std::unique_ptr<llvm::TargetMachine> TM (llvm::EngineBuilder().selectTarget());
std::unique_ptr<llvm::DataLayout> DL;
DL.reset (new llvm::DataLayout (TM->createDataLayout()));
std::unique_ptr<llvm::ExecutionEngine> EE;
typedef llvm::orc::ObjectLinkingLayer<> ObjLayerT;
typedef llvm::orc::IRCompileLayer<ObjLayerT> CompileLayerT;
typedef CompileLayerT::ModuleSetHandleT ModuleHandleT;
ObjLayerT Objlayer;
CompileLayerT Compilelayer (Objlayer, llvm::orc::SimpleCompiler(*TM));
std::unique_ptr<llvm::Module> M (new llvm::Module("module", Context));
M->setDataLayout (*DL);

// Declare stub for external function sqr
auto type_float = llvm::Type::getFloatTy (Context);
llvm::Type* one_float[] = { type_float };
llvm::FunctionType *functype_ff = llvm::FunctionType::get (type_float, one_float, false);
llvm::Function::Create (functype_ff, llvm::Function::ExternalLinkage,
"sqr", M.get());

// Create myfunc and generate its IR, which just calls sqr on its argument
llvm::Function *myfunc = llvm::Function::Create (functype_ff,
llvm::Function::ExternalLinkage,
"myfunc", M.get());
llvm::IRBuilder<> builder (Context);
auto block = llvm::BasicBlock::Create (Context, "", myfunc);
builder.SetInsertPoint (block);
llvm::Value *a = llvm::cast<llvm::Value>(myfunc->arg_begin());
llvm::Value *asq = builder.CreateCall (M->getFunction ("sqr"), a);
builder.CreateRet (asq);

// Set up compilation
if (orc) {
auto Resolver = llvm::orc::createLambdaResolver(
// External lookup functor
[&](const std::string &name) {
if (auto Sym = Compilelayer.findSymbol(name, true))
return llvm::RuntimeDyld::SymbolInfo(Sym.getAddress(), Sym.getFlags());
// If not found as a symbol, look up in current process.
// Why doesn't this work?
if (auto Addr = llvm::RTDyldMemoryManager::getSymbolAddressInProcess(name))
return llvm::RuntimeDyld::SymbolInfo(Addr, llvm::JITSymbolFlags::Exported);
return llvm::RuntimeDyld::SymbolInfo(nullptr);
},
// Dylib lookup functor
[&](const std::string &name) { return nullptr; }
);
Compilelayer.addModuleSet (singletonSet(std::move(M)),
llvm::make_unique<llvm::SectionMemoryManager>(),
std::move(Resolver));
} else {
// MCJIT
std::string engine_errors;
llvm::EngineBuilder engine_builder (std::move(M));
engine_builder.setEngineKind (llvm::EngineKind::JIT)
.setOptLevel (llvm::CodeGenOpt::Default) // Aggressive?
.setErrorStr (&engine_errors);
EE.reset (engine_builder.create());
EE->finalizeObject ();
}

// Ask for a callable function
typedef float (*FuncFloatFloat)(float);
FuncFloatFloat my_executable_function = NULL;
if (orc) {
auto ExprSymbol = Compilelayer.findSymbol ("myfunc", true);
my_executable_function = (FuncFloatFloat) ExprSymbol.getAddress ();
} else {
my_executable_function = (FuncFloatFloat) EE->getFunctionAddress ("myfunc");
}

assert (my_executable_function);
printf ("myfunc(42.0f) = %g\n", (*my_executable_function)(42.0f));
}

--
Larry Gritz
l...@larrygritz.com


_______________________________________________
LLVM Developers mailing list
llvm...@lists.llvm.org
http://lists.llvm.org/cgi-bin/mailman/listinfo/llvm-dev

David Blaikie via llvm-dev的个人资料照片
David Blaikie via llvm-dev
2016年5月18日 04:05:53
收件人 Larry Gritz、Lang Hames、llvm-dev
+Lang for all your ORC related inquiries
Lang Hames via llvm-dev的个人资料照片
Lang Hames via llvm-dev
2016年5月19日 04:52:22
收件人 Larry Gritz、LLVM Developers Mailing List
Hi Larry,

You're basically there, but you're hitting a couple of subtle issues:

(1) On both platforms you'll want to call llvm::sys::DynamicLibrary::LoadLibraryPermanently(nullptr) at program startup. This makes exported symbols in the main program searchable by RTDyldMemoryManager::getSymbolAddressInProcess (important for making 'sqr' findable on any platform).

(2) On Linux (if I understand correctly) symbols aren't exported from the main process by default, so even if you've called DynamicLibrary::LoadLibraryPermanently, sqr won't show up. To fix this you can add -Wl,export-dynamic to your link line. This will flip the default and export symbols from the main binary, allowing getSymbolAddressInProcess to find sqr. On MacOS this isn't necessary.

(3) On MacOS, C names are mangled by prepending an '_'. To maintain consistency with statically linked programs, the JIT uses these mangled names so you need to search for '_myfunc', rather than 'myfunc'. I usually add a 'mangle' function that uses the DataLayout, then search for 'mangle(<name>)', which does the right thing on all platforms.

I've attached a new version of your code with these changes included. Hope this helps!

- Lang.


On Tue, May 17, 2016 at 12:13 PM, Larry Gritz via llvm-dev <llvm...@lists.llvm.org> wrote:

larrys_jit.cpp
Larry Gritz via llvm-dev的个人资料照片
Larry Gritz via llvm-dev
2016年5月20日 02:57:45
收件人 Lang Hames、LLVM Developers Mailing List
Thanks so much! This seems to do the trick. I would have spun my wheels for a long time before discovering all of this, wow.

Do I even want to know what additional chickens need to be sacrificed to get this to work on Windows?

	-- lg

<larrys_jit.cpp>

--
Larry Gritz
l...@larrygritz.com


Lang Hames via llvm-dev的个人资料照片
Lang Hames via llvm-dev
2016年5月21日 01:30:16
收件人 Larry Gritz、LLVM Developers Mailing List
Hi Larry,

Thanks so much! This seems to do the trick. I would have spun my wheels for a long time before discovering all of this, wow.

No worries. :)
I'll try to keep this in mind and make sure I address it in future Kaleidoscope tutorial chapters - these issues tripped me up the first time I encountered them too.

Do I even want to know what additional chickens need to be sacrificed to get this to work on Windows?

I don't build on Windows myself, so I'm not certain. I know there are people using ORC on Windows though, and evidently at least simple code works. I only know of one big gotcha, which is that COFF files don't properly handle the SF_Exported symbol flag, so whenever you go searching for a symbol in the JIT on windows you have to call findSymbol with ExportedSymbolsOnly=false. Other than that I don't think usage should differ too much between platforms.

On the feature support side: the JIT linker for Windows (RuntimeDyldCOFF) is less mature, so you may run into unimplemented features there. While I don't have much time to work on Windows support myself, we definitely want to improve support for it so patches, bug reports and feature requests are all very welcome.

In case it's useful too, here's a link to the LLILC JIT file, which uses a simple ORC setup on Windows: https://github.com/dotnet/llilc/blob/dd12743f9cdb5418f1c39b2cd756da1e8396a922/lib/Jit/LLILCJit.cpp#L299

Cheers,
Lang.
Alex Denisov via llvm-dev的个人资料照片
Alex Denisov via llvm-dev
2016年5月23日 03:23:35
收件人 Lang Hames、LLVM Developers Mailing List、Larry Gritz
>> llvm::sys::DynamicLibrary::LoadLibraryPermanently(nullptr)

This is one is a bit tricky and hard to find.
I spent quiet some time digging into MC and ORC JIT execution engines trying to find what makes them work.
The problem is that this trick (LoadLibraryPermanently) happens inside of EngineBuilder, despite that the functionality belongs to a JIT engine itself, not to the builder.

I think this part can be improved by moving the method call into an engine itself, or by documenting the steps required to create custom engine on top of ORC.

As for documentation: I will (hopefully soon) start another thread to discuss how to describe parts of ORC better.

signature.asc
Lang Hames via llvm-dev的个人资料照片
Lang Hames via llvm-dev
2016年5月24日 05:05:37
收件人 Alex Denisov、LLVM Developers Mailing List、Larry Gritz
Hi Alex, Larry,

As for documentation: I will (hopefully soon) start another thread to discuss how to describe parts of ORC better.

Yep. ORC could definitely use better documentation, and the interface is more mature now (and less of a moving target) so it's a good time to do this.

At a high level I think we want two documents: An "ORC Overview/Manual" describing the design and components, and a tutorial series showing how to use them for a simple-but-real use case. I started work on the tutorial series over the weekend, and committed my work-in-progress first chapter today as r270487.

Any help (feedback, patches, etc.) will be most welcome, especially because some of the hurdles and gotchas will be more fresh in your mind than they are mine.

Thanks to both of you for asking about this - it's a good motivator to start working on it. :)

Cheers,
Lang.
这是一封来自 LLVM 开发者邮件列表（llvm-dev）的讨论帖，主题是关于 在 ORC JIT 中如何正确解析外部函数（External function resolution），并将其与旧的 MCJIT 进行了对比。

以下是该讨论的核心内容总结和中文解释：

问题背景 (Larry Gritz 提出)

*   用户: Larry Gritz 正在将他的 DSL（领域特定语言）从旧版 LLVM 升级，并尝试使用新的 ORC JIT API。
*   问题: 他在 IR 中声明了一个外部 C 函数 sqr（在宿主应用程序中定义），使用 MCJIT 时一切正常，但切换到 ORC JIT 后，程序报错："Program used external function 'sqr' which could not be resolved."（程序使用了无法解析的外部函数'sqr'）。
*   疑问: 他怀疑自己在 ORC JIT 的“魔法咒语”（即配置步骤）上出了错，因为他的代码逻辑看起来和教程很像。

他提供了一个最小可复现的 C++ 示例代码，清晰地展示了问题。

解决方案 (Lang Hames 回答)

LLVM JIT 专家 Lang Hames 指出了问题所在，并提供了三个关键的修复点：

1.  加载主程序符号 (LoadLibraryPermanently):
    *   在程序启动时，必须调用 llvm::sys::DynamicLibrary::LoadLibraryPermanently(nullptr)。
    *   原因: 这个调用会告诉 LLVM 的动态库系统去索引当前主程序（main executable）中的所有导出符号。没有这一步，getSymbolAddressInProcess 就无法找到 sqr 函数。

2.  Linux 平台的特殊处理 (链接器标志):
    *   在 Linux 上，默认情况下，主程序（可执行文件）中的符号不会被导出，即使你调用了 LoadLibraryPermanently 也没用。
    *   解决方案: 在链接你的程序时，需要加上 -Wl,--export-dynamic（或 -rdynamic）标志。这会让链接器将主程序的符号也放入动态符号表中，从而可以被 JIT 找到。
    *   注意: macOS 上不需要此步骤。

3.  macOS 平台的符号修饰 (Name Mangling):
    *   在 macOS (以及大多数 Unix-like 系统) 上，C 语言的符号在目标文件中会被加上一个下划线前缀（例如，myfunc 变成 _myfunc）。
    *   解决方案: 在通过 findSymbol 查找函数地址时，不能直接用 "myfunc"，而应该用 "myfunc" 的修饰后名称。最佳实践是使用 LLVM 提供的 Mangler 工具，根据 DataLayout 来正确地修饰符号名，以保证跨平台兼容性。

Lang 还贴心地附上了修复后的完整代码。

后续讨论

*   Windows 平台: Larry 询问 Windows 是否更复杂。Lang 回应说，Windows 上的主要坑是 COFF 格式对符号导出标志（SF_Exported）的支持不好，所以在调用 findSymbol 时需要传入 ExportedSymbolsOnly=false。他还提供了一个在 Windows 上使用 ORC 的项目（LLILC）作为参考。
*   文档与易用性: Alex Denisov 指出，LoadLibraryPermanently(nullptr) 这个关键步骤很难被发现，因为它隐藏在 EngineBuilder 内部，而不是 JIT 引擎本身的显式要求。他认为这要么应该被移到引擎内部自动处理，要么需要在文档中明确说明。
*   改进计划: Lang Hames 承认 ORC 的文档确实不足，并宣布他已经开始着手编写一个新的、更完善的 ORC 教程系列，欢迎大家贡献反馈和补丁。

核心要点总结

这个讨论揭示了从 MCJIT 迁移到 ORC JIT 时一个非常典型的陷阱：外部符号解析。

*   MCJIT 在内部自动处理了很多细节（比如调用 LoadLibraryPermanently），对用户更“友好”但也更“黑盒”。
*   ORC JIT 作为一个更模块化、更灵活的框架，将这些控制权交给了用户，因此用户需要显式地完成以下步骤：
    1.  调用 LoadLibraryPermanently(nullptr) 以使主程序符号可查。
    2.  （仅限 Linux）在链接时使用 -Wl,--export-dynamic 导出主程序符号。
    3.  使用 Mangler 来正确处理跨平台的符号名称修饰。
    4.  在符号解析器（LambdaResolver）中，先在 JIT 内部查找符号，找不到时再回退到 getSymbolAddressInProcess 去主程序中查找。

这次讨论也直接推动了 LLVM 社区对 ORC JIT 文档和入门体验的改进。
*********************************************************