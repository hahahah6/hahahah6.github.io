---
title: FreeRTOS_StudyNotes
date: 2025-11-17
permalink: /posts/2025/11/FreeRTOS_StudyNotes/
tags:
  - embedded-systems
  - freertos
  - stm32
---

# FreeRTOS 任务状态转换图

  

以下是 FreeRTOS 任务的四种状态及其转换关系：


```mermaid
graph RL
    A[挂起状态] -- "vTaskResume()" --> B[就绪状态]
    B -- "vTaskSuspend()" --> A
    B -- "调度器分配CPU" --> C[运行状态]
    C -- "时间片用尽/任务切换" --> B
    C -- "调用阻塞API" --> D[阻塞状态]
    D -- "事件触发" --> B
```


## 状态说明

  

1.  **挂起状态 (Suspended)**：任务被主动挂起，通过 `vTaskSuspend()` 进入。

2.  **就绪状态 (Ready)**：任务可以运行，但等待调度器分配 CPU。

3.  **运行状态 (Running)**：任务正在执行。

4.  **阻塞状态 (Blocked)**：任务因等待某事件（如信号量、延时）而暂时停止，事件触发后返回就绪状态。

  

## 任务状态切换函数

-  `vTaskSuspend()`：将任务从就绪或运行状态切换到挂起状态。

-  `vTaskResume()`：将任务从挂起状态切换到就绪状态。

- 调用阻塞 API（如 `vTaskDelay()`）：从运行状态切换到阻塞状态。

- 事件触发（如信号量释放）：从阻塞状态切换到就绪状态。

  

---

  

## 环境搭建

  

1.  **FPU 使能**

- 打开 CubeMX

- 进入 FREERTOS > Config parameters > MCU/FPU

- 选择对应的 Cortex-M4/M7 浮点数加速选项

  

2.  **启用 USE_NEWLIB_REENTRANT**

- 打开 CubeMX

- 进入 FREERTOS > Advanced settings

  

### 问题背景：多任务环境中的共享资源

在多任务环境（例如 RTOS）中，多个任务可能会调用相同的库函数（如 `malloc()`、`printf()`、`fopen()` 等）。这些函数通常涉及共享资源：

- 输入输出缓冲区（如 `stdin`、`stdout`）

- 内存管理（如堆）

- 文件描述符等

  

如果未正确隔离资源，可能会导致资源冲突或数据混乱。

  

### USE_NEWLIB_REENTRANT 的作用

启用 `USE_NEWLIB_REENTRANT` 后，Newlib C 库会为每个任务分配独立的资源副本：

- 独立的输入输出缓冲区，避免打印内容混淆。

- 独立的堆和内存管理，避免内存争用。

  

---

  

## 任务创建与删除

  

1.  **动态创建任务**

```c

BaseType_t  xTaskCreate();

```

2.  **静态创建任务**

```c

TaskHandle_t  xTaskCreateStatic();

```

- 使用 `vApplicationGetIdleTaskMemory()` 为空闲任务分配内存

- 使用 `vApplicationGetTimerMemory()` 为软件定时器分配内存

  

3.  **任务删除**

```c

vTaskDelete(TaskHandle_t  xTask);

```

-  `NULL` 参数表示删除当前运行的任务。

- 动态创建的任务内存会在空闲任务中释放。

- 静态创建的任务需要手动释放申请的内存，否则会导致内存泄漏。

  

---

  

## 任务挂起与恢复

  

1.  **挂起任务**

```c

vTaskSuspend(TaskHandle_t  xTask);

```

- 如果传入 `NULL`，表示挂起当前运行的任务。

  

2.  **恢复任务**

```c

vTaskResume(TaskHandle_t  xTask);

```

  

3.  **中断中恢复任务**

```c

vTaskResumeFromISR(TaskHandle_t  xTask);

```

- 适用于中断服务函数（ISR）。

- 返回值：

-  `pdTRUE`：任务恢复后需要进行任务切换。

-  `pdFALSE`：任务恢复后不需要进行任务切换。

- 注意：

- 中断优先级不能高于 FreeRTOS 所管理的最高优先级（5~15）。

- 任务优先级数值越高，优先级越高；中断优先级数值越小，优先级越高。

  
  

# **FreeRTOS 中断管理**

  

## 1. NVIC 优先级分组说明

  


| 优先级分组             | 抢占优先级        | 子优先级        | 优先级配置寄存器高 4 位                  |
|------------------------|-------------------|-----------------|-----------------------------------------|
| **NVIC_PriorityGroup_0** | 0 级抢占优先级    | 0-15 级子优先级 | 0bit 用于抢占优先级，4bit 用于子优先级  |
| **NVIC_PriorityGroup_1** | 0-1 级抢占优先级  | 0-7 级子优先级  | 1bit 用于抢占优先级，3bit 用于子优先级  |
| **NVIC_PriorityGroup_2** | 0-3 级抢占优先级  | 0-3 级子优先级  | 2bit 用于抢占优先级，2bit 用于子优先级  |
| **NVIC_PriorityGroup_3** | 0-7 级抢占优先级  | 0-1 级子优先级  | 3bit 用于抢占优先级，1bit 用于子优先级  |
| **NVIC_PriorityGroup_4** | 0-15 级抢占优先级 | 0 级子优先级    | 4bit 用于抢占优先级，0bit 用于子优先级  |


  

## 2. 中断优先级分组设置

1. 低于 `configMAX_SYSCALL_INTERRUPT_PRIORITY`（一般为5）的优先级的中断才运行，才能调用 FreeRTOS 的 API 函数。

- 优先级（5~15）允许调用 FreeRTOS 的 API 函数。

2. 建议将所有优先级指定为抢占优先级，方便 FreeRTOS 管理。

- 调用 `HAL_NVIC_SetPriorityGrouping(NVIC_PRIORITYGROUP_4)`。

3. 任务优先级数值越高，优先级越高；中断优先级数值越小，优先级越高。

  

## 3. 中断相关寄存器

1. 3个中断优先级配置寄存器

1.  `SHPR1` 地址 `0xE000ED18`

2.  `SHPR2` 地址 `0xE000ED1C`

3.  `SHPR3` 地址 `0xE000ED20`

  

2. 3个中断屏蔽寄存器

1.  `PRIMASK`：这是个只有 1 个位的寄存器。当它置 1 时，关闭所有可屏蔽的异常，只剩下 NMI 和硬 fault 可以响应。它的缺省值是 0，表示没有关闭中断。

2.  `FAULTMASK`：这是个只有 1 位的寄存器。当它置 1 时，只有 NMI 才能响应，所有其它的异常，包括中断和 fault，通通关闭。它的缺省值也是 0，表示没有关闭异常。

3.  `BASEPRI`：这个寄存器最多有 9 位（由表达优先级的位数决定）。它定义了被屏蔽优先级的阈值。当它被设成某个值时，所有优先级号大于等于此值的中断都被关（优先级号越大，优先级越低）。在 FreeRTOS中断的主要手段。

  

**例子：**

-  `BASEPRI` 设置为 `0x50` 代表中断优先级在 5~15 均被屏蔽，0~4 的中断优先级正常执行。

  

3.  **功能及实现：**

-  `portENABLE_INTERRUPTS()`: 开启中断。

-  `portDISABLE_INTERRUPTS()`: 关闭中断，将所有的可屏蔽中断（maskable interrupts）暂时屏蔽，以确保当前代码段在执行过程中不会被中断。

  

**功能及实现**：

- 在不同的平台上，`portDISABLE_INTERRUPTS` 的具体实现可能不同，通常会涉及到如下操作：

- 设置中断屏蔽寄存器：

- 在 ARM Cortex-M 内核上，`portDISABLE_INTERRUPTS` 通常通过设置 `BASEPRI` 寄存器实现，屏蔽优先级低于某个阈值的中断。

- 例如：将 `BASEPRI` 设置为一个较高的优先级值，从而屏蔽掉低优先级的中断。

- 全局中断禁用：

- 在某些架构上，可能直接禁用所有中断（包括高优先级的不可屏蔽中断）。这通常通过设置 CPU 的全局中断屏蔽位实现，例如 ARM 的 CPSID I 指令。

  

**使用场景：**

-  `portDISABLE_INTERRUPTS` 常用于以下场景：

- 关键代码段保护：当某段代码需要避免中断打断时，使用 `portDISABLE_INTERRUPTS` 禁用中断。例如，对共享资源的访问，防止中断干扰导致数据不一致。

- 切换上下文前后：FreeRTOS 的任务切换涉及修改任务堆栈和调度器状态，需要禁用中断以避免中断在关键操作期间打断切换过程。

  

## 4. FreeRTOS 几个特殊中断：

1.  **PendSV (Pendable Service Call)**

-  **作用**：PendSV 中断是一个低优先级中断，通常用于在任务切换时进行延迟和延迟任务管理。

-  **用途**：PendSV 中断在 RTOS 中用于任务切换（context switching）和系统调用的切换，这样可以确保在进行高优先级的任务调度时不会被打断。

  

2.  **SysTick**

-  **作用**：SysTick 是一种系统时钟中断，用于产生定时中断。

-  **用途**：在 RTOS 中，SysTick 通常用作系统时钟源，用于实现系统计时、任务调度、时间片轮转等功能。

  

3.  **HardFault**

-  **作用**：硬故障中断，用于处理处理器内部的致命错误（如内存访问错误、堆栈溢出等）。

-  **用途**：用于捕获硬件故障或异常，例如内存访问越界等，确保系统可以进行故障处理或重启。

  

4.  **BusFault**

-  **作用**：总线故障中断，用于处理处理器总线错误。

-  **用途**：当发生总线错误（如非法地址访问、总线错误等）时，BusFault 中断会触发，用于诊断和处理这些错误。

  

5.  **UsageFault**

-  **作用**：使用故障中断，用于捕获程序的使用错误。

-  **用途**：当程序访问非法内存地址、除零、堆栈溢出等使用错误时，UsageFault 中断会触发，用于进行错误处理和调试。

  

**提示**：

- 使用 `portDISABLE_INTERRUPTS();` 是否可以调用 `vTaskDelay`？

- 答：在调用 `portDISABLE_INTERRUPTS()` 禁用中断后，不可以调用 `vTaskDelay()` 或其他涉及 FreeRTOS 调度器的 API。

-  **原因分析**：

-  `vTaskDelay()` 依赖 FreeRTOS 调度器：vTaskDelay() 的作用是让当前任务进入阻塞状态，并将控制权交还给调度器，以便其他任务可以运行。

- 如果中断被禁用，调度器无法触发任务切换（因为调度器的核心依赖于 SysTick 中断），导致 FreeRTOS 的整体运行中断。

- 禁用中断会阻止系统正常运行：portDISABLE_INTERRUPTS() 禁用所有优先级低于 BASEPRI 的中断，包括 FreeRTOS 调度所依赖的 SysTick 中断。

- 如果 `vTaskDelay()` 被调用，调度器尝试切换任务，但由于中断被禁用，任务切换无法完成，系统可能会陷入死锁或卡死。

  

5. FreeRTOS API 的使用限制：

- 在临界区或中断禁用状态下，FreeRTOS 规定只能使用特定的 API，例如以 `FromISR` 结尾的函数（如 `xQueueSendFromISR`）。

-  `vTaskDelay()` 并不是一种中断安全的 API，因此在中断被禁用时不可使用。

### 临界段保护及调度器的挂起与恢复

  

#### 1. 临界段代码保护

  

##### 介绍：

-  **临界段**：必须完整运行，不能被打断的代码段。

-  **适用场景**：

1. 外设：如IIC、SPI等。

2. 系统：系统自身的需求。

3. 用户：用户自定义的需求。

  

##### 函数：

- 进入临界段时，FreeRTOS会关闭中断，处理完临界段代码后再打开中断。

  

-  **临界段代码保护函数**：

-  `taskENTER_CRITICAL()`：任务级进入临界段（关闭中断）。

-  `taskEXIT_CRITICAL()`：任务级退出临界段（开启中断）。

-  `taskENTER_CRITICAL_FROM_ISR()`：中断级进入临界段（关闭中断）。

-  `taskEXIT_CRITICAL_FROM_ISR()`：中断级退出临界段（开启中断）。

  

##### 任务级临界区调用格式示例：

```c

taskENTER_CRITICAL();

{

/* 临界区代码 */

}

taskEXIT_CRITICAL();

```

  

##### 中断级临界区调用格式示例：

```c

uint32_t save_status;

save_status = taskENTER_CRITICAL_FROM_ISR();

{

/* 临界区代码 */

}

taskEXIT_CRITICAL_FROM_ISR(save_status);

```

  

##### 特点：

1.  **成对使用**：`taskENTER_CRITICAL` 和 `taskEXIT_CRITICAL` 必须配对使用。

2.  **支撑嵌套**：可以嵌套调用，但系统会确保中断管理正确。

3.  **保持临界段耗时短**：避免关闭中断时间过长，导致系统延时。

4.  **强悍**：保证临界段内代码不被打断。

  

---

  

#### 2. 任务调度器的挂起与恢复

  

##### 介绍：

-  **挂起任务调度器**：在此期间，任务调度器会暂停，但中断仍然可以正常响应。

  

##### 函数：

-  `vTaskSuspendAll()`：挂起任务调度器。

-  `vTaskResumeAll()`：恢复任务调度器。

  

##### 特点：

1.  **与临界区不同**：挂起任务调度器不需要关闭中断。

2.  **仅防止任务间的资源争夺**：中断仍然可以直接响应。

3.  **适用场景**：用于任务间的临界区处理，不需要延迟中断，可以安全地管理任务间的临界区。

  

##### 示例：

```c

vTaskSuspendAll();

{

/* 任务间临界区内容 */

}

vTaskResumeAll();

```

  

---

  

## FreeRTOS的列表和列表项

  

### 简介

  

在FreeRTOS中，列表（`List_t`）和列表项（`ListItem_t`）是用于管理任务和其他数据结构的核心部分。虽然它们的概念与传统链表类似，FreeRTOS中的列表实际上是**双向环形链表**，用来管理任务调度和其他与任务相关的数据。

  

-  **列表**：类似于链表，是一个双向环形链表，通常用来管理任务。

-  **列表项**：存储在列表中的数据项，类似于链表的节点。

  

#### 列表特点

  

-  **列表**：列表项的地址不连续，且它们在内存中是人为连接的。列表中的项数是动态的，可以根据需要随时改变。

-  **数组**：数组的元素地址是连续的，且一旦定义后，数组的大小通常是固定的。

  

在操作系统中，任务的数量和状态是动态变化的，因此使用链表结构来管理任务是非常合适的。

  
  

#### 1.**列表结构 (`List_t`)**

  

`List_t` 是 FreeRTOS 中表示列表的结构体，定义了一个双向环形链表。

  

```c

typedef  struct xLIST

{

listFIRST_LIST_INTEGRITY_CHECK_VALUE /* 校验值 */

volatile  UBaseType_t uxNumberOfItems; /* 列表中列表项的数量 */

ListItem_t * configLIST_VOLATILE pxIndex; /* 遍历列表项的指针 */

MiniListItem_t xListEnd; /* 迷你列表项，标记链表的尾部 */

listSECOND_LIST_INTEGRITY_CHECK_VALUE /* 校验值 */

} List_t;

```

##### 成员变量说明：

  

1.  **校验值（`listFIRST_LIST_INTEGRITY_CHECK_VALUE` 和 `listSECOND_LIST_INTEGRITY_CHECK_VALUE`）**

这两个宏是已知的常量，FreeRTOS会使用它们来检查列表在程序运行过程中是否发生了破坏。这通常用于调试时开启，默认情况下不会启用。

2.  **`uxNumberOfItems`**

记录列表中项的数量（不包括尾部的迷你列表项 `xListEnd`）。这是列表的大小，随时可以变化。

  

3.  **`pxIndex`**

用于遍历列表的指针，指向列表中的某个项。它在操作列表时非常有用，尤其是需要遍历整个链表时。

  

4.  **`xListEnd`**

这个成员是一个迷你列表项，它位于列表的尾部。它的作用是标记链表的结束，通常用来简化链表的操作，不需要存储额外的信息，如 `pvOwner` 和 `pxContainer`。

  

#### 2. **列表项结构 (`ListItem_t`)**

  

每个列表项（`ListItem_t`）存储一个数据项，通常与任务相关的信息。这些列表项通过 `pxNext` 和 `pxPrevious` 成员形成双向环形链表。

  

##### 结构体定义

```c

typedef  struct xLIST_ITEM

{

listFIRST_LIST_ITEM_INTEGRITY_CHECK_VALUE /* 校验值 */

TickType_t xItemValue; /* 列表项的值 */

struct xLIST_ITEM * pxNext; /* 下一个列表项 */

struct xLIST_ITEM * pxPrevious; /* 上一个列表项 */

} ListItem_t;

```

  

##### 成员变量说明：

  

1.  **`xItemValue`**

这是列表项的值，用于排序或标识列表项。FreeRTOS常根据该值来决定任务的优先级或其他特定排序。

  

2.  **`pxNext` 和 `pxPrevious`**

这两个指针分别指向列表项的下一个项和上一个项，从而形成双向链表的结构。它们使得在遍历或修改列表时更加高效。

  

3.  **校验值（`listFIRST_LIST_ITEM_INTEGRITY_CHECK_VALUE`）**

这个值用于调试和验证列表项数据的完整性，通常也用于检测数据是否遭到破坏。

  
  

#### **总结**

  

- FreeRTOS中的**列表**是一个**双向环形链表**，用于动态管理任务或其他对象的状态。

- 每个列表包含多个**列表项**，每个列表项代表一个数据项，通常与任务的优先级、状态等信息有关。

- 列表和列表项结构设计简洁，提供高效的增删改查操作，适合用于实时操作系统中任务调度等场景。

- 列表的尾部由一个**迷你列表项**（`xListEnd`）标记，用于简化链表操作并优化内存使用。

---

  

### 列表项

  

列表项是列表中用于存放数据的地方。在 `list.h` 文件中，有列表项的相关结构体定义：

  

##### 结构体定义

```c

struct xLIST_ITEM

{

listFIRST_LIST_ITEM_INTEGRITY_CHECK_VALUE; /* 用于检测列表项的数据完整性 */

configLIST_VOLATILE TickType_t xItemValue; /* 列表项的值 */

struct xLIST_ITEM * configLIST_VOLATILE pxNext; /* 下一个列表项 */

struct xLIST_ITEM * configLIST_VOLATILE pxPrevious; /* 上一个列表项 */

void * pvOwner; /* 列表项的拥有者 */

struct xLIST * configLIST_VOLATILE pxContainer; /* 列表项所在列表 */

listSECOND_LIST_ITEM_INTEGRITY_CHECK_VALUE; /* 用于检测列表项的数据完整性 */

};

  

typedef  struct xLIST_ITEM ListItem_t;

```

#### 列表项成员变量说明

  

1.  **成员变量 `xItemValue`**

这是列表项的值，用于按升序对列表中的列表项进行排序。

  

2.  **成员变量 `pxNext` 和 `pxPrevious`**

分别用于指向列表中列表项的下一个和上一个列表项。

  

3.  **成员变量 `pvOwner`**

用于指向包含列表项的对象（通常是任务控制块）。

  

4.  **成员变量 `pxContainer`**

用于指向列表项所在的列表。

  

##### 列表项结构示意图

  

<pre>
xList_Item
├── xItemValue
├── *pvOwner
├── *pxContainer
├── *pxPrevious
└── *pxNext
</pre>


  

---

  

### 迷你列表项

  

迷你列表项（`xMINI_LIST_ITEM`）是 FreeRTOS 中链表的一种优化设计，用于简化链表的操作，特别是在标记列表的末尾和挂载其他列表项时。它与常规的列表项（`xLIST_ITEM`）有所不同，省略了一些不必要的成员变量，从而节省内存开销。

  

##### 结构体定义

  

```c

struct xMINI_LIST_ITEM

{

listFIRST_LIST_ITEM_INTEGRITY_CHECK_VALUE /* 用于检测数据完整性 */

configLIST_VOLATILE TickType_t xItemValue; /* 列表项的值 */

struct xLIST_ITEM * configLIST_VOLATILE pxNext; /* 下一个列表项 */

struct xLIST_ITEM * configLIST_VOLATILE pxPrevious; /* 上一个列表项 */

};

typedef  struct xMINI_LIST_ITEM MiniListItem_t;

```

  

##### 成员变量说明

  

1.  **`xItemValue`**：表示列表项的值。这个值通常用于排序列表中的项。

2.  **`pxNext`**：指向下一个列表项。

3.  **`pxPrevious`**：指向上一个列表项。

4.  **省略的成员变量**：迷你列表项省略了 `pvOwner` 和 `pxContainer`，减少了内存消耗，因为它只用于标记链表的末尾，并挂载其他列表项。

  

#### 迷你列表项的值如何得出

  

迷你列表项的 `xItemValue` 在 FreeRTOS 中通常具有固定的值，它有以下几个用途：

  

##### 1. **初始化时的最大值**

  

-  `xItemValue` 在初始化时通常会被设置为系统所能表示的最大值。例如，在 32 位系统中，`xItemValue` 会被初始化为 `0xFFFFFFFF`。

- 这样设计的目的是保证插入的所有普通列表项的值都小于迷你列表项的 `xItemValue`，确保迷你列表项总是作为列表的“边界”存在。

  

##### 示例：

  

```c

MiniListItem_t xMiniListItem;

  

/* 将迷你列表项的值初始化为最大值 */

  

xMiniListItem.xItemValue = portMAX_DELAY;

```

  

##### 2. **列表排序的锚点**

  

迷你列表项作为链表的边界，通常处于链表的末尾。在链表排序中，它总是列表中 `xItemValue` 最大的元素。其他列表项会按照 `xItemValue` 递增顺序插入到迷你列表项前。

  

##### 3. **便于操作的简化值**

  

由于迷你列表项只用于标记链表尾部，它的 `xItemValue` 值是固定的，不会动态改变。这简化了链表操作，避免了不必要的内存开销，同时提升了操作效率。

  
  
  

FreeRTOS 列表相关的 API 函数介绍，具体如下：

  
| **函数**               | **描述**               |
|------------------------|------------------------|
| `vListInitialise()`    | 初始化列表             |
| `vListInitialiseItem()`| 初始化列表项           |
| `vListInsertEnd()`     | 列表末尾插入列表项     |
| `vListInsert()`        | 列表插入列表项         |
| `uxListRemove()`       | 列表移除列表项         |

---

  

## 函数 `void vListInitialise(List_t * const pxList)`

  

### 函数功能

初始化一个列表 (`List_t`)，设置其初始状态，使其可以用于后续的操作。

  

#### 参数说明

| 参数 | 描述 |
|--------|--------------|
| pxList | 待初始化的列表 |

  

### 函数实现

```c

void  vListInitialise(List_t * const  pxList)

{

/* 初始化时，列表中只有 xListEnd，因此 pxIndex 指向 xListEnd */

pxList->pxIndex = ( ListItem_t * ) &( pxList->xListEnd );

  

/* xListEnd 的值初始化为最大值，用于列表项升序排序时，排在最后 */

pxList->xListEnd.xItemValue = portMAX_DELAY;

  

/* 初始化时，列表中只有 xListEnd，因此上一个和下一个列表项都为 xListEnd 本身 */

pxList->xListEnd.pxNext = ( ListItem_t * ) &( pxList->xListEnd );

pxList->xListEnd.pxPrevious = ( ListItem_t * ) &( pxList->xListEnd );

  

/* 初始化时，列表中的列表项数量为 0（不包含 xListEnd） */

pxList->uxNumberOfItems = ( UBaseType_t ) 0U;

  

/* 初始化用于检测列表数据完整性的校验值 */

listSET_LIST_INTEGRITY_CHECK_1_VALUE( pxList );

listSET_LIST_INTEGRITY_CHECK_2_VALUE( pxList );

}

```

  

## 函数 `vListInitialiseItem()`

  

### 函数声明

```c

void  vListInitialiseItem(ListItem_t * const  pxItem);

```

### 参数说明

| 参数 | 描述 |
|--------|------------------|
| pxItem | 待初始化的列表项 |

  

### 函数功能

初始化一个列表项 (`ListItem_t`)，将其设置为初始状态。

  

### 函数实现

```c

void  vListInitialiseItem(ListItem_t * const  pxItem)

{

/* 初始化时，列表项所在列表设为空 */

pxItem->pxContainer = NULL;

  

/* 初始化用于检测列表项数据完整性的校验值 */

listSET_FIRST_LIST_ITEM_INTEGRITY_CHECK_VALUE(pxItem);

listSET_SECOND_LIST_ITEM_INTEGRITY_CHECK_VALUE(pxItem);

}

```

  

### 初始化后的列表项结构

```plaintext

xList_Item:

xItemValue

pvOwner

*pxContainer = NULL

*pxPrevious

*pxNext

```

  

## 函数 `vListInsert()`

  

#### 函数声明

```c

void  vListInsert(List_t * const  pxList, ListItem_t * const  pxNewListItem);

```

  

#### 函数功能

此函数用于将待插入列表项按照列表项值升序进行排序，有序地插入到列表中。

  
  

#### 参数说明

| 参数 | 描述 |
|-----------------|------------|
| `pxList` | 列表 |
| `pxNewListItem` | 待插入列表项 |

  

#### 内容

  

```c

void  vListInsert( List_t * const  pxList, ListItem_t * const  pxNewListItem )

{

ListItem_t * pxIterator;

const  TickType_t xValueOfInsertion = pxNewListItem->xItemValue; /* 获取列表项的数值，依据数值排序 */

listTEST_LIST_INTEGRITY( pxList ); /* 检查参数是否正确 */

listTEST_LIST_ITEM_INTEGRITY( pxNewListItem ); /* 检查参数是否正确 */

  

if( xValueOfInsertion == portMAX_DELAY )

{

pxIterator = pxList->xListEnd.pxPrevious; /* 插入的位置为列表 xListEnd 前面 */

}

else

{

for( pxIterator = ( ListItem_t * ) &( pxList->xListEnd ); /* 遍历列表中的列表项，找到插入的位置 */

pxIterator->pxNext->xItemValue <= xValueOfInsertion;

pxIterator = pxIterator->pxNext )

{

}

}

  

pxNewListItem->pxNext = pxIterator->pxNext; /* 将待插入的列表项插入指定位置 */

pxNewListItem->pxNext->pxPrevious = pxNewListItem;

pxNewListItem->pxPrevious = pxIterator;

pxIterator->pxNext = pxNewListItem;

pxNewListItem->pxContainer = pxList; /* 更新待插入列表项所在列表 */

  

( pxList->uxNumberOfItems )++; /* 更新列表中列表项的数量 */

}

  

```

#### 注释翻译说明

  

1.  **获取列表项的数值，依据数值排序**

获取 `pxNewListItem` 的 `xItemValue`，用于判断插入位置。

  

2.  **检查参数是否正确**

使用 `listTEST_LIST_INTEGRITY` 和 `listTEST_LIST_ITEM_INTEGRITY` 确保列表和项的完整性。

  

3.  **插入的位置为列表 xListEnd 前面**

如果 `xValueOfInsertion` 的值为 `portMAX_DELAY`，将新项插入到列表尾部（`xListEnd` 的前一个位置）。

  

4.  **遍历列表中的列表项，找到插入的位置**

当 `xValueOfInsertion` 不为 `portMAX_DELAY` 时，遍历列表，找到第一个 `xItemValue` 大于插入值的位置。

  

5.  **将待插入的列表项插入指定位置**

修改指针，插入 `pxNewListItem` 到正确位置。

  

6.  **更新待插入列表项所在列表**

设置 `pxContainer` 指向 `pxList`，并更新 `uxNumberOfItems` 表示列表项数量增加。

  

---

  

## 函数 `vListInsertEnd`

  

#### 内容

```c

void  vListInsertEnd( List_t * const  pxList, ListItem_t * const  pxNewListItem )

{

// 省略部分非关键代码 …

/* 获取列表 pxIndex 指向的列表项 */

ListItem_t * const pxIndex = pxList->pxIndex;

  

/* 更新待插入列表项的指针成员变量 */

pxNewListItem->pxNext = pxIndex;

pxNewListItem->pxPrevious = pxIndex->pxPrevious;

  

/* 更新列表中原本列表项的指针成员变量 */

pxIndex->pxPrevious->pxNext = pxNewListItem;

pxIndex->pxPrevious = pxNewListItem;

  

/* 更新待插入列表项的所在列表成员变量 */

pxNewListItem->pxContainer = pxList;

  

/* 更新列表中列表项的数量 */

( pxList->uxNumberOfItems )++;

}

```

  

#### 参数说明

| **形参** | **描述** |
|------------------|-------------------|
| `pxList` | 列表 |
| `pxNewListItem` | 待插入列表项 |

  

#### 注释翻译说明

  

1.  **获取列表 `pxIndex` 指向的列表项**

获取 `pxList` 的 `pxIndex` 成员，用作插入操作的目标位置。

  

2.  **更新待插入列表项的指针成员变量**

设置 `pxNewListItem` 的 `pxNext` 和 `pxPrevious`，指向正确的位置。

  

3.  **更新列表中原本列表项的指针成员变量**

修改 `pxIndex` 的前一项和后一项的指针，使其正确连接到 `pxNewListItem`。

  

4.  **更新待插入列表项的所在列表成员变量**

设置 `pxNewListItem->pxContainer` 指向当前列表 `pxList`。

  

5.  **更新列表中列表项的数量**

增加列表项计数变量 `uxNumberOfItems`。

  

---

#### 功能说明

该函数用于将待插入的列表项插入到 `pxList` 的尾部（即 `pxIndex` 指针指向的列表项之前）。这是一种**无序的插入方式**。

  

---

  

## 函数 `uxListRemove()`

  

#### 声明

```c

UBaseType_t  uxListRemove(ListItem_t * const  pxItemToRemove)

```

#### 内容

```c

UBaseType_t  uxListRemove(ListItem_t * const  pxItemToRemove)

{

List_t * const pxList = pxItemToRemove->pxContainer;

  

/* 从列表中移除列表项 */

pxItemToRemove->pxNext->pxPrevious = pxItemToRemove->pxPrevious;

pxItemToRemove->pxPrevious->pxNext = pxItemToRemove->pxNext;

  

/* 如果 pxIndex 正指向待移除的列表项 */

if (pxList->pxIndex == pxItemToRemove)

{

/* pxIndex 指向上一个列表项 */

pxList->pxIndex = pxItemToRemove->pxPrevious;

}

else

{

mtCOVERAGE_TEST_MARKER();

}

  

/* 将待移除的列表项的所在列表指针清空 */

pxItemToRemove->pxContainer = NULL;

  

/* 更新列表中列表项的数量 */

(pxList->uxNumberOfItems)--;

  

/* 返回移除后列表项的数量 */

return  pxList->uxNumberOfItems;

}

```

#### 功能描述

  

此函数用于将列表项从列表项所在的列表中移除。

  

#### 参数说明

  

| 参数 | 描述 |
|--------------------|----------------|
| `pxItemToRemove` | 待移除的列表项 |

  
  

#### 返回值

  

| 类型 | 描述 |
|--------|--------------------------------------------|
| 整数 | 待移除列表项移除后，所在列表剩余列表项的数量 |

  
  

#### 功能描述

  

该函数的功能是将指定的 `pxItemToRemove` 列表项从其所属的列表中移除，调整相关指针并更新列表中剩余的列表项数量。

  
  

#### 关键步骤

  

1.  **从列表中移除列表项：**

- 更新待移除列表项的前后指针，使其从链表中脱离。

  

2.  **更新 `pxIndex` 指针：**

- 如果列表的 `pxIndex` 正指向被移除的项，则将其调整为指向被移除项的前一个列表项。

  

3.  **清空列表指针：**

- 将待移除列表项的 `pxContainer` 设置为 `NULL`。

  

4.  **更新列表项数量：**

- 列表中的数量减 1。

  

5.  **返回值：**

- 返回移除后的列表项数量。

  

---

### 参考

代码详情可查阅《FreeRTOS开发指南》第七章——“FreeRTOS列表和列表项”。

  

---