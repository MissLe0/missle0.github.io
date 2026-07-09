# 一、代码编译过程（VisualStudio）
## 编译流程总览
+ 在VS中，`C/C++`代码编译经历了4个核心阶段（`预处理→编译→汇编→链接`）

```plain
源代码(.cpp/.c) 
   ↓ [预处理]
预处理文件(.i)
   ↓ [编译]
汇编代码(.asm)
   ↓ [汇编]
目标文件(.obj)
   ↓ [链接]
可执行文件(.exe/.dll)
```

+ VS编译`C/C++`代码的本质是调用MSVC工具链完成以上四步，最终将`.cpp/.c`源文件转换为可执行文件（`.exe`）、动态库（`.dll`）或静态库（`.lib`）

| 阶段 | 核心工具 | 输入文件 | 输出文件 | 核心作用 |
| --- | --- | --- | --- | --- |
| 预处理 | `cl.exe`（预处理模块） | `.c/.cpp` | `.i`：（C）<br/>`.ii`：（C++） | 处理`#`：开头的预处理指令，文本替换，无语法检查 |
| 编译 | `cl.exe`（编译模块） | `.i/.ii` | `.asm` | 语法 / 语义分析，生成汇编代码（Debug/Release 优化差异在此阶段体现） |
| 汇编 | `ml.exe`（MASM 汇编器） | `.asm` | `.obj`<br/>（目标文件） | 汇编代码转机器码，生成包含符号表的二进制目标文件 |
| 链接 | `link.exe` | `.obj`/`.lib` | `.exe/.dll/.lib` | 解析符号引用，合并多个目标文件 / 库，生成最终可执行文件 / 库 |


+ 注：VS 默认不会保留`.i/.ii/.asm`中间文件，需手动指定参数生成；日常编译中`cl.exe`会合并「预处理 + 编译 + 汇编」三步，直接输出`.obj`。

## 预处理（Preprocessing）
+ 执行工具：`cl.exe`配合`/E`或`/P`参数
+ 主要任务：
    - 宏替换：`#define`定义的宏会被展开
    - 文件包含：`#include`头文件内容会被插入
    - 条件编译：`#ifdef`、`#ifndef`等指令处理
    - 删除注释：所有注释被移除
    - 添加行标记：`#line`标记用于调试
+ VS中的体现：

```bash
cl /P main.cpp  # 生成main.i预处理文件
```

+ 示例：

```cpp
// main.cpp
#define PI 3.14159
#include <iostream>

int main() {
    std::cout << PI;  // 预处理后变为 std::cout << 3.14159;
    return 0;
}
```

## 编译（Compilation）
+ 执行工具：`cl.exe`（C/C++编译器前端）
+ 主要任务：
    - 词法分析：源代码拆分为token
    - 语法分析：构建抽象语法树（AST）
    - 语义分析：类型检查、作用域解析
    - 优化：执行代码优化（O1, O2, O3, Ox等级别）
    - 代码生成：生成汇编代码（`.asm`）
+ 生成汇编代码：

```bash
cl /FA main.cpp  # 生成main.asm汇编文件
```

## 汇编（Assembly）
+ 执行工具：`ml.exe`（MASM）或`armasm.exe`（ARM）
+ 主要任务：
    - 将汇编代码（`.asm`）转换为机器码
    - 生成目标文件（`.obj`）
    - 包含重定位信息和符号表
+ 目标文件结构：

```plain
.text段：可执行代码
.data段：已初始化全局/静态变量
.bss段：未初始化变量
.rdata段：只读数据
符号表：函数和变量名称
```

## 链接（Linking）
+ 执行工具：`link.exe`（链接器）
+ 主要任务：将多个 `.obj` 文件、库文件（`.lib`）合并，解析符号引用，生成最终的可执行文件 / 动态库（PE 格式）
    - 符号解析
        * 将目标文件中的符号（函数/变量）引用与定义关联
        * 处理外部符号（如`printf`来自C标准库）
    - 重定位
        * 为代码和数据分配最终内存地址
        * 修正指令中的地址引用
    - 合并段
        * 将多个`.obj`文件的相同段合并
        * 生成最终PE格式文件
    - 库链接
        * 静态链接（`.lib`）：代码被复制到`.exe`
        * 动态链接（`.dll`）：运行时加载

## 多个源文件编译（一个项目）
+ 独立编译：每个源文件并行调用`cl.exe`生成独立的`.obj`目标文件（编译单元）

```bash
cl /c a.cpp b.cpp c.cpp  → a.obj, b.obj, c.obj
```

+ 统一链接：`link.exe`将所有`.obj`合并，解析跨文件的符号引用（函数/变量），生成最终`.exe`

```bash
link a.obj b.obj c.obj /OUT:app.exe
```

+ 本质：编译阶段各自为政，链接阶段全局整合。

# 二、什么是PE文件
+ PE文件就是在Windows电脑上看到的绝大多数可执行程序（比如`.exe`文件）和动态链接库（比如`.dll`文件），它不仅是`.exe`、`.dll`的格式，`.sys`（设备驱动）、`.ocx`（控件）甚至 `.scr`（屏幕保护）等文件也是。其核心设计目标是提供一种统一的结构，让Windows加载器（Loader）能够知道如何将磁盘上的二进制代码正确地映射到内存中并执行
+ 它的全称是Portable Executable（可移植可执行文件）
    - Executable（可执行）： 意思是电脑可以运行它
    - Portable（可移植）： 意思是这种格式在不同的Windows版本（如Win 7, Win 10, Win 11）甚至不同的硬件架构（只要都是Windows系统）上，结构基本是一样的，系统都能认识并运行它
+ 它的作用是告诉Windows加载器（Windows在运行程序前必须知道）
    - 程序从哪里开始执行
    - 代码放在哪、数据放在哪
    - 需要哪些 DLL、哪些函数
    - 内存怎么分配、基址、对齐、权限
    - ...
+ 所以：PE文件 = Windows运行程序的说明书

# 三、PE文件结构
## 基础概念
+ 把PE文件想象成一本印刷好的实体书，而Windows 操作系统（加载器）就是图书管理员
+ 文件偏移地址（FOA - File Offset Address）
    - 通俗比喻：实体书的物理页码，比如某段话印在第50页
    - 专业解释：文件静静地躺在硬盘上时，某个数据距离文件开头（0字节处）的相对距离
+ 虚拟内存地址（VA - Virtual Address）
    - 通俗比喻：图书管理员把这本书放到了图书馆的几楼几号书架上
    - 专业解释：双击运行程序后，PE文件被装载进内存，数据在内存中的绝对地址
+ 相对虚拟地址（RVA - Relative Virtual Address）
    - 通俗比喻：不管这本书被放在哪个书架上，某段话永远在距离本书第一页第50 页的位置
    - 专业解释：数据在内存中距离程序加载基址（ImageBase）的偏移量，公式：`VA = ImageBase + RVA`

## 基本结构示意图
<!-- 这是一张图片，ocr 内容为： -->
![](https://cdn.nlark.com/yuque/0/2026/png/43151352/1773476281850-1f7411c0-0e2a-4213-96c4-c30fbbeaaa5a.png)

```plain
PE文件结构
├── 1. DOS 头 (IMAGE_DOS_HEADER)--历史遗留的保护套
├── 2. DOS 存根 (DOS Stub)--一句废话
├── 3. NT 头 (IMAGE_NT_HEADERS)--核心：真正的PE档案袋
│   ├── 3.1 签名 (Signature)
│   ├── 3.2 文件头 (IMAGE_FILE_HEADER)
│   └── 3.3 可选头 (IMAGE_OPTIONAL_HEADER)
├── 4. 节表 (IMAGE_SECTION_HEADER)--目录
└── 5. 节区实体 (Sections)--真正的正文数据（代码、数据、资源等）
    ├── 5.1 .text (代码段)
    ├── 5.2 .data (数据段)
    ├── 5.3 .rdata (只读数据段)
    └── 5.4 .rsrc (资源段)
```

## 分段解析
### DOS头（IMAGE_DOS_HEADER）
+ 如果用现代的蓝光光盘机去读80年代的老式黑胶唱片，机器会出问题。DOS头就像在光盘外面套了一个兼容套，告诉老式的DOS系统这是一个新时代的Windows程序，你运行不了的。
+ DOS头的以下两个字段是核心的：
    - `e_magic`（2字节）：魔数，它的值永远是`0x5A4D`（ASCII码为MZ，这是MS-DOS早期核心程序员Mark Zbikowski的名字缩写），操作系统看到 `MZ` 开头，才承认它是个可执行文件

<!-- 这是一张图片，ocr 内容为： -->
![](https://cdn.nlark.com/yuque/0/2026/png/43151352/1773477153184-2b00cb42-bb2a-4067-9dde-3d6609a8fa70.png)

    - `e_lfanew`（4字节）：这是位于DOS头末尾的一个4字节的指针，可以把它当成写在书本第一页底部的精准导航坐标。在底层结构中，由于中间那段被废弃的历史遗留代码（DOS存根）长短不一，操作系统无法预知现代程序真正的核心区域到底从哪开始，因此`e_lfanew`里专门记录了一个文件偏移量（Offset），就像是告诉系统“不管中间夹杂了多少历史垃圾，真正的PE核心档案袋（NT 头），在距离文件最开头（起跑线）正好X个字节的地方”，也就是真正的PE头在相对文件开头偏移XXX的位置。操作系统读到这个具体的数值后，就能直接无视中间的废弃数据，像输入了GPS坐标一样，可以精准跨越过去并开始解析真正的程序

<!-- 这是一张图片，ocr 内容为： -->
![](https://cdn.nlark.com/yuque/0/2026/png/43151352/1773477310157-47ae4a3c-63f8-4fae-8003-8c1f6a1637e6.png)

### DOS存根（DOS Stub）
+ 就像印在现代蓝光光盘包装盒底部的一句警告语：“警告：请勿将此光盘放入老式录像机中播放”，对于现代Windows来说这句话毫无意义，但如果真有人把它塞进老式DOS系统），就会弹出一个警告
+ 它紧挨着DOS头之后，在NT头之前。这部分的大小并不固定，但通常很小。其实DOS存根本身就是一个完整的、小型的MS-DOS小程序
+ 主要作用：如果在极度古老的纯DOS系统中双击运行这个现代EXE文件，老系统看不懂后面的NT头，它只会执行这段存根代码，执行后，会在黑底白字的屏幕上打印出一句话：`This program cannot be run in DOS mode.`（此程序不能在DOS模式下运行），然后安全退出，防止系统崩溃

<!-- 这是一张图片，ocr 内容为： -->
![](https://cdn.nlark.com/yuque/0/2026/png/43151352/1773478062790-bc2364a6-19f3-4093-b2b9-e6f551589d57.png)

+ 对于现代 Windows 系统来说，这段区域基本没用

### NT头（IMAGE_NT_HEADERS）
+ 顺着`e_lfanew`指引的地址，来到了真正的核心，这里是程序的总档案袋，分为三个部分：

#### 签名（Signature）
+ 占据4个字节，永远是`50 45 00 00`，对应的ASCII码就是PE..

<!-- 这是一张图片，ocr 内容为： -->
![](https://cdn.nlark.com/yuque/0/2026/png/43151352/1773478499099-41b11b5c-a26c-4325-8a8d-6dc1d5928241.png)

+ 如果说MZ是第一道安检，这里就是第二道安检，确认过眼神，是真正的PE文件

#### 文件头（IMAGE_FILE_HEADER）
+ 通俗来说就是记录这本书的基本物理属性：什么语言写的？有几个大章节？什么时候出版的？
+ 也就说主要记录文件的基础信息，主要有：
    - `Machine`：运行平台，比如`0x8664` 代表x64架构

<!-- 这是一张图片，ocr 内容为： -->
![](https://cdn.nlark.com/yuque/0/2026/png/43151352/1773478865058-2b25d007-e9bd-4337-9f1d-ea5f3eb8fe76.png)

    - `NumberOfSections`：节区的数量，也就是告诉系统下面有几张“节表”，如果这里写了6，说明后面有6个章节（代码段、数据段等）

<!-- 这是一张图片，ocr 内容为： -->
![](https://cdn.nlark.com/yuque/0/2026/png/43151352/1773478898335-ccf8be59-9d2a-46cf-a870-6fc9f7bba3f3.png)

    - `TimeDateStamp`：编译时间。程序员点下生成 EXE那一刻的时间戳

<!-- 这是一张图片，ocr 内容为： -->
![](https://cdn.nlark.com/yuque/0/2026/png/43151352/1773478928934-dbad1519-e8cd-4771-9670-b8ad83f9d4b3.png)

    - `Characteristics`：文件属性，告诉你这是个EXE还是DLL，能不能执行

<!-- 这是一张图片，ocr 内容为： -->
![](https://cdn.nlark.com/yuque/0/2026/png/43151352/1773479210996-28af1a80-1382-4707-a2ff-23da3e0d1fbd.png)

#### 扩展头（IMAGE_OPTIONAL_HEADER）
+ 通俗来说就是告诉图书管理员，应该如何把这本书放到书架上，以及读者应该从哪一页开始读
+ 有以下重要的核心字段：
    - `Magic`：指代这个程序（如EXE）本身是被设计成什么规格的，而不是指用户当前用的电脑系统是什么规格，`0x010B`是32位，`0x020B`是64位

<!-- 这是一张图片，ocr 内容为： -->
![](https://cdn.nlark.com/yuque/0/2026/png/43151352/1773479504554-56d3a108-950d-405b-810a-bf3bbc3cadd5.png)

    - `AddressOfEntryPoint（OEP）`：程序入口点，这是RVA地址，是双击EXE后CPU执行的第一条指令的地址（不是文件偏移）
        * RVA（相对虚拟地址）就像是程序内部的相对坐标，它表示的是数据被装入内存后，距离程序起始点（ImageBase基址）的偏移距离
        * 由于程序在硬盘上时，根本不知道自己运行后会被操作系统分配到内存的哪一个绝对物理位置，所以它不能写死地址，只能记录一个相对位置
        * 通俗来说，OEP里存的RVA就是在告诉CPU：无论系统最终把我整个程序搬进内存的哪一层楼（基址），你要执行的第一条代码，永远位于距离我程序开头位置往后数第X个字节的地方
        * 底层核心公式：`真实内存绝对地址（VA）= ImageBase + RVA`

<!-- 这是一张图片，ocr 内容为： -->
![](https://cdn.nlark.com/yuque/0/2026/png/43151352/1773479666362-c1bdfdce-80fe-45a4-baa7-bdbf99983218.png)

    - `ImageBase（建议装载基址）`：可以理解为程序向操作系统申请的“内存首选门牌号”。程序在被编译好的那一刻，会假设未来一定会被装载到这个基址（例如0x400000），并以此为原点，提前把程序内部成千上万个函数和变量的绝对内存地址全部算死并固定在代码里。因此这个字段的核心作用就是告诉系统：“双击运行程序时，把整个程序从这个坐标开始塞进内存”。当用户双击运行时，如果这个内存空间是空闲的，系统就会直接把它放在这，程序直接启动运行；但如果这个空间被其他软件占用了，系统就会被迫给它安排一个新地址（重定位），并把程序内部写死的地址全部重新计算修改一遍，这会增加一点点启动开销

<!-- 这是一张图片，ocr 内容为： -->
![](https://cdn.nlark.com/yuque/0/2026/png/43151352/1773479705414-8ca04c38-fad8-4e6c-8434-7a8dfbc57f91.png)

    - `SectionAlignment（内存对齐大小）`：，通常是 `0x1000`，也就是说，每节在内存中必须占据4KB的倍数。这是程序被装进内存后的“空间排版规则”。因为现代CPU读写内存不是一个字节一个字节扣的，而是按“页”（通常是 4KB）一抓一大把。这就好比租写字楼必须“整层起租”，哪怕你的某段代码实际上只有10个字节，系统也绝不允许你只占10个字节的坑，而是强制在内存里给你分配4096字节（4KB）的整块空间，没用完的地方全部用`00`填满，这在底层叫做“用空间浪费去换取CPU极限读取速度”。

<!-- 这是一张图片，ocr 内容为： -->
![](https://cdn.nlark.com/yuque/0/2026/png/43151352/1773480406383-4b013e52-cc19-4959-b14d-e35b10735806.png)

    - `FileAlignment（文件对齐大小）`：通常是`0x200`，硬盘上数据按照512字节对齐。这是程序作为静态文件躺在硬盘上时的“物理打包规则”。因为机械硬盘的物理结构决定了它读写数据的最小扇区单位恰好是512字节。为了照顾硬盘的“脾气”，防止磁头跨扇区读取导致卡顿，编译器在生成EXE文件时，会强行把每一个章节（代码、数据）拼凑成512字节的整数倍。假设数据实际上只有300字节，文件内部也会硬塞进去212个毫无意义的空白`00`来凑数占位。这也就是为什么用16进制工具打开文件时，能看到数据和数据之间夹杂着一大片`00`缝隙的原因。

<!-- 这是一张图片，ocr 内容为： -->
![](https://cdn.nlark.com/yuque/0/2026/png/43151352/1773480561735-0f99283d-3cac-435d-ba36-e4ad74fa362e.png)

    - `SizeOfImage`：程序装入内存后占用的总大小

<!-- 这是一张图片，ocr 内容为： -->
![](https://cdn.nlark.com/yuque/0/2026/png/43151352/1773481053875-4dcd2885-a8b8-49f1-a863-b76c629baa08.png)

    - `Subsystem`：表明了该程序是GUI应用还是控制台应用等

<!-- 这是一张图片，ocr 内容为： -->
![](https://cdn.nlark.com/yuque/0/2026/png/43151352/1773485847815-baac5987-527e-45c5-bded-fee05c02c244.png)

### 节表（Section Table / IMAGE_SECTION_HEADER）
+ 通俗来说就是书的目录。目录里有一行行的条目，比如：
    - 第一章叫`.text`，讲的是代码，在硬盘第5页，搬到内存要放在第10页
    - 第二章叫`.data`，讲的是数据，在硬盘第8页，搬到内存放在第20页
+ 文件头里的`NumberOfSections`字段写了有几个节，这里就有几个结构体，每个结构体主要包含以下几个字段：
    - `Name`：8个字节的名字，如 `.text`（代码）、`.data`（数据）、`.rsrc`（资源）
    - `VirtualSize`：该节在内存中的真实大小（没对齐前的实际占用）
    - `VirtualAddress`：该节在内存中的RVA（相对虚拟地址）
    - `SizeOfRawData`：该节在硬盘文件中的大小（按FileAlignment对齐后的）
    - `PointerToRawData`：该节在硬盘文件中的确切物理位置（FOA）， 在内存映射是`ImageBase + VirtualAddress`，也就是VA
    - `Characteristics`：节的属性，决定了这个节在内存里是可读（R）、可写（W）、还是可执行（E）

<!-- 这是一张图片，ocr 内容为：FOH SECTIONHEADERS[6] 200H STRUCT IMAGE SECTION HEADER SECTIONHEADERS[0] 200H 28H  STRUCT IMAGE SECTION HEADER .TEXT 200H 8H BYTE NAME[8] .TEXT CAN END WITHOUT ZERO 208H 4H MISC UNION 208H PHYSICALADDRESS DWORD 17FCH 4H 208H 17FCH 4H VIRTUALSIZE DWORD 20CH 1000H 4H VIRTUALADDRESS DWORD 210H 4H 1800H DWORD SIZEOFRAWDATA 214H 4H 400H DWORD POINTERTORAWDATA 218H 4H DWORD POINTERTORELOCATIONS OH 21CH 4H 0 POINTERTOLINENUMBERS DWORD 220H 2H NUMBEROFRELOCATIONS WORD 222H 2H 0 NUMBEROFLINENUMBERS WORD 4H 224H  STRUCT SECTION CHARACTERISTICS CHARACTERISTICS -->
![](https://cdn.nlark.com/yuque/0/2026/png/43151352/1773491986667-86a16ea9-8ff7-4608-968b-a2471f25018e.png)

### 节区（Sections）
+ 通俗来说就是目录（节表）翻完后，找到的对应的正文
+ 节区中存放着程序运行所需的所有实际数据。常见的节有：
    - `.text`： 可执行代码段（代码通常放这） ，里面是CPU要执行的机器码（汇编指令）
    - `.data`：初始化的可写数据，比如定义的全局变量（`int score = 100;`，100就存放在这里）
    - `.rdata`：只读数据节，放一些不能修改的东西。 比如导出表、导入描述符表、字符串常量等
    - `.bss`：未初始化数据节。比如定义了 `int buffer[1000];`但没赋值，为了节省硬盘空间，它在PE文件里不占实际大小，只有等到程序加载到内存时，才会给它分配空间并清零
    - `.rsrc`：资源节，程序的图标、对话框、菜单栏，甚至内嵌的音乐等，都在这里
    - `.reloc`：基址重定位表（当加载基址不等于ImageBase时，用于修正内存地址）

<!-- 这是一张图片，ocr 内容为： -->
![](https://cdn.nlark.com/yuque/0/2026/png/43151352/1773484768971-0b0b379b-3d42-4fd8-b787-1a03f43d1b65.png)

# 四、VA, RVA, FOA
## PE文件的两种状态
+ PE文件的两种状态：PE文件在硬盘里和被加载到内存运行后，它的长相（结构间距）是不一样的
    - 状态一：在硬盘中（静态存储区域）：为了节省硬盘空间，PE文件的各个部分（节或段，如代码段`.text`、数据段`.data`）是相对紧凑地挨在一起的。这就好比把一顶帐篷打包塞进背包里，为了省空间，折叠得很紧密
    - 状态二：在内存中（动态运行区域）：当双击运行`.exe`文件时，操作系统会把它拉伸并映射到内存中。为了配合CPU和内存分页机制的高效管理，各个节之间的间距会被拉大。这就好比把背包里的帐篷拿出来，在营地上撑开，它占用的空间变大了。
+ 核心结论：正是因为“拉伸”的存在（`SectionAlignment (0x1000) > FileAlignment (0x200)`），导致了同一个数据在硬盘上的位置，和在内存中的位置发生了错位（PE文件被加载到内存后，各个节之间的间隙变大了），这就需要RVA和FOA这两套地址系统

## VA, RVA, FOA
### ImageBase：基址
+ 定义：PE文件被加载到虚拟内存空间时，其分配到的首地址。它在PE头的`OptionalHeader.ImageBase`中定义（常见的默认值为`0x00400000`或`0x00010000`）
+ 安全机制干扰：现代Windows启用了ASLR（地址空间布局随机化） 技术，这意味着同一个程序，每次运行装载时，操作系统为其分配的`ImageBase`都是动态随机变化的

<!-- 这是一张图片，ocr 内容为： -->
![](https://cdn.nlark.com/yuque/0/2026/png/43151352/1773494456190-f9e7735a-fdec-4b41-a063-d196167d859b.png)

### VA（Virtual Address）：虚拟内存地址
+ 定义：PE文件被加载到内存空间后，数据内存中的绝对地址。当CPU执行指令（如`JMP [地址]`）或通过指针访问内存时，使用的都是VA
+ 计算公式：`VA = ImageBase + RVA`
+ VA类似于地图上的绝对经纬度，无论你是怎么到达这个位置的，这个经纬度在当前地图体系下是唯一且确定的

### RVA（Relative Virtual Address）：相对虚拟地址
+ 定义：PE文件被加载到内存后，数据相对于PE文件在内存中起始位置（基址，ImageBase）的偏移量
+ 作用：这是PE文件内部使用最广泛的地址体系。由于ASLR机制导致`ImageBase`不固定（VA会随之失效），PE文件内部的各类目录（如导出表、导入表、重定位表）全部采用RVA来互相引用。这样无论程序被加载到内存的哪个位置，内部结构的相对位置关系始终保持不变
+ 类似于如果你所在的办公楼（ImageBase）整体搬迁了，大楼在城市的绝对地址（VA）变了，但“从大门进去上三楼左转”（RVA）这个相对路径永远有效
+ 计算公式：`RVA = VA - ImageBase`

### FOA（File Offset Address）：文件偏移地址（也叫RAW Offset）
+ 定义：目标数据在未运行的静态PE文件（硬盘中的`.exe/.dll`）中，相对于文件首字节的物理偏移量。用十六进制编辑器（如010Editor）查看文件时，左侧标尺显示的即为FOA
+ 类似于一本实体书第50页的第3行，这属于物理介质上的硬性排版，与它被谁拿在手里阅读（装载到哪里）无关
+ 作用：在进行静态逆向分析、特征码扫描等不运行程序而直接修改磁盘文件的操作时，必须将动态调试得到的RVA转换为FOA，才能在文件中准确找到要修改的物理字节

## RVA转换为FOA
+ 程序运行在内存布局，但PE文件存储在磁盘布局，但是内存布局不等于文件在磁盘的布局（内存按 SectionAlignment对齐，文件按FileAlignment对齐），所以地址必须通过节表（Section Table） 转换
+ 转换过程依赖于PE 头部的节表（Section Table），它由多个`IMAGE_SECTION_HEADER` 结构体组成。我们需要用到该结构体中的三个核心字段：
    - `VirtualAddress`：该节在内存中的起始RVA
    - `VirtualSize`：该节在内存中的实际大小
    - `PointerToRawData`：该节在磁盘文件中的起始FOA

### 计算步骤
+ 判定目标RVA归属哪个节：若满足以下条件，则目标RVA位于该节内

```plain
节的VirtualAddress ≤ RVA < 节的VirtualAddress + 节的VirtualSize
```

+ 计算节内偏移量（内部相对偏移）：数据在节内部的相对位置，无论是内存中还是硬盘上都是不变的

```plain
节内偏移 = RVA - 该节的VirtualAddress
```

+ 映射到物理文件，得出FOA：将求出的节内偏移量，叠加到该节在磁盘上的物理起始地址上

```plain
FOA = 该节的PointerToRawData + 节内偏移
```

###  RVA在PE头
+ 如果`RVA < 第一个节 VirtualAddress`那么`FOA = RVA`
+ 因为PE头文件布局=内存布局

# 五、三大表
+ PE头的`IMAGE_OPTIONAL_HEADER`末尾，存在一个非常核心的数组--数据目录表（DataDirectory）
+ 结构：该数组包含16个元素，每个元素长8字节，每个元素都是一个`IMAGE_DATA_DIRECTORY`结构体，用于记录某一种重要数据结构在映像中的RVA和大小。因此，数据目录表本质上是一个索引表，通过它可以快速定位导入表、导出表、重定位表等核心结构在PE映像中的位置，由两个核心字段组成
    - `VirtualAddress`（DWORD/4字节）：一个RVA（相对虚拟地址），指向某张表在内存中的起始位置
    - `Size`（DWORD/4字节）：该表占据的总字节数

<!-- 这是一张图片，ocr 内容为： -->
![](https://cdn.nlark.com/yuque/0/2026/png/43151352/1773583585788-13beabfa-b8e4-43a3-bf89-9d7725d87c67.png)

+ 索引映射：
    - `DataDirectory[0]`：指向 导出表（Export Table）
    - `DataDirectory[1]`：指向 导入表（Import Table）
    - `DataDirectory[5]`：指向 重定位表（Relocation Table）

<!-- 这是一张图片，ocr 内容为： -->
![](https://cdn.nlark.com/yuque/0/2026/png/43151352/1773583907671-486f2100-f99a-4f3d-9172-a880e020d246.png)

## 导入表
### 定义
+ 程序运行时需要调用外部动态链接库（DLL）的功能，但这些外部函数在运行前的真实内存地址是绝对未知的（就像你找人办事，却不知道他今天在哪间办公室），根本无法在编译时直接把地址写死在代码里
+ 导入表的作用正是充当一份外部依赖清单，精确记录下程序需要的DLL名称和具体函数名（比如：注明需要`user32.dll`里的`MessageBox`），当程序双击启动时，操作系统会首先阅读这份清单，把所需的DLL加载进内存，查到该函数此时真实的内存地址，并帮你把这个真实地址填写到程序预留的空白表格（IAT）中，从而确保你的程序能顺利找到并执行这些外部代码

### 结构
+ 导入表记录了当前程序运行所需调用的外部DLL名称，以及具体需要调用其中的哪些函数。系统通过解析`DataDirectory[1]`的RVA，会找到一个由`IMAGE_IMPORT_DESCRIPTOR`（简称IID）结构体组成的数组，程序依赖几个DLL，就有几个IID，最后以一个全0的IID作为结束标志

<!-- 这是一张图片，ocr 内容为： -->
![](https://cdn.nlark.com/yuque/0/2026/png/43151352/1773585124550-be20ec92-bf01-4325-a957-aa1f84bddd69.png)

+ 下图示例中，`Import Size = 0xC8`，`IMAGE_IMPORT_DESCRIPTOR = 0x14`，`0xC8 / 0x14 ≈ 10`，说明大约有10个IID，其中：9个有效（上图显示的9个），1 个结束

<!-- 这是一张图片，ocr 内容为： -->
![](https://cdn.nlark.com/yuque/0/2026/png/43151352/1773587246673-e47c0f9b-c584-43be-88c7-4b8430d972ad.png)

+ 一个IID有以下核心字段
    - `DUMMYUNIONNAME`（即`OriginalFirstThunk/INT`）：它本质上是一个指向导入名称表（INT，Import Name Table）的指针数组首地址，可以把它理解为一张纯只读的需求清单，OS加载器会顺着这个地址找过去，依次读取程序到底需要DLL里的哪些具体函数
    - `TimeDateStamp`（时间戳）：这个字段用于一种叫绑定导入（Bound Import）的优化技术。如果这里有具体的时间戳数值，说明在编译时，编译器已经提前把目标函数的真实绝对地址硬编码到程序里了。值为0则代表未绑定，这意味着当前程序必须老老实实地走标准流程，在程序启动时，由操作系统动态去查找地址并填入IAT中

<!-- 这是一张图片，ocr 内容为： -->
![](https://cdn.nlark.com/yuque/0/2026/png/43151352/1773586087547-950e5d69-9a72-43cf-93fb-765519f620f0.png)

    - `ForwarderChain`（转发链）：这个字段用于处理API转发机制（即你调用`A.dll`的函数，`A.dll`告诉你这个函数其实在`B.dll`里），在正常的导入表中，通常值为0 或 `0xFFFFFFFF(-1)`，表示当前依赖的DLL没有使用转发链机制

<!-- 这是一张图片，ocr 内容为： -->
![](https://cdn.nlark.com/yuque/0/2026/png/43151352/1773586100537-da965d23-65ce-4c59-aede-502060699372.png)

    - `Name`：一个RVA，指向当前依赖的DLL名称字符串

<!-- 这是一张图片，ocr 内容为： -->
![](https://cdn.nlark.com/yuque/0/2026/png/43151352/1773584946177-337d7aae-ef41-4c4b-b971-5714821feafc.png)

    - `FirstThunk`（即IAT，导入地址表）：这是导入表中最核心的动态数据区，它的本质是一个指针数组。在程序静态存放在硬盘时，这个表里存放的不是真正的函数地址，而是指向目标函数名（字符串）的相对地址（RVA）；但是，当程序被加载到内存准备运行时，操作系统会接管这个表，它会去外部DLL中查出每个函数真正的物理内存地址（VA），并将这些真实的内存地址直接覆盖写入到这个数组中。此后，程序代码在执行汇编的CALL指令时，实际上就是去这个已被系统动态填好真实地址的IAT数组中，读取目标地址并完成跳转
    - `ImportByName`数组（函数定位明细）：这是程序用来记录具体需要导入哪些函数的底层数据结构（即`IMAGE_IMPORT_BY_NAME`结构体）。该数组中的每一项都包含两个关键的寻址线索：一个是Hint（预期序号），另一个是Name（函数名ASCII 字符串）。操作系统在查找外部函数真地址时会先尝试直接把Hint的数值当作下标索引，去目标DLL的导出表中提取；如果因为系统版本更新导致目标DLL内部函数排序发生变化（即该索引对应的函数名不再匹配），系统就会提取后面的Name字符串，去DLL的导出表中按名字进行常规的完整搜索，从而确保最终一定能找到正确的真实地址并交由IAT完成装填

<!-- 这是一张图片，ocr 内容为： -->
![](https://cdn.nlark.com/yuque/0/2026/png/43151352/1773586901736-c10444bb-8ecc-431d-8853-97c8790a781f.png)

### 打印导入表
```plain
#include <stdio.h>
#include <windows.h>
#include <stdlib.h>

/* =========================================================================

 * PE 文件被 OS 加载到内存时，按 0x1000 (4KB) 对齐；在硬盘上静态存储时，按 0x200 (512字节) 对齐。
 * 因为本程序是直接按二进制读取的静态硬盘文件，而 PE 头中的指针全是指向内存的RVA
 * 因此，必须遍历节表 (Section Table)，利用公式 (RVA - 节区内存起始 RVA + 节区文件物理偏移) 算出真实 FOA。
 * ========================================================================= */
DWORD RvaToFoa(DWORD rva, PIMAGE_NT_HEADERS32 pNtHeader) {
    PIMAGE_SECTION_HEADER pSection = IMAGE_FIRST_SECTION(pNtHeader);

    for (int i = 0; i < pNtHeader->FileHeader.NumberOfSections; i++, pSection++) {
        // 若 RVA 落在当前节区的内存虚拟地址区间内
        if (rva >= pSection->VirtualAddress && rva < pSection->VirtualAddress + pSection->Misc.VirtualSize) {
            // 线性换算公式：目标FOA = RVA - 节区内存起始RVA + 节区文件物理起始FOA
            return rva - pSection->VirtualAddress + pSection->PointerToRawData;
        }
    }
    return rva;  // 若未落在任何节区 (通常在 PE 头部)，RVA 与 FOA 等价
}

void ParseImportTable(const char* filePath) {
    // 将 PE 文件按二进制载入内存 Buffer (模拟静态硬盘状态)
    FILE* fp = fopen(filePath, "rb");
    if (!fp) return;
    fseek(fp, 0, SEEK_END);
    long fileSize = ftell(fp);
    fseek(fp, 0, SEEK_SET);
    BYTE* buffer = (BYTE*)malloc(fileSize);
    fread(buffer, 1, fileSize, fp);
    fclose(fp);

    // 校验 DOS 头与 NT 头基线
    PIMAGE_DOS_HEADER pDos = (PIMAGE_DOS_HEADER)buffer;
    if (pDos->e_magic != IMAGE_DOS_SIGNATURE) return;

    PIMAGE_NT_HEADERS32 pNt = (PIMAGE_NT_HEADERS32)(buffer + pDos->e_lfanew);
    if (pNt->Signature != IMAGE_NT_SIGNATURE) return;

    /* =========================================================================
     * 一、 提取总入口：定位导入表
     * 数据目录表 (DataDirectory) 的第 2 项 (索引 1) 记录了导入表的 RVA 与 Size
     * ========================================================================= */
    DWORD importRva = pNt->OptionalHeader.DataDirectory[1].VirtualAddress;
    if (importRva == 0) return;  // 若为 0，说明无导入表或已被加壳抹除

    // 获取 IMAGE_IMPORT_DESCRIPTOR (IID) 结构体数组的首地址
    PIMAGE_IMPORT_DESCRIPTOR pIID = (PIMAGE_IMPORT_DESCRIPTOR)(buffer + RvaToFoa(importRva, pNt));

    /* =========================================================================
     * 二、 模块级解析：外部 DLL 依赖链
     * 程序依赖了几个外部 DLL，就会生成几个连续的 IID 结构体。
     * 数组以一个全 0 填充的 IID 结构体作为终止符 (Null-terminated)。
     * ========================================================================= */
    while (pIID->Name != 0) {
        // Name 字段指向 DLL 的名称字符串 (如 "KERNEL32.dll")。
        // OS 运行初期，会读取此字符串并暗中调用 LoadLibrary 将其映射至当前进程。
        printf("\n[+] 依赖模块 (DLL): %s\n", (char*)(buffer + RvaToFoa(pIID->Name, pNt)));

        /* -------------------------------------------------------------------------
         * INT 与 IAT 的双表机制
         * OriginalFirstThunk 指向 INT（只读的调用需求清单）
         * FirstThunk         指向 IAT（预留的真实地址收货区）
         *
         * 静态真相：在当前硬盘文件中，这两个指针指向的底层数据完全一致，全是指向函数名的 RVA
         * 动态突变：当 OS 加载该程序时，会根据 INT 的名字去找到外部函数的真实物理绝对地址 (VA)
         *           然后暴力覆写进 FirstThunk (IAT) 对应的内存格子里
         *           此后，程序代码中所有的 CALL 指令，访问的将是被系统填满真实地址的 IAT 区域。
         * ------------------------------------------------------------------------- */
        printf("    |-- INT (导入名称表 OriginalFirstThunk RVA): %08X\n", pIID->OriginalFirstThunk);
        printf("    |-- IAT (导入地址表 FirstThunk         RVA): %08X\n", pIID->FirstThunk);

        /* =========================================================================
         * 三、 函数级解析：提取具体函数清单 (Thunk 数组)
         * INT 内部是由 IMAGE_THUNK_DATA 构成的数组，记录了对当前 DLL 内部具体的函数调用需求
         * ========================================================================= */
        DWORD thunkRva = pIID->OriginalFirstThunk != 0 ? pIID->OriginalFirstThunk : pIID->FirstThunk;
        PIMAGE_THUNK_DATA32 pThunk = (PIMAGE_THUNK_DATA32)(buffer + RvaToFoa(thunkRva, pNt));

        while (pThunk->u1.AddressOfData != 0) {

            /* =====================================================================
             * 四、 底层寻址决策逻辑
             * 微软通过判断这 4 字节数据（IMAGE_THUNK_DATA）的最高有效位(MSB)，
             * 来决定操作系统去外部 DLL 找这个函数真实地址的寻址策略
             * ===================================================================== */
            if (IMAGE_SNAP_BY_ORDINAL32(pThunk->u1.Ordinal)) {

                // 分支 A：按序号寻址 (最高位为 1)
                // 剔除最高位后，保留的低 31 位即为目标函数在 DLL 导出表中的绝对序号。
                // OS 加载器直接将此序号作为数组下标提取目标真实地址。速度极快，常用于系统底层隐蔽调用。
                printf("    [-] [按序号] Ordinal = %u\n", IMAGE_ORDINAL32(pThunk->u1.Ordinal));
            }
            else {

                // 分支 B：按名称寻址 (最高位为 0)
                // 此时数据是一个 RVA 指针，指向包含函数名明细的 IMAGE_IMPORT_BY_NAME 结构体。
                PIMAGE_IMPORT_BY_NAME pByName = (PIMAGE_IMPORT_BY_NAME)(buffer + RvaToFoa(pThunk->u1.AddressOfData, pNt));

                // Hint (WORD/2字节) : 编译器预测的导出序号。OS 加载器优先尝试以此序号去目标DLL中极速盲抽。
                // Name (变长字符串) : 若系统更新导致目标DLL内部排序错位 (Hint失效)，OS 将启动备用方案，
                //                     提取此 ASCII 字符串进行二分查找，确保绝对准确地定位真实地址。
                printf("    [-] [按名称] Hint = %-4u | Name = %s\n", pByName->Hint, pByName->Name);
            }
            pThunk++;
        }
        pIID++;
    }

    free(buffer);
}

int main() {
    ParseImportTable("C:\\Windows\\SysWOW64\\notepad.exe");
    return 0;
}
```

## 导出表
### 定义
+ 导出表的设计初衷就是把自己的代码打包共享给其他程序使用
+ 外部程序面对一个完全陌生的文件，根本不知道里面具体包含了哪些可用的功能，更不知道这些功能对应的代码藏在文件内部的哪个相对位置（就像顾客进了饭店不知道能点什么菜）
+ 导出表的作用正是充当一份对外公开的服务菜单与指路牌，它精确记录了当前文件愿意提供给外界使用的所有函数名称（或序号）以及它们在自己内部的RVA，当其他程序（或系统加载器）顺着导入表来找人办事时，只需查阅这个DLL的导出表，就能找到目标函数的真实代码入口，从而实现跨模块的代码共享与服务提供

### 结构
+ NT头中的`IMAGE_OPTIONAL_HEADER`内部的16元素数据目录表（Data Directory）是关键；最后通过数据目录表索引为0的第一个元素，即可获取导出表对应的RVA（相对虚拟地址）和大小（Size），借助这个RVA（内存中直接使用，文件中需转换为 FOA）就能精准定位到导出表的核心结构

<!-- 这是一张图片，ocr 内容为： -->
![](https://cdn.nlark.com/yuque/0/2026/png/43151352/1773660442147-66658912-724b-4ad3-8ff0-98d9fa5682fc.png)

+ 主要有以下一些核心字段
    - `Characteristics`（特征值 / 属性标志）：它的设计初衷是想用来打上一些特殊的标记，告诉操作系统这个导出表有一些什么特殊的属性。但在实际的Windows操作系统发展历史中，这个字段一直没有被真正启用，所以它一般都是0，操作系统在加载DLL时也会无视它

<!-- 这是一张图片，ocr 内容为： -->
![](https://cdn.nlark.com/yuque/0/2026/png/43151352/1773659424783-47e65471-cf60-4ce7-9e59-88b7cc56926e.png)

    - `TimeDateStamp`（时间戳）：记录的是这个DLL文件被程序员用编译器编译生成的精确时间

<!-- 这是一张图片，ocr 内容为： -->
![](https://cdn.nlark.com/yuque/0/2026/png/43151352/1773659474839-4ff68e67-8ab5-4477-a406-f0a0152e28cb.png)

    - `MajorVersion`与`MinorVersion`（主版本号与次版本号）：这两个字段的作用是由开发者自己定义的，用来标记这个导出表或者DLL的版本（比如1.0 版、2.1版）。Windows操作系统在运行程序时不校验这两个字段的内容，所以很多编译器把它们默认写死成0

<!-- 这是一张图片，ocr 内容为： -->
![](https://cdn.nlark.com/yuque/0/2026/png/43151352/1773659580807-c68f0967-2c4b-4309-818a-908ddb9c9552.png)

    - `Name`（模块名称指针）：这是一个RVA，这里面存的不是字符串本身，而是一个地址，顺着这个地址找过去，在内存（或文件）的另一个角落，存放着一串英文字符，这串字符才是这个DLL真正的内部名字

<!-- 这是一张图片，ocr 内容为： -->
![](https://cdn.nlark.com/yuque/0/2026/png/43151352/1773659663167-ef0bfb2a-80df-4f76-8b51-180892042890.png)

    - `Base`（导出序号基数）：通常它的值是1，在 PE 结构中，有些程序调用函数是不靠名字的，而是靠序号（Ordinal），这个`Base`就是所有序号的起步，当系统收到指令说“我要调用序号为10的函数”时，系统会用`10 - Base`，算出一个差值，然后再用这个差值去数组里找具体的函数

<!-- 这是一张图片，ocr 内容为： -->
![](https://cdn.nlark.com/yuque/0/2026/png/43151352/1773659838359-9521cd2c-b584-4266-8ccf-3dda7a5165ec.png)

    - `NumberOfFunctions`（导出函数总数）：它规定了三大核心表之一的导出地址表（EAT）里面到底有多少个元素（有多少条记录），它代表了这个DLL物理上真实提供了多少个函数代码供别人使用

<!-- 这是一张图片，ocr 内容为： -->
![](https://cdn.nlark.com/yuque/0/2026/png/43151352/1773659820630-eabb2af8-eb33-4b12-a741-61575a512d3e.png)

    - `NumberOfNames`（以名称导出的函数总数）：只统计有名字的导出项，不含仅以序号（NONAME）导出的函数，其数值等于`AddressOfNames`指向的导出名称表数组的元素个数，同时也与`AddressOfNameOrdinals` 名称序号表的元素个数一致，且一定小于或等于`NumberOfFunctions`，是解析按名称导出函数、遍历名称表、定位函数地址的核心计数依据

<!-- 这是一张图片，ocr 内容为： -->
![](https://cdn.nlark.com/yuque/0/2026/png/43151352/1773660028457-eba38f97-aad7-46c2-a8be-24bd62fbcb2a.png)

    - `AddressOfFunctions`（导出函数地址表，EAT）：这个RVA指向一个装满内存地址的数组，这个数组里的每一个元素，都是一段真正可以被CPU执行的机器码代码在内存中的起始位置。这是所有寻找函数过程的最终目的地。通俗来讲这是一张“藏宝图”，图上标满了每一个宝箱（函数代码）的精准GPS坐标。不管你前面怎么绕，最终都必须来到这张图上找坐标，才能拿到真金白银
    - `AddressOfNames`（导出函数名称表，ENT）：这个RVA指向一个装满字符串地址的数组，数组里的每一个地址，都指向一个函数的名字（比如`printf`）。通俗来说这类似于挂在餐厅门口的一本菜单，上面的菜名不仅全，而且严格按拼音/字母等排序，顾客想吃什么，直接按字母索引翻找，一找一个准
    - `AddressOfNameOrdinals`（导出函数序号表，EOT）：这个RVA指向一个装满短整数的数组，它是连接ENT和EAT的桥梁，当系统在ENT的第N行找到了需要的函数名，系统就会来到这个EOT的第N行取出一个数字，拿到这个数字后，系统才会去EAT里寻找最终的位置

<!-- 这是一张图片，ocr 内容为： -->
![](https://cdn.nlark.com/yuque/0/2026/png/43151352/1773660313742-b74b8009-6ad7-4beb-9be4-2cee0ec09d3a.png)

### 打印导出表
```c
#include <windows.h>
#include <stdio.h>

/*
整体流程
1. 文件读入内存（buffer）
2. 解析 DOS头 → 找 NT头
3. 从 NT头 找到 导出表入口
4. 定位 IMAGE_EXPORT_DIRECTORY
5. 解析三张表：
       - AddressOfFunctions      → 函数地址表 (DWORD[])
       - AddressOfNames          → 名字表 (DWORD[])
       - AddressOfNameOrdinals   → 序号表 (WORD[])
6. 用三表关系解析出：
       函数名 → 函数地址
*/


/*
RVA → FOA
    PE结构里的地址（导出表、函数名等）基本都是 RVA
    但现在操作的是“文件 buffer”所以必须转成文件偏移（FOA）
这个函数就是做这个转换

*/
DWORD RvaToFoa(DWORD rva, PIMAGE_NT_HEADERS nt, BYTE* base)
{
    PIMAGE_SECTION_HEADER section = IMAGE_FIRST_SECTION(nt);

    // 遍历所有节（.text .rdata .data 等）
    for (int i = 0; i < nt->FileHeader.NumberOfSections; i++, section++)
    {
        // 判断RVA属于哪个节
        if (rva >= section->VirtualAddress &&
            rva < section->VirtualAddress + section->Misc.VirtualSize)
        {
            // 转换成文件偏移返回
            return rva - section->VirtualAddress + section->PointerToRawData;
        }
    }

    return rva;
}


int main()
{
    const char* path = "D:\\code\\C\\active_desktop_render\\x64\\Release\\libcares-2.dll";

    /*
    1：把DLL文件完整读入内存：因为要手动解析PE结构，所以必须自己掌控每一字节
    此时buffer就是“文件镜像”≠ 程序加载后的内存
    所以后面所有访问都基于 buffer + 偏移
    */

    FILE* fp = fopen(path, "rb");

    if (!fp)
    {
        printf("文件打开失败\n");
        return -1;
    }

    // 获取文件大小
    fseek(fp, 0, SEEK_END);
    DWORD size = ftell(fp);
    rewind(fp);

    // 分配内存
    BYTE* buffer = (BYTE*)malloc(size);

    // 读入整个文件
    fread(buffer, 1, size, fp);
    fclose(fp);

    printf("文件已加载到内存: %p\n\n", buffer);

    /*
    2：解析 DOS Header
    */

    PIMAGE_DOS_HEADER dos = (PIMAGE_DOS_HEADER)buffer;

    // 校验是否是MZ
    if (dos->e_magic != IMAGE_DOS_SIGNATURE)
    {
        printf("不是有效PE文件\n");
        return -1;
    }

    printf("[DOS] e_lfanew = 0x%X\n\n", dos->e_lfanew);

    /*
    3：解析 NT Header】
    NT头位置：buffer + e_lfanew
    IMAGE_NT_HEADERS 作用：
        PE签名（PE\0\0）
        节区信息（有多少节）
        OptionalHeader
    */

    PIMAGE_NT_HEADERS nt =
        (PIMAGE_NT_HEADERS)(buffer + dos->e_lfanew);

    if (nt->Signature != IMAGE_NT_SIGNATURE)
    {
        printf("NT头错误\n");
        return -1;
    }

    printf("[NT] 节数量: %d\n\n", nt->FileHeader.NumberOfSections);

    /*
    4：找到导出表入口
    在OptionalHeader中有DataDirectory[16]，每个元素代表一种数据结构：
        [0] 导出表
        [1] 导入表
        [2] 资源表
        ...
        现在只关心DataDirectory[EXPORT]
    */

    IMAGE_DATA_DIRECTORY exportData =
        nt->OptionalHeader.DataDirectory[IMAGE_DIRECTORY_ENTRY_EXPORT];

    if (exportData.VirtualAddress == 0)
    {
        printf("该DLL没有导出表\n");
        return 0;
    }

    printf("[Export] RVA: 0x%X\n", exportData.VirtualAddress);

    /*
        把导出表位置从 RVA 转为 FOA
    */
    DWORD exportFOA = RvaToFoa(exportData.VirtualAddress, nt, buffer);

    printf("[Export] FOA: 0x%X\n\n", exportFOA);

    /*
    5：解析 IMAGE_EXPORT_DIRECTORY
    这个结构就是“导出表头”
    它不直接存函数，而是存三张表的位置
    */

    PIMAGE_EXPORT_DIRECTORY exp =
        (PIMAGE_EXPORT_DIRECTORY)(buffer + exportFOA);

    printf("======== 导出表基本信息 ========\n");
    printf("函数总数: %d\n", exp->NumberOfFunctions);
    printf("有名字的函数数: %d\n", exp->NumberOfNames);
    printf("序号起始(Base): %d\n\n", exp->Base);

    /*
    6：定位三张核心表
    exp 里有三个最重要字段：
        AddressOfFunctions      -> 函数地址表
        AddressOfNames          -> 函数名表
        AddressOfNameOrdinals   -> 序号表
    注意：这些字段都是 RVA，需要转换为FOA后再进行访问
    */

    DWORD funcFOA = RvaToFoa(exp->AddressOfFunctions, nt, buffer);
    DWORD nameFOA = RvaToFoa(exp->AddressOfNames, nt, buffer);
    DWORD ordFOA = RvaToFoa(exp->AddressOfNameOrdinals, nt, buffer);

    /*
        把它们当成数组来用
    */
    DWORD* funcTable = (DWORD*)(buffer + funcFOA);
    DWORD* nameTable = (DWORD*)(buffer + nameFOA);
    WORD* ordTable = (WORD*)(buffer + ordFOA);

    /*
    7：导出表的结构
    nameTable:
        [0] -> "funcA"
        [1] -> "funcB"

    ordTable:
        [0] -> 2
        [1] -> 0

    funcTable:
        [0] -> RVA_X
        [1] -> RVA_Y
        [2] -> RVA_Z

    --------------------------------------------------

    查找流程：

        i = 0:
            name = "funcA"
            index = 2
            addr = funcTable[2]

        i = 1:
            name = "funcB"
            index = 0
            addr = funcTable[0]

    --------------------------------------------------

    重点：
    名字顺序 ≠ 函数顺序
    中间必须经过 ordTable
    */

    printf("======== 导出函数列表 ========\n");

    for (DWORD i = 0; i < exp->NumberOfNames; i++)
    {
        // 获取函数名
        DWORD nameRVA = nameTable[i]; // 字符串地址（RVA）
        DWORD nameFOA_real = RvaToFoa(nameRVA, nt, buffer);

        char* funcName = (char*)(buffer + nameFOA_real);

        // 获取索引：ordTable[i] 不是序号，而是 funcTable 的下标
        WORD index = ordTable[i];

        // 通过索引找到函数地址
        DWORD funcRVA = funcTable[index];

        /*
        计算“对外显示的序号”
        实际导出序号：ordinal = Base + index
        */
        DWORD ordinal = exp->Base + index;

        /*
        核心逻辑
            funcName
                ↓
            ordTable[i]  -> index
                ↓
            funcTable[index] -> 函数地址
        */

        printf("%-30s | Ordinal: %-5d | RVA: 0x%X\n",
            funcName, ordinal, funcRVA);
    }

    free(buffer);
    return 0;
}
```

## 重定位表
### 为什么需要重定位表
+ 假设写了一段代码，里面有一个全局变量`GlobalVar`，还有一个指令要把它的值赋给寄存器

```c
mov eax, dword ptr [GlobalVar]
```

+ 情况A：不需要重定位
    - 编译器在编译时，假设程序会按期望加载到`0x00400000`（ImageBase），它算出来`GlobalVar`的RVA是`0x2000`
    - 那么它直接把这条指令编译成硬编码的机器码：去访问绝对地址`0x00402000`（VA）
    - 如果程序运行时，操作系统真的把它放到了`0x00400000`，一切正常，完美运行
+ 情况B：必须重定位
    - 如果`0x00400000`这个内存位置已经被其他DLL占用了，或者操作系统开启了ASLR（地址空间布局随机化），操作系统只能把程序加载到另一个基址，比如 `0x00600000`
    - 这时候，程序内部那条指令还在傻傻地访问绝对地址`0x00402000`，但在这个新环境下，真正的变量地址应该是`0x00600000 + 0x2000 = 0x00602000`
    - 结果： 程序访问了错误的内存，直接崩溃（Access Violation）
+ 结论：因为程序内部有很多写死的绝对地址（如全局变量地址、函数指针等），当程序实际加载的基址和编译器期望的基址不一样时，这些绝对地址就全部失效了，重定位表就是为了解决这个问题而存在的。

### 重定位表的作用
+ 重定位表（Relocation Table）是程序或可执行文件中用于记录地址修正信息的数据结构，当程序被加载到内存中的实际地址与编译或链接时的假定地址不一致时，操作系统或加载器会根据重定位表中的条目，对代码和数据中涉及地址的部分进行调整。
+  可以简单理解为地址修正清单：程序在编译时并不知道将来会被放到内存的哪个具体位置，所以里面有些地址是暂时写的，当程序真正运行时，操作系统会根据重定位表，把这些需要修改的地址统一调整成当前内存中的真实位置
+ 总的来说它的作用就是让同一个程序不管被放到内存的哪里都能正常运行
+ 当操作系统（Windows PE Loader）把程序加载到内存时，它会做以下判断：
    - 如果`实际加载基址 == 期望基址`：直接运行，完全不管重定位表
    - 如果`实际加载基址 != 期望基址`：计算出差值（`Delta = 实际基址 - 期望基址`），然后根据重定位表这份清单，把程序里所有写死的绝对地址，全部加上这个差值，从而修正所有的内存指向
    - 这个修正的过程，就叫重定位

### 结构
+ 重定位表（Base Relocation Table）是通过NT头中的`IMAGE_OPTIONAL_HEADER`里的数据目录表来定位的，在数据目录表中，索引为5（`IMAGE_DIRECTORY_ENTRY_BASERELOC`）的元素存放的就是重定位表的RVA和Size。通过这个RVA（内存中可直接使用，文件中需转换为FOA），就可以定位到整个重定位表结构

<!-- 这是一张图片，ocr 内容为： -->
![](https://cdn.nlark.com/yuque/0/2026/png/43151352/1774165439175-690b4d16-9749-4fe3-9554-69626dae2786.png)

+ 主要有以下一些核心字段：
    - `VirtualAddress`（页面起始的相对虚拟地址）：这是一个RVA，表示当前这个重定位块所作用的内存页的起始地址。可以理解为这一批需要修改的地址都在这一页里，后续所有偏移都是基于这个地址计算的

<!-- 这是一张图片，ocr 内容为： -->
![](https://cdn.nlark.com/yuque/0/2026/png/43151352/1774165857923-eb614773-d83e-4170-a814-7e5604009c28.png)

    - `SizeOfBlock`（当前块的总大小）：它记录了当前这个重定位块占用的总字节数（包括头部和后面的所有重定位项）。它是遍历重定位表时的核心计数依据，系统通过它能算出当前这个页面内有多少个地方需要被打上重定位补丁，同时也指明了下一个重定位块在内存中的物理起点

<!-- 这是一张图片，ocr 内容为： -->
![](https://cdn.nlark.com/yuque/0/2026/png/43151352/1774166242106-a7515ac8-2bcf-4730-87c9-cf4816304f2f.png)

    - `Block`（重定位项数组）其实是整个重定位表里最核心的部分，它本质上是一串连续的2字节数据，每一项都精确描述这一页里某个具体位置该怎么修。每个Block项是一个16位的值，被拆成两部分：高4位是类型（Type），决定修正方式（比如DIR64表示对64位地址加上基址差值，ABSOLUTE表示占位不处理）；低12位是偏移（Offset），表示要修正的位置相对于当前块的VirtualAddress的偏移量。系统处理时，会先拿块头里的VirtualAddress作为这一页的基准地址，然后遍历这些Block项，用`VirtualAddress + Offset`定位到具体需要修改的内存位置（`RVA = VirtualAddress + Offset`），再根据Type的规则进行地址修正。可以把Block理解为在一页内，把所有需要修改的地址用`类型+偏移`的压缩方式打包存起来，系统逐条解析并打补丁，从而完成整个程序的重定位

<!-- 这是一张图片，ocr 内容为： -->
![](https://cdn.nlark.com/yuque/0/2026/png/43151352/1774166706214-38596e04-e2cb-46a0-8675-0209c4a04088.png)

### 打印重定位表
```c
#include <stdio.h>
#include <stdlib.h>
#include <windows.h> 

// 模块一：RVA 到 FOA 转换函数 (核心工具)
// 作用：因为我们在读取磁盘文件，PE 头里给的地址是内存地址(RVA)
// 需要根据“节表(Section Table)”把内存地址转换为磁盘文件偏移(FOA)
DWORD RvaToOffset(DWORD rva, PIMAGE_SECTION_HEADER sectionHeader, WORD numSections) {
    // 遍历每一个节（如 .text, .data, .reloc 等）
    for (int i = 0; i < numSections; i++) {
        // 判断这个 RVA 是否落在当前这个节的内存范围内
        if (rva >= sectionHeader[i].VirtualAddress &&
            rva < (sectionHeader[i].VirtualAddress + sectionHeader[i].Misc.VirtualSize)) {

            // 如果在这个节里，计算公式：
            // 真实文件偏移 = RVA - 节在内存中的起始RVA + 节在文件中的起始偏移
            return rva - sectionHeader[i].VirtualAddress + sectionHeader[i].PointerToRawData;
        }
    }
    return rva; // 如果找不到，直接返回（通常意味着出错或属于PE头部）
}

int main() {
    // 1. 打开指定的 DLL 文件
    const char* filePath = "D:\\code\\C\\active_desktop_render\\x64\\Release\\libcares-2.dll";
    FILE* file = fopen(filePath, "rb");
    if (!file) {
        printf("文件打开失败，请检查路径是否正确！\n");
        return -1;
    }

    // 2. 获取文件大小并把整个文件读取到内存中（申请一个大数组存放文件字节）
    fseek(file, 0, SEEK_END);
    long fileSize = ftell(file);
    fseek(file, 0, SEEK_SET);

    // fileBuffer 这个指针变量就记录了这段内存空间的起始物理内存地址
    // 这个地址在逻辑上对应着硬盘文件的绝对开头（即第1个字节，偏移量为0的位置）
    // 确立了这个零点映射关系后，整个文件在内存中就变成了一把展开的标尺
    // 后续我们无论解析出哪个数据结构（如重定位表）在文件中的真实字节偏移量（FOA），只需将其直接加上 fileBuffer 这个基准地址
    // C语言的指针就能通过 基准地址 + 偏移量 的简单指针加法，精准越过前面的无关数据，直接定位并提取我们需要的核心结构。
    unsigned char* fileBuffer = (unsigned char*)malloc(fileSize);
    fread(fileBuffer, 1, fileSize, file);
    fclose(file);


    // 解析 PE 头部，定位重定位表
    // 3. 解析 DOS 头 (占据文件最开始的位置)
    PIMAGE_DOS_HEADER dosHeader = (PIMAGE_DOS_HEADER)fileBuffer;
    if (dosHeader->e_magic != IMAGE_DOS_SIGNATURE) { // 检查 "MZ" 标志
        printf("不是有效的 PE 文件 (找不到 MZ 标志)\n");
        free(fileBuffer); return -1;
    }

    // 4. 解析 NT 头
    PIMAGE_NT_HEADERS64 ntHeader = (PIMAGE_NT_HEADERS64)(fileBuffer + dosHeader->e_lfanew);
    if (ntHeader->Signature != IMAGE_NT_SIGNATURE) { // 检查 "PE\0\0" 标志
        printf("不是有效的 PE 文件 (找不到 PE 标志)\n");
        free(fileBuffer); return -1;
    }

    // 5. 获取节表 (紧跟在 NT 头里的 OptionalHeader 后面)
    // IMAGE_FIRST_SECTION 是系统宏，用来越过可选头，精准找到第一个节表
    PIMAGE_SECTION_HEADER sectionHeader = IMAGE_FIRST_SECTION(ntHeader);
    WORD numSections = ntHeader->FileHeader.NumberOfSections;

    // 6. 从可选头的数据目录表(Data Directory)中，提取索引为 5 的重定位表信息
    DWORD relocRva = ntHeader->OptionalHeader.DataDirectory[IMAGE_DIRECTORY_ENTRY_BASERELOC].VirtualAddress;
    DWORD relocSize = ntHeader->OptionalHeader.DataDirectory[IMAGE_DIRECTORY_ENTRY_BASERELOC].Size;

    if (relocRva == 0 || relocSize == 0) {
        printf("该文件没有重定位表！\n");
        free(fileBuffer); return 0;
    }

    // 7. 把重定位表的内存相对地址(RVA) 转换为 文件偏移(FOA)
    DWORD relocOffset = RvaToOffset(relocRva, sectionHeader, numSections);

    // 模块三：核心！遍历解析重定位表的 Block (块) 结构
    // 让指针指向重定位表在文件中的真实起始位置
    PIMAGE_BASE_RELOCATION relocBlock = (PIMAGE_BASE_RELOCATION)(fileBuffer + relocOffset);

    printf("============ 开始解析重定位表 ============\n");
    printf("重定位表 RVA: 0x%08X, 总大小: %d 字节\n\n", relocRva, relocSize);

    int blockCount = 0; // 记录一共有多少个 4KB 页面块

    // 循环条件：遍历直到遇到一个 VirtualAddress 和 SizeOfBlock 都是 0 的空块（结束标志）
    while (relocBlock->VirtualAddress != 0 && relocBlock->SizeOfBlock != 0) {
        blockCount++;

        printf("[Block %d] 所在 4KB 页的基准 RVA: 0x%08X | 当前块总大小: %d 字节\n",
            blockCount, relocBlock->VirtualAddress, relocBlock->SizeOfBlock);

        // 计算当前这个 Block 里面，一共有多少个需要打补丁的项
        // 公式：(当前块总大小 - 头部8个字节) / 每个具体项占2个字节
        int entryCount = (relocBlock->SizeOfBlock - 8) / 2;
        printf("          包含的重定位项数量: %d 项\n", entryCount);

        // TypeOffset 数组并没有在 PIMAGE_BASE_RELOCATION 结构体里定义
        // 它们紧紧挨在头部(8字节)的后面，所以将指针向后移动8个字节，并强转为 WORD* (2字节指针)
        WORD* typeOffsetArray = (WORD*)((unsigned char*)relocBlock + 8);

        // 遍历当前块里的每一个具体重定位项
        for (int i = 0; i < entryCount; i++) {
            WORD entry = typeOffsetArray[i]; // 取出这 2 个字节 (16比特)的数据

            // 核心位运算：拆分这 2 个字节
            // 1. 取高 4 位：把这16位数据向右移动12位，剩下的就是高4位（类型）
            WORD type = entry >> 12;

            // 2. 取低 12 位：用 0xFFF (二进制 0000 1111 1111 1111) 进行按位与操作，屏蔽掉高4位
            WORD offset = entry & 0x0FFF;

            // 如果 type == 0 (IMAGE_REL_BASED_ABSOLUTE)，说明是为了内存对齐凑数的废数据，忽略
            if (type == 0) {
                continue;
            }

            // 计算出最终需要修改的数据的真实内存相对地址(RVA)
            DWORD targetRva = relocBlock->VirtualAddress + offset;

            // 为了避免控制台输出几万行数据导致卡死，每个 Block 只打印前 50 项作为演示
            if (i < 50) {
                printf("          -> [项 %d] 类型: %d, 内部偏移: 0x%03X, 需修正的真实RVA: 0x%08X\n",
                    i + 1, type, offset, targetRva);
            }
        }

        printf("--------------------------------------------------\n");

        // 移动指针，跳过当前的 Block，准备解析下一个 Block
        // 当前块的首地址 + 当前块的总大小 = 下一个块的首地址
        relocBlock = (PIMAGE_BASE_RELOCATION)((unsigned char*)relocBlock + relocBlock->SizeOfBlock);
    }

    printf("============ 重定位表解析结束，共解析 %d 个 Block ============\n", blockCount);

    // 释放申请的内存空间
    free(fileBuffer);
    return 0;
}
```

## 总结
+ 导入表（Import Table）：导入表用于描述一个PE文件在运行时依赖哪些外部DLL及其函数，本质上是一张依赖清单，核心原理是动态链接。当程序需要调用自身代码中没有的函数（如MessageBox等WindowsAPI）时，编译器不会把这些函数的实际代码打包进可执行文件中，而是在导入表中记录下所需的DLL名称及具体函数名（或序号）。在程序启动时，操作系统的PE装载器会解析这张表，找到目标DLL，并将这些外部函数在内存中的实际物理地址动态填写到程序的导入地址表（IAT）中，这样程序就可以通过IAT间接调用API函数
+ 导出表（Export Table）：导出表用于描述一个DLL向外提供了哪些函数或符号，是DLL对外的“接口说明书”，其核心原理是接口地址映射。它记录了当前模块愿意分享给其他程序使用的函数名称、导出序号，以及这些函数在模块内部的相对虚拟地址（RVA）。当其他程序（通过其导入表）或通过代码动态请求某个函数时，系统会在这张导出表里查找目标函数的名称或序号，从而定位并计算出该函数在内存中的真实执行地址。它的主要作用是赋予PE文件向外输出功能接口的能力
+ 重定位表（Relocation Table）：重定位表用于解决PE文件无法加载到其首选基址（ImageBase）时的地址修正问题，其核心原理是基址重置（Base Relocation）。在编译时，编译器会假设程序加载到一个默认的优先基址（ImageBase，如0x400000），并据此在代码中生成许多绝对内存地址；但实际运行时，由于该基址可能已经被其他模块占用，或者受系统安全机制（如ASLR，地址空间布局随机化）影响，PE文件往往会被加载到一个全新的随机地址。此时，重定位表记录了程序中所有硬编码绝对地址的具体位置，PE装载器会计算出实际加载地址与默认基址的差值（Delta），并根据重定位表的指引，把这个差值逐一加到那些失效的绝对地址上，按照新的基址差值对这些地址进行调整，这样可以保证程序在不同内存地址下仍能正确运行
+ 一句话理解：

```c
导入表：我需要用谁的功能？（买家购物清单，系统负责填发货地址）
导出表：别人能用我的什么功能？（卖家商品目录，标明了商品在仓库的哪个货架）
重定位表：如果我没分到想要的内存地盘，代码里哪些死地址需要重新计算？（搬家后的地址修改备忘录）
```

