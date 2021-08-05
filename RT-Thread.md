#### **RT-Thread** 

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
void rt_object_init(struct rt_object* object, enum rt_object_class_type type, const char* name)	
//初始化静态对象，object--对象控制块的指针
void rt_object_detach(rt_object_t object);		//脱离对象，从内核对象管理器链表上删除相应的对象节点，内存未被释放
rt_object_t rt_object_allocate(enum rt_object_class_type type, const char* name) //分配动态对象，先获取对象信息，再分配内存空间，然后进行初始化，最后将其插入所在的对象容器链表中。
void rt_object_delete(rt_object_t object)		//删除动态对象，对象容器链表中脱离动态对象，并释放占用的内存
rt_err_t rt_object_is_systemobject(rt_object_t object)	//辨别对象，判断一个对象是否为系统对象（静态）
```

​		内核配置：主要通过修改工程目录下的rtconfig.h文件进行，打开/关闭该文件中的宏定义来对代码进行条件编译，达到系统配置和裁剪的目的。