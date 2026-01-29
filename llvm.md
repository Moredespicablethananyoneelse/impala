
一共只有三个ToIR函数：
```cpp
llvm::Constant* NullIndicatorOffset::ToIR(LlvmCodeGen* codegen) const {   // 将 C++ 层面的 NullIndicatorOffset 结构体实例，转换为 LLVM IR 层面的常量结构体（便于 LLVM 后续进行常量传播等优化） 返回常量，便于常量传播
  llvm::StructType* null_indicator_offset_type =
      codegen->GetStructType<NullIndicatorOffset>();
  // Populate padding at end of struct with zeroes.
  llvm::ConstantAggregateZero* zeroes =
      llvm::ConstantAggregateZero::get(null_indicator_offset_type);
  return llvm::ConstantStruct::get(null_indicator_offset_type,
      {codegen->GetI32Constant(byte_offset),
          codegen->GetI8Constant(bit_mask),
          zeroes->getStructElement(2)});
}

llvm::Constant* SlotOffsets::ToIR(LlvmCodeGen* codegen) const {
  return llvm::ConstantStruct::get(
      codegen->GetStructType<SlotOffsets>(),
      {null_indicator_offset.ToIR(codegen),
          codegen->GetI32Constant(tuple_offset)});
}



llvm::ConstantStruct* ColumnType::ToIR(LlvmCodeGen* codegen) const {
  // ColumnType = { i32, i8, i32, i32, i32, <vector>, <vector>, <vector> }
  llvm::StructType* column_type_type = codegen->GetStructType<ColumnType>();

  DCHECK_EQ(sizeof(type), sizeof(int32_t));
  llvm::Constant* type_field = codegen->GetI32Constant(type);

  DCHECK_EQ(sizeof(len), sizeof(int32_t));
  llvm::Constant* len_field = codegen->GetI32Constant(len);
  DCHECK_EQ(sizeof(precision), sizeof(int32_t));
  llvm::Constant* precision_field = codegen->GetI32Constant(precision);
  DCHECK_EQ(sizeof(scale), sizeof(int32_t));
  llvm::Constant* scale_field = codegen->GetI32Constant(scale);

  // Create empty 'children', 'field_names' and 'field_ids' vectors
  DCHECK(children.empty()) << "Nested types NYI";
  DCHECK(field_names.empty()) << "Nested types NYI";
  DCHECK(field_ids.empty()) << "Nested types NYI";
  llvm::Constant* children_field =
      llvm::Constant::getNullValue(column_type_type->getElementType(4));
  llvm::Constant* field_names_field =
      llvm::Constant::getNullValue(column_type_type->getElementType(5));
  llvm::Constant* field_ids_field =
      llvm::Constant::getNullValue(column_type_type->getElementType(6));

  DCHECK_EQ(sizeof(is_binary_), sizeof(uint8_t));
  llvm::Constant* is_binary_field = codegen->GetI8Constant(is_binary_);

  llvm::Constant* padding =
      llvm::Constant::getNullValue(column_type_type->getElementType(8));

  return llvm::cast<llvm::ConstantStruct>(
      llvm::ConstantStruct::get(column_type_type, type_field, len_field, precision_field,
        scale_field, children_field, field_names_field, field_ids_field,
        is_binary_field, padding));
}
```


为什么需要将 C++ 层面的 NullIndicatorOffset 结构体实例，转换为 LLVM IR 层面的常量结构体？他们难道不是一样的吗？

你想弄明白为什么明明都是表示「空值指示器偏移信息」的结构体，却要把 C++ 层面的 `NullIndicatorOffset` 实例转换成 LLVM IR 层面的常量结构体，核心疑问是“它们难道不是一样的吗？”——答案是**完全不一样**，两者分属编译器的不同阶段、服务于不同目标，转换的核心目的是让编译器能理解、优化并生成最终的机器码。

下面我用通俗的方式拆解「为什么必须转换」，以及「两者的核心差异」：

### 一、先搞懂：C++ 结构体 vs LLVM IR 常量结构体，根本不是一回事
可以用一个比喻理解：
- **C++ 结构体实例**：是你用“人类语言（C++）”写的「数据描述」，只存在于**编译器的前端处理阶段**（比如语法分析、语义分析），是程序员能看懂的抽象概念；
- **LLVM IR 常量结构体**：是编译器把“人类语言”翻译成的「编译器内部语言（IR）」，存在于**编译器的中端优化 + 后端代码生成阶段**，是 LLVM 优化器/代码生成器能看懂的标准化格式。

两者的核心差异可以用表格清晰对比：

| 维度                | C++ 层面的 `NullIndicatorOffset` 结构体       | LLVM IR 层面的常量结构体（`llvm::ConstantStruct`） |
|---------------------|-----------------------------------------------|---------------------------------------------------|
| 存在阶段            | 编译前端（处理 C++ 代码时）                   | 编译中端/后端（优化、生成机器码时）               |
| 受众                | 程序员、C++ 编译器前端                        | LLVM 优化器、目标机器码生成器                     |
| 核心属性            | 面向“运行时内存”，有语言特有的 padding/对齐    | 面向“编译期优化”，是标准化的常量，可被 LLVM 分析  |
| 可操作性            | 只能在 C++ 代码逻辑中访问（如 `obj.byte_offset`） | 可被 LLVM 优化（如常量传播、死代码消除）           |

### 二、为什么必须做这个转换？（核心原因）
转换的本质是“把高级语言的抽象数据，翻译成编译器能处理的中间表示”，具体有 4 个关键目的：

#### 1. 让 LLVM 优化器能“看懂并优化”这个值
`NullIndicatorOffset` 里的 `byte_offset`/`bit_mask` 是**编译期就能确定的常量**（比如空值指示器固定在第 8 字节），如果只停留在 C++ 层面，LLVM 优化器根本不知道这个值是什么，无法做优化。

转换成 IR 常量后，LLVM 可以：
- 直接把后续代码中“读取 `byte_offset`”的操作，替换成具体的常量值（比如 8），省去运行时内存读取；
- 消除依赖这个值的无效代码（比如如果 `bit_mask=0`，直接删掉相关的判断逻辑）。

#### 2. 保证内存布局的“跨阶段一致性”
C++ 结构体的内存布局（成员顺序、padding、对齐）是由你的编译器（如 GCC/Clang）决定的（比如 64 位系统下，`int32_t + uint8_t` 会加 3 字节 padding 凑 8 字节）。

如果不转换成 IR 层面的结构体，LLVM 后端生成机器码时，无法准确复刻这个布局，会导致：
- 运行时访问结构体成员时，读错内存地址（比如把 padding 当成 `bit_mask`）；
- 内存对齐错误（比如某些架构要求结构体按 8 字节对齐，IR 层面没对齐会崩溃）。

而转换时用 `ConstantAggregateZero` 处理 padding，就是为了让 IR 结构体的布局和 C++ 完全一致。

#### 3. 适配 LLVM 的“语言无关性”
LLVM 是一个通用的编译器框架，不仅能处理 C++，还能处理 Rust、Go、Swift 等语言。它不认识 C++ 的结构体，只认识自己的 IR 类型（`StructType`/`ConstantStruct`）。

把 C++ 结构体转成 IR 常量，相当于把“C++ 方言”翻译成“LLVM 通用语”，这样不管前端是哪种语言，LLVM 中端/后端都能统一处理。

#### 4. 支撑后续的 IR 指令生成
在生成最终的机器码前，所有的逻辑（比如访问空值指示器、计算内存偏移）都要基于 IR 指令实现。如果 `NullIndicatorOffset` 没转成 IR 常量，后续无法构造“基于这个值计算内存地址”的 IR 指令，自然也生成不了对应的机器码。

### 三、举个简单例子，帮你理解转换的价值
假设 C++ 中定义：
```cpp
// C++ 层面的结构体实例（编译期常量）
const NullIndicatorOffset kNullOffset = {.byte_offset = 8, .bit_mask = 0x01};
```

如果不转换，LLVM 只能看到“C++ 代码里有个常量”，但不知道具体值；转换后，IR 层面会生成：
```llvm
; IR 层面的常量结构体（LLVM 能直接识别）
@kNullOffset = constant { i32, i8, i8, i8, i8 } { i32 8, i8 1, i8 0, i8 0, i8 0 }
```

此时 LLVM 优化器看到这个常量后，会把后续所有用到 `kNullOffset.byte_offset` 的地方，直接替换成 `8`，比如原本的 C++ 代码：
```cpp
// 计算空值指示器的内存地址
void* ptr = base + kNullOffset.byte_offset;
```
会被优化成：
```llvm
; 直接用常量 8 计算，省去读取结构体的操作
%ptr = getelementptr i8, i8* %base, i32 8
```
最终生成的机器码会更高效（少一条内存读取指令）。

### 总结
1. **本质差异**：C++ 结构体是“高级语言抽象”，IR 常量结构体是“编译器中间表示”，两者面向的阶段和受众完全不同；
2. **核心目的**：转换是为了让 LLVM 能识别、优化这个值，同时保证内存布局一致，支撑最终机器码的生成；
3. **关键价值**：编译期常量转换能触发 LLVM 的优化（如常量传播），提升最终程序的执行效率。

************************************************************************
这个是lowered的过程吗？
根据：
  /// Stores this value in an alloca allocation, and returns the pointer, which has the
  /// unlowered type. This *Val should be non-null. The output variable is called 'name'.
  llvm::Value* GetUnloweredPtr(const std::string& name = "") const;
  这个不是llvm的lowered的过程

***************************************************************88