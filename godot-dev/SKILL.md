---
name: godot-dev
description: Godot 4.x 踩坑与调试经验库（调试技能，与 godot-collab 协作流程技能配套使用）：GDScript 语法坑、手写/生成 .tscn/.tres/CSV 坑、无头测试与探针、动画、Jolt 物理、Terrain3D 地面、渲染性能、Windows/Android/PCK 导出打包中大量与直觉相反的静默失败。凡在 Godot 项目里排查"改了没生效/静默失败/测试假通过/沉地穿地/手机端哑火"、写脚本/改场景/跑测试/打包前的坑位自查、或做性能优化，都应使用本技能——规则来自真实项目近 200 条踩坑沉淀，不查会白干。
---

# Godot 4.x 实战开发经验

规则来自一个完整上线的 Godot 4.7 3D 项目（GL Compatibility 渲染 + Jolt 物理 + Terrain3D 起伏地形）高强度开发中近 200 次踩坑的沉淀。这些坑的共性：**静默失败**——不报错、不崩溃，只是悄悄不生效、挂死、或测试假通过。

本技能定位是**坑库/调试技能**：人机协作流程、分工边界与确认制见成对的 `godot-collab` 技能（两者同级安装，分享时一起打包）。

标注了 Jolt / Terrain3D / GL Compatibility 的条目按项目实际引擎配置取用，其余为 Godot 4.x 通用。

## 按需深入（做对应任务前先读对应 reference）

| 任务 | 先读 |
|---|---|
| 写 / 改 GDScript | [references/gdscript.md](references/gdscript.md) |
| 手写或脚本生成 .tscn / .tres / CSV 配置表 | [references/scenes-tscn.md](references/scenes-tscn.md) |
| 动画 / AnimationTree / tween 演出 | [references/animation.md](references/animation.md) |
| 无头测试、探针、性能测量 | [references/headless-testing.md](references/headless-testing.md) |
| 物理、碰撞、Terrain3D、地面贴地逻辑 | [references/physics-terrain.md](references/physics-terrain.md) |
| 渲染调优、手机帧率、截图、移动端适配 | [references/rendering-performance.md](references/rendering-performance.md) |
| 导出打包（Windows / Android / PCK） | [references/export-packaging.md](references/export-packaging.md) |
| 导入缓存、uid、编辑器行为、git 协作 | [references/editor-project.md](references/editor-project.md) |

## 通用纪律（任何任务都适用）

**命名语言分工**——类/函数/变量名用英文常规风格；节点名、`@export` 参数、调试文本可用中文。但以下位置**必须 ASCII**，中文会被静默拒绝、静默忽略或报错不指名文件：Autoload 注册名、tscn 的 sub_resource/ext_resource id、metadata 键、主题 type variation 名、shader uniform、测试场景文件名（Windows CLI 传参）、导出包路径与文件名。

**强类型**：变量、参数、返回值显式标注类型。`:=` 禁止从动态值推断（untyped 字段、get_node 返回值、动态方法返回值）——轻则 Parse Error，重则探针窗口白屏挂死。

**资源引用只用 `res://`**：只引用工程内已导入的资源，不硬编码磁盘绝对路径。新建资源（.tscn/.tres/贴图/csv）后先跑 `godot --headless --import` 确保 .import/.uid 就绪，再被其他资源引用。

**数值唯一来源是配置数据**（CSV/Resource），脚本不硬编码常量；CSV 手改 `.import` 为 `importer="keep"`（默认会导成翻译文件，导出后读不到源表）。

**全局/跨场景状态用 Autoload**（注册名必须 ASCII），不用 static 字段；需要常驻的对象挂 root 或 Autoload。

**查找节点用 group**（`add_to_group` + `get_first_node_in_group`），不按名递归遍历。group 是纯字符串无需声明，但 tscn 里必须写在节点头部属性 `groups=["组名"]`——写成节点下方的独立属性行不会被解析。

**物理体不作预制件根节点**：根用 Node3D，物理体作子节点且自身缩放恒为 1（非 1 缩放的物理体在 Jolt 下全链不可靠，编辑器还会警告）。射线/查询返回的 collider 是子物理体，取 `collider.get_parent()`。

**禁平地假设**：凡"离地高度"以脚下地面为基准——物理帧向下射线或地形高度查询取地面再加相对偏移，无命中时明确不钳制。角色在坑里、坡上、楼顶时逻辑必须仍成立；固定世界高度（`y < 常量` 钳制、`y = 常量` 生成基准、缓存出生点地面高度当长期常量）都是雷。

**存在性检查一律 `ResourceLoader.exists()`**：`FileAccess.file_exists()` 判 res:// 导入资源在导出包里恒 false（包内只有 .import 边车没有源文件）——整块功能静默跳过，PC 编辑器永不复现、手机端才炸。

**动效不覆写编辑器设定值**：动效代码把属性写死（如回落 `Vector2.ONE`）会打回用户在编辑器里调的值——`_ready` 捕获编辑器值作基准，动效围绕基准进行。

**成对信号成对连接**（进入/离开、开/关）：补另一半前先查连接是否已存在；恢复态不要无条件把属性设回，置缓存失效交还驱动器下帧重刷。

**演出期间锁退出输入**：协程在 await 处被早退后，挂起/恢复逻辑无人解除，之后所有同类状态更新被永久吞掉（症状：HUD 不再刷新、状态卡死）。

**导入缓存治理**：`Identifier "Xxx" not declared`（改过 class_name/重命名后）→ 只删 `.godot/global_script_class_cache.cfg` 再 `--import` 重建。**绝不删 `.godot/uid_cache.bin`**——删了主场景 UID 无法解析，Godot 启动即 abort。

**多实例互斥**：编辑器开着时绝不再起第二个 Godot 实例（含 headless 测试）——抢 `.godot/` 出 file_access 报错、锁死日志文件、游戏窗口僵尸化。

**改配置文件前先 grep 现值**：编辑器开着时会用内存里的旧值顶掉外部写入（export_presets.cfg 等）。

## 排障速查（症状 → 按顺序查）

| 症状 | 排查 |
|---|---|
| "改了没生效" | ① 用户跑的是不是旧包（对比包时间戳与修复时间）② 编辑器进程级资源缓存 → 重启编辑器验证 ③ 导出包不含未保存改动 ④ 批量脚本替换静默跳过 → grep 验证 |
| 测试 ALL PASS 但可疑 | grep `SCRIPT ERROR`；typed 参数收内联数组字面量会中断断言但测试继续跑；`_ready` 调含 await 的协程没 await；`--script` 模式脚本挂载失败变纯 Node 全空转 |
| 场景加载失败 / 实例"消失" | sub_resource 前向引用；变体属性写成裸逗号列表（报错行号指向引用行，病灶在文件头）；sub_resource id 含中文 |
| @export 覆盖不生效（节点都在、无报错） | 手写场景中文属性名没带引号；属性写在 script 行之前（script 必须第一属性） |
| 实例化出来是"另一个资源" | ext_resource 的 uid 抄错——静默换成别的资源，无报错指向 |
| 手机端整块功能哑火、PC 正常 | `FileAccess.file_exists()` 判包内资源恒假 → 换 `ResourceLoader.exists()` |
| 角色沉地 / 穿地 / 悬空 | Terrain3D collision_mode（DYNAMIC 只建相机周围 64m 碰撞）；物理体缩放非 1；落面没取 maxf(射线命中, 数据高度) |
| 射线"无命中/无阻挡" | collision_mask 没显式设；起点扎进目标体内（hit_from_inside）；可命中碰撞体尺寸远小于视觉模型 |
| 验证"修好了"却还在报 | 资源缓存骗人——验证必须独立新进程；grep 默认尊重 .gitignore 会整个跳过 .godot/（要显式路径） |

细节与更多形态见各 reference。
