# Agent + Godot 人机协同游戏开发

两个成对的 **Agent Skills**（SKILL.md 规范）技能，沉淀自一个完整上线的 Godot 4.7 3D 游戏项目（GL Compatibility 渲染 + Jolt 物理 + Terrain3D 起伏地形）近 200 条真实踩坑与整套人机协作实践。

## 两个技能

| 技能 | 定位 | 适用场景 |
|---|---|---|
| [godot-collab](godot-collab/) | 协作流程（主入口） | 接需求、实现功能、搭场景、做预制件——人机分工边界、方案确认单、自主自测、验收报告、编辑器交接 |
| [godot-dev](godot-dev/) | 踩坑/调试 | 排查"改了没生效/静默失败/测试假通过/沉地穿地/手机端哑火"；写代码、改场景、跑测试、打包前的坑位自查 |

两者成对使用：godot-collab 管流程与分工，末尾按场景路由到 godot-dev 的 8 个坑库文件；**安装时必须放同一目录互为同级**。

## 安装

把 `godot-collab/`、`godot-dev/` 两个文件夹整体拷入所用工具的技能发现目录：

- **ZCode**：`~/.agents/skills/`（全局）或 `<项目>/.agents/skills/`（项目级）
- **Claude Code**：`~/.claude/skills/` 或 `<项目>/.claude/skills/`
- **其他支持 Agent Skills 规范的工具**（SKILL.md + name/description frontmatter）：拷入对应目录即可，格式无需改动
- **无技能机制的工具**：把 SKILL.md 正文并入 AGENTS.md / 系统提示词，`references/` 留作按需阅读的文档

装完**新会话生效**；技能按 description 自动触发，也可手动 `/godot-collab`、`/godot-dev` 调用（视工具支持）。

## 内容概览

**godot-collab（协作流程）**
- 角色分工：编辑器观感归人（场景搭建/材质定稿/验收拍板），逻辑代码归 AI（脚本/流程调度/探针自测）
- 五步循环：澄清需求 → 方案确认单 → 实现 → 自主自测 → 验收报告
- 预制件静态优先：编辑器是唯一搭建与调参面，代码只驱动逻辑与表现（覆写 vs 驱动判据、@export 暴露规范、批量生成例外）
- 配置表驱动数值、编辑器交接单写法、拍板文化

**godot-dev（踩坑库，8 个分册）**
- `gdscript.md`：类型推断、Callable.bind 参数序、协程遗弃、节流计时失真等语言坑
- `scenes-tscn.md`：手写/生成 .tscn·.tres·CSV 的静默失败（中文引号、script 第一属性、uid 纪律）
- `animation.md`：step=0.02 禁 30/45/60、AnimationTree 共享 tree_root、Discrete 轨道
- `headless-testing.md`：无头测试/探针的挂死、假通过形态清单、墙钟法、freed-instance 家族
- `physics-terrain.md`：Jolt 物理体缩放、休眠 Area、Terrain3D 落面公式、禁平地假设
- `rendering-performance.md`：掉帧三真凶排序、MultiMesh 合批、GL Compatibility 限制、手机适配
- `export-packaging.md`：PCK zstd 判定法、Android 签名与 Gradle、导出包资源路径
- `editor-project.md`：导入缓存/uid 治理、编辑器行为、「改了没生效」排查序、git 协作

## 适用范围

- Godot 4.x（规则来自 4.7 实战；Jolt / Terrain3D / GL Compatibility 相关条目按项目实际引擎配置取用）
- 内容为中文；协作流程与踩坑经验本身与工具无关，任何 AI 编码助手都能按文档执行
