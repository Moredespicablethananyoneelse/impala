*******************************************************88
int LlvmCodeGen::InlineConstFnAttrs(const FunctionContext::TypeDesc& ret_type,
    const vector<FunctionContext::TypeDesc>& arg_types, llvm::Function* fn) {
  int replaced = 0;
  for (llvm::inst_iterator iter = inst_begin(fn), end = inst_end(fn); iter != end;) {
    // Increment iter now so we don't mess it up modifying the instruction below
    llvm::Instruction* instr = &*(iter++);

    // Look for call instructions
    if (!llvm::isa<llvm::CallInst>(instr)) continue;
    llvm::CallInst* call_instr = llvm::cast<llvm::CallInst>(instr);
    llvm::Function* called_fn = call_instr->getCalledFunction();

    // Look for call to FunctionContextImpl::GetConstFnAttr().
    if (called_fn == nullptr ||
        called_fn->getName() != FunctionContextImpl::GET_CONST_FN_ATTR_SYMBOL) {
      continue;
    }

    // 't' and 'i' arguments must be constant
    llvm::ConstantInt* t_arg =
        llvm::dyn_cast<llvm::ConstantInt>(call_instr->getArgOperand(1));
    llvm::ConstantInt* i_arg =
        llvm::dyn_cast<llvm::ConstantInt>(call_instr->getArgOperand(2));
    // This optimization is only applied to built-ins which should have constant args.
    DCHECK(t_arg != nullptr)
        << "Non-constant 't' argument to FunctionContextImpl::GetConstFnAttr()";
    DCHECK(i_arg != nullptr)
        << "Non-constant 'i' argument to FunctionContextImpl::GetConstFnAttr";

    // Replace the called function with the appropriate constant
    FunctionContextImpl::ConstFnAttr t_val =
        static_cast<FunctionContextImpl::ConstFnAttr>(t_arg->getSExtValue());
    int i_val = static_cast<int>(i_arg->getSExtValue());
    DCHECK(state_ != nullptr);
    // All supported constants are currently integers.
    call_instr->replaceAllUsesWith(GetI32Constant(FunctionContextImpl::GetConstFnAttr(
        state_->query_options().decimal_v2, state_->query_options().utf8_mode, ret_type,
        arg_types, t_val, i_val)));
    call_instr->eraseFromParent();
    ++replaced;
  }
  return replaced;
}
工作原理：
比如在string函数中：

StringVal StringFunctions::Upper(FunctionContext* context, const StringVal& str) {
  if (str.is_null) return StringVal::null();
  if (context->impl()->GetConstFnAttr(FunctionContextImpl::UTF8_MODE)) {
    return UpperUtf8(context, str);
  }
  return UpperAscii(context, str);
}

调用了GetConstFnAttr。其实GetConstFnAttr有两个版本：在C++代码实现的时候使用的是第一个版本：

int FunctionContextImpl::GetConstFnAttr(FunctionContextImpl::ConstFnAttr t, int i) {
  return GetConstFnAttr(state_->decimal_v2(), state_->utf8_mode(), return_type_,
      arg_types_, t, i);
}
在int LlvmCodeGen::InlineConstFnAttrs(const FunctionContext::TypeDesc& ret_type,
    const vector<FunctionContext::TypeDesc>& arg_types, llvm::Function* fn)使用的是第二个版本

int FunctionContextImpl::GetConstFnAttr(bool uses_decimal_v2, bool is_utf8_mode,
    const FunctionContext::TypeDesc& return_type,
    const vector<FunctionContext::TypeDesc>& arg_types, ConstFnAttr t, int i) {
  switch (t) {
    case RETURN_TYPE_SIZE:
      assert(i == -1);
      return GetTypeByteSize(return_type);
    case RETURN_TYPE_PRECISION:
      assert(i == -1);
      assert(return_type.type == FunctionContext::TYPE_DECIMAL);
      return return_type.precision;
    case RETURN_TYPE_SCALE:
      assert(i == -1);
      assert(return_type.type == FunctionContext::TYPE_DECIMAL);
      return return_type.scale;
    case ARG_TYPE_SIZE:
      assert(i >= 0);
      assert(i < arg_types.size());
      return GetTypeByteSize(arg_types[i]);
    case ARG_TYPE_PRECISION:
      assert(i >= 0);
      assert(i < arg_types.size());
      assert(arg_types[i].type == FunctionContext::TYPE_DECIMAL);
      return arg_types[i].precision;
    case ARG_TYPE_SCALE:
      assert(i >= 0);
      assert(i < arg_types.size());
      assert(arg_types[i].type == FunctionContext::TYPE_DECIMAL);
      return arg_types[i].scale;
    case DECIMAL_V2:
      return uses_decimal_v2;
    case UTF8_MODE:
      return is_utf8_mode;
    default:
      assert(false);
      return -1;
  }

********************************************************************************
