# 第 8 讲：音频采集 —— AudioCapturer（WASAPI 抓声音 + 统一成 AudioChunk + 丢进队列）

> **对应提交**：`1ce7f62`（2026-10-18 19:39）
> **提交信息**：用 Windows 的 WASAPI 抓到声音，统一转换成项目约定的 AudioChunk 格式，再丢进队列给下游用
> **改动规模**：5 个文件，约 940 行（新增 `src/core/capture/AudioCapturer.h` 31 行、`AudioCapturer.cpp` 501 行、`tests/test_audio_capture.cpp` 400 行，两个 `CMakeLists.txt` 各加几行，`.gitignore` 加一行 `*.wav`）
> **本讲时长建议**：约 2.5 小时（含动手实验）

---

## 开场白：画面已经有了，该给直播配上声音了

同学们，我们先回头看看我们已经走到哪一步了。

前面几讲，我们把"画面"这条路彻底打通了：

- commit 4：`ScreenCapturer` 把桌面抓成 `VideoFrame`（BGRA 裸像素）；
- commit 5：`CameraCapturer` 用 DirectShow 把摄像头帧接出来，装进 `CameraRawFrame`；
- commit 6：`CameraFrameConverter` 把五花八门的摄像头原生格式统一成标准 `VideoFrame`。

也就是说，**画面已经有了，而且已经统一成"标准件"了**。

但是同学们，一个只有画面的直播，观众打开三秒就会关掉。人能忍受画质差，忍不了没声音。**声音是直播的"另一半灵魂"**，没有它，这场直播在观众眼里就是"坏了"。

所以今天这一讲，我们要完成采集层的最后一块拼图：**把声音也抓进来，并且统一成项目约定的 `AudioChunk`，丢进队列给下游用**。

先请大家看一眼今天新模块的样子，和我们熟悉的摄像头模块做个对比：

```cpp
AudioCapturer capturer;
capturer.open(AudioSourceType::SystemAudio, -1, log);   // 打开系统声音（回环）
AudioChunkPtr chunk = capturer.nextChunk(50);           // 阻塞等下一块
```

是不是很眼熟？`open / nextChunk / close` 三件套，和 `CameraCapturer` 的 `open / nextFrame / close` 几乎一模一样。这是**有意的对称**——采集层的两个源（画面、声音）对外呈现同一种使用姿势，上层写起来就不用动脑子切换。

但是！同学们千万别被这个"长得像"骗了。**音频采集和视频采集的本质，完全不同**。这一个差异会贯穿整讲，请在开讲前就把它刻进脑子里：

### 本质差异一：连续采样流 vs 离散帧

视频是**离散的帧**。摄像头每隔 33.3 毫秒（30fps）吐出一张完整的画面，帧与帧之间有明确的边界。每一帧自带"宽度 × 高度"的二维结构，你拿到一帧就知道它有多大。

音频是**连续的采样流**。声卡以固定频率（比如每秒 48000 次）对声音波形"点一次名"，每次点名得到一个数值。这些数值**像水流一样连绵不绝，没有天然的"块"的概念**。

所谓"一块音频"（我们叫它 `AudioChunk`），是**我们人为切出来的**——系统攒够多少个采样点就打包给我们一次，我们就管这一包叫一块。这个"块的大小"既不是固定的，也不由我们决定，而是由声卡的缓冲区节奏决定的。

所以请大家记住这个对照表：

| | 视频（CameraCapturer） | 音频（AudioCapturer） |
|---|---|---|
| 数据形态 | 离散的帧，天然有边界 | 连续采样流，边界是人为的 |
| 一"份"叫什么 | 一帧（frame） | 一块（chunk / packet） |
| 一份的大小 | 固定（宽 × 高 × 每像素字节） | **不固定**（取决于缓冲区节奏） |
| 驱动者 | 摄像头硬件，固定帧率 | 声卡 + 系统混音器，固定采样率 |
| 丢一份的后果 | 画面卡一下，下一帧照常 | **时间轴出现空洞，音画开始不同步** |
| 用什么接 | 回调线程 → pending 单槽 | 工作线程 → `BoundedQueue`（队列） |

最后一行那个差异，我要单独强调一下，因为它是今天最重要的一个设计变化：

**摄像头用"单槽最新帧"（丢旧保新），音频必须"一块都不能丢"（用队列）。**

为什么？因为丢一帧画面，观众看到的是卡一下，下一帧来了就恢复正常，**不会累积**。而丢一块音频，时间轴上就少了那么几十毫秒，这一块的时长**永远不会补回来**——从丢的那一刻起，声音就比画面"早"了一点点，而且随着丢的块越来越多，这个偏差**一直累积下去**，几分钟后就变成"嘴型对不上声音"的灾难。

本讲测试程序的最后一句话，就是这条真理的浓缩：

```
音频验收只要记住一条：
  采了多久，就该有多少帧。少一帧都会在几分钟后变成音画不同步。
```

这句话请同学们抄下来。它是今天整讲的验收标准，也是判断任何音频采集模块写得好不好的唯一硬指标。

### 本质差异二：声音没有"静音帧"这回事

还有一个反直觉的地方，同学们提前有个心理准备：

摄像头不管你对着墙还是捂住镜头，它**永远在吐帧**。而音频不一样——**当系统真的没有播放任何声音时，WASAPI 的回环采集（loopback）一个包都不给你**。

这不是 bug，是设计。声卡没活干就不产生数据，很合理。但对我们是灾难：如果下游编码器等着音频块才能干活，系统一安静，你的音频时间轴就**彻底静止**了，而视频还在按 30fps 往前跑——音画立刻错位。

所以今天的模块里有一段非常关键的补救逻辑：**静音补帧**。用一个高精度时钟盯着"理论上现在应该采到多少采样了"，发现实际采到的不够，就自己造一块全零的静音块补进去。**没声音也要凑够时间**——这就是音频采集独有的"保底机制"，视频那边从来没有这个烦恼。

好，两个本质差异讲完了。另外提醒一下：今天我们会第二次遇到 **COM**（第 5 讲 DirectShow 那套东西）。不过这次轻松多了——音频这套 API 的头文件都在 SDK 里活着，不用我们再手写了。我们今天真正要啃的硬骨头是**音频格式本身**（采样率、位深、声道、浮点还是整数）和**缓冲区时序**。

我们正式开始。

---

## 第一章：知识点预习

今天的预习内容比往常多，因为音频有一整套视频那边没有的概念。但请大家放心，这些概念都很直观，我们一个一个用大白话讲清楚。每一条都会在后面的代码里出现，一个都不会浪费。

### 1.1 PCM：声音是怎么变成数字的

先问一个最根本的问题：**声音是什么？**

声音本质是空气的疏密振动，传到耳朵里就是气压随时间的变化。如果我们画一张图，横轴是时间、纵轴是气压相对正常值的偏移，那么声音就是**一条上下起伏的连续曲线**。

要把这条曲线存进电脑，就得把它变成一串数字。办法很朴素：**每隔一小段时间，量一次这条曲线的高度，记下一个数**。量得足够密，这一串数字就能还原出原来的曲线。

这种"直接记录波形高度值"的编码方式，就叫 **PCM（Pulse Code Modulation，脉冲编码调制）**。同学们不用被这个学名吓到，它其实就是"**把波形按时间点采样成数字**"，是最原始、最没有压缩的音频表示法。WAV 文件里装的通常就是 PCM 数据，MP3/AAC 是对它做压缩后的产物。

PCM 有三个决定性的参数，也就是我们天天挂在嘴边的"音频三要素"：**采样率、位深、声道数**。下面三节分别讲。

### 1.2 采样率与采样定理：为什么是 44100 和 48000

**采样率（sample rate）** 就是"每秒量多少次"，单位是赫兹（Hz）。

- 8000 Hz：电话音质，只够听清人说话；
- 44100 Hz（44.1 kHz）：CD 音质，音乐行业标准；
- 48000 Hz（48 kHz）：影视/视频行业标准，也是**我们项目 `StreamConfig` 里写死的 48000**。

为什么是四万多这么个奇怪的数字？这就要说到通信领域最著名的定理之一——**采样定理（Nyquist-Shannon 采样定理）**：

> 要无失真地还原一个信号，采样频率必须**至少是信号中最高频率的两倍**。

直觉上怎么理解？想象一条正弦波，如果每个周期我只量一个点，我可能量到的全是零点（整条曲线看起来就是一条直线）——完全还原不出来。如果每个周期量两个点，至少能看出它在上下动。再多量一些，还原得就越像。**"每个周期至少量两个点"就是"两倍频率"这个说法的来历**。

人耳能听到的最高频率大约是 20000 Hz（20 kHz，而且随着年龄增长还会下降）。两倍就是 40000 Hz。44.1 kHz 就是在 40 kHz 基础上留了点余量——这就是它的来历。48 kHz 同理，是影视行业定的另一个标准值，留的余量更大一些。

采样定理还有一个反面教训，同学们要记住：**如果采样率不够高，高频信号会"伪装"成低频信号**，这个现象叫**混叠（aliasing）**，听起来就是一种刺耳的金属噪声。所以采集到的音频如果要降采样率（比如 48k 降到 44.1k），必须先把高频成分滤掉再做，否则会引入混叠噪声。这件事叫**重采样（resampling）**，我们 1.11 节会再提。

### 1.3 位深：每个采样点用几个字节记

采样率决定了"横轴有多密"，**位深（bit depth）** 决定"纵轴有多精细"——也就是每个采样点的数值用多少位二进制来表示。

- **8 位**：256 个等级，粗糙，有明显底噪，基本淘汰；
- **16 位**：65536 个等级，CD 音质，动态范围约 96 dB，**这是最主流的"PCM16"**；
- **24 位**：专业录音用；
- **32 位浮点**：不是"更精细的刻度"，而是**换了一种记数法**（见 1.5 节）。

位深每增加 1 位，动态范围大约增加 6 dB（因为每多一位，可表示的等级翻倍，对应的能量比是 20×log₁₀2 ≈ 6.02）。所以 16 位 ≈ 96 dB，24 位 ≈ 144 dB。

对我们的代码来说，位深直接决定**一个采样点占几个字节**：

```
16 位 → 2 字节（int16_t）
32 位浮点 → 4 字节（float）
```

后面算缓冲区大小、算 stride、算 chunk 里有多少字节，全靠这个换算。

### 1.4 声道与交错布局：L R L R L R ...

**声道（channel）** 就是"几路独立的声音"：

- 1 声道：单声道（mono），一个麦克风、一路信号；
- 2 声道：立体声（stereo），左（L）+ 右（R）；
- 6 声道：5.1 环绕；8 声道：7.1 环绕。

关键问题来了：两路（或多路）声音在内存里怎么排？

答案叫**交错（interleaved）**布局——也叫"交织"、"打包（packed）"：

```
立体声，4 个采样帧的内存排列：

  索引:  0    1    2    3    4    5    6    7
       ┌────┬────┬────┬────┬────┬────┬────┬────┐
       │ L0 │ R0 │ L1 │ R1 │ L2 │ R2 │ L3 │ R3 │
       └────┴────┴────┴────┴────┴────┴────┴────┘
        └─── 第0帧 ───┘└─── 第1帧 ───┘└─ 第2帧 ─┘
```

注意这里有个术语陷阱，同学们一定要分清：

- **采样（sample）**：单个数值，一个声道在某一个时刻的高度值。
- **帧（frame）**：同一时刻**所有声道**的采样点合起来叫一帧。立体声的一帧 = 2 个采样。

所以：

```
采样总数 = 帧数 × 声道数
帧数     = 采样总数 ÷ 声道数
```

今天的代码里 `totalSamples` 这个变量，虽然名字里有 "Samples"，但它**累加的是帧数**（注意 `pushChunk` 里除以了 channels）。这是代码里一个容易读错的地方，我们到第七章会专门提醒。

还有另一种布局叫**平面（planar）**布局——先把所有 L 存完再存所有 R（`L0 L1 L2 L3 R0 R1 R2 R3`）。FFmpeg 内部很多地方用 planar。我们项目用的是交错，这也是 PCM 和 `AudioChunk` 的约定。

### 1.5 float32 与 PCM16：同一个声音的两种记法

这是今天代码里真正要动手处理的东西。

**PCM16（也叫 S16、int16）**：用 -32768 到 32767 之间的整数表示波形高度。0 就是静音（波形在中线），32767 是最大正振幅。-32768 是最大负振幅。

**float32（也叫 FLT、IEEE float）**：用 -1.0 到 1.0 之间的浮点数表示。0.0 是静音，1.0 是满幅（最大不失真振幅），-1.0 是负满幅。超过 ±1.0 就是"过载"，播放时会被削顶，也就是我们说的**爆音（clipping）**。

两者怎么换算？

```
float → int16:   i = round(f × 32767)      （先把 f 限制在 [-1, 1]）
int16 → float:   f = i × (1 / 32768)
```

注意两个方向用的系数**不一样**（32767 和 32768），这不是笔误：

- 转成 int16 时用 32767，是为了保证 +1.0 映射到的 32767 不会溢出成 -32768（因为 int16 的正数最大只到 32767，负数却能到 -32768，是不对称的）；
- 转成 float 时用 32768，是为了让 int16 的最小值 -32768 映射到 -1.0，恰好落在区间端点上。

**为什么音频处理喜欢用 float32？** 三个理由：

1. **信号处理方便**：调音量就是乘个系数，混音就是相加，不用考虑整数溢出；
2. **中间计算不损失精度**：多次滤波、混音时，整数格式每次都要重新量化，误差会累积；
3. **WASAPI 在 Windows 上默认给的就是 float32**——我们今天的 `GetMixFormat` 拿到的通常就是 32 位浮点。

所以项目第 3 讲把 `AudioChunk::samples` 定成 `std::vector<float>` 是非常正确的选择：**内部统一用 float32，到需要落盘或编码时再转换**。今天的代码就是这条流水线上的第一道工序——把 WASAPI 给的 float32 或 int16，**统一成 float32**。

### 1.6 WASAPI 是什么

**WASAPI** 是 **Windows Audio Session API** 的缩写，Windows Vista 时代引入，用来取代老的 WaveIn/WaveOut 和 DirectSound。它是 Windows 上**最底层、延迟最低、也最受推荐**的音频接口。OBS、Discord、各种专业音频软件在 Windows 上抓声音，用的都是它。

WASAPI 的核心对象只有四个，我们今天全都会用到，先把它们的关系记住：

```
┌───────────────────────────────────────────────────────┐
│ IMMDeviceEnumerator   设备枚举器                       │
│   "这台机器上有哪些声卡/麦克风？"                       │
│             │ EnumAudioEndpoints / GetDefaultAudio...  │
│             ▼                                          │
│ IMMDevice              一个音频端点设备                 │
│   "就是这块声卡/这个麦克风"                             │
│             │ Activate                                 │
│             ▼                                          │
│ IAudioClient            音频会话客户端（流的控制器）      │
│   "我要开始/停止采集，缓冲区多大，什么格式"              │
│             │ GetService                               │
│             ▼                                          │
│ IAudioCaptureClient     采集客户端（取数据的手）         │
│   "给我下一包数据"                                      │
└───────────────────────────────────────────────────────┘
```

- **`IMMDeviceEnumerator`**：设备枚举器。问它"有哪些设备"，它给你一个 `IMMDeviceCollection` 集合。前缀 `IMM` 来自 "Multimedia"，`MMDevice` 就是"多媒体设备"。
- **`IMMDevice`**：一个具体的音频端点（endpoint），比如"扬声器 (Realtek)"或"麦克风 (USB Audio)"。注意它**不是**能直接读写数据的东西，它只是一个"身份"。要拿数据得 `Activate`（激活）出别的接口。
- **`IAudioClient`**：流控制器。负责"初始化流"（协商格式、缓冲区大小、共享/独占模式）、`Start()`、`Stop()`、以及通过 `GetService` 派生出采集/渲染客户端。
- **`IAudioCaptureClient`**：真正取数据的手。`GetNextPacketSize` 问"下一包多大"，`GetBuffer` 拿指针，`ReleaseBuffer` 还回去。

这四个对象的获取顺序，就是我们 `open()` 函数的主干。同学们先记住这张图，读代码时就不迷路了。

补充一个名词：**端点（endpoint）**。WASAPI 里不叫"声卡"而叫"音频端点设备"，因为现代系统里一个物理声卡可以暴露多个逻辑端点（比如"扬声器"和"光纤输出"）。端点 = 一个可以被打开来播放或录音的逻辑设备。

### 1.7 共享模式 vs 独占模式

`IAudioClient::Initialize` 的第一个参数要选模式：

**共享模式（shared mode，本项目用这个）**：

- 你和系统上**所有其他程序共用**这个音频设备；
- 数据经过 Windows 的**音频引擎（Audio Engine）**——也就是混音器，它把各个程序的声音混在一起，再统一送给声卡；
- 格式由音频引擎说了算（就是你 `GetMixFormat` 拿到的那个格式），**你不能指定格式**，只能按引擎的格式来；
- 好处：**不独占设备**，别的程序照样能出声；
- 代价：延迟稍高（因为多走了一层混音），且格式固定。

**独占模式（exclusive mode）**：

- 你一个人霸占设备，别的程序**全部静音**；
- 你可以指定格式（比如强制 192kHz/24bit），数据**绕过混音器**直达声卡，延迟极低；
- 适合专业音频制作、ASIO 那种场景；
- 代价：一开就把别人踢下线，用户体验很差。

对直播推流来说，**共享模式是唯一合理的选择**——你总不能一开播，观众就听不到主播的游戏声音了吧？而且共享模式还有个意外的好处：**正因为数据经过混音器，我们才能实现"抓系统声音"**（见 1.8 节）。

代码里这个选择就是这一行：

```cpp
impl_->audioClient->Initialize(
    AUDCLNT_SHAREMODE_SHARED, streamFlags, kBufferDuration, 0, impl_->mixFormat,
    nullptr);
```

### 1.8 系统声音回环（loopback）vs 麦克风

这是直播场景最关心的一个选择：**我到底要采"电脑里放出来的声音"（游戏声、BGM、视频声音）还是"麦克风的声音"（人声）？**

答案通常是**两个都要**（主播讲话 + 游戏声音）。但今天我们只能二选一，因为一次 `open()` 只能开一个流。先看两种模式在 API 层怎么区分。

**数据流向（EDataFlow）** 是个枚举：

```cpp
const EDataFlow flow =
    (type == AudioSourceType::SystemAudio) ? eRender : eCapture;
```

- **`eRender`（渲染端）**：用来**播放**声音的设备，也就是扬声器、耳机。选这一类就是"我要抓系统声音"。
- **`eCapture`（采集端）**：用来**录入**声音的设备，也就是麦克风、线路输入。选这一类就是"我要抓麦克风"。

这里同学们一定会觉得别扭：**我要"采集"系统声音，为什么要去开一个"播放"设备？**

答案就在于那个神奇的 **loopback（回环）标志**：

```cpp
DWORD streamFlags = AUDCLNT_STREAMFLAGS_EVENTCALLBACK;
if (impl_->loopback)
    streamFlags |= AUDCLNT_STREAMFLAGS_LOOPBACK;
```

原理是这样的：在共享模式下，所有程序的声音都要经过音频引擎混音，混完之后这份"混合好的总输出"会被送往下两个地方——一是声卡（让你听见），二是一个**特殊的回环分路**。你在**渲染端点（扬声器）**上打开一个采集流，并加上 `AUDCLNT_STREAMFLAGS_LOOPBACK`，WASAPI 就把"即将送给扬声器播出来的那份混音结果"**复制一份给你**。

打个比方：扬声器的线本来是从"混音器"接到"喇叭"上的。loopback 相当于在这根线上**接了个三通**，分一路信号给你。你不碰喇叭，也不影响喇叭，只是"偷听"了它要播的内容。

所以两个模式完整对比：

| | 系统声音（loopback） | 麦克风 |
|---|---|---|
| 项目枚举 | `AudioSourceType::SystemAudio` | `AudioSourceType::Microphone` |
| 数据流向 | `eRender`（渲染端点） | `eCapture`（采集端点） |
| 开的设备 | 扬声器 / 耳机 | 麦克风 / 线路输入 |
| 额外标志 | `AUDCLNT_STREAMFLAGS_LOOPBACK` | 无 |
| 抓到什么 | 系统正在播放的一切声音的混音 | 麦克风拾到的空气振动 |
| 系统静音时 | **一个包都不给**（关键！） | 照常给包（只是内容是静音） |
| 典型用途 | 游戏声、BGM、视频伴音 | 主播人声 |

最后两行的差异，就是 1.14 节要讲的"静音补帧"存在的理由。

还有两个枚举参数顺便解释一下：

- **`DEVICE_STATE_ACTIVE`**：只要"当前可用"的设备。设备还有 `DEVICE_STATE_DISABLED`（被禁用）、`DEVICE_STATE_UNPLUGGED`（拔掉了）、`DEVICE_STATE_NOTPRESENT` 等状态，枚举时我们只关心能用的。
- **`eConsole`（控制台角色）**：设备可以扮演不同"角色"——`eConsole`（游戏/桌面应用）、`eCommunications`（语音通话，比如 QQ 语音）、`eMultimedia`（影音播放）。Windows 允许用户给不同角色配不同的默认设备。我们取 `eConsole`，就是"用户在声音设置里看到的那个默认设备"。

### 1.9 音频缓冲区与 packet：数据是怎么"一包一包"给我们的

音频是连续流，但系统不可能一个采样点通知你一次（48000 次/秒的回调会把 CPU 打死）。所以 WASAPI 的做法是：**攒成一包（packet）再给你**。

流程是标准三步，请大家记牢，因为它就是 `workerLoop` 里的核心循环：

```
① GetNextPacketSize(&frames)   →  问：下一包有多少帧？
         │
         │ frames == 0 表示暂时没数据，等下次
         ▼
② GetBuffer(&data, &frames, &flags, ...)  →  拿：数据指针 + 帧数 + 标志位
         │
         │ 现在 data 指向这块缓冲区，你可以读了
         ▼
③ ReleaseBuffer(frames)        →  还：告诉系统这块我用完了，可以回收
```

三个要点：

1. **必须成对**：`GetBuffer` 之后一定要 `ReleaseBuffer`，否则缓冲区不归还，采集会卡死（系统没地方写新数据了）。这和我们在 DirectShow 里学到的"回调里的指针寿命很短"是同一类规矩。
2. **`frames` 是帧数不是字节数**。要算字节数得 `frames × 声道数 × 每采样字节数`。今天的 `sampleCount` 算的是 `frames × channels`（采样数），因为我们要转成 float，一个采样就是一个 float。
3. **`flags` 里有个重要标志**：`AUDCLNT_BUFFERFLAGS_SILENT`，意思是"这一包其实是静音，数据内容不可信，请当全零处理"。系统用它来告诉你"这段时间确实没声音"。

顺便说明**缓冲区时长**这个概念。`Initialize` 的第三个参数是"缓冲区持续时间"，单位是 **100 纳秒**（这个单位叫 REFERENCE_TIME，是 Windows 多媒体的祖传单位，1 秒 = 10,000,000 个单位）。我们传的是：

```cpp
constexpr REFERENCE_TIME kBufferDuration = 1000000;   // 1,000,000 × 100ns = 0.1 秒 = 100 毫秒
```

意思是"请给我准备一个能装 100 毫秒音频的缓冲区"。缓冲区越大越不容易卡（抗抖动能力强），但延迟越高。100 毫秒对采集来说是相当保守（安全）的选择。

### 1.10 事件回调 vs 轮询：怎么知道"有新数据了"

工作线程怎么知道该去取数据了？两种办法：

**方式一：事件回调（本项目优先用）**

初始化时加 `AUDCLNT_STREAMFLAGS_EVENTCALLBACK` 标志，然后 `SetEventHandle(event)` 把一个 Windows 事件对象的句柄交给 WASAPI。**每当缓冲区里有一包新数据准备好了，WASAPI 就把这个事件置为"有信号"**。我们的线程只要 `WaitForSingleObject(event, ...)` 睡着等它就行——不占 CPU，响应及时。

**方式二：轮询（降级方案）**

不支持事件回调时（某些机器的 loopback 就是这样），只能定时去问：`WaitForSingleObject(event, 10)` —— 注意这行代码的妙处：**事件句柄仍然存在，但没人会置位它，于是这个等待必定超时，10 毫秒后返回**，我们就顺便去查一次缓冲区。一次调用同时实现了"睡 10 毫秒"和"等事件"两件事，非常巧妙。

代码里那个降级逻辑就是这么写的：

```cpp
if (FAILED(hr)) {
    if (impl_->loopback) {
        streamFlags &= ~AUDCLNT_STREAMFLAGS_EVENTCALLBACK;   // 去掉事件标志
        hr = impl_->audioClient->Initialize(..., streamFlags, ...);  // 重试
        if (SUCCEEDED(hr))
            say("这台机器的 loopback 不支持事件回调，改用 10ms 轮询");
    }
    if (FAILED(hr))
        return fail("初始化音频客户端失败");
}
```

这就是我们第 5 讲见过的"**降级重试**"策略——先提理想条件，谈不拢再让步。硬件编程里这个套路会一直出现。

顺便解释一下事件对象的创建：

```cpp
impl_->event = CreateEventW(nullptr, FALSE, FALSE, nullptr);
```

四个参数依次是：安全属性（不用）、**是否手动复位**（FALSE = 自动复位，等到了就自动变回无信号，正是我们要的）、**初始状态**（FALSE = 无信号）、名字（匿名）。

### 1.11 重采样：为什么今天"没做"也要知道

**重采样（resampling）** 就是把音频从一个采样率换成另一个采样率，比如 44100 → 48000。

为什么需要它？因为**上游设备的格式和下游编码器的要求经常不一致**：

- 你机器的混音格式可能是 44100 Hz（很多笔记本声卡默认是这个）；
- 但我们的 `StreamConfig` 里写死了 `sampleRate = 48000`；
- AAC 编码器通常也要求特定的采样率。

如果直接把 44100 的数据当成 48000 播，声音会**变快变尖**（每秒多放了 3900 帧）；反过来会变慢变闷。所以必须转换。

重采样不是简单地"插几个点"——按 1.2 节的采样定理，**降采样前必须先做低通滤波**（抗混叠），否则高频会混叠成噪声。工业级实现用的是多相滤波器或 FFT 算法，FFmpeg 里的 `swr_resample`（libswresample）就是干这个的。

**今天这个 commit 没有做重采样。** 我们只是**照单全收** `GetMixFormat` 给的格式，然后把这个真实的采样率写进每一个 `AudioChunk`：

```cpp
chunk->sampleRate = sampleRate;   // 真实值，可能是 44100 也可能是 48000
```

这正是第 3 讲设计 `AudioChunk` 时说的那句话的兑现——**"块随身带着自己的真实格式，下游的重采样器据此工作"**。采集层不撒谎：我采到多少就报多少，转换是下游的事。这个"如实上报、延迟处理"的分层思想，和摄像头那边 `CameraRawFrame` 忠实保存原始格式是**完全一致的设计哲学**。

同学们把这条记下来，它是本项目的核心分层原则之一。

### 1.12 COM 在音频里的用法

第二讲 COM 了，我们快速回顾 + 补充音频特有的部分。

COM 的三条基本功，我们在第 5 讲 DirectShow 里已经见过两条（虚表布局、GUID），这里再强调音频场景的三个具体用法：

**用法一：`CoCreateInstance` 创建对象，用 `__uuidof` 取 GUID**

```cpp
CoCreateInstance(__uuidof(MMDeviceEnumerator), nullptr, CLSCTX_ALL,
                 IID_PPV_ARGS(&enumerator));
```

`__uuidof(MMDeviceEnumerator)` 是 MSVC 的扩展，直接从一个类型上取它的 GUID 属性——比手写 GUID 常量方便多了（前提是头文件里声明时带上了 `__declspec(uuid(...))`，SDK 里都带了）。

注意第二个到第四个参数：

- `nullptr`：不从某个聚合对象创建（不用管）；
- `CLSCTX_ALL`：允许进程内、进程外、远程服务器，随便哪种，你看着办（音频 API 都是系统组件，实际都是进程外或进程内服务）；
- `IID_PPV_ARGS(&p)`：宏，自动推导 IID 和二级指针类型，防止手抄错。

**用法二：`Activate` 代替 `CoCreateInstance`**

音频设备不是"凭空创建"的，而是"在某个设备上激活一个接口"：

```cpp
impl_->device->Activate(__uuidof(IAudioClient), CLSCTX_ALL, nullptr,
                        reinterpret_cast<void**>(&impl_->audioClient));
```

`Activate` 和 `CoCreateInstance` 的区别：`CoCreateInstance` 是"给我造一个新的"，`Activate` 是"**在这个已存在的对象上，给我一张新的面孔**"。同一个 `IMMDevice`（扬声器），你可以激活 `IAudioClient`（音频流）、也可以激活 `IAudioEndpointVolume`（音量控制）、`IPropertyStore`（属性）——都是同一个设备的不同接口。这就是 COM 的"一个对象多张面孔"。

这里因为 `Activate` 的最后一个参数是 `void**`，没法用 `IID_PPV_ARGS`（那个宏要类型推导），所以只能手写 `reinterpret_cast<void**>(&ptr)`。

**用法三：COM 内存用 `CoTaskMemFree` 释放**

```cpp
if (impl_->mixFormat) {
    CoTaskMemFree(impl_->mixFormat);
    impl_->mixFormat = nullptr;
}
```

`GetMixFormat` 返回的 `WAVEFORMATEX*` 是系统用 **COM 任务分配器**分配的内存。规矩是：谁用什么分配器分配，就用什么分配器释放。COM 任务分配器分配的内存必须用 `CoTaskMemFree`，**不能用 `delete` 或 `free`**——分配器不匹配会导致堆损坏，而且这种崩溃往往发生在很久以后，极难排查。

第 5 讲的 `freeMediaType` 里用的是同一套规矩，同学们可以对照着复习。

**用法四：线程相关的 COM 初始化**

这一条今天特别重要。COM 初始化是**按线程**的——你在 A 线程 `CoInitializeEx` 了，B 线程没初始化就用 COM 对象，一样会失败。

而我们今天的函数分布在不同的线程上：

| 函数 | 跑在哪个线程 |
|---|---|
| `listDevices()` | 调用方线程，用 `ComScope` 包住 |
| `open()` | 调用方线程，手动 `CoInitializeEx` |
| `close()` | 调用方线程，手动 `CoUninitialize` |
| `workerLoop()` | **我们新开的工作线程**，用 `ComScope` 包住 |

注意最后一行：**工作线程必须自己初始化 COM**，不能蹭主线程的。这就是 `workerLoop()` 第一行为什么是 `ComScope com;` 的原因。这是新手写多线程 COM 代码时最常见的崩溃来源。

### 1.13 AudioChunk 契约回顾

第 3 讲定的契约，今天我们第一次真正用它。再贴一遍：

```cpp
struct AudioChunk {
    int sampleRate = 0;     // 这块数据的采样率（Hz）
    int channels = 0;       // 声道数
    int64_t pts = 0;        // 时间戳
    std::vector<float> samples;   // 交错存放的 float32 采样
};

using AudioChunkPtr = std::shared_ptr<AudioChunk>;
```

四条约定，今天我们逐条兑现：

1. **`sampleRate` / `channels` 必须如实填写**——不填 48000 假装，而是填 WASAPI 实际给的值。
2. **`samples` 是 float32，交错布局**——所以 WASAPI 给 int16 时，我们必须转换（1.5 节的公式）。
3. **`pts` 是时间戳**——今天我们的单位是"**采样帧序号**"（从 0 开始，每帧 +1），不是毫秒。这个单位选择和视频侧的 pts 是否需要统一，是坑点环节要讨论的问题。
4. **用 `shared_ptr` 传递**——零拷贝，队列里存指针。

顺带回忆一下 `BoundedQueue`（第 3 讲）：

```cpp
BoundedQueue<AudioChunkPtr> queue{64};
```

- `push()`：**满了就丢最旧的**（`pop_front` 再 `push_back`）——保新不保旧；
- `tryPop(timeoutMs)`：等 `timeoutMs` 毫秒，超时返回 `std::nullopt`；
- `clear()`：清空。

注意一个矛盾点，同学们先思考一下：`push` 满了会丢最旧的块，而我们开头刚说过"音频一块都不能丢"。这两个是冲突的。答案在坑点环节揭晓（提示：丢的是"队列里的积压"，不是"时间"——时间账由静音补帧机制单独维护）。

### 1.14 为什么必须"静音补帧"

这是今天最后一个预习点，也是本模块最有特色的一段逻辑。

复习 1.8 节那张表的倒数第二行：**系统声音（loopback）在系统没播放任何东西时，一个包都不给**。

想象这个场景：主播开了直播，正在打字，电脑没放任何声音。这时候：

- 视频：照常 30fps 出帧，时间轴在走；
- 音频：WASAPI 沉默，一个包都没有，时间轴**完全静止**；
- 结果：两秒后，声音比画面落后两秒。等主播一开口，观众听到的是两秒前的口型——**彻底崩了**。

所以必须补。补的办法很朴素：**拿一个高精度时钟盯着"按采样率算，现在本该采到多少帧了"，发现实际不够，就自己造一块全零（静音）的块补进去**。

代码里的三个参数就是这套机制的调校旋钮：

```cpp
constexpr DWORD kPollIntervalMs = 10;    // 轮询间隔（也是超时等待时长）
constexpr REFERENCE_TIME kBufferDuration = 1000000;  // 缓冲区 100 毫秒
constexpr int kMinSilenceMs = 10;        // 缺口小于 10 毫秒就不补（避免抖动误判）
constexpr int kMaxSilenceMs = 200;       // 一次最多补 200 毫秒（防止一次补过头）
```

- **下限 10 毫秒**：因为每次轮询本来就有 10 毫秒的粒度，一两个采样点的误差是正常抖动，不值得为它专门造一块——**太敏感会导致疯狂补帧**；
- **上限 200 毫秒**：假设程序卡了一下（比如 UI 线程阻塞了 2 秒），如果不设上限，会一次性补 2 秒的静音，队列瞬间爆满。**补帧是补"正常的小缺口"，不是补"程序卡死"**——卡死那段时间就该丢掉，补回来反而更糟。

这个思路在实时音视频领域有个专门的名字，叫**时钟对齐（clock alignment）**或**时间轴保底**。同学们以后做任何实时流处理，都会遇到同样的问题。

预习完毕，一共十四条。下面正式读代码。

---

## 第二章：头文件契约 —— `AudioCapturer.h`

先看对外契约。31 行，非常干净。

```cpp
#pragma once

#include "core/common/Common.h"

#include <memory>
#include <string>
#include <vector>

class AudioCapturer {
public:
    AudioCapturer();
    ~AudioCapturer();

    AudioCapturer(const AudioCapturer&) = delete;
    AudioCapturer& operator=(const AudioCapturer&) = delete;

    static std::vector<std::string> listDevices(AudioSourceType type);

    bool open(AudioSourceType type, int deviceIndex, LogCallback log);

    void close();

    AudioChunkPtr nextChunk(int timeoutMs);

    int sampleRate() const;
    int channels() const;

private:
    struct Impl;
    std::unique_ptr<Impl> impl_;
};
```

### 2.1 和 CameraCapturer 的接口对照

同学们把两个采集器放一起看，会发现它们是**刻意做成对称的**：

| | `CameraCapturer` | `AudioCapturer` |
|---|---|---|
| 枚举设备 | `listDevices()` | `listDevices(AudioSourceType)` |
| 打开 | `open(index, log, requestFormat)` | `open(type, index, log)` |
| 取一份 | `nextFrame(timeoutMs)` | `nextChunk(timeoutMs)` |
| 关闭 | `close()` | `close()` |
| 查询参数 | `width() / height() / pixelFormat()` | `sampleRate() / channels()` |
| 实现隐藏 | `struct Impl` + `unique_ptr` | 同 |
| 拷贝语义 | 禁止 | 禁止 |

几个差异值得说道：

**差异一：`listDevices` 带参数。** 音频要区分"列举扬声器（用来抓系统声音）"还是"列举麦克风"，所以必须传入 `AudioSourceType`。视频只有一种设备类型，不用传。

**差异二：`open` 没有"请求格式"参数。** 摄像头可以请求 RGB24 还是 MJPG，音频**不能**——因为在共享模式下，格式由 Windows 音频引擎说了算，你只能接受（1.7 节）。所以 `open` 的签名里没有格式参数，格式是"问出来的"而不是"指定的"。

这也带来一个 API 使用上的注意点：**你要先 `open()` 成功，才能用 `sampleRate()` / `channels()` 问到实际格式**。

**差异三：返回值是 `AudioChunkPtr` 而不是裸数据。** 因为音频块要跨线程（工作线程 → 消费线程），用 `shared_ptr` 传递零拷贝、生命周期自动。

### 2.2 PIMPL 的意义在这里特别大

为什么音频类的实现必须藏进 `Impl`？因为它要 include 一大堆 Windows 头文件：

```cpp
#include <windows.h>
#include <audioclient.h>
#include <mmdeviceapi.h>
#include <propsys.h>
#include <propkeydef.h>
#include <functiondiscoverykeys_devpkey.h>
```

这些头文件会往全局命名空间里塞进成百上千个宏和类型名（`PlaySound`、`CreateEvent`、`GetMessage`…… 随便一个都可能和别的地方撞车）。如果它们出现在 `AudioCapturer.h` 里，**任何 include 了这个头文件的 .cpp 都会被污染**。

PIMPL（Pointer to Implementation，"指向实现的指针"）把这些全关在 `.cpp` 里，对外只暴露一个不透明的 `struct Impl;` 声明。这是 C++ 项目隔离平台依赖的标准手段。

我们这个项目的三个采集/转换类全都用了 PIMP，同学们写新模块时请照抄这个习惯。

---

## 第三章：工具函数群 —— 匿名命名空间里的螺丝钉

`AudioCapturer.cpp` 开头是一个匿名命名空间（`namespace { ... }`），里面装着本模块的全部"基础设施"。匿名命名空间的成员只在当前编译单元可见——**不会污染全局符号表**，是 C++ 里替代 `static` 的现代写法。

我们逐个看。

### 3.1 safeRelease 与 toUtf8

```cpp
template <typename T>
void safeRelease(T*& p)
{
    if (p) {
        p->Release();
        p = nullptr;
    }
}
```

和第 5 讲那个一模一样（每个用到 COM 的模块都会重新写一遍，因为它太小太专用，跨模块共享反而麻烦）。三步：判空 → `Release()` → 置空。`close()` 里要连用四次。

```cpp
std::string toUtf8(const wchar_t* wide)
{
    if (!wide || !*wide)
        return {};

    const int size =
        WideCharToMultiByte(CP_UTF8, 0, wide, -1, nullptr, 0, nullptr, nullptr);
    if (size <= 1)
        return {};

    std::string out(static_cast<size_t>(size), '\0');
    WideCharToMultiByte(CP_UTF8, 0, wide, -1, out.data(), size, nullptr, nullptr);
    out.resize(static_cast<size_t>(size) - 1);
    return out;
}
```

Windows API 用 UTF-16 宽字符，项目内部用 UTF-8 的 `std::string`，所以需要这个转换。

注意它是**两段式**调用（第 5 讲讲过）：第一次输出缓冲传 `nullptr`，只为问"需要多少字节"；拿到数字开好缓冲，第二次才真转换。返回的 `size` 含末尾空终止符，所以最后 `resize(size - 1)` 去掉。

这个函数在音频模块里用得比视频模块还频繁——因为**设备名**（"扬声器 (Realtek High Definition Audio)"）全是宽字符串。

### 3.2 ComScope：带 owned() 查询的 COM RAII 包装

```cpp
class ComScope {
public:
    ComScope()
    {
        const HRESULT hr = CoInitializeEx(nullptr, COINIT_MULTITHREADED);
        owned_ = (hr == S_OK || hr == S_FALSE);
    }

    ~ComScope()
    {
        if (owned_)
            CoUninitialize();
    }

    ComScope(const ComScope&) = delete;
    ComScope& operator=(const ComScope&) = delete;

    bool owned() const { return owned_; }

private:
    bool owned_ = false;
};
```

和第 5 讲那个 `ComScope` 几乎一样，只多了一个 `owned()` 查询方法。

RAII（Resource Acquisition Is Initialization，"资源获取即初始化"）——构造时获取，析构时自动释放，靠栈对象的确定析构期保证不泄漏。这是 C++ 处理资源最地道的写法。

`owned_` 标志体现的是"**谁初始化谁负责**"的礼仪：只有当我们真的初始化成功了（`S_OK` = 首次成功，`S_FALSE` = 同线程重复调用同模式），析构时才 `CoUninitialize()`。如果线程早就被别人初始化成 STA（返回 `RPC_E_CHANGED_MODE`），我们就不碰别人的东西。

**本模块用它三处**：`listDevices()`、`workerLoop()`。而 `open()` 因为 COM 的生命周期要跨越函数边界（一直活到 `close()`），所以用不了 RAII，只能手动管理 `comOwned` 标志。这个"作用域内用 RAII、跨作用域用手动标志"的搭配，是 COM 编程的标准实践。

顺便说明：`COINIT_MULTITHREADED` 表示 MTA（多线程套间），任何线程都能直接调用 COM 对象，COM 不帮你做线程同步。采集类代码通常选它（第 5 讲 1.6 节讲过 STA/MTA 的区别）。

### 3.3 friendlyNameOf：从设备属性包里读名字

```cpp
std::string friendlyNameOf(IMMDevice* device)
{
    IPropertyStore* store = nullptr;
    if (!device || FAILED(device->OpenPropertyStore(STGM_READ, &store)))
        return {};

    std::string name;
    PROPVARIANT value;
    PropVariantInit(&value);
    if (SUCCEEDED(store->GetValue(PKEY_Device_FriendlyName, &value))
        && value.vt == VT_LPWSTR && value.pwszVal) {
        name = toUtf8(value.pwszVal);
    }
    PropVariantClear(&value);
    store->Release();
    return name;
}
```

这段是 DirectShow 那套 `IPropertyBag + VARIANT` 的**音频版**，但用的是新一代的物件：

| DirectShow（第 5 讲） | WASAPI（今天） |
|---|---|
| `IPropertyBag` | `IPropertyStore` |
| `VARIANT` | `PROPVARIANT` |
| `VariantInit` / `VariantClear` | `PropVariantInit` / `PropVariantClear` |
| `bag->Read(L"FriendlyName")` | `store->GetValue(PKEY_Device_FriendlyName)` |
| `value.bstrVal`（BSTR） | `value.pwszVal`（`LPWSTR`） |
| 用名字字符串查属性 | 用 `PKEY` 常量查属性 |

逐个解释：

- **`OpenPropertyStore(STGM_READ, &store)`**：打开设备的属性仓库，`STGM_READ` 表示只读（`STGM` = Storage Mode）。
- **`PROPVARIANT`**：和 `VARIANT` 一样是"带类型标签的万能值容器"，`vt` 字段说明里面装的是什么。这里期望 `VT_LPWSTR`（宽字符串指针）。
- **`PKEY_Device_FriendlyName`**：属性的"键"。这是 Windows Vista 引入的**属性系统（Property System）**，每个属性有一个标准键。这个键定义在 `functiondiscoverykeys_devpkey.h` 里，`PKEY` 宏本身定义在 `propkeydef.h`——这就是 `.cpp` 开头 include 这两个头文件的原因。同时也需要链接 `propsys.lib`（`CMakeLists.txt` 里加的）。
- **三重检查**：`SUCCEEDED(...)` && `vt == VT_LPWSTR` && `pwszVal` 非空。少任何一条都可能读到垃圾。
- **`PropVariantClear` 无条件调用**：不管读没读成功都要清理（里面可能有堆内存）。

任何一步失败就返回空串——**容忍无名设备**，不因为一个设备读不出名字就让整个枚举失败。这个"降级不崩"的态度和第 5 讲一脉相承。

### 3.4 两个手写 GUID 常量：KSDATAFORMAT_SUBTYPE

```cpp
const GUID kSubtypeIeeeFloat = {
    0x00000003, 0x0000, 0x0010, {0x80, 0x00, 0x00, 0xAA, 0x00, 0x38, 0x9B, 0x71}};
const GUID kSubtypePcm = {
    0x00000001, 0x0000, 0x0010, {0x80, 0x00, 0x00, 0xAA, 0x00, 0x38, 0x9B, 0x71}};
```

同学们看到这两个 GUID 应该会会心一笑——**又出现手写的 GUID 常量了**，和第 5 讲手写 Sample Grabber 的 CLSID 是同一个套路。

它们是 `WAVEFORMATEXTENSIBLE` 里的 **SubFormat（子格式）** GUID。看这两个 GUID 的结构：

```
{ 00000003-0000-0010-8000-00AA00389B71 }   ← IEEE float
{ 00000001-0000-0010-8000-00AA00389B71 }   ← PCM
  ↑        ↑    ↑    └── 固定的 8 字节后缀
  │        │    └── 固定
  │        └── 固定
  └── 只有第一个字段不同！
```

它们共享同一套模板，只有开头那个数字不同——这是 Windows 定义**音频子格式**的惯例（`KSDATAFORMAT_SUBTYPE_*`）。PCM = 1，IEEE_FLOAT = 3。

**为什么这里要手写，而不像前面用 `__uuidof`？** 因为这两个是**纯数据常量**（GUID 值），不是绑定到某个 C++ 类型上的 uuid 属性。`__uuidof` 只能用在带 `__declspec(uuid(...))` 声明的类型上，对裸 GUID 值无效。SDK 里其实也有现成的 `KSDATAFORMAT_SUBTYPE_IEEE_FLOAT` 常量，但那个定义在 `ks.h`/`ksmedia.h` 里，引起来会带进一堆内核流媒体（Kernel Streaming）的东西，作者选择手写这两行，是为了**少引一个重量级头文件**。这个取舍是合理的。

### 3.5 SampleFormat 枚举

```cpp
enum class SampleFormat {
    Float32,
    Int16,
    Unsupported,
};
```

模块内部的私有枚举（在匿名命名空间里，外部看不见），用来描述"混音格式到底是哪一种样本类型"。三种取值对应 1.5 节讲的两种格式 + 一个"不认识"。

注意它**是模块内部的**——对外暴露的是 `AudioChunk::samples`（永远是 float）。这个"内部可以有多种、对外只有一种"的设计，正是"统一格式"这个 commit 目标的体现。

### 3.6 detectSampleFormat：WAVEFORMATEX 格式侦探

这是本模块最有技术含量的一段格式判断，我们单独开一章讲（第四章）。

### 3.7 convertSamples：统一成 float32

把 WASAPI 给的任意格式转成 float32，是"统一格式"这个目标的直接执行者。第五章专门讲。

### 3.8 四个调校常量

```cpp
constexpr DWORD kPollIntervalMs = 10;

constexpr REFERENCE_TIME kBufferDuration = 1000000;

constexpr int kMinSilenceMs = 10;

constexpr int kMaxSilenceMs = 200;
```

1.14 节已经详细解释过了，这里补充两点：

1. **`constexpr`**：编译期常量，不占运行时存储，还能参与编译期计算。用它代替 `#define` 是现代 C++ 的推荐做法（有类型、有作用域、能调试）。
2. **`kPollIntervalMs` 身兼两职**：既是"轮询模式下每轮睡多久"，也是"事件模式下 `WaitForSingleObject` 的超时时间"。事件模式下超时只是为了兜底（万一事件句柄设置失败，我们还能靠超时醒过来干活），正常情况事件一到就立刻返回。

---

## 第四章：detectSampleFormat —— 三种 WAVEFORMATEX 变体的侦探工作

格式判断是今天最容易出错的地方，因为它要同时对付**三种不同的结构体变体**。我们仔细看。

```cpp
SampleFormat detectSampleFormat(const WAVEFORMATEX* format)
{
    if (!format)
        return SampleFormat::Unsupported;

    if (format->wFormatTag == WAVE_FORMAT_IEEE_FLOAT)
        return format->wBitsPerSample == 32 ? SampleFormat::Float32
                                            : SampleFormat::Unsupported;

    if (format->wFormatTag == WAVE_FORMAT_PCM)
        return format->wBitsPerSample == 16 ? SampleFormat::Int16
                                            : SampleFormat::Unsupported;

    if (format->wFormatTag == WAVE_FORMAT_EXTENSIBLE
        && format->cbSize >= sizeof(WAVEFORMATEXTENSIBLE) - sizeof(WAVEFORMATEX)) {
        const auto* ext = reinterpret_cast<const WAVEFORMATEXTENSIBLE*>(format);
        if (ext->SubFormat == kSubtypeIeeeFloat && format->wBitsPerSample == 32)
            return SampleFormat::Float32;
        if (ext->SubFormat == kSubtypePcm && format->wBitsPerSample == 16)
            return SampleFormat::Int16;
    }

    return SampleFormat::Unsupported;
}
```

### 4.1 背景：WAVEFORMATEX 是个"会变形"的结构体

要读懂这个函数，先要理解 Windows 描述音频格式的结构体族：

**`WAVEFORMATEX`（基础版）**：

```cpp
typedef struct {
    WORD  wFormatTag;       // 格式标签：PCM / IEEE_FLOAT / EXTENSIBLE / ...
    WORD  nChannels;        // 声道数
    DWORD nSamplesPerSec;   // 采样率
    DWORD nAvgBytesPerSec;  // 每秒字节数（= 采样率 × 块对齐）
    WORD  nBlockAlign;      // 块对齐（= 声道数 × 每采样字节数）
    WORD  wBitsPerSample;   // 位深
    WORD  cbSize;           // 后面还跟着多少字节的"扩展数据"
} WAVEFORMATEX;
```

关键字段是 `wFormatTag`（格式标签）和 `cbSize`（尾部扩展数据的字节数）。

**为什么要有 `cbSize`？** 因为微软的惯例是：**结构体的尾部可以额外追加数据**。`WAVEFORMATEX` 只是"头部"，后面可能还跟着一段变长的扩展块，`cbSize` 告诉你有多长。这种"头部 + 变长尾巴"的结构体在 Windows API 里到处都是。

**`WAVEFORMATEXTENSIBLE`（可扩展版）** 就是在 `WAVEFORMATEX` 后面追加了 22 字节：

```cpp
typedef struct {
    WAVEFORMATEX Format;        // 前面就是完整的 WAVEFORMATEX
    union {
        WORD wValidBitsPerSample;   // 有效位数
        WORD wSamplesPerBlock;
        WORD wReserved;
    } Samples;
    DWORD dwChannelMask;        // 声道位掩码（哪个声道对应哪个喇叭）
    GUID  SubFormat;            // 真正的格式 GUID
} WAVEFORMATEXTENSIBLE;
```

### 4.2 三条判断分支

**分支一：`WAVE_FORMAT_IEEE_FLOAT`（值为 3）**

最直白的情况：格式标签直接说"我是 IEEE 浮点"。但还要**再检查位深是不是 32 位**——因为浮点还有 64 位的（double），我们不处理：

```cpp
return format->wBitsPerSample == 32 ? SampleFormat::Float32
                                    : SampleFormat::Unsupported;
```

这个三元表达式的意思是"是 32 位就接，不是就拒绝"。**不能只看标签不看位深**——这是新手常犯的错。

**分支二：`WAVE_FORMAT_PCM`（值为 1）**

同样道理：标签说是 PCM，还得确认是 **16 位**：

```cpp
return format->wBitsPerSample == 16 ? SampleFormat::Int16
                                    : SampleFormat::Unsupported;
```

为什么只支持 16 位？因为 8 位音质太差、24/32 位整数太罕见，**16 位 PCM 是绝大多数麦克风和老设备的实际输出**。8 位或 24 位的设备碰到我们这里会返回 `Unsupported`，`open()` 会明确失败并打日志——**宁可明确失败，也不要默默用错误的公式解读数据**（用 int16 的公式读 24 位数据，出来的全是噪声）。

**分支三：`WAVE_FORMAT_EXTENSIBLE`（值为 0xFFFE）**

这是最麻烦的一种，也是**现代 Windows 上最常见的一种**。它的意思是："我用了扩展结构体，真正的格式请看尾部的 `SubFormat` GUID"。

关键在这个防御性检查：

```cpp
if (format->wFormatTag == WAVE_FORMAT_EXTENSIBLE
    && format->cbSize >= sizeof(WAVEFORMATEXTENSIBLE) - sizeof(WAVEFORMATEX)) {
```

`sizeof(WAVEFORMATEXTENSIBLE) - sizeof(WAVEFORMATEX)` = 22 —— 就是扩展部分（`Samples` + `dwChannelMask` + `SubFormat`）的大小。

**为什么必须先查 `cbSize` 再 `reinterpret_cast`？** 因为 `format` 只是个 `WAVEFORMATEX*`，我们并不知道它后面真的有 22 字节。如果某个驱动声称自己是 `WAVE_FORMAT_EXTENSIBLE` 但 `cbSize` 却小于 22（不合规矩的驱动是存在的），我们直接强转就会**读到缓冲区外面去**——典型的越界读。

这就是第 5 讲强调过的那条纪律：**`pbFormat` 是裸指针，里面是什么全靠"格式标签 + 长度"两个线索互相印证**。今天的代码完美遵守了这条纪律。

然后是两次比对：

```cpp
if (ext->SubFormat == kSubtypeIeeeFloat && format->wBitsPerSample == 32)
    return SampleFormat::Float32;
if (ext->SubFormat == kSubtypePcm && format->wBitsPerSample == 16)
    return SampleFormat::Int16;
```

注意这里是**双重确认**：`SubFormat` GUID 对了，位深也对了，才算数。

**一个实战提醒**：`WAVE_FORMAT_EXTENSIBLE` 在现代 Windows 上的实际行为是——`wFormatTag` 常常也填成 `WAVE_FORMAT_PCM`（1），只有在需要表达"超出老结构体能力"的信息（比如声道掩码）时才标 0xFFFE。所以三个分支都要有，缺一不可。

### 4.3 兜底

```cpp
return SampleFormat::Unsupported;
```

走过三个分支都没匹配上（比如是杜比数字、DTS、或者某种压缩格式），返回不支持。`open()` 会因此失败并给出明确提示。

**这里体现了一种重要的工程态度：宁可明确说"我不会"，也不要猜着处理。** 猜错的后果是下游拿到一堆噪声，而且要查很久才查得回来；明确失败的后果只是"这个功能用不了"，一分钟就能定位。

---

## 第五章：convertSamples —— 统一成 float32 的最后一道工序

```cpp
void convertSamples(const BYTE* src, size_t sampleCount, SampleFormat format,
                    std::vector<float>& out)
{
    out.resize(sampleCount);

    if (format == SampleFormat::Float32) {
        std::memcpy(out.data(), src, sampleCount * sizeof(float));
        return;
    }

    if (format == SampleFormat::Int16) {
        const auto* p = reinterpret_cast<const int16_t*>(src);
        constexpr float kScale = 1.0f / 32768.0f;
        for (size_t i = 0; i < sampleCount; ++i)
            out[i] = static_cast<float>(p[i]) * kScale;
    }
}
```

### 5.1 签名解读

四个参数：

- `src`：源数据指针（WASAPI 缓冲区，可能是 float 也可能是 int16）；
- `sampleCount`：**采样总数**（不是字节数，也不是帧数）= 帧数 × 声道数；
- `format`：源格式；
- `out`：输出，传引用（复用调用方的 vector，避免频繁分配）。

### 5.2 第一句：resize

```cpp
out.resize(sampleCount);
```

先把输出数组开到 `sampleCount` 个元素。`resize` 变大时，新元素会**值初始化**——对 `float` 来说就是 `0.0f`。

这一句有个额外的好处：**它是"兜底"**。看函数末尾——如果 `format` 是 `Unsupported`（理论上不该发生，因为 `open()` 已经拦住了），循环不会执行，`out` 里留着的就是一整块 0.0，也就是静音。**默认安全**，不会因为格式不认识而输出随机内存。

### 5.3 Float32 分支：纯内存拷贝

```cpp
if (format == SampleFormat::Float32) {
    std::memcpy(out.data(), src, sampleCount * sizeof(float));
    return;
}
```

源已经是 float32 了，**格式一模一样，直接整块内存拷过去**。`memcpy` 是最快的批量拷贝方式（编译器和 CPU 都会对它做优化，比手写循环快得多）。

这里能直接用 `memcpy` 的前提是：**float 在内存里的布局和 WASAPI 给的 float32 完全一致**——都是 IEEE 754 单精度、小端序。在 x86/x64（我们唯一的平台）上这是天然成立的。

注意 `sampleCount * sizeof(float)` 算的是字节数，这正是 `memcpy` 需要的单位。**用 `memcpy` 时永远传字节数，用循环时永远传元素个数**——这个单位差异是 C 语言最经典的错误来源之一。

### 5.4 Int16 分支：逐点乘系数

```cpp
if (format == SampleFormat::Int16) {
    const auto* p = reinterpret_cast<const int16_t*>(src);
    constexpr float kScale = 1.0f / 32768.0f;
    for (size_t i = 0; i < sampleCount; ++i)
        out[i] = static_cast<float>(p[i]) * kScale;
}
```

三步：

1. **强转指针**：把 `BYTE*`（`unsigned char*`）重新解释成 `int16_t*`。因为源缓冲区就是一串紧密排列的 16 位有符号整数。
2. **定系数**：`kScale = 1/32768`，就是 1.5 节讲的"int16 → float"公式。用 `constexpr` 声明，编译器会把它当编译期常量优化掉。
3. **逐点转换**：读一个 `int16_t`，转成 `float`，乘系数，写进 `out[i]`。

循环里的 `static_cast<float>(p[i])` 是**必须的显式转换**——先转成 float 再乘，否则 `p[i] * kScale` 会先被提升成 float 再乘（其实结果一样），但显式写出来意图更清楚，也让编译器不会报精度警告。

### 5.5 性能提示（拓展）

这个函数在实时音频路径上，**每秒要处理 48000 × 2 = 96000 个采样**。目前这个朴素循环对现代 CPU 来说毫无压力（几十微秒的事）。但如果将来要支持 7.1 声道 192kHz，可以考虑：

- 用 SIMD 指令集（SSE/AVX）一次处理 4/8 个采样；
- 或者建一张 65536 项的查找表（int16 的全部取值 → float），用空间换时间。

不过同学们记住优化的第一原则：**先测再优化**。当前这个循环不是瓶颈（真正的开销在后面 `pushChunk` 的内存分配和队列同步上）。

---

## 第六章：Impl 结构体 —— 状态全景

读核心函数之前，先把 `Impl` 里的状态过一遍。这样读每一行代码时，我们都知道它在操作哪个变量。

```cpp
struct AudioCapturer::Impl {
    IMMDeviceEnumerator* enumerator = nullptr;
    IMMDevice* device = nullptr;
    IAudioClient* audioClient = nullptr;
    IAudioCaptureClient* captureClient = nullptr;
    WAVEFORMATEX* mixFormat = nullptr;

    HANDLE event = nullptr;
    std::thread worker;

    std::atomic<bool> stopping{false};

    int sampleRate = 0;
    int channels = 0;
    SampleFormat sampleFormat = SampleFormat::Unsupported;
    bool loopback = false;

    int64_t totalSamples = 0;

    BoundedQueue<AudioChunkPtr> queue{64};

    bool comOwned = false;
    LogCallback log;

    void workerLoop();
    void pushChunk(const float* data, size_t sampleCount, bool silent);
    void padSilence(int64_t frames);
};
```

### 6.1 五组状态

**第一组：COM 对象（五个指针）**

按获取顺序排列，也就是 1.6 节那张图的顺序：

1. `enumerator`：`IMMDeviceEnumerator`，设备枚举器；
2. `device`：`IMMDevice`，具体设备；
3. `audioClient`：`IAudioClient`，流控制器（由 device `Activate` 得来）；
4. `captureClient`：`IAudioCaptureClient`，取数据的手（由 audioClient `GetService` 得来）；
5. `mixFormat`：`WAVEFORMATEX*`，混音格式描述（**不是 COM 对象，但是 COM 分配器给的内存**）。

注意最后一个 `mixFormat` 虽然名字不像 COM 对象，但它必须用 `CoTaskMemFree` 释放——所以 `close()` 里要单独处理它，不能用 `safeRelease`。

**第二组：线程与同步**

- `event`：`HANDLE`，Windows 事件对象句柄。用于事件回调模式和轮询超时。
- `worker`：`std::thread`，我们的工作线程。`workerLoop()` 跑在里面。
- `stopping`：`std::atomic<bool>`，停止标志。**必须是原子的**，因为它被主线程写（`close()` 置 true）、被工作线程读（`workerLoop()` 循环条件），是典型的跨线程标志。

**第三组：协商出来的格式**

- `sampleRate`：采样率（Hz）；
- `channels`：声道数；
- `sampleFormat`：样本格式（float32 / int16）；
- `loopback`：是不是回环模式（抓系统声音）。

这四个值在 `open()` 成功后填好，之后工作线程一直在用。**它们在整个采集期间不变**——这是安全的前提（如果设备格式中途变了，比如用户改了系统设置，我们没有处理，见坑点环节）。

**第四组：时间账本**

- `totalSamples`：`int64_t`，**已经产出多少"帧"**（注意名字有歧义，见下）。

这个变量是"静音补帧"机制的核心。它记录的是"到目前为止，我们交给队列的总帧数"。工作线程拿它和"理论上应该有的帧数"对比，差额就是缺口。

**名字的坑**：它叫 `totalSamples`，但累加的是 `sampleCount / channels`，也就是**帧数**。同学们读代码时别被名字骗了。要是叫 `totalFrames` 就清楚多了。这是本模块一个小小的命名遗憾，我们在坑点环节会再提一次。

**第五组：队列与辅助**

- `queue`：`BoundedQueue<AudioChunkPtr>`，容量 64。工作线程 push，消费线程 pop。
- `comOwned`：COM 是不是我们初始化的（决定 `close()` 要不要 `CoUninitialize`）。
- `log`：日志回调。
- 三个成员函数：`workerLoop`（工作线程主体）、`pushChunk`（把数据包成 `AudioChunk` 入队）、`padSilence`（补静音块）。

### 6.2 为什么内部类成员要用 `std::atomic` 而不是普通 bool

多说一句 `stopping` 为什么必须是 `std::atomic<bool>`。

如果它是普通 `bool`，那么"主线程写 true、工作线程读"这种操作在 C++ 标准里属于**数据竞争（data race）**，是**未定义行为**。实际后果可能是：

- 工作线程永远看不到这个变化（编译器把读操作优化成只读一次寄存器）；
- 或者看到一半写了一半的垃圾值（虽然 bool 不太会，但原则是原则）。

`std::atomic` 保证：写入立即对其他线程可见，且操作不可分割。这是多线程代码的最低门槛。

---

## 第七章：pushChunk 与 padSilence —— 两条入队路径

这两个函数是"产出 `AudioChunk`"的两个入口：**一条走真实数据，一条走补的静音**。我们先看它们，因为 `workerLoop` 会调用它们。

### 7.1 pushChunk：把采样包成 AudioChunk

```cpp
void AudioCapturer::Impl::pushChunk(const float* data, size_t sampleCount, bool silent)
{
    auto chunk = std::make_shared<AudioChunk>();
    chunk->sampleRate = sampleRate;
    chunk->channels = channels;
    chunk->pts = totalSamples;

    if (silent || !data)
        chunk->samples.assign(sampleCount, 0.0f);
    else
        chunk->samples.assign(data, data + sampleCount);

    totalSamples += static_cast<int64_t>(sampleCount / (channels > 0 ? channels : 1));
    queue.push(std::move(chunk));
}
```

逐行讲：

**第 1 步：造一个 `AudioChunk`。**

```cpp
auto chunk = std::make_shared<AudioChunk>();
```

用 `make_shared` 而不是 `new` + `shared_ptr`——一次分配搞定对象和控制块，更快也更安全（异常安全）。

**第 2 步：填元数据。**

```cpp
chunk->sampleRate = sampleRate;
chunk->channels = channels;
chunk->pts = totalSamples;
```

这就是 1.13 节契约的兑现：**采样率和声道数如实填写**（实际协商值，不是假装 48000），**pts 填当前时间账本的值**。

注意 `pts` 填的是 `totalSamples`，也就是**这一块的第一帧在整个流中的帧序号**。这是标准做法：pts 标记块的**起始时刻**。

**第 3 步：填数据（两条分支）。**

```cpp
if (silent || !data)
    chunk->samples.assign(sampleCount, 0.0f);
else
    chunk->samples.assign(data, data + sampleCount);
```

- `silent == true`（或 `data` 为空指针）：填 `sampleCount` 个 `0.0f`——**静音块**。
- 否则：从 `data` 拷贝 `sampleCount` 个 float 进来。

`vector::assign` 有两个重载，这里用得很巧：

- `assign(n, value)`：填 n 个相同的值；
- `assign(first, last)`：拷贝迭代器区间。

两种写法都一次搞定"开空间 + 填数据"，比 `resize` + `memcpy` 简洁。

**注意 `!data` 这个防御**：即使调用方忘了传 `silent` 但传了空指针，也不会崩，而是退化为静音块。**默认安全**原则。

**第 4 步：推进时间账本。**

```cpp
totalSamples += static_cast<int64_t>(sampleCount / (channels > 0 ? channels : 1));
```

这是**最关键的一行**，也是最容易读错的一行。请同学们停下来看清楚：

- `sampleCount` 是**采样总数**；
- 除以 `channels` 得到**帧数**；
- 所以 `totalSamples` 累加的其实是**帧数**。

那个 `channels > 0 ? channels : 1` 是防除零——如果 `channels` 还没初始化（理论上不会，因为 `pushChunk` 只在 `open` 成功后调用），就除以 1 而不是崩溃。

**这就是为什么 1.4 节要大家分清"采样"和"帧"**。搞不清这两个概念，这行代码就会读错，进而 pts 全错，音画同步全崩。

**第 5 步：入队。**

```cpp
queue.push(std::move(chunk));
```

`std::move` 把 `shared_ptr` 转移进队列，避免一次原子引用计数加加减减。`push` 之后 `chunk` 变成空指针（但我们已经不用它了）。

队列容量 64，满了会丢最旧的（1.13 节）。

### 7.2 padSilence：补静音

```cpp
void AudioCapturer::Impl::padSilence(int64_t frames)
{
    if (frames <= 0 || sampleRate <= 0 || channels <= 0)
        return;

    auto chunk = std::make_shared<AudioChunk>();
    chunk->sampleRate = sampleRate;
    chunk->channels = channels;
    chunk->pts = totalSamples;
    chunk->samples.assign(static_cast<size_t>(frames) * channels, 0.0f);

    totalSamples += frames;
    queue.push(std::move(chunk));
}
```

和 `pushChunk` 非常像，但有几个区别，同学们对比着看：

**区别一：前置检查。**

```cpp
if (frames <= 0 || sampleRate <= 0 || channels <= 0)
    return;
```

三个"不合理就什么都不做"的检查。`frames <= 0` 是最常见的（没有缺口时根本不会调到这里，但防御一下）。

**区别二：参数是"帧数"而不是"采样数"。**

```cpp
chunk->samples.assign(static_cast<size_t>(frames) * channels, 0.0f);
```

因为补静音时我们是"我知道缺了多少帧"，所以要乘 `channels` 换算成采样数。**注意方向和 `pushChunk` 相反**：

- `pushChunk`：入参是采样数，内部除以 channels 得到帧数；
- `padSilence`：入参是帧数，内部乘以 channels 得到采样数。

这个"一个除一个乘"的对称性，正好印证了 1.4 节那个公式：**采样数 = 帧数 × 声道数**。

**区别三：时间账本直接加。**

```cpp
totalSamples += frames;
```

不用换算，因为入参本来就是帧数。

**为什么 `padSilence` 不复用 `pushChunk(nullptr, frames*channels, true)`？** 其实完全可以复用，功能上等价。分开写的好处是**意图清晰**：读代码的人一眼就知道"这是补静音，不是正常数据"。而且补静音的入参语义（帧数）和正常数据（采样数）不同，分开写避免了单位混淆。这是可读性优先于代码复用的一个合理取舍。

---

## 第八章：workerLoop —— 工作线程主体全流程

这是本模块的心脏，约 60 行，我们一段一段拆。

```cpp
void AudioCapturer::Impl::workerLoop()
{
    ComScope com;

    LARGE_INTEGER frequency = {};
    LARGE_INTEGER startCounter = {};
    QueryPerformanceFrequency(&frequency);
    QueryPerformanceCounter(&startCounter);

    while (!stopping) {
        WaitForSingleObject(event, kPollIntervalMs);

        if (stopping)
            break;

        UINT32 packetFrames = 0;
        while (SUCCEEDED(captureClient->GetNextPacketSize(&packetFrames))
               && packetFrames > 0) {
            BYTE* data = nullptr;
            UINT32 frames = 0;
            DWORD flags = 0;

            const HRESULT hr =
                captureClient->GetBuffer(&data, &frames, &flags, nullptr, nullptr);
            if (FAILED(hr) || frames == 0) {
                if (SUCCEEDED(hr))
                    captureClient->ReleaseBuffer(frames);
                break;
            }

            const size_t sampleCount =
                static_cast<size_t>(frames) * static_cast<size_t>(channels);

            if (flags & AUDCLNT_BUFFERFLAGS_SILENT) {
                pushChunk(nullptr, sampleCount, true);
            } else {
                std::vector<float> converted;
                convertSamples(data, sampleCount, sampleFormat, converted);
                pushChunk(converted.data(), sampleCount, false);
            }

            captureClient->ReleaseBuffer(frames);
        }

        LARGE_INTEGER now = {};
        QueryPerformanceCounter(&now);
        const double elapsedSeconds =
            static_cast<double>(now.QuadPart - startCounter.QuadPart)
            / static_cast<double>(frequency.QuadPart);
        const int64_t expected =
            static_cast<int64_t>(elapsedSeconds * static_cast<double>(sampleRate));

        int64_t deficit = expected - totalSamples;
        const int64_t minSilence = static_cast<int64_t>(sampleRate) * kMinSilenceMs / 1000;
        if (deficit >= minSilence) {
            const int64_t maxSilence =
                static_cast<int64_t>(sampleRate) * kMaxSilenceMs / 1000;
            deficit = std::min(deficit, maxSilence);
            padSilence(deficit);
        }
    }
}
```

### 8.1 开场：COM 初始化 + 高精度时钟基准

```cpp
ComScope com;
```

**第一件事必须是初始化 COM。** 因为工作线程要调 `captureClient` 的 COM 方法，而 COM 初始化是**按线程**的（1.12 节第四条）。主线程初始化过不等于这个线程初始化过。这是新手写多线程 COM 最常见的崩溃点，请同学们务必记住。

```cpp
LARGE_INTEGER frequency = {};
LARGE_INTEGER startCounter = {};
QueryPerformanceFrequency(&frequency);
QueryPerformanceCounter(&startCounter);
```

这四行建立**高精度时间基准**。

- **`QueryPerformanceCounter`（QPC）**：Windows 提供的高精度计时器，读取一个单调递增的计数值。精度通常达到微秒级（比 `GetTickCount` 的毫秒级高得多，也更稳定）。
- **`QueryPerformanceFrequency`**：这个计数器每秒跳多少下（典型值 10,000,000，也就是 100 纳秒一下）。
- **`LARGE_INTEGER`**：一个 64 位整数的联合体，`QuadPart` 字段就是那个 64 位值。

我们记下**开始时刻的计数值**（`startCounter`）和**频率**（`frequency`），后面每一轮循环就能算出"从开始到现在过了多少秒"。

**为什么不用 `steady_clock`？** 其实 `std::chrono::steady_clock` 在 MSVC 上底层就是 QPC，两者等价。这里直接用 Windows API，是为了和整文件的 Windows 风格保持一致，也避免混用两套时间体系。

### 8.2 主循环：等待

```cpp
while (!stopping) {
    WaitForSingleObject(event, kPollIntervalMs);

    if (stopping)
        break;
```

循环条件是 `stopping` 这个原子标志。`close()` 会把它置 true，循环就退出。

`WaitForSingleObject(event, kPollIntervalMs)` 这一行**同时实现了两种模式**（1.10 节），非常巧妙：

- **事件模式**：WASAPI 有新数据时把 `event` 置为有信号 → 立刻返回，零延迟响应；
- **轮询模式**：事件句柄存在但没人置位 → 等待必然**超时**，10 毫秒后返回 → 我们就顺便查一次缓冲区。

一行代码，两种语义，靠"超时"这个兜底机制统一起来。这个写法值得同学们记住。

`if (stopping) break;` 放在等待之后：因为 `close()` 会 `SetEvent` 唤醒我们（见第十章），所以等待会立刻返回，然后我们检查标志退出——**停止响应及时，不会多睡 10 毫秒**。

### 8.3 内层循环：把缓冲区抽干

```cpp
UINT32 packetFrames = 0;
while (SUCCEEDED(captureClient->GetNextPacketSize(&packetFrames))
       && packetFrames > 0) {
```

**为什么是内层 `while` 而不是 `if`？** 因为一次唤醒时，缓冲区里可能**攒了好几包**（尤其轮询模式下，10 毫秒内可能攒了不止一包）。我们必须**一口气全部抽干**，否则：

- 数据会越积越多，最终缓冲区溢出；
- 延迟会越来越大（新数据要排队等旧数据被取走）。

这个"抽干"动作在第 5 讲的摄像头里不存在（那里是单槽覆盖式，天然只关心最新帧），是**队列模式特有的必要操作**。

`GetNextPacketSize` 返回的是**下一包的帧数**。返回 0 表示"暂时没数据了"，循环结束。

### 8.4 取包：GetBuffer 与错误分支

```cpp
BYTE* data = nullptr;
UINT32 frames = 0;
DWORD flags = 0;

const HRESULT hr =
    captureClient->GetBuffer(&data, &frames, &flags, nullptr, nullptr);
if (FAILED(hr) || frames == 0) {
    if (SUCCEEDED(hr))
        captureClient->ReleaseBuffer(frames);
    break;
}
```

`GetBuffer` 的五个参数：

1. `&data`：返回数据指针；
2. `&frames`：返回这一包有多少帧；
3. `&flags`：返回标志位；
4. `nullptr`：**设备位置（device position）**，这一包第一帧对应的设备采样序号——我们不用；
5. `nullptr`：**性能计数器（QP counter）**，这一包对应的精确时刻——我们也不用。

（第 4、5 个参数其实很有用——它们是硬件提供的精确时间戳，比我们后面用墙钟估算要准得多。坑点环节会讨论为什么这里没用。）

错误分支的逻辑值得细看：

```cpp
if (FAILED(hr) || frames == 0) {
    if (SUCCEEDED(hr))
        captureClient->ReleaseBuffer(frames);
    break;
}
```

翻译成人话：

- **`GetBuffer` 失败了**：直接 `break`，不还缓冲区（因为根本没拿到）；
- **`GetBuffer` 成功但 `frames == 0`**：这是个"空包"，**必须 `ReleaseBuffer` 还回去**（否则缓冲区泄漏，采集卡死），然后 `break`。

注意 `ReleaseBuffer(frames)` 传的是 `frames`，此时它是 0——意思是"释放 0 帧"，也就是"我没读任何东西，请归还"。这是 WASAPI 规定的"放弃这一包"的写法。

**这个分支体现了对 API 契约的精确理解**：成功拿到的东西，无论用不用，都要还。

### 8.5 计算采样数

```cpp
const size_t sampleCount =
    static_cast<size_t>(frames) * static_cast<size_t>(channels);
```

1.4 节的公式：**采样数 = 帧数 × 声道数**。

两个 `static_cast` 是为了避免 `UINT32 × int` 的类型提升规则带来的意外（尤其是 `channels` 是 `int`，如果它是负数，隐式转换会出问题）。显式写出来，编译器也不会警告。

### 8.6 静音标志 vs 真实数据

```cpp
if (flags & AUDCLNT_BUFFERFLAGS_SILENT) {
    pushChunk(nullptr, sampleCount, true);
} else {
    std::vector<float> converted;
    convertSamples(data, sampleCount, sampleFormat, converted);
    pushChunk(converted.data(), sampleCount, false);
}
```

**分支一：`AUDCLNT_BUFFERFLAGS_SILENT`。**

WASAPI 明确告诉我们："这一包是静音，数据内容不可信"。这时候 `data` 指向的缓冲区**可能是未初始化的内存，也可能是上一包的残留**。总之**不能读**。

所以我们直接补 `sampleCount` 个零。注意：**即使这一包是"静音"，它在时间轴上的长度是真实的**——所以 `pushChunk` 照样推进 `totalSamples`。这一点非常关键：**静音也是时间**。

**分支二：正常数据。**

```cpp
std::vector<float> converted;
convertSamples(data, sampleCount, sampleFormat, converted);
pushChunk(converted.data(), sampleCount, false);
```

三步：开一个临时 vector → 转换（第五章）→ 入队。

注意 `converted` 是**每包都重新分配**的局部变量。这在实时路径上是一个小的性能隐患（每秒可能几十次堆分配）。优化方向：把它提到循环外做成员复用（坑点环节讨论）。

### 8.7 还缓冲区

```cpp
captureClient->ReleaseBuffer(frames);
```

1.9 节强调的"三步走"的最后一步。**必须调用，且必须在下一次 `GetBuffer` 之前调用**。

注意它在 `if/else` 之后——两个分支都要走到这里。这样写比在两个分支里各写一次更不容易漏。

### 8.8 时间对账：静音补帧的核心算法

数据抽干之后，进入本函数最精彩的部分：

```cpp
LARGE_INTEGER now = {};
QueryPerformanceCounter(&now);
const double elapsedSeconds =
    static_cast<double>(now.QuadPart - startCounter.QuadPart)
    / static_cast<double>(frequency.QuadPart);
```

三步算出"从开始到现在过了多少秒"：

1. 读当前计数值 `now`；
2. `now - startCounter` 得到经过的"跳数"；
3. 除以 `frequency`（每秒跳数）得到秒数。

**为什么先转成 `double` 再除？** 因为整数除法会截断小数部分，而我们需要亚毫秒级的精度。

```cpp
const int64_t expected =
    static_cast<int64_t>(elapsedSeconds * static_cast<double>(sampleRate));
```

**理论上本该采到多少帧** = 经过的秒数 × 采样率。

举例：过了 2.5 秒，采样率 48000 → 应该有 120000 帧。

```cpp
int64_t deficit = expected - totalSamples;
```

**缺口** = 理论值 − 实际值。

- `deficit > 0`：采少了（系统没出声 / 丢包），需要补；
- `deficit <= 0`：够了或者超前了（超前可能因为缓冲区初始就有积压），不补。

```cpp
const int64_t minSilence = static_cast<int64_t>(sampleRate) * kMinSilenceMs / 1000;
if (deficit >= minSilence) {
    const int64_t maxSilence =
        static_cast<int64_t>(sampleRate) * kMaxSilenceMs / 1000;
    deficit = std::min(deficit, maxSilence);
    padSilence(deficit);
}
```

两个阈值，1.14 节讲过：

- **下限**（`kMinSilenceMs = 10`）：`sampleRate × 10 / 1000`，48000Hz 下是 480 帧。缺口小于一帧轮询的量，属于正常抖动，**不值得补**；
- **上限**（`kMaxSilenceMs = 200`）：`sampleRate × 200 / 1000`，48000Hz 下是 9600 帧。**一次最多补这么多**，防止程序卡死时一次性补出海量静音。

`std::min(deficit, maxSilence)` 就是那个截断操作。

**这个算法用一个比喻总结**：就像食堂打饭，我们按"营业时间 × 平均客流"估算今天该卖出多少份饭，实际卖少了就自己补上几份空的——保证账本上的"份数"对得上营业时间。下游（编码器）按份数出餐，时间轴就是准的。

**注意它只在"没数据"的时候才真正起作用**：如果 loopback 一直在给数据，`totalSamples` 会紧跟 `expected`，`deficit` 始终是 0 或负数，补帧逻辑从不触发。它是纯粹的**兜底机制**。

### 8.9 一张完整的数据流图

把整条链路画出来，同学们对照着回忆一遍：

```
                    ┌──────────────────────────────┐
                    │  Windows 音频引擎（混音器）    │
                    │  所有程序的声音在这里混合      │
                    └──────────────┬───────────────┘
                                   │ 复制一份（loopback）
                                   ▼
                    ┌──────────────────────────────┐
                    │  WASAPI 缓冲区（100 毫秒）     │
                    └──────────────┬───────────────┘
                                   │ 事件置位 / 10ms 轮询
                                   ▼
   ┌───────────────────────────────────────────────────────────┐
   │  workerLoop（我们的工作线程）                               │
   │                                                            │
   │  while (!stopping) {                                       │
   │      WaitForSingleObject(event, 10ms)                      │
   │      while (GetNextPacketSize > 0) {                       │
   │          GetBuffer → convertSamples → pushChunk ──┐        │
   │          ReleaseBuffer                            │        │
   │      }                                            │        │
   │      时间对账 → padSilence（缺了才补）──────────────┤        │
   │  }                                                │        │
   └───────────────────────────────────────────────────┼────────┘
                                                       ▼
                                          ┌────────────────────────┐
                                          │ BoundedQueue<AudioChunkPtr> │
                                          │ 容量 64，满了丢最旧     │
                                          └────────────┬───────────┘
                                                       │ nextChunk(timeoutMs)
                                                       ▼
                                                  消费方（编码器）
```

---

## 第九章：open() 全流程十二步详解

`open()` 是本 commit 最重的函数，约 125 行。我们拆成十二步。先看全景地图：

```
open()
 ├─ 第 0 步：复位（close + 存日志回调）
 ├─ 第 1 步：定义 say / fail 两个 lambda
 ├─ 第 2 步：COM 初始化（兼容 RPC_E_CHANGED_MODE）
 ├─ 第 3 步：创建设备枚举器 IMMDeviceEnumerator
 ├─ 第 4 步：按类型决定数据流向（eRender / eCapture）
 ├─ 第 5 步：定位设备（默认设备 or 按索引枚举）
 ├─ 第 6 步：读设备名（friendlyNameOf）
 ├─ 第 7 步：激活 IAudioClient
 ├─ 第 8 步：读混音格式 GetMixFormat + 解析三要素
 ├─ 第 9 步：创建事件对象
 ├─ 第 10 步：Initialize（含 loopback 事件模式降级）
 ├─ 第 11 步：SetEventHandle + GetService
 └─ 第 12 步：Start + 启动工作线程 + 打日志
```

### 9.1 第 0~1 步：复位与失败框架

```cpp
bool AudioCapturer::open(AudioSourceType type, int deviceIndex, LogCallback log)
{
    close();

    impl_->log = std::move(log);

    const auto say = [this](const std::string& message) {
        if (impl_->log)
            impl_->log(message);
    };
    const auto fail = [this, &say](const std::string& reason) {
        say(reason);
        close();
        return false;
    };
```

和第 5 讲 `CameraCapturer::open` 完全一样的开场结构，同学们应该很熟悉了：

1. **`open()` 开头先 `close()`**：允许"没关就再开"，对调用方友好；同时保证重开时状态干净。这要求 `close()` 必须**幂等**（可以反复调）。
2. **`say`**：有日志回调就打一条。
3. **`fail`**：统一失败出口——打日志 + 完整清理（`close()` 会把建了一半的东西拆掉）+ 返回 false。后面十几处失败分支全靠它收口，保证**任何一步失败都不留半拉子资源**。

注意 `std::move(log)`：把调用方传进来的 `std::function` **移动**进 `Impl`，避免一次拷贝。传 `std::function` 这种可能持有堆资源的大对象，用 move 是好习惯。

### 9.2 第 2 步：COM 初始化

```cpp
const HRESULT comHr = CoInitializeEx(nullptr, COINIT_MULTITHREADED);
if (comHr == RPC_E_CHANGED_MODE) {
    say("当前线程已用单线程套间(STA)初始化过 COM，直接复用");
} else if (FAILED(comHr)) {
    return fail("CoInitializeEx 失败，无法使用 WASAPI");
} else {
    impl_->comOwned = true;
}
```

三分支模式，第 5 讲 1.6 节讲过：

1. `RPC_E_CHANGED_MODE`：这个线程已经被别人（典型是 Qt 主线程）初始化成 STA 了，改不了。**复用它，打个日志继续**——WASAPI 在 STA 下也能工作；
2. 真失败：放弃；
3. 成功：`comOwned = true`，记下"COM 是我们初始化的，`close()` 时我们负责反初始化"。

**谁初始化谁反初始化**——这个 `comOwned` 标志就是这条纪律的执行者。

### 9.3 第 3~4 步：创建枚举器，决定数据流向

```cpp
if (FAILED(CoCreateInstance(__uuidof(MMDeviceEnumerator), nullptr, CLSCTX_ALL,
                            IID_PPV_ARGS(&impl_->enumerator))))
    return fail("创建设备枚举器失败");

const EDataFlow flow =
    (type == AudioSourceType::SystemAudio) ? eRender : eCapture;
impl_->loopback = (type == AudioSourceType::SystemAudio);
```

`CoCreateInstance(__uuidof(MMDeviceEnumerator), ...)` —— 1.12 节讲过，`__uuidof` 从类型上取 GUID。

然后**决定数据流向**（1.8 节）：

- 要抓**系统声音** → 去 `eRender`（渲染端点，也就是扬声器）那边开流；
- 要抓**麦克风** → 去 `eCapture`（采集端点）那边开流。

同时把 `loopback` 标志记下来（后面加 `AUDCLNT_STREAMFLAGS_LOOPBACK` 要用）。

`type` 和 `loopback` 是同一个判断的两种表达（一个是枚举，一个是布尔），都存下来是为了后面用起来方便。

### 9.4 第 5 步：定位设备（两条路）

```cpp
std::string deviceName;

if (deviceIndex < 0) {
    if (FAILED(impl_->enumerator->GetDefaultAudioEndpoint(flow, eConsole,
                                                          &impl_->device)))
        return fail("找不到默认音频设备");
} else {
    IMMDeviceCollection* collection = nullptr;
    if (FAILED(impl_->enumerator->EnumAudioEndpoints(flow, DEVICE_STATE_ACTIVE,
                                                     &collection))
        || !collection)
        return fail("枚举音频设备失败");

    UINT count = 0;
    collection->GetCount(&count);
    if (static_cast<UINT>(deviceIndex) >= count) {
        collection->Release();
        return fail("音频设备索引越界：一共只有 " + std::to_string(count) + " 个");
    }

    const HRESULT hr = collection->Item(static_cast<UINT>(deviceIndex),
                                        &impl_->device);
    collection->Release();
    if (FAILED(hr) || !impl_->device)
        return fail("打开指定音频设备失败");
}
```

**路线 A：`deviceIndex < 0` → 用默认设备。**

```cpp
GetDefaultAudioEndpoint(flow, eConsole, &impl_->device)
```

- `flow`：`eRender` 或 `eCapture`（第 4 步定的）；
- `eConsole`：设备角色（1.8 节），也就是"用户在系统设置里看到的那个默认设备"。

这是最常用的一条路——绝大多数场景用户就用系统默认设备。

**路线 B：`deviceIndex >= 0` → 按索引枚举。**

```cpp
EnumAudioEndpoints(flow, DEVICE_STATE_ACTIVE, &collection)
```

- `DEVICE_STATE_ACTIVE`：只要当前可用的（1.8 节）。

然后：

```cpp
UINT count = 0;
collection->GetCount(&count);
```

问集合里有几个设备。**越界检查在这里做**，而且错误信息里直接告诉你"一共只有 N 个"——**错误信息要替用户想到他接下来想问的问题**（第 5 讲强调过同样的原则）。

```cpp
const HRESULT hr = collection->Item(static_cast<UINT>(deviceIndex), &impl_->device);
collection->Release();
```

`Item(i, &device)` 取第 i 个设备。取完立刻 `Release` 集合——COM 编程的"**用完即还**"原则（不攒着一起还，减少出错窗口）。

注意 `static_cast<UINT>(deviceIndex)`：因为前面已经确认 `deviceIndex >= 0` 且 `< count`，所以这个转换是安全的。如果不加检查直接转，负数的 `int` 转成无符号会变成一个巨大的数——**经典的整数类型陷阱**。这里检查在先、转换在后，顺序正确。

### 9.5 第 6 步：读设备名

```cpp
deviceName = friendlyNameOf(impl_->device);
```

一行，用 3.3 节那个函数读友好名字。读不到就是空串，后面日志里会显示 `<未知设备>`。

### 9.6 第 7 步：激活 IAudioClient

```cpp
if (FAILED(impl_->device->Activate(__uuidof(IAudioClient), CLSCTX_ALL, nullptr,
                                   reinterpret_cast<void**>(&impl_->audioClient))))
    return fail("激活 IAudioClient 失败");
```

1.12 节讲过的 `Activate`：在 `IMMDevice` 上激活 `IAudioClient` 接口。

注意最后一个参数是 `void**`，所以不能用 `IID_PPV_ARGS`（那个宏需要具体类型来推导），只能手写 `reinterpret_cast<void**>(&ptr)`。这是 WASAPI 代码里常见的一个小别扭。

**这一步是"从设备身份到数据流"的转折点**：之前我们只是在"找设备"，从这里开始我们真正要开流了。

### 9.7 第 8 步：读混音格式，解析三要素

```cpp
if (FAILED(impl_->audioClient->GetMixFormat(&impl_->mixFormat))
    || !impl_->mixFormat)
    return fail("读取系统混音格式失败");

impl_->sampleRate = static_cast<int>(impl_->mixFormat->nSamplesPerSec);
impl_->channels = static_cast<int>(impl_->mixFormat->nChannels);
impl_->sampleFormat = detectSampleFormat(impl_->mixFormat);

if (impl_->sampleFormat == SampleFormat::Unsupported)
    return fail("不支持的采样格式：只有 float32 和 16bit PCM 会被处理");
```

**`GetMixFormat`** 问音频引擎："你的混音格式是什么？" 返回一个 `WAVEFORMATEX*`（**COM 分配器给的内存，`close()` 里要用 `CoTaskMemFree` 还，1.12 节**）。

**为什么必须问它，而不能自己指定？** 因为在共享模式下（1.7 节），数据流经音频引擎，格式由引擎决定。你只能按它的格式接收，**没有讨价还价的余地**。这是共享模式的代价。

然后解析出三个要素（1.1~1.5 节）：

- `nSamplesPerSec` → 采样率；
- `nChannels` → 声道数；
- `detectSampleFormat(...)` → float32 还是 int16（第四章）。

最后一句是关键的质量控制：

```cpp
if (impl_->sampleFormat == SampleFormat::Unsupported)
    return fail("不支持的采样格式：只有 float32 和 16bit PCM 会被处理");
```

**不认识的格式直接失败，不硬扛。** 这比"猜着处理然后输出噪声"高明得多。错误信息也写得很清楚——把我们支持什么直接告诉用户。

### 9.8 第 9 步：创建事件对象

```cpp
impl_->event = CreateEventW(nullptr, FALSE, FALSE, nullptr);
if (!impl_->event)
    return fail("创建音频事件失败");
```

1.10 节讲过：匿名、自动复位、初始无信号的事件对象。

注意检查返回值——`CreateEventW` 失败返回 `nullptr`（极少见，但系统句柄耗尽时会发生）。

**为什么要创建它，哪怕最后降级到轮询模式？** 因为轮询模式下的 `WaitForSingleObject(event, 10)` 也需要一个有效的句柄来"睡 10 毫秒"。所以事件对象是**两种模式都要用**的，必须无条件创建。

### 9.9 第 10 步：Initialize —— 带降级的流初始化

这是本函数策略性最强的一段：

```cpp
DWORD streamFlags = AUDCLNT_STREAMFLAGS_EVENTCALLBACK;
if (impl_->loopback)
    streamFlags |= AUDCLNT_STREAMFLAGS_LOOPBACK;

HRESULT hr = impl_->audioClient->Initialize(
    AUDCLNT_SHAREMODE_SHARED, streamFlags, kBufferDuration, 0, impl_->mixFormat,
    nullptr);

if (FAILED(hr)) {
    if (impl_->loopback) {
        streamFlags &= ~AUDCLNT_STREAMFLAGS_EVENTCALLBACK;
        hr = impl_->audioClient->Initialize(AUDCLNT_SHAREMODE_SHARED, streamFlags,
                                            kBufferDuration, 0, impl_->mixFormat,
                                            nullptr);
        if (SUCCEEDED(hr))
            say("这台机器的 loopback 不支持事件回调，改用 10ms 轮询");
    }
    if (FAILED(hr))
        return fail("初始化音频客户端失败");
}
```

**先组装标志位。**

```cpp
DWORD streamFlags = AUDCLNT_STREAMFLAGS_EVENTCALLBACK;
if (impl_->loopback)
    streamFlags |= AUDCLNT_STREAMFLAGS_LOOPBACK;
```

`streamFlags` 是位掩码（bitmask），用 `|=` 叠加标志：

- `AUDCLNT_STREAMFLAGS_EVENTCALLBACK`：我要用事件回调（1.10 节）；
- `AUDCLNT_STREAMFLAGS_LOOPBACK`：我要抓的是系统声音（1.8 节）。

**第一次 Initialize。**

```cpp
Initialize(AUDCLNT_SHAREMODE_SHARED,   // 共享模式
           streamFlags,                // 事件回调 + 可能的 loopback
           kBufferDuration,            // 缓冲区 100 毫秒
           0,                          // 周期性 = 0（共享模式下忽略）
           impl_->mixFormat,           // 格式就用混音格式
           nullptr);                   // 会话 GUID（不用，用默认会话）
```

六个参数逐个解释：

1. **共享模式**（1.7 节）——不独占设备；
2. **标志位**——见上；
3. **缓冲区时长**——`kBufferDuration = 1000000`（100 毫秒，单位是 100 纳秒）；
4. **周期（periodicity）**——独占模式下才有意义（表示设备中断周期），共享模式传 0；
5. **格式**——`mixFormat`，我们刚问来的；
6. **会话 GUID**——可以把多个流归到同一个"音频会话"里（会话在音量 mixer 里显示为一个滑块）。传 `nullptr` 用默认会话。

**降级重试。**

```cpp
if (FAILED(hr)) {
    if (impl_->loopback) {
        streamFlags &= ~AUDCLNT_STREAMFLAGS_EVENTCALLBACK;
        hr = impl_->audioClient->Initialize(..., streamFlags, ...);
        if (SUCCEEDED(hr))
            say("这台机器的 loopback 不支持事件回调，改用 10ms 轮询");
    }
    if (FAILED(hr))
        return fail("初始化音频客户端失败");
}
```

`streamFlags &= ~AUDCLNT_STREAMFLAGS_EVENTCALLBACK` 是"**清掉某一位**"的标准写法：`~` 取反得到"除了这一位全是 1"的掩码，`&=` 就把这一位清零，其他位保留。

为什么 loopback 可能不支持事件回调？这是某些音频驱动的**实现限制**——回环流的事件通知机制在这些驱动上没实现好。微软官方文档里也提到过这类兼容性问题。作者显然是在真机（或用户反馈）上踩过这个坑，才写下这段降级。

**这个降级策略和第 5 讲 `RenderStream` 的两级降级（换引脚 → 放弃格式）是同一种思维模式**：先提理想条件，谈不拢就让步，让到不能再让才放弃。硬件世界充满不确定性，写代码要像谈判。

注意降级只在 `loopback` 时尝试——因为麦克风（`eCapture`）不支持事件回调的机器极其罕见，不值得为它写降级路径。**降级也要有针对性**。

### 9.10 第 11 步：绑定事件句柄 + 取采集客户端

```cpp
if (streamFlags & AUDCLNT_STREAMFLAGS_EVENTCALLBACK) {
    if (FAILED(impl_->audioClient->SetEventHandle(impl_->event)))
        say("设置音频事件句柄失败，退回轮询模式");
}

if (FAILED(impl_->audioClient->GetService(
        __uuidof(IAudioCaptureClient),
        reinterpret_cast<void**>(&impl_->captureClient))))
    return fail("获取 IAudioCaptureClient 失败");
```

**`SetEventHandle`**：把我们的事件句柄交给 WASAPI，告诉它"有新数据就置位这个事件"。

注意这里的失败处理很特别：

```cpp
if (FAILED(...))
    say("设置音频事件句柄失败，退回轮询模式");
```

**只打日志，不返回失败。** 为什么？因为**即使事件绑定失败，我们依然能工作**——靠 `WaitForSingleObject` 的 10 毫秒超时兜底，退化成轮询模式。功能不打折，只是响应延迟从"立即"变成"最多 10 毫秒"。

这是一个很漂亮的**优雅降级**：非致命错误不当失败处理，只提示。同学们写自己的代码时也要区分"致命错误"和"可以凑合的错误"。

**`GetService`**：从 `IAudioClient` 上取 `IAudioCaptureClient`（1.6 节那张图的最后一环）。和 `Activate` 一样是"同一对象换张面孔"，参数也一样是 `void**`。

这一步失败就是真失败了——没有采集客户端就没法取数据，所以走 `fail`。

### 9.11 第 12 步：启动流与工作线程

```cpp
impl_->totalSamples = 0;
impl_->stopping = false;

if (FAILED(impl_->audioClient->Start()))
    return fail("启动音频采集失败");

impl_->worker = std::thread([this] { impl_->workerLoop(); });

say(std::string(impl_->loopback ? "系统声音" : "麦克风") + "已启动: "
    + (deviceName.empty() ? std::string("<未知设备>") : deviceName));
say("采集格式: " + std::to_string(impl_->sampleRate) + " Hz, "
    + std::to_string(impl_->channels) + " 声道, "
    + (impl_->sampleFormat == SampleFormat::Float32 ? "float32" : "int16"));

return true;
```

**先复位计数器。** `totalSamples = 0`（时间账本从头开始）、`stopping = false`（工作线程可以跑了）。注意 `stopping` 的复位必须在**启动线程之前**（`close()` 里置过 true 了）。

**`Start()`**：通知 WASAPI 开始往缓冲区里灌数据。这一步之后，数据就开始流动了。

注意顺序：**先 `Start()` 再开线程**。如果反过来，线程会先跑起来，然后立刻发现"缓冲区里没数据"（因为流还没启动），白白浪费一轮循环。虽然不影响正确性，但这个顺序更自然。

**开工作线程。**

```cpp
impl_->worker = std::thread([this] { impl_->workerLoop(); });
```

`std::thread` 构造时就开始运行。lambda 捕获 `this`（`AudioCapturer` 对象的指针），然后调 `impl_->workerLoop()`。

**这里有一个重要的安全前提**：`AudioCapturer` 对象必须在工作线程结束之前**一直活着**。因为 `workerLoop` 通过 `this->impl_` 访问所有状态。如果对象先析构了，线程还在访问 `impl_`——**野指针，崩溃**。

代码用 `close()` 里的 `join()` 保证这一点（第十章），析构函数也调 `close()`。所以只要用户不手动 `delete` 一个还在采集的对象，就是安全的。

**打两条日志。**

```cpp
say(std::string(impl_->loopback ? "系统声音" : "麦克风") + "已启动: " + deviceName);
say("采集格式: " + std::to_string(impl_->sampleRate) + " Hz, " + ...);
```

这两条日志非常重要，是现场排查"我到底开了什么设备、拿到什么格式"的第一手证据。同学们跑测试时一定要看这两行。

第一条还处理了无名设备的情况：`deviceName.empty() ? "<未知设备>" : deviceName`。

### 9.12 open() 十二步小结

回顾一下这个函数体现的工程素养：

1. **每一步都可能失败，每个失败都有明确日志**——十几处 `fail`，每一条错误信息都说明了"哪一步、为什么"；
2. **失败路径完整清理**——`fail` 里调 `close()`，不留半拉子资源；
3. **分级错误处理**——致命的（`GetService` 失败）直接失败，可凑合的（`SetEventHandle` 失败）只提示；
4. **兼容性降级**——loopback 不支持事件回调就退轮询；
5. **信息透明**——启动后把实际格式打印出来，不藏着。

`open()` 讲完了。同学们可以和第 5 讲的 `CameraCapturer::open()` 对比着看——**两个函数的结构几乎一模一样**（复位 → COM → 找设备 → 协商格式 → 配置 → 启动），这说明作者已经形成了一套稳定的"打开硬件"的方法论。这种一致性本身就是代码质量的体现。

---

## 第十章：close() 与 nextChunk() —— 关停顺序与消费端

### 10.1 close：关停顺序就是一切

```cpp
void AudioCapturer::close()
{
    impl_->stopping = true;
    if (impl_->event)
        SetEvent(impl_->event);

    if (impl_->worker.joinable())
        impl_->worker.join();

    if (impl_->audioClient)
        impl_->audioClient->Stop();

    impl_->queue.clear();

    safeRelease(impl_->captureClient);
    safeRelease(impl_->audioClient);
    safeRelease(impl_->device);
    safeRelease(impl_->enumerator);

    if (impl_->mixFormat) {
        CoTaskMemFree(impl_->mixFormat);
        impl_->mixFormat = nullptr;
    }
    if (impl_->event) {
        CloseHandle(impl_->event);
        impl_->event = nullptr;
    }

    impl_->sampleRate = 0;
    impl_->channels = 0;
    impl_->sampleFormat = SampleFormat::Unsupported;
    impl_->loopback = false;
    impl_->totalSamples = 0;

    if (impl_->comOwned) {
        CoUninitialize();
        impl_->comOwned = false;
    }
}
```

这个顺序是经过仔细考虑的，我们一段段体会。

**第 1 段：通知工作线程退出（三步）**

```cpp
impl_->stopping = true;
if (impl_->event)
    SetEvent(impl_->event);

if (impl_->worker.joinable())
    impl_->worker.join();
```

三步缺一不可：

1. **置 `stopping = true`**：告诉工作线程"该退出了"。原子写，工作线程下一轮循环条件就能看到。
2. **`SetEvent(event)`**：**关键的一步。** 工作线程此刻正睡在 `WaitForSingleObject(event, 10)` 里。如果我们不主动置位事件，它还要**最多睡满 10 毫秒**才会醒来发现 `stopping`。主动 `SetEvent` 让它**立刻醒来**——关停响应从"最多 10 毫秒"缩短到"几乎立即"。
3. **`join()`**：**阻塞等待线程真正结束**。

第 3 步是**绝对不能省**的。同学们记住这条铁律：**`std::thread` 对象析构时如果线程还在运行且没被 join/detach，程序会直接 `std::terminate` 崩溃**。而更重要的是，工作线程通过 `this->impl_` 访问状态——**必须在释放任何资源之前确保它已经停了**。否则就是经典的"线程还在用，资源已经没了"的竞态崩溃。

`joinable()` 检查是必要的：因为 `close()` 可能被调用多次（幂等要求），第二次调用时线程已经被 join 过了，不能再 join（会抛异常）。

**第 2 段：停流、清队列**

```cpp
if (impl_->audioClient)
    impl_->audioClient->Stop();

impl_->queue.clear();
```

`Stop()` 通知 WASAPI 停止灌数据。放在 join **之后**——因为线程里还在用 `captureClient`，必须先保证线程退出了。

`queue.clear()` 清掉没消费的块。放在这里是为了**释放内存**（每个块可能有几十 KB，64 块就是几 MB）。

**第 3 段：释放 COM 对象（逆序）**

```cpp
safeRelease(impl_->captureClient);
safeRelease(impl_->audioClient);
safeRelease(impl_->device);
safeRelease(impl_->enumerator);
```

**逆序释放**——和获取顺序（enumerator → device → audioClient → captureClient）完全相反。这是资源释放的通用纪律：**先拿到的后还**。原因是后获取的对象往往依赖先获取的对象（比如 `captureClient` 是从 `audioClient` 上取下来的），先释放被依赖者会把依赖者架空。

**第 4 段：释放非 COM 资源**

```cpp
if (impl_->mixFormat) {
    CoTaskMemFree(impl_->mixFormat);
    impl_->mixFormat = nullptr;
}
if (impl_->event) {
    CloseHandle(impl_->event);
    impl_->event = nullptr;
}
```

两种资源，两套释放方式：

- `mixFormat`：COM 任务分配器给的内存 → **`CoTaskMemFree`**（1.12 节）；
- `event`：Windows 内核对象句柄 → **`CloseHandle`**。

**分配方式决定了释放方式**，混用会导致内存泄漏或堆损坏。

**第 5 段：复位状态**

```cpp
impl_->sampleRate = 0;
impl_->channels = 0;
impl_->sampleFormat = SampleFormat::Unsupported;
impl_->loopback = false;
impl_->totalSamples = 0;
```

把所有协商结果清零。这样：

- `close()` 之后调 `sampleRate()` 返回 0，调用方能看出来"没开"；
- 再次 `open()` 时不会残留旧值。

**第 6 段：COM 反初始化**

```cpp
if (impl_->comOwned) {
    CoUninitialize();
    impl_->comOwned = false;
}
```

只在"我们初始化的"时候才反初始化（9.2 节的 `comOwned`）。并且**置回 false**，保证幂等（第二次 `close()` 不会重复 `CoUninitialize`）。

**注意**：`CoUninitialize` 放在最后，而且它只反初始化**调用 `close()` 的这个线程**的 COM。如果 `open()` 在主线程、`close()` 在另一个线程，`comOwned` 的账就记错了。这是坑点环节的一条。

### 10.2 nextChunk：消费端

```cpp
AudioChunkPtr AudioCapturer::nextChunk(int timeoutMs)
{
    return impl_->queue.tryPop(timeoutMs).value_or(nullptr);
}
```

一行，但信息量不小。

**`tryPop`** 返回 `std::optional<AudioChunkPtr>`：

- 队列里有东西 → `optional` 里有值；
- 超时了队列还是空的 → `std::nullopt`。

**`value_or(nullptr)`** 把 `optional` 拍平：有值就返回值，没值就返回 `nullptr`。

于是调用方看到的是一个非常干净的契约：

```cpp
AudioChunkPtr chunk = capturer.nextChunk(50);
if (!chunk) {
    // 这一拍没数据（超时）
}
```

这就是 `std::optional` 的典型用法——**用类型系统表达"可能没有"**，比返回裸指针 + 文档说明"可能为空"要安全得多，也比抛异常轻量。

对比第 5 讲的 `CameraCapturer::nextFrame`：那个是自己写 mutex + condition_variable + pending 单槽，今天这个直接**复用了第 3 讲的 `BoundedQueue`**。同学们可以看到：**基础设施做好了，上层代码就是这么短**。

### 10.3 两个 getter 与构造/析构

```cpp
int AudioCapturer::sampleRate() const
{
    return impl_->sampleRate;
}

int AudioCapturer::channels() const
{
    return impl_->channels;
}
```

返回协商结果。`open()` 成功之后才有意义。

注意它们**没有加锁**。`int` 的读写在 x86 上是原子的，但严格来说，`open()`/`close()` 写、其他线程读，理论上是数据竞争。实践中因为这些值只在 `open`/`close` 时变（而不是每帧变），风险很低。坑点环节会提。

```cpp
AudioCapturer::AudioCapturer() : impl_(std::make_unique<Impl>())
{
}

AudioCapturer::~AudioCapturer()
{
    close();
}
```

构造：用 `make_unique` 造 `Impl`（异常安全，比 `new` 好）。

析构：只调 `close()`。因为 `close()` 是幂等的，用户手动 `close()` 过再析构也不会双重释放。**析构函数只做一件事，而且必须做**——这是 RAII 的基本要求：对象消失时，它持有的所有资源（COM 对象、线程、句柄、内存）必须全部归还。

---

## 第十一章：listDevices —— 静态枚举函数

```cpp
std::vector<std::string> AudioCapturer::listDevices(AudioSourceType type)
{
    std::vector<std::string> names;

    ComScope com;

    IMMDeviceEnumerator* enumerator = nullptr;
    if (FAILED(CoCreateInstance(__uuidof(MMDeviceEnumerator), nullptr, CLSCTX_ALL,
                                IID_PPV_ARGS(&enumerator))))
        return names;

    const EDataFlow flow =
        (type == AudioSourceType::SystemAudio) ? eRender : eCapture;

    IMMDeviceCollection* collection = nullptr;
    if (SUCCEEDED(enumerator->EnumAudioEndpoints(flow, DEVICE_STATE_ACTIVE,
                                                 &collection))
        && collection) {
        UINT count = 0;
        collection->GetCount(&count);
        for (UINT i = 0; i < count; ++i) {
            IMMDevice* device = nullptr;
            if (SUCCEEDED(collection->Item(i, &device)) && device) {
                names.push_back(friendlyNameOf(device));
                device->Release();
            }
        }
        collection->Release();
    }

    enumerator->Release();
    return names;
}
```

### 11.1 为什么是静态函数

`listDevices` 不需要任何实例状态——还没 `open` 就能查。所以设计成 `static`，调用方式是 `AudioCapturer::listDevices(...)`。

UI 的设备下拉框就靠它填：启动时调一次 `SystemAudio` 列扬声器、调一次 `Microphone` 列麦克风。

### 11.2 用 ComScope 而不是手动管理

```cpp
ComScope com;
```

因为 COM 的生命周期**就在这个函数内**（进去枚举一下就出来），所以用 RAII 最合适（3.2 节）。函数返回时（无论是正常返回还是中途 `return names`），`ComScope` 的析构函数都会被调用，COM 一定被反初始化。

**对比**：`open()` 里 COM 要活到 `close()`，跨函数了，RAII 管不了，只能用 `comOwned` 手动记。

**这是本模块一个值得学习的对照**：同样是 COM 初始化，作用域内能管住的用 RAII，跨作用域的用标志位。两种手法在同一个文件里各司其职。

### 11.3 主流程

1. **创建枚举器**——失败就返回空 vector（不抛异常、不打日志，因为静态函数没有 log 回调）；
2. **决定数据流向**——和 `open()` 第 4 步完全一样的判断；
3. **枚举**——`EnumAudioEndpoints(flow, DEVICE_STATE_ACTIVE, &collection)`；
4. **循环取名字**——`collection->Item(i, &device)` 取设备，`friendlyNameOf` 读名字，`device->Release()` 立刻还；
5. **释放**——`collection->Release()`、`enumerator->Release()`。

注意第 4 步里 `device->Release()` 在循环**内部**：每取一个设备用完立刻释放，不等循环结束。这是 COM 的"用完即还"。

### 11.4 一个细节：为什么失败时返回空 vector 而不是抛异常

```cpp
if (FAILED(CoCreateInstance(...)))
    return names;     // names 此时是空的
```

**返回一个空的 vector，而不是抛异常、也不是返回错误码。**

这个设计的理由：设备枚举失败（比如系统音频服务没启动）是**可以容忍的情况**——调用方（UI）拿到空列表，显示"没有可用设备"就行了，不需要特殊的错误处理路径。

这是"**让常见情况简单**"的设计原则：调用方写 `for (auto& name : listDevices(...))` 就行，不用 try-catch，不用判错误码。

---

## 第十二章：测试程序教学 —— 六段测试

`tests/test_audio_capture.cpp`（400 行）是本次提交的另一半。和第 5 讲那个一样，它不是"跑通就行"的冒烟测试，而是一份**教你如何验证音频采集代码**的教案。

### 12.0 准备工作：参数解析与工具函数

```cpp
int main(int argc, char** argv)
{
#if defined(_WIN32)
    std::system("chcp 65001 > nul");
#endif

    static char stdoutBuffer[8192];
    std::setvbuf(stdout, stdoutBuffer, _IOLBF, sizeof(stdoutBuffer));

    int durationMs = 2000;
    bool withSound = false;

    for (int i = 1; i < argc; ++i) {
        if (std::strcmp(argv[i], "--sound") == 0) {
            withSound = true;
        } else {
            const int requested = std::atoi(argv[i]);
            if (requested >= 500 && requested <= 30000)
                durationMs = requested;
        }
    }
```

**`chcp 65001`**：把控制台切到 UTF-8 代码页，中文日志不乱码（`> nul` 是把 chcp 自己的输出丢掉）。这是 Windows 上写中文控制台程序的必备咒语。

**`setvbuf(stdout, buffer, _IOLBF, size)`**：设置 stdout 的缓冲模式。

- `_IOLBF` = **行缓冲（Line Buffered）**：遇到换行符就刷新。
- 为什么要手动设？因为 Windows 上 stdout 重定向到管道时默认是全缓冲，日志会攒一大坨才输出，看起来像"程序卡住了"。行缓冲让日志实时可见。
- `static char stdoutBuffer[8192]`：缓冲区是静态的——**必须是静态的**！因为 `setvbuf` 只保存指针，不拷贝内容；如果是局部变量，函数一返回就悬空了。这是一个容易踩的坑。

**参数解析**：简单直接——

- `--sound`：采集时同时播放一段系统提示音（后面解释为什么）；
- 其他参数：当成"采集多少毫秒"，范围限制 500 ~ 30000（太短采不到东西，太长测试跑得慢）。**范围检查很实用**：用户手滑输个 `5` 也不会出问题。

然后是开场横幅：

```cpp
std::printf("  用法: test_audio_capture.exe [采集毫秒数] [--sound]\n");
std::printf("        --sound  采集时自己播放一段系统提示音，用来证明\n");
std::printf("                  loopback 真的能收到声音，而不是只有静音补帧\n");
```

最后那句话点出了 `--sound` 参数的**设计意图**，非常关键，我们 12.3 节细讲。

### 12.1 工具函数：check / computeRms / saveWav

**check：断言计数器**

```cpp
int g_passed = 0;
int g_failed = 0;

void check(bool condition, const char* what)
{
    if (condition) {
        ++g_passed;
        std::printf("  [通过] %s\n", what);
    } else {
        ++g_failed;
        std::printf("  [失败] %s\n", what);
    }
}
```

两个全局变量计数，最后 `return g_failed == 0 ? 0 : 1`——**进程退出码反映测试结果**，CI（持续集成）可以直接用。

**`elapsedMs`**：算从某个时间点到现在过了多少毫秒。

```cpp
long long elapsedMs(std::chrono::steady_clock::time_point from)
{
    return std::chrono::duration_cast<std::chrono::milliseconds>(
               std::chrono::steady_clock::now() - from)
        .count();
}
```

用 `steady_clock` 而不是 `system_clock`——因为 `steady_clock` **单调递增，不受系统时间调整影响**。测时间间隔必须用 steady。

**`computeRms`：音量有多大？**

```cpp
double computeRms(const std::vector<float>& samples)
{
    if (samples.empty())
        return 0.0;
    double sum = 0.0;
    for (float v : samples)
        sum += static_cast<double>(v) * static_cast<double>(v);
    return std::sqrt(sum / static_cast<double>(samples.size()));
}
```

**RMS** 是 **Root Mean Square（均方根）** 的缩写，也就是"先平方、再平均、再开方"。

它是衡量信号强度的**标准指标**。为什么不能简单地取平均值？因为音频波形是正负对称的，**直接平均永远是 0**（正半周和负半周抵消了）。所以先把所有值平方（全变正数），求平均，再开方回到原来的量纲。

- RMS = 0：完全静音；
- RMS ≈ 0.1：正常音量；
- RMS > 1.0：过载（会爆音）。

测试里用 `rms < 0.0005` 判断"是不是静音"，这个阈值非常小（相当于 -66 dB 以下），是很严格的静音判据。

**`saveWav`：把 float 写成 WAV 文件**

这是测试程序里最实用的一段，我们仔细看：

```cpp
bool saveWav(const std::string& path, const std::vector<float>& samples,
             int sampleRate, int channels)
{
    if (samples.empty() || sampleRate <= 0 || channels <= 0)
        return false;

    std::vector<int16_t> pcm(samples.size());
    for (size_t i = 0; i < samples.size(); ++i) {
        const float clamped = std::max(-1.0f, std::min(1.0f, samples[i]));
        pcm[i] = static_cast<int16_t>(clamped * 32767.0f);
    }
```

**第一步：float32 → int16。**

两个动作：

1. **限幅（clamp）**：`std::max(-1.0f, std::min(1.0f, samples[i]))` —— 把值限制在 [-1, 1]。**这一步必须有**，因为 WAV 文件我们写的是 16 位整数，超过 ±1.0 的值乘 32767 会**溢出**（比如 1.5 × 32767 = 49150，超出 int16 范围，截断后会变成一个负数——听感就是"啪"的一声爆音）。
2. **转整数**：乘 32767（1.5 节的公式）。

```cpp
    const uint32_t dataBytes = static_cast<uint32_t>(pcm.size() * sizeof(int16_t));
    const uint16_t blockAlign = static_cast<uint16_t>(channels * sizeof(int16_t));
    const uint32_t byteRate = static_cast<uint32_t>(sampleRate) * blockAlign;
```

**第二步：算 WAV 头需要的三个量。**

- `dataBytes`：数据区总字节数；
- `blockAlign`（块对齐）：一帧占多少字节 = 声道数 × 每采样字节数 = 2 × 2 = 4（立体声 16 位）；
- `byteRate`（字节率）：每秒多少字节 = 采样率 × 块对齐。

这三个是 WAV 头的必填字段，播放器靠它们解读数据。

```cpp
    std::FILE* file = std::fopen(path.c_str(), "wb");
    if (!file)
        return false;

    const auto writeU32 = [file](uint32_t v) { std::fwrite(&v, 4, 1, file); };
    const auto writeU16 = [file](uint16_t v) { std::fwrite(&v, 2, 1, file); };
```

**第三步：开文件 + 两个 lambda 写二进制整数。**

`"wb"` = 写 + **二进制模式**。Windows 上这个 `b` 很重要——文本模式下 `fwrite` 会把 `\n` 自动转成 `\r\n`，二进制数据会被破坏。

两个 lambda 封装"写一个 32 位/16 位整数"。注意 `std::fwrite(&v, 4, 1, file)` 直接写变量的内存——**依赖小端序**。x86 是小端序，WAV 格式也是小端序，所以正好（在大端序机器上这段代码要改）。

```cpp
    std::fwrite("RIFF", 1, 4, file);
    writeU32(36 + dataBytes);
    std::fwrite("WAVE", 1, 4, file);

    std::fwrite("fmt ", 1, 4, file);
    writeU32(16);
    writeU16(1);
    writeU16(static_cast<uint16_t>(channels));
    writeU32(static_cast<uint32_t>(sampleRate));
    writeU32(byteRate);
    writeU16(blockAlign);
    writeU16(16);

    std::fwrite("data", 1, 4, file);
    writeU32(dataBytes);
    std::fwrite(pcm.data(), 1, dataBytes, file);

    std::fclose(file);
    return true;
}
```

**第四步：写 WAV 头 + 数据。**

WAV 文件是 **RIFF（Resource Interchange File Format）** 容器，结构是"一个个块（chunk）嵌套"：

```
┌─────────────────────────────────────────────────┐
│ "RIFF"                                          │  文件标识
│ 36 + dataBytes                                  │  从下一个字节到文件尾的总字节数
│ "WAVE"                                          │  格式标识
├─────────────────────────────────────────────────┤
│ "fmt "                                          │  格式块开始
│ 16                                              │  本块数据区长度（16 字节）
│   1            ← 音频格式：1 = PCM              │
│   channels     ← 声道数                         │
│   sampleRate   ← 采样率                         │
│   byteRate     ← 字节率                         │
│   blockAlign   ← 块对齐                         │
│   16           ← 位深                           │
├─────────────────────────────────────────────────┤
│ "data"                                          │  数据块开始
│ dataBytes                                       │  数据区字节数
│ < PCM 数据 ... >                                │
└─────────────────────────────────────────────────┘
```

那个 `36 + dataBytes` 的来历：WAV 头除了前 8 字节（"RIFF" + 大小字段）之外，剩下的部分是 `4（"WAVE"） + 8（"fmt " + 长度） + 16（fmt 数据） + 8（"data" + 长度）= 36`，再加上数据区大小。

**这个函数值得同学们抄走备用**——它是"把裸 PCM 变成能双击播放的文件"的标准写法，以后调试任何音频代码都用得上。

而且它体现了第 5 讲那个 MJPG 测试的高明思路：**借力系统工具做验证**。把字节写成 WAV，让系统的播放器（或者 Audacity、ffplay）去解码——**能听出声音就是字节级正确的铁证**，比写一百个断言都有说服力。

### 12.2 第一段：枚举设备

```cpp
std::printf("\n--- 1. 枚举设备 ---\n");
{
    const auto speakers = AudioCapturer::listDevices(AudioSourceType::SystemAudio);
    const auto mics = AudioCapturer::listDevices(AudioSourceType::Microphone);

    std::printf("  播放设备（用来抓系统声音）%d 个：\n",
                static_cast<int>(speakers.size()));
    for (size_t i = 0; i < speakers.size(); ++i)
        std::printf("    [%d] %s\n", static_cast<int>(i),
                    speakers[i].empty() ? "<无名字>" : speakers[i].c_str());

    std::printf("  录音设备（麦克风）%d 个：\n", static_cast<int>(mics.size()));
    for (size_t i = 0; i < mics.size(); ++i)
        std::printf("    [%d] %s\n", static_cast<int>(i),
                    mics[i].empty() ? "<无名字>" : mics[i].c_str());

    check(!speakers.empty(), "至少有一个播放设备（系统声音要靠它采）");
}
```

**注意这里用 `{}` 包了一个作用域**——C++ 里可以随时开一个块来限制变量生命周期。这里的 `speakers`、`mics` 只在这一段用，用完就销毁。这是让长函数保持清爽的小技巧。

**只断言"有播放设备"，不断言"有麦克风"**——因为很多台式机、CI 服务器根本没有麦克风。**区分"环境的错"和"代码的错"**，是硬件相关测试的基本修养（第 5 讲强调过）。

打印时 `speakers[i].empty() ? "<无名字>" : ...` 处理无名设备，避免打印出空行让用户困惑。

### 12.3 第二段：采集系统声音（重点）

```cpp
std::vector<std::string> logs;
const auto log = [&logs](const std::string& message) {
    logs.push_back(message);
    std::printf("  [采集器日志] %s\n", message.c_str());
};

std::printf("\n--- 2. 采集系统声音 %d 毫秒 ---\n", durationMs);
std::printf("  建议这段时间放点音乐或者视频，不然采到的会全是静音。\n");
std::printf("  但即便是静音，下面的帧数也必须对得上 —— 那才是重点。\n");

AudioCapturer capturer;
if (!capturer.open(AudioSourceType::SystemAudio, -1, log)) {
    std::printf("\n  打开系统声音失败，后面的测试无法进行。\n");
    ...
    return 1;
}
```

**日志回调的做法**：一个 lambda，把消息存进 vector **同时**打印。存起来是为了后面可能的统一分析，打印是为了实时可见。

**那两行提示至关重要**：

```
建议这段时间放点音乐或者视频，不然采到的会全是静音。
但即便是静音，下面的帧数也必须对得上 —— 那才是重点。
```

它在教用户**正确读取测试结果**：不要看到"RMS = 0"就以为失败了。**帧数对不对，比有没有声音重要得多**——因为帧数验证的是时间轴，那才是音画同步的命脉。

```cpp
const int sampleRate = capturer.sampleRate();
const int channels = capturer.channels();

check(sampleRate > 0, "协商出了有效的采样率");
check(channels > 0, "协商出了有效的声道数");
std::printf("  实际格式: %d Hz, %d 声道\n", sampleRate, channels);
```

验证协商结果有效，并把实际值打出来——这就是"我请求的 vs 我拿到的"在音频侧的对应（音频不能请求，只能接受，所以只看拿到的）。

**`--sound` 参数的用意：**

```cpp
if (withSound) {
    const BOOL started =
        PlaySoundW(L"C:\\Windows\\Media\\Alarm01.wav", nullptr,
                   SND_FILENAME | SND_ASYNC | SND_LOOP);
    std::printf("  [测试] %s正在循环播放系统提示音\n",
                started ? "" : "播放失败（找不到文件），");
}
```

`PlaySoundW` 是 WinMM 的播放函数（所以要链接 `winmm`，见 CMakeLists）。三个标志：

- `SND_FILENAME`：第一个参数是文件名；
- `SND_ASYNC`：**异步播放**，不阻塞当前线程；
- `SND_LOOP`：循环播放。

**为什么要自己放声音？** 这就是 `--sound` 的精髓，请同学们好好体会：

如果我们只采集、不播放，那采到的静音有两种可能：

1. loopback 正常工作，只是系统确实没出声；
2. **loopback 根本没工作**（驱动不支持、设备选错了）。

**这两种情况从结果上看完全一样**（RMS 都是 0），无法区分。

而加上 `--sound` 之后：测试**主动制造声音**，如果还能采到静音，那就只能说明 loopback 有问题。**这就把"无法区分的两种情况"变成了可判定的**。

这是实验设计里一个极其重要的思想：**要验证一个观测系统是好的，不能只观测"没东西"的状态，必须给它一个已知的输入，看它能不能正确响应。** 这叫"**注入已知信号**"，是所有测试方法论的核心。

后来停止播放：

```cpp
if (withSound)
    PlaySoundW(nullptr, nullptr, 0);
```

传 `nullptr`（文件名）+ 0（标志）就是"停止当前播放"。

### 12.4 captureFor：采集循环的统计器

```cpp
struct CaptureReport {
    int chunks = 0;
    int64_t totalFrames = 0;
    long long wallMs = 0;
    bool ptsMonotonic = true;
    bool ptsContinuous = true;
    bool metaConsistent = true;
    int ptsGaps = 0;
    double rms = 0.0;
    std::vector<float> samples;
    std::map<int64_t, int> sizeHistogram;
};
```

这个结构体是"一次采集的体检报告"，九个字段各有用途：

- `chunks`：采到几块；
- `totalFrames`：总帧数（**核心指标**）；
- `wallMs`：实际耗时（用来算"应该有多少帧"）；
- `ptsMonotonic`：pts 有没有倒流；
- `ptsContinuous`：pts 有没有空洞；
- `metaConsistent`：每块的采样率/声道数是否一致；
- `ptsGaps`：空洞出现了几次；
- `rms`：整体音量；
- `samples`：所有采样拼起来（用来存 WAV）；
- `sizeHistogram`：块大小的分布（**诊断利器**，后面讲）。

```cpp
CaptureReport captureFor(AudioCapturer& capturer, int durationMs)
{
    CaptureReport report;

    const int expectedRate = capturer.sampleRate();
    const int expectedChannels = capturer.channels();

    int64_t expectedPts = 0;
    int64_t lastPts = -1;
    bool first = true;

    const auto start = std::chrono::steady_clock::now();
    while (elapsedMs(start) < durationMs) {
        AudioChunkPtr chunk = capturer.nextChunk(50);
        if (!chunk)
            continue;

        const int64_t frames =
            chunk->channels > 0
                ? static_cast<int64_t>(chunk->samples.size() / chunk->channels)
                : 0;
```

**循环条件**：`elapsedMs(start) < durationMs` —— 按墙钟时间跑，不是按块数跑。

**`nextChunk(50)`**：50 毫秒超时。为什么是 50？因为正常情况下每 10 毫秒左右就有一块（或者补帧），50 毫秒足够宽松；同时如果采集器真的卡住了，50 毫秒就能发现，不至于让测试卡死。

**`frames` 的计算**：`samples.size() / channels` —— 又是那个公式（1.4 节）。同学们现在应该对它很熟了。

```cpp
        if (chunk->sampleRate != expectedRate || chunk->channels != expectedChannels)
            report.metaConsistent = false;

        if (lastPts >= 0 && chunk->pts <= lastPts)
            report.ptsMonotonic = false;

        if (!first && chunk->pts != expectedPts) {
            report.ptsContinuous = false;
            ++report.ptsGaps;
        }
        first = false;
        lastPts = chunk->pts;
        expectedPts = chunk->pts + frames;
```

三个检查，一个比一个重要：

1. **`metaConsistent`**：每块自带的采样率/声道数必须和第一次一致。**验证"块自描述"这个契约被兑现了。**
2. **`ptsMonotonic`**：新块的 pts 必须**严格大于**上一块的 pts。时间不能倒流。
3. **`ptsContinuous`**：新块的 pts 必须**恰好等于**"上一块 pts + 上一块的帧数"。也就是说：**块与块之间严丝合缝，既没重叠也没空隙。**

第 3 条是**音画同步的直接检验**。它的逻辑是：

```
第 N 块的 pts  = P，长度 = F 帧
那么第 N+1 块的 pts 必须 = P + F
```

如果不等，就说明中间丢了（或多了）一段——**时间轴断了**。

```cpp
        ++report.chunks;
        report.totalFrames += frames;
        ++report.sizeHistogram[frames];
        report.samples.insert(report.samples.end(), chunk->samples.begin(),
                              chunk->samples.end());
    }

    report.wallMs = elapsedMs(start);
    report.rms = computeRms(report.samples);
    return report;
}
```

统计三件事：块数、总帧数、**块大小分布**。

`sizeHistogram[frames]++` 是一个直方图（histogram）：键是"这一块有多少帧"，值是"出现了几次"。这个数据结构看似不起眼，却是**本测试最巧妙的诊断工具**，我们马上就会看到。

### 12.5 第三段：帧数对账（本测试的核心）

```cpp
std::printf("\n  采到 %d 块，共 %lld 个采样帧\n", report.chunks, ...);
std::printf("  实际耗时 %lld 毫秒，相当于 %lld 帧\n", wallMs,
            static_cast<long long>(static_cast<int64_t>(sampleRate) * wallMs / 1000));
std::printf("  整体 RMS = %.5f %s\n", report.rms,
            report.rms < 0.0005 ? "（几乎是静音）" : "（有声音）");
```

三行概览。注意第二行那个换算：**耗时（毫秒）× 采样率 ÷ 1000 = 理论帧数**。

然后是块大小分布的打印：

```cpp
std::vector<std::pair<int64_t, int>> sizes(report.sizeHistogram.begin(),
                                           report.sizeHistogram.end());
std::sort(sizes.begin(), sizes.end(),
          [](const std::pair<int64_t, int>& a, const std::pair<int64_t, int>& b) {
              return a.second > b.second;
          });

std::printf("  块大小分布（前 5 种）：\n");
for (size_t i = 0; i < sizes.size() && i < 5; ++i) {
    std::printf("    %6lld 帧 (%.1f 毫秒)  x %d 次\n",
                static_cast<long long>(sizes[i].first),
                1000.0 * static_cast<double>(sizes[i].first) / sampleRate,
                sizes[i].second);
}
```

`std::map` 本身按键排序，我们要按**出现次数**排序，所以先拷到 vector 里再 `sort`（`map` 不能直接 sort）。排序后取前 5 打印。

**这个分布为什么是诊断利器？** 看这段：

```cpp
if (report.rms < 0.0005) {
    std::printf("\n  RMS 为 0 有两种可能：系统真的没出声，或者系统音量被拉到 0 了。\n");
    std::printf("  看上面的分布就能分辨：\n");
    std::printf("    - 几乎全是 10 毫秒左右一块 -> 数据全是补的静音，\n");
    std::printf("      loopback 一个包都没给，先把系统音量调大再跑。\n");
    std::printf("    - 出现了 20 毫秒或 1024 帧之类的其它大小 -> loopback 在工作，\n");
    std::printf("      只是那两秒确实安静。\n");
}
```

太漂亮了。同学们看明白了吗？

**静音补帧产生的块，大小必然是"10 毫秒左右"**（因为补帧逻辑每轮最多补到当前时刻，缺口通常在 10 毫秒量级）。

**而 WASAPI 真实给的包，大小是系统缓冲区决定的**，通常是一些"整齐"的数字——比如 480 帧（10 毫秒 @48kHz）、1024 帧、或者 441 帧（10 毫秒 @44.1kHz）……关键是**它们往往不是精确的"10 毫秒"**，而是设备/驱动决定的某个值。

所以：

- 分布里**只有一种大小，且 ≈ 10 毫秒** → 说明**所有的块都是补出来的**，loopback 一点真数据都没给 → **loopback 没工作**；
- 分布里**有多种大小** → 说明 loopback 在给真实包（真实包的大小和补帧块的大小不同），只是内容恰好是静音 → **loopback 工作正常**。

**用一个直方图，把"RMS=0"这个无法区分的状态，变成了可以确诊的信息。** 这就是好的测试设计的样子，同学们请把这个思路记下来。

然后是核心断言：

```cpp
const int64_t expectedFrames =
    static_cast<int64_t>(sampleRate) * wallMs / 1000;
const double errorRatio =
    expectedFrames > 0
        ? std::fabs(static_cast<double>(report.totalFrames - expectedFrames))
              / static_cast<double>(expectedFrames)
        : 1.0;

std::printf("  应有 %lld 帧，实有 %lld 帧，偏差 %.2f%%\n", ...);

check(errorRatio < 0.05,
      "总帧数与真实时间对得上（偏差 < 5%，说明没有静默丢时间）");
check(report.ptsMonotonic, "pts 始终向前，没有倒流");
check(report.ptsContinuous, "pts 连续，块与块之间没有空洞");
```

**这就是开场白那句"采了多久，就该有多少帧"的代码实现。**

- `expectedFrames` = 采样率 × 耗时 ÷ 1000；
- `errorRatio` = |实际 − 理论| ÷ 理论；
- 阈值 **5%**：允许一点点误差（墙钟测量本身有误差、启动和停止各有一两轮循环的抖动），但不允许系统性缺失。

后两个断言（`ptsMonotonic`、`ptsContinuous`）是**更严格的检查**——即使总帧数对得上，如果中途丢了一块又补了一块，总数是平的但时间轴是断的。`ptsContinuous` 专门抓这种情况。

**这三个断言合起来，就是音频采集的完整验收标准。** 同学们以后自己写音频代码，请照抄这三个检查。

### 12.6 第四段：保存成 WAV

```cpp
std::printf("\n--- 4. 保存成 wav ---\n");
check(saveWav("audio_system.wav", report.samples, sampleRate, channels),
      "已保存为 audio_system.wav");
std::printf("  双击 audio_system.wav 应该能听到刚才系统播放的声音。\n");
std::printf("  如果听到的是噪声、断断续续或者速度不对，说明声道/采样率理解错了。\n");
```

最后那句提示列出了**三种典型的听感故障及其原因**，非常有实战价值：

| 听感 | 可能原因 |
|---|---|
| 全是噪声（沙沙声/白噪音） | 声道数或交错布局理解错了（比如把立体声当单声道读） |
| 断断续续（有规律的咔哒） | 丢块了（队列溢出被丢） |
| 速度不对（变快变慢） | **采样率理解错了**（用 48000 播 44100 的数据会变快变尖） |

这个"听感 → 原因"的对照表，是调试音频最实用的经验。同学们以后听到异常声音，可以反过来查这张表。

### 12.7 第五段：麦克风

```cpp
std::printf("\n--- 5. 麦克风 ---\n");
{
    AudioCapturer mic;
    if (!mic.open(AudioSourceType::Microphone, -1, log)) {
        std::printf("  没有可用的麦克风，跳过（不算失败）\n");
    } else {
        const int micRate = mic.sampleRate();
        const int micChannels = mic.channels();
        std::printf("  麦克风格式: %d Hz, %d 声道\n", micRate, micChannels);

        const CaptureReport micReport = captureFor(mic, 1000);
        mic.close();

        check(micReport.chunks > 0, "麦克风采到了数据");

        const int64_t micExpected =
            static_cast<int64_t>(micRate) * micReport.wallMs / 1000;
        const double micError = ...;
        check(micError < 0.10, "麦克风的时间轴也对得上（偏差 < 10%）");

        check(saveWav("audio_mic.wav", micReport.samples, micRate, micChannels),
              "已保存为 audio_mic.wav");
        if (micReport.rms < 0.0005)
            std::printf("  （麦克风没收到声音，对着它说句话再跑一次会更有意思）\n");
    }
}
```

**麦克风打不开不算失败**（`跳过（不算失败）`）——和第 5 讲"没有摄像头不算失败"是同一个原则：**硬件缺席不等于代码有错**。

**阈值放宽到 10%**（系统声音那边是 5%）：因为麦克风通常是单声道、采样率各异，而且不受"静音补帧"影响（麦克风一直在给包），误差来源不同。这个放宽是有考虑的。

### 12.8 第六段：反复开关稳定性

```cpp
std::printf("\n--- 6. 反复开关的稳定性 ---\n");
{
    AudioCapturer loop;
    bool allOk = true;
    for (int round = 0; round < 3; ++round) {
        if (!loop.open(AudioSourceType::SystemAudio, -1, nullptr)) {
            allOk = false;
            break;
        }
        if (!loop.nextChunk(500)) {
            allOk = false;
            break;
        }
        loop.close();
    }
    check(allOk, "连续 open/取数/close 三次都没问题");
}
```

和第 5 讲那个"反复开关三次"的测试是**同样的目的**：抓资源泄漏类 bug。

音频这边专门能抓出的泄漏：

- **COM 对象没 Release** → 第二次 open 时设备状态异常；
- **线程没 join 干净** → 第二次开线程时旧线程还在跑（甚至 `std::terminate` 崩溃）；
- **`mixFormat` 没 `CoTaskMemFree`** → 内存泄漏；
- **事件句柄没 `CloseHandle`** → 句柄泄漏（Windows 上句柄泄漏累积到一定数量后，`CreateEvent` 会失败，然后 open 就失败了）；
- **`CoUninitialize` 调用次数不匹配** → COM 状态错乱。

这类 bug **单次开关完全看不出来**——第一次 open 永远成功。只有反复开关才现形。

### 12.9 结尾横幅

```cpp
std::printf("  音频验收只要记住一条：\n");
std::printf("    采了多久，就该有多少帧。少一帧都会在几分钟后变成音画不同步。\n");
```

再次点题。同学们跑完测试，请记住这句话。

---

## 第十三章：动手环节

理论讲完了，轮到大家动手。请按下面的步骤做实验，每一步都观察输出。

### 13.1 编译

项目用 CMake + Qt6 + FFmpeg。在 Windows 上，建议用 Visual Studio 的生成器：

```bash
# 在 Streaming-tools 目录下
cmake -S . -B build -G "Visual Studio 17 2022" -A x64
cmake --build build --config Release
```

编译产物里会有 `build/tests/Release/test_audio_capture.exe`（路径取决于你的 CMake 配置）。

**编译要点提醒**：

- 新增了 `propsys` 库（在 `src/core/capture/CMakeLists.txt` 里）——`PKEY_Device_FriendlyName` 需要它。如果链接报错 `unresolved external symbol _PKEY_Device_FriendlyName`，说明这个库没链上；
- 测试程序额外链了 `winmm`（`PlaySoundW` 需要）；
- 需要 Windows SDK（`audioclient.h`、`mmdeviceapi.h`、`functiondiscoverykeys_devpkey.h` 都在里面）。

### 13.2 实验一：基本运行

```bash
test_audio_capture.exe
```

默认采集 2000 毫秒系统声音。观察点：

1. **第一段**：你机器上有几个播放设备？几个录音设备？名字认识吗？
   - 常见现象：列表里可能有"扬声器 (Realtek)"、"耳机 (USB Audio)"、还有虚拟设备（比如 "OBS Virtual Audio"）。
2. **第二段开头**：`实际格式: 48000 Hz, 2 声道`（也可能是 44100）——这就是你系统的混音格式。
3. **日志里的"采集格式"那行**：显示的是 float32 还是 int16？同学们记一下，大部分机器是 float32。
4. **采到的块数和总帧数**：对照"相当于 N 帧"那行，看偏差百分比。
5. **块大小分布**：出现了几种大小？换算成毫秒分别是多少？

### 13.3 实验二：注入已知信号（最重要）

```bash
test_audio_capture.exe 3000 --sound
```

这次测试会自己循环播放 `C:\Windows\Media\Alarm01.wav`。

观察点：

1. **RMS 是不是明显大于 0.0005**？如果还是静音，看程序给出的三条排查提示；
2. **块大小分布变了吗？** 对比实验一：
   - 实验一（静音）：分布里可能只有"10 毫秒"一种；
   - 实验二（有声音）：应该出现 WASAPI 真实包的大小。
   - **这个对比就是 12.5 节那个诊断思路的实证**，同学们请务必亲手看一遍。
3. **`--sound` 的第三个断言**（`播放提示音期间，loopback 收到了非零样本`）过了吗？

**如果 `--sound` 依然采不到声音**，按程序提示排查：

- 系统音量是不是 0 或静音了（**最常见**）；
- 默认播放设备是不是 HDMI 或虚拟声卡（声音去了别处，loopback 抓的是默认设备）；
- 这台机器的驱动不支持 loopback。

### 13.4 实验三：用 Audacity 验证 WAV

跑完测试后，工作目录里会有 `audio_system.wav`（和 `audio_mic.wav`）。

**用 Audacity 打开**（免费开源的音频编辑软件）：

1. 菜单 `文件 → 导入 → 音频...`，选择 `audio_system.wav`；
2. 如果 Audacity 问"要按什么格式导入"，说明文件头有问题——但我们写的头是标准 PCM，应该能自动识别为 `48000 Hz, 16-bit, 立体声`；
3. 观察波形：
   - **有起伏** → 采到了真实声音；
   - **一条直线** → 全静音（对照实验二排查）；
   - **波形上下顶到边（削平了）** → 过载爆音（说明 `clamp` 之前的值就超 1.0 了）。

**Audacity 里还能看这几个东西**（同学们试试）：

- `效果 → 音量标准化`：看看原始音量有多大；
- `分析 → 绘制频谱`：如果有声音，频谱上应该能看到 Alarm01 那个提示音的频率成分；
- 底部状态栏显示的项目采样率——**确认它和程序打印的采样率一致**。如果不一致，说明 WAV 头写错了。

### 13.5 实验四：用 ffplay 验证（命令行党）

项目里已经有 FFmpeg，可以直接用 `ffplay`：

```bash
ffplay -f wav -i audio_system.wav
```

或者更"较真"的玩法——**绕过 WAV 头，直接指定裸 PCM 格式播放**（这能验证数据本身，而不依赖我们写的头是否正确）：

```bash
# 如果你的采样率是 48000、双声道、16 位有符号小端
ffplay -f s16le -ar 48000 -ac 2 -i audio_system.wav
```

（注意：这样会把 44 字节的 WAV 头也当成音频数据播出来，开头会有极短的一声"咔"，后面的声音是对的。这个玩法主要用来验证采样率/声道数。）

**如果听到声音变快变尖**：说明你用 48000 播了实际是 44100 的数据——**采样率不对**。这正好印证 12.6 节那张"听感故障表"。

想验证得更彻底，可以用 ffprobe 读文件头：

```bash
ffprobe audio_system.wav
# 输出里应该有：Stream #0:0: Audio: pcm_s16le, 48000 Hz, 2 channels, s16, 1536 kb/s
```

### 13.6 实验五：麦克风

```bash
test_audio_capture.exe 5000
```

第五段会自动开麦克风采 1 秒。**跑的时候对着麦克风说句话**，然后再跑一次不说话，对比两次的 RMS。

观察点：

- 麦克风的格式和系统声音一样吗？（麦克风的采样率常常是 44100 或 16000，声道数常常是 1 或 2）
- 说话时 `audio_mic.wav` 能听清吗？
- 不说话时 RMS 接近 0 吗？（麦克风一般会持续给包，不像 loopback 会断流）

### 13.7 实验六：拉长采集时间，看偏差是否累积

```bash
test_audio_capture.exe 500
test_audio_capture.exe 5000
test_audio_capture.exe 20000
```

三次不同时长，记录每次的"偏差百分比"。

**思考**：如果时间越长偏差越大，说明什么？（提示：说明有系统性的时间丢失，比如补帧补得不够，或者队列在丢块。）如果偏差始终在 1% 左右，说明时间轴是稳的。

### 13.8 实验七：故意制造队列溢出（进阶）

这一项需要改代码玩。把消费端的超时调大（比如把 `captureFor` 里的 `nextChunk(50)` 改成 `nextChunk(500)`），让消费变慢，看看：

- 会不会出现 `pts 跳变`的提示？
- 总帧数偏差会不会变大？

**为什么会这样？** 因为队列容量只有 64，消费慢了之后 `push` 会丢最旧的块——**丢的块在时间轴上留下空洞**，虽然后面补帧会补一部分回来，但补的是静音，而且 pts 会跳变。

这个实验能让同学们亲眼看到"丢音频块"的后果。

---

## 第十四章：排错讨论 —— 常见症状与诊断

这一章我们专门讨论"出问题了怎么办"。同学们以后遇到类似症状，可以回来查这张表。

### 14.1 症状一：完全没声音（RMS = 0）

**可能原因与排查顺序：**

1. **系统音量被拉到 0 或静音**（最常见，占八成）。
   - 检查：任务栏音量图标，打开音量混合器看各个程序。
   - 注意：loopback 抓的是**混音后的总输出**，系统主音量静音了，抓到的就是静音。
2. **默认播放设备选错了**。
   - 检查：Windows 声音设置 → "输出" → 当前默认设备是不是你正在用的那个。
   - 典型坑：插着 HDMI 显示器时默认输出可能变成 HDMI，声音"去了别处"，你听不见，loopback 抓的也是那个没声音的 HDMI 通道。
3. **块大小分布诊断**（12.5 节）：
   - 全都是 10 毫秒一块 → **loopback 根本没给数据**，去查驱动和音量；
   - 有其它大小 → loopback 在工作，只是那几秒确实安静。
4. **驱动不支持 loopback**。
   - 某些老声卡、某些 USB 音频设备的驱动不实现回环。
   - 验证：换一个播放设备试试（用 `open(SystemAudio, 1, ...)` 指定索引）。

### 14.2 症状二：爆音、咔哒声、噪声

这是"能出声但声音不对"的一类，分几种：

**A. 有规律的周期性"咔哒"**

- 表现：每隔一段时间一声脆响。
- 原因：**丢块**（队列溢出被丢，或者消费跟不上）。每丢一块就断一次。
- 排查：看总帧数偏差是否偏大；减小消费端负载；加大队列容量。

**B. 持续的"沙沙"白噪声**

- 原因：**格式解读错了**。典型是把立体声当单声道读（左右声道数据被当成前后相继的采样，听起来就是噪声），或者把 int16 当 float 读（数值完全错乱）。
- 排查：确认 `channels` 的值和 `samples` 的实际布局是否匹配；确认 `detectSampleFormat` 判断对了。

**C. 声音"劈了"、失真**

- 原因：**削顶（clipping）**——信号幅度超过 ±1.0，转 int16 时被截斷。
- 排查：用 Audacity 看波形是不是上下被削平了。解决：采集端降低音量（系统的麦克风加强、扬声器音量），或者下游加限幅器（limiter）。

**D. 声音变快变尖 / 变慢变闷**

- 原因：**采样率搞错了**。用 48000 的参数播 44100 的数据 → 每秒多播 3900 帧 → 变快变尖；反之变慢。
- 排查：`ffprobe` 看文件头，对照程序打印的采样率。
- 这正好呼应 1.11 节：**我们不重采样，但必须如实上报采样率**。如果下游忽略了 `chunk->sampleRate` 而写死用 48000，就会出现这个症状。

### 14.3 症状三：设备被占用 / 打不开

**表现**：`open()` 返回 false，日志显示"激活 IAudioClient 失败"或"初始化音频客户端失败"。

**原因与排查：**

1. **独占模式被别的程序占了**。
   - 如果某个程序（专业音频软件、某些游戏）开了**独占模式**，共享模式的流就开不了。
   - 排查：关掉所有可能占用音频的程序（DAW、ASIO 软件、某些游戏）。
2. **设备被禁用或拔出**。
   - 排查：设备管理器里看设备状态。注意我们枚举时只取 `DEVICE_STATE_ACTIVE`，被禁用的设备根本不会出现在列表里。
3. **音频服务没启动**。
   - Windows Audio 服务（Audiosrv）挂了的话，整个 WASAPI 都用不了。
   - 排查：服务管理器里看"Windows Audio"是否在运行；或者重启。
4. **权限/隐私设置**（麦克风）。
   - Windows 10/11 有"麦克风隐私设置"，禁止应用访问麦克风后，`Activate` 会失败。
   - 排查：设置 → 隐私 → 麦克风。

### 14.4 症状四：帧数对不上（偏差 > 5%）

**表现**：`应有 X 帧，实有 Y 帧，偏差 Z%` 里 Z 超标，或者 `pts 连续` 断言失败。

**原因与排查：**

1. **消费者太慢，队列丢块**。
   - 现象：出现"pts 跳变"提示。
   - 排查：看 `ptsGaps` 的数字；加大队列容量；优化消费端。
2. **程序卡住了（UI 线程阻塞）**。
   - 现象：偏差大，但没有 pts 跳变（因为补帧补回来了，只是补的是静音）。
   - 排查：看块大小分布里有没有"200 毫秒"的大块——那正是 `kMaxSilenceMs` 上限截断的痕迹！**这也说明补帧机制在上限处被截断了**，时间轴因此短了一截。
   - 这个诊断很妙：分布里出现"刚好 200 毫秒"的块，就是"程序曾经卡住超过 200 毫秒"的铁证。
3. **系统休眠/挂起**。
   - 笔记本合盖、系统睡眠时，音频设备停了，但墙钟还在走。唤醒后 `expected` 暴涨，缺口巨大，但被 `kMaxSilenceMs` 截断 → 时间轴永久性短了一截。
   - 这是这个设计的固有局限（见坑点环节）。

### 14.5 症状五：反复开关失败

**表现**：第六段测试失败，第二次或第三次 `open()` 返回 false。

**原因**：资源泄漏。按可能性排序：

1. **COM 对象没 Release 干净**；
2. **工作线程没 join**（第二次开线程时旧的还在跑）；
3. **句柄泄漏**（`CloseHandle` 漏了）；
4. **`CoUninitialize` 次数不匹配**。

**排查手段**：

- 任务管理器里看进程的**句柄数**和**线程数**，反复开关时是否持续增长；
- 用 Visual Studio 的诊断工具看内存；
- 用 `Process Explorer` 看句柄明细。

### 14.6 症状六：编译错误

| 错误信息 | 原因 | 解决 |
|---|---|---|
| `unresolved external symbol _PKEY_Device_FriendlyName` | 没链 `propsys.lib` | 确认 `src/core/capture/CMakeLists.txt` 里有 `propsys` |
| `cannot open file 'functiondiscoverykeys_devpkey.h'` | Windows SDK 版本太旧 | 升级 SDK（需要 Vista 以后版本的 SDK） |
| `unresolved external symbol _PlaySoundW` | 没链 `winmm.lib` | 确认 `tests/CMakeLists.txt` 里 `test_audio_capture` 链了 `winmm` |
| `C2039: 'IAudioClient': is not a member of ...` | 没 include `audioclient.h` | 检查 include |

---

## 第十五章：坑点与改进

老规矩，讲完优点讲局限。这份代码作为教学样本非常优秀，但作为生产代码还有这些坑：

### 15.1 没有重采样，采样率只能照单全收

`open()` 只是记下 `GetMixFormat` 给的采样率，不做任何转换。而 `StreamConfig` 里写死了 `sampleRate = 48000`。

**后果**：如果你的机器混音格式是 44100（很多笔记本的默认），那采到的就是 44100 的数据。下游编码器如果不看 `chunk->sampleRate` 而直接按 48000 处理，**声音会变快变尖**。

**改进方向**：接入 FFmpeg 的 `libswresample`（`swr_alloc` / `swr_convert`），在采集层或编码层做统一重采样到目标采样率。这是下一个 commit 该干的事。

好消息是：`AudioChunk` 自带 `sampleRate` 字段，**契约已经为重采样器准备好了**（1.11 节）。采集层没撒谎，锅在下游。

### 15.2 pts 单位是"帧序号"，基准没和视频统一

`chunk->pts = totalSamples`，单位是**采样帧的序号**（0, 1, 2, ... N）。

而视频侧（commit 5）的 `CameraRawFrame::pts` 是**帧序号**（0, 1, 2, ... M），`VideoFrame::pts` 也还没定单位。

**问题**：两个 pts 的**量纲完全不同**。音频 pts 走 48000/秒，视频 pts 走 30/秒。下游做音画同步时，必须把它们换算到**同一个时间基准**（通常是微秒或 FFmpeg 的 time_base）。

**改进方向**：项目应该明确一个统一的时间单位（建议微秒），采集层负责换算：

```
音频 pts（微秒）= 帧序号 × 1,000,000 ÷ 采样率
```

或者约定 `AudioChunk::pts` 直接存微秒。**这件事必须尽早定下来**，否则越往后越难改（多个模块已经在按各自的假设写了）。

### 15.3 没用 WASAPI 提供的硬件时间戳

`GetBuffer` 的第 4、5 个参数（**设备位置**和**性能计数器值**）我们传了 `nullptr`，直接丢弃了。

这两个参数是**声卡硬件给出的精确时间戳**，比我们用 `QueryPerformanceCounter` 估算准确得多——它们反映的是"这一包数据实际对应哪个时刻"，不受线程调度、程序卡顿影响。

**现在的补帧算法是"用墙钟估算"**，本质上是个近似：

- 它假设"采集速率恒定等于采样率"；
- 它假设"程序没卡"；
- 它假设"系统没休眠"。

而硬件时间戳能直接告诉我们"真实采到了多少、缺了哪一段"。

**改进方向**：用 `GetBuffer` 的 `pu64DevicePosition` 和 `pu64QPCPosition` 建立"设备时钟 ↔ 墙钟"的映射，用它来算缺口，精度会高一个数量级。这是专业级实现（OBS 就是这么做的）。

### 15.4 静音补帧的两个固有局限

**局限一：程序卡死会永久性丢失时间。**

`kMaxSilenceMs = 200` 意味着一次最多补 200 毫秒。如果程序卡了 2 秒（比如 UI 线程阻塞、或者系统休眠），那 1.8 秒就**永久丢了**——时间轴从此短了 1.8 秒。

**这是有意的设计权衡**：补太多静音会让队列瞬间爆满、延迟飙升。但代价是卡死之后音画永久错位。

**改进方向**：区分"短暂卡顿"和"长时间中断"。长时间中断（比如超过 1 秒）应该**通知上层**（"音频流断了"），让上层决定是重开流还是重置时间轴，而不是默默补 200 毫秒了事。

**局限二：补帧用的是墙钟，不是音频设备时钟。**

声卡的采样时钟和 CPU 的墙钟**不是同一个时钟源**，它们之间有微小的频率差（典型值几十 ppm，也就是百万分之几十）。

**后果**：长时间运行（比如几小时）后，墙钟估算的"应有帧数"和声卡实际产生的帧数会缓慢偏离，导致补帧越补越多（或者该补的时候不补），最终产生**缓慢累积的音画漂移**。

**改进方向**：用 15.3 节的硬件时间戳做长期校准（PLL，锁相环），或者直接以声卡的时钟为准。专业音视频系统都会做这个"时钟驯服（clock servo）"。

### 15.5 队列丢块与时间账本的矛盾

`BoundedQueue` 满了会丢最旧的块，但我们开头说过"音频一块都不能丢"。

**这两个其实不冲突**，同学们想清楚：

- `pushChunk` 里 `totalSamples` **照常推进**（丢块不影响时间账本）；
- 所以丢掉的是"队列里的积压数据"，**时间轴依然是连续的**；
- 后面补帧机制会按时间账本把缺口补上（补成静音）。

**真正的问题是**：丢掉的那段真实音频**变成了静音**——观众听到的是"声音断了一下"。

**改进方向**：

1. 队列容量加大（64 块 × 10 毫秒 = 640 毫秒，其实已经不小了）；
2. 丢块时**记一个计数器**（`droppedChunks`），暴露给上层做监控和告警；
3. 消费端跟不上时，考虑**降采样或降质**（比如丢掉一半的块但保证时间轴对）而不是让它自然溢出。

### 15.6 每包一次堆分配

```cpp
std::vector<float> converted;
convertSamples(data, sampleCount, sampleFormat, converted);
```

`converted` 在**每个包**都构造一次，触发一次堆分配 + 释放。按每秒 100 包算，就是每秒 200 次堆操作。

**后果**：内存碎片 + 偶发的分配延迟（虽然对现代分配器来说不算大负担，但在实时音频路径上任何不确定性都是隐患）。

**改进方向**：把 `converted` 提为 `Impl` 的成员（或者 `workerLoop` 的局部变量，在循环外），每次 `resize` 复用——`vector::resize` 到更小的尺寸不会释放容量，所以复用是完全可行的。

### 15.7 close() 的线程假设

`close()` 里的 `CoUninitialize()` 只反初始化**调用 `close()` 的那个线程**的 COM。而 `comOwned` 是在 `open()` 的线程上置位的。

**如果 `open()` 在主线程、`close()` 在另一个线程**（这在多线程程序里很常见），那么：

- 反初始化的是错误的线程；
- 主线程的 COM 永远没被反初始化。

**改进方向**：把"哪个线程初始化了 COM"记下来（`std::thread::id`），`close()` 时检查当前线程是否一致，不一致就跳过（或者文档明确要求"open 和 close 必须在同一线程"）。

同理，`join()` 也有类似假设——不过 `join` 本身是线程安全的（任何线程都能 join 一个 joinable 的 thread）。

### 15.8 close() 在工作线程里调用会死锁

如果有人（比如未来加的错误处理路径）在工作线程里调 `close()`，那么 `impl_->worker.join()` 就是**自己等自己**——永久死锁。

**改进方向**：`close()` 开头加一句：

```cpp
if (std::this_thread::get_id() == impl_->worker.get_id())
    return;   // 或者在文档里明确禁止
```

或者用 `std::async`/状态机重构，避免这种自锁可能。至少应该在头文件里写明这个约束。

### 15.9 getter 无锁

`sampleRate()` / `channels()` 直接读 `impl_->sampleRate`，没有加锁。

虽然这些值只在 `open`/`close` 时变，但严格来说是数据竞争（C++ 标准层面）。而且更实际的问题是：**`close()` 之后 Impl 还在，但值被清零了**——如果另一个线程正好在读，可能读到 0。

**改进方向**：要么加互斥锁，要么改成 `std::atomic<int>`。成本很低。

### 15.10 不支持设备热插拔和格式变化

`open()` 时协商的格式，在整个采集期间**假定不变**。但如果用户：

- 拔掉了耳机（默认设备变了）；
- 在系统设置里改了采样率（混音格式变了）；
- 设备被禁用；

我们**完全不知道**——工作线程会继续用旧格式解读新数据，或者干脆一直采不到数据。

**改进方向**：WASAPI 提供 `IMMNotificationClient` 接口，可以注册回调接收设备变更事件（`OnDefaultDeviceChanged`、`OnDeviceStateChanged`）。生产代码应该接上，收到事件后通知上层"设备变了，请重开"。

### 15.11 不支持多声道下混

如果系统的混音格式是 5.1 或 7.1（`nChannels` = 6 或 8），我们会**照单全收**，`AudioChunk::channels` 就是 6 或 8。

**后果**：下游的 AAC 编码器通常只支持单声道/立体声。收到 6 声道数据要么报错，要么编出来播放器解不了。

**改进方向**：在采集层做**下混（downmix）**——把多声道按标准系数（ITU-R BS.775）混成立体声。或者至少把声道数如实报给上层，让上层决定。

### 15.12 没有音量和增益控制

没有提供任何"调音量"的接口。麦克风采集经常需要增益调节（Windows 的"麦克风加强"），系统声音采集可能需要衰减。

**改进方向**：通过 `IAudioEndpointVolume`（设备音量）或 `ISimpleAudioVolume`（会话音量）控制。或者更简单：在 `convertSamples` 之后乘一个增益系数（这个最灵活，但要注意别乘出爆音）。

### 15.13 totalSamples 命名误导

第 6.1 节提过：`totalSamples` 实际存的是**帧数**（`sampleCount / channels`）。

这个命名会让读代码的人（包括几个月后的自己）误以为它是采样数，进而在计算中多除或少乘一次 `channels`。

**改进方向**：改名 `totalFrames`。一行改动，省掉未来的一个 bug。同学们写自己的代码时，名字要经得起推敲——**名字是代码里最重要的注释**。

### 15.14 其余小坑

- **`listDevices` 失败无日志**：静态函数没有 `LogCallback`，枚举失败时静默返回空列表，排查时无从下手；
- **事件模式降级只针对 loopback**：麦克风理论上也可能遇到（罕见），降级路径没覆盖；
- **`kBufferDuration = 100 毫秒`偏保守**：延迟较高（对需要低延迟监听的场景不友好），但换取了稳定性，这个取舍对直播是合理的；
- **`padSilence` 和 `pushChunk` 有大量重复代码**：可以合并，但当前分开写可读性更好（7.2 节讨论过）；
- **没有单元测试**：测试程序依赖真实硬件，无法在 CI 里跑（只能跑到"枚举设备"这一段）。

---

## 第十六章：小结

这一讲我们完成了采集层的最后一块拼图。回顾主线：

1. **音频是连续采样流，视频是离散帧**。这个本质差异决定了：音频必须用队列（不能丢），视频可以用单槽（丢旧保新）。**音频丢一块，时间轴就短一截，而且会一直累积。**
2. **音频三要素**：采样率（每秒多少点，由采样定理决定下限）、位深（每点多精细）、声道数（几路，交错存放）。**采样数 = 帧数 × 声道数**——这个公式今天用了三次（一个除、一个乘、一个在测试里）。
3. **WASAPI 四个对象**：`IMMDeviceEnumerator`（找设备）→ `IMMDevice`（一个设备）→ `Activate` → `IAudioClient`（流控制器）→ `GetService` → `IAudioCaptureClient`（取数据）。
4. **共享模式 + loopback**：不独占设备，靠音频引擎的"回环分路"偷听系统声音。所以"抓系统声音"要开 `eRender` 端点并加 `LOOPBACK` 标志——这是最反直觉的一处。
5. **格式是"问出来的"不是"指定的"**：`GetMixFormat` 给什么我们收什么，`detectSampleFormat` 判断是 float32 还是 int16，`convertSamples` 统一成 float32。**不支持的格式明确失败，绝不硬扛。**
6. **静音补帧是音频独有的保底机制**：loopback 在系统静音时一个包都不给，我们用高精度时钟对账，缺口超过 10 毫秒就补静音，一次最多补 200 毫秒。**没声音也要凑够时间。**
7. **三级降级策略**：事件回调 → 轮询（loopback 不支持时）；`SetEventHandle` 失败 → 只打日志不失败（因为超时能兜底）；不支持的格式 → 明确失败。**分级处理错误，是可维护性的关键。**
8. **关停顺序就是一切**：置 `stopping` → `SetEvent` 唤醒 → `join` 等线程结束 → `Stop` → 清队列 → 逆序释放 COM → `CoTaskMemFree` / `CloseHandle` → 复位状态 → `CoUninitialize`。
9. **测试即教案**：注入已知信号（`--sound`）区分"静音"和"没工作"；块大小直方图做诊断；三个时间轴断言（总帧数、pts 单调、pts 连续）；借系统工具（Audacity/ffplay/WAV 文件）做字节级验证。

至此，采集层的三个源（屏幕、摄像头、音频）**全部齐了**，而且都统一到了项目约定的数据结构上：

```
ScreenCapturer  ──▶ VideoFrame（BGRA，顶向下）
CameraCapturer  ──▶ CameraRawFrame ──▶ CameraFrameConverter ──▶ VideoFrame
AudioCapturer   ──▶ AudioChunk（float32，交错）──▶ [重采样器，待补]
```

下一阶段，我们就要把这些产出的 `VideoFrame` 和 `AudioChunk` 送进**编码链路**——FFmpeg 编码、FLV/RTMP 封装、以及最终的音画同步。**那时候，今天讲的"时间轴不能断"就会真正发挥作用。**

---

## 思考题

1. 为什么说"音频丢一块的后果比视频丢一帧严重得多"？请从"是否会累积"的角度论证，并说明为什么视频侧可以用单槽覆盖、音频侧必须用队列。

2. 某机器的混音格式是 44100 Hz、2 声道、float32。请计算：
   - 采集 3 秒理论上应该有多少帧、多少采样、多少字节（WASAPI 侧）？
   - 如果 `kBufferDuration` 是 100 毫秒，缓冲区能装多少帧？
   - 如果系统静音了 500 毫秒，补帧机制会补几次、每次补多少帧？（提示：注意 `kMaxSilenceMs` 的截断。）

3. `detectSampleFormat` 里为什么要先检查 `format->cbSize >= 22` 再 `reinterpret_cast`？如果去掉这个检查，在什么情况下会出问题？请从内存安全的角度说明。

4. `workerLoop` 里为什么内层要用 `while` 抽干缓冲区，而不是 `if` 取一包就走？如果改成取一包就走，会发生什么现象？

5. 请论证 `WaitForSingleObject(event, kPollIntervalMs)` 这一行为什么能同时支持"事件模式"和"轮询模式"两种工作方式。如果 `kPollIntervalMs` 改成 1000，两种模式各会发生什么？

6. 测试程序里的 `sizeHistogram`（块大小分布）为什么能区分"loopback 没工作"和"系统确实静音"这两种情况？请设计一个反例：在什么情况下这个诊断会失效？

7. `open()` 里 `SetEventHandle` 失败只打日志不返回失败，而 `GetService` 失败就返回失败。请分析这两个决定的理由，并说明判断"某个错误是否致命"的一般原则。

8. 如果一个用户报告"我采到的声音变快变尖了"，请列出至少三种可能的原因，并给出对应的排查步骤。（提示：想想采样率、声道数、重采样。）

9. `close()` 里如果把 `audioClient->Stop()` 移到 `worker.join()` **之前**，会发生什么？请从线程安全的角度分析。

10. 请设计一套改进方案，让 `AudioChunk::pts` 和视频侧的 `pts` 统一到同一个时间基准（比如微秒）。需要考虑：换算公式放在哪一层？谁来定义基准？已有的代码要改哪些地方？

11. 同学们注意到 `totalSamples` 存的是帧数而不是采样数。请找出代码里所有用到它的地方，确认每一处的单位都是一致的，并说明如果有人误以为它是采样数、在 `padSilence` 里多乘一次 `channels`，会出现什么现象。

12. 如果要把这个模块改成"同时采集系统声音和麦克风并混音"，你会怎么设计？需要考虑哪些问题？（提示：两个流的采样率可能不同、时钟不同步、混音要在 float 域做、会不会过载。）

---

**本讲配套文件**：

- `src/core/capture/AudioCapturer.h`（31 行）
- `src/core/capture/AudioCapturer.cpp`（501 行）
- `tests/test_audio_capture.cpp`（400 行）
- `src/core/capture/CMakeLists.txt`（新增两个源文件 + `propsys` 库）
- `tests/CMakeLists.txt`（新增 `test_audio_capture`，链 `winmm`）
- `.gitignore`（新增 `*.wav`）

**下一讲预告**：音频重采样与音画同步 —— 把 `AudioChunk` 的采样率统一到 `StreamConfig` 的 48000、声道下混成立体声、pts 换算到统一时间基准，然后送进编码器。

---

## 附：本讲"五章对照索引"

| 标准章节 | 在本讲的位置 | 一句话要点 |
|---|---|---|
| **核心原理** | 开场白 + 第一章知识点预习 | 连续采样流 vs 离散帧；音频丢块会累积成音画不同步，所以必须用队列不能丢；WASAPI 共享模式 + loopback 回环偷听系统声音；共享模式下格式是"问出来的"不是"指定的"；静音补帧保住时间轴 |
| **用到的技术 / 知识点** | 第一章全部小节 | PCM 与采样率/位深/声道；采样定理（Nyquist）与混叠；交错布局与"采样 vs 帧"；float32 与 PCM16 的换算（32767/32768 不对称的由来）；WASAPI 四对象（IMMDeviceEnumerator/IMMDevice/IAudioClient/IAudioCaptureClient）；共享模式 vs 独占模式；loopback vs 麦克风（eRender/eCapture）；缓冲区与 packet 三步走（GetNextPacketSize/GetBuffer/ReleaseBuffer）；`AUDCLNT_BUFFERFLAGS_SILENT`；事件回调 vs 轮询与降级；重采样概念；COM 四用法（CoCreateInstance+__uuidof、Activate、CoTaskMemFree、线程相关初始化）；AudioChunk 与 BoundedQueue 契约回顾 |
| **实现细节** | 第二~十一章 | 头文件契约（与 CameraCapturer 的接口对称性、PIMPL 隔离 Windows 头文件）；工具函数群（safeRelease/toUtf8/ComScope/friendlyNameOf+PROPVARIANT+两个手写 SubFormat GUID）；`detectSampleFormat` 三种 WAVEFORMATEX 变体与 cbSize 防御；`convertSamples` 的 memcpy 与乘系数两条路；Impl 五组状态；`pushChunk`/`padSilence` 两条入队路径；`workerLoop` 全流程（COM 初始化 + QPC 对账 + 抽干循环 + 补帧算法）；`open()` 十二步（含 loopback 事件降级）；`close()` 关停顺序；`listDevices` 与 ComScope |
| **关键方法说明** | 各章方法表 + 第八章数据流图 | `listDevices(type)` / `open(type, index, log)` / `nextChunk(timeoutMs)` / `close()` / `sampleRate()` / `channels()`；内部方法 `workerLoop` / `pushChunk` / `padSilence` / `detectSampleFormat` / `convertSamples` / `friendlyNameOf`；open 的降级链；从音频引擎到消费方的完整数据流图 |
| **缺陷与局限** | 第十四章排错 + 第十五章"坑点与改进" | 无重采样（采样率照单全收）；pts 单位未与视频统一；丢弃了 GetBuffer 的硬件时间戳；补帧用墙钟导致长期漂移、程序卡死永久丢时间、kMaxSilenceMs 截断；队列丢块与时间账本的矛盾；每包一次堆分配；close 的线程假设（open/close 跨线程会错）；工作线程内调 close 会死锁；getter 无锁；不支持热插拔（未接 IMMNotificationClient）；不支持下混（5.1/7.1 照单全收）；无音量增益控制；totalSamples 命名误导；listDevices 失败无日志 |
