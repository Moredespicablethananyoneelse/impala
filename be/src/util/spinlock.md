
4. 现代替代

gutil 已过时，Google 推荐 Abseil 的 absl::Mutex（支持 SpinLock 模式）和 absl::SpinLock（类似但更现代）。
如果新项目，用 Abseil；Impala 仍用 gutil（兼容性）。