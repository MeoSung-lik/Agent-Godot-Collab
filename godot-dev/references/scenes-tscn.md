# 手写 / 生成 .tscn · .tres · CSV 配置表

手写或脚本批量改场景文件前通读。这里的坑共性：**静默失败**——场景照常加载、节点都在，唯独你的改动没生效。

## 中文属性覆盖必须带引号（最高频）

- 手写 tscn 覆盖 `@export` 中文属性必须带引号：`"大厅模式" = true`。不带引号**解析器静默忽略该行**，运行时就是脚本默认值，无任何报错。排查特征：场景能加载、节点都在、唯独覆盖不生效。
- 同规则适用于 project.godot 的 `[input]` 中文动作名（`"背包"={...}`）——不带引号 InputMap 里查无此动作。
- 反向教训：「带引号的中文覆盖也失效」的真因通常是场景本身加载失败——先验场景完整加载。

## 属性顺序与类型

- tscn 属性按序应用，**script 必须第一属性**——script 之后的属性才会生效保留，插在 script 之前的会被重置。编辑器保存的场景 script 恒第一即此理；手写工具重排属性照此。
- 变体类型属性必须构造器形式：`size = Vector3(1, 0.1, 1)`。裸逗号列表 `size = 1, 0.1, 1` 无效——sub_resource 注册失败，报 `Parse Error: Invalid parameter` + `!int_resources.has(id)`；**报错行号指向引用行不指定义行**，去文件头 sub_resource 块找病灶。浮点 `-0` 归一成 `0`。
- 类型对照易错：`BoxMesh`/`BoxShape3D` 的 `size` 是 `Vector3`；`PlaneMesh` 的 `size` 是 **`Vector2`**。写错只报 "Expected N arguments for constructor" 不指名属性，逐行对照类型文档查。
- 新增/删除 sub_resource 必须同步改头部 `load_steps`。
- sub_resource 定义必须在首个 `[node]` 标签之前，且**禁止前向引用**（引用后面才定义的材质 → 整场景 Parse Error，实例消失）。
- 全屏遮罩 ColorRect 必须 `mouse_filter = 2`（IGNORE）——否则吃掉下层所有点击，真机"点不了按钮"的常见根因。
- 场景根节点漏 `visible = false` 会初始就渲染。

## ASCII id 与节点路径

- sub_resource/ext_resource 的 id 必须纯 ASCII。中文 id 场景能加载，但每次加载刷 `The scene unique ID must contain only letters, numbers, and underscores` 且**不指名是哪个文件**。
- metadata 键必须 ASCII——tscn 里的 `metadata/中文键` 实例化即报 `Invalid metadata identifier`；`set_meta` 中文键同样报错。
- **多段中文 parent 路径会炸**：`parent="联机UI/TouchLayer"` 实例化解析失败 → 子节点散落根上改名（父路径#名）→ `$` 引用 null 崩；**单段中文 parent 全部正常**。手写场景节点尽量一层平铺，嵌套结构标识用 ASCII。
- `[connection]` 的 from/to 路径不跟层级改名 = **静默失效**：按钮在、方法在、点击无反应 → 先查 connection 路径。
- 主题 type variation 名必须 ASCII。

## uid 纪律

- 手写 tscn 的 ext_resource uid 抄错 → **静默实例化成另一个资源**（实例是裸模型无子节点、get_node 找不到、无报错指向）。没有把握就一律不写 uid 属性。
- `PackedScene.pack()` + ResourceSaver 存出的 .tscn 头部不带 uid（4.7 headless）——需后处理补 `uid="uid://…"` 再 `--import` 注册；任何「场景 API 重存」工具都要**保留原 uid**。
- 工程内备份场景副本必须剥掉 gd_scene 头的 uid 属性，否则 uid 缓存把原路径引用解析到备份副本（实例 scene_file_path 变备份路径，文本 grep 看不出）。临时对比场景（同 uid 拷贝）删除后报 Cannot open file——`--import` 重扫即消。

## 程序化生成场景

- `pack()` 只序列化 owner 指向场景根的节点——忘设 owner 存出空根；**owner 必须在 add_child 之后设**（之前设报 "Invalid owner" 且静默丢节点）。
- 代码 new 的物理节点必须 `add_child(碰撞形状)`——漏了形状不进物理空间，测试全员平行自由落体假通过（玩家坠到 -364m 才暴露）。
- 容器换类型（VBoxContainer → Control 等）必须同步脚本的 typed `@onready` 声明，否则 F5 直接崩。
- editable children 覆写范式：文件尾 `[editable path="宿主"]` + 覆写节点 `[node name="X" parent="宿主" index="0"]`。

## group 与 CSV 导入

- tscn 里给节点加组必须写在节点头部属性：`[node name="X" type="Node3D" parent="." groups=["组名"]]`——写成节点下方的独立属性行**不会被解析**，`get_first_node_in_group` 查不到。
- CSV 默认导入器是 csv_translation：新建 csv 跑 `--import` 会生成 .translation 边车、导出包后 FileAccess 读不到源 csv——手改 .import 为 `importer="keep"`、删边车、重跑 --import。
- CSV 行字段数必须与表头严格对齐——少一列**整体左移**，加载器按列位读、短行丢尾列且不报错。测试加「行齐全」断言。
- GDScript 中文 Dictionary 键与点号访问（`条目.弹夹容量`）4.7 完全可用。表头/参数键建议中文，id 与 res:// 资源路径绑定值保持英文。

## 场景手术（脚本批量改场景）

- 搬带子节点的节点必须连子块一起切，只切节点头会子节点失配（@onready null）。
- 正则 `.*?\n\n` 删节点会吞紧邻单换行的下一节点。
- 提取 ext_resource id 必须用 `\bid="` 词边界——否则先命中 `uid="uid://..."` 里的 id= 子串，引用重定向全部静默空转。
- 新增 ext_resource 的 id 要查重（前缀冲突会静默串资源）；parent 行逐个核对——少一层 `/行` 节点挂错父、`$` Node not found、_ready 半中断且多年未察觉。
- 文本场景超 ~1 MiB 弹「很可能包含嵌入二进制数据」是固定误伤：0 处 PackedByteArray + 全是 instance= 摆放即正常体量。**不转二进制 .scn**（要 diff 要手编）。
- 大场景 .tscn **不能线程加载**——文本场景线程解析在快速退出时被撕裂报 parse error。

## 材质

- 多实例各自改材质必须 `resource_local_to_scene = true`，否则共享材质互相覆盖。
