# RISC-V: Makefile 与 GDB/QEMU 调试 (C 语言)

本节为 RISC-V 裸机编程提供了一个逐步构建的 Makefile 教程，帮助你使用 `riscv64-unknown-elf-gcc` 进行编译和链接，同时通过 QEMU 启动虚拟机，并利用 GDB 进行调试。该教程适用于裸机编程环境，并逐步引入编译、链接、启动和调试的各个步骤。

### 项目结构

假设你的项目目录如下：

```
riscv-baremetal/
├── Makefile
├── hello.c
├── start.s
├── linker.ld
```

- `hello.c`：C 语言编写的主程序。
- `start.s`：汇编启动文件，负责启动 CPU 并跳转到 C 代码。
- `linker.ld`：链接器脚本，控制程序的内存布局。

通过这些文件，我们将使用 `riscv64-unknown-elf-gcc` 编译和链接，并使用 QEMU 进行模拟调试。

## 第一步：编译和链接

首先，我们使用 `riscv64-unknown-elf-gcc` 交叉编译工具链来编译项目中的源文件，并将它们链接为一个可执行的裸机程序 `hello.elf`。在这个步骤中，我们将构建最基础的编译规则。

### 编译命令说明

为了编译和链接 `hello.c` 和 `start.s`，我们需要在命令行中使用以下命令：

```bash
riscv64-unknown-elf-gcc -g -nostartfiles -nostdlib -T linker.ld -mcmodel=medany -o hello.elf start.s hello.c
```

**参数解释**：

- **`-g`**：生成调试信息，使得我们可以在 GDB 中查看源代码级的调试信息。
- **`-nostartfiles`**：告诉编译器不要链接标准的启动文件。标准启动文件通常用于初始化环境（如 C 库），但裸机编程中我们通常自己编写启动代码，所以禁用它。
- **`-nostdlib`**：禁用标准库（如 `libc`），因为在裸机环境中，标准库并不可用，我们必须自己管理所有内存和 I/O 操作。
- **`-T linker.ld`**：指定链接器脚本，控制程序在内存中的布局。这个脚本将告诉编译器如何将各个段（如代码段 `.text` 和数据段 `.data`）布局到内存的正确位置。
- **`-mcmodel=medany`**：这是 RISC-V 的内存模型，用于裸机编程。它允许指令和数据在较大范围的内存地址中进行操作。
- **`-o hello.elf`**：指定输出文件的名称为 `hello.elf`，这是最终生成的可执行文件。
- **`start.s` 和 `hello.c`**：这两个文件分别为启动汇编代码和主程序。

### 使用 Makefile 管理编译和链接

为了简化编译过程并避免每次手动输入复杂的编译命令，我们可以使用 Makefile 来自动管理编译和链接过程。Makefile 可以帮助我们定义规则，自动化整个构建过程。

## Makefile 第一步：基础编译规则

以下是一个基础的 Makefile，用于编译和链接 `start.s` 和 `hello.c` 生成 `hello.elf`：

```makefile
# 定义工具链和编译器选项
CC = riscv64-unknown-elf-gcc
CFLAGS = -g -nostartfiles -nostdlib -mcmodel=medany
LDFLAGS = -T linker.ld

# 目标文件
TARGET = hello.elf

# 源文件
SRCS = start.s hello.c

# 编译规则
$(TARGET): $(SRCS)
	$(CC) $(CFLAGS) $(LDFLAGS) -o $(TARGET) $(SRCS)

# 伪目标 clean，用于清理生成的文件
.PHONY: clean
clean:
	rm -f $(TARGET)
```

- **`CC`**：定义了我们使用的编译器，这里是 `riscv64-unknown-elf-gcc`，适用于 RISC-V 架构的交叉编译器。
- **`CFLAGS`**：编译选项，包括调试信息（`-g`）、禁用标准启动文件（`-nostartfiles` 和 `-nostdlib`），以及指定 RISC-V 的内存模型（`-mcmodel=medany`）。
- **`LDFLAGS`**：链接选项，指定链接器脚本 `linker.ld`。
- **`TARGET`**：目标文件，即最终生成的可执行文件 `hello.elf`。
- **`SRCS`**：源文件列表，包括汇编启动文件 `start.s` 和主程序 `hello.c`。
- **编译规则**：当 Makefile 检测到目标 `hello.elf` 需要更新时，它会调用 `riscv64-unknown-elf-gcc` 进行编译和链接。
- **`clean` 伪目标**：定义一个伪目标 `clean`，用于清理生成的文件。每当运行 `make clean` 时，它会删除 `hello.elf`，保持目录的整洁。

### 使用步骤：

1. **创建 Makefile**：
   - 在项目的根目录下（与 `hello.c`、`start.s`、`linker.ld` 同级）创建一个名为 `Makefile` 的文件，并将上面的内容粘贴进去。

2. **编译项目**：
   - 打开终端，导航到项目根目录，运行以下命令：
     ```bash
     make
     ```
   - 这会调用 `riscv64-unknown-elf-gcc`，并使用指定的编译选项和链接器脚本生成 `hello.elf`。

3. **查看生成的文件**：
   - 编译完成后，项目目录下会生成 `hello.elf`。你可以运行以下命令查看生成的文件：
     ```bash
     ls -l hello.elf
     ```

4. **清理生成文件**：
   - 为了保持项目目录整洁，您可以使用 `make clean` 来删除生成的 `hello.elf` 文件：
     ```bash
     make clean
     ```

5. **调试信息**：
   - `hello.elf` 文件中包含调试信息（由于使用了 `-g` 选项）。这在后续使用 GDB 进行调试时非常重要，因为它允许我们在调试时查看源代码、设置断点等。

## 第二步：使用 QEMU 启动裸机程序

在编译并生成了 RISC-V 裸机程序 `hello.elf` 之后，接下来我们将使用 QEMU 启动该程序。QEMU 是一个强大的虚拟化和仿真工具，支持多种架构，包括 RISC-V。它能够模拟裸机环境，帮助我们运行和调试 `hello.elf`，无需在物理硬件上执行。

我们将通过 QEMU 启动 `hello.elf` 程序，并使用 GDB 进行调试。QEMU 将模拟 RISC-V CPU，加载并运行该裸机程序，并提供调试接口供 GDB 使用。

### QEMU 启动命令详解

为了启动 `hello.elf` 裸机程序，我们需要执行以下命令：

```bash
qemu-system-riscv64 -machine virt -nographic -kernel hello.elf -s -S
```

**参数说明**：

- **`qemu-system-riscv64`**：启动 QEMU，模拟 RISC-V 64 位架构的系统。
- **`-machine virt`**：虚拟化环境下的 RISC-V 机器类型。这个选项告诉 QEMU 使用“virt”虚拟平台，它是 RISC-V 模拟中常用的一个虚拟开发平台。
- **`-nographic`**：不使用图形界面，将所有输入输出重定向到终端。这使得输出可以直接在终端中显示，适合裸机环境的开发。
- **`-kernel hello.elf`**：加载 `hello.elf` 作为裸机程序。这是我们使用 `riscv64-unknown-elf-gcc` 编译生成的可执行文件，它将被 QEMU 模拟器加载并执行。
- **`-s`**：开启 GDB 服务器，默认监听端口 `1234`，允许 GDB 通过远程调试接口连接到 QEMU 上运行的程序。
- **`-S`**：启动 QEMU 后立即暂停程序执行。QEMU 将在启动时等待 GDB 连接，直到我们在 GDB 中下达 `continue` 命令后才开始执行程序。这使得我们能够在程序执行之前设置断点或查看初始状态。

### QEMU 启动过程

1. **加载和启动裸机程序**：`hello.elf` 被 QEMU 作为内核程序加载。这与操作系统环境不同，因为裸机程序并没有任何操作系统提供的服务或库。QEMU 将模拟硬件平台，为程序提供执行环境。
2. **暂停并等待 GDB 调试**：通过 `-S` 选项，QEMU 在启动后暂停程序的执行，这为我们调试提供了准备时间。我们可以连接 GDB 并设置断点、检查寄存器或执行其他调试操作。
3. **无图形输出**：由于裸机程序一般不涉及 GUI 操作，我们通过 `-nographic` 将输出直接映射到终端。这有助于捕获程序的标准输出、日志或其他调试信息。

## Makefile 第二步：添加 QEMU 启动规则

为了简化启动过程，我们可以将 QEMU 启动命令添加到 Makefile 中。这样可以通过运行 `make qemu` 来自动启动 QEMU 并加载我们的程序。

以下是更新后的 Makefile，增加了 QEMU 启动的规则：

```makefile
# 定义工具链和编译器选项
CC = riscv64-unknown-elf-gcc
CFLAGS = -g -nostartfiles -nostdlib -mcmodel=medany
LDFLAGS = -T linker.ld

# 目标文件
TARGET = hello.elf

# 源文件
SRCS = start.s hello.c

# QEMU 启动命令
QEMU = qemu-system-riscv64
QEMU_FLAGS = -machine virt -nographic -kernel $(TARGET) -s -S

# 编译规则
$(TARGET): $(SRCS)
	$(CC) $(CFLAGS) $(LDFLAGS) -o $(TARGET) $(SRCS)

# 启动 QEMU 进行调试
.PHONY: qemu
qemu: $(TARGET)
	$(QEMU) $(QEMU_FLAGS)

# 伪目标 clean，用于清理生成的文件
.PHONY: clean
clean:
	rm -f $(TARGET)
```

- **`QEMU`**：定义了 QEMU 模拟器的名称，这里是 `qemu-system-riscv64`，用于模拟 RISC-V 64 位架构。
- **`QEMU_FLAGS`**：QEMU 启动时所需的参数，包含虚拟平台配置、无图形输出模式、内核加载、调试支持等选项。
- **`qemu` 伪目标**：这是一个伪目标，用于启动 QEMU。当我们执行 `make qemu` 时，它会检查是否已经生成了目标文件 `hello.elf`，然后执行 QEMU 启动命令。

### 使用步骤：

1. **编译程序**：
   首先，确保程序已经成功编译并生成 `hello.elf` 文件。你可以通过运行以下命令来构建项目：
   
   ```bash
   make
   ```

   这将会调用 `riscv64-unknown-elf-gcc` 生成 `hello.elf`，并准备好用于 QEMU 启动。

2. **启动 QEMU**：
   一旦程序编译完成，你可以通过以下命令启动 QEMU：
   
   ```bash
   make qemu
   ```

   QEMU 将加载 `hello.elf`，并根据 `-s` 和 `-S` 参数暂停在程序的起始位置，等待 GDB 调试器的连接。

3. **连接 GDB**（下一步）：
   启动 QEMU 后，它会暂停程序的执行，并在端口 `1234` 上开启一个 GDB 服务器。在下一步中，我们将使用 GDB 连接到这个服务器，并调试 `hello.elf` 程序。

4. **清理项目**：
   如果你想清理生成的文件，可以运行以下命令：
   
   ```bash
   make clean
   ```

   这会删除编译生成的 `hello.elf` 文件，使目录保持干净。

## 第三步：使用 GDB 调试

在第二步中，我们使用 QEMU 启动了 `hello.elf` 裸机程序，并让程序处于暂停状态，等待调试器的连接。现在我们将使用 `riscv64-unknown-elf-gdb` 连接到 QEMU，进行程序的调试。GDB 是一个强大的调试工具，它支持设置断点、单步执行、检查寄存器、查看内存内容等功能，非常适合用于调试裸机程序。

通过 GDB，我们可以在程序运行的不同阶段插入断点、跟踪变量和内存状态，或者单步执行代码，深入了解程序的执行流程。

### 使用 GDB 连接 QEMU

QEMU 启动时使用了 `-s` 和 `-S` 选项，分别开启了 GDB 服务器并让程序在启动时暂停。现在我们可以通过 GDB 连接到 QEMU 提供的调试接口，开始调试 `hello.elf`。

#### 启动 GDB 的命令

首先，在终端中启动 GDB，并加载 `hello.elf` 文件以获取调试信息：

```bash
riscv64-unknown-elf-gdb hello.elf
```

这将启动 GDB，并加载 `hello.elf` 中的调试符号表（因为在编译时使用了 `-g` 选项）。接下来，我们需要让 GDB 连接到 QEMU 服务器。

#### 连接到 QEMU 服务器

QEMU 在启动时通过 `-s` 选项开启了 GDB 服务器，默认监听在 `localhost:1234` 端口。我们需要在 GDB 中执行以下命令，连接到 QEMU 的 GDB 服务器：

```bash
(gdb) target remote localhost:1234
```

一旦连接成功，GDB 将能够控制程序的执行。由于 QEMU 使用了 `-S` 选项，程序已经在启动时暂停，等待进一步的指令。

## Makefile 第三步：添加 GDB 调试规则

为了进一步简化调试流程，我们可以将 GDB 调试命令整合到 Makefile 中。通过添加 GDB 调试的伪目标，可以直接通过 `make gdb` 启动 GDB 并自动连接到 QEMU。

以下是更新后的 Makefile，增加了 GDB 调试的规则：

```makefile
# 定义工具链和编译器选项
CC = riscv64-unknown-elf-gcc
CFLAGS = -g -nostartfiles -nostdlib -mcmodel=medany
LDFLAGS = -T linker.ld

# 目标文件
TARGET = hello.elf

# 源文件
SRCS = start.s hello.c

# QEMU 启动命令
QEMU = qemu-system-riscv64
QEMU_FLAGS = -machine virt -nographic -kernel $(TARGET) -s -S

# GDB 命令
GDB = riscv64-unknown-elf-gdb

# 编译规则
$(TARGET): $(SRCS)
	$(CC) $(CFLAGS) $(LDFLAGS) -o $(TARGET) $(SRCS)

# 启动 QEMU 进行调试
.PHONY: qemu
qemu: $(TARGET)
	$(QEMU) $(QEMU_FLAGS)

# 启动 GDB 并连接到 QEMU
.PHONY: gdb
gdb: $(TARGET)
	$(GDB) $(TARGET) -ex "target remote localhost:1234"

# 伪目标 clean，用于清理生成的文件
.PHONY: clean
clean:
	rm -f $(TARGET)
```

- **`GDB`**：我们定义了 `riscv64-unknown-elf-gdb` 作为 GDB 的命令。
- **`gdb` 伪目标**：这个伪目标将自动启动 GDB，并使用 `-ex "target remote localhost:1234"` 命令连接到 QEMU 提供的 GDB 服务器，简化了调试过程。
- **QEMU 和 GDB 的独立目标**：`qemu` 目标启动 QEMU，而 `gdb` 目标启动 GDB，并连接到正在运行的 QEMU 实例。

### 使用步骤：

1. **构建程序**：
   首先，确保程序已经成功编译并生成 `hello.elf` 文件。你可以通过以下命令编译项目：
   
   ```bash
   make
   ```

2. **启动 QEMU**：
   运行以下命令启动 QEMU，并让程序处于暂停状态，等待调试器连接：
   
   ```bash
   make qemu
   ```

3. **启动 GDB 并连接到 QEMU**：
   在另一个终端中，运行以下命令启动 GDB 并自动连接到 QEMU：
   
   ```bash
   make gdb
   ```

   这将会打开 GDB 并连接到 QEMU 的 GDB 服务器，准备调试。

4. **开始调试**：
   你现在可以在 GDB 中执行各种调试操作，例如设置断点、查看寄存器、单步执行等。例如：

   - **继续执行程序**：
   
     使用 `continue` 命令让程序从当前暂停的状态继续执行，直到遇到断点或程序结束。
   
     ```bash
     (gdb) continue
     ```
   
   - **设置断点**：
   
     在特定的函数或代码行处设置断点，程序执行到此处时会暂停。例如，设置一个断点在 `main` 函数：
   
     ```bash
     (gdb) break main
     ```
   
     GDB 会在程序执行到 `main` 函数的第一行时暂停，等待进一步指令。
   
   - **单步执行**：
   
     单步执行可以逐行检查程序的运行情况，适合调试复杂的逻辑。`step` 命令会执行当前代码行，并进入函数调用。
   
     ```bash
     (gdb) step
     ```
   
     使用 `next` 命令可以跳过函数调用，不进入函数内部。
   
     ```bash
     (gdb) next
     ```
   
   - **查看寄存器状态**：
   
     裸机程序需要密切关注寄存器的状态。通过 GDB，你可以查看 CPU 寄存器的值，例如：
   
     ```bash
     (gdb) info registers
     ```
   
     这将显示当前所有 CPU 寄存器的内容。
   
   - **查看内存**：
   
     你还可以检查特定内存地址的内容。例如，查看地址 `0x80000000` 处的 16 个字节：
   
     ```bash
     (gdb) x/16xb 0x80000000
     ```
   
     其中，`x` 表示检查内存，`16xb` 表示以 16 个字节为单位查看。
   
   - **退出调试**：
   
     调试完成后，可以使用 `quit` 命令退出 GDB。
   
     ```bash
     (gdb) quit
     ```
   

---

## 包含多个源文件

最后我们可以结合上一节的内容，创建一个包含多个源文件（`main.c`、`utils.c`、`utils.h`）的 Makefile，使用 GDB 调试和 QEMU 启动，并加入自动依赖生成的功能。这个 Makefile 适合用于裸机开发，并且可以自动管理编译、链接、清理、调试等任务。

### 项目结构

假设项目的目录结构如下：

```
riscv-baremetal/
├── Makefile
├── hello.c
├── start.s
├── utils.c
├── utils.h
├── linker.ld
```

- `hello.c`：主程序文件。
- `start.s`：启动汇编代码。
- `utils.c`：工具模块，包含辅助函数。
- `utils.h`：工具模块的头文件，声明 `utils.c` 中的函数。
- `linker.ld`：链接器脚本，定义程序的内存布局。

## Makefile 各部分功能

### 编译与链接

为了编译 `hello.c` 和 `utils.c`，并将它们与启动代码 `start.s` 链接在一起，我们需要定义一个包含多文件的 Makefile。在这里，我们还会自动生成头文件的依赖关系，确保当 `utils.h` 发生变化时，相关的 `.c` 文件会自动重新编译。

### 启动 QEMU 并连接 GDB

Makefile 中会有两个独立的目标：
1. **`qemu`**：用于启动 QEMU 虚拟机并加载 `hello.elf`，虚拟机启动后处于暂停状态，等待 GDB 连接。
2. **`gdb`**：用于启动 GDB 并自动连接到 QEMU 的调试接口。

### 最终 Makefile

以下是完整的 Makefile，它将编译多文件项目、生成依赖文件、清理生成的目标文件、并提供通过 QEMU 和 GDB 进行调试的功能。

```makefile
# 定义工具链和编译器选项
CC = riscv64-unknown-elf-gcc
CFLAGS = -g -nostartfiles -nostdlib -mcmodel=medany -Wall -MMD
LDFLAGS = -T linker.ld

# 定义目标文件
TARGET = hello.elf

# 源文件
SRCS = start.s hello.c utils.c

# 对象文件 (将所有 .c 文件和 .s 文件编译为 .o 文件)
OBJS = $(SRCS:.c=.o)
OBJS := $(OBJS:.s=.o)

# QEMU 启动命令
QEMU = qemu-system-riscv64
QEMU_FLAGS = -machine virt -nographic -kernel $(TARGET) -s -S

# GDB 命令
GDB = riscv64-unknown-elf-gdb

# 编译规则
$(TARGET): $(OBJS)
	$(CC) $(CFLAGS) $(LDFLAGS) -o $@ $^

# 定义编译规则，使用自动化变量，适用于 .c 和 .s 文件
%.o: %.c
	$(CC) $(CFLAGS) -c $< -o $@

%.o: %.s
	$(CC) $(CFLAGS) -c $< -o $@

# 包含自动生成的依赖文件
-include $(SRCS:.c=.d)

# 启动 QEMU 进行调试
.PHONY: qemu
qemu: $(TARGET)
	$(QEMU) $(QEMU_FLAGS)

# 启动 GDB 并连接到 QEMU
.PHONY: gdb
gdb: $(TARGET)
	$(GDB) $(TARGET) -ex "target remote localhost:1234"

# 伪目标 clean，用于清理生成的文件
.PHONY: clean
clean:
	rm -f $(OBJS) $(TARGET) *.d
```

1. **自动化编译规则**：
   - `%` 通配符规则让 `.c` 文件和 `.s` 文件都可以自动编译为 `.o` 文件，避免手动为每个文件编写规则。
   - `$@` 代表当前目标，`$^` 代表所有依赖文件。每个 `.o` 文件都会根据其依赖的 `.c` 或 `.s` 文件自动生成。
   
2. **自动生成依赖文件**：
   - `-MMD` 选项告诉编译器在编译 `.c` 文件时自动生成 `.d` 文件，记录头文件的依赖关系。
   - `-include` 包含所有生成的 `.d` 文件，以确保在头文件修改时，Make 会自动重新编译相关源文件。

3. **QEMU 与 GDB 调试**：
   - **`qemu` 伪目标**：启动 QEMU，加载 `hello.elf`，并暂停等待调试。
   - **`gdb` 伪目标**：启动 GDB 并连接到 QEMU 提供的 GDB 服务器。

4. **清理规则**：
   - `clean` 伪目标用于删除所有生成的 `.o` 文件、`hello.elf` 可执行文件，以及自动生成的 `.d` 依赖文件。

### 使用步骤

1. **编译项目**：

   首先运行 `make` 命令，Makefile 会自动编译所有源文件并生成 `hello.elf`：

   ```bash
   make
   ```

   如果一切正常，你将会看到 `hello.elf` 文件生成。

2. **启动 QEMU**：

   使用 `make qemu` 启动 QEMU，并让程序暂停等待 GDB 连接：

   ```bash
   make qemu
   ```

   QEMU 将加载 `hello.elf`，并根据 `-s` 和 `-S` 参数启动调试服务器并暂停程序。

3. **启动 GDB 并连接 QEMU**：

   在另一个终端中，运行 `make gdb`，这将自动启动 GDB 并连接到 QEMU：

   ```bash
   make gdb
   ```

   你可以在 GDB 中使用常用的调试命令，例如设置断点、单步执行、查看寄存器和内存等。

4. **清理项目**：

   完成调试后，你可以使用 `make clean` 清理生成的文件，保持项目目录整洁：

   ```bash
   make clean
   ```

## 总结

通过这个综合的 Makefile，我们实现了：

1. **多文件编译与链接**：编译多个 `.c` 和 `.s` 文件，生成裸机可执行文件 `hello.elf`。
2. **自动生成依赖文件**：通过 `-MMD` 自动生成 `.d` 文件，确保源文件和头文件之间的依赖关系被正确管理。
3. **QEMU 启动与 GDB 调试**：通过 `make qemu` 启动虚拟机并加载程序，通过 `make gdb` 启动调试器并自动连接 QEMU。
4. **清理规则**：提供了 `clean` 目标，快速清理生成的中间文件和目标文件。

这个 Makefile 是裸机开发的基础，可以根据项目的复杂性进一步扩展。例如，你可以添加更多的源文件或头文件，或者调整编译选项以适应特定的硬件平台和应用需求。
