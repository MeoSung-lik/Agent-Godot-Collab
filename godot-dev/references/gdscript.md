# GDScript 语言坑与脚本规约

写或改 GDScript 前通读。本文件条目多数是**静默失败**：不报错，只是悄悄不生效、断言消失、或探针挂死。

## 类型与推断

- `:=` 不从动态值推断类型。以下都会 Parse Error 或运行时才炸：untyped/Node 字段取值赋给 `:=`；`get_node()` 返回值上调方法后推断；动态方法返回值推断（`var x := 单例.方法()` 报 "Cannot infer type"）。后者编辑器里看不出、`--script` 运行时才炸，症状是探针窗口**白屏静默挂着不退出**。一律显式 `: Type`。
- 外部赋值类型化 `Array[String]`：`_数组 = []` 报 Invalid assignment——用 `.clear()`。
- `dict.get(键)` 缺键返回 nil，直接赋给 typed var 报错——先判 nil 或用带默认值形式。
- typed 参数拒收内联数组字面量：`foo([1,2])` 报 Invalid type 且**中断当前函数但调用方继续跑**——在测试里表现为断言凭空消失、假 ALL PASS。先赋给 typed 变量再传；测试输出要 grep 具体用例名确认真跑过。

## 运算符与缺失 API

- `as` 优先级低于 `==`：`x == [..] as Array[int]` 解析报错；断言里 `as` 还会把比较参数变成数组。先建期望变量再比较。`as` 转型必须排在 `is_instance_valid` 之后，否则 freed 对象 cast 报错。
- 没有 `bool()` 构造（报 `Nonexistent 'bool' constructor`），Variant 转布尔用 `== true`。
- String 没有 `.to_string()`，用 `str()`。
- `%` 格式化不支持 `%g`（原样输出不报错），用 `%d` + roundi；裸 `%` 撞车写 `%%`。
- `Image` 无静态 `load_from_buffer`——是实例方法 `load_png_from_buffer()` / `load_jpg_from_buffer()`，返回 Error 非 null；`Image` 用 `get_size()` 无 size 属性；`Rect2i.size` 是属性别加括号。
- 方法名是 `ResourceUID.id_to_text()`，没有 uid_to_text。
- typed `Array[T].erase()` 返回 void——判"存在才删"用 `has()` + `erase()`。
- GDScript 不支持逗号表达式；`Array` 无 `insert_range`（倒序逐条 insert）。

## lambda 与 Callable

- lambda 对局部变量**按值捕获**：信号 handler 里给局部变量赋值改的是捕获副本，外层永远读不到。被信号更新的状态放成员变量；计数器走字典引用（dict 捕获的是引用）。
- `Callable.bind` 的实参**追加在发射实参之后**：`信号.emit(a)` + `connect(handler.bind(b))` → handler 收到 `(a, b)`。`gui_input.connect(_on.bind(下标))` + `func _on(下标, event)` 是错的——下标收到 event，类型不匹配报错发生在调用层但**静默失效**。统一用 lambda 闭包：`connect(func(event): _on(数据, event))`。

## 协程

- `_ready` 里调含 await 的协程不 await = **静默遗弃**：首个 await 挂起后主流程继续跑完，协程后半段全部不执行且无任何报错。表现为断言凭空消失、失败 0 项假全过。
- 测试函数不 await 协程尾句，尾部语句与下一个测试竞态假通过。
- 无法 await 的地方（信号回调）要重新设计流程，不能"顺手加 await"了事。

## 节流 / 预算 / 计时

- 节流计时器**必须每帧递减**——漏了递减行，计时器停在初值，功能永久失效。
- 被节流的代码块里，时间累计不能用帧 delta（节流 0.15s 求值一次、按帧 delta 累计会实际拉长 ~9 倍）——用「距上次求值的真实间隔」。
- 预算/配额类机制：被拒方必须**随机退避**（负计时=欠账时长），否则固定裁决序 + 逐帧重试会把树序优先级固化成稳态饥饿。
- `Performance` 枚举名：`TIME_PHYSICS_PROCESS`（没有 TIME_PHYSICS）、`OBJECT_ORPHAN_NODE_COUNT`、`RENDER_TOTAL_*`、`RENDER_VIDEO_MEM_USED`。写错 = 脚本解析失败 = 场景无脚本干坐退出，无报错指向。

## 脚本挂载与节点路径

- 脚本挂在场景内部某个「根」节点（非场景根）时，`@export NodePath`、`$`、`@onready` 全部**从「根」起算**——按场景根推路径必错一层。
- autoload 在树中的名字是**注册键**（project.godot 里写的名字），不是脚本/场景文件名——按 /root/场景名/ 找节点会扑空。
- 内部类声明会把同缩进后续函数吞进类体——内部类放文件末尾。
- 内部类 Control 必须 `set_anchors_and_offsets_preset(FULL_RECT)`——只 `set_anchors_preset` 撑不开（size 恒 0）。
- `is_visible_in_tree()` 是方法不是属性——写成属性访问静默返回 null，整段判定失效。
- 鸭子代理类的方法签名必须与本尊对齐**全参**——少抄一个参数，白盒探针全绿、真机必炸。

## 语义设计

- 扣减类 API 语义写明：「有多少扣多少」（返回实扣）还是「必须足额」（先查余额再扣）——付门票这类足额场景直接扣会半扣还当成功。
- shader 语言不支持中文标识符：uniform 一律英文。shader 编译失败时 ColorRect 按自身 color（默认白）渲染 → 全屏白——ShaderMaterial 的 ColorRect 务必把 color 设成透明兜底。
- 恢复态不要无条件把属性设回编辑器值：恢复只把缓存置占位值失效，交还驱动器下帧重刷。
- 重试节流状态机要处理「设置中途关掉则放弃」「目标更换重新计时」。
- `remove_theme_color_override` 会连场景覆盖一起剥——剥主题覆盖前想清楚层级。
