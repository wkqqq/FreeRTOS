# FreeRTOS
FreeRTOS学习笔记
## 第一章：任务管理

### 例1:创建任务
```C
void vTask1( void *pvParameters )	//任务的实现代码
{
	while(1)
	{
		...
	}
	
}

int main(void)
{
    	//创建任务,需要说明的是一个实用的应用程序中应检测函数xTaskCreate()的
	//返回值，以确保任务创建成功。返回值有两个：
	//1. pdTRUE	表明任务创建成功
	//2. errCOULD_NOT_ALLOCATE_REQUIRED_MEMORY	由于内存堆空间不足，FreeRTOS
	//无法分配足够的空间来保存任务数据结构和任务栈，因此无法创建任务。
    xTaskCreate( vTask1,		/*指向任务的指针*/
            	"Task1", 		/*任务的文本名字，只会在调试中用到*/
		512, 			/*栈深度*/
		NULL, 			/*没有任务参数*/
		1, 			/*此任务运行在优先级1上，越大运行优先级越高*/
		NULL );		/*不会用到任务句柄*/
	//启动调度器，任务开始执行
	vTaskStartScheduler();
}
```

### 例2：使用任务参数
```C
void vTaskFunction( void *pvParameters )
{
	char *pcTaskName;
	volatile unsigned long ul;
	/*强制转换为字符指针。 */
	pcTaskName = ( char * ) pvParameters;
	while(1)
	{
		vPrintString( pcTaskName );
		for( ul = 0; ul < mainDELAY_LOOP_COUNT; ul++ )
		{
		}
	}
}

/* 定义将要通过任务参数传递的字符串。定义为const，且不是在栈空间上，以保证任务执行时也有效。 */
static const char *pcTextForTask1 = “Task 1 is running\r\n”;
static const char *pcTextForTask2 = “Task 2 is running\t\n”;
int main( void )
{
	/* Create one of the two tasks. */
	xTaskCreate( vTaskFunction, /*指向任务函数的指针. */
				"Task 1", /* 任务名. */
				1000, /* 栈深度. */
				(void*)pcTextForTask1, /* 通过任务参数传入需要打印输出的文本. */
				1, /*任务运行在优先级1上. */
				NULL ); /* 不会用到此任务的句柄. */
	/* 同样的方法创建另一个任务。至此，由相同的任务代码(vTaskFunction)创建了多个任务，仅仅是传入
	的参数不同。同一个任务创建了两个实例。 */
	xTaskCreate( vTaskFunction, "Task 2", 1000, (void*)pcTextForTask2, 1, NULL );
	vTaskStartScheduler();    //启动调度器
	while(1);
}
```
FreeRTOSConfig.h中配置一些常量的值：
>configMAX_PRIORITIES：最大可配置的优先级数目（够用时越小越好）
>
>configTICK_RATE_HZ：系统心跳中断的频率。例：设为1000Hz，则时间片长度为1ms
>
>configUSE_IDLE_HOOK：必须定义为1，这样空闲任务钩子函数才会被调用。
>


### 例3：优先级实验（优先级越大，级别越高，与STM32相反）

运行状态   
非运行状态：  
阻塞状态：正在等待某个事件（1.定时事件 2.同步事件-源于其他任务或中断的事件）的任务则处于“阻塞态”。    
挂起状态：让任务对调度器而言不可见。挂起：vTaskSuspend()  唤醒任务：vTaskResume() 或 vTaskResumeFromISR()  
就绪状态：如果任务处于非运行状态，没有阻塞或挂起，则这个任务处于就绪状态。只是准备运行，但当前尚未运行。

### 例4：利用阻塞态实验延迟
```C
void vTaskFunction( void *pvParameters )
{
	char *pcTaskName;
	pcTaskName = ( char * ) pvParameters;
	for( ;; )
	{

	vPrintString( pcTaskName );
	vTaskDelay( 250 / portTICK_RATE_MS );//任务延迟250ms
	}
}
/*则两个不同优先级的任务都可以得到运行*/
```

### 例5：转换示例任务使用vTaskDelayUntil()，使任务以固定频率运行
```C
void vTaskFunction( void *pvParameters )
{
	char *pcTaskName;
	
	portTickType xLastWakeTime;
	
	pcTaskName = ( char * ) pvParameters;
	
	xLastWakeTime = xTaskGetTickCount();
	
	for( ;; )
	{
	
	vPrintString( pcTaskName );
	/* 本任务将精确的以250MS为周期执行*/
	vTaskDelayUntil( &xLastWakeTime, ( 250 / portTICK_RATE_MS ) );
	}
}
```

### 例6：合并阻塞与非阻塞任务


##空闲任务与空闲任务钩子函数
空闲任务拥有最低优先级(优先级0)以保证其不会妨碍具有更高优先级的应用任务进入运行态  
运行在最低优先级可以保证一旦有更高优先级的任务进入就绪态，空闲任务就会立即切出运行态。  

`空闲任务钩子函数通常被用于:`
* 执行低优先级，后台或需要不停处理的功能代码。
* 测试系统处理裕量(测量出空闲任务占用的处理时间就可以清楚的知道系统有多少富余的处理时间)。
* 将处理器配置到低功耗模式——提供一种自动省电方法，使得在没有任何应用功能需要处理的时候，系统自动进入省电模式。

`空闲任务钩子函数必须遵从以下规则:`
* 绝不能阻或挂起。
* 如果应用程序用到了vTaskDelete() AP 函数，空闲钩子函数必须能够尽快返回。因为在任务被删除后，空闲任务负责回收内核资源

### 例7：定义一个空闲任务钩子函数
```C
unsigned long ulIdleCycleCount = 0UL;
/* 空闲钩子函数必须命名为vApplicationIdleHook(),无参数也无返回值。 */
void vApplicationIdleHook( void )
{
	ulIdleCycleCount++;
}

void vTaskFunction( void *pvParameters )
{
	char *pcTaskName;
	pcTaskName = ( char * ) pvParameters;
	for( ;; )
	{
	/* 打印输出任务名，以及调用计数器ulIdleCycleCount的值。 */
	vPrintStringAndNumber( pcTaskName, ulIdleCycleCount );

	vTaskDelay( 250 / portTICK_RATE_MS );
	}
}
```
### 例8：改变任务优先级
```C
void vTask1( void *pvParameters )
{
	unsigned portBASE_TYPE uxPriority;
	/*查询本任务当前运行的优先级 – 传递一个NULL值表示说“返回我自己的优先级”。 */
	uxPriority = uxTaskPriorityGet( NULL );
	for( ;; )
	{
	vPrintString( "Task1 is running\r\n" );
	/* 把任务2的优先级设置到高于任务1的优先级，会使得任务2立即得到执行(因为任务2现在是所有任务
	中具有最高优先级的任务)。注意调用vTaskPrioritySet()时用到的任务2的句柄。程序清单24将展示
	如何得到这个句柄。 */
	vPrintString( "About to raise the Task2 priority\r\n" );
	vTaskPrioritySet( xTask2Handle, ( uxPriority + 1 ) );
	}
}

void vTask2( void *pvParameters )
{
	unsigned portBASE_TYPE uxPriority;
	/*查询本任务当前运行的优先级 – 传递一个NULL值表示说“返回我自己的优先级”。 */
	uxPriority = uxTaskPriorityGet( NULL );
	for( ;; )
	{
	vPrintString( "Task2 is running\r\n" );
	/* 将自己的优先级设置回原来的值。传递NULL句柄值意味“改变我己自的优先级”。把优先级设置到低
	于任务1使得任务1立即得到执行 – 任务1抢占本任务。 */
	vPrintString( "About to lower the Task2 priority\r\n" );
	vTaskPrioritySet( NULL, ( uxPriority - 2 ) );
	}
}

/* 声明变量用于保存任务2的句柄。 */
xTaskHandle xTask2Handle;
int main( void )
{
	/* 任务1创建在优先级2上。任务参数没有用到，设为NULL。任务句柄也不会用到，也设为NULL */
	xTaskCreate( vTask1, "Task 1", 1000, NULL, 2, NULL );
	/* 任务2创建在优先级1上 – 此优先级低于任务1。任务参数没有用到，设为NULL。但任务2的任务句柄会被
	用到，故将xTask2Handle的地址传入。 */
	xTaskCreate( vTask2, "Task 2", 1000, NULL, 1, &xTask2Handle );
	vTaskStartScheduler();
	for( ;; );
}
```
### 例9：删除任务
```C
xTaskCreate( vTask2, "Task 2", 1000, NULL, 2, &xTask2Handle );
vTaskDelete( xTask2Handle );
```

### 例10：调度算法

* 优先级抢占式调度
* 选择任务优先级
* 协作式调度

































