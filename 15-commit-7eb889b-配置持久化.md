# 第 15 课：配置持久化 —— 让软件"记住"用户上一次填的参数

> **对应提交**：`7eb889bf3c2c3b4d99c0534247cdf08ddfa5a2b5`（简称 `7eb889b`）
> **提交时间**：2026-10-23 18:03
> **提交信息原文**：配置持久化
> **改动规模**：5 个文件，**+185 行**（`MainWindow.cpp` +60 行 / `MainWindow.h` +3 行 / `src/app/CMakeLists.txt` +16 行 / 新增 `package.bat` 29 行 / `.gitignore` +1 行）
> **本课核心代码**：
> - `Streaming-tools/src/app/MainWindow.cpp`（本次新增 `loadSettings()` 与 `saveSettings()` 两个函数，以及三处调用点）
> - `Streaming-tools/src/app/MainWindow.h`（新增两个成员函数声明）
> - `Streaming-tools/package.bat`（新增，一键打包脚本）
> - `Streaming-tools/src/app/CMakeLists.txt`（新增 install 规则）
> **课前准备**：第 3 课（`StreamConfig` 的定义与"默认值即推荐配置"哲学）、第 12 课（Qt 主界面，每个控件的成员变量名、`collectConfig()` 怎么把界面变成配置）
> **课后产物**：一台"关掉再打开参数还在"的 LightLiveStreamer，以及一个可以整目录拷走的 `dist/` 发布包
> **本课一句话**：**参数每次打开程序都要重填，是软件体验的最后一公里——这节课让配置记住。**

---

## 目录（这节课我们怎么走）

| 章节 | 内容 |
|---|---|
| 第 0 章 | 开场白：最后一公里；本次提交 5 个文件总览 |
| 第 1 章 | 知识点预习（一）：什么是配置持久化；四种方案对比（QSettings / INI / JSON / 注册表）；文本与二进制 |
| 第 2 章 | 知识点预习（二）：QSettings 工作原理——组织名与应用名、键的层次、`value(key, 默认值)`、Windows 上到底存哪了 |
| 第 3 章 | 知识点预习（三）：五个工程问题——路径选择、写入时机、读取兜底、版本兼容、UI 与配置的双向绑定 |
| 第 4 章 | 逐行教学（一）：`MainWindow.h` 的两行声明、`#include <QSettings>`、构造函数里的调用点 |
| 第 5 章 | 逐行教学（二）：`loadSettings()` 全文 33 行——读取方向的每一行 |
| 第 6 章 | 逐行教学（三）：`saveSettings()` 全文 19 行 + `startStreaming()` / `closeEvent()` 两处保存时机 |
| 第 7 章 | 逐行教学（四）：`.gitignore`、`package.bat`、`src/app/CMakeLists.txt` 的 install 规则——把程序打包送人 |
| 第 8 章 | 动手环节：改参数→关→开→去注册表亲眼看见配置；三个破坏实验 |
| 第 9 章 | 排错手册：配置没存上、中文乱码、配置损坏、下拉框越界 |
| 第 10 章 | 小结、速查表、局限与思考题（8 题含答案） |
| 附 | 本讲「五章对照索引」 |

> **老师的课前提醒**：这次提交只有 185 行，是本项目到目前为止**最小的一次提交**。但同学们不要小瞧它——前面十四课我们解决的都是"程序能不能跑"的问题，这一课解决的是"**人愿不愿意用**"的问题。功能决定软件的下限，体验决定软件的上限。而且这节课要讲的 QSettings，是 Qt 里最"润物细无声"的一个类：两行声明、两个函数、三处调用，配置持久化就做完了。**简单，但每一个设计决策都值得掰开讲。**

---

# 第 0 章 开场白：参数重填，是软件体验的最后一公里

## 0.1 先体验一下"没有这节课"的世界

同学们，我们先做一个思想实验。假设没有这次提交，我们的 LightLiveStreamer 在第 14 课就已经"功能完整"了：能抓屏、能抓摄像头、能采声音、能编码、能推流、能预览、断线还能重连。你兴冲冲地双击打开它，填好：

- 视频来源选"摄像头"，设备选"USB Camera"；
- 分辨率 1920×1080，帧率 30，码率 4500；
- 音频选"麦克风"，设备选"USB 声卡"；
- 推流地址工工整整贴了两行：B 站一路、抖音一路。

点"开始推流"，一切正常。下播，关窗口，睡觉。

第二天再打开——**所有参数全部回到了出厂状态**：来源是"屏幕"，分辨率是 1280×720，码率 2500，推流地址一片空白。你昨天贴的那两行 rtmp 地址，没了。你得翻聊天记录重新找、重新贴、重新调。

一次两次能忍，天天如此呢？**用户会用脚投票。**

我们常说"最后一公里"这个词——它本来指物流配送中从站点到用户家门口的最后一段路，路程最短，却直接决定用户对整个服务的评价。软件也一样：采集、编码、推流这些"主干道"我们修了十四课，而"**记住用户的参数**"就是送到家门口的那最后一步。路程最短（185 行），却最影响体验。

这节课，我们就把这最后一公里修通。

## 0.2 回头看看：我们的流水线全貌

| 课次 | 提交 | 一句话 | 流水线上对应哪一段 |
|---|---|---|---|
| 第 1 课 | `20b267a1` | 项目骨架与构建系统 | 盖厂房、通水通电 |
| 第 2 课 | `df3603c8` | 假画面 → H.264 → FLV → 文件 | 第一次把整条线跑通 |
| 第 3 课 | `a4453bb1` | `StreamConfig` / `VideoFrame` / `AudioChunk` / `BoundedQueue` | 定"零件规格"和"传送带" |
| 第 4 课 | `81eaa1f5` | DXGI / GDI 抓屏 | 原料采购（视频） |
| 第 5 课 | `281a22be` | DirectShow 摄像头 | 原料采购（视频，另一路） |
| 第 6 课 | `e92cbd3` | 摄像头五种格式 → BGRA | 原料预处理 |
| 第 7 课 | `1ce7f62` | WASAPI 抓声音 | 原料采购（音频） |
| 第 8 课 | `66e0768` | 两个编码器 | 加工车间 |
| 第 10 课 | `0ccd108` | FLV 封装 + 多目标分发 | 打包发货 |
| 第 11 课 | `31c3f57` | 三线程两队列 | 给流水线装马达 |
| 第 12 课 | `3b22c69` | Qt 主界面 | 盖调度室 |
| 第 13 课 | `bafb4ea` | 实时预览 | 装监控屏幕 |
| 第 14 课 | `c2b8021` | 断线重连 | 加应急预案 |
| **第 15 课（本次）** | **`7eb889b`** | **配置持久化** | **把操作员的配方记进本子** |

注意一个呼应关系：第 3 课我们定义了 `StreamConfig`——内存里的"一份直播的全部配置"；第 12 课我们写了 `collectConfig()`——把界面上的控件值收集成 `StreamConfig`；这节课我们补上的，是**把时间维度拉长**：不只活在这一次运行里，还要活到下一次启动。

## 0.3 本次提交改了哪 5 个文件

先看全貌，心里有个地图：

```
.gitignore                 |   +1     忽略 dist/ 打包目录
package.bat                |  +29     新增：一键打包脚本
src/app/CMakeLists.txt     |  +16     新增：install 规则（exe + Qt 运行库 + 插件）
src/app/MainWindow.cpp     |  +60     核心：loadSettings() / saveSettings() + 三处调用
src/app/MainWindow.h       |   +3     两个成员函数声明
```

这个提交其实是**两件事**：

1. **主线（标题说的"配置持久化"）**：`MainWindow.h` + `MainWindow.cpp` 里的 63 行，用 `QSettings` 把界面参数存起来、读回来。
2. **配套（顺手把"发布"也做了）**：`package.bat` + `CMakeLists.txt` 的 install 规则 + `.gitignore`，让 `cmake --install` 能把 exe 和它依赖的 Qt DLL 收进一个 `dist/` 目录，整个目录拷给别人就能跑。

为什么打包要蹭这次提交？老师的理解是：**配置持久化是"给用户用"的最后一环，打包是"到用户手里"的最后一环**——两件事都是"交付"，作者就一起做了。讲稿里我们都会覆盖，主线在第 4~6 章，配套在第 7 章。

## 0.4 老师先剧透：核心代码只有"两个函数 + 三处调用"

```cpp
void loadSettings();   // 启动时：存储 → 界面（读）
void saveSettings();   // 关键时机：界面 → 存储（写）
```

三处调用点：

| 调用点 | 时机 | 作用 |
|---|---|---|
| 构造函数里 `loadSettings()` | 窗口建好之后、显示之前 | 上次的参数填回界面 |
| `startStreaming()` 开头 `saveSettings()` | 点"开始推流"时 | 把这份"确认可用"的参数立刻落盘 |
| `closeEvent()` 里 `saveSettings()` | 关窗口时 | 无论推没推过流，走之前都记一笔 |

同学们先记住这个骨架，后面逐行讲的时候不断回来看它。

---

# 第 1 章 知识点预习（一）：配置持久化的方案选型

## 1.1 什么是"配置持久化"

**持久化**（persistence），通俗讲就是"**让数据活得比进程久**"。程序运行时的变量都存在内存里，进程一退出，内存被操作系统回收，一切归零。持久化就是把需要跨运行保留的数据，写到掉电也不丢的介质上——通常是磁盘——下次启动再读回来。

我们这里要持久化的东西很明确，就是第 12 课界面上那一排控件的当前值：来源、设备、分辨率、帧率、码率、推流地址……它们加起来就是"用户的偏好"。

## 1.2 摆在桌面上的四种方案

| 方案 | 存成什么样 | 优点 | 缺点 | 适合谁 |
|---|---|---|---|---|
| **手写 JSON 文件** | 一个 `.json` 文本文件 | 跨平台、结构灵活、人能读懂、git 友好 | 要自己选库（nlohmann/json 等）、自己处理读写和解析失败 | 配置结构复杂、需要跨语言交换 |
| **手写 INI 文件** | 一个 `.ini` 文本文件 | 格式简单、人能读懂 | 自己写解析器是重复造轮子 | 已基本没人手写 |
| **QSettings（INI 模式）** | Qt 帮你读写 `.ini` | API 极简、跨平台、文件可见可改 | 层级只支持一层分组 | 中小型 Qt 程序 |
| **QSettings（原生模式，默认）** | Windows 注册表 / macOS plist / Linux ini | **零路径管理**，跟着系统走，用户级隔离天然支持 | Windows 上配置藏在注册表里，不透明 | 只想"能用就好"的桌面软件 |

我们这次的选择是第四种：**`QSettings` 原生模式**。选它的理由很朴素——**我们的配置只有十来个扁平的键值对，没有任何嵌套结构，用最省事的方案就是最优解**。这就是工程里的"够用原则"：不为了用 JSON 而用 JSON。

> **专业名词**：**注册表**（Windows Registry）是 Windows 系统自带的一棵"键值对大树"，系统和软件都把配置存在里面。可以把它理解成"Windows 内置的、带权限管理的巨型配置文件"。普通用户通过 `regedit.exe`（注册表编辑器）查看它。

## 1.3 文本文件与二进制：为什么不能把内存直接"拍"到磁盘上

有同学会问：配置不就是一个结构体吗，我 `fwrite(&config, sizeof(config), 1, fp)` 一把梭写进二进制文件不行吗？

不行，至少不能这么粗暴。三个原因：

1. **结构体内存布局不稳定**。`StreamConfig` 里有 `std::vector<StreamTarget>`，vector 存的是指向堆内存的指针，你把指针的值写进文件，下次读出来指向的是一片已经不属于你的内存——这叫**悬垂指针**（dangling pointer，指针指向的对象已经没了），一解引用就是未定义行为。
2. **布局随编译器和版本变**。结构体成员的**对齐**（alignment，编译器为了访问效率在成员之间插的填充字节）会因编译选项、平台、甚至编译器版本不同而不同。今天写的文件，换个编译器编译的明天版本就读不出来了。
3. **没法人工查看和修复**。文本配置坏了，用户用记事本打开还能看懂、能手改；二进制坏了就是一堆乱码，只能删了重来。

所以持久化的正确姿势永远是：**把内存对象"翻译"成一种稳定的、与内存布局无关的外部格式**（这个过程叫**序列化**，serialization；反过来叫**反序列化**，deserialization）。QSettings 做的就是这个翻译——它把我们的 int、QString、QSize 翻译成注册表里的字符串值。

## 1.4 顺带认识：键值对思维

不管选哪种方案，持久化配置在概念上都是一张"键 → 值"的表：

```
video/source      = 1
video/width       = 1920
video/height      = 1080
video/fps         = 30
video/bitrate     = 4500
audio/source      = 1
audio/device      = 0
targets/text      = "bilibili=rtmp://...\ndouyin=rtmp://..."
```

读配置 = 按键查表；写配置 = 按键填表。QSettings 的 API 就是这张表的直接映射：`value(键, 默认值)` 查，`setValue(键, 值)` 填。掌握了这个思维，第 5、6 章的代码就是"查表填表"的体力活。

---

# 第 2 章 知识点预习（二）：QSettings 工作原理

## 2.1 一个类，三种后端

`QSettings` 是 Qt 提供的配置读写类。它的设计妙处在于：**API 完全一样，存到哪里由"格式 + 作用域"决定**。

- **格式**（format）：
  - `NativeFormat`（默认）：用操作系统最"原生"的方式——Windows 是**注册表**，macOS 是 plist 属性列表文件，Linux/Unix 是 ini 文件。
  - `IniFormat`：不管什么平台，都存成一个 ini 文本文件。
- **作用域**（scope）：
  - `UserScope`（默认）：当前用户的配置。
  - `SystemScope`：全机器所有用户共享的配置（Windows 上写它需要管理员权限）。

我们代码里写的是 `QSettings settings;`——**全默认**，也就是"NativeFormat + UserScope"。在 Windows 上，这意味着配置进了注册表。

## 2.2 `QSettings settings;` 凭什么知道我是谁？

同学们注意，构造函数一个参数都没传。那它怎么知道该往注册表的哪个位置写？答案在 `main.cpp`（第 12 课就写好的三行）：

```cpp
app.setApplicationName("LightLiveStreamer");
app.setApplicationDisplayName("LightLiveStreamer");
app.setOrganizationName("LightLiveStreamer");
```

QSettings 的默认构造函数会自己去问 `QCoreApplication`："你叫什么？你的组织叫什么？"然后拼出存储位置。Windows 注册表里的完整路径是：

```
HKEY_CURRENT_USER\Software\LightLiveStreamer\LightLiveStreamer
                ↑                ↑                ↑
        用户作用域的固定根   组织名          应用名
```

`HKEY_CURRENT_USER`（常缩写 HKCU）是注册表里"当前登录用户"专属的大分支，写它不需要管理员权限——这就是为什么 UserScope 是安全省心的默认选择。

> **排错预告**：如果哪天有人把 `main.cpp` 里 `setOrganizationName` 那行删了，QSettings 会退而用一个空组织名，配置写到另一个位置，表现为"配置突然全丢了"。第 9 章排错我们会回来说这条。

## 2.3 键的层次：`"video/source"` 里的斜杠是什么

QSettings 的键（key）里可以用 `/` 表示**分组**：

```cpp
settings.setValue("video/fps", 30);
```

在注册表里，这会变成：`HKCU\Software\LightLiveStreamer\LightLiveStreamer\` 下面建一个子键（子文件夹）`video`，里面有一个名为 `fps` 的字符串值，内容是 `"30"`。

打开注册表编辑器看，长这样：

```
LightLiveStreamer          ← 组织
└── LightLiveStreamer      ← 应用
    ├── audio
    │   ├── device = "-1"
    │   └── source = "0"
    ├── targets
    │   └── text = "bilibili=rtmp://..."
    └── video
        ├── bitrate = "2500"
        ├── cameraIndex = "0"
        ├── fps = "30"
        ├── height = "720"
        ├── source = "0"
        └── width = "1280"
```

分组纯粹是为了**人类可读**——键多了之后，`video/`、`audio/`、`targets/` 三个前缀让配置一目了然。对程序来说 `"video/fps"` 就是一个普通的字符串键名。

## 2.4 `value(key, 默认值)`：本课最重要的一个 API

```cpp
const int fps = settings.value("video/fps", 30).toInt();
```

这一行浓缩了三个知识点：

1. `settings.value(...)` 的返回类型是 **`QVariant`**。
   > **专业名词**：**QVariant** 是 Qt 的"万能容器"类型，一个 QVariant 可以装 int、double、QString、QSize……各种类型。因为配置里的值类型五花八门，所以 QSettings 统一用 QVariant 返回，你再调 `.toInt()`、`.toString()`、`.toSize()` 转成你要的类型。
2. 第二个参数 `30` 是**默认值**：如果注册表里**根本没有**这个键（比如软件第一次运行），`value()` 不会报错，而是返回这个默认值。**这是"读取失败兜底"的第一道防线，而且是 API 层面自带的。**
3. `.toInt()` 做类型转换。**转换失败怎么办？**——比如注册表里的值被人手贱改成了 `"abc"`——`toInt()` 不会抛异常、不会崩，而是返回 `0`。这是第二道防线（虽然不精确，但绝不崩）。

把这三点连起来就是 QSettings 的使用哲学：**读配置永远给一个默认值，转换永远有兜底，因此读配置这条路在代码层面"不可能失败"**。这正是我们第 3 章要专门讲的"读取失败的兜底"在 Qt 里的标准答案。

## 2.5 写出去：`setValue` 与"什么时候真正落盘"

```cpp
settings.setValue("video/fps", 30);
```

`setValue` 把键值放进 QSettings 的内部缓冲区。什么时候真正写进注册表？——**QSettings 对象析构时**（也可以手动调 `sync()` 强制立刻写）。

回看我们的代码：

```cpp
void MainWindow::saveSettings()
{
    QSettings settings;   // 栈上对象
    settings.setValue(...);
    ...
}   // ← 函数结束，settings 析构，所有改动一次性写盘
```

`settings` 是函数里的**局部变量**（栈对象），函数一return它就析构，析构自动落盘。所以我们的代码里没有出现 `sync()`，却从不用担心"忘了保存"。这是 C++ 的 **RAII**（Resource Acquisition Is Initialization，资源获取即初始化）惯用法又一次立功——和第 10 课"谁 new 谁 free"是同一个思想：**把清理动作绑在对象的生命周期上，想忘都忘不掉**。

## 2.6 一个合理的疑问：为什么每次函数里 new 一个 QSettings，而不是存成成员变量？

可以存成成员，但没必要。QSettings 对象本身很轻（它不"持有文件句柄"，更像一个"路径 + 缓存"的壳），构造一次的开销微乎其微；而存成成员反而要考虑"什么时候 sync""多线程谁在用"这些问题。**用到哪、建到哪、用完析构自动落盘**，是最省心也最不容易错的用法。

---

# 第 3 章 知识点预习（三）：五个绕不开的工程问题

选型定了、API 会了，还差一层：**设计决策**。这一章的五个问题，每一个都是真实项目里吵过架的。

## 3.1 路径选择：配置放哪？

候选答案有三个：

| 位置 | 例子 | 评价 |
|---|---|---|
| **程序所在目录** | `C:\Tools\LightLiveStreamer\config.ini` | "绿色软件"风格，整个文件夹拷走配置跟着走。但装在 `C:\Program Files` 下时**普通用户没写权限**，保存会失败 |
| **系统标准目录** | `%APPDATA%\<App>\`、macOS `~/Library/Preferences` | 跟随系统规范，多用户各自隔离，权限天然正确。Qt 里用 `QStandardPaths` 查询这类目录 |
| **注册表 / 系统原生存储** | HKCU 分支 | 路径问题**彻底消失**——系统帮你管理 |

> **专业名词**：**QStandardPaths** 是 Qt 提供的"标准目录查询器"，比如 `QStandardPaths::writableLocation(QStandardPaths::AppDataLocation)` 会返回当前平台上"该程序该存数据的标准位置"，Windows 上大概是 `C:\Users\<用户名>\AppData\Roaming\<应用名>`。手写 JSON/INI 方案时应该用它定位，而不是拍脑袋写死路径。

我们选了注册表方案，等于把"路径选择"这个难题**外包给了操作系统**。代价是：配置文件不再是一个能双击打开的文件——想看见它，得用 `regedit`。第 8 章动手环节我们就干这个。

## 3.2 写入时机：关闭时保存，还是改动即保存？

| 策略 | 做法 | 优点 | 缺点 |
|---|---|---|---|
| **关闭时保存** | 只在 `closeEvent` 里存一次 | 写得少，逻辑简单 | 程序崩溃 / 断电 → 这次的改动全丢 |
| **改动即保存** | 每个控件的 change 信号里都存 | 永不丢 | 写得太频繁；控件初始化时会触发一堆信号，容易写出一半的"半成品"配置 |
| **关键时机保存（我们的选择）** | 关窗口存一次 + 点开始推流存一次 | 崩溃也只丢"没推过流的改动"；推流成功的那份配置一定落了盘 | 要选好"关键时机"，属于设计判断 |

特别注意"点开始推流存一次"的深意：**用户点"开始推流"这个动作，本身就是对当前配置的一次"确认"**——他拿这份参数真的去推了。把这一刻的配置立刻落盘，哪怕推流中程序崩了、断电了，下次打开依然是这份"上次实战用过"的参数。这是一个很细腻的工程判断：**写入时机不只是技术问题，更是"哪份配置对用户最有价值"的业务判断。**

## 3.3 读取失败的兜底：宁可回默认，不能崩

读配置可能遇到的各种"坏情况"：

- 第一次运行，键根本不存在 → `value(key, 默认值)` 返回默认值；
- 键存在但值被改坏了（`"abc"`）→ `toInt()` 返回 0，再被控件的合法范围兜住（下面细讲）；
- 存的序号超出了当前选项数量（比如上次选的是 2 号摄像头，这次只插了一个）→ **代码里手动做了边界检查**，越界就不设置，保持界面默认值。

三层防线叠起来，保证一件事：**无论配置坏成什么样，程序都能以"出厂默认 + 能救回来的部分"正常启动**。配置是给人服务的，绝不能反过来让配置把程序绑架了。

## 3.4 默认值与版本兼容：新增字段时旧配置怎么办

假设下个版本我们给界面加了一个"音频码率"下拉框，`loadSettings` 里会多一行：

```cpp
audioKbpsCombo_->setCurrentIndex(...settings.value("audio/kbpsIndex", 2)...)
```

老用户的注册表里**没有** `audio/kbpsIndex` 这个键——怎么办？不用办。`value()` 的默认值参数天然解决了"**旧配置 + 新程序**"的前向兼容：老用户升级后，新字段自动拿默认值，老字段照常读回来。

反过来"**新配置 + 旧程序**"（用户装回旧版）也安全：旧程序压根不读那个键，多余的键躺在注册表里无害。

真正要小心的是第三种：**同一个键的含义变了**（比如 `video/source` 下版本想改成"0=摄像头 1=屏幕"，顺序对调）。这时老配置读出来就是错的。工业级做法是在配置里存一个版本号（`settings.setValue("meta/version", 2)`），读的时候按版本号做迁移。我们现在的键语义都很稳定，还没引入版本号——这属于"知道有这回事，暂时不为它付成本"的合理取舍，第 10 章思考题会让大家动手设计。

## 3.5 UI 与配置对象的双向绑定：三条数据线

把第 12 课和这一课连起来看，界面上那排控件其实连着三条数据线：

```
        启动时                    点"开始推流"时
  存储 ────────→ 界面控件 ────────────→ StreamConfig ──→ 推流管线
  (loadSettings)   ↑    (collectConfig，第 12 课)
                   │
                   └──── 界面控件 ──→ 存储   (saveSettings)
                        保存时机：推流时 / 关窗时
```

- **方向一（存储 → 界面）**：`loadSettings()`，本课新增。一个个 `setCurrentIndex` / `setValue` / `setPlainText`，把上次的值"灌"回控件。
- **方向二（界面 → 存储）**：`saveSettings()`，本课新增。`currentIndex()` / `value()` / `toPlainText()`，把控件的当前值"舀"出来写进 QSettings。
- **方向三（界面 → StreamConfig）**：`collectConfig()`，第 12 课已有，本课**一行没改**。

> 同学们注意一个漂亮的分工：**持久化层完全不碰 `StreamConfig`**。`StreamConfig` 是第 3 课定义的"内存里的配置契约"，它只服务于"一次推流"；而 QSettings 里存的是"界面的状态"。两者之间以界面控件为中转站。为什么不让 `StreamConfig` 直接序列化？因为两者关心的字段不完全重合——比如 `StreamConfig` 里有 `sampleRate = 48000` 这种"程序定死、界面上没得选"的字段，不需要持久化；而界面上的"下拉框当前序号"这种纯 UI 状态，`StreamConfig` 又不关心。**各存各的，界面做桥梁**，这个解耦值得学。

---

# 第 4 章 逐行教学（一）：声明、include 与调用点

从这一章开始逐行过 diff。先看三处"接线"改动。

## 4.1 `MainWindow.h`：两行声明

```diff
     StreamConfig collectConfig() const;
 
+    void loadSettings();
+    void saveSettings();
+
     QWidget* previewWidget_ = nullptr;
```

- 位置选得很讲究：紧跟在 `collectConfig()` 后面。**`collectConfig` 是"界面 → 内存配置"，`loadSettings`/`saveSettings` 是"存储 ↔ 界面"**，三个函数是同一个语义家族——都是"数据搬运工"。声明放在一起，读头文件的人一眼看到"这个类有三条数据搬运线"。
- 都是 `void` 无参、私有（在 `private:` 区域内）：纯内部实现细节，不对外暴露。
- 为什么不是 `const`？`loadSettings` 要调控件的 setter，改的就是 this 的成员；`saveSettings` 虽然只读控件，但保持和 `loadSettings` 对称、且 `QSettings` 写盘也不算逻辑上的 const，索性都不标。这是合理的小取舍。

## 4.2 `MainWindow.cpp`：一行 include

```diff
 #include <QPushButton>
+#include <QSettings>
 #include <QSize>
```

按字母序插在 `QPushButton` 和 `QSize` 之间——项目从第 1 课起就保持 include 按字母排序的习惯，diff 干干净净。头文件里只有声明、没用到 QSettings，所以 include 只加在 .cpp——**"用到才包含，且尽量只包含在实现文件里"**，这是控制编译依赖的基本功。

## 4.3 构造函数里的调用点：顺序是设计出来的

```diff
     refreshVideoDevices();
     refreshAudioDevices();
+    loadSettings();
     updateSourceFields();
     updateUiState();
```

`loadSettings()` 插在四个初始化步骤的正中间，这个位置不是随手放的，往前挪一格、往后挪一格都会出问题：

- **必须在 `refreshVideoDevices()` / `refreshAudioDevices()` 之后**：这两个函数负责把"摄像头列表""音频设备列表"填进下拉框。**下拉框里得先有选项，`setCurrentIndex(2)` 才有意义**——空下拉框里设序号，设了个寂寞。
- **必须在 `updateSourceFields()` 之前**：`updateSourceFields` 根据"视频来源是不是摄像头"决定设备行显不显示。先把上次的来源读回来，再根据它刷新显隐，界面第一次亮相就是对的，不会"先显示再隐藏"闪一下。
- **必须在 `updateUiState()` 之前**：`updateUiState` 统一把控件置为可编辑状态。参数先就位，状态再统一刷，次序井然。

一句话：**先填选项，再回填值，再按值刷新界面**。同学们以后写"恢复上次状态"的代码，都套这个顺序。

---

# 第 5 章 逐行教学（二）：`loadSettings()` 全文

读方向：存储 → 界面。先贴全文（diff 新增的第 240~272 行），再分段讲。

```cpp
void MainWindow::loadSettings()
{
    QSettings settings;

    const int videoSource = settings.value("video/source", 0).toInt();
    if (videoSource >= 0 && videoSource < videoSourceCombo_->count())
        videoSourceCombo_->setCurrentIndex(videoSource);

    const int cameraIndex = settings.value("video/cameraIndex", 0).toInt();
    if (cameraIndex >= 0 && cameraIndex < cameraCombo_->count())
        cameraCombo_->setCurrentIndex(cameraIndex);

    const QSize size(settings.value("video/width", 1280).toInt(),
                     settings.value("video/height", 720).toInt());
    const int resolutionIndex = resolutionCombo_->findData(size);
    if (resolutionIndex >= 0)
        resolutionCombo_->setCurrentIndex(resolutionIndex);

    fpsSpin_->setValue(settings.value("video/fps", 30).toInt());
    bitrateSpin_->setValue(settings.value("video/bitrate", 2500).toInt());

    const int audioSource = settings.value("audio/source", 0).toInt();
    if (audioSource >= 0 && audioSource < audioSourceCombo_->count()) {
        audioSourceCombo_->setCurrentIndex(audioSource);
    }

    const int audioDevice = settings.value("audio/device", -1).toInt();
    const int audioIndex = audioDeviceCombo_->findData(audioDevice);
    if (audioIndex >= 0)
        audioDeviceCombo_->setCurrentIndex(audioIndex);

    targetsEdit_->setPlainText(settings.value("targets/text").toString());
}
```

## 5.1 第一段：创建 QSettings

```cpp
    QSettings settings;
```

第 2 章讲过：默认构造，NativeFormat + UserScope，Windows 上指向 `HKCU\Software\LightLiveStreamer\LightLiveStreamer`。栈对象，函数结束析构。

## 5.2 第二段：视频来源——"带边界检查的恢复"模板

```cpp
    const int videoSource = settings.value("video/source", 0).toInt();
    if (videoSource >= 0 && videoSource < videoSourceCombo_->count())
        videoSourceCombo_->setCurrentIndex(videoSource);
```

三行，每一行一个防线：

1. `value("video/source", 0)`：键不存在就返回 `0`（"屏幕"，下拉框第一项）。**第一次运行的用户无感。**
2. `if (videoSource >= 0 && videoSource < videoSourceCombo_->count())`：**手动边界检查**。为什么不能直接 `setCurrentIndex(videoSource)`？因为存的是"上次退出时的序号"，而选项数量是"这次运行时的"——两者之间没有永恒担保。比如未来版本给来源下拉框加了第三个选项"窗口采集"，旧版本（只有两项）读到一个 `2` 就越界了。`QComboBox::setCurrentIndex` 收到越界值不会崩（它会忽略），但**依赖"它恰好不崩"和"明确检查"是两种工程态度**——后者让意图可读，也不依赖具体 Qt 版本的未文档化行为。
3. 检查通过才 `setCurrentIndex`；不通过就什么都不做，**保持 `buildUi()` 里设的默认值**。这就是第 3 章说的"越界兜底"。

这个"读 → 查边界 → 有才设"的三行模板，本函数里用了三次（视频来源、摄像头、音频来源），请同学们把它背下来，这是恢复下拉框状态的标准动作。

## 5.3 第三段：摄像头序号——同一个模板

```cpp
    const int cameraIndex = settings.value("video/cameraIndex", 0).toInt();
    if (cameraIndex >= 0 && cameraIndex < cameraCombo_->count())
        cameraCombo_->setCurrentIndex(cameraIndex);
```

和上一段一模一样的模板。但这里藏着一个**已知的局限**，老师要坦白：**存的是"序号"，不是"设备身份"**。上次"序号 1"对应 USB 摄像头，这次用户把内置摄像头禁用了，"序号 1"可能就对应另一个设备了，甚至序号列表整个重排。更稳的做法是存设备名字符串、恢复时 `findText` 匹配（第 12 课的 `refreshVideoDevices` 里就有 `findText(previous)` 的先例）。没这么做的原因也直白：设备名在 Windows 上可能重复（两个同型号 UVC 摄像头），而且当前代码追求的是"够用"。第 10 章思考题会追这个点。

## 5.4 第四段：分辨率——不是恢复序号，是恢复"值"

```cpp
    const QSize size(settings.value("video/width", 1280).toInt(),
                     settings.value("video/height", 720).toInt());
    const int resolutionIndex = resolutionCombo_->findData(size);
    if (resolutionIndex >= 0)
        resolutionCombo_->setCurrentIndex(resolutionIndex);
```

分辨率这段和前两个下拉框**思路不同**，值得细讲。

回看第 12 课 `buildUi()` 里分辨率下拉框是怎么建的：

```cpp
    resolutionCombo_->addItem("640 x 360", QSize(640, 360));
    resolutionCombo_->addItem("1280 x 720", QSize(1280, 720));
    resolutionCombo_->addItem("1920 x 1080", QSize(1920, 1080));
```

`addItem(显示文字, 数据)` 的第二个参数叫 **item data**——每个选项除了显示文字，还偷偷携带一个 QVariant 数据（这里是 `QSize`）。所以分辨率存的时候存的是**宽和高两个数**（见第 6 章），恢复时：

1. 读出宽高，拼成一个 `QSize`；
2. `findData(size)`：**在下拉框所有选项的 data 里找"哪个选项携带的数据等于这个 QSize"**，找到了返回它的序号，找不到返回 `-1`；
3. 找到了才 `setCurrentIndex`。

为什么不直接存序号（0/1/2）？因为**序号是"列表的偶然"，值才是"配置的本质"**。哪天产品要在中间插一个 "1600 x 900" 的档位，所有序号全变，存序号的旧配置全错位；而存宽高的配置，只要列表里还有 1280×720 这一项，`findData` 照样精确定位。**存值不存序号，是配置持久化里一条通用原则**——音频设备那段还会再用一次。

## 5.5 第五段：帧率与码率——QSpinBox 自带兜底

```cpp
    fpsSpin_->setValue(settings.value("video/fps", 30).toInt());
    bitrateSpin_->setValue(settings.value("video/bitrate", 2500).toInt());
```

这两行**没有手动边界检查**，为什么？因为 `QSpinBox` 自己有合法范围（第 12 课设过：fps 5~60，码率 300~20000），`setValue` 收到越界的值会**自动夹到范围内**（clamp）——注册表里被改成 `9999`，设进去就变成 `60`。

再叠上 `toInt()` 的兜底（值是 `"abc"` 就变 0，再被夹到 5），这两行形成了"`value` 默认值 → `toInt` 转换兜底 → `QSpinBox` 范围夹取"的三层防线。**控件自身有约束时，信任控件的约束**，代码就可以写得这么短。

注意默认值：`fps 30 / bitrate 2500 / 1280×720`。和第 3 课 `StreamConfig` 的默认（1080p、4500kbps）**不一样**！这不是笔误——`buildUi()` 里界面的出厂值就是 720p/2500，`loadSettings` 的默认值必须和界面出厂值**保持一致**，否则"第一次运行"和"配置丢了"会呈现出两种不同的界面。持久化层的默认值 ≠ 契约层的默认值，但持久化层的默认值必须等于 UI 层的初始值。这是个容易踩的小坑，记住了。

## 5.6 第六段：音频来源——模板第三次出现（带了个花括号）

```cpp
    const int audioSource = settings.value("audio/source", 0).toInt();
    if (audioSource >= 0 && audioSource < audioSourceCombo_->count()) {
        audioSourceCombo_->setCurrentIndex(audioSource);
    }
```

模板老三。细心的同学会发现：前三处 `if` 都没写花括号，唯独这里写了 `{ }`。这是 diff 里真实存在的小不一致——**讲稿不粉饰，指出来**：功能完全等价，纯粹是作者写到这里时手滑了一下。借它说一句：风格一致性靠 code review 和格式化工具（如 clang-format）保障，人总有手滑的时候。

还有一个**值得点名的隐式行为**：如果存的不是 0（上次选了"麦克风"），这行 `setCurrentIndex(1)` 会触发 `currentIndexChanged` 信号，而 `buildUi()` 里连过：

```cpp
    connect(audioSourceCombo_, &QComboBox::currentIndexChanged, this,
            [this](int) { refreshAudioDevices(); });
```

也就是**恢复"来源"会自动触发一次音频设备列表的刷新**——恰好在下一行恢复"设备"之前，把设备列表按正确的来源类型（系统声音 or 麦克风）重建好。这个"信号带动刷新"的联动是第 12 课就埋好的，`loadSettings` 搭了它的便车。如果当初构造函数里 `loadSettings()` 的位置放得不对（比如在 `refreshAudioDevices` 之前），这个联动就会乱套——现在理解 4.3 节那个顺序的分量了吧。

## 5.7 第七段：音频设备——再演一遍"存值不存序号"

```cpp
    const int audioDevice = settings.value("audio/device", -1).toInt();
    const int audioIndex = audioDeviceCombo_->findData(audioDevice);
    if (audioIndex >= 0)
        audioDeviceCombo_->setCurrentIndex(audioIndex);
```

回看第 12 课设备下拉框的填充：

```cpp
    audioDeviceCombo_->addItem("（系统默认）", -1);
    // 之后每个设备 addItem(名字, 序号)
```

第一项"（系统默认）"携带的 data 是 `-1`，真实设备携带 0、1、2……。所以这里默认值给 `-1`：**第一次运行的用户落在"系统默认"上**，最不容易错的选择。

`findData(audioDevice)` 按 data 精确匹配。如果上次选的设备这次不在了（设备被拔了），`findData` 返回 -1，`if` 不进，**下拉框保持默认的"（系统默认）"**——设备丢失自动降级到系统默认，优雅。

## 5.8 第八段：推流地址——唯一一个不给默认值的

```cpp
    targetsEdit_->setPlainText(settings.value("targets/text").toString());
```

全函数唯一一个 `value()` **没写第二参数**的调用。没写 = 默认返回一个无效 QVariant，`toString()` 得到空字符串，`setPlainText("")` = 清空文本框。语义完全正确：地址本来就没有"出厂值"，第一次运行就该是空的（界面上还有 `setPlaceholderText` 的灰色提示文字顶着）。

这也是一个小设计信号：**默认值给不给、给什么，本身就是对"新用户第一眼看到什么"的设计**。

## 5.9 `loadSettings` 全景复盘

把八段连起来，这个函数做了一张"键 → 控件 → 兜底策略"的对照表：

| 键 | 目标控件 | 恢复手段 | 兜底 |
|---|---|---|---|
| `video/source` | `videoSourceCombo_` | 按序号 | 边界检查，越界不动 |
| `video/cameraIndex` | `cameraCombo_` | 按序号 | 边界检查，越界不动 |
| `video/width` + `video/height` | `resolutionCombo_` | **findData(QSize)** | 找不到不动 |
| `video/fps` | `fpsSpin_` | setValue | toInt 兜底 + SpinBox 夹取 |
| `video/bitrate` | `bitrateSpin_` | setValue | 同上 |
| `audio/source` | `audioSourceCombo_` | 按序号 | 边界检查，越界不动 |
| `audio/device` | `audioDeviceCombo_` | **findData(int)** | 找不到 → 停在"系统默认" |
| `targets/text` | `targetsEdit_` | setPlainText | 缺省即空 |

读方向的 33 行，行行的潜台词都是同一句话：**"配置可能有，可能没有，可能有错——但界面必须体面地站起来。"**

---

# 第 6 章 逐行教学（三）：`saveSettings()` 与两处保存时机

## 6.1 `saveSettings()` 全文

写方向：界面 → 存储。

```cpp
void MainWindow::saveSettings()
{
    QSettings settings;

    settings.setValue("video/source", videoSourceCombo_->currentIndex());

    settings.setValue("video/cameraIndex", cameraCombo_->currentIndex());

    const QSize size = resolutionCombo_->currentData().toSize();
    settings.setValue("video/width", size.width());
    settings.setValue("video/height", size.height());

    settings.setValue("video/fps", fpsSpin_->value());
    settings.setValue("video/bitrate", bitrateSpin_->value());

    settings.setValue("audio/source", audioSourceCombo_->currentIndex());

    settings.setValue("audio/device", audioDeviceCombo_->currentData().toInt());

    settings.setValue("targets/text", targetsEdit_->toPlainText());
}
```

## 6.2 逐行讲：和 loadSettings 一一镜像

这个函数是 `loadSettings` 的**镜像**——读方向用过的每一招，写方向都有对应：

```cpp
    QSettings settings;
```

同样的默认构造。函数结束析构时统一落盘（第 2.5 节）。

```cpp
    settings.setValue("video/source", videoSourceCombo_->currentIndex());
```

视频来源：存**当前序号**。读方向按序号恢复 + 边界检查，写方向就直接写序号，镜像。

```cpp
    settings.setValue("video/cameraIndex", cameraCombo_->currentIndex());
```

摄像头序号，同上。

```cpp
    const QSize size = resolutionCombo_->currentData().toSize();
    settings.setValue("video/width", size.width());
    settings.setValue("video/height", size.height());
```

分辨率：**不存序号，存值**。`currentData()` 取当前选项携带的那个 QSize（第 12 课 `addItem` 塞进去的），拆成宽、高两个键写出去。为什么拆成两个键而不存一个 "1280x720" 字符串？因为读方向要用它们拼 `QSize` 做 `findData`，**整数键最不容易出格式歧义**（"1280x720" 和 "1280×720" 和 "1280*720" 字符串匹配就是灾难）。

```cpp
    settings.setValue("video/fps", fpsSpin_->value());
    settings.setValue("video/bitrate", bitrateSpin_->value());
```

帧率、码率：SpinBox 的值本身就保证在合法范围内（控件约束），**取出来直接存，天然干净**。

```cpp
    settings.setValue("audio/source", audioSourceCombo_->currentIndex());
```

音频来源，序号。

```cpp
    settings.setValue("audio/device", audioDeviceCombo_->currentData().toInt());
```

音频设备：存 **data**（-1 = 系统默认，0/1/2… = 设备序号），和读方向的 `findData` 镜像。

```cpp
    settings.setValue("targets/text", targetsEdit_->toPlainText());
```

推流地址：**整个多行文本原样存**。没有逐行拆成结构化数据——为什么？同学们想想：`collectConfig()` 已经负责"把文本解析成 targets 列表"了，持久化层如果再存一份解析后的结构，就产生了两份真相（文本一份、结构一份），以后改了解析规则两边都要动。**存原始文本，解析永远只有一份**，这又是"单一真相来源"（Single Source of Truth）的老朋友。

## 6.3 写方向为什么不需要任何检查？

对比一下：读方向到处是 `if`（边界检查、findData 判 -1），写方向一个 `if` 都没有。原因一句话：**界面上能呈现出来的值，必然是合法值**——非法值在进来的路上（`loadSettings`、用户操作、控件约束）早就被拦住了。写方向只是"拍照"，拍照不需要美颜。

这就是双向绑定的不对称之美：**读要防御，写要直接。**

## 6.4 保存时机之一：`startStreaming()` 开头

```diff
 void MainWindow::startStreaming()
 {
     if (starting_.load() || running_.load())
         return;
 
+    saveSettings();
+
     const StreamConfig config = collectConfig();
```

位置在函数最开头、重复启动守卫之后、`collectConfig()` 之前。两个细节：

- 在守卫**之后**：重复点击、推流中误触，不会触发多余的写盘。
- 在 `collectConfig()` **之前**：先把界面状态存了，再做"界面 → StreamConfig"的收集。顺序上两者无所谓先后，但把"存"放前面有个心态上的好处：**哪怕 `collectConfig()` 之后校验失败（没填地址直接 return），配置也已经存了**——用户填了一半的地址不会因为他忘了填完整而丢掉。

业务含义第 3.2 节讲过：**"开始推流"是用户对配置的确认动作，这一刻的配置最值得记住。**

## 6.5 保存时机之二：`closeEvent()` 收尾

```diff
 void MainWindow::closeEvent(QCloseEvent* event)
 {
     statsTimer_->stop();
 
     if (running_.exchange(false) || starting_.exchange(false)) {
         std::lock_guard<std::mutex> lock(pipelineMutex_);
         pipeline_.stop();
     }
 
     if (worker_.joinable())
         worker_.join();
 
+    saveSettings();
+
     QMainWindow::closeEvent(event);
 }
```

`saveSettings()` 插在"停推流、等线程结束"之后、交给基类 `closeEvent` 之前。这个位置也是设计过的：

- **先停管线再存配置**：推流停止、工作线程 join 完毕，此刻界面状态稳定（`updateUiState` 早把控件恢复可编辑了），存的是一份"安静的快照"。
- **在 `QMainWindow::closeEvent(event)` 之前**：基类调用是"真正接受关闭"的动作，所有清理必须赶在它之前完成。
- 为什么关窗要再存一次，推流时不是存过了吗？——覆盖"**改了参数但没推流就关窗**"的场景。比如用户下播后顺手把码率从 4500 调回 2500（明天换个网络用），没点开始就直接关窗——这一存，把这个改动也留下了。

两个保存时机合起来，就是第 3.2 节的"关键时机保存"策略：**推流时存"实战配置"，关窗时存"最终状态"**。

---

# 第 7 章 逐行教学（四）：打包三件套

主线讲完了，讲配套——这次提交顺手把"发布"也做了。配置持久化让软件"好用"，打包让软件"能到用户手里"，都是交付。

## 7.1 `.gitignore`：一行

```diff
 build/
+dist/
 out/
```

`dist/` 是即将生成的"发布目录"（distribution 的缩写），里面是拷贝出来的成品。它和 `build/` 一样是构建产物，**不进版本库**——版本库存源代码，产物随时能重新生成。一句话原则：**能再生的东西不进 git**。

## 7.2 `src/app/CMakeLists.txt`：install 规则

```cmake
install(TARGETS LightLiveStreamer RUNTIME DESTINATION .)
```

告诉 CMake：执行 `cmake --install` 时，把 `LightLiveStreamer.exe` 拷到安装前缀目录（`RUNTIME DESTINATION .` 的 `.` = 前缀根目录）。

```cmake
install(CODE "
  file(GLOB _root_dlls \"${CMAKE_BINARY_DIR}/*.dll\")
  file(COPY \${_root_dlls} DESTINATION \"\${CMAKE_INSTALL_PREFIX}\")
")
```

`install(CODE ...)` 内嵌一段 CMake 脚本，在安装时执行：把构建目录根下所有 `.dll` 拷进安装目录。这些 DLL 哪来的？是构建时（前几课的 CMake 配置里）从 Qt/FFmpeg 拷到 exe 旁边、保证开发时能跑的——安装时原样再拷一份。

注意 `\${_root_dlls}` 和 `\${CMAKE_INSTALL_PREFIX}` 的**转义**：因为这段脚本被包在双引号字符串里，如果不转义，`${...}` 会在**配置期**就被展开（那时 `CMAKE_INSTALL_PREFIX` 还是配置期的值）；转义后，变量留到**安装期**才展开，拿到的是 `cmake --install --prefix` 实际传入的路径。这是 CMake 字符串嵌套的经典坑，作者处理得很标准。

```cmake
foreach(_plugin_dir platforms styles tls imageformats iconengines generic networkinformation)
  if(EXISTS "${CMAKE_BINARY_DIR}/${_plugin_dir}")
    install(DIRECTORY "${CMAKE_BINARY_DIR}/${_plugin_dir}"
            DESTINATION . FILES_MATCHING PATTERN "*.dll")
  endif()
endforeach()
```

Qt 的**插件目录**整目录拷贝。这里要补一个背景知识：

> **专业名词**：**Qt 插件**（Qt plugins）是 Qt 的"运行时按需加载"机制。Qt 的核心库不含具体平台代码，窗口怎么画到屏幕上由 `platforms/qwindows.dll` 这样的插件在程序启动时动态加载；`styles` 是界面风格、`tls` 是加密传输（https/wss 靠它）、`imageformats` 是图片解码、`iconengines` 是图标渲染……**少了 `platforms/qwindows.dll`，程序双击直接报"Failed to load platform plugin windows"然后退出**——这是 Qt 部署的第一大经典事故。

`foreach` 循环列了七个目录、`if(EXISTS ...)` 先判存在再装——**有就拷，没有就跳过**，不会让安装因为某个可选插件缺失而失败。`FILES_MATCHING PATTERN "*.dll"` 表示目录里只拷 dll，pdb 调试符号之类的杂文件不带进发布包，包更小。

> 提一句：Qt 官方有个自动化工具 **windeployqt**（给它 exe 路径，它自动分析依赖并把需要的 DLL 和插件拷齐）。本项目没用它，而是"构建期已把依赖拷到 exe 旁边 + install 时原样搬运"——思路等价，好处是构建产物本身随时可运行可调试，install 只是纯粹的"归置"。

## 7.3 `package.bat`：一键打包脚本

新增文件，29 行，逐段讲：

```bat
@echo off
setlocal
set "ROOT=%~dp0"
```

- `@echo off`：不回显每条命令，输出干净。
- `setlocal`：本脚本里设的环境变量脚本结束自动失效，不污染调用者的命令行。
- `%~dp0`：**脚本自身所在目录**（drive + path）。不管用户从哪个目录调用这个 bat，ROOT 都指向项目根。这是 bat 脚本的标配开头。

```bat
if not exist "%ROOT%build\LightLiveStreamer.exe" (
  echo [error] build\LightLiveStreamer.exe not found. Run build.bat first.
  goto :fail
)
```

**前置检查**：没编译就想打包？直接报错并提示"先跑 build.bat"。打包脚本的第一个职责是**把低级错误挡在门外，并给出可操作的提示**。

```bat
echo [1/2] Cleaning dist...
if exist "%ROOT%dist" rmdir /s /q "%ROOT%dist"
```

先删旧 `dist/`。**每次打包从干净的目录开始**，防止上次打包残留的旧 DLL 混进新包（"这个 bug 修了怎么用户那儿还在"的悬案多半是脏目录造成的）。

```bat
echo [2/2] Installing runtime...
cmake --install "%ROOT%build" --prefix "%ROOT%dist"
if errorlevel 1 goto :fail
```

核心就一条命令：`cmake --install`，把 7.2 节那些 install 规则执行一遍，输出到 `dist/`。`if errorlevel 1` 检查上条命令退出码非 0（失败）就跳到 `:fail`。

```bat
echo.
echo [done] %ROOT%dist
echo.
echo Contents:
dir /b "%ROOT%dist"
echo.
echo Copy the whole dist folder to another machine and run LightLiveStreamer.exe
exit /b 0
```

成功路径：打印产物清单（`dir /b` 简洁列出），并给出使用说明——**整个 dist 目录拷走、双击 exe**。最后 `exit /b 0` 返回成功码。

```bat
:fail
echo.
echo [failed] Packaging did not finish.
exit /b 1
```

失败路径：明确报错、返回失败码 `1`。批处理里"成功 0 / 失败非 0"的退出码约定不能省——以后接 CI 自动打包时，CI 就是靠退出码判断成败的。

整个脚本的结构是标准的"**检查 → 清理 → 执行 → 报告**"四段式，同学们以后写任何自动化脚本都可以套这个骨架。

## 7.4 配置与打包的化学反应

把这一章和主线连起来想，有个很妙的细节：**配置在注册表（HKCU），程序在 dist 目录**——这意味着 dist 目录是**完全自包含、无状态**的。用户可以把 dist 拷到 U 盘里换台电脑跑，配置不会跟着走（在各自机器的注册表里），也不会互相污染。想做成"配置跟着 U 盘走"的便携版（portable），把 QSettings 换成 IniFormat 并指向程序目录即可——思考题里我们会让大家设计这个改造。

---

# 第 8 章 动手环节：亲眼看见"配置被记住"

## 8.1 实验一：基本流程——改、关、开、看

1. 编译运行（`build.bat` 后跑 `build\LightLiveStreamer.exe`）。
2. 把参数改成一组**有辨识度**的值，比如：
   - 视频来源：摄像头
   - 分辨率：1920 x 1080
   - 帧率：60（拉到顶，好认）
   - 码率：4500
   - 推流地址随便贴一行：`test=rtmp://example.com/live/key123`
3. **直接关窗口**（不用点开始推流——正好验证 `closeEvent` 那条保存路径）。
4. 重新打开程序。
5. **预期**：上面每一项都原样回来。回来的那一刻，这节课的目标就达成了。

## 8.2 实验二：去注册表里"亲眼看见"配置

光看见界面恢复还不够，我们要看见**存储本身**。

1. `Win + R` 打开"运行"，输入 `regedit`，回车，打开注册表编辑器。
2. 在地址栏粘贴（或逐级点开）：

   ```
   计算机\HKEY_CURRENT_USER\Software\LightLiveStreamer\LightLiveStreamer
   ```

3. 你应该看到三个子键（子文件夹）：`audio`、`targets`、`video`。点开 `video`，右侧列出：

   | 名称 | 类型 | 数据 |
   |---|---|---|
   | bitrate | REG_SZ | 4500 |
   | cameraIndex | REG_SZ | 0 |
   | fps | REG_SZ | 60 |
   | height | REG_SZ | 1080 |
   | source | REG_SZ | 1 |
   | width | REG_SZ | 1920 |

4. 点开 `targets`，看到 `text` 里就是你贴的那行 rtmp 地址。

**思考题现场版**：为什么值全是 REG_SZ（字符串）类型，明明 fps 是个整数？——因为 QSettings 在注册表里的默认表示就是字符串，读出时再 `toInt()`。类型信息是靠"读写双方约定"维护的，这正是 2.4 节那套 QVariant 转换存在的意义。

## 8.3 实验三：改坏它！——健壮性破坏测试

现在当老师布置的"坏人"，直接在注册表里动手脚，每改一处，重启程序观察一次：

| 破坏操作 | 预期结果 | 验证的是哪道防线 |
|---|---|---|
| 把 `video/fps` 的数据改成 `9999` | 程序正常启动，帧率显示 **60** | QSpinBox 范围夹取 |
| 把 `video/fps` 改成 `abc` | 程序正常启动，帧率显示 **5**（abc→toInt→0→夹到下限） | toInt 转换兜底 + 夹取 |
| 把 `video/source` 改成 `7` | 程序正常启动，来源停在默认"屏幕" | 手动边界检查 |
| 把 `video/width` 改成 `1234` | 分辨率停在默认 1280x720（1234x1080 不在列表里，findData 返回 -1） | findData 找不到不动 |
| 把 `audio/device` 改成 `99` | 音频设备停在"（系统默认）" | findData 降级到默认项 |
| **右键删掉整个 `LightLiveStreamer` 键** | 程序正常启动，全部回到出厂值——和第一次运行一模一样 | `value(key, 默认值)` 全程兜底 |

做完这一圈，同学们对第 3.3 节"读取失败的兜底"会有肌肉记忆级的理解：**这个程序的读配置路径，想让它崩，你得换个思路。**

## 8.4 实验四：验证"推流时保存"这个时机

1. 打开程序，把码率改成 5000，**不点任何按钮，用任务管理器直接"结束任务"强杀进程**（模拟崩溃）。
2. 重启程序 → 码率**没**记住（closeEvent 没机会跑，saveSettings 没执行）。
3. 再改成 5000，这次**点一下"开始推流"**（哪怕因为地址无效推不起来），然后用任务管理器强杀。
4. 重启程序 → 码率**记住了**。

这个实验把 3.2 节和 6.4 节的设计意图变成了可触摸的事实：**推流时那一次保存，就是为"崩溃"买的保险。**

## 8.5 实验五：打个包送人

1. 在项目根目录跑 `package.bat`。
2. 看输出：`[1/2] Cleaning dist...` → `[2/2] Installing runtime...` → `[done]` + 文件清单。
3. 打开 `dist/` 目录，确认有 `LightLiveStreamer.exe`、`platforms/qwindows.dll`、若干 Qt6*.dll 和 ffmpeg 的 dll。
4. 把 dist 整个拷到一台**没装 Qt、没装 FFmpeg** 的机器（或虚拟机），双击 exe——能起来，打包成功。
5. 在那台机器上改参数、关窗、重开——配置照样记住（配置在那台机器自己的注册表里）。

---

# 第 9 章 排错手册

## 9.1 配置根本没存上 / 下次打开全丢

按顺序查：

1. **查 `main.cpp` 的三行还在不在**：`setApplicationName` / `setOrganizationName` 被删或改名，QSettings 会写到另一个位置，表现为"配置全丢"。**改应用名 = 全体用户配置清零**，发布后要慎重。
2. **查注册表路径有没有走错**：打开 regedit 确认 `HKCU\Software\<组织名>\<应用名>`。公司名写错一个字母，配置就存到平行世界去了。
3. **查是不是以管理员/其他用户身份跑的程序**：HKCU 是"当前用户"的分支。开发时用自己的账户调试存了一份，部署时用别的方式（比如某些以 SYSTEM 身份运行的启动器）拉起程序，读的是另一个用户的 HKCU，自然"没存上"。
4. **查杀进程方式**：任务管理器强杀 = closeEvent 不执行 = 只丢没推过流的改动。这是设计行为，不是 bug（实验四已经验证过边界）。

## 9.2 权限问题

- **HKCU 不需要管理员权限**，正常桌面程序不会有权限问题。
- 真遇到"写不进去"的罕见场景（公司组策略锁了注册表某些分支），QSettings 的写入会静默失败——这是 QSettings 的一个短板：**它不报错**。排查手段是 `settings.sync()` 后查 `settings.status()`（`QSettings::NoError` / `AccessError` / `FormatError`）。我们的代码没查——项目规模下可以接受，写大项目时建议至少在 debug 构建里断言一下。
- 如果哪天改成 SystemScope（HKLM）或 IniFormat 存到 `Program Files` 下的程序目录，权限问题就会真的找上门——**那两条路都需要管理员**。这也是为什么 UserScope + 原生格式是省心之选。

## 9.3 中文乱码

- **注册表路径下不存在乱码问题**：Windows 注册表字符串是 UTF-16，Qt 内部 QString 也是 UTF-16，推流地址里的中文平台名（如 `哔哩哔哩=rtmp://...`）存取全程无损。
- **改成 IniFormat 才要操心编码**：Qt 6 的 ini 默认按 UTF-8 读写。如果用户用记事本手改 ini 又"另存为 ANSI"，中文就花了——程序读回来地址里的中文平台名变乱码。规避办法：文档里写明"配置文件请用 UTF-8 编辑器修改"，或者干脆教育用户别手改。
- **顺带提醒**：`targets/text` 里 rtmp 地址本身都是 ASCII，真正可能带中文的是 `平台名=` 前缀。乱码只影响日志里显示的平台名，不影响推流——伤害可控，但影响观瞻。

## 9.4 配置损坏

- 注册表几乎不会"文件损坏"（系统管理，有事务保护），要坏多半是**人为改坏**——而 8.3 的破坏实验已经证明：改坏了程序也站得住。
- 真要做到"损坏自检"的级别，工业界做法是：读到的值过一遍业务校验（比如 fps ∈ {24,25,30,50,60} 白名单），不合法就回默认值并记日志。我们靠"边界检查 + findData + SpinBox 夹取"已经覆盖了现实中最常见的坏法，更深度的校验属于"知道有这回事"。
- 如果是 IniFormat/JSON 方案，"文件半个字被截断""编码错乱"这类物理损坏才真实存在，那时需要"解析失败 → 备份坏文件 → 用默认值启动"的完整预案。选 QSettings 原生格式，等于把这些预案全省了——**方案的简单性本身消灭了整类故障**。

## 9.5 下拉框越界 / 设备对不上号

- **现象**：上次选的 2 号摄像头，这次开机变成别的设备或"找不到摄像头"。
- **原因**：存的是序号不是设备身份（5.3 节讲过的已知局限）。
- **缓解**：程序启动时本来就会重新枚举设备，`loadSettings` 的边界检查保证至少不越界、不崩；对不上号属于"配置语义漂移"，不是代码 bug。
- **根治方向**：存设备名 + `findText` 匹配，匹配不上回默认。留给思考题。

## 9.6 打包相关

- **双击 exe 报 "Failed to load platform plugin windows"**：`dist/platforms/qwindows.dll` 缺失。查 CMakeLists 的 foreach 里 `platforms` 一项、查构建目录里 `platforms/` 是否真的存在。
- **报缺 `Qt6Widgets.dll` / `avcodec-*.dll`**：构建目录根下的 dll 没拷全，查 `install(CODE ...)` 那段。
- **https/wss 地址连不上**：`tls/` 插件没带上。
- **dist 里有旧文件混着**：说明没走 `package.bat`（它先删 dist），手工 install 到旧目录了。

---

# 第 10 章 小结、速查表、局限与思考题

## 10.1 本课必记知识点（10 条）

1. **配置持久化 = 让数据活得比进程久**。界面参数每次重填是体验的最后一公里，这节课用 63 行核心代码修通了它。
2. **方案选型按"够用原则"**：十来个扁平键值对 → QSettings 原生格式，不为用 JSON 而用 JSON。
3. **QSettings 默认构造** = NativeFormat + UserScope，Windows 上落在 `HKCU\Software\<组织名>\<应用名>`，身份来自 `main.cpp` 的 `setApplicationName` / `setOrganizationName`。
4. **`value(key, 默认值)` 是兜底第一防线**：键不存在就回默认，读配置在代码层面"不可能失败"；`toInt()` 转坏值回 0 是第二道。
5. **读要防御，写要直接**：读方向到处是边界检查和 findData 判空，写方向一个 if 都没有——界面上能呈现的值必然合法。
6. **存值不存序号**：分辨率存宽高、音频设备存 data，用 `findData` 恢复——序号是列表的偶然，值是配置的本质。
7. **写入时机是业务判断**：推流时存"实战确认过的配置"（防崩溃丢失），关窗时存"最终状态"。
8. **初始化顺序有讲究**：先填选项（refresh*Devices），再回填值（loadSettings），再刷新界面（updateSourceFields/updateUiState）。
9. **三层数据线的解耦**：存储 ↔ 界面（本课）与 界面 → StreamConfig（第 12 课）各管各的，界面做桥梁，持久化层不碰 StreamConfig。
10. **打包四段式**：检查 → 清理 → `cmake --install` → 报告；Qt 插件（尤其 `platforms/qwindows.dll`）必须随行。

## 10.2 QSettings 速查表

| 需求 | 代码 |
|---|---|
| 读整数（带默认） | `settings.value("video/fps", 30).toInt()` |
| 读字符串（默认空） | `settings.value("targets/text").toString()` |
| 写值 | `settings.setValue("video/fps", 30)` |
| 立刻落盘（一般用不到，析构自动） | `settings.sync()` |
| 查写入状态 | `settings.status()` |
| 改用 ini 文件 | `QSettings s(path, QSettings::IniFormat);` 或 `QSettings::setDefaultFormat(QSettings::IniFormat)` |
| 恢复下拉框（按序号） | 读 → 边界检查 → `setCurrentIndex` |
| 恢复下拉框（按值） | 读 → `findData(v)` → 非 -1 才 `setCurrentIndex` |
| 标准数据目录（手写文件方案用） | `QStandardPaths::writableLocation(QStandardPaths::AppDataLocation)` |

## 10.3 坦白局限（4 条）

1. **摄像头存序号不存身份**：设备增删后序号漂移，可能"记住的是错的设备"（5.3 节）。
2. **没有配置版本号**：未来某个键语义变了，老配置会读错；届时需引入 `meta/version` 做迁移（3.4 节）。
3. **写入失败静默**：没查 `status()`，极端环境（组策略锁注册表）下存不上也无感知（9.2 节）。
4. **配置不随程序走**：注册表方案下 dist 拷到别的机器是"全新出厂"；想要便携版需改 IniFormat + 程序目录（7.4 节）。

四条都是"知道有这回事、现阶段不付成本"的清醒取舍——**能把局限说清楚的代码，才是被真正理解了的代码。**

## 10.4 思考题（8 题，附参考答案）

**Q1**：为什么 `loadSettings()` 里视频/音频来源要做 `if (index >= 0 && index < count())` 检查，而帧率、码率不需要？

> **答**：下拉框的序号合法性取决于"选项数量"这个运行期变量，控件不会替你把关语义，必须手动查；`QSpinBox` 自带 range 约束，`setValue` 自动夹取，控件本身就是防线。**控件有约束就信任控件，没有就自己补上。**

**Q2**：分辨率为什么存宽高两个数、`findData` 恢复，而不是直接存下拉框序号？

> **答**：序号依赖"选项的排列"这个偶然因素——将来在中间插入新档位，所有旧配置的序号全错位。宽高是配置的本质值，只要列表里还有这项就能精确恢复。通用原则：**存值不存序号。**

**Q3**：`saveSettings()` 为什么在 `startStreaming()` 和 `closeEvent()` 各调一次？只留一处行不行，各有什么后果？

> **答**：只留 `closeEvent`：推流中崩溃/断电会丢掉"本次实战配置"——而这份恰恰是用户最想要的。只留 `startStreaming`：用户改完参数没推流直接关窗，改动丢失。两处各覆盖一种丢失场景，合起来形成"实战配置 + 最终状态"的双保险。

**Q4**：`targets/text` 为什么存原始多行文本，而不是解析成结构化的目标列表再存？

> **答**：解析逻辑在 `collectConfig()` 里只有一份；若持久化层再存结构化副本，就产生两份真相，将来改解析规则要同步两处。存原始文本保持**单一真相来源**。

**Q5**：假设下版本要新增"音频码率"配置项，老用户升级后会怎样？需要写迁移代码吗？

> **答**：不需要。老注册表里没有新键，`value("audio/kbpsIndex", 默认值)` 自动回默认值，老键照常读取——QSettings 的默认值机制天然完成前向兼容。只有"同一键语义改变"时才需要版本号 + 迁移。

**Q6**：把本课方案改造成"便携版"（配置存程序目录的 ini 文件，跟着 U 盘走），要动哪几处？会引入什么新风险？

> **答**：把 `QSettings settings;` 换成 `QSettings settings(程序目录 + "/config.ini", QSettings::IniFormat)`（程序目录用 `QCoreApplication::applicationDirPath()` 取）。新风险：① 程序装在 `Program Files` 时普通用户无写权限，保存静默失败（需查 `status()`）；② ini 文本文件面临编码（中文 ANSI 乱码）与物理损坏问题，需要"解析失败回默认"的预案；③ 多实例同时写要考虑冲突。

**Q7**：第 5.3 节说摄像头"存序号"是个局限。请设计一个"存设备名"的改造，并指出它自己的坑。

> **答**：保存时 `settings.setValue("video/cameraName", cameraCombo_->currentText())`，恢复时 `findText` 匹配，找不到回默认 0。坑：① 两台同型号 UVC 摄像头设备名相同，无法区分（可退化为序号行为）；② 设备名含特殊字符/中文时的读写一致性（注册表 UTF-16 无碍，ini 要小心编码）；③ 用户给设备改名（少见）后匹配失败——但失败只是回默认，可接受。

**Q8（综合）**：产品提需求："记住窗口上次的大小和位置"。请仿照本课给出完整设计：存什么键、什么时候存、什么时候读、有哪些坑。

> **答**：存 `ui/geometry`（`saveGeometry()` 返回的 QByteArray 整体存）或拆存 `ui/x/y/w/h`。读取时机：构造函数 `buildUi()` 之后；保存时机：`closeEvent()`（仿照本课）。坑：① 上次窗口在第二块显示器上，这次显示器没接——恢复后窗口"在屏幕外"，必须做"与可用屏幕区域求交"的校验，交集为空就回默认居中；② 最大化状态要单独存（`isMaximized`），否则恢复的是最大化前的普通尺寸；③ 高分屏缩放（DPI）变化时像素坐标含义不同，QSettings 存的像素值可能偏移。**几何恢复比参数恢复多一类"环境失效"问题**——和摄像头序号漂移是同一族病。

## 10.5 下节课预告

到这里，我们的 LightLiveStreamer 已经是一个"功能完整 + 体验闭环 + 可以打包送人"的桌面软件了。回看这十五课：从空目录到能推流，从能推流到好用，从好用到可交付。**但"可交付"还差最后一块拼图**——用户拿到 dist 后出问题了怎么办？日志只显示在窗口里，关了就没了。下一课我们给程序装上"黑匣子"：把运行日志持久化到文件，让用户能拿着日志来找你，而不是拿着"它不行"三个字来找你。

---

## 附：本讲「五章对照索引」

> **用法说明**：这张表把本讲的全部内容压缩成**五个维度**的对照，方便你复习、检索、以及跟前几课建立联系。横向五行是本讲的**五个部分**，纵向五列是**五个维度**。

| 本讲部分 | 覆盖章节 | 核心知识点 | diff 中的文件与行号 | 怎么验证 / 上哪去练 |
|---|---|---|---|---|
| **① 预习：概念与原理** | 第 1~3 章 | 持久化定义；QSettings/INI/JSON/注册表四方案对比；文本 vs 二进制与序列化；QSettings 格式与作用域、组织名/应用名、注册表路径、键的分组、`value(key, 默认)` 三层兜底、QVariant、RAII 析构落盘；路径选择（QStandardPaths）、写入时机、版本兼容、双向绑定三条数据线 | 概念章，对应实现见 `main.cpp` 的 `setApplicationName/setOrganizationName` 三行 | **实验二**（regedit 亲眼看见配置树）、思考题 Q1/Q2 |
| **② 实现：代码本体** | 第 4~6 章 | 头文件两行声明的位置讲究；构造函数初始化顺序（填选项→回填值→刷界面）；`loadSettings` 八段（三种恢复模板：序号+边界检查 / findData 按值 / SpinBox 信任夹取）；`saveSettings` 与读方向的镜像对称；"读要防御、写要直接"；两处保存时机 | `MainWindow.h` +2 行声明；`MainWindow.cpp`：`#include <QSettings>`、`loadSettings()`（新增 33 行）、`saveSettings()`（新增 19 行）、`startStreaming()` 与 `closeEvent()` 各 +1 处调用 | **实验一**（改关开）、**实验四**（强杀验证推流时保存）、思考题 Q3/Q4 |
| **③ 配套：打包发布** | 第 7 章 | `.gitignore` 的"能再生不进库"；`install(TARGETS)`、`install(CODE)` 变量转义、Qt 插件目录拷贝（platforms 是命门）；`package.bat` 四段式（检查→清理→安装→报告）与退出码约定 | `.gitignore` +1 行；`src/app/CMakeLists.txt` +16 行；`package.bat` 新增 29 行（全文） | **实验五**（打包拷到干净机器跑）、9.6 节排错 |
| **④ 实践：动手与排错** | 第 8~9 章 | 五个实验（基本流程 / 注册表观察 / 六项破坏测试 / 强杀验证 / 跨机打包）；排错六类（没存上 / 权限 / 中文乱码 / 配置损坏 / 设备漂移 / 打包缺件） | 不新增代码，全部是对 ② 部分行为的观察与攻击 | 按 8.1~8.5 逐个做；出问题按 9.1~9.6 逐条排查 |
| **⑤ 沉淀：总结与延伸** | 第 10 章 | 10 条必记知识点；QSettings 速查表；4 条坦白局限（序号漂移 / 无版本号 / 写入静默 / 不便携）；8 道思考题（含便携版改造、设备名方案、窗口几何综合题） | 对全讲 185 行 diff 的回顾与批判 | 做完 8 道思考题（尤其 Q6/Q8 综合题） |

### 附：本讲与前面几课的衔接点（复习时对照）

| 本讲用到的东西 | 在哪一课学的 | 本讲怎么用的 |
|---|---|---|
| `StreamConfig` 与"默认值即推荐配置" | **第 3 课**（`a4453bb1`） | §3.5：持久化层不碰 StreamConfig、以界面为桥；§5.5：持久化默认值必须对齐 UI 出厂值而非契约默认值 |
| 界面控件与 `collectConfig()` | **第 12 课**（`3b22c69`） | §4~6：所有键的读写对象都是第 12 课建的那排控件；`collectConfig` 一行未改 |
| `addItem(文字, data)` 的 item data 机制 | **第 12 课** | §5.4/5.7/6.2：`findData`/`currentData` 是"存值不存序号"的技术基础 |
| `closeEvent` 停管线流程 | **第 14 课**（`c2b8021`） | §6.5：`saveSettings()` 插在"停管线、join 线程"之后、交基类之前 |
| RAII（析构自动清理） | 第 10 课等反复使用 | §2.5：QSettings 栈对象析构落盘，"想忘都忘不掉" |
| 构建目录里 Qt/FFmpeg DLL 的归置 | **第 1 课**（`20b267a1`） | §7.2：install 规则把"构建期已拷到 exe 旁"的依赖原样搬进 dist |

### 附：本讲两个"可以立刻动手的改进练习"

| # | 位置 | 现状 | 改成 |
|---|---|---|---|
| 1 | `MainWindow.cpp` `loadSettings()` 摄像头段 | 存/恢复设备**序号**，设备增删后可能对错号 | 改存 `currentText()` 设备名 + `findText` 恢复（思考题 Q7 的落地） |
| 2 | `MainWindow.cpp` 音频来源段 | 前三处 `if` 无花括号、此处有，风格不一（5.6 节） | 统一为无花括号（或全项目接入 clang-format，一次性格式化） |

**做完第 1 项之后，跑一遍 8.1~8.3 的实验，确认行为不退化。**

---

> **老师最后的话**：这一课的代码量是整个项目最小的一次，但请记住它的分量——**前十四课让程序"能跑"，这一课让程序"记得"**。一个会记得用户的软件，才第一次有了"产品"的样子。工程里真正的功夫，常常不在写了多少行，而在**每一行放在哪个时机、为哪种失败留了哪条退路**。同学们，下课前再默背一遍那三行模板：读、查边界、有才设——以后你们写的每一个"恢复上次状态"，都会用到它。
