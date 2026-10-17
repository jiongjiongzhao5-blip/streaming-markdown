# 第 5 讲：摄像头采集 —— CameraCapturer（DirectShow 过滤器图 + Sample Grabber 回调）

> **对应提交**：`281a22be`（2026-10-16 16:57）
> **提交信息**：CameraCapturer 实现了从摄像头抓帧，也就是 StreamConfig 里那个 SourceType::Camera 分支
> **改动规模**：6 个文件，1052 行（新增 `CameraCapturer.h/.cpp`、`tests/test_camera_capture.cpp`）
> **本讲时长建议**：约 2.5 小时（含动手实验）

---

## 开场白：为什么摄像头是"推"，屏幕是"拉"

同学们，我们前面四讲已经把屏幕采集跑通了。回忆一下 commit 4 里 `ScreenCapturer` 的用法：

```cpp
VideoFramePtr frame = capturer.capture();   // 我想要一帧，就调一次
```

我们想什么时候抓就什么时候抓，调一次函数，系统把当前桌面"递"到我们手上。屏幕画面永远躺在显存里，它不主动找我们，是我们去"拉"它。这叫**拉（pull）模型**——主动权在我们手里。

今天我们要面对的摄像头，完全是另一种脾气。

摄像头是一个**有自己节奏的硬件**。它内部的传感器以固定帧率（比如 30fps）不停地曝光、读出、吐数据，**不管你接不接，它都在往外推**。你不接，这一帧就永远地流失了——它不会等你。这叫**推（push）模型**——主动权在硬件手里，我们只能"竖起耳朵等回调"。

这一个本质差异，直接决定了今天这个类的 API 形态：

```cpp
capturer.open(0, log, CameraPixelFormat::RGB24);  // 先"打开水龙头"
CameraRawFramePtr frame = capturer.nextFrame(2000); // 然后阻塞等下一帧被"推"过来
```

请大家注意对比：

| | ScreenCapturer（屏幕） | CameraCapturer（摄像头） |
|---|---|---|
| 模型 | 拉（pull） | 推（push） |
| 取帧方式 | 同步调用 `capture()` 立即返回 | `nextFrame(timeoutMs)` 阻塞等待 |
| 帧的驱动者 | 我们（想抓才抓） | 硬件（固定帧率自己跑） |
| 需要一个"启动"动作吗 | 不需要 | 需要 `open()` 启动流水线 |
| 数据到达的线程 | 我们自己的线程 | DirectShow 内部线程，通过回调推给我们 |

理解了"推 vs 拉"，今天后面所有的设计——为什么有回调、为什么要 `open/close` 配对、为什么要一个 `pending` 槽在线程之间交接帧——全都是顺理成章的推论。

另外还要打个预防针：今天我们会一次性面对一大堆"老派 Windows 技术"——DirectShow、COM、GUID、VARIANT、宽字符。这些技术年纪比在座很多同学都大，文档又臭又长，但它们是 Windows 多媒体开发的"必修课"。本讲的目标不是让大家背 API，而是让大家看懂**这套代码是怎么在没有任何 DirectShow SDK 头文件的情况下，徒手把摄像头里的字节流接出来的**。

我们正式开始。

---

## 第一章：知识点预习

老规矩，动手读代码之前，先把今天要用到的背景知识一次性补齐。这一章内容比较多，但每一条都会在后面的代码里出现，请大家耐心。

### 1.1 DirectShow 过滤器图（Filter Graph）

DirectShow 是 Windows 上一个经典的多媒体流处理框架，上世纪 90 年代末诞生，微软后来主推的替代品叫 MediaFoundation。但直到今天，论摄像头兼容性，DirectShow 仍然是最广的——几乎任何一款 USB 摄像头驱动都会暴露 DirectShow 接口。所以我们选它，是务实，不是怀旧。

DirectShow 的核心思想特别漂亮，叫**流水线组装**：

- 数据在"**过滤器（Filter）**"之间流动。过滤器就是一个个功能块：有的负责从硬件读数据，有的负责解码，有的负责显示。
- 每个过滤器身上有若干"**引脚（Pin）**"，分输入引脚和输出引脚——就像集成电路芯片上的管脚。
- 把一个过滤器的输出引脚连到另一个过滤器的输入引脚，数据就流过去了。把一堆过滤器连起来，形成一张有向图，就叫**过滤器图（Filter Graph）**。

我们今天要搭的图非常简单，只有三段：

```
┌──────────────┐    ┌─────────────────┐    ┌───────────────┐
│  摄像头源     │ →  │  Sample Grabber │ →  │ Null Renderer │
│  Source      │    │  （取样器）      │    │ （丢弃渲染器） │
│  硬件驱动包装  │    │  回调给我们数据  │    │  图形的终点    │
└──────────────┘    └─────────────────┘    └───────────────┘
```

一个一个说：

1. **Source Filter（源过滤器）**：摄像头的 DirectShow 包装。它里面封装了驱动，负责把摄像头吐出的帧包装成 DirectShow 世界里的"样本（Sample）"往外送。它没有输入引脚，只有输出引脚——它是整张图的数据源头。

2. **Sample Grabber（取样器）**：系统自带的一个"截流"过滤器。每有一帧流过它，它就回调一次我们注册的函数，把数据指针和长度交到我们手里。**它就是我们的数据出口**，整张图是替它打工的。

3. **Null Renderer（空渲染器）**：图的终点。这里有个规矩：DirectShow 要求一张采集图必须以"渲染器"收尾——数据总得有个最终去处，否则图不允许运行。但我们根本不想显示画面，我们只要字节。Null Renderer 就是"收到数据直接扔掉"的最廉价终点。

这里有个概念要特别强调：**自动连线**。我们手写代码时，并没有亲自把三根"线"一根根插上去，而是用了一个叫 `ICaptureGraphBuilder2` 的助手对象，调它的 `RenderStream` 方法，告诉它"把摄像头的输出、经过 Sample Grabber、送到 Null Renderer"，它就自动帮我们找路径。

更妙的是：如果摄像头只出 MJPG 压缩流，而我们请求的是 RGB24 原始像素，`RenderStream` 会**自动在中间插一个 MJPG 解码器过滤器**，让两端接得上。这就是后面反复出现的"**格式协商**"——连线的过程本身就是讨价还价的过程。

### 1.2 COM：虚表布局 + GUID，所以能手写"消失的头文件"

接下来是今天最硬核、也最能长本事的一段。

DirectShow 的所有对象都是 **COM 对象**。COM（Component Object Model）是微软的二进制组件标准。大家不用怕它，它的本质就两条：

**第一条：COM 接口 = 一张虚函数表。** 一个 COM 接口，在 C++ 里就是一个纯虚基类。调用方拿到的是指向这张虚表的指针，按固定的偏移位置调函数。只要两边约定的**函数顺序、参数、调用约定**一致，代码就能跨编译器、跨语言、跨 DLL 边界互相调用。顺序就是命——虚表第 3 个槽放了什么函数，是二进制契约，错一个就全崩。

**第二条：每个接口有一张"身份证"，叫 GUID。** GUID 是一个 128 位的全局唯一标识符，长得像 `0579154a-2b53-4994-b0d0-e773148eff85`。COM 的世界里不认类名、不认头文件，只认 GUID。你想问一个对象"你支持某个接口吗"，就拿着这个接口的 GUID 去问（`QueryInterface`）；想创建某个组件，就拿着组件的 CLSID（类 ID，也是一种 GUID）去 `CoCreateInstance`。

明白了这两条，我们就能理解这段代码里最有胆量的操作了：

```cpp
struct __declspec(uuid("0579154a-2b53-4994-b0d0-e773148eff85"))
    ISampleGrabberCB : public IUnknown {
    virtual HRESULT STDMETHODCALLTYPE SampleCB(double sampleTime,
                                               IMediaSample* sample) = 0;
    virtual HRESULT STDMETHODCALLTYPE BufferCB(double sampleTime, BYTE* buffer,
                                               long bufferLen) = 0;
};
```

背景是：Sample Grabber 的接口声明本来在 DirectShow SDK 的 `qedit.h` 头文件里，但**现代 Windows SDK 把这个头文件删了**——`#include <qedit.h>` 会直接报"找不到文件"。

怎么办？作者的解法是：**手写。**

因为 COM 的本质就是"虚表布局 + GUID"，而 Sample Grabber 的 GUID 和方法列表是微软官方文档里白纸黑字写死、永远不会变的固定值。所以我们只要：

1. 用 `__declspec(uuid("..."))` 把官方 GUID 绑到这个结构体上；
2. 按官方文档的顺序，把每个虚函数原样声明出来（继承自 `IUnknown`，所以 `QueryInterface/AddRef/Release` 三个方法由基类带上）；
3. 再手写两个过滤器的 CLSID 常量：

```cpp
const GUID kClsidSampleGrabber = {
    0xC1F400A0, 0x3F08, 0x11D3, {0x9F, 0x0B, 0x00, 0x60, 0x08, 0x03, 0x9E, 0x37}};

const GUID kClsidNullRenderer = {
    0xC1F400A4, 0x3F08, 0x11D3, {0x9F, 0x0B, 0x00, 0x60, 0x08, 0x03, 0x9E, 0x37}};
```

做完这三件事，我们就可以在**完全没有 DirectShow SDK 的情况下**，用上系统自带的 Sample Grabber 和 Null Renderer。这是 COM"二进制兼容"思想的一次教科书式应用——接口不靠头文件传播，靠二进制契约传播。

请大家记住这个思想，以后遇到任何"SDK 里没有但系统里有"的 COM 组件，都可以用同样的办法徒手声明。

### 1.3 Moniker 设备枚举：怎么找到"第 0 个摄像头"

我们要打开摄像头，第一步是回答一个朴素的问题：**这台机器上有哪些摄像头？**

COM 给出的答案是一个叫 **Moniker（名字对象）** 的东西。Moniker 可以理解成"一个持久化的对象引用"——它本身不是摄像头，但它"知道怎么找到那个摄像头"，就像书签知道怎么翻到那一页。

枚举的流程是这样的：

```
CLSID_SystemDeviceEnum（系统设备枚举器）
        │ CoCreateInstance
        ▼
ICreateDevEnum
        │ CreateClassEnumerator(CLSID_VideoInputDeviceCategory)
        ▼
IEnumMoniker ──Next()──▶ IMoniker（第 0 个）──BindToStorage──▶ IPropertyBag
        │                                                        │ Read("FriendlyName")
        │──Next()──▶ IMoniker（第 1 个）                          ▼
        │                                                "USB 视频设备" 等人类可读名字
        │──Next()──▶ ...
        │
        └── 选定某个 Moniker ──BindToObject──▶ IBaseFilter（真正的摄像头过滤器）
```

两个动作要分清楚：

- **`BindToStorage`**：把 Moniker 绑定到它的"属性包（IPropertyBag）"上，从里面读元数据，比如 `FriendlyName`（友好名字）——就是设备管理器里你看到的那行字。
- **`BindToObject`**：把 Moniker 变成**真正的对象**——这一步之后，摄像头以 `IBaseFilter` 的形态正式登场，可以加进过滤器图了。

### 1.4 VARIANT 与 BSTR：COM 的"万能值"和宽字符串

读 `FriendlyName` 的时候，我们会撞上两个 COM 老物件：

**VARIANT** 是 COM 世界的"弱类型值容器"——一个带类型标签的大 union。它的 `vt` 字段告诉你里面装的是什么类型（`VT_BSTR` 表示字符串，`VT_I4` 表示 32 位整数……），然后你去对应的字段取值。用之前要 `VariantInit` 初始化，用完要 `VariantClear` 清理（如果里面有堆资源的话）。

**BSTR** 是 COM 的标准字符串类型：一个带长度前缀的宽字符（UTF-16）字符串。我们从 VARIANT 里拿到的设备名就是 BSTR，存在 `value.bstrVal` 里。

这段代码是标准套路，大家记住就行：

```cpp
VARIANT value;
VariantInit(&value);
if (SUCCEEDED(bag->Read(L"FriendlyName", &value, nullptr))
    && value.vt == VT_BSTR)                       // 先确认确实是字符串
    name = toUtf8(value.bstrVal);
VariantClear(&value);                              // 不管成不成功都要清
```

### 1.5 宽字符转 UTF-8

Windows API 清一色用 UTF-16 宽字符（`wchar_t`），而我们的项目内部统一用 UTF-8 的 `std::string`。所以需要一个转换工具：

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

注意它是**两段式**调用：第一次 `WideCharToMultiByte` 输出缓冲传 `nullptr`，目的是只问"需要多少字节"；拿到数字后开好缓冲，第二次调用才真正转换。返回的 `size` 包含末尾的空终止符，所以最后 `resize(size - 1)` 把它去掉。

这个"先问长度再转换"的两段式，是 Windows API 的祖传风格，大家以后会反复见到。

### 1.6 COM 套间（Apartment）与 RPC_E_CHANGED_MODE

用任何 COM 对象之前，线程必须先"进入 COM 世界"——调 `CoInitializeEx`。调用时要选一个**套间模型**：

- **STA（单线程套间）**：这个 COM 对象只认创建它的那个线程，别的线程调用会被自动编组（marshal）回那个线程。UI 框架（比如 Qt）通常把主线程初始化成 STA。
- **MTA（多线程套间）**：任何线程都能直接调用，COM 不管线程安全，全靠你自己。采集类代码通常选 MTA。

问题来了：如果我们的 `open()` 跑在一个**已经被别人初始化成 STA** 的线程里（典型场景：Qt 主线程里直接调 open），`CoInitializeEx(MTA)` 会失败，返回一个专门的错误码 `RPC_E_CHANGED_MODE`——"这个线程的套间模式已经被定下来了，改不了"。

这段代码的处理非常老练：

```cpp
const HRESULT comHr = CoInitializeEx(nullptr, COINIT_MULTITHREADED);
if (comHr == RPC_E_CHANGED_MODE) {
    say("当前线程已用单线程套间(STA)初始化过 COM，直接复用");
} else if (FAILED(comHr)) {
    return fail("CoInitializeEx 失败，无法使用 DirectShow");
} else {
    impl_->comOwned = true;   // 只有初始化是我们做的，我们才负责反初始化
}
```

三层分支：

1. `RPC_E_CHANGED_MODE`：复用现有的 STA——DirectShow 大部分组件在 STA 下也能工作，不是死罪，打个日志继续走。
2. 真失败：放弃。
3. `S_OK` / `S_FALSE`：初始化成功（或是同线程同模式的重复调用），记下"COM 是我们初始化的"，`close()` 时才负责 `CoUninitialize()`。

谁初始化、谁反初始化——这个 `comOwned` 标志体现的是最基本的资源礼仪。

### 1.7 摄像头像素格式大观园

摄像头给什么格式的字节，完全取决于硬件和驱动。我们的枚举把常见的五种都列出来了：

```cpp
enum class CameraPixelFormat {
    Unknown = 0,
    RGB24,
    RGB32,
    YUY2,
    UYVY,
    MJPG,
};
```

逐个说它们长什么样：

| 格式 | 每像素字节 | 内存布局 | 说明 |
|---|---|---|---|
| **RGB24** | 3 | `B G R B G R ...` | 注意实际是 **BGR** 顺序（Windows 传统）。便宜摄像头往往不给这个格式。 |
| **RGB32** | 4 | `B G R X B G R X ...` | 第 4 字节是填充（X），对齐用。带宽大户，USB 摄像头很少原生给。 |
| **YUY2** | 2（平均） | `Y0 U Y1 V Y2 U Y3 V ...` | 打包式 YUV 4:2:2：两个相邻像素共享一组色度（U/V），各自有独立的亮度（Y）。 |
| **UYVY** | 2（平均） | `U Y0 V Y1 U Y2 V Y3 ...` | 和 YUY2 同样是 4:2:2，只是字节顺序换了。 |
| **MJPG** | 不定 | 一整张 JPEG 压缩数据 | **每帧就是一张完整的 JPEG 图片**！省 USB 带宽，高清摄像头几乎都用它，代价是下游必须解码。 |

两个关键认知：

1. **YUY2/UYVY 不是"每像素独立"的格式**。你不能像 RGB 那样"取第 (x,y) 个像素的字节"。两个像素共享色度，解读必须以 2 像素为一组。
2. **MJPG 根本没有"行跨度"的概念**。它不是裸像素矩阵，是压缩流，每帧字节数都不一样。

### 1.8 DIB 行对齐公式：为什么一行的字节数不等于宽度×像素字节数

对于 RGB24/RGB32 这类裸像素格式，直觉上"一行 = 宽度 × 每像素字节数"。但 Windows 位图（DIB，Device Independent Bitmap）有一条祖传规矩：**每行的字节数必须是 4 字节（32 位）的整数倍**，不够就用零补齐。

于是行跨度（stride）的计算公式是：

```cpp
int dibRowBytes(int width, int bitCount)
{
    return ((width * bitCount + 31) / 32) * 4;
}
```

我们来推一遍这个公式。`width * bitCount` 是这一行图像**真实占用的比特数**。要补齐到 32 比特（4 字节）的倍数：先加 31 再整除 32，得到需要多少个 32 位单元（这是"向上取整"的标准写法），最后乘 4 换回字节数。

举两个例子：

- 宽 1920、RGB24（24 位）：`1920×24 = 46080` 比特，`46080 / 32 = 1440` 整除无余，`1440 × 4 = 5760` 字节——恰好对齐，没有填充。
- 宽 1919、RGB24：`1919×24 = 46056` 比特，`46056 / 32 = 1439.25`，向上取整 1440，`1440 × 4 = 5760` 字节——比真实的 `1919×3 = 5757` 字节**多了 3 字节填充**。

这个填充是后面无数 bug 的根源：**下游转换器必须按 stride 跳行，绝不能按"宽度×像素字节数"算**，否则画面会出现斜纹错位。

### 1.9 biHeight 的正负号 = 扫描方向

Windows 位图还有一个坑：**`biHeight` 字段的正负号表示像素的扫描方向**。

- **正值**：自底向上存储——字节流里第一行是画面的**最下面一行**，最后一行是画面的最上面一行。图像是"倒着"存的。这是 DIB 的历史传统，DirectShow 摄像头给 RGB 格式时几乎总是这种。
- **负值**：自顶向下存储——第一行就是画面最上面一行，符合现代人的直觉。

所以代码里这样解读：

```cpp
impl_->height = static_cast<int>(signedHeight < 0 ? -signedHeight : signedHeight);
impl_->flipVertical = signedHeight > 0;
```

高度取绝对值，然后把"需要垂直翻转"这个事实用 `flipVertical` 标志传给下游。我们自己不翻——采集层只管忠实记录事实，翻不翻是转换层的决定。

### 1.10 为什么回调里必须拷贝字节

最后一条预习，也是一条铁律。

Sample Grabber 回调给我们的是这样的签名：

```cpp
HRESULT BufferCB(double sampleTime, BYTE* buffer, long bufferLen);
```

这个 `buffer` 指针指向的内存**不是我们的**——它属于 DirectShow 过滤器图的内部缓冲池。回调函数一返回，DirectShow 就认为这块内存可以回收复用了，下一帧的数据很可能就覆盖上来。

所以回调里**唯一正确的事就是立刻把字节拷走**：

```cpp
frame->bytes.assign(data, data + size);   // 拷到我们自己的 vector 里
```

谁要是图省事把指针存起来"回头再看"，看到的就是一堆被覆盖的乱码。这一类 bug 还有个特点：偶尔能跑对（下一帧还没来你就读完了），偶尔花屏——最难查的那种。请大家把"回调里的指针寿命只到回调返回"刻在脑子里。

预习完毕。下面正式读代码。

---

## 第二章：头文件契约 —— `CameraCapturer.h`

老规矩，先看对外契约。头文件只有 62 行，但每一处设计都有讲究。

### 2.1 CameraRawFrame：为什么故意不是 VideoFrame

```cpp
struct CameraRawFrame {
    int width = 0;
    int height = 0;

    int stride = 0;                      // 行跨度（MJPG 时为 0，无意义）

    CameraPixelFormat format = CameraPixelFormat::Unknown;

    bool flipVertical = false;           // 需要垂直翻转吗

    int64_t pts = 0;                     // 这里只是帧序号（0,1,2...）

    std::vector<uint8_t> bytes;          // 原始字节（未解码、未转换）
};

using CameraRawFramePtr = std::shared_ptr<CameraRawFrame>;
```

这里有一个非常重要的架构决策，请大家注意：**它故意不是 commit 3 里定义的那个 `VideoFrame`**。

回忆一下，`VideoFrame` 的假设是"BGRA、顶向下、stride = 宽度×4"——那是屏幕采集链路内部统一好的"标准件"。而摄像头给的是**五花八门的原生格式**：可能是 YUY2，可能是 MJPG，可能倒着存，可能带行填充。

如果我们在采集层就强行把它塞进 `VideoFrame`，等于在采集层就承担了格式转换的责任——而转换需要解码器、需要色彩空间换算，那是另一个模块该干的事。

所以 `CameraRawFrame` 的设计哲学是"**忠实保存协商出来的原始事实**"：

- `format`：实际协商出的像素格式（注意，是"实际拿到的"，不是"请求的"）；
- `stride`：实际的行跨度（可能带填充）；
- `flipVertical`：实际是否需要翻转；
- `bytes`：原样字节，未解码、未转换。

commit 信息里和日志里多次出现"**下一轮转换器**"这个说法——作者很明确地把格式转换的职责留给了下一个模块。这是诚实的分层：**采集层只保证"字节原样拿到"，解读是转换层的事。**每一层只对自己的事实负责。

还有一个细节：`pts` 字段注释里明说"这里只是帧序号"。它不是时间戳，是 0、1、2、3……的计数。这个先记下来，坑点环节我们要专门批判它。

### 2.2 类的接口面

```cpp
class CameraCapturer {
public:
    CameraCapturer();
    ~CameraCapturer();

    CameraCapturer(const CameraCapturer&) = delete;
    CameraCapturer& operator=(const CameraCapturer&) = delete;

    static std::vector<std::string> listDevices();

    bool open(int deviceIndex, LogCallback log,
              CameraPixelFormat requestFormat = CameraPixelFormat::RGB24);

    void close();

    CameraRawFramePtr nextFrame(int timeoutMs);

    int width() const;
    int height() const;
    CameraPixelFormat pixelFormat() const;

private:
    struct Impl;
    std::unique_ptr<Impl> impl_;
};
```

走一遍每个成员的语义：

- **`listDevices()`**：静态方法，枚举系统里所有摄像头的友好名字。静态是因为它不需要任何实例状态——还没 open 就能查。UI 的设备下拉框就靠它填。
- **`open(deviceIndex, log, requestFormat)`**：打开第 `deviceIndex` 个摄像头，请求 `requestFormat` 格式（默认 RGB24）。注意"请求"二字——拿不拿得到另说。`log` 是一个回调，采集器把过程日志通过它报给调用方。
- **`nextFrame(timeoutMs)`**：阻塞等下一帧，最多等 `timeoutMs` 毫秒；超时返回空指针。这就是"推模型"的消费端形态。
- **`close()`**：停止采集、释放全部资源。幂等——可以反复调。
- **三个 getter**：读协商结果（宽、高、格式）。
- **PIMPL + 禁拷贝**：和前面几个模块一样的手法——实现细节藏进 `Impl`，头文件不暴露任何 DirectShow/COM 类型，拷贝语义禁止（一个摄像头对象没有"复制一份"的合理含义）。

接口面非常干净：5 个动词 + 3 个名词。好的硬件封装就该长这样——调用方完全不需要知道 DirectShow 的存在。

---

## 第三章：工具函数群 —— 匿名命名空间里的螺丝钉

`.cpp` 文件开头是一个匿名命名空间，里面装着本模块的全部"基础设施"。我们一个个看。

### 3.1 手写的 COM 接口声明

预习里已经讲过原理，这里再看一眼全貌：

```cpp
struct __declspec(uuid("0579154a-2b53-4994-b0d0-e773148eff85"))
    ISampleGrabberCB : public IUnknown {
    virtual HRESULT STDMETHODCALLTYPE SampleCB(double sampleTime,
                                               IMediaSample* sample) = 0;
    virtual HRESULT STDMETHODCALLTYPE BufferCB(double sampleTime, BYTE* buffer,
                                               long bufferLen) = 0;
};

struct __declspec(uuid("6b652fff-11fe-4fce-92ad-0266b5d7c78f"))
    ISampleGrabber : public IUnknown {
    virtual HRESULT STDMETHODCALLTYPE SetOneShot(BOOL oneShot) = 0;
    virtual HRESULT STDMETHODCALLTYPE SetMediaType(const AM_MEDIA_TYPE* type) = 0;
    virtual HRESULT STDMETHODCALLTYPE GetConnectedMediaType(AM_MEDIA_TYPE* type) = 0;
    virtual HRESULT STDMETHODCALLTYPE SetBufferSamples(BOOL bufferThem) = 0;
    virtual HRESULT STDMETHODCALLTYPE GetCurrentBuffer(long* bufferSize,
                                                       long* buffer) = 0;
    virtual HRESULT STDMETHODCALLTYPE GetCurrentSample(IMediaSample** sample) = 0;
    virtual HRESULT STDMETHODCALLTYPE SetCallback(ISampleGrabberCB* callback,
                                                  long whichMethodToCallback) = 0;
};
```

请大家注意两点：

1. **`ISampleGrabber` 声明了 7 个方法，但我们只会用到其中 4 个**（`SetMediaType`、`GetConnectedMediaType`、`SetOneShot`、`SetBufferSamples`、`SetCallback`）。不用的也必须原样声明——因为虚表的槽位要占住。少声明一个，后面所有方法的偏移全错，调用就会跳到莫名其妙的地址。
2. `STDMETHODCALLTYPE`（即 `__stdcall`）是 COM 的调用约定，必须一字不差。

### 3.2 safeRelease 与 toUtf8

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

COM 对象的析构方式就是调 `Release()`（引用计数减一，归零时对象自毁）。这个小模板把"判空 + Release + 置空"三步打包，`close()` 里要连用七次。

`toUtf8` 在预习 1.5 节讲过了，不再重复。注意它处理了两种空：空指针返回空串；只有空终止符（`size <= 1`）也返回空串。

### 3.3 guidToString：把 GUID 打印成人能看的样子

```cpp
std::string guidToString(const GUID& guid)
{
    wchar_t buffer[64] = {};
    StringFromGUID2(guid, buffer, 64);
    return toUtf8(buffer);
}
```

`StringFromGUID2` 把 GUID 格式化成 `{XXXXXXXX-XXXX-...}` 形式的宽字符串。什么时候用？当我们协商出一个**不认识的像素格式**时，把它的 GUID 打到日志里——排查"摄像头到底给了个啥"的时候，这行日志就是唯一线索。

### 3.4 freeMediaType：按 COM 规矩还东西

`AM_MEDIA_TYPE` 是 DirectShow 描述"媒体类型"的结构体，里面有两块需要手动释放的资源：

```cpp
void freeMediaType(AM_MEDIA_TYPE& type)
{
    if (type.cbFormat != 0) {
        CoTaskMemFree(type.pbFormat);   // 格式块是 COM 任务分配器给的
        type.cbFormat = 0;
        type.pbFormat = nullptr;
    }
    if (type.pUnk) {
        type.pUnk->Release();           // 持有者接口是 COM 引用计数
        type.pUnk = nullptr;
    }
}
```

- `pbFormat`：指向格式详情块（比如 `VIDEOINFOHEADER`），这块内存是 DirectShow 用 COM 任务分配器（task allocator）分配的，所以释放必须用配套的 `CoTaskMemFree`——**不能用 `free()` 或 `delete[]`**，分配器和释放器必须成对，否则堆损坏。
- `pUnk`：一个可选的"持有者"接口指针，是 COM 对象，用 `Release()` 还。

谁分配、用什么分配，就用什么还——COM 世界的规矩比普通 C++ 更严格。

### 3.5 像素格式 ↔ GUID 的双向映射

```cpp
CameraPixelFormat formatFromSubtype(const GUID& subtype)
{
    if (subtype == MEDIASUBTYPE_RGB24)
        return CameraPixelFormat::RGB24;
    if (subtype == MEDIASUBTYPE_RGB32 || subtype == MEDIASUBTYPE_ARGB32)
        return CameraPixelFormat::RGB32;
    if (subtype == MEDIASUBTYPE_YUY2)
        return CameraPixelFormat::YUY2;
    if (subtype == MEDIASUBTYPE_UYVY)
        return CameraPixelFormat::UYVY;
    if (subtype == MEDIASUBTYPE_MJPG)
        return CameraPixelFormat::MJPG;
    return CameraPixelFormat::Unknown;
}

const GUID* subtypeOf(CameraPixelFormat format)
{
    switch (format) {
    case CameraPixelFormat::RGB24: return &MEDIASUBTYPE_RGB24;
    case CameraPixelFormat::RGB32: return &MEDIASUBTYPE_RGB32;
    case CameraPixelFormat::YUY2:  return &MEDIASUBTYPE_YUY2;
    case CameraPixelFormat::UYVY:  return &MEDIASUBTYPE_UYVY;
    case CameraPixelFormat::MJPG:  return &MEDIASUBTYPE_MJPG;
    case CameraPixelFormat::Unknown: break;
    }
    return nullptr;
}
```

一对方向相反的翻译函数：

- `formatFromSubtype`：协商完之后，DirectShow 告诉我们 subtype GUID，我们翻译成自己的枚举。**注意 `ARGB32` 也归到 `RGB32`**——对我们而言"带不带 alpha 通道"无所谓，都是 4 字节一像素。
- `subtypeOf`：open 时把用户请求的枚举翻译回 GUID。返回的是指针，这样可以用 `nullptr` 表达"不指定格式"——这是个小巧的空值设计。

`MEDIASUBTYPE_*` 这些 GUID 常量来自 `dvdmedia.h`——这个头文件在现代 SDK 里还活着，所以可以直接用，不用手写。

### 3.6 friendlyNameOf：从 Moniker 里读出设备名

```cpp
std::string friendlyNameOf(IMoniker* moniker)
{
    IPropertyBag* bag = nullptr;
    if (!moniker
        || FAILED(moniker->BindToStorage(nullptr, nullptr, IID_PPV_ARGS(&bag))))
        return {};

    std::string name;
    VARIANT value;
    VariantInit(&value);
    if (SUCCEEDED(bag->Read(L"FriendlyName", &value, nullptr))
        && value.vt == VT_BSTR)
        name = toUtf8(value.bstrVal);
    VariantClear(&value);
    bag->Release();
    return name;
}
```

把预习 1.3 和 1.4 的知识串起来了：`BindToStorage` 拿属性包 → `Read(L"FriendlyName")` 读名字 → 确认是 `VT_BSTR` → 转 UTF-8。任何一步失败都返回空串——**容忍无名设备**，枚举时不因为一个设备读不出名字就全军覆没。

### 3.7 ComScope：COM 初始化的 RAII 包装

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

private:
    bool owned_ = false;
};
```

RAII（Resource Acquisition Is Initialization，"资源获取即初始化"）是 C++ 的看家本领：构造时获取资源，析构时自动释放，靠栈对象的确定析构期保证不泄漏。

`ComScope` 用在 `listDevices()` 里——那是个静态工具函数，进去枚举一下就出来，用它保证 COM 一定被反初始化。而 `owned_` 标志（`S_OK`/`S_FALSE` 才算"我们拥有的"）体现了和 `open()` 里 `comOwned` 一样的礼仪：不是自己的东西不乱动。

---

## 第四章：SampleGrabberCallback —— 手写 COM 回调对象的三方法

这是本讲最能体现"COM 功力"的一个类，我们单独给它一章。

### 4.1 我们要实现什么

Sample Grabber 过滤器的工作方式是：你给它一个实现了 `ISampleGrabberCB` 接口的对象，每过一帧它调你一次。也就是说，**我们要自己写一个 COM 对象**——不是用别人的，是亲手造一个。

一个最小 COM 对象必须正确实现继承自 `IUnknown` 的三个方法：`QueryInterface`、`AddRef`、`Release`。

### 4.2 QueryInterface：身份证查验

```cpp
HRESULT STDMETHODCALLTYPE QueryInterface(REFIID riid, void** ppv) override
{
    if (!ppv)
        return E_POINTER;
    if (riid == __uuidof(IUnknown) || riid == __uuidof(ISampleGrabberCB)) {
        *ppv = static_cast<ISampleGrabberCB*>(this);
        AddRef();
        return S_OK;
    }
    *ppv = nullptr;
    return E_NOINTERFACE;
}
```

COM 的规矩：任何人拿到你的对象，都有权拿着任意 GUID 问你"你支持这个接口吗"。我们的回答策略是：

- 输出参数是空指针 → `E_POINTER`（防御性检查，COM 边界处的参数校验是义务）；
- 问的是 `IUnknown` 或 `ISampleGrabberCB` → 把自己交出去，并且**按规矩 AddRef 一次**（多一个人拿着你，引用计数就要多一）；
- 问别的 → 老老实实回答 `E_NOINTERFACE`，并把输出置空。

### 4.3 AddRef / Release：只计数、不管生死

```cpp
ULONG STDMETHODCALLTYPE AddRef() override
{
    return ++refCount_;
}

ULONG STDMETHODCALLTYPE Release() override
{
    return --refCount_;
}
```

标准 COM 惯例里，`Release` 减到零应该 `delete this` 自我销毁。但这里**故意没有**！

为什么？因为这个回调对象的生命周期不归 COM 管——它是 `Impl` 结构体里的一个**值成员**：

```cpp
struct CameraCapturer::Impl {
    // ...
    SampleGrabberCallback callback;   // 栈式成员，跟着 Impl 活、跟着 Impl 死
};
```

Impl 由 `unique_ptr` 管，callback 跟着 Impl 活。COM 的引用计数在这里只是"满足协议"——让 Sample Grabber 内部那一套 AddRef/Release 调用不出错，但生死大权在 `unique_ptr` 手里。

`refCount_` 用 `std::atomic<ULONG>`，因为 AddRef/Release 可能从不同线程进来（DirectShow 内部是多线程的）。

这个"违反惯例"的设计可不可取？坑点环节我们会讨论它的风险。但至少在这里，它逻辑自洽：所有权清晰，协议完整。

### 4.4 SampleCB 与 BufferCB：两条回调通道取其一

`ISampleGrabberCB` 有两条回调通道：

- `SampleCB(sampleTime, IMediaSample*)`：给你一个 COM 样本对象，你还要再调它的方法才能拿到数据指针；
- `BufferCB(sampleTime, BYTE* buffer, long len)`：**直接给你数据指针和长度**，一步到位。

我们当然选后者：

```cpp
HRESULT STDMETHODCALLTYPE SampleCB(double, IMediaSample*) override
{
    return S_OK;   // 空实现，这条通道不用，但不能不实现（纯虚函数）
}

HRESULT STDMETHODCALLTYPE BufferCB(double, BYTE* buffer, long bufferLen) override
{
    if (handler_ && buffer && bufferLen > 0)
        handler_(buffer, static_cast<size_t>(bufferLen));
    return S_OK;
}
```

`BufferCB` 做的事只有一件：把数据转手交给 `handler_`。`handler_` 是个 `std::function`，构造时绑定：

```cpp
Impl()
    : callback([this](const uint8_t* data, size_t size) { onFrame(data, size); })
{
}
```

看明白这个链条了吗：**这个类是 COM 世界到 C++ 世界的适配器（Adapter）**。外面是 GUID、HRESULT、虚表那一套 COM 仪式，里面是一个普普通通的 `std::function`。过了这道桥，我们就可以忘掉 COM 了。

注意 `BufferCB` 的第一个参数 `sampleTime`（流时间，这一帧的时间戳）被直接丢弃了——形参连名字都没写。这也是坑点环节的素材。

---

## 第五章：Impl 结构体 —— 状态全景

读 `open()` 之前，先把 `Impl` 里的状态过一遍，这样读到每一行代码时都知道它在操作哪个变量：

```cpp
struct CameraCapturer::Impl {
    IGraphBuilder* graph = nullptr;              // 过滤器图本体
    ICaptureGraphBuilder2* graphBuilder = nullptr; // 自动连线助手
    IMediaControl* control = nullptr;            // 运行/停止控制器
    IBaseFilter* sourceFilter = nullptr;         // 摄像头源
    IBaseFilter* grabberFilter = nullptr;        // Sample Grabber（过滤器形态）
    IBaseFilter* nullRenderer = nullptr;         // Null Renderer
    ISampleGrabber* grabber = nullptr;           // Sample Grabber（接口形态）

    AM_MEDIA_TYPE connectedType = {};            // 实际协商出的媒体类型

    SampleGrabberCallback callback;              // 回调对象（值成员）

    bool comOwned = false;                       // COM 是我们初始化的吗

    int width = 0;
    int height = 0;
    int stride = 0;
    CameraPixelFormat format = CameraPixelFormat::Unknown;
    bool flipVertical = false;

    std::mutex mutex;
    std::condition_variable cv;
    CameraRawFramePtr pending;                   // "最新一帧"单槽

    std::atomic<bool> stopping{false};
    std::atomic<int64_t> frameCounter{0};

    LogCallback log;

    void onFrame(const uint8_t* data, size_t size) { /* ... */ }
};
```

三组状态：

1. **COM 对象组**：七个指针，建图和拆图的主角。注意 `grabberFilter` 和 `grabber` 指向**同一个对象的两个接口**——`IBaseFilter` 是它作为过滤器的面孔，`ISampleGrabber` 是它作为取样器的面孔。COM 里"一个对象多张面孔"是常态。
2. **协商结果组**：`width/height/stride/format/flipVertical`，open 成功后填好，onFrame 里快照进每一帧。
3. **线程交接组**：`mutex + cv + pending`，负责把帧从 DirectShow 回调线程安全地递给消费线程。

---

## 第六章：open() 全流程十步详解

`open()` 是本 commit 最重的函数，约 190 行。我们把它拆成十步，一步一讲。先给出全景地图：

```
open()
 ├─ 第0步：复位（close 清旧状态 + 存日志回调 + 重置计数器）
 ├─ 第1步：COM 初始化（兼容 RPC_E_CHANGED_MODE）
 ├─ 第2步：建图骨架（FilterGraph + CaptureGraphBuilder2 + IMediaControl）
 ├─ 第3步：枚举并定位第 deviceIndex 个摄像头（Moniker → BindToObject）
 ├─ 第4步：创建三个过滤器并加入图（Source / Grabber / NullRenderer）
 ├─ 第5步：格式请求（SetMediaType）
 ├─ 第6步：RenderStream 两级降级连线
 ├─ 第7步：读协商结果（GetConnectedMediaType → VIDEOINFOHEADER/2）
 ├─ 第8步：配置 Grabber（OneShot/BufferSamples/Callback）
 └─ 第9步：Run!
```

### 6.1 第 0 步：复位与失败处理框架

```cpp
bool CameraCapturer::open(int deviceIndex, LogCallback log,
                          CameraPixelFormat requestFormat)
{
    close();                                // 先清掉上一次的残留状态

    impl_->log = std::move(log);
    impl_->stopping = false;
    impl_->frameCounter = 0;

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

两个细节：

1. **`open()` 开头先 `close()`**：允许"没关就直接再开"，对调用方友好；同时保证重开时状态干净。这也让 `close()` 必须是幂等的。
2. **`say` / `fail` 两个 lambda**：`say` 是"有日志回调就打"；`fail` 是统一的失败出口——打日志、完整清理（close 会把建了一半的图拆掉）、返回 false。后面十几处失败分支全靠它收口，保证**任何一步失败都不留半拉子资源**。这是 RAII 之外的另一种纪律：失败路径上的资源清理和成功路径同样重要。

注意这里和 ScreenCapturer 的风格差异：屏幕采集是"降级不报错"（千方百计给你一帧），摄像头是"打不开就是真失败"——没有兜底路径，摄像头不像桌面那样永远存在。

### 6.2 第 1 步：COM 初始化

```cpp
const HRESULT comHr = CoInitializeEx(nullptr, COINIT_MULTITHREADED);
if (comHr == RPC_E_CHANGED_MODE) {
    say("当前线程已用单线程套间(STA)初始化过 COM，直接复用");
} else if (FAILED(comHr)) {
    return fail("CoInitializeEx 失败，无法使用 DirectShow");
} else {
    impl_->comOwned = true;
}
```

预习 1.6 节讲过，不再重复。请大家记住这个三分支模式，它是"在别人线程里用 COM"的标准写法。

### 6.3 第 2 步：建图骨架

```cpp
if (FAILED(CoCreateInstance(CLSID_FilterGraph, nullptr, CLSCTX_INPROC_SERVER,
                            IID_PPV_ARGS(&impl_->graph))))
    return fail("创建 FilterGraph 失败");

if (FAILED(CoCreateInstance(CLSID_CaptureGraphBuilder2, nullptr,
                            CLSCTX_INPROC_SERVER,
                            IID_PPV_ARGS(&impl_->graphBuilder))))
    return fail("创建 CaptureGraphBuilder2 失败");

impl_->graphBuilder->SetFiltergraph(impl_->graph);

if (FAILED(impl_->graph->QueryInterface(IID_PPV_ARGS(&impl_->control))))
    return fail("获取 IMediaControl 失败");
```

四个动作：

1. 创建**过滤器图本体**（`CLSID_FilterGraph` → `IGraphBuilder`）。它是容器，负责管理图里的所有过滤器和连线。
2. 创建**自动连线助手**（`CLSID_CaptureGraphBuilder2`）。它是采集场景的专用"接线工"。
3. `SetFiltergraph` 把二者绑定——告诉接线工"你服务的图是这张"。
4. 从图上 `QueryInterface` 拿 `IMediaControl`——图的"运行/停止开关"。同一张图，换张面孔就是另一种控制器。

`IID_PPV_ARGS` 是个宏，展开后给 `QueryInterface` 传 IID 和二级指针，是微软推荐的防错写法。

### 6.4 第 3 步：定位第 deviceIndex 个摄像头

```cpp
ICreateDevEnum* deviceEnumerator = nullptr;
if (FAILED(CoCreateInstance(CLSID_SystemDeviceEnum, nullptr, CLSCTX_INPROC_SERVER,
                            IID_PPV_ARGS(&deviceEnumerator))))
    return fail("创建视频设备枚举器失败");

IEnumMoniker* enumMoniker = nullptr;
if (deviceEnumerator->CreateClassEnumerator(CLSID_VideoInputDeviceCategory,
                                            &enumMoniker, 0) != S_OK
    || !enumMoniker) {
    deviceEnumerator->Release();
    return fail("系统里没有找到任何视频输入设备（摄像头）");
}

std::string deviceName;
int seen = 0;
IMoniker* moniker = nullptr;
while (enumMoniker->Next(1, &moniker, nullptr) == S_OK) {
    if (seen == deviceIndex) {
        deviceName = friendlyNameOf(moniker);
        moniker->BindToObject(nullptr, nullptr, IID_PPV_ARGS(&impl_->sourceFilter));
        moniker->Release();
        break;
    }
    moniker->Release();
    ++seen;
}
enumMoniker->Release();
deviceEnumerator->Release();

if (!impl_->sourceFilter) {
    return fail("打开第 " + std::to_string(deviceIndex)
                + " 个摄像头失败（索引越界，或被别的程序独占）");
}
```

就是预习 1.3 那张流程图的代码版。几个观察点：

- **数到第 `deviceIndex` 个才 `BindToObject`**：前面的 Moniker 看一眼就 `Release` 掉。BindToObject 是相对重的操作（真正初始化设备对象），不该对每个设备都做。
- **`BindToObject` 可能失败**：摄像头被别的程序（腾讯会议、相机应用）独占时，枚举能枚举到，但绑定拿不到对象。所以后面用 `!impl_->sourceFilter` 统一判失败，错误信息里也写明"索引越界，或被别的程序独占"——错误信息要替用户想到最常见的原因。
- **资源释放顺序**：`enumMoniker` 和 `deviceEnumerator` 用完立刻 Release，不拖到最后。COM 编程里"用完即还"比攒着一起还更安全。

```cpp
say("摄像头 #" + std::to_string(deviceIndex) + ": "
    + (deviceName.empty() ? std::string("<未知设备名>") : deviceName));
```

打一条日志告诉用户开了哪台设备。现场调试时，"我以为开的是前置摄像头，结果开的是 OBS 虚拟摄像头"这类乌龙，全靠这行日志现形。

### 6.5 第 4 步：三个过滤器进图

```cpp
if (FAILED(impl_->graph->AddFilter(impl_->sourceFilter, L"Camera Source")))
    return fail("把摄像头加进过滤器图失败");

if (FAILED(CoCreateInstance(kClsidSampleGrabber, nullptr, CLSCTX_INPROC_SERVER,
                            IID_PPV_ARGS(&impl_->grabberFilter))))
    return fail("创建 Sample Grabber 失败");

if (FAILED(impl_->grabberFilter->QueryInterface(IID_PPV_ARGS(&impl_->grabber))))
    return fail("从 Sample Grabber 上取 ISampleGrabber 接口失败");

if (FAILED(CoCreateInstance(kClsidNullRenderer, nullptr, CLSCTX_INPROC_SERVER,
                            IID_PPV_ARGS(&impl_->nullRenderer))))
    return fail("创建 Null Renderer 失败");

if (FAILED(impl_->graph->AddFilter(impl_->grabberFilter, L"Sample Grabber"))
    || FAILED(impl_->graph->AddFilter(impl_->nullRenderer, L"Null Renderer")))
    return fail("把 Sample Grabber / Null Renderer 加进过滤器图失败");
```

注意 `kClsidSampleGrabber` 和 `kClsidNullRenderer` 就是我们 1.2 节**手写的两个 CLSID 常量**——这里就是用它们的地方。

`AddFilter` 的第二个参数是过滤器的显示名（宽字符），调试用，GraphEdit 工具里能看到。

此时图里有三个"孤岛"过滤器，互不相连。下一步开始接线。

### 6.6 第 5 步：格式请求

```cpp
const GUID* wantedSubtype = subtypeOf(requestFormat);
if (wantedSubtype) {
    AM_MEDIA_TYPE request = {};
    request.majortype = MEDIATYPE_Video;
    request.subtype = *wantedSubtype;
    request.formattype = FORMAT_VideoInfo;
    impl_->grabber->SetMediaType(&request);
    say(std::string("向摄像头请求 ") + cameraPixelFormatName(requestFormat) + " ...");
} else {
    say("不指定格式，直接接受摄像头给的原生格式");
}
```

`AM_MEDIA_TYPE` 三联单：

- `majortype = MEDIATYPE_Video`：我要的是视频（不是音频）；
- `subtype = 目标像素格式`：我要 RGB24 / YUY2 / MJPG……
- `formattype = FORMAT_VideoInfo`：格式详情用 `VIDEOINFOHEADER` 结构承载。

`SetMediaType` 设到 **Grabber** 上，语义是："我（Grabber）只接受这种媒体类型的输入"。接线时，如果摄像头给不了这个格式，连线助手就会尝试在中间插转换器来满足我们。

注意这里**没有检查 SetMediaType 的返回值**——因为它只是个"愿望"，真正的裁决在连线时。愿望落空不丢人，连不上才失败。

### 6.7 第 6 步：RenderStream 两级降级连线

这是全函数策略性最强的一段：

```cpp
HRESULT connectHr = impl_->graphBuilder->RenderStream(
    &PIN_CATEGORY_CAPTURE, &MEDIATYPE_Video, impl_->sourceFilter,
    impl_->grabberFilter, impl_->nullRenderer);

if (FAILED(connectHr)) {
    connectHr = impl_->graphBuilder->RenderStream(
        &PIN_CATEGORY_PREVIEW, &MEDIATYPE_Video, impl_->sourceFilter,
        impl_->grabberFilter, impl_->nullRenderer);
}

if (FAILED(connectHr)) {
    say(std::string(cameraPixelFormatName(requestFormat))
        + " 请求没被接受，改为接受摄像头能给的原生格式");
    impl_->grabber->SetMediaType(nullptr);

    connectHr = impl_->graphBuilder->RenderStream(
        &PIN_CATEGORY_CAPTURE, &MEDIATYPE_Video, impl_->sourceFilter,
        impl_->grabberFilter, impl_->nullRenderer);
    if (FAILED(connectHr)) {
        connectHr = impl_->graphBuilder->RenderStream(
            &PIN_CATEGORY_PREVIEW, &MEDIATYPE_Video, impl_->sourceFilter,
            impl_->grabberFilter, impl_->nullRenderer);
    }
}

if (FAILED(connectHr))
    return fail("摄像头与采集器连线失败（格式协商不成功）");
```

先解释 `RenderStream` 的参数：

```cpp
RenderStream(引脚类别, 媒体类型, 源过滤器, 中间过滤器, 终点过滤器)
```

- **引脚类别**：摄像头通常有两个输出引脚——`PIN_CATEGORY_CAPTURE`（采集引脚，给录像用的高分辨率）和 `PIN_CATEGORY_PREVIEW`（预览引脚，给取景器用的低分辨率）。有些老摄像头只有其一，所以要**两级降级**：先试采集引脚，不行换预览引脚。
- 中间传 `grabberFilter`、终点传 `nullRenderer`，意思是"必须经过我的 Grabber，最后到 Null Renderer"。

降级策略整体是一棵决策树：

```
请求格式 = RGB24（举例）
        │
        ▼
RenderStream(CAPTURE) ──成功──▶ 完事
        │失败
        ▼
RenderStream(PREVIEW) ──成功──▶ 完事
        │失败
        ▼
放弃格式要求（SetMediaType(nullptr)）
        │
        ▼
RenderStream(CAPTURE) ──成功──▶ 完事（拿到原生格式）
        │失败
        ▼
RenderStream(PREVIEW) ──成功──▶ 完事
        │失败
        ▼
真失败，放弃
```

**四层尝试，两次降级维度**：先降引脚，再降格式要求。每一步都有明确的日志，用户能从日志里清楚看到"协商走到了哪一层"。

这种"逐级放宽条件重试"的模式在硬件编程里极其常见——硬件的世界充满不确定性，写代码时要像谈判一样：先提理想条件，谈不拢再让步，让到不能再让才掀桌子。

### 6.8 第 7 步：读协商结果

连线成功后，**立刻读"实际"协商出的格式**——请求的 vs 拿到的，经常不一样：

```cpp
AM_MEDIA_TYPE connected = {};
if (FAILED(impl_->grabber->GetConnectedMediaType(&connected)))
    return fail("读取实际协商格式失败");

impl_->connectedType = connected;   // 存起来，close 时统一 free

int bitCount = 0;
LONG signedHeight = 0;
if (connected.formattype == FORMAT_VideoInfo
    && connected.cbFormat >= sizeof(VIDEOINFOHEADER)) {
    const auto* vih = reinterpret_cast<const VIDEOINFOHEADER*>(connected.pbFormat);
    impl_->width = static_cast<int>(vih->bmiHeader.biWidth);
    signedHeight = vih->bmiHeader.biHeight;
    bitCount = vih->bmiHeader.biBitCount;
} else if (connected.formattype == FORMAT_VideoInfo2
           && connected.cbFormat >= sizeof(VIDEOINFOHEADER2)) {
    const auto* vih2 = reinterpret_cast<const VIDEOINFOHEADER2*>(connected.pbFormat);
    impl_->width = static_cast<int>(vih2->bmiHeader.biWidth);
    signedHeight = vih2->bmiHeader.biHeight;
    bitCount = vih2->bmiHeader.biBitCount;
} else {
    return fail("拿到一个不认识的媒体类型格式: formattype = "
                + guidToString(connected.formattype));
}
```

几个防御性细节值得学：

1. **先查 `cbFormat >= sizeof(...)` 再 reinterpret_cast**：`pbFormat` 是裸指针，里面是什么全靠 `formattype` + 长度两个线索互相印证。直接强转是缓冲区溢出的温床。
2. **VIDEOINFOHEADER 和 VIDEOINFOHEADER2 两个分支**：后者是扩展版（多了隔行扫描等字段），但开头内嵌的 `bmiHeader` 布局相同，解读方法一样。
3. **不认识的 formattype 也不硬扛**：打印 GUID 日志后干净地失败——`guidToString` 在这里派上用场。

然后是四个派生量的计算：

```cpp
impl_->height = static_cast<int>(signedHeight < 0 ? -signedHeight : signedHeight);
impl_->flipVertical = signedHeight > 0;
impl_->format = formatFromSubtype(connected.subtype);
impl_->stride = (impl_->format == CameraPixelFormat::MJPG || bitCount <= 0)
                    ? 0
                    : dibRowBytes(impl_->width, bitCount);
```

- 高度取绝对值，`biHeight > 0` 意味着自底向上，标 `flipVertical`（预习 1.9）；
- subtype GUID 翻译成枚举（3.5 节）；
- stride 用 DIB 公式算（预习 1.8），**但 MJPG 直接置 0**——压缩流没有行跨度的概念。

随后是一段信息量很足的日志：

```cpp
say("实际协商结果: " + std::to_string(impl_->width) + "x"
    + std::to_string(impl_->height) + "  " + cameraPixelFormatName(impl_->format)
    + "  " + std::to_string(bitCount) + " 位/像素  行跨度 "
    + std::to_string(impl_->stride) + " 字节"
    + (impl_->flipVertical ? "  自底向上(需要翻转)" : "  自顶向下"));
```

以及两个针对特殊格式的提醒：

```cpp
if (impl_->format == CameraPixelFormat::MJPG) {
    say("注意: MJPG 每帧是一整张 JPEG，stride 对它没有意义，下一轮转换器里需要解码");
} else if (impl_->format == CameraPixelFormat::Unknown) {
    say("警告: 这个子类型本模块还不认识 -> " + guidToString(connected.subtype)
        + "，下一轮的转换器需要补上对它的支持");
}
```

日志里两次出现"下一轮转换器"——作者在整个模块里反复给下游模块"留言"，这种跨模块的契约意识值得学习。

### 6.9 第 8 步：配置 Grabber

```cpp
impl_->grabber->SetOneShot(FALSE);
impl_->grabber->SetBufferSamples(FALSE);
impl_->grabber->SetCallback(&impl_->callback, 1);
```

三个配置项，逐个解释：

- **`SetOneShot(FALSE)`**：不要"抓一张就停"。OneShot 模式下 Grabber 抓一帧后自动把图停掉，那是截图场景用的；我们要连续流。
- **`SetBufferSamples(FALSE)`**：不要在 Grabber 内部攒缓冲。TRUE 模式下它会自己把样本复制攒起来等你主动取；FALSE 模式下每帧立即回调。我们要的是实时流，立即回调。
- **`SetCallback(&impl_->callback, 1)`**：注册我们的回调对象，第二个参数 `1` 表示用 `BufferCB` 通道（`0` 是 `SampleCB` 通道）。这就是第四章那个对象登场的位置。

### 6.10 第 9 步：Run

```cpp
if (FAILED(impl_->control->Run()))
    return fail("启动过滤器图失败（摄像头可能被其它程序占用）");

return true;
```

`IMediaControl::Run()` 一调用，整张图进入运行态。**从这一刻起，摄像头帧开始沿着流水线流动**：Source → Grabber →（回调到我们的 onFrame）→ Null Renderer 丢弃。

注意 Run 也可能失败——有些独占型摄像头在 AddFilter 时不报错，到 Run 才翻脸。所以错误信息再次提醒"可能被其它程序占用"。

至此 `open()` 十步讲完。请大家整体回顾一下：**这个函数没有任何一行是多余的**——每一步都可能失败，每个失败都有明确日志和完整清理，每个降级都有记录。这就是系统级代码该有的样子。

---

## 第七章：onFrame —— 回调线程里的"单槽最新帧"设计

图跑起来之后，数据端的入口是 `Impl::onFrame`（经 BufferCB → handler_ 链路调进来）：

```cpp
void onFrame(const uint8_t* data, size_t size)
{
    if (stopping || !data || size == 0)
        return;                                  // close 已开始就丢弃

    auto frame = std::make_shared<CameraRawFrame>();
    frame->width = width;
    frame->height = height;
    frame->stride = stride;
    frame->format = format;
    frame->flipVertical = flipVertical;
    frame->pts = frameCounter.fetch_add(1);      // 原子自增当帧序号
    frame->bytes.assign(data, data + size);      // 拷贝字节（必须拷贝！）

    {
        std::lock_guard<std::mutex> lock(mutex);
        pending = std::move(frame);              // 覆盖式写入最新帧槽
    }
    cv.notify_one();
}
```

### 7.1 逐行讲

1. **前置检查**：`stopping` 是原子标志，close 流程一开始就会被置上——正在飞行的回调尽快返回，不再往 pending 里塞新帧。
2. **快照协商参数**：width/height/stride/format/flipVertical 在 open 后不变，但把它们**拷进每一帧**而不是让消费方回头读 Impl——帧一旦离手就自包含，消费方拿到帧就拿到了解读它所需的全部事实。这是"数据自描述"的好习惯。
3. **`pts = frameCounter.fetch_add(1)`**：原子计数器当帧序号。记住：这不是时间戳。
4. **`bytes.assign(data, data + size)`**：预习 1.10 的铁律——回调里的指针寿命只到回调返回，必须拷贝。这是整条链路里**唯一一次数据拷贝**，不可省略，也仅此一次。
5. **覆盖式写入 `pending`**：注意 `pending = std::move(frame)` 不是追加，是**覆盖**——旧帧的 `shared_ptr` 被顶掉，引用计数归零自动析构。

### 7.2 "单槽最新帧"的哲学

`pending` 不是队列，是**容量为 1 的槽位**。这意味着：

- 回调来得快、消费取得慢 → 旧帧直接被新帧覆盖，**丢旧保新**；
- 消费方永远拿到的是"此刻最新的一帧"，不会拿到积压的历史帧。

这和 commit 3 里 `BoundedQueue` 的哲学一脉相承，只是容量=1 的特化。对实时预览/推流场景，这是**正确**的选择：实时性优先于完整性——观众要的是"现在"的画面，没人想看三秒前的积压帧。

线程交接的机制和 `BoundedQueue::tryPop` 同构：mutex 保护共享状态 + condition_variable 做等待唤醒。只不过这里还没接 BoundedQueue，先自管。

---

## 第八章：nextFrame 与 close —— 消费端与关停顺序

### 8.1 nextFrame：带超时的谓词等待

```cpp
CameraRawFramePtr CameraCapturer::nextFrame(int timeoutMs)
{
    std::unique_lock<std::mutex> lock(impl_->mutex);

    const bool arrived = impl_->cv.wait_for(
        lock, std::chrono::milliseconds(timeoutMs),
        [this] { return static_cast<bool>(impl_->pending); });

    if (!arrived)
        return nullptr;

    CameraRawFramePtr frame = std::move(impl_->pending);
    impl_->pending.reset();
    return frame;
}
```

标准的三件套：

1. `cv.wait_for(lock, 超时, 谓词)`——谓词 `pending != nullptr` 保证**虚假唤醒**（spurious wakeup）时被正确过滤：条件变量偶尔会在没人 notify 的情况下醒来，带谓词版本会自动再睡回去；
2. 超时返回 `nullptr`——调用方用空指针判断"这一拍没等到帧"；
3. 取走帧的同时 `pending.reset()` 清槽——下一帧来了再放。

消费方的典型写法就是一个循环：

```cpp
while (running) {
    auto frame = capturer.nextFrame(2000);
    if (!frame) continue;   // 超时，没帧
    // 处理 frame ...
}
```

### 8.2 close：顺序就是一切

```cpp
void CameraCapturer::close()
{
    impl_->stopping = true;                       // ① 让在飞的回调尽快返回

    if (impl_->control)
        impl_->control->Stop();                   // ② 停图，不再产生新回调

    if (impl_->grabber)
        impl_->grabber->SetCallback(nullptr, 0);  // ③ 摘掉回调

    {
        std::lock_guard<std::mutex> lock(impl_->mutex);
        impl_->pending.reset();                   // ④ 清掉没消费掉的帧
    }

    freeMediaType(impl_->connectedType);          // ⑤ 还媒体类型内嵌资源

    safeRelease(impl_->grabber);                  // ⑥ 逆序释放七个 COM 对象
    safeRelease(impl_->nullRenderer);
    safeRelease(impl_->grabberFilter);
    safeRelease(impl_->sourceFilter);
    safeRelease(impl_->control);
    safeRelease(impl_->graphBuilder);
    safeRelease(impl_->graph);

    impl_->width = 0;                             // ⑦ 复位字段
    impl_->height = 0;
    impl_->stride = 0;
    impl_->format = CameraPixelFormat::Unknown;
    impl_->flipVertical = false;
    impl_->frameCounter = 0;

    if (impl_->comOwned) {                        // ⑧ 自己的 COM 自己还
        CoUninitialize();
        impl_->comOwned = false;
    }
}
```

这个顺序是经过考虑的，我们体会一下：

- **先置 `stopping` 再 `Stop()`**：Stop 是同步的，它会等正在执行的回调返回。如果某个回调正在 onFrame 里慢吞吞地拷贝，stopping 标志保证后续回调进来就立即返回，缩短 Stop 的等待。
- **`Stop()` 在 `SetCallback(nullptr)` 之前**：先保证不再有新回调产生，再摘回调。顺序反了的话，摘回调和回调执行之间会有竞争窗口。
- **COM 对象逆序释放**：`grabber`（接口）在 `grabberFilter`（过滤器）之前，`control` 在 `graph` 之前——先拿到的后还，和构造顺序相反，这是资源释放的通用纪律。
- **`CoUninitialize` 放最后，且只在 `comOwned` 时**：自己的债自己还。

析构函数只调 `close()`：

```cpp
CameraCapturer::~CameraCapturer()
{
    close();
}
```

因为 close 是幂等的，用户手动 close 过再析构也不会双重释放。

---

## 第九章：测试程序教学 —— 六段测试

`tests/test_camera_capture.cpp`（387 行）是本次提交的另一半。它不是"跑通就行"的冒烟测试，而是一份**教你如何验证硬件采集代码**的教案。我们逐段讲。

### 9.0 准备工作：参数与日志

```cpp
int main(int argc, char** argv)
{
#if defined(_WIN32)
    std::system("chcp 65001 > nul");    // 控制台切到 UTF-8，中文日志不乱码
#endif

    int deviceIndex = 0;
    if (argc > 1) {
        deviceIndex = std::atoi(argv[1]);
        if (deviceIndex < 0)
            deviceIndex = 0;
    }

    CameraPixelFormat requestFormat = CameraPixelFormat::RGB24;
    if (argc > 2)
        requestFormat = parseFormat(argv[2]);
```

`parseFormat` 把命令行字符串（`rgb24`/`yuy2`/`mjpg`/`any`……）解析成枚举，不认识的名字降级成 `Unknown`（不指定格式）并打印提示——工具程序对输错的参数也保持礼貌。

开头那段横幅值得抄下来贴在工位上：

```
对着同一台摄像头换个格式名再跑一遍，就能亲眼看到
"我请求的格式"和"我拿到手的格式"往往不是同一个。
```

这句话就是动手环节的核心实验。

### 9.1 第一段：枚举摄像头

```cpp
const std::vector<std::string> devices = CameraCapturer::listDevices();
std::printf("  找到 %d 个视频输入设备：\n", ...);
for (size_t i = 0; i < devices.size(); ++i) { ... }

if (devices.empty()) {
    std::printf("\n  这个机器上没有摄像头，后面的检查跳过。\n");
    std::printf("  （这不是代码的错，插上一个 USB 摄像头再跑一次即可。）\n");
    // ...
    return g_failed == 0 ? 0 : 1;   // 正常退出，不算失败
}
```

**设计要点：没有摄像头时正常退出、不计失败。**为什么？因为这样的测试要在 CI（持续集成）服务器上跑——服务器没有摄像头是常态，硬件缺席不等于代码有错。区分"环境的错"和"代码的错"，是硬件相关测试的基本修养。

### 9.2 第二段：打开并读协商结果

```cpp
const bool ok = capturer.open(deviceIndex, log, requestFormat);
check(ok, "open() 返回成功");
std::printf("  建图耗时 %lld 毫秒\n", openMs);

if (!ok) {
    std::printf("  常见原因：摄像头被别的程序（钉钉/腾讯会议/相机应用）占用了。\n");
    return 1;
}

check(width > 0 && height > 0, "协商出了有效的分辨率");
check(format != CameraPixelFormat::Unknown, "协商出了本模块认识的一种像素格式");
```

注意失败提示里直接点名"钉钉/腾讯会议/相机应用"——**错误提示要替用户说出他接下来想问的问题**。

然后是"请求的 vs 拿到的"对比打印：

```cpp
if (requestFormat != CameraPixelFormat::Unknown && requestFormat != format) {
    std::printf("    → 请求的和拿到的不一样。DirectShow 为了让两端接上，\n");
    std::printf("      要么在中间插了个转换滤镜，要么干脆没搭理你的请求。\n");
}
```

这就是格式协商最直观的教学：你请求 RGB24，摄像头只出 MJPG，DirectShow 要么插解码器满足你，要么两手一摊给你原生格式。打印出来的对比让这个抽象机制变成肉眼可见的事实。

### 9.3 第三段：连取 30 帧

```cpp
constexpr int kFrames = 30;
constexpr int kTimeoutMs = 2000;

for (int i = 0; i < kFrames; ++i) {
    CameraRawFramePtr frame = capturer.nextFrame(kTimeoutMs);
    if (!frame) {
        ++timeouts;
        continue;
    }
    // 记录首帧；后续帧检查字节数、stride 是否和首帧一致
}

check(timeouts == 0, "连续 30 帧没有超时");
check(sizeConsistent && strideConsistent, "每帧的字节数和行跨度都保持一致");
check(firstFrame != nullptr, "至少取到了第一帧");
```

三个断言背后的学问：

- **无超时**：帧在持续流动，图活着；
- **字节数一致**：格式稳定（对 MJPG 这条其实可能不成立——JPEG 每帧大小不一，这是测试的一个小小不严谨，但对裸格式是必须成立的）；
- **估算帧率**：`1000.0 × 帧数 / 耗时`，对照摄像头标称的 30fps 看看是否合理。

### 9.4 第四段：解读字节（本测试的精华）

这一段按格式分三条路，每一条都有独立的教学价值。

**路线一：MJPG —— 存成 jpg，双击即验证**

```cpp
if (format == CameraPixelFormat::MJPG) {
    std::printf("  首帧开头 16 字节: ");
    printHexPrefix(firstFrame->bytes, 16);
    check(looksLikeJpeg(firstFrame->bytes),
          "首帧以 FF D8 开头、FF D9 结尾，确实是一张完整 JPEG");
    check(writeFile(frameBase + ".jpg", firstFrame->bytes),
          "首帧已保存为 " + frameBase + ".jpg");
    std::printf("  请直接双击打开 %s.jpg —— 能正常显示就说明字节完全没问题。\n", ...);
}
```

`looksLikeJpeg` 检查 JPEG 的**魔数（magic number）**：

```cpp
bool looksLikeJpeg(const std::vector<uint8_t>& bytes)
{
    if (bytes.size() < 4)
        return false;
    return bytes[0] == 0xFF && bytes[1] == 0xD8              // SOI：图像开始
           && bytes[bytes.size() - 2] == 0xFF
           && bytes[bytes.size() - 1] == 0xD9;               // EOI：图像结束
}
```

`FF D8`（SOI，Start Of Image）和 `FF D9`（EOI，End Of Image）是 JPEG 文件的头尾标记。但最妙的验证不是魔数，是**把字节原样存成 `.jpg` 文件让你双击打开**——系统的看图软件就是最权威的 JPEG 解码器，能显示出画面就是字节级正确的铁证。这种"借力系统工具做验证"的思路，比写一堆断言高明得多。

**路线二：裸格式（RGB24/RGB32/YUY2/UYVY）—— stride 对账**

```cpp
const int packedBytesPerRow(...);   // 理论行宽：宽度 × 每像素字节数

std::printf("  每行理论字节 %d，实际行跨度 %d\n", packed, strideOfFirst);
if (strideOfFirst == packed)
    std::printf("  → 这一路没有行填充\n");
else
    std::printf("  → 驱动做了行填充（每行多 %d 字节），下一轮转换器必须按 stride 跳行\n", ...);

check(firstFrame->bytes.size() >= expectedStride, "字节数足够装下 stride x height");
if (firstFrame->bytes.size() > expectedStride) {
    std::printf("  [注意] 驱动报告了 %zu 字节，而一帧真正需要的是 %zu 字节，多出 %zu 字节。\n", ...);
    std::printf("         多出来的那部分不是画面（实测全是零填充）。\n");
    std::printf("         下一轮转换时必须按 stride x height 读，不能把整个 bytes 都当成像素。\n");
}
```

这里揭示了**两个真实的硬件坑**：

1. **行填充**：实际 stride 可能大于理论行宽（预习 1.8 的 DIB 对齐）；
2. **尾部多余字节**：驱动给的缓冲可能比 `stride × height` 还多，多出来的是零填充。

测试程序实测发现并打印出来，等于替下一个模块（转换器）提前趟了雷：**必须按 stride × height 读，按 width × 像素字节数跳行，不能把整个 bytes 当画面**。这三条规矩不遵守，转出来的图就是花的。

**路线三：纯绿帧排障 —— 全场最实战的知识**

```cpp
const int distinctValues = countDistinctByteValues(firstFrame->bytes);
if (distinctValues <= 4) {
    std::printf("  [注意] 这一帧几乎是纯色 —— 真实画面不可能只用到这么几种字节值。\n");
    std::printf("         常见原因是：\n");
    std::printf("           - 摄像头被镜头盖/物理挡板挡住\n");
    std::printf("           - Windows 隐私设置里禁止了摄像头\n");
    std::printf("           - 摄像头被别的程序占用，驱动只往外吐空缓冲\n");
    std::printf("         （全零的 YUV 经过色彩转换后正好是这种 (0,134,0) 的绿色，\n");
    std::printf("           所以\"纯绿\"往往等于\"根本没拿到画面\"。）\n");
}
```

检测方法很朴素：数这一帧用到了几种不同的字节值。真实画面（哪怕是白墙，也有噪点）不可能只有 4 种以下的字节值——**纯色帧 = 异常帧**。

而"纯绿"的来历是一段宝贵的排障知识：很多摄像头异常时驱动会吐**全零缓冲**。全零的 YUV（Y=0, U=0, V=0）经过标准的 BT.601 色彩转换公式算出来，恰好是 RGB 的 **(0, 134, 0)**——一种刺眼的纯绿。所以业内有句经验之谈：**"看到纯绿，不是色彩转换写错了，是根本没拿到画面。"** 很多新手会在这个坑里去查自己的 YUV→RGB 公式，白白浪费一天。这个测试用 4 行打印把这一天省下来了。

### 9.5 第五段：画面在动吗

```cpp
size_t different = 0;
for (size_t i = 0; i < compareSize; ++i) {
    if (a[i] != b[i]) ++different;
}
const double ratio = 100.0 * different / compareSize;
std::printf("  首帧与末帧有 %.1f%% 的字节不同\n", ratio);
std::printf("  （对着静止的墙拍，这个比例接近 0 是正常的；晃一下手再跑，数字应该明显变大）\n");
```

逐字节对比首帧和末帧的差异比例。这个测试验证的是"**我们拿到的是活体画面，不是同一帧被反复投递**"——如果采集链路某处有 bug 导致一直回调同一块缓冲，这个比例会是 0。

它还顺便给了我们一个体感校准：静止场景 ≈ 0%，挥手场景明显变大。一个数字，两种读法。

### 9.6 第六段：重复开关稳定性

```cpp
for (int round = 0; round < 3; ++round) {
    capturer.close();
    const bool reopened = capturer.open(deviceIndex, nullptr, requestFormat);
    if (!reopened) { check(false, ...); break; }
    if (!capturer.nextFrame(kTimeoutMs)) { check(false, ...); break; }
    if (round == 2)
        check(true, "反复 close()/open() 三次都成功");
}
```

连续三次 `close → open → 取帧`。这个测试专门抓**资源泄漏类 bug**：

- COM 对象没 Release 干净 → 第二次 open 时设备仍被占用，BindToObject 失败；
- 图没停干净 → 驱动句柄泄漏，第二次 Run 失败；
- 回调没摘干净 → 崩溃或野指针。

这类 bug 单次开关完全看不出来，只有反复开关才现形。**"能开一次"和"能开一百次"之间隔着整个资源管理的功底**，这个测试就是功底探测器。

---

## 第十章：动手环节

理论讲完了，轮到大家动手。请按下面的步骤做实验，每一步都观察输出。

### 实验一：换设备索引

```bash
test_camera_capture.exe 0      # 第一个摄像头（默认）
test_camera_capture.exe 1      # 第二个摄像头（如果枚举列表里有的话）
```

观察点：

- 第一段的枚举列表里有几个设备？名字都认识吗？（有没有"OBS Virtual Camera"这类虚拟设备混在里面？）
- 传一个越界的索引（比如 99），看错误提示。

### 实验二：换格式名，看"请求的和拿到的不一样"

```bash
test_camera_capture.exe 0 rgb24
test_camera_capture.exe 0 mjpg
test_camera_capture.exe 0 yuy2
test_camera_capture.exe 0 any
```

观察点：

- 每次的"实际协商结果"日志一样吗？
- 请求 `rgb24` 时拿到的是 RGB24 吗？如果不是，拿到的是什么？
- 请求 `mjpg` 时，第三段的"每帧字节数一致"断言过了吗？（想想为什么 MJPG 每帧大小会不一样。）
- `any` 模式下，摄像头原生给的是什么格式？

### 实验三：MJPG 的 jpg 文件

如果你的摄像头支持 MJPG，跑完 `test_camera_capture.exe 0 mjpg` 后，到工作目录找 `camera_frame_dev0.jpg`，双击打开。

- 能看到画面吗？画面是倒的吗？
- 用十六进制编辑器（或 VS Code 的 hex 插件）打开这个 jpg，确认开头两个字节是 `FF D8`、结尾是 `FF D9`。

### 实验四：裸格式的 raw 文件与 stride 对账

跑 `test_camera_capture.exe 0 rgb24`（或 yuy2），观察第四段输出：

- "每行理论字节"和"实际行跨度"相等吗？差几个字节？用 1.8 节的公式手算验证一下。
- "字节数 ≥ stride × height" 成立吗？多出了多少字节？

### 实验五：纯绿帧复现（可选）

用手把摄像头镜头完全捂住，再跑一次测试，观察第四段的"首帧里用到了 N 种不同的字节值"。N 是多少？看到纯绿警告了吗？

### 实验六：画面在动

- 先对着静止的墙跑一次，记下第五段的差异百分比；
- 再在摄像头前挥手跑一次，对比数字。

---

## 第十一章：坑点与改进

老规矩，讲完优点讲局限。这份代码作为教学样本非常优秀，但作为生产代码还有这些坑：

### 11.1 pts 只是帧序号，不是时间戳

`frame->pts = frameCounter.fetch_add(1)` 给出的是 0、1、2、3……的序号。**而 `BufferCB` 的第一个参数 `sampleTime`（DirectShow 的流时间戳）被直接丢弃了**——函数签名里那个形参连名字都没写。

为什么这是硬伤？推流要做**音画同步**：视频帧和音频帧各自带时间戳，编码器和播放器靠它对齐。序号代替不了时间戳——摄像头实际帧率漂移、丢帧，序号都反映不出来。改进方向：把 `sampleTime`（单位是 100 纳秒的 REFERENCE_TIME）换算成毫秒填进 pts。

### 11.2 无热插拔处理

采集过程中拔掉 USB 摄像头会怎样？图会**静默停帧**——不报错、不回调，`nextFrame` 只剩超时返回 nullptr。调用方看到的只是"没帧了"，分不清是摄像头被拔了还是画面静止（静止也会持续有帧，只是内容不变）。

DirectShow 其实提供 `IMediaEventEx` 接口，能收到 `EC_DEVICE_LOST` 事件。本项目没接。生产代码应该接：收到事件后通知上层"设备丢失"，UI 弹提示。

### 11.3 DirectShow 本身已老

DirectShow 是上世纪的技术，微软主推的继任者是 **MediaFoundation**——对 MJPG 硬件解码、低延迟采集、现代 UVC 特性的支持都更好。选 DirectShow 是兼容性优先的务实选择（老摄像头、老驱动全兼容），但要清楚代价：新特性拿不到，且这个框架不会再演进。

### 11.4 pending 单槽无统计

丢旧保新是正确策略，但**丢了多少完全无感知**。消费者不知道自己落后了。接 BoundedQueue（commit 3）之后，至少能统计 dropped 计数。另外单槽意味着"拿到的可能已经被覆盖过几轮"，对需要逐帧处理的场景（比如录像）不够用——实时预览够用，录像得换队列。

### 11.5 Release 不 delete this 的 COM 对象

第四章讲过，`SampleGrabberCallback::Release` 归零不自毁，严格说违反 COM 惯例。对象生命周期由 Impl 明确拥有，**当前**可控；但设想这个场景：DirectShow 内部某处在 `Stop()` 之后仍持着回调的引用、又晚一拍调了一次 BufferCB——此时 Impl 可能已析构，回调打到野对象上。代码用 `stopping` 标志 + "Stop 先于摘回调"的顺序缓解，但缓解不是铁律。更稳的写法是让回调对象自己也走 `shared_ptr` 式的所有权，或者 Release 归零时真正自毁、Impl 改持裸指针。

### 11.6 不能选分辨率和帧率

`open()` 只协商**像素格式**，分辨率完全是驱动默认值（很多摄像头默认 640×480，明明是 1080p 的硬件）。DirectShow 的 `IAMStreamConfig` 接口可以枚举和设置分辨率/帧率，本项目没实现。下一阶段的改进清单里应该有它。

### 11.7 MJPG 还没解码

作者自己在日志里说了"下一轮转换器里需要解码"。当前 MJPG 摄像头拿到的 bytes 是一帧帧 JPEG 压缩数据，**进不了编码链路**——必须先解码成裸像素。好在 `bytes` 本身就是标准 JPEG，任何 JPEG 解码库都能吃。

### 11.8 其余小坑

- `frame->bytes` 可能含尾部零填充（测试程序实测发现），结构层面没有约束，靠日志提醒下游；
- `close()` 里 `Stop()` 是同步阻塞，回调若正在拷贝大帧会有可感知延迟；
- 手写 COM 接口虽有官方固定 GUID 保底，但仍是"协议靠人肉对齐"的维护风险——更稳的做法是项目里自带一份 qedit.h。

---

## 第十二章：小结

这一讲我们完成了采集层的一半版图。回顾主线：

1. **摄像头是推模型**：硬件按自己的节奏产帧，我们只能在回调里接。这决定了 `open / nextFrame / close` 的 API 形态，与屏幕采集的 `capture()` 拉模型形成镜像。
2. **DirectShow 过滤器图**：Source → Sample Grabber → Null Renderer 三段流水线，`CaptureGraphBuilder2::RenderStream` 自动连线并在需要时插入转换器。
3. **COM 的本质是虚表 + GUID**：所以 `qedit.h` 消失了也能手写 `ISampleGrabber`，GUID 不变、虚表顺序不变，二进制契约就成立。
4. **格式是"协商"出来的，不是"指定"的**：请求 → 连线 → 读实际结果，两级降级（换引脚 → 放弃格式），请求的 vs 拿到的经常不一样。
5. **`CameraRawFrame` 忠实记录原始事实**：格式、stride、翻转方向全部入帧，转换职责留给下游——采集层只对"字节原样拿到"负责。
6. **单槽最新帧 + 必须拷贝**：回调线程到消费线程用 mutex + cv + 覆盖式 pending 交接；DirectShow 的 buffer 不拷贝就是野指针。
7. **测试即教案**：六段测试覆盖了枚举、协商、连取、字节解读（jpg 双击验证 / stride 对账 / 纯绿排障）、活性检查、反复开关稳定性。

至此，采集层两个源（屏幕 + 摄像头）都齐了。下一讲将进入**格式转换**：把 YUY2、MJPG、倒立的 RGB24 统一成编码器能吃的标准帧——也就是日志里反复提到的"下一轮转换器"。

---

## 思考题

1. 为什么说 `pending = std::move(frame)` 的"覆盖"语义在实时场景下是优点而不是缺陷？如果换成容量为 8 的队列，会发生什么现象？（提示：想想"延迟"和"积压"的关系。）
2. `open()` 里先 `close()` 开头、`fail` 里再 `close()` 收尾——请论证为什么 `close()` 必须幂等，以及如果它不幂等会出什么事。
3. 手写 COM 接口时，如果漏声明了 `ISampleGrabber` 的某一个我们"用不到"的方法，会发生什么？请从虚表布局的角度解释。
4. 某摄像头协商结果是 1919×1080、RGB24、biHeight = 1080（正值）。请算出 stride、`bytes` 的理论最小长度，并回答：消费方把字节直接按"从左到右、从上到下"渲染，画面会是什么样？
5. 测试第六段"反复开关三次"为什么能抓出 COM 泄漏？请描述一个"第一次 open 成功、第二次 open 必失败"的泄漏场景。
6. 如果要把 `sampleTime` 真正用起来做音画同步，`CameraRawFrame::pts` 该存什么？请设计换算方案（注意 `sampleTime` 的单位和起点）。
7. `listDevices()` 用 `ComScope` 做 RAII 初始化，而 `open()` 手动管理 `comOwned`——为什么不统一用 `ComScope`？（提示：想想两个函数里 COM 生命周期的作用域。）

---

**本讲配套文件**：`src/core/capture/CameraCapturer.h`（62 行）、`src/core/capture/CameraCapturer.cpp`（587 行）、`tests/test_camera_capture.cpp`（387 行）
**下一讲预告**：格式转换器 —— YUY2/UYVY 解包、MJPG 解码、垂直翻转与 stride 剔除，统一输出 BGRA。

---

## 附：本讲"五章对照索引"

| 标准章节 | 在本讲的位置 | 一句话要点 |
|---|---|---|
| **核心原理** | 开场白 + 第一章知识点预习 | 摄像头"推"模型 vs 屏幕"拉"模型；DirectShow 过滤器图三段流水线；格式协商而非指定；pending 单槽最新帧；回调必须拷贝字节 |
| **用到的技术 / 知识点** | 第一章全部小节 | 过滤器图与 Pin、COM=虚表布局+GUID（手写消失的 qedit.h）、Moniker 枚举、VARIANT/BSTR、宽字符转 UTF-8、COM 套间与 RPC_E_CHANGED_MODE、RGB24/RGB32/YUY2/UYVY/MJPG、DIB 行对齐 `((w*bits+31)/32)*4`、biHeight 正负、VIDEOINFOHEADER/2、AM_MEDIA_TYPE |
| **实现细节** | 第二~九章 | 头文件契约（CameraRawFrame 为何不是 VideoFrame）；工具函数群七件套；SampleGrabberCallback 手写三方法；Impl 状态；open() 十步全流程；onFrame；nextFrame/close 关停顺序；测试程序六段 |
| **关键方法说明** | 各章方法表 + 数据流全景 | `listDevices` / `open` / `nextFrame` / `close` / `onFrame` / `width|height|pixelFormat`；open 的两级降级链；帧从驱动到 pending 的完整链路 |
| **缺陷与局限** | 第十一章"坑点与改进" | pts 只是帧序号（丢弃了 sampleTime）、pending 单槽无队列无统计、手写 COM 接口维护风险、DirectShow 已过时、Release 不自毁、无热插拔（未接 IMediaEventEx）、不能选分辨率/帧率（未用 IAMStreamConfig）、MJPG 未解码、bytes 尾部零填充、Stop 同步阻塞 |
