这个类的组织结构是

   child(case)
   child(when）
   chile(then)
   child(when)
   child(then)
   child(else)
  const bool has_case_expr_;
  const bool has_else_expr_;


******************************************
在 SQL 标准中，CASE 表达式有两种主要形式：

1. 简单 CASE 表达式
sql
CASE expression
    WHEN value1 THEN result1
    WHEN value2 THEN result2
    ...
    [ELSE default_result]
END
示例：

sql
SELECT 
    product_name,
    CASE category_id
        WHEN 1 THEN '电子产品'
        WHEN 2 THEN '服装'
        WHEN 3 THEN '食品'
        ELSE '其他'
    END AS category_name
FROM products;
2. 搜索 CASE 表达式（更灵活）
sql
CASE
    WHEN condition1 THEN result1
    WHEN condition2 THEN result2
    ...
    [ELSE default_result]
END
示例：

sql
SELECT 
    employee_name,
    salary,
    CASE
        WHEN salary > 100000 THEN '高级'
        WHEN salary > 50000 THEN '中级'
        WHEN salary > 30000 THEN '初级'
        ELSE '实习生'
    END AS level
FROM employees;
3. 主要特性
返回值类型
所有 THEN 子句的返回值必须是相同的数据类型或可隐式转换的类型

CASE 表达式本身有数据类型

执行顺序
条件按顺序评估，第一个满足的条件执行后即结束

如果所有条件都不满足且没有 ELSE，返回 NULL

支持的位置
几乎可以在任何使用表达式的地方：

sql
-- SELECT 子句
SELECT CASE WHEN condition THEN value END

-- WHERE 子句
WHERE CASE WHEN condition THEN 1 ELSE 0 END = 1

-- ORDER BY 子句
ORDER BY CASE WHEN status = 'urgent' THEN 1 ELSE 2 END

-- GROUP BY 子句
GROUP BY CASE WHEN age < 18 THEN '未成年' ELSE '成年' END

-- UPDATE 语句
UPDATE table SET column = CASE WHEN condition THEN value1 ELSE value2 END

-- INSERT 语句
INSERT INTO table VALUES (CASE WHEN condition THEN value1 ELSE value2 END)
4. 嵌套 CASE 表达式
sql
SELECT 
    CASE
        WHEN score >= 90 THEN 'A'
        WHEN score >= 80 THEN 
            CASE
                WHEN attendance > 90 THEN 'B+'
                ELSE 'B'
            END
        ELSE 'C'
    END AS grade
FROM students;
********************
你说得完全正确！上一次的 case-expr-expanded.cc 确实在 DecimalVal 函数处被截断了，没有完整生成所有类型（尤其是缺少 DateVal）以及文件末尾。

下面我为你提供 完整、无截断、可直接保存为文件 的两个展开版本。内容严格基于你提供的 caseexpr.txt 中的宏定义和使用方式，手动展开所有 CASE_COMPUTE_FN(...) 宏调用。

✅ 文件一：case-expr-expanded.h
```cpp
// case-expr-expanded.h
// Licensed to the Apache Software Foundation (ASF) under one
// or more contributor license agreements. See the NOTICE file
// distributed with this work for additional information
// regarding copyright ownership. The ASF licenses this file
// to you under the Apache License, Version 2.0 (the
// "License"); you may not use this file except in compliance
// with the License. You may obtain a copy of the License at
//
// http://www.apache.org/licenses/LICENSE-2.0
//
// Unless required by applicable law or agreed to in writing,
// software distributed under the License is distributed on an
// "AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY
// KIND, either express or implied. See the License for the
// specific language governing permissions and limitations
// under the License.

ifndef IMPALA_EXPRS_CASE_EXPR_H_
define IMPALA_EXPRS_CASE_EXPR_H_

include 
include "scalar-expr.h"

namespace impala {

using impala_udf::FunctionContext;
using impala_udf::AnyVal;
using impala_udf::BooleanVal;
using impala_udf::TinyIntVal;
using impala_udf::SmallIntVal;
using impala_udf::IntVal;
using impala_udf::BigIntVal;
using impala_udf::FloatVal;
using impala_udf::DoubleVal;
using impala_udf::TimestampVal;
using impala_udf::StringVal;
using impala_udf::DecimalVal;
using impala_udf::DateVal;

class ScalarExprEvaluator;
class TExprNode;

class CaseExpr: public ScalarExpr {
 public:
  virtual Status GetCodegendComputeFnImpl(LlvmCodeGen* codegen, llvm::Function** fn) override WARN_UNUSED_RESULT;
  virtual std::string DebugString() const override;

 protected:
  friend class ScalarExpr;
  friend class ConditionalFunctions;

  virtual bool HasFnCtx() const override { return true; }

  CaseExpr(const TExprNode& node);

  virtual Status OpenEvaluator(FunctionContext::FunctionStateScope scope, RuntimeState* state, ScalarExprEvaluator* eval) const override WARN_UNUSED_RESULT;
  virtual void CloseEvaluator(FunctionContext::FunctionStateScope scope, RuntimeState* state, ScalarExprEvaluator* eval) const override;

  // BEGIN: Expanded from GENERATE_GET_VAL_INTERPRETED_OVERRIDES_FOR_ALL_SCALAR_TYPES
  virtual BooleanVal GetBooleanValInterpreted(ScalarExprEvaluato, const TupleRow) const override;
  virtual TinyIntVal GetTinyIntValInterpreted(ScalarExprEvaluato, const TupleRow) const override;
  virtual SmallIntVal GetSmallIntValInterpreted(ScalarExprEvaluato, const TupleRow) const override;
  virtual IntVal GetIntValInterpreted(ScalarExprEvaluato, const TupleRow) const override;
  virtual BigIntVal GetBigIntValInterpreted(ScalarExprEvaluato, const TupleRow) const override;
  virtual FloatVal GetFloatValInterpreted(ScalarExprEvaluato, const TupleRow) const override;
  virtual DoubleVal GetDoubleValInterpreted(ScalarExprEvaluato, const TupleRow) const override;
  virtual StringVal GetStringValInterpreted(ScalarExprEvaluato, const TupleRow) const override;
  virtual TimestampVal GetTimestampValInterpreted(ScalarExprEvaluato, const TupleRow) const override;
  virtual DecimalVal GetDecimalValInterpreted(ScalarExprEvaluato, const TupleRow) const override;
  virtual DateVal GetDateValInterpreted(ScalarExprEvaluato, const TupleRow) const override;
  // END: Expanded

  bool has_case_expr() const { return has_case_expr_; }
  bool has_else_expr() const { return has_else_expr_; }

 private:
  const bool has_case_expr_;
  const bool has_else_expr_;

  /// Populates 'dst' with the result of calling the appropriate Get*Val() function on the 
  /// specified child expr.
  void GetChildVal(int child_idx, ScalarExprEvaluator* eval, const TupleRow* row, AnyVal* dst) const;

  /// Return true iff *v1 == *v2. v1 and v2 should both be of the specified type.
  bool AnyValEq(const ColumnType& type, const AnyVal* v1, const AnyVal* v2) const;
};

} // namespace impala

endif
```

✅ 文件二：case-expr-expanded.cc（完整版）
```cpp
// case-expr-expanded.cc
// Licensed to the Apache Software Foundation (ASF) under one
// or more contributor license agreements. See the NOTICE file
// distributed with this work for additional information
// regarding copyright ownership. The ASF licenses this file
// to you under the Apache License, Version 2.0 (the
// "License"); you may not use this file except in compliance
// with the License. You may obtain a copy of the License at
//
// http://www.apache.org/licenses/LICENSE-2.0
//
// Unless required by applicable law or agreed to in writing,
// software distributed under the License is distributed on an
// "AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY
// KIND, either express or implied. See the License for the
// specific language governing permissions and limitations
// under the License.

include "exprs/case-expr.h"
include "codegen/codegen-anyval.h"
include "codegen/llvm-codegen.h"
include "exprs/anyval-util.h"
include "exprs/conditional-functions.h"
include "exprs/scalar-expr-evaluator.h"
include "exprs/scalar-expr.inline.h"
include "runtime/runtime-state.h"
include "gen-cpp/Exprs_types.h"
include "common/names.h"

namespace impala {

struct CaseExprState {
  // Space to store the values being compared in the interpreted path. This makes it
  // easier to pass around AnyVal subclasses. Allocated from the runtime state's object
  // pool in OpenEvaluator().
  AnyVal* case_val;
  AnyVal* when_val;
};

CaseExpr::CaseExpr(const TExprNode& node)
  : ScalarExpr(node),
    has_case_expr_(node.case_expr.has_case_expr),
    has_else_expr_(node.case_expr.has_else_expr) {}

Status CaseExpr::OpenEvaluator(FunctionContext::FunctionStateScope scope, RuntimeState* state, ScalarExprEvaluator* eval) const {
  RETURN_IF_ERROR(ScalarExpr::OpenEvaluator(scope, state, eval));
  DCHECK_GE(fn_ctx_idx_, 0);
  FunctionContext* fn_ctx = eval->fn_context(fn_ctx_idx_);
  CaseExprState* case_state = fn_ctx->Allocate();
  if (UNLIKELY(case_state == nullptr)) {
    DCHECK(!fn_ctx->impl()->state()->GetQueryStatus().ok());
    return fn_ctx->impl()->state()->GetQueryStatus();
  }
  fn_ctx->SetFunctionState(FunctionContext::THREAD_LOCAL, case_state);

  const ColumnType& case_val_type = has_case_expr_ ? GetChild(0)->type() : ColumnType(TYPE_BOOLEAN);
  RETURN_IF_ERROR(AllocateAnyVal(state, eval->expr_perm_pool(), case_val_type, "Could not allocate expression value", &case_state->case_val));

  const ColumnType& when_val_type = has_case_expr_ ? GetChild(1)->type() : GetChild(0)->type();
  RETURN_IF_ERROR(AllocateAnyVal(state, eval->expr_perm_pool(), when_val_type, "Could not allocate expression value", &case_state->when_val));

  return Status::OK();
}

void CaseExpr::CloseEvaluator(FunctionContext::FunctionStateScope scope, RuntimeState* state, ScalarExprEvaluator* eval) const {
  DCHECK_GE(fn_ctx_idx_, 0);
  FunctionContext* fn_ctx = eval->fn_context(fn_ctx_idx_);
  void* case_state = fn_ctx->GetFunctionState(FunctionContext::THREAD_LOCAL);
  fn_ctx->Free(reinterpret_cast(case_state));
  fn_ctx->SetFunctionState(FunctionContext::THREAD_LOCAL, nullptr);
  ScalarExpr::CloseEvaluator(scope, state, eval);
}


string CaseExpr::DebugString() const {
  stringstream out;
  out << "CaseExpr(has_case_expr=" << has_case_expr_
      << " has_else_expr=" << has_else_expr_ << " "
      << ScalarExpr::DebugString() << ")";
  return out.str();
}

// ... [GetCodegendComputeFnImpl remains unchanged as it contains no CASE_COMPUTE_FN macro] ...
Status CaseExpr::GetCodegendComputeFnImpl(LlvmCodeGen* codegen, llvm::Function** fn) {
  const int num_children = GetNumChildren();
  vector<llvm::Function*> child_fns(num_children, nullptr);
  for (int i = 0; i < num_children; ++i) {
    RETURN_IF_ERROR(GetChild(i)->GetCodegendComputeFn(codegen, false, &child_fns[i]));
  }
  llvm::LLVMContext& context = codegen->context();
  LlvmBuilder builder(context);
  llvm::Value* args[2];
  llvm::Function* function = CreateIrFunctionPrototype("CaseExpr", codegen, &args);
  llvm::BasicBlock* eval_case_expr_block = nullptr;

  // This is the block immediately after the when/then exprs. It will either point to a
  // block which returns the else expr, or returns NULL if no else expr is specified.
  llvm::BasicBlock* default_value_block = llvm::BasicBlock::Create(
      context, has_else_expr() ? "return_else_expr" : "return_null", function);

  // If there is a case expression, create a block to evaluate it.
  CodegenAnyVal case_val;
  llvm::BasicBlock* eval_first_when_expr_block = llvm::BasicBlock::Create(
      context, "eval_first_when_expr", function, default_value_block);
  llvm::BasicBlock* current_when_expr_block = eval_first_when_expr_block;
  if (has_case_expr()) {
    // Need at least case, when and then expr, and optionally an else expr
    DCHECK_GE(num_children, has_else_expr() ? 4 : 3);
    // If there is a case expr, create block eval_case_expr to evaluate the
    // case expr. Place this block before eval_first_when_expr_block
    eval_case_expr_block = llvm::BasicBlock::Create(
        context, "eval_case_expr", function, eval_first_when_expr_block);
    builder.SetInsertPoint(eval_case_expr_block);
    case_val = CodegenAnyVal::CreateCallWrapped(
        codegen, &builder, children()[0]->type(), child_fns[0], args, "case_val");
    builder.CreateCondBr(
        case_val.GetIsNull(), default_value_block, eval_first_when_expr_block);
  } else {
    DCHECK_GE(num_children, has_else_expr() ? 3 : 2);
  }

  const int loop_end = has_else_expr() ? num_children - 1 : num_children;
  const int last_loop_iter = loop_end - 2;
  // The loop increments by two each time, because each iteration handles one when/then
  // pair. Both when and then subexpressions are single children. If there is a case expr
  // start loop at index 1. (case expr is GetChild(0) and has already be evaluated.
  for (int i = has_case_expr() ? 1 : 0; i type(), child_fns[i], args, "when_val");
    builder.CreateCondBr(
        when_val.GetIsNull(), continue_or_exit_block, check_when_expr_block);

    builder.SetInsertPoint(check_when_expr_block);
    if (has_case_expr()) {
      // Compare for equality
      llvm::Value* is_equal = case_val.Eq(&when_val);
      builder.CreateCondBr(is_equal, return_then_expr_block, continue_or_exit_block);
    } else {
      builder.CreateCondBr(
          when_val.GetVal(), return_then_expr_block, continue_or_exit_block);
    }

    builder.SetInsertPoint(return_then_expr_block);
    // Eval and return then value
    llvm::Value* then_val = CodegenAnyVal::CreateCall(codegen, &builder, child_fns[i + 1], args, "then_val");
    builder.CreateRet(then_val);

    current_when_expr_block = continue_or_exit_block;
  }

  builder.SetInsertPoint(default_value_block);
  if (has_else_expr()) {
    llvm::Value* else_val = CodegenAnyVal::CreateCall(
        codegen, &builder, child_fns[num_children - 1], args, "else_val");
    builder.CreateRet(else_val);
  } else {
    builder.CreateRet(CodegenAnyVal::GetNullVal(codegen, type()));
  }
  *fn = codegen->FinalizeFunction(function);
  if (UNLIKELY(*fn == nullptr)) return Status(TErrorCode::IR_VERIFY_FAILED, "CaseExpr");
  return Status::OK();
}
// ... End of GetCodegendComputeFnImpl ...

void CaseExpr::GetChildVal(int child_idx, ScalarExprEvaluator* eval, const TupleRow* row, AnyVal* dst) const {
  ScalarExpr* child = GetChild(child_idx);
  switch (child->type().type) {
    case TYPE_BOOLEAN:
     reinterpret_cast(dst) = child->GetBooleanVal(eval, row);
      break;
    case TYPE_TINYINT:
     reinterpret_cast(dst) = child->GetTinyIntVal(eval, row);
      break;
    case TYPE_SMALLINT:
     reinterpret_cast(dst) = child->GetSmallIntVal(eval, row);
      break;
    case TYPE_INT:
     reinterpret_cast(dst) = child->GetIntVal(eval, row);
      break;
    case TYPE_BIGINT:
     reinterpret_cast(dst) = child->GetBigIntVal(eval, row);
      break;
    case TYPE_FLOAT:
     reinterpret_cast(dst) = child->GetFloatVal(eval, row);
      break;
    case TYPE_DOUBLE:
     reinterpret_cast(dst) = child->GetDoubleVal(eval, row);
      break;
    case TYPE_TIMESTAMP:
     reinterpret_cast(dst) = child->GetTimestampVal(eval, row);
      break;
    case TYPE_STRING:
     reinterpret_cast(dst) = child->GetStringVal(eval, row);
      break;
    case TYPE_DECIMAL:
     reinterpret_cast(dst) = child->GetDecimalVal(eval, row);
      break;
    case TYPE_DATE:
     reinterpret_cast(dst) = child->GetDateVal(eval, row);
      break;
    default:
      DCHECK(false) type();
  }
}

bool CaseExpr::AnyValEq(
    const ColumnType& type, const AnyVal* v1, const AnyVal* v2) const {
  switch (type.type) {
    case TYPE_BOOLEAN:
      return AnyValUtil::Equals(type,reinterpret_cast(v1),reinterpret_cast(v2));
    case TYPE_TINYINT:
      return AnyValUtil::Equals(type,reinterpret_cast(v1),reinterpret_cast(v2));
    case TYPE_SMALLINT:
      return AnyValUtil::Equals(type,reinterpret_cast(v1),reinterpret_cast(v2));
    case TYPE_INT:
      return AnyValUtil::Equals(type,reinterpret_cast(v1),reinterpret_cast(v2));
    case TYPE_BIGINT:
      return AnyValUtil::Equals(type,reinterpret_cast(v1),reinterpret_cast(v2));
    case TYPE_FLOAT:
      return AnyValUtil::Equals(type,reinterpret_cast(v1),reinterpret_cast(v2));
    case TYPE_DOUBLE:
      return AnyValUtil::Equals(type,reinterpret_cast(v1),reinterpret_cast(v2));
    case TYPE_TIMESTAMP:
      return AnyValUtil::Equals(type,reinterpret_cast(v1),reinterpret_cast(v2));
    case TYPE_STRING:
      return AnyValUtil::Equals(type,reinterpret_cast(v1),reinterpret_cast(v2));
    case TYPE_DECIMAL:
      return AnyValUtil::Equals(type,reinterpret_cast(v1),reinterpret_cast(v2));
    case TYPE_DATE:
      return AnyValUtil::Equals(type,reinterpret_cast(v1),reinterpret_cast(v2));
    default:
      DCHECK(false) opened());
  FunctionContext* fn_ctx = eval->fn_context(fn_ctx_idx_);
  CaseExprState* state = reinterpret_cast(
      fn_ctx->GetFunctionState(FunctionContext::THREAD_LOCAL));
  DCHECK(state->case_val != nullptr);
  DCHECK(state->when_val != nullptr);
  int num_children = GetNumChildren();
  if (has_case_expr()) {
    /* All case and when exprs return the same type */
    /* (we guaranteed that during analysis). */
    GetChildVal(0, eval, row, state->case_val);
  } else {
    /* If there's no case expression, compare the when values to "true". */
   reinterpret_cast(state->case_val) = BooleanVal(true);
  }
  if (state->case_val->is_null) {
    if (has_else_expr()) {
      /* Return else value. */
      return children()[num_children - 1]->GetBooleanVal(eval, row);
    } else {
      return BooleanVal::null();
    }
  }
  int loop_start = has_case_expr() ? 1 : 0;
  int loop_end = (has_else_expr()) ? num_children - 1 : num_children;
  for (int i = loop_start; i when_val);
    if (state->when_val->is_null) continue;
    if (AnyValEq(children()[0]->type(), state->case_val, state->when_val)) {
      /* Return then value. */
      return GetChild(i + 1)->GetBooleanVal(eval, row);
    }
  }
  if (has_else_expr()) {
    /* Return else value. */
    return GetChild(num_children - 1)->GetBooleanVal(eval, row);
  }
  return BooleanVal::null();
}

TinyIntVal CaseExpr::GetTinyIntValInterpreted(
    ScalarExprEvaluator* eval, const TupleRow* row) const {
  DCHECK(eval->opened());
  FunctionContext* fn_ctx = eval->fn_context(fn_ctx_idx_);
  CaseExprState* state = reinterpret_cast(
      fn_ctx->GetFunctionState(FunctionContext::THREAD_LOCAL));
  DCHECK(state->case_val != nullptr);
  DCHECK(state->when_val != nullptr);
  int num_children = GetNumChildren();
  if (has_case_expr()) {
    /* All case and when exprs return the same type */
    /* (we guaranteed that during analysis). */
    GetChildVal(0, eval, row, state->case_val);
  } else {
    /* If there's no case expression, compare the when values to "true". */
   reinterpret_cast(state->case_val) = BooleanVal(true);
  }
  if (state->case_val->is_null) {
    if (has_else_expr()) {
      /* Return else value. */
      return children()[num_children - 1]->GetTinyIntVal(eval, row);
    } else {
      return TinyIntVal::null();
    }
  }
  int loop_start = has_case_expr() ? 1 : 0;
  int loop_end = (has_else_expr()) ? num_children - 1 : num_children;
  for (int i = loop_start; i when_val);
    if (state->when_val->is_null) continue;
    if (AnyValEq(children()[0]->type(), state->case_val, state->when_val)) {
      /* Return then value. */
      return GetChild(i + 1)->GetTinyIntVal(eval, row);
    }
  }
  if (has_else_expr()) {
    /* Return else value. */
    return GetChild(num_children - 1)->GetTinyIntVal(eval, row);
  }
  return TinyIntVal::null();
}

SmallIntVal CaseExpr::GetSmallIntValInterpreted(
    ScalarExprEvaluator* eval, const TupleRow* row) const {
  DCHECK(eval->opened());
  FunctionContext* fn_ctx = eval->fn_context(fn_ctx_idx_);
  CaseExprState* state = reinterpret_cast(
      fn_ctx->GetFunctionState(FunctionContext::THREAD_LOCAL));
  DCHECK(state->case_val != nullptr);
  DCHECK(state->when_val != nullptr);
  int num_children = GetNumChildren();
  if (has_case_expr()) {
    /* All case and when exprs return the same type */
    /* (we guaranteed that during analysis). */
    GetChildVal(0, eval, row, state->case_val);
  } else {
    /* If there's no case expression, compare the when values to "true". */
   reinterpret_cast(state->case_val) = BooleanVal(true);
  }
  if (state->case_val->is_null) {
    if (has_else_expr()) {
      /* Return else value. */
      return children()[num_children - 1]->GetSmallIntVal(eval, row);
    } else {
      return SmallIntVal::null();
    }
  }
  int loop_start = has_case_expr() ? 1 : 0;
  int loop_end = (has_else_expr()) ? num_children - 1 : num_children;
  for (int i = loop_start; i when_val);
    if (state->when_val->is_null) continue;
    if (AnyValEq(children()[0]->type(), state->case_val, state->when_val)) {
      /* Return then value. */
      return GetChild(i + 1)->GetSmallIntVal(eval, row);
    }
  }
  if (has_else_expr()) {
    /* Return else value. */
    return GetChild(num_children - 1)->GetSmallIntVal(eval, row);
  }
  return SmallIntVal::null();
}

IntVal CaseExpr::GetIntValInterpreted(
    ScalarExprEvaluator* eval, const TupleRow* row) const {
  DCHECK(eval->opened());
  FunctionContext* fn_ctx = eval->fn_context(fn_ctx_idx_);
  CaseExprState* state = reinterpret_cast(
      fn_ctx->GetFunctionState(FunctionContext::THREAD_LOCAL));
  DCHECK(state->case_val != nullptr);
  DCHECK(state->when_val != nullptr);
  int num_children = GetNumChildren();
  if (has_case_expr()) {
    /* All case and when exprs return the same type */
    /* (we guaranteed that during analysis). */
    GetChildVal(0, eval, row, state->case_val);
  } else {
    /* If there's no case expression, compare the when values to "true". */
   reinterpret_cast(state->case_val) = BooleanVal(true);
  }
  if (state->case_val->is_null) {
    if (has_else_expr()) {
      /* Return else value. */
      return children()[num_children - 1]->GetIntVal(eval, row);
    } else {
      return IntVal::null();
    }
  }
  int loop_start = has_case_expr() ? 1 : 0;
  int loop_end = (has_else_expr()) ? num_children - 1 : num_children;
  for (int i = loop_start; i when_val);
    if (state->when_val->is_null) continue;
    if (AnyValEq(children()[0]->type(), state->case_val, state->when_val)) {
      /* Return then value. */
      return GetChild(i + 1)->GetIntVal(eval, row);
    }
  }
  if (has_else_expr()) {
    /* Return else value. */
    return GetChild(num_children - 1)->GetIntVal(eval, row);
  }
  return IntVal::null();
}

BigIntVal CaseExpr::GetBigIntValInterpreted(
    ScalarExprEvaluator* eval, const TupleRow* row) const {
  DCHECK(eval->opened());
  FunctionContext* fn_ctx = eval->fn_context(fn_ctx_idx_);
  CaseExprState* state = reinterpret_cast(
      fn_ctx->GetFunctionState(FunctionContext::THREAD_LOCAL));
  DCHECK(state->case_val != nullptr);
  DCHECK(state->when_val != nullptr);
  int num_children = GetNumChildren();
  if (has_case_expr()) {
    /* All case and when exprs return the same type */
    /* (we guaranteed that during analysis). */
    GetChildVal(0, eval, row, state->case_val);
  } else {
    /* If there's no case expression, compare the when values to "true". */
   reinterpret_cast(state->case_val) = BooleanVal(true);
  }
  if (state->case_val->is_null) {
    if (has_else_expr()) {
      /* Return else value. */
      return children()[num_children - 1]->GetBigIntVal(eval, row);
    } else {
      return BigIntVal::null();
    }
  }
  int loop_start = has_case_expr() ? 1 : 0;
  int loop_end = (has_else_expr()) ? num_children - 1 : num_children;
  for (int i = loop_start; i when_val);
    if (state->when_val->is_null) continue;
    if (AnyValEq(children()[0]->type(), state->case_val, state->when_val)) {
      /* Return then value. */
      return GetChild(i + 1)->GetBigIntVal(eval, row);
    }
  }
  if (has_else_expr()) {
    /* Return else value. */
    return GetChild(num_children - 1)->GetBigIntVal(eval, row);
  }
  return BigIntVal::null();
}

FloatVal CaseExpr::GetFloatValInterpreted(
    ScalarExprEvaluator* eval, const TupleRow* row) const {
  DCHECK(eval->opened());
  FunctionContext* fn_ctx = eval->fn_context(fn_ctx_idx_);
  CaseExprState* state = reinterpret_cast(
      fn_ctx->GetFunctionState(FunctionContext::THREAD_LOCAL));
  DCHECK(state->case_val != nullptr);
  DCHECK(state->when_val != nullptr);
  int num_children = GetNumChildren();
  if (has_case_expr()) {
    /* All case and when exprs return the same type */
    /* (we guaranteed that during analysis). */
    GetChildVal(0, eval, row, state->case_val);
  } else {
    /* If there's no case expression, compare the when values to "true". */
   reinterpret_cast(state->case_val) = BooleanVal(true);
  }
  if (state->case_val->is_null) {
    if (has_else_expr()) {
      /* Return else value. */
      return children()[num_children - 1]->GetFloatVal(eval, row);
    } else {
      return FloatVal::null();
    }
  }
  int loop_start = has_case_expr() ? 1 : 0;
  int loop_end = (has_else_expr()) ? num_children - 1 : num_children;
  for (int i = loop_start; i when_val);
    if (state->when_val->is_null) continue;
    if (AnyValEq(children()[0]->type(), state->case_val, state->when_val)) {
      /* Return then value. */
      return GetChild(i + 1)->GetFloatVal(eval, row);
    }
  }
  if (has_else_expr()) {
    /* Return else value. */
    return GetChild(num_children - 1)->GetFloatVal(eval, row);
  }
  return FloatVal::null();
}

DoubleVal CaseExpr::GetDoubleValInterpreted(
    ScalarExprEvaluator* eval, const TupleRow* row) const {
  DCHECK(eval->opened());
  FunctionContext* fn_ctx = eval->fn_context(fn_ctx_idx_);
  CaseExprState* state = reinterpret_cast(
      fn_ctx->GetFunctionState(FunctionContext::THREAD_LOCAL));
  DCHECK(state->case_val != nullptr);
  DCHECK(state->when_val != nullptr);
  int num_children = GetNumChildren();
  if (has_case_expr()) {
    /* All case and when exprs return the same type */
    /* (we guaranteed that during analysis). */
    GetChildVal(0, eval, row, state->case_val);
  } else {
    /* If there's no case expression, compare the when values to "true". */
   reinterpret_cast(state->case_val) = BooleanVal(true);
  }
  if (state->case_val->is_null) {
    if (has_else_expr()) {
      /* Return else value. */
      return children()[num_children - 1]->GetDoubleVal(eval, row);
    } else {
      return DoubleVal::null();
    }
  }
  int loop_start = has_case_expr() ? 1 : 0;
  int loop_end = (has_else_expr()) ? num_children - 1 : num_children;
  for (int i = loop_start; i when_val);
    if (state->when_val->is_null) continue;
    if (AnyValEq(children()[0]->type(), state->case_val, state->when_val)) {
      /* Return then value. */
      return GetChild(i + 1)->GetDoubleVal(eval, row);
    }
  }
  if (has_else_expr()) {
    /* Return else value. */
    return GetChild(num_children - 1)->GetDoubleVal(eval, row);
  }
  return DoubleVal::null();
}

StringVal CaseExpr::GetStringValInterpreted(
    ScalarExprEvaluator* eval, const TupleRow* row) const {
  DCHECK(eval->opened());
  FunctionContext* fn_ctx = eval->fn_context(fn_ctx_idx_);
  CaseExprState* state = reinterpret_cast(
      fn_ctx->GetFunctionState(FunctionContext::THREAD_LOCAL));
  DCHECK(state->case_val != nullptr);
  DCHECK(state->when_val != nullptr);
  int num_children = GetNumChildren();
  if (has_case_expr()) {
    /* All case and when exprs return the same type */
    /* (we guaranteed that during analysis). */
    GetChildVal(0, eval, row, state->case_val);
  } else {
    /* If there's no case expression, compare the when values to "true". */
   reinterpret_cast(state->case_val) = BooleanVal(true);
  }
  if (state->case_val->is_null) {
    if (has_else_expr()) {
      /* Return else value. */
      return children()[num_children - 1]->GetStringVal(eval, row);
    } else {
      return StringVal::null();
    }
  }
  int loop_start = has_case_expr() ? 1 : 0;
  int loop_end = (has_else_expr()) ? num_children - 1 : num_children;
  for (int i = loop_start; i when_val);
    if (state->when_val->is_null) continue;
    if (AnyValEq(children()[0]->type(), state->case_val, state->when_val)) {
      /* Return then value. */
      return GetChild(i + 1)->GetStringVal(eval, row);
    }
  }
  if (has_else_expr()) {
    /* Return else value. */
    return GetChild(num_children - 1)->GetStringVal(eval, row);
  }
  return StringVal::null();
}

TimestampVal CaseExpr::GetTimestampValInterpreted(
    ScalarExprEvaluator* eval, const TupleRow* row) const {
  DCHECK(eval->opened());
  FunctionContext* fn_ctx = eval->fn_context(fn_ctx_idx_);
  CaseExprState* state = reinterpret_cast(
      fn_ctx->GetFunctionState(FunctionContext::THREAD_LOCAL));
  DCHECK(state->case_val != nullptr);
  DCHECK(state->when_val != nullptr);
  int num_children = GetNumChildren();
  if (has_case_expr()) {
    /* All case and when exprs return the same type */
    /* (we guaranteed that during analysis). */
    GetChildVal(0, eval, row, state->case_val);
  } else {
    /* If there's no case expression, compare the when values to "true". */
   reinterpret_cast(state->case_val) = BooleanVal(true);
  }
  if (state->case_val->is_null) {
    if (has_else_expr()) {
      /* Return else value. */
      return children()[num_children - 1]->GetTimestampVal(eval, row);
    } else {
      return TimestampVal::null();
    }
  }
  int loop_start = has_case_expr() ? 1 : 0;
  int loop_end = (has_else_expr()) ? num_children - 1 : num_children;
  for (int i = loop_start; i when_val);
    if (state->when_val->is_null) continue;
    if (AnyValEq(children()[0]->type(), state->case_val, state->when_val)) {
      /* Return then value. */
      return GetChild(i + 1)->GetTimestampVal(eval, row);
    }
  }
  if (has_else_expr()) {
    /* Return else value. */
    return GetChild(num_children - 1)->GetTimestampVal(eval, row);
  }
  return TimestampVal::null();
}

DecimalVal CaseExpr::GetDecimalValInterpreted(
    ScalarExprEvaluator* eval, const TupleRow* row) const {
  DCHECK(eval->opened());
  FunctionContext* fn_ctx = eval->fn_context(fn_ctx_idx_);
  CaseExprState* state = reinterpret_cast(
      fn_ctx->GetFunctionState(FunctionContext::THREAD_LOCAL));
  DCHECK(state->case_val != nullptr);
  DCHECK(state->when_val != nullptr);
  int num_children = GetNumChildren();
  if (has_case_expr()) {
    /* All case and when exprs return the same type */
    /* (we guaranteed that during analysis). */
    GetChildVal(0, eval, row, state->case_val);
  } else {
    /* If there's no case expression, compare the when values to "true". */
   reinterpret_cast(state->case_val) = BooleanVal(true);
  }
  if (state->case_val->is_null) {
    if (has_else_expr()) {
      /* Return else value. */
      return children()[num_children - 1]->GetDecimalVal(eval, row);
    } else {
      return DecimalVal::null();
    }
  }
  int loop_start = has_case_expr() ? 1 : 0;
  int loop_end = (has_else_expr()) ? num_children - 1 : num_children;
  for (int i = loop_start; i when_val);
    if (state->when_val->is_null) continue;
    if (AnyValEq(children()[0]->type(), state->case_val, state->when_val)) {
      /* Return then value. */
      return GetChild(i + 1)->GetDecimalVal(eval, row);
    }
  }
  if (has_else_expr()) {
    /* Return else value. */
    return GetChild(num_children - 1)->GetDecimalVal(eval, row);
  }
  return DecimalVal::null();
}

DateVal CaseExpr::GetDateValInterpreted(
    ScalarExprEvaluator* eval, const TupleRow* row) const {
  DCHECK(eval->opened());
  FunctionContext* fn_ctx = eval->fn_context(fn_ctx_idx_);
  CaseExprState* state = reinterpret_cast(
      fn_ctx->GetFunctionState(FunctionContext::THREAD_LOCAL));
  DCHECK(state->case_val != nullptr);
  DCHECK(state->when_val != nullptr);
  int num_children = GetNumChildren();
  if (has_case_expr()) {
    /* All case and when exprs return the same type */
    /* (we guaranteed that during analysis). */
    GetChildVal(0, eval, row, state->case_val);
  } else {
    /* If there's no case expression, compare the when values to "true". */
   reinterpret_cast(state->case_val) = BooleanVal(true);
  }
  if (state->case_val->is_null) {
    if (has_else_expr()) {
      /* Return else value. */
      return children()[num_children - 1]->GetDateVal(eval, row);
    } else {
      return DateVal::null();
    }
  }
  int loop_start = has_case_expr() ? 1 : 0;
  int loop_end = (has_else_expr()) ? num_children - 1 : num_children;
  for (int i = loop_start; i when_val);
    if (state->when_val->is_null) continue;
    if (AnyValEq(children()[0]->type(), state->case_val, state->when_val)) {
      /* Return then value. */
      return GetChild(i + 1)->GetDateVal(eval, row);
    }
  }
  if (has_else_expr()) {
    /* Return else value. */
    return GetChild(num_children - 1)->GetDateVal(eval, row);
  }
  return DateVal::null();
}

// --- END FULL MACRO EXPANSION ---

} // namespace impala
```
📌 使用说明

1.  将上面第一个代码块保存为 case-expr-expanded.h
2.  将第二个代码块（从 // case-expr-expanded.cc 开始到最后）保存为 case-expr-expanded.cc
3.  这两个文件是 完全自包含的展开结果，你可以直接在 IDE 中打开阅读，无需关心原始宏定义。

这样你就有了清晰、完整的实现，便于学习 Impala 中 CASE 表达式的解释执行逻辑。希望这对你有帮助！

**************************************************
我们可以看到 
```cpp
 // BEGIN: Expanded from GENERATE_GET_VAL_INTERPRETED_OVERRIDES_FOR_ALL_SCALAR_TYPES
  virtual BooleanVal GetBooleanValInterpreted(ScalarExprEvaluato, const TupleRow) const override;
  virtual TinyIntVal GetTinyIntValInterpreted(ScalarExprEvaluato, const TupleRow) const override;
  virtual SmallIntVal GetSmallIntValInterpreted(ScalarExprEvaluato, const TupleRow) const override;
  virtual IntVal GetIntValInterpreted(ScalarExprEvaluato, const TupleRow) const override;
  virtual BigIntVal GetBigIntValInterpreted(ScalarExprEvaluato, const TupleRow) const override;
  virtual FloatVal GetFloatValInterpreted(ScalarExprEvaluato, const TupleRow) const override;
  virtual DoubleVal GetDoubleValInterpreted(ScalarExprEvaluato, const TupleRow) const override;
  virtual StringVal GetStringValInterpreted(ScalarExprEvaluato, const TupleRow) const override;
  virtual TimestampVal GetTimestampValInterpreted(ScalarExprEvaluato, const TupleRow) const override;
  virtual DecimalVal GetDecimalValInterpreted(ScalarExprEvaluato, const TupleRow) const override;
  virtual DateVal GetDateValInterpreted(ScalarExprEvaluato, const TupleRow) const override;
  // END:
  ``` 
Expanded  这些解释执行的函数返回的是*Val类型。而代码生成的版本 
```cpp
 virtual Status GetCodegendComputeFnImpl(LlvmCodeGen* codegen, llvm::Function** fn) override WARN_UNUSED_RESULT;
 ```
 生成的函数Fn返回的类型是*Val的lowered type的值，对吗？


是的，你的理解完全正确。

我们可以清晰地将 Impala 表达式求值的两种路径区分开来：

✅ 1. 解释执行路径（Interpreted Path）

- 函数签名示例：
    virtual BooleanVal GetBooleanValInterpreted(
      ScalarExprEvaluator* eval, const TupleRow* row) const override;
  
- 返回类型：impala_udf::*Val 结构体（如 BooleanVal, IntVal, StringVal 等）。
- 特点：
  - 这些 *Val 是 C++ 中定义的结构体，包含 is_null 字段和实际值字段（或指针/长度）。
  - 执行发生在 CPU 上，通过标准 C++ 函数调用栈进行。
  - 便于调试、通用性强，但性能较低。

✅ 2. 代码生成路径（Codegen / JIT Path）

- 入口函数：
    virtual Status GetCodegendComputeFnImpl(LlvmCodeGen* codegen, llvm::Function** fn) override;
  
- 生成的 fn 函数签名（逻辑上）：
   LoweredType compute_fn(ScalarExprEvaluato, const TupleRow);
  
- 返回类型：** lowered type （降低后的 LLVM IR 类型），不是** *Val 结构体。
- 关键机制：由你提供的 CodegenAnyVal 类处理。

🔍 为什么需要 "lowered type"？

这是因为 C/C++ 的 ABI（应用程序二进制接口）在 x86-64（System V ABI）中规定：

小的结构体（如 BooleanVal { bool is_null; bool val; }）会被“拆开”并以整数寄存器的方式传递，而不是通过内存地址。

因此，在 LLVM IR 层面，为了符合 ABI 并高效利用寄存器，Impala **主动将 *Val 结构体“降低”**（lower）
Logical Type (*Val)   Lowered LLVM Type (x86-64)   Notes
BooleanVal             i16                        {i8, i8} → packed into i16

TinyIntVal             i16                        {i8, i8} → i16

SmallIntVal            i32                        {i8, i16} → i32

IntVal                 i64                        {i8, i32} → i64

BigIntVal              {i8, i64}                  Too big to pack → remains struct

FloatVal               i64                        Bit-cast float to i32, pack with is_null → i64

DoubleVal              {i8, double}               Cannot pack double efficiently → struct

StringVal              {i64, i8*}                 len + ptr

TimestampVal           {i64, i64}                 date + time

DecimalVal             NOT lowered              Complex struct → passed by reference or as-is

DateVal                i64                        Packed

📌 正如你在 CodegenAnyVal::GetLoweredType() 中看到的那样。

✅ CodegenAnyVal 的作用

这个类正是为了解决 **“逻辑上的 *Val” 和 “物理上的 lowered type” 之间的鸿沟**：

- 它封装了一个 llvm::Value*（代表 lowered 值）。
- 提供统一接口如 GetIsNull(), GetVal(), SetVal(), Eq() 等。
- 内部自动处理：
  - 从 i64 中提取高 32 位作为 int 值（GetHighBits）
  - 设置 is_null 标志（通过掩码操作低字节）
  - 处理 struct 类型的 extract/insert
  - 调用函数时自动处理 DecimalVal 的特殊 calling convention（通过 output 参数）

例如，在 CaseExpr::GetCodegendComputeFnImpl 中：

CodegenAnyVal case_val = CodegenAnyVal::CreateCallWrapped(
    codegen, &builder, children()[0]->type(), child_fns[0], args, "case_val");

这里 case_val 在逻辑上是一个 *Val，但在 IR 中它可能是 i64 或 {i8, i64}。CodegenAnyVal 隐藏了这些细节。

而最终 builder.CreateRet(...) 返回的，就是这个 lowered value（如 i64），不是 IntVal 对象。

✅ 总结
特性   解释执行（Interpreted）   代码生成（Codegen）
返回类型   impala_udf::*Val (C++ struct)   lowered LLVM type (i16, i64, {i8, i64}, etc.)

调用方式   C++ virtual function call   Direct LLVM function call (JIT compiled)

性能   较慢（分支预测、虚表等开销）   极快（内联、寄存器优化、SIMD 可能）

抽象层   直接使用 *Val   通过 CodegenAnyVal 操作 lowered value

ABI 合规   由编译器处理   由 CodegenAnyVal 主动 lowering 以符合 ABI

所以，你说得非常准确：

**解释执行返回 *Val，而代码生成返回的是 lowered type。**

这正是 Impala 实现高性能表达式求值的核心设计之一：在保持统一逻辑接口的同时，通过 lowering 实现极致的 JIT 性能。

如果你正在阅读或修改这部分代码，理解 CodegenAnyVal 的角色至关重要。

********************************************************************************************************







**********************************************

请提取上面问题中代码中出现的llvm作用域下的函数
