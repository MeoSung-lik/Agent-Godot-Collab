# 无头测试 / 探针 / 性能测量

跑 `--headless`、`--script` 或任何自动化测试前通读。这里的坑共性：**挂死、静默回退主场景、假通过**。

## 启动方式与命令行

- **测试场景文件名一律 ASCII**。中文场景名 Git Bash 直传、`cmd //c` 都会**静默回退主场景永挂**（无任何报错）；python subprocess（CreateProcess 原生 UTF-16 argv）可正常直传中文 res:// 路径。
- `.gd` 不能直接当场景跑（同效回退主场景）——必须包一层 .tscn 探针场景。
- `--path=X` 等号形式引擎**静默不解析** → 进无头编辑器模式卡死。必须空格形式 `--path X`，且必须**绝对路径**——cd 漂移后 `--path .` 静默开进项目管理器（进程挂着、日志只有启动 banner）。
- 自定义参数分界：独立 `++` token（`xx.tscn ++ 参数 值`）；`++key=val` 连写不进 user args。多个 `++` 时第二个 ++ 会留在 args 里——按位置取 args[1] 必失效，用 `in` 包含判断。
- `--script` 模式**不注册 Autoload**：场景脚本直接引用 autoload 标识符报 `Identifier not found: Xxx`——**不代表工程有问题**（假警报）。加载验证走正常场景启动（ASCII 名场景 + _ready 里 load+instantiate+断言+quit）。
- `--check-only --script` 同理误报。新 class_name 在 --script 编译期不识别 → 测试脚本用 `const Xxx = preload(...)` 绑定、鸭子类型（`var x: Node` + 动态调用），不写 script-class 类型标注（会拉入依赖链在编译期炸）。
- `-e -s` 跑不了 EditorScript。
- **测试/探针必须会退出**：Runner 忘写 `func _ready(): _run()` 入口或漏 quit → 进程永挂。加载中途 quit 在拆卸期刷 Parse Error 噪音勿当真。
- `--script` 模式下有真实 Autoload 在场——手动 new 同名单例会被改名成影子节点，应 `root.get_node("注册名")` 操作真单例。

## headless 的渲染与帧限制

- `--headless` 是 dummy 渲染器：`get_texture().get_image()` 返回 null——截图/渲染图标类探针必须开窗跑。
- `get_camera_3d()` 仍返回有效相机，但 `frame_post_draw` 永不触发，await 挂死——加 `DisplayServer.get_name() != "headless"` 守卫。
- delta 不真实：1 帧可吞整个 tween/闪光窗口——动效断言走白盒内部状态，不采样表现属性。
- `Camera3D.is_position_in_frustum` 冻在相机入场朝向，旋转后全 false——涉视锥的无头测试只能断言几何换位。
- headless 帧饥饿两面：idle 帧比物理帧稀疏（等调度器类 idle 逻辑用 `await process_frame`）；但另一些场景 `await process_frame` 永不 resolve——遇挂死先分辨等的是哪种帧。
- Terrain3D 在 --script 与 headless 场景模式都**不构建碰撞**（数据在、碰撞无，射线全扑空）——探针须开窗场景模式。
- headless 物理以 60Hz 恒定跑，健康 ≈16.7ms/tick——这是墙钟法的基线。

## 性能测量

- headless 下 Performance 监视器不可信：TIME_PROCESS/TIME_PHYSICS_PROCESS 随节点数超线性膨胀（裸 Node3D×3900 报 43ms，真机 8 万节点 0.07ms）；`_process 0.04ms` 不含全部节点脚本成本。**物理结论用墙钟法**（超载实打实变长）；定位渲染 vs 脚本用探针分相实测。
- `Performance.TIME_FPS` 在 shader 编译瞬态期读到假低帧（process_frame 计数同秒 180）——两把尺并用。
- headless 的枚举坑：`--script` 模式无 `MEMORY_DYNAMIC` 只有 STATIC。

## 输出与日志

- 管道 stdout 全缓冲：被 kill 时缓冲丢失 = 零输出假象；`| tail` 吞实时日志且 exit code 是 tail 的。活日志用 `--log-file` 或文件直写 flush；bash `>` 重定向看不到日志要用 PowerShell `Start-Process -RedirectStandardOutput`。
- **运行期报错完整堆栈在 `%APPDATA%/Godot/app_userdata/<游戏名>/logs/godot.log`**——grep 报错文本拿 backtrace 直达文件:行号。**先翻日志再读代码**。
- 引擎 ERROR 与 SCRIPT ERROR 是同次调用的两级输出，日志里都要 grep。
- 多 Godot 实例互斥：编辑器开着绝不再起第二个实例（headless 测试也是）——抢 .godot/ 出 file_access 报错、锁死 godot.log、窗口僵尸化。
- 资源验证必须新进程：编辑器进程内存缓存旧导入资源，会骗人。

## 测试写法纪律

- 期望值从配置表推导（CSV 读值再算），不写死——策划调参不破测试。哨兵参数断言检查解析后的生效值，不查 @export 原始哨兵值。
- **测试污染真实存档**：--script 模式 autoload 在场，测试会写真档 user://save.cfg，前序测试留档使后跑的「新档断言」跨测试失败。隔离手法：先改道 `真Autoload.存档路径 = "user://xxx_tmp.cfg"` 再触发落盘；跑前备份/跑后还原；跑完核对 mtime。
- 白盒手法：`脚本.new()` 不入树直调私有方法（_ready 不跑、不依赖场景）；大脑类组件白盒必须挂到宿主下（靠 get_parent() 取宿主的方法挂 Runner 下静默失效）；注入状态字段必须三件套齐全（漏一项首帧即放弃）；Timer.time_left 只读不能白盒写，倒计时步进走内部走秒函数；测试各节清场防计数污染。
- **测试地板要写进 .tscn 静态摆**：代码建 StaticBody 漏 `add_child(形状)` → 无碰撞全员平行自由落体假通过。诊断手法：排除自身 intersect_ray + 打印 y 值。
- 跑测要 grep `SCRIPT ERROR`，不能只看 ALL PASS——GDScript 非致命错误继续跑，掩盖假对局（实测一次 24 条隐藏 SCRIPT ERROR）。
- --script 模式脚本挂载失败变纯 Node，断言全空转假通过——跑前先验证脚本真的挂上了。
- 设计探针先确认前提真的成立：假容器带变换导致「注册在原点」不成立；摆位与注册同格导致「重挂与否」键一样——探针测不出差异就是白测。
- `Engine.time_scale` 加速端到端时，改调度参数要同步检查测试窗口（断言窗口恒假 = 提交时即坏）。
- 端到端几何断言先在纸面推演路径与窗口（跑毒斜线绕远超预警窗的教训）。

## 摆位与输入注入

- dummy 在原点出生第一帧就以贴身感知捕获目标并种记忆——进树**前**设 position；背向用 `rotation.y = atan2(-to.x, -to.z)`（look_at 是 -Z，前方 +Z 的控制器会被当反）。
- AI 排直线面朝彼此时互在视野锥内互不可派——环形背向摆位或间距 > 感知范围。
- 感知冷却窗内立刻断言读到陈旧状态——等 ≥0.3s 再断言。
- 探针注入动作给 process 轮询方：必须 `await get_tree().process_frame`（帧首信号恢复）后再 `Input.action_press`——从 SceneTreeTimer 超时恢复（帧行尾）按下的动作，process 帧号已过期，`is_action_just_pressed` 永远看不到；press 下一帧帧首再 release。
- physics_frame 信号在帧初、_physics_process 在帧中——状态断言要再让一拍。
- 探针实例化地图后通常要 `get_tree().current_scene = 地图`（否则扫 current_scene 子树的逻辑扫空，如按 current_scene 算弹道）；但别在会「激活对局状态机」的语义下改指地图——会激活计时/播报/相机抢夺。原则：看被测代码扫什么、激活什么。
- `_ready` 里 `add_child` 到 root 报 `Parent node is busy setting up children` **静默失败**（节点不在树、@onready 全 Nil、错误连环）——call_deferred + 入树校验。

## freed-instance 与对象池

- `is_instance_valid` + `is_queued_for_deletion` 挡不住「活着但不在树」的池化对象——快照容器（网格/缓存数组）过滤条件必须含 `is_inside_tree()`。
- 缓存入口守卫挡不住所有时序（拾取与使用之间释放）——**返回侧兜底**：返回前再校验，失效本帧按无目标处理。
- freed 实例上调方法报错会静默吞掉协程后续断言（假全过）——先 is_instance_valid；注意它对字符串恒 false，别拿它断言文案。
- freed 的 pending SceneTreeTimer 被引擎静默跳过（日志 0 提示），只有「出树未释放」的池化态会炸。
- 池化对象的定时器**世代守卫必须在出树时也推进**，不能只在复用重置时——入池等待期空窗会串世 + 重复入池 + 账目污染。

## 归因方法

- 确认工作区 diff 全是本次改动后 `git stash push -- <文件>` 跑基线再 pop（不带 -u，未跟踪新文件原位不动）；存量失败用旧版 checkout 对照归因。
- 「隐形碰撞」类问题：半径剖面探针（逐米 × 16 向射线打印命中高度 + 碰撞体父名）五分钟定位。
- 「只在地图里出现的掉帧」要用带完整 HUD/真玩家的地图探针——空场最小复现因缺依赖不复现；探针假玩家缺宿主方法会刷屏 SCRIPT ERROR 虚标耗时（测扫描耗时必须真宿主）。
- 归因四板斧：全树 _process 普查 + 实锤开关 / 逐层二分 / 对照实验（A/B 摘除）/ stash 基线。
- 大场景资源加载：.tscn 例外不能线程加载（文本场景线程解析在快速退出时撕裂报 parse error）；同步 load 大地图 10s+ 会饿死网络——`load_threaded_request` 轮询。
