# 第 13 讲：实时预览 —— 把 BGRA 帧变成屏幕上的画面（PreviewWidget + 预览回调 + 帧率限流）

> **对应提交**：`bafb4ea`（2026-10-23 17:20）
> **提交信息**：实时预览
> **改动规模**：7 个文件，315 行 diff（154 行新增 / 9 行删除），新增 `src/app/PreviewWidget.h/.cpp` 两个文件
> **前置课程**：第 12 讲（`3b22c69`，Qt 主界面：信号槽、`QMetaObject::invokeMethod` 跨线程 UI 更新、`QTimer` 轮询统计）
> **本讲时长建议**：约 1.5 ~ 2 小时（含动手实验）

---

## 开场白：主播必须能看到自己在推什么

同学们，先想象一个场景。

你做的是一个直播推流软件。用户填好 RTMP 地址，点"开始推流"，状态栏变成"推流中"，日志里哗哗地刷着"已连接到平台""视频 1830 帧 / 1830 包 / 丢 0"。看起来一切正常。

可是用户在电脑这头，完全不知道**观众看到的画面是什么**。

- 他以为自己推的是"全屏桌面"，结果分辨率栏还停在 1280×720，画面被拉伸变形了；
- 他以为自己推的是摄像头，结果摄像头被别的程序（腾讯会议）占着，采集出来的是一片纯黑；
- 他把窗口最小化到后台去了，DXGI 抓到的是"最小化动画"，观众看到的是一个卡住的桌面角落；
- 他要演示"翻页"，可是翻页的那一刻到底有没有被送出去，他只能去平台直播间里看——而直播间有 3~10 秒的延迟，他得等，等完才知道错了。

这就是**没有预览**的代价：**反馈闭环被拉长到了秒级甚至十秒级，而且闭环的终点在别人的服务器上。**

所以预览窗口的第一个业务意义非常朴素：

> **把"推流是否正常"的判断权，从远端平台拉回到本机；把反馈延迟从 3~10 秒压到 100 毫秒以内。**

一个直播软件，只要屏幕上那块小窗在动，用户的心就定了一半。这是产品体验层面的意义，同学们做界面时一定要记住：**状态不是"日志里有字"就算反馈，用户不看日志。**

### 但是：预览绝不是"把采集帧直接丢给界面"这么简单

现在到了本讲的核心矛盾。很多同学第一次做预览，脑子里的模型是：

```cpp
// 天真的写法（错误示范）
void MainWindow::onPreviewFrame(const VideoFramePtr& frame) {
    previewLabel->setPixmap(QPixmap::fromImage(toQImage(*frame)));  // 每来一帧就 set 一次
}
```

逻辑上没毛病：采集线程给我一帧，我就把它显示出来。可是我们要问三个问题：

**第一问：这一帧有多大？**

我们的 `VideoFrame` 里存的是 **BGRA 32 位原始像素**（第 3 课 `Common.h` 定的契约）。一帧 1920×1080 的 BGRA 是：

```
1920 × 1080 × 4 字节 = 8,294,400 字节 ≈ 7.91 MB
```

每秒 30 帧就是 **237 MB/s** 的像素搬运量。这不是"丢给界面"四个字能概括的——这是每秒 237 兆字节的内存带宽消耗（读写通常各算一遍，实际翻倍）。

**第二问：这一帧要经过几道工序才能上屏？**

从采集缓冲区到屏幕像素，中间至少发生这几件事：

| 工序 | 发生了什么 | 代价 |
|---|---|---|
| ① 保活 | 帧必须活到界面画完为止（`shared_ptr` 续命） | 一次原子引用计数 |
| ② 包装 | 把裸字节包成 `QImage` 描述（不复制像素，只填元数据） | 几十纳秒 |
| ③ 投递 | 从采集线程跨线程送到 GUI 线程（事件排队） | 一次堆分配 + 事件入队 |
| ④ 缩放 | 1920×1080 缩到窗口大小（比如 700×394），**这一步是真正的重活** | 数百万次采样 |
| ⑤ 格式/合成 | Qt 光栅后端把图像画进 backing store（后备缓冲） | 又一次全像素写 |
| ⑥ 上屏 | 操作系统把 backing store 合成到窗口 | 显存拷贝 |

看见了吗？**"像素会被复制、缩放、转换"，每一步都是 O(宽×高)**。其中 ② 是免费的关键——这正是本讲代码里最重要的一个技巧（用外部缓冲构造 `QImage`，零拷贝）。

**第三问：多久刷一次合适？**

采集是 30fps（每 33.3ms 一帧）。可是：

- 人眼在"看大画面有没有动、构图对不对"这件事上，15fps 完全够用；
- GUI 线程是**全界面独一份**的：按钮点击、日志 `appendPlainText`、统计刷新、窗口重绘，全挤在这一条线程上。你把 30 次缩放塞进去，按钮点下去就要等；
- 更糟的是，如果用户没在看预览（窗口最小化 / 在别的标签页），你还在那儿拼命缩放，白烧 CPU；
- 而 CPU 被预览抢走了，编码和采集那两个线程就饿着——**预览做不好会反过来拖垮真正重要的推流链路**。这是最讽刺也最致命的后果。

所以本讲这个提交的标题叫"实时预览"，但它真正的工程含量在**后面那半句看不见的话**：

> **预览要在"看起来实时"和"几乎不花钱"之间取一个平衡点。**

作者给出的答案是三个设计决策，同学们先把结论记住，后面我们一行一行看代码验证：

| 决策 | 做法 | 省下了什么 |
|---|---|---|
| ① **零拷贝包装** | 用 `QImage(外部指针, w, h, stride, Format_ARGB32, 清理回调, 持有者)` 直接引用 `frame->bgra.data()` | 省掉每帧 7.91 MB 的 `memcpy` |
| ② **生产者侧限流** | 在**采集线程**里就判定"距上次投递不足 66ms 就丢弃"，连 `QImage` 都不构造 | 30fps → 15fps，省掉一半的 ①~⑥ 全流程 |
| ③ **异步投递** | `QMetaObject::invokeMethod(..., Qt::QueuedConnection)` 排队给 GUI 线程，采集线程**永不阻塞** | 采集节奏不被 UI 拖慢 |

66ms 这个数字，是本讲最值得玩味的一行代码。我们后面会算它。

好，开场就到这里。今天这一讲的代码量不大（315 行 diff），但**密度很高**：它是我们这门课第一次把"音视频数据"和"Qt 绘制"这两条线真正焊在一起。请同学们打起精神。

我们正式开始。

---

## 第一章：知识点预习

老规矩，动手读代码之前，把今天要用的背景知识一次性补齐。每一条都会在后面的代码里出现。

### 1.1 预览在数据链路上的位置：一个"旁路分叉"

先看清全局。第 11 课（`31c3f57`）我们已经把管线做成了三线程两队列（严格说这次提交时是：视频采集线程 + 音频采集线程 + 写线程，编码在采集线程里同步做）。视频链路长这样：

```
[videoLoop 线程]
  采集（ScreenCapturer / CameraCapturer+Converter）
        ↓ VideoFramePtr (BGRA)
  打 pts（时间规则）
        ↓
  缩放对齐到目标分辨率（sws_scale）
        ↓
  ┌─────────────────────────┐
  │ ★ 本讲新增：预览分叉点   │ ──→ PreviewCallback ──→ MainWindow ──→ PreviewWidget
  └─────────────────────────┘
        ↓
  编码（VideoEncoder）
        ↓ AVPacket
  [有界视频队列] ──→ [writeLoop 线程] ──→ StreamOutput ──→ RTMP
```

请同学们特别注意**分叉点的位置**：它在 `scaleToTarget()` **之后**、编码**之前**。

为什么放在这里？两个理由：

1. **预览看到的就是"送去编码的那一帧"**——同一份分辨率、同一份像素、同一个 pts。如果放在缩放之前，用户看到的（原始 3840×2160）和推上去的（缩放后 1280×720）就不是一回事了，预览就失去了"所见即所推"的意义。这一点是本讲的灵魂，后面"怎么验证预览与实推一致"就靠它。
2. **只分叉一次**，不额外复制。预览拿到的是 `shared_ptr`，编码拿到的是同一个 `shared_ptr`，谁都没复制像素。

> **术语：旁路（tap / side branch）**。从主数据流上引出一支"只看不改"的分支，就叫旁路。旁路的第一原则是**不能影响主干**——这也是为什么我们的预览回调必须"发完就走"，绝不能在里面做耗时操作。

### 1.2 QImage 是什么：一块"带说明书的内存"

同学们先建立一个直觉：

- **`QImage`** = 一块**你自己能摸到的像素内存** + 一份说明书（宽、高、每行字节数、像素格式）。
  - 它活在你的进程内存里（CPU 能直接读写），**你可以直接改某一个像素**。
  - 它是**隐式共享（implicit sharing，也叫写时复制 COW）**的：拷贝一个 `QImage` 只是引用计数 +1，不复制像素；只有当你调用会修改它的函数时，Qt 才会真正复制一份出来（`detach()`）。
  - 适合：加载/保存图片、逐像素处理、以及**从外部缓冲区零拷包构造**（本讲用法）。

- **`QPixmap`** = 一块**交给 Qt/显示系统去管的图**。
  - 它是为"往屏幕上画"而优化的。Qt 可以把它放在离 GPU 更近的地方（不同平台实现不同，Windows 上常在显存/系统后端缓存）。
  - 它**不在**你的普通内存里，想改像素必须先转成 `QImage`，改完再转回去。
  - 适合：**反复绘制**同一张图（比如按钮图标、贴图）。

一句话记住：

> **改像素用 `QImage`，画图用 `QPixmap`。**
> 本讲为什么用 `QImage`？因为我们的像素来自摄像头的内存缓冲区，必须走"外部内存零拷贝"这条路；而它是每帧都在变的，**缓存成 `QPixmap` 没意义**（下一帧就不一样了）。这一点后面第 12 章会再展开成一个优化思考题。

### 1.3 本节最大考点：Format 与"BGRA 到底该配哪个 Format"

这是本讲最容易踩、也最值得讲透的一个坑。请同学们把这一段读三遍。

#### （1）`VideoFrame::bgra` 的内存布局

第 3 课定义的契约是：

```cpp
struct VideoFrame {
    int width = 0;
    int height = 0;
    int64_t pts = 0;
    std::vector<uint8_t> bgra;   // ← 每 4 个字节一个像素
};
```

`bgra` 这个名字就写明了字节顺序：一个像素的 4 个字节，从低地址到高地址依次是：

```
地址:  [p+0]   [p+1]   [p+2]   [p+3]
字节:    B       G       R       A
        蓝      绿      红     透明度
```

这就是所谓的 **BGRA 字节序（byte order）**。

#### （2）Qt 的 Format 命名用的是"32 位整数"而不是"字节序列"

`QImage::Format` 的名字看起来是"ARGB32""RGB32"，很容易让人以为内存里存的是 A、R、G、B 四个字节。**这是错的。**

Qt 的命名法指的是：把这 4 个字节**当作一个 32 位整数**读出来时，它的十六进制写法是什么。

| Qt 格式 | 32 位整数的写法 | 含义 |
|---|---|---|
| `Format_RGB32` | `0xffRRGGBB` | 高字节恒为 0xff（没有 alpha），低 3 字节是 R、G、B |
| `Format_ARGB32` | `0xAARRGGBB` | 最高字节是 alpha，其后 R、G、B |
| `Format_ARGB32_Premultiplied` | `0xAARRGGBB` | 同上，但 RGB 已**预乘** alpha（乘完再存） |
| `Format_RGB888` | —— | 每像素 3 字节，R、G、B |
| `Format_RGBA8888` | 字节序明确为 R,G,B,A | Qt 5.14+ 引入的"字节序无关"格式 |
| `Format_BGRA8888` | 字节序明确为 B,G,R,A | Qt 5.14+ 引入的"字节序无关"格式 |

#### （3）关键跳变：整数 → 内存字节，要经过"端序"

现在关键问题来了：一个 32 位整数 `0xAARRGGBB` 放到内存里，四个字节怎么排？

这取决于 CPU 的**端序（endianness，字节序）**：

- **小端（little-endian）**：低字节放低地址。我们用的 x86 / x64 Windows 就是小端。
- **大端（big-endian）**：高字节放低地址（一些老 PowerPC、部分网络设备）。

在小端机器上，整数 `0xAARRGGBB` 的内存是这样摊开的：

```
整数:   0x AA    RR    GG    BB      （AA 是最高字节，BB 是最低字节）
                ↓ 小端：低字节占低地址
地址:    p+0    p+1   p+2   p+3
字节:     BB     GG    RR    AA
          B      G     R     A   ← 这不就是 BGRA 吗！
```

**所以：在小端机器上，`QImage::Format_ARGB32` 的实际内存字节序列恰好就是 B、G、R、A——和我们的 `VideoFrame::bgra` 完全一致。**

这就是本讲代码里这一行为什么是正确的：

```cpp
QImage image(frame->bgra.data(), frame->width, frame->height, frame->width * 4,
             QImage::Format_ARGB32, ...);
```

> **一句话记住**：`Format_ARGB32` 里的 "ARGB" 说的是**整数的写法**，不是**字节的排列**。在小端机器上，"ARGB32 的整数" == "BGRA 的字节"。

对照表（小端机，也就是我们所有人现在用的机器）：

| Qt Format | 32 位整数 | 内存字节序列（低→高） | 与 `bgra` 匹配吗 |
|---|---|---|---|
| `Format_ARGB32` | `0xAARRGGBB` | **B, G, R, A** | ✅ 完全匹配 |
| `Format_RGB32` | `0xffRRGGBB` | **B, G, R, 0xff** | ⚠️ 前 3 字节对，alpha 被忽略 |
| `Format_BGRA8888`（Qt 5.14+） | 字节序明确 | **B, G, R, A** | ✅ 匹配，且**不依赖端序** |
| `Format_RGBA8888`（Qt 5.14+） | 字节序明确 | **R, G, B, A** | ❌ 红蓝会反 |
| `Format_ARGB32`（在**大端**机上） | `0xAARRGGBB` | A, R, G, B | ❌ 全乱 |

#### （4）那如果配错了会怎样？

同学们记住两张"症状图"，以后看到就知道往哪儿查：

| 错误配置 | 屏幕现象 | 原因 |
|---|---|---|
| 用了 `Format_RGBA8888` | **人脸发蓝、天空发红**（R 和 B 互换） | 红蓝通道对调了 |
| 用了 `Format_RGB888`（3 字节） | **花屏 + 斜条纹 + 颜色错乱** | 步进（stride）对不上：Qt 认为每行 3w 字节，实际是 4w，每一行都会"错位累积" |
| 用了大端语义（如在异构平台） | 全乱 + alpha 污染 | 四个字节全错位 |
| alpha 字节是 0 却用了 `Format_ARGB32` | **预览一片黑/透明**，但推流画面正常 | 见下条 |

关于最后一条，我特别提醒一句，这是本讲代码里**一个真实存在的隐患**：

- `CameraFrameConverter` 里明确写了 `dst[3] = 255;`（摄像头路径）——所以摄像头预览没问题；
- `ScreenCapturer` 的 **DXGI 路径**是 `DXGI_FORMAT_B8G8R8A8_UNORM` 直接 `memcpy`，桌面副本的 alpha **通常是** 255；
- 但 **GDI 降级路径**用的是 `GetDIBits` 拿 32 位 DIB，Windows **不保证**填充 alpha 字节，很可能是 0。

如果 alpha = 0，`Format_ARGB32` 会认为"整幅图完全透明"，画出来就是**背景色**（我们的 `#181818` 深灰），用户看到的就是"预览是黑的，但推流是对的"。

> **排错口诀**：预览黑、推流正常 → 先怀疑 alpha；预览发蓝 → 先怀疑通道顺序；预览花屏斜条 → 先怀疑 stride/Format 的字节数。

修法也简单，二选一：
- 用 `Format_RGB32`（alpha 恒被忽略，不管你填的是什么）；
- 或者构造完 `QImage` 后手动把 alpha 刷成 255。

我们后面第 11 章排错手册里会再详细列。

### 1.4 `QImage` 的"外部缓冲构造函数"：零拷贝 + 清理回调

本讲最重要的一行代码，用的是 `QImage` 的一个重载构造：

```cpp
QImage(uchar *data,
       int width,
       int height,
       qsizetype bytesPerLine,
       QImage::Format format,
       QImageCleanupFunction cleanupFunction = nullptr,
       void *cleanupInfo = nullptr);
```

它的语义是：

1. **不复制像素**。`QImage` 只是记住"我的像素在 `data` 这个地址"，画的时候直接去那儿读。所以这一行是 O(1) 的，一帧 7.91 MB 也不怕。
2. **`QImage` 不拥有这块内存**。所以它必须有一个机制，在"最后一个引用这份图的 `QImage` 被销毁时"通知我们——这就是 `cleanupFunction` + `cleanupInfo`。
3. `cleanupFunction` 的类型是 `void (*)(void*)`；`cleanupInfo` 是你自己传的任意指针，Qt 会原封不动地还给你。
4. **触发时机**：因为 `QImage` 是隐式共享的，你可能会 `QImage b = a;` 拷出好几份。Qt 用原子引用计数跟踪，**当引用计数归零时**才调用一次清理函数。

这个组合拳配上我们的 `shared_ptr`，就是一个非常漂亮的"生命周期接力"：

```
采集线程：new VideoFramePtr(frame)   ← 引用计数 +1，帧被"钉"在堆上
          ↓
       构造 QImage（零拷贝，只引用 frame->bgra.data()）
          ↓
       值捕获进 lambda，投递到 GUI 线程
          ↓
GUI 线程：PreviewWidget::setImage(image) → image_ = image（引用计数 +1）
          ↓
       画完，image_ 被下一帧覆盖 / 或被 clear()
          ↓
       最后一个 QImage 副本析构 → 引用计数归零
          ↓
       cleanupFunction(cleanupInfo) → delete static_cast<VideoFramePtr*>(info)
          ↓
       shared_ptr 析构 → VideoFrame 归还内存
```

**这就是"零拷贝"和"生命周期安全"能同时成立的原因。** 同学们一定要把这个接力图看懂，这是本讲的技术制高点。

> **注意一个硬约束**：`QImage` 要求外部缓冲区**按 4 字节对齐**（对 32 位格式）。我们的 `std::vector<uint8_t>` 由 `operator new` 分配，实际对齐通常是 16 字节，所以安全。但如果你哪天改成"从某个偏移开始"的指针（比如 `data + 1`），就要出事了。

### 1.5 显示一张图的三种做法，我们选了哪一种

在 Qt 里把一张图显示到界面上，常见的有三条路。既然本讲是"预览"专题，我把三条路都摊开讲，讲清楚**为什么作者选了第三条**。

#### 做法一：`QLabel` + `setPixmap`（最省事）

```cpp
// 在 setImage 里：
label->setPixmap(QPixmap::fromImage(image).scaled(
    label->size(), Qt::KeepAspectRatio, Qt::SmoothTransformation));
```

- 优点：一行搞定，`QLabel` 自己管缩放、管居中、管重绘。
- 缺点：**缩放在 CPU 上做，而且每帧都要重新 `scaled()` 一次**——`QPixmap::scaled` 会分配一块新的 pixmap 并做全像素重采样，是货真价实的 O(W×H)。另外 `QLabel` 内部还有布局/尺寸提示的额外计算。
- 适用场景：静态图片、偶尔换一次的图。

> **术语：`Qt::KeepAspectRatio`（保持宽高比）**。`scaled(w, h, Qt::KeepAspectRatio)` 的意思是"缩到能完整塞进 `w×h` 这个框里的**最大**尺寸，且不改变宽高比"，结果一定会留边（letterbox 黑边）。相对的还有 `Qt::IgnoreAspectRatio`（拉满，会变形）和 `Qt::KeepAspectRatioByExpanding`（铺满并裁掉超出部分）。本讲用的是 `KeepAspectRatio`，所以预览左右或上下会留深色边——这是**故意的**，因为变形的画面会让主播误判构图。

#### 做法二：`QLabel` + 每次 `QPixmap::fromImage` 但不预先缩放

把缩放交给 `QLabel` 自己（`setScaledContents(true)`），或者留到 paint 阶段。省了 CPU 缩放，但 `setScaledContents(true)` 走的是"拉伸填充"，**会变形**，不能接受。

#### 做法三：自定义 `QWidget` + 重写 `paintEvent`（本讲的做法）★

```cpp
void PreviewWidget::paintEvent(QPaintEvent*) {
    QPainter painter(this);
    painter.fillRect(rect(), QColor(24, 24, 24));         // 先刷背景（顺便就是黑边）
    if (image_.isNull()) { ...画"未开始推流"...; return; }
    const QSize scaled = image_.size().scaled(size(), Qt::KeepAspectRatio);  // 算目标尺寸
    const QRect target(QPoint(dx, dy), scaled);                             // 居中
    painter.setRenderHint(QPainter::SmoothPixmapTransform, true);           // 平滑
    painter.drawImage(target, image_);                                      // 画
}
```

- 优点：**缩放发生在绘制阶段，且只做一次**；没有中间 `QPixmap` 的额外分配；黑底、居中、占位文字全都在一个地方可控；想加"红边录制框""FPS 角标""分辨率水印"，直接在这个函数里加两行就行。
- 缺点：要自己写代码（也就 20 行）。

> **为什么选它**：预览不是一个"贴一张图"的需求，它是一个"**画布**"。今天画图像，明天可能要在上面画时间码、画"LIVE"角标、画音量条、画鼠标高亮。自定义 `paintEvent` 是**唯一可扩展**的选法。这也是工业界做播放器、做相机取景器的标准做法。

### 1.6 `paintEvent` 与 `update()`：Qt 的重绘是怎么发生的

同学们要建立一个正确的心智模型：**你永远不主动"画"，你只是"请求画"。**

```
   你的代码                 Qt 事件循环                      操作系统
 ─────────────────────────────────────────────────────────────────────
  update()  ──→  投递一个 QEvent::UpdateRequest  ──→  事件循环取出
                                                        ↓
                                              合并脏矩形（同一帧多次
                                              update() 只画一次！）
                                                        ↓
                                              调用 paintEvent(QPaintEvent*)
                                                        ↓
                                              QPainter 画进 backing store
                                                        ↓
                                              窗口区域标记为脏 ──→ 合成上屏
```

要点三条，都是面试和实战都会考的：

1. **`update()` 是异步的**：它只是往事件队列里塞一个请求，立刻返回。所以从采集线程调它是错的（必须走 `invokeMethod` 排队到 GUI 线程），而且连续调 10 次 `update()`，Qt 会把它们合并成**一次** `paintEvent`——这是 Qt 自带的一层限流，非常贴心。
2. **`repaint()` 是同步的**：立刻执行 `paintEvent`。除非你真的知道自己在干什么（比如在非 GUI 线程里画到 `QPixmap`），否则不要用；在采集线程里调它会导致**跨线程绘制，直接崩**。
3. **`paintEvent` 里必须构造 `QPainter`**，并且**只在这个函数里画**。Qt 规定：除了 `paintEvent`，任何地方对 widget 绘制都是非法的（Qt 会警告 "QPainter::begin: Widget painting can only begin as a result of a paintEvent"）。

顺带两个本讲"可以做但没做"的性能细节，我提一句，同学们记在本上：

- `setAttribute(Qt::WA_OpaquePaintEvent, true)`：告诉 Qt"我的 `paintEvent` 会把整个区域都涂满，你不用先帮我擦背景"。我们这里确实 `fillRect(rect(), ...)` 涂满了，加上这一句能省一次全区域擦除。
- 双缓冲：Qt 默认开启（backing store），所以我们不会看到闪烁，不用自己处理。

> **术语：backing store（后备缓冲）**。Qt 为每个顶层窗口维护的一块离屏内存，widget 先画到它上面，再由系统一次性合成到屏幕。这样做的好处是避免"边画边显示"造成的闪烁。

### 1.7 限流的三种做法，以及"66ms"的算术

现在讲本讲第二个技术制高点：**帧率控制**。

#### 方案 A：GUI 线程收到就画（无限流）

30fps 采集 → 每秒 30 次缩放 + 30 次全窗口重绘。GUI 线程被吃满，按钮变卡。**这是我们要避免的。**

#### 方案 B：`QTimer` 定时拉取（消费者侧限流）

```cpp
// 构造函数里：
previewTimer_ = new QTimer(this);
previewTimer_->setInterval(66);            // ≈15fps
connect(previewTimer_, &QTimer::timeout, this, [this] {
    if (auto f = latestFrame_.load())      // 取"最新的一帧"
        showFrame(f);
});
```

思路：采集线程只负责把"最新一帧"存到一个槽里（`std::shared_ptr` 的原子交换），`QTimer` 每 66ms 来取一次。

- 优点：**节奏完全由 GUI 线程掌控**，采集侧几乎零成本；界面忙的时候 `QTimer` 自然被延后，天然自适应。
- 缺点：需要一个额外的"最新帧槽"和同步原语（原子量或互斥锁 + 条件变量）；而且"存最新帧"这一步本身还是每帧都做一次（虽然很便宜）。

> 顺便复习：`QTimer` 我们在第 12 课已经用过一次了——`statsTimer_`，500ms 轮询 `pipeline_.stats()` 刷新统计文字。那个场景是"周期性查数据"，用 `QTimer` 正合适。而本讲的预览是"数据主动来敲门"，所以作者选了下面这个方案。

#### 方案 C：生产者侧时间戳节流（本讲的做法）★

```cpp
const auto now = std::chrono::steady_clock::now();
if (now - lastPreviewTime_ < std::chrono::milliseconds(66))
    return;                       // 不到 66ms 的帧，直接丢掉，连 QImage 都不构造
lastPreviewTime_ = now;
```

- 优点：**被丢弃的帧在采集线程就被扔了**，连"包 `QImage`、投递事件"这两步都省了，浪费最小；实现只要 3 行；不需要额外的同步结构。
- 缺点：节流的最小单位是"采集帧"，所以实际预览帧率是**采集帧率的整数分之一**（30fps → 15fps，60fps → 15fps）；另外 `lastPreviewTime_` 是被采集线程写的普通成员（严格来说是数据竞争，后面第 11 章会讨论）。

**为什么是 66ms？** 我们来算一下：

```
1000 ms ÷ 66 ms = 15.15 帧/秒
```

而采集是 30fps（33.33ms 一帧）。于是：

```
第 1 帧：t=0     → 距上次 ∞ ≥ 66ms，投递   ✅
第 2 帧：t=33.3  → 距上次 33.3 < 66ms，丢   ❌
第 3 帧：t=66.7  → 距上次 66.7 ≥ 66ms，投递 ✅
第 4 帧：t=100   → 距上次 33.3 < 66ms，丢   ❌
...
```

**完美地"隔一帧丢一帧"，输出 15fps。** 66ms 这个数字就是这么来的：比"两帧的间隔 33.3ms"稍短一点点（留出抖动余量），比"三帧的间隔 100ms"短很多。如果写成 67ms，遇到采集抖动（比如某一帧晚了 1ms）就会退化成"隔两帧丢一帧"→ 只有 10fps。写成 66 是个很讲究的工程取值。

> 同学们可以做个对照实验：把 66 改成 200，预览会明显变成"幻灯片"；改成 0，界面立刻开始卡。

#### 一张表总结三种方案

| 方案 | 帧率上限 | 丢弃发生在哪 | 额外同步 | 采集侧成本 | 本讲是否采用 |
|---|---|---|---|---|---|
| A 无限流 | = 采集帧率 | 不丢 | 无 | `QImage` 构造 + 投递，每帧都做 | ❌ |
| B `QTimer` 拉取 | 定时器频率 | GUI 侧（旧帧被覆盖） | 需要"最新帧槽" | 每帧存一次指针（极便宜） | ❌（思考题里请大家实现） |
| C 生产者节流 | 采集帧率的整数分之一 | **采集线程**（最省） | 无（但有一次数据竞争） | 被丢的帧几乎零成本 | ✅ |

### 1.8 跨线程投递帧：第 12 讲知识的"第二次实战"

第 12 讲我们学过一条铁律：

> **Qt 的界面对象只能在 GUI 线程里碰。** 别的线程想改界面，必须"排队"，不能直接调。

排队的标准写法就是：

```cpp
QMetaObject::invokeMethod(接收者对象指针, 可调用对象, Qt::QueuedConnection);
```

它的意思是：把"这个 lambda"包装成一个事件，投到 **接收者所在线程**的事件队列里，等那条线程转回来时执行。

本讲是这条知识的**第二次实战**，但这次的数据比第 12 讲的"一行日志字符串"重得多——是一次 7.91 MB 的像素。所以我们必须格外小心两个问题：

**问题一：lambda 捕获什么？**

我们捕获的是 `QImage image` **值**。因为 `QImage` 是隐式共享的，"值捕获"只是引用计数 +1（原子的、线程安全的），**不会复制 7.91 MB 的像素**。这是能这么写的前提。

> 如果 `QImage` 不是隐式共享的，这么写就完蛋了——每帧复制 8 MB。同学们要养成习惯：**Qt 的容器和 `QImage`/`QPixmap`/`QString` 都是隐式共享的，大胆按值传。**

**问题二：被引用内存的生命周期？**

`QImage` 只是个"引用"，真正的像素在 `frame->bgra` 里。如果 `VideoFrame` 在 GUI 线程画之前就被释放了，`QImage` 就变成了**悬垂引用**——画出来的可能是垃圾数据，也可能直接崩。

解决办法就是我们 1.4 节那张"生命周期接力图"：`new VideoFramePtr(frame)`，把 `shared_ptr` 的一份**拷贝**放到堆上，让 `QImage` 的清理回调在最后负责 `delete` 它。**帧的寿命 = QImage 的寿命。**

### 1.9 CPU 占用的量化直觉

最后给同学们一个能算账的本事。假设 1920×1080 采集、窗口预览区约 700×394：

| 项目 | 每帧成本 | 每秒 15 帧的成本 |
|---|---|---|
| `QImage` 构造（零拷贝） | ~0 | ~0 |
| 事件投递（堆分配 + 入队） | ~1 μs | 忽略 |
| `drawImage` 缩放（双线性，源 207 万像素 → 目标 27 万像素） | ~3~8 ms | **45~120 ms/s**，即 4.5%~12% 的一个核 |
| 背景 `fillRect` + 合成上屏 | ~0.5 ms | ~7.5 ms/s |

也就是说：**一个写得不错的预览，大概吃掉 5%~12% 的单核 CPU**。而如果不限流（30fps），这个数字直接翻倍到 10%~25%，再加上 GUI 线程被占满导致的连锁卡顿——这就是为什么限流不是"锦上添花"，而是**必须做**。

顺带说一句：这里的缩放是 **CPU 光栅缩放**，不是 GPU。Qt Widgets 走的是软件光栅后端（`QPainter` + 光栅 paint engine）。如果哪天要做 4K@60 的预览，就得考虑 `QOpenGLWidget` 或者把缩放任务丢给 `sws_scale`（我们已经链接了 FFmpeg，它高度优化过，还带 SIMD）。这留作思考题。

### 1.10 本讲还有两个"顺手改的小东西"

diff 里还有两处不起眼但值得一提的改动，我先在预习里点一下，后面逐行讲：

1. **`set_target_properties(... WIN32_EXECUTABLE TRUE)`**：这是一个 CMake 属性，作用是把可执行文件的**子系统**从 `Console` 改成 `Windows`。效果：程序启动时**不再弹出黑色控制台窗口**。代价：`printf`/`std::cout` 没有地方输出了（要输出得用 `OutputDebugString` 或日志控件）。对一个 GUI 程序，这是正确的选择。
2. **窗口 `resize(920, 700)` → `resize(1180, 800)`，布局从"上下两栏"改成"左预览右配置"**：这是给预览腾地方。同学们注意，一次看似"功能是预览"的提交，往往伴随着**布局重构**——这是真实开发的常态。

---

## 第二章：diff 全景 —— 315 行里到底改了什么

在逐行读代码之前，先把这次提交的全貌摆出来。 **7 个文件，315 行 diff（154 增 / 9 删）。**

| # | 文件 | 增 | 删 | 角色 |
|---|---|---|---|---|
| 1 | `src/app/PreviewWidget.h` | **18**（新文件） | 0 | 预览控件的**契约**：一个 `QImage` 成员 + 三个函数 |
| 2 | `src/app/PreviewWidget.cpp` | **45**（新文件） | 0 | 预览控件的**实现**：`paintEvent` 里画 |
| 3 | `src/core/pipeline/StreamPipeline.h` | 5 | 0 | 新增 `PreviewCallback` 类型别名 + `setPreviewCallback()` 声明 |
| 4 | `src/core/pipeline/StreamPipeline.cpp` | 19 | 0 | 帧分叉点 + 回调存储 + 设置函数实现 |
| 5 | `src/app/MainWindow.h` | 6 | 0 | 新增 `onPreviewFrame()`、`previewWidget_`、`lastPreviewTime_` |
| 6 | `src/app/MainWindow.cpp` | 60 | 9 | 布局重构 + 回调接线 + `onPreviewFrame()` 主体 |
| 7 | `src/app/CMakeLists.txt` | 4 | 0 | 把新文件挂进构建 + `WIN32_EXECUTABLE` |
| | **合计** | **157** | **9** | |

### 2.1 分层看图：这次改动跨越了"三层"

```
┌─────────────────────────────────────────────────────────────────┐
│ 表现层  src/app/                                                 │
│                                                                  │
│   PreviewWidget  ← 新增：纯粹的"一块画布"，不知道推流的存在       │
│        ↑ setImage(QImage)                                        │
│   MainWindow     ← 改：桥接者。知道推流，也知道界面              │
│        ↑ onPreviewFrame(VideoFramePtr)                           │
└─────────────────────────────────────────────────────────────────┘
                            ↑ PreviewCallback（函数对象）
┌─────────────────────────────────────────────────────────────────┐
│ 核心层  src/core/pipeline/                                       │
│                                                                  │
│   StreamPipeline ← 改：在 videoLoop 里多了一个"分叉点"           │
│                    它不认识 Qt，只认识 std::function              │
└─────────────────────────────────────────────────────────────────┘
                            ↑ VideoFramePtr（第 3 课的契约）
┌─────────────────────────────────────────────────────────────────┐
│ 契约层  src/core/common/Common.h     ← 本讲一行没改               │
│   VideoFrame { width, height, pts, bgra }                        │
└─────────────────────────────────────────────────────────────────┘
```

**这个分层是本讲最值得同学们学习的架构习惯**，请特别注意：

> **`StreamPipeline`（核心层）完全不认识 Qt。** 它没有 `#include <QImage>`，它只认识 `std::function<void(const VideoFramePtr&)>`。

这意味着什么？意味着：

1. 核心层可以脱离界面单独编译、单独测试（我们 `tests/` 目录里那些测试就是这么跑的）；
2. 哪天你想换个界面框架（比如 Qt Quick、或者命令行），核心层一行不用改；
3. "回调"是解耦的通用手段：**核心层说"我这儿有数据，谁要谁注册一个函数给我"**，至于那个函数是去更新 Qt 界面、还是去写文件、还是去统计直方图，核心层一概不问。

对应地，`PreviewWidget`（表现层的画布）也**完全不认识推流**。它只有一个 `setImage(QImage)` 接口，你给它一张 `QImage` 它就画。它不知道 png、不知道 RTMP、不知道 BGRA。

**夹在中间的 `MainWindow`，就是唯一的"翻译官"**：它把 `VideoFramePtr`（核心层语言）翻译成 `QImage`（Qt 语言），并且负责限流、负责跨线程投递。

> 这个分工叫**单一职责**。以后你们做功能，要习惯问自己："这块代码放在这一层合适吗？它是不是知道了太多不该知道的东西？"

### 2.2 一帧预览的完整时序（把三个文件串起来）

```
采集线程（videoLoop）          MainWindow（桥接）            GUI 线程（事件循环）
─────────────────────         ─────────────────           ────────────────────
frame = 采集 + 打 pts
frame = scaleToTarget(frame)
        │
        ├─ 加锁取 previewCallback ─→ callback(frame)
                                        │
                                   onPreviewFrame(frame)
                                        │
                                   ① 判空（帧无效则 return）
                                        │
                                   ② 距上次 < 66ms ? ── 是 ─→ return（丢弃，成本≈0）
                                        │ 否
                                   ③ new VideoFramePtr(frame)  ← 帧被钉在堆上
                                        │
                                   ④ 构造 QImage（零拷贝引用 bgra.data()）
                                        │
                                   ⑤ invokeMethod(QueuedConnection)
                                        │             ┌──────────────┐
                                        └────────────→│  事件队列     │
                                   ⑥ return（采集线程  │  QImage 值捕获│
                                      继续编码，不等） └──────┬───────┘
                                                             ↓
                                                  PreviewWidget::setImage(image)
                                                             │
                                                  image_ = image（引用计数+1）
                                                  update() → UpdateRequest 事件
                                                             ↓
                                                  paintEvent()
                                                   fillRect 背景
                                                   算 scaled 尺寸（KeepAspectRatio）
                                                   居中 → drawImage()
                                                             ↓
                                                  上屏 ✅
                                                             ↓
                                        下一帧覆盖 image_ → 旧 QImage 引用归零
                                                             ↓
                                        cleanupFunction → delete VideoFramePtr
                                                             ↓
                                                  VideoFrame 内存释放 ♻
```

请同学们把这张图记住。接下来 10 章的代码，全都是把这张图的某个框展开。

---

## 第三章：契约层改动 —— `StreamPipeline.h` 的 5 行

我们先从**最上面的一层**开始读：核心管线对外暴露的新契约。

### 3.1 完整改动（diff 第 291~316 行）

```cpp
 #pragma once
 
 #include "core/common/Common.h"
 
 #include <cstdint>
+#include <functional>
 #include <memory>
 #include <string>
 
+using PreviewCallback = std::function<void(const VideoFramePtr&)>;
+
 struct PipelineStats {
     int64_t videoFramesCaptured = 0;
     ...
 };
 
 class StreamPipeline {
 public:
     ...
     bool start(const StreamConfig& config, LogCallback log);
 
+    void setPreviewCallback(PreviewCallback callback);
+
     void stop();
```

### 3.2 逐行讲解

#### 第 1 处：`+#include <functional>`

（diff 第 298 行）

```cpp
+#include <functional>
```

- **为什么加**：因为下面要用 `std::function`。
- **为什么原来没有**：原来的 `StreamPipeline.h` 里已经用了 `LogCallback`（也是 `std::function`），但它是在 `Common.h` 里定义并 `#include <functional>` 的，所以这里是"**间接包含**"碰巧能编过。这次我们自己在 `.h` 里写出 `std::function` 这几个字了，就必须显式包含——这叫**include what you use（用啥包含啥）**。
- **顺带复习**：`<functional>` 提供 `std::function`（可调用对象包装器）。关于 `std::function` 我们在第 2 课就讲过（`LogCallback`）。它是一种**类型擦除**容器：任何"能像函数一样被调用"的东西——普通函数、lambda、仿函数、绑定表达式——只要签名匹配，都能塞进同一个 `std::function` 变量里。

#### 第 2 处：`+using PreviewCallback = std::function<void(const VideoFramePtr&)>;`

（diff 第 302 行）

```cpp
+using PreviewCallback = std::function<void(const VideoFramePtr&)>;
```

拆开看：

| 片段 | 含义 |
|---|---|
| `using X = Y;` | 类型别名，等价于老式的 `typedef Y X;`，但写法更直观（尤其对函数指针/模板） |
| `std::function< 签名 >` | 一个能装下"符合这个签名的任何可调用物"的对象 |
| `void` | 返回类型：回调**没有返回值** |
| `const VideoFramePtr&` | 参数：`VideoFrame` 的 `shared_ptr` 的**常量引用** |

三个设计细节，每个都有讲究：

**（1）为什么参数是 `const VideoFramePtr&`（常引用）而不是 `VideoFramePtr`（值传）？**

值传递一个 `shared_ptr` 会触发一次**原子引用计数加减**（+1 然后 -1）。原子操作在多核上不便宜（几十纳秒，还有缓存行争用）。传常引用则完全不动引用计数。因为回调只是"看"这一帧，不会把它存起来（真要存的话它在内部自己拷一份），常引用完全够用。

**（2）为什么是 `shared_ptr` 而不是裸引用 `const VideoFrame&`？**

这是**生命周期**的考量。回调是同步调用的，看起来用引用也行。但调用方（MainWindow）需要把这一帧**带走**——跨线程送到 GUI 线程去画，那一帧得活到画完为止。用 `shared_ptr`，接收方可以"自己持有一份"（引用计数 +1），帧就不会被采集循环下一帧覆盖掉。

如果用裸引用，MainWindow 就只能在回调里**立刻深拷贝** 7.91 MB —— 那正是我们要避免的。

> **一句话**：`shared_ptr` 在这里的作用不是"共享所有权"，而是"**让接收方有能力延长这一帧的寿命**"。

**（3）为什么返回 `void` 而不是 `bool`（"我还要不要继续"）？**

保持简单。预览是**纯旁路**，它不该有能力影响主干。返回 `void` 在接口层面就宣示了这一点：**你只能看，不能管。**

#### 第 3 处：`+void setPreviewCallback(PreviewCallback callback);`

（diff 第 311 行）

```cpp
+    void setPreviewCallback(PreviewCallback callback);
```

- 放在 `public:` 区，和 `start/stop` 并列——它是管线的公开 API 之一。
- **参数是按值传的 `PreviewCallback`**。这是 `std::function` 的惯用写法：调用方可以传 lambda 临时对象，函数内部 `std::move` 进成员变量，只搬移不复制。
- **注意它可以被传 `nullptr`**：`std::function` 有一个"空"状态，`PreviewCallback(nullptr)` 就是"没有回调"。我们后面 `stopStreaming()` 里就用它来**注销**回调（`pipeline_.setPreviewCallback(nullptr);`）。

> **术语：注册/注销（register / unregister）**。这种模式也叫**观察者模式（Observer）**或**监听器（Listener）**：一个对象（管线）在发生某件事（采集到新帧）时，主动通知一批"登记过的"函数。登记叫注册，撤销登记叫注销。**注销这一步特别容易漏**，漏了就会导致"停止推流后还在往已失效的对象里塞数据"——悬垂指针的经典来源。本讲作者没漏，同学们要学。

### 3.3 一个小观察：为什么没有 `getPreviewCallback()`？

因为不需要。这是一个"只写"的配置项。同学们做接口设计时要克制：**每多一个 public 函数，就多一份要维护、要测试、要向后兼容的承诺。** 用不到的就别加。

---

## 第四章：管线侧改动 —— `StreamPipeline.cpp` 的 19 行

现在看核心层是怎么把帧"分叉"出来的。

### 4.1 改动一：Impl 里加两个成员（diff 第 254~256 行）

```cpp
     std::mutex logMutex;
 
+    std::mutex previewMutex;
+    PreviewCallback previewCallback;
+
     std::atomic<bool> running{false};
```

#### 逐行讲

**（1）`std::mutex previewMutex;`**

- 一把**专用的互斥锁**，只保护下面那个 `previewCallback`。
- 为什么需要锁？因为 `previewCallback` 会被**两个线程**同时碰：
  - **GUI 线程**：`MainWindow::startStreaming()` 里调用 `setPreviewCallback(...)` 写它；
  - **采集线程**（`videoLoop`）：每一帧都读它。
  一边写一边读，没有同步就是**数据竞争（data race）**——C++ 标准说这是未定义行为（UB）。`std::function` 内部有指针和可能的堆缓冲，读到一个"写了一半"的 `std::function`，后果是随机崩溃。

**（2）`PreviewCallback previewCallback;`**

- 存着用户注册进来的那个 `std::function`。
- 默认构造出来是**空的**（`if (callback)` 为假），所以"没注册"和"注册了空"是一回事，代码里不用额外加 `bool hasPreview` 标志位——很省事。

**（3）为什么不用现成的 `logMutex`？**

可以，但**不推荐**。这叫**锁粒度（lock granularity）**问题：

- 用一把大锁保护所有东西 → 简单，但**竞争多**：日志回调和预览回调会互相阻塞；
- 用多把小锁各管各的 → 稍复杂，但**并发度高**，而且能避免一类讨厌的 bug：**锁顺序死锁**（A 等 B 的锁、B 等 A 的锁）。

作者选了小锁，是对的。同学们记住一个原则：

> **一把锁，护一组总是一起变的数据。**

**（4）位置为什么放在 `logMutex` 后面、`running` 前面？**

纯风格：把"回调类"的成员放一起。阅读代码时，相关的东西在物理位置上挨着，能大幅降低理解成本。同学们写结构体时也请养成"**分组 + 空行**"的习惯。

### 4.2 改动二：`videoLoop()` 里的分叉点（diff 第 264~272 行）

这是本讲核心层最关键的一段。先看它在函数里的位置：

```cpp
         frame = scaleToTarget(frame);
         if (!frame) {
             waitNextFrame();
             continue;
         }
 
+        {
+            PreviewCallback callback;
+            {
+                std::lock_guard<std::mutex> lock(previewMutex);
+                callback = previewCallback;
+            }
+            if (callback)
+                callback(frame);
+        }
+
         for (auto& package : videoEncoder.encode(*frame)) {
             videoQueue.push(std::move(packet));
             ++videoPackets;
         }
         ++videoFrames;
```

#### 4.2.1 位置：为什么在 `scaleToTarget` 之后、`encode` 之前

复习 1.1 节：**预览看到的必须正是送进编码器的那一帧。** 代码位置就是设计意图的直接体现。

- 放在 `scaleToTarget` 之后 → 预览的分辨率 = 推流的分辨率（`config.width × config.height`），所见即所推 ✅
- 放在 `encode` 之前 → 预览不依赖编码结果；即使编码器挂了、即使码率设得极低导致画面糊成一团，预览依然是原始清晰的（这也意味着：**预览清晰 ≠ 推流清晰**，这一点在第 10 章"验证一致性"时要特别注意）
- `pts` 也已经打好了（`frame->pts = pts;` 在前几行），所以预览帧和推流帧严格一一对应

#### 4.2.2 逐行讲：那个"两层花括号"

```cpp
+        {
+            PreviewCallback callback;
+            {
+                std::lock_guard<std::mutex> lock(previewMutex);
+                callback = previewCallback;
+            }
+            if (callback)
+                callback(frame);
+        }
```

同学们第一次看这段代码，可能会觉得"套两层花括号，好啰嗦"。我一条条解释，解释完你就明白**每一个花括号都是必要的**。

**第 1 层（外层大括号）**：纯粹为了**限定作用域**。里头定义的 `PreviewCallback callback;` 局部变量，出了这个块就没了。作用：避免这个临时变量污染后面编码那段的命名空间，也避免它被误用。这一层是"整洁"，不是"必需"。

**第 2 行 `PreviewCallback callback;`**：

定义一个**局部副本**。为什么要副本？因为接下来我们要**在锁外面调用它**。

**第 3~6 层（内层花括号 + `lock_guard`）**：

```cpp
             {
                 std::lock_guard<std::mutex> lock(previewMutex);
                 callback = previewCallback;      // 在锁内：把成员拷到局部副本
             }                                     // ← 锁在这里就释放了！
```

- `std::lock_guard` 是 **RAII 锁**（第 3 课讲过）：构造时加锁，**出了作用域（右花括号）自动解锁**。即使中间抛异常也保证解锁。
- 内层花括号的唯一目的，就是**精确控制锁的生命周期**——让锁在"拷贝完成"的那一刻就释放，而不是等到整个块结束。
- **这一步叫"拷出来再放锁"**，是并发编程的标准手法。

**为什么要"拷出来再放锁"？这是本段最值钱的一个知识点。**

请看，如果写成这样（错误示范）：

```cpp
// ❌ 错误示范：持锁调用回调
{
    std::lock_guard<std::mutex> lock(previewMutex);
    if (previewCallback)
        previewCallback(frame);     // ← 锁还拿着！
}
```

会有三个严重后果：

1. **死锁风险**：假设回调里（或者回调的下游）又调了 `setPreviewCallback()`（比如用户在预览上右键→"关闭预览"），而 `std::mutex` 是**不可重入**的——同一线程第二次加锁会直接死锁。
2. **性能灾难**：`previewCallback(frame)` 里面会做 `QImage` 构造、事件投递、甚至（如果改成同步的话）界面绘制。这些可能耗时数毫秒。**采集线程拿着锁做这些事**，而 GUI 线程如果正好想改回调，就得干等。**采集线程被卡住 = 帧率掉了 = 推流卡了**。
3. **优先级倒置**：高优先级线程（采集）等低优先级线程（UI）持有的资源。

所以正确的姿势是：

> **锁只用来保护"取数据"这一瞬间；拿到数据副本后立刻放锁；然后才做真正的工作。**

记住这个模式，你们以后写多线程代码会天天用：

```cpp
值类型 副本;
{
    lock_guard 锁(mu);
    副本 = 共享数据;      // 临界区：越短越好
}                        // 锁释放
用(副本);                // 真正的活儿在锁外干
```

**最后两行**：

```cpp
             if (callback)
                 callback(frame);
```

- `if (callback)`：`std::function` 重载了 `operator bool`，可以直接判断"是否为空"。等价于 `if (callback != nullptr)`，写法更简洁。
- `callback(frame)`：调用。`frame` 是 `VideoFramePtr`，自动绑定到 `const VideoFramePtr&` 参数上。
- **注意这里是同步调用**：采集线程会一直执行到回调返回才继续。所以**回调里必须快**！本讲的回调做了"判空 + 时间戳比较 + 构造 QImage + 投递事件"，全是微秒级，可以接受。如果哪天有人在回调里加了一句"存 BMP 到磁盘"，推流立刻卡成幻灯片。

> 顺带指出一个"可以更好"的地方（留给思考题）：如果回调抛异常，会直接穿透 `videoLoop`，把采集线程搞死。工业代码里通常会在回调外面套 `try/catch`。本讲代码没做，是因为我们的回调是自己写的、不会抛。同学们要知道这个风险点。

### 4.3 改动三：`setPreviewCallback()` 实现（diff 第 286~295 行区域）

```cpp
+void StreamPipeline::setPreviewCallback(PreviewCallback callback)
+{
+    std::lock_guard<std::mutex> lock(impl_->previewMutex);
+    impl_->previewCallback = std::move(callback);
+}
```

放在 `StreamPipeline::~StreamPipeline()` 之后、`start()` 之前。

#### 逐行讲

**（1）签名 `void StreamPipeline::setPreviewCallback(PreviewCallback callback)`**

注意实现里**重复写了参数名 `callback`**。头文件里可以不写参数名（只写类型），但实现里要写（因为要用它）。

**（2）`std::lock_guard<std::mutex> lock(impl_->previewMutex);`**

- 和 `videoLoop` 里那把锁是**同一把**（都是 `impl_->previewMutex`），所以写和读被正确同步了。
- `impl_` 是我们的 **PImpl 指针**（第 8 课讲过：Pointer to Implementation，把所有私有成员藏到一个前向声明的 `Impl` 结构体里，好处是改私有成员不用重新编译所有包含这个头文件的代码）。

**（3）`impl_->previewCallback = std::move(callback);`**

- `std::move` 把参数（一个已经有值的 `std::function`）**搬移**进成员，而不是复制。
- 为什么参数要按值收？因为这样**调用方两种写法都支持**：
  ```cpp
  auto cb = [](const VideoFramePtr& f){ ... };
  pipeline.setPreviewCallback(cb);                    // 左值 → 拷贝进参数（1 次拷贝）
  pipeline.setPreviewCallback([](...){ ... });        // 右值（临时 lambda）→ 移动进参数（0 次拷贝）
  pipeline.setPreviewCallback(nullptr);               // 空
  ```
  这叫 **"按值传参 + move"惯用法**，是 C++11 之后的标准写法。如果参数写成 `const PreviewCallback&`，传临时对象时就只能拷贝了。
- `std::function` 的移动很便宜（搬几个指针，被搬走的对象变成空状态）。

**（4）为什么可以在运行时随时改？**

因为锁保护着。这意味着：
- 推流开始前注册 → 有预览；
- 推流中换成另一个回调 → 换显示目标；
- 传 `nullptr` → 关掉预览，**采集线程下一帧就自动不再调用了**，零成本。

### 4.4 一个"没改但是要强调"的地方：`stop()` 里没有清理回调

同学们去看 `StreamPipeline::stop()`，会发现它没有 `setPreviewCallback(nullptr)`。那"注销"是在哪做的？

在 `MainWindow::stopStreaming()` 里（我们第 8 章讲）：

```cpp
pipeline_.stop();
pipeline_.setPreviewCallback(nullptr);
```

**为什么放在调用方而不是 `stop()` 里？**

这是一个设计选择，各有优劣：

- 放 `stop()` 里：更"自动化"，但 `stop()` 就被迫要知道"预览"这件事；而且如果用户想"停止后再重新 start，还保留回调"，自动清空反而碍事。
- 放调用方：职责清晰——**谁注册，谁注销**。这也是一条通用的资源纪律（和 `new/delete` 配对、和 `fopen/fclose` 配对是同一个道理）。

作者选了后者。我认同：**注册和注销放在一起看，代码才好读**。`startStreaming` 里注册、`stopStreaming` 里注销，一眼就能确认配对上了。

### 4.5 核心层小结：三处改动串成一句话

```cpp
// 1. 存：有锁保护的槽位
std::mutex previewMutex;
PreviewCallback previewCallback;

// 2. 发：缩放之后、编码之前，拷出副本、放锁、调用
{ PreviewCallback cb; { lock; cb = previewCallback; } if (cb) cb(frame); }

// 3. 配：加锁 move 进去，nullptr 即注销
void setPreviewCallback(PreviewCallback cb) { lock; impl_->previewCallback = std::move(cb); }
```

**19 行，一个完整的、线程安全的、可插拔的数据旁路。** 同学们，这就是好代码的样子：小、清楚、不侵入。

---

## 第五章：新文件 `PreviewWidget.h` —— 18 行的契约

现在我们下到表现层，先看新控件的头文件。**完整代码（diff 第 226~245 行，这是文件的全部 18 行）：**

```cpp
#pragma once

#include <QImage>
#include <QWidget>

class PreviewWidget : public QWidget {
public:
    explicit PreviewWidget(QWidget* parent = nullptr);

    void setImage(const QImage& image);
    void clear();

protected:
    void paintEvent(QPaintEvent* event) override;

private:
    QImage image_;
};
```

### 5.1 逐行讲解

#### 第 1 行 `#pragma once`

- **include guard（头文件保护）**：防止同一个头文件在一个编译单元里被展开两次（会导致"类重复定义"编译错误）。
- 老式写法是 `#ifndef XXX_H / #define XXX_H / ... / #endif`。`#pragma once` 是现代写法，几乎所有编译器都支持（MSVC、GCC、Clang），写起来干净。我们的项目统一用它。

#### 第 3~4 行 两个 include

```cpp
#include <QImage>
#include <QWidget>
```

- `<QWidget>`：基类。**必须包含**——我们要继承它，编译器得知道它的完整定义（大小、成员、虚函数表）。
- `<QImage>`：成员 `QImage image_;`。也是**按值成员**，同样需要完整定义。
- **用啥包含啥**，一个不多。注意这里**没有** `#include <QPaintEvent>`——因为 `paintEvent(QPaintEvent* event)` 里 `event` 只是个指针，而且 `.cpp` 里我们根本不用它，所以前向声明就够了（`QWidget` 头文件里已经间接带了）。这也是减少编译依赖的小技巧。

> 对比一下 `MainWindow.h`（第 12 课）：那里用了一堆 `class QComboBox;` `class QLabel;` 这样的**前向声明**，因为那些成员都是**指针**（指针不需要完整定义）。而本文件的 `QImage image_` 是**值成员**，必须包含头文件。**"成员是指针→前向声明；成员是值→必须包含"**，这条规则同学们要背下来。

#### 第 6 行 `class PreviewWidget : public QWidget {`

- 继承 `QWidget`。`QWidget` 是所有界面控件的基类（按钮、标签、输入框都是它的子类）。
- 这里**没有 `Q_OBJECT` 宏**！对比 `MainWindow.h` 里有 `Q_OBJECT`。

**为什么不需要 `Q_OBJECT`？** 这是本讲一个很好的复习点：

> `Q_OBJECT` 宏的作用是启用 Qt 的**元对象系统**：**信号、槽、属性、动态翻译、元对象反射**（`metaObject()`）。
> 本控件**没有定义任何信号或槽**——它只有普通成员函数 `setImage/clear` 和一个虚函数重写 `paintEvent`。而 `paintEvent` 是 C++ 的**虚函数重写**，不是槽，走的是 C++ 虚表，不归元对象系统管。所以不需要 `Q_OBJECT`。

反过来想：如果我们以后想给预览加一个 `void frameShown()` 信号，那就必须加 `Q_OBJECT`（并且要被 `AUTOMOC` 处理到，见第 9 章）。

> **没有 `Q_OBJECT` 的好处**：不需要 moc 生成额外代码，编译更快，文件更干净。

#### 第 8 行 `explicit PreviewWidget(QWidget* parent = nullptr);`

四层含义，逐个拆：

1. **`explicit`**：禁止隐式转换。没有它的话，`PreviewWidget w = someParentWidget;` 这种奇怪的写法也能编译（`QWidget*` 被隐式转成 `PreviewWidget`）。加上 `explicit` 就只允许 `PreviewWidget w(parent);` 这种显式构造。**现代 C++ 铁律：单参数构造函数一律加 `explicit`**（除非你真的想要隐式转换，比如 `std::string` 从 `const char*`）。
2. **构造函数名与类名相同**，无返回类型——C++ 基本语法。
3. **`QWidget* parent = nullptr`**：Qt 的**父对象（parent）**参数。
   > **术语：Qt 对象树 / 父子关系**。Qt 里 `QObject` 之間可以建立父子关系，作用是**自动内存管理**：父对象析构时，会自动 `delete` 所有子对象。所以我们在 `MainWindow::buildUi()` 里 `new PreviewWidget(central)`，把 `central` 设为父对象，**之后完全不用管释放**——`central` 销毁时它就没了，不会内存泄漏。这就是 Qt 里到处 `new` 却不 `delete` 的原因。
   > 另外，对 `QWidget` 而言，`parent` 还有**几何含义**：子控件的坐标是相对父控件的，且会被裁剪在父控件范围内。
4. **`= nullptr`**：默认参数，允许不传（`PreviewWidget w;`）。

#### 第 10~11 行 两个"输入"接口

```cpp
    void setImage(const QImage& image);
    void clear();
```

这是本控件**对外的全部能力**。请注意它们的命名——都是**"命令式"**的，不是 `setImage` 立刻画，而是"我记下了，待会儿画"。

- **`setImage(const QImage& image)`**：
  - 参数用 **`const QImage&`（常量引用）**：避免复制。但因为 `QImage` 是隐式共享的，"复制"其实也只是引用计数 +1，所以就算写成值传也不贵。用常引用是标准习惯：**只读参数用常引用**。
  - 语义："用这张图作为下一次重绘的内容"。
  - **线程安全性**：这个函数必须在 **GUI 线程**调用！因为它要写成员 `image_`，并且会触发 `update()`。从别的线程调它是错的（这也是为什么 `MainWindow` 要用 `invokeMethod`）。
- **`clear()`**：
  - 语义："清空图像，回到占位状态"。
  - 为什么单独做一个 `clear()` 而不是让用户调 `setImage(QImage())`？**可读性**。`setImage(QImage())` 要读者反应一下"哦，空的 QImage 表示清空"，而 `clear()` 一眼就懂。**接口设计要照顾读代码的人。**

#### 第 13~14 行 `protected:` + `paintEvent`

```cpp
protected:
    void paintEvent(QPaintEvent* event) override;
```

- **`protected:`**：`paintEvent` 是 Qt 的**事件处理虚函数**，Qt 框架内部会调用它。它不应该被外部代码随便调用，所以放在 `protected` 区（子类可见，外部不可见）。这是 Qt 的约定，所有 `xxxEvent` 都是 `protected`。
- **`override` 关键字**（C++11）：明确告诉编译器"我是要重写基类的虚函数"。**强烈建议每次都写**——万一你拼错了函数名或者签名写错了（比如写成 `paintEvent(QPaintEvent&)`），没有 `override` 的话编译器会以为你在定义一个全新的函数，静悄悄地就不生效了，你会盯着"为什么我画的东西不出来"查半天。`override` 让这种错误变成编译错误。

> 这是我在第 12 课强调过的习惯，本讲代码继续遵守，非常好。

#### 第 16~17 行 `private: QImage image_;`

```cpp
private:
    QImage image_;
```

- **唯一的成员变量**：一张当前要显示的图。
- 类型直接是 `QImage`（不是指针）：因为 `QImage` 隐式共享、拷贝便宜，值成员比指针更安全（不用管生命周期）。
- 默认构造出来是**空（null）的 `QImage`**，所以控件刚创建时自然处于"未开始推流"状态——**初始状态就是正确状态**，不需要额外的 `bool hasImage` 标志。这又是一个"用类型表达状态"的好例子。
- **命名约定**：成员名后加下划线 `image_`，和局部变量 `image` 区分开。这是本项目统一的风格（`MainWindow` 里全是 `xxx_`）。

#### 第 18 行 `};`

注意——**类定义末尾必须有分号**。这是 C++ 的老坑，忘了会报一堆莫名其妙的错。

### 5.2 头文件设计点评

同学们，这 18 行是一个**教科书级的"最小接口"**：

| 设计点 | 体现 |
|---|---|
| 单一职责 | 它只干一件事：**把一张 `QImage` 按比例画出来** |
| 不知道业务 | 没有 `StreamConfig`、没有 `VideoFrame`、没有 RTMP。给它图它就画 |
| 状态最小化 | 一个 `QImage` 成员，没有冗余标志位 |
| 不可见的不暴露 | `paintEvent` 是 `protected`，`image_` 是 `private` |
| 内存安全 | 靠 Qt 父对象树，不自己管 `delete` |
| 不用元对象 | 没信号槽就不加 `Q_OBJECT` |

以后你们写任何控件，都可以对照这张表自查。

---

## 第六章：新文件 `PreviewWidget.cpp` —— 45 行的实现

**完整代码（diff 第 176~221 行，这是文件的全部 45 行）：**

```cpp
#include "PreviewWidget.h"

#include <QColor>
#include <QPainter>
#include <QPoint>
#include <QRect>
#include <QSize>

PreviewWidget::PreviewWidget(QWidget* parent) : QWidget(parent)
{
    setMinimumSize(320, 180);
    setSizePolicy(QSizePolicy::Expanding, QSizePolicy::Expanding);
}

void PreviewWidget::setImage(const QImage& image)
{
    image_ = image;
    update();
}

void PreviewWidget::clear()
{
    image_ = QImage();
    update();
}

void PreviewWidget::paintEvent(QPaintEvent*)
{
    QPainter painter(this);
    painter.fillRect(rect(), QColor(24, 24, 24));

    if (image_.isNull()) {
        painter.setPen(QColor(150, 150, 150));
        painter.drawText(rect(), Qt::AlignCenter, "未开始推流");
        return;
    }

    const QSize scaled = image_.size().scaled(size(), Qt::KeepAspectRatio);
    const QRect target(QPoint((width() - scaled.width()) / 2,
                              (height() - scaled.height()) / 2),
                       scaled);

    painter.setRenderHint(QPainter::SmoothPixmapTransform, true);
    painter.drawImage(target, image_);
}
```

下面我们**逐行**拆。

### 6.1 include 区（第 1~7 行）

```cpp
#include "PreviewWidget.h"

#include <QColor>
#include <QPainter>
#include <QPoint>
#include <QRect>
#include <QSize>
```

- **第 1 行：先包含自己的头文件**。这是 C++ 的一条重要纪律——**每个 `.cpp` 第一个包含自己的 `.h`**，这样能立刻发现"头文件本身是不是自洽的"（比如头文件里用了 `QImage` 却忘了 include，单独编 `.h` 时才不会漏）。
- **第 3~7 行**：把用到的 Qt 类一个个包含进来：
  - `QColor` → `fillRect` / `setPen` 要用
  - `QPainter` → 绘制器本身
  - `QPoint` → 构造 `QRect` 的左上角
  - `QRect` → 目标矩形
  - `QSize` → `QSize::scaled()` 和 `size()`
- 有趣的细节：**没有 `#include <QPaintEvent>`**。因为 `paintEvent(QPaintEvent*)` 里参数被省略了名字（写成 `QPaintEvent*`），而且函数体里完全没用它。只要基类声明过这个类型（通过 `QWidget`），声明一个"指向它的指针"就够了，不需要完整定义。**少一个 include，编译就快一点。**

> 顺带说一句：Qt 也提供 `<QtWidgets>` 这种"一把梭"的头文件，但**不要用**——它会把所有 Widgets 头文件都展开，编译时间翻好几倍。我们的项目坚持"用啥包含啥"。

### 6.2 构造函数（第 9~13 行）

```cpp
PreviewWidget::PreviewWidget(QWidget* parent) : QWidget(parent)
{
    setMinimumSize(320, 180);
    setSizePolicy(QSizePolicy::Expanding, QSizePolicy::Expanding);
}
```

#### 第 9 行：函数头

```cpp
PreviewWidget::PreviewWidget(QWidget* parent) : QWidget(parent)
```

- `PreviewWidget::` → 类作用域限定，表示这是 `PreviewWidget` 的成员。
- `: QWidget(parent)` → **初始化列表**：把 `parent` 传给基类 `QWidget` 的构造函数。
  > **为什么要走初始化列表而不是在函数体里赋值？** 因为**基类必须在进入函数体之前完成构造**。这是 C++ 的硬规则。所有基类构造、成员初始化都应该写在初始化列表里（效率高、且对 const 成员/引用成员是唯一途径）。
- 注意这里**实现里没写 `explicit`**（`explicit` 只写在声明处，写在定义处是语法错误）。

#### 第 11 行 `setMinimumSize(320, 180);`

```cpp
    setMinimumSize(320, 180);
```

- 设置**最小尺寸**为 320×180。
- 为什么是这两个数？**16:9 的比例**（320/180 = 1.778 = 16/9）。这保证了即使用户把窗口缩到最小，预览区也不会变成一个细条——**最小尺寸就已经是 16:9 了**，为后面的 `KeepAspectRatio` 打底。
- 作用：布局系统（QLayout）在计算空间分配时，不会把这个控件压得比这更小。
- 顺带复习：`setMinimumSize` vs `setFixedSize` vs `setMaximumSize`——分别是最低限、固定死、最高限。我们这里只设下限。

#### 第 12 行 `setSizePolicy(QSizePolicy::Expanding, QSizePolicy::Expanding);`

```cpp
    setSizePolicy(QSizePolicy::Expanding, QSizePolicy::Expanding);
```

这一行很关键，同学们认真听。

> **术语：尺寸策略（QSizePolicy）**。当一个控件被放进**布局（QLayout）**里时，布局要决定"多余的空间分给谁、空间不够时先压缩谁"。`QSizePolicy` 就是这个"竞争规则"的声明。它分**水平方向**和**垂直方向**两个维度，本行两个参数都给了 `Expanding`。

`QSizePolicy::Expanding` 的含义是：

> **"我愿意长大，把多余空间给我。"**

对比几个常见策略：

| 策略 | 含义 | 典型使用者 |
|---|---|---|
| `Fixed` | 尺寸固定，不参与伸缩 | 小按钮 |
| `Preferred` | 理想尺寸是 `sizeHint()`，可以伸缩但**不主动抢** | 普通按钮、输入框 |
| `Expanding` ★ | 理想尺寸是 `sizeHint()`，**主动抢多余空间** | 预览区、日志区、文本编辑框 |
| `MinimumExpanding` | 最小尺寸就是理想尺寸，且主动抢空间 | 窄而必须可见的东西 |
| `Ignored` | 完全忽略 `sizeHint()`，给多少占多少 | 特殊用途 |

**为什么这里必须写 `Expanding`？**

因为如果不写，默认是 `Preferred`（不主动抢空间）。那么当我们把窗口拉大时，多出来的空间会被别的控件（比如配置面板）吃掉，预览区就会"**卡在 `sizeHint()` 那么大不动**"。而 `QWidget` 的默认 `sizeHint()` 是**无效值（-1,-1）**，预览区可能干脆就不长了。

写上 `Expanding`，配合第 8 章要讲的布局 `stretch` 系数（预览 3 : 配置 2），预览区才能"随窗口一起长大，且拿走 60% 的新增空间"。

> **一句话**：`setSizePolicy` = 告诉布局"我想不想要更多空间"，`stretch` = 告诉布局"多余空间按什么比例分"。**两者都要设，缺一个行为就不对。** 这是 Qt 布局里最常被忽略的一对搭配。

### 6.3 `setImage()`（第 15~19 行）

```cpp
void PreviewWidget::setImage(const QImage& image)
{
    image_ = image;
    update();
}
```

只有两行，但每一行都有话说。

#### 第 17 行 `image_ = image;`

- **赋值，不是复制像素**。因为 `QImage` 隐式共享，这行做的是：
  1. 新的 `QImage` 的引用计数 +1；
  2. 旧的 `image_` 的引用计数 -1（如果归零，就触发清理回调，释放那一帧的 `VideoFrame`）；
  3. 把 `image_` 内部指针指向新的数据块。
- **所以"换一帧"的成本是 O(1)**，跟图有多大无关。这是 Qt 隐式共享给我们最大的红利。
- 顺带复习一下**写时复制（COW）**：如果我们之后调一个会修改 `image_` 的函数（比如 `image_.setPixel(...)`），Qt 会先检查引用计数，发现 >1（还有别人在用同一块内存），就**先复制一份**再改，避免影响别人。本代码只读不写，永远不触发复制。

#### 第 18 行 `update();`

- **请求重绘**（异步）。复习 1.6 节：它往事件队列里投一个 `UpdateRequest` 事件，立即返回。
- **为什么不在这里直接画？** 两个原因：
  1. **合法性**：Qt 规定只能在 `paintEvent` 里对 widget 绘制。在其他地方调 `QPainter painter(this);` 会收到警告且不生效。
  2. **效率**：如果一瞬间来了 3 帧（比如程序刚启动时积压了），`update()` 被调用 3 次，Qt 会把它们合并成**一次** `paintEvent`，只画最后一次的图。**自动丢弃中间帧**，正是我们想要的。
- **为什么不用 `repaint()`？** `repaint()` 是同步的，会立刻执行 `paintEvent`。在高频调用的场景下会失去"合并"这个免费限流；而且如果从非 GUI 线程调，直接崩。

> 同学们记住：**`update()` 是"请"，`repaint()` 是"现在立刻马上"。99% 的情况用 `update()`。**

### 6.4 `clear()`（第 21~25 行）

```cpp
void PreviewWidget::clear()
{
    image_ = QImage();
    update();
}
```

- 第 23 行 `image_ = QImage();`：赋值一个**默认构造的 `QImage`**。
  - 默认构造的 `QImage` 是 **null image**（`isNull()` 返回 true，`size()` 是 0×0）。
  - 这一步会让**旧图像的引用计数 -1**。如果这是最后一个引用，就触发我们的清理回调 → `delete VideoFramePtr` → 那一帧被释放。**所以 `clear()` 还兼有"释放最后一帧内存"的作用**，很重要：停止推流后，如果不清，最后一帧的 7.91 MB 会一直挂在那儿（虽然不算泄漏，因为 `image_` 析构时最终也会放，但窗口还开着的话就是白占）。
- 第 24 行 `update();`：请求重绘，画出"未开始推流"占位文字。

### 6.5 `paintEvent()`（第 27~45 行）—— 本讲的重头戏

```cpp
void PreviewWidget::paintEvent(QPaintEvent*)
{
    QPainter painter(this);
    painter.fillRect(rect(), QColor(24, 24, 24));

    if (image_.isNull()) {
        painter.setPen(QColor(150, 150, 150));
        painter.drawText(rect(), Qt::AlignCenter, "未开始推流");
        return;
    }

    const QSize scaled = image_.size().scaled(size(), Qt::KeepAspectRatio);
    const QRect target(QPoint((width() - scaled.width()) / 2,
                              (height() - scaled.height()) / 2),
                       scaled);

    painter.setRenderHint(QPainter::SmoothPixmapTransform, true);
    painter.drawImage(target, image_);
}
```

#### 6.5.1 函数签名（第 27 行）

```cpp
void PreviewWidget::paintEvent(QPaintEvent*)
```

- **`void` 返回值**：事件处理函数不返回值（Qt 不关心你的返回值，想"接受/忽略事件"要调 `event->accept()/ignore()`）。
- **参数 `QPaintEvent*` 没有名字**：这是 C++ 允许的写法，意思是"我知道有这个参数，但我不用它"。
  - 为什么可以不用？`QPaintEvent` 里装的是**本次需要重绘的矩形区域**（`event->rect()`）和需要擦除的区域。精打细算的程序可以只画那一小块（比如一个光标闪烁，只重画光标那一小块）。我们这里是简单的整块重画，用不上。
  - 不给名字的好处：编译器**不会报"未使用的参数"警告**（项目开了 `/W4` 之类的警告级别时很重要）。
- **`override` 在实现里不写**（只写在声明里）——和 `explicit` 一样。

#### 6.5.2 第 29 行 `QPainter painter(this);`

```cpp
    QPainter painter(this);
```

- **`QPainter`** 是 Qt 的"画笔/画师"对象：所有的画线、画矩形、画文字、画图像，都通过它。
- **构造参数 `this`**：指定"往哪个设备上画"。这里传 `this`（本 widget），所以画的东西就直接出现在这个控件上。这个参数叫 **paint device（绘制设备）**，可以是 `QWidget`、`QPixmap`、`QImage`、`QPrinter`……
- **RAII**：构造时 `begin()`，析构时 `end()`。所以**必须在栈上创建**，函数结束时自动结束绘制。如果你 `new QPainter` 忘了 `delete`，绘制状态会一直挂着。
- **生命周期约束**：`QPainter` 必须在 `paintEvent` 里创建，且必须在 `paintEvent` 返回前销毁。
- 这一行的开销：很小（主要是保存/恢复绘制状态）。

#### 6.5.3 第 30 行 `painter.fillRect(rect(), QColor(24, 24, 24));`

```cpp
    painter.fillRect(rect(), QColor(24, 24, 24));
```

- `rect()`：`QWidget` 的成员函数，返回**本控件的矩形**，等价于 `QRect(0, 0, width(), height())`。注意是**控件自身坐标系**，左上角永远是 (0,0)。
- `fillRect(矩形, 颜色)`：用纯色填满这个矩形。
- `QColor(24, 24, 24)`：R=24, G=24, B=24 的**深灰色**（十六进制 `#181818`）。
  - 为什么用这么暗的颜色？**这是视频软件的标准做法**（想想 Premiere、OBS、PotPlayer 的画面区都是深灰/纯黑）。深背景能让画面"跳出来"，也让 letterbox 黑边显得自然。
- **这一行有两个作用**，同学们看清楚：
  1. **清屏**：因为 `update()` 只请求重绘，上一帧的内容还留在 backing store 里，不盖掉就会"残影叠加"。
  2. **画黑边**：当画面比例和控件比例不一致时，`KeepAspectRatio` 会留边，露出来的就是这个深灰。**一举两得**。

> 小优化提示（前面 1.6 节提过）：因为我们**总是**涂满整个控件，可以加 `setAttribute(Qt::WA_OpaquePaintEvent, true);`，告诉 Qt "不用先帮我擦背景了"，省一次全区域擦除。这是留给同学们的重构练习之一。

#### 6.5.4 第 32~36 行 占位分支

```cpp
    if (image_.isNull()) {
        painter.setPen(QColor(150, 150, 150));
        painter.drawText(rect(), Qt::AlignCenter, "未开始推流");
        return;
    }
```

- **第 32 行 `image_.isNull()`**：`QImage::isNull()` 判断"这张图是不是空的"。
  - 什么时候为 true？默认构造（`clear()` 之后的 `image_`）、或者加载图片失败。
  - 注意区分 `isNull()` 和 `isEmpty()`（Qt 6 里 `QImage` 也有 `size().isEmpty()`）：`isNull()` 是"根本没有图"，语义更准。
- **第 33 行 `setPen(QColor(150,150,150))`**：设置**画笔颜色**为中灰。
  > **术语：Pen 与 Brush**。Qt 里 **Pen（画笔）= 描边**（画线、画文字的颜色和线宽），**Brush（画刷）= 填充**（填充矩形、圆的内部）。`fillRect` 用的是 Brush，`drawText`/`drawLine` 用的是 Pen。**这两套是分开的**，新手常常改了 Pen 却发现 `fillRect` 颜色没变——因为它俩不是一个东西。
- **第 34 行 `drawText(rect(), Qt::AlignCenter, "未开始推流")`**：
  - 在 `rect()` 这块区域里画文字，对齐方式是 `Qt::AlignCenter`（**水平居中 + 垂直居中**）。
  - 这比"自己算字符串宽度再算坐标"省事一百倍，是 Qt 的常用技巧。
  - `"未开始推流"` 是**中文字符串字面量**。这里有个工程前提：源文件必须存成 **UTF-8**，而且编译器要被告知"按 UTF-8 解释"。我们的顶层 `CMakeLists.txt` 里有：
    ```cmake
    if(MSVC)
      add_compile_options(/utf-8)
    endif()
    ```
    没有这一句，MSVC 会按本地代码页（GBK）解释 UTF-8 字节，界面上就会出现**乱码**（"鏈寮€鎺ㄦ祦"）。**这是 Windows + Qt 开发的经典坑**，同学们务必记住。（第 12 课我们讲过一次，这里再强调。）
- **第 35 行 `return;`**：画完占位就返回，**不走后面的缩放逻辑**（因为 `image_` 是空的，后面 `scaled()` 会算出 0 尺寸，虽然不崩但没必要）。

#### 6.5.5 第 38~41 行 计算目标矩形（本讲最"数学"的一段）

```cpp
    const QSize scaled = image_.size().scaled(size(), Qt::KeepAspectRatio);
    const QRect target(QPoint((width() - scaled.width()) / 2,
                              (height() - scaled.height()) / 2),
                       scaled);
```

这两行做的是"**保持宽高比地塞进控件，并且居中**"。我们一步一步来。

**第 38 行：**

```cpp
    const QSize scaled = image_.size().scaled(size(), Qt::KeepAspectRatio);
```

- `image_.size()`：图像的原始尺寸，比如 1280×720（也就是 `config` 里选的分辨率）。
- `size()`：控件当前的尺寸，比如 700×394。
- **`QSize::scaled(目标框, 模式)`**：一个非常方便的成员函数，返回"按指定模式缩放后的尺寸"。
  - `Qt::KeepAspectRatio`：结果**一定能完整放进**目标框，且**保持宽高比**。也就是取"宽缩放比"和"高缩放比"里**较小**的那个：
    ```
    宽缩放比 = 框宽 / 图宽 = 700 / 1280 = 0.547
    高缩放比 = 框高 / 图高 = 394 / 720  = 0.547
    取较小者 0.547 → scaled = 1280×0.547 × 720×0.547 = 700×394  ✅
    ```
    再举个例子，如果控件是 700×300（比较扁）：
    ```
    宽缩放比 = 700/1280 = 0.547
    高缩放比 = 300/720  = 0.417   ← 较小
    取 0.417 → scaled = 534×300
    ```
    于是左右各留 `(700-534)/2 = 83` 像素的黑边。✅
- **`const`**：算出来的值不再改，加 `const` 是习惯，也能让编译器帮你检查误改。

> **术语：letterbox / pillarbox（黑边）**。画面比例与容器比例不符时留的边：上下留边叫 letterbox（像信箱口），左右留边叫 pillarbox（像柱子）。我们的代码两种都会自动处理。

**第 39~41 行：**

```cpp
    const QRect target(QPoint((width() - scaled.width()) / 2,
                              (height() - scaled.height()) / 2),
                       scaled);
```

这是一个 `QRect` 的构造，用的是 **`QRect(const QPoint& topLeft, const QSize& size)`** 这个重载。

- **左上角坐标：**
  - `width()`：控件宽；`scaled.width()`：缩放后图像宽。
  - `width() - scaled.width()` = 左右**总共**空出来的像素。
  - `/ 2` → 左边分一半，于是图像**水平居中**。
  - 同理 `(height() - scaled.height()) / 2` → 垂直居中。
- **尺寸：** `scaled`。
- **为什么 `(width() - scaled.width()) / 2` 不会出现负数？** 因为 `KeepAspectRatio` 保证 `scaled` 一定 ≤ `size()`，所以差 ≥ 0。整数除法会向下取整，最多差 1 像素，肉眼不可见。

> 同学们，这两行就是"**保持比例 + 居中**"的全部数学。很多同学第一反应是去找 `QLabel::setScaledContents` 或者自己算浮点比例，其实 Qt 已经给了 `QSize::scaled`。**先查 Qt 有没有现成的，再自己写**——这是提高 Qt 开发效率的第一要诀。

#### 6.5.6 第 43 行 `setRenderHint`

```cpp
    painter.setRenderHint(QPainter::SmoothPixmapTransform, true);
```

- **`RenderHint`（渲染提示）**：告诉绘制引擎"质量优先还是速度优先"。
- `QPainter::SmoothPixmapTransform`：开启**平滑变换**。开启后缩放图像用**双线性插值**（相邻 4 个像素加权平均），画面平滑；不开启则用**最近邻**（直接取最近的那个像素），缩放后会有明显的锯齿和"马赛克块"。
- 对我们的场景：**必须开**。因为预览是从 1280/1920 宽**缩小**到 700 宽，最近邻缩小时会丢弃大量像素，画面会闪烁、细节丢失严重。
- 代价：双线性插值比最近邻慢一些（但和"整块重绘"相比仍是小头）。
- 另一个常用的 hint 是 `QPainter::Antialiasing`（抗锯齿，画斜线圆角时用）。我们这里没有画几何图形，不需要。

> 小知识：这个 hint 叫 "Pixmap" Transform，但对 `drawImage()` 同样生效——在 Qt 的光栅后端里，图像缩放会查询这个 hint 决定用哪种采样器。所以这里是有效的，不是摆设。

#### 6.5.7 第 44 行 `drawImage` —— 真正的"画"

```cpp
    painter.drawImage(target, image_);
```

- `QPainter::drawImage(const QRectF& targetRect, const QImage& image)`（这里传 `QRect` 会隐式转成 `QRectF`）：把 `image` **缩放绘制到** `target` 这个矩形里。
- **这是本函数里唯一一次真正的像素级工作量**：
  - 源：1280×720 = 921,600 像素 × 4 字节 = 3.69 MB；
  - 目标：700×394 = 275,800 像素 × 4 字节 = 1.10 MB；
  - 引擎要为目标每个像素做一次双线性采样（读源图 4 个点、加权、写回）。
- **注意：`drawImage` 不修改 `image_`**。它只读源、写到 widget 的 backing store。所以不会触发 `QImage` 的 COW 复制。
- 到了这一行返回，`painter` 析构（`end()`），一次绘制完成，Qt 把结果合成上屏。

#### 6.5.8 `paintEvent` 的完整心智模型（复习）

```
paintEvent 被调用
   │
   ├─ ① 构造 QPainter(this)              ← 打开绘制上下文
   ├─ ② fillRect(整块, #181818)          ← 清屏 + 铺底（含黑边）
   ├─ ③ 判空？是 → 画灰字"未开始推流" → return
   ├─ ④ 算 scaled（KeepAspectRatio）     ← 保比例
   ├─ ⑤ 算 target（居中）                ← 居中
   ├─ ⑥ 开 SmoothPixmapTransform         ← 质量
   └─ ⑦ drawImage(target, image_)        ← 真正的缩放绘制
   │
painter 析构 → 结束绘制 → Qt 合成上屏
```

### 6.6 `PreviewWidget` 小结

45 行代码，实现了：

- ✅ 保比例显示（不变形）
- ✅ 自动居中 + 深色黑边
- ✅ 空状态占位提示
- ✅ 平滑缩放
- ✅ O(1) 换帧（隐式共享）
- ✅ 自动合并重绘请求（`update()`）
- ✅ 内存自动管理（Qt 对象树 + `QImage` 清理回调）

**没有**做的事（留给大家）：显示分辨率角标、显示 FPS、双击全屏、右键菜单、`WA_OpaquePaintEvent` 优化、`QPixmap` 缓存。这些我们放在思考题里。

---

## 第七章：`MainWindow.h` 的 6 行改动

**完整改动（diff 第 146~171 行）：**

```cpp
 #include <QMainWindow>
 
 #include <atomic>
+#include <chrono>
 #include <mutex>
 #include <thread>
 
 ...
     void postLog(const std::string& message);
 
+    void onPreviewFrame(const VideoFramePtr& frame);
+
     StreamConfig collectConfig() const;
 
+    QWidget* previewWidget_ = nullptr;
+    std::chrono::steady_clock::time_point lastPreviewTime_;
+
     QFormLayout* videoForm_ = nullptr;
```

### 7.1 `+#include <chrono>`（diff 第 153 行）

- 因为下面要用 `std::chrono::steady_clock::time_point`。**用啥包含啥。**
- **为什么用 `steady_clock`（单调时钟）而不是 `system_clock`（系统时钟）？** 这是本讲又一个重要考点：

| 时钟 | 语义 | 会不会跳变 | 适合 |
|---|---|---|---|
| `std::chrono::system_clock` | 墙上时间（wall clock），对应"现在是几点" | **会跳**！用户改系统时间、NTP 校时、夏令时，都会让它往前或往后跳 | 显示日期、日志时间戳 |
| `std::chrono::steady_clock` ★ | 单调时间，从某个固定起点（通常是开机）开始**只增不减** | **永不跳变** | **测间隔、超时、节流、帧率控制** |

如果用 `system_clock` 做节流，用户一把系统时间往后调一小时，`now - last` 变成负数（或者巨大值），节流逻辑直接失效。**所以：所有"计时/节流"场景，一律用 `steady_clock`。** 这条规则同学们请刻在脑子里。

> 顺便复习：`QElapsedTimer`（第 12 课 `MainWindow` 里统计推流时长用的 `elapsed_` 就是它）也是单调时钟，Qt 自己的实现。本讲的代码在**核心/桥接层**（不依赖 Qt 的地方）用 `std::chrono::steady_clock`，在 **Qt 层**用 `QElapsedTimer`——分得很清楚，这是好习惯。

### 7.2 `+void onPreviewFrame(const VideoFramePtr& frame);`（diff 第 161 行）

- 声明新的私有成员函数。放在 `postLog` 之后——**日志回调旁边放预览回调**，语义分组，读起来顺。
- 参数 `const VideoFramePtr&`：和 `PreviewCallback` 的签名一致，所以它能被直接塞进 `std::function`。
- **它会从采集线程被调用**！这一点极其重要：虽然它是 `MainWindow` 的成员函数，但**调用它的线程不是 GUI 线程**。所以函数体内**绝对不能碰任何界面对象**——只能做"算 + 投递"。这一点我们在 8.4 节详细讲。
- **命名**：`on` 前缀表示"这是一个被回调触发的处理函数"（和 Qt 的 `on_xxx` 槽命名习惯一脉相承）。

### 7.3 `+QWidget* previewWidget_ = nullptr;`（diff 第 165 行）

```cpp
    QWidget* previewWidget_ = nullptr;
```

- 保存预览控件的指针。
- **类型是 `QWidget*` 而不是 `PreviewWidget*`** —— 这是本讲代码里**一个明显的设计瑕疵**，我必须指出来给同学们当反面教材：

  1. 为什么作者这么写？可能是为了**避免在头文件里 `#include "PreviewWidget.h"`**（因为只是个指针，可以用前向声明 `class PreviewWidget;`）。但……文件里并没有这么写，而是直接存成了基类指针。
  2. **代价**：每次用都要 `static_cast<PreviewWidget*>(previewWidget_)`（代码里出现了 **3 次**！），又丑又不安全（万一以后换成别的控件，`static_cast` 不检查，直接未定义行为）。
  3. **正确的写法**：
     ```cpp
     // MainWindow.h
     #include "PreviewWidget.h"      // 或者 class PreviewWidget; 前向声明
     ...
     PreviewWidget* previewWidget_ = nullptr;
     ```
     这样后面就不需要任何 `static_cast`。
  4. 我在第 12 章的重构练习里列了这一条，同学们可以自己改。

- **为什么成员要初始化成 `nullptr`？** 因为 `buildUi()` 里才 `new`。在 `buildUi()` 之前（或者万一 `buildUi()` 没跑到），这个指针是空的。**指针成员一律给初值**，这是防野指针的基本功。
- **生命周期**：`previewWidget_` 的父对象是 `central`（见 8.2 节），所以由 Qt 对象树管理，`MainWindow` 只持有**裸指针**即可（**不要用 `shared_ptr`/`unique_ptr` 管理 `QObject`**，会和 Qt 的对象树重复释放！这是 Qt 的铁律）。

### 7.4 `+std::chrono::steady_clock::time_point lastPreviewTime_;`（diff 第 166 行）

```cpp
    std::chrono::steady_clock::time_point lastPreviewTime_;
```

- 记录"上一次投递预览帧的时刻"，用于 66ms 节流。
- **没有给初值**：`time_point` 默认构造出来是"纪元起点"（时钟的零点，通常是开机时刻）。这刚好符合我们的需要——**第一次调用时，`now - lastPreviewTime_` 一定是个巨大的值 ≥ 66ms，所以第一帧必定被投递**。这算是一个"歪打正着"的正确初始化。
  > 严格说，依赖于"默认构造值"是有点隐晦的。更清楚的写法是 `= std::chrono::steady_clock::now();`。但在构造函数里取一次 `now()` 会稍微增加复杂度。作者这样写能工作，我在重构练习里建议改成显式初始化，让意图明确。
- **⚠️ 线程安全警告**：这个成员**会被采集线程写**（`lastPreviewTime_ = now;`），而 `MainWindow` 的其他部分都在 GUI 线程。它是一个**非原子**的 `time_point`（内部通常是一个 64 位整数）。严格按 C++ 标准，如果**两个线程同时**读写它，就是**数据竞争（UB）**。

  实际会不会出问题？在 x64 上，8 字节对齐的 64 位读写是原子的（一条指令完成），所以**实践上不会崩**，最多是"两个采集线程同时进来"时的极小概率竞态——而我们**只有一个视频采集线程**，所以实际上只有一个写者。作者这样写是可以接受的。

  **但同学们要知道这个风险**，并且知道正确解法（见思考题 Q5）：
  ```cpp
  std::atomic<std::chrono::steady_clock::time_point> lastPreviewTime_;   // C++20 起好用
  // 或者
  std::atomic<int64_t> lastPreviewMs_{0};   // 存毫秒计数，兼容性最好
  ```

### 7.5 头文件改动小结

6 行，把"桥接"需要的三样东西备齐了：
1. `<chrono>` → 节流要用的时钟；
2. `onPreviewFrame()` → 回调处理函数；
3. `previewWidget_` + `lastPreviewTime_` → 目标控件 + 节流状态。

---

## 第八章：`MainWindow.cpp` 的 60 行改动 —— 桥接层的全部真相

这是本次提交改动最大的文件。我们分成 **5 段**来讲：

| 段落 | 位置（diff 行号） | 内容 |
|---|---|---|
| 8.1 | 第 32~33 行 | 新增 include |
| 8.2 | 第 41~42、53~59、71~81 行 | **布局重构**：上下栏 → 左预览右配置 |
| 8.3 | 第 89~91 行 | `startStreaming()` 里注册回调 |
| 8.4 | 第 99、103~109 行 | `stopStreaming()` 里注销回调 + 清空预览 |
| 8.5 | 第 117~140 行 | **`onPreviewFrame()` 主体**（本讲核心） |

### 8.1 新增 include（diff 第 32~33 行）

```cpp
 #include "MainWindow.h"
 
+#include "PreviewWidget.h"
+
 #include "core/capture/AudioCapturer.h"
 #include "core/capture/CameraCapturer.h"
```

- 为什么必须包含？因为 `buildUi()` 里要 `new PreviewWidget(...)`，并且要调 `setImage()/clear()`——**需要完整定义**（不能只前向声明，因为要构造对象、调成员函数）。
- **位置**：紧跟自己的头文件 `"MainWindow.h"` 之后，然后是 Qt 的、然后是项目核心层的。这个项目统一按"**自己 → Qt → 项目 → 标准库 → 第三方**"的顺序排 include，很好。
- **注意**：因为头文件里 `previewWidget_` 存的是 `QWidget*`，**头文件其实不需要**这个 include，只有 `.cpp` 需要——这也解释了作者为什么把成员声明成基类指针（可能是想让头文更"轻"）。但我前面说了，这是得不偿失的。

### 8.2 布局重构（diff 第 41~81 行）

这一段的改动量最大，但逻辑其实很清晰：**给预览腾出一块地，并且把配置项纵向堆到右边。**

#### 8.2.1 窗口变大（diff 第 41~42 行）

```cpp
-    resize(920, 700);
+    resize(1180, 800);
```

- 窗口初始尺寸从 920×700 加到 **1180×800**。
- 换算一下：宽多了 260 像素，高多了 100 像素。这 260 像素主要是给预览区（占 3/5，约 156 像素）的。
- 一句题外话：**1180×800 这个比例（1.475）已经接近 3:2**，比原来的 920×700（1.31）更"宽"，符合"左边一块 16:9 画面 + 右边一列配置"的经典布局（想想 OBS 的界面）。

#### 8.2.2 创建预览控件 + 配置面板（diff 第 53~59 行）

```cpp
+    previewWidget_ = new PreviewWidget(central);
+
+    auto* configPanel = new QWidget(central);
+    auto* configLayout = new QVBoxLayout(configPanel);
+    configLayout->setContentsMargins(0, 0, 0, 0);
+    configLayout->addWidget(videoGroup);
+    configLayout->addWidget(audioGroup);
```

逐行：

**第 53 行 `previewWidget_ = new PreviewWidget(central);`**

- `new` 出来，**父对象是 `central`**。
  - 复习：这意味着 `central` 析构时会自动 `delete` 它（Qt 对象树），我们**永远不用手动 delete**。
  - 也意味着它会被放进 `central` 的显示区域（不过最终位置由布局决定）。
- 赋给基类指针成员 `previewWidget_`（`QWidget*`）——隐式向上转换，合法。
- **创建时机**：在 `videoGroup`/`audioGroup` 之后，在 `targetGroup` 之前。顺序本身不重要（布局决定一切），但**新建的控件必须在被 add 到布局之前创建**。

**第 55 行 `auto* configPanel = new QWidget(central);`**

- **新建一个"容器 widget"**，用来装右边那一列配置。
- 为什么要一个"中间容器"？因为我们的布局是：
  ```
  顶层 QHBoxLayout
    ├─ previewWidget   （一个控件，直接放）
    └─ configPanel     （一个容器，里面再套一个 QVBoxLayout 装 5 样东西）
  ```
  **一个 `QWidget` 只能有一个顶层布局**。想在一个"格子"里纵向堆多个控件，就必须先造一个容器 widget，给它一个 `QVBoxLayout`，再把容器放进外层布局。**这是 Qt 布局的核心套路**，同学们一定要掌握——以后任何"嵌套布局"都是这么做的。

**第 56 行 `auto* configLayout = new QVBoxLayout(configPanel);`**

- 创建**纵向**布局（`QVBoxLayout` = Vertical Box Layout），并**直接指定它的父 widget 是 `configPanel`**。
- 这个构造函数一步完成两件事：创建布局 + 把它设置为 `configPanel` 的布局。等价于：
  ```cpp
  auto* configLayout = new QVBoxLayout;
  configPanel->setLayout(configLayout);
  ```
  前者是简写，推荐。

**第 57 行 `configLayout->setContentsMargins(0, 0, 0, 0);`**

- **`contentsMargins`（内容边距）**：布局在四周留的空白（左、上、右、下）。
- 默认是有边距的（通常是 9~11 像素）。这里设成 0，是因为 `configPanel` **本身就是个透明容器**，如果它自己再留一圈边距，右边这一列就会整体往里缩，和左边的预览区对不齐。
- **这是做精致界面的关键细节**：容器 widget 一律把边距清零，让里面的内容"顶格"。

**第 58~59 行 `configLayout->addWidget(videoGroup); configLayout->addWidget(audioGroup);`**

- 把"视频源"和"音频源"两个 `QGroupBox` 纵向塞进配置列。
- 对比原来的写法：
  ```cpp
-    auto* sourceLayout = new QHBoxLayout;
-    sourceLayout->addWidget(videoGroup, 1);
-    sourceLayout->addWidget(audioGroup, 1);
  ```
  原来是**左右并排**（`QHBoxLayout`），现在是**上下堆叠**（`QVBoxLayout`）。因为横向空间被预览抢走了，只能竖向排。

#### 8.2.3 把剩下的东西也塞进配置列（diff 第 71~74 行）

```cpp
-    mainLayout->addLayout(sourceLayout);
-    mainLayout->addWidget(targetGroup);
-    mainLayout->addLayout(controlLayout);
-    mainLayout->addLayout(statsLayout);
+    configLayout->addWidget(targetGroup);
+    configLayout->addLayout(controlLayout);
+    configLayout->addLayout(statsLayout);
+    configLayout->addStretch();
```

- **前 3 行**：`targetGroup`（推流地址）、`controlLayout`（开始/停止按钮行）、`statsLayout`（统计文字行）——**从"顶层纵向布局"搬到"配置列的纵向布局"**。
  - 注意 `addWidget()` vs `addLayout()`：加**控件**用 `addWidget`，加**子布局**用 `addLayout`。Qt 布局可以任意嵌套。
- **第 4 行 `configLayout->addStretch();`** ★

  > **术语：`addStretch()`（弹性空间 / 弹簧）**。往布局里插一个"可伸缩的空白项"。它的作用是**把所有内容往上顶**，多余的空间全被这个弹簧吃掉。

  为什么这里需要它？如果不加，`QVBoxLayout` 会把多余空间**平均分给每个子项**——结果就是"视频源""音频源""推流地址"三个 `QGroupBox` 都被**纵向拉长**，中间出现大片空白，非常难看。加了 `addStretch()`，所有控件都保持自己的自然高度，多出来的空间全在最底部。

  **这是 Qt 布局里第二常用的技巧**（第一是 `addSpacing`）。口诀：
  > **想让内容顶格、空白沉底 → 末尾加 `addStretch()`；想让内容居中 → 首尾各加一个 `addStretch()`；想在两项之间推开 → 中间加 `addStretch()`。**

  顺便看一眼第 12 课已有的代码里，`controlLayout` 和 `statsLayout` 末尾也都加了 `addStretch()`——一致的风格。

#### 8.2.4 组装顶层布局（diff 第 76~81 行）

```cpp
-    mainLayout->addLayout(topLayout);      // （原代码没有，这是新增的）
+    auto* topLayout = new QHBoxLayout;
+    topLayout->addWidget(previewWidget_, 3);
+    topLayout->addWidget(configPanel, 2);
+
+    mainLayout->addLayout(topLayout, 3);
     mainLayout->addWidget(logGroup, 1);
```

**第 76~78 行：**

```cpp
    auto* topLayout = new QHBoxLayout;
    topLayout->addWidget(previewWidget_, 3);
    topLayout->addWidget(configPanel, 2);
```

- 新建**横向**布局 `topLayout`，里面放两样：预览（权重 **3**）和配置面板（权重 **2**）。
- **权重（stretch factor）**：`addWidget(w, stretch)` 的第二个参数。它决定"多余空间怎么分"——**3:2**，也就是预览占 60%、配置占 40%。
- **注意**：stretch 只分配"**超出 sizeHint 的那部分**"空间，不是总空间的 60%。但因为我们的 `PreviewWidget` 设了 `Expanding` 策略且 `sizeHint` 是无效的，实际观感就是"预览明显更大"。
- 这是一个**没有父 widget 的布局**（`new QHBoxLayout` 不传参数）——它下面会被 `addLayout` 挂到 `mainLayout` 上。这是"子布局"的标准写法。

**第 80 行 `mainLayout->addLayout(topLayout, 3);`**

- 把 `topLayout` 加到顶层纵向布局，**权重 3**。
- 第 81 行 `mainLayout->addWidget(logGroup, 1);` → 日志区权重 1。
- 所以**上下比例是 3:1**：上面（预览+配置）占 75% 高度，下面（日志）占 25%。

**第 81 行之后**原来的 4 行（`addLayout(sourceLayout)` 等）被删掉了——这正是 diff 里那 9 行删除的绝大部分。

#### 8.2.5 布局全景图

改完之后，整个窗口的结构是：

```
MainWindow (QMainWindow)
└─ central (QWidget)
   └─ mainLayout (QVBoxLayout)                    ← 纵向
      ├─ topLayout (QHBoxLayout)          [权重3]  ← 横向
      │   ├─ previewWidget_               [权重3]  ← 预览画布（Expanding）
      │   └─ configPanel (QWidget)        [权重2]
      │      └─ configLayout (QVBoxLayout)        ← 纵向，边距 0
      │          ├─ videoGroup (QGroupBox)   视频源
      │          ├─ audioGroup (QGroupBox)   音频源
      │          ├─ targetGroup (QGroupBox)  推流地址
      │          ├─ controlLayout (QHBoxLayout)  开始/停止/状态
      │          ├─ statsLayout (QHBoxLayout)    时长/视频/音频/连接
      │          └─ [Stretch]                    ← 弹簧，空白沉底
      └─ logGroup (QGroupBox)             [权重1] ← 日志
```

**同学们请把这张图记下来。** 以后要往界面上加东西（比如"录制到本地"按钮），你就知道该往哪个 `QxxxLayout` 里 `add` 了。

> 顺便提一句这个重构体现的**工程习惯**：作者没有"在原来的 `sourceLayout` 旁边硬塞一个预览"，而是**重新组织了整个布局层级**。这是对的——**布局是有结构的，功能变了就要改结构，不能打补丁**。硬塞的结果就是界面越来越难维护。

### 8.3 `startStreaming()` 里注册回调（diff 第 89~91 行）

```cpp
         {
             std::lock_guard<std::mutex> lock(pipelineMutex_);
+
+            pipeline_.setPreviewCallback(
+                [this](const VideoFramePtr& frame) { onPreviewFrame(frame); });
+
             ok = pipeline_.start(config, ...);
```

#### 逐行讲

**位置**：在 `worker_` 线程的函数体内、在 `pipelineMutex_` 的**锁内**、在 `pipeline_.start()` **之前**。

三个"为什么"：

**（1）为什么在 `start()` 之前注册？**

因为 `start()` 一调用，采集线程就起来了，**第一帧可能在你还没注册的时候就到了**。先注册再启动，保证**一帧不漏**（第一帧就能显示出来）。

> 这个"先配好再启动"的顺序，是初始化类代码的通用原则。就像你先把摄像头架好再按录制键。

**（2）为什么在 `pipelineMutex_` 锁内？**

`pipelineMutex_` 是 `MainWindow` 用来保护**整个 `pipeline_` 对象**的锁（第 12 课建立）：所有对 `pipeline_` 的调用（`start`/`stop`/`stats`）都在这把锁下。

而 `setPreviewCallback` 内部**还有一把自己的锁**（`previewMutex`）——这是**两把不同的锁，各管一层**：

| 锁 | 保护什么 | 粒度 |
|---|---|---|
| `MainWindow::pipelineMutex_` | 对 `pipeline_` 的**整体操作**（start/stop/stats 互斥） | 粗 |
| `StreamPipeline::Impl::previewMutex` | 只保护 `previewCallback` 这个成员的读写 | 细 |

> 同学们注意：这里**没有死锁风险**，因为两把锁的加锁顺序是固定的（外层 `pipelineMutex_` → 内层 `previewMutex`），而且 `previewMutex` 只在极短的临界区内持有。这叫**锁层次（lock hierarchy）**。

**（3）那个 lambda 是什么？**

```cpp
[this](const VideoFramePtr& frame) { onPreviewFrame(frame); }
```

- `[this]`：**捕获 `this` 指针**。这样 lambda 内部才能调用 `MainWindow::onPreviewFrame()`（成员函数必须通过对象调用）。
- 参数 `const VideoFramePtr& frame`：签名与 `PreviewCallback` 完全匹配，所以能被装进 `std::function`。
- 函数体：转发给成员函数。为什么要多这一层？直接写 `[this](auto& f){...}` 也行，但**单独做一个成员函数**更好：逻辑归逻辑、接线归接线，`onPreviewFrame` 还能被单独阅读和测试。
- 它会被 `std::move` 进 `previewCallback` 成员，一直活到 `setPreviewCallback(nullptr)` 为止。

**（4）⚠️ 捕获 `this` 的风险（本讲最值得警惕的一点）**

这个 lambda 持有 `MainWindow` 的裸指针 `this`，并且被**管线持有**，会在**采集线程**里被调用。

如果 `MainWindow` 先被销毁，而采集线程还在跑 → **悬垂 `this` → 崩溃**。

我们为什么现在是安全的？看 `closeEvent` 和析构函数：

```cpp
void MainWindow::closeEvent(QCloseEvent* event) {
    statsTimer_->stop();
    if (running_.exchange(false) || starting_.exchange(false)) {
        std::lock_guard<std::mutex> lock(pipelineMutex_);
        pipeline_.stop();                 // ① 先停管线
    }
    if (worker_.joinable())
        worker_.join();                   // ② 再等线程结束
    QMainWindow::closeEvent(event);
}

MainWindow::~MainWindow() {
    if (worker_.joinable())
        worker_.join();                   // 析构时也要 join
}
```

顺序是**① 停管线 → ② join 线程 → ③ 对象析构**。只要 `pipeline_.stop()` 能保证"返回后采集线程不会再调回调"（看 `videoLoop`：`while (running)` 循环会在 `running=false` 后退出），那就是安全的。

**但这个安全性依赖于 `stop()` 的实现细节**，是脆弱的。更稳妥的做法（工业级）是让回调持有一个**弱引用**（`QPointer<MainWindow>` 或 `std::weak_ptr`）：

```cpp
QPointer<MainWindow> self(this);
pipeline_.setPreviewCallback([self](const VideoFramePtr& frame) {
    if (self) self->onPreviewFrame(frame);     // 对象已销毁则自动跳过
});
```

> **`QPointer`** 是 Qt 提供的"弱指针"：当被指向的 `QObject` 被销毁时，`QPointer` **自动变成 nullptr**。用它做跨线程回调的守卫，是 Qt 程序的标准防御手法。

这一点我放在思考题 Q6。同学们要意识到：**跨线程回调 + 裸 `this`，是我们这个项目目前最大的一处技术债。**

### 8.4 `stopStreaming()` 里注销回调 + 清空预览（diff 第 99、103~109 行）

```cpp
         {
             std::lock_guard<std::mutex> lock(pipelineMutex_);
             pipeline_.stop();
+            pipeline_.setPreviewCallback(nullptr);
         }
         QMetaObject::invokeMethod(
-            this, [this] { appendLog("已停止"); }, Qt::QueuedConnection);
+            this,
+            [this] {
+                appendLog("已停止");
+                if (previewWidget_)
+                    static_cast<PreviewWidget*>(previewWidget_)->clear();
+            },
+            Qt::QueuedConnection);
     });
```

#### 8.4.1 `+pipeline_.setPreviewCallback(nullptr);`

- **位置**：`pipeline_.stop()` **之后**、同在 `pipelineMutex_` 锁内。
- **为什么在 stop 之后？** 因为 `stop()` 会让 `running=false` 并 join 采集线程。理论上 `stop()` 返回后采集线程已经不会再调回调了，此时注销是"清理状态"。
  > 更保守的顺序其实是**先注销再 stop**：这样即使 `stop()` 过程中还有最后一帧被发出去，回调已经是空的，什么都不会发生。**先断源头，再停流水线**，这是更稳的顺序。这算是一个可以改进的点（重构练习 3）。
- **`nullptr` 的作用**：把 `std::function` 置空。之后 `if (callback)` 为假，采集线程不再调用。**"注销"这一步绝不是可选的**——如果不注销，回调（持有 `this`）会一直挂在管线里，下次 `start()` 之前如果用户关了窗口再开新窗口，就会往旧对象里塞数据。
- 复习第 4.4 节：**谁注册谁注销**。`startStreaming` 注册、`stopStreaming` 注销，配对。

#### 8.4.2 停止后清空预览

```cpp
+            [this] {
+                appendLog("已停止");
+                if (previewWidget_)
+                    static_cast<PreviewWidget*>(previewWidget_)->clear();
+            },
```

- 这段 lambda 通过 `QMetaObject::invokeMethod(..., Qt::QueuedConnection)` **排队到 GUI 线程执行**。
- **为什么必须排队？** 因为这段代码在 `worker_` 线程里跑（整个 lambda 是 `std::thread` 的函数体）。而 `clear()` 会碰 `previewWidget_`（触发 `update()`、写 `image_`）——**界面对象只能在 GUI 线程碰**。第 12 课讲过的铁律，这里再次应用。 ✅
- **为什么清空？** 停止推流后，如果不清除，预览区会**停在最后一帧**。用户会以为"还在推"（画面定格 vs 推流卡死的观感是一样的）。清成"未开始推流"的占位，状态才诚实。**这是很重要的交互细节**。
- **`if (previewWidget_)` 判空**：防御性编程。虽然 `buildUi()` 一定创建了它，但养成"用指针前判空"的习惯能在重构时救命。
- **`static_cast<PreviewWidget*>(previewWidget_)`**：就是 7.3 节批评的那个瑕疵。因为成员是基类 `QWidget*` 指针，必须向下转换才能调 `setImage/clear`。**改成 `PreviewWidget*` 成员就可以省掉这里和后面一共 3 次 `static_cast`。**
- **顺序**：先 `appendLog`（写日志），再 `clear()`（清预览）。都是 GUI 线程内的操作，顺序无所谓，但"先记日志"是本项目的一贯风格。

### 8.5 `onPreviewFrame()` —— 本讲最核心的 24 行（diff 第 117~140 行）

**完整代码：**

```cpp
void MainWindow::onPreviewFrame(const VideoFramePtr& frame)
{
    if (!frame || frame->bgra.empty() || frame->width <= 0 || frame->height <= 0)
        return;

    const auto now = std::chrono::steady_clock::now();
    if (now - lastPreviewTime_ < std::chrono::milliseconds(66))
        return;
    lastPreviewTime_ = now;

    auto* holder = new VideoFramePtr(frame);
    QImage image(frame->bgra.data(), frame->width, frame->height, frame->width * 4,
                 QImage::Format_ARGB32,
                 [](void* info) { delete static_cast<VideoFramePtr*>(info); },
                 holder);

    QMetaObject::invokeMethod(
        this,
        [this, image] {
            if (previewWidget_)
                static_cast<PreviewWidget*>(previewWidget_)->setImage(image);
        },
        Qt::QueuedConnection);
}
```

**这 24 行是本讲的全部精华。** 我们一段一段、一句一句地拆。

#### 第 1 层：函数入口与"有效性兜底"

```cpp
void MainWindow::onPreviewFrame(const VideoFramePtr& frame)
{
    if (!frame || frame->bgra.empty() || frame->width <= 0 || frame->height <= 0)
        return;
```

**第 1 行（函数签名）**：注意这个函数的**执行线程是采集线程**，不是 GUI 线程。请在脑子里贴一张标签：**"此函数在采集线程运行，禁止碰界面！"**

**第 3~4 行（四重判空）**：

```cpp
    if (!frame || frame->bgra.empty() || frame->width <= 0 || frame->height <= 0)
        return;
```

四个条件，一个都不能少，我逐个说明为什么：

| 条件 | 防的是什么 | 现实中的触发场景 |
|---|---|---|
| `!frame` | 空 `shared_ptr` | 采集失败（摄像头被占用、DXGI 会话失效）时 `scaleToTarget` 可能返回 nullptr——虽然管线里已经判过一次，但**回调是公共 API，不能假设调用者都判过** |
| `frame->bgra.empty()` | 空的像素缓冲 | 理论上不该发生，但万一某条采集路径 resize 失败了 |
| `frame->width <= 0` | 非法宽度 | 分辨率配置异常、摄像头协商出 0 |
| `frame->height <= 0` | 非法高度 | 同上 |

**为什么必须判？** 因为下面要用这些数据构造 `QImage`。`QImage` 用 `width<=0` 或 `height<=0` 或 `data==nullptr` 构造，会得到一张 null image（还行），但用 `empty()` 的 vector 调 `.data()` 可能拿到 nullptr，某些 Qt 版本会直接断言崩溃。

> **编程纪律**：**跨越模块边界的数据，一律视为不可信，进入即校验。** 这条规则在写回调、写 API、写解析器时永远适用。作者这里做得很好。

#### 第 2 层：66ms 节流

```cpp
    const auto now = std::chrono::steady_clock::now();
    if (now - lastPreviewTime_ < std::chrono::milliseconds(66))
        return;
    lastPreviewTime_ = now;
```

**第 6 行：**

```cpp
    const auto now = std::chrono::steady_clock::now();
```

- `auto` 自动推导类型为 `std::chrono::steady_clock::time_point`。写 `auto` 避免又长又容易写错的类型名——**`auto` 在类型名冗长时是正当且推荐的**（模板类型尤其）。
- `const`：这个时刻不再改。
- `steady_clock::now()`：**取单调时钟的当前时刻**。复习 7.1 节：这里**必须**用 `steady_clock`，用 `system_clock` 会被改系统时间搞坏。

**第 7~8 行：**

```cpp
    if (now - lastPreviewTime_ < std::chrono::milliseconds(66))
        return;
```

- `now - lastPreviewTime_`：`time_point` 相减得到 `duration`（时间间隔）。类型是 `steady_clock::duration`（通常是纳秒）。
- 和 `std::chrono::milliseconds(66)` 比较：C++ 的 `<chrono>` 会**自动做单位换算**（不同 `duration` 类型比较时，转换成两者的"最大公约数单位"再比），所以不用手动转换单位。这是 `std::chrono` 相比老式 `GetTickCount()` 的巨大优势——**单位写在类型里，永远不会搞错**。
- **`<` 而不是 `<=`**：严格小于才丢。等于 66ms 时放行（无所谓，差一微秒）。
- **`return` 就是"丢帧"**：注意这里**在构造 `QImage` 之前就返回了**。所以被丢掉的帧的成本 = 一次时钟读取（~20ns）+ 一次比较。这就是 1.7 节说的"**生产者侧节流最省**"。

**第 9 行 `lastPreviewTime_ = now;`**

- 更新时间戳。**注意是"通过了的帧"才更新**，不是"每一帧都更新"。
  - 如果每帧都更新，那就变成"距上一帧 ≥66ms 才投递"，也就是只有在采集卡顿（帧间隔 > 66ms）时才投递——**正好把逻辑搞反了**。
  - 现在的写法是"距上次**投递** ≥66ms"，所以稳定输出 15fps。同学们体会一下这个细节。

- ⚠️ **线程安全**：`lastPreviewTime_` 是非原子的普通成员，被采集线程读写。前面 7.4 节讨论过：因为只有一个视频采集线程，实践上安全，但严格来说是数据竞争。见思考题 Q5。

**这一段的三行，就是整个"帧率控制"的全部实现。** 同学们可以对比一下：如果用 `QTimer` 方案，这里要改成"存到原子槽里"，然后 GUI 线程定时取，代码量翻倍。作者的 3 行方案，成本最低。

#### 第 3 层：`new` 一个 `shared_ptr` 的堆副本 —— 生命周期接力的第一棒

```cpp
    auto* holder = new VideoFramePtr(frame);
```

这是我们整讲最"反直觉"的一行，很多同学第一次看会愣住：**为什么要 `new` 一个 `shared_ptr`？`shared_ptr` 不是自己会管内存吗？**

答案是：**我们要让这份 `shared_ptr` 脱离 C++ 的自动生命周期管理，改由 `QImage` 的清理回调来管。**

拆开看：

- `VideoFramePtr(frame)`：**拷贝构造**一个新的 `shared_ptr`，指向同一个 `VideoFrame`。
  - 效果：`VideoFrame` 的引用计数 +1（从 1 变成 2）。
  - 为什么要拷贝？因为参数 `frame` 是**常引用**，它是管线里那个 `shared_ptr` 的引用，生命周期由管线管。我们需要一份**属于我们自己、能活到界面画完**的所有权。
- `new ...`：把这份 `shared_ptr` **放到堆上**，得到一个 `VideoFramePtr*` 裸指针。
  - 为什么放堆上？因为接下来要把它交给 `QImage` 的 `cleanupInfo`（类型是 `void*`）。Qt 不认识 `shared_ptr`，它只认一个 `void*` 和一个"清理函数"。所以**必须有一块地址稳定的堆内存**装着这个 `shared_ptr`。
- **`auto* holder`**：类型是 `VideoFramePtr*`。

**现在这份帧的所有权情况是：**

```
VideoFrame 对象（引用计数 = 2）
   ↑                    ↑
管线的 frame        堆上的 *holder   ← 由 QImage 的清理回调负责 delete
（采集循环结束就释放）  （活到 QImage 最后一个副本析构）
```

**只要 `*holder` 还活着，`VideoFrame` 就绝不会被释放，它的 `bgra.data()` 就永远有效。** 这就保证了 `QImage`（它只是个引用）不会变成悬垂引用。

> **这是 C++ 与 C 风格 API（`void*` + 回调函数指针）对接的经典手法**，同学们以后会经常遇到：
> 1. `new` 一个持有所有权的对象；
> 2. 把指针转成 `void*` 交给 C API；
> 3. 在 C API 的回调里 `static_cast` 回来并 `delete`。
>
> 这个模式的变体你们还会见到：ffmpeg 的 `opaque`、libuv 的 `handle->data`、Windows 的 `GWLP_USERDATA`……**统统是同一个套路**。

**一个必须问的问题：如果 `new` 之后、`QImage` 构造失败会怎样？**

会**泄漏**一个 `VideoFramePtr`（虽然 `QImage` 构造基本不会失败，除非内存耗尽）。工业代码里会写成：

```cpp
auto holder = std::make_unique<VideoFramePtr>(frame);   // 用 unique_ptr 先兜着
QImage image(..., holder.get());
holder.release();                                        // QImage 接手了才释放所有权
```

这是**异常安全**的写法。我把它放在重构练习里。

#### 第 4 层：构造 `QImage` —— 零拷贝包装（本讲技术制高点）

```cpp
    QImage image(frame->bgra.data(), frame->width, frame->height, frame->width * 4,
                 QImage::Format_ARGB32,
                 [](void* info) { delete static_cast<VideoFramePtr*>(info); },
                 holder);
```

这五行（一个语句跨 5 行）是**整讲的核心**。我们把 7 个参数一个个对上号：

| 参数位置 | 实参 | 含义 | 讲解 |
|---|---|---|---|
| 1 `uchar* data` | `frame->bgra.data()` | 像素缓冲区首地址 | `std::vector<uint8_t>::data()` 返回 `uint8_t*`，可隐式转成 `uchar*`（同一类型）。**不复制！** |
| 2 `int width` | `frame->width` | 图像宽度（像素） | 来自采集/缩放后的实际宽度 |
| 3 `int height` | `frame->height` | 图像高度 | |
| 4 `qsizetype bytesPerLine` | `frame->width * 4` | **每行的字节数（stride / 步幅）** | `4` = 每像素 4 字节（BGRA）。我们的是**紧密排列**的（行尾无填充），所以 stride = width×4 |
| 5 `Format` | `QImage::Format_ARGB32` | 像素格式 | 见 1.3 节：小端机上它的内存布局就是 **B,G,R,A**，与 `bgra` 严丝合缝 ✅ |
| 6 `cleanupFunction` | lambda | 清理回调 | 最后一个 `QImage` 副本析构时被调用 |
| 7 `cleanupInfo` | `holder` | 回调的参数 | 就是刚 `new` 出来的那个 `VideoFramePtr*` |

**几个必须讲透的点：**

**（1）参数 4：`bytesPerLine`（步幅/stride）为什么是 `width * 4`？**

> **术语：stride / pitch / bytesPerLine（步幅）**。图像数据在内存里按行存放，**一行占用的字节数**就叫步幅。它**不一定**等于 `宽度 × 每像素字节数`——很多系统（显卡、DIB 位图、视频帧）要求每行按 4 字节或 16 字节对齐，会在行尾**填充**一些无用字节。第 5 课讲摄像头 DIB 时我们就见过对齐公式。

我们的 `VideoFrame::bgra` 是 `std::vector<uint8_t>`，是**紧密排列（packed）**的：一行紧接着一行，没有行尾填充。所以 stride 精确等于 `width * 4`。

**如果 stride 填错会怎样？** 这是"预览花屏"的头号原因：

```
假设真实 stride = 1920×4 = 7680，你填成了 6400：
第 0 行：Qt 从 data+0     读到 data+6400   （只读到真实画面的 5/6）
第 1 行：Qt 从 data+6400  读 —— 但这其实是真实第 0 行的中间！
结果：每一行都相对上一行偏移，累积成"斜条纹"——这就是花屏。
```

**记住：stride 错了 → 斜条纹花屏；通道顺序错了 → 颜色不对；两者都错 → 又斜又变色。**

**（2）参数 5：`Format_ARGB32` 的正确性（再确认一次）**

1.3 节讲透了，这里只做最终确认：

```
我们的内存：   [B][G][R][A]   ← VideoFrame::bgra
小端 ARGB32：  [B][G][R][A]   ← 0xAARRGGBB 摊开
                完全一致 ✅
```

**（3）参数 6：清理回调 lambda**

```cpp
                 [](void* info) { delete static_cast<VideoFramePtr*>(info); },
```

- **`[]` 空捕获**：这是一个**无状态 lambda**。C++ 规定：无捕获的 lambda 可以**隐式转换成函数指针**。而 `QImageCleanupFunction` 的类型正是 `void (*)(void*)`。**如果这里写了 `[this]` 或 `[holder]` 捕获任何东西，就无法转换成函数指针，编译报错！**

  > 这是本行一个非常隐蔽但重要的约束：**清理回调必须是无捕获 lambda（或普通静态函数）**。同学们记住：凡是 C 风格 API 要"函数指针"的地方，你的 lambda 必须无捕获。

- **参数 `void* info`**：Qt 会把第 7 个参数 `holder` 原样传回来。
- **`static_cast<VideoFramePtr*>(info)`**：把 `void*` 转回 `VideoFramePtr*`。
  - 用 `static_cast` 是对的：因为我们知道它本来就是 `VideoFramePtr*` 转过去的（`static_cast` 用于"我知道类型，转回去"）。
- **`delete`**：
  - 销毁这个堆上的 `shared_ptr`。
  - **触发引用计数 -1** → 如果没有别人持有，`VideoFrame` 被析构 → `bgra` 的 7.91 MB 归还。
  - **正是这一步保证了：帧活到最后一刻，然后精确释放。**
- **什么时候被调用？** 当**最后一个**引用这块像素的 `QImage` 被销毁时。我们的场景里可能是：
  - `PreviewWidget::image_` 被下一帧覆盖（`image_ = image;` 时旧图引用归零）；
  - 或者 `clear()`（赋值空 `QImage`）；
  - 或者 `PreviewWidget` 被销毁；
  - 或者（理论上）投递出去但 GUI 线程还没来得及处理的多个副本全都没了。
- **线程**：清理回调在**销毁 `QImage` 的那个线程**执行。我们的场景是 GUI 线程。所以 `delete` 发生在 GUI 线程——安全。

**（4）整行执行完之后，内存状态是什么？**

```
栈上：QImage image（几十字节的元数据：指针、宽高、stride、format、引用计数指针）
                     │
                     └──────────→ 堆：VideoFrame.bgra（7.91 MB，被 holder 钉住）
                                       ↑
                                  *holder（VideoFramePtr，引用计数 2）

零拷贝：没有任何 7.91 MB 的 memcpy。 ✅
```

**（5）为什么这行不危险？有人会问"引用别人的内存，万一被改了怎么办？"**

确实：`QImage` 引用的内存，如果被别处修改，`QImage` 看到的内容也会变。

这里我们安全，因为：
- `VideoFrame` 是**一帧一份**的（`scaleToTarget` 里 `std::make_shared<VideoFrame>()` 新建）；
- 采集循环不会复用同一块 `bgra`（除非 `lastFrame` 缓存，但那是同一份内容）；
- 而且从构造到 GUI 线程画完，只有我们在读。

> **共享内存的通用警告**：零拷贝的代价就是"共享可变状态"。用之前一定要确认：**在我持有期间，有没有人会写这块内存？** 确认没有，才可以安全零拷贝。

#### 第 5 层：跨线程投递

```cpp
    QMetaObject::invokeMethod(
        this,
        [this, image] {
            if (previewWidget_)
                static_cast<PreviewWidget*>(previewWidget_)->setImage(image);
        },
        Qt::QueuedConnection);
```

**第 1 部分：`QMetaObject::invokeMethod(this, functor, Qt::QueuedConnection)`**

- **作用**：把这个 lambda **包装成一个事件**，投递到 **`this`（`MainWindow`）所在线程**的事件队列，然后**立刻返回**。
- `this` 在这里有两个作用：① 决定"投到哪个线程"（`this->thread()`，也就是 GUI 线程）；② 作为"上下文对象"——**如果 `this` 在事件被处理前就被销毁了，Qt 会自动丢弃这个事件**（Qt 内部有守卫）。这是 Qt 给我们的一层保护，很贴心。
- **`Qt::QueuedConnection`**：排队连接。必须显式指定——默认是 `Qt::AutoConnection`，它会判断"当前线程 == 接收者线程？"来决定直连还是排队。我们这里**当前线程是采集线程**，所以 `AutoConnection` 也会自动选排队。但**显式写 `QueuedConnection` 更好**：意图明确，且不依赖判断。
- **为什么不能直接 `previewWidget_->setImage(image);`？** 因为我们在采集线程，直接碰 widget 是**未定义行为**（可能崩、可能画不出来、可能和 GUI 线程的绘制竞态）。第 12 课讲过，这里复习。

**第 2 部分：lambda 的捕获 `[this, image]`**

```cpp
        [this, image] {
```

- **`this`**：要在 lambda 里访问 `previewWidget_` 成员。
- **`image`**：**值捕获** `QImage`。关键问题：**这会不会复制 7.91 MB？**

  **不会！** 因为 `QImage` 是**隐式共享**的，值捕获只是：
  1. 拷贝 `QImage` 的几个字段（指针、宽高、格式……）；
  2. **引用计数 +1**（原子操作）。

  真正的像素**一次都没被复制**。

  > 同学们，这就是 Qt 这套隐式共享设计的价值：**语法上按值传（安全、符合直觉），性能上按引用传（零拷贝）。** 这种"两全其美"是 Qt 相对于很多 GUI 框架的优势。

- **`image` 没有加 `&`**：不能捕获引用！因为 `image` 是栈上的局部变量，函数一返回就没了，捕获引用会得到悬垂引用。**值捕获是对的。**

**第 3 部分：lambda 函数体**

```cpp
            if (previewWidget_)
                static_cast<PreviewWidget*>(previewWidget_)->setImage(image);
```

- 这两行**在 GUI 线程执行**（因为排队到了 GUI 线程）。
- `if (previewWidget_)` → 防御性判空。
- `static_cast<PreviewWidget*>(previewWidget_)` → 向下转型（第 3 次出现这个 cast 了，再次说明成员类型应该是 `PreviewWidget*`）。
- `setImage(image)` → 把图交给画布（`image_ = image` 引用计数 +1，然后 `update()` 请求重绘）。
  - **注意**：`setImage` 是**同步**调用的（在 GUI 线程里，排队事件就是同步执行的）。它本身非常快（O(1) 赋值 + 投一个事件），不会卡住 GUI 线程。
- **到这里，`image` 的一份副本存进了 `PreviewWidget::image_`，帧的寿命被成功延长到了"画完为止"。**

**第 4 部分：这一段的整体时序**

```
采集线程                                    GUI 线程
─────────                                  ────────
onPreviewFrame(frame)
  判空 ✓
  节流 ✓
  new holder
  QImage 构造（零拷贝）
  invokeMethod(Queued) ──── 事件 ────→   [事件队列]
  return  ← 立刻返回！                        │
  继续 encode(frame)                          ↓ （事件循环转到这里）
  ...                                    lambda 执行
                                          previewWidget_->setImage(image)
                                            image_ = image（引用+1）
                                            update() → UpdateRequest
                                                        ↓
                                          paintEvent()
                                            缩放 + drawImage
                                                        ↓
                                          上屏 ✅
```

**注意看：采集线程在 `invokeMethod` 之后立即返回，完全不等 GUI 线程。** 这就是"异步投递"的价值——**预览永远不会拖慢推流**。

#### 8.5.6 `onPreviewFrame` 的 5 步心法（背下来）

```
① 验：帧有效吗？            → 四重判空，无效直接 return
② 限：够 66ms 了吗？        → 不到就丢（此时还没花任何钱）
③ 钉：new VideoFramePtr     → 把帧钉在堆上，交给清理回调管
④ 包：QImage 零拷贝包装      → 只引用不复制，配好清理回调
⑤ 投：invokeMethod 排队      → 异步交给 GUI 线程，立刻返回
```

**这 5 步，就是"把音视频帧安全地送进 Qt 界面"的标准答案。** 同学们以后做任何"实时画面显示"的功能（相机取景器、视频播放器、医学影像、工业相机），都是这 5 步。

### 8.6 `MainWindow.cpp` 改动小结

60 行新增、9 行删除，做了四件事：

1. **布局重构**（约 30 行）：`QHBoxLayout{预览(3) : 配置面板(2)}` 嵌进 `QVBoxLayout{(上,3) : 日志(1)}`，配置面板内部再套 `QVBoxLayout` + `addStretch()`；
2. **注册**（3 行）：`startStreaming` 里、`start()` 之前、`pipelineMutex_` 锁内；
3. **注销 + 清空**（8 行）：`stopStreaming` 里、`stop()` 之后、GUI 线程排队执行 `clear()`；
4. **桥接**（24 行）：`onPreviewFrame` 的验-限-钉-包-投五步。

---

## 第九章：`src/app/CMakeLists.txt` 的 4 行

**完整改动（diff 第 7~24 行）：**

```cmake
 add_executable(LightLiveStreamer
   main.cpp
   MainWindow.cpp
   MainWindow.h
+  PreviewWidget.cpp
+  PreviewWidget.h
 )
 
 target_link_libraries(LightLiveStreamer PRIVATE
   core_pipeline
   Qt6::Widgets
 )
+
+set_target_properties(LightLiveStreamer PROPERTIES WIN32_EXECUTABLE TRUE)
```

### 9.1 新增两个源文件

```cmake
+  PreviewWidget.cpp
+  PreviewWidget.h
```

- **`PreviewWidget.cpp`**：必须加，否则不会编译，链接时报"找不到 `PreviewWidget::paintEvent` 等符号"（`LNK2019 unresolved external symbol`）。
- **`PreviewWidget.h`**：**技术上不必加**（CMake 不编译头文件），但加上有两个好处：
  1. **IDE 友好**：Visual Studio / Qt Creator 的解决方案资源管理器里能看到这个头文件，双击就能打开编辑；
  2. **依赖正确**：CMake 会把它当作源文件的依赖，改动它时触发重新编译。
- **这是一个好习惯**：我们项目里 `MainWindow.h` 也在列表里，保持一致。

> 注意：因为顶层 `CMakeLists.txt` 里有 `set(CMAKE_AUTOMOC ON)`，**如果** `PreviewWidget.h` 里加了 `Q_OBJECT`，CMake 会自动调用 moc 生成 `moc_PreviewWidget.cpp` 并加进编译。我们现在没有 `Q_OBJECT`，所以 moc 不会处理它（省了编译时间）。

### 9.2 `set_target_properties(... WIN32_EXECUTABLE TRUE)`

```cmake
+set_target_properties(LightLiveStreamer PROPERTIES WIN32_EXECUTABLE TRUE)
```

这是一个**很有意思、也很有实际影响**的改动，同学们认真听。

#### 它做了什么

- `WIN32_EXECUTABLE` 是 CMake 的一个**目标属性**。设为 `TRUE` 后，链接器会使用 **`/SUBSYSTEM:WINDOWS`** 而不是 `/SUBSYSTEM:CONSOLE`。
- 直接效果：**程序启动时不再弹出黑色控制台窗口（cmd 那个黑框）**。
- 间接效果：**程序的入口点变成 `WinMain` 而不是 `main`**。
  - 等等，我们的 `main.cpp` 里写的明明是 `int main(int argc, char** argv)`！
  - 因为 **Qt 的 `Qt6::Widgets` 库链接了 `Qt6EntryPoint`（Qt6 自带），它提供了一个 `WinMain` 实现，内部转调到我们的 `main`**。所以不用改代码就能工作。Qt 5 时代需要手动链 `qtmain.lib`，Qt 6 自动处理了。

#### 为什么要加它

- 对一个**正式的 GUI 程序**，启动时闪一个黑框非常不专业（用户会以为程序出错了）。
- 我们之前不加是为了**调试方便**：`printf` / `std::cout` / FFmpeg 的 `av_log` 能打到控制台里。
- 这次加上，说明作者认为"界面部分已经稳定，可以当成品看了"。

#### 加了之后的代价（重要！）

**控制台输出全部失效。** 也就是说：

- `printf("xxx")` → 看不到；
- `std::cout << "xxx"` → 看不到；
- FFmpeg 的 `av_log` 默认输出到 `stderr` → 看不到。

怎么办？三种办法（同学们记一下，调试时会用到）：

| 办法 | 做法 | 适用场景 |
|---|---|---|
| ① 临时改回 | 把这行删掉，或者设成 `FALSE` | 调试期间 |
| ② 用调试器输出 | `OutputDebugStringA("xxx")` + Visual Studio 的"输出"窗口 / Sysinternals 的 DebugView | 长期 |
| ③ 用日志控件 | 我们的 `appendLog()` 就是干这个的 | 长期，本项目首选 |
| ④ 附加控制台 | `AllocConsole()` + 重定向 stdout | 高级玩法 |

> 顺便说一个**排错经验**：如果你发现"程序双击没反应，也不报错"，先检查是不是加了 `WIN32_EXECUTABLE` 之后把所有诊断输出都吞了。我们项目里所有信息都走 `appendLog`，所以不受影响——**这也是"日志走 UI"这个设计的另一个好处**。

### 9.3 为什么 `target_link_libraries` 没变？

因为 `PreviewWidget` 只用了 `QImage` / `QPainter` / `QWidget` / `QColor` / `QRect` —— **全都在 `Qt6::Widgets` 模块里**。

Qt 的模块划分复习一下：

| 模块 | 内容 | 我们的用处 |
|---|---|---|
| `Qt6::Core` | `QString`、`QObject`、`QTimer`、`QFile`…… | 间接依赖 |
| `Qt6::Gui` | `QImage`、`QPainter`、`QColor`、`QFont`…… | `PreviewWidget` 用到 |
| `Qt6::Widgets` | `QWidget`、`QMainWindow`、`QLabel`、布局…… | `MainWindow` + `PreviewWidget` |

因为 **`Widgets` 依赖 `Gui` 依赖 `Core`**，链接 `Qt6::Widgets` 就自动带上了另外两个。所以不需要改。

> 这也是为什么第 12 课链接里只有 `core_pipeline` 和 `Qt6::Widgets` 两项就够了。

---

## 第十章：动手环节

讲了这么多，我们上手跑一遍。这一章请同学们**真的跟着做**，光看代码是学不会画界面的。

### 10.1 编译

```bat
cd C:\...\Streaming-tools
build.bat
```

`build.bat` 会做：CMake 配置 → 编译 → 把 exe 放到 `build/` 目录。

**编译期可能遇到的问题**：

| 现象 | 原因 | 解法 |
|---|---|---|
| `LNK2019: 无法解析的外部符号 "public: void __cdecl PreviewWidget::setImage(...)"` | `PreviewWidget.cpp` 没加进 CMake | 检查 `src/app/CMakeLists.txt` 里有没有那两行 |
| `error C2065: "QPainter": 未声明的标识符` | 忘了 `#include <QPainter>` | 加上 |
| `error C3668: 方法 "paintEvent" 未重写任何基类方法` | 签名写错（比如漏了 `*` 或多写 `const`） | 对照基类：`void paintEvent(QPaintEvent*) override` |
| 中文变成"鏈寮€鎺ㄦ祦" | MSVC 按 GBK 解释了 UTF-8 源文件 | 确认顶层 CMake 有 `add_compile_options(/utf-8)` |
| 编译很慢（第一次） | `PreviewWidget.h` 被 moc 扫了？没 `Q_OBJECT` 就不该扫 | 正常，第一次编译 Qt 相关代码本来就慢 |

### 10.2 运行：你会看到什么

启动 `build/LightLiveStreamer.exe`（**注意：现在不会弹黑框了**，这是 9.2 节那个属性的效果）。

**第一步：未推流状态**

窗口 1180×800，左边一大块深灰（`#181818`），正中间一行中灰色文字"**未开始推流**"。右边是配置项。

✅ 这一眼就能验证：
- `PreviewWidget::paintEvent` 的 `image_.isNull()` 分支生效了；
- `clear()`（或初始状态）正确；
- 中文没有乱码（`/utf-8` 生效）；
- 布局比例：预览区明显比配置区宽（3:2）。

**第二步：随便拉一拉窗口**

- 把窗口拉宽 → 预览区跟着变宽，**比例始终是 3:2**（因为 stretch 3:2）；
- 把窗口**缩到最小** → 预览区停在 **320×180** 不再变小（`setMinimumSize` 生效）；
- 配置那一列的控件**始终顶在上方**，多余的空白沉在底部（`addStretch()` 生效）；
- 日志区始终占底部约 1/4 高度（`mainLayout` 的 3:1）。

✅ 这些验证了 8.2 节的全部布局设计。

**第三步：开始推流**

1. 在"推流地址"里填一个地址。
   - **没有真实地址怎么办？** 用**本地 RTMP 服务器**（比如 `nginx-rtmp`、或者更简单的：填一个本地文件路径，如果 `StreamOutput` 支持；或者用 `rtmp://127.0.0.1:1935/live/test` 配合本地 nginx）。
   - 也可以用第 10 课测试里的方式：起一个本地 RTMP 服务。
2. "视频源"选"屏幕"（最简单，肯定有画面）或"摄像头"。
3. 分辨率选 1280×720，帧率 30。
4. 点"开始推流"。

**你会看到：**

- 状态变成"推流中"；
- **左边的预览区开始显示你的桌面/摄像头画面，并且在动**；
- 画面**不变形**（保持 16:9），左右或上下有深灰边；
- 底部的统计文字每 500ms 刷新一次（第 12 课的 `statsTimer_`）。

✅ 到这里，本讲的功能就跑通了。

### 10.3 关键实验：怎么验证"预览 == 实推"

这是本讲最重要的实验，也是预览这个功能的**唯一验收标准**。同学们一定要做。

> **验收原则**：预览的意义在于"所见即所推"。如果预览看到的和推上去的不一致，那预览不但没用，还会**误导**用户——比没有预览更糟。

#### 实验 10.3.1 构图对照（最直观，必做）

**做法：**
1. 在桌面上打开一个**特征明显**的东西：比如一个写着大字号的记事本、一个满屏的时钟、或者浏览器打开一个颜色鲜艳的网页。
2. 开始推流（屏幕源）。
3. **用观众端看**：
   - 如果有真实平台（B站/抖音），打开直播间（**注意有 3~10 秒延迟**）；
   - 或者用 `ffplay rtmp://...` / VLC 拉流看；
   - 或者最简单：把 `StreamOutput` 写成文件（第 10 课的测试里就写 `.flv`），用 VLC 打开。
4. **逐项对比**：

| 检查项 | 预览里看到的 | 观众端看到的 | 一致吗 |
|---|---|---|---|
| 画面内容 | ? | ? | 应该是**完全相同的一帧**（可能差几秒） |
| 构图/裁切 | ? | ? | 分辨率栏选 1280×720，两者都应是完整的桌面缩放结果，**边缘内容不能缺失** |
| 宽高比 | ? | ? | 都不变形 |
| 颜色 | ? | ? | 一致（尤其注意**红色和蓝色有没有互换**） |
| 有没有镜像 | ? | ? | 桌面源不该镜像；**摄像头源**如果预览和实推都镜像，那也是一致的 |

**如果出现"预览里有的东西，观众端看不到"** → 说明分叉点位置不对，或者预览用了原始帧而推流用了缩放帧。回去查 `videoLoop` 里回调是不是在 `scaleToTarget` 之后。

#### 实验 10.3.2 改分辨率，看预览跟着变（验证"分叉点在缩放之后"）

**做法：** 推流中……（我们的 `updateUiState()` 在推流时禁用了分辨率下拉框，所以**改不了**）。那就：

1. 停止推流；
2. 把分辨率从 **1280×720** 改成 **640×360**；
3. 重新推流；
4. 观察预览。

**预期：** 预览的**画面内容不变**（还是整个桌面），但**分辨率感明显变低**（细节模糊、文字边缘糊），因为预览的源帧已经是 640×360 的了，放大到 700 宽显示自然糊。

✅ 如果你看到"预览糊了"，说明**预览用的确实是缩放后的帧**——分叉点位置正确。

❌ 如果你改了分辨率预览却**毫无变化**（还是很清晰），说明预览用的可能是原始帧——那就要检查代码了。

#### 实验 10.3.3 验证限流：预览到底几 fps？（进阶，需要加日志）

**做法：** 临时在 `PreviewWidget::setImage` 里加计数：

```cpp
void PreviewWidget::setImage(const QImage& image)
{
    static int count = 0;
    static QElapsedTimer t; 
    if (!t.isValid()) t.start();
    if (++count % 30 == 0)
        qDebug("预览 fps = %.1f", count * 1000.0 / t.elapsed());
    image_ = image;
    update();
}
```

**预期：** 采集 30fps 时，预览约 **15 fps**。

**然后把 66 改成别的值再测：**

| `milliseconds(X)` | 理论预览 fps | 实测 | 界面手感 |
|---|---|---|---|
| `0` | 30（=采集帧率） | ? | 明显变卡，点按钮有延迟 |
| `33` | 30（几乎不丢） | ? | 和 0 差不多 |
| `66`（现在） | 15.15 | ? | **流畅，且不卡** ← 甜点 |
| `100` | 10 | ? | 略有顿挫感 |
| `200` | 5 | ? | 明显的"幻灯片" |

✅ 做完这个实验，同学们对"帧率控制"就有**体感**了，不再是纸上谈兵。

> 顺便观察任务管理器里的 CPU 占用：把 66 改成 0，CPU 应该**明显上升**（0.5% → 几个百分点，取决于分辨率）。这就是 1.9 节那笔账的实证。

#### 实验 10.3.4 验证"停止后清空"

1. 推流中（预览有画面）；
2. 点"停止推流"；
3. **预期**：日志出现"已停止"，同时**预览区立刻变回深灰 + "未开始推流"**。

✅ 验证了 8.4 节那段"排队到 GUI 线程执行 `clear()`"的代码。

❌ 如果画面**定格在最后一帧** → 说明 `clear()` 没被调用（检查 `invokeMethod` 是不是漏了，或者 `previewWidget_` 是不是空的）。

#### 实验 10.3.5 验证零拷贝（内存视角）

**做法：** 开始推流前后，看任务管理器里进程的**内存占用**（或者用 Visual Studio 的诊断工具）。

另一更有说服力的方式：在清理回调里加计数：

```cpp
                 [](void* info) { 
                     static std::atomic<int> n{0};
                     qDebug("清理第 %d 帧", ++n);
                     delete static_cast<VideoFramePtr*>(info); 
                 },
```

**预期：** 每显示一帧，最终都会对应一次清理。**如果清理次数远少于显示次数**，说明有 `QImage` 副本泄漏（引用计数没归零）；**如果远多于**，说明你可能在别处不小心复制了。

✅ 这个实验能证明"生命周期接力"是闭合的、**没有内存泄漏**。

### 10.4 几个"改造实验"（每个都很小，但能加深理解）

| # | 改什么 | 预期现象 | 验证的知识点 |
|---|---|---|---|
| 1 | `Format_ARGB32` → `Format_RGBA8888` | **红色和蓝色互换**（人脸发蓝） | 1.3 节通道顺序 |
| 2 | `frame->width * 4` → `frame->width * 3` | **斜条纹花屏** | stride 的作用 |
| 3 | `Format_ARGB32` → `Format_RGB32` | 画面正常（若 alpha 本来就是 255）；**若原来是黑的，这会让它变亮** | alpha 是否被使用 |
| 4 | `Qt::KeepAspectRatio` → `Qt::IgnoreAspectRatio` | 画面**铺满预览区但变形**（人脸变扁/变长） | 为什么要保比例 |
| 5 | 删掉 `setRenderHint(SmoothPixmapTransform, true)` | 画面出现**锯齿/马赛克**（尤其缩小看文字时） | 平滑缩放 |
| 6 | `update()` → `repaint()` | 大概率**程序崩溃**（跨线程绘制）或告警 | 为什么必须用 `update()` |
| 7 | 删掉 `setSizePolicy(...Expanding...)` | 预览区**不再随窗口长大**（卡在小尺寸） | `QSizePolicy` 的作用 |
| 8 | 删掉 `configLayout->addStretch()` | 右边三个 GroupBox **被纵向拉长**，中间出现大片空白 | `addStretch()` 的作用 |
| 9 | 在 `onPreviewFrame` 开头加 `std::this_thread::sleep_for(50ms)` | **推流掉帧、画面卡**（采集线程被阻塞） | 回调必须快！ |
| 10 | 把 `setPreviewCallback(nullptr)` 那行删掉 | 停止后再开始，**可能往旧对象塞数据**（轻则没预览，重则崩） | 注销的必要性 |

**实验 9 特别值得做**——它能让你"看见"整个架构的命门：**采集线程是推流的心脏，任何阻塞它的代码都是犯罪。**

做完实验记得把代码改回来（或者用 `git checkout` 恢复）。

---

## 第十一章：排错手册

这一章按"**你看到了什么现象**"来组织，同学们遇到问题时按症状查表。

### 11.1 画面类问题

#### ❌ 症状 1：预览一片黑（深灰），但推流画面正常

**最可能的原因：alpha 通道是 0，被 `Format_ARGB32` 当成"全透明"。**

回顾 1.3 节：`Format_ARGB32` 会读第 4 个字节当 alpha。如果它是 0，整幅图透明，画出来只剩 `fillRect` 的深灰背景。

什么时候 alpha 是 0？
- `ScreenCapturer` 的 **GDI 降级路径**：`GetDIBits` 拿 32 位 DIB，Windows 不保证填 alpha（很可能是 0）；
- DXGI 路径（`B8G8R8A8_UNORM`）通常是不透明的，但**不保证**；
- 摄像头路径：`CameraFrameConverter` 明确写了 `dst[3] = 255`，安全。

**排查：**
```cpp
// 在 onPreviewFrame 里临时加：
qDebug("第一个像素: B=%d G=%d R=%d A=%d",
       frame->bgra[0], frame->bgra[1], frame->bgra[2], frame->bgra[3]);
```
如果 `A=0` 而你看到的画面不是黑的 → 就是这个坑。

**解法（三选一）：**
1. 改用 `QImage::Format_RGB32`（alpha 被忽略，恒不透明）——**最简单，推荐**；
2. 构造后手动刷 alpha（慢，不推荐）；
3. 在采集端填 255（治本，但要改采集器）。

#### ❌ 症状 2：颜色发蓝 / 发红（红蓝互换）

**原因：通道顺序配错了。** 用了 `Format_RGBA8888`（内存是 R,G,B,A）去解释 B,G,R,A 的数据。

**解法：** 换回 `Format_ARGB32`（小端）或 `Format_BGRA8888`（端序无关，更推荐用于新代码）。

#### ❌ 症状 3：花屏 —— 斜条纹、错位、颜色乱七八糟

**原因排行：**

| 排名 | 原因 | 表现 |
|---|---|---|
| 1 | **stride 填错**（`bytesPerLine` 不是 `width*4`） | 每一行相对上一行偏移 → 斜条纹 |
| 2 | **宽高填错**（比如填了缩放前的尺寸） | 画面被切成几块错位的条带 |
| 3 | Format 的每像素字节数不对（比如 `Format_RGB888` 是 3 字节） | 又斜又变色 |
| 4 | 数据缓冲区不是紧密排列的（有行尾对齐填充） | 同上 |

**排查：**
```cpp
qDebug("w=%d h=%d stride=%d 实际字节数=%zu",
       frame->width, frame->height, frame->width*4, frame->bgra.size());
// 检查：bgra.size() 应该 == w*h*4
```
如果 `bgra.size() != w*h*4`，说明缓冲区**不是紧密排列**的，stride 就不能用 `width*4`，得用实际的 stride（需要在 `VideoFrame` 里加一个 `stride` 字段——这其实是我们契约层的一个**已知缺陷**，见思考题 Q8）。

**解法：** 确认 `bgra.size() == width*height*4`；确认 stride 参数 = `width*4`。

#### ❌ 症状 4：画面上下颠倒

**原因：图像的"扫描方向"。**

- Windows 的 **DIB 位图**默认是**自下而上（bottom-up）**存储的：第一行数据其实是图像的最底行（`biHeight` 为正时）。第 5 课我们讲过这个知识点。
- 我们的 `CameraFrameConverter` 已经处理过翻转（它在转换时做了逐行倒序），所以摄像头路径正常。
- `ScreenCapturer` 的 GDI 路径用 `GetDIBits` 拿到的是 bottom-up，**如果采集端没翻转，预览就是倒的**。

**有意思的推断**：如果预览**上下颠倒**但**推流画面也颠倒**，那说明采集端就是倒的（预览是忠实的）；如果**预览倒、推流正**，那说明推流链路里有翻转而预览没有——**但我们的代码里预览和推流用的是同一帧，不可能出现这种不一致**。所以：

> **结论**：在我们的架构下，"预览倒" ⟺ "推流也倒"。见到倒的画面，去查**采集器**（`ScreenCapturer::gdiCapture` 有没有翻转），不要去改 `PreviewWidget`。

**解法（在采集端，或在 `scaleToTarget` 之后统一翻）：**
```cpp
// 手动翻转（行倒序）
std::vector<uint8_t> flipped(bgra.size());
const size_t row = w * 4;
for (int y = 0; y < h; ++y)
    memcpy(&flipped[(h-1-y)*row], &bgra[y*row], row);
```
或者更省事：在 `paintEvent` 里用 `QPainter` 变换（`painter.scale(1, -1)` / `drawImage` 传翻转的矩形）——但**不要这么做**！预览要忠实反映"送进编码器的是什么"，在预览上做翻转就破坏了"所见即所推"。

#### ❌ 症状 5：画面是静止的（不动）

**排查顺序：**
1. 是不是**真的在推**？看统计里"视频 XXX 帧"的数字有没有在涨。不涨 → 采集有问题，不是预览的问题。
2. `onPreviewFrame` 被调用了吗？在开头加 `qDebug("preview frame");`。
3. 是不是被节流全丢了？把 66 改成 0 试试。
4. 是不是 `invokeMethod` 投出去了但 GUI 线程没执行？看 GUI 线程是不是被别的东西卡住了（比如在 `paintEvent` 里死循环）。
5. `update()` 被调用了吗？在 `setImage` 里加日志。

#### ❌ 症状 6：预览和实推不一致

| 现象 | 可能原因 |
|---|---|
| 预览清晰、推流糊 | **正常**！预览是编码前的原始帧，推流经过 H.264 压缩（尤其码率设低时）。见 10.3.1 的提醒 |
| 预览内容全、推流被裁切 | 分叉点位置不对（预览用了原始帧，推流用了缩放帧）。查 `videoLoop` 顺序 |
| 预览有画面、推流黑屏 | 采集正常，编码器/输出有问题。看日志 |
| 推流有画面、预览黑 | alpha 问题（症状 1）或者 `clear()` 被误调 |

### 11.2 性能类问题

#### ❌ 症状 7：界面卡顿、点按钮有延迟、日志刷新变慢

**原因：GUI 线程被预览绘制占满了。**

**排查：**
1. 打开任务管理器，看 CPU 是不是很高；
2. 临时把 `onPreviewFrame` 里的 `return` 提前（直接 `return;`，什么都不做），如果界面立刻流畅 → 确认是预览的锅。

**解法（按优先级）：**
1. **加大节流间隔**：66 → 100 或 150（降到 10fps / 6.7fps）；
2. **降低预览分辨率**：不要直接把 1920×1080 丢给 `drawImage`，先用 `sws_scale` 缩到 640×360 再画（FFmpeg 有 SIMD 优化，比 Qt 快）；
3. **加 `WA_OpaquePaintEvent`**：省一次背景擦除；
4. **窗口不可见时不画**：
   ```cpp
   if (!isVisible()) return;   // 在 setImage 里
   ```
5. 极端情况：改用 `QOpenGLWidget` 走 GPU。

#### ❌ 症状 8：推流掉帧（统计里"丢 N"在涨）

**注意：这可能正是预览造成的！**

如果预览回调太慢（比如你在里面做了同步绘制、存文件、或者加了 sleep），采集线程被拖慢 → 帧间隔变大 → 管线自己的丢帧逻辑（`pts <= lastPts`）开始丢帧。

**排查：** 临时注释掉 `setPreviewCallback(...)` 那一行，看丢帧是否消失。

**解法：** 保证 `onPreviewFrame` 只做"验-限-钉-包-投"五步，别加任何耗时操作。

#### ❌ 症状 9：内存持续上涨

**可能原因：**
1. **清理回调没生效** → 每帧的 `VideoFrame` 都泄漏。检查 `QImage` 构造的第 6、7 个参数是不是传了。
2. **事件队列堆积** → 投递速度 > GUI 处理速度，`QImage` 副本在队列里越积越多。这其实是限流没做好的表现（加大间隔）。
3. 采集端本身的问题（不在本讲范围）。

**排查：** 用 10.3.5 的清理计数实验，看清理次数是否跟上了显示次数。

### 11.3 崩溃类问题

#### ❌ 症状 10：程序随机崩溃（尤其在停止推流 / 关窗口时）

**最可能的原因：悬垂 `this`。**

回顾 8.3 节：回调捕获了裸 `this`，被管线持有，在采集线程调用。如果 `MainWindow` 先销毁而采集线程还在跑 → 崩溃。

虽然 `closeEvent` 里做了"先 stop 再 join"，但**如果你没走 `closeEvent`**（比如程序异常路径、或者 `MainWindow` 是栈对象且提前析构），就有风险。

**解法（推荐）：用 `QPointer` 守卫：**
```cpp
#include <QPointer>
...
QPointer<MainWindow> self(this);
pipeline_.setPreviewCallback([self](const VideoFramePtr& frame) {
    if (self) self->onPreviewFrame(frame);
});
```

#### ❌ 症状 11：`QPainter::begin: Widget painting can only begin as a result of a paintEvent`

**原因：在非 `paintEvent` 的地方创建了 `QPainter(this)`。**

常见触发：有人"好心"地在 `setImage` 里直接画：
```cpp
void PreviewWidget::setImage(const QImage& image) {
    image_ = image;
    QPainter p(this);     // ❌ 错！
    p.drawImage(...);
}
```

**解法：** `setImage` 里只赋值 + `update()`，绘制全交给 `paintEvent`。

#### ❌ 症状 12：从采集线程直接调 `setImage` 导致崩溃

**原因：** 跨线程碰 widget。

**解法：** 必须走 `QMetaObject::invokeMethod(..., Qt::QueuedConnection)`。

#### ❌ 症状 13：`delete` 时崩溃（清理回调里）

**原因：** `static_cast<VideoFramePtr*>(info)` 转回来的指针不对。

可能触发：
- 你传的第 7 个参数不是 `new` 出来的 `VideoFramePtr*`；
- 或者同一块内存被清理了两次（比如 `QImage` 被复制时，你误以为每次复制都要 delete——**不会**，Qt 只在引用归零时调一次）。

**排查：** 确认 `holder` 是 `new` 出来的，且**只交给一个 `QImage`**。

### 11.4 界面类问题

#### ❌ 症状 14：预览区不随窗口变大

**原因：** 少了 `setSizePolicy(QSizePolicy::Expanding, ...)` 或 stretch 系数。

**解法：** 检查 6.2 节的 `setSizePolicy` 和 8.2.4 节的 `addWidget(previewWidget_, 3)`。

#### ❌ 症状 15：中文乱码

**原因：** 源文件编码与编译器解释不一致。

**解法：** 确认顶层 CMake 有 `add_compile_options(/utf-8)`；确认源文件存为 **UTF-8（不带 BOM 也行，MSVC 认 `/utf-8`）**。

#### ❌ 症状 16：程序启动没反应，也看不到任何输出

**原因：** `WIN32_EXECUTABLE TRUE` 把控制台输出吞了。

**解法：** 临时把那行删掉或改成 `FALSE`；或者用 `OutputDebugString` + DebugView。

### 11.5 排错速查表（打印贴墙上）

| 现象 | 第一怀疑对象 | 一行验证 |
|---|---|---|
| 全黑 | alpha = 0 | 打印 `bgra[3]` |
| 发蓝 | 通道顺序 | 改用 `Format_RGB32` 试试 |
| 斜条纹 | stride | 打印 `bgra.size()` vs `w*h*4` |
| 上下颠倒 | 采集端扫描方向 | 打印第一行像素，和画面底部对比 |
| 界面卡 | GUI 线程被绘制占满 | 把 66 改成 200 |
| 推流掉帧 | 回调太慢 | 注释掉 `setPreviewCallback` |
| 内存涨 | 清理回调没生效 | 数清理次数 |
| 随机崩 | 悬垂 `this` | 改用 `QPointer` |
| 没输出 | `WIN32_EXECUTABLE` | 临时去掉 |

---

## 第十二章：小结、思考题与下一课预告

### 12.1 本讲必记的 12 条

同学们，这一讲只有 315 行 diff，但我希望你们记住以下 12 条。这 12 条在你们以后做任何"实时画面"相关的功能时都用得上。

1. **预览的业务意义**：把反馈闭环从远端平台拉回本机，把延迟从秒级压到百毫秒级。**用户不看日志，用户看画面。**

2. **预览不是"把帧丢给界面"**：每一帧要经过保活、包装、投递、缩放、合成、上屏六道工序，每道都是 O(W×H)。

3. **零拷贝的关键**：用 `QImage` 的"外部缓冲 + 清理回调"构造，只填元数据不复制像素。**配 `new` 出来的 `shared_ptr` 持有者，让帧活到画完为止。**

4. **BGRA ↔ `Format_ARGB32`**：Qt 的 Format 名字说的是"32 位整数的写法"，在小端机上 `Format_ARGB32` 的内存字节序列恰好是 B,G,R,A。**大端机上要用 `Format_BGRA8888`。**

5. **stride（步幅）**：`bytesPerLine` 是"一行占多少字节"。紧密排列时 = `width × 4`。**填错 → 斜条纹花屏。**

6. **alpha 陷阱**：`Format_ARGB32` 会读第 4 个字节。如果采集端没填 alpha（GDI 路径常见），预览会全黑而推流正常。**`Format_RGB32` 可免疫。**

7. **限流必须做，且要在生产者侧做**：被丢的帧连 `QImage` 都不构造，成本最低。**66ms → 15fps（30fps 隔一帧丢一帧）。**

8. **节流时间戳用 `steady_clock`**，绝不用 `system_clock`（会被改系统时间搞坏）。

9. **跨线程投递帧的三要素**：值捕获隐式共享的 `QImage`（不复制像素）+ `Qt::QueuedConnection`（排队到 GUI 线程）+ 内存生命周期由清理回调兜底。

10. **限流的本质是保护采集线程**：采集线程是推流的心脏，**任何阻塞它的代码都是犯罪**。回调必须 O(1)。

11. **分叉点的位置决定了预览的可信度**：在 `scaleToTarget` 之后、编码之前 → 所见即所推。但**预览清晰 ≠ 推流清晰**（编码会压缩）。

12. **架构分层**：`StreamPipeline` 不认识 Qt（只用 `std::function`），`PreviewWidget` 不认识推流（只收 `QImage`），`MainWindow` 是唯一的翻译官。**单一职责 + 依赖倒置。**

### 12.2 三张速查表

**表 A：本讲新增/修改的 API**

| API | 位置 | 作用 |
|---|---|---|
| `PreviewCallback` | `StreamPipeline.h` | `std::function<void(const VideoFramePtr&)>` 类型别名 |
| `StreamPipeline::setPreviewCallback(cb)` | `StreamPipeline.h/.cpp` | 注册/注销（`nullptr`）预览回调，线程安全 |
| `PreviewWidget(QWidget*)` | `PreviewWidget.h/.cpp` | 预览画布控件 |
| `PreviewWidget::setImage(QImage)` | 同上 | 设置当前帧（GUI 线程调用） |
| `PreviewWidget::clear()` | 同上 | 清空，回到占位态 |
| `PreviewWidget::paintEvent()` | 同上 | 保比例 + 居中 + 平滑绘制 |
| `MainWindow::onPreviewFrame(frame)` | `MainWindow.h/.cpp` | 桥接：验-限-钉-包-投 |

**表 B：Qt 知识点速查**

| 知识点 | 一句话 |
|---|---|
| `QImage` vs `QPixmap` | 改像素用 `QImage`，反复绘制用 `QPixmap` |
| 隐式共享（COW） | 拷贝只加引用计数，写时才复制 |
| `update()` vs `repaint()` | 异步请求 vs 同步立即；99% 用 `update()` |
| `paintEvent` | 唯一合法的绘制时机；`override` 必写 |
| `QSizePolicy::Expanding` | "我要抢多余空间" |
| `addStretch()` | 弹簧：顶格/居中/推开 |
| `QSize::scaled(size, KeepAspectRatio)` | 保比例算目标尺寸 |
| `QRect(QPoint topLeft, QSize size)` | 由左上角+尺寸构造矩形 |
| `SmoothPixmapTransform` | 双线性平滑缩放（vs 最近邻锯齿） |
| `QMetaObject::invokeMethod(obj, λ, QueuedConnection)` | 跨线程投递到对象所在线程 |
| `QPointer<T>` | `QObject` 的弱指针，对象销毁后自动为 null |
| `setContentsMargins(0,0,0,0)` | 容器 widget 清边距 |
| `WIN32_EXECUTABLE` | `/SUBSYSTEM:WINDOWS`，不弹控制台 |

**表 C：C++ 知识点速查**

| 知识点 | 一句话 |
|---|---|
| `std::function` | 类型擦除的可调用对象包装器；`if (f)` 判空 |
| 按值传参 + `std::move` | 支持左值拷贝、右值移动的最优参数写法 |
| `lock_guard` 作用域控制 | "拷出来再放锁"：临界区越短越好 |
| `std::chrono::steady_clock` | 单调时钟，测间隔的唯一正确选择 |
| 无捕获 lambda → 函数指针 | C 风格回调（如 `QImageCleanupFunction`）的硬约束 |
| `new` + `void*` + 回调 `delete` | C++ 对象交给 C API 托管的经典套路 |
| `override` / `explicit` | 编译期防错，必写 |
| 前向声明 vs 完整包含 | 指针成员可前向声明，值成员必须包含 |

### 12.3 本讲代码的 5 处可改进点（诚实评价）

好代码也值得批判。同学们要学会**用审视的眼光读代码**：

| # | 位置 | 现状 | 建议 |
|---|---|---|---|
| 1 | `MainWindow.h:165` | `QWidget* previewWidget_`，导致 3 处 `static_cast` | 改成 `PreviewWidget* previewWidget_`，省掉所有 cast |
| 2 | `MainWindow.h:166` | `lastPreviewTime_` 是非原子成员，被采集线程写 | 改成 `std::atomic<int64_t> lastPreviewMs_` |
| 3 | `MainWindow.cpp:onPreviewFrame` | `new VideoFramePtr` 后若 `QImage` 构造失败会泄漏 | 用 `unique_ptr` 先接管，`release()` 交给 `QImage` |
| 4 | `MainWindow.cpp:startStreaming` | 回调捕获裸 `this` | 用 `QPointer<MainWindow>` 守卫 |
| 5 | `PreviewWidget.cpp` | 总是涂满整个控件却没声明 | 加 `setAttribute(Qt::WA_OpaquePaintEvent, true)` 省一次擦除 |

另外还有两个"架构级"的问题：

- **`VideoFrame` 没有 `stride` 字段**：我们**假设**了缓冲区紧密排列。今天成立，但将来接入有行对齐的采集源（很多硬件/解码器输出都有）就会花屏。**建议在契约层补一个 `int stride = 0;`（0 表示 = width*4）。**
- **预览没有"自适应分辨率"**：现在 1920×1080 采集时，`drawImage` 要缩 207 万像素。更好的做法是先用 `sws_scale`（FFmpeg，带 SIMD）缩到预览尺寸，再交给 Qt 画。

### 12.4 思考题（8 道，第 8 道是综合题）

**Q1（概念）**：为什么 `QImage::Format_ARGB32` 在小端机器上的内存布局是 B,G,R,A？如果我们把程序移植到一台大端机器（比如某些 MIPS/老 PowerPC），代码要改哪一行？改成什么？

**Q2（概念）**：`QImage` 和 `QPixmap` 的区别是什么？本讲为什么选 `QImage`？如果预览的画面**长时间不变**（比如推一张静态图片），选哪个更合适？为什么？

**Q3（代码）**：`paintEvent` 里 `image_.size().scaled(size(), Qt::KeepAspectRatio)`：假设 `image_` 是 1920×1080，控件是 800×800（正方形），请算出 `scaled` 的值和 `target` 矩形的左上角坐标。

**Q4（性能）**：当前实现中，被节流丢弃的帧，其成本是"一次时钟读取 + 一次比较"。如果改用"消费者侧 `QTimer` 拉取"方案（1.7 节方案 B），被丢弃的帧的成本是多少？两种方案在"采集 60fps、预览只要 10fps"这个极端场景下，谁更省？为什么？

**Q5（并发）**：`lastPreviewTime_` 是普通成员，被采集线程读写。
   (a) 为什么实践中不会出问题？
   (b) 严格按 C++ 标准，它是什么问题？
   (c) 请写出三种改法（提示：`std::atomic`、存毫秒整数、把节流逻辑移到 GUI 线程侧）。

**Q6（安全）**：回调捕获裸 `this`。请用 `QPointer` 改写 `startStreaming()` 里那三行，并说明 `QPointer` 为什么能防止崩溃（提示：查一下 `QPointer` 是怎么知道对象被销毁的）。

**Q7（设计）**：`StreamPipeline` 里 `previewMutex` 保护 `previewCallback`。请回答：
   (a) 为什么 `videoLoop` 里要先把回调**拷到局部变量**再调用，而不是持锁直接调用？
   (b) 如果持锁调用，最坏会发生什么？
   (c) 如果用户在预览控件上点右键→"关闭预览"，这个操作会不会和采集线程死锁？为什么？

**Q8（综合题，认真做）**：现在请你给预览加上"**显示实时预览帧率**"功能：在预览画面左上角用半透明黑底 + 白字显示 "预览 15 fps"（每 500ms 更新一次显示的数字）。要求：
   - 不能影响推流性能；
   - 不能引入新的线程安全问题；
   - 帧率的计算要基于"实际投递的帧数 / 实际经过的时间"；
   - 说清楚你会在**哪几个文件的哪几个函数**里加代码，以及为什么放在那里。
   （提示：计数放在谁身上？`PreviewWidget` 还是 `MainWindow`？为什么？）

### 12.5 下一课预告

我们这个项目现在已经"能采、能编、能推、能看"了。但你把软件关掉再打开，会发现——**所有配置都没了**：分辨率回到默认、推流地址空空如也。

下下个提交（`c2b8021`「断线重连」、`7eb889b`「配置持久化」）会解决这些问题。按仓库的提交顺序，我们下一讲会先讲**断线重连**：

- 网络断了，怎么检测？（`writePacket` 失败 / `AVERROR_EOF` / 心跳超时）
- 重连时要不要重新发 SPS/PPS？（**要！** 第 10 课思考题里埋过这个伏笔）
- 重连的"退避策略"怎么做？（立刻重试？等 1 秒？指数退避？）
- 重连期间采集到的数据往哪儿放？

而"配置持久化"一课则要回答另外一串问题：

- 用什么存？（`QSettings` / INI / JSON / 注册表）
- 什么时候存？（关窗口时？每次改动时？）
- 存哪些？（推流地址是敏感信息，要不要加密？）

同学们可以先想想：**如果是你，会把 `StreamConfig` 序列化成什么格式？为什么？**

好，第 13 讲到此结束。

---

## 附：本讲「五章对照索引」

> **用法说明**：这张表把本讲的全部内容压缩成**五个维度**的对照，方便你复习、检索、以及跟前几课建立联系。横向五行是本讲的**五个部分**，纵向五列是**五个维度**。

| 本讲部分 | 覆盖章节 | 核心知识点 | diff 中的文件与行号 | 怎么验证 / 上哪去练 |
|---|---|---|---|---|
| **① 预习：概念与原理** | 第 1 章（1.1~1.10） | 预览的业务意义与反馈闭环；六道工序的 O(W×H) 代价；`QImage` vs `QPixmap`；`Format_ARGB32` 在小端机 = BGRA 字节序；stride/步幅；alpha 陷阱；`QImage` 外部缓冲构造 + 清理回调 + 隐式共享；`QLabel`+`setPixmap` vs 自定义 `paintEvent`；`update()`/`repaint()`/`paintEvent`/双缓冲；三种限流方案与 66ms 算术；跨线程投递三要素；CPU 占用量化 | 概念章，对应实现见 diff 第 128~130 行（`QImage` 构造三参数）、第 123~125 行（66ms 节流）、第 133~139 行（`invokeMethod`） | **实验 10.3.1**（构图对照）、**10.3.3**（改 66 看 fps）、**10.4 表**第 1/2/3/4/5 项（改 Format / stride / KeepAspectRatio / RenderHint 看现象） |
| **② 实现：模块本体** | 第 3~6 章 | `PreviewCallback` 类型别名与"只写接口"；`Impl` 加 `previewMutex` + `previewCallback`；`videoLoop` 分叉点（缩放后、编码前）与"拷出来再放锁"；`setPreviewCallback` 加锁 `std::move`、`nullptr` 注销；`PreviewWidget.h` 18 行最小接口（无 `Q_OBJECT`、`override`、隐式共享值成员）；`PreviewWidget.cpp` 45 行（`setMinimumSize(320,180)`、`Expanding` 策略、`setImage`/`clear`、`paintEvent` 七步） | `StreamPipeline.h` diff 第 298/302/311 行；`StreamPipeline.cpp` diff 第 254~256 行（成员）、第 264~272 行（分叉）、第 286~292 行（setter）；`PreviewWidget.h` diff 第 226~245 行（全 18 行）；`PreviewWidget.cpp` diff 第 176~221 行（全 45 行） | 编译 `LightLiveStreamer`；拉窗口验证最小尺寸与 3:2 比例；**实验 10.3.2**（改分辨率看预览是否跟着糊 → 验证分叉点位置） |
| **③ 验证：桥接与接线** | 第 7~9 章 | `MainWindow.h` 加 `<chrono>`/`onPreviewFrame`/`previewWidget_`/`lastPreviewTime_`；`buildUi` 布局重构（`configPanel` 中间容器 + `addStretch()` + `topLayout` 3:2 + `mainLayout` 3:1）；`startStreaming` 注册（`start()` 之前、锁内）；`stopStreaming` 注销 + 排队 `clear()`；**`onPreviewFrame` 五步（验-限-钉-包-投）**；CMake 挂两个新文件 + `WIN32_EXECUTABLE` | `MainWindow.h` diff 第 153/161/165/166 行；`MainWindow.cpp` diff 第 32~33 行（include）、第 41~42 行（resize）、第 53~59 行（预览+configPanel）、第 71~81 行（布局组装）、第 89~91 行（注册）、第 99~109 行（注销+clear）、第 117~140 行（`onPreviewFrame` 全 24 行）；`CMakeLists.txt` diff 第 13~14、24 行 | **实验 10.2**（启动看"未开始推流"、拉窗口看布局）、**10.3.4**（停止后是否清空）、**10.3.5**（清理回调计数验证无泄漏）；关掉 `WIN32_EXECUTABLE` 看控制台输出是否回来 |
| **④ 实践：动手与排错** | 第 10~11 章 | 10 个动手实验（含构图对照、改分辨率、fps 实测、零拷贝计数、10 个破坏性改造）；16 条排错条目（画面类 6 条：黑屏/发蓝/花屏/颠倒/静止/不一致；性能类 3 条：卡顿/掉帧/内存涨；崩溃类 4 条：悬垂 `this`/非法 `QPainter`/跨线程/重复 `delete`；界面类 3 条：不长大/乱码/无输出）；9 行排错速查表 | 不新增代码，全部是对 ②③ 部分代码的**修改与观察** | 按 10.2~10.4 逐个做实验；出问题时按 11.1~11.5 的清单逐条排查；把 11.5 的速查表打印贴墙 |
| **⑤ 沉淀：总结与延伸** | 第 12 章 | 12 条必记知识点；3 张速查表（新增 API / Qt / C++）；5 处可改进点（基类指针成员、非原子时间戳、`new` 异常安全、裸 `this`、缺 `WA_OpaquePaintEvent`）；2 个架构级缺陷（`VideoFrame` 无 `stride`、预览未预缩放）；8 道思考题（含 Q8 综合题：实现预览帧率角标） | 对全讲 315 行 diff 的回顾与批判；5 处改进点分别位于 `MainWindow.h` 第 165/166 行、`MainWindow.cpp` `onPreviewFrame` 第 127 行、`startStreaming` 第 89~91 行、`PreviewWidget.cpp` 构造函数 | 做完 8 道思考题（尤其 Q8 综合题）；预习下一讲 `c2b8021`（断线重连）与 `7eb889b`（配置持久化） |

### 附：本讲与前面几课的衔接点（复习时对照）

| 本讲用到的东西 | 在哪一课学的 | 本讲怎么用的 |
|---|---|---|
| `VideoFrame { width, height, pts, bgra }` / `VideoFramePtr` | **第 3 课**（`a4453bb1`） | 全讲的基础：`QImage` 直接引用 `frame->bgra.data()`，`shared_ptr` 负责续命 |
| `LogCallback` = `std::function<void(const std::string&)>` | **第 2 课**（`df3603c8`） | 本讲 `PreviewCallback` 是它的"同构兄弟"——同样的回调注册模式，只是参数换成了视频帧 |
| 屏幕采集 DXGI/GDI 双路径 + BGRA 输出 | **第 4 课**（`81eaa1f5`） | 预览的数据来源；**GDI 路径 alpha 可能为 0 → 预览全黑**（11.1 症状 1） |
| DIB 行对齐公式、`biHeight` 正负号 = 扫描方向 | **第 5 课**（`281a22be`） | stride 概念的来源；"上下颠倒"症状的排查依据 |
| `CameraFrameConverter` 里 `dst[3] = 255` | **第 6 课**（`e92cbd3`） | 摄像头路径 alpha 已填 → 预览不会黑；反衬屏幕 GDI 路径的风险 |
| PImpl 惯用法 + 三线程两队列 + `videoLoop` 结构 | **第 11 课**（`31c3f57`） | 预览分叉点插在 `videoLoop` 里；`Impl` 结构体加成员 |
| `scaleToTarget()`（`sws_scale` 缩放到目标分辨率） | **第 11 课**（`31c3f57`） | **分叉点必须在它之后** → 保证"所见即所推" |
| Qt 信号槽、`QMetaObject::invokeMethod` 跨线程 UI 更新、`QTimer` 轮询 | **第 12 课**（`3b22c69`） | 本讲第二次实战：投递 `QImage`（比字符串重得多，需配生命周期管理）；`statsTimer_` 是 `QTimer` 方案的现成参照 |
| `QElapsedTimer`（单调计时） | **第 12 课** | 与 `std::chrono::steady_clock` 互为对照：Qt 层用前者，核心/桥接层用后者 |
| "存一张图让人看"的测试纪律 | **第 0 课** 提出，第 4/5/10 课用过 | 本讲体现为**实验 10.3.1 构图对照**：预览截图 vs 拉流画面，肉眼比对 |
| `addStretch()` / 布局嵌套 / `QSizePolicy` | **第 12 课**（`buildUi`） | 8.2 节布局重构的全部基础；本次从"上下两栏"升级为"左预览右配置" |

### 附：本讲五个"可以立刻动手的重构练习"

| # | 位置 | 现状 | 改成 | 做完怎么验 |
|---|---|---|---|---|
| 1 | `MainWindow.h` 第 165 行 | `QWidget* previewWidget_`，导致 3 处 `static_cast` | `PreviewWidget* previewWidget_ = nullptr;`（需 `#include "PreviewWidget.h"` 或前向声明 `class PreviewWidget;`） | 编译通过；三处 `static_cast` 全部删掉；运行行为不变 |
| 2 | `MainWindow.h` 第 166 行 | `std::chrono::steady_clock::time_point lastPreviewTime_;`（非原子，跨线程写） | `std::atomic<int64_t> lastPreviewMs_{0};`，比较时统一换算成毫秒 | 编译通过；**实验 10.3.3** 实测 fps 仍约 15 |
| 3 | `MainWindow.cpp` `onPreviewFrame` 第 127 行 | `auto* holder = new VideoFramePtr(frame);` 后直接给 `QImage`（构造失败则泄漏） | `auto holder = std::make_unique<VideoFramePtr>(frame);`<br>`QImage image(..., holder.get());`<br>`holder.release();` | 编译通过；**实验 10.3.5** 清理计数仍然闭合 |
| 4 | `MainWindow.cpp` `startStreaming` 第 89~91 行 | lambda 捕获裸 `this` | `QPointer<MainWindow> self(this);`<br>`pipeline_.setPreviewCallback([self](const VideoFramePtr& f){ if (self) self->onPreviewFrame(f); });` | 编译通过（需 `#include <QPointer>`）；推流/停止反复 10 次不崩 |
| 5 | `PreviewWidget.cpp` 构造函数 | 每次 `paintEvent` 都涂满全控件，Qt 仍先擦一次背景 | 构造里加 `setAttribute(Qt::WA_OpaquePaintEvent, true);` | 编译通过；画面正常（**若出现残影说明哪里没涂满，需回退**）；用实验 10.3.3 对比 CPU 占用应有小幅下降 |

**做完这五项之后，重新跑一遍第 10 章的全部实验，确认行为没有变化。**

**这就是重构的正确姿势：先有可观察的行为基线（实验），再动代码，动完再验一遍。**

### 附：本讲一句总结

> **"实时预览"这四个字，一半是"实时"（15fps 足矣，人眼看不出），一半是"不花钱"（零拷贝 + 限流 + 不阻塞采集线程）。**
> **把这两半都做到，才叫做好了预览。**

---

> **老师最后的话**：同学们，这一讲的代码量是这门课到目前为止最少的一次之一（315 行）。但我想请你们回过头看看：这 315 行里，藏着**零拷贝、隐式共享、字节序、步幅、跨线程投递、生命周期接力、帧率控制、锁粒度、布局权重、单一职责**——十个硬知识点。
>
> 少的代码不等于简单的代码。**越是"看起来就该这么写"的功能，越考验你对底层机制的理解。** 一个不懂字节序的人，写完这行 `QImage` 构造会得到一张发蓝的图，然后花三天找不到原因；一个不懂隐式共享的人，会在这里加上一句"为了安全我复制一份"，然后让预览吃掉 20% 的 CPU。
>
> 我特别希望你们记住 `onPreviewFrame` 那五步——**验、限、钉、包、投**。这五个字是我给"把数据从采集线程送进界面"这件事总结的标准答案。以后你们做任何实时图像系统，都会再遇到它。
>
> 下一课我们讲断线重连，那是另一个味道的硬仗：**网络是不可靠的，而主播是不能停的。**
>
> 下课。
