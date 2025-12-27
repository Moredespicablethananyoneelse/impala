ScalarExpr 是 Apache Impala（一个开源的 MPP SQL 查询引擎）中用于表示标量表达式（scalar expression）的核心类。它的设计目标是高效地支持两种执行模式：
解释执行（interpreted execution）
代码生成（codegen，即 LLVM JIT 编译）

下面从多个维度介绍 ScalarExpr 的设计思想与实现要点。

一、基本概念
什么是 ScalarExpr？
ScalarExpr 表示返回单个值（scalar value）的表达式，例如：
字面量（literal）：如 "hello"、42
列引用（SlotRef）：如 table.col
函数调用：如 abs(x)、concat(a, b)
条件表达式：如 CASE WHEN ... THEN ... ELSE ... END
UDF（用户自定义函数）
它是 Expr 的子类，但专门用于非聚合、非窗口的标量计算。

二、核心设计目标

1. 统一接口：无论表达式类型如何，都通过统一的 GetVal() 接口（如 GetIntVal()）获取结果。
2. 双模执行：
解释模式：调用虚函数 GetValInterpreted()。
代码生成模式：生成 LLVM IR，编译为机器码，通过函数指针直接调用。
3. 内存效率：支持紧凑的内存布局（见 ScalarExprsResultsRowLayout）。
4. 可组合性：表达式以树形结构组织，子表达式递归求值。
5. 资源管理：支持 OpenEvaluator() / CloseEvaluator() 生命周期管理（如 UDF 的 FunctionContext）。

三、关键组件与机制
1. 类型系统与返回值
所有返回值使用 UDF AnyVal 类型（如 IntVal, StringVal），这些类型包含 is_null 标志和实际值。
每种可能的返回类型都有对应的 GetVal() 和 GetValInterpreted() 方法。
默认的 GetValInterpreted() 实现会 DCHECK(false)，强制子类重写。
2. 双模执行调度

cpp
IntVal GetIntVal(ScalarExprEvaluator eval, const TupleRow row) const {
if (codegend_compute_fn_ != nullptr) {
return reinterpret_cast<IntVal()(...)>(
codegend_compute_fn_.load())(eval, row);
}
return GetIntValInterpreted(eval, row);
}
如果已生成 JIT 函数（codegend_compute_fn_ 非空），则直接调用；
否则回退到解释执行。
3. 代码生成（Codegen）
每个 ScalarExpr 子类需实现：
cpp
virtual Status GetCodegendComputeFnImpl(LlvmCodeGen codegen, llvm::Function fn);
该函数使用 IRBuilder 构建 LLVM IR，内联子表达式的计算，避免虚函数调用开销。
生成的函数签名统一为：
cpp
Val ComputeFn(ScalarExprEvaluator, const TupleRow);

4. 入口点优化（Entry Point）
若表达式是“根节点”（可能被解释代码直接调用），则将其 JIT 函数注册为 entry point：
cpp
codegen->AddFunctionToJit(fn, &codegend_compute_fn_);
这样 GetVal() 就能直接跳转到 JIT 代码，提升性能。
5. 常量折叠与常量表达式
is_constant_ 标志表示该表达式是否为常量（如 5 + 3）。
常量表达式可在查询计划阶段预计算，避免运行时重复计算。
6. FunctionContext 支持
某些表达式（如 UDF）需要 FunctionContext 来管理状态。
通过 AssignFnCtxIdx() 为每个需要上下文的节点分配索引。
在 OpenEvaluator() 中初始化上下文，CloseEvaluator() 中清理。
7. 内存布局优化
ScalarExprsResultsRowLayout 用于批量计算多个表达式的结果。
将定长类型靠前、变长类型（如 String）靠后，便于向量化处理和内存对齐。

四、子类扩展机制

Impala 为不同表达式类型提供了具体子类，例如：

表达式类型 对应子类
---------- --------
字面量 Literal
列引用 SlotRef
函数调用 ScalarFnCall
CASE 表达式 CaseExpr
UDF（Java） HiveUdfCall
IS NULL IsNullExpr

每个子类：
重写 GetValInterpreted()
可选重写 GetCodegendComputeFnImpl()（若需定制 IR）
可重写 IsInterpretable()（默认为 true）

五、生命周期管理

1. 创建：通过 ScalarExpr::Create() 从 Thrift TExpr 构建表达式树。
2. 初始化：Init() 递归初始化子节点，并注册到 codegen 队列。
3. 打开：OpenEvaluator() 初始化运行时资源（如 FunctionContext）。
4. 求值：通过 GetVal() 或子表达式递归调用。
5. 关闭：CloseEvaluator() 释放资源；Close() 释放缓存等。

六、设计亮点总结

特性 说明
------ ------
双模执行 解释 + Codegen，兼顾灵活性与性能
零虚函数调用（Codegen 路径） 内联子表达式，极致性能
类型安全 通过模板化 AnyVal 和 DCHECK 保证类型正确
资源隔离 每个表达式树独立管理 FunctionContext
可扩展 易于添加新表达式类型
内存友好 结果布局优化，利于批处理

七、典型使用场景

cpp
// 创建表达式
ScalarExpr expr;
ScalarExpr::Create(texpr, row_desc, state, &expr);

// 求值
ScalarExprEvaluator eval(expr, state);
TupleRow* row = ...;
IntVal result = expr->GetIntVal(&eval, row);

在 Join、Filter、Projection 等算子中广泛使用。

总结

ScalarExpr 是 Impala 表达式系统的核心抽象，其设计体现了高性能 OLAP 引擎对表达式求值的极致优化：
在保持解释执行灵活性的同时，通过 LLVM Codegen 消除解释开销，实现接近手写 C++ 的性能。

这种“解释兜底 + JIT 加速”的混合执行模型，已成为现代分析型数据库（如 Spark, Presto, Doris）的标配。
****************************************************************************
