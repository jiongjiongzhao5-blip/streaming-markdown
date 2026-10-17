# 第 4 讲：屏幕采集 —— ScreenCapturer（DXGI 桌面复制 + GDI 兜底）

> **Commit**：`81eaa1f5`（2026-10-15 15:03）
> **提交信息**：把屏幕上显示的画面抓成一帧 VideoFrame（BGRA32）
> **改动规模**：7 个文件，约 590 行。新增 `src/core/capture/ScreenCapturer.h/.cpp`、`tests/test_screen_capture.cpp`、capture 模块的 CMake，修改根 `CMakeLists.txt` 和 `.gitignore`。
> **本讲性质**：培训班讲稿。请同学们打开电脑，跟着老师一步一步来。

---

## 目录

- [一、开场白：屏幕采集到底在干什么](#一开场白屏幕采集到底在干什么)
- [二、知识点预习](#二知识点预习)
  - [2.1 DWM 桌面合成器](#21-dwm-桌面合成器)
  - [2.2 为什么 GPU 内存 CPU 读不了：staging 纹理](#22-为什么-gpu-内存-cpu-读不了staging-纹理)
  - [2.3 COM 是什么：引用计数、HRESULT、IID_PPV_ARGS](#23-com-是什么引用计数hresultiid_ppv_args)
  - [2.4 DXGI 对象链：Factory → Adapter → Output → Duplication](#24-dxgi-对象链factory--adapter--output--duplication)
  - [2.5 两种取帧模型："变化推送" vs "主动截屏"](#25-两种取帧模型变化推送-vs-主动截屏)
  - [2.6 RowPitch 行跨度与"斜纹"](#26-rowpitch-行跨度与斜纹)
  - [2.7 GDI 五件套](#27-gdi-五件套)
  - [2.8 BMP 文件格式与自底向上存储](#28-bmp-文件格式与自底向上存储)
  - [2.9 biHeight 负数的秘密](#29-biheight-负数的秘密)
  - [2.10 DPI 缩放问题](#210-dpi-缩放问题)
- [三、工程改动总览](#三工程改动总览)
- [四、逐函数教学：ScreenCapturer.h](#四逐函数教学screencapturerh)
- [五、逐函数教学：init() —— 六关逐级初始化与降级哲学](#五逐函数教学init-六关逐级初始化与降级哲学)
- [六、逐函数教学：close() 与 capture() 调度](#六逐函数教学close-与-capture-调度)
- [七、逐函数教学：captureDxgi() 全流程](#七逐函数教学capturedxgi-全流程)
- [八、逐函数教学：gdiCapture() 逐行讲](#八逐函数教学gdicapture-逐行讲)
- [九、测试程序教学](#九测试程序教学)
  - [9.1 测试设计思路：为什么屏幕采集一半正确性只能人眼核对](#91-测试设计思路为什么屏幕采集一半正确性只能人眼核对)
  - [9.2 测试基建：check、日志收集、计时](#92-测试基建check日志收集计时)
  - [9.3 saveBmp：手写 BMP 文件](#93-savebmp手写-bmp-文件)
  - [9.4 averageBrightness：亮度统计](#94-averagebrightness亮度统计)
  - [9.5 五段测试正文](#95-五段测试正文)
  - [9.6 静止 200 连抓：静止画面兜底行为](#96-静止-200-连抓静止画面兜底行为)
- [十、动手环节](#十动手环节)
  - [10.1 编译运行](#101-编译运行)
  - [10.2 看 bmp 文件](#102-看-bmp-文件)
  - [10.3 三大经典症状自查清单](#103-三大经典症状自查清单)
- [十一、坑点与改进](#十一坑点与改进)
- [十二、小结](#十二小结)
- [十三、思考题](#十三思考题)

---

## 一、开场白：屏幕采集到底在干什么

同学们，上一讲我们把项目的"骨架"搭好了——`VideoFrame`、日志回调、有界队列这些公共契约都有了。从这一讲开始，我们要往骨架上**长肉**了。直播推流的第一块肉，永远是"画面从哪儿来"。

画面从哪儿来？最常见、最刚需的答案就是：**电脑屏幕本身**。游戏直播播的是屏幕，教学直播播的是屏幕，连你们平时用的"屏幕共享"，本质都是同一件事。

那"屏幕采集"这件事，用一句大白话说清楚，到底是什么？

> **把显卡里正在显示的那张桌面图像，搬到 CPU 内存里，变成一块规规矩矩的 BGRA 像素数组，并且要每秒重复 30 次。**

这句话里有三个关键词，我们先圈出来：

1. **"显卡里"**。屏幕上显示的画面，最终形态是显卡显存里的数据。显存是 GPU 的地盘，我们普通的 C++ 代码跑在 CPU 上，**CPU 是摸不着显存的**。这中间有一道鸿沟，怎么跨过去，是本讲一半的技术含量所在。
2. **"搬到 CPU 内存"**。我们项目的公共数据格式是 `VideoFrame`，里面有一个 `std::vector<uint8_t> bgra`——这是普普通通的堆内存。采集器的产出必须落到这里，后续的编码器（FFmpeg x264）才吃得到。
3. **"每秒 30 次"**。采集不是截一张图就完事，它是一个**持续的流水线源头**。这立刻带来一个矛盾：如果用户盯着桌面发呆，屏幕一动不动，我们还要不要每秒产出 30 帧？要！因为编码器是恒定帧率的吃货，断粮它会乱。那静止时帧从哪儿来？这就是 `lastFrame` 缓存存在的理由，后面详讲。

Windows 上做这件事，有两代技术：

| 技术 | 年代 | 一句话原理 | 性能 | 兼容性 |
|---|---|---|---|---|
| **GDI BitBlt** | Windows 95 时代 | 让显示驱动把屏幕内容"画"到一块内存位图上，再把位图像素读出来 | 慢（1080p 一帧约 10~30 毫秒） | 任何 Windows 都能用，老掉牙但死不了 |
| **DXGI Desktop Duplication** | Windows 8 起 | 桌面合成器每合成出一帧新桌面，系统直接给你一块 GPU 纹理 | 快（GPU→GPU 拷贝，接近零开销） | Win8+，且可能被别的录屏软件占用 |

我们这次提交的 `ScreenCapturer` 是一个**双路径**设计：

- **主路径**：DXGI Desktop Duplication，快、省电、还能感知"画面有没有变化"；
- **兜底路径**：GDI BitBlt，慢，但**只要 Windows 还活着就能抓到图**。

两条路径之间靠一套"降级哲学"串起来，这套哲学用一句话概括就是：

> **有画面，比画面来源高级重要。**

初始化 DXGI 的任何一步失败，我们都不是报错退出，而是"记一条日志、擦干净手、走 GDI"。这是一个做产品的思路，不是做玩具的思路。同学们以后写自己的采集器，请把这句话贴在显示器上。

本讲的路线图：

1. 先花比较大的篇幅做**知识点预习**——这一讲涉及 Windows 图形栈的底层概念特别多（DWM、COM、DXGI、RowPitch、BMP、DPI……），不预习直接看代码会懵；
2. 然后**逐函数**把 `ScreenCapturer` 的实现讲完；
3. 再讲**测试程序**——这次的测试很特别，因为屏幕采集"对不对"有一半只能靠人眼看 bmp 图片判断；
4. 最后**动手环节**、坑点清单、思考题。

好，翻开课本，进入预习环节。

---

## 二、知识点预习

### 2.1 DWM 桌面合成器

同学们先看自己眼前的 Windows 桌面：壁纸、任务栏、几个窗口叠在一起。有没有想过，这些窗口各自画的像素，最后是怎么"拼"成你看到的这一整张图的？

答案是 **DWM（Desktop Window Manager，桌面窗口管理器）**。从 Windows Vista 开始，Windows 的每个窗口不再直接往屏幕上画，而是各自画到一块**离屏表面（off-screen surface）**上，由 DWM 这个"总合成师"把所有窗口的表面按层次、透明度、位置**合成**成一张最终图像，交给显示器。

这个机制对我们的意义是革命性的：

> **既然 DWM 手里本来就有"整张桌面的最终图像"，那我们能不能向它要一份拷贝？**

能。Windows 8 起提供的 **DXGI Desktop Duplication API**（桌面复制 API）干的就是这件事：应用程序可以"订阅"DWM 的合成输出，每当桌面内容变化产生新的一帧，系统就把这帧以 GPU 纹理的形式交给订阅者。

请注意这里的一个关键词：**"变化"**。DWM 是懒惰而高效的——屏幕不动，它就不重新合成，也就**没有新帧给你**。这不是 bug，是特性。但这个特性和我们"恒定 30fps 喂编码器"的需求是拧着的，怎么解，2.5 节讲。

### 2.2 为什么 GPU 内存 CPU 读不了：staging 纹理

上节说了，DXGI 给我们的帧是一块 **GPU 纹理（texture）**。纹理是什么？通俗讲就是"放在显存里的一张二维图片，GPU 可以随时高速读写它"。

问题来了：我们的目标是拿到 CPU 内存里的 `std::vector<uint8_t>`，而纹理躺在显存里。**CPU 直接读显存，要么根本做不到，要么慢到没法用**（要经过 PCI-E 总线一个一个字节地拽）。这是硬件层面的鸿沟，谁也绕不开。

Direct3D 11 给出的标准解法叫 **staging 纹理（暂存纹理）**——"staging"这个词本意是"中转站、 staging area"。它是一张特殊的纹理，创建时带上两个标志：

- `D3D11_USAGE_STAGING`：告诉 D3D"这张纹理不做渲染，只做数据中转"；
- `D3D11_CPU_ACCESS_READ`：告诉 D3D"CPU 要读它"。

有了 staging 纹理，搬运流程就变成三步：

```
DWM 的桌面纹理（GPU 显存，CPU 读不了）
        │  CopyResource     ← GPU 内部拷贝，显存到显存，极快
        ▼
staging 纹理（特殊内存，CPU 可读的纹理）
        │  Map              ← 拿到一个 CPU 指针
        ▼
CPU 内存指针 → 逐行 memcpy → VideoFrame.bgra（std::vector）
```

打个生活化的比方：显存是机场禁区内停机坪上的货，CPU 是市区里的收货人。收货人不能进禁区，怎么办？机场提供一辆**摆渡车**（CopyResource），把货运到**中转仓库**（staging 纹理），收货人拿着**仓库钥匙**（Map 返回的指针）进去慢慢搬（memcpy），搬完**还钥匙**（Unmap）。

这三个动词——`CopyResource`、`Map`、`Unmap`——请同学们记住，第七讲代码里会反复出现。

### 2.3 COM 是什么：引用计数、HRESULT、IID_PPV_ARGS

接下来预习一个绕不开的"老古董"：**COM（Component Object Model，组件对象模型）**。

COM 是微软上世纪九十年代搞出来的一套**二进制级别的对象接口规范**。DXGI、D3D11、WASAPI（下一讲的音频采集）这些 Windows 多媒体 API，几乎全是 COM 对象。我们不需要成为 COM 专家，但必须掌握它的三条"生存规则"：

**规则一：引用计数（Reference Counting）。**
COM 对象自己记账：内部维护一个计数器。`AddRef()` 加一，`Release()` 减一，减到零对象自杀。**你永远不写 `delete`，你只调 `Release()`**。拿到一个 COM 指针，就意味着你欠了一笔债：用完必须 `Release()`。漏调 = 内存泄漏（而且是 GPU 资源泄漏，比普通泄漏更疼）；多调 = 崩溃。

**规则二：HRESULT 返回值。**
COM 函数不抛异常，全部返回一个 32 位的 `HRESULT`。最高位是 0 表示成功（`S_OK` 就是 0），是 1 表示失败。配套两个宏：

- `SUCCEEDED(hr)`：判断是否成功；
- `FAILED(hr)`：判断是否失败。

本讲会见到两个具体的失败码：

- `DXGI_ERROR_WAIT_TIMEOUT`：等帧超时——屏幕没变化，没有新帧；
- `DXGI_ERROR_ACCESS_LOST`：访问丢失——锁屏、切换用户、远程桌面断开都会导致桌面复制会话失效。

**规则三：QueryInterface 与 IID_PPV_ARGS。**
一个 COM 对象可以有多张"面孔"（接口）。比如你手里有一个 `IDXGIOutput*`（显示器接口），但你想要它的升级版 `IDXGIOutput1*`（才有 `DuplicateOutput` 方法），怎么拿？用 `QueryInterface`："喂，对象，你有没有 IDXGIOutput1 这张面孔？有的话给我。"

写法是固定套路：

```cpp
obj->QueryInterface(IID_PPV_ARGS(&ptr));
```

`IID_PPV_ARGS` 是个宏，展开后等价于：

```cpp
obj->QueryInterface(IID_IDXGIOutput1, reinterpret_cast<void**>(&ptr));
```

`IID_xxx` 是接口的全球唯一编号（GUID），宏帮我们自动填对了它，避免手写 IID 写错。**记忆口诀：IID_PPV_ARGS 一套，换脸成功**。

最后再介绍一个本讲代码里自己写的小工具，它就是为规则一服务的：

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

模板函数，参数是指针的引用（`T*&`），所以能把调用者的指针**本体**置空。这是个好习惯：Release 之后立刻置 nullptr，杜绝"悬垂指针二次 Release"这种经典崩溃。

### 2.4 DXGI 对象链：Factory → Adapter → Output → Duplication

**DXGI（DirectX Graphics Infrastructure，DirectX 图形基础设施）**是 DirectX 家族里负责"底层杂务"的成员：枚举显卡、枚举显示器、管理交换链这些活儿都归它。我们要创建桌面复制会话，必须沿着一条固定的对象链，一级一级往下走：

```
IDXGIFactory1（工厂，一切 DXGI 对象的入口）
    │  CreateDXGIFactory1() 创建
    ▼
IDXGIAdapter1（显卡适配器，一块物理显卡对应一个 adapter）
    │  factory->EnumAdapters1(0, ...) 枚举第 0 块显卡
    ▼
IDXGIOutput（显示器输出，一台显示器对应一个 output）
    │  adapter->EnumOutputs(0, ...) 枚举第 0 台显示器
    ▼
IDXGIOutput1（output 的升级版接口，QueryInterface 换来）
    │  output1->DuplicateOutput(device, ...)
    ▼
IDXGIOutputDuplication（桌面复制会话，主角登场）
```

这条链的口诀是：**厂子办卡，卡里选屏，屏上复制**。

中间还有一个 D3D11 的家伙要插进来：调用 `DuplicateOutput` 时必须提供一个 `ID3D11Device`。这里出现两个新概念：

- **ID3D11Device（设备）**：GPU 在软件世界的"逻辑代表"。创建任何 GPU 资源（纹理、缓冲……）都通过它。可以把它理解成"GPU 营业厅的柜台"。
- **ID3D11DeviceContext（设备上下文）**：向 GPU 提交命令的通道。`CopyResource`、`Map` 这些动作都走 context。柜台（device）负责办资源，跑腿的（context）负责干活。

这两个由 `D3D11CreateDevice` 一次性创建出来。

同学们可能问：为什么要先 QueryInterface 把 `IDXGIOutput` 升级成 `IDXGIOutput1`？因为 `DuplicateOutput` 这个方法**只定义在 1 版接口上**——这是 COM 的版本演进方式：老接口一个字节不改，新功能全加到新接口上，靠 QueryInterface 互通。

### 2.5 两种取帧模型："变化推送" vs "主动截屏"

这是本讲**最重要的一个思想差异**，请同学们务必理解透。

**GDI 的模型是"主动截屏"**：你想抓，随时抓。调用 `BitBlt` 的那一刻，系统就把当前屏幕内容拷给你。屏幕动没动、动多少，系统不关心——你问一次，它拍一张。这是**拉（pull）模型**，主动权完全在调用方。

**DXGI 的模型是"变化推送"**：你调用 `AcquireNextFrame`，意思是"下一帧**新**桌面给我"。如果屏幕静止，DWM 没有合成新画面，这个调用就会**超时**，返回 `DXGI_ERROR_WAIT_TIMEOUT`——系统在说："没变化，没有新东西给你。"这是**推（push）模型**，节奏由系统掌握。

两种模型各有优劣：

| | GDI 主动截屏 | DXGI 变化推送 |
|---|---|---|
| 静止屏幕 | 照样每秒给你 30 张一模一样的图（浪费） | 什么都不给（省电，但断了粮） |
| 性能 | 每帧都要 CPU 全程参与，慢 | GPU 内部拷贝，快 |
| 画面变化感知 | 不知道变没变 | 天然知道（不给你就是没变） |

我们的编码器要的是**恒定 30fps**，而 DXGI 在静止时**断粮**。怎么办？桥接方案其实非常朴素：

> **自己缓存上一帧（lastFrame）。DXGI 超时了，就把上一帧的拷贝交出去。**

对编码器来说，它永远能按时拿到帧；对系统来说，画面没变时我们没有做任何重复的 GPU 搬运——只是把内存里已有的像素数组再 memcpy 一份。这就是把"变化推送"模型适配成"恒定帧率拉取"模型的桥。测试程序的第三段（静止 200 连抓）专门验证这件事。

### 2.6 RowPitch 行跨度与"斜纹"

这一段讲的是本讲最容易踩、症状最诡异的一个坑。

当我们 `Map` 一张 staging 纹理时，拿到的不只是数据指针 `pData`，还有一个 `RowPitch`——**行跨度**，即"内存里相邻两行像素起始地址之间隔多少字节"。

直觉上，1920 宽的 BGRA 图，一行应该是 1920×4 = 7680 字节，对吧？**不一定**。GPU 出于对齐优化的考虑，可能把每一行的实际存储宽度**补齐**到某个对齐边界，比如 7680 补齐到 7936。那么多出来的 256 字节就是每行末尾的"填充垃圾"，不是像素。

所以逐行拷贝时的正确姿势是：

```cpp
源指针每次跳 mapped.RowPitch 字节   ← 跟着 GPU 的实际排布走
目标指针每次跳 width*4 字节         ← 我们的 vector 是紧密排列的
```

**如果偷懒，假设 RowPitch 一定等于 width×4，直接一次性整块 memcpy 会怎样？**

答案就是著名的"**斜纹图**"：因为每行都错位了几个字节，错位会逐行累积，画面里的直线全部变成斜线，整张图像像被拧过一样。这个症状太经典了，经典到我们测试程序的提示语里专门写了它。

记忆口诀：**"源按 Pitch 走，目标按宽走；整块 memcpy，斜纹管够。"**

### 2.7 GDI 五件套

预习完 DXGI，回头看老前辈 GDI。**GDI（Graphics Device Interface，图形设备接口）**是 Windows 最古老的 2D 绘图 API，从 Win95 用到现在。GDI 编程有个核心概念叫 **DC（Device Context，设备上下文）**——注意和 D3D11 的 DeviceContext 名字撞车但完全是两码事——通俗理解，DC 就是"一块可以往上画东西的画布的句柄"。屏幕有屏幕的 DC，打印机有打印机的 DC，内存里也可以造一块虚拟画布（内存 DC）。

GDI 截屏的固定套路是五个函数，江湖人称"**GDI 五件套**"：

| 步骤 | 函数 | 干什么 |
|---|---|---|
| 1 | `GetDC(nullptr)` | 拿到**整个屏幕**的 DC（nullptr 表示全屏） |
| 2 | `CreateCompatibleDC(screenDc)` | 造一块和屏幕"兼容"的内存 DC（虚拟画布） |
| 3 | `CreateCompatibleBitmap(screenDc, w, h)` | 造一张和屏幕兼容的**位图**（画纸），大小 = 屏幕分辨率 |
| 4 | `SelectObject(memDc, bmp)` | 把画纸铺到虚拟画布上（DC 必须选中一张位图才能存像素） |
| 5 | `BitBlt(...)` | **位块传输**（Bit Block Transfer）：把屏幕 DC 的内容原样拷贝到内存 DC |

拷完之后，像素在 `bmp` 这张位图里，但 GDI 位图的内部格式我们不方便直接用，所以还有第六步：`GetDIBits`——按我们指定的格式（32 位 BGRA、顶向下）把位图像素**读出来**，直接灌进 `frame->bgra` 这个 vector。

最后是**资源释放四步曲**，顺序固定、一步不能少：

```cpp
SelectObject(memDc, oldBitmap);   // 把旧位图选回去（释放我们位图的占用）
DeleteObject(bmp);                // 销毁位图
DeleteDC(memDc);                  // 销毁内存 DC
ReleaseDC(nullptr, screenDc);     // 归还屏幕 DC
```

GDI 句柄是**全系统共享的有限资源**（每个进程默认上限约一万个），漏了任何一步就是句柄泄漏——泄漏多了，整个系统的图形程序都会出怪病。

另外注意 `BitBlt` 最后一个参数：`SRCCOPY | CAPTUREBLT`。

- `SRCCOPY`：光栅操作码，意思是"源内容直接覆盖拷贝"，不做任何混合；
- `CAPTUREBLT`：把**分层窗口**（layered window——半透明窗口、某些悬浮窗、毛玻璃效果窗口）也一起抓进来。不加这个标志，某些窗口内容会漏抓。

### 2.8 BMP 文件格式与自底向上存储

测试程序里我们要把抓到的帧存成图片给人眼看。存什么格式？PNG 要引入库，麻烦；**BMP 是最"裸"的图片格式**——文件头后面直接跟原始像素，不压缩。手写一个 BMP 写入器只要几十行代码，所以我们自己写。

BMP 文件的结构，从头到尾是三段：

```
BITMAPFILEHEADER（14 字节）：魔数 "BM"、文件总大小、像素数据偏移
BITMAPINFOHEADER（40 字节）：宽、高、位深、压缩方式……
像素数据（width × height × 4 字节）：BGRA，一个像素挨一个像素
```

魔数 `"BM"` 两个字节，按小端序写成 `uint16_t` 就是 `0x4D42`（'B'=0x42，'M'=0x4D，低字节在前）。看到文件开头是 `BM`，所有看图软件都知道这是 BMP。

这里有一个**历史遗留的怪癖**，是 BMP 格式最著名的知识点：

> **BMP 的像素数据默认是自底向上（bottom-up）存储的。**

也就是说，文件里第一行像素，对应的是**画面的最下面一行**；文件最后一行，对应画面最上面一行。这是上世纪 OS/2 时代定下的规矩（当时屏幕坐标系原点在左下角），一直被兼容性拖到了今天。

后果是什么？我们 `VideoFrame.bgra` 里的第 0 行是**画面顶行**，而写 BMP 时文件要求先写**底行**。所以 `saveBmp` 里的写入循环是**倒序**的：

```cpp
for (int y = frame.height - 1; y >= 0; --y)
    std::fwrite(frame.bgra.data() + y * stride, stride, 1, file);
```

从最后一行写起，一路写到第 0 行。如果你正着写，保存出来的图就是**上下颠倒**的——这就是测试提示语里"画面是否上下颠倒"这条检查项的由来。

还有一个工程细节：两个头部结构体用 `#pragma pack(push, 1)` 包起来。这是告诉编译器"**关闭结构体对齐填充**"，让字段一个挨一个紧密排列。否则编译器为了对齐会在 `uint16_t` 后面塞填充字节，结构体大小就和 BMP 文件格式的真实字节布局对不上了，`fwrite` 出去的头就是错的。

### 2.9 biHeight 负数的秘密

上一节讲了 BMP **文件**的自底向上存储。GDI 的 `GetDIBits` 读取位图时，也有一个一模一样的方向问题，而控制开关藏在 `BITMAPINFOHEADER` 的 `biHeight` 字段里：

- `biHeight = h`（正值）：给我**自底向上**排列的像素——第 0 行是画面底行；
- `biHeight = -h`（**负值**）：给我**顶向下**（top-down）排列的像素——第 0 行是画面顶行。

我们的 `VideoFrame.bgra` 约定是顶向下（第 0 行 = 屏幕顶行），所以代码里写的是：

```cpp
bmi.bmiHeader.biHeight = -h;   // 负号 = 我要顶向下的数据
```

这个负号只要忘了打，GDI 路径抓出来的每一帧都是**上下颠倒**的。它不会报错、不会崩溃，图也是全的，就是倒了——这类"静默错误"最难排查，所以老师在这里敲一次黑板：**biHeight 的负号，是 GDI 截屏代码里身价最高的一个字符。**

有意思的是：BMP 文件写入靠"倒序循环"解决方向问题，GetDIBits 靠"负 biHeight"解决方向问题——同一个历史包袱，两处不同的补丁。同学们把这两处对照着记，就不容易混。

### 2.10 DPI 缩放问题

最后一个预习知识点：**DPI 缩放**。

现在很多人的显示器是高分辨率屏，Windows 会开"缩放"——比如 150%、200%。开了缩放之后，Windows 的世界里就有两种像素：

- **物理像素**：显示器真实的像素点数，比如 3840×2160；
- **逻辑像素**：除以缩放比例后的数字，比如 150% 缩放时，3840×2160 的屏逻辑分辨率只有 2560×1440。

麻烦在于：`GetSystemMetrics(SM_CXSCREEN)` 返回的是**逻辑像素**还是**物理像素**，取决于进程有没有声明 **DPI awareness（DPI 感知）**。我们的程序没声明，于是系统把我们当"老程序"对待：返回逻辑像素，再由系统把我们画的东西放大。

对 GDI 路径来说，这意味着：150% 缩放的机器上，`GetSystemMetrics` 拿到的宽只有物理宽的 2/3，`BitBlt` 抓的也只有这么大——**抓到的画面只有屏幕左上角的一块**，或者整张图是糊的。DXGI 路径没有这个问题（它拿到的永远是物理分辨率），这反而衬托得兜底路径更粗糙。

这个坑我们现在**不修**（见第十一讲"坑点与改进"），但测试程序把它列进了人眼检查清单——知道病根在哪儿，比蒙在鼓里强。

好，预习结束。同学们喝口水，我们正式进入代码。

---

## 三、工程改动总览

先看这次提交在工程层面动了哪些地方，再进代码细节。

**1. `.gitignore` 追加三行：**

```
*.bmp
*.raw
*.pcm
```

测试程序会往仓库目录里写 bmp 图片（后面音频模块还会写 pcm），这些是产物不是代码，不配进版本库。顺手把 `*.raw` 也预判性地加进去，是写工程文件的好习惯。

**2. 根 `CMakeLists.txt` 加一行：**

```cmake
add_subdirectory(src/core/capture)
```

把新的采集模块挂进构建树，位置放在 `core/common` 之后——因为 capture 依赖 common（要用 `VideoFrame` 和 `LogCallback`）。

**3. 新增 `src/core/capture/CMakeLists.txt`（21 行）：**

```cmake
add_library(core_capture STATIC
  ScreenCapturer.cpp
  ScreenCapturer.h
)

target_include_directories(core_capture PUBLIC
  ${CMAKE_CURRENT_SOURCE_DIR}/../..
)

target_link_libraries(core_capture PUBLIC
  core_common
)

if(WIN32)
  target_link_libraries(core_capture PUBLIC
    d3d11
    dxgi
    gdi32
    user32
  )
endif()
```

逐条讲：

- 静态库 `core_capture`，两个源文件；
- 头文件搜索路径设为 `src/` 的上一层目录，所以包含写法是 `"core/capture/ScreenCapturer.h"` 这种全路径风格——和项目其它模块保持一致；
- `PUBLIC` 链接 `core_common`：因为 `ScreenCapturer.h` 的头文件里就包含了 `core/common/Common.h`，依赖必须传递出去；
- 最后四个 Windows 系统库：`d3d11`（D3D11 设备）、`dxgi`（桌面复制）、`gdi32`（GDI 五件套）、`user32`（GetSystemMetrics、GetDC）。用 `if(WIN32)` 包起来，给未来跨平台留门——虽然目前实现是 Windows 专属的。

**4. `tests/CMakeLists.txt` 追加测试目标：**

```cmake
add_executable(test_screen_capture
  test_screen_capture.cpp
)

target_link_libraries(test_screen_capture PRIVATE
  core_capture
)
```

**5. 两个新源文件**：`ScreenCapturer.h`（29 行）、`ScreenCapturer.cpp`（258 行）、`tests/test_screen_capture.cpp`（269 行）。这就是我们接下来三讲的全部内容。

---

## 四、逐函数教学：ScreenCapturer.h

先看头文件，29 行，非常克制：

```cpp
#pragma once

#include "core/common/Common.h"

#include <memory>

class ScreenCapturer {
public:
    ScreenCapturer();
    ~ScreenCapturer();

    ScreenCapturer(const ScreenCapturer&) = delete;
    ScreenCapturer& operator=(const ScreenCapturer&) = delete;

    bool init(LogCallback log);

    void close();

    VideoFramePtr capture();

private:
    VideoFramePtr captureDxgi();
    VideoFramePtr captureGdi();

    struct Impl;
    std::unique_ptr<Impl> impl_;

    bool dxgiReady_ = false;
};
```

老师带同学们逐块过：

**第一块：对外接口只有三个函数。**

- `init(LogCallback log)`：初始化。传入契约层定义的日志回调（还记得吗，`using LogCallback = std::function<void(const std::string&)>`）。返回值 bool，但先剧透一下：它**几乎永远返回 true**，因为失败会降级而不是报错。
- `capture()`：抓一帧，返回 `VideoFramePtr`（即 `std::shared_ptr<VideoFrame>`）。这是上层采集线程每 33 毫秒要调一次的函数。
- `close()`：释放全部资源。可重复调用（幂等）。

**第二块：拷贝构造和拷贝赋值被 `= delete` 禁掉。**

为什么？因为对象肚子里全是 COM 指针——COM 资源的生命周期归引用计数管，一个 `ScreenCapturer` 如果被随便拷贝，两份对象都以为自己持有同一批 COM 指针，析构时就会双重 Release，直接崩溃。**资源独占型对象，禁拷贝是标配。**

**第三块：PIMPL 惯用法。**

```cpp
struct Impl;                       // 前置声明，只有名字，没有内容
std::unique_ptr<Impl> impl_;       // 编译期只需要知道指针大小
```

**PIMPL（Pointer to Implementation，指向实现的指针）**，俗称"编译防火墙"。 Impl 的完整定义藏在 .cpp 文件里，里面全是 `IDXGIFactory1*` 这类 COM 成员。这么干有两大好处：

1. **头文件不需要包含 `<windows.h>` 和 `<d3d11.h>`**。这两个头文件是出了名的"宏污染大户"（`min`/`max` 宏之类的恶心东西全在里面），而且巨大无比。用了 PIMPL，包含 `ScreenCapturer.h` 的下游文件编译又快又干净；
2. **实现改动不触发下游重编译**。以后我们改 Impl 的成员，只有 ScreenCapturer.cpp 自己重编，其它文件纹丝不动。

代价是析构函数必须在 .cpp 里定义（不能在类内 inline），因为 `unique_ptr` 析构时需要 Impl 的完整类型——所以头文件里显式声明了 `~ScreenCapturer();`。

**第四块：`dxgiReady_` 为什么不放 Impl 里？**

注意这个布尔值放在外层类，而不是 Impl。这是有讲究的分层：

- **Impl 里装的是"资源句柄"**——COM 指针、分辨率、lastFrame 缓存，全是"东西"；
- **`dxgiReady_` 是"逻辑状态"**——当前调度策略该走哪条路，是"决策"。

资源归资源，决策归决策。后面会看到，降级 GDI 时 `close()` 会把 Impl 里的资源全清掉，但 `dxgiReady_` 的翻转发生在 `capture()` 的调度逻辑里——两个层次各管各的。

---

## 五、逐函数教学：init() —— 六关逐级初始化与降级哲学

进入 .cpp。先交代文件头部的三件套：

```cpp
#include "core/capture/ScreenCapturer.h"

#include <windows.h>

#include <d3d11.h>
#include <dxgi1_2.h>

#include <cstring>
```

windows.h（GDI、基础类型）、d3d11.h（设备与纹理）、dxgi1_2.h（`IDXGIOutput1` 和 `DuplicateOutput` 从 1.2 版 DXGI 才有）、cstring（memcpy）。然后是无名命名空间里的 `safeRelease`（2.3 节讲过）和 `gdiCapture`（第八节讲）。

接着是 Impl 的完整定义：

```cpp
struct ScreenCapturer::Impl {
    IDXGIFactory1* factory = nullptr;
    IDXGIAdapter1* adapter = nullptr;
    IDXGIOutput* output = nullptr;
    IDXGIOutput1* output1 = nullptr;
    ID3D11Device* device = nullptr;
    ID3D11DeviceContext* context = nullptr;
    IDXGIOutputDuplication* duplication = nullptr;
    ID3D11Texture2D* staging = nullptr;

    int width = 0;
    int height = 0;

    VideoFramePtr lastFrame;

    LogCallback log;
};
```

八个 COM 指针，正好对应 2.4 节的对象链加 D3D11 双件套加 staging 纹理。所有成员都有**默认初始值**——这是现代 C++ 的好习惯，保证 Impl 一构造出来就是干净的空状态，`close()` 随时可以安全调用。

构造函数和析构函数平平无奇：

```cpp
ScreenCapturer::ScreenCapturer()
    : impl_(std::make_unique<Impl>())
{
}

ScreenCapturer::~ScreenCapturer()
{
    close();
}
```

重头戏是 `init()`。老师先给同学们看它的**灵魂**——开头定义的 lambda：

```cpp
bool ScreenCapturer::init(LogCallback log)
{
    close();
    impl_->log = std::move(log);

    const auto fallbackToGdi = [this](const std::string& reason) {
        if (impl_->log)
            impl_->log(reason);
        close();
        return true;
    };
```

三件事：

1. 开头先 `close()`——允许重复 init，先把上一次的资源清干净，幂等设计；
2. 日志回调 `std::move` 进 Impl（`std::function` 拷贝有开销，move 是礼仪）；
3. **`fallbackToGdi` lambda**：做三件事——把降级原因告诉上层（记日志）、`close()` 释放建了一半的 COM 对象、**返回 true**。

注意那个 `return true`。这是本讲最重要的设计哲学：

> **降级不是失败。GDI 路径再慢、再糙，它也能出图。对用户来说，"画面来自 GDI"和"画面来自 DXGI"的差别他根本看不出来；但"有画面"和"黑屏报错"的差别是天壤之别。**

所以 init 的返回值语义其实是："**采集器是否已经可以开始干活**"，而不是"DXGI 是否初始化成功"。

接下来是六关。老师把它们画成闯关地图：

```
第1关：创建 DXGI 工厂
第2关：枚举显卡 + 枚举显示器
第3关：读出显示器分辨率并校验
第4关：QueryInterface 拿 IDXGIOutput1
第5关：创建 D3D11 设备
第6关：创建桌面复制会话
  ↓ 全部通过
创建 staging 纹理 → dxgiReady_ = true
```

**第 1 关：创建 DXGI 工厂。**

```cpp
if (FAILED(CreateDXGIFactory1(IID_PPV_ARGS(&impl_->factory))))
    return fallbackToGdi("DXGI 初始化失败，使用 GDI 采集");
```

`CreateDXGIFactory1` 是一切 DXGI 对象的入口。`IID_PPV_ARGS` 的用法复习一下：自动填对接口 IID 并转型指针。这关失败非常罕见（基本只在系统文件损坏时），但按规矩照样查。

**第 2 关：枚举显卡和显示器。**

```cpp
if (FAILED(impl_->factory->EnumAdapters1(0, &impl_->adapter))
    || FAILED(impl_->adapter->EnumOutputs(0, &impl_->output)))
    return fallbackToGdi("枚举显示器失败，使用 GDI 采集");
```

`EnumAdapters1(0, ...)`：枚举第 0 块显卡；`EnumOutputs(0, ...)`：枚举这块显卡的第 0 台显示器。注意这里写死了索引 0——**只支持主显示器**。多屏用户想抓副屏？做不到，这是已登记的局限（第十一节细谈）。

**第 3 关：读分辨率并校验。**

```cpp
DXGI_OUTPUT_DESC outputDesc = {};
impl_->output->GetDesc(&outputDesc);
impl_->width = outputDesc.DesktopCoordinates.right
               - outputDesc.DesktopCoordinates.left;
impl_->height = outputDesc.DesktopCoordinates.bottom
                - outputDesc.DesktopCoordinates.top;
if (impl_->width <= 0 || impl_->height <= 0)
    return fallbackToGdi("显示器尺寸无效，使用 GDI 采集");
```

显示器描述结构里的 `DesktopCoordinates` 是一个 RECT（矩形），表示这台显示器在虚拟桌面坐标系里占的位置。右减左得宽、下减上得高。注意这里拿到的是**物理像素**分辨率——DXGI 路径天然免疫 2.10 节讲的 DPI 问题。

为什么还要校验 `<= 0`？防御性编程：枚举出来的东西未必真连着显示器（比如显卡驱动虚报的输出口），尺寸异常时宁可降级。

**第 4 关：QueryInterface 拿升级接口。**

```cpp
if (FAILED(impl_->output->QueryInterface(IID_PPV_ARGS(&impl_->output1))))
    return fallbackToGdi("获取 IDXGIOutput1 失败，使用 GDI 采集");
```

2.4 节讲过：`DuplicateOutput` 只在 `IDXGIOutput1` 上。COM 换脸，一套 IID_PPV_ARGS 搞定。

**第 5 关：创建 D3D11 设备。**

```cpp
if (FAILED(D3D11CreateDevice(impl_->adapter, D3D_DRIVER_TYPE_UNKNOWN, nullptr,
                             0, nullptr, 0, D3D11_SDK_VERSION,
                             &impl_->device, nullptr, &impl_->context)))
    return fallbackToGdi("创建 D3D11 设备失败，使用 GDI 采集");
```

参数逐个解释：

- `impl_->adapter`：在**这块显卡**上创建设备——必须和显示器同一块卡，否则跨卡拷贝麻烦大了；
- `D3D_DRIVER_TYPE_UNKNOWN`：意思是"驱动类型我自己指定了 adapter，你别再帮我挑"。如果传 `HARDWARE`，D3D 会自己选默认显卡，可能和我们枚举的不是同一块；
- 第三个参数：软件驱动模块句柄，硬件路径用不到，nullptr；
- 第四个参数：标志位，0（调试时可以加 `D3D11_CREATE_DEVICE_DEBUG`）；
- 中间三个：特性级别列表——传 nullptr + 0 表示"默认就好"；
- `D3D11_SDK_VERSION`：SDK 版本常量，固定写法；
- 最后三个输出参数：设备、（特性级别，我们不关心，nullptr）、上下文。

**第 6 关：创建桌面复制会话——最容易失败的一关。**

```cpp
if (FAILED(impl_->output1->DuplicateOutput(impl_->device, &impl_->duplication)))
    return fallbackToGdi(
        "Desktop Duplication 不可用（可能被其它录屏软件占用），使用 GDI 采集");
```

注意日志文案直接写了"可能被其它录屏软件占用"——因为**桌面复制是独占资源**，同一时刻只能有一个进程对同一台显示器开复制会话。如果你机器上正开着 OBS、某些远控软件，这关就过不去。此外远程桌面会话、某些安全软件也会让它失败。日志把最可能的原因写出来，是降低排查成本的细节功夫。

**终章：创建 staging 纹理。**

```cpp
D3D11_TEXTURE2D_DESC stagingDesc = {};
stagingDesc.Width = impl_->width;
stagingDesc.Height = impl_->height;
stagingDesc.MipLevels = 1;
stagingDesc.ArraySize = 1;
stagingDesc.Format = DXGI_FORMAT_B8G8R8A8_UNORM;
stagingDesc.SampleDesc.Count = 1;
stagingDesc.Usage = D3D11_USAGE_STAGING;
stagingDesc.CPUAccessFlags = D3D11_CPU_ACCESS_READ;
if (FAILED(impl_->device->CreateTexture2D(&stagingDesc, nullptr,
                                          &impl_->staging)))
    return fallbackToGdi("创建 staging 纹理失败，使用 GDI 采集");
```

纹理描述结构逐字段：

- 宽高 = 屏幕分辨率；
- `MipLevels = 1`、`ArraySize = 1`：不要 mipmap 链、不要纹理数组，就一张平面；
- `Format = DXGI_FORMAT_B8G8R8A8_UNORM`：**每像素 4 字节，B、G、R、A 各一个字节**，UNORM 表示 0~255 归一化整数。这正是 `VideoFrame.bgra` 约定的格式，后面拷贝零转换；
- `SampleDesc.Count = 1`：不开多重采样；
- 最后两个字段是 staging 纹理的身份证（2.2 节背过）：`USAGE_STAGING` + `CPU_ACCESS_READ`。

第二参数 nullptr 表示"不给初始数据"，纹理内容后面由 CopyResource 灌。

**收官：**

```cpp
dxgiReady_ = true;
if (impl_->log)
    impl_->log("DXGI 屏幕采集已启动: " + std::to_string(impl_->width) + "x"
               + std::to_string(impl_->height));
return true;
```

六关全过，亮旗 `dxgiReady_ = true`，记一条带分辨率的成功日志。注意这条日志的文案——测试程序第一段就是靠**搜这条日志**来判断当前走的是哪条路，日志文案也是接口契约的一部分，别随手改。

---

## 六、逐函数教学：close() 与 capture() 调度

`init()` 讲完了，配对的 `close()` 很简单，但有一个顺序细节值得说：

```cpp
void ScreenCapturer::close()
{
    safeRelease(impl_->staging);
    safeRelease(impl_->duplication);
    safeRelease(impl_->context);
    safeRelease(impl_->device);
    safeRelease(impl_->output1);
    safeRelease(impl_->output);
    safeRelease(impl_->adapter);
    safeRelease(impl_->factory);

    impl_->width = 0;
    impl_->height = 0;
    impl_->lastFrame.reset();
    dxgiReady_ = false;
}
```

对照 init 的创建顺序，你会发现 **Release 的顺序正好反过来**：先建的后放、后建的先放。这是 COM 资源释放的通用礼仪——先释放"依赖者"，再释放"被依赖者"（staging 依赖 device，duplication 依赖 output1……）。虽然现代驱动多半容忍乱序，但按逆序释放是零成本的好习惯。

`safeRelease` 内部"Release + 置空"两步走，所以 `close()` 可以随便重复调——第二次调时所有指针已经是 nullptr，什么都不发生。**幂等的 close 是容错设计的基石**：init 开头敢无脑调 close，降级时敢无脑调 close，靠的就是它。

然后是 `capture()`——对外唯一的取帧入口，它的职责不是抓帧，而是**调度**：

```cpp
VideoFramePtr ScreenCapturer::capture()
{
    if (dxgiReady_) {
        if (VideoFramePtr frame = captureDxgi())
            return frame;

        dxgiReady_ = false;
        if (impl_->log)
            impl_->log("DXGI 采集异常，已切换 GDI 采集");
    }
    return captureGdi();
}
```

逻辑只有三行：

1. DXGI 可用就先走 DXGI，拿到帧就回家；
2. DXGI 返回空（运行期失败，比如锁屏触发 ACCESS_LOST）→ **永久降级**：`dxgiReady_` 置 false，记日志，然后转 GDI；
3. 之后每次 capture 都直接走 GDI，**不再尝试恢复 DXGI**。

注意"永久降级"这个决策：哪怕只是锁屏一秒钟导致的暂时失效，我们也一辈子用 GDI 直到下次 init。这是简单可靠的保守策略——切路径这件事做多了容易出状态错乱，不如一刀切。代价是锁屏之后画质性能回不到 DXGI。这个权衡我们在第十一节会重新审视（更优解是 ACCESS_LOST 时重建 duplication）。

`captureGdi()` 是个一行转发：

```cpp
VideoFramePtr ScreenCapturer::captureGdi()
{
    return gdiCapture();
}
```

真正的 GDI 实现在无名命名空间里的自由函数 `gdiCapture()`（第八节讲）。为什么把它做成自由函数而不是成员函数？因为它**完全无状态**——不碰 Impl 里任何成员，自给自足。无状态的逻辑就该是无状态的形态，这样哪天想把它抽成公共工具都方便。

---

## 七、逐函数教学：captureDxgi() 全流程

这是本讲技术含量最高的一个函数，我们放慢速度，分四幕讲。

**第一幕：取帧与四种返回码分支。**

```cpp
VideoFramePtr ScreenCapturer::captureDxgi()
{
    IDXGIResource* desktopResource = nullptr;
    DXGI_OUTDUPL_FRAME_INFO frameInfo = {};

    const HRESULT acquireHr =
        impl_->duplication->AcquireNextFrame(0, &frameInfo, &desktopResource);
```

`AcquireNextFrame` 三个参数：超时毫秒数、帧信息输出、桌面纹理资源输出。

这里超时传的是 **0**——"不等待，没有新帧立刻返回"。这意味着上层必须自己把握调用节奏（每 33ms 来一次），节奏乱了要么拿到重复帧要么错过变化。这个选择的利弊第十一节再讨论。

返回值的四种情况，对应四个分支：

```cpp
if (acquireHr == DXGI_ERROR_WAIT_TIMEOUT) {
    if (impl_->lastFrame)
        return std::make_shared<VideoFrame>(*impl_->lastFrame);
    return nullptr;
}
```

**分支一：WAIT_TIMEOUT——屏幕静止，没有新帧。**
这是最常见、也最"正常"的失败。应对就是 2.5 节讲的桥：有缓存就返回**上一帧的深拷贝**。注意 `std::make_shared<VideoFrame>(*impl_->lastFrame)` 调的是 VideoFrame 的拷贝构造——像素 vector 整个复制一份，全新对象。为什么必须深拷贝而不是直接把 `lastFrame` 这个 shared_ptr 交出去？因为下游（编码线程）可能持有帧很久、甚至可能改它；如果共享同一份数据，上游下一帧缓存更新时就把下游手里的数据改了。当然，深拷贝有代价——1080p 一帧 8.3MB，静止时每秒 30 次深拷贝约 250MB/s 的内存带宽白白烧掉。这是登记在册的改进点（第十一节第 4 条）。

如果连缓存都没有（刚启动、屏幕就一直静止），只能返回 nullptr。

```cpp
if (acquireHr == DXGI_ERROR_ACCESS_LOST) {
    if (impl_->log)
        impl_->log("DXGI 桌面连接丢失");
    return nullptr;
}
```

**分支二：ACCESS_LOST——复制会话失效。**
锁屏、切用户、远程桌面断开、显示驱动重置都会触发。返回 nullptr，让 `capture()` 的调度逻辑把我们永久降级到 GDI。

```cpp
if (FAILED(acquireHr)) {
    if (impl_->log)
        impl_->log("DXGI 取帧失败");
    return nullptr;
}
```

**分支三：其它一切失败。**兜底记日志，返回 nullptr。

走完三个失败分支还活着，就是第四种情况：`S_OK`，手里有了一张新桌面纹理。

**第二幕：换脸、验格式。**

```cpp
ID3D11Texture2D* desktopTexture = nullptr;
if (FAILED(desktopResource->QueryInterface(IID_PPV_ARGS(&desktopTexture)))) {
    desktopResource->Release();
    impl_->duplication->ReleaseFrame();
    return nullptr;
}
```

`AcquireNextFrame` 给我们的是 `IDXGIResource*`（通用资源接口），要干活得换成 `ID3D11Texture2D*`（纹理接口）。

**同学们注意这里的失败处理**：`desktopResource->Release()` + `duplication->ReleaseFrame()`，一步不能少。为什么？因为 AcquireNextFrame 成功的那一刻起，我们就欠了两笔债：

1. 桌面资源 resource 要 Release（COM 规矩）；
2. 这一帧要 `ReleaseFrame()` 归还——**这是 Desktop Duplication 的特殊纪律**：Acquire 和 ReleaseFrame 必须严格配对，上一帧不还，下一次 Acquire 直接失败。

```cpp
D3D11_TEXTURE2D_DESC texDesc = {};
desktopTexture->GetDesc(&texDesc);

if (texDesc.Format != DXGI_FORMAT_B8G8R8A8_UNORM) {
    desktopTexture->Release();
    desktopResource->Release();
    impl_->duplication->ReleaseFrame();
    return nullptr;
}
```

验一下纹理格式。理论上桌面纹理永远是 BGRA8，但防御性地检查一次——万一是别的格式（比如某些 HDR 场景下的 16 位浮点格式），我们的逐行 memcpy 逻辑就不成立，与其拷出花屏不如放弃这一帧。注意失败分支里又是**完整的三件套**（texture、resource、ReleaseFrame）。

**第三幕：GPU 搬运与 CPU 读取。**

```cpp
impl_->context->CopyResource(impl_->staging, desktopTexture);
```

摆渡车开动（回忆 2.2 节）：GPU 内部把桌面纹理整张拷到 staging 纹理。显存到显存，极快，CPU 完全不参与。

```cpp
D3D11_MAPPED_SUBRESOURCE mapped = {};
if (FAILED(impl_->context->Map(impl_->staging, 0, D3D11_MAP_READ, 0, &mapped))) {
    desktopTexture->Release();
    desktopResource->Release();
    impl_->duplication->ReleaseFrame();
    return nullptr;
}
```

Map = 拿中转仓库的钥匙。`D3D11_MAP_READ` 声明"我只读"。成功后 `mapped.pData` 是 CPU 可读的指针，`mapped.RowPitch` 是行跨度（2.6 节敲过黑板的知识点）。Map 失败？三件套伺候，放弃本帧。

**第四幕：逐行拷贝与资源归还。**

```cpp
auto frame = std::make_shared<VideoFrame>();
frame->width = static_cast<int>(texDesc.Width);
frame->height = static_cast<int>(texDesc.Height);
frame->bgra.resize(static_cast<size_t>(frame->width) * frame->height * 4);

const uint8_t* src = static_cast<const uint8_t*>(mapped.pData);
uint8_t* dst = frame->bgra.data();
const size_t rowBytes = static_cast<size_t>(frame->width) * 4;
for (int y = 0; y < frame->height; ++y) {
    std::memcpy(dst, src, rowBytes);
    src += mapped.RowPitch;
    dst += rowBytes;
}
```

注意帧的宽高用的是**纹理描述里的宽高**而不是 init 时存的屏幕宽高——以更权威的实物为准。然后就是那个口诀的活教材：

```
源按 Pitch 走（src += mapped.RowPitch）   ← 跟着 GPU 的补齐排布跳行
目标按宽走（dst += rowBytes = width*4）   ← 我们的 vector 紧密排列
```

每行只拷 `width*4` 字节的有效像素，行末的填充垃圾被自然跳过。**这个循环是"斜纹"与"正常图"的分水岭。**

拷完之后的归还动作，顺序固定：

```cpp
impl_->context->Unmap(impl_->staging, 0);    // 还钥匙

desktopTexture->Release();                    // 还债 1
desktopResource->Release();                   // 还债 2

impl_->duplication->ReleaseFrame();           // 还债 3（归还帧）

impl_->lastFrame = frame;                     // 更新静止缓存
return frame;
```

三件事收尾：

1. `Unmap` 还钥匙——Map/Unmap 也是一对纪律；
2. COM 双 Release + ReleaseFrame，和失败分支里的三件套一模一样，只是位置在成功路径末尾；
3. `impl_->lastFrame = frame`——shared_ptr 赋值，新帧成为静止时的候补；旧的 lastFrame 引用计数减一，如果没人持有它就自动释放。内存管理零手工。

老师再强调一遍这个函数的**资源配对纪律**，它是本函数最容易写错的地方：

> 从 AcquireNextFrame 成功那一刻起，每一笔"拿到"都必须有对应的"归还"，**无论你从哪条路径退出函数**。代码里三处失败分支各自老老实实重复三件套，一处都不能省。如果以后维护时在这里加新的提前 return，请先把三件套抄上。

有同学问：为什么不用 RAII 封装自动释放？问得好——这是正经的改进方向（比如 `wil::com_ptr` 或自写小包装），本项目为了保持"零依赖、看得见每一步"的教学风格，选择了手工配对。学习阶段先看清每一笔债，工程化阶段再谈封装。

---

## 八、逐函数教学：gdiCapture() 逐行讲

兜底路径，住在无名命名空间里（不对外暴露，纯内部工具）。我们逐行走：

```cpp
VideoFramePtr gdiCapture()
{
    const int w = GetSystemMetrics(SM_CXSCREEN);
    const int h = GetSystemMetrics(SM_CYSCREEN);
    if (w <= 0 || h <= 0)
        return nullptr;
```

`GetSystemMetrics` 拿主屏尺寸。复习 2.10 节：这是**逻辑像素**，高 DPI 机器上可能小于物理分辨率——已知局限，先记着。尺寸异常直接空手而归。

```cpp
HDC screenDc = GetDC(nullptr);
HDC memDc = CreateCompatibleDC(screenDc);
HBITMAP bmp = CreateCompatibleBitmap(screenDc, w, h);
HGDIOBJ oldBitmap = SelectObject(memDc, bmp);
```

五件套的前四件（2.7 节背过）：

1. `GetDC(nullptr)`：全屏 DC；
2. `CreateCompatibleDC`：内存虚拟画布；
3. `CreateCompatibleBitmap(screenDc, w, h)`：和屏幕兼容的画纸，屏幕大小；
4. `SelectObject(memDc, bmp)`：画纸铺上画布。**返回值 `oldBitmap` 是换下来的旧画纸，必须存好**——释放阶段要把它选回去。

```cpp
BitBlt(memDc, 0, 0, w, h, screenDc, 0, 0, SRCCOPY | CAPTUREBLT);
```

第五件，真正的"截屏"动作。参数语义：往 memDc 的 (0,0) 位置，拷贝 w×h 大小，源是 screenDc 的 (0,0)，光栅操作 `SRCCOPY`（直接覆盖）| `CAPTUREBLT`（连分层窗口一起抓，2.7 节讲过）。一行调用，屏幕图像进了 bmp。

```cpp
BITMAPINFO bmi = {};
bmi.bmiHeader.biSize = sizeof(BITMAPINFOHEADER);
bmi.bmiHeader.biWidth = w;
bmi.bmiHeader.biHeight = -h;
bmi.bmiHeader.biPlanes = 1;
bmi.bmiHeader.biBitCount = 32;
bmi.bmiHeader.biCompression = BI_RGB;
```

配 `GetDIBits` 的输出格式要求：

- `biSize`：头大小，Windows API 的通用"版本号"写法；
- `biHeight = -h`：**那个身价最高的负号**（2.9 节）——要顶向下数据，不要颠倒图；
- `biBitCount = 32`：每像素 32 位（BGRA 四字节）；
- `biCompression = BI_RGB`：不压缩；
- `biPlanes = 1`：历史字段，永远 1。

```cpp
auto frame = std::make_shared<VideoFrame>();
frame->width = w;
frame->height = h;
frame->bgra.resize(static_cast<size_t>(w) * h * 4);

GetDIBits(memDc, bmp, 0, h, frame->bgra.data(), &bmi, DIB_RGB_COLORS);
```

造帧、按 w×h×4 扩容 vector、然后 `GetDIBits` 把位图像素按我们指定的格式**直接灌进 vector 的内存**。`DIB_RGB_COLORS` 表示"调色板项是 RGB 值"（32 位位图其实用不到调色板，但参数得给）。注意这里没有 RowPitch 的烦恼——GetDIBits 会按我们给的宽度紧凑输出，API 帮我们处理了。

```cpp
SelectObject(memDc, oldBitmap);
DeleteObject(bmp);
DeleteDC(memDc);
ReleaseDC(nullptr, screenDc);
return frame;
```

释放四步曲，一步不少（2.7 节）：旧位图选回去 → 删我们的位图 → 删内存 DC → 归还屏幕 DC。漏任何一步都是 GDI 句柄泄漏，泄漏几千次之后整个系统的界面都会抽风。

和 DXGI 路径对比一下两个函数的气质：

- `captureDxgi()`：资源是**持久的**（init 建好，反复用），每帧只做"取帧 → 摆渡 → 拷出 → 归还"；
- `gdiCapture()`：资源是**一次性的**，每帧都重建 DC、重建位图、再销毁。开销不小——但它是兜底路径，"能出图"已经完成任务，性能不是它的 KPI。

---

## 九、测试程序教学

### 9.1 测试设计思路：为什么屏幕采集一半正确性只能人眼核对

同学们，讲测试之前，老师先问一个问题：**屏幕采集的测试，能不能像普通单元测试那样"断言一切"？**

不能。想想采集器的产品是什么——一张桌面截图。什么叫"对"？断言分辨率对、像素不是全黑，这些机器能判；但**"抓到的画面到底是不是你的桌面"**，只有人眼能判。电脑不知道你桌面上开着什么窗口。

所以这份测试的设计是**组合拳**：

> **能自动断言的全部自动断言（分辨率、空帧数、尺寸一致性、亮度合理性），不能断言的把图片存下来，打印出检查清单，让人眼做终审。**

五段测试，层层递进：

1. **初始化**：init 是否成功、走的是哪条路；
2. **连续抓 60 帧**：有效性、尺寸一致性、性能、亮度分布；
3. **静止画面 200 连抓**：验证 lastFrame 兜底机制，一个空帧都不许有；
4. **画面内容检查**：非全黑、非全白——机器能做的"内容合理性"粗判；
5. **保存图片**：首帧末帧存 BMP，人眼核对三大经典症状。

### 9.2 测试基建：check、日志收集、计时

```cpp
namespace {

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

极简断言框架：两个全局计数器 + 一个 check 函数。不搞 Google Test，保持项目"零外部依赖"的风格。

main 开头的两行环境准备也值得讲：

```cpp
#if defined(_WIN32)
    std::system("chcp 65001 > nul");
#endif

static char stdoutBuffer[8192];
std::setvbuf(stdout, stdoutBuffer, _IOLBF, sizeof(stdoutBuffer));
```

- `chcp 65001`：把控制台代码页切到 UTF-8，否则中文日志在 Windows 控制台全是乱码；
- `setvbuf` 设行缓冲：保证 printf 的输出及时可见（程序崩溃时缓冲区里的日志不至于丢失）。

日志回调的处理是个亮点：

```cpp
std::vector<std::string> logs;
const auto log = [&logs](const std::string& message) {
    logs.push_back(message);
    std::printf("  [采集器日志] %s\n", message.c_str());
};
```

日志**既打印又存档**——存进 vector 供后面检索。这是契约层 `LogCallback` 设计的第一次实战：采集器通过回调吐日志，调用方想怎么用就怎么用。马上我们就用存档判断路径：

```cpp
bool usingDxgi = false;
for (const std::string& line : logs) {
    if (line.find("DXGI 屏幕采集已启动") != std::string::npos)
        usingDxgi = true;
}
std::printf("  当前使用路径: %s\n", usingDxgi ? "DXGI" : "GDI 兜底");
```

靠搜 init 成功日志的文案来判断当前路径。这也是为什么第五节老师说"日志文案是契约的一部分"——改了文案，测试的判断逻辑就瞎了。

计时小工具：

```cpp
long long elapsedMs(std::chrono::steady_clock::time_point from)
{
    return std::chrono::duration_cast<std::chrono::milliseconds>(
               std::chrono::steady_clock::now() - from)
        .count();
}
```

`steady_clock`（单调时钟，不受系统时间调整影响）——测耗时永远用它，不用 `system_clock`。

### 9.3 saveBmp：手写 BMP 文件

2.8 节预习过格式，现在看实现。先是用 `#pragma pack(push, 1)` 关闭对齐的两个头部结构体，字段和 BMP 文件字节布局一一对应。然后：

```cpp
bool saveBmp(const std::string& path, const VideoFrame& frame)
{
    if (frame.width <= 0 || frame.height <= 0
        || frame.bgra.size() < static_cast<size_t>(frame.width) * frame.height * 4)
        return false;
```

入口防御：尺寸要正、像素数组要够长（**`>=` 而不是 `==`**——容忍 vector 比理论值大，只要不短缺）。

```cpp
const uint32_t imageBytes =
    static_cast<uint32_t>(frame.width) * frame.height * 4;

BmpFileHeader fileHeader = {};
fileHeader.type = 0x4D42;                 // "BM" 魔数
fileHeader.size = sizeof(BmpFileHeader) + sizeof(BmpInfoHeader) + imageBytes;
fileHeader.offBits = sizeof(BmpFileHeader) + sizeof(BmpInfoHeader);

BmpInfoHeader infoHeader = {};
infoHeader.size = sizeof(BmpInfoHeader);
infoHeader.width = frame.width;
infoHeader.height = frame.height;
infoHeader.planes = 1;
infoHeader.bitCount = 32;
infoHeader.compression = 0;               // BI_RGB
infoHeader.sizeImage = imageBytes;
infoHeader.xPelsPerMeter = 2835;
infoHeader.yPelsPerMeter = 2835;
```

填两个头。注意 `infoHeader.height` 这里是**正值**——配合下面倒序写行，存出来的就是标准自底向上 BMP。`xPelsPerMeter = 2835` 是每米像素数，2835 ≈ 72 DPI，纯惯例值，写多少都不影响看图。

```cpp
std::FILE* file = std::fopen(path.c_str(), "wb");
if (!file)
    return false;

std::fwrite(&fileHeader, sizeof(fileHeader), 1, file);
std::fwrite(&infoHeader, sizeof(infoHeader), 1, file);

const size_t stride = static_cast<size_t>(frame.width) * 4;
for (int y = frame.height - 1; y >= 0; --y)
    std::fwrite(frame.bgra.data() + static_cast<size_t>(y) * stride, stride, 1,
                file);

std::fclose(file);
return true;
```

`"wb"` 二进制模式打开（Windows 上文本模式会篡改 `\n`，写二进制文件必须带 b）。两个头原样 fwrite——**这就是 `#pragma pack(1)` 的意义**：结构体内存布局 = 文件字节布局，一把 fwrite 直出。然后像素**从最后一行倒着写到第 0 行**（自底向上，2.8 节）。

### 9.4 averageBrightness：亮度统计

```cpp
int averageBrightness(const VideoFrame& frame)
{
    const size_t pixelCount = static_cast<size_t>(frame.width) * frame.height;
    if (pixelCount == 0)
        return 0;

    uint64_t sum = 0;
    const uint8_t* p = frame.bgra.data();
    for (size_t i = 0; i < pixelCount; ++i) {
        sum += p[i * 4 + 0];
        sum += p[i * 4 + 1];
        sum += p[i * 4 + 2];
    }
    return static_cast<int>(sum / (pixelCount * 3));
}
```

把所有像素的 B、G、R 三个通道值加起来求平均（A 通道不参与）。两个细节：

- `sum` 用 `uint64_t`：1920×1080×3×255 上限约 16 亿，超 int32 上限一半，必须用 64 位防溢出；
- 这不是严格的"亮度"（人眼对绿光最敏感，标准亮度公式是 0.299R+0.587G+0.114B），只是粗粗的通道平均——但对"是不是全黑""是不是全白"这种判断够用了。测试不是图像处理，别过度设计。

它的用途：亮度 0~1 → 全黑帧（什么都没抓到或抓到黑屏）；亮度 255 → 全白（往往是内存没正确填充）；正常桌面一般落在中间区间。

### 9.5 五段测试正文

**第 1 段：初始化。**

```cpp
ScreenCapturer capturer;
const auto initStart = std::chrono::steady_clock::now();
const bool ok = capturer.init(log);
const long long initMs = elapsedMs(initStart);

check(ok, "init() 返回成功");
std::printf("  初始化耗时 %lld 毫秒\n", initMs);
```

init 失败直接 `return 1` 收场——后面所有测试都依赖采集器可用。

**第 2 段：连续抓 60 帧。**

循环里每帧做五件事：数空帧、记首帧分辨率、校验后续帧尺寸一致、算亮度并维护"首/末/最小/最大"四个统计值、数全黑帧。循环结束打印汇总：

```cpp
std::printf("  抓了 %d 次，得到有效帧 %d 个，空帧 %d 个\n", ...);
std::printf("  分辨率 %dx%d，总耗时 %lld 毫秒，平均每帧 %.2f 毫秒\n", ...);
std::printf("  亮度: 首帧 %d, 末帧 %d, 最小 %d, 最大 %d, 全黑帧数 %d/%d\n", ...);

check(nullFrames == 0, "没有空帧");
check(width > 0 && height > 0, "得到了有效的分辨率");
check(sizeConsistent, "所有帧的尺寸一致");
check(width >= 640 && height >= 480, "分辨率看起来是正常的桌面尺寸");
```

四个断言各有分工：空帧数为 0（哪怕静止也有 lastFrame 兜底）、分辨率有效、**所有帧尺寸一致**（采集器输出尺寸抖动对编码器是灾难）、分辨率大于等于 640×480（防止 DPI 问题抓到一块小不点还自以为成功）。

性能数据也在这一段：平均每帧耗时。DXGI 正常应该远低于 33ms（30fps 的预算），如果逼近或超过，说明路径或机器有问题。

### 9.6 静止 200 连抓：静止画面兜底行为

```cpp
int burstNulls = 0;
constexpr int kBurst = 200;
for (int i = 0; i < kBurst; ++i) {
    if (!capturer.capture())
        ++burstNulls;
}
std::printf("  连续快速抓 %d 次，空帧 %d 个\n", kBurst, burstNulls);
check(burstNulls == 0, "快速连抓也没有空帧（屏幕静止时返回上一帧的拷贝）");
```

这段测试的智慧在于：连续 200 次**不带任何间隔**地狂调 capture，人根本来不及动鼠标——屏幕几乎必然静止，DXGI 几乎必然超时。这正是 lastFrame 机制的主考场：**一个空帧都不许有**。第 2 段跑过 60 帧后 lastFrame 一定已缓存，所以这里合法结果只有满分。

**第 4 段：画面内容检查。**

```cpp
check(nonBlack > 0, "至少有一部分帧不是全黑（采集确实拿到了画面）");
check(maxBrightness < 255, "画面不是全白（全白往往说明内存没被正确填充）");
```

机器能做的两条"内容合理性"粗判。之后还有一段特别有人情味的提示：如果检测到全黑帧，打印一段解释——

```cpp
std::printf("  这是 DXGI Desktop Duplication 的已知行为：\n");
std::printf("  刚开始采集时，系统还没有合成出新的桌面画面，\n");
std::printf("  前几帧可能是空帧。...\n");
```

采集刚启动时 DWM 可能还没合成新画面，头几帧黑是已知现象。测试**把已知行为写成提示而不是断言失败**，这是写测试的成熟度：测试的职责是区分"病"和"正常现象"，不是见红就报。

**第 5 段：保存图片。**

```cpp
check(saveBmp("screen_capture_first.bmp", *firstFrame), ...);
check(saveBmp("screen_capture_last.bmp", *lastFrame), ...);
std::printf("\n  请打开这两个 bmp 文件，确认内容是你的桌面。\n");
std::printf("  检查要点：\n");
std::printf("    - 画面是否上下颠倒（GDI 路径最容易出这个问题）\n");
std::printf("    - 是否有斜纹（RowPitch 处理错了的典型症状）\n");
std::printf("    - 是否只抓到屏幕左上角一块（DPI 缩放问题）\n");
```

首帧和末帧各存一张——对比两张图能区分"只有头几帧黑"和"一直黑"。然后把三大经典症状打印成**人眼检查清单**。这份清单我们下一节专门用。

最后 `capturer.close()`、打印通过/失败总数、按 `g_failed == 0` 返回 0 或 1——返回码规范，方便接入 CI。

---

## 十、动手环节

### 10.1 编译运行

同学们，理论讲完了，动手。在项目根目录：

```bash
cmake -B build -G "Visual Studio 17 2022"
cmake --build build --config Debug
./build/tests/Debug/test_screen_capture.exe
```

（生成器名字按你机器上的 VS 版本调整。）

跑起来之后，盯着输出看三个地方：

1. **"当前使用路径"**是 DXGI 还是 GDI 兜底？正常 Win8+ 机器应该是 DXGI。如果是 GDI，往上翻日志，看六关里折在哪一关——最常见的是"被其它录屏软件占用"，把 OBS 之类的软件关掉再跑。
2. **平均每帧耗时**。DXGI 路径应该在几毫秒量级；如果接近 33ms，留意是不是走了 GDI。
3. **通过 N 项，失败 0 项**。有失败项先别看 bmp，直接回头查对应断言。

### 10.2 看 bmp 文件

测试结束后，在测试程序的工作目录下找到两个文件：

```
screen_capture_first.bmp
screen_capture_last.bmp
```

双击打开，对着屏幕问自己的第一个问题：**这是不是我的桌面？**

- 两张都是你的桌面、内容有细微差别（比如时钟变了）→ 完美；
- 两张一样 → 也正常，说明采集期间屏幕没动；
- first 全黑、last 正常 → 已知行为（DWM 还没合成出头几帧），不用慌；
- 两张都全黑 → 真有问题，回到输出看全黑帧统计。

### 10.3 三大经典症状自查清单

这一节是本讲的"临床手册"。打开 bmp，对照下表自查：

**症状一：画面上下颠倒。**

```
病根：方向处理错误。
├── GDI 路径：BITMAPINFO 的 biHeight 忘打负号（写的是 h 而不是 -h）
│   → 去 ScreenCapturer.cpp 的 gdiCapture() 检查 biHeight
└── 存图环节：saveBmp 的行循环写成了正序
    → 去 test_screen_capture.cpp 检查 for 循环是不是从 height-1 倒着写
```

**症状二：画面有斜纹（直线全变斜线，像被拧过）。**

```
病根：RowPitch 处理错误。
└── DXGI 路径：逐行 memcpy 时假设了 RowPitch == width*4
    → 去 captureDxgi() 检查拷贝循环：
      src 必须 += mapped.RowPitch，dst 必须 += width*4
      绝不能对 mapped.pData 做一次性整块 memcpy
```

**症状三：只抓到屏幕左上角一块（或画面模糊、分辨率偏小）。**

```
病根：DPI 缩放。
└── 你的显示器开了 125%/150%/200% 缩放，GDI 路径的
    GetSystemMetrics 拿到的是逻辑像素
    → 确认当前路径（看测试输出"当前使用路径"）
    → 短期规避：让程序走 DXGI 路径（物理分辨率，无此问题）
    → 根治：给程序声明 DPI awareness（见第十一节）
```

同学们请注意一个共同规律：**这三种病都不会崩溃、不会报错，程序跑得欢天喜地，只是图不对**。这就是为什么屏幕采集的测试必须"机器断言 + 人眼终审"双轨并行——静默错误是采集器最阴险的敌人。

---

## 十一、坑点与改进

老规矩，最后盘点这次实现登记的坑和未来的改进方向。同学们读别人的代码要养成这个习惯：能说出它哪里不够好，才算真读懂了。

1. **只抓主显示器。** `EnumAdapters1(0)` + `EnumOutputs(0)` 写死"第一块显卡的第一台显示器"。多屏用户想抓副屏、想抓跨屏虚拟桌面，都做不到。改进方向：把 adapter/output 索引做成 init 参数，或者提供枚举 API 让上层挑选。

2. **GDI 路径的 DPI 盲区。** 2.10 节讲过：没声明 DPI awareness 的进程，`GetSystemMetrics` 拿到逻辑像素，高缩放屏上抓不全或画面糊。改进方向：用 manifest 或 `SetProcessDpiAwarenessContext` 声明 Per-Monitor V2 感知。注意这是个**进程级**开关，会影响整个程序的 UI 行为，不能随手改。

3. **DXGI 失败后不自动恢复。** 一旦 `dxgiReady_ = false`，哪怕只是锁屏一秒触发的 ACCESS_LOST，也永远用 GDI 直到下次 init。改进方向：捕获 ACCESS_LOST 时**重建 duplication**——把 init 的后半段（DuplicateOutput + staging 纹理创建）抽成一个 `reinitDxgi()`，失败时重试几次再降级。

4. **超时复用帧是深拷贝。** 静止时每秒 30 次、每次 8.3MB 的 memcpy，约 250MB/s 内存带宽白白烧掉。改进方向：直接返回同一个 `shared_ptr`（零拷贝共享），但这要求下游承诺只读——契约层目前没有"只读帧"概念，这是下一个模块可以讨论的设计。

5. **每帧新分配 8.3MB vector。** 高频大块堆分配会造成堆碎片和性能抖动。改进方向：**帧缓冲池**（buffer pool）——帧用完后归还到池里复用，配上自定义 deleter 的 shared_ptr 就能优雅实现。

6. **AcquireNextFrame timeout=0 是轮询模型。** 上层必须以固定节奏调 capture()，节奏错了要么重复帧要么错过变化。改进方向：传一个小的非零超时（比如 100ms），让系统帮我们等新帧，采集线程可以阻塞在 Acquire 上自然对齐节奏。

7. **GDI 路径每次全量重建资源。** 每帧 CreateCompatibleBitmap + GetDIBits，分配销毁开销不小。兜底路径可容忍，但如果哪天 GDI 成为主力（比如老系统专项支持），应该把 DC 和位图缓存成成员。

8. **capture() 返回 nullptr 的语义模糊。** 可能是"屏幕静止且还没有第一帧缓存"、可能是"GDI 也失败了"、可能是"DXGI 刚挂且 GDI 也挂了"。上层无法区分。改进方向：返回 `optional<Frame>` 加错误码，或者定义 Result 类型。

9. **权限/企业策略场景无感知。** 某些受策略限制的企业环境里，采集可能静默返回黑屏——测试只能发现"全黑"但无法定位是权限问题。改进方向：日志里补充环境探测信息（是否远程会话、是否锁屏状态）。

---

## 十一·附、本讲速查表

讲课内容多，老师把本讲需要"印在脑子里"的东西浓缩成三张速查表，同学们可以截图保存。

**表一：两条路径对比速查**

| 维度 | DXGI Desktop Duplication | GDI BitBlt |
|---|---|---|
| 引入年代 | Windows 8 | Windows 95 |
| 取帧模型 | 变化推送（静止时超时） | 主动截屏（随叫随拍） |
| 数据在哪 | GPU 纹理（需 staging 中转） | 内存位图（GetDIBits 直出） |
| 分辨率 | 物理像素（免疫 DPI） | 逻辑像素（怕 DPI 缩放） |
| 独占性 | 是（被别的录屏软件占用则失败） | 否 |
| 一帧耗时 | 毫秒级 | 10~30ms（1080p） |
| 本讲角色 | 主路径 | 兜底路径 |

**表二：资源配对纪律速查（哪个分支退出都不能漏）**

| 拿到的 | 必须归还的 | 归还函数 |
|---|---|---|
| AcquireNextFrame 成功的帧 | ReleaseFrame | `duplication->ReleaseFrame()` |
| desktopResource | Release | COM 规矩 |
| desktopTexture（QueryInterface 换的） | Release | COM 规矩 |
| Map 拿到的指针 | Unmap | `context->Unmap(staging, 0)` |
| GetDC 拿的屏幕 DC | ReleaseDC | GDI 规矩 |
| CreateCompatibleDC 造的内存 DC | DeleteDC | GDI 规矩 |
| CreateCompatibleBitmap 造的位图 | DeleteObject（先 SelectObject 换回去） | GDI 规矩 |
| init 建的 8 个 COM 指针 | close() 里逆序 safeRelease | 析构/降级/重 init |

**表三：症状 → 病根 → 定位 速查**

| bmp 里的症状 | 病根 | 去哪个函数查什么 |
|---|---|---|
| 上下颠倒 | 方向处理错 | `gdiCapture()` 的 `biHeight` 有没有负号；`saveBmp` 的行循环是不是倒序 |
| 斜纹 | RowPitch 错 | `captureDxgi()` 拷贝循环：src += RowPitch、dst += width*4 |
| 只有左上角一块 | DPI 缩放 | 看测试输出走的是不是 GDI 路径；显示器缩放是否 >100% |
| 头几帧全黑、后面正常 | DWM 尚未合成新帧 | 正常现象，不用修 |
| 一直全黑 | 采集真的失败/权限受限 | 看日志折在哪一关；检查是否有其它录屏软件占用 |

**本讲金句回顾**：

> 有画面，比画面来源高级重要。
>
> 厂子办卡，卡里选屏，屏上复制。（DXGI 对象链）
>
> 源按 Pitch 走，目标按宽走；整块 memcpy，斜纹管够。
>
> biHeight 的负号，是 GDI 截屏代码里身价最高的一个字符。
>
> 静默错误是采集器最阴险的敌人——所以测试要机器断言和人眼终审双轨并行。

### 10.4 课堂演示脚本（老师版）

如果同学们将来要给别人演示这个模块，可以按下面这个脚本走，五分钟讲完全部亮点：

1. **先跑测试**（30 秒）：指着"当前使用路径: DXGI"说——看，我们优先走快车道。
2. **打开 last.bmp**（30 秒）：这就是程序眼里看到的桌面，和真实桌面一字不差。
3. **演示静止兜底**（1 分钟）：双手离开键盘鼠标，重跑测试，指着第 3 段"空帧 0 个"说——屏幕一动不动，帧照样每秒 30 张地出，靠的就是 lastFrame 缓存。这就是"变化推送"模型适配"恒定帧率"需求的桥。
4. **演示降级哲学**（1 分钟）：先开一个占用 Desktop Duplication 的录屏软件（比如 OBS 开始录制），再跑测试。日志里会出现"Desktop Duplication 不可用……使用 GDI 采集"，然后"当前使用路径: GDI 兜底"，但测试照样全部通过。指着说——**用户看不到任何报错，画面照出**，这就是"有画面比画面来源高级重要"。
5. **演示三大症状**（2 分钟，需要改代码再编译）：
   - 把 `biHeight = -h` 的负号删掉 → 重跑 → 打开 bmp，全场看到颠倒的桌面；
   - 把逐行拷贝循环改成一次性 `memcpy(dst, src, rowBytes * height)` → 重跑 → 斜纹登场（如果机器 RowPitch 恰好对齐则演示不出，提前在自己机器上验证）；
   - 把显示器缩放调到 150%，强制走 GDI → 只抓到左上角。
   每演示一个，就回到 10.3 的自查清单，把"症状—病根—定位"串一遍。

这个脚本的核心思想：**先证明它正常工作，再故意弄坏它**。学员亲眼看到"一个负号毁掉整张图"，比背十遍知识点都印象深刻。

### 10.5 常见现场问答

**问：为什么不用现成的截图库（比如 Windows.Graphics.Capture）？**
答：那是 UWP/WinRT 时代的 API，带系统授权弹窗、依赖运行时组件，对我们这种"零依赖纯 Win32 教学项目"来说太重。而且 DXGI + GDI 是所有录屏软件（OBS 等）的底层正统路线，学它不亏。

**问：capture() 在采集线程里多久调一次？**
答：目标 30fps，即约每 33ms 一次。本讲还没引入采集线程，是测试程序用 for 循环手动驱动的；等后面流水线模块进来，会有专门的采集线程按固定节奏调它。

**问：为什么 VideoFrame 用 shared_ptr 而不是直接传值？**
答：一帧 8.3MB，传值就是一次大拷贝；shared_ptr 传的是引用计数，帧的所有权可以安全地跨线程移交（下一步就是 push 进有界队列给编码线程）。这是契约层早就定下的规矩，本讲只是第一次真正兑现它。

**问：GDI 路径能抓到鼠标指针吗？**
答：BitBlt 默认不画指针。要抓指针需要另外 `GetCursorInfo` + `DrawIconEx` 手工合成上去。本讲没做，属于锦上添花，登记为可选改进。

---

## 十二、小结

同学们，这一讲我们干了什么？回顾一下：

- 我们搞清楚了屏幕采集的本质：**把显卡里的桌面图像搬到 CPU 内存**，每秒 30 次；
- 我们认识了 Windows 上的两代采集技术：**DXGI Desktop Duplication**（快、变化推送、Win8+）和 **GDI BitBlt**（慢、主动截屏、到处能用）；
- 我们预习了一整套底层知识：DWM 合成器、staging 纹理摆渡、COM 三规矩（引用计数、HRESULT、IID_PPV_ARGS）、DXGI 对象链、RowPitch 与斜纹、GDI 五件套、BMP 自底向上、biHeight 负号、DPI 缩放；
- 我们逐行读完了 `ScreenCapturer`：**init() 的六关与降级哲学**（有画面比画面来源高级重要）、**capture() 的调度**、**captureDxgi() 的资源配对纪律**（Acquire/ReleaseFrame、Map/Unmap、COM 双 Release，哪个分支退出都不能漏）、**gdiCapture() 的五件套**；
- 我们读懂了测试的组合拳设计：机器断言能断的一切（空帧、尺寸、亮度），人眼终审断不了的（三大经典症状），并且动手跑了一遍、看了 bmp。

至此，我们的推流项目有了第一个"画面源头"。下一讲，我们将给项目配上第二个源头——**音频采集（WASAPI）**，到时候大家会发现，COM 的三条规矩、回调日志的设计、双路径的哲学，全都会原样回来和我们见面。

---

## 十三、思考题

1. `init()` 里降级 GDI 时返回的是 `true` 而不是 `false`。请用一句话说明这个设计选择背后的产品哲学，并说说如果改成返回 false，上层调用代码会发生什么连锁反应。

2. `captureDxgi()` 里有三处失败分支都重复写着 `desktopTexture->Release(); desktopResource->Release(); impl_->duplication->ReleaseFrame();`。如果让你用 RAII 消除这三处重复，你会怎么设计？（提示：想想"作用域守卫"或者给 COM 指针写个 unique_ptr 自定义 deleter。）

3. 为什么静止画面时我们返回 `lastFrame` 的**深拷贝**，而不是直接把 `lastFrame` 这个 shared_ptr 返回出去？如果要改成零拷贝共享，需要对下游做什么约定？这个约定适合放在哪一层表达？

4. `AcquireNextFrame` 的超时参数本讲传的是 0。如果改成 16（毫秒），采集线程的行为会发生什么变化？对上层的调用节奏设计有什么影响？

5. GDI 路径里 `biHeight = -h` 控制 GetDIBits 输出顶向下数据；而 `saveBmp` 里 `infoHeader.height = frame.height`（正值）配合倒序写行。两个地方都在处理"方向"，为什么一个用负号、一个用正号加倒序？如果 saveBmp 里也用负的高度，行循环该怎么改？

6. 你的显示器开了 150% DPI 缩放，物理分辨率 3840×2160。在没声明 DPI 感知的情况下，GDI 路径的 `GetSystemMetrics(SM_CXSCREEN)` 大概会返回多少？抓出来的 bmp 会是什么样？

7. 测试第 3 段"静止 200 连抓"为什么要求**一个空帧都不许有**？如果允许有少量空帧（比如 ≤5 个），会掩盖什么 bug？

8. 假设你要支持"采集副屏"，基于现有代码，最小改动方案是什么？需要动哪几个函数？

---

> 下一讲预告：第 5 讲 —— 音频采集（WASAPI  loopback 与麦克风），我们将再次见到 COM 引用计数、日志回调契约，以及一个全新概念：环形缓冲区。

---

## 附：本讲"五章对照索引"

| 标准章节 | 在本讲的位置 | 一句话要点 |
|---|---|---|
| **核心原理** | 开场白 + 第二章知识点预习（10 节） | 屏幕采集=把 GPU 里的桌面图像搬到 CPU 内存；DWM 合成、staging 纹理回读、"变化推送"vs"主动截屏"、RowPitch 与斜纹、屏幕静止时的 lastFrame 复用 |
| **用到的技术 / 知识点** | 第二章全部 + 十一·附速查表 | COM 引用计数 / HRESULT / IID_PPV_ARGS、DXGI 对象链 Factory→Adapter→Output→Duplication、D3D11 设备与上下文、PIMPL、GDI 五件套、CAPTUREBLT、BMP 文件结构与自底向上、biHeight 负号、DPI 缩放 |
| **实现细节** | 第四~九章逐函数教学 | ScreenCapturer.h 契约；init() 六关逐级降级；close() 逆序释放；capture() 调度；captureDxgi() 四分支 + CopyResource→Map→逐行 memcpy→ReleaseFrame；gdiCapture() 逐行；测试程序五段 + saveBmp 手写 BMP |
| **关键方法说明** | 第四章开头方法表 + 速查表 | `init` / `capture` / `captureDxgi` / `captureGdi` / `close` / `safeRelease` 的签名、语义、调用链、数据流 |
| **缺陷与局限** | 第十一章"坑点与改进" | 只抓主显示器、GDI 的 DPI 盲区、DXGI 失败不自动恢复、超时复用帧是深拷贝、每帧新分配 8.3MB 无帧池、timeout=0 忙等、GDI 每帧重建 DC、nullptr 语义模糊、无权限问题定位 |
