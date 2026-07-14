## 1.任务管理

#### （1）任务状态

**就绪态：**创建任务，就进入就绪态，已经能够被执行，但是CPU还没有空

**运行态：**任务正在占用CPU

**阻塞态：** 等待延时或等待外部事件发生（等信号量 / 队列 ）

**挂起态：**类似于暂停

**总结**：仅就绪态可以变成运行态，其他状态要变成运行态都必须经过就绪态



#### （2）任务列表

除了运行态，其他三种任务状态都有对应的任务状态列表：

**就绪列表：**pxReadyTasksList[x] ，其中x代表任务优先级数值

**阻塞列表：**pxDelayedTaskList[x]

**挂起列表：**pxSuspendedTasksLists[x]



#### （3）任务堆栈

**每个任务有自己独立栈，用来存局部变量、函数调用、寄存器现场**

栈太小 → 栈溢出 → 直接死机

栈太大 → 浪费 RAM

工程上：一般任务给 256~1024，简单任务 256 足够。

```C
// 1. 定义任务句柄（必须）
TaskHandle_t Handle_Task1;
TaskHandle_t Handle_Task2;

// 2. 任务1函数：LED闪烁（低优先级）
void Task1_LED(void *pvParameters)
{
    while(1) // 任务死循环，不能return
    {
        HAL_GPIO_TogglePin(GPIOA, GPIO_PIN_5); // LED翻转
        vTaskDelay(500); // 阻塞500ms，让出CPU
    }
}

// 3. 任务2函数：串口打印（高优先级）
void Task2_UART(void *pvParameters)
{
    while(1)
    {
        HAL_UART_Transmit(&huart1, (uint8_t*)"Task2 Running\r\n", 14, 100);
        vTaskDelay(1000); // 阻塞1000ms
    }
}

// 4. main函数中创建任务+启动调度器
int main(void)
{
    HAL_Init();
    SystemClock_Config();
    MX_GPIO_Init();
    MX_USART1_UART_Init();

    // 创建任务1：优先级2，堆栈128
    xTaskCreate(Task1_LED, "LED_Task", 128, NULL, 2, &Handle_Task1);
    // 创建任务2：优先级3，比任务1高
    xTaskCreate(Task2_UART, "UART_Task", 128, NULL, 3, &Handle_Task2);

    // 启动调度器（从此CPU由RTOS接管，main函数不会再执行）
    vTaskStartScheduler();

    while(1){} // 调度器启动失败才会走到这里
}
```



---------------



## 2.任务调度

#### （1）抢占优先级+时间片轮转

高优先级任务可以抢占低优先级任务的CPU使用权，被抢占的任务进入就绪态

同优先级任务时间片轮转，任务调度器会在每个时钟节拍到来的时候切换任务

```C
// 任务1：低优先级2，循环打印
void Task1_Low(void *pvParameters)
{
    while(1)
    {
        HAL_UART_Transmit(&huart1, (uint8_t*)"低优先级任务运行\r\n", 20, 100);
        // 无延时，一直占用CPU
    }
}

// 任务2：高优先级3，每隔1s打印
void Task2_High(void *pvParameters)
{
    while(1)
    {
        HAL_UART_Transmit(&huart1, (uint8_t*)"高优先级任务抢占！\r\n", 22, 100);
        vTaskDelay(1000);
    }
}
```



#### （2）临界区保护

多任务同时改同一个全局变量 / 硬件时，会乱套。

```c
// 共享全局变量
uint32_t share_num = 0;

void Task1(void *pvParameters)
{
    while(1)
    {
        // 进入临界区：这段代码不被任何任务/中断打断
        taskENTER_CRITICAL();
        share_num++; // 操作共享变量
        taskEXIT_CRITICAL(); // 退出临界区
        
        vTaskDelay(100);
    }
}
```

作用：这段代码不被打断，保证数据安全。



#### （3）上下文切换

切换任务时，CPU 把当前任务的寄存器保存到栈，再加载下一个任务的寄存器。



#### （4）系统时钟

**时间片：** 同等优先级任务轮流享有CPU的时间（可设置），叫做时间片，在FreeRTOS中，一个时间片等于SysTick中断周期

**两个延时函数：**

- `vTaskDelay(100)`：**相对延时**，延时 100ms，期间任务阻塞，CPU 给别人用（90% 场景用它）

- `vTaskDelayUntil()`：**绝对延时**，固定周期执行（比如 100ms 采集一次传感器，保证周期精准）

  ```C
  // 1. 相对延时 vTaskDelay：从调用时开始延时
  void Task_Delay(void *pvParameters)
  {
      while(1)
      {
          // 执行逻辑...
          vTaskDelay(100); // 阻塞100个Tick（100ms）
      }
  }
  
  // 2. 绝对延时 vTaskDelayUntil：固定周期执行
  void Task_Period(void *pvParameters)
  {
      TickType_t LastWakeTime = xTaskGetTickCount(); // 记录初始时刻
      const TickType_t Frequency = 100; // 周期100ms
  
      while(1)
      {
          // 执行传感器采集逻辑...
          // 绝对延时：保证每隔100ms执行一次，不受任务执行时间影响
          vTaskDelayUntil(&LastWakeTime, Frequency);
      }
  }
  ```

  

-------

### 3.同步机制

#### （1）二值信号量

作用：只有 **0 / 1**，主打**事件同步**，一个任务 / 中断唤醒另一个任务。

**典型场景：**

串口收到数据 → 释放信号量 → 唤醒串口处理任务

按键中断 → 释放信号量 → 唤醒按键解析任务

```C
SemaphoreHandle_t BinarySem_Handle; // 二值信号量句柄

// main函数中创建信号量
BinarySem_Handle = xSemaphoreCreateBinary();
// 初始化释放一次（避免第一次获取阻塞）
xSemaphoreGive(BinarySem_Handle);

// 外部中断服务函数（按键中断）
void EXTI0_IRQHandler(void)
{
    HAL_GPIO_EXTI_IRQHandler(GPIO_PIN_0);
}

// 中断回调函数
void HAL_GPIO_EXTI_Callback(uint16_t GPIO_Pin)
{
    if(GPIO_Pin == GPIO_PIN_0)
    {
        BaseType_t xHigherPriorityTaskWoken = pdFALSE;
        // 中断中专用API：释放信号量
        xSemaphoreGiveFromISR(BinarySem_Handle, &xHigherPriorityTaskWoken);
        // 触发任务切换
        portYIELD_FROM_ISR(xHigherPriorityTaskWoken);
    }
}

void Task_SemWait(void *pvParameters)
{
    while(1)
    {
        // 等待信号量，portMAX_DELAY=永久阻塞
        if(xSemaphoreTake(BinarySem_Handle, portMAX_DELAY) == pdPASS)
        {
            // 信号量获取成功：执行按键处理逻辑
            HAL_UART_Transmit(&huart1, (uint8_t*)"按键触发，任务唤醒\r\n", 24, 100);
        }
    }
}
```

**核心用途**：中断只发信号，耗时逻辑放任务，不阻塞中断（嵌入式标准写法）。



#### （2）计数信号量

**作用：**限制同时访问的资源数量（比如 3 个串口缓冲区），生产者消费者模型。有 **0~N**，主打**资源计数**（限最多 N 个任务用）

#### （3）互斥锁

**作用：互斥**，保护共享资源，防止多任务同时操作。

典型场景：

- 两个任务都要读写 SPI Flash

- 多个任务修改同一个全局参数

  特点：主打**临界资源互斥访问**，自带**优先级继承**，避免高优先级任务被卡住（优先级反转）。
  
  ```C
  SemaphoreHandle_t Mutex_Handle; // 互斥锁句柄
  
  // main函数中创建互斥锁
  Mutex_Handle = xSemaphoreCreateMutex();
  
  // 任务1：写Flash
  void Task_WriteFlash(void *pvParameters)
  {
      while(1)
      {
          // 获取互斥锁
          xSemaphoreTake(Mutex_Handle, portMAX_DELAY);
          // 临界区：操作SPI Flash
          SPI_Flash_WriteData(0x00, 123);
          // 释放互斥锁
          xSemaphoreGive(Mutex_Handle);
          
          vTaskDelay(1000);
      }
  }
  
  // 任务2：读Flash
  void Task_ReadFlash(void *pvParameters)
  {
      while(1)
      {
          xSemaphoreTake(Mutex_Handle, portMAX_DELAY);
          // 临界区：操作SPI Flash
          SPI_Flash_ReadData(0x00);
          xSemaphoreGive(Mutex_Handle);
          
          vTaskDelay(1000);
      }
  }
  ```
  
  **作用**：互斥锁保证同一时间只有一个任务操作 Flash，避免数据错乱。
  
  
  
  1. 包含关系
  
  - 计数信号量：通用版，N 个名额
  
  - 二值信号量：计数信号量的特例（最大计数 = 1），谁都能拿、**谁都能放**。
  
  - 互斥锁：**加了所有权 + 优先级继承**的特殊二值信号量，**谁加锁，只能谁解锁**。
  
    2.**功能重叠但定位完全不同**
  
  - 想做**等待事件、唤醒任务** → 用：二值信号量
  - 想做**限制资源数量、限流** → 用：计数信号量
  - 想做**多任务抢同一个硬件 / 全局变量、防冲突** → 用：互斥锁
  - 

#### （4）事件组

作用：一个任务**等待多个事件**，全部满足才执行。

例：等待 “按键按下 + 传感器数据就绪” 才处理。

---------------

### 4.通信机制

### （1）队列 Queue（FreeRTOS 通信核心）

**作用：**任务 / 中断之间传递结构化数据。

**典型场景：**

- 按键任务 → 队列 → 显示任务（传键值）
- 传感器采集任务 → 队列 → 无线发送任务（传采集值）

```c
QueueHandle_t DataQueue_Handle; // 队列句柄
#define QUEUE_LEN 5    // 队列长度
#define QUEUE_SIZE 4    // 每个元素4字节（int）

// main函数中创建队列：存储5个int类型数据
DataQueue_Handle = xQueueCreate(QUEUE_LEN, QUEUE_SIZE);

void Task_SendQueue(void *pvParameters)
{
    uint32_t send_data = 0;
    while(1)
    {
        send_data++; // 模拟传感器数据
        // 往队列发数据，超时0
        xQueueSend(DataQueue_Handle, &send_data, 0);
        vTaskDelay(500);
    }
}

void Task_RecvQueue(void *pvParameters)
{
    uint32_t recv_data;
    while(1)
    {
        // 从队列取数据，永久等待
        if(xQueueReceive(DataQueue_Handle, &recv_data, portMAX_DELAY) == pdPASS)
        {
            // 处理数据
            HAL_UART_Transmit(&huart1, (uint8_t*)&recv_data, 4, 100);
        }
    }
}
```



----------------



## 5.高级特性层（进阶优化，80% 项目不用）

这一层是**锦上添花**，只有做低功耗、高性能、超复杂项目时才用到，普通学习、做小项目可以直接跳过。

## 1. 内存管理

FreeRTOS 提供 5 种堆方案 heap_1~heap_5，区分动态 / 静态分配。

- 解决：内存碎片、内存泄漏、RAM 紧张问题
- 工程上：默认用 heap_4 即可，不用深究 5 种区别

## 2. 低功耗管理

- Tickless 模式：关闭系统心跳，让 CPU 深度休眠
- 适用：电池供电设备（手环、表、无线传感器）
- 普通插电设备 / 工控板，完全不用

## 3. 轻量级高效组件

### （1）任务通知 Task Notify

比信号量 / 队列更快、更省 RAM，可以替代二值信号量。

进阶优化用，初学先用信号量即可。

### （2）软件定时器

不用硬件定时器，实现定时回调函数。

简单定时任务可用，不是必须。

## 4. 钩子函数

- 空闲钩子：CPU 空闲时执行（低功耗、后台计数）

- 栈溢出钩子：栈溢出报警，调试用

- 调度器钩子：调度时调用，调试追踪用

  仅调试、优化时使用，正常业务逻辑不用