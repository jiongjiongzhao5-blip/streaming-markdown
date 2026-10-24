# 第 6 讲：格式转换 —— CameraFrameConverter（五种裸格式 + MJPG 统一成 BGRA）

> **对应提交**：`e92cbd3`（2026-10-18 15:30）
> **提交信息**：CameraFrameConverter把五种格式统一成 BGRA，输出 VideoFrame
> **改动规模**：4 个文件，约 1016 行（新增 `CameraFrameConverter.h` 24 行、`CameraFrameConverter.cpp` 384 行、`tests/test_camera_convert.cpp` 600 行，另改两个 `CMakeLists.txt` 共 8 行）
> **本讲时长建议**：约 2.5 小时（含动手实验）

---

## 开场白：第 5 课留下的三条尾巴，今天全部收口

同学们，上一讲我们把摄像头的数据"接"出来了。回忆一下第 5 课结尾，我们拿到的是 `CameraRawFrame`——

```cpp
struct CameraRawFrame {
    int width = 0;
    int height = 0;
    int stride = 0;                                  // 行跨度，可能带填充
    CameraPixelFormat format = CameraPixelFormat::Unknown;   // 可能是五种之一
    bool flipVertical = false;                       // 可能是"倒着"存的
    int64_t pts = 0;
    std::vector<uint8_t> bytes;                      // 原始字节，含义取决于 format
};
```

当时我在第 5 讲第 11 章"坑点与改进"里，给大家留了三条没解决的问题。今天把它们一条条念出来，因为**这三条就是今天这一讲存在的全部理由**：

| 第 5 课留下的尾巴 | 原话 | 今天怎么收口 |
|---|---|---|
| **尾巴一：格式五花八门** | "RGB24 / RGB32 / YUY2 / UYVY / MJPG 五种格式，下游编码器不可能为每一种写一套逻辑" | 统一成 **BGRA** 一种 |
| **尾巴二：MJPG 未解码** | "MJPG 摄像头拿到的 bytes 是一帧帧 JPEG 压缩数据，**进不了编码链路**" | 用 FFmpeg 的 MJPEG 解码器 + swscale 解出来 |
| **尾巴三：需要翻转** | "biHeight 为正说明自底向上存储，翻不翻是转换层的决定" | `flipVertical` 在写目标行时直接反着写 |

请注意第三条的措辞——我在第 5 讲里特意说过：**"采集层只管忠实记录事实，翻不翻是转换层的决定"**。今天就是兑现这句话的时候。`CameraCapturer` 从未翻过一帧，它只是把一个布尔量塞进帧里；真正倒着写数据的活儿，在今天这个类的第 362 行。这就是**分层**的力量：每一层只做一件事，做彻底。

再看一眼整体的数据流向，感受一下我们今天站的这个位置：

```
   摄像头硬件
       │  驱动
       ▼
  DirectShow 过滤器图        ← 第 5 讲：CameraCapturer
       │  Sample Grabber 回调
       ▼
  CameraRawFrame             ← 第 5 讲的产物：字节 + 元数据
       │
       ▼
  ┌────────────────────┐
  │ CameraFrameConverter│     ← 第 6 讲（今天）：本讲主角
  │  · YUY2/UYVY 解包   │
  │  · MJPG 解码        │
  │  · 剔除 stride 填充 │
  │  · 处理垂直翻转     │
  └────────────────────┘
       │
       ▼
  VideoFrame { width, height, pts, bgra }   ← 第 4 讲就定义好的标准帧
       │
       ▼
   编码器 / 预览 / 推流               ← 后续课程
```

请特别注意 `VideoFrame` 这个类型——它早在第 4 讲（屏幕采集）就定义好了，屏幕采集产出的就是它。**今天的本质，是把摄像头这条支路"并"进屏幕那条主干**。从此以后，下游代码再也分不出这一帧来自屏幕还是摄像头，它只知道：`bgra` 里躺着 `width × height × 4` 个字节，第 0 行就是画面顶部，B 在前 R 在后。

这就是软件工程里常说的**收敛点 / 统一表示（canonical form）**：上游可以有一千种方言，但只要我们在某一点把它们全部翻译成同一种普通话，后面的所有代码就只需要写一遍。今天这个类，就是这个翻译官。

还有一点要提前说明：今天是这个项目**第一次引入 FFmpeg**。前五讲我们一直在跟 Windows API 打交道，从今天开始，我们要进入 FFmpeg 的世界。FFmpeg 是音视频领域的事实标准开源库，后面我们要用它做编码、做封装、做推流，今天先用它的**解码器**和**图像缩放/色彩转换模块（swscale）**热热身。这段经历很重要，FFmpeg 的 API 风格——**分配器返回裸指针、用 `free` 系列函数释放、用负整数当错误码**——和现代 C++ 的 RAII 风格格格不入，我们会花不少篇幅讲怎么把它安全地裹进 C++ 的壳子里。

好，我们正式开始。

---

## 第一章：知识点预习

这一章是今天分量最重的一章。我们要一次性把"颜色是怎么用数字表示的"这件事讲透——因为本讲所有的代码，归根结底都在做同一件事：**把一种数字表示翻译成另一种数字表示**。

### 1.1 为什么需要"统一格式"这一层：扇入扇出

先看清楚不做统一会怎样。假设没有这一层，下游的编码器要面对：

```
RGB24  ┐
RGB32  │
YUY2   ├─→ 编码器
UYVY   │
MJPG   ┘
```

编码器要为 5 种输入各写一套处理逻辑，这是 5 份代码。将来我们再加屏幕采集（BGRA 已经算一种）、加图片文件输入（PNG/JPEG 又是一堆格式）、加网络流……输入格式是 **M** 种，下游环节（编码、预览、截图、加水印、推流）是 **N** 个，那就是 **M × N** 份代码——这是典型的**笛卡尔积爆炸**。

加一层转换之后：

```
RGB24  ┐
RGB32  │
YUY2   ├─→ [统一成 BGRA] ─→ 编码器 / 预览 / 截图 / 推流
UYVY   │                        （只认一种）
MJPG   ┘
```

变成 **M + N** 份。**M×N 变成 M+N**，这就是中间层唯一的、也是最强的理由。软件工程里叫**适配器模式（Adapter）**，也叫引入**中间表示（IR，Intermediate Representation）**——编译器里那个"前端 N 种语言 → IR → 后端 M 种 CPU"也是一模一样的思路。

### 1.2 为什么偏偏选 BGRA

统一成什么格式，是个要认真权衡的决策。我们的选择是 **BGRA**（蓝-绿-红-透明，每像素 4 字节）。理由有四条：

**理由一：每像素 4 字节，天然 32 位对齐。** 现代 CPU 读写内存喜欢对齐的地址，4 字节一组的像素也让 SIMD 指令（单指令多数据，一条指令同时处理多个数据）好施展。对比 RGB24 每像素 3 字节——一个像素跨两个 32 位字的概率是 2/3，处理起来别扭。

**理由二：下游全都是这个口味。** 请先记住这几个事实（后面课程会逐一验证）：
- Windows 的 GDI（图形设备接口）位图、Qt 的 `QImage::Format_ARGB32`、Direct3D 纹理，主流都是 32 位 BGRA 家族；
- x264/x265 编码器接受的输入格式里，BGRA 是标准的 `AV_PIX_FMT_BGRA`；
- 我们第 4 讲 `VideoFrame` 里的字段就已经叫 `bgra` 了——这个决定在那时就做下了。

**理由三：Alpha 通道虽然没用，但留着有用。** 摄像头不会给我们透明度，所以第 4 个字节永远是 255（完全不透明）。那为什么要留？因为**固定 4 字节的布局让"跳到下一个像素"就是加 4**，不用分支判断"这个格式是 3 还是 4"。用 1 字节的空间换掉一条分支，在每帧百万次的循环里是值得的。而且将来做画中画（把摄像头小窗叠到屏幕大画面上）时，Alpha 就是现成的混合通道。

**理由四：它是"有符号历史"的正确选择。** 见下一节。

### 1.3 BGR 还是 RGB：一段"谁先谁后"的历史账

同学们一定会问：**为什么是 BGRA 不是 RGBA**？顺序反着写，不是自找麻烦吗？

答案是历史遗留，但有理可循。早期 PC 的显示硬件和小端序（little-endian，低位字节存在低地址）CPU 结合起来，产生了一个有趣的效果：如果你把一个 32 位整数 `0xAARRGGBB` 写进内存，在小端机器上字节顺序恰好是 `BB GG RR AA`，也就是**内存里 B 在最前面**。于是很多 Windows API 就直接用"内存顺序"来命名格式，叫做 BGRA。

而网络传输、图像文件格式（PNG、BMP 文件头之后的数据）大多用"人类读写顺序"RGBA。

**结论：BGRA 和 RGBA 指的是同一块数据的两种读法，区别纯粹是"你从哪头开始数"。** 这就是为什么今天的代码里会出现这样的"陷阱"：我们说"红色"，在 BGRA 内存里是 `B=0, G=0, R=255, A=255`，也就是字节序列 `00 00 FF FF`。

今天测试程序的第一段（第 5 讲预留的"字节解读"思路的延续）就是专门验证这个的：写一个红色像素进去，看读出来是不是 `B=0,G=0,R=255`。**如果 R 和 B 写反，红色会变成蓝色**——这是颜色处理里最经典的 bug，没有之一。

### 1.4 色彩空间：YUV 到底是什么，为什么视频这么爱它

现在进入今天最核心的概念：**YUV**。

RGB 的思路很直白：一个颜色 = 红光强度 + 绿光强度 + 蓝光强度。三个分量地位平等，缺一不可。

YUV 的思路完全不同，它把颜色拆成两部分：

- **Y（Luma，亮度）**：这个像素**有多亮**。黑白电视机只需要这一个分量就能显示画面。
- **U / V（Chroma，色度）**：这个像素**偏什么颜色**。U 大致是"蓝色分量与亮度的差"（B−Y），V 大致是"红色分量与亮度的差"（R−Y）。

为什么这么拆？两个原因，一个是历史的，一个是生理的：

**历史原因**：电视机是从黑白过渡到彩色的。彩色信号必须**向后兼容**——老黑白电视收到彩色信号，只取 Y 分量就能正常显示。如果当年直接用 RGB，彩色电视普及的那十几年里所有黑白电视都收不到节目，产业不会答应。

**生理原因（更重要）**：人眼的视网膜上，感知明暗的视杆细胞数量远多于感知颜色的视锥细胞，而且视锥细胞的空间分辨率很低。**换句话说：人眼对亮度极其敏感，对颜色的细微变化相当迟钝。** 既然迟钝，那色度信息就可以"少存一点"——反正你看不出来。

这就是**色度下采样（chroma subsampling）**的全部依据，也是 YUV 能省带宽的根本原因。

### 1.5 4:4:4 / 4:2:2 / 4:2:0：那三个数字在说什么

YUV 格式命名里那串数字（4:2:2、4:2:0、4:4:4），念法是"**每 4 个亮度像素对应多少个色度像素**"。严谨的记法是 `J:a:b`，意思是：在**水平方向 4 个像素**的参考块里——

- **J**（通常写 4）：亮度采样的基准宽度；
- **a**：**第一行**有多少个色度采样；
- **b**：**第二行**有多少个色度采样（相对第一行有多少变化）。

我们用图说话（● = 亮度 Y，○ = 色度 UV 对，每行代表一行像素）：

```
4:4:4（无下采样，全分辨率色度）
行1: ●○ ●○ ●○ ●○
行2: ●○ ●○ ●○ ●○
     每个像素都有自己的一套色度。数据量 = RGB 的 100%（但通道不再是 RGB）

4:2:2（水平方向色度减半，垂直不变）      ← YUY2 / UYVY 属于这一类
行1: ●○ ●  ●○ ●
行2: ●○ ●  ●○ ●
     每两个水平相邻像素共享一套色度。数据量 = RGB24 的 2/3

4:2:0（水平减半 + 垂直减半）
行1: ●○ ●  ●○ ●
行2: ●  ●  ●  ●
行3: ●○ ●  ●○ ●
行4: ●  ●  ●  ●
     每 2×2 的四个像素共享一套色度。数据量 = RGB24 的 1/2
```

**今天我们要处理的 YUY2 和 UYVY，都是 4:2:2**。这意味着：**水平方向上，第 0 个和第 1 个像素共用同一套 U/V，第 2 个和第 3 个像素共用另一套**。理解这一点，今天一半的代码就通了。

### 1.6 打包格式（packed）与平面格式（planar）

同样是一帧 YUV，在内存里怎么摆，又有两大流派：

**平面格式（planar）**：三个分量分开存，各占一块连续内存。比如 YUV420P 就是：

```
[Y 平面: width*height 字节][U 平面: width/2 * height/2 字节][V 平面: ...]
```

好处是处理某个分量时内存连续、缓存友好；坏处是要拿三个指针。

**打包格式（packed / interleaved）**：不同分量交错存在同一块内存里，按"宏像素"为单位重复。YUY2 就是打包格式。

**宏像素（macro pixel）**：打包格式里最小的、不可再分的重复单元。YUY2 的宏像素是 **4 个字节，代表 2 个像素**。这个概念非常关键：**你不能从 YUY2 里单独取出"第 3 个像素"的字节，你必须先定位它属于哪个宏像素，再取出整个宏像素一起解。**

顺便说一句，FFmpeg 的像素格式命名里，**带 P 后缀的（如 `AV_PIX_FMT_YUV420P`）就是 planar，不带的就是 packed**。今天解码 MJPG 出来的多半是 `YUVJ420P`（planar），我们要交给 swscale 去处理；而 YUY2 是我们自己手写的。

### 1.7 YUY2 与 UYVY 的字节布局：今天要手写的两个格式

这两个格式都是 4:2:2 打包，**每 2 个像素占 4 字节，平均 2 字节/像素**。区别只是这 4 个字节内部的排列顺序。

**YUY2**（也常写成 YUYV）的宏像素布局：

```
字节下标:   0     1     2     3
含义:      Y0    U     Y1    V
对应像素:  第0个  ←共用→  第1个
```

也就是说，一行的字节流是：

```
Y0 U Y1 V | Y2 U Y3 V | Y4 U Y5 V | ...
└─像素0,1─┘ └──像素2,3─┘ └──像素4,5─┘
```

**UYVY** 的宏像素布局：

```
字节下标:   0     1     2     3
含义:      U     Y0    V     Y1
对应像素:  第0个  ←共用→  第1个
```

一行的字节流：

```
U Y0 V Y1 | U Y2 V Y3 | U Y4 V Y5 | ...
└─像素0,1─┘ └──像素2,3─┘ └──像素4,5─┘
```

看出来了吗？**UYVY 就是把 YUY2 的每 4 字节做了一次"左右两两交换"**（准确的说是把 Y 和色度交错的位置对调）。这个"只差一个字节顺序"的特性，让 UYVY 成为绝佳的**排障探针**：如果你把 UYVY 当成 YUY2 读，会发生什么？

- `src[0]`（原本是 U）被当成 Y0 → 亮度取到了色度值；
- `src[1]`（原本是 Y0）被当成 U → 色度取到了亮度值；
- `src[2]`（原本是 V）被当成 Y1；
- `src[3]`（原本是 Y1）被当成 V。

结果就是：**U 和 V 互换了**（因为 U←Y0, V←Y1, 而原本应该 U=U, V=V）。U 和 V 互换在视觉上是什么效果？**红色变蓝色、蓝色变红色，绿色基本不变**——因为 R 主要由 V 决定，B 主要由 U 决定，G 是两个都减。所以测试程序第 6 段有一句提示：

```cpp
std::printf("  若这里的 R 和 B 对调，说明把 UYVY 当成 YUY2 读了\n");
```

这就是"用可预期的错误现象反向定位 bug"的排障艺术。请大家记住这个现象：**画面里人脸发蓝、天空发橙 = U/V 弄反了**。

还有一条算术要算清楚：**一行的字节数**。宽度 `w` 的 4:2:2，宏像素个数是 `w/2` 个（向上取整，因为奇数宽度时最后一个宏像素也要占满 4 字节，后半是填充）。所以：

```cpp
rowBytes = ((width + 1) / 2) * 4;
```

`(width + 1) / 2` 是整数除法下的**向上取整**写法。验证：
- `width = 2` → `(3)/2 = 1` → 4 字节 ✓（1 个完整宏像素）
- `width = 3` → `(4)/2 = 2` → 8 字节 ✓（2 个宏像素，第 2 个的后两字节代表一个不存在的像素）
- `width = 1920` → `1921/2 = 960` → 3840 字节 = 1920 × 2 ✓

### 1.8 YUV → RGB 的公式：BT.601、BT.709 与整数近似

现在到了真正"算颜色"的环节。YUV 转 RGB 不是什么神秘的黑箱，它就是**一个 3×3 矩阵乘一个向量**。不同的"标准"给出不同的矩阵系数。

**BT.601**（ITU-R Recommendation BT.601）：为**标清电视（SDTV）**制定的标准，适用于 480i/576i 这一代。摄像头、USB 采集卡的世界里，它仍然是默认。

它的浮点公式（有限范围输入，见 1.9）是：

```
R = 1.164 * (Y-16) + 1.596 * (V-128)
G = 1.164 * (Y-16) - 0.392 * (U-128) - 0.813 * (V-128)
B = 1.164 * (Y-16) + 2.017 * (U-128)
```

**BT.709**：为**高清电视（HDTV）**制定，720p/1080i 及以上用。系数不同：

```
R = 1.164 * (Y-16) + 1.793 * (V-128)
G = 1.164 * (Y-16) - 0.213 * (U-128) - 0.533 * (V-128)
B = 1.164 * (Y-16) + 2.112 * (U-128)
```

看出来了吗？**BT.709 里 V 对 R 的贡献更大（1.793 vs 1.596）**。选错标准的后果是颜色"偏"——饱和度不对，红色要么发暗要么过艳，但通常不会离谱到认不出来。这也是为什么"选错色彩标准"是个难以察觉的问题：它不像 U/V 反了那样一眼看出，它是"好像有点怪，但说不上哪里怪"。

**为什么代码里是整数而不是浮点？** 因为一帧 1920×1080 有 200 万个像素，每个像素 3 次浮点乘法 = 600 万次。在实时系统里这很贵。而整数运算（尤其是位移）快得多。经典做法是：把系数乘以 256（即 2⁸），用整数乘法，最后右移 8 位当除法。

```
1.164 * 256 = 298.0   →  298
1.596 * 256 = 408.6   →  409
0.392 * 256 = 100.4   →  100
0.813 * 256 = 208.1   →  208
2.017 * 256 = 516.4   →  516
```

再加 `+128` 是为了**四舍五入**（加半个除数再整除 = 四舍五入）。

于是就有了今天代码里的这三行（BT.601 整数版）：

```cpp
const int r = (298 * c + 409 * e + 128) >> 8;
const int g = (298 * c - 100 * d - 208 * e + 128) >> 8;
const int b = (298 * c + 516 * d + 128) >> 8;
```

这是一套在业界流传了二十多年的经典整数近似，很多库（包括 FFmpeg 内部的某些路径、libyuv、无数嵌入式代码）都在用。它的精度损失在 1~2 个色阶以内，**人眼完全看不出来**。

### 1.9 有限范围（16–235）与全范围（0–255）：一个反复踩的坑

这一节非常重要，它解释了今天测试里两个看起来很怪的数字。

**有限范围（limited range / TV range / video range）**：Y 只用 **16~235**，U/V 只用 **16~240**（名义上），0 和 255 被保留做同步信号。这是广播电视的传统。摄像头、摄像机、绝大多数视频文件用这个范围。

**全范围（full range / PC range / JPEG range）**：Y、U、V 都用满 **0~255**。这是计算机图形和 **JPEG** 的传统。

**这个区别意味着什么？** 意味着"同样的 Y=128"在两种范围下代表不同的亮度：

- 有限范围：Y=128 落在 16~235 的中间偏上一点点，对应 RGB 约 **130**（不是 128！）；
- 全范围：Y=128 就是中灰，对应 RGB **128**。

所以今天测试第 5 段里有这么一段注释和断言，请同学们仔细读：

```cpp
setYuy2Pair(gray, 0, 0, 128, 128, 128, 128);
...
check(pixelIs(*grayOut, 0, 0, 130, 130, 130, 3),
      "Y=U=V=128 得到约 130 的灰（说明做了 16~235 -> 0~255 的拉伸）");
std::printf("  实际值: %s（偏移处理后应约为 130，不是 128 也不是 149）\n", ...);
```

我们手算一遍：Y=128, U=128, V=128 → `c = 128-16 = 112`，`d = 0`，`e = 0`。

```
r = (298 * 112 + 0 + 128) >> 8 = (33376 + 128) >> 8 = 33504 >> 8 = 130.875 → 130
```

（整数右移直接截断，33504 / 256 = 130.875，取整 130）

**得到 130，不是 128**。这就是"先减 16 再乘 1.164"这个**拉伸**操作带来的效果：把 16~235 的 219 级拉伸到 0~255 的 256 级，比例 256/219 ≈ 1.1689。所以 Y=128 被拉伸成了 `(128-16) × 1.164 ≈ 130`。

如果某个工程师忘了这个偏移，直接用 `R = Y + 1.402*(V-128)`，那 Y=128 会得到精确的 128 灰——**整个画面会偏暗一点点，暗部会"发灰发雾"**，对比度不足。这种 bug 极难发现，因为画面看起来"能用"。

再看另一个例子（测试第 5 段）：

```cpp
setYuy2Pair(raw, 0, 0, 81, 81, 90, 240);   // Y=81 U=90 V=240
check(pixelIs(*out, 0, 0, 0, 0, 255, 2), "像素(0,0) 接近纯红");
```

Y=81, U=90, V=240 → `c=65, d=-38, e=112`：

```
r = (298*65 + 409*112 + 128) >> 8 = (19370 + 45808 + 128) >> 8 = 65306 >> 8 = 255.1 → 255 ✓
g = (298*65 - 100*(-38) - 208*112 + 128) >> 8 = (19370 + 3800 - 23296 + 128) >> 8 = 2 >> 8 = 0 ✓
b = (298*65 + 516*(-38) + 128) >> 8 = (19370 - 19608 + 128) >> 8 = (-110) >> 8 = -1 → clamp 后 0 ✓
```

**Y=81 U=90 V=240 就是 BT.601 有限范围下的"纯红"**，转换出来恰好是 `(B=0, G=0, R=255)`。同理，测试里用的 `Y=41 U=240 V=110` 是纯蓝。这两个数字组合请大家记牢，它们是我们后面所有 YUV 测试的基准值。

顺带一提，第 5 讲的测试里我们造过一个"纯绿帧"来排查。绿色对应的 YUV 是 `Y=145 U=54 V=34` 左右（有兴趣的同学可以自己推）。

### 1.10 JPEG 与 MJPG：每帧一张独立照片

**JPEG** 大家都很熟悉，一种有损压缩的图片格式。它压缩的是**一张静止图片**，核心步骤是：颜色转 YUV → 色度下采样（通常 4:2:0）→ 分 8×8 块做 DCT（离散余弦变换，把空间信息变成频率信息）→ 量化（丢掉高频细节，这是"有损"的来源）→ 熵编码（Huffman，无损压缩）。

**MJPG / MJPEG（Motion JPEG）** 朴素得可爱：**一串 JPEG 图片排起来就是视频**。每一帧都是一张完全独立、可单独解码的 JPEG。

对比一下真正的视频压缩（H.264 等）：

| | Motion JPEG | H.264 / HEVC |
|---|---|---|
| 帧间关系 | **每帧独立**，互不参考 | 有 I/P/B 帧，P 帧参考前面的帧 |
| 压缩率 | 低（约 1/10） | 高（约 1/50 ~ 1/100） |
| 丢一帧的后果 | 只丢一帧，后面照常 | 后面一串帧全废 |
| 编辑/seek | 随便跳，随便剪 | 要找到关键帧才能开始解 |
| 解码开销 | 小 | 大 |
| 适合场景 | **采集**、监控、编辑中间格式 | 存储、传输 |

**摄像头为什么爱 MJPG？** 因为 USB 带宽不够！1920×1080 的 YUY2 是 `1920×1080×2 = 4.15 MB` 一帧，30fps 就是 **124 MB/s**——USB 2.0 的实际带宽约 40 MB/s，根本塞不下。而 MJPG 压到 1/10 就是 12 MB/s，USB 2.0 轻松跑。所以**所有高清 USB 摄像头都支持 MJPG**，这是刚需，不是可选。

代价就是：**下游必须解码**。这就是今天 `convertMjpeg` 存在的理由。

### 1.11 JPEG 的标记（marker）：SOI 与 EOI

JPEG 文件的结构是一串"标记段"。每个标记以 `0xFF` 开头，后跟一个类型字节。我们只需要记住两个：

- **SOI（Start Of Image，图像开始）**：`FF D8`。每个 JPEG 文件的头两个字节必然是它。
- **EOI（End Of Image，图像结束）**：`FF D9`。JPEG 正式结束的地方。

这两个标记在今天的**容错逻辑**里是关键角色。

为什么要容错？因为在 DirectShow 的 Sample Grabber 里，我们拿到的是一个**定长缓冲区**（驱动按最大帧长分配），而实际这一帧 JPEG 只用了其中一部分。缓冲区尾部常常残留**上一帧的旧数据**或者**一堆零**。文档里第 5 讲也提到过"bytes 可能含尾部零填充"。

于是就有了今天这段代码（第 226~236 行的 EOI 兜底）：解不出来时，**从后往前扫，找到最后一个 `FF D9`，把数据截断到那里，再解一次**。测试第 8 段专门验证这一点：

```cpp
CameraRawFrame padded = raw;
padded.bytes.resize(jpeg.size() + 4096, 0);
check(converter.convert(padded) != nullptr,
      "帧尾补了 4096 个零字节，仍能靠 EOI 兜底解出来");
```

顺带说一句：`FF D9` 这个两字节序列，理论上也可能在 JPEG 的压缩数据（熵编码段）里偶然出现。但 JPEG 规范规定熵编码数据里的 `FF` 后面必须跟 `0x00`（叫"字节填充 / byte stuffing"），所以真正的图像数据里**不会出现裸的 `FF D9`**。这个兜底是安全的。

### 1.12 什么时候手写转换，什么时候用 swscale？

这是今天一个很值得讨论的**工程决策**。我们看一下本讲实际做了什么选择：

| 输入格式 | 用什么做转换 | 为什么 |
|---|---|---|
| RGB24 → BGRA | **手写**（`convertRgb24Row`） | 只是复制 3 字节 + 补一个 255。写 4 行比调库快得多，还没有库调用开销 |
| RGB32 → BGRA | **手写**（`convertRgb32Row`） | 同上，几乎就是 memcpy + 强制 alpha |
| YUY2 → BGRA | **手写**（`convertYuy2Row`） | 解包 + BT.601 整数公式，可控、可预测、零依赖 |
| UYVY → BGRA | **手写**（`convertUyvyRow`） | 同上，只差字节顺序 |
| MJPG → BGRA | **FFmpeg**（解码器 + `swscale`） | 解码 JPEG 我们自己写不了（几千行），必须借库；解码出来是 planar YUV，用 swscale 转最省事 |

这个选择背后的原则，请大家记住：

1. **能自己写清楚的就自己写**。自己写的转换零依赖、行为完全可预测、调试时能看到每一行在干什么、不会有版本差异。
2. **自己写不了/写不划算的，果断借库**。JPEG 解码涉及 Huffman 表、DCT 反变换、量化表、色彩空间**三选一**（4:2:0/4:2:2/4:4:4）——这是几千行代码和几十年的细节积累，绝不能手写。
3. **决策的边界是"复杂度"，不是"性能"**。很多人以为手写是为了快，其实不是——`swscale` 有 SIMD 优化，大概率比我们手写的快。手写是为了**可控和可调试**。

### 1.13 swscale 与 SwsContext：FFmpeg 的"图像变换器"

**swscale**（读音 "s-w-scale"）是 FFmpeg 里负责**图像缩放和像素格式转换**的模块。它的核心对象叫 `SwsContext`（sws 上下文）。

它的典型用法是"三步走"：

```cpp
// 1. 创建上下文：告诉它"从什么变到什么，多大，用什么算法"
SwsContext* sws = sws_getContext(srcW, srcH, srcFormat,
                                 dstW, dstH, dstFormat,
                                 SWS_BILINEAR, nullptr, nullptr, nullptr);

// 2. 干活：把 src 的像素写进 dst
sws_scale(sws, srcData, srcLinesize, 0, srcH, dstData, dstLinesize);

// 3. 用完释放
sws_freeContext(sws);
```

几个概念要解释：

- **`SWS_BILINEAR`（双线性插值）**：缩放时新像素怎么算出来的算法之一。双线性是"取周围 4 个像素按距离加权平均"，质量和速度平衡得最好。其它选项有 `SWS_POINT`（最近邻，最快最糙）、`SWS_BICUBIC`（双三次，最慢最好）、`SWS_FAST_BILINEAR` 等。**注意：如果输入输出尺寸相同（我们今天就是），缩放算法其实用不上，只有色彩空间转换生效**——但参数还是得给。
- **`linesize`（也叫 stride）**：每一行的字节数。FFmpeg 里的帧，每行末尾可能有对齐填充，所以必须显式告诉它行跨度。这和我们在第 5 讲讲的 DIB 行对齐是同一个概念。**FFmpeg 里这个字段叫 `linesize`，Windows 里叫 `stride`，DirectShow 里叫 `biWidth` 对齐——三个名字，一件事。**
- **`data[4]` / `linesize[4]`**：为什么是长度 4 的数组？因为 planar 格式最多有 4 个平面（Y、U、V、Alpha）。packed 格式只用 `data[0]`。

**上下文为什么要缓存？** `sws_getContext` 内部要计算一堆查找表（色彩转换矩阵、缩放权重表），**相当贵**。如果每帧都创建释放，性能会崩。所以今天的做法是把 `SwsContext*` 存在 `Impl` 里，只在**尺寸或格式变化**时才重建（第 251~252 行的判断）。这是 FFmpeg 编程的标准套路，也是新手最容易忽略的性能陷阱。

### 1.14 stride 与 flipVertical：第 5 讲的两个遗留概念复习

**stride（行跨度，也叫 pitch）**：一行像素在内存里实际占用的字节数。它可能**大于**"宽度 × 每像素字节数"，多出来的叫**填充（padding）**。原因是各种历史对齐要求（Windows DIB 要求 4 字节对齐，硬件解码器可能要求 16 或 64 字节对齐）。

**为什么填充很危险？** 因为如果你按"宽度×3"跳行，而实际行跨度是"宽度×3 + 填充"，那么第 1 行你就少跳了几个字节，第 2 行少跳更多……**误差逐行累积，画面就变成一条条斜纹**。这个现象有个专门的名字叫 **斜纹错位（shearing / skew artifact）**，看到它就该立刻想到 stride。

**flipVertical（垂直翻转）**：源数据可能是"自底向上"存的（第 0 行是画面底部）。转换层要负责把它正过来。今天的做法极其优雅——**不先转换再翻转（那要两趟），而是在写的时候直接写到"镜像的目标行"**：

```cpp
const int dstY = raw.flipVertical ? (raw.height - 1 - y) : y;
```

源第 0 行写到目标第 `height-1` 行，源第 1 行写到目标第 `height-2` 行……一趟搞定，零额外开销。这叫**在写入时完成变换（transform on write）**，是数据搬运类代码里最该掌握的技巧之一。

### 1.15 FFmpeg 的 API 风格：裸指针、负错误码、_free 系列

第一次用 FFmpeg 的同学会被它的风格"劝退"。我们来打个预防针，把它的三条不成文规矩讲清楚：

**规矩一：分配函数返回裸指针，配对的 `xxx_free` 接受指针的地址（双指针）。**

```cpp
AVCodecContext* ctx = avcodec_alloc_context3(codec);
...
avcodec_free_context(&ctx);      // 注意是 &ctx
```

为什么是双指针？因为 `free` 函数要在内部**把你的指针置空**，防止你后面用到悬空指针（dangling pointer，指向已释放内存的指针）。所以调用完 `avcodec_free_context(&ctx)` 之后，`ctx` 自动变成 `nullptr`。这是 C 语言里模拟"安全释放"的经典手法。

**规矩二：错误码是负整数。** FFmpeg 的函数成功时返回 0 或正数，失败时返回一个负数（其实就是 POSIX 的 `errno` 负值，比如 `-EINVAL` = -22 = 无效参数，`-EAGAIN` = -11 = 再试一次）。判断是否失败统一用 `ret < 0`，不要 `ret != 0`（因为成功也可能返回正数）。

**规矩三：错误码要用 `av_strerror` 翻译成人话。**

```cpp
char buffer[AV_ERROR_MAX_STRING_SIZE] = {};
av_strerror(err, buffer, sizeof(buffer));
```

`AV_ERROR_MAX_STRING_SIZE` 是 FFmpeg 定义的常量，保证足够装下任何错误描述。这个函数不会失败。

**规矩四（额外）：新解码 API 是"发送/接收"双函数模型。** 老 API 是 `avcodec_decode_video2(ctx, frame, &gotFrame, packet)`，一次调用塞一个包吐一个帧。新 API（FFmpeg 3.1 之后）拆成两个：

```cpp
avcodec_send_packet(ctx, packet);      // 塞一个压缩包进去
avcodec_receive_frame(ctx, frame);     // 尝试取出一个解好的帧
```

为什么要拆？因为现代编解码器有**内部延迟**（B 帧需要缓存），塞进去一个包不一定立刻吐出一个帧；反过来，一个包也可能吐出多个帧（某些格式）。分开之后就能用循环把 decoder 里的东西"抽干"。今天的代码只调用一次 receive（因为 MJPEG 没有帧间延迟，一对一），但同学们要知道完整写法应该是个 `while (receive_frame(...) >= 0)` 的循环。

### 1.16 PIMPL：把 FFmpeg 的头文件关在 .cpp 里

最后一条预习，也是解释"为什么这个类的头文件这么干净"。

**PIMPL（Pointer to Implementation，指向实现的指针）**，也叫"编译期防火墙（compilation firewall）"。写法是：头文件里只声明一个不完整类型 `struct Impl;` 和一个 `std::unique_ptr<Impl>`，真正的定义放在 .cpp 里。

```cpp
// .h
private:
    struct Impl;                        // 只有声明，没有定义
    std::unique_ptr<Impl> impl_;        // 指针大小是固定的，编译器不需要知道 Impl 长啥样
```

为什么今天特别需要它？**因为 FFmpeg 的头文件不能外泄。** `libavcodec/avcodec.h` 之类的头文件：

- 定义了成百上千个宏和类型，会**污染**所有 include 了我们的头文件的翻译单元；
- 必须用 `extern "C" { }` 包起来（FFmpeg 是 C 库，C++ 编译会因为名字改编 name mangling 而链接失败）；
- 版本间有差异，暴露出去就是把 FFmpeg 版本绑死在我们的公共接口上。

用了 PIMPL，`CameraFrameConverter.h` **只需要 `<memory>` 和 `<string>`**，任何 include 它的文件都完全不知道 FFmpeg 的存在。这是 C++ 项目里隔离第三方依赖最标准的手段。代价是每次访问成员多一次指针间接跳转（可忽略），以及 `Impl` 的构造/析构必须在 .cpp 里（因为 `unique_ptr` 的析构需要完整类型）。

顺带一个必考点：**析构函数必须在 .cpp 里定义**。

```cpp
CameraFrameConverter::~CameraFrameConverter() = default;   // 写在 .cpp 里，不能写在 .h
```

如果写在 .h 里，编译器在那个位置要生成 `unique_ptr<Impl>` 的析构代码，但 `Impl` 还不完整——它不知道 `Impl` 有多大、有没有非平凡析构，**直接编译报错**。这是 PIMPL 最经典的编译错误，今天代码第 299 行就是为此存在的。

预习完毕。下面正式读代码。

---

## 第二章：构建系统的两处改动 —— 让 FFmpeg 进来

老规矩，先看看为了让这个新模块能编译，构建脚本要改什么。虽然只有 8 行，但每一处都有讲究。

### 2.1 `src/core/capture/CMakeLists.txt`

```cmake
add_library(core_capture STATIC
  ScreenCapturer.cpp
  ScreenCapturer.h
  CameraCapturer.cpp
  CameraCapturer.h
  CameraFrameConverter.cpp      # ← 新增
  CameraFrameConverter.h        # ← 新增
)

target_include_directories(core_capture
  PUBLIC ${CMAKE_CURRENT_SOURCE_DIR}/../..     # ← 原来是 PUBLIC 独占一行
  PRIVATE ${FFMPEG_INCLUDE_DIRS}               # ← 新增
)

target_link_libraries(core_capture PUBLIC
  core_common
  ${FFMPEG_LIBRARIES}                          # ← 新增
)
```

**第一处：源文件清单。** 把新的 `.h` 也列进去是 CMake 的惯例——`.h` 不参与编译，但列进去能让 IDE（Visual Studio、CLion）在解决方案资源管理器里看到它们，方便跳转。CMake 看到 `.h` 会自动标记为头文件，不做处理。

**第二处：这是最关键的一处，也是一个"关键字位置"的精妙差异。**

```cmake
target_include_directories(core_capture
  PUBLIC ${CMAKE_CURRENT_SOURCE_DIR}/../..
  PRIVATE ${FFMPEG_INCLUDE_DIRS}
)
```

注意原来是两个目标共用一个 `PUBLIC` 关键字，现在拆成了两个关键字。这两个关键字的含义天差地别：

| 关键字 | 含义 | 谁能看到 |
|---|---|---|
| **PUBLIC** | "我自己要用，**任何链接我的人也要用**" | 会**传递**给下游 |
| **PRIVATE** | "只有我自己要用，别告诉别人" | **不传递** |
| **INTERFACE** | "我自己不用，但链接我的人要用" | 只传给下游 |

`FFMPEG_INCLUDE_DIRS` 用 `PRIVATE`，是因为**只有 `CameraFrameConverter.cpp` 需要 `#include <libavcodec/avcodec.h>`**。任何链接 `core_capture` 的模块（比如我们后面要写的编码器模块）都不该自动获得 FFmpeg 的头文件搜索路径——它们要用 FFmpeg，应该自己去声明依赖。这就是**依赖最小化原则**：不要把实现细节通过构建系统泄漏出去。这跟 1.16 讲的 PIMPL 是同一个思想，只是一个在编译层面，一个在构建层面。

**第三处：`${FFMPEG_LIBRARIES}` 用 `PUBLIC` 链接。** 这里是 `target_link_libraries(core_capture PUBLIC ...)`，FFmpeg 的库会被**传递**给链接 `core_capture` 的下游。这是必需的：静态库（STATIC）里的符号引用，在最终链接可执行文件时必须能找到定义，所以链接性必须传递。这也意味着下游即使不直接调 FFmpeg，也会被链上 FFmpeg 的库——这是静态库的固有特性，可以接受。

### 2.2 `tests/CMakeLists.txt`

```cmake
add_executable(test_camera_convert
  test_camera_convert.cpp
)

target_link_libraries(test_camera_convert PRIVATE
  core_capture
)
```

干净利落：一个可执行文件，链 `core_capture`。注意它**只链了 `core_capture`**，没有直接链 FFmpeg——但因为它链的是 `core_capture`，而 `core_capture` 用 `PUBLIC` 链了 FFmpeg，所以 FFmpeg 的库会被自动传递过来；同时因为 `FFMPEG_INCLUDE_DIRS` 是 `PRIVATE`，测试程序**看不到** FFmpeg 的头文件——

等等，可是 `test_camera_convert.cpp` 明明 `#include <libavcodec/avcodec.h>` 了！它能编译吗？

**能。** 因为顶层 `CMakeLists.txt` 里大概率有一个全局的 `include_directories(${FFMPEG_INCLUDE_DIRS})`（或者测试目录自己有设置），这让所有目标都能找到 FFmpeg 头文件。这是一个小小的**不一致**：转换器的 FFmpeg 头文件可见性是 `PRIVATE`（严格），测试程序却靠全局设置（宽松）。作为测试程序可以接受，但也提示我们：**项目的 FFmpeg 依赖管理还不够精细**，理想的做法是定义一个 `core_ffmpeg` 的接口库（INTERFACE library）统一封装。这一点我们放到第九章"坑点与改进"里详细说。

---

## 第三章：对外契约 —— `CameraFrameConverter.h`

整个头文件只有 24 行，我们逐行看。

```cpp
#pragma once

#include "core/capture/CameraCapturer.h"
#include "core/common/Common.h"

#include <memory>
#include <string>

class CameraFrameConverter {
public:
    CameraFrameConverter();
    ~CameraFrameConverter();

    CameraFrameConverter(const CameraFrameConverter&) = delete;
    CameraFrameConverter& operator=(const CameraFrameConverter&) = delete;

    VideoFramePtr convert(const CameraRawFrame& raw);

    const std::string& lastError() const;

private:
    struct Impl;
    std::unique_ptr<Impl> impl_;
};
```

### 3.1 两个 include 各自带来什么

```cpp
#include "core/capture/CameraCapturer.h"
```

带来两样东西：`CameraPixelFormat` 枚举和 `CameraRawFrame` 结构体——也就是**输入**。

```cpp
#include "core/common/Common.h"
```

带来 `VideoFrame` 和 `VideoFramePtr`——也就是**输出**。

请同学们把这两个 include 连起来看，整个类的契约就跃然纸上了：

```
CameraCapturer.h（输入方言）  →  CameraFrameConverter  →  Common.h（输出普通话）
```

**而且注意：这里没有 `#include` 任何 FFmpeg 头文件。** 这就是 1.16 讲的 PIMPL 的直接成果。对比一下如果没有 PIMPL，这个头文件会变成什么样子——得写 `extern "C" { #include <libavcodec/avcodec.h> ... }`，然后所有 include 它的文件都被 FFmpeg 的宏污染。**24 行 vs 可能上千行的宏污染**，这就是 PIMPL 的价值。

### 3.2 删除拷贝：一个持有 FFmpeg 资源的类不该被复制

```cpp
CameraFrameConverter(const CameraFrameConverter&) = delete;
CameraFrameConverter& operator=(const CameraFrameConverter&) = delete;
```

为什么要删？因为这个对象内部持有**四个裸指针形式的 FFmpeg 资源**：`AVCodecContext*`、`AVFrame*`、`AVPacket*`、`SwsContext*`。

如果我们允许拷贝，默认的拷贝构造函数会**逐个成员浅拷贝指针**。结果就是两个对象指向同一个 `AVCodecContext`。当第一个对象析构时，它 `avcodec_free_context` 把上下文释放了；第二个对象还以为自己持有有效指针——**悬空指针**，下次调用就是未定义行为（通常是崩溃）。更糟的是第二个对象析构时会**二次释放（double free）**。

这是 C++ 资源管理最经典的陷阱。**规则很简单：一个类如果持有独占资源，就删掉拷贝，只留移动（或者像这里，干脆什么都不留）。**

另外，`std::unique_ptr<Impl> impl_` 这个成员本身就**不可拷贝**（`unique_ptr` 的拷贝构造被删除了），所以即使我们不写这两行，编译器也会隐式删除拷贝构造。那为什么还要显式写？

**为了表达意图（intent）。** 显式 `= delete` 是在告诉读代码的人："这个类的设计者**考虑过**拷贝这件事，并且明确禁止它"，而不是"忘了写"。而且它让错误信息更友好——写了 `= delete`，误用时编译器报"调用了已删除的函数"，非常明确；不写的话，报错会指向 `unique_ptr` 内部，让人一头雾水。这叫**自文档化代码（self-documenting code）**。

### 3.3 `convert` 的签名：为什么返回 `shared_ptr` 而不是值

```cpp
VideoFramePtr convert(const CameraRawFrame& raw);
```

拆解这个签名，四个决策：

**决策一：返回值用 `VideoFramePtr`（即 `std::shared_ptr<VideoFrame>`）。**

为什么不直接返回 `VideoFrame` 值？因为一帧 1080p 的 BGRA 是 `1920×1080×4 = 8.3 MB`。返回值的成本——在现代 C++ 里有 RVO/NRVO（返回值优化）和移动语义，其实**已经很低了**。但用 `shared_ptr` 有另一个好处：**可以零成本地在多个消费者之间共享同一帧**（比如同时送给编码器和预览窗口，谁都不用拷贝）。在实时视频管线里，"一帧多处消费"是常态，共享指针是对的选择。

**决策二：失败时返回 `nullptr`。** 不用异常（项目里 `CameraCapturer` 也没用异常），错误信息通过 `lastError()` 取。这是一个**内部一致的错误处理风格**：整个项目都用"返回空/假 + 查询错误串"，我们延续它。**一致性比"哪种方案更好"更重要**。

**决策三：参数是 `const CameraRawFrame&`。** 常量引用——不拷贝入参（帧数据可能几 MB），也不修改它。转换器是**只读取**输入的。

**决策四：`convert` 不是 `const` 成员函数。** 注意这里没有结尾的 `const`。为什么？因为转换过程会**修改内部状态**：MJPEG 解码器上下文、`SwsContext` 缓存、以及最重要的 `lastError`。所以它语义上不是"只读查询"。这是正确的设计——`const` 是承诺"我不会改自己"，我们不承诺。

### 3.4 `lastError()`：返回常量引用

```cpp
const std::string& lastError() const;
```

两个 `const` 各管一段：前面的 `const std::string&` 是说"**不给你拷贝，也不许你改**"；后面的 `const` 是说"**调用这个函数不会修改对象**"。

返回引用而不返回值，是为了避免每次查询都拷贝一次字符串。代价是调用方不能持有这个引用太久——下一次 `convert()` 会 `clear()` 它。这是 C++ 里常见的"借用（borrow）"语义，使用时要注意生命周期：**如果你需要长期保存错误信息，自己拷贝一份**。

### 3.5 `struct Impl;` 与 `unique_ptr`

```cpp
private:
    struct Impl;
    std::unique_ptr<Impl> impl_;
```

这两行就是 1.16 讲的全部。`struct Impl;` 是**前向声明（forward declaration）**，告诉编译器"有这么个类型，具体长啥样你以后会知道"。因为 `unique_ptr` 的大小只取决于指针大小（不取决于 `Impl` 的大小），编译器有足够信息算出 `CameraFrameConverter` 的布局。

**这也意味着：`CameraFrameConverter` 的大小是固定的 8 字节（64 位机器上一个指针），无论 `Impl` 里塞了多少东西。** 这就是为什么 PIMPL 也叫"编译期防火墙"——改变 `Impl` 的内部实现，**不会改变 `CameraFrameConverter` 的内存布局，于是所有依赖它的代码都不需要重新编译**（如果用的是动态库，甚至不用重新链接）。对大型项目来说，这个特性价值连城。

---

## 第四章：`CameraFrameConverter.cpp` 上半场 —— 匿名命名空间里的八件工具

现在进入主菜。我们先看文件开头到匿名命名空间结束的部分（第 1~123 行）。

### 4.1 头文件区：为什么 FFmpeg 要 `extern "C"`

```cpp
#include "core/capture/CameraFrameConverter.h"

extern "C" {
#include <libavcodec/avcodec.h>
#include <libavutil/error.h>
#include <libswscale/swscale.h>
}

#include <cstdint>
#include <cstring>
```

**第 1 行**：包含自己的头文件。这是每个 .cpp 都该做的第一件事——它能立刻暴露"头文件本身能不能独立编译"的问题（比如忘了 include 依赖）。

**第 3~7 行：`extern "C"` 块。** 这是 C++ 调用 C 库的标准写法。原理是：**C++ 编译器会对函数名做"名字改编（name mangling）"**，把 `avcodec_open2(AVCodecContext*, const AVCodec*, AVDictionary**)` 改编成类似 `_Z14avcodec_open2P14AVCodecContextPK9AVCodecPP12AVDictionary` 这样的符号名（因为 C++ 支持重载，参数类型必须编进符号名里）。但 FFmpeg 的库是用 C 编译器编译的，它导出的符号就是朴素的 `avcodec_open2`。**两边符号名对不上，链接器报"未定义的引用"。**

`extern "C"` 的作用就是告诉 C++ 编译器：**"这些声明按 C 的规则来，不要改编名字"**。于是符号名对上，链接成功。

**为什么用块的形式而不是逐个声明？** 因为这三个头文件互相依赖，而且 FFmpeg 的头文件**自己内部也会 include 其它 FFmpeg 头文件**。用 `extern "C" { ... }` 把整个 include 块包起来，一次性解决所有。这是 FFmpeg C++ 项目的标准写法，请大家记住。

> **补充知识**：其实 FFmpeg 的头文件里**已经自己写了** `#ifdef __cplusplus extern "C" { #endif` 的保护。那为什么我们还要包一层？因为历史上有些版本/某些头文件保护不完整，加上去是**防御性编程**，零成本。而且有些第三方分发版（比如某些 vcpkg 包）会改头文件。加了没坏处。

**第 9~10 行**：`<cstdint>` 提供 `uint8_t` 等定宽整数类型；`<cstring>` 提供 `std::memcpy` 等。注意 `<cstring>` 在这里其实**并没有被用到**（代码里没有调用任何 `<cstring>` 的函数），这是一个小小的冗余 include。不影响正确性，但严格来说可以删掉。我们在这里指出来，是希望大家养成"每个 include 都要有理由"的习惯。

### 4.2 匿名命名空间：C++ 的"文件私有"

```cpp
namespace {

// ... 八个工具函数 ...

}
```

**匿名命名空间（anonymous namespace）**：在 `namespace { }` 里定义的名字，具有**内部链接（internal linkage）**——它们只在当前翻译单元（.cpp 文件）内可见，其它文件即使声明了同名函数也不会冲突。

对比一下老式的 `static` 关键字：

```cpp
static int packedRowBytes(...) { ... }   // C 风格：也是内部链接
```

两种都能达到"文件私有"的效果，但匿名命名空间**更现代、更彻底**——它对类型（class/struct）也有效，而 `static` 只能用于函数和变量。C++ 标准推荐用匿名命名空间。

**为什么要文件私有？** 因为这些工具函数（`clampByte`、`packedRowBytes` 等）是**实现细节**。如果它们有外部链接，那么另一个 .cpp 里定义一个同名的 `clampByte` 就会在链接时冲突（或更糟——被静默地解析成同一个函数）。私有化之后，每个文件可以有自己的 `clampByte`，互不干扰。这是**命名空间污染（namespace pollution）**的经典防御。

### 4.3 工具一：`clampByte` —— 把任意整数压到 0~255

```cpp
inline uint8_t clampByte(int value)
{
    if (value < 0)
        return 0;
    if (value > 255)
        return 255;
    return static_cast<uint8_t>(value);
}
```

**这个函数在解决什么问题？**

回顾 1.8 节的公式：`r = (298 * c + 409 * e + 128) >> 8`。这里的 `c`、`d`、`e` 都是有范围的（`c` 最大 235-16=219，`e` 最小 16-128=-112），算出来的 `r/g/b` **在数学上应该落在 0~255**，但——

**"应该"不等于"一定"**。两种情况会越界：

1. **非法/超范围的输入**：如果摄像头给了 Y=255、V=255（全范围数据被当成有限范围解读），`c = 239`，`e = 127`，`r = (298*239 + 409*127 + 128) >> 8 = (71222 + 51943 + 128) >> 8 = 123293 >> 8 = 481`——**远超 255**。
2. **数学上的边缘**：即使输入合法，某些极端组合也会略微溢出。

如果不 clamp 会怎样？ `static_cast<uint8_t>(481)` 的结果是 **481 - 256 = 225**（截断高位，只保留低 8 位）。于是"应该是纯白 255"的像素变成了 **225 灰色**，甚至更糟——**255 变成 0，纯白变纯黑**。这在画面上是**黑点/白点噪点**，非常难看。

所以 clamp（钳位，把值限制在区间内）是必须的一步。而 clamp 的正确顺序是：**先判断越界，再转换类型**。如果先 `static_cast` 再判断，就已经丢失信息了。

**为什么用 `inline`？** 因为这个函数会在每像素的循环里被调用（1080p 一帧 200 万次）。`inline` 是给编译器的**建议**："请把函数体直接嵌到调用处，省掉函数调用开销"。现代编译器（开了 -O2）自己也会做这个优化，加上 `inline` 是双保险，也表达了"这个函数就是为内联而生"的意图。

**为什么参数是 `int` 而不是 `uint8_t`？** 关键！因为我们要判断 `value < 0`——**如果参数是无符号类型，`value < 0` 永远为假，编译器甚至会警告"这个比较永远为 false"**。中间的算术结果是 `int`（有符号），所以用 `int` 接收。这是很多新手会踩的坑：**无符号类型做下界检查是无效的**。

### 4.4 工具二：`packedRowBytes` —— 一行的"有效"字节数

```cpp
int packedRowBytes(CameraPixelFormat format, int width)
{
    switch (format) {
    case CameraPixelFormat::RGB24:
        return width * 3;
    case CameraPixelFormat::RGB32:
        return width * 4;
    case CameraPixelFormat::YUY2:
    case CameraPixelFormat::UYVY:
        return ((width + 1) / 2) * 4;
    default:
        return 0;
    }
}
```

这个函数回答的问题是：**"给定宽度和格式，一行里'真正有用的'字节是多少？"**

注意关键词是"**有效**"——它**不包括 stride 的填充**。这是它和第 5 讲 `CameraCapturer` 里那个 `dibRowBytes` 的根本区别：

| 函数 | 所在文件 | 算的是什么 | 公式 |
|---|---|---|---|
| `dibRowBytes(w, bits)` | `CameraCapturer.cpp` | 摄像头**实际给的**行跨度（含 DIB 对齐填充） | `((w * bits + 31) / 32) * 4` |
| `packedRowBytes(fmt, w)` | `CameraFrameConverter.cpp` | 一行**有效数据**的最小字节数（不含填充） | 见上 |

两者都必须存在，因为：
- 我们用 `stride`（来自采集层，等于 `dibRowBytes`）来**跳行**；
- 我们用 `rowBytes`（本函数的结果）来**算缓冲区最小需要多大**、以及**最后一行不需要填充**。

逐个 case 看：

- **RGB24**：每像素 3 字节（B、G、R），所以 `width * 3`。
- **RGB32**：每像素 4 字节（B、G、R、X），所以 `width * 4`。
- **YUY2 / UYVY**：两个 case 共用一个 return（C++ 的 case 穿透/fall-through）。公式是 1.7 节推导过的 `((width + 1) / 2) * 4`——**宏像素个数 × 4 字节**。
- **default（Unknown 和 MJPG）**：返回 0。**返回 0 是一个"我不知道"的信号**，调用方（`convert()` 第 335 行）会检查 `rowBytes <= 0` 并报错。为什么 MJPG 也返回 0？因为 MJPG 是压缩流，**根本没有"一行"的概念**——它会在 `convert()` 里被提前分流到 `convertMjpeg`，根本走不到这里。

> **思考**：为什么不把 `packedRowBytes` 做成 `CameraCapturer.h` 里的公共函数？因为它是**转换层的概念**（"我要按行吃数据，所以我要知道一行有多少"），采集层不关心。放在转换层的匿名命名空间里，是恰当的归属。代价是测试程序里又抄了一遍（见 7.2 节）。

### 4.5 工具三：`avErrorText` —— FFmpeg 错误码翻译器

```cpp
std::string avErrorText(int err)
{
    char buffer[AV_ERROR_MAX_STRING_SIZE] = {};
    av_strerror(err, buffer, sizeof(buffer));
    return buffer;
}
```

三行代码，把 FFmpeg 的负错误码翻译成人话。

- **`AV_ERROR_MAX_STRING_SIZE`**：FFmpeg 在 `libavutil/error.h` 里定义的常量（当前是 64），保证够装任何错误描述。**用它而不是自己写 256，是为了跟着 FFmpeg 的版本走**。
- **`= {}`**：C++ 的**值初始化（value initialization）**语法，把整个数组清零。为什么要清零？因为 `av_strerror` 在遇到**未知错误码**时，会写入类似 `"Unknown error -12345"` 的东西，但**不保证**在所有路径上都正确终止字符串。清零保证最坏情况下也是个空字符串，不会读到数组外的内存。
- **`sizeof(buffer)`**：传数组大小。这里 `buffer` 还是数组类型（没有退化成指针），所以 `sizeof` 正确。
- **`return buffer;`**：隐式把 `char[]` 转成 `std::string`。只要 buffer 里有正确的 `\0` 结尾就安全。

这个函数让我们能写出这样的错误信息：

```cpp
lastError = "打开 MJPEG 解码器失败: " + avErrorText(openRet);
```

用户看到的是 `打开 MJPEG 解码器失败: Invalid data found when processing input`，而不是 `-1094995529`。**好的错误信息是排障效率的分水岭**，这几行代码投入产出比极高。

### 4.6 工具四：`yuvToBgra` —— 今天最核心的 15 行

```cpp
inline void yuvToBgra(int y, int u, int v, uint8_t* dst)
{
    const int c = y - 16;
    const int d = u - 128;
    const int e = v - 128;

    const int r = (298 * c + 409 * e + 128) >> 8;
    const int g = (298 * c - 100 * d - 208 * e + 128) >> 8;
    const int b = (298 * c + 516 * d + 128) >> 8;

    dst[0] = clampByte(b);
    dst[1] = clampByte(g);
    dst[2] = clampByte(r);
    dst[3] = 255;
}
```

这 15 行是今天全部 YUV 处理的心脏，1.8 和 1.9 两节的公式在这里落地。我们逐行拆：

**第 47~49 行：减去偏移量（offset removal）。**

```cpp
const int c = y - 16;      // 亮度去掉黑台阶：16 对应黑，235 对应白
const int d = u - 128;     // 色度是"有符号"的：128 是零（无色偏）
const int e = v - 128;
```

- **`c`（contrast？其实是 Y 的居中值）**：Y 的有效范围是 16~235，减去 16 之后变成 0~219。这一步是 1.9 节讲的"**去黑台阶**"。
- **`d` 和 `e`**：U/V 是**双极性**的——128 表示"色度为零"（灰色），小于 128 偏一种颜色，大于 128 偏另一种。减去 128 变成 -112~+112 的有符号值，这样后面的乘法才有意义（想想看：如果 U=128 时不减 128，`516 * 128` 会给蓝色加一个巨大的常量，灰画面会变成纯蓝）。

**命名为什么是 c/d/e 而不是 y0/u0/v0？** 这是从经典参考实现（很多教材和开源代码）里传下来的命名习惯，对应"三项输入"。可读性上确实不如 `yCentered` 之类的名字，但因为它和公式一一对应，熟悉的人反而看得更快。我们保持原样，但在讲课时要说明。

**第 51~53 行：BT.601 整数矩阵。**

```cpp
const int r = (298 * c + 409 * e + 128) >> 8;
const int g = (298 * c - 100 * d - 208 * e + 128) >> 8;
const int b = (298 * c + 516 * d + 128) >> 8;
```

对应 1.8 节的解释：
- `298 = 1.164 × 256`（亮度增益）
- `409 = 1.596 × 256`（V 对 R 的贡献）
- `-100 = -0.392 × 256`（U 对 G 的削减）
- `-208 = -0.813 × 256`（V 对 G 的削减）
- `516 = 2.017 × 256`（U 对 B 的贡献）
- `+128`：加半个除数（256/2）实现四舍五入
- `>> 8`：除以 256

**观察几个有趣的性质：**

1. **R 的公式里没有 `d`（U）**：红色**只由亮度和 V 决定**，与 U 无关。这是 YUV 定义的直接推论（V = R−Y 相关的分量）。
2. **B 的公式里没有 `e`（V）**：蓝色**只由亮度和 U 决定**。同理。
3. **G 的公式里两者都有，且都是负系数**：绿色 = 亮度 − 蓝色成分 − 红色成分。这也符合直觉：RGB 里绿色是"剩下的"。
4. **G 的系数绝对值（100, 208）比 B 的（516）小**：说明 U 主要影响蓝色，对绿色影响小。

**这三行解释了 1.7 节那个排障结论**：如果 U 和 V 弄反（`d` 和 `e` 互换），则 R 和 B 的公式互换 → **红蓝对调**。我们现在是从公式层面证明了这个现象。

**第 55~58 行：写入 BGRA。**

```cpp
dst[0] = clampByte(b);      // B 在第 0 个字节
dst[1] = clampByte(g);      // G
dst[2] = clampByte(r);      // R 在 B 之后！
dst[3] = 255;               // A 永远不透明
```

注意顺序：**先写 `b`，再写 `g`，最后写 `r`**。变量名是 `r`、`g`、`b`（人类习惯），但写入顺序是 `b`、`g`、`r`（BGRA 内存序）。**这就是 1.3 节讲的"BGR 与 RGB 是同一块数据的两种读法"在代码里的体现。**

如果谁不小心写成 `dst[0] = r; dst[2] = b;`，画面就是经典的**红蓝对调**——和 U/V 弄反的现象一模一样！所以排障时要分两步查：

| 现象 | 可能原因 A | 可能原因 B |
|---|---|---|
| 红蓝对调 | U/V 读反（YUY2 当 UYVY 读） | BGRA 写成 RGBA |
| 整体发暗、灰蒙蒙 | 忘了减 16/128（没做范围拉伸） | 误用了全范围系数 |
| 画面有斜纹 | stride 没用对 | — |
| 画面上下颠倒 | `flipVertical` 没生效 | — |

**第 58 行 `dst[3] = 255;`**：强制不透明。因为我们统一到 BGRA 的约定就是"A 恒为 255"。即使是 RGB32 输入（第 4 个字节是垃圾值 X），我们也覆盖成 255。**统一格式的本质就是"消除歧义"**——下游不需要猜这个字节是什么。

**为什么 `dst` 是裸指针 `uint8_t*`？** 因为这个函数在每像素的热循环里，用裸指针比用容器引用开销小、也更灵活（调用者传的是"当前像素位置"）。这是性能敏感代码的常见取舍。安全性由调用方保证（调用方保证 `dst` 后面至少还有 4 字节）。

### 4.7 工具五：`convertRgb24Row` —— 3 字节变 4 字节

```cpp
void convertRgb24Row(const uint8_t* src, uint8_t* dst, int width)
{
    for (int x = 0; x < width; ++x) {
        dst[0] = src[0];
        dst[1] = src[1];
        dst[2] = src[2];
        dst[3] = 255;
        src += 3;
        dst += 4;
    }
}
```

这是最简单的转换，**只是搬运 + 补一个字节**。

**为什么"补一个 255"就完事了？** 因为 RGB24 在内存里已经是 `B G R`（第 5 讲确认过：Windows 的 RGB24 实际是 BGR 顺序），而我们的目标 BGRA 也是 `B G R A` 顺序。**字节顺序完全一致！** 所以不需要交换任何字节，只要在每个像素后面塞一个 `A=255`。

这就是为什么第 1 段测试这么写：

```cpp
setRgb24(raw, 0, 0, 255, 0, 0);        // 造一个"红色"像素
...
check(pixelIs(*out, 0, 0, 0, 0, 255), "(0,0) 是红色");   // 读出来必须是 B=0 G=0 R=255
```

**第 68~69 行：指针推进而不是下标索引。**

```cpp
src += 3;
dst += 4;
```

为什么不用 `src[x*3]` 和 `dst[x*4]`？因为下标形式每次都要做一次乘法（`x*3`、`x*4`），而指针推进是加法。虽然现代编译器通常会做**强度削减（strength reduction）**把乘法优化成累加，但写成指针推进更明确，也更符合"流式处理"的思维。这是 C 语言时代的经典写法，在图像处理代码里依然常见。

**注意 `src` 和 `dst` 是形参，但被修改了。** 参数是 `const uint8_t* src`（指针本身可变，指向的内容不可变），所以 `src += 3` 合法，`*src = ...` 非法。这个 `const` 的位置很重要：`const uint8_t*` = "指向常量的指针"，`uint8_t* const` = "常量指针"。前者常见，后者少见。这里用前者，表达了"**我不会修改源数据**"的承诺。

### 4.8 工具六：`convertRgb32Row` —— 覆盖掉那个垃圾字节

```cpp
void convertRgb32Row(const uint8_t* src, uint8_t* dst, int width)
{
    for (int x = 0; x < width; ++x) {
        dst[0] = src[0];
        dst[1] = src[1];
        dst[2] = src[2];
        dst[3] = 255;
        src += 4;
        dst += 4;
    }
}
```

和 RGB24 版**几乎一模一样**，唯一区别是 `src += 4` 而不是 `src += 3`。

但这里有个**语义上的关键差异**，请同学们特别注意：

- **RGB24 的 `dst[3] = 255` 是"添加"**：源里根本没有第 4 个字节，我们凭空造一个。
- **RGB32 的 `dst[3] = 255` 是"覆盖"**：源里**有**第 4 个字节，但它是**未定义/垃圾值**（可能是 0、可能是上一帧残留、可能是驱动随便填的）。我们**必须覆盖它**，不能照抄。

测试第 4 段就是专门钉死这一点的：

```cpp
setRgb32(raw, 0, 0, 200, 100, 50);     // 内部把第 4 字节写成 0x00
...
const uint8_t alpha = out->bgra[3];
check(alpha == 255, "第 4 字节被强制成不透明");
```

**如果不覆盖会怎样？** 源的第 4 字节是 `0x00`，那么输出的 BGRA 就是 `(50, 100, 200, 0)`——**完全透明**。下游（比如 Qt 的绘制、或者将来的画中画混合）看到 Alpha=0 就会认为"这一帧什么都不显示"，**整个画面消失**。而排查时你会一头雾水："我的 RGB 明明是对的，为什么画面是空的？"

这就是为什么测试注释里写：

```cpp
std::printf("  否则下游看到半透明画面，会以为是自己混色算错了。\n");
```

**"上游的脏数据会在下游变成别人的 bug"**——这是分层系统里最昂贵的教训。统一格式层必须**彻底消除歧义**，不能把"可能是什么"的东西传下去。

> **可优化点**：RGB32 → BGRA 其实只差一个"把第 4 字节改成 255"。理论上可以用 `memcpy` 拷整行再循环改 alpha，或者用 SIMD 一次处理 4 个像素。当前实现是逐像素，简单但非最优。我们放到第九章讨论。

### 4.9 工具七：`convertYuy2Row` —— 解包宏像素

```cpp
void convertYuy2Row(const uint8_t* src, uint8_t* dst, int width)
{
    for (int x = 0; x < width; x += 2) {
        const int y0 = src[0];
        const int u = src[1];
        const int y1 = src[2];
        const int v = src[3];

        yuvToBgra(y0, u, v, dst);
        dst += 4;

        if (x + 1 < width) {
            yuvToBgra(y1, u, v, dst);
            dst += 4;
        }
        src += 4;
    }
}
```

这是今天第一个"真正有内容"的转换。逐行讲：

**第 87 行：`x += 2` 而不是 `++x`。** 因为每轮循环处理**两个像素**（一个宏像素）。这是 4:2:2 的本质。

**第 88~91 行：解包。**

```cpp
const int y0 = src[0];     // 第 0 个像素的亮度
const int u  = src[1];     // 两个像素共用的 U
const int y1 = src[2];     // 第 1 个像素的亮度
const int v  = src[3];     // 两个像素共用的 V
```

这四行的顺序就是 1.7 节那张 `Y0 U Y1 V` 表的代码化。

**为什么用 `int` 而不是 `uint8_t` 接收？** 因为 `yuvToBgra` 的参数是 `int`（它要做减法，需要能表示负数）。如果用 `uint8_t`，`y - 16` 会先做**整数提升（integer promotion）**到 `int` 再减，其实也没问题；但显式用 `int` 更清楚，也避免某些编译器对 `uint8_t` 的意外窄化警告。

**第 93~94 行：转换第 0 个像素。**

```cpp
yuvToBgra(y0, u, v, dst);
dst += 4;
```

注意传的是 `y0`（第一个像素的亮度）和**同一套** `u, v`。

**第 96~99 行：转换第 1 个像素——注意边界检查！**

```cpp
if (x + 1 < width) {
    yuvToBgra(y1, u, v, dst);
    dst += 4;
}
```

**这是这段代码里最容易被忽略、也最重要的一行。** 为什么要检查 `x + 1 < width`？

因为**宽度可能是奇数**！如果 `width = 3`：
- `x=0`：处理像素 0 和 1（`x+1=1 < 3` ✓）
- `x=2`：处理像素 2（`x+1=3` 不 < 3 ✗，**只写像素 2**），`y1` 被丢弃

如果没有这个检查，我们会**多写一个像素到 `dst`**，而 `dst` 的缓冲区只分配了 `width * 4` 字节——**缓冲区溢出（buffer overflow）**！写到了不属于我们的内存里。在 Release 模式下它可能"看起来正常"（覆盖了相邻结构体的某个字段），在 Debug 模式下可能直接崩溃。**这是典型的"只在奇数宽度下触发"的隐藏 bug**，而奇数宽度的摄像头分辨率是存在的（比如 1919×1080、641×480 这类裁剪过的分辨率）。

这个检查体现了**防御性编程**：即使"理论上"摄像头不会给奇数宽度，我们也不赌。

**第 100 行：`src += 4;`** 推进一个宏像素。注意它放在循环体**最后**，且在 `if` 之外——无论有没有写第二个像素，源指针都要前进 4 字节。

**一个细节**：循环条件 `x < width`，当 `width` 是奇数时，最后一轮 `x = width-1`（偶数），`src` 会指向最后一个（不完整的）宏像素。这是对的，因为 `packedRowBytes` 用向上取整保证了缓冲区里有这 4 个字节。

### 4.10 工具八：`convertUyvyRow` —— 只差字节顺序

```cpp
void convertUyvyRow(const uint8_t* src, uint8_t* dst, int width)
{
    for (int x = 0; x < width; x += 2) {
        const int u = src[0];
        const int y0 = src[1];
        const int v = src[2];
        const int y1 = src[3];

        yuvToBgra(y0, u, v, dst);
        dst += 4;

        if (x + 1 < width) {
            yuvToBgra(y1, u, v, dst);
            dst += 4;
        }
        src += 4;
    }
}
```

**和 `convertYuy2Row` 逐字符对比，你会发现：只有第 107~110 行（解包那四行）的顺序不同，其余完全一致。**

```cpp
// YUY2                          // UYVY
const int y0 = src[0];           const int u  = src[0];
const int u  = src[1];           const int y0 = src[1];
const int y1 = src[2];           const int v  = src[2];
const int v  = src[3];           const int y1 = src[3];
```

**关键洞察：解包之后，后面的 `yuvToBgra(y0, u, v, dst)` 调用完全一样！** 因为解包的意义就是"把字节流还原成语义变量"，一旦还原成 `y0/u/v/y1`，格式差异就消失了。

这是一个很好的**抽象示范**：**差异被隔离在"解包"这一步，后续处理完全复用**。如果我们把这两个函数合并，可以写成"先按格式解包，再共用转换"：

```cpp
void convertPacked422Row(const uint8_t* src, uint8_t* dst, int width, bool isUyvy)
{
    for (int x = 0; x < width; x += 2) {
        int y0, u, y1, v;
        if (isUyvy) { u = src[0]; y0 = src[1]; v = src[2]; y1 = src[3]; }
        else        { y0 = src[0]; u  = src[1]; y1 = src[2]; v  = src[3]; }
        // 以下共用...
    }
}
```

但作者选择了**写成两个函数**。为什么？因为分支放在**循环外**（函数选择）比放在**循环内**（每像素判断）快——这叫**循环外提（loop unswitching）**，是一种重要的优化手法。每帧百万次循环里省掉一次分支，是值得的。**"重复一点代码换性能"在热路径上是合理的取舍。**

### 4.11 上半场小结：八件工具的分工

| 工具 | 行号 | 职责 | 被调用者 |
|---|---|---|---|
| `clampByte` | 14–21 | 把 int 压到 0~255 | `yuvToBgra` |
| `packedRowBytes` | 23–36 | 算一行的有效字节数 | `convert()` |
| `avErrorText` | 38–43 | FFmpeg 错误码 → 字符串 | `ensureDecoder` / `convertMjpeg` |
| `yuvToBgra` | 45–59 | BT.601 公式，写一个 BGRA 像素 | `convertYuy2Row` / `convertUyvyRow` |
| `convertRgb24Row` | 61–71 | 整行 RGB24 → BGRA | `convert()` |
| `convertRgb32Row` | 73–83 | 整行 RGB32 → BGRA | `convert()` |
| `convertYuy2Row` | 85–102 | 整行 YUY2 → BGRA | `convert()` |
| `convertUyvyRow` | 104–121 | 整行 UYVY → BGRA | `convert()` |

**注意一个设计规律：所有"整行"函数都遵循同一个签名 `(const uint8_t* src, uint8_t* dst, int width)`。** 这不是巧合——统一签名让 `convert()` 里的 `switch` 可以写得极其干净（见 6.7 节）。这叫**统一接口（uniform interface）**，是让调用方代码简洁的前提。

---

## 第五章：`Impl` 与 MJPEG 解码链路

从第 125 行开始，我们进入 FFmpeg 的世界。

### 5.1 `Impl` 结构体：全景状态

```cpp
struct CameraFrameConverter::Impl {
    std::string lastError;

    AVCodecContext* mjpegCtx = nullptr;
    AVFrame* decodedFrame = nullptr;
    AVPacket* packet = nullptr;

    SwsContext* sws = nullptr;
    int swsWidth = 0;
    int swsHeight = 0;
    int swsFormat = -1;

    ~Impl()
    {
        releaseDecoder();
    }

    void releaseDecoder()
    {
        if (sws) {
            sws_freeContext(sws);
            sws = nullptr;
        }
        if (packet) {
            av_packet_free(&packet);
            packet = nullptr;
        }
        if (decodedFrame) {
            av_frame_free(&decodedFrame);
            decodedFrame = nullptr;
        }
        if (mjpegCtx) {
            avcodec_free_context(&mjpegCtx);
            mjpegCtx = nullptr;
        }
        swsWidth = 0;
        swsHeight = 0;
        swsFormat = -1;
    }

    bool ensureDecoder();
    bool convertMjpeg(const CameraRawFrame& raw, VideoFrame& out);
};
```

**这是今天的"状态全景图"。** 七个数组成员分成三组，我们一组组看。

#### 5.1.1 第一组：MJPEG 解码三件套（第 128~130 行）

```cpp
AVCodecContext* mjpegCtx = nullptr;
AVFrame* decodedFrame = nullptr;
AVPacket* packet = nullptr;
```

这三个是 FFmpeg 解码的**标准三件套**，缺一不可：

- **`AVCodecContext`（编解码器上下文）**：解码器的"工作台"。它持有解码器的所有运行时状态——内部缓冲区、Huffman 表、量化表、以及我们设置的参数（宽高、线程数等）。**它是最重的对象**，创建和销毁都贵，所以必须复用。
- **`AVFrame`（帧）**：解码**输出**。一帧原始像素数据。它是"容器"——解码器会把解出来的数据填进去（或让它引用内部缓冲）。
- **`AVPacket`（包）**：解码**输入**。一个压缩数据包。对 MJPEG 来说，一个 packet 就是一张 JPEG 的全部字节。

**为什么这三个都要缓存而不是每帧创建？** 因为 MJPEG 解码器每次创建都要重新初始化 Huffman 表等一堆东西，而我们是**每秒 30 次**地调用。缓存下来之后，每帧只是 `send_packet` / `receive_frame` 两个调用。这是 FFmpeg 编程的第一条性能铁律。

**为什么是裸指针？** 因为 FFmpeg 是 C 库，它返回裸指针。我们的职责是用 C++ 的机制把它们管好——这就是 `~Impl()` 和 `releaseDecoder()` 存在的理由。

#### 5.1.2 第二组：swscale 上下文与它的三个"指纹"（第 132~135 行）

```cpp
SwsContext* sws = nullptr;
int swsWidth = 0;
int swsHeight = 0;
int swsFormat = -1;
```

`sws` 是 1.13 节讲的图像变换上下文。后面三个 `swsWidth/swsHeight/swsFormat` 是**缓存键（cache key）**——它们记录"当前这个 `sws` 是为哪种输入尺寸和格式创建的"。

**为什么要记这三个？** 因为 `sws_getContext` 很贵（要算一堆表），我们想复用。但只有当**输入参数没变**时才能复用。所以每次转换前要比较："现在的输入和我上次建上下文时的输入一样吗？"

**`swsFormat` 初值是 `-1` 而不是 `0`**，这是一个精心选择。因为 FFmpeg 的 `AVPixelFormat` 枚举里 `AV_PIX_FMT_NONE = -1`，而 `0` 是 `AV_PIX_FMT_YUV420P`（一个合法格式）。用 `-1` 表示"还没设置过"，语义上最准确——**用枚举里的"无效值"做初值，而不是随便挑个 0**。

同时 `sws = nullptr` 也表示"还没创建"。两个判据其实冗余，但冗余在这里是好事——双保险。

#### 5.1.3 析构：RAII 的兜底（第 137~140 行）

```cpp
~Impl()
{
    releaseDecoder();
}
```

**RAII（Resource Acquisition Is Initialization，资源获取即初始化）**：C++ 最核心的资源管理思想——**把资源的生命周期绑定到对象的生命周期**。对象构造时获取资源，对象析构时释放资源。这样只要对象正常离开作用域，资源必然被释放，**不可能泄漏**。

`~Impl()` 只有一行，但它保证了：只要 `CameraFrameConverter` 对象被销毁（无论是正常离开作用域、还是异常处理栈展开），`unique_ptr<Impl>` 会析构 `Impl`，`~Impl()` 会调用 `releaseDecoder()`，**四个 FFmpeg 资源全部释放**。

这就是把 C 库裹进 C++ 的标准姿势：**成员用裸指针，析构函数里统一释放**。

#### 5.1.4 `releaseDecoder()`：释放顺序是有讲究的（第 142~163 行）

```cpp
void releaseDecoder()
{
    if (sws) {
        sws_freeContext(sws);
        sws = nullptr;
    }
    if (packet) {
        av_packet_free(&packet);
        packet = nullptr;
    }
    if (decodedFrame) {
        av_frame_free(&decodedFrame);
        decodedFrame = nullptr;
    }
    if (mjpegCtx) {
        avcodec_free_context(&mjpegCtx);
        mjpegCtx = nullptr;
    }
    swsWidth = 0;
    swsHeight = 0;
    swsFormat = -1;
}
```

**这段代码有四个值得学习的点：**

**点一：每个 `if` 都检查空指针。** 为什么？因为 `releaseDecoder()` 会被调用**多次**：
- 从 `~Impl()` 调用（正常销毁）；
- 从 `ensureDecoder()` 的第 199 行调用（分配 frame/packet 失败时的清理）。

第二次调用时，某些指针可能已经是 `nullptr`（因为 `xxx_free` 已经把它们置空了）。检查空指针让它**幂等（idempotent）**——调用一次和调用多次效果一样。而 FFmpeg 的 `xxx_free` 系列对 `nullptr` 也是安全的（内部会检查），所以双重保险。

**点二：释放顺序——从"依赖者"到"被依赖者"。**

- `sws` 最独立（它只是个转换表），先释放。
- `packet`（输入）和 `decodedFrame`（输出）：它们引用的是数据缓冲，和 codec 有关联但相对独立。
- `mjpegCtx` 最后：因为**它可能持有对内部帧缓冲的引用**。如果先释放了 codec，而 frame 还引用着 codec 分配的缓冲，就有悬空风险。

严格的顺序应该是：**先释放"引用别人"的，再释放"被别人引用"的**。这里 `decodedFrame` 可能引用 `mjpegCtx` 的内部分配，所以 frame 要在 codec 之前释放——**代码确实是这样做的**（frame 在 152 行，codec 在 156 行）。✓

**点三：`xxx_free(&ptr)` 会自动置空，为什么还要手动 `= nullptr`？**

确实，`av_frame_free(&decodedFrame)` 内部会执行 `*decodedFrame = NULL`。手动再置空是冗余的。但这是**防御性冗余**——万一将来 FFmpeg 改了行为，或者有人换了个不置空的释放函数，这里还能保底。而且它让代码**自解释**：读代码的人一眼看到"这个指针现在空了"，不用去查 FFmpeg 文档。**在资源释放这种关键路径上，冗余的清晰胜过简洁。**

**点四：最后重置三个缓存键。**

```cpp
swsWidth = 0;
swsHeight = 0;
swsFormat = -1;
```

这三个必须在释放后重置，否则下次调用时，`sws == nullptr` 但 `swsWidth` 还是旧值，判断逻辑（第 251 行 `if (!sws || swsWidth != width || ...)`）虽然因为 `!sws` 短路而得出正确结论（重建），但状态不自洽。**保持状态自洽是避免诡异 bug 的基础。**

#### 5.1.5 两个前向声明的方法（第 165~166 行）

```cpp
bool ensureDecoder();
bool convertMjpeg(const CameraRawFrame& raw, VideoFrame& out);
```

在结构体里声明、在结构体外定义。为什么不直接定义在里面？

**因为定义在类内会被隐式 `inline`**，而 `convertMjpeg` 有 88 行（206~293），内联它会让代码膨胀，而且可读性差——**声明和实现分离，让 `Impl` 的定义成为一个清晰的"状态清单"，一眼看完有什么**。这是很好的组织习惯。

### 5.2 `ensureDecoder()`：懒加载（lazy initialization）

```cpp
bool CameraFrameConverter::Impl::ensureDecoder()
{
    if (mjpegCtx)
        return true;

    const AVCodec* codec = avcodec_find_decoder(AV_CODEC_ID_MJPEG);
    if (!codec) {
        lastError = "找不到 MJPEG 解码器（这版 FFmpeg 没编译进去？）";
        return false;
    }

    mjpegCtx = avcodec_alloc_context3(codec);
    if (!mjpegCtx) {
        lastError = "avcodec_alloc_context3 失败";
        return false;
    }

    mjpegCtx->thread_count = 1;

    const int openRet = avcodec_open2(mjpegCtx, codec, nullptr);
    if (openRet < 0) {
        lastError = "打开 MJPEG 解码器失败: " + avErrorText(openRet);
        avcodec_free_context(&mjpegCtx);
        return false;
    }

    decodedFrame = av_frame_alloc();
    packet = av_packet_alloc();
    if (!decodedFrame || !packet) {
        lastError = "分配解码缓冲区失败";
        releaseDecoder();
        return false;
    }

    return true;
}
```

#### 5.2.1 懒加载：为什么不在构造函数里初始化

**第 171~172 行是整个函数的灵魂：**

```cpp
if (mjpegCtx)
    return true;
```

这叫**懒加载（lazy initialization）**——**不到真正需要，绝不创建**。

为什么？因为**只有 MJPG 摄像头才需要解码器**。如果一个用户用的是 YUY2 摄像头，他一辈子都不会调用 `convertMjpeg`，也就永远不需要 `mjpegCtx`。如果在构造函数里无条件创建，那么每个 YUY2 用户都要白白付一份 MJPEG 解码器的内存和初始化开销。

**懒加载的三个好处：**
1. **省资源**：不用的不分配；
2. **启动快**：构造 `CameraFrameConverter` 是零成本的（只 `make_unique<Impl>`）；
3. **容错**：即使这台机器的 FFmpeg 没编译 MJPEG 解码器，`CameraFrameConverter` 对象依然能构造成功，YUY2/RGB 转换照样能用——**只在你真的用 MJPG 时才报错**。这叫**故障隔离（fault isolation）**，非常优雅。

**注意 `return true` 而不是 `return false`**：已经初始化过了，就是"确保成功"了。这个函数的语义是"**保证**解码器可用"，已经可用当然返回真。

#### 5.2.2 第一步：找解码器（第 174~178 行）

```cpp
const AVCodec* codec = avcodec_find_decoder(AV_CODEC_ID_MJPEG);
if (!codec) {
    lastError = "找不到 MJPEG 解码器（这版 FFmpeg 没编译进去？）";
    return false;
}
```

**`avcodec_find_decoder`**：FFmpeg 的编解码器注册表查询函数。FFmpeg 编译时通过 `--enable-decoder=mjpeg` 之类的开关决定编解码哪些格式，运行时用一个全局注册表管理。这个函数按 `AVCodecID` 查表。

**`const AVCodec*`**：注意是**常量指针**——`AVCodec` 是 FFmpeg 内部的静态描述结构（描述"这个解码器叫什么、支持什么"），**不归我们所有，不需要释放**。这和 `AVCodecContext`（我们分配的）完全不同。FFmpeg 里大量这种"静态描述符 vs 运行时实例"的二分，要分清楚：

| 类型 | 是否我们分配 | 是否要释放 |
|---|---|---|
| `const AVCodec*`（描述符） | 否，查表得到 | **否** |
| `AVCodecContext*`（实例） | 是，`alloc_context3` | 是，`avcodec_free_context` |
| `AVFrame*` | 是，`av_frame_alloc` | 是，`av_frame_free` |
| `AVPacket*` | 是，`av_packet_alloc` | 是，`av_packet_free` |

**错误信息写得非常好：**
```cpp
lastError = "找不到 MJPEG 解码器（这版 FFmpeg 没编译进去？）";
```
它不仅说"失败了"，还**给出了最可能的原因**（FFmpeg 编译配置问题）和**排查方向**（去检查你的 FFmpeg 构建）。这就是好错误信息的样子——**告诉用户下一步该干什么**。

#### 5.2.3 第二步：分配上下文（第 180~184 行）

```cpp
mjpegCtx = avcodec_alloc_context3(codec);
if (!mjpegCtx) {
    lastError = "avcodec_alloc_context3 失败";
    return false;
}
```

`avcodec_alloc_context3(codec)`：分配一个 `AVCodecContext` 并**预填该 codec 的默认值**。

**为什么每次分配都要检查失败？** 因为这是 C 库，分配失败返回 `NULL` 而不是抛异常。C++ 程序员在调用 C 库时最容易犯的错就是"忘了检查返回值"。**规则：每一次 FFmpeg 调用都可能失败，都要检查。** 这段代码的每一处 `if` 都是这条规则的体现。

#### 5.2.4 第三步：`thread_count = 1`（第 186 行）

```cpp
mjpegCtx->thread_count = 1;
```

**这一行非常值得讨论，它体现了实时系统的思维方式。**

`thread_count` 控制解码器用几个线程。设成更大的值（比如 4）能让单帧解码更快吗？**对 MJPEG，基本不能**——JPEG 解码本身是逐块（8×8）独立的，理论上可以并行，但 FFmpeg 的 MJPEG 解码器的多线程支持有限，而且要付出线程同步开销。

**那为什么显式设成 1（虽然默认可能就是 1）？** 三个理由：

1. **确定性**：明确声明"我们不要多线程解码"，不受 FFmpeg 版本默认值变化的影响。
2. **避免与上层并发冲突**：我们的推流管线本身是多线程的（采集线程、编码线程、推流线程）。如果解码器再开 4 个线程，在 4 核机器上就会**过度订阅（oversubscription）**——线程数远超 CPU 核数，上下文切换开销反而拖慢整体。
3. **我们的场景不需要**：MJPEG 解码一帧 1080p 大约 2~5 毫秒，30fps 的预算是 33 毫秒，单线程绰绰有余。**用多线程优化一个本来就不慢的东西，是过度优化。**

**这是一个很重要的工程直觉：在实时管线里，"减少线程"往往比"增加线程"更有效**，因为延迟的可预测性比吞吐量更重要。

#### 5.2.5 第四步：打开解码器（第 188~193 行）

```cpp
const int openRet = avcodec_open2(mjpegCtx, codec, nullptr);
if (openRet < 0) {
    lastError = "打开 MJPEG 解码器失败: " + avErrorText(openRet);
    avcodec_free_context(&mjpegCtx);
    return false;
}
```

`avcodec_open2(ctx, codec, options)`：真正初始化解码器（分配内部缓冲、准备各种表）。第三个参数是 `AVDictionary**` 类型的选项字典，传 `nullptr` 表示"用默认选项"。

**注意失败时的清理：**
```cpp
avcodec_free_context(&mjpegCtx);
```
**分配了的东西，失败时要还回去。** 如果这里不释放，`mjpegCtx` 就泄漏了——而且因为 `ensureDecoder` 可能被反复调用（每次 `convertMjpeg` 都会调），泄漏会累积。**泄漏 + 循环调用 = 灾难**，这是最典型的内存泄漏场景。

这个模式叫**失败路径清理（cleanup on failure path）**，是 C 风格代码里最容易漏的地方。同学们以后写类似代码时，养成习惯：**写完一个 `if (失败) return`，立刻问自己"到这里为止分配了什么？该还什么？"**。

#### 5.2.6 第五步：分配帧和包（第 195~201 行）

```cpp
decodedFrame = av_frame_alloc();
packet = av_packet_alloc();
if (!decodedFrame || !packet) {
    lastError = "分配解码缓冲区失败";
    releaseDecoder();
    return false;
}
```

**注意这里的判断方式：`if (!decodedFrame || !packet)`——两个一起判。** 因为如果 `av_frame_alloc` 成功但 `av_packet_alloc` 失败，我们需要清理**已经成功的那个**。用 `||` 合并判断，然后在清理时用 `releaseDecoder()`（它内部每个都检查空指针，幂等）——**这是一个非常干净的组合**：

- 合并判断 → 代码简洁；
- 幂等清理 → 不需要区分"哪个失败了"。

这个"**合并判断 + 幂等清理**"的搭配，是处理多资源分配的标准套路。对比一下如果要分别处理会写成什么样：

```cpp
decodedFrame = av_frame_alloc();
if (!decodedFrame) { lastError = "..."; avcodec_free_context(&mjpegCtx); return false; }
packet = av_packet_alloc();
if (!packet) { lastError = "..."; av_frame_free(&decodedFrame); avcodec_free_context(&mjpegCtx); return false; }
```

啰嗦且容易漏。**幂等清理函数是简化错误处理的关键技巧。**

#### 5.2.7 `ensureDecoder` 小结：五步初始化链

| 步骤 | 行号 | 调用 | 失败时清理 |
|---|---|---|---|
| 0. 已初始化则跳过 | 171–172 | `if (mjpegCtx) return true;` | — |
| 1. 查解码器 | 174–178 | `avcodec_find_decoder` | 无需（没分配） |
| 2. 分配上下文 | 180–184 | `avcodec_alloc_context3` | 无需 |
| 3. 设线程数 | 186 | `thread_count = 1` | — |
| 4. 打开 | 188–193 | `avcodec_open2` | `avcodec_free_context` |
| 5. 分配 frame/packet | 195–201 | `av_frame_alloc` / `av_packet_alloc` | `releaseDecoder()`（全部） |

### 5.3 `convertMjpeg()`：MJPG → BGRA 的全过程

这是今天最长的函数（206~293 行，88 行）。我们分成六段讲。

#### 5.3.1 第一段：确保解码器 + 定义 decode lambda（第 206~222 行）

```cpp
bool CameraFrameConverter::Impl::convertMjpeg(const CameraRawFrame& raw, VideoFrame& out)
{
    if (!ensureDecoder())
        return false;

    const auto decode = [this](const uint8_t* data, size_t size) -> int {
        av_packet_unref(packet);
        packet->data = const_cast<uint8_t*>(data);
        packet->size = static_cast<int>(size);

        const int sendRet = avcodec_send_packet(mjpegCtx, packet);
        if (sendRet < 0)
            return sendRet;

        av_frame_unref(decodedFrame);
        return avcodec_receive_frame(mjpegCtx, decodedFrame);
    };
```

**第 208~209 行**：先确保解码器就位。

**第 211~219 行：一个 lambda（匿名函数）封装"解码一次"的动作。**

为什么要用 lambda？因为**后面要解码两次**（正常一次，EOI 兜底一次），把重复的代码包成 lambda 避免复制粘贴。这是"**重复代码是万恶之源**"的直接应用。

**`[this]` 捕获**：lambda 里用到了 `packet`、`mjpegCtx`、`decodedFrame` 三个成员，所以要捕获 `this`。

**返回类型 `-> int`**：显式声明返回 `int`（即 FFmpeg 的错误码：0 成功，负数是错误）。为什么显式写？因为 lambda 里有两个 `return`（`sendRet` 和 `receive_frame` 的结果），虽然都是 `int`，但显式声明能让**读代码的人一眼看出"这个函数返回的是 FFmpeg 错误码"**，不用去推断。**显式返回类型是自文档化。**

**第 212 行：`av_packet_unref(packet);`（重点！）**

```cpp
av_packet_unref(packet);
```

`unref`（取消引用）的作用：**把 packet 恢复到"刚分配出来"的干净状态**——释放它持有的数据缓冲引用，清零各种字段。

为什么每次都要？因为 `AVPacket` 是**复用的**（我们每帧都用同一个 `packet` 对象）。上一帧的数据还挂在上面，如果不 unref 就直接塞新数据，旧数据的引用就泄漏了（FFmpeg 的引用计数不会归零，缓冲永不释放）。

**这是 FFmpeg 编程最容易犯的错误之一：复用 AVPacket/AVFrame 前忘记 unref。** 症状是内存缓慢增长，跑一小时吃掉几个 G。

**第 213~214 行：手动填 packet（又一个重点）**

```cpp
packet->data = const_cast<uint8_t*>(data);
packet->size = static_cast<int>(size);
```

标准的做法是 `av_packet_from_data(packet, buffer, size)`（它会接管 buffer 所有权）或者 `av_new_packet` + `memcpy`。这里作者选择了**手动赋值**——直接把指针指到 `raw.bytes` 的缓冲区上。

**这样做的好处：零拷贝。** `raw.bytes` 已经是内存里的一张完整 JPEG，我们不需要再拷一份，直接让 packet 指向它。对每帧几百 KB 的数据、每秒 30 帧来说，省掉的拷贝是可观的。

**这样做的风险和 `const_cast`：**
- `packet->data` 的类型是 `uint8_t*`，而我们的 `data` 是 `const uint8_t*`（因为 `raw` 是 const 引用）。所以要 `const_cast` 去掉 const。
- **`const_cast` 是"我知道我在干什么"的声明**。这里安全吗？**安全**，因为：
  1. 解码是**同步**的——`avcodec_receive_frame` 返回时，解码器已经用完这块数据了；
  2. FFmpeg 不会保存这个指针跨调用使用（对于 `send_packet` 的语义，数据在调用期间有效即可）；
  3. packet 没有 `buf`（AVBufferRef）字段，所以 `av_packet_unref` **不会**去 free 这块内存——它只清字段。**这一点至关重要**：如果 packet 有 `buf`，`unref` 会调用 `free` 释放一块不属于 FFmpeg 分配器的内存，直接崩溃。因为我们只设了 `data` 没设 `buf`，`unref` 时 `buf == NULL`，走的是"只重置字段"的分支。✓

> **深入提醒**：这个"手动填 data 不填 buf"的技巧在 FFmpeg 社区是有争议的，因为它依赖 `av_packet_unref` 的内部实现细节。更稳妥的官方做法是 `av_packet_from_data(packet, buf, size)` 配合自己分配的缓冲，或者干脆 `av_new_packet` + `memcpy`。当前写法在 FFmpeg 4.x/5.x/6.x 上都验证可用，但属于"踩在 API 边界上"。我们在第九章会讨论更稳妥的替代方案。

**`static_cast<int>(size)`**：`size_t`（无符号 64 位）转 `int`（有符号 32 位）。如果 JPEG 大于 2 GB 会溢出——**实际上不可能**（MJPEG 单帧最多几 MB），所以这里没有检查。这是合理的：不为不可能发生的情况写代码（呼应"避免过度设计"）。

**第 216~218 行：发送包。**

```cpp
const int sendRet = avcodec_send_packet(mjpegCtx, packet);
if (sendRet < 0)
    return sendRet;
```

失败就直接返回错误码。注意**失败时没有 unref**——因为下一次调用 `decode` 时第 212 行会 unref，所以不会有泄漏。但如果解码器进入了错误状态，后续调用可能持续失败——这正是后面 EOI 兜底要赌的地方。

**第 220~221 行：接收帧。**

```cpp
av_frame_unref(decodedFrame);
return avcodec_receive_frame(mjpegCtx, decodedFrame);
```

同样，先 `av_frame_unref` 清干净再接收。第二个 `unref`，同样的理由（`decodedFrame` 也是复用的）。

**`avcodec_receive_frame` 的返回值语义：**
- `0`：成功拿到一帧；
- `AVERROR(EAGAIN)`（负值）：**还没准备好**，需要再 `send_packet` 更多数据。对 MJPEG 这种一包一帧的格式，正常不会出现；
- `AVERROR_EOF`：流结束；
- 其它负数：真错误。

我们统一用 `ret < 0` 判失败，够用。

#### 5.3.2 第二段：EOI 兜底（第 224~236 行）

```cpp
    int ret = decode(raw.bytes.data(), raw.bytes.size());

    if (ret < 0) {
        size_t trimmed = 0;
        for (size_t i = raw.bytes.size(); i >= 2; --i) {
            if (raw.bytes[i - 2] == 0xFF && raw.bytes[i - 1] == 0xD9) {
                trimmed = i;
                break;
            }
        }
        if (trimmed > 2 && trimmed < raw.bytes.size())
            ret = decode(raw.bytes.data(), trimmed);
    }

    if (ret < 0) {
        lastError = "MJPEG 解码失败: " + avErrorText(ret);
        return false;
    }
```

**第 224 行：第一次尝试，用全部字节。**

**第 226~234 行：失败后的兜底——从后往前找 EOI 标记。**

这是 1.11 节讲过的逻辑。注意几个细节：

**循环写法：`for (size_t i = raw.bytes.size(); i >= 2; --i)`**

- 从 `size` 开始（不是 `size-1`），判断的是 `bytes[i-2]` 和 `bytes[i-1]`——也就是**检查以 `i` 结尾的两字节序列**。
- 条件 `i >= 2`：保证 `i-2 >= 0`，不越界。**用 `size_t`（无符号）时，`i >= 2` 这个写法是安全的**；如果写成 `i >= 0` 就会死循环（无符号数永远 >= 0）。这是无符号下标循环的必备注意事项。

**为什么是"从后往前找"而不是"从前往后找"？**

因为 JPEG 里可能**不止一个** `FF D9`（比如某些摄像头在帧内嵌入了缩略图，或者数据里碰巧出现）。**真正的 EOI 是最后一个**。从后往前找，第一个找到的就是最后一个——一次命中，效率也最高（通常 EOI 就在数据末尾附近）。

**`trimmed = i;`**：`i` 是 EOI 结束后的位置（不含 EOI 的下一字节）。因为 `bytes[i-2]` 和 `bytes[i-1]` 是 EOI 的两个字节，所以截断长度应该是 `i`（包含这两个字节）。✓

**第 231 行的双重条件：`trimmed > 2 && trimmed < raw.bytes.size()`**

- `trimmed > 2`：保证截出来的数据至少有 3 个字节（SOI 是 2 字节，加 1 才有意义）。如果一个 JPEG 只有 SOI+EOI（`FF D8 FF D9`，4 字节），`trimmed` 会是 4，`4 > 2` ✓。
- `trimmed < raw.bytes.size()`：**如果截出来的长度等于原长度，那就白截了**——说明 EOI 就在末尾，问题不在尾部填充，重试没有意义，白白浪费一次解码。**"不要用重试去掩盖真问题"**，这个判断体现了克制。

**第 232 行：用截断后的长度重试。** 注意传的还是 `raw.bytes.data()`——只是长度变小了。**复用同一块内存，只改长度**，零拷贝。

**这个兜底逻辑体现了什么工程思维？** 它体现了"**对上游数据的不信任**"。第 5 讲我们说过"bytes 可能含尾部零填充"，当时只是记录了这个事实，没有处理。现在到了该处理的地方，我们就在**消费端**处理它——这正是"**在能处理的地方处理**"的原则。采集层不知道那是零填充（它只看到字节），转换层知道"这应该是 JPEG，JPEG 以 EOI 结尾"，所以它来做这个判断。**谁有知识，谁负责。**

**第 238~241 行：两次都失败，报错返回。**

```cpp
lastError = "MJPEG 解码失败: " + avErrorText(ret);
```

这里的 `ret` 是**最后一次**的错误码（如果重试过，就是重试的错误码；没重试过就是第一次的）。合理——最近的错误最有诊断价值。

#### 5.3.3 第三段：校验解码结果尺寸（第 243~249 行）

```cpp
    const int width = decodedFrame->width;
    const int height = decodedFrame->height;
    if (width <= 0 || height <= 0) {
        lastError = "MJPEG 解码出来的尺寸无效";
        av_frame_unref(decodedFrame);
        return false;
    }
```

**为什么还要检查？`avcodec_receive_frame` 成功返回不就是解出来了吗？**

防御性编程。理论上成功返回时 `width/height` 必然有效，但：
- JPEG 文件里的尺寸字段（SOF 段）可能被损坏；
- 某些畸形 JPEG 能解出宽高为 0 的帧；
- 如果后面我们用 `width * height * 4` 去 `assign` 一个 vector，宽高为 0 或者负数会导致**分配巨大内存或崩溃**。

**更重要的是第 247 行：`av_frame_unref(decodedFrame);`**

失败路径上也要清理。因为 `decodedFrame` 是复用的，如果不 unref，这个畸形帧会一直挂在上面，占用内存直到下次调用。**失败路径的清理和成功路径同样重要。**

#### 5.3.4 第四段：SwsContext 缓存与重建（第 251~280 行）

```cpp
    if (!sws || swsWidth != width || swsHeight != height
        || swsFormat != decodedFrame->format) {
        if (sws) {
            sws_freeContext(sws);
            sws = nullptr;
        }
        sws = sws_getContext(width, height,
                             static_cast<AVPixelFormat>(decodedFrame->format), width,
                             height, AV_PIX_FMT_BGRA, SWS_BILINEAR, nullptr, nullptr,
                             nullptr);
        if (!sws) {
            lastError = "创建 YUV->BGRA 转换上下文失败";
            av_frame_unref(decodedFrame);
            return false;
        }

        const bool sourceIsFullRange =
            decodedFrame->format == AV_PIX_FMT_YUVJ420P
            || decodedFrame->format == AV_PIX_FMT_YUVJ422P
            || decodedFrame->format == AV_PIX_FMT_YUVJ444P
            || decodedFrame->color_range == AVCOL_RANGE_JPEG;

        const int* coefficients = sws_getCoefficients(SWS_CS_ITU601);
        sws_setColorspaceDetails(sws, coefficients, sourceIsFullRange ? 1 : 0,
                                 coefficients, 1, 0, 1 << 16, 1 << 16);

        swsWidth = width;
        swsHeight = height;
        swsFormat = decodedFrame->format;
    }
```

**第 251~252 行：缓存失效判断。**

```cpp
if (!sws || swsWidth != width || swsHeight != height
    || swsFormat != decodedFrame->format) {
```

四个条件用 `||` 连接，任一成立就重建：
1. `!sws`：从没创建过（第一次调用）；
2. `swsWidth != width`：宽度变了；
3. `swsHeight != height`：高度变了；
4. `swsFormat != decodedFrame->format`：像素格式变了。

**为什么要判断格式变化？** 因为**同一个摄像头可能在中途改变输出格式**（分辨率切换、光照变化触发驱动切换子格式），或者 MJPEG 内部从 4:2:0 变成 4:2:2（某些摄像头在不同分辨率下采样方式不同）。如果格式变了还用旧上下文，**颜色会完全错乱**（按 4:2:0 的方式去读 4:2:2 的数据，画面会变成双影/彩色条纹）。

这里的"三指纹"设计非常经典：**缓存 + 失效检测**。任何"重建很贵、可复用"的对象，都应该配一组"生成它时的输入参数"作为失效判据。数据库里的缓存键、编译器的增量编译判断，都是同一个思想。

**第 253~256 行：先释放旧的，再建新的。**

```cpp
if (sws) {
    sws_freeContext(sws);
    sws = nullptr;
}
```

必须先释放，否则旧的上下文泄漏。置空是双保险（万一 `sws_getContext` 失败，我们不会拿着一个已释放的指针）。

**第 257~260 行：`sws_getContext` 的七个参数。**

```cpp
sws = sws_getContext(width, height,
                     static_cast<AVPixelFormat>(decodedFrame->format), width,
                     height, AV_PIX_FMT_BGRA, SWS_BILINEAR, nullptr, nullptr,
                     nullptr);
```

参数依次是：源宽、源高、源格式、目标宽、目标高、目标格式、缩放算法、源滤波器、目标滤波器、额外参数。

注意几个点：
- **源宽高 = 目标宽高**：我们**不缩放**，只做格式转换。这是正确的——分辨率缩放应该由编码器配置决定（后续课程），转换层不应该擅自改尺寸。
- **源格式是 `decodedFrame->format`**：MJPEG 解码出来通常是 `AV_PIX_FMT_YUVJ420P`（全范围 4:2:0 平面）或 `AV_PIX_FMT_YUVJ422P`（4:2:2）、`AV_PIX_FMT_YUVJ444P`（4:4:4）。**具体是哪一种，取决于 JPEG 文件里的采样因子**——所以我们不能写死，必须动态读。
- **`decodedFrame->format` 的类型是 `int`**，而参数要 `AVPixelFormat` 枚举，所以要 `static_cast`。这是 FFmpeg C API 的常见现象（结构体字段用 `int` 存枚举）。
- **`AV_PIX_FMT_BGRA`**：我们的目标，1.2 节讲过。
- **`SWS_BILINEAR`**：因为不缩放，这个算法其实用不上，但必须给。
- **后三个 `nullptr`**：源/目标滤波器（用于高质量缩放的自定义系数）和参数指针，用默认即可。

**第 261~265 行：创建失败的处理。**

```cpp
if (!sws) {
    lastError = "创建 YUV->BGRA 转换上下文失败";
    av_frame_unref(decodedFrame);
    return false;
}
```

同样，失败路径上 `av_frame_unref`。

**第 267~271 行：判断源数据的色彩范围（这是精华）。**

```cpp
const bool sourceIsFullRange =
    decodedFrame->format == AV_PIX_FMT_YUVJ420P
    || decodedFrame->format == AV_PIX_FMT_YUVJ422P
    || decodedFrame->format == AV_PIX_FMT_YUVJ444P
    || decodedFrame->color_range == AVCOL_RANGE_JPEG;
```

这五行是 1.9 节"有限范围 vs 全范围"的实战。解释背景：

FFmpeg 里有一批"**J 系列**"像素格式（`YUVJ420P` / `YUVJ422P` / `YUVJ444P`），字母 **J 代表 JPEG**。它们是历史遗留：**因为 JPEG 规定用全范围（0~255）**，早期 FFmpeg 就专门造了这几个格式来表示"全范围 YUV"。后来 FFmpeg 改用更通用的方式：格式就是普通的 `YUV420P`，另加一个 `color_range` 字段（`AVCOL_RANGE_JPEG` 表示全范围，`AVCOL_RANGE_MPEG` 表示有限范围）。

**所以判断"源是不是全范围"要两条路都走：**
1. 看是不是 J 系列格式（老式表达）；
2. 看 `color_range` 是不是 `AVCOL_RANGE_JPEG`（新式表达）。

用 `||` 连接——**满足任一即认为是全范围**。这段代码是 FFmpeg 版本兼容的经典写法：新版本可能已经废弃 J 系列格式（改成 `YUV420P` + `color_range`），老版本可能不设 `color_range`。两个都查，两个版本都能正确工作。**这就是兼容性代码的样子：不赌任何一种，都覆盖。**

**如果不做这个判断会怎样？** 如果源是全范围（0~255）而我们按有限范围（16~235）解读：
- 黑色（Y=0）会被算成 `c = 0 - 16 = -16`，`r = 298*(-16) >> 8 = -18` → clamp 到 0。碰巧还是 0。
- 白色（Y=255）会被算成 `c = 239`，`r = 298*239 >> 8 = 278` → clamp 到 255。碰巧还是 255。
- **但中间调全错了**：Y=128 会被算成 130 而不是 128，Y=64 会被算成 `(64-16)*1.164 = 56` 而不是 64。

整体效果是**画面对比度过大、暗部发黑、亮部发白（"过冲"）**，灰阶被压缩在两端。这正是 1.9 节说的那种"看起来能用但就是不对"的 bug。

**第 273~275 行：设置色彩空间细节。**

```cpp
const int* coefficients = sws_getCoefficients(SWS_CS_ITU601);
sws_setColorspaceDetails(sws, coefficients, sourceIsFullRange ? 1 : 0,
                         coefficients, 1, 0, 1 << 16, 1 << 16);
```

`sws_getCoefficients(SWS_CS_ITU601)`：取 BT.601 的转换系数表（1.8 节那三个矩阵，不过是 swscale 内部用的整数表格式）。返回 `const int*`——**静态表，不需要释放**。

`sws_setColorspaceDetails` 的参数依次是：

| 位置 | 实参 | 含义 |
|---|---|---|
| 1 | `sws` | 上下文 |
| 2 | `coefficients` | **源→RGB** 的转换矩阵 |
| 3 | `sourceIsFullRange ? 1 : 0` | **源**是不是全范围（1=是，0=否） |
| 4 | `coefficients` | **RGB→目标** 的转换矩阵（同样用 BT.601） |
| 5 | `1` | **目标**是不是全范围（**1 = 是**，我们要 0~255 的 BGRA） |
| 6 | `0` | brightness（亮度偏移），0 = 不调 |
| 7 | `1 << 16` | contrast（对比度），`1<<16` 在 16.16 定点数里表示 **1.0**，即不调 |
| 8 | `1 << 16` | saturation（饱和度），同样是 1.0，不调 |

**为什么要显式设置？** 因为 `sws_getContext` 的默认行为是"按格式猜"——它看到 `YUVJ420P` 会猜全范围，看到 `YUV420P` 会猜有限范围。但**"猜"在不同 FFmpeg 版本上行为不同**（尤其是 J 系列格式被废弃的过程中，默认行为变过好几次）。我们的判断逻辑已经算出了准确答案，那就**显式告诉 swscale，别让它猜**。**显式优于隐式，尤其是在版本差异频繁的地方。**

**第 16.16 定点数**：把整数的高 16 位当整数部分、低 16 位当小数部分，`1 << 16 = 65536` 表示 1.0，`0.5 << 16 = 32768` 表示 0.5。这是没有浮点单元的时代传下来的技巧，FFmpeg 里遍地都是。

**第 277~279 行：更新三个缓存键。** 建好上下文后把参数记下来，下次就不用重建了。

#### 5.3.5 第五段：真正干活 —— sws_scale（第 282~289 行）

```cpp
    out.width = width;
    out.height = height;
    out.bgra.assign(static_cast<size_t>(width) * height * 4, 0);

    uint8_t* dstData[4] = {out.bgra.data(), nullptr, nullptr, nullptr};
    int dstLinesize[4] = {width * 4, 0, 0, 0};
    sws_scale(sws, decodedFrame->data, decodedFrame->linesize, 0, height, dstData,
              dstLinesize);
```

**第 282~284 行：填 `VideoFrame` 的三个字段。**

```cpp
out.width = width;      // 注意：用的是解码出来的尺寸，不是 raw.width！
out.height = height;
out.bgra.assign(static_cast<size_t>(width) * height * 4, 0);
```

**第一个要点：尺寸来自解码结果。** 对 MJPG 来说，`raw.width/height` 只是"摄像头声称的分辨率"，而**真实分辨率以 JPEG 头里的为准**。用解码结果更可靠。**这是一个"信任数据而非信任声明"的选择。**

**第二个要点：`assign(size, 0)` 而不是 `resize(size)`。**

- `resize` 只保证大小，**新分配的元素值不确定**（对 `vector<uint8_t>` 是值初始化为 0，但已有元素保持原值）；
- `assign(n, 0)` 是"**重新填充 n 个 0**"，无论之前是什么，全部清零。

为什么这里要清零？因为 `sws_scale` 理论上会写满每一个字节，但如果某些边缘情况没写满（比如解码出的帧尺寸比预期小），残留的旧数据就会变成画面上的**随机噪点**。清零保证了"最坏情况是黑块，而不是鬼影"。

> 对比一下：`convert()` 里对非 MJPG 路径用的是 `out->bgra.resize(...)`（第 354 行），因为那里的每一行都会被四个 `convertXxxRow` 之一**完整覆盖**，不会有残留。而 MJPG 路径依赖第三方库写满，所以更保守地清零。**不同路径用不同策略，是基于"谁保证写满"的判断**——这个细节体现了作者对每一行代码的认真。

**第三个要点：`static_cast<size_t>(width) * height * 4`。**

为什么要先转 `size_t`？因为 `width * height * 4` 如果全是 `int`，1080p 是 `1920 * 1080 * 4 = 8294400`，还在 int 范围内（21 亿），但 4K 就是 `3840*2160*4 = 33177600`，也还好。**真正的风险是 `width * height` 先算**：如果 width 和 height 都很大（比如驱动谎报 100000×100000），`width*height` 会**先溢出 int**，然后再乘 4，结果完全错误——分配一块错误大小的内存，然后 swscale 写到越界位置。

转 `size_t`（64 位无符号，最大 1.8e19）之后，乘法不会溢出。**这是 C++ 数值计算的经典防御：在做可能溢出的乘法前，先把一个操作数提升到更宽的类型。**

**第 286~287 行：准备 sws_scale 的输出参数。**

```cpp
uint8_t* dstData[4] = {out.bgra.data(), nullptr, nullptr, nullptr};
int dstLinesize[4] = {width * 4, 0, 0, 0};
```

回忆 1.13 节：`data[4]` 和 `linesize[4]` 是为 planar 格式准备的（最多 4 个平面）。我们的目标 BGRA 是 **packed 格式，只有一个平面**，所以：
- `dstData[0] = out.bgra.data()`，其余 `nullptr`；
- `dstLinesize[0] = width * 4`（BGRA 每行字节数，**没有填充，紧密排列**），其余 0。

**注意：`dstLinesize[0] = width * 4` 而不是带对齐的 stride。** 这是我们的约定——`VideoFrame::bgra` 是**紧密排列**的，下游可以按 `width * 4` 直接跳行，**不需要处理 stride**。这就是"统一格式"带来的好处：**下游彻底摆脱了 stride 的概念**。第 5 讲说的"stride 是无数 bug 的根源"，在这里被终结了。

**第 288~289 行：调用 `sws_scale`。**

```cpp
sws_scale(sws, decodedFrame->data, decodedFrame->linesize, 0, height, dstData,
          dstLinesize);
```

参数：`sws`、源数据指针数组、源行跨度数组、**起始行** `0`、**行数** `height`、目标数据指针数组、目标行跨度数组。

**源参数直接用 `decodedFrame->data` 和 `decodedFrame->linesize`**——FFmpeg 解出来的帧自带正确的行跨度（可能有对齐填充），我们原样传进去，让 swscale 自己处理。**这就是"用库就要用对它的约定"：不自己推算 linesize，而是用它给的。**

#### 5.3.6 第六段：收尾（第 291~293 行）

```cpp
    av_frame_unref(decodedFrame);
    return true;
}
```

**成功路径也要 `av_frame_unref`！**

这一点让很多新手困惑："我解出来的帧，为什么要立刻 unref 掉？"

答案是：**我们已经把数据拷走了。** `sws_scale` 已经把像素写进了 `out.bgra`，`decodedFrame` 里的数据我们不再需要。unref 之后：
1. FFmpeg 内部的数据缓冲**引用计数归零**，可以被复用或释放；
2. `decodedFrame` 回到干净状态，下次调用可以直接接收新帧；
3. **内存占用峰值降低**——不 unref 的话，这一帧的数据会一直挂到下次调用（虽然下次会 unref，但期间一直占着）。

**这是 FFmpeg 编程的核心节奏：`alloc` 一次（长期持有）、每帧 `unref` 两次（packet 和 frame 各一次）、最后 `free` 一次。** 记住这个节奏，FFmpeg 的内存管理就不会出错。

#### 5.3.7 `convertMjpeg` 全景流程图

```
convertMjpeg(raw, out)
   │
   ├─ ensureDecoder()  ── 懒加载 MJPEG 解码器三件套
   │
   ├─ decode lambda（可复用两次）
   │     ├─ av_packet_unref(packet)          ← 清干净
   │     ├─ packet->data = raw.bytes.data()  ← 零拷贝，指向源数据
   │     ├─ avcodec_send_packet(...)         ← 送进解码器
   │     ├─ av_frame_unref(decodedFrame)     ← 清干净
   │     └─ avcodec_receive_frame(...)       ← 取出解码帧
   │
   ├─ 第 1 次 decode(全部字节)
   │     失败？→ 从后往前找 FF D9 → 第 2 次 decode(截断后)
   │
   ├─ 两次都失败 → lastError + return false
   │
   ├─ 校验 width/height（失败要 unref）
   │
   ├─ SwsContext 缓存判断：尺寸或格式变了？
   │     ├─ 是 → 释放旧的 → sws_getContext → 判断全范围 → setColorspaceDetails → 记缓存键
   │     └─ 否 → 直接用
   │
   ├─ out.bgra.assign(w*h*4, 0)   ← 清零分配
   │
   ├─ sws_scale(解码帧 → out.bgra)
   │
   └─ av_frame_unref(decodedFrame) ← 成功路径也要清
      return true
```

---

## 第六章：`convert()` —— 主流程与四条裸格式的处理

现在看整个类的入口（第 306~384 行）。这是唯一对外的转换函数，也是本讲的"总调度"。

### 6.1 构造、析构、错误查询（第 295~304 行）

```cpp
CameraFrameConverter::CameraFrameConverter() : impl_(std::make_unique<Impl>())
{
}

CameraFrameConverter::~CameraFrameConverter() = default;

const std::string& CameraFrameConverter::lastError() const
{
    return impl_->lastError;
}
```

**第 295~297 行：构造函数。** 只有一件事——`make_unique<Impl>()`。

用 `std::make_unique` 而不是 `new`：
1. **异常安全**：如果构造过程中抛异常，`make_unique` 保证不会泄漏；
2. **不出现裸 `new`**：代码里少一个 `new` 就少一个忘了 `delete` 的风险；
3. **简洁**：`make_unique<Impl>()` 比 `std::unique_ptr<Impl>(new Impl())` 短，也不用重复写类型。

**构造函数体是空的**——因为 `Impl` 的所有成员都有默认值初始化（`= nullptr`、`= 0`、`= -1`），不需要额外代码。这就是 PIMPL 的好处：**构造极轻，真正的初始化推迟到懒加载**。

**第 299 行：析构函数（PIMPL 的必考点）。**

```cpp
CameraFrameConverter::~CameraFrameConverter() = default;
```

这行**必须写在 .cpp 里**，原因见 1.16 节：`unique_ptr<Impl>` 的析构需要 `Impl` 是**完整类型**。在 .h 里 `Impl` 只有前向声明，`= default` 会导致编译错误："invalid application of sizeof to incomplete type"。

`= default` 的意思是"用编译器生成的默认版本"——这里就是"调用 `impl_` 的析构函数"，进而调用 `~Impl()`，进而调用 `releaseDecoder()`。**一行 `= default` 触发了整条 FFmpeg 资源释放链。**

**第 301~304 行：`lastError()`。** 返回 `impl_->lastError` 的常量引用。注意这个函数是 `const` 的，但访问 `impl_->lastError` 是允许的——因为 `const` 成员函数里，`impl_` 是 `const std::unique_ptr<Impl>`，**指针本身是 const，指向的对象不是**。所以可以修改 `Impl` 的内容（虽然这里只读）。这是 PIMPL 的一个副作用：**PIMPL 会让 `const` 的语义变弱**（const 成员函数仍能修改 Impl 里的东西）。这是已知的代价，通常可以接受。

### 6.2 第一步：清空错误信息（第 308 行）

```cpp
    impl_->lastError.clear();
```

**每次调用开头先清错误。** 这个细节很重要：它保证 `lastError()` 反映的**永远是最近一次调用**的结果。

如果不清，会发生什么？第一次调用失败（lastError = "数据不完整"），第二次调用成功——但 `lastError` 还留着上次的错误。调用方如果习惯性地在成功后也查一次 `lastError`，就会看到一条**过期的错误信息**，然后花两小时排查一个根本不存在的问题。**"陈旧的错误信息"是极难查的一类 bug。**

规则：**每次公开调用开始，先重置状态。**

### 6.3 第二步：三重入参校验（第 310~323 行）

```cpp
    if (raw.width <= 0 || raw.height <= 0) {
        impl_->lastError = "帧尺寸无效";
        return nullptr;
    }

    if (raw.format == CameraPixelFormat::Unknown) {
        impl_->lastError = "未知的像素格式，不知道怎么解读";
        return nullptr;
    }

    if (raw.bytes.empty()) {
        impl_->lastError = "帧数据为空";
        return nullptr;
    }
```

三道闸门，每道都配一条**人话错误信息**。这叫**前置校验（precondition check）**，也叫"卫语句（guard clause）"——**在函数开头把非法输入挡掉，让主逻辑不用考虑这些情况。**

逐条看：

**闸门一：尺寸必须为正（第 310~313 行）。**

`<= 0` 而不只是 `== 0`——**负数也要挡**。宽高为负在语义上无意义，而且后面 `raw.height - 1 - y` 之类的计算会出大问题。

**闸门二：格式必须已知（第 315~318 行）。**

错误信息写得很幽默："**未知的像素格式，不知道怎么解读**"。这句话信息量很高：它不仅说失败了，还说了**失败的原因是我们不认识这种格式**（而不是数据坏了）。

**闸门三：数据非空（第 320~323 行）。**

`bytes.empty()`。注意这道闸门在尺寸和格式之后——**顺序是有讲究的**：先检查"元信息"，再检查"数据"。因为如果格式是 Unknown，查数据大小没意义。

**为什么三道闸门都要单独的错误信息，而不是统一的"参数无效"？**

因为**错误信息的目标是缩短排障时间**。三种失败对应三种完全不同的原因：
- 尺寸无效 → 采集层的协商有问题（第 5 讲的格式协商失败了）；
- 格式未知 → 摄像头给了我们不认识的格式（要扩展 `CameraPixelFormat` 枚举）；
- 数据为空 → 回调没拷到数据，或者帧真的空。

每条信息都直接指向不同的排查方向。**好的错误信息是"诊断建议"，不是"状态播报"。**

### 6.4 第三步：创建输出帧并继承 pts（第 325~326 行）

```cpp
    auto out = std::make_shared<VideoFrame>();
    out->pts = raw.pts;
```

**`make_shared`**：一次分配搞定"对象 + 引用计数块"，比 `shared_ptr<VideoFrame>(new VideoFrame)` 快且异常安全。

**`out->pts = raw.pts;`：时间戳透传。**

这是很关键的一行。PTS（Presentation TimeStamp，显示时间戳）是帧的"身份"——它决定了这一帧什么时候被播放、怎么和音频对齐。**转换只改变像素的表示方式，不改变帧的时间属性。** 所以直接透传。

注意这是**在任何分支之前**做的——无论走 MJPG 路径还是裸格式路径，pts 都会被设置。这个"先做公共部分，再分支"的写法很干净：**公共逻辑只写一次**。

（顺便复习第 5 讲的遗留问题：`CameraCapturer` 给的 `pts` 只是帧序号，不是真正的时间戳。今天原样透传，所以这个问题依然留着——我们在第九章会再提一次。）

### 6.5 第四步：MJPG 分流（第 328~332 行）

```cpp
    if (raw.format == CameraPixelFormat::MJPG) {
        if (!impl_->convertMjpeg(raw, *out))
            return nullptr;
        return out;
    }
```

**这是整个 `convert()` 的分支点。** MJPG 走一条完全不同的路（解码 + swscale），其余四种走"逐行手写转换"的路。

**为什么 MJPG 要提前 return？** 因为它的输出尺寸**来自解码结果**，而不是 `raw.width/height`；它也不需要 `stride`、`flipVertical` 这些裸格式概念。**压缩格式和裸格式是两套世界观，强行统一反而复杂。**

**注意 `*out` 的用法**：`convertMjpeg` 的第二个参数是 `VideoFrame&`（引用），所以传 `*out`（解引用 shared_ptr）。这样 MJPEG 路径直接往同一个对象里填，不需要额外的拷贝或赋值。

**一个值得讨论的设计问题**：这里 MJPG 路径**忽略了 `raw.width/height` 和 `raw.flipVertical`**。如果某摄像头的 MJPG 帧是自底向上的（虽然罕见——JPEG 总是自顶向下），我们会输出倒置画面。这是一个已知的简化，放到第九章讨论。

### 6.6 第五步：算行字节数与有效 stride（第 334~340 行）

```cpp
    const int rowBytes = packedRowBytes(raw.format, raw.width);
    if (rowBytes <= 0) {
        impl_->lastError = "这种格式没有逐行转换的实现";
        return nullptr;
    }

    const int stride = raw.stride >= rowBytes ? raw.stride : rowBytes;
```

**第 334 行**：调 4.4 节那个函数，算一行的有效字节数。

**第 335~338 行：又一道防御。** 走到这里说明格式不是 MJPG（已分流）也不是 Unknown（已挡），理论上 `packedRowBytes` 必然返回正数。但仍然检查——**因为将来如果 `CameraPixelFormat` 增加了新成员（比如 NV12），而 `packedRowBytes` 忘了加 case，它会返回 0**。这道检查就是为那一天准备的。这叫**面向未来的防御**：不是为了现在，是为了将来有人改代码时不会静默出错。

**第 340 行：这行是本讲最精妙的容错之一。**

```cpp
const int stride = raw.stride >= rowBytes ? raw.stride : rowBytes;
```

意思是：**如果上游给的 `stride` 比理论最小值还小（显然不合理），就用理论最小值。**

为什么会有这种情况？回忆第 5 讲：采集层对 **MJPG** 把 `stride` 设成了 0（因为"stride 对 MJPG 没意义"）。如果某种情况下一个裸格式帧的 stride 也是 0（比如驱动没填、或者手工构造的测试帧），直接用 0 会导致：

```cpp
src = srcBase + y * 0;    // 所有行都指向第 0 行！
```

画面变成"**第一行重复 height 次**"——一条条横条纹。用 `rowBytes` 兜底就避免了这个灾难。

**这是一个"不相信上游，但不崩溃"的典范**：我们不假设上游永远正确，而是用一个安全的默认值兜住。这叫**优雅降级（graceful degradation）**。

### 6.7 第六步：缓冲区大小校验（第 342~350 行）

```cpp
    const size_t needed =
        static_cast<size_t>(stride) * static_cast<size_t>(raw.height - 1)
        + static_cast<size_t>(rowBytes);

    if (raw.bytes.size() < needed) {
        impl_->lastError = "帧数据不完整：需要 " + std::to_string(needed) + " 字节，实际只有 "
                           + std::to_string(raw.bytes.size()) + " 字节";
        return nullptr;
    }
```

**第 342~344 行：算"最少需要多少字节"。**

公式是：

```
needed = stride × (height - 1) + rowBytes
```

**为什么是 `height - 1` 而不是 `height`？**

因为**最后一行不需要 stride 的填充**——它后面没有数据了，只需要 `rowBytes` 个有效字节。这是一个非常重要的细节：

```
第 0 行:  [有效数据 rowBytes][填充 stride-rowBytes]
第 1 行:  [有效数据 rowBytes][填充 stride-rowBytes]
...
第 h-2行: [有效数据 rowBytes][填充 stride-rowBytes]
第 h-1行: [有效数据 rowBytes]                        ← 最后一行没有填充！
```

如果用 `stride × height`，就会**多要求 `stride - rowBytes` 个字节**（最多 3 字节）。在正常情况下这没问题（驱动给的缓冲区通常按 `stride × height` 分配），但在**测试构造的帧**、或者**某些抠门的驱动**上，就会误判为"数据不完整"。

**用最小需求量做校验，是"宽进严出"的正确姿势**：只要数据够我读，我就接受，不多要求。

**`static_cast<size_t>` 转两次**：同样是防溢出（见 5.3.5 节）。`stride * (height-1)` 对 4K 帧是 `3840*4 * 2159 ≈ 33 MB`，还在 int 范围内，但转 `size_t` 是零成本的保险。

**第 346~349 行：不够就报错，错误信息里带上具体数字。**

```cpp
"帧数据不完整：需要 2764800 字节，实际只有 2764799 字节"
```

**这条错误信息堪称教科书级。** 它包含了：
- 现象（数据不完整）；
- 期望值（需要多少）；
- 实际值（有多少）。

有了这三个数字，排障者**立刻**能算出差值（1 字节），进而推断"stride 算错了 1 字节"或"少拷了 1 字节"。**如果只写"数据不完整"，排障者得自己加日志、重新编译、再跑一遍。**

**这就是"错误信息要包含诊断所需的全部数据"原则。** 请大家以后写错误信息时都问自己：看到这条信息的人，还需要什么才能定位问题？

### 6.8 第七步：分配输出缓冲（第 352~354 行）

```cpp
    out->width = raw.width;
    out->height = raw.height;
    out->bgra.resize(static_cast<size_t>(raw.width) * raw.height * 4);
```

尺寸直接来自 `raw`（裸格式路径里，`raw.width/height` 就是权威）。

**`resize` 而不是 `assign`**（对比 MJPEG 路径的 `assign(..., 0)`）：因为接下来每一行都会被 `convertXxxRow` **完整覆盖**，不存在"未写满"的风险。`resize` 对 `vector<uint8_t>` 也是值初始化为 0，所以新分配的内存也是干净的。

**为什么这里不用清零？** 因为循环 `for (int y = 0; y < raw.height; ++y)` 会遍历**每一行**，而每个 `convertXxxRow` 会处理**该行的每一个像素**（`for x < width`）。所以 `width * height * 4` 个字节**每一个都会被写**。`resize` 就够了。

（注意：YUY2/UYVY 的 `convertXxxRow` 在奇数宽度时，最后一个宏像素的第 2 个像素不写——但这只影响 `dst` 多推进一次的问题，而它用 `if (x+1 < width)` 挡住了，所以不会越界，也不会漏写有效像素。✓）

### 6.9 第八步：逐行转换循环（第 356~381 行）

```cpp
    const int dstRowBytes = raw.width * 4;
    const uint8_t* srcBase = raw.bytes.data();

    for (int y = 0; y < raw.height; ++y) {
        const uint8_t* src = srcBase + static_cast<size_t>(y) * stride;

        const int dstY = raw.flipVertical ? (raw.height - 1 - y) : y;
        uint8_t* dst = out->bgra.data() + static_cast<size_t>(dstY) * dstRowBytes;

        switch (raw.format) {
        case CameraPixelFormat::RGB24:
            convertRgb24Row(src, dst, raw.width);
            break;
        case CameraPixelFormat::RGB32:
            convertRgb32Row(src, dst, raw.width);
            break;
        case CameraPixelFormat::YUY2:
            convertYuy2Row(src, dst, raw.width);
            break;
        case CameraPixelFormat::UYVY:
            convertUyvyRow(src, dst, raw.width);
            break;
        default:
            break;
        }
    }

    return out;
```

**这是整个转换器的主循环，也是"stride 剔除"和"垂直翻转"两个遗留问题的最终解决地。** 我们一行一行看：

**第 356 行：`dstRowBytes = raw.width * 4`。** 目标行跨度——**没有填充**，紧密排列。这就是 `VideoFrame` 的约定。

**第 357 行：`srcBase = raw.bytes.data()`。** 把基地址提出循环外，避免每次都通过 `raw.bytes.data()` 访问（虽然编译器通常会优化，但显式提出来更清晰）。

**第 359 行：循环 `y` 从 0 到 height-1。** 注意循环变量是**源行号**。

**第 360 行：用 stride 跳行（第 5 讲遗留问题的解决之一）。**

```cpp
const uint8_t* src = srcBase + static_cast<size_t>(y) * stride;
```

**这里是全函数最关键的一行。** 用 `stride`（含填充）而不是 `width * 像素字节数` 来定位源行。填充分字节被自动跳过。

如果写成 `y * width * 3`（假设 RGB24），而实际 stride 是 `width*3 + 3`，那么：
- 第 0 行：正确；
- 第 1 行：少跳 3 字节 → 读到的是第 0 行末尾的 3 个填充字节 + 第 1 行的前 `width*3-3` 字节；
- 第 2 行：少跳 6 字节 → 偏移更大……

**视觉结果**：每一行都比上一行多偏移一点，画面变成**斜条纹**（1.14 节讲过的 shearing artifact）。测试第 2 段就是专门造一个带填充的帧来验证这一点的。

**第 362 行：垂直翻转（第 5 讲遗留问题的解决之二）。**

```cpp
const int dstY = raw.flipVertical ? (raw.height - 1 - y) : y;
```

1.14 节讲过的"写入时变换"。源第 `y` 行 → 目标第 `height-1-y` 行。

**为什么这个写法好？** 对比"先正着转，再翻一遍"：
- 方案 A（当前）：一趟完成，O(n) 时间，零额外内存；
- 方案 B（先转后翻）：两趟，还要再遍历一次 8 MB 的缓冲。

**"把变换融合进数据搬运"是高性能图像处理的通用技巧。** 类似的还有：缩放 + 格式转换融合（swscale 就是这么做的）、裁剪 + 翻转融合等等。原则：**尽量一趟走完，减少对大缓冲的遍历次数**——因为内存带宽才是瓶颈，不是 CPU。

**第 363 行：用 dstRowBytes 定位目标行。**

```cpp
uint8_t* dst = out->bgra.data() + static_cast<size_t>(dstY) * dstRowBytes;
```

**第 365~380 行：`switch` 分发到四个转换函数。**

注意四个 case 的调用签名**完全一致**：`(src, dst, raw.width)`。这就是 4.11 节说的"统一接口"带来的好处——调用方代码极其干净。

**第 378~379 行：`default: break;`**

这一个 `default` 分支很重要。虽然理论上走不到（Unknown 和 MJPG 都已被挡），但：
1. **编译器会警告"switch 没有覆盖所有枚举值"**（`-Wswitch`），加了 `default` 消除警告；
2. 万一将来加了新格式，走到这里也只是"这一行不转换"（输出该行的黑色/旧数据），**不会崩溃**。

这是**防御性 default**——不是为了处理已知情况，是为了让未来最坏情况也是"安静地错"而不是"崩溃"。

**第 383 行：`return out;`** 返回 `shared_ptr`。

### 6.10 `convert()` 全景流程图

```
convert(raw)
  │
  ├─ lastError.clear()                 ← 重置状态
  │
  ├─ 三重校验：尺寸 > 0？格式 != Unknown？bytes 非空？
  │
  ├─ out = make_shared<VideoFrame>(); out->pts = raw.pts
  │
  ├─【分支点】format == MJPG ?
  │     ├─ 是 → convertMjpeg(raw, *out) → return out / nullptr
  │     └─ 否 ↓
  │
  ├─ rowBytes = packedRowBytes(format, width)
  │     rowBytes <= 0 ? → 报错返回
  │
  ├─ stride = max(raw.stride, rowBytes)      ← 优雅降级
  │
  ├─ needed = stride*(h-1) + rowBytes        ← 最后一行无填充
  │     bytes.size() < needed ? → 报错返回（带具体数字）
  │
  ├─ out->bgra.resize(w*h*4)
  │
  └─ for y in [0, height):
        src = srcBase + y * stride               ← stride 跳行
        dstY = flipVertical ? (h-1-y) : y        ← 写入时翻转
        dst = bgra.data() + dstY * (w*4)
        switch (format) → convertRgb24Row / Rgb32Row / Yuy2Row / UyvyRow
     return out
```

### 6.11 两种路径的横向对比

| 维度 | 裸格式路径（RGB24/32、YUY2/UYVY） | MJPG 路径 |
|---|---|---|
| 用的技术 | 手写逐行转换 | FFmpeg 解码 + swscale |
| 输出尺寸来源 | `raw.width / raw.height` | **解码结果** |
| 是否处理 stride | **是**（`y * stride` 跳行） | 否（压缩流无 stride 概念） |
| 是否处理 flipVertical | **是**（`dstY` 镜像） | **否**（已知局限） |
| 是否校验缓冲区大小 | **是**（`needed`） | 否（靠解码器自己报错） |
| 输出缓冲初始化 | `resize` | `assign(..., 0)` 清零 |
| 是否复用昂贵对象 | 否（纯计算） | **是**（解码器 + SwsContext） |
| 失败可能性 | 很低（只可能输入非法） | 较高（解码、内存、版本兼容） |

---

## 第七章：测试程序教学 —— `tests/test_camera_convert.cpp`

同学们，我一直强调一句话：**测试即教案**。一份好的测试程序，是最好的使用文档——它告诉你"这个模块该怎么用、边界在哪、出错时是什么样"。今天这份 600 行的测试尤其出色，我们用一整章来读它。

### 7.0 测试工具的准备（第 1~60 行）

#### 7.0.1 include 区（第 1~17 行）

```cpp
#include "core/capture/CameraCapturer.h"
#include "core/capture/CameraFrameConverter.h"

extern "C" {
#include <libavcodec/avcodec.h>
#include <libavutil/frame.h>
}

#include <chrono>
#include <cmath>
#include <cstdint>
#include <cstdio>
#include <cstdlib>
#include <cstring>
#include <string>
#include <vector>
```

注意测试程序**自己**也 include 了 FFmpeg 头文件（因为要用 MJPEG **编码器**现场造测试素材）。这印证了 2.2 节说的：测试程序靠全局的 include 路径找到 FFmpeg。

`<cmath>` 是为了 `std::abs`（像素容差比较用）；`<chrono>` 是为了计时（性能测试用）；`<cstdlib>` 是为了 `std::atoi` 和 `std::system`；`<cstring>` 是为了 `std::memset`（填 YUV 用）。

#### 7.0.2 计数器与 `check` 函数（第 20~35 行）

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

一个极简的测试框架：**两个全局计数器 + 一个 `check` 函数**。

**为什么要有 `std::string` 的重载？** 因为很多断言的描述需要**拼接运行时数据**，比如：

```cpp
check(perFrame < 20.0, "单帧转换耗时低于 20 毫秒，够跑 30fps");
```

如果要带上实际值，就必须是 `std::string`。有了重载，调用方不用写 `.c_str()`。**这是"为调用方着想"的 API 设计。**

**两个全局变量的命名：`g_` 前缀**表示 global。在测试程序里用全局变量是可接受的（单线程、短生命周期），但在产品代码里要避免。

#### 7.0.3 计时助手（第 39~44 行）

```cpp
long long elapsedMs(std::chrono::steady_clock::time_point from)
{
    return std::chrono::duration_cast<std::chrono::milliseconds>(
               std::chrono::steady_clock::now() - from)
        .count();
}
```

**为什么用 `steady_clock` 而不是 `system_clock`？** 这是 C++ 计时的必备知识：

- **`system_clock`**：系统墙上时钟，**会被调整**（NTP 校时、用户改时间、夏令时）。用它测间隔，可能测出负数！
- **`steady_clock`**：单调时钟，**只增不减**，不受任何调整影响。**测时间间隔必须用它。**

`duration_cast<std::chrono::milliseconds>`：把纳秒精度的差值转成毫秒（向下取整）。

#### 7.0.4 `makeFrame`：测试素材工厂（第 61~71 行）

```cpp
CameraRawFrame makeFrame(int width, int height, CameraPixelFormat format,
                         int stride = 0)
{
    CameraRawFrame frame;
    frame.width = width;
    frame.height = height;
    frame.format = format;
    frame.stride = stride > 0 ? stride : packedRowBytes(format, width);
    frame.bytes.assign(static_cast<size_t>(frame.stride) * height, 0);
    return frame;
}
```

**这个工厂函数是整个测试能"确定性"运行的基础。**

关键设计：
- **`stride` 默认 0** → 自动用 `packedRowBytes` 算出**无填充**的行跨度。大多数测试不需要填充，默认不给。
- **`stride > 0` 时用传入值** → 第 2 段测试专门传 12 来造"带填充"的帧。
- **`bytes.assign(stride * height, 0)`** → 全部清零。**清零保证了"没写过的像素是黑色"，让测试的期望值完全确定。**

**为什么测试要自己维护一份 `packedRowBytes`（第 46~59 行）？** 因为它在 `CameraFrameConverter.cpp` 的匿名命名空间里，**对外部不可见**（4.2 讲的内部链接）。测试只能抄一份。这里有个小小的**重复代码**问题，可以讨论（见第九章）。但优点也很明显：**测试用的是"独立算出来的期望值"，而不是"调用被测代码的同一个函数"**——如果用同一个函数，就变成了"自己验证自己"，永远会通过。**测试要独立推导期望值，这是测试有效性的一条基本原则。**

#### 7.0.5 四个 `setXxx` 造像素函数（第 73~112 行）

```cpp
void setRgb24(CameraRawFrame& frame, int x, int y, uint8_t r, uint8_t g, uint8_t b)
{
    uint8_t* p = frame.bytes.data() + static_cast<size_t>(y) * frame.stride
                 + static_cast<size_t>(x) * 3;
    p[0] = b;
    p[1] = g;
    p[2] = r;
}
```

注意参数顺序是 `(r, g, b)`（人类习惯），但**写入时 `p[0] = b`**——第 0 个字节是 B。这就是 1.3 节讲的"BGR 内存序"的直接体现。

**这个"参数用 RGB、存储用 BGR"的错位，正是最容易写错的地方。** 测试里刻意保留这个错位，就是为了钉死它：**如果有人把转换器的 `dst[0] = src[0]` 写错成 `dst[2] = src[0]`，这个测试立刻会红。**

```cpp
void setRgb32(CameraRawFrame& frame, int x, int y, uint8_t r, uint8_t g, uint8_t b)
{
    uint8_t* p = frame.bytes.data() + static_cast<size_t>(y) * frame.stride
                 + static_cast<size_t>(x) * 4;
    p[0] = b;
    p[1] = g;
    p[2] = r;
    p[3] = 0x00;      // ← 重点：故意写 0
}
```

**`p[3] = 0x00` 这一行是精心设计的陷阱。** 它把 RGB32 的第 4 个字节设成"完全透明"，用来验证转换器**一定会把它覆盖成 255**。如果转换器照抄了这个字节，测试就会失败。

```cpp
void setYuy2Pair(CameraRawFrame& frame, int x, int y, uint8_t y0, uint8_t y1,
                 uint8_t u, uint8_t v)
{
    uint8_t* p = frame.bytes.data() + static_cast<size_t>(y) * frame.stride
                 + static_cast<size_t>(x / 2) * 4;
    p[0] = y0;
    p[1] = u;
    p[2] = y1;
    p[3] = v;
}
```

注意 `x / 2 * 4`——**先除 2 找宏像素编号，再乘 4 找字节偏移**。这印证了 1.7 节的布局。

函数名里的 **"Pair"** 是个好命名——它提醒调用者"**一次设置两个像素**"。

```cpp
void setUyvyPair(CameraRawFrame& frame, int x, int y, uint8_t y0, uint8_t y1,
                 uint8_t u, uint8_t v)
{
    uint8_t* p = frame.bytes.data() + static_cast<size_t>(y) * frame.stride
                 + static_cast<size_t>(x / 2) * 4;
    p[0] = u;
    p[1] = y0;
    p[2] = v;
    p[3] = y1;
}
```

和 YUY2 版**只有赋值顺序不同**——这正是 1.7 节讲的"只差字节顺序"。

#### 7.0.6 `pixelIs`：带容差的像素断言（第 114~124 行）

```cpp
bool pixelIs(const VideoFrame& frame, int x, int y, int b, int g, int r,
             int tolerance = 0)
{
    if (x < 0 || y < 0 || x >= frame.width || y >= frame.height)
        return false;
    const uint8_t* p =
        frame.bgra.data() + (static_cast<size_t>(y) * frame.width + x) * 4;
    return std::abs(static_cast<int>(p[0]) - b) <= tolerance
           && std::abs(static_cast<int>(p[1]) - g) <= tolerance
           && std::abs(static_cast<int>(p[2]) - r) <= tolerance && p[3] == 255;
}
```

**三个设计要点：**

**要点一：参数顺序是 `(b, g, r)`——BGRA 内存序，与 `setRgb24` 的 `(r, g, b)` 相反。**

这个"反向"是**故意的**：`setXxx` 站在"源头数据"的角度（人类说 RGB），`pixelIs` 站在"内存字节"的角度（BGRA）。两边各用各的术语，中间由转换器负责翻译。**如果两边都用同一个顺序，反而掩盖了"顺序会变"这件事。**

**要点二：`tolerance`（容差）参数——这是色彩测试的关键。**

为什么需要容差？因为 **YUV → RGB → YUV 是有损的往返**：8 位量化、整数近似（1.8 节的 `>> 8`）、色度下采样，都会引入误差。测试期望"纯红"，实际可能得到 `(1, 0, 254)`。**严格的 `==` 会让测试变成"脆弱测试（brittle test）"**——在某些编译器/平台上通过，在另一些上失败。

用容差 `2` 表示"允许 ±2 的色阶偏差"——**这个量级人眼完全看不出来，但足以吸收计算误差**。这是图像测试的行业标准做法。

对比：
- RGB24/RGB32 测试用 `tolerance = 0`（默认）——因为纯字节搬运，**必须精确**；
- YUY2/UYVY 测试用 `tolerance = 2`——因为有整数运算误差；
- MJPEG 测试用 `tolerance = 16`——因为 **JPEG 是有损压缩**，误差更大。

**容差的大小，反映了各环节的误差量级。读懂这个，就读懂了整个管线的精度特性。**

**要点三：`&& p[3] == 255`——顺手验证 Alpha。** 每个像素断言都顺便检查不透明，等于免费多了一条测试。

**要点四：越界返回 `false`。** 而不是崩溃或断言。这让测试能优雅地报告"失败"而不是"程序挂了"。

#### 7.0.7 `pixelText`：失败时的诊断输出（第 126~135 行）

```cpp
std::string pixelText(const VideoFrame& frame, int x, int y)
{
    if (x < 0 || y < 0 || x >= frame.width || y >= frame.height)
        return "<越界>";
    const uint8_t* p =
        frame.bgra.data() + (static_cast<size_t>(y) * frame.width + x) * 4;
    char buffer[64] = {};
    std::snprintf(buffer, sizeof(buffer), "(B=%d G=%d R=%d A=%d)", p[0], p[1], p[2],
                  p[3]);
    return buffer;
}
```

**这个函数体现了"测试要可诊断"的原则。**

`check` 只告诉你"失败了"，但**不告诉你实际值是什么**。如果你想重跑一遍加打印，那就违背了"测试应该一次跑出问题"的初衷。`pixelText` 让每个断言都能**顺手打印实际值**：

```cpp
std::printf("  实际值: %s\n", pixelText(*out, 0, 0).c_str());
```

于是失败输出长这样：

```
  [失败] (0,0) 是红色
  实际值: (B=255 G=0 R=0 A=255)
```

一眼看出：R 和 B 对调了。

**用 `snprintf` 而不是 `sprintf`**：`snprintf` 接受缓冲区大小，**不会溢出**。`sprintf` 是 C 语言最经典的安全漏洞来源之一。测试代码也要写安全。

**`char buffer[64] = {};`**：清零，保证字符串终止。

#### 7.0.8 BMP 保存：把结果变成"人能看的证据"（第 138~202 行）

```cpp
#pragma pack(push, 1)
struct BmpFileHeader {
    uint16_t type;
    uint32_t size;
    uint16_t reserved1;
    uint16_t reserved2;
    uint32_t offBits;
};

struct BmpInfoHeader {
    uint32_t size;
    int32_t width;
    int32_t height;
    uint16_t planes;
    uint16_t bitCount;
    uint32_t compression;
    uint32_t sizeImage;
    int32_t xPelsPerMeter;
    int32_t yPelsPerMeter;
    uint32_t clrUsed;
    uint32_t clrImportant;
};
#pragma pack(pop)
```

**`#pragma pack(push, 1)` 是这段代码的核心，必须讲清楚。**

**背景**：C/C++ 的结构体成员之间会有**对齐填充（alignment padding）**——编译器为了让每个成员落在"自然对齐"的地址上（4 字节的 `uint32_t` 要放在 4 的倍数地址），会在成员之间插入空隙。

我们算一下不打包时 `BmpFileHeader` 有多大：

```
偏移 0: uint16_t type       (2 字节)  → 占 [0,1]
偏移 2: (填充 2 字节)                → 为了让 size 对齐到 4
偏移 4: uint32_t size       (4 字节)  → 占 [4,7]
偏移 8: uint16_t reserved1  (2 字节)  → 占 [8,9]
偏移 10: uint16_t reserved2 (2 字节)  → 占 [10,11]
偏移 12: uint32_t offBits   (4 字节)  → 占 [12,15]
总大小: 16 字节
```

而 **BMP 文件格式规定文件头就是 14 字节**！如果用 16 字节的结构体直接 `fwrite`，写出来的文件头就多了 2 字节垃圾，**所有图片查看器都打不开**。

`#pragma pack(push, 1)` 的意思是"**从现在开始，结构体成员按 1 字节对齐（不插填充）**"，`#pragma pack(pop)` 恢复原来的设置。`push/pop` 成对使用，是**保存/恢复**的语义，不会影响后面的代码。

加上之后：

```
偏移 0: uint16_t type       → [0,1]
偏移 2: uint32_t size       → [2,5]
偏移 6: uint16_t reserved1  → [6,7]
偏移 8: uint16_t reserved2  → [8,9]
偏移 10: uint32_t offBits   → [10,13]
总大小: 14 字节 ✓
```

`BmpInfoHeader` 同理，打包后正好 40 字节（BMP 规范要求）。

> **这是二进制文件格式处理的必备知识**：任何"直接把结构体写进文件"的代码，都必须考虑对齐。可移植的替代方案是逐字段序列化，但 `#pragma pack` 在 Windows/MSVC/GCC 上都支持，是最简洁的做法。

**`saveBmp` 函数（第 162~202 行）：**

```cpp
bool saveBmp(const std::string& path, const VideoFrame& frame)
{
    if (frame.width <= 0 || frame.height <= 0
        || frame.bgra.size()
               < static_cast<size_t>(frame.width) * frame.height * 4)
        return false;

    const uint32_t imageBytes =
        static_cast<uint32_t>(frame.width) * frame.height * 4;

    BmpFileHeader fileHeader = {};
    fileHeader.type = 0x4D42;
    fileHeader.size = sizeof(BmpFileHeader) + sizeof(BmpInfoHeader) + imageBytes;
    fileHeader.offBits = sizeof(BmpFileHeader) + sizeof(BmpInfoHeader);

    BmpInfoHeader infoHeader = {};
    infoHeader.size = sizeof(BmpInfoHeader);
    infoHeader.width = frame.width;
    infoHeader.height = frame.height;
    infoHeader.planes = 1;
    infoHeader.bitCount = 32;
    infoHeader.compression = 0;
    infoHeader.sizeImage = imageBytes;
    infoHeader.xPelsPerMeter = 2835;
    infoHeader.yPelsPerMeter = 2835;

    std::FILE* file = std::fopen(path.c_str(), "wb");
    if (!file)
        return false;

    std::fwrite(&fileHeader, sizeof(fileHeader), 1, file);
    std::fwrite(&infoHeader, sizeof(infoHeader), 1, file);

    const size_t stride = static_cast<size_t>(frame.width) * 4;
    for (int y = frame.height - 1; y >= 0; --y) {
        std::fwrite(frame.bgra.data() + static_cast<size_t>(y) * stride, stride, 1,
                    file);
    }

    std::fclose(file);
    return true;
}
```

**第 164~168 行：入参校验。** 帧必须有效、`bgra` 必须够大。

**第 171 行：`fileHeader.type = 0x4D42;`** —— 这是 ASCII 码 `B`（0x42）和 `M`（0x4D）的**小端序排列**：内存里第一个字节是 `0x42`（'B'），第二个是 `0x4D`（'M'），读出来正好是 "BM"。BMP 文件的魔数（magic number，标识文件类型的固定字节）。

> 用 `0x4D42` 而不是 `'BM'` 是因为：多字节字符常量 `'BM'` 的值是 `0x424D`（M 在高位），在小端机器上存进 `uint16_t` 后内存序是 `4D 42`——**正好反了**！所以必须写 `0x4D42`。这是小端序编程的经典陷阱，请大家记住。

**第 183 行：`infoHeader.height = frame.height;`（正数！）**

BMP 规范：`biHeight` 为**正数**表示"自底向上"存储（第一个像素行是图像的**底部**）；为负数表示"自顶向下"。

我们的 `bgra` 是**自顶向下**的（第 0 行是画面顶部）。所以要写成自底向上的 BMP，必须**倒序写行**——这就是第 194~197 行：

```cpp
for (int y = frame.height - 1; y >= 0; --y) {
    std::fwrite(frame.bgra.data() + static_cast<size_t>(y) * stride, stride, 1, file);
}
```

从最后一行（画面底部）写到第 0 行（画面顶部）。于是文件里第一行是画面底部，查看器按"自底向上"解读，正好还原成正确方向。✓

**这个"倒着写"是本章最容易让人困惑的地方，请大家在草稿纸上画一遍。**

**第 188~189 行：`xPelsPerMeter = 2835`。** 2835 像素/米 ≈ 72 DPI（1 英寸 = 0.0254 米，2835 × 0.0254 ≈ 72）。这是屏幕显示的标准 DPI。BMP 里这个字段通常被忽略，但填上是个好习惯。

**`"wb"` 模式**：`w` = 写，`b` = **二进制模式**。在 Windows 上必须加 `b`，否则 C 运行时会把 `\n`（0x0A）自动转成 `\r\n`（0x0D 0x0A）——**图像数据里随便一个 0x0A 字节都会被插入一个 0x0D，文件彻底损坏**。这是 Windows 上写二进制文件的经典陷阱。

**为什么测试要保存 BMP？** 因为**自动化断言只能验证"数字对不对"，验证不了"画面好不好看"**。BMP 让人能用眼睛确认：方向正不正、颜色对不对、有没有斜纹。测试程序第 9 段末尾直接打印了三条人工检查清单：

```cpp
std::printf("\n  请打开这两个 bmp 确认三件事：\n");
std::printf("    1. 画面正不正 —— 上下颠倒说明 flipVertical 没生效\n");
std::printf("    2. 颜色对不对 —— 人脸发蓝/天空发橙说明 U/V 弄反了\n");
std::printf("    3. 有没有斜纹 —— 有就是 stride 处理错了\n");
```

**这就是"自动断言 + 人工确认"的混合测试策略**：能自动化的自动化，不能自动化的给出清晰的检查清单。**好的测试不假装自己能验证一切。**

#### 7.0.9 `encodeTestJpeg`：现场编一张 JPEG 当素材（第 205~255 行）

```cpp
std::vector<uint8_t> encodeTestJpeg(int width, int height, int y, int u, int v)
{
    const AVCodec* codec = avcodec_find_encoder(AV_CODEC_ID_MJPEG);
    if (!codec)
        return {};

    AVCodecContext* ctx = avcodec_alloc_context3(codec);
    if (!ctx)
        return {};

    ctx->width = width;
    ctx->height = height;
    ctx->pix_fmt = AV_PIX_FMT_YUV420P;
    ctx->color_range = AVCOL_RANGE_JPEG;
    ctx->time_base = AVRational{1, 25};

    std::vector<uint8_t> result;

    if (avcodec_open2(ctx, codec, nullptr) == 0) {
        AVFrame* frame = av_frame_alloc();
        AVPacket* packet = av_packet_alloc();

        if (frame && packet) {
            frame->format = ctx->pix_fmt;
            frame->width = width;
            frame->height = height;

            if (av_frame_get_buffer(frame, 32) == 0) {
                for (int row = 0; row < height; ++row)
                    std::memset(frame->data[0] + row * frame->linesize[0], y, width);
                for (int row = 0; row < height / 2; ++row) {
                    std::memset(frame->data[1] + row * frame->linesize[1], u, width / 2);
                    std::memset(frame->data[2] + row * frame->linesize[2], v, width / 2);
                }

                if (avcodec_send_frame(ctx, frame) == 0
                    && avcodec_receive_packet(ctx, packet) == 0) {
                    result.assign(packet->data, packet->data + packet->size);
                }
            }
        }

        if (packet)
            av_packet_free(&packet);
        if (frame)
            av_frame_free(&frame);
    }

    avcodec_free_context(&ctx);
    return result;
}
```

**这个函数解决了一个很关键的测试问题：MJPEG 的输入素材从哪来？**

三种选择：

| 方案 | 优点 | 缺点 |
|---|---|---|
| 硬编码一段 JPEG 字节数组 | 零依赖 | 600 行的二进制 blob，看不懂、改不了 |
| 依赖真实摄像头 | 真实 | **没有摄像头就测不了**，且结果不确定 |
| **现场用编码器生成** | 结果**完全确定**、参数可调、无外部依赖 | 依赖 FFmpeg 的 MJPEG **编码器**（可能没编译） |

作者选了第三种，并且**优雅地处理了它的缺点**：

```cpp
if (jpeg.empty()) {
    std::printf("  这版 FFmpeg 没带 MJPEG 编码器，本节跳过（不是解码失败）\n");
}
```

**注意括号里那句"（不是解码失败）"**——这是极其贴心的一笔。它预先堵住了"测试跳过了，是不是有问题？"的疑虑。**好的测试输出要能自我解释。**

**技术细节：**

**第 210~214 行：编码器上下文参数。** 注意这是**编码**器（`find_encoder`），和转换器里的解码器是相反方向。

- `pix_fmt = AV_PIX_FMT_YUV420P`：4:2:0 平面格式。
- `color_range = AVCOL_RANGE_JPEG`：**全范围**。这很关键——JPEG 标准用全范围，而且它和测试里的 YUV 值配套。
- `time_base = AVRational{1, 25}`：时间基，25 fps。对单帧编码无实际影响，但 FFmpeg 要求设置。

**第 220 行：`av_frame_get_buffer(frame, 32)`** —— 给 frame 分配数据缓冲，第二个参数 32 是**对齐要求**（32 字节对齐，便于 SIMD）。**这是必须的一步**：`av_frame_alloc` 只分配结构体本身，不分配像素数据。

**第 222~228 行：填充 YUV 数据（这里是 planar 布局的教科书演示）。**

```cpp
for (int row = 0; row < height; ++row)
    std::memset(frame->data[0] + row * frame->linesize[0], y, width);     // Y 平面
for (int row = 0; row < height / 2; ++row) {
    std::memset(frame->data[1] + row * frame->linesize[1], u, width / 2); // U 平面
    std::memset(frame->data[2] + row * frame->linesize[2], v, width / 2); // V 平面
}
```

- **Y 平面**：`height` 行，每行 `width` 字节；
- **U/V 平面**：`height/2` 行，每行 `width/2` 字节（4:2:0，两个方向都减半）。

这就是 1.6 节 planar 布局的实证。

**注意用 `frame->linesize[i]` 跳行，而不是 `width`**——因为 `av_frame_get_buffer` 分配时用了 32 字节对齐，**linesize 可能大于 width**。这是 FFmpeg 编程的又一条铁律：**永远用 frame 自己给的 linesize。**

**第 230~233 行：编码（send/receive 模型，方向反过来）。**

```cpp
avcodec_send_frame(ctx, frame)      // 送原始帧
avcodec_receive_packet(ctx, packet) // 取压缩包
```

和解码的 `send_packet / receive_frame` 完全对称。**记住这个对称性，FFmpeg 编解码就掌握了一半。**

**第 235~236 行：把结果拷进 `std::vector`。**

```cpp
result.assign(packet->data, packet->data + packet->size);
```

必须拷贝——因为 `packet` 马上要被 `av_packet_free` 释放。**C 库的缓冲进了 C++ 容器才安全。**

**第 240~243 行：清理。** 注意 `if (packet)` / `if (frame)` 的空检查（可能分配失败）。

**第 246 行：`avcodec_free_context(&ctx);`** 无论如何都要释放——它写在 `if (avcodec_open2(...) == 0)` 的**外面**，即使打开失败也要释放（因为 `alloc_context3` 成功了）。**这个位置是对的**，请同学们注意对比 5.2.5 节"失败路径清理"的讨论。

### 7.1 主函数开头：三件事（第 259~282 行）

```cpp
int main(int argc, char** argv)
{
#if defined(_WIN32)
    std::system("chcp 65001 > nul");
#endif

    static char stdoutBuffer[8192];
    std::setvbuf(stdout, stdoutBuffer, _IOLBF, sizeof(stdoutBuffer));

    int deviceIndex = 0;
    if (argc > 1) {
        deviceIndex = std::atoi(argv[1]);
        if (deviceIndex < 0)
            deviceIndex = 0;
    }
```

**第一件事：`chcp 65001`（第 261~263 行）。**

`chcp` 是 Windows 的"改变控制台代码页"命令，`65001` 是 **UTF-8**。因为我们的 printf 输出里有中文，而 Windows 控制台默认是 GBK（936）或 GB2312，直接输出 UTF-8 会是乱码。

`> nul` 把 chcp 自己的输出（"Active code page: 65001"）丢掉。

`#if defined(_WIN32)` 保证在 Linux/Mac 上不执行（那些系统默认就是 UTF-8）。

**第二件事：`setvbuf` 设行缓冲（第 265~266 行）。**

```cpp
static char stdoutBuffer[8192];
std::setvbuf(stdout, stdoutBuffer, _IOLBF, sizeof(stdoutBuffer));
```

`_IOLBF` = **行缓冲（line buffered）**：遇到换行就刷新。默认是全缓冲（攒满 4KB 才输出），在测试程序里如果崩溃了，最后的日志就丢了；行缓冲保证每一行都及时出现。

**`static` 关键字是必须的！** 因为 `setvbuf` 只是"借用"这块缓冲，**不拷贝**。如果它是局部自动变量，函数返回后缓冲失效，stdout 就会往已失效的内存里写——**未定义行为**。`static` 让它活到程序结束。

**第三件事：命令行参数 `deviceIndex`（第 268~273 行）。**

允许 `test_camera_convert.exe 1` 指定第 1 个摄像头。`std::atoi` 解析，负数兜底成 0。**这是第 5 讲测试程序的延续设计**，保持体验一致。

### 7.2 第 1 段测试：RGB24 的颜色顺序（第 284~303 行）

```cpp
std::printf("\n--- 1. RGB24 的颜色顺序 ---\n");
std::printf("  RGB24 在内存里是 B,G,R。如果 B 和 R 写反了，\n");
std::printf("  这里第一个像素就会从\"红\"变成\"蓝\"。\n");
{
    CameraRawFrame raw = makeFrame(2, 2, CameraPixelFormat::RGB24);
    setRgb24(raw, 0, 0, 255, 0, 0);      // 红
    setRgb24(raw, 1, 0, 0, 255, 0);      // 绿
    setRgb24(raw, 0, 1, 0, 0, 255);      // 蓝
    setRgb24(raw, 1, 1, 255, 255, 255);  // 白

    VideoFramePtr out = converter.convert(raw);
    check(out != nullptr, "转换成功");
    if (out) {
        check(out->width == 2 && out->height == 2, "输出尺寸正确");
        check(out->bgra.size() == 2 * 2 * 4, "bgra 大小等于 width*height*4");
        check(pixelIs(*out, 0, 0, 0, 0, 255), "(0,0) 是红色");
        check(pixelIs(*out, 1, 0, 0, 255, 0), "(1,0) 是绿色");
        check(pixelIs(*out, 0, 1, 255, 0, 0), "(0,1) 是蓝色");
        check(pixelIs(*out, 1, 1, 255, 255, 255), "(1,1) 是白色");
        if (!pixelIs(*out, 0, 0, 0, 0, 255)) {
            std::printf("  实际值: %s\n", pixelText(*out, 0, 0).c_str());
        }
    }
}
```

**为什么用 2×2 这么小的帧？** 因为**测试要最小化**。2×2 有 4 个像素，正好放下红绿蓝白四色——**足够验证所有情况，且肉眼可数、手算可验**。这是单元测试的黄金法则：**用能暴露问题的最小输入**。如果帧是 1920×1080，出错时你根本看不出哪里错了。

**四个颜色的选择也有讲究：**
- **红**（R=255, G=0, B=0）：验证 R 通道；
- **绿**（R=0, G=255, B=0）：验证 G 通道；
- **蓝**（R=0, G=0, B=255）：验证 B 通道；
- **白**（全 255）：验证"全通道"和"最大值"边界。

**如果 B 和 R 写反**，红色像素会变成 `(B=255, G=0, R=0)`，`pixelIs(*out, 0, 0, 0, 0, 255)` 就是 false（它期望 B=0）。**测试立刻红，且 `pixelText` 会打印实际值让你一眼看穿。**

**注意第 302~304 行的条件打印**：只有失败才打印实际值，避免输出噪音。**这是"安静的成功，详细的失败"原则。**

**`out->bgra.size() == 2 * 2 * 4`** 这条断言钉死了 `VideoFrame` 的契约：**紧密排列，大小严格等于 `w*h*4`**。这是下游能"不用管 stride"的前提，必须测。

### 7.3 第 2 段测试：stride 带填充（第 306~350 行）

```cpp
std::printf("\n--- 2. 行跨度带填充 ---\n");
std::printf("  3 像素宽，每行只需 9 字节，却按 12 字节存（补了 3 字节）。\n");
std::printf("  如果转换器按 9 字节跳行，从第 2 行开始就会读错位置。\n");
{
    constexpr int kWidth = 3;
    constexpr int kHeight = 3;
    constexpr int kStride = 12;

    CameraRawFrame raw = makeFrame(kWidth, kHeight, CameraPixelFormat::RGB24,
                                   kStride);

    for (int y = 0; y < kHeight; ++y) {
        for (int x = 0; x < kWidth; ++x) {
            if (y == 0)
                setRgb24(raw, x, y, 255, 0, 0);       // 第 0 行全红
            else if (y == 1)
                setRgb24(raw, x, y, 0, 255, 0);       // 第 1 行全绿
            else
                setRgb24(raw, x, y, 0, 0, 255);       // 第 2 行全蓝
        }
        uint8_t* padding =
            raw.bytes.data() + static_cast<size_t>(y) * kStride + kWidth * 3;
        padding[0] = 0xAA;
        padding[1] = 0xBB;
        padding[2] = 0xCC;
    }

    VideoFramePtr out = converter.convert(raw);
    check(out != nullptr, "带填充的帧转换成功");

    bool allGood = out != nullptr;
    for (int x = 0; x < kWidth && out; ++x) {
        if (!pixelIs(*out, x, 0, 0, 0, 255))
            allGood = false;
        if (!pixelIs(*out, x, 1, 0, 255, 0))
            allGood = false;
        if (!pixelIs(*out, x, 2, 255, 0, 0))
            allGood = false;
    }
    check(allGood, "三行颜色各自正确，填充字节被正确跳过");

    if (out) {
        std::printf("  第 2 行第 0 个像素: %s（应为 B=0 G=255 R=0）\n",
                    pixelText(*out, 0, 1).c_str());
    }
}
```

**这一段是整个测试程序里最有诊断价值的。**

**参数设计**：`width=3`（RGB24 → 有效 9 字节），`stride=12`（补 3 字节）。这是**最小能产生填充的组合**——宽 3 是能被 4 整除的边界外的最小值。

**填充字节为什么要填 `0xAA 0xBB 0xCC`（第 325~330 行）？**

这是**本段测试最精妙的一笔**。如果留着默认的 0，那即使转换器**错误地**读了填充字节，读到的也是 0——可能碰巧"看起来没问题"。填上醒目的 `0xAA/0xBB/0xCC` 之后：

- 如果转换器按 12 跳行 ✓ → 填充被跳过，输出正确；
- 如果转换器按 9 跳行 ✗ → 第 1 行的开头会读到第 0 行末尾的 `0xAA 0xBB 0xCC` → 输出**一个诡异的灰紫色像素**，测试必红。

**这叫"毒化填充（poison padding）"**：**故意把不该被读到的内存填上有特征的值，让错误立刻显形。** 这是内存调试里最重要的技巧之一，Debug 模式下堆管理器填 `0xCDCDCDCD`、栈填 `0xCCCCCCCC` 都是同一个思路。

**验证一下我们的 `needed` 公式**：`stride*(h-1) + rowBytes = 12*2 + 9 = 33`。而 `makeFrame` 分配了 `12*3 = 36` 字节。`36 >= 33` ✓ 通过。

**注意测试打印的那行**（第 343~345 行）：无条件打印第 1 行第 0 个像素的值。因为这一段即使断言通过，看看实际值也能增强信心——**成功的测试也可以给信息，只要不啰嗦。**

### 7.4 第 3 段测试：垂直翻转（第 353~375 行）

```cpp
std::printf("\n--- 3. 自底向上 -> 翻转 ---\n");
std::printf("  源数据第 0 行画红、第 1 行画蓝。flipVertical=false 时\n");
std::printf("  输出第 0 行应该是红；=true 时应该变成蓝。\n");
{
    CameraRawFrame raw = makeFrame(2, 2, CameraPixelFormat::RGB24);
    setRgb24(raw, 0, 0, 255, 0, 0);
    setRgb24(raw, 1, 0, 255, 0, 0);
    setRgb24(raw, 0, 1, 0, 0, 255);
    setRgb24(raw, 1, 1, 0, 0, 255);

    raw.flipVertical = false;
    VideoFramePtr straight = converter.convert(raw);
    check(straight != nullptr, "不翻转时转换成功");
    if (straight)
        check(pixelIs(*straight, 0, 0, 0, 0, 255), "不翻转：第 0 行仍是红");

    raw.flipVertical = true;
    VideoFramePtr flipped = converter.convert(raw);
    check(flipped != nullptr, "翻转时转换成功");
    if (flipped) {
        check(pixelIs(*flipped, 0, 0, 255, 0, 0), "翻转后：第 0 行变成蓝");
        check(pixelIs(*flipped, 0, 1, 0, 0, 255), "翻转后：第 1 行变成红");
    }
}
```

**这一段是第 5 讲"biHeight 正负号"遗留问题的闭环验证。**

**测试设计的巧妙之处：同一份数据，测两次。**

`raw` 只构造一次，然后改 `flipVertical` 再转一次。这保证了**唯一变量是 `flipVertical`**——两次转换的输入字节完全相同，所以任何差异都必然来自翻转逻辑。**这是"控制变量法"在测试中的应用。**

**为什么用红和蓝而不是红和绿？** 因为红蓝在视觉上反差最大，而且（更重要的）**红 = R 通道，蓝 = B 通道**——如果翻转的同时还发生了字节顺序错误，现象会混淆。用红蓝，可以区分"上下颠倒"和"红蓝对调"两种故障。

**这个测试直接对应我们在第 9 段保存 BMP 后要人工检查的第 1 条**："画面正不正"。

### 7.5 第 4 段测试：RGB32 的第 4 个字节（第 378~393 行）

```cpp
std::printf("\n--- 4. RGB32 的第 4 个字节 ---\n");
std::printf("  构造时把第 4 字节写成 0x00，输出必须是 A=255。\n");
std::printf("  否则下游看到半透明画面，会以为是自己混色算错了。\n");
{
    CameraRawFrame raw = makeFrame(1, 1, CameraPixelFormat::RGB32);
    setRgb32(raw, 0, 0, 200, 100, 50);

    VideoFramePtr out = converter.convert(raw);
    check(out != nullptr, "RGB32 转换成功");
    if (out) {
        check(pixelIs(*out, 0, 0, 50, 100, 200), "BGR 三个通道顺序正确");
        const uint8_t alpha = out->bgra[3];
        check(alpha == 255, "第 4 字节被强制成不透明");
        if (alpha != 255)
            std::printf("  实际 alpha = %d\n", alpha);
    }
}
```

**1×1 的帧——最小到不能再小。** 因为这一段只关心"一个像素的四个字节"，不需要更多。

**`setRgb32(raw, 0, 0, 200, 100, 50)`** 造的是 R=200, G=100, B=50。

**`pixelIs(*out, 0, 0, 50, 100, 200)`**：注意参数顺序变成了 `(b=50, g=100, r=200)`。**同一个颜色，写进去是 RGB，读出来是 BGR**——如果转换器的通道顺序错了，这里立刻红。

**`out->bgra[3]` 直接取第 4 个字节**验证 alpha。这是唯一一条**直接访问原始字节**的断言（其它都用 `pixelIs`），因为 `pixelIs` 已经检查了 alpha，这里单独再确认一次是因为**这是本段的测试重点，值得显式断言**。

**测试注释里那句"否则下游看到半透明画面，会以为是自己混色算错了"**，是 4.8 节讲过的"上游脏数据变成下游 bug"的最好注解。请大家记住这个教训。

### 7.6 第 5 段测试：YUY2 的宏像素（第 396~425 行）

```cpp
std::printf("\n--- 5. YUY2 的宏像素 ---\n");
std::printf("  4:2:2 里两个像素共用一组 U/V，所以它们只能同色系。\n");
std::printf("  Y=81 U=90 V=240 是 BT.601 的纯红。\n");
{
    CameraRawFrame raw = makeFrame(2, 2, CameraPixelFormat::YUY2);
    setYuy2Pair(raw, 0, 0, 81, 81, 90, 240);
    setYuy2Pair(raw, 0, 1, 41, 41, 240, 110);

    VideoFramePtr out = converter.convert(raw);
    check(out != nullptr, "YUY2 转换成功");
    if (out) {
        check(pixelIs(*out, 0, 0, 0, 0, 255, 2), "像素(0,0) 接近纯红");
        check(pixelIs(*out, 1, 0, 0, 0, 255, 2), "像素(1,0) 也是红（共用 U/V）");
        check(pixelIs(*out, 0, 1, 255, 0, 0, 2), "像素(0,1) 接近纯蓝");
        check(pixelIs(*out, 1, 1, 255, 0, 0, 2), "像素(1,1) 也是蓝");

        std::printf("  实际值 (0,0): %s\n", pixelText(*out, 0, 0).c_str());
        std::printf("  实际值 (0,1): %s\n", pixelText(*out, 0, 1).c_str());
    }

    CameraRawFrame gray = makeFrame(2, 1, CameraPixelFormat::YUY2);
    setYuy2Pair(gray, 0, 0, 128, 128, 128, 128);
    VideoFramePtr grayOut = converter.convert(gray);
    check(grayOut != nullptr, "中性灰转换成功");
    if (grayOut) {
        check(pixelIs(*grayOut, 0, 0, 130, 130, 130, 3),
              "Y=U=V=128 得到约 130 的灰（说明做了 16~235 -> 0~255 的拉伸）");
        std::printf("  实际值: %s（偏移处理后应约为 130，不是 128 也不是 149）\n",
                    pixelText(*grayOut, 0, 0).c_str());
    }
}
```

**这一段有两个独立的子测试，各测一件事。**

**子测试 A：宏像素共享色度（第 399~414 行）。**

`setYuy2Pair(raw, 0, 0, 81, 81, 90, 240)` 设置的是：像素 0 的 Y=81，像素 1 的 Y=81，共用 U=90 V=240。

**两个 Y 都给 81**，是为了让两个像素亮度相同——于是它们转换后**必然是同一个颜色**。断言 `(0,0)` 和 `(1,0)` 都是"接近纯红"就验证了"共用色度"这件事。

**如果代码错误地给两个像素用了不同的 U/V**（比如误以为每像素独立），两个像素颜色会不同，断言就红。

**子测试 B：范围拉伸（第 416~424 行）。**

这就是 1.9 节手算过的那个：`Y=U=V=128` → 期望 **130**，容差 3。

**注释里那句"不是 128 也不是 149"值得品味**：
- **128**：如果完全不做偏移（直接 `R = Y`）；
- **149**：如果用了错误的系数（`128 * 1.164 = 149`，即**忘了先减 16**）。

**这两个数字是两种典型错误的"指纹"。** 测试不仅告诉你"对了"，还告诉你"如果错了，错成的样子是什么"。**这就是可诊断的测试。**

### 7.7 第 6 段测试：UYVY 的字节顺序（第 428~440 行）

```cpp
std::printf("\n--- 6. UYVY 的字节顺序 ---\n");
std::printf("  UYVY 只是把 U/V 挪到前面。读错顺序的话红蓝会对调。\n");
{
    CameraRawFrame raw = makeFrame(2, 1, CameraPixelFormat::UYVY);
    setUyvyPair(raw, 0, 0, 81, 81, 90, 240);

    VideoFramePtr out = converter.convert(raw);
    check(out != nullptr, "UYVY 转换成功");
    if (out) {
        check(pixelIs(*out, 0, 0, 0, 0, 255, 2), "像素(0,0) 是红而不是蓝");
        check(pixelIs(*out, 1, 0, 0, 0, 255, 2), "像素(1,0) 也是红");
        std::printf("  若这里的 R 和 B 对调，说明把 UYVY 当成 YUY2 读了\n");
    }
}
```

**这一段存在的唯一目的，是防止"复制 YUY2 代码时忘了改字节顺序"。**

因为 `convertUyvyRow` 和 `convertYuy2Row` 只差四行的顺序，**最容易犯的错就是从 YUY2 复制过来忘了改**。有了这个测试，一复制错就变红。

**断言的措辞"是红而不是蓝"**，直接点出了故障现象。

**最后那句 printf**（第 437 行）是**无条件打印**的——它在测试通过时也会输出。为什么？因为它是一句**排障提示**，写给"将来某天这个测试变红的人"看的。**为未来的排障者留言，是很高的测试素养。**

### 7.8 第 7 段测试：异常输入（第 443~461 行）

```cpp
std::printf("\n--- 7. 异常输入 ---\n");
{
    CameraRawFrame shortData = makeFrame(4, 4, CameraPixelFormat::RGB24);
    shortData.bytes.resize(shortData.bytes.size() - 1);
    check(converter.convert(shortData) == nullptr, "数据不够时返回 nullptr");
    std::printf("  失败原因: %s\n", converter.lastError().c_str());
    check(!converter.lastError().empty(), "同时给出了失败原因");

    CameraRawFrame unknown = makeFrame(2, 2, CameraPixelFormat::RGB24);
    unknown.format = CameraPixelFormat::Unknown;
    check(converter.convert(unknown) == nullptr, "未知格式返回 nullptr");

    CameraRawFrame empty = makeFrame(2, 2, CameraPixelFormat::RGB24);
    empty.bytes.clear();
    check(converter.convert(empty) == nullptr, "空数据返回 nullptr");

    CameraRawFrame zeroSize = makeFrame(1, 1, CameraPixelFormat::RGB24);
    zeroSize.width = 0;
    check(converter.convert(zeroSize) == nullptr, "尺寸为 0 返回 nullptr");
}
```

**这一段是"负面测试（negative testing）"——专门验证"该失败的时候确实失败了"。**

**很多初学者只测"正常路径"，不测"异常路径"。** 但对于一个会被上游脏数据喂的模块，**异常路径的正确性比正常路径更重要**——因为正常路径用户会立刻发现，异常路径可能几个月后才暴露，那时已经是线上事故。

**四个场景，对应 `convert()` 的三道闸门 + 缓冲区校验：**

| 场景 | 构造方式 | 命中哪道闸门 |
|---|---|---|
| 数据只少 1 字节 | `resize(size - 1)` | 缓冲区大小校验（`needed`） |
| 格式 Unknown | `format = Unknown` | 闸门二 |
| 数据空 | `bytes.clear()` | 闸门三 |
| 宽度为 0 | `width = 0` | 闸门一 |

**特别注意第一个场景：`resize(size - 1)`——只少 1 个字节。**

这是精心挑选的边界值。如果少一半，任何实现都会失败；**只少 1 字节**才能精确地验证"我们的 `needed` 公式算得对不对"——它必须**恰好**要求到最后一行末尾，不多不少。这是**边界值分析（boundary value analysis）**的经典应用。

**`check(!converter.lastError().empty(), ...)`** 这一条测的是"失败时要给出原因"——**不只是"失败"，而是"失败且可诊断"**。这一条断言的存在，把"错误信息"从一个软性要求变成了硬性契约。

### 7.9 第 8 段测试：MJPG 解码（第 464~507 行）

```cpp
std::printf("\n--- 8. MJPG 解码 ---\n");
std::printf("  MJPG 每帧就是一张完整 JPEG，得先解码才能谈 stride。\n");
std::printf("  这里现场编一张纯红 64x64 的 JPEG 当输入，结果完全确定。\n");
std::printf("  注意 YUV 值要用**全范围**的：JPEG 的 Y 是 0..255（Y=76 U=85 V=255），\n");
std::printf("  不是摄像头那种有限范围的 16..235（Y=81 U=90 V=240）。\n");
{
    const std::vector<uint8_t> jpeg = encodeTestJpeg(64, 64, 76, 85, 255);

    if (jpeg.empty()) {
        std::printf("  这版 FFmpeg 没带 MJPEG 编码器，本节跳过（不是解码失败）\n");
    } else {
        std::printf("  生成了 %zu 字节的 JPEG，开头是 %02X %02X（JPEG 的 SOI 标记）\n",
                    jpeg.size(), jpeg[0], jpeg[1]);

        CameraRawFrame raw;
        raw.width = 64;
        raw.height = 64;
        raw.format = CameraPixelFormat::MJPG;
        raw.stride = 0;
        raw.bytes = jpeg;

        VideoFramePtr out = converter.convert(raw);
        check(out != nullptr, "MJPG 解码成功");
        if (out) {
            check(out->width == 64 && out->height == 64, "解码出来的尺寸正确");
            check(out->bgra.size() == 64 * 64 * 4, "bgra 大小正确");
            check(pixelIs(*out, 32, 32, 0, 0, 255, 16), "中心像素接近纯红");
            std::printf("  中心像素: %s（应为 B≈0 G≈0 R≈255）\n",
                        pixelText(*out, 32, 32).c_str());
        }

        CameraRawFrame padded = raw;
        padded.bytes.resize(jpeg.size() + 4096, 0);
        check(converter.convert(padded) != nullptr,
              "帧尾补了 4096 个零字节，仍能靠 EOI 兜底解出来");

        CameraRawFrame garbage;
        garbage.width = 64;
        garbage.height = 64;
        garbage.format = CameraPixelFormat::MJPG;
        garbage.bytes.assign(1024, 0x5A);
        check(converter.convert(garbage) == nullptr, "喂垃圾数据时返回 nullptr");
        std::printf("  失败原因: %s\n", converter.lastError().c_str());
    }
}
```

**这一段有四个子测试，覆盖了 MJPEG 路径的全貌。**

**子测试 A：正常解码（第 470~489 行）。**

`encodeTestJpeg(64, 64, 76, 85, 255)`——**注意这里的 YUV 值和第 5 段完全不同**：

| | Y | U | V | 范围 | 用在 |
|---|---|---|---|---|---|
| 第 5 段（YUY2） | 81 | 90 | 240 | **有限**（16~235） | 手写 BT.601 |
| 第 8 段（MJPG） | **76** | **85** | **255** | **全**（0~255） | JPEG 编码 |

**为什么同一"纯红"有两个不同的数字组合？** 因为 1.9 节的范围差异！这正是测试注释里那两行 printf 要说的：

```
注意 YUV 值要用**全范围**的：JPEG 的 Y 是 0..255（Y=76 U=85 V=255），
不是摄像头那种有限范围的 16..235（Y=81 U=90 V=240）。
```

**这两行注释价值极高**——它把两个容易混淆的数字并列出来，防止将来的维护者在复制测试时搞混。**测试里最宝贵的往往不是代码，是解释"为什么是这个数"的注释。**

验算全范围纯红（1.9 节）：`R = 76 + 1.402*127 = 254`，`G = 76 - 0.344*(-43) - 0.714*127 = 0`，`B = 76 + 1.772*(-43) = 0`。✓ 所以期望 `(B≈0, G≈0, R≈255)`。

**容差为什么是 16？** 因为 **JPEG 是有损压缩**——DCT + 量化会引入误差，尤其是色度下采样到 4:2:0 之后，U/V 的精度损失更大。16 个色阶（约 6%）足以吸收这些误差，同时仍能捕获"红蓝对调"（差 255）或"没解码"（全 0）这类真实故障。

**`raw.stride = 0`**：对 MJPG 无意义，但显式设为 0 表达了"这里不用 stride"的意图。

**`pixelIs(*out, 32, 32, ...)` 取中心像素**：为什么不取 (0,0)？因为 JPEG 是 **8×8 分块**处理的，**边缘块**可能有边界效应（块效应，blocking artifact）。取中心最稳定。**这也是图像测试的常识：避开边缘。**

**子测试 B：EOI 兜底（第 491~494 行）。**

```cpp
CameraRawFrame padded = raw;
padded.bytes.resize(jpeg.size() + 4096, 0);
check(converter.convert(padded) != nullptr,
      "帧尾补了 4096 个零字节，仍能靠 EOI 兜底解出来");
```

这就是 1.11 节和 5.3.2 节讲的兜底逻辑的验证。**模拟了真实场景**：DirectShow 给的定长缓冲区尾部有零填充。

**为什么补 4096 而不是 1？** 因为驱动分配的缓冲通常是**按页/按 4KB 对齐**的，4096 是最真实的模拟量级。

**顺带说明**：这里有个有趣的现象——**第一次 `decode` 可能就成功了**（很多 JPEG 解码器会忽略尾部的垃圾）。那么这个测试就没有真正走到兜底分支。但没关系：**测试验证的是"最终结果正确"，而不是"走了哪条路"**。这叫**黑盒测试**——只关心行为，不关心实现。将来如果把兜底逻辑删了而解码器变了，这个测试就会红。**测试保护的是行为契约。**

**子测试 C：垃圾数据（第 496~503 行）。**

```cpp
garbage.bytes.assign(1024, 0x5A);
check(converter.convert(garbage) == nullptr, "喂垃圾数据时返回 nullptr");
```

1024 个 `0x5A`。`0x5A = 0101 1010`，一个"看起来很随机"的值。

**这个测试验证的是"解码失败时不能崩溃，且要报告错误"。** 这是**健壮性测试（robustness testing）**——模块必须能优雅地拒绝垃圾输入。

**为什么重要？** 因为真实场景里垃圾数据太常见了：USB 传输错误、摄像头插拔瞬间的半帧、驱动 bug。**一个在垃圾输入上崩溃的模块，会在最不适合的时候崩溃。**

**注意 `garbage` 没有设 `stride`**——用的是默认值 0（结构体默认值）。对 MJPG 无影响。

### 7.10 第 9 段测试：真实摄像头端到端（第 510~546 行）

```cpp
std::printf("\n--- 9. 真实摄像头端到端 ---\n");
{
    CameraCapturer capturer;
    const auto log = [](const std::string& message) {
        std::printf("  [采集器] %s\n", message.c_str());
    };

    if (!capturer.open(deviceIndex, log)) {
        std::printf("  打不开摄像头，这一节跳过（不是转换器的问题）\n");
    } else {
        std::printf("  摄像头: %d x %d, %s\n", capturer.width(),
                    capturer.height(),
                    cameraPixelFormatName(capturer.pixelFormat()));

        VideoFramePtr first;
        VideoFramePtr last;
        int failures = 0;

        const auto start = std::chrono::steady_clock::now();
        constexpr int kFrames = 10;
        for (int i = 0; i < kFrames; ++i) {
            CameraRawFramePtr raw = capturer.nextFrame(2000);
            if (!raw) {
                ++failures;
                continue;
            }
            VideoFramePtr frame = converter.convert(*raw);
            if (!frame) {
                std::printf("  转换失败: %s\n", converter.lastError().c_str());
                ++failures;
                continue;
            }
            if (!first)
                first = frame;
            last = frame;
        }
        const long long ms = elapsedMs(start);

        check(failures == 0, "10 帧全部取到并转换成功");
        if (first) {
            check(first->width == capturer.width()
                      && first->height == capturer.height(),
                  "输出尺寸与协商值一致");
            check(first->bgra.size()
                      == static_cast<size_t>(first->width) * first->height * 4,
                  "bgra 大小严格等于 width*height*4（没有 stride 残留）");

            std::printf("  10 帧总耗时 %lld 毫秒（含等待摄像头出帧）\n", ms);

            check(saveBmp("camera_converted_first.bmp", *first),
                  "首帧已保存为 camera_converted_first.bmp");
            if (last)
                check(saveBmp("camera_converted_last.bmp", *last),
                      "末帧已保存为 camera_converted_last.bmp");

            std::printf("\n  请打开这两个 bmp 确认三件事：\n");
            std::printf("    1. 画面正不正 —— 上下颠倒说明 flipVertical 没生效\n");
            std::printf("    2. 颜色对不对 —— 人脸发蓝/天空发橙说明 U/V 弄反了\n");
            std::printf("    3. 有没有斜纹 —— 有就是 stride 处理错了\n");
        }

        capturer.close();
    }
}
```

**这一段是"集成测试（integration test）"——把第 5 讲的 `CameraCapturer` 和第 6 讲的 `CameraFrameConverter` 串起来跑真家伙。**

**关键设计一：优雅跳过。**

```cpp
if (!capturer.open(deviceIndex, log)) {
    std::printf("  打不开摄像头，这一节跳过（不是转换器的问题）\n");
}
```

**括号里那句"（不是转换器的问题）"同样是神来之笔。** 没有摄像头的机器（CI 服务器、虚拟机）跑这个测试，会看到这句话，不会误以为是 bug。**测试必须能在"不完整环境"下给出清晰的结论。**

**关键设计二：前 8 段是"确定性测试"，第 9 段是"真实测试"。**

| | 前 8 段 | 第 9 段 |
|---|---|---|
| 输入 | 手工构造 | 真实摄像头 |
| 结果 | **完全确定** | 依赖硬件 |
| 能在 CI 跑吗 | **能** | 不能（没摄像头就跳过） |
| 价值 | 验证逻辑正确性 | 验证**集成正确性** |

**两者缺一不可。** 只有构造测试 → 不知道真实数据能不能跑通；只有真实测试 → CI 上跑不了，而且失败时不知道是哪里错了。**这是测试策略的经典二分。**

**关键设计三：`first` 和 `last` 两个 `shared_ptr`。**

```cpp
if (!first)
    first = frame;
last = frame;
```

只保留**首帧和末帧**，中间帧让 `shared_ptr` 自动释放。为什么？
1. **省内存**：10 帧 1080p BGRA 是 83 MB，全存着没必要；
2. **够用**：首帧和末帧**足以验证"画面在动"**（第 5 讲的测试用过同样的技巧，可以比较两帧差异）；
3. **演示 `shared_ptr` 的生命周期管理**：`last = frame` 会让上一帧的引用计数归零而自动释放。

**关键设计四：三个断言各有深意。**

```cpp
check(failures == 0, "10 帧全部取到并转换成功");
```
端到端的基本可用性。

```cpp
check(first->width == capturer.width() && first->height == capturer.height(),
      "输出尺寸与协商值一致");
```
**验证"转换器没有偷偷改尺寸"**。对裸格式路径，输出尺寸必须等于采集层协商的尺寸。

```cpp
check(first->bgra.size() == (size_t)first->width * first->height * 4,
      "bgra 大小严格等于 width*height*4（没有 stride 残留）");
```
**这一条是本讲的核心契约。** 括号里那句"（没有 stride 残留）"点出了目的：**验证 `VideoFrame` 是紧密排列的**，stride 在这个环节被彻底消除了。下游从此可以高枕无忧。

**关键设计五：两条 BMP + 三条人工检查清单。**

这是 7.0.8 节讲过的"自动断言 + 人工确认"。三条清单：

1. **画面正不正** → `flipVertical`（第 3 段自动测过，这里再肉眼确认一次真实数据）；
2. **颜色对不对** → U/V 顺序（第 5、6 段自动测过）；
3. **有没有斜纹** → stride（第 2 段自动测过）。

**为什么自动测过了还要人工看？** 因为自动测试用的是 2×2、3×3 的玩具帧，**真实分辨率下可能暴露玩具帧暴露不了的问题**（比如 1920 宽度下的对齐问题、奇数宽度、大缓冲的性能问题）。**这是"小样本测逻辑，大样本测集成"的思路。**

### 7.11 第 10 段测试：转换耗时（第 575~592 行）

```cpp
std::printf("\n--- 10. 转换耗时 ---\n");
{
    CameraRawFrame raw = makeFrame(1280, 720, CameraPixelFormat::RGB24);
    for (size_t i = 0; i < raw.bytes.size(); ++i)
        raw.bytes[i] = static_cast<uint8_t>((i * 7) & 0xFF);

    constexpr int kRounds = 30;
    const auto start = std::chrono::steady_clock::now();
    for (int i = 0; i < kRounds; ++i)
        converter.convert(raw);
    const long long ms = elapsedMs(start);

    const double perFrame = static_cast<double>(ms) / kRounds;
    std::printf("  RGB24 1280x720 转换 %d 次，共 %lld 毫秒，平均 %.2f 毫秒/帧\n",
                kRounds, ms, perFrame);
    std::printf("  按 30fps 算，一帧的预算是 33.3 毫秒\n");
    check(perFrame < 20.0, "单帧转换耗时低于 20 毫秒，够跑 30fps");
}
```

**这是"性能回归测试（performance regression test）"——最容易被忽略、但长期价值极高的一类测试。**

**为什么用 1280×720？** 这是最常见的摄像头分辨率（720p），比 1080p 更有代表性，且帧小一点跑得快。

**为什么填 `i * 7 & 0xFF`？** 造一个**有变化的数据**，避免全 0 时编译器/CPU 的某种优化（比如全 0 页的写时复制）影响计时。乘 7 是为了让字节值变化不均匀。**这是微基准测试的常见技巧：让数据"看起来真实"。**

**为什么跑 30 轮取平均？** 因为**单次测量噪声太大**（操作系统调度、CPU 频率调整、缓存冷热）。30 轮平均能把噪声压下去。**第一轮通常最慢（缓存冷），多轮平均后影响被摊薄。**

**为什么阈值是 20 毫秒而不是 33.3？**

```cpp
std::printf("  按 30fps 算，一帧的预算是 33.3 毫秒\n");
check(perFrame < 20.0, "单帧转换耗时低于 20 毫秒，够跑 30fps");
```

**这是很专业的取舍。** 30fps 的预算确实是 33.3ms，但**转换只是整个管线的一环**——后面还有编码（更贵）、推流、预览。如果转换就吃掉 33ms，整条管线必然超时。所以留了**三分之一的余量**给其它环节。

这体现了**系统级的预算思维**：不要只看"我这一环够不够快"，要看"我这一环占整条管线的合理比例是多少"。

**性能测试的局限**（要诚实说明）：
- 依赖机器性能，慢机器上可能误报失败；
- 依赖编译器优化级别（Debug 模式下可能慢 10 倍！）；
- 单线程测量，没考虑并发。

所以**性能断言的阈值要宽松**（这里是 3 倍余量），并且**失败时不要当致命错误**（这个测试失败了也算 `g_failed++`，这点其实可以改进——理想情况应该是"警告"而非"失败"）。我们在第九章讨论。

### 7.12 收尾（第 594~600 行）

```cpp
    std::printf("\n================================================\n");
    std::printf("  通过 %d 项，失败 %d 项\n", g_passed, g_failed);
    std::printf("================================================\n");
    std::fflush(stdout);

    return g_failed == 0 ? 0 : 1;
}
```

**`return g_failed == 0 ? 0 : 1;`** —— **退出码（exit code）是测试程序和 CI 的契约**。返回 0 表示成功，非 0 表示失败。有了这个，CI 系统（或 `cmake --build` 后的 `ctest`）就能自动判断测试是否通过。

**`std::fflush(stdout)`**：显式刷新。虽然程序正常退出时也会刷新，但显式写出来是为了**确保输出完整**（某些环境下 stdout 的缓冲可能没被正确 flush）。

**为什么要在意退出码？** 因为**一个测试如果不能自动化判断成败，它就只能靠人肉看输出**——而人肉看输出这件事，第三次以后就会被跳过。**退出码是"让机器替你守门"的第一步。**

### 7.13 十段测试的覆盖矩阵

| 段 | 名称 | 测什么 | 类型 | 需要硬件 |
|---|---|---|---|---|
| 1 | RGB24 颜色顺序 | BGR 通道序 | 构造 | 否 |
| 2 | stride 带填充 | 跳行正确性（毒化填充） | 构造 | 否 |
| 3 | 自底向上翻转 | `flipVertical` | 构造 | 否 |
| 4 | RGB32 第 4 字节 | Alpha 强制 255 | 构造 | 否 |
| 5 | YUY2 宏像素 | 解包 + BT.601 + 范围拉伸 | 构造 | 否 |
| 6 | UYVY 字节顺序 | 防"复制忘了改" | 构造 | 否 |
| 7 | 异常输入 | 四道闸门 | 负面 | 否 |
| 8 | MJPG 解码 | 解码 + EOI 兜底 + 垃圾输入 | 构造 | 否（需编码器） |
| 9 | 真实摄像头端到端 | 集成 + BMP 人工确认 | 集成 | **是**（可跳过） |
| 10 | 转换耗时 | 性能预算 | 性能 | 否 |

**十段覆盖四个维度：正确性、健壮性、集成性、性能。** 这是一份非常完整的测试设计，值得大家当作模板。

---

## 第八章：动手环节

讲了这么多，现在该同学们自己动手了。这一章给出完整的编译运行步骤、预期输出和六个实验。

### 8.1 编译与运行

#### 步骤一：确认 FFmpeg 可用

在配置 CMake 之前，先确认 FFmpeg 的开发包在机器上。项目应该已经配置好了 `FFMPEG_INCLUDE_DIRS` 和 `FFMPEG_LIBRARIES`（第 2 讲的内容）。可以用下面的命令确认：

```bash
# 查看顶层 CMakeLists 里 FFmpeg 的查找逻辑
grep -rn "FFMPEG" CMakeLists.txt cmake/ 2>/dev/null
```

如果之前几讲能编译出 `test_camera_capture`，说明 FFmpeg 已经就位——因为本讲没有新增任何外部依赖，只是开始使用它。

#### 步骤二：配置与编译

```bash
# 在项目根目录
cmake -S . -B build -DCMAKE_BUILD_TYPE=Release
cmake --build build --config Release
```

**注意一定要用 Release 配置！** 原因在第 10 段测试那里说过：Debug 模式下转换代码没有优化，可能慢 5~10 倍，性能测试会失败。

编译产物里会有两个新东西：
- 静态库 `core_capture.lib`（现在包含了 `CameraFrameConverter`）；
- 可执行文件 `test_camera_convert.exe`。

#### 步骤三：运行

```bash
# 用第 0 个摄像头
./build/tests/test_camera_convert.exe

# 或指定第 1 个摄像头
./build/tests/test_camera_convert.exe 1
```

**建议第一次运行时先不接摄像头**（或者干脆在没摄像头的机器上跑），这样你能看到第 9 段被跳过的样子，确认"优雅跳过"的行为。

#### 步骤四：看退出码

```bash
echo $?        # Linux/Mac/Git Bash
echo %ERRORLEVEL%   # Windows cmd
```

返回 `0` = 全通过。

### 8.2 预期输出（节选）

在"没有摄像头 + FFmpeg 带 MJPEG 编码器"的机器上，你应该看到类似这样的输出：

```
================================================
  模块 5（下）：原始字节 -> BGRA 转换测试
================================================

--- 1. RGB24 的颜色顺序 ---
  RGB24 在内存里是 B,G,R。如果 B 和 R 写反了，
  这里第一个像素就会从"红"变成"蓝"。
  [通过] 转换成功
  [通过] 输出尺寸正确
  [通过] bgra 大小等于 width*height*4
  [通过] (0,0) 是红色
  [通过] (1,0) 是绿色
  [通过] (0,1) 是蓝色
  [通过] (1,1) 是白色

--- 2. 行跨度带填充 ---
  3 像素宽，每行只需 9 字节，却按 12 字节存（补了 3 字节）。
  如果转换器按 9 字节跳行，从第 2 行开始就会读错位置。
  [通过] 带填充的帧转换成功
  [通过] 三行颜色各自正确，填充字节被正确跳过
  第 2 行第 0 个像素: (B=0 G=255 R=0 A=255)（应为 B=0 G=255 R=0）

--- 3. 自底向上 -> 翻转 ---
  源数据第 0 行画红、第 1 行画蓝。flipVertical=false 时
  输出第 0 行应该是红；=true 时应该变成蓝。
  [通过] 不翻转时转换成功
  [通过] 不翻转：第 0 行仍是红
  [通过] 翻转时转换成功
  [通过] 翻转后：第 0 行变成蓝
  [通过] 翻转后：第 1 行变成红

--- 4. RGB32 的第 4 个字节 ---
  构造时把第 4 字节写成 0x00，输出必须是 A=255。
  否则下游看到半透明画面，会以为是自己混色算错了。
  [通过] RGB32 转换成功
  [通过] BGR 三个通道顺序正确
  [通过] 第 4 字节被强制成不透明

--- 5. YUY2 的宏像素 ---
  4:2:2 里两个像素共用一组 U/V，所以它们只能同色系。
  Y=81 U=90 V=240 是 BT.601 的纯红。
  [通过] YUY2 转换成功
  [通过] 像素(0,0) 接近纯红
  [通过] 像素(1,0) 也是红（共用 U/V）
  [通过] 像素(0,1) 接近纯蓝
  [通过] 像素(1,1) 也是蓝
  实际值 (0,0): (B=0 G=0 R=255 A=255)
  实际值 (0,1): (B=255 G=0 R=0 A=255)
  [通过] 中性灰转换成功
  [通过] Y=U=V=128 得到约 130 的灰（说明做了 16~235 -> 0~255 的拉伸）
  实际值: (B=130 G=130 R=130 A=255)（偏移处理后应约为 130，不是 128 也不是 149）

--- 6. UYVY 的字节顺序 ---
  UYVY 只是把 U/V 挪到前面。读错顺序的话红蓝会对调。
  [通过] UYVY 转换成功
  [通过] 像素(0,0) 是红而不是蓝
  [通过] 像素(1,0) 也是红
  若这里的 R 和 B 对调，说明把 UYVY 当成 YUY2 读了

--- 7. 异常输入 ---
  [通过] 数据不够时返回 nullptr
  失败原因: 帧数据不完整：需要 48 字节，实际只有 47 字节
  [通过] 同时给出了失败原因
  [通过] 未知格式返回 nullptr
  [通过] 空数据返回 nullptr
  [通过] 尺寸为 0 返回 nullptr

--- 8. MJPG 解码 ---
  MJPG 每帧就是一张完整 JPEG，得先解码才能谈 stride。
  这里现场编一张纯红 64x64 的 JPEG 当输入，结果完全确定。
  注意 YUV 值要用**全范围**的：JPEG 的 Y 是 0..255（Y=76 U=85 V=255），
  不是摄像头那种有限范围的 16..235（Y=81 U=90 V=240）。
  生成了 6xx 字节的 JPEG，开头是 FF D8（JPEG 的 SOI 标记）
  [通过] MJPG 解码成功
  [通过] 解码出来的尺寸正确
  [通过] bgra 大小正确
  [通过] 中心像素接近纯红
  中心像素: (B=0 G=0 R=254 A=255)（应为 B≈0 G≈0 R≈255）
  [通过] 帧尾补了 4096 个零字节，仍能靠 EOI 兜底解出来
  [通过] 喂垃圾数据时返回 nullptr
  失败原因: MJPEG 解码失败: Invalid data found when processing input

--- 9. 真实摄像头端到端 ---
  打不开摄像头，这一节跳过（不是转换器的问题）

--- 10. 转换耗时 ---
  RGB24 1280x720 转换 30 次，共 xx 毫秒，平均 x.xx 毫秒/帧
  按 30fps 算，一帧的预算是 33.3 毫秒
  [通过] 单帧转换耗时低于 20 毫秒，够跑 30fps

================================================
  通过 30 项，失败 0 项
================================================
```

**几个要核对的关键点：**

1. **第 5 段的"实际值"必须是 130**（不是 128，也不是 149）。如果看到 128，说明 `yuvToBgra` 忘了减 16；看到 149，说明减了 16 但没用对系数。
2. **第 7 段的失败原因必须带具体数字**："需要 48 字节，实际只有 47 字节"。4×4 的 RGB24，stride = 12，`needed = 12*3 + 12 = 48`。✓
3. **第 8 段的 JPEG 开头必须是 `FF D8`**。
4. **第 8 段中心像素的 R 应该是 254 左右**（不是精确的 255，因为 JPEG 有损）。

### 8.3 实验一：故意写错，看测试怎么抓你

**这是最有价值的一个实验**——亲手制造 bug，看测试如何报警。

**实验 1a：把 B 和 R 写反。**

打开 `CameraFrameConverter.cpp`，找到第 55~57 行：

```cpp
dst[0] = clampByte(b);
dst[1] = clampByte(g);
dst[2] = clampByte(r);
```

改成：

```cpp
dst[0] = clampByte(r);   // 故意错
dst[1] = clampByte(g);
dst[2] = clampByte(b);   // 故意错
```

重新编译运行。**预期**：第 1 段测试全红（"红色"变成了"蓝色"），并且 `pixelText` 会打印出 `(B=255 G=0 R=0 A=255)`，让你一眼看穿。

**注意**：第 5 段（YUY2）也会红，因为它们共用 `yuvToBgra`。**这体现了共用函数的连锁效应**——一个 bug，多个测试报警。

改回来。

**实验 1b：忘了减 16。**

把 `const int c = y - 16;` 改成 `const int c = y;`。

**预期**：第 5 段的"中性灰"断言失败，实际值大约是 **149**（`128 * 1.164`）。第 5 段的红/蓝断言可能仍然通过（因为它们本来就是钳位到 0 或 255 的极端值）——**这说明"极端值测试"发现不了"中间调错误"，所以必须有中性灰那个子测试**。请大家体会：**测试用例的选择，决定了你能发现哪类 bug。**

改回来。

**实验 1c：按 9 字节跳行（忘用 stride）。**

把第 360 行的 `stride` 改成 `rowBytes`：

```cpp
const uint8_t* src = srcBase + static_cast<size_t>(y) * rowBytes;   // 故意错
```

**预期**：第 2 段测试红（因为那个测试专门造了带填充的帧）。第 1、3、4 段**仍然通过**——因为它们没有填充，`stride == rowBytes`。**这再次证明：只测"正常情况"的测试是无效的，必须专门构造"带填充"的用例。**

改回来。

**实验 1d：删掉 `flipVertical` 的处理。**

把第 362 行改成 `const int dstY = y;`。

**预期**：第 3 段后半部分红（"翻转后：第 0 行变成蓝"失败）。

改回来。

### 8.4 实验二：观察 MJPEG 解码的真实格式

在 `convertMjpeg` 里临时加一行日志（第 280 行之后）：

```cpp
std::printf("[调试] MJPEG 解码出: %dx%d  format=%d  color_range=%d  fullRange=%d\n",
            width, height, decodedFrame->format, decodedFrame->color_range,
            (int)sourceIsFullRange);
```

（需要 `#include <cstdio>`）

**观察要点：**
1. `format` 的值是多少？查 `AVPixelFormat` 枚举（在 `libavutil/pixfmt.h`）：`AV_PIX_FMT_YUVJ420P = 12`、`YUVJ422P = 13`、`YUVJ444P = 14`、`YUV420P = 0`。
2. `color_range` 是多少？`AVCOL_RANGE_UNSPECIFIED = 0`、`MPEG = 1`（有限）、`JPEG = 2`（全范围）。
3. **不同的摄像头可能给出不同的组合**——这正是 5.3.4 节那段"两条路都查"的兼容性代码存在的理由。如果你只测了一个摄像头，很可能只看到其中一种组合。

**如果你手上有多个摄像头**，逐个跑一遍，把结果记下来。你会发现**同一个摄像头在不同分辨率下也可能给不同格式**——这就解释了为什么 `swsFormat` 要作为缓存键。

### 8.5 实验三：观察尾部零填充的真实性

接上真实摄像头，在 `convertMjpeg` 的 `decode` 前后加日志，打印 `raw.bytes.size()` 和"EOI 出现在哪个位置"：

```cpp
// 在 EOI 搜索循环里
if (trimmed > 0)
    std::printf("[调试] 数据 %zu 字节，EOI 在第 %zu 字节处，尾部 %zu 字节是垃圾\n",
                raw.bytes.size(), trimmed, raw.bytes.size() - trimmed);
```

**观察要点**：
- 有些摄像头 `trimmed == raw.bytes.size()`（没有填充），此时第 231 行的 `trimmed < raw.bytes.size()` 判断会**跳过重试**——这是正确的；
- 有些摄像头尾部有几十到几千字节的垃圾；
- 垃圾可能是全 0，也可能是**上一帧的残留数据**（如果你看到 `FF D8` 之类的模式，那就是残留的 JPEG 头）。

### 8.6 实验四：性能测试的机器依赖性

分别用 **Debug** 和 **Release** 配置编译，各跑一次第 10 段，记录数字。

**预期差异**：Debug 可能是 Release 的 3~10 倍。

**思考**：如果你们的 CI 机器性能较弱，第 10 段的 `perFrame < 20.0` 可能会失败。**这时候该怎么办？** 是放宽阈值、还是跳过性能测试、还是改用"相对基准"（跟上次运行比）？这是个开放问题，我们在思考题里讨论。

### 8.7 实验五：造一个奇数宽度的帧

在测试程序里加一段：

```cpp
{
    std::printf("\n--- 附加：奇数宽度 ---\n");
    CameraRawFrame raw = makeFrame(3, 2, CameraPixelFormat::YUY2);
    setYuy2Pair(raw, 0, 0, 81, 81, 90, 240);
    setYuy2Pair(raw, 2, 0, 81, 81, 90, 240);   // 第 2、3 个像素（第 3 个不存在）
    VideoFramePtr out = converter.convert(raw);
    check(out != nullptr, "奇数宽度 YUY2 转换成功");
    if (out) {
        check(out->bgra.size() == 3 * 2 * 4, "输出大小正确（没有多写）");
        check(pixelIs(*out, 0, 0, 0, 0, 255, 2), "像素(0,0) 是红");
        check(pixelIs(*out, 2, 0, 0, 0, 255, 2), "像素(2,0) 是红（最后一个有效像素）");
    }
}
```

**验证的是什么？** 4.9 节讲的 `if (x + 1 < width)` 边界检查。如果删掉那个检查，输出缓冲区会被多写 4 字节——**在 Debug 模式下可能被堆管理器抓到，在 Release 模式下可能静默破坏相邻内存**。

**可以用 AddressSanitizer 验证**（GCC/Clang）：

```bash
cmake -S . -B build-asan -DCMAKE_BUILD_TYPE=Debug \
      -DCMAKE_CXX_FLAGS="-fsanitize=address -fno-omit-frame-pointer"
cmake --build build-asan
./build-asan/tests/test_camera_convert.exe
```

删掉 `if (x + 1 < width)` 后，ASan 会报 **heap-buffer-overflow**。**这是学习内存安全最好的工具，强烈推荐同学们装上试一次。**

### 8.8 实验六：把 BMP 换成 PPM 对比

`saveBmp` 里有"倒着写行"的逻辑。为了确认自己真的理解了，请写一个 `savePpm`（PPM 是**自顶向下**的简单格式）：

```
P6
<width> <height>
255
<RGB 字节流，从上到下>
```

注意 PPM 是 **RGB**（不是 BGR），所以要交换字节。

写一个 2×2 的测试帧（红/绿/蓝/白），分别用 `saveBmp` 和 `savePpm` 保存，用图片查看器打开对比。**如果两个文件看起来不一样，说明你对 BMP 的"自底向上"理解还不到位。**

---

## 第九章：坑点与改进

这一章讲的是"这段代码还有哪些不足"。**能看出自己代码的不足，是工程师成熟的标志。** 每一条我都会给出改进方向。

### 9.1 MJPG 路径忽略了 `flipVertical`

**问题**：`convertMjpeg` 完全没有读 `raw.flipVertical`。如果某个摄像头的 MJPG 流是自底向上的，输出画面会上下颠倒。

**实际情况**：JPEG 规范规定图像数据自顶向下存储，所以绝大多数 MJPEG 摄像头不会 Flip。但"绝大多数"不等于"全部"。

**改进方案**：在 `sws_scale` 之后（或之前）加一次翻转。最优雅的做法是**利用 swscale 本身**——把目标 `linesize` 设成负数、把 `data[0]` 指向最后一行，swscale 就会倒着写：

```cpp
uint8_t* dstData[4] = {out.bgra.data() + (size_t)(height - 1) * width * 4, nullptr, nullptr, nullptr};
int dstLinesize[4] = {-width * 4, 0, 0, 0};   // 负行跨度 = 倒着写
```

**这是 swscale 的一个不太为人知但非常好用的技巧**——零成本翻转。有兴趣的同学可以自己试。

### 9.2 MJPG 路径忽略了 `raw.width / raw.height`

**问题**：输出尺寸完全来自解码结果，不校验是否与采集层协商的一致。如果两者不一致（驱动谎报），下游的编码器配置（按协商尺寸设的）就会和实际帧尺寸冲突。

**改进方案**：至少加一条日志，或者在尺寸不符时警告：

```cpp
if (raw.width > 0 && raw.height > 0 && (raw.width != width || raw.height != height)) {
    // 记日志：摄像头声称 AxB，实际解码出 CxD，以解码结果为准
}
```

### 9.3 色彩标准写死为 BT.601

**问题**：第 273 行无条件用 `SWS_CS_ITU601`。高清摄像头（720p/1080p 及以上）的 MJPEG 内容**可能按 BT.709 编码**（尤其 newer 摄像头、或者摄像头内部做了色彩处理后输出）。用 BT.601 解 BT.709 的内容，颜色会偏（饱和度不对）。

**改进方案**：
1. 读 `decodedFrame->colorspace`（FFmpeg 会解析 JPEG 里的 APP14/Adobe 段或 EXIF 里的色彩空间标记）；
2. 根据它选 `SWS_CS_ITU601` / `SWS_CS_ITU709` / `SWS_CS_DEFAULT`；
3. 或者把色彩标准做成 `CameraFrameConverter` 的一个可配置参数，让上层根据分辨率选（< 720p 用 601，>= 720p 用 709）。

**同样的注意**：`yuvToBgra`（手写路径）也写死了 BT.601。对 YUY2/UYVY 摄像头来说这通常是对的（标清传统），但也不是 100%。

### 9.4 手写转换没有 SIMD 优化

**问题**：`convertRgb24Row` 等四个函数是纯标量（scalar，一次处理一个像素）实现。1080p 一帧 200 万像素，每像素 4~6 次内存访问 + 3 次乘法。

**实际影响有多大？** 第 10 段测试给出了答案：720p 通常 2~8 毫秒/帧，够 30fps。但**如果将来要跑 60fps 或 4K，就会吃紧**。

**改进方案**（按性价比排序）：
1. **换用 libyuv**（Google 的开源库，专做这个）：它有高度优化的 SIMD 实现（`I420ToARGB`、`YUY2ToARGB` 等），而且 API 简单；
2. **换用 swscale** 处理所有格式（统一路径，代码更少）——但会引入"解码器之外的又一次库调用开销"，且失去可控性；
3. **手写 SIMD**（SSE2/AVX2 intrinsic）：性能最好，但可移植性和可读性最差。

**我的建议**：先用当前的标量实现跑起来（**正确性优先**），等性能真的成为瓶颈时再换 libyuv。**过早优化是万恶之源**——这段话在 1.12 节讲"手写 vs 用库"时也说过。

### 9.5 `packedRowBytes` 在测试里被复制了一份

**问题**：`CameraFrameConverter.cpp` 的匿名命名空间里有一份，`test_camera_convert.cpp` 里有一份（第 46~59 行）。**两份代码，改一处忘另一处就会不一致。**

**但这个问题是"有意为之"还是"疏忽"？** 7.0.4 节讲过：测试**应该**独立推导期望值，而不是调用被测代码的同一个函数。所以从"测试有效性"角度，复制是对的。

**更好的方案**：把它提升为一个公共函数，放在 `CameraCapturer.h`（或新的 `PixelFormatUtils.h`）里，两边都用。同时测试里**增加**一个"手算值 vs 函数值"的交叉验证用例。这样既避免了重复，又保留了独立性。

### 9.6 `const_cast` 手动填 packet 依赖 FFmpeg 内部行为

**问题**（5.3.1 节详细讨论过）：`packet->data = const_cast<uint8_t*>(data)` 只设了 `data` 没设 `buf`，安全性依赖"`av_packet_unref` 在 `buf == NULL` 时不 free data"这个内部实现细节。

**改进方案**：用官方 API：

```cpp
// 方案 A：让 FFmpeg 接管我们分配的缓冲（要自己先分配）
uint8_t* buf = (uint8_t*)av_malloc(size);
memcpy(buf, data, size);
av_packet_from_data(packet, buf, size);   // FFmpeg 负责释放

// 方案 B：让 FFmpeg 分配，我们拷进去
av_new_packet(packet, size);
memcpy(packet->data, data, size);
```

代价是**多一次拷贝**；收益是**不依赖内部实现**。对于每帧几百 KB、每秒 30 帧的场景，拷贝开销约几十 MB/s，通常可以接受。**在"零拷贝的性能"和"依赖内部实现的脆弱"之间，我倾向于后者**——除非压测证明拷贝确实是瓶颈。

### 9.7 解码失败后解码器状态可能不干净

**问题**（5.3.2 节提过）：第一次 `decode` 失败后，我们直接重试。但 FFmpeg 的解码器在 `send_packet` 失败后，内部状态可能已经"半初始化"。虽然实践中 MJPEG 解码器通常能恢复，但**没有文档保证**。

**改进方案**：重试前先 `avcodec_flush_buffers(mjpegCtx)`，或者更彻底——失败时 `releaseDecoder()` 释放整个解码器，下次调用时懒加载重建。**后者更简单也更安全**，代价是重建开销（只在真正的错误路径上发生，可接受）。

### 9.8 EOI 搜索的潜在误判

**问题**：代码从后往前找第一个 `FF D9`，认为它就是 EOI。1.11 节说过 JPEG 的字节填充保证了熵编码数据里不会有裸 `FF D9`。但如果：
- 摄像头的 JPEG **不合规**（没做字节填充）；
- 或者数据末尾是**上一帧的残留**，残留里恰好有 `FF D9`；

就可能截错位置。

**改进方案**：找到 EOI 后**额外验证**——检查前两个字节是不是 `FF D8`（SOI）。或者更严格：只在"截断点之后的数据全为零"时才截断（这才是真实场景的实际情况）。

```cpp
// 更严格的判据：EOI 之后必须全是 0（或全是无意义填充）
bool tailIsPadding = true;
for (size_t i = trimmed; i < raw.bytes.size(); ++i)
    if (raw.bytes[i] != 0) { tailIsPadding = false; break; }
if (tailIsPadding) ret = decode(raw.bytes.data(), trimmed);
```

### 9.9 性能测试在慢机器上会误报

**问题**（7.11 节提过）：第 10 段的 `perFrame < 20.0` 是硬断言。在 CI 的虚拟机、或者低端笔记本上可能失败——**但那不是代码的问题**。

**改进方案**：
1. 把性能断言降级为**警告**（打印 `[警告]` 但不计入 `g_failed`）；
2. 或者用**环境变量**控制是否启用性能断言；
3. 或者把阈值调得更宽松（比如 30 毫秒），只抓"数量级的退化"。

**原则**：**正确性测试必须是硬断言，性能测试应该是软警告**——因为它们失败的含义完全不同。

### 9.10 没有线程安全保证

**问题**：`CameraFrameConverter` **不是线程安全的**。它内部持有可变的 FFmpeg 对象，如果两个线程同时调 `convert()`，会数据竞争（data race）。

**当前可以接受吗？** 可以——因为典型的用法是"一个采集线程调 `convert()`，转换完把 `VideoFrame` 交给下游"。**单消费者模型**。

**但要注意**：如果将来要"并行转换多路摄像头"，就必须**每个摄像头一个 `CameraFrameConverter` 实例**（不能共享）。这一点应该写进类的注释里。

### 9.11 pts 依然是帧序号

**问题**（6.4 节提过）：`out->pts = raw.pts` 只是透传。而第 5 讲的 `pts` 是 `frameCounter.fetch_add(1)`——**帧序号，不是时间戳**。所以下游拿到的"时间戳"是 0、1、2、3…… 如果按毫秒解释，就会得出"帧率 1000fps"的荒谬结论。

**这不是本讲引入的问题，但本讲是"最后一次能顺手修的地方"**。改进方案在第 5 讲的思考题里讨论过：把 `sampleTime` 换算成毫秒。这个问题会一直留到我们做编码器时间戳时才被迫解决——**到时候要改的地方会更多**。这是"技术债"的典型形态：**越晚还，利息越高。**

### 9.12 其余小坑

- **`out->bgra.resize()` 可能抛 `std::bad_alloc`**：1080p 是 8 MB，4K 是 33 MB。如果内存不足会抛异常，而 `convert()` 没有捕获。**在长期运行的推流程序里，内存压力是真实存在的**，应该考虑捕获并转成"返回 nullptr + lastError"。
- **`mjpegCtx->thread_count = 1` 是硬编码**：某些场景（比如批量转码已有视频文件）多线程会更快。可以做成构造参数。
- **`<cstring>` 是冗余 include**（4.1 节提过）。
- **BMP 保存没有错误检查中间步骤**：`fwrite` 失败（磁盘满）不会被发现。
- **测试第 8 段依赖 MJPEG 编码器**：如果 FFmpeg 没编编码器，MJPEG 解码路径就完全没有测试覆盖。可以硬编码一小段 JPEG 作为兜底素材。

---

## 第十章：小结

这一讲我们完成了从"采集"到"可用"的关键一跃。回顾主线：

1. **统一格式层的价值是 M×N → M+N**：五种输入方言收敛成一种 BGRA 普通话，下游从此只写一遍。这是本讲最重要的架构思想。
2. **为什么是 BGRA**：4 字节对齐、下游（GDI/Qt/编码器）全是这个口味、Alpha 留位让"跳下一个像素 = +4"、以及小端序的历史。
3. **BGR 与 RGB 是同一块数据的两种读法**：参数用 RGB（人类），内存用 BGR（机器）。红蓝对调 = 要么通道写反，要么 U/V 读反。
4. **YUV 的本质是亮度/色度分离 + 色度下采样**：4:2:2 里两个像素共用一组 U/V，这是"宏像素"概念的来源，也是所有解包代码的依据。
5. **BT.601 整数公式（298/409/100/208/516 + `>> 8`）**：1.164 等浮点系数的 256 倍近似，加 128 是四舍五入。
6. **有限范围 vs 全范围是反复踩的坑**：`Y=U=V=128` 应该得到 **130**（不是 128 也不是 149），这三个数字是三种实现的指纹。
7. **MJPEG 是"一串独立 JPEG"**：省 USB 带宽的刚需，代价是必须解码。JPEG 全范围 + SOI/EOI 标记 + EOI 兜底容错。
8. **swscale 上下文必须缓存**：`sws_getContext` 很贵，用"宽/高/格式"三指纹做失效判断。
9. **stride 和 flipVertical 在这一层被终结**：用 `stride` 跳行、用 `dstY` 镜像写——两个第 5 讲遗留问题在此收口，下游彻底摆脱 stride。
10. **FFmpeg 用 C 风格 API，靠 PIMPL 和 RAII 裹进 C++**：头文件零 FFmpeg 痕迹，析构函数兜底释放，每帧 `unref` 两次。
11. **测试即教案**：十段测试覆盖正确性、健壮性、集成性、性能四个维度，毒化填充、边界值、控制变量、人工确认清单——每一处都是可复用的测试技巧。

至此，**采集层全部完成**。屏幕和摄像头两条支路都产出了标准的 `VideoFrame`。下一讲我们将进入**编码器**：把这些 BGRA 帧压成 H.264，装进 FLV，推向服务器——那是 FFmpeg 真正大显身手的地方。

---

## 思考题

1. 本讲说"统一格式层把 M×N 变成 M+N"。请结合本项目实际论证：如果不加这一层，编码器模块要为几种输入写几份代码？再往后加"图片文件输入（PNG/JPEG）"和"桌面+摄像头画中画"两个功能时，两种方案（有/无统一层）的工作量分别是多少？

2. `convert()` 第 340 行写的是 `const int stride = raw.stride >= rowBytes ? raw.stride : rowBytes;`。请分析：如果改成直接用 `raw.stride`（不兜底），在什么情况下会出错？出错的画面是什么样？再回答：为什么用 `rowBytes` 兜底是"安全"的（它一定不小于真实需求吗）？

3. `needed` 公式是 `stride * (height - 1) + rowBytes`，最后一行只要求 `rowBytes`。请论证这个公式的正确性，并回答：如果改成 `stride * height`，在什么真实场景下会**误判为"数据不完整"**？举一个具体的驱动行为例子。

4. 假设某个 1920×1080 的 RGB24 摄像头，`stride = 5760`（恰好无填充），`flipVertical = true`。请计算：转换后 `out->bgra.size()` 是多少？源第 0 行的第一个字节，会被写到输出缓冲的第几个字节（给出偏移量计算过程）？

5. `yuvToBgra` 里三个公式中，R 的公式不含 `d`（U），B 的公式不含 `e`（V）。请从 YUV 的定义（U ≈ B−Y，V ≈ R−Y）出发解释这是为什么。进一步：如果有人把 `convertUyvyRow` 写成了 `convertYuy2Row`（字节顺序错），请推导 U 和 V 会变成什么值，并预测画面现象。

6. `convertMjpeg` 里 `sws_setColorspaceDetails(sws, coefficients, sourceIsFullRange ? 1 : 0, coefficients, 1, 0, 1 << 16, 1 << 16)`。请逐个解释后四个参数（`1`、`0`、`1<<16`、`1<<16`）的含义；再回答：如果把第 5 个参数（目标范围）改成 `0`，输出画面会变成什么样？为什么？

7. 本讲在 MJPG 路径用 `out.bgra.assign(size, 0)`（清零），在裸格式路径用 `out->bgra.resize(size)`（不清零）。请论证这两种选择各自的依据是什么。如果把裸格式路径也改成 `assign(..., 0)`，会有什么代价？反过来把 MJPG 路径改成 `resize`，会有什么风险？

8. `ensureDecoder()` 采用了懒加载。请设想：如果改成在 `CameraFrameConverter` 的构造函数里无条件初始化 MJPEG 解码器，会带来哪三个具体问题？其中哪一个最严重？再回答：懒加载在什么场景下反而是**不好**的选择（提示：想想"第一次调用的延迟抖动"）。

9. 测试第 2 段把填充字节填成 `0xAA 0xBB 0xCC` 而不是 0。请解释这个选择的作用，并举出至少两个其它场景（内存调试、结构体填充、未初始化变量）中同类技巧的应用。

10. 本讲第 9 章列了 12 条局限。请挑出你认为**最需要优先修复**的三条，说明排序理由（考虑：后果严重性 × 发生概率 × 修复成本），并给出其中一条的具体修改代码。

---

**本讲配套文件**：`src/core/capture/CameraFrameConverter.h`（24 行）、`src/core/capture/CameraFrameConverter.cpp`（384 行）、`tests/test_camera_convert.cpp`（600 行）、`src/core/capture/CMakeLists.txt`、`tests/CMakeLists.txt`

**上一讲回顾**：`commit-docs/05-commit-281a22be-摄像头采集DirectShow.md` —— CameraCapturer（DirectShow 过滤器图 + Sample Grabber 回调）

**下一讲预告**：视频编码 —— 把 BGRA 帧压成 H.264（FFmpeg 编码器 + 时间基与 PTS 换算）

---

## 附：本讲"五章对照索引"

| 标准章节 | 在本讲的位置 | 一句话要点 |
|---|---|---|
| **核心原理** | 开场白 + 第一章 1.1/1.2/1.4/1.5/1.6/1.7/1.9/1.14 | 统一表示层把 M×N 降为 M+N；选 BGRA 的四条理由；YUV 的亮度/色度分离与色度下采样（4:2:2 宏像素）；打包 vs 平面；有限范围 16~235 与全范围 0~255；stride 填充导致斜纹、flipVertical 用"写入时镜像"解决 |
| **用到的技术 / 知识点** | 第一章全部小节 + 第二、七章 | BT.601/BT.709 矩阵及整数近似（298/409/100/208/516，`+128` 四舍五入，`>>8`）；YUY2 (`Y0 U Y1 V`) 与 UYVY (`U Y0 V Y1`) 字节布局；JPEG/MJPEG、SOI `FFD8` / EOI `FFD9`、字节填充；swscale 与 SwsContext 缓存；FFmpeg 解码 send/receive 模型、`av_strerror`、`unref` 节奏；PIMPL + RAII；PIXELFORMAT 与 BMP 文件（`#pragma pack`、`0x4D42`、自底向上）；CMake 的 PUBLIC/PRIVATE；测试技巧（毒化填充、边界值、控制变量、容差、退出码） |
| **实现细节** | 第二~七章 | 两处 CMakeLists 改动（`PRIVATE ${FFMPEG_INCLUDE_DIRS}` 的意义）；头文件 24 行契约（禁拷贝、返回 `shared_ptr`、PIMPL）；匿名命名空间八件工具（`clampByte`/`packedRowBytes`/`avErrorText`/`yuvToBgra`/四个 `convertXxxRow`）；`Impl` 的状态与 `releaseDecoder()` 的幂等释放顺序；`ensureDecoder()` 五步懒加载；`convertMjpeg()` 六段（decode lambda、EOI 兜底、尺寸校验、Sws 缓存重建、`sws_scale`、unref）；`convert()` 八步（清错误、三重校验、pts 透传、MJPG 分流、stride 兜底、`needed` 校验、resize、逐行循环） |
| **关键方法说明** | 第四章工具表 + 5.3.7 / 6.10 / 6.11 全景图 | `convert`（主入口，唯一对外方法）；`convertMjpeg`（MJPG 专属链路）；`ensureDecoder`（懒加载，五步初始化 + 失败路径清理）；`releaseDecoder`（幂等，从依赖者到被依赖者）；`yuvToBgra`（BT.601 核心 15 行）；四个 `convertXxxRow`（统一签名 `(src, dst, width)`，开关在循环外）；`lastError()`；测试侧 `makeFrame`/`setXxx`/`pixelIs`/`pixelText`/`saveBmp`/`encodeTestJpeg` |
| **缺陷与局限** | 第九章"坑点与改进"（9.1~9.12） | MJPG 路径忽略 `flipVertical` 与 `raw.width/height`；色彩标准写死 BT.601（未读 `colorspace`）；手写转换无 SIMD（可换 libyuv）；`packedRowBytes` 与测试重复；`const_cast` 手动填 packet 依赖 FFmpeg 内部行为；解码失败后未 flush/重建解码器；EOI 搜索可能误判；性能测试硬断言在慢机器误报；无线程安全保证（多摄像头需多实例）；pts 仍是帧序号（第 5 讲遗留）；`resize` 可能抛 `bad_alloc`；`thread_count` 硬编码；`<cstring>` 冗余；测试第 8 段依赖 MJPEG 编码器 |
