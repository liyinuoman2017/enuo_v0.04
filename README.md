# enuo_v0.04
从零开始构建嵌入式实时操作系统5——设计延时功能


![在这里插入图片描述](https://img-blog.csdnimg.cn/1926919d64af40b387b5adb6ef93a8c7.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAbGl5aW51bzIwMTc=,size_20,color_FFFFFF,t_70,g_se,x_16)



## 1.前言

> 人生只有三天，昨天、今天和明天。昨天已然成为过去，明天尚在未来，拥有的不过是今天。
> 每一个今天，终将成为昨天，每一个明天，也都会成为今天，如此往复，抓住现在，珍惜未来，才能过好这一生。

这段箴言道出了时间的宝贵，我们需要珍惜时间，高效的利用时间。对于人如此，对于软件设计也同样如此，优良的软件设计往往能高效利用处理器的执行时间，最大程度的减少低效率的操作。

![在这里插入图片描述](https://img-blog.csdnimg.cn/b718fe8d3a82477d8a7a346fb3dfc84f.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAbGl5aW51bzIwMTc=,size_20,color_FFFFFF,t_70,g_se,x_16)


很久以前做过一个测量温湿度的项目，当时使用的额是AHT21这款芯片，该芯片再触发测量之后，需要等待80ms，AHT21的datasheet如下：

![在这里插入图片描述](https://img-blog.csdnimg.cn/d2c89ce9d5034beb881128269f43ea29.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAbGl5aW51bzIwMTc=,size_19,color_FFFFFF,t_70,g_se,x_16)

当时设计的代码大致如下：

![在这里插入图片描述](https://img-blog.csdnimg.cn/75eb1f33bb174b85bce601df7425bd09.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAbGl5aW51bzIwMTc=,size_19,color_FFFFFF,t_70,g_se,x_16)

程序每次读AHT21温度数据时都需要延时**等待100ms**，假设程序设计的采样频率是5HZ，就是一秒钟读5次AHT21温度数据。那么MCU在执行过程中**每一秒就有500ms在**“**死等**”，MCU的有效使用率直接下降为**50%**！由此可见设计高效的延时对提高系统的执行效率有多么重要。

![在这里插入图片描述](https://img-blog.csdnimg.cn/e25398ba2c3d42e7b192c90256d74199.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAbGl5aW51bzIwMTc=,size_19,color_FFFFFF,t_70,g_se,x_16)


## 2.设计背景

在enuo v0.03版本中增加了延时表，这个延时表并非真正的延时表，它本质上是一个停止表，需要停止运行的任务就存放到该表中，该表中的任务将一直处于停止状态，直到有其它任务恢复此表中的任务。
这种设计只是实现了任务两个状态切换（就绪态和停止态），工程如下：


![在这里插入图片描述](https://img-blog.csdnimg.cn/be44d3fab8724f6c91648c155e18bfe7.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAbGl5aW51bzIwMTc=,size_20,color_FFFFFF,t_70,g_se,x_16)

## 3.设计目标

enuo v0.04的设计目标是增加一个阻塞延时功能，当任务需要延时等待一定时间时，任务调用延时函数，此时操作系统会将该任务从就绪表中移除，同时该任务会插入延时列表，延时表中的任务在延时时间没有到期之前不会被执行。当延时完成时任务自动恢复到就绪状态，之后操作系统会调度执行该任务。为完成设计目的需要完成以下功能：
**1、增加一个计时功能，使用一个32位的全局变量作为计时器，在系统节拍中断函数中完成自加。
2、每次系统节拍函数进入中断后，完成延时表中的任务延时检测，检测任务延时是否到期。
3、完成任务延时表功能，任务根据延时时间长短插入延时表，延时短的任务在延时列表的前端。**

![在这里插入图片描述](https://img-blog.csdnimg.cn/517f36de37e24da39ae4307e2fea8499.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAbGl5aW51bzIwMTc=,size_20,color_FFFFFF,t_70,g_se,x_16)



## 4.设计环境

硬件环境是使用STM32F401RE为核心的自制开发板，软件环境是使用的KEIL V5.2 开发工具。

![在这里插入图片描述](https://img-blog.csdnimg.cn/5baeab54b38e4476a61a192ba3154e07.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAbGl5aW51bzIwMTc=,size_20,color_FFFFFF,t_70,g_se,x_16)

## 5.设计过程

## 5.1阻塞式延时

相信大家去吃一些美味小食时都排队的经历吧，这种漫长的等待经历是不是还历历在目？除了在队伍里傻站着和刷刷手机，啥也干不了。

![在这里插入图片描述](https://img-blog.csdnimg.cn/bc4ff36a646a40fd994ca92b9a2dbc53.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAbGl5aW51bzIwMTc=,size_20,color_FFFFFF,t_70,g_se,x_16)

但是有些商场就有一种很好的等待机制：无线提示器。客户点餐后就可以在一定范围内自由活动，当你点的餐做好了，无线提示器会提示你去取餐。这种方式可以极大的减轻客户站立排队的低效和痛苦。无线提示器如下图：

![在这里插入图片描述](https://img-blog.csdnimg.cn/b5b7efbbe0974b9eb2272d5bd9a9c901.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAbGl5aW51bzIwMTc=,size_20,color_FFFFFF,t_70,g_se,x_16)

  上面展示了两种等待模式：
**1、阻塞式等待
2、非阻塞式等待**

阻塞式等待和非阻塞式等待关注的是任务在执行等待时的状态，阻塞式等待是指任务在等待期间会被挂起（处于停止状态），等待结束之后任务将会继续运行，**记忆方法：当前任务被阻塞（不运行）**。非阻塞式等待任务在等待期间不会被挂起，**记忆方法：当前任务被不阻塞（一直运行）**。

 回到上面排队的例子，第一种站立式排队模式，当你处在这种等待模式下时你必须一直保存这种排队等待的状态，直到你买到你需要的商品，**这种模式就是非阻塞式等待**。
 第二种无线提示器模式，当客户完成商品付款后，就可以在一定范围内自由活动，直到收到无线提示器会提示就可以直接去取商品，**这种模式就是阻塞式等待**。非常明显在大多数情况下**阻塞式等待会更加高效**。



## 5.2延时策略

  

  实现等待很多种策略，常见的策略有以下两种：

**1、倒计时法
2、闹钟法**

大家一定都看过火箭发射的场面，那清晰洪亮的倒计时声仿佛就在耳边回响：

**10...9...8...7...6...5...4...3...2...1...0**

火箭发射的就是使用的倒计时法，这种策略只需要关注剩余时间。

大部分人早上起床的时候肯定是被设定的闹钟叫醒，这种策略使用起来十分方便，只用关注设定的时间即可。

**倒计时法的算法逻辑是**：每一次判断剩余时间是否为0，如果剩余时间不为0，就将剩余时间减一，如果剩余时间为0则完成延时等待。

**闹钟法的算法逻辑是**：比较当前时间是否等于设定时间，不等于设定时间则继续等待，等于设定时间则完成延时等待。


![在这里插入图片描述](https://img-blog.csdnimg.cn/b1003f48efff4a0cb758e11eb93b5462.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAbGl5aW51bzIwMTc=,size_20,color_FFFFFF,t_70,g_se,x_16)

## 5.3两种策略对比

      
**倒计时法需要完成一次时间判断操作和一次时间减法操作，闹钟法只需要完成一次时间判断操作。**

假设现在有100个任务，使用倒计时法需要完成**100次时间判断操作和100次时间减法操作**。而闹钟法只要完成**100次时间判断操作**。

如果我们将延时任务按照顺序排列，**延时短的任务放在延时队列的前端**，只就意味着最快完成延时的任务肯定是延时队列的第一个任务，因此我们**只用判断延时队列的第一个任务是否完成延时操作即可**。
 
假设现在有100个任务，使用倒计时法**需要完成100次时间减法操作和1次判断延时队列首个任务操作**。而闹钟法**只要完成1次判断延时队列首个任务操作**。 由于闹钟法执行步骤较少，enuo使用闹钟法机制。


闹钟法机制虽然执行效率高，但是同时存在一个时间归零问题，我们知道当前时间是23点，如果需要等待2小时，闹钟时间就变成了1点，**因此闹钟法需要额外处理时间归零问题**。


![在这里插入图片描述](https://img-blog.csdnimg.cn/794e397e30704f4caa864f41977bb330.png)

## 5.4系统节拍


操作系统需要一个周期性的时钟源，这个时钟源称之为时钟节拍。为了生成时周期性钟节拍，通常需要使用硬件定时器，配置硬件定时器产生一个固定频率的中断（通常为10~1000Hz之间），当中断发生时，中断服务程序调用操作系统中的一个特殊程序：系统时钟节拍服务。

![请添加图片描述](https://img-blog.csdnimg.cn/03c81c5747484476b1141be2a293ab1b.gif)


系统时钟节拍可以配置为10Hz,也可以配置为1000Hz。时钟节拍值越大说明硬件定时器产生的中断越频繁，系统时钟节拍服务就越频繁的执行。

高频率时钟节拍的优点：

**1、提高操作系统时间管理精度
2、提高任务抢占准确度**

例如10Hz的时钟节拍，意味着的操作系统执行时间粒度为100ms，系统中的周期性事件最快为100ms一次，无法有更高的精度。例如1000Hz的时钟节拍，此时的执行时间粒度就提高了100倍，此时系统中的周期性事件最快为1ms一次，时间精度可以达到1ms。



enuo系统定义一个全局变量**heartbeat ，记录了系统节拍时间**。系统中断服务程序中执行heartbeat 自加工作，更新系统节拍时间。代码如下：

```c
volatile uint32_t  heartbeat ;
/*********************************************************************************************************
* @名称	: SysTick_Handler
* @描述	: 系统中断服务程序
**********************************************************************************************************/
void SysTick_Handler(void)
{	
	/* 系统节拍心跳计数  */	
	heartbeat++;
	/* 延时处理  */	
	delay_handle();
	/* 开始任务切换调度 */
	scheduler_task();
}
```


## 5.5加入延时队列

需要延时等待的任务执行延时函数delay，延时函数会将指定的任务从原有的链表中移除，并将任务插入延时列表，延时列表中的任务将不会被执行。
任务延时的结束时间是当前的系统节拍时间加上延时时间，延时列表中的任务按照结束时间从小到大进行排列，延时列表的第一个任务就是最快被唤醒的任务。
延时函数中末尾有一次主动任务调度请求，此时操作系统将立即执行下一个就绪的任务。代码如下：

```c
/*********************************************************************************************************
* @名称	: 延时
* @描述	: 延时单位为一个系统心跳节拍
**********************************************************************************************************/
void delay( uint32_t delay ,task_tcb_t *task)
{
	/* 保存滑动指针位置 */
	list_node_t * const new_node = &task->link;
	task->link.sort_value = heartbeat + delay;
	/* 移除原有链表关系 */
	list_remove(new_node);
	/* 插入滑动指针末尾 */	
	list_sort_insert( &delay_list , new_node);
	/* 调度任务 */
	scheduler_task();
}
```

![在这里插入图片描述](https://img-blog.csdnimg.cn/c5a10e36a7734eaaa70421902bed8953.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAbGl5aW51bzIwMTc=,size_20,color_FFFFFF,t_70,g_se,x_16)

系统开始运行时task0，task1，task2都处于就绪状态，当task1执行延时操作后，系统将task1从就绪列表中移除，同时将task1插入延时列表中，此时系统任务的总体状态如下：

![在这里插入图片描述](https://img-blog.csdnimg.cn/3734239aad4142fa9f9bf5b09ae25f0d.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAbGl5aW51bzIwMTc=,size_20,color_FFFFFF,t_70,g_se,x_16)


## 5.6延时处理

系统中断服务程序中执行heartbeat自加工作，更新系统节拍时间后执行延时处理函数delay_handle，进入延时处理函数后先判断延时列表中的任务数量；如果延时列表中的任务数量不为0，就会判断延时列表第一个任务是否完成延时；如果延时列表第一个任务延时时间到期，就会将任务从延时列表中移除，同时将该任务加入就绪列表末尾。代码如下：

```c
/*********************************************************************************************************
* @名称	: delay_handle
* @描述	: 延时处理  	
**********************************************************************************************************/
void delay_handle(void)
{
	/* 判断延时列表是否有任务  */	
	if( delay_list.node_value != 0 )
	{
		/* 判断延时列表第一个任务 是否完成延时 */
		if(delay_list.head.next->sort_value <= heartbeat)
		{
			list_node_t * delay_node;
			delay_node = delay_list.head.next;
			/* 移除原有链表关系 */
			list_remove(delay_node);
			/* 插入滑动指针末尾 */	
			list_insert_sliding_pointer_end( &ready_list , delay_node);			
		}
	}
}
/*********************************************************************************************************
* @名称	: SysTick_Handler
* @描述	: 系统中断服务程序
**********************************************************************************************************/
void SysTick_Handler(void)
{	
	/* 系统节拍心跳计数  */	
	heartbeat++;
	/* 延时处理  */	
	delay_handle();
	/* 开始任务切换调度 */
	scheduler_task();
}
```
假设task1，task2在延时列表中，此时系统任务的状态如下：

![在这里插入图片描述](https://img-blog.csdnimg.cn/241d802487564213a32c45d3284053a6.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAbGl5aW51bzIwMTc=,size_20,color_FFFFFF,t_70,g_se,x_16)

当系统节拍时间更新后，task1延时时间到期，系统将task1从延时列表中移除，将task1插入就绪列表中，此时系统任务的总体状态如下：

![在这里插入图片描述](https://img-blog.csdnimg.cn/793ce78f0e494042b0d145e5bf9ae285.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAbGl5aW51bzIwMTc=,size_20,color_FFFFFF,t_70,g_se,x_16)



## 6.运行结果

测试任务代码如下：

```c
/*********************************************************************************************************
* @名称	:task0 
* @描述	:任务0
**********************************************************************************************************/
void task0(void)
{
	while(1)
	{
		/* 测试跟踪 */
		task_debug_num0++;	
		test_function();
		/* 延时 */
		delay(5,&my_task0);			
	}
}           
/*********************************************************************************************************
* @名称	: task1
* @描述	: 任务1
**********************************************************************************************************/  
void task1(void)
{	
	while(1)
	{
		task_debug_num1++;	/* 测试跟踪 */
		test_function();
		/* 延时 */		
		delay(50,&my_task1);
	}
}
/*********************************************************************************************************
* @名称	: task2
* @描述	: 任务2
**********************************************************************************************************/ 
void task2(void)
{	
	while(1)
	{
		task_debug_num2++;	/* 测试跟踪 */
		test_function();
		/* 延时 */
		delay(100,&my_task2);
	}
}
```

工程中有3个任务task0，task1，task2，其中task0每执行一次就延时5个节拍，task1每执行一次就延时50个节拍，task2每执行一次就延时100个节拍。

因此运行一段时间后task0，task1，task2各种的**task_debug_num之比约为20：2：1**

仿真结果如下：

![在这里插入图片描述](https://img-blog.csdnimg.cn/57cec6deda9a4b2e8f8a38af4c9bbd07.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAbGl5aW51bzIwMTc=,size_15,color_FFFFFF,t_70,g_se,x_16)


**task_debug_num之比约为20：2：1 ，实现延时功能的设计目标。**

**总结：本文讲解了延时的不同模式和策略，描述了enuo延时功能的设计过程**


**希望获取源码的朋友们在评论区里留言。**

**未完待续…**
	
**实时操作系统系列将持续更新**
	
**创作不易希望朋友们点赞，转发，评论，关注。**
	
**您的点赞，转发，评论，关注将是我持续更新的动力**
	
**作者：李巍**
	
**Github：liyinuoman2017**
	
**CSDN：liyinuo2017**
	
**今日头条：程序猿李巍**

  
![在这里插入图片描述](https://img-blog.csdnimg.cn/7df2c5f7d3e04918b3361cb0147b8ab9.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAbGl5aW51bzIwMTc=,size_20,color_FFFFFF,t_70,g_se,x_16)
