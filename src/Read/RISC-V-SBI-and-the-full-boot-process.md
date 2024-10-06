# 【翻译】RISC-V SBI and the full boot process

> **原文作者**： [@popovicu](https://github.com/popovicu)
>
> **原文链接**： [RISC-V SBI and the full boot process](https://popovicu.com/posts/risc-v-sbi-and-full-boot-process/)
>
> **版权声明**： 本文为  [@popovicu](https://github.com/popovicu) 的原创作品。本文的翻译仅供学习参考，更多详细信息请访问原文链接。

# RISC-V SBI 和完整的启动过程

发表于：*2023年9月9日 | 晚上09:00*

**[@popovicu94](https://x.com/intent/follow?original_referer=https%3A%2F%2Fpopovicu.com%2F&ref_src=twsrc%5Etfw%7Ctwcamp%5Ebuttonembed%7Ctwterm%5Efollow%7Ctwgr%5Epopovicu94&region=follow_link&screen_name=popovicu94)**

在上一篇文章中，我们讨论了[RISC-V上的裸机编程](https://popovicu.com/posts/bare-metal-programming-risc-v)。在继续阅读本文之前，请先熟悉之前的内容，因为本文是对前述文章的直接延续。

这次我们将讨论 RISC-V 的 **SBI（Supervisor Binary Interface，监督者二进制接口）**，并以 **OpenSBI** 作为示例。我们将探讨 SBI 如何帮助我们实现操作系统内核的基本功能，并以 `riscv64 virt` 机器为例结束本文。

## RISC-V 和“BIOS”

在前文中，我们详细讨论了 RISC-V 启动过程的最初阶段。我们提到，首先运行的是 ZSBL（零阶段引导加载程序），它初始化一些寄存器并跳转到 ZSBL 硬编码的地址。在 QEMU 的 `riscv64 virt` 虚拟机中，这个硬编码的地址是 `0x80000000`。这就是用户提供的代码首次运行的地方，默认情况下，QEMU 会在这里加载 `OpenSBI`。

### 机器模式

到目前为止，我们避免讨论不同的机器模式，现在是引入它们的最佳时机。机器模式的概念在于，并不是每个软件都应该能够访问机器上的任意内存地址，甚至不是所有的指令都应该被 CPU 执行。传统的教科书示例中，通常有两种主要的权限划分：

1. 特权模式
2. 非特权模式

*特权模式* 是机器在启动时的模式。此时允许执行任何指令，且没有地址访问违规的概念。一旦操作系统接管系统控制并开始执行用户代码（即用户空间代码），模式就会开始切换。当用户代码在 CPU 核心上运行时，它是在 *非特权模式* 下运行，此时并不能访问所有资源。返回内核模式意味着切换回特权模式。

这是操作权限的一种非常教科书化的简化视角，但问题来了：为什么只有两种模式？

在系统中，通常有超过两种模式，形成多个访问模式的[保护环](https://en.wikipedia.org/wiki/Protection_ring)。RISC-V 规范并没有明确规定一个核心必须实现哪些模式，除了 **M（机器）模式**，这是最具特权的模式。

通常，只有 M 模式的处理器是简单的嵌入式系统，而更安全的系统会支持 M 和 S 模式，直到可以运行类似 Unix 操作系统的完整系统（支持 M、S 和 U 模式）。

### SBI

[官方文档](https://github.com/riscv-non-isa/riscv-sbi-doc) 提供了正式定义，我将在此对其进行简化解释，以帮助更直观地理解。

RISC-V 的 SBI 规范定义了 RISC-V 软件栈底层的软件层。这非常类似于 BIOS，后者通常是机器上运行的第一段软件。你可能看到过一些从零开始开发简单内核的指南，它们通常涉及与我们在[RISC-V裸机编程指南](https://popovicu.com/posts/bare-metal-programming-risc-v)中所做的类似操作，带有一点不同——它们往往依赖于一些预先存在的软件来完成某些 I/O 操作。与我们之前的指南相似之处在于，它们也仔细对齐了第一条指令的地址，以确保处理器的执行流程按预期进行，并且简单的内核能够在正确的时间接管。然而，我通常在这些简短的指南中观察到，目标通常是向**VGA屏幕**打印类似‘Hello world’的东西。最后这一点看起来相当复杂，事实上确实如此。

那么，如何轻松地向 VGA 打印内容呢？答案是 BIOS 在这里帮助我们完成最基本的 I/O 操作，例如向屏幕上打印字符，因此它被称为 **B**asic **I**nput **O**utput **S**ystem（基本输入输出系统）！请注意裸机编程指南的开头部分：我们是在*没有*依赖于机器上*任何*现有软件的情况下与用户交互的（严格来说，这并不完全正确，我们仍然通过了零阶段引导加载程序，但我们并不依赖于它的任何结果，也无法真正控制它；它是系统中硬编码的）。如果我们要在 VGA 屏幕上打印内容，而不是通过 UART 发送字符，我们将需要做的不仅仅是向某个地址发送 ASCII 码。VGA 需要通过发送多个值来设置显示设备，配置不同的参数等。这是一个相当复杂的操作。

那么 BIOS 通常如何帮助处理这些任务？主要的概念是，无论最终在机器上安装了什么操作系统，它都需要一些基本功能，比如向 VGA 屏幕打印信息。因此，机器可以将这些标准操作预先集成到系统中，供任何操作系统使用。从概念上讲，我们可以将这些过程视为我们应用程序编写时所依赖的日常库。

此外，如果操作系统是针对这样的“库”编写的，它将自动变得更加可移植。“库”应该包含所有底层细节，例如“输出到 UART 意味着写入 `0x10000000`”（在 QEMU 的 `riscv64 virt` 虚拟机中），与“输出到 UART 意味着写入 `0x12345678`”相比，操作系统只需要调用“输出到 UART”的过程，而这个“库”将知道如何与硬件进行交互。

### 精妙的抽象

这些讨论归根结底是编程中从第一天就开始使用的一个非常简单的概念：我们在编程中应用了**抽象层**。想象一下，一个 Python 函数执行类似“将本地文件发送到电子邮件地址”的操作。从高层视角看，我们只是简单地调用一个函数 `send_file_to_email(file, email)`，然后底层的库会打开网络连接并开始传输字节。这可能是另一个 Python 库。在某个时刻，这个操作可能会下移到软件栈的底层，Python 库将依赖于用 C 语言编写的 Python 运行时，后者通过系统调用操作系统来执行核心操作（例如打开网络套接字）。操作系统的深处有一个网络驱动程序，它知道应该将字节发送到地址空间中的哪个地址，以便将字节通过网络传输出去。这里的主要概念是，我们已经建立了一种隐藏操作复杂性的方式，通过将它们委托给软件栈的底层来处理。我们并不是从原子部分构建更大的系统，而是从“分子”构建。

如果我们将复杂性委托给底层库，这可能只是一个函数调用。然而，一旦需要将复杂性委托给操作系统及以下层次，这就会通过一个**二进制接口**来实现。

### 二进制接口

自从计算机开始广泛使用以来，`x86` 架构一直是我们使用的台式机或笔记本电脑的主导架构。近年来，情况发生了很大变化，其他架构开始进入我们的视野，但让我们暂时集中于 `x86`。那么，是什么使为 Linux 构建的应用程序与 Windows 的应用程序不兼容呢？如果它是为 `x86` 编写的，并且 Linux 和 Windows 都运行在 `x86` 上，那么差异究竟在哪里？CPU 指令并没有因平台不同而改变，那么原因是什么呢？答案是**应用程序和操作系统之间的接口**。用户软件与操作系统之间的这个特定链接称为**应用二进制接口（ABI）**。ABI 只是一个定义，规定了如何从用户应用程序调用操作系统的服务。

因此，当我们说“这个软件是为平台 X 编写的”时，仅仅说 X 是 `x86` 或 `RISC-V` 还不够，我们必须明确说 `x86/Linux` 或 `x86/Windows` 或 `RISC-V Linux` 等。如果涉及动态链接，平台定义甚至可能更加复杂，但我们暂且不讨论这个问题。

让我们来看一个用汇编语言为 `x86/Linux` 编写的程序的简单例子，该程序只是将‘Hello’字符串打印到标准输出。

```
.global _start

.section .text

_start: mov $4, %eax ; 4 是 'write' 系统调用的代码
        mov $1, %ebx ; 我们正在写入文件 1，即 '标准输出'
        mov $message, %ecx ; 我们要打印的数据位于 message 符号定义的地址
        mov $5, %edx ; 我们要打印的数据长度是 5
        int $0x80 ; 调用系统调用，即请求内核将数据打印到标准输出

        mov $1, %eax ; 1 是 'exit' 系统调用的代码
        mov $0, %ebx ; 0 是进程的返回码
        int $0x80 ; 调用系统调用，即请求内核关闭该进程

.section .data
message: .ascii "Hello"
```

用以下命令汇编该程序：

```
as -o syscall.o syscall.s
```

链接：

```
ld -o syscall syscall.o
```

运行：

```
./syscall
```

你应该会看到输出“Hello”。如果你使用的是 Bash，并且想要检查进程的返回码，可以运行：

```
echo $?
```

你应该会看到 `0`。

*提示：如果你没有 `x86/Linux` 机器，但想尝试上面的例子，可以通过一个模拟 `x86` 系统的 JavaScript 虚拟机来实现[这里](https://bellard.org/jslinux/)；这是一个非常酷的网站！*

这样我们就得到了一个在 `x86` 机器上运行并在 Linux 内核下打印消息的程序。**没有使用 C 标准库**。最终生成的 `ELF` 二进制文件应该能够在 Linux 上运行，唯一的依赖条件是它运行在正确的平台上。

现在回到问题本身，是什么可能使这个二进制文件与 Windows 不兼容？**不同的操作系统以不同的方式编码系统调用（例如，写入操作的代码在 Linux 是 4，而在 Windows 可能是 123，或者参数通过不同的 CPU 寄存器传递）。** 现在你对如何直接与内核交互有了更好的理解，而不依赖标准库（尽管你几乎不会想这么做）。这意味着你已经揭示了软件执行诸如打开文件、分配内存、发送信号等操作的那一层。C 标准库可以看作是一个封装器，它隐藏了通过 `int` 指令调用软件中断与内核通信的复杂性，而是使其看起来像一个正常的 C 函数调用，但实际上它的底层机制就是这样。公平地说，库做的远不止这些，但对于本文的目的，可以简单地将其视为一个封装器。

现在在 RISC-V 的世界里，情况也是类似的：用户应用程序通过软件中断的 CPU 指令与内核交互，并通过 CPU 寄存器传递参数。而内核则通过 SBI 来调用它的服务，**执行的方式完全相同**！不同之处在于，这一层的逻辑调用被称为 **SBI**，而不是 **ABI**。可以把它理解为，这不是应用程序在工作，而是应用程序的**监督者**在工作。名称不同，概念却是完全一致的。

## 使用 OpenSBI 的实际示例

到此为止，我们已经明确了，SBI 和 ABI 类似，都是调用软件栈底层功能的一种方式。同时我们也明确了，SBI 处于 RISC-V 机器的软件栈的最底层，并在最具特权的 M 模式下运行。现在让我们进一步补充一些细节。

现在应该可以理解为什么 QEMU 开发者选择使用 `-bios` 标志来接受 SBI 软件镜像（因为其功能与 BIOS 基本相同）。需要提醒的是，`-bios` 标志应该指向一个 `ELF` 文件，该文件将从地址 `0x80000000` 开始布局 SBI 软件。

让我们在仅加载 OpenSBI 的情况下启动 QEMU 的虚拟机，看看会发生什么。我们不需要传递任何参数给 QEMU，因为默认情况下它会将 OpenSBI 加载到 `0x80000000`。

```
qemu-system-riscv64 -machine virt
```

这是输出（在串行端口上，不是 VGA）：

```
OpenSBI v0.8
   ____                    _____ ____ _____
  / __ \                  / ____|  _ \_   _|
 | |  | |_ __   ___ _ __ | (___ | |_) || |
 | |  | | '_ \ / _ \ '_ \ \___ \|  _ < | |
 | |__| | |_) |  __/ | | |____) | |_) || |_
  \____/| .__/ \___|_| |_|_____/|____/_____|
        | |
        |_|

Platform Name       : riscv-virtio,qemu
Platform Features   : timer,mfdeleg
Platform HART Count : 1
Boot HART ID        : 0
Boot HART ISA       : rv64imafdcsu
BOOT HART Features  : pmp,scounteren,mcounteren,time
BOOT HART PMP Count : 16
Firmware Base       : 0x80000000
Firmware Size       : 96 KB
Runtime SBI Version : 0.2

MIDELEG : 0x0000000000000222
MEDELEG : 0x000000000000b109
PMP0    : 0x0000000080000000-0x000000008001ffff (A)
PMP1    : 0x0000000000000000-0xffffffffffffffff (A,R,W,X)
```

虚拟机就这样一直运行下去，可能是因为默认情况下它被设置为这样运行，因为没有其他软件传递给 QEMU 来在 OpenSBI 之后接管控制。此时看来一切正常，OpenSBI 已经成功设置（其输出也确认它位于 `0x80000000`）。

接下来我们如何继续扩展软件栈呢？新的一层可以是操作系统内核，因此类似于我们之前构建的包含指令的 `ELF` 文件，它被放置在 `0x80000000`，我们将构建另一个 `ELF` 文件，供 QEMU 加载到内存中，但这次指令将加载到另一个地址，因为从 `0x80000000` 开始的部分已经被 OpenSBI 占用。

那么，我们应该将这个“虚拟内核”加载到哪个地址呢？

## 在 SBI 之后启动操作系统内核并调用 OpenSBI

当我们加载 BIOS/SBI/无论你想怎么称呼它时，这个地址基本上是“写死”在机器逻辑中的。最初的几条指令是零阶段引导加载程序（ZSBL），它的最后一条指令跳转到硬编码的地址 `0x80000000`。正如我们之前提到的，这是我们所使用的平台的不可更改的事实，它就是这样工作的。然而，这就是它目前所硬编码的全部内容：它只是硬编码了你必须从 `0x80000000` 开始，而我们现在已经在这里放置了 OpenSBI，那么 OpenSBI 接下来将带我们到哪里呢？

这时，**ZSBL** 的重要性再次出现了，现在它如何在执行跳转到 `0x80000000` 之前初始化这些寄存器真的很关键。ZSBL 实际上做了两件事：

1. 确保在 OpenSBI 初始化后运行的软件可以运行，这基本上是操作系统内核的引导加载程序，或者直接是内核本身（这就是你在 QEMU 指南中启动 Linux 时通常看到的，跳过引导加载程序，内存中直接加载内核）。
2. 跳转到 OpenSBI。

我们已经详细讨论了第二点，现在让我们深入探讨它如何完成第1点。

### ZSBL 中究竟发生了什么？

我们之前提到 ZSBL 的执行从地址 `0x1000` 开始。让我们通过 QEMU 跟踪执行情况，看看发生了什么。为此，我们将在 QEMU 命令行中添加两个标志：`-s` 和 `-S`。这些标志确保 QEMU 暴露一个 `gdb` 调试端口，并且虚拟机在创建时立即暂停，等待我们通过 `gdb` 手动驱动它。

我们现在开始这个逆向工程的过程。我们首先启动 QEMU：

```
qemu-system-riscv64 -machine virt -s -S
```

在另一个终端中，我们连接到嵌入在 QEMU 中的 `gdb` 服务器，以便控制虚拟机的执行。我是在 `x86` 机器上执行的，所以我会使用 `gdb-multiarch` 来进行跨平台的 `riscv` 调试。在这个新的终端中，我运行以下命令：

```
gdb-multiarch
```

在连接到虚拟机之前，我想先设置几件事：

```
set architecture riscv:rv64
```

这行命令的作用显而易见。接下来，我希望每次执行一条指令时都能在终端中打印出实际的运行指令：

```
set disassemble-next-line on
```

现在是时候连接到 QEMU 的 `gdb` 服务器了（端口 `1234` 基本上是 QEMU 的默认设置，虽然可能可以通过 `-s` 标志进行某种方式的配置；不过我从未尝试过，也认为你不需要改变这个行为）。

```
target remote :1234
```

此时，`gdb` 正在等待我们，它停在了地址 `0x1000`，这是系统上电后执行的第一条指令的地址。我们将使用 `si` 多次单步执行指令，直到我们跳转到位于 `0x80000000` 的 SBI。

```
(gdb) target remote:1234
Remote debugging using :1234
warning: No executable has been specified and target does not support
determining executable automatically.  Try using the "file" command.
0x0000000000001000 in ?? ()
=> 0x0000000000001000: 97 02 00 00 auipc t0,0x0
(gdb) si
0x0000000000001004 in ?? ()
=> 0x0000000000001004: 13 86 82 02 addi a2,t0,40
(gdb) si
0x0000000000001008 in ?? ()
=> 0x0000000000001008: 73 25 40 f1 csrr a0,mhartid
(gdb) si
0x000000000000100c in ?? ()
=> 0x000000000000100c: 83 b5 02 02 ld a1,32(t0)
(gdb) si
0x0000000000001010 in ?? ()
=> 0x0000000000001010: 83 b2 82 01 ld t0,24(t0)
(gdb) si
0x0000000000001014 in ?? ()
=> 0x0000000000001014: 67 80 02 00 jr t0
(gdb) si
0x0000000080000000 in ?? ()
=> 0x0000000080000000: 33 04 05 00 add s0,a0,zero
```

在 ZSBL 中只有 6 条指令，之后控制权就交给了 OpenSBI，包含跳转指令。然而，这几条指令的意义是什么呢？

事实证明，这也是 SBI 规范的一部分，它是启动序列的一部分。然而，对于 OpenSBI，有三种不同的实现方式，在我们深入讨论 ZSBL 之后发生的事情之前，先来看看这些不同的实现方式。

### OpenSBI 的三种实现方式

你可以通过三种不同的方式构建 OpenSBI：

1. `FW_PAYLOAD`（[官方文档](https://github.com/riscv-software-src/opensbi/blob/master/docs/firmware/fw_payload.md)）
2. `FW_JUMP`（[官方文档](https://github.com/riscv-software-src/opensbi/blob/master/docs/firmware/fw_jump.md)）
3. `FW_DYNAMIC`（[官方文档](https://github.com/riscv-software-src/opensbi/blob/master/docs/firmware/fw_dynamic.md)）

#### `FW_PAYLOAD`

这个方式概念上最容易理解。构建这个版本的 OpenSBI 时，你会将 `make` 工具指向你的内核或“你希望在 OpenSBI 之后运行的任何软件”镜像，然后你将获得一个可以直接加载的单一二进制文件。在 QEMU 的虚拟机中，地址为 `0x80000000`。据我了解，可以调整你的软件在内存中与 OpenSBI blob 相关的位置，但为了简化起见，我们可以认为 OpenSBI 和你的软件被组合在一起成一个单一的 blob，一旦 OpenSBI 初始化完成，下一条指令就是你的软件（基本上 OpenSBI 之后直接过渡到你的软件）。

实现这个的方式是：

1. 确保在 `make` 过程中设置 `FW_PAYLOAD=y`，这将确保生成一个名为 `fw_payload` 的文件。
2. 在 `make` 过程中设置 `FW_PAYLOAD_PATH`，指向你希望在 OpenSBI 之后运行的软件。

根据上面链接的文档，如果你省略了第二个标志，OpenSBI 会与一个非常简单的无限循环软件一起拼接。这就解释了为什么当我们只启动 QEMU 而不加任何标志时，机器基本上只是原地打转——OpenSBI 很可能就是这样构建的（因为你不能继续执行随机的内存内容），它只是忙于原地等待。

这种方式的优点在于，现在你有了一个单一的、拼接的、整体的软件镜像，可以加载到你的机器中。你不需要处理多个独立的软件块，只需处理一个整体。如果你的软件构建过程比较简单，你甚至可能最终找到一种非常简单的方式来管理目标机器上的所有软件，同时还能享受 OpenSBI 为你提供的一些工作便利。

缺点是，现在你需要负责将所有东西拼接在一起，包括 OpenSBI。更糟的是，如果机器已经有了 OpenSBI，例如，烧录在某个 ROM 中，它已经有了启动 OpenSBI 的能力，重复加载 OpenSBI 可能会导致问题。

#### `FW_JUMP`

这种方式也相对简单：你基本上将 OpenSBI 之后的软件地址硬编码。类似于上述情况，也需要两步。

1. 确保在 `make` 过程中设置 `FW_JUMP=y`，这将生成一个名为 `fw_jump` 的文件。
2. 在 `make` 过程中设置 `FW_JUMP_ADDR`，指向 OpenSBI 完成后跳转的地址。

这与前一种情况非常相似，只是跳转地址是硬编码的。在这种情况下，你仍然需要负责构建 OpenSBI 镜像，但它易于重建，并且可以针对不同的机器指向不同的地址（例如，具有不同内存布局的机器）。

#### `FW_DYNAMIC`

这是最通用的方式，因此我们最后讨论这一种。在这种方式中，ZSBL 中寄存器设置的重要性开始显现出来。

在这种模式下，OpenSBI 之前的启动阶段负责向 OpenSBI 传递一些指针。在这个例子中，当然是 ZSBL。如果我们仔细观察，会看到它修改了寄存器 `a2`。

在这里，我鼓励读者阅读[这篇文章](https://embeddedinn.xyz/articles/tutorial/RISCV-Uncovering-the-Mysteries-of-Linux-Boot-on-RISC-V-QEMU-Machines/#the-zero-stage-bootloader-zsbl)中的 ZSBL 部分。整篇文章都很棒，只是我一开始觉得它有点难以理解，所以可以把本文看作是理解那篇文章的预备知识，值得花时间去阅读。

简单来说，ZSBL 设置 `a2` 寄存器的意义是什么？**它指向一个 `struct fw_dynamic_info` 结构**，这为动态 OpenSBI 提供了继续启动过程的方式！实际上，这个结构中的一项数据是 OpenSBI 之后运行的软件的地址。一个很好的问题是：在真实机器上，谁来填充这个结构？根据我们将看到的内容，显然 QEMU 会将该内容硬编码到内存中，这一逻辑并不是 ZSBL 的一部分，但我可以想象，在某些设备上 ZSBL 实际上会填充这个结构并传递给 OpenSBI。

来自西部数据的一名工程师的[演讲](https://riscv.org/wp-content/uploads/2019/06/13.30-RISCV_OpenSBI_Deep_Dive_v5.pdf)的第17页（他可能是 OpenSBI 的核心贡献者）概述了这个 `struct` 的内容：

1. 魔术数
2. 版本号
3. 下一个地址
4. 下一个模式
5. 选项

所有这些都是无符号长整型（我猜是 64 位，8 字节？）。

#### 探索 `fw_dynamic_info` 结构

到此为止，我们来一个快速的绕道，确保我们在讨论相同的版本。我们检查一下 OpenSBI 的版本，因为不同系统的 QEMU 可能带有不同版本的 OpenSBI。构建 OpenSBI 源码非常简单，让我们快速进行。首先，我们需要克隆 Git 仓库（本文写作时是 2023 年 9 月 10 日；如果你想要完全可重复的构建，可以使用这一天的提交版本）：

```
git clone https://github.com/riscv-software-src/opensbi.git
cd opensbi
make ARCH=riscv CROSS_COMPILE=riscv64-linux-gnu- PLATFORM=generic
```

构建速度应该非常快，且占用资源较少。我们感兴趣的输出文件是 `build/platform/generic/firmware/fw_dynamic.bin`。我们将通过 QEMU 的 `-bios` 标志来传递这个文件。从刚刚克隆的 `opensbi` 文件夹中启动 QEMU：

```
qemu-system-riscv64 -machine virt -s -S -bios build/platform/generic/firmware/fw_dynamic.bin
```

在 `gdb` 中执行几次 `si` 之后，我们回到之前的状态。让我们通过 `gdb` 查看 QEMU 的内存，以了解 ZSBL 结束时的情况。在 ZSBL 的最后一条指令处，我们查看寄存器转储（使用 `i r` 来查看）：

```
=> 0x0000000080000000: 33 04 05 00 add s0,a0,zero
(gdb) i r
```

结果显示 `a2` 指向 `0x1028`。正如我们所说的，让我们通过 `gdb` 检查该内存。我们让它读取从 `0x1028` 开始的 10 个连续的 8 字节值，并以十六进制格式显示。

```
(gdb) x/10xg 0x1028
```

`g` 标志用于以 8 字节的“巨大”块打印内存内容。

```
(gdb) x/10xg 0x1028
0x1028: 0x000000004942534f 0x0000000000000002
0x1038: 0x0000000000000000 0x0000000000000001
0x1048: 0x0000000000000000 0x0000000000000000
0x1058: 0x0000000000000000 0x0000000000000000
0x1068: 0x0000000000000000 0x0000000000000000
```

这大致符合[Vysakh的文章](https://embeddedinn.xyz/articles/tutorial/RISCV-Uncovering-the-Mysteries-of-Linux-Boot-on-RISC-V-QEMU-Machines/#the-zero-stage-bootloader-zsbl)中的描述。我们确实看到了文章中提到的魔术数，后面跟着的是 `0x02` 版本信息。接下来应该是下一个跳转的地址，但这里全是零……这有点奇怪，不过我们继续看。下一个值是 `0x01`，根据文章，它应该对应于执行的下一个模式，也就是 `S` 模式。这是正确的，我们从运行 SBI 的 `M` 模式切换到运行操作系统内核引导程序（或直接是内核）的 `S` 模式。为什么下一个跳转地址全是零呢？在这个时候，我决定让 QEMU 不再受 `gdb` 干扰而继续运行。我在 `gdb` 中运行以下命令：

```
continue
```

现在一切都停止了，但由于我正在运行更新版本的 OpenSBI，所以 UART 上输出了更新的内容：

```
OpenSBI v1.3-54-g901d3d7
   ____                    _____ ____ _____
  / __ \                  / ____|  _ \_   _|
 | |  | |_ __   ___ _ __ | (___ | |_) || |
 | |  | | '_ \ / _ \ '_ \ \___ \|  _ < | |
 | |__| | |_) |  __/ | | |____) | |_) || |_
  \____/| .__/ \___|_| |_|_____/|____/_____|
        | |
        |_|

Platform Name             : riscv-virtio,qemu
Platform Features         : medeleg
Platform HART Count       : 1
Platform IPI Device       : aclint-mswi
Platform Timer Device     : aclint-mtimer @ 10000000Hz
Platform Console Device   : uart8250
Platform HSM Device       : ---
Platform PMU Device       : ---
Platform Reboot Device    : syscon-reboot
Platform Shutdown Device  : syscon-poweroff
Platform Suspend Device   : ---
Platform CPPC Device      : ---
Firmware Base             : 0x80000000
Firmware Size             : 322 KB
Firmware RW Offset        : 0x40000
Firmware RW Size          : 66 KB
Firmware Heap Offset      : 0x48000
Firmware Heap Size        : 34 KB (total), 2 KB (reserved), 9 KB (used), 22 KB (free)
Firmware Scratch Size     : 4096 B (total), 768 B (used), 3328 B (free)
Runtime SBI Version       : 1.0

Domain0 Name              : root
Domain0 Boot HART         : 0
Domain0 HARTs             : 0*
Domain0 Region00          : 0x0000000002000000-0x000000000200ffff M: (I,R,W) S/U: ()
Domain0 Region01          : 0x0000000080040000-0x000000008005ffff M: (R,W) S/U: ()
Domain0 Region02          : 0x0000000080000000-0x000000008003ffff M: (R,X) S/U: ()
Domain0 Region03          : 0x0000000000000000-0xffffffffffffffff M: () S/U: (R,W,X)
Domain0 Next Address      : 0x0000000000000000
Domain0 Next Arg1         : 0x0000000087e00000
Domain0 Next Mode         : S-mode
Domain0 SysReset          : yes
Domain0 SysSuspend        : yes

Boot HART ID              : 0
Boot HART Domain          : root
Boot HART Priv Version    : v1.10
Boot HART Base ISA        : rv64imafdc
Boot HART ISA Extensions  : zicntr
Boot HART PMP Count       : 16
Boot HART PMP Granularity : 4
Boot HART PMP Address Bits: 54
Boot HART MHPM Info       : 0 (0x00000000)
Boot HART MIDELEG         : 0x0000000000000222
Boot HART MEDELEG         : 0x000000000000b109
```

这与我们之前看到的相符，接下来的地址全是零……这很奇怪，不可能是对的。我现在运行 QEMU 时没有让它暂停，而是让它直接运行，并异步连接 `gdb`。我就不详细说明了，但在这种“实时运行”中检查寄存器时，确实显示没有任何指令在 `0x0000000000000000` 区域执行。CPU 似乎在某个其他地址上循环。

这可能与我实际上没有向 QEMU 传递除了 OpenSBI 以外的其他软件有关，所以这可能导致了问题。QEMU 可能将该结构填充为全零，OpenSBI 识别到这是一个非法的边缘情况，因此它一直在 OpenSBI 中无休止地循环——这是我的推测。

我们如何传递一些除 OpenSBI 以外的软件呢？**与传递 OpenSBI 的方式相同，只是使用不同的标志！**这次我们使用 `-kernel` QEMU 标志。我们将如何构建这个软件？与之前的“伪 BIOS”构建方式相同，只是将其映射到不同的内存位置。让我们试着把它加载到 `0x80200000`。

#### 构建一个“无限循环的伪内核”

我们的操作系统内核将无限循环。它将在 `0x80200000` 处包含一条跳转指令，永远保持在那里。以下是汇编源代码：

```
 .global _start
 .section .text.kernel

_start: j _start
```

链接脚本如下：

```
MEMORY {
  kernel_space (rwx) : ORIGIN = 0x80200000, LENGTH = 128
}

SECTIONS {
  .text : {
    infinite_loop.o(.text.kernel)
  } > kernel_space
}
```

*有关如何使用这些文件构建可加载到 QEMU 的 `ELF` 映像的详细信息，请参阅原始的裸机编程文章。*

一旦我们构建完成，我们会得到一个名为 `infinite_loop` 的 `ELF` 文件，可以作为我们的伪内核。现在我们运行 QEMU：

```
qemu-system-riscv64 -machine virt -s -S -bios build/platform/generic/firmware/fw_dynamic.bin -kernel ~/work/github_demo/risc-v-bare-metal-fake-kernel/infinite_loop
```

再次，我连接 `gdb` 并用 `si` 一步步执行到 ZSBL 结束。现在，当我查看臭名昭著的 `0x1028` 结构时，情况好多了，这证实了 QEMU 之前奇怪地填充了那个结构的理论。

```
=> 0x0000000080000000: 33 04 05 00 add s0,a0,zero
(gdb) x/10xg 0x1028
0x1028: 0x000000004942534f 0x0000000000000002
0x1038: 0x0000000080200000 0x0000000000000001
0x1048: 0x0000000000000000 0x0000000000000000
0x1058: 0x0000000000000000 0x0000000000000000
0x1068: 0x0000000000000000 0x0000000000000000
```

我们现在看到该结构中的新地址已经填充，正如预期的那样。这也反映在 UART 上的 OpenSBI 输出中。让我们通过 `gdb` 继续进入我们的伪内核，看看那里一切是否正常。

```
(gdb) break *0x080200000
Breakpoint 1 at 0x80200000
(gdb) continue
Continuing.

Breakpoint 1, 0x0000000080200000 in ?? ()
=> 0x0000000080200000: 6f 00 00 00 j 0x80200000
```

一切看起来都很正常。让我们总结一下：

1. ZSBL 是上电后运行的第一段代码。它初始化了一些寄存器，关键的寄存器是 `a2`，它指向一个 `fw_dynamic_info` 结构，该结构包含 OpenSBI 动态模式操作的关键信息。在 QEMU 的情况下，这个结构在上电时被虚拟化引擎神奇地填充，但在现实中，这**很可能**是 ZSBL 的工作。不管怎样，OpenSBI 现在知道完成后要做什么。
2. OpenSBI 提供了一个基于中断的接口，供上层软件（大概是操作系统内核引导程序或内核本身）调用。这个接口称为 SBI，它在概念上与操作系统之上的应用程序软件的 ABI 相同。
3. 我们将内核映像作为另一个 ELF 文件传递给 QEMU，加载到内存的另一个区域。QEMU 填充该结构，使得 OpenSBI 能够将控制权传递到那里，并且在切换到那里之前，它会进入 `S` 模式。

### 有意略过的细节

ZSBL 还修改了 `a0` 和 `a1` 寄存器。

`a0` 与 RISC-V 的 `hart` 有关，但我们不深入探讨这些细节，因为它们对本文的其余部分没有太大相关性。此外，这个引导过程中的这一步似乎并不重要，参见[Github评论](https://github.com/riscv-software-src/opensbi/issues/170#issuecomment-642679348)。

`a1` 是一个有趣的指针，它指向内存中的**设备树**数据结构。对于本文的其余部分，这个数据结构并不相关，因此我们可以忽略它。然而，对于像 Linux 这样的真实内核，设备树非常有用。Linux 能够从内存中扫描设备树，并了解它运行的机器的结构，而不是针对每种硬件组合编写大量的 `if/else` 分支。你可以从[维基百科文章](https://en.wikipedia.org/wiki/Devicetree#Linux)中了解它在 Linux 中的使用情况。不过，正如所提到的，我们不会在本文中讨论设备树的细节。

## Hello World 伪内核

现在我们已经掌握了足够的知识，可以编写一个伪操作系统内核，它只是在 UART 设备上打印“Hello world”。这个功能与我们在前一篇文章中看到的裸机程序没有太大区别，但我们到达目标的方式有显著不同。这次，我们将通过 SBI 调用来打印到 UART，而不是直接与 UART 设备交互（我们将这项工作委托给一个更底层的特权软件层）。即使在像“Hello world”这样简单的例子中，这也可能带来重大影响：**我们将与 UART 硬件交互的责任委托给 SBI 层，从而实现跨不同符合 SBI 接口的机器的可移植性**。

如何调用 RISC-V 的 SBI 层？概念上，这与在 x86 Linux 上调用标准输出打印非常相似——我们需要填充一些寄存器并触发软件中断/陷阱，将控制传递到软件栈的底层 OpenSBI。OpenSBI 在 SBI 层提供了许多服务，其中许多服务对于开发可移植的操作系统内核非常有用，例如与定时器的交互（这与时间切片和使多个线程共享同一 CPU 核心相关）。有关 SBI 层暴露的全部功能列表，请查看[这里](https://github.com/riscv-non-isa/riscv-sbi-doc/blob/master/riscv-sbi.adoc)。

在本指南中，我们将专注于[调试控制台](https://github.com/riscv-non-isa/riscv-sbi-doc/blob/master/src/ext-debug-console.adoc)功能，即通过 SBI 向 UART 写入内容。让我们开始编写代码吧！

首先，我们需要知道如何通过寄存器编码我们希望 OpenSBI 执行的功能。这在[这里](https://github.com/riscv-non-isa/riscv-sbi-doc/blob/master/src/binary-encoding.adoc)有详细记录。简而言之，SBI 功能被分组到“扩展”中。寄存器 `a7` 包含扩展 ID (EID)，而 `a6` 编码该扩展中的功能 ID (FID)。然后通过 `a0`、`a1`、`a2` 等寄存器传递参数。

要打印到控制台，我们需要的 EID 是 `0x4442434E`（一个相当有趣的值），而 FID 则是 `0x00`。

这次，和之前裸机编程指南中的逐字打印不同，我们将一次性调用打印功能。毕竟，我们应该从 SBI 层提供的高级功能中受益。因此，我们的二进制文件应该将输出字符串存储在内存中的某个地方，理想情况下，我们希望通过调用 SBI 从该地址开始打印。我们将这样做：

```asm
        .global _start
        .section .text.kernel

_start: li a7, 0x4442434E
        li a6, 0x00
        li a0, 12
        lla a1, debug_string
        li a2, 0
        ecall

loop:   j loop

        .section .rodata
debug_string:
        .string "Hello world\n"
```

这里有几个需要注意的地方：

1. 我们在这里使用了 PC 相对寻址来处理输出字符串。需要提醒的是，内核存储在一个非常大的无符号整数表示的地址上。这个值太大，无法用任何 RISC-V 32 位指令字编码。这不是问题，我们只需使用一小段 `AUIPC` 和 `ADDI` 指令即可到达那里（有关更多信息，请查看[这篇文章](https://michaeljclark.github.io/asm.html)）。如果你不理解这一点，请务必复习不同的内存寻址模式及其差异：这是任何裸机编程的关键。由于这是一个常见的模式，RISC-V 汇编器提供了一个**伪指令** `LLA`，我们在这里使用了它。
2. SBI 需要将要打印的字符串指针分为两部分传递。可以看到其中一部分是 0。我不太确定为什么需要这样，但这是 API 设计的要求。

### SBI 调用涉及的寄存器

1. `a7` 标识 SBI 扩展。
2. `a6` 标识扩展中的功能（在本例中是调试控制台扩展）。
3. `a0` 包含需要发送到调试控制台输出的字符串长度。
4. `a1` 和 `a2` 结合在一起，形成指向需要打印的字符串地址的 64 位指针。

SBI 调用通过 `ecall` 指令触发，它会激活 CPU 陷阱。此时，OpenSBI 接管并通过 UART 输出数据，方式与我们在最初的裸机编程指南中所做的完全相同。如果你想知道为什么简单的 `ecall` 调用会将控制权交给 OpenSBI，这是因为 OpenSBI 设置了陷阱处理机制，当我们的内核进入陷阱时，程序计数器会跳转到 OpenSBI 的软件部分。这些细节超出了本文的范围，但我们可能会在其他文章中详细讨论。

现在，只需查看 QEMU 的串行端口，确认“Hello world”已正确打印：

```
qemu-system-riscv64 -machine virt -s -S -bios build/platform/generic/firmware/fw_dynamic.bin -kernel ~/work/github_demo/risc-v-bare-metal-fake-kernel/hello_world_kernel
```

输出应该如下所示：

```
OpenSBI v1.3-54-g901d3d7
   ____                    _____ ____ _____
  / __ \                  / ____|  _ \_   _|
 | |  | |_ __   ___ _ __ | (___ | |_) || |
 | |  | | '_ \ / _ \ '_ \ \___ \|  _ < | |
 | |__| | |_) |  __/ | | |____) | |_) || |_
  \____/| .__/ \___|_| |_|_____/|____/_____|
        | |
        |_|

Platform Name             : riscv-virtio,qemu
Hello world
```

作为练习，我建议通过 `gdb` 探索[基础扩展（`0x10`）](https://github.com/riscv-non-isa/riscv-sbi-doc/blob/master/src/ext-base.adoc)，以调查你构建的 QEMU 机器和 OpenSBI 能提供哪些功能。

## 结论

最终，我们得到了一个完全可移植的伪内核，它能够通过 UART 打印“Hello world”！虽然看起来不特别，但其背后的概念非常强大。你可以在支持调试控制台扩展的不同 RISC-V 64 位机器上直接使用相同的内核映像，无需重新构建。

事实上，我在这里玩了个小技巧 :) 让我建议你从源代码构建 OpenSBI 的原因之一是，某些 Linux 发行版包管理器提供的 QEMU 版本不支持调试控制台扩展（它们的版本较旧）。这正是我在 Debian 的 QEMU 中默认使用的 OpenSBI 遇到的情况。

最后，提醒大家我们主要关注的是带有 RISC-V 核心的 QEMU `virt` 机器，本文的所有细节都与此相关。不过，我希望读者已经了解了足够的启动顺序和裸机编程概念，以便将这些知识轻松应用到现实中的特定场景。

在接下来的文章中，我们将讨论如何进一步引导完整的 Linux 内核，并逐步扩展，直到我们实现一个能够处理键盘、鼠标、屏幕和以太网网络的 Linux 部署。

希望你喜欢这篇长篇文章！

## 代码指引

如果你不想复制粘贴代码，它可以在[这个 GitHub 仓库](https://github.com/popovicu/risc-v-bare-metal-fake-kernel)中找到。
