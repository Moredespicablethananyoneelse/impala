// IR codegen for compound and/or predicates.  Compound predicate has non trivial
// null handling as well as many branches so this is pretty complicated.  The IR
// for x && y is:
//
// define i16 @CompoundPredicate(%"class.impala::ScalarExprEvaluator"* %eval,
//                               %"class.impala::TupleRow"* %row) #20 {
// entry:
//   %lhs_call = call i16 @GetSlotRef1(%"class.impala::ScalarExprEvaluator"* %eval,
//                                     %"class.impala::TupleRow"* %row)
//   %rhs_call = call i16 @Eq_IntVal_IntValWrapper(
//                                %"class.impala::ScalarExprEvaluator"* %eval,
//                                %"class.impala::TupleRow"* %row)
//   %is_null = trunc i16 %lhs_call to i1
//   %is_null1 = trunc i16 %rhs_call to i1
//   %0 = ashr i16 %lhs_call, 8
//   %1 = trunc i16 %0 to i8
//   %val = trunc i8 %1 to i1
//   %2 = ashr i16 %rhs_call, 8
//   %3 = trunc i16 %2 to i8
//   %val2 = trunc i8 %3 to i1
//   %tmp_and = and i1 %val, %val2
//   br i1 %is_null, label %lhs_null, label %lhs_not_null
//
// lhs_null:                                         ; preds = %entry
//   br i1 %is_null1, label %null_block, label %lhs_null_rhs_not_null
//
// lhs_not_null:                                     ; preds = %entry
//   br i1 %is_null1, label %lhs_not_null_rhs_null, label %not_null_block
//
// lhs_null_rhs_not_null:                            ; preds = %lhs_null
//   br i1 %val2, label %null_block, label %not_null_block
//
// lhs_not_null_rhs_null:                            ; preds = %lhs_not_null
//   br i1 %val, label %null_block, label %not_null_block
//
// null_block:                                       ; preds = %lhs_null_rhs_not_null,
//                                                     %lhs_not_null_rhs_null, %lhs_null
//   br label %ret
//
// not_null_block:                                   ; preds = %lhs_null_rhs_not_null,
//                                                   %lhs_not_null_rhs_null, %lhs_not_null
//   %4 = phi i1 [ false, %lhs_null_rhs_not_null ],
//               [ false, %lhs_not_null_rhs_null ],
//               [ %tmp_and, %lhs_not_null ]
//   br label %ret
//
// ret:                                              ; preds = %not_null_block, %null_block
//   %ret3 = phi i1 [ false, %null_block ], [ %4, %not_null_block ]
//   %5 = zext i1 %ret3 to i16
//   %6 = shl i16 %5, 8
//   %7 = or i16 0, %6
//   ret i16 %7
// }

这段注释和代码展示了 Apache Impala 中对 AND / OR 复合谓词（compound predicates）的 LLVM IR 代码生成逻辑，核心难点在于 正确处理 NULL 值的三值逻辑（Three-Valued Logic, 3VL）。

我们先解释你问的 IR 示例（即注释中给出的 LLVM IR），再结合 C++ 代码理解其设计思想。

🎯 背景：SQL 中的 AND/OR 与 NULL

在 SQL 中，布尔运算遵循 三值逻辑：
- TRUE, FALSE, NULL（表示“未知”）

AND (x && y) 规则：
x         y         x AND y
FALSE     any       FALSE

TRUE      TRUE      TRUE

TRUE      NULL      NULL

NULL      FALSE     FALSE

NULL      TRUE      NULL

NULL      NULL      NULL

✅ 关键点：只要有一个是 FALSE，结果就是 FALSE（短路）；否则，如果有 NULL，结果就是 NULL；只有两个都是 TRUE，才是 TRUE。

OR (x || y) 规则：
x         y         x OR y
TRUE      any       TRUE

FALSE     FALSE     FALSE

FALSE     NULL      NULL

NULL      TRUE      TRUE

NULL      FALSE     NULL

NULL      NULL      NULL

✅ 关键点：只要有一个是 TRUE，结果就是 TRUE；否则，如果有 NULL，结果就是 NULL；只有两个都是 FALSE，才是 FALSE。

🔍 IR 示例解析：x && y 的 LLVM 实现

define i16 @CompoundPredicate(...) {
entry:
  %lhs_call = call i16 @...   ; 左操作数，返回 BooleanVal（i16）
  %rhs_call = call i16 @...   ; 右操作数

1. BooleanVal 的内存布局（i16 表示）
Impala 用 16 位整数（i16） 表示 BooleanVal：
- 最低 1 位（bit 0）：is_null（1 = NULL，0 = 非 NULL）
- 高 8 位（bit 8~15）：val（0 = false，非 0 = true）

所以：
%is_null = trunc i16 %lhs_call to i1          ; 取 bit 0 → is_null
%0 = ashr i16 %lhs_call, 8                    ; 算术右移 8 位 → val 在低 8 位
%1 = trunc i16 %0 to i8
%val = trunc i8 %1 to i1                      ; 转为 bool（0 或 1）

💡 这种设计允许用单个寄存器传递 is_null + val，高效且兼容 ABI。

2. 控制流：处理所有 NULL 组合

IR 构建了一个 决策树，覆盖所有 (lhs_is_null, rhs_is_null) 组合：

主分支：根据 lhs_is_null
br i1 %is_null, label %lhs_null, label %lhs_not_null

情况 A: lhs is NULL
- 再看 rhs：
  - 如果 rhs is NULL → 结果 NULL（进入 null_block）
  - 如果 rhs is NOT NULL → 进入 %lhs_null_rhs_not_null

情况 B: lhs is NOT NULL
- 再看 rhs：
  - 如果 rhs is NULL → 进入 %lhs_not_null_rhs_null
  - 如果 rhs is NOT NULL → 直接计算 lhs && rhs（进入 not_null_block）

3. 特殊子情况处理

子情况：lhs=NULL, rhs=NOT NULL
br i1 %val2, label %null_block, label %not_null_block

- 如果 rhs = TRUE → NULL && TRUE = NULL → 走 null_block
- 如果 rhs = FALSE → NULL && FALSE = FALSE → 走 not_null_block（结果为 false）

子情况：lhs=NOT NULL, rhs=NULL
br i1 %val, label %null_block, label %not_null_block

- 如果 lhs = TRUE → TRUE && NULL = NULL
- 如果 lhs = FALSE → FALSE && NULL = FALSE

4. PHI 节点：合并结果

not_null_block 中的结果（非 NULL 路径）
%4 = phi i1 [ false, %lhs_null_rhs_not_null ],
            [ false, %lhs_not_null_rhs_null ],
            [ %tmp_and, %lhs_not_null ]

- 来自 lhs_null_rhs_not_null（即 NULL && FALSE）→ false
- 来自 lhs_not_null_rhs_null（即 FALSE && NULL）→ false
- 来自 lhs_not_null（即 lhs && rhs 都非 NULL）→ %tmp_and

ret 块：最终合并 NULL vs 非 NULL
%ret3 = phi i1 [ false, %null_block ], [ %4, %not_null_block ]

- 注意：即使结果是 NULL，这里也返回 false 作为 val 字段！
- 因为 BooleanVal 的完整表示 = (is_null, val)
  - 在 null_block 中，is_null = true，val 可以是任意值（通常设为 false）

最后打包成 i16 返回：
%5 = zext i1 %ret3 to i16     ; val → bit 8
%6 = shl i16 %5, 8
%7 = or i16 0, %6             ; is_null = 0（因为非 NULL 路径），但等等...

⚠️ 等等！这里似乎有问题？  
实际上，在 ret 块中应该有两个 PHI：
- 一个用于 is_null
- 一个用于 val

而 IR 示例为了简化，只展示了 val 的 PHI，并假设 is_null 单独处理（可能在更完整的 IR 中）。

✅ 总结：这段 IR 的核心思想

1. 精确实现 SQL 三值逻辑：覆盖所有 NULL 组合。
2. 避免运行时函数调用：全部展开为分支和位操作。
3. 利用 PHI 节点合并路径：确保 SSA 形式正确。
4. 短路优化隐含在控制流中：
   - 虽然没有显式“提前返回”，但通过分支跳过不必要的计算（如 lhs=false 时仍会算 rhs？——实际上在这个 IR 中 没有短路！）

❗ 注意：这个 IR 没有实现解释器中的短路求值（short-circuiting）！
- 解释器版本：if (!val1.is_null && !val1.val) return false;（不计算 rhs）
- 但 JIT 版本：总是计算 lhs 和 rhs（因为函数调用已内联/预计算）
这是因为在向量化或批处理场景中，分支预测失败的代价 > 多算一个表达式，所以选择“全计算 + 分支合并”。

📌 为什么这么复杂？

因为 NULL 语义不能简单用 && / || 模拟。例如：
- NULL && FALSE 必须是 FALSE，不是 NULL
- NULL || TRUE 必须是 TRUE，不是 NULL

这种“非对称”的 NULL 传播规则，导致必须显式处理 四种组合：
1. (非 NULL, 非 NULL)
2. (NULL, 非 NULL)
3. (非 NULL, NULL)
4. (NULL, NULL)

每种都可能产生不同结果（TRUE/FALSE/NULL），所以需要多个基本块和 PHI 节点。

🔚 后续：C++ 代码如何生成这个 IR？

CompoundPredicate::CodegenComputeFn 正是通过：
- 创建多个 BasicBlock
- 使用 CreateCondBr 构建决策树
- 用 PHINode 合并结果
- 调用 CodegenAnyVal 封装 is_null/val 提取

来程序化生成上述 IR，从而在 JIT 编译时得到高性能、语义正确的 AND/OR 实现。

这体现了现代分析型数据库的核心技术：将复杂的 SQL 语义编译为高效的机器码。



********************************************************************************8
