# 导入缓存 · uid · 编辑器行为 · git 协作

## 导入缓存治理

- 改了 class_name 或重命名脚本后报 `Identifier "Xxx" not declared`：**只删 `.godot/global_script_class_cache.cfg`**，然后 `godot --headless --import` 重建。**绝不删 `.godot/uid_cache.bin`**——它记录文件→UID 映射，删了主场景 UID 无法解析，Godot 启动即 abort（`Main scene's path could not be resolved from UID`）。
- 编辑器开着时没法跑 --import 重建 class_name 缓存——新 class_name 在探针里用 `const Xxx = preload(...)` 绑定，.tres 头去掉 script_class。
- 导入缓存哈希含 res:// 源路径**大小写**：磁盘目录物理大小写与 git 记录不一致时（`Textures` vs `textures`）**每次 --import 都改写哈希无限抖动**。修法=磁盘两步 mv 改回小写 → `rm .godot/imported/<名>-*` → --import 连跑两遍稳定。缓存名只含文件名不含目录——清同名会波及全部同名贴图（重跑 --import 即恢复）。
- GLB 内部 JSON 贴图 URI 大小写与磁盘不一致 → Case mismatch 警告、**大小写敏感平台（安卓）真丢贴图**——GLB JSON 块等长原地补丁（头长度/偏移全不动）。修完编辑器进程内存还缓存旧导入资源——完全关闭编辑器重开；仍警告删 `.godot/editor/filesystem_cache10` 残留后重启（自动重建）。
- 本机 grep（ugrep）默认尊重 .gitignore → `.godot/` 整个被跳过、全工程 grep 假阴性——要扫 .godot 必须显式路径。
- 工程内截图/临时目录会被 Godot 当资源导入出 .import——输出与临时文件放工程外绝对路径。
- csv 默认导入器是 csv_translation（详见 scenes-tscn.md）——新建 csv 手改 .import 为 `importer="keep"`。

## 编辑器行为

- 编辑器开着时会用内存旧值**顶掉外部写入**（export_presets.cfg 等）——改配置前先 grep 现值，改完让编辑器侧重载或重启。
- `@tool` 不随 extends 继承：子脚本必须自带 `@tool` + `is_editor_hint()` 守卫——否则编辑器里跑默认逻辑（占位实例、雾开关 setter 不执行、顶视雾关不掉）。
- 顶视/正交机位从高处看雾深度吃满（整屏糊紫）——编辑器「预览环境」勾选项可清；眼睛隐藏 WorldEnvironment **会存盘**，保存前点回来。
- 已知无害噪音别排查：4.7 `instance_reset_physics_interpolation() is deprecated` 刷屏（引擎兼容层，工程侧无解）、退出时 `resources still in use at exit`、删 .gdshader 后 --import 报 Cannot load shader（editor_layout.cfg 最近文件记录，自愈）。
- 编辑器打开卡顿归因法：隔离副本（robocopy）+ `godot --headless --path <副本> --import` 计时（自退出、走完整编辑器启动链）做对照实验。大工程重开 10~30s 是正常成本；`Package/` 等非资源目录加 .gdignore 省扫描；Defender 加工程目录排除；preload 网是有意摊销设计别惰性化。

## uid 撞车与备份

- 场景备份复制进工程内再 --import，uid 缓存把原路径引用解析到备份副本（实例 scene_file_path 变备份路径，文本 grep 看不出）——**备份必须剥掉 gd_scene 头的 uid 属性**（sed 头行，别丢右括号否则 Parse Error）。
- 临时对比场景（同 uid 拷贝）放工程根被扫进 uid 缓存，删除后报 Cannot open file——`--import` 重扫即消。

## git 协作

- 多分支合并（冲突以本地为准模式）会**静默用旧版覆盖场景文件**——合并后跑结构探针秒验，别只看 diff。
- `git log -S` 默认不查合并提交——考古要加 `--diff-merges`。
- master 侧的错误合并会被 git 视为「有效改动」，之后正常合并不自动修回（三方合并传播了删除）——必须显式 `git restore --source=<正确提交> <文件>` 并 amend 进合并提交。
- `git merge/cherry-pick … | head` 吞失败退出码，`&&` 链在失败后继续跑——结果用 `git rev-parse -q --verify MERGE_HEAD` 或退出码判断。
- 解决冲突**禁止 `git add -A` 收尾**（会把未解决文件的冲突标记原样提交）——逐文件按名单 add，提交前 `git grep -c "^<<<<<<< " HEAD` 复查零残留。
- 审计「未引用资产」：先读资源 .import/.uid 里的 uid 再全工程 grep（project.godot 的 config/icon 走 uid://，文件名 grep 零命中是假象）；`git ls-files -c core.quotepath=off`（中文路径）；Windows 大小写不敏感磁盘与索引大小写不一致会误报缺失；Kenney GLB 是外部贴图引用（内部 uri）非内嵌，审计须解包 GLB JSON chunk。
- worktree 首次 --import 报项目字体 ext_resource 找不到（先有鸡先有蛋）——跑两遍即消；切分支后 .godot 导入缓存过期（报 ctex 无法打开）先 --import 再验证。
- 大场景 .tscn 结构性三方合并手法：节点身份=(parent,name) 三方对比；主线版整体为底 + 只移植对方「不在 base」的真新增节点（「不在主线」≠「对方新增」）+ 引用重定向（同 (type,path) 改写为主线 ext_resource id、id 防撞加前缀）+ 移植块按父子拓扑排序接文件尾 + 重算 load_steps。

## 「改了没生效」排查序

1. 用户实际跑的是不是旧包（对比 Package/ 时间戳与修复时间）——最高频。
2. 编辑器资源缓存进程级——重启编辑器验证。
3. 导出包不含未保存改动。
4. 批量脚本替换静默失败：`str.replace` 源串与实际差一处（行尾注释等）就**静默跳过**——改完必须 grep 验证或跑行为探针。
5. bash quoted heredoc 会把 `\n` 吃成真换行、`\` 折叠——写文件用编辑器写入工具别用 heredoc，字面 `\n` 用 `chr(92)+'n'` 构造。
6. ps1 必须 UTF-8 BOM 否则 PS5.1 当 ANSI 中文全花、命令写单行。

## 探针 / 工具类脚本通用

- 输出目录统一工程外缓存目录 + ASCII 文件名（工程内会被导入、中文名链路乱码）。
- 下载资源校验：GitHub 下载字体等二进制首下会截断——按声明大小校验。
- 写日期以系统时钟为准，不从会话记忆链推算（长会话早期条目带旧日期越推越偏）。
