# 第 11 课：三线程推流管线 —— 用三个线程、两个队列、一套时间规则，变成真正能推流

> **课程编号**：Commit 11 / `31c3f57`
> **提交时间**：2026-10-21 19:59
> **提交信息**：用三个线程、两个队列、一套时间规则，变成真正能推流
> **改动规模**：7 个文件，约 880 行 diff（新增 `src/core/pipeline/StreamPipeline.{h,cpp}` 共 488 行、`src/core/pipeline/CMakeLists.txt` 20 行、`tests/test_pipeline.cpp` 298 行；另修改顶层 `CMakeLists.txt` +1 行、`tests/CMakeLists.txt` +8 行、`src/core/common/Common.h` +2 行）
> **本课关键词**：线程划分、`std::thread`、`joinable`/`join`、PImpl、生产者-消费者落地、`std::atomic<bool>` 停止标志、`steady_clock` 单调时钟、`sleep_until` 节拍器、pts 时间轴统一、音视频同步（A/V sync）、丢帧策略、队列排空、`timeBeginPeriod`、`sws_scale` 缩放、`BoundedQueue` 容量选型、`PipelineStats`

---

## 目录

1. [第一章 开场白：从"零件"到"整机"](#第一章-开场白从零件到整机)
2. [第二章 知识点预习（上课前的十一件兵器）](#第二章-知识点预习上课前的十一件兵器)
3. [第三章 diff 总览：这次提交到底加了什么](#第三章-diff-总览这次提交到底加了什么)
4. [第四章 CMake 三处接线](#第四章-cmake-三处接线)
5. [第五章 Common.h：一个新的配置项 audioDeviceIndex](#第五章-commonh一个新的配置项-audiodeviceindex)
6. [第六章 StreamPipeline.h：整机的"操作面板"](#第六章-streampipelineh整机的操作面板)
7. [第七章 StreamPipeline.cpp（上）：Impl 的家底与两个工具方法](#第七章-streampipelinecpp上impl-的家底与两个工具方法)
8. [第八章 videoLoop：采集线程的心脏](#第八章-videoloop采集线程的心脏)
9. [第九章 audioLoop：最短的一条龙](#第九章-audioloop最短的一条龙)
10. [第十章 writeLoop：把两条河合成一条河](#第十章-writeloop把两条河合成一条河)
11. [第十一章 start()：开机的十三个步骤](#第十一章-start开机的十三个步骤)
12. [第十二章 stop() 与其余五个成员](#第十二章-stop-与其余五个成员)
13. [第十三章 test_pipeline.cpp：端到端验收](#第十三章-test_pipelinecpp端到端验收)
14. [第十四章 动手环节：编译、运行、看输出](#第十四章-动手环节编译运行看输出)
15. [第十五章 排错讨论：四种典型故障](#第十五章-排错讨论四种典型故障)
16. [第十六章 小结](#第十六章-小结)
17. [第十七章 思考题](#第十七章-思考题)
18. [附：本讲"五章对照索引"](#附本讲五章对照索引)

---

## 第一章 开场白：从"零件"到"整机"

同学们，我们这门课走到今天，已经攒了一大堆零件了。我们先把它们摆在桌面上数一数：

| 零件 | 哪一课做的 | 它现在能干什么 |
|---|---|---|
| `StreamConfig` / `VideoFrame` / `AudioChunk` | **第 3 课**（`a4453bb1`） | 定义"数据长什么样"，是全体模块共同遵守的合同 |
| `BoundedQueue<T>` | **第 3 课**（同上） | 线程之间安全地传货，满了丢最旧的 |
| `ScreenCapturer` | **第 4 课**（`81eaa1f5`） | 截一张桌面图，返回 BGRA 的 `VideoFrame` |
| `CameraCapturer` + `CameraFrameConverter` | **第 5、6 课**（`281a22be` / `e92cbd3`） | 摄像头出裸帧，转成统一的 BGRA `VideoFrame` |
| `AudioCapturer` | **第 8 课**（`1ce7f62`） | WASAPI 抓系统声音/麦克风，返回 float 的 `AudioChunk` |
| `VideoEncoder` / `AudioEncoder` | **第 9 课**（`66e0768`） | 吃进 `VideoFrame`/`AudioChunk`，吐出 `AVPacketPtr` |
| `StreamOutput` | **第 10 课**（`0ccd108`） | 把压缩后的包封装成 FLV，可以同时发给多个目标 |

每一个零件，我们都在它自己的测试里**单独验证过了**。`test_encode` 能编 60 帧，`test_output` 能写出双流的 flv，`test_audio_capture` 能抓到 300 块音频。

但是——同学们请注意这个"但是"——**这些零件到现在为止还从来没在同一个程序里同时跑过**。它们是散在仓库各个目录里的一堆静态库，谁也不认识谁。这就像你已经造好了发动机、变速箱、轮胎、方向盘，全都摆在车间地上，单个零件上台架测试指标漂亮，但**车还没装起来，一步都开不了**。

所以这一次提交 `31c3f57`，干的就是"总装"这件事：

> **把散落的模块用三条线程串起来，变成一个 `StreamPipeline` 类——你只要 `start(config, log)`，它就自己开始推流；你 `stop()`，它就体面收尾。**

这就是标题里那句话的全部含义：**"用三个线程、两个队列、一套时间规则，变成真正能推流"**。

我们把这个标题拆开，它其实已经是今天这堂课的目录了：

```
用三个线程     → 视频线程 / 音频线程 / 写出线程        （第八章、第九章、第十章）
两个队列       → videoQueue(24) / audioQueue(256)     （第 3 课的 BoundedQueue 首次实战）
一套时间规则   → steady_clock + pts 换算 + 毫秒基准    （第八章 §8.4，本课的灵魂）
```

这里我要先给同学们打个预防针：**这一课的代码"看起来"不复杂，一共 488 行，没有一个新算法、没有一个新的第三方 API。它的难度不在"写"，在"想"。**

难在哪里？三个地方：

1. **线程边界画在哪？** 为什么是三个线程而不是四个、五个？为什么不多开一个专门的编码线程？答案是"谁会阻塞、谁独占资源"，我们 §2.2 详细算这笔账。
2. **时间轴怎么统一？** 视频的时钟单位是一帧，音频的时钟单位是一个采样点，FLV 的时钟单位是毫秒，三个单位互不相认。如果不统一，录出来的视频就是"嘴型对不上声音"。我们 §2.6~§2.8 专门讲。
3. **怎么停下来？** 多线程程序最难的不是启动，是**收尾**。直接 exit？写到一半的 FLV 文件是坏的，RTMP 连接没关，编码器里的延迟帧没吐出来。我们第十一章、十二章逐行看 `start`/`stop` 的对称关系。

最后再说一句这门课的学习方法。前面的课，你一行行往下读代码就够了；这一课不行——**你必须一边读，一边在脑子里"同时跑"三个线程**：视频线程在干什么的时候，音频线程在干什么，write 线程又卡在哪一步。建议同学们拿三张便签纸，各写一个循环体，读到哪就把哪张挪一步，把"时间"可视化。这是理解并发代码唯一靠谱的办法。

我们开始。

> **另外提醒一件事**：这次 commit 顺手在 `StreamConfig` 里加了一个字段 `audioDeviceIndex`（第五章），因为真正串起来之后才发现——**"用哪个音频设备"还没办法配置**。这就是集成的典型收益：**零件阶段的盲区，只有总装的时候才暴露出来**。

---

## 第二章 知识点预习（上课前的十一件兵器）

读这次 commit 需要的预备知识比较多。老规矩，每个都讲到"够用"的程度，第一次出现的专业名词都给通俗解释。

### 2.1 流水线（Pipeline）到底是什么

**流水线**（pipeline）：把一个大任务拆成若干个**阶段**（stage），前一阶段的输出是后一阶段的输入，所有阶段**同时进行**，像工厂的装配线一样。

为什么一定要流水？我们先看不流水的做法——**串行**（sequential）：

```
采集第 1 帧 → 编码第 1 帧 → 发送第 1 帧 → 采集第 2 帧 → 编码第 2 帧 → ...
```

假设采集 5ms、编码 20ms、发送 5ms，那么一帧要 30ms，**最快也就 33fps**，而且_codec_稍微卡一下立刻掉帧。

改成流水之后：

```
时刻 T:    [采集员]采第 1 帧   [编码员]闲着        [发送员]闲着
时刻 T+5:  [采集员]采第 2 帧   [编码员]编第 1 帧   [发送员]闲着
时刻 T+10: [采集员]采第 3 帧   [编码员]编第 2 帧   [发送员]发第 1 帧
```

三个岗位同时开工，**吞吐量的瓶颈就只剩下最慢的那一个环节**（20ms → 理论 50fps），而不是所有环节之和。

这个概念在 CPU 设计里叫指令流水线，在数据处理里叫流式计算，在我们的直播软件里就叫 **推流管线**。同学们记住这个公式：

> **流水线的吞吐率 = 1 / 最慢阶段的耗时**（这是上限，称为"瓶颈定律"或阿姆达尔定律的简化版）

### 2.2 为什么是"三个线程"——线程边界的画法

这是本课最需要先想清楚的问题。我先给结论：**线程不是按"功能数量"分的，是按"必须独占的资源"和"会阻塞的点"分的。**

我们把真实运行时会**卡住等待**（阻塞）的地方列一张表：

| 会阻塞的操作 | 卡多久 | 谁负责 |
|---|---|---|
| `camera.nextFrame(1000)` 等摄像头下一帧 | 最多 1000 ms | 视频采集 |
| `audio.nextChunk(100)` 等 WASAPI 缓冲区 | 最多 100 ms | 音频采集 |
| `av_interleaved_write_frame` 写网络/写磁盘 | 取决于网速，**可能几百 ms 甚至几秒** | 输出 |
| `avcodec_send_frame` / `receive_packet` 编码 | 纯 CPU，通常 5~30 ms | 编码器 |

再看**独占性**（不能被两个线程同时调用的东西）：

| 资源 | 能不能多线程同时调用 | 原因 |
|---|---|---|
| `VideoEncoder` | **不能** | FFmpeg 的 `AVCodecContext` 内部有状态（参考帧队列、延迟帧），并发调用会串味 |
| `AudioEncoder` | **不能** | 同上，还多一个 `pending` 缓冲区（第 9 课讲的攒批） |
| `ScreenCapturer` | 不能 | 持有一个 `IDXGIOutputDuplication`，要按 `AcquireNextFrame`/`ReleaseFrame` 配对调用 |
| `StreamOutput` | 不能 | 内部遍历 `targets` 向量，还会修改 `connected` 状态 |

好，把这些约束摆在一起，线程怎么分就呼之欲出了：

```
线程 A（videoThread）：独占 ScreenCapturer/CameraCapturer + VideoEncoder
                      职责：截一帧 → 打时间戳 → 缩放 → 编码 → 塞进 videoQueue
线程 B（audioThread）：独占 AudioCapturer + AudioEncoder
                      职责：取一块采样 → 编码 → 塞进 audioQueue
线程 C（writeThread）：独占 StreamOutput
                      职责：从两个队列取包 → 交错写出
```

同学们看清楚一件事：**没有"编码线程"**。编码这个动作被"搭便车"塞进了采集线程里。

为什么不单独开一条编码线程？因为一旦开了，你就变成 **采集 → 编码 → 发送** 三段，中间是**两条**线程交接 + **四个**队列（视频侧多一条队列）。代价是：

- 原始 `VideoFrame`（1080p BGRA = 7.9MB）要跨线程传一次，**多一次 7.9MB 拷贝/帧**（哪怕是 `shared_ptr` 零拷贝，也要多做一次队列同步）；
- 采集 5ms + 编码 20ms = 25ms，但分成两个线程后，**端到端延迟反而变长**（每过一个队列就多一跳延迟）；
- 线程数增加，出错的机会、调试的难度都上升。

而我们现在的分工，视频从采集到编码是**一气呵成**的同一个线程，跨线程传的只有压缩后的小包（几 KB 到几十 KB）。这是典型的工程取舍：**牺牲一点并行度，换取更低的延迟和更简单的模型。** 直播场景延迟敏感，这个取舍是对的。

> **一句话记住线程划分的依据**：能独占的就独占，会阻塞的就切出去，剩下的能合并就合并。**线程的边界应该落在"数据体积最小、语义最清晰"的地方**——这里就是"压缩后的包"这一处。

### 2.3 生产者-消费者：第 3 课的 BoundedQueue 第一次上真战场

同学们翻回第 3 课。那时候我们在白板上谈"生产者-消费者模式"（Producer-Consumer），`BoundedQueue` 写了一堆方法但只在 `test_common` 里用 `int` 测过。今天它终于有了真实的岗位。

先把 `BoundedQueue.h` 的关键语义复习一遍（**这一段很重要，因为本课所有的"丢包"行为都源于它**）：

```cpp
template <typename T>
class BoundedQueue {
public:
    explicit BoundedQueue(size_t maxSize = 64) : maxSize_(maxSize) {}

    void push(T item)
    {
        std::lock_guard<std::mutex> lock(mutex_);
        if (queue_.size() >= maxSize_)   // ← 满了
            queue_.pop_front();          // ← 丢掉最旧的那个（不会阻塞生产者！）
        queue_.push_back(std::move(item));
        cv_.notify_all();
    }

    std::optional<T> tryPop(int timeoutMs)
    {
        std::unique_lock<std::mutex> lock(mutex_);
        cv_.wait_for(lock, std::chrono::milliseconds(timeoutMs),
                     [this] { return !queue_.empty(); });   // 谓词：醒来先看真的有货没
        if (queue_.empty())
            return std::nullopt;
        T item = std::move(queue_.front());
        queue_.pop_front();
        return item;
    }

    void clear() { /* 加锁清空 */ }
    bool empty() const { /* 加锁判空 */ }
};
```

注意这里我贴的是删掉了注释、只保留主干的简化版。**对本课最关键的一条语义**：

> **`push` 永不阻塞。** 队列满了它**丢最旧的**，然后照样返回。

这条语义决定了整个系统的**降级方式**：当写网络的速度跟不上编码速度时，**队列会堆满 → 开始丢旧包 → 视频出现花屏/跳帧，但不会卡住编码，也不会内存爆炸**。这叫"优雅降级"（graceful degradation），是实时系统的标配。

对应的反面教材是 `std::queue` 无限增长：网络一卡，内存几分钟就吃满，进程 OOM 崩溃。**实时系统里，"宁可丢数据，不可无界"。**

本课的两个队列容量是这样选的（第 124~125 行）：

```cpp
BoundedQueue<AVPacketPtr> videoQueue{24};
BoundedQueue<AVPacketPtr> audioQueue{256};
```

| 队列 | 容量 | 换算成时间 | 为什么这么选 |
|---|---|---|---|
| `videoQueue` | 24 个包 | 30fps 下约 **0.8 秒** | 视频包大（几 KB~几百 KB），24 个够吸收网络抖动；再大就是纯增加延迟，因为直播最忌讳"越推越慢" |
| `audioQueue` | 256 个包 | AAC 每包 1024 样本、48kHz → 每包 21.3ms，256 包约 **5.5 秒** | 音频包小而密（每包约 320 字节），**256 个包的内存还不如 1 个视频包**；容量给足能吸收更长的网络抖动，而且人耳对"卡顿"比对"延迟"敏感得多 |

同学们把这张表记住，这是**"按数据的体积和节奏来定队列大小"**的典型例子，面试问到流控设计可以直接用。

### 2.4 std::thread 与 join/joinable——线程必须有主人

C++11 的 `std::thread` 用法极简：

```cpp
std::thread t([]{ /* 线程里干的事 */ });   // 构造即启动
```

但有三个必须知道的点：

1. **`std::thread` 对象析构时如果还是 joinable（可汇合）状态，程序直接 `std::terminate()` 崩溃。** 这是新手最常踩的雷。
2. **`join()`**：当前线程**阻塞等待**这个线程跑完，然后回收它的资源。之后该对象不再是 joinable。
3. **`detach()`**：把线程放生，让它自生自灭。放生之后你就完全失去控制（不能 join，不知道它什么时候结束，它还可能访问已析构的对象）。**本课绝不用 detach。**

什么叫"joinable"（可汇合）：`std::thread` 对象**当前确实关联着一条活着的、还没被 join/detach 过的线程**。默认构造的 `std::thread`（没给任务）是不可 joinable 的——这正好让我们可以用"三个 thread 成员是否 joinable"来判断"管线到底有没有启动过"（第十二章 `stop()` 的分支判断就靠它）。

一条铁律送给同学们：

> **每一个线程都必须有明确的主人负责它的终止。** 谁 start，谁 stop；没有主人的线程就是野线程，迟早出事。

### 2.5 std::atomic<bool> 停止标志：为什么不能用 volatile

三个线程怎么被告知"该收工了"？答案是共享一个**停止标志**：

```cpp
std::atomic<bool> running{false};
```

为什么不能用 `bool` 或 `volatile bool`？

- **普通的 `bool`**：编译器很聪明，它可能优化成"把值读进寄存器，以后再也不读内存"（因为单线程视角看它没被改过）；而且两个线程同时读写一个非原子对象，在 C++ 标准里叫**数据竞争**（data race），属于**未定义行为**（UB, Undefined Behavior）——你今天测着好好的，换个优化级别、换个 CPU 就崩了。
- **`volatile bool`**：`volatile` 只能保证"每次都真的读内存"，**它不提供原子性，也不提供线程间的可见性顺序保证**。在 x86 上碰巧常常能用，但这是运气，不是规范。
- **`std::atomic<bool>`**：标准保证三件事——① 读写是原子的（不可能读到"半个值"）；② 一个线程的写对另一个线程**可见**（默认 `memory_order_seq_cst`，顺序一致）；③ 不会被编译器优化掉重复读取。

本课一共有 6 个 atomic 成员：1 个 `running` 开关 + 5 个统计计数器（第 131~135 行）。统计计数器之所以要用 `atomic<int64_t>`，是因为两个线程分别在 `++`、而第三条线程在读：

```cpp
++videoPackets;     // videoLoop 里
++videoDropped;     // videoLoop 里
++audioChunks;      // audioLoop 里
++audioPackets;     // audioLoop 里
```

而 `stats()` 从**第四条线程**（调用者，通常是 UI 线程）用 `.load()` 读这些值。全部靠 atomic 来避免锁。

> **顺便复习一个术语**：**无锁编程**（lock-free）——用原子操作代替互斥锁。`++counter` 这种极简单的操作用 `atomic` 比上锁快得多（锁涉及内核态切换，约几十纳秒到几微秒；原子加法是一条 CPU 指令 `lock xadd`，几纳秒）。

### 2.6 单调时钟 steady_clock：为什么不能用手表时间打 pts

C++ 有三种时钟（都在 `<chrono>` 里）：

| 时钟 | 是什么 | 会不会跳变 | 适合干什么 |
|---|---|---|---|
| `system_clock` | **墙上时钟**（wall clock），就是你的系统时间 2026-10-21 19:59:27 | **会跳**！NTP 对时、用户改时区、闰秒都可能让它往前往后跳 | 显示给用户看的时间、文件名时间戳 |
| `steady_clock` | **单调时钟**（monotonic clock），通常从开机起算的滴答数 | **永不减少**，不受任何对时影响 | **一切"测时长"的场合** |
| `high_resolution_clock` | 往往是上面两者的 typedef，实现不确定 | 不确定 | 基本不用 |

本课第 92 行：

```cpp
using Clock = std::chrono::steady_clock;
```

为什么必须用 `steady_clock`？设想我们在打 pts 时用了 `system_clock`，然后推流推到一半，**系统自动和 NTP 服务器对了一次时，把时间往回拨了 200ms**。那么接下来几帧的"流逝时间"就是负数——pts 倒退，播放器要么报错要么画面卡住。

> **铁律**：**凡是"测两个时刻之间过了多久"，一律用 `steady_clock`。** 这条规则在音视频、游戏、任何定时器场景都适用。

配套的两个小工具（本课第 94~97 行的匿名命名空间）：

```cpp
int64_t msSince(Clock::time_point from, Clock::time_point now)
{
    return std::chrono::duration_cast<std::chrono::milliseconds>(now - from).count();
}
```

这是 `<chrono>` 的标准套路：`now - from` 得到一个 `duration`（ duration = 时长），然后 `duration_cast<毫秒>` 把它换算成毫秒单位，最后 `.count()` 取出整数。

**匿名命名空间**（anonymous namespace，`namespace { ... }`）：里面的符号只在当前 .cpp 文件可见，等价于给它们加了 `static`。这样 `Clock` 和 `msSince` 就不会污染全局符号表，跟别的模块的同名东西打架。这是 C++ 里替代 `static` 的现代写法。

### 2.7 时间戳三兄弟：pts / dts / time_base

这三个概念是音视频领域的核心词汇，同学们务必搞清楚（FFmpeg 相关在第 2、9、10 课出现过，这里做一次系统梳理）。

- **pts**（Presentation Time Stamp，显示时间戳）：这一帧画面/这一段声音**应该在什么时刻被播放出来**。
- **dts**（Decoding Time Stamp，解码时间戳）：这一帧应该**在什么时刻被解码**。只在有 B 帧（双向预测帧）时才跟 pts 不同——因为 B 帧要参考"未来"的帧，所以解码顺序 ≠ 播放顺序。**本课两个编码器都设了 `max_b_frames = 0`，没有 B 帧，所以 pts == dts，我们暂时省了这个烦恼。**
- **time_base**（时间基）：pts 的**单位**。pts 本身只是一个整数，它代表多少秒，完全取决于 time_base。公式：

  ```
  真实秒数 = pts × time_base      （time_base 是一个分数 num/den）
  ```

本项目的三套单位是这样的：

| 环节 | time_base | pts 的含义 | 1 个 pts 单位 = |
|---|---|---|---|
| 视频采集（videoLoop 手算） | — | 帧序号 | — |
| `VideoEncoder` | `{1, fps}` = `1/30` | 第几帧 | 33.33 ms |
| `AudioEncoder` | `{1, sampleRate}` = `1/48000` | 第几个采样点 | 0.0208 ms |
| FLV / MP4 容器（StreamOutput） | `{1, 1000}` = `1/1000` | 第几毫秒 | 1 ms |

同学们看，**视频用的是"帧数"，音频用的是"采样点数"，容器用的是"毫秒"，三者互不相认**。所以必须有一步"换算"，这一步在 FFmpeg 里叫 **`av_packet_rescale_ts`**（第 10 课讲过，位于 `StreamOutput.cpp` 第 206 行）。本课 `writeLoop` 每写一个包就做一次换算：

```cpp
output.writeVideo(**packet, videoEncoder.timeBase());   // 传编码器的 time_base 进去
```

换算的数学本质是：

```
pts_new = pts_old × (old_time_base / new_time_base)
```

例如视频第 90 帧：pts=90，time_base=1/30 → 显示时刻 = 90 × (1/30) = 3.0 秒。换成毫秒基准：90 × (1/30) ÷ (1/1000) = 90 × 1000/30 = **3000 ms**。这就是为什么测试里断言"三秒时最后时间戳接近 3000 ms"。

### 2.8 音视频同步（A/V Sync）：嘴型怎么对上

**音视频同步**（A/V synchronization）：让声音和画面在播放端在时间轴上对齐。

先说一个反直觉的真相：**在实时推流里，"同步"靠的不是某一方去追赶另一方，而是两方各自记录"自己这条时间轴的真实时间"，并且都从同一个零点起步。**

本课这两条时间轴是这么来的：

- **视频时间轴**：第一帧到达时记下 `firstFrameAt`（第 227、252~255 行），之后每帧的 pts = （当前时刻 − firstFrameAt）× fps = **按真实流逝时间算出的帧序号**。
- **音频时间轴**：`AudioEncoder` 内部维护 `nextPts`，每喂进一帧就 `nextPts += frameSize`（第 9 课 `AudioEncoder.cpp` 第 111~113 行）。因为 WASAPI 每 21ms 左右给一块，`frameSize=1024` 样本也是 21.3ms，**所以"样本数"天然就是"真实时间"的化身**（48000 样本 = 1 秒）。

两条线的起点都是 0，都按真实时间推进，所以**它们自动对齐**。同学们记一句口诀：**同步靠的不是"同源"，而是"同规则"**——两方不需要知道对方的存在，只要都诚实地记录自己的真实时间，对齐就是自然的结果。

那为什么还会漂移呢？因为三条线程启动有先后、编码延迟不同。误差来源：

1. **AAC 的 `initial_padding`（初始填充）**：AAC 编码器为了做前后重叠，开头要多吃约 1024 个样本，导致**第一包的 pts 是负数**。这个坑我们在第 9 课的日志里预告过，第 10 课已经在 `StreamOutput::Impl::writePacket` 里用 `audioOffset` 把首包 pts 归零了——所以本课 `writeLoop` 不需要再操心这件事，可以直接享受到成果。
2. **视频第一帧的采集耗时**：`firstFrameAt` 是在第一帧**成功采到之后**才记的，而 `audio.open()` 在它之前，所以音频可能比视频早几十毫秒。
3. **WASAPI 的补静音**：如果一段时间没有声音（比如系统静音），`AudioCapturer` 会主动补静音保持时间轴连续（第 8 课讲的 `padSilence`，阈值 10ms~200ms）——**这保证了音频时间轴不会因为"没声音"而停止推进**，这一点对同步极其关键。

测试怎么验证？测完以后把两个文件**解封装读回来**，比较视频最后时间戳和音频最后时间戳：

```cpp
const int64_t drift = a.lastVideoMs - a.lastAudioMs;
check(drift > -500 && drift < 500, "音视频时间差在 500ms 以内（没有累积漂移）");
```

500ms 这个阈值是有讲究的：业界公认**音视频偏差超过 ±125ms 人就开始察觉**，超过 ±500ms 就明显难受（ITU-R BT.1359 标准给的是 -125ms~+45ms 的"不可察觉"区间）。我们这版 ±500ms 是**工程上可接受的粗标准**，后面还可以做得更好（第十七章思考题 Q4 会让你算漂移率）。

### 2.9 丢帧策略：什么时候丢、丢了怎么补

**丢帧**（frame drop）：采集到了帧但因为种种原因丢弃、不送去编码。

实时系统里丢帧不可怕，可怕的是**不知道自己丢了多少、丢得没有章法**。本课的丢帧发生在**两个层面**：

**层面一：显式丢帧（有统计），第 262~266 行**

```cpp
if (pts <= lastPts) {
    ++videoDropped;
    waitNextFrame();
    continue;
}
```

含义：这一帧和上一帧落在了**同一个时间刻度**上（pts 相同），说明采集比目标帧率快了，丢掉。典型场景——摄像头实际输出 60fps 而你配置 30fps，那么每两帧就有一帧的 pts 与前一帧相同，被丢掉。**这就是"降采样到目标帧率"的实现。**

另一个触发场景是 §8.3 会讲的"追赶"：某帧处理超时后，节拍器不再等待，立刻开下一帧，此时可能两帧落在同一刻度。

**层面二：隐式丢包（无统计），`BoundedQueue::push` 满了丢最旧的**

这是危险的，因为**丢一个视频包可能丢的是 GOP 中间的 P 帧**，接收端解码会花屏。第 3 课的坑点 §8.1 就批评过"丢帧无统计"，本次 commit 仍然没修，同学们第十七章 Q2 会针对它讨论。

**关键的好处在于丢帧后 pts 怎么处理**。本课的做法是 **pts 完全按真实时间算**，所以丢帧后 pts **不会**出现空洞。举例：

```
第 1 帧 pts=0   第 2 帧 pts=1   第 3 帧被丢   第 4 帧 pts=3
```

第 4 帧仍然是 pts=3（对应 100ms），而不是 pts=2。播放器收到的时间轴是连续的，**画面会在丢帧处"跳"一下，但整体速度不变**。

反过来，如果 pts 按"帧计数"累加（每来一帧 +1，不管丢没丢），丢帧后时间轴就被"压缩"了，视频会**整体变慢、声音越拉越远**——这是新手做推流器最常见的音画不同步 bug。**请同学们务必记住这一条：pts 必须来自时间，不能来自计数器。**

### 2.10 sleep_until 与漂移：怎么做一个不越走越慢的节拍器

视频要按固定帧率（比如 30fps = 每 33.33ms 一帧）节奏生产。最朴素的写法：

```cpp
while (running) {
    captureAndEncode();
    std::this_thread::sleep_for(33ms);   // 朴素但错误
}
```

错在哪？因为 `captureAndEncode()` 本身要花时间（比如 20ms），所以实际周期是 20+33 = **53ms**，相当于 19fps。而且这个误差**会随帧数累加**，这叫**漂移**（drift）。十分钟之后你的视频就慢了一大截。

正确做法是**绝对时间唤醒**（absolute wakeup），本课第 216~224 行：

```cpp
const auto waitNextFrame = [&]() {
    if (!isScreen) return;
    nextFrameAt += frameInterval;                 // ① 目标时间 = 上一次目标 + 一个间隔
    const auto after = Clock::now();
    if (nextFrameAt < after) nextFrameAt = after; // ② 已经落后了？就地对齐到"现在"
    std::this_thread::sleep_until(nextFrameAt);   // ③ 睡到那个绝对时刻
};
```

三句话分别解决了三个问题：

- ① 用的是**累加目标时刻**，不是"从现在再算 33ms"，所以误差不累积。
- ② **自我保护**：如果上一帧处理超时（比如编码花了 50ms），那么"目标时刻"已经落在过去了。这时候有两种选择——要么立刻返回继续拼命追（结果是把 CPU 累死也追不回来），要么把时间基准重新拉回"现在"。本课选后者：**认命，从现在重新起跑**。这是实时系统的正确心态：**过去的就让它过去，别试图弥补，补是补不回来的。**
- ③ `sleep_until` 接受一个绝对时间点，睡到那一刻自然醒。

为什么摄像头路径（`if (!isScreen) return;`）不需要节拍器？因为 `camera.nextFrame(1000)` 本身就最多阻塞 1000ms 等驱动给帧，**摄像头的硬件节奏已经在替我们打拍子了**。再加一个 sleep 就是画蛇添足。

### 2.11 timeBeginPeriod：Windows 的定时器精度开关

这是一个 Windows 独有的坑。Windows 默认的系统定时器粒度约为 **15.6ms**（64Hz），也就是说你让它睡 2ms，它可能实际睡了 15.6ms。

解决办法是在进程里请求更高精度：

```cpp
timeBeginPeriod(1);   // 请求 1ms 精度（还要 #include <windows.h> <mmsystem.h>，链接 winmm）
timeEndPeriod(1);     // 用完必须还回去
```

本课第 401~402 行和第 422~427 行成对使用，并且在 CMake 里链接了 `winmm`（§4.2 会看到）。

**为什么不用硬 sleep 而要这个？** 因为 `writeLoop` 里 `sleep_for(2ms)`、`videoLoop` 里 `sleep_until` 下一帧只有 33ms，如果粒度是 15.6ms，帧就会产生明显的抖动（jitter）——本来均匀的 33ms 变成 15.6/31.2/46.8 的锯齿，播放器端就会感觉"一卡一卡"的。

> 副作用提示：`timeBeginPeriod` 是**进程级全局**的，它会让整个进程的定时器都变准，代价是系统全局耗电略增（笔记本上更明显）。所以用完必须 `timeEndPeriod` 配对注销。本课的 `timerRaised` 标志就是用来保证成对注销的。

### 2.12 PImpl 惯用法：为什么 Impl 要藏在 .cpp 里

**PImpl**（Pointer to Implementation，指向实现的指针）：头文件里只声明 `struct Impl;` 和一个 `std::unique_ptr<Impl> impl_;`，真正的成员全写在 .cpp 里。

好处有三个，本课体会最深：

1. **编译防火墙**：`StreamPipeline.h` 里不需要 include `<thread>`、`<atomic>`、FFmpeg 头文件、各种采集器的头文件。谁 `#include "StreamPipeline.h"`，谁就只用 45 行的小头文件，**编译速度飞快**。
2. **依赖隔离**：`Core_pipeline` 内部的 ffmpeg include 路径是 `PRIVATE` 的（§4.2），所以使用方连 FFmpeg 的头文件在哪都不需要知道。
3. **ABI 稳定**：将来往 Impl 里加成员，`StreamPipeline` 的大小不变（永远是一个指针），库的使用者不用重新编译。

代价也有两个：**每次访问多一层指针间接**（可忽略）、**析构函数必须在 Impl 完整定义之后**才能写（所以必须在 .cpp 里写 `~StreamPipeline()`，否则 `unique_ptr<不完整类型>` 的析构编译不过）。第 10 课详细讲过这个坑，同学们应该还有印象。

### 2.13 sws_scale：把任意尺寸统一到目标分辨率

采集到的帧尺寸不一定等于配置的输出尺寸：桌面可能是 2560×1440，摄像头可能是 640×480，而你要推 1280×720。所以需要一个**缩放**步骤。

**libswscale** 是 FFmpeg 的图像缩放/像素格式转换库。核心三件套：

```cpp
SwsContext* ctx = sws_getContext(srcW, srcH, srcFormat,
                                 dstW, dstH, dstFormat,
                                 SWS_BILINEAR, nullptr, nullptr, nullptr);  // ① 造转换器
sws_scale(ctx, srcSlice, srcStride, 0, srcH, dstSlice, dstStride);          // ② 转换
sws_freeContext(ctx);                                                        // ③ 销毁
```

几个术语解释：

- **`linesize` / `stride`（跨距）**：图像一行**在内存里实际占多少字节**。注意它**可能大于** 宽度×每像素字节数，因为内存对齐（比如 GPU 传出来的图每行会补齐到 256 字节的倍数）。所以**永远要用 stride 去算行偏移，不能用 width**。本课第 198、200 行用的是 `width * 4`（因为是自己 malloc 的紧凑 BGRA，没有 padding，这样写没问题）。
- **`SWS_BILINEAR`（双线性插值）**：新像素的值由周围 4 个原像素按距离加权平均得到。画质一般但速度快。其他选项有 `SWS_POINT`（最近邻，最快最糊/锯齿）、`SWS_BICUBIC`（双三次，最慢最清晰）、`SWS_LANCZOS`（最好但最贵）。直播用 bilinear 是标准选择。
- **`SwsContext` 的复用**：`sws_getContext` 比较贵，**千万不要每帧都造一个**。本课用"上一次的源尺寸"缓存起来判断要不要重建（第 177~189 行），这是标准优化。

### 2.14 优雅启停：停止管线到底要干几件事

这是本课后半段的重点，先给同学们一个心智模型。**停止一条流水线，必须按顺序做四件事，顺序错了就丢数据：**

```
① 通知：把 running 置 false（告诉所有生产者"别再生产了"）
       ↓ 但是！这时候队列里还有没写完的数据
② 汇合：join 三个线程（等它们各自跑完当前这一次循环体，生产者会在退出前把编码器 flush 的包推进队列）
       ↓ 现在所有数据都在队列里了
③ 排空：writeLoop 必须把两个队列里剩下的包全部写完再退出
       ↓ 现在所有数据都在文件或网络里了
④ 收尾：flush 容器（写 FLV trailer）、close 编码器/采集器/输出、清空队列、注销定时器
```

注意这里的 **②③ 顺序陷阱**：如果第 ③ 步在第 ② 步之前（比如先把 writeLoop 退了再 join 生产者），那么生产者 flush 出来的最后几个包就没人写了，录出来的文件结尾会缺一小段。

本课的实现里，这个顺序是通过 **`writeLoop` 的循环条件**来保证的：

```cpp
while (running || !videoQueue.empty() || !audioQueue.empty()) {
```

即使 `running` 已经是 false，只要**两个队列里还有货**，write 线程就继续跑。而 Join 的顺序（第 446~451 行）是先 video/audio 后 write——**恰好保证了生产者先结束、消费者最后结束**。

我们把上面十四条串起来，得到一个完整的"出发前的行囊清单"。下面正式开始看代码。

---
## 第三章 diff 总览：这次提交到底加了什么

先给同学们一张"施工图"，让大家知道今天要盖哪几层楼。diff 一共动了 **7 个文件**：

| # | 文件 | 类型 | 行数 | 干什么 |
|---|---|---|---|---|
| 1 | `CMakeLists.txt`（顶层） | 修改 | +1 | 把新的 `src/core/pipeline` 目录挂进编译树 |
| 2 | `src/core/common/Common.h` | 修改 | +2 | `StreamConfig` 增加 `audioDeviceIndex` 字段 |
| 3 | `src/core/pipeline/CMakeLists.txt` | **新建** | 20 | 定义 `core_pipeline` 静态库，声明它依赖谁 |
| 4 | `src/core/pipeline/StreamPipeline.h` | **新建** | 45 | 对外接口：`PipelineStats` 结构体 + 类声明 |
| 5 | `src/core/pipeline/StreamPipeline.cpp` | **新建** | 443 | **本课主体**：Impl、三个线程循环、start/stop |
| 6 | `tests/CMakeLists.txt` | 修改 | +8 | 增加 `test_pipeline` 可执行文件 |
| 7 | `tests/test_pipeline.cpp` | **新建** | 298 | 端到端测试：参数校验 + 跑 3 秒 + 重复启停 |

画成依赖图，它是这样一层层叠起来的（`core_pipeline` 站在最上面，因为它认识所有人）：

```
                 ┌──────────────────────────┐
                 │      core_pipeline       │  ← 本次新建（StreamPipeline）
                 └────────────┬─────────────┘
        ┌──────────┬──────────┼──────────┬──────────┐
        ▼          ▼          ▼          ▼
  core_capture  core_encode  core_output  core_common
        │          │          │              ▲
        └──────────┴──────────┴──────────────┘
                 （capture/encode/output 也都依赖 common）
```

同学们注意这个图里体现的一条**架构原则**：**上层依赖下层，下层永远不知道上层的存在。**

`core_output`（第 10 课）完全不知道"谁在给它喂包"，它可能以为自己是单线程被调用的；`VideoEncoder`（第 9 课）也不知道自己会被塞进哪个线程。**正因为这些模块当初写得足够"无知"（不知道自己将来的运行环境），今天才能被直接拿来拼装，一行都不用改。** 这就是"低耦合"最实际的回报——请同学们在自己写代码时记住这个画面。

反过来说，`core_pipeline` 是**唯一一个知道全局的模块**：它知道有几个线程、知道视频要走哪几步、知道什么时候该关掉谁。**这种"知道全局的上层 + 无知的底层"结构，几乎是所有的自动驾驶框架、操作系统、编译器的共同范式。**

还有一个细节值得注意：这次 diff **没有动一行采集器、编码器、输出模块的代码**。新加的 `audioDeviceIndex` 只是被**传给了已经存在的** `AudioCapturer::open(type, deviceIndex, log)`——也就是说，**这个签名第 8 课就写好了，只是当时没人用得上**。这再次说明：接口预留是有价值的。

---

## 第四章 CMake 三处接线

每次新增一个模块，CMake 要改三处。这次也不例外，我们一个个看。

### 4.1 顶层 `CMakeLists.txt`：一行

```diff
 add_subdirectory(src/core/common)
 add_subdirectory(src/core/capture)
 add_subdirectory(src/core/encode)
 add_subdirectory(src/core/output)
+add_subdirectory(src/core/pipeline)
 add_subdirectory(demo/01-mini-pusher)
 add_subdirectory(src/app)
 add_subdirectory(tests)
```

注意 `add_subdirectory` 的**顺序**：`common → capture → encode → output → pipeline`。CMake 处理顺序大致是自上而下，虽然现代 CMake 的 target 依赖关系不严格要求先定义后使用（`target_link_libraries` 里提到的 target 哪怕是后面才定义也行），但**按依赖层次的顺序书写是良好的习惯**，人读起来一眼就能看出层级。

同学们可能会问：`core_pipeline` 放在 `core/output` 之后、`demo` 之前，是不是巧合？不是——它**必须在**所有被它依赖的模块之后（可读性），**必须在**使用它的 `tests` / `src/app` 之前（同样是可读性）。

### 4.2 `src/core/pipeline/CMakeLists.txt`：完整的 20 行

```cmake
add_library(core_pipeline STATIC
  StreamPipeline.cpp
  StreamPipeline.h
)

target_include_directories(core_pipeline
  PUBLIC ${CMAKE_CURRENT_SOURCE_DIR}/../..
  PRIVATE ${FFMPEG_INCLUDE_DIRS}
)

target_link_libraries(core_pipeline PUBLIC
  core_common
  core_capture
  core_encode
  core_output
)

if(WIN32)
  target_link_libraries(core_pipeline PUBLIC winmm)
endif()
```

我们一句一句讲。

**第一句：定义库。**

```cmake
add_library(core_pipeline STATIC
  StreamPipeline.cpp
  StreamPipeline.h
)
```

- **`STATIC`**：静态库（Windows 上是 `.lib`，Linux 上是 `.a`）。链接时它的代码会被**复制**到最终的可执行文件里。相对的是 `SHARED`（动态库 `.dll`/`.so`，运行时才加载）。本项目全是静态库，好处是发布时不用带一堆 dll，坏处是 exe 会大一些。
- **头文件也写进了源文件列表**：`StreamPipeline.h` 并不是必需的（不写也能编译），但**写进去有两大好处**：① IDE（Visual Studio / Qt Creator）会把它显示在工程树里，方便跳转查看；② 有些安装/打包规则会把列出的头文件一并打包。**这是项目的一贯风格，同学们保持一致即可。**

**第二句：包含目录。**

```cmake
target_include_directories(core_pipeline
  PUBLIC ${CMAKE_CURRENT_SOURCE_DIR}/../..
  PRIVATE ${FFMPEG_INCLUDE_DIRS}
)
```

这一句学问最大，请同学们务必弄懂 `PUBLIC` 和 `PRIVATE` 的区别（现代 CMake 的核心概念）：

| 关键字 | 含义 | 会不会传给链接我的人 |
|---|---|---|
| `PUBLIC` | 我自己要用，**用我的人也要用** | **会**传递 |
| `PRIVATE` | 只有我自己要用 | 不传递 |
| `INTERFACE` | 我自己不用，**但用我的人要用** | 会传递 |

套到本例：

- `${CMAKE_CURRENT_SOURCE_DIR}/../..` 是 `src/` 的父目录。有了它，`StreamPipeline.cpp` 里才能写 `#include "core/pipeline/StreamPipeline.h"` 这种**带完整前缀**的写法。这个包含路径用了 `PUBLIC`，意味着**凡是链接 `core_pipeline` 的人（比如 `test_pipeline`），自动也获得了这个包含路径**——所以测试文件里才能直接 `#include "core/pipeline/StreamPipeline.h"` 而不用再配一遍路径。**这就是依赖传播（transitive dependency）。**
- `${FFMPEG_INCLUDE_DIRS}` 用了 `PRIVATE`，因为 FFmpeg 的头文件只在 `StreamPipeline.cpp` 内部用到（`swscale.h`）。**用样子本电流说明**：正因为头文件 `StreamPipeline.h` 里**一个 FFmpeg 类型都没露出来**（PImpl 的功劳！），我们才有可能把它设为 PRIVATE。**PImpl 不只是编译加速技巧，它直接决定了你的依赖能不能"藏住"。** 这是第二章 §2.12 讲的"编译防火墙"在构建系统上的体现。

**第三句：链接依赖。**

```cmake
target_link_libraries(core_pipeline PUBLIC
  core_common
  core_capture
  core_encode
  core_output
)
```

四个被依赖的库，全部 `PUBLIC`。为什么不用 `PRIVATE`？因为虽然使用者不直接调用 `VideoEncoder` 的方法，但 CMake 的 `PUBLIC` 会把这些库**继续往上传播**，最终在链接 `test_pipeline.exe` 的时候自动把四个库的 `.lib` 全带上。如果这里写 `PRIVATE`，链接 exe 时就会报"找不到 VideoEncoder 的符号"。

一句话总结：

> **记得住的标准答案：库的私有依赖用 `PRIVATE`（如 FFmpeg include），需要随接口暴露的用 `PUBLIC`（如 include root 和兄弟库）。**

**第四句：Windows 专用库。**

```cmake
if(WIN32)
  target_link_libraries(core_pipeline PUBLIC winmm)
endif()
```

**`winmm`**（Windows Multimedia）：提供 `timeBeginPeriod` / `timeEndPeriod` 这两个函数（第二章 §2.11 讲过）。

为什么要用 `if(WIN32)` 包起来？因为这两个函数在 Linux/macOS 上根本不存在，不加条件会链接失败。虽然有 `{awt}`——不，就算本项目目前只在 Windows 上跑，同学们也要养成这个习惯：**平台相关的东西一定要用条件包起来**，这是可移植性的第一道防线。

细心的同学会问：`winmm` 用 `PUBLIC` 合适吗？严格说它只是实现细节，`PRIVATE` 更准确（使用者不需要自己调 `timeBeginPeriod`）。这里写成 `PUBLIC` 不会出错，但会略微扩大依赖面。**留给第十七章 Q7 讨论。**

### 4.3 `tests/CMakeLists.txt`：加一个测试可执行文件

```diff
 add_executable(test_output PRIVATE ... )
 target_link_libraries(test_output PRIVATE
   core_output
 )
 
+add_executable(test_pipeline
+  test_pipeline.cpp
+)
+
+target_link_libraries(test_pipeline PRIVATE
+  core_pipeline
+)
+
 add_executable(test_audio_capture
   test_audio_capture.cpp
 )
```

这里有两个细节：

1. **`target_link_libraries(test_pipeline PRIVATE core_pipeline)`**：`test_pipeline.cpp` 里写了一堆 FFmpeg 调用（`avformat_open_input` 等），但 CMake 里**没有**给它显式链接 FFmpeg！为什么能编过？

   答案是：**靠传播**。`core_pipeline` 虽然把 `${FFMPEG_INCLUDE_DIRS}` 设成了 PRIVATE，但 `core_output` 或 `core_common` 里必然有某个库把 FFmpeg 设成了 `PUBLIC`（编译 09/10 课时讲过 `core_encode` 把 FFmpeg include 设为 PUBLIC 的那个"小失误"反而帮了忙）。依赖是**沿链路传递**的，`test_pipeline` 作为末端节点，继承了整条链上的 include 路径和库。

   同学们，这里我要泼一盆冷水：**靠"碰巧传播过来的"依赖能不能编译，不是好设计**。正规做法是让 `test_pipeline` 明确声明自己需要的东西。但因为测试文件确实用到了 FFmpeg 的 demux API，**建议在 real 项目里给 tests 也加一行 visibility**（第十七章 Q7 会让你动手改）。

2. **测试 exe 本身用 `PRIVATE`**：因为不会再有人去"链接一个测试程序"，没必要往外传播。

**给全体测试排个队**，我们看看项目到现在的测试清单：

| 测试 | 验的是什么 | 第几课 |
|---|---|---|
| `test_common` | `BoundedQueue` 七组行为 | 3 |
| `test_screen_capture` | DXGI/GDI 截屏 | 4 |
| `test_camera_capture` | DirectShow 枚举与取帧 | 5 |
| `test_camera_convert` | YUY2/MJPG → BGRA | 6 |
| `test_audio_capture` | WASAPI 采集与 pts 连续性 | 8 |
| `test_encode` | H.264/AAC 编码正确性 | 9 |
| `test_output` | FLV 封装与多目标分发 | 10 |
| **`test_pipeline`** | **端到端全流程** | **11（本课）** |

这条链非常漂亮：**每一课结束都有一个"能被运行、能被人眼验收"的东西**。这是整个项目最值得同学们模仿的工程习惯。

---

## 第五章 Common.h：一个新的配置项 audioDeviceIndex

这次对 `Common.h` 的改动只有两行，但它讲了一个很好的"接口演进"故事。

```diff
 struct StreamConfig {
     ...
     int cameraIndex = 0;
 
+    int audioDeviceIndex = -1;
+
     int width = 1920;
     int height = 1080;
     int fps = 30;
```

现在的 `StreamConfig` 全貌（第 3 课定义，本课加了一行）：

```cpp
struct StreamConfig {
    SourceType sourceType = SourceType::Screen;          // 画面来源：屏幕 / 摄像头
    AudioSourceType audioType = AudioSourceType::SystemAudio;  // 声音来源：系统声音 / 麦克风

    int cameraIndex = 0;          // 第几个摄像头
    int audioDeviceIndex = -1;    // ← 本次新增：第几个音频设备

    int width = 1920;
    int height = 1080;
    int fps = 30;
    int videoKbps = 4500;
    int audioKbps = 128;

    int sampleRate = 48000;
    int channels = 2;

    std::vector<StreamTarget> targets;
};
```

### 5.1 为什么默认值是 -1

`-1` 是一个**约定俗成的"未指定"哨兵值**（sentinel value）。它在用户看来就是"我不挑，你给我用默认的那个"。

这个约定会一路传下去，最终到达第 8 课写的 `AudioCapturer::open(type, deviceIndex, log)` 内部。我们去看一眼它是怎么处理 -1 的（复习第 8 课的内容）：

```cpp
// AudioCapturer.cpp 内部（简化）
if (deviceIndex >= 0) {
    // 用户指定了，用枚举出来的第 deviceIndex 个设备
    devices->Item(deviceIndex, &device);
} else {
    // 用默认设备
    enumerator->GetDefaultAudioEndpoint(eRender/eCapture, eConsole, &device);
}
```

同学们看，** `-1` 这个小小的约定，把"指定设备"和"使用默认设备"两条路径压缩成了一个参数**，调用方不需要学第二个函数。这种设计叫 **"用哨兵值表达可选语义"**，在 C/C++ 库里非常常见（比如 `std::string::npos`、`waitpid` 的 `-1`）。

当然它也有缺点：**类型系统帮不了你**——某天手滑传了个 `-2`，编译照样过，行为未定义。现代 C++ 更推荐用 `std::optional<int>` 来表达"可能有也可能没有"，这是第 3 课 §2.4 讲 `std::optional` 时的伏笔。改成 `std::optional<int>` 会强迫调用方写 `std::nullopt` 而不是 `-1`，可读性更好。**这是一个留给同学们的改进练习（第十七章 Q6）。**

### 5.2 为什么偏偏是"现在"才加这个字段

这是个很有教育意义的观察。回想第 8 课——写 `AudioCapturer` 的时候，`audioDeviceIndex` 根本不存在，当时测试里只能"用默认设备"抓麦。可奇怪的是 `open()` 的签名里**早就有** `deviceIndex` 参数了，不是吗？

是的。**第 8 课的 author 已经想到将来要支持多设备，于是签名里预留了；但没人调用它，所以没人关心默认值该是多少。** 直到今天——我们第一次要"从上层把用户的选择一路传下去"——才发现 **`StreamConfig` 里缺这个字段**。

这就是"总装"的价值之一：

> **零件阶段的设计漏洞，只有在集成的时候才会暴露。** 你永远可以写单元测试证明自己的零件没问题，但没法证明零件之间的接口是完整的。

所以同学们不要怕"改契约"。`StreamConfig` 是第 3 课签的合同，到今天改了。合同不是不能改，而是**改合同必须让所有人同步**——这次改动只加了带默认值的字段，所以老代码一行都不用动（这就是"给新字段默认值"的另一个好处：**向后兼容**）。

---

## 第六章 StreamPipeline.h：整机的"操作面板"

现在正式进入本课主角。头文件只有 45 行，我们先看全貌，再逐段讲。

```cpp
#pragma once

#include "core/common/Common.h"

#include <cstdint>
#include <memory>
#include <string>

struct PipelineStats {
    int64_t videoFramesCaptured = 0;
    int64_t videoPacketsEncoded = 0;
    int64_t videoFramesDropped = 0;
    int64_t audioChunksCaptured = 0;
    int64_t audioPacketsEncoded = 0;

    int connectedTargets = 0;
    int totalTargets = 0;

    double elapsedSeconds = 0.0;
};

class StreamPipeline {
public:
    StreamPipeline();
    ~StreamPipeline();

    StreamPipeline(const StreamPipeline&) = delete;
    StreamPipeline& operator=(const StreamPipeline&) = delete;

    bool start(const StreamConfig& config, LogCallback log);

    void stop();

    bool isRunning() const;

    PipelineStats stats() const;

    int reconnectDisconnected();

    const std::string& lastError() const;

private:
    struct Impl;
    std::unique_ptr<Impl> impl_;
};
```

### 6.1 头文件包含：只有 4 行，这是刻意的

```cpp
#pragma once

#include "core/common/Common.h"

#include <cstdint>
#include <memory>
#include <string>
```

同学们对比一下 `.cpp` 文件——那里 include 了一大堆：`<windows.h>`、`<thread>`、`<atomic>`、`<mutex>`、`CameraCapturer.h`、`VideoEncoder.h`、`swscale.h`……**全都藏在了 .cpp 里。**

头文件只暴露了：

- `Common.h`（因为 `PipelineStats` 用到了 `std::string`，而接口里出现了 `StreamConfig` 和 `LogCallback`）；
- `<cstdint>`（`int64_t`）、`<memory>`（`unique_ptr`）、`<string>`（`std::string`）。

**这就是第二章讲的"编译防火墙"**。带来的实惠是：任何想用 `StreamPipeline` 的人（尤其是 UI 层），include 这个文件只需要 45 行的编译量，**不会把 `<windows.h>` 那上万个宏定义带进自己的编译单元**。

> 小知识：`<windows.h>` 里定义了 `#define max(a,b)` 这类宏，一旦泄漏到你的代码里，`std::max` 就会被宏替换炸掉，报错信息极其难懂。**把 `<windows.h>` 隔离在 .cpp 里，是 Windows C++ 项目的一条保命法则。** 本项目的每一层 CMake 都严格区分 PUBLIC/PRIVATE，正是为此。

### 6.2 PipelineStats：给外人看的仪表盘

```cpp
struct PipelineStats {
    int64_t videoFramesCaptured = 0;    // ① 采到了多少帧
    int64_t videoPacketsEncoded = 0;    // ② 编出了多少个视频包
    int64_t videoFramesDropped = 0;     // ③ 丢了多少帧
    int64_t audioChunksCaptured = 0;    // ④ 抓到了多少块音频
    int64_t audioPacketsEncoded = 0;    // ⑤ 编出了多少个音频包

    int connectedTargets = 0;           // ⑥ 当前连着几路
    int totalTargets = 0;               // ⑦ 一共配了几路

    double elapsedSeconds = 0.0;        // ⑧ 跑了多久（秒）
};
```

这 8 个字段就是整个系统的**仪表盘**（dashboard）。设计上有几点讲究：

**（1）为什么这是一份"快照"而不是一堆 getter？**

因为**跨线程的一致性**。假设 UI 线程每隔 500ms 想显示一次状态，如果它分别调 8 个 getter，那么这些 getter 读到的值来自**不同的时刻**（中间隔着好几百毫秒，数据在一直变），显示出来的画面可能自相矛盾（比如"编码包数 > 采集帧数"，因为读到的是一前一后两次采样的混合）。

一次性 8 个字段一起读出来，虽然不能保证是同一瞬间（没有加锁），但**时间窗口极小**（8 个 atomic load，几十纳秒），足以认为是一致快照。这是并发监控的实用做法。

**（2）为什么 `elapsedSeconds` 用 `double`？**

因为秒数天然有小数（3.247 秒）。而其他用整数的原因是——**帧数、包数这类"计数"永远不会是小数，而且可能非常大**（推一小时的 1080p60 就是 216000 帧），`int64_t` （最大值约 9.2×10^18）可以**稳定跑几百年不会溢出**。

> 顺带提一下溢出常识：如果用 `int32_t`（最大 21.4 亿），1080p60 推流大约 **4.1 天**就会溢出变成一个负数——这就是经典的"直播软件推几天就崩"的 bug 之一。**涉及累计数，一律 `int64_t`，不要省。**

**（3）为什么把它放在头文件而不是 Impl 里？**

因为它要作为**函数返回值**交给调用方：

```cpp
PipelineStats stats() const;
```

调用方拿到的是一个**值**（拷贝出来的快照），不是引用。这意味着：

- 调用方随便改，不影响管线内部（**值语义天然安全**）；
- Impl 内部的那些 atomic 成员永远藏得住。

这就是"用简单的值对象做跨层传递"的典型手段，比返回裸指针安全一万倍。

### 6.3 类声明：五个 joins 和六个 gets

```cpp
class StreamPipeline {
public:
    StreamPipeline();
    ~StreamPipeline();

    StreamPipeline(const StreamPipeline&) = delete;
    StreamPipeline& operator=(const StreamPipeline&) = delete;

    bool start(const StreamConfig& config, LogCallback log);

    void stop();

    bool isRunning() const;

    PipelineStats stats() const;

    int reconnectDisconnected();

    const std::string& lastError() const;

private:
    struct Impl;
    std::unique_ptr<Impl> impl_;
};
```

**① 拷贝构造与赋值为什么必须 delete？**

```cpp
StreamPipeline(const StreamPipeline&) = delete;
StreamPipeline& operator=(const StreamPipeline&) = delete;
```

因为这个类内部有**三个 `std::thread`、七个 heavy 的采集/编码对象、`unique_ptr`**。拷贝一份意味着什么？两个对象共享同一批采集器和编码器，析构时 double-free（第二次释放已经释放的资源），或者 A 停止了但 B 还在运行导致 use-after-free。

> **`= delete` 是比"private 里声明但不实现"更现代、更清晰的做法**：编译器会在你误写 `StreamPipeline b = a;` 时给出明确报错"C2280: 尝试引用已删除的函数"，而不是"链接错误"。C++11 之后所有"不可拷贝资源"的类都应该这么写。

注意：本项目所有具体资源的类（`VideoEncoder`、`AudioEncoder`、`StreamOutput`、`AudioCapturer`…）都这么干。**一个项目里的资源类风格统一，是最容易被忽视的工程素养。**

**② 六个公开方法的语义**

| 方法 | 语义 | 什么时候调 | 线程安全吗 |
|---|---|---|---|
| `start(config, log)` | 校验参数 → 打开所有模块 → 起三条线程 | 用户点"开始推流" | **只能由主线程调**，不可并发；内部会先 `stop()` |
| `stop()` | 通知 → join → 排空 → 关资源 | 用户点"停止"，或析构时 | 幂等（可以调多次） |
| `isRunning()` | 一句话看是否是运行中 | UI 刷新按钮状态 | 安全（atomic） |
| `stats()` | 取当前统计快照 | UI 每 500ms 刷一次 | 安全（atomic load） |
| `reconnectDisconnected()` | 让输出模块重连断开的目标 | 网络波动后 | 单次调用安全（底层有 internal 状态） |
| `lastError()` | 取上次失败原因 | `start` 返回 false 之后 | 返回 const 引用，注意生命周期 |

**③ `const std::string& lastError()` 为什么要返回引用？**

避免一次字符串拷贝。但同学们要注意这里的**生命周期约定**：返回的是内部成员的引用，**调用方必须在 pipeline 活着的时候用**。如果你这样写就危险了：

```cpp
const char* getMsg() {
    StreamPipeline p;
    return p.lastError().c_str();    // ✗ 危险！p 出了作用域就析构，指针悬空
}
```

这是一个典型的**悬垂引用**（dangling reference）陷阱。正确的用法是立刻用掉，或者拷到自己手里（`std::string s = p.lastError();`）。

测试里就是这么干的（第 710 行）：

```cpp
std::printf("  失败原因: %s\n", pipeline.lastError().c_str());
```

`pipeline` 在作用域内活着，`printf` 用完就结束——安全。

**④ 为什么接口设计成"外部只管 start/stop/stats"？**

这就是**封装**（encapsulation）的威力。UI 层（将来 Qt 写的窗口）**完全不需要知道有几个线程、队列多大、pts 怎么算**。它只需要：

```cpp
pipeline_.start(config, [this](const std::string& s){ appendLog(s); });
// ... 定时器里
auto s = pipeline_.stats();
updateUI(s.videoFramesCaptured, s.elapsedSeconds);
// ... 关窗口时
pipeline_.stop();
```

**这就是为什么我们把 `start` 的参数设计成一个完整的 `StreamConfig`**：UI 一次把全部设置打包交过来，管线内部爱怎么用怎么用。将来加功能（比如硬件编码器开关、水印、录制文件夹）只需要给 `StreamConfig` 加字段，**接口签名一个字不变**。

**这叫"用数据结构替代参数列表"**——当你的函数塞到第 4 个参数还觉得不够用时，就该考虑打包了。今天 `StreamConfig` 已经有 12 个字段，如果当初没打包成结构体，`start()` 就得写成 12 个参数的函数，改一次接口全部调用点跟着改——无法维护。

---

## 第七章 StreamPipeline.cpp（上）：Impl 的家底与两个工具方法

.cpp 一共 443 行。同学们先看前 206 行（含 include、Impl 成员、缩放工具），这部分是"家底"，后面三个线程循环都建立在它之上。

### 7.1 include 区（第 1~24 行）

```cpp
#include "core/pipeline/StreamPipeline.h"

#include "core/capture/AudioCapturer.h"
#include "core/capture/CameraCapturer.h"
#include "core/capture/CameraFrameConverter.h"
#include "core/capture/ScreenCapturer.h"
#include "core/common/BoundedQueue.h"
#include "core/encode/AudioEncoder.h"
#include "core/encode/VideoEncoder.h"
#include "core/output/StreamOutput.h"

#include <windows.h>

#include <mmsystem.h>

extern "C" {
#include <libswscale/swscale.h>
}

#include <algorithm>
#include <atomic>
#include <chrono>
#include <mutex>
#include <thread>
#include <vector>
```

按惯例，我们把它们分门别类：

| 类别 | 内容 | 备注 |
|---|---|---|
| **自己的头文件** | `StreamPipeline.h` | **永远第一个 include** |
| **本项目其他模块** | 4 个采集器相关 + `BoundedQueue` + 2 个编码器 + `StreamOutput` | 本课要"攒 loosely"的全部零件 |
| **平台头** | `<windows.h>`, `<mmsystem.h>` | `<mmsystem.h>` 提供 `timeBeginPeriod` |
| **第三方 C 库** | `<libswscale/swscale.h>` | 必须 `extern "C"` 包裹 |
| **标准库** | `<algorithm>`, `<atomic>`, `<chrono>`, `<mutex>`, `<thread>`, `<vector>` | 并发与时间工具集 |

几个需要解释的点：

**（1）为什么自己的头文件要放在最前面？**

这是 Google / LLVM 等大型项目的通用规范，原因很诚实：**如果 `StreamPipeline.h` 自己依赖某个头而忘了 include，放在最前面能立刻暴露这个问题**。如果放在最后，前面的 include 可能"碰巧"把它需要的符号带进来了，于是你的头文件看起来能独立使用，实际上换个地方 include 就编译失败——这种 bug 极难查。

**（2）为什么 FFmpeg 的头文件要 `extern "C"` 包裹？**

FFmpeg 是纯 C 写的库。C++ 为了支持函数重载（同名函数不同参数），会对**函数名做装饰**（name mangling）：C++ 里的 `void foo(int)` 编译后的符号可能是 `?foo@@YAXH@Z`（MSVC）或 `_Z3fooi`（GCC），而 C 编译器产生的符号就是朴素的 `foo`。

如果你不用 `extern "C"` 告诉 C++ 编译器"这段声明里的函数是 C 风格的、按 C 的规矩产生符号"，那么链接器会去找 `?sws_getContext@@YAPEA...` 这种带装饰的符号，而 FFmpeg 的 .lib 里只有朴素的 `sws_getContext`——于是报 **LNK2019: 无法解析的外部符号**。

> **这是 C++ 调用 C 库必背的一条知识。** 同学们在本项目的每一见到 `extern "C" { #include <libav...> }` 的地方，都是这个原因。

**（3）`<chrono>`、`<thread>`、`<atomic>`、`<mutex>` 这四个新头，分别提供了什么？**

- `<chrono>`：`steady_clock`（第二章 §2.6）、`duration`、`milliseconds`、`microseconds`、`duration_cast`。
- `<thread>`：`std::thread`、`std::this_thread::sleep_for` / `sleep_until`。
- `<atomic>`：`std::atomic<T>`。
- `<mutex>`：`std::mutex` 和 `std::lock_guard`（本课用来保护日志回调）。

这四个头凑在一起，基本就是一个"多线程时间驱动流水线"的最小工具集了。同学们可以按图索骥：**C++ 标准库里跟并发相关的头就那么几个，全部认识一遍，并不需要多久。**

### 7.2 匿名命名空间里的时钟别名（第 26~37 行）

```cpp
namespace {

using Clock = std::chrono::steady_clock;

int64_t msSince(Clock::time_point from, Clock::time_point now)
{
    return std::chrono::duration_cast<std::chrono::milliseconds>(now - from).count();
}

}
```

**第 1 层理解：`using Clock = std::chrono::steady_clock;`**

这是个**类型别名**（type alias，用 `using` 的现代写法，等价于 `typedef std::chrono::steady_clock Clock;`）。这样写的好处是**嘴上说 Clock，心里时刻记着"这是单调时钟"**；将来哪天你想换成别的时钟，只改这一行。

同学们看后面的代码会发现，我们到处写 `Clock::now()` 而不是 `steady_clock::now()` ——这不只是为了少打字，**它把"这个项目用哪种时钟"这个决策，收敛到了唯一一处**。这是非常好的仙——不，非常好的工程习惯，叫 **"单点决策"（single source of truth）**。

**第 2 层理解：`msSince` 这个工具函数**

```cpp
int64_t msSince(Clock::time_point from, Clock::time_point now)
```

注意它的参数顺序：`(起点, 现在)`。两个参数都用 `Clock::time_point` 类型，明确、不会传错。

函数体是 `<chrono>` 的标准三步：

```cpp
std::chrono::duration_cast<std::chrono::milliseconds>(now - from).count()
//  ① now - from                        → 得到一个 duration（时长，单位可能是微秒或纳秒）
//  ② duration_cast<milliseconds>(...)  → 把它换算成"毫秒"为单位（向下取整）
//  ③ .count()                          → 取出里面的整数值
```

> **为什么要显式 `duration_cast`？** 因为 `steady_clock` 在不同平台上的精度不一样（Windows 上是 100ns，Linux 上可能是 1ns）。`<chrono>` 的设计哲学是**让单位显式化**：两个不同精度的 duration 之间**不允许隐式转换**，必须显式 `duration_cast`。这样编译器就能在你想"偷偷换个单位"的时候拦住你。**这是 `<chrono>` 系列 API 比古老的 `GetTickCount()` / `clock()` 好用得多的主要原因：单位写在你脸上，不可能搞错。**

它在本课哪里被用到？只有一处——`stats()` 里算"跑了多久"（第 491~492 行）。虽然只用了一次，仍然值得抽成函数：**因为 `duration_cast` 那一行实在太长了，抽出来 `stats()` 才能保持清爽。**

**第 3 层理解：为什么放在匿名 namespace 里？**

```cpp
namespace {
    // ...
}
```

**匿名命名空间**（anonymous namespace）：里面的符号具有**内部链接性**（internal linkage），只在本翻译单元（本 .cpp）可见，别的 .cpp 看不到，也就不可能跟别人的 `Clock` 或 `msSince` 冲突（`LNK2005: 已在 xxx.obj 中定义`）。

C++ 里它是取代老式 `static` 关键字的推荐做法（`static` 不能修饰类型别名，作用范围也有限）。**写 demo 无所谓，写库必须用这个习惯。**

### 7.3 Impl 结构体：家底清单（第 39~125 行）

现在到最核心的部分了。我把整个 Impl 结构体的成员按**职责分组**，逐组讲解。

```cpp
struct StreamPipeline::Impl {
    // ── 组 1：配置与日志 ──────────────────────────────────
    StreamConfig config;
    LogCallback log;
    std::string lastError;

    std::mutex logMutex;

    // ── 组 2：运行状态 ────────────────────────────────────
    std::atomic<bool> running{false};

    // ── 组 3：采集侧（三条数据源）────────────────────────
    ScreenCapturer screen;
    CameraCapturer camera;
    CameraFrameConverter cameraConverter;
    AudioCapturer audio;

    // ── 组 4：编码侧 ──────────────────────────────────────
    VideoEncoder videoEncoder;
    AudioEncoder audioEncoder;

    // ── 组 5：输出侧 ──────────────────────────────────────
    StreamOutput output;

    // ── 组 6：三条线程 ────────────────────────────────────
    std::thread videoThread;
    std::thread audioThread;
    std::thread writeThread;

    // ── 组 7：两个队列 ────────────────────────────────────
    BoundedQueue<AVPacketPtr> videoQueue{24};
    BoundedQueue<AVPacketPtr> audioQueue{256};

    // ── 组 8：图像缩放上下文 ──────────────────────────────
    SwsContext* scaleContext = nullptr;
    int scaleSrcWidth = 0;
    int scaleSrcHeight = 0;

    // ── 组 9：统计计数器 ──────────────────────────────────
    std::atomic<int64_t> videoFrames{0};
    std::atomic<int64_t> videoPackets{0};
    std::atomic<int64_t> videoDropped{0};
    std::atomic<int64_t> audioChunks{0};
    std::atomic<int64_t> audioPackets{0};

    // ── 组 10：时间基准与定时器 ───────────────────────────
    Clock::time_point clockStart;

    bool timerRaised = false;

    // ── 组 11：三个线程主函数 + 工具方法声明 ─────────────
    void videoLoop();
    void audioLoop();
    void writeLoop();

    void emit(const std::string& message)
    {
        std::lock_guard<std::mutex> lock(logMutex);
        if (log)
            log(message);
    }

    LogCallback guardedLog()
    {
        return [this](const std::string& message) { emit(message); };
    }

    VideoFramePtr scaleToTarget(const VideoFramePtr& frame);
    void releaseScale()
    {
        if (scaleContext) {
            sws_freeContext(scaleContext);
            scaleContext = nullptr;
        }
        scaleSrcWidth = 0;
        scaleSrcHeight = 0;
    }
};
```

#### 组 1：配置与日志

```cpp
StreamConfig config;
LogCallback log;
std::string lastError;

std::mutex logMutex;
```

**（1）为什么 `config` 要拷一份进来？**

因为 `start(config, log)` 的参数是 `const StreamConfig&`（引用），函数返回之后调用方的对象还在不在鬼知道。存一份**值**，管线内部就永远有自己的一份配置，线程里随便读。

这里有个隐藏的好处：**线程可以安全地读 `config`，因为它启动之后就再没被改写过**。这就避免了"配置热更新"带来的加锁问题。将来要做真正的动态改码率，就得改成 atomic 或加锁——这是一个已知的**技术债**（第十七章 Q5）。

**（2）`logMutex` 为什么必须存在？**

这是本课**最容易被人忽视、但一旦去掉立刻出事**的一个设计。

回忆一下：三个人会同时调日志——

| 谁在调 | 什么时候调 |
|---|---|
| `videoThread` | 编码失败、缩放失败、摄像头转换失败 |
| `audioThread` | 编码器内部的日志（虽然目前 audioLoop 自己不打） |
| **`AudioCapturer` 内部的 WASAPI worker 线程**（第 8 课） | 设备丢失、重采样 |
| `writeThread` | 写失败、断开重连 |
| 调用方（UI 线程） | 也可能同时打别的日志 |

如果直接 `log(message)` 而这个 `log` 最终指向一个往控制台 `printf` 的函数，那么**多个线程交错调用会导致输出字符错乱**（有时甚至 `std::bad_function_call`，如果另一个线程正在改 `log`）。

而且更糟的是 `log` 这个 `std::function` 成员本身——**它不是线程安全的**（读可以，并发读没人写时也行，但 future 版本的 UI 可能会替换它）。用 `logMutex` 把"读 log + 调用 log"包成一个临界区，**从今天起就彻底避免了这一类 bug**。

> **设计原则**：**永远假设你的日志回调会被多线程同时调用，然后在这里一次性解决。** 不要把这个责任甩给调用方——因为调用方（UI 程序员）绝对想不到。

**（3）`lastError` 为什么不用 atomic / 不加锁？**

因为它只在 `start()` 里被写（`start` 假定外部串行调用），在读的时候（ `stats` / `lastError()`）只被调用方读。这是一个合理的简化，但**严格说不是线程安全的**（如果 UI 线程在另一个线程 `start()` 的同时调 `lastError()`，理论上可能有竞争）。第十七章 Q5 会让你补上。

#### 组 2：运行状态

```cpp
std::atomic<bool> running{false};
```

这就是第二章 §2.5 讲过的**总开关**。三条线程每轮循环都检查它。

为什么是 `{false}` 而不是 `= false`？这是 atomic 的初始化习惯写法——`std::atomic<bool> x{false}` 是**直接列表初始化**，绕开了拷贝初始化的坑（`std::atomic<bool> x = false;` 也能编，但老 compiler 上会有转换警告）。**推荐统一用花括号。**

#### 组 3~5：七个业务对象

```cpp
ScreenCapturer screen;
CameraCapturer camera;
CameraFrameConverter cameraConverter;
AudioCapturer audio;

VideoEncoder videoEncoder;
AudioEncoder audioEncoder;

StreamOutput output;
```

注意它们都是**值成员**（不是指针），这意味着：

- **构造顺序 = 声明顺序**，`Impl` 构造时它们就被构造好；
- **析构顺序 = 逆序**，`Impl` 析构时自动析构；
- **不需要 `new` / `delete`**，没有内存泄漏风险。这是 RAII 的最大好处（第 9 课反复强调过）。

这七个对象每一个都有自己的析构函数做清理（三部 ——`ScreenCapturer::~ScreenCapturer()` 会 `close()`，`VideoEncoder::~VideoEncoder()` 会 `close()`）。所以哪怕 `stop()` 忘了写某个 close，析构兜底也不会泄漏。**这叫"双保险"，是本项目的优良传统。**

> **`camera` 和 `cameraConverter` 的关系**：摄像头路径要两步——`camera.nextFrame()` 取裸帧（`CameraRawFrame`，可能是 YUY2/MJPG），`cameraConverter.convert()` 转成统一的 BGRA `VideoFrame`。这是第 5、6 课的内容。本课只是把它们串起来。

#### 组 6：三条线程

```cpp
std::thread videoThread;
std::thread audioThread;
std::thread writeThread;
```

三个默认构造的 `std::thread`——**此时还没有关联任何真正的线程**（第二章 §2.4 讲过，默认构造的 thread 是不可 joinable 的）。

这个"空"的状态非常重要，它是我们判断"管线有没有启动过"的依据：

```cpp
if (!impl_->running && !impl_->videoThread.joinable() && ...)
```

（第十二章 `stop()` 会用到。）

第二个好处：它们是成员而不是局部变量，所以生命周期跟随 `Impl`，`stop()` 里做完 join 之后可以安全地交给 `std::thread` 的析构处理。

#### 组 7：两个队列

```cpp
BoundedQueue<AVPacketPtr> videoQueue{24};
BoundedQueue<AVPacketPtr> audioQueue{256};
```

这是第 3 课的 `BoundedQueue` **第一次实例化成真实类型**。同学们注意：

- 模板参数 `T = AVPacketPtr = std::unique_ptr<AVPacket, AVPacketDeleter>`（第 9 课 `EncoderCommon.h` 定义的别名）。
- **队列里存的是智能指针，不是包本身**。往队列里 push 一次的成本 = 8 字节的指针移动 + 一次原子操作的对象移动——**零拷贝**。
- `unique_ptr` 是**移动语义**的——它不能拷贝！但 `BoundedQueue::push` 恰好用的是 `std::move(item)`，`T item` 按值接收再 `std::move` 进 deque，**完美配合**。同学们体会一下这里没有被明说的设计默契：**当初第 3 课把 `push` 写成"按值接收 + move 进容器"，今天才使得往队列里放 `unique_ptr` 成为可能**。如果当初写的是 `const T& item` + `queue_.push_back(item)`，今天这段代码就编译不过了。这叫**面向未来设计**。

容量 24 和 256 的理由在 §2.3 已经算过了。这里补一句实践建议：

> **视频队列容量 ≈ GOP 大小比较合理。** 本项目的 GOP 默认是 fps×2 = 60（`gopSize` 默认 0 会被 `VideoEncoder` 处理成 `fps * 2`），24 大约是一个非常保守值——意味着缓冲不超过半秒，网络抖动超过 0.8 秒就会开始丢包。**直播宁可丢一点也不要延迟累积，这是正确的取舍**（会议软件则相反，宁可延迟也不丢）。

#### 组 8：缩放上下文

```cpp
SwsContext* scaleContext = nullptr;
int scaleSrcWidth = 0;
int scaleSrcHeight = 0;
```

这是一处 C 风格的资源（`SwsContext*` 是裸指针），所以：

- 必须手动 `sws_freeContext` 释放（见 `releaseScale`）；
- 缓存了上一次的源尺寸，用来判断要不要**重建上下文**（第二章 §2.13）。

为什么**三个裸指针周围要配两个 int？** 因为 `SwsContext` 一旦创建，它的输入尺寸就固定了。**当桌面分辨率变化（比如用户插拔外接显示器）时，源尺寸会变，旧上下文必须销毁重建**——如果不判断，直接拿旧的 context 去 scale 新尺寸的图，轻则画错，重则内存越界崩溃。

这是一个典型的 **"缓存 + 失效检测"** 模式：

```
有上下文吗？ ── 没有 ──→ 创建
   │
   有 ──→ 源尺寸变了吗？ ── 变了 ──→ 释放 + 重建
                │
               没变 ──→ 直接复用（快速路径）
```

同学们可以在自己的代码里大量套用这个模式，它适用于任何"创建代价高但可以复用"的资源（数据库连接、SSL 上下文、线程池……）。

#### 组 9：五个统计计数器

```cpp
std::atomic<int64_t> videoFrames{0};
std::atomic<int64_t> videoPackets{0};
std::atomic<int64_t> videoDropped{0};
std::atomic<int64_t> audioChunks{0};
std::atomic<int64_t> audioPackets{0};
```

五个计数器，三条写线程（videoLoop 写前三个，audioLoop 写后两个），一条读线程（调用方，`stats()`）。全靠 atomic，一个锁都不需要。

注意 **缺一个 `audioChunksDropped`**——音频没有丢帧统计。为什么不丢？因为 **音频队列 256 个包 ≈ 5.5 秒**的缓冲，正常情况下不会满；而且**音频丢包比视频丢帧危害大得多**（耳朵对短促的缺口极其敏感，视频丢帧人眼不太明显）。

这是一个真实的工程判断：**不同数据有不同的降级策略。视频丢得，音频丢不得。**

#### 组 10：时间基准与定时器

```cpp
Clock::time_point clockStart;

bool timerRaised = false;
```

- `clockStart`：`start()` 时记录的起点时刻，`stats()` 用它算 `elapsedSeconds`。
- `timerRaised`：记录"我们是否已经调用过 `timeBeginPeriod(1)`"，保证 `timeEndPeriod` 成对注销（第二章 §2.11）。

注意 **`timerRaised` 是普通 `bool` 而不是 atomic**。这是本课的一个小瑕疵：如果 `start()` 和 `stop()` 被不同线程调用（比如 UI 线程启动、定时器线程停止），理论上存在竞争。实践中我们的调用是串行的，所以没问题，但这是值得指出的局限（第十七章 Q5）。

#### 组 11：方法声明 + 三个 inline 工具

我们把最后这一段完整看一遍。

```cpp
void videoLoop();
void audioLoop();
void writeLoop();

void emit(const std::string& message)
{
    std::lock_guard<std::mutex> lock(logMutex);
    if (log)
        log(message);
}

LogCallback guardedLog()
{
    return [this](const std::string& message) { emit(message); };
}

VideoFramePtr scaleToTarget(const VideoFramePtr& frame);
void releaseScale()
{
    if (scaleContext) {
        sws_freeContext(scaleContext);
        scaleContext = nullptr;
    }
    scaleSrcWidth = 0;
    scaleSrcHeight = 0;
}
```

**（1）为什么 `emit` 要在类里 inline 定义而不是放到 .cpp？**

因为它太短了（3 行），放这里读起来最方便，而且编译器一定会内联优化。当然这只是风格问题。

**`emit` 的三步走：**

```cpp
void emit(const std::string& message)
{
    std::lock_guard<std::mutex> lock(logMutex);   // ① 加锁（RAII，出作用域自动释放）
    if (log)                                       // ② 判空：外部没传回调就什么都不做
        log(message);                              // ③ 调用
}
```

- ① **为什么这里用 `lock_guard` 而不是 `unique_lock`？** 因为**这个锁不需要配合条件变量、不需要中途解锁**，用最轻量的 `lock_guard` 就对了。这是第 3 课 §2.3 讲的"能轻就轻"原则。
- ② `if (log)` 这一句让 `start(config, nullptr)` 成为可能——测试里就用这个能力去做"静默启动"（第 856 行 `pipeline.start(config, nullptr)`）。同学们写代码的时候要习惯这种**"可选功能用空值表达，调用前判空"**的契约。

**（2）`guardedLog()`：一个会造回调的回调工厂**

```cpp
LogCallback guardedLog()
{
    return [this](const std::string& message) { emit(message); };
}
```

这一行信息量很大，请同学们务必看懂。

- 返回类型是 `LogCallback`（即 `std::function<void(const std::string&)>`）。
- 返回值是一个 **lambda 表达式**（匿名函数），它**捕获了 `this`**（`[this]`），这样它内部才能调用 `this->emit(...)`。
- 因为 `emit` 里有锁，所以**这个 lambda 就是一个"线程安全的日志回调"**。

它的用法在 `start()` 里：

```cpp
const LogCallback say = impl_->guardedLog();
// 然后把这个 say 交给各个子模块：
impl_->videoEncoder.open(videoConfig, say);
impl_->audioEncoder.open(audioConfig, say);
impl_->output.open(config.targets, ..., say);
impl_->screen.init(say);
impl_->audio.open(config.audioType, config.audioDeviceIndex, say);
```

**这个设计解决了一个非常实际的问题**：所有子模块将来打印的日志，**自动获得了线程安全保护和统一的出口**，而子模块自己完全不知道（`VideoEncoder` 以为自己拿到的就是一个普通回调）。

> **这就是装饰者模式（Decorator Pattern）的一个微小实例**：我们不修改 `VideoEncoder`，而是给它"包"了一层带锁的外壳。同学们记住这个套路——**想给第三方/下层组件加能力，又不想改它的代码，就把它需要的回调/接口包装一层再塞给它。**

有一个需要注意的**生命周期问题**：这个 lambda 捕获了 `Impl*` 裸指针。只要 `Impl` 活着它就安全；`Impl` 析构之后调用它就会 use-after-free。这依赖"析构前一定会 `stop()` 并且 join 所有线程"来保证——**析构函数里调用 `stop()` 正是为此**（第十二章会看到）。

**（3）`releaseScale()`：销毁缩放上下文**

```cpp
void releaseScale()
{
    if (scaleContext) {
        sws_freeContext(scaleContext);
        scaleContext = nullptr;
    }
    scaleSrcWidth = 0;
    scaleSrcHeight = 0;
}
```

- `if (scaleContext)` 判空：**FFmpeg 的 free 系列函数通常接受 nullptr（内部自己判），但我们先判一次更清晰**，而且避免了某些老版本的坑。
- `scaleContext = nullptr;` 这一步非常关键，它保证了 **幂等性**（idempotent）——调用两次 `releaseScale()` 不会 double-free。**任何资源释放函数都应该设计成幂等的**，这是防御式编程的基本功。
- 最后把两个尺寸也归零，保证下次一定会重建。

我们会在两个地方调用它：`scaleToTarget` 里需要重建时、`stop()` 里清理时。

### 7.4 videoLoop 之前的最后一个工具：scaleToTarget（第 169~206 行）

```cpp
VideoFramePtr StreamPipeline::Impl::scaleToTarget(const VideoFramePtr& frame)
{
    if (!frame)
        return nullptr;

    if (frame->width == config.width && frame->height == config.height)
        return frame;

    if (!scaleContext || scaleSrcWidth != frame->width
        || scaleSrcHeight != frame->height) {
        releaseScale();
        scaleContext = sws_getContext(frame->width, frame->height, AV_PIX_FMT_BGRA,
                                      config.width, config.height, AV_PIX_FMT_BGRA,
                                      SWS_BILINEAR, nullptr, nullptr, nullptr);
        if (!scaleContext) {
            emit("创建缩放上下文失败");
            return nullptr;
        }
        scaleSrcWidth = frame->width;
        scaleSrcHeight = frame->height;
    }

    auto out = std::make_shared<VideoFrame>();
    out->width = config.width;
    out->height = config.height;
    out->pts = frame->pts;
    out->bgra.resize(static_cast<size_t>(config.width) * config.height * 4);

    const uint8_t* srcData[4] = {frame->bgra.data(), nullptr, nullptr, nullptr};
    const int srcLinesize[4] = {frame->width * 4, 0, 0, 0};
    uint8_t* dstData[4] = {out->bgra.data(), nullptr, nullptr, nullptr};
    const int dstLinesize[4] = {config.width * 4, 0, 0, 0};

    sws_scale(scaleContext, srcData, srcLinesize, 0, frame->height, dstData,
              dstLinesize);

    return out;
}
```

逐段讲。

#### 第 170~174 行：两条快速返回路径

```cpp
if (!frame)
    return nullptr;

if (frame->width == config.width && frame->height == config.height)
    return frame;
```

- 第一行**防御空指针**——万一采集失败（`ScreenCapturer::capture()` 返回 `nullptr`），这里直接传回去。
- 第二行是**性能优化的最重要一行**：如果尺寸刚好等于目标，**directly 把这个 `shared_ptr` 原样返回**（引用计数 +1，像素数据一个字节都不拷）。大多数情况下（桌面分辨率 = 输出分辨率）就是这条路径。**这叫"零开销快路径"（fast path）。**

同学们体会这个结构：**先处理特殊/简单的情况快速返回，把昂贵的逻辑留到最后**。这是写高性能代码的基本套路。

#### 第 177~189 行：上下文的懒创建与重建

```cpp
if (!scaleContext || scaleSrcWidth != frame->width
    || scaleSrcHeight != frame->height) {
    releaseScale();
    scaleContext = sws_getContext(frame->width, frame->height, AV_PIX_FMT_BGRA,
                                  config.width, config.height, AV_PIX_FMT_BGRA,
                                  SWS_BILINEAR, nullptr, nullptr, nullptr);
    if (!scaleContext) {
        emit("创建缩放上下文失败");
        return nullptr;
    }
    scaleSrcWidth = frame->width;
    scaleSrcHeight = frame->height;
}
```

这就是 §7.3 讲的"缓存 + 失效检测"。三个条件用 `||` 连接：

1. `!scaleContext`：**第一次进来**，上下文还没建。
2. `scaleSrcWidth != frame->width`：**源宽度变了**（换了显示器分辨率）。
3. `scaleSrcHeight != frame->height`：**源高度变了**。

只要有一个成立，就 `releaseScale()` 销毁旧的，然后 `sws_getContext` 造新的。

`sws_getContext` 的 9 个参数，从左到右是：

| 参数 | 值 | 含义 |
|---|---|---|
| `srcW, srcH` | `frame->width/height` | 源图尺寸 |
| `srcFormat` | `AV_PIX_FMT_BGRA` | 源像素格式（采集层统一产出 BGRA，第 3 课的合同！） |
| `dstW, dstH` | `config.width/height` | 目标尺寸 |
| `dstFormat` | `AV_PIX_FMT_BGRA` | 目标格式（**还是 BGRA**，所以这一步只是缩放，不换格式；换格式是 `VideoEncoder` 内部 BGRA→YUV420P 的事） |
| `flags` | `SWS_BILINEAR` | 缩放算法（第二章 §2.13） |
| 后面三个 `nullptr` | — | 源/目标滤波器、调色板等高级选项，用不上 |

**失败处理**：`sws_getContext` 可能失败（内存不足、不支持的分辨率组合）。这里**没有让整个管线失败**，而是 `emit` 一条日志后返回 `nullptr`——调用方会把这一帧跳过。

> 这里体现了实时系统的一个原则：**"单帧失败 ≠ 整条流水线失败"**。缩放造不出来，最多是这一帧没法推；下一帧再试一次说不定就好了（内存释放后）。这叫 **"故障隔离到最小单元"**。

#### 第 191~195 行：造输出帧

```cpp
auto out = std::make_shared<VideoFrame>();
out->width = config.width;
out->height = config.height;
out->pts = frame->pts;
out->bgra.resize(static_cast<size_t>(config.width) * config.height * 4);
```

四个动作：

1. **`std::make_shared<VideoFrame>()`**：在堆上造一个 `VideoFrame`并返回 `shared_ptr`。

   > 为什么用 `make_shared` 而不是 `new`？两个原因：① **异常安全**（`new` + `shared_ptr` 构造之间如果抛异常就泄漏了）；② **只分配一次内存**（对象本体和引用计数块连续分配，少一次 malloc，也更 cache-friendly）。**C++11 之后永远用 `make_shared`。**
2. **`out->pts = frame->pts;`**：**时间戳必须透传**，这一点极其重要——缩放不能改变"这是第几毫秒的画面"这个事实。忘了这一行，缩放到来之后画面还是对的但音画就错位了。**任何时候你产生一个新帧，第一件事就是想清楚 pts 怎么办。**
3. **`out->bgra.resize(width * height * 4)`**：分配 BGRA 缓冲区。每像素 4 字节（蓝、绿、红、透明各 1 字节）。
4. **`static_cast<size_t>(...)`**：避免整数溢出。1920×1080×4 = 8,294,400 还塞得下 int，但 4K 是 3840×2160×4 = 33,177,600，仍在 int 范围内；可是如果哪天不小心写歪了（比如忘了除以某个数），`int` 乘法溢出后再转 `size_t` 就变成一个荒谬的巨大数。**先转 `size_t` 再乘是好习惯。**

> **内存账复习**（第 3 课 §2.9）：720p BGRA = 1280×720×4 = **3,686,400 字节 ≈ 3.5 MB**。1080p ≈ 7.9 MB。这意味着什么？**每帧都 new 一块 3.5MB 的内存，30fps 就是每秒 105 MB 的分配/释放压力**。这是本课一个明显的性能隐患（第十七章 Q3 会让你改成帧池复用）。

#### 第 197~203 行：sws_scale 调用

```cpp
const uint8_t* srcData[4] = {frame->bgra.data(), nullptr, nullptr, nullptr};
const int srcLinesize[4] = {frame->width * 4, 0, 0, 0};
uint8_t* dstData[4] = {out->bgra.data(), nullptr, nullptr, nullptr};
const int dstLinesize[4] = {config.width * 4, 0, 0, 0};

sws_scale(scaleContext, srcData, srcLinesize, 0, frame->height, dstData,
          dstLinesize);
```

为什么是长度为 4 的**指针数组**？因为 FFmpeg 要支持**平面格式**（planar）的数据，比如 YUV420P 有三个平面（Y、U、V 各自一块连续内存），某些格式甚至有第四个 plane（alpha）。所以 `sws_scale` 的接口统一接收"最多 4 个平面的指针数组"和"最多 4 个平面的行距数组"。

我们的 BGRA 是**打包格式**（packed / interleaved），所有分量交错存在**一个**连续缓冲区里，所以只有下标 0 有值，其余三个是 `nullptr`。

**`linesize` 的坑**（第二章 §2.13 讲过）：这里写的是 `width * 4` 而不是某个 stride 参数，因为**我们自己 vector 分配的缓冲区是紧凑的**（每行恰好 width×4 字节，没有 padding），这样写是对的。**但如果你的源图来自 GPU/摄像头驱动，千万不要这么算——必须用驱动给的 stride。**

`sws_scale` 的参数：

| 位置 | 实参 | 含义 |
|---|---|---|
| 1 | `scaleContext` | 转换器上下文 |
| 2 | `srcData` | 源平面指针数组 |
| 3 | `srcLinesize` | 源行距数组 |
| 4 | `0` | 从第几行开始处理（0 = 从顶开始） |
| 5 | `frame->height` | 处理多少行（全高 = 整幅） |
| 6 | `dstData` | 目标平面指针数组 |
| 7 | `dstLinesize` | 目标行距数组 |

> 第 4、5 参数支持"只缩放一部分区域"（比如只更新屏幕变化的中间 200 行），这在很多商业推流器里用来做局部更新优化。**这是我们未来可以做的一个优化点。**

#### 第 205 行：返回

```cpp
return out;
```

返回一个新的 `VideoFramePtr`。调用方拿到的可能是两种东西之一：

- 原帧（尺寸匹配时）——同一块数据；
- 新帧（缩放后的）——独立的一块数据。

**调用方不需要知道是哪种**，这就是封装的好处。下一帧原本的 `frame` 会因为 `shared_ptr` 离开作用域而自动释放（如果它就是 out 的话。评标——不影响，引用计数为 1 时释放）。

好，家底清点完毕。**总结一下 Impl 的成员清单**（同学们把这张表抄一遍，它就是你理解后面三个循环的地图）：

| 成员 | 谁写 | 谁读 |
|---|---|---|
| `config` | `start()` | 三个线程 |
| `log` / `logMutex` / `emit` | `start()` | 所有线程的日志出口 |
| `running` | `start()` / `stop()` | 三个线程循环条件 |
| `screen` / `camera` / `cameraConverter` | — | **videoThread 独占** |
| `audio` | — | **audioThread 独占** |
| `videoEncoder` | — | **videoThread 独占** |
| `audioEncoder` | — | **audioThread 独占** |
| `output` | — | **writeThread 独占** |
| `videoQueue` | videoThread push | writeThread pop |
| `audioQueue` | audioThread push | writeThread pop |
| `scaleContext` 三件套 | — | **videoThread 独占** |
| 五个计数器 | 各自线程 `++` | `stats()` 读 |
| `clockStart` | `start()` | `stats()` |
| `timerRaised` | `start()` / `stop()` | `stop()` |

**没有一处需要跨线程加锁（除了 logMutex）**——这就是把资源划分干净带来的红利。请同学们盯着这张表看 30 秒：**每个对象只有一个所有者线程。这就是本课并发设计的核心秘密。**

---
## 第八章 videoLoop：采集线程的心脏

这是本课最长、最重要的一段代码（第 208~288 行，81 行）。它的职责是：**按目标帧率的节奏，从屏幕或摄像头拿一帧，给它打时间戳，缩放到目标尺寸，编码成压缩包，塞进 `videoQueue`。**

我们先看全貌，然后分七个阶段逐电缆——逐段讲。

```cpp
void StreamPipeline::Impl::videoLoop()
{
    // ── 阶段 A：开局的常量准备（第 210~213 行）─────────────────
    const bool isScreen = (config.sourceType == SourceType::Screen);
    const int fps = std::max(1, config.fps);
    const auto frameInterval = std::chrono::microseconds(1000000 / fps);

    auto nextFrameAt = Clock::now();

    // ── 阶段 B：帧率节拍器（第 216~224 行）─────────────────────
    const auto waitNextFrame = [&]() {
        if (!isScreen)
            return;
        nextFrameAt += frameInterval;
        const auto after = Clock::now();
        if (nextFrameAt < after)
            nextFrameAt = after;
        std::this_thread::sleep_until(nextFrameAt);
    };

    // ── 阶段 C：时间轴的三个状态变量（第 226~228 行）───────────
    bool firstFrame = true;
    Clock::time_point firstFrameAt;
    int64_t lastPts = -1;

    while (running) {
        VideoFramePtr frame;

        // ── 阶段 D：取帧（第 233~246 行）───────────────────────
        if (isScreen) {
            frame = screen.capture();
        } else {
            CameraRawFramePtr raw = camera.nextFrame(1000);
            if (!raw) {
                continue;
            }
            frame = cameraConverter.convert(*raw);
            if (!frame) {
                if (log && !cameraConverter.lastError().empty())
                    log("摄像头帧转换失败: " + cameraConverter.lastError());
                continue;
            }
        }

        if (!frame)
            continue;

        // ── 阶段 E：算 pts（第 251~267 行）─────────────────────
        const auto now = Clock::now();
        if (firstFrame) {
            firstFrameAt = now;
            firstFrame = false;
        }

        const int64_t elapsedUs =
            std::chrono::duration_cast<std::chrono::microseconds>(now - firstFrameAt)
                .count();
        const int64_t pts = (elapsedUs * fps + 500000) / 1000000;

        if (pts <= lastPts) {
            ++videoDropped;
            waitNextFrame();
            continue;
        }
        lastPts = pts;

        // ── 阶段 F：缩放 + 编码 + 入队（第 269~281 行）─────────
        frame->pts = pts;

        frame = scaleToTarget(frame);
        if (!frame) {
            waitNextFrame();
            continue;
        }

        for (auto& packet : videoEncoder.encode(*frame)) {
            videoQueue.push(std::move(packet));
            ++videoPackets;
        }
        ++videoFrames;

        waitNextFrame();
    }

    // ── 阶段 G：退出前的 flush（第 286~287 行）─────────────────
    for (auto& packet : videoEncoder.flush())
        videoQueue.push(std::move(packet));
}
```

### 8.1 阶段 A：开局的三个常量（第 210~213 行）

```cpp
const bool isScreen = (config.sourceType == SourceType::Screen);
const int fps = std::max(1, config.fps);
const auto frameInterval = std::chrono::microseconds(1000000 / fps);

auto nextFrameAt = Clock::now();
```

**第一句：把"是不是屏幕源"缓存成局部变量。**

```cpp
const bool isScreen = (config.sourceType == SourceType::Screen);
```

两种意义：① 这个循环里会判断 5 次，缓存一次读起来更清爽；② `const` 表明"线程启动之后配置不会再变"，**这是一个正式的承诺**——如果你想支持运行时切换采集源，这一行就是第一个要改的地方（得改成每次读 atomic 或者重新 start）。

**第二句：`std::max(1, config.fps)` —— 防御除零。**

```cpp
const int fps = std::max(1, config.fps);
```

为什么这里还要兜底？明明 `start()` 里已经校验过 `fps > 0` 了啊！

这是**"纵深防御"**（defense in depth）思想：**每一层都假设上一层可能出错。** 万一将来有人在别的地方绕过校验改了 config，这里的 `max(1, ...)` 能保证**至少不会做除以零**（除零在整数运算里会让整个 CPU 抛异常，进程立刻崩溃）。

> 这条经验值 memorization-worthy（值得记）：**凡是出现 `a / fps` 这种表达式，前面一定要有"fps 至少为 1"的保证，不管上游校验了几次。** 因为除零 crash 是没有堆栈、没有日志、最难查的一种崩溃。
>
> 顺带一提：这也是为什么 `<algorithm>` 被 include 进来——就为了这一个 `std::max`。

**第三句：算帧间隔。**

```cpp
const auto frameInterval = std::chrono::microseconds(1000000 / fps);
```

1 秒 = 1,000,000 微秒，除以 fps 得到"每帧多少微秒"。

**为什么用微秒而不是毫秒？** 因为毫秒太粗了！假设你想做一个 60fps 的推流，帧间隔是 16.67ms，用整数毫秒只能表示成 16 或 17，**误差 2%，累积起来每秒差 0.4 帧，一分钟就差 24 帧**——画面会明显地"拖后累积延迟"。微秒的精度是足够够的（普通 240fps 也能整除）。

> `<chrono>` 里预定义的时间单位别名：**纳秒 `nanoseconds`、微秒 `microseconds`、毫秒 `milliseconds`、秒 `seconds`、分钟 `minutes`、小时 `hours`**。挑一个"比你的目标精度再细一档"的，这是通行的做法。

**第四句：起始锚点。**

```cpp
auto nextFrameAt = Clock::now();
```

这是节拍器的第一个"目标时刻"。后面会把它的值累加，所以不能是 const。

### 8.2 阶段 B：帧率节拍器 waitNextFrame（第 216~224 行）

```cpp
const auto waitNextFrame = [&]() {
    if (!isScreen)
        return;
    nextFrameAt += frameInterval;
    const auto after = Clock::now();
    if (nextFrameAt < after)
        nextFrameAt = after;
    std::this_thread::sleep_until(nextFrameAt);
};
```

这是整个 videoLoop 里最精妙的一小块，同学们务必看懂。

**先看语法：`[&]` 是什么？**

这是一个 **lambda（匿名函数）**，捕获列表 `[&]` 表示"**按引用捕获所有用到的外部变量**"。所以 lambda 里能直接改 `nextFrameAt`（它是引用捕获的），也能读 `isScreen` 和 `frameInterval`。

> **为什么不写成普通成员函数？** 因为它要改的 `nextFrameAt` 是这个循环里的**局部变量**。写成成员函数就得把它提成 Impl 的成员，污染 Impl 的结构。**lambda 让我们可以把一小段逻辑内联在使用它旁边，而且能访问局部状态**——这就是 lambda 最大的价值。

再补一句：`const auto waitNextFrame = ...` 说明这个 lambda 对象本身不会被重新赋值，但它内部改的是别人。

**然后看语义，四行分别做什么：**

```cpp
if (!isScreen) return;              // ① 摄像头路径自己有节奏，不需要打拍子
nextFrameAt += frameInterval;       // ② 目标时刻 = 上一帧目标 + 一个间隔
const auto after = Clock::now();
if (nextFrameAt < after)            // ③ 目标已过期（处理超时）→ 就地对齐到现在
    nextFrameAt = after;
std::this_thread::sleep_until(nextFrameAt);  // ④ 睡到那个绝对时刻
```

这四条已经在第二章 §2.10 详细讲过了，这里只补充同学们的常见疑问：

**Q：为什么摄像头路径要 `return`（不打拍子）？**

因为 `camera.nextFrame(1000)` **本身会阻塞等待摄像头驱动给出下一帧**，摄像头的硬件已经在输出 30fps 了。如果再加一个 33ms 的 sleep，实际就变成 66ms 一帧——**帧率直接腰斩**。这是个非常典型的集成 bug：**两个地方都在打拍子，结果打了两拍。**

> 记住一句话：**节奏必须在唯一的地方产生。** 谁提供数据，谁提供节奏。

**Q：`sleep_until` 会不会因为 "timer resolution" 不准？**

会，这正是第二章 §2.11 讲 `timeBeginPeriod(1)` 的原因。`sleep_until` 的底层是 Windows 的定时器队列，粒度默认是 15.6ms。调了 `timeBeginPeriod(1)` 之后，误差降到 1ms 级别，对 33ms 的帧间隔来说是完全够用的（3% 误差）。

**Q：为什么 `waitNextFrame()` 在 `continue` 之前被调用了三次？**

同学们注意看代码里有三处 `continue`：**pts 重复时、缩放失败时、摄像头取不到帧时**，每次 continue 之前（除了摄像头那条）都调用了 `waitNextFrame()`。

这是一个容易写错的地方：**如果你在某个提前 `continue` 的分支忘了调 `waitNextFrame()`，那么这一轮就变成"空转"（busy spin）——不 sleep 直接开始下一轮**，CPU 立刻跑满 100%，而且会把采集器逼疯。

> 这是初学并发最常见的 bug 之一：**循环里有一条路径没有"休眠/阻塞"操作**。同学们写这种循环的时候，画一张流程图把每个 `continue` 标出来，逐个确认"它之前有没有阻塞"。
>
> 顺带解释摄像头那条为什么不用调：因为 `camera.nextFrame(1000)` 已经阻塞过 1 秒了（返回 null 说明超时），不需要再睡。

### 8.3 阶段 C：时间轴的三个状态变量（第 226~228 行）

```cpp
bool firstFrame = true;
Clock::time_point firstFrameAt;
int64_t lastPts = -1;
```

| 变量 | 作用 | 初始值为什么是这个 |
|---|---|---|
| `firstFrame` | 标记"第一帧还没到"，用来给 `firstFrameAt` 赋一次值 | `true` |
| `firstFrameAt` | **整条视频时间轴的零点**（epoch） | 未初始化，等第一帧时再定 |
| `lastPts` | 上一帧成功编码的 pts，用于去重 | `-1` |

**为什么 `lastPts` 初值是 `-1` 而不是 `0`？**

因为 pts 从 0 开始（第一帧 elapsedUs=0 → pts=0），如果 `lastPts` 也初始化为 0，那么第一帧的 `pts(0) <= lastPts(0)` 成立，**第一帧就会被误判为重复帧丢掉**！用 `-1` 保证了第一帧一定能通过。

> 这类"哨兵初始值"的 bug 非常隐蔽——表现为"第一帧永远丢失"，且只在你去看pts 的时候才发现。**凡是"比大小去重"的逻辑，第一个值一定要设成一个不可能出现在正常数据里的边界值。** 这是 §5.1 讲的 sentinel 模式的又一次出现。

**为什么 `firstFrameAt` 在第一帧到来时才赋值，而不是在 `now` 之前？**

如果写成"循环开始前 `firstFrameAt = Clock::now()`"，那么 pts 会包含"打开摄像头花了 200ms"这类启动开销，导致时间轴起点不干净。现在的做法保证了：**视频时间轴的 0 时刻 == 第一帧画面的真实时刻**。这样音视频两条时间轴的零点才尽可能对齐（音频的零点其实是 `AudioCapturer` 内部 WASAPI 开始工作的时刻）。

### 8.4 阶段 D：取帧（第 233~246 行）——本课的灵魂区域之一

```cpp
if (isScreen) {
    frame = screen.capture();
} else {
    CameraRawFramePtr raw = camera.nextFrame(1000);
    if (!raw) {
        continue;
    }
    frame = cameraConverter.convert(*raw);
    if (!frame) {
        if (log && !cameraConverter.lastError().empty())
            log("摄像头帧转换失败: " + cameraConverter.lastError());
        continue;
    }
}

if (!frame)
    continue;
```

**屏幕路径（第 234 行）：**

```cpp
frame = screen.capture();
```

回忆第 4 课：DXGI 路径用的是 `AcquireNextFrame(0, ...)`（超时 0，立刻返回），如果屏幕没更新，它返回**上一帧的一份拷贝**（`ScreenCapturer.cpp` 里 `std::make_shared<VideoFrame>(*impl_->lastFrame)`）；如果连上一帧都没有，返回 `nullptr`。

所以屏幕路径**永远不会阻塞**。这就是为什么它必须要外部节拍器（§8.2）。

> 顺便说一个值得思考的现象：桌面如果没有更新（你啥也不动），`capture()` 每 33ms 返回一个内容相同的帧，pts 却是递增的。**编码器会照样把它们编出来**（每帧只有几 KB 的 P 帧数据，因为画面完全没变化），最终产出的码流会比真实视频小很多。这是 P 帧预测的自然结果，属于正常现象。

**摄像头路径（第 236~245 行）：**

```cpp
CameraRawFramePtr raw = camera.nextFrame(1000);
if (!raw) {
    continue;
}
frame = cameraConverter.convert(*raw);
if (!frame) {
    if (log && !cameraConverter.lastError().empty())
        log("摄像头帧转换失败: " + cameraConverter.lastError());
    continue;
}
```

三个要点：

1. **`camera.nextFrame(1000)` 里的 1000 是"最多等 1000 毫秒"**。这就是为什么 `stop()` 之后视频线程可能要花**最多 1 秒**才退出——等待是不可中断的。**这是本课"停止延迟"的最主要来源**，第十五章排错会仔细算这笔账。

2. **`if (!raw) continue;`** —— 超时或被拔出摄像头时返回 `nullptr`，跳过这一轮。**注意这里没有 sleep**（原因见 §8.2），因为这一次调用本身已经阻塞了 1 秒。

3. **错误信息收费打印**：

   ```cpp
   if (log && !cameraConverter.lastError().empty())
       log("摄像头帧转换失败: " + cameraConverter.lastError());
   ```

   这里有两处值得学习：

   - **双条件**：`log` 判空（外部没传回调）+ `lastError().empty()` 判空（避免打印"摄像头帧转换失败: "加一个空字符串这种无意义日志）。
   - **`lastError()` 模式**：这是本项目的一贯约定——**每个模块都自己存最后一次错误原因，让别人决定要不要看、什么时候看。** 比起"模块内部直接 printf"，这种模式让错误信息可以流转到 UI 层。同学们在自己写代码时用这个模式。
   - 顺带指出一个**本课的瑕疵**：这里用的是 `log(...)` 而**不是 `emit(...)`**！

     **这意味着这条日志绕过了 `logMutex` 的保护**，如果 UI 层也在打日志就会串台。这是个真实的 bug，虽然不严重。**建议改成 `emit(...)`（第十七章 Q7 让你动手改）。**

   > 回顾第 6 课的教训：`CameraFrameConverter::convert()` 可能因为 MJPG 解码失败、不支持的像素格式等原因返回 `nullptr`，并且把原因存进 `lastError()`。这里我们正确地查阅了它。

**第四层保险（第 248~249 行）：**

```cpp
if (!frame)
    continue;
```

屏幕路径可能返回 `nullptr`（连上一帧都没有时），在这里统一兜住。**这条语句同时覆盖两条路径，避免在每个分支里重复判断一遍。** 这是"统一收口"的写法，值得模仿。

### 8.5 阶段 E：算 pts（第 251~267 行）——本课最重要的算术

```cpp
const auto now = Clock::now();
if (firstFrame) {
    firstFrameAt = now;
    firstFrame = false;
}

const int64_t elapsedUs =
    std::chrono::duration_cast<std::chrono::microseconds>(now - firstFrameAt)
        .count();
const int64_t pts = (elapsedUs * fps + 500000) / 1000000;

if (pts <= lastPts) {
    ++videoDropped;
    waitNextFrame();
    continue;
}
lastPts = pts;
```

#### 8.5.1 时间轴的零点

```cpp
const auto now = Clock::now();
if (firstFrame) {
    firstFrameAt = now;
    firstFrame = false;
}
```

第一帧到来时，把当前时刻记为 `firstFrameAt`。之后 `firstFrame` 一直是 false，这段就再也不执行了。

注意这里用的是 `Clock::now()` 的副本 `now`：**同一个时刻要被用两次**（一次给 `firstFrameAt`，一次算 elapsedUs），如果写两遍 `Clock::now()`，就会拿到**两个略有不同的时刻**，导致第一帧的 pts 不是精确 0。这种细节体现了作者的严谨。

#### 8.5.2 pts 的换算公式（重点）

```cpp
const int64_t elapsedUs =
    std::chrono::duration_cast<std::chrono::microseconds>(now - firstFrameAt)
        .count();
const int64_t pts = (elapsedUs * fps + 500000) / 1000000;
```

我们把这个式子展开讲。**目标**：把"从开场到现在过了多少时间"换算成"这是第几帧"（因为 `VideoEncoder` 的 time_base 是 `{1, fps}`，pts 的单位就是"帧"）。

数学上应该是：

```
pts = 流逝秒数 × fps = (elapsedUs / 1,000,000) × fps
```

如果直接用浮点数：`int64_t pts = (int64_t)(elapsedUs / 1e6 * fps);`

但**作者刻意避免了浮点运算**，用了纯整数运算：

```
pts = (elapsedUs × fps + 500000) / 1000000
```

同学们看那个 `+ 500000` ——它是什么？

**它是"四舍五入"的实现。**

分母是 1,000,000，那么加"半个分母"就是加 500,000，再整除，等价于数学上的 `round(x)`：

```
round(a / b) = (a + b/2) / b       （整除，a、b 都是正整数）
```

举例（fps=30）：

| 流逝时间 elapsedUs | 真实 pts | 直接截断 `(elapsedUs*30)/1e6` | 四舍五入 `(elapsedUs*30+5e5)/1e6` |
|---|---|---|---|
| 0 µs | 0.0 | 0 | 0 |
| 16666 µs | 0.49998 | 0 | 0 |
| 16667 µs | 0.50001 | 0 | **1** |
| 33333 µs | 0.99999 | 0 | **1** |
| 50000 µs | 1.5 | 1 | **2** |

看到了吗？**没有 `+500000` 的话，pts 总是偏小**（0.99999 帧会被当成 0 帧），这会导致**每帧的 pts 系统性偏小一点点**——虽然每帧只差不到 1 个单位，但它意味着"同一个 tick 内容易出现两帧"（pts 增长慢 → 更容易触发 <= lastPts）。

> **为什么避免浮点？** 三个理由：
> 1. **确定性**：整数运算在任何 CPU 上结果完全一致，浮点有精度和舍入模式差异。
> 2. **速度**：整数除法比浮点除法快，在每帧都跑的热路径上不是无所谓。
> 3. **不用当心 `double` 表示不了 16666666 这种大整数**（其实能，但没必要）。
>
> **给同学们的一句话**：音视频引擎里到处都是这种"整数实现带上四舍五入"的技巧，看到 `+ b/2` 就要立刻反应过来——这是 round。

#### 8.5.3 丢帧判定

```cpp
if (pts <= lastPts) {
    ++videoDropped;
    waitNextFrame();
    continue;
}
lastPts = pts;
```

**这是本课的"显式丢帧"闸门**（第二章 §2.9）。含义：**这一帧和上一帧落在了同一个 pts 刻度上 → 放弃它。**

三个必须理解的细节：

1. **`++videoDropped` 是 atomic 自增**，多线程安全，UI 层能通过 `stats().videoFramesDropped` 看到它。
2. **计数后 `continue`，这一帧不被缩放、不被编码**——所以丢帧的代价只有"已经采集过"那一点点。这是正确的顺序：**越早丢越省。**
3. **`waitNextFrame()` 在 continue 之前**——否则这一轮就是纯空转。

**这个闸门真正会触发的场景有哪些？** 同学们想一分钟，我们列出来：

| 场景 | 现象 | 说明 |
|---|---|---|
| 摄像头实际帧率 > 配置帧率 | 稳定地每 N 帧丢 N-1 帧 | **这是"降帧到目标帧率"的实现方式** |
| 某一帧处理超时（编码 60ms） | 立刻接着来下一帧，两帧落同一 tick | 节拍器的"认命"策略导致，见 §8.2 ③ |
| **屏幕没更新**时不适用 | pts 一直在涨 | DXGI 返回重复帧但 pts 递增，照编不误 |

> 同学们注意最后一行：**pts 是按真实时间算的，所以"屏幕没变化"不会触发丢帧**。很多人会误以为这里在丢重复帧，其实它丢的是"同一时间刻度内的多余帧"。

#### 8.5.4 更新时间戳迚 ——更新 lastPts

```cpp
lastPts = pts;
```

只有**通过了闸门**的（未被丢弃的）帧才会更新 `lastPts`。这个顺序很重要：**被丢弃的帧不应该推进 lastPts**（虽然这里 pts 相同与否无所谓，但如果将来放宽成 `<` 就比较重要）。

### 8.6 阶段 F：缩放 + 编码 + 入队（第 269~281 行）

```cpp
frame->pts = pts;

frame = scaleToTarget(frame);
if (!frame) {
    waitNextFrame();
    continue;
}

for (auto& packet : videoEncoder.encode(*frame)) {
    videoQueue.push(std::move(packet));
    ++videoPackets;
}
++videoFrames;

waitNextFrame();
```

**第一句：`frame->pts = pts;`**

把算好的时间戳写进帧里。（第七章 §7.4 讲过 `scaleToTarget` 会把 pts 透传到输出帧。）

这里要注意一个**所有权细节**：`frame` 是 `VideoFramePtr = shared_ptr<VideoFrame>`，有可能是共享对象（比如 `ScaleToTarget` 走快路径返回的就是原对象）。直接改 `frame->pts` 会改动被共享的对象——但因为这里只有一个持有者（采集器已经放手），所以安全。**如果将来改成多线程共享帧，这里要小心。**

**第二句：缩放**

```cpp
frame = scaleToTarget(frame);
```

注意这里是**赋值回同一个变量**：局部的 `frame` 指针被换成了新的（或者同一个）。原来的帧如果引用计数归零会自动释放。

失败处理（返回 `nullptr`）：打日志已经在 `scaleToTarget` 内部做过了（"创建缩放上下文失败"），这里只需要 `waitNextFrame(); continue;`。

**第三句：编码 + 入队（本段核心）**

```cpp
for (auto& packet : videoEncoder.encode(*frame)) {
    videoQueue.push(std::move(packet));
    ++videoPackets;
}
```

一行包含四个知识点：

1. **`videoEncoder.encode()` 返回 `std::vector<AVPacketPtr>`**（第 9 课定义的契约）。为什么是**复数**？因为**一帧进去可能产出多个包**（比如刚好遇到关键帧带 SPS/PPS 时，某些编码器会多吐一个包）；也可能产出 **0 个包**（H.264 的延迟——B 帧参考需要缓存若干帧，本项目没 B 帧但仍可能有 1 帧延迟）。**不要假设 1 帧 = 1 包。**

2. **`auto& packet`（引用遍历）而不是 `auto packet`（拷贝遍历）**：拷贝一个 `unique_ptr` 是非法的（不能编译）！**必须用引用。** 这是 `unique_ptr` 强制你写正确代码的例子。

3. **`std::move(packet)`**：把包的**所有权移交**给队列。`move` 之后 `packet` 变成空指针，原位置的资源已经在队列里了。**没有 this 一步就没有拷贝。**

4. **`++videoPackets` 在循环体内**：**每个包算一次**，而不是每帧算一次。这样 `stats().videoPacketsEncoded` 反映的是真实的包数。

> 复习第 3 课的零拷贝思想（§2.5）：`AVPacketPtr` 是 `unique_ptr<AVPacket>`，只有 8 字节。**把这几十 KB 甚至几 MB 的压缩数据放进队列的成本 = 移动 8 字节的指针。** 这就是为什么我们敢让第二个线程去消费它。

**第四句：`++videoFrames;`**

每成功编码一帧计一次（注意它在 for 循环**外面**，因为单位是"帧"）。

**第五句：`waitNextFrame();`**

正常路径也要打拍子！这是循环体最后一句，也是最容易忘的一句。**忘了它就变成忘了它就变成"能跑多快跑多快"的满负荷空转**（屏幕路径因为没有阻塞会直接开始下一轮）。

> 同学们记住检查清单：**正常路径结尾 + 每个 continue 分支，都必须有阻塞操作。**

### 8.7 阶段 G：循环退出后的 flush（第 286~287 行）

```cpp
for (auto& packet : videoEncoder.flush())
    videoQueue.push(std::move(packet));
```

当 `running` 变成 `false`，while 循环结束。但**编码器肚子里可能还压着几帧没吐出来**（编码延迟、帧重排序缓冲）——如果不 flush，最后几帧就丢了，录出来的视频会短一点点，而且**最后一个 GOP 不完整**（解码器可能把它丢掉）。

`flush()` 是第 9 课定义的：**往编码器 `avcodec_send_frame(nullptr)`（告诉它"没输入了"），然后 drain 出所有残留的包。**

**为什么这一次不用判断 running？** 因为 `flush()` 的目的在于把数据吐完。**这一点非常重要**：即使停止了，也**必须**把这些数据推入队列——因为 `writeLoop` 的退出条件是"running 为假且两个队列都空"，它会在 join 之前把它们写完（第十二章详细分析时序）。

**这也解释了为什么 join 的顺序必须是"先生产者、后消费者"**（§11.5）：生产者的最后一步（flush）必须在消费者退出之前完成。

> **时序小结**（这是本课最值得背下来的一段）：
> ```
> stop() 设 running=false
>   → videoLoop 跳出 while → 执行 flush() → 推入最后几个包 → 线程函数返回
>   → audioLoop 同理 → 线程函数返回
>   → 主线程 join(videoThread) 成功（此时所有视频包已经在队列里了）
>   → 主线程 join(audioThread) 成功
>   → writeLoop 发现 running=false 但队列非空 → 继续写 → 写完 → 队列空 → 退出
>   → 主线程 join(writeThread) 成功
>   → 所有数据都已落盘/发出，可以安全地 close 了
> ```

如果把 `join(writeThread)` 挪到最前面会怎样？我们第十七章 Q1 会专门讨论。

### 8.8 videoLoop 全流程图

给同学们一张图收尾，把这一章的所有分支画清楚：

```
              ┌──────────────────────┐
              │      进入 while      │
              └──────────┬───────────┘
                         ▼
               ┌─── running 为假？──是──→ 跳出循环 → flush → 线程结束
               │         否
               ▼
        ┌──────────────────┐
        │ 屏幕 or 摄像头？  │
        └───┬──────────┬───┘
        屏幕│          │摄像头
            ▼          ▼
    screen.capture()  camera.nextFrame(1000)   ← 这里阻塞（摄像头路径 ≤1s）
            │              │
            │           有帧？─否─→ continue（不睡，因为已经睡了 1s）
            │              │是
            │              ▼
            │      converter.convert()
            │              │
            │        转换成功？─否─→ 打日志 + continue
            └──────┬───────┘
                   ▼
            frame 有效？─否─→ continue
                   │是
                   ▼
           第一帧？→ 记录 firstFrameAt
                   ▼
           算 elapsedUs → 算 pts（四舍五入）
                   ▼
            pts <= lastPts？─是─→ ++videoDropped → waitNextFrame → continue
                   │否
                   ▼
            lastPts = pts;  frame->pts = pts
                   ▼
            scaleToTarget() → 失败？─是─→ waitNextFrame → continue
                   │成功
                   ▼
            encode() → 每个包 push 进 videoQueue; ++videoPackets
                   ▼
            ++videoFrames
                   ▼
              waitNextFrame()   ← 睡到下一拍
                   ▼
             回到 while 顶部
```

---

## 第九章 audioLoop：最短的一条龙

音频线程只有 17 行（第 290~306 行），但它承担的职责一点也不轻。**看它的目的是为了对照**，同学们把两个并列放在一起来理解"为什么视频这么复杂而音频这么简单"。

```cpp
void StreamPipeline::Impl::audioLoop()
{
    while (running) {
        AudioChunkPtr chunk = audio.nextChunk(100);
        if (!chunk)
            continue;

        ++audioChunks;
        for (auto& packet : audioEncoder.encode(*chunk)) {
            audioQueue.push(std::move(packet));
            ++audioPackets;
        }
    }

    for (auto& packet : audioEncoder.flush())
        audioQueue.push(std::move(packet));
}
```

逐行讲解。

### 9.1 主循环

```cpp
while (running) {
    AudioChunkPtr chunk = audio.nextChunk(100);
    if (!chunk)
        continue;
    ...
}
```

**（1）`audio.nextChunk(100)` 里的 100 是什么？**

最多**等待 100 毫秒**。这个数字是有讲究的：WASAPI 每隔大约 **10ms**（第 8 课 `kPollIntervalMs = 10`）把缓冲区里的数据推一次，通常一推就是 480 或 1024 个样本。所以正常情况下 `nextChunk` 几乎立刻返回一 chunk 数据。

超时才会返回 `nullptr`——比如音频设备被拔掉、或者系统没声音设备。

> **这里 100ms vs 摄像头那里的 1000ms，是个很好的对比**：
> - 音频给 100ms：因为我们的最大可容忍停止延迟大概是几百毫秒级；
> - 摄像头给 1000ms：因为 DirectShow 采集 filter 的某些初始化确实慢，给太少会频繁超时。
> **这两个数字是"停止响应速度"和"避免忙等"之间的权衡。** 数字越大，单轮阻塞越久，退出越慢；数字越小，空转越频繁。**这就是为什么我们必须用带超时的阻塞调用而不是无限阻塞**——它是"响应速度"和"CPU 占用"的可调阀门。

**（2）`if (!chunk) continue;` ——注意这里没有 sleep！**

和摄像头那条一样：超时返回说明**已经等过了 100ms**，不需要再睡。

### 9.2 编码与入队

```cpp
++audioChunks;
for (auto& packet : audioEncoder.encode(*chunk)) {
    audioQueue.push(std::move(packet));
    ++audioPackets;
}
```

和 videoLoop 完全同构——同样的 `auto&`、同样的 `std::move`、同样的原子自增。**唯一的区别是队列从 `videoQueue` 换成了 `audioQueue`（容量 256 vs 24，第二章 §2.3 讲过为什么）。**

同学们思考一个问题：**这里为什么没有 pts 计算？**

答案是——**因为音频的时间戳由 `AudioEncoder` 内部维护**（第 9 课 `AudioEncoder.cpp`）：

```cpp
frame->pts = nextPts;        // nextPts 从 0 开始
nextPts += frameSize;        // 每帧 1024 个样本（AAC 固定）
```

也就是说，**音频的时间轴 = "我已经喂进多少样本"**。因为 WASAPI 实时地按 10ms 一块的速度供货，所以"喂进的样本数"天然等价于"真实时间"（48000 样本 = 1 秒）。

> 这里有个非常精妙的隐含前提：**这条等价关系成立，当且仅当我们不丢任何音频。** 如果我们因为处理不及而丢掉了某些 chunk，那么 `nextPts` 的增长就落后于真实时间，音画就开始错位。
>
> **好消息**：`AudioCapturer` 内部还有一个 64 容量的 `BoundedQueue`（第 8 课），加上本课的 256 容量 `audioQueue`，巨大的缓冲足以吸收抖动。而且我记得第 8 课还讲了 **`padSilence` 补静音**机制：当系统短暂没有声音输出时，`AudioCapturer` 会**主动补一段静音**（`kMinSilenceMs=10` 到 `kMaxSilenceMs=200`），保证时间轴不断裂。这两条加起来，使得"音频时间轴 == 真实时间"这条假设在实践中非常可靠。
>
> 这就是我在 §2.8 说的"两条时间轴各自独立、自动对齐"的具体机制。同学们如果不记得 padSilence，请回去翻第 8 课的对应章节，它是理解本课同步策略的前提。

### 9.3 退出后的 flush

```cpp
for (auto& packet : audioEncoder.flush())
    audioQueue.push(std::move(packet));
```

作用与 §8.7 完全一样：把 AAC 编码器里残留的包（以及 `pending` 缓冲区里不足一帧的零头——第 9 课讲过 `flush()` 会**补零凑成一帧**）吐出来。

> **这里有个隐藏的礼物**：因为 `AudioEncoder::flush()` 会把不足 1024 个样本的零头**补 0 凑整**，所以最后会多出一帧最多 21.3ms 的（部分真实 + 部分静音的）音频。**这让收尾更自然**——播放到结尾不会突然断掉。

### 9.4 三条线程复杂度对比表

把三条线程的循环体放在一起比一比，同学们体会"分工"的含义：

| 维度 | videoThread | audioThread | writeThread |
|---|---|---|---|
| 行数 | 81 行 | 17 行 | 20 行 |
| 独占的资源 | `screen`/`camera`/`cameraConverter`/`videoEncoder`/`scaleContext` | `audio`/`audioEncoder` | `output` |
| 是否需要节拍器 | **是**（屏幕路径） | 否（数据源自带节奏） | 否（队列空就睡 2ms） |
| 是否自己算 pts | **是**（按真实时间） | 否（编码器内部累加） | 否（透传 + 换算） |
| 有没有丢数据行为 | 有（pts 重复 → 计数丢弃） | 无 | 无 |
| 单轮最长阻塞 | ≤1000ms（摄像头）或节拍器剩余 | ≤100ms | 取决于网络/磁盘写 |
| 退出前要做的事 | flush 编码器 | flush 编码器 | 排空两个队列 |

这张表建议同学们抄一遍。**它回答了"为什么视频线程最复杂"**：因为它同时要管**采集、画情况——画质缩放、时间戳生成、帧率控制、丢帧策略**这四件事，而音频线程只管"搬运"，write 线程只管"发货"。

---

## 第十章 writeLoop：把两条河合成一条河

最后一条线程（第 308~328 行，20 行），职责是：**从两个队列里取包，交错地交给 `StreamOutput` 写出去。**

```cpp
void StreamPipeline::Impl::writeLoop()
{
    while (running || !videoQueue.empty() || !audioQueue.empty()) {
        bool didWork = false;

        if (auto packet = audioQueue.tryPop(1)) {
            output.writeAudio(**packet, audioEncoder.timeBase());
            didWork = true;
        }
        if (auto packet = videoQueue.tryPop(1)) {
            output.writeVideo(**packet, videoEncoder.timeBase());
            didWork = true;
        }

        if (!didWork) {
            if (!running)
                break;
            std::this_thread::sleep_for(std::chrono::milliseconds(2));
        }
    }
}
```

### 10.1 循环条件：优雅退出的关键

```cpp
while (running || !videoQueue.empty() || !audioQueue.empty()) {
```

这一行是本课**第二重要的一行**（第一行是 pts 公式）。它用三个条件的"或"表达了完整的工作使命：

| 条件 | 含义 |
|---|---|
| `running` | 还在推流，正常干活 |
| `!videoQueue.empty()` | 已经停止了，**但视频队列里还有货**——继续写 |
| `!audioQueue.empty()` | 已经停止了，**但音频队列里还有货**——继续写 |

这叫**排空语义**（drain semantics），是生产者-消费者问题最标准的收尾方式。

**一个必须想通的问题**：这三个条件的求值顺序会不会出问题？设想一下——`running` 还是 `true`，两个队列都恰好为空（下一批包还没到），按照这个条件，循环**会退出**！那不是 bug 吗？

**不是 bug，但确实是一个需要注意的时序点。** 我们仔细分析：

- `writeLoop` 是在 `start()` 里和另外两条线程一起启动的，而 `running` 在**线程创建之前**就被设为 `true`（第 404 行）。
- 如果 write 线程先跑起来，此时队列确实为空、running 为 true → 条件为 true → 正常进入循环体 → 两个 tryPop 都空转 1ms → `didWork=false` → 因为 `running=true` 所以**不 break**，sleep 2ms 继续。**所以不会退出。**
- 只有当 `running` 变成 false 且两个队列都空时才会真正退出。

同学们看清楚第二个分支：**`didWork` 为 false 时的处理才是真正的"退出判断"**，它下面还藏了一层 `if (!running) break;`。**这实际上是一种双保险的循环退出写法。**

> 那么第 310 行的 `while` 条件是不是冗余了？不完全是：它的存在使得"停止之后立刻醒来一次检查"成为可能。由于 CS 的顺序关系，实际上第 322~326 行的 `if (!didWork)` 分支已经能完成全部退出判断了。**这是一个"条件写得比较宽松，靠循环体内部收紧"的写法**，在实践里很常见，但第十七章 Q1 会让同学们讨论"能不能简化"。

### 10.2 取包与写出：为什么音频在前面？

```cpp
if (auto packet = audioQueue.tryPop(1)) {
    output.writeAudio(**packet, audioEncoder.timeBase());
    didWork = true;
}
if (auto packet = videoQueue.tryPop(1)) {
    output.writeVideo(**packet, videoEncoder.timeBase());
    didWork = true;
}
```

**① `tryPop(1)`：1 毫秒超时**

第 3 课定义的 `BoundedQueue::tryPop(timeoutMs)`：**等待最多 1ms，队列里有货就立刻取出，没有就返回 `std::nullopt`。**

为什么是 1ms 而不是 0 或 ∞？

- **0 的话**（纯轮询）会让这个线程跑成忙等，CPU 白烧；
- **∞ 的话**（阻塞到有货为止）——那 `running` 变 false 之后线程永远醒不过来（除非有人 push），**stop 就死锁了**；
- **1ms** 是个漂亮的折中：**既不会忙等，又保证停止信号最多 1ms 内被响应。**

> 这就是我在 §9.1 说的"带超时的阻塞调用是响应速度阀门"。**多线程的停止必须靠"超时轮询"，不能靠"无限等待 + 通知"**——除非你有一套完整的 cv 通知机制（第 3 课的课后思考题里讨论过"BoundedQueue 缺一个 close 语义"）。

**② `auto packet = ...` 的类型**

`tryPop` 返回 `std::optional<AVPacketPtr>`。`if (auto packet = ...)` 利用了 `std::optional` 的**显式 bool 转换**：有值就是 true。**这让代码读起来像"如果能取到一个包"。**

**③ `**packet` —— 两个星号**

这是本课最容易看晕的一处写法，我们拆开：

```cpp
output.writeAudio(**packet, ...);
//               ↑↑
//               │└─ 第二个 *：解引用 optional，得到 AVPacketPtr（unique_ptr）
//               └─── 第一个 *：解引用 unique_ptr，得到 AVPacket&
```

详细展开：

| 表达式 | 类型 |
|---|---|
| `packet` | `std::optional<AVPacketPtr>` |
| `*packet` | `AVPacketPtr` = `std::unique_ptr<AVPacket, AVPacketDeleter>` |
| `**packet` | `AVPacket&` (左值引用) |

而 `writeAudio` 的签名（第 10 课）是 `bool writeAudio(const AVPacket& packet, AVRational encoderTimeBase)`——**接收 `const AVPacket&`**，所以传 `**packet` 正好。

> 这里也体现了第 9 课定 `AVPacketPtr` 为 `unique_ptr` 的好处：**它禁止拷贝**（包的所有权唯一），但同时支持 `*ptr` 拿到引用去读。**所有权只能转移，不能复制**——这正是压缩数据包需要的语义。

**④ 第二个参数：`audioEncoder.timeBase()`**

```cpp
output.writeAudio(**packet, audioEncoder.timeBase());
```

这是时间戳换算的关键（第二章 §2.7）：告诉输出模块"**这个包的 pts 是以 1/48000 秒为单位的**"，`StreamOutput` 内部会用 `av_packet_rescale_ts` 把它换算成 FLV 的 1/1000 毫秒单位。

> **为什么不直接统一单位？** 因为 `VideoEncoder`/`AudioEncoder` 的 time_base 是 FFmpeg 内部约定（跟随编码器 ctx），我们不能随便改。所以在"交接处做换算"比"让源头改变格式"更简单。**这是集成者的智慧：适配，而不是改造。**

**⑤ 为什么音频写在视频前面？**

两个原因，同学们记下来：

1. **人的耳朵比眼睛敏感**。同样的停顿，音频断流 100ms 听众立刻察觉（嗡——），而视频丢一帧完全没感觉。所以**资源紧张时优先保音频**。
2. **音频包小、数量多、间隔密**（21ms 一个）。如果反过来先写视频，一旦视频队列堆积，write 线程会花大量时间写大包，**可能饿死音频队列**（虽然队列有 256 容量缓冲，但长时间饥饿会让音频延迟累积）。先音频可以保证它稳定出队。

> 这个"优先级"也体现在队列容量上（音频 256 vs 视频 24）：**给更容易被饿死的一方更多缓冲。**

**但这里也有一个已知的不足**：每一轮只处理**一个音频包 + 一个视频包**。如果视频积累很快（比如刚启动时的 GOP 突发），每轮只能消化一个视频包，可能跟不上生产速度 → 最终触发 `BoundedQueue` 的丢旧包。改进方向见第十七章 Q2。

### 10.3 didWork：让"无所事事"变得可感知

```cpp
bool didWork = false;
...
if (!didWork) {
    if (!running)
        break;
    std::this_thread::sleep_for(std::chrono::milliseconds(2));
}
```

这是一个非常实用的并发编程技巧：**用布尔量标记"这一轮是否真的做了事"**，如果没做事就稍等一会儿再试。它叫 **"工作检测 + 退避"（work detection with backoff）**。

为什么不简单地写成"每轮睡 2ms"？因为那样会在**有货时也不必要地等待**，白白浪费吞吐。现在的写法保证了：**忙的时候全速处理（只在 tryPop 上花最多 1ms 的等待），闲的时候才退避到 2ms 轮询。**

还有一层更深的考虑：如果写成"有货就继续、无货才睡"，那么**高负载时线程会 100% 占满一个核**。商业推流器通常会加"每轮无论忙闲都 yield 一下"（`std::this_thread::yield()`）来避免抢占写网络的其他线程。这属于进一步优化，本项目没做。

**最后一个细节：`if (!running) break;` 放在 sleep 之前**

这使得"退出"可以在两个时机发生：

- 队列空了且 running=false → 下一轮 `while` 条件直接不成立（正常情况下不会走到这里，因为上一轮就 break 了）；
- 一轮下来什么都没做且 running=false → 立即 `break`（这是**实际生效**的那条路径）。

两种都覆盖了。即使去掉 while 条件里的后半部分也能工作，但保留它们是**显式表达意图**——读代码的人一眼就看出"这个线程会把队列排空"。**好的代码要能被读懂，这是写给自己半年后看的。**

### 10.4 交错写入（Interleaving）：为什么"顺序"这么重要

最后一个知识点。同学们注意：**我们是把音频包和视频包交替交给 `StreamOutput` 的**。这叫**交错**（interleaving）。

为什么要交错？因为 FLV/MP4 这类容器里，**音频和视频是按时间顺序混合存放的**：

```
FLV 文件内部（简化）：

[FLV header][metadata]
[video tag 0ms][audio tag 0ms][audio tag 21ms][audio tag 42ms]
[video tag 33ms][audio tag 64ms][video tag 66ms] ...
       ↑ 播放器读到这里就能解码这一段的所有音视频数据
```

如果你把**所有视频包先写完、所有音频包最后写**，播放器必须先把整个文件读完（或者缓冲很久）才能开始播——这对点播来说凑合能用，**对直播来说是致命的**（永远追不上）。

而且更严格地说，FFmpeg 的 `av_interleaved_write_frame` 内部维护了一个**按 pts 排序的小缓冲**（interleaving queue），它会根据时间戳自动把包排到合适的位置再写出去。**所以我们的写法即便偶尔顺序不太准，ffmpeg 也会自动帮我们整理一下**——这就是为什么第 10 课要用 `av_interleaved_write_frame` 而不是 `av_write_frame`。

> 顺带复习第 10 课的一个坑：正因为 FFmpeg 内部会缓冲重排，**你在 `close()` 之前必须 `write_trailer` 让 FFmpeg 把最后的缓冲冲出去**（第 10 课讲过 trailer）。本课的 `stop()` 会调用 `output.flush()` 和 `output.close()`，就是为了这件事。下一章就看到。

---
## 第十一章 start()：开机的十三个步骤

`start()`（第 339~418 行，80 行）是整个管线的**总装线**，也是本课第二个必须倒背如流的函数。我们按执行顺序把它切成十三个步骤来看。

```cpp
bool StreamPipeline::start(const StreamConfig& config, LogCallback log)
{
    // ── 步骤 1：先关上一次（幂等）────────────────────────────
    stop();

    // ── 步骤 2：记录配置与回调 ───────────────────────────────
    impl_->config = config;
    impl_->log = std::move(log);
    impl_->lastError.clear();

    // ── 步骤 3：准备两个 lambda 工具 ─────────────────────────
    const LogCallback say = impl_->guardedLog();
    const auto fail = [this, &say](const std::string& reason) {
        say(reason);
        stop();
        impl_->lastError = reason;
        return false;
    };

    // ── 步骤 4：参数合法性 ───────────────────────────────────
    if (config.width <= 0 || config.height <= 0 || config.fps <= 0)
        return fail("推流参数无效：宽高和帧率都要大于 0");

    if (config.targets.empty())
        return fail("没有配置推流目标");

    // ── 步骤 5：宽高对齐到偶数 ───────────────────────────────
    impl_->config.width = config.width & ~1;
    impl_->config.height = config.height & ~1;
    if (impl_->config.width != config.width || impl_->config.height != config.height) {
        say("宽高已对齐到偶数: " + std::to_string(impl_->config.width) + "x"
            + std::to_string(impl_->config.height));
    }

    // ── 步骤 6：打开视频编码器 ───────────────────────────────
    VideoEncoderConfig videoConfig;
    videoConfig.width = impl_->config.width;
    videoConfig.height = impl_->config.height;
    videoConfig.fps = impl_->config.fps;
    videoConfig.bitrateKbps = impl_->config.videoKbps;

    if (!impl_->videoEncoder.open(videoConfig, say))
        return fail("视频编码器打开失败: " + impl_->videoEncoder.lastError());

    // ── 步骤 7：打开音频编码器 ───────────────────────────────
    AudioEncoderConfig audioConfig;
    audioConfig.sampleRate = config.sampleRate;
    audioConfig.channels = config.channels;
    audioConfig.bitrateKbps = config.audioKbps;

    if (!impl_->audioEncoder.open(audioConfig, say))
        return fail("音频编码器打开失败: " + impl_->audioEncoder.lastError());

    // ── 步骤 8：打开输出（多路目标）─────────────────────────
    if (!impl_->output.open(config.targets, impl_->videoEncoder.codecParameters(),
                            impl_->audioEncoder.codecParameters(), say)) {
        return fail("推流输出打开失败: " + impl_->output.lastError());
    }

    // ── 步骤 9：打开画面源 ───────────────────────────────────
    if (config.sourceType == SourceType::Screen) {
        if (!impl_->screen.init(say))
            return fail("屏幕采集初始化失败");
    } else {
        if (!impl_->camera.open(config.cameraIndex, say))
            return fail("摄像头打开失败: 索引 " + std::to_string(config.cameraIndex));
    }

    // ── 步骤 10：打开声音源 ──────────────────────────────────
    if (!impl_->audio.open(config.audioType, config.audioDeviceIndex, say))
        return fail("音频采集打开失败");

    // ── 步骤 11：提高定时器精度 ──────────────────────────────
    timeBeginPeriod(1);
    impl_->timerRaised = true;

    // ── 步骤 12：置位标志 + 启动三条线程 ─────────────────────
    impl_->running = true;
    impl_->clockStart = Clock::now();

    impl_->videoThread = std::thread([this] { impl_->videoLoop(); });
    impl_->audioThread = std::thread([this] { impl_->audioLoop(); });
    impl_->writeThread = std::thread([this] { impl_->writeLoop(); });

    // ── 步骤 13：报告成果 ────────────────────────────────────
    say("推流已启动: " + std::to_string(impl_->config.width) + "x"
        + std::to_string(impl_->config.height) + " @"
        + std::to_string(impl_->config.fps) + "fps, "
        + std::to_string(impl_->output.connectedCount()) + "/"
        + std::to_string(config.targets.size()) + " 路");

    return true;
}
```

### 11.1 步骤 1：为什么第一件事是 stop()

```cpp
stop();
```

这一行有两个完全不同的作用，同学们都要想到：

**作用一：重入保护。** 如果调用方忘了先 stop 就再次 start（比如用户快速点了两下"开始"），这一行保证旧的线程被回收、旧的资源被关闭，**不会漏了一条线程在外面跑**。

**作用二：清理上一次的残留。** 如果上次是失败的（比如编码器开了一半失败），残留状态在这里被抹干净。

而且这里体现出 `stop()` 的**幂等性**——无论当前处于什么状态（从没启动过、正在跑、已经停了），调用 `stop()` 都是安全的。**幂等（idempotent）** 这个词同学们记住：**f(f(x)) == f(x)**，调一次和调 N 次效果一样。**任何"关闭/清理/释放"类的函数都应该设计成幂等的**，这是接口设计的基本功。

> 有意思的细节：这一行也让 **`start()` 可以被当成"重启"用**。测试的第 3 段（第 855~866 行）就是靠这个能力做"连续启停三轮"的验证。

### 11.2 步骤 2：记录下来

```cpp
impl_->config = config;
impl_->log = std::move(log);
impl_->lastError.clear();
```

- **`impl_->config = config;`**：把外部传进来的 `const StreamConfig&` **拷贝**一份。（第七章 §7.3 讲过，之后线程只读它。）
- **`impl_->log = std::move(log);`**：这是本课唯一一处用 `move` 转移 `std::function`。注意参数 `LogCallback log` 是**按值传入**的（不是引用！），所以调用方的那个临时对象已经被移动构造到参数里了，这里再 move 给成员，**全程零拷贝**。

  > **复习移动语义**（第 3 课铺垫，第 9 课实践）：`std::move` 本身不做任何拷贝，它只是把左值转换成右值引用，让编译器选中"移动构造函数"。`std::function` 的移动是廉价的（几个指针的转移），而拷贝可能要分配内存。
  >
  > **接口设计启发**：当一个参数是"会被保存起来"的可调用对象时，标准做法就是**按值接收 + move 到成员**（Effective Modern C++ 的 Item 41 专门讲这个）。本项目从头到尾都遵循这个模式。

- **`impl_->lastError.clear();`**：清空上次的失败原因。**这一行很重要**——否则调用方会看到"上次的错误"被当成"这次的原因"。同学们想想你写过的代码里，有多少 StateReset 是被漏掉的。

### 11.3 步骤 3：两个 lambda 工具

```cpp
const LogCallback say = impl_->guardedLog();
const auto fail = [this, &say](const std::string& reason) {
    say(reason);
    stop();
    impl_->lastError = reason;
    return false;
};
```

**（1）`say`：带锁的日志出口。**

```cpp
const LogCallback say = impl_->guardedLog();
```

这就是第七章 §7.3 讲的"回调工厂"产出的线程安全日志回调。**后面所有子模块拿到的都是 `say`。**

**（2）`fail`：统一失败处理。**

这是一个非常值得学习的设计——它把"失败时要做的四件事"收敛到一个 lambda 里：

| 动作 | 为什么必须有 |
|---|---|
| `say(reason)` | 让用户看到原因（UI 上显示"推流输出打开失败: xxx"） |
| `stop()` | **回滚**：把已经打开的一半资源关掉（比如视频编码器开了、音频编码器失败，此时不能留着视频编码器） |
| `impl_->lastError = reason` | 让 `lastError()` 能事后查（程序逻辑用，比如弹窗标题） |
| `return false` | 让 `start()` 返回失败 |

于是后面每一步的开阳——开机动作都只需要一行：

```cpp
if (!xxx.open(...))
    return fail("xxx 打开失败: " + xxx.lastError());
```

**这就是 DRY 原则**（Don't Repeat Yourself，不要重复自己）：如果不抽成 lambda，我们要写 6 遍完全相同四行代码，而且将来加第五件事（比如"记录到日志文件"）就要改 6 个地方，**改一处漏一处是必然的**。

> 顺带说一句捕获列表：`[this, &say]`
> - `this`：lambda 里要访问 `impl_->lastError` 并调用成员函数 `stop()`；
> - `&say`：**按引用捕获** `say`。为什么这里用引用而不是值拷贝 `[say]`？拷贝一个 `std::function` 是可行的（它支持拷贝），但没必要多一次动态分配。**注意生命周期**：`say` 是本函数的局部变量，而 `fail` 也只在函数体内使用，所以引用捕获是安全的。**这是一个容易出错的点：如果 lambda 逃出了当前作用域（比如存给异步线程），绝对不能引用捕获局部变量。**

### 11.4 步骤 4~5：参数校验与预处理

```cpp
if (config.width <= 0 || config.height <= 0 || config.fps <= 0)
    return fail("推流参数无效：宽高和帧率都要大于 0");

if (config.targets.empty())
    return fail("没有配置推流目标");

impl_->config.width = config.width & ~1;
impl_->config.height = config.height & ~1;
if (impl_->config.width != config.width || impl_->config.height != config.height) {
    say("宽高已对齐到偶数: " + std::to_string(impl_->config.width) + "x"
        + std::to_string(impl_->config.height));
}
```

**（1）校验的位置：在所有的 `open` 之前。**

这是"**快速失败**"（fail fast）原则：**在最便宜的地方拒绝。** 参数检查的成本是几纳秒，而打开一个编码器的成本是几十毫秒。如果先开编码器再检查参数，用户就要等几百毫秒才被告知"帧率填错了"。

**（2）为什么 `targets.empty()` 也要检查？**

因为"没有目标"意味着用户白忙一场——录了、编了、结果**什么都不输出**。这是个**逻辑错误**而不是技术错误，必须显式地告诉用户。

> 同学们记住：**凡是"配置层无意义但技术上跑得通"的输入，都要显式拦截。** 这类 bug 最难查，因为程序"看起来在工作"，CPU 也在转，就是没产出。

**（3）`width & ~1`：把宽高对齐到偶数。**

这是一个**位运算技巧**：

```
~1 的二进制 = ...11111110（最低位是 0，其余全是 1）

x & ~1 的结果 = 把 x 的最低二进制位清成 0

例：
  1920 = 11110000000₂   → & ~1 → 11110000000₂ = 1920（本来就是偶数，不变）
  1921 = 11110000001₂   → & ~1 → 11110000000₂ = 1920（向下取整到偶数）
  1281 = 10100000001₂   → & ~1 → 10100000000₂ = 1280
```

**为什么必须对齐到偶数？** 因为 H.264 编码内部用的是 **YUV420P** 格式，它的色度（U/V）分量在**水平和垂直方向都各降采样一半**（每 2×2 个亮度像素共用一组色度）。

```
亮度 Y:  ██ ██ ██ ██     ← 每个像素一个 Y
色度 U:  ▓▓▓▓              ← 每 4 个像素一组（2×2）
色度 V:  ▓▓▓▓
```

**如果宽度是奇数（比如 1281）**，最后那一列像素没有配对的伙伴，降采样就没法干净地做——FFmpeg 通常会在最后一行/列补充边缘像素（padding），但很多编码器实现会**直接报错或产出花屏**。**所以业界约定：H.264 编码的输入必须是偶数宽高。**

> 这就是为什么本项目的第 9 课 `VideoEncoder::encode()` 遇到尺寸不匹配会直接拒绝并返回错误。**用户友好起见**： 与其让用户看到"编码器初始化失败"这种莫名其妙的错误，不如**主动帮他对齐**，并且友好地提示一句"宽高已对齐到偶数: 1280x720"。这就是第 5 行 `say(...)` 的意义。
>
> **教学点评**：这条设计原则叫 **"容忍用户的输入，纠正它并告诉他"**（Postel's Law：be conservative in what you send, be liberal in what you accept）。这是很好的 API 设计感。

**（4）为什么写在 `width > 0` 校验之后？**

因为 `& ~1` 对负数不成立（`-3 & ~1` = `-4`，不是我们想要的）。**先校验再换算，顺序不能反。** 这又是一处"顺序敏感"的细节。

### 11.5 步骤 6~8：开门三兄弟（编码器 + 输出）

```cpp
VideoEncoderConfig videoConfig;
videoConfig.width = impl_->config.width;
videoConfig.height = impl_->config.height;
videoConfig.fps = impl_->config.fps;
videoConfig.bitrateKbps = impl_->config.videoKbps;

if (!impl_->videoEncoder.open(videoConfig, say))
    return fail("视频编码器打开失败: " + impl_->videoEncoder.lastError());

AudioEncoderConfig audioConfig;
audioConfig.sampleRate = config.sampleRate;
audioConfig.channels = config.channels;
audioConfig.bitrateKbps = config.audioKbps;

if (!impl_->audioEncoder.open(audioConfig, say))
    return fail("音频编码器打开失败: " + impl_->audioEncoder.lastError());

if (!impl_->output.open(config.targets, impl_->videoEncoder.codecParameters(),
                        impl_->audioEncoder.codecParameters(), say)) {
    return fail("推流输出打开失败: " + impl_->output.lastError());
}
```

**（1）为什么要先看这些"字段级 đối"字段传递？**

注意两个编码器用的都是"独立的小配置结构体"（`VideoEncoderConfig` / `AudioEncoderConfig`），而不是把 `StreamConfig` 整个塞进去。这是**又一次"面向接口编程"**：

```
StreamConfig（业务的全部配置，12 个字段）
   ├──→ VideoEncoderConfig（编码器关心的 4 个字段）
   ├──→ AudioEncoderConfig（音频编码器关心的 3 个字段）
   └──→ std::vector<StreamTarget>（输出模块关心的东西）
```

好处：**每个子模块只依赖自己关心的那几个字段**，`VideoEncoder` 永远不需要知道世界上有个东西叫 `audioDeviceIndex`。将来给 `StreamConfig` 加 100 个字段，编码器一个字都不用改。

> 反过来的坏做法：把 `StreamConfig` 传给编码器，编码器内部 `config.width`。这样看起来省了几行搬运代码，但实际上**让编码器依赖了整个业务对象**，耦合立刻上升。同学们写自己的分层代码时，一定要写这种"字段搬运"，它看起来啰嗦，实际是在买保险。

**（2）为什么必须先开编码器、后开输出？**

同学们看这个依赖：

```cpp
impl_->output.open(config.targets,
                   impl_->videoEncoder.codecParameters(),   // ← 需要编码器已经 open
                   impl_->audioEncoder.codecParameters(),   // ← 同上
                   say)
```

`StreamOutput::open` 需要把编码器的 **`codecParameters()`**（编解码器参数，包含 SPS/PPS、AudioSpecificConfig 这些 extradata）写进容器的头部。

**这就是所谓的"顺序依赖"**：没有编码器的参数，FLV 头就写不出来。**这种"依赖链"必须靠书写顺序来保证，这是最容易出错的地方**——如果哪天有人把这两段调换了，`output.open()` 拿到的就是空指针。

> **工程改进建议**：这种依赖应该由**类型系统或显式检查**来保证，而不是靠顺序。比如可以让 `StreamOutput::open` 接受"编码器引用"而不是"裸的参数指针"，或者在 open 里对这个 probleme nullptr 做校验。第十七章 Q7 会让你动手加防线。

**（3）连接多路目标是在这一步发生的。**

回顾第 10 课：`StreamOutput::open()` 会对 `targets` 里的每一个目标执行 `openTarget()`——**这一步可能耗时几百毫秒到几秒**（如果是 RTMP，要完成 TCP 握手、RTMP 握手、connect、createStream、publish）。

这就是为什么测试里专门打印"启动耗时"：

```cpp
std::printf("  启动耗时 %lld 毫秒（含编码器初始化 + 两路连接）\n", startupMs);
```

**（4）`connectedCount()` 可能小于目标数量。**

第 10 课讲过："部分成功也算成功"——如果三路目标里有一路连不上，另外两路照样推。这就是本课前面说的"优雅降级"。最后的日志会显示 `2/3 路`。

### 11.6 步骤 9~10：开门——另两个资源（采集源）

```cpp
if (config.sourceType == SourceType::Screen) {
    if (!impl_->screen.init(say))
        return fail("屏幕采集初始化失败");
} else {
    if (!impl_->camera.open(config.cameraIndex, say))
        return fail("摄像头打开失败: 索引 " + std::to_string(config.cameraIndex));
}

if (!impl_->audio.open(config.audioType, config.audioDeviceIndex, say))
    return fail("音频采集打开失败");
```

**注意这里的 if/else：只打开其中一个采集器。**

这是资源节约的好习惯——用户选了屏幕，就**不要**去占用摄像头（摄像头被占用时其他程序就用不了了，而且很多笔记本摄像头有个"使用中"的指示灯会亮）。

> 反例是某些推流软件，明明只录屏，却把摄像头也打开了，指示灯一直亮着让用户以为被偷拍。同学们写代码要有这种"不打扰用户"的意识。

**注意第五行传的就是我们新加的 `audioDeviceIndex`**（第五章）。这就是那个 `-1` 哨兵值被用起来的地方。

**失败信息里带上下文是好习惯：**

```cpp
return fail("摄像头打开失败: 索引 " + std::to_string(config.cameraIndex));
```

把"是第几个摄像头"写进错误信息。将来用户报 bug 时，你一眼就能知道是他选错了设备。这叫 **"让错误信息携带现场数据"**。

### 11.7 步骤 11：提高定时器精度

```cpp
timeBeginPeriod(1);
impl_->timerRaised = true;
```

第二章 §2.11 讲过。这里补充三点：

1. **必须在线程创建之前调用**——因为线程一创建就立刻开始需要精确睡眠了。
2. **`timerRaised = true` 紧跟其后**：这个标志必须和 `timeBeginPeriod` 成对出现，中间不能有任何可能失败/返回的代码。**这里其实有一个微小的隐患**：如果 `timeBeginPeriod` 和 `timerRaised = true` 之间将来被人插入了别的逻辑并且提前返回了，就会出现"调了 begin 没调 end"。建议改成用一个 RAII 的小结构体来管理（第十七章 Q5）。
3. **为什么失败时不检查返回值？** `timeBeginPeriod` 返回 `TIMERR_NOERROR` 或 `TIMERR_NOCANDO`（表示系统不支持这个精度）。本项目没检查——这是合理的，**因为只是"尽力而为"的优化，失败也不影响正确性**（最多抖动大一点）。这就是"非关键路径的错误可以忽略"的判断。

### 11.8 步骤 12：置位 + 起线程（本课的"发令枪"）

```cpp
impl_->running = true;
impl_->clockStart = Clock::now();

impl_->videoThread = std::thread([this] { impl_->videoLoop(); });
impl_->audioThread = std::thread([this] { impl_->audioLoop(); });
impl_->writeThread = std::thread([this] { impl_->writeLoop(); });
```

**为什么 `running = true` 必须在线程创建之前？**

这是一个**经典的并发初始化顺序问题**。设想反过来的顺序：

```
先创建线程 → 线程立刻检查 while(running) → 此时 running 还是 false → 线程直接退出！
再设置 running = true
```

结果就是"start() 报告成功，但一个包都没推出来"。**这种 bug 的表现是偶发的**（取决于线程调度——有时候新线程会稍后才启动，此时 running 已经是 true 了，于是程序看起来正常），所以极其难查。

> **铁律**：**"启动所需的全部共享状态，必须在线程创建之前完成初始化。"** 请同学们记住这句话，它可以让你少掉好多根头发。
>
> 更一般地说，这属于 C++ 内存模型里"happens-before"的范畴：`std::thread` 的构造与它带来的线程执行之间，**标准保证有一个同步点**——在构造之前发生的所有写操作，对新线程一定可见。这不只是"碰巧"正确，是标准保证的。

**为什么 `clockStart` 也在这里赋值？**

因为 `stats()` 用它算 `elapsedSeconds`。必须在 `running=true` 之后但在线程启动之前设置好，保证"运行时间"和"开始推流"这两个事件是同一个瞬间。

**三条线程的 lambda：**

```cpp
std::thread([this] { impl_->videoLoop(); });
```

- 捕获列表 `[this]`：**按值捕获 this 指针**。捕获指针本身是"值"，但通过它访问的是同一对象。等价于在线程函数里访问 `this->impl_->videoLoop()`。
- **生命周期警告**：这三个 lambda 捕获了 `Impl*`。**只要 `Impl` 被析构，它们就变成野指针。** 保证的手段就是"析构前先 stop() 并 join"。第十二章的析构函数正是这么做的。
- 为什么写成 lambda 而不是直接传成员函数指针？两种都行：
  ```cpp
  std::thread([this] { impl_->videoLoop(); });              // 本课写法
  std::thread(&StreamPipeline::Impl::videoLoop, impl_.get());  // 等价的成员函数指针写法
  ```
  lambda 版本可读性更好，也更灵活（将来想在线程里加 try/catch 很方便）。

> **一个本课没有做、但生产环境必做的事：给线程加 try/catch。**
>
> ```cpp
> std::thread([this] {
>     try { impl_->videoLoop(); }
>     catch (const std::exception& e) { emit("视频线程异常: " + std::string(e.what())); }
> });
> ```
>
> **C++ 里如果线程函数抛出异常而没有捕获，程序会直接 `std::terminate()` 崩溃**（这是最容易忽略的知识点之一：异常不能跨线程传播）。很多"直播软件推送几分钟后闪退"就是这么来的。**第十七章 Q8 会让你补上。**

### 11.9 步骤 13：报告成果

```cpp
say("推流已启动: " + std::to_string(impl_->config.width) + "x"
    + std::to_string(impl_->config.height) + " @"
    + std::to_string(impl_->config.fps) + "fps, "
    + std::to_string(impl_->output.connectedCount()) + "/"
    + std::to_string(config.targets.size()) + " 路");

return true;
```

**为什么用 `say` 而不是 `emit`？** 两者效果一样（`say` 内部最终调用 `emit`）。这里用 `say` 更多是风格统一：`start()` 从头到尾只用 `say`。

**为什么这条日志放在线程启动之后？**

因为放前面容易误导用户："日志说启动了，结果——对，确实是启动了"。不过同学们注意：**这条日志是在线程已经跑起来之后才打印的，所以调用方收到这条日志时，第一批数据包很可能已经在队列里了。** 这在 UI 上是有意义的：你看到"推流已启动"的时候，是真的开始了。

顺带复习一下**内存**：注意这里混用了 `std::to_string` 和字符串拼接。这种写法效率不高（会创建一堆临时 string），但**每上线只执行一次**，不值得优化。**优化要看是不是热点，别看到 string 拼接就紧张。**

### 11.10 start() 完整流程图

```
start(config, log)
 │
 ├─① stop()                       ← 幂等清理
 ├─② 拷 config / move log / 清 lastError
 ├─③ 造 say / fail 两个 lambda
 │
 ├─④ 校验宽高帧率 ─────失败→fail（含 stop 回滚）
 ├─④ 校验 targets 非空 ───失败→fail
 ├─⑤ 宽高对齐偶数
 │
 ├─⑥ open videoEncoder ───失败→fail
 ├─⑦ open audioEncoder ───失败→fail
 ├─⑧ open output（依赖 ⑥⑦ 的 codecParameters）──失败→fail
 │
 ├─⑨ init screen 或 open camera ──失败→fail
 ├─⑩ open audio ────────────────失败→fail
 │
 ├─⑪ timeBeginPeriod(1) + timerRaised=true
 │
 ├─⑫ running = true; clockStart = now();
 │     ↓（此刻之前，新线程需要的一切都准备好了）
 ├─⑬ 创建三条线程
 │
 └─⑭ 打印启动日志 + return true
```

同学们把这个顺序图看熟。**尤其是 ⑥⑦⑧⑨⑩ 的顺序（依赖顺序）和 ⑫⑬ 的顺序（必须先置状态后起线程），这两处是这个函数真正的难点。**

---

## 第十二章 stop() 与其余五个成员

`stop()`（第 420~470 行，51 行）是 `start()` 的镜像。多线程里最难的部分就是收尾，所以这一章要讲得比 `start()` 更细。

### 12.1 stop() 全貌

```cpp
void StreamPipeline::stop()
{
    const auto lowerTimer = [this]() {
        if (impl_->timerRaised) {
            timeEndPeriod(1);
            impl_->timerRaised = false;
        }
    };

    // ── 分支 A：压根没跑过，只做清理 ────────────────────────
    if (!impl_->running && !impl_->videoThread.joinable()
        && !impl_->audioThread.joinable() && !impl_->writeThread.joinable()) {
        impl_->screen.close();
        impl_->camera.close();
        impl_->audio.close();
        impl_->videoEncoder.close();
        impl_->audioEncoder.close();
        impl_->output.close();
        impl_->videoQueue.clear();
        impl_->audioQueue.clear();
        impl_->releaseScale();
        lowerTimer();
        return;
    }

    // ── 分支 B：正在跑（或线程还没回收），走完整的停止流程 ──
    impl_->running = false;

    if (impl_->videoThread.joinable())
        impl_->videoThread.join();
    if (impl_->audioThread.joinable())
        impl_->audioThread.join();
    if (impl_->writeThread.joinable())
        impl_->writeThread.join();

    impl_->output.flush();

    impl_->screen.close();
    impl_->camera.close();
    impl_->audio.close();

    impl_->videoEncoder.close();
    impl_->audioEncoder.close();

    impl_->output.close();

    impl_->videoQueue.clear();
    impl_->audioQueue.clear();
    impl_->releaseScale();

    lowerTimer();
    impl_->emit("推流已停止");
}
```

### 12.2 lowerTimer：一个小 lambda 解决"成对注销"

```cpp
const auto lowerTimer = [this]() {
    if (impl_->timerRaised) {
        timeEndPeriod(1);
        impl_->timerRaised = false;
    }
};
```

这是整个函数里最容易被忽略、但设计得最好的一小段。

**为什么要用 lambda？** 因为 `timeEndPeriod` 需要在**两个 return/出口**执行：分支 A 要调，分支 B 也要调。如果不抽出来，就得在两个地方各写一遍（违反 DRY）；如果只写在末尾，分支 A 的提前 `return` 就漏了。

> 这个手法叫**"出口清理 lambda"**（类似 Go 语言的 `defer`，或者 RAII 的穷人版）。同学们记住它：**当一个函数有多个出口、每个出口都要做相同的清理时，抽一个 lambda 放在最前面**，这样既清晰又不会漏。

**为什么 `timerRaised` 判空是必须的？**

因为 `timeBeginPeriod` / `timeEndPeriod` 是**引用计数式**的系统 API：**调 N 次 begin 就要调 N 次 end**（Windows 内部为每个请求计数）。如果我们在 `timerRaised == false` 时也调 end，就会把计数减成负数或者破坏别的模块的请求。

> 举例：假如 Qt 界面库自己也调了 `timeBeginPeriod(1)` 保持 counters=1。我们再 begin 一次 → 计数=2。如果我们无脑 end 两次 → 计数=0， Qt 那边的 1ms 精度被我们破坏了（它绝对想不到）。**这就是为什么"成对调用"必须严格。**

### 12.3 分支 A：没跑过，只清理

```cpp
if (!impl_->running && !impl_->videoThread.joinable()
    && !impl_->audioThread.joinable() && !impl_->writeThread.joinable()) {
    ... 各种 close ...
    lowerTimer();
    return;
}
```

**这个条件的含义非常巧妙**，我们要花点时间读懂它。

它是"**我们真的启动过吗？**"的判断，用了四个条件的"与"：

| 条件 | 含义 |
|---|---|
| `!running` | 停止标志是 false（可能是从没启动，也可能是已经停了） |
| `!videoThread.joinable()` | 视频线程要么没创建过，要么**已经被 join 过了** |
| `!audioThread.joinable()` | 同上 |
| `!writeThread.joinable()` | 同上 |

**为什么要查 joinable 而不是只查 running？**

因为存在一种状态：**running 已经是 false，但线程还没被 join**（比如上一次 `stop()` 执行到一半被打断，或者 start 失败时 `fail` lambda 调了 stop 但线程已 arrived 但还没回收）。

> 更关键的洞察：**`std::thread::join()` 之后，该对象就不再是 joinable 的。** 所以"三个都不可 join"精确地表达了"**现在没有任何一条鲜活且尚未回收的线程**"这一状态。这是 §2.4 讲的"默认构造的 thread 不可 joinable"这个语义的实战运用。

**然后看清理顺序（同学们记一下）：**

```cpp
impl_->screen.close();
impl_->camera.close();
impl_->audio.close();
impl_->videoEncoder.close();
impl_->audioEncoder.close();
impl_->output.close();
impl_->videoQueue.clear();
impl_->audioQueue.clear();
impl_->releaseScale();
```

顺序是 **"先关数据源 → 再关编码器 → 再关输出 → 最后清队列和缩放上下文"**。

为什么？

1. **先关采集**：让它们停止往内部缓冲里写，释放硬件资源（摄像头指示灯会灭）。
2. **再关编码器**：编码器内部可能还有 pending 数据，`close()` 会把它们释放（这里不做 flush——注意这里**不再 flush**，因为分支 A 意味着没有线程在跑，之前没 flush 的话数据就丢了。这是合理的：既然没启动成功，也没必要 flush）。
3. **再关输出**：`output.close()` 内部会写 trailer 并关闭文件/网络。
4. **最后清队列**：队列里可能还有上次残留的包，`clear()` 释放它们的内存（**`AVPacketPtr` 是 `unique_ptr`，`clear()` 会逐个析构、自动 `av_packet_free`**——零泄漏，这就是 RAII 的便利）。

**为什么所有 close 都不检查返回值？** 因为它们是 `void`，而且都被设计成幂等的。**再次体会"幂等设计"带给调用方的轻松感：调用方可以无条件调用，不用管状态。**

**注意分支 A 不打印"推流已停止"**。因为根本没有推过流，打印这个日志会让用户困惑（"我都没开始推，怎么就停止了"）。**这就是"日志要考虑读者的感受"。**

### 12.4 分支 B：跑着呢，走完整流程

这是本课的第二核心，我们把它拆成五步。

#### 第 1 步：发通知

```cpp
impl_->running = false;
```

一句话，但它是**整个停止流程的起点**。三条线程会在各自的下一轮循环（或者超时唤醒后）看到它。

注意这里有个**重要的时序特性**：`running = false` 之后，线程**不会立刻停下**——它们可能正卡在某个阻塞调用里（摄像头 1 秒、WASAPI 100ms、网络写 more）。这是"协作式停止"（cooperative cancellation）的本质：**发出请求，等待对方配合，而不是强制杀死**。

> **为什么不能用"杀死线程"？** 因为被杀的线程可能正持有锁（死锁）、正写一半文件（文件损坏）、或者刚 malloc 完还没释放（内存泄漏）。Windows 的 `TerminateThread`、pthread 的 `pthread_cancel` 都是"最后手段"，正经程序一律不用。**协作式停止是唯一正确的方式。**

#### 第 2 步：汇合三条线程

```cpp
if (impl_->videoThread.joinable())
    impl_->videoThread.join();
if (impl_->audioThread.joinable())
    impl_->audioThread.join();
if (impl_->writeThread.joinable())
    impl_->writeThread.join();
```

三个 if + join，**顺序是 video → audio → write**，这个顺序是经过设计的：

1. **`join()` 会阻塞**，直到对应线程的线程函数返回。
2. **先 join 生产者（video/audio）**：保证它们都执行完了 `flush()`（第八章 §8.7、第九章 §9.3 讲的最后几步），所以**最后的数据一定进了队列**。
3. **最后 join 消费者（write）**：因为 `writeLoop` 的退出条件是"running 为假 **且** 两个队列都空"，它会自动把队列排干。**它排在最后 join，正好给了它排干的机会。**

> **这就是 §2.14 讲的"②③ 顺序陷阱"的实际落地。** 我们把这段完整推演一遍（假设此时两个队列里各有 5 个包）：
>
> ```
> T0: stop() 设 running=false
> T1: videoLoop 正在 (可能阻塞≤1s) → 完成当前帧 → while 条件 false → flush() → 推入剩余包 → 线程函数返回
> T2: audioLoop 正在 nextChunk(≤100ms) → while 条件 false → flush() → 推入剩余包 → 返回
> T3: join(videoThread) 返回
> T4: join(audioThread) 返回
> T5: writeLoop 发现 running=false 但队列非空 → 继续取包写出 → 直到两个队列都空 → 退出
> T6: join(writeThread) 返回
> T7: 此时所有数据都已写出
> ```
>
> **如果顺序反过来（先 join write）会怎样？** writeLoop 可能在生产者还没 flush 完时就发现"两个队列暂时为空"而退出，然后生产者 flush 出来的最后几个包**永远留在队列里没人要**——录出的视频结尾会缺最后一小段，或者缺最后一个 GOP 的一部分。**这是非常隐蔽的 bug，第十七章 Q1 会详细讨论。**

**为什么每个 join 前要判断 joinable？**

因为 `std::thread::join()` 有个铁律：**只能 join 一次，第二次会抛 `std::system_error`。** 如果不判就写死三行 join，那么：

- 假如上次 `stop()` 已经 join 过了（本次是第二次 stop），再次 join 会**抛异常**；
- 假如线程根本没创建过（`start()` 失败得早），join 也会抛异常。

所以用 `joinable()` 做守卫。**这是本课最标准的 `std::thread` 用法，同学们照抄即可。**

> **一个必须注意的历史遗留问题**：`join()` 之后，`std::thread` 对象本身还在（它不代表任何线程了）。**本课没有对它做 `= std::thread()` 重置**，但因为 joinable 会变成 false，逻辑上已经安全。另一方面，**分支 A 的判断正依赖于"join 过之后 joinable 变 false"这一点**，所以这个设计是自洽的。

#### 第 3 步：flush 输出

```cpp
impl_->output.flush();
```

这是第 10 课讲的 `StreamOutput::flush()`——把 FFmpeg 内部 interleaving 缓冲区里的**最后几个包**强制写出去。

**为什么必须在 join 之后、close 之前？**

- 必须在 join 之后：因为 write 线程可能刚写完最后一个包，ffmpeg 内部还压着几个没 flush 出去（interleaving queue）。
- 必须在 close 之前：因为 `close()` 会销毁 `AVFormatContext`，里面的东西就再也出不来了。

> 这是典型的三明治顺序：**join → flush → close**。同学们如果以后做类似的事情（比如数据库事务），类比一下：**等所有写入完成 → 提交事务 → 关闭连接**。

#### 第 4 步：关资源（和分支 A 一样的七件事）

```cpp
impl_->screen.close();
impl_->camera.close();
impl_->audio.close();

impl_->videoEncoder.close();
impl_->audioEncoder.close();

impl_->output.close();

impl_->videoQueue.clear();
impl_->audioQueue.clear();
impl_->releaseScale();
```

`output.close()` 内部会写 trailer（`av_write_trailer`）——**这一步决定了 FLV 文件能不能被播放器正确识别。** 忘了这一步，`ffprobe` 会报"moov atom not found"之类，或者文件时长为 0。

#### 第 5 步：注销定时器 + 打日志

```cpp
lowerTimer();
impl_->emit("推流已停止");
```

**为什么最后才注销定时器？** 因为理论上 join 期间某些底层（比如 WASAPI 的 worker 线程，它是 `AudioCapturer` 内部的，不归我们 join）还在等待-event。提前注销不影响正确性，但放在最后最保险。

**为什么这里用 `emit` 而不是 `say`？** 因为 `say` 是 `start()` 里的局部变量，这里没有。而 `emit` 是 Impl 的成员方法，语义相同（都是线程安全日志）。

> **这条日志同时也是给自动化测试用的信号**。后面の test 里如果打OH 日志收集，就能在线性 Casserole 里面检查有没有出现它。第十七章 Q6 会让你加这个断言。

### 12.5 析构函数：最后一道防线

```cpp
StreamPipeline::~StreamPipeline()
{
    stop();
}
```

只有一行，但它是**整个多线程安全性的最后一道保险**。

**为什么必须要有？**

因为如果不析构时不 stop，那么：

1. `std::thread` 对象析构时如果还 joinable → **`std::terminate()` 崩溃**（§2.4 讲过）；
2. 就算侥幸不崩，线程还在跑而 `Impl` 已经被 `unique_ptr` 析构了 → **use-after-free**，而且 lambda 捕获的就是 `Impl*`（§11.8 讲过）；
3. 各种采集器/编码器/输出的资源泄漏。

所以这一行保证了：**无论调用方怎么写（哪怕忘了调 stop 就 delete 对象、或者异常导致提前返回），我们都能优雅收场。**

> 这就是 RAII 思想的极致体现：**把"必须做的事"放进析构函数，让语言替你保证执行。** 同学们记住：
> **凡是"持有资源（线程/锁/文件/句柄）的类，析构函数必须释放它。"**

### 12.6 isRunning()

```cpp
bool StreamPipeline::isRunning() const
{
    return impl_->running;
}
```

一行。`impl_->running` 是 `std::atomic<bool>`，**隐式转换为 bool**（`operator bool()`），所以能直接 return。

**注意它的语义：`running` 是在 `start()` 末尾置 true、在 `stop()` 开头置 false。** 它不是"线程还活着"的精确指示——在 `stop()` 执行期间，`running` 已经 false 了但线程还在 join。**这是一个约定差异，文档里要说清楚**（测试的第 787 行 `check(!pipeline.isRunning(), "停止后状态正确")` 就是在验证这个约定）。

### 12.7 stats()：把 atomic 翻译成快照

```cpp
PipelineStats StreamPipeline::stats() const
{
    PipelineStats result;
    result.videoFramesCaptured = impl_->videoFrames.load();
    result.videoPacketsEncoded = impl_->videoPackets.load();
    result.videoFramesDropped = impl_->videoDropped.load();
    result.audioChunksCaptured = impl_->audioChunks.load();
    result.audioPacketsEncoded = impl_->audioPackets.load();

    const std::vector<OutputTargetInfo> targets = impl_->output.targets();
    result.totalTargets = static_cast<int>(targets.size());
    result.connectedTargets = impl_->output.connectedCount();

    if (impl_->running) {
        result.elapsedSeconds =
            static_cast<double>(msSince(impl_->clockStart, Clock::now())) / 1000.0;
    }

    return result;
}
```

**（1）为什么每个都要 `.load()`？**

`std::atomic<int64_t>` 到普通 `int64_t` 之间有隐式转换，`result.x = impl_->videoFrames;` 也能编译通过。但**显式写 `.load()` 有三个好处**：

- **可读性**：一眼看出这是原子读；
- **明确内存序**：默认 `load()` 用 `memory_order_seq_cst`（顺序一致，最保守）；如果将来要优化成 `load(std::memory_order_relaxed)`（放松内存序，对纯统计来说足够了，性能更好）就很自然；
- **不会误写**：避免不小心写成 `impl_->videoFrames = result.x` 这种反向赋值。

> **性能小知识**：`memory_order_seq_cst` 的 load 在 x86 上其实就是一条普通 mov（几乎零成本）；在 ARM 上会插入内存屏障。所以这里的写法在当前平台上已经足够好。**不要为了微优化把代码写得没人看得懂，除非你用 profiler 证明它是瓶颈。**

**（2）`output.targets()` 返回的是快照 vector**

第 10 课讲过，`StreamOutput::targets()` 会把内部的 `Target` 结构**拷贝**成 `OutputTargetInfo` 列表返回。这里是 `const std::vector<OutputTargetInfo> targets = ...`（**注意 const！**）——表明我们不会修改它。

> **注意这里有个隐含的线程安全问题**：`stats()` 可能由 UI 线程调用，而 `output` 由 write 线程使用。**`StreamOutput::targets()` 内部有没有加锁？** 第 10 课的实现里，我记得它是直接遍历成员的。这是一个**潜在的数据竞争**（如果 write 线程正在写失败而修改 `target.lastError`，UI 同时读就会读到撕裂状态）。这是本课的已知局限（第十七章 Q5）。

**（3）`elapsedSeconds` 只在 running 时计算**

```cpp
if (impl_->running) {
    result.elapsedSeconds =
        static_cast<double>(msSince(impl_->clockStart, Clock::now())) / 1000.0;
}
```

为什么？因为 `clockStart` 只在 `start()` 时被赋值，**停止之后它依然保留着上一次的值**。如果不加这个 if，停止之后调 `stats()` 会得到一个"从启动到现在的时间"不断增长的荒谬结果。

> **这是一个重要的使用约定**：**要拿最终数据，必须在 `stop()` 之前调 `stats()`。**
>
> 测试里正是这么做的（第 776 行：先 `const PipelineStats finalStats = pipeline.stats();`，第 779 行才 `pipeline.stop();`）。**顺序反了的话，`finalStats.elapsedSeconds` 就是 0，后面算帧率时会除以 0（得到 0.0），所有断言全挂。** 同学们做作业时最容易在这里栽跟头。
>
> 顺便复习一下第 790~794 行的写法：
> ```cpp
> const double capturedFps =
>     finalStats.elapsedSeconds > 0
>         ? static_cast<double>(finalStats.videoFramesCaptured) / finalStats.elapsedSeconds
>         : 0.0;
> ```
> 这里用了三元表达式做除零保护（而不是像 §8.1 那样用 `std::max`）。**两种写法目的相同：不让除零发生。**

### 12.8 reconnectDisconnected()

```cpp
int StreamPipeline::reconnectDisconnected()
{
    return impl_->output.reconnectDisconnected();
}
```

一行转发（delegation）到第 10 课的 `StreamOutput::reconnectDisconnected()`，返回"重连成功了几个"。

**这体现了管线的角色**：它自己是**协调者**（orchestrator），具体的事让专业模块去做。UI 层将来可以每隔几秒调一次这个方法处理抖动的 RTMP 连接。

> 注意它也跨越了线程边界：这个调用来自 UI 线程，而 `output` 归 write 线程。**如果 write 线程此刻正在 `writeVideo()` 里遍历 targets，同时又有重连在修改 targets 列表，就会有数据竞争。** 这是本课的真实局限之一。**正确的修法是给 StreamOutput 加一把锁，或者用消息队列把"重连"这个请求交给 write 线程执行**（第十七章 Q5、Q7 会讨论）。这也是后面某次 commit 要改进的重点。

### 12.9 lastError()

```cpp
const std::string& StreamPipeline::lastError() const
{
    return impl_->lastError;
}
```

一行，返回内部字符串引用（第六章 §6.3 讲过悬垂引用的坑）。

注意到它是 **`const` 成员函数**：不修改任何成员。这类"纯 getter"都应该标 const，**让编译器帮你保证不会因为手滑改了状态**。

### 12.10 start / stop 的对称性总表

最后给同学们一张"对照表"，这是理解整个 `StreamPipeline` 生命周期的关键：

| 资源 | `start()` 里 | `stop()` 里 | 对称吗 |
|---|---|---|---|
| `running` | = true（第 404 行） | = false（第 444 行） | ✅ |
| `clockStart` | = now()（第 405 行） | 不重置 | ⚠️ 靠 `if (running)` 规避 |
| `videoThread` | 创建（第 407 行） | `join()`（第 447 行） | ✅ |
| `audioThread` | 创建（第 408 行） | `join()`（第 449 行） | ✅ |
| `writeThread` | 创建（第 409 行） | `join()`（第 451 行） | ✅ |
| `lastError` | `clear()`（第 345 行） | 由 `fail` 写入 | ✅ |
| `timerRaised` | = true（第 402 行） | `lowerTimer()` 置 false | ✅ |
| `video/audioEncoder` | `open()` | `close()` | ✅ |
| `output` | `open()` | `flush()` + `close()` | ✅ |
| `screen/camera/audio` | `init()` / `open()` | `close()` | ✅ |
| 两个队列 | 隐式可用 | `clear()` | ✅ |
| `scaleContext` | 懒创建 | `releaseScale()` | ✅ |
| `log` | `move` 进来 | 不清理 | ⚠️ 保留给下次（合理） |
| 五个计数器 | 不重置 | 不重置 | ⚠️ **重复 start 会累加**（见测试第 3 段） |

最后一行值得单独说：**计数器不重置，意味着"连续启停三轮"之后 `stats().videoFramesCaptured` 是三轮的总和。** 这在某些场合是 bug（第二次推流的统计不对），在另一些场合是特性（累计统计）。**本项目没说清楚，属于设计意图不明确。**

> 好消息是，测试第 3 段（第 863 行）的判断用的是 `if (pipeline.stats().videoFramesCaptured == 0) allOk = false;`——即"只要不是 0 就算正常"，所以它对累计不敏感。**同学们如果要做严格的作业，建议把计数器的重置写进 `start()` 或 `stop()`**（第十七章 Q6）。

---
## 第十三章 test_pipeline.cpp：端到端验收

298 行的测试文件。这是整个项目到目前为止**最重要的一个测试**——它不是测某个模块，而是测"整个系统能不能工作"。我们按三段（外加一个工具函数）来讲。

### 13.0 前置：极简断言框架与 `probeFlv` 工具

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

void check(bool condition, const std::string& what)
{
    check(condition, what.c_str());
}
```

**（1）为什么要有两个重载？**

因为第 822 行以后会有这样的调用：

```cpp
check(a.videoPackets == b.videoPackets, "两路视频包数完全一致");        // const char*
check(finalStats.videoFramesCaptured >= 70, "3 秒采到了 70 帧以上..."); // const char*
```

以及将来可能拼字符串的：

```cpp
check(x, "第 " + std::to_string(i) + " 轮通过");   // std::string
```

如果没有 `std::string` 的重载，第二种调用就得写成 `.c_str()`。**这两个重载一共 15 行，省掉了调用方无数个 `.c_str()`。**

> 顺带复习一个 C++ 知识点：**`const std::string&` 参数可以接收 `const char*` 字面量**（因为有隐式构造），所以即使没有第一个重载也能工作；反过来不行（`const char*` 参数接不了 `std::string`，除非 `.c_str()`）。**那为什么还要第一个重载？** 因为第一个重载避免了每次调用都临时构造一个 `std::string`（可能有内存分配）。这叫**"用重载消除临时对象"**。

**为什么用全局变量 `g_passed` / `g_failed`？**

因为它们被放在匿名 namespace 里（内部链接），理论上也可以用 `static`。测试程序追求简短，全局变量是最直接的写法。**注意这里每条 assertions 都 immediately 打印**，这是为了让测试过程"看得见"——你能实时看到进行到哪一步了。**这个细节很重要：长时间的黑屏会让使用者以为程序死了。**

**（2）`FlvInfo` 与 `probeFlv`：把写出去的文件再读回来**

这是本课的"验收工具"。思路是：**我们写的 FLV 文件好不好，不能自己说了算，要交给 FFmpeg 的解封装器（`libavformat`）去读一遍，看它认不认。**

```cpp
struct FlvInfo {
    bool ok = false;
    int streamCount = 0;
    bool hasVideo = false;
    bool hasAudio = false;
    int videoPackets = 0;
    int audioPackets = 0;
    int64_t lastVideoMs = 0;
    int64_t lastAudioMs = 0;
    bool sawNegativePts = false;
};
```

九个字段，覆盖了我们要验收的全部维度：能不能打开、有几条流、有没有视频、有没有音频、各多少包、各自的**最后时间戳**（用来判时长和音画漂移）、有没有出现负时间戳。

```cpp
FlvInfo probeFlv(const std::string& path)
{
    FlvInfo info;

    AVFormatContext* ctx = nullptr;
    if (avformat_open_input(&ctx, path.c_str(), nullptr, nullptr) < 0)
        return info;

    if (avformat_find_stream_info(ctx, nullptr) < 0) {
        avformat_close_input(&ctx);
        return info;
    }

    info.streamCount = static_cast<int>(ctx->nb_streams);

    int videoIndex = -1;
    int audioIndex = -1;
    for (unsigned i = 0; i < ctx->nb_streams; ++i) {
        if (ctx->streams[i]->codecpar->codec_type == AVMEDIA_TYPE_VIDEO && videoIndex < 0)
            videoIndex = static_cast<int>(i);
        if (ctx->streams[i]->codecpar->codec_type == AVMEDIA_TYPE_AUDIO && audioIndex < 0)
            audioIndex = static_cast<int>(i);
    }
    info.hasVideo = videoIndex >= 0;
    info.hasAudio = audioIndex >= 0;

    AVPacket* packet = av_packet_alloc();
    while (av_read_frame(ctx, packet) >= 0) {
        if (packet->pts != AV_NOPTS_VALUE && packet->pts < 0)
            info.sawNegativePts = true;

        if (packet->stream_index == videoIndex) {
            ++info.videoPackets;
            if (packet->pts != AV_NOPTS_VALUE)
                info.lastVideoMs = std::max(info.lastVideoMs, packet->pts);
        } else if (packet->stream_index == audioIndex) {
            ++info.audioPackets;
            if (packet->pts != AV_NOPTS_VALUE)
                info.lastAudioMs = std::max(info.lastAudioMs, packet->pts);
        }
        av_packet_unref(packet);
    }

    av_packet_free(&packet);
    avformat_close_input(&ctx);
    info.ok = true;
    return info;
}
```

我们逐段解释里面的门道。

**第一步：打开**

```cpp
AVFormatContext* ctx = nullptr;
if (avformat_open_input(&ctx, path.c_str(), nullptr, nullptr) < 0)
    return info;
```

`avformat_open_input` 是 FFmpeg 的**解封装**(demux) 入口，和我们在第 10 课用的 `avformat_alloc_output_context2`（封装 mux）正好是一对。

注意失败时直接 `return info`——此时 `info.ok` 还是 `false`，调用方能据此判断"文件打不开"。

> **为什么不 throw？** 因为这是 C API 风格的封装，且这里是测试代码——返回带状态的 struct 比异常更简洁。**风格选择而已，没有对错。**

**第二步：找流信息**

```cpp
if (avformat_find_stream_info(ctx, nullptr) < 0) {
    avformat_close_input(&ctx);
    return info;
}
```

`avformat_find_stream_info` 会**实际读取文件头部的若干数据包**来推断各条流的编码参数。这一步通常需要读 1~2 个包。

**为什么需要它？** 因为容器头部不一定写着全部信息（比如 FLV 的 metadata 可能缺失，但 H.264 的 SPS/PPS 在视频 tag 里），不调用的话 `codecpar` 可能是空的。

**注意这段的错误处理**：失败时**先 `avformat_close_input` 再 return**。这防止了 `AVFormatContext` 泄漏。**这就是为什么每一个可能失败的 FFmpeg API 都要配上对应的 close/release。**

**第三步：遍历流，找出视频流和音频流的下标**

```cpp
int videoIndex = -1;
int audioIndex = -1;
for (unsigned i = 0; i < ctx->nb_streams; ++i) {
    if (ctx->streams[i]->codecpar->codec_type == AVMEDIA_TYPE_VIDEO && videoIndex < 0)
        videoIndex = static_cast<int>(i);
    if (ctx->streams[i]->codecpar->codec_type == AVMEDIA_TYPE_AUDIO && audioIndex < 0)
        audioIndex = static_cast<int>(i);
}
info.hasVideo = videoIndex >= 0;
info.hasAudio = audioIndex >= 0;
```

`&& videoIndex < 0` 这个附加条件保证了**只取第一个**视频流/音频流（如果文件里有两条视频流，我们只看第一条）。这是一个"取首次出现"的技巧。

**类型安全的细节**：`i` 是 `unsigned`（因为 `nb_streams` 是 `unsigned int`），赋给 `int` 时要 `static_cast<int>(i)`。**避免隐式窄化转换的编译警告。** 同学们注意本项目处处有这种显式转换，这是 `-Wconversion` 友好项目的特征。

**第四步：逐个读包，统计**

```cpp
AVPacket* packet = av_packet_alloc();
while (av_read_frame(ctx, packet) >= 0) {
    ...
    av_packet_unref(packet);
}
av_packet_free(&packet);
```

这是 FFmpeg **解封装的标准循环**：`av_read_frame` 一次读一个包，返回 >= 0 表示成功。

三个必须做对的动作：

1. **`av_packet_alloc()`**：分配一个包对象。
2. **`av_packet_unref(packet)`**（循环体内每一轮）：把包里引用的数据缓冲**减引用计数**。漏了它 → 内存泄漏且后续读到脏数据。
3. **`av_packet_free(&packet)`**（循环结束后）：释放包对象本身。**注意它接收的是 `AVPacket**`（二级指针），因为函数内部会把指针置为 nullptr。**

> 顺带复习第 9 课：本项目用 `AVPacketPtr = unique_ptr<AVPacket, AVPacketDeleter>` 管理包，删除器里调用的就是 `av_packet_free`。**这里因为要用 FFmpeg 的 read API（它要求 `AVPacket*` 裸指针且会自己 unref），所以手写管理更方便。** 同学们要能根据场合选择：用哪个舒服就用哪个，关键是**配对**。

**统计逻辑：**

```cpp
if (packet->pts != AV_NOPTS_VALUE && packet->pts < 0)
    info.sawNegativePts = true;

if (packet->stream_index == videoIndex) {
    ++info.videoPackets;
    if (packet->pts != AV_NOPTS_VALUE)
        info.lastVideoMs = std::max(info.lastVideoMs, packet->pts);
} else if (packet->stream_index == audioIndex) {
    ++info.audioPackets;
    if (packet->pts != AV_NOPTS_VALUE)
        info.lastAudioMs = std::max(info.lastAudioMs, packet->pts);
}
```

四个知识点：

1. **`AV_NOPTS_VALUE`**（Not a valid PTS value）：FFmpeg 里表示一个"无效时间戳"的常量，实际值是 `INT64_MIN`。**任何时候用 pts 之前都要先判断它不是 AV_NOPTS_VALUE**——拿 -9223372036854775808 去做算术会得到荒谬的结果（比如 `std::max` 会把 MAX 值变成正无穷级别）。本课统计"最大时间戳"用 `std::max` 而不是"+="，正好避开了溢出风险。

2. **为什么这里能直接把 pts 当毫秒用？** 因为 FLV 的时间基就是 `{1, 1000}`（第 10 课讲的 `kFlvTimeBase`），而 `av_read_frame` 返回的 packet 的 pts 已经是**换算到流时间基之后**的值。所以 `packet->pts` 的单位就是毫秒。**这就是为什么后面可以直接和 3000（3 秒 = 3000 毫秒）比较。**

3. **`std::max` 而不是"最后的包"**：用"最大值"而不是"最后一个包的 pts"。为什么？因为**理想情况下最后一个包的 pts 最大，但如果容器里有乱序（B 帧），最后一个包的 pts 未必最大**。用 max 更稳健。

4. **`sawNegativePts` 检测**：这是我们给" AAC 的 initial_padding 导致负时间戳"这个坑设的警报器。如果第 10 课的 `audioOffset` 归零逻辑失效了，这个字段会立刻变成 true，断言就会挂。**这就是"为已知的坑写回归测试"。**

### 13.1 main() 的开场：三件小事

```cpp
int main()
{
#if defined(_WIN32)
    std::system("chcp 65001 > nul");
#endif

    static char stdoutBuffer[8192];
    std::setvbuf(stdout, stdoutBuffer, _IOLBF, sizeof(stdoutBuffer));

    std::printf("================================================\n");
    std::printf("  模块 9：完整推流管线测试\n");
    std::printf("================================================\n");
    std::printf("  屏幕采集 + 系统声音 -> 编码 -> 封装 -> 两路本地 flv\n");
    std::printf("  跑 3 秒，然后看帧数、时间轴、文件完整性。\n");
    std::printf("================================================\n");
```

**（1）`std::system("chcp 65001 > nul")`**

把 Windows 控制台的代码页改成 **65001 = UTF-8**。否则 printf 输出的中文会变成乱码（因为 Windows 中文版默认是 GBK 代码页 936，而源文件是 UTF-8）。

`> nul` 是把 chcp 命令自己的输出丢掉（否则会打印一行"Active code page: 65001"）。

> 这是本项目的标准开场动作，同学们在每个测试里都会看到。**改这个世界观的一个副作用：**如果你的 Windows Terminal / VS Code 终端已经支持 UTF-8，这行是多余的但无害。

**（2）`std::setvbuf(stdout, stdoutBuffer, _IOLBF, sizeof(stdoutBuffer))`**

把 stdout 的缓冲模式改成**行缓冲**（`_IOLBF` = I/O Line Buffered），并给一块 8KB 的缓冲区。

为什么？同学们可能会觉得"这是优化 I/O"——**不是，这是为了让输出及时出现**。

- 如果 stdout 是全缓冲（默认重定向到文件/管道时），printf 的内容会攒到 4KB 才刷一次，你 3 秒钟才可能看到一次输出；
- 改成行缓冲后，**每碰到一个 `\n` 就 flush**，你能实时看到测试进度。

> **注意那个 `static`**：`stdoutBuffer` 声明为 `static char[8192]`，因为 `setvbuf` **要求缓冲区在这个流关闭之前一直有效**。如果用局部数组（栈上），函数返回后 stdio 还握着一个野指针——这是经典的 use-after-scope bug。**这个 static 不是可有可无的。**

**（3）打印横幅（banner）**

Banner（横幅）是有意义的：它一行话说明了**这个测试要验证什么**（屏幕采集 + 系统声音 → 两路本地 flv）、**跑多久**（3 秒）、**然后看什么**（帧数、时间轴、文件完整性）。半年后回来看这个测试，不用读代码就知道它在干什么。这是很重要的工程习惯。

### 13.2 第一段：参数校验（第 704~721 行）

```cpp
std::printf("\n--- 1. 参数校验 ---\n");
{
    StreamPipeline pipeline;
    StreamConfig config;

    check(!pipeline.start(config, nullptr), "没有推流目标时拒绝启动");
    std::printf("  失败原因: %s\n", pipeline.lastError().c_str());
    check(!pipeline.isRunning(), "失败后没有处于运行状态");

    StreamConfig bad = config;
    StreamTarget target;
    target.platform = "A";
    target.url = "pipeline_bad.flv";
    bad.targets.push_back(target);
    bad.fps = 0;

    check(!pipeline.start(bad, nullptr), "帧率为 0 时拒绝启动");
}
```

三个检查，各有分工：

| 断言 | 验证了什么 |
|---|---|
| 没有目标时拒绝启动 | `start()` 第 359 行的 `targets.empty()` 检查生效 |
| 失败后没有处于运行状态 | `fail` lambda 里调用了 `stop()`，`running` 没有被误置为 true |
| 帧率为 0 时拒绝启动 | `fps <= 0` 检查生效 |

**为什么第二个断言很重要？** 因为它验证的是"失败路径不会留下半吊子状态"。如果一个 `start()` 失败了，但 `running` 却是 true，那 UI 上就会显示"正在推流"而实际根本没推——**这种"状态不一致"的 bug 是最难发现的一类**。

**为什么这里传 `nullptr` 作为 log 回调？**

这正好验证了 §7.3 讲的 `emit` 里的 `if (log)` 判空：**没有回调时管线不能崩溃**。这是"可选功能必须能被关闭"的测试。

**为什么用 `{}` 包成一个块？**

为了让 `StreamPipeline pipeline;` 在块的结尾析构——**这会触发析构函数里的 `stop()`**（§12.5）。所以这一段顺带验证了"**析构一个从没成功启动过的对象，不会崩溃**"。这是一个不起眼但极重要的测试路径（很多崩溃都发生在"空对象析构"上）。

### 13.3 第二段：跑一条完整管线（第 723~835 行）

这是本课的核心实验。我们分五小段看。

#### （1）配置与启动

```cpp
StreamConfig config;
config.sourceType = SourceType::Screen;
config.audioType = AudioSourceType::SystemAudio;
config.width = 1280;
config.height = 720;
config.fps = 30;
config.videoKbps = 2000;
config.audioKbps = 128;

StreamTarget targetA;
targetA.platform = "A";
targetA.url = "pipeline_a.flv";
StreamTarget targetB;
targetB.platform = "B";
targetB.url = "pipeline_b.flv";
config.targets = {targetA, targetB};

StreamPipeline pipeline;

const auto startTime = std::chrono::steady_clock::now();
const bool started = pipeline.start(config, log);
const long long startupMs =
    std::chrono::duration_cast<std::chrono::milliseconds>(
        std::chrono::steady_clock::now() - startTime)
        .count();

check(started, "管线启动成功");
std::printf("  启动耗时 %lld 毫秒（含编码器初始化 + 两路连接）\n", startupMs);
```

配置要点：

- **`1280x720 @30fps`**：偶数宽高（不用对齐）、常见的直播分辨率。
- **`videoKbps = 2000`**：720p 给 2Mbps 是合理的（1080p 通常给 4500）。
- **`SystemAudio`（系统声音/立体声混音）**：这样即使你没开麦克风，也能录到音乐播放器的声音。
- **两路输出都是本地文件**：`"pipeline_a.flv"` 和 `"pipeline_b.flv"`。用本地文件代替 RTMP 服务器是关键设计——**测试不依赖网络**。第 10 课讲过这个技巧（"用本地文件做替身"）。

**测启动耗时**：这是为了让读者直观知道"开锁一次要多久"。典型值 200~800ms（主要是两个 x264 编码器初始化 + 两次文件/R TMP 打开）。

#### （2）主循环：每 500ms 看一眼仪表盘

```cpp
for (int tick = 1; tick <= 6; ++tick) {
    std::this_thread::sleep_for(std::chrono::milliseconds(500));

    const PipelineStats s = pipeline.stats();
    std::printf("  [%4.1fs] 采集 %lld 帧，编码 %lld 包，丢 %lld 帧，"
                "音频 %lld 块 / %lld 包，连接 %d/%d\n",
                s.elapsedSeconds,
                static_cast<long long>(s.videoFramesCaptured),
                static_cast<long long>(s.videoPacketsEncoded),
                static_cast<long long>(s.videoFramesDropped),
                static_cast<long long>(s.audioChunksCaptured),
                static_cast<long long>(s.audioPacketsEncoded),
                s.connectedTargets, s.totalTargets);
}
```

**这一段 mimics（模仿）了真实 UI 的行为**：每 500ms 取一次 `stats()` 刷新界面。这是在测试"`stats()` 可以被**外部线程在运行时**安全调用"这一能力。

注意 printf 的格式化：

- `[%4.1fs]`：宽度 4、一位小数的秒数（如 ` 0.5s`、` 3.0s`）。
- `%lld`：64 位整数。**注意所有 `int64_t` 都被 cast 成了 `long long`** ——因为 `int64_t` 在 MSVC 上是 `long long`，在 GCC/Linux 上是 `long`，用 `%lld` 打印 `long` 会 UB。显式 cast 到 `long long` 就两边都安全。**这是跨平台 printf 的标准写法，同学们务必学会。**
- 一个容易忽视的细节：**这条 printf 是从 UI 线程（主线程）调的，而 5 个计数器正在被两条其他线程 `++`。** 正因为它们是 `atomic`，这里才敢这么读。**如果当初偷懒用了普通 `int64_t`，这个测试就会随机读到撕裂的值（32 位机器上尤其明显）。**

#### （3）停止并计时

```cpp
const PipelineStats finalStats = pipeline.stats();

const auto stopStart = std::chrono::steady_clock::now();
pipeline.stop();
const long long stopMs =
    std::chrono::duration_cast<std::chrono::milliseconds>(
        std::chrono::steady_clock::now() - stopStart)
        .count();

std::printf("  停止耗时 %lld 毫秒（要把队列里剩下的包写完）\n", stopMs);

check(!pipeline.isRunning(), "停止后状态正确");
check(stopMs < 3000, "停止过程没有卡死");
```

**两个顺序要点**（前面已经强调过，这里是它在代码里的样子）：

1. **`finalStats` 必须在 `stop()` 之前取**！否则 `elapsedSeconds` 是 0（§12.7）。
2. **测停止耗时**：这是本课的**第二个卡死探测器**。为什么阈值是 3000ms？

   我们来算理论上的最大停止时间：

   | 环节 | 最长耗时 |
   |---|---|
   | videoLoop 当前一轮（屏幕路径，节拍器剩余最多 33ms） | ≤ 33ms |
   | videoLoop 当前一轮（摄像头路径，`nextFrame(1000)`） | ≤ 1000ms |
   | audioLoop 当前一轮（`nextChunk(100)`） | ≤ 100ms |
   | writeLoop 排空队列（1080p 的话每包几十 ms） | 通常 < 500ms |
   | `output.flush()` + `close()`（写 trailer） | 通常 < 100ms |
   | **合计（屏幕路径）** | **≈ 700ms** |

   3000ms 是个宽松的上限：既给足了余量（避免 CI 机器上偶发超时），又足以抓住"彻底卡死"（比如某个线程无限阻塞）。**这种"用时间上限做死锁检测"的方法很实用，同学们写多线程测试时要学会用它。**

#### （4）帧率断言

```cpp
const double capturedFps =
    finalStats.elapsedSeconds > 0
        ? static_cast<double>(finalStats.videoFramesCaptured)
              / finalStats.elapsedSeconds
        : 0.0;

std::printf("  实际采集帧率 %.1f fps（目标 %d）\n", capturedFps, config.fps);

check(finalStats.videoFramesCaptured >= 70,
      "3 秒采到了 70 帧以上（30fps 的合理下限）");
check(capturedFps > 20.0 && capturedFps < 40.0,
      "实际帧率落在 20~40 之间，没有被编码拖垮");
check(finalStats.videoPacketsEncoded >= 60, "编码出了足够多的视频包");
check(finalStats.audioChunksCaptured > 150, "音频块数量合理（约 300 块）");
check(finalStats.connectedTargets == 2, "两路都保持着连接");
```

**为什么是"70 帧"而不是"90 帧"？**

30fps × 3 秒 = 90 帧是理论值。但：

- 第一帧之前有采集启动开销；
- 某一帧的编码可能超时导致后续赶工，产生少量丢帧；
- 屏幕路径下如果桌面完全没变化，虽然不会丢帧但也可能因为超时保护丢失个别帧。

**给 70 帧（78% 的下限）是有意为之的宽松**——测试的目的是"抓住彻底坏了"（比如只采到 3 帧），而不是"卡精确度"。**太严格的断言会让测试变成"flaky test"（随机失败），那比没有测试更糟。**

> **这是很重要的一条测试哲学**：**断言的严格程度要匹配"要检测的故障类型"。** 检测"完全不工作"用宽松阈值；检测"精度偏差"才用严格阈值，而且要放在单独的用例里。

**为什么"实际帧率 < 40"也要断言？**

这是在防"节拍器失效"！如果 `waitNextFrame` 写错了（比如摄像头路径忘了 `return`，或者 `sleep_until` 被删了），帧率会飙到 200+ fps，CPU 跑满。**这是一个"上限保护"，防止程序跑得比预期更快**——这跟大多数人"只测下限"的习惯相反，值得学习。

**音频为什么要 > 150 块？**

WASAPI 每块大约 10ms 的数据，**3 秒 ≈ 300 块**。给 150（一半）是因为某些环境下系统声音会断续或因为 `padSilence` 合并。同样，这是"抓住完全没声音"，不是"精确计数"。

#### （5）读回文件并检查时间轴

```cpp
std::printf("\n  读回两个文件：\n");
const FlvInfo a = probeFlv("pipeline_a.flv");
const FlvInfo b = probeFlv("pipeline_b.flv");

check(a.ok && b.ok, "两个 flv 都能被正常打开");
check(a.streamCount == 2 && b.streamCount == 2, "两个文件都是双流");
check(a.hasVideo && a.hasAudio, "视频流和音频流都在");

std::printf("    A: 视频 %d 包（末 %lld ms）/ 音频 %d 包（末 %lld ms）\n", ...);
std::printf("    B: 视频 %d 包（末 %lld ms）/ 音频 %d 包（末 %lld ms）\n", ...);

check(a.videoPackets == b.videoPackets, "两路视频包数完全一致");

std::printf("\n  时间轴对齐检查（目标 3 秒 = 3000 ms）：\n");
check(a.lastVideoMs >= 2700 && a.lastVideoMs <= 3400,
      "视频时间轴落在 3 秒附近");
check(a.lastAudioMs >= 2700 && a.lastAudioMs <= 3400,
      "音频时间轴落在 3 秒附近");

const int64_t drift = a.lastVideoMs - a.lastAudioMs;
std::printf("    视频与音频的时间差 %lld ms\n", static_cast<long long>(drift));
check(drift > -500 && drift < 500,
      "音视频时间差在 500ms 以内（没有累积漂移）");

check(!a.sawNegativePts && !b.sawNegativePts, "没有负时间戳");
```

这组断言是**对第五章到第十二章全部设计的最终验收**：

| 断言 | 验证了什么 | 如果挂了说明哪里坏了 |
|---|---|---|
| 两个文件都能打开 | `output.close()` 写了正确的 trailer | 忘了写 trailer（第 10 课） |
| 都是双流 | 音视频 `codecParameters` 都注册成功了 | `StreamOutput::open` 的参数有问题 |
| **两路视频包数完全一致** | **多目标分发的一致性** | 某一路写失败但没被发现 |
| 视频时间轴 2700~3400ms | **pts 换算正确**（frame → ms） | `timeBase()` 传错，或 pts 公式错 |
| 音频时间轴 2700~3400ms | 音频 pts 换算正确 | 重采样/帧长错了 |
| **drift 在 ±500ms** | **音视频同步** | 时间轴不统一 |
| 没有负时间戳 | `audioOffset` 归零生效 | AAC `initial_padding` 处理漏了 |

**注意"两路视频包数完全一致"这个断言的分量**：它验证的是"**三份数据（A 文件、B 文件、以及源数据）逐字节等价**"的一个代理指标。如果将来某一路因为写失败被静默丢弃（第 10 课讲的"失败隔离"），这个断言会立刻发现。**这是测试设计里非常漂亮的一手：用"多个独立副本的一致性"来验证"没有静默丢包"。**

### 13.4 第三段：重复启停（第 837~869 行）

```cpp
std::printf("\n--- 3. 重复启停 ---\n");
{
    StreamConfig config;
    config.sourceType = SourceType::Screen;
    config.audioType = AudioSourceType::SystemAudio;
    config.width = 640;
    config.height = 360;
    config.fps = 30;
    config.videoKbps = 800;

    StreamTarget target;
    target.platform = "loop";
    target.url = "pipeline_loop.flv";
    config.targets = {target};

    StreamPipeline pipeline;
    bool allOk = true;

    for (int round = 0; round < 3; ++round) {
        if (!pipeline.start(config, nullptr)) {
            allOk = false;
            std::printf("  第 %d 轮启动失败: %s\n", round + 1,
                        pipeline.lastError().c_str());
            break;
        }
        std::this_thread::sleep_for(std::chrono::milliseconds(400));
        if (pipeline.stats().videoFramesCaptured == 0)
            allOk = false;
        pipeline.stop();
    }

    check(allOk, "连续启动/停止三轮都正常");
}
```

**这段测的是本课最难、最容易出 bug 的路径：状态机的可重复性。**

三个设计要点：

1. **用 640x360 + 800kbps 的小配置**：因为这段的压力在"启停"而不在于"编码"，用小分辨率跑得快。
2. **每轮只跑 400ms**：够采到十几帧，又不至于让测试太慢。
3. **`allOk` 标志 + 一条汇总断言**：如果三轮里有一轮失败，最后只报一条"连续启动/停止三轮都正常 [失败]"，而不是 9 条断言混在一起。**这让输出更清爽。**

**这段能抓住哪些 bug？** 同学们想一下，如果我们的实现有下面这些 bug，哪一条会被这里抓到：

| 潜在 bug | 会被抓到吗 | 表现 |
|---|---|---|
| `start()` 忘了先 `stop()` | ✅ | 第二轮 start 时三个 thread 还在 joinable → join 抛异常或未定义 |
| `stop()` 后 `std::thread` 没被 join 过 | ✅ | 第二轮创建时 assignment 到一个 joinable 的 thread → `std::terminate` |
| `timeBeginPeriod` 没配 `timeEndPeriod` | ❌ | 不会崩，但会累积（可以用 GetTickCount 精度测，本项目没测） |
| 计数器没重置 | ❌ | 断言是 `== 0` 判断，累计反而"更容易通过"（见 §12.10） |
| 缩放上下文没释放 | ❌ | 内存泄漏，肉眼不可见（需要 Dr.Memory / VLD 这类工具检测） |

**这就是"测试覆盖率"的真实含义**：好的测试不能覆盖所有 bug，但**必须覆盖那些"最可能发生、后果最严重"的**。启动/停止的状态管理就属于这一类。

### 13.5 main() 的收尾

```cpp
std::printf("\n================================================\n");
std::printf("  通过 %d 项，失败 %d 项\n", g_passed, g_failed);
std::printf("================================================\n");
std::printf("  pipeline_a.flv / pipeline_b.flv 可以直接用播放器打开，\n");
std::printf("  应该是你的桌面画面加上这段测试期间的系统声音。\n");
std::printf("================================================\n");
std::fflush(stdout);

return g_failed == 0 ? 0 : 1;
```

最后三条尤其重要：

- **告诉人类结果文件在哪、里面应该有什么**。这继承了第 0 课立下的规矩：**测试必须产出"人能用眼睛验收"的东西**。所以写完测试不要只盯着 PASSED/FAILED，**一定要用播放器打开 `pipeline_a.flv` 看一眼**（第十四章就是干这个）。
- **`std::fflush(stdout)`**：确保所有输出真的刷出去（Ctrl+C 或 pipe 的情况下尤其重要）。
- **`return g_failed == 0 ? 0 : 1;`**：返回 0 表示成功。**CI（持续集成）就靠这个退出码判断通过与否。** 这是自动化集成的前提。

---
## 第十四章 动手环节：编译、运行、看输出

讲了这么多，终于轮到同学们自己动手了。这一章我们做四件事：编译、运行并逐行读懂输出、用工具验证产出、再把它推到真实的 RTMP 服务器。

### 14.1 编译

本项目自带 `build.bat`（第 1 课配的），在仓库根目录打开 **"x64 Native Tools Command Prompt for VS"**（普通 cmd 也行，只要 cmake 和编译器在 PATH 里）：

```bat
cd C:\path\to\Streaming-tools
build.bat
```

它等价于：

```bat
cmake -B build -S . -G "Visual Studio 17 2022" -A x64
cmake --build build --config Release
```

编译完成后，`core_pipeline.lib` 和 `test_pipeline.exe` 都会出现在 `build` 目录里。单独编译测试也可以：

```bat
cmake --build build --config Release --target test_pipeline
```

**编译过程中的两个"必修，"**

| 现象 | 原因 | 怎么修 |
|---|---|---|
| `LNK2019: 无法解析的外部符号 timeBeginPeriod` | 忘了链接 `winmm` | 检查 `src/core/pipeline/CMakeLists.txt` 里有没有 `target_link_libraries(core_pipeline PUBLIC winmm)`（§4.2 第四句） |
| `C2039: 'sws_getContext': 不是 'xxx' 的成员` | FFmpeg 头文件没被 `extern "C"` 包裹 | 检查 `StreamPipeline.cpp` 第 79~81 行（第七章 §7.1） |
| `C2280: 尝试引用已删除的函数 StreamPipeline::StreamPipeline(...)` | 你写了拷贝（比如放进 `std::vector<StreamPipeline>`） | 改用 `unique_ptr` 或引用（§6.3） |

### 14.2 运行与预期输出

```bat
cd build\tests\Release
test_pipeline.exe
```

一次健康的运行，输出大致是这样（数字会有差异，但结构完全一致）：

```
================================================
  模块 9：完整推流管线测试
================================================
  屏幕采集 + 系统声音 -> 编码 -> 封装 -> 两路本地 flv
  跑 3 秒，然后看帧数、时间轴、文件完整性。
================================================

--- 1. 参数校验 ---
  [管线] 推流参数无效：宽高和帧率都要大于 0
  [通过] 没有推流目标时拒绝启动
  失败原因: 没有配置推流目标
  [通过] 失败后没有处于运行状态
  [管线] 推流参数无效：宽高和帧率都要大于 0
  [通过] 帧率为 0 时拒绝启动
```

> **第一个要看的点**：虽然我们第一轮传的是"默认 config"（fps=30、宽高正常），但因为 `targets` 为空，先命中了 `targets.empty()` 检查。同学们对照 §11.4 想一想：为什么是"没有配置推流目标"而不是"参数无效"？因为第 355 行的宽高帧率检查在前，默认参数的宽高帧率都是合法的，所以一路走到了第 359 行。**这个顺序决定了报错的内容。**

```
--- 2. 跑一条完整管线（3 秒，两路输出）---
  [管线] H.264 编码器已就绪: 1280x720 @30fps, 2000 kbps, preset=veryfast
  [管线] AAC 编码器已就绪: 48000 Hz, 2 声道, 128 kbps, 每帧 1024 样本, extradata 2 字节
  [管线] 注意: AAC 有 1024 个样本的编码延迟，第一包的 pts 会是负的
  [管线] 音频起点偏移 -1024（AAC 编码延迟），已归零
  [管线] 推流已启动: 1280x720 @30fps, 2/2 路
  启动耗时 412 毫秒（含编码器初始化 + 两路连接）
```

> **第二个要看的点**：这几条 **`[管线]` 前缀的日志，全部是通过 `say`（即 `guardedLog` 包装过的回调）打印的**。它们来自**三条不同的线程**（编码器日志来自 start 主线程、偏移归零来自 write 线程、（可能的）DXGI 日志来自 video 线程），但因为都被 `emit` 的 `logMutex` 保护，**每一行都是完整的，没有字符交错**。**这就是 §7.3 那把锁的价值，同学们可以在这里亲眼验证它。**
>
> **第三个要看的点**：`AAC 有 1024 个样本的编码延迟` 紧接着 `音频起点偏移 -1024 ... 已归零`。这就是第 9 课预言、第 10 课处理的那个坑，今天在全流程里被自动消化掉了。

```
  [ 0.5s] 采集  15 帧，编码  15 包，丢 0 帧，音频  48 块 /  45 包，连接 2/2
  [ 1.0s] 采集  30 帧，编码  30 包，丢 0 帧，音频  96 块 /  93 包，连接 2/2
  [ 1.5s] 采集  45 帧，编码  45 包，丢 1 帧，音频 144 块 / 141 包，连接 2/2
  [ 2.0s] 采集  60 帧，编码  60 包，丢 2 帧，音频 192 块 / 188 包，连接 2/2
  [ 2.5s] 采集  75 帧，编码  75 包，丢 3 帧，音频 240 块 / 235 包，连接 2/2
  [ 3.0s] 采集  91 帧，编码  91 包，丢 4 帧，音频 291 块 / 286 包，连接 2/2
  [通过] 跑到一半时仍处于运行状态
  停止耗时 268 毫秒（要把队列里剩下的包写完）
  [通过] 停止后状态正确
  [通过] 停止过程没有卡死
  实际采集帧率 30.3 fps（目标 30）
```

**这一段是本节最重要的观察对象**，我们把每一列都讲一遍：

| 列 | 值 | 怎么读 | 异常时说明 |
|---|---|---|---|
| 时间 | 0.5 → 3.0 | `elapsedSeconds`，从 `clockStart` 算起 | 跳跃说明主线程 sleep 不准 |
| 采集帧 | 15/30/45/60/75/91 | **每 0.5s 应该稳定增加约 15 帧** | 增长慢 → 编码太慢或屏幕采集失败 |
| 编码包 | 与采集帧几乎相等 | 30fps 且无 B 帧时通常 1 帧 1 包 | 明显少于帧数 → 编码有延迟；明显多于帧数 → x264 输出了 SEI/SPS 分包 |
| **丢帧** | 0~4 | **应该是个位数** | 上百 → 你配了比屏幕刷新率还高的 fps，或者 CPU 不够 |
| 音频块 | 每 0.5s 约 +48 | WASAPI 每块约 10ms → 100 块/秒 | 明显偏少 → 没开声音、设备被占用 |
| 音频包 | 略少于音频块 | AAC 要攒够 1024 样本（21.3ms）才出一包 | 远少于块数 → 重采样出错 |
| 连接 | 2/2 | 两路都在 | 变成 1/2 → 某一路写失败被隔离（第 10 课） |

> **重点观察：音频块 ≈ 291，音频包 ≈ 286，而音频块毎 0.5 秒增加 48 块。** 48 块 × 10ms = 480ms ≈ 0.5s，完美符合预期。**这说明音频时间轴确实跟着真实时间走**（§2.8 的核心结论）。同学们自己跑的时候请务必核对这一列，它是同步是否正常的第一个信号。

```
  读回两个文件：
    A: 视频 91 包（末 3000 ms）/ 音频 283 包（末 2998 ms）
    B: 视频 91 包（末 3000 ms）/ 音频 283 包（末 2998 ms）
  [通过] 两个 flv 都能被正常打开
  [通过] 两个文件都是双流
  [通过] 视频流和音频流都在
  [通过] 两路视频包数完全一致

  时间轴对齐检查（目标 3 秒 = 3000 ms）：
  [通过] 视频时间轴落在 3 秒附近
  [通过] 音频时间轴落在 3 秒附近
    视频与音频的时间差 2 ms
  [通过] 音视频时间差在 500ms 以内（没有累积漂移）
  [通过] 没有负时间戳
```

**恭喜，这一段是全课上下来最有成就感的输出。** 逐看点：

- **`A` 和 `B` 两行数字完全一样** → 第 10 课的"多目标分发"正常工作；
- **视频末 3000ms、音频末 2998ms** → 两条时间轴都准确落地到毫秒基准，**换算全链路（pts → encoder time_base → FLV 1/1000）正确**；
- **时间差 2ms** → 音视频同步极好（远低于 ±500ms 阈值）；
- **没有负时间戳** → `audioOffset` 归零生效。

```
--- 3. 重复启停 ---
  [通过] 连续启动/停止三轮都正常

================================================
  通过 22 项，失败 0 项
================================================
  pipeline_a.flv / pipeline_b.flv 可以直接用播放器打开，
  应该是你的桌面画面加上这段测试期间的系统声音。
================================================
```

### 14.3 用外部工具验收产出文件（必做）

跑完之后，**测试目录里会多出三个文件**：

```
pipeline_a.flv       ← 第 2 段的输出（双流，约 3 秒）
pipeline_b.flv       ← 第 2 段的第二路输出（应逐字节等价的内容）
pipeline_loop.flv    ← 第 3 段三轮启停累积的输出
pipeline_bad.flv     ← 通常不会有（第 1 段在写文件之前就失败了）
```

**实验一：用 ffprobe 看元信息**

```bat
ffprobe -hide_banner pipeline_a.flv
```

应该看到类似：

```
Input #0, flv, from 'pipeline_a.flv':
  Duration: 00:00:03.04, start: 0.000000, bitrate: 2137 kb/s
  Stream #0:0: Video: h264, yuv420p, 1280x720, 30 fps, 30 tbr, 1k tbn
  Stream #0:1: Audio: aac, 48000 Hz, stereo, fltp, 128 kb/s
```

**四个必看字段：**

| 字段 | 期望 | 不对说明什么 |
|---|---|---|
| `Duration` | ≈ 3.0 秒 | 0 或缺失 → **trailer 没写**（第十二章第 4 步） |
| `Video: h264, yuv420p, 1280x720` | 与目标一致 | 尺寸不对 → 缩放上下文有问题（§7.4） |
| `30 fps, 30 tbr` | 与配置一致 | → 检查 pts 换算 |
| 有 Audio 流 | 必须 | 没有 → 音频采集失败（第 8 课） |

**实验二：用 ffplay 播放**

```bat
ffplay -hide_banner pipeline_a.flv
```

你会看到自己桌面这三秒的画面（如果这段时间是静止的桌面，那就看 ffprobe 的数字即可）。

**实验三（推荐）：打开音乐播放器再跑一次**

因为麦克风通常没声音，而 `SystemAudio`（立体声混音）录的是**系统正在播放的声音**。所以：

1. 打开一个音乐播放器开始播歌；
2. 再跑一次测试；
3. 用 ffplay 播放 → **应该能听到歌**。

如果听不到，检查两件事：① 系统音量不是 0；② 有没有把某个应用静音（Windows 的音量合成器里可以单独静音某个 app）。

**实验四：用 VLC 看 +- —— 看图声同步**

在 ffplay 播放时，按 **`W` 键**可以切换显示"音视频同步偏移"（它会打印 `A-V:  0.012` 这样的值，单位是秒）。**这个值应该稳定在 ±0.1 秒以内**，并且**不随时间变大**（漂移的直观检测）。这是验证 §2.8 最直接的手段。

### 14.4 推到真实 RTMP 服务器

本地文件验证通过之后，我们来试真实的推流。

**第一步：起一个本地 RTMP 服务器**

最简单的是用 Docker 跑 nginx-rtmp：

```bat
docker run -d -p 1935:1935 -p 8080:8080 tiangolo/nginx-rtmp
```

或者用 ffmpeg 自带的简单方式（不需要 docker 的方式详见第十七章 Q8）。

**第二步：改一行配置**

在测试的基础上创建一个新实验（建议同学们自己新建一个 `experiments/rtmp_push.cpp`，或者临时改 `test_pipeline.cpp` 第 736~739 行）：

```cpp
StreamTarget targetA;
targetA.platform = "本地nginx";
targetA.url = "rtmp://127.0.0.1:1935/live/test";
config.targets = {targetA};     // 只留一路
```

> 注意 URL 的语法：`rtmp://<host>:<port>/<app>/<stream_key>`。`live` 是应用名，`test` 是流名（密钥）。

**第三步：用 ffplay 拉流验证**

```bat
ffplay -hide_banner rtmp://127.0.0.1:1935/live/test
```

如果你看到自己的桌面画面在动、还能听到系统声音——**恭喜，你已经做出了一个能真正推流的直播推流器**。从第一课到今天的所有零件，在这一刻全部打通了。

**第四步：观察差异**

换成 RTMP 之后，你会明显观察到三个变化，同学们要注意体会：

| 观察项 | 本地文件 | RTMP |
|---|---|---|
| 启动耗时 | ~400ms | 可能 1~3 秒（TCP + RTMP 握手） |
| 停止耗时 | ~270ms | 可能 1~2 秒（要把积压在 ffmpeg 内部缓冲里的包发完） |
| `writeLoop` 的阻塞 | 几乎为 0 | 取决于网速，**这才是 §2.2 讲的"为什么要把 write 单独放在一条线程"的真实现场** |

尤其是最后一条：**本地文件模式下，`writeLoop` 的必要性完全体现不出来**（写文件太快了）。只有接上真实网络，你才会理解"为什么要两个队列"——因为网络和本地延迟的数量级差了 100 倍以上。

### 14.5 五个扩展实验（建议至少做三个）

| # | 实验 | 怎么做 | 预期看到什么 |
|---|---|---|---|
| **1** | **把 `waitNextFrame()` 删掉** | 注释掉第 216~224 行的调用（三处） | 帧率飙到 200+，CPU 跑满，`视频时间轴`断言挂掉（帧数超过 90 很多）。**这是理解节拍器价值最快的实验** |
| **2** | **把 videoQueue 容量改成 1** | 第 124 行改成 `BoundedQueue<AVPacketPtr> videoQueue{1};` | 大量包被队列丢弃（虽然没统计，但文件里能用 ffprobe 看到），**画面出现马赛克/跳帧**，同时**延迟明显降低**——亲身体会"缓冲 vs 延迟"的取舍 |
| **3** | **把 pts 改成逐帧累加** | 把第 257~260 行换成 `static int64_t c = 0; const int64_t pts = c++;` | 丢帧后时间轴被压缩，**drift 断言大概率挂掉**，视频比音频慢。这是 §2.9 讲的"pts 必须来自时间"的反面演示 |
| **4** | **把 `stop()` 里 join 的顺序倒过来** | 把第 446~451 行的顺序改成 write→video→audio | 文件内容变短（少了最后几个包），`视频时间轴`可能掉到 2900ms 以下。这是 §12.4 顺序陷阱的实证 |
| **5** | **去掉 `timeBeginPeriod(1)`** | 注释掉第 401 行 | 帧间隔变得不均匀（15.6ms 粒度锯齿），`丢帧`数可能上升。可以用 ffmpeg 的 `ffprobe -show_frames` 看 pkt_pts_time 的间隔是否均匀 |

> **做实验的正确姿势**：每次只改一处，跑完把结果记下来，然后**还原**。记住第一条 DCF 原则：**先有测试保底，再动代码。** 你现在有 `test_pipeline.exe` 保底了，这就是第 3 课说的"117 组测试的仪式感"。

---

## 第十五章 排错讨论：四种典型故障

这一章我们把这门课上可能出现的四类典型问题系统地梳理一遍。**排错的思路比排错的答案重要**，所以每个问题我都按"现象 → 定位 → 原因 → 修法"四步来写。

### 15.1 故障类型一：卡死（hang / deadlock）

**现象**：程序不崩溃，也不报错，就是不动了。或者 `stop()` 迟迟不返回，最后测试 .exe 卡在那里。

**定位方法（三条，按优先级）：**

1. **先看是哪一条线程卡住了。** 用 Visual Studio → 调试 → 全部中断（Ctrl+Alt+Break）→ 线程窗口，看三条线程各自停在哪一行。
2. **如果 `stop()` 卡住，用我们打的时间戳判断**：ائي التقر——如果测试输出停留在"停止耗时"之前，说明卡在某个 `join()` 上。
3. **按 join 的先后顺序判断**：卡在第几个 join能直接告诉我们是谁没退出。

**常见原因与答案：**

| 原因 | 怎么确认 | 怎么修 |
|---|---|---|
| **`writeLoop` 阻塞在网络写里** | 线程停在 `av_interleaved_write_frame` | 第 10 课讲过 `StreamOutput` 里设了 `rw_timeout`（IO 超时），检查它有没有被设为 0（无限等待）。**RTMP 场景必须设超时，否则网线一拔就永远卡住** |
| **`videoLoop` 在 `camera.nextFrame(1000)` 里** | 摄像头拔了/被别的程序占用 | 最多 1 秒后返回，属于正常。如果想更快，把 1000 改小（但会增加空转） |
| **`audioLoop` 在 `nextChunk(100)` 里** | WASAPI 设备状态异常 | 同上，最多 100ms |
| **`BoundedQueue::push` 以为会阻塞**（其实不会） | — | 复习 §2.3：**push 永不阻塞，满了丢旧的**。如果有人改成"BLOCKING"版本，这里就会死锁 |
| **`stop()` 被自己线程调用** | 比如在 log 回调里调了 stop | **永远不要在回调里调 stop！** 会自己 join 自己 → 死锁。要用异步的方式排队执行 |
| **忘记 join 顺序（先消费者）** | 见 §12.4 | 修正顺序 |

> **一个真实的隐藏杀手：`std::thread` 析构时还 joinable → `std::terminate()`。** 这不算"卡死"而是崩溃，但症状类似（程序突然没了）。**检查所有 `std::thread` 成员的路径能不能保证 join。**

### 15.2 故障类型二：音画不同步（A/V sync drift）

**现象**：播放出来的视频嘴型对不上声音；或者刚开始同步，越到后面差得越多（**漂移**）。

**定位**：先量化！

```bat
ffplay pipeline_a.flv        # 播放时按 W 键看 A-V 偏移
ffprobe -show_packets pipeline_a.flv -of csv | findstr ",video,"   # 看视频包时间戳
ffprobe -show_packets pipeline_a.flv -of csv | findstr ",audio,"   # 看音频包时间戳
```

关键判断：**差值是不是随时间线性增长？**

- **是（前 10 秒差 50ms，60 秒差 300ms）→ 是 drift（速率不同步）**
- **不是（一直差 200ms）→ 是 constant offset（固定偏移），不是 rate 问题**

**原因与答案：**

| 现象 | 根因 | 修法 |
|---|---|---|
| **固定偏移** | 视频起点记晚了（`firstFrameAt` 在第一帧采到之后才记，而 audio 更早开始） | 在 `stop()` 之前统一一个零点：比如让 audio 的第一块时间戳也对齐到"视频首帧时刻"。**本项目接受这个 ~100ms 的偏移** |
| **drift（音频越走越快）** | 音频 pts 按"已处理样本数"累加，而某些样本被丢了导致计数落后（**注意：如果丢的是样本而 pts 不补，会导致音频变慢；如果重采样产生了额外样本而 pts 没算，会导致音频变快**） | 让音频 pts 也由真实时间推导，而不是由样本数累加 |
| **drift（视频越走越慢）** | pts 用"帧计数"而不是"真实时间"（§2.9 的反面教材） | 改回按 `elapsedUs` 算 |
| **视频周期性落后，然后跳帧** | CPU 不够，编码超时，触发 pts 去重丢帧，但因为 pts 按真实时间所以不累积——表现为"偶发跳一下" | 降低分辨率/码率，或换更快的 preset（如 `ultrafast`） |
| **负时间戳导致播放端混乱** | AAC `initial_padding` 没被归零 | 检查 `StreamOutput` 的 `audioOffset` 逻辑（第 10 课） |

> **给同学们的一个量化工具**：把测试里的 3 秒改成 30 秒跑一次，看 `drift` 是不是从 2ms 涨到 20ms。**如果成正比，说明有 rate mismatch（速率不匹配），必须修；如果基本不变，说明只是固定偏移，可以接受。**

### 15.3 故障类型三：CPU 飙高

**现象**：一个核或者多个核跑满，风扇狂转。

**定位**：

1. **任务管理器 → 详细信息 → 右键进程 → 分析等待链**，或者
2. Visual Studio → 调试 → 性能探查器 → CPU 使用率，看热点在哪个函数。

**原因与答案：**

| 热点出现的位置 | 原因 | 怎么修 |
|---|---|---|
| `writeLoop` 里 `sleep_for(2ms)` 附近 | 每轮 2ms→ 每秒 500 次轮询，加上两次 `tryPop(1)`（各最多 1ms） | 空闲时把等待加长到 10ms，或者用条件变量代替轮询（第 3 课讨论过） |
| `videoLoop` 的空 `continue` 分支 | **某个 continue 之前忘了 `waitNextFrame()` / `sleep`** | 逐个检查 §8.6 提到的三个 continue |
| `sws_scale` | 缩放本身很贵（尤其是 1080p→1080p 还要 bilinear） | ① 确认走的不是这条路径（尺寸相等时有快路径，见 §7.4）；② 用 `SWS_POINT` 或 `SWS_FAST_BILINEAR`；③ **直接让采集端输出目标尺寸**（摄像头支持的话） |
| `sws_getContext` 反复出现 | 源尺寸一直在变导致反复重建 | 检查 `scaleSrcWidth/Height` 的缓存逻辑 |
| `libx264` 的 encode | 正常现象，编码就是要算力 | 换 `preset=ultrafast`、降分辨率、或者用硬件编码（NVENC/QSV，本项目未实现） |
| **采集循环 counting thousands of fps** | 节拍器失效（摄像头路径忘了 return） | §8.2 的 `if (!isScreen) return;` |

> **一个常被忽略的点：`std::make_shared<VideoFrame>` + `resize` 每帧分配 3.5MB（720p）**。如果 CPU 花在内存管理上，说明该做**帧池**（第十七章 Q3）。用性能探查器看 `operator new` 的占比就能确认。

### 15.4 故障类型四：停止后不退出 / 资源没释放

**现象**：主窗口关了但进程还在（VS 里还在跑）；第二次启动失败；摄像头指示灯还亮着。

**原因与答案：**

| 原因 | 怎么确认 | 怎么修 |
|---|---|---|
| **析构函数没调用 `stop()`** | 第 334~337 行被删了 | 加回去（§12.5） |
| **`stop()` 的分支 A 判断错误** | 第 429~430 行的条件不成立 | 用调试器看 `running` 和三个 joinable 的值 |
| **`timeBeginPeriod` 没配对** | 第二次启动时 `timeBeginPeriod` 再调一次 | 确保 `lowerTimer()` 在所有出口都被调用（§12.2） |
| **摄像头/音频没 close** | 分支 A 或 B 的 close 列表被漏了 | 逐个核对 §12.3 / §12.4 |
| **`AudioCapturer` 内部的 WASAPI worker 线程没停** | `audio.close()` 没被调用，或 close 内部没 stopping | 第 8 课的内容，检查 `audioSource __remove`——检查 `close()` 有没有设 stopping 并 join 内部线程 |
| **队列里的包没被释放** | 队列的 clear() 被跳过 | §12.3 第 4 步 |
| **`std::thread` 对象 join 过但没重置** | 通常无害 | 可选：`videoThread = std::thread();` |

> **推荐的检查手段**：
> ```bat
> REM 1. 用 VLD（Visual Leak Detector）看有没有内存泄漏
> REM 2. 简单版：连续跑 5 次 start/stop，看内存占用是否单调上升（任务管理器）
> ```
> 用测试第 3 段改造一个"跑 20 轮"的版本，如果每轮内存 +20MB，那一定有东西泄漏。

### 15.5 排错检查清单（贴在你的显示器边框上）

```
① 三条线程是否都在"睡着"或"阻塞"？
   → 不是 → 空转，补 sleep/waitNextFrame

② 每个共享资源是否只有一个所有者线程？
   → 不是 → 数据竞争，加锁或重新划分

③ 每个 continue / break 之前是否保持了节奏（有阻塞）？
   → 不是 → 忙等或漏睡

④ join 的顺序是否是"先生产者、后消费者"？
   → 不是 → 最后几个包丢失

⑤ start/stop 是否严格对称？
   → 不是 → 资源泄漏/重复释放

⑥ 每一处阻塞是否有超时？
   → 没有 → 停止不响应

⑦ pts 是否来自时间而不是计数器？
   → 来自计数器 → 必然漂移

⑧ 是否让 UI 线程去 join 从而自锁？（应改为异步通知）
   → 是 → 改为异步通知
```

---

## 第十六章 小结

同学们，我们这节课从"零件"走到了"整机"，最后再收一次网。

### 16.1 本课的十一条硬性知识点

1. **线程边界按"独占资源 + 阻塞点"划分，不按功能数量划分。** 本课的三个线程对应三组独占资源（采集+视频编码 / 采集+音频编码 / 输出），编码被"搭便车"放进采集线程以换取低延迟。

2. **每一个进程内的 golden rule： Jedes Objekt.**——**每个对象只能有一个所有者线程。** 本课所有七个资源对象都满足了这一点，所以全程只需要一把锁（`logMutex`）。

3. **停止多线程的唯一方式是"协作式"**：`atomic<bool> running` 通知 → 线程各自退出循环 → `join()` 汇合。**永远不要杀线程。**

4. **每个阻塞调用都必须有超时**（1000ms/100ms/1ms/2ms），这是"停止响应速度"和"CPU 占用"之间的可调阀门。

5. **`join` 的顺序必须是"先生产者、后消费者"**，否则生产者 flush 出来的最后几个包会无人接管。

6. **起线程的黄金法则**：`running = true` 和其他所有共享状态**必须在 `std::thread` 构造之前**完成初始化（标准保证 happens-before）。

7. **pts 必须来自真实时间（`steady_clock`），不能来自帧计数。** 这样丢帧之后时间轴仍然连续，画面会"跳"但不会"慢"。

8. **`steady_clock` 而不是 `system_clock`**（时钟回拨会让 pts 倒退）；**整数运算带上 `+ b/2` 实现四舍五入**（避免浮点不确定性）。

9. **`sleep_until`（绝对时刻）而不是 `sleep_for`（相对时长）**，并且在落后时把基准拉回"现在"（承认失败，不追赶）。

10. **队列容量是设计决策**：视频给 0.8 秒（live 优先于延迟），音频给 5.5 秒（人耳忍受不了丢帧）。队列满是"丢旧包"，是优雅降级。

11. **优雅启停四步：通知 → join → 排空 → 收尾。** 加上 `flush()`、`close()`、`timeEndPeriod` 三件配套。

### 16.2 本课的六条代码级技巧

| 技巧 | 出现位置 | 价值 |
|---|---|---|
| **出口清理 lambda**（`lowerTimer`） | 第 422~427 行 | 多 return 出口时保证清理不遗漏（穷人版 `defer`） |
| **回调工厂**（`guardedLog`） | 第 152~155 行 | 给下层模块包一层线程安全外壳，不改它们的源码 |
| **失败处理收敛**（`fail` lambda） | 第 348~353 行 | DRY，将来加"记录日志"只改一处 |
| **缓存 + 失效检测**（`scaleContext`） | 第 177~189 行 | 昂贵资源的复用模式（适用 DB 连接、SSL ctx…） |
| **工作检测退避**（`didWork`） | 第 311~326 行 | 忙时全速、闲时才轮询 |
| **哨兵初值**（`lastPts = -1`） | 第 228 行 | 保证第一帧不被误判为重复帧 |

### 16.3 API 速查表

| 我要... | 怎么写 |
|---|---|
| 起一条线程 | `std::thread t([this]{ impl_->xxxLoop(); });` |
| 安全地停止并回收 | `if (t.joinable()) t.join();` |
| 让别的所有线程看见的 bool | `std::atomic<bool> running{false};` |
| 取当前时刻 | `auto now = Clock::now();` （Clock = `steady_clock`） |
| 算毫秒差 | `duration_cast<milliseconds>(now - from).count()` |
| 睡到某一时刻 | `std::this_thread::sleep_until(target);` |
| 睡一段时间 | `std::this_thread::sleep_for(milliseconds(2));` |
| 四舍五入的整数除法 | `(a * b + b/2) / b` |
| 从 optional 取值 | `if (auto p = q.tryPop(1)) { use(*p); }` |
| 把 unique_ptr 移交 | `q.push(std::move(packet));` |
| scrub(清理)队列 | `q.clear();` （元素析构自动释放） |

### 16.4 本课的已知局限（诚实清单）

同学们，**做一个诚实的工程师比做一个看起来完美的工程师重要**。这一版本的代码有下面这些明确的不足，第十七章的思考题就是围绕它们展开的：

| # | 局限 | 位置 | 后果 |
|---|---|---|---|
| 1 | **队列满丢包没有统计** | `BoundedQueue` + 第 278 行 | 花屏了不知道是丢了多少包 |
| 2 | **`stats()` 与多个线程访问 `output` 存在竞争** | 第 486~488 行 | 理论上可能读到撕裂状态 |
| 3 | **每帧 `make_shared` + `resize` 分配 3.5MB** | 第 191~195 行 | 内存分配压力大，有优化空间 |
| 4 | **`log` / `lastError` / `timerRaised` 非线程安全访问** | 多处 | 跨线程 start/stop/lastError 有竞争 |
| 5 | **线程函数没有 try/catch** | 第 407~409 行 | 抛异常会整个进程 terminate |
| 6 | **计数器不重置** | §12.10 | 重复 start 统计累加 |
| 7 | **`videoLoop` 里有一处直接用 `log()` 而不是 `emit()`** | 第 242~243 行 | 绕过日志锁 |
| 8 | **不支持运行时改配置**（动态码率/切换源） | 第 210 行 `const` | 必须 stop 再 start |
| 9 | **write 线程每轮只处理 1 音频 + 1 视频包** | 第 313~320 行 | 视频突发时可能积压 |
| 10 | **没有 pacing（按真实时间发送）** | `writeLoop` | 本地文件/低延迟网络下会瞬间发完，RTMP 上可能造成突发 |

> 这些不是"错误"，是**"现在不做"的决策**。同学们要能分辨"bug"和"已知局限"：**bug 是不知道的行为，局限是知道并接受的代价。** 写代码时把它们列出来，比假装完美有价值得多。

### 16.5 下节课预告

到这一课为止，**我们的推流器已经"能用"了**：它能采、能编、能封装、能分发、能启停、能统计。

但它还有一个很大的身份问题：**它是一个命令行程序，不是给用户用的软件。** 用户需要：

- 一个窗口让他填 RTMP 地址、选分辨率和码率；
- 一个"开始/停止"按钮和一个预览画面；
- 一个日志窗口看到实时状态；
- 掉线自动重连、配置能保存。

**从下一课开始，我们进入 Qt6 的界面层**：把 `StreamPipeline` 接到一个真正的 Windows 窗口程序上，`stats()` 驱动界面上的计时器和帧率显示，`emit` 出来的日志进滚动文本框。**这时候你们会真切感受到今天这 45 行头文件的价值**——因为 UI 层只需要认识它，就能驱动整台机器。

> **老师最后的话**：这一刻请同学们回头看看。第 1 课我们配环境，第 2 课造假数据跑通 RTMP，然后我们用 8 节课做零件，今天终于把它们装成了一台机器。**软件工程最有成就感的瞬间，往往不是某个算法的灵光一现，而是"第一次真正跑起来的那一刻"。** 而这台机器之所以能跑起来，靠的不是某个天才设计，而是每一次都把 Contract（合同）、所有权、成对调用、幂等这些"笨功夫"做对了。**笨功夫，才是真功夫。**

---

## 第十七章 思考题

下面 8 道题，覆盖本次 commit 的设计、实现、测试和不足。建议同学们**先自己写一遍思路再往下看要点**——对着参考答案"看懂"和自己做出来，是两种完全不同的收获。

**Q1（重点题 · 顺序与死锁）**
如果把 `stop()` 里三个 `join()` 的顺序改成 `writeThread → videoThread → audioThread`，会发生什么？请具体说明：
(a) 丢失的是哪些数据？
(b) 为什么不会死锁？
(c) 有没有办法让它"反过来也不丢数据"？（提示：想想 write 循环的退出条件需要什么额外信息）

**Q2（实现题 · 丢帧策略）**
`videoQueue` 满了会静默丢旧包（第 3 课的语义）。请：
(a) 估算 1920×1080@30fps、3000kbps 下，队列从空到满需要多久（假设网络突然断掉）；
(b) 给 `BoundedQueue` 加一个"丢弃计数"，并在 `PipelineStats` 里加字段暴露出来；
(c) 讨论：丢旧包和丢新包（if full then don't push），哪个更适合直播？哪个更适合视频会议？

**Q3（性能题 · 内存分配）**
每帧都 `std::make_shared<VideoFrame>()` + `bgra.resize(3.5MB)`：
(a) 用某种方式（任务管理器/性能探查器/计数器）量化一下，30fps 下每秒的分配量是多少；
(b) 设计一个"帧池"（frame pool）：循环复用 3~5 个 VideoFrame 对象。要注意什么？（提示：pts 字段、`shared_ptr` 的释放时机、以及"什么时候可以回收"）

**Q4（计算题 · 音视频漂移）**
假设你的机器上视频 pts 每秒实际推进 999000 微秒（因为某些帧编码超时被"去重"丢了），而音频按真实时间推进：
(a) 推 10 分钟之后，音视频会差多少毫秒？
(b) 这个误差会触发本课的哪一个断言？
(c) 如果想让视频时间轴"自我校正"（每隔一段时间拉回真实时间），该怎么做？会引入什么问题？

**Q5（线程安全题 · 已知局限）**
本课程指出多处"理论上有竞争"的地方：`log`、`lastError`、`timerRaised`、以及 `stats()` 里读 `output.targets()`。请：
(a) 挑一处，用具体场景说明它会在什么时序下出错；
(b) 给出你的修法（加锁 / 改 atomic / 改 RAII 包装三选一），并说明为什么不选另外两种；
(c) 把 `timerRaised` 改成 RAII 小类（构造 `timeBeginPeriod`、析构 `timeEndPeriod`），写出代码。

**Q6（测试题 · 提升覆盖）**
(a) 在第 2 段测试里加一条断言：检查日志列表 `logs` 中是否出现了"推流已停止"；为什么这条断言比 `check(!pipeline.isRunning())` 更能验证"优雅停止"？
(b) `start()` 里没有重置五个计数器。请修改代码让第二轮 `stats()` 从 0 开始，并给你的修改加一条断言（想想该加在哪一行之后）。

**Q7（重构题 · 三处改动）**
请在不改变外部行为的前提下做下面三处重构，并重跑 `test_pipeline.exe` 确认 22 条断言仍然全绿：
1. 第 242~243 行的 `log(...)` 改成 `emit(...)`；
2. `pipeline/CMakeLists.txt` 里 `winmm` 从 `PUBLIC` 改成 `PRIVATE`（试试看会不会链接失败，为什么）；
3. 给 `StreamOutput::open` 加一道防线：如果 `videoParams` 或 `audioParams` 是 `nullptr`，返回明确的错误信息，而不是崩溃。

**Q8（综合题 · 健壮性）**
把三条线程的创建改成带异常保护的形式：

```cpp
impl_->videoThread = std::thread([this] {
    try { impl_->videoLoop(); }
    catch (const std::exception& e) { impl_->emit("视频线程异常: " + std::string(e.what())); }
    catch (...) { impl_->emit("视频线程发生未知异常"); }
});
```

然后回答：
(a) 为什么 C++ 的异常不能跨线程传播？丢掉了会怎样？
(b) 加了 catch 之后，这条线程"死"了，但另外两条还在跑。**这时候 `running` 还是 true，`stats()` 还正常，用户以为在推流。** 这个"部分故障"该怎么处理？至少说出两种方案（比如：主动拉停整条管线 / 上报 UI 让用户决定），并说明各自的取舍。

---

### 参考答案要点（先自己做完再看）

**Q1**
(a) 丢失的是 `videoLoop` 和 `audioLoop` 在 `while` 循环退出后 `flush()` 出来的那几个包（视频通常 1~2 个包含最后几帧，音频至少 1 个包）。因为它们是在 `join()` 之前才被推进队列的，如果 write 线程已经退出，就再也没人消费了。
(b) 不会死锁：`writeLoop` 的 `tryPop` 有 1ms 超时，它能自主退出；两个生产者最终也会因 `running=false` 退出；join 只是等待。
(c) 可以：让生产者退出时"不会丢 cv reload"之外的方法——更标准的做法是**给队列加 `close()` 语义**：生产者调 `q.close()` 表示"我不再生产了"，消费者收到 close 信号后**必须等到队列真的空且明确被告知结束**才退出。第 3 课 §8.2 提到的正是这个缺失。另一种做法是给 `writeLoop` 加一个"生产者已全部结束"的标志（两个 atomic<bool>），退出条件改成"两个 producerDone && 两个队列都空"。

**Q2**
(a) 3000 kbps = 375 KB/s = 30 fps 下每帧约 12.5 KB。24 个包 ≈ 300 KB → **约 0.8 秒**就填满（这也是 §2.3 表格里的数字来源）。
(b) 给 `BoundedQueue` 加 `std::atomic<size_t> dropped_` 或用 mutex 保护；`push` 里 `pop_front` 时自增；加 `size_t droppedCount() const`；`PipelineStats` 加 `videoPacketsDropped`；`stats()` 里填上。
(c) 直播选**丢旧**（观众要看最新画面，宁可跳一下也不要延迟一分钟）；视频会议选**丢新**（实时对话中，新内容更重要，且人人对延迟极敏感，宁可画面停一下也不要跳帧打断语义）。

**Q3**
(a) 720p@30fps：3.5 MB × 30 = **105 MB/s** 的分配+释放（还没算多目标时的 clone）。1080p 是 237 MB/s。
(b) 设计要点：① 池里预分配 N 个 `VideoFrame`（按需 resize 一次，之后只改内容）；② 用第二个队列 `BoundedQueue<VideoFramePtr> freePool` 或者简单的 mutex+vector 管理；③ **回收时机**是关键——必须等消费者用完，靠 `shared_ptr` 自定义删除器（`std::shared_ptr<VideoFrame>(raw, [pool](auto p){ pool->release(p); })`）自动归还最优雅；④ 注意 pts 字段要在取出时重置，否则复用脏数据；⑤ 池大小建议 ≥ 编码延迟帧数 + 队列容量，避免池空而被迫新建（那就失去意义了）。

**Q4**
(a) 每秒差 1000 微秒 = 1ms → 600 秒 = **600ms**（而且符号取决于谁快）。这就是典型的"累积漂移"。
(b) 会触发 `check(drift > -500 && drift < 500, ...)`——**只要测试跑得够久**（3 秒的测试只能看到 3ms，完全看不出来）。**这告诉我们：短时测试抓不到漂移，要有专门的长时间回归用例。**
(c) 校正方式：每隔 N 帧（比如每 5 秒）用真实时钟修正 `firstFrameAt`（把它的偏移量调整 1ms），实现"缓慢拉回"。**代价**：修正瞬间 pts 会出现一次不连续（比上一帧多跳或少跳 1~2 个单位），画面可能有微小抖动；如果修正过猛会引入新的可见抖动。所以校正要**小步慢校**（每次校正量小于半帧），这就是所谓的 **PLL/时钟恢复**思路。

**Q5**
(a) 举例：`stats()` 读 `output.targets()`（UI 线程）的同时，write 线程正在 `writePacket` 里写 `target.lastError`（写失败时）。前者可能在读 `std::string lastError` 的过程中读到"一半被改写"的状态（string 的内部指针/长度分离写入）→ 崩溃或乱码。
(b) 选**加锁**（给 `StreamOutput` 内部加 mutex 并让 `writePacket`/`targets()`/`reconnectDisconnected()` 共用）：因为要保护的是"一组相关状态"而不是单个变量，atomic 做不到（atomic 只保护单个对象）；RAII 包装也不能解决"多个变量的组合一致性"。
(c) RAII 类：
```cpp
class TimerResolution {
public:
    explicit TimerResolution(UINT period) : period_(period) {
        ok_ = (timeBeginPeriod(period_) == TIMERR_NOERROR);
    }
    ~TimerResolution() { if (ok_) timeEndPeriod(period_); }
    TimerResolution(const TimerResolution&) = delete;
    TimerResolution& operator=(const TimerResolution&) = delete;
private:
    UINT period_; bool ok_ = false;
};
```
这样 `Impl` 里用 `std::optional<TimerResolution> timer;` 或 `std::unique_ptr<TimerResolution>`，**任何退出路径都会自动注销**。

**Q6**
(a) 因为 `isRunning()` 只看一个 atomic 标志，而"推流已停止"这条日志是在 `join` 完成、资源清理完毕之后才 emit 的（第 469 行），**它证明的是"整条停止流程走完了"**，而不只是"标志位改了"。这是"行为断言"强于"状态断言"的例子。
(b) 在 `start()` 的步骤 2（第 343~345 行附近）加：
```cpp
impl_->videoFrames = 0; impl_->videoPackets = 0; impl_->videoDropped = 0;
impl_->audioChunks = 0; impl_->audioPackets = 0;
```
注意要**写成 `.store(0)`**（因为是 atomic）。然后在测试第 3 段每轮 stop 之后加一句：
```cpp
if (pipeline.start(config, nullptr)) {
    // 第二轮起跑之后立刻取一次，几个计数器应该都很小
}
```
更简洁的做法：第 2 段末尾 `stop()` 之后断言 `pipeline.stats().videoFramesCaptured == 0`（如果 stop 里重置的话）——**两种设计（start 重置 vs stop 重置）都可以，但要在文档里写清楚。**

**Q7**
1. 直接替换即可，`emit` 签名一致（都是 `void(const std::string&)`）。这是 §8.4 指出的瑕疵修复。
2. 改成 `PRIVATE` **不会**失败：因为 `test_pipeline` 自己不用 `timeBeginPeriod`，而静态库的链接传播只需要 winmm 出现在最终链路上（PUBLIC/PRIVATE 对 STATIC 库的链接传播影响有限，但对 include 目录影响很大；真正的差异体现在 SHARED 库和 include 目录上）。**做完要确认 exe 仍能被链接出来。** 记住：判断该用 PUBLIC 还是 PRIVATE，标准是"**使用者是否需要这份东西才能编译/链接**"。
3. 在 `StreamOutput::Impl::open` 开头加：
```cpp
if (!videoParams || !audioParams)
    return fail("缺少编解码器参数（编码器可能没有正确打开）");
```
这样即使有人把 §11.5 的 open 顺序写反了，也会得到一条清晰的错误而不是崩溃或写出坏文件。**这叫"防御式 API"。**

**Q8**
(a) C++ 异常是"栈展开"（stack unwinding）机制驱动的：抛异常时会沿着调用栈向上寻找 catch。**线程有自己独立的调用栈**，它的"上游"是操作系统而不是创建它的那条线程，所以异常无处可去。标准规定这种情况下调用 **`std::terminate()`** → 进程直接终止（不会有堆栈，不会有日志）。所以"某一帧 malloc 失败"这种小事可能导致整个软件消失。
(b) 方案一**主动拉停**：catch 里调 `running = false` 并 emit 错误，让另外两条线程也优雅收尾。优点：状态一致（用户看到"已停止"+ 错误原因），缺点：用户可能正在直播，突然中断体验差（可以配合"通知 + 询问"）。
   方案二**上报 UI 让上层决定**：catch 里只 emit + 置一个 `videoFailed` 标志，UI 轮询时发现就弹窗提示"视频采集异常，[重试]/[结束推流]"。优点：把决策权交给用户/上层策略（这也是 §16.5 说的"上层知道全局"），缺点：中间有一段"残废推流"的时间（只有声音没有画面）。
   **推荐做法**：两者结合——立刻上报 + 给自己一个宽限期（比如 3 秒内自动重试一次采集器），失败再拉停。**实时系统的容错通常都有"快速自愈 → 慢速降级 → 最终上报"三层。**

---

**本讲配套文件**：`src/core/pipeline/StreamPipeline.h`（45 行）、`src/core/pipeline/StreamPipeline.cpp`（443 行）、`src/core/pipeline/CMakeLists.txt`（20 行）、`tests/test_pipeline.cpp`（298 行）；另有顶层 `CMakeLists.txt`（+1 行）、`tests/CMakeLists.txt`（+8 行）、`src/core/common/Common.h`（+2 行）

**前置讲稿**：第 3 课（契约层与 `BoundedQueue`）、第 4/5/6 课（采集三件套）、第 8 课（WASAPI 音频采集与 `padSilence`）、第 9 课（两个编码器与 `AVPacketPtr`/`timeBase()`/`codecParameters()`）、第 10 课（`StreamOutput` 封装与多目标分发）

**后续讲稿预告**：Qt6 界面层——把 `StreamPipeline` 接进一个有窗口、有按钮、有预览框的真正软件里

---

## 附：本讲"五章对照索引"

> **用法说明**：这张表把本讲的全部内容压缩成**五个维度**的对照，方便你复习、检索、以及跟前几课建立联系。横向五行是本讲的**五个部分**，纵向五列是**五个维度**。

| 本讲部分 | 覆盖章节 | 核心知识点 | diff 中的文件与行号 | 怎么验证 / 上哪去练 |
|---|---|---|---|---|
| **① 预习：概念与原理** | 第 1~2 章 | 流水线与吞吐瓶颈定律；**线程边界按"独占资源+阻塞点"划分**（为什么没有独立编码线程）；生产者-消费者落地与两个队列容量（24/0.8s、256/5.5s）；`std::thread`/`joinable`/`join`；`atomic<bool>` 停止标志 vs `volatile`；`steady_clock` 单调时钟；pts/dts/time_base 三套单位换算；音视频同步（两条独立时间轴 + 同规则）；两级丢帧策略；`sleep_until` 无漂移节拍器；`timeBeginPeriod`；PImpl 编译防火墙；`sws_scale` 与 stride；优雅启停四步 | 概念章，对应实现见 `StreamPipeline.cpp` 第 124~125 行（两个队列）、第 92 行（`Clock` 别名）、第 216~224 行（节拍器）、第 257~260 行（pts 公式）、第 401~402 行（`timeBeginPeriod`）、第 169~206 行（`scaleToTarget`） | **实验 1**（删掉 `waitNextFrame` 看帧率飙飞）、**实验 2**（队列改 1）、**实验 3**（pts 改成累计计数）、**实验 4**（倒换 join 顺序）、**实验 5**（去掉 `timeBeginPeriod`） |
| **② 实现：模块本体** | 第 3~5、7、11、12 章 | CMake 三处接线（`add_subdirectory` / `PUBLIC`-`PRIVATE` 传播 / `winmm` 平台条件）；`audioDeviceIndex = -1` 哨兵值与向后兼容；`PipelineStats` 八字段快照仪表盘；PImpl + 不可拷贝 + 私有 Impl 指针；Impl 十组成员清单（每个对象唯一所有者线程）；`emit`/`guardedLog` 回调工厂与日志锁；`scaleToTarget` 缓存+失效检测；`start()` 十三步（含"\*\*共享状态必须先于线程创建\*\*"与 fail lambda 回滚）；`stop()` 两分支与 join 顺序 | `src/core/pipeline/StreamPipeline.h`（45 行，全）、`src/core/pipeline/StreamPipeline.cpp`（443 行，全：Impl 第 39~167 行、三个循环第 208~328 行、`start` 第 339~418 行、`stop` 第 420~470 行）、`src/core/pipeline/CMakeLists.txt`（20 行，全）、顶层 `CMakeLists.txt` 第 +36 行、`tests/CMakeLists.txt` 第 +50~57 行、`src/core/common/Common.h` 第 +31~32 行 | 编译 `core_pipeline` 静态库；对着 §16.3 API 速查表默写三函数；把第十七章 Q5(c) 的 RAII Timer 类加进去并重跑测试 |
| **③ 验证：端到端测试** | 第 13 章 | `check` 断言框架与 `std::string`/`const char*` 双重载；`FlvInfo` + `probeFlv` 回读（`avformat_open_input`→`find_stream_info`→`av_read_frame`→`unref`/`free` 配对）；`AV_NOPTS_VALUE` 判空；FLV 毫秒时间基让 pts 可直接当毫秒比；`setvbuf` 行缓冲；`chcp 65001`；四组验收（参数校验 / 3 秒双路 / 文件回读与时间轴 / 重复启停三轮） | `tests/test_pipeline.cpp`（298 行，全）：第 27~41 行 `check`、第 35~63 行 `FlvInfo`、第 65~112 行 `probeFlv`、第 216~245 行主循环打印、第 245~267 行停止与帧率断言、第 271~297 行文件回读断言、第 300~332 行重复启停 | `build.bat` → `test_pipeline.exe`；看 22 条断言全绿；**用 `ffprobe` 看 Duration/流信息、用 `ffplay` 按 W 键看 A-V 偏移**；对比 A/B 两个文件的包数完全一致 |
| **④ 实践：动手与排错** | 第 14~15 章 | 编译（含三条必修报错）、运行输出逐列解读（帧率/丢帧/音频块/连接数四列的含义）、五个扩展实验、本地 RTMP 实验（`rtmp://host:port/app/key`）、四类故障的"现象→定位→原因→修法"（卡死 / 音画漂移 / CPU 飙高 / 停止不退出）、八条排错检查清单 | 不新增代码，全部是对 ②③ 部分代码的**修改与观察**；第 14.5 节的实验 1~5 分别指向 `StreamPipeline.cpp` 第 216~224、124、257~260、446~451、401 行 | 按 §14.2~14.5 逐个做；换 RTMP 目标看启动/停止耗时的变化（验证 §14.4 的三条差异）；出问题时按 §15.5 清单逐条排查 |
| **⑤ 沉淀：总结与延伸** | 第 16~17 章 | 11 条硬性知识点 + 6 条代码技巧 + API 速查表 + **10 条已知局限**（丢包无统计、`stats()` 竞争、每帧 3.5MB 分配、非线程安全字段、线程无 try/catch、计数器不重置、`log` 绕过锁、不支持热更新、write 每轮仅各 1 包、无 pacing）；8 道思考题（含 Q1 join 顺序、Q4 漂移量化 600ms/10 分钟、Q8 异常捕获与"部分故障"处理） | 对全讲 880 行 diff 的回顾与批判；十处局限分别位于 `StreamPipeline.cpp` 第 278、486~488、191~195、343~345、407~409、242~243、210、313~320 行等；三处重构练习见 Q7 | 做完 8 道思考题（尤其 Q1、Q4、Q8 综合题）；预习下一阶段 Qt6 界面层——`StreamPipeline` 的 45 行头文件就是 UI 层唯一的接口 |

### 附：本讲与前面几课的衔接点（复习时对照）

| 本讲用到的东西 | 在哪一课学的 | 本讲怎么用的 |
|---|---|---|
| `StreamConfig` / `VideoFrame` / `AudioChunk` / `LogCallback` | **第 3 课**（`a4453bb1`） | §5、§7.3：全部作为 Impl 成员；`LogCallback` 被 `guardedLog()` 包一层后分发给所有子模块 |
| **`BoundedQueue`（`push` 永不阻塞、`tryPop` 超时、`std::optional`）** | **第 3 课** | §2.3、§7.3 组 7、§10.2：**首次实战**，`videoQueue{24}` / `audioQueue{256}`；正因为当初 `push` 写成"按值+move"，今天才能存 `unique_ptr` |
| `ScreenCapturer`（DXGI 超时 0 + 复用 lastFrame） | **第 4 课**（`81eaa1f5`） | §8.4：屏幕路径不阻塞 → 必须由节拍器打拍子 |
| `CameraCapturer` + `CameraFrameConverter` | **第 5、6 课**（`281a22be` / `e92cbd3`） | §8.4：摄像头路径自带节奏 → `waitNextFrame` 里 `if (!isScreen) return;`；转换失败要靠 `lastError()` |
| `AudioCapturer`（`nextChunk(100)`、`padSilence` 补静音） | **第 8 课**（`1ce7f62`） | §9.1、§9.2：`padSilence` 是"音频时间轴==真实时间"这条假设成立的前提 |
| `VideoEncoder` / `AudioEncoder`、`AVPacketPtr`、`timeBase()`、`codecParameters()`、`flush()` | **第 9 课**（`66e0768`） | §8.6、§9.3、§10.2：`timeBase()` 是 pts 换算的输入；`flush()` 是优雅停止的必要动作 |
| `StreamOutput`（多目标、`audioOffset` 归零、trailer、`av_interleaved_write_frame`） | **第 10 课**（`0ccd108`） | §10.4、§12.4：负 pts 被自动归零（测试才敢断言"没有负时间戳"）；`flush()`+`close()` 决定文件能不能打开 |
| PImpl 与 RAII、`= delete` 拷贝 | 第 9/10 课反复用过 | §2.12、§6.3、§12.5：本课再次系统验证——正因为 all lower modules 都用这套，今天才能零改动拼装 |
| "存一张图/产出一个文件让人能验收"的测试纪律 | **第 0 课**提出 | §13.5：产出 `pipeline_a.flv` / `pipeline_b.flv`，并明确告诉用户"用播放器打开应该能看到桌面和系统声音" |

### 附：可以立刻动手的三项重构（对应 Q7）

| # | 位置 | 现状 | 改成 |
|---|---|---|---|
| 1 | `StreamPipeline.cpp` 第 242~243 行 | 直接用 `log(...)`，绕过了 `logMutex` | `emit(...)` |
| 2 | `pipeline/CMakeLists.txt` 第 17 行 | `winmm` 写成 `PUBLIC`（扩大了依赖面） | `PRIVATE`（并确认仍能链接过） |
| 3 | `StreamOutput::open` 开头 | `videoParams`/`audioParams` 为空时行为不确定 | 加判空并返回清晰错误（防御式 API） |

**做完这三项之后，重新跑一遍 `test_pipeline.exe`，确认 22 条断言还是全绿。**

**这就是重构的正确姿势：先有测试保底，再动代码。**

---





