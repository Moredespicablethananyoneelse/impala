◆ Create()
BasicBlock * llvm::BasicBlock::Create	(	LLVMContext &	Context,
const Twine &	Name = "",
Function *	Parent = nullptr,
BasicBlock *	InsertBefore = nullptr )
inlinestatic
Creates a new BasicBlock.

If the Parent parameter is specified, the basic block is automatically inserted at either the end of the function (if InsertBefore is 0), or before the specified basic block.
https://llvm.org/doxygen/classllvm_1_1BasicBlock.html
*********************************************************

using llvm::BasicBlock::iterator = InstListType::iterator
Instruction iterators...

Definition at line 170 of file BasicBlock.h.
https://llvm.org/doxygen/classllvm_1_1BasicBlock.html
*******************************************
LLVM_ABI void 	llvm::ReplaceInstWithValue (BasicBlock::iterator &BI, Value *V)
 	Replace all uses of an instruction (specified by BI) with a value, then remove and delete the original instruction.
https://llvm.org/doxygen/BasicBlockUtils_8h.html
*************************
/// ReplaceInstWithValue - Replace all uses of an instruction (specified by BI)
/// with a value, then remove and delete the original instruction.
///
void llvm::ReplaceInstWithValue(BasicBlock::InstListType &BIL,
                                BasicBlock::iterator &BI, Value *V) 
https://github.com/numba/llvm-mirror/blob/master/lib/Transforms/Utils/BasicBlockUtils.cpp
*********************************************************
GlobalVariable *globalVar = new GlobalVariable(
    module,                     // 所属模块
    globalVarType,              // 变量类型
    /*isConstant*/ false,       // 是否为常量（true 表示只读）
    GlobalValue::ExternalLinkage, // 链接类型
    initValue,                  // 初始化常量值
    globalVarName               // 变量名称
);
https://mukulrathi.com/create-your-own-programming-language/llvm-ir-cpp-api-tutorial/?referrer=grok.com
**********************************************************

bool llvm::InitializeNativeTargetAsmParser	(		)	
inline
InitializeNativeTargetAsmParser - The main program should call this function to initialize the native target asm parser.

Definition at line 142 of file TargetSelect.h.

bool llvm::InitializeNativeTargetAsmPrinter	(		)	
inline
InitializeNativeTargetAsmPrinter - The main program should call this function to initialize the native target asm printer.

Definition at line 130 of file TargetSelect.h.

bool llvm::InitializeNativeTargetDisassembler	(		)	
inline
InitializeNativeTargetDisassembler - The main program should call this function to initialize the native target disassembler.
http://www.few.vu.nl/~lsc300/LLVM/doxygen/namespacellvm.html#aee99a98ed13b83d3ea311e62cb6429f7

****************************************************8
Status LlvmCodeGen::InitializeLlvm(const char* procname, bool load_backend)

这段我没有看懂：// Treat all functions as having the inline hint
  std::array<const char*, 2> argv = { { procname, "-inline-threshold=325" } };
  CHECK(llvm::cl::ParseCommandLineOptions(argv.size(), argv.data()));

  你提到的这段代码来自 LLVM 的初始化过程，确实有点 tricky。我们来逐行解释它的作用和目的：

cpp
// Treat all functions as having the inline hint
std::array<const char, 2> argv = { { procname, "-inline-threshold=325" } };
CHECK(llvm::cl::ParseCommandLineOptions(argv.size(), argv.data()));

📌 背景：LLVM 的命令行选项系统

LLVM 内部使用一个全局的命令行参数解析系统（llvm::cl），很多优化行为（比如内联、循环展开等）其实是通过命令行标志（flags）控制的——即使你不是从命令行启动程序，也可以在代码中“假装”传入这些参数。

🔍 逐行解释
1. 注释：
cpp
// Treat all functions as having the inline hint

意思是：“让所有函数都带有内联提示（inline hint）”。
在 LLVM 中，是否内联一个函数通常由函数的“内联阈值”（inline threshold）决定。提高这个阈值，会让编译器更激进地内联函数。
2. 构造 argv：
cpp
std::array<const char, 2> argv = { { procname, "-inline-threshold=325" } };
procname 是当前可执行文件的名称（比如 "impalad"），这是命令行参数解析器要求的第一个参数（类似 argv[0]）。
第二个参数是 "-inline-threshold=325"，这是一个 LLVM 内置的命令行选项，用于设置内联阈值为 325（默认值通常远小于此，例如 225 或更低）。
💡 内联阈值（inline threshold）是一个启发式指标，代表“允许被内联的函数其 IR 指令数量上限”。值越大，越容易内联大函数。
3. 解析并应用这些“伪命令行参数”：
cpp
CHECK(llvm::cl::ParseCommandLineOptions(argv.size(), argv.data()));

这行代码告诉 LLVM：“请像从命令行启动一样，解析并应用这些选项”。
虽然你的程序不是通过 ./program -inline-threshold=325 启动的，但通过调用 ParseCommandLineOptions，你可以在运行时动态注入这些配置。

✅ 这样做的效果是什么？
所有后续由 LLVM JIT 编译的函数，都会使用更高的内联阈值（325）。
编译器会更积极地将小到中等大小的函数内联展开，减少函数调用开销。
对于像 Impala 这样的查询引擎，大量小函数（如表达式计算）的内联能显著提升性能。

❓为什么不用 API 直接设置，而要“伪造命令行”？

因为 LLVM 的许多优化策略（尤其是 Pass 管理器和内联决策）深度依赖其全局命令行标志系统。
虽然新版本 LLVM 正在迁移到更模块化的接口，但在较老的代码库（如 Impala 使用的 LLVM 版本）中，这是最简单、最可靠的方式来调整这些内部参数。

🧠 类比理解

你可以把它想象成：
“在程序启动时，偷偷告诉 LLVM：‘嘿，假装用户加了 -inline-threshold=325 这个参数’。”

这样，LLVM 的优化器就会按照这个设定工作，而无需修改每个函数的属性或重写优化流程。

总结

这段代码的作用是：
在 LLVM 初始化阶段，通过模拟命令行参数的方式，全局提高函数内联的激进程度，从而优化生成的 JIT 代码性能。
这是一种在嵌入式 LLVM 场景中常见的“黑科技”用法。
、
*****************************************************
我没有看懂：  llvm::remove_fatal_error_handler();
  llvm::install_fatal_error_handler(LlvmCodegenHandleError);并且[[noreturn]] static void LlvmCodegenHandleError(
    void* user_data, const string& reason, bool gen_crash_diag) {
  LOG(FATAL) << "LLVM hit fatal error: " << reason.c_str();
}好像什么都没干
你这个问题问得非常好！表面上看，这段代码“好像什么都没干”，但实际上它在做一件非常关键的事情：接管 LLVM 内部的致命错误处理机制，让程序在遇到 LLVM 无法恢复的错误时，能以我们期望的方式崩溃并记录日志，而不是直接 abort() 或静默退出。

我们来一步步拆解：

🔧 原始问题：LLVM 默认的致命错误行为是什么？

当 LLVM 遇到无法恢复的错误（比如 IR 验证失败、内存不足、断言失败等），它会调用一个全局的致命错误处理器（fatal error handler）。
默认行为：打印一条简短错误信息到 stderr，然后调用 abort()，直接终止进程。
这对生产系统（如数据库、服务端程序）来说是不可接受的——我们希望：
记录完整的错误上下文（比如写入日志文件）；
触发自己的 crash reporting 机制；
确保错误信息格式统一（比如带时间戳、线程 ID 等）。

✅ 你的代码做了什么？

cpp
llvm::remove_fatal_error_handler(); // 1. 移除默认的错误处理器
llvm::install_fatal_error_handler(LlvmCodegenHandleError); // 2. 安装我们自己的
第一步：remove_fatal_error_handler()
清除 LLVM 当前注册的任何致命错误回调（通常是默认的 abort 行为）。
第二步：install_fatal_error_handler(LlvmCodegenHandleError)
注册你自定义的函数 LlvmCodegenHandleError 作为新的错误处理器。

📝 自定义错误处理器：LlvmCodegenHandleError

cpp
[[noreturn]] static void LlvmCodegenHandleError(
void user_data, const string& reason, bool gen_crash_diag) {
LOG(FATAL) << "LLVM hit fatal error: " << reason.c_str();
}

虽然看起来只有一行日志，但它的作用非常明确：

1. 捕获错误原因：reason 是 LLVM 传过来的具体错误信息（比如 "Invalid type for instruction"）。
2. 使用项目自己的日志系统：LOG(FATAL) 是 Google Logging（glog）或类似日志库的宏，它会：
将错误写入日志文件（带时间、线程、堆栈等）；
自动终止程序（FATAL 级别日志通常会调用 exit() 或 abort()）。
3. [[noreturn]] 告诉编译器这个函数不会返回，符合“致命错误必须终止程序”的语义。
💡 所以它不是“什么都没干”，而是把“粗糙的 stderr + abort”替换成了“结构化日志 + 受控崩溃”。

🌰 举个实际场景

假设你在 JIT 编译一段非法 IR（比如给 add 指令传了指针类型），LLVM 会触发致命错误。
没有这段代码：
程序直接 abort()，终端可能只显示：

LLVM FATAL ERROR: Invalid type for instruction
Aborted (core dumped)

日志系统完全不知道发生了什么！
有这段代码：
你的日志文件里会记录：

F1224 10:30:45.123456 12345 codegen.cc:100] LLVM hit fatal error: Invalid type for instruction
Check failure stack trace: *
@ 0x7f1234567890 google::LogMessage::Fail()
...

运维/开发人员能立刻定位问题，并且 crash 信息被集中管理。

❓为什么看起来“什么都没干”？

因为：
错误处理函数只在极端情况下触发（正常运行时根本不会执行）；
它不返回、不抛异常、不修复错误——它的使命就是“优雅地死掉并留下遗言”。

这就像消防警报：平时安静无声，一旦拉响，就必须确保所有人都知道发生了火灾。

✅ 总结

这段代码的作用是：
将 LLVM 的“粗暴崩溃”行为，替换成项目统一的“日志记录 + 受控终止”流程，提升可维护性和可观测性。

在大型 C++ 项目（尤其是嵌入 LLVM 的系统，如数据库、JIT 引擎）中，这是标准且必要的做法。

*******************************************************************
FeSupport.java

你补充的代码非常关键，它完整地展示了 Impala 中 libfesupport.so 的用途、加载机制和 JNI 接口定义。现在我可以从心回答你最初的问题（虽然你没有明确写出问题，但从上下文可以合理推断）：
“libfesupport.so 是什么？它的作用是什么？为什么需要它？”

✅ 简明回答

libfesupport.so 是 Apache Impala 项目中的一个 JNI（Java Native Interface）共享库，用于在 Impala 的 Java 前端（Frontend, FE） 和 C++ 后端（Backend, BE / Impalad） 之间建立高效通信桥梁。

它的核心作用是：
让 Java 代码能够调用 C++ 实现的高性能函数（如表达式求值、谓词评估、元数据操作等），同时避免重复实现逻辑。

🔍 深入解析（基于你提供的代码）
1. 命名与定位
文件名：libfesupport.so（Linux 下的动态链接库）
包路径：org.apache.impala.service.FeSupport
C++ 头文件：impala/service/fe_support.h
初始化函数：InitFeSupport()（在 C++ 侧注册 JNI 函数）
2. 核心功能
从 FeSupport.java 可见，该库提供了大量 native 方法，例如：

功能 Native 方法 说明
------ ------------- ------
表达式求值 NativeEvalExprsWithoutRow 在 C++ 中执行 SQL 表达式（无行上下文）
谓词评估 EvalPredicate → 调用 native 用于分区裁剪、谓词下推等优化
元数据操作 NativePrioritizeLoad, NativeGetPartitionStats 与 Catalog Server 交互
Jar 缓存 NativeCacheJar 缓存 HDFS 上的 UDF jar 到本地
Bloom Filter 计算 MinLogSpaceForBloomFilter 利用 C++ 高效计算
时间/日期解析 nativeParseDateString 高性能日期解析
这些操作若用纯 Java 实现，要么性能差，要么难以复用 Impala BE 已有的成熟 C++ 逻辑。
3. 加载机制
java
public static synchronized void loadLibrary() {
if (loaded_) return;
LOG.info("Loading libfesupport.so");
NativeLibUtil.loadLibrary("libfesupport.so"); // ← 关键！
loaded_ = true;
NativeFeInit(externalFE_);
}
仅在需要时加载（如单元测试、外部 FE 模式）
正常运行时（Impalad 内部）不需要显式加载，因为 InitFeSupport() 已在启动时通过 JNI_OnLoad 或手动注册了 native 函数（见 fe-support.cc）
正如注释所说：
> “If the planner is executed inside Impalad, Impalad would have registered all the JNI native functions already. There's no need to load the shared library.”
4. 为何不直接用 Java 实现？
性能：表达式求值、位运算、内存操作等 C++ 更快
一致性：BE 和 FE 使用同一套执行逻辑，避免语义差异（如时间处理、类型转换）
复用：Impala 的执行引擎（PlanFragment、ExprContext 等）全是 C++，没必要重写
5. 许可证说明
你提供的代码顶部有 Apache License 2.0 声明，表明这是 Apache Impala 的官方开源代码，符合 ASF 规范。

🧩 总结

项目 说明
------ ------
是什么 Impala 的 JNI 支持库（C++ ↔ Java 桥梁）
为什么存在 让 Java FE 能高效调用 C++ BE 的核心功能
何时加载 单元测试或外部 FE 模式下显式加载；Impalad 内部已预注册
关键函数 EvalPredicate, EvalExpr, CacheJar, GetPartitionStats 等
是否必需 对 Impala 完整功能而言：是；对纯 Java 测试：可选但推荐

💡 附加提示
如果你在开发或调试 Impala 时遇到：
UnsatisfiedLinkError: libfesupport.so not found
Native method not implemented

请检查：
1. 是否正确编译了 fe-support.cc 并生成了 libfesupport.so
2. LD_LIBRARY_PATH 是否包含该 so 文件路径
3. 是否在非 Impalad 环境（如 mvn test）中忘记调用 FeSupport.loadLibrary()

希望这个“从心”回答彻底解开了你对 libfesupport.so 的疑惑！如果还有具体使用场景或错误信息，欢迎继续提问。
******************************************************************************************88888
是的，你说得非常对！

在你提到的上下文中，“符号”（symbols）主要指的就是 libfesupport.so 动态库中定义的函数（以及可能的全局变量），这些符号会被 LLVM 在运行时（JIT 编译阶段或执行阶段）所引用。
具体解释：
符号（symbol） 是链接器和动态加载器用来标识代码或数据的一个名字。常见的符号类型包括：
函数名（如 DecimalAdd, HandleUdfCall 等）
全局变量名
某些特殊标记或桩（stub）
当 Impala 使用 LLVM 进行 JIT（Just-In-Time）编译 生成查询执行代码时，生成的机器码可能会调用一些辅助函数（比如处理 DECIMAL 类型、字符串操作、UDF 调用框架等），而这些函数并不在 LLVM 生成的代码内部，而是实现在 C++ 编写的运行时支持库中 —— 比如 libfesupport.so。
在 生产环境 中，这些函数可能已经通过主程序（如 impalad）被链接进进程地址空间，因此 LLVM 可以直接解析它们。
但在 单元测试或某些隔离的测试环境 中，主程序可能没有加载这些符号，所以必须显式加载 libfesupport.so，让 LLVM 的 DynamicLibrary::LoadLibraryPermanently 把这些符号注册到全局符号表中，供后续 JIT 代码使用。
举个例子：

假设 JIT 生成的代码中有这样一条调用：

llvm
call void @DecimalValue_Add(%struct.DecimalVal, %struct.DecimalVal, %struct.DecimalVal*)

那么运行时就需要一个名为 DecimalValue_Add 的函数地址。这个函数很可能就定义在 libfesupport.so 中。如果不加载该库，LLVM 就找不到这个符号，执行时会崩溃或报错。

总结：

✅ 是的，这里的“符号”主要就是指 libfesupport.so 里实现的函数（也可能包括少量全局变量），它们为 LLVM JIT 生成的代码提供必要的运行时支持。加载这个动态库，就是为了把这些函数“暴露”给 LLVM 的执行引擎，使其能成功链接并运行生成的代码。
*****************************************************************
这段代码的作用是探测当前主机 CPU 的型号和特性（指令集支持），并筛选出允许用于运行时代码生成（JIT 编译）的 CPU 特性。下面我逐行解释，帮助你理解它的逻辑：

1. 获取 CPU 型号名称
cpp
cpu_name_ = llvm::sys::getHostCPUName().str();
LOG(INFO) << "CPU class for runtime code generation: " << cpu_name_;
llvm::sys::getHostCPUName() 是 LLVM 提供的函数，用于自动检测当前机器的 CPU 型号（例如 "skylake", "zen2", "cortex-a76" 等）。
这个名字会被用作 LLVM Target Machine 的 “CPU” 参数，告诉 LLVM 针对哪种 CPU 生成优化代码。
日志会打印类似：

CPU class for runtime code generation: skylake

2. 获取 CPU 支持的所有特性（指令集扩展）
cpp
GetHostCPUAttrs(&cpu_attrs_);
LOG(INFO) << "Detected CPU flags: " << boost::join(cpu_attrs_, ",");
GetHostCPUAttrs 是项目自定义函数（通常封装了 LLVM 的 sys::getHostCPUFeatures），用于获取当前 CPU 支持的所有特性标志（feature flags）。
这些“特性”指的是 CPU 支持的指令集扩展，比如：
sse4.2
avx
avx2
bmi2
popcnt
fma
等等（具体取决于你的 CPU 架构，如 x86、ARM）
cpu_attrs_ 是一个 std::vector<std::string>，保存这些特性名。
日志可能输出：

Detected CPU flags: sse4.2,avx,avx2,bmi2,popcnt,fma

3. 应用白名单过滤
cpp
cpu_attrs_ = ApplyCpuAttrWhitelist(cpu_attrs_);
虽然 CPU 支持很多高级指令，但并不是所有特性都适合在 JIT 编译中使用！原因可能包括：
某些指令在旧版本 LLVM 中有 bug；
某些特性在分布式环境中可能导致兼容性问题（比如集群中有些机器不支持 AVX2）；
安全或稳定性考虑（如避免使用某些实验性指令）。
ApplyCpuAttrWhitelist 是一个过滤函数：它只保留项目认为“安全可用”的 CPU 特性，丢弃其他。
例如，即使 CPU 支持 avx512，但如果白名单没包含它，就会被剔除。

4. 生成最终启用的特性字符串
cpp
target_features_attr_ = boost::join(cpu_attrs_, ",");
LOG(INFO) << "CPU flags enabled for runtime code generation: " << target_features_attr_;
把过滤后的特性列表用逗号拼接成一个字符串（如 "sse4.2,avx2,popcnt"）。
这个字符串后续会传递给 LLVM 的 TargetMachine，作为 -mattr=+sse4.2,+avx2,... 参数，指导 LLVM 在 JIT 编译时使用哪些指令集优化。
日志示例：

CPU flags enabled for runtime code generation: sse4.2,avx2,popcnt

🎯 总结：这段代码的目的
让运行时代码生成（JIT）尽可能利用当前 CPU 的高性能指令，但又确保只使用经过验证、安全可靠的指令集，避免兼容性或稳定性问题。

这在像 Impala、Spark、ClickHouse 这类依赖 LLVM JIT 加速查询执行的系统中非常常见。

💡 补充说明
如果你在一台支持 AVX2 的机器上运行，但日志显示最终启用的只有 SSE4.2，那很可能是因为 ApplyCpuAttrWhitelist 故意禁用了 AVX2（比如为了兼容集群中的老机器）。
这种“探测 + 白名单”机制，是在性能和稳定性/兼容性之间做权衡的典型做法。

希望这下清楚啦！如果还有疑问，欢迎继续问 😊
****************************************************************************
[IR代码的编译](././../codegen/CMakeLists.md)

