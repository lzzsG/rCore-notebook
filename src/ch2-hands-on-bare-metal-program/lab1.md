# Lab1

- [1. 设置环境](#1-设置环境)
- [2. 创建项目结构/试运行](#2-创建项目结构试运行)
- [3. 理解执行环境与平台支持](#3-理解执行环境与平台支持)
- [4. 移除标准库依赖](#4-移除标准库依赖)
- [5. 构建用户态执行环境](#5-构建用户态执行环境)
- [6 构建裸机执行环境](#6-构建裸机执行环境)

---

# 1. 设置环境

- 参考lab0，确保你已安装 Rust 、Qemu以及相关工具。

# 2. 创建项目结构/试运行

- 创建一个新的 Cargo 项目：`cargo new os`

  - 这将创建一个名为 `os` 的新项目，文件结构如下：

    ```apache
    os
    ├── Cargo.toml
    └── src
        └── main.rs
    ```

- 在项目目录下创建 `bootloader` 文件夹，并将 `rustsbi-qemu.bin` 放入其中。

  - **rustsbi-qemu.bin**：这是内核依赖的运行在 M 特权级的 SBI（Supervisor Binary Interface）实现。本项目中使用 RustSBI，通常用于在 QEMU 虚拟机中启动内核。

- 尝试运行项目，使用命令 `cargo run`

  - 这将编译并运行项目，输出如下：

    ```lua
    Compiling os v0.1.0 (/path/to/os)
    Finished dev [unoptimized + debuginfo] target(s) in 1.15s
    Running `target/debug/os`
    Hello, world!
    ```

  
  - 确保项目目录结构如下：
  
    ```apache
    os/
    ├── Cargo.toml
    ├── bootloader/
    │   └── rustsbi-qemu.bin
    └── src/
        └── main.rs
    ```

> - 在 `src/main.rs`文件中，`cargo`已经为我们准备了最简单的 Rust 应用程序：
>
>  ``` rust
>     fn main() {
>         println!("Hello, world!");
>     } 
>  ```
>
> - 尽管这个 Rust 代码中只有几行代码来实现 `Hello, world!` 的输出，但实际上，背后涉及许多复杂的底层支持才能实现这一功能。这包括：
>
>   - Rust 标准库 `std` 提供的 `println!` 宏。
>   - `std` 库依赖的底层实现，如 GNU Libc 库提供的标准输入输出函数。
>   - 操作系统（如 Linux）提供的系统调用接口，用于实现底层的输入输出操作。
>   - 硬件驱动程序和设备管理程序，这些由操作系统管理，以支持屏幕显示等硬件操作。
>
>   因此，看似简单的几行代码实际上依赖于多层次的软件和硬件支持，才能最终在屏幕上打印出 "Hello, world!"。



# 3. 理解执行环境与平台支持

## 3.1 理解应用程序执行环境

<div style="text-align:center;">     <img src="./assets/app-software-stack.png" alt="../_images/app-software-stack.png" style="display:block; margin:auto;" /> </div>

<div style="text-align:center; font-size:14px;">
应用程序执行环境栈：图中的白色块自上而下表示各级执行环境，黑色块则表示相邻两层执行环境之间的接口。 下层作为上层的执行环境，支持上层代码运行。
</div>

该图片展示了一个典型的应用程序执行环境栈，从底层硬件到最顶层的应用程序。每一层都为上层提供支持，并且各层之间通过接口进行交互。

1. **硬件平台**：
   - 底层的物理硬件，包括 CPU、内存和输入输出设备。
   - 所有软件层次最终都要与硬件平台交互来执行实际的操作。

2. **指令集**：
   - 硬件平台支持的 CPU 指令集架构（ISA），如 x86_64、ARM 或 RISC-V。
   - 指令集定义了 CPU 可以执行的指令集合，是硬件与软件之间的接口。

3. **内核/操作系统**：
   - 操作系统内核管理硬件资源，并提供基本的系统功能，如进程调度、内存管理、文件系统和设备驱动程序。
   - 操作系统通过系统调用接口（System Call Interface）与用户空间程序交互。

4. **系统调用**：
   - 系统调用是应用程序与操作系统交互的接口。
   - 通过系统调用，应用程序可以请求操作系统执行特权操作，如文件操作、进程管理和网络通信。

5. **标准库**：
   - 标准库（如 Rust 的 `std` 库或 C 的 `glibc`）为应用程序提供了高层次的功能抽象，封装了系统调用，简化了编程。
   - 标准库通过系统调用与操作系统交互，从而实现其功能。

6. **函数调用**：
   - 函数调用是应用程序内部或者不同库之间的调用。
   - 这是应用程序逻辑的具体实现部分，通过调用库函数实现各种功能。

7. **应用程序**：
   - 顶层是最终运行的应用程序。
   - 应用程序使用标准库和第三方库提供的功能，通过函数调用实现其特定的业务逻辑。

### 例子：Hello, world! 程序

让我们用 `Hello, world!` 程序来具体说明这一执行环境栈：

```rust
fn main() {
    println!("Hello, world!");
}
```

1. **应用程序**：
   - `main` 函数调用了 `println!` 宏，这就是我们的应用程序层。

2. **函数调用**：
   - `println!` 宏展开后会调用标准库中的 I/O 函数，这属于函数调用层。

3. **标准库**：
   - `println!` 宏是由 Rust 标准库 `std` 提供的。`std` 库内部实现了调用系统调用的逻辑。

4. **系统调用**：
   - `std` 库中的 I/O 函数最终通过系统调用接口请求操作系统将字符串输出到标准输出（屏幕）。

5. **内核/操作系统**：
   - 操作系统内核管理和调度硬件资源，执行系统调用，将字符串数据传递给适当的设备驱动程序。

6. **指令集**：
   - 操作系统内核和驱动程序通过 CPU 指令集与硬件进行交互，执行相应的操作指令。

7. **硬件平台**：
   - 最终，物理硬件（如显示器）在屏幕上显示 "Hello, world!" 字符串。

> **执行环境 (Execution Environment)** 是指应用程序在运行时所依赖的各种软件和硬件组件的集合。这个环境为应用程序提供了必需的基础设施和支持，以便其能够正常运行并执行预期的功能。执行环境通常包括以下几个层次：
>
> - **硬件平台**：物理设备，包括 CPU、内存、存储、网络和输入输出设备。
> - **操作系统**：管理硬件资源，提供基本服务，如文件系统、网络通信和进程调度。
> - **系统调用接口**：应用程序与操作系统之间的接口，通过它们可以访问操作系统提供的服务。
> - **标准库和第三方库**：提供高层次的功能抽象，简化应用程序的开发。标准库通常由编程语言的运行时提供，如 Rust 的 `std` 库。
> - **应用程序代码**：实际运行的业务逻辑，通过调用标准库和系统调用来完成具体的功能。

## 3.2 平台与目标三元组

**平台 (Platform)** 和 **目标三元组 (Target Triplet)** 是编译器在编译和链接时需要了解的，以便生成在特定环境中运行的可执行文件。

### 目标三元组

目标三元组是一个由三部分组成的字符串，用来描述目标平台的信息：
1. **CPU 指令集**：例如 x86_64 或 riscv64。
2. **操作系统类型**：例如 linux 或 none（裸机平台）。
3. **标准运行时库**：例如 gnu（使用 GNU C 库）或 elf（没有标准库，仅生成 ELF 格式的可执行文件）。

#### 示例分析

让我们分析当前的 `Hello, world!` 程序的目标三元组：

```apache
$ rustc --version --verbose
rustc 1.61.0-nightly (68369a041 2022-02-22)
binary: rustc
commit-hash: 68369a041cea809a87e5bd80701da90e0e0a4799
commit-date: 2022-02-22
host: x86_64-unknown-linux-gnu
release: 1.61.0-nightly
LLVM version: 14.0.0
```

输出中 `host` 一项表明默认目标平台是 `x86_64-unknown-linux-gnu`，具体解释如下：
- **CPU 架构**：`x86_64`
- **CPU 厂商**：`unknown`
- **操作系统**：`linux`
- **运行时库**：`gnu libc`

### 移植到 RISC-V 平台

我们希望将 `Hello, world!` 程序移植到 RISC-V 目标平台 `riscv64gc-unknown-none-elf` 上运行。

- **riscv64gc**：CPU 架构是 RISC-V 64 位，支持压缩指令和浮点指令集扩展。
- **unknown**：CPU 厂商是未知。
- **none**：操作系统是无操作系统（裸机平台）。
- **elf**：没有标准运行时库，但可以生成 ELF 格式的可执行文件。

我们选择 `riscv64gc-unknown-none-elf` 而不是 `riscv64gc-unknown-linux-gnu`，因为我们的目标是开发操作系统内核，而不是在 Linux 系统上运行的应用程序。

### 修改目标平台

为了将程序的目标平台换成 `riscv64gc-unknown-none-elf`，可以按照以下步骤进行。让我们试试看会发生什么：

```apache
$ cargo run --target riscv64gc-unknown-none-elf
   Compiling os v0.1.0 (/home/shinbokuow/workspace/v3/rCore-Tutorial-v3/os)
error[E0463]: can't find crate for `std`
  |
  = note: the `riscv64gc-unknown-none-elf` target may not be installed
```

报错的原因是目标平台上确实没有 Rust 标准库 `std`，也不存在任何受 OS 支持的系统调用。这样的平台被我们称为 **裸机平台 (bare-metal)**。

### 裸机平台 (Bare-metal)

在裸机平台上，程序直接运行在硬件上，没有操作系统的支持。因此，许多高级功能（如文件系统、网络堆栈等）无法使用。裸机平台通常用于嵌入式系统、微控制器和自定义操作系统的开发。

幸运的是，除了标准库 `std` 之外，Rust 还有一个不需要任何操作系统支持的核心库 `core`，它包含了 Rust 语言的大部分核心机制，可以满足我们在裸机平台上的需求。有许多第三方库也不依赖于标准库 `std`，而仅仅依赖于核心库 `core`。

### 使用核心库 `core`

为了以裸机平台为目标编译程序，我们需要将对标准库 `std` 的引用换成核心库 `core`。

> ###  ELF 文件
>
> ELF（Executable and Linkable Format，可执行与可链接格式）是 Unix 系统中常用的一种文件格式，主要用于可执行文件、目标代码、共享库和核心转储。以下是对 ELF 文件的一些详细解释：
>
> #### ELF 文件的基本结构
>
> ELF 文件由以下几部分组成：
>
> 1. **ELF 头部（ELF Header）**：
>    - 包含文件的总体信息，如文件类型（可执行文件、共享库、目标文件等）、架构类型、入口点地址、程序头部表和节头部表的位置和大小等。
> 2. **程序头部表（Program Header Table）**：
>    - 描述了程序在内存中的布局。每个条目描述一个段（segment），包括段的类型、虚拟地址、文件偏移、内存大小、文件大小、权限等。用于程序加载器将文件加载到内存中。
> 3. **节头部表（Section Header Table）**：
>    - 描述了文件中的各个节（section）。每个条目包含节的名称、类型、地址、偏移、大小、属性等。节用于连接和调试等用途。例如，`.text` 节通常包含代码，`.data` 节通常包含已初始化的数据。
> 4. **各个节（Sections）**：
>    - 文件的实际内容分布在各个节中，如代码节（.text）、数据节（.data）、只读数据节（.rodata）、符号表（.symtab）、字符串表（.strtab）等。
>
> #### ELF 文件的类型
>
> - **可执行文件（Executable）**：包含可以直接运行的程序代码。
> - **目标文件（Relocatable）**：包含链接器使用的代码和数据片段，用于生成可执行文件或共享库。
> - **共享库（Shared Object）**：动态链接库，可以在运行时被其他程序加载和使用。
> - **核心转储（Core Dump）**：程序异常终止时的内存映像，用于调试。
>
> #### ELF 文件的优点
>
> - **可扩展性**：结构清晰，易于扩展。
> - **可移植性**：支持不同的处理器架构。
> - **兼容性**：广泛应用于各类 Unix 系统，包括 Linux 和 BSD 系统。

---



# 4. 移除标准库依赖

## 4.1 安装目标平台工具链

```apache
$ rustup target add riscv64gc-unknown-none-elf
```

**作用**：

- 这一步安装了目标平台工具链，使你的 Rust 编译器能够生成 `riscv64gc-unknown-none-elf` 目标平台的二进制文件。
- 这是为交叉编译准备的基础步骤，因为没有安装目标平台工具链，就无法生成该平台的可执行文件。

## 4.2 设置默认目标平台

首先在 `os` 目录下新建 `.cargo` 目录，并在这个目录下创建 `config` 文件，输入如下内容：

```toml
# os/.cargo/config
[build]
target = "riscv64gc-unknown-none-elf"
```

### `.cargo/config` 的作用和效果

- **作用**：通过在 `.cargo/config` 文件中指定目标平台，可以让 Cargo 在编译和运行项目时默认使用 `riscv64gc-unknown-none-elf` 作为目标平台。
- **效果**：配置 `.cargo/config` 文件后，开发人员不再需要每次编译或运行项目时手动指定目标平台，简化了命令行操作。

### 交叉编译 (Cross Compile)

这种编译器运行的平台（如 x86_64）与可执行文件运行的目标平台（如 `riscv64gc-unknown-none-elf`）不同的情况，称为 **交叉编译** (Cross Compile)。交叉编译允许在开发机上生成适用于其他架构或操作系统的可执行文件。

配置 `.cargo/config` 文件的具体效果是：当运行 Cargo 命令（如 `cargo build` 或 `cargo run`）时，Cargo 工具会自动使用 `riscv64gc-unknown-none-elf` 作为目标平台，而不需要每次手动指定 `--target` 参数。例如：

- **未配置 `.cargo/config` 文件时**：
  ```apache
  $ cargo build --target riscv64gc-unknown-none-elf
  $ cargo run --target riscv64gc-unknown-none-elf
  ```

- **配置 `.cargo/config` 文件后**：
  ```apache
  $ cargo build
  $ cargo run
  ```

这样不仅简化了操作，还确保了项目的一致性，特别是在团队协作时，每个开发者都可以使用相同的目标平台配置。

>  **交叉编译（Cross Compile）**
>
> **交叉编译** 是指在一个平台上生成在另一个平台上运行的可执行文件。即编译器运行的平台（host）与可执行文件运行的目标平台（target）不同的情况。
>
> #### 举例说明
>
> 在本项目中，我们的开发环境可能是基于 x86_64 架构的 Linux 系统（如个人电脑），而目标平台是 `riscv64gc-unknown-none-elf`，这是一个基于 RISC-V 架构的裸机平台。
>
> - **编译器平台（Host Platform）**：`x86_64-unknown-linux-gnu`
> - **目标平台（Target Platform）**：`riscv64gc-unknown-none-elf`
>
> 交叉编译就是在 `x86_64-unknown-linux-gnu` 平台上编译生成 `riscv64gc-unknown-none-elf` 平台上的可执行文件。
>
> ### **为什么需要交叉编译？**
>
> 交叉编译在多种场景下非常有用，尤其是在嵌入式开发、操作系统开发以及跨平台软件开发中。以下是一些原因：
>
> 1. **目标平台资源受限**：
>    - 许多嵌入式设备和裸机平台资源有限，无法直接进行编译操作。交叉编译允许在功能更强大的开发机上编译代码，然后将生成的可执行文件部署到目标设备上。
> 2. **提高开发效率**：
>    - 交叉编译可以利用功能强大的开发机进行编译，提高编译速度和效率。
> 3. **多平台支持**：
>    - 通过交叉编译，开发者可以在一个平台上编写代码并生成多个平台的可执行文件，支持不同的架构和操作系统。
>
> **`.cargo/config` 的作用**
>
> `.cargo/config` 文件用于为 Cargo 项目提供自定义配置选项，以便控制项目的构建和编译行为。在这个文件中，可以指定诸如默认目标平台、编译选项、链接器配置等。
>
> 1. **指定默认目标平台**：
>    - 通过在 `.cargo/config` 文件中设置目标平台，可以让 Cargo 在编译和运行项目时自动使用该目标平台，而不需要每次手动指定。
>    - 配置示例：
>      ```toml
>      # os/.cargo/config
>      [build]
>      target = "riscv64gc-unknown-none-elf"
>      ```
>
> 2. **简化命令行操作**：
>    - 配置了 `.cargo/config` 文件后，所有与构建相关的命令都会默认使用该文件中指定的配置，从而简化了命令行操作。
>    - 减少了开发人员在每次构建或运行项目时重复输入目标平台的麻烦。
>
> 3. **项目一致性**：
>    - 在团队协作中，确保所有开发人员使用相同的构建配置，从而避免由于不同开发环境导致的构建不一致问题。
>    - 提高了项目的可维护性和一致性。
>

## 4.3 移除 println! 宏

回到我们的程序，为了以裸机平台为目标编译程序，我们需要将对标准库 `std` 的引用换成核心库 `core`。

在 `main.rs` 的开头加上一行 `#![no_std]`，告诉 Rust 编译器不使用标准库 `std` 而转向核心库 `core`。由于 `println!` 宏是由标准库 `std` 提供的，所以这一步会导致编译错误。

### 步骤

1. **在 `main.rs` 中添加 `#![no_std]`**：
    ```rust
    #![no_std]
    
    fn main() {
        println!("Hello, world!");
    }
    ```

2. **重新编译**：
    ```apache
    $ cargo build
    ```

3. **错误信息**：
    ```
    error: cannot find macro `println` in this scope
    --> src/main.rs:4:5
      |
    4 |     println!("Hello, world!");
      |     ^^^^^^^
    ```

### 解释

- `println!` 宏是由标准库 `std` 提供的，依赖于 `write` 系统调用。
- 在没有标准库的情况下，无法使用 `println!` 宏，因此会报错。
- 为了继续编译，我们需要移除或注释掉这行代码。我们先直接把`main`函数注释掉。

## 4.4 提供语义项 `panic_handler`

在移除 `println!` 宏后，我们还需要处理另一个错误：缺少 `#[panic_handler]` 函数。

### 步骤

1. **编译后遇到的错误信息**：
    ```apache
    $ cargo build
    ```
    ```
    error: `#[panic_handler]` function required, but not found
    ```

### 解释

- 标准库 `std` 提供了 Rust 错误处理函数 `#[panic_handler]`，其功能是打印出错位置和原因并终止当前应用。
- 核心库 `core` 并没有提供这种功能，所以我们需要自己实现一个 `panic_handler`。

### 实现 `panic_handler`

1. **创建子模块 `lang_items.rs`**：
    - 在 `src` 目录下创建一个新的文件 `lang_items.rs`。

2. **在 `lang_items.rs` 中编写 `panic_handler` 函数**：
    ```rust
    // os/src/lang_items.rs
    use core::panic::PanicInfo;
    
    #[panic_handler]
    fn panic(_info: &PanicInfo) -> ! {
        loop {}
    }
    ```

    - `#[panic_handler]` 标记告诉编译器这个函数是我们的 panic 处理函数。
    - 当程序发生 panic 时，这个函数会被调用。当前实现只是进入一个无限循环。

3. **在 `main.rs` 中引入 `lang_items` 模块**：
    - 修改 `main.rs`，引入我们定义的 `panic_handler`：
    ```rust
    #![no_std]
    
    mod lang_items;
    
    fn main() {
       // println!("Hello, world!"); // 注释掉这一行
    }
    ```

4. **重新编译**：
   
    ```apache
    $ cargo build
    ```
    
    又有了新错误：
    
    ```apache
    $ cargo build
       Compiling os v0.1.0 (/home/shinbokuow/workspace/v3/rCore-Tutorial-v3/os)
    error: requires `start` lang_item
    ```

## 4.5 移除 main 函数

编译器提醒我们缺少一个名为 `start` 的语义项。 `start` 语义项代表了标准库 std 在执行应用程序之前需要进行的一些初始化工作。由于我们禁用了标准库，编译器也就找不到这项功能的实现了。

在 `main.rs` 的开头加入设置 `#![no_main]` 告诉编译器我们没有一般意义上的 `main` 函数， 并将原来的 `main` 函数删除。这样编译器也就不需要考虑初始化工作了。

### 步骤

1. **在 `main.rs` 的开头加入 `#![no_main]`**：
   - 这行代码告诉编译器我们没有通常意义上的 `main` 函数。
   - 然后删除原来的 `main` 函数。
   - 修改后的 `main.rs` 文件内容如下：
     ```rust
     #![no_std]
     #![no_main]
     
     mod lang_items;
     ```

2. **重新编译**：
    ```apache
    $ cargo build
    ```

    执行 `cargo build`，编译成功：

```apache
$ cargo build
   Compiling os v0.1.0 (/home/shinbokuow/workspace/v3/rCore-Tutorial-v3/os)
    Finished dev [unoptimized + debuginfo] target(s) in 0.06s
```

至此，我们终于移除了所有标准库依赖，目前的代码如下：

```rust
// os/src/main.rs
#![no_std]
#![no_main]

mod lang_items;

// os/src/lang_items.rs
use core::panic::PanicInfo;

#[panic_handler]
fn panic(_info: &PanicInfo) -> ! {
    loop {}
}
```

### 分析被移除标准库的程序

我们可以通过一些工具来分析目前的程序：

1. **文件格式**：
    ```apache
    $ file target/riscv64gc-unknown-none-elf/debug/os
    target/riscv64gc-unknown-none-elf/debug/os: ELF 64-bit LSB executable, UCB RISC-V, ......
    ```

2. **文件头信息**：
   
    ```apache
    $ rust-readobj -h target/riscv64gc-unknown-none-elf/debug/os
       File: target/riscv64gc-unknown-none-elf/debug/os
       Format: elf64-littleriscv
       Arch: riscv64
       AddressSize: 64bit
       ......
       Type: Executable (0x2)
       Machine: EM_RISCV (0xF3)
       Version: 1
       Entry: 0x0
       ......
       }
    ```
    
3. **反汇编导出汇编程序**：
    ```apache
    $ rust-objdump -S target/riscv64gc-unknown-none-elf/debug/os
       target/riscv64gc-unknown-none-elf/debug/os:       file format elf64-littleriscv
    ```

通过 `file` 工具对二进制程序 `os` 的分析可以看到，它好像是一个合法的 RV64 执行程序，但 `rust-readobj` 工具告诉我们它的入口地址 `Entry` 是 0。再通过 `rust-objdump` 工具把它反汇编，没有生成任何汇编代码。可见，这个二进制程序虽然合法，但它是一个空程序，原因是缺少了编译器规定的入口函数 `_start`。

下面我们将着手实现本节移除的、由用户态执行环境提供的功能。

> 在构建和分析一个移除标准库的程序时，使用了一些常用的命令行工具来检查生成的二进制文件。这些工具帮助我们理解生成的程序是否正确以及是否包含了预期的内容。以下是对这些命令和它们的作用的详细解释：
>
> ### 1. **文件格式**
>
> ```apache
> $ file target/riscv64gc-unknown-none-elf/debug/os
> target/riscv64gc-unknown-none-elf/debug/os: ELF 64-bit LSB executable, UCB RISC-V, ......
> ```
>
> - **`file` 命令**：这个命令用于确定文件的类型。它通过检查文件的头部信息，推断文件的格式和内容。
> - **输出说明**：输出结果表明 `os` 文件是一个 ELF 64-bit LSB（Little-Endian）可执行文件，适用于 UCB RISC-V 架构。其他细节省略表示更多的文件头信息。
>
> ### 2. **文件头信息**
>
> ```apache
> $ rust-readobj -h target/riscv64gc-unknown-none-elf/debug/os
>    File: target/riscv64gc-unknown-none-elf/debug/os
>    Format: elf64-littleriscv
>    Arch: riscv64
>    AddressSize: 64bit
>    ......
>    Type: Executable (0x2)
>    Machine: EM_RISCV (0xF3)
>    Version: 1
>    Entry: 0x0
>    ......
>    }
> ```
>
> - **`rust-readobj` 命令**：这是一个 Rust 工具，类似于 `readelf`，用于读取和显示 ELF 格式文件的内容。`-h` 选项表示显示文件头信息。
> - **输出说明**：
>   - **Format**: 文件格式是 ELF 64-bit 小端（littleriscv）。
>   - **Arch**: 架构是 riscv64。
>   - **AddressSize**: 地址大小是 64 位。
>   - **Type**: 文件类型是可执行文件（0x2）。
>   - **Machine**: 机器类型是 RISC-V（EM_RISCV）。
>   - **Entry**: 程序入口地址是 0x0（这意味着程序没有正确设置入口点）。
>
> ### 3. **反汇编导出汇编程序**
>
> ```apache
> $ rust-objdump -S target/riscv64gc-unknown-none-elf/debug/os
>    target/riscv64gc-unknown-none-elf/debug/os:       file format elf64-littleriscv
> ```
>
> - **`rust-objdump` 命令**：这是一个 Rust 工具，用于反汇编 ELF 文件。`-S` 选项表示显示反汇编后的代码。
> - **输出说明**：输出结果显示文件格式为 elf64-littleriscv。由于没有进一步的输出，意味着该文件没有包含任何有效的汇编指令。
>
> ### 总结
>
> 通过上述工具的输出，我们得出以下结论：
>
> 1. **文件格式**：`file` 命令确认生成的 `os` 文件是一个合法的 RISC-V 架构的 ELF 可执行文件。
> 2. **文件头信息**：`rust-readobj` 显示文件的入口地址为 0，这表明程序缺少有效的入口点 `_start`。其他头部信息如架构和类型都是正确的。
> 3. **反汇编结果**：`rust-objdump` 没有生成任何有效的汇编代码，这表明程序是一个空程序，因为缺少有效的入口函数 `_start`。
>
> 这些工具帮助我们确认了生成的二进制文件是一个合法的 ELF 文件，但由于缺少有效的入口点 `_start`，程序实际上是空的，无法执行。因此，必须在程序中定义一个入口函数 `_start`，以确保生成的程序可以正确运行。

---

# 5. 构建用户态执行环境

## 5.1 用户态最小化执行环境

为了运行用户态程序，我们需要构建一个最小化的执行环境，并提供必要的初始化代码。以下是具体步骤和相关解释。

### 5.1.1 执行环境初始化

首先，我们需要为 Rust 编译器提供一个入口函数 `_start`。这是程序执行的起点，类似于标准库程序中的 `main` 函数。在 `main.rs` 文件中添加如下内容：

```rust
// os/src/main.rs
#[no_mangle]
extern "C" fn _start() {
    loop {}
}
```

### 代码解析

- **`#[no_mangle]`**：这是一个属性宏，告诉编译器不要对函数名进行修改（即不进行名称重整），确保生成的符号名与定义的一致。
- **`extern "C"`**：这表明函数采用 C 语言的调用约定，使其与其他语言（如汇编、C）的代码兼容。
- **`_start` 函数**：这是程序的入口点。该函数进入一个无限循环，表示程序已开始执行。

### 重新编译与分析

我们重新编译上述代码，并用工具分析生成的二进制文件：

```apache
$ cargo build
   Compiling os v0.1.0 (/home/user/workspace/os)
    Finished dev [unoptimized + debuginfo] target(s) in 0.06s
```

使用 `rust-objdump` 反汇编导出汇编程序：

```apache
$ rust-objdump -S target/riscv64gc-unknown-none-elf/debug/os
target/riscv64gc-unknown-none-elf/debug/os:       file format elf64-littleriscv

Disassembly of section .text:

0000000000011120 <_start>:
   ; loop {}
     11120: 09 a0            j       2 <_start+0x2>
     11122: 01 a0            j       0 <_start+0x2>
```

反汇编出的两条指令表示一个死循环，说明编译器生成了一个合理的程序。我们可以使用 QEMU 执行该程序：

```apache
$ qemu-riscv64 target/riscv64gc-unknown-none-elf/debug/os
```

### 5.1.2 程序正常退出

为了进一步验证，我们可以将 `_start` 函数中的循环语句注释掉，重新编译并分析其汇编代码：

```rust
// os/src/main.rs
#[no_mangle]
extern "C" fn _start() {
    // loop {}
}
```

重新编译后，用 `rust-objdump` 反汇编：

```bash
$ rust-objdump -S target/riscv64gc-unknown-none-elf/debug/os
target/riscv64gc-unknown-none-elf/debug/os: file format elf64-littleriscv

Disassembly of section .text:

0000000000011120 <_start>:
 ; }
   11120: 82 80              ret
```

虽然这看起来是合法的执行程序，但如果我们执行它，会引发问题：

```bash
$ qemu-riscv64 target/riscv64gc-unknown-none-elf/debug/os
段错误 (核心已转储)
```

这个简单的程序导致 QEMU 崩溃了。这是因为程序试图从 `_start` 函数返回，而没有设置正确的返回地址，导致非法操作。

> ### QEMU的运行模式
>
> QEMU 有两种运行模式：
>
> 1. **User mode（用户态模式）**：如 `qemu-riscv64`，模拟不同处理器的用户态指令的执行，并可以直接解析 ELF 可执行文件，加载运行为不同处理器编译的用户级 Linux 应用程序。
> 2. **System mode（系统态模式）**：如 `qemu-system-riscv64`，模拟一个完整的基于不同 CPU 的硬件系统，包括处理器、内存及其他外部设备，支持运行完整的操作系统。
>
> 在用户态模式下，QEMU 可以直接执行 ELF 可执行文件，但该文件必须符合特定的用户态执行环境要求。例如，需要有正确的入口点和合适的返回地址。而在系统态模式下，QEMU 会模拟完整的硬件环境，执行更复杂的操作系统和应用程序。
>
> #### 用户态模式（User Mode）
>
> 在用户态模式下，QEMU 可以运行特定架构的用户态程序。对于 RISC-V 架构，命令通常为：
>
> ```bash
> qemu-riscv64 target/riscv64gc-unknown-none-elf/debug/os
> ```
>
> 此命令将运行编译后的 RISC-V 用户态程序 `os`。在这种模式下，QEMU 仅仿真用户态指令，不涉及整个系统的模拟。
>
> #### 系统态模式（System Mode）
>
> 在系统态模式下，QEMU 可以仿真完整的系统，包括 CPU、内存和其他外设。命令通常为：
>
> ```bash
> qemu-system-riscv64 target/riscv64gc-unknown-none-elf/debug/os
> ```
>
> 这种模式可以启动一个完整的操作系统或仿真裸机环境。对于系统态模式，需要提供更多的参数来指定内存大小、硬件设备、内核映像等。例如：
>
> ```bash
> qemu-system-riscv64 -machine virt -kernel target/riscv64gc-unknown-none-elf/debug/os -nographic
> ```
>
> - **`-machine virt`**：指定虚拟机硬件类型。
> - **`-kernel`**：指定要加载的内核映像。
> - **`-nographic`**：不使用图形输出，使用控制台输出。
>
> 所以。。。

### 深入解释代码和退出机制

为了完成一个简陋的用户态最小化执行环境，我们需要提供程序退出的机制。在 RISC-V 架构下，通过使用 `ecall` 指令来发出系统调用，下面是具体实现：

#### 代码解析

##### `main.rs` 文件

```rust
// os/src/main.rs

const SYSCALL_EXIT: usize = 93;

fn syscall(id: usize, args: [usize; 3]) -> isize {
    let mut ret;
    unsafe {
        core::arch::asm!(
            "ecall",
            inlateout("x10") args[0] => ret,
            in("x11") args[1],
            in("x12") args[2],
            in("x17") id,
        );
    }
    ret
}

pub fn sys_exit(xstate: i32) -> isize {
    syscall(SYSCALL_EXIT, [xstate as usize, 0, 0])
}

#[no_mangle]
extern "C" fn _start() {
    sys_exit(9);
}
```

#### 代码解析

1. **定义系统调用号**：
   - `const SYSCALL_EXIT: usize = 93;`：定义 `exit` 系统调用的编号为 93，这是 RISC-V 的约定。

2. **实现系统调用函数**：
   - `fn syscall(id: usize, args: [usize; 3]) -> isize`：这是一个通用的系统调用函数。它使用内联汇编 `ecall` 指令来执行系统调用。
   - `core::arch::asm!`：这是 Rust 中用于编写内联汇编的宏。
     - `"ecall"`：执行 RISC-V 的 `ecall` 指令，发出系统调用。
     - `inlateout("x10") args[0] => ret`：输入输出操作数。将 `args[0]` 的值传入寄存器 `x10`，并将 `x10` 的结果值赋给 `ret`。
     - `in("x11") args[1]` 和 `in("x12") args[2]`：将 `args[1]` 和 `args[2]` 分别传入寄存器 `x11` 和 `x12`。
     - `in("x17") id`：将系统调用号传入寄存器 `x17`。

3. **实现 `sys_exit` 函数**：
   - `pub fn sys_exit(xstate: i32) -> isize`：这是一个特定的系统调用函数，用于调用 `exit` 系统调用。
   - `syscall(SYSCALL_EXIT, [xstate as usize, 0, 0])`：调用通用的 `syscall` 函数，传入 `exit` 系统调用号和退出状态。

4. **定义程序入口点 `_start`**：
   - `#[no_mangle] extern "C" fn _start()`：定义程序的入口点 `_start`，使用 `no_mangle` 防止函数名被修改，并指定使用 C 语言调用约定。
   - `sys_exit(9)`：调用 `sys_exit` 函数，退出状态为 9。

#### 编译和运行

编译程序：

```bash
$ cargo build --target riscv64gc-unknown-none-elf
```

运行程序并打印返回值：

```bash
$ qemu-riscv64 target/riscv64gc-unknown-none-elf/debug/os; echo $?
9
```

上述步骤展示了如何在一个简陋的用户态最小化执行环境中实现程序的退出机制。通过定义系统调用和使用内联汇编，我们能够发出 `exit` 系统调用，使程序正确退出，并返回指定的退出状态码。这样，我们就完成了一个基本的用户态执行环境的构建。

> ### 相关寄存器
>
> 在 RISC-V 架构中，寄存器是处理器内的一种快速存储器，用于存放数据和指令。以下是代码中涉及的几个关键寄存器：
>
> - **`x10` (a0)**：第一个参数寄存器，用于传递系统调用的第一个参数，并存储返回值。
> - **`x11` (a1)**：第二个参数寄存器，用于传递系统调用的第二个参数。
> - **`x12` (a2)**：第三个参数寄存器，用于传递系统调用的第三个参数。
> - **`x17` (a7)**：系统调用号寄存器，用于传递系统调用的编号。
>
> RISC-V 寄存器的命名有两种表示方式：物理编号（x0-x31）和功能名称（如 a0-a7 表示函数调用的参数和返回值）。在内联汇编中，我们通常使用物理编号进行操作。
>
> ### 系统调用号和 `syscall` 函数的参数
>
> #### 系统调用号
>
> 系统调用号是操作系统用来标识不同系统调用的唯一编号。在 RISC-V 架构下，系统调用号被传递给 `x17` (a7) 寄存器。例如：
>
> - **`SYSCALL_EXIT`**：编号为 93，用于标识 `exit` 系统调用。
>
> 每个系统调用号对应一个特定的操作，例如创建进程、读写文件或退出程序。操作系统根据系统调用号执行相应的内核函数。
>
> #### `syscall` 函数的参数
>
> ```rust
> fn syscall(id: usize, args: [usize; 3]) -> isize {
>     let mut ret;
>     unsafe {
>         core::arch::asm!(
>             "ecall",
>             inlateout("x10") args[0] => ret,
>             in("x11") args[1],
>             in("x12") args[2],
>             in("x17") id,
>         );
>     }
>     ret
> }
> ```
>
> `syscall` 函数是一个通用的系统调用封装器，用于发出各种系统调用。它接受以下参数：
>
> - **`id: usize`**：系统调用号，指定要执行的系统调用类型。
> - **`args: [usize; 3]`**：系统调用的参数数组，包含三个参数。
>
> ##### 参数解释
>
> - **`inlateout("x10") args[0] => ret`**：将 `args[0]` 的值传入寄存器 `x10`，并将 `x10` 的返回值存储到变量 `ret` 中。`ret` 存储系统调用的返回值。
> - **`in("x11") args[1]`**：将 `args[1]` 的值传入寄存器 `x11`。
> - **`in("x12") args[2]`**：将 `args[2]` 的值传入寄存器 `x12`。
> - **`in("x17") id`**：将系统调用号 `id` 传入寄存器 `x17`。
>
> #### `sys_exit` 函数
>
> ```rust
> pub fn sys_exit(xstate: i32) -> isize {
>     syscall(SYSCALL_EXIT, [xstate as usize, 0, 0])
> }
> ```
>
> `sys_exit` 函数是一个特定的系统调用函数，用于执行 `exit` 系统调用。它接受一个参数 `xstate`，表示退出状态，并调用 `syscall` 函数：
>
> - **`SYSCALL_EXIT`**：系统调用号 93，用于退出程序。
> - **`[xstate as usize, 0, 0]`**：传递退出状态和两个未使用的参数（设置为 0）。
>
> > 在后续章节中，我们将更深入地探讨系统调用相关内容。

## 5.2 有显示支持的用户态执行环境

为了让用户态执行环境支持字符串输出，我们需要实现类似 `println!` 的功能。虽然标准库 `std` 的 `println!` 宏无法使用，但我们可以通过扩展 `core` 库中的特性和数据结构，来定制一个类似的功能。

### 实现输出字符串的相关函数

我们需要以下几个步骤：

1. **封装对 `SYSCALL_WRITE` 系统调用的封装**
2. **实现基于 `Write` Trait 的数据结构**
3. **实现 Rust 格式化宏**

<br/>

### 5.2.1 封装 `SYSCALL_WRITE` 系统调用

```rust
const SYSCALL_WRITE: usize = 64;

pub fn sys_write(fd: usize, buffer: &[u8]) -> isize {
    syscall(SYSCALL_WRITE, [fd, buffer.as_ptr() as usize, buffer.len()])
}
```

##### 解释：

- **`SYSCALL_WRITE`**：这是一个常量，表示写操作的系统调用号。在 RISC-V 架构中，写操作的系统调用号是 64。
- **`sys_write` 函数**：该函数封装了 `SYSCALL_WRITE` 系统调用，参数包括文件描述符 `fd` 和待写入的字节数组 `buffer`。

  - **`fd`**：文件描述符，`1` 表示标准输出（stdout）。
  - **`buffer`**：待写入的数据，以字节数组形式传递。

  函数内部调用了 `syscall` 函数，并传递 `SYSCALL_WRITE`、文件描述符和数据指针及长度。

#### 5.2.2 实现基于 `Write` Trait 的数据结构

```rust
struct Stdout;

impl core::fmt::Write for Stdout {
    fn write_str(&mut self, s: &str) -> core::fmt::Result {
        sys_write(1, s.as_bytes());
        Ok(())
    }
}

pub fn print(args: core::fmt::Arguments) {
    Stdout.write_fmt(args).unwrap();
}
```

##### 解释：

- **`struct Stdout`**：定义一个名为 `Stdout` 的空结构体，用于表示标准输出设备。

- **实现 `Write` Trait**：
  - **`impl core::fmt::Write for Stdout`**：为 `Stdout` 实现 `core::fmt::Write` Trait。这使得 `Stdout` 可以使用 `write_fmt` 方法来格式化输出。
  - **`write_str` 函数**：实现 `Write` Trait 所需的 `write_str` 方法。该方法将字符串转换为字节数组，并调用 `sys_write` 函数，将数据写入标准输出。

- **`print` 函数**：
  - **参数**：接受一个 `fmt::Arguments` 类型的参数，这是格式化后的输出内容。
  - **实现**：调用 `Stdout.write_fmt(args).unwrap()`，将格式化后的内容输出到标准输。

#### 5.2.3 实现 Rust 格式化宏

最后，我们实现 `print` 和 `println` 宏。

#### 宏定义

```rust
#[macro_export]
macro_rules! print {
    ($fmt: literal $(, $($arg: tt)+)?) => {
        $crate::console::print(format_args!($fmt $(, $($arg)+)?));
    }
}

#[macro_export]
macro_rules! println {
    ($fmt: literal $(, $($arg: tt)+)?) => {
        print(format_args!(concat!($fmt, "\n") $(, $($arg)+)?));
    }
}
```

##### `print!` 宏

`print!` 宏用于格式化输出，不带换行符。以下是详细解释：

- **`#[macro_export]`**：标记宏可以在模块外部使用。这个属性告诉编译器将该宏导出，使其可以在包的任何地方使用。
  
- **`macro_rules! print`**：定义宏 `print`。`macro_rules!` 是 Rust 用来定义宏的关键字。
  
- **模式匹配**：
  ```rust
  ($fmt: literal $(, $($arg: tt)+)?) => {
      $crate::console::print(format_args!($fmt $(, $($arg)+)?));
  }
  ```
  - **`$fmt: literal`**：匹配一个字面量格式字符串。
  - **`$(, $($arg: tt)+)?`**：匹配可选的一个或多个参数。`$(...)?` 表示可选模式，`$(...)+` 表示至少一个，`$arg: tt` 表示捕获任意 Token Tree。
  
- **宏体**：
  ```rust
  $crate::console::print(format_args!($fmt $(, $($arg)+)?));
  ```
  - **`$crate`**：表示当前包的根模块。
  - **`$crate::console::print`**：调用当前包中定义的 `print` 函数。
  - **`format_args!($fmt $(, $($arg)+)?)`**：调用 `format_args!` 宏来生成格式化的参数，传递给 `print` 函数。

##### `println!` 宏

`println!` 宏用于格式化输出，并在末尾添加换行符。以下是详细解释：

- **`#[macro_export]`**：同样标记宏可以在模块外部使用。

- **模式匹配**：
  ```rust
  ($fmt: literal $(, $($arg: tt)+)?) => {
      print(format_args!(concat!($fmt, "\n") $(, $($arg)+)?));
  }
  ```
  - **`$fmt: literal`**：匹配一个字面量格式字符串。
  - **`$(, $($arg: tt)+)?`**：匹配可选的一个或多个参数。
  
- **宏体**：
  ```rust
  print(format_args!(concat!($fmt, "\n") $(, $($arg)+)?));
  ```
  - **`concat!($fmt, "\n")`**：使用 `concat!` 宏将格式字符串和换行符 `\n` 连接起来。
  - **`format_args!(concat!($fmt, "\n") $(, $($arg)+)?)`**：生成格式化的参数，包括换行符。
  - **调用 `print` 宏**：最终调用 `print` 宏进行输出。

### 5.2.4 应用示例编译和执行

#### 应用示例

将定义好的宏应用到我们的用户态程序中：

```rust
#[no_mangle]
extern "C" fn _start() {
    println!("Hello, world!");
    sys_exit(9);
}
```

#### 代码解析

- **`#[no_mangle]`**：告诉编译器不要对函数名进行修饰，使得函数名保持为 `_start`。
- **`extern "C"`**：指定函数使用 C 语言的调用约定，使其可以被外部链接和调用。
- **`_start` 函数**：程序的入口点。程序启动时，会首先执行这个函数。
  - **`println!("Hello, world!");`**：调用我们定义的 `println!` 宏，输出 "Hello, world!" 字符串并换行。
  - **`sys_exit(9);`**：调用 `sys_exit` 函数，发出退出系统调用，退出码为 9。

#### 编译和执行

编译和执行修改后的程序：

```bash
$ cargo build --target riscv64gc-unknown-none-elf
   Compiling os v0.1.0 (/path/to/project)
   Finished dev [unoptimized + debuginfo] target(s) in 0.61s

$ qemu-riscv64 target/riscv64gc-unknown-none-elf/debug/os; echo $?
  Hello, world!
  9
```

- **编译**：
  - `cargo build --target riscv64gc-unknown-none-elf`：使用 Cargo 构建目标为 `riscv64gc-unknown-none-elf` 的项目。成功后生成 `os` 可执行文件。
  
- **执行**：
  - `qemu-riscv64 target/riscv64gc-unknown-none-elf/debug/os`：使用 QEMU 运行编译好的 RISC-V 可执行文件 `os`。
  - `echo $?`：打印上一条命令的退出状态码，显示为 9，表示程序正确退出。

### 结果

运行程序后，我们可以看到输出 "Hello, world!"，并且程序以退出码 9 正确退出。通过这个过程，我们实现了一个支持 `println!` 输出和正确退出的简陋用户态执行环境。后续章节将详细介绍更多系统调用和复杂功能的实现。

---

# 6 构建裸机执行环境

## 6.1 裸机启动过程

### 讲解裸机上的最小执行环境

我们将在本节中将之前实现的用户态的最小执行环境改造为裸机上的最小执行环境，并模拟 RISC-V 64 计算机的启动过程。通过 QEMU 模拟器，我们可以加载并运行内核程序。

#### 裸机启动过程

用 QEMU 软件 `qemu-system-riscv64` 来模拟 RISC-V 64 计算机，并加载内核程序的命令如下：

```bash
qemu-system-riscv64 \
    -machine virt \
    -nographic \
    -bios $(BOOTLOADER) \
    -device loader,file=$(KERNEL_BIN),addr=$(KERNEL_ENTRY_PA)
```

##### 参数详解

1. **`qemu-system-riscv64`**：
   - 这是 QEMU 的一个实例，用于模拟 RISC-V 64 架构的计算机。

2. **`-machine virt`**：
   - 指定虚拟机的硬件类型为虚拟机类型（virt），这是一个适合于虚拟化的通用 RISC-V 机器模型。

3. **`-nographic`**：
   - 禁用图形输出，使用纯文本控制台。这对于没有显示设备的裸机环境非常有用。

4. **`-bios $(BOOTLOADER)`**：
   - 指定 BIOS 文件路径。`$(BOOTLOADER)` 是一个环境变量，表示引导加载程序（BootLoader）的路径。
   - 例如，RustSBI 是一个常见的 RISC-V BootLoader 实现。

5. **`-device loader,file=$(KERNEL_BIN),addr=$(KERNEL_ENTRY_PA)`**：
   - **`file=$(KERNEL_BIN)`**：指定要加载的内核二进制文件路径，`$(KERNEL_BIN)` 是一个环境变量，表示内核二进制文件的路径。
   - **`addr=$(KERNEL_ENTRY_PA)`**：指定内核加载的物理地址，`$(KERNEL_ENTRY_PA)` 是一个环境变量，表示内核的入口地址，通常是 `0x80200000`。

#### 启动流程

1. **虚拟机启动**：
   - 当执行 `qemu-system-riscv64` 命令时，模拟器会启动虚拟的 RISC-V 64 计算机，这相当于给虚拟计算机加电。

2. **CPU 初始化**：
   - CPU 的所有通用寄存器初始化为零，程序计数器（PC）指向地址 `0x1000`，这里存储着固化在硬件中的一小段引导代码。

3. **引导代码执行**：
   - 引导代码从地址 `0x1000` 开始执行，并很快跳转到地址 `0x80000000` 的 RustSBI 处。

4. **RustSBI 初始化**：
   - RustSBI 是一个 RISC-V 的 SBI（Supervisor Binary Interface）实现。它负责完成硬件初始化，并提供基本的服务（如关机和输出字符等）。
   - 初始化完成后，RustSBI 会跳转到操作系统内核的入口地址 `0x80200000`，开始执行内核的第一条指令。

![image-20240514160944976](./assets/image-20240514160944976.png)

### RustSBI 介绍

**RustSBI 是什么？**

- **SBI（Supervisor Binary Interface）**：RISC-V 的一种底层规范，用于在不同特权级之间提供标准化的接口。它类似于 x86 架构中的 BIOS 或者 ARM 架构中的 SMC（Secure Monitor Call）。
- **RustSBI**：是 SBI 的一个实现，用 Rust 编写。它运行在最高特权级（Machine 模式），提供一些基本服务，如关机、字符输出等。

**操作系统内核与 RustSBI 的关系**：

- 操作系统内核运行在更低的特权级（Supervisor 模式），并依赖于 RustSBI 提供的一些基本服务。
- 这种关系类似于应用程序依赖操作系统内核提供服务的关系，只不过 RustSBI 提供的服务更少且更底层。

通过 QEMU 模拟器，我们可以模拟 RISC-V 64 架构的计算机，并加载和执行内核程序。使用 RustSBI 作为 BootLoader，完成硬件初始化后，跳转到内核入口地址执行内核代码。这种机制使得我们能够在虚拟环境中开发和测试裸机程序，了解和验证其启动过程和运行行为。

## 6.2 实现关机功能

### 实现关机功能

为了在裸机环境中实现关机功能，我们需要通过 `ecall` 调用 RustSBI 提供的关机服务。下面是具体的实现和详细的解释。

#### 代码讲解

##### 1. SBI 调用函数 `sbi_call`

首先，我们实现一个通用的 SBI 调用函数 `sbi_call`：

```rust
// os/src/sbi.rs
fn sbi_call(which: usize, arg0: usize, arg1: usize, arg2: usize) -> usize {
    let mut ret;
    unsafe {
        core::arch::asm!(
            "ecall",  // 发出系统调用
            inlateout("a0") arg0 => ret,  // 将 arg0 放入 a0 寄存器，返回值存入 ret
            in("a1") arg1,               // 将 arg1 放入 a1 寄存器
            in("a2") arg2,               // 将 arg2 放入 a2 寄存器
            in("a7") which,              // 将调用号放入 a7 寄存器
        );
    }
    ret
}
```

- **`sbi_call` 函数**：用于发出 SBI 调用。
  - **参数**：
    - `which`：SBI 调用号。
    - `arg0`、`arg1`、`arg2`：传递给 SBI 调用的参数。
  - **返回值**：调用的返回值。

- **`ecall` 指令**：发出一个系统调用。这个指令用于从一个特权级切换到另一个特权级。

- **寄存器使用**：
  - `a0`：第一个参数，同时也是返回值。
  - `a1`、`a2`：第二个和第三个参数。
  - `a7`：调用号。

##### 2. 实现关机功能

我们定义一个常量 `SBI_SHUTDOWN` 表示关机的调用号，并实现 `shutdown` 函数：

```rust
const SBI_SHUTDOWN: usize = 8;

pub fn shutdown() -> ! {
    sbi_call(SBI_SHUTDOWN, 0, 0, 0);
    panic!("It should shutdown!");
}
```

- **`SBI_SHUTDOWN`**：关机调用号，为 8。
- **`shutdown` 函数**：
  - 调用 `sbi_call(SBI_SHUTDOWN, 0, 0, 0)` 发出关机请求。
    - RustSBI 在 Machine Mode 运行，负责处理系统调用。它识别到调用号为 `SBI_SHUTDOWN` 后，会执行关机操作。
  - 如果关机失败，程序会继续执行并触发 `panic!`，表示关机应该成功。

##### 3. 程序入口 `_start`

在 `main.rs` 中调用 `shutdown` 函数：

```rust
// os/src/main.rs
#[no_mangle]
extern "C" fn _start() {
    shutdown();
}
```

- **`#[no_mangle]`**：防止编译器对函数名进行重整，使得函数名保持为 `_start`。
- **`extern "C"`**：指定函数使用 C 语言调用约定。
- **`_start` 函数**：程序入口，调用 `shutdown` 函数以执行关机操作。

#### 应用程序访问系统调用

- **`ecall` 指令**：
  - 应用程序通过 `ecall` 指令访问操作系统提供的系统调用。
  - 操作系统通过 `ecall` 指令访问 RustSBI 提供的 SBI 调用。
  - 虽然指令相同，但它们所在的特权级不同：应用程序在用户特权级（User Mode），操作系统在内核特权级（Supervisor Mode），RustSBI 在机器特权级（Machine Mode）。

### 编译和执行

#### 编译生成 ELF 格式的执行文件

```bash
$ cargo build --release
 Compiling os v0.1.0 (/path/to/os)
  Finished release [optimized] target(s) in 0.15s
```

#### 转换 ELF 执行文件为二进制文件

```bash
$ rust-objcopy --binary-architecture=riscv64 target/riscv64gc-unknown-none-elf/release/os --strip-all -O binary target/riscv64gc-unknown-none-elf/release/os.bin
```

- **`rust-objcopy`**：用于处理对象文件的工具。
  - `--binary-architecture=riscv64`：指定目标架构为 RISC-V 64。
  - `--strip-all`：移除所有符号表信息。
  - `-O binary`：输出格式为纯二进制。

#### 加载并运行内核

```bash
$ qemu-system-riscv64 -machine virt -nographic -bios ../bootloader/rustsbi-qemu.bin -device loader,file=target/riscv64gc-unknown-none-elf/release/os.bin,addr=0x80200000
```

- **`-bios ../bootloader/rustsbi-qemu.bin`**：指定 RustSBI 引导加载程序。
- **`-device loader,file=target/riscv64gc-unknown-none-elf/release/os.bin,addr=0x80200000`**：将内核二进制文件加载到地址 `0x80200000`。

### 问题：程序入口地址设置

在尝试运行程序时，程序陷入死循环，可能是由于程序入口地址设置不正确导致的。通过 `rust-readobj` 分析可执行程序，可以发现其入口地址不是 RustSBI 约定的 `0x80200000` 。我们需要修改程序的内存布局并设置好栈空间。



## 6.3 设置正确的程序内存布局

为了使生成的可执行文件符合我们的预期内存布局，我们需要通过链接脚本（Linker Script）调整链接器的行为。我们可以通过修改 Cargo 配置文件来使用我们自定义的链接脚本 `os/src/linker.ld`。

#### 修改 Cargo 配置文件

首先，编辑 `os/.cargo/config` 文件，指向我们的链接脚本：

```toml
# os/.cargo/config
[build]
target = "riscv64gc-unknown-none-elf"

[target.riscv64gc-unknown-none-elf]
rustflags = [
    "-Clink-arg=-Tsrc/linker.ld", "-Cforce-frame-pointers=yes"
]
```

- **`-Clink-arg=-Tsrc/linker.ld`**：指定链接器使用 `src/linker.ld` 链接脚本。
- **`-Cforce-frame-pointers=yes`**：强制生成帧指针，便于调试。

#### 链接脚本 `os/src/linker.ld`

下面是我们的链接脚本，它定义了程序的内存布局：

```ld
/* os/src/linker.ld */
OUTPUT_ARCH(riscv)
ENTRY(_start)
BASE_ADDRESS = 0x80200000;

SECTIONS
{
    . = BASE_ADDRESS;
    skernel = .;

    stext = .;
    .text : {
        *(.text.entry)
        *(.text .text.*)
    }

    . = ALIGN(4K);
    etext = .;
    srodata = .;
    .rodata : {
        *(.rodata .rodata.*)
    }

    . = ALIGN(4K);
    erodata = .;
    sdata = .;
    .data : {
        *(.data .data.*)
    }

    . = ALIGN(4K);
    edata = .;
    .bss : {
        *(.bss.stack)
        sbss = .;
        *(.bss .bss.*)
    }

    . = ALIGN(4K);
    ebss = .;
    ekernel = .;

    /DISCARD/ : {
        *(.eh_frame)
    }
}
```

##### 关键内容解释

1. **`OUTPUT_ARCH(riscv)`**：
   - 设置目标架构为 RISC-V。

2. **`ENTRY(_start)`**：
   - 定义程序的入口点为 `_start`。

3. **`BASE_ADDRESS = 0x80200000;`**：
   - 定义一个常量 `BASE_ADDRESS`，其值为 `0x80200000`。这是 RustSBI 期望的操作系统起始地址。

4. **段定义**：
   - **`. = BASE_ADDRESS;`**：设置程序的起始地址。
   - **`skernel = .;`**：定义一个标记 `skernel`，表示内核的起始地址。
   - **`.text` 段**：包含代码段，所有代码（包括入口代码）都放在这里。
   - **`.rodata` 段**：只读数据段，包含只读数据。
   - **`.data` 段**：数据段，包含已初始化的数据。
   - **`.bss` 段**：未初始化的数据段，包含所有未初始化的数据。

5. **对齐和段结束地址**：
   - **`. = ALIGN(4K);`**：将段地址对齐到 4KB 边界。
   - **`etext`、`erodata`、`edata`、`ebss`**：这些符号表示各段的结束地址。

6. **丢弃段**：
   - **`/DISCARD/ : { *(.eh_frame) }`**：丢弃 `.eh_frame` 段，避免不必要的信息占用空间。

### 编译并使用新链接脚本

执行以下命令进行编译和转换：

```bash
$ cargo build --release
$ rust-objcopy --binary-architecture=riscv64 target/riscv64gc-unknown-none-elf/release/os --strip-all -O binary target/riscv64gc-unknown-none-elf/release/os.bin
$ qemu-system-riscv64 -machine virt -nographic -bios ../bootloader/rustsbi-qemu.bin -device loader,file=target/riscv64gc-unknown-none-elf/release/os.bin,addr=0x80200000
```

通过以上步骤，我们确保程序的内存布局符合预期，并设置了正确的入口地址和栈空间，使程序能够正常启动和运行。



## 6.4 正确配置栈空间布局

在裸机环境中运行操作系统时，必须正确配置栈空间以确保程序正常执行。我们通过汇编代码初始化栈空间，并在 Rust 代码中引用这段汇编代码来实现。

#### 汇编代码初始化栈空间

首先，我们在 `os/src/entry.asm` 文件中编写汇编代码，用于初始化栈空间：

```assembly
// os/src/entry.asm
.section .text.entry
.globl _start
_start:
    la sp, boot_stack_top
    call rust_main

.section .bss.stack
.globl boot_stack
boot_stack:
    .space 4096 * 16
.globl boot_stack_top
boot_stack_top:
```

##### 代码详解

1. **定义代码段和入口点**
   ```assembly
   .section .text.entry
   .globl _start
   _start:
       la sp, boot_stack_top
       call rust_main
   ```
   - **`.section .text.entry`**：定义代码段 `.text.entry`，用于放置入口代码。
   - **`.globl _start`**：声明 `_start` 为全局符号，即程序的入口点。
   - **`_start` 标签**：入口点标签，程序从这里开始执行。
   - **`la sp, boot_stack_top`**：将栈指针 `sp` 设置为栈顶 `boot_stack_top`。
   - **`call rust_main`**：调用 `rust_main` 函数。

2. **定义栈空间段**
   ```assembly
   .section .bss.stack
   .globl boot_stack
   boot_stack:
       .space 4096 * 16
   .globl boot_stack_top
   boot_stack_top:
   ```
   - **`.section .bss.stack`**：定义栈空间段 `.bss.stack`。
   - **`.globl boot_stack`**：声明 `boot_stack` 为全局符号，即栈底。
   - **`boot_stack:` 标签**：栈底标签。
   - **`.space 4096 * 16`**：预留 64KB 的栈空间。
   - **`.globl boot_stack_top`**：声明 `boot_stack_top` 为全局符号，即栈顶。

#### 嵌入汇编代码并声明应用入口

接下来，在 `os/src/main.rs` 文件中嵌入汇编代码，并声明应用入口 `rust_main`：

```rust
// os/src/main.rs
#![no_std]
#![no_main]

mod lang_items;

core::arch::global_asm!(include_str!("entry.asm"));

#[no_mangle]
pub fn rust_main() -> ! {
    shutdown();
}
```

##### 代码详解

1. **禁用标准库和入口点**
   ```rust
   #![no_std]
   #![no_main]
   ```
   - **`#![no_std]`**：禁用标准库，因为我们在裸机环境中运行。
   - **`#![no_main]`**：禁用默认入口点 `main`，我们将使用自定义入口 `_start`。

2. **导入必要的模块**
   ```rust
   mod lang_items;
   ```

3. **嵌入汇编代码**
   ```rust
   core::arch::global_asm!(include_str!("entry.asm"));
   ```
   - 使用 `global_asm!` 宏，将同目录下的 `entry.asm` 文件中的汇编代码嵌入到 Rust 代码中。

4. **声明应用入口 `rust_main`**
   ```rust
   #[no_mangle]
   pub fn rust_main() -> ! {
       shutdown();
   }
   ```
   - **`#[no_mangle]`**：防止编译器对函数名进行重整，使函数名保持为 `rust_main`。
   - **`rust_main` 函数**：应用入口，调用 `shutdown` 函数执行关机操作。

#### 编译、生成和运行

通过以下命令进行编译、生成和运行，我们可以看到 QEMU 模拟的 RISC-V 64 计算机优雅地退出：

```bash
$ cargo build --release
$ rust-objcopy --binary-architecture=riscv64 target/riscv64gc-unknown-none-elf/release/os --strip-all -O binary target/riscv64gc-unknown-none-elf/release/os.bin
$ qemu-system-riscv64 -machine virt -nographic -bios ../bootloader/rustsbi-qemu.bin -device loader,file=target/riscv64gc-unknown-none-elf/release/os.bin,addr=0x80200000
```

### 结果分析

运行结果如下：

```bash
[rustsbi] Version 0.1.0
.______       __    __      _______.___________.  _______..______   __
|   _  \     |  |  |  |    /       |           | /       ||   _  \ |  |
|  |_)  |    |  |  |  |   |   (----`---|  |----`|   (----`|  |_)  ||  |
|      /     |  |  |  |    \   \       |  |      \   \    |   _  < |  |
|  |\  \----.|  `--'  |.----)   |      |  |  .----)   |   |  |_)  ||  |
| _| `._____| \______/ |_______/       |__|  |_______/    |______/ |__|

[rustsbi] Platform: QEMU
[rustsbi] misa: RV64ACDFIMSU
[rustsbi] mideleg: 0x222
[rustsbi] medeleg: 0xb1ab
[rustsbi] Kernel entry: 0x80200000
```

- **QEMU 输出**：显示了 RustSBI 的版本和平台信息，表明模拟器成功加载并运行了我们的操作系统二进制文件。
- **优雅退出**：调用 `shutdown` 函数后，QEMU 模拟的计算机优雅地退出，表示我们的配置和程序运行成功。

通过上述步骤，我们成功配置了栈空间布局，并确保程序正确运行和关机。具体来说，我们通过编写汇编代码初始化栈空间，并在 Rust 代码中嵌入这段汇编代码，最终实现了在裸机环境中运行并优雅退出的功能。这种方法不仅确保了程序的正确执行，还为后续更复杂的系统功能实现打下了基础。

## 6.5 清空 .bss 段

### 清空 `.bss` 段

在裸机编程中，`.bss` 段用于存储未初始化的全局变量和静态变量。在程序启动时，通常需要将 `.bss` 段清零，以确保这些变量初始为零。我们将在 `rust_main` 函数中添加清零 `.bss` 段的功能。

### 代码实现

首先，在 `os/src/main.rs` 文件中添加清零 `.bss` 段的函数 `clear_bss`：

```rust
// os/src/main.rs
fn clear_bss() {
    extern "C" {
        fn sbss();
        fn ebss();
    }
    (sbss as usize..ebss as usize).for_each(|a| {
        unsafe { (a as *mut u8).write_volatile(0) }
    });
}

#[no_mangle]
pub fn rust_main() -> ! {
    clear_bss();
    shutdown();
}
```

### 代码详解

1. **定义 `clear_bss` 函数**：
   ```rust
   fn clear_bss() {
       extern "C" {
           fn sbss();
           fn ebss();
       }
       (sbss as usize..ebss as usize).for_each(|a| {
           unsafe { (a as *mut u8).write_volatile(0) }
       });
   }
   ```
   - **`extern "C"` 块**：声明外部符号 `sbss` 和 `ebss`，它们由链接脚本定义，表示 `.bss` 段的起始和结束地址。
   - **遍历 `.bss` 段**：使用 Rust 的范围语法 `(sbss as usize..ebss as usize)` 遍历 `.bss` 段的每个地址。
   - **清零 `.bss` 段**：对每个地址，使用 `write_volatile(0)` 将其写为零。`write_volatile` 确保编译器不会优化掉这段代码。

2. **修改 `rust_main` 函数**：
   ```rust
   #[no_mangle]
   pub fn rust_main() -> ! {
       clear_bss();
       shutdown();
   }
   ```
   - **调用 `clear_bss`**：在执行关机前调用 `clear_bss` 函数，以清零 `.bss` 段。
   - **调用 `shutdown`**：执行关机操作。

### 链接脚本中的全局符号

在 `linker.ld` 文件中，我们需要确保定义了 `sbss` 和 `ebss` 符号，表示 `.bss` 段的起始和结束地址。

从 `BASE_ADDRESS` 开始，代码段 `.text`, 只读数据段 `.rodata`，数据段 `.data`, bss 段 `.bss` 由低到高依次放置， 且每个段都有两个全局变量给出其起始和结束地址（比如 `.text` 段的开始和结束地址分别是 `stext` 和 `etext` ）。

参见上面的 [linker.ld](#链接脚本-ossrclinkerld)

## 6.6 添加裸机打印相关函数

### 6.6.1 `os/src/console.rs`

在上一节中我们为用户态程序实现的 `println` 宏，略作修改即可用于本节的内核态操作系统。 详见[ `os/src/console.rs`](https://github.com/LearningOS/rCore-Tutorial-Code-2024S/blob/ch1/os/src/console.rs)。

讲解参见 [1. console.rs补充讲解](https://lzzs.fun/zh/blog/bare-metal/4-Writing-the-main-program#heading-23) , [2. 有显示支持的用户态执行环境](#52-有显示支持的用户态执行环境)



### 6.6.2 `panic` | `os/src/lang_items.rs`

利用 `println` 宏，我们重写异常处理函数 `panic`，使其在 panic 时能打印错误发生的位置。 相关代码位于 [`os/src/lang_items.rs`](https://github.com/LearningOS/rCore-Tutorial-Code-2024S/blob/ch1/os/src/lang_items.rs) 中。

讲解参见 [lang_items.rs补充讲解](https://lzzs.fun/zh/blog/bare-metal/4-Writing-the-main-program#heading-38)



### 6.6.3 `os/src/logging.rs`

我们还使用第三方库 `log` 为你实现了日志模块，相关代码位于 [`os/src/logging.rs`](https://github.com/LearningOS/rCore-Tutorial-Code-2024S/blob/ch1/os/src/logging.rs) 中。

在 cargo 项目中引入外部库 log，需要修改 `Cargo.toml` 加入相应的依赖信息。

```rust
[dependencies]
log = "0.4"
```



讲解参见 [logging.rs补充讲解](https://lzzs.fun/zh/blog/bare-metal/4-Writing-the-main-program#heading-31)

---

现在，让我们重复一遍本章开头的试验，`make run LOG=TRACE`！

![image-20240514173019226](./assets/image-20240514173019226.png)
