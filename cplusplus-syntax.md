template <bool COLLECT_VAR_LEN_VALS, bool NO_POOL>
void Tuple::MaterializeExprs(TupleRow* row, const TupleDescriptor& desc,
    ScalarExprEvaluator* const* evals, MemPool* pool,
    CodegenTypes::StringValuePtrVecType* non_null_string_values,
    CodegenTypes::CollValuePtrAndSizeVecType* non_null_collection_values,
    int* total_varlen_lengths,
    int* num_non_null_string_values, int* num_non_null_collection_values) noexcept {
  if constexpr (COLLECT_VAR_LEN_VALS) {
    DCHECK(non_null_string_values != nullptr);
    DCHECK(non_null_collection_values != nullptr);
    DCHECK(total_varlen_lengths != nullptr);
    DCHECK(num_non_null_string_values != nullptr);
    DCHECK(num_non_null_collection_values != nullptr);
  }

  ClearNullBits(desc);

  // Evaluate the materialize_expr_evals and place the results in the tuple.
  for (int i = 0; i < desc.slots().size(); ++i) {
    SlotDescriptor* slot_desc = desc.slots()[i];
    // The FE ensures we don't get any TYPE_NULL expressions by picking an arbitrary type
    // when necessary, but does not do this for slot descs.
    // TODO: revisit this logic in the FE
    DCHECK(slot_desc->type().type == TYPE_NULL ||
        slot_desc->type() == evals[i]->root().type());
    void* src = evals[i]->GetValue(row);

    const size_t old_num_string_values = COLLECT_VAR_LEN_VALS ?
        non_null_string_values->size() : 0;
    const size_t old_num_collection_values = COLLECT_VAR_LEN_VALS ?
        non_null_collection_values->size() : 0;
    RawValue::Write<COLLECT_VAR_LEN_VALS>(src, this, slot_desc, pool,
        non_null_string_values, non_null_collection_values);

    if constexpr (COLLECT_VAR_LEN_VALS) {
      const size_t new_num_string_values = non_null_string_values->size();
      for (size_t i = old_num_string_values; i < new_num_string_values; i++) {
        const StringValue* string_val = (*non_null_string_values)[i];
        DCHECK(!string_val->IsSmall());
        *total_varlen_lengths += string_val->Len();
      }
      (*num_non_null_string_values) += new_num_string_values - old_num_string_values;

      const size_t new_num_collection_values = non_null_collection_values->size();
      for (size_t i = old_num_collection_values; i < new_num_collection_values; i++) {
        const pair<CollectionValue*, int64_t>& collection_val_pair =
            (*non_null_collection_values)[i];
        *total_varlen_lengths += collection_val_pair.second;
      }
      (*num_non_null_collection_values) +=
          new_num_collection_values - old_num_collection_values;
    }
  }
}

在你提供的代码中：

if constexpr (COLLECT_VAR_LEN_VALS) {
  // ...
}

这里的 constexpr 是 C++17 引入的 编译期条件判断（compile-time if） 特性，正式名称是 if constexpr。

🔍 为什么使用 if constexpr？

背景：
MaterializeExprs 是一个模板函数，其行为由两个布尔模板参数控制：

- COLLECT_VAR_LEN_VALS：是否收集可变长度值（如字符串、集合）。
- NO_POOL（或 NULL_POOL）：是否使用内存池。

当 COLLECT_VAR_LEN_VALS == false 时，函数中涉及 non_null_string_values、non_null_collection_values 等 vector 的操作 不应该被编译，因为这些参数可能传入的是 nullptr，或者根本不会被使用。如果用普通的 if，即使条件为 false，编译器仍会 解析并实例化 if 块内的代码，从而导致编译错误（比如对空指针解引用、调用不存在的方法等）。

✅ if constexpr 的作用：
- 在 编译期 根据模板参数的值决定是否 丢弃（discard） 某一分支的代码。
- 如果条件为 false，那么该分支中的代码 不会被实例化（not instantiated），也就不会参与类型检查、符号解析等过程。
- 这使得我们可以安全地在同一个函数模板中处理完全不同的逻辑路径，而不会因未使用的代码导致编译失败。

📌 举个例子说明区别

假设没有 constexpr，写成普通 if：

template 
void foo(std::vector* v) {
    if (COLLECT) {
        v->push_back(42);  // 如果 COLLECT=false，v 可能是 nullptr！
    }
}

即使调用时 COLLECT=false，编译器仍会检查 v->push_back(42) 是否合法。如果 v 是 nullptr，虽然运行时不会执行，但语法上必须合法。更严重的是，如果 v 根本没传（比如设为 nullptr 字面量），这段代码依然要能编译通过——这在某些上下文中是不可能的。

而用 if constexpr：

template 
void foo(std::vector* v) {
    if constexpr (COLLECT) {
        v->push_back(42);  // 当 COLLECT=false 时，这行完全被忽略！
    }
}

当 COLLECT == false 时，整个 if 块被 从 AST 中移除，编译器根本不会看到 v->push_back(...)，因此即使 v 是 nullptr 或类型不匹配，也不会报错。

🧠 回到你的代码

在 Tuple::MaterializeExprs 中：

if constexpr (COLLECT_VAR_LEN_VALS) {
    DCHECK(non_null_string_values != nullptr);
    // ... 使用 non_null_string_values 和 non_null_collection_values
    // ... 计算 total_varlen_lengths
}

- 当 COLLECT_VAR_LEN_VALS = false 时，这些 DCHECK 和 vector 操作 完全不会被编译。
- 这允许同一个模板函数既能用于“只物化基本类型”的场景（高性能路径），也能用于“需要收集变长数据”的场景（如排序、序列化），而无需写两个独立函数。
- 同时支持 LLVM 代码生成（codegen）：在 codegen 路径中通常只启用 COLLECT_VAR_LEN_VALS = false，避免 vector 操作带来的开销和复杂性。

✅ 总结
特性   普通 if   if constexpr
判断时机   运行时（或常量折叠）   编译期

分支代码是否实例化   总是实例化   仅满足条件的分支被实例化

用途   控制流程   模板元编程、条件编译、消除无效代码

所以，if constexpr (COLLECT_VAR_LEN_VALS) 的存在是为了：
- 安全地在同一个函数模板中支持两种模式；
- 避免未使用分支引发编译错误；
- 提升性能（编译器可完全优化掉无用代码）；
- 便于与 LLVM Codegen 集成（生成更干净的 IR）。

这是现代 C++ 模板编程中非常关键的一项技术。
*************************************************************************
这是if constexpr的常用场景吗/
if constexpr 是 C++17 引入的一个编译期条件判断特性，主要用于模板编程中根据类型或常量表达式在编译期选择不同的代码路径。它与运行时的 if 语句不同，未被选中的分支不会被实例化（instantiated），因此可以安全地编写仅对某些类型有效的代码。

✅ 常见且推荐的使用场景包括：

1. 根据类型特征（type traits）做不同处理
template
void process(T value) {
    if constexpr (std::is_integral_v) {
        // 只有当 T 是整型时，这段代码才会被编译
        std::cout ) {
        std::cout 
void print_all(Args... args) {
    ((std::cout 
void print_tuple(const std::tuple& t) {
    if constexpr (I (t) (t);
    }
}

3. 可选功能的编译期开关
template
void do_something(T&& x) {
    // 主逻辑
    process(x);

    if constexpr (EnableLogging) {
        log("Processed ", x);
    }
}

这样可以在不牺牲性能的情况下，通过模板参数控制是否包含日志等调试代码。

4. 适配不同接口（例如有无某个成员函数）
虽然不能直接检测成员是否存在（需配合 requires 或 SFINAE），但一旦通过 trait 知道了，就可以用 if constexpr：
template
void maybe_call_resize(T& container) {
    if constexpr (has_resize_method_v) {
        container.resize(100);
    }
}

❌ 不适合的场景：

- 运行时条件判断：if constexpr 的条件必须是编译期常量表达式。如果用运行时变量，会报错。
    int n = 5;
  if constexpr (n > 0) { /* 错误！n 不是 constexpr */ }
  

- 非模板上下文：在非模板函数中使用 if constexpr 虽然语法合法，但通常没必要，因为所有分支都会被检查（除非条件恒真/假），不如普通 if。

总结：

✅ 是的，if constexpr 的上述场景是非常常用且推荐的，尤其在泛型编程中能大幅简化代码、替代复杂的 SFINAE 或模板特化，提高可读性和维护性。

如果你有具体代码片段，我可以帮你判断是否适合用 if constexpr。
**********************************************************************