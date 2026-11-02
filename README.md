# NCM Decode

自用 Android NCM 解密工具，Kotlin 外壳 + Rust `ncmc_lib`。目标是把“手动解密”和“限时监听”做可靠，界面保持简单；FLAC 输出自动转为 MP3 320 kbps 以兼容只支持 MP3 的播放器。

- 支持手动选择 `.ncm`，恢复到用户选择的 SAF 输出目录。
- 支持一次最长 180 分钟的监听会话，默认 30 分钟。到期后停止接收新任务，排空队列后退出。
- 支持 `ncmdecode://watch/start?minutes=30` 深链接和显式 Intent 触发。
- 输出先写临时文件，Rust 完整解码验证通过后才重命名为 `.mp3` / `.flac`；provider 不支持安全重命名时拒绝发布。
- FLAC 自动转 MP3 320 kbps CBR；MP3 原样恢复不重编码。不删除源 `.ncm`、不做开机启动/常驻/被杀自动拉起。

**实测设备**：Xiaomi Redmi K Pad（25079RPDCC），Android 16 / API 36，arm64-v8a。详见 [真机测试记录](docs/test-record.md) 与 [结论与限制](docs/findings.md)。

## 交付物

| 文件 | 说明 |
|---|---|
| `dist/NcmDecode-0.2.0-arm64.apk` | 可安装主 APK，仅 arm64-v8a，debug key 签名 |
| `dist/NcmTrigger-0.1.0.apk` | 可选外部触发测试 App |
| `dist/ncm-decode-0.2.0-source.zip` | 完整源码（不含 SDK/构建产物） |
| `docs/test-record.md` | 真机测试记录 |
| `docs/findings.md` | 关键问题结论、已知限制 |
| `docs/code-review.md` | 代码审阅、已修复问题和剩余风险 |

## 使用

1. 安装 APK，打开 App。
2. 授权：
   - **所有文件访问**：监听模式必需，用于直接读取网易云下载目录和跨 App `FileObserver`。
   - **通知**：前台服务通知、延长/停止按钮。
   - 电池优化建议设为“不受限制”（锁屏测试见限制说明）。
3. 选择**输出目录**（SAF，持久授权）。手动模式也使用这个目录。
4. 选择**监听目录**。默认 `/storage/emulated/0/CloudMusic/Music`；若该目录为空而其它候选目录含 `.ncm`，App 会自动迁移到候选目录。可用“探测目录”查看常见位置并手动改选。
5. 手动：点“选择 .ncm 并解密”，在系统文件选择器里选文件。
6. 监听：设置分钟数，点“开始监听”，或执行：

```bash
adb shell am start -a android.intent.action.VIEW \
  -d "ncmdecode://watch/start?minutes=30"
```

显式 Intent：

```bash
adb shell am start -n dev.ensomi.ncmdecode/.MainActivity \
  -a dev.ensomi.ncmdecode.action.START_WATCH --ei minutes 30
```

通知操作：HyperOS 上需**长按** NCM Decode 通知，才会展开“延长 30 分钟 / 停止”按钮。也可在 App 内操作。

## 输出规则

- MP3 → `.mp3`，`audio/mpeg`（原样恢复，不重新编码）
- FLAC → 转码为 **MP3 320 kbps CBR** 后发布 `.mp3`，`audio/mpeg`；转码成功后删除临时 FLAC。
  - 编码器为随 APK 打包的 LAME 3.100（`third_party/lame`，LGPL，见 `THIRD_PARTY_NOTICES.md`）。设备上的 MediaCodec `audio/mpeg` 编码器可选且厂商相关，不作为依赖。
  - 采样率与声道数保持不变（44.1/48/32 kHz；64/88.2/96/128/176.4/192 kHz 由 LAME 内部重采样到 MPEG-1 支持的速率并记录 `resampled=true`）。单/双声道以外的 FLAC 不转码。
  - 编码器不可用（不支持的采样率/声道、初始化或编码失败、输出未通过完整解码校验）时，回退为发布原 FLAC 并记录 `TRANSCODE_FALLBACK`，任务仍算成功。
- 文件名优先使用 NCM 元数据 `艺术家 - 歌名`，否则使用源文件名。
- 同名时追加 ` (1)`、` (2)`。
- 不写 ID3/Vorbis 标签；MP3 只包含编码器默认帧（实测无 ID3v1/ID3v2/APE 标签），FLAC 保持原始帧。

## 构建

环境：Windows、JDK 21、Android SDK 36、Build-Tools 36.0.0、NDK 28.2.13676358、Rust stable + `aarch64-linux-android` target。

```powershell
$env:JAVA_HOME='C:\Program Files\Java\jdk-21'
$env:ANDROID_HOME='D:\path\to\android-sdk'
$env:ANDROID_SDK_ROOT=$env:ANDROID_HOME
.\gradlew.bat assembleRelease
```

`app/build.gradle.kts` 中的 `buildRustArm64` 任务会调用 NDK clang 交叉编译 `ncm_jni`，并复制 `libncm_jni.so` 到 `jniLibs/arm64-v8a`。可用 `-PcargoPath=...` 指定 cargo。

更完整的依赖安装、测试命令见 [构建与验证](docs/build.md)。

## 结构

```
app/                     Kotlin UI、SAF、前台服务、FileObserver、队列
trigger/                 可选外部触发测试 App
ncm_jni/                 JNI 桥 + Rust 解密/验证 + 测试 CLI
third_party/ncmc_lib/    vendored MIT ncmc_lib 0.3.1
tools/                   合成 NCM 生成器、SDK（不随源码包分发）
docs/                    测试记录、结论、构建说明
```

## 安全与隐私

- 所有解密在本机进行；没有网络权限，不上传文件。
- 输出目录由用户通过 SAF 选择；监听目录只从 App 已配置路径读取。
- 外部触发只能启动/合并会话，不能指定任意目录。
- 前台服务类型：manifest 同时声明 `specialUse|mediaProcessing`。Android 16 实测只保留 `specialUse`，会话中切换到 `mediaProcessing` 被系统拒绝；实际运行按 `specialUse` 记录。详见 findings。

## 已知限制（摘要）

- 网易云 Android 9.6.15 在本机的真实下载目录是 `/storage/emulated/0/Download/netease/cloudmusic/Music`，不是默认的 `/CloudMusic/Music`。0.1.0 起已加入该候选路径，并在旧默认目录为空时自动迁移；仍可在“探测目录”里手动改选。
- 已用该目录下 27 个真实 `.ncm` 跑通批量监听/解密（212 MB，全部输出 MP3；后又新增 1 个，重跑共 28 个，同样全部成功）。0.1.0 最终构建回归：27 个文件在开始会话后约 8 秒内全部入队，已处理指纹正确跳过；新增一个真实 NCM 副本触发 `CLOSE_WRITE`，约 3 秒后入队并在 1 秒内解密成功，到期后队列为空并正常退出。实时下载过程中的暂停/恢复/取消等行为仍需继续观察，详见测试记录。
- 锁屏下前台服务本身不保证 CPU 持续运行；已为整个限时会话持有 `PARTIAL_WAKE_LOCK`，但 HyperOS 默认电池策略下仍观察到延迟（一次约 80 秒）。建议设为“不受限制”，不保证所有机型。
- 不恢复标签；FLAC 转 MP3 后不再保留无损副本。320 kbps CBR 对安静/低码率源可能比原 FLAC 更大；不支持的采样率/声道会回退为发布 FLAC。
- 不同 ROM 的后台限制可能影响监听实时性；本项目只对上述真机结果负责。
