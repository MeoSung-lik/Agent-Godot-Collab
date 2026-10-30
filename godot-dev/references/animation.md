# 动画系统（AnimationPlayer / AnimationTree / tween 演出）

## Animation 资源

- 所有 Animation 资源统一 `step = 0.02`（50fps）。**禁用 30/45/60 FPS**：Godot 编辑器动画面板存在浮点换算 bug——当 1/FPS 是无限循环小数（30/45/60）时，编辑器内部位置计算回绕、拖轴时顶部时间显示错乱。用 25/40/50/100。`step` 写入 .tscn 可跨机器固化——新建/复制的动画必须检查此属性。
- String/枚举类轨道（字幕ID/指令/状态标记）用 **Discrete（离散）** Update 模式：Continuous 会对 String 轨道触发引擎实验性字符串混合算法（警告 "blends String types"）。数值/颜色轨道（modulate 等）保持 Continuous。
- 配套：轨道写入目标属性的 setter 加幂等守卫（`if value == 属性: return`），防止重复赋值触发 setter 副作用。
- 动画控制优先 `AnimationPlayer`；动画名通过 `@export` 暴露便于换素材。

## AnimationTree

- **tree_root 是共享子资源**：场景子资源被所有实例共享——任何实例写 loco 动画全体生效（症状：「移动射击时脚不动」，站立者把走路者的动画覆写回 idle）。修法：setup 时 `tree_root = tree_root.duplicate(true)` 每实例独享。

## 演出与 tween

- headless 下 delta 不真实（1 帧可吞掉整个 0.25s tween 或 0.35s 闪光窗口）——动效测试不能 await 帧后采样表现属性，须**白盒同帧断内部状态**（直调设置函数后立采内部系数）。
- 隐藏分支容器不排版：CanvasLayer 整层隐藏期间 Control 布局不重排——弹层亮出后容器孩子居中排序停留在旧状态（面板贴左上角）。动效先把容器显示后等 1~2 帧再记录休息态；或改用锚点居中（anchors_preset 8）即时生效、不依赖容器排序。
- 单次动效（`set_process(false)` 播完即停）挂在可复用宿主上时，宿主隐藏后要**重新武装**，否则复用场景只播一次。
- `tween_method` 驱动 `set_shader_parameter` 实测不生效——改「shader 内乘 COLOR.a + 代码 tween 节点 modulate」可靠。
- 演出期间必须锁退出输入：逐格结算演出中点离开 → 协程在 await 处早退 → 挂起/恢复逻辑无人解除 → 之后所有同类状态更新被永久吞（症状：金币 HUD 不再变化）。
- 动效代码不覆写编辑器设定值：写死回落 `Vector2.ONE` 会打回用户在预制件里调的 scale=2——`_ready` 捕获编辑器值作 `_基准缩放`，动效围绕基准进行。

## 截图时机

- 截图比属性变更慢一帧：改完属性立刻截图拿到旧画面——隔一帧再拍。
- tween 弹入等瞬态演出截图窗口 0.05~0.3s。
- SubViewport 摆相机后：`look_at()` → `await process_frame` → `await frame_post_draw` → `get_image()`。只等 frame_post_draw 偶发旧相机矩阵坏帧；实拍截图只等一帧拿到旧帧（误判"不渲染"白查四轮）——等 ≥0.5 秒。
