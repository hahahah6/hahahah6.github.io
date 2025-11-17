---
title: FreeRTOS_StudyNotes
date: 2025-11-17
# FreeRTOS Task State Transition Diagram

Below are the four FreeRTOS task states and how they transition between each other:

```mermaid
graph RL
  A[Suspended] -- "vTaskResume()" --> B[Ready]
  B -- "vTaskSuspend()" --> A
  B -- "Scheduler allocates CPU" --> C[Running]
  C -- "Time slice expires / context switch" --> B
  C -- "Calls blocking API" --> D[Blocked]
  D -- "Event triggered" --> B
```

## State Descriptions

1. **Suspended**: The task is explicitly suspended by calling `vTaskSuspend()`.
2. **Ready**: The task is ready to run but is waiting for the scheduler to give it CPU time.
3. **Running**: The task is currently executing on the CPU.
4. **Blocked**: The task is waiting for an event (for example, a semaphore or delay) and will return to the Ready state once the event occurs.

## Task State Transition Functions

- `vTaskSuspend()`: Moves a task from the Ready or Running state into Suspended.
- `vTaskResume()`: Moves a task from Suspended back into Ready.
- Blocking APIs such as `vTaskDelay()`: Move a task from Running into Blocked.
- Events (for example, a semaphore being given): Move a task from Blocked back into Ready.

---

## Environment Setup

1. **Enable the FPU**
   - Open CubeMX.
   - Navigate to `FREERTOS > Config parameters > MCU/FPU`.
   - Select the floating-point acceleration option that matches your Cortex-M4/M7 target.

2. **Enable `USE_NEWLIB_REENTRANT`**
   - Open CubeMX.
   - Navigate to `FREERTOS > Advanced settings` and enable the option.

### Background: Shared Resources in Multitasking Environments

In an RTOS, multiple tasks might invoke the same library functions such as `malloc()`, `printf()`, or `fopen()`. These functions access shared resources:

- I/O buffers (for example, `stdin`, `stdout`).
- Memory management structures (the heap).
- File descriptors and related handles.

Without proper isolation, these shared resources can easily become corrupted.

### Why `USE_NEWLIB_REENTRANT` Matters

When `USE_NEWLIB_REENTRANT` is enabled, the Newlib C library creates per-task copies of critical resources:

- Dedicated I/O buffers prevent log output from different tasks from interleaving.
- Independent heap and memory management metadata reduce the risk of contention.

---

## Creating and Deleting Tasks

1. **Dynamically Creating a Task**

```c
BaseType_t xTaskCreate();
```

2. **Statically Creating a Task**

```c
TaskHandle_t xTaskCreateStatic();
```

- Use `vApplicationGetIdleTaskMemory()` to allocate memory for the idle task.
- Use `vApplicationGetTimerMemory()` to allocate memory for the timer service task.

3. **Deleting a Task**

```c
vTaskDelete(TaskHandle_t xTask);
```

- Passing `NULL` deletes the currently running task.
- Memory for dynamically created tasks is reclaimed by the idle task.
- For statically created tasks, you must release any allocated memory yourself to avoid leaks.

---

## Suspending and Resuming Tasks

1. **Suspend a Task**

```c
vTaskSuspend(TaskHandle_t xTask);
```

- Passing `NULL` suspends the task that is currently running.

2. **Resume a Task**

```c
vTaskResume(TaskHandle_t xTask);
```

3. **Resume a Task from an ISR**

```c
vTaskResumeFromISR(TaskHandle_t xTask);
```

- Can only be called from an interrupt service routine (ISR).
- Return values:
  - `pdTRUE`: A context switch is required after the ISR exits.
  - `pdFALSE`: No context switch is necessary.
- Notes:
  - Interrupt priorities must not be higher (numerically lower) than the maximum priority managed by FreeRTOS (typically 5).
  - Higher task priority numbers represent higher task priorities, whereas lower interrupt priority numbers represent higher interrupt priorities.

# FreeRTOS Interrupt Management

## 1. NVIC Priority Grouping

| Priority Group             | Preempt Priority Range | Subpriority Range | Upper 4 Bits of Priority Register |
|----------------------------|------------------------|-------------------|-----------------------------------|
| `NVIC_PriorityGroup_0`     | 0                      | 0–15              | 0 bits preempt, 4 bits subpriority|
| `NVIC_PriorityGroup_1`     | 0–1                    | 0–7               | 1 bit preempt, 3 bits subpriority |
| `NVIC_PriorityGroup_2`     | 0–3                    | 0–3               | 2 bits preempt, 2 bits subpriority|
| `NVIC_PriorityGroup_3`     | 0–7                    | 0–1               | 3 bits preempt, 1 bit subpriority |
| `NVIC_PriorityGroup_4`     | 0–15                   | 0                 | 4 bits preempt, 0 bits subpriority|

## 2. Interrupt Priority Configuration

1. Only interrupts with priorities numerically greater than or equal to `configMAX_SYSCALL_INTERRUPT_PRIORITY` (usually 5) may call FreeRTOS APIs.
   - Interrupt priorities in the 5–15 range are safe to use with FreeRTOS APIs.
2. Prefer using only preempt priorities for clarity; configure with `HAL_NVIC_SetPriorityGrouping(NVIC_PRIORITYGROUP_4)`.
3. Remember: higher task priority numbers mean higher task priority, but lower interrupt priority numbers mean higher interrupt urgency.

## 3. Interrupt-Related Registers

1. Three system handler priority registers:
   - `SHPR1` at `0xE000ED18`
   - `SHPR2` at `0xE000ED1C`
   - `SHPR3` at `0xE000ED20`

2. Three interrupt masking registers:
   - `PRIMASK`: Single-bit register. When set to 1, masks all maskable exceptions; only NMI and HardFault remain active. Default is 0.
   - `FAULTMASK`: Single-bit register. When set to 1, only NMI remains active; all other exceptions are masked. Default is 0.
   - `BASEPRI`: Up to 9 bits (depending on the number of implemented priority bits). Defines the minimum priority value that remains unmasked. Any interrupt with a numerically higher priority value (lower urgency) is disabled.

**Example**

- Setting `BASEPRI` to `0x50` masks interrupt priorities 5–15, allowing priorities 0–4 to run.

### Functions and Implementation Notes

- `portENABLE_INTERRUPTS()`: Re-enables interrupts.
- `portDISABLE_INTERRUPTS()`: Temporarily masks interrupts so that critical code sections are not interrupted.

Depending on the platform, `portDISABLE_INTERRUPTS()` may:

- Set masking registers—for example, on ARM Cortex-M it typically writes `BASEPRI` to block lower-priority interrupts.
- Disable all maskable interrupts globally (such as using the `CPSID I` instruction on ARM).

**Typical Use Cases**

- Protecting critical sections that access shared resources.
- Bracketing context switches to prevent nested interrupts from corrupting task state.

## 4. Notable FreeRTOS-Related Exceptions

1. **PendSV (Pendable Service Call)**
   - Purpose: Low-priority interrupt used to defer context switching.
   - Usage: Ensures task switches occur at a controlled priority level so higher-priority interrupts are serviced first.

2. **SysTick**
   - Purpose: System tick interrupt used to generate periodic ticks.
   - Usage: Provides the system clock for scheduling, time slicing, and delays.

3. **HardFault**
   - Purpose: Handles fatal processor errors such as invalid memory access or stack overflow.
   - Usage: Captures unrecoverable hardware faults so the system can log or recover.

4. **BusFault**
   - Purpose: Reports bus errors (for example, invalid address access).
   - Usage: Helps diagnose bus-related faults when they occur.

5. **UsageFault**
   - Purpose: Reports program errors such as executing undefined instructions or divide-by-zero.
   - Usage: Useful for debugging incorrect application logic.

**Tip**

- It is unsafe to call `vTaskDelay()` while interrupts are disabled with `portDISABLE_INTERRUPTS()`.
  - `vTaskDelay()` relies on the scheduler to place the task into the Blocked state and then select another task.
  - With interrupts disabled, the SysTick handler cannot run, preventing the scheduler from switching tasks and likely causing deadlock.

### FreeRTOS API Restrictions

- In critical sections or when interrupts are disabled, you may only call API variants that end in `FromISR` (such as `xQueueSendFromISR`).
- `vTaskDelay()` is not interrupt-safe and must not be called while interrupts are masked.

### Critical Sections and Suspending the Scheduler

#### 1. Protecting Critical Sections

- A **critical section** must execute atomically and must not be interrupted.
- Common scenarios:
  1. Peripheral access (I²C, SPI, etc.).
  2. Core system requirements.
  3. User-defined synchronization.

When entering a critical section, FreeRTOS masks interrupts until the code completes.

**APIs**

- `taskENTER_CRITICAL()`: Enter a task-level critical section (masks interrupts).
- `taskEXIT_CRITICAL()`: Exit a task-level critical section (restores interrupts).
- `taskENTER_CRITICAL_FROM_ISR()`: Enter a critical section from an ISR.
- `taskEXIT_CRITICAL_FROM_ISR()`: Exit a critical section from an ISR.

**Task-Level Example**

```c
taskENTER_CRITICAL();
{
  /* Critical section code */
}
taskEXIT_CRITICAL();
```

**ISR-Level Example**

```c
uint32_t saved_status;
saved_status = taskENTER_CRITICAL_FROM_ISR();
{
  /* Critical section code */
}
taskEXIT_CRITICAL_FROM_ISR(saved_status);
```

**Characteristics**

1. APIs must be used in pairs.
2. Nested calls are supported; FreeRTOS tracks the nesting depth.
3. Keep critical sections short to minimize interrupt latency.
4. Guarantees that the enclosed code cannot be pre-empted.

---

#### 2. Suspending and Resuming the Scheduler

- Suspending the scheduler pauses task switching but leaves interrupts enabled.

**APIs**

- `vTaskSuspendAll()`: Suspend the scheduler.
- `xTaskResumeAll()`: Resume the scheduler (returns `pdTRUE` if a context switch is required).

**Characteristics**

1. Unlike critical sections, interrupts remain enabled.
2. Prevents other tasks from running, avoiding race conditions between tasks.
3. Use when you need mutual exclusion between tasks but want interrupts to keep firing.

**Example**

```c
vTaskSuspendAll();
{
  /* Critical section spanning tasks */
}
xTaskResumeAll();
```

---

## FreeRTOS Lists and List Items

### Overview

FreeRTOS uses lists (`List_t`) and list items (`ListItem_t`) to manage tasks and other kernel objects. Although they resemble traditional linked lists, FreeRTOS implements them as **doubly linked circular lists** to support efficient scheduling.

- **List**: A doubly linked circular list that tracks multiple items.
- **List item**: A node within the list that stores data about a task or object.

#### List Characteristics

- List items are not stored contiguously; pointers link them together dynamically.
- Arrays, in contrast, have contiguous storage and fixed sizes once created.
- Because task counts and priorities change at runtime, linked lists are a natural fit in an RTOS.

#### 1. `List_t` Structure

`List_t` defines the list data structure:

```c
typedef struct xLIST
{
  listFIRST_LIST_INTEGRITY_CHECK_VALUE
  volatile UBaseType_t uxNumberOfItems;     /* Number of items in the list */
  ListItem_t * configLIST_VOLATILE pxIndex; /* Iteration cursor */
  MiniListItem_t xListEnd;                  /* Marks the end of the list */
  listSECOND_LIST_INTEGRITY_CHECK_VALUE
} List_t;
```

**Members**

1. `listFIRST_LIST_INTEGRITY_CHECK_VALUE` and `listSECOND_LIST_INTEGRITY_CHECK_VALUE`: Optional sentinels used to detect corruption when list integrity checking is enabled.
2. `uxNumberOfItems`: Number of items currently in the list (excluding the sentinel `xListEnd`).
3. `pxIndex`: Iterator pointer used during traversal.
4. `xListEnd`: A mini list item that marks the logical end of the circular list.

#### 2. `ListItem_t` Structure

Each `ListItem_t` represents one entry:

```c
typedef struct xLIST_ITEM
{
  listFIRST_LIST_ITEM_INTEGRITY_CHECK_VALUE
  TickType_t xItemValue;     /* Value used for sorting */
  struct xLIST_ITEM * pxNext;
  struct xLIST_ITEM * pxPrevious;
} ListItem_t;
```

**Members**

1. `xItemValue`: Determines the item's position when inserted (lists are sorted in ascending order by this value).
2. `pxNext` and `pxPrevious`: Pointers to neighboring items, forming the doubly linked list.
3. `listFIRST_LIST_ITEM_INTEGRITY_CHECK_VALUE`: Optional sentinel used for integrity checking.

**Summary**

- Lists provide dynamic, circular double-linked storage for kernel objects.
- Each list contains multiple list items that carry scheduling metadata.
- The sentinel `xListEnd` simplifies list operations and reduces memory overhead.

---

### Detailed List Item Structure

The broader definition from `list.h` includes owner metadata:

```c
struct xLIST_ITEM
{
  listFIRST_LIST_ITEM_INTEGRITY_CHECK_VALUE;
  configLIST_VOLATILE TickType_t xItemValue;
  struct xLIST_ITEM * configLIST_VOLATILE pxNext;
  struct xLIST_ITEM * configLIST_VOLATILE pxPrevious;
  void * pvOwner;                              /* Typically points to the task control block */
  struct xLIST * configLIST_VOLATILE pxContainer;
  listSECOND_LIST_ITEM_INTEGRITY_CHECK_VALUE;
};

typedef struct xLIST_ITEM ListItem_t;
```

**Member Notes**

1. `xItemValue`: Sorting key.
2. `pxNext` / `pxPrevious`: Forward and backward links.
3. `pvOwner`: Pointer to the object that owns the list item (for example, a TCB).
4. `pxContainer`: Pointer back to the parent list.

```
xList_Item
├── xItemValue
├── *pvOwner
├── *pxContainer
├── *pxPrevious
└── *pxNext
```

---

### Mini List Items

A mini list item (`xMINI_LIST_ITEM`) is an optimized sentinel that marks the end of a list. It omits some fields to save memory because it does not need owner or container information.

```c
struct xMINI_LIST_ITEM
{
  listFIRST_LIST_ITEM_INTEGRITY_CHECK_VALUE
  configLIST_VOLATILE TickType_t xItemValue;
  struct xLIST_ITEM * configLIST_VOLATILE pxNext;
  struct xLIST_ITEM * configLIST_VOLATILE pxPrevious;
};

typedef struct xMINI_LIST_ITEM MiniListItem_t;
```

**Member Notes**

1. `xItemValue`: Usually set to the maximum possible value to act as a boundary.
2. `pxNext`: Points to the first real list item.
3. `pxPrevious`: Points to the last real list item.
4. Owner/container fields are omitted to reduce memory usage.

**Determining `xItemValue`**

- During list initialization, `xItemValue` is set to `portMAX_DELAY` (the largest representable value). Because items are inserted in ascending order, all real items have lower values and therefore appear before the mini list item.

```c
MiniListItem_t xMiniListItem;
xMiniListItem.xItemValue = portMAX_DELAY;
```

**Roles of the Mini List Item**

1. Acts as a sorting anchor at the end of the list.
2. Provides a fixed boundary value that simplifies insertion and iteration logic.

---

### List API Summary

| Function                | Description                              |
|-------------------------|------------------------------------------|
| `vListInitialise()`     | Initialize a list                         |
| `vListInitialiseItem()` | Initialize a list item                    |
| `vListInsertEnd()`      | Insert an item at the end of a list       |
| `vListInsert()`         | Insert an item into a sorted list         |
| `uxListRemove()`        | Remove an item from its containing list   |

---

## Function `void vListInitialise(List_t * const pxList)`

### Purpose

Initializes a `List_t` so that it is ready for use.

### Parameters

| Parameter | Description         |
|-----------|---------------------|
| `pxList`  | Pointer to the list |

### Implementation

```c
void vListInitialise(List_t * const pxList)
{
  pxList->pxIndex = (ListItem_t *)&(pxList->xListEnd);      /* Iterator starts at the end item */
  pxList->xListEnd.xItemValue = portMAX_DELAY;              /* Sentinel holds the max value */
  pxList->xListEnd.pxNext = (ListItem_t *)&(pxList->xListEnd);
  pxList->xListEnd.pxPrevious = (ListItem_t *)&(pxList->xListEnd);
  pxList->uxNumberOfItems = (UBaseType_t)0U;
  listSET_LIST_INTEGRITY_CHECK_1_VALUE(pxList);
  listSET_LIST_INTEGRITY_CHECK_2_VALUE(pxList);
}
```

## Function `vListInitialiseItem()`

### Declaration

```c
void vListInitialiseItem(ListItem_t * const pxItem);
```

### Parameters

| Parameter | Description            |
|-----------|------------------------|
| `pxItem`  | Pointer to the list item |

### Purpose

Initializes a `ListItem_t` before it is inserted into a list.

### Implementation

```c
void vListInitialiseItem(ListItem_t * const pxItem)
{
  pxItem->pxContainer = NULL;                                /* Not yet linked to any list */
  listSET_FIRST_LIST_ITEM_INTEGRITY_CHECK_VALUE(pxItem);
  listSET_SECOND_LIST_ITEM_INTEGRITY_CHECK_VALUE(pxItem);
}
```

### Resulting Structure

```plaintext
xList_Item:
xItemValue
pvOwner
*pxContainer = NULL
*pxPrevious
*pxNext
```

## Function `vListInsert()`

### Declaration

```c
void vListInsert(List_t * const pxList, ListItem_t * const pxNewListItem);
```

### Purpose

Inserts an item into a list so that the list remains sorted by `xItemValue`.

### Parameters

| Parameter        | Description      |
|------------------|------------------|
| `pxList`         | Target list      |
| `pxNewListItem`  | Item to insert   |

### Implementation

```c
void vListInsert(List_t * const pxList, ListItem_t * const pxNewListItem)
{
  ListItem_t * pxIterator;
  const TickType_t xValueOfInsertion = pxNewListItem->xItemValue;

  listTEST_LIST_INTEGRITY(pxList);
  listTEST_LIST_ITEM_INTEGRITY(pxNewListItem);

  if (xValueOfInsertion == portMAX_DELAY)
  {
    pxIterator = pxList->xListEnd.pxPrevious;              /* Insert before the sentinel */
  }
  else
  {
    for (pxIterator = (ListItem_t *)&(pxList->xListEnd);
       pxIterator->pxNext->xItemValue <= xValueOfInsertion;
       pxIterator = pxIterator->pxNext)
    {
      /* Iterate to find the correct insertion point */
    }
  }

  pxNewListItem->pxNext = pxIterator->pxNext;
  pxNewListItem->pxNext->pxPrevious = pxNewListItem;
  pxNewListItem->pxPrevious = pxIterator;
  pxIterator->pxNext = pxNewListItem;
  pxNewListItem->pxContainer = pxList;
  (pxList->uxNumberOfItems)++;
}
```

**Key Points**

1. Fetch `xItemValue` to determine where to insert the item.
2. Validate list and item integrity macros.
3. If `xItemValue` equals `portMAX_DELAY`, insert the item just before the sentinel.
4. Otherwise, iterate to find the first list item with a larger `xItemValue`.
5. Update forward/backward pointers so the list remains consistent.
6. Update `pxContainer` and increment the item count.

---

## Function `vListInsertEnd()`

### Implementation

```c
void vListInsertEnd(List_t * const pxList, ListItem_t * const pxNewListItem)
{
  ListItem_t * const pxIndex = pxList->pxIndex;

  pxNewListItem->pxNext = pxIndex;
  pxNewListItem->pxPrevious = pxIndex->pxPrevious;

  pxIndex->pxPrevious->pxNext = pxNewListItem;
  pxIndex->pxPrevious = pxNewListItem;

  pxNewListItem->pxContainer = pxList;
  (pxList->uxNumberOfItems)++;
}
```

### Parameters

| Parameter        | Description      |
|------------------|------------------|
| `pxList`         | Target list      |
| `pxNewListItem`  | Item to insert   |

**Notes**

1. Uses `pxIndex` as the insertion cursor.
2. Updates the new item's forward/backward pointers.
3. Relinks the adjacent list items.
4. Sets `pxContainer` and increments the item count.

This function appends items without sorting; it is typically used for ready lists where ordering is maintained elsewhere.

---

## Function `uxListRemove()`

### Declaration

```c
UBaseType_t uxListRemove(ListItem_t * const pxItemToRemove)
```

### Implementation

```c
UBaseType_t uxListRemove(ListItem_t * const pxItemToRemove)
{
  List_t * const pxList = pxItemToRemove->pxContainer;

  pxItemToRemove->pxNext->pxPrevious = pxItemToRemove->pxPrevious;
  pxItemToRemove->pxPrevious->pxNext = pxItemToRemove->pxNext;

  if (pxList->pxIndex == pxItemToRemove)
  {
    pxList->pxIndex = pxItemToRemove->pxPrevious;
  }
  else
  {
    mtCOVERAGE_TEST_MARKER();
  }

  pxItemToRemove->pxContainer = NULL;
  (pxList->uxNumberOfItems)--;

  return pxList->uxNumberOfItems;
}
```

### Description

Removes an item from the list it currently belongs to, fixing pointers and updating the item count.

### Parameters

| Parameter        | Description           |
|------------------|-----------------------|
| `pxItemToRemove` | Item to be removed    |

### Return Value

| Type   | Description                           |
|--------|---------------------------------------|
| Number | Remaining number of items in the list |

**Key Steps**

1. Unlink the item by fixing neighboring pointers.
2. If `pxIndex` pointed to the item, move it to the previous item.
3. Clear `pxContainer` to indicate the item is no longer in a list.
4. Decrement the item count and return the new length.

---

### Reference

For further details, refer to Chapter 7 (“FreeRTOS Lists and List Items”) of the *FreeRTOS Developer Guide*.

---

