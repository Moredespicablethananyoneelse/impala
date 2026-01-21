提供两种从tuple中获取值的接口，一种是CodegenAnyVal CodegenValue(LlvmCodeGen* codegen, LlvmBuilder* builder,
      llvm::Function* fn, llvm::Value* eval_ptr, llvm::Value* row_ptr,
      llvm::BasicBlock* entry_block = nullptr);tuple中存放的是DecimalValue值，这个函数从tuple中读取DecimalVal的封装类型CodeAnyVal。用于jit执行路径


      另一种是：  GENERATE_GET_VAL_INTERPRETED_OVERRIDES_FOR_ALL_SCALAR_TYPES
  virtual CollectionVal GetCollectionValInterpreted(
      ScalarExprEvaluator*, const TupleRow*) const override;
  virtual StructVal GetStructValInterpreted(
      ScalarExprEvaluator*, const TupleRow*) const override;
      tuple中存放的是DecimalValue值，这个函数从tuple中读取DecimalVal.用于解释执行路径

打算梳理这几种类型的关系，请参考slot-ref和tuple的实现，以及scalar-expr-evaluator的实现。
*****************************************************************************************8
      