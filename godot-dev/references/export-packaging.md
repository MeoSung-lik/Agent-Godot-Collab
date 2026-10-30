# 导出打包（Windows / Android / PCK）

## PCK 内容判定

- 导出的 .gdc 是 zstd 压缩（magic `28 b5 2f fd`）——grep 明文判断「包里有没有某修复」**全落空**。可行判定：查包内某天新增的资源文件名，或时间线三件套（提交时刻 → 导出中间产物 mtime → 包体 mtime）。
- 4.7 导出常是散件形态 `assets/.godot/exported/<hash>-<名>.scn`，无整体 project.pck（APK 无独立 .pck 是 gradle 管线常态）——别把「没找到 .pck」当故障。
- zip 包内中文文件名需 cp437 → utf-8 重解码后再比对。
- 4.7 PCK format 4 解析：头 `[GDPC][ver=4][engine×3][flags][file_base u64][data_size u64]`；exe 内嵌 footer=文件尾 `[pck_size i64][GDPC]`；目录条目=`[len][path\0][offset u64][size i64][md5 16]`；目录在包尾、路径无 res:// 前缀。

## 导出包里的资源（整块功能静默哑火的根源）

- 导入资源在包里以 .import 边车列出，**没有源文件**（wav/png/csv 都不在包里）：
  - 存在性判断一律 `ResourceLoader.exists()`——`FileAccess.file_exists()` 判 res:// 导入资源恒 false，整库静默跳过（PC 编辑器永不复现，手机端才炸）。
  - DirAccess 扫目录收源文件在包里必空——`名.trim_suffix(".import")` 后按原路径 load + 同名去重（扫描路径与存在性检查两处都要改，一族坑）。
- GDExtension DLL 打不进 PCK（4.7 桌面导出生成在 exe 旁）：自定义导出插件把 DLL 塞进 PCK 实测**比不带还糟**（包内 LoadLibrary 相对路径报 Error 126，且 PCK 内副本会让 exe 旁查找短路回退）——分发姿势 = exe+dll 伴生 zip。
- 扩展加载失败错误**不进日志文件**（发生在文件日志启动前、只进 stdout，窗口版无控制台）——第一报错是场景加载期 "No loader found for resource"。`--verbose` 跑包存 stdout 验证。

## Android

- **export_presets 的 keystore 两行易丢**（`keystore/release="路径"` + `keystore/release_user="别名"`）：被弄丢不报错、包照样出但可能 debug 签名，提审才炸。每次 CLI 打包前 `grep keystore/release= export_presets.cfg`；导出后必验 `apksigner verify --print-certs` 的 MD5。
- Godot 4.7 预设 keystore/debug 字段**不传给 Gradle**（构建命令无 keystore 属性，模板回退引擎自带调试证书 CN=Godot）——真机 SDK「连不上/包名签名错误」的根因。需要真签名的 gradle 构建：证书放 `android/build/debug.keystore` + 模板 build.gradle 签名配置写死。
- Gradle 导出 stdout 全缓冲无输出（连 gradle 日志都看不到）——**判定完成只看 APK mtime 新鲜且 40s+ 不再变**；gradle 完成后 Godot 管道不退出（需手动收尾）；冷构建 5~12 分钟正常（看 java.exe 进程内存判断在干活）。
- 环境缺失时从 `editor_settings-4.7.tres` grep java_sdk_path / android_sdk_path，命令行前置 JAVA_HOME + ANDROID_HOME。
- APK 验证三件：python 解 manifest 验版本（UTF-16LE 搜版本串——**报包版本以 APK 内嵌为准别信口头**）；apksigner verify --print-certs（JDK17 加 PATH 否则静默无输出）；v2 签名块在包尾无 META-INF/*.RSA 是 apksigner v2 方案常态，keytool 读不了 v2 签名包。
- Android 插件 v2 规范（4.2+）：v1 gdap 插件（android/plugins/）**弃用且导出静默不检测**（Gradle `-Pplugins_*` 空=没检测）——桥接代码并入 android/build 模板源码 + manifest v2 元数据 + build.gradle 挂 Maven 依赖；`<queries>` 必须在 `<manifest>` 层（放 application 内 AAPT 报错）；预设 plugins/* 键无效别加；android/plugins 目录为空是正常的**别误判插件缺失**。
- 沙箱壳（TapPlay 类）PiP 闪退：Godot 模板默认 `isPiPEnabled()=true`，壳 Activity 无 PiP 标志 → 启动完成调 setPictureInPictureParams 被系统拒绝、IllegalStateException 主线程崩（manifest 的 supportsPictureInPicture 救不了）——`GodotApp.java` 的 `isPiPEnabled()` 改 `return false`（疑似所有 4.x 默认模板游戏均踩）。验证：javap 反编译 gradle 中间产物看 isPiPEnabled 是否 iconst_0。
- 提审包调试工具摘除：作弊/调试工具统一 `OS.has_feature("store")` 自摘，导出时临时把预设 custom_features 翻成 "store" 导完翻回——**手工复制的第二预设块逐字节一致仍被 Godot 校验拒**，走「临时翻转原预设」更稳；验证 = unzip 出 assets/project.binary 看头部 `_custom_features: store`。
- 杂项：config/name 中文名会被消毒成非法包名——包名固定拼音 `com.xxx.yyy`；导出 export_path 用 ASCII 名（中文 apk 文件名传输链路疑乱码）；`aapt2 dump badging` 验 label；adb 安装版本验证 python 解 manifest。
- adb/MIUI：fresh install 被 MIUI 拦 `INSTALL_FAILED_USER_RESTRICTED`（USB 安装开着也没用）——`install -r` 覆盖装不拦，手机上保底装着一个包；`adb push` 中文名文件假成功实不落地（用 ASCII 名）。

## 版本与门禁

- 打包前先升 export_presets 版本号——版本号递增比记忆快，旧版本号不能复用（已提审的版本会冲突）。
- 网络时间门禁在真机不可靠：「接口失败/超时/解析失败一律锁死」误伤率极高（真机网络稍有波动就永久锁屏）——门禁应放行失败路径或至少留重试；对外包导出前确认门禁状态与截止时间。

## 云服 / Linux headless 形态

- PCK 装不下 GDExtension（dlopen 要真实文件、地形资产不入包）→ 对局服秒退无崩溃事件——PCK + 侧车磁盘兜底形态。
- 官方编辑器版 Godot 是 debug 变体，加载 `terrain.*.debug`——侧车瘦身删 debug 二进制必挂。
- `--main-pack/--headless` 会被 `get_cmdline_args` 吞掉——PCK 形态识别靠 user args 自报。
