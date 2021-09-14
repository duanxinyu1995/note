## **RT-Thread** 

### 内核

**启动流程：**汇编代码startup_stm32f103xe.s-->c代码，RT-Thread系统功能初始化-->用户程序入口main()。

**程序内存分布：**

​		Code--代码段，存放程序代码部分；RO-data--只读数据段， 存放程序中定义的常量；

​		RW-data--读写数据段，存放初始化为非0值的全局变量；ZI-data--0数据段，存放未初始化的全局变量及初始化为0的变量。

​		RO Size--包含Code及RO-data，表示程序占Flash空间的大小；RW Size--包含RW-data及ZI-data，运行时占用的RAW大小，烧录时也存进ROM， 运行时搬运至RAM，剩余RAM空间即为动态内存堆。

**自动初始化机制**：

​		初始化函数不需要被显式调用，只需要在函数定义处通过宏定义的方式进行声明，就会在系统启动过程（rtthread_startup())被执行，不同宏定义，则在rtthread_startup()内不同的初始化函数中被自动调用。

**内核对象模型**：

​		系统级的基础设施都是内核对象，如线程、信号量、互斥量、定时器等。静态内核对象--通常放在RW段和ZI段，系统启动后在程序中初始化，占用RAM空间，不依赖内存管理堆；动态内核对象--从内存堆中创建，而后进行手工初始化，当对象被删除后，占用的RAM空间被释放。

​		内核对象管理系统--访问/管理所有内核对象，对象容器（结构体）给每类内核对象分配了一个链表（结构体内包含一个链表），所有内核对象都被链接到该链表上。

```c
//对象容器的数据结构
struct rt_object_information{
    enum rt_object_class_type type;	//对象类型
    rt_list_t object_list;			//对象链表
    rt_size_t object_size			//对象大小
}
```

```c
//对象控制块
struct rt_object{
    char name[RT_NAME_MAX]；			//内核对象名称
    rt_uint8_t type;				//内核对象类型
    rt_uint8_t flag; 				//内核对象参数
    rt_list_t list;					//内核对象管理链表
}
```

```c
void rt_object_init(struct rt_object* object, enum rt_object_class_type type, const char* name)	//初始化静态对象，object--对象控制块的指针
    
void rt_object_detach(rt_object_t object);		//脱离对象，从内核对象管理器链表上删除相应的对象节点，内存未被释放

rt_object_t rt_object_allocate(enum rt_object_class_type type, const char* name) //分配动态对象，先获取对象信息，再分配内存空间，然后进行初始化，最后将其插入所在的对象容器链表中。
    
void rt_object_delete(rt_object_t object)		//删除动态对象，对象容器链表中脱离动态对象，并释放占用的内存
    
rt_err_t rt_object_is_systemobject(rt_object_t object)	//辨别对象，判断一个对象是否为系统对象（静态）
```

​		内核配置：主要通过修改工程目录下的rtconfig.h文件进行，打开/关闭该文件中的宏定义来对代码进行条件编译，达到系统配置和裁剪的目的。

#### 线程管理

​		简介：线程是实现任务的载体，是RT-Thread中最基本的调度单元，描述了任务执行的运行环境，以及任务所处的优先等级，运行环境为上下文（即各个变量和数据）；主要分为两类--系统线程（内核创建）和用户线程（应用程序创建）。

##### 线程管理功能及特点

​		主要功能--对线程进行管理和调度，线程都会从内核对象容器中分配线程对象，线程被删除时，对象也会从对象容器中删除。

​		线程调度器：抢占式，从就绪线程列表中查找最高优先级线程，保证其能被运行；高优先级可立即得到CPU的使用权；中断服务程序使高优先级线程猫族运行条件，则中断完成时，被中断线程直接被挂起，高优先级先运行；线程切换时，保存当前线程的上下文，切回时调度器恢复该线程的上下文。

##### 线程的工作机制：

###### 		**线程控制块**

--用于操作系统管理线程的数据结构，存放线程的信息（优先级、名称、状态等），也包含线程间连接用的链表结构、线程等待事件集合等。

```c
struct rt_thread{
    char name[RT_NAME_MAX];				//线程名称
    rt_uint8_t 		type;				//对象类型
    rt_uint8_t 		flags;				//标志位
    rt_list_t 		list；			   //对象列表
    rt_list_t		tlist;				//线程列表
    /*栈指针与入口指针*/
    void			*sp;				//栈指针
    void			*entry;				//入口函数指针
    void 			*parameter;			//参数
    void 			*stack_addr;		//栈地址指针
    rt_uint32_t		stack_size;			//栈大小
    /* 错误代码 */
    rt_err_t 		error;				//线程错误代码
    rt_uint8_t 		stat;				//线程状态
    /* 优先级 */
    rt_uint8_t 		current_priority;	//当前优先级
    rt_uint8_t 		init_priority;		//初始优先级
    rt_uint32_t		number_mask;
    .......
    rt_ubase_t 		init_tick;			//线程初始化计数值
    rt_ubase_t		remaining_tick;		//线程剩余计数值
    struct rt_timer thread_timer;		//内置线程定时器
    void (*cleanup) (struct rt_thread *tid); //线程退出清除函数，在线程退出时被空闲线程回调，执行清理现场等工作
    rt_uint32_t 	user_data;			//用户数据
}
```

###### 		**线程的重要属性：**

​			**线程栈**--线程切换时保存上下文；第一次运行线程时，可以手工构造上下文设置初始环境-入口函数（PC寄存器）、入口参数（R0寄存器）、返回位置（LR寄存器）、当前机器运行状态（CPSR寄存器）。

​			**线程状态**--初始状态RT_THREAD_INIT（创建未运行，不参与调度）；

​								就绪状态RT_THREAD_READY(线程按优先级排队，等待被执行)；

​								运行状态RT_THREAD_RUNNING(线程正在运行，单核中，只有rt_thread_self()函数返回的正在运行)；

​								挂起状态RT_THREAD_SUSPEND(也称阻塞态，因资源不可用或主动延时而挂起等待，不参与调度)；

​								关闭状态RT_THREAD_CLOSE(线程运行结束，不参与线程调度)；

​			**线程优先级**--被调度的优先程度，支撑256个线程优先级（0~255），数值越小优先级越高。ARM Cortex-M系列普遍为32个优先级，最低优先级默认分配给空闲线程。

​			**时间片**--仅对优先级相同的就绪状态线程有效，起到约束线程单次运行时长的作用，单位为系统节拍（OS Tick）。

​			**线程的入口函数**--线程控制块中的entry为线程的入口函数，线程实现预期功能的函数，两种代码模式如下：

```c
//无限循环模式，让该线程一直被系统循环调度运行，永不删除
void thread_entry(void* paramenter){
    while(1){
        /* 等待事件发生*/
        
        /* 对事件进行服务、处理*/
    }
}

//顺序执行或有限次循环模式，线程一定会被执行完，然后被系统自动删除
static void thread_entry(void* parameter){
    /*处理事务#1*/
    ...
    /*处理事务#2*/
    ...
    /*处理事务#3*/
    ...
}
```

​			**线程错误代码**--与执行环境密切相关，每个线程配备一个变量，用于保存错误码

```c
#define RT_EOK			0			//无错误
#define RT_ERROR		1			//普通错误
#define RT_ETIMEOUT		2			//超时错误
#define RT_EFULL		3			//资源已满
#define RT_EEMPTY		4			//无资源
#define RT_ENOMEM		5			//无内存
#define RT_ENOSYS		6			//系统不支持
#define RT_EBUSY		7			//系统忙
#define RT_EIO			8			//IO错误
#define RT_EINTR		9			//中断系统调用
#define RT_EINVAL		10			//非法参数
```

###### 		线程状态切换

​		线程通过rt_thread_create()或rt_thread_init()进入初始状态(RT_THREAD_INIT);

​		初始状态通过rt_thread_startup()进入就绪状态(RT_THREAD_READY)，调度后进入运行状态(RT_THREAD_RUNNING);

​		运行态调用rt_thread_delay()、rt_sem_take()、rt_mutex_take()、rt_mb_recv()等函数或获取不到资源时，进入挂起状态（RT_THREAD_SUSPEND），若获得资源或等待资源超时，将返回就绪状态；运行结束则执行rt_thread_exit()函数进入关闭状态。（RT_Thread中，实际上线程并不存在运行状态，就绪状态与运行状态时等同的）

​		挂起状态调用rt_thread_delete()、rt_thread_detach()将转换为关闭状态(RT_THREAD_CLOSE)；

###### 	系统线程

​		内核中的系统线程有空闲线程和主线程。

​		空闲线程--系统创建的最低优先级线程，线程永远为就绪状态，系统中无其它就绪线程时被调度，永不会被挂起；

​				特殊用途：线程运行完毕，系统将自动删除线程，自动执行rt_thread_exit(),  从就绪队列中删除，改为关闭状态，然后挂入rt_thread_defunct僵尸队列（资源未回收、处于关闭状态的线程队列）中，由空闲线程回收被删除线程的资源；提供接口运行用户设置的钩子函数，在空闲进程运行时调用钩子函数。

​		主线程--系统启动时创建main线程，入口函数为main_thread_entry()，系统调度器启动后，main线程就开始运行，过程如下

$Sub$$main() --> rtthread_starup() --> rt_application_init() --> main_thread_entry() --> main()

##### 线程的管理方式

```c
rt_thread_creat/init()				//creat创建动态线程（使能系统动态堆heap后才可使用），init初始化静态线程
rt_thread_starup()					//启动
rt_thread_delay/control()...		//运行
rt_thread_delete/detach()			//delete删除，detach脱离
```

###### 创建和删除线程

```c
rt_thread_t rt_thread_creat(const char* name,					//线程名称
                           void (*entry)(void* parameter,)		//线程入口函数
                           void* parameter,						//线程入口函数参数
                           rt_uint32_t stack_size,				//线程栈大小
                           rt_uint8_t priority,					//线程优先级
                           rt_uint32_t tick);					//时间片大小
//创建成功则返回函数句柄，否则返回RT_NULL

rt_err_t rt_thread_delete(rt_thread_t thread);			//把线程完全删除，改线程为关闭状态，空闲线程完成删除动作
//返回RT_EOK则删除成功，-RT_ERROR则失败
```

###### 初始化和脱离线程

```C
rt_err_t rt_thread_init(struct rt_thread* thread,				//线程句柄
                       const char* name,
                       void (*entry)(void* parameter),	
                       void *parameter,
                       void *stack_start, 						//线程栈起始地址
                       rt_uint32_t stack_size,
                       rt_uint8_t priority, rt_uint32_t tick);
//返回RT_EOK则初始化成功，-RT_ERROR则失败

rt_err_t rt_thread_detach(rt_thread_t thread);			//将线程对象从线程队列和内核对象管理器中脱离
```

###### 启动线程

```c
rt_err_t rt_thread_startup(rt_thread_t thread);			
//使创建（初始化）的线程进入就绪状态，若优先级大于当前运行线程，则立刻运行
```

###### 获得当前线程

```c
rt_thread_t rt_thread_self(void);						//获取当前执行的线程句柄
//创建成功则返回函数句柄，否则返回RT_NULL
```

###### 使线程让出处理器资源

```c
rt_err_t rt_thread_yield(void);							
//线程调用后，将自己从坐在的就绪优先级线程队列中删除，挂到队列链表的尾部，依旧在就绪队列中
```

###### 使线程睡眠

```c
//让当前运行的线程延迟一段时间，即线程睡眠，延时过后被唤醒进入就绪状态
rt_err_t rt_thread_sleep(rt_tick_t tick);
rt_err_t rt_thread_delay(rt_tick_t tick);
rt_err_t rt_thread_mdelay(rt_int32_t ms);
```

###### 挂起和恢复线程

​		调用rt_sem_take()、rt_mb_recv()等函数时，资源不可用也将导致线程挂起，等待资源超时或得到资源将回到就绪状态

```c
rt_err_t rt_thread_suspend(rt_thread_t thread);			//将线程挂起，thread为线程句柄
rt_err_t rt_thread_resume(rt_thread_t thread);			//将线程恢复，重新进入就绪状态
```

###### 控制线程

```c
rt_err_t rt_thread_control(rt_thread_t thread, 
                          rt_uint8_t cmd,				//指示控制命令
                          void* arg);					//控制参数
//cmd--RT_THREAD_CTRL_CHANGE_PRIORITY 动态更改线程优先级；RT_THREAD_CTRL_STARTUP 开始运行一个线程；RT_THREAD_CTRL_CLOSE 关闭一个线程
```

###### 设置和删除空闲钩子

```c
//在系统执行空闲线程时自动执行空闲钩子函数来做一些其它的事情，空闲线程永远为就绪，所以钩子函数中不能导致其挂起
rt_err_t rt_thread_idle_sethook(void (*hook)(void));
rt_err_t rt_thread_idle_delhook(void (*hook)(void));
```

###### 设置调度器钩子

```c
//设置调度器钩子，在线程切换时做一些额外的事情
void rt_scheduler_sethook(void (*hook)(struct rt_thread* from, struct rt_therad* to));
void hook(struct rt_thread* from, struct rt_thread* to);	//钩子函数声明
//from 表示系统所要切换出的线程控制块指针，to 表示系统所要切换到的线程控制块 ，
```

#### 时钟管理

##### 	时钟节拍

​		简介--供系统处理所有与事件有关的事件（延时、时间片等），本质是特定的周期性中断，节拍长度根据RT_TICK_PER_SECOND的定义来调整，等于1/RE_TICK_PER_SECOND秒。

###### 		实现方式

​		由配置为中断触发模式的硬件定时器产生，中断到来时调用void rt_tick_increase(void)，通知系统过去了一个系统时钟。

```c
void SysTick_Handler(void){
    //进入中断
    rt_interrupt_enter();
    ...
    rt_tick_increase();
    //退出中断
    rt_interrupt_leave();
}			//中断函数

void rt_tick_increase(void){
    struct rt_thread *thread;
    //全局变量rt_tick自加1
    ++rt_tick;
    //检查时间片
    thread = rt_thread_self();
    -- thread->remaining_tick;
    if(thread->remaining_tick ==0){
        //重新赋值
        thread->remaining_tick = thread->init_tick;
        //线程挂起
        rt_thread_yield();
    }
    //检查系统硬件定时器链表，超时定时器将调用超时函数，从定时器链表中被移除
    rt_timer_check();
}
```

###### 		获取时钟节拍

```c
rt_tick_t rt_tick_get(void);		//返回当前时钟节拍值，显示系统运行的时间长短或测量某任务运行的时间
```

##### 定时器管理

​		硬件定时器--芯片自身提供的定时功能；软件定时器--由操作系统提供的系统接口，使系统提供不受数目限制的定时器服务。

###### 定时器介绍

​		简介：单次触发定时器--定时器启动后只会触发一次定时器事件，然后自动停止定时器；周期触发定时器--周期性地触发定时器事件，除非用户手动停止，否则永远执行下去。根据超时函数所处的上下文环境分为HARD_TIMER模式（中断环境）和SOFT_TIMER模式（线程环境）。

​		HARD_TIMER模式--在初始化/创建定时器时使用参数RT_TIMER_FLAG_HARD_TIMER来指定，超时函数在中断环境，不能去申请/释放动态内存等。

​		SOFT_TIMER模式--通过宏定义RT_USING_TIMER_SOFT启用该模式，启用后，系统初始化时会创建timer线程，然后超时函数会在该线程的上下文环境中执行，初始化/创建定时器时使用RT_TIMER_FLAG_SOFT_TIMER设置为该模式。

###### 定时器工作机制

​		定时器模块中维护两个全局变量：当前系统经过的tick时间rt_tick(硬件中断时自加1)；定时器链表rt_timer_list，新创建与激活的定时器以超时时间的排序方式插入rt_timer_list链表中

​		rt_tick启动后一直记录节拍，创建并启动定时器后，将定时时间加当前节拍数得到超时时间，插入rt_timer_list中，当rt_tick到达该超时时间时，执行超时函数并将该定时器从rt_timer_list链表中删除。

```c
//定时器控制块，用于系统管理定时器，存储定时器信息（初始节拍数、超时节拍数、超时回调函数等）
struct rt_timer{
    struct rt_object parent;
    rt_list_t row[RT_TIMER_SKIP_LIST_LEVEL];		//定时器链表节点
    void (*timeout_func)(void *parameter);			//定时器超时时调用的函数
    void *parameter;								//超时函数的参数
    rt_tick_t init_tick;							//定时器初始超时节拍数
    rt_tick_t timeout_tick;							//定时器实际超时的节拍数
};
typedef struct rt_timer *rt_timer_t;
```

​		定时器跳表算法：加快搜索链表元素的速度。在有序链表中，提取部分中间元素作为索引，小于搜索值可跳着索引，搜索时可以减少比较次数，在RT-Thread中通过宏定义RT_TIMER_SKIP_LIST_LEVEL配置跳表层数，默认为1。

###### 定时器的管理方式

```c
void rt_system_timer_init(void);							//初始化定时器管理系统
void rt_system_timer_thread_init(void);						//使用SOFT_TIMER

/* 动态创建定时器， 成功则返回定时器句柄，否则返回RT_NULL*/
rt_timer_t rt_timer_create(const char* name,				//定时器名称
                          void (*timeout)(void*parameter),	//超时函数指针
                          void* parameter,					//超时函数的入口函数
                          rt_tick_t time,					//定时器的超时时间
                          rt_uint8_t flag);	//定时器创建时的参数，单次或周期定时，硬件或软件定时，可各取一项再或集
rt_err_t rt_timer_delete(rt_timer_t timer);					//系统将定时器从rt_timer_list中删除并释放内存
//返回RT_EOK则成功

/* 初始化和脱离定时器*/
void rt_timer_init(rt_timer_t timer,						//定时器句柄，指向要初始化的定时器控制块
                  const char* name,							
                  void (*timeout)(void * parameter),
                  void* parameter,
                  rt_tick_t time,
                  rt_uint8_t flag);							//比动态创建多一个定时器句柄参数
rt_err_t rt_timer_detach(rt_timer_t timer);					//系统将定时器对象从内核对象容器中脱离，未释放其内存

/* 启动和停止定时器*/
rt_err_t rt_timer_start(rt_timer_t timer);					//启动定时器，使定时器的状态更改为激活状态
rt_err_t rt_timer_stop(rt_timer_t timer);			//停止定时器，使状态更改为停止状态，返回RT_EOK成功

/* 控制定时器*/
rt_err_t rt_timer_control(rt_timer_t timer, 
                          rt_uint8_t cmd,			//控制定时器的命令 
                          void* arg);				//与cmd相对应的参数，补充cmd命令
//cmd支持的命令：RT_TIMER_CTRL_SET_TIME设置超时时间；RT_TIMER_CTRL_GET_TIME获得超时时间；
			// RT_TIMER_CTRL_SET_ONESHOT设置为单次定时；RT_TIMER_CTRL_SET_PERIODIC设置为周期定时器；
```

###### 高精度延时

​		当需要实现比OS Tick更小的延时时，可读取系统某个硬件定时器的计数器或直接使用硬件定时器。

#### 线程间同步

​		同步--按预定的先后次序运行，线程同步则指多个线程通过指定的机制来控制线程之间的执行顺序。

​		核心思想--在访问临界区时只允许一个（或一类）线程运行。

##### 信号量

###### 信号量工作机制

​		用于解决线程间同步问题的内核对象，线程获取或释放它，达到同步或互斥的目的；每个信号量对象含一个信号量值（实例数目或资源数目）和一个线程等待队列，信号量值大于0则有信号量实例可以被使用，等于0则申请信号量的线程将被挂起至等待队列。

###### 信号量控制块

```c
struct rt_semaphre{
    struct rt_ipc_object parent;				//继承自ipc_object类
    rt_uint16_t value;							//信号量的值，最大值为65536
}
typedef struct rt_semaphore* rt_sem_t;			//指向semaphore结构体的指针rt_sem_t
```

###### 信号量的管理方式

```c
//动态创建和删除
rt_sem_t rt_sem_create(const char *name,			//信号量名称
                      rt_uint32_t value,		//信号量初始值
                      rt_uint8_t flag);			//信号量标志，选排队方式RT_IPC_FLAG_FIFO或RT_IPC_FLAG_PRIO
	//返回RT_NULL创建失败，返回信号量控制块指针则成功
rt_err_t rt_sem_delete(rt_sem_t sem);			//删除信号量，若有等待线程，则唤醒，再释放信号量的内存资源

//初始化和脱离
rt_err_t rt_sem_init(rt_sem_t	sem,			//信号量句柄
                     const char	*name,
                     rt_uint32_t value,
                     rt_uint8_t	flag);
rt_err_t rt_sem_detach(rt_sem_t sem);			//使信号量对象从内核对象管理器中脱离

//获取和释放
rt_err_t rt_sem_take(rt_sem_t sem,				//信号量句柄
                     rt_int32_t time);			//等待时间
	//线程通过获取信号量来获得信号量资源实例，信号量大于0时，线程获得信号量，同时信号量值减1
rt_err_t rt_sem_trytake(rt_sem_t sem);			//无等待获取信号量，获取失败时不等待，直接返回
rt_err_t rt_sem_release(rt_sem_t sem);			//释放信号量
	//唤醒挂起在该信号量上的线程，没有等待线程则信号量值加1
```

###### 信号量使用场合

​		线程同步：信号量值表示可用的资源实例，等于0时，线程申请将挂起等待，大于0时可被申请使用资源；

​		锁：二值信号量，0或1，应用于多个线程对同一资源（临界区）的访问，为1时可被访问并上锁置零，其它进程不可访问。

​		中断与线程的同步：设置信号量初值为0，线程等待信号量释放，中断触发后释放信号量。

​		资源计数：适合线程间工作速度不匹配的场合，为快线程计数，后续慢线程可以连续处理多个操作

##### 互斥量

​		相互排斥的二值信号量。

###### 工作机制

​		拥有互斥量的线程拥有互斥量的所有权，支持递归访问（持有互斥锁时递归持有）且能防止线程优先级反转；互斥量只能由持有线程释放。

​		信号量优先级反转：设优先级A>B>C，C先运行并持有M信号量，A就绪运行，同时需求M，所以被挂起，C继续执行，B就绪运行，B先执行完，C再执行完，释放M，A最后才运行。 

​		优先级继承算法：A挂起等待时，将运行中的线程C优先级提升至与A一致，运行完恢复原先的优先级，可避免C与A被B抢先运行

###### 互斥量控制块

```c
struct rt_mutex{
    struct rt_ipc_object parent;			//继承自ipc_object类
    rt_uint16_t			value;				//互斥量的值
    rt_uint8_t			original_priority;	//持有线程的原始优先级
    rt_uint8_t			hold;				//持有线程的持有次数
    struct rt_thread	*owner;				//当前拥有互斥量的线程
};
typedef struct rt_mutex* rt_mutex_t;		//指向互斥量结构体的指针rt_mutex_t
```

###### 互斥量的管理方式

```c
//创建和删除
rt_mutex_t rt_mutex_create(const char* name, rt_uint8_t flag);	//返回互斥量句柄则创建成功，返回RT_NULL则失败
rt_err_t rt_mutex_delete(rt_mutex_t mutex);			//返回RT_EOK则删除成功，所有等待线程获得返回值-RT_ERROR

//初始化和脱离
rt_err_t rt_mutex_init(rt_mutex_t mutex,			//互斥量句柄
                       const char* name,
                       rt_uint8_t flag);
rt_err_t rt_mutex_detach(rt_mutex mutex);		//将互斥量对象从内核对象管理器中脱离，唤醒等待线程返回值-RT_ERROR

//获取和释放
rt_err_t rt_mutex_take(rt_mutex_t mutex, rt_int32_t time);
	//持有线程再次调用则持有计数加1，返回RT_EOK、-RT_ETIMEOUT、-RT_ERROR
rt_err_t rt_mutex_release(rt_mutex_t mutex);
	//持有线程再次调用则持有计数减1，持有计数为零时，互斥量变为可用
```

###### 互斥量使用场合

​		多线程同步避免优先级翻转情况；线程多次持有互斥量，避免递归持有造成死锁。

##### 事件集

​		一个事件集可包含多个事件，完成一对多、多对多的线程间同步。

###### 工作机制

​		多个事件的集合可以用一个32位无符号整型变量来表示，每一位代表一个事件，线程通过“与”（关联型同步）或“或”（独立型同步）将多个事件关联起来。

​		事件集特点：事件只与线程相关，事件间相互独立，每个线程可拥有32个事件标志，每位代表一个时间；仅用于同步；无排队性，多次向线程发送同一事件（不清零），效果等同于一次置一；

###### 事件集控制块

```c
struct rt_event{
    struct rt_ipc_object parent;		//继承自ipc_object类
    rt_uint32_t set;					//事件集合，每位代表一个事件，标记某个事件是否发生
};
typedef struct rt_event* rt_event_t;	//指向事件结构体的指针
```

###### 事件集的管理方式

```c
//创建和删除
rt_event_t rt_event_create(const char* name, rt_uint8_t flag);
	//flag-RT_IPC_FLAG_FIFO、RT_IPC_FLAG_PRIO
rt_event_t rt_event_delete(rt_event_t event);	//删除事件集释放资源，成功则返回RT_EOK

//初始化和脱离
rt_err_t rt_event_init(rt_event_t event, const char* name, rt_uint8_t flag);	//成功则返回RT_EOK
rt_err_t rt_event_detach(rt_event_t event);		//使事件集从内核对象管理器中脱离

//发送和接收
rt_err_t rt_event_send(rt_event_t event, rt_uint32_t set);	//set指定事件标志，设定event事件集对象的事件标志位
rt_err_t rt_event_recv(rt_event_t event,		//事件集对象的句柄
                       rt_uint32_t set, 		//线程感兴趣的事件标志位
                       rt_int8_t option,		//接收选项，设置set是与还是或唤醒，以及是否唤醒后置零
                       rt_int32_t timeout,		//指定超时时间
                       rt_uint32_t* recved);	//指向接收到的事件
```

###### 事件集使用场合

​		接收线程等待多种事件，同时等待多种类型的释放；事件的发送操作在事件未清除时不可累计。

#### 线程间通信

##### 邮箱

###### 	工作机制

​		线程或中断服务将4字节长的邮件发送至邮箱，再由一个或多个线程从邮箱中接收邮件并处理；

​	邮箱控制块

```c
struct rt_mailbox{
    struct rt_ipc_object parent;		//
    rt_uint32_t* msg_pool;				//邮箱缓存区的开始地址
    rt_uint16_t size;					//邮箱缓冲区的大小
    rt_uint16_t entry;					//邮箱中邮件的数目
    rt_uint16_t in_offset, out_offset;	//邮箱缓冲的进出指针
    rt_list_t suspend_sender_thread;	//发送线程的挂起等待队列
};
typedef struct rt_mailbox* rt_mailbox_t;//指向邮件控制块的指针
```

###### 	管理方式

```c
//创建和删除
rt_mailbox_t rt_mb_create(const char* name,		//邮箱名称 
                          rt_size_t size, 		//容量
                          rt_uint*_t flag);		//标志，等待方式FIFO/PRIO
	//动态分配4*size大小内存
rt_mailbox_t rt_mb_delete(rt_mailbox_t mb);		//唤醒所有等待线程（让其返回-RT_ERROR)再删除邮箱对象

//初始化和脱离
rt_err_t rt_mb_init(rt_mailbox_t mb,			//邮箱对象的句柄
                    const char* name,			
                    void* msgpool,				//缓冲区指针
                    rt_size_t size,
                    rt_uint8_t flag,
                    );
rt_err_t rt_mb_detach(rt_mailbox_t mb);			//唤醒所有等待线程，再将邮箱对象从内核对象管理器中脱离

//发送和接收
rt_err_t rt_mb_send(rt_mailbox_t mb, rt_uint32_t value);	//value邮件内容，可为缓冲区首地址
rt_err_t rt_mb_send(rt_mailbox_t mb, rt_uint32_t value, rt_int32_t timeout);
	//等待方式发送邮件，timeout等待时间
rt_err_t rt_mb_recv(rt_mailbox_t mb, 
                    rt_uint32_t* value,			//接收到邮件的存放地址 
                    rt_int32_t timeout);		//超时等待时间
	//返回-RT_ERROR则失败；-RT_ERIMEOUT则超时；RT_EOK则成功
```

###### 使用场合

​		简单的线程间消息传递方式，固定长度（4字节）先进先出，开销较低，效率较高。

##### 消息队列

​		邮箱的扩展，应用于线程间的消息交换、使用串口接收不定长数据等。

###### 工作机制

​		消息队列接收不固定长度的消息，并将消息缓存在自己的内存空间中，消息先进先出，队列为空则将线程挂起。

###### 控制块

```c
struct rt_messagequeue{
    struct rt_ipc_object parent;	
    void* msg_pool;					//指向存放消息的消息池指针
    rt_uint16_t msg_size;			//每个消息的长度
    rt_uint16_ max_msgs;			//最大能够容纳的消息数
    rt_uint16_t entry;				//队列中已有的消息数
    void* msg_queue_head;			//消息链表头
    void* msg_queue_tail;			//消息链表尾
    void* msg_queue_free;			//空闲消息链表
}
typedef struct rt_messagequeue* rt_mq_t; //指向消息队列控制块的指针
```

###### 管理方式

```c
//创建和删除
rt_mq_t rt_mq_creat(const char* name, 
                    rt_size_t msg_size,		//消息队列中一条消息的最大长度（字节）
                    rt_size_t max_msgs,		//消息队列的最大个数
                    rt_uint8_t flag);		//等待方式
	//分配内存大小=（消息大小+消息头）*消息队列最大个数，返回RT_EOK,RT_NULL,消息队列对象句柄。
rt_err_t rt_mq_delete(rt_mq_t mq);			//删除消息队列

//初始化和脱离
rt_err_t rt_mq_init(rt_mq_t mq,
                    const char* name,
                    void *msgpool, 			//指向存放消息的缓冲区指针
                    rt_size_t msg_size,
                    rt_size_t pool_size,	//存放消息的缓冲区大小
                    rt_uint8_t flag);
rt_err_t rt_mq_detach(rt_mq_t mq);			//脱离

//发送和接收
rt_err_t rt_mq_send(rt_mq_t mq, void* buffer,		//消息内容
                    rt_size_t size);				//消息大小
	//无等待，返回RT_EOK, -RT_EFULL, -RT_ERROR
rt_err_t rt_mq_urgent(rt_mq_t mq, void* buffer, rt_size_t size);//发送紧急消息，消息挂到队首优先发送
rt_err_t rt_mq_recv(rt_mq_t mq, void* buffer,				//消息内存缓冲区
                   	rt_size_t size, rt_init32_t timeout);	//超市等待时间
```

###### 使用场合

​		发送消息--不定长消息，且可发送紧急消息；同步消息--两个线程间可采用[消息队列+信号量（确认标志）或邮箱（4字节消息）]

##### 信号

​		软中断信号，在软件层次上对中断机制的一种模拟，线程收到一个信号与处理器收到一个中断请求类似。

###### 工作机制

​		线程给线程发信号，通知线程发生了异步事件，乙方不需要任何操作等待信号，收到信号的处理方法：指定处理函数；忽略；保留系统默认值。

​		信号来到，被挂起线程将状态改为就绪，运行中线程在当前线程栈基础上建立新栈帧空间去处理对应信号。

###### 管理方式

```c
//信号安装
rt_sighandler_t rt_signal_install(int signo,		//信号值（SIGUSR1/SIGUSR2） 
                                  rt_sighandler_t handler);//设置对信号的处理方式
	//返回SIG_ERR则错误信号，返回安装信号前的handle值则成功
	//handler--指向处理函数的指针；SIG_IGN忽略某个信号；SIG_DFL调用默认的处理函数_signal_default_handler()。

//阻塞信号，屏蔽信号，信号不会传达给安装了该信号的线程
void rt_signal_mask(int signo);						//信号值

//解除信号阻塞
void rt_signal_unmask(int signo);	

//发送信号，tid接收信号的线程，sig信号值
int rt_thread_kill(rt_thread_t tid, int sig); //返回RT_EOK则成功，RT_EINVAL参数错误

//等待信号
int rt_signal_wait(const rt_sigset_t *set,		//指定等待的信号
                   rt_siginfo_t *si, 			//指向存储等到信号信息的指针
                   rt_int32_t timeout);			//等待时间
	//返回RT_EOK等到信号；-RT_ETIMEOUT超时；-RT_EINVAL参数错误
```

#### 内存管理

​		变量、中间数据一般存放在RAM，实际使用时调入CPU中进行运算。在用户需要一段内存空间时，向系统提出申请，系统选择合适的内存空间分配给用户，用户使用完毕后，再释放回系统。RT-Thread中两种内存管理方式--动态内存堆管理和静态内存池管理。

##### 功能特点

​		分配内存的时间必须是确定的；内存区域碎片越来越多；嵌入式系统资源环境不同，内存分配算法也不同；

​		分类：内存堆管理（小内存管理算法、slab管理算法、memheap管理算法）和内存池管理算法。

##### 内存堆管理

​		可执行文件（RO段和RW段）处于ROM，启动后RW段移入RAW中，ZI段（0数据段，存放未初始化的全局变量及初始化为0的变量）也存入RAW，剩下ZI段尾部至内存尾部的空间作为内存堆。

​		小内存管理算法：针对小于2MB内存空间的系统；

​		slab管理算法：在系统资源比较丰富时提供一种近似多内存池管理算法的快速算法。

​		memheap管理算法：针对多内存堆的管理，将多个内存粘贴在一起，形成大的内存堆进行管理。

###### 小内存管理算法

​		初始时是一块大的内存，分配内存时分割出相匹配的内存块，剩下的还给堆管理系统。每个内存块包含管理用的数据头和内存数据本身。数据头由magic（变数，初始化成0X1ea0即heap，标记内存块是一个内存管理用的内存数据块）、used（指示当前内存块是否已经分配）、next和prev（前后指针）组成。

​		过程：需要分配内存，从空闲链表指针往后遍历，找到大于所需内存大小+数据头大小的内存块，已分配的跳过，分割相应内存大小，剩余置于next；释放内存时，自动与前后空闲内存块合并。

###### slab管理算法

​		纯粹的缓冲型内存池算法，slab分配器根据对象的大小分成多个区（zone），一个区的大小在32K~128K之间，zone_arry最多包括72种对象（数据块大小从8~16K），每个对象指向链表的首地址，每个链表连接着一串大小相同的数据块。

​		内存分配：slab分配器按照所需大小在zone_arry[]数组中，找到指向合适大小的zone节点，非空则必有数据块，无空闲数据块时zone_arry[]将会删去该zone节点；无对应zone节点则可向页分配器分配一个新的zone。

​		内存释放：分配器根据大小找到内存块所在的zone节点，将内存块连接进去，若zone的空闲链表都被释放，系统将该全空闲的zone释放回页面分配器中。

###### memheap管理算法

​		当系统存在多个内存堆时，用户在系统初始化时将多个所需的memheap初始化，并开启memheap功能就可以很方便的把多个memheap粘合起来，用于系统heap分配。分配内存块时，先从默认的内存堆分配，当分配不够时，自动查找memheap_item链表，从其它内存堆上分配内存块。

###### 内存堆配置和初始化

```c
void rt_system_heap_init(void* begin_addr, void* end_addr);//将begin至end区域的内存空间作为内存堆来使用
rt_err_t rt_memheap_init(struct rt_memheap *memheap,	//memheap控制块
                         const char *name,				//内存堆名称
                         void 		*start_addr,		//堆内存区域起始地址
                         rt_uint32_t size);				//对内存大小
	//使用memheap堆内存时必须在系统初始化时进行堆内存的初始化
```

###### 内存堆的管理方式

```c
//分配和释放内存块
void *rt_malloc(rt_size_t nbytes);		//从系统堆空间中找到合适大小的内存块，将其地址返回
void *rt_free(void *ptr);				//将待释放的内存还给堆管理器

//重新分配内存块
void *rt_realloc(void *rmem, rt_size_t newsize);	//与C语言类似realloc，对rmem内存块的大小重新赋值

//分配多内存块，从内存堆中分配连续内存地址的多个内存块
void *rt_calloc(rt_size_t count,	//内存块数量
                rt_size_t size);	//内存块容量
	//返回起始内存块的地址则成功，返回RT_NULL则失败
```

###### 设置内存钩子函数

```c
//内存分配钩子函数
void rt_malloc_sethook(void (*hook)(void *ptr, rt_size_t size));//在内存分配完成后进行回调
void hook(void *ptr, rt_size_t size);

//内存释放钩子函数
void rt_free_sethook(void (*hook)(void *ptr));//在内存释放完成前进行回调
void hook(void *ptr);
```

内存堆缺点--分配效率不高且容易产生内存碎片

##### 内存池

​		一种内存分配方式，用于分配大量大小相同的内存块，支持线程挂起功能，当内存池无空闲线程块时，申请线程将会被挂起。

###### 工作机制

​		内存池控制块用于管理内存池，存放内存池的信息，如开始地址、内存块大小、内存块列表等，也包含内存块与内存块之间连接用的链表结构、因内存块不可用而被挂起的线程等待事件集合等

```c
struct rt_mempool{
    struct rt_object parent;			
    void *start_address;			//内存池数据区域开始地址
    rt_size_t size;					//内存池数据区域大小
    rt_size_t block_size;			//内存块大小
    rt_uint8_t *block_list;			//内存块列表
    rt_size_t block_total_count;	//内存池数据区域中所能容纳的最大内存块数
    rt_size_t block_free_count;		//内存池中空闲的内存块数
    rt_list_t suspend_thread;		//因为内存块不可用而挂起的线程列表
    rt_size_t suspend_thread_count;	//因内存块不可用而挂起的线程数
};
typedef struct rt_mempool* rt_mp_t;	//指向内存池控制块的指针，表示内存块句柄
```

###### 内存块分配机制

​		内存池申请一大块内存，分成多个等大小的内存块，初始化后内存块大小不可再改变，内存块通过链表连接，分配时从表头开始逐个分配给申请者，同时内存池对象会被分配一个内存池控制块，不同内存块大小的内存池可以并存。

###### 内存池管理方式

```c
//创建和删除内存池（动态）
rt_mp_t rt_mp_creat(const char* name,		//内存池名
                    rt_size_t  block_count,	//内存块数量
                    rt_size_t block_size);	//内存块容量
	//返回内存池句柄则成功创建，否则返回RT_NULL
rt_err_t rt_mp_delete(rt_mp_t mp);	//唤醒等待线程（使其返回-RT_ERROR），再删除内存池对象并释放内存

//初始化和脱离内存池
rt_err_t rt_mp_init(rt_mp_t mp,				//内存池对象
                    const char* name, 
                    void *start, 			//内存池起始地址
                    rt_size_t size, 		//内存块数据区大小，字节单位
                    rt_size_t block_size);	//内存块容量，内存池块个数=size/(block_size+4),4为链表指针
rt_err_t rt_mp_detach(rt_mp_t mp);			//使内存池对象从内核对象管理器中脱离

//分配和释放内存块
void *rt_mp_alloc(rt_mp_t mp, rt_int32_t time);	//从指定的内存池中分配一个内存块，空闲块数目减1，time为等待时间
void rt_mp_free(void *blick);//block为内存块指针，将该内存块加入对应内存池的空闲内存块链表上
```

#### 中断管理

​		CPU再处理内部数据时，外界发生紧急情况，要求CPU暂停当前工作区处理该异步事件，处理完再回到原来被中断的地址，继续原来的工作。

##### Cortex-M CPU架构基础

###### 寄存器介绍

​		Cortex-M系列寄存器R0~R15共16个通用寄存器组（R13为堆栈指针寄存器SP、R14为连接寄存器LR-在调用子程序时，存储返回地址、R15为程序计数器PC），和特殊功能寄存器（程序状态寄存器PSR-保存算数与逻辑标志、中断屏蔽寄存器组PRIMASK/FULLMASK/BASEPRI-控制Cortex_M的中断功能、控制寄存器SCON-用来定义特权级别和当前使用哪个堆栈指针）

###### 操作模式和特权级别

​		操作模式--处理模式（进入异常或中断处理），线程模式

​		运行级别--用户级（线程模式可以工作在特权级或者用户级），特权级（处理模式工作在特权级）

​		可通过CONTROL特殊寄存器控制，复位-》特权线程模式（触发异常进入特权级处理模式）-》用户级线程模式（触发异常进入特权级处理模式）。

###### 嵌套向量中断控制器

​		Cortex-M 中断控制器名为NVIC（嵌套向量中断控制器），中断触发时，处理器将当前运行位置的上下文寄存器（包括PSR、PC、LR、R12、R3~R0）压入中断栈中。

###### PendSV系统调用

​		可悬起的系统调用，可以向普通中断一样被挂起，用来辅助操作系统进行上下文切换，PSV异常初始化为最低优先级。

##### RT-Thread中断工作机制

###### 中断向量表

​		即所有中断处理程序的入口，Cortex-M内核中，所有中断都采用中断向量表的方式进行处理，中断触发，处理器直接判定中断源，跳转到相应的固定位置进行处理。

###### 中断处理过程

​		中断前导程序：保存CPU中断现场，对于Cortex-M，由硬件自动完成，将上下文寄存器压入中断栈中；通知内核进入中断状态，调用rt_interrupt_enter()函数，把全局变量rt_interrupt_nest加1，记录中断嵌套的层数。

​		用户中断服务程序（ISR）：分两种情况--不进行线程切换，用户中断服务程序和中断后续程序运行完毕后退出中断模式；进行线程切换，调用rt_hw_context_switch_interrupt()函数进行上下文切换。

​		中断后续程序：通知内核离开中断状态，调用rt_interrupt_leave()函数，将全局变量rt_interrupt_nest减1；恢复中断前的CPU上下文

###### 中断嵌套

​		在允许中断嵌套的情况下，执行中断服务程序过程中，若出现高优先级的中断，打断当前执行，去执行完高优先级中断再回来执行

###### 中断栈

​		保存当前线程的上下文，RT-Thread提供独立的中断栈，中断发生时，前期处理程序将用户的栈指针更换到中断栈空间中，主堆栈指针MSP为默认堆栈指针，在运行第一个线程之前以及中断和异常服务程序里使用，中断和异常服务程序退出时，修改LR寄存器第二位的值为1，线程的SP由MSP切换至PSP，线程堆栈指针PSP在线程里使用。

###### 中断的底半处理

​		对于某些中断服务程序在取得硬件状态或数据后，还需要进行一系列耗时的处理，则将该中断分为两部分，上半部分取得硬件状态和数据后打开被屏蔽的中断，同时给相关线程发通知进行底半处理（对状态或数据进行进一步处理）。

##### RT-Thread中断管理接口

```c
//中断服务程序挂接，调用该API后，当这个中断源产生中断时，系统将自动调用装载的中断服务程序，注意特权模式下不能挂起线程
rt_isr_handler_t rt_hw_interrupt_install(int vector, 				//挂载的中断号
                                         rt_isr_handler_t handler, 	//新挂载的中断服务程序
                                         void *param,				//作为参数传递给中断服务程序
                                         char *name);				//中断名称

//中断源管理
void rt_hw_interrupt_mask(int vector);	//屏蔽相应的中断vector
void rt_hw_interrupt_umask(int vector);	//打开被屏蔽的中断源vector

//全局中断开关
rt_base_t rt_hw_interrupt_diasble(void);//中断锁，关闭中断，返回函数运行前的中断状态
void rt_hw_interrupt_enable(rt_base_t level);//恢复中断，输入参数是前一次关闭中断的返回值

//中断通知，不能在应用程序中调用中断通知API
void rt_interrupt_enter(void);		//通知内核，当前已经进入中断状态（释放信号量后不能立即切换线程）并增加中断嵌套深度
void rt_interrupt_leave(void);		//通知内核当前离开了中断状态，并减少中断嵌套深度（rt_interrupt_nest++)。

//查询当前嵌套中断深度
rt_uint8_t rt_interrupt_get_nest(void);		//返回0则不处于中断，返回1则处于中断，大于1则为中断嵌套深度
```

##### 中断与轮询

 		轮询模式：不断执行去判断是否能进行下一步；中断模式：当可以进行下一步时发送中断通知

#### 内核移植

#####  CPU架构移植

​		RT-Thread提供libcpu抽象层来适配不同CPU架构，libcpu层向上对内核提供统一的接口，向下提供了一套统一的CPU架构移植接口，包括全局中断开关函数、线程上下文切换函数、时钟节拍的配置和中断函数、Cache等。

###### 实现全局中断开关

​		RT-Thread中提供的一系列线程间同步和通信机制都需要用到libcpu中提供的全局中断开关函数如下

```c
rt_base_t rt_hw_interrupt_disable(void);
void rt_hw_interrupt_enable(rt_base_t level);
```

Cortex-M架构中利用CPS指令实现快速开关中断，如下

```s
CPSID I; PRIMASK=1, 	;关中断
CPSIE I; PRIMASK=0, 	;开中断
```

则实现代码如下

```s
;关闭全局中断 rt_base_t rt_hw_interrupt_disable(void);
rt_hw_interrupt_disable		PROC		;PROC 伪指令定义函数
	EXPORT rt_hw_interrupt_disable		;EXPORT输出定义的函数，类似C语言extern
	MRS	   r0, PRIMASK					;读取PRIMASK寄存器的值到r0寄存器
	CPSID  I							;关闭全局中断（Cortex-M）
	BX	   LR							;函数返回
	ENDP								;ENDP函数结束

;打开全局中断void rt_hw_interrupt_enable(rt_base_t level);
rt_hw_interrupt_enable 		PROC		;定义函数
	EXPORT	rt_hw_interrupt_enable		;输出定义的函数
	MSR		PRIMASK, r0					;将r0寄存器的值写入PRIMASK寄存器
	BX		LR							;
	ENDP								;
```

###### 实现线程栈初始化

​		动态创建线程和初始化线程时，会用到线程初始化函数_rt_thread_init()，该函数会调用栈初始化函数rt_hw_stack_init()，该函数会手动构造一个上下文内容，作为每个线程第一次执行的初始值,PC为线程入口点，LR为线程退出点，R0为线程入口参数，R1 R2 R3 置零待使用。

```c
rt_uint8_t *rt_hw_stack_init(void	*tentry, 			//线程入口函数地址
                             void 	*parameter,			//参数
                             rt_uint8_t *stack_addr,	//栈指针
                             void 	*texit){			//线程退出函数的地址
    struct stack_frame *stack_frame;
    rt_uint8_t 		*stk;
    unsigned long	i;
    /*对传入的栈指针做对齐处理*/
    stk= stack_addr + sizeof(rt_uint32_t);
    stk= (rt_uint8_t *)RT_ALIGN_DOWN((rt_uint32_t)stk, 8);
    std -= sizeof(struct stack_frame);
    /* 得到上下文的栈帧的地址*/
    stack_frame = (struct stack_frame *)stk;
    
    /* 将所有寄存器的默认值设置为0xdeadbeef*/
    for(i= 0; i<sizeof(struct stack_frame)/sizeof(rt_uint32_t);i++)
        ((rt_uint32_t *)stack_frame)[i]= 0xdeadbeef;
    
    /*将第一个参数保存在r0寄存器 */
    stack_frame->exception_stack_frame.r0= (unsigned long)parameter;
    /*剩下r1 r2 r3置零， 线程入口函数地址保存在pc寄存器，退出函数地址保存在lr寄存器*/
    
}
```

###### 上下文切换

```c
rt_hw_context_switch()
```



###### 时钟节拍

```c
rt_tick_increase()
```

##### BSP移植



#### I/O设备管理

##### I/O设备介绍

###### 框架

​		一套I/O设备管理框架，位于硬件和应用程序之间，三层--I/O设备管理层、设备驱动框架层、设备驱动层。

​		设备驱动层：一组驱使硬件设备工作的程序，是先访问硬件设备的功能。负责创建和注册I/O设备。过程如下：

​				1）设备驱动根据设备模型定义，创建具备硬件访问能力的设备实例rt_device_create(int type, int attach_size)，将设备通过rt_device_register()接口注册到I/O设备管理器中。

​				2）应用程序通过rt_device_find()接口查找设备，使用I/O设备管理接口访问硬件。

###### I/O设备模型

```c
struct rt_device{
    struct rt_object			parent;							//内核对象基类
    enum rt_device_class_type	type;							//设备类型
    rt_uint16_t					flag;							//设备参数
    rt_uint16_t					open_flag;						//设备打开标志
    rt_uint8_t 					ref_count;						//设备被引用次数
    rt_uint8_t					device_id;						//设备ID, 0 -255
    
    /* 数据手法回调函数 */
    rt_err_t	(*rx_indicate)(rt_device_t dev, rt_size_t size);
    rt_err_t 	(*tx_complete)(rt_device_t dev, void *buffer);
    
    const struct rt_device_ops *ops;							//设备操作方法
    
    /* 设备的私有数据 */
    void *user_data;
}
```

I/O设备类型

```c
RT_Device_Class_Char	//字符设备
RT_Device_Class_Block	//块设备
RT_Device_Class_NetIf	//网络接口设备
RT_Device_Class_MTD		//内存设备
RT_Device_Class_RTC		//
RT_Device_Class_Sound	//
RT_Device_Class_Graphic	//图形设备
RT_Device_Class_I2CBUS	//
RT_Device_Class_USBDevice	//
RT_Device_Class_USBHost		//
RT_Device_Class_SPIBUS		//spi总线设备
RT_Device_Class_SPIDevice	//
RT_Device_Class_SDIO		//	
RT_Device_Class_Miscellaneous	//杂类设备
```

