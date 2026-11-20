# hafu
.哈弗大狗各种软件


# 第三方音乐上仪表 · 使用说明书

> 适用机型：22 款哈弗大狗 / 安波福 403A 车机（瑞萨 R-Car H3，Android 9 / API 28，SELinux Enforcing）
> 补丁形态：覆盖更新包（高版本号，`/data/app` 副本，可卸载回滚，不改 `/system` 分区）
> 宿主应用：`com.aptiv.thirdmediaparty`（原车第三方音源桥接服务）
> 当前版本：versionCode 29 / versionName 10（原车 28 / 9，仅高一位），平台签名与原车一致

---

## 一、这个补丁是做什么的

原车机只让内置的酷我、喜马拉雅、考拉、爱趣听、U 盘、蓝牙、FM 等专属音源把歌名 / 状态 / 进度 / 封面推到液晶仪表；普通第三方音乐 APP（走 Android 标准媒体会话 MediaSession）上不了仪表。

本补丁在原车第三方音源桥接服务内增加一个**标准 MediaSession 采集器**：自动枚举当前正在播放的标准媒体会话，把第三方音乐 APP 的歌名、歌手、专辑、播放状态、进度、专辑封面，复用原车既有报文通道上报到仪表。它与原车各专属音源并行工作，已有专属通道的音源优先，互不串台。

## 二、已实现并在车上实证的功能

### 1. 自动识别并选择音源
- 注册系统媒体会话监听，会话列表变化时自动重新扫描；另有每 4 秒兜底重扫，监听漏发 / 会话晚建立时自动恢复选源。
- 多个活跃会话按规则评分选源：正在播放 / 缓冲 > 暂停 / 快进快退 > 已停止；当前源仍在播放时保持锁定，旧源暂停 / 停止后让位给正在播放的新源，避免旧曲目粘滞。
- 前台正在使用的 APP（`getRunningTasks` 顶部包名）在评分上加权优先，避免后台残留会话抢过前台正在听的源。
- **云听按 AIDL 进度增长判活**：云听标准会话不回传 PlaybackState（state 为 null），不能按标准播放态识别。补丁通过其 AIDL 每 1 秒读取 `progress`，进度在增长即判为在播（给播放分），超过 3 秒未增长视为暂停；判活独立于当前选中源——只要云听会话在候选列表里就后台刷新进度，会话消失才清缓存，不会因为当前选的是别的 APP 就连带把云听在播状态清掉。判活由停转播、而当前源不是云听时，立即在主线程重选一次，不等 4 秒兜底重扫。
- 排除已有专属通道和系统内部音源，避免与原车通道重复上仪表。

### 2. 曲目信息上仪表
上报当前曲目：歌名（标题）、歌手、专辑名；元数据缺失时对应字段留空，不传空指针。

### 3. 播放状态同步
把 APP 播放状态映射为仪表状态：

| APP 状态（Android） | 仪表状态 |
|---|---|
| 播放中 / 缓冲中 | 播放 |
| 暂停 | 暂停 |
| 快进 | 快进 |
| 快退 | 快退 |
| 其他 / 异常 | 无效 |

切歌、暂停、继续、停止时，通过会话状态回调实时刷新。

### 4. 播放进度同步
- 每 1 秒轮询一次，上报当前播放位置与总时长（单位：秒，与原车通道一致）。
- 进度按"起点位置 + 经过时长"估算；暂停后进度停住。

### 5. 专辑封面上仪表
- 优先取元数据 `ALBUM_ART` 封面 Bitmap，其次 `ART`、再兜底 `DISPLAY_ICON`；压缩为 JPEG（质量 75）后经原车文件传输通道（transferFile）推到仪表。
- APP 只给封面 Uri（无 Bitmap）时，后台线程限时 4 秒下载该 Uri 图片再上报；下载成功后本曲目内锁定以 Uri 图为准，避免残留旧 Bitmap 把新封面回刷成上一首。
- 封面文件编号与曲目编号一致，仪表据此把封面关联到当前曲目。
- 封面内容按压缩字节 hash 单独去重：曲目未变但封面更新时，沿用当前文件编号重传覆盖；晚到的真封面由 1 秒复查线程补发。
- 车上实证：汽水音乐、网易云音乐封面正常；云听经 AIDL 取到专辑图 Url 后下载上报，0.2–0.3 秒内拿到新封面。

### 6. 曲目 / 封面去重
- 以"包名 + 歌名 + 歌手 + 专辑 + 时长"作曲目签名；同一首歌未切换时，只刷新状态和进度，**不重复下发曲目报文、不重复传封面**，避免文件编号单调暴涨和重复占用传输通道。
- 封面另按图片字节 hash 判断是否变化，避免同曲目封面重复传输；重新抢占音源后会强制补发一次曲目与封面。

### 7. 音源抢占与让位
- 有第三方音乐会话活跃时，向仪表抢占"在线音乐"音源（sourceType 7）。
- 会话退出、销毁、停止 / 报错时，主动放弃音源并清空显示，不残留旧歌名。
- 原车专属音源（U 盘、蓝牙、FM、酷我等）由各自通道上报，采集器按包名让位，不抢它们的仪表位。

### 8. 云听（`com.edog.car`）解耦取数
云听基于考拉 FM SDK，标准 MediaSession 状态 / 元数据不完整（state 为 null、歌手字段为空）。补丁额外绑定其导出 AIDL 服务 `com.kaolafm.launcher.client.ClientService`（descriptor `com.kaolafm.sdk.client.IClientAPI`，事务码 12 `getCurrentMusicInfo`）直接取当前播放信息：
- 歌名 / 歌手：SDK 把歌手拼在歌名后缀"（演唱：歌手）"里，正则拆出歌手、清洗歌名后缀；歌手为空时优先 SDK `authorName`，再用括号解析值。
- 封面：取 SDK 专辑图 `picUrl`，按第 5 节 Uri 下载通道上报。
- 进度 / 时长：用 SDK 返回的 `progress` / `duration` 兜底，1 秒周期后台刷新；`progress` 是否增长同时作为云听在播判活依据（见第 1 节）。
- 云听会话从媒体会话列表消失时，才清空该路 AIDL 缓存（曲目 / 进度 / 判活标志）；当前切到别的 APP 播放时，保留云听缓存供独立判活与后台探测，不连带清空。

## 三、仪表上能看到什么

第三方音乐 APP 播放时，仪表对应音源位显示：
- 歌名、歌手、专辑名；
- 播放 / 暂停等状态标识；
- 播放进度与总时长；
- 当前曲目专辑封面。

> 音源类别按"在线音乐"通道（sourceType 7）上报。报文里只带音源编号、不带音源名字段；仪表上这一路显示什么名称，由仪表 MCU 固件按编号在自身名称表里查出，车机补丁只能选编号、填不进自定义音源名。

## 四、支持与不支持的音源

### 已在车上实证可用
| APP | 包名 | 封面来源 |
|---|---|---|
| 汽水音乐 | `com.luna.music.car` | `ART` Bitmap（228×228） |
| 网易云音乐（车机版） | `com.netease.cloudmusic.iot` | `ALBUM_ART` / `ART` Bitmap（320×320） |
| 云听 | `com.edog.car` | 考拉 SDK AIDL 取 `picUrl` 下载 |

### 不上报（已由原车专属通道或系统内部覆盖，自动排除）
包名前缀匹配以下的音源不进入采集器：

| 前缀 | 对应音源 |
|---|---|
| `com.gwmv3` | 原车系统应用（U 盘音乐等） |
| `com.tencent.wecarflow` | 爱趣听（腾讯车联） |
| `com.android.bluetooth` | 蓝牙音频 |
| `com.android.systemui` | 系统界面 |
| `com.aptiv` | 原车多屏 / 系统服务 |
| `cn.kuwo` | 酷我音乐车机版（原车专属 SDK 通道） |
| `com.ximalaya` | 喜马拉雅 |
| `com.gwm.app.bookshelf` | 原车书架 / 有声书 |

## 五、当前版本的限制与已知观察项

1. **云听遗留项**：SDK 未单独给专辑名，专辑位仍显示"分类 歌手"拼接串（歌手已单独拆到歌手位）；封面为专辑级，同专辑曲目共用一张图；首轮绑定 AIDL 尚未连上时可能取一次空，约 1 秒后自愈；本地缓存图路径 `localPic` 暂未用于读图，现走 `picUrl` 下载。云听在播判活依赖 AIDL `progress` 持续增长：若该 APP 某首曲目进度回调长时间不动（如直播 / 静态页），可能被误判为暂停而让位给其他源。
2. **进度为秒级估算**：1 秒上报一次；个别 APP 不回传准确位置时进度可能有偏差。
3. **封面刷新有短暂延迟**：切歌 / 切 APP 后，封面要等 Bitmap 加载或 Uri 下载完成再经文件通道传输；异步晚到时仪表会先出文字、后出图，1 秒复查线程会补发晚到封面。
4. **偶发落到视频槽位**：个别时机下仪表可能短暂显示到第三方音频 / 视频槽位（编号 18），属音源让位间隙的仪表回退显示；补丁自身报文固定发编号 7，未主动发 18。复现时需同步抓服务端报文定位。
5. **蓝牙等原生音源抢入后的自动夺回**：采集器抢占后不持续重发请求；蓝牙等原生音源抢入压下第三方源后，是否自动夺回策略尚未固化。

## 六、安装（覆盖更新）

### 前置条件
- 车机已可在 root / adb shell 下执行 `pm install`（本机 adb shell 为 root）；
- 补丁包与系统原包同包名、同平台签名、同 system 身份，versionCode 更高，可作为更新覆盖安装。

### 步骤
1. 把补丁包推到车机临时目录（示例文件名 `TP_v29.apk`）：
   ```
   adb push ThirdMediaPartyService_final_v29.apk /data/local/tmp/TP_v29.apk
   ```
2. 覆盖安装：
   ```
   adb shell pm install -r /data/local/tmp/TP_v29.apk
   ```
   > 若车上此前装过版本号更高的调试包（如 49），`29 < 49` 会被版本降级规则拒绝；需先卸载覆盖版回落原车 28，再装本定稿包：
   > ```
   > adb shell pm uninstall com.aptiv.thirdmediaparty
   > adb shell pm install -r /data/local/tmp/TP_v29.apk
   > ```
3. 该服务非常驻，安装后需手动拉起（或重启车机由系统拉起）：
   ```
   adb shell am force-stop com.aptiv.thirdmediaparty
   adb shell am startservice -n com.aptiv.thirdmediaparty/.ThirdMediaPartyService
   ```

### 确认更新副本在运行
```
adb shell dumpsys package com.aptiv.thirdmediaparty | grep -E "versionCode|codePath|flags"
```
- `codePath` 指向 `/data/app/...`、`versionCode=29`、flags 含 `UPDATED_SYSTEM_APP`，即更新副本生效。

## 七、回滚

卸载更新副本即回到 `/system` 原车版本（versionCode 28），不改动 `/system` 分区：
```
adb shell pm uninstall com.aptiv.thirdmediaparty
```
或在设置 → 应用 → 找到本应用 → 卸载更新。原车 APK 已单独备份，可随时比对 / 还原。

## 八、验证与排障

### 抓日志
```
# 采集器自身日志（初始化、选源、曲目 / 状态 / 进度 / 封面上报、异常）
adb shell logcat -d -s TP.MS:*

# 服务端音源抢占与文件传输
adb shell "logcat -d | grep -E 'MultiDisplay.Service|MultiDisplay.FileTransfer|file transfer|storeFile'"
```
- `TP.MS`：补丁采集器日志标签。
- `MultiDisplay.Service` / `MultiDisplay.FileTransfer`：原车多屏服务端，记录音源抢占、封面文件传输结果（`226 Transfer complete` 即传输成功）。

### 上车观察清单
- 第三方音乐播放：歌名、歌手、专辑是否出现；
- 播放 / 暂停状态是否跟随；进度条是否走动；
- 封面是否随曲目显示、切歌后是否换成新封面；
- 切歌后歌名是否更新；退出 / 停止后仪表是否清空、不残留；
- 切回 U 盘 / 蓝牙 / FM / 酷我，确认采集器让位、不串台、不重复。

### 常见问题
| 现象 | 可能原因 | 处理方向 |
|---|---|---|
| 装不上，报 `UPDATE_INCOMPATIBLE` / `SHARED_USER_INCOMPATIBLE` | ROM 禁止覆盖 system 核心应用 | 改用直接替换 `/system/priv-app` 文件的方式（备选回退方案） |
| 装上但仪表无显示，日志枚举不到会话 | 会话枚举权限被收紧 | 核对宿主权限后重出包 |
| 歌名 / 歌手为空 | APP 元数据未填标准字段 | 属 APP 侧数据缺失，非补丁问题 |
| 有文字、封面空白 | APP 既未给 Bitmap、封面 Uri 也不可达（超时 / 无图） | 查 `TP.MS` 封面日志；属 APP 侧未提供可用封面 |
| 进度不动或偏差大 | APP 未回传准确播放位置 | 按日志调进度策略 |
| 两个 APP 间仪表来回跳 | 多会话选源抖动 | 收紧选源评分 / 加当前源粘滞 |
| 云听在播却被暂停的其他 APP 顶掉 | 云听无 PlaybackState，判活未置真 | 查 `TP.MS` 里 `edog alive=` 与 `prog=`：进度在增长应 `alive=true`；若一直 false，核对 AIDL 是否连上、该曲目进度回调是否在动 |

## 九、技术规格（备查）

| 项 | 值 |
|---|---|
| 宿主包名 | `com.aptiv.thirdmediaparty` |
| 版本 | versionCode 29 / versionName 10（原车 28 / 9） |
| 身份 | `sharedUserId=android.uid.system`，privileged system app，非 persistent |
| 签名 | 平台密钥（与系统原包同证书，SHA-256 `c8a2e9bc…f1192ab8`） |
| 音源类别 | sourceType 7（ONLINE_MUSIC 在线音乐） |
| 音源编号覆盖 | 反射读系统属性 `persist.tp.source`，合法 0–32 覆盖；缺省 7；改后需 force-stop 再拉起本进程才重读 |
| 曲目信息 | `updateNowPlayingTrackInfo`（MediaTrackInfo：歌名 / 歌手 / 专辑 / 封面文件 ID） |
| 播放状态 | `updatePlaybackStatus` |
| 播放进度 | `updateNowPlayingTime`，单位秒 |
| 封面 | `updateNowPlayingAlbum(mediaType, fileID, InputStream)`，JPEG 质量 75，经 transferFile 传输；Bitmap 三级兜底（ALBUM_ART→ART→DISPLAY_ICON），无 Bitmap 时后台限时 4 秒下载封面 Uri；按字节 hash 去重、1 秒复查补发 |
| 云听取数 | 绑定 `com.kaolafm.launcher.client.ClientService`，AIDL descriptor `com.kaolafm.sdk.client.IClientAPI`，事务码 12 `getCurrentMusicInfo`；正则拆"（演唱：歌手）" |
| 采集器包 | `com.aptiv.thirdmediaparty.ms`（ClusterBridge / MsCollector / EdogAidl / 回调 / 定时任务） |
| 进度刷新周期 | 1 秒；兜底重扫 4 秒 |
| 最低 / 目标 SDK | 19 / 31（运行于 Android 9，API 28） |

## 十、对后续 U 盘系统升级的影响

- 本补丁是 `/data/app` 覆盖更新副本，**未改 `/system`、`boot`、`vbmeta`、`vendor`、`extbin` 等引导与校验分区**；本固件 U 盘包为整包 raw 直写，不校验车机当前 system 是否被改动，因此**不会挡住后续插 U 盘整包升级**。
- 升级前建议先卸掉更新副本：
  ```
  adb shell pm uninstall com.aptiv.thirdmediaparty
  ```
  避免"只刷 system 槽、不清 userdata"的局部升级场景下，高版本副本残留遮蔽新系统版。
- 整包升级会用官方镜像覆盖 system、重置 userdata，补丁随之失效、回到干净官方状态；需要同款功能时，基于新固件的原车桥接 APK 重新接线、回编、平台签名，不能直接沿用本补丁 APK。
- 若将来官方出**增量 OTA**（按 block / 文件哈希校验当前 system），被改过的文件可能导致增量校验失败；那种情况需先刷回干净官方 system 再升级。本固件当前 U 盘包为整包直写，不受此限。

> （注：部分内容由豆包 AI 生成）