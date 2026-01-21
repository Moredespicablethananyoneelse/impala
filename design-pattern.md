```cpp 
void* ScalarExprEvaluator::GetValue(const ScalarExpr& expr, const TupleRow* row) {
  switch (expr.type_.type) {
    case TYPE_BOOLEAN: {
      impala_udf::BooleanVal v = expr.GetBooleanVal(this, row);
      if (v.is_null) return nullptr;
      result_.bool_val = v.val;
      return &result_.bool_val;
    }
    case TYPE_TINYINT: {
      impala_udf::TinyIntVal v = expr.GetTinyIntVal(this, row);
      if (v.is_null) return nullptr;
      result_.tinyint_val = v.val;
      return &result_.tinyint_val;
    }
    case TYPE_SMALLINT: {
      impala_udf::SmallIntVal v = expr.GetSmallIntVal(this, row);
      if (v.is_null) return nullptr;
      result_.smallint_val = v.val;
      return &result_.smallint_val;
    }
    case TYPE_INT: {
      impala_udf::IntVal v = expr.GetIntVal(this, row);
      if (v.is_null) return nullptr;
      result_.int_val = v.val;
      return &result_.int_val;
    }
    case TYPE_BIGINT: {
      impala_udf::BigIntVal v = expr.GetBigIntVal(this, row);
      if (v.is_null) return nullptr;
      result_.bigint_val = v.val;
      return &result_.bigint_val;
    }
    case TYPE_FLOAT: {
      impala_udf::FloatVal v = expr.GetFloatVal(this, row);
      if (v.is_null) return nullptr;
      result_.float_val = v.val;
      return &result_.float_val;
    }
    case TYPE_DOUBLE: {
      impala_udf::DoubleVal v = expr.GetDoubleVal(this, row);
      if (v.is_null) return nullptr;
      result_.double_val = v.val;
      return &result_.double_val;
    }
    case TYPE_STRING:
    case TYPE_VARCHAR: {
      impala_udf::StringVal v = expr.GetStringVal(this, row);
      if (v.is_null) return nullptr;
      result_.string_val.Assign(reinterpret_cast<char*>(v.ptr), v.len);
      return &result_.string_val;
    }
    case TYPE_CHAR:
    case TYPE_FIXED_UDA_INTERMEDIATE: {
      impala_udf::StringVal v = expr.GetStringVal(this, row);
      if (v.is_null) return nullptr;
      result_.string_val.Assign(reinterpret_cast<char*>(v.ptr), v.len);
      return result_.string_val.Ptr();
    }
    case TYPE_TIMESTAMP: {
      impala_udf::TimestampVal v = expr.GetTimestampVal(this, row);
      if (v.is_null) return nullptr;
      result_.timestamp_val = TimestampValue::FromTimestampVal(v);
      return &result_.timestamp_val;
    }
    case TYPE_DECIMAL: {
      DecimalVal v = expr.GetDecimalVal(this, row);
      if (v.is_null) return nullptr;
      switch (expr.type_.GetByteSize()) {
        case 4:
          result_.decimal4_val = v.val4;
          return &result_.decimal4_val;
        case 8:
          result_.decimal8_val = v.val8;
          return &result_.decimal8_val;
        case 16:
          result_.decimal16_val = v.val16;
          return &result_.decimal16_val;
        default:
          DCHECK(false) << expr.type_.GetByteSize();
          return nullptr;
      }
    }
    case TYPE_DATE: {
      impala_udf::DateVal v = expr.GetDateVal(this, row);
      if (v.is_null) return nullptr;
      const DateValue dv = DateValue::FromDateVal(v);
      if (UNLIKELY(!dv.IsValid())) return nullptr;
      result_.date_val = dv;
      return &result_.date_val;
    }
    case TYPE_ARRAY:
    case TYPE_MAP: {
      impala_udf::CollectionVal v = expr.GetCollectionVal(this, row);
      if (v.is_null) return nullptr;
      result_.collection_val.ptr = v.ptr;
      result_.collection_val.num_tuples = v.num_tuples;
      return &result_.collection_val;
    }
    case TYPE_STRUCT: {
      StructVal v = expr.GetStructVal(this, row);
      if (v.is_null) return nullptr;
      result_.struct_val = v;
      return &result_.struct_val;
    }
    default:
      DCHECK(false) << "Type not implemented: " << expr.type_.DebugString();
      return nullptr;
  }
}
```
你观察得非常准确！这种设计确实初看有些“反直觉”，但它实际上是一种 精心设计的双向协作模式，其背后融合了 Visitor 模式（访问者模式） 与 Interpreter 模式（解释器模式） 的思想，并结合了 C++ 中常见的 CRTP（Curiously Recurring Template Pattern）或类似多态分发机制 来实现高效的表达式求值。

🧠 核心设计模式：Visitor + Interpreter 的混合体

1. 为什么说它像 Visitor 模式？
- 在标准的 Visitor 模式 中：
  - 有一组结构稳定的“元素”（比如各种 ScalarExpr 子类：Literal, SlotRef, AddExpr, CaseExpr 等）。
  - 有一个“访问者”（这里是 ScalarExprEvaluator），它对每种元素执行不同的操作（求值）。
  - 元素类提供一个 accept(Visitor*) 方法，调用 visitor->visit(this)。
  
- 在你的代码中：
  - ScalarExpr 是抽象基类，有多个子类（如 Literal, SlotRef, ScalarFnCall 等）。
  - ScalarExprEvaluator::GetValue(expr, row) 并不直接知道 expr 的具体类型。
  - 而是调用 expr.GetBooleanVal(this, row)、expr.GetIntVal(this, row) 等——这些是虚函数，由具体子类实现。
  - 这些子类方法内部会使用传入的 this（即 ScalarExprEvaluator*）来获取上下文（如 FunctionContext、内存池等）。

✅ 所以本质上：ScalarExpr 的每个子类“接受”一个 ScalarExprEvaluator 并在其上执行求值逻辑 —— 这正是 Visitor 模式的变体！

但注意：这里没有显式的 accept()，而是通过 **重载的 GeVal(ScalarExprEvaluator, TupleRow*) 虚函数** 实现了分发。这是一种 “逆向 Visitor” 或 “半 Visitor” 的实现。

2. 为什么又像 Interpreter 模式？
- Interpreter 模式 用于表示语言的文法，并解释句子（表达式）。
- ScalarExpr 树本身就是一个 抽象语法树（AST）：
  - 叶子节点：Literal, SlotRef
  - 内部节点：+, CASE, UDF, etc.
- ScalarExprEvaluator 就是这个 AST 的 解释器（interpreter），负责遍历并求值。

✅ 所以整体架构是：用 Visitor 风格在 Interpreter 框架下求值表达式树。

🔍 为什么这样设计？（看似“反直觉”的原因）

你可能觉得：“为什么不把求值逻辑全放在 ScalarExprEvaluator 里？为什么要让 ScalarExpr 子类自己实现 Get*Val？”

这是因为：

✅ 1. 解耦表达式结构与求值策略
- ScalarExpr 只定义“是什么”（结构），不硬编码“怎么算”。
- 不同的 evaluator（比如 JIT 编译 vs 解释执行）可以复用同一套 ScalarExpr 树。
- 未来可扩展：比如加一个 CodegenScalarExprEvaluator，只需实现新的 Get*Val 分支或替换底层机制。

✅ 2. 支持 UDF / 内置函数的统一接口
- 所有函数（包括 UDF）都通过 FunctionContext 与 evaluator 交互。
- ScalarFnCall::GetBigIntVal(...) 内部会调用注册的函数指针，并传入 fn_context()。
- 如果求值逻辑全在 evaluator 里，就需要 giant switch-case 或 type-id 判断，难以维护。

✅ 3. 性能考虑（虚函数 vs 大型 switch）
- 虽然虚函数有开销，但现代 CPU 分支预测对稳定调用模式很友好。
- 相比在一个 evaluator 里写上百个 if (type == ADD) ... else if (type == MUL) ...，虚函数更模块化、可测试、可内联（在 leaf node 如 Literal 中常被优化掉）。

✅ 4. 支持复杂类型（如 STRUCT, ARRAY）
- 对于 STRUCT，需要递归求值每个字段，而每个字段可能有自己的 evaluator（见 childEvaluators_）。
- 这种嵌套结构天然适合由 ScalarExpr 子类自己控制求值流程。

🔄 数据流向示例

// 用户调用
evaluator->GetIntVal(row);

// → 调用 root_.GetIntVal(this, row)
//    假设 root_ 是 AddExpr

// AddExpr::GetIntVal(ScalarExprEvaluator* eval, TupleRow* row) {
//   IntVal lhs = children()[0]->GetIntVal(eval, row);
//   IntVal rhs = children()[1]->GetIntVal(eval, row);
//   return lhs.val + rhs.val;
// }

可以看到：控制流在 ScalarExpr 子类中，但上下文和资源由 ScalarExprEvaluator 提供。

📌 总结
特性   说明
主要模式   Visitor 模式（变体） + Interpreter 模式

角色分工   ScalarExpr：定义表达式结构；ScalarExprEvaluator：提供求值上下文和资源管理

为何“反直觉”   因为求值逻辑分散在 ScalarExpr 子类中，而非集中于 evaluator —— 但这恰恰是解耦和扩展性的体现

优势   模块化、支持 UDF、易于 JIT 扩展、类型安全、避免巨型 switch

💡 这不是设计缺陷，而是一种 高性能查询引擎中常见的“表达式-求值器分离”架构，在 Impala、Spark Catalyst、Presto、ClickHouse 等系统中都有类似设计。

如果你熟悉 LLVM 或 JIT 编译，还会发现这种设计也便于后续将 ScalarExpr 树编译成 native code —— 因为语义已经清晰分离。

如有兴趣，我可以进一步画出类图或数据流图来辅助理解！
*************************************************************
正常的访问者模式是什么样的

非常好的问题！我们来对比一下 “标准（经典）的访问者模式（Visitor Pattern）” 和你在 Impala 代码中看到的 “变体”，这样就能更清楚为什么说它是 Visitor 的一种变形。

✅ 一、标准 Visitor 模式（GoF 定义）

🎯 目的
在不修改元素类的前提下，为一组对象结构（通常是树或图）增加新的操作。

典型场景：AST（抽象语法树）、文档对象模型（DOM）、几何图形集合等。

🧱 核心组成
角色   职责
Element（元素）   定义一个 accept(Visitor*) 方法；通常是抽象基类 + 多个子类

ConcreteElementA/B/...   具体元素类（如 AddExpr, Literal, SlotRef）

Visitor（访问者）   声明对每种元素的访问接口，如 visit(ConcreteElementA*)

ConcreteVisitor   实现具体操作（如求值、打印、类型检查、代码生成）

💻 经典 C++ 示例

// 1. 元素基类
class Element {
public:
    virtual ~Element() = default;
    virtual void accept(class Visitor* visitor) = 0; // 关键：接受访问者
};

// 2. 具体元素
class Literal : public Element {
public:
    int value_;
    Literal(int v) : value_(v) {}
    void accept(Visitor* visitor) override {
        visitor->visit(this); // 回调访问者
    }
};

class AddExpr : public Element {
public:
    Element* left_, *right_;
    AddExpr(Element* l, Element* r) : left_(l), right_(r) {}
    void accept(Visitor* visitor) override {
        visitor->visit(this);
    }
};

// 3. 访问者接口
class Visitor {
public:
    virtual ~Visitor() = default;
    virtual void visit(Literal* lit) = 0;
    virtual void visit(AddExpr* add) = 0;
};

// 4. 具体访问者：求值器
class Evaluator : public Visitor {
public:
    int result_;

    void visit(Literal* lit) override {
        result_ = lit->value_;
    }

    void visit(AddExpr* add) override {
        add->left_->accept(this);   // 递归左子树
        int left_val = result_;
        add->right_->accept(this);  // 递归右子树
        int right_val = result_;
        result_ = left_val + right_val;
    }
};

🔁 使用方式

Element* expr = new AddExpr(new Literal(2), new Literal(3));
Evaluator eval;
expr->accept(&eval);  // 启动访问
std::cout visit(this)，实现 双分派（double dispatch）

🔁 二、Impala 中的“变体 Visitor”

现在回看你的代码：

void* ScalarExprEvaluator::GetValue(const ScalarExpr& expr, const TupleRow* row) {
    switch (expr.type_.type) {
        case TYPE_INT:
            IntVal v = expr.GetIntVal(this, row); // ← 注意：expr 调用自己的方法，并传入 this（evaluator）
            ...
    }
}

而 ScalarExpr::GetIntVal(...) 是一个 虚函数，由子类实现：

// 在 AddExpr.cc 中
IntVal AddExpr::GetIntVal(ScalarExprEvaluator* eval, const TupleRow* row) {
    IntVal lhs = children()[0]->GetIntVal(eval, row);
    IntVal rhs = children()[1]->GetIntVal(eval, row);
    return IntVal(lhs.val + rhs.val);
}

❗关键区别
特性   标准 Visitor   Impala 变体
谁发起调用？   Element 调用 visitor->visit(this)   Evaluator 调用 expr->Get*Val(this, ...)

分派方向   Element → Visitor   Visitor → Element

是否显式 accept()？   是   否

是否双分派？   是（运行时确定 Element 类型 + Visitor 方法）   单分派 + 手动类型 switch（外层有 switch，内层靠虚函数）

扩展新操作   新增 Visitor 子类即可   需要给所有 ScalarExpr 子类新增 GetXxxVal 方法（但已通过宏或模板自动化）

🔍 所以 Impala 的设计 不是严格意义上的 GoF Visitor，而是 “逆向 Visitor” 或 “基于虚函数的解释器”。

但它 达到了 Visitor 的核心目标：
- 将“操作”（求值）与“结构”（表达式树）分离
- 支持多种操作（求值、打印、常量折叠、代码生成）——虽然每种操作需要预定义接口

🤔 为什么不用标准 Visitor？

因为在高性能系统（如数据库执行引擎）中：

1. 标准 Visitor 需要每个元素实现 accept() → 代码冗余
2. 双分派有轻微性能开销
3. C++ 虚函数表已经能高效分发到具体子类
4. **求值是核心操作，值得为它定制接口（Get*Val）**

所以 Impala 选择了 更直接、更高效、更贴近 UDF 接口 的方式。

✅ 总结：两种风格对比
风格   优点   缺点   适用场景
标准 Visitor   扩展操作极其方便，完全解耦   需要 accept()，双分派略重   编译器、文档处理、频繁新增操作

Impala 变体（虚函数求值）   高效、直接、与 UDF 无缝集成   新增返回类型需改所有子类   数据库执行引擎、固定操作集、极致性能

💡 你可以把 Impala 的设计理解为：“为特定领域（表达式求值）优化的 Visitor-like 架构”。

如果你感兴趣，我也可以展示如何用 真正的标准 Visitor 模式重写 Impala 的求值逻辑（虽然实际不会这么做 😄）。

*************************************************************************
单分派和双分派指的是什么

这是一个非常关键且深刻的问题！“单分派（single dispatch）”和“双分派（double dispatch）”是面向对象编程中关于 方法调用如何根据运行时类型进行分发 的核心概念。我们来详细解释。

🔹 一、什么是“分派”（Dispatch）？

分派（Dispatch）：指在运行时决定调用哪个具体函数（方法）的过程。

- 静态分派（Static Dispatch）：编译时确定（如函数重载、模板）。
- 动态分派（Dynamic Dispatch）：运行时根据对象的实际类型确定（如 C++ 虚函数）。

🔸 二、单分派（Single Dispatch）

✅ 定义：
方法的调用只根据 一个对象的运行时类型 来决定。

这是 绝大多数面向对象语言（C++、Java、Python 等）默认支持的方式。

💡 示例（C++ 虚函数）：
class Expr {
public:
    virtual void print() = 0;
};

class Literal : public Expr {
    void print() override { cout print(); // ← 运行时根据 e 的实际类型（Add）调用 Add::print()

✅ 这里只有 一个对象 e 的类型 决定了调用哪个 print() —— 单分派。

🔸 三、双分派（Double Dispatch）

✅ 定义：
方法的调用根据 两个对象的运行时类型 共同决定。

⚠️ 大多数语言 不直接支持双分派，但可以通过 Visitor 模式模拟实现。

💡 经典场景：碰撞检测
// 假设有不同形状：Circle, Rectangle
// 我们想实现：shape1.collide(shape2)
// 但 collide 行为取决于 shape1 和 shape2 的具体类型组合！

❌ 单分派做不到：
class Shape {
public:
    virtual void collide(Shape* other) = 0;
};

class Circle : public Shape {
    void collide(Shape* other) override {
        // 这里只知道 this 是 Circle，
        // 但不知道 other 是 Circle 还是 Rectangle！
        // 因为 other->collide(...) 只会调用 Shape::collide，不是虚函数多态？
        // 实际上：other 的类型在编译时未知，无法静态分派到 Circle::collide(Circle*)
    }
};

✅ Visitor 模式实现双分派：
class Shape {
public:
    virtual void accept(Visitor* v) = 0;
    virtual void collideWithCircle() = 0;
    virtual void collideWithRect() = 0;
};

class Circle : public Shape {
    void accept(Visitor* v) override { v->visit(this); }
    void collideWithCircle() override { cout visit(this); }
    void collideWithCircle() override { cout collideWithCircle(); // ← 第二次分派！
    }
    void visit(Rect* r) override {
        target_->collideWithRect();   // ← 第二次分派！
    }
};

// 使用
Shape* s1 = new Circle();
Shape* s2 = new Rect();

CollisionVisitor v(s1);
s2->accept(&v); // 输出 "Circle-Rect"

🔍 双分派的两次动态分派：
1. 第一次：s2->accept(&v) → 根据 s2 的类型（Rect）调用 Rect::accept
2. 第二次：在 Rect::accept 中调用 v->visit(this) → 根据 v 的类型（CollisionVisitor）调用 CollisionVisitor::visit(Rect*)
   - 然后 visit(Rect*) 调用 target_->collideWithRect() → 根据 target_（即 s1，Circle）调用 Circle::collideWithRect()

✅ 最终行为由 s1 和 s2 的运行时类型共同决定 → 双分派！

🔸 四、回到 Impala 的对比
特性   标准 Visitor   Impala 变体
是否双分派？   ✅ 是   ❌ 否（本质是单分派）

为什么标准 Visitor 是双分派？
- element->accept(visitor)
  - 第一次分派：根据 element 的类型 → 调用对应 accept
  - 第二次分派：在 accept 中调用 visitor->visit(this) → 根据 visitor 类型 + this 的静态类型（但通过重载解析到具体 visit(ConcreteElement*)）
  - 实际效果：操作（visitor）和元素（element）的类型共同决定行为

为什么 Impala 是单分派？
看这段代码：
void* ScalarExprEvaluator::GetValue(const ScalarExpr& expr, const TupleRow* row) {
    switch (expr.type_.type) {          // ← 手动类型判断（非动态分派！）
        case TYPE_INT:
            IntVal v = expr.GetIntVal(this, row); // ← 虚函数调用（单分派）
            ...
    }
}

- expr.GetIntVal(...) 是虚函数 → 根据 expr 的运行时类型 分派到 AddExpr::GetIntVal 或 Literal::GetIntVal 等。
- 但 没有第二个运行时类型参与分派！
- 外层的 switch (expr.type_.type) 是 手动类型检查，不是动态分派（它依赖 type_ 成员变量，而非虚函数表）。

📌 所以整个过程只依赖 一个对象（expr）的运行时类型 → 单分派。

即使你在 AddExpr::GetIntVal 中递归调用子表达式的 GetIntVal，那也只是 多次单分派，不是双分派。

✅ 总结
概念   说明
单分派   方法调用仅根据 一个接收者对象 的运行时类型决定（C++ 虚函数就是典型）

双分派   方法调用根据 两个对象 的运行时类型共同决定（需 Visitor 模式等技巧模拟）

Impala   使用 单分派（虚函数） + 手动类型 switch 实现求值，高效但非双分派

标准 Visitor   通过两次虚函数调用实现 双分派，更通用但略重

💡 简单记忆：  
- 单分派：obj->method() → 看 obj 是谁  
- 双分派：obj1->method(obj2) → 看 obj1 和 obj2 分别是谁（大多数语言做不到，需 trick）

希望这能彻底澄清你的疑问！如果还有不清楚的地方，欢迎继续追问 😊
************************************************************************************************
*************************************************************
我发现
```cpp
Status ScalarExpr::GetCodegendComputeFn(
    LlvmCodeGen* codegen, bool is_codegen_entry_point, llvm::Function** fn) {
  if (ir_compute_fn_ != nullptr) {
    *fn = ir_compute_fn_;
  } else {
    RETURN_IF_ERROR(GetCodegendComputeFnImpl(codegen, fn));
    ir_compute_fn_ = *fn;
  }
  if (is_codegen_entry_point && !added_to_jit_) {
    // Ensure Get*Val() is made callable if this function is called at least once
    // with is_codegen_entry_point=true.
    added_to_jit_ = true;
    codegen->AddFunctionToJit(*fn, &codegend_compute_fn_);
  }
  return Status::OK();
} ```

父类ScarlarExpr调用了子类的GetCodegendComputeFnImpl对吗？这是什么设计模式？

而

```cpp
 /// Expr for evaluating and (&&) operators
 class AndPredicate: public CompoundPredicate {
 public:
  virtual BooleanVal GetBooleanValInterpreted(
      ScalarExprEvaluator*, const TupleRow*) const;

  virtual Status GetCodegendComputeFnImpl(LlvmCodeGen* codegen, llvm::Function** fn) {
    return CompoundPredicate::CodegenComputeFn(true, codegen, fn);
  }

 protected:
  friend class ScalarExpr;
  AndPredicate(const TExprNode& node) : CompoundPredicate(node) { }

  virtual std::string DebugString() const;

 private:
  friend class OpcodeRegistry;
};
```
中 
```cpp
 virtual Status GetCodegendComputeFnImpl(LlvmCodeGen* codegen, llvm::Function** fn) {
    return CompoundPredicate::CodegenComputeFn(true, codegen, fn);
  }
```
子类调用了父类的函数，这是什么设计模式



你观察得非常准确！这段代码中确实存在 两层嵌套的、方向相反的函数调用关系，分别体现了两种经典的设计模式：
✅ 第一层：模板方法模式（Template Method Pattern）
调用方向：
父类 ScalarExpr → 虚函数 → 子类实现（如 AndPredicate::GetCodegendComputeFnImpl）
代码体现：
cpp

编辑



// 父类 ScalarExpr 中
```cpp
Status ScalarExpr::GetCodegendComputeFn(...) {
  if (ir_compute_fn_ != nullptr) { ... }
  else {
    RETURN_IF_ERROR(GetCodegendComputeFnImpl(codegen, fn)); // ← 虚函数调用！
    ir_compute_fn_ = *fn;
  }
  ...
}
```
// 纯虚函数声明（子类必须实现）
```cpp
virtual Status GetCodegendComputeFnImpl(LlvmCodeGen* codegen, llvm::Function** fn) = 0;
```
设计意图：
父类定义算法骨架：GetCodegendComputeFn 负责通用逻辑（缓存检查、JIT 注册等）。
子类提供具体实现：每个表达式类型（AndPredicate, Literal, SlotRef 等）自己决定如何生成 IR。
这是典型的 模板方法模式：父类控制流程，子类填充细节。
💡 类比：就像做菜，父类说“先洗菜 → 再切菜 → 然后炒”，但“怎么切”、“放什么调料”由子类（川菜、粤菜）决定。
✅ 第二层：策略复用 / 共同行为提取（Common Behavior Extraction）
调用方向：
子类 AndPredicate → 调用其父类 CompoundPredicate 的普通成员函数
代码体现：
cpp

编辑


```cpp
// AndPredicate（子类）中
virtual Status GetCodegendComputeFnImpl(...) {
  return CompoundPredicate::CodegenComputeFn(true, codegen, fn); // ← 调用父类非虚函数
}

// OrPredicate（另一个子类）中
virtual Status GetCodegendComputeFnImpl(...) {
  return CompoundPredicate::CodegenComputeFn(false, codegen, fn);
}
```
而 CompoundPredicate::CodegenComputeFn 是一个 非虚的 protected 成员函数，实现了 AND/OR 公共的复杂 IR 生成逻辑。
设计意图：
AndPredicate 和 OrPredicate 的 IR 生成逻辑 高度相似，只有少量差异（AND vs OR 的布尔操作、短路语义）。
把公共逻辑抽到中间基类 CompoundPredicate 中，避免重复代码。
子类通过传入 bool and_fn 参数来定制行为。
💡 这不是 GoF 23 种模式中的某一种，但属于 面向对象设计的基本原则：DRY（Don't Repeat Yourself） + 继承复用。
可以看作是 “受控的模板方法变体” —— 公共逻辑在中间层实现，叶子类只负责配置参数。

*******************************************************************
基类的
```cpp 
void PlanNode::Codegen(FragmentState* state) {
  DCHECK(state->ShouldCodegen());
  DCHECK(state->codegen() != nullptr);
  for (PlanNode* child : children_) {
    child->Codegen(state);
  }
}
```
和子类的
```cpp
void AggregationPlanNode::Codegen(FragmentState* state) {
  DCHECK(state->ShouldCodegen());
  PlanNode::Codegen(state);
  if (IsNodeCodegenDisabled()) return;
  for (auto& agg : aggs_) {
    agg->Codegen(state);
  }
}
```
之间是什么设计模式
这段代码中，AggregationPlanNode::Codegen() 覆盖（override）了基类 PlanNode::Codegen() 的行为，并在其中 先调用父类方法，再执行自身特有的逻辑。这种结构体现了典型的 模板方法模式（Template Method Pattern） 的一种变体或应用方式。

具体分析：

1. 基类方法：通用流程骨架
```cpp
void PlanNode::Codegen(FragmentState* state) {
  DCHECK(state->ShouldCodegen());
  DCHECK(state->codegen() != nullptr);
  for (PlanNode* child : children_) {
    child->Codegen(state); // 递归处理子节点
  }
}
```

- 这是一个通用的代码生成入口。
- 它定义了一个固定流程：对所有子节点递归调用 Codegen()。
- 但没有处理当前节点自身的代码生成逻辑（因为 PlanNode 是抽象概念）。

1. 子类方法：扩展特定行为

```cpp
void AggregationPlanNode::Codegen(FragmentState* state) {
  DCHECK(state->ShouldCodegen());
  PlanNode::Codegen(state);           // ← 先执行通用流程（处理子节点）
  if (IsNodeCodegenDisabled()) return;
  for (auto& agg : aggs_) {
    agg->Codegen(state);              // ← 再执行本节点特有的代码生成
  }
}
```

- 子类 复用 了父类的通用逻辑（遍历子树）。
- 然后 追加 自己特有的逻辑（为每个聚合器 agg 生成代码）。
- 这种“先通用、后特有”的结构，是模板方法模式的典型体现。

✅ 为什么是模板方法模式？

模板方法模式：在一个方法中定义一个算法的骨架，而将一些步骤延迟到子类中。使得子类可以在不改变算法结构的情况下重新定义该算法的某些特定步骤。

虽然这里 PlanNode::Codegen() 本身不是一个完整的“算法骨架”（它只处理子节点，没留出钩子），但整个设计意图非常接近：

- 父类提供递归遍历子树的标准流程（这是所有 plan node codegen 的共性）。
- 子类在继承该流程的基础上，插入自己的 codegen 逻辑（这是个性）。
- 这避免了每个子类都重复写 for (child: children_) child->Codegen() 的样板代码。

因此，这是一种 轻量级的模板方法模式，或者说 基于继承的流程扩展模式。

🔁 对比经典模板方法

经典模板方法通常像这样：
```cpp
// 基类
void TemplateMethod() {
  Step1();      // 固定
  Hook();       // 可被子类覆盖
  Step2();      // 固定
}

virtual void Hook() {} // 默认空实现

而这里的变体是：
// 基类
void Codegen() {
  ProcessChildren(); // 固定：递归子节点
}

// 子类
void Codegen() {
  Parent::Codegen(); // ← 相当于调用 ProcessChildren()
  DoMyOwnWork();     // ← 相当于覆盖 Hook()
}
```
虽然没有显式的 Hook() 函数，但通过 显式调用父类方法 + 追加逻辑，达到了同样的效果。

📌 结论

这属于 模板方法模式（Template Method Pattern） 的一种 实践变体，其核心思想是：

“复用通用流程，扩展特定行为”。

在 Impala 这样的大型查询引擎中，这种模式被广泛用于：
- Prepare()
- Open()
- Close()
- Codegen()

等生命周期方法中，确保树形执行计划的每个节点既能遵循统一的执行协议，又能注入自身特有的逻辑。

✅ 答案：模板方法模式（Template Method Pattern）。
******************************************************************************8
