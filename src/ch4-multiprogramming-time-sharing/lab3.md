# Lab3

待整理

---

## 第四讲 多道程序与分时多任务

### 代码树解释

```plaintext
── os
   ├── build.rs
   ├── Cargo.toml
   ├── Makefile
   └── src
       ├── batch.rs (移除：功能分别拆分到 loader 和 task 两个子模块)
       ├── config.rs (新增：保存内核的一些配置)
       ├── console.rs
       ├── logging.rs
       ├── sync
       ├── entry.asm
       ├── lang_items.rs
       ├── link_app.S
       ├── linker.ld
       ├── loader.rs (新增：将应用加载到内存并进行管理)
       ├── main.rs (修改：主函数进行了修改)
       ├── sbi.rs (修改：引入新的 sbi call set_timer)
       ├── syscall (修改：新增若干 syscall)
       │   ├── fs.rs
       │   ├── mod.rs
       │   └── process.rs
       ├── task (新增：task 子模块，主要负责任务管理)
       │   ├── context.rs (引入 Task 上下文 TaskContext)
       │   ├── mod.rs (全局任务管理器和提供给其他模块的接口)
       │   ├── switch.rs (将任务切换的汇编代码解释为 Rust 接口 __switch)
       │   ├── switch.S (任务切换的汇编代码)
       │   └── task.rs (任务控制块 TaskControlBlock 和任务状态 TaskStatus 的定义)
       ├── timer.rs (新增：计时器相关)
       └── trap
           ├── context.rs
           ├── mod.rs (修改：时钟中断相应处理)
           └── trap.S
```

### 详细解释

#### 顶层结构
- `build.rs`: 编译脚本，用于编译期间的自定义构建步骤。
- `Cargo.toml`: Rust 项目的配置文件，定义了项目的依赖和元数据。
- `Makefile`: 使用 `make` 构建项目的配置文件。

#### `src` 目录
- `batch.rs`: 原来负责批处理功能的模块，现已移除，其功能被拆分到 `loader` 和 `task` 两个子模块中。
- `config.rs`: 新增的模块，用于保存内核的一些配置。
- `console.rs`: 控制台相关功能。
- `logging.rs`: 日志记录相关功能。
- `sync`: 同步相关功能。
- `entry.asm`: 程序入口的汇编代码。
- `lang_items.rs`: 定义了一些语言项目（如 panic 处理）。
- `link_app.S`: 用于链接用户应用程序的汇编代码。
- `linker.ld`: 链接脚本，用于定义内存布局。
- `loader.rs`: 新增的模块，负责将应用加载到内存并进行管理。
- `main.rs`: 主函数，进行了修改以适应新增和修改的功能。
- `sbi.rs`: 修改后的模块，引入了新的 SBI 调用 `set_timer`。
- `syscall`: 系统调用相关功能，新增了若干系统调用。
  - `fs.rs`: 文件系统相关系统调用。
  - `mod.rs`: 系统调用模块入口。
  - `process.rs`: 进程相关系统调用。
- `task`: 新增的任务管理子模块。
  - `context.rs`: 引入了任务上下文 `TaskContext`。
  - `mod.rs`: 定义了全局任务管理器并提供接口给其他模块。
  - `switch.rs`: 将任务切换的汇编代码解释为 Rust 接口 `__switch`。
  - `switch.S`: 任务切换的汇编代码。
  - `task.rs`: 定义了任务控制块 `TaskControlBlock` 和任务状态 `TaskStatus`。
- `timer.rs`: 新增的计时器相关模块。
- `trap`: 中断和异常处理相关功能。
  - `context.rs`: 中断和异常处理上下文。
  - `mod.rs`: 修改了时钟中断的相应处理。
  - `trap.S`: 中断和异常处理的汇编代码。

通过这些模块的拆分和新增，内核实现了多道程序与分时多任务的功能，提高了系统的并发能力和响应速度。

---

我们将详细深入讲解 `loader.rs` 文件中的 `load_apps` 函数每个细节。

### `loader.rs` 文件

```rust
// os/src/loader.rs

pub fn load_apps() {
    extern "C" {
        fn _num_app();
    }
    let num_app_ptr = _num_app as usize as *const usize;
    let num_app = unsafe { num_app_ptr.read() };
    let app_start = unsafe { core::slice::from_raw_parts(num_app_ptr.add(1), num_app + 1) };
    // clear i-cache first
    unsafe {
        core::arch::asm!("fence.i");
    }
    // load apps
    for i in 0..num_app {
        let base_i = get_base_i(i);
        // clear region
        (base_i..base_i + APP_SIZE_LIMIT)
            .for_each(|addr| unsafe { (addr as *mut u8).write_volatile(0) });
        // load app from data section to memory
        let src = unsafe {
            core::slice::from_raw_parts(app_start[i] as *const u8, app_start[i + 1] - app_start[i])
        };
        let dst = unsafe { core::slice::from_raw_parts_mut(base_i as *mut u8, src.len()) };
        dst.copy_from_slice(src);
    }
}
```

### 详细解释

#### 1. 获取应用程序数量

```rust
extern "C" {
    fn _num_app();
}
let num_app_ptr = _num_app as usize as *const usize;
let num_app = unsafe { num_app_ptr.read() };
```

- **`extern "C"`**: 这里声明了一个外部函数 `_num_app`，它的定义在其他地方（通常是汇编代码中）。
- **`_num_app as usize as *const usize`**: 将 `_num_app` 的函数指针转换为 `usize` 类型，再转换为指向 `usize` 的指针。
- **`num_app_ptr.read()`**: 读取指针指向的值，即应用程序的数量。由于这个操作涉及到裸指针，需要使用 `unsafe` 块来保证内存安全。

#### 2. 获取应用程序起始地址数组

```rust
let app_start = unsafe { core::slice::from_raw_parts(num_app_ptr.add(1), num_app + 1) };
```

- **`core::slice::from_raw_parts`**: 将一个原始指针和长度转换为一个切片。
- **`num_app_ptr.add(1)`**: 指针向后移动一位，跳过应用程序数量的存储位置，指向应用程序起始地址数组的开始。
- **`num_app + 1`**: 切片的长度为 `num_app + 1`，包括所有应用程序的起始地址和结束位置。

#### 3. 清除指令缓存

```rust
unsafe {
    core::arch::asm!("fence.i");
}
```

- **`core::arch::asm!("fence.i")`**: 使用内联汇编清除指令缓存（I-cache），确保后续加载的指令能够被正确执行。`fence.i` 是 RISC-V 指令，用于指令序列之间的隔离。

#### 4. 加载应用程序

```rust
for i in 0..num_app {
    let base_i = get_base_i(i);
    // clear region
    (base_i..base_i + APP_SIZE_LIMIT)
        .for_each(|addr| unsafe { (addr as *mut u8).write_volatile(0) });
    // load app from data section to memory
    let src = unsafe {
        core::slice::from_raw_parts(app_start[i] as *const u8, app_start[i + 1] - app_start[i])
    };
    let dst = unsafe { core::slice::from_raw_parts_mut(base_i as *mut u8, src.len()) };
    dst.copy_from_slice(src);
}
```

- **`for i in 0..num_app`**: 遍历每个应用程序。
- **`get_base_i(i)`**: 计算每个应用程序的起始地址 `base_i`。

##### 清除目标内存区域

```rust
(base_i..base_i + APP_SIZE_LIMIT)
    .for_each(|addr| unsafe { (addr as *mut u8).write_volatile(0) });
```

- **`(base_i..base_i + APP_SIZE_LIMIT)`**: 创建从 `base_i` 到 `base_i + APP_SIZE_LIMIT` 的地址范围。
- **`for_each`**: 对该范围内的每个地址执行操作。
- **`(addr as *mut u8).write_volatile(0)`**: 将地址 `addr` 转换为可变指针，并写入 `0`。使用 `write_volatile` 确保编译器不会优化掉这段代码，确保每个字节都被清零。

##### 加载应用程序数据

```rust
let src = unsafe {
    core::slice::from_raw_parts(app_start[i] as *const u8, app_start[i + 1] - app_start[i])
};
let dst = unsafe { core::slice::from_raw_parts_mut(base_i as *mut u8, src.len()) };
dst.copy_from_slice(src);
```

- **`core::slice::from_raw_parts`**: 将应用程序在数据段中的起始地址转换为切片。
- **`src`**: 来源数据切片，表示应用程序的二进制数据。
- **`core::slice::from_raw_parts_mut`**: 将目标地址转换为可变切片。
- **`dst`**: 目标数据切片，表示应用程序在内存中的位置。
- **`copy_from_slice(src)`**: 将 `src` 中的数据复制到 `dst`。

#### `get_base_i` 函数

每个应用程序被加载到以物理地址 `base_i` 开头的一段物理内存上，而 `base_i` 的计算方式如下：

```rust
fn get_base_i(app_id: usize) -> usize {
    APP_BASE_ADDRESS + app_id * APP_SIZE_LIMIT
}
```

- **`APP_BASE_ADDRESS`**: 应用程序的基地址，通常在 `config` 模块中定义。这里设置为 `0x80400000`。
- **`APP_SIZE_LIMIT`**: 每个应用程序的内存大小限制。这里设置为 `0x20000`。
- **`APP_BASE_ADDRESS + app_id * APP_SIZE_LIMIT`**: 计算第 `app_id` 个应用程序的起始地址，确保每个应用程序都有一个独立的内存区域。

#### `config.rs` 文件中的常数定义

```rust
// os/src/config.rs

pub const APP_BASE_ADDRESS: usize = 0x80400000;
pub const APP_SIZE_LIMIT: usize = 0x20000;
```

- **`APP_BASE_ADDRESS`**: 基地址，设置为 `0x80400000`。
- **`APP_SIZE_LIMIT`**: 每个应用程序的大小限制，设置为 `0x20000`。

通过这些步骤，内核实现了多道程序的加载和执行，为系统带来了并发处理能力。每个应用程序都有独立的内存区域，确保了它们可以同时驻留在内存中并被正确执行。

---

### 多道程序放置与加载中的硬件和操作系统流程

#### 1. 获取应用程序数量

- **代码位置**: `loader.rs`
- **函数名**: `load_apps`

```rust
extern "C" {
    fn _num_app();
}
let num_app_ptr = _num_app as usize as *const usize;
let num_app = unsafe { num_app_ptr.read() };
```

**硬件状态**:
- **内存读取**: CPU 从固定的内存位置读取应用程序数量。
- **内存位置**: 该位置由 `_num_app` 函数指向，可能是由汇编代码或链接器设置的一个标记位置。

**操作系统状态**:
- **内核数据准备**: 内核通过读取 `num_app_ptr` 获取应用程序数量并存储在 `num_app` 变量中。
- **内核流程**: 准备遍历和加载多个应用程序。

#### 2. 获取应用程序起始地址数组

- **代码位置**: `loader.rs`
- **函数名**: `load_apps`

```rust
let app_start = unsafe { core::slice::from_raw_parts(num_app_ptr.add(1), num_app + 1) };
```

**硬件状态**:
- **内存读取**: CPU 访问内存中存储的应用程序起始地址数组。
- **内存位置**: 起始地址数组从 `num_app_ptr` 开始的下一个位置开始。

**操作系统状态**:
- **内核数据准备**: 内核通过 `core::slice::from_raw_parts` 获取应用程序起始地址和结束地址，存储在 `app_start` 切片中。
- **内核流程**: 准备按照这些地址加载应用程序。

#### 3. 清除指令缓存

- **代码位置**: `loader.rs`
- **函数名**: `load_apps`

```rust
unsafe {
    core::arch::asm!("fence.i");
}
```

**硬件状态**:
- **CPU 操作**: 执行 `fence.i` 指令，清除指令缓存（I-cache）。
- **指令缓存**: 确保指令缓存中的旧指令不会影响新加载的应用程序。

**操作系统状态**:
- **内核初始化**: 确保内存中的新指令可以被正确执行，准备加载应用程序。

#### 4. 加载应用程序

- **代码位置**: `loader.rs`
- **函数名**: `load_apps`

```rust
for i in 0..num_app {
    let base_i = get_base_i(i);
    // clear region
    (base_i..base_i + APP_SIZE_LIMIT)
        .for_each(|addr| unsafe { (addr as *mut u8).write_volatile(0) });
    // load app from data section to memory
    let src = unsafe {
        core::slice::from_raw_parts(app_start[i] as *const u8, app_start[i + 1] - app_start[i])
    };
    let dst = unsafe { core::slice::from_raw_parts_mut(base_i as *mut u8, src.len()) };
    dst.copy_from_slice(src);
}
```

**硬件状态**:
- **内存操作**: CPU 清除目标内存区域 (`base_i..base_i + APP_SIZE_LIMIT`) 并写入应用程序数据。
- **内存读取和写入**: CPU 从 `app_start` 读取应用程序数据，并写入 `base_i` 开始的内存区域。

**操作系统状态**:
- **内核数据准备**: 内核计算每个应用程序的基地址 (`get_base_i`)。
- **内核初始化**: 清除目标内存区域，确保没有残留数据。
- **内核加载**: 复制应用程序数据到目标内存区域，确保每个应用程序都在独立的内存区域中正确加载。

#### 5. 计算应用程序的基地址

- **代码位置**: `loader.rs`
- **函数名**: `get_base_i`

```rust
fn get_base_i(app_id: usize) -> usize {
    APP_BASE_ADDRESS + app_id * APP_SIZE_LIMIT
}
```

- **代码位置**: `config.rs`
- **常量定义**: `APP_BASE_ADDRESS` 和 `APP_SIZE_LIMIT`

```rust
pub const APP_BASE_ADDRESS: usize = 0x80400000;
pub const APP_SIZE_LIMIT: usize = 0x20000;
```

**硬件状态**:
- **内存布局**: 计算每个应用程序在物理内存中的起始地址。

**操作系统状态**:
- **内核地址计算**: 使用 `APP_BASE_ADDRESS` 和 `APP_SIZE_LIMIT`，通过应用程序编号 `app_id` 计算每个应用程序的基地址 `base_i`。
- **内存管理**: 确保每个应用程序都有独立的内存区域，防止地址冲突。

### 流程总结

1. **获取应用程序数量**:
   - **硬件**: 从固定内存位置读取数量。
   - **操作系统**: 读取并存储在 `num_app` 变量中。

2. **获取应用程序起始地址数组**:
   - **硬件**: 读取内存中的地址数组。
   - **操作系统**: 存储在 `app_start` 切片中。

3. **清除指令缓存**:
   - **硬件**: 执行 `fence.i` 指令，清除 I-cache。
   - **操作系统**: 确保新指令可以正确执行。

4. **加载应用程序**:
   - **硬件**: 清除目标内存区域并写入应用程序数据。
   - **操作系统**: 计算基地址，初始化内存，复制数据。

5. **计算应用程序基地址**:
   - **硬件**: 基于常量计算地址。
   - **操作系统**: 确保内存布局合理，防止地址冲突。

通过这些步骤，操作系统成功实现了多道程序的加载和运行，为系统提供了多任务并发处理的能力。每个应用程序都有独立的内存区域，确保它们可以同时驻留在内存中并被正确执行。

---

### 任务切换

任务切换是操作系统的核心机制之一，使得应用可以在运行中主动或被动地交出 CPU 的使用权，内核可以选择另一个程序继续执行。任务切换的关键在于保证用户程序在两次运行期间，任务上下文（如寄存器、栈等）保持一致。

#### 任务切换的设计与实现

任务切换与 Trap 控制流切换相比，有如下异同：
- **不同点**:
  - 不涉及特权级切换，部分由编译器完成。
- **相同点**:
  - 对应用是透明的。

任务切换实质上是来自两个不同应用在内核中的 Trap 控制流之间的切换。当一个应用 Trap 到 S 态 OS 内核中进行进一步处理时，其 Trap 控制流可以调用一个特殊的 `__switch` 函数。在 `__switch` 返回之后，Trap 控制流将继续从调用该函数的位置继续向下执行。

#### `__switch` 函数

任务切换通过 `__switch` 函数实现。在 `__switch` 函数中，保存 CPU 的某些寄存器，它们就是任务上下文 (`Task Context`)。

下面是 `__switch` 的实现：

```assembly
# os/src/task/switch.S

.altmacro
.macro SAVE_SN n
    sd s\n, (\n+2)*8(a0)
.endm
.macro LOAD_SN n
    ld s\n, (\n+2)*8(a1)
.endm
    .section .text
    .globl __switch
__switch:
    # __switch(
    #     current_task_cx_ptr: *mut TaskContext,
    #     next_task_cx_ptr: *const TaskContext
    # )
    # save kernel stack of current task
    sd sp, 8(a0)
    # save ra & s0~s11 of current execution
    sd ra, 0(a0)
    .set n, 0
    .rept 12
        SAVE_SN %n
        .set n, n + 1
    .endr
    # restore ra & s0~s11 of next execution
    ld ra, 0(a1)
    .set n, 0
    .rept 12
        LOAD_SN %n
        .set n, n + 1
    .endr
    # restore kernel stack of next task
    ld sp, 8(a1)
    ret
```

##### 流程细节

1. **函数调用**:
   - **函数名**: `__switch`
   - **参数**: 
     - `current_task_cx_ptr`（当前任务的上下文指针，通过寄存器 `a0` 传入）
     - `next_task_cx_ptr`（下一个任务的上下文指针，通过寄存器 `a1` 传入）

2. **保存当前任务上下文**:
   - **保存栈指针（sp）**: 
     ```assembly
     sd sp, 8(a0)
     ```
     - **硬件状态**: 将当前任务的栈指针 `sp` 保存到 `current_task_cx_ptr` 指向的内存位置。
     - **操作系统状态**: 当前任务的栈状态被保存。

   - **保存返回地址（ra）和保存寄存器（s0~s11）**:
     ```assembly
     sd ra, 0(a0)
     .set n, 0
     .rept 12
         SAVE_SN %n
         .set n, n + 1
     .endr
     ```
     - **硬件状态**: 将返回地址 `ra` 和保存寄存器 `s0~s11` 保存到 `current_task_cx_ptr` 指向的内存位置。
     - **操作系统状态**: 当前任务的寄存器状态被保存。

3. **恢复下一个任务上下文**:
   - **恢复返回地址（ra）和保存寄存器（s0~s11）**:
     ```assembly
     ld ra, 0(a1)
     .set n, 0
     .rept 12
         LOAD_SN %n
         .set n, n + 1
     .endr
     ```
     - **硬件状态**: 从 `next_task_cx_ptr` 指向的内存位置恢复返回地址 `ra` 和保存寄存器 `s0~s11`。
     - **操作系统状态**: 下一个任务的寄存器状态被恢复。

   - **恢复栈指针（sp）**:
     ```assembly
     ld sp, 8(a1)
     ```
     - **硬件状态**: 从 `next_task_cx_ptr` 指向的内存位置恢复栈指针 `sp`。
     - **操作系统状态**: 下一个任务的栈状态被恢复。

4. **返回（ret）**:
   - **硬件状态**: 返回到下一个任务的执行点。
   - **操作系统状态**: CPU 开始执行下一个任务。

#### 总结

**任务切换的关键步骤**:
1. **保存当前任务的上下文**:
   - 保存当前任务的栈指针 `sp` 和寄存器 `ra`, `s0~s11` 到 `current_task_cx_ptr`。
2. **恢复下一个任务的上下文**:
   - 从 `next_task_cx_ptr` 恢复下一个任务的栈指针 `sp` 和寄存器 `ra`, `s0~s11`。

**硬件状态变化**:
- **内存操作**: 多次读写内存，用于保存和恢复任务上下文。
- **寄存器操作**: 读写多个寄存器值，包括 `sp`, `ra`, `s0~s11`。

**操作系统状态变化**:
- **上下文切换**: 当前任务的上下文被保存，下一个任务的上下文被恢复。
- **任务执行**: 切换到下一个任务的执行点，继续执行下一个任务的代码。

通过 `__switch` 函数，内核能够有效地在不同的任务之间切换，确保每个任务在两次运行之间保持上下文一致。这是实现多任务并发运行的基础。

---

### 任务切换中的硬件和操作系统流程细节

任务切换是操作系统中的核心机制，它使得应用可以在运行中主动或被动地交出 CPU 的使用权，从而允许另一个程序继续执行。在这一过程中，操作系统需要确保用户程序两次运行期间，任务上下文（如寄存器、栈等）保持一致。

#### 代码位置和函数名
- **代码位置**: `os/src/task/switch.S`  `os/src/task/switch.rs`
- **函数名**: `__switch`
- **相关变量**: `current_task_cx_ptr`, `next_task_cx_ptr`

#### 详细流程

##### 1. 获取当前任务和下一个任务的上下文指针
- **操作系统状态**:
  - **函数调用**: `__switch(current_task_cx_ptr: *mut TaskContext, next_task_cx_ptr: *const TaskContext)`
  - **变量传递**: `current_task_cx_ptr` 和 `next_task_cx_ptr` 分别通过寄存器 `a0` 和 `a1` 传入。

##### 2. 保存当前任务的上下文

- **函数名**: `__switch`

1. **保存栈指针（sp）**:
   - **硬件状态**: CPU 将当前任务的栈指针 `sp` 保存到 `current_task_cx_ptr` 指向的内存位置（通过 `sd sp, 8(a0)`）。
   - **操作系统状态**: 当前任务的栈状态被保存。

2. **保存返回地址（ra）和保存寄存器（s0~s11）**:
   - **硬件状态**: CPU 将返回地址 `ra` 和保存寄存器 `s0~s11` 保存到 `current_task_cx_ptr` 指向的内存位置（通过 `sd ra, 0(a0)` 和 `SAVE_SN` 宏）。
   - **操作系统状态**: 当前任务的寄存器状态被保存。

##### 3. 恢复下一个任务的上下文

- **函数名**: `__switch`

1. **恢复返回地址（ra）和保存寄存器（s0~s11）**:
   - **硬件状态**: CPU 从 `next_task_cx_ptr` 指向的内存位置恢复返回地址 `ra` 和保存寄存器 `s0~s11`（通过 `ld ra, 0(a1)` 和 `LOAD_SN` 宏）。
   - **操作系统状态**: 下一个任务的寄存器状态被恢复。

2. **恢复栈指针（sp）**:
   - **硬件状态**: CPU 从 `next_task_cx_ptr` 指向的内存位置恢复栈指针 `sp`（通过 `ld sp, 8(a1)`）。
   - **操作系统状态**: 下一个任务的栈状态被恢复。

##### 4. 返回到下一个任务的执行点

- **函数名**: `__switch`
- **指令**: `ret`

- **硬件状态**: CPU 执行 `ret` 指令，跳转到恢复的返回地址 `ra`，开始执行下一个任务。
- **操作系统状态**: CPU 切换到下一个任务的执行点，继续执行下一个任务的代码。

### 总结

#### 硬件状态变化
- **内存读写**: 多次读写内存，用于保存和恢复任务上下文（寄存器和栈指针）。
- **寄存器操作**: 读写多个寄存器值，包括 `sp`, `ra`, `s0~s11`。
- **指令执行**: 执行 `ret` 指令，切换到下一个任务的执行点。

#### 操作系统状态变化
- **上下文切换**: 当前任务的上下文被保存，下一个任务的上下文被恢复。
- **任务执行**: 切换到下一个任务的执行点，继续执行下一个任务的代码。

通过 `__switch` 函数，操作系统实现了不同任务之间的切换，确保每个任务在两次运行之间保持上下文一致。这是实现多任务并发运行的基础。

---

### 任务上下文（TaskContext）和任务切换（__switch）的实现

#### 任务上下文（TaskContext）

在任务切换中，保存和恢复任务的上下文是关键。上下文保存了任务在切换前的状态，以便在切换回来时能从中断点继续执行。具体来说，上下文包括返回地址（ra）、栈指针（sp）、以及保存寄存器（s0~s11）。

#### TaskContext 结构体

```rust
// os/src/task/context.rs

#[repr(C)]
pub struct TaskContext {
    ra: usize,
    sp: usize,
    s: [usize; 12],
}
```

##### 详细解释

- **#[repr(C)]**: 这个属性保证结构体的内存布局与 C 语言兼容，以确保在汇编代码中能正确访问这些字段。
- **ra: usize**: 保存返回地址寄存器（Return Address）。`ra` 寄存器记录了函数返回后应该跳转到的地址。
- **sp: usize**: 保存栈指针寄存器（Stack Pointer）。`sp` 寄存器指向当前栈顶位置。
- **s: [usize; 12]**: 保存被调用者保存寄存器（Saved Registers）。`s0~s11` 是被调用者保存寄存器，在函数调用过程中需要保持它们的值。

#### __switch 的 Rust 封装

```rust
// os/src/task/switch.rs

core::arch::global_asm!(include_str!("switch.S"));

extern "C" {
    pub fn __switch(
        current_task_cx_ptr: *mut TaskContext,
        next_task_cx_ptr: *const TaskContext);
}
```

##### 详细解释

- **core::arch::global_asm!(include_str!("switch.S"))**:
  - 将 `switch.S` 中的汇编代码包含进来，确保汇编函数 `__switch` 可被 Rust 代码调用。
  
- **extern "C"**:
  - 声明外部函数 `__switch`，表示该函数是用 C ABI 调用的汇编函数。

- **pub fn __switch(current_task_cx_ptr: *mut TaskContext, next_task_cx_ptr: *const TaskContext)**:
  - 该函数接受两个参数：`current_task_cx_ptr` 和 `next_task_cx_ptr`，分别是当前任务和下一个任务的上下文指针。
  - `current_task_cx_ptr` 是一个可变指针，指向当前任务的上下文，表示需要保存的当前任务的寄存器状态。
  - `next_task_cx_ptr` 是一个不可变指针，指向下一个任务的上下文，表示需要恢复的下一个任务的寄存器状态。

通过 `TaskContext` 结构体和 `__switch` 汇编函数的实现，操作系统能够在不同任务之间进行切换，确保每个任务在切换过程中保持上下文一致。这是实现多任务并发运行的关键机制。

---

### 管理多道程序

内核需要管理多个任务以实现多道程序的并发执行。管理任务的关键在于维护任务信息，包括任务运行状态、任务控制块、以及任务相关的系统调用。

#### 任务运行状态

任务运行状态包括：
- **未初始化**: 任务尚未准备好执行。
- **准备执行**: 任务已经准备好，可以执行。
- **正在执行**: 任务当前正在 CPU 上执行。
- **已退出**: 任务已经完成，不再需要执行。

这些状态帮助内核跟踪每个任务的执行进度和调度需求。

#### 任务控制块（Task Control Block）

任务控制块 (TCB) 是用于维护每个任务的状态和上下文的结构体。TCB 包含了任务的上下文（如寄存器、栈指针等）以及任务的运行状态。

#### 任务相关系统调用

系统调用是用户程序与内核交互的接口，任务相关的系统调用包括：
- **主动暂停（sys_yield）**: 任务主动交出 CPU 使用权，让其他任务执行。
- **主动退出（sys_exit）**: 任务主动退出，表示任务已完成。

#### yield 系统调用

`sys_yield` 系统调用允许任务主动交出 CPU 使用权，让内核调度其他任务执行。这在多道程序中尤为重要，可以避免 CPU 资源的浪费。

```rust
// user/src/syscall.rs

pub fn sys_yield() -> isize {
    syscall(SYSCALL_YIELD, [0, 0, 0])
}

// user/src/lib.rs
// yield 是 Rust 的关键字
pub fn yield_() -> isize { sys_yield() }
```

##### `sys_yield` 系统调用流程

1. **调用 `sys_yield` 函数**:
   - **函数名**: `sys_yield`
   - **定义位置**: `user/src/syscall.rs`
   - **功能**: 应用主动交出 CPU 使用权，并切换到其他应用。
   - **返回值**: 总是返回 0。
   - **syscall ID**: 124

2. **用户库封装 `sys_yield`**:
   - **函数名**: `yield_`
   - **定义位置**: `user/src/lib.rs`
   - **功能**: 用户调用 `yield_` 函数，内部调用 `sys_yield` 实现。

##### 硬件和操作系统流程细节

1. **应用调用 `yield_` 函数**:
   - **操作系统状态**: 应用调用 `yield_`，实际上调用了 `sys_yield` 系统调用。

2. **`sys_yield` 系统调用实现**:
   - **操作系统状态**: 内核处理 `sys_yield` 系统调用，将当前任务的上下文保存到 TCB 中，并选择下一个任务执行。
   - **硬件状态**: 
     - **上下文切换**: 内核保存当前任务的寄存器和栈指针。
     - **CPU 调度**: 内核调度器选择下一个任务，将其上下文恢复到寄存器和栈指针。

3. **内核调度下一个任务**:
   - **操作系统状态**: 内核根据调度策略选择下一个任务，将其状态从 "准备执行" 切换为 "正在执行"。
   - **硬件状态**: 
     - **恢复上下文**: 内核恢复下一个任务的寄存器和栈指针。
     - **CPU 执行**: CPU 开始执行下一个任务。

#### 多道程序的典型执行情况

通过 `sys_yield` 系统调用，任务可以在需要等待外设返回结果时主动交出 CPU 使用权，让其他任务执行。以下是一个典型的多道程序执行过程：

1. **蓝色应用请求外设**:
   - **操作系统状态**: 蓝色应用向外设提交请求，外设开始工作，需要一段时间才能返回结果。
   - **硬件状态**: 外设开始处理请求。

2. **蓝色应用调用 `sys_yield`**:
   - **操作系统状态**: 蓝色应用调用 `sys_yield` 系统调用，主动交出 CPU 使用权。
   - **硬件状态**: 内核保存蓝色应用的上下文，调度绿色应用执行。

3. **绿色应用执行**:
   - **操作系统状态**: 内核将绿色应用的上下文恢复到寄存器，调度绿色应用执行。
   - **硬件状态**: CPU 开始执行绿色应用的代码。

4. **外设返回结果前的多次 `sys_yield`**:
   - **操作系统状态**: 蓝色应用在外设返回结果前多次调用 `sys_yield`，内核多次在蓝色应用和其他任务之间进行调度。
   - **硬件状态**: CPU 多次在不同任务之间切换。

5. **外设返回结果**:
   - **操作系统状态**: 蓝色应用最终等待到外设返回结果，可以继续执行。
   - **硬件状态**: CPU 继续执行蓝色应用的代码。

通过上述流程，操作系统实现了多道程序的并发执行，充分利用了 CPU 资源，提高了系统的响应速度和效率。

---

### 任务控制块与任务运行状态

任务控制块（Task Control Block，TCB）和任务运行状态是操作系统管理任务的核心组件。下面我们深入讲解它们的实现和作用。

#### 任务运行状态

任务运行状态用于描述任务在生命周期中的不同阶段。定义在 `os/src/task/task.rs` 中：

```rust
// os/src/task/task.rs

#[derive(Copy, Clone, PartialEq)]
pub enum TaskStatus {
    UnInit,  // 未初始化
    Ready,   // 准备运行
    Running, // 正在运行
    Exited,  // 已退出
}
```

##### 详细解释

- **UnInit**: 任务未初始化，尚未准备好执行。
- **Ready**: 任务已准备好，可以运行，但尚未开始执行。
- **Running**: 任务当前正在 CPU 上执行。
- **Exited**: 任务已经完成执行并退出。

这些状态帮助内核跟踪每个任务的执行进度和调度需求。

#### 任务控制块（Task Control Block）

任务控制块是用于维护每个任务状态和上下文的数据结构。定义在 `os/src/task/task.rs` 中：

```rust
// os/src/task/task.rs

#[derive(Copy, Clone)]
pub struct TaskControlBlock {
    pub task_status: TaskStatus,
    pub task_cx: TaskContext,
}
```

##### 详细解释

- **task_status**: 任务状态（TaskStatus），表示任务当前的运行状态。
- **task_cx**: 任务上下文（TaskContext），包含任务的寄存器和栈指针等信息。

#### 任务管理器

任务管理器是内核中用于管理多个任务控制块的全局结构。定义在 `os/src/task/mod.rs` 中：

```rust
// os/src/task/mod.rs

pub struct TaskManager {
    num_app: usize,
    inner: UPSafeCell<TaskManagerInner>,
}

struct TaskManagerInner {
    tasks: [TaskControlBlock; MAX_APP_NUM],
    current_task: usize,
}
```

##### 详细解释

- **TaskManager**
  - **num_app**: 应用程序数量，在 `TaskManager` 初始化后保持不变。
  - **inner**: 包含实际任务管理数据的结构体，用 `UPSafeCell` 包装以保证线程安全。

- **TaskManagerInner**
  - **tasks**: 任务控制块数组，包含所有任务的状态和上下文。
  - **current_task**: 当前正在执行的任务编号。

这种设计将不变的字段（如 `num_app`）和变化的字段（如 `tasks` 和 `current_task`）分离，保证了代码的可读性和维护性。

### 硬件和操作系统流程细节

#### 任务状态管理

1. **任务初始化**:
   - **操作系统状态**: 将任务状态设置为 `TaskStatus::UnInit`，准备初始化任务。

2. **任务准备运行**:
   - **操作系统状态**: 将任务状态设置为 `TaskStatus::Ready`，表示任务已准备好，可以调度执行。

3. **任务开始运行**:
   - **操作系统状态**: 将任务状态设置为 `TaskStatus::Running`，表示任务正在 CPU 上执行。
   - **硬件状态**: CPU 开始执行任务的代码。

4. **任务退出**:
   - **操作系统状态**: 将任务状态设置为 `TaskStatus::Exited`，表示任务已完成执行并退出。
   - **硬件状态**: CPU 结束任务的执行。

#### 任务控制块管理

1. **保存任务上下文**:
   - **操作系统状态**: 在任务切换时，内核将当前任务的上下文保存到对应的 `TaskControlBlock` 中。
   - **硬件状态**: CPU 将寄存器值写入内存中的 `TaskContext` 结构体。

2. **恢复任务上下文**:
   - **操作系统状态**: 在任务切换时，内核将下一个任务的上下文从对应的 `TaskControlBlock` 中恢复。
   - **硬件状态**: CPU 从内存中的 `TaskContext` 结构体读取寄存器值，并恢复到寄存器中。

#### 任务管理器

1. **初始化任务管理器**:
   - **操作系统状态**: 在内核启动时，初始化 `TaskManager` 和 `TaskManagerInner`，并设置应用程序数量 `num_app`。

2. **管理任务控制块数组**:
   - **操作系统状态**: 任务管理器维护一个包含所有任务控制块的数组 `tasks`，并跟踪当前正在执行的任务编号 `current_task`。

3. **调度任务**:
   - **操作系统状态**: 调度器选择下一个任务，将其状态从 "准备运行" 切换为 "正在运行"，并通过任务控制块恢复其上下文。
   - **硬件状态**: CPU 切换到下一个任务的上下文，继续执行任务代码。

---

### 初始化 TaskManager 的全局实例 TASK_MANAGER

为了管理所有任务，操作系统需要一个全局的任务管理器实例 `TASK_MANAGER`。我们使用 `lazy_static` 宏来实现这个全局实例的懒加载初始化。

#### 代码位置

```rust
// os/src/task/mod.rs
```

#### 代码详解

```rust
lazy_static! {
    pub static ref TASK_MANAGER: TaskManager = {
        let num_app = get_num_app();
        let mut tasks = [TaskControlBlock {
            task_cx: TaskContext::zero_init(),
            task_status: TaskStatus::UnInit,
        }; MAX_APP_NUM];
        for (i, t) in tasks.iter_mut().enumerate().take(num_app) {
            t.task_cx = TaskContext::goto_restore(init_app_cx(i));
            t.task_status = TaskStatus::Ready;
        }
        TaskManager {
            num_app,
            inner: unsafe {
                UPSafeCell::new(TaskManagerInner {
                    tasks,
                    current_task: 0,
                })
            },
        }
    };
}
```

#### 详细解释

1. **使用 `lazy_static!` 宏**:
   - **宏名**: `lazy_static!`
   - **作用**: 实现全局静态变量的懒初始化，即在第一次使用时进行初始化。

2. **获取应用总数**:
   - **函数名**: `get_num_app`
   - **作用**: 获取链接到内核的应用程序总数。
   - **位置**: 调用 `loader` 子模块提供的接口。

   ```rust
   let num_app = get_num_app();
   ```

3. **初始化任务控制块数组**:
   - **结构体**: `TaskControlBlock`
   - **数组大小**: `MAX_APP_NUM`
   - **初始状态**: 
     - `task_cx`: 使用 `TaskContext::zero_init()` 初始化。
     - `task_status`: 设置为 `TaskStatus::UnInit`。

   ```rust
   let mut tasks = [TaskControlBlock {
       task_cx: TaskContext::zero_init(),
       task_status: TaskStatus::UnInit,
   }; MAX_APP_NUM];
   ```

4. **遍历并初始化每个任务控制块**:
   - **迭代器**: `tasks.iter_mut().enumerate().take(num_app)`
   - **初始化任务上下文**: 使用 `TaskContext::goto_restore(init_app_cx(i))`。
   - **设置任务状态**: 将任务状态设置为 `TaskStatus::Ready`。

   ```rust
   for (i, t) in tasks.iter_mut().enumerate().take(num_app) {
       t.task_cx = TaskContext::goto_restore(init_app_cx(i));
       t.task_status = TaskStatus::Ready;
   }
   ```

   - **函数**: `init_app_cx(i)`
     - **作用**: 初始化每个应用程序的上下文。
   - **函数**: `TaskContext::goto_restore`
     - **作用**: 设置任务上下文的初始值，确保任务能够从正确的位置恢复执行。

5. **创建并返回 `TaskManager` 实例**:
   - **结构体**: `TaskManager`
   - **字段**:
     - `num_app`: 应用程序总数。
     - `inner`: 包含实际任务管理数据的结构体，用 `UPSafeCell` 包装以保证线程安全。
   - **内部结构体**: `TaskManagerInner`
     - **字段**:
       - `tasks`: 任务控制块数组。
       - `current_task`: 当前正在执行的任务编号。

   ```rust
   TaskManager {
       num_app,
       inner: unsafe {
           UPSafeCell::new(TaskManagerInner {
               tasks,
               current_task: 0,
           })
       },
   }
   ```

### 任务管理器的初始化流程

1. **获取应用总数**:
   - **操作系统状态**: 调用 `get_num_app` 获取链接到内核的应用程序总数。
   - **位置**: `loader` 子模块提供的接口。

2. **初始化任务控制块数组**:
   - **操作系统状态**: 创建并初始化一个大小为 `MAX_APP_NUM` 的任务控制块数组。
   - **初始状态**: 每个任务控制块的 `task_cx` 初始化为零，`task_status` 设置为 `TaskStatus::UnInit`。

3. **遍历并初始化每个任务控制块**:
   - **操作系统状态**: 使用迭代器遍历任务控制块数组的前 `num_app` 个元素。
   - **任务上下文初始化**:
     - **函数**: `init_app_cx(i)`
       - **作用**: 初始化应用程序上下文。
     - **函数**: `TaskContext::goto_restore`
       - **作用**: 设置任务上下文的初始值，确保任务能够从正确的位置恢复执行。
   - **设置任务状态**: 将任务状态设置为 `TaskStatus::Ready`。

4. **创建 `TaskManager` 实例**:
   - **操作系统状态**: 创建并初始化 `TaskManager` 实例。
   - **字段**:
     - `num_app`: 应用程序总数。
     - `inner`: 包含实际任务管理数据的结构体，用 `UPSafeCell` 包装以保证线程安全。

5. **返回 `TaskManager` 实例**:
   - **操作系统状态**: 返回初始化后的 `TaskManager` 实例，赋值给全局变量 `TASK_MANAGER`。

### 总结

通过以上步骤，操作系统完成了 `TaskManager` 全局实例 `TASK_MANAGER` 的初始化。`TASK_MANAGER` 负责管理所有任务控制块，并通过其内部结构体 `TaskManagerInner` 来维护任务状态和上下文。这个设计确保了任务的正确初始化和调度，为多任务并发执行提供了基础。

---



#### ASK_MANAGER, TaskControlBlock 和 TaskContext 的对比

1. **TASK_MANAGER**:
   - **位置**: `os/src/task/mod.rs`
   - **类型**: 全局静态实例，使用 `lazy_static!` 宏初始化。
   - **功能**: 管理所有任务控制块（TCB），负责任务调度和切换。
   - **结构**: 
     - `num_app`: 应用程序数量。
     - `inner`: 包含实际任务管理数据的结构体 `TaskManagerInner`。

2. **TaskControlBlock**:
   - **位置**: `os/src/task/task.rs`
   - **类型**: 结构体。
   - **功能**: 维护单个任务的状态和上下文。
   - **结构**:
     - `task_status`: 任务状态（TaskStatus）。
     - `task_cx`: 任务上下文（TaskContext）。

3. **TaskContext**:
   - **位置**: `os/src/task/context.rs`
   - **类型**: 结构体。
   - **功能**: 保存任务的寄存器和栈指针等上下文信息。
   - **结构**:
     - `ra`: 返回地址寄存器（Return Address）。
     - `sp`: 栈指针寄存器（Stack Pointer）。
     - `s`: 被调用者保存寄存器（Saved Registers），包含 `s0~s11`。

---

### 实现 `sys_yield` 和 `sys_exit`

`sys_yield` 和 `sys_exit` 是两个重要的系统调用，分别用于让任务主动放弃 CPU 使用权和退出任务。它们依赖于任务管理器提供的接口来实现任务的调度和状态管理。

#### `sys_yield` 的实现

`sys_yield` 通过 `suspend_current_and_run_next` 接口实现，这个接口的作用是暂停当前的任务并切换到下一个任务。

##### 代码位置和实现

```rust
// os/src/syscall/process.rs

use crate::task::suspend_current_and_run_next;

pub fn sys_yield() -> isize {
    suspend_current_and_run_next();
    0
}
```

1. **函数名**: `sys_yield`
   - **作用**: 应用主动交出 CPU 使用权，让内核调度其他任务执行。
   - **返回值**: 总是返回 0。

2. **调用 `suspend_current_and_run_next` 接口**
   - **位置**: `os/src/task/mod.rs`
   - **作用**: 暂停当前任务并切换到下一个任务。

#### `sys_exit` 的实现

`sys_exit` 通过 `exit_current_and_run_next` 接口实现，这个接口的作用是退出当前的任务并切换到下一个任务。

##### 代码位置和实现

```rust
// os/src/syscall/process.rs

use crate::task::exit_current_and_run_next;

pub fn sys_exit(exit_code: i32) -> ! {
    println!("[kernel] Application exited with code {}", exit_code);
    exit_current_and_run_next();
    panic!("Unreachable in sys_exit!");
}
```

1. **函数名**: `sys_exit`
   - **参数**: `exit_code` (i32)，表示任务退出时的状态码。
   - **作用**: 应用主动退出，并让内核调度其他任务执行。
   - **返回值**: 永远不返回（`!` 表示返回类型为 Never）。

2. **调用 `exit_current_and_run_next` 接口**
   - **位置**: `os/src/task/mod.rs`
   - **作用**: 退出当前任务并切换到下一个任务。

3. **打印退出信息**:
   - **输出**: `"[kernel] Application exited with code {}"`。
   - **作用**: 在任务退出时打印退出状态码。

4. **触发 panic**:
   - **作用**: 理论上不应到达此处，触发 panic 以捕获错误。

#### `suspend_current_and_run_next` 和 `exit_current_and_run_next` 的实现

这两个函数都是先修改当前任务的运行状态，然后尝试切换到下一个任务。

##### 代码位置和实现

```rust
// os/src/task/mod.rs

pub fn suspend_current_and_run_next() {
    TASK_MANAGER.mark_current_suspended();
    TASK_MANAGER.run_next_task();
}

pub fn exit_current_and_run_next() {
    TASK_MANAGER.mark_current_exited();
    TASK_MANAGER.run_next_task();
}
```

1. **函数名**: `suspend_current_and_run_next`
   - **作用**: 暂停当前任务并切换到下一个任务。
   - **步骤**:
     1. 调用 `TASK_MANAGER.mark_current_suspended()` 将当前任务状态标记为暂停。
     2. 调用 `TASK_MANAGER.run_next_task()` 切换到下一个任务。

2. **函数名**: `exit_current_and_run_next`
   - **作用**: 退出当前任务并切换到下一个任务。
   - **步骤**:
     1. 调用 `TASK_MANAGER.mark_current_exited()` 将当前任务状态标记为退出。
     2. 调用 `TASK_MANAGER.run_next_task()` 切换到下一个任务。

---

### 修改任务运行状态和任务切换

#### 修改运行状态：`mark_current_suspended`

修改运行状态主要涉及到任务控制块数组中的当前任务状态。在任务管理器 `TaskManager` 中实现一个方法来修改当前任务的状态，例如 `mark_current_suspended`。

##### 代码位置和实现

```rust
// os/src/task/mod.rs

impl TaskManager {
    fn mark_current_suspended(&self) {
        let mut inner = self.inner.exclusive_access();
        let current = inner.current_task;
        inner.tasks[current].task_status = TaskStatus::Ready;
    }
}
```

##### 详细解释

1. **获取内部任务管理器的可变引用**:
   - **方法**: `self.inner.exclusive_access()`
   - **作用**: 获取 `TaskManagerInner` 的可变引用，以便修改内部状态。

2. **获取当前任务的索引**:
   - **变量**: `current`
   - **作用**: 从 `TaskManagerInner` 中获取当前任务的索引。

3. **修改当前任务的状态**:
   - **变量**: `inner.tasks[current].task_status`
   - **作用**: 将当前任务的状态修改为 `TaskStatus::Ready`，表示任务暂停，准备再次运行。

#### 切换到下一个任务：`run_next_task`

`run_next_task` 方法负责切换到下一个准备运行的任务。

##### 代码位置和实现

```rust
// os/src/task/mod.rs

impl TaskManager {
    fn run_next_task(&self) {
        if let Some(next) = self.find_next_task() {
            let mut inner = self.inner.exclusive_access();
            let current = inner.current_task;
            inner.tasks[next].task_status = TaskStatus::Running;
            inner.current_task = next;
            let current_task_cx_ptr = &mut inner.tasks[current].task_cx as *mut TaskContext;
            let next_task_cx_ptr = &inner.tasks[next].task_cx as *const TaskContext;
            drop(inner);
            // before this, we should drop local variables that must be dropped manually
            unsafe {
                __switch(current_task_cx_ptr, next_task_cx_ptr);
            }
            // go back to user mode
        } else {
            panic!("All applications completed!");
        }
    }
}
```

##### 详细解释

1. **寻找下一个准备运行的任务**:
   - **方法**: `self.find_next_task()`
   - **作用**: 寻找一个状态为 `TaskStatus::Ready` 的任务，并返回其 ID。

2. **获取内部任务管理器的可变引用**:
   - **方法**: `self.inner.exclusive_access()`
   - **作用**: 获取 `TaskManagerInner` 的可变引用，以便修改内部状态。

3. **设置下一个任务的状态**:
   - **变量**: `inner.tasks[next].task_status`
   - **作用**: 将下一个任务的状态设置为 `TaskStatus::Running`，表示任务正在运行。

4. **更新当前任务索引**:
   - **变量**: `inner.current_task`
   - **作用**: 更新当前任务的索引为下一个任务的索引。

5. **获取当前和下一个任务的上下文指针**:
   - **变量**: `current_task_cx_ptr` 和 `next_task_cx_ptr`
   - **作用**: 分别获取当前任务和下一个任务的上下文指针，以便在任务切换时使用。

6. **手动 drop 内部任务管理器的可变引用**:
   - **方法**: `drop(inner)`
   - **作用**: 手动释放 `inner` 的可变引用，以确保 `TASK_MANAGER` 的 `inner` 字段回到未被借用的状态。

7. **任务切换**:
   - **函数**: `__switch`
   - **作用**: 使用汇编代码实现的任务切换函数，切换当前任务的上下文到下一个任务的上下文。

8. **处理所有任务完成的情况**:
   - **作用**: 如果没有找到准备运行的任务，`find_next_task` 返回 `None`，内核会触发 `panic`，表示所有任务都已经完成。

#### 寻找下一个任务：`find_next_task`

`find_next_task` 方法用于寻找下一个准备运行的任务，并返回其 ID。

##### 代码位置和实现

```rust
// os/src/task/mod.rs

impl TaskManager {
    fn find_next_task(&self) -> Option<usize> {
        let inner = self.inner.exclusive_access();
        let current = inner.current_task;
        (current + 1..current + self.num_app + 1)
            .map(|id| id % self.num_app)
            .find(|id| inner.tasks[*id].task_status == TaskStatus::Ready)
    }
}
```

##### 详细解释

1. **获取内部任务管理器的可变引用**:
   - **方法**: `self.inner.exclusive_access()`
   - **作用**: 获取 `TaskManagerInner` 的可变引用，以便读取内部状态。

2. **获取当前任务的索引**:
   - **变量**: `current`
   - **作用**: 从 `TaskManagerInner` 中获取当前任务的索引。

3. **遍历任务数组寻找准备运行的任务**:
   - **方法**: 
     - `current + 1..current + self.num_app + 1`: 从当前任务的下一个任务开始遍历，循环遍历任务数组。
     - `.map(|id| id % self.num_app)`: 确保任务索引在任务数组范围内循环。
     - `.find(|id| inner.tasks[*id].task_status == TaskStatus::Ready)`: 找到第一个状态为 `TaskStatus::Ready` 的任务，并返回其 ID。

### 总结

通过 `mark_current_suspended` 方法，我们可以将当前任务的状态修改为 `Ready`，表示任务暂停，准备再次运行。`run_next_task` 方法负责切换到下一个准备运行的任务，并调用 `__switch` 进行上下文切换。`find_next_task` 方法用于寻找下一个准备运行的任务，确保任务切换的正确进行。

#### 修改运行状态和任务切换的关键步骤

1. **修改当前任务的状态**:
   - 获取 `TaskManagerInner` 的可变引用。
   - 修改当前任务的状态为 `Ready` 或其他状态。

2. **任务切换**:
   - 寻找下一个准备运行的任务。
   - 更新任务状态和当前任务索引。
   - 获取上下文指针。
   - 调用汇编实现的 `__switch` 进行上下文切换。

通过这些步骤，操作系统实现了多任务的调度和切换，确保任务能够有效地并发执行。

---

### 第一次进入用户态

#### 背景

在第二章中，CPU 第一次从内核态进入用户态的方法是通过在内核栈上压入构造好的 Trap 上下文，并通过调用 `__restore` 函数恢复上下文。本章在此基础上进行扩展，详细解释任务上下文的初始化和任务切换的实现。

#### 初始化任务控制块

在任务管理器中初始化任务控制块时，我们使用 `init_app_cx` 函数向内核栈压入一个 Trap 上下文，并返回压入 Trap 上下文后栈指针（sp）的值。`goto_restore` 函数保存传入的 sp，并将返回地址（ra）设置为 `__restore` 的入口地址。

##### 代码位置和实现

```rust
// os/src/task/mod.rs

for (i, t) in tasks.iter_mut().enumerate().take(num_app) {
    t.task_cx = TaskContext::goto_restore(init_app_cx(i));
    t.task_status = TaskStatus::Ready;
}
```

1. **初始化任务上下文**:
   - **函数名**: `init_app_cx`
   - **作用**: 向内核栈压入一个 Trap 上下文，并返回压入 Trap 上下文后栈指针的值。

2. **设置任务上下文**:
   - **函数名**: `TaskContext::goto_restore`
   - **作用**: 保存传入的栈指针，并将返回地址设置为 `__restore` 的入口地址。

#### TaskContext 实现

`TaskContext` 结构体用于保存任务上下文，包括返回地址（ra）、栈指针（sp）和保存寄存器（s0~s11）。

##### 代码位置和实现

```rust
// os/src/task/context.rs

impl TaskContext {
    pub fn goto_restore(kstack_ptr: usize) -> Self {
        extern "C" { fn __restore(); }
        Self {
            ra: __restore as usize,
            sp: kstack_ptr,
            s: [0; 12],
        }
    }
}
```

1. **返回地址（ra）**:
   - **设置为 `__restore` 的入口地址**:
     - **函数名**: `__restore`
     - **作用**: 恢复 Trap 上下文并进入用户态。

2. **栈指针（sp）**:
   - **设置为 `init_app_cx` 返回的值**:
     - **作用**: 指向内核栈上的 Trap 上下文。

3. **保存寄存器（s0~s11）**:
   - **初始化为 0**:
     - **作用**: 在任务初始化时，保存寄存器的值为 0。

#### 运行第一个任务

在 `rust_main` 函数中，我们调用 `task::run_first_task` 来执行第一个应用。该函数切换到第一个任务并进入用户态。

##### 代码位置和实现

```rust
// os/src/task/mod.rs

fn run_first_task(&self) -> ! {
    let mut inner = self.inner.exclusive_access();
    let task0 = &mut inner.tasks[0];
    task0.task_status = TaskStatus::Running;
    let next_task_cx_ptr = &task0.task_cx as *const TaskContext;
    drop(inner);
    let mut _unused = TaskContext::zero_init();
    // before this, we should drop local variables that must be dropped manually
    unsafe {
        __switch(&mut _unused as *mut TaskContext, next_task_cx_ptr);
    }
    panic!("unreachable in run_first_task!");
}
```

1. **获取第一个任务的上下文指针**:
   - **变量**: `next_task_cx_ptr`
   - **作用**: 获取第一个任务的上下文指针。

2. **设置任务状态**:
   - **变量**: `task0.task_status`
   - **作用**: 将第一个任务的状态设置为 `TaskStatus::Running`。

3. **获取任务上下文指针**
   - **获取任务上下文引用**:
     - **代码**: `&task0.task_cx`
     - **作用**: 获取第一个任务的任务上下文 `task_cx` 的引用。
   - **转换为指针**:
     - **代码**: `as *const TaskContext`
     - **作用**: 将任务上下文的引用转换为原生指针（const pointer），指向 `TaskContext` 结构体。
     - **原因**: `__switch` 函数接受指向 `TaskContext` 的原生指针。
   - **变量**: `next_task_cx_ptr`
     - **类型**: `*const TaskContext`
     - **作用**: 保存指向第一个任务的任务上下文的指针。

4. **手动释放 `TaskManagerInner` 的可变引用**:
   - **方法**: `drop(inner)`
   - **作用**: 手动释放 `inner` 的可变引用，以确保 `TASK_MANAGER` 的 `inner` 字段回到未被借用的状态。

5. **任务切换**:
   - **函数名**: `__switch`
   - **参数**: `_unused` 和 `next_task_cx_ptr`
   - **作用**: 切换到第一个任务的上下文，进入用户态。
     - **声明未使用的任务上下文**:
       - **代码**: `let mut _unused = TaskContext::zero_init();`
       - **作用**: 创建一个未使用的任务上下文 `_unused`，用作 `__switch` 的第一个参数。
       - **原因**: 在第一次任务切换时，没有真正的上一个任务上下文，所以使用一个占位符 `_unused`。
     - **转换为指针**:
       - **代码**: `&mut _unused as *mut TaskContext`
       - **作用**: 将未使用的任务上下文的引用转换为原生指针（mut pointer），指向 `TaskContext` 结构体。
       - **原因**: `__switch` 函数接受指向 `TaskContext` 的原生指针。
     - **调用 `__switch`**:
       - **代码**: `__switch(&mut _unused as *mut TaskContext, next_task_cx_ptr);`
       - **作用**: 调用 `__switch` 函数进行任务切换。
       - 参数:
         - `&mut _unused as *mut TaskContext`: 指向未使用的任务上下文的指针，作为当前任务的上下文指针。
         - `next_task_cx_ptr`: 指向第一个任务的任务上下文的指针，作为下一个任务的上下文指针。
     - **使用 `unsafe` 块**:
       - **原因**: 调用 `__switch` 函数涉及到直接操作原生指针，这在 Rust 中是 `unsafe` 的，需要用 `unsafe` 块包裹。

#### `__restore` 函数的实现

在 `__switch` 中恢复 sp 后，sp 将指向 `init_app_cx` 构造的 Trap 上下文，后面就回到第二章的情况了。此外，`__restore` 的实现需要做出变化：它不再需要在开头 `mv sp, a0`，因为在 `__switch` 之后，sp 就已经正确指向了我们需要的 Trap 上下文地址。

### 总结

通过初始化任务控制块和任务上下文，我们能够将 CPU 从内核态切换到用户态。具体步骤包括：

1. **初始化任务控制块**:
   - 使用 `init_app_cx` 向内核栈压入 Trap 上下文，并返回栈指针。
   - 使用 `TaskContext::goto_restore` 设置任务上下文，包括返回地址和栈指针。

2. **运行第一个任务**:
   - 调用 `task::run_first_task` 切换到第一个任务的上下文，并进入用户态。

3. **任务切换**:
   - 在 `__switch` 中切换任务上下文，恢复 sp 后进入用户态。

通过这些步骤，操作系统能够正确地初始化任务并在首次运行时切换到用户态，确保任务能够正确执行。



---

### 分时多任务系统

分时多任务系统通过时间片轮转算法 (Round-Robin, RR) 来实现任务调度。在这种系统中，每个任务只能连续执行一个时间片（可能在毫秒量级），然后内核强制性切换到下一个任务。

### 关键概念

1. **时间片 (Time Slice)**:
   - 是任务连续执行的时间度量单位。
   - 一般在毫秒量级。

2. **时间片轮转算法 (Round-Robin, RR)**:
   - 每个任务按顺序轮流执行一个时间片。

3. **时钟中断**:
   - 计时器到达设定时间时触发，用于实现时间片轮转调度。

### 时钟中断与计时器

RISC-V 架构要求处理器维护时钟计数器 `mtime` 和比较寄存器 `mtimecmp`。当 `mtime` 的值超过 `mtimecmp` 时，会触发时钟中断。

#### 获取当前时间

`get_time` 函数用于获取当前的 `mtime` 计数器值。

##### 代码位置和实现

```rust
// os/src/timer.rs

use riscv::register::time;

pub fn get_time() -> usize {
    time::read()
}
```

##### 详细解释

1. **导入 `riscv::register::time` 模块**:
   - **模块**: `riscv::register::time`
   - **作用**: 提供读取 RISC-V 时钟计数器 `mtime` 的功能。

2. **定义 `get_time` 函数**:
   - **返回类型**: `usize`
   - **作用**: 返回当前的 `mtime` 计数器值。

3. **读取当前时间**:
   - **方法**: `time::read()`
   - **作用**: 读取 `mtime` 计数器的当前值并返回。

### 设置时钟中断

`set_timer` 函数用于设置 `mtimecmp` 的值，从而在指定时间后触发时钟中断。

##### 代码位置和实现

```rust
// os/src/sbi.rs

const SBI_SET_TIMER: usize = 0;

pub fn set_timer(timer: usize) {
    sbi_call(SBI_SET_TIMER, timer, 0, 0);
}
```

##### 详细解释

1. **定义常量 `SBI_SET_TIMER`**:
   - **类型**: `usize`
   - **值**: 0
   - **作用**: SBI 调用 `set_timer` 的函数编号。

2. **定义 `set_timer` 函数**:
   - **参数**: `timer` (类型 `usize`)，表示设置的 `mtimecmp` 的值。
   - **作用**: 调用 SBI 接口设置 `mtimecmp` 的值。

3. **调用 `sbi_call` 函数**:
   - **参数**:
     - `SBI_SET_TIMER`: SBI 调用编号。
     - `timer`: 设置的 `mtimecmp` 的值。
     - 其他两个参数为 0。
   - **作用**: 通过 SBI 接口调用设置 `mtimecmp` 的值。

### 定时触发

`set_next_trigger` 函数用于计算下一个时钟中断的触发时间，并设置 `mtimecmp` 的值。

##### 代码位置和实现

```rust
// os/src/timer.rs

use crate::config::CLOCK_FREQ;

const TICKS_PER_SEC: usize = 100;

pub fn set_next_trigger() {
    set_timer(get_time() + CLOCK_FREQ / TICKS_PER_SEC);
}
```

##### 详细解释

1. **导入 `CLOCK_FREQ` 常量**:
   - **模块**: `config`
   - **作用**: 表示平台的时钟频率，单位为赫兹。

2. **定义常量 `TICKS_PER_SEC`**:
   - **类型**: `usize`
   - **值**: 100
   - **作用**: 表示每秒的时钟中断次数（即每 10ms 一次）。

3. **定义 `set_next_trigger` 函数**:
   - **作用**: 设置下一个时钟中断的触发时间。

4. **计算下一个时钟中断时间**:
   - **方法**: `get_time() + CLOCK_FREQ / TICKS_PER_SEC`
   - **作用**: 获取当前时间 `get_time()`，加上 `CLOCK_FREQ / TICKS_PER_SEC` 的增量，计算出 10ms 后的时间。

5. **设置时钟中断**:
   - **函数**: `set_timer`
   - **参数**: 计算出的下一个时钟中断时间。
   - **作用**: 设置 `mtimecmp` 的值，使得 10ms 后触发时钟中断。

### 时钟中断处理

当 `mtime` 超过 `mtimecmp` 的值时，会触发时钟中断。时钟中断处理程序需要执行以下步骤：

1. **保存当前任务的上下文**。
2. **调度下一个任务**。
3. **恢复下一个任务的上下文**。

##### 时钟中断处理流程

1. **触发时钟中断**:
   - **硬件**: 当 `mtime` 超过 `mtimecmp` 的值时，硬件触发时钟中断。
   - **作用**: 通知内核需要进行任务调度。

2. **保存当前任务的上下文**:
   - **内核**: 保存当前任务的寄存器、栈指针等上下文信息。
   - **作用**: 保持当前任务的状态，以便以后恢复。

3. **调度下一个任务**:
   - **内核**: 调用任务调度算法（如 RR 算法）选择下一个任务。
   - **作用**: 确定下一个任务，并准备切换上下文。

4. **恢复下一个任务的上下文**:
   - **内核**: 恢复下一个任务的寄存器、栈指针等上下文信息。
   - **作用**: 切换到下一个任务，开始执行。

5. **重新设置时钟中断**:
   - **内核**: 调用 `set_next_trigger` 函数，设置下一个时钟中断时间。
   - **作用**: 确保下一个时钟中断能够正确触发，实现持续的任务调度。

### 总结

通过上述步骤和代码实现，我们构建了一个分时多任务系统。该系统利用时钟中断和时间片轮转算法，实现任务的定时调度和上下文切换，确保每个任务能够公平地获得 CPU 使用权，并在特定时间片后强制切换任务，提高系统的响应速度和资源利用率。

---

### 计时需求和新系统调用

为了满足后续的计时需求，我们需要设计一个能够以微秒为单位返回当前计时器值的函数，并新增一个系统调用，使应用能够获取当前时间。

#### 以微秒为单位返回当前计时器值

在 `timer` 子模块中，我们设计了 `get_time_us` 函数，用于以微秒为单位返回当前计时器的值。

##### 代码位置和实现

```rust
// os/src/timer.rs

use riscv::register::time;
use crate::config::CLOCK_FREQ;

const MICRO_PER_SEC: usize = 1_000_000;

pub fn get_time_us() -> usize {
    time::read() / (CLOCK_FREQ / MICRO_PER_SEC)
}
```

##### 详细解释

1. **常量定义**:
   - **常量**: `MICRO_PER_SEC`
   - **类型**: `usize`
   - **值**: 1_000_000（表示一秒中的微秒数）

2. **函数定义**: `get_time_us`
   - **返回类型**: `usize`
   - **作用**: 以微秒为单位返回当前计时器的值

3. **读取当前时间**:
   - **函数**: `time::read()`
   - **作用**: 读取 RISC-V 时钟计数器 `mtime` 的当前值

4. **计算当前时间（微秒）**:
   - **公式**: `time::read() / (CLOCK_FREQ / MICRO_PER_SEC)`
   - **作用**: 将 `mtime` 值转换为微秒数

#### 新增系统调用：获取当前时间

为了使应用能够获取当前时间，我们设计了一个新的系统调用 `sys_get_time`，并定义了 `TimeVal` 结构体来存储时间值。

##### 系统调用定义

```rust
/// 功能：获取当前的时间，保存在 TimeVal 结构体 ts 中，_tz 在我们的实现中忽略
/// 返回值：返回是否执行成功，成功则返回 0
/// syscall ID：169
fn sys_get_time(ts: *mut TimeVal, _tz: usize) -> isize;
```

##### 结构体 `TimeVal` 的定义

```rust
// os/src/syscall/process.rs

#[repr(C)]
pub struct TimeVal {
    pub sec: usize,
    pub usec: usize,
}
```

##### 系统调用实现

```rust
// os/src/syscall/process.rs

use crate::timer::get_time_us;

pub fn sys_get_time(ts: *mut TimeVal, _tz: usize) -> isize {
    if ts.is_null() {
        return -1;
    }
    
    let us = get_time_us();
    let time_val = TimeVal {
        sec: us / 1_000_000,
        usec: us % 1_000_000,
    };
    
    unsafe {
        *ts = time_val;
    }
    
    0
}
```

##### 详细解释

1. **引入 `get_time_us` 函数**:
   - **模块**: `timer`
   - **作用**: 获取当前时间（微秒）

2. **函数定义**: `sys_get_time`
   - **参数**:
     - `ts`: 指向 `TimeVal` 结构体的指针，用于存储当前时间
     - `_tz`: 时区参数，在我们的实现中被忽略
   - **返回值**: `isize`，表示执行结果，成功返回 0

3. **检查指针是否为空**:
   - **判断**: `ts.is_null()`
   - **作用**: 检查传入的指针是否为 `null`，如果是则返回错误码 `-1`

4. **获取当前时间（微秒）**:
   - **函数**: `get_time_us()`
   - **作用**: 获取当前时间，单位为微秒

5. **计算时间值**:
   - **变量**: `us`
   - **公式**:
     - `sec: us / 1_000_000`：将微秒转换为秒
     - `usec: us % 1_000_000`：取余数，得到剩余的微秒数

6. **创建 `TimeVal` 结构体**:
   - **结构体**: `TimeVal`
   - **字段**:
     - `sec`: 秒数
     - `usec`: 微秒数

7. **写入时间值到指针**:
   - **代码**: `*ts = time_val;`
   - **作用**: 使用 `unsafe` 块将计算的时间值写入传入的指针所指向的内存位置

8. **返回成功码**:
   - **返回值**: `0`
   - **作用**: 表示系统调用成功执行

### 总结

通过上述步骤和代码实现，我们完成了以微秒为单位返回当前计时器值的函数 `get_time_us`，以及一个新的系统调用 `sys_get_time`，使应用能够获取当前时间。

#### 关键步骤

1. **定义 `get_time_us` 函数**:
   - 获取当前 `mtime` 计数器值并转换为微秒数。

2. **定义 `TimeVal` 结构体**:
   - 用于存储秒和微秒两个时间字段。

3. **实现 `sys_get_time` 系统调用**:
   - 检查指针合法性。
   - 获取当前时间并计算秒和微秒。
   - 将时间值写入传入的 `TimeVal` 结构体指针。

通过这些步骤，我们能够实现一个分时多任务系统中的计时功能，并通过系统调用使用户应用能够获取当前时间。

---

### 解释涉及的公式及其原因

#### 1. 设置下一个时钟中断触发时间

```rust
pub fn set_next_trigger() {
    set_timer(get_time() + CLOCK_FREQ / TICKS_PER_SEC);
}
```

##### 详细解释

- **`get_time()`**:
  - **作用**: 获取当前的 `mtime` 计数器值。`mtime` 是 RISC-V 架构中的一个硬件计数器，用于计时。

- **`CLOCK_FREQ / TICKS_PER_SEC`**:
  - **公式**: `CLOCK_FREQ` 表示时钟频率（每秒的计数值），`TICKS_PER_SEC` 表示每秒的时钟中断次数。
  - **作用**: 计算每个时间片的时钟计数器增量。`CLOCK_FREQ` 是一秒内的时钟计数器增量，`TICKS_PER_SEC` 是每秒的时间片数，`CLOCK_FREQ / TICKS_PER_SEC` 就是每个时间片对应的时钟计数器增量。

- **`get_time() + CLOCK_FREQ / TICKS_PER_SEC`**:
  - **作用**: 计算下一个时钟中断触发的时间点。当前时间加上一个时间片的增量就是下一个时钟中断触发的时间点。

- **`set_timer()`**:
  - **作用**: 将计算出的下一个触发时间点设置到 `mtimecmp` 中，以便触发时钟中断。

##### 原因
- 这个公式确保每个时间片后触发一次时钟中断，以实现时间片轮转调度。`CLOCK_FREQ / TICKS_PER_SEC` 确保时间片的长度固定，从而实现公平的任务调度。

#### 2. 获取当前时间（微秒）

```rust
pub fn get_time_us() -> usize {
    time::read() / (CLOCK_FREQ / MICRO_PER_SEC)
}
```

##### 详细解释

- **`time::read()`**:
  - **作用**: 读取当前的 `mtime` 计数器值。

- **`CLOCK_FREQ / MICRO_PER_SEC`**:
  - **公式**: `CLOCK_FREQ` 表示时钟频率，`MICRO_PER_SEC` 表示一秒内的微秒数（1,000,000）。
  - **作用**: 计算每微秒对应的时钟计数器增量。`CLOCK_FREQ` 是一秒内的时钟计数器增量，`MICRO_PER_SEC` 是一秒内的微秒数，`CLOCK_FREQ / MICRO_PER_SEC` 就是每微秒对应的时钟计数器增量。

- **`time::read() / (CLOCK_FREQ / MICRO_PER_SEC)`**:
  - **作用**: 将当前的 `mtime` 计数器值转换为微秒数。通过除以每微秒的计数器增量，可以得到当前的时间（单位为微秒）。

##### 原因
- 这个公式确保 `mtime` 计数器值可以被转换为更精细的时间单位（微秒），从而实现高精度的计时功能。

#### 3. 定义 `TimeVal` 结构体及其初始化

```rust
pub struct TimeVal {
    pub sec: usize,
    pub usec: usize,
}

let time_val = TimeVal {
    sec: us / 1_000_000,
    usec: us % 1_000_000,
};
```

##### 详细解释

- **结构体 `TimeVal`**:
  - **字段**:
    - `sec`: 秒数
    - `usec`: 微秒数

- **初始化 `TimeVal` 结构体**:
  - **变量**: `us`
  - **类型**: `usize`
  - **作用**: 表示当前时间（单位为微秒）

- **`sec: us / 1_000_000`**:
  - **公式**: `us / 1_000_000`
  - **作用**: 将微秒数转换为秒数。通过将微秒数除以 1,000,000（每秒的微秒数），得到当前的秒数。

- **`usec: us % 1_000_000`**:
  - **公式**: `us % 1_000_000`
  - **作用**: 获取当前秒数之外的剩余微秒数。通过取余操作，得到当前秒数之外的微秒数。

##### 原因
- 这个公式将微秒数分解为秒数和微秒数，使得时间表示更加精确和易于理解。`TimeVal` 结构体将时间分为两个部分，便于系统调用返回更高精度的时间值。

### 总结

这些公式和计算方法确保了系统能够以高精度计时，并实现时间片轮转调度。通过这些机制，操作系统可以准确地管理任务执行时间和调度，确保系统的公平性和响应速度。

1. **时间片轮转调度**:
   - `set_next_trigger` 计算下一个时钟中断触发时间，以实现时间片轮转调度。
2. **高精度计时**:
   - `get_time_us` 将 `mtime` 计数器值转换为微秒，以实现高精度计时。
3. **时间表示**:
   - `TimeVal` 结构体将时间分为秒和微秒，提供更高精度的时间表示。

这些机制和计算方法共同构成了一个高效的分时多任务操作系统。

---

### RISC-V 架构中的嵌套中断问题

在 RISC-V 架构中，嵌套中断（Nested Interrupt）指在处理一个中断的过程中，又被同特权级或高特权级的中断打断。默认情况下，RISC-V 硬件会屏蔽同特权级的中断，以避免嵌套中断。

#### 1. Trap 和中断处理

在 RISC-V 中，当 Trap（包括中断、异常和系统调用）发生时，系统进入某个特权级（如 S 特权级），并进行相应的处理。在这个过程中，默认情况下，当前特权级的中断会被屏蔽。

#### 2. S 特权级的中断屏蔽机制

##### sstatus 寄存器

`sstatus` 是一个控制和状态寄存器，包含多个字段，其中 `sie` 和 `spie` 与中断屏蔽相关：

- **sstatus.sie**: S 特权级中断使能位。若为 1，表示使能 S 特权级的中断；若为 0，表示屏蔽 S 特权级的中断。
- **sstatus.spie**: 保存先前的 S 特权级中断使能状态。

##### Trap 处理流程

1. **Trap 发生**:
   - **动作**: `sstatus.sie` 被保存在 `sstatus.spie` 中，同时 `sstatus.sie` 被置零。
   - **结果**: 屏蔽所有 S 特权级的中断，确保当前 Trap 处理过程中不会被其他 S 特权级中断打断。

2. **Trap 处理完毕**:
   - **动作**: `sret` 指令执行时，将 `sstatus.sie` 恢复为 `sstatus.spie` 的值。
   - **结果**: 恢复 S 特权级中断使能状态。

#### 3. 嵌套中断与嵌套 Trap

##### 嵌套中断

嵌套中断指在处理一个中断的过程中，又被同特权级或高特权级的中断打断。

- **默认情况**:
  - 硬件会避免同特权级中断的嵌套，因为 `sstatus.sie` 在 Trap 处理过程中被置零。
- **手动设置**:
  - 可以通过手动设置 `sstatus` 寄存器来允许同特权级中断的嵌套。
- **高特权级中断**:
  - 高特权级的中断仍可以打断低特权级的中断处理，这种情况是无法避免的。

##### 嵌套 Trap

嵌套 Trap 指在处理一个 Trap 的过程中，再次发生 Trap。嵌套中断是嵌套 Trap 的一种情况。

- **例子**:
  - 在处理系统调用时发生页面缺失异常。
  - 在处理中断时发生另一种类型的 Trap（如非法指令异常）。

### 处理嵌套中断的机制

RISC-V 硬件和操作系统提供机制来处理嵌套中断，以确保系统的稳定性和响应性。

#### 1. sstatus CSR 的设置

操作系统可以通过手动设置 `sstatus` 寄存器来控制中断的屏蔽和使能。

##### 代码示例

```rust
use riscv::register::sstatus;

fn enable_nested_interrupts() {
    unsafe {
        sstatus::set_sie();
    }
}

fn disable_nested_interrupts() {
    unsafe {
        sstatus::clear_sie();
    }
}
```

#### 2. Trap 和中断处理流程

操作系统在处理 Trap 和中断时，可以根据需要选择是否允许嵌套中断。

##### 示例流程

1. **进入 Trap 处理程序**:
   - 屏蔽同特权级中断：`sstatus.sie` 被置零。
   - 根据需要决定是否启用嵌套中断：调用 `enable_nested_interrupts`。

2. **处理 Trap**:
   - 处理异常、系统调用或中断。
   - 可能发生新的 Trap（如嵌套中断或异常）。

3. **退出 Trap 处理程序**:
   - 恢复中断使能状态：`sret` 指令将 `sstatus.sie` 恢复为 `sstatus.spie` 的值。

#### 3. 操作系统中的嵌套中断处理

操作系统可以在设计中考虑嵌套中断的处理，以提高系统的响应性和鲁棒性。

##### 设计策略

1. **确定哪些中断可以嵌套**:
   - 某些高优先级中断可以嵌套，例如时钟中断或紧急系统事件。
   - 低优先级中断可能不允许嵌套。

2. **实现嵌套中断处理**:
   - 在中断处理程序中，根据中断优先级决定是否允许嵌套。
   - 保存和恢复上下文时，确保不会影响正在处理的中断或 Trap。

### 总结

在 RISC-V 架构中，默认情况下同特权级的中断在 Trap 处理过程中会被屏蔽，以避免嵌套中断。通过手动设置 `sstatus` CSR，可以允许嵌套中断的发生。而高特权级的中断可以打断低特权级的中断处理，这是无法避免的。

嵌套中断与嵌套 Trap 是操作系统设计中需要处理的复杂问题。通过合理设计中断处理机制和使用适当的硬件控制寄存器，操作系统可以有效管理和处理嵌套中断，确保系统的稳定性和响应速度。

---

### 抢占式调度

抢占式调度是一种调度算法，通过时钟中断和计时器强制任务切换。利用 RISC-V 架构中的时钟中断机制，我们可以实现时间片轮转调度算法（Round-Robin, RR），确保每个任务都能公平地获得 CPU 资源。

#### 实现抢占式调度的步骤

1. **时钟中断处理**:
   - 当触发 S 特权级时钟中断时，重新设置计时器，暂停当前任务，并切换到下一个任务。

2. **启用时钟中断**:
   - 在执行第一个应用前，启用 S 特权级时钟中断，并设置第一个 10ms 的计时器。

### 时钟中断处理

在 `trap_handler` 函数中新增一个分支，用于处理 S 特权级时钟中断。

##### 代码位置和实现

```rust
// os/src/trap/mod.rs

match scause.cause() {
    Trap::Interrupt(Interrupt::SupervisorTimer) => {
        set_next_trigger();
        suspend_current_and_run_next();
    }
}
```

##### 详细解释

1. **识别时钟中断**:
   - **方法**: `scause.cause()`
   - **枚举**: `Trap::Interrupt(Interrupt::SupervisorTimer)`
   - **作用**: 识别 S 特权级时钟中断。

2. **重新设置计时器**:
   - **函数**: `set_next_trigger()`
   - **作用**: 设置下一个时钟中断触发时间，以便在 10ms 后再次触发中断。

3. **暂停当前任务并切换到下一个任务**:
   - **函数**: `suspend_current_and_run_next()`
   - **作用**: 暂停当前任务的执行，并切换到下一个任务。

### 启用时钟中断

在执行第一个应用前，启用 S 特权级时钟中断，并设置第一个 10ms 的计时器。

##### 代码位置和实现

```rust
// os/src/main.rs

#[no_mangle]
pub fn rust_main() -> ! {
    // ...
    trap::enable_timer_interrupt();
    timer::set_next_trigger();
    // ...
}
```

##### 详细解释

1. **启用 S 特权级时钟中断**:
   - **函数**: `trap::enable_timer_interrupt()`
   - **作用**: 设置 S 特权级时钟中断使能位，使得时钟中断不会被屏蔽。

2. **设置第一个 10ms 的计时器**:
   - **函数**: `timer::set_next_trigger()`
   - **作用**: 设置下一个时钟中断触发时间，为 10ms 后。

##### 启用时钟中断的具体实现

在 `trap` 模块中，实现 `enable_timer_interrupt` 函数。

```rust
// os/src/trap/mod.rs

use riscv::register::sie;

pub fn enable_timer_interrupt() {
    unsafe { sie::set_stimer(); }
}
```

##### 详细解释

1. **导入 `sie` 模块**:
   - **模块**: `riscv::register::sie`
   - **作用**: 提供设置和清除 S 特权级中断使能位的功能。

2. **启用 S 特权级时钟中断**:
   - **函数**: `sie::set_stimer()`
   - **作用**: 设置 S 特权级时钟中断使能位，使得时钟中断不会被屏蔽。

### 核心函数和机制

#### `suspend_current_and_run_next`

暂停当前任务并切换到下一个任务。

##### 代码位置和实现

```rust
// os/src/task/mod.rs

pub fn suspend_current_and_run_next() {
    TASK_MANAGER.mark_current_suspended();
    TASK_MANAGER.run_next_task();
}
```

##### 详细解释

1. **标记当前任务为暂停**:
   - **函数**: `TASK_MANAGER.mark_current_suspended()`
   - **作用**: 将当前任务的状态设置为 `TaskStatus::Ready`。

2. **运行下一个任务**:
   - **函数**: `TASK_MANAGER.run_next_task()`
   - **作用**: 调度并运行下一个准备好的任务。

#### `set_next_trigger`

设置下一个时钟中断触发时间。

##### 代码位置和实现

```rust
// os/src/timer.rs

use crate::config::CLOCK_FREQ;

const TICKS_PER_SEC: usize = 100;

pub fn set_next_trigger() {
    set_timer(get_time() + CLOCK_FREQ / TICKS_PER_SEC);
}
```

##### 详细解释

1. **获取当前时间**:
   - **函数**: `get_time()`
   - **作用**: 获取当前的 `mtime` 计数器值。

2. **计算下一个时钟中断触发时间**:
   - **公式**: `CLOCK_FREQ / TICKS_PER_SEC`
   - **作用**: 计算 10ms 的计时器增量。

3. **设置下一个时钟中断触发时间**:
   - **函数**: `set_timer(get_time() + CLOCK_FREQ / TICKS_PER_SEC)`
   - **作用**: 设置 `mtimecmp` 的值，使得 10ms 后触发时钟中断。

### 总结

通过利用 RISC-V 架构中的时钟中断机制，我们实现了抢占式调度。这一调度算法通过时钟中断定期强制任务切换，确保每个任务都能公平地获得 CPU 资源，实现了时间片轮转调度算法。

#### 关键步骤

1. **时钟中断处理**:
   - 在 `trap_handler` 中新增分支，处理 S 特权级时钟中断。
   - 重新设置计时器，暂停当前任务并切换到下一个任务。

2. **启用时钟中断**:
   - 在执行第一个应用前，启用 S 特权级时钟中断。
   - 设置第一个 10ms 的计时器。

3. **核心函数**:
   - `suspend_current_and_run_next`: 暂停当前任务并切换到下一个任务。
   - `set_next_trigger`: 设置下一个时钟中断触发时间。

通过这些机制，我们实现了一个高效的抢占式调度系统，确保多任务操作系统中的任务能够公平地竞争 CPU 资源。

---

### 多道程序与分时多任务操作系统的全流程描述

#### 1. 系统初始化

**文件**: `src/main.rs`

**主要步骤**:
- `rust_main` 函数初始化系统，包括启用时钟中断和设置第一个 10ms 的计时器。

**关键函数和变量**:
- `trap::enable_timer_interrupt()`
- `timer::set_next_trigger()`
- `task::run_first_task()`

#### 2. 启用 S 特权级时钟中断

**文件**: `src/trap/mod.rs`

**主要步骤**:
- `enable_timer_interrupt` 函数设置 S 特权级时钟中断使能位，确保时钟中断不会被屏蔽。

**关键函数和变量**:
- `sie::set_stimer()`

#### 3. 设置计时器

**文件**: `src/timer.rs`

**主要步骤**:
- `set_next_trigger` 函数设置下一个时钟中断触发时间。
- `get_time` 函数获取当前的 `mtime` 计数器值。
- `set_timer` 函数设置 `mtimecmp` 的值。

**关键函数和变量**:
- `CLOCK_FREQ`
- `TICKS_PER_SEC`
- `set_timer(get_time() + CLOCK_FREQ / TICKS_PER_SEC)`

#### 4. 加载应用程序

**文件**: `src/loader.rs`

**主要步骤**:
- `load_apps` 函数将应用程序加载到内存。
- `init_app_cx` 函数初始化任务控制块，构造 Trap 上下文，并返回应用程序的初始栈指针。

**关键函数和变量**:
- `get_num_app()`
- `TaskContext::goto_restore()`

#### 5. 任务上下文和任务控制块

**文件**: `src/task/context.rs`

**主要步骤**:
- `TaskContext` 结构体定义任务上下文，包括返回地址（ra）、栈指针（sp）和保存寄存器（s0~s11）。
- `TaskControlBlock` 结构体定义任务控制块，包括任务状态和上下文。

**关键函数和变量**:
- `TaskContext::goto_restore()`

#### 6. 任务管理器

**文件**: `src/task/mod.rs`

**主要步骤**:
- `TASK_MANAGER` 全局实例初始化任务管理器。
- `suspend_current_and_run_next` 函数暂停当前任务并切换到下一个任务。
- `run_next_task` 函数调度并运行下一个任务。
- `find_next_task` 函数查找下一个准备运行的任务。

**关键函数和变量**:
- `TaskManager`
- `TaskManagerInner`
- `mark_current_suspended()`
- `mark_current_exited()`

#### 7. 任务切换

**文件**: `src/task/switch.S` 和 `src/task/switch.rs`

**主要步骤**:
- `__switch` 函数使用汇编实现任务上下文的保存和恢复，进行任务切换。

**关键函数和变量**:
- `__switch`

#### 8. 时钟中断处理

**文件**: `src/trap/mod.rs`

**主要步骤**:
- `trap_handler` 函数处理 S 特权级时钟中断，重新设置计时器，并进行任务切换。

**关键函数和变量**:
- `scause.cause()`
- `Trap::Interrupt(Interrupt::SupervisorTimer)`
- `set_next_trigger()`
- `suspend_current_and_run_next()`

### 总结

通过以上步骤，我们实现了一个多道程序与分时多任务的操作系统。关键步骤包括系统初始化、启用时钟中断、设置计时器、加载应用程序、管理任务上下文和任务控制块、实现任务切换以及处理时钟中断。这些步骤确保了操作系统能够高效地调度任务，实现公平的时间片轮转调度。

---







---





