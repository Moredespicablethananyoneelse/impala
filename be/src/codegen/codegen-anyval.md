/// 在代码生成（codegen）过程中用于处理 AnyVal 子类的类。
/// 代码生成函数应使用此包装器，而不是直接创建或操作 *Val 值（大多数情况下）。
/// 这是因为为了符合标准调用约定（例如 { i8, i32 } => i64），结构体类型必须被降级为整数类型。
/// 该类封装了每个 *Val 结构体对应的降级类型。
///
/// 从概念上讲，该类表示一个可变的 *Val，但其内部通过生成对 value_ 的 IR 指令来实现（由于 IR 使用 SSA，
/// 每条指令都会生成一个新的 Value*），然后将 value_ 更新为最新生成的 Value*。
/// 生成的指令执行与设置原始结构体字段等价的整数操作。
///
/// 降级类型（x86-64 ABI 下）：
/// TYPE_BOOLEAN/BooleanVal: i16
/// TYPE_TINYINT/TinyIntVal: i16
/// TYPE_SMALLINT/SmallIntVal: i32
/// TYPE_INT/IntVal: i64
/// TYPE_BIGINT/BigIntVal: { i8, i64 }
/// TYPE_FLOAT/FloatVal: i64
/// TYPE_DOUBLE/DoubleVal: { i8, double }
/// TYPE_STRING,TYPE_VARCHAR,TYPE_CHAR,TYPE_FIXED_UDA_INTERMEDIATE/StringVal: { i64, i8* }
/// TYPE_ARRAY/TYPE_MAP/CollectionVal: { i64, i8* }
/// TYPE_STRUCT/StructVal: { i64, i8* }
/// TYPE_TIMESTAMP/TimestampVal: { i64, i64 }
/// TYPE_DECIMAL/DecimalVal（未降级）：
/// %"struct.impala_udf::DecimalVal" { {i8}, [15 x i8], {i128} }
/// TYPE_DATE/DateVal: i64
///
/// TODO:
/// - 单元测试
*****************************************
请介绍下这个类的设计目的
CodegenAnyVal 是 Apache Impala（一个开源的 MPP SQL 查询引擎）中用于 LLVM 代码生成（codegen）阶段处理 Val 类型（如 IntVal, StringVal, DecimalVal 等）的一个关键封装类。其设计目的主要围绕以下几点：

1. 抽象 LLVM 中 AnyVal 类型的底层表示差异
Impala 的 UDF（用户自定义函数）接口使用一系列 C++ 结构体（如 BooleanVal, IntVal, StringVal 等），统称为 Val，它们都继承自 AnyVal 基类。这些结构体在 C++ 运行时有统一的语义（包含 is_null 标志和值字段），但在 LLVM IR 层面，为了符合平台 ABI（特别是 x86-64 和 AArch64 的调用约定），这些结构体需要被“降级”（lowered）为更简单的整数、指针或数组类型。

例如：
BooleanVal 在 C++ 中是 { bool is_null; bool val; }
在 x86-64 LLVM IR 中被 lowered 为 i16
在 AArch64 中可能被 lowered 为 i64

CodegenAnyVal 封装了这种 lowered 表示，并提供统一接口（如 GetVal(), SetIsNull()），使上层 codegen 逻辑无需关心底层 ABI 差异。

2. 统一操作接口，简化代码生成逻辑
在生成 LLVM IR 时，开发者不需要手动拼接位操作、结构体提取/插入等繁琐指令。CodegenAnyVal 提供了高层方法，例如：
GetIsNull() / SetIsNull()：读写 null 标志
GetVal() / SetVal()：读写实际值（对 int/float 等）
GetPtr() / GetLen()：处理字符串、数组等复杂类型
Eq(), Compare()：生成比较逻辑

这些方法内部会根据类型和平台（x86 vs ARM）自动选择正确的 IR 指令（如 extractvalue, trunc, zext, bitcast 等）。

3. 支持跨平台（x86-64 与 AArch64）
代码中大量使用 #ifndef __aarch64__ 条件编译，说明 CodegenAnyVal 显式处理了不同架构下 lowered 类型的差异：
x86-64 倾向于使用 struct { i8, T }
AArch64 则常使用 array [2 x i64] 或纯 i64

该类确保同一段 codegen 逻辑可在不同 CPU 架构上正确生成高效 IR。

4. 桥接 lowered 与 unlowered 表示
Lowered type：用于函数参数/返回值，符合 ABI（如 i64 代替 IntVal）
Unlowered type：保留原始 struct 名称（如 %struct.impala_udf::IntVal），用于调试、类型检查或调用某些需要原始类型的 helper 函数

CodegenAnyVal 提供 GetLoweredType() / GetUnloweredType() 以及对应的指针类型获取方法，并能通过 GetUnloweredPtr() 安全地在两者间转换（使用 bitcast）。

5. 支持复杂类型（String, Timestamp, Decimal, Struct, Collection）
除了基本类型，它还处理：
String/Collection：通过 { len, ptr } 表示
Timestamp：拆分为 date + time_of_day
Decimal：特殊处理 128 位值和 null 标志嵌套结构
Struct：递归处理子字段，支持嵌套结构体的 codegen

这使得整个表达式求值、UDF 调用、聚合等 codegen 流程能统一处理所有数据类型。

6. 与运行时无缝集成
CreateCall() 方法能正确处理返回 DecimalVal 的函数（因其过大，需通过 output 参数返回）
EqToNativePtr() 可比较 codegen 生成的 Val 与运行时原生类型（如 StringValue）
支持 canonical form（如 NaN 统一化），确保 hash/group by 正确性

总结
CodegenAnyVal 的核心设计目的是：在 LLVM 代码生成阶段，提供一个类型安全、平台无关、操作便捷的抽象层，用于统一处理 Impala 中所有 Val 类型的 lowered 表示，从而简化表达式编译、UDF 调用、比较、序列化等关键 codegen 逻辑。

它是 Impala 高性能查询引擎中连接 高级 SQL 语义 与 底层 LLVM IR 实现** 的重要桥梁。
**************************************************************
你提到“CreateCall() 方法能正确处理返回 DecimalVal 的函数（因其过大，需通过 output 参数返回）”，这样处理是LLVM的规定么


你的问题非常好！关于“CreateCall() 方法能正确处理返回 DecimalVal 的函数（因其过大，需通过 output 参数返回）”这一说法，并不是 LLVM 本身的硬性规定，而是 LLVM 的调用约定（calling convention）在特定目标平台（如 x86-64）下的实现细节所导致的行为。

下面详细解释：

1. LLVM IR 层面的抽象
在 LLVM IR 中，函数可以返回任意大小的结构体或类型。例如：

llvm
%DecimalVal = type { i64, i32 }

define %DecimalVal @foo() {
; ...
}

从 IR 的角度看，这是完全合法的——LLVM IR 允许函数直接返回结构体。

2. 但底层 ABI 有限制
然而，当 LLVM 将 IR 编译成目标机器码（如 x86-64）时，必须遵守该平台的 ABI（Application Binary Interface） 规范。

以 System V AMD64 ABI（Linux/macOS 等使用）为例：
如果返回值太大（通常 > 16 字节），或者包含无法通过寄存器高效传递的类型，
则 ABI 规定：调用者分配返回值的空间，并将该地址作为隐式第一个参数传给被调用函数。
被调用函数将结果写入该地址，而不是通过寄存器返回。

例如，一个 24 字节的 struct 在 x86-64 下就会触发这种“sret”（struct return）机制。

3. LLVM 如何处理？
LLVM 在代码生成阶段（CodeGen）会自动将 IR 中的“大结构体返回”转换为符合 ABI 的形式：
它会在函数签名中插入一个 sret（structure return）参数。
同时修改调用点，传入一个指向结果存储位置的指针。

这个过程对 IR 编写者是透明的，但在使用 LLVM C++ API（如 IRBuilder::CreateCall） 构建函数调用时，如果你手动构造了一个返回大结构体的函数类型，LLVM 的后端仍会按 ABI 自动处理。

不过，在某些情况下（比如你使用了不匹配的函数类型，或绕过了正常 IR 构建流程），就可能需要显式模拟 sret 行为——即把返回类型改为 void，并添加一个指向输出缓冲区的指针参数。

4. 回到你的问题
“CreateCall() 方法能正确处理返回 DecimalVal 的函数（因其过大，需通过 output 参数返回）”

这实际上意味着：
LLVM 的 CreateCall 在构建调用时，如果目标函数的返回类型是一个“大结构体”，
并且当前目标平台的 ABI 要求使用 sret，
那么 LLVM 后端会在 lowering 阶段自动将其转换为 sret 形式，无需你在 IR 中手动添加 output 参数。

✅ 所以：这不是 LLVM IR 的规定，而是目标平台 ABI 的要求，LLVM 自动帮你处理了。

但注意：
如果你在 IR 中手动定义函数为返回大结构体，那是合法的；
如果你手动定义函数为 void 并带 output 参数，那也是一种等效的手动实现方式（有时用于控制或兼容）。

5. 验证方法
你可以用 llc 编译一个返回大结构体的 .ll 文件，观察生成的汇编是否使用了 rdi（x86-64 下 sret 参数通常通过第一个整数参数寄存器传递）。

总结

问题 回答
------ ------
这是 LLVM 的规定吗？ ❌ 不是 LLVM IR 的规定
是谁规定的？ ✅ 目标平台的 ABI（如 System V ABI）
LLVM 会自动处理吗？ ✅ 是的，在代码生成阶段自动插入 sret
需要手动加 output 参数吗？ ❌ 通常不需要，除非你绕过正常 IR 构建流程

希望这能澄清你的疑问！如果你有具体的 DecimalVal 定义或平台环境，我可以进一步分析。
*****************************************************************
既然LLVM会自动处理：“在代码生成阶段自动插入 sret”。为什么apache impala还要在此处处理
这是一个非常深入且关键的问题！你已经准确抓住了核心矛盾点：
既然 LLVM 在代码生成阶段会自动处理大结构体返回（通过 sret），为什么 Apache Impala 还要在 IR 构建阶段就手动把返回值改为 output 参数？

答案是：Impala 的这种做法，是为了绕过 LLVM 的自动 sret 处理机制，从而获得更可控、更高效、更兼容的代码生成行为。

下面从几个角度详细解释原因：

1. LLVM 的 sret 自动转换发生在“后端”（CodeGen），但 Impala 需要在“前端”（IR 构建）就确定调用约定
LLVM 的自动 srect 插入是在 SelectionDAG / ISel 阶段（即从 LLVM IR 到机器码的 lowering 过程中）完成的。
然而，Impala 是一个 JIT 编译器，它在运行时动态生成函数，并且这些函数之间会互相调用（例如 UDF 调用内置函数，或表达式求值函数调用 Decimal 运算函数）。
如果 Impala 让 LLVM 自动处理 sret，那么：
它无法在 IR 层面知道某个函数是否会被转为 sret 形式；
它也无法在构建调用点（CreateCall）时提前知道是否需要传入额外的隐式指针参数；
这会导致 IR 构建与实际 ABI 不一致，可能引发崩溃或未定义行为。
💡 关键点：JIT 场景下，函数签名必须在 IR 构建时就与最终 ABI 一致，不能依赖后端“偷偷改写”。

2. Impala 使用统一的“output parameter”约定来简化函数接口

Impala 中大量使用如下模式：

cpp
void AddDecimal(const DecimalVal lhs, const DecimalVal rhs, DecimalVal result);

而不是：

cpp
DecimalVal AddDecimal(DecimalVal lhs, DecimalVal rhs);

这样做的好处包括：
避免拷贝大结构体：即使在 IR 层面，传递结构体也可能触发内存拷贝（尤其在优化未开启时）；
统一错误处理：DecimalVal 可能包含 is_null 标志，通过指针输出可以自然支持 null；
与 C++ 函数直接兼容：Impala 的很多内建函数是 C++ 编写的，它们天然使用 output 参数风格。如果 IR 也用同样签名，就可以直接 getFunction() 获取符号并调用，无需包装。
✅ 手动使用 output 参数，使得 C++ 函数 ↔ LLVM IR 函数 的 ABI 完全一致，避免 thunk 或适配层。

3. LLVM 的自动 sret 行为依赖于 DataLayout 和 TargetMachine 设置
LLVM 是否将某结构体视为“需 sret”取决于：
结构体大小；
成员对齐；
目标平台 ABI；
是否标记了 byval / sret 属性等。
如果 Impala 依赖自动 sret，就必须确保：
所有 IR 模块的 DataLayout 严格匹配目标平台；
所有函数类型在创建时都经过 ABI 查询（如使用 TargetLowering::CanLowerReturn）；
这会大大增加 IR 构建的复杂性。
🛠️ 更简单可靠的做法：显式使用 output 参数，彻底避开 ABI 判断逻辑。

4. 性能与可预测性
自动 sret 可能导致函数被重写为：
llvm
define void @foo(%DecimalVal sret %agg.result)

而调用点变成：
llvm
%result = alloca %DecimalVal
call void @foo(%DecimalVal* %result)
但如果 Impala 自己管理 %result 的分配（比如复用临时 buffer、放在栈上特定位置、甚至寄存器化），它可以做得比 LLVM 默认更优。
此外，某些优化（如 SROA、mem2reg）对 alloca + store 模式不如对显式指针参数敏感。

5. 历史与工程实践

Impala 早期设计就采用了“所有复杂类型通过指针传入/传出”的原则（类似 Google 的 C++ 风格指南）。这一约定贯穿整个表达式系统、UDF 接口、代码生成器。改变它代价巨大，而收益有限。

总结：为什么 Impala 要手动处理？

原因 说明
------ ------
✅ JIT 兼容性 必须在 IR 构建时就确定 ABI，不能依赖后端重写
✅ 与 C++ 函数 ABI 对齐 避免 thunk，直接调用 native 函数
✅ 简化 IR 生成逻辑 不必查询 ABI 规则或处理 sret 属性
✅ 性能可控 自主管理临时对象生命周期和内存布局
✅ 工程一致性 整个系统统一采用 output-parameter 模式
🔚 结论：
LLVM 的自动 sret 是为“静态编译”场景设计的，在 JIT 或高度控制的运行时（如 Impala、Spark SQL、Presto 等）中，显式建模 ABI 行为反而更安全、高效、可维护。

如果你查看 Impala 的源码（如 decimal-functions-ir.cc 或 codegen 相关模块），会发现它甚至会对 BooleanVal、StringVal 等类型也采用类似模式——这正是其设计哲学的体现。

******************************************************************
### `CodegenAnyVal` 类详尽介绍与实现解析

`CodegenAnyVal` 是 Apache Impala 代码生成（codegen）系统中一个**极其核心的辅助类**，它的设计目的是**在 LLVM IR 层面安全、高效地操作 Impala UDF（用户定义函数）接口中的 `*Val` 类型值**（如 `BooleanVal`、`IntVal`、`StringVal`、`TimestampVal`、`DecimalVal` 等）。

#### 1. 为什么需要 `CodegenAnyVal`？（背景与动机）

Impala 的 UDF 接口（`impala_udf`）使用一组结构体（如 `BooleanVal`、`StringVal`）来表示可能为 NULL 的标量值：
```cpp
struct BooleanVal {
  bool is_null;
  bool val;
};
struct StringVal {
  bool is_null;
  int len;
  uint8_t* ptr;
};
struct DoubleVal {
  bool is_null;
  double val;
};
```

这些结构体在 **x86-64 系统调用约定（ABI）** 中返回时会遇到问题：
- 返回值超过 2 个寄存器时，需要通过隐藏指针参数返回（sret）。
- 结构体返回可能被“lowered”（降级）为整数或多个寄存器。

为了兼容 ABI，Impala 在 codegen 时将这些 `*Val` 结构体**降级（lowered）为更简单的类型**（如 `i64`、`{i8, double}`、`{i64, i8*}`），但运行时代码仍然需要操作“逻辑上的字段”（`is_null`、`val`、`ptr`、`len`）。

`CodegenAnyVal` 正是为了**在 lowered 类型上模拟 unlowered 结构体的字段访问**而设计的。它封装了所有类型特异的位操作和提取/插入逻辑，让上层 codegen 代码（如 `TupleRowComparator`）可以统一、安全地读写 `*Val` 值。

#### 2. Lowered 类型映射（x86-64 vs AArch64）

| 类型               | x86-64 Lowered 类型                  | AArch64 Lowered 类型            | 说明 |
|--------------------|---------------------------------------|----------------------------------|------|
| BooleanVal         | i16                                   | i64                              | is_null 在最低位 |
| TinyIntVal         | i16                                   | i64                              |      |
| SmallIntVal        | i32                                   | i64                              |      |
| IntVal/DateVal     | i64                                   | i64                              |      |
| BigIntVal          | {i8, i64}                             | [2 x i64]                        |      |
| FloatVal           | i64                                   | i64                              | 值在高32位 |
| DoubleVal          | {i8, double}                          | [2 x i64]                        |      |
| StringVal/CollectionVal | {i64, i8*}                       | [2 x i64]                        | len 在高32位，is_null 在最低位 |
| TimestampVal       | {i64, i64}                            | [2 x i64]                        | date 高32位 + is_null 低位 |
| DecimalVal         | struct {i8, [15 x i8], i128}          | struct {i8, i128}                | 未 lowered |

AArch64 使用数组而非结构体，是为了兼容其更严格的 ABI。

#### 3. 核心成员与构造函数

```cpp
CodegenAnyVal(LlvmCodeGen* codegen, LlvmBuilder* builder, const ColumnType& type,
              llvm::Value* value = nullptr, const char* name = "");
```

- `value_`：当前持有的 lowered 值（SSA 值，每次修改生成新值）。
- 如果 `value == nullptr`，自动 alloca 并 load 一个零初始化值。
- 所有操作都基于当前 `value_` 生成新指令，并更新 `value_`（符合 LLVM SSA）。

#### 4. 关键操作详解

##### (1) `GetIsNull()` / `SetIsNull()`
- 提取最低字节（或第一个字段）作为 `bool`。
- 对于整数类型：`trunc` 到 `i1`。
- 对于结构体：`extractvalue` + `trunc`。
- 设置时：清零最低字节，再 `or` 上新值。

##### (2) `GetVal()` / `SetVal()`
- 对于整数/浮点：使用 `GetHighBits` / `SetHighBits` 操作高位。
  - `GetHighBits`：右移 + trunc（取出高半部分）。
  - `SetHighBits`：扩展 + 左移 + mask + or（设置高半部分）。
- 对于 BigInt/Double：`extractvalue` / `insertvalue`。
- Decimal：特殊索引到 i128 字段。

##### (3) StringVal / CollectionVal 专用
- `GetPtr()` / `SetPtr()`：提取/插入第二个字段（指针）。
- `GetLen()` / `SetLen()`：高32位操作。

##### (4) TimestampVal 专用
- `GetDate()` / `SetDate()`：高32位。
- `GetTimeOfDay()` / `SetTimeOfDay()`：低64位。

##### (5) `Compare(CodegenAnyVal* other)`
- 将两个值存到 alloca（得到 void*）。
- 构造常量 `ColumnType`。
- 调用预编译的 `RawValue::Compare`（通过 `IRFunction::RAW_VALUE_COMPARE` 获取）。

##### (6) `Eq()` / `EqToNativePtr()`
- 整数/浮点：直接 `icmp` / `fcmp`。
- String/Timestamp：调用预编译的专用相等函数（`CODEGEN_ANYVAL_STRING_VAL_EQ` 等）。

##### (7) `ConvertToCanonicalForm()`
- 处理浮点数的 NaN 和 -0.0：
  - 所有 NaN → 统一 bit pattern（quiet NaN）。
  - -0.0 → +0.0。
- 重要：分组（group by）时需要 NaN == NaN，-0 == +0。

#### 5. 静态工具函数

- `CreateCallWrapped`：封装函数调用，处理 DecimalVal 的特殊返回（sret）。
- `GetNullVal` / `GetNonNullVal`：生成常量 null / non-null 值。
- `GetLoweredType` / `GetUnloweredType`：类型映射。

#### 6. 与 `TupleRowComparator` 的协作（实际使用示例）

在 `CodegenLexicalCompare` 中：
```cpp
CodegenAnyVal lhs_value = CodegenAnyVal::CreateCallWrapped(...);
CodegenAnyVal rhs_value = CodegenAnyVal::CreateCallWrapped(...);

lhs_value.GetIsNull();           // 生成提取 is_null 指令
lhs_value.Compare(&rhs_value);   // 生成 RawValue::Compare 调用
```

所有复杂位操作都被 `CodegenAnyVal` 隐藏，上层代码只需写逻辑清晰的字段访问。

#### 7. 总结：`CodegenAnyVal` 的设计价值

| 优势                         | 说明 |
|------------------------------|------|
| **抽象 ABI 差异**            | 屏蔽 x86-64 和 AArch64 的 lowered 类型差异 |
| **类型安全**                 | 所有操作基于 `ColumnType`，编译期检查 |
| **性能极高**                 | 全 bit 操作 + 内联预编译函数 |
| **代码简洁**                 | 上层 codegen 代码像操作 C++ 结构体一样清晰 |
| **支持复杂类型**             | String、Timestamp、Decimal、Collection 完整支持 |

`CodegenAnyVal` 是 Impala codegen 系统中“**最聪明的一个封装**”，它让生成的高性能排序、聚合、join 等算子代码既**正确**又**可读**，是 Impala 能达到接近手写汇编性能的关键基石之一。
************************************************************88
以下是针对您的问题（理解 LLVM 中的 "lowered 类型" 和 "unlowered 结构体"）的背景知识推荐。这些概念主要源于 LLVM IR（Intermediate Representation）的类型系统和 ABI（Application Binary Interface）调用约定。在 LLVM 中，结构体类型（如 C++ 中的 struct）在生成机器码时往往需要“lowering”（降级/扁平化）为更简单的类型（如整数或数组），以符合目标平台的调用约定（例如 x86-64 或 AArch64 如何传递/返回结构体）。Unlowered 指原始的结构体表示，lowered 指优化后的扁平形式（如将 {bool, int} 合并为 i64 以减少参数传递开销）。

我推荐的资料分为**权威官方文档**（适合深入学习）和**通俗解释**（适合入门）。这些基于可靠来源（如 LLVM 官方和社区讨论），并结合您的需求（通俗易懂）。如果需要，我可以进一步解释这些资料的核心点。

### 1. **权威推荐：LLVM Language Reference Manual**
   - **链接**：https://llvm.org/docs/LangRef.html
   - **为什么推荐**：这是 LLVM 官方文档，最权威的来源。重点看“Structure Types” 和 “Calling Conventions” 部分。它解释了 LLVM IR 中的结构类型（struct types）如何被唯一化（uniqued），以及在后端（backend）如何 lowering 为目标机器的 ABI 兼容形式。文档用伪代码和 IR 示例说明，适合有编程基础的人。
   - **通俗度**：中等（技术性强，但有清晰示例）。建议从“Identified Struct Types vs Literal Struct Types” 开始读，理解 uniqued（唯一化的类型）和如何在 IR 中表示/操作结构体。
   - **适用性**：直接对应 Impala codegen 中的 lowered/unlowered（Impala 基于 LLVM IR 生成代码）。

### 2. **通俗推荐：Stack Overflow - What are they talking about when they talk about 'uniqued types' and 'types' in LLVM?**
   - **链接**：https://stackoverflow.com/questions/29756212/what-are-they-talking-about-when-they-talk-about-uniqued-types-and-types-in-l
   - **为什么推荐**：这是一个简短、通俗的问答，解释了 LLVM 中的“uniqued types”（唯一化类型，即相同结构的类型被视为同一个）和结构体在 IR 中的表示。回答者用简单例子说明 lowering 的必要性（如结构体在函数返回时的扁平化）。非常适合初学者快速入门。
   - **通俗度**：高（对话式，少代码）。
   - **适用性**：直接解答您的“unlowered 结构体”疑问（unlowered 类似 uniqued 的原始 struct）。

### 3. **权威 + 通俗：LLVM Discourse - [RFC] An ABI lowering library for LLVM**
   - **链接**：https://discourse.llvm.org/t/rfc-an-abi-lowering-library-for-llvm/84495
   - **为什么推荐**：这是 LLVM 社区的提案讨论，解释了 ABI lowering 的背景——为什么需要将高层次类型（如结构体）降级为低层次表示（如整数/指针），以抽象不同平台的差异。结合 Impala 的场景（x86-64 ABI），它讨论了函数参数/返回值的 lowering。提案中有代码示例和动机分析。
   - **通俗度**：中等（有技术细节，但动机部分易懂）。
   - **适用性**：完美匹配 lowered 类型（ABI lowering 的产物），帮助理解 Impala 为什么在 codegen 时扁平化 `*Val` 结构体。

### 4. **通俗推荐：Mapping High Level Constructs to LLVM IR - Structures**
   - **链接**：https://mapping-high-level-constructs-to-llvm-ir.readthedocs.io/en/latest/basic-constructs/structures.html
   - **为什么推荐**：这是一个开源书籍/文档，专注于从高层次语言（如 C++）映射到 LLVM IR。Structures 章节用简单示例解释了结构体在 IR 中的表示、incomplete types（不完整类型）、lowering 到后端的过程。包括 IR 代码片段，易于对比 Impala 的实现。
   - **通俗度**：高（逐步教程式）。
   - **适用性**：直接覆盖 lowered/unlowered 的转换过程，适合学习背景知识。

### 学习建议
- **入门路径**：先读 Stack Overflow（5 分钟），再看 Mapping High Level Constructs（10-15 分钟），最后深入 LLVM 官方文档。
- **为什么这些资料**：它们权威（LLVM 官方/社区）、通俗（示例丰富），避免了过于学术的论文。Impala 的 codegen 直接基于这些概念，如果您有具体疑问（如 x86-64 ABI），我可以进一步解释或推荐子章节。
- **如果不够**：如果需要视频教程，搜索 YouTube 的“LLVM IR Tutorial” 或“ABI and Calling Conventions in LLVM” 会找到更多资源。

如果这些资料帮助您理解了，再告诉我，我可以继续解答上文中的具体代码！
*********************************************************88
https://discourse.llvm.org/t/rfc-an-abi-lowering-library-for-llvm/84495?referrer=grok.com
请翻译成中文：A key premise of LLVM is that it provides an abstraction over different targets. A frontend just needs to worry about generating LLVM IR, and then LLVM takes care of everything else, generating highly efficient code for a wide variety of targets.

The main area where this premise currently breaks down is target-specific ABI differences, especially concerning the call ABI. Part of the call ABI is handled by LLVM, but a large part of it is a frontend responsibility. There are many different ABIs, with many complex and subtle rules. Failing to implement them correctly will result in miscompilations.

Every LLVM-based frontend that wants to expose a C FFI interface currently has to manually implement these ABI rules. Clang implements these in the CodeGen/Targets.

We regularly get questions on how to support C FFI on Discord, and for some reason nobody is ever happy about “you have to re-implement these ten thousand lines of Clang code” being the answer. :slight_smile:

Why doesn’t LLVM “just” handle this?
LLVM handles part of the call ABI lowering, such as the assignment of scalar arguments/returns to registers. Why can’t LLVM simply do the right thing for all argument types?

The primary reason for this is that the LLVM type system is not expressive enough to make all ABI decisions. Here are a few examples:

LLVM does not have a representation for unions at all.
LLVM does not have type-level alignment annotations. For example, two otherwise identical structs, but one with an explicit alignment attribute that matches its default alignment, do not have the same ABI. In fact, alignment has to be computed in at least three different ways to satisfy various ABI rules.
Do you think __int128 and _BitInt(128) have the same ABI? Think again.
Why don’t we extend LLVM’s type system to handle these cases? LLVM generally tries to omit any type information that is not semantically relevant. LLVM will represent both int and unsigned as an i32. This kind of inherent canonicalization is important for optimization purposes.

While directly extending the type system would be a bad idea, it would in principle be possible to convey additional information using attributes/metadata at call-sites and declarations only, similar to how we have the zeroext and signext attributes to distinguish unsigned/signed integers for ABI purposes.

Doing this in full generality, providing all the information necessary for struct and union passing, would be significantly harder though, and essentially introduce a second shadow type system into LLVM IR.

An additional consideration here is that it is beneficial for optimization purposes if cases where large structures need to be passed in memory, are explicitly represented as such in LLVM IR. For example, this allows optimizing away redundant copies of the memory, etc.

LLVM’s IR design is generally very hostile towards working with aggregate SSA values, and our historical trend is to reduce reliance on struct types in IR. Making these first-class citizens would be a substantial shift in optimization philosophy, which goes far beyond ABI questions.

Proposal
The proposal is to introduce an LLVM ABI lowering library (LLVMABI), which provides information to frontends on how to correctly produce LLVM IR for a specific target. The initial focus of the library would be on call ABI lowering, as this is the hardest part. It can be extended to handle other ABI aspects as well.

The high level sketch of how this would look like is:

The library will have its own type system, which is independent of both the LLVM IR type system and the Clang type system. The types will support encode exactly as much information as is necessary for correct ABI lowering.
The library will provide per-target implementations of ABIInfo, extracting what Clang currently does.
The main result of ABI classification will be something like ABIArgInfo, which specifies whether the argument is passed directly, indirectly, etc.
The frontend is then responsible for generating LLVM IR based on the ABIArgInfo.
Some notes:

Clang will be switched to use the new ABI lowering library. I think it’s very important that Clang makes use of it, not just 3rd-party frontends, otherwise we’ll certainly get divergences.
Yes, this does mean that Clang will have to lower to an additional type system. My hope is that this will not add a lot of additional overhead if it is cached, but that’s one of the things that remains to be seen.
While the motivation here is purely about the C ABI, I think it will be unavoidable to also support the parts of the C++ ABI that relate to the calling convention. The C++ ABI is a minor modification of the C ABI, and I don’t think they can be usefully separated.
Layering: I think this library could be implemented completely independently of IR, depending only on Support. But having it depend on IR would probably make it more useful, as we could also provide LLVM IR types where relevant.
Implementation
I’d like to offer creating a prototype for this as a GSoC project. At this point, I’m mainly looking for some feedback on the general direction, and any insights people familiar with ABI lowering may have. The details of the design will have to be ironed out later.

There is an old llvm-abi project, also discussed here, which started implementing this concept out-of-tree. It’s many years out of date now and only support x86, but may serve as inspiration. I think it’s important that the ABI lowering library is in-tree and used by Clang to guarantee continued maintenance.

An alternative approach to the call ABI problem has recently been explored in Ideas about C calling convention lowering to LLVM IR. The approach there is to have the frontend attach additional ABI classification metadata, that allows LLVM to perform the ABI lowering.
LLVM 的一个核心理念是它提供了针对不同目标的抽象。前端只需专注于生成 LLVM IR，然后 LLVM 会负责处理其他所有事项，为广泛多样的目标生成高效的代码。

目前这一理念失效的主要领域是目标特定的 ABI 差异，尤其是关于调用 ABI 的部分。调用 ABI 的一部分由 LLVM 处理，但很大一部分是前端的责任。存在许多不同的 ABI，它们有许多复杂而细微的规则。如果未能正确实现这些规则，将导致错误的编译结果。

目前，任何想要暴露 C FFI 接口的基于 LLVM 的前端都不得不手动实现这些 ABI 规则。Clang 在 CodeGen/Targets 中实现了这些规则。

我们在 Discord 上经常收到关于如何支持 C FFI 的问题，出于某种原因，对于 "你必须重新实现 Clang 中那上万行代码" 这个答案，似乎从来没有人感到高兴。😊

**为什么 LLVM 不"直接"处理这个问题？**
LLVM 确实处理了一部分调用 ABI 的 lowering，例如将标量参数/返回值分配到寄存器。为什么 LLVM 不能简单地针对所有参数类型都做正确的事呢？

这主要是因为 LLVM 的类型系统表达能力不足，无法做出所有的 ABI 决策。这里有几个例子：

*   **联合体**：LLVM 根本没有联合体的表示方式。
*   **类型级对齐注解**：例如，两个结构体在其他方面相同，但一个具有与其默认对齐方式匹配的显式对齐属性，它们的 ABI 并不相同。实际上，为了满足各种 ABI 规则，对齐方式至少需要以三种不同的方式计算。
*   **`__int128` 和 `_BitInt(128)`**：你认为它们具有相同的 ABI 吗？再想想。

**为什么我们不扩展 LLVM 的类型系统来处理这些情况？**
LLVM 通常倾向于省略在语义上不相关的任何类型信息。LLVM 会将 `int` 和 `unsigned` 都表示为 `i32`。这种内在的规范化对于优化目的很重要。

虽然直接扩展类型系统不是一个好主意，但原则上，可以仅在调用点和声明处使用属性/元数据来传递附加信息，类似于我们使用 `zeroext` 和 `signext` 属性来区分无符号/有符号整数以实现 ABI 目的。

然而，要全面做到这一点，为结构体和联合体传递提供所有必要的信息，将会困难得多，并且实质上相当于在 LLVM IR 中引入了第二个影子类型系统。

这里还有一个额外的考虑因素：如果大型结构体需要在内存中传递的情况能在 LLVM IR 中明确表示为这样，对于优化目的来说是有益的。例如，这允许优化掉冗余的内存拷贝等。

LLVM 的 IR 设计通常对处理聚合 SSA 值非常不友好，我们的历史趋势是减少对 IR 中结构体类型的依赖。使这些成为一等公民将是优化理念的重大转变，其影响远超 ABI 问题。

**提案**
该提案是引入一个 LLVM ABI lowering 库（LLVMABI），它为前端提供如何为特定目标正确生成 LLVM IR 的信息。该库的初始重点将是调用 ABI lowering，因为这是最困难的部分。它也可以扩展以处理其他 ABI 方面。

其高级别框架如下：

1.  该库将拥有其独立的类型系统，既不依赖于 LLVM IR 类型系统，也不依赖于 Clang 类型系统。这些类型将仅编码为正确进行 ABI lowering 所必需的信息。
2.  该库将为每个目标提供 ABIInfo 的实现，提取 Clang 目前所做的内容。
3.  ABI 分类的主要结果将是类似 `ABIArgInfo` 的东西，它指定参数是直接传递、间接传递等。
4.  前端随后负责基于 `ABIArgInfo` 生成 LLVM IR。

**一些说明：**

*   **Clang 将切换为使用新的 ABI lowering 库**。我认为至关重要的是 Clang 也要使用它，而不仅仅是第三方前端，否则我们肯定会面临分歧。
*   是的，这意味着 Clang 将不得不降低到额外的类型系统。我希望如果进行缓存，这不会增加太多额外开销，但这是有待观察的事情之一。
*   虽然这里的动机纯粹是关于 C ABI，但我认为不可避免地也需要支持与调用约定相关的 C++ ABI 部分。C++ ABI 是 C ABI 的一个微小修改，我认为它们无法有效地分离。
*   **分层**：我认为这个库可以完全独立于 IR 实现，仅依赖于 Support 库。但让它依赖于 IR 可能会使其更有用，因为我们也可以在相关时提供 LLVM IR 类型。

**实现**
我想提议将此作为一个 GSoC 项目来创建原型。目前，我主要寻求对总体方向的反馈，以及熟悉 ABI lowering 的人可能有的见解。设计细节将需要稍后敲定。

有一个旧的 llvm-abi 项目（此处也有讨论），它曾开始实现这个概念（树外实现）。它现在已过时多年，并且仅支持 x86，但或许能提供一些灵感。我认为关键在于 ABI lowering 库必须在树内，并且被 Clang 使用，以保证持续的维护。

最近，在 *关于 C 调用约定 lowering 到 LLVM IR 的想法* 一文中探索了解决调用 ABI 问题的另一种方法。该方法建议让前端附加额外的 ABI 分类元数据，从而使 LLVM 能够执行 ABI lowering。
****************************************************************
https://discourse.llvm.org/t/ideas-about-c-calling-convention-lowering-to-llvm-ir/83126
请翻译成中文：I’ve had some ideas about function-call ABIs in my head, but have never gotten time to make a full design for them. Since the issue came up again in the devconf keynote, I thought I’d at least write down some of the thoughts. A lot more work would be required to move this beyond “rough idea” stage, and I probably won’t find the time to do so.

Background
The function-call ABI lowering/support in LLVM IR leaves much to be desired. This has been a known issue for a long time.

It’s currently quite difficult for a frontend to craft LLVM IR which properly adhere to the platform-specific C ABI calling convention. Even something relatively simple can be compiled into vastly different IR on different targets. E.g.:

struct TwoDouble {
  double a, b;
};

TwoDouble f(TwoDouble x) { ... }
Ignoring the return value for a minute, the IR function might have arguments like:

@f(%struct.TwoDouble %x)
@f(double %x.a, double %x.b)
@f([2 x double] %x)
@f([2 x i64] %x)
@f(ptr byval(%struct.TwoDouble) %x.addr)
…and probably others.
The return value may be similarly diverse. It might be like:

void @f(ptr sret(%struct.TwoDouble) %result, ...)
%struct.TwoDouble @f(...)
<2 x double> @f(...)
…or others.
Frontends need to deal with all of this. And worse, it’s all undocumented — the LLVM IR function ABI is effectively a secret handshake with Clang, and the only way to really know what IR you should generate for a particular target is to see what Clang emits, and do the same.

Creation of a new ABI lowering library has been discussed as a solution. This library could take the C types as input, and output the correct LLVM IR function declarations/calls in the current IR format. It seems feasible, but I don’t think this would be really satisfying: every frontend already has its own type representation. Such an ABI library would effectively require creation of a fully-duplicate copy of the types. Trying to narrow the library API to only require passing in “interesting” parts of the C type information seems to me unlikely to succeed, because the platform psABIs may use just about any property of the C type system.

One could also say that we should put all of the handling into LLVM: the frontend always emits %struct.TwoDouble @f(%struct.TwoDouble %x), and LLVM handles everything else. I think this would be nice, but is isn’t entirely possible: the ABI is based on C type information, and not all of the C type info is reflected into LLVM-IR types. Unions can be a particular headache, and there are also ABIs which treat _Complex double vs struct {double real, imag;}; differently, despite both being represented as {double, double} in IR. Thus, we necessarily require some additional information beyond that which is available in the LLVM type system.

Ideal
What I would like is for the C-to-LLVM-IR function lowering to be consistent, understandable, and simple, so that frontends can have as easy a job as possible. I would therefore like to see a scheme which is as close to that “naive” version as feasible. An LLVM-IR function’s signature should be the obvious result of lowering each argument and the return value. Each C function argument should result in one LLVM-IR function argument, and the type of each argument be the same as would be generated for any non-ABI context. Given such a function signature, LLVM should able to generate good code (it would not, today), even if not ABI-compatible.

On top of that “naive” lowering, the frontend could emit additional ABI-related metadata for each argument, to allow generation of C ABI-compliant functions (only additional metadata! no function-signature changes!). The metadata required should be clearly derived from each platform’s psABI document, and be context-free: the same C type should generate the same ABI metadata regardless of how many (or what kind of) previous arguments were given. This ABI metadata may be explicitly specific to a particular calling-convention/ABI’s required data, rather than trying to shoehorn information into a limited pseudo-generic vocabulary.

This means frontends will still require target-specific code, in order to generate correct ABI metadata. But, it can be simpler: the IR function signature/number of arguments no longer differs between ABIs, and if the classification metadata is a target-specific description of the required ABI attributes, it can be significantly easier to understand (and to understand how to emit), than the current state (where we translate into pseudo-generic IR concepts with no obvious connection to the platform ABI documents).

For example, the x86-64 psABI says to classify each 8-byte piece of an argument into INTEGER, SSE, SSEUP, X87, X87UP, COMPLEX_X87, or MEMORY, via a series of rules. I would have the frontend be responsible for this classification. It would emit exactly that classification, for each argument, into the IR. The backend uses that classification to correctly choose the appropriate memory/register locations.

Problem: SSA values vs memory
Maybe the above sounds nice, but there is a core issue here: LLVM IR is not able to efficiently handle the inherent ambiguity of location for IR function-arguments/returns. Inside the body of a function, an argument might be used as a value, or might be used by address. The value might have been passed in a register, or might have been on the stack (or sometimes in both places!). If we want to abstract the ABI choice away, we must be able to optimally handle all combinations of these options.

Today, if the platform ABI calls for passing an argument on the stack, the frontend can choose between the obvious “by value” representation, e.g. (i32 %x) or using a “by byval ptr” representation, e.g. (ptr byval(i32) %x). For a value you know is passed on the stack, both of these may result in identical machine-instruction-level ABI, despite looking entirely different at the IR-level ABI. But, using the former can result in non-optimal code. If it turns out we need the address, we need to create a temporary alloca and store the value into it – but the value “%x” was already on the stack, created by an “invisible” load. We should’ve just used a pointer to the existing stack slot. (Some obvious such inefficiencies are eliminated already via SelectionDAG’s findArgumentCopyElisionCandidates, but that can’t always trigger.).

But, to emit a “ptr byval” argument, the frontend needs to know that the ABI is going to pass on the stack.

It would be great to fix this issue. Perhaps we can introduce explicit support for the notion of a function-argument value’s stack-address? Create a new intrinsic like llvm.arg_address(%x), and let frontends emit that if they want the address of an argument value? With such an operation, the backend could decide either to expose an existing address directly (if the argument was passed on the stack) or create an alloca and store the value into it (if the argument was passed in a register). This allows isolating the frontend from the decision.

The situation for return values is similar. If we want to allow a frontend to emit %struct.TwoDouble @f() even for an ABI which returns the value by passing a hidden pointer, how may we do so in a way that preserves efficiency – including returning a value back through multiple layers of functions without modification?

This direction feels feasible to me, but does need a lot more investigation…

Problem: IR compatibility
Any change we make in how frontends lower a C function to LLVM IR would necessarily result in the same machine-level ABI, but it will result in functions having drastically different IR-level ABIs – different numbers and types of arguments and return values.

I think it’d not really be possible to bitcode-upgrade the current function signatures to the new ones, since there wouldn’t be a 1:1 correspondence. So, how to deal with the compatibility? Perhaps it’s fine if we simply avoid analyzing across the boundary between old and new kinds of functions.
关于函数调用的ABI，我脑海中一直有些想法，但从未有时间进行完整的设计。既然这个问题在开发大会的主题演讲中又被提起，我想至少应该把这些思考记录下来。要将这些想法从"粗略构思"阶段推进下去，还需要大量工作，而我可能找不到时间来做这些。

**背景**
LLVM IR中函数调用ABI的降低/支持方面存在很多不足。这是一个长期以来众所周知的问题。

目前，前端要构建完全符合平台特定C ABI调用约定的LLVM IR相当困难。即使是相对简单的东西，在不同目标平台上也可能被编译成完全不同的IR。例如：

```c
struct TwoDouble {
  double a, b;
};

TwoDouble f(TwoDouble x) { ... }
```

暂时忽略返回值，IR函数的参数可能是这样的：
- `@f(%struct.TwoDouble %x)`
- `@f(double %x.a, double %x.b)`
- `@f([2 x double] %x)`
- `@f([2 x i64] %x)`
- `@f(ptr byval(%struct.TwoDouble) %x.addr)`
- 可能还有其他形式

返回值可能同样多样化，可能是：
- `void @f(ptr sret(%struct.TwoDouble) %result, ...)`
- `%struct.TwoDouble @f(...)`
- `<2 x double> @f(...)`
- 或其他形式

前端需要处理所有这些情况。更糟的是，这些都是没有文档记录的——LLVM IR函数ABI实际上是Clang的一个秘密握手协议，真正了解特定目标应该生成什么IR的唯一方法是查看Clang生成了什么，然后照做。

**已有讨论**
创建一个新的ABI降低库已被讨论作为一个解决方案。这个库可以接受C类型作为输入，并输出当前IR格式的正确LLVM IR函数声明/调用。这似乎可行，但我不认为这真的令人满意：每个前端都已经有自己的类型表示。这样的ABI库实际上需要创建类型的一个完全重复副本。试图将库API缩小到只要求传入C类型信息的"有趣"部分，在我看来不太可能成功，因为平台psABI可能使用C类型系统的几乎任何属性。

人们也可能说我们应该把所有处理都放到LLVM中：前端总是生成`%struct.TwoDouble @f(%struct.TwoDouble %x)`，然后LLVM处理其他所有事情。我认为这样很好，但并非完全可行：ABI是基于C类型信息的，但并非所有C类型信息都反映到LLVM-IR类型中。联合体可能特别令人头疼，还有一些ABI区别对待`_Complex double`和`struct {double real, imag;}`，尽管两者在IR中都表示为`{double, double}`。因此，我们必然需要一些超出LLVM类型系统可用信息的额外信息。

**理想情况**
我希望C到LLVM-IR的函数降低能够一致、易懂且简单，以便前端的工作尽可能轻松。因此，我希望看到一个尽可能接近那个"朴素"版本的方案。LLVM-IR函数的签名应该是降低每个参数和返回值的明显结果。每个C函数参数应该产生一个LLVM-IR函数参数，每个参数的类型应该与任何非ABI上下文中生成的类型相同。给定这样的函数签名，LLVM应该能够生成良好的代码（虽然目前不能），即使不是ABI兼容的。

在那个"朴素"降低之上，前端可以为每个参数发出额外的ABI相关元数据，以允许生成符合C ABI的函数（仅仅是额外的元数据！没有函数签名变化！）。所需的元数据应该清晰地从每个平台的psABI文档中派生，并且是无上下文的：无论之前给出了多少（或什么样的）参数，相同的C类型应该生成相同的ABI元数据。这些ABI元数据可能明确特定于特定调用约定/ABI所需的数据，而不是试图将信息塞入有限的伪通用词汇表中。

这意味着前端仍然需要特定目标的代码，以生成正确的ABI元数据。但是，它可以更简单：IR函数签名/参数数量不再因ABI而异，如果分类元数据是所需ABI属性的目标特定描述，那么理解（以及理解如何发出）比当前状态（我们将其转换为与平台ABI文档没有明显联系的伪通用IR概念）要容易得多。

例如，x86-64 psABI说通过一系列规则将参数的每个8字节片段分类为INTEGER、SSE、SSEUP、X87、X87UP、COMPLEX_X87或MEMORY。我将让前端负责这个分类。它会为每个参数在IR中准确发出这个分类。后端使用该分类正确选择适当的内存/寄存器位置。

**问题：SSA值与内存**
也许上述听起来不错，但这里存在一个核心问题：LLVM IR无法高效处理IR函数参数/返回值位置固有的模糊性。在函数体内，参数可能被用作值，也可能被用作地址。值可能通过寄存器传递，也可能在栈上（或有时同时在两个地方！）。如果我们想要抽象化ABI选择，我们必须能够最优地处理所有这些选项的组合。

目前，如果平台ABI要求在栈上传递参数，前端可以在明显的"按值"表示（如`(i32 %x)`）和使用"byval指针"表示（如`(ptr byval(i32) %x)`）之间选择。对于一个你知道在栈上传递的值，这两种方式可能导致相同的机器指令级ABI，尽管在IR级ABI上看起来完全不同。但是，使用前者可能导致非最优代码。如果事实证明我们需要地址，我们需要创建一个临时的alloca并将值存储进去——但值"`%x`"已经通过一个"不可见"的加载在栈上了。我们本应该只使用现有栈槽的指针。（一些明显的这种低效已经通过SelectionDAG的`findArgumentCopyElisionCandidates`消除了，但这并不总是能触发。）

但是，要发出"`ptr byval`"参数，前端需要知道ABI将在栈上传递。

修复这个问题会很好。也许我们可以显式支持函数参数值的栈地址概念？创建一个新的内部函数如`llvm.arg_address(%x)`，让前端在需要参数值的地址时发出这个？有了这样的操作，后端可以决定直接暴露现有地址（如果参数在栈上传递）或创建alloca并将值存储进去（如果参数在寄存器中传递）。这可以使前端与决策隔离。

返回值的情况类似。如果我们想要允许前端发出`%struct.TwoDouble @f()`，即使对于通过传递隐藏指针返回值的ABI，我们如何以一种保持效率的方式实现——包括在不修改的情况下将值返回通过多个函数层？

这个方向对我来说似乎可行，但确实需要更多调查……

**问题：IR兼容性**
我们在前端如何将C函数降低到LLVM IR方面所做的任何改变，必然导致相同的机器级ABI，但会导致函数具有完全不同的IR级ABI——不同数量和类型的参数和返回值。

我认为实际上不可能将当前函数签名位码升级到新的签名，因为不会有1:1对应关系。那么，如何处理兼容性？也许如果我们简单地避免分析新旧类型函数之间的边界，那就没问题了。
*****************************************************************
[AnyVal和CodegenAnyVal的区别和联系](../../src/udf/udf.md)
*****************************************************************
