---
title: 有限状态机FSM的理解与实现
date: 2017-06-23 18:24:36
categories: 技术
tags: 编程框架
---
有限状态机（finite state machine）简称FSM，表示有限个状态及在这些状态之间的转移和动作等行为的数学模型，在计算机领域有着广泛的应用。FSM是一种逻辑单元内部的一种高效编程方法，在服务器编程中，服务器可以根据不同状态或者消息类型进行相应的处理逻辑，使得程序逻辑清晰易懂。

**那有限状态机通常在什么地方被用到？**

处理程序语言或者自然语言的 tokenizer,自底向上解析语法的parser，
各种通信协议发送方和接受方传递数据对消息处理，游戏AI等都有应用场景。

状态机有以下几种实现方法，我将一一阐述它们的优缺点。

## 一、使用if/else if语句实现的FSM


使用if/else if语句是实现的FSM最简单最易懂的方法，我们只需要通过大量的if /else if语句来判断状态值来执行相应的逻辑处理。

看看下面的例子，我们使用了大量的if/else if语句实现了一个简单的状态机，做到了根据状态的不同执行相应的操作，并且实现了状态的跳转。
```

//比如我们定义了小明一天的状态如下
enum
{
	GET_UP,
	GO_TO_SCHOOL,
	HAVE_LUNCH,
	GO_HOME,
	DO_HOMEWORK,
	SLEEP,
};


int main()
{
	int state = GET_UP;
	//小明的一天
	while (1)
	{
		if (state == GET_UP)
		{
			GetUp(); //具体调用的函数
			state = GO_TO_SCHOOL;  //状态的转移
		}
		else if (state == GO_TO_SCHOOL)
		{
			Go2School();
			state = HAVE_LUNCH;
		}
		else if (state == HAVE_LUNCH)
		{
			HaveLunch();
		}
		...
		else if (state == SLEEP)
		{
			Go2Bed();
			state = GET_UP;
		}
	}

	return 0;
}

```

看完上面的例子，大家有什么感受？是不是感觉程序虽然简单易懂，但是使用了大量的if判断语句，使得代码很低端，同时代码膨胀的比较厉害。这个状态机的状态仅有几个，代码膨胀并不明显，但是如果我们需要处理的状态有数十个的话，该状态机的代码就不好读了。


## 二、使用switch实现FSM
使用switch语句实现的FSM的结构变得更为清晰了，其缺点也是明显的：这种设计方法虽然简单，通过一大堆判断来处理，适合小规模的状态切换流程，但如果规模扩大难以扩展和维护。

```
int main()
{
	int state = GET_UP;
	//小明的一天
	while (1)
	{

		switch(state)
		{
		case GET_UP:
			GetUp(); //具体调用的函数
			state = GO_TO_SCHOOL;  //状态的转移
			break;
		case GO_TO_SCHOOL:
			Go2School();
			state = HAVE_LUNCH;
			break;
		case HAVE_LUNCH:
			HaveLunch();
			state = GO_HOME;
			break;
			...
		default:
			break;
	    }
	}

	return 0;
}

```

## 三、使用函数指针实现FSM
使用函数指针实现FSM的思路：建立相应的状态表和动作查询表，根据状态表、事件、动作表定位相应的动作处理函数，执行完成后再进行状态的切换。

当然使用函数指针实现的FSM的过程还是比较费时费力，但是这一切都是值得的，因为当你的程序规模大时候，基于这种表结构的状态机，维护程序起来也是得心应手。

下面给出一个使用函数指针实现的FSM的框架：

我们还是以“小明的一天”为例设计出该FSM。

先给出该FSM的状态转移图：
![](1093303-20170623190453773-1051238237.png)



**下面讲解关键部分代码实现**

首先我们定义出小明一天的活动状态

```
//比如我们定义了小明一天的状态如下
enum
{
	GET_UP,
	GO_TO_SCHOOL,
	HAVE_LUNCH,
	DO_HOMEWORK,
	SLEEP,
};
```

我们也定义出会发生的事件

```
enum
{
	EVENT1 = 1,
	EVENT2,
	EVENT3,
};
```


定义状态表的数据结构

```
typedef struct FsmTable_s
{
	int event;   //事件
	int CurState;  //当前状态
	void (*eventActFun)();  //函数指针
	int NextState;  //下一个状态
}FsmTable_t;
```

接下来定义出最重要FSM的状态表，我们整个FSM就是根据这个定义好的表来运转的。

```
FsmTable_t XiaoMingTable[] =
{
	//{到来的事件，当前的状态，将要要执行的函数，下一个状态}
	{ EVENT2,  SLEEP,           GetUp,        GET_UP },
	{ EVENT1,  GET_UP,          Go2School,    GO_TO_SCHOOL },
	{ EVENT2,  GO_TO_SCHOOL,    HaveLunch,    HAVE_LUNCH },
	{ EVENT3,  HAVE_LUNCH,      DoHomework,   DO_HOMEWORK },
	{ EVENT1,  DO_HOMEWORK,     Go2Bed,       SLEEP },

	//add your codes here
};
```

状态机的注册、状态转移、事件处理的动作实现

```
/*状态机注册*/
void FSM_Regist(FSM_t* pFsm, FsmTable_t* pTable)
{
	pFsm->FsmTable = pTable;
}

/*状态迁移*/
void FSM_StateTransfer(FSM_t* pFsm, int state)
{
	pFsm->curState = state;
}

/*事件处理*/
void FSM_EventHandle(FSM_t* pFsm, int event)
{
	FsmTable_t* pActTable = pFsm->FsmTable;
	void (*eventActFun)() = NULL;  //函数指针初始化为空
	int NextState;
	int CurState = pFsm->curState;
	int flag = 0; //标识是否满足条件
	int i;

	/*获取当前动作函数*/
	for (i = 0; i<g_max_num; i++)
	{
		//当且仅当当前状态下来个指定的事件，我才执行它
		if (event == pActTable[i].event && CurState == pActTable[i].CurState)
		{
			flag = 1;
			eventActFun = pActTable[i].eventActFun;
			NextState = pActTable[i].NextState;
			break;
		}
	}


	if (flag) //如果满足条件了
	{
		/*动作执行*/
		if (eventActFun)
		{
			eventActFun();
		}

		//跳转到下一个状态
		FSM_StateTransfer(pFsm, NextState);
	}
	else
	{
		// do nothing
	}
}
```

主函数我们这样写，然后观察状态机的运转情况

```
int main()
{
	FSM_t fsm;
	InitFsm(&fsm);
	int event = EVENT1; 
	//小明的一天,周而复始的一天又一天，进行着相同的活动
	while (1)
	{
		printf("event %d is coming...\n", event);
		FSM_EventHandle(&fsm, event);
		printf("fsm current state %d\n", fsm.curState);
		test(&event); 
		sleep(1);  //休眠1秒，方便观察
	}

	return 0;
}
```

看一看该状态机跑起来的状态转移情况：

![](1093303-20170623190514710-334307320.png)




上面的图可以看出，当且仅当在指定的状态下来了指定的事件才会发生函数的执行以及状态的转移，否则不会发生状态的跳转。这种机制使得这个状态机不停地自动运转，有条不絮地完成任务。

与前两种方法相比，使用函数指针实现FSM能很好用于大规模的切换流程，只要我们实现搭好了FSM框架，以后进行扩展就很简单了（只要在状态表里加一行来写入新的状态处理就可以了）。


需要FSM完整代码的童鞋请访问[我的github](https://github.com/AstarLight/FSM-framework/tree/master)