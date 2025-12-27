这个函数 SubtractLarge 是 Apache Impala 中用于安全地执行两个大十进制数相减的底层工具函数。它处理的是 “不能直接用普通整数运算相减” 的高精度数值（例如 DECIMAL(38,10) 类型），并且有一个关键前提：
两个操作数符号相反（一正一负），且都不为零。

这实际上意味着：
SubtractLarge(x, y) 本质上是在计算 x - y，但其中 x 和 y 符号相反 → 等价于两个同号数相加！

例如：
若 x = +123.45，y = -67.89，则 x - y = 123.45 - (-67.89) = 191.34
若 x = -123.45，y = +67.89，则 x - y = -123.45 - 67.89 = -191.34

所以，这个“减法”函数实际是通过“加法”实现的——因为减去一个负数等于加上正数。

下面逐段详细解释其逻辑：

✅ 前提条件（由 DCHECK 保证）

cpp
DCHECK(x != 0 && y != 0);
DCHECK((x > 0) != (y > 0));
两数均非零。
一正一负 → 它们的“减法”等价于绝对值相加，结果符号由绝对值大的一方决定（但这里通过统一处理避免显式判断）。

🔍 第一步：分离整数与小数部分

cpp
SeparateFractional(x, x_scale, y, y_scale, &x_left, &x_right, &y_left, &y_right);
将 x 拆为 x_left（整数部分）、x_right（小数部分）
同理拆 y
注意：x 和 y 可能为负！所以 x_left, x_right 等也可能为负
💡 SeparateFractional 应该会保留原始符号。例如：
若 x = -12345，x_scale = 2 → x_left = -123, x_right = -45
若 y = +6789，y_scale = 2 → y_left = +67, y_right = +89

📏 第二步：对齐 scale

cpp
int max_scale = std::max(x_scale, y_scale);
int result_scale_decrease = max_scale - result_scale;
DCHECK_GE(result_scale_decrease, 0);
所有小数部分以 max_scale 为基准（SeparateFractional 内部应已完成对齐）
最终要缩放到 result_scale，所以需要除以 10^(max_scale - result_scale)

➕ 第三步：直接相加整数和小数部分（关键！）

cpp
left = x_left + y_left;
right = x_right + y_right;

由于 x 和 y 符号相反，但我们在做 x - y，而 y 本身带符号，所以：
实际上 x - y = x + (-y)
而 -y 与 x 同号
因此 x_left + y_left 和 x_right + y_right 都是同号相加（或一正一负但整体代表加法）
✅ 注释说：“Overflow is not possible because one number is positive and the other one is negative.”
这句话容易误解！其实意思是：在“减法转加法”的语境下，不会出现两个大正数相加溢出的情况？
但更准确的理解是：这里的 left 和 right 是中间表示，其绝对值不会超过合法范围（由后续 DCHECK 保证）

⚖️ 第四步：统一整数与小数部分的符号（借位/进位调整）

这是本函数最精妙的部分！

cpp
if (left < 0 && right > 0) {
left += 1;
right -= DecimalUtil::GetScaleMultiplier<int128_t>(max_scale);
} else if (left > 0 && right < 0) {
left -= 1;
right += DecimalUtil::GetScaleMultiplier<int128_t>(max_scale);
}
🤔 为什么需要这个？

考虑一个例子：
x = -1.1（即 x_left = -1, x_right = -10，scale=2）
y = +0.9（即 y_left = 0, y_right = +90）
我们要算 x - y = -1.1 - 0.9 = -2.0

调用 SeparateFractional 后：
x_left = -1, x_right = -10
y_left = 0, y_right = +90
直接相加：left = -1 + 0 = -1, right = -10 + 90 = +80

现在 left = -1（负），right = +80（正）→ 符号不一致！

但这在十进制表示中是非法的！
正确的表示应该是：-2.0 → left = -2, right = 0
🔧 如何修正？
当 left < 0 且 right > 0：说明小数部分“多出来了”，应该从整数部分“借 1”
left += 1 → -1 + 1 = 0 ❌ 不对！
实际上：-1 + 0.8 = -(1 - 0.8) = -0.2？不对，我们想要 -2.0

等等，上面例子可能不恰当。再换一个：

✅ 更好的例子：
计算 -0.1 - 0.9 = -1.0
x = -0.1 → x_left = 0, x_right = -10（scale=2）
y = +0.9 → y_left = 0, y_right = +90
left = 0 + 0 = 0
right = -10 + 90 = +80 → 表示 +0.80，但真实结果是 -1.0！

问题在于：x_right = -10 表示的是 -0.10，但当我们把它和 +90 相加，得到 +80，这掩盖了整体为负的事实。

但实际上，在 SeparateFractional 中，负数的拆分应该是：
-0.1（scale=2）应表示为：left = -1, right = +90
因为：-1 100 + 90 = -100 + 90 = -10 → -0.10

啊！这才是关键。
📌 负数的小数部分在内部表示中通常被“补码化”为正数，通过减少整数部分来补偿。

所以，如果 SeparateFractional 正确实现了这一点，那么 x_left 和 x_right 应该满足：
x = x_left 10^scale + x_right
且 0 <= x_right < 10^scale（即使 x 为负！）

但你的代码中 没有保证 x_right >= 0，反而有 DCHECK(abs(right) <= ...)，说明 right 可正可负。

因此，函数假设 SeparateFractional 返回的 x_right 和 y_right 保留原始符号。

于是，当 left 和 right 符号不一致时，我们需要规范化：
left < 0 && right > 0：整体为负，但小数部分写成正 → 应该把 right 变负，向 left 借 1
left += 1（比如从 -2 变成 -1）
right -= 10^max_scale（比如 +30 → +30 - 100 = -70）
结果：-1 100 + (-70) = -170 → -1.70，但原本可能是 -2.30？似乎方向反了。

等等，再仔细看代码逻辑：

cpp
if (left < 0 && right > 0) {
left += 1; // 整数部分增加（向零靠近）
right -= scale_mult; // 小数部分减去 10^s（变负）
}

例如：
left = -2, right = +30（scale=2）
调整后：left = -1, right = +30 - 100 = -70
数值：-1 100 + (-70) = -170 → -1.70
原始未调整：-2 100 + 30 = -170 → 数值不变！

✅ 所以这是一个恒等变换，目的是让 left 和 right 符号一致，便于后续组合。

同理：
left = +1, right = -30 → 调整为 left = 0, right = +70
+1100 + (-30) = 70；0100 + 70 = 70 → 相等
🎯 目的：确保最终 left 和 right 同号（或其中一个为零），这样 left 10^s + right 才是规范的十进制表示。*

📉 第五步：缩放小数部分到目标 scale

cpp
if (result_scale_decrease > 0) {
right = DecimalUtil::ScaleDownAndRound<int128_t>(
right, result_scale_decrease, round);
}
除以 10^(max_scale - result_scale)
可选四舍五入
注释提到：即使四舍五入导致 right == 10^result_scale，也不特殊处理，因为这会自然导致 left += sign(right)，在最终加法中体现

✅ 第六步：溢出检查与结果组合

cpp
DCHECK(left == 0 right == 0 (left > 0) == (right > 0));

int128_t mult = DecimalUtil::GetScaleMultiplier<int128_t>(result_scale);
if (UNLIKELY(abs(left) > (MAX_UNSCALED_DECIMAL16 - abs(right)) / mult)) {
overflow = true;
}
return DecimalUtil::SafeMultiply(left, mult, overflow) + right;
先确认 left 和 right 同号（规范化成功）
检查 left 10^s + right <= MAX_UNSCALED_DECIMAL16
等价于 left <= (MAX - right ) / mult
如果不满足，设 overflow = true
最终结果：left 10^result_scale + right
⚠️ 注意：这里没有使用 ArithmeticUtil::AsUnsigned，因为 left 和 right 已经是规范化的带符号值，且乘法由 SafeMultiply 处理溢出。

✅ 总结：SubtractLarge 做了什么？
安全地计算两个符号相反的大十进制数的差（即等价于绝对值相加），并返回指定 scale 的结果，同时检测溢出。
关键步骤：
1. 利用“减负数等于加正数”的性质，将减法转化为加法
2. 分离整数/小数部分（保留符号）
3. 直接相加整数和小数部分
4. 规范化：确保整数和小数部分符号一致（通过 ±1 和 ±10^scale 调整）
5. 缩放小数部分到目标精度（可选四舍五入）
6. 检查最终结果是否超出 DECIMAL128 范围
7. 组合结果：left 10^result_scale + right
为什么复杂？
因为要避免有符号整数溢出（UB）
要精确控制 scale（小数位数）
要支持四舍五入
要在 128 位范围内模拟任意精度十进制运算**

这正是数据库系统中高精度 DECIMAL 运算的典型实现方式。
**********************************************************888