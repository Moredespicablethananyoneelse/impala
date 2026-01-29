参见/workspaces/impala/fe/src/main/java/org/apache/impala/catalog/Function.java
be/src/exprs/decimal-operators.h
/workspaces/impala/be/src/exprs/operators.h
be/src/exprs/anyval-util.h

看上去apache impala是将四则运算当成函数节点了。而不是另外一种Expr的叫oper的子节点。
同时这几个类的实现也揭示了。*Val只是接口类型，内部进行四则运算/逻辑运算等操作的是*Value类型（除了cpu支持的类型外的，都需要先转换成TimestampValue,DecimalValue,StringValue，然后调用这些*Value的相关的成员函数）。

参见/workspaces/impala/be/src/runtime/type.md关于ColumnType的ToIR函数
参见be/src/runtime/descriptors.cc的llvm::Value* SlotDescriptor::CodegenStoreNonNullAnyValToNewAlloca可以对比下slot-ref.h中的对应函数
*****************************************************************************
请将文件operators-ir.cc中的宏都展开，方便我学习各个函数的实现方式

### 核心需求梳理
你希望将 Impala `operators-ir.cc` 中的所有宏完全展开，得到每个运算符函数的完整实现代码，从而清晰学习各函数的具体实现方式。以下是**全宏展开后的完整代码**，按「位运算、算术运算、阶乘、比较运算、NULL 安全比较」分模块整理，保留完整语义和注释，方便你对照学习。

### 宏展开后的完整代码（按功能模块划分）
```cpp
// Licensed to the Apache Software Foundation (ASF) under one
// or more contributor license agreements.  See the NOTICE file
// distributed with this work for additional information
// regarding copyright ownership.  The ASF licenses this file
// to you under the Apache License, Version 2.0 (the
// "License"); you may not use this file except in compliance
// with the License.  You may obtain a copy of the License at
//
//   http://www.apache.org/licenses/LICENSE-2.0
//
// Unless required by applicable law or agreed to in writing,
// software distributed under the License is distributed on an
// "AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY
// KIND, either express or implied.  See the License for the
// specific language governing permissions and limitations
// under the License.

#include "exprs/operators.h"

#include <cstdint>
#include <functional>

#include "exprs/anyval-util.h"
#include "gutil/strings/substitute.h"
#include "runtime/string-value.inline.h"
#include "runtime/timestamp-value.h"
#include "util/bit-util.h"

#include "common/names.h"

namespace impala {

// ========================== 1. 位运算（Bitnot）==========================
// Bitnot_TinyIntVal 展开实现
TinyIntVal Operators::Bitnot_TinyIntVal(FunctionContext* c, const TinyIntVal& v) {
  if (v.is_null) return TinyIntVal::null();
  return TinyIntVal(~v.val);
}

// Bitnot_SmallIntVal 展开实现
SmallIntVal Operators::Bitnot_SmallIntVal(FunctionContext* c, const SmallIntVal& v) {
  if (v.is_null) return SmallIntVal::null();
  return SmallIntVal(~v.val);
}

// Bitnot_IntVal 展开实现
IntVal Operators::Bitnot_IntVal(FunctionContext* c, const IntVal& v) {
  if (v.is_null) return IntVal::null();
  return IntVal(~v.val);
}

// Bitnot_BigIntVal 展开实现
BigIntVal Operators::Bitnot_BigIntVal(FunctionContext* c, const BigIntVal& v) {
  if (v.is_null) return BigIntVal::null();
  return BigIntVal(~v.val);
}

// ========================== 2. 算术运算（加减乘除模）==========================
// ------------------------- 2.1 整数加减乘（无符号溢出处理）-------------------------
// Add_TinyIntVal_TinyIntVal 展开实现
TinyIntVal Operators::Add_TinyIntVal_TinyIntVal(
    FunctionContext* c, const TinyIntVal& v1, const TinyIntVal& v2) {
  if (v1.is_null || v2.is_null) return TinyIntVal::null();
  return TinyIntVal(ArithmeticUtil::AsUnsigned<std::plus>(v1.val, v2.val));
}

// Add_SmallIntVal_SmallIntVal 展开实现
SmallIntVal Operators::Add_SmallIntVal_SmallIntVal(
    FunctionContext* c, const SmallIntVal& v1, const SmallIntVal& v2) {
  if (v1.is_null || v2.is_null) return SmallIntVal::null();
  return SmallIntVal(ArithmeticUtil::AsUnsigned<std::plus>(v1.val, v2.val));
}

// Add_IntVal_IntVal 展开实现
IntVal Operators::Add_IntVal_IntVal(
    FunctionContext* c, const IntVal& v1, const IntVal& v2) {
  if (v1.is_null || v2.is_null) return IntVal::null();
  return IntVal(ArithmeticUtil::AsUnsigned<std::plus>(v1.val, v2.val));
}

// Add_BigIntVal_BigIntVal 展开实现
BigIntVal Operators::Add_BigIntVal_BigIntVal(
    FunctionContext* c, const BigIntVal& v1, const BigIntVal& v2) {
  if (v1.is_null || v2.is_null) return BigIntVal::null();
  return BigIntVal(ArithmeticUtil::AsUnsigned<std::plus>(v1.val, v2.val));
}

// Subtract_TinyIntVal_TinyIntVal 展开实现
TinyIntVal Operators::Subtract_TinyIntVal_TinyIntVal(
    FunctionContext* c, const TinyIntVal& v1, const TinyIntVal& v2) {
  if (v1.is_null || v2.is_null) return TinyIntVal::null();
  return TinyIntVal(ArithmeticUtil::AsUnsigned<std::minus>(v1.val, v2.val));
}

// Subtract_SmallIntVal_SmallIntVal 展开实现
SmallIntVal Operators::Subtract_SmallIntVal_SmallIntVal(
    FunctionContext* c, const SmallIntVal& v1, const SmallIntVal& v2) {
  if (v1.is_null || v2.is_null) return SmallIntVal::null();
  return SmallIntVal(ArithmeticUtil::AsUnsigned<std::minus>(v1.val, v2.val));
}

// Subtract_IntVal_IntVal 展开实现
IntVal Operators::Subtract_IntVal_IntVal(
    FunctionContext* c, const IntVal& v1, const IntVal& v2) {
  if (v1.is_null || v2.is_null) return IntVal::null();
  return IntVal(ArithmeticUtil::AsUnsigned<std::minus>(v1.val, v2.val));
}

// Subtract_BigIntVal_BigIntVal 展开实现
BigIntVal Operators::Subtract_BigIntVal_BigIntVal(
    FunctionContext* c, const BigIntVal& v1, const BigIntVal& v2) {
  if (v1.is_null || v2.is_null) return BigIntVal::null();
  return BigIntVal(ArithmeticUtil::AsUnsigned<std::minus>(v1.val, v2.val));
}

// Multiply_TinyIntVal_TinyIntVal 展开实现
TinyIntVal Operators::Multiply_TinyIntVal_TinyIntVal(
    FunctionContext* c, const TinyIntVal& v1, const TinyIntVal& v2) {
  if (v1.is_null || v2.is_null) return TinyIntVal::null();
  return TinyIntVal(ArithmeticUtil::AsUnsigned<std::multiplies>(v1.val, v2.val));
}

// Multiply_SmallIntVal_SmallIntVal 展开实现
SmallIntVal Operators::Multiply_SmallIntVal_SmallIntVal(
    FunctionContext* c, const SmallIntVal& v1, const SmallIntVal& v2) {
  if (v1.is_null || v2.is_null) return SmallIntVal::null();
  return SmallIntVal(ArithmeticUtil::AsUnsigned<std::multiplies>(v1.val, v2.val));
}

// Multiply_IntVal_IntVal 展开实现
IntVal Operators::Multiply_IntVal_IntVal(
    FunctionContext* c, const IntVal& v1, const IntVal& v2) {
  if (v1.is_null || v2.is_null) return IntVal::null();
  return IntVal(ArithmeticUtil::AsUnsigned<std::multiplies>(v1.val, v2.val));
}

// Multiply_BigIntVal_BigIntVal 展开实现
BigIntVal Operators::Multiply_BigIntVal_BigIntVal(
    FunctionContext* c, const BigIntVal& v1, const BigIntVal& v2) {
  if (v1.is_null || v2.is_null) return BigIntVal::null();
  return BigIntVal(ArithmeticUtil::AsUnsigned<std::multiplies>(v1.val, v2.val));
}

// ------------------------- 2.2 浮点加减乘（直接运算）-------------------------
// Add_FloatVal_FloatVal 展开实现
FloatVal Operators::Add_FloatVal_FloatVal(
    FunctionContext* c, const FloatVal& v1, const FloatVal& v2) {
  if (v1.is_null || v2.is_null) return FloatVal::null();
  return FloatVal(v1.val + v2.val);
}

// Add_DoubleVal_DoubleVal 展开实现
DoubleVal Operators::Add_DoubleVal_DoubleVal(
    FunctionContext* c, const DoubleVal& v1, const DoubleVal& v2) {
  if (v1.is_null || v2.is_null) return DoubleVal::null();
  return DoubleVal(v1.val + v2.val);
}

// Subtract_FloatVal_FloatVal 展开实现
FloatVal Operators::Subtract_FloatVal_FloatVal(
    FunctionContext* c, const FloatVal& v1, const FloatVal& v2) {
  if (v1.is_null || v2.is_null) return FloatVal::null();
  return FloatVal(v1.val - v2.val);
}

// Subtract_DoubleVal_DoubleVal 展开实现
DoubleVal Operators::Subtract_DoubleVal_DoubleVal(
    FunctionContext* c, const DoubleVal& v1, const DoubleVal& v2) {
  if (v1.is_null || v2.is_null) return DoubleVal::null();
  return DoubleVal(v1.val - v2.val);
}

// Multiply_FloatVal_FloatVal 展开实现
FloatVal Operators::Multiply_FloatVal_FloatVal(
    FunctionContext* c, const FloatVal& v1, const FloatVal& v2) {
  if (v1.is_null || v2.is_null) return FloatVal::null();
  return FloatVal(v1.val * v2.val);
}

// Multiply_DoubleVal_DoubleVal 展开实现
DoubleVal Operators::Multiply_DoubleVal_DoubleVal(
    FunctionContext* c, const DoubleVal& v1, const DoubleVal& v2) {
  if (v1.is_null || v2.is_null) return DoubleVal::null();
  return DoubleVal(v1.val * v2.val);
}

// ------------------------- 2.3 浮点除法（直接运算）-------------------------
// Divide_DoubleVal_DoubleVal 展开实现
DoubleVal Operators::Divide_DoubleVal_DoubleVal(
    FunctionContext* c, const DoubleVal& v1, const DoubleVal& v2) {
  if (v1.is_null || v2.is_null) return DoubleVal::null();
  return DoubleVal(v1.val / v2.val);
}

// ------------------------- 2.4 整数除法/取模（检查除数为0）-------------------------
// Int_divide_TinyIntVal_TinyIntVal 展开实现
TinyIntVal Operators::Int_divide_TinyIntVal_TinyIntVal(
    FunctionContext* c, const TinyIntVal& v1, const TinyIntVal& v2) {
  if (v1.is_null || v2.is_null || v2.val == 0) return TinyIntVal::null();
  return TinyIntVal(v1.val / v2.val);
}

// Int_divide_SmallIntVal_SmallIntVal 展开实现
SmallIntVal Operators::Int_divide_SmallIntVal_SmallIntVal(
    FunctionContext* c, const SmallIntVal& v1, const SmallIntVal& v2) {
  if (v1.is_null || v2.is_null || v2.val == 0) return SmallIntVal::null();
  return SmallIntVal(v1.val / v2.val);
}

// Int_divide_IntVal_IntVal 展开实现
IntVal Operators::Int_divide_IntVal_IntVal(
    FunctionContext* c, const IntVal& v1, const IntVal& v2) {
  if (v1.is_null || v2.is_null || v2.val == 0) return IntVal::null();
  return IntVal(v1.val / v2.val);
}

// Int_divide_BigIntVal_BigIntVal 展开实现
BigIntVal Operators::Int_divide_BigIntVal_BigIntVal(
    FunctionContext* c, const BigIntVal& v1, const BigIntVal& v2) {
  if (v1.is_null || v2.is_null || v2.val == 0) return BigIntVal::null();
  return BigIntVal(v1.val / v2.val);
}

// Mod_TinyIntVal_TinyIntVal 展开实现
TinyIntVal Operators::Mod_TinyIntVal_TinyIntVal(
    FunctionContext* c, const TinyIntVal& v1, const TinyIntVal& v2) {
  if (v1.is_null || v2.is_null || v2.val == 0) return TinyIntVal::null();
  return TinyIntVal(v1.val % v2.val);
}

// Mod_SmallIntVal_SmallIntVal 展开实现
SmallIntVal Operators::Mod_SmallIntVal_SmallIntVal(
    FunctionContext* c, const SmallIntVal& v1, const SmallIntVal& v2) {
  if (v1.is_null || v2.is_null || v2.val == 0) return SmallIntVal::null();
  return SmallIntVal(v1.val % v2.val);
}

// Mod_IntVal_IntVal 展开实现
IntVal Operators::Mod_IntVal_IntVal(
    FunctionContext* c, const IntVal& v1, const IntVal& v2) {
  if (v1.is_null || v2.is_null || v2.val == 0) return IntVal::null();
  return IntVal(v1.val % v2.val);
}

// Mod_BigIntVal_BigIntVal 展开实现
BigIntVal Operators::Mod_BigIntVal_BigIntVal(
    FunctionContext* c, const BigIntVal& v1, const BigIntVal& v2) {
  if (v1.is_null || v2.is_null || v2.val == 0) return BigIntVal::null();
  return BigIntVal(v1.val % v2.val);
}

// ------------------------- 2.5 位运算（与/异或/或）-------------------------
// Bitand_TinyIntVal_TinyIntVal 展开实现
TinyIntVal Operators::Bitand_TinyIntVal_TinyIntVal(
    FunctionContext* c, const TinyIntVal& v1, const TinyIntVal& v2) {
  if (v1.is_null || v2.is_null) return TinyIntVal::null();
  return TinyIntVal(v1.val & v2.val);
}

// Bitand_SmallIntVal_SmallIntVal 展开实现
SmallIntVal Operators::Bitand_SmallIntVal_SmallIntVal(
    FunctionContext* c, const SmallIntVal& v1, const SmallIntVal& v2) {
  if (v1.is_null || v2.is_null) return SmallIntVal::null();
  return SmallIntVal(v1.val & v2.val);
}

// Bitand_IntVal_IntVal 展开实现
IntVal Operators::Bitand_IntVal_IntVal(
    FunctionContext* c, const IntVal& v1, const IntVal& v2) {
  if (v1.is_null || v2.is_null) return IntVal::null();
  return IntVal(v1.val & v2.val);
}

// Bitand_BigIntVal_BigIntVal 展开实现
BigIntVal Operators::Bitand_BigIntVal_BigIntVal(
    FunctionContext* c, const BigIntVal& v1, const BigIntVal& v2) {
  if (v1.is_null || v2.is_null) return BigIntVal::null();
  return BigIntVal(v1.val & v2.val);
}

// Bitxor_TinyIntVal_TinyIntVal 展开实现
TinyIntVal Operators::Bitxor_TinyIntVal_TinyIntVal(
    FunctionContext* c, const TinyIntVal& v1, const TinyIntVal& v2) {
  if (v1.is_null || v2.is_null) return TinyIntVal::null();
  return TinyIntVal(v1.val ^ v2.val);
}

// Bitxor_SmallIntVal_SmallIntVal 展开实现
SmallIntVal Operators::Bitxor_SmallIntVal_SmallIntVal(
    FunctionContext* c, const SmallIntVal& v1, const SmallIntVal& v2) {
  if (v1.is_null || v2.is_null) return SmallIntVal::null();
  return SmallIntVal(v1.val ^ v2.val);
}

// Bitxor_IntVal_IntVal 展开实现
IntVal Operators::Bitxor_IntVal_IntVal(
    FunctionContext* c, const IntVal& v1, const IntVal& v2) {
  if (v1.is_null || v2.is_null) return IntVal::null();
  return IntVal(v1.val ^ v2.val);
}

// Bitxor_BigIntVal_BigIntVal 展开实现
BigIntVal Operators::Bitxor_BigIntVal_BigIntVal(
    FunctionContext* c, const BigIntVal& v1, const BigIntVal& v2) {
  if (v1.is_null || v2.is_null) return BigIntVal::null();
  return BigIntVal(v1.val ^ v2.val);
}

// Bitor_TinyIntVal_TinyIntVal 展开实现
TinyIntVal Operators::Bitor_TinyIntVal_TinyIntVal(
    FunctionContext* c, const TinyIntVal& v1, const TinyIntVal& v2) {
  if (v1.is_null || v2.is_null) return TinyIntVal::null();
  return TinyIntVal(v1.val | v2.val);
}

// Bitor_SmallIntVal_SmallIntVal 展开实现
SmallIntVal Operators::Bitor_SmallIntVal_SmallIntVal(
    FunctionContext* c, const SmallIntVal& v1, const SmallIntVal& v2) {
  if (v1.is_null || v2.is_null) return SmallIntVal::null();
  return SmallIntVal(v1.val | v2.val);
}

// Bitor_IntVal_IntVal 展开实现
IntVal Operators::Bitor_IntVal_IntVal(
    FunctionContext* c, const IntVal& v1, const IntVal& v2) {
  if (v1.is_null || v2.is_null) return IntVal::null();
  return IntVal(v1.val | v2.val);
}

// Bitor_BigIntVal_BigIntVal 展开实现
BigIntVal Operators::Bitor_BigIntVal_BigIntVal(
    FunctionContext* c, const BigIntVal& v1, const BigIntVal& v2) {
  if (v1.is_null || v2.is_null) return BigIntVal::null();
  return BigIntVal(v1.val | v2.val);
}

// ========================== 3. 阶乘运算（Factorial）==========================
static const int64_t FACTORIAL_MAX = 20;
static const int64_t FACTORIAL_LOOKUP[] = {
    1LL, // 0!
    1LL, // 1!
    2LL, // 2!
    6LL, // 3!
    24LL, // 4!
    120LL, // 5!
    720LL, // 6!
    5040LL, // 7!
    40320LL, // 8!
    362880LL, // 9!
    3628800LL, // 10!
    39916800LL, // 11!
    479001600LL, // 12!
    6227020800LL, // 13!
    87178291200LL, // 14!
    1307674368000LL, // 15!
    20922789888000LL, // 16!
    355687428096000LL, // 17!
    6402373705728000LL, // 18!
    121645100408832000LL, // 19!
    2432902008176640000LL, // 20!
};

// Compute factorial - return -1 if out of range
// Factorial of any number <= 1 returns 1
static int64_t ComputeFactorial(int64_t n) {
  // Check range based on arg: 20! < 2^63 -1 < 21!
  if (n > FACTORIAL_MAX) {
    return -1;
  } else if (n < 0) {
    return 1;
  }

  return FACTORIAL_LOOKUP[n];
}

// Factorial_TinyIntVal 展开实现
BigIntVal Operators::Factorial_TinyIntVal(FunctionContext* c, const TinyIntVal& v) {
  if (v.is_null) return BigIntVal::null();
  int64_t fact = ComputeFactorial(v.val);
  if (fact < 0) {
    c->SetError(Substitute("factorial $0! too large for BIGINT", v.val).c_str());
    return BigIntVal::null();
  }
  return BigIntVal(fact);
}

// Factorial_SmallIntVal 展开实现
BigIntVal Operators::Factorial_SmallIntVal(FunctionContext* c, const SmallIntVal& v) {
  if (v.is_null) return BigIntVal::null();
  int64_t fact = ComputeFactorial(v.val);
  if (fact < 0) {
    c->SetError(Substitute("factorial $0! too large for BIGINT", v.val).c_str());
    return BigIntVal::null();
  }
  return BigIntVal(fact);
}

// Factorial_IntVal 展开实现
BigIntVal Operators::Factorial_IntVal(FunctionContext* c, const IntVal& v) {
  if (v.is_null) return BigIntVal::null();
  int64_t fact = ComputeFactorial(v.val);
  if (fact < 0) {
    c->SetError(Substitute("factorial $0! too large for BIGINT", v.val).c_str());
    return BigIntVal::null();
  }
  return BigIntVal(fact);
}

// Factorial_BigIntVal 展开实现
BigIntVal Operators::Factorial_BigIntVal(FunctionContext* c, const BigIntVal& v) {
  if (v.is_null) return BigIntVal::null();
  int64_t fact = ComputeFactorial(v.val);
  if (fact < 0) {
    c->SetError(Substitute("factorial $0! too large for BIGINT", v.val).c_str());
    return BigIntVal::null();
  }
  return BigIntVal(fact);
}

// ========================== 4. 普通比较运算（Eq/Ne/Gt/Lt/Ge/Le）==========================
// ------------------------- 4.1 数值类型比较（Boolean/TinyInt/SmallInt/Int/BigInt/Float/Double/Date）-------------------------
// Eq_BooleanVal_BooleanVal 展开实现
BooleanVal Operators::Eq_BooleanVal_BooleanVal(
    FunctionContext* c, const BooleanVal& v1, const BooleanVal& v2) {
  if (v1.is_null || v2.is_null) return BooleanVal::null();
  return BooleanVal(v1.val == v2.val);
}

// Eq_TinyIntVal_TinyIntVal 展开实现
BooleanVal Operators::Eq_TinyIntVal_TinyIntVal(
    FunctionContext* c, const TinyIntVal& v1, const TinyIntVal& v2) {
  if (v1.is_null || v2.is_null) return BooleanVal::null();
  return BooleanVal(v1.val == v2.val);
}

// Eq_SmallIntVal_SmallIntVal 展开实现
BooleanVal Operators::Eq_SmallIntVal_SmallIntVal(
    FunctionContext* c, const SmallIntVal& v1, const SmallIntVal& v2) {
  if (v1.is_null || v2.is_null) return BooleanVal::null();
  return BooleanVal(v1.val == v2.val);
}

// Eq_IntVal_IntVal 展开实现
BooleanVal Operators::Eq_IntVal_IntVal(
    FunctionContext* c, const IntVal& v1, const IntVal& v2) {
  if (v1.is_null || v2.is_null) return BooleanVal::null();
  return BooleanVal(v1.val == v2.val);
}

// Eq_BigIntVal_BigIntVal 展开实现
BooleanVal Operators::Eq_BigIntVal_BigIntVal(
    FunctionContext* c, const BigIntVal& v1, const BigIntVal& v2) {
  if (v1.is_null || v2.is_null) return BooleanVal::null();
  return BooleanVal(v1.val == v2.val);
}

// Eq_FloatVal_FloatVal 展开实现
BooleanVal Operators::Eq_FloatVal_FloatVal(
    FunctionContext* c, const FloatVal& v1, const FloatVal& v2) {
  if (v1.is_null || v2.is_null) return BooleanVal::null();
  return BooleanVal(v1.val == v2.val);
}

// Eq_DoubleVal_DoubleVal 展开实现
BooleanVal Operators::Eq_DoubleVal_DoubleVal(
    FunctionContext* c, const DoubleVal& v1, const DoubleVal& v2) {
  if (v1.is_null || v2.is_null) return BooleanVal::null();
  return BooleanVal(v1.val == v2.val);
}

// Eq_DateVal_DateVal 展开实现
BooleanVal Operators::Eq_DateVal_DateVal(
    FunctionContext* c, const DateVal& v1, const DateVal& v2) {
  if (v1.is_null || v2.is_null) return BooleanVal::null();
  return BooleanVal(v1.val == v2.val);
}

// Ne_BooleanVal_BooleanVal 展开实现
BooleanVal Operators::Ne_BooleanVal_BooleanVal(
    FunctionContext* c, const BooleanVal& v1, const BooleanVal& v2) {
  if (v1.is_null || v2.is_null) return BooleanVal::null();
  return BooleanVal(v1.val != v2.val);
}

// Ne_TinyIntVal_TinyIntVal 展开实现
BooleanVal Operators::Ne_TinyIntVal_TinyIntVal(
    FunctionContext* c, const TinyIntVal& v1, const TinyIntVal& v2) {
  if (v1.is_null || v2.is_null) return BooleanVal::null();
  return BooleanVal(v1.val != v2.val);
}

// Ne_SmallIntVal_SmallIntVal 展开实现
BooleanVal Operators::Ne_SmallIntVal_SmallIntVal(
    FunctionContext* c, const SmallIntVal& v1, const SmallIntVal& v2) {
  if (v1.is_null || v2.is_null) return BooleanVal::null();
  return BooleanVal(v1.val != v2.val);
}

// Ne_IntVal_IntVal 展开实现
BooleanVal Operators::Ne_IntVal_IntVal(
    FunctionContext* c, const IntVal& v1, const IntVal& v2) {
  if (v1.is_null || v2.is_null) return BooleanVal::null();
  return BooleanVal(v1.val != v2.val);
}

// Ne_BigIntVal_BigIntVal 展开实现
BooleanVal Operators::Ne_BigIntVal_BigIntVal(
    FunctionContext* c, const BigIntVal& v1, const BigIntVal& v2) {
  if (v1.is_null || v2.is_null) return BooleanVal::null();
  return BooleanVal(v1.val != v2.val);
}

// Ne_FloatVal_FloatVal 展开实现
BooleanVal Operators::Ne_FloatVal_FloatVal(
    FunctionContext* c, const FloatVal& v1, const FloatVal& v2) {
  if (v1.is_null || v2.is_null) return BooleanVal::null();
  return BooleanVal(v1.val != v2.val);
}

// Ne_DoubleVal_DoubleVal 展开实现
BooleanVal Operators::Ne_DoubleVal_DoubleVal(
    FunctionContext* c, const DoubleVal& v1, const DoubleVal& v2) {
  if (v1.is_null || v2.is_null) return BooleanVal::null();
  return BooleanVal(v1.val != v2.val);
}

// Ne_DateVal_DateVal 展开实现
BooleanVal Operators::Ne_DateVal_DateVal(
    FunctionContext* c, const DateVal& v1, const DateVal& v2) {
  if (v1.is_null || v2.is_null) return BooleanVal::null();
  return BooleanVal(v1.val != v2.val);
}

// Gt_BooleanVal_BooleanVal 展开实现
BooleanVal Operators::Gt_BooleanVal_BooleanVal(
    FunctionContext* c, const BooleanVal& v1, const BooleanVal& v2) {
  if (v1.is_null || v2.is_null) return BooleanVal::null();
  return BooleanVal(v1.val > v2.val);
}

// Gt_TinyIntVal_TinyIntVal 展开实现
BooleanVal Operators::Gt_TinyIntVal_TinyIntVal(
    FunctionContext* c, const TinyIntVal& v1, const TinyIntVal& v2) {
  if (v1.is_null || v2.is_null) return BooleanVal::null();
  return BooleanVal(v1.val > v2.val);
}

// Gt_SmallIntVal_SmallIntVal 展开实现
BooleanVal Operators::Gt_SmallIntVal_SmallIntVal(
    FunctionContext* c, const SmallIntVal& v1, const SmallIntVal& v2) {
  if (v1.is_null || v2.is_null) return BooleanVal::null();
  return BooleanVal(v1.val > v2.val);
}

// Gt_IntVal_IntVal 展开实现
BooleanVal Operators::Gt_IntVal_IntVal(
    FunctionContext* c, const IntVal& v1, const IntVal& v2) {
  if (v1.is_null || v2.is_null) return BooleanVal::null();
  return BooleanVal(v1.val > v2.val);
}

// Gt_BigIntVal_BigIntVal 展开实现
BooleanVal Operators::Gt_BigIntVal_BigIntVal(
    FunctionContext* c, const BigIntVal& v1, const BigIntVal& v2) {
  if (v1.is_null || v2.is_null) return BooleanVal::null();
  return BooleanVal(v1.val > v2.val);
}

// Gt_FloatVal_FloatVal 展开实现
BooleanVal Operators::Gt_FloatVal_FloatVal(
    FunctionContext* c, const FloatVal& v1, const FloatVal& v2) {
  if (v1.is_null || v2.is_null) return BooleanVal::null();
  return BooleanVal(v1.val > v2.val);
}

// Gt_DoubleVal_DoubleVal 展开实现
BooleanVal Operators::Gt_DoubleVal_DoubleVal(
    FunctionContext* c, const DoubleVal& v1, const DoubleVal& v2) {
  if (v1.is_null || v2.is_null) return BooleanVal::null();
  return BooleanVal(v1.val > v2.val);
}

// Gt_DateVal_DateVal 展开实现
BooleanVal Operators::Gt_DateVal_DateVal(
    FunctionContext* c, const DateVal& v1, const DateVal& v2) {
  if (v1.is_null || v2.is_null) return BooleanVal::null();
  return BooleanVal(v1.val > v2.val);
}

// Lt_BooleanVal_BooleanVal 展开实现
BooleanVal Operators::Lt_BooleanVal_BooleanVal(
    FunctionContext* c, const BooleanVal& v1, const BooleanVal& v2) {
  if (v1.is_null || v2.is_null) return BooleanVal::null();
  return BooleanVal(v1.val < v2.val);
}

// Lt_TinyIntVal_TinyIntVal 展开实现
BooleanVal Operators::Lt_TinyIntVal_TinyIntVal(
    FunctionContext* c, const TinyIntVal& v1, const TinyIntVal& v2) {
  if (v1.is_null || v2.is_null) return BooleanVal::null();
  return BooleanVal(v1.val < v2.val);
}

// Lt_SmallIntVal_SmallIntVal 展开实现
BooleanVal Operators::Lt_SmallIntVal_SmallIntVal(
    FunctionContext* c, const SmallIntVal& v1, const SmallIntVal& v2) {
  if (v1.is_null || v2.is_null) return BooleanVal::null();
  return BooleanVal(v1.val < v2.val);
}

// Lt_IntVal_IntVal 展开实现
BooleanVal Operators::Lt_IntVal_IntVal(
    FunctionContext* c, const IntVal& v1, const IntVal& v2) {
  if (v1.is_null || v2.is_null) return BooleanVal::null();
  return BooleanVal(v1.val < v2.val);
}

// Lt_BigIntVal_BigIntVal 展开实现
BooleanVal Operators::Lt_BigIntVal_BigIntVal(
    FunctionContext* c, const BigIntVal& v1, const BigIntVal& v2) {
  if (v1.is_null || v2.is_null) return BooleanVal::null();
  return BooleanVal(v1.val < v2.val);
}

// Lt_FloatVal_FloatVal 展开实现
BooleanVal Operators::Lt_FloatVal_FloatVal(
    FunctionContext* c, const FloatVal& v1, const FloatVal& v2) {
  if (v1.is_null || v2.is_null) return BooleanVal::null();
  return BooleanVal(v1.val < v2.val);
}

// Lt_DoubleVal_DoubleVal 展开实现
BooleanVal Operators::Lt_DoubleVal_DoubleVal(
    FunctionContext* c, const DoubleVal& v1, const DoubleVal& v2) {
  if (v1.is_null || v2.is_null) return BooleanVal::null();
  return BooleanVal(v1.val < v2.val);
}

// Lt_DateVal_DateVal 展开实现
BooleanVal Operators::Lt_DateVal_DateVal(
    FunctionContext* c, const DateVal& v1, const DateVal& v2) {
  if (v1.is_null || v2.is_null) return BooleanVal::null();
  return BooleanVal(v1.val < v2.val);
}

// Ge_BooleanVal_BooleanVal 展开实现
BooleanVal Operators::Ge_BooleanVal_BooleanVal(
    FunctionContext* c, const BooleanVal& v1, const BooleanVal& v2) {
  if (v1.is_null || v2.is_null) return BooleanVal::null();
  return BooleanVal(v1.val >= v2.val);
}

// Ge_TinyIntVal_TinyIntVal 展开实现
BooleanVal Operators::Ge_TinyIntVal_TinyIntVal(
    FunctionContext* c, const TinyIntVal& v1, const TinyIntVal& v2) {
  if (v1.is_null || v2.is_null) return BooleanVal::null();
  return BooleanVal(v1.val >= v2.val);
}

// Ge_SmallIntVal_SmallIntVal 展开实现
BooleanVal Operators::Ge_SmallIntVal_SmallIntVal(
    FunctionContext* c, const SmallIntVal& v1, const SmallIntVal& v2) {
  if (v1.is_null || v2.is_null) return BooleanVal::null();
  return BooleanVal(v1.val >= v2.val);
}

// Ge_IntVal_IntVal 展开实现
BooleanVal Operators::Ge_IntVal_IntVal(
    FunctionContext* c, const IntVal& v1, const IntVal& v2) {
  if (v1.is_null || v2.is_null) return BooleanVal::null();
  return BooleanVal(v1.val >= v2.val);
}

// Ge_BigIntVal_BigIntVal 展开实现
BooleanVal Operators::Ge_BigIntVal_BigIntVal(
    FunctionContext* c, const BigIntVal& v1, const BigIntVal& v2) {
  if (v1.is_null || v2.is_null) return BooleanVal::null();
  return BooleanVal(v1.val >= v2.val);
}

// Ge_FloatVal_FloatVal 展开实现
BooleanVal Operators::Ge_FloatVal_FloatVal(
    FunctionContext* c, const FloatVal& v1, const FloatVal& v2) {
  if (v1.is_null || v2.is_null) return BooleanVal::null();
  return BooleanVal(v1.val >= v2.val);
}

// Ge_DoubleVal_DoubleVal 展开实现
BooleanVal Operators::Ge_DoubleVal_DoubleVal(
    FunctionContext* c, const DoubleVal& v1, const DoubleVal& v2) {
  if (v1.is_null || v2.is_null) return BooleanVal::null();
  return BooleanVal(v1.val >= v2.val);
}

// Ge_DateVal_DateVal 展开实现
BooleanVal Operators::Ge_DateVal_DateVal(
    FunctionContext* c, const DateVal& v1, const DateVal& v2) {
  if (v1.is_null || v2.is_null) return BooleanVal::null();
  return BooleanVal(v1.val >= v2.val);
}

// Le_BooleanVal_BooleanVal 展开实现
BooleanVal Operators::Le_BooleanVal_BooleanVal(
    FunctionContext* c, const BooleanVal& v1, const BooleanVal& v2) {
  if (v1.is_null || v2.is_null) return BooleanVal::null();
  return BooleanVal(v1.val <= v2.val);
}

// Le_TinyIntVal_TinyIntVal 展开实现
BooleanVal Operators::Le_TinyIntVal_TinyIntVal(
    FunctionContext* c, const TinyIntVal& v1, const TinyIntVal& v2) {
  if (v1.is_null || v2.is_null) return BooleanVal::null();
  return BooleanVal(v1.val <= v2.val);
}

// Le_SmallIntVal_SmallIntVal 展开实现
BooleanVal Operators::Le_SmallIntVal_SmallIntVal(
    FunctionContext* c, const SmallIntVal& v1, const SmallIntVal& v2) {
  if (v1.is_null || v2.is_null) return BooleanVal::null();
  return BooleanVal(v1.val <= v2.val);
}

// Le_IntVal_IntVal 展开实现
BooleanVal Operators::Le_IntVal_IntVal(
    FunctionContext* c, const IntVal& v1, const IntVal& v2) {
  if (v1.is_null || v2.is_null) return BooleanVal::null();
  return BooleanVal(v1.val <= v2.val);
}

// Le_BigIntVal_BigIntVal 展开实现
BooleanVal Operators::Le_BigIntVal_BigIntVal(
    FunctionContext* c, const BigIntVal& v1, const BigIntVal& v2) {
  if (v1.is_null || v2.is_null) return BooleanVal::null();
  return BooleanVal(v1.val <= v2.val);
}

// Le_FloatVal_FloatVal 展开实现
BooleanVal Operators::Le_FloatVal_FloatVal(
    FunctionContext* c, const FloatVal& v1, const FloatVal& v2) {
  if (v1.is_null || v2.is_null) return BooleanVal::null();
  return BooleanVal(v1.val <= v2.val);
}

// Le_DoubleVal_DoubleVal 展开实现
BooleanVal Operators::Le_DoubleVal_DoubleVal(
    FunctionContext* c, const DoubleVal& v1, const DoubleVal& v2) {
  if (v1.is_null || v2.is_null) return BooleanVal::null();
  return BooleanVal(v1.val <= v2.val);
}

// Le_DateVal_DateVal 展开实现
BooleanVal Operators::Le_DateVal_DateVal(
    FunctionContext* c, const DateVal& v1, const DateVal& v2) {
  if (v1.is_null || v2.is_null) return BooleanVal::null();
  return BooleanVal(v1.val <= v2.val);
}

// ------------------------- 4.2 字符串/时间戳/Char 类型比较 -------------------------
// Eq_StringVal_StringVal 展开实现
BooleanVal Operators::Eq_StringVal_StringVal(
    FunctionContext* c, const StringVal& v1, const StringVal& v2) {
  if (v1.is_null || v2.is_null) return BooleanVal::null();
  StringValue iv1 = StringValue::FromStringVal(v1);
  StringValue iv2 = StringValue::FromStringVal(v2);
  return BooleanVal(iv1 == iv2);
}

// Eq_TimestampVal_TimestampVal 展开实现
BooleanVal Operators::Eq_TimestampVal_TimestampVal(
    FunctionContext* c, const TimestampVal& v1, const TimestampVal& v2) {
  if (v1.is_null || v2.is_null) return BooleanVal::null();
  TimestampValue iv1 = TimestampValue::FromTimestampVal(v1);
  TimestampValue iv2 = TimestampValue::FromTimestampVal(v2);
  return BooleanVal(iv1 == iv2);
}

// Eq_Char_Char 展开实现
BooleanVal Operators::Eq_Char_Char(
    FunctionContext* c, const StringVal& v1, const StringVal& v2) {
  if (v1.is_null || v2.is_null) return BooleanVal::null();
  StringValue iv1 = StringValue::FromStringVal(v1);
  StringValue iv2 = StringValue::FromStringVal(v2);
  iv1.SetLen(StringValue::UnpaddedCharLength(iv1.Ptr(), c->GetArgType(0)->len));
  iv2.SetLen(StringValue::UnpaddedCharLength(iv2.Ptr(), c->GetArgType(1)->len));
  return BooleanVal(iv1 == iv2);
}

// Ne_StringVal_StringVal 展开实现
BooleanVal Operators::Ne_StringVal_StringVal(
    FunctionContext* c, const StringVal& v1, const StringVal& v2) {
  if (v1.is_null || v2.is_null) return BooleanVal::null();
  StringValue iv1 = StringValue::FromStringVal(v1);
  StringValue iv2 = StringValue::FromStringVal(v2);
  return BooleanVal(iv1 != iv2);
}

// Ne_TimestampVal_TimestampVal 展开实现
BooleanVal Operators::Ne_TimestampVal_TimestampVal(
    FunctionContext* c, const TimestampVal& v1, const TimestampVal& v2) {
  if (v1.is_null || v2.is_null) return BooleanVal::null();
  TimestampValue iv1 = TimestampValue::FromTimestampVal(v1);
  TimestampValue iv2 = TimestampValue::FromTimestampVal(v2);
  return BooleanVal(iv1 != iv2);
}

// Ne_Char_Char 展开实现
BooleanVal Operators::Ne_Char_Char(
    FunctionContext* c, const StringVal& v1, const StringVal& v2) {
  if (v1.is_null || v2.is_null) return BooleanVal::null();
  StringValue iv1 = StringValue::FromStringVal(v1);
  StringValue iv2 = StringValue::FromStringVal(v2);
  iv1.SetLen(StringValue::UnpaddedCharLength(iv1.Ptr(), c->GetArgType(0)->len));
  iv2.SetLen(StringValue::UnpaddedCharLength(iv2.Ptr(), c->GetArgType(1)->len));
  return BooleanVal(iv1 != iv2);
}

// Gt_StringVal_StringVal 展开实现
BooleanVal Operators::Gt_StringVal_StringVal(
    FunctionContext* c, const StringVal& v1, const StringVal& v2) {
  if (v1.is_null || v2.is_null) return BooleanVal::null();
  StringValue iv1 = StringValue::FromStringVal(v1);
  StringValue iv2 = StringValue::FromStringVal(v2);
  return BooleanVal(iv1 > iv2);
}

// Gt_TimestampVal_TimestampVal 展开实现
BooleanVal Operators::Gt_TimestampVal_TimestampVal(
    FunctionContext* c, const TimestampVal& v1, const TimestampVal& v2) {
  if (v1.is_null || v2.is_null) return BooleanVal::null();
  TimestampValue iv1 = TimestampValue::FromTimestampVal(v1);
  TimestampValue iv2 = TimestampValue::FromTimestampVal(v2);
  return BooleanVal(iv1 > iv2);
}

// Gt_Char_Char 展开实现
BooleanVal Operators::Gt_Char_Char(
    FunctionContext* c, const StringVal& v1, const StringVal& v2) {
  if (v1.is_null || v2.is_null) return BooleanVal::null();
  StringValue iv1 = StringValue::FromStringVal(v1);
  StringValue iv2 = StringValue::FromStringVal(v2);
  iv1.SetLen(StringValue::UnpaddedCharLength(iv1.Ptr(), c->GetArgType(0)->len));
  iv2.SetLen(StringValue::UnpaddedCharLength(iv2.Ptr(), c->GetArgType(1)->len));
  return BooleanVal(iv1 > iv2);
}

// Lt_StringVal_StringVal 展开实现
BooleanVal Operators::Lt_StringVal_StringVal(
    FunctionContext* c, const StringVal& v1, const StringVal& v2) {
  if (v1.is_null || v2.is_null) return BooleanVal::null();
  StringValue iv1 = StringValue::FromStringVal(v1);
  StringValue iv2 = StringValue::FromStringVal(v2);
  return BooleanVal(iv1 < iv2);
}

// Lt_TimestampVal_TimestampVal 展开实现
BooleanVal Operators::Lt_TimestampVal_TimestampVal(
    FunctionContext* c, const TimestampVal& v1, const TimestampVal& v2) {
  if (v1.is_null || v2.is_null) return BooleanVal::null();
  TimestampValue iv1 = TimestampValue::FromTimestampVal(v1);
  TimestampValue iv2 = TimestampValue::FromTimestampVal(v2);
  return BooleanVal(iv1 < iv2);
}

// Lt_Char_Char 展开实现
BooleanVal Operators::Lt_Char_Char(
    FunctionContext* c, const StringVal& v1, const StringVal& v2) {
  if (v1.is_null || v2.is_null) return BooleanVal::null();
  StringValue iv1 = StringValue::FromStringVal(v1);
  StringValue iv2 = StringValue::FromStringVal(v2);
  iv1.SetLen(StringValue::UnpaddedCharLength(iv1.Ptr(), c->GetArgType(0)->len));
  iv2.SetLen(StringValue::UnpaddedCharLength(iv2.Ptr(), c->GetArgType(1)->len));
  return BooleanVal(iv1 < iv2);
}

// Ge_StringVal_StringVal 展开实现
BooleanVal Operators::Ge_StringVal_StringVal(
    FunctionContext* c, const StringVal& v1, const StringVal& v2) {
  if (v1.is_null || v2.is_null) return BooleanVal::null();
  StringValue iv1 = StringValue::FromStringVal(v1);
  StringValue iv2 = StringValue::FromStringVal(v2);
  return BooleanVal(iv1 >= iv2);
}

// Ge_TimestampVal_TimestampVal 展开实现
BooleanVal Operators::Ge_TimestampVal_TimestampVal(
    FunctionContext* c, const TimestampVal& v1, const TimestampVal& v2) {
  if (v1.is_null || v2.is_null) return BooleanVal::null();
  TimestampValue iv1 = TimestampValue::FromTimestampVal(v1);
  TimestampValue iv2 = TimestampValue::FromTimestampVal(v2);
  return BooleanVal(iv1 >= iv2);
}

// Ge_Char_Char 展开实现
BooleanVal Operators::Ge_Char_Char(
    FunctionContext* c, const StringVal& v1, const StringVal& v2) {
  if (v1.is_null || v2.is_null) return BooleanVal::null();
  StringValue iv1 = StringValue::FromStringVal(v1);
  StringValue iv2 = StringValue::FromStringVal(v2);
  iv1.SetLen(StringValue::UnpaddedCharLength(iv1.Ptr(), c->GetArgType(0)->len));
  iv2.SetLen(StringValue::UnpaddedCharLength(iv2.Ptr(), c->GetArgType(1)->len));
  return BooleanVal(iv1 >= iv2);
}

// Le_StringVal_StringVal 展开实现
BooleanVal Operators::Le_StringVal_StringVal(
    FunctionContext* c, const StringVal& v1, const StringVal& v2) {
  if (v1.is_null || v2.is_null) return BooleanVal::null();
  StringValue iv1 = StringValue::FromStringVal(v1);
  StringValue iv2 = StringValue::FromStringVal(v2);
  return BooleanVal(iv1 <= iv2);
}

// Le_TimestampVal_TimestampVal 展开实现
BooleanVal Operators::Le_TimestampVal_TimestampVal(
    FunctionContext* c, const TimestampVal& v1, const TimestampVal& v2) {
  if (v1.is_null || v2.is_null) return BooleanVal::null();
  TimestampValue iv1 = TimestampValue::FromTimestampVal(v1);
  TimestampValue iv2 = TimestampValue::FromTimestampVal(v2);
  return BooleanVal(iv1 <= iv2);
}

// Le_Char_Char 展开实现
BooleanVal Operators::Le_Char_Char(
    FunctionContext* c, const StringVal& v1, const StringVal& v2) {
  if (v1.is_null || v2.is_null) return BooleanVal::null();
  StringValue iv1 = StringValue::FromStringVal(v1);
  StringValue iv2 = StringValue::FromStringVal(v2);
  iv1.SetLen(StringValue::UnpaddedCharLength(iv1.Ptr(), c->GetArgType(0)->len));
  iv2.SetLen(StringValue::UnpaddedCharLength(iv2.Ptr(), c->GetArgType(1)->len));
  return BooleanVal(iv1 <= iv2);
}

// ========================== 5. NULL 安全比较（DistinctFrom/NotDistinct）==========================
// ------------------------- 5.1 数值类型 NULL 安全比较 -------------------------
// DistinctFrom_BooleanVal_BooleanVal 展开实现
BooleanVal Operators::DistinctFrom_BooleanVal_BooleanVal(
    FunctionContext* c, const BooleanVal& v1, const BooleanVal& v2) {
  if (v1.is_null) return BooleanVal(!v2.is_null);
  if (v2.is_null) return BooleanVal(true);
  return BooleanVal(v1.val != v2.val);
}

// DistinctFrom_TinyIntVal_TinyIntVal 展开实现
BooleanVal Operators::DistinctFrom_TinyIntVal_TinyIntVal(
    FunctionContext* c, const TinyIntVal& v1, const TinyIntVal& v2) {
  if (v1.is_null) return BooleanVal(!v2.is_null);
  if (v2.is_null) return BooleanVal(true);
  return BooleanVal(v1.val != v2.val);
}

// DistinctFrom_SmallIntVal_SmallIntVal 展开实现
BooleanVal Operators::DistinctFrom_SmallIntVal_SmallIntVal(
    FunctionContext* c, const SmallIntVal& v1, const SmallIntVal& v2) {
  if (v1.is_null) return BooleanVal(!v2.is_null);
  if (v2.is_null) return BooleanVal(true);
  return BooleanVal(v1.val != v2.val);
}

// DistinctFrom_IntVal_IntVal 展开实现
BooleanVal Operators::DistinctFrom_IntVal_IntVal(
    FunctionContext* c, const IntVal& v1, const IntVal& v2) {
  if (v1.is_null) return BooleanVal(!v2.is_null);
  if (v2.is_null) return BooleanVal(true);
  return BooleanVal(v1.val != v2.val);
}

// DistinctFrom_BigIntVal_BigIntVal 展开实现
BooleanVal Operators::DistinctFrom_BigIntVal_BigIntVal(
    FunctionContext* c, const BigIntVal& v1, const BigIntVal& v2) {
  if (v1.is_null) return BooleanVal(!v2.is_null);
  if (v2.is_null) return BooleanVal(true);
  return BooleanVal(v1.val != v2.val);
}

// DistinctFrom_FloatVal_FloatVal 展开实现
BooleanVal Operators::DistinctFrom_FloatVal_FloatVal(
    FunctionContext* c, const FloatVal& v1, const FloatVal& v2) {
  if (v1.is_null) return BooleanVal(!v2.is_null);
  if (v2.is_null) return BooleanVal(true);
  return BooleanVal(v1.val != v2.val);
}

// DistinctFrom_DoubleVal_DoubleVal 展开实现
BooleanVal Operators::DistinctFrom_DoubleVal_DoubleVal(
    FunctionContext* c, const DoubleVal& v1, const DoubleVal& v2) {
  if (v1.is_null) return BooleanVal(!v2.is_null);
  if (v2.is_null) return BooleanVal(true);
  return BooleanVal(v1.val != v2.val);
}

// DistinctFrom_DateVal_DateVal 展开实现
BooleanVal Operators::DistinctFrom_DateVal_DateVal(
    FunctionContext* c, const DateVal& v1, const DateVal& v2) {
  if (v1.is_null) return BooleanVal(!v2.is_null);
  if (v2.is_null) return BooleanVal(true);
  return BooleanVal(v1.val != v2.val);
}

// NotDistinct_BooleanVal_BooleanVal 展开实现
BooleanVal Operators::NotDistinct_BooleanVal_BooleanVal(
    FunctionContext* c, const BooleanVal& v1, const BooleanVal& v2) {
  if (v1.is_null) return BooleanVal(v2.is_null);
  if (v2.is_null) return BooleanVal(false);
  return BooleanVal(v1.val == v2.val);
}

// NotDistinct_TinyIntVal_TinyIntVal 展开实现
BooleanVal Operators::NotDistinct_TinyIntVal_TinyIntVal(
    FunctionContext* c, const TinyIntVal& v1, const TinyIntVal& v2) {
  if (v1.is_null) return BooleanVal(v2.is_null);
  if (v2.is_null) return BooleanVal(false);
  return BooleanVal(v1.val == v2.val);
}

// NotDistinct_SmallIntVal_SmallIntVal 展开实现
BooleanVal Operators::NotDistinct_SmallIntVal_SmallIntVal(
    FunctionContext* c, const SmallIntVal& v1, const SmallIntVal& v2) {
  if (v1.is_null) return BooleanVal(v2.is_null);
  if (v2.is_null) return BooleanVal(false);
  return BooleanVal(v1.val == v2.val);
}

// NotDistinct_IntVal_IntVal 展开实现
BooleanVal Operators::NotDistinct_IntVal_IntVal(
    FunctionContext* c, const IntVal& v1, const IntVal& v2) {
  if (v1.is_null) return BooleanVal(v2.is_null);
  if (v2.is_null) return BooleanVal(false);
  return BooleanVal(v1.val == v2.val);
}

// NotDistinct_BigIntVal_BigIntVal 展开实现
BooleanVal Operators::NotDistinct_BigIntVal_BigIntVal(
    FunctionContext* c, const BigIntVal& v1, const BigIntVal& v2) {
  if (v1.is_null) return BooleanVal(v2.is_null);
  if (v2.is_null) return BooleanVal(false);
  return BooleanVal(v1.val == v2.val);
}

// NotDistinct_FloatVal_FloatVal 展开实现
BooleanVal Operators::NotDistinct_FloatVal_FloatVal(
    FunctionContext* c, const FloatVal& v1, const FloatVal& v2) {
  if (v1.is_null) return BooleanVal(v2.is_null);
  if (v2.is_null) return BooleanVal(false);
  return BooleanVal(v1.val == v2.val);
}

// NotDistinct_DoubleVal_DoubleVal 展开实现
BooleanVal Operators::NotDistinct_DoubleVal_DoubleVal(
    FunctionContext* c, const DoubleVal& v1, const DoubleVal& v2) {
  if (v1.is_null) return BooleanVal(v2.is_null);
  if (v2.is_null) return BooleanVal(false);
  return BooleanVal(v1.val == v2.val);
}

// NotDistinct_DateVal_DateVal 展开实现
BooleanVal Operators::NotDistinct_DateVal_DateVal(
    FunctionContext* c, const DateVal& v1, const DateVal& v2) {
  if (v1.is_null) return BooleanVal(v2.is_null);
  if (v2.is_null) return BooleanVal(false);
  return BooleanVal(v1.val == v2.val);
}

// ------------------------- 5.2 字符串/时间戳/Char 类型 NULL 安全比较 -------------------------
// DistinctFrom_StringVal_StringVal 展开实现
BooleanVal Operators::DistinctFrom_StringVal_StringVal(
    FunctionContext* c, const StringVal& v1, const StringVal& v2) {
  if (v1.is_null) return BooleanVal(!v2.is_null);
  if (v2.is_null) return BooleanVal(true);
  StringValue iv1 = StringValue::FromStringVal(v1);
  StringValue iv2 = StringValue::FromStringVal(v2);
  return BooleanVal(iv1 != iv2);
}

// DistinctFrom_TimestampVal_TimestampVal 展开实现
BooleanVal Operators::DistinctFrom_TimestampVal_TimestampVal(
    FunctionContext* c, const TimestampVal& v1, const TimestampVal& v2) {
  if (v1.is_null) return BooleanVal(!v2.is_null);
  if (v2.is_null) return BooleanVal(true);
  TimestampValue iv1 = TimestampValue::FromTimestampVal(v1);
  TimestampValue iv2 = TimestampValue::FromTimestampVal(v2);
  return BooleanVal(iv1 != iv2);
}

// DistinctFrom_Char_Char 展开实现
BooleanVal Operators::DistinctFrom_Char_Char(
    FunctionContext* c, const StringVal& v1, const StringVal& v2) {
  if (v1.is_null) return BooleanVal(!v2.is_null);
  if (v2.is_null) return BooleanVal(true);
  StringValue iv1 = StringValue::FromStringVal(v1);
  StringValue iv2 = StringValue::FromStringVal(v2);
  iv1.SetLen(StringValue::UnpaddedCharLength(iv1.Ptr(), c->GetArgType(0)->len));
  iv2.SetLen(StringValue::UnpaddedCharLength(iv2.Ptr(), c->GetArgType(1)->len));
  return BooleanVal(iv1 != iv2);
}

// NotDistinct_StringVal_StringVal 展开实现
BooleanVal Operators::NotDistinct_StringVal_StringVal(
    FunctionContext* c, const StringVal& v1, const StringVal& v2) {
  if (v1.is_null) return BooleanVal(v2.is_null);
  if (v2.is_null) return BooleanVal(false);
  StringValue iv1 = StringValue::FromStringVal(v1);
  StringValue iv2 = StringValue::FromStringVal(v2);
  return BooleanVal(iv1 == iv2);
}

// NotDistinct_TimestampVal_TimestampVal 展开实现
BooleanVal Operators::NotDistinct_TimestampVal_TimestampVal(
    FunctionContext* c, const TimestampVal& v1, const TimestampVal& v2) {
  if (v1.is_null) return BooleanVal(v2.is_null);
  if (v2.is_null) return BooleanVal(false);
  TimestampValue iv1 = TimestampValue::FromTimestampVal(v1);
  TimestampValue iv2 = TimestampValue::FromTimestampVal(v2);
  return BooleanVal(iv1 == iv2);
}

// NotDistinct_Char_Char 展开实现
BooleanVal Operators::NotDistinct_Char_Char(
    FunctionContext* c, const StringVal& v1, const StringVal& v2) {
  if (v1.is_null) return BooleanVal(v2.is_null);
  if (v2.is_null) return BooleanVal(false);
  StringValue iv1 = StringValue::FromStringVal(v1);
  StringValue iv2 = StringValue::FromStringVal(v2);
  iv1.SetLen(StringValue::UnpaddedCharLength(iv1.Ptr(), c->GetArgType(0)->len));
  iv2.SetLen(StringValue::UnpaddedCharLength(iv2.Ptr(), c->GetArgType(1)->len));
  return BooleanVal(iv1 == iv2);
}

} // namespace impala
```

### 总结
展开后的代码完整呈现了 Impala 运算符函数的实现逻辑，核心关键点如下：
1. **NULL 语义优先**：所有函数第一步均检查 `is_null`，符合 SQL 中 `NULL 参与运算结果为 NULL` 的核心规则；
2. **类型差异化处理**：
   - 整数加减乘通过 `ArithmeticUtil::AsUnsigned` 处理溢出（C++ 有符号溢出未定义，无符号溢出安全）；
   - 除法/取模额外检查除数为 0，避免运行时错误；
   - Char 类型比较需先去除填充空格，保证语义正确；
3. **宏的核心价值**：批量生成同逻辑不同类型的函数，减少代码冗余，保证所有类型的运算逻辑一致；
4. **NULL 安全比较**：`DistinctFrom/NotDistinct` 专门处理 NULL 参与的比较（普通比较中 NULL 会返回 NULL，而 NULL 安全比较会返回明确的 true/false）。

你可以对照展开后的代码，清晰看到每个函数的输入处理、核心运算、NULL 语义处理和返回逻辑，理解 Impala 如何基于 `*Val` 接口层实现统一且合规的运算符体系。
***********************************************************************************

我们可以看到这些函数通常输入参数都是*Val类型（impala_udf::AnyVal子类型)，返回结果通常也是*Val类型（impala_udf::AnyVal子类型）。具体到每个子类型，在函数体中有些需要转换成*Value类型（DecimalValue，StringValue,TimestampValue），而有些不需要转换成*Value类型，比如BoolenVal,TinyIntVal,SmallIntVal,IntVal,BigInt,FloatVal,DoubleVal，DateValue

### 核心观察确认
你的总结**完全精准**！Impala 中 `*Val` 类型是否需要转换为 `*Value` 类型，核心取决于**该 `*Val` 的内部存储形态 + 运算所需的语义复杂度** —— 简单类型的 `*Val` 可直接操作 `val` 成员，复杂类型则必须通过 `*Value` 封装运算逻辑。下面我们拆解这两类 `*Val` 的本质差异，以及“转换/不转换”的底层原因：

## 一、先明确：两类 `*Val` 的划分（修正你的小笔误）
先纠正一个你提到的小笔误（不影响核心）：`DateValue` 是笔误，正确的是 `DateVal`（无 `DateValue` 类型）；且 `BigInt` 应为 `BigIntVal`。

### 最终清晰的两类划分
| 类别                | 具体 `*Val` 类型                                                                 | 核心特征                          |
|---------------------|----------------------------------------------------------------------------------|-----------------------------------|
| 无需转 `*Value`     | BooleanVal、TinyIntVal、SmallIntVal、IntVal、BigIntVal、FloatVal、DoubleVal、DateVal | 存储极简 + 运算无额外语义         |
| 必须转 `*Value`     | DecimalVal、StringVal、TimestampVal                                               | 存储复杂 + 运算需额外语义/规则    |

## 二、为什么部分 `*Val` 无需转换为 `*Value`？（核心：原生存储 + 无额外语义）
这类 `*Val` 的设计目标是「**接口层（承载 NULL 语义）+ 原生数值（直接运算）**」，其 `val` 成员就是**硬件原生类型**，无需中转即可直接运算：

### 1. 存储形态：`val` 直接对应硬件原生类型
这类 `*Val` 的内部结构极度简单，`val` 成员就是 CPU 可直接运算的原生类型，无任何间接存储/压缩/封装：
```cpp
// 以 TinyIntVal 为例（所有简单类型的*Val结构一致）
struct TinyIntVal : public AnyVal {
  int8_t val;  // 直接是原生int8_t，无任何封装
  bool is_null;

  static TinyIntVal null() { return TinyIntVal(0, true); }
  TinyIntVal(int8_t v) : val(v), is_null(false) {}
  // ... 其他构造函数
};

// DateVal 同理：val是int32_t（存储自epoch的天数）
struct DateVal : public AnyVal {
  int32_t val;  // 直接是原生int32_t，代表“天数”
  bool is_null;
  // ...
};
```
- 运算时只需：① 检查 `is_null`（SQL 语义）→ ② 直接操作 `val`（原生运算）→ ③ 返回新的 `*Val`；
- 比如 `Add_IntVal_IntVal`：`return IntVal(v1.val + v2.val)`，CPU 可直接执行加法，无任何额外成本。

### 2. 运算语义：无额外规则，直接运算即可
这类类型的运算无需处理「刻度对齐、字符填充、时间精度、时区」等额外规则：
- 整数/浮点运算：`v1.val + v2.val` 就是 SQL 语义的加法（仅需处理 NULL 和溢出）；
- DateVal 运算：`val` 是“自 epoch 的天数”，`v1.val + v2.val` 就是“天数相加”，直接对应 SQL 的日期运算；
- BooleanVal 比较：`v1.val == v2.val` 直接对应 SQL 的布尔比较，无额外规则。

### 3. 性能：零成本运算，无需中转
这类 `*Val` 的运算本质是「NULL 语义检查 + 原生数值运算」，CPU 可直接操作寄存器中的 `val` 值，无需内存拷贝/类型转换，是极致高效的。

## 三、为什么部分 `*Val` 必须转换为 `*Value`？（核心：复杂存储 + 语义依赖）
这类 `*Val` 的 `val` 成员**不是直接可运算的原生值**，且运算需要遵循复杂的 SQL 语义规则 —— `*Value` 是专门为「**复杂类型的运算逻辑**」设计的载体，`*Val` 仅作为「框架对接的接口层」。

### 1. 存储形态：`val` 是“间接/压缩/封装值”，无法直接运算
以三类典型类型为例：
#### （1）StringVal：`val` 是“指针+长度”，不是直接可比较的字符
```cpp
struct StringVal : public AnyVal {
  int len;    // 字符串长度
  const char* ptr;  // 字符串指针（间接存储，非原生类型）
  bool is_null;

  // 运算时无法直接用 ptr/len 比较，必须转 StringValue
  // StringValue 封装了字符比较、填充去除等逻辑
};
```
- 比如 Char 类型的比较：需要先去除末尾填充的空格（`UnpaddedCharLength`），再比较字符 —— 这些逻辑封装在 `StringValue` 中，`StringVal` 本身无法处理。

#### （2）TimestampVal：`val` 是压缩的时间戳，需解包后运算
```cpp
struct TimestampVal : public AnyVal {
  int64_t val;  // 压缩存储（日期+时间+微秒），不是原生可运算的“时间值”
  bool is_null;

  // 运算时需转 TimestampValue，解包为“年/月/日/时/分/秒/微秒”后再比较/运算
};
```
- 比如 TimestampVal 的比较：直接比较 `val` 可能因压缩规则导致错误，必须通过 `TimestampValue` 解包后，按 SQL 时间语义比较。

#### （3）DecimalVal：`val` 是 union，需按精度拆分后运算
```cpp
struct DecimalVal : public AnyVal {
  union {
    int32_t val4;
    int64_t val8;
    __int128 val16;
  };  // 不同精度的数值存在union中，无刻度信息
  bool is_null;

  // 运算时需转 Decimal4/8/16Value，结合“刻度/精度”做算术运算
};
```
- 比如 DecimalVal 的加法：需要先对齐刻度（如 Decimal(10,2) + Decimal(8,3)），运算后舍入，这些逻辑封装在 `DecimalValue` 中，`DecimalVal` 仅存储数值，无刻度信息。

### 2. 运算语义：必须遵循 SQL 规范的额外规则
这类类型的运算不是“原生数值运算”，而是“带语义的业务运算”：
- StringVal/Char：需处理字符编码、填充空格、空字符串 vs NULL 等规则；
- TimestampVal：需处理时区、闰年、夏令时、微秒精度等规则；
- DecimalVal：需处理刻度对齐、舍入模式、溢出检测等金融级规则；
- 这些规则全部封装在 `*Value` 中，`*Val` 仅负责“接收参数 + 返回结果”，不处理具体运算逻辑。

### 3. 架构：`*Value` 是“运算核心”，`*Val` 是“接口外壳”
- `*Val`：面向 Impala 执行框架（ExprEvaluator/UDF），统一接口 + 承载 NULL 语义，不关心运算细节；
- `*Value`：面向“复杂类型的运算逻辑”，封装所有 SQL 语义规则，是真正的“运算载体”；
- 转换的本质：将「框架友好的接口层（*Val）」转换为「运算友好的核心层（*Value）」，运算完成后再转回 `*Val` 返回。

## 四、总结：核心规则（`*Val` 是否转 `*Value` 的判断依据）
1. **存储维度**：若 `*Val` 的 `val` 是「硬件原生类型」（int8/int16/int32/int64/float/double），无需转换；若 `val` 是「间接/压缩/封装值」（指针、union、压缩编码），必须转换；
2. **语义维度**：若运算仅需“原生数值运算”（加减乘除/比较），无需转换；若运算需处理「SQL 额外语义」（刻度、字符填充、时间规则），必须转换；
3. **设计维度**：`*Value` 是复杂类型的“运算专用载体”，`*Val` 是所有类型的“框架统一接口” —— 简单类型的 `*Val` 可兼顾“接口+运算”，复杂类型则需拆分。

简单来说：**`*Val` 是否需要转 `*Value`，取决于该类型的运算是否“超出了原生数值运算的范畴”** —— 简单类型的运算就是原生数值运算，复杂类型的运算则是“带 SQL 语义的业务运算”，必须通过 `*Value` 封装。

***************************************************************************
### 核心观察确认
你通过分析Impala的`decimal-operator`和`operators`相关源码，精准捕捉到了Impala UDF运算符体系的核心设计规律：**所有运算符函数对外统一以`impala_udf::AnyVal`的子类（`*Val`类型）作为输入输出，但内部处理时，基础数值类型可直接操作`*Val`的`val`成员，无需转换；而Decimal/String/Timestamp等复杂类型必须先转为对应的`*Value`类型完成运算，再转回`*Val`类型**。下面结合源码拆解这一设计的实现细节和底层原因。

## 一、基础类型（无需转`*Value`）：直接操作`*Val`的原生数值
对于`BooleanVal`/`TinyIntVal`/`SmallIntVal`/`IntVal`/`BigIntVal`/`FloatVal`/`DoubleVal`/`DateVal`这类基础类型，源码中完全无需转换为`*Value`，直接操作`*Val`的`val`成员即可完成运算。

### 1. 代码实现：极简的宏定义直接操作`val`
在`operators-ir.cc`中，基础类型的运算符通过宏定义直接实现，核心逻辑是「检查NULL → 操作`val` → 返回新`*Val`」：
```cpp
// 基础算术运算宏：直接操作val成员
#define BINARY_OP_FN(NAME, TYPE, OP) \
  TYPE Operators::NAME##_##TYPE##_##TYPE(\
      FunctionContext* c, const TYPE& v1, const TYPE& v2) {\
    if (v1.is_null || v2.is_null) return TYPE::null();\
    return TYPE(v1.val OP v2.val);\
  }

// 示例：Add_TinyIntVal_TinyIntVal的实际展开
TinyIntVal Operators::Add_TinyIntVal_TinyIntVal(
    FunctionContext* c, const TinyIntVal& v1, const TinyIntVal& v2) {
  if (v1.is_null || v2.is_null) return TinyIntVal::null();
  return TinyIntVal(v1.val + v2.val); // 直接操作val（int8_t）
}
```

### 2. 底层原因：`val`是原生硬件类型，无额外语义
这类`*Val`的`val`成员本质是CPU可直接运算的**原生数值类型**（如`TinyIntVal::val`是`int8_t`，`BigIntVal::val`是`int64_t`，`DateVal::val`是`int32_t`），且运算无需处理额外SQL语义：
- 数值运算：`v1.val + v2.val`直接对应SQL的加法，仅需处理NULL和溢出；
- 比较运算：`v1.val == v2.val`直接对应SQL的相等判断，无字符填充、刻度对齐等额外逻辑；
- 即使是溢出处理（如`BINARY_OP_AS_UNSIGNED_FN`），也是直接对`val`做无符号转换，无需`*Value`中转。

## 二、复杂类型（必须转`*Value`）：`*Value`是运算的核心载体
对于`DecimalVal`/`StringVal`/`TimestampVal`这类复杂类型，`*Val`仅作为「对外接口层」，真正的运算逻辑全部封装在`*Value`中，源码中必须先完成`*Val → *Value`的转换。

### 1. DecimalVal：依赖`Decimal4/8/16Value`处理精度/刻度
`DecimalVal`的`val`是`union`（`val4`/`val8`/`val16`），仅存储无刻度的原始数值，运算需结合精度/刻度规则，因此必须转`Decimal4/8/16Value`：
#### （1）核心转换逻辑
在`decimal-operator-ir.cc`中，通过`GetDecimal4Value`/`GetDecimal8Value`/`GetDecimal16Value`将`DecimalVal`转为`*Value`：
```cpp
// 示例：将DecimalVal转为Decimal4Value
static inline Decimal4Value GetDecimal4Value(
    const DecimalVal& val, int val_byte_size, bool* overflow) {
  switch (val_byte_size) {
    case 4: return ToDecimal4(Decimal4Value(val.val4), overflow);
    case 8: return ToDecimal4(Decimal8Value(val.val8), overflow);
    case 16: return ToDecimal4(Decimal16Value(val.val16), overflow);
  }
}
```

#### （2）运算逻辑封装在`*Value`中
以`Add_DecimalVal_DecimalVal`为例，核心是调用`DecimalValue::Add`方法处理刻度对齐和溢出：
```cpp
// decimal-operator-ir.cc中DECIMAL_ARITHMETIC_OP宏展开后
DecimalVal DecimalOperators::Add_DecimalVal_DecimalVal(
    FunctionContext* ctx, const DecimalVal& x, const DecimalVal& y) {
  if (x.is_null || y.is_null) return DecimalVal::null();
  bool overflow = false;
  // 1. 转*Value：DecimalVal → Decimal4/8/16Value
  Decimal4Value x_val = GetDecimal4Value(x, x_size, &overflow);
  Decimal4Value y_val = GetDecimal4Value(y, y_size, &overflow);
  // 2. 调用*Value的Add方法（处理刻度对齐）
  Decimal4Value result = x_val.Add(x_scale, y_val, y_scale, 
      return_precision, return_scale, round, &overflow);
  // 3. 转回*Val：Decimal4Value → DecimalVal
  return DecimalVal(result.value());
}
```

#### （3）精度/刻度调整：依赖`ScaleDecimalValue`
`DecimalVal`的刻度转换（如`RoundDecimal`）完全依赖`DecimalValue::ScaleTo`方法，这是`*Val`无法直接完成的：
```cpp
// 示例：ScaleDecimalValue封装*Value的ScaleTo逻辑
IR_ALWAYS_INLINE DecimalVal DecimalOperators::ScaleDecimalValue(FunctionContext* ctx,
    const Decimal4Value& val, int val_scale, int output_precision, int output_scale) {
  Decimal4Value scaled_val = val.ScaleTo(
      val_scale, output_scale, output_precision, /*round=*/false, &overflow);
  return DecimalVal(scaled_val.value());
}
```

### 2. StringVal：依赖`StringValue`处理字符语义
`StringVal`的`val`是`ptr+len`（指针+长度），无法直接比较，必须转为`StringValue`处理字符填充、编码等语义：
```cpp
// operators-ir.cc中Char类型比较逻辑
#define BINARY_PREDICATE_CHAR_NONNULL(OP, V1, V2) \
  StringValue iv1 = StringValue::FromStringVal(V1);\
  StringValue iv2 = StringValue::FromStringVal(V2);\
  // 处理Char类型的末尾填充空格（SQL语义）\
  iv1.SetLen(StringValue::UnpaddedCharLength(iv1.Ptr(), c->GetArgType(0)->len)); \
  iv2.SetLen(StringValue::UnpaddedCharLength(iv2.Ptr(), c->GetArgType(1)->len)); \
  return BooleanVal(iv1 OP iv2)
```

### 3. TimestampVal：依赖`TimestampValue`处理时间语义
`TimestampVal`的`val`是压缩的`int64_t`，需转为`TimestampValue`解包为「年/月/日/时/分/秒/微秒」，处理时区、精度等规则：
```cpp
// decimal-operator-ir.cc中Decimal转Timestamp逻辑
TimestampValue::FromUnixTimeNanos(
    seconds, nanoseconds, local_tz).ToTimestampVal(&result);
// 先通过TimestampValue处理时区转换，再转回TimestampVal
```

## 三、核心设计：`*Val`是接口，`*Value`是运算核心
| 层级       | 类型       | 作用                                                                 |
|------------|------------|----------------------------------------------------------------------|
| 对外接口层 | `*Val`     | 统一承载NULL语义，作为UDF函数的输入输出，屏蔽内部运算细节             |
| 内部运算层 | `*Value`   | 封装复杂类型的运算逻辑（精度/字符/时间语义），是真正的运算执行载体     |

所有运算符函数对外暴露的都是`*Val`类型，保证了接口的统一性；而`*Value`作为内部实现，封装了复杂类型的运算规则，避免将刻度、字符填充等细节暴露到接口层。

### 总结
1. **基础类型无需转`*Value`**：`*Val`的`val`是原生数值类型，运算无额外SQL语义，直接操作`val`即可完成；
2. **复杂类型必须转`*Value`**：`*Val`仅存储间接值（指针/压缩值/union），运算需处理精度、字符填充、时间规则等语义，这些逻辑全部封装在`*Value`中；
3. **接口与实现分离**：`*Val`作为对外统一接口承载NULL语义，`*Value`作为内部运算核心封装业务逻辑，是Impala兼顾接口统一性和运算灵活性的关键设计。

******************************************************************************************************