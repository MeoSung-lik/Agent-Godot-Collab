# 渲染与性能 · 移动端适配 · 截图

背景：GL Compatibility 渲染器 + 手机端发布。Vulkan/Forward+ 项目部分条目不适用（按渲染器标注取舍）。

## 掉帧归因方法论

- 三大真凶排序（反复验证）：①逐帧脚本/轮询（数千拾取物每帧 _process + 管理器全表扫，合计可达帧预算数倍）；②HUD 每帧调全扫接口（多个读取口各扫一遍 = 9~18ms/帧）；③draw call 大盘（GL Compat 无遮挡剔除，数千件全视锥绘制 + 阴影翻倍）。**先杀前两个，再谈 draw call**。
- 性能监视器读数不可信（draw call 读数与真机矛盾、_process 读数不含全部节点脚本成本）——定位必须探针分相实测 + 墙钟。
- 归因四板斧：全树 _process 普查 + 实锤开关 / 逐层二分 / 对照实验（A/B 摘除）/ stash 基线。
- 模拟器数字虚高：arm64-only APK 走二进制翻译 + GLES 转译层，OpenGL 模式比 Vulkan 更慢——模拟器永远不能当真机承诺。
- 真机诊断：`dumpsys gfxinfo` 的 Total frames 统计 Android View 层帧（闪屏/装饰），**不含 Godot SurfaceView 游戏帧**——「冻在 26 帧」可能是假指标；真机「卡死」先截屏看实际画面 + 确认指标含义。

## 优化组合拳（实测 9~12 倍的套路）

- **休眠契约**：远处隐藏模型 + set_process(false)，三态缓存早退（可见/隐藏/休眠），受击即恢复全激活。
- **查询收拢**：0.1s 共享缓存收拢多读取口 + 注册表 O(1)（字典索引 + 尾交换删除）+ 距离粗筛后才进虚调用 + 空扫退避（粗筛无结果推迟到 0.5s，注册置脏立即恢复）。实测：AI 物品扫描 1.23→0.132ms、交互 1.74→0.147ms。
- **空间网格**：近邻查询**格内不做精确距离过滤**——留给调用方单次读坐标，先滤距离会让调用方二读坐标 1.7× 慢。
- **可见距离剔除**：静态资产根 @export「可见距离」运行时套用子树全部 GeometryInstance3D（编辑器不介入）——恒渲染 1.2 万 → 2 千件（-82%）。
- **预生成 + 热身摊销**：新激活实体首次物理活动帧有一次性尖峰（~43ms）——「生成→激活热身（相机前渲染一帧付 shader 编译+地板 2 帧付物理尖峰）→藏匿（process_mode DISABLED）」全做进加载画面，出库=恢复处理零开销。
- 「无上限累积的实体 + 逐帧逻辑」是性能灾难模式：治理组合=寿命淡出 + 数量上限 + 回收最旧 + 对象池 + MultiMesh 单 draw call 演出替身（数千枚非物理体匀速下落 + 空间哈希抓取）。**「复用物理刚体」会把灾难请回来**。

## MultiMesh / 合批（4.7）

- MultiMesh 变换 buffer **不序列化**（tscn/tres 皆丢）→ 静态烘焙必须 ArrayMesh 顶点烘焙 + 外部二进制 .res（烘顶点进 tscn 文本会撑到 91MB）。
- 独立树 Node3D.global_* 恒单位阵——离线合批工具要沿父链手乘局部 transform。
- ArrayMesh 顶点用 `Transform3D * PackedVector3Array`（C++ 速度）。
- 嵌套在外来场景内部的实例不能在宿主场景持久化删除（重载复活）——只能跳过或先「救出」到父层级。
- **合批时碰撞形状必须跟着搬**——只留裸 StaticBody 占位 = 碰撞整体丢失（整棵树可穿透的事故）。
- 运行时 MultiMesh（金币雨类）：`visible_instance_count` 必须与已用数同步——buffer 残留会渲染**幽灵实例**（拾取幽灵币的根因）。
- 高空俯瞰下块级 AABB 整块可见，有 +draw call 固有代价；密集区实测 -26~35%。

## GL Compatibility 限制

- Environment 的 adjustment（对比度/饱和度）后处理不生效（疑空转）——别拿它们调亮度。
- 半透明材质走混合变体会**整根消失**——光柱类用 unshaded 不透明。
- 自发光**不照明其他表面**——夜景大面积增亮走材质边缘光（rim），大平面正视时 rim 趋零。
- 全屏描边在 GL Compat 永久跳过且毒害透明队列。
- 透明底渲染：窗口后备缓冲无 alpha，根视口 transparent_bg 截不出透明底——必须独立 SubViewport。
- 首帧 shader 编译 ~2.2s 是换场景卡顿大头：加载画面满格后在本界面可见期间 instantiate+add_child 挂新场景（不用 change_scene_to_packed），再留 12 帧盖编译期自毁（按帧倒计时在卡帧期间自动冻结=盖得住）；**别单独 free 旧 current_scene（它就是加载界面自己）**。导出预设 `shader_baker/enabled=true` 只治首次编译，不治全屏后期层逐帧 fragment 带宽。

## 灯光调色

- 刺眼 ≠ 亮度高——主因是「材质级 × 环境级」饱和度**双重叠加相乘**。调色先查两层叠加，别只动一层。
- 亮度大头 = 环境光 + 雾，非太阳（GL Compat 实测太阳 1.0→0.3 草地只降 ~19%）；卡通着色下能量走 LIGHT_COLOR（ATTENUATION 只是阴影掩码）。
- 辉光必须设 `glow_hdr_threshold`（缺省 1.0 时自发光 1.5~3.0 全部泛光）。
- ACES + 曝光 >1.0 + 低阈值会把天空/白衣洗白——改 Filmic + 曝光 1.0 + 阈值 1.35。
- 方向光默认 `directional_shadow_max_distance=140`——顶视/远景全图无影，1000m 级才实影。
- visibility_range 低模**必须补 end**（只有 begin 会画到无限远，远景白楼不消失的根子）；扫 LOD 只能扫 GeometryInstance3D（扫 Node3D 会在 StaticBody3D 上报 Invalid access 中断）。

## 内存泄漏定位

- 「每实例 duplicate 独立材质 + 按源建缓存」是材质级内存泄漏模式（缓存条目永留、原生堆匀速涨、节点/GL mtrack 平）——定位用临时 Autoload MemProbe 探针（引擎侧资源泄漏的正解）；修法=高频件进排除组 + 缓存挂清死源钩子（节流，源弱引用已死即删）。
- 共享染色材质级联：递归强套的 ShaderMaterial 作用于整棵子树绘制——某面板要自成配色必须自带空白材质（`material = CanvasItemMaterial.new()`）或 meta 豁免。

## 移动端

- 填充率瓶颈判别：draw call 砍 4-5 倍帧耗不降 = 填充率瓶颈——`scaling_3d/mode=1` + `scaling_3d/scale=0.35` 实测 16→30fps。**坑：只设 scale 不设 mode=1(bilinear) 不生效**。0.35 缩放在大色块卡通风画质无感、2D UI 不受影响。
- `emulate_mouse_from_touch=false` 会杀死全部标准 UI（Button/Slider 收不到合成鼠标）——触屏→鼠标与鼠标→触屏双向模拟两开关缺一不可。
- `DisplayServer.is_touchscreen_available()` 在无头/触屏本恒 true——按它分流会锁死滚轮（滚轮是鼠标专属事件触屏不产生）。
- Android 上 EXCLUSIVE_FULLSCREEN 才完整藏状态栏+导航（WINDOW_MODE_FULLSCREEN 藏不干净且窗口被 inset 缩进）；Autoload 初始化早于 Activity 恢复流程会被覆盖——加 1s Timer 补设一次。
- 音频：每脚步一个 one_shot 实例的实例化风暴挤声——每角色常驻 AudioStreamPlayer3D 换 stream 播放；`mix_rate=48000` + `output_latency=40` 抗 underrun；按表内最大距离对相机距离早退；首帧后分帧预热全变体池消首播 IO 爆音。
- `AudioStreamPlayer3D` 不是 AudioStreamPlayer 子类（Node3D 系）——find_children("AudioStreamPlayer") 采不到一切 3D 音效，采样两类各查；采样+去重窗口会把 11 秒长 wav 假象成「每秒重放风暴」（播报只发一次以信号登记为准）。
- UI 基准视口：stretch canvas_items+expand 下 UI 坐标按基准视口算，不按窗口分辨率算；`parse_input_event` 吃窗口坐标而 `_input` 收 stretch 变换后画布坐标——窗口≠画布时按 (window.size / visible_rect.size) 反算。
- 材质贴图被内嵌成 CompressedTexture2D 固定桌面格式（load_path 写死 .s3tc.ctex）→ PC 正常手机丢图——排查 `grep "s3tc.ctex" --include="*.tres"`。

## 截图 / 录制管线

- SubViewport 截图：`look_at()` → `await process_frame` → `await frame_post_draw` → `get_image()`；实拍等 ≥0.5s（只等一帧拿到旧帧）。
- SubViewport 的 `direct_space_state` 取不到（null 报错）——地面射线走根视口。
- GL Compat 下渲染前藏全屏合成层与污染源（描边/数据雨层）；CanvasLayer 非 CanvasItem，隐藏要 `set("visible")`。
- Movie Maker（4.x 内置）分辨率 = project viewport 基准尺寸（stretch canvas_items 下），`--resolution` 与 `window_set_size` 均无效——正解=工程根临时 override.cfg 覆盖 viewport_width/height，**拍完必须删**。
- 粒子材质必须 unshaded + vertex_color_use_as_albedo（emission 压掉 color_ramp 彩色）。
- 探针/工具输出统一放工程外目录——工程内截图目录会被 Godot 当资源导入出 .import。
- 物理插值：physics_interpolation_mode=OFF 的节点取 `get_global_transform_interpolated()`（父链插值合成慢一拍）会漂移——按节点插值模式取值（OFF→global_transform）。
