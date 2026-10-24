# 第十课：FLV 封装与多目标分发 —— 把编码好的 AVPacket 打包成 FLV，同时分发到多个目标

> **对应提交**：`0ccd1085026a52b7ba5c9d804f903555f7266aa8`（简称 `0ccd108`）
> **提交时间**：2026-10-20 19:24
> **提交信息原文**：把编码好的 AVPacket 封装成 FLV，同时分发到多个目标
> **改动规模**：6 个文件，**+1025 行**（新增 `src/core/output/` 模块共 442 行 + 测试 `test_output.cpp` 559 行 + 三处 CMake 挂接 24 行）
> **本课核心代码**：
> - `Streaming-tools/src/core/output/StreamOutput.h`（53 行）
> - `Streaming-tools/src/core/output/StreamOutput.cpp`（389 行）
> - `Streaming-tools/tests/test_output.cpp`（559 行）
> **课前准备**：第 2 课（FLV 封装第一次跑通）、第 6/7 课（`VideoEncoder` / `AudioEncoder` 能吐出 `AVPacket`，并且各自能交出 `codecParameters()` 和 `timeBase()`）
> **课后产物**：`output_single.flv`、`output_a.flv`、`output_b.flv`、`output_isolated.flv` 四个能直接双击播放的文件，外加一张 `output_first_frame.bmp`
> **本课一句话**：**编码一次，封装 N 次，任何一路炸了都不许影响其它路。**

---

## 目录（这节课我们怎么走）

| 章节 | 内容 |
|---|---|
| 第 0 章 | 开场白：封装器是"打包发货"的工位；多目标分发的业务场景 |
| 第 1 章 | 知识点预习（上）：FLV 封装要点回扣（毫秒时间基、`stream_index`、header/trailer、全局头） |
| 第 2 章 | 知识点预习（中）：`avformat` 输出上下文管理、RTMP url 与本地文件 url 的统一抽象 |
| 第 3 章 | 知识点预习（下）：一个 packet 写多处（引用计数）、写失败隔离、AAC 负时间戳 |
| 第 4 章 | 逐行教学（一）：`StreamOutput.h`（53 行）与 PImpl 惯用法 |
| 第 5 章 | 逐行教学（二）：`StreamOutput.cpp`（389 行），七个内部函数 + 十二个公开接口 |
| 第 6 章 | 逐行教学（三）：三处 CMakeLists 改动（顶层 / `src/core/output` / `tests`） |
| 第 7 章 | 逐行教学（四）：`tests/test_output.cpp`（559 行）与四个测试用例 |
| 第 8 章 | 动手环节：本地文件 + rtmp 双目标实验（六个改造实验） |
| 第 9 章 | 排错手册：连不上、写失败、时间戳乱跳、只有视频没声音 |
| 第 10 章 | 小结、速查表、局限与思考题（8 题含答案）、下节课预告 |
| 附 | 本讲「五章对照索引」 |

> **老师的课前提醒**：这一课的"新 API"其实很少，`StreamOutput` 用到的 FFmpeg 函数你在第 2 课基本都见过。真正的**新知识集中在三个方面**：① 怎么把"一个输出"抽象成"N 个输出"并且互不干扰；② `AVPacket` 的**引用计数**到底是什么、为什么一份数据能安全地写进 N 个互不认识的 `AVFormatContext`；③ 怎么用**没有任何服务器**的方式，把"推流"这件事测出来。第 3 章请务必听懂，那是本课的灵魂；第 7 章的测试代码请务必读完，那是本项目"工程素养"含量最高的一段。

---

# 第 0 章 开场白：流水线的最后一公里，是"打包发货"

## 0.1 先回头看看：我们的流水线建到哪儿了

同学们，我们先把镜头拉远，看看整个项目这十次提交到底在搭什么。

| 课次 | 提交 | 一句话 | 流水线上对应哪一段 |
|---|---|---|---|
| 第 1 课 | `20b267a1` | 项目骨架与构建系统 | 盖厂房、通水通电 |
| 第 2 课 | `df3603c8` | 假画面 → H.264 → FLV → 文件 | **第一次把整条线跑通（walking skeleton）** |
| 第 3 课 | `a4453bb1` | `StreamConfig` / `VideoFrame` / `AudioChunk` / `BoundedQueue` | 定"零件规格"和"传送带" |
| 第 4 课 | `81eaa1f5` | DXGI / GDI 抓屏 | ★ 原料采购（视频） |
| 第 5 课 | `281a22be` | DirectShow 摄像头 | ★ 原料采购（视频，另一路） |
| 第 6 课 | `e92cbd3` | 摄像头五种格式 → BGRA | 原料预处理（统一规格） |
| 第 7 课 | `1ce7f62` | WASAPI 抓声音 → `AudioChunk` | ★ 原料采购（音频） |
| 第 8 课 | `66e0768` | 两个编码器，压成 `AVPacket` | ★ 加工车间 |
| **第 10 课（本次）** | **`0ccd108`** | **FLV 封装 + 多目标分发** | **★ 打包发货** |
| 第 11 课 | `31c3f57` | 三线程两队列，真正能推流 | 给流水线装马达（并行化） |
| 第 12 课 | `3b22c69` | Qt 主界面 | 盖个调度室（外壳） |
| 第 13 课 | `bafb4ea` | 实时预览 | 装监控屏幕 |
| 第 14 课 | `c2b8021` | 断线重连 | 加应急预案 |
| 第 15 课 | `7eb889b` | 配置持久化 | 把配方写进本子 |

看清楚了吗？**我们今天要装的，是流水线的最后一道工位。**

## 0.2 今天的工位：封装器 = "打包发货"

我再给大家把这个比喻讲透一点。想象一个电商仓库：

```
┌──────────┐   ┌──────────┐   ┌──────────┐   ┌──────────┐   ┌──────────┐
│  采购部   │ → │  质检    │ → │  加工车间 │ → │ 打包发货  │ → │   物流    │
│ 采集模块  │   │ 格式转换 │   │  编码器   │   │ 封装器    │   │  网络/磁盘│
│ ScreenCapturer │ CameraFrameConverter │ VideoEncoder/AudioEncoder │ ★StreamOutput │ TCP/文件
└──────────┘   └──────────┘   └──────────┘   └──────────┘   └──────────┘
   原料：像素       规格统一        半成品：压缩包      成品：FLV Tag      送到目的地
```

**加工车间（编码器）交出来的东西，是一包包"压缩好的内容"**——H.264 的一帧、AAC 的一段。它们就像工厂里做好的商品，**本身没有外包装、没有快递单**。

**打包发货工位（封装器 = muxer，复用器）干三件事**：

1. **装箱**：把商品装进统一规格的箱子（FLV Tag），箱子上写明"这是视频还是音频"、"长度多少"；
2. **贴单**：写上"寄件时间"（时间戳 pts/dts）、"属于哪一票货"（`stream_index`）；
3. **附说明书**：在最前面塞一份"本批货物的规格说明"（FLV 的 `onMetaData` + AVC sequence header，里面装着 SPS/PPS）。

**没有这道工位会怎样？** 第 2 课我们讲过：编码器吐出来的是裸的 NAL 单元序列，播放器拿到它，既不知道这是什么编码，也不知道每一包什么时候播，更不知道有几条流。**它就像一堆散装零件寄到客户手上——客户根本没法用。**

所以今天的第一个关键词是 **mux（复用）**——把多路压缩流"复用"进一个容器。反过来叫 **demux（解复用）**。

## 0.3 多目标分发的业务场景：为什么要一份数据发多处

好，现在老师问一个问题：**打包好的货，是不是只寄给一个客户？**

我们看看真实的直播场景，同学们就明白了。

### 场景一：多平台同播（simulcast）

一个游戏主播，想同时开播：

```
                     ┌──→ rtmp://send.douyu.com/live/xxxxx   （斗鱼）
  我的电脑（编码一次）─┼──→ rtmp://live-push.bilivideo.com/... （B 站）
                     └──→ rtmp://push.example.com/live/yyyyy  （自有 CDN）
```

这是最典型的诉求。主播不可能开三个 OBS、编码三遍——**CPU 会直接烧掉**（x264 编码 1080p60 一路就要吃掉大半个 CPU，三路就是三倍）。

正确做法是：**只编码一次，把同一个 `AVPacket` 复制给三个输出上下文**。这就是"多目标分发"（multi-target distribution / fan-out）的核心。

> **老师插话**：这里"复制"是加了引号的。我们**不会真的把几 KB 的数据 memcpy 三份**——那样也浪费。真正的做法是"**引用计数**"，三个输出共享同一块内存，只是各自有一个引用。这是第 3 章的重点。

### 场景二：直播 + 本地录像

```
                     ┌──→ rtmp://live.bilivideo.com/...  （推给观众）
  我的电脑（编码一次）─┤
                     └──→ D:/Record/2026-10-20.flv        （本地留档）
```

这个场景在业内叫 **"边推边录"**。它的好处是：录像内容跟观众看到的是**同一份码流**，质量完全一致，而且**零额外编码成本**（只是多写一次磁盘）。

很多主播下播后要剪"高光时刻"，用的就是这份本地录像。

### 场景三：主备双推（热备）

```
                     ┌──→ rtmp://主线路/...   （正常推）
  我的电脑（编码一次）─┤
                     └──→ rtmp://备用线路/... （同时推，随时可切）
```

商业直播（赛事、发布会）常用。主线路挂了，CDN 侧立刻切到备用线路，观众几乎无感。**这要求两路的码流完全一致**——所以必须是"一次编码、两路输出"，而不是"编码两次"。

### 场景四：开发与联调

这个功能对我们开发者还有一个巨大的**隐藏价值**，先剧透一下，第 7 章会详细讲：

> **RTMP 的 URL 和本地文件的 URL，在 FFmpeg 里是同一套抽象。**
>
> 所以"推流到斗鱼"和"写成一个 .flv 文件"，**代码路径完全一样，只是一个字符串不同**。
>
> 这意味着：**我们可以在没有服务器、没有网络、没有直播账号的情况下，把"多目标分发"这件事完整地测出来。**

这就是我们今天的测试 `test_output.cpp` 能自动跑通的原因——它把"第二个目标"写成 `output_b.flv`（本地文件），把"死掉的目标"写成 `rtmp://127.0.0.1:1/live/nothing`（一个肯定没人监听的端口）。

## 0.4 一个关键认知：多目标 ≠ 编码多次

同学们，请把这句话抄在笔记本第一页：

> **多路输出是在"封装层"分叉的，不是在"编码层"分叉的。**

```
                    ✗ 错误做法                          ✓ 正确做法
            ┌─→ 编码器 A ─→ 封装 ─→ 目标1          ┌─→ 封装(FLV) ─→ 目标1
   采集 ────┼─→ 编码器 B ─→ 封装 ─→ 目标2   采集 ─→ 编码器 ─┼─→ 封装(FLV) ─→ 目标2
            └─→ 编码器 C ─→ 封装 ─→ 目标3                └─→ 封装(FLV) ─→ 目标3
                                                     （同一个 packet，引用计数）
            CPU ×3，画面还可能有细微差别              CPU ×1，三路码流逐字节相同
```

左边错在哪儿？

1. **浪费 3 倍 CPU**（编码是整个直播系统最贵的一步）；
2. **三路码流不一致**——编码器是有状态机和时间依赖的，三个独立编码器即使参数一样，产出的 GOP 边界、码率波动也会不同，主备切换时会花屏；
3. **内存也翻三倍**（每个编码器都有一堆参考帧缓冲）。

右边的做法，就是今天的 `StreamOutput`：

```cpp
// 一个 packet 进来，写进 N 个目标
for (Target& target : targets) {
    AVPacket* copy = av_packet_clone(&packet);   // 只加引用，不拷数据
    av_packet_rescale_ts(copy, timeBase, stream->time_base);
    copy->stream_index = stream->index;
    av_interleaved_write_frame(target.ctx, copy);
    av_packet_free(&copy);
}
```

看见那个 `for` 循环了吗？**整个"多目标分发"的实现，本质上就是这么一个循环。** 但要把这个循环写对，你得解决三个工程问题：

| 问题 | 解决手段 | 在第几章讲 |
|---|---|---|
| 一个包怎么安全地给 N 个互不相干的输出？ | `av_packet_clone`（引用计数） | 第 3 章 + 5.4 节 |
| 其中一个目标写失败了怎么办？ | 就地隔离：标记断开、关资源、`continue` | 第 3 章 + 5.4 节 |
| 怎么知道每一路的健康状况？ | 每个 target 自带统计与 `lastError` | 第 4 章 + 5.1 节 |

## 0.5 本次提交改了哪 6 个文件

老规矩，先看地图：

```
Streaming-tools/
├── CMakeLists.txt                    ← 改：+1 行（把新的 output 目录挂进构建）
├── src/core/output/
│   ├── CMakeLists.txt                ← 新增 14 行（定义 core_output 静态库）
│   ├── StreamOutput.h                ← 新增 53 行  ★ 对外契约
│   └── StreamOutput.cpp              ← 新增 389 行 ★ 今天的绝对主角
├── tests/
│   ├── CMakeLists.txt                ← 改：+9 行（加 test_output 可执行目标）
│   └── test_output.cpp               ← 新增 559 行 ★ 今天的"验收报告"
└── src/core/{common,capture,encode}/ ← 一行没动（这就是分层的好处）
```

一共 **1025 行新增，0 行删除**。同学们注意最后一行：**我们对已有代码一个字都没改**。

这一点非常值得说。为什么能做到？因为第 8 课（`66e0768`）的两个编码器留了三个干净的"插头"：

```cpp
class VideoEncoder {
    AVRational timeBase() const;                     // 插头 1：我的时间基是多少
    const AVCodecParameters* codecParameters() const; // 插头 2：我的编码参数是什么
    std::vector<AVPacketPtr> encode(const VideoFrame&); // 插头 3：我吐出来的包
};
```

`StreamOutput` 只要插上这三个插头就能干活，**它完全不需要知道画面是抓屏来的还是摄像头来的，也不需要知道编码器里面是 x264 还是别的什么**。

> **老师点评**：这就是**面向接口编程**的威力。当一个新模块能"一行不改老代码"地接进系统，说明之前的分层是对的。同学们以后评审代码时，可以用这个当一条硬指标：**"新增一个功能，需要改多少行已有代码？"** 答案越接近 0，架构越健康。

## 0.6 老师先剧透：`StreamOutput` 只有 12 个成员函数

别被 389 行吓到。我们把它拆开看，其实非常清爽：

### 内部（`Impl`，藏在 .cpp 里，外面看不见）

| 序号 | 名字 | 一句话职责 | 行数（约） |
|---|---|---|---|
| 0 | `avErrorText(err)` | FFmpeg 错误码 → 中文可打印字符串 | 6 |
| 1 | `kFlvTimeBase` / `kIoTimeout` | 两个常量：1/1000 时间基、5 秒 IO 超时 | 2 |
| 2 | `struct Target` | 一个输出目标要记的**全部**状态 | 15 |
| 3 | `Impl::release()` | 一键全拆（写 trailer + 关 IO + 释放上下文 + 清参数） | 18 |
| 4 | `Impl::openTarget(t)` | **建一条输出管道**：建上下文 → 建流 → 拷参数 → 开 IO → 写头 | 69 |
| 5 | `Impl::closeTarget(t,)` | 拆一条输出管道（可选写 trailer） | 17 |
| 6 | `Impl::writePacket(pkt,tb,v)` | **★ 核心**：一个包写进所有目标，带隔离 | 70 |

### 公开（`StreamOutput`，头文件里给外面用）

| 序号 | 名字 | 一句话职责 |
|---|---|---|
| 1 | `StreamOutput()` / `~StreamOutput()` | 构造 / 析构（析构自动 `close()`） |
| 2 | `open(targets, vParams, aParams, log)` | 一次性建好 N 条管道；**至少连上一路才算成功** |
| 3 | `close()` | 全部拆掉 |
| 4 | `writeVideo(pkt, tb)` | 写一个视频包（内部转 `writePacket(..., true)`） |
| 5 | `writeAudio(pkt, tb)` | 写一个音频包（内部转 `writePacket(..., false)`） |
| 6 | `flush()` | 冲刷每个输出内部的交织缓冲 |
| 7 | `targets()` | 取回每个目标的状态快照（名字/URL/是否连着/发了多少包/多少字节/最后一次错误） |
| 8 | `connectedCount()` | 现在还连着几路 |
| 9 | `reconnectDisconnected()` | 把断掉的重新连一遍，返回重连成功的个数 |
| 10 | `lastError()` | 最后一次全局错误 |

看见没？**对外只有 10 个函数，语义全是"人话"**：开、关、写视频、写音频、冲刷、查状态、重连。

使用者根本不需要知道 `AVFormatContext` 是什么。这就是好的模块设计——**把复杂度关在门后面，门上只留几个按钮**。

---

# 第 1 章 知识点预习（上）：FLV 封装要点回扣

> 这一章我们**不重复第 2 课的内容**，只做"回扣"——把当时讲过、今天马上要用到的四个要点拎出来，并且补上当时没讲的、多目标场景下才暴露出来的细节。

## 1.1 三句话回扣 FLV

第 2 课我们花了整整一节讲 FLV，同学们还记得多少？老师用三句话帮你回忆：

**第一句：FLV 就是"9 字节文件头 + 一串接一串的 Tag"。**

```
[ 'F''L''V' | 版本 | 类型标志 | 头长=9 ]  ← 9 字节
[ PreviousTagSize = 0 ]                  ← 4 字节
[ Tag1 ][ PreviousTagSize1 ][ Tag2 ][ PreviousTagSize2 ] ...
```

**第二句：每个 Tag 自带 11 字节头，里面藏着"类型 + 长度 + 毫秒时间戳 + 流ID"。**

```
1 字节 TagType   : 8=音频  9=视频  18=脚本数据
3 字节 DataSize  : 后面数据多大
3 字节 Timestamp : 时间戳低 24 位（毫秒）
1 字节 TimeExt   : 时间戳高 8 位
3 字节 StreamID  : 恒为 0
```

**第三句：RTMP 传输的媒体数据，格式就是 FLV Tag。**

所以"写本地 .flv"和"推 rtmp://"在 FFmpeg 里是**同一套代码**，区别只在 `avio` 层走文件协议还是走 RTMP 协议。

好，回忆完毕。下面进入今天真正要用到的四个要点。

## 1.2 要点一：时间基必须是 1/1000（毫秒）

这是**最容易出错、又最不容易被发现**的一点。

为什么必须是 1/1000？看上面那个 Tag 头：**时间戳字段只有 3 字节 + 1 字节扩展 = 4 字节，单位规定为毫秒**。FLV 规范写死的，没有任何商量余地。

所以 FFmpeg 里，FLV 输出流的 `time_base` 必须设成：

```cpp
constexpr AVRational kFlvTimeBase = {1, 1000};   // ★ 今天的代码第 64 行
...
videoStream->time_base = kFlvTimeBase;
audioStream->time_base = kFlvTimeBase;
```

> **老师打比方**：`time_base` 是**尺子的刻度**。`{1,1000}` 这把尺子，刻度是"1 毫米"；`{1,30}` 这把尺子，刻度是"一帧"（1/30 秒）。**同一个真实时刻，用不同的尺子量出来数值不同。** 换尺子的动作叫"换算"，千万别忘。

我们今天会碰到**三把不同的尺子**，同学们一定要分清：

| 位置 | 时间基 | 谁的时间戳用它 | 举例：第 30 帧（1 秒处） |
|---|---|---|---|
| 视频编码器 | `{1, 30}`（1/fps，fps=30） | `AVFrame::pts`、`AVPacket::pts`（编码器刚吐出来时） | pts = 30 |
| 音频编码器 | `{1, 48000}`（1/采样率） | 音频 `AVPacket::pts` | 1 秒处 pts = 48000 |
| **FLV 输出流** | **`{1, 1000}`** | **写进容器的 packet pts/dts** | **两个都变成 1000** |

从 30 变成 1000、从 48000 变成 1000，靠的就是 `av_packet_rescale_ts`：

```cpp
av_packet_rescale_ts(copy, timeBase, stream->time_base);
//                          ^^^^^^^^ 源：编码器时间基（调用方传进来）
//                                    ^^^^^^^^^^^^^^^^ 目标：流的 {1,1000}
```

**在多目标场景下，这里有一个非常重要的推论**，同学们记一下：

> 因为**每个目标的时间基都是 `{1,1000}`**（都写死成 FLV 了），所以换算结果对所有目标都一样。
>
> 但代码并没有依赖这个巧合——它**对每个目标独立调用 `av_packet_rescale_ts`**（因为每个目标的 `stream->time_base` 是各自读的）。
>
> 这样做的好处是：**如果将来某个目标改成 MP4（`{1,90000}`）或者 MPEG-TS，代码一行都不用改。**

这就是"**不要假设，要去读**"的工程原则。当年第 2 课的 demo 里我们只有一个输出，随手写死无所谓；今天有 N 个输出，就必须老老实实从每个 `stream` 上读。

### 忘了换算会怎样？

| 症状 | 原因 |
|---|---|
| 画面快进（2 秒内容 0.06 秒播完） | 把 `{1,30}` 的 pts 当成 `{1,1000}` 写了，时间被压缩了 33 倍 |
| 画面慢动作 | 换算方向反了 |
| 播放器报 "missing pts" / 卡在第一帧 | 压根没设 pts |
| 音画不同步，越播越偏 | 音视频没归到同一时间基准 |

我们第 7 章的测试里有两条断言专门盯这个：

```cpp
check(info.maxVideoMs >= 1800 && info.maxVideoMs <= 2100, "视频时间轴落在 2 秒附近（时间基换算对了）");
check(info.maxAudioMs >= 1800 && info.maxAudioMs <= 2200, "音频时间轴落在 2 秒附近（重采样和偏移都没跑偏）");
```

**"把文件读回来，检查最后一包的时间戳是不是落在 2 秒附近"——这就是验证时间基换算对不对的最直接办法。** 同学们做实验时一定要加上这条断言。

## 1.3 要点二：`stream_index` —— 这一包属于哪条流

一个 FLV 里有两条流（视频、音频）。FFmpeg 怎么知道你交给它的这个 packet 该写进哪条流？靠 `AVPacket::stream_index`。

```cpp
copy->stream_index = stream->index;
```

这里的 `stream` 是**当前这个目标自己的** `AVStream*`：

```cpp
AVStream* stream = isVideo ? target.videoStream : target.audioStream;
```

**注意这里有个多目标场景下特别容易踩的坑**：

```
目标 A 的 AVFormatContext：
    streams[0] = 视频  → index = 0
    streams[1] = 音频  → index = 1

目标 B 的 AVFormatContext：
    streams[0] = 视频  → index = 0
    streams[1] = 音频  → index = 1
```

好巧，两个目标里视频都是 0、音频都是 1（因为我们建流的顺序固定：**先视频后音频**）。**但你绝对不能依赖这个巧合！**

原因有两个：

1. **每个 `AVFormatContext` 是独立的**，`stream->index` 是 FFmpeg 在 `avformat_new_stream()` 时自己分配的编号。**如果有第三个目标因为某种原因先建了音频流，编号就反了。**
2. **如果某个目标没有音频流**（比如今天的测试 4：`output.open(targets, encoder.codecParameters(), nullptr, nullptr)`，音频参数传了 `nullptr`），那它根本没有 `audioStream`，你写音频包时 `stream` 是 `nullptr`，**必须跳过**。

所以今天的写法是**教科书式的正确**：

```cpp
AVStream* stream = isVideo ? target.videoStream : target.audioStream;
if (!stream)
    continue;              // ★ 这个目标没有这条流，安静跳过，不是错误
copy->stream_index = stream->index;   // ★ 从目标自己的 stream 上读编号
```

> **老师点评**：这两行看着平平无奇，但它体现了**"多目标分发"最核心的一条纪律**——**每个目标的一切参数，都从这个目标自己的对象上读，绝不从"另一个目标"或者"上游"继承。** 写代码时脑子里要有一根弦：**"如果 N 个目标配置不一样，我这段代码还对吗？"**

### `avformat_new_stream` 的顺序决定了 index

顺便复习一下 `AVStream::index` 是怎么来的（第 2 课略过了）：

```cpp
AVStream* videoStream = avformat_new_stream(ctx, nullptr);   // 返回 streams[0]，index=0
AVStream* audioStream = avformat_new_stream(ctx, nullptr);   // 返回 streams[1]，index=1
```

`avformat_new_stream` 内部就是 `ctx->streams[ctx->nb_streams++] = 新流; 新流->index = nb_streams - 1;`。**先建的就是 0，后建的就是 1。**

**而且这条流的所有者（owner）是 `ctx`**——你**不能**单独 `free` 它，只能 `avformat_free_context(ctx)` 时一起带走。这一点第 2.5 节细讲。

## 1.4 要点三：header 与 trailer —— 容器的首尾

每个容器格式都有"头"和"尾"，FLV 也不例外。

### 头：`avformat_write_header(ctx, nullptr)`

它干了什么？对 FLV 来说：

1. 写 **9 字节 FLV 文件头**（`FLV` + 版本 + 类型标志 + 头长）；
2. 写**第一个 PreviousTagSize = 0**；
3. 写 **`onMetaData` Script Tag（type=18）**——里面装着分辨率、帧率、码率、时长、编码器名……
4. **如果视频是 H.264**：写**第一个 Video Tag（type=9），且 `AVCPacketType=0`**，这个特殊 Tag 里装的是 **AVCDecoderConfigurationRecord**——也就是我们说的 **SPS / PPS**；
5. **如果音频是 AAC**：写**第一个 Audio Tag（type=8），`AACPacketType=0`**，里面装 **AudioSpecificConfig**（采样率、声道数、profile）。

> **关键认知**：第 4 点就是"**全局头（global header）**"在 FLV 里的落地形式。为什么必须把 SPS/PPS 放在最前面？
>
> 因为**观众随时可能中途进入直播间**。他一进来就要能从**下一个关键帧**开始解码，而解码 H.264 必须先拿到 SPS/PPS。如果这些信息只在第一个 Tag 里有，中途进来的人就永远解不开。
>
> 所以直播系统通常会**周期性重复发送 SPS/PPS**（或者在每个关键帧前带一份）。FFmpeg 的 FLV muxer 至少保证开头发一次。

**这些 SPS/PPS 是从哪儿来的？** 从 `AVCodecParameters::extradata` 里来，而它是我们从编码器拷过来的：

```cpp
avcodec_parameters_copy(videoStream->codecpar, videoParams);
```

这就是为什么第 8 课的编码器里必须有这一行：

```cpp
impl_->ctx->flags |= AV_CODEC_FLAG_GLOBAL_HEADER;   // ★ 让编码器把 SPS/PPS 导出到 extradata
```

**没有这一行，extradata 是空的，写出来的 FLV 没有序列头，播放器解不开。** 这是初学者最容易踩的坑之一（第 2 课 4.2 节讲过，这里回扣一下）。

### 尾：`av_write_trailer(ctx)`

它干什么？

1. 把内部**交织缓冲区**里剩下的包全部写出；
2. 写容器的"结束标记"（FLV 没有专门的结束 Tag，但可以写 `AVC end of sequence`）；
3. **如果输出可 seek（比如本地文件）**，回过头去更新头部的元数据（比如 `duration` 总时长）；
4. 刷盘、flush 缓冲区。

**这一步不能省！** 省掉的后果：

- 最后几帧丢失（还在交织缓冲里没出去）；
- 文件的 `duration` 字段是错的或者为 0，播放器拖动条乱跳；
- 网络流没发出结束信号，服务器那边可能一直等着。

我们今天的代码里，`av_write_trailer` 出现在两个地方，都很讲究：

```cpp
// 地方一：正常关闭（release 里），写 trailer
release()   → closeTarget(target, /*writeTrailer=*/true)

// 地方二：写失败后被踢掉，不写 trailer
writePacket() 失败时 → 直接 avio_closep + avformat_free_context，不写 trailer
```

**为什么失败时不写 trailer？** 同学们想一想。

> **答案**：因为网络已经断了，`av_write_trailer` 大概率**会阻塞很久然后也失败**（白白浪费几秒钟），甚至可能触发更糟的行为。而且一个"被截断的 FLV"本来就是**合法**的——第 2 课讲过，FLV 是纯流式格式，随便截断，前面写出去的部分照样能播。**这时候"快速放弃"比"优雅告别"更正确。**

## 1.5 要点四：`codec_tag = 0` 是个什么操作

代码里有这么一行，很多同学第一次见会懵：

```cpp
videoStream->codecpar->codec_tag = 0;
```

**`codec_tag`（也写作 fourcc）** 是容器里用来标识编码格式的**四字符码**。比如：

| 容器 | codec_tag | 含义 |
|---|---|---|
| AVI | `'H264'`、`'XVID'`、`'DIVX'` | 见得最多 |
| MOV/MP4 | `'avc1'`、`'hvc1'` | |
| **FLV** | **内部用 CodecID 数字（7 = AVC / H.264）**，不用 fourcc | |

**把它设成 0 的意思是："我不知道该用哪个 fourcc，muxer 你自己看着办。"**

FFmpeg 的 muxer 在写 header 时，如果发现 `codec_tag == 0`，就会调用 `av_codec_get_tag()` 根据自己的格式表挑一个合适的；如果挑不到，还会 fallback 到 `codec_id` 对应的默认值。

**为什么要主动清零？** 因为 `avcodec_parameters_copy` 会把**源参数包里的 `codec_tag` 一起拷过来**。而源参数包是从 `AVCodecContext` 导出的，`AVCodecContext::codec_tag` 可能带有一个**不属于 FLV 的**值（比如 x264 那边残留的默认值）。直接抄过去，某些严格的播放器或者服务器会拒收。

> **一句话记住**：**`codecpar` 拷完之后、写 header 之前，顺手把 `codec_tag` 清零，让 muxer 自己决定。** 这是 FFmpeg 输出侧的"标准动作"，很多官方示例都这么干。

## 1.6 小结：FLV 封装的四个"必须"

同学们，第 2 课的 FLV 知识，落到 `StreamOutput` 里就是这四条硬约束。我们对一下表：

| # | 约束 | 代码里的体现 | 不加会怎样 |
|---|---|---|---|
| 1 | 流的时间基必须是 `{1,1000}` | `videoStream->time_base = kFlvTimeBase;` | 快进/慢动作/播放器卡死 |
| 2 | 每个包必须打上目标自己的 `stream_index` | `copy->stream_index = stream->index;` | 数据串行到音频流里，或者直接崩 |
| 3 | 写第一个包之前必须写 header（含全局头） | `avformat_write_header(ctx, nullptr);` | 文件没有 SPS/PPS，解不开 |
| 4 | 结束前必须写 trailer（正常关闭时） | `av_write_trailer(target.ctx);` | 丢尾帧、duration 为 0 |
| 附 | `codec_tag` 清零 | `->codecpar->codec_tag = 0;` | 兼容性问题，某些播放器打不开 |

**而且这四条，对每个目标都要独立做一遍。** 这就是多目标分发的第一个层次的含义——**N 个目标 = N 套完整的封装状态机，各管各的。**

---

# 第 2 章 知识点预习（中）：`avformat` 输出上下文管理

这一章我们讲"管道"本身的生命周期管理。这是本课最容易出内存泄漏的地方，同学们打起精神。

## 2.1 一个 `AVFormatContext` = 一条完整的输出管道

第 2 课讲过 FFmpeg 四大角色，这里只回扣第四位：

> **`AVFormatContext`（封装上下文）** = "这一次输出/输入的**总控**"。
> 它记录：容器格式是什么（`oformat`）、里面有几条流（`nb_streams` / `streams[]`）、底层 IO 在哪（`pb`）、全局元数据、以及一些内部状态（交织缓冲区、当前写到哪了）。

**在多目标场景下，这句定义要升级成**：

> **一个 `AVFormatContext` = 一条完整的、独立的输出管道。**
> **N 个目标 = N 个 `AVFormatContext`，它们之间没有任何共享状态。**

画个图：

```
                        ┌─────────────────────────────────────┐
                        │  StreamOutput::Impl                 │
                        │  ┌───────────────────────────────┐  │
   AVPacket ─── 分发 ───┼─→│ Target[0] "斗鱼"                │  │
                        │  │   AVFormatContext* ctx ──→ RTMP │  │
                        │  │     streams[0]=video            │  │
                        │  │     streams[1]=audio            │  │
                        │  │   AVIOContext* pb (TCP 1935)    │  │
                        │  │   connected=true, sent=1234     │  │
                        │  └───────────────────────────────┘  │
                        │  ┌───────────────────────────────┐  │
                        ├─→│ Target[1] "B站"                 │  │
                        │  │   AVFormatContext* ctx ──→ RTMP │  │
                        │  │   ...（完全独立的一套）          │  │
                        │  └───────────────────────────────┘  │
                        │  ┌───────────────────────────────┐  │
                        └─→│ Target[2] "本地录像"            │  │
                           │   AVFormatContext* ctx ──→ 文件 │  │
                           └───────────────────────────────┘  │
                        └─────────────────────────────────────┘
```

**"完全独立"这四个字非常重要。** 它意味着：

- 目标 0 的 TCP 连接断了，**不影响**目标 1、2 的 `ctx`；
- 目标 0 的交织缓冲区里积压了多少包，**跟目标 1 无关**；
- 你可以关掉目标 2 而其它两路继续跑；
- 每路的统计（发了多少包、多少字节）各自记账。

## 2.2 建管道的五步 与 拆管道的五步

FFmpeg 的输出侧 API 是严格配对的。同学们把这张表背下来：

### 建（对应 `Impl::openTarget`）

| 步 | 函数 | 干了什么 | 失败后要回滚什么 |
|---|---|---|---|
| 1 | `avformat_alloc_output_context2(&ctx, nullptr, "flv", url)` | 分配上下文、**按名字找 muxer**、按 url 后缀猜格式（作为兜底） | 不需要，ctx 还是 nullptr |
| 2 | `avformat_new_stream(ctx, nullptr)` | 建流（视频；有音频再建一条） | `avformat_free_context(ctx)` |
| 3 | `avcodec_parameters_copy(stream->codecpar, params)` | 把编码参数（含 extradata/SPS/PPS）拷进流 | 同 2 |
| 4 | `avio_open2(&ctx->pb, url, AVIO_FLAG_WRITE, nullptr, &options)` | 打开底层 IO（**按 URL 前缀分派协议**） | 同 2 |
| 5 | `avformat_write_header(ctx, nullptr)` | 写容器头 + 元数据 + 全局头 | `avio_closep(&ctx->pb)` + `avformat_free_context(ctx)` |

### 拆（对应 `Impl::closeTarget`）

| 步 | 函数 | 干了什么 |
|---|---|---|
| 1 | `av_write_trailer(ctx)` | 写尾（**只在正常关闭时**） |
| 2 | `avio_closep(&ctx->pb)` | 关 IO，**并把 `pb` 指针置空**（注意是带 `p` 的版本） |
| 3 | `avformat_free_context(ctx)` | 释放上下文，**顺带释放它下面所有 `AVStream`** |
| 4 | `target.ctx = nullptr` | **手动置空，防止悬垂指针（dangling pointer）** |

**注意顺序不能乱**：先 trailer（要把数据写出去），再关 IO（关了就写不了了），最后释放上下文。

> **老师打比方**：这就像关店——**先把门口还排队的客人服务完（trailer），再拉卷帘门（avio_close），最后退租交钥匙（free_context）**。你要是先把门拉了，客人的数据就丢在屋里了。

### 那个"回滚"的写法值得学

同学们看 `openTarget` 里的错误处理，它是一个**阶梯式回滚**：

```cpp
// 第 2、3 步失败 → 只回滚第 1 步
if (!videoStream || avcodec_parameters_copy(...) < 0) {
    avformat_free_context(ctx);       // ← 只回滚到这一步
    target.lastError = "创建视频流失败";
    return false;
}

// 第 4 步失败 → 还是只回滚第 1 步（pb 还没开成功，不用关）
if (ret < 0) {
    avformat_free_context(ctx);
    target.lastError = "连接失败: " + avErrorText(ret);
    return false;
}

// 第 5 步失败 → 回滚第 4 步 + 第 1 步（pb 已经开了，得关）
if (ret < 0) {
    if (ctx->pb)
        avio_closep(&ctx->pb);        // ← 多了这一步
    avformat_free_context(ctx);
    target.lastError = "写文件头失败: " + avErrorText(ret);
    return false;
}
```

**这就是"RAII 之前"的 C 语言标准写法：失败时，把之前成功过的步骤按相反顺序全部撤销。**

> **老师点评**：第 2 课的 demo 里我们用 `exit(1)` 一走了之，老师当时就批评过（第 2 课 8.4 节缺陷 1）。今天这段代码**就是那个缺陷的正解**——每一条失败路径都把资源清干净了。同学们写 C 风格代码时，请照这个模板来：**每成功一步，就想好"如果下一步失败，这一步怎么撤"。**

## 2.3 `AVFMT_NOFILE`：不是所有输出都需要 `avio_open`

代码里有这么一个判断，很多同学会忽略：

```cpp
if (!(ctx->oformat->flags & AVFMT_NOFILE)) {
    // 才需要 avio_open2
}
```

**`AVFMT_NOFILE`** 是 muxer 的一个标志位，意思是"**这个格式不需要 FFmpeg 帮我开文件/开网络，我自己管理 IO**"。

哪些格式是这样？

- 有些 muxer 要求你自己提供一个 `AVIOContext`（比如你想把数据直接写进自己的内存缓冲区）；
- 有些 muxer 压根不产出字节流（极少见）。

**FLV 不属于这类**——它需要 `avio_open`。但写上这个判断是**正确且专业**的，因为：

1. 它让你的代码对"任意格式"都成立（万一以后换成 `AVFMT_NOFILE` 的 muxer）；
2. 更关键的是——**如果你对一个 `AVFMT_NOFILE` 的 muxer 调 `avio_open`，会打开一个多余的文件描述符，造成资源泄漏。**

> **一句话记住**：**开 IO 之前先问一句"这个格式需要我开吗"。** 这是 FFmpeg 官方示例 `muxing.c` 里的标准写法。

## 2.4 `avio_open2` 与 `AVDictionary`：`rw_timeout` 是救命稻草

### `avio_open` 与 `avio_open2` 的区别

```cpp
int avio_open(AVIOContext** s, const char* url, int flags);

int avio_open2(AVIOContext** s, const char* url, int flags,
               const AVIOInterruptCB* int_cb,   // 中断回调
               AVDictionary** options);          // ★ 私有选项
```

`avio_open2` 多出来的第二个参数（options）让我们能传 **协议层的私有选项**。今天我们只传了一个，但极其关键：

```cpp
constexpr const char* kIoTimeout = "5000000";   // 第 66 行

AVDictionary* options = nullptr;
av_dict_set(&options, "rw_timeout", kIoTimeout, 0);
ret = avio_open2(&ctx->pb, target.url.c_str(), AVIO_FLAG_WRITE, nullptr, &options);
av_dict_free(&options);
```

### `rw_timeout` 是什么？单位是微秒！

**`rw_timeout` = "单次读写操作的最长等待时间"，单位是微秒（μs）。**

```
5000000 μs = 5,000 ms = 5 秒   ← 我们设的值
```

**为什么必须设？** 同学们，这是直播系统里血泪教训换来的经验：

> 如果不设超时，当网络出现"半死不活"的状态（比如 TCP 连接建立了，但对端不回 ACK），`avio_open2` 或 `av_interleaved_write_frame` 会**无限期阻塞**。
>
> 表现就是：**你的推流程序卡死了，界面无响应，Ctrl+C 都不理你，只能杀进程。**

设了 5 秒超时之后， worst case 是"卡 5 秒然后返回一个 `-ETIMEDOUT` 错误"，我们的代码就能捕获它、标记这个目标断开、然后继续服务其它目标。**这就是"写失败隔离"能够成立的前提。**

### `AVDictionary` 的用法与坑

`AVDictionary` 是 FFmpeg 的**字符串键值对容器**，用来传私有选项。用法是经典的三步：

```cpp
AVDictionary* options = nullptr;              // ① 必须是 nullptr 开头！
av_dict_set(&options, "rw_timeout", "5000000", 0);   // ② set（内部会按需分配）
avio_open2(..., &options);                    // ③ 用
av_dict_free(&options);                       // ④ 用完必须释放！
```

**两个坑**：

1. **指针必须初始化为 `nullptr`**。因为 `av_dict_set` 发现 `*options == nullptr` 时会帮你分配一个新的字典；如果指针是个野值，直接崩。
2. **必须 `av_dict_free`**。FFmpeg 的协议层会把它**不认识的**选项留在字典里（认识的会被"消费"掉）。所以用完一定要释放剩下的。

> **老师补充**：`avio_open2` 的第四个参数 `int_cb`（中断回调）是另一种超时手段——你可以传一个函数指针，FFmpeg 在阻塞 IO 时会周期性调用它问"要不要中断"。它比 `rw_timeout` 更灵活（可以做成"用户点取消按钮就中断"），但也更麻烦。`rw_timeout` 是"够用就好"的选择。

## 2.5 生命周期归谁：谁 new 谁 free，谁只清谁不管

这是本课**最容易写错、也最容易泄漏**的地方。同学们，我们逐个对象理清楚：

| 对象 | 谁创建的 | 归谁 | 怎么释放 | 备注 |
|---|---|---|---|---|
| `AVFormatContext` | `avformat_alloc_output_context2` | **你** | `avformat_free_context(ctx)` | **释放它会连带释放所有 `AVStream`** |
| `AVStream` | `avformat_new_stream` | **`ctx`** | ❌ **不能单独 free！** | 随 `avformat_free_context` 一起走 |
| `ctx->pb`（`AVIOContext`） | `avio_open2` | **你** | `avio_closep(&ctx->pb)` | 用带 `p` 的版本，会自动置空 |
| `AVCodecParameters`（我们自己的两份） | `avcodec_parameters_alloc` | **你** | `avcodec_parameters_free(&p)` | 跟 `ctx` 无关，独立存在 |
| `stream->codecpar` | `avformat_new_stream` 内部 | **`ctx`** | ❌ 不能单独 free | 随 ctx 走 |
| `AVPacket* copy` | `av_packet_clone` | **你**（暂时） | `av_packet_free(&copy)` | **写完就 free**（见 3.5 节） |

**几个必须记住的点**：

### 点一：`AVStream` 绝不能单独释放

初学者常犯的错：

```cpp
// ✗ 大错特错！
avformat_free_context(ctx);
av_free(videoStream);      // 双重释放 → 崩溃
```

`videoStream` 指向的内存是 `ctx` 内部数组的一个元素，`avformat_free_context` 已经释放过了。**正确的做法是：释放 ctx 之后，把你保存的裸指针手动置空**，防止变成悬垂指针：

```cpp
void StreamOutput::Impl::closeTarget(Target& target, bool writeTrailer)
{
    ...
    avformat_free_context(target.ctx);
    target.ctx = nullptr;          // ① 主指针置空

    target.videoStream = nullptr;  // ② 两个"借来的"指针也置空 ★
    target.audioStream = nullptr;
    target.connected = false;      // ③ 状态复位
}
```

**这就是"防御性编程"**：野指针的可怕之处在于它**不一定立刻崩**——它可能指向一块恰好还合法的内存，让你在某些机器上跑得好好的，到客户那儿随机崩。**置空之后，任何误用都会立刻在 `if (!target.ctx)` 这种检查处被拦下，或者干脆段错误当场暴露。**

### 点二：我们自己的 `AVCodecParameters` 是"模板"，跟任何 ctx 无关

```cpp
struct StreamOutput::Impl {
    AVCodecParameters* videoParams = nullptr;   // 视频参数"模板"
    AVCodecParameters* audioParams = nullptr;   // 音频参数"模板"
};
```

为什么需要这两份拷贝？

**因为我们要给 N 个目标各建一套流，每套流都要一份 `codecpar`。** 而上游（编码器）只给我们**一份** `const AVCodecParameters*`。

两个选择：

- **选择 A**：每次建流时直接找编码器要（那 `StreamOutput` 就得持有编码器的指针，耦合）；
- **选择 B（我们选的）**：`open()` 时**深拷贝一份存起来**，之后建 N 个流就从这份模板拷 N 次。

**选择 B 的好处是解耦**：`open()` 调用完，`StreamOutput` 就**跟编码器彻底没关系了**。编码器可以马上被销毁，或者重新配置，`StreamOutput` 都不在乎。

> **这个设计还有一个隐藏好处，第 10 课会兑现**：`reconnectDisconnected()` 重连时，需要重新建流、重新拷 `codecpar`。如果当时编码器已经关了，选择 A 就抓瞎了；选择 B 因为手里有模板，随时能重建。
>
> **这就是"在正确的时机做一份快照"的价值。**

注意 `avcodec_parameters_copy` 是**深拷贝**——它会连 `extradata`（SPS/PPS）一起拷，分配新内存。所以两份模板必须自己 `avcodec_parameters_free`。

### 点三：释放顺序（`release()`）

```cpp
void StreamOutput::Impl::release()
{
    for (Target& target : targets)
        closeTarget(target, true);     // ① 先拆所有管道（写 trailer）
    targets.clear();                   // ② 再清空目标数组

    if (videoParams) {                 // ③ 最后释放参数模板
        avcodec_parameters_free(&videoParams);
        videoParams = nullptr;
    }
    if (audioParams) {
        avcodec_parameters_free(&audioParams);
        audioParams = nullptr;
    }

    audioOffsetReady = false;          // ④ 复位音频偏移状态
    audioOffset = 0;
}
```

**顺序讲究**：先关管道（管道里可能还在用参数模板？其实不会，但逻辑上"先停后清"永远是对的），再清数组，最后清模板。

**而且它是幂等的**——`release()` 里所有释放都带 `if (xxx)` 检查 + 释放后置空，所以**调用两次也不会崩**。这一点非常重要，因为：

```cpp
bool StreamOutput::open(...) { close(); ... }   // open 开头调 close
StreamOutput::~StreamOutput() { close(); }      // 析构也调 close
```

`open()` 失败时还可能再调一次 `close()`（见 5.6 节）。**一个会被调用多次的清理函数，必须幂等。**

## 2.6 RTMP url 与本地文件 url 的统一抽象

这一节是本章的"彩蛋"，也是本课最省事的一块——**因为我们什么都不用做**。

### FFmpeg 的 `avio` 层：URL 前缀分派协议

FFmpeg 有一个抽象层叫 **`avio`（AV I/O）**，还有一个更底层的 **`URLProtocol`** 机制。它的工作方式跟浏览器的地址栏一模一样：

```
avio_open2(&pb, url, AVIO_FLAG_WRITE, ...)
        │
        ├─ 取出 url 的前缀（"://" 之前的部分）
        │
        ├─ "rtmp"  → 找 rtmp 协议实现   → 建立 TCP 连接到 host:1935 → RTMP 握手
        ├─ "file"  → 找 file 协议实现   → CreateFileW / fopen
        ├─ "http"  → 找 http 协议实现   → WinHTTP / libcurl
        ├─ "pipe"  → 找 pipe 协议实现   → 匿名管道
        ├─ "tcp"   → 找 tcp 协议实现    → 裸 TCP
        └─ （没有 "://"）→ 默认按 file 协议处理
```

**所以这几行 URL 走的是完全不同的底层实现，但上面这段代码一行都不用改**：

```cpp
{"斗鱼",  "rtmp://send.douyu.com/live/xxxxx"}
{"B站",   "rtmp://live-push.bilivideo.com/live-ybd?streamname=xxx&key=yyy"}
{"录像",  "D:/Record/2026-10-20.flv"}
{"测试A", "output_a.flv"}                          ← 没有前缀 → 文件
{"死路",  "rtmp://127.0.0.1:1/live/nothing"}       ← 端口 1，没人监听
```

> **老师补充**：那 `avformat_alloc_output_context2(&ctx, nullptr, "flv", url)` 里**已经显式写了 "flv"**，为什么还要传 url？
>
> 因为 FFmpeg 的规则是：**第三个参数（format name）优先级最高；如果你传了 `nullptr`，它就根据 url 的后缀猜**（`.flv` → flv，`.mp4` → mp4，`.ts` → mpegts）。
>
> 我们**显式写死 "flv"** 是有意为之——因为 RTMP 的 URL 后缀通常是 `/live/xxxx`，**没有任何格式信息**，不显式指定 FFmpeg 会猜不出来。第 2 课讲过这一点，这里回扣。

### 这个抽象带来的三个巨大好处

#### 好处一：测试不需要服务器 ★★★

这是今天 `test_output.cpp` 能自动跑通的根本原因。我们来看测试用例 2：

```cpp
const std::vector<StreamTarget> targets = {{"A", "output_a.flv"},
                                           {"B", "output_b.flv"}};
```

**两个"目标"都是本地文件，但走的是跟 RTMP 完全一样的 `StreamOutput` 代码路径**（`avformat_alloc_output_context2` → `avio_open2` → `write_header` → `write_frame` → `trailer`）。

唯一的区别是 `avio` 底层调的是 `fopen` 还是 `connect`。**这就把"能不能推流"这个难以自动化验证的问题，转化成了"写出来的 flv 对不对"这个容易验证的问题。**

> **老师点评**：同学们，这种"**用等价的、可控的替身来测试难以触达的路径**"的技巧，在软件工程里叫 **Test Double（测试替身）**，具体到这里是 **Fake（伪实现）**——本地文件就是 RTMP 的 Fake。
>
> 这个方法非常实用。以后你们遇到"这段代码依赖 XX 服务器/硬件/第三方服务，没法在 CI 里跑"的时候，第一反应就应该是：**能不能给它做个 Fake？**

#### 好处二：断线隔离可以自动化验证

看测试用例 3：

```cpp
const std::vector<StreamTarget> targets = {
    {"live", "output_isolated.flv"},
    {"dead", "rtmp://127.0.0.1:1/live/nothing"}};
```

**端口 1 是系统保留端口，正常情况下绝不可能有人在监听。** 所以第二路 `avio_open2` 一定会失败（连接被拒绝）。

于是我们就能**自动验证**：

```cpp
check(run.ok,                "整体仍算成功（有一路可用就算成功）");
check(run.targets[0].connected,  "文件那一路正常连接");
check(!run.targets[1].connected, "rtmp 那一路被判定为断开");
check(!run.targets[1].lastError.empty(), "断开的理由被记录下来了");
check(run.targets[0].packetsSent > 0,    "好的那一路照常发出了包");
check(run.targets[1].packetsSent == 0,   "坏的那一路一个包都没发出去");
```

**不需要真的断网、不需要真的有个会挂的服务器**——一个没人监听的端口就够。

#### 好处三：业务代码零改动

最后这一点最实在：**从"只写本地文件"升级到"真的推流"，改动量 = 改一个字符串。**

```cpp
// 开发期
std::vector<StreamTarget> targets = {{"local", "debug.flv"}};

// 上线
std::vector<StreamTarget> targets = {
    {"斗鱼", "rtmp://send.douyu.com/live/" + key1},
    {"B站",  "rtmp://live-push.bilivideo.com/live-ybd/" + key2},
    {"录像", "D:/Record/" + today + ".flv"}};
```

**这就是"抽象"的价值**——它把"变化点"压缩到了最小。

---

# 第 3 章 知识点预习（下）：一份数据写多处 与 写失败隔离

> **本章是本课的灵魂。** 前面两章是"回扣"，这一章是"新东西"。请同学们打起十二分精神。

## 3.1 朴素想法错在哪：一个 `AVPacket` 能不能直接写两次？

老师先问一个问题。假设我们有一包数据 `pkt`，要写进两个目标，**最朴素的写法是不是这样**：

```cpp
// ✗ 天真写法（错误示范，别抄）
for (Target& t : targets) {
    pkt.stream_index = t.videoStream->index;
    av_packet_rescale_ts(&pkt, encoderTb, t.videoStream->time_base);
    av_interleaved_write_frame(t.ctx, &pkt);
}
```

这段代码**会出三个问题**，我们一个一个看。

### 问题一：`av_interleaved_write_frame` 会"吃掉"你的 packet

这是最致命的。**这个函数成功返回之后，你传进去的那个 `AVPacket` 已经被 `unref` 了**——它的数据引用被移交给 FFmpeg 内部的交织缓冲区（或者已经被写出去了）。

所以第二次循环时，`pkt.data` 已经是一个悬垂指针，`pkt.size` 是 0。**你往第二个目标写的是一包空气。**

> FFmpeg 的文档原文（对 `av_interleaved_write_frame`）：
> *"...the packet is **blanked on success**... whether or not the output format is interleaved, this function will **take ownership of the packet**."*
>
> 翻译：**"这个包在成功时被清空……本函数会取得这个包的所有权。"**

**"取得所有权"（take ownership）** 这个词同学们要敏感——它意味着"**这个东西现在归我了，你别再碰了，也不用你释放了**"。

### 问题二：反复 `rescale_ts` 会把时间戳搞乱

```cpp
av_packet_rescale_ts(&pkt, encoderTb, tb);   // 第一次：30 → 1000
av_packet_rescale_ts(&pkt, encoderTb, tb);   // 第二次：1000 → 33333 !!!
```

`av_packet_rescale_ts` 是**原地修改**的。它假设输入的时间戳**当前处于 `src_tb` 时间基下**。第二次调用时，`pkt.pts` 已经是 1000（毫秒）了，你还告诉它"这玩意儿是 `{1,30}` 尺子量出来的"，它就算出 `1000 × (1000/30) = 33333`。

**时间戳第二次就飞了。**

### 问题三：一个目标失败，循环就断了

如果第一个目标写失败抛异常或者直接 `return false`，**第二个目标就永远收不到数据了**。这跟我们 0.3 节说的"互不干扰"完全背道而驰。

### 正解：给每个目标一份独立的副本

```cpp
for (Target& target : targets) {
    AVPacket* copy = av_packet_clone(&packet);   // ★ 每个目标一份独立副本
    if (!copy) { /* 只跳过这一个目标 */ continue; }

    // 之后所有的修改都作用在 copy 上，原包不受影响
    copy->pts -= ptsOffset;
    copy->stream_index = stream->index;
    av_packet_rescale_ts(copy, timeBase, stream->time_base);

    av_interleaved_write_frame(target.ctx, copy);
    av_packet_free(&copy);
}
```

**这样就三个问题一起解决了**：
- 每个目标有自己的 `copy`，被吃掉的是 `copy`，原包毫发无损；
- `rescale_ts` 作用在 `copy` 上，每次都是从 `timeBase` 换算到目标时间基，正确；
- 每个目标的写操作独立，失败只影响自己。

## 3.2 引用计数：`AVPacket` 内部的 `AVBufferRef`

好，现在老师要讲透一个关键问题：**`av_packet_clone` 到底拷没拷数据？**

答案是：**没有拷数据（大部分情况下）。它只增加了一个引用计数。**

### `AVPacket` 的内部结构

我们看一下（简化版，去掉了一堆我们不关心的字段）：

```cpp
typedef struct AVPacket {
    AVBufferRef* buf;        // ★ 引用计数块（真正的数据在这里）
    uint8_t*     data;       // 指向 buf->data（可能带偏移）
    int          size;       // 数据字节数

    int64_t pts;             // 显示时间戳
    int64_t dts;             // 解码时间戳
    int64_t duration;        // 时长
    int     stream_index;    // 属于哪条流
    int     flags;           // AV_PKT_FLAG_KEY 等
    ...
} AVPacket;
```

注意：**`data` 只是一个指针，`buf` 才是数据的所有者。**

`AVBufferRef` 是 FFmpeg 的**引用计数智能指针**，结构大概是：

```cpp
typedef struct AVBufferRef {
    AVBuffer* buffer;   // → 指向真正的引用计数块
    uint8_t*  data;     // 数据地址
    size_t    size;
};

struct AVBuffer {
    uint8_t* data;              // 真正的内存
    size_t   size;
    std::atomic<int> refcount;  // ★ 引用计数（线程安全）
    void (*free)(...);          // 释放回调
};
```

**内存布局是这样的**：

```
   pkt1 (AVPacket)          pkt2 (AVPacket)          pkt3 (AVPacket)
   ┌──────────────┐         ┌──────────────┐         ┌──────────────┐
   │ buf ─────────┼──┐      │ buf ─────────┼──┐      │ buf ─────────┼──┐
   │ data ────────┼──┼──┐   │ data ────────┼──┼──┐   │ data ────────┼──┼──┐
   │ size = 5120  │  │  │   │ size = 5120  │  │  │   │ size = 5120  │  │  │
   │ pts  = 30    │  │  │   │ pts  = 1000  │  │  │   │ pts  = 90000 │  │  │
   │ stream_index │  │  │   │ stream_index │  │  │   │ stream_index │  │  │
   └──────────────┘  │  │   └──────────────┘  │  │   └──────────────┘  │  │
                     │  │                     │  │                     │  │
                     ▼  ▼                     ▼  ▼                     ▼  ▼
        ┌──────────────────────────────────────────────────────┐
        │  AVBuffer                                            │
        │  refcount = 3   ← ★ 三个引用                          │
        │  ┌────────────────────────────────────────────────┐  │
        │  │  真正的 H.264 数据（5 KB），只有一份！            │  │
        │  └────────────────────────────────────────────────┘  │
        └──────────────────────────────────────────────────────┘
```

**看明白了吗？三个 `AVPacket` 结构体是三份独立的（各自的 pts、stream_index 可以不同），但它们共享同一块 5 KB 的数据内存，靠 `refcount = 3` 记账。**

> **老师打比方**：这就像**三个同事共享一份云端文档**。每个人都可以在自己的电脑上改"备注"、"标签"（`AVPacket` 自己的字段），但**文档正文只有一份**（`AVBuffer`）。谁都不用了（refcount 归 0），文档才被真正删掉。

### 这对多目标分发意味着什么？

**意味着"一个包写 N 个目标"的内存代价 ≈ 一个包。**

| 方案 | 内存开销 | CPU 开销 |
|---|---|---|
| 真拷贝 N 份（`memcpy`） | `N × 5 KB` | N 次 memcpy |
| **引用计数（我们用的）** | `5 KB + N × ~100 字节`（结构体） | N 次整数自增 |

一路 1080p 直播按 5 Mbps、25 fps 算，一包平均 25 KB。推 5 个目标：

- 真拷贝：每秒 `25 × 5 × 25 KB = 3.1 MB` 的 memcpy；
- 引用计数：每秒**零拷贝**，只有 125 次引用计数操作。

**差别不大？在 1080p60 高码率场景就是每秒 15 MB 的拷贝，而且在高并发的转码服务器上这个数字要乘以几十路。** 更别提 memcpy 会污染 CPU cache。

## 3.3 `av_packet_ref` / `av_packet_clone` / `av_packet_unref` / `av_packet_free` 四兄弟

FFmpeg 的 packet API 一共有四个，同学们必须分得清清楚楚：

| 函数 | 干了什么 | 类比 C++ 概念 | 之后 data 还能用吗 |
|---|---|---|---|
| `av_packet_alloc()` | 分配一个**空**的 `AVPacket` 结构体 | `new AVPacket()` | 空的，没有数据 |
| `av_packet_ref(dst, src)` | **让 dst 共享 src 的数据**（refcount++），并拷贝所有字段 | 拷贝构造（共享数据） | ✅ 能 |
| `av_packet_clone(src)` | `= av_packet_alloc() + av_packet_ref()` 的简写 | `make_shared` | ✅ 能 |
| `av_packet_unref(pkt)` | **归还数据引用**（refcount--，归 0 就释放数据），并把所有字段重置为默认值 | `reset()` | ❌ 空了，但**对象还在，可以复用** |
| `av_packet_free(&pkt)` | `= av_packet_unref() + 释放结构体`，**并把指针置 nullptr** | `delete` + 置空 | ❌ 对象都没了 |

**关系图**：

```
av_packet_alloc()  ──→  [空 packet]
                             │
                             │ av_packet_ref(dst, src)
                             ▼
                        [有数据的 packet]  ←── av_packet_clone(src) 一步到位
                             │
        ┌────────────────────┴────────────────────┐
        │                                          │
  av_packet_unref()                          av_packet_free()
   数据引用 -1                                先 unref，再释放结构体
   字段重置                                   指针置 nullptr
   对象可复用                                 对象没了
```

### 最容易搞混的一对：`unref` vs `free`

```cpp
AVPacket* pkt = av_packet_alloc();

// 循环复用同一个对象（常见于读文件、编码循环）
while (...) {
    av_read_frame(fmt, pkt);
    process(pkt);
    av_packet_unref(pkt);      // ★ 归还数据，对象留着下一轮用
}
av_packet_free(&pkt);          // 最后才真正释放对象

// 一次性使用（我们今天的场景）
AVPacket* copy = av_packet_clone(&packet);
av_interleaved_write_frame(ctx, copy);
av_packet_free(&copy);         // ★ 用完直接 free（内部会先 unref）
```

**我们今天用的是第二种**：每个 `copy` 用完就不要了，直接 `av_packet_free`。

### 那 `av_packet_free` 之后为什么不用再 `unref`？

因为 `av_packet_free` **内部会先调 `av_packet_unref`**。源码逻辑（简化）：

```c
void av_packet_free(AVPacket** pkt) {
    if (!pkt || !*pkt) return;
    av_packet_unref(*pkt);     // ① 先归还数据引用
    av_freep(pkt);             // ② 再释放结构体并置空
}
```

所以这段代码是**完全正确**的：

```cpp
const int ret = av_interleaved_write_frame(target.ctx, copy);
av_packet_free(&copy);         // 不管写成功还是失败，都 free
if (ret < 0) { ... }
```

注意：**即使 `av_interleaved_write_frame` 已经"吃掉"（unref 了）这个包，我们再 `av_packet_free` 一次也是安全的**——因为 `unref` 是幂等的（对一个已经空了的 packet 再 unref，什么也不做）。

> **老师点评**：这个"**无论成败都释放**"的写法，比"成功就不释放（因为被吃掉了）、失败才释放"要**健壮得多**。
>
> 后者依赖一个脆弱的假设："我确切知道 FFmpeg 在这个分支下会不会 unref"。一旦 FFmpeg 某个版本改了行为，你就泄漏或者双重释放。
>
> **工程原则**：**能不依赖库的隐式行为，就别依赖。** 我们直接无条件 free，反正它对两种情况都正确。

## 3.4 为什么用 `clone` 而不是 "ref 原包 + 改字段"

有同学可能会问：既然 `av_packet_ref` 就够了，为什么不这样写：

```cpp
// 思路：先给所有目标 ref 好，再逐个改字段？
```

不行。**因为 `ref` 是单向的：`av_packet_ref(dst, src)` 之后，`dst` 和 `src` 仍然共享同一块数据，但字段是拷贝的。**

我们要的是"**每个目标一个独立的 `AVPacket` 对象，可以随便改自己的 pts/stream_index，而不能动原包**"。

- 如果直接改 `src`（原包）的字段再 `ref` → 改了原包，下一个目标读到的就是被改过的，**而且原包是调用方的（上游编码器给的），我们没权利改**；
- 如果先 `ref` 到 `dst` 再改 `dst` → 那 `dst` 也得是 `av_packet_alloc()` 出来的新对象。

**而 `av_packet_clone` 正好就是这两步的合体**：

```c
AVPacket* av_packet_clone(const AVPacket* src) {
    AVPacket* ret = av_packet_alloc();     // ① 新对象
    if (!ret) return NULL;
    if (av_packet_ref(ret, src) < 0) {     // ② 共享数据 + 拷字段
        av_packet_free(&ret);
        return NULL;
    }
    return ret;
}
```

**所以 `clone` 之后的 `copy` 是一个全新的对象：字段跟原包一样（可以随便改），数据共享（零拷贝）。** 完美符合我们的需求。

### `clone` 会失败吗？

会。`av_packet_alloc()` 在内存不足时返回 `NULL`。所以代码里检查了：

```cpp
AVPacket* copy = av_packet_clone(&packet);
if (!copy) {
    target.lastError = "av_packet_clone 失败（内存不足？）";
    continue;    // ★ 只跳过这一个目标，其它目标继续
}
```

**注意这里又是"隔离"思想的体现**：内存不够了，只放弃这一个目标的这一次写入，其它目标不受影响。而且**没有把整个 target 标记为断开**——因为这可能只是暂时的内存压力，下一个包可能就成功了。

> **老师点评**：同学们注意区分两种失败的"**严重等级**"：
>
> | 失败 | 后果 | 处理 |
> |---|---|---|
> | `av_packet_clone` 失败 | 本次这个包没发出去 | 记 lastError，**保持连接**，下次再试 |
> | `av_interleaved_write_frame` 失败 | 连接坏了 | 记 lastError，**断开并释放资源**，等重连 |
>
> **不是所有失败都要断连的。** 分清"暂时性错误"和"致命错误"，是网络程序的基本功。

## 3.5 `av_interleaved_write_frame` 与 `av_write_frame` 的区别

顺便把这一对也讲清楚，因为第 8 课之后我们会反复碰到。

| 函数 | 行为 | 优点 | 缺点 |
|---|---|---|---|
| `av_write_frame(fmt, pkt)` | **直接写**，不缓冲、不排序 | 延迟最低 | **你必须保证包是按 dts 递增交给它的**，否则写出来的文件交错错乱 |
| `av_interleaved_write_frame(fmt, pkt)` | **先放进内部缓冲队列，按 dts 排序后再写** | 自动处理音视频交错；容错 | 有额外延迟（一个包的缓冲）；内部队列会占内存 |

**我们用的是 `av_interleaved_write_frame`**，因为：

1. 我们有音视频两条流，**它们的包是交替产生的**（视频 33ms 一帧，音频 21ms 一帧），到达顺序跟 dts 顺序未必一致；
2. 交织函数会帮我们按 dts 排好，保证文件里的音视频是"时间相近的放在一起"，这样播放器读起来才不会在视频和音频之间来回跳（跳来跳去 = 磁盘 seek = 卡顿）。

**代价**：内部会缓冲，所以**必须记得在结束时冲刷**——这就是 `flush()`：

```cpp
void StreamOutput::flush()
{
    for (Impl::Target& target : impl_->targets) {
        if (target.connected && target.ctx)
            av_interleaved_write_frame(target.ctx, nullptr);   // ★ nullptr = 冲刷
    }
}
```

**传 `nullptr` 就是"冲刷交织缓冲区"的信号。** 这是 FFmpeg 的约定：`av_interleaved_write_frame(fmt, nullptr)` = "把缓存的都吐出来"。

> **老师提醒**：`av_write_trailer` 内部其实也会做冲刷，所以严格来说正常关闭时 `flush()` 是"多做一次"。但 **`flush()` 的语义更轻**——它不结束流。在"暂停/恢复"或者"强制立刻把数据推出去（比如关键帧到了，想让服务器马上收到）"的场景下，它是必需的。
>
> 我们把它作为公开接口暴露出来，就是给后面第 10 课（多线程流水线）和第 13 课（断线重连）留的钩子。

## 3.6 写失败隔离：一个目标挂了不能拖垮其它

好，现在到了本章最重要的一节。

### 隔离的三条规则

`writePacket` 循环里，对每个目标都遵守这三条：

```cpp
for (Target& target : targets) {
    // 规则一：不健康的、没这条流的，干脆跳过，不浪费资源
    if (!target.connected || !target.ctx)
        continue;
    AVStream* stream = isVideo ? target.videoStream : target.audioStream;
    if (!stream)
        continue;

    // 规则二：准备工作失败 → 只记错，不株连
    AVPacket* copy = av_packet_clone(&packet);
    if (!copy) {
        target.lastError = "av_packet_clone 失败（内存不足？）";
        continue;                       // ← 注意：不断开
    }

    ... 改字段 ...

    // 规则三：写入失败 → 就地隔离，把这个目标"摘掉"，然后继续下一个
    const int ret = av_interleaved_write_frame(target.ctx, copy);
    av_packet_free(&copy);

    if (ret < 0) {
        target.connected = false;                              // ① 标记断开
        target.lastError = "写入失败: " + avErrorText(ret);      // ② 记原因
        if (log)
            log("[" + target.name + "] 已断开: " + target.lastError);  // ③ 通知上层

        if (target.ctx) {                                      // ④ 释放资源
            if (target.ctx->pb)
                avio_closep(&target.ctx->pb);
            avformat_free_context(target.ctx);
            target.ctx = nullptr;
        }
        target.videoStream = nullptr;                          // ⑤ 清悬垂指针
        target.audioStream = nullptr;
        continue;                                              // ⑥ ★ 继续下一个目标
    }

    target.packetsSent += 1;
    target.bytesSent += packet.size;
    anyOk = true;
}
```

### 返回值 `anyOk`：只要有一路活着就算成功

```cpp
bool anyOk = false;
for (...) { ... anyOk = true; ... }
return anyOk;
```

**这个设计非常关键。** 想一想：如果返回值是"所有目标都成功才算成功"，那只要有一路网络抖动，上层就以为"推流失败了"，可能会停止编码、弹错误框、甚至退出程序——**而实际上另外两路推得好好的**。

**正确的语义是**：

> **`writeVideo()` / `writeAudio()` 返回"至少有一路收到数据了吗？"**
>
> - `true` → 至少一路 OK，继续跑；
> - `false` → **一路都不行了**，这时候上层才该采取措施（停止编码、提示用户、准备重连）。

`open()` 里的逻辑是同一个思路：

```cpp
if (connected == 0) {
    impl_->lastError = "所有推流目标都没连上";
    return false;      // ★ 全部失败才失败
}
say("推流输出就绪: " + std::to_string(connected) + "/"
    + std::to_string(targets.size()) + " 路可用");
```

**"3 路里连上 2 路" = 成功，并且明确告诉你"2/3 路可用"。** 这就是 **"部分成功"（partial success）** 的处理范式。

> **老师点评**：很多同学写网络代码时，习惯性地把所有失败都当成"致命错误"，一失败就 return false 往外抛。
>
> 但对于"多目标分发"这种场景，**部分失败是常态，不是异常**——五个平台里有一两个抽风太正常了。**你的代码必须在"部分失败"的情况下依然正确工作**，这才是健壮的系统。
>
> 这个思想在分布式系统里叫 **"优雅降级"（graceful degradation）**——能用的继续用，不能用的先摘掉，整体服务不中断。

### 隔离的边界在哪儿？

同学们注意，**隔离也不是无条件的**。代码里有两类情况它**不隔离**：

1. **`open()` 阶段**：如果构造参数本身就有问题（`targets` 为空、`videoParams` 为空），直接 `return false`——因为这是**调用方用错了**，不是运行时故障；
2. **参数保存失败**（`avcodec_parameters_copy` 失败）：直接 `close()` 后返回 `false`——因为**没有参数模板，后面所有目标都建不起来**，继续下去没有意义。

**判断标准**：**"这个错误是否只影响一个目标？"**

- 是 → 隔离它，继续；
- 否（影响全局） → 整体失败，让调用方处理。

## 3.7 顺带解决的老大难：AAC 的负时间戳（priming）

这一节是"买一送一"——`StreamOutput` 顺手帮我们解决了一个音频领域的经典坑。

### 问题：AAC 编码器第一包的 pts 是负的

第 8 课（`AudioEncoder`）里有这么一段日志：

```cpp
if (impl_->ctx->initial_padding > 0) {
    say("注意: AAC 有 " + std::to_string(impl_->ctx->initial_padding)
        + " 个样本的编码延迟，第一包的 pts 会是负的");
}
```

**为什么是负的？**

AAC（以及所有基于 MDCT 的变换编码器）有一个特性叫 **`initial_padding`（也叫 priming / encoder delay，编码延迟）**。

简单说：为了正确编码**最开头**的样本，编码器需要"前瞻"一段数据；反过来，它输出的第一个包，其内部音频内容实际上**对应的是输入信号开始之前的一小段**（技术上讲，是 MDCT 窗函数的对称延拓部分）。

FFmpeg 把这个延迟如实反映到了时间戳上：**第一个输出包的 pts = `-initial_padding`**。

对 AAC 来说，`initial_padding` 典型值是 **1024 个样本**。在 48000 Hz 下：

```
-1024 样本 ÷ 48000 Hz = -21.33 毫秒
```

所以音频流的第一个包，pts 是 **-1024**（以 `{1,48000}` 为时间基），换算成毫秒就是 **-21**。

### 这会造成什么问题？

| 现象 | 原因 |
|---|---|
| 播放器静音、或者直接跳过开头 | 负时间戳让播放器的时钟逻辑混乱 |
| ffprobe 报警告 "non-monotonous DTS" | 时间戳不单调 |
| 某些严格的 CDN/服务器拒收 | 认为数据非法 |
| **音画不同步（固定偏移 21ms）** | 音频整体被"提前"了 |

### 解法：把音频起点平移到 0

思路极其简单：**记下第一包的 pts，之后所有音频包的时间戳都减掉它。**

```cpp
int64_t ptsOffset = 0;
if (!isVideo) {                                  // 只对音频做
    if (!audioOffsetReady) {                     // 只在第一次
        audioOffset = packet.pts == AV_NOPTS_VALUE ? 0 : packet.pts;
        audioOffsetReady = true;
        if (audioOffset != 0 && log) {
            log("音频起点偏移 " + std::to_string(audioOffset)
                + "（AAC 编码延迟），已归零");
        }
    }
    ptsOffset = audioOffset;
}
```

然后在每个目标的 `copy` 上减掉：

```cpp
if (ptsOffset != 0) {
    if (copy->pts != AV_NOPTS_VALUE) copy->pts -= ptsOffset;
    if (copy->dts != AV_NOPTS_VALUE) copy->dts -= ptsOffset;
}
```

**几个细节同学们要注意**：

#### 细节一：为什么只对音频做，不对视频做？

因为 H.264 编码器**没有**这种 priming 延迟，视频第一帧的 pts 是正常的（0 或正数）。**这个偏移是 AAC 特有的**。

#### 细节二：为什么在 `copy` 上减，而不是在 `packet` 上减？

因为 `packet` 是 **`const AVPacket&`**（常量引用，调用方的数据，我们不能改），而且**我们要对 N 个目标各减一次**——在 `copy` 上减，每个目标独立，原包不动。

#### 细节三：为什么 `AV_NOPTS_VALUE` 要判断？

**`AV_NOPTS_VALUE`** 是 FFmpeg 表示"**这个时间戳不存在/无意义**"的哨兵值，它的值是 `INT64_MIN`。

如果你对它做减法（`copy->pts -= ptsOffset`），会**整数下溢**（underflow），得到一个巨大的正数——比直接崩还糟，因为它会静默地产生一个荒谬的时间戳。

**所以任何对 pts/dts 的算术运算，都必须先检查 `!= AV_NOPTS_VALUE`。** 这是铁律。

#### 细节四：`audioOffsetReady` 是"一次性"的

```cpp
bool audioOffsetReady = false;
int64_t audioOffset = 0;
```

**只在第一个音频包到达时计算一次，之后一直用这个值。** 因为如果每包都重新算，那等于什么都不做（每包都减自己，全变 0）。

**而且它在 `release()` 里被复位**：

```cpp
audioOffsetReady = false;
audioOffset = 0;
```

**为什么？** 因为 `close()` 之后你可能重新 `open()`——那时候编码器是新开的，第一包的 pts 还是 -1024，需要重新测量。**状态复位是"可重用对象"的基本要求。**

#### 细节五：`release()` 和 `close()` 的关系

```cpp
void StreamOutput::close() { impl_->release(); }
```

`close()` 只是转发，真正的清理在 `release()` 里。这样设计的好处：**`Impl` 内部可以自己调 `release()` 而不用绕回外层类**（比如 `open()` 失败时）。

### 测试怎么验证这一点？

第 7 章的测试里有两条断言：

```cpp
// probeFlv 里，遍历所有包时检查
if (pts < 0) info.sawNegativePts = true;

// main 里
check(!info.sawNegativePts, "没有任何负时间戳（AAC 的 priming 被归零了）");
```

**"把文件读回来，遍历每一个包，看看有没有 pts < 0。"** ——简单粗暴但极其有效。

---

好，三个预习章节讲完了。同学们歇口气，我们进入代码。

---

# 第 4 章 逐行教学（一）：`StreamOutput.h`（53 行）

先看头文件。**头文件是"契约"——它定义了这个模块对外承诺什么、要求什么。** 同学们以后读任何 C++ 模块，都养成习惯：**先读头文件，再读实现。**

## 4.1 完整代码

```cpp
 1  #pragma once
 2
 3  #include "core/common/Common.h"
 4
 5  extern "C" {
 6  #include <libavcodec/avcodec.h>
 7  #include <libavutil/rational.h>
 8  }
 9
10  #include <memory>
11  #include <string>
12  #include <vector>
13
14  struct OutputTargetInfo {
15      std::string name;
16      std::string url;
17      bool connected = false;
18      int64_t packetsSent = 0;
19      int64_t bytesSent = 0;
20      std::string lastError;
21  };
22
23  class StreamOutput {
24  public:
25      StreamOutput();
26      ~StreamOutput();
27
28      StreamOutput(const StreamOutput&) = delete;
29      StreamOutput& operator=(const StreamOutput&) = delete;
30
31      bool open(const std::vector<StreamTarget>& targets,
32                const AVCodecParameters* videoParams,
33                const AVCodecParameters* audioParams, LogCallback log);
34
35      void close();
36
37      bool writeVideo(const AVPacket& packet, AVRational encoderTimeBase);
38      bool writeAudio(const AVPacket& packet, AVRational encoderTimeBase);
39
40      void flush();
41
42      std::vector<OutputTargetInfo> targets() const;
43
44      int connectedCount() const;
45
46      int reconnectDisconnected();
47
47      const std::string& lastError() const;
48
49  private:
50      struct Impl;
51      std::unique_ptr<Impl> impl_;
52  };
```

## 4.2 逐段讲解

### 第 1~12 行：头文件保护与依赖

```cpp
#pragma once

#include "core/common/Common.h"

extern "C" {
#include <libavcodec/avcodec.h>
#include <libavutil/rational.h>
}

#include <memory>
#include <string>
#include <vector>
```

| 行 | 内容 | 为什么 |
|---|---|---|
| 1 | `#pragma once` | 头文件保护，防止重复包含。项目里统一用它，不用 `#ifndef` 守卫（第 1 课定的规矩） |
| 3 | `core/common/Common.h` | 拿到 `StreamTarget`、`LogCallback` 这两个**项目内公共契约** |
| 5~8 | `extern "C" { ... }` | **FFmpeg 是 C 库**，头文件必须包在 `extern "C"` 里，否则 C++ 的名字改写（name mangling）会导致链接时找不到符号 |
| 6 | `libavcodec/avcodec.h` | 需要 `AVCodecParameters` 和 `AVPacket` 的**完整定义** |
| 7 | `libavutil/rational.h` | 需要 `AVRational`（时间基） |
| 10 | `<memory>` | `std::unique_ptr<Impl>` —— PImpl 惯用法 |
| 11~12 | `<string>` `<vector>` | 略 |

> **老师注意**：这里只 include 了 **`avcodec.h` + `rational.h`** 两个头文件。**`libavformat/avformat.h` 一个字都没有！**
>
> 这就是 PImpl 的第一个好处——**FFmpeg 最庞大、编译最慢的 `avformat.h` 被挡在了 `.cpp` 里**。任何 `#include "StreamOutput.h"` 的翻译单元都不会被它拖慢。
>
> 一个真实的对比：`avformat.h` 展开后有 **5 万多行**，编译一次要好几百毫秒。我们的项目里有十几个文件会 include 这个头文件，省下来的编译时间相当可观。

### 第 14~21 行：`OutputTargetInfo` —— 一个目标的"体检报告"

```cpp
struct OutputTargetInfo {
    std::string name;          // 目标名字（比如 "斗鱼"、"B站"、"录像"）
    std::string url;           // 目标地址
    bool connected = false;    // 现在还连着吗
    int64_t packetsSent = 0;   // 累计发出去多少个包
    int64_t bytesSent = 0;     // 累计发出去多少字节
    std::string lastError;     // 最后一次错误的原因（空 = 没出过错）
};
```

这是一个**纯数据的值结构体（value struct / DTO）**，由 `targets()` 返回给调用方。

**为什么需要它？** 因为调用方（未来的 Qt 界面）需要展示这样的东西：

```
┌────────────────────────────────────────────────────┐
│ 推流目标                                            │
│  ● 斗鱼   rtmp://send.douyu.com/...   已发送 12.4 MB │
│  ● B站    rtmp://live-push.bilivi...  已发送 12.4 MB │
│  ○ 本地录像 D:/Record/2026-10-20.flv   已发送 12.4 MB │
│  ✗ 备用线  rtmp://backup...  断开：Connection refused│
└────────────────────────────────────────────────────┘
```

**设计上有三个讲究，同学们学一下**：

#### 讲究一：返回"快照"而不是"引用"

```cpp
std::vector<OutputTargetInfo> targets() const;   // 按值返回
```

而不是：

```cpp
const std::vector<Target>& targets() const;      // ✗ 会把内部结构暴露出去
```

**"快照" 的好处**：

1. **解耦**：内部 `Target` 结构里全是 FFmpeg 裸指针（`AVFormatContext*` 等），绝不能暴露给外部；
2. **线程安全友好**：调用方拿到的是一份拷贝，随便看，不怕内部状态在变（第 10 课多线程化之后这一点会救命）；
3. **可以自由演进**：以后内部 `Target` 加字段、改结构，都不影响调用方。

> 这就是所谓的 **"信息隐藏"（information hiding）**：**对外暴露"你该知道的"，隐藏"我怎么实现的"。**

#### 讲究二：全字段带默认值

```cpp
bool connected = false;
int64_t packetsSent = 0;
```

**C++11 的类内成员初始化器**。好处：默认构造出来就是"干净"的状态，**不会出现未初始化的垃圾值**。

同学们，C++ 里"忘记初始化成员变量"是 bug 的经典来源。养成习惯：**每个成员都给个初始值，哪怕只是 `= 0`。**

#### 讲究三：名字不叫 `getTargets()` 而叫 `targets()`

这是**现代 C++ 的风格倾向**：当一个函数**没有参数、只是取一个属性**时，省略 `get` 前缀，读起来更顺：

```cpp
output.targets();        // "output 的 targets"
output.getTargets();     // 啰嗦
```

Qt 的 API 就是这个风格（`widget.width()`、`widget.size()`）。我们的项目用了 Qt，保持一致。

### 第 23~29 行：构造、析构、禁用拷贝

```cpp
class StreamOutput {
public:
    StreamOutput();
    ~StreamOutput();

    StreamOutput(const StreamOutput&) = delete;
    StreamOutput& operator=(const StreamOutput&) = delete;
```

**第 28~29 行：`= delete` 显式禁用拷贝构造和拷贝赋值。**

为什么必须禁？因为这个类**持有独占的资源**：

- N 个 `AVFormatContext*`（每条都对应一个 TCP 连接或一个文件句柄）；
- 两份 `AVCodecParameters*`；
- 一个 `std::unique_ptr<Impl>`（unique_ptr 本身就不可拷贝）。

如果允许拷贝，两个 `StreamOutput` 对象会**指向同一批 `AVFormatContext`**，然后析构时**双重释放**——程序当场崩溃，而且是那种"有时候崩有时候不崩"的玄学 bug。

**`= delete` 是编译期的硬保护**：谁敢写 `StreamOutput a = b;`，编译器直接报错，把问题拦在编译阶段。

> **老师点评**：这是 C++ 的 **"Rule of Five / Rule of Zero"** 里最基本的一条。
>
> **判断标准很简单**：**你的类管理了独占资源吗？是 → 禁用拷贝（或者实现移动语义）。**
>
> 顺便复习一下，我们这个项目的三个核心类全都是这个套路：
> ```cpp
> VideoEncoder(const VideoEncoder&) = delete;
> AudioEncoder(const AudioEncoder&) = delete;
> StreamOutput(const StreamOutput&) = delete;
> ```
> **一致的风格比什么都重要**——新人看到第三个类时，不用问就知道规矩。

**第 25~26 行：构造函数和析构函数都声明在这里，但定义在 .cpp 里。** 这一点**至关重要**，下面 4.3 节专门讲。

### 第 31~33 行：`open()` —— 一次建好 N 条管道

```cpp
bool open(const std::vector<StreamTarget>& targets,
          const AVCodecParameters* videoParams,
          const AVCodecParameters* audioParams, LogCallback log);
```

四个参数，我们逐个看它"要求调用方提供什么"：

| 参数 | 类型 | 含义 | 能不能为空 |
|---|---|---|---|
| `targets` | `const std::vector<StreamTarget>&` | 要分发到哪些目标（`StreamTarget{platform, url}`） | **不能为空**，空了直接返回 false |
| `videoParams` | `const AVCodecParameters*` | 视频编码参数模板（含 SPS/PPS） | **不能为空**，直播必须有视频 |
| `audioParams` | `const AVCodecParameters*` | 音频编码参数模板 | **可以为空** → 那就只有视频流 |
| `log` | `LogCallback`（按值传） | 日志回调 `void(const std::string&)` | 可以为空 → 内部所有地方都要判空 |

**几个设计点**：

#### 点一：`videoParams` 是 `const` 指针，且我们内部会深拷贝

```cpp
const AVCodecParameters* videoParams
```

`const` 明确表达："**我只读你的，不改你的**"。而 `StreamOutput` 内部用 `avcodec_parameters_copy` 做了一份深拷贝（5.6 节讲）。

**这个组合（`const` 入参 + 内部深拷贝）是"拿快照"的标准写法**——调用方可以 `open()` 完立刻销毁编码器，`StreamOutput` 毫发无损。

#### 点二：`log` 按值传 `LogCallback`，内部 `std::move`

```cpp
bool open(..., LogCallback log);            // 按值
impl_->log = std::move(log);                // .cpp 里 move 进成员
```

为什么按值传而不是 `const LogCallback&`？

因为我们要**把它存起来**（存到 `impl_->log`）。按值传 + `move` 的组合，对调用方传进来的各种情况都最优：

| 调用方写法 | 按值 + move 的开销 |
|---|---|
| `open(..., nullptr)` | 构造一个空 `std::function`，零分配 |
| `open(..., myLog)`（左值） | 1 次拷贝 + 1 次 move（反正要存一份，拷贝躲不掉） |
| `open(..., [&](const std::string& s){...})`（右值） | 1 次 move，**零拷贝** |

**这是 C++11 之后"要存储的参数就按值传"的标准建议**（Scott Meyers《Effective Modern C++》Item 41）。

#### 点三：`LogCallback` 从哪来的

`Common.h` 里定义的（第 3 课）：

```cpp
using LogCallback = std::function<void(const std::string&)>;
```

**为什么不直接 `printf`？** 因为：

1. **库不该决定输出到哪**——GUI 程序要打到界面上，服务程序要打到日志文件里，测试程序要收集到 vector 里（我们今天的测试就是这么干的！）；
2. **`std::function` 什么都能装**：普通函数、lambda、`std::bind`、成员函数绑定器。

看看测试里是怎么用的（第 7 章详讲）：

```cpp
std::vector<std::string> logs;
const auto log = [&logs](const std::string& message) {
    logs.push_back(message);          // 收集起来
    std::printf("  [输出] %s\n", message.c_str());   // 同时打印
};
```

**一个回调，同时做了"打印"和"收集"两件事。** 如果当初写死 `printf`，测试就没法自动校验日志内容了。

### 第 35~40 行：关闭与写入

```cpp
void close();

bool writeVideo(const AVPacket& packet, AVRational encoderTimeBase);
bool writeAudio(const AVPacket& packet, AVRational encoderTimeBase);

void flush();
```

| 函数 | 参数 | 返回值的含义 |
|---|---|---|
| `close()` | 无 | 无。幂等，可重复调用 |
| `writeVideo(pkt, tb)` | 包 + **编码器时间基** | `true` = 至少一路收到了；`false` = 一路都没有 |
| `writeAudio(pkt, tb)` | 同上 | 同上 |
| `flush()` | 无 | 无。冲刷所有输出的内部缓冲 |

**重点说说第二参数 `encoderTimeBase`。**

为什么写个包还要传时间基？因为——

> **`AVPacket` 里的 pts/dts 只是一个整数，它本身不携带"单位"信息。**

`StreamOutput` 收到一个 `pts = 720` 的包，它**无法知道**这是"第 720 帧（24 秒）"还是"720 毫秒"还是"720 个采样点（15 毫秒）"。**必须调用方告诉它。**

调用方怎么知道的？从编码器问：

```cpp
output.writeVideo(*packet, video.timeBase());    // {1, 30}
output.writeAudio(*packet, audio.timeBase());    // {1, 48000}
```

> **老师点评**：这个设计看着"多传了一个参数"，其实是**正确的**。
>
> 另一种设计是让 `StreamOutput` 在 `open()` 时就记住两个时间基。但那样有个问题：**如果视频编码器被重建了（分辨率变了、帧率变了），时间基可能也变了**，而 `StreamOutput` 还拿着旧值，就会算错。
>
> **"每次调用都把当前的时间基带过来"**，虽然啰嗦一点，但**永远是对的**。这就是"**显式优于隐式**"。

**另外注意 `const AVPacket&`**：常量引用，明确表达"我不改你的包，也不拿走你的所有权"。调用方可以放心地继续持有这个包（我们的 `AVPacketPtr` 智能指针会在合适的时候释放它）。

### 第 42~47 行：状态查询

```cpp
std::vector<OutputTargetInfo> targets() const;   // 所有目标的快照
int connectedCount() const;                      // 还连着几路
int reconnectDisconnected();                     // 重连断开的，返回成功个数
const std::string& lastError() const;            // 最后一次全局错误
```

| 函数 | const? | 用途 |
|---|---|---|
| `targets()` | ✅ const | 界面展示；判断每一路的健康状态 |
| `connectedCount()` | ✅ const | 快速判断"还有路活着吗"；返回 0 就该告警了 |
| `reconnectDisconnected()` | ❌ 非 const | 会**修改**内部状态（重建 `ctx`），所以不能是 const |
| `lastError()` | ✅ const | `open()` 失败后的原因 |

**`lastError()` 返回 `const std::string&`** —— 返回常引用，避免拷贝，同时禁止调用方修改。这是标准做法。

> **老师提醒一个易错点**：返回 `const std::string&` 有一个生命周期陷阱——
> ```cpp
> const char* p = output.lastError().c_str();   // ✗ 危险！
> ```
> 如果 `output` 被销毁，或者 `lastError` 被修改，这个 `p` 就悬空了。
> **安全的做法**：要么立刻用掉，要么拷贝一份 `std::string s = output.lastError();`。
>
> 第 7 章的测试里这么写是安全的，因为 `output` 对象还在作用域内：
> ```cpp
> std::printf("  open() 失败原因: %s\n", output.lastError().c_str());
> ```

### 第 49~52 行：PImpl 惯用法 ★

```cpp
private:
    struct Impl;                        // ① 只有前向声明，没有定义
    std::unique_ptr<Impl> impl_;        // ② 通过指针持有
```

这就是 **PImpl（Pointer to Implementation，也叫"编译防火墙"、Cheshire Cat 惯用法、opaque pointer）**。

#### 它解决了什么问题？

**问题：C++ 的头文件暴露得太多了。**

如果不用 PImpl，头文件就得这么写：

```cpp
// ✗ 不用 PImpl 的写法
#include <libavformat/avformat.h>      // 5 万行！
#include <vector>

struct StreamOutput::Target {          // 内部结构也必须写在头文件里
    AVFormatContext* ctx;
    AVStream* videoStream;
    ...
};

class StreamOutput {
    std::vector<Target> targets;
    AVCodecParameters* videoParams;
    ...
};
```

后果：

1. **编译传染**：所有 `#include "StreamOutput.h"` 的文件都被迫编译 5 万行的 `avformat.h`；
2. **实现细节暴露**：内部结构、私有成员全写在头文件里，调用方看得一清二楚（想偷偷改都不行）；
3. **改实现就要重编一片**：在 `Impl` 里加一个私有成员，**所有 include 这个头文件的 .cpp 全部重新编译**；
4. **ABI 不稳定**：如果你是做库的，改私有成员会导致二进制不兼容。

**用了 PImpl 之后**：

```
                 调用方看到的                     实现方才知道的
              ┌──────────────────┐          ┌──────────────────────────┐
              │ StreamOutput.h   │          │ StreamOutput.cpp         │
              │  class StreamOutput {       │  struct StreamOutput::Impl {
              │    struct Impl;  │          │    std::vector<Target>   │
              │    unique_ptr<Impl> impl_; ─┼─→  AVFormatContext* ...   │
              │  };              │          │    ... 389 行全在这里     │
              └──────────────────┘          └──────────────────────────┘
                  只有 53 行                      实现细节完全隐藏
                  不需要 avformat.h              随便改，不影响调用方
```

#### 它的代价

不是免费的，有两个代价：

1. **每次访问多一次指针间接寻址**（`impl_->xxx` 而不是 `xxx`）——可忽略，编译器会优化；
2. **多一次堆分配**（`make_unique<Impl>()`）——可忽略，`open()` 才调一次；
3. **代码稍微啰嗦**——所有成员访问都要加 `impl_->`。

**对于"一个进程里只有一两个实例、但被很多文件 include"的模块（正是我们的场景），PImpl 稳赚不赔。**

#### ★ 一个必须知道的坑：析构函数不能是 inline 的

```cpp
class StreamOutput {
public:
    StreamOutput();        // ① 声明在这里
    ~StreamOutput();       // ② 声明在这里
private:
    struct Impl;           //    不完整类型（incomplete type）
    std::unique_ptr<Impl> impl_;
};
```

**关键问题**：`std::unique_ptr<T>` 的析构函数需要 `T` 是**完整类型**（因为要 `delete ptr`，得知道 `T` 的大小和析构函数）。

- 如果析构函数**定义在头文件里（隐式 inline）**，编译器在那个位置看到的 `Impl` 还是不完整类型 → **编译错误**；
- 所以**析构函数必须在 .cpp 里定义**（那时候 `Impl` 已经完整了）。

看我们的代码：

```cpp
// StreamOutput.h 第 26 行：只声明
~StreamOutput();

// StreamOutput.cpp 第 284~287 行：定义在 .cpp 里
StreamOutput::~StreamOutput()
{
    close();
}
```

✅ **完全正确。**

**常见错误信息**（MSVC）：
```
error C2338: static_assert failed: 'can't delete an incomplete type'
error C2027: use of undefined type 'StreamOutput::Impl'
```

**如果同学们以后遇到这个错误，第一反应就是：析构函数（还有所有会销毁 unique_ptr 的函数）是不是写成 inline 了？**

> **老师补充**：同理，**拷贝构造/拷贝赋值也不能 inline**（虽然我们 `= delete` 了，没问题）。还有 `std::make_unique<Impl>()` 也只能在 .cpp 里调。
>
> 一句话总结规则：**头文件里，任何需要 `Impl` 完整定义的操作，都只能"声明"，不能"定义"。**

## 4.3 头文件设计总结：一张"契约清单"

同学们，读完这个头文件，我们应该能**不看 .cpp 就回答**这些问题：

| 问题 | 从头文件得到的答案 |
|---|---|
| 这个模块干什么？ | 把 `AVPacket` 封装成 FLV 分发到多个目标 |
| 我需要先准备什么？ | 视频 `AVCodecParameters`（必需）、音频（可选）、目标列表（非空）、日志回调（可选） |
| 我怎么给它数据？ | `writeVideo(pkt, tb)` / `writeAudio(pkt, tb)`，包是常量引用，时间基要显式传 |
| 我怎么知道成功了没？ | 返回值（至少一路 OK？）+ `targets()` 快照 + `connectedCount()` + `lastError()` |
| 我怎么收尾？ | `flush()` 冲刷 → `close()` 关闭（析构也会自动 close） |
| 出错了怎么办？ | `lastError()` 看原因；断了的用 `reconnectDisconnected()` 重连 |
| 我能拷贝它吗？ | 不能，`= delete` |
| 它需要哪些外部依赖？ | FFmpeg 的 `avcodec.h` + `rational.h`（`avformat` 被隐藏了） |

**这就是一个好头文件应该达到的效果——自解释。**

---

# 第 5 章 逐行教学（二）：`StreamOutput.cpp`（389 行）

终于到主菜了。389 行，我们按"从上到下、逐函数"的顺序啃。

先给一张全景图：

```
第  1~ 18 行  头文件和 avErrorText            ← 工具
第 20~ 22 行  两个常量                        ← 配置
第 24~119 行  struct Impl（含 Target + release）
第 121~189 行  Impl::openTarget               ← 建管道
第 191~207 行  Impl::closeTarget              ← 拆管道
第 209~278 行  Impl::writePacket              ← ★ 核心：分发 + 隔离
第 280~282 行  StreamOutput::StreamOutput
第 284~287 行  StreamOutput::~StreamOutput
第 289~358 行  StreamOutput::open             ← 建 N 条管道
第 360~363 行  StreamOutput::close
第 365~373 行  writeVideo / writeAudio
第 375~381 行  StreamOutput::flush
第 383~400 行  StreamOutput::targets
第 402~410 行  StreamOutput::connectedCount
第 412~428 行  StreamOutput::reconnectDisconnected
第 430~433 行  StreamOutput::lastError
```

## 5.1 第 1~22 行：文件头与两个常量

```cpp
 1  #include "core/output/StreamOutput.h"
 2
 3  extern "C" {
 4  #include <libavformat/avformat.h>
 5  #include <libavutil/error.h>
 6  #include <libavutil/opt.h>
 7  }
 8
 9  #include <cstring>
10
11  namespace {
12
13  std::string avErrorText(int err)
14  {
15      char buffer[AV_ERROR_MAX_STRING_SIZE] = {};
16      av_strerror(err, buffer, sizeof(buffer));
17      return buffer;
18  }
19
20  constexpr AVRational kFlvTimeBase = {1, 1000};
21
22  constexpr const char* kIoTimeout = "5000000";
23
24  }
```

### 第 1 行：包含自己的头文件

```cpp
#include "core/output/StreamOutput.h"
```

**规则：每个 .cpp 文件的第一行都应该是 `#include "自己的头文件"`。**

为什么？因为它能**自动检查头文件的自足性（self-containedness）**——如果头文件少 include 了什么，这一行就会暴露出来，而不是等到别人 include 它时才炸。

**注意路径是 `core/output/StreamOutput.h`** 而不是 `"StreamOutput.h"`。因为 CMake 里配的 include 目录是 `${CMAKE_CURRENT_SOURCE_DIR}/../..`（也就是 `src/`），所以从 `src/` 开始写全路径。

> 项目里所有跨模块的 include 都遵守这个约定：
> ```cpp
> #include "core/common/Common.h"
> #include "core/encode/VideoEncoder.h"
> #include "core/output/StreamOutput.h"
> ```
> **好处**：一眼就能看出这个文件的依赖方向，而且**不会跟系统头文件重名**。

### 第 3~7 行：FFmpeg 头文件

```cpp
extern "C" {
#include <libavformat/avformat.h>    // 封装层：alloc_output_context2 / new_stream / write_header ...
#include <libavutil/error.h>         // av_strerror / AV_ERROR_MAX_STRING_SIZE
#include <libavutil/opt.h>           // av_opt_*（本文件其实没直接用，但保留着）
}
```

**这里的 `avformat.h` 就是被 PImpl 挡在门外的那个大头文件。** 现在它在 .cpp 里，只有这一个翻译单元需要编译它。

> **老师提醒（小瑕疵）**：第 6 行的 `<libavutil/opt.h>` 在本文件里其实没被用到（我们用的是 `av_dict_set`，它在 `libavutil/dict.h` 里，而 `dict.h` 被 `avformat.h` 间接包含了）。这是一处**多余的 include**，无害但不够干净。同学们自己的代码里，include 要遵循"用到了才包含"的原则。

### 第 9 行

```cpp
#include <cstring>
```

**这一行也是多余的**——本文件里没有用任何 `<cstring>` 的函数（`strlen` / `memcpy` / `strcmp` 都没有）。

> **老师点评**：这两处多余的 include 不是什么大错，但它反映了一个常见问题——**"防御性 include"**。
>
> 很多同学写代码时会把"可能用到"的头文件全堆上去，觉得"多 include 又不会错"。但代价是**编译变慢**，而且**掩盖了真实的依赖关系**（你要是删掉一个 include 还能编过，说明你根本没用它）。
>
> **建议**：include 只写实际用到的。现代 IDE（VS、CLion、VSCode + clangd）都能自动提示"未使用的 include"。

### 第 11~24 行：匿名命名空间

```cpp
namespace {

std::string avErrorText(int err) { ... }

constexpr AVRational kFlvTimeBase = {1, 1000};

constexpr const char* kIoTimeout = "5000000";

}
```

**匿名命名空间（anonymous namespace）** 是 C++ 里替代 C 语言 `static` 的**推荐做法**。

它的效果：**里面的符号具有"内部链接"（internal linkage）——只在当前翻译单元可见，不会跟其它 .cpp 文件里的同名符号冲突。**

| 写法 | 作用范围 | 现代 C++ 推荐度 |
|---|---|---|
| `static void foo() {}` | 内部链接（C 风格） | ⚠️ 可用，但不够彻底 |
| `namespace { void foo() {} }` | 内部链接（C++ 风格） | ✅ **推荐** |
| `void foo() {}`（全局） | 外部链接，**可能跟别人撞名** | ❌ 库代码绝对禁止 |

**为什么必须内部链接？** 因为 `avErrorText` 这个函数名太普通了——`VideoEncoder.cpp`、`AudioEncoder.cpp`、`StreamOutput.cpp` 里**全都有一份一模一样的 `avErrorText`**！

如果是外部链接，链接器会报：

```
error LNK2005: "class std::basic_string<char,...> __cdecl avErrorText(int)" 已经在 VideoEncoder.obj 中定义
```

**多重定义错误。** 而匿名命名空间让每个 .cpp 里的 `avErrorText` 都是"自己文件私有的"，互不干扰。

> **老师点评**：这里其实暴露了一个**小小的重复**——三个文件里都复制了一份 `avErrorText`。
>
> **理想的做法**是把它抽到一个公共的地方（比如 `src/core/common/AvError.h` 里做成 inline 函数）。但作者选择了"每个文件复制一份"，这在**模块独立性优先**的小项目里是可以接受的权衡：**每个模块自包含，拷走就能用。**
>
> 同学们要知道这个 trade-off：
> - **复制**：简单、独立，但改一处要改三处（DRY 原则违反）；
> - **抽取**：无重复，但引入了模块间依赖。
>
> 这里因为函数只有 6 行且极其稳定，**复制的代价小于依赖的代价**，可以接受。但如果这个函数有 50 行，就必须抽。

### 第 13~18 行：`avErrorText`

```cpp
std::string avErrorText(int err)
{
    char buffer[AV_ERROR_MAX_STRING_SIZE] = {};
    av_strerror(err, buffer, sizeof(buffer));
    return buffer;
}
```

**FFmpeg 的错误码是负数**（`-EINVAL`、`-ENOENT`、`-ETIMEDOUT` 之类），这个函数把它翻译成人话。

| FFmpeg 错误码 | 值 | `av_strerror` 输出 | 什么时候会碰到 |
|---|---|---|---|
| `AVERROR(EAGAIN)` | -11 | "Resource temporarily unavailable" | 编码器暂时没产出（**不是错误**） |
| `AVERROR_EOF` | -541478725 | "End of file" | 编码器已排空（**不是错误**） |
| `AVERROR(ENOENT)` | -2 | "No such file or directory" | 文件路径不存在 |
| `AVERROR(EACCES)` | -13 | "Permission denied" | 没权限写这个文件 |
| `AVERROR(ECONNREFUSED)` | -111 | "Connection refused" | **rtmp 连不上（我们测试里的场景）** |
| `AVERROR(ETIMEDOUT)` | -110 | "Connection timed out" | **超过 `rw_timeout`** |
| `AVERROR(ENOMEM)` | -12 | "Cannot allocate memory" | 内存不足 |

**三个细节**：

1. **`char buffer[AV_ERROR_MAX_STRING_SIZE] = {};`** —— `AV_ERROR_MAX_STRING_SIZE` 是 64。`= {}` 是**零初始化**，保证即使 `av_strerror` 失败（比如错误码不认识），buffer 也是个合法的空字符串，不会输出乱码；
2. **`av_strerror` 的返回值被忽略了** —— 理论上应该检查（`if (av_strerror(...) < 0) return "未知错误";`）。但从 FFmpeg 4.x 开始，即使遇到不认识的错误码，它也会往 buffer 里写一句 "Unknown error occurred"，所以忽略是安全的；
3. **返回 `std::string`（按值）** —— 依赖编译器的 **RVO/NRVO（返回值优化）**，零拷贝。

### 第 20 行：`kFlvTimeBase`

```cpp
constexpr AVRational kFlvTimeBase = {1, 1000};
```

第 1 章讲过，FLV 的时间戳是毫秒，所以必须是 `{1, 1000}`。

**`constexpr`** 意味着编译期常量，可以放匿名命名空间里（内部链接），不会有多重定义问题。

### 第 22 行：`kIoTimeout`

```cpp
constexpr const char* kIoTimeout = "5000000";
```

**5,000,000 微秒 = 5 秒**，传给 `avio_open2` 的 `rw_timeout` 选项（第 2.4 节讲过）。

**为什么写成字符串常量？** 因为 `av_dict_set` 的值参数就是 `const char*`。

> **老师建议一个小改进**：这里的 `"5000000"` 缺少单位提示，读代码的人要愣一下才知道是微秒。更好的写法是：
> ```cpp
> constexpr int kIoTimeoutUs = 5 * 1000 * 1000;   // 5 秒（单位：微秒）
> constexpr const char* kIoTimeout = "5000000";
> ```
> 或者干脆加个注释。同学们写常量时，**把单位写进名字或者注释里**（`kTimeoutMs`、`kSizeBytes`、`kDurationUs`），这是能省下无数调试时间的好习惯。

## 5.2 第 24~119 行：`struct StreamOutput::Impl`

这是整个模块的"状态中心"。我们分段看。

### 5.2.1 第 26~39 行：`struct Target` —— 一个目标的全部状态

```cpp
26  struct StreamOutput::Impl {
27      struct Target {
28          std::string name;
29          std::string url;
30
31          AVFormatContext* ctx = nullptr;
32          AVStream* videoStream = nullptr;
33          AVStream* audioStream = nullptr;
34
35          bool connected = false;
36          std::string lastError;
37
38          int64_t packetsSent = 0;
39          int64_t bytesSent = 0;
40      };
```

**这是嵌套结构体（`Impl` 里面再嵌一个 `Target`）。** 好处：`Target` 的作用域被限制在 `Impl` 里，不会污染外层命名空间。

八个字段，分成四组：

| 组 | 字段 | 谁填的 | 说明 |
|---|---|---|---|
| **身份** | `name`、`url` | `open()` 从 `StreamTarget` 拷进来 | `name` 是显示用的（"斗鱼"），`url` 是地址 |
| **资源** | `ctx`、`videoStream`、`audioStream` | `openTarget()` | **FFmpeg 的三大句柄** |
| **状态** | `connected`、`lastError` | `openTarget()` / `writePacket()` | 健康状态与故障原因 |
| **统计** | `packetsSent`、`bytesSent` | `writePacket()` 成功时累加 | 界面展示 / 监控用 |

**注意第 32~33 行**：`videoStream` 和 `audioStream` 是**两个独立的指针**，而不是一个数组。因为：

- 最多两条流（视频、音频），用两个具名指针**语义最清晰**；
- **音频流可能不存在**——这时 `audioStream == nullptr`，代码里到处靠 `if (!stream) continue;` 判断。

如果用数组 `AVStream* streams[2]`，语义就模糊了（"streams[1] 是空"是"没有音频"还是"出错了"？）。

> **老师点评**：**用"具名"而不是"编号"来表达语义，可读性提升巨大。** 这是个小技巧但很实用。

**为什么 `Target` 里要存 `name` 和 `url` 的副本，而不是只存索引？**

因为 `reconnectDisconnected()` 要对断掉的目标**重新建管道**，那时候需要 url。如果只存个 `size_t index` 指向外部 vector，外部 vector 一变就失效了。**自己存一份最安全。**

### 5.2.2 第 41~50 行：全局状态

```cpp
41      std::vector<Target> targets;
42
43      AVCodecParameters* videoParams = nullptr;
44      AVCodecParameters* audioParams = nullptr;
45
46      bool audioOffsetReady = false;
47      int64_t audioOffset = 0;
48
49      LogCallback log;
50      std::string lastError;
```

| 字段 | 作用 | 生命周期 |
|---|---|---|
| `targets` | N 个目标 | `open()` 时 `resize`，`release()` 时 `clear()` |
| `videoParams` | 视频参数**模板**（深拷贝，含 SPS/PPS） | `open()` 时分配，`release()` 时释放 |
| `audioParams` | 音频参数模板（可空） | 同上 |
| `audioOffsetReady` | "音频偏移算过了吗"（一次性标志） | `writePacket` 首次处理音频时置 true |
| `audioOffset` | AAC priming 偏移量 | 同上，`release()` 时复位 |
| `log` | 日志回调 | `open()` 时 move 进来 |
| `lastError` | **全局**最后一次错误（区别于每个 target 的 `lastError`） | `open()` 失败时填 |

**注意区分两个 `lastError`**：

```
impl_->lastError        ← 全局的：整个模块最后一次失败的原因
                          （比如 "没有配置任何推流目标"、"所有推流目标都没连上"）

impl_->targets[i].lastError   ← 单个目标的：这一路为什么断了
                                （比如 "连接失败: Connection refused"）
```

**这个区分很重要**：全局错误回答"**这事儿整体黄了没有**"，单目标错误回答"**这一路具体怎么了**"。

### 5.2.3 第 52~55 行：内部方法声明

```cpp
52      bool openTarget(Target& target);
53      void closeTarget(Target& target, bool writeTrailer);
54      bool writePacket(const AVPacket& packet, AVRational timeBase, bool isVideo);
55
56      void release()
57      {
58          ...（第 101~118 行，下面讲）
59      }
```

三个内部方法声明在 `Impl` 里，**定义在 `Impl` 外面**（`bool StreamOutput::Impl::openTarget(Target& target) { ... }`）。

这种写法是必要的：因为 `Target` 的定义在 `Impl` 内部，而成员函数的定义如果也写在类内，会让 389 行的代码缩进三层，很难读。

> 顺便说一句，**`release()` 是唯一一个定义在类内的函数**，因为它只有 18 行，而且它是"清理逻辑"，跟数据成员放在一起读起来更连贯。

### 5.2.4 第 56~118 行：`release()` —— 一键全拆

```cpp
 56      void release()
 57      {
 58          for (Target& target : targets)
 59              closeTarget(target, true);
 60          targets.clear();
 61
 62          if (videoParams) {
 63              avcodec_parameters_free(&videoParams);
 64              videoParams = nullptr;
 65          }
 66          if (audioParams) {
 67              avcodec_parameters_free(&audioParams);
 68              audioParams = nullptr;
 69          }
 70
 71          audioOffsetReady = false;
 72          audioOffset = 0;
 73      };
```

第 2.5 节讲过释放顺序，这里只补充三点：

#### 补充一：`closeTarget(target, true)` 的 `true` 是什么意思

```cpp
closeTarget(target, /*writeTrailer=*/true);
```

**`true` = "这是正常关闭，请写容器尾（trailer）"。**

对比 `openTarget()` 开头那次调用：

```cpp
bool StreamOutput::Impl::openTarget(Target& target)
{
    closeTarget(target, false);   // ← false：不写 trailer
    ...
}
```

**为什么重连前不写 trailer？** 因为上一次是**异常中断**的（写失败被踢掉了），那条连接已经废了，你再对它调 `av_write_trailer` 是徒劳的（还会阻塞）。**直接拆掉重建即可。**

#### 补充二：`targets.clear()` 会调用每个 `Target` 的析构函数

`Target` 里有 `std::string` 和 `int64_t`，编译器生成的默认析构函数会把它们清理掉。而三个 FFmpeg 裸指针已经在 `closeTarget` 里被释放并置空了，**`clear()` 不会二次释放它们**（裸指针的"析构"什么都不做）。

**这正是"先手动释放再 clear"的原因**——如果 `Target` 用的是智能指针，那就不需要手动释放了。（第 10 章我们会讨论这个改进方向。）

#### 补充三：第 73 行多了一个分号

```cpp
    }
};          // ← 注意这里：}; 而不是 }
```

**这是合法的但多余的**——类定义结束后本来就需要一个 `;`，`release()` 函数体后面的 `}` 再加一个 `;` 就是一个"空声明"（empty declaration），C++11 之后允许。

**无害，但不够干净。** 同学们写的时候去掉它。

## 5.3 第 121~189 行：`Impl::openTarget()` —— 建一条输出管道

这是本课**第一个大函数**（69 行）。我们分段讲。

### 5.3.1 第 121~128 行：前置清理与前置检查

```cpp
121  bool StreamOutput::Impl::openTarget(Target& target)
122  {
123      closeTarget(target, false);
124
125      if (!videoParams) {
126          target.lastError = "没有视频编码参数，无法建流";
127          return false;
128      }
```

**第 123 行：先无条件拆一遍。** 这让 `openTarget` 变成**幂等**的——无论 `target` 之前是什么状态（从未打开过、连着、断了一半），调用它都能得到一个干净的新管道。

**这是 `reconnectDisconnected()` 能直接复用它的前提。**

**第 125~128 行：前置检查。** 没有视频参数就没法建流（我们规定视频是必需的）。

> **老师提问**：为什么这里不检查 `audioParams`？
>
> **因为音频是可选的。** `audioParams == nullptr` 是**合法状态**（表示"只要视频"），只在后面 `if (audioParams)` 时跳过建音频流。

### 5.3.2 第 130~135 行：第一步，创建输出上下文

```cpp
130      AVFormatContext* ctx = nullptr;
131      int ret = avformat_alloc_output_context2(&ctx, nullptr, "flv", target.url.c_str());
132      if (ret < 0 || !ctx) {
133          target.lastError = "创建输出上下文失败: " + avErrorText(ret);
134          return false;
135      }
```

**函数签名**：

```c
int avformat_alloc_output_context2(AVFormatContext** ctx,
                                   const AVOutputFormat* oformat,  // 传 nullptr = 让 FFmpeg 猜
                                   const char* format_name,        // "flv"
                                   const char* filename);          // url
```

它干了什么：

1. 分配一个 `AVFormatContext`；
2. **按 `format_name`（"flv"）找到 `AVOutputFormat`（FLV 的 muxer）**；
3. 如果 `format_name` 是 `nullptr`，就根据 `filename` 的后缀猜；
4. 把找到的 muxer 挂到 `ctx->oformat` 上。

**第 132 行的双重检查（`ret < 0 || !ctx`）是必要的**：

- `ret < 0`：明确失败（比如找不到 "flv" 这个 muxer）；
- `ret == 0` 但 `ctx == nullptr`：理论上不该发生，但**防御性检查是好习惯**（而且这个函数的行为在不同 FFmpeg 版本间有过变化）。

**注意：此时还没有任何 IO 发生**，只是内存里建了个对象。所以失败原因基本只有两个：**FFmpeg 编译时没带 flv muxer**，或者**内存不足**。

### 5.3.3 第 137~145 行：第二步，建视频流 + 拷参数

```cpp
137      AVStream* videoStream = avformat_new_stream(ctx, nullptr);
138      if (!videoStream
139          || avcodec_parameters_copy(videoStream->codecpar, videoParams) < 0) {
140          avformat_free_context(ctx);
141          target.lastError = "创建视频流失败";
142          return false;
143      }
144      videoStream->codecpar->codec_tag = 0;
145      videoStream->time_base = kFlvTimeBase;
```

**第 137 行**：`avformat_new_stream(ctx, nullptr)` 建一条新流。第二个参数是 `const AVCodec*`（可以传编码器让 FFmpeg 帮你初始化部分字段），我们传 `nullptr`，因为**参数全部从参数包拷**。

**第 138~139 行**：用 `||` 把两个检查连起来，任一失败都走同一条回滚路径。这是**简洁的错误处理写法**（对比一下分两个 `if` 会重复两遍回滚代码）。

**第 140 行**：回滚——**第 1 步成功过，所以要释放 `ctx`**。（此时还没开 IO，所以不用 `avio_closep`。）

**第 144 行**：`codec_tag = 0`，让 muxer 自己挑 fourcc（第 1.5 节讲过）。

**第 145 行**：**时间基设为 `{1, 1000}`**——这是第 1.2 节的要点一，FLV 的硬要求。

> **老师插播一个小知识**：`avcodec_parameters_copy` 会拷哪些东西？
>
> ```c
> typedef struct AVCodecParameters {
>     enum AVMediaType codec_type;   // 视频 / 音频 / 字幕
>     enum AVCodecID   codec_id;     // AV_CODEC_ID_H264 / AV_CODEC_ID_AAC
>     uint32_t         codec_tag;    // fourcc
>     uint8_t*         extradata;    // ★ SPS/PPS（会被深拷贝）
>     int              extradata_size;
>     int              format;       // 视频：像素格式；音频：采样格式
>     int64_t          bit_rate;
>     int              width, height;          // 视频
>     int              sample_rate;            // 音频
>     AVChannelLayout  ch_layout;              // 音频声道布局
>     ...
> } AVCodecParameters;
> ```
>
> **关键是 `extradata`：它是深拷贝的**（`avcodec_parameters_copy` 内部会 `av_malloc` 新内存并 `memcpy`）。所以 N 个目标各有各的一份 SPS/PPS，互不影响，释放时也不会double free。

### 5.3.4 第 147~158 行：第三步，建音频流（可选）

```cpp
147      AVStream* audioStream = nullptr;
148      if (audioParams) {
149          audioStream = avformat_new_stream(ctx, nullptr);
150          if (!audioStream
151              || avcodec_parameters_copy(audioStream->codecpar, audioParams) < 0) {
152              avformat_free_context(ctx);
153              target.lastError = "创建音频流失败";
154              return false;
155          }
156          audioStream->codecpar->codec_tag = 0;
157          audioStream->time_base = kFlvTimeBase;
158      }
```

**结构跟视频流完全一样，只是包在 `if (audioParams)` 里。**

**注意第 152 行的回滚**：这里 `avformat_free_context(ctx)` 会**连带释放已经建好的视频流**——因为视频流也是 `ctx` 的一部分。**一次释放，全部干净。** 这就是"所有权归 ctx"的好处。

**建流顺序**：先视频（index 0），后音频（index 1）。第 1.3 节讲过，这个顺序决定了 `stream->index`。

### 5.3.5 第 160~172 行：第四步，打开底层 IO

```cpp
160      if (!(ctx->oformat->flags & AVFMT_NOFILE)) {
161          AVDictionary* options = nullptr;
162          av_dict_set(&options, "rw_timeout", kIoTimeout, 0);
163          ret = avio_open2(&ctx->pb, target.url.c_str(), AVIO_FLAG_WRITE, nullptr,
164                           &options);
165          av_dict_free(&options);
166
167          if (ret < 0) {
168              avformat_free_context(ctx);
169              target.lastError = "连接失败: " + avErrorText(ret);
170              return false;
171          }
172      }
```

**第 160 行**：`AVFMT_NOFILE` 判断（第 2.3 节讲过）。FLV 需要文件/网络 IO，所以这个条件成立，会进去。

**第 161~165 行**：`AVDictionary` 三步曲（第 2.4 节讲过）：

```cpp
AVDictionary* options = nullptr;                        // ① 初始化为 nullptr（必须！）
av_dict_set(&options, "rw_timeout", kIoTimeout, 0);      // ② 设置
avio_open2(..., &options);                               // ③ 使用
av_dict_free(&options);                                  // ④ 释放（必须！）
```

**第 163~164 行**：`avio_open2` 的第四个参数是 `AVIOInterruptCB* int_cb`（中断回调），我们传 `nullptr`（第 2.4 节讲过原因）。

**第 167~171 行**：失败处理。这里**是最常见的失败点**——RTMP 连不上就在这里返回 `-ECONNREFUSED`。

我们的测试用例 3 就是靠这个：

```cpp
{"dead", "rtmp://127.0.0.1:1/live/nothing"}
```

`avio_open2` 会尝试连接 `127.0.0.1:1`，被拒绝，返回负值，走 167 行的分支，`target.lastError` 被设成：

```
连接失败: Connection refused
```

**注意：此时 `ctx->pb` 可能已经被部分初始化了！** 那要不要 `avio_closep`？

**FFmpeg 的契约是：`avio_open2` 失败时，`*s`（也就是 `ctx->pb`）保持未被修改（通常还是 nullptr）。** 所以这里直接 `avformat_free_context(ctx)` 是安全的——`avformat_free_context` 内部会检查 `pb` 并在非空时关闭它。

> **老师点评（一个值得讨论的细节）**：严格来说，代码在这里**依赖了 FFmpeg 的行为约定**（"失败时 pb 不变"）。更防御性的写法是：
> ```cpp
> if (ret < 0) {
>     if (ctx->pb) avio_closep(&ctx->pb);
>     avformat_free_context(ctx);
>     ...
> }
> ```
> 就像第 176~177 行（写 header 失败时）那样。
>
> 两处不一致，说明作者在这里略微掉以轻心了。**实际不会出问题**（`avformat_free_context` 会兜底），但同学们要意识到这个不一致，并养成"失败路径也要完全清理"的习惯。

### 5.3.6 第 174~181 行：第五步，写容器头

```cpp
174      ret = avformat_write_header(ctx, nullptr);
175      if (ret < 0) {
176          if (ctx->pb)
177              avio_closep(&ctx->pb);
178          avformat_free_context(ctx);
179          target.lastError = "写文件头失败: " + avErrorText(ret);
180          return false;
181      }
```

**第 174 行**：`avformat_write_header(ctx, nullptr)`。第二个参数是 `AVDictionary**` 选项（比如 FLV 可以传 `"flvflags"`），我们传 `nullptr`。

它写出：FLV 文件头 + `onMetaData` + **AVC sequence header（SPS/PPS）** + AAC AudioSpecificConfig（第 1.4 节讲过）。

**对 RTMP 来说，这一步会真正把握手后的第一批数据发出去**——包括 `connect`、`createStream`、`publish` 命令和元数据。**如果服务器拒绝你的推流密钥，通常就是在这里失败。**

**第 176~177 行**：这次**显式关了 `pb`**，因为此时 `pb` 肯定是打开状态了。这才是最完整的回滚。

### 5.3.7 第 183~188 行：成功，登记状态

```cpp
183      target.ctx = ctx;
184      target.videoStream = videoStream;
185      target.audioStream = audioStream;
186      target.connected = true;
187      target.lastError.clear();
188      return true;
189  }
```

**注意第 187 行：`target.lastError.clear()`。**

**"成功了就把上一次的错误清空"** ——这一点非常重要！否则会出现这种诡异现象：

```
目标先断了（lastError = "连接失败: Connection refused"）
    ↓
重连成功
    ↓
界面上还显示着 "连接失败"，但实际已经连上了  ← 用户一脸懵
```

**清状态是"成功路径"的一部分。** 同学们记住：**任何一个状态字段，都要想清楚"什么时候设、什么时候清"**，两边都别漏。

### 5.3.8 `openTarget` 全景小结

```
openTarget(t)
  │
  ├─ closeTarget(t, false)          ← 幂等：先拆干净
  ├─ 检查 videoParams
  │
  ├─ ① avformat_alloc_output_context2   失败 → return false（无需回滚）
  ├─ ② avformat_new_stream (video)       失败 → free_context → return false
  │   avcodec_parameters_copy
  │   codec_tag = 0; time_base = 1/1000
  ├─ ③ avformat_new_stream (audio)?      失败 → free_context → return false
  │   （可选，audioParams 为空则跳过）
  ├─ ④ avio_open2 (rw_timeout=5s)        失败 → free_context → return false
  ├─ ⑤ avformat_write_header             失败 → avio_closep + free_context → return false
  │
  └─ 登记：ctx / videoStream / audioStream / connected=true / 清 lastError
      return true
```

**每一步的回滚范围，正好等于"这一步之前成功过的所有步骤"的逆序。** 这就是标准的 C 风格资源管理。

## 5.4 第 191~207 行：`Impl::closeTarget()` —— 拆一条输出管道

```cpp
191  void StreamOutput::Impl::closeTarget(Target& target, bool writeTrailer)
192  {
193      if (target.ctx) {
194          if (writeTrailer && target.connected)
195              av_write_trailer(target.ctx);
196
197          if (target.ctx->pb)
198              avio_closep(&target.ctx->pb);
199
200          avformat_free_context(target.ctx);
201          target.ctx = nullptr;
202      }
203
204      target.videoStream = nullptr;
205      target.audioStream = nullptr;
206      target.connected = false;
207  }
```

### 第 193 行：`if (target.ctx)` —— 幂等保护

整个函数被包在 `if (target.ctx)` 里。**如果 `ctx` 已经是 `nullptr`，整个函数什么都不做**（除了 204~206 行的置空，那也是幂等的）。

**所以 `closeTarget` 可以随便调用多少次。** 这是它能同时服务于"首次清理"、"正常关闭"、"重连前清理"三个场景的原因。

### 第 194~195 行：写 trailer 的两个条件

```cpp
if (writeTrailer && target.connected)
    av_write_trailer(target.ctx);
```

**必须两个条件都满足**：

| 条件 | 为什么 |
|---|---|
| `writeTrailer == true` | 调用方要求"优雅关闭"。重连前的清理传 `false`（连接已废，别浪费时间） |
| `target.connected == true` | **只有"曾经成功连上并写过 header"的，才能写 trailer** |

**第二个条件是关键**：如果对一条**从来没连上过**的 `ctx` 调 `av_write_trailer`，FFmpeg 会因为"没写过 header"而断言失败或者返回错误。

那什么时候会出现 `ctx != nullptr && connected == false`？

**在 `writePacket` 的失败分支里！** 看 5.5 节的代码：

```cpp
target.connected = false;      // 先标记断开
...
avformat_free_context(target.ctx);   // 然后释放 ctx
target.ctx = nullptr;
```

它已经把 `ctx` 置空了。所以正常路径下不会出现这个组合。

**但 `openTarget` 里有个中间状态**：第 174 行 `write_header` 失败时，`ctx` 还在、`connected` 还是 `false`——不过那条路径直接 `return false` 了，不会走到 `closeTarget`。

**所以这个 `&& target.connected` 是一道"以防万一"的保险。** 同学们，**在资源清理代码里多加一层保险，永远不亏**——它不会让代码变慢，但能挡住未来某个改动引入的崩溃。

### 第 197~198 行：`if (target.ctx->pb)` + `avio_closep`

```cpp
if (target.ctx->pb)
    avio_closep(&target.ctx->pb);
```

**先判空再关。** 为什么要判空？因为 `openTarget` 的第 160 行有 `AVFMT_NOFILE` 分支——**如果 muxer 不需要 IO，`pb` 会一直是 `nullptr`**，对它调 `avio_closep` 虽然不会崩（`avio_closep` 内部会检查），但判空更清晰。

**`avio_closep`（带 p）vs `avio_close`（不带 p）**：

```c
void avio_close(AVIOContext* s);       // 只关闭，指针值不变 → 变成悬垂指针！
void avio_closep(AVIOContext** s);     // 关闭 + 把 *s 置 nullptr ← 用它！
```

**永远用带 `p` 的版本。** 这是 FFmpeg 的命名约定：`xxx_freep(&p)` / `xxx_closep(&p)` 都会置空，`xxx_free(p)` / `xxx_close(p)` 不会。

| 不置空的 | 置空的（推荐） |
|---|---|
| `avio_close(pb)` | **`avio_closep(&pb)`** |
| `avformat_close_input(fmt)` | `avformat_closep(&fmt)`（少用） |
| `av_packet_unref(pkt)` | `av_packet_free(&pkt)` |
| `av_frame_unref(f)` | `av_frame_free(&f)` |
| `avcodec_free_context(&ctx)` | （本身就带 &，会置空） |

### 第 200~201 行：`avformat_free_context`

```cpp
avformat_free_context(target.ctx);
target.ctx = nullptr;
```

`avformat_free_context` 会：

1. 关闭并释放 `ctx->pb`（**如果还没关的话**——所以第 197 行的关闭其实是"显式做一遍"，更清晰）；
2. 释放所有 `AVStream`（包括它们的 `codecpar` 和 `extradata`）；
3. 释放交织缓冲区里**还没写出去的包**（所以尾部数据会丢——正常，因为我们刚写了 trailer）；
4. 释放 `ctx` 自己。

**第 201 行的手动置空是必须的**——`avformat_free_context` 的参数是 `AVFormatContext*`（不是 `**`），它没法帮你置空。

### 第 204~206 行：清理"借来的指针"和状态

```cpp
target.videoStream = nullptr;
target.audioStream = nullptr;
target.connected = false;
```

**这三行在 `if (target.ctx)` 外面**——即使 `ctx` 已经是 `nullptr`（比如目标从来没连上过），也要执行。

**为什么？** 因为 `videoStream` / `audioStream` 可能因为某种原因（比如 `openTarget` 中途失败）残留了非空值。**统一在这里清干净，保证 `closeTarget` 之后 `Target` 一定处于"全新"状态。**

> **老师点评**：这三行是本课最不起眼、但最能体现功力的地方。
>
> 很多同学写完 `if (target.ctx) { ...释放... }` 就完事了，**忘了外面还有两个悬垂指针**。而这两个指针如果残留，下一次 `writePacket` 时 `if (!stream) continue;` 就拦不住了，会往一个已经释放的 `AVStream` 上写 `stream->index`——**随机崩溃**。
>
> **清理函数的最高标准：执行完之后，对象必须回到"刚构造出来"的状态。**

## 5.5 第 209~278 行：`Impl::writePacket()` —— ★ 本课的核心

终于到了整个模块的心脏。70 行，我们分成五段。

```cpp
209  bool StreamOutput::Impl::writePacket(const AVPacket& packet, AVRational timeBase,
210                                       bool isVideo)
211  {
```

**三个参数**：

| 参数 | 含义 |
|---|---|
| `const AVPacket& packet` | 上游给的一包数据（常量引用，我们不改它） |
| `AVRational timeBase` | 这包数据当前所处的时间基（编码器时间基） |
| `bool isVideo` | 是视频还是音频（决定走哪条流、要不要做音频偏移） |

### 5.5.1 第 212~223 行：计算音频偏移（AAC priming 归零）

```cpp
212      int64_t ptsOffset = 0;
213      if (!isVideo) {
214          if (!audioOffsetReady) {
215              audioOffset = packet.pts == AV_NOPTS_VALUE ? 0 : packet.pts;
216              audioOffsetReady = true;
217              if (audioOffset != 0 && log) {
218                  log("音频起点偏移 " + std::to_string(audioOffset)
219                      + "（AAC 编码延迟），已归零");
220              }
221          }
222          ptsOffset = audioOffset;
223      }
```

第 3.7 节讲过原理，这里只补充代码层面的三点：

**第 215 行**：`packet.pts == AV_NOPTS_VALUE ? 0 : packet.pts` —— 如果第一包没有时间戳，偏移就当 0（不做平移），**避免 `INT64_MIN` 参与运算导致下溢**。

**第 217 行**：`if (audioOffset != 0 && log)` —— **只在"确实有偏移"时才打日志**。如果音频编码器没有 priming（比如换成 MP3 或者 Opus），`audioOffset` 是 0，就不打这条日志，避免噪音。

**第 222 行**：`ptsOffset = audioOffset;` —— 之后的每一包都用这个已算好的值。

> **老师提问**：为什么 `ptsOffset` 是**局部变量**，而不是直接用成员 `audioOffset`？
>
> **答案**：两个原因。
> ① **视频路径必须是 0**——用局部变量，视频时它天然是 0，不用写 `isVideo ? 0 : audioOffset`；
> ② **语义清晰**——"这一包要减多少"，是个局部概念，不该让循环体去读成员。

### 5.5.2 第 225~233 行：循环前的准备

```cpp
225      bool anyOk = false;
226
227      for (Target& target : targets) {
228          if (!target.connected || !target.ctx)
229              continue;
230
231          AVStream* stream = isVideo ? target.videoStream : target.audioStream;
232          if (!stream)
233              continue;
```

**第 225 行**：`anyOk` —— "至少有一路成功了吗"（第 3.6 节讲过）。

**第 227 行**：`for (Target& target : targets)` —— 注意是**非常量引用** `Target&`，因为循环体里要修改 `target.connected`、`target.packetsSent` 等。

**第 228~229 行：第一道过滤**：

```cpp
if (!target.connected || !target.ctx)
    continue;
```

两个条件都要检查，虽然它们理论上同步变化。**冗余检查在资源管理的代码里是值得的**——万一将来有个路径只改了其中一个，另一道检查能兜住。

**第 231~233 行：第二道过滤**：

```cpp
AVStream* stream = isVideo ? target.videoStream : target.audioStream;
if (!stream)
    continue;
```

**这是"多目标可以有不同配置"的关键支持**：如果某个目标只有视频流（`open()` 时音频参数传了 `nullptr`），你给它写音频包时 `stream` 是 `nullptr`，**安静跳过，不报错**。

### 5.5.3 第 235~250 行：克隆 + 改字段

```cpp
235          AVPacket* copy = av_packet_clone(&packet);
236          if (!copy) {
237              target.lastError = "av_packet_clone 失败（内存不足？）";
238              continue;
239          }
240
241          if (ptsOffset != 0) {
242              if (copy->pts != AV_NOPTS_VALUE)
243                  copy->pts -= ptsOffset;
244              if (copy->dts != AV_NOPTS_VALUE)
245                  copy->dts -= ptsOffset;
246          }
247
248          copy->stream_index = stream->index;
249
250          av_packet_rescale_ts(copy, timeBase, stream->time_base);
```

**第 235 行**：`av_packet_clone` —— 第 3.2 节讲过，零拷贝的引用计数克隆。

**第 236~239 行**：克隆失败 → 只记错误，**保持连接**（第 3.4 节讲过，这是"暂时性错误"）。

**第 241~246 行**：音频偏移。两个 `if` 分别保护 pts 和 dts（第 3.7 节细节三讲过 `AV_NOPTS_VALUE` 的下溢危险）。

**第 248 行**：`stream_index` —— 从**这个目标自己的** `stream` 上读（第 1.3 节讲过，绝不假设是 0 或 1）。

**第 250 行**：`av_packet_rescale_ts` —— 时间基换算（第 1.2 节讲过）。

**顺序很重要**：

```
① clone            → 拿到独立副本
② 减音频偏移        → 在"源时间基"下做减法（因为 offset 也是源时间基下的值）★
③ 设 stream_index  → 告诉 FFmpeg 这包给谁
④ rescale_ts       → 换算到目标时间基
⑤ 写出
```

**第 ② 步和第 ④ 步的顺序不能颠倒！** 因为 `audioOffset` 是从**编码器时间基**（比如 `{1,48000}`）下的 pts 测出来的，所以必须在**换算之前**减掉。

如果先换算（48000 → 1000）再减 1024，那就减掉了 1024 毫秒——**整整一秒多的偏移！**

> **老师点评**：这类"**单位一致性**"的 bug 特别难查，因为程序不崩、不报错，只是"音画差了一点点"。
>
> 同学们做时间相关的运算时，**每一步都在脑子里问一句："这个数的单位是什么？跟旁边那个数一样吗？"**

### 5.5.4 第 252~270 行：写出 与 失败隔离

```cpp
252          const int ret = av_interleaved_write_frame(target.ctx, copy);
253          av_packet_free(&copy);
254
255          if (ret < 0) {
256              target.connected = false;
257              target.lastError = "写入失败: " + avErrorText(ret);
258              if (log)
259                  log("[" + target.name + "] 已断开: " + target.lastError);
260
261              if (target.ctx) {
262                  if (target.ctx->pb)
263                      avio_closep(&target.ctx->pb);
264                  av_format_free_context(target.ctx);   // ← 原文是 avformat_free_context
265                  target.ctx = nullptr;
266              }
267              target.videoStream = nullptr;
268              target.audioStream = nullptr;
269              continue;
270          }
```

**第 252~253 行**：写出 + 无条件释放（第 3.3 节讲过为什么无条件释放是健壮的）。

**第 255~270 行：失败隔离的完整流程**，六个动作：

| 行 | 动作 | 目的 |
|---|---|---|
| 256 | `connected = false` | **立刻摘牌**：后续所有包都会在第 228 行被过滤掉 |
| 257 | `lastError = "..."` | 记录原因，供界面展示 |
| 258~259 | `log(...)` | 通知上层（**带 `if (log)` 判空！**） |
| 261~266 | 关 IO + 释放 ctx + 置空 | **释放资源**，避免泄漏 |
| 267~268 | 清两个 stream 指针 | **防悬垂** |
| 269 | `continue` | **★ 继续服务下一个目标** |

**注意第 261~266 行跟 `closeTarget()` 的代码是重复的**，但**不能直接调 `closeTarget(target, false)`**——因为……

**等等，其实是可以的！** 让我再看一下：

```cpp
// 这里的代码
if (target.ctx) {
    if (target.ctx->pb) avio_closep(&target.ctx->pb);
    avformat_free_context(target.ctx);
    target.ctx = nullptr;
}
target.videoStream = nullptr;
target.audioStream = nullptr;
target.connected = false;    // ← closeTarget 里也有
continue;
```

```cpp
// closeTarget(target, false) 的代码
if (target.ctx) {
    if (false && ...) ;      // 不写 trailer
    if (target.ctx->pb) avio_closep(&target.ctx->pb);
    avformat_free_context(target.ctx);
    target.ctx = nullptr;
}
target.videoStream = nullptr;
target.audioStream = nullptr;
target.connected = false;
```

**完全等价！** 所以这里本可以写成：

```cpp
if (ret < 0) {
    target.lastError = "写入失败: " + avErrorText(ret);
    if (log) log("[" + target.name + "] 已断开: " + target.lastError);
    closeTarget(target, false);      // ★ 一行搞定，不会漏
    continue;
}
```

> **老师点评（本讲第一个值得改进的地方）**：
>
> 作者在这里**复制了 11 行清理代码**，而不是复用 `closeTarget(target, false)`。
>
> 后果是什么？**如果将来清理逻辑要加一步（比如"记录断开时间"），你得记得改两个地方。** 这就是典型的 **DRY（Don't Repeat Yourself）违反**，是维护期的 bug 温床。
>
> 同学们引以为戒：**发现了重复的清理逻辑，第一反应应该是"能不能抽成函数"，而不是"再抄一遍"。**
>
> 不过话说回来，作者这么写也**不算错**——它是正确的，只是不够优雅。在"能跑通"和"优雅"之间，作者选了前者，这在 walking skeleton 阶段是可以理解的。

**第 264 行的小提醒**：上面代码里我写成了 `av_format_free_context`，**这是笔误**，原文是 `avformat_free_context`。同学们看 diff 时注意。

### 5.5.5 第 272~277 行：成功统计与返回

```cpp
272          target.packetsSent += 1;
273          target.bytesSent += packet.size;
274          anyOk = true;
275      }
276
277      return anyOk;
278  }
```

**第 272~273 行**：统计。

**注意第 273 行用的是 `packet.size` 而不是 `copy->size`** —— 因为 `copy` 已经在 253 行被 free 了！用原包的 size 是对的（两者相等，因为 clone 共享数据）。

**这也说明了一个道理**：**如果某个值你在释放之后还需要，就提前存好，或者从别处拿。** 作者选择从原包拿，简洁。

> **老师补一句**：`bytesSent` 统计的是**压缩数据的字节数**，不是"实际写到网络的字节数"。
>
> 真正的网络字节数还要算上 FLV Tag 头（11 字节/包）、RTMP chunk 头、TCP/IP 头。对于小包（比如音频包只有几百字节），这些开销能占到 10%~30%。
>
> 所以 **`bytesSent` 适合用来做"相对比较"（A 路和 B 路是不是一样多），不适合用来精确计算带宽**。第 10 章我们会看到更准确的做法（从 `AVIOContext` 上读 `bytes_written`）。

## 5.6 第 280~358 行：`StreamOutput::open()` —— 建 N 条管道

```cpp
280  StreamOutput::StreamOutput() : impl_(std::make_unique<Impl>())
281  {
282  }
283
284  StreamOutput::~StreamOutput()
285  {
286      close();
287  }
288
289  bool StreamOutput::open(const std::vector<StreamTarget>& targets,
290                          const AVCodecParameters* videoParams,
291                          const AVCodecParameters* audioParams, LogCallback log)
292  {
293      close();
294
295      impl_->log = std::move(log);
296      impl_->lastError.clear();
297
298      const auto say = [this](const std::string& message) {
299          if (impl_->log)
300              impl_->log(message);
301      };
302
303      if (targets.empty()) {
304          impl_->lastError = "没有配置任何推流目标";
305          return false;
306      }
307
308      if (!videoParams) {
309          impl_->lastError = "没有视频编码参数";
310          return false;
311      }
312
313      impl_->videoParams = avcodec_parameters_alloc();
314      if (!impl_->videoParams
315          || avcodec_parameters_copy(impl_->videoParams, videoParams) < 0) {
316          impl_->lastError = "保存视频参数失败";
317          close();
318          return false;
319      }
320
321      if (audioParams) {
322          impl_->audioParams = avcodec_parameters_alloc();
323          if (!impl_->audioParams
324              || avcodec_parameters_copy(impl_->audioParams, audioParams) < 0) {
325              impl_->lastError = "保存音频参数失败";
326              close();
327              return false;
328          }
329      }
330
331      impl_->targets.resize(targets.size());
332      int connected = 0;
333
334      for (size_t i = 0; i < targets.size(); ++i) {
335          Impl::Target& target = impl_->targets[i];
336          target.name = targets[i].platform.empty()
337                            ? ("target" + std::to_string(i))
338                            : targets[i].platform;
339          target.url = targets[i].url;
340
341          if (impl_->openTarget(target)) {
342              ++connected;
343              say("[" + target.name + "] 已连接 " + target.url);
344          } else {
345              say("[" + target.name + "] " + target.lastError);
346          }
347      }
348
349      if (connected == 0) {
350          impl_->lastError = "所有推流目标都没连上";
351          return false;
352      }
353
354      say("推流输出就绪: " + std::to_string(connected) + "/"
355          + std::to_string(targets.size()) + " 路可用");
356
357      return true;
358  }
```

我们逐段看。

### 5.6.1 第 280~287 行：构造与析构

```cpp
StreamOutput::StreamOutput() : impl_(std::make_unique<Impl>())
{
}

StreamOutput::~StreamOutput()
{
    close();
}
```

**构造函数**：用 **成员初始化列表** 初始化 `impl_`（第 4.2 节讲过 PImpl）。

**析构函数**：只调 `close()`。

**这一行是整个 PImpl 设计的价值所在**——**析构函数定义在 .cpp 里，此时 `Impl` 是完整类型，`unique_ptr` 能正确析构它**（第 4.2 节末尾讲过，如果析构函数写在头文件里会编译失败）。

**而且析构自动 `close()`，保证了 RAII**：

```cpp
{
    StreamOutput output;
    output.open(...);
    output.writeVideo(...);
    // ... 中间就算抛异常、提前 return
}   // ← 离开作用域，析构函数自动 close()，所有 FFmpeg 资源被释放
```

**这就是 C++ 相对于 C 的最大优势之一**——**资源的生命周期绑在对象的生命周期上，不需要手动记得释放。**

对比第 2 课那个 demo：出错就 `exit(1)`，所有 FFmpeg 对象全泄漏。**今天我们终于有了正解。**

### 5.6.2 第 293~301 行：初始化

```cpp
close();                            // ① 先清空（幂等）

impl_->log = std::move(log);        // ② move 进日志回调
impl_->lastError.clear();           // ③ 清空上次的错误

const auto say = [this](const std::string& message) {   // ④ 定义一个"说话"的小工具
    if (impl_->log)
        impl_->log(message);
};
```

**第 293 行**：`close()` 打头。让 `open()` 可以**重复调用**（第二次 `open()` 会先关掉第一次的资源）。

**第 295 行**：`std::move(log)` —— 因为 `log` 是按值传进来的参数（第 4.2 节讲过）。

**第 298~301 行：`say` 这个小 lambda 是个好设计**。

它把"判空 + 调用"这两步封装起来，后面所有打日志的地方就只用写：

```cpp
say("[" + target.name + "] 已连接 " + target.url);
```

而不用每次都写：

```cpp
if (impl_->log) impl_->log("...");   // 啰嗦而且容易忘判空
```

**这个模式在本项目的三个模块里都出现了**（`VideoEncoder::open`、`AudioEncoder::open`、`StreamOutput::open`），而且都叫 `say`。**一致的代码风格让读代码的人能快速适应。**

> **老师点评**：同学们注意 `say` 的捕获列表是 `[this]`——它捕获 `this` 指针，然后访问 `impl_->log`。
>
> 能不能捕获 `[&]`？可以，但 `[this]` 更精确（只捕获 `this`）。**精确的捕获列表是好的 lambda 习惯**——它告诉读者"这个 lambda 只依赖对象本身"。

### 5.6.3 第 303~311 行：前置检查

```cpp
if (targets.empty()) {
    impl_->lastError = "没有配置任何推流目标";
    return false;
}

if (!videoParams) {
    impl_->lastError = "没有视频编码参数";
    return false;
}
```

**这两条是"调用方用错了"，不是运行时故障**（第 3.6 节末尾讲过判断标准）。

**注意这里没有调 `close()`** ——因为第 293 行已经调过了，而这两条检查在**分配任何资源之前**，所以无需清理。

> **老师提问**：为什么检查顺序是"先 targets 后 videoParams"？
>
> **答案**：**从最可能出错、最容易理解的原因开始检查。** 用户最容易犯的错是"忘了配推流地址"，其次才是"编码器没开"。**错误信息的顺序应该跟"用户最可能需要的帮助"一致。**

### 5.6.4 第 313~329 行：保存参数模板

```cpp
impl_->videoParams = avcodec_parameters_alloc();
if (!impl_->videoParams
    || avcodec_parameters_copy(impl_->videoParams, videoParams) < 0) {
    impl_->lastError = "保存视频参数失败";
    close();                    // ★ 注意这里
    return false;
}

if (audioParams) {
    impl_->audioParams = avcodec_parameters_alloc();
    if (!impl_->audioParams
        || avcodec_parameters_copy(impl_->audioParams, audioParams) < 0) {
        impl_->lastError = "保存音频参数失败";
        close();                // ★ 和这里
        return false;
    }
}
```

这两段是"深拷贝快照"（第 2.5 节点二讲过）。

**关键在第 317、326 行的 `close()`**：

- 视频参数分配成功、但拷贝失败 → `videoParams` 是个有效的空对象 → 需要 `close()` 释放它；
- 音频参数分配成功、但拷贝失败 → `videoParams` 和 `audioParams` 都要释放 → `close()` 一次搞定。

**`close()` 在这里的作用是"统一的失败清理出口"**，比手写 `avcodec_parameters_free(&videoParams)` 更不容易漏。

> **老师点评**：这里作者**用了 `close()` 而不是复制清理代码**——跟 5.5.4 节那个"复制 11 行"形成了鲜明对比。
>
> 同一个文件里，两种风格。说明作者是在写的过程中逐渐意识到这一点的。**同学们，你们现在就站在作者的肩膀上——直接养成"清理逻辑只写一份"的习惯。**

### 5.6.5 第 331~347 行：循环建管道

```cpp
impl_->targets.resize(targets.size());
int connected = 0;

for (size_t i = 0; i < targets.size(); ++i) {
    Impl::Target& target = impl_->targets[i];
    target.name = targets[i].platform.empty()
                      ? ("target" + std::to_string(i))
                      : targets[i].platform;
    target.url = targets[i].url;

    if (impl_->openTarget(target)) {
        ++connected;
        say("[" + target.name + "] 已连接 " + target.url);
    } else {
        say("[" + target.name + "] " + target.lastError);
    }
}
```

**第 331 行**：`resize(targets.size())` —— 一次性分配好 N 个默认构造的 `Target`。

**为什么用 `resize` 而不是 `reserve` + `push_back`？** 因为我们需要**按下标访问**（`impl_->targets[i]`），而且**每个 target 的状态要独立保存**（不能每次重建）。

**第 336~338 行：名字的兜底逻辑**：

```cpp
target.name = targets[i].platform.empty()
                  ? ("target" + std::to_string(i))     // 没名字 → "target0"/"target1"
                  : targets[i].platform;               // 有名字 → 用平台的名字
```

**这是一个很贴心的设计**：配置里可能只写了 URL 没写平台名（比如本地录像），那就自动生成 `"target0"` 这样的名字，**保证界面上永远有东西可显示**。

> **老师点评**：这类"**用户输入缺失时的兜底**"最能体现工程细致度。
>
> 反面教材是：`target.name = targets[i].platform;` 然后界面上出现一堆空白行，用户根本不知道哪行是哪个目标。
>
> **规则：任何要展示给用户的字段，都要有非空兜底。**

**第 341~346 行**：逐个建管道，**成功失败都记一笔，但都不中断循环**。

这就是"部分成功"范式的体现（第 3.6 节讲过）。

### 5.6.6 第 349~357 行：判定整体成败

```cpp
if (connected == 0) {
    impl_->lastError = "所有推流目标都没连上";
    return false;
}

say("推流输出就绪: " + std::to_string(connected) + "/"
    + std::to_string(targets.size()) + " 路可用");

return true;
```

**"全军覆没才算失败"**（第 3.6 节讲过）。

**第 354~355 行的日志**是本课最有"产品感"的一行：

```
[斗鱼] 已连接 rtmp://send.douyu.com/live/xxxxx
[B站] 连接失败: Connection refused
[录像] 已连接 D:/Record/2026-10-20.flv
推流输出就绪: 2/3 路可用
```

**一眼看清全局状态。** 这就是好的日志——**不仅告诉你"成功了"，还告诉你"成功的程度"。**

> **老师补充一个小瑕疵**：第 350 行 `return false` 之前**没有调 `close()`**。
>
> 后果：`impl_->targets` 里还留着 N 个 `Target`（虽然它们的 `ctx` 都是 nullptr，`openTarget` 失败时已经清理过了），`videoParams` / `audioParams` 还占着内存。
>
> 不会泄漏（因为析构函数会 `close()`），但**状态不干净**：`open()` 失败后，`connectedCount()` 返回 0，`targets()` 却返回 N 个条目。
>
> **更严格的做法**是：
> ```cpp
> if (connected == 0) {
>     impl_->lastError = "所有推流目标都没连上";
>     close();          // ← 加上这句，回到"全新"状态
>     return false;
> }
> ```
> 这样语义就统一了：**`open()` 失败 = 对象回到未打开状态。**

## 5.7 第 360~433 行：其余公开接口

### 5.7.1 第 360~373 行：`close()` / `writeVideo()` / `writeAudio()`

```cpp
360  void StreamOutput::close()
361  {
362      impl_->release();
363  }
364
365  bool StreamOutput::writeVideo(const AVPacket& packet, AVRational encoderTimeBase)
366  {
367      return impl_->writePacket(packet, encoderTimeBase, true);
368  }
369
370  bool StreamOutput::writeAudio(const AVPacket& packet, AVRational encoderTimeBase)
371  {
372      return impl_->writePacket(packet, encoderTimeBase, false);
373  }
```

**三个"薄包装"函数**。它们只做一件事：**给 `Impl` 的私有方法套一层公开的外壳，并固定某个参数。**

```
writeVideo(pkt, tb)  ──→  writePacket(pkt, tb, /*isVideo=*/true)
writeAudio(pkt, tb)  ──→  writePacket(pkt, tb, /*isVideo=*/false)
```

**为什么不直接把 `writePacket` 做成公开的**（让用户传 `isVideo`）？

因为 **`bool isVideo` 是个"布尔参数陷阱"（boolean trap）**：

```cpp
output.write(packet, tb, true);    // ← 调用方看到这行，知道 true 是什么意思吗？
output.writeVideo(packet, tb);     // ← 一目了然
```

**用两个语义化的函数名，替代一个布尔参数**，可读性提升巨大。这是《Effective C++》《Clean Code》都推荐的做法。

> **老师补充**：更彻底的做法是用枚举：
> ```cpp
> enum class MediaKind { Video, Audio };
> bool write(const AVPacket& packet, AVRational tb, MediaKind kind);
> ```
> 但既然只有两种，两个具名函数最简单。

### 5.7.2 第 375~381 行：`flush()`

```cpp
375  void StreamOutput::flush()
376  {
377      for (Impl::Target& target : impl_->targets) {
378          if (target.connected && target.ctx)
379              av_interleaved_write_frame(target.ctx, nullptr);
380      }
381  }
```

第 3.5 节讲过：传 `nullptr` = "冲刷交织缓冲区"。

**这里的 `nullptr` 跟 `avcodec_send_frame(ctx, nullptr)`（flush 编码器）是同一个套路**——FFmpeg 里"传 nullptr 表示 flush"是一个贯穿全局的约定：

| 函数 | 传 nullptr 的含义 |
|---|---|
| `avcodec_send_frame(ctx, nullptr)` | 冲刷编码器（把缓存的 B 帧吐出来） |
| `av_interleaved_write_frame(fmt, nullptr)` | 冲刷封装器的交织缓冲 |
| `avcodec_receive_packet` 返回 `AVERROR_EOF` | 编码器已排空 |

### 5.7.3 第 383~400 行：`targets()`

```cpp
383  std::vector<OutputTargetInfo> StreamOutput::targets() const
384  {
385      std::vector<OutputTargetInfo> result;
386      result.reserve(impl_->targets.size());
387
388      for (const Impl::Target& target : impl_->targets) {
389          OutputTargetInfo info;
390          info.name = target.name;
391          info.url = target.url;
392          info.connected = target.connected;
393          info.packetsSent = target.packetsSent;
394          info.bytesSent = target.bytesSent;
395          info.lastError = target.lastError;
396          result.push_back(std::move(info));
397      }
398
399      return result;
400  }
```

**第 386 行**：`reserve()` —— 预分配容量，避免 `push_back` 时反复扩容（第 3 课讲过 `BoundedQueue` 时提过内存分配的代价）。

**第 388 行**：`const Impl::Target&` —— 只读遍历，**因为 `targets()` 是 `const` 成员函数**。

> **老师提问**：为什么 `targets()` 是 `const` 的，但里面能访问 `impl_->targets`？
>
> **答案**：因为 `impl_` 是 `std::unique_ptr<Impl>`，`const` 成员函数里 `impl_` 的类型是 `const std::unique_ptr<Impl>&`。
>
> 对 `const unique_ptr` 解引用，得到的是 `Impl&`（**不是 `const Impl&`**）——**`unique_ptr` 的 `operator*` 是 const 成员函数，返回非 const 引用**（因为指针本身的 const ≠ 指向对象的 const）。
>
> 所以**严格来说，`const` 成员函数里是能修改 `impl_` 指向的内容的**。这是 C++ 的一个"漏洞"（很多人批评过）。

**第 396 行**：`result.push_back(std::move(info));`

**这里 `std::move` 是多余的**——`info` 是个即将销毁的局部对象，`push_back` 的右值重载会被自动选择（编译器会做 NRVO/implicit move）。

> 严格来说，C++11 之后，返回局部变量时会自动尝试 move，所以 `push_back(info)` 也几乎一样高效（会拷贝一次）。**加上 `std::move` 让它变成移动，省掉一次拷贝。**
>
> **无害，但也可以不写。** 同学们知道这里 `move` 的语义就行。

### 5.7.4 第 402~410 行：`connectedCount()`

```cpp
402  int StreamOutput::connectedCount() const
403  {
404      int count = 0;
405      for (const Impl::Target& target : impl_->targets) {
406          if (target.connected)
407              ++count;
408      }
409      return count;
410  }
```

简单计数。

> **老师补充**：学了 `<algorithm>` 的同学可能想写成：
> ```cpp
> return std::count_if(impl_->targets.begin(), impl_->targets.end(),
>                      [](const Impl::Target& t) { return t.connected; });
> ```
> 但要注意它返回 `ptrdiff_t`，要转 `int`。**两种写法都行，手写循环在这里更简单直白。**

### 5.7.5 第 412~428 行：`reconnectDisconnected()`

```cpp
412  int StreamOutput::reconnectDisconnected()
413  {
414      int reconnected = 0;
415
416      for (Impl::Target& target : impl_->targets) {
417          if (target.connected)
418              continue;
419
420          if (impl_->openTarget(target)) {
421              ++reconnected;
422              if (impl_->log)
423                  impl_->log("[" + target.name + "] 重连成功");
424          }
425      }
426
427      return reconnected;
428  }
```

**逻辑非常简单**：遍历，跳过还连着的，对断掉的调 `openTarget`。

**它能工作，靠的是前面三个设计**：

| 设计 | 支撑了什么 |
|---|---|
| `openTarget` 开头 `closeTarget(target, false)` | 重连前自动清理残留状态，**幂等** |
| `Target` 里存了 `url` 的副本 | 重连时不需要外部再提供 URL |
| `Impl` 里存了 `videoParams` / `audioParams` 模板 | 重连时能重建流，即使上游编码器已经销毁 |

**第 422~423 行**：注意这里用的是 `impl_->log` 而**不是** `say`——因为 `say` 是 `open()` 里的局部 lambda，这里访问不到。所以只能手写 `if (impl_->log)`。

> **老师点评（第二个值得改进的地方）**：这里又出现了 `if (impl_->log) impl_->log(...)` 的重复。
>
> 而且**失败时没有任何日志**！对比 `open()` 里失败会 `say(... + target.lastError)`，这里失败是**静默的**。
>
> 更好的写法是把 `say` 提成 `Impl` 的成员函数：
> ```cpp
> struct StreamOutput::Impl {
>     ...
>     void say(const std::string& message) const {
>         if (log) log(message);
>     }
> };
> ```
> 然后三个地方都用它。**这是本讲留给同学们的"重构练习"之一（第 10 章思考题）。**

### 5.7.6 第 430~433 行：`lastError()`

```cpp
430  const std::string& StreamOutput::lastError() const
431  {
432      return impl_->lastError;
433  }
```

一行返回。第 4.2 节讲过返回 `const&` 的生命周期注意事项。

## 5.8 `StreamOutput.cpp` 全景回顾

同学们，389 行讲完了。我们把它压成一张图：

```
        ┌──────────────────── open(targets, vParams, aParams, log) ────────────────────┐
        │  close()  →  检查入参  →  深拷贝参数模板  →  resize targets  →  for each:    │
        │                                                              openTarget(t)   │
        │                                                                  ↓           │
        │                                                          成功? ++connected   │
        │                                                                              │
        │  connected == 0 ? return false : return true                                 │
        └──────────────────────────────────────────────────────────────────────────────┘
                                          ↓ 之后，每一包
        ┌──────────── writeVideo / writeAudio ────────→ writePacket(pkt, tb, isVideo) ──┐
        │  ① 算音频偏移（仅首包、仅音频）                                               │
        │  ② for each target:                                                          │
        │       跳过：!connected || !ctx || !stream                                     │
        │       clone → 减偏移 → 设 stream_index → rescale_ts                          │
        │       av_interleaved_write_frame                                             │
        │       av_packet_free（无条件）                                                │
        │       失败? → 标记断开 + 记错 + 日志 + 关 IO + free ctx + 清指针 + continue    │
        │       成功? → packetsSent++ / bytesSent += size / anyOk = true                │
        │  ③ return anyOk                                                              │
        └──────────────────────────────────────────────────────────────────────────────┘
                                          ↓ 结束
        ┌──────── flush() ───→ close() / 析构 ───→ release() ──────────────────────────┐
        │  flush: 对每个目标 av_interleaved_write_frame(ctx, nullptr)                  │
        │  release: for each → closeTarget(t, true)  [写 trailer + 关 IO + free ctx]    │
        │           targets.clear()                                                    │
        │           avcodec_parameters_free(videoParams / audioParams)                 │
        │           audioOffsetReady = false; audioOffset = 0                          │
        └──────────────────────────────────────────────────────────────────────────────┘
```

**三条数据流的走向**：

```
packet  ──clone──→  copy ──→ target[0].ctx ──→ RTMP  ──→ 斗鱼
                    copy ──→ target[1].ctx ──→ RTMP  ──→ B站
                    copy ──→ target[2].ctx ──→ 文件  ──→ 本地录像
                    （三份 copy 共享同一块数据内存，靠 refcount 记账）

统计  ──────────→  packetsSent / bytesSent  ──→  targets()  ──→ 界面

错误  ──────────→  每路 lastError + 全局 lastError  ──→  targets() / lastError()  ──→ 界面
```

---

# 第 6 章 逐行教学（三）：三处 CMakeLists 改动（24 行）

工程侧的三处改动很套路化，但**每一处都不能漏**。漏了任何一处，编译就会报"找不到库"或者"没生成测试程序"。

## 6.1 顶层 `CMakeLists.txt`：+1 行

```diff
--- a/CMakeLists.txt
+++ b/CMakeLists.txt
@@ -33,6 +33,7 @@ message(STATUS "FFMPEG 头文件: ${FFMPEG_INCLUDE_DIRS}")
 add_subdirectory(src/core/common)
 add_subdirectory(src/core/capture)
 add_subdirectory(src/core/encode)
+add_subdirectory(src/core/output)
 add_subdirectory(demo/01-mini-pusher)
 add_subdirectory(src/app)
 add_subdirectory(tests)
```

**就一行，但位置很有讲究。**

### 为什么插在 `encode` 之后、`demo` 之前？

CMake 的 `add_subdirectory` 是**按顺序处理**的。规则是：

> **一个 target 必须先被定义（用 `add_library` / `add_executable`），才能被其它 target 用 `target_link_libraries` 引用。**

我们的依赖方向是：

```
core_common  ←（INTERFACE，纯头文件，无链接依赖）
    ↑
core_capture, core_encode, core_output
    ↑
demo/01-mini-pusher, src/app（主程序）
    ↑
tests（测试，依赖上面所有）
```

所以顺序必须是：**先 common，再 capture/encode/output，再 demo/app，最后 tests。**

如果写反了（比如把 `add_subdirectory(tests)` 放在最前面），CMake 会报：

```
CMake Error at tests/CMakeLists.txt:12 (target_link_libraries):
  Target "test_output" links to target "core_output" but the target was not found.
```

> **老师点评**：这个"顺序即依赖方向"的约束，其实是个**好事**——它强制你把依赖图画清楚。
>
> 第 1 课我们讲过"单向依赖纪律"：**`app` 可以依赖 `core`，`core` 不能依赖 `app`；`core` 内部各模块之间尽量不互相依赖。** CMake 的 `add_subdirectory` 顺序无意中帮我们 enforcing 了这条纪律。
>
> 如果你发现"我必须把 A 放在 B 前面，同时 B 又必须放在 A 前面"——**说明你有循环依赖，架构出问题了。**

## 6.2 `src/core/output/CMakeLists.txt`：新增 14 行

```cmake
 1  add_library(core_output STATIC
 2    StreamOutput.cpp
 3    StreamOutput.h
 4  )
 5
 6  target_include_directories(core_output
 7    PUBLIC ${CMAKE_CURRENT_SOURCE_DIR}/../..
 8    PRIVATE ${FFMPEG_INCLUDE_DIRS}
 9  )
10
11  target_link_libraries(core_output PUBLIC
12    core_common
13    ${FFMPEG_LIBRARIES}
14  )
```

**跟 `src/core/encode/CMakeLists.txt` 逐字对照一下**：

```cmake
# encode 的（第 8 课）
add_library(core_encode STATIC
  VideoEncoder.cpp
  VideoEncoder.h
  AudioEncoder.cpp
  AudioEncoder.h
  EncoderCommon.h
)
target_include_directories(core_encode
  PUBLIC ${CMAKE_CURRENT_SOURCE_DIR}/../..
  PRIVATE ${FFMPEG_INCLUDE_DIRS}
)
target_link_libraries(core_encode PUBLIC
  core_common
  ${FFMPEG_LIBRARIES}
)
```

**结构完全一样，只是名字和文件列表不同。** 这就是"**模板化**"的好处——第 8 课定好了规矩，第 9 课照抄即可，不会出错。

### 逐条讲解

#### 第 1~4 行：`add_library(core_output STATIC ...)`

```cmake
add_library(core_output STATIC
  StreamOutput.cpp
  StreamOutput.h
)
```

**第 2~3 行把 `.h` 也列进了源文件列表。** 这是**故意的**，两个作用：

1. **IDE 可见性**：Visual Studio / CLion / Qt Creator 会把这些头文件显示在项目的"源文件"树里，方便点开编辑；
2. **重新构建触发**：改了头文件，CMake 知道要重编（其实 CMake 自己会扫描依赖，但列出来更保险）。

**`STATIC`** = 静态库（`.lib`）。我们的三个 core 模块都是静态库，最终被链接进可执行文件。

> **为什么不做成动态库（SHARED）？**
>
> 因为这个项目是**单一可执行文件**的产品，没有插件机制，不需要运行时加载。静态库的优点：
> - **没有 DLL Hell**（不用操心 DLL 放哪、版本对不对）；
> - **链接期可以做更多优化**（LTO）；
> - **部署简单**（一个 exe 走天下）。
>
> 缺点：**每个可执行文件都有一份拷贝**。我们只有一个主程序 + 几个测试程序，无所谓。

#### 第 6~9 行：include 目录的 `PUBLIC` / `PRIVATE` 之分 ★

```cmake
target_include_directories(core_output
  PUBLIC ${CMAKE_CURRENT_SOURCE_DIR}/../..
  PRIVATE ${FFMPEG_INCLUDE_DIRS}
)
```

**这是 CMake 现代用法里最重要的一个概念**，同学们务必理解。

| 关键字 | 含义 | 传播性 |
|---|---|---|
| `PUBLIC` | 我自己要用，**依赖我的 target 也要用** | 向上传播 |
| `PRIVATE` | 只有我自己用 | **不传播** |
| `INTERFACE` | 我自己不用，**依赖我的 target 要用** | 向上传播 |

对应到我们的例子：

```
${CMAKE_CURRENT_SOURCE_DIR}/../..   =  src/ 目录
        ↓ PUBLIC
所有链接 core_output 的 target，都能 #include "core/output/StreamOutput.h"
        ↓
这就是为什么 tests/test_output.cpp 能写：
        #include "core/output/StreamOutput.h"    ✅

${FFMPEG_INCLUDE_DIRS}  =  FFmpeg 的头文件目录
        ↓ PRIVATE
只有 StreamOutput.cpp 自己能 #include <libavformat/avformat.h>
        ↓
test_output.cpp 想 include FFmpeg 头文件？它自己得重新找一遍（它确实也这么做了）
```

**为什么 FFmpeg 的 include 是 PRIVATE？**

**这正是 PImpl 在构建系统层面的体现！**

因为 `StreamOutput.h` 里**没有** include 任何 FFmpeg 头文件（除了 `avcodec.h` 和 `rational.h`，那是通过 `PRIVATE` 不太严谨……），所以**使用方不需要 FFmpeg 的 include 路径**。

> **老师注意一个小瑕疵**：严格来说，`StreamOutput.h` **确实** include 了 `libavcodec/avcodec.h` 和 `libavutil/rational.h`。所以理论上 `${FFMPEG_INCLUDE_DIRS}` 应该是 `PUBLIC` 的。
>
> 现在能编过，是因为 **`tests/CMakeLists.txt` 里给 `test_output` 也链了 `core_encode`，而 `core_encode` 又因为某种原因把 FFmpeg 的 include 传出来了**——或者更可能是，FFmpeg 的头文件装在 vcpkg 的 `include/` 下，而这个目录**早就被顶层配置或者 vcpkg 的工具链带进来了**。
>
> **这是个"碰巧能编过"的隐患。** 如果哪天把 `core_encode` 的链接去掉，`test_output` 可能就编不过了。
>
> **正确的做法**：要么把 `${FFMPEG_INCLUDE_DIRS}` 设为 `PUBLIC`，要么提供一个 `core_ffmpeg` 的 INTERFACE 库统一管理 FFmpeg 的 include 和 lib。**这留给同学们做第 10 章的重构练习。**

#### 第 11~14 行：链接库

```cmake
target_link_libraries(core_output PUBLIC
  core_common
  ${FFMPEG_LIBRARIES}
)
```

两个依赖：

| 依赖 | 为什么 |
|---|---|
| `core_common` | 需要 `StreamTarget`、`LogCallback`（虽然是 INTERFACE 库只有头文件，但链上更规范，能传递 include 路径） |
| `${FFMPEG_LIBRARIES}` | 需要 `avformat`、`avcodec`、`avutil` 这三个库 |

**这里用 `PUBLIC` 是合适的**——因为 FFmpeg 是"传递性依赖"（任何链接 `core_output` 的程序，最终都需要 FFmpeg 的库才能链接成功）。

> **老师补充**：`core_common` 是个 `INTERFACE` 库：
> ```cmake
> add_library(core_common INTERFACE)
> target_include_directories(core_common INTERFACE
>   ${CMAKE_CURRENT_SOURCE_DIR}/../..
> )
> ```
> 它没有编译产物，只是"传递一组使用要求（usage requirements）"。这是纯头文件模块的**最佳载体**（第 1 课讲过）。

## 6.3 `tests/CMakeLists.txt`：+9 行

```diff
--- a/tests/CMakeLists.txt
+++ b/tests/CMakeLists.txt
@@ -38,6 +38,15 @@ target_link_libraries(test_encode PRIVATE
   core_encode
 )
 
+add_executable(test_output
+  test_output.cpp
+)
+
+target_link_libraries(test_output PRIVATE
+  core_encode
+  core_output
+)
+
 add_executable(test_audio_capture
   test_audio_capture.cpp
 )
```

### 第 41~43 行：`add_executable(test_output ...)`

定义一个新的可执行目标 `test_output`。

**注意这里用的是 `PRIVATE`**（不是 `PUBLIC`）：

```cmake
target_link_libraries(test_output PRIVATE core_encode core_output)
```

**可执行文件（exe）永远用 `PRIVATE`**——因为没有别人会再链接一个 exe。用 `PUBLIC` 也不会报错，但语义不对。

### 第 45~48 行：链接两个库

```cmake
target_link_libraries(test_output PRIVATE
  core_encode      # 需要 VideoEncoder / AudioEncoder（造测试数据）
  core_output      # 需要 StreamOutput（被测试的对象）
)
```

**为什么需要 `core_encode`？**

因为 `test_output.cpp` 要**真的编出码流**才能验证封装——它不能凭空造 `AVPacket`。

```
makeTestFrame()  ──→  VideoEncoder::encode()  ──→  AVPacket（H.264）
makeSineChunk()  ──→  AudioEncoder::encode()  ──→  AVPacket（AAC）
                                                       ↓
                                              StreamOutput::writeVideo/writeAudio
                                                       ↓
                                                   output_*.flv
                                                       ↓
                                              probeFlv() 读回来验证
```

**这是"端到端"测试**（第 7 章详讲），比"只测 StreamOutput"更有说服力。

> **老师点评**：这种测试组织方式非常值得学习。
>
> 很多人的单元测试是"给被测模块喂假数据"。但**对于"封装"这种东西，假数据测不出真问题**——你喂一个瞎编的 `AVPacket`，它照样能写进文件，但你根本不知道写出来的东西播放器认不认。
>
> **用真实的编码器产出真实数据 → 封装 → 再用真实的解码器读回来验证**，这才叫"能证明系统能工作"的测试。代价是跑得慢一点（要真的编码 60 帧），但**完全值得**。

## 6.4 三处改动的" checklist"

同学们，以后你们在这个项目里加新模块，就照这个清单走：

| # | 要改的地方 | 内容 | 漏了会怎样 |
|---|---|---|---|
| 1 | `src/core/<新模块>/CMakeLists.txt` | 新建，`add_library` + `target_include_directories` + `target_link_libraries` | 模块根本不参与构建 |
| 2 | 顶层 `CMakeLists.txt` | `add_subdirectory(src/core/<新模块>)`，**放在正确的位置**（依赖它的之前） | CMake 报"target not found" |
| 3 | `tests/CMakeLists.txt`（可选） | `add_executable` + `target_link_libraries` | 没有测试程序 |
| 4 | 需要重新跑 CMake | `cmake -S . -B build` 或者直接 `build.bat` | 用旧的缓存，新模块不生效 |

**第 4 条特别提醒新手**：改了 CMakeLists 之后，**必须重新跑 CMake 配置**。只点"生成"（build）是不够的——CMake 会在构建开始时自动检测 CMakeLists 是否变化并重跑配置，但**如果你用的是 IDE 的"编译"按钮，有时候不会触发**。

保险起见，改完 CMakeLists 就跑一次：

```bat
build.bat
```

或者手动：

```bat
cmake -S . -B build -G "Visual Studio 17 2022" -A x64
cmake --build build --config Release
```

---

# 第 7 章 逐行教学（四）：`tests/test_output.cpp`（559 行）

同学们，前面讲了 442 行的实现。这一章的 559 行**比实现还长**——这不是偶然。

> **本项目有一个贯穿始终的规矩**（第 0 课 4.3 节讲过）：**每个核心模块都配一个独立的可执行测试程序，而且这个测试程序不只是跑断言，它还要把结果存成文件，让你用眼睛看。**

`test_output.cpp` 就是 `StreamOutput` 的"陪练"。它做的事情，用一句话概括：

> **真的编码 → 真的封装 → 写成本地 flv → 再用 FFmpeg 读回来 → 检查每一条断言 → 顺便存一张图给你看。**

## 7.1 整体结构鸟瞰

```
第   1~ 18 行   头文件
第  20~ 48 行   测试基础设施：计数器 + check + elapsedMs
第  50~112 行   BMP 结构体 + saveBmp（把解码出来的第一帧存成图片）
第 114~176 行   两个"造数据"函数：makeTestFrame / makeSineChunk
第 178~259 行   RunResult + runOnce（端到端跑一次）
第 261~377 行   FlvInfo + probeFlv（把文件读回来检查）
第 379     行   匿名命名空间结束
第 381~559 行   main：四个测试用例
   第 403~453 行   用例 1：单路输出到本地 flv
   第 455~488 行   用例 2：多路分发，一次编码写两路
   第 490~517 行   用例 3：断线隔离
   第 519~545 行   用例 4：重连接口
   第 547~559 行   汇总与退出码
```

## 7.2 第 1~18 行：头文件

```cpp
 1  #include "core/encode/AudioEncoder.h"
 2  #include "core/encode/VideoEncoder.h"
 3  #include "core/output/StreamOutput.h"
 4
 5  extern "C" {
 6  #include <libavcodec/avcodec.h>
 7  #include <libavformat/avformat.h>
 8  #include <libswscale/swscale.h>
 9  }
10
11  #include <algorithm>
12  #include <chrono>
13  #include <cmath>
14  #include <cstdint>
15  #include <cstdio>
16  #include <cstdlib>
17  #include <string>
18  #include <vector>
```

| 行 | 内容 | 用途 |
|---|---|---|
| 1~2 | 两个编码器 | **造测试数据**（真的编出 H.264 / AAC） |
| 3 | `StreamOutput.h` | **被测对象** |
| 6 | `avcodec.h` | 解码器（`probeFlv` 里要用） |
| 7 | `avformat.h` | **解封装**（`avformat_open_input` / `av_read_frame`） |
| 8 | `swscale.h` | 解码后的 YUV → BGRA（存 BMP 用） |
| 11 | `<algorithm>` | `std::max` |
| 12 | `<chrono>` | 计时（`steady_clock`） |
| 13 | `<cmath>` | `std::sin`（造正弦波音频） |
| 16 | `<cstdlib>` | `std::system()`（第 901 行 `chcp 65001`） |

## 7.3 第 20~48 行：测试基础设施

```cpp
20  namespace {
21
22  int g_passed = 0;
23  int g_failed = 0;
24
25  constexpr double kPi = 3.14159265358979323846;
26
27  void check(bool condition, const char* what)
28  {
29      if (condition) {
30          ++g_passed;
31          std::printf("  [通过] %s\n", what);
32      } else {
33          ++g_failed;
34          std::printf("  [失败] %s\n", what);
35      }
36  }
37
38  void check(bool condition, const std::string& what)
39  {
40      check(condition, what.c_str());
41  }
42
43  long long elapsedMs(std::chrono::steady_clock::time_point from)
44  {
45      return std::chrono::duration_cast<std::chrono::milliseconds>(
46                 std::chrono::steady_clock::now() - from)
47          .count();
48  }
```

### 第 22~23 行：两个全局计数器

```cpp
int g_passed = 0;
int g_failed = 0;
```

**为什么是全局变量而不是成员变量？** 因为这是个**单文件的小测试程序**，没有类、没有对象，全局计数器最简单。

**为什么放在匿名命名空间里？** 保证它只在这个 .cpp 里可见，不会跟其它测试程序的 `g_passed` 撞名（第 5.1 节讲过）。

### 第 27~41 行：两个 `check` 重载

```cpp
void check(bool condition, const char* what);       // ① 接受字符串字面量
void check(bool condition, const std::string& what); // ② 接受 std::string，转发给 ①
```

**为什么需要两个？** 因为测试里有这两种写法：

```cpp
check(run.ok, "推流输出建立成功");                    // ① 字面量，零开销

std::string msg = "第 " + std::to_string(i) + " 路";
check(ok, msg);                                     // ② std::string，需要先构造
```

如果没有 ②，写 `check(ok, msg)` 会编译失败（因为 `std::string` 不能隐式转 `const char*`）。

**重载 ② 的实现只有一行**：`check(condition, what.c_str());` —— 转发给 ①，避免重复计数逻辑。

**这是"测试框架"的最小实现**——总共 15 行，够用了。真实项目里你们会用 Google Test / Catch2 / doctest，但**原理完全一样**：断言 → 计数 → 汇报。

> **老师点评**：同学们注意 `check` 的**失败不中断**——它记录失败然后**继续执行**。
>
> 这跟 `assert()` 不一样（`assert` 失败会 `abort()` 整个进程）。
>
> **"失败继续"对诊断更有价值**：一次跑完 30 条断言，你能看到"哪些过了、哪些没过"，一眼就定位到问题层次。如果用 `assert`，第一个失败就退出，你得改一次跑一次。

### 第 43~48 行：`elapsedMs`

```cpp
long long elapsedMs(std::chrono::steady_clock::time_point from)
{
    return std::chrono::duration_cast<std::chrono::milliseconds>(
               std::chrono::steady_clock::now() - from)
        .count();
}
```

**为什么用 `steady_clock` 而不是 `system_clock`？**

| 时钟 | 特点 | 用途 |
|---|---|---|
| `std::chrono::system_clock` | **墙上时钟**，可以被系统时间调整（NTP 同步、用户改时间）影响，**可能倒退** | 显示当前日期时间 |
| **`std::chrono::steady_clock`** | **单调时钟**，只增不减，不受系统时间调整影响 | **★ 测量时间间隔** |

**测量耗时必须用 `steady_clock`。** 如果你用 `system_clock`，一旦 NTP 把系统时间往回调了 2 秒，你算出来的"耗时"就是负数。

> 第 2 课的缺陷 6 里我们说过"pts 必须来自采集那一刻的真实时钟"——用的就是 `steady_clock`。**这是同一个道理。**

## 7.4 第 50~72 行：BMP 文件头结构体

```cpp
50  #pragma pack(push, 1)
51  struct BmpFileHeader {
52      uint16_t type;
53      uint32_t size;
54      uint16_t reserved1;
55      uint16_t reserved2;
56      uint32_t offBits;
57  };
58
59  struct BmpInfoHeader {
60      uint32_t size;
61      int32_t width;
62      int32_t height;
63      uint16_t planes;
64      uint16_t bitCount;
65      uint32_t compression;
66      uint32_t sizeImage;
67      int32_t xPelsPerMeter;
68      int32_t yPelsPerMeter;
69      uint32_t clrUsed;
70      uint32_t clrImportant;
71  };
72  #pragma pack(pop)
```

### 为什么需要自己定义 BMP 头？

因为**测试要"存一张图让人看"**，但我们的项目**没有引入任何图像库**（没有 stb_image_write，没有 OpenCV，Qt 的 QImage 又不想在 core 层用）。

**BMP 是唯一一个"手写 54 字节头 + 原始像素"就能生成的文件格式**，不需要任何库、不需要压缩算法。第 4/5 课的测试程序里也是这么干的。

### `#pragma pack(push, 1)` 是什么？

**这是本课最重要的 C/C++ 底层知识点之一。**

默认情况下，编译器会对结构体做**内存对齐（alignment）**——在成员之间插入填充字节，让每个成员的起始地址是它大小的整数倍（这样 CPU 访问更快）。

看 `BmpFileHeader`：

```cpp
struct BmpFileHeader {
    uint16_t type;        // 2 字节，偏移 0
    uint32_t size;        // 4 字节
    ...
};
```

**默认对齐下**，编译器会这样排布：

```
偏移 0~1   : type       (2 字节)
偏移 2~3   : ★ 填充 2 字节（为了让 size 对齐到 4）
偏移 4~7   : size       (4 字节)
偏移 8~9   : reserved1
偏移 10~11 : reserved2
偏移 12~15 : offBits
─────────────────────────
sizeof = 16   ← 而不是 2+4+2+2+4 = 14！
```

**但 BMP 文件格式要求头是紧凑的 14 字节**，多出 2 字节填充，写出来的文件就是坏的。

**`#pragma pack(1)` 就是告诉编译器："别加填充，按 1 字节对齐"。**

| 指令 | 含义 |
|---|---|
| `#pragma pack(push, 1)` | ① 把当前的对齐设置**压栈**，② 设置为 1 字节对齐 |
| `#pragma pack(pop)` | 恢复之前压栈的对齐设置 |

**用 `push`/`pop` 而不是直接 `#pragma pack(1)` ... `#pragma pack()`**，是因为前者能**精确恢复**到之前的值（哪怕之前是 4 或 8），而后者会恢复到编译器默认（可能跟之前不同）。

> **老师提醒**：这个"结构体直接对应文件格式"的技巧，在处理**二进制协议、文件头、网络包头**时极其常用。但有两个坑：
>
> 1. **字节序（endianness）**：BMP 是小端，x86 也是小端，所以这里碰巧对；如果要跨平台（大端的 ARM/PowerPC），必须做字节序转换；
> 2. **类型大小**：必须用 `<cstdint>` 里的定宽类型（`uint16_t`、`uint32_t`），**绝不能用 `int` / `long`**（它们的大小随平台变）。代码里做对了 ✅。

### 两个结构体的字段对照

**`BmpFileHeader`（14 字节）**：

| 字段 | 大小 | 值 | 含义 |
|---|---|---|---|
| `type` | 2 | `0x4D42` | 魔数。`0x4D='M'`, `0x42='B'` → **"BM"**（注意小端存储，文件里读出来是 "BM"） |
| `size` | 4 | 整个文件大小 | |
| `reserved1/2` | 2+2 | 0 | 保留 |
| `offBits` | 4 | 54 | **像素数据从文件第几个字节开始**（14 + 40） |

**`BmpInfoHeader`（40 字节）**：

| 字段 | 值 | 含义 |
|---|---|---|
| `size` | 40 | 这个头自己的大小 |
| `width` / `height` | 640 / 360 | 尺寸 |
| `planes` | 1 | 恒为 1 |
| `bitCount` | 32 | **每像素 32 位 = BGRA** |
| `compression` | 0 | **BI_RGB = 不压缩** |
| `sizeImage` | w×h×4 | 像素数据字节数 |
| `xPelsPerMeter` | 2835 | DPI（2835 px/m ≈ 72 DPI） |
| `clrUsed` / `clrImportant` | 0 | 不用调色板 |

## 7.5 第 74~112 行：`saveBmp()`

```cpp
 74  bool saveBmp(const std::string& path, const VideoFrame& frame)
 75  {
 76      if (frame.width <= 0 || frame.height <= 0
 77          || frame.bgra.size() < static_cast<size_t>(frame.width) * frame.height * 4)
 78          return false;
 79
 80      const uint32_t imageBytes =
 81          static_cast<uint32_t>(frame.width) * frame.height * 4;
 82
 83      BmpFileHeader fileHeader = {};
 84      fileHeader.type = 0x4D42;
 85      fileHeader.size = sizeof(BmpFileHeader) + sizeof(BmpInfoHeader) + imageBytes;
 86      fileHeader.offBits = sizeof(BmpFileHeader) + sizeof(BmpInfoHeader);
 87
 88      BmpInfoHeader infoHeader = {};
 89      infoHeader.size = sizeof(BmpInfoHeader);
 90      infoHeader.width = frame.width;
 91      infoHeader.height = frame.height;
 92      infoHeader.planes = 1;
 93      infoHeader.bitCount = 32;
 94      infoHeader.sizeImage = imageBytes;
 95      infoHeader.xPelsPerMeter = 2835;
 96      infoHeader.yPelsPerMeter = 2835;
 97
 98      std::FILE* file = std::fopen(path.c_str(), "wb");
 99      if (!file)
100          return false;
101
102      std::fwrite(&fileHeader, sizeof(fileHeader), 1, file);
103      std::fwrite(&infoHeader, sizeof(infoHeader), 1, file);
104
105      const size_t stride = static_cast<size_t>(frame.width) * 4;
106      for (int y = frame.height - 1; y >= 0; --y)
107          std::fwrite(frame.bgra.data() + static_cast<size_t>(y) * stride, stride, 1,
108                      file);
109
110      std::fclose(file);
111      return true;
112  }
```

### 第 76~78 行：参数校验

```cpp
if (frame.width <= 0 || frame.height <= 0
    || frame.bgra.size() < static_cast<size_t>(frame.width) * frame.height * 4)
    return false;
```

**检查"数据够不够"**——防止后面 `fwrite` 读到 `bgra` 之外（缓冲区溢出）。

**注意第 77 行的 `static_cast<size_t>`**：`frame.width * frame.height * 4` 如果都是 `int`，640×360×4 = 921600，**还不会溢出 int**（int 上限 21 亿）。但如果是 1920×1080×4 = 8294400，也还好。**如果是 4K（3840×2160×4 = 33 MB）就危险了**——虽然也不溢出，但养成"做乘法前先转大类型"的习惯，能避免将来踩坑。

> **老师补充**：更严谨的写法是先转再乘：
> ```cpp
> static_cast<size_t>(frame.width) * static_cast<size_t>(frame.height) * 4
> ```
> 因为 `static_cast<size_t>(frame.width) * frame.height` 中，`frame.height` 会被提升为 `size_t`，所以**第一个转换就够了**（代码里正是这么写的 ✅）。

### 第 83、88 行：`= {}` 零初始化

```cpp
BmpFileHeader fileHeader = {};
BmpInfoHeader infoHeader = {};
```

**必须零初始化！** 那些没显式赋值的字段（`reserved1`、`compression`、`clrUsed`……）必须都是 0。

如果不加 `= {}`，这些字段是**未初始化的垃圾值**，写出来的 BMP 头里有一堆随机数——**有些看图软件能打开，有些打不开**，这种"玄学 bug"最难查。

### 第 106~108 行：★ BMP 的行序是"自下而上"

```cpp
for (int y = frame.height - 1; y >= 0; --y)
    std::fwrite(frame.bgra.data() + static_cast<size_t>(y) * stride, stride, 1, file);
```

**这是 BMP 格式最经典的一个坑**：

> **BMP 的像素数据是从"最后一行"开始存的（bottom-up，自下而上）。**

也就是说：文件的第一行数据，是图像的**底部**；文件的最后一行数据，是图像的**顶部**。

所以循环要**倒着写**：`y` 从 `height-1` 递减到 `0`。

**如果忘了倒序，图像会上下颠倒。** 这个 bug 极其经典——第 4 课（屏幕采集）和第 5 课（摄像头）里都出现过"图像上下颠倒"的问题，根源往往就在这里或者 `sws_scale` 那里。

> **老师的历史小知识**：为什么 BMP 要自下而上？
>
> 这要追溯到 OS/2 时代的 **Presentation Manager** 和 IBM 的显示硬件约定——当时的图形子系统用"数学坐标系"（y 轴向上）。这个约定被 Windows 的 GDI 继承了下来，然后就一直留在 BMP 格式里了。
>
> **对比**：PNG / JPEG 用的是"自上而下"（top-down）。所以你在做格式转换时，**BMP ↔ PNG 一定要翻行**。

### 第 98~100、110 行：文件句柄管理

```cpp
std::FILE* file = std::fopen(path.c_str(), "wb");
if (!file) return false;
...
std::fclose(file);
```

**`"wb"` 的 `b` 不能省**（在 Windows 上尤其重要）：

| 模式 | 含义 |
|---|---|
| `"w"` | 文本模式。**Windows 下会把 `\n` 自动转成 `\r\n`！** |
| **`"wb"`** | **二进制模式**，不做任何转换 |

**如果你写二进制文件用了 `"w"`，每遇到一个字节 `0x0A` 就会被写成 `0x0D 0x0A`** —— 图像数据里 `0x0A` 到处都是，文件立刻就毁了。

**这是 Windows 平台特有的经典 bug**（Linux 上 `"w"` 和 `"wb"` 没区别，所以很多 Linux 上跑得好好的代码移植到 Windows 就出问题）。

另外，这里用 `std::FILE*` 而**不是 RAII 的 `std::ofstream`**，风格上有点退步（如果中间 `fwrite` 失败提前 return，文件句柄会泄漏）。**理想的做法**是用一个 RAII 包装。不过对于测试程序，可以接受。

## 7.6 第 114~155 行：`makeTestFrame()` —— 造一帧测试画面

```cpp
114  VideoFramePtr makeTestFrame(int width, int height, int index)
115  {
116      auto frame = std::make_shared<VideoFrame>();
117      frame->width = width;
118      frame->height = height;
119      frame->pts = index;
120      frame->bgra.resize(static_cast<size_t>(width) * height * 4);
121
122      static const uint8_t kColors[8][3] = {
123          {255, 255, 255}, {255, 255, 0}, {0, 255, 255}, {0, 255, 0},
124          {255, 0, 255},   {255, 0, 0},   {0, 0, 255},   {0, 0, 0}};
125
126      for (int y = 0; y < height; ++y) {
127          for (int x = 0; x < width; ++x) {
128              const int bar = x * 8 / width;
129              uint8_t* p =
130                  frame->bgra.data() + (static_cast<size_t>(y) * width + x) * 4;
131              p[0] = kColors[bar][2];
132              p[1] = kColors[bar][1];
133              p[2] = kColors[bar][0];
134              p[3] = 255;
135          }
136      }
137
138      const int boxSize = height / 6;
139      const int span = width - boxSize;
140      const int boxX = span > 0 ? (index * 13) % span : 0;
141      const int boxY = height / 2 - boxSize / 2;
142
143      for (int y = boxY; y < boxY + boxSize; ++y) {
144          for (int x = boxX; x < boxX + boxSize; ++x) {
145              uint8_t* p =
146                  frame->bgra.data() + (static_cast<size_t>(y) * width + x) * 4;
147              p[0] = 0;
148              p[1] = 0;
149              p[2] = 0;
150              p[3] = 255;
151          }
152      }
153
154      return frame;
155  }
```

### 第 116 行：为什么用 `make_shared`？

```cpp
auto frame = std::make_shared<VideoFrame>();
```

因为项目的契约（`Common.h`）定义的：`using VideoFramePtr = std::shared_ptr<VideoFrame>;`

**为什么 `VideoFrame` 要用 `shared_ptr`？** 因为第 3 课设计它时就是为了**跨线程传递**（采集线程 → 队列 → 编码线程），`shared_ptr` 天然支持多线程共享所有权。

**`make_shared` 而不是 `shared_ptr<VideoFrame>(new VideoFrame)`**，两个原因：

1. **异常安全**：`make_shared` 一次分配完，不会出现在"new 成功但 shared_ptr 构造抛异常"时泄漏；
2. **性能**：`make_shared` 把"对象"和"控制块"分配在**同一块内存**（1 次 malloc），`shared_ptr(new T)` 要 2 次。

### 第 119 行：`frame->pts = index`

跟第 2 课的 demo 一样——**用帧号当 pts**。

**这在测试里是完全可以接受的**（我们在第 2 课 3.1 节讲过它的局限），因为：

- 测试里我们是**同步顺序执行**的，"第 N 帧的时刻"就是"生成第 N 帧的时刻"；
- 我们要验证的是"**时间戳这套机制接没接通**"，而不是"时钟准不准"。

### 第 122~124 行：八色彩条

```cpp
static const uint8_t kColors[8][3] = {
    {255, 255, 255},  // 白
    {255, 255, 0},    // 黄
    {0, 255, 255},    // 青
    {0, 255, 0},      // 绿
    {255, 0, 255},    // 品红
    {255, 0, 0},      // 红
    {0, 0, 255},      // 蓝
    {0, 0, 0}};       // 黑
```

**注意这里的顺序是 RGB，不是 BGRA。**

**第 131~133 行的"倒着取"很讲究**：

```cpp
p[0] = kColors[bar][2];   // B ← R
p[1] = kColors[bar][1];   // G ← G
p[2] = kColors[bar][0];   // R ← B
p[3] = 255;               // A
```

因为 `VideoFrame::bgra` 的内存布局是 **B, G, R, A**（第 3 课定的规矩），而 `kColors` 数组写成了人眼习惯的 **R, G, B**。所以要**倒过来取**。

**这种"数组按人类直觉写，赋值时转换"的写法，比"把数组直接写成 BGR 顺序"可读性好得多**——看数组就知道是"白黄青绿品红蓝黑"（标准的彩条测试图顺序）。

> **老师补充**：这 8 个颜色就是标准的 **SMPTE 彩条（color bars）**的简化版——电视台调试用的那张图。**为什么用彩条测试？**
>
> 1. **高对比度、边缘锐利** → 能暴露编码器的块效应、色度采样错误；
> 2. **包含饱和的纯色** → 能暴露 YUV 转换的电平范围错误（16-235 vs 0-255）；
> 3. **人眼一眼就能看出错** → 如果颜色串位了（比如红变蓝），你立刻知道 RGB/BGR 弄反了。

### 第 128 行：`const int bar = x * 8 / width;`

把画面横向分成 8 条，第 `x` 个像素属于第 `bar` 条彩条。

**注意运算顺序**：`x * 8 / width` 而不是 `x / (width / 8)`。

- `x * 8 / width`：`x` 最大 639，`639*8 = 5112`，**不会溢出 int**，而且除法是整数除法向下取整，**边界正确**；
- `x / (width / 8)`：如果 `width` 不能被 8 整除，会分得不均匀。

### 第 138~152 行：移动的黑方块

```cpp
const int boxSize = height / 6;                        // 方块边长 = 60（360/6）
const int span = width - boxSize;                      // 可移动范围 = 640-60 = 580
const int boxX = span > 0 ? (index * 13) % span : 0;   // X 位置随帧号变
const int boxY = height / 2 - boxSize / 2;             // Y 居中
```

**为什么要加一个移动的方块？**

因为**纯彩条是静态的，看不出帧率对不对、也看不出帧序有没有乱**。

加一个每帧移动 13 像素的黑方块，你打开 flv 一看：

- **方块匀速从左往右扫** → 帧序正确、帧率正确；
- **方块一跳一跳** → 丢帧了；
- **方块倒着走** → 帧序反了；
- **方块不动** → 时间戳没生效，播放器在"来一帧播一帧"。

> **老师点评**：这是"**给测试数据加一个可观测的动态特征**"的经典技巧。
>
> 在调试视频系统时，老师强烈建议你：**往画面里加一个随时间变化的标记**（移动的方块、递增的数字、秒表）。它能让你在 3 秒内判断出"帧率、帧序、时间戳"这三件事对不对——而这三件事恰恰是最容易出错的。

### 第 140 行：`(index * 13) % span`

**13 这个数字是随便选的吗？** 不完全是。它跟 `span = 580` **互质**（gcd(13, 580) = 1），所以方块的移动轨迹会在整个宽度上**均匀遍历**，不会重复走几条固定路线。

这种"选互质的数"的技巧在**哈希、随机数发生器、采样**里都很常见。

> 同学们做实验时可以把它改成 1（走得慢，看清细节）或者 100（走得快，快速验证）。

## 7.7 第 157~176 行：`makeSineChunk()` —— 造一段测试音频

```cpp
157  AudioChunkPtr makeSineChunk(int64_t startSample, int frames)
158  {
159      constexpr int kRate = 48000;
160      constexpr int kChannels = 2;
161
162      auto chunk = std::make_shared<AudioChunk>();
163      chunk->sampleRate = kRate;
164      chunk->channels = kChannels;
165      chunk->pts = startSample;
166      chunk->samples.resize(static_cast<size_t>(frames) * kChannels);
167
168      for (int i = 0; i < frames; ++i) {
169          const double t =
170              static_cast<double>(startSample + i) / static_cast<double>(kRate);
171          const float value = static_cast<float>(std::sin(2.0 * kPi * 440.0 * t)) * 0.3f;
172          for (int ch = 0; ch < kChannels; ++ch)
173              chunk->samples[static_cast<size_t>(i) * kChannels + ch] = value;
174      }
175      return chunk;
176  }
```

### 第 159~160 行：48kHz / 双声道

这是**直播的标准配置**（第 7 课 WASAPI 采集定下的规矩）：

| 参数 | 值 | 为什么 |
|---|---|---|
| 采样率 | 48000 Hz | 专业音频/视频行业标准（44.1k 是音乐 CD 标准） |
| 声道 | 2（立体声） | 直播平台普遍要求 |
| 采样格式 | `float`（交错） | `AudioChunk::samples` 是 `std::vector<float>` |

### 第 165 行：`chunk->pts = startSample`

**音频的 pts 单位是"采样点"**，不是毫秒也不是帧号。

这是一个**关键设计**：第 7 课定的规矩是"`AudioChunk::pts` = 这个 chunk 的第一个采样点在整个流中的序号"。

**为什么用采样点？** 因为音频是按采样处理的，`48000 个采样 = 1 秒`是最自然的单位，而且**不会丢精度**（用毫秒的话，48kHz 下每个采样是 0.0208 毫秒，必然有舍入误差累积）。

### 第 168~174 行：440 Hz 正弦波

```cpp
const double t = (startSample + i) / 48000.0;              // 当前时刻（秒）
const float value = std::sin(2π × 440 × t) * 0.3f;         // 幅度 0.3
```

**440 Hz** 是国际标准音高（A4，就是乐队的定音笛那个音）。

**幅度为什么是 0.3 而不是 1.0？**

因为：

1. **留余量（headroom）**——编码、重采样过程中可能有增益；
2. **避免削波（clipping）**——两个声道同相叠加时峰值会翻倍；
3. **AAC 是有损编码**，满幅正弦在某些码率下会产生失真。

**0.3 大约是 -10 dBFS**，是个很安全的测试电平。

**第 172~173 行：两个声道填同样的值**——这是"双声道单声道信号"（dual mono），相位相同，方便验证声道没串位。

> **老师补充一个小瑕疵**：`std::sin` 每帧调用一次，是**比较低效**的（虽然测试里无所谓）。高效做法是用**递推**（旋转矢量法）：
> ```cpp
> // sin(θ + Δ) = sin θ cos Δ + cos θ sin Δ
> ```
> 每帧只需 4 次乘法。这在实时音频合成里是标准优化。

## 7.8 第 178~259 行：`RunResult` 与 `runOnce()` —— 端到端跑一次

### 7.8.1 第 178~183 行：`RunResult`

```cpp
178  struct RunResult {
179      bool ok = false;
180      int videoPackets = 0;
181      int audioPackets = 0;
182      std::vector<OutputTargetInfo> targets;
183  };
```

**"跑一次的结果"**：整体成功了吗？写了多少视频包、多少音频包？每个目标最后什么状态？

**`targets` 字段是测试的关键**——它让我们能在 `runOnce` 返回之后，**检查每一路的健康状态**（这正是用例 3 要验证的）。

### 7.8.2 第 185~259 行：`runOnce()`

```cpp
185  RunResult runOnce(const std::vector<StreamTarget>& targets, int frames, double seconds,
186                    LogCallback log)
187  {
```

| 参数 | 含义 | 用例里的值 |
|---|---|---|
| `targets` | 要测哪些目标 | 用例 1：`{{"local", "output_single.flv"}}` |
| `frames` | 编多少帧视频 | 用例 1：60 帧 |
| `seconds` | 编多少秒音频 | 用例 1：2.0 秒 |
| `log` | 日志回调（传进去） | `log` lambda |

**注意 `frames` 和 `seconds` 是两个独立的参数**——因为视频按帧算、音频按秒算。用例 1 是"60 帧 @ 30fps = 2 秒视频" + "2 秒音频"，正好对齐。

#### 第 187~197 行：配置

```cpp
188      VideoEncoderConfig videoConfig;
189      videoConfig.width = 640;
190      videoConfig.height = 360;
191      videoConfig.fps = 30;
192      videoConfig.bitrateKbps = 800;
193
194      AudioEncoderConfig audioConfig;
195      audioConfig.sampleRate = 48000;
196      audioConfig.channels = 2;
197      audioConfig.bitrateKbps = 128;
```

**注意这里只设了 5 个 / 3 个字段，其它用默认值**：

| 未设置的字段 | 默认值（来自头文件） | 说明 |
|---|---|---|
| `videoConfig.gopSize` | `0` → 编码器内部会用 `fps * 2 = 60` | 2 秒一个关键帧 |
| `videoConfig.maxBFrames` | `0` | **直播必须关 B 帧** |
| `videoConfig.preset` | `"veryfast"` | |
| `videoConfig.tune` | `"zerolatency"` | |

**640×360 @ 30fps @ 800kbps** 是个很轻的配置，测试跑得快。

#### 第 199~203 行：建编码器

```cpp
199      VideoEncoder video;
200      AudioEncoder audio;
201      if (!video.open(videoConfig, nullptr) || !audio.open(audioConfig, nullptr))
202          return result;
```

**这里传的 log 是 `nullptr`** —— 编码器的日志我们不看（第 5.6 节讲过，所有 `log` 调用点都做了判空）。

**第 201 行用 `||` 短路**：如果视频编码器打开失败，**音频编码器根本不会被打开**。这是合理的（视频是必需的）。

> **老师提问**：那 `video` 打开成功了、`audio` 打开失败了怎么办？
>
> **答案**：`return result`（`ok` 保持 `false`）。而 `video` 和 `audio` 是**栈上的局部对象**，函数返回时析构函数会自动 `close()` 它们。**RAII 又一次帮我们兜住了底**（第 5.6.1 节讲过）。

#### 第 205~216 行：★ 先把所有音频包编好存起来

```cpp
205      std::vector<AVPacketPtr> audioPackets;
206      const int64_t totalSamples = static_cast<int64_t>(seconds * 48000.0);
207      for (int64_t pos = 0; pos < totalSamples; pos += 480) {
208          int n = 480;
209          if (pos + n > totalSamples)
210              n = static_cast<int>(totalSamples - pos);
211
212          for (auto& packet : audio.encode(*makeSineChunk(pos, n)))
213              audioPackets.push_back(std::move(packet));
214      }
215      for (auto& packet : audio.flush())
216          audioPackets.push_back(std::move(packet));
```

**为什么要"先把音频全编完"？** 这是本函数最需要解释的一段。

**原因**：我们是**同步顺序执行**的程序。真实系统里，音频采集和视频采集是**两个并行线程**，各自按自己的节奏产生数据，然后**按时间戳交错**交给封装器。

但我们没有线程，怎么办？**模拟**：

```
真实系统（第 10 课之后）：
   采集线程 A ──→ 音频队列 ──┐
                             ├──→ 发送线程：按 pts 取小的先发
   采集线程 B ──→ 视频队列 ──┘

我们的测试（同步）：
  ① 先把 2 秒的音频全部编好，存进 audioPackets 数组（相当于"预先录好"）
  ② 然后一边编视频一边发：每编完一帧视频，就把"所有 pts ≤ 当前时刻"的音频包发出去
     （相当于"按时间戳交错"）
```

**第 207~210 行**：每次送 480 个采样（10 毫秒）进去编。

**为什么是 480？** 因为 AAC 的帧长是 1024 采样，但 FFmpeg 的 AAC 编码器接受任意长度，**内部会缓存到够 1024 个再编一帧**。480 是 48kHz 下 10ms 的采样数，是个"喂数据"的舒适粒度。

**第 212~213 行**：`audio.encode(...)` 返回 `std::vector<AVPacketPtr>`，用范围 for + `std::move` 收进数组。

**第 215~216 行**：**flush 编码器**（`avcodec_send_frame(ctx, nullptr)`），把最后不满一帧的数据挤出来。

> **老师提醒**：这一步**绝对不能忘**！忘了的话，最后不到 1024 个采样的那部分音频就永远留在编码器里，写出来的文件**少了一小截声音**。这在编码视频时也是一样的（第 2 课 4.6 节讲过）。

#### 第 218~222 行：打开输出

```cpp
218      StreamOutput output;
219      if (!output.open(targets, video.codecParameters(), audio.codecParameters(), log))
220          return result;
```

**这一行是"编码器 → 封装器"的交接点**：

```
video.codecParameters()  ──→  StreamOutput::open(...)  ──→  内部深拷贝，存为模板
audio.codecParameters()  ──→                            ──→  内部深拷贝，存为模板
        （const AVCodecParameters*，含 SPS/PPS / AudioSpecificConfig）
```

**第 3 个参数传的是 `audio.codecParameters()`（非空）**——所以用例 1~3 都是有音频的双流输出。只有用例 4 传了 `nullptr`（纯视频）。

#### 第 224~234 行：`pushAudioUpTo` —— 交错的核心

```cpp
224      size_t audioCursor = 0;
225      const auto pushAudioUpTo = [&](int64_t timeMs) {
226          while (audioCursor < audioPackets.size()) {
227              const AVPacket* packet = audioPackets[audioCursor].get();
228              const int64_t audioMs =
229                  packet->pts == AV_NOPTS_VALUE ? 0 : packet->pts * 1000 / 48000;
230              if (audioMs > timeMs)
231                  break;
232              output.writeAudio(*packet, audio.timeBase());
233              ++result.audioPackets;
234              ++audioCursor;
235          }
236      };
```

**这个 lambda 就是"音视频交错器"的雏形。**

它的语义：**"把所有时间戳不超过 `timeMs` 的音频包发出去。"**

**第 228~229 行**：把音频包的 pts（单位：采样点，`{1,48000}`）换算成毫秒：

```cpp
audioMs = pts × 1000 / 48000
```

**注意：这里没有用 `av_packet_rescale_ts`**，而是手写了乘除法。

**为什么可以？** 因为这是在**测试代码**里，我们**确切知道**音频的时间基是 `{1, 48000}`，而且这只是"决定发送顺序"，不是"真正的时间戳"。写进容器的时间戳还是由 `StreamOutput` 内部的 `av_packet_rescale_ts` 负责。

> **老师提醒**：在生产代码里，**永远用 `av_packet_rescale_ts`，不要手写**。手写容易在两个地方出错：
> ① **溢出**：`pts * 1000` 当 pts 很大时会溢出 `int64_t`（虽然 64 位下很难，但 pts 是 64 位、值是采样点，长时间运行后确实可能）；
> ② **舍入**：`av_packet_rescale_ts` 用的是 `av_rescale_q`，有正确的舍入和溢出保护（内部用 128 位整数）。

**第 224 行的 `audioCursor` 是"按引用捕获"（`[&]`）的外部变量**——它记录"下一个要发的音频包"的下标，跨调用保持。这是**"单调推进的游标"**模式，因为音频包是按时间排好序的，所以只需要一个游标。

#### 第 237~247 行：主循环

```cpp
237      for (int i = 0; i < frames; ++i) {
238          for (auto& packet : video.encode(*makeTestFrame(640, 360, i))) {
239              output.writeVideo(*packet, video.timeBase());
240              ++result.videoPackets;
241          }
242          pushAudioUpTo(static_cast<int64_t>(i + 1) * 1000 / videoConfig.fps);
243      }
```

**第 238 行**：造第 `i` 帧 → 编码 → 可能有 0 个或多个输出包（x264 有延迟，可能几个帧后才吐一包）。

**第 239 行**：写视频包。**注意第二个参数是 `video.timeBase()`**（`{1, 30}`），第 4.2 节讲过为什么必须显式传。

**第 242 行**：编完第 `i` 帧（对应时刻 `(i+1)/30` 秒 ≈ `(i+1)*1000/30` 毫秒）之后，把"这一时刻之前的所有音频包"发出去。

```
i = 0：视频第 0 帧（0~33ms）→ 发 pts ≤ 33ms 的音频包
i = 1：视频第 1 帧（33~66ms）→ 发 pts ≤ 66ms 的音频包
...
```

**这就是"按时间对齐的交错"**——视频和音频不会一个全发完再发另一个。

#### 第 249~257 行：收尾

```cpp
249      for (auto& packet : video.flush()) {
250          output.writeVideo(*packet, video.timeBase());
251          ++result.videoPackets;
252      }
253      pushAudioUpTo(INT64_MAX);
254
255      output.flush();
256      result.targets = output.targets();
257      result.ok = true;
258
259      video.close();
260      audio.close();
261      output.close();
262      return result;
```

**第 249~252 行**：flush 视频编码器（挤出最后的包）。

**第 253 行**：`pushAudioUpTo(INT64_MAX)` —— **把剩下的所有音频包一次性发完**。

`INT64_MAX` 是个很巧妙的写法：**"把截止时刻设成无穷大"**，while 循环里的 `audioMs > timeMs` 永远不成立，所以全部发完。

> **老师点评**：这种"用一个极大值表示'全部'"的技巧很常见（Python 里是 `float('inf')`，C++ 里是 `std::numeric_limits<T>::max()`）。
>
> 但要小心 `INT64_MAX` 参与的**算术运算**——`INT64_MAX + 1` 会溢出成负数。这里只做比较，安全。

**第 255 行**：`output.flush()` —— 冲刷封装器的交织缓冲（第 5.7.2 节）。

**第 256 行**：**在 `close()` 之前**取状态快照。

**顺序很重要！** 因为 `close()` 之后所有 `connected` 都会变成 `false`（`closeTarget` 里有 `target.connected = false`）。**必须在关闭前抓快照**。

**第 259~261 行**：显式关闭三个对象。**其实不写也行**（析构函数会做），但显式写出来意图更清晰，而且**能精确控制 trailer 的写入时机**（在函数返回前、而不是在 `result` 被拷贝之后）。

## 7.9 第 261~377 行：`FlvInfo` 与 `probeFlv()` —— 把文件读回来检查

这是本课**最有工程价值**的一段：**用 FFmpeg 自己把写出来的文件读回来，逐条检查。**

### 7.9.1 第 261~273 行：`FlvInfo`

```cpp
261  struct FlvInfo {
262      bool ok = false;                  // 文件能打开吗
263      int streamCount = 0;              // 有几条流
264      bool hasVideo = false;            // 有视频吗
265      bool hasAudio = false;            // 有音频吗
266      int videoPackets = 0;             // 多少个视频包
267      int audioPackets = 0;             // 多少个音频包
268      int64_t maxVideoMs = 0;           // 视频最后一包的时间戳
269      int64_t maxAudioMs = 0;           // 音频最后一包的时间戳
270      bool sawNegativePts = false;      // ★ 见到过负时间戳吗
271      VideoFrame firstFrame;            // 解码出来的第一帧（存 BMP 用）
272      bool firstFrameDecoded = false;   // 第一帧解出来了吗
273  };
```

**这 11 个字段，每一个都对应一条或多条断言。** 同学们看测试代码时，可以反过来推：**"我想验证什么" → "我需要 probe 出什么信息" → "FlvInfo 里加什么字段"。**

### 7.9.2 第 275~290 行：打开文件

```cpp
275  FlvInfo probeFlv(const std::string& path)
276  {
277      FlvInfo info;
278
279      AVFormatContext* ctx = nullptr;
280      if (avformat_open_input(&ctx, path.c_str(), nullptr, nullptr) < 0)
281          return info;
282
283      if (avformat_find_stream_info(ctx, nullptr) < 0) {
284          avformat_close_input(&ctx);
285          return info;
286      }
```

**这是"解封装（demux）"的标准开场**，跟我们写文件正好相反：

| 写（mux） | 读（demux） |
|---|---|
| `avformat_alloc_output_context2` | `avformat_open_input` |
| `avformat_new_stream` | `avformat_find_stream_info`（自动发现流） |
| `avio_open2` | （内部自动打开） |
| `avformat_write_header` | — |
| `av_interleaved_write_frame` | `av_read_frame` |
| `av_write_trailer` | — |
| `avformat_free_context` | `avformat_close_input` |

**第 280 行**：`avformat_open_input` 会自动探测格式（读文件头几个字节，看魔数）。

**第 283 行**：`avformat_find_stream_info` —— **读一部分数据来分析流的参数**（分辨率、编码格式、采样率……）。对 FLV 来说，它会解析 `onMetaData` 和第一个视频 Tag 里的 SPS/PPS。

**第 284 行**：失败时 `avformat_close_input(&ctx)` —— **注意这个带 `&`**，它会关闭并释放 `ctx`（跟 `avformat_free_context(ctx)` 对应）。

### 7.9.3 第 292~299 行：找出视频流和音频流的编号

```cpp
292      info.streamCount = static_cast<int>(ctx->nb_streams);
293
294      int videoIndex = -1;
295      int audioIndex = -1;
296      for (unsigned i = 0; i < ctx->nb_streams; ++i) {
297          if (ctx->streams[i]->codecpar->codec_type == AVMEDIA_TYPE_VIDEO && videoIndex < 0)
298              videoIndex = static_cast<int>(i);
299          if (ctx->streams[i]->codecpar->codec_type == AVMEDIA_TYPE_AUDIO && audioIndex < 0)
300              audioIndex = static_cast<int>(i);
301      }
302      info.hasVideo = videoIndex >= 0;
303      info.hasAudio = audioIndex >= 0;
```

**注意第 297、299 行的 `&& videoIndex < 0`**（`&& audioIndex < 0`）—— **只取第一条**。如果文件里有两条视频流，只认第一条。

**第 296 行的 `unsigned i`**：因为 `ctx->nb_streams` 是 `unsigned int`。用 `unsigned` 避免编译器警告（有符号/无符号比较）。

> **老师点评**：这种"**遍历流找类型**"的代码，在 FFmpeg 里出现频率极高。同学们可以把它背下来当模板：
> ```cpp
> int videoIndex = av_find_best_stream(ctx, AVMEDIA_TYPE_VIDEO, -1, -1, nullptr, 0);
> ```
> 其实 FFmpeg 提供了 `av_find_best_stream()` 一行搞定（还会处理"多视频流选最好的那条"）。手写循环更透明，**两种都行**。

### 7.9.4 第 305~318 行：准备解码器

```cpp
305      AVCodecContext* decoder = nullptr;
306      AVFrame* decoded = nullptr;
307      SwsContext* sws = nullptr;
308      if (videoIndex >= 0) {
309          const AVCodec* codec = avcodec_find_decoder(
310              ctx->streams[videoIndex]->codecpar->codec_id);
311          if (codec) {
312              decoder = avcodec_alloc_context3(codec);
313              if (decoder
314                  && avcodec_parameters_to_context(
315                         decoder, ctx->streams[videoIndex]->codecpar)
316                         >= 0
317                  && avcodec_open2(decoder, codec, nullptr) >= 0) {
318                  decoded = av_frame_alloc();
319              }
320          }
321      }
```

**这一段是完整的"打开解码器"流程**，回忆第 8 课的"打开编码器"，两者是对称的：

| 步骤 | 编码器（第 8 课） | 解码器（这里） |
|---|---|---|
| 找 | `avcodec_find_encoder_by_name("libx264")` | `avcodec_find_decoder(codec_id)` |
| 分配 | `avcodec_alloc_context3(codec)` | 同 |
| 填参数 | 手动设 `width/height/...` | **`avcodec_parameters_to_context(decoder, codecpar)`** ← 从流里读 |
| 打开 | `avcodec_open2(...)` | 同 |
| 用 | `send_frame` / `receive_packet` | `send_packet` / `receive_frame` |

**第 314~316 行**是关键：**`avcodec_parameters_to_context()`** 把 `AVStream::codecpar`（从文件里读出来的参数）**灌进** `AVCodecContext`。这正是第 8 课 `avcodec_parameters_from_context()` 的**逆操作**。

```
编码器这边：  AVCodecContext  ──from_context──→  AVCodecParameters  ──→ 写进容器
                                                        ↓ （文件）
解码器这边：  AVCodecContext  ←──to_context────  AVCodecParameters  ←── 从容器读出
```

**这个"参数包"就是编解码器和容器之间传递信息的标准载体。** 同学们把这两个函数记牢。

**第 313~317 行的链式 `&&`**：三个条件依次判断，短路求值。只有全部成功才 `av_frame_alloc()`。

### 7.9.5 第 320~355 行：遍历所有包

```cpp
320      AVPacket* packet = av_packet_alloc();
321      while (av_read_frame(ctx, packet) >= 0) {
322          const int64_t pts = packet->pts;
323
324          if (packet->stream_index == videoIndex) {
325              ++info.videoPackets;
326              if (pts != AV_NOPTS_VALUE) {
327                  info.maxVideoMs = std::max(info.maxVideoMs, pts);
328                  if (pts < 0)
329                      info.sawNegativePts = true;
330              }
331
332              if (decoder && decoded && !info.firstFrameDecoded) {
333                  if (avcodec_send_packet(decoder, packet) >= 0
334                      && avcodec_receive_frame(decoder, decoded) >= 0) {
```

**第 321 行**：`av_read_frame(ctx, packet)` 循环读包，返回 `< 0` 表示读完了（EOF）或出错。

**第 322 行**：取出 pts。**注意这里 pts 已经是"流的时间基"（`{1,1000}` 毫秒）了**——因为 `av_read_frame` 会自动做换算。这就是为什么 `info.maxVideoMs` 可以**直接**跟 1800/2100 比较。

**第 324~330 行**：统计视频包数、最大 pts、有没有负时间戳。

**第 328~329 行**就是 3.7 节讲的负时间戳检查——**这是验证 AAC priming 归零是否生效的那条断言。**

**第 332~334 行**：只解第一帧（`!info.firstFrameDecoded` 条件），解出来之后就不再解了。因为**我们只需要一张图存 BMP**，解全部 60 帧纯属浪费。

### 7.9.6 第 335~352 行：解码 + 转格式 + 存起来

```cpp
335                      const int w = decoded->width;
336                      const int h = decoded->height;
337                      info.firstFrame.width = w;
338                      info.firstFrame.height = h;
339                      info.firstFrame.bgra.resize(static_cast<size_t>(w) * h * 4);
340
341                      sws = sws_getContext(w, h,
342                                           static_cast<AVPixelFormat>(decoded->format),
343                                           w, h, AV_PIX_FMT_BGRA, SWS_BILINEAR,
344                                           nullptr, nullptr, nullptr);
345                      if (sws) {
346                          uint8_t* dstData[4] = {info.firstFrame.bgra.data(), nullptr,
347                                                 nullptr, nullptr};
348                          const int dstLinesize[4] = {w * 4, 0, 0, 0};
349                          sws_scale(sws, decoded->data, decoded->linesize, 0, h,
350                                    dstData, dstLinesize);
351                          info.firstFrameDecoded = true;
352                      }
```

**第 341~344 行**：`sws_getContext` 建一个"像素格式转换器"。

| 参数 | 值 | 含义 |
|---|---|---|
| 源宽高 | `w, h` | |
| 源格式 | `decoded->format` | 解码出来是 YUV420P |
| 目标宽高 | `w, h` | 同尺寸（只转格式，不缩放） |
| 目标格式 | `AV_PIX_FMT_BGRA` | 我们要 BGRA（存 BMP） |
| 缩放算法 | `SWS_BILINEAR` | 双线性（这里不缩放，无所谓） |

**第 346~348 行**：准备输出的"数据指针数组"和"行跨度数组"。

FFmpeg 的 `sws_scale` 用**四个指针的数组**表示多平面图像：

```cpp
uint8_t* dstData[4] = {bgra数据, nullptr, nullptr, nullptr};   // BGRA 是单平面（packed）
const int dstLinesize[4] = {w * 4, 0, 0, 0};                   // 一行 w*4 字节
```

对比 YUV420P（三平面）：

```cpp
srcData[0] = Y 平面;  srcLinesize[0] = Y 行跨度
srcData[1] = U 平面;  srcLinesize[1] = U 行跨度
srcData[2] = V 平面;  srcLinesize[2] = V 行跨度
srcData[3] = nullptr
```

**第 349 行**：`sws_scale` 执行转换。

**第 351 行**：标记"第一帧已解出"，后续包不再解码。

> **老师点评（一个小瑕疵）**：第 341 行的 `sws_getContext` 在循环外声明（`SwsContext* sws = nullptr;` 在第 307 行），但**只在解码第一帧时创建一次**——这没问题。
>
> 但如果**第一帧解码失败**（`avcodec_receive_frame` 返回 `EAGAIN`，因为 H.264 解码器前几帧可能不输出），`sws` 就不会被创建，`firstFrameDecoded` 保持 `false`，测试会报"第一帧能解码出来"失败。
>
> 更健壮的做法是：**对前 N 个视频包都尝试解码，直到成功一帧**。不过对于我们的 I 帧开头的码流，通常第一个包就能解出来，所以现在能过。

### 7.9.7 第 356~366 行：音频包统计

```cpp
356          } else if (packet->stream_index == audioIndex) {
357              ++info.audioPackets;
358              if (pts != AV_NOPTS_VALUE) {
359                  info.maxAudioMs = std::max(info.maxAudioMs, pts);
360                  if (pts < 0)
361                      info.sawNegativePts = true;
362              }
363          }
364
365          av_packet_unref(packet);
366      }
```

**第 365 行极其重要**：`av_packet_unref(packet)`。

`av_read_frame` 会把数据填进你传进去的 packet（内部 `av_packet_unref` 再 `ref`）。**如果你不 `unref`，数据引用就一直累积，内存泄漏。**

**这是 `av_read_frame` 循环的标准写法**（第 3.3 节讲过 `unref` 和 `free` 的区别）：

```cpp
AVPacket* packet = av_packet_alloc();
while (av_read_frame(ctx, packet) >= 0) {
    ... 处理 ...
    av_packet_unref(packet);      // ★ 归还数据，对象复用
}
av_packet_free(&packet);          // 最后释放对象
```

### 7.9.8 第 368~377 行：清理

```cpp
368      av_packet_free(&packet);
369      if (sws)
370          sws_freeContext(sws);
371      if (decoded)
372          av_frame_free(&decoded);
373      if (decoder)
374          avcodec_free_context(&decoder);
375
376      avformat_close_input(&ctx);
377      info.ok = true;
378      return info;
379  }
```

**标准的"逆序释放"**：packet → sws → frame → decoder → format context。

**每一个都带 `if (xxx)` 判空**——因为 FFmpeg 的 `xxx_free(&p)` 系列对 `nullptr` 是安全的（内部会检查），**显式判空在这里更多是为了代码清晰**，但也是好习惯。

**第 378 行**：走到这里说明一切顺利，`info.ok = true`。

> **老师点评**：同学们注意 `probeFlv` 的**错误处理风格**：任何一步失败，就**立刻返回当前的 `info`**（其中 `ok` 还是 `false`）。
>
> 这叫 **"早返回 + 默认值"** 模式：**默认假设失败，只有全部走通才标记成功。** 这样**绝不会出现"出错了但 ok 还是 true"**的假阳性。
>
> **反过来的写法（默认 true，出错时设 false）容易漏掉某个分支，导致测试假通过。** 同学们写检查函数时请用前者。

## 7.10 第 381~453 行：`main()` 与 用例 1

```cpp
381  int main()
382  {
383  #if defined(_WIN32)
384      std::system("chcp 65001 > nul");
385  #endif
386
387      static char stdoutBuffer[8192];
388      std::setvbuf(stdout, stdoutBuffer, _IOLBF, sizeof(stdoutBuffer));
389
390      std::printf("================================================\n");
391      std::printf("  模块 8：封装与多路分发测试\n");
392      std::printf("================================================\n");
393      std::printf("  没有真服务器也能测：把 rtmp:// 换成本地文件名，\n");
394      std::printf("  同一个 StreamOutput 会写成 .flv，再读回来验证。\n");
395      std::printf("================================================\n");
396
397      std::vector<std::string> logs;
398      const auto log = [&logs](const std::string& message) {
399          logs.push_back(message);
400          std::printf("  [输出] %s\n", message.c_str());
401      };
```

### 第 383~385 行：`chcp 65001`

```cpp
std::system("chcp 65001 > nul");
```

**`chcp 65001`** 是 Windows 命令行命令，把控制台的**代码页（code page）切成 UTF-8**（65001 = UTF-8）。

为什么需要？因为：

- 我们的源文件是 **UTF-8**（CMakeLists 里有 `add_compile_options(/utf-8)`）；
- 但 **Windows 控制台默认是 GBK（代码页 936）**；
- 所以中文字符串打印出来是**乱码**。

`> nul` 是把 `chcp` 命令自己的输出（"Active code page: 65001"）丢掉。

**用 `#if defined(_WIN32)` 包起来**，因为 Linux/macOS 上没有 `chcp` 命令。

> **老师补充**：这是 Windows 中文开发的经典三件套之一：
> 1. 源文件存成 UTF-8（最好带 BOM，MSVC 才认）；
> 2. 编译选项加 `/utf-8`（告诉 MSVC "我的源码是 UTF-8"）；
> 3. 运行时 `chcp 65001`（告诉控制台"按 UTF-8 显示"）。
>
> **三者缺一，中文就是乱码。**

### 第 387~388 行：`setvbuf` 行缓冲

```cpp
static char stdoutBuffer[8192];
std::setvbuf(stdout, stdoutBuffer, _IOLBF, sizeof(stdoutBuffer));
```

**`setvbuf`** 设置 stdout 的缓冲模式：

| 模式 | 含义 |
|---|---|
| `_IONBF` | 无缓冲（每次 printf 都立刻写） |
| `_IOLBF` | **行缓冲**（遇到 `\n` 就刷新） |
| `_IOFBF` | 全缓冲（缓冲区满才刷新） |

**为什么需要这一行？** 因为 Windows 上 stdout **重定向到管道/文件时是全缓冲的**。如果程序崩溃，或者你在 IDE 里看输出，**可能什么都看不到**（数据在缓冲区里没刷出来）。

设成行缓冲后，每打印一行（我们的 printf 都以 `\n` 结尾）就刷新，输出及时可见。

**`static char stdoutBuffer[8192]` 必须是 static 的**——因为 `setvbuf` 只是**记住这个指针**，不拷贝缓冲区。如果是栈上的局部数组，`main` 返回后指针就失效了（虽然程序也快退出了，但这是未定义行为）。

### 第 397~401 行：日志回调 —— 一个 lambda 同时做两件事

```cpp
std::vector<std::string> logs;
const auto log = [&logs](const std::string& message) {
    logs.push_back(message);                            // ① 收集起来
    std::printf("  [输出] %s\n", message.c_str());      // ② 同时打印
};
```

**这就是第 4.2 节讲的"`LogCallback` 的价值"**：

- **打印**：让人能实时看到发生了什么；
- **收集**：让代码能自动校验日志内容（`logs.clear()` / 检查 `logs`）。

> **老师点评（第三个值得改进的地方）**：同学们注意，`logs` 这个 vector 在测试里**只被 `clear()` 和 `push_back()`，从来没有被检查过**！
>
> 这说明作者**本来打算**验证日志内容（比如"日志里应该出现'已连接'"），但**最终没写那条断言**。
>
> 这是"未完成的测试意图"，是个小遗憾。同学们可以把它补完（第 8 章实验 6 会带你做）。

### 第 403~453 行：用例 1 —— 单路输出到本地 flv

```cpp
403      std::printf("\n--- 1. 单路输出到本地 flv ---\n");
404      {
405          const std::vector<StreamTarget> targets = {{"local", "output_single.flv"}};
406
407          const auto start = std::chrono::steady_clock::now();
408          const RunResult run = runOnce(targets, 60, 2.0, log);
409          const long long ms = elapsedMs(start);
410
411          check(run.ok, "推流输出建立成功");
412          std::printf("  写入 %d 个视频包、%d 个音频包，耗时 %lld 毫秒\n",
413                      run.videoPackets, run.audioPackets, ms);
414
415          check(!run.targets.empty() && run.targets[0].connected, "目标保持连接");
416          check(!run.targets.empty() && run.targets[0].packetsSent > 0,
417                "统计到了写出的包数");
418
419          std::printf("\n  读回 output_single.flv：\n");
420          const FlvInfo info = probeFlv("output_single.flv");
421          check(info.ok, "文件能被 avformat 正常打开");
422
423          if (info.ok) {
424              std::printf("    流数量 %d（视频 %d / 音频 %d）\n", info.streamCount,
425                          info.hasVideo ? 1 : 0, info.hasAudio ? 1 : 0);
426              std::printf("    视频 %d 包，最后一包 pts=%lld ms\n", info.videoPackets,
427                          static_cast<long long>(info.maxVideoMs));
428              std::printf("    音频 %d 包，最后一包 pts=%lld ms\n", info.audioPackets,
429                          static_cast<long long>(info.maxAudioMs));
430
431              check(info.streamCount == 2, "容器里有两条流");
432              check(info.hasVideo && info.hasAudio, "视频流和音频流都在");
433              check(info.videoPackets >= 55, "视频包数量对得上（60 帧）");
434              check(info.audioPackets >= 90, "音频包数量对得上（2 秒约 94 包）");
435
436              check(info.maxVideoMs >= 1800 && info.maxVideoMs <= 2100,
437                    "视频时间轴落在 2 秒附近（时间基换算对了）");
438              check(info.maxAudioMs >= 1800 && info.maxAudioMs <= 2200,
439                    "音频时间轴落在 2 秒附近（重采样和偏移都没跑偏）");
440
441              check(!info.sawNegativePts,
442                    "没有任何负时间戳（AAC 的 priming 被归零了）");
443
444              if (info.firstFrameDecoded) {
445                  check(info.firstFrame.width == 640 && info.firstFrame.height == 360,
446                        "读回来的第一帧尺寸正确");
447                  check(saveBmp("output_first_frame.bmp", info.firstFrame),
448                        "第一帧已保存为 output_first_frame.bmp");
449              } else {
450                  check(false, "第一帧能解码出来");
451              }
452          }
453      }
```

#### 第 405 行：`{{"local", "output_single.flv"}}`

**这是一个"聚合初始化"（aggregate initialization）的嵌套写法**：

```cpp
const std::vector<StreamTarget> targets = {{"local", "output_single.flv"}};
                                          ↑ ↑                          ↑
                                          │ └─ StreamTarget{"local", "output_single.flv"}
                                          └─── vector 的初始化列表
```

`StreamTarget` 是个简单的 POD 结构体（`struct StreamTarget { std::string platform; std::string url; };`），所以可以用 `{"local", "output_single.flv"}` 直接初始化，不需要写构造函数。

#### 第 407~409 行：计时

```cpp
const auto start = std::chrono::steady_clock::now();
const RunResult run = runOnce(targets, 60, 2.0, log);
const long long ms = elapsedMs(start);
```

测的是**"编码 60 帧 + 封装 + 写文件"的总耗时**。

#### 第 412~413 行：`%lld` 与 `static_cast<long long>`

```cpp
std::printf("  写入 %d 个视频包、%d 个音频包，耗时 %lld 毫秒\n",
            run.videoPackets, run.audioPackets, ms);
```

**为什么后面几处要 `static_cast<long long>(...)` 而这里不用？**

因为 `printf` 是**变参函数**，编译器**不会**帮你做类型检查或转换。你必须保证：

| 格式符 | 对应的实参类型 |
|---|---|
| `%d` | `int` |
| `%lld` | `long long` |
| `%zu` | `size_t` |

- 第 413 行的 `ms` 本来就是 `long long` → 直接配 `%lld` ✅
- 第 427 行的 `info.maxVideoMs` 是 `int64_t`（在 MSVC 上 = `long long`，在 GCC 上可能是 `long`）→ **必须 cast 成 `long long`** 才能配 `%lld` ✅

> **老师提醒**：**`int64_t` 在不同平台上的底层类型不同**（MSVC 是 `long long`，Linux GCC x64 上是 `long`）。直接配 `%lld` 在 Linux 上会有编译警告甚至错误。
>
> **两个解决办法**：
> ① 像代码里这样 `static_cast<long long>(x)` 配 `%lld`（**代码里用的这种，可移植**）✅
> ② 用 `<cinttypes>` 里的 `PRId64` 宏：`std::printf("%" PRId64, x);`
>
> **第 ② 种更标准**，但代码写起来啰嗦。同学们至少要知道第 ① 种是安全的。

#### 第 431~434 行：数量断言

```cpp
check(info.streamCount == 2, "容器里有两条流");
check(info.hasVideo && info.hasAudio, "视频流和音频流都在");
check(info.videoPackets >= 55, "视频包数量对得上（60 帧）");
check(info.audioPackets >= 90, "音频包数量对得上（2 秒约 94 包）");
```

**为什么是 `>= 55` 而不是 `== 60`？**

因为 **x264 有编码延迟**——你送进去 60 帧，它在 flush 之前可能只吐出 55~58 个包（取决于 `preset`、`tune`、B 帧设置）。**用 `>=` 而不是 `==`，是为了让测试对编码器内部行为有容错。**

**这是测试设计的重要原则**：

> **断言应该检查"我关心的性质"，而不是"某个具体的实现细节"。**
>
> 我们关心的是"**60 帧视频基本都被写进去了**"（`>= 55`），而不是"**x264 恰好吐出 60 个包**"。后者只要换个 `preset` 就会失败，是**脆弱测试（brittle test）**。

**音频包的 90 是怎么算的？**

```
2 秒 × 48000 Hz = 96000 个采样
AAC 帧长 = 1024 采样
96000 / 1024 ≈ 93.75  → 约 94 包
```

断言写 `>= 90`，留了 4 包的余量（编码器延迟、flush 行为）。

#### 第 436~439 行：★ 时间戳断言（本课最重要的一组）

```cpp
check(info.maxVideoMs >= 1800 && info.maxVideoMs <= 2100,
      "视频时间轴落在 2 秒附近（时间基换算对了）");
check(info.maxAudioMs >= 1800 && info.maxAudioMs <= 2200,
      "音频时间轴落在 2 秒附近（重采样和偏移都没跑偏）");
```

**这两条断言验证的东西**：

1. **时间基换算对了**（`{1,30}` → `{1,1000}`）；
2. **音频起点偏移对了**（AAC priming 归零）；
3. **音视频时间轴对齐了**（都在 2 秒附近）。

**为什么是 1800~2100 而不是 1950~2050？**

因为最后一帧的时间戳 = `(N-1) × 33.33 ms`，其中 N 是实际写出的帧数。如果只写出 55 帧，最后一包是 `54 × 33.33 = 1800 ms`；如果写出 60 帧，是 `59 × 33.33 = 1967 ms`。所以下界 1800、上界 2100 **覆盖了 54~63 帧的情况**。

**音频的上界放宽到 2200**，是因为 AAC 的最后一包可能因为 flush 而略超出 2 秒。

> **老师点评**：**放宽范围但保留语义**——这是数值断言的艺术。
>
> 范围太窄 → 脆弱测试，换个编码器版本就挂；
> 范围太宽 → 等于没测（比如写成 `> 0 && < 100000`）。
>
> **正确的做法是：算出理论上界和下界，然后留 5%~10% 的工程余量。**

#### 第 441~442 行：负时间戳断言

```cpp
check(!info.sawNegativePts, "没有任何负时间戳（AAC 的 priming 被归零了）");
```

这就是 3.7 节的验证。**如果 `StreamOutput` 里没做音频偏移归零，这条会失败。**

#### 第 444~451 行：存一张图

```cpp
if (info.firstFrameDecoded) {
    check(info.firstFrame.width == 640 && info.firstFrame.height == 360,
          "读回来的第一帧尺寸正确");
    check(saveBmp("output_first_frame.bmp", info.firstFrame),
          "第一帧已保存为 output_first_frame.bmp");
} else {
    check(false, "第一帧能解码出来");
}
```

**这就是第 0 课讲的"存一张图让人看"的测试。**

**为什么必须存图？** 因为有些问题**断言测不出来**：

| 问题 | 断言能测吗 |
|---|---|
| 包数对不对 | ✅ 能 |
| 时间戳对不对 | ✅ 能 |
| **颜色对不对（RGB/BGR 反了）** | ❌ **测不出来** |
| **图像上下颠倒** | ❌ **测不出来** |
| **画面全是绿的（YUV 平面串了）** | ❌ **测不出来** |
| **画面糊了（码率太低）** | ❌ **测不出来** |

**这些只能靠人眼。** 所以测试把第一帧存成 BMP，你跑完测试打开 `output_first_frame.bmp` 看一眼——**是彩条吗？有个黑方块吗？颜色对吗？**

> 第 4 课（屏幕采集）、第 5 课（摄像头）的测试都用了同样的手法。**这是本项目的一条"测试纪律"，同学们做自己的项目时可以照搬。**

## 7.11 第 455~488 行：用例 2 —— 多路分发

```cpp
455      std::printf("\n--- 2. 多路分发：一次编码写两路 ---\n");
456      {
457          const std::vector<StreamTarget> targets = {{"A", "output_a.flv"},
458                                                     {"B", "output_b.flv"}};
459
460          logs.clear();
461          const RunResult run = runOnce(targets, 30, 1.0, log);
462
463          check(run.ok, "两路都建立成功");
464          check(run.targets.size() == 2, "两个目标都在列表里");
465          check(run.targets.size() == 2 && run.targets[0].connected
466                    && run.targets[1].connected,
467                "两条连接都处于连接状态");
468
469          if (run.targets.size() == 2) {
470              std::printf("    A: %lld 包 / %lld 字节\n",
471                          static_cast<long long>(run.targets[0].packetsSent),
472                          static_cast<long long>(run.targets[0].bytesSent));
473              std::printf("    B: %lld 包 / %lld 字节\n",
474                          static_cast<long long>(run.targets[1].packetsSent),
475                          static_cast<long long>(run.targets[1].bytesSent));
476
477              check(run.targets[0].packetsSent == run.targets[1].packetsSent,
478                    "两路收到的包数一模一样（同一份数据分发出去）");
479          }
480
481          const FlvInfo a = probeFlv("output_a.flv");
482          const FlvInfo b = probeFlv("output_b.flv");
483
484          check(a.ok && b.ok, "两个文件都能正常打开");
485          check(a.streamCount == 2 && b.streamCount == 2, "两个文件都是双流");
486          check(a.videoPackets == b.videoPackets,
487                "两个文件的视频包数相同");
488      }
```

**这个用例验证"多目标分发"的核心承诺**：

| 断言 | 验证什么 |
|---|---|
| `targets.size() == 2` | 两路都登记了 |
| 两路都 `connected` | 两路都建起来了 |
| **`packetsSent` 相等** | **★ 同一份数据，写进了两个地方** |
| 两个文件都能打开、都是双流 | 两路都写成功了 |
| **`a.videoPackets == b.videoPackets`** | **★ 两个文件内容一致（包数层面）** |

### 第 477 行的断言为什么这么有说服力？

```cpp
check(run.targets[0].packetsSent == run.targets[1].packetsSent, ...);
```

**这一条同时验证了三件事**：

1. **分发逻辑正确**——循环遍历了所有目标；
2. **没有目标被漏掉**——两路的计数完全一致；
3. **`av_packet_clone` 没有失败**——如果某次 clone 失败，那一路的计数就会少。

**而且它验证的是"从文件读回来的包数"**（第 486 行），不是"我以为写了多少"——**这是真正的端到端验证**。

> **老师点评**：同学们注意，**测试要验证"结果"，不要只验证"过程"**。
>
> 弱测试：`check(output.connectedCount() == 2, "两路都连上了");`（只看了内部状态）
> 强测试：**把两个文件都读回来，逐项对比内容。**（看了实际产出）
>
> **内部状态对了不代表产出对了**——比如两路都"连上了"，但因为 `stream_index` 写错，文件里的数据全乱了，弱测试照样通过。

### 第 460 行：`logs.clear()`

清空日志，让这个用例的日志独立可查（虽然最终没检查 `logs` 的内容，见 7.10 节的点评）。

## 7.12 第 490~517 行：用例 3 —— 断线隔离

```cpp
490      std::printf("\n--- 3. 断线隔离：一路死掉，另一路照常 ---\n");
491      std::printf("  第二个目标指向 127.0.0.1:1（没人监听的端口），\n");
492      std::printf("  预期：它连不上，但第一路必须一点不受影响。\n");
493      {
494          const std::vector<StreamTarget> targets = {
495              {"live", "output_isolated.flv"},
496              {"dead", "rtmp://127.0.0.1:1/live/nothing"}};
497
498          logs.clear();
499          const RunResult run = runOnce(targets, 30, 1.0, log);
500
501          check(run.ok, "整体仍算成功（有一路可用就算成功）");
502          check(run.targets.size() == 2, "两个目标都在列表里");
503
504          if (run.targets.size() == 2) {
505              check(run.targets[0].connected, "文件那一路正常连接");
506              check(!run.targets[1].connected, "rtmp 那一路被判定为断开");
507              check(!run.targets[1].lastError.empty(), "断开的理由被记录下来了");
508              std::printf("    dead 路的原因: %s\n", run.targets[1].lastError.c_str());
509
510              check(run.targets[0].packetsSent > 0, "好的那一路照常发出了包");
511              check(run.targets[1].packetsSent == 0, "坏的那一路一个包都没发出去");
512          }
513
514          const FlvInfo info = probeFlv("output_isolated.flv");
515          check(info.ok && info.streamCount == 2, "隔离之后文件依然完整");
516          check(info.videoPackets > 0 && info.audioPackets > 0, "音视频都写进去了");
517      }
```

**这是本课设计的精华——用一个没人监听的端口，验证"断线隔离"这件本该很难测的事。**

### 为什么选 `127.0.0.1:1`？

| 选项 | 问题 |
|---|---|
| `rtmp://192.0.2.1/live/x`（TEST-NET 保留地址） | 会等 TCP 超时，**几十秒** |
| `rtmp://不存在的域名/live/x` | 要先 DNS 解析失败，**慢且不确定** |
| **端口 1** | **本地回环，连接立刻被拒绝（RST），毫秒级返回** ✅ |
| 端口 65535 | 也可能没人听，但某些系统上有服务 |

**端口 1 是"系统端口"（0~1023 是特权端口），正常机器绝不会有服务监听。** 而且目标是 `127.0.0.1`（本机回环），**TCP 握手不用出网卡，操作系统立刻回 RST**。

**所以 `avio_open2` 会立刻返回 `-ECONNREFUSED`（Connection refused）。**

> **老师点评**：这个技巧叫 **"用确定失败的输入测试失败路径"**。
>
> 测试"错误处理"向来是难点，因为你很难**可靠地制造一个错误**。这里作者找到了一个**100% 确定、0 延迟、无副作用**的失败点——**了不起的设计**。
>
> 同学们以后写测试时，遇到"这个错误分支没法测"，先想想：**有没有一个输入，是必然触发这个错误的？**

### 六条断言分别验证什么

| 断言 | 验证的隔离承诺 |
|---|---|
| `run.ok == true` | **"部分成功"**：一路挂了，整体仍算成功 |
| `targets[0].connected == true` | 好路没被"株连" |
| `targets[1].connected == false` | 坏路被正确标记 |
| `!targets[1].lastError.empty()` | 失败原因被记录（**可诊断**） |
| `targets[0].packetsSent > 0` | **好路照常工作**（这是最关键的一条） |
| `targets[1].packetsSent == 0` | 坏路一个包都没发（没有"假成功"） |

**最后两条组合起来，就是"隔离"的完整证明**：

```
好的那一路：正常发包 ✓
坏的那一路：0 个包，且被标记断开 ✓
       → 互不干扰 ✓
```

### 第 514~516 行：好路的文件必须依然完好

```cpp
const FlvInfo info = probeFlv("output_isolated.flv");
check(info.ok && info.streamCount == 2, "隔离之后文件依然完整");
check(info.videoPackets > 0 && info.audioPackets > 0, "音视频都写进去了");
```

**这排除了一种很隐蔽的失败模式**：隔离逻辑写对了，但**因为某个共享状态被污染**，导致好路写出来的文件也坏了。

比如：如果 `writePacket` 在坏路失败时**错误地 return 了**（而不是 `continue`），那么坏路之后的包**好路也收不到**——文件会缺一大截。这条断言能抓住它。

## 7.13 第 519~545 行：用例 4 —— 重连接口

```cpp
519      std::printf("\n--- 4. 重连接口 ---\n");
520      {
521          StreamOutput output;
522          VideoEncoderConfig config;
523          config.width = 640;
524          config.height = 360;
525          config.fps = 30;
526
527          VideoEncoder encoder;
528          encoder.open(config, nullptr);
529
530          const std::vector<StreamTarget> targets = {
531              {"dead", "rtmp://127.0.0.1:1/live/nothing"}};
532
533          const bool opened =
534              output.open(targets, encoder.codecParameters(), nullptr, nullptr);
535
536          check(!opened, "全部目标都连不上时，open() 返回失败");
537          check(output.connectedCount() == 0, "连接数为 0");
538          std::printf("  open() 失败原因: %s\n", output.lastError().c_str());
539
540          const auto start = std::chrono::steady_clock::now();
541          const int reconnected = output.reconnectDisconnected();
542          const long long ms = elapsedMs(start);
543
544          check(reconnected == 0, "对着死地址重连，返回 0");
545          std::printf("  重连尝试耗时 %lld 毫秒（有超时保护，不会无限等）\n", ms);
546
547          encoder.close();
548      }
```

**这个用例有点特殊——它测的是"全失败"的情况，而且不写任何文件。**

### 第 534 行：三个 `nullptr`

```cpp
output.open(targets, encoder.codecParameters(), nullptr, nullptr);
//                                              ↑         ↑
//                                          音频参数为空   日志为空
```

**这是唯一一个"只有视频、没有音频"的用例**（第 3 参数 = `nullptr`）。

它顺便验证了：**`StreamOutput` 在"没有音频参数"时能正常工作**（`openTarget` 里 `if (audioParams)` 分支被跳过，不建音频流）。

**第 4 个参数 `log = nullptr`** —— 验证了**所有日志调用点都做了判空**（第 5.6 节讲过 `say` lambda 里的 `if (impl_->log)`）。

> **老师点评**：**"传 nullptr 测一遍"是被严重低估的测试手段。**
>
> 很多崩溃 bug 都是"某个可选参数为空时没判空"导致的。而这个用例**强制覆盖了这条路径**——如果哪天有人在 `StreamOutput` 里加了一句 `impl_->log("xxx")` 而忘了判空，**这个用例会立刻崩给你看**。

### 第 536~537 行：验证"全军覆没才算失败"

```cpp
check(!opened, "全部目标都连不上时，open() 返回失败");
check(output.connectedCount() == 0, "连接数为 0");
```

这跟用例 3 形成**对照**：

| 场景 | `open()` 返回 | `connectedCount()` |
|---|---|---|
| 2 路中连上 1 路（用例 3） | **true** | 1 |
| 1 路都没连上（用例 4） | **false** | 0 |

**两个方向都测了，语义才完整。**

### 第 540~545 行：验证重连"不卡死"

```cpp
const auto start = std::chrono::steady_clock::now();
const int reconnected = output.reconnectDisconnected();
const long long ms = elapsedMs(start);

check(reconnected == 0, "对着死地址重连，返回 0");
std::printf("  重连尝试耗时 %lld 毫秒（有超时保护，不会无限等）\n", ms);
```

**这里测的不是"重连成功"，而是"重连不会卡死"。**

因为 `rw_timeout = 5 秒`（第 5.1 节），所以即使对着一个死地址重连，也**最多等 5 秒**就会返回。

**这一条的价值**：它把"超时保护"这个**非功能性需求**（你很难用断言说"它必须不卡"）转化成了**可观测的数字**（耗时多少毫秒）。

> **老师点评**：非功能性需求（性能、不卡死、不泄漏）**不是不能测，而是要找到可观测的代理指标**。
>
> - "不卡死" → 测**耗时上界**；
> - "不泄漏" → 测**内存占用**（Windows 上可以用 `GetProcessMemoryInfo`）；
> - "不丢帧" → 测**输出的帧数**。
>
> 这个测试打印耗时让我们**人眼判断**，更严格的做法是加一条断言：
> ```cpp
> check(ms < 6000, "重连有超时保护，不会无限等待");
> ```
> 同学们第 8 章实验 5 会做这个。

## 7.14 第 547~559 行：汇总与退出码

```cpp
547      std::printf("\n================================================\n");
548      std::printf("  通过 %d 项，失败 %d 项\n", g_passed, g_failed);
549      std::printf("================================================\n");
550      std::printf("  output_single.flv / output_a.flv / output_b.flv\n");
551      std::printf("  都可以直接用播放器打开，确认画面和声音。\n");
552      std::printf("================================================\n");
553      std::fflush(stdout);
554
555      return g_failed == 0 ? 0 : 1;
556  }
```

**第 555 行是 CI（持续集成）的关键**：

```cpp
return g_failed == 0 ? 0 : 1;
```

**进程的退出码（exit code）**：

| 退出码 | 含义 |
|---|---|
| `0` | 成功 |
| 非 0 | 失败 |

CI 系统（GitHub Actions、Jenkins、GitLab CI）**就是靠退出码判断测试过没过**。所以：

```bat
test_output.exe
if errorlevel 1 (
    echo 测试失败！
    exit /b 1
)
```

**如果测试程序永远返回 0，CI 就形同虚设。**

**第 553 行**：`std::fflush(stdout)` —— 确保退出前把所有输出刷出来（第 7.10 节讲过缓冲模式）。

## 7.15 测试覆盖矩阵

同学们，我们把四个用例的覆盖情况汇总一下：

| 被测能力 | 用例 1 | 用例 2 | 用例 3 | 用例 4 |
|---|---|---|---|---|
| 单路输出 | ✅ | ✅ | ✅ | — |
| 多路分发（一份数据写多处） | — | ✅ | — | — |
| 断线隔离（一路挂不影响其它） | — | — | ✅ | — |
| 部分成功（connected > 0 即成功） | — | — | ✅ | — |
| 全失败（connected == 0 即失败） | — | — | — | ✅ |
| 只有视频（无音频流） | — | — | — | ✅ |
| `log = nullptr` 不崩 | — | — | — | ✅ |
| 时间基换算 | ✅ | — | — | — |
| AAC priming 归零 | ✅ | — | — | — |
| 内容可解码（存图） | ✅ | — | — | — |
| 重连接口 | — | — | — | ✅ |
| 超时保护（不卡死） | — | — | — | ✅ |
| 统计正确（packetsSent/bytesSent） | ✅ | ✅ | ✅ | — |

**13 项能力，4 个用例，26 条断言（用例 1 有 12 条，用例 2 有 7 条，用例 3 有 8 条，用例 4 有 3 条）。**

---

# 第 8 章 动手环节：本地文件 + rtmp 双目标实验

光看不练假把式。这一章我们真的跑起来，然后做六个改造实验。

## 8.1 编译与运行

### 第一步：重新配置 CMake（因为改了 CMakeLists）

```bat
cd C:\...\Streaming-tools
cmake -S . -B build -G "Visual Studio 17 2022" -A x64
```

或者直接跑项目自带的一键脚本：

```bat
build.bat
```

**看到这几行就说明新模块被识别了**：

```
-- 正在配置 core_output ...
-- 正在配置 test_output ...
```

### 第二步：只编译测试程序

```bat
cmake --build build --config Release --target test_output
```

### 第三步：运行

```bat
cd build
test_output.exe
```

### 期望看到的输出

```
================================================
  模块 8：封装与多路分发测试
================================================
  没有真服务器也能测：把 rtmp:// 换成本地文件名，
  同一个 StreamOutput 会写成 .flv，再读回来验证。
================================================

--- 1. 单路输出到本地 flv ---
  [输出] [local] 已连接 output_single.flv
  [输出] 推流输出就绪: 1/1 路可用
  [输出] 音频起点偏移 -1024（AAC 编码延迟），已归零
  [通过] 推流输出建立成功
  写入 58 个视频包、94 个音频包，耗时 1234 毫秒
  [通过] 目标保持连接
  [通过] 统计到了写出的包数

  读回 output_single.flv：
  [通过] 文件能被 avformat 正常打开
    流数量 2（视频 1 / 音频 1）
    视频 58 包，最后一包 pts=1900 ms
    音频 94 包，最后一包 pts=1973 ms
  [通过] 容器里有两条流
  [通过] 视频流和音频流都在
  [通过] 视频包数量对得上（60 帧）
  [通过] 音频包数量对得上（2 秒约 94 包）
  [通过] 视频时间轴落在 2 秒附近（时间基换算对了）
  [通过] 音频时间轴落在 2 秒附近（重采样和偏移都没跑偏）
  [通过] 没有任何负时间戳（AAC 的 priming 被归零了）
  [通过] 读回来的第一帧尺寸正确
  [通过] 第一帧已保存为 output_first_frame.bmp

--- 2. 多路分发：一次编码写两路 ---
  [输出] [A] 已连接 output_a.flv
  [输出] [B] 已连接 output_b.flv
  [输出] 推流输出就绪: 2/2 路可用
  [输出] 音频起点偏移 -1024（AAC 编码延迟），已归零
  [通过] 两路都建立成功
  [通过] 两个目标都在列表里
  [通过] 两条连接都处于连接状态
    A: 128 包 / 245678 字节
    B: 128 包 / 245678 字节
  [通过] 两路收到的包数一模一样（同一份数据分发出去）
  [通过] 两个文件都能正常打开
  [通过] 两个文件都是双流
  [通过] 两个文件的视频包数相同

--- 3. 断线隔离：一路死掉，另一路照常 ---
  第二个目标指向 127.0.0.1:1（没人监听的端口），
  预期：它连不上，但第一路必须一点不受影响。
  [输出] [live] 已连接 output_isolated.flv
  [输出] [dead] 连接失败: Connection refused
  [输出] 推流输出就绪: 1/2 路可用
  [通过] 整体仍算成功（有一路可用就算成功）
  [通过] 两个目标都在列表里
  [通过] 文件那一路正常连接
  [通过] rtmp 那一路被判定为断开
  [通过] 断开的理由被记录下来了
    dead 路的原因: 连接失败: Connection refused
  [通过] 好的那一路照常发出了包
  [通过] 坏的那一路一个包都没发出去
  [通过] 隔离之后文件依然完整
  [通过] 音视频都写进去了

--- 4. 重连接口 ---
  open() 失败原因: 所有推流目标都没连上
  [通过] 全部目标都连不上时，open() 返回失败
  [通过] 连接数为 0
  [通过] 对着死地址重连，返回 0
  重连尝试耗时 15 毫秒（有超时保护，不会无限等）

================================================
  通过 30 项，失败 0 项
================================================
  output_single.flv / output_a.flv / output_b.flv
  都可以直接用播放器打开，确认画面和声音。
================================================
```

### 第四步：★ 用眼睛验证（这一步不能省！）

打开 `build` 目录，你会看到：

```
output_single.flv       ← 2 秒，60 帧，双流
output_a.flv            ← 1 秒，30 帧，双流
output_b.flv            ← 1 秒，30 帧，双流（内容跟 a 一模一样）
output_isolated.flv     ← 1 秒，30 帧，双流
output_first_frame.bmp  ← ★ 打开它！
```

**做三件事**：

#### ① 双击 `output_first_frame.bmp`

应该看到：**8 条彩条（白黄青绿品红蓝黑）+ 中间偏左一个黑方块。**

检查表：

| 检查项 | 对了 | 错了说明什么 |
|---|---|---|
| 8 条彩条 | ✅ | 编码/解码链路通 |
| 颜色是"白黄青绿品红蓝黑" | ✅ | BGRA 顺序没搞反 |
| 方块在中间偏左（不是上下颠倒的偏右） | ✅ | 行序正确 |
| 图像不是全绿/全灰 | ✅ | YUV 转换正确 |

#### ② 用播放器打开 `output_single.flv`

推荐用 **VLC** 或 **PotPlayer**（Windows 自带的"电影和电视"也能放 flv）。

**重点看**：那个黑方块是不是**匀速从左往右扫**（约 2 秒扫一遍）。

- 匀速 → 帧率、时间戳、帧序全对；
- 一跳一跳 → 丢帧了；
- 不动 → **时间戳没生效**；
- 倒着走 → 帧序反了。

#### ③ 用 ffprobe 看细节

```bat
ffprobe -v error -show_format -show_streams output_single.flv
```

**重点看这几行**：

```
[STREAM]
codec_name=h264              ← 视频编码
width=640
height=360
r_frame_rate=30/1
avg_frame_rate=30/1
[/STREAM]
[STREAM]
codec_name=aac               ← 音频编码
sample_rate=48000
channels=2
[/STREAM]
[FORMAT]
duration=2.020000            ← 总时长
nb_streams=2                 ← 两条流
format_name=flv              ← 容器格式
[/FORMAT]
```

**如果 `duration` 是 `N/A` 或者 0** → 说明 `av_write_trailer` 没写成功（第 1.4 节）。

**如果只看到一条流** → 说明音频流没建起来（检查 `audioParams` 是不是传了 `nullptr`）。

还可以只看视频的包列表：

```bat
ffprobe -v error -select_streams v -show_packets -show_entries packet=pts,dts,flags,size output_single.flv
```

输出类似：

```
[PACKET]
pts=0
dts=0
size=28341
flags=K_          ← K = 关键帧
[/PACKET]
[PACKET]
pts=33
dts=33
size=3120
flags=__          ← 非关键帧
[/PACKET]
...
```

**检查**：pts 是不是 0, 33, 67, 100, ...（每帧 33 毫秒递增）？第一个包是不是 `K_`（关键帧）？

## 8.2 实验 1：把"第二个目标"换成真的 rtmp（需要本地服务器）

**这个实验可选**——如果你没有 RTMP 服务器，可以先跳过，或者做实验 1b。

### 实验 1a：用 nginx-rtmp 或 SRS 起本地服务器

最简单的办法是用 Docker 起一个 SRS：

```bat
docker run -p 1935:1935 -p 1985:1985 ossrs/srs:5
```

然后改测试代码里的一行：

```cpp
// 原来
const std::vector<StreamTarget> targets = {{"A", "output_a.flv"},
                                           {"B", "output_b.flv"}};

// 改成
const std::vector<StreamTarget> targets = {{"本地", "output_a.flv"},
                                           {"SRS",  "rtmp://127.0.0.1/live/livestream"}};
```

重新编译运行，**看看第 2 个目标能不能连上**。

然后用 ffplay 拉流验证：

```bat
ffplay rtmp://127.0.0.1/live/livestream
```

**你应该能看到彩条 + 移动的方块。**

### 实验 1b：没有服务器也能"假装推流"

如果没服务器，可以写成这样：

```cpp
const std::vector<StreamTarget> targets = {
    {"本地文件", "output_a.flv"},
    {"管道",     "pipe:1"}};              // ← 输出到 stdout
```

`pipe:1` 是 FFmpeg 的特殊 URL，表示"写到标准输出"。你可以把它重定向到文件：

```bat
test_output.exe > out.flv 2>nul
```

**注意**：这个实验会污染 stdout（因为测试也在往 stdout 打日志），所以实际效果不干净。**建议还是用实验 1a。**

## 8.3 实验 2：注释掉 `codec_tag = 0` 会发生什么

**动手**：打开 `StreamOutput.cpp`，把第 144 行和第 156 行注释掉：

```cpp
// videoStream->codecpar->codec_tag = 0;
// audioStream->codecpar->codec_tag = 0;
```

重新编译运行，看测试还能不能过。

**预期**：大概率**还能过**（因为 FLV muxer 会忽略 `codec_tag`，它内部用 CodecID）。

**但是**——用 ffprobe 看：

```bat
ffprobe -v error -show_entries stream=codec_tag,codec_tag_string output_single.flv
```

对比注释前后，**看看 `codec_tag_string` 变没变**。

再用一些"挑剔"的播放器（比如某些网页端的 flv.js）试试，可能就放不出来了。

> **这个实验告诉我们**：`codec_tag = 0` 是个**兼容性保险**。它不保证一定有区别，但它消除了一个"可能出问题"的维度。**这种"消除不确定性"的写法，值得养成习惯。**

## 8.4 实验 3：把时间基改成 `{1, 90000}`，看测试怎么报警

**动手**：

```cpp
// 第 64 行
constexpr AVRational kFlvTimeBase = {1, 90000};   // 原来是 {1, 1000}
```

重新编译运行。

**预期结果**：

```
[失败] 视频时间轴落在 2 秒附近（时间基换算对了）
    视频 58 包，最后一包 pts=171000 ms      ← 应该是 1900，现在是它的 90 倍
[失败] 音频时间轴落在 2 秒附近（重采样和偏移都没跑偏）
```

**而且你打开 flv 文件，会看到画面以 90 倍速快进**（或者播放器直接卡住）。

**为什么？** 因为 FLV 的 Tag 头规定时间戳是毫秒，你写了 171000，播放器以为这是 171 秒。

**这个实验让你亲眼看到"时间基错了"的后果**，比看十遍文档都直观。

**做完记得改回来！**

## 8.5 实验 4：注释掉音频偏移归零，看负时间戳

**动手**：把 `writePacket` 里的这段注释掉（第 241~246 行）：

```cpp
// if (ptsOffset != 0) {
//     if (copy->pts != AV_NOPTS_VALUE) copy->pts -= ptsOffset;
//     if (copy->dts != AV_NOPTS_VALUE) copy->dts -= ptsOffset;
// }
```

重新编译运行。

**预期**：

```
[失败] 没有任何负时间戳（AAC 的 priming 被归零了）
    音频 94 包，最后一包 pts=1973 ms
```

用 ffprobe 看音频包：

```bat
ffprobe -v error -select_streams a -show_entries packet=pts output_single.flv | head -20
```

你会看到开头几包的 pts 是**负数**：

```
pts=-1024
pts=-512
pts=0
pts=512
...
```

**这正是 AAC 的 `initial_padding`（1024 采样）留下的痕迹。**

**为什么播放器会出问题？** 因为播放器的时钟逻辑通常假设"第一包的时间戳是 0 或正数"。负时间戳会让它：

- 认为"这段音频还没到播放时刻" → 开头静音一小段；
- 或者触发 `non-monotonous DTS` 警告 → 丢包。

**做完记得改回来！**

## 8.6 实验 5：给重连加上"超时上界"断言

**动手**：在第 544 行后面加一条断言：

```cpp
check(reconnected == 0, "对着死地址重连，返回 0");
std::printf("  重连尝试耗时 %lld 毫秒（有超时保护，不会无限等）\n", ms);
check(ms < 6000, "重连有超时保护（5 秒 rw_timeout），不会无限等待");   // ← 新增
```

**然后做一个验证实验**：把 `kIoTimeout` 改成 `"30000000"`（30 秒），重新编译，看看会发生什么。

**预期**：因为 `127.0.0.1:1` 是立刻拒绝连接（不走超时），所以**耗时基本不变**。这正说明——

> **`rw_timeout` 主要防的是"半死不活"的网络（连上了但不回应），而不是"立刻拒绝"的场景。**

**想真正验证 `rw_timeout`，可以这样**：把目标改成一个**不可达的公网 IP**（比如 `rtmp://10.255.255.1/live/x`，这是个私有网段里几乎肯定不存在的地址），这时候 TCP 会真的等待超时。

```cpp
{"dead", "rtmp://10.255.255.1/live/nothing"}
```

**对比 `kIoTimeout = "5000000"` 和 `"30000000"` 两种情况下的耗时**，你会看到明显差别。

## 8.7 实验 6：补完那个"未完成的测试意图"——检查日志内容

第 7.10 节点评过：`logs` 这个 vector 被 `clear()` 和 `push_back()`，**但从来没被检查过**。

**动手**：在用例 3 后面加上：

```cpp
// 在用例 3 的 check 之后加
bool sawDisconnectLog = false;
for (const std::string& line : logs) {
    if (line.find("[dead]") != std::string::npos
        && line.find("连接失败") != std::string::npos) {
        sawDisconnectLog = true;
        break;
    }
}
check(sawDisconnectLog, "断线的原因被写进了日志（上层能感知）");
```

**这个断言验证的是"可观测性"**：光有内部状态不够，**上层必须能通过日志知道发生了什么**。

> **为什么这很重要？** 想象线上出问题了，用户跟你说"推流断了"。如果日志里什么都没有，你根本无从查起。**日志是分布式系统里唯一的"黑匣子"。**

**进阶实验**：在用例 2（两路都成功）里加一条：

```cpp
bool sawReadyLog = false;
for (const std::string& line : logs)
    if (line.find("2/2 路可用") != std::string::npos)
        sawReadyLog = true;
check(sawReadyLog, "日志里报告了 2/2 路可用");
```

## 8.8 实验 7（加分）：观察引用计数的"零拷贝"

**动手**：在 `writePacket` 的循环里加一行调试打印：

```cpp
AVPacket* copy = av_packet_clone(&packet);
std::printf("    [调试] 目标 %s: 原包 data=%p size=%d，克隆包 data=%p\n",
            target.name.c_str(), packet.data, packet.size, copy->data);
```

**预期输出**：

```
[调试] 目标 A: 原包 data=000001A2B3C4D5E0 size=3120，克隆包 data=000001A2B3C4D5E0
[调试] 目标 B: 原包 data=000001A2B3C4D5E0 size=3120，克隆包 data=000001A2B3C4D5E0
```

**看到三个地址一模一样了吗？**

**这就是第 3.2 节讲的"零拷贝"**：`clone` 出来的包，`data` 指针**指向同一块内存**。三个 `AVPacket` 结构体（原包 + 2 个克隆）共享一份 3120 字节的数据，靠 `refcount` 记账。

**验证一下内存节省**：假如推 5 路、每秒 30 包、每包 5 KB：

- 真拷贝：`30 × 5 × 5 KB = 750 KB/秒` 的 memcpy；
- 引用计数：**0 KB/秒** 的 memcpy，只有 150 次整数自增。

---

# 第 9 章 排错手册

同学们，这一章是"当你遇到问题时，回来查这里"。

## 9.1 编译期错误

### 错误 1：`error C2338: can't delete an incomplete type`

```
error C2338: static_assert failed: 'can't delete an incomplete type'
```

**原因**：PImpl 的析构函数被写成了 inline（第 4.2 节讲过）。

**检查**：

```cpp
// ✗ 错：析构函数定义在头文件里
class StreamOutput {
    ~StreamOutput() {}      // inline！Impl 还是不完整类型
    std::unique_ptr<Impl> impl_;
};

// ✓ 对：只声明，定义在 .cpp
class StreamOutput {
    ~StreamOutput();        // 只有声明
};
// StreamOutput.cpp:
StreamOutput::~StreamOutput() { close(); }
```

### 错误 2：`LNK2005: avErrorText 已经在 xxx.obj 中定义`

**原因**：`avErrorText` 没有放在匿名命名空间里（第 5.1 节讲过）。

**解决**：确认它被 `namespace { ... }` 包住。

### 错误 3：`Target "test_output" links to target "core_output" but the target was not found`

**原因**：`add_subdirectory(src/core/output)` 的位置在 `add_subdirectory(tests)` 之后（第 6.1 节讲过）。

**解决**：把它移到 `tests` 之前。

### 错误 4：找不到 FFmpeg 头文件

```
fatal error C1083: 无法打开包括文件: "libavformat/avformat.h": No such file or directory
```

**原因**：`PRIVATE ${FFMPEG_INCLUDE_DIRS}` 只给了 `core_output` 自己，而 `test_output.cpp` 也需要（第 6.2 节点评过这个隐患）。

**解决**：在 `tests/CMakeLists.txt` 里也给 `test_output` 加上 include 目录，或者把 `${FFMPEG_INCLUDE_DIRS}` 改成 `PUBLIC`（更推荐）。

## 9.2 运行期错误

### 错误 5：`open() 失败原因: 所有推流目标都没连上`

**排查步骤**：

1. **看每一路的 `lastError`**：
   ```cpp
   for (const auto& t : output.targets())
       std::printf("[%s] %s\n", t.name.c_str(), t.lastError.c_str());
   ```
2. **常见原因**：

| `lastError` 内容 | 原因 | 怎么办 |
|---|---|---|
| `创建输出上下文失败: ...` | FFmpeg 没编译 flv muxer | 换一个带 flv 的 FFmpeg（vcpkg 默认带） |
| `创建视频流失败` | 参数拷贝失败（内存不足/参数非法） | 检查编码器的 `codecParameters()` 非空且有效 |
| `连接失败: Connection refused` | **RTMP 服务器没开 / 端口错 / 推流地址错** | 检查服务器、地址、端口 |
| `连接失败: No such file or directory` | **本地文件路径不存在** | 目录要先建好！FFmpeg **不会**自动创建目录 |
| `连接失败: Permission denied` | 没写权限 | 换目录，或者用管理员权限 |
| `写文件头失败: ...` | 通常是服务器拒绝了推流（密钥错误） | 检查推流密钥 |
| `没有视频编码参数` | `codecParameters()` 返回了 nullptr | **编码器必须先 `open()` 成功** |

**第 3 条和第 4 条占 90% 的情况。**

### 错误 6：中文日志乱码

```
[???] ?? ?????
```

**原因**：第 7.10 节讲的三件套没配齐。

**解决**：确认 `main()` 里有 `std::system("chcp 65001 > nul");`，且源文件是 UTF-8，且编译选项有 `/utf-8`。

### 错误 7：程序卡死，Ctrl+C 都不响应

**原因**：**`rw_timeout` 没生效**。

**排查**：

1. 确认 `kIoTimeout` 被正确传给了 `avio_open2`；
2. 确认 `avio_open2`（**不是 `avio_open`**）——`avio_open` 不接受 options！

```cpp
// ✗ 错
avio_open(&ctx->pb, url, AVIO_FLAG_WRITE);      // 没有 options 参数，超时设不上

// ✓ 对
avio_open2(&ctx->pb, url, AVIO_FLAG_WRITE, nullptr, &options);
```

3. **真正卡死的更可能原因是 `avformat_write_header` 或 `av_interleaved_write_frame`**——它们也受 `rw_timeout` 影响，但如果服务器的 TCP 连接正常、只是不回 RTMP 响应，FFmpeg 可能一直等。这种情况需要 `AVIOInterruptCB`（中断回调）。

### 错误 8：文件能生成，但播放不了 / 只有视频没声音

**排查清单**：

| 症状 | 检查 |
|---|---|
| 完全放不了 | ① 没有 `av_write_trailer`；② `codec_tag` 问题；③ FFmpeg 版本太老 |
| **只有视频没声音** | ① `open()` 时音频参数传了 `nullptr`；② 音频包的 `stream_index` 写错了；③ `probeFlv` 里 `audioIndex` 没找到 |
| **只有声音没画面** | ① 视频流的 `codecpar` 没拷对（SPS/PPS 丢了）；② `AV_CODEC_FLAG_GLOBAL_HEADER` 没设 |
| 画面花屏 | ① YUV 转换错了（第 8 课）；② 分辨率不匹配 |
| **画面快进/慢动作** | **时间基错了**（实验 3） |
| 音画不同步 | ① 音视频时间基没统一；② AAC priming 没归零（实验 4） |

**万能诊断命令**：

```bat
ffprobe -v error -show_format -show_streams -show_packets output_single.flv > probe.txt
```

然后**仔细读 `probe.txt`**——里面什么都有。

### 错误 9：`[失败] 视频时间轴落在 2 秒附近`

**可能原因**：

1. **时间基不是 `{1,1000}`**（实验 3）；
2. **编码的帧数不对**——检查 `run.videoPackets` 是不是 0（编码器没打开？）；
3. **`pts` 根本没设**——检查 `makeTestFrame` 里 `frame->pts = index`；
4. **编码器时间基传错了**——检查 `video.timeBase()`。

### 错误 10：`[失败] 第一帧能解码出来`

**排查**：

1. **解码器没打开**——`probeFlv` 第 308 行的 `if (videoIndex >= 0)` 进去了吗？
2. **第一个视频包解码失败**——H.264 解码器前几个包可能返回 `EAGAIN`（正常），但我们**只尝试了第一个包**。可以按 7.9.6 节的点评改成"尝试前 N 个包"。
3. **`sws_getContext` 返回 nullptr**——源格式不支持。

**万能办法**：加打印。

```cpp
if (avcodec_send_packet(decoder, packet) < 0)
    std::printf("    [调试] send_packet 失败\n");
else if (avcodec_receive_frame(decoder, decoded) < 0)
    std::printf("    [调试] receive_frame 返回 EAGAIN，需要再来几个包\n");
```

### 错误 11：内存一直在涨

**排查清单**（按可能性排序）：

1. **`av_packet_clone` 之后忘了 `av_packet_free`** —— 检查 252~253 行；
2. **`av_read_frame` 之后忘了 `av_packet_unref`** —— 检查 365 行；
3. **目标断掉后资源没释放** —— 检查 261~266 行；
4. **`release()` 没被调用** —— 检查析构函数。

**验证方法**：在任务管理器里看测试程序的内存曲线，或者用 Visual Studio 的诊断工具。

## 9.3  behavioural（行为）层面的问题

### 错误 12：多路分发时某一路偶尔少几个包

**原因**：`av_packet_clone` 失败了（内存不足），被 `continue` 跳过（第 5.5.3 节）。

**这是设计上可接受的**（我们选择"丢一个包"而不是"断一路"）。但如果你发现**频繁发生**，说明：

1. **内存真的不够了**——检查有没有泄漏（错误 11）；
2. **目标太多了**——5 路以上要考虑内存压力。

### 错误 13：一个目标断了之后，再也连不回来

**原因**：`reconnectDisconnected()` 从来没被调用。

**这是正常的**——`StreamOutput` **不会自动重连**，它只是提供了接口。第 14 课（`c2b8021` 断线重连）会实现自动重连的策略（什么时候重连、重试几次、指数退避）。

**现阶段你可以手动验证**：

```cpp
// 跑到一半，人为让一个目标断开，然后：
int n = output.reconnectDisconnected();
std::printf("重连成功 %d 路\n", n);
```

---

# 第 10 章 小结、速查表、局限与思考题

## 10.1 本课一句话总结

> **我们用 442 行实现了一个"多目标 FLV 分发器"，用 559 行测试证明了它。它的核心价值有三条：① 一次编码、N 次封装（引用计数零拷贝）；② 每一路独立封装、独立记账、独立失败；③ RTMP 和本地文件走同一套抽象，所以"没有服务器也能完整测试"。**

## 10.2 十一个"必须记住"的知识点

| # | 知识点 | 一句话 |
|---|---|---|
| 1 | FLV 的时间基 | **必须是 `{1,1000}`**（Tag 头规定毫秒） |
| 2 | `stream_index` | **从目标自己的 `stream` 上读**，绝不假设是 0/1 |
| 3 | header / trailer | 头写 SPS/PPS 和元数据；尾回写 duration、冲刷缓冲 |
| 4 | `codec_tag = 0` | 让 muxer 自己挑 fourcc，消除兼容性隐患 |
| 5 | `AVFMT_NOFILE` | 开 IO 前先问"这个格式需要吗" |
| 6 | `rw_timeout` | **必须设**，单位是**微秒**，防无限阻塞 |
| 7 | `av_packet_clone` | **零拷贝**（只加引用计数），一个包写 N 处就靠它 |
| 8 | `av_interleaved_write_frame` | **会吃掉你的 packet**，所以每个目标一份 clone |
| 9 | 写失败隔离 | 记错 → 标记断开 → 释放资源 → **continue** |
| 10 | `anyOk` | **"至少一路成功"就算成功**（部分成功范式） |
| 11 | AAC priming | 第一包 pts 是负的，**统一减掉偏移归零** |

## 10.3 FFmpeg API 速查表（本课新增/重点）

### 封装输出（libavformat）

| 函数 | 作用 | 配对 |
|---|---|---|
| `avformat_alloc_output_context2(&ctx, ofmt, "flv", url)` | 建输出上下文，**显式指定格式** | `avformat_free_context` |
| `avformat_new_stream(ctx, nullptr)` | 建流（**归 ctx 所有，不能单独 free**） | 随 ctx 释放 |
| `avio_open2(&pb, url, AVIO_FLAG_WRITE, int_cb, &opts)` | 开 IO，**支持私有选项** | `avio_closep(&pb)` |
| `avformat_write_header(ctx, &opts)` | 写容器头（含全局头） | `av_write_trailer` |
| `av_interleaved_write_frame(ctx, pkt)` | 写包（**自动按 dts 交错**）；`pkt=nullptr` = 冲刷 | — |
| `av_write_trailer(ctx)` | 写容器尾 | — |
| `avio_closep(&pb)` | 关 IO **并置空** | — |
| `avformat_free_context(ctx)` | 释放（连带所有 stream 和 pb） | — |

### 解封装输入（测试里用的）

| 函数 | 作用 | 配对 |
|---|---|---|
| `avformat_open_input(&ctx, path, fmt, &opts)` | 打开文件/流 | `avformat_close_input(&ctx)` |
| `avformat_find_stream_info(ctx, nullptr)` | 探测流参数 | — |
| `av_read_frame(ctx, pkt)` | 读一个包（**自动换算到流时间基**） | `av_packet_unref(pkt)` |
| `avformat_close_input(&ctx)` | 关闭并释放 | — |

### 参数包与解码器

| 函数 | 作用 |
|---|---|
| `avcodec_parameters_alloc()` / `_free()` | 分配/释放参数包 |
| `avcodec_parameters_from_context(par, ctx)` | **编码器 → 参数包**（open2 之后） |
| `avcodec_parameters_to_context(ctx, par)` | **参数包 → 解码器上下文**（open2 之前） |
| `avcodec_parameters_copy(dst, src)` | 参数包 → 参数包（**深拷贝 extradata**） |
| `avcodec_find_decoder(codec_id)` | 找解码器 |
| `av_dict_set(&dict, key, value, 0)` / `av_dict_free(&dict)` | 私有选项字典 |

### 包管理

| 函数 | 作用 |
|---|---|
| `av_packet_alloc()` | 分配空包对象 |
| `av_packet_clone(src)` | `= alloc + ref`，**零拷贝** |
| `av_packet_ref(dst, src)` | 共享数据 + 拷字段 |
| `av_packet_unref(pkt)` | 归还数据引用 + 重置字段（**对象可复用**） |
| `av_packet_free(&pkt)` | `= unref + 释放对象 + 置空` |
| `av_packet_rescale_ts(pkt, src_tb, dst_tb)` | 换算 pts/dts/duration |

### 关键宏 / 常量

| 宏 | 含义 |
|---|---|
| `AVFMT_NOFILE` | muxer 不需要 `avio_open` |
| `AVIO_FLAG_WRITE` | 以写方式打开 |
| `AV_NOPTS_VALUE` | "时间戳不存在"（`INT64_MIN`），**算术运算前必须判** |
| `AV_PKT_FLAG_KEY` | 关键帧标志 |
| `AVMEDIA_TYPE_VIDEO` / `_AUDIO` | 流类型 |
| `AV_PIX_FMT_BGRA` | 32 位 BGRA（存 BMP） |
| `SWS_BILINEAR` | sws_scale 的双线性算法 |

## 10.4 局限与下一步（这些"缺陷"就是后面几课的需求）

同学们，老规矩——**知道自己差在哪才算本事**。

### 局限 1：没有自动重连策略

`reconnectDisconnected()` 只是个**接口**，没人调用它。真实的系统需要：

- **什么时候重连**？（检测到断开立刻？还是等 3 秒？）
- **重试几次**？（无限重试会打爆服务器）
- **退避策略**？（指数退避：1s、2s、4s、8s……）
- **重连后要不要插关键帧**？（★ **必须插**！否则新连接收到的第一个包是 P 帧，解不出来）

**这就是第 14 课（`c2b8021` 断线重连）要解决的。**

> **老师剧透一个关键点**：重连之后，`StreamOutput` 会建一个**全新的 `AVFormatContext`**，写一套全新的 header（含 SPS/PPS）。**但码流会继续从当前帧写下去**——如果当前帧是 P 帧，观众端会花屏直到下一个关键帧。
>
> **正解**：重连成功后，**立刻向编码器请求一个 IDR 帧**（`avcodec_send_frame` 前设置 `AVFrame::flags |= AV_FRAME_FLAG_KEY`，或者用 `av_opt_set(ctx->priv_data, "forced-idr", "1", 0)`）。这是第 13 课的重点。

### 局限 2：没有发送节流（pacing）

现在的 `writeVideo()` 是**来一包写一包，写多快发多快**。

在真实推流里这是**灾难**：

```
编码器 1 秒编出 30 帧（累积 1 秒的数据量）
        ↓
但我们可能在 100 毫秒内就全发出去了
        ↓
服务器瞬间收到 10 倍速的数据 → 缓冲区溢出 → 掐流
```

**正解**：**按 pts 做 pacing**——每一包 sleep 到它"该发的时刻"再发。这需要：

- 一个**单调递增的时钟**；
- 一个"当前播放位置"的概念；
- 处理"落后了怎么办"（追帧 or 丢帧）。

**这是第 11 课（`31c3f57` 三线程两队列）的一部分。**

### 局限 3：单线程，阻塞上游

`av_interleaved_write_frame` 是**阻塞**的（网络慢时可能阻塞几百毫秒）。

现在它跟编码在同一个线程上——**网络一卡，编码就停**，采集队列就爆，然后丢帧。

**正解**：**单独的发送线程 + 有界队列**（第 3 课的 `BoundedQueue` 就是为这个准备的）。

```
采集线程 → [有界队列] → 编码线程 → [有界队列] → 发送线程 → StreamOutput
                                                    ↑
                                              网络阻塞只卡这里
```

**这也是第 10 课的核心内容。**

### 局限 4：`bytesSent` 统计不准

前面 5.5.5 节点评过：它统计的是**压缩数据大小**，不含 FLV Tag 头、RTMP chunk 头、TCP/IP 头。

**更准的做法**：从 `AVIOContext` 上读：

```cpp
int64_t realBytes = avio_size(target.ctx->pb);        // 已写的总字节数（可 seek 时）
// 或者
int64_t written = target.ctx->pb->bytes_written;      // 累计写出的字节数（FFmpeg 5.x+）
```

**这对"带宽监控"很重要**——你要知道真实的上行带宽，才能判断"我家网速够不够推 4500kbps"。

### 局限 5：错误分级太粗

现在只有"断"和"不断"两档。真实系统需要更细：

| 错误类型 | 例子 | 应该怎么办 |
|---|---|---|
| **暂时性** | `EAGAIN`、内存不足 | 重试，**保持连接** |
| **可恢复** | 连接被重置、超时 | **断开 → 重连** |
| **不可恢复** | 推流密钥错误、格式不支持 | **永久放弃这个目标**，别重试了 |

**现在的 `writePacket` 把所有 `ret < 0` 都当成"可恢复"**——如果推流密钥错了，它会反复重试，白白消耗资源。

**改进方向**：给 `Target` 加一个 `retryCount` 和 `permanentlyFailed` 标志。

### 局限 6：`release()` 之后 `targets` 被清空，`reconnectDisconnected()` 就失效了

看 `release()`：

```cpp
for (Target& target : targets) closeTarget(target, true);
targets.clear();        // ★ 清空了！
```

**这意味着 `close()` 之后，`reconnectDisconnected()` 什么都做不了**（循环体一次都不执行）。

**这个行为是合理的**（都关了还重连什么），但**接口语义要清晰**：应该明确文档化"close() 之后必须重新 open()"。

**或者在 `reconnectDisconnected()` 里加一句**：

```cpp
if (impl_->targets.empty()) {
    impl_->lastError = "还没 open()，无法重连";
    return 0;
}
```

### 局限 7：本讲发现的三个"可改进点"（复习）

我们在讲解过程中发现了三处值得改进的地方，同学们可以当作**重构练习**：

| # | 位置 | 问题 | 改进 |
|---|---|---|---|
| 1 | 5.5.4 节 | 复制了 11 行清理代码 | 复用 `closeTarget(target, false)` |
| 2 | 5.7.5 节 | 重复 `if (impl_->log)`，且**重连失败无日志** | 把 `say` 提成 `Impl` 的成员函数 |
| 3 | 5.6.6 节 | `open()` 全失败时**没调 `close()`** | 加上，保证"失败 = 回到未打开状态" |

**还有两处小瑕疵**：

- `StreamOutput.cpp` 第 6 行 `<libavutil/opt.h>` 和第 9 行 `<cstring>` 是**多余的 include**；
- 第 73 行多了一个分号 `};`。

> **老师点评**：同学们，发现这些瑕疵**不是挑刺，而是训练"代码评审"的眼光**。
>
> 作者在一个 commit 里写了 1025 行，有几处不够干净是**完全正常的**。重要的是——**你现在能看出它们了**。这就是进步。

## 10.5 思考题（8 题，附参考答案）

同学们先自己想，想完再看答案。**想不出来很正常。**

---

**Q1**：`writePacket` 里为什么每个目标都要 `av_packet_clone` 一份，而不是直接把原包写 N 次？

<details>
<summary>参考答案</summary>

三个原因：

1. **`av_interleaved_write_frame` 会取得 packet 的所有权**——它成功返回时会 `av_packet_unref` 掉你传进去的包。第二次循环时，`packet.data` 已经是悬垂指针、`packet.size == 0`，写出去的是空数据。
2. **`av_packet_rescale_ts` 是原地修改**的。第二次调用时，pts 已经被换算过一次了，再按同样的源时间基换算会被乘两次（例如 `{1,30}`→`{1,1000}`→ 再算一次变成 33333）。
3. **每个目标的时间基可能不同**（虽然现在都是 `{1,1000}`，但代码不该依赖这个巧合）。

`av_packet_clone` 创建一个新的 `AVPacket` 对象，字段跟原包一样但**独立可改**，数据共享（引用计数，零拷贝）。三个问题一起解决。
</details>

---

**Q2**：`av_packet_clone` 会拷贝数据吗？如果推 5 个目标、每包 25 KB、每秒 30 包，内存和 CPU 开销各是多少？

<details>
<summary>参考答案</summary>

**不会拷贝数据**（绝大多数情况）。

`av_packet_clone = av_packet_alloc() + av_packet_ref()`。`av_packet_ref` 只做两件事：
1. 把 `src` 的所有**字段**拷到 `dst`（pts、dts、size、stream_index……）；
2. 让 `dst->buf` 指向 `src->buf` 同一个 `AVBuffer`，并 **refcount++**。

真正的 H.264/AAC 数据（`AVBuffer::data`）**只有一份**。

**开销**：

| 项目 | 引用计数方案 | 真拷贝方案 |
|---|---|---|
| 内存 | 25 KB + 5 × ~100 字节 ≈ **25.5 KB** | 5 × 25 KB = **125 KB** |
| CPU（每秒） | 150 次整数自增（可忽略） | `30 × 5 × 25 KB = 3.75 MB/秒` 的 memcpy |

**而且 memcpy 会污染 CPU cache**，在高并发场景下影响更大。

**注意**：有一种情况 `av_packet_ref` 会真的拷贝——当源 packet 的数据**不在 `AVBuffer` 里**（比如你用 `av_new_packet` 之外的方式手动填的 `data`）。这时 `ref` 会分配新内存并 memcpy。我们项目里的 packet 都来自 `avcodec_receive_packet`，都在 `AVBuffer` 里，所以是零拷贝。
</details>

---

**Q3**：`writeVideo()` 返回 `false` 意味着什么？调用方应该怎么处理？返回 `true` 又意味着什么？

<details>
<summary>参考答案</summary>

`writePacket` 的返回值是 `anyOk`：

```
true  = 至少有一个目标成功收到了这一包
false = 一个目标都没有（所有目标都断了，或者本来就没连着）
```

**返回 `true`**：正常，继续跑。（**注意**：`true` 不意味着"所有目标都收到了"——可能有 1 路断了但其它 4 路正常。）

**返回 `false`**：这才是"链路整体不可用"的信号。调用方应该：

1. **停止编码**（继续编也没人收，白费 CPU）；
2. **提示用户**（"所有推流目标都断开了"）；
3. **尝试重连**（`reconnectDisconnected()`）；
4. 如果重连 N 次仍失败 → **停止推流**或进入"等待恢复"状态。

**错误的处理方式**是：把 `false` 当成"这一包没发出去"，然后重试这一包。因为：
- packet 已经被 `av_interleaved_write_frame` 吃掉了（成功的目标），你没法再发一次；
- 而且视频包必须按顺序发，重发旧的包没有意义。

**这也是第 13 课（断线重连）要处理的核心逻辑。**
</details>

---

**Q4**：如果某个目标只有视频流（没有音频流），我们给它写音频包会发生什么？代码是怎么处理的？

<details>
<summary>参考答案</summary>

**代码处理**（第 231~233 行）：

```cpp
AVStream* stream = isVideo ? target.videoStream : target.audioStream;
if (!stream)
    continue;      // 安静跳过
```

**为什么 `stream` 会是 `nullptr`？** 因为 `openTarget` 里：

```cpp
AVStream* audioStream = nullptr;
if (audioParams) {                                    // 只有音频参数非空才建流
    audioStream = avformat_new_stream(ctx, nullptr);
    ...
}
```

如果 `open()` 时 `audioParams` 传了 `nullptr`，这个目标就**没有音频流**，`target.audioStream == nullptr`。

**这时写音频包会被安静跳过**——不是错误，只是"这个目标不要音频"。

**这正是"多目标可以有不同配置"的支持**：

```
目标 1（斗鱼）：  视频 + 音频
目标 2（B站）：   视频 + 音频
目标 3（纯录像）：只要视频     ← 写音频包时会被跳过
```

**如果没这个 `if (!stream) continue;`**，会往 `nullptr->index` 上写，**直接段错误**。

**这就是第 1.3 节强调的"每个目标的一切参数都从它自己的对象上读"的价值。**
</details>

---

**Q5**：为什么 `open()` 时要把 `AVCodecParameters` 深拷贝一份存起来，而不是直接持有编码器的指针？

<details>
<summary>参考答案</summary>

三个理由，按重要性排序：

**① 解耦（最重要）**

`open()` 调用完之后，`StreamOutput` 跟编码器**彻底没关系了**。编码器可以被：
- 销毁（`video.close()`）；
- 重新配置（改分辨率、改码率）；
- 换成另一个编码器实例；

`StreamOutput` 都不在乎。**它手里有一份"快照"，自给自足。**

如果持有指针，编码器一销毁，`StreamOutput` 就悬垂了。

**② 支持重连（第 5.2.2 节点二）**

`reconnectDisconnected()` 要**重建流**，需要重新 `avcodec_parameters_copy`。如果当时编码器已经关了，持有指针的方案就抓瞎了。

**③ 支持 N 个目标**

我们要给 N 个目标各拷一份 `codecpar`，如果每次都找编码器要，就成了"N 次跨模块调用 + N 次依赖编码器还活着"。拷一份模板之后，变成"1 次跨模块调用 + N 次本地拷贝"。

**代价**：多占 `sizeof(AVCodecParameters) + extradata_size` 字节（extradata 通常只有几十字节）。**完全可以忽略。**

> **一句话总结**：这是"**在正确的时机做一份快照**"的经典案例——**用一点点内存，换来彻底的解耦**。
</details>

---

**Q6**：`release()` 里为什么先 `closeTarget` 再 `targets.clear()`，最后才释放 `videoParams`？顺序能颠倒吗？

<details>
<summary>参考答案</summary>

**逻辑上的顺序是"先停后清"**：

```
① closeTarget(target, true)  ← 停止所有输出（写 trailer、关 IO、释放 ctx）
② targets.clear()            ← 清空目标数组（析构 std::string 等）
③ avcodec_parameters_free    ← 释放参数模板
④ 复位 audioOffset 状态
```

**能不能颠倒？**

**①② 不能颠倒**：`closeTarget` 需要访问 `targets` 里的元素。如果先 `clear()`，就没东西可关了，**所有 FFmpeg 资源全泄漏**。

**②③ 之间**：技术上可以颠倒（`codecpar` 跟 `ctx` 是独立的，第 2.5 节点二讲过）。但**逻辑上先 ② 后 ③ 更清晰**——先清"使用者"，再清"被使用的模板"。

**④ 放最后**：它是"状态复位"，放最后表示"一切清理完毕，回到初始状态"。

**还有一个隐藏的严格要求**：`closeTarget` 里的 `av_write_trailer` 会写容器尾，**它可能需要读流的信息**（比如回写 duration）。虽然 `av_write_trailer` 主要用 `ctx` 内部的数据（已经拷过一份了），但保持"先关管道、再清模板"的顺序**永远不会错**。

> **工程原则**：**清理顺序 = 创建顺序的逆序**。创建是"参数模板 → 目标 → 管道"，清理就是"管道 → 目标 → 参数模板"。**记住这条规则，你永远不会在清理顺序上犯错。**
</details>

---

**Q7**：本课测试用 `rtmp://127.0.0.1:1/live/nothing` 来模拟"断线的目标"。请说出三个这样选的理由，以及如果你想测"超时"（而不是"立刻拒绝"）该怎么做。

<details>
<summary>参考答案</summary>

**选端口 1 的三个理由**：

1. **100% 确定失败**：端口 1 是特权端口，正常机器不会有服务监听；
2. **0 延迟**：目标是 `127.0.0.1`（本机回环），TCP 握手不出网卡，操作系统立刻回 RST，`avio_open2` 毫秒级返回。**测试跑得快**；
3. **无副作用**：不依赖网络、不依赖 DNS、不依赖任何外部服务。**CI 上能稳定跑**。

**对比**：

| 选项 | 问题 |
|---|---|
| 不存在的域名 | 要先 DNS 解析失败，慢且结果不确定 |
| 保留 IP（如 192.0.2.1） | 会等 TCP 超时，几十秒 |
| 拔网线 | 没法自动化 |

**想测"超时"怎么办？**

选一个**不可达但不会立刻被拒绝**的地址——比如私有网段里一个不存在的主机：

```cpp
{"dead", "rtmp://10.255.255.1/live/nothing"}
```

`10.255.255.1` 在私有网段（`10.0.0.0/8`）里，但你的局域网里大概率没有这个主机。这时：

- 如果它在同一网段 → 交换机发 ARP 请求，没人响应 → **等 ARP 超时**；
- 如果跨网段 → 包被路由出去，然后被丢弃 → **等 TCP 重传超时**。

**两种情况都会耗时数秒**，正好用来验证 `rw_timeout` 是否生效。

**验证方法**：对比 `kIoTimeout = "5000000"`（5 秒）和 `"30000000"`（30 秒）时的耗时差异（实验 5）。

> **进阶**：最可靠的办法是在本机起一个"只接受连接、永不回应"的 TCP 服务器（几十行 socket 代码，或者用 `nc -l -p 1935` 然后不说话），然后推 `rtmp://127.0.0.1:1935/...`。这时 TCP 握手成功、RTMP 握手永远得不到回应，`rw_timeout` 会精确触发。
</details>

---

**Q8**（综合题）：如果让你给 `StreamOutput` 加一个"自动重连"功能，你会怎么设计？请说出至少四个要考虑的点。

<details>
<summary>参考答案</summary>

**① 谁来触发重连？**

三个选择：

| 方案 | 优点 | 缺点 |
|---|---|---|
| A. `StreamOutput` 内部开一个重连线程 | 调用方零负担 | **库不该偷偷开线程**；线程安全复杂 |
| B. 调用方周期性调 `reconnectDisconnected()` | 简单、可控 | 调用方要记得调 |
| **C. 在 `writePacket` 失败时就地异步重试** | 及时 | 会阻塞写路径 |

**推荐 B**：**把"策略"留给调用方，把"机制"放在库里。** 这也是 `StreamOutput` 现在的设计哲学（它提供接口，不做决策）。

**② 什么时候重连？重试几次？**

- **立刻重连一次**（可能是瞬断）；
- 失败 → **指数退避**：1s → 2s → 4s → 8s → 16s → 上限 60s；
- **连续失败 N 次（比如 5 次）→ 放弃这个目标**；
- 加**随机抖动（jitter）**：避免所有目标同时重连造成"惊群"。

**③ ★ 重连成功后必须插关键帧！**

这是**最容易被忽略、后果最严重**的一点。

重连会建一个**全新的 `AVFormatContext`**、写一套全新的 header。**但码流会继续从当前帧写下去**——如果当前帧是 P 帧，它依赖前面已经发出去的参考帧，而**新连接的那条流没有那些参考帧**。

**后果**：观众端花屏、马赛克，直到下一个自然关键帧（可能要等 2 秒）。

**解决**：重连成功后，通知编码器：**"下一帧请编成 IDR（即时解码刷新）帧"**。

```cpp
// 方式一：AVFrame 上打标记（FFmpeg 5.x+）
frame->flags |= AV_FRAME_FLAG_KEY;
avcodec_send_frame(ctx, frame);

// 方式二：x264 私有选项
av_opt_set(encCtx->priv_data, "forced-idr", "1", 0);
```

**④ 重连期间的数据怎么办？**

两个问题：

- **积压的数据要不要补发？** —— **不要**。直播是实时的，补 3 秒前的旧帧毫无意义（观众已经跳过了）。**正确做法是丢弃积压，从"当前"继续。**
- **要不要记日志？** —— **必须**。记录"什么时候断的、断了多久、重连成功了吗"，这是线上排查的唯一线索。

**⑤ 重连会影响其它目标吗？**

**绝不能。** 重连某一路时：

- 其它路照常写包；
- 重连失败不影响其它路；
- 重连期间的日志要标明是哪一路（`[斗鱼] 重连成功`）。

**⑥ 状态怎么展示？**

给 `OutputTargetInfo` 加字段：

```cpp
struct OutputTargetInfo {
    ...
    int reconnectAttempts = 0;   // 已重试次数
    int64_t lastDisconnectMs = 0; // 上次断开的时刻
    bool permanentlyFailed = false; // 是否已放弃
};
```

**这就是第 14 课（`c2b8021`）要实现的东西。** 同学们可以先自己试着写一版，然后跟作者的实现对一下——**看看你想到几个点。**
</details>

---

## 10.6 下节课预告

下一课我们讲 **`31c3f57`：用三个线程、两个队列、一套时间规则，变成真正能推流**。

这一课我们做出来的 `StreamOutput` 有个致命问题：**它是阻塞的，而且跑在调用方的线程上**。网络一卡，整条流水线就停了。

下一课我们要做的是：

```
采集线程 ──→ [有界视频队列] ──→ 编码线程 ──→ [有界包队列] ──→ 发送线程 ──→ StreamOutput
                  ↑                                              ↑
            队列满了怎么办？                              按 pts 做 pacing
            （丢帧策略）                              （sleep 到该发的时刻）
```

三个新问题：

1. **三个线程怎么协作**（生产者-消费者、条件变量、优雅退出）；
2. **有界队列满了怎么办**（丢 I 帧还是丢 P 帧？丢最新的还是丢最旧的？）；
3. **一套时间规则**（采集时钟、编码时钟、发送时钟怎么统一，音画怎么同步）。

**第 3 课的 `BoundedQueue` 和本课的 `StreamOutput`，将在第 11 课第一次合体。**

下课。

---

## 附：本讲「五章对照索引」

> **用法说明**：这张表把本讲的全部内容压缩成**五个维度**的对照，方便你复习、检索、以及跟前几课建立联系。横向五行是本讲的**五个部分**，纵向五列是**五个维度**。

| 本讲部分 | 覆盖章节 | 核心知识点 | diff 中的文件与行号 | 怎么验证 / 上哪去练 |
|---|---|---|---|---|
| **① 预习：概念与原理** | 第 1~3 章 | FLV 毫秒时间基、`stream_index`、header/trailer、`codec_tag=0`；`avformat` 输出上下文五步建/五步拆、`AVFMT_NOFILE`、`rw_timeout`、URL 统一抽象；`av_packet_clone` 引用计数零拷贝、写失败隔离、AAC priming 归零 | 概念章，对应实现见 `src/core/output/StreamOutput.cpp` 第 64 行（`kFlvTimeBase`）、第 66 行（`kIoTimeout`）、第 121~189 行（`openTarget`）、第 209~278 行（`writePacket`） | **实验 3**（改时间基看快进）、**实验 4**（关掉偏移看负时间戳）、**实验 7**（打印 `data` 指针验证零拷贝） |
| **② 实现：模块本体** | 第 4~6 章 | PImpl 惯用法与"析构函数不能 inline"；`OutputTargetInfo` 快照；`Target` 状态机（`connected`/`lastError`/统计）；`openTarget` 阶梯式回滚；`closeTarget` 幂等清理与防悬垂；`writePacket` 克隆-改字段-换算-写出-隔离五步；`say` 日志小工具；三处 CMake 挂接与 `PUBLIC`/`PRIVATE` 传播 | `src/core/output/StreamOutput.h`（53 行，全）、`src/core/output/StreamOutput.cpp`（389 行，全）、`src/core/output/CMakeLists.txt`（14 行，全）、顶层 `CMakeLists.txt` 第 +33 行、`tests/CMakeLists.txt` 第 +38~47 行 | 编译 `core_output` 静态库；用 `ffprobe` 看产出的 flv；检查有没有 `duration` 字段（验证 trailer 写了） |
| **③ 验证：端到端测试** | 第 7 章 | `check` 断言框架；BMP 手写 54 字节头与"自下而上"行序；`makeTestFrame` 彩条+移动方块；`makeSineChunk` 440Hz 正弦；`runOnce` 端到端流水线与 `pushAudioUpTo` 交错；`probeFlv` 解封装回读（含解码第一帧 + `sws_scale`）；四个用例覆盖 13 项能力 | `tests/test_output.cpp`（559 行，全）：第 27~41 行 `check`、第 74~112 行 `saveBmp`、第 185~259 行 `runOnce`、第 275~377 行 `probeFlv`、第 403~453 行用例 1、第 455~488 行用例 2、第 490~517 行用例 3、第 519~545 行用例 4 | `build.bat` → `test_output.exe`；看 30 条断言全绿；**打开 `output_first_frame.bmp` 用眼睛看**；用 VLC 播 `output_single.flv` 看方块是否匀速扫描 |
| **④ 实践：动手与排错** | 第 8~9 章 | 七个改造实验（换真 rtmp / 去掉 `codec_tag` / 改时间基 / 关音频偏移 / 加超时断言 / 补日志断言 / 观察零拷贝）；13 条排错条目（编译期 4 条 + 运行期 7 条 + 行为层 2 条） | 不新增代码，全部是对 ②③ 部分代码的**修改与观察** | 按 8.2~8.8 逐个做实验；出问题时按 9.1~9.3 的清单逐条排查 |
| **⑤ 沉淀：总结与延伸** | 第 10 章 | 11 条必记知识点；四张 API 速查表；7 条局限（无自动重连、无 pacing、单线程阻塞、`bytesSent` 不准、错误分级粗、`release` 后不可重连、三处可改进点）；8 道思考题（含"重连必须插 IDR"这一关键认知） | 对全讲 1025 行 diff 的回顾与批判；三处改进点分别位于 `StreamOutput.cpp` 第 261~269 行、第 416~425 行、第 349~352 行 | 做完 8 道思考题（尤其 Q8 综合题）；预习第 11 课 `31c3f57`（三线程两队列） |

### 附：本讲与前面几课的衔接点（复习时对照）

| 本讲用到的东西 | 在哪一课学的 | 本讲怎么用的 |
|---|---|---|
| FLV 容器结构、RTMP 与 FLV 的关系 | **第 2 课**（`df3603c8`） | 第 1 章回扣：毫秒时间基、`stream_index`、全局头；**不重复讲结构** |
| `AV_CODEC_FLAG_GLOBAL_HEADER` | **第 2 课** 4.2 节 | §1.4：`extradata`（SPS/PPS）的来源，没有它 FLV 就解不开 |
| `StreamTarget` / `LogCallback` / `VideoFrame` / `AudioChunk` | **第 3 课**（`a4453bb1`） | §4.2、§7.6：跨模块契约；`LogCallback` 让测试能"打印 + 收集"两件事 |
| `BoundedQueue` | **第 3 课** | 本讲**没用上**——第 10 课三线程化时才会用 |
| 两个编码器 + `AVPacketPtr` + `codecParameters()` / `timeBase()` | **第 8 课**（`66e0768`） | §5.6、§7.8：三个"插头"让 `StreamOutput` 一行不改老代码就能接进来 |
| AAC 的 `initial_padding` | **第 8 课**（`AudioEncoder::open` 的日志） | §3.7：`StreamOutput` 用 `audioOffset` 把负 pts 归零 |
| PImpl 惯用法 | 第 8 课（`VideoEncoder`）用过 | §4.2 系统讲解原理与"析构不能 inline"的坑 |
| "存一张图让人看"的测试纪律 | **第 0 课** 4.3 节提出，第 4/5 课用过 | §7.5 `saveBmp` + §7.10 用例 1 最后两条断言 |

### 附：本讲三个"可以立刻动手的重构练习"

| # | 位置 | 现状 | 改成 |
|---|---|---|---|
| 1 | `StreamOutput.cpp` 第 261~269 行 | 复制了 11 行清理代码 | `closeTarget(target, false);` 一行 |
| 2 | `StreamOutput.cpp` 第 416~425 行 | 重复判空 `if (impl_->log)`，且重连失败无日志 | 把 `say` 提成 `Impl` 的成员函数，三个地方共用 |
| 3 | `StreamOutput.cpp` 第 349~352 行 | `open()` 全失败时没调 `close()` | 在 `return false` 前加 `close();` |

**做完这三项之后，重新跑一遍 `test_output.exe`，确认 30 条断言还是全绿。**

**这就是重构的正确姿势：先有测试保底，再动代码。**

---

> **老师最后的话**：这一课我们的代码"只是"在写文件和打包。但同学们回头看看——**"一次编码、多处分发""失败隔离""用替身做端到端测试"**——这三件事，是把一个"能跑的 demo"变成"能上线的系统"的分水岭。第 10 课我们就要给这个系统装上马达了。

