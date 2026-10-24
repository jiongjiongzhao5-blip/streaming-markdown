# 第十二课：Qt 主界面 —— 给黑乎乎的控制台穿上图形外衣，以及"子线程不许直接碰界面"这条铁律

> **对应提交**：`3b22c698694e46c5a4ddc815d81b34a5b4cf2741`（简称 `3b22c69`）
> **提交时间**：2026-10-22 16:49
> **提交信息原文**：Qt 主界面
> **改动规模**：4 个文件，**+536 / −129**
> - `src/app/MainWindow.h`（86 行，新增）
> - `src/app/MainWindow.cpp`（437 行，新增）
> - `src/app/main.cpp`（132 行 → 17 行，基本重写）
> - `src/app/CMakeLists.txt`（14 行 → 6 行，精简）
> **课前准备**：
> - 第 3 课（`StreamConfig` 契约 —— 今天最大的一行类型转换就是"界面 → `StreamConfig`"）
> - 第 4 / 5 / 7 课（三个采集器的 `listDevices()` —— 今天下拉框里的数据就是从这来的）
> - 第 8 / 10 课（编码器与 `StreamOutput` —— 今天的"开始"按钮按下去，最终启动的就是它们）
> - **第 11 课（`StreamPipeline` —— 这一课必须听懂，否则今天的每个按钮都不知道在指挥谁）**
> **课后产物**：一个能点、能选、能看实时统计数据、能看滚动日志的真窗口 `LightLiveStreamer.exe`
> **本课一句话**：**界面从来不是设备本身，它只是一个遥控器；而这个遥控器能不能按得稳，全靠"子线程不许直接碰界面、有事排队发消息"这条规矩撑着。**

---

## 目录（这节课我们怎么走）

| 章节 | 内容 |
|---|---|
| 第 0 章 | 开场白：前面都是黑乎乎的控制台，今天给它穿上图形外衣；"控制台验证 → GUI 包装"这个开发顺序为什么值钱 |
| 第 1 章 | 知识点预习（上）：Qt 世界入门 —— `QApplication`、主事件循环、对象树与父子所有权、`QWidget` 体系 |
| 第 2 章 | 知识点预习（中）：信号与槽、moc（元对象编译器）、`Q_OBJECT`、CMake AUTOMOC |
| 第 3 章 | 知识点预习（下）：★ **跨线程更新 UI 的纪律**（本课灵魂）—— 线程亲和性、为什么不能跨线程碰 `QWidget`、三种投递写法 |
| 第 4 章 | 知识点预习（补）：布局管理三大件 + 常用控件 + `QTimer` / `QElapsedTimer` |
| 第 5 章 | 逐行教学（一）：`src/app/CMakeLists.txt` —— 六行里的四个门道 |
| 第 6 章 | 逐行教学（二）：`main.cpp` —— 从 132 行自检程序砍到 17 行启动器 |
| 第 7 章 | 逐行教学（三）：`MainWindow.h`（86 行）—— 一个类怎么表达"界面 + 状态机" |
| 第 8 章 | 逐行教学（四）：构造函数与析构函数 —— 顺序就是生命线 |
| 第 9 章 | 逐行教学（五）：`buildUi()` —— 造控件、摆布局、连信号，三步走 |
| 第 10 章 | 逐行教学（六）：设备枚举三兄弟 —— `refreshVideoDevices` / `refreshAudioDevices` / `updateSourceFields` |
| 第 11 章 | 逐行教学（七）：`collectConfig()` —— 界面到契约的翻译官 |
| 第 12 章 | 逐行教学（八）：★ `startStreaming()` —— 本课最难的一段，逐 token 讲 |
| 第 13 章 | 逐行教学（九）：`stopStreaming()` 与 `updateUiState()` |
| 第 14 章 | 逐行教学（十）：`pollStats()` / `appendLog()` / `postLog()` / `closeEvent()` |
| 第 15 章 | 动手环节：编译运行、界面长什么样、操作验收清单、七个改造实验 |
| 第 16 章 | 排错手册：Qt DLL 缺失、moc 未生成、跨线程断言崩溃等 14 条 |
| 第 17 章 | 小结、局限、速查表、思考题 8 题（含答案要点）、下节课预告 |
| 附 | 本讲「五章对照索引」 |

> **老师的课前提醒**：这一课的"新 API"数量是十二课以来最多的，但**真正烧脑的只有一件事**：我们的推流管线里有**三条内部线程**（采集、音频、写网络），它们随时可能产生日志；而 Qt 规定**只有创建界面的那个线程（主线程）能碰界面**。怎么把"子线程说的话"安全地送到"主线程的文本框里"，就是第 3 章和第 12 章要解决的问题。**第 3 章请务必听懂，那是本课的灵魂；第 12 章请逐行读完，那是本课的代码高峰。**
>
> 另外提醒一句：这一课**没有新增任何一行采集、编码、封装的业务代码**。它改的四个文件，全部属于"外壳"。这本身就是一种很重要的工程态度 —— 我们稍后在第 0 章细讲。

---

# 第 0 章 开场白：前面都是黑乎乎的控制台，今天给它穿上图形外衣

## 0.1 同学们，先看看我们一路走来用的是什么界面

我们把时间线拉回前十一课，回想一下我们每次"验收成果"时在干什么：

| 课次 | 提交 | 当时的验收方式 | 屏幕上有什么 |
|---|---|---|---|
| 第 1 课 | `20b267a1` | 跑 `cmake` 看有没有报错 | 黑窗口，一堆编译输出 |
| 第 2 课 | `df3603c8` | 跑 `01-mini-pusher`，然后看 `out.flv` | 黑窗口，几行 printf |
| 第 3 课 | `a4453bb1` | 跑 `test_common.exe` | 黑窗口，30 条 `[OK]` |
| 第 4 课 | `81eaa1f5` | 跑 `test_screen_capture.exe` | 黑窗口 + 生成一张 bmp |
| 第 5 课 | `281a22be` | 跑摄像头测试 | 黑窗口，一堆 DirectShow 设备名 |
| 第 6 课 | `e92cbd3` | 跑格式转换测试 | 黑窗口 + 一堆 bmp |
| 第 7 课 | `1ce7f62` | 跑 WASAPI 测试 | 黑窗口，一堆音频设备名 |
| 第 8 课 | `66e0768` | 跑编码器测试 | 黑窗口，帧数/字节数 |
| 第 10 课 | `0ccd108` | 跑 `test_output.exe` + VLC 播 flv | 黑窗口 + 4 个能播的文件 |
| 第 11 课 | `31c3f57` | 跑三线程管线（仍是命令行驱动） | 黑窗口，滚动日志 |
| **第 12 课（本次）** | **`3b22c69`** | **双击 exe，鼠标点按钮** | **★ 一个有按钮、有下拉框、有计数器的窗口** |

同学们注意到了吗：**前十一课我们一次也没有见过"窗口"**。我们用的是 printf、用断言、用"生成一张图片让你用眼睛看"。这不是偷懒，这是一种**刻意安排的开发顺序**。今天就把它讲透。

## 0.2 "控制台验证 → GUI 包装"这个顺序到底好在哪

很多同学上手就想在 Qt Designer 里拖界面，拖完再往里塞逻辑。这条路十个人走了九个掉坑。我们走的是另一条：**先把内核做出来并且用控制台证明它是对的，最后才包一层皮。**

这么做的好处有五条，一条都不虚：

**第一条：把"工艺问题"和"逻辑问题"分开。**

假设我们一上来就做界面，然后发现推不出流。这时候嫌疑范围有多大？采集没抓到？格式不对？编码器参数错？地址填错？**还是按钮的信号没连上、跨线程更新把界面搞崩了？** 嫌疑人一大堆，排查成本高得吓人。

反过来，我们前十一课已经在控制台里证明了：**"屏幕 → BGRA → H.264 + AAC → FLV → RTMP"这条链路每一段单独跑都对**。那今天如果推不出流，嫌疑范围就只剩"界面这一层"了。**把搜索空间砍掉 80%，这就是最大的生产力。**

**第二条：控制台是最诚实的调试器。**

`printf` 出来的东西管道就能重定向、能 grep、能塞进日志、能写进测试脚本断言。而 GUI 里的信息藏在窗口里，没法自动化。所以我们早就养成了这个规矩：**每个内核模块都要能脱离界面自己跑，并且顺便产出可供断言的东西**（测试数据、bmp、flv）。

今天的 `MainWindow` 里的日志框，说白了就是把以前打在控制台的那批话，**换了个地方显示**。内核一行没改。

**第三条：内核对界面零依赖，反过来取证才容易。**

请大家注意一个事实：这次 commit 改了四个文件，**全在 `src/app/` 下**。而 `src/core/` 下的采集、转换、编码、封装、管线 —— **一个字都没动**。这说明我们的分层是真分层，不是嘴上说说的分层：核心模块不知道 `QApplication` 是什么，它只认 `StreamConfig` 和 `LogCallback`。

这条性质太值钱了：

- 想写命令行版？把界面层删掉，写个 17 行的 `main` 调 `pipeline_.start(config, printf)` 就行；
- 想写移动端/网页端？换个界面层皮；
- 想加自动化压测？绕过界面直接调核心；
- 出 bug 了？在控制台里就能复现，不用手点半天。

**第四条：界面是"最不稳定的那一层"，留到最后写最划算。**

需求变更最快的是哪一层？永远是界面："这里加个输入框""这个按钮改成灰的""加个实时预览窗口"。看后面的提交就知道，第 13 课 `bafb4ea` 就要加"实时预览"，第 15 课 `7eb889b` 要加"配置持久化"——**界面层天天改，内核层基本不动**。如果一开始就把两者搅在一起，每次改界面都要重编译、重验证内核，成本爆炸。

**第五条：最后包界面的人，才是真正把系统"收口"的人。**

包界面不是"画几个按钮"那么简单。它强迫你回答这些问题：

- 用户能改哪些参数？哪些参数**不许在推流过程中改**（今天 `updateUiState()` 里的 `editable` 就是在回答这个）；
- 操作失败了怎么告诉用户？（今天还只写日志，后面要弹窗）
- 关窗口的时候没停推流，谁负责收尾？（今天 `closeEvent()` 在回答这个）
- 后台在跑，怎么让用户知道它活着？（今天 `pollStats()` + 状态栏在回答这个）

这些问题，写内核的时候可以躲开，写界面的时候一个都躲不掉。所以界面层不是"不重要的一层"，它是**把半成品变成产品的一层**。

## 0.3 那今天到底要做成什么样

我把今天的窗口画出来，大家先看个全貌：

```
┌─ LightLiveStreamer ─────────────────────────────────────────── □ ─ × ─┐
│                                                                        │
│  ┌── 视频源 ──────────────────────┐  ┌── 音频源 ───────────────────┐  │
│  │ 来源   [ 屏幕            ▾ ]   │  │ 来源  [ 系统声音       ▾ ]  │  │
│  │ 设备   [              ▾ ][刷新]│  │ 设备  [ (系统默认)    ▾ ][刷新]│ │
│  │ 分辨率 [ 1280 x 720    ▾ ]    │  │                              │  │
│  │ 帧率   [ 30 fps        ]      │  │                              │  │
│  │ 码率   [ 2500 kbps     ]      │  │                              │  │
│  └───────────────────────────────┘  └──────────────────────────────┘  │
│                                                                        │
│  ┌── 推流地址 ──────────────────────────────────────────────────────┐ │
│  │ bilibili=rtmp://live-push.bilivideo.com/live-bvc/?streamname=...  │ │
│  │ douyin=rtmp://push-rtmp-l6.douyincdn.com/stage/...                │ │
│  └───────────────────────────────────────────────────────────────────┘ │
│                                                                        │
│  [ 开始推流 ]  [ 停止推流 ]        推流中                                │
│                                                                        │
│  时长 00:01:23    视频 2790 帧 / 2790 包 / 丢 3                          │
│                  音频 5600 块 / 2795 包      连接 1/2                    │
│                                                                        │
│  ┌── 日志 ──────────────────────────────────────────────────────────┐ │
│  │ [20:31:02] 就绪。填好推流地址后点「开始推流」。                      │ │
│  │ [20:31:05] 正在连接 2 路...                                        │ │
│  │ [20:31:05] 枚举到 1 个摄像头                                       │ │
│  │ [20:31:06] 推流已开始                                              │ │
│  │ [20:31:06] 推流已启动: 1280x720 @30fps, 2/2 路                     │ │
│  └───────────────────────────────────────────────────────────────────┘ │
└────────────────────────────────────────────────────────────────────────┘
```

五块内容，从上到下：

1. **参数区**（左：视频源 / 右：音频源）——负责把用户的选择变成 `StreamConfig`；
2. **推流地址区**——一个多行文本框，每行一个目标，支持 `名称=地址` 的写法；
3. **控制区**——开始/停止按钮 + 粗体状态字；
4. **统计区**——每 500ms 刷一次的四个数字；
5. **日志区**——从**所有线程**汇聚过来的一行行带时间戳的文字。

第 5 块看着最朴素，其实是今天技术含量最高的地方。为什么？因为写日志的人是别人：可能是 GUI 线程自己（"正在连接 2 路..."），可能是**启动线程**（"推流已启动：1280x720..."这条是 `StreamPipeline::start()` 里说的），也可能是**管线内部的视频线程/音频线程/写线程**（比如"连接断开""重连成功"）。这些线程谁都不认识 `QPlainTextEdit`，但它们说的话都要出现在那个框里。

这件事怎么做才是安全的，就是我们第 3 章的主菜。

## 0.4 今天的"外壳"和内核的接口，其实只有三个

我们再回头看一眼整个调用图，你会发现界面和内核之间，**只有三个接触点**：

```
   ┌──────────────────────── GUI 线程（主线程，唯一能碰界面的线程）─────────────┐
   │                                                                             │
   │   MainWindow                                                                │
   │     ├── collectConfig()  ──生成──→  StreamConfig ──┐                        │
   │     │                                              │                        │
   │     └── 点击「开始」───→ 开一条 std::thread ────────┤                        │
   │                                                    ↓                        │
   │                              pipeline_.start(config, logCallback)           │
   │                                                    │                        │
   └────────────────────────────────────────────────────┼────────────────────────┘
                                                        ↓
                      StreamPipeline 内部捞出三条线程（video / audio / write）
                                                        │
                                       logCallback( std::string )  ← 任意线程都可能调用
                                                        ↓
                                            MainWindow::postLog()
                                                        ↓
                               QMetaObject::invokeMethod(..., QueuedConnection)
                                                        ↓   （跨线程投递，排到 GUI 线程的事件队列）
                                                    appendLog()
                                                        ↓
                                               logView_->appendPlainText()
```

三个接触点分别是：

| # | 接触点 | 方向 | 代码位置 |
|---|---|---|---|
| ① | `StreamConfig` | UI → 内核（一次性传参） | `collectConfig()`（`MainWindow.cpp:226`） |
| ② | `LogCallback` | 内核 → UI（持续回话） | `startStreaming()` 里的 lambda（`MainWindow.cpp:296`） |
| ③ | `PipelineStats` | UI → 内核定时拉取 | `pollStats()`（`MainWindow.cpp:383`） |

注意 ② 的方向：**内核主动说，界面被动听，而且说话的人可能不是主线程**。这条线就是本课所有复杂度的来源。

好，开场白讲完。我们记住一句话：**这次 commit 没写一行新业务逻辑，但它决定了前面十一课做出来的东西，能不能被人用。**

---

# 第 1 章 知识点预习（上）：Qt 世界入门

这一章给没写过 Qt 的同学补课。写过的同学可以快速过，但请不要跳过 1.3 和 1.4 —— 那两节连老手也常常搞混。

## 1.1 Qt 是什么，以及它不是什么

**Qt**（读作"cute"）是一个跨平台的 C++ 应用程序框架。它提供的东西远超"图形界面库"的范畴：

- **界面部分**（`QtWidgets`）：今天要用的按钮、下拉框、布局；
- **核心部分**（`QtCore`）：字符串 `QString`、容器 `QVector`/`QList`、事件循环、定时器、线程（`QThread`）、文件/配置读写；
- **元对象系统**：信号与槽、`QObject` 树、反射（`metaObject()`）——这是 Qt 最独特的部分，C++ 语言本身没有这些东西，Qt 用一套自己的办法补齐了；
- **其他**：网络、数据库、多媒体、OpenGL……

所以准确的说法是：**Qt 是一个"给 C++ 补了一整套现代框架"的开发平台，GUI 只是它的一个子集。**

我们项目用的是 **Qt 6**，模块是 **`QtWidgets`**（传统桌面控件那一套，不是新一代的 QML/Qt Quick）。这两者的区别是：

- **QWidgets**：C++ 代码手写的经典桌面控件，Windows/Linux/macOS 上都能用，样式偏"原生控件"。适合工具类软件（也就是我们这种）。手写代码为主。
- **QML / Qt Quick**：声明式语言描述界面，动画炫酷，适合消费类 App 和嵌入式触屏。要学 JavaScript 风格的语法。

**我们选 Widgets，因为它能用纯 C++ 写清楚，跟整个项目的 C++ 学习路线一致。**

## 1.2 CMake 里 Qt 是怎么接进来的

回头看看第 1 课（`20b267a1`）就在根目录 `CMakeLists.txt` 里埋好的两句（第 8 行、第 25 行）：

```cmake
set(CMAKE_AUTOMOC ON)              # 第 8 行：打开自动 moc（第 2 章细讲）

find_package(Qt6 REQUIRED COMPONENTS Widgets)   # 第 25 行：找到 Qt6 的 Widgets 组件
```

这句 `find_package` 做的事情是：在系统里找到 Qt 6 的安装位置，**创建一个叫 `Qt6::Widgets` 的 CMake 目标**。这个目标身上挂着三样东西，谁 `target_link_libraries(XXX PRIVATE Qt6::Widgets)`，谁就能自动继承：

| Qt6::Widgets 携带的内容 | 作用 |
|---|---|
| **头文件搜索路径**（INTERFACE_INCLUDE_DIRECTORIES） | 让 `#include <QWidget>` 等语句能找到头文件本身 |
| **库文件**（lib/.lib/.dll.a） | 链接时找到符号实现 |
| **编译特性**（INTERFACE_COMPILE_FEATURES → `cxx_std_17`） | 自动把使用者的 C++ 标准拉到至少 C++17 |

正因为第 2 条，我们在 `src/app/CMakeLists.txt` 里只要写目标名 `Qt6::Widgets` 就够了，**不需要再写头文件路径，也不需要管 DLL 名字**。这是现代 CMake 的"以目标为单位建模依赖"思想，跟第 1 课讲的 `INTERFACE` 库是一脉相承的。

> **老师提醒**：Qt6 强制要求 C++17。我们项目本来就是 `set(CMAKE_CXX_STANDARD 17)`，正好对上。如果你在别的项目里用了 Qt6 却报一堆 `std::optional` / `if constexpr` 的错，八成是 C++ 标准没设对。

## 1.3 `QApplication` 与"主事件循环"—— Qt 程序的发动机

这是 Qt 程序和传统 C++ 程序**最根本的区别**。我们看今天的 `main.cpp`（第 6 章会逐行讲，这里先看骨架）：

```cpp
int main(int argc, char** argv)
{
    QApplication app(argc, argv);   // ① 造一个"应用程序对象"

    MainWindow window;              // ② 造主窗口（还没显示）
    window.show();                  // ③ 让它显示出来

    return app.exec();              // ④ ★ 进入主事件循环，此处阻塞，直到程序退出
}
```

四步里，**第 ④ 步是整个 GUI 程序的灵魂**。

### 主事件循环（event loop）到底是什么

传统的控制台程序是一条直线：从 `main` 第一行走到最后一行，走到 `return` 就结束。中间想读用户输入？`std::cin >> x`，**在那里原地阻塞等着**。

GUI 程序不能这么干。为什么？因为用户一会儿点这个按钮，一会儿拉那个下拉框，还可能同时有"定时器到点了""网络数据来了""该重绘窗口了"这些事发生，**你根本不知道下一个事件是什么、什么时候来**。

所以 GUI 程序的写法是一个**死循环**：

```
while (程序还没退出) {
    事件 = 从队列里取出一个事件();          // 没事件就睡觉，不占 CPU
    把这个事件交给对应的处理者();
}
```

这个 while 循环，Qt 写好了，叫**主事件循环**，入口就是 `app.exec()`。它内部大致是这样：

```
┌───────────────────── Qt 主事件循环（在 GUI 线程里跑）─────────────────────┐
│                                                                            │
│   ① 鼠标移动 / 鼠标点击 / 键盘按下   →  QMouseEvent / QKeyEvent            │
│   ② 定时器到点                        →  QTimerEvent                        │
│   ③ 窗口需要重画                      →  QPaintEvent                        │
│   ④ 别的对象投递过来的"待办事项"      →  QMetaCallEvent（槽的排队执行）  ★ │
│   ⑤ socket 有数据                     →  QSocketNotifier                    │
│                        ↓ 全部塞进「事件队列」                              │
│                    主线程一个个取出来处理                                   │
└────────────────────────────────────────────────────────────────────────────┘
```

**第 ④ 类事件就是我们今天要反复利用的东西**：别的地方不方便直接调用的函数，包装成一个"待办事项"塞进这个队列，主线程一会儿就会取出来执行。**所谓"跨线程更新 UI"，本质上就是往这个队列里塞一个待办事项。**

### 一条铁的推论：主线程不许被堵住

既然所有事件（包括"窗口重画"）都要排队等着主线程处理，那么结论就非常直接了：

> **只要你写的某个函数在主线程里卡了 3 秒，这 3 秒内窗口就拖动不了、按钮点不了、重画也做不了 —— 用户看到的就是"窗口变白/未响应"。**

这条推论直接决定了我们今天为什么要在按钮按下的瞬间**新开一条线程**去做 `pipeline_.start()`：因为 `start()` 里面要连 RTMP 服务器，网络不好的时候可能卡 5 秒、10 秒，甚至等到超时。这种活**绝对不能在主线程干**。

请同学们记住这个判断法则，比记住任何 API 都重要：

| 操作的预期耗时 | 该在哪执行 | 例子 |
|---|---|---|
| 微秒 ~ 几毫秒 | 主线程直接干 | 改个标签文字、读下拉框当前值、算个字符串 |
| 几十毫秒 ~ 几百毫秒 | 主线程**勉强**能干，但已经开始掉帧 | 枚举摄像头设备、解析一个配置文件 |
| 几百毫秒以上 / 网络 IO / 可能超时 | **必须开线程** | 连 RTMP、打开摄像头、等管线停止、写文件刷盘 |

## 1.4 对象树：Qt 里 `new` 出来的东西谁负责 `delete`

同学们马上会看到，`buildUi()` 里全是 `new`，**一个 `delete` 都没有**：

```cpp
auto* central = new QWidget(this);                     // 没 delete
videoSourceCombo_ = new QComboBox(videoGroup);         // 没 delete
startButton_ = new QPushButton("开始推流", central);   // 没 delete
```

一个 `delete` 都没有，内存不泄漏吗？答案是：**Qt 有一套"对象树"（object tree）机制，父对象负责析构所有子对象。**

规则很简单：

1. `QObject` 的构造函数可以接收一个 `parent` 参数：`new QPushButton("确定", parentWidget)`；
2. 一旦指定了父亲，这个孩子就被登记进父亲的 `children()` 列表；
3. **父亲析构时，自动 delete 所有还活着的孩子**（Qt 的 `~QObject` 里会做这件事）；
4. 孩子自己被 `delete` 时，也会先从父亲的列表里注销自己，所以不会有"野指针孩子"。

于是整个界面形成一棵树：

```
MainWindow (QMainWindow)
└── central (QWidget)                    ← setCentralWidget(central) 后被登记为孩子
    ├── videoGroup (QGroupBox "视频源")
    │   ├── videoSourceCombo_ (QComboBox)
    │   ├── cameraCombo_ (QComboBox)     ← 进 QHBoxLayout，但 parent 仍是 videoGroup
    │   ├── refreshCameraButton_ (QPushButton)
    │   ├── resolutionCombo_ (QComboBox)
    │   ├── fpsSpin_ (QSpinBox)
    │   └── bitrateSpin_ (QSpinBox)
    ├── audioGroup (QGroupBox "音频源")
    │   ├── audioSourceCombo_ ... etc
    ├── targetGroup (QGroupBox "推流地址")
    │   └── targetsEdit_ (QPlainTextEdit)
    ├── startButton_ / stopButton_ / stateLabel_   ← 进了 controlLayout，parent=central
    ├── durationLabel_ / videoLabel_ / ...         ← 进了 statsLayout，parent=central
    └── logGroup (QGroupBox "日志")
        └── logView_ (QPlainTextEdit)
外加：
    statsTimer_ (QTimer)                 ← new QTimer(this)，parent=MainWindow
```

只要最顶上的 `MainWindow` 被销毁（它是 `main()` 里的栈对象 `window`，出了作用域自动析构），整棵树从上到下自动清理干净。**这就是为什么 `MainWindow::~MainWindow()` 里只有两行 join 线程的代码，一个指向控件的 delete 都没有。**

> **一条重要的澄清**：**布局（QLayout）不等于父亲。** 看这行：`videoForm_->addRow("来源", videoSourceCombo_);` —— `addRow` 只是让布局"管理"这个控件的摆放，**不会**改变控件的 parent。控件的 parent 早在 `new QComboBox(videoGroup)` 这一步就定了。
>
> 但 Qt 有个贴心的地方：**如果你把一个已有 parent 的控件塞进某个布局，Qt 会自动把它的 parent 重设为"这个布局所在的那个控件"**（因为布局必然装在某个 QWidget 上）。所以我们才会看到 `startButton_ = new QPushButton("开始推流", central)` 里 parent 写的是 `central` —— 它知道自己一会儿要进 `controlLayout`，而 `controlLayout` 的主人正是 `central`。写对了就一致；万一写错了（比如传了另一个 GroupBox），Qt 会帮你搬，但**显式写对是好习惯**。

补充三点必须知道的：

- **`this` 做 parent 的写法**：`new QTimer(this)` —— 意思是"我的父亲是当前的 MainWindow 对象"。这样定时器跟着窗口一起销毁。**定时器的 parent 尤其重要，否则窗口关了定时器还在跑，等它一 tick 就去调用已经析构的 `pollStats()` —— 直接崩。**
- **栈上也能造 QWidget**：`MainWindow window;` 就是栈对象。规则是"只要保证父亲比孩子活得久"就行。Qt 的规则是：**父对象先析构并 delete 孩子；如果孩子是栈对象，Qt 会在孩子析构时自动从父亲列表注销，不会出问题。** 实践中推荐"顶层窗口用栈、里面的控件用 new+parent"。
- **不要用智能指针管理有 parent 的 QWidget**：会造成"双重释放"（Qt 父亲删一次、`unique_ptr` 再删一次）。Qt 的世界里，"有 parent 的 QObject"用裸指针 + parent 机制就够了。

## 1.5 `QString`、`QChar`、以及 Qt 自己的那套类型哲学

今天的代码里到处是 `QString`。为什么 Qt 不直接用 `std::string`？

因为 Qt 设计于 1990 年代，那时候 STL 还不成气候。Qt 自己造了一整套字符串/容器体系（`QString`、`QByteArray`、`QList`、`QVector`、`QMap`……），好处是它们深度配合 Qt 的元对象系统，而且内部用**隐式共享**（implicit sharing / copy-on-write）做优化 —— 拷贝一个 `QString` 是 O(1)，只有在要修改时才真的把底层数据复制一份。

所以我们的代码里出现了两个方向的转换：

```cpp
QString::fromStdString(name)    // std::string → QString（第 188 行、第 212 行等）
line.toStdString()              // QString → std::string（第 254~257 行）
QString::fromLatin1(...)        // 老版本 Qt 里常见，本项目没用到但对理解有帮助
```

> **一个必须记得的细节**：`QString` 内部存的是 **UTF-16**（不是 UTF-8）。所以 `QString::fromStdString()` 会做一次 UTF-8 → UTF-16 的转码；反过来 `toStdString()` 是 UTF-16 → UTF-8。**这个转换是有成本的**，一行日志无所谓，一帧 1920×1080 的数据用 QString 传就傻了 —— 所以我们整个项目的像素数据都用 `std::vector<uint8_t>` 走，**只在"给人看的字符串"这一层用 QString**。这是正确的分工。

还有一类 Qt 的"看不见的成员"，同学在看文档时会遇到：

| 写法 | 含义 | 何时用 |
|---|---|---|
| `QStringLiteral("abc")` | **编译期**构造 QString，零运行时开销 | 性能敏感的常量字符串（第 6 章以前的老代码用过） |
| `QLatin1String("abc")` / Qt6 的 `QLatin1StringView` | 对纯 ASCII 字面量的轻量包装，避免转码 | 跟 ASCII 比较、传参时 |
| `tr("abc")` | 标记为"可翻译文本"（ linguist 工具用） | 需要做多语言的界面文案，**本项目没用 tr，说明暂时没打算国际化** |

今天的代码里中文字符串是**直接写**的（`"推流已开始"`），这依赖两件事：

1. 源文件必须是 **UTF-8 编码**；
2. MSVC 必须加 **`/utf-8`** 编译选项（第 1 课就在根 `CMakeLists.txt` 里加好了：`add_compile_options(/utf-8)`）。

没有 `/utf-8` 的话，MSVC 会按系统本地编码（GBK）去解释源码里的字节，界面上就是一串"鎺ㄦ祦宸插紑濮"之类的乱码。**这是 Windows 上写 Qt 中文界面第一高频的坑**，我们第 16 章还会回头说。

## 1.6 小结：第一章我们学到了什么

| 概念 | 一句话 |
|---|---|
| Qt / QtWidgets | 跨平台 C++ 框架；Widgets 是传统桌面控件那一套，适合工具类软件 |
| `find_package(Qt6 COMPONENTS Widgets)` | 生成 `Qt6::Widgets` 目标，携带头文件路径、库、C++17 要求 |
| `QApplication` | 应用程序对象，管理全局资源、事件队列、命令行参数 |
| **主事件循环**（`app.exec()`） | GUI 程序的发动机：一个从队列取事件、分发处理的死循环 |
| **主线程不许阻塞** | 阻塞 = 界面卡死；可能耗时的操作必须另开线程 |
| **对象树 / parent** | `new QXxx(parent)` 的孩子会被父亲自动 delete，上面一个 item 都不用管 |
| **QLayout ≠ parent** | 布局只管摆放；parent 在构造时指定（塞进布局时 Qt 可能帮忙修正） |
| `QString` | Qt 的字符串，内部 UTF-16、隐式共享；`fromStdString` / `toStdString` 来回转换 |

---

# 第 2 章 知识点预习（中）：信号与槽、moc、Q_OBJECT、AUTOMOC

这一章我们要解释 Qt 最独特的地方：**C++ 语言本身没有"信号"和"槽"，Qt 是怎么变出来的？**

## 2.1 为什么需要"信号与槽"

先看一个最朴素的需求：**按钮被点了，我要执行一段代码。** 用传统 C++ 怎么做？

- **方案一：回调函数。** `button->setCallback(myFunction);` —— 能用，但函数指针没法带上下文（不能绑定成员函数+this 组合），类型不安全，而且生命周期管理全靠人工。
- **方案二：虚函数覆写。** `class MyButton : public QPushButton { void onClick() override; };` —— 每做一个按钮就要派生一个类，类爆炸，而且"谁关心这个事件"和"谁产生这个事件"被焊死了。
- **方案三：轮询。** 定时器里不断检查"按钮是不是被点了"—— 荒唐，但真有人写。

Qt 给的答案叫**信号与槽（Signal & Slot）**，核心是两个角色：

- **信号（signal）**：对象在"某件事发生了"的时候**发射**（emit）出来的一个通知。比如 `QPushButton` 有一个信号叫 `clicked`。
- **槽（slot）**：一个普通的成员函数（或者 lambda），准备好"当那个通知来了，我来干活"。

中间的"连线"动作由 `QObject::connect()` 完成：

```cpp
connect(startButton_, &QPushButton::clicked,   // 谁的信号
        this,                                  // 谁来干活（上下文对象）
        &MainWindow::startStreaming);          // 干什么活
```

它的革命性在于两点：

1. **彻底解耦**：发射方（`QPushButton`）完全不知道谁会接。可能一个人接，可能五个人接，可能一个人都不接 —— 按钮的代码一行都不用改。这在设计模式里叫**观察者模式（Observer Pattern）**的一种漂亮实现。
2. **支持 lambda**：Qt 5 起允许直接把 lambda 当槽，于是大量"只有一句话的小活"再也不用去头文件加一个成员函数了。今天我们就能看到三处 lambda 槽（第 165~168 行的两处，和第 296 行的日志回调）。

## 2.2 古怪的地方：C++ 里哪来的 `emit`？

同学们去翻 `QPushButton` 的头文件，会发现信号是这样声明的：

```cpp
class QPushButton : public QAbstractButton {
    Q_OBJECT
public:
    ...
signals:
    void clicked(bool checked = false);   // ← signals 是什么关键字？
};
```

`signals`、`slots`、`emit`、`Q_OBJECT` —— **这四个都不是 C++ 的关键字**。你用 `g++`/`MSVC` 单独编译这段代码，编译器会懵。

秘密在于：这些"关键字"其实是**宏**，定义在 Qt 的 `qobjectdefs.h` 里：

```cpp
#define signals  public
#define slots                        // 展开成空！
#define emit                         // 展开成空！
#define Q_OBJECT    /* 展开成一堆 public 成员声明 + 两个 friend 声明，见下 */
```

也就是说：

- `signals:` 展开就是 `public:`；
- `slots:` 和 `emit` 展开成**空**（纯粹是给人读的标记）；
- `emit someSignal(...)` 实际就是 `someSignal(...)` —— 一次普通的成员函数调用！

那"发射一个信号"到底干了什么？**调用那个信号函数，而它的函数体里做的事是：遍历所有连到这个信号的槽，把它们挨个调用一遍。**

而这个信号函数的**函数体是由 moc 生成的**，这就是下一节的内容。

## 2.3 moc：Qt 的"元对象编译器"

**moc**（Meta-Object Compiler，元对象编译器）是 Qt 自带的一个小程序。它的工作流程是这样的：

```
  MainWindow.h（含 Q_OBJECT）
          │
          │  moc 扫描：哦，这里有 Q_OBJECT，还有 signals/slots 声明
          ↓
  moc_MainWindow.cpp          ← 自动生成的 .cpp 文件
    ├── MainWindow 的静态/常量元数据表（类名、信号名、槽名、参数类型的字符串表）
    ├── MainWindow::metaObject()、qt_metacast()、qt_metatype_query() 等虚函数实现
    ├── 每个 signal 的函数体实现（那个"遍历槽并调用"的循环）
    └── moc 自己用的内部辅助表（比如 signals 的下标蹦床函数 qt_static_metacall）
          │
          │  和其他 .cpp 一起编译
          ↓
  最终的可执行文件
```

一句话：**`Q_OBJECT` 是给 moc 的记号；moc 看到它就生成一坨 C++ 代码来"描述"这个类，从而让 Qt 拥有 C++ 原生没有的反射能力（运行时按名字找信号/槽/属性）。**

既然 C++ 没有反射，Qt 就必须自己造一个注册表。这个注册表就是 moc 生成的那些静态常量 + `metaObject()`。

### `Q_OBJECT` 到底必须写在哪

规则：**任何想拥有 Qt 元对象能力的类，都必须在类声明的最开头写上 `Q_OBJECT`。** 具体来说，以下事情需要它：

| 想要的能力 | 是否需要 `Q_OBJECT` |
|---|---|
| 声明自己的 `signals:` / `slots:` | **必须** |
| 用 `tr()` 做国际化 | **必须** |
| `metaObject()`、`className()`、`inherits()` | **必须** |
| `qobject_cast<>`、`dynamic` 属性系统 | **必须** |
| 继承一个已有 `Q_OBJECT` 的类来当普通派生类 | 通常也要写（否则新类没有自己的元数据） |
| **只是**把普通成员函数当槽连出去（`connect(btn, &QPushButton::clicked, this, &MyClass::foo)`） | **技术上不需要**（第 2.4 节解释） |

我们 `MainWindow` 里其实**没有** `signals:` 段、`slots:` 段 —— 它的槽是普通私有成员函数和 lambda。这在第 2.4 节有专门讨论。**但代码里还是写了 `Q_OBJECT`**，这是 Qt 程序员的肌肉记忆，也是对的：

- 哪天你想加一个 `signals: void logReady(QString);`，不用回头补宏（补宏忘记重跑 cmake 的话会报很难懂的链接错误，见第 16 章）；
- `tr()`、`metaObject()` 随时可用；
- 成本是零（除了多几百字节的元数据和一个 .cpp 的生成）。

> **老师的取舍建议**：写 Qt 类就写 `Q_OBJECT`，不要玩"这次能不能省"的心眼。省下来的那点编译时间，远不够你将来 debug "为什么我的 signal 连不上" 花的十分钟。

## 2.4 `connect()` 的三种写法，以及 Qt 6 里的新语法

`connect` 在 Qt 历史上出现过三种写法，今天的代码用的是**第二种**（函数指针 + lambda）：

### 写法一：老字符串宏语法（Qt 4 时代，已淘汰）

```cpp
connect(startButton_, SIGNAL(clicked()), this, SLOT(startStreaming()));
```

原理是把信号和槽的名字变成**字符串**，运行时去 moc 生成的元数据里查。毛病一大堆：

- **编译期不做检查**：名字拼错了（`clicked` 写成 `cliked`）编译照样过，**运行时**才打印一行警告 "QObject::connect: No such slot"，然后功能静默失效 —— 新人找这种 bug 能找一下午。
- 参数类型必须完全匹配，不支持隐式转换（比如 `int` 到 `double` 就不行）。
- 不能连 lambda。

### 写法二：函数指针语法（Qt 5 起，★ 本项目用这个）

```cpp
connect(startButton_, &QPushButton::clicked, this, &MainWindow::startStreaming);
connect(statsTimer_,  &QTimer::timeout,      this, &MainWindow::pollStats);
connect(videoSourceCombo_, &QComboBox::currentIndexChanged, this,
        [this](int) { updateSourceFields(); });
```

**优点**：

- **编译期检查**：`&MainWindow::startStreaming` 拼错了（或者写了个不存在的函数名）→ **编译直接报错**，红波浪线当场出现。
- 允许槽的参数比信号**少**（多余的参数被丢弃）。例如 `QPushButton::clicked(bool checked)` 有 1 个 bool 参数，而我们的槽 `MainWindow::startStreaming()` 是 0 个参数 —— 完全合法，那个 bool 被丢掉了。这个特性非常实用。
- 第三个参数 **`this`（上下文对象）**很关键：它决定了**这个连接什么时候自动断开**（`this` 析构时自动断）、以及在多线程时**槽在哪个线程执行**（见第 3 章）。
- 支持 lambda 作为第 3/4 参数（省略 this 时 lambda 将会在发送者所在线程被调用）。

**注意此处 `this` 的真正含义**：`connect(发送者, 信号, 上下文对象, 槽)` 的 4 参数形式，**第 3 个参数不是"由谁执行"的意思** —— 槽永远在第 4 参数这个可调用物被调用时决定。第 3 个参数的作用有两个：**(a) 生命周期：上下文对象销毁时自动断开连接；(b) 确定接收方的线程亲和性，从而决定连接类型（Direct vs Queued）。** 这一点必须理解准确，否则第 3 章的连接类型你会搞反。

### 写法三：lambda 语法（Qt 5 起）

```cpp
connect(videoSourceCombo_, &QComboBox::currentIndexChanged, this,
        [this](int /*index*/) { updateSourceFields(); });
```

用在"槽里只有一两句"的场景，省得去头文件加成员函数。**捕获 `[this]` 是最常见的写法**，也暗藏危险（见下）。

> **★ 危险提示：`[this]` 捕获 + 跨对象的 lambda 是 Qt 的经典血案。**
> `[this]` 捕获的是**裸指针**。如果 lambda 在另外一个线程里延迟执行，而 `this` 指向的对象已经析构了 —— **悬垂指针，必崩**。Qt 提供了带上下文的重载来缓解：`connect(sender, sig, this, [this]{...});` 中的 `this` 会让连接随对象销毁而断开。但如果 lambda 已经排进队列了呢？第 3 章和第 14 章会正面讨论这个问题，今天 `postLog` 的实现就默认祈祷 ("the invokeMethod with context object doesn't post after destruction")。先记着，后面用得上。

## 2.5 `CMakeLists.txt` 里的 AUTOMOC 到底做了啥

回到第 1 课就在根 `CMakeLists.txt` 第 8 行打开的那句话：

```cmake
set(CMAKE_AUTOMOC ON)
```

**AUTOMOC** 是 CMake 提供的一个便利功能。没有它的时候，我们要手工列出 moc 的目标：

```cmake
qt_wrap_cpp(sources MainWindow.h)    # 老办法，或者：
set_property(SOURCE MainWindow.h PROPERTY SKIP_AUTOMOC OFF)
```

有了 AUTOMOC，CMake 会在构建时做这件事：

```
构建 LightLiveStreamer 之前，扫描它的所有源文件（含 .h）:
    MainWindow.cpp   → 里面有 #include "moc_MainWindow.cpp"?  现在没有 → 不管
    MainWindow.h     → 里面有 Q_OBJECT！ → 调用 moc 生成 moc_MainWindow.cpp
                     → 把生成的文件加进这个目标的编译列表
  ↓
正常编译 main.cpp、MainWindow.cpp、moc_MainWindow.cpp，链接成 exe
```

**这里有个非常重要的陷阱**，正好解释了我们这次 diff 里一个小细节：

> **自动扫描会不会漏掉我的头文件？**
>
> CMake 的 AUTOMOC 会智能处理"和 .cpp 同名且同目录的 .h"（比如 `MainWindow.cpp` 配 `MainWindow.h`）——这种情况即使你不在 target 里列出 .h，AUTOMOC 也能找到它。
>
> 但如果是**重命名后不同名**、**只被 include 的头文件**（比如某个 `Foo.h` 里写了 `Q_OBJECT`，却只有 `Bar.cpp` include 它），CMake 就可能扫不到，结果就是：**链接期报 `unresolved external symbol ... MainWindow::staticMetaObject` 或者 `vtable` 错误**。
>
> **保险做法：把含 `Q_OBJECT` 的 .h 明确写进 `add_executable` 的源文件列表里。** CMake 知道 `.h` 不是编译单元，不会拿它去编译；只是让 AUTOMOC 一定看得到它。

看我们今天的 `src/app/CMakeLists.txt`：

```cmake
add_executable(LightLiveStreamer
  main.cpp
  MainWindow.cpp
  MainWindow.h        # ← 这个 .h 被明确列进来了！不为编译，只为 AUTOMOC 能扫到
)
```

**这一行不是多余，是老手的保险。** 请大家在自己的项目里也养成这个习惯。第 16 章我们会把"忘了这一行的报错长什么样"完整演示一遍。

## 2.6 小结：第二章我们学到了什么

| 概念 | 一句话 |
|---|---|
| 信号与槽 | 观察者模式的 Qt 实现：发射方不知道谁在听，`connect` 负责连线 |
| `emit` / `signals` / `slots` | **都是宏**，`slots` 和 `emit` 展开为空，`signals` 展开为 `public` |
| **moc**（元对象编译器） | 扫带 `Q_OBJECT` 的头文件，生成 `moc_xxx.cpp` 补充 C++ 缺失的反射能力 |
| `Q_OBJECT` | 给 moc 的记号；想用 signals/slots/tr/metaObject 就必须写，写 Qt 类就该写 |
| **函数指针式 `connect`** | 编译期检查类型、支持参数丢弃、支持 lambda；第三个参数是**上下文对象**（管生命周期和线程亲和性） |
| **AUTOMOC** | CMake 自动调用 moc；把含 `Q_OBJECT` 的 .h 显式写进 target 源文件列表是保险做法 |
| `[this]` 捕获的风险 | lambda 延迟执行时可能 `this` 已析构；用带上下文的 connect 重载缓解 |

---
# 第 3 章 知识点预习（下）：★ 跨线程更新 UI 的纪律（本课灵魂）

> **这一章是本次课的核心。如果整套讲稿你只记住一件事，请是这一件：**
> **在 Qt 里，只有创建界面的那个线程（一般就是主线程 / GUI 线程）可以碰界面；其他线程想通知界面改东西，只能通过"投递消息队列"的方式，不能直接调用。**

这一章我会花大量篇幅讲"为什么"，因为**不理解的规矩记不住，记住了也不会用对**。

## 3.1 先让事故真实发生一次

假设我们把今天代码里的 `postLog`（第 417 行）写成这个样子 —— 这是新手最自然的第一反应：

```cpp
// ❌ 错误示范！子线程直接操作界面
void MainWindow::postLog(const std::string& message)
{
    const QString text = QString::fromStdString(message);
    logView_->appendPlainText(text);      // ← 从别的线程直接调用 QWidget 的方法！
}
```

会发生什么？可能的现象清单（**注意是"可能的现象之一"，不是"一定会出现某一个"**）：

| 现象 | 概率 |
|---|---|
| 看起来"一切正常"，日志正常显示 | 有，而且不少 |
| 日志显示错乱/丢行/重复 | 常见 |
| 界面某一块不刷新，或者窗口某处变黑 | 常见 |
| **随机崩溃**，调用栈落在 `QWidget`、`QPainter`、`QTextDocument` 深处 | 常见 |
| 触发 Qt 的调试断言：`ASSERT failure in QCoreApplication::sendEvent: "Cannot send events to objects owned by a different thread"` | Debug 版本常见 |
| 直接在 `Q_ASSERT_X(this->thread() == QThread::currentThread())` 处炸掉 | 部分 Qt 版本/某些 Widget API |

**这才是最可怕的地方：它"有时候能跑"。** 这种 bug 在你本地开发机上可能一直不犯事，到了客户机器上、换了个 Qt 版本、多开了一个线程或者 CPU 忙一点的时候，突然一天崩两次。**偶发的未定义行为，是最难修的一类 bug。**

## 3.2 为什么会这样：Qt 对象有"归属线程"（线程亲和性）

Qt 有一个概念叫**线程亲和性**（thread affinity，也叫"归属线程"）。它的意思是：**每个 `QObject` 都记住了"我是属于哪个线程的"**。

规则非常简单：

```
一个 QObject 属于「创建它的那个线程」。
之后可以用 obj->moveToThread(T) 把它整个搬到另一个线程去。
obj->thread() 可以查它现在属于哪个线程。
QCoreApplication::instance()->thread() 就是主线程（GUI 线程）。
```

所以，我们在 GUI 线程里 `new` 出来的 `logView_`、`startButton_`、`stateLabel_`……它们的归属线程无一例外都是主线程。

Qt 官方对 QWidget 的线程安全性说明写得非常直白（我翻译成大白话）：

> **`QWidget` 及它的所有子类都不是线程安全的（not thread-safe）**：你不能从非 GUI 线程调用它们的方法。具体的限制是：**只有创建 QWidget 的那个线程（通常是主线程）才能访问它，包括它的所有派生类。** Qt 只对少数几个函数例外（比如 `QCoreApplication::postEvent`，它是专门为了跨线程通信设计的）。

注意"显式例外"这四个字：**Qt 只开了 `QCoreApplication::postEvent` 这一扇门，剩下的全是墙。**

## 3.3 深层原因：为什么 Qt 不能干脆给 QWidget 加把锁？

有同学会问："加个互斥锁不就线程安全了吗？mutex 我们在第 3 课 `BoundedQueue` 里用过啊，很简单啊。"

这个问题问得非常好，答案是：**不是 Qt 的作者不会加锁，是加了也没用，而且代价大到无法接受。** 我给四个层面的原因：

### 原因一：界面操作不是一个"瞬间动作"，而是一连串动作的叠加

对 `QWidget` 来说，"设置文本"这个操作至少包含：

1. 修改内部数据；
2. 计算新的尺寸提示（`sizeHint`）、让布局系统重新分配空间；
3. 标记这块区域"需要重画"（`update()`，往事件队列塞一个 `QPaintEvent`）；
4. 一会儿主线程处理这个 paint 事件，用 `QPainter` 真正画到屏幕上。

这四个步骤跨了好几个对象（控件本身、它的父控件、布局对象、Qt 的全局样式引擎）。就算给每个对象都加锁，**'步骤组合起来' 依然不是原子的**：子线程改了一半文本，主线程正好开始画，画出来的就是不完整的中间状态。想让它原子，就得锁住一整棵界面子树 + 布局 + 样式 —— 那基本等于"把所有界面操作串行化"，多线程的意义就没了。

### 原因二：底层操作系统本身就把窗口句柄"绑"在创建线程上

在 Windows 上（我们这项目就是 Windows）：

- 每个可见控件背后都有一个 **HWND**（窗口句柄）；
- Windows 的消息循环是**按线程**组织的：**发到某个 HWND 的消息，只会被创建它的那个线程处理**；
- 如果你从别的线程 `SendMessage()` 给这个 HWND，Windows 会**强行切换到拥有它的线程**去执行（这是个跨线程阻塞调用），如果那个线程此刻在等你……**这就死锁了**。整个 Windows GUI 体系的设计前提就是"窗口属于创建它的线程"。

Qt 没法推翻操作系统。它只能遵守同样的规矩。

### 原因三：Qt 内部有大量"只属于主线程的全局状态"

举几个例子：

- **样式引擎**（`QStyle`）：算一个按钮长什么样要查一大片共享缓存；
- **字体数据库**（`QFontDatabase`）：字体匹配结果缓存；
- **`QApplication` 的全局单例数据**（比如"当前焦点窗口""剪贴板""拖放状态"）；
- **屏幕/高 DPI 信息**（`QScreen`、`devicePixelRatio`）。

这些东西太多了，Qt 全部当作"主线程私有数据"来用，外面一个锁都没有。让子线程进来调用，等于让两个人同时改同一个没有被保护的大结构体。

### 原因四：加所有这会引入新的死锁

界面操作最终触发的往往是"给操作系统发消息"，而发消息可能**阻塞等待主线程响应**。如果主线程正在等某个子线程（比如 `worker_.join()`），而子线程正在锁着界面锁等界面重画……**死锁**。Qt 选择了一条更干脆的路：**直接规定子线程不许进来，比加锁省事、比加锁可靠、比加锁快。**

> **一句话记诵**：不是 Qt 懒，是**图形界面天生就是"单线程 + 事件驱动"的模型**。GUI 线程模型的设计目标是"响应快、无锁、不会被画到一半"，而不是"能被任意线程调用"。

## 3.4 那正确的路只有一条：往 GUI 线程的事件队列里"投消息"

还记得第 1.3 节那个事件队列吗？它是整件事的关键。

**正确的跨线程更新姿势：**

```
┌────────────────────┐                    ┌───────────────────────────────┐
│  工作线程          │                    │  GUI 线程（主线程）            │
│                    │                    │                               │
│  拿到一条日志      │                    │   while (app.exec()) {         │
│         │          │                    │      事件 = 队列.take();      │
│         ↓          │   投递（异步）      │      处理(事件);              │
│  invokeMethod(...) │ ──────────────────→│   }                           │
│  QueuedConnection  │   排队，立即返回    │                               │
│         ↓          │                    │   ...轮到这个事件了...         │
│  继续干活/结束     │                    │       ↓                       │
│                    │                    │   执行 appendPlainText()  ★   │
└────────────────────┘                    └───────────────────────────────┘
         ↑                                              ↑
    这里绝不能直接调 QWidget 的方法             只有这里能碰 QWidget
```

三个要素缺一不可：

1. **投递是异步的**：调用 `invokeMethod` 之后立刻返回，工作线程继续干活，不等界面真的更新完成。**这一点非常重要** —— 如果界面更新要等很久（比如你现在 1 秒刷 100 条日志），工作线程也不会被拖慢。
2. **投递需要一个"接收者对象"（context）**：我们今天传的是 `this`（`MainWindow`）。Qt 根据这个对象的归属线程决定"把这条消息投递到哪个线程的事件队列"。
3. **被投递的那段代码，最终在 GUI 线程里执行**：所以在里面碰 widget 是 100% 安全的。

## 3.5 连接类型五兄弟：Direct / Queued / Auto / BlockingQueued / Unique

`Qt::ConnectionType` 是 Qt 线程模型里最核心的一组常量。我们先把它一次性讲清楚：

| 类型 | 行为 | 什么时候有用 |
|---|---|---|
| **`Qt::DirectConnection`** | 发射信号的那一刻，**立即、同步地**调用槽，**在发射者线程里执行**（也就是调用堆栈还留在发射者线程） | 同一个线程内、需要"调用完立刻看到结果"的场景；**缺点是跨线程用它会让槽在错误的线程执行** |
| **`Qt::QueuedConnection`** | 不立即调用，而是把"该调用的槽 + 参数"打包成一个事件，**投递到接收者所在线程的事件队列**，立即返回 | ★ **跨线程通信的标准答案**。也是我们今天全部三处跨线程调用用的类型 |
| **`Qt::AutoConnection`**（默认） | **自己判断**：如果接收者线程 == 当前线程 → 走 Direct；否则 → 走 Queued | `connect()` 的默认值，大多数同线程场景用起来最自然 |
| **`Qt::BlockingQueuedConnection`** | 像 Queued 一样投递过去，**但发射者线程会阻塞等待槽执行完**（用信号量等待） | 需要"跨线程调用并取得返回值"；**⚠ 发射者和接收者不能在同一个线程，否则死锁** |
| `Qt::UniqueConnection` | 不是独立类型，是一个可以和上面按位或的**修饰符**：同样的"信号+接收者+槽"只允许连一次 | 防止重复 connect 造成"一次点击触发两次" |

换成一张动作图，让大家一眼看出差别：

```
Qt::DirectConnection                    Qt::QueuedConnection
─────────────────────                   ─────────────────────
线程A:  emit sig()                      线程A:  emit sig()
         │ 同步，同栈                            │ 塞事件进队列，立即返回
         ↓                                      ↓  （线程A 继续往下跑）
线程A:  slot()  ← 仍在线程A                    ...过了一会儿...
         ↓                                线程B:  事件循环取到它
线程A:  emit 后面的代码                          ↓
                                        线程B:  slot()  ← 在线程B（接收者的线程）跑
```

### 一个高频陷阱：AutoConnection 的"半叛变"

很多同学写跨线程代码时会想："我就用默认的 AutoConnection 吧，Qt 会自己判断的。" 大部分时候是对的。但请看这个场景：

```cpp
// 这个函数可能来自 GUI 线程，也可能来自工作线程（比如 start() 内部的日志回调）
void MainWindow::postLog(const std::string& message)
{
    const QString text = QString::fromStdString(message);
    QMetaObject::invokeMethod(this, [this, text] { appendLog(text); });  // ← 没写连接类型
}
```

没写第三个参数时，Qt 用默认的 **`Qt::AutoConnection`**。于是：

- 如果此时是**工作线程**在调用 → 检测发现"我 != this 的归属线程" → 走 Queued → ✅ 正确；
- 如果此时恰好是 **GUI 线程**在调用（想想看：`pipeline_.start()` 内部第一件事就是 `emit` 一条日志，而我们的 `start` 是在工作线程里跑的；但将来如果有别的路径在主线程调 start 呢？）→ 检测发现"我 == this 的归属线程" → 走 Direct → **lambda 立刻在当前栈里执行**。

Direct 本身不崩（因为在 GUI 线程），但**行为不一致**：同一份代码在两种调用来源下时序不同，某些依赖"异步"的假设就会翻车（比如你以为日志排在最后一条，结果它插到了最前面；或者你以为不会被马上改动的状态被立刻改了）。

**所以本项目的做法是：凡是"跨线程投递"的地方，一律显式写 `Qt::QueuedConnection`。** 这个态度值得大家抄：

> **纪律一：跨线程投递，显式写 `Qt::QueuedConnection`，不要依赖 AutoConnection 判断。**
> 理由：显式写出来，行为在两种来源下完全一致，代码可读性也强 —— 读代码的人一眼就知道"这里是打算跨线程的"。

## 3.6 三种投递写法，以及本项目为什么选了第三种

Qt 提供了三种"把这段代码送到 GUI 线程去执行"的办法。我们从老到新看：

### 写法 A：自定义信号 + QueuedConnection 的槽（最"正统 Qt"的做法）

```cpp
// MainWindow.h 里：
signals:
    void logRequested(const QString& text);     // 需要 Q_OBJECT + moc

// 构造函数里：
connect(this, &MainWindow::logRequested, this, &MainWindow::appendLog,
        Qt::QueuedConnection);

// 任意线程里：
emit logRequested(QString::fromStdString(message));
```

优点：最符合 Qt 的设计意图，moc 生成的元数据里能看到这个信号；槽可以直接是成员函数（便于复用）。
缺点：要在头文件里加 signals 段；**信号的参数类型必须是 Qt 元类型系统认识的类型**（见第 3.8 节的坑）。

### 写法 B：`QMetaObject::invokeMethod` + 槽名字（字符串版）

```cpp
QMetaObject::invokeMethod(this, "appendLog", Qt::QueuedConnection,
                          Q_ARG(QString, text));
```

缺点：**参数是字符串**，拼错编译不报错、运行时静默失败；而且对中文无助。Qt 5 时代的遗物，**新代码别用**。

### 写法 C（★ 本项目的选择）：`QMetaObject::invokeMethod` + lambda（Qt 5.10 起提供）

```cpp
QMetaObject::invokeMethod(
    this,                                   // ① 上下文对象：决定投递到哪个线程
    [this, text] { appendLog(text); },      // ② 要在 GUI 线程执行的代码
    Qt::QueuedConnection);                  // ③ 显式指定排队投递
```

两个重载要区分清楚：

```cpp
// (1) 带"上下文对象"的重载 —— 我们用的这个。Qt 5.10 起才支持 lambda
template<typename Functor>
static bool invokeMethod(QObject *context, Functor function,
                         Qt::ConnectionType type = Qt::AutoConnection, ...);

// (2) 老的字符串版 —— 见写法 B，不推荐
```

优点：

- **编译期检查**：lambda 体里写错东西立刻报错，不像字符串版那样默默失效；
- **不用在头文件加 signals**，少写 moc 代码；
- **捕获什么类型都行**（`std::string`、`std::vector`、你自己的类……），**绕开了 QVariant 元类型注册的限制**（见 3.8）；
- 想捕获几个变量捕获几个，不用为了传参去定义新类型。

代价：**要求 C++11 lambda + Qt ≥ 5.10**（我们 Qt 6，完全够）。

> **老师的选择建议**：项目里"日志往窗口里塞"这种零散小活，用写法 C；如果是"结构性的一件事"（比如"预览帧准备好了"这种会成为模块契约的通知），用写法 A 定义正式信号更清晰 —— 因为它会出现在类的元数据里，别人看头文件就知道这个类对外发什么信号。

## 3.7 一个必踩的坑：跨线程传 `std::string` 会被 Qt 拦下来

这一节是我们选写法 C 的**具体技术理由**，请务必看懂。

假设我们用写法 A，并且图省事直接让信号携带 `std::string`：

```cpp
signals:
    void logRequested(const std::string& text);     // ← 危险
```

然后跨线程 `emit`。运行时控制台会出现这行红字：

```
QObject::connect: Cannot queue arguments of type 'std::string'
(Make sure 'std::string' is registered using qRegisterMetaType().)
```

原因：**QueuedConnection 的原理是"把这次调用存成一个事件"。** Qt 需要：(1) 把参数拷贝一份保存起来；(2) 一会儿在目标线程里把它还原出来。做这两件事需要知道参数的**元类型信息**（怎么拷贝、怎么析构、多大/怎么创建）—— 这就是 Qt 的 **元类型系统**（`QMetaType`）。

而 `QMetaType` 只认识：

- Qt 自己的类型（`QString`、`int`、`QSize`、`QVariant`……）；
- 用 **`Q_DECLARE_METATYPE(MyType)`**（编译期）+ **`qRegisterMetaType<MyType>()`**（运行时，跨线程排队时必须）注册过的自定义类型。

**`std::string` 没被 Qt 注册过**，所以它不能被放进排队的事件里。

两条出路：

```cpp
// 出路一：注册它（正确，但要记得在任何 connect 之前调用）
#include <QMetaType>
qRegisterMetaType<std::string>("std::string");

// 出路二（本项目做法）：转成 Qt 认识的类型再传，或者干脆用 lambda 捕获
const QString text = QString::fromStdString(message);   // std::string → QString
QMetaObject::invokeMethod(this, [this, text] { appendLog(text); }, Qt::QueuedConnection);
                                // ↑ lambda 捕获，Qt 只需要拷 layout
```

**注意：lambda 捕获不走 QVariant！** Qt 对"带上下文对象的 invokeMethod + lambda"用的是 `QFunctorSlotObject`，它把整个 lambda 当成一份二进制数据拷进事件对象，**不需要注册任何元类型**。这就是为什么我们可以随身携带 `std::string`、`std::vector<...>` 之类的类型 —— 它们作为"捕获物"，不作为"排队参数"。

> **纪律二：跨线程投递时，要么传 Qt 认识的类型（QString/int/…），要么注册元类型，要么用 lambda 捕获三者选一。别以为编译过了就没事 —— 这条是运行时才会报的错误。**

## 3.8 最后一个不那么显然的问题：对象销毁了，投出去的消息怎么办

这是 Qt 跨线程编程的**进阶坑**，今天我们被刚好踩上去一次，值得讲。

回顾一下我们代码里会出现的场景：

```
    GUI 线程                                    工作线程
    ─────────                                    ─────────
    用户点"停止"
      ↓
    开一条线程去 pipeline_.stop()   ─────────→  跑 stop()
      ↓                                          ↓
    用户直接关窗口                                stop 完了
      ↓                                      调用 invokeMethod(this, ..., Queued)
    closeEvent()
      ↓  (第 428 行) worker_.join();  ← 就是这一步在保护我们
    ~MainWindow 开始执行
```

关键风险：**如果 GUI 线程已经把 `MainWindow` 析构掉了，工作线程才对着那个"已经消失的对象"调用 `invokeMethod(this, ...)`，那 `this` 就是野指针 —— 立刻未完成行为，通常表现为开车（悬垂指针访问）。**

代码里做了两件事来防它：

1. **`closeEvent`（第 424~429 行）**：关窗口时先停定时器 → 再 `worker_.join()`，**把还在跑的工作线程彻底等完** → 才调用基类的 `closeEvent`；
2. **`~MainWindow`（第 35~39 行）**：析构第一件事也是 `worker_.join()`。

有了这两道，`this` 在被投递之前永远不会被销毁。

那么反过来：如果一条消息**已经投递出去排队了**，但对象这时候被析构了怎么办？Qt 帮我们处理：

- **信号 连接 会在 receiver 析构时自动断开**（`~QObject` 做这件事）；
- **已经投递出去的事件**，`~QObject` 会调用 `QCoreApplication::removePostedEvents(this)` 把它从队列里删掉 —— 事件被删除时，里面包着的那份 lambda 也会被析构（捕获的变量跟着一起正确释放，不会泄漏，也不会被错误执行）。

**这是 Qt 提供的最贴心的保护之一：只要你是"通过对象投递"的，对象没了消息就没了，不会帮助你执行 + 也不泄漏。**

> **纪律三：任何持有 `this`（或别的 QObject 指针）跨线程使用的代码，必须保证"析构之前把还在跑的线程 join 完"。** 这是我们 `closeEvent` 和析构函数里那几行的全部意义。**如果图省事写成 `worker_.detach()`，那你等着迎接"隔三差五崩一次"的人生吧。**

## 3.9 本课的跨线程调用点总表 + 纪律清单

我们把今天所有可能跨越线程边界的调用列出来，逐个确认它是否合规：

| # | 调用点 | 发起线程 | 落地对象在哪一维 | 做法 | 合规 |
|---|---|---|---|---|---|
| 1 | `pipeline_.start(config, logCb)`（`MainWindow.cpp:296`） | 新建的 `worker_` 线程 | ——（不碰 UI） | 放在 `std::thread` 里跑 | ✅ |
| 2 | 日志回调 `logCb(msg)`（`MainWindow.cpp:296` 的 lambda） | **管线内部的 video / audio / write 线程，或 start 所在的 worker 线程** | `MainWindow`（GUI） | `postLog` → `invokeMethod(..., QueuedConnection)` | ✅ |
| 3 | 启动完成后更新 UI（`MainWindow.cpp:308~326`） | `worker_` 线程 | `MainWindow`（GUI） | `invokeMethod(..., QueuedConnection)` | ✅ |
| 4 | 停止完成后 `appendLog("已停止")`（`MainWindow.cpp:347`） | `worker_` 线程 | `MainWindow`（GUI） | 同上 | ✅ |
| 5 | `pollStats()`（`MainWindow.cpp:383`） | **GUI 线程**（QTimer 的 parent 是 this，定时器在 GUI 线程） | `MainWindow`（GUI） | 不用投递，本来就在 GUI 线程 | ✅ |
| 6 | `pipeline_.stats()`（被 pollStats 调用） | GUI 线程 | 读的是 `std::atomic` 计数器 + 加了 `pipelineMutex_` | 自己加锁保护 | ✅ |
| 7 | `updateUiState()`（多处） | GUI 线程（从槽/ tous lambda 里来） | `MainWindow` | 直接调用 | ✅ |
| 8 | `closeEvent` 里 `pipeline_.stop()` | GUI 线程 | —— | 持 `pipelineMutex_` 同步停，**可以接受**（关窗口允许短暂等待） | ⚠️ 见第 13 章讨论 |

**纪律清单（请抄下来贴在显示器边上）：**

```
┌──────────────────────── Qt 跨线程纪律六条 ────────────────────────┐
│                                                                    │
│  ①  只有 GUI 线程能碰 QWidget（含 QWidget 子类的任何方法）。         │
│  ②  子线程要更新 UI → 往 GUI 线程事件队列投递，三种写法见 3.6。      │
│  ③  投递显式写 Qt::QueuedConnection，别指望 AutoConnection。        │
│  ④  跨线程传参：用 Qt 认识的类型，或注册 QMetaType，或用 lambda 捕获。│
│  ⑤  BlockingQueuedConnection 绝不能用于同线程，会死锁。             │
│  ⑥  持有 this 的后台线程，析构前必须 join；不许 detach 后不管。      │
│                                                                    │
│  附加：Q_OBJECT 类的槽靠 connect 自动在正确线程排队；                │
│        手搓 std::thread 就全靠你自己 invokeMethod。                 │
└────────────────────────────────────────────────────────────────────┘
```

> **老师的一点感悟**：这套纪律看起来繁琐，但它的本质是"**用消息代替共享内存**"。我们第 3 课学的 `BoundedQueue` 也是这个思路：生产者不直接摸消费者的锅，把东西放窗口就走。Qt 的事件队列就是 GUI 版的 `BoundedQueue`，只不过它的消费者固定是 GUI 线程。**同学们会发现，好的并发设计，思路都是相通的**。

---

# 第 4 章 知识点预习（补）：布局管理三大件与常用控件

我们把第 3 章的"线程纪律"放一放，回到更轻松的话题：界面怎么摆。

## 4.1 为什么绝对不能用"绝对坐标"

新手最容易犯的第二个错（第一个是跨线程）：这样写代码：

```cpp
label->move(20, 30);
label->resize(120, 25);
edit->move(20, 60);
edit->resize(300, 25);
```

这在 Qt 里叫"绝对布局"（absolute positioning），毛病集四件套：

1. **窗口一拉大就露白**：右上角空一大块；
2. **窗口一缩小控件就跑出去**；
3. **中文字体一换就挤成一团**（同样一段"推流地址"，不同字体下宽度差 30%）；
4. **不同屏幕 DPI / 不同语言全崩**：笔记本 125% 缩放 vs 外接 100% 缩放，位置全错。

Qt 的答案是**布局管理器**（layout manager）：你告诉它"这几个控件排成一列""这两个并排，左边那个多占空间"，剩下的算位置和大小的事它全包了。窗口尺寸一变，布局重算一遍，控件跟着走。

## 4.2 三大布局类和它们的适用场景

| 布局类 | 排布方式 | 典型用途 |
|---|---|---|
| **`QVBoxLayout`** | **垂直**排列：一个摞一个 | 窗口整体从上到下；把组套起来 |
| **`QHBoxLayout`** | **水平**排列：一个挨一个 | 工具栏式的一行；把两个分组左右并排 |
| **`QFormLayout`** | **两列**：左边标签、右边控件 | ★ 表单（设置项）—— 今天的"视频源""音频源"两组就是它 |
| `QGridLayout` | 网格：任意 `第x行第y列` | 表格状的复杂面板（今天没用，但你们会用上） |

三者可以**任意嵌套**，因为布局本身既可以装控件也可以装另一个布局：

```
QMainWindow
└── mainLayout (QVBoxLayout)              ← 总骨架：从上到下
    ├── sourceLayout (QHBoxLayout)        ← 第 1 行：左右各一组
    │   ├── videoGroup (QGroupBox)
    │   │   └── videoForm_ (QFormLayout)  ← 里面是表单
    │   │       ├── 行 1: "来源"  → videoSourceCombo_
    │   │       ├── 行 2: "设备"  → cameraRow_ (QHBoxLayout)  ← 嵌套！一行里放两个控件
    │   │       ├── 行 3: "分辨率" → resolutionCombo_
    │   │       ├── 行 4: "帧率"  → fpsSpin_
    │   │       └── 行 5: "码率"  → bitrateSpin_
    │   └── audioGroup (QGroupBox)
    │       └── audioForm (QFormLayout)
    ├── targetGroup (QGroupBox "推流地址")
    │   └── targetLayout (QVBoxLayout)
    │       └── targetsEdit_
    ├── controlLayout (QHBoxLayout)       ← 第 3 行：按钮 + 状态
    ├── statsLayout (QHBoxLayout)         ← 第 4 行：四个统计数字
    └── logGroup (QGroupBox "日志", stretch=1)   ← 第 5 行：额外占多余空间
        └── logLayout (QVBoxLayout)
            └── logView_
```

看着复杂，其实是**同一种操作重复六次**：`addWidget` / `addLayout`。

## 4.3 `addWidget(widget, stretch)` 里那个数字到底什么意思

```cpp
cameraRow_->addWidget(cameraCombo_, 1);          // stretch = 1
cameraRow_->addWidget(refreshCameraButton_);     // stretch = 0（默认）

sourceLayout->addWidget(videoGroup, 1);
sourceLayout->addWidget(audioGroup, 1);
```

**stretch（拉伸因子）**的含义：**当布局有多余空间可以分配时，按这个比例分。**

- `stretch = 0`（默认）："我只要我的 `sizeHint`（尺寸提示）就够了，多余空间别给我。"

  → `refreshCameraButton_` 保持它刚好能显示"刷新"两个字的宽度。
- `stretch = 1`："多余空间给我，我可以变宽（或变高）。"

  → `cameraCombo_` 会被拉宽，把整行剩下的空间占掉，下拉框看起来就宽了。
- 两个都是 1 → 多余空间**平分**（`videoGroup` 和 `audioGroup` 各占一半宽度）。
- 一个是 1 一个是 2 → 前者拿 1/3，后者拿 2/3。
- 两个都是 0 → 谁都不扩展，多余空间由布局的 `addStretch()` 或留白处理。

一句话：**stretch 管的是"空间富余时怎么分"，不是"初始大小"。**

## 4.4 `addStretch()` / `addSpacing()` / `addSpacerItem()`

这一组很容易混淆，我们一次讲清：

| 调用 | 效果 | 今天的用法 |
|---|---|---|
| `layout->addSpacing(16)` | 加 **16 像素的固定空白**，永远不会缩放 | `controlLayout->addSpacing(16)`（把状态字和按钮隔开） |
| `layout->addStretch(n = 0)` | 加一个**弹性空间**（像弹簧），它会把后面所有控件推到一边，自己吃掉所有多余空间 | `controlLayout->addStretch()`、`statsLayout->addStretch()`（让剩下的控件靠左，右侧是空的） |
| `addSpacerItem(new QSpacerItem(40, 20, QSizePolicy::Expanding, QSizePolicy::Minimum))` | 完整版，可以同时控制水平和垂直策略 | 今天没用到 |

看这两个例子，体会"弹簧"的效果：

```cpp
// 无弹簧：三个控件把整行平分拉开，按钮之间有缝隙
// [开始推流]         [停止推流]         [未开始]

// 加了 addStretch() 后：前两个控件紧贴左侧，剩下的空间全被弹簧吃掉
// [开始推流] [停止推流]   [状态字]  ←────── 弹簧在这里 ──────→（右侧留白）
```

**记住这个套路：`addStretch()` 放在最后 = 让前面的控件靠边；`addStretch()` 放在最前 = 让后面的控件靠右；前后都放 = 中间那堆居中。**

## 4.5 `QGroupBox`：自带边框和标题的分组容器

```cpp
auto* videoGroup = new QGroupBox("视频源", central);
videoForm_ = new QFormLayout(videoGroup);
```

**`QGroupBox`** 是一个容器控件，外观是"一个带标题的方框"。用法固定三步：

1. `new QGroupBox("标题", 父)`；
2. `new QxxxLayout(groupBox)` —— **把 title 和 layout 直接传给它，布局自动"安装"到这个 groupBox 上**（等价于写 `groupBox->setLayout(layout)`）；
3. 往布局里 addWidget。

它还有个常用属性 **`setCheckable(true)`**：标题前加一个复选框，勾掉时组内所有控件变灰。今天没用（我们用的是 `setRowVisible` + `setEnabled`，见第 10/13 章）。

## 4.6 控件逐个过一遍（含和"看起来很像的邻居"的对比）

### ① `QComboBox` —— 下拉框（今天的重头戏）

```cpp
videoSourceCombo_->addItem("屏幕");                       // ① 只加文本
resolutionCombo_->addItem("1280 x 720", QSize(1280, 720)); // ② 文本 + 附加数据（QVariant）
audioDeviceCombo_->addItem("（系统默认）", -1);              // ② 附加 int
```

`addItem()` 的第二个参数是 **`QVariant`**（Qt 的"万能盒子"，能装 int/QString/QSize/自定义类型……），叫 **userData**。这招非常好用：**界面上显示人话（"1280 x 720"），程序里拿到机器数值（`QSize(1280,720)`）**，不用再去解析字符串、也不用维护"显示名 → 数值"的映射表。

取回来的方式：

```cpp
const QSize size = resolutionCombo_->currentData().toSize();   // QVariant → QSize
const int   idx  = audioDeviceCombo_->currentData().toInt();   // QVariant → int
```

其他常用 API：

| API | 作用 |
|---|---|
| `addItem(text, userData)` / `addItems(QStringList)` | 加条目 |
| `clear()` | 清空（会发射 currentIndexChanged，**注意！**枚举设备时的重点，见 10.2） |
| `count()` | 条目数 |
| `currentIndex()` / `setCurrentIndex(i)` | 当前选中项的下标（-1 = 无） |
| `currentText()` | 当前显示文本 |
| `currentData()` | 当前附加数据（QVariant） |
| `findText(s)` | 按文本找下标，找不到返回 **-1** |

**信号**：`currentIndexChanged(int)`（下标变了）、`currentTextChanged(const QString&)`（文本变了）、`activated(int)`（**只有用户主动选**才发，`setCurrentIndex` 代码改的不发）。

> **Qt 5 vs Qt 6 的小差别**：Qt 5 里 `currentIndexChanged` 有两个重载（`(int)` 和 `(const QString&)`），写 `&QComboBox::currentIndexChanged` 编译器会分不清，必须写成 `QOverload<int>::of(&QComboBox::currentIndexChanged)`。**Qt 6 删掉了 QString 那个重载，所以我们可以直接写**`&QComboBox::currentIndexChanged`（第 165 行）而不会编译报错。这是一个"Qt 6 更省事"的小例子。

### ② `QSpinBox` —— 带上下箭头的整数输入框

```cpp
fpsSpin_->setRange(5, 60);          // 允许范围
fpsSpin_->setValue(30);             // 当前值
fpsSpin_->setSuffix(" fps");        // 后缀：显示成 "30 fps"
bitrateSpin_->setSingleStep(500);   // 点一下箭头走多少（默认 1）
```

它自己负责：越界剪裁、只能输入数字、键盘上下键调节。取出来就是一个 `int`（`value()`）。

同类还有 **`QDoubleSpinBox`**（小数）。

### ③ `QPlainTextEdit` vs `QTextEdit` —— 今天两次出场的主角

**这两个最容易混，但今天的课preparation 要求掌握 QTextEdit 的概念，所以我们把对比讲透。**

| 特性 | **`QPlainTextEdit`**（本项目用） | `QTextEdit` |
|---|---|---|
| 内容 | **纯文本**，一个大字符串 | 富文本：颜色、字体、加粗、表格、图片 |
| 内部模型 | 为**逐行纯文本**优化的轻量级实现 | 完整的 `QTextDocument`（文本文档对象模型） |
| 性能 | ★ **高**（尤其在频繁追加、大量文本时） | 低（同一操作要维护更多结构） |
| `setMaximumBlockCount(n)` | ★ 有！最多保留 n 个"块"（大致是一段/一行），超出自动从头删 | 没有这个概念，只能手动清 |
| 追加一行 | `appendPlainText(s)`（自带换行） | `append(s)`（按 HTML 解释！） |
| 语法高亮 | 不支持（需自己 paint） | 支持（配合 `QSyntaxHighlighter`） |
| 适用场景 | ★ **日志输出**、代码编辑器、输入多行地址 | 富文本邮件编辑器、聊天框、需要图文混排 |

我们选 `QPlainTextEdit` 的理由非常明确：

1. **日志输出**需要"高频追加 + 不能无限吃内存" → `setMaximumBlockCount(2000)`（第 154 行）正好掐住上限；
2. **推流地址输入**就是纯文本 → 不需要富文本能力；
3. `appendPlainText()` 会把输入当**纯文本**处理。如果用 `QTextEdit::append()`，日志里万一出现 `<b>` 之类的标签就会被当成 HTML 渲染 —— 对于一个把**任意第三方错误信息**都往里倒的日志框来说，这是不可接受的（甚至是安全隐患）。

其他用到的 API：

```cpp
targetsEdit_->setPlaceholderText("...");   // 灰色提示文字，用户一开始输入就消失
targetsEdit_->setMaximumHeight(88);        // 最多 88 像素高
targetsEdit_->toPlainText();               // 取出全部文本
targetsEdit_->setEnabled(editable);        // 灰掉/恢复
logView_->setReadOnly(true);               // 只读（用户不能编辑日志）
```

### ④ `QPushButton` —— 按钮

```cpp
startButton_ = new QPushButton("开始推流", central);
stopButton_->setEnabled(false);
refreshCameraButton_->setFixedWidth(64);
```

- 信号：**`clicked(bool checked = false)`**（那个 bool 对普通按钮永远是 false，只有 checkable 按钮才有意义）；
- `setEnabled(false)` → 变灰，**不再发射 clicked**；
- `setFixedWidth(64)` → 固定宽度 64px（相对布局的"建议size"而言的硬性约束）。

### ⑤ `QLabel` —— 静态文字

```cpp
durationLabel_ = new QLabel("时长 00:00:00", central);
QFont bold = stateLabel_->font();      // 取出当前字体
bold.setBold(true);                    // 改粗
stateLabel_->setFont(bold);            // 放回去（★ 必须这样，不能 font().setBold()）
```

注意那个"取出 → 修改 → 放回"的三步。`font()` 返回的是**副本**，写 `stateLabel_->font().setBold(true)` 是修改了一个马上销毁的临时对象，**毫无效果** —— 新手经典 3 分钟 bug。

> 顺便指出代码里的一个小瑕疵：这里**用了 `QFont` 却没有 `#include <QFont>`**（第 6~19 行的 include 列表里确实没有）。它能编过是因为 `QLabel` / `QWidget` 的头文件间接把它带进来了。**这叫"隐式依赖"，能跑但不健壮** —— 换个小版本的 Qt 就可能断。好习惯是"用了什么就显式 include 什么"。

### ⑥ `QTimer` —— 定时器（心跳）

```cpp
statsTimer_ = new QTimer(this);        // ★ parent 是 this，跟着窗口销毁
statsTimer_->setInterval(500);         // 500 毫秒
connect(statsTimer_, &QTimer::timeout, this, &MainWindow::pollStats);
// 之后：statsTimer_->start() 启动；statsTimer_->stop() 停止
```

**关键点：`QTimer::timeout` 是在「创建它的那个线程」发出的。** 我们的 timer parent 是 `this`（属于 GUI 线程），所以 **`pollStats()` 是在 GUI 线程里执行的** —— 它里面放心地调用 `videoLabel_->setText(...)`，完全不用投递。

> **反例警告**：如果你真的需要"在另一个线程里跑定时器"（比如 QThread 里循环的定时器），那 `timeout` 槽会在那个线程执行，**里面就不能碰 UI** —— 又回到第 3 章的问题了。判断标准永远是"`timer->thread()` 是谁"。

还有个兄弟叫 **`QTimer::singleShot(ms, this, []{...})`**：只响一次的定时器，很适合做"延迟执行"。

### ⑦ `QElapsedTimer` —— 秒表

```cpp
elapsed_.start();                                  // 开始计时
const qint64 ms = elapsed_.elapsed();              // 现在过了多少毫秒
```

跟 `QTimer` 的区别：`QTimer` 是"定时器，会**发信号**"；`QElapsedTimer` 是"秒表，只**回答问题**"，可以重复无尽次查询。它内部优先用 `QueryPerformanceCounter` 之类高精度时钟。

我们甚至可以用它做**超时控制**（比如" push 20 秒还没连上就算失败"），不必在各组件里手搓超时逻辑。

### ⑧ `QTime` 和 `QString::arg` 的花式用法

```cpp
QTime::currentTime().toString("HH:mm:ss")   // 当前时间 → "20:31:02"
```

`toString` 的格式化字符串：`HH`=24小时补零，`mm`=分，`ss`=秒，`zzz`=毫秒。

```cpp
QString("时长 %1:%2:%3")
    .arg(totalSeconds / 3600, 2, 10, QLatin1Char('0'))
    .arg((totalSeconds / 60) % 60, 2, 10, QLatin1Char('0'))
    .arg(totalSeconds % 60, 2, 10, QLatin1Char('0'));
```

这是 `QString::arg` 的**四参数重载**：

```cpp
QString arg(qlonglong a,          // 值
            int fieldWidth,       // 最小宽度，不够就在左边补
            int base,             // 进制（10 = 十进制）
            QChar fillChar) const // 补什么字符（这里是 '0'）
```

效果：`totalSeconds / 3600 == 0` → 显示 `"00"`（而不是 `"0"`）。这就是**补零右对齐**的标准做法，比手写 `if (s<10) ...` 干净得多。

链式调用的顺序：第一个 `arg` 吃掉 `%1`，第二个吃掉 `%2`……所以顺序必须对应。

> **易错点**：如果写成 `QString("%1 %2").arg(a).arg(b)` 而 `a` 本身包含 `%2`，会被第二次 `arg` 替换掉。复杂场景务必一次性 `arg` 或者用 `QString::arg(a, b)` 的多参数重载。

## 4.7 小结：第四章我们学到了什么

| 概念 | 一句话 |
|---|---|
| 布局管理器 | 替代绝对坐标：`VBox`=竖排、`HBox`=横排、`Form`=两列表单，可任意嵌套 |
| **stretch 因子** | 多余空间的分配比例；0 = 不扩展，1:1 = 平分，1:2 = 三分之一/三分之二 |
| `addSpacing` vs `addStretch` | 前者固定像素，后者"弹簧"（吃多余空间）；最后加弹簧=内容靠左 |
| `QGroupBox` | 带标题边框的分组容器，配 layout 使用 |
| `QComboBox` | 下拉框；**`addItem(text, userData)` 藏了个 QVariant**，显示名和数据一起带着走 |
| `QSpinBox` | 整数输入：`setRange` / `setValue` / `setSuffix` / `setSingleStep` |
| **`QPlainTextEdit` vs `QTextEdit`** | 前者纯文本、快、有 `setMaximumBlockCount`；后者富文本。**日志必须用前者** |
| `QPushButton` | `clicked(bool)` 信号；`setEnabled(false)` 变灰且不再发射 |
| `QLabel` | 静态文字；改字体必须"取出→修改→放回" |
| **`QTimer`** | `timeout` 在**创建它的线程**发出；parent 给 this 就随窗口销毁 |
| `QElapsedTimer` | 秒表，只回答"过了多久"，可反复查询 |
| `arg` 四参数版 | `.arg(值, 最小宽度, 进制, 填充字符)` → 补零对齐 |

---
# 第 5 章 逐行教学（一）：`src/app/CMakeLists.txt` —— 六行里的四个门道

这是本次改动里**最小的文件（14 行 → 6 行）**，但它身上挂着四个很重要的 CMake 知识点。我们先看 diff：

```diff
 add_executable(LightLiveStreamer
   main.cpp
-)
-
-target_include_directories(LightLiveStreamer PRIVATE
-  ${CMAKE_CURRENT_SOURCE_DIR}/..
-  ${FFMPEG_INCLUDE_DIRS}
+  MainWindow.cpp
+  MainWindow.h
 )
 
 target_link_libraries(LightLiveStreamer PRIVATE
-  core_common
+  core_pipeline
   Qt6::Widgets
-  ${FFMPEG_LIBRARIES}
 )
```

改完之后的文件只有 6 行：

```cmake
add_executable(LightLiveStreamer
  main.cpp
  MainWindow.cpp
  MainWindow.h
)

target_link_libraries(LightLiveStreamer PRIVATE
  core_pipeline
  Qt6::Widgets
)
```

## 5.1 门道一：为什么要把 `.h` 写进 `add_executable`

很多同学会问：`.h` 又不是编译单元，写进去干嘛？CMake 不认识它吗？

CMake 当然认识 —— 它有 SKIP 规则，**`.h` 不会参与编译**。写进去的目的只有一个：**喂给 AUTOMOC 扫描**。

回顾第 2.5 节说的：CMake 的 AUTOMOC 为了找"带 `Q_OBJECT` 的头文件"，会扫描这个 target 的源文件列表。`MainWindow.h` 在这个列表里，`Q_OBJECT` 就会被扫到，`moc` 就会被调用生成 `moc_MainWindow.cpp` 并加入编译。

**这是最保险的做法。** 虽然现代 CMake 也能通过"和同名 .cpp 配对的 .h"自动推断，但只要有一天你把头文件重命名了、或者出现了 `A.cpp` include `B.h` 而 `B.h` 里写了 `Q_OBJECT` 的情况，漏网就会以"链接错误"的形式炸出来（第 16 章有完整演示）。

> **一句话**：`add_executable` 里写 `.h` 不是多余，是给 AUTOMOC 导航。我们在第 2.5 节已经强调过一次，这里是用真代码再确认一次。

## 5.2 门道二：`target_include_directories` 为什么被删掉了（依赖方向的进步）

改动之前是这样的：

```cmake
target_include_directories(LightLiveStreamer PRIVATE
  ${CMAKE_CURRENT_SOURCE_DIR}/..      # = src/
  ${FFMPEG_INCLUDE_DIRS}              # = FFmpeg 的头文件
)
```

那时候的 `LightLiveStreamer` 是个命令行自检程序，它**直接 include 了 FFmpeg 的头文件**（`libavcodec/avcodec.h` 等等），所以必须自己手动加 FFmpeg 的 include 路径。

现在呢？`main.cpp` 和 `MainWindow.cpp` 里**一行 FFmpeg 的 include 都没有**，而是靠 `#include "core/pipeline/StreamPipeline.h"` 这种方式访问内核。而这个路径**谁来提供**？

看一眼 `src/core/pipeline/CMakeLists.txt`（这个文件不是本次改动，但它今天终于发挥作用了）：

```cmake
target_include_directories(core_pipeline
  PUBLIC ${CMAKE_CURRENT_SOURCE_DIR}/../..     # ★ PUBLIC！= src/core/pipeline/../.. = src/
  PRIVATE ${FFMPEG_INCLUDE_DIRS}
)
```

**`PUBLIC` 的意思是：这条包含路径既给我自己用，也"传播"给所有链接我的目标。**

于是依赖传递链是这样流动的：

```
LightLiveStreamer  ──link(PRIVATE)──→  core_pipeline
                                            │ PUBLIC include dir: src/
                                            ↓ 自动继承
                                    LightLiveStreamer 拿到 src/ 作为包含路径
                                            │
                                            │ 而 core_pipeline 又 PUBLIC link 了：
                                            ├──→ core_common  (PUBLIC include src/)
                                            ├──→ core_capture (PUBLIC include src/, PRIVATE ffmpeg)
                                            ├──→ core_encode
                                            ├──→ core_output
                                            └──→ winmm（Windows 库）
                                            ↓ 全部自动继承给 LightLiveStreamer
```

**这就是第 1 课讲的"目标建模"思想的兑现**：app 只需要说一句"我依赖 `core_pipeline`"，底下七个库、五个 Windows 系统库、FFmpeg 的链接、头文件路径，全部自动到位。中间层任何一级换了 FFmpeg 装在哪、换了需要多一个系统库，**app 的 CMakeLists 一行都不用改**。

> **对比一下就知道价值**：老的写法里，app 自己手动写了 core_common + FFmpeg 的 include + FFmpeg 的 lib，一共三处"手抄"。这种重复是技术债：一旦 FFmpeg 依赖方式变了，所有手抄过的地方都要改一遍。
>
> **结论：外层的 target 不应该知道内核用什么第三方库。这叫"依赖倒置"，也叫"不要把实现细节泄漏到调用者"。**

## 5.3 门道三：`core_common` → `core_pipeline`：依赖方向更深了

```
改前：LightLiveStreamer → core_common + Qt6::Widgets + FFmpeg
改后：LightLiveStreamer → core_pipeline + Qt6::Widgets
```

这个变化的语义是：**"这个程序现在不再只是'验证第 3 课的契约'了，它要指挥整条流水线。"**

层次上完全说得通。我们把整个项目的依赖图摊开（注意箭头方向）：

```
        ┌────────────────┐
        │  LightLiveStreamer  (src/app)     ← 今天的改动在这
        └────────┬───────┘
                 │ link core_pipeline
                 ↓
        ┌────────────────┐
        │  core_pipeline  │ ← 第 11 课的东西，今天的界面全在指挥它
        └────────┬───────┘
       ┌─────────┼─────────┬──────────┐
       ↓         ↓         ↓          ↓
   core_capture core_encode core_output core_common
   (采集)       (编码)     (封装)     (契约层，谁都依赖它)
```

**注意箭头永远是单向的**：app 知道 pipeline，pipeline 不知道 app；core 层谁都不知道 Qt Widgets 的存在（`StreamPipeline.h` 里 `#include` 的只有 `Common.h`、`<memory>`、`<string>`）。

这条规矩一旦破了会怎样？假设哪天有人想在采集模块里弹个 Qt 消息框 —— 那 `core_capture` 就得 link `Qt6::Widgets`，然后所有用到它的目标都被迫依赖 Qt。**测试程序、未来的命令行工具全部被拖下水。**所以我们宁可通过 `LogCallback`（一个 `std::function`）把话传出来，也不让内核碰界面。今天 `postLog` 之所以存在，根就在这个设计决定上。

## 5.4 门道四：这个文件里"没有出现"的东西，也是一种设计

大家扫一眼这 6 行，看看有什么**故意没写**的：

| 没写的东西 | 为什么没写 | 什么时候要写 |
|---|---|---|
| `target_include_directories(... Qt6)` | `Qt6::Widgets` 这个 target 自己带着 INTERFACE include dirs，链接它就自动有了 | 从不 |
| `target_compile_features(cxx_std_17)` | 根 CMakeLists 里全局设了 `CMAKE_CXX_STANDARD 17`；而且 `Qt6::Widgets` 也要求 C++17 | 从不（这个项目） |
| **`WIN32_EXECUTABLE`** | **本次没加！** | 见下面的重要提醒 |
| `set_target_properties(... VS_DEBUGGER_ENVIRONMENT ...)` | 为了方便大家 clone 后能直接跑 | 想在 VS 里一键调试 Qt exe 时（需要把 Qt 的 bin 加进调试环境） |

> **★ 一个今天留下的小尾巴：`WIN32_EXECUTABLE` 没加。**
>
> CMake 的 `add_executable` 默认生成的是**控制台子系统**（`/SUBSYSTEM:CONSOLE`）程序 —— 因为我们的 `main()` 就是普通 `int main(...)`。结果是：**双击 `LightLiveStreamer.exe` 时，会先弹出一个黑乎乎的控制台窗口，然后才是我们的 Qt 界面。** 而且（虽然本项目现在没人 printf 了）控制台窗口关掉会连带把 Qt 窗口也关掉（点了没注意就中招）。
>
> 正确的做法，是在 `add_executable` 后面加一行：
>
> ```cmake
> set_target_properties(LightLiveStreamer PROPERTIES WIN32_EXECUTABLE TRUE)
> ```
>
> 或者在 `add_executable(LightLiveStreamer WIN32 main.cpp ...)` 里写 `WIN32`。
>
> 加了之后子系统变成 Windows，`main()` 仍然合法（Qt 提供 `qtmain` 入口适配），**双击就不再有黑窗口了**。代价是：`printf` 的输出看不见了（调试要看 IDE 的输出窗口或者改用 `OutputDebugString`）。
>
> 我们把这个留给大家在第 15 章的"改造实验"里亲手加上去试试。**记住这个现象，将来你们自己写 GUI 一定会遇上。**

## 5.5 自检验收：看看 CMake 到底有没有跑 moc

讲了半天 AUTOMOC，我们给个"眼见为实"的检查方法。配完项目之后：

```bat
cmake -B build -S . -DCMAKE_TOOLCHAIN_FILE=C:/vcpkg/scripts/buildsystems/vcpkg.cmake
```

然后去构建目录里找：

```
build/src/app/
├── LightLiveStreamer_autogen/
│   ├── include/
│   │   └── ui_MainWindow.h?          （本项目没用 .ui 文件，所以没有）
│   └── mocs_compilation.cpp          ← ★ AUTOMOC 生成的汇总文件
│       └── #include "moc_MainWindow.cpp"
└── CMakeFiles/
    └── LightLiveStreamer.dir/
        └── LightLiveStreamer_autogen/
            └── moc_MainWindow.cpp    ← ★ moc 的产出
```

**如果没看到 `moc_MainWindow.cpp`，说明 moc 没跑**，那就去第 16 章查表。

---

# 第 6 章 逐行教学（二）：`main.cpp` —— 从 132 行自检程序砍到 17 行

## 6.1 先看看被删掉的那 129 行是什么

这次改动删掉了 `main.cpp` 里的 129 行。**删之前**它是第 1 课写的"环境自检程序"，干的事同学们可能还有印象：

```cpp
// （旧版 main.cpp 的功能概览，已删除）
int main()
{
    std::system("chcp 65001 > nul");            // 控制台切 UTF-8
    std::printf("  LightLiveStreamer 环境自检\n");

    // [1/4] Qt：查 qVersion() 和 QT_VERSION_STR 是否一致
    // [2/4] FFmpeg：打印 libavcodec / libavformat / ... 的版本
    // [3/4] 推流协议：用 avio_enum_protocols 枚举，看有没有 "rtmp" 输出协议
    // [4/4] 编码器：avcodec_find_encoder_by_name("libx264") 和 AAC 有没有
    //       （没有的话，打印 vcpkg install ffmpeg[gpl,x264] 的提示）
    return ok ? 0 : 1;
}
```

**它是一个合格的临时工**：在项目第一天，它用 132 行证明了"Qt 装好了、FFmpeg 装好了、x264 有了、rtmp 协议有了"，让后面十课都能安心开工。它甚至还贴心地处理了 Windows 控制台中文乱码（`chcp 65001`）。

今天它被删了 —— 因为**任何一个带界面的程序，入口 `main` 都应该干且只干一件事：把应用程序对象建起来，把主窗口显示出来，然后进入事件循环。**

> **老师的一点感慨**：同学们做项目的时候要允许这种代码存在 —— 它叫**脚手架**（scaffolding）。脚手架的生命周期就是"从打地基到封顶"，封了顶就该拆。把它删掉不丢人，把它永远留在 `main` 里才是问题。
>
> **顺便记一笔债务**：这个自检程序被删了，意味着**以后换机器、重新配环境，就没有一键自检的手段了**，只能等编译/运行时报错。如果要补，正确做法不是塞回 `main.cpp`，而是**单独做一个 target**（比如 `add_subdirectory(tools/env-check)`），保留它但和主程序解耦。这个留作第 15 章的思考。

## 6.2 新版 `main.cpp` 全文（17 行）逐行讲解

```cpp
 1  #include "MainWindow.h"
 2
 3  #include <QApplication>
 4
 5  int main(int argc, char** argv)
 6  {
 7      QApplication app(argc, argv);
 8
 9      app.setApplicationName("LightLiveStreamer");
10      app.setApplicationDisplayName("LightLiveStreamer");
11      app.setOrganizationName("LightLiveStreamer");
12
13      MainWindow window;
14      window.show();
15
16      return app.exec();
17  }
```

### 第 1 行：`#include "MainWindow.h"`

用双引号而不是尖括号 —— 编译器先在**当前文件所在目录**找，找不到才去系统路径。这是包含"自己的头文件"的标准写法。

### 第 3 行：`#include <QApplication>`

**`QApplication`** 是 Qt Widgets 程序的"应用程序对象"。它负责：

- 解析命令行里 Qt 认识的参数（比如 `-style fusion`、`-platform windows`）；
- 管理**主事件队列**（第 1.3 节那个 while 循环就跑在它身上）；
- 管理全局资源：字体数据库、样式、剪贴板、屏幕信息、会话管理；
- 提供 `applicationName` / `organizationName` 之类的全局属性。

**注意**：如果你写的是**不带界面的**控制台 Qt 程序（只用 `QString`、`QTimer` 之类），用 **`QCoreApplication`** 就够，不用拉进来整套 Widgets。`QApplication` 是它的子类，多了 GUI 相关的一切。

> **必须在创建任何 `QWidget` 之前创建 `QApplication`**（包括 `QPixmap`、`QFont` 这类看起来无害的东西）。反过来，也**必须在最后一个 QWidget 销毁之后才销毁 `QApplication`**。我们下面的代码顺序正好满足这两条。

### 第 5 行：`int main(int argc, char** argv)`

注意老版本是 `int main()`。为什么会变成带参数？

因为 **`QApplication` 要吃命令行参数**：`QApplication app(argc, argv)` 会把 Qt 认识的参数（比如 `-style`、`-platformtheme`）**从 argv 里摘掉**，剩下的才是你的程序该处理的。这是一次协作：Qt 帮你过滤掉了它该管的东西。

（注意它是按**引用**接收 `argc/argv` 的，所以 Qt 真的会改这两个变量。）

> 顺带科普：Windows 上还有一个宽字符版本入口 `wWinMain`/`GetCommandLineW`，Qt 通过 `qtmain` 库做了适配，所以你照样可以写标准 `int main`。这也是为什么 Qt 程序能跨平台保持一致。

### 第 7 行：`QApplication app(argc, argv);`

**栈对象**。它的生命周期从这里开始，到 `main` 函数结束时结束。

这一行的位置很讲究，必须满足两条约束（前面提过，这里用代码证实）：

```cpp
QApplication app(argc, argv);     // ① 必须最早：创建任何 QWidget 之前
MainWindow window;                // ② window 在 app 之后创建
...
return app.exec();                // ③ app.exec() 返回后：
                                  //    window（后创建的）先析构 → app（先创建的）后析构 ✅
```

C++ 的规则是"**后创建的对象先销毁**"。因为 `app` 先创建，所以 `window` 会在 `app` 之前析构 —— **完全符合 Qt 的要求**（所有 widget 必须在 QApplication 之前销毁）。

> 如果反过来写，会发生什么？我在这里帮同学踩过坑：某些 Qt 版本会在 `~QWidget` 里访问已经被销毁的 `QApplication` 全局单例，导致**退出时崩溃**。这种"关闭程序才崩"的 bug 特别难查，因为调试器只给你一个看不懂的调用栈。**记住顺序，别写反。**

### 第 9~11 行：三个 `setXxxName`

```cpp
app.setApplicationName("LightLiveStreamer");
app.setApplicationDisplayName("LightLiveStreamer");
app.setOrganizationName("LightLiveStreamer");
```

这三个看起来只是"起名字"，实际上它们决定了 **Qt 的标准路径**：

| 属性 | 影响什么 |
|---|---|
| `applicationName` | ① 某些系统（尤其是 Linux）的窗口标题默认值；② **`QSettings` 默认配置文件的文件名** |
| `applicationDisplayName` | ① **Windows 任务栏上显示的名字**；② 窗口标题在某些样式下的兜底 |
| `organizationName` | **`QSettings` 默认路径的组织目录名**（Windows 上是注册表 `HKEY_CURRENT_USER\Software\<组织名>\<应用名>` 或 ini 的路径） |

**今天这三行完全用不上**（我们还没做配置持久化），但它们是为**第 15 课（`7eb889b` 配置持久化）**提前铺的路 —— 到时候 `QSettings settings;` 不传任何参数就能自动落到正确的位置。

> **这是值得学习的一点：当你知道下一件事要来的时候，可以先把它的"基础设施"写对，成本是三行，收益是未来少踩一个坑。** 但请注意分寸 —— 我们只是设了三个名字，**没有**提前写好一整套配置读写。**预铺最小基础设施，不提前实现完整功能**，这是很好的平衡。

### 第 13 行：`MainWindow window;`

构造主窗口。注意此时窗口**还没有显示**，只是对象建好了（第 9 章会看到构造函数里干了什么：建控件、连信号、枚举设备、往日志框写第一行字）。

这里用的是**栈对象**。整个 Qt 里"顶层窗口用栈、窗口内部控件用 new+parent"是最常见的搭配。理由：

- 顶层窗口的生命周期本来就明确（跟 `main` 一致），放栈上最省心；
- 而且**它必须比 `app` 早销毁**，放栈上天然满足（后构造先析构）。

### 第 14 行：`window.show();`

`show()` 的意思是"请显示"。注意 **它不是立刻把窗口画到屏幕上**（真正画要等事件循环开始处理那个 ShowEvent/PaintEvent），它做的事是：

1. 把 widget 标记为"可见"（`setVisible(true)`，而下层的 `QApplication::setQuitOnLastWindowClosed` 之类机制会关注这个）；
2. 安排一个"显示/重画"事件进队列；
3. 如果是顶层窗口，还可能触发"把 Qt 窗口对象和底层原生窗口句柄（Win32 的 HWND）关联起来"的动作。

> **冷知识**：`QWidget` 默认 `visible = false`。所以你 `new` 出来一堆控件但忘了 `show()` 或者忘了把它们塞进已显示的布局里，结果就是"程序跑了但啥也没有"—— 这也是新手常见景象。

### 第 16 行：`return app.exec();`

**这是整个 GUI 程序的"开始"，也是它的"终点"。**

```cpp
int QApplication::exec()
{
    // 简化版伪码
    if (!eventLoopIsRunning) {
        enterMainEventLoop();        // ← 从这里进去，一般不回来
    }
    return exitCode;                 // ← 直到有人 call quit()/exit()，才走到这里
}
```

- 调用 `exec()` 之后，`main` 函数的栈就**停在这一行不动了**，直到用户关闭最后一个窗口（或者代码里调用 `app.quit()`），事件循环退出，`exec()` 才返回一个 int（退出码），然后 ① 栈对象 window 析构 ② 栈对象 app 析构 ③ `main` return。
- **界面里发生的每一件事，都是在这条调用栈的深处被执行的。** 你在调试器里 pause 一下，会发现调用栈的最底下通常就是 `QApplication::exec()` → `QEventLoop::exec()` → …… 这个轮廓要认得。
- **所有自定义代码都直接或间接被它调用**（这就是所谓的"好莱坞原则"：不要打电话给我们，我们会打给你）。

> **一个常见误解**：有同学以为 `app.exec()` 之后的代码是"退出后执行"。没错 —— 但请注意，`return app.exec();` 后面（本例第 17 行就是 `}`）能放的代码非常有限，因为那时所有对象都已经在析构的路上了。**清理工作应该放在对象的析构函数里，或者 `aboutToQuit` 信号连接的槽里，而不是写在 `exec()` 后面。**

## 6.3 小结：第 6 章的九要点

| # | 知识点 | 具体说明 |
|---|---|---|
| 1 | `QApplication` | GUI 程序的总管家，必须在任何 QWidget 之前创建 |
| 2 | `QCoreApplication` | 无界面版的 Application（不用 Widgets 时用） |
| 3 | `argc/argv` 传给它 | Qt 会摘掉它认识的参数（`-style` 等） |
| 4 | **创建/销毁顺序** | app 最先创建、最后销毁；window 夹在中间 |
| 5 | 三个 setName | 影响窗口标题、任务栏名字、**`QSettings` 默认路径**（给第 15 课铺路） |
| 6 | 栈上 window | 顶层窗口用栈，内部控件用 new+parent |
| 7 | `show()` 的含义 | 只是"请求显示 + 排队"，真正绘制在事件循环跑起来之后 |
| 8 | **`app.exec()`** | 进入主事件循环，阻塞在此；所有 slot 都是从这里被调用的 |
| 9 | 脚手架该拆就拆 | 旧的 132 行自检程序完成使命后删掉；若还要用，做成独立 target |

---

# 第 7 章 逐行教学（三）：`MainWindow.h`（86 行）

这是本课唯一一个"完全是新东西"的文件。看起来只是一个类声明 86 行，但**它把这个类要做的事、处于什么状态、靠什么数据活着，全部表达清楚了**。我们一行一行读。

## 7.1 全文

```cpp
 1  #pragma once
 2
 3  #include "core/common/Common.h"
 4  #include "core/pipeline/StreamPipeline.h"
 5
 6  #include <QElapsedTimer>
 7  #include <QMainWindow>
 8
 9  #include <atomic>
10  #include <mutex>
11  #include <thread>
12
13  class QComboBox;
14  class QFormLayout;
15  class QHBoxLayout;
16  class QLabel;
17  class QPlainTextEdit;
18  class QPushButton;
19  class QSpinBox;
20  class QTimer;
21
22  class MainWindow : public QMainWindow {
23      Q_OBJECT
24
25  public:
26      explicit MainWindow(QWidget* parent = nullptr);
27      ~MainWindow() override;
28
29  protected:
30      void closeEvent(QCloseEvent* event) override;
31
32  private:
33      void buildUi();
34
35      void refreshVideoDevices();
36      void refreshAudioDevices();
37      void updateSourceFields();
38
39      void startStreaming();
40      void stopStreaming();
41      void updateUiState();
42
43      void pollStats();
44      void appendLog(const QString& message);
45      void postLog(const std::string& message);
46
47      StreamConfig collectConfig() const;
48
49      QFormLayout* videoForm_ = nullptr;
50      QHBoxLayout* cameraRow_ = nullptr;
51      QComboBox* videoSourceCombo_ = nullptr;
52      QComboBox* cameraCombo_ = nullptr;
53      QPushButton* refreshCameraButton_ = nullptr;
54      QComboBox* resolutionCombo_ = nullptr;
55      QSpinBox* fpsSpin_ = nullptr;
56      QSpinBox* bitrateSpin_ = nullptr;
57
58      QComboBox* audioSourceCombo_ = nullptr;
59      QComboBox* audioDeviceCombo_ = nullptr;
60      QPushButton* refreshAudioButton_ = nullptr;
61
62      QPlainTextEdit* targetsEdit_ = nullptr;
63
64      QPushButton* startButton_ = nullptr;
65      QPushButton* stopButton_ = nullptr;
66      QLabel* stateLabel_ = nullptr;
67
68      QLabel* durationLabel_ = nullptr;
69      QLabel* videoLabel_ = nullptr;
70      QLabel* audioLabel_ = nullptr;
71      QLabel* targetLabel_ = nullptr;
72
73      QPlainTextEdit* logView_ = nullptr;
74      QTimer* statsTimer_ = nullptr;
75
76      StreamPipeline pipeline_;
77
78      std::mutex pipelineMutex_;
79
80      std::atomic<bool> starting_{false};
81      std::atomic<bool> running_{false};
82
83      std::thread worker_;
84
85      QElapsedTimer elapsed_;
86  };
```

## 7.2 第 1~11 行：include 的三段分组

```cpp
#pragma once                                 // 第 1 行：头文件保护

#include "core/common/Common.h"              // 第 3 行
#include "core/pipeline/StreamPipeline.h"    // 第 4 行

#include <QElapsedTimer>                     // 第 6 行
#include <QMainWindow>                       // 第 7 行

#include <atomic> <mutex> <thread>           // 第 9~11 行
```

三段各有明确理由，**不是随手堆的**：

**第一段（自己的内核头文件）**：

- `Common.h` 提供 `StreamConfig`、`SourceType`、`AudioSourceType`、`StreamTarget` —— 供第 47 行的 `collectConfig()` 返回值使用；
- `StreamPipeline.h` 提供 `StreamPipeline` 类型 —— 供第 76 行的成员 **`StreamPipeline pipeline_;`** 使用。

**这里有个必须讲清楚的技术点**：`pipeline_` 是**值成员**（不是指针），所以编译器必须在这里看到 `StreamPipeline` 的**完整定义**（知道它有多大、怎么构造）。这就是为什么**必须 include 它的头文件**，而不能只写前向声明 `class StreamPipeline;`。

> 对比第 8 课学过的 **PImpl 惯用法**：`StreamPipeline` 自己把 `Impl` 藏在不透明指针后面，所以它的头文件干干净净，我们 include 它不会把 FFmpeg/Windows 的头文件拖进来。**这就是 PImpl 的红利之一 —— 我们今天在 app 层享受到了。**

**第二段（Qt 头文件）**：只有两个，同样是"按需、最小化"：

- `QMainWindow` —— 基类，必须有完整定义（要继承）；
- `QElapsedTimer` —— 第 85 行的成员 `elapsed_` 是**值成员**，必须有完整定义。

**第三段（C++ 标准库）**：`<atomic>`（80/81 行）、`<mutex>`（78 行）、`<thread>`（83 行）—— 都是值成员，必须完整定义。

## 7.3 第 13~20 行：八个前向声明 —— "编译防火墙"

```cpp
class QComboBox;
class QFormLayout;
class QHBoxLayout;
class QLabel;
class QPlainTextEdit;
class QPushButton;
class QSpinBox;
class QTimer;
```

这是 Qt 代码里非常标志性的八行。它们声明了 8 个 Qt 类**存在**，但不引入它们的定义。

**为什么可以这么做？** 因为下面第 49~74 行的成员**全部是裸指针**：

```cpp
QComboBox* videoSourceCombo_ = nullptr;   // 指针大小固定 8 字节，不需要知道类的大小
```

编译器只需要知道"有 `QComboBox` 这个类型"就能声明指向它的指针。**只有当你要解引用（`combo_->addItem(...)`）、或者知道它有多大时才需要完整定义** —— 那些操作都发生在 `MainWindow.cpp` 里，那里 include 了 `<QComboBox>` 等具体头文件（第 7~19 行）。

**这么做的好处是什么？（很重要，请听懂）**

| 好处 | 说明 |
|---|---|
| **编译速度** | 任何 `#include "MainWindow.h"` 的文件（目前只有 `main.cpp`）不会被迫拉进 8 个 Qt 控件的完整头文件（每个后面都跟了一大串） |
| **降低耦合** | 如果你重构界面，把 `QComboBox` 换成 `QListView`，**只要 `.cpp` 改，`MainWindow.h` 一行不动**，所有 include 它的文件都不用重编译 |
| **避免循环依赖** | 头文件互相包含时的救命稻草 |

> 这个技巧叫**编译防火墙**（compilation firewall），是比 PImpl 轻量得多的一个版本：**"不需要知道大小的东西，统统只前向声明。"** 只有"值成员 + 基类"这两类必须 include 完整定义。请大家对照 7.2 节列表体会差别 —— **这也是为什么文件名叫 `.h` 却能保持 86 行的原因。**

**一个细节**：`QCloseEvent` 没有出现在前向声明里！第 30 行的 `closeEvent(QCloseEvent* event)` 只用了它的指针，为什么不需要前向声明？因为 **`QMainWindow.h` 已经间接帮我们声明过它了**（`QWidget` 里有 `virtual void closeEvent(QCloseEvent* event);` 声明，所以它的头肯定已经前向声明过这个类型）。这也是"能用但不保险"的隐式依赖 —— 为了严谨，加上 `class QCloseEvent;` 更好（我们第 4.6 节讲 `QFont` 时说过同类问题）。

## 7.4 第 22~23 行：类声明与 `Q_OBJECT`

```cpp
class MainWindow : public QMainWindow {
    Q_OBJECT
```

**继承关系**：`MainWindow` **是一个** `QMainWindow`（Qt 的"主窗口"类）。

`QMainWindow` 是干什么用的？它自带了一套标准桌面程序的"骨架槽位"：

```
┌─────────────── QMainWindow（自带骨架）───────────────┐
│  MenuBar 菜单（← setMenuBar()）                       │
│  ToolBar 工具栏 ×N（← addToolBar()）                  │
│  ┌──────────── Central Widget ★ 必须有 ─────────────┐ │
│  │                你真正的内容放这                   │ │
│  │               （← setCentralWidget()）            │ │
│  └──────────────────────────────────────────────────┘ │
│  Dock Widgets 停靠窗 ×N（← addDockWidget()）          │
│  StatusBar 状态栏（← statusBar()）                    │
└──────────────────────────────────────────────────────┘
```

今天我们只用了一个槽位 —— **中央部件**（`setCentralWidget(central)`，见第 163 行）。后面几课会加更多（第 13 课的实时预览很可能就是往这里再塞一个 PreviewWidget）。

> **`Q_OBJECT` 在第 23 行，是类体里的第一行。** 这个位置是规矩（必须在 `private:` 之类的访问控制之前，或者说必须在类体开头）。忘记写它，编译器会给你一堆关于 `staticMetaObject` / `metaObject()` 的链接错误，见第 16 章。

## 7.5 第 25~30 行：public 和 protected 各两行，信息量很大

```cpp
public:
    explicit MainWindow(QWidget* parent = nullptr);   // 第 26 行
    ~MainWindow() override;                           // 第 27 行

protected:
    void closeEvent(QCloseEvent* event) override;     // 第 30 行
```

### 第 26 行：`explicit MainWindow(QWidget* parent = nullptr);`

三个知识点：

1. **`parent = nullptr`**：默认没有父亲（顶层窗口）。所有 Qt 窗口类的构造函数都是这个风格的参数表。
2. **`explicit`**：禁止隐式转换。`explicit` 的作用是防止编译器偷偷写出 `MainWindow w = someWidgetPointer;` 这种"自动把 QWidget* 变成 MainWindow"的魔法。**单参数的构造函数都应该加 `explicit`** —— 这是 C++ 的老规矩，可以预防一类很难发现的 bug。
3. 构造函数签名里的 `: QMainWindow(parent)`（在 .cpp 第 23 行体现）意味着**把 parent 传给基类**，这样父对象树就接上了。

### 第 27 行：`~MainWindow() override;`

析构函数上有 **`override`** —— 有意思吗？析构函数是虚函数（继承自 QObject），覆写它理论上不需要 `override` 关键字。写上它的好处是：**如果将来 `QMainWindow` 不再有虚析构函数了，编译器会立刻报错提醒你。** 属于"便宜的未来保险"。

**更重要的是：这个析构函数必须在 .cpp 里定义**（第 35 行），不能写成 `~MainWindow() = default;` 放在头文件里。原因有两个：

1. **要在这里 join 线程**（防止 `this` 销毁后工作线程还拿着它用 —— 第 3.8 节的纪律三）；
2. （通用原则）如果将来把某个成员改成不完整类型（`std::unique_ptr<T>` 之类），inline 的析构函数会在错误的位置实例化，导致编译错误。**这也是第 10 课讲过的"PImpl 的析构不能写进头文件"的同一条原理。**

### 第 30 行：`closeEvent(QCloseEvent* event) override`

这是 Qt 提供的**事件处理钩子**：用户点右上角的 ×、按 Alt+F4、或者右键任务栏选"关闭"的时候，**Qt 会把这个动作变成一个 `QCloseEvent` 事件发给窗口**，让你的 `closeEvent()` 有机会处理。

它的威力在于 `event` 参数：

```cpp
event->accept();    // 同意关闭（默认行为）
event->ignore();    // ★ 拒绝关闭！窗口不会被关掉
```

常见的用法：

```cpp
void MainWindow::closeEvent(QCloseEvent* event)
{
    if (正在推流) {
        QMessageBox::question(this, "确认", "还在推流，确定要退出吗？") == Yes
            ? event->accept() : event->ignore();
        return;
    }
    event->accept();
}
```

我们今天的实现没有弹窗，而是**直接保证"安全关闭"**（停定时器 → 停管线 → join 线程），见第 14.4 节。

## 7.6 第 32~47 行：私有方法的四个分组（这个类的全部能力）

```cpp
private:
    void buildUi();                              // 组 1：搭积木

    void refreshVideoDevices();                  // 组 2：设备枚举
    void refreshAudioDevices();
    void updateSourceFields();

    void startStreaming();                       // 组 3：状态机
    void stopStreaming();
    void updateUiState();

    void pollStats();                            // 组 4：输出
    void appendLog(const QString& message);
    void postLog(const std::string& message);

    StreamConfig collectConfig() const;          // 组 5：翻译
```

请把这**四个徬上的十一处理方法**看成这个类的全部对外能力。它们的分组是有逻辑的：

| 组 | 方法 | 谁调用它 | 什么时候调用 |
|---|---|---|---|
| **① 搭建** | `buildUi()` | 构造函数，**仅一次** | 对象构造时 |
| **② 刷新** | `refreshVideoDevices()` / `refreshAudioDevices()` / `updateSourceFields()` | 构造函数 + 三个信号（刷新按钮、来源切换） | 需要重新枚举设备/调整可见行时 |
| **③ 状态机** | `startStreaming()` / `stopStreaming()` / `updateUiState()` | 按钮点击槽 + 内部调用 | 推流开始/结束/状态变化时 |
| **④ 输出** | `pollStats()` / `appendLog()` / `postLog()` | 定时器（500ms）/ 任意地方 / **任意线程** | 周期性 / 有话说时 |
| **⑤ 翻译** | `collectConfig() const` | `startStreaming()` | 点开始那一刻 |

**注意三个设计上的细节**：

- 组 ② 里，`refreshVideoDevices` 和 `refreshAudioDevices` **既是槽（被按钮信号的 connect 指向）又是普通函数**（构造函数直接调）—— Qt 的类里这种情况很常见，**槽就是普通成员函数**，没有什么特殊的标记（因为我们已经不带 `slots:` 关键字了）。
- 组 ④ 里，`postLog` 是**唯一一个被设计成"跨线程安全"的方法**（内部做投递），而 `appendLog` 是**只允许 GUI 线程调用**的方法。两者的分工是刻意的 —— 这是一个非常重要的 API 设计约定：**带 `post` 前缀 = 线程安全入口，可以直接从任意线程调用；不带 = 只能在 GUI 线程调用。** 命名即文档。

> 但请同学们注意：这个约定靠的是**人的纪律**，编译器不会帮你检查。更严格的做法是写成 `void appendLogGui(QString)` 或者加断言 `Q_ASSERT(QThread::currentThread() == thread())`。我们在第 15 章的实验里会亲手加一条这样的断言试试。

- 组 ⑤ 的 `collectConfig() const` 是唯一的 `const` 成员函数 —— 因为它**只读界面的当前值**，不修改任何东西（连界面也不改）。这是很好的 const-correctness 实践（如果你的某个成员函数不修改对象状态，就该标 `const`）。

## 7.7 第 49~74 行：所有界面成员的"分区摆放"

这 26 行看着重复，但它的组织方式就是**照着界面的视觉区块顺序**来的：

```cpp
    // ── 视频源组（左）────────────────────────────
    QFormLayout* videoForm_;          // 表单布局本身也要存！因为要 setRowVisible
    QHBoxLayout* cameraRow_;          // 设备那一行也要存！因为要整行隐藏
    QComboBox* videoSourceCombo_;     // 来源：屏幕/摄像头
    QComboBox* cameraCombo_;          // 设备名
    QPushButton* refreshCameraButton_;// 刷新
    QComboBox* resolutionCombo_;      // 分辨率
    QSpinBox* fpsSpin_;               // 帧率
    QSpinBox* bitrateSpin_;           // 码率

    // ── 音频源组（右）────────────────────────────
    QComboBox* audioSourceCombo_;     // 来源：系统声音/麦克风
    QComboBox* audioDeviceCombo_;     // 设备名
    QPushButton* refreshAudioButton_;

    // ── 推流地址区 ──────────────────────────────
    QPlainTextEdit* targetsEdit_;

    // ── 控制区 ──────────────────────────────────
    QPushButton* startButton_;
    QPushButton* stopButton_;
    QLabel* stateLabel_;

    // ── 统计区 ──────────────────────────────────
    QLabel* durationLabel_;
    QLabel* videoLabel_;
    QLabel* audioLabel_;
    QLabel* targetLabel_;

    // ── 日志区 ──────────────────────────────────
    QPlainTextEdit* logView_;
    QTimer* statsTimer_;
```

**三条必须注意的设计点：**

**① 为什么有的布局对象要存成成员，有的不用？**

对比这四处：

```cpp
videoForm_ = new QFormLayout(videoGroup);   // ★ 存成成员
cameraRow_ = new QHBoxLayout;              // ★ 存成成员
auto* audioForm = new QFormLayout(audioGroup);   // 局部变量，用完就扔
auto* targetLayout = new QVBoxLayout(targetGroup);  // 局部变量
auto* controlLayout = new QHBoxLayout;        // 局部变量
auto* statsLayout = new QHBoxLayout;          // 局部变量
auto* logLayout = new QVBoxLayout(logGroup);  // 局部变量
```

判断标准只有一条：**"这个对象在 `buildUi()` 结束之后还会被用到吗？"**

- `videoForm_` 和 `cameraRow_` → **会**（`updateSourceFields()` 第 223 行要用它们来整行隐藏）→ 存成成员；
- 其他七个布局 → 只在 `buildUi()` 内部 用一次之后就没人引用 → 局部变量就够了。**它们也不需要 `delete`，因为它们 install 到了各自的父 widget 上，由对象树负责回收。**

> 这个判断法则非常实用：**"只有需要跨函数访问的 UI 对象才做成成员。"** 反过来如果你把 20 个控件全都做成成员，头文件的可读性会烂掉 —— 读代码的人分不清哪些重要。**成员变量应该少而精。**

**② 所有指针成员都初始化成 `nullptr`**

注意第 49~74 行每个成员后面都跟着 `= nullptr`。为什么？

这是一个**廉价的生命保险**。考虑这个场景：如果某天有人在构造函数里忘了给某个控件赋值（比如注释掉了一行），而 `updateUiState()` 又去调用它的方法 —— 如果是野指针，程序的行为是随机的；如果是 `nullptr`，至少：

1. 调试器里一眼看得出来它是空的；
2. 加上一句 `if (!ptr) return;` 就能优雅降级；
3. 某些 Qt 版本会给出明确的错误信息而不是随机崩溃。

**"成员指针一律初始化"应该成为肌肉记忆。**

**③ 命名风格：`xxx_` 后缀下划线**

这是本项目内核代码一直用的风格（`impl_`、`logMutex_`…），界面接着用。**下划线的位置很重要**：

- **`name_`**（下划线在后）：Qt / STL / Google 风格之一；
- **`_name`**（下划线在前）：**不推荐**，因为 C++ 标准规定"以下划线开头 + 大写字母"和"包含双下划线"的标识符被保留给编译器/标准库，容易踩到;
- **`m_name` / `mName`**：也是常见派系。

没有对错，**关键是同一个项目里保持一致**。我们统一用 `xxx_`。

## 7.8 第 76~85 行：非界面成员 —— 这个类的"状态"

最后十行是整个类的**灵魂**所在地，因为它们承载了"推流进行到哪一步"这件事：

```cpp
    StreamPipeline pipeline_;            // 第 76 行：真正干活的那个人（值成员）

    std::mutex pipelineMutex_;           // 第 78 行：保护对 pipeline_ 的并发访问

    std::atomic<bool> starting_{false};  // 第 80 行：正在启动中？
    std::atomic<bool> running_{false};   // 第 81 行：已经跑起来了？

    std::thread worker_;                 // 第 83 行：执行"启动/停止"的那个临时工

    QElapsedTimer elapsed_;              // 第 85 行：秒表，算"已经推了多久"
```

我们逐个讲透，因为**这几个成员的组合，定义了这个界面的状态机**。

### 7.8.1 `StreamPipeline pipeline_;` —— 干活的是它，界面只是遥控器

类型来自第 11 课，是我们流水线的总控。它的公开接口（供我们今天用）：

| 接口 | 作用 |
|---|---|
| `bool start(config, LogCallback)` | **阻塞式**启动：打开编码器、打开输出、打开采集、拉起三条线程。**可能耗时几秒** |
| `void stop()` | **阻塞式**停止：通知三条线程退出并 join、flush、清理。**可能耗时几百毫秒** |
| `PipelineStats stats() const` | 取一份统计快照（读的都是 atomic，很快） |
| `const std::string& lastError() const` | 失败原因 |
| `bool isRunning() const` | 是否在运行 |
| `int reconnectDisconnected()` | （第 12 课还没用到，第 14 课的主角） |

**注意两次"阻塞式"** —— 这两个字串起了本课所有关于线程的决策。它们是 `startStreaming()` 要开一条 `std::thread` 的**唯一原因**。

### 7.8.2 `std::atomic<bool> starting_` 和 `running_` —— 为什么要 atomic

这是同学们最容易问的两个问题：**"为什么用 atomic 不用 bool？"** 和 **"为什么是两个，不是一个？"**

**为什么是两个？** 因为界面的状态其实是**三态**的，不是二态：

```
      ┌──────────────┐   点开始     ┌──────────────┐   启动成功   ┌──────────────┐
      │   未开始     │ ──────────→ │   连接中      │ ──────────→ │    推流中     │
      │ Setup: 全可编辑│            │ Starting    │             │ Running      │
      │ Stop: 灰      │            │ 全部灰掉     │             │ Stop: 可用    │
      └──────────────┘             └──────┬───────┘             └──────┬───────┘
            ↑                             │ 启动失败（几秒后才知道）        │ 点停止
            └─────────────────────────────┘                            ↓
                                      写日志"启动失败: xxx"        ┌──────────────┐
                                                                   │  停止中      │
                                                                   │  (异步)     │
                                                                   └──────────────┘
```

**`starting_` 是给"连接中"这个中间状态用的**：从用户点下按钮，到连接真的建立起来，中间可能要几秒。这几秒里你不能让用户再点一次开始，也不能让用户去改分辨率。所以必须有一个独立的状态位。

如果只有 `running_`，那么"点了开始但还没连上"这个中间状态就无法表达 —— 界面会出现"还没连上但按钮又亮了"的空窗期，所以必须老老实实用两个状态位。

**为什么必须是 `std::atomic<bool>` 而不是 `bool`？** 两个缺一不可的理由：

**理由 1（技术层）：跨线程读写同一个非 atomic 变量 = 数据竞争 = 未定义行为。**

`starting_`/`running_` 会被**多个线程**读写：GUI 线程（槽）、工作线程（`startStreaming` 的 lambda）。C++ 标准对"一个线程写、另一个线程读且没有同步"的说法是：**未定义行为**。不是"可能读到旧值"那么温和，而是**编译器可以做任何它想做的事**——包括把读操作提到循环外面（因为编译器假设没人改它）、包括读到撕裂的值。

编译器优化的例子（非常真实）：

```cpp
// 假设 running_ 是普通 bool
if (running_) return;          // 编译器可能把 running_ 缓存在寄存器里
// ...
// 之后再次读 running_，编译器可能直接用缓存值，不再去内存里取！
```

**理由 2（工程层）：atomic 自带正确的内存序，不用加它的锁。**

默认的 `std::atomic<bool>` 操作是 `memory_order_seq_cst`（顺序一致性）—— 这是**最强、最容易推理**的内存序保证：所有线程看到的 atomic 操作顺序是一致的。代价是性能上的一点点开销（x86 上写 atomic bool 大约是一条带 lock 前缀的指令，比普通写入贵几倍），但对"点一下按钮改一次"这种频率来说**完全可以忽略**。

> **取舍建议**：永远先用默认的 `seq_cst`。只有在性能剖析证明它是瓶颈时（比如音频回调里每秒 48000 次），才考虑 `memory_order_relaxed`。**写课件的时候用强保证，写的时候省心，读的人也省心。**

### 7.8.3 `std::mutex pipelineMutex_` —— 它保护的是什么

这把锁保护的资源是 **"`pipeline_` 这个对象本身不被两个线程同时操作**"。

谁会操作 `pipeline_`？

| 调用者 | 调用 | 线程 |
|---|---|---|
| `startStreaming` 的 lambda | `pipeline_.start(...)` | worker_ |
| `stopStreaming` 的 lambda | `pipeline_.stop()` | worker_ |
| `closeEvent` | `pipeline_.stop()` | **GUI 线程** |
| `pollStats` | `pipeline_.stats()` | **GUI 线程** |

四条路径至少涉及两种线程（GUI + worker）。如果不加锁：

- `pollStats()` 正在 `stats()` 里读 `impl_->output.targets()`（返回一个 vector），同时 `stop()` 正在 `output.close()` → **读的容器被另一边改着** → 崩溃或读到野数据；
- 用户点了"停止"（在 worker 线程跑 `stop()`），同时关窗口（GUI 线程跑 `stop()`）→ 两个 `stop()` 并发执行 → `output.close()` 被调用两次 → 我们第 10 课专门做了"幂等"，但其他部件不一定。

所以**凡是摸 `pipeline_` 的地方，一律先用 `std::lock_guard` 包起来**。看代码，四处都加了（`MainWindow.cpp` 第 294、338、381、423 行），没有遗漏。

> **一个容易被忽视的细节**：`std::mutex` 是**不可拷贝**的，所以 `MainWindow` 天然不可拷贝 —— 这正好（Qt 的窗口类本来就不该被拷贝）。同理 `std::thread`、`std::atomic` 也不可拷贝。

### 7.8.4 `std::thread worker_` —— 生命周期就是一条竖线

```cpp
std::thread worker_;
```

这是**唯一**一个我们亲手管理的线程句柄。它的生命周期是这样演化的：

```
 构造 MainWindow       ───→  worker_ 是空的（joinable()==false）
      │
      ↓ 用户点「开始」
  startStreaming()     ───→  worker_ = std::thread(lambda)   ← 线程开始跑
      │                       按住 joinable()==true
      ↓ 用户点「停止」（或下次点「开始」，或关窗口）
  stopStreaming()      ───→  if (worker_.joinable()) worker_.join();   ← 阻塞等它跑完
      │                       （等完之后 joinable() 变 false）
      ↓
  worker_ = std::thread(停止用lambda) ← 又被赋新的线程
```

**几个必须懂的规则：**

1. **`std::thread` 析构时如果还 joinable，程序直接 `std::terminate()` 崩掉。** 这是 C++ 标准的硬性规定。所以**必须**在析构前 `join()` 或 `detach()`。我们的 `~MainWindow` 里<｜hy_place▁holder▁no▁813｜> join ✅；
2. **`worker_ = std::thread(...)`（移动赋值）会先把旧的 thread 对象析构掉** —— 如果旧的还是 joinable 状态 → 同样 terminate！**这就是为什么 `startStreaming()` 第 283 行要先 `if (worker_.joinable()) worker_.join();`**。这一步不是可有可无，是必须的；
3. **`join()` 会阻塞调用者线程直到那条线程跑完。** 在 GUI 线程里调用 `join()` 就意味着 GUI 会卡住一小会儿。今天有三处 join：**第 283 行（开始前清理旧线程）、第 331 行（停止前等启动线程）、第 428 行（关窗口时）**。前两个通常很快（几毫秒到几十毫秒），第三个可能慢一些（第 13 章讨论）。

### 7.8.5 `QElapsedTimer elapsed_` —— 为什么不用 `stats.elapsedSeconds`

有意思的是，第 11 课写的 `PipelineStats` 里**本来就有一个 `elapsedSeconds` 字段**（`StreamPipeline.h` 第 16 行），但 `pollStats()` **没用它**，而是用了自己的秒表。

为什么？两个理由可以讨论（这一条留在第 17 章思考题里）：

| 方案 | 起点时刻 | 好处 | 坏处 |
|---|---|---|---|
| 用 `stats.elapsedSeconds` | 管线内部 `Clock::now()`（在第 11 课 `start()` 里第 `clockStart` 那一行） | 和管线内部逻辑完全同步 | ① 需要先 ask pipeline，还要判断 `running` 才非零；② 管线可能在 UI 知道之前就停了 |
| **用 UI 自己的 `QElapsedTimer`**（本项目做法） | UI 收到"启动成功"的那一刻（`elapsed_.start()`，第 315 行） | ① UI 完全掌握，跟 UI 显示的"状态"严格绑定；② 不依赖管线内部实现；③ 不需要加锁 | 比管线内部的起点晚几十毫秒（消息投递的延迟） |

**两者都对，这是设计取舍。** 本项目选"UI 端计时"，体现的是一条原则：**界面展示的数据，尽量由界面自己持有；自己拿不准的才去问下层。**

## 7.9 小结：第 7 章我们能从这个头文件读出什么

回头看这 86 行，它其实告诉了我们很多关于"这个类怎么运作"的信息：

| 从哪几行 | 读出来的信息 |
|---|---|
| 5~7 行继承 `QMainWindow` + `Q_OBJECT` | 它是一个 Qt 主窗口，有自己的元对象 |
| 第 30 行 `closeEvent` | 它需要介入"关闭"这个动作（异步清理） |
| 49~56 行（视频源成员） | 界面有一整组"视频源设置"，而且其中两行（Form/Row）需要被**整行操作** |
| 第 62 行 `QPlainTextEdit*` | 地址是一个多行文本（可以多路推流） |
| 73~74 行 `logView_` + `statsTimer_` | 有日志输出 + 有定时刷新 |
| 第 76 行 `pipeline_` 是**值成员** | 这个类"拥有"一条流水线，生命周期与窗口同寿 |
| 第 78 行 mutex / 83 行 thread | 有多线程行为（这是唯一的线索告诉我们别把它当单线程类看） |
| 80~81 行两个 atomic bool | 状态机是三态的（未开始 / 连接中 / 推流中） |
| 第 85 行 QElapsedTimer | 它需要向用户展示"已推多久" |

**一个设计良好的头文件，应该让人不读 .cpp 就猜出个大概。这个头文件做到了。**

---
# 第 8 章 逐行教学（四）：构造函数与析构函数 —— 顺序就是生命线

从第 8 章开始我们读 `MainWindow.cpp`（437 行）。先从最短、但最讲究顺序的两段开始。

## 8.1 先看头 22 行的 include：这是一份"用到了什么"的清单

```cpp
 1  #include "MainWindow.h"
 2
 3  #include "core/capture/AudioCapturer.h"
 4  #include "core/capture/CameraCapturer.h"
 5
 6  #include <QCloseEvent>
 7  #include <QComboBox>
 8  #include <QFormLayout>
 9  #include <QGroupBox>
10  #include <QHBoxLayout>
11  #include <QLabel>
12  #include <QPlainTextEdit>
13  #include <QPushButton>
14  #include <QSize>
15  #include <QSpinBox>
16  #include <QStringList>
17  #include <QTime>
18  #include <QTimer>
19  #include <QVBoxLayout>
20
21  #include <algorithm>
22
```

对照第 7 章的"前向声明"就知道这些是为什么存在的：**头文件只声明了指针，.cpp 里要真正调用它们的方法，所以必须在这里 include 完整定义。**

| 包含项 | 因为要用 |
|---|---|
| `"MainWindow.h"` | 自己的声明（**永远放在第一个**） |
| `AudioCapturer.h` / `CameraCapturer.h` | 第 186/210 行的 `listDevices()`（设备枚举） |
| `<QCloseEvent>` | 第 424 行 `closeEvent` 的 `event` 参数 |
| `<QComboBox>` 等 8 个控件头 | 第 52~74 行那些控件的构造与方法调用 |
| `<QSize>` | 第 66~68 行 `QSize(640, 360)`（作为 QVariant 放进 combo） |
| `<QStringList>` | 第 250 行 `const QStringList lines = ...split('\n')` |
| `<QTime>` | 第 419 行 `QTime::currentTime().toString("HH:mm:ss")` |
| `<QFormLayout>` / `<QHBoxLayout>` / `<QVBoxLayout>` | 布局三大件 |
| `<QGroupBox>` | 分组框 |
| `<algorithm>` | 第 230 行 `std::max` |

> **第 1 行是个约定俗成的规矩**：每个 .cpp 的第一个 include 是它自己的头文件。好处是**如果头文件自己缺少某个依赖（写得不合格），编译这个 .cpp 时会立刻暴露** —— 而不是等某一天别人 include 它时才炸。这叫"自包含的头文件"（self-contained header），是良好的接口 hygiene。

> **再看一个小瑕疵**（第 4.6 节提过）：这里**用了 `QFont`（第 123~125 行）却没有包含 `<QFont>`**，靠 `QLabel`/`QWidget` 的头文件间接带进来。今天我们补这一行作为第 15 章的实验内容。

## 8.2 构造函数：`MainWindow::MainWindow`（第 23~33 行）

```cpp
23  MainWindow::MainWindow(QWidget* parent) : QMainWindow(parent)
24  {
25      buildUi();
26
27      refreshVideoDevices();
28      refreshAudioDevices();
29      updateSourceFields();
30      updateUiState();
31
32      appendLog("就绪。填好推流地址后点「开始推流」。");
33  }
```

短短短 11 行，但**每一行都不能换位置**。我们逐行拆解它的"五步协议"：

### 第 23 行：`: QMainWindow(parent)`

初始化列表里**先把 parent 传给基类**。这一步之后，`this` 就被登记进父亲的 children 列表（如果 parent 不是 nullptr）。

注意 Qt 的所有窗口类都遵循这个签名：`ClassName(QWidget* parent = nullptr)`。

### 第 25 行：`buildUi();` —— 必须第一个调用

为什么这一步必须在最前面？因为**后面四步全都在访问第 49~74 行那些指针成员**：

```cpp
refreshVideoDevices();   // 要用 cameraCombo_、logView_
refreshAudioDevices();   // 要用 audioDeviceCombo_、audioSourceCombo_
updateSourceFields();    // 要用 videoSourceCombo_、videoForm_、cameraRow_
updateUiState();         // 要用几乎所有成员（10 个掩码 + 3 个标签）
```

如果 `buildUi()` 没先跑，这些指针全是 `nullptr` —— 一行 `*nullptr->setEnabled()` 就是崩溃。（这就是第 7.7 节讲的"`= nullptr` 初始化"的价值：即便漏了，调试器也能立刻看出"它是空的"。）

**这是一个通用原则**：**构造函数里的步骤顺序 = 依赖关系顺序。先建对象，再初始化内容，最后刷新显示。**

### 第 27~30 行：四个"刷新/同步"调用

```cpp
refreshVideoDevices();   // 枚举摄像头，填进下拉框
refreshAudioDevices();   // 根据当前音频来源，枚举音频设备，填进下拉框
updateSourceFields();    // 根据当前视频来源，决定"设备"这一行要不要显示
updateUiState();         // 根据当前状态（未开始），决定哪些控件可用、状态字写什么
```

这四步都属于"**让界面的显示和内部状态达成一致**"。请大家注意一个很有价值的设计模式：**Qt 程序里通常有一个"统一的刷新函数"（这里是 `updateUiState()`），所有状态变化最后都归到它一处修改控件。** 好处：

- 不会出现"A 处改了按钮可用性、B 处忘了改"的不一致；
- 要新增一个"推流时禁止编辑"的控件，只改一个地方；
- 状态机是集中表达的（谁 enabled、写什么字，都在一个函数里看得见）。

那种"在每个地方都写一行 `xxx->setEnabled(...)`"的散弹式写法，是 Qt 项目里最常见的技术债来源之一。

**第二步的隐藏依赖**：`refreshAudioDevices()` 会读 `audioSourceCombo_->currentIndex()`（第 203 行）来决定枚举"系统声音"还是"麦克风"。所以它必须在 `buildUi()` 之后（有这个 combo）并且在 combo 有默认选中项之后。`buildUi()` 里第 89~90 行刚加完两个 item，**默认选中第 0 项**（屏幕/系统声音），所以这里天然 OK。

### 第 32 行：`appendLog("就绪。填好推流地址后点「开始推流」。");`

日志框的第一句话。这一步**必须放在最后**，因为 `appendLog` 要用到 `logView_`（第 407~410 行），而它是在 `buildUi()` 里创建的（第 152~155 行）。

这一行还有个"用户体验"层面的小心思：**让用户一打开程序就知道下一步该干什么。** 界面设计上讲叫"空状态引导"（empty state guidance）。很多软件的日志框一打开是空的，用户不知道自己该做什么 —— 我们不这样。

> **顺带一个 Qt 小知识**：第 25~32 行这些操作**都是在窗口还没显示的时候做的**。Qt 完全允许你在 `show()` 之前操作控件 —— 控件的数据先准备好，等第一次 PaintEvent 到来时一次性画出来。**所以你不会看到"窗口先出现，然后日志框里闪着出现一行字"的过程。** 这也解释了为什么构造函数里可以做这么多事：它们都还没"上屏"，成本极低。

### 构造函数小结：五步图

```
┌─ 构造 MainWindow ────────────────────────────────────┐
│                                                      │
│  ① : QMainWindow(parent)     登记到父对象树            │
│  ② buildUi();                创建全部控件 + 布局 + 连接 │
│  ③ refreshXxx();             填充动态内容（设备列表）   │
│  ④ updateUiState();          让显示与状态一致           │
│  ⑤ appendLog("就绪...");     给用户第一句话             │
│                                                      │
│  顺序原则：依赖在前，被依赖的在后；空白在前，内容在后     │
│  ★ 注：以上全部在 GUI 线程执行，可以直接碰控件           │
└──────────────────────────────────────────────────────┘
```

## 8.3 析构函数：`MainWindow::~MainWindow`（第 35~39 行）

```cpp
35  MainWindow::~MainWindow()
36  {
37      if (worker_.joinable())
38          worker_.join();
39  }
```

这四行是**第 3.8 节"纪律三"的兑现**：

> **持有 `this` 的后台线程，析构前必须 join。**

为什么这条如此重要？我们画一下"如果不 join 会发生什么"：

```
   GUI 线程                                         worker_ 线程
   ─────────                                        ────────────
   ~MainWindow() 开始
   成员逐个析构（pipeline_、combo_、logView_...）
      ↓
   ~MainWindow() 结束，this 内存已释放 ‼
                                                    还在跑 pipeline_.stop()/start()
                                                      ↓
                                                    invokeMethod(this, ...)  ← this 是野指针！💥
```

反过来，加上 join 之后：

```
   GUI 线程                                         worker_ 线程
   ~MainWindow() 开始
   if (worker_.joinable()) worker_.join();
      │ ← 阻塞在这里，直到 worker 跑完
      │                                             跑完最后一行
      │                                             （invokeMethod 已经投出去了）
      ├───────────────────────────────────────────→ 线程结束
   成员逐个析构 ✅（此时已经没有线程引用 this 了）
```

**那 join 之后还留着"已经投递出去的事件"怎么办？** 第 3.8 节讲过：`~QObject` 会调用 `removePostedEvents(this)` 把排队的事件删掉（连带析构里面包着的 lambda），不会执行、也不会泄漏。

### 两个必须懂的判断后缀 `.joinable()`

```cpp
if (worker_.joinable())
    worker_.join();
```

为什么要问 `joinable()`？因为 **`std::thread::join()` 对一个不可 join 的线程调用会抛异常**（`std::system_error`）。`joinable()` 为 true 的条件是：这个 thread 对象**关联着一个还在跑或者还没被 join 的真实线程**。

什么时候 `joinable()` 为 false？

| 情况 | joinable |
|---|---|
| 默认构造的 `std::thread`（我们成员此时就是） | ❌ false |
| 已经被 `join()` 过了 | ❌ false |
| 已经被 `detach()` 过了 | ❌ false |
| 刚被 `thread(lambda)` 赋值、线程还在跑 | ✅ true |
| 线程已经跑完但还没 join | ✅ true（"joinable" 指的是"还能被 join"，不是"还在跑"） |

**本项目里一共有三处这个组合**（第 37、283、331 行），它们是同一套 idiom：**"如果还欠着一条线程，就先把它的账结了。"**

### 为什么这里不用 RAII 包一层？

有同学会问：这样手写 join 容易忘，C++20 有 `std::jthread`（析构时自动 join），我们为什么不用？

三个现实原因：

1. **我们是 C++17**（第 1 课定的 `CMAKE_CXX_STANDARD 17`），`std::jthread` 是 C++20 才有的；
2. `jthread` 析构时 join 的顺序依赖于成员声明顺序，容易被误改，不如显式写在析构函数体里清楚；
3. 我们要的**不只是"析构时 join"**，还有"**重新开始/停止之前先 join 掉上一次的线程**"（第 283、331 行）—— 这个需求 `jthread` 也帮不了。

所以：**显式 join 三处，统一写成"先 .joinable() 判断、再 .join()"这个固定套路，是这个场景最合适的写法。**

> **一个进阶思考（留在第 17 章）**：这个析构函数目前是" 崩溃安全"的，但它可能让用户等很久（如果用户点了"停止"但网络卡住，`join()` 会一直等到管线 stop 完成）。布局上更友好的做法是给停止加一个超时。我们今天的版本足够好，但值得想一想。

---

# 第 9 章 逐行教学（五）：`buildUi()` —— 造控件、摆布局、连信号三步走

这是本文件最长的一个函数（第 41~179 行，139 行）。我们把它切成三段来读，并且**每一段的代码都原样贴出来**。

先给出这个函数的整体流程图：

```
┌─────────────── buildUi() ────────────────────────┐
│                                                  │
│  第 43~44 行   ① 窗口外形：标题、尺寸             │
│  第 46~47 行   ② 搭根骨架：central + mainLayout  │
│  第 49~83 行   ③ 建「视频源」组（左侧）          │
│  第 85~99 行   ④ 建「音频源」组（右侧）          │
│  第 101~103 行 ⑤ 两组左右并排放进 sourceLayout   │
│  第 105~114 行 ⑥ 建「推流地址」组                │
│  第 116~131 行 ⑦ 建控制行（开始/停止/状态）      │
│  第 133~147 行 ⑧ 建统计行（四个数字）            │
│  第 149~155 行 ⑨ 建「日志」组                    │
│  第 157~161 行 ⑩ 把上面所有东西按顺序塞进        │
│                 mainLayout，末位 logGroup 加权   │
│  第 163 行     ⑪ 挂中绥（setCentralWidget）      │
│  第 165~174 行 ⑫ 六个 connect（信号接线）        │
│  第 176~178 行 ⑬ 造统计定时器（不 start）        │
│                                                  │
│  一句话：前十步在"建UI"，最后两步在"接电线"      │
└──────────────────────────────────────────────────┘
```

## 9.1 第 41~47 行：函数签名、窗口外形与根骨架

```cpp
41  void MainWindow::buildUi()
42  {
43      setWindowTitle("LightLiveStreamer");
44      resize(920, 700);
45
46      auto* central = new QWidget(this);
47      auto* mainLayout = new QVBoxLayout(central);
```

### 第 43~44 行：

- `setWindowTitle()`：标题栏文字；
- `resize(920, 700)`：**初始建议尺寸** 920×700。注意这只是初始值，用户可以拉大拉小 —— 布局系统会让控件跟着走（这就是第 4.1 节我们讲布局的理由）。

为什么是这两个数？因为一块 1080p 屏幕上，920 宽大约占一半，日志区能显示几十行；700 高能放下"参数区+地址区+控制+统计+日志"这五层。这就是经验的产物。

### 第 46~47 行：

```cpp
auto* central = new QWidget(this);              // ① 中央部件，parent=this
auto* mainLayout = new QVBoxLayout(central);    // ② 布局直接建在 central 上
```

`QMainWindow` 必须用 `setCentralWidget()` 指定一个"中央部件"，我们这里造了一个空白 `QWidget` 当容器。

`new QVBoxLayout(central)` 这个构造方式（**把父 widget 传给布局的构造函数**）等价于：

```cpp
auto* mainLayout = new QVBoxLayout;
central->setLayout(mainLayout);
```

前者更简洁，也更不容易忘掉第二步。**推荐写法。**

> **`auto*` 的取舍**：这里用了 `auto*` 而不是显式类型。对于" new Xxx(...) "这种等号右边已经写清了类型的场景，`auto*` 是安全的而且可读性更好。**但如果右边类型不明显**（比如调用某个返回指针的函数），请写显式类型。**我们的类成员（后面各节）全部写了显式类型 `QComboBox* xxx_ = new QComboBox(...)`**，因为头文件里已经声明了类型，写一遍有助于对照。

## 9.2 第 49~83 行：视频源分组（左）

```cpp
49      auto* videoGroup = new QGroupBox("视频源", central);
50      videoForm_ = new QFormLayout(videoGroup);
51
52      videoSourceCombo_ = new QComboBox(videoGroup);
53      videoSourceCombo_->addItem("屏幕");
54      videoSourceCombo_->addItem("摄像头");
55      videoForm_->addRow("来源", videoSourceCombo_);
56
57      cameraCombo_ = new QComboBox(videoGroup);
58      refreshCameraButton_ = new QPushButton("刷新", videoGroup);
59      refreshCameraButton_->setFixedWidth(64);
60      cameraRow_ = new QHBoxLayout;
61      cameraRow_->addWidget(cameraCombo_, 1);
62      cameraRow_->addWidget(refreshCameraButton_);
63      videoForm_->addRow("设备", cameraRow_);
64
65      resolutionCombo_ = new QComboBox(videoGroup);
66      resolutionCombo_->addItem("640 x 360", QSize(640, 360));
67      resolutionCombo_->addItem("1280 x 720", QSize(1280, 720));
68      resolutionCombo_->addItem("1920 x 1080", QSize(1920, 1080));
69      resolutionCombo_->setCurrentIndex(1);
70      videoForm_->addRow("分辨率", resolutionCombo_);
71
72      fpsSpin_ = new QSpinBox(videoGroup);
73      fpsSpin_->setRange(5, 60);
74      fpsSpin_->setValue(30);
75      fpsSpin_->setSuffix(" fps");
76      videoForm_->addRow("帧率", fpsSpin_);
77
78      bitrateSpin_ = new QSpinBox(videoGroup);
79      bitrateSpin_->setRange(300, 20000);
80      bitrateSpin_->setSingleStep(500);
81      bitrateSpin_->setValue(2500);
82      bitrateSpin_->setSuffix(" kbps");
83      videoForm_->addRow("码率", bitrateSpin_);
```

### 第 49~50 行：一个 GroupBox + 一个 FormLayout

`QFormLayout` 的 `addRow(标签文字, 控件/布局)` 会自动生成左边的 `QLabel`。省掉一半代码。

### 第 52~55 行：来源下拉框

```cpp
videoSourceCombo_->addItem("屏幕");      // index 0
videoSourceCombo_->addItem("摄像头");    // index 1
```

注意：**"屏幕"必须是 index 0，"摄像头"必须是 index 1**，因为后面两处代码是照着这个下标判断的：

```cpp
// updateSourceFields() 第 220 行
const bool isCamera = videoSourceCombo_->currentIndex() == 1;

// collectConfig() 第 228 行
config.sourceType = videoSourceCombo_->currentIndex() == 1 ? SourceType::Camera
                                                           : SourceType::Screen;
```

> **⚠ 这是一个值得警惕的耦合**："界面的显示顺序"和"业务判断逻辑"被下标焊死了。哪天有人在"屏幕"前面插一个"窗口捕获"，第 220 行和第 228 行两处都要改 —— 而且**改掉一个就会出 bug**（界面显示对了、但实际行为反了，而且编译不报错）。
>
> 更好的写法有两种（留作第 17 章思考题）：
> ```cpp
> // 方案 1：用 userData 带枚举值，跟下标解耦
> videoSourceCombo_->addItem("屏幕", static_cast<int>(SourceType::Screen));
> videoSourceCombo_->addItem("摄像头", static_cast<int>(SourceType::Camera));
> // 取值：SourceType( combo->currentData().toInt() )
>
> // 方案 2：用 enum 做顺序表，或者给下标起名字（constexpr int kIndexCamera = 1;）
> ```
> **记住这个教训：凡是"魔法数字"（magic number），都应该给它一个名字或者一个数据表。**

### 第 57~63 行：设备行 —— 一个布局塞进另一个布局

```cpp
cameraCombo_ = new QComboBox(videoGroup);
refreshCameraButton_ = new QPushButton("刷新", videoGroup);
refreshCameraButton_->setFixedWidth(64);
cameraRow_ = new QHBoxLayout;                     // ← 注意：没有 parent
cameraRow_->addWidget(cameraCombo_, 1);           // stretch=1：占剩余宽度
cameraRow_->addWidget(refreshCameraButton_);      // stretch=0：保持自身宽度
videoForm_->addRow("设备", cameraRow_);           // ★ 把整条布局当作"一个表单项"放进去
```

这是本课最值得学的一手：**`QFormLayout::addRow` 的第二个参数既可以是一个 Widget，也可以是一个 Layout。** 于是"左边一个下拉框、右边一个按钮"这种"一行两件"的效果就出来了。

```
        ┌─ 「设备」表单行 ─────────────────────────────┐
        │  设备   [ Integrated Camera      ▾ ] [刷新]  │
        │          └── cameraCombo_ stretch=1 ──┘ └64px┘│
        └──────────────────────────────────────────────┘
```

**为什么 `cameraRow_` 要存成成员？** 因为第 63 行 `addRow` 接收布局之后，我们第 223 行要用 `videoForm_->setRowVisible(cameraRow_, isCamera)` **把整行隐藏起来**（选"屏幕"时不需要选摄像头）。

### 第 65~70 行：分辨率下拉框 —— userData 展示法

```cpp
resolutionCombo_->addItem("640 x 360",   QSize(640, 360));
resolutionCombo_->addItem("1280 x 720",  QSize(1280, 720));
resolutionCombo_->addItem("1920 x 1080", QSize(1920, 1080));
resolutionCombo_->setCurrentIndex(1);        // 默认选中 720p
```

**这是第 4.6 节讲的"显示文本 + userData"的最佳示例。** 三份数据各得其所：

| 用途 | 数据 | 存在哪 |
|---|---|---|
| 给人看 | `"1280 x 720"` | combo 的 displayRole |
| 给程序用 | `QSize(1280, 720)` | combo 的 userRole（`QVariant`） |

取值的时候（第 232 行）：

```cpp
const QSize size = resolutionCombo_->currentData().toSize();
config.width = size.width();
config.height = size.height();
```

干净、类型安全、改显示文案不影响逻辑。**请大家在自己的界面代码里广泛采用这一招。**

**为什么默认是 720p（index 1）而不是 1080p？** 这是 OBS 等成熟软件的默认选择：720p30 是"画质可接受 + 编码压力适中 + 上行带宽 2.5Mbps 够用"的甜点。1080p 对 CPU 和带宽都要求更高，第一次用的人很容易卡。

**为什么 640×360、1280×720、1920×1080 都是偶数？** 因为 H.264 编码要求**宽高必须是偶数**（YUV420 的色度是 2×2 抽样，奇数尺寸会让最右/最下一列半途而废）。第 11 课的 `StreamPipeline::start` 里有这个保险：

```cpp
impl_->config.width = config.width & ~1;    // 强制对齐到偶数
impl_->config.height = config.height & ~1;
```

我们界面给的本来就是偶数，所以这条保险永远触发不了 —— 但**内核不信任界面传进来的输入**，这是正确的态度：**每一层都应该自己兜底，不能假定上游是乖的。**

### 第 72~83 行：帧率和码率两个 SpinBox

```cpp
fpsSpin_->setRange(5, 60);            // 5~60 fps
fpsSpin_->setValue(30);               // 默认 30
fpsSpin_->setSuffix(" fps");          // 显示成 "30 fps"

bitrateSpin_->setRange(300, 20000);   // 300 kbps ~ 20 Mbps
bitrateSpin_->setSingleStep(500);     // 每次点箭头 ±500
bitrateSpin_->setValue(2500);         // 默认 2500 kbps
bitrateSpin_->setSuffix(" kbps");     // 显示成 "2500 kbps"
```

这几个范围的现实意义：

| 参数 | 范围 | 为什么 |
|---|---|---|
| fps 5~60 | 低于 5 基本不能看（除非做监控）；高于 60 对普通推流没意义（观众侧也不支持） | |
| 码率 300~20000 | 300k 是"能看出画面的下限"；20M 是普通家庭上行的上限（再高观众也拉不动） | |
| 单步 500 | 码率这种量级，每次调 1 kbps 太细；500 是个实用的手感 | |

**为什么默认码率是 2500 而不是 `StreamConfig` 里写的 4500？** 看第 3 课 `Common.h`：`int videoKbps = 4500;`。界面这里给了 2500。这是"默认值不必要一致"的例子 —— 界面面对的是真实用户，更倾向于保守起始值；而 `StreamConfig` 的默认值是给 API 调用者兜底的。**不过这也埋了个小坑**：如果用户不去动那个 SpinBox，就永远不会有 4500 出现。没关系，反正是界面自己给的值优先。

## 9.3 第 85~99 行：音频源分组（右）

```cpp
85      auto* audioGroup = new QGroupBox("音频源", central);
86      auto* audioForm = new QFormLayout(audioGroup);
87
88      audioSourceCombo_ = new QComboBox(audioGroup);
89      audioSourceCombo_->addItem("系统声音");
90      audioSourceCombo_->addItem("麦克风");
91      audioForm->addRow("来源", audioSourceCombo_);
92
93      audioDeviceCombo_ = new QComboBox(audioGroup);
94      refreshAudioButton_ = new QPushButton("刷新", audioGroup);
95      refreshAudioButton_->setFixedWidth(64);
96      auto* audioRow = new QHBoxLayout;
97      audioRow->addWidget(audioDeviceCombo_, 1);
98      audioRow->addWidget(refreshAudioButton_);
99      audioForm->addRow("设备", audioRow);
```

跟视频源那组几乎一样的模板。**唯一的结构性差异**：

- 视频组的 `videoForm_` 和 `cameraRow_` 是**类成员**（因为要被 `updateSourceFields` 使用）；
- 音频组的 `audioForm` 和 `audioRow` 是**局部变量**（用完就丢，反正没人再访问它）。

这正是第 7.7 节讲的判断标准："**只有需要跨函数访问的 UI 对象才做成成员。**"

为什么音频不需要隐藏"设备"这一行？因为**无论选"系统声音"还是"麦克风"，你都需要选设备**（系统默认设备 vs 指定输出设备 / 默认麦克风 vs 指定麦克风）。而视频选"屏幕"时根本不需要摄像头列表。

## 9.4 第 101~114 行：左右两组的并排 + 推流地址区

```cpp
101      auto* sourceLayout = new QHBoxLayout;
102      sourceLayout->addWidget(videoGroup, 1);
103      sourceLayout->addWidget(audioGroup, 1);
```

两个 stretch 都是 1 → **左右各占一半宽度**，窗口拉宽时两边同步变宽。

```cpp
105      auto* targetGroup = new QGroupBox("推流地址", central);
106      auto* targetLayout = new QVBoxLayout(targetGroup);
107
108      targetsEdit_ = new QPlainTextEdit(targetGroup);
109      targetsEdit_->setMaximumHeight(88);
110      targetsEdit_->setPlaceholderText(
111          "每行一个地址，可以写成「名称=地址」方便区分：\n"
112          "bilibili=rtmp://live-push.bilivideo.com/live-bvc/?streamname=...\n"
113          "douyin=rtmp://push-rtmp-l6.douyincdn.com/stage/...");
114      targetLayout->addWidget(targetsEdit_);
```

三个知识点：

**① `setMaximumHeight(88)`**：限制最大高度。配合结点的 stretch 策略，这个输入框最多 88 像素高（大约能显示 4~5 行），不会把窗口撑得老高。

> **为什么要用 `QPlainTextEdit` 而不是 `QLineEdit`？** 因为`我们需要多行` —— 支持**同时推多个平台**（第 10 课做的多目标分发，界面这次终于用上了）。这是第 10 课成果的 UI 兑现。

**② `setPlaceholderText()`**：灰色提示文字，**用户一开始输入就自动消失**（这是 Qt 内建的行为，不用写任何代码）。这里还顺手做了一件很贴心的事：**直接把两个真实平台的地址模板写在提示里**，用户照着填就行。

注意两个 `\n`：**这是 C++ 字符串字面量的自动拼接** —— 相邻的字符串字面量会被编译器自动连成一个。所以第 111~113 行实际是一个长字符串。这种写法比一行写到底可读性好。

**③ 这里的分 Meanwhile 」「名称=地址」** 这个小小的用例设计值得称赞：

- 纯地址（`rtmp://...`）→ 平台名自动生成"路 1""路 2"（第 260~261 行）；
- 带名字（`bilibili=rtmp://...`）→ 平台名就是 `bilibili`；
- 以 `#` 开头 → 注释，跳过（第 248 行）。

**"允许注释"是一个很受用户欢迎的特性**：推多个平台时，临时想把某一路停掉，就在前面加个 `#`，不用删掉整行 —— 下次想恢复删掉 `#` 就行。这是"用户角度思考"的体现。

## 9.5 第 116~147 行：控制行与统计行

```cpp
116      auto* controlLayout = new QHBoxLayout;
117
118      startButton_ = new QPushButton("开始推流", central);
119      stopButton_ = new QPushButton("停止推流", central);
120      stopButton_->setEnabled(false);
121      stateLabel_ = new QLabel("未开始", central);
122
123      QFont bold = stateLabel_->font();
124      bold.setBold(true);
125      stateLabel_->setFont(bold);
126
127      controlLayout->addWidget(startButton_);
128      controlLayout->addWidget(stopButton_);
129      controlLayout->addSpacing(16);
130      controlLayout->addWidget(stateLabel_);
131      controlLayout->addStretch();
```

**注意第 118~119 行的 parent 是 `central` 而不是 `controlLayout`** —— 因为布局不是 QObject 树意义上的父亲（第 1.4 节讲过）。这两个按钮一会儿要进 `controlLayout`，而 `controlLayout` 安装在 `central` 上，所以它们的"真正的父亲"是 `central`。写对了。

第 123~125 行的字体三部曲（取出 → 修改 → 放回）在第 4.6 节讲过，这里就是它的落地写法。

**第 131 行的 `addStretch()`**：把这一行剩余的空间全吃掉 → **三个控件靠左，右边留白**。如果不加，`startButton_`/`stopButton_` 会被拉宽到填满整行（因为 QPushButton 的默认水平 sizePolicy 是 Preferred，但布局只要有多余空间就会分给 stretch>0 的项；按钮 stretch 为 0 时通常保持 sizeHint）。

```cpp
133      auto* statsLayout = new QHBoxLayout;
134
135      durationLabel_ = new QLabel("时长 00:00:00", central);
136      videoLabel_ = new QLabel("视频 -", central);
137      audioLabel_ = new QLabel("音频 -", central);
138      targetLabel_ = new QLabel("连接 -", central);
139
140      statsLayout->addWidget(durationLabel_);
141      statsLayout->addSpacing(16);
142      statsLayout->addWidget(videoLabel_);
143      statsLayout->addSpacing(16);
144      statsLayout->addWidget(audioLabel_);
145      statsLayout->addSpacing(16);
146      statsLayout->addWidget(targetLabel_);
147      statsLayout->addStretch();
```

**注意这里的初始值不是空的，而是带了占位符**：`"视频 -"`、`"时长 00:00:00"`。为什么？

这叫**预留宽度**（width reservation）。QLabel 的宽度默认跟着内容走，内容一变，后面的控件就会左右跳。给一个"最长形态"的初始值，布局一开始就算好了宽度，后面数字变化时布局不会抖。

> **这是一个非常实用的界面技巧**：凡是会动态变化又需要实时更新的文本 UI，**先给一个和目标格式最长的占位**，避免抖动。反之，如果这些 Label 初始是空的，`statsLayout` 一开始就给它算了个很窄的宽度，后来数字一多又要重新 RequestLayout —— 视觉上就是"每隔半秒抖一下"。

同样的技巧也用在 `pollStats` 里的格式化上（补零对齐，第 388~391 行）。

## 9.6 第 149~163 行：日志区与"最后塞进 mainLayout"

```cpp
149      auto* logGroup = new QGroupBox("日志", central);
150      auto* logLayout = new QVBoxLayout(logGroup);
151
152      logView_ = new QPlainTextEdit(logGroup);
153      logView_->setReadOnly(true);
154      logView_->setMaximumBlockCount(2000);
155      logLayout->addWidget(logView_);
```

**第 153~154 行这两个调用是日志框的灵魂**：

```cpp
logView_->setReadOnly(true);             // 用户不能编辑
logView_->setMaximumBlockCount(2000);    // ★ 最多 2000 "块"，超出从头丢
```

`setMaximumBlockCount(2000)` 是一个**内存护栏**。什么叫"块"（block）？在 `QPlainTextEdit` 里，一次 `appendPlainText()` 追加的内容大致构成一个块。所以它的含义是：**日志最多保留最近 2000 条，超过之后最老的会被自动删除。**

为什么必须有这一句？请同学们算一笔账：

- 一条日志大约 80~150 字节；
- 推流过程中，"每 500ms 一条状态日志" + "管线内部的各种事件日志"，保守估计每秒 3~5 条；
- 推 1 小时 = 3600 秒 × 4 条 = **14400 条**，还在 2000 以内（会被截断）；
- **如果不设上限**：推一晚上（8 小时）= 115200 条 × 100 字节 ≈ **11 MB 的纯文本**，而且 `QPlainTextEdit` 内部还要维护行号、换行位置等结构，实际内存可能是文本的 2~3 倍，再加上每次追加都可能触发重排 → 界面越用越卡。**这是非常典型的"内存/性能慢慢泄漏"事故。**

> 请大家记住这一招：**任何"日志输出到文本框"的界面，都要设 `setMaximumBlockCount`。** 这是 Qt GUI 开发的行业常识。

```cpp
157      mainLayout->addLayout(sourceLayout);
158      mainLayout->addWidget(targetGroup);
159      mainLayout->addLayout(controlLayout);
160      mainLayout->addLayout(statsLayout);
161      mainLayout->addWidget(logGroup, 1);
162
163      setCentralWidget(central);
```

**第 157~161 行是"组装"**：把前面造好的五大块，按从上到下的顺序塞进根布局 `mainLayout`（它是 QVBoxLayout，所以是竖排）。

**第 161 行的那个 `1` 是关键**：

```cpp
mainLayout->addWidget(logGroup, 1);      // stretch = 1
```

回忆第 4.3 节：stretch 决定"多余空间怎么分"。这里只有 `logGroup` 有 stretch=1，其他四块都是 0。结果是：**窗口拉高时，多出来的垂直空间全部给日志框**；其余四块保持自己的自然高度。

**这个设计的意图很清楚**：参数区、控制区的高度是固定的，只有"日志"是越多越好 —— 用户拉高窗口，多半就是想多看几行日志。**把空间分配给用户最想要的东西。**

最后第 163 行 `setCentralWidget(central)` 把中央部件挂到 `QMainWindow` 的骨架上。**注意这一步必须在 `central` 的内容都创建完之后**（虽然 Qt 允许事后改，但我们不需要）。

> **一个小知识点**：`setCentralWidget` 会把 `central` 的 parent 设为 `this`（因为 QMainWindow 必须拥有它）。如果之前 central 的 parent 已经是 this（第 46 行我们就是这么写的），那这一步就只是"登记为中央部件"。写对了就没事。

## 9.7 第 165~178 行：接线 —— 六个 connect 与一个定时器

```cpp
165      connect(videoSourceCombo_, &QComboBox::currentIndexChanged, this,
166              [this](int) { updateSourceFields(); });
167      connect(audioSourceCombo_, &QComboBox::currentIndexChanged, this,
168              [this](int) { refreshAudioDevices(); });
169      connect(refreshCameraButton_, &QPushButton::clicked, this,
170              &MainWindow::refreshVideoDevices);
171      connect(refreshAudioButton_, &QPushButton::clicked, this,
172              &MainWindow::refreshAudioDevices);
173      connect(startButton_, &QPushButton::clicked, this, &MainWindow::startStreaming);
174      connect(stopButton_, &QPushButton::clicked, this, &MainWindow::stopStreaming);
175
176      statsTimer_ = new QTimer(this);
177      statsTimer_->setInterval(500);
178      connect(statsTimer_, &QTimer::timeout, this, &MainWindow::pollStats);
```

我们把这六条线整理成表：

| # | 发送者 | 信号 | 槽 | 触发时机 |
|---|---|---|---|---|
| 1 | `videoSourceCombo_` | `currentIndexChanged(int)` | lambda → `updateSourceFields()` | 视频来源切换（屏幕↔摄像头） |
| 2 | `audioSourceCombo_` | `currentIndexChanged(int)` | lambda → `refreshAudioDevices()` | 音频来源切换（系统声音↔麦克风） |
| 3 | `refreshCameraButton_` | `clicked(bool)` | `refreshVideoDevices()` | 点"刷新" |
| 4 | `refreshAudioButton_` | `clicked(bool)` | `refreshAudioDevices()` | 点"刷新" |
| 5 | `startButton_` | `clicked(bool)` | `startStreaming()` | 点"开始推流" |
| 6 | `stopButton_` | `clicked(bool)` | `stopStreaming()` | 点"停止推流" |
| 7 | `statsTimer_` | `timeout()` | `pollStats()` | 每 500ms（启动后才 start） |

### 第 165~168 行：两个 lambda 槽

```cpp
connect(videoSourceCombo_, &QComboBox::currentIndexChanged, this,
        [this](int) { updateSourceFields(); });
```

这里用 lambda 而不是成员函数，是因为槽的内容只有一句转调用。**注意形参写成了 `(int)` 而没给名字** —— C++ 允许省略未使用的参数名（避免编译器警告 `unused parameter`）。

另一个要点（第 4.6 节讲过）：Qt 6 里 `currentIndexChanged` **只有一个 `int` 重载**，所以能直接写 `&QComboBox::currentIndexChanged`。Qt 5 里必须写成 `QOverload<int>::of(&QComboBox::currentIndexChanged)`。

### 第 169~172 行：两个"信号参数比槽多"的例子

```cpp
connect(refreshCameraButton_, &QPushButton::clicked, this,
        &MainWindow::refreshVideoDevices);
```

信号 `clicked(bool)` 有 1 个参数，槽 `refreshVideoDevices()` 有 0 个参数 —— **合法**，Qt 会在调用时把多出来的参数丢掉。这是 Qt 的一个很方便的规则。

### 第 173~174 行：两个"槽其实是普通私有成员函数"

`startStreaming()` / `stopStreaming()` 在头文件里是**普通的 private 成员函数**（第 39、40 行），没有任何 `slots:` 标记。**它们仍然是合法的槽** —— 因为 Qt 5 之后基于函数指针的 connect 不需要槽有任何特殊身份，它只是一个"能被 `QMetaObject` 记录并延时调用的函数指针"。

> 这一点经常被误解：**Qt 的槽不一定非得写在 `slots:` 下面。** `slots:` 那个宏在现代 Qt 里已经几乎不用了（展开还是空）。唯一还需要它的场合是配合字符串版 `connect`（老写法）。

### 第 176~178 行：定时器

```cpp
statsTimer_ = new QTimer(this);        // parent = this ★
statsTimer_->setInterval(500);         // 500 毫秒
connect(statsTimer_, &QTimer::timeout, this, &MainWindow::pollStats);
```

三个要点：

1. **`parent = this` 是必需的**（第 4.6 节讲过）：定时器跟着窗口销毁。**如果不给 parent，窗口关了它还在 tick，一下 tick 到已经析构的 MainWindow 身上 → 崩。** 而且这是"关了窗还崩"的一类典型 bug，非常难查。
2. **这里只 `setInterval`，没有 `start()`**。定时器创建出来是停止状态，要等第 316 行（启动成功）才 `statsTimer_->start()`。所以"未推流"时不会有统计刷新 —— 省 CPU，也避免对着一条没在跑的管线取数据。
3. **`timeout` 在哪个线程发出？在 `statsTimer_` 所属的线程，也就是 GUI 线程。** 所以 `pollStats()` 里可以放心地 `setText()`，**不需要任何投递**。这一点在第 3.9 节的总表里已经确认过。

## 9.8 `buildUi()` 的设计评价

我们花了很多行读它，现在回头评价一下这个函数：

**做得好的地方：**

| 点 | 评价 |
|---|---|
| **拆出一个独立的 `buildUi()`** | 构造函数只有 11 行，读一下就知道干了什么；界面的 chaos 被关在一个函数里 | 
| **不言自明的分组** | 每段 @"是以一个 GroupBox 开始的"，顺序跟着视觉顺序走 | 
| **成员变量最小化** | 只有两个布局需要跨函数访问，其他七个全用局部变量，头文件干净 |
| **接线集中在末尾** | 想看"点这个按钮会发生什么"，翻到最后 15 行就有答案，不用通读 139 行 |
| **默认值是产品思维** | 720p30/2500kbps、地址模板、占位宽度、空状态引导 —— 都是站在用户角度 |

**可以改进的地方**（第 17 章会展开）：

| 点 | 问题 |
|---|---|
| 139 行有点长 | 可以拆成 `createVideoGroup()` / `createAudioGroup()` / `createTargetGroup()` / `createControlBar()` / `createLogArea()` 五个私有函数 |
| 下标魔法数字 | `currentIndex()==1` 出现两处（异常 See 第 9.2 节警告） |
| 缺 `<QFont>` 的显式 include | 隐式依赖 |

> **给同学们的作业建议**：看完这一章，试着把 `buildUi()` 拆成五个小函数，各返回一个指针/布局。做完之后对比现在这份代码，体会一下"函数多长算长"这件事。**标准很简单：一个函数如果要用注释分段，往往就该拆了。**

---
# 第 10 章 逐行教学（六）：设备枚举三兄弟

这一章的三个函数负责一件具体的事：**把"计算机上现在有哪些摄像头/哪些音频设备"列到下拉框里**。它们看似简单，但藏着几个很实用的 Qt 套路。

## 10.1 `refreshVideoDevices()`（第 181~200 行）

```cpp
181  void MainWindow::refreshVideoDevices()
182  {
183      const QString previous = cameraCombo_->currentText();
184
185      cameraCombo_->clear();
186
187      const std::vector<std::string> devices = CameraCapturer::listDevices();
188      for (const std::string& name : devices)
189          cameraCombo_->addItem(QString::fromStdString(name));
190
191      if (devices.empty()) {
192          cameraCombo_->addItem("（没有找到摄像头）");
193          appendLog("没有枚举到任何摄像头");
194      } else {
195          const int index = cameraCombo_->findText(previous);
196          if (index >= 0)
197              cameraCombo_->setCurrentIndex(index);
198          appendLog(QString("枚举到 %1 个摄像头").arg(devices.size()));
199      }
200  }
```

### 第 183 行：先记住"用户刚才选的什么"

```cpp
const QString previous = cameraCombo_->currentText();
```

这一行是整个函数里**最有心机的一句**。它的目的是**保持用户的选择不被刷新打断**。

想象没有这一行会怎样：用户选了"Logitech C920"，然后手滑点了一下"刷新" → 列表重建 → 默认选中第 0 项 → 用户的选择没了，可能还会在不知情的情况下用错摄像头开始推流。**这是很糟糕的体验。**

有了这一行，刷新完之后第 195~197 行会尝试"找回刚才那一项"。

### 第 185 行：`clear()` —— 一个容易踩的信号陷阱

```cpp
cameraCombo_->clear();
```

`clear()` 会：

1. 清空所有条目；
2. 把当前下标改成 -1；
3. **发射 `currentIndexChanged(-1)` 信号**（因为下标确实变了）。

**这就引出了一个关键问题**：我们第 165 行把 `videoSourceCombo_` 的 `currentIndexChanged` 连到了 `updateSourceFields()`，但这里 clear 的是 `cameraCombo_` —— 幸好 `cameraCombo_` **没有连任何信号**，所以没事。

> **⚠ 但请大家记住这个陷阱**：**如果某个 combo 的 `currentIndexChanged` 连了槽，那么你在代码里调用 `clear()` / `setCurrentIndex()` / `addItem()`（首次 item 会让下标从 -1 变 0）时，都会触发那个槽。** 有时会引发"槽还没准备好就被调用"的问题。
>
> 应对手段（Qt 程序员的工具箱里必备）：
> ```cpp
> // 手段一：信号阻塞（最常用，Qt 内建支持）
> QSignalBlocker blocker(combo);      // RAII：构造时阻塞，析构时自动恢复
> combo->clear();
> combo->addItems(names);
> combo->setCurrentIndex(saved);
> // 出了作用域 blocker 析构，信号恢复
>
> // 手段二：用一个 bool 成员标记"正在初始化"，槽里先判断它
> ```
> **`QSignalBlocker` 是 RAII 的，异常安全，强烈推荐。** 有兴趣的同学可以试着给 `refreshAudioDevices()` 加上它，看看日志有没有变化（第 15 章实验）。

### 第 187~189 行：跨"Qt 世界"和"标准库世界"

```cpp
const std::vector<std::string> devices = CameraCapturer::listDevices();
for (const std::string& name : devices)
    cameraCombo_->addItem(QString::fromStdString(name));
```

这是一个**边界转换**场景：内核接口返回的是 `std::vector<std::string>`（第 3 课定的规矩：内核不碰 Qt），而 Qt 的控件要吃 `QString`。所以这里必须做转换 —— **`QString::fromStdString()`**。

> **第 1.5 节的知识在这里落地**：记住这个转换是有 UTF-8↔UTF-16 成本的。设备名很短，一两百个，完全无所谓。**但这个"S "" 的位置要选对**：我们把它放在"边界"上（UI 层），而不是让内核返回 QString。**这就是分层的好处 —— 转换只在这一处发生，将来想换成另一种界面技术，改这一个函数就行。**

### 第 191~199 行：空列表的体贴处理

```cpp
if (devices.empty()) {
    cameraCombo_->addItem("（没有找到摄像头）");
    appendLog("没有枚举到任何摄像头");
} else {
    const int index = cameraCombo_->findText(previous);
    if (index >= 0)
        cameraCombo_->setCurrentIndex(index);
    appendLog(QString("枚举到 %1 个摄像头").arg(devices.size()));
}
```

两个分支都值得学：

**① 空列表时不留一个空下拉框，而是放一条"人话"占位。**

这是 UI 设计里常见的"空状态处理"。对比一下两种处理：

- ❌ 空下拉框：用户懵了 —— "是还没加载完？是我电脑有问题？"（而且如果用户点它，什么也没有）
- ✅ 显示"（没有找到摄像头）"：用户立刻明白是没设备，并且顺手还可以去看日志。

**② `findText()` 的三段式用法**

```cpp
const int index = cameraCombo_->findText(previous);   // 找不到返回 -1
if (index >= 0)
    cameraCombo_->setCurrentIndex(index);
```

`findText` **找不到时返回 -1**，所以必须先判断再用。**忘记判断 `-1` 是一个经典错误**：`setCurrentIndex(-1)` 虽然不崩（会变成"未选中"），但会把用户的下拉框变成一个空框，看起来像 bug。

还有一个细节：**这里判断的是 `>= 0` 而不是 `!= -1`**，语义上等价但更符合"下标"这个概念的习惯写法。

**③ 注意 `previous` 的处理有个"历史遗留情况"**

如果用户之前选中的其实就是那个占位符"（没有找到摄像头）"，刷新之后 `previous` 就是这句中文 —— 而新列表如果这次有设备了，`findText("（没有找到摄像头）")` 会返回 -1，于是保持默认选中第 0 项。**这是正确行为**（占位符本来就不该被"选中回来"）。

## 10.2 `refreshAudioDevices()`（第 202~216 行）

```cpp
202  void MainWindow::refreshAudioDevices()
203  {
204      const AudioSourceType type = audioSourceCombo_->currentIndex() == 0
205                                       ? AudioSourceType::SystemAudio
206                                       : AudioSourceType::Microphone;
207
208      audioDeviceCombo_->clear();
209      audioDeviceCombo_->addItem("（系统默认）", -1);
210
211      const std::vector<std::string> devices = AudioCapturer::listDevices(type);
212      for (size_t i = 0; i < devices.size(); ++i) {
213          audioDeviceCombo_->addItem(QString::fromStdString(devices[i]),
214                                     static_cast<int>(i));
215      }
216  }
```

### 第 204~206 行：又一次「下标 → 枚举」的翻译

```cpp
const AudioSourceType type = audioSourceCombo_->currentIndex() == 0
                                 ? AudioSourceType::SystemAudio
                                 : AudioSourceType::Microphone;
```

这里没有用常见的换行缩进，而是**让三元表达式的 `?` 对齐到上一行的 `=`**（见第 205 行）。这是代码格式化的一个小选择 —— 本项目用的是 clang-format 的某种风格，不必强求，但要保持一致。

**又一次抱怨一次**：下标 `0` / `1` 和含义的耦合再次出现（第 9.2 节已经批评过一次）。这里至少有改善的空间：

```cpp
// 推荐改法：把值写进 userData，不再依赖顺序
audioSourceCombo_->addItem("系统声音", static_cast<int>(AudioSourceType::SystemAudio));
audioSourceCombo_->addItem("麦克风",   static_cast<int>(AudioSourceType::Microphone));
// 之后：
const auto type = static_cast<AudioSourceType>(audioSourceCombo_->currentData().toInt());
```

### 第 208~209 行：为什么音频要多一项"（系统默认）"

```cpp
audioDeviceCombo_->clear();
audioDeviceCombo_->addItem("（系统默认）", -1);       // ★ userData = -1
```

这项永远在第 0 位，它的 userData 是 **`-1`**。

对照第 3 课 `Common.h`：

```cpp
struct StreamConfig {
    int audioDeviceIndex = -1;      // ← 默认就是 -1！
};
```

以及第 7 课 `AudioCapturer::open(type, deviceIndex, log)` 的语义：`deviceIndex = -1` 表示"用系统默认设备"。

**所以"（系统默认）"这一项是一个"哨兵值"（sentinel value）**，它让用户可以直接说"别管哪个设备，用系统设的那个"，这在大多数场景就是用户想要的。

> **设计点评**：用 `-1` 表示"默认"是很常见的 C 风格做法。它的优点是简单直观；缺点是**占用了一个本来可能合法的索引空间**（好在设备索引永远 ≥ 0，所以 -1 安全）。更"现代"的做法是用 `std::optional<int>` —— 有值就是要指定设备，没值就是默认。**我们项目里两者并存：`StreamConfig::audioDeviceIndex` 用 -1，`BoundedQueue::tryPop` 用 optional。** 这就是现实工程：**新代码用新习惯，旧代码能不动就不动，重要的是把语义写清楚。**

### 第 212~215 行：`size_t i` 循环而不是 range-for

```cpp
for (size_t i = 0; i < devices.size(); ++i) {
    audioDeviceCombo_->addItem(QString::fromStdString(devices[i]),
                               static_cast<int>(i));
}
```

对比 10.1 节的 range-for，这里特意用了带下标的普通 for —— **因为我们需要 `i` 本身作为 userData**（设备索引）。这是"什么时候用 range-for、什么时候用下标循环"的活教材：**需要下标信息时用下标循环，否则用 range-for。**

注意 `static_cast<int>(i)`：把 `size_t`（无符号）转成 `int`。这步很有必要，因为：

1. `QComboBox::addItem` 的 userData 是 `QVariant`，它的 `int` 重载最符合；
2. 如果直接传 `size_t`，`QVariant` 会匹配到 `uint`/`qulonglong` 重载，将来 `toInt()` 的时候要小心溢出和符号。

**显式转换 + 明确的类型，是好习惯。** MSVC 在 /W4 下会对 `size_t → int` 隐式转换发警告，这里用 `static_cast` 明确"我知道这个转换是安全的（设备数量不可能超过 INT_MAX）"。

### 为什么这个函数写得比 `refreshVideoDevices()` 简单？

对比一下会发现它**没有**做三件事：

| `refreshVideoDevices` 做了 | `refreshAudioDevices` 没做 |
|---|---|
| 记住 previous 再找回来 | ❌ 每次都回到"（系统默认）" |
| 空列表时给占位提示 | ❌ 只有"（系统默认）"一项 |
| 往日志写枚举结果 | ❌ 静默 |

这算不算不一致？我认为是一个**可接受的小缺陷**（第 17 章会列为局限之一）：

- **合理的一面**：音频设备的稳定性比摄像头高得多（摄像头可能被拔出/被别的软件占用），而且"（系统默认）"本身就是一个安全的选择，用户重选一次成本很低；
- **不合理的一面**：如果用户指定了"麦克风阵列"，然后手滑点了刷新，选择就丢了。**用户体验的一致性被打断。**

**作为练习**：请把 10.1 节那套"记住 previous + findText + 占位 + 日志"搬到 `refreshAudioDevices` 里（第 15 章实验三）。

## 10.3 `updateSourceFields()`（第 218~224 行）

```cpp
218  void MainWindow::updateSourceFields()
219  {
220      const bool isCamera = videoSourceCombo_->currentIndex() == 1;
221
222      if (videoForm_ && cameraRow_)
223          videoForm_->setRowVisible(cameraRow_, isCamera);
224  }
```

### 第 222 行的防御性判空

```cpp
if (videoForm_ && cameraRow_)
```

为什么要判空？因为这个类里的成员都初始化成了 `nullptr`（第 49~50 行），而 `updateSourceFields()` 可能被非常早地调用。

实际调用序列是：构造函数第 25 行先 `buildUi()`（创建这两个对象），第 29 行才 `updateSourceFields()` —— **所以判空永远不会被触发**。

那为什么还要写？

- **防御式编程**：万一哪天有人调整了构造顺序（或者在 `buildUi` 里加了一条信号连接导致更早触发），这一行能保命；
- 成本为零（两个指针比较）；
- 它明示了一个事实："这两个成员可能还没建好。"

> 但也请同学们知道：**过度防御会把代码变成噪音。** 如果某个条件在调用路径上"逻辑上不可能"，更好的做法是加 `Q_ASSERT(videoForm_)`（Debug 版本中断）而不是静默跳过 —— 静默跳过的 bug 更难发现。这里的写法属于"温和派"，可以接受。

### 第 223 行：`QFormLayout::setRowVisible(layout, bool)`

```cpp
videoForm_->setRowVisible(cameraRow_, isCamera);
```

**`setRowVisible`** 是 Qt 5.8 引入的一个便利 API，它能把表单里的**一整行**（标签 + 内容）隐藏/显示。它有两个重载：

```cpp
void setRowVisible(QWidget* widget, bool on);    // 按某个控件隐藏它所在的行
void setRowVisible(QLayout* layout, bool on);    // ★ 按某个子布局隐藏它所在的行
```

我们用第二个（因为"设备"行的内容是一个 `QHBoxLayout`，而不是单个控件）。

**它内部做了什么？** 大致等价于：

```cpp
// 简化理解版
if (on) {
    restoreWidgetsInRow();        // 重新 add 标签和控件，并 show()
} else {
    takeRow(...);                 // 从布局里摘掉
    hideWidgetsInRow();           // 把标签和里面的控件 hide()
}
```

**如果没有这个 API 要怎么做？** 麻烦得多：你得自己保存那一行的 label 指针和内容控件指针，手动 `label->hide(); widget->hide();` 而且**布局仍然会给它们留空间**（这是 Qt 的一个反直觉点：hide() 不等于从布局里消失）。所以 `setRowVisible` 是个很值得记住的 API。

### 一个隐藏的小疏漏

请注意：**"设备"行被隐藏时，`cameraCombo_` 里的当前值并没有变**，仍然保留着用户上次选的那个摄像头。

然后看 `collectConfig()` 第 232~233 行：

```cpp
if (config.sourceType == SourceType::Camera)
    config.cameraIndex = std::max(0, cameraCombo_->currentIndex());
```

幸好有 `if (...) == SourceType::Camera` 这层保护：**选了"屏幕"时不会去读 cameraIndex**。所以功能是对的。

但这里有一个**不够干净的地方**：

- 如果用户曾经选了 index 2 的摄像头，然后切回"屏幕" —— `cameraCombo_->currentIndex()` 仍然返回 2；
- 万一列表里有一项那个占位"（没有找到摄像头）"，`std::max(0, ...)` 会把它变成 0；

`std::max(0, currentIndex())` 这句是干什么的？把 `-1`（未选中）钳到 `0`。**这是一个"随便给它一个默认值"的处理 —— 不算错，但也不算好**：因为用户看不出来"没选设备被自动改成了第一个设备"。

> **改进方向**：更严格的话，应该在"选了摄像头但没有有效设备"时**在 `startStreaming` 之前弹个提示**，而不是默默打开第 0 个摄像头然后让用户发现推的是错的画面。**界面的职责就是把"可能出错的选择"明确告诉用户。**

---

# 第 11 章 逐行教学（七）：`collectConfig()` —— 界面到契约的翻译官

这个函数（第 226~275 行，50 行）承担的是本课**最重要的类型转换**：`一堆 QWidget 的当前值` → `StreamConfig`。

```
┌────────────────────────────────────────────────────────────┐
│                                                            │
│   界面的各种控件值（QString / int / index / userData）      │
│              │                                             │
│              │  collectConfig()                            │
│              ↓                                             │
│   一份 StreamConfig（纯 C++ 数据，不认识 Qt）               │
│              │                                             │
│              │  pipeline_.start(config, ...)               │
│              ↓                                             │
│   第 11 课那套流水线（它完全不知道界面长什么样）            │
│                                                            │
└────────────────────────────────────────────────────────────┘
```

**这个函数的存在本身就是"分层"的证据**：界面层负责收集，契约层负责传递，管线层负责执行。三层之间靠的是**两个纯数据**（`StreamConfig` 和 `LogCallback`），而不是互相引用。

## 11.1 全文

```cpp
226  StreamConfig MainWindow::collectConfig() const
227  {
228      StreamConfig config;
229
230      config.sourceType = videoSourceCombo_->currentIndex() == 1 ? SourceType::Camera
231                                                                : SourceType::Screen;
232      if (config.sourceType == SourceType::Camera)
233          config.cameraIndex = std::max(0, cameraCombo_->currentIndex());
234
235      const QSize size = resolutionCombo_->currentData().toSize();
236      config.width = size.width();
237      config.height = size.height();
238      config.fps = fpsSpin_->value();
239      config.videoKbps = bitrateSpin_->value();
240
241      config.audioType = audioSourceCombo_->currentIndex() == 0
242                             ? AudioSourceType::SystemAudio
243                             : AudioSourceType::Microphone;
244      config.audioDeviceIndex = audioDeviceCombo_->currentData().toInt();
245
246      config.sampleRate = 48000;
247      config.channels = 2;
248      config.audioKbps = 128;
249
250      const QStringList lines = targetsEdit_->toPlainText().split('\n');
251      for (const QString& raw : lines) {
252          const QString line = raw.trimmed();
253          if (line.isEmpty() || line.startsWith('#'))
254              continue;
255
256          StreamTarget target;
257          const int separator = line.indexOf('=');
258          if (separator > 0) {
259              target.platform = line.left(separator).trimmed().toStdString();
260              target.url = line.mid(separator + 1).trimmed().toStdString();
261          } else {
262              target.url = line.toStdString();
263          }
264
265          if (target.platform.empty())
266              target.platform = "路" + std::to_string(config.targets.size() + 1);
267
268          if (target.url.empty())
269              continue;
270
271          config.targets.push_back(target);
272      }
273
274      return config;
275  }
```

## 11.2 第 228 行：从"默认值"开始的习惯

```cpp
StreamConfig config;      // 用的是 Common.h 里的默认值
```

`StreamConfig` 有很多带默认值的成员（第 3 课）：`sourceType = Screen`、`width = 1920`、`videoKbps = 4500`、`audioDeviceIndex = -1`……

**构造一个默认的，然后只覆盖"界面能控制的字段"** —— 这是很好的写法：

- 不需要一个个列举所有字段（将来 `StreamConfig` 加新字段时，**这个函数一行都不用改**，新字段自动取默认值）；
- 读者一眼就能看出"界面到底控制了哪几个参数" （14 个字段里只改了 11 个）。

**反模式**（请大家不要这么写）：把 `StreamConfig` 的所有字段都手写一遍，包括那些界面上根本没有控件的。那样的代码会随契约变化而爆炸。

## 11.3 第 230~233 行：视频源

```cpp
config.sourceType = videoSourceCombo_->currentIndex() == 1 ? SourceType::Camera
                                                           : SourceType::Screen;
if (config.sourceType == SourceType::Camera)
    config.cameraIndex = std::max(0, cameraCombo_->currentIndex());
```

注意第 232 行的 `if` 写得很聪明：**它先判断了 sourceType，而不是无条件读 cameraCombo_**。这保证了"屏幕"模式下不会把摄像头的下标带进去。

`std::max(0, cameraCombo_->currentIndex())`：这是 `<algorithm>` 提供的模板函数（所以第 21 行 include 了它）。作用是把 `-1`（未选中）钳到 `0`。

> 注意这里的**类型安全**：`std::max` 的两个参数类型必须一致。`0` 是 `int`，`currentIndex()` 返回 `int` ✓。如果写成 `std::max(0u, ...)` 就编译不过了（unsigned vs int）—— 这是 `std::max` 常见的坑。

## 11.4 第 235~239 行：视频参数 —— userData 的兑现

```cpp
const QSize size = resolutionCombo_->currentData().toSize();
config.width = size.width();
config.height = size.height();
config.fps = fpsSpin_->value();
config.videoKbps = bitrateSpin_->value();
```

**第 235 行是第 9.2 节埋下的伏笔的收获**：分辨率的解析完全不需要碰字符串。

对比一下"糟糕的写法"：

```cpp
// ❌ 如果当初不用 userData，就得这样：
QString text = resolutionCombo_->currentText();      // "1280 x 720"
QStringList parts = text.split(" x ");               // ["1280", "720"]
config.width = parts[0].toInt();
config.height = parts[1].toInt();
```

看出来问题了吗？这种写法：

1. 依赖显示格式（哪天把 " x " 改成 "×" 就崩）；
2. 多了两次字符串解析（还可能有 failure）；
3. 性能差（每次 collection 都要 parse）；
4. **把"显示"和"数据"焊死了。**

**用 userData 就完全没有这些问题。** 请大家在自己的下拉框里推广这个做法。

> **`currentData()` 返回的是 `QVariant`。** 如果当前项没有 userData，`QVariant` 是无效的，`toSize()` 会返回一个空 `QSize`（宽高都是 -1）。幸运的是我们的每一项都设了 userData。**但这提醒我们：用 `currentData()` 前最好确认每一项都带了数据，或者加一个 `if (data.isValid())` 检查。**

## 11.5 第 241~244 行：音频参数

```cpp
config.audioType = audioSourceCombo_->currentIndex() == 0
                       ? AudioSourceType::SystemAudio
                       : AudioSourceType::Microphone;
config.audioDeviceIndex = audioDeviceCombo_->currentData().toInt();
```

第 244 行把"（系统默认）"这一项的 `-1` 直接传给 `audioDeviceIndex` —— 一路畅通地落到 `AudioCapturer::open(type, -1, log)`，意思是"用系统默认"。**哨兵值从 UI 一路传到内核，语义完整保留。**

## 11.6 第 246~248 行：三个硬编码常量

```cpp
config.sampleRate = 48000;
config.channels = 2;
config.audioKbps = 128;
```

这三个值：

- **没法让用户改**（界面上没有对应的控件）；
- **也不需要让用户改** —— 48000Hz/立体声/128kbps 是直播界的事实标准，几乎所有平台都接受，改了反而容易出问题（比如某些平台不支持 44100Hz）；
- 它们和第 3 课 `StreamConfig` 的默认值**恰好一致**（`sampleRate = 48000`、`channels = 2`、`audioKbps = 128`），所以第 246~248 行其实是"重复了一遍默认值"，可以删掉。

> **值得讨论的一个取舍**：既然和默认值一致，为什么要写这三行？
>
> - **支持写**：显式表达"UI 就是这么规定的"，将来改默认值时不会被意外带走；读代码的人知道这三个参数是 UI 决定的，不是"忘了设"；
> - **支持删**：重复了，违反 DRY（Don't Repeat Yourself）；应该在 `Common.h` 里定义一个 `kDefaultAudioConfig` 常量，两边都用它。
>
> 我倾向于后者：**把"这三个值"定义成一个命名的常量/结构，UI 和 Common 都引用它**。留作第 17 章的思考题。

## 11.7 第 250~272 行：多行地址解析 —— 本课唯一有"业务规则"的一段

这是整个 `collectConfig` 里唯一有"逻辑"的部分，因为它要处理一段自由文本。我们把它看成一个小型解析器。

```cpp
250      const QStringList lines = targetsEdit_->toPlainText().split('\n');
251      for (const QString& raw : lines) {
252          const QString line = raw.trimmed();
253          if (line.isEmpty() || line.startsWith('#'))
254              continue;
```

### 第 250 行：`split('\n')` 的小陷阱

`toPlainText()` 返回全部文本；`split('\n')` 按换行切成 `QStringList`。

**注意 `\n` 用的是单引号：它是 `QChar`（不是 `const char*`）。**`QString::split` 有 `QChar`、`QString`、`QRegularExpression` 三种重载 —— 这里匹配的是 `QChar` 版本，效率最高。

> **一个跨平台陷阱**：Windows 的换行习惯是 `\r\n`，Linux 是 `\n`。`QPlainTextEdit` 内部会把用户按回车产生的换行**统一存成 `\n`**（Qt 做了规范化），所以这里按 `\n` 切是安全的。**但如果你是从文件里读的配置**，就可能遇到 `\r\n`，那时需要 `QString::split(QRegularExpression("[\r\n]+"), Qt::SkipEmptyParts)`。**这就是为什么第 253 行的 `trimmed()` 很重要** —— 它能顺手把可能残留的 `\r` 去掉。

### 第 252~254 行：清洗 + 跳过规则

```cpp
const QString line = raw.trimmed();                    // 去掉首尾空白（含 \r）
if (line.isEmpty() || line.startsWith('#'))            // 空行 / 注释行
    continue;
```

两条跳过规则：

- **空行**：用户经常会按出一堆空行；
- **`#` 开头的行**：注释 —— 想临时停掉某一路，加个 `#` 就行，不用删。

**这是典型的"对用户宽容、对机器严格"的解析哲学输入处理原则**：接受各种合理的输入形式，把无意义的部分过滤掉，只对"看起来是想表达一件事"的内容做严格处理。

注意 **`trimmed()` 返回的是QString 副本**（QString 是不可变的 / COW），所以这里必须用一个新变量接住它，不能 `raw.trimmed()` 之后直接用 `raw`。

### 第 256~263 行：`名称=地址` 语法解析

```cpp
256          StreamTarget target;
257          const int separator = line.indexOf('=');
258          if (separator > 0) {
259              target.platform = line.left(separator).trimmed().toStdString();
260              target.url = line.mid(separator + 1).trimmed().toStdString();
261          } else {
262              target.url = line.toStdString();
263          }
```

**语法规则**：如果有 `=`，左边是平台名，右边是 URL；否则整行就是 URL。

用到的三个 `QString` 方法（这组 API 请记牢，日常写代码天天见）：

| 方法 | 作用 | 例子（`line = "bilibili = rtmp://a/b"`） |
|---|---|---|
| `indexOf('=')` | 找字符位置，找不到返回 **-1** | 返回 9 |
| `left(n)` | 取前 n 个字符 | `"bilibili"`（含尾空格） |
| `mid(pos)` | 从 pos 取到结尾 | `" rtmp://a/b"` |

为什么要加 `trimmed()`？因为要允许用户输入 `"bilibili = rtmp://..."`（等号两边带空格也能接受）。**这就是对用户宽容。**

**为什么判断 `separator > 0` 而不是 `separator >= 0`？**

- `>= 0` 表示"找到了 `=`"，包括它在**第 0 个位置**的情况；
- `> 0` 排除了 `=rtmp://...` 这种"以等号开头"的畸形输入 —— 那种情况下左边是空字符串，`platform` 会变空，走第 265 行的分支变成"路 N"，语义混乱。

更重要的是：**RTMP 的 URL 里可能本来就带 `=`！**

```
rtmp://live-push.bilivideo.com/live-bvc/?streamname=abc&key=xyz
                                                    ↑    ↑ 两个等号
```

这时候 `indexOf('=')` 找到的是**第一个**等号（在 `?streamname=` 里）！于是：

- `platform` = `"rtmp://live-push.bilivideo.com/live-bvc/?streamname"` ❌ 完全错了
- `url` = `"abc&key=xyz"` ❌ 更错

**这是一个真实的 bug**（第 17 章会列为局限），而且这种 bug 非常阴险 —— 它对"简单的地址"工作得很好，只对带查询参数的地址失效。

三种修法：

| 修法 | 做法 | 评价 |
|---|---|---|
| 只在 URL 方案标记之前找 | 找 `=` 之前先检查这一段**不含 `:` 和 `/`** | 能解决大部分情况 |
| 用正则匹配行首 | `QRegularExpression("^([A-Za-z0-9_\\-]+)=(.+)$")` | ★ 最稳健：平台名必须是"简单的标识符"，URL 部分可以包含任何东西 |
| 约定平台名不含某些字符 | 用户手册里写清楚 | 最弱，靠人 |

**推荐的做法是用正则**（第 16 章会给出具体代码）。请大家把这个坑记牢 —— **文本解析的 bug 往往是因为低估了输入的自由度**。

### 第 265~266 行：自动生成平台名

```cpp
265          if (target.platform.empty())
266              target.platform = "路" + std::to_string(config.targets.size() + 1);
```

如果用户没写名字（`platform` 为空，也就是"纯 URL 行"的情况），就自动叫"路 1""路 2"…… 编号从 `targets.size() + 1` 算起。

**为什么是 `size() + 1`？** 因为此时还没 `push_back`，所以这是"即将成为第几条"的自然编号。

> **小细节**：`"路" + std::to_string(n)` —— 这里左操作数是一个 C 风格字符串字面量（`const char[4]`），`std::to_string` 返回 `std::string`。C++ 提供了 `operator+(const char*, const std::string&)` 重载，所以能成立。如果是 `"路" + 1` 那就成了指针算术了 —— 这正是新手常见的错误。

### 第 268~269 行：URL 为空则跳过

```cpp
268          if (target.url.empty())
269              continue;
```

防的是这一行：**`bilibili=`**（有等号、有名，但没地址）。这种行显然是一句废话，跳过比塞一个空 URL 进去好 —— 否则管线会拿空 URL 去 `avformat` 打开，`StreamPipeline::Impl::openTarget` 会失败，最后日志里出现"推流输出打开失败"。

> **这就是"界面层校验"和"内核层校验"的关系**：界面层拦掉明显无效的输入（至少省得跑一趟）；但**内核层仍然要做同样的检查**（第 11 课 `start()` 里有 `if (config.targets.empty()) return fail("没有配置推流目标");`）。**两层都要检查，这是对的 —— 界面只是诸多调用者之一。**

### 第 271 行：收进 vector

```cpp
271          config.targets.push_back(target);
```

至此，一个 `StreamTarget` 完成。**注意第 256 行的 `StreamTarget target;` 声明在循环体内部** —— 每轮循环都是一个新的干净的 target。这是一个容易写错的地方：如果把 `target` 声明在循环外，就必须手动清空每个字段，否则上一轮的 `platform` 会残留（这种 bug 也非常常见，尤其在后来往这个循环里加字段的时候）。

## 11.8 `collectConfig()` 的完整数据流图

```
 targetsEdit_ 里用户输入的文本
        │
        │ toPlainText().split('\n')
        ↓
 ["bilibili=rtmp://a/b?k=v", "douyin=rtmp://c/d", "# 注释掉的一路", "", "rtmp://e/f"]
        │
        │ 逐行：trimmed() → 空/#跳过
        ↓
  ┌─────────────────────────────────────────────────┐
  │ 行 1: indexOf('=') > 0 → platform="bilibili"     │
  │                         url="rtmp://a/b?k=v"     │
  │ 行 2: platform="douyin"  url="rtmp://c/d"        │
  │ 行 3: # 开头 → 跳过                              │
  │ 行 4: 空 → 跳过                                  │
  │ 行 5: 无 '=' → url="rtmp://e/f"                  │
  │                 platform 自动 = "路 1"           │
  └─────────────────────────────────────────────────┘
        │
        ↓
 config.targets = 3 个 StreamTarget
```

再加上其他字段，最终产出：

```cpp
StreamConfig {
    sourceType        = Screen / Camera
    cameraIndex       = 用户选的下标
    width, height     = 720p / 1080p / 360p（来自 QVariant）
    fps               = 30
    videoKbps         = 2500
    audioType         = SystemAudio / Microphone
    audioDeviceIndex  = -1 或具体索引
    sampleRate        = 48000
    channels          = 2
    audioKbps         = 128
    targets           = [ StreamTarget{platform,url}, ... ]
}
```

这份结构完全是第 3 课定义的样子 —— **界面从头到尾没有要求内核为它改一行代码。这就是契约层的胜利。**

---

# 第 12 章 逐行教学（八）：★ `startStreaming()` —— 本课的代码高峰

终于到它了。**这 49 行（第 277~325 行）是整个第 12 课最值得研读的代码**，因为它把前面讲的所有东西串到了一起：状态机、原子变量、std::thread、lambda 捕获、互斥锁、跨线程投递、UI 更新的纪律。

**请同学们把这一节读三遍。**

## 12.1 全文

```cpp
277  void MainWindow::startStreaming()
278  {
279      if (starting_.load() || running_.load())
280          return;
281
282      const StreamConfig config = collectConfig();
283      if (config.targets.empty()) {
284          appendLog("还没有填推流地址");
285          return;
286      }
287
288      if (worker_.joinable())
289          worker_.join();
290
291      starting_ = true;
292      updateUiState();
293      appendLog(QString("正在连接 %1 路...").arg(config.targets.size()));
294
295      worker_ = std::thread([this, config] {
296          bool ok = false;
297          QString error;
298
299          {
300              std::lock_guard<std::mutex> lock(pipelineMutex_);
301              ok = pipeline_.start(config, [this](const std::string& message) {
302                                       postLog(message);
303                                   });
304              if (!ok)
305                  error = QString::fromStdString(pipeline_.lastError());
306          }
307
308          QMetaObject::invokeMethod(
309              this,
310              [this, ok, error] {
311                  starting_ = false;
312                  running_ = ok;
313
314                  if (ok) {
315                      elapsed_.start();
316                      statsTimer_->start();
317                      appendLog("推流已开始");
318                  } else {
319                      appendLog("启动失败: " + error);
320                  }
321                  updateUiState();
322              },
323              Qt::QueuedConnection);
324      });
325  }
```

> 注：第 301~303 行在源码里写成了三行（缩进不同），为了版面我把 lambda 折行了，语义完全一致。

## 12.2 先看全局流程图

在看代码之前，**请先在脑子里建立这张图**：

```
【GUI 线程】点击「开始推流」
   │
   ├─① 状态检查：starting_ || running_ ? 直接 return（防重入）
   │
   ├─② collectConfig()：界面 → StreamConfig（纯读，很快）
   │
   ├─③ 校验：targets 为空？写日志 + return
   │
   ├─④ 清理上一条线程：if (worker_.joinable()) worker_.join()
   │
   ├─⑤ starting_ = true;  updateUiState();  appendLog("正在连接 N 路...");
   │     ↑ 界面立刻给出反馈（按钮灰掉、状态字变"正在连接..."）★
   │
   └─⑥ worker_ = std::thread(...)    ← 开一条新线程，GUI 线程就此返回 ✔ 不卡界面
           │
           │  ╔══════════════【worker_ 线程】══════════════╗
           ├─→║ ⑦ lock(pipelineMutex_)                     ║
           │  ║ ⑧ pipeline_.start(config, logCallback)     ║ ← 可能阻塞几秒（连 RTMP）
           │  ║      └── 期间 logCallback 可能被多条线程调用 ║
           │  ║ ⑨ !ok ? error = pipeline_.lastError()      ║
           │  ║ unlock（作用域结束）                        ║
           │  ║                                            ║
           │  ║ ⑩ invokeMethod(this, lambda, QueuedConnection) ║
           │  ╚═════════════════ 线程结束 ══════════════════╝
           │                        │
           │                        ↓ 事件进 GUI 线程队列
【GUI 线程】└────────→ ⑪ 事件循环取到它 → 执行 lambda
                          starting_ = false;
                          running_ = ok;
                          ok ? (启秒表、启定时器、"推流已开始")
                             : ("启动失败: " + error)
                          updateUiState();    ← 按钮恢复/停止键亮起 ★
```

**三个关键点先记住**：

1. **GUI 线程在第 ⑥ 步之后就自由了**（不阻塞），所以它还能继续响应"点停止""拖窗口""看日志滚动"；
2. **耗时的连接有优化移到了 worker 线程**（第 ⑧ 步）；
3. **任何 UI 更新都必须回到 GUI 线程做**（第 ⑪ 步）—— 第 ⑧ 步期间产生的日志和第 ⑩ 步的终态更新都是走这个通道。

## 12.3 第 279~280 行：防重入（re-entrancy guard）

```cpp
279      if (starting_.load() || running_.load())
280          return;
```

**这是整个函数的第一道防线。** 它的作用是：**不允许重复启动。**

什么时候会被重复触发？虽然 `startButton_` 在推流中会被 `setEnabled(false)` 灰掉（第 352 行），看起来点不了。但**不要假设单一入口**：

| 可能的重复触发 | 说明 |
|---|---|
| 用户快速双击 | 两次 clicked 事件可能在按钮还没变灰之前就都排上了队（Qt 的事件循环处理有延迟） |
| 键盘 | Qt 按钮支持空格键激活，也可能连按 |
| 未来加菜单/快捷键 | 菜单里再加一个"开始推流"入口，信号从别处来 |
| 未来加命令行/DBus/HTTP 控制 | 那就更不通过按钮了 |

**所以"状态检查必须写在真正的执行函数里"，而不能只靠"按钮变灰"。** 这是一个通用的健壮性原则：

> **不要在 UI 的"表象"上做防护。**
> 换句话说：**界面的可用性（enabled）只是给人看的提示，真正的守卫必须在逻辑层。**

### 为什么这里要写 `.load()`

```cpp
starting_.load()      // 显式读取
starting_             // 也能编译（atomic 有 operator T() 隐式转换）
```

两者都能工作。**显式写 `.load()` 的好处是提醒读代码的人"这是一个跨线程共享的变量"** —— 这是很好的可读性实践。代价是多写 7 个字符。

## 12.4 第 282~286 行：收集参数与前置校验

```cpp
282      const StreamConfig config = collectConfig();
283      if (config.targets.empty()) {
284          appendLog("还没有填推流地址");
285          return;
286      }
```

两件事：

**① `const StreamConfig config`**（第 282 行）：

- **`const`**：这份配置生成之后不再修改；
- **`= collectConfig()`**（而不是 `const StreamConfig& config = ...`）：**这里是拷贝，而且拷贝是必须的**（下面要把它捕获进 lambda 里长期保存，不能留引用）。

**② 提前校验 + 友好提示**（第 283~286 行）：

这是**界面层的职责**：与其让用户点了开始、等了三秒、然后看到一行"推流输出打开失败"，不如**在动工之前就告诉他"你忘了填地址"**。这是"快速失败"（fail fast）原则。

> **注意这里是提前检查，不是等管线返回再说。** 第 11 课 `StreamPipeline::start()` 里其实也有同样的检查（`if (config.targets.empty()) return fail("没有配置推流目标");`）—— **两层都做**，这就是我们第 11.7 节说的"界面只是调用者之一，内核不能信任调用者"。

## 12.5 第 288~289 行：★ 容易被忽略的必填步骤

```cpp
288      if (worker_.joinable())
289          worker_.join();
```

**这两行看起来人畜无害，其实是防止程序崩溃的关键。**

为什么必须？回忆第 7.8.4 节讲的规则：

> **`std::thread` 的赋值运算符会把原来管理的线程对象析构掉；如果那条线程还是 joinable 状态，`std::terminate()` 被调用 —— 程序立刻终止。**

具体到这个场景：上一轮用户点了"停止"，`stopStreaming()` 里新建过一条线程做 `pipeline_.stop()`（第 342 行）。如果用户在那条线程跑完之前**又点了"开始"**（理论上按钮可能已经亮了），那么第 295 行 `worker_ = std::thread(...)` 就会试图覆盖它 —— 崩。

所以：**每次要给 `worker_` 赋新线程之前，都先结掉旧账。** 本项目三处一致地做了这件事（第 288、331、428 行），非常规范。

> **一个"差一点就更好"的问题**：这里的 `join()` 是在 **GUI 线程**执行的。如果上一条线程干的是 `pipeline_.stop()`，而 stop 卡在网络 flush 上，GUI 会短暂卡住。更好的做法是在 UI 上先禁用按钮，或者用超时 join（C++ 没有标准超时 join，需要用 `std::async` + `wait_for` 或者 condition_variable 手搓）。**本项目选择简单版，可接受，但要知道它的代价。**

## 12.6 第 291~293 行：立即给反馈（★ 用户体验的核心）

```cpp
291      starting_ = true;
292      updateUiState();
293      appendLog(QString("正在连接 %1 路...").arg(config.targets.size()));
```

这三行虽然短，但它们是**本课最有"产品感"的三行**。

**为什么？** 因为用户点下按钮之后，大脑最多等待 **200 毫秒**就会开始焦躁："它响应了吗？"

有了这三行：

- **状态字立刻变成"正在连接..."**（第 292 行 → `updateUiState()` 第 357 行）；
- **所有控件立刻灰掉**（第 292 行 → 第 364~375 行的 `editable = false`）；
- **日志立刻出现"正在连接 2 路..."**。

**这叫"即时反馈"（immediate feedback）**，是所有交互设计的第一原则。哪怕后台要跑三秒，用户也知道"程序听到了，正在做"。

> 请对比一下"错误的做法"：先开线程，等线程连上了再改 UI。那样用户点下去之后有三秒"什么都没发生"的窗口 —— 会再点一次（这就是为什么 12.3 节的防重入那么重要）。**很多时候，用户乱点的根源是软件反馈太慢。**

### 关于第 291~293 行的顺序

注意顺序是：**先改状态位 → 再刷新 UI → 最后写日志**。

- 状态位必须在 `updateUiState()` 之前改（否则 UI 读到的还是旧状态）；
- 日志放最后（无关紧要，但保持"先硬件后话术"的顺序更自然）。

### `QString("正在连接 %1 路...").arg(...)` 的两个细节

```cpp
QString("正在连接 %1 路...").arg(config.targets.size());
```

- `%1` 会被替换成 `arg()` 的值；更多参数是 `%2`、`%3`……；
- `config.targets.size()` 返回 `size_t`（无符号）—— `QString::arg` 有一个 `qulonglong` 重载能对上 ✓。

## 12.7 第 295 行：开线程 —— 本课最核心的一行

```cpp
295      worker_ = std::thread([this, config] {
```

拆成三份来看：

### ① `std::thread(lambda)` —— 线程从这里开始跑

构造 `std::thread` 对象的那一刻，新线程**立即开始执行** lambda。不需要额外的 `start()`（这是 std::thread 和 QThread 的区别之一：QThread 对象本身代表线程，需要 `start()`）。

### ② 捕获列表 `[this, config]` —— 有两种截然不同的东西

```cpp
[this, config]
  │      │
  │      └── 值捕获：把 StreamConfig 拷贝一份带走！
  └───────── 引用/指针捕获：捕获 MainWindow* 指针（不是对象！）
```

这是本课最需要讲清楚的一个细节。

**`config` 值捕获是必须的**：

```cpp
const StreamConfig config = collectConfig();      // 局部变量，出了函数就没了
worker_ = std::thread([this, config] { ... });
// ← 函数返回了，config 这个局部变量已经销毁。
//   但线程还活着，它手里必须有一份自己的拷贝！
```

如果写成 `[this, &config]`（引用捕获），那么线程跑起来的时候，`config` 指向的是一个**已经销毁的栈上对象** —— 典型的悬垂引用灾难。**这是多线程 lambda 最经典的死亡陷阱之一。**

> **铁律：跨线程 lambda 只值捕获（或 shared_ptr），绝不引用捕获栈上的局部变量。**

**`this` 捕获的是什么？** 捕获的是 `MainWindow` 对象的**地址**（指针大小的拷贝）。它指向的对象还在（因为窗口还没关），所以是安全的 —— **但前提是"对象活得比线程久"**，这正是我们在析构函数和 `closeEvent` 里 join 的原因（第 3.8 节纪律三）。

| 捕获方式 | 捕获了什么 | 安全吗 |
|---|---|---|
| `[this]` | 裸指针 8 字节 | **有条件安全**：对象必须活到线程结束 |
| `[this, &config]` | 引用局部变量 | ❌ **必崩**（悬垂引用） |
| `[this, config]` | 值拷贝 config | ✅ 安全（本项目） |
| `[=]` | 全部值捕获（含 this 也按"浅"拷指针） | 不推荐，看不清抓了什么 |
| `[&]` | 全部引用捕获 | ❌ **多线程里永远别写** |
| `[weak = QPointer(this)]` | Qt 提供的弱引用 | 进阶做法，可用 `QPointer` 判空（注意 `QPointer` 只能在有事件循环的线程判空，仍有局限） |

### ③ 为什么用 `std::thread` 而不是 `QThread`？

这是新手经常会问的问题。两者都能开线程，区别是：

| | `std::thread` | `QThread` |
|---|---|---|
| 归属 | C++ 标准库 | Qt 框架 |
| 模型 | 执行一个函数/lambda，**跑完就结束** | 一个"线程对象"，通常**派生并覆写 `run()`**，自带事件循环（可选） |
| 是否和 Qt 事件循环集成 | 否（要自己投递） | 是（`moveToThread` 后该对象上的槽在这个线程执行） |
| 适合场景 | ★ **一次性后台任务**（干活、报告结果） | 长期驻留的工作线程（需要接收/处理 Qt 事件） |
| 和 `QObject` 生命周期 | 无管理，**自己保证 join** | 有 `finished` 信号、`deleteLater` 等配套机制 |

**我们这个场景**：**干一件可能需要几秒钟的活（连 RTMP），干完报告结果** —— 这是典型的一次性后台任务，**`std::thread` + `invokeMethod` 是干净利落的选择**。

> 如果用 `QThread`，我们通常要：派生一个新类、覆写 `run()`、定义 signal 报告结果、`moveToThread` 或者在 `run()` 里跑……对于"十几行的一个任务"来说太重了。**这是工程上的常见判断：不要为了"更 Qt"而引入不必要的框架复杂度。**

## 12.8 第 296~306 行：worker 线程的正事

```cpp
296          bool ok = false;
297          QString error;
298
299          {
300              std::lock_guard<std::mutex> lock(pipelineMutex_);
301              ok = pipeline_.start(config, [this](const std::string& message) {
302                                       postLog(message);
303                                   });
304              if (!ok)
305                  error = QString::fromStdString(pipeline_.lastError());
306          }
```

### 第 296~297 行：两个"待会儿要交给 GUI 线程"的变量

```cpp
bool ok = false;      // 启动成功了吗
QString error;        // 失败的话，原因是什么
```

这两个变量是**为了跨越线程边界而存在的**：它们活在 worker 线程的栈上，然后**被值捕获进第 310 行的 lambda**，随着投递事件被拷贝到 GUI 线程。

**注意 `error` 用的是 `QString` 而不是 `std::string`**：因为第 305 行转换好之后，它要在 GUI 线程直接使用（第 319 行 `"启动失败: " + error`）。在工作线程用 QString **是安全的**（QString 是值语义、线程之间传递副本没问题；它的引用计数是原子的）。

### 第 299~306 行：一个故意缩短的作用域

```cpp
{
    std::lock_guard<std::mutex> lock(pipelineMutex_);   // ① 加锁
    ok = pipeline_.start(...);                          // ② 干活
    if (!ok) error = ...;                               // ③ 取错误
}                                                       // ④ 析构 → 自动解锁 ★
```

那个孤零零的大括号是干什么的？**它纯粹是为了限制锁的生命周期。**

为什么要在第 306 行就让锁释放？因为**锁一定要在第 308 行的 `invokeMethod` 之前释放**！原因有两层：

**第一层（死锁风险）**：如果还拿着 `pipelineMutex_`，同时 GUI 线程在 `pollStats()` 里也在等这把锁 —— 两边死锁的概率虽然不高（因为 invokeMethod 是异步的，不会立刻卡住 worker），但如果换成 `BlockingQueuedConnection` 就**必然死锁**。

**第二层（性能）**：`invokeMethod` 之后可能会排队等一会儿（GUI 线程在处理别的事件）。如果 worker 还占着锁，GUI 的 `pollStats()` 就得干等。**锁持有的时间越短越好，这是并发编程的常识。**

> 顺便把这个写法记下来：**`{ std::lock_guard lock(m); ... }` —— 一个裸的大括号包住锁，是 C++ 里控制临界区范围的标准手法。** 比写 `lock.unlock()` 好得多（RAII 自动解锁，异常也安全）。

### 第 301~303 行：★ 日志回调 —— 本课第二重要的两行

```cpp
ok = pipeline_.start(config, [this](const std::string& message) {
                         postLog(message);
                     });
```

**`pipeline_.start` 的第二个参数是 `LogCallback`**（第 3 课定义：`using LogCallback = std::function<void(const std::string&)>`）。我们传了一个 lambda，把接到的字符串转交给 `postLog()`。

**请同学们务必想清楚：这个 lambda 会被谁调用？** 答案（看第 11 课 `StreamPipeline::Impl`）：

```cpp
// StreamPipeline.cpp（第 11 课）
void emit(const std::string& message)
{
    std::lock_guard<std::mutex> lock(logMutex);     // ← 管线内部加了锁，串行调用回调
    if (log)
        log(message);
}

LogCallback guardedLog()
{
    return [this](const std::string& m) { emit(m); };
}
```

调用者包括：

| 调用者 | 所在线程 |
|---|---|
| `pipeline_.start()` 内部的各种 `say(...)` 日志 | **worker_ 线程** |
| `videoLoop()` 里的日志（采集失败、丢帧等） | **videoThread** |
| `audioLoop()` 里的日志 | **audioThread** |
| `writeLoop()` 里的日志（网络错误、重连等） | **writeThread** |
| `stop()` 里的日志 | 调用 stop 的那个线程 |

**至少四种不同的、非 GUI 的线程可能进来！**

所以 `postLog()` **必须**是线程安全的、异步的。**这就是第 3 章那一整套规矩的用武之地** —— 而 `postLog` 的实现正是在那里做投递（第 417~421 行，第 14.3 节细讲）。

> **如果没有那层投递会怎样？**`postLog` 直接去 `logView_->appendPlainText(...)`，那么 videoThread 会在不清醒的情况下访问 GUI 对象 —— 于是第 3.1 节那张"可能的现象清单"就会在你身后一一上演，平均每隔几十分钟崩一次，而且永远抓不到现场。

### 一个值得注意的细节：lambda 里为什么是 `[this]` 而不是别的

```cpp
[this](const std::string& message) { postLog(message); }
```

因为 `postLog` 是成员函数，需要 `this`。而它本身是线程安全的（内部投递），所以这里直接调用没问题。

**这里的生命周期假设**：`pipeline_` 是 `MainWindow` 的成员，只要 pipeline 还在跑（三条线程还在），`MainWindow` 就必须活着。这个假设由我们的 join 纪律保证 ✅。

## 12.9 第 308~323 行：★ `QMetaObject::invokeMethod` 的完整体现

```cpp
308          QMetaObject::invokeMethod(
309              this,                                  // ① 上下文对象 = MainWindow（GUI 线程）
310              [this, ok, error] {                    // ② 要在 GUI 线程执行的那段代码
311                  starting_ = false;
312                  running_ = ok;
313
314                  if (ok) {
315                      elapsed_.start();
316                      statsTimer_->start();
317                      appendLog("推流已开始");
318                  } else {
319                      appendLog("启动失败: " + error);
320                  }
321                  updateUiState();
322              },
323              Qt::QueuedConnection);                 // ③ 连接类型：排队投递
```

这是本课最核心的 API 调用。我们逐个参数讲。

### 参数 ①：`this` —— 上下文对象（context object）

它的作用是回答一个问题：**"这段代码要投递到哪个线程的事件队列？"**

Qt 的做法很简单：查 `this->thread()`。我们第 176 行创建了 `statsTimer_` 时 parent 是 this；而 `MainWindow` 本身是在 `main()` 的 GUI 线程里构造的（第 13 行），所以 `this->thread()` 就是 **GUI 线程** → **投递到 GUI 线程的事件队列** ✓。

**另外一层作用**：Qt 内部把这个 functor 包成一个 `QFunctorSlotObject`，**它的 QObject "context" 用于在对象销毁时清理**（第 3.8 节讲过）。

### 参数 ②：lambda —— 要执行的那段代码

```cpp
[this, ok, error] { ... }
```

**捕获列表是这个函数里最讲究的地方**：

| 捕获项 | 类型 | 为什么这么捕获 |
|---|---|---|
| `this` | 裸指针 | 要访问 `starting_`/`running_`/`statsTimer_`/`logView_` 等成员 |
| `ok` | `bool`，**值拷贝** | ✓ 正确。worker 线程栈上的 `ok` 马上就要随线程结束而消失，必须拷走 |
| `error` | `QString`，**值拷贝** | ✓ 正确，同理 |

**如果这里写成 `[&]` 会怎样？** `ok` 和 `error` 都会被"引用捕获"，而 lambda 在 GUI 线程、很久之后（相对而言）才执行 —— 那时这两个栈变量早随 worker 线程栈一起销毁了。**读取悬垂引用 = 未定义行为**，可能打印出"启动失败: <乱码>"，也可能直接崩。

> **再强调一次这条最容易写错的规则**：
> **★ `invokeMethod` / 异步队列投递的 lambda，捕获的局部变量一律值捕获（`[=]` 或小心的显式列表），永远不要 `[&]`。**
> 唯一允许引用捕获的是"保证活到 lambda 执行完"的对象（比如类成员通过 `[this]` 间接访问）。

### lambda 体：第 311~321 行

```cpp
311                  starting_ = false;
312                  running_ = ok;
```

**第一步永远是更新状态机。** 顺序很重要：**先清 `starting_` 再设 `running_`**。

如果反着来（`running_ = ok; starting_ = false;`），中间那一瞬间两个都是 true —— 如果这时正好有个 `updateUiState()` 从别处被调用（比如某个没被禁用的控件触发），会看到一个不一致的状态。**虽然概率极低，但写对了不费力。**

> 想要更强的保证，可以用 while false 的原子交換、或者干脆用一个 `enum class State` + 单个 atomic 变量。这是一种更好的设计（第 17 章思考题会讨论）—— 因为两个 bool 理论上能表达 **4 种**组合，而实际只有 **3 种**合法状态，**多出来的那一种（starting && running）是非法的，却能被表达出来**。这是"布尔标志组合"的通病。

```cpp
314                  if (ok) {
315                      elapsed_.start();
316                      statsTimer_->start();
317                      appendLog("推流已开始");
318                  } else {
319                      appendLog("启动失败: " + error);
320                  }
321                  updateUiState();
```

成功分支做三件事：

| 行 | 动作 | 为什么必须在这里做 |
|---|---|---|
| 315 | `elapsed_.start()` | 秒表归零开始。放在 GUI 线程做，因为这块 QElapsedTimer 只在 GUI 线程读写（第 383 行的 pollStats 也在这里读） |
| 316 | `statsTimer_->start()` | 启动 500ms 定时器。**注意 `QTimer::start()` 不是线程安全的 —— 必须从它所属的线程调用** ✓ 我们正好在 GUI 线程 |
| 317 | `appendLog("推流已开始")` | 给用户反馈 |

> **`statsTimer_->start()` 这行特别能说明问题**：Qt 的 `QTimer::start()` 内部要操作定时器所属的线程状态，**它明确不是线程安全的**。如果我们在 worker 线程里直接调用 `statsTimer_->start()`，行为未定义。所以必须投递到 GUI 线程。
>
> 这也解释了为什么代码的组织是" **在 worker 里收集信息 → 投递 → 在 GUI 线程里执行所有 UI 操作**"，而不是"worker 里事无巨细都干了"。

失败分支：写一行日志。**注意第 319 行用了 `error`** —— 就是第 305 行从 `pipeline_.lastError()` 取出来的那个。这样用户能看到真正的失败原因（"Connection refused" 还是 "Invalid data"），而不是一句"失败了"。

**最后第 321 行 `updateUiState()`**：无论成功失败，状态机变了，UI 要跟着变。

### 参数 ③：`Qt::QueuedConnection`

**如果省略这个参数会怎样？** 第 3.5 节讲过：默认是 `Qt::AutoConnection`。

在我们的场景里（调用者是 worker 线程，接收者是 GUI 线程），AutoConnection 也会走 Queued —— 功能上是对的。

**但为什么还要显式写？**

1. **显式 > 隐式**：读代码的人一眼看到 `Qt::QueuedConnection`，立刻知道"这里是有意的跨线程投递"，而不是"碰巧跨线程了"；
2. **行为一致**：万一将来某天这个 lambda 被调用时恰好在 GUI 线程（比如代码重构后），显式 Queued 仍然保证"异步执行"，不会突然变成同步；
3. **这是 Qt 官方推荐的做法**：Qt 文档明确建议在跨线程通信时**显式指定连接类型**。

> **纪律一再一次出现**：**跨线程投递，显式写 `Qt::QueuedConnection`。**

### 关于返回值

`QMetaObject::invokeMethod` 是有 `bool` 返回值的：

```cpp
bool success = QMetaObject::invokeMethod(this, [...]{...}, Qt::QueuedConnection);
```

返回 `false` 的情况：目标对象可能已经销毁/没有有效的 `metaObject`（基本不会）。**我们的代码忽略了返回值**。

要不要检查？**在这个场景可以忽略**：

- 如果 `this` 已经析构，我们的 join 纪律保证 worker 线程不可能在那时还活着；
- Queued 投递本身几乎不会失败。

> 但如果这是一个"必须保证被执行"的操作（比如"告诉 GUI 出错了"），就值得检查并处理：
> ```cpp
> if (!QMetaObject::invokeMethod(...)) {
>     // 投递失败，退而求其次：至少不要丢掉这条错误
> }
> ```

## 12.10 第 324 行：为什么 lambda 一结束线程就没了

```cpp
324      });
```

lambda 的最后一行执行完（`invokeMethod` 调用返回），worker 线程就**寿终正寝**了。

注意：**投递出去的 lambda 不是在这个线程执行的**（它是被拷进事件队列，等待 GUI 线程去取）。所以 worker 线程在"发出这个待办事项"之后就可以安心退出了。这就是第 3.4 节那张图的完整兑现。

```
worker_ 线程的一生：
      t0: 开始执行
      t0+3ms: 拿到 pipelineMutex_
      t0+3ms ~ t0+2.5s: pipeline_.start() 阻塞在连 RTMP ★ 期间可能有多线程日志
      t0+2.5s: start 返回，解锁
      t0+2.5s: invokeMethod 投递 lambda（微秒级）
      t0+2.5s: lambda 结束 → 线程结束 ✔
      
GUI 线程：
      t0+2.5s: 事件队列里多了一个 QMetaCallEvent
      t0+2.5s + 一点点: 事件循环取到它 → 执行 lambda
                    → starting_=false, running_=true
                    → 启秒表、启定时器、写日志、刷新 UI ✔
```

**注意" Pipeline 的三条线程continue 活着"**：`start()` 成功后，管线内部的 video/audio/write 三条线程开始了它们的工作。**它们产生的日志会通过第 301 行的 LogCallback → `postLog` → 持续投递到 GUI 线程。** 也就是说：**worker_ 线程结束了，但"跨线程日志通道"仍然开着，而且后面还有三条线程在用它。**

这就是为什么我们在多个场合强调：**跨线程投递的通道，比这一次线程任务活得更久。**

## 12.11 反事实推演：如果我们要写错，会错成什么样

为了让同学们加深印象，我们做三组"反面教材"，请务必能自己指出错误：

### 反例 1：不开线程，直接在槽里启动

```cpp
void MainWindow::startStreaming()          // ❌
{
    starting_ = true;
    updateUiState();
    pipeline_.start(collectConfig(), ...);   // GUI 线程阻塞 2.5 秒！
    running_ = true;
    updateUiState();
}
```

后果：

- **界面冻住 2.5 秒**（这段时间用户什么都做不了，标题栏可能显示"未响应"）；
- 而且 `updateUiState()` 之前那句虽然能执行到，但 Qt 的 repaint 事件也排在队列里排队 —— **连"正在连接"这几个字都画不出来**（因为第一次绘制要等这个槽返回之后）！用户看到的是"点了没反应"。

### 反例 2：开了线程，但 lambda 里直接改 UI

```cpp
worker_ = std::thread([this, config] {      // ❌
    bool ok = pipeline_.start(config, ...);
    if (ok) {
        statsTimer_->start();               // ❌ 从 worker 线程操作 GUI 对象
        appendLog("推流已开始");             // ❌ 同上
    }
    running_ = ok;
    updateUiState();                        // ❌ 同上，整整 13 行的 UI 操作
});
```

后果：第 3.1 节那张清单。运气好的话能跑；运气不好随机崩溃。

### 反例 3：投递了，但用了引用捕获

```cpp
worker_ = std::thread([this, config] {
    bool ok = pipeline_.start(config, ...);
    QMetaObject::invokeMethod(this, [this, &ok] {   // ❌ &ok 引用捕获！
        running_ = ok;                              // 读取已经销毁的栈变量
    }, Qt::QueuedConnection);
});
```

后果：worker 线程结束后 `ok` 的栈空间被回收（实际上是被后续函数调用 reused）。GUI 线程晚一点读取时，拿到的是垃圾值 —— 可能显示"推流中"但实际没推，也可能反之。**这种 bug 特别难查，因为它依赖于调度时序。**

> **请大家把这三个反例和正解对比着看，这就是"学好跨线程 UI 编程"的最短路径。**

---
# 第 13 章 逐行教学（九）：`stopStreaming()` 与 `updateUiState()`

## 13.1 `stopStreaming()`（第 327~350 行）

```cpp
327  void MainWindow::stopStreaming()
328  {
329      statsTimer_->stop();
330
331      if (!running_.exchange(false)) {
332          updateUiState();
333          return;
334      }
335
336      if (worker_.joinable())
337          worker_.join();
338
339      updateUiState();
340      appendLog("正在停止...");
341
342      worker_ = std::thread([this] {
343          {
344              std::lock_guard<std::mutex> lock(pipelineMutex_);
345              pipeline_.stop();
346          }
347          QMetaObject::invokeMethod(
348              this, [this] { appendLog("已停止"); }, Qt::QueuedConnection);
349      });
350  }
```

### 第 329 行：先停定时器

```cpp
statsTimer_->stop();
```

**无条件先停**（不看状态）。这一步很 cheap 而且很安全：

- 如果没在跑，停了也就停了（`stop()` 是幂等的）；
- 如果在跑，接下来 `pipeline_` 要被 stop 了，`pollStats()` 再跑也没意义（数据不再增长），而且它还要争抢 `pipelineMutex_` —— **先停掉能避免和 stop 抢锁**。

这个顺序体现了一种好习惯：**"要拆东西之前，先把还在不停操作的部件停下来。"**

### 第 331~334 行：★ `running_.exchange(false)` —— 为什么要原子交换

```cpp
331      if (!running_.exchange(false)) {
332          updateUiState();
333          return;
334      }
```

这是本课第二个需要重点理解的点。把它拆成两步就明白了：

```cpp
// exchange(false) 的语义：
//   1. 把 running_ 设置成 false
//   2. 返回 running_ 修改之前的那个值
const bool wasRunning = running_.exchange(false);
if (!wasRunning) {          // 之前就是 false → 本来就没在跑
    updateUiState();
    return;
}
// 之前是 true → 确实在跑，继续往下做停止动作
```

**为什么不能写成两步？**

```cpp
// ❌ 错误写法（两步之间有时间窗）
if (!running_.load()) { return; }      // ← 读到这里 running_ == true
running_ = false;                      // ← 写
```

在单线程 GUI 逻辑里，这两步之间"看起来"没有任何代码。但请想想：

- **`starting_` / `running_` 是被多条线程共享的原子变量**（ worker 线程会在第 312 行设 `running_ = ok`）；
- 如果 worker 线程刚好在这两步之间完成启动并写入 `running_ = true`，然后紧接着我们的 GUI 线程执行 `running_ = false` —— **状态就错了**：管线在跑，但 `running_` 说没在跑。

虽然在这个具体场景概率极低（stop 按钮运行时才亮），但**这是并发里最经典的 TOCTOU**（Time Of Check To Time Of Use，检查时刻与使用时刻不一致）问题。

> **记住这条原则：凡是"判断状态 + 修改状态"必须一口气完成的，用原子的读改写操作（`exchange`、`compare_exchange`），不要用"先读再写"。**
>
> 这和我们第 3 课 `BoundedQueue::tryPop` 用的思路是一脉相承的：**原子操作是原子的，因为它是 CPU 一条不可打断的指令；"读 + 写"两条指令之间一定有空隙。**

**注意 $exchange$ 的返回值用法还有一个副产品**：它还顺便把 `running_` 置为 false 了 —— 也就是说第 339 行的 `updateUiState()` 看到的已经是"不在跑了"的状态。看不出这是 bug 的话请再读一遍：**`exchange(false)` 无论之前是什么值，执行完之后 `running_` 一定是 false** ✓ 这正是我们要的。

### 第 331 行的另一个作用：幂等保护

`stopStreaming()` 可以被调用多次（用户连点停止 / 关窗口时也调用）。有了这个检查，第二次调用会直接 `updateUiState(); return;` —— **不会重复 stop 管线**。

> 顺便一提：`StreamPipeline::stop()` 本身也是幂等的（第 11 课的实现里有一个"没在跑就把清理做完返回"的分支）。**两层幂等**和前面说的"两层校验"是同样的道理。

### 第 336~337 行：又是那条必填步骤

```cpp
336      if (worker_.joinable())
337          worker_.join();
```

第三次出现这个 idiom（另两处在第 288、433 行）。

这一处的语义是：**如果上次那条线程还在跑（比如还在 `pipeline_.start()` 里连 RTMP），就等它跑完**。为什么必须等？

因为接下来我们要在**另一条新线程**（第 342 行）里调用 `pipeline_.stop()`。如果不等，**两条线程同时操作管线** —— 一条在 start、一条在 stop。**虽然有 `pipelineMutex_` 保护，但结果仍然很糟糕**：start 会走到一半被 stop 打断（或者反过来），管线进入半死不活的状态。

> 顺带又一次提醒：**这一步是在 GUI 线程里阻塞等待的。** 如果 `pipeline_.start()` 卡在网络连接上（比如对方服务器不响应，要等 10 秒超时），**用户会觉得"点了停止，界面卡死了"**。这是本项目已知的一个体验问题（第 17 章的局限清单会写进去）。
>
> 想彻底解决，可以把"正在连接"也建模成一个可取消的操作（比如给 RTMP 打开加超时、或者给 worker 加一个 `stopRequested_` 标志），让停放在 start 自己身上。**这是第 14 课（`c2b8021` 断线重连）会涉及的话题。**

### 第 339~340 行：立即反馈 + 一个有趣的小差异

```cpp
339      updateUiState();
340      appendLog("正在停止...");
```

和 `startStreaming()` 的第 291~293 行是同一个套路：**先刷 UI，再写日志**。区别是这里**没有写状态字**（因为 343~349 行的停止是异步的，`updateUiState()` 里 `running_` 已经 false，`starting_` 也是 false → 状态字显示"未开始"）。

> 这里有个体验上的小瑕疵可以说一下：用户点了"停止"之后，状态字立刻变成"未开始"，但**管线还在线上收尾中要几百毫秒**。严格说这期间应该显示"停止中"。
>
> 修法很简单：**再加一个状态位 `stopping_`**（第三个 bool！）或者更好 —— **用一个 `enum class State { Idle, Starting, Running, Stopping }` 加单个 atomic**（这正是第 7.8.2 节说的"用 enum 替代多个 bool"的理由，也是第 17 章思考题的第 3 题）。

### 第 342~349 行：停止线程 —— 和 start 几乎一样的模板

```cpp
342      worker_ = std::thread([this] {
343          {
344              std::lock_guard<std::mutex> lock(pipelineMutex_);
345              pipeline_.stop();
346          }
347          QMetaObject::invokeMethod(
348              this, [this] { appendLog("已停止"); }, Qt::QueuedConnection);
349      });
```

和 `startStreaming` 的 worker 对比：

| | start 的 worker | stop 的 worker |
|---|---|---|
| 捕获 | `[this, config]`（要带一份配置拷贝） | `[this]`（不需要额外数据） |
| 局部变量 | 有 `ok` / `error` 要传出去 | 无 |
| 临界区 | `pipeline_.start(config, logCb)` | `pipeline_.stop()` |
| 投递的 lambda | `[this, ok, error] { 更新状态机 + 日志 }` | `[this] { appendLog("已停止"); }` |

**注意第 348 行投递的 lambda 没有捕获任何局部变量** —— 所以它不存在"引用捕获悬垂"的风险（第 12.9 节那个坑）。这提醒我们：**越简单的捕获越安全，能不带数据就不带。**

**为什么 `pipeline_.stop()` 也要放线程里？** 因为它也是阻塞的（第 11 课的实现：要把三条线程都 join 回来，每条线程最长阻塞一个超时周期，加上 `output.flush()` 把队列里剩下的包写完）。整个过程几百毫秒到几秒不等 —— **够让界面明显卡一下了**。

### 一个"看不见"的协作：停止时谁负责刷新 UI？

把 `stopStreaming` 完整串起来看：

```
GUI 线程                                  停止 worker 线程
──────────                               ──────────────
statsTimer_->stop();
running_.exchange(false) → true
worker_.join();  (等旧线程)
updateUiState();      ← UI 立刻变成"未开始"
appendLog("正在停止...");
开新线程 ─────────────────────────────→  lock(mutex)
（GUI 线程返回了！用户可以继续操作）      pipeline_.stop()   ← 阻塞几百毫秒
                                          unlock
                                          invokeMethod(this, ..., Queued)
                                          线程结束
...某时...
事件循环取到事件 → appendLog("已停止") ✓
```

**用户看到的是**：点"停止" → 立刻按钮变灰、状态变"未开始" → 几百毫秒后日志出现"已停止"。**全程界面不卡。** 这就是我们想要的。

## 13.2 `updateUiState()`（第 352~376 行）

```cpp
352  void MainWindow::updateUiState()
353  {
354      const bool starting = starting_.load();
355      const bool running = running_.load();
356
357      startButton_->setEnabled(!starting && !running);
358      stopButton_->setEnabled(running && !starting);
359
360      if (starting)
361          stateLabel_->setText("正在连接...");
362      else if (running)
363          stateLabel_->setText("推流中");
364      else
365          stateLabel_->setText("未开始");
366
367      const bool editable = !starting && !running;
368
369      videoSourceCombo_->setEnabled(editable);
370      cameraCombo_->setEnabled(editable);
371      refreshCameraButton_->setEnabled(editable);
372      resolutionCombo_->setEnabled(editable);
373      fpsSpin_->setEnabled(editable);
374      bitrateSpin_->setEnabled(editable);
375
376      audioSourceCombo_->setEnabled(editable);
377      audioDeviceCombo_->setEnabled(editable);
378      refreshAudioButton_->setEnabled(editable);
379
380      targetsEdit_->setEnabled(editable);
381  }
```

> 注：第 369~380 行是十处 `setEnabled(editable)`，与源码逐条对应。

### 这个函数为什么重要

它是**第 8.2 节提到的"统一刷新函数"模式**的落地。**整个程序里所有"界面可不可用"的判断，全在这 25 行里。** 好处：

- 加一个新的"推流时不许改"的控件，只需要加一行；
- 想知道"推流中界面长什么样"，看这一个函数就够了；
- 状态的变化点（哪里改了状态位）和界面的绘制点（这里）分离 —— 改状态的三个地方都只需要调一次它。

### 第 354~355 行：先读一遍快照

```cpp
const bool starting = starting_.load();
const bool running = running_.load();
```

**为什么要先读两个局部变量，而不是直接用 `starting_.load()` 写满全篇？**

两个理由：

1. **可读性**：`!starting && !running` 比 `!starting_.load() && !running_.load()` 干净得多；
2. **一致性**：这两个值在这一次调用里应该保持"同一时刻的快照"。如果每次都重新读原子变量，理论上中间可能被别的线程改掉（虽然概率极低），导致这次刷新里出现"前半段用旧值、后半段用新值"的混合状态。

> 这是一个小但很好的习惯：**跨线程共享状态，读一次存成局部快照，然后统一用快照做判断。**

### 第 357~358 行：两个按钮的使能逻辑

```cpp
startButton_->setEnabled(!starting && !running);      // 只有"未开始"才能点开始
stopButton_->setEnabled(running && !starting);        // 只有"真的跑起来了"才能点停止
```

把真值表写出来：

| starting_ | running_ | 开始 | 停止 | 状态字 | editable |
|---|---|---|---|---|---|
| false | false | ✅ 可点 | ❌ | 未开始 | ✅ 可编辑 |
| **true** | false | ❌ | ❌ | 正在连接... | ❌ |
| false | **true** | ❌ | ✅ 可点 | 推流中 | ❌ |
| true | true | ❌ | ❌ | 正在连接...（非法！） | ❌ |

最后一列那一行是**理论上非法但能被表达出来的状态**（第 12.9 节讨论过）。可以看到这里的应对是：`stopButton_` 要求 `running && !starting` —— **即使出现非法状态也是"最保守"的表现**（两个按钮都不能点），不会让程序崩。

> **这是一个很好的防御设计：当状态不确定时，选择"什么都不能做"而不是"什么都让做"。** 这个原则在 UI、在权限系统、在并发控制里都适用。

### 第 360~365 行：状态字的三分支

```cpp
if (starting)
    stateLabel_->setText("正在连接...");
else if (running)
    stateLabel_->setText("推流中");
else
    stateLabel_->setText("未开始");
```

**注意 `starting` 的判断在最前面** —— 这在"非法状态（两者都 true）"下会优先显示"正在连接..."，符合直觉。

三个状态字都是三到五个字，**宽度接近**（这也是为了避免标签宽度抖动的细节之一）。

### 第 367~380 行：`editable` 变量 —— 一句话省掉十行重复

```cpp
const bool editable = !starting && !running;
```

**这个变量是本函数最值得学的一手。** 如果没有它，代码会变成：

```cpp
videoSourceCombo_->setEnabled(!starting && !running);
cameraCombo_->setEnabled(!starting && !running);
refreshCameraButton_->setEnabled(!starting && !running);
...  // 十个一模一样的表达式
```

**提取一个语义化的名字（`editable`），让这段代码的意图从"三个布尔的运算"变成"能不能编辑"。** 这就是"给表达式起名字"的价值 —— 而且改判据时只改一行。

> 顺便提一句 `const`：这里 `const bool editable` 是加 const 的好例子 —— **任何"算出来就不打算再改"的局部变量都应该加 const**。它能防止你在后面不小心写 `editable = false;` 这种 bug，也让读代码的人放心。

### 哪些控件被"推流时禁用"了，哪些没有？

数一数第 369~380 行，一共 **10 个**控件：

- 视频组 6 个（来源、设备、刷新、分辨率、帧率、码率）；
- 音频组 3 个（来源、设备、刷新）；
- 地址框 1 个。

**为什么全禁用？** 因为这些都是**"启动参数"** —— `collectConfig()` 在 `start()` 的那一刻读一次就固定了，**推流过程中改它们没有任何效果**（改了也不会作用到已经在跑的管线上）。

> **这正是界面设计里最容易犯的一个错误**：允许用户在"改了也没用"的控件上操作，然后用户以为改了。**最好的做法是干脆禁掉，让用户知道"这几个东西现在是锁死的"。**
>
> 而如果你想真的支持"推流中改码率"（OBS 支持），那就要走另一条路：**加一条动态控制通道发给编码器**（`x264_encoder_reconfig` 之类），而不是偷偷去改 `config`。这是第 17 章的延伸思考。

### 没有被禁用的控件

注意剩下的：**两个按钮**（单独处理了）、**四个统计标签**（只读，无所谓）、**日志框**（只读）。所以一个都不需要额外的处理。**这种"分类处理"的完整性很重要** —— 加新控件时要问一句："它应该被 `editable` 控制吗？"

---

# 第 14 章 逐行教学（十）：统计、日志与关闭

## 14.1 `pollStats()`（第 383~409 行）

```cpp
383  void MainWindow::pollStats()
384  {
385      PipelineStats stats;
386      {
387          std::lock_guard<std::mutex> lock(pipelineMutex_);
388          stats = pipeline_.stats();
389      }
390
391      const qint64 totalSeconds = elapsed_.elapsed() / 1000;
392      durationLabel_->setText(
393          QString("时长 %1:%2:%3")
394              .arg(totalSeconds / 3600, 2, 10, QLatin1Char('0'))
395              .arg((totalSeconds / 60) % 60, 2, 10, QLatin1Char('0'))
396              .arg(totalSeconds % 60, 2, 10, QLatin1Char('0')));
397
398      videoLabel_->setText(QString("视频 %1 帧 / %2 包 / 丢 %3")
399                               .arg(stats.videoFramesCaptured)
400                               .arg(stats.videoPacketsEncoded)
401                               .arg(stats.videoFramesDropped));
402
403      audioLabel_->setText(
404          QString("音频 %1 块 / %2 包").arg(stats.audioChunksCaptured).arg(
405              stats.audioPacketsEncoded));
406
407      targetLabel_->setText(
408          QString("连接 %1/%2").arg(stats.connectedTargets).arg(stats.totalTargets));
409  }
```

### 第 385~389 行：先捞快照，快速释放锁

```cpp
PipelineStats stats;
{
    std::lock_guard<std::mutex> lock(pipelineMutex_);
    stats = pipeline_.stats();          // 拷贝一份快照出来
}
// ★ 到这里锁已经释放了，后面的字符串格式化不再持有锁
```

**这是一个非常重要的并发习惯：临界区里只做"必须做的事"，把耗活的活挪到锁外面。**

这里的"必要的事"是 `pipeline_.stats()`（它内部读的是一堆 atomic；`output.targets()` 会拷贝一个 vector）。`"格式化四个字符串 + 更新四个 Label"` 都可以、也应该在锁外面做。

**如果把整个函数都放进锁里会怎样？** GUI 线程会一直占着 `pipelineMutex_` 直到 500ms 的槽跑完（虽然也就几十微秒，但更新 QWidget 会触发布局计算和重画事件，可能排队很久）。这期间 worker 线程想 stop/start 就得干等。**缩短临界区是并发编程的头号优化手段。**

### 第 391~396 行：时长的格式化

```cpp
const qint64 totalSeconds = elapsed_.elapsed() / 1000;
durationLabel_->setText(
    QString("时长 %1:%2:%3")
        .arg(totalSeconds / 3600, 2, 10, QLatin1Char('0'))
        .arg((totalSeconds / 60) % 60, 2, 10, QLatin1Char('0'))
        .arg(totalSeconds % 60, 2, 10, QLatin1Char('0')));
```

三步都要看清：

1. **`elapsed_.elapsed()`** 返回毫秒（`qint64`），除以 1000 得秒；
2. **`%1:%2:%3`** 分别是时、分、秒；
3. **`.arg(值, 2, 10, QLatin1Char('0'))`** 是第 4.6 节讲的四参数版 `arg`：最小宽度 2、十进制、不足补 `'0'`。

于是 `1 小时 2 分 3 秒` 会显示成 `时长 01:02:03`，而不是 `1:2:3`。**细节到位，界面才显得专业。**

> **`QLatin1Char`** 是 Qt 提供的一个轻量类型，表示"一个 Latin-1（ASCII/西欧）字符"。它避免了把一个 char 转成 `QString`（那是 2~4 字节到 UTF-16 的完整转换，还要分配内存）。**属于"知道就捡起来用"的性能小习惯。**

### 第 398~408 行：四个统计数字

```cpp
videoLabel_->setText(QString("视频 %1 帧 / %2 包 / 丢 %3")
                         .arg(stats.videoFramesCaptured)   // int64_t → qlonglong 重载
                         .arg(stats.videoPacketsEncoded)
                         .arg(stats.videoFramesDropped));
```

三个字段的含义（第 11 课 `PipelineStats`）：

| 字段 | 含义 | 为什么值得展示给用户 |
|---|---|---|
| `videoFramesCaptured` | 抓了多少帧画面 | 反映采集是否正常（一直不动 = 采集中断了） |
| `videoPacketsEncoded` | 编码出了多少个 packet | 反映编码器是否在干活 |
| `videoFramesDropped` | 丢了多少帧 | ★ **最重要的健康指标**：丢帧多 = CPU 不够或带宽不够 |

**"丢帧数"这个指标特别值得表扬**：一个有经验的推流软件一定会把它摆在最显眼的位置。因为：

- 丢 0~3 帧：正常（启动时的对齐损耗）；
- 丢帧**持续缓慢增长**：CPU 跟不上，应该降分辨率/帧率/码率；
- 丢帧**快速增长**：网络带宽不够或者编码器卡死了。

界面把 `videoFramesCaptured` 和 `videoPacketsEncoded` 放在一起展示还有个额外的好处：**正常情况下两者应该同步增长**（一帧进、一包出，B 帧/GOP 会有小的滞后）。如果只涨帧数不涨包数，说明编码器卡住了。**这就是"让数据自己说话"的设计。**

```cpp
targetLabel_->setText(
    QString("连接 %1/%2").arg(stats.connectedTargets).arg(stats.totalTargets));
```

`连接 1/2` 这样的展示，**让"多路推流"这个功能变得可见** —— 用户能立刻知道"哦，我配了两路，其中一路没连上"。这正是第 10 课多目标分发的 UI 兑现。

### 这个函数在哪个线程执行？

**必须强调一次**（第 4.6 节埋过的伏笔）：`pollStats` 是被 `statsTimer_->timeout` 触发的，**而 `statsTimer_` 的 parent 是 `this`（属于 GUI 线程）**，所以 `timeout` 在 GUI 线程发出 → **`pollStats` 在 GUI 线程执行** → 里面直接 `setText()` 完全合法，**不需要投递**。

对照一下第 3.9 节总表的第 5 行确认 ✓。

> **反例思考**：假设某天有人把 `statsTimer_` 改成 `new QTimer(nullptr)`（没 parent），并且在某个 worker 线程里 `statsTimer_->start()`。那么 `timeout` 会在哪个线程发？答案是**创建它的那个线程**（而 timer 对象自己会被 `moveToThread` 影响……）。总之：**QTimer 的线程归属变了，`pollStats` 的执行线程也就变了，那里就不再能直接碰 UI。**

## 14.2 `appendLog()`（第 411~415 行）

```cpp
411  void MainWindow::appendLog(const QString& message)
412  {
413      logView_->appendPlainText(
414          QString("[%1] %2").arg(QTime::currentTime().toString("HH:mm:ss")).arg(message));
415  }
```

简短但有三个知识点：

### ① 时间戳

`QTime::currentTime().toString("HH:mm:ss")` → `"20:31:02"`。

> **如果想毫秒级**（排查"两条日志到底谁先谁后"时很有用）：`toString("HH:mm:ss.zzz")`。

### ② `appendPlainText()` 自带的行为

- 自动在末尾追加一行（自带换行）；
- **自动限流**：受第 154 行 `setMaximumBlockCount(2000)` 限制；
- **自动跟随滚动**：文字追加后，QPlainTextEdit 会保持滚动条在最下方吗？**默认不一定** —— 但 Qt 的 `appendPlainText` 内部调用了 `ensureCursorVisible()`，通常会把光标移到末尾，**日常使用大多表现正常**。如果日志很多发现不滚动，可以显式加一句：
  ```cpp
  logView_->moveCursor(QTextCursor::End);      // 谨慎使用会让用户的选中失效
  // 更斯文的做法：只在滚动条已经在底部时才自动滚
  ```

### ③ 它是"GUI 专用"的（★ 重要约定）

**`appendLog` 只能在 GUI 线程调用。** 这是通过"它直接操作 `logView_`"这一事实决定的。

搭配 `postLog` 形成了完美的分工：

```
任何地方想写日志？
   ├── 你在 GUI 线程（比如按钮槽里）→ 直接 appendLog("...")  ✅
   └── 你在别的线程（比如 worker / 管线内部线程）→ postLog(msg) ✅
                                                    ↓ 内部投递
                                                 appendLog(...)
```

**这就是"用命名表达线程安全性"的实践**（第 7.6 节提过）：**`post` 前缀 = 线程安全入口**。

## 14.3 ★ `postLog()`（第 417~422 行）

```cpp
417  void MainWindow::postLog(const std::string& message)
418  {
419      const QString text = QString::fromStdString(message);
420      QMetaObject::invokeMethod(this, [this, text] { appendLog(text); },
421                                Qt::QueuedConnection);
422  }
```

**这 6 行是第 3 章全部理论的最小落地样本。** 我们把它掰开看：

### 第 419 行：先在调用方线程做转换

```cpp
const QString text = QString::fromStdString(message);
```

两个考量：

1. **`std::string → QString` 的转码在任何线程都能做**（QString 是值类型，构造出来的这个副本只属于当前线程）；
2. **把"转换"这个偏 CPU 的活放在调用方线程**，让 GUI 线程在真正执行 lambda 时只做一次 `appendPlainText`，**减少 GUI 线程的负担**（GUI 线程的时间应该留给"响应交互"）。

### 第 420~421 行：值捕获 + 显式 Queued

```cpp
QMetaObject::invokeMethod(this, [this, text] { appendLog(text); },
                          Qt::QueuedConnection);
```

- **`[this, text]`**：`text` 是**值拷贝**（QString 是隐式共享，拷贝本身 O(1)，只有在 GUI 线程里被修改时才会真的深拷贝 —— 这里没人改它，所以全程零拷贝）；
- **`Qt::QueuedConnection`**：显式写明，理由在第 3.5 节（一致性 + 可读性）；
- **`this`**：投递目标 = GUI 线程。

### 为什么它是"跨线程通道"里最忙的一条

回顾一下谁会调用 `postLog`（通过第 301 行的 lambda 间接）：

```
pipeline_.start() 里的 say(...)           → worker_ 线程 ┐
videoLoop() 里的 emit(...)                → videoThread  ┤
audioLoop() 里的 emit(...)                → audioThread  ├→ postLog()
writeLoop() 里的 emit(...)                → writeThread  │
pipeline_.stop() 里的 emit(...)           → 调用 stop 的线程 ┘
```

**五条不同来源的线程，共用同一个通道。** 而且它们可能**同时**想说话 —— 好在 `StreamPipeline::Impl::emit()` 内部有 `logMutex` 保证串行：**同一时刻只有一个线程在执行`log(message)` 这条回调链**（所以 `postLog` 本身不会被并发调用；即使被并发调用，`invokeMethod` 也是线程安全的）。

### 一个性能话题：日志洪水

如果某处的 512 行开始每毫秒产生一条日志（比如某个 Errors loop 里丢帧告警），会发生什么？

```
每秒 1000 条 postLog → 每秒 1000 个 QMetaCallEvent 进 GUI 队列
→ GUI 线程每秒处理 1000 次 appendPlainText
→ 每次都触发 QPlainTextEdit 的内部文档重排 + 可能的重画
→ 界面掉帧，用户感觉"卡"
```

**这是"跨线程 UI 更新"容易忽略的一个成本**：投递本身很便宜，**但接收端每次都要做一次完整的 UI 更新**。Qt 有 `__postEvent` 的事件压缩优化（同名事件会被合并），但对 `QMetaCallEvent` 不适用。

> **工业界的解法：日志攒批。** 不在每条日志到来时都投递，而是攒够 N 条或者攒够 100ms，一次性投递一批。**这是第 17 章的思考题之一。**
>
> 我们这个项目目前的情况还好：日志频率低（每秒几条到十几条），所以简单的实现完全够用。**过度优化是万恶之源 —— 先做对的，等真的慢了再优化。**

## 14.4 `closeEvent()`（第 424~437 行）

```cpp
424  void MainWindow::closeEvent(QCloseEvent* event)
425  {
426      statsTimer_->stop();
427
428      if (running_.exchange(false) || starting_.exchange(false)) {
429          std::lock_guard<std::mutex> lock(pipelineMutex_);
430          pipeline_.stop();
431      }
432
433      if (worker_.joinable())
434          worker_.join();
435
436      QMainWindow::closeEvent(event);
437  }
```

### 为什么必须覆写这个函数

如果不覆写，用户点 × 时会发生什么？Qt 的默认行为是**接受关闭** → 窗口销毁 → `~MainWindow()` → join 线程 → 看起来也没问题？

**但有两个问题：**

1. **管线还在跑**！析构时才 join `pipeline_.stop()`，而 `pipeline_.stop()` 是阻塞的 —— 用户在点击关闭之后还要等几百毫秒到几秒窗口才真的消失，**而且这段时间窗口已经"看不见"了，像是卡住**；
2. **更重要**：万一有人在某个路径下绕过了析构（比如 `deleteLater`），或者将来把管线改成非 Qt 管理的资源，清理就会漏掉。

**显式覆写 `closeEvent` 的价值：把"关闭窗口"这件事变成一个有完整处置流程的动作。**

### 第 426 行：先停定时器

和 `stopStreaming()` 第 329 行同一个道理。

### 第 428 行：★ 一行里的两个 `exchange` 和短路求值

```cpp
if (running_.exchange(false) || starting_.exchange(false)) {
```

**这一行非常有意思，值得停下半分钟看。** 它用了 C++ 逻辑或运算符 `||` 的**短路求值**（short-circuit evaluation）特性：

```cpp
A || B        // 如果 A 为 true，B 根本不会被求值！
```

于是：

| 情况 | `running_.exchange(false)` | `starting_.exchange(false)` 是否会执行 | 结果 |
|---|---|---|---|
| 正在推流 | 返回 true → 短路 | ❌ 不执行 | 进 if，stop |
| 正在启动中 | 返回 false → 不短路 → 执行 | ✅ 执行，返回 true | 进 if，stop |
| 都没事 | false → false | ✅ 执行，返回 false | 不进 if |

这个写法很紧凑，但有两点要说：

1. **它的确是对的**：无论哪种情况，`running_` 和 `starting_` 最终都被清成 false ✓（`exchange(false)` 无论原值是什么，执行后都是 false）；
2. **但可读性一般**：第一次读的人要愣一下才能意识到"哦，前面短路了后面就不会执行"。

> **更清晰的等价写法**（代价是多写两行）：
> ```cpp
> const bool wasRunning  = running_.exchange(false);
> const bool wasStarting = starting_.exchange(false);
> if (wasRunning || wasStarting) { ... }
> ```
> **我更推荐这种写法** —— 课程里讲到"清晰 > 巧妙"的时候，这就是一个很好的例子。**代码是写给人看的，顺便让机器执行。**

### 第 429~430 行：在 GUI 线程同步停管线

```cpp
429          std::lock_guard<std::mutex> lock(pipelineMutex_);
430          pipeline_.stop();
```

**注意这里是同步停止**（不像 `stopStreaming` 那样开线程）。为什么这里可以？

- 关窗口的时候，用户已经预期"程序要退出了"，**短暂的等待是可以接受的**；
- 而且这样写**有决定性好处**：`closeEvent` 返回之前，管线一定已经停了。**确定性**比"界面不卡"更重要。

> 但请注意这确实会让关闭变慢（几百毫秒到几秒）。如果 `pipeline_.stop()` 因为网络原因卡住（比如 `output.flush()` 在等 TCP 写出），**用户会觉得"关不掉"**。
> **权衡**：关闭时清理是"必须做对"的事，慢一点可接受。**这也是很多专业软件的表现：点关闭之后要等一下才退出（它们通常在关闭前显示一个"正在保存/正在停止"的模态提示）。**

另外注意：**这个 `std::lock_guard` 的作用域是整个 if 块**（第 428~431 行的大括号），所以第 430 行执行完之后锁在第 431 行释放 ✓（和第 12.8 节那个故意缩短的 `{ }` 是同一个套路，只不过是 if 块自带的）。

### 第 433~434 行：最后一道 join

```cpp
433      if (worker_.joinable())
434          worker_.join();
```

第三处（也是最后一处）join。它保证了：**`closeEvent` 返回之后，绝对没有任何后台线程持有 `this`。**

于是第 436 行 `QMainWindow::closeEvent(event)` 执行时可以放心：销毁流程可以开始了。

### 第 436 行：别忘了调用基类

```cpp
436      QMainWindow::closeEvent(event);
```

**这一行不能省！** 基类 `QWidget::closeEvent` 的默认实现是 `event->accept()` —— 也就是"同意关闭"。

如果忘了调用：

- `QCloseEvent` 的默认状态其实是 `isAccepted() == true`，所以**大概率还是会关**（这就是为什么很多人忘了写也没发现）；
- 但 **Qt 不保证**；而且子类有 veto 权力的正确写法应该是显式 accept/ignore，依赖默认值是不好的习惯。

> **通用规则：覆写 Qt 的事件处理函数时，如果不打算自己处理这条事件，请把它交给基类：**
> ```cpp
> void MyWidget::keyPressEvent(QKeyEvent* e) {
>     if (e->key() == Qt::Key_Escape) { close(); return; }   // 只处理我关心的
>     QBaseClass::keyPressEvent(e);                          // 其余的交给基类
> }
> ```

### 一个遗漏：这里没有更新 UI

注意 `closeEvent()` **没有调用 `updateUiState()`**。因为窗口马上就要销毁了，界面长什么样已经不重要了。**这是正确的取舍** —— 不要为了"整齐"去做马上就要被丢弃的工作。

> 类似的：这里也没有 `appendLog("已退出")`。因为在同步 stop 的过程中，`pipeline_` 已经通过 `postLog` 投递了"推流已停止"这条日志（第 11 课 `stop()` 最后有 `emit("推流已停止")`），而这条日志很可能**永远不会被显示**（它的目标对象马上就要析构了，事件会被 `removePostedEvents` 清掉）。
>
> **这提醒我们一件事**：异步投递的"丢失"在关闭流程里是正常的、也是应该被接受的。**不要为了"保证日志一定显示"而在关闭路径上做出别扭的同步等待。**

---
# 第 15 章 动手环节：编译、运行、验收清单、七个实验

## 15.1 编译：两条命令就够了

项目根目录里有一个一键脚本 `build.bat`（第 1 课就写好了）。在 `x64 Native Tools Command Prompt`（或者任意配好 MSVC 的终端）里执行：

```bat
cd C:\Users\cutcutzjj\WorkBuddy\2026-10-17-19-56-46\Streaming-tools
build.bat
```

它内部大致做两件事（等价于手敲）：

```bat
:: ① 配置（只在第一次或 CMakeLists 改了之后才需要重跑）
cmake -B build -S . ^
  -G "Visual Studio 17 2022" -A x64 ^
  -DCMAKE_TOOLCHAIN_FILE=C:/vcpkg/scripts/buildsystems/vcpkg.cmake

:: ② 编译 Debug 版
cmake --build build --config Debug
```

> 之所以要在 VS 的 Native Tools 命令行里跑，是因为 **MSVC 的 `cl.exe` 需要一堆环境变量**（`INCLUDE`、`LIB`、`PATH`）。普通 cmd 里没有。VS Code / CLion 的用户则需要配置对应的 kit。

### 编译产物在哪

根 CMakeLists 第 18 行设了 `set(CMAKE_RUNTIME_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR})`，所以：

```
build/
└── Debug/  (MSVC 多配置生成器时会有这一层)
    ├── LightLiveStreamer.exe        ★ 我们今天的主角
    ├── mini_pusher.exe              (第 2 课那个 demo)
    ├── test_common.exe              (第 3 课)
    ├── test_output.exe              (第 10 课)
    └── ... 其他测试
```

（MinGW / Ninja 单配置生成器则在 `build/` 根目录下。）

### 确认 moc 真的跑了（★ 别忘了这一分钟）

编译输出里应该能看到（MSVC）：

```
[1/xx] Automatic MOC for target LightLiveStreamer
[2/xx] Building CXX object ...\LightLiveStreamer_autogen\mocs_compilation.cpp.obj
[3/xx] Building CXX object ...\MainWindow.cpp.obj
[4/xx] Building CXX object ...\main.cpp.obj
...
```

或者去第 5.5 节说的目录里直接看有没有 `moc_MainWindow.cpp`。**两边都对得上，说明 moc 参与工作了。**

## 15.2 运行：三种方式各自的坑

### 方式一：命令行启动（推荐第一次用这个）

```bat
cd build\Debug
LightLiveStreamer.exe
```

**如果提示"找不到 Qt6Widgets.dll"**（第一次几乎必然遇到），说明 Qt 的 DLL 不在系统 PATH 里。三条解决道路：

```bat
:: 道路 A（临时，仅当前窗口有效）：把 Qt 的 bin 加进 PATH
set PATH=C:\Qt\6.5.0\msvc2019_64\bin;%PATH%
:: 如果是 vcpkg 装的 Qt：
set PATH=C:\vcpkg\installed\x64-windows\bin;%PATH%
LightLiveStreamer.exe

:: 道路 B（一次性发布）：用 windeployqt 把依赖 DLL 拷到 exe 旁边
windeployqt LightLiveStreamer.exe
LightLiveStreamer.exe

:: 道路 C（长期）：把 Qt/bin 永久加进系统环境变量 PATH（控制面板里改）
```

> **`windeployqt` 是 Qt 官方的部署工具**，它会扫描 exe 的依赖，把需要的 Qt DLL **以及 Qt 插件**（尤其是 `platforms/qwindows.dll` —— 缺了这个会报 "no Qt platform plugin could be initialized"）一起拷到 exe 所在目录。**发布给别人用的时候必须走这一步。**

### 方式二：在 Visual Studio 里跑

用 `CMake` 打开项目（"文件 → 打开 → CMake"），选中 `LightLiveStreamer.exe` 作为启动项，F5 调试。

**如果想让 VS 自动帮你配好 Qt DLL 的环境**，可以在 `src/app/CMakeLists.txt` 里加：

```cmake
set_target_properties(LightLiveStreamer PROPERTIES
    VS_DEBUGGER_ENVIRONMENT "PATH=$(Qt6_DIR)/../../../bin;$ENV{PATH}")
```

（路径需要根据你的 Qt 安装调一下，CMake 提供的 `TARGET_FILE:Qt6::Core` 等生成器表达式能拼得更准。）

### 方式三：直接双击 exe

**本次提交下，双击会先弹一个黑控制台窗口**，然后才是 Qt 窗口（第 5.4 节讲过的 `WIN32_EXECUTABLE` 没加）。别慌，这是正常的，不是编译错了。想干掉它，加上那个属性即可（见实验一）。

## 15.3 界面到手：先对着这张图认一认

第一次启动之后，你应该看到这样的窗口（920×700）：

```
┌─ LightLiveStreamer ─────────────────────────────────────────── □ ─ × ─┐
│  ┌── 视频源 ──────────────────────┐  ┌── 音频源 ───────────────────┐  │
│  │ 来源   [ 屏幕            ▾ ]   │  │ 来源  [ 系统声音       ▾ ]  │  │
│  │ 设备   [              ▾ ][刷新]│  │ 设备  [ (系统默认)    ▾ ][刷新]│ │
│  │ 分辨率 [ 1280 x 720    ▾ ]    │  │                              │  │
│  │ 帧率   [ 30 fps        ]      │  │                              │  │
│  │ 码率   [ 2500 kbps     ]      │  │                              │  │
│  └───────────────────────────────┘  └──────────────────────────────┘  │
│  ┌── 推流地址 ──────────────────────────────────────────────────────┐ │
│  │ （灰色提示：每行一个地址，可以写成「名称=地址」方便区分：...）      │ │
│  └───────────────────────────────────────────────────────────────────┘ │
│  [ 开始推流 ]  [ 停止推流 ](灰)      未开始                              │
│  时长 00:00:00    视频 -    音频 -    连接 -                             │
│  ┌── 日志 ──────────────────────────────────────────────────────────┐ │
│  │ [20:31:02] 就绪。填好推流地址后点「开始推流」。                      │ │
│  │ [20:31:02] 枚举到 1 个摄像头                                       │ │
│  └───────────────────────────────────────────────────────────────────┘ │
└────────────────────────────────────────────────────────────────────────┘
```

**注意日志的两条顺序**（第 8.2 节讲过构造顺序）：

1. 构造函数最后才 `appendLog("就绪...")`（第 32 行）；
2. 但**枚举摄像头的日志在它之前**（第 27 行的 `refreshVideoDevices()` → 第 198 行）。

所以实际顺序是：

```
[20:31:02] 枚举到 1 个摄像头
[20:31:02] 就绪。填好推流地址后点「开始推流」。
```

**如果发现顺序反了或者少了第一条，说明你看的不是这一版代码。** 这种"从日志顺序反推构造顺序"的小核对，是验证自己理解的好办法。

## 15.4 操作验收清单（逐条打勾）

请你按下面这张表一步步做，每步都要看到对应的现象。**这是本课最重要的动手环节。**

| # | 操作 | 预期现象 | 对应代码 | ☐ |
|---|---|---|---|---|
| 1 | 启动程序 | 窗口出现；日志有"枚举到 N 个摄像头"（或"没有枚举到"）；最后一行"就绪…" | §8.2 | ☐ |
| 2 | 视频"来源"切到**摄像头** | "设备"这一行**出现**（选屏幕时它是隐藏的） | §10.3 `setRowVisible` | ☐ |
| 3 | 切回"屏幕" | "设备"这一行**消失** | 同上 | ☐ |
| 4 | 点视频"刷新" | 日志新增一行"枚举到 N 个摄像头"；下拉框内容重建、**当前选择保持** | §10.1 `findText(previous)` | ☐ |
| 5 | 音频"来源"切到**麦克风** | 音频设备下拉框内容立刻变（= 重新枚举） | §10.2 + 第 167 行 connect | ☐ |
| 6 | 看音频设备的第一项 | 永远是"**（系统默认）**" | 第 209 行 `addItem(..., -1)` | ☐ |
| 7 | 什么都不填直接点"开始推流" | **日志出现"还没有填推流地址"**，没有任何后台动作 | 第 283~286 行 | ☐ |
| 8 | 在地址框填两行（下面有两个本地/同机 RTMP 的例子） | 占位灰字消失 | 第 110~113 行 | ☐ |
| 9 | 点"开始推流" | ★ **立刻**（不是几秒后）出现：<br>· 状态字"正在连接..."<br>· 所有控件变灰<br>· 开始按钮变灰<br>· 日志"正在连接 2 路..." | 第 291~293 行 + §13.2 | ☐ |
| 10 | 拖动/最小化/还原窗口 | **界面依然响应自如**（这证明 start 确实不在 GUI 线程）★ | 第 295 行 | ☐ |
| 11 | 连接成功后 | 状态字"推流中"；停止按钮亮起；日志"推流已开始" | 第 314~317 行 | ☐ |
| 12 | 观察统计区每半秒刷新 | 时长递增（补零 `00:00:07`）；视频帧数/包数递增；丢帧缓慢增长；连接数 N/M | §14.1 | ☐ |
| 13 | 观察日志继续滚动 | 不断有来自管线各线程的日志（**这证明跨线程投递在工作**）★ | §14.3 | ☐ |
| 14 | 点"停止推流" | 界面**立刻**变回"未开始"、控件恢复可用；日志"正在停止..."；约几百毫秒后"已停止" | 第 339~349 行 | ☐ |
| 15 | 统计定时器 | 停完后**时长停止不再增长**（定时器被 stop 了） | 第 329 行 | ☐ |
| 16 | 推流中点 × 关闭窗口 | **不会崩**；也没有留下孤儿进程（任务管理器里检查）★ | §14.4 | ☐ |
| 17 | 连续反复启动/停止 5 次 | 一切正常，没有崩溃、没有重复 счет（Join保证了这点） | 第 288/336 行 | ☐ |

> 第 10、13、16 三条加星的项目，是本次课三个核心技术点的**直接验证**：线程化（GUI 不卡）、跨线程投递（日志能显示）、生命周期（关窗不崩）。**请务必亲眼看到。**

### 地址框里可以填什么

没有 RTMP 服务器也能跑！第 10 课讲过，FFmpeg 的输出 url 也可以是**本地文件**：

```
路1=file:///C:/temp/live_a.flv
路2=file:///C:/temp/live_b.flv
```

或者在本机 1935 端口起一个 RTMP 服务（nginx-rtmp / SRS / MediaMTX 都行），然后填 `rtmp://127.0.0.1:1935/live/test`。

**推荐第一次用本地文件方式**：

- 不需要任何服务器；
- 产物能用 VLC 直接播（验证帧、音视频是否同步）；
- 还能让你明显体会到"多目标分发"在工作（生成两个文件，内容一样）。

## 15.5 七个改造实验（做完这七个，你就真的会了）

每个实验都建议**先自己想答案，再动手改**，改完用上面的清单复验一遍。

### 实验一：干掉讨厌的黑控制台（15 分钟）

在 `src/app/CMakeLists.txt` 里加一行：

```cmake
add_executable(LightLiveStreamer
  main.cpp
  MainWindow.cpp
  MainWindow.h
)

set_target_properties(LightLiveStreamer PROPERTIES WIN32_EXECUTABLE TRUE)   # ← 加这句

target_link_libraries(LightLiveStreamer PRIVATE
  core_pipeline
  Qt6::Widgets
)
```

重新 cmake + 编译，**再双击 exe** —— 黑窗口没了。

**留意一下副作用**：如果在某处写了 `printf`（现在没有），输出就看不到了。**这就是为什么 GUI 程序要用日志而不是 printf。**

### 实验二：把跨线程纪律破坏一次（★ 强烈建议做，30 分钟）

把 `postLog` 改成"直接操作界面"（第 14.2 节提醒过这是错的）：

```cpp
// MainWindow.cpp 第 417 行，临时改成：
void MainWindow::postLog(const std::string& message)
{
    const QString text = QString::fromStdString(message);
    logView_->appendPlainText(text);        // ← 故意犯错
}
```

然后：**Debug 版本 + 推流十分钟以上**（推到本地文件即可，不需要真服务器）。

你大概率会观察到以下之一：

- 日志正常（说明你运气好）；
- 日志错乱/丢行；
- 崩溃，栈落在 `QTextDocument`/`QWidget` 里；
- Qt 的调试断言：`ASSERT failure in QCoreApplication::sendEvent: "Cannot send events to objects owned by a different thread"`。

**做完后一定把代码改回来。** 这个实验的价值在于：**你亲眼见过的 bug，比课本上读过的规矩记得牢十倍。**

如果你一次都没崩，请在 `appendPlainText` 前加一行 `QThread::msleep(1);` 或者在推流时疯狂拖动窗口 —— 你会更容易看到效果。**这正是"并发 bug 依赖时序"的活证明。**

### 实验三：给 `refreshAudioDevices` 加上"记住上次选择"（20 分钟）

照着第 10.1 节的做法改第 202~216 行。要点：

```cpp
void MainWindow::refreshAudioDevices()
{
    const int previousDevice = audioDeviceCombo_->currentData().toInt();   // 记住的是"索引"而不是文本

    ... clear() + addItem(...) ...

    const int index = audioDeviceCombo_->findData(previousDevice);         // ★ 按 userData 找
    if (index >= 0)
        audioDeviceCombo_->setCurrentIndex(index);
}
```

**思考：为什么这里用 `findData(int)` 而不是 `findText(QString)`？**（答案在本章末的思考题 3）

顺手把"清空时不要误触发信号"也解决了 —— 用 `QSignalBlocker`（第 10.1 节提过）。

### 实验四：修掉 RTMP URL 里等号那个坑（30 分钟）

目标输入：

```
bilibili=rtmp://live-push.bilivideo.com/live-bvc/?streamname=abc&key=xyz
```

**期望**：`platform="bilibili"`，`url="rtmp://...?streamname=abc&key=xyz"`。
**现状**（第 11.7 节分析过）：会被第一个 `=`（在 `?streamname=` 里）切断，全错。

用正则改写第 257~263 行：

```cpp
static const QRegularExpression kNamedTarget("^([A-Za-z0-9_\\-]+)\\s*=\\s*(.+)$");
const QRegularExpressionMatch match = kNamedTarget.match(line);
if (match.hasMatch()) {
    target.platform = match.captured(1).toStdString();
    target.url      = match.captured(2).toStdString();
} else {
    target.url = line.toStdString();
}
```

**别忘了：** 需要 `#include <QRegularExpression>`，并且 `QRegularExpression` 对象应该**静态存着**（编译一次模式，重复复用）——编译正则是有成本的，别放在循环里每次构造。

验证方法：在 `collectConfig()` 末尾加一行日志，把解析出来的 platform/url 打出来：

```cpp
appendLog(QString("解析结果: 路%1 [%2] %3")
              .arg(i+1)
              .arg(QString::fromStdString(t.platform))
              .arg(QString::fromStdString(t.url)));
```

### 实验五：把 `<QFont>` 显式 include 补上（5 分钟）

在 `MainWindow.cpp` 的 include 区加 `#include <QFont>`（第 4.6 节提过的隐式依赖）。

顺便检查：**还有哪些"用了却没显式 include"的东西？**（提示：看看 `QVariant`、`QStringList`、`QLatin1Char` 各自对应的头文件，本项目今天用到了它们，但并没有全部显式 include。）

### 实验六：给"停止中"加一个真实状态（40 分钟，进阶）

第 13.1 节提过：现在点停止之后，状态字立刻显示"未开始"，但后台还在收尾。

改造目标：推一个 **`std::atomic<State>`** 或者再加一个 `stopping_` 状态位，让：

- 点停止之后 → 显示"停止中..."；
- 停止线程完成、投递回来之后 → 显示"未开始"。

**进阶做法**（推荐）：用 `enum class State { Idle, Starting, Running, Stopping }` 加单个 `std::atomic<State>`，彻底消灭"两个 bool 的非法组合"。

完成后对照第 13.2 节的真值表重写 `updateUiState()`。**你会发现代码变短了、变清晰了** —— 这就是"用好数据建模"的威力。

### 实验七：给日志加一个"行数和速率"的监控（30 分钟，可选）

在 `appendLog` 里加一个计数器，在 `pollStats` 里顺便显示出来：

```cpp
// 头文件加： std::atomic<int> logLines_{0};   // 需要 #include <atomic>

void MainWindow::appendLog(const QString& message)
{
    logLines_.fetch_add(1);
    logView_->appendPlainText(...);
}
```

然后观察：**推流 10 分钟后日志多少行？**（接近 2000 了吗？）**如果超过 2000，`setMaximumBlockCount` 就开始丢最老的了** —— 用这个实验亲眼看一遍"内存护栏"在工作。

---

# 第 16 章 排错手册：14 条清单

这一章是"出问题时翻的地方"。我按**编译期 / 链接期 / 运行期 / 行为层**四类整理。

## 16.1 编译期错误（4 条）

### ① `fatal error C1083: 无法打开包括文件: "QMainWindow": No such file or directory`

**原因**：目标没有继承到 Qt 的 include 路径。
**排查**：

```cmake
target_link_libraries(LightLiveStreamer PRIVATE Qt6::Widgets)   # ← 这一句在吗？
```

**注意必须是 `Qt6::Widgets`（带 `Qt6::` 前缀的 imported target），不能写成老的 `${Qt6Widgets_LIBRARIES}`。**

### ② `fatal error C1083: 无法打开包括文件: "core/pipeline/StreamPipeline.h"`

**原因**：`#include "core/pipeline/StreamPipeline.h"` 的查找基目录是 `src/`，而 `src/` 不在本目标的 include 路径里。
**根因**：`target_link_libraries` 里没有 `core_pipeline`，或者 `core_pipeline` 的 `target_include_directories` 写成了 `PRIVATE`（那样就不会传播出来）。

```cmake
# src/core/pipeline/CMakeLists.txt 必须是 PUBLIC：
target_include_directories(core_pipeline PUBLIC ${CMAKE_CURRENT_SOURCE_DIR}/../..)
```

### ③ `error C2039: 'MainWindow': is not a member of 'Ui'` / `ui_MainWindow.h 找不到`

**原因**：你用了 Qt Designer 的 `.ui` 文件的写法（`#include "ui_MainWindow.h"`），但本项目是**纯手写代码**，没有 `.ui` 文件。

**对照思路**：Qt 有两种写界面的方式 —— `.ui` 由 `uic` 生成一个 `Ui::MainWindow` 类和 `setupUi()`（配合 `AUTOUIC`）；我们今天是手写 `buildUi()`。**不要把两种写法混着用。**

### ④ 中文全是乱码（"鎺ㄦ祦宸插紑濮"之类）

**原因**：MSVC 在 Windows 上按本地编码（GBK）解释源码里的字节，而你的源文件是 UTF-8。
**解决**（第 1 课已经在根 `CMakeLists.txt` 加好了）：

```cmake
if(MSVC)
  add_compile_options(/utf-8)
endif()
```

**另外确认两件事**：

- 源文件真的是 **UTF-8**（VS：文件 → 高级保存选项）；
- 如果不带 BOM 又有问题，改成 **UTF-8 with BOM** 试一下（MSVC 的老毛病）。

## 16.2 链接期错误（3 条）

### ⑤ ★ `LNK2001: 无法解析的外部符号 "public: virtual struct QMetaObject const * __cdecl MainWindow::metaObject(void)const " (?metaObject@MainWindow@@UEBAPEBUQMetaObject@@XZ)`

或者 Linux/gcc 版：

```
undefined reference to `vtable for MainWindow'
undefined reference to `MainWindow::staticMetaObject'
```

**这是 Qt 最常见、也最让人抓狂的错误，原因有且只有三种：**

| 可能原因 | 检查方法 | 解决 |
|---|---|---|
| **头文件没被 moc 处理**（AUTOMOC 没扫到） | 去 build 目录找 `moc_MainWindow.cpp`，有没有？ | 把 `MainWindow.h` 加进 `add_executable` 的源文件列表（§5.1） |
| **加了 `Q_OBJECT` 之后没重新 cmake** | CMake 缓存里的源文件列表还是旧的 | 删除 `build/` 重新 cmake（或至少 `cmake .` 重跑一次配置） |
| **`Q_OBJECT` 写了，但所在的 `.h` 在 include 路径之外** | 是不是拼错了类名/文件名？ | 检查文件名和 include 语句 |

> **诊断口诀**：**moc 没生成 → 一定是 CMake 没看到那个头文件。** 两者必居其一。

### ⑥ `LNK2019: 无法解析的外部符号 ... QPushButton::staticMetaObject`

**原因**：链接了 Qt 但没链接 Widgets 模块（只 link 了 Qt6::Core），或者 `find_package(Qt6 REQUIRED COMPONENTS Widgets)` 里漏了 Widgets。

### ⑦ `LNK2019: 无法解析的外部符号 avcodec_...` / `swscale_...`

**原因**：app 间接依赖了 FFmpeg 但没链接。检查 `core_capture` / `core_encode` / `core_output` 的 `target_link_libraries` 里有没有 `${FFMPEG_LIBRARIES}`，以及它们是不是 `PUBLIC`（要能传播到 app）。

> **注意**：本项目今天**不应该**出现这个错误 —— 因为 `main.cpp` 和 `MainWindow.cpp` 里没有一行 FFmpeg 代码。如果出现了，说明你改了代码或者某个中间层的 `PUBLIC` 被改成了 `PRIVATE`。

## 16.3 运行期错误（5 条）

### ⑧ ★ `由于找不到 Qt6Widgets.dll，无法继续执行代码。重新安装程序可能会解决此问题。`

**原因**：Qt DLL 不在系统的 DLL 搜索路径上。
**解决**：见第 15.2 节的三条道路（`PATH` / `windeployqt` / 永久环境变量）。

> **Windows DLL 搜索顺序复习**：① exe 所在目录 → ② 当前工作目录 → ③ System32 → ④ PATH 里的各个目录。**所以把 DLL 拷到 exe 旁边（windeployqt 干的事）是最稳的。**

### ⑨ ★ `This application failed to start because no Qt platform plugin could be initialized.`

**原因**：找不到 Qt 的**平台插件** `platforms/qwindows.dll`。这个插件是 Qt 在 Windows 上实现"画出窗口"的那一层。
**解决**：

```bat
:: 方法一：windeployqt 会自动拷
windeployqt LightLiveStreamer.exe
:: 之后 exe 旁边会出现 platforms\qwindows.dll

:: 方法二：手动指定插件搜索路径
set QT_QPA_PLATFORM_PLUGIN_PATH=C:\Qt\6.5.0\msvc2019_64\plugins\platforms
LightLiveStreamer.exe

:: 方法三：确认环境变量 QT_PLUGIN_PATH 指到了 Qt 的 plugins 目录
```

### ⑩ ★ `ASSERT failure in QCoreApplication::sendEvent: "Cannot send events to objects owned by a different thread"` —— 然后崩溃

**原因**：**你在非 GUI 线程碰了 QWidget。** 百分之百是这条。
**排查**：崩溃时看调用栈，最上面那个"你自己写的函数"是哪个？它是在哪个线程被调的？

**三处最常见的肇事点**：

1. `postLog` 里直接调了 `appendLog`/`logView_->appendPlainText`（第 15.5 节实验二）；
2. `startStreaming` 的 worker lambda 里直接操作了 UI（第 12.11 节反例 2）；
3. 某个 `std::thread` 里直接调用了 `pipeline_.start(config, cb)`，**而 cb 里直接 `appendLog`**（没走 postLog）。

**修法**：对照第 3.9 节的"纪律六条"，该投递的投递（`QMetaObject::invokeMethod(..., Qt::QueuedConnection)`）。

另外推荐加一个"哨兵断言"，能让你在**开发阶段**就抓到这类错误：

```cpp
// 在 appendLog 的第一行加：
Q_ASSERT_X(QThread::currentThread() == qApp->thread(),
           "MainWindow::appendLog",
           "appendLog 只能在 GUI 线程调用！");
```

需要 `#include <QThread>` 和 `#include <QApplication>`。**Debug 版本会立刻中断、Release 版本零开销。** 这是把"口头约定"变成"机器可验证"的好办法。

### ⑪ 启动后立刻崩溃，栈指向 "pure virtual method called" 或 "Access violation reading 0x0"

**原因**：成员指针为空就被用了。最典型的情况是**某个函数在 `buildUi()` 之前被调用**（第 8.2 节讲过构造顺序）。

**排查**：断在崩溃处，看那个空指针是哪个成员，然后看调用栈 —— 它是不是从构造函数很早的位置来的？

### ⑫ 点"开始推流"之后一切正常，**但界面卡住不动了**

**原因**：你把耗时操作放到 GUI 线程了。检查：

- `pipeline_.start()` 是不是**没有**包在 `std::thread` 里？（第 12.11 节反例 1）
- 是不是在 worker lambda 里调用了某个"偷偷做了阻塞 I/O"的东西？

**验证方法**：卡住的时候用 VS 的"全部中断"（Break All），看 GUI 线程（主线程）的调用栈停在哪。**如果停在 `avio_open` / `connect` / `join` 这类函数上，就是它了。**

## 16.4 行为层错误（2 条）

### ⑬ 日志框一直不更新（或者要过很久才更新）

**原因排排查顺序**：

1. **GUI 线程被阻塞了**（如果有 stop/start 在跑，回到第 ⑫ 条）；
2. **投递出去的事件没人处理** —— GUI 线程的事件循环是不是卡在别的地方了？（比如某个槽里写了死循环）；
3. **`postLog` 被改成了直接写控件**（实验二），此时 Qt 可能不报错但悄悄出错（界面表现随机，回到第 ⑩ 条排查）。

### ⑭ 填了地址但还是"启动失败: xxx"

看看日志里的具体原因：

| 日志 | 原因 | 怎么办 |
|---|---|---|
| `视频编码器打开失败: ...` | FFmpeg 缺 libx264（第 1 课就提示过的 `vcpkg install ffmpeg[gpl,x264]`） | 重装带 x264 的 FFmpeg |
| `推流输出打开失败: ...` | RTMP 地址不对 / 服务器没起 / 密钥过期 | 先用 `file:///C:/temp/x.flv` 试本地文件排除 Newbie |
| `屏幕采集初始化失败` | DXGI/GDI 初始化失败（远程桌面会话、显卡驱动异常） | 换个源试试（摄像头） |
| `摄像头打开失败: 索引 N` | 摄像头被别的软件占用 / 索引不对 | 关闭占用的软件；点"刷新"重新枚举仔细选 |
| `音频采集打开失败` | 系统声音设备被独占 / 麦克风权限 | Windows 设置 → 隐私 → 麦克风；或切到"麦克风"试试 |
| `启动失败: 推流参数无效` | 宽高/帧率是 0 | 检查分辨率下拉框有没有选中项 |

**第一步建议**：把地址换成 `file:///C:/temp/test.flv`，如果这个能通，那就是网络/服务器的问题，跟界面无关。

---

# 第 17 章 小结、局限、思考题与下节预告

## 17.1 必记的 14 条知识点

| # | 知识点 | 一句话 |
|---|---|---|
| 1 | **GUI 线程专用性** | 只有创建界面的线程能碰 QWidget；别的线程要更新 UI，只能投递消息 |
| 2 | **主事件循环** | `app.exec()` 是一个"取事件→处理"的死循环，所有 slot 都在这里被调用 |
| 3 | **主线程不许阻塞** | 阻塞 = 界面未响应；可能耗时的操作一律开线程 |
| 4 | **对象树** | `new QXxx(parent)` 的孩子由父亲自动 delete；布局不等于 parent |
| 5 | **信号与槽** | 观察者模式；`connect(sender, sig, context, slot)` 里的 context 决定生命周期和线程亲和性 |
| 6 | **moc / AUTOMOC / Q_OBJECT** | `Q_OBJECT` 给 moc 的记号；把含它的 .h 显式写进 target 源文件列表 |
| 7 | **三种跨线程投递写法** | 自定义信号+Queued（正式）/ invokeMethod+字符串（淘汰）/ **invokeMethod+lambda（本项目）** |
| 8 | **`Qt::QueuedConnection` 必须显式写** | AutoConnection 在"碰巧同线程"时会走 Direct，行为不一致 |
| 9 | **异步 lambda 只能值捕获** | `[&]` 捕获栈变量 = 悬垂引用灾难；`[this]` 捕获要靠 join 保证对象活得够久 |
| 10 | **`Userdata` 优于`currentText` 解析** | 显示和数据分离，改文案不影响逻辑 |
| 11 | **`QPlainTextEdit` 做日志** | 纯文本、快、`setMaximumBlockCount` 兜住内存 |
| 12 | **布局 > 绝对坐标** | stretch 分配多余空间；`addStretch()` 是弹簧；`addSpacing()` 是死宽度 |
| 13 | **生命周期纪律** | 持有 `this` 的后台线程，析构前必须 join；三处 join（开始/停止/关闭）缺一不可 |
| 14 | **命名即文档** | `postXxx` = 线程安全入口，`xxxGui` = 只能在 GUI 线程调用 |

## 17.2 API 速查表

### 跨线程投递三件套

```cpp
// ① lambda + context（本项目用法，Qt ≥ 5.10）
QMetaObject::invokeMethod(this, [this, data] { doSomething(data); },
                          Qt::QueuedConnection);

// ② 信号 + QueuedConnection
signals: void ready(const QString& s);
connect(this, &MyClass::ready, this, &MyClass::handle, Qt::QueuedConnection);
emit ready(QString::fromStdString(s));      // 参数必须是 Qt 元类型

// ③ 注册自定义类型后跨线程传
qRegisterMetaType<std::string>("std::string");
```

### 五个连接类型

| 类型 | 行为 |
|---|---|
| `DirectConnection` | 立即、同步，在发射者线程执行 |
| `QueuedConnection` | 排队投递到接收者线程，**异步** |
| `AutoConnection` | 同线程走 Direct，跨线程走 Queued（默认） |
| `BlockingQueuedConnection` | 投递 + 阻塞等待返回；**同线程用会死锁** |
| `UniqueConnection` | 修饰符，防止重复连线 |

### 常用控件 API

```cpp
// QComboBox
addItem(text, QVariant userData); clear(); count();
currentIndex(); setCurrentIndex(i); currentText(); currentData(); findText(s); findData(v);
signals: currentIndexChanged(int) / currentTextChanged(QString) / activated(int)

// QSpinBox
setRange(lo, hi); setSingleStep(n); setValue(v); setSuffix(" fps"); value();

// QPlainTextEdit
setPlaceholderText(); toPlainText(); appendPlainText(); setReadOnly();
setMaximumBlockCount(n); setMaximumHeight(n);

// QPushButton / QLabel
setEnabled(bool); setFixedWidth(n); clicked(bool);
setText(); font() → setBold(true) → setFont()

// QTimer / QElapsedTimer
new QTimer(this); setInterval(ms); start(); stop(); timeout
start(); elapsed();     // 毫秒

// QString 格式化
QString("%1 / %2").arg(a).arg(b);
QString("%1").arg(num, 2, 10, QLatin1Char('0'));      // 补零到 2 位
```

## 17.3 本次实现留存的 9 条局限

诚实一点，这次的代码不是尽善尽美的。**下面这些"不足"是下一阶段改进的方向，也是你们的加分题。**

| # | 局限 | 位置 | 影响 |
|---|---|---|---|
| 1 | **没加 `WIN32_EXECUTABLE`** | `src/app/CMakeLists.txt` | 双击 exe 会带一个黑控制台 |
| 2 | **URL 解析会被第一个 `=` 切断** | 第 257 行 | 带查询参数的 RTMP 地址会解析错误 |
| 3 | **下标魔法数字**（`== 1` 出现两处） | 第 220、230、241 行 | 界面顺序调整时容易漏改 |
| 4 | **`starting_` + `running_` 两个 bool** | 第 80~81 行 | 能表达 4 种组合，其中 1 种非法 |
| 5 | **停止时没有"停止中"状态** | 第 339 行 | 点停止后立刻显示"未开始"，但后台还在收尾 |
| 6 | **`stopStreaming` 里的 join 会卡 GUI** | 第 336 行 | 如果在连接中点停止，界面会卡住直到连接超时 |
| 7 | **日志没有攒批** | 第 417 行 | 高频日志下 GUI 线程会疲于奔命 |
| 8 | **`<QFont>` 没显式 include** | 第 123 行用了 | 隐式依赖，换 Qt 版本可能断 |
| 9 | **`refreshAudioDevices` 不保持上次选择** | 第 208 行 | 刷新后用户选择丢失，和 `refreshVideoDevices` 行为不一致 |

> **作业建议**：挑其中 3 条改掉，每改一条重跑一遍第 15.4 节的验收清单。你会发现"看着不起眼的小问题"改起来也要小心翼翼 —— 这就是维护性打磨的日常。

## 17.4 思考题（8 题，先做完再看要点）

**Q1.** `startStreaming()` 明明可以让"开始推流"的按钮点下去之后协作完成，为什么非要开一条 `std::thread`？如果直接在槽里调用 `pipeline_.start()`，用户在几秒的过程中会看到什么？请把事件循环（重画、"正在连接"标签的绘制）的排队关系讲清楚。

**Q2.** 如果把第 323 行的 `Qt::QueuedConnection` 改成 `Qt::BlockingQueuedConnection`，会发生什么？如果把第 151 行假设存在的某个 `Qt::QueuedConnection` 改成 `Qt::DirectConnection` 呢？

**Q3.** `refreshVideoDevices` 里用 `findText(previous)` 恢复用户的选择，而 audio 版平时用 `currentData().toInt()` 存的是设备索引。如果要给 `refreshAudioDevices` 也加上"恢复选择"，为什么**用 `findData(索引)` 比 `findText(设备名)` 更好**？（提示：设备名可能重复。）

**Q4.** 为什么 `starting_` / `running_` 用 `std::atomic<bool>`？如果换成普通 `bool`，编译器可能做出什么优化导致 bug？请举一个具体的"看起来不可能出错但会出错"的场景。

**Q5.** `stopStreaming()` 第 331 行用的是 `running_.exchange(false)`，而不是"先 load 判断、再 store false"。请说明这两者在多线程下的差别，并说明本代码里 `exchange` 还顺带完成了什么额外的工作。

**Q6.** `updateSourceFields()` 把摄像头那一行隐藏了，但 `cameraCombo_` 的当前值还在。请找出代码里防止"屏幕模式下误用摄像头索引"的那一行保护。如果你要给 collectors 多列四项之一，怎么改写才能让它变成"不需要保护也安全"的设计？

**Q7.** `pipelineMutex_` 保护哪些操作？如果把 `pollStats()` 里的锁去掉（第 387 行），会出现什么问题？反过来，如果给整个 `pollStats()` 都加上锁（包括四个 `setText`），又有什么坏处？

**Q8（综合题）.** 假设日志洪水：某小时的推流过程中一毫秒产生一条日志（每秒 1000 条）。请分析 (a) 现在的 `postLog` 会怎样？(b) 界面为什么越来越卡？(c) 设计一种"攒批"方案，说清楚它要在哪个线程攒、什么时候投递、投递的数据结构是什么、以及关窗口时如何保证不丢最后一批。

### 参考答案要点（请先自己答完再看）

1. 直接在槽里调，`start()` 阻塞 2.5 秒期间 GUI 线程被占住，事件队列里的 PaintEvent 排不上，"正在连接…"这几个字**根本画不出来**（第 12.11 节反例 1）。用户看到"点了没反应"，会再点一次（这也是防重入为什么重要）。
2. `BlockingQueuedConnection`：worker 线程会阻塞等待 GUI 线程执行完 lambda；本项目 GUI 没被占住所以不会死锁，但它让 worker 白白等待，**并且如果将来有人在持有 `pipelineMutex_` 的情况下用 Blocking 投递，就会和 GUI 侧的 `pollStats` 构成死锁**（第 12.8 节"为什么锁要在 invokeMethod 之前释放"的原因）。改成 Direct 则 lambda 会在 worker 线程同步执行 —— 整段 UI 更新代码在错误的线程跑 → 第 3.1 节的事故清单。
3. `findData(索引)` 更好：设备名可能重复（比如两个"立体声混音"、或者同名设备），按文本找会找错；而 userData 存的是枚举时的真实下标，唯一且和内核 `open(type, index)` 的语义对齐。
4. 编译器可能把"读 `running_`"优化成"用寄存器里缓存的旧值"，因为它可以假定没人从别的线程改这个变量（数据竞争是 UB）。具体场景：worker 线程写 `running_ = true`，GUI 线程在另一个函数里读，但读到的是编译期优化后的缓存值 → 界面永远显示"未开始"。另外 `__exchange` 的缓存一致性保证也不同。**这是" UB "，不是"偶尔读到旧值"。**
5. `exchange` 是**一条不可打断的读改写指令**：先把 false 写进去，再把旧值取出来返回。load+store 之间有窗口：worker 线程可能在中间把 `running_` 写成 true，你的 store 就会把这个新值覆盖成 false（TOCTOU）。额外功能：无论原值是什么，执行后 `running_` 一定是 false，所以第 339 行的 `updateUiState()` 天然看到"没在跑"。
6. 保护在第 232 行：`if (config.sourceType == SourceType::Camera)`。**改成不需要保护的设计**：把"选了哪种源"这件事的所有信息塞进 userData（第 9.2 节方案 1），或者干脆在 `updateSourceFields` 里把 `cameraCombo_->setCurrentIndex(-1)` 清空，让"隐藏 = 未选中"这两个事实永远一致（消除非法状态的存在空间）。
7. 保护四处：`start`（worker）、`stop`（worker 或 GUI）、`pollStats`（GUI）、以及 `closeEvent`（GUI）。去掉 `pollStats` 的锁：GUI 线程读 atomic 的同时 worker 在 `stop()` 里 join 线程/清队列，`stats()` 内部读的 `output.targets()` 是个 vector，**可能被并发修改** → 崩溃或读到野数据。整段加锁的坏处：`setText` 会触发布局计算和重画事件，GUI 线程长期持锁，worker 想 start/stop 就得干等，**扩大临界区 = 人为制造串行化**（第 14.1 节）。
8. (a) 每秒 1000 个 `QMetaCallEvent` 进 GUI 队列；(b) GUI 线程每秒执行 1000 次 `appendPlainText`，每次触发 `QPlainTextEdit` 内部文档重排和可能的重画 → 用户交互延迟、掉帧；而且 `setMaximumBlockCount(2000)` 会以 2000 行为上限疯狂删除旧内容，进一步加剧重排；
   (c) 攒批方案：在工作侧维护一个 `QStringList buffer_`（加互斥锁或只在 número 单生产者的前提下使用 `appendPlainText` 组合），每凑够 N 条或每 200ms 用 `QTimer::singleShot`/一次 `invokeMethod` 投递整批，GUI 侧在一个 lambda 里一次性 `appendPlainText(buffer.join("\n"))`；关窗口时在 `closeEvent` 里强制 flush 剩余的一批（并且因为 join 在前，这批数据一定能被投递出去）。

## 17.5 下节课预告

第 13 课我们要讲的是 **`bafb4ea`「实时预览」**。

现在的界面有一个很明显的缺失：**用户看不到自己在推什么画面。** 只能靠日志里增长的数字去猜"应该是在正常推"。这是不可接受的 —— 直播软件没有预览窗口，就像摄影师没有取景器。

下一次我们要解决的问题是：**管线里每一帧画面都在一条worker 线程（videoThread）上流动，我们怎么把它画到 GUI 线程的窗口里？**

先给三个预习思考：

1. 一帧 1280×720 的 BGRA 是 3.7 MB，30fps 就是每秒 110 MB。**预览要拷贝这些数据吗？拷贝几次？**
2. 预览必须限帧（比如最多 15fps），**多余的帧应该丢还是排队？**（回忆第 3 课 `BoundedQueue` 的"丢旧保新"。）
3. 把 `VideoFrame` 的数据送到 GUI 线程去画，我们今天学的三种投递写法够不够？**一帧 3.7 MB 塞进事件队列合理吗？**（提示：想想 `QImage` 的隐式共享和 `shared_ptr`。）

带着这三个问题去预习，下一课会很有意思。

---

**本讲配套文件**：`src/app/MainWindow.h`（86 行）、`src/app/MainWindow.cpp`（437 行）、`src/app/main.cpp`（17 行）、`src/app/CMakeLists.txt`（6 行）
**前置讲稿**：第 3 课（`StreamConfig` 契约）、第 4/5/7 课（采集器的 `listDevices`）、第 8/10 课（编码器与多目标输出）、**第 11 课（`StreamPipeline`，必读）**
**配套素材**：`main.cpp`（17 行启动器）、`build/src/app/LightLiveStreamer_autogen/moc_MainWindow.cpp`（moc 的产出，值得看一眼）

---

## 附：本讲「五章对照索引」

> **用法说明**：这张表把本讲的全部内容压缩成**五个维度**的对照，方便复习、检索，以及跟前面几课建立联系。横向五行是本讲的**五个部分**，纵向五列是**五个维度**。

| 本讲部分 | 覆盖章节 | 核心知识点 | diff 中的文件与行号 | 怎么验证 / 上哪去练 |
|---|---|---|---|---|
| **① 预习：概念与原理** | 第 1~4 章 | 主事件循环与"主线程不许阻塞"；对象树与 parent 自动回收；`signals/slots/emit` 都是宏、moc 生成元数据、`Q_OBJECT` 与 AUTOMOC、`MainWindow.h` 要写进 target；**线程亲和性**、跨线程碰 QWidget 的四条深层原因（原子性/句柄归属/全局状态/死锁）；五种连接类型与 AutoConnection 陷阱；三种投递写法比较、元类型注册与 lambda 捕获绕坑；布局三大件、stretch/`addStretch`/`addSpacing`、`QGroupBox`、`QComboBox` 的 userData、`QPlainTextEdit` vs `QTextEdit`、`QTimer` 的线程归属 | 概念章，对应实现见 `src/app/main.cpp` 第 7 行（`QApplication`）、第 16 行（`app.exec()`）、`MainWindow.cpp` 第 295 行（`std::thread`）、第 308~323 行（`invokeMethod`）、第 417~422 行（`postLog`） | **实验二**（故意破坏跨线程纪律看崩溃）、§15.3 界面图对照、§16.3 第 ⑩ 条诊断清单 |
| **② 实现：模块本体** | 第 5~14 章 | `src/app/CMakeLists.txt` 六行的四个门道（`.h` 喂 AUTOMOC、`PUBLIC` 传播 include、`core_common→core_pipeline`、缺 `WIN32_EXECUTABLE`）；`main.cpp` 从 132 行脚手架砍到 17 行；`MainWindow.h` 三段 include + 八个前向声明 + 五个职责分组的私有方法 + 分区成员 + 状态成员；`buildUi()` 造控件/摆布局/连信号三步走；设备枚举三兄弟；`collectConfig()` 的解析器；`startStreaming` 的状态检查→收集→清理→反馈→开线程→投递；`updateUiState()` 统一刷新；`pollStats/appendLog/postLog/closeEvent` | `src/app/CMakeLists.txt`（6 行，全）、`src/app/main.cpp`（17 行，全）、`src/app/MainWindow.h`（86 行，全）、`src/app/MainWindow.cpp`（437 行，全：第 23~39 行构造/析构、第 41~179 行 `buildUi`、第 181~224 行三兄弟、第 226~275 行 `collectConfig`、第 277~325 行 `startStreaming`、第 327~350 行 `stopStreaming`、第 352~381 行 `updateUiState`、第 383~409 行 `pollStats`、第 411~422 行两个日志函数、第 424~437 行 `closeEvent`） | 编译 `LightLiveStreamer` exe；确认 `moc_MainWindow.cpp` 生成（§5.5）；按 §15.4 的 17 条清单逐条验收 |
| **③ 验证：动手与排错** | 第 15~16 章 | 三种运行方式（`PATH`/`windeployqt`/VS）；地址框可用 `file:///…flv` 免服务器验证；17 条操作验收清单（其中第 10/13/16 条直接验证线程化、跨线程投递、生命周期）；7 个改造实验；14 条排错表（编译 4 / 链接 3 / 运行 5 / 行为 2） | 不新增代码，全部是对 ② 部分代码的**运行、观察与故意破坏** | `build.bat` → 双击 exe；**实验二务必亲手做一次**；用 §16 的表按图索骥 |
| **④ 沉淀：总结与延伸** | 第 17 章 | 14 条必记知识点；四张 API 速查表（投递三件套/连接类型/常用控件/QString 格式化）；9 条局限（含 WIN32_EXECUTABLE、URL 等号坑、下标魔法数、双 bool 状态、无"停止中"状态、join 阻塞 GUI、日志无攒批、`QFont` 隐式依赖、音频刷新丢选择）；8 道思考题及答案要点 | 对全讲 536 行 diff 的回顾与批判 | 做完 8 道思考题（尤其 Q4/Q5/Q8）；改掉 9 条局限里的 3 条并复验清单 |
| **⑤ 衔接：与前几课的映射** | 全讲各处 | `StreamConfig`/`SourceType`/`AudioSourceType`/`StreamTarget`/`LogCallback` ← 第 3 课；`CameraCapturer::listDevices`/`AudioCapturer::listDevices` ← 第 5/7 课；`StreamPipeline`/`PipelineStats` ← 第 11 课；多目标（N 路地址输入框）← 第 10 课；PImpl 让 `include "core/pipeline/StreamPipeline.h"` 保持轻量 ← 第 8/10 课；"丢旧保新"的队列哲学在 Qt 事件队列上的类比 ← 第 3 课 | 见上面各行 | 预习第 13 课 `bafb4ea`（实时预览）：想想 canvas 怎么拿到 worker 线程的帧数据 |

### 附：本讲与前面几课的衔接点（复习时对照）

| 本讲用到的东西 | 在哪一课学的 | 本讲怎么用的 |
|---|---|---|
| `StreamConfig` / `SourceType` / `AudioSourceType` / `StreamTarget` | **第 3 课**（`a4453bb1`） | §11 `collectConfig()` 的全部产出；界面是这份契约的"填充者" |
| `LogCallback`（`std::function<void(const std::string&)>`） | **第 3 课** | §12.8 的 lambda，成为内核和界面之间唯一的"回话通道" |
| `BoundedQueue` 的"生产者不动手、放窗口就走"的哲学 | **第 3 课** | §3.4 类比：Qt 事件队列就是 GUI 版的 BoundedQueue，消费者固定是 GUI 线程 |
| `CameraCapturer::listDevices()` | **第 5 课**（`281a22b`） | §10.1 `refreshVideoDevices()` 的数据源 |
| `AudioCapturer::listDevices(AudioSourceType)` 与 `-1` 表示系统默认 | **第 7 课**（`1ce7f62`） | §10.2 `refreshAudioDevices()`，并把 `-1` 做成"（系统默认）"这一项 |
| `PImpl` 惯用法 | 第 8/10 课用过 | §7.2：`StreamPipeline.h` 因为用了 PImpl，include 它不会把 FFmpeg/Windows 头拖进 app 层 |
| 多目标分发（`std::vector<StreamTarget>`） | **第 10 课**（`0ccd108`） | §9.4 多行地址输入框 + §11.7 解析器；"连接 N/M"的统计展示 |
| `StreamPipeline::start/stop/stats/lastError` 及其阻塞语义 | **第 11 课**（`31c3f57`） | §12 全部的决策依据（为什么必须开线程）；§14.1 的快照读取 |

### 附：本讲九个「可以立刻动手的改进练习」

| # | 位置 | 现状 | 改成 |
|---|---|---|---|
| 1 | `src/app/CMakeLists.txt` | 没加 `WIN32_EXECUTABLE` | 加 `set_target_properties(... PROPERTIES WIN32_EXECUTABLE TRUE)` |
| 2 | `MainWindow.cpp` 第 257 行 | `indexOf('=')` 会被 URL 里的等号骗 | 改用 `QRegularExpression("^([A-Za-z0-9_\\-]+)\\s*=\\s*(.+)$")` |
| 3 | `MainWindow.cpp` 第 220/230/241 行 | 下标魔法数字 `== 1` / `== 0` | combo 的 userData 里存枚举值，用 `currentData().toInt()` 判断 |
| 4 | `MainWindow.h` 第 80~81 行 | 两个 `atomic<bool>` 能表达非法组合 | `std::atomic<State>`（`Idle/Starting/Running/Stopping`） |
| 5 | `MainWindow.cpp` 第 339 行 | 点停止立刻显示"未开始" | 加"停止中"状态（配合第 4 项一起改最自然） |
| 6 | `MainWindow.cpp` 第 417 行 | 每条日志一次投递 | 攒批（N 条或 200ms 投递一次），并在 `closeEvent` 里 flush |
| 7 | `MainWindow.cpp` 第 6~19 行 | 用了 `QFont` 没 include | 加 `#include <QFont>`（顺便自查 `QVariant`/`QLatin1Char`） |
| 8 | `MainWindow.cpp` 第 202~216 行 | 刷新不恢复用户选择、且 `clear()` 可能触发信号 | 用 `QSignalBlocker` + `findData(previousIndex)` |
| 9 | `MainWindow.cpp` 第 41~179 行 | `buildUi()` 139 行偏长 | 拆成 `createVideoGroup()` / `createAudioGroup()` / `createTargetGroup()` / `createControlBar()` / `createLogArea()` |

**改完任意一项之后，重新跑一遍第 15.4 节的 17 条验收清单，确认没有回归。**

**这就是重构的正确姿势：先有验收标准，再动代码。**

---

> **老师最后的话**：同学们，这一课我们没有写一行新的采集、编码、网络代码。**我们做的，是把前十一课打出来的东西，交到了用户的手上。**
>
> 请记住今天最硬的那条规矩 —— **子线程不许直接碰界面，有事排队发消息。** 它不是 Qt 作者的洁癖，它是整个"单线程 + 事件驱动"GUI 模型的地基。理解它，你写 Qt、写 Win32、写 Android Handler、写浏览器 Worker 都能融会贯通 —— 因为它们背后是同一件事：**用消息代替共享内存。**
>
> 下一课我们要给这个遥控器装上屏幕：让用户看见自己在推什么画面。**那时候你会发现，今天学的这套投递机制，还是不够用的** —— 因为一帧 3.7 MB 的画面塞进事件队列，跟一条 100 字节的日志，完全不是一回事。
>
> 好，第 12 课到此结束。






