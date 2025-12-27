这段代码实现了一个 LLVM JIT 符号事件监听器（CodegenSymbolEmitter），主要用于在 Impala 查询引擎中：
为 JIT 编译生成的机器码生成调试符号（debug symbols）
支持 Linux perf 性能分析工具（通过 /tmp/perf-<pid>.map 文件）
可选地将函数反汇编（disassembly）输出到文件

下面从 目的、核心机制、关键功能 三个方面详细介绍。

🎯 一、目的：为什么需要这个类？

Impala 使用 LLVM MCJIT 在运行时动态编译查询执行代码（如表达式、聚合函数等）。这些代码是“即时生成”的，操作系统和性能分析工具（如 perf）默认不知道它们的存在，因此：
perf top / perf record 看不到这些函数，只能显示 <unknown>；
调试或性能调优困难；
无法关联机器码与源码位置（行号、函数名等）。

CodegenSymbolEmitter 就是为了解决这些问题而设计的：它作为 LLVM 的 JITEventListener，在 JIT 模块加载/卸载时自动捕获符号信息，并导出供外部工具使用。

⚙️ 二、核心机制：如何工作？
1. 继承 llvm::JITEventListener
cpp
class CodegenSymbolEmitter : public llvm::JITEventListener

LLVM 允许注册监听器，在以下时机回调：
NotifyObjectEmitted()：当一个 JIT 编译的对象文件（含机器码）被加载到内存；
NotifyFreeingObject()：当该对象被释放（例如查询结束）。
2. 注册到 LLVM ExecutionEngine
虽然代码中没展示注册过程，但在 Impala 初始化 LLVM 引擎时会调用：
cpp
execution_engine->RegisterJITEventListener(new CodegenSymbolEmitter(...));

🔍 三、关键功能详解
✅ 功能 1：生成 perf 可识别的符号映射文件（/tmp/perf-<pid>.map）
原理：
Linux perf 工具会读取 /tmp/perf-<pid>.map 文件，格式为：

<address> <size> <symbol_name>

例如：

7f1234000000 128 Query_Expr_Eval:fragment_123
7f1234000080 256 Agg_HashJoin:fragment_123
实现：
在 NotifyObjectEmitted() 中：
遍历所有函数符号（SymbolRef::ST_Function）；
记录其 虚拟地址（addr）、大小（size）、名称（带 ID 后缀）；
存入全局 perf_map_（静态 map）；
调用 WritePerfMapLocked() 写入临时文件并原子替换 /tmp/perf-<pid>.map。
在 NotifyFreeingObject() 中：
从 perf_map_ 中移除已释放模块的符号；
更新 perf.map，避免残留无效地址。
💡 为什么要加 id_（如 fragment ID）？
因为不同查询可能生成同名函数（如 EvalExpr），加上唯一 ID 可区分来源，避免混淆。

✅ 功能 2：输出函数反汇编（Disassembly）到文件
触发条件：
用户通过配置指定 asm_path_（例如 --asm_module_dir=/tmp/asm）。
实现：
在 ProcessSymbol() 中，若 asm_file.is_open()，则调用 EmitFunctionAsm()；
使用 LLVM C Disassembler API（LLVMDisasmInstruction）逐条反汇编机器码；
结合 DWARF 调试信息（来自 DWARFContext），在每条指令前插入源码位置（文件名、行号、列号）；
输出示例：
asm
Disassembly for EvalPredicate:frag_abc (0x7f1234000000):
exprs/predicate.cc:exprs/predicate.cc:EvalPredicate:42:5
0: mov rax, rdi
3: test rax, rax
...

这对 性能分析、JIT 优化验证、bug 调试 极其有用。

✅ 功能 3：资源安全与生命周期管理
引用计数：non_freed_objects_ 跟踪已分配但未释放的对象数量；
析构检查：在 ~CodegenSymbolEmitter() 中，如果 non_freed_objects_ != 0，说明仍有活跃的 JIT 对象，此时销毁监听器会导致 use-after-free（见注释 IMPALA-12306）；
线程安全：对 perf_map_ 的访问由 SpinLock perf_map_lock_ 保护；
原子写入：先写临时文件 .tmp，再 rename，确保 perf 读取时不会看到半成品。

📌 总结：这个类的作用

功能 用途 用户受益
------ ------ --------
✅ perf 符号支持 生成 /tmp/perf-<pid>.map perf top 能看到 JIT 函数名，精准定位热点
✅ 反汇编输出 将 JIT 代码转为可读汇编 + 源码位置 调试性能问题、验证编译优化效果
✅ 符号去重 用 id_ 区分不同查询的同名函数 避免分析时混淆
✅ 内存安全 跟踪对象生命周期，防止 use-after-free 提高系统稳定性
这是 生产级 JIT 系统必备的可观测性组件，体现了 Impala 在性能工程和可维护性上的深度投入。

如果你正在开发基于 LLVM JIT 的系统（数据库、脚本引擎、AI 推理框架等），强烈建议参考此设计！
**************************************************************************88
perf top 和 perf record 是 Linux 系统中 性能分析（profiling）工具 perf 的两个核心子命令，用于实时或离线分析程序的 CPU 使用情况、热点函数、调用栈等，帮助开发者定位性能瓶颈。

它们都基于 Linux 内核的 perf_events 子系统（也叫 Performance Counters for Linux, PCL），无需修改代码、无需重启服务，开销低，功能强大。

🔍 1. perf record：记录性能数据（离线分析）
作用：
在程序运行期间 采样并记录性能事件（如 CPU 周期、缓存未命中、分支预测失败、函数调用等），保存到一个文件（默认 perf.data）中，供后续分析。
常见用法：
bash
记录整个系统的 CPU 热点（按 Ctrl+C 停止）
sudo perf record -a
只记录某个进程（例如 PID=1234）
sudo perf record -p 1234
记录某个命令的执行过程
perf record ./my_program
记录时包含调用栈（非常重要！）
perf record -g ./my_program
记录特定事件，比如缓存未命中
perf record -e cache-misses ./my_program
输出：
生成 perf.data 文件（二进制格式），不能直接阅读，需要用 perf report 查看。

🔍 2. perf report：分析 perf record 的结果

虽然你问的是 perf top，但 perf record 通常和 perf report 配对使用：

bash
查看记录结果（交互式界面）
perf report
以扁平列表显示（不展开调用栈）
perf report --no-children

输出示例：

Overhead Command Shared Object Symbol
35.2% my_app [JIT] [.] Query_Eval_Expr
22.1% my_app libc-2.31.so [.] memcpy
10.5% my_app [unknown] [.] 0x7f12abcd1234
注意：如果看到 [unknown] 或地址而非函数名，说明符号缺失（这正是 CodegenSymbolEmitter 要解决的问题！）。

🔥 3. perf top：实时动态查看性能热点（类似 top）
作用：
实时显示当前系统或进程中 最耗 CPU 的函数，类似 top 命令，但粒度是“函数级别”。
常见用法：
bash
实时监控整个系统的热点函数
sudo perf top
只监控某个进程
sudo perf top -p 1234
显示调用关系（需要内核支持）
sudo perf top -g
输出示例（动态刷新）：

Samples: 1K of event 'cpu-cycles', 4000 Hz
Overhead Shared Object Symbol
42.3% [JIT] [.] Agg_HashJoin
18.7% libpthread.so.0 [.] pthread_mutex_lock
9.2% [kernel] [k] _raw_spin_lock
✅ 如果你看到 [JIT] 或具体的函数名（如 Agg_HashJoin），说明 JIT 符号已正确导出（感谢 CodegenSymbolEmitter！）
❌ 如果看到 [unknown] 或十六进制地址，说明符号丢失，perf 无法识别 JIT 代码。

🧩 关键概念补充
为什么需要 /tmp/perf-<pid>.map？
普通 C/C++ 程序的符号在 ELF 文件中，perf 能自动读取。
JIT 编译的代码（如 Java HotSpot、V8、Impala、PyTorch）是在运行时生成的，没有 ELF 文件。
Linux perf 约定：如果进程在 /tmp 下提供 perf-<pid>.map 文件，就从中读取地址 → 函数名的映射。
格式很简单：

<起始地址（16进制）> <大小（16进制或10进制）> <函数名>

例如：

7f1234000000 200 Query_Eval_Expr:frag_abc
这就是 CodegenSymbolEmitter::WritePerfMap() 的作用：让 perf 能“看见”JIT 函数！

✅ 总结对比

命令 类型 用途 是否需要事后分析
------ ------ ------ ------------------
perf record 离线 记录一段时间内的性能数据 是（需 perf report）
perf top 实时 动态显示当前最热函数 否（直接看）
典型工作流：
1. 用 perf top 快速发现哪个函数占 CPU 高；
2. 用 perf record -g 深入记录调用栈；
3. 用 perf report 分析完整调用链；
4. 结合源码或反汇编优化热点路径。

如果你在开发高性能系统（数据库、引擎、AI 框架等），熟练使用 perf 是必备技能。而像 Impala 这样的系统，通过 CodegenSymbolEmitter 主动向 perf 提供符号，极大提升了 JIT 代码的可观测性。
***************************************************************************
void CodegenSymbolEmitter::NotifyObjectEmitted(...) 是 LLVM JIT 编译过程中一个关键的回调函数，由 LLVM 的 MCJIT（或 ORC JIT）引擎在 成功将一个编译后的机器码对象加载到内存后自动调用。

它的核心任务是：从刚刚加载的 JIT 对象中提取调试符号、函数地址、大小等信息，并用于生成性能分析（perf）映射文件和/或反汇编输出。

下面逐层详细解析这个函数的逻辑和作用：

🔧 函数签名回顾

cpp
void CodegenSymbolEmitter::NotifyObjectEmitted(
const llvm::object::ObjectFile& obj,
const llvm::RuntimeDyld::LoadedObjectInfo& loaded_obj) override;
obj：表示 已编译但尚未加载 的目标文件（如 ELF 片段），包含符号表、节区等。
loaded_obj：包含 实际加载到内存后的重定位信息，比如每个节区最终的虚拟地址。
这两个参数共同提供了“符号名 ↔ 内存地址”的完整映射。

📌 1. 增加未释放对象计数（生命周期跟踪）

cpp
non_freed_objects_++;
每当一个 JIT 对象被加载，就 +1；
后续在 NotifyFreeingObject() 中 -1；
析构时检查是否为 0，防止 use-after-free（见 IMPALA-12306）；
这是一种 资源泄漏检测机制。

📌 2. 打开反汇编输出文件（可选）

cpp
ofstream asm_file;
if (!asm_path_.empty()) {
asm_file.open(asm_path_, fstream::out fstream::trunc);
if (asm_file.fail()) {
LOG(ERROR) << "Could not save disassembly to: " << asm_path_;
}
}
如果用户设置了 asm_path_（例如通过配置指定路径），就打开文件准备写入反汇编；
失败只报错，不中断执行（非关键路径）。

📌 3. 获取用于调试的对象（Debug Object）

cpp
llvm::object::OwningBinary<llvm::object::ObjectFile> debug_obj_owner =
loaded_obj.getObjectForDebug(obj);
const llvm::object::ObjectFile& debug_obj = *debug_obj_owner.getBinary();
getObjectForDebug() 返回一个 包含调试信息（如 DWARF）的 ObjectFile；
即使原始 obj 被优化或剥离，LLVM 仍可能保留调试副本；
这个 debug_obj 用于后续获取 源码行号、文件名、函数名 等。

📌 4. 创建 DWARF 调试上下文

cpp
llvm::DWARFContextInMemory dwarf_ctx(debug_obj);
DWARFContext 是 LLVM 读取 DWARF 调试信息的入口；
InMemory 表示数据来自内存中的 ObjectFile，而非磁盘文件；
后续 getLineInfoForAddressRange() 就靠它来映射 地址 → 源码位置。

📌 5. 遍历所有符号并处理函数

cpp
for (const auto& pair : computeSymbolSizes(debug_obj)) {
ProcessSymbol(&dwarf_ctx, pair.first, pair.second, &perf_map_entries, asm_file);
}
关键点：
computeSymbolSizes(debug_obj)：
LLVM 工具函数，遍历符号表；
返回 <SymbolRef, size> 对，其中 size 是函数或数据的大小（通过下一符号地址推算）；
只有 类型为 ST_Function 的符号 才会被 ProcessSymbol 处理；
其他符号（如全局变量、标签）被忽略。

📌 6. 关闭反汇编文件（如果打开过）

cpp
if (asm_file.is_open()) {
asm_file.close();
LOG(INFO) << "Saved disassembly to " << asm_path_;
}
所有函数的反汇编都写入同一个文件（按顺序）；
日志提示用户文件位置。

📌 7. 更新 perf 符号映射（核心！）

cpp
if (emit_perf_map_) {
lock_guard<SpinLock> perf_map_lock(perf_map_lock_);
DCHECK(perf_map_.find(obj.getData().data()) == perf_map_.end());
perf_map_[obj.getData().data()] = std::move(perf_map_entries);
WritePerfMapLocked();
}
详解：
obj.getData().data()：
获取该 ObjectFile 在内存中的原始字节起始地址；
作为 唯一 key，用于后续 NotifyFreeingObject 时精准删除；
perf_map_：
全局静态 map，存储所有活跃 JIT 模块的符号；
格式：{ object_data_ptr → [PerfMapEntry...] }
WritePerfMapLocked()：
将整个 perf_map_ 写入 /tmp/perf-<pid>.map；
使用 临时文件 + rename 实现原子更新，避免 perf 读到半截文件；
文件格式：

<addr_hex> <size_dec> <symbol_name>
✅ 正是这一步，让 perf top / perf record 能看到类似 Agg_HashJoin:frag_abc 的函数名，而不是 [unknown]！

🧠 总结：这个函数做了什么？

步骤 功能 目的
------ ------ ------
1️⃣ non_freed_objects_++ 跟踪对象生命周期，防 use-after-free
2️⃣ 打开反汇编文件 为后续输出做准备
3️⃣–4️⃣ 获取调试对象 + DWARF 上下文 支持源码级调试信息
5️⃣ 遍历函数符号 提取每个 JIT 函数的信息
6️⃣ 写反汇编（可选） 供开发者分析生成的机器码
7️⃣ 更新 perf.map 让 Linux perf 工具能识别 JIT 函数！

💡 为什么这对 Impala 至关重要？

Impala 在运行时为每个查询片段（fragment）动态生成大量 LLVM IR 并 JIT 编译成机器码。这些代码：
不在任何 .so 或可执行文件中；
没有传统符号表；
默认对 perf、gdb 等工具“不可见”。

NotifyObjectEmitted 通过主动导出符号到 /tmp/perf-<pid>.map，打通了 JIT 代码与系统级性能工具的桥梁，使得：
DBA 可用 perf top 实时看到哪个表达式或算子最耗 CPU；
开发者可结合反汇编优化关键路径；
系统可观测性大幅提升。
这是 生产级 JIT 系统工程实践的典范。
*************************************************************************
CodegenSymbolEmitter::ProcessSymbol 是一个 处理单个符号（symbol）的核心辅助函数，它的作用是从 LLVM JIT 生成的对象文件中 提取一个函数符号的信息，并根据配置决定是否：

1. 将其加入 perf 性能分析所需的符号映射（perf_map_entries）；
2. 将其反汇编（disassembly）写入指定的输出文件。

下面逐行详细解释其逻辑和设计意图。

🔧 函数签名

cpp
void CodegenSymbolEmitter::ProcessSymbol(
llvm::DIContext debug_ctx, // 调试信息上下文（用于源码位置映射）
const llvm::object::SymbolRef& symbol, // 当前要处理的符号引用
uint64_t size, // 该符号所占内存大小（通常为函数长度）
vector<PerfMapEntry> perf_map_entries, // 输出：待写入 perf.map 的条目列表
ofstream& asm_file // 输出：反汇编写入的文件流（可能未打开）
)
注意：这个函数只处理 函数类型的符号，其他类型（如全局变量、标签等）会被跳过。

✅ 第一步：检查符号类型是否为函数

cpp
llvm::Expected<llvm::object::SymbolRef::Type> symType = symbol.getType();
if (!symType symType.get() != llvm::object::SymbolRef::ST_Function) return;
symbol.getType() 返回符号类型（LLVM 用 Expected<T> 处理可能的错误）；
ST_Function 表示这是一个可执行的函数；
如果不是函数（比如数据符号），直接返回，不处理。
💡 这是因为 perf 和反汇编只关心 可执行代码，数据符号无意义。

✅ 第二步：获取符号名称和内存地址

cpp
llvm::Expected<llvm::StringRef> name_or_err = symbol.getName();
llvm::Expected<uint64_t> addr_or_err = symbol.getAddress();
if (!name_or_err !addr_or_err) return;
getName()：返回函数名（如 "EvalPredicate"）；
getAddress()：返回该函数在内存中的 实际加载地址（虚拟地址）；
使用 Expected<T> 是因为某些符号可能损坏或信息缺失；
任一失败就跳过该符号（防御性编程）。

✅ 第三步：构造唯一化的函数名（关键！）

cpp
uint64_t addr = addr_or_err.get();
string fn_symbol = Substitute("$0:$1", name_or_err.get().data(), id_);
原始函数名（如 "AggCompute"）在不同查询中可能重复；
id_ 是构造 CodegenSymbolEmitter 时传入的唯一标识（如 fragment instance ID）；
最终符号名形如："AggCompute:frag_abc123"。
🎯 目的：避免多个 JIT 函数同名导致 perf 分析混淆。
例如：两个不同查询都生成了 ExprEval，但来源不同，必须区分。

✅ 第四步：如果启用 perf 映射，则记录条目

cpp
if (emit_perf_map_) {
PerfMapEntry entry;
entry.symbol = fn_symbol;
entry.addr = addr;
entry.size = size;
perf_map_entries->push_back(entry);
}
PerfMapEntry 结构体包含：符号名、起始地址、大小；
这些数据最终会写入 /tmp/perf-<pid>.map，格式为：

<addr_hex> <size_dec> <symbol_name>
例如：

7f12a0001000 256 AggCompute:frag_abc123
✅ 这使得 perf top 能正确显示 JIT 函数名，而不是 [unknown]。

✅ 第五步：如果反汇编文件已打开，则输出函数汇编

cpp
if (asm_file.is_open()) {
EmitFunctionAsm(debug_ctx, fn_symbol, addr, size, asm_file);
}
调用 EmitFunctionAsm 执行实际反汇编；
传入：
debug_ctx：用于获取每条指令对应的源码位置（文件、行号等）；
fn_symbol：带 ID 的函数名；
addr + size：定义反汇编范围；
asm_file：输出流。
📝 反汇编结果示例：
asm
Disassembly for AggCompute:frag_abc123 (0x7f12a0001000):
exprs/agg.cc:exprs/agg.cc:AggCompute:42:5
0: mov rax, rdi
3: add rax, 1
...

🧠 总结：ProcessSymbol 的核心职责

功能 实现方式 目的
------ -------- ------
过滤非函数符号 检查 getType() == ST_Function 只处理可执行代码
提取元信息 getName() + getAddress() 获取函数名和内存位置
符号去重 拼接 id_（如 fragment ID） 避免不同查询的同名函数冲突
支持 perf 构造 PerfMapEntry 并追加 让 perf 能识别 JIT 函数
支持调试 调用 EmitFunctionAsm 输出带源码位置的反汇编

💡 设计亮点

1. 防御性编程：所有 LLVM 接口调用都检查 Expected<T> 是否成功；
2. 按需启用：emit_perf_map_ 和 asm_file.is_open() 控制开销；
3. 唯一符号命名：解决 JIT 系统中最常见的“符号冲突”问题；
4. 解耦设计：本函数只负责“收集信息”，写文件由其他函数完成。

这个函数虽小，却是 连接 LLVM JIT 与系统级性能工具的关键桥梁。
*********************************************************************8
CodegenSymbolEmitter::EmitFunctionAsm 是一个 将 JIT 编译生成的机器码反汇编为可读汇编指令，并附带源代码调试信息（如文件名、行号） 的函数。它的核心目标是：帮助开发者理解运行时生成的代码，便于性能调优和调试。

下面逐段详细解析其工作原理和设计意图。

🔧 函数签名

cpp
void EmitFunctionAsm(
llvm::DIContext debug_ctx, // 调试信息上下文（用于地址 → 源码映射）
const string& fn_symbol, // 唯一化的函数名（如 "AggCompute:frag_abc"）
uint64_t addr, // 函数在内存中的起始虚拟地址
uint64_t size, // 函数机器码的字节长度
ofstream& asm_file // 输出流（已打开的反汇编文件）
)
前置条件：DCHECK(asm_file.is_open()) 确保文件可写。

📌 第一步：获取地址范围对应的源码位置信息

cpp
llvm::DILineInfoTable di_lines = debug_ctx->getLineInfoForAddressRange(addr, size);
auto di_line_it = di_lines.begin();
debug_ctx 是之前创建的 DWARFContextInMemory，包含 DWARF 调试元数据；
getLineInfoForAddressRange(addr, size) 返回一个 地址 → 源码位置 的映射表；
类型 DILineInfoTable 实际是 std::vector<std::pair<uint64_t, DILineInfo>>；
每个条目：{ 指令地址, {FileName, Line, Column, FunctionName} }
di_line_it 用于遍历这些调试信息。
✅ 这使得每条汇编指令都能关联到原始 C++/IR 源码位置。

📌 第二步：创建 LLVM 反汇编器（使用 C API）

cpp
string triple = llvm::sys::getProcessTriple(); // 如 "x86_64-unknown-linux-gnu"
LLVMDisasmContextRef disasm = LLVMCreateDisasm(triple.c_str(), NULL, 0, NULL, NULL);
if (disasm == NULL) {
LOG(WARNING) << "Could not create LLVM disassembler for target triple " << triple;
return;
}
使用 LLVM C API（而非 C++ API），因为更简单、稳定；
triple 描述当前进程的 CPU 架构和 ABI；
LLVMCreateDisasm 创建一个针对该架构的反汇编上下文；
失败则仅警告并退出（不影响主流程）。

📌 第三步：初始化指针和输出函数头

cpp
char line_buf[2048];
uint8_t code = reinterpret_cast<uint8_t>(addr);
uint8_t code_end = code + size;

asm_file << "Disassembly for " << fn_symbol << " (" << reinterpret_cast<void>(addr)
<< "):" << "\n";
code 指向函数机器码起始位置；
line_buf 用于存储单条反汇编结果（如 "mov rax, rdi"）；
先写入函数标识头，便于阅读。

📌 第四步：逐条反汇编指令 + 插入调试信息

cpp
while (code < code_end) {
uint64_t inst_addr = reinterpret_cast<uint64_t>(code);

// ▼▼▼ 插入所有 ≤ 当前地址的调试信息 ▼▼▼
for (; di_line_it != di_lines.end() && di_line_it->first <= inst_addr; ++di_line_it) {
llvm::DILineInfo line = di_line_it->second;
asm_file << "\t" << line.FileName << ":" << line.FileName << ":"
<< line.FunctionName << ":" << line.Line << ":" << line.Column << "\n";
}

// ▼▼▼ 反汇编当前指令 ▼▼▼
size_t inst_len = LLVMDisasmInstruction(disasm, code, code_end - code, 0, line_buf,
sizeof(line_buf));
if (inst_len == 0) {
LOG(WARNING) << "Invalid instruction at " << static_cast<void>(code);
break;
}

uint64_t offset = inst_addr - addr;
asm_file << offset << ":\t" << line_buf << "\n";
code += inst_len;
}
关键逻辑：

1. 按地址顺序交错输出调试信息和汇编指令：
在每条指令前，先输出所有 地址 ≤ 当前指令地址 的源码位置；
这样就能看到“哪段汇编对应哪行源码”。

2. 反汇编单条指令：
LLVMDisasmInstruction 将 code 处的机器码转为字符串（存入 line_buf）；
返回指令长度（字节数），用于推进指针；
若返回 0，说明非法指令（可能因 JIT 优化或边界错误），则中断。

3. 输出格式：

<offset>:\t<assembly mnemonic>

例如：

0: mov rax, rdi
3: add rax, 1
⚠️ 注意：line.FileName 被写了两次（可能是笔误，应为 FileName:FunctionName:Line:Column），但不影响功能。

📌 第五步：输出剩余调试信息（收尾）

cpp
for (; di_line_it != di_lines.end(); ++di_line_it) {
llvm::DILineInfo line = di_line_it->second;
asm_file << "\t" << line.FileName << ":" << line.FileName << ":"
<< line.FunctionName << ":" << line.Line << ":" << line.Column << "\n";
}
asm_file << "\n";
理论上所有调试信息应在指令循环中输出完；
此处是防御性处理，确保没有遗漏；
最后加一个空行分隔不同函数。

🧠 总结：这个函数做了什么？

功能 技术实现 价值
------ -------- ------
反汇编 JIT 代码 使用 LLVM C 反汇编器逐条解析机器码 查看实际执行的汇编
关联源码位置 通过 DWARF 调试信息映射地址 → 文件/行号 理解“哪段 IR/C++ 生成了哪些汇编”
结构化输出 先调试信息，再汇编指令，按地址排序 便于人工或工具分析
容错处理 非法指令仅警告，不崩溃 保证系统稳定性

💡 示例输出（简化版）

asm
Disassembly for AggCompute:frag_abc (0x7f12a0001000):
/home/user/impala/exprs/agg.cc:/home/user/impala/exprs/agg.cc:AggCompute:42:5
0: mov rax, rdi
3: add rax, 8
6: ret

/home/user/impala/exprs/agg.cc:/home/user/impala/exprs/agg.cc:AggCompute:43:1
开发者一眼看出：第 42 行 C++ 代码生成了 mov + add + ret。

🎯 为什么这对 Impala 至关重要？
Impala 的表达式、聚合、连接等算子都是 运行时 JIT 生成 的；
生成的代码性能直接影响查询速度；
通过此函数，开发者可以：
验证 LLVM 优化是否生效；
发现低效指令序列（如冗余 load/store）；
结合 perf 热点定位具体 IR 或 C++ 源码问题。
这是 JIT 系统可观测性（Observability）的关键组成部分。
****************************************************************88888
void CodegenSymbolEmitter::WritePerfMapLocked() 
是的，在 类 Unix 系统（如 Linux） 上，rename() 系统调用对 同一文件系统内的文件 是 原子操作（atomic）。

✅ 具体到你的代码：

cpp
if (rename(tmp_perf_map_path.c_str(), perf_map_path.c_str()) != 0) {
// handle error
}

这段代码的意图是：

1. 先将新的 perf-<pid>.map 内容写入一个临时文件（如 /tmp/perf-1234.map.tmp）；
2. 然后通过 rename() 将临时文件 原子地替换 原始文件（/tmp/perf-1234.map）。

这正是 “原子更新文件” 的经典模式。

🔍 为什么 rename() 是原子的？

根据 POSIX 标准（以及 Linux 实现）：
If the rename() succeeds, the old file is removed and the new file appears in its place, and this operation is atomic: any process attempting to open the target path will see either the old file or the new file — never a partially updated or corrupted file.

关键点：
同一文件系统内：rename() 只在源和目标位于同一挂载点（文件系统）时才是原子的。
→ 你的代码中 /tmp/... 和 /tmp/....tmp 显然在同一目录，满足条件。
覆盖行为：如果 perf_map_path 已存在，rename() 会 直接覆盖它（unlink + link），且整个过程不可分割。
崩溃安全：即使系统在 rename() 执行中途断电，也不会出现“半写文件”——要么旧文件还在，要么新文件完整生效。

🛡️ 这种做法的好处

问题 解决方案
------ --------
直接写 perf-<pid>.map 时被 perf 读取 → 读到不完整内容 先写临时文件，再原子替换
多个线程/进程同时更新 → 文件内容混合 rename() 保证读者看到一致状态
写入过程中程序崩溃 → 留下损坏文件 临时文件可安全丢弃，原文件不受影响
💡 正因如此，perf 工具能可靠地读取 /tmp/perf-*.map 而不会解析失败。

⚠️ 注意事项

1. 跨文件系统 rename() 不是原子的（会返回 EXDEV 错误）；
但 /tmp 内部操作没问题。
2. Windows 上 rename() 行为不同：
在 Windows 中，如果目标文件已存在，rename() 通常会失败（不像 POSIX 那样覆盖）；
但 Impala 主要运行在 Linux，所以无需担心。
3. 临时文件应与目标文件在同一目录：
你的代码正是这么做的（.tmp 后缀），✅ 正确。

✅ 结论
是的，在 Linux/Unix 系统上，你代码中的 rename() 调用是原子的，并且这是一种 广泛采用、安全可靠的文件更新模式，特别适用于像 perf.map 这样需要被外部工具实时读取的场景。

这种设计体现了良好的系统编程实践 👍。