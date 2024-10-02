# GDB安装

安装 GDB 流程来自[【笔记】rCore (RISC-V)：GDB 使用记录 ](https://zjp-cn.github.io/posts/rcore-gdb/)-  *2024/04/25*  - *[苦瓜小仔](https://github.com/zjp-CN)*。

以下是对整个流程的详细描述和讲解：

## 1. 安装依赖项

首先，我们需要为编译 GDB 安装一些必要的依赖项。`libncurses5-dev` 是为终端用户界面 (TUI) 提供支持的库，其他依赖项包括 `texinfo` 和 `libreadline-dev`，它们用于支持命令行输入以及生成文档。

```bash
sudo apt-get install libncurses5-dev texinfo libreadline-dev
```

在这里，Python 和 Python 开发库（`python-dev`）并不严格要求是 Python 2，也可以是 Python 3。只要你的系统中有 Python 3，它可以正常编译和运行。

> ### 使用 Python 3 进行配置
>
> 由于 Python 2 逐步被淘汰，建议你使用 Python 3。请确保系统中已经安装了 Python 3，并且有对应的开发库 `python3-dev`。你可以通过以下命令安装这些包：
>
> ```bash
> sudo apt-get install python3 python3-dev
> ```

## 2. 检查本地 Python 路径

在编译 GDB 时，需要确保 Python 的路径正确。你可以通过以下命令来检查 Python3 的路径：

```bash
which python3
```

也可以通过以下命令查看详细的 Python 路径信息，确认是否链接到 Python 3：

```bash
ll $(which python3)
```

例如，输出可能是：

```
/usr/bin/python3 -> python3.8
```

## 3. 下载最新的 GDB 源码

从清华大学的开源镜像站下载 GDB 源码。确保下载的是最新版本，比如 `gdb-14.2.tar.xz`：

```bash
wget https://mirrors.tuna.tsinghua.edu.cn/gnu/gdb/gdb-14.2.tar.xz
```

## 4. 解压源码文件

解压下载好的压缩文件。你可以使用标准的 `tar` 命令，或者使用其他工具（如 `ouch`）来解压文件。

```bash
tar -xvf gdb-14.2.tar.xz
```

或

```bash
ouch d gdb-14.2.tar.xz
```

解压后，源码文件将位于当前目录的 `gdb-14.2` 文件夹中。

## 5. 创建构建目录

进入 `gdb-14.2` 目录，并在该目录下创建一个新的目录来存放构建结果和生成的二进制文件：

```bash
cd gdb-14.2
mkdir build-riscv64
```

## 6. 配置编译参数

进入创建的构建目录 `build-riscv64`，并通过 `configure` 脚本配置编译参数。指定安装路径、Python 路径、目标架构（RISC-V），并启用 TUI（终端用户界面）支持：

```bash
cd build-riscv64

../configure --prefix=/root/qemu/gdb-14.2/build-riscv64 --with-python=/usr/bin/python3 --target=riscv64-unknown-elf --enable-tui=yes
```

- `../configure`: `configure`在`gdb-14.2`目录。
- `--prefix=/root/qemu/gdb-14.2/build-riscv64`：指定安装路径。
- `--with-python=/usr/bin/python3`：指定 Python 路径。
- `--target=riscv64-unknown-elf`：指定目标平台（RISC-V 架构）。
- `--enable-tui=yes`：启用 TUI 支持。

> 在编译 GDB（GNU 调试器）时，`configure` 脚本可以使用多种选项来指定编译过程中的配置和行为。这些选项可以用于指定编译目标、启用或禁用某些功能、配置依赖项路径等。下面详细介绍一些常见的 `configure` 配置参数及其作用。
>
> ### 1. `--prefix=PREFIX`
>
> - **作用**: 指定安装的根目录。编译后的 GDB 文件会被安装到该目录中。
>
> - **示例**:
>
>   ```bash
>   --prefix=/usr/local/gdb
>   ```
>
>   这将会把 GDB 安装到 `/usr/local/gdb`，其中可执行文件会放在 `bin/` 目录，库文件会放在 `lib/` 目录等。
>
> ### 2. `--target=TARGET`
>
> - **作用**: 指定要调试的目标架构。这是交叉编译的情况下非常重要的选项，定义你希望调试的架构类型。
>
> - **示例**:
>
>   ```bash
>   --target=riscv64-unknown-elf
>   ```
>
>   这指定了要生成支持 RISC-V 架构的 GDB。
>
> ### 3. `--with-python[=PYTHON]`
>
> - **作用**: 启用 Python 支持并可选择性地指定 Python 解释器的路径。GDB 支持 Python 插件，因此 Python 支持对于扩展 GDB 功能非常重要。
>
> - **默认**: 如果没有提供路径，GDB 会自动查找系统中的 Python。
>
> - **示例**:
>
>   ```bash
>   --with-python=/usr/bin/python3
>   ```
>
>   如果你不指定路径，你可以使用：
>
>   ```bash
>   --with-python=yes
>   ```
>
>   这会让 `configure` 脚本自动找到可用的 Python 解释器。
>
> ### 4. `--enable-tui`
>
> - **作用**: 启用 TUI（Text User Interface，文本用户界面）模式，这使得 GDB 提供类似于 `vi` 和 `emacs` 的命令行界面。
>
> - **示例**:
>
>   ```bash
>   --enable-tui
>   ```
>
>   启用 TUI 模式后，GDB 会有分屏窗口用于显示源代码和寄存器内容等，非常适合调试时查看复杂的程序状态。
>
> ### 5. `--disable-nls`
>
> - **作用**: 禁用 NLS（Native Language Support，本地语言支持），即禁用国际化功能。如果你不需要在调试器中显示非英语的本地化内容，可以选择禁用此选项。
>
> - **示例**:
>
>   ```bash
>   --disable-nls
>   ```
>
> ### 6. `--with-gmp=DIR` 和 `--with-mpfr=DIR`
>
> - **作用**: 指定 GMP 和 MPFR 的安装路径，GDB 需要这两个库来进行高精度数学计算。GMP 和 MPFR 是很多编译器和调试器的数学库依赖项。
>
> - **示例**:
>
>   ```bash
>   --with-gmp=/usr/local
>   --with-mpfr=/usr/local
>   ```
>
> ### 7. `--disable-werror`
>
> - **作用**: 在编译过程中，GDB 默认会将所有警告视为错误。启用这个选项可以避免警告被视为错误，从而使编译能够继续，即使出现了非致命性警告。
>
> - **示例**:
>
>   ```bash
>   --disable-werror
>   ```
>
> ### 8. `--enable-targets=TARGETS`
>
> - **作用**: 启用对多个目标架构的支持。GDB 可以在一个实例中支持多个不同的架构，因此可以同时调试不同平台的程序。
>
> - **示例**:
>
>   ```bash
>   --enable-targets=all
>   ```
>
>   这将启用所有支持的目标架构，但你也可以指定特定架构：
>
>   ```bash
>   --enable-targets=riscv64-unknown-elf,x86_64-pc-linux-gnu
>   ```
>
> ...

## 7. 编译并安装

使用 `make` 编译 GDB，并指定使用多核编译以提高编译速度：

```bash
make -j$(nproc)
```

接下来，运行 `make install` 安装编译好的二进制文件到指定的安装目录中：

```bash
make install
```

> 遇到权限问题使用：
>
> ```bash
> sudo make install
> ```
>
> 如果你不希望使用 `root` 权限或者不想将文件安装到 `/root` 目录，可以在配置时指定一个你有写权限的路径（例如：你的 `home` 目录）。你需要在 `configure` 阶段指定一个新的 `--prefix` 目录，如下所示：
>
> ```bash
> ../configure --prefix=$HOME/apps/gdb-14.2/build-riscv64 --with-python=/usr/bin/python3 --target=riscv64-unknown-elf --enable-tui=yes
> ```

## 8. 确认 GDB 二进制文件

生成的 GDB 二进制文件位于 `build-riscv64/bin/` 目录下。通过 `--version` 来检查  GDB 二进制文件（相对`build-riscv64`目录）：

```bash
./bin/riscv64-unknown-elf-gdb --version
```

## 9. 配置环境变量

你可以将该目录（ `build-riscv64/bin/`）添加到系统的 `PATH` 环境变量中，以便在终端中全局使用 GDB：

```bash
export PATH="/root/qemu/gdb-14.2/build-riscv64/bin:$PATH"
```

> 根据你选择的安装路径修改。

为了在每次登录时自动生效，可以将该行代码添加到你的 `~/.bashrc` 文件中，并执行以下命令使其立即生效：

```bash
source ~/.bashrc
```

确认环境变量，检查版本号：

```bash
riscv64-unknown-elf-gdb --version
```

## 10. 安装 GDB Dashboard 扩展

为了增强 GDB 的使用体验，可以安装 GDB Dashboard，它是一个用 Python 编写的启动扩展，提供了更多调试功能和用户界面。你可以使用以下命令将 GDB Dashboard 下载到 `~/.gdbinit` 文件中：

```bash
wget -P ~ https://github.com/cyrus-and/gdb-dashboard/raw/master/.gdbinit
```

安装完成后，每次启动 GDB 时，GDB Dashboard 都会自动加载并提供更加友好的调试界面。

通过以上步骤，你就成功编译并安装了 GDB，并且安装了 GDB Dashboard 扩展工具。你现在可以使用这个定制的 GDB 来调试 RISC-V 架构的项目。

---

> 等一下，安装好了，怎么用？
