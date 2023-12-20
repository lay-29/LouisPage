# C# 多线程

同步

同步单线程：按顺序执行



任何的异步多线程都离不开委托



异步对线程：发起调用，不等待结束就直接进入下一行

```
Action<string> action =this.DosomethingLong

//单线程
action.Invoke("btnAsync_Click_1");
action("btnAsync_Click_2");

//异步多线程：发起调用不等结束就直接进入下一行，动作会由一个新线程来执行（子线程），并发了
action.BeginInvoke("btnAsync_Click_3",null,null);

```



同步单线程卡界面——主线程（UI线程）忙于计算

异步单线程不卡界面——创建了子线程去计算，主线程已经闲置



同步单线程方法慢——因为只有一个线程干活

异步多线程方法快——有多个线程干活

多线程就是用资源换性能  

 1个线程13000  5个线程4269  性能只有3倍提升

a 多线程的协调管理额外成本——项目经理

b资源也有有上限——5辆车只有3条道

线程不是越多越好



无序性——不可预测

启动无序：几乎同一时间向操作系统请求线程，也是个需求Cpu处理请求

​					线程是操作系统资源，CLR只能去申请，具体是什么顺序，无法掌控

​	执行时间不确定：同一线程同一任务耗时也可能不同，其实跟操作系统调度策略有关

​									Cpu分片（计算能力太强）

​									线程优先级可以影响操作系统的调度

结束无序：



​	正是因为多线程不可预测性，很多时候你的想法并不一定能够贯彻实施，也许大多数情况ok，但是总有一定概率出现问题。

##  BeginInvoke

### 委托异步回调控制顺序   BeginInvoke

```c#
Action<string> action =this.UpdataDb; //创建委托

//该线程完成时调用callback函数
action.BeginInvoke("btnAsyncAdvanced_Click",callback,"sunday");
		（参数一：输入string，参数二：AsyncCallback对象,参数三：是object 可以是string 也可以是对象）
            

AsyncCallback callback =ar=>
{
    Console.WriteLine(ar.AsyncState);//通过该方法调用到传入的“sundy”
    Console.WriteLine($"btnAsyncAdvanced_Click完成{Thread,CurrentThread.ManagedThreadId}")
}
```



### 异步上传进度展示   IAsyncResult

需求：用户必须操作完成，才能返回——上传文件，只有成功之后才能预览

```c#
//创建IAsyncResult对象来接受线程执行返回结果
IAsyncResult asyncResult=ation.BeginInvoke("文件上传",null,null);

int i=0;
while (!asyncResult.IsCompleted)
{	
    if(i<9 )
    {
       lable1.Text="当前文件上传进度为{++i*10}%"; 
    }
   else
   {
       lable1.Text="当前文件上传进度为99.99%";
   }
  	Thread.Sleep(200);
}
Console.WriteLine("完成文件上传，执行预览，绑定到界面")；


```

重点：创建IAsyncResult对象来接受线程返回值，线程结束时将IAsyncResult对象的IsCompleted赋值为true



### 信号量   asynResult.AsyncWaitHandle.WaitOne();



```
//创建线程调用接口，将结果返回给asynResult
var asynResult =action.BeginInvoke("调用接口"，null,null);

Console.WriteLine("Do Something Else.....");
Console.WriteLine("Do Something Else.....");
Console.WriteLine("Do Something Else.....");
Console.WriteLine("Do Something Else.....");

//阻塞当前线程，等待子线程结束继续运行
asynResult.AsyncWaitHandle.WaitOne();
Console.WriteLine("接口调用成功")
```

```c#
//一直等待
asynResult.AsyncWaitHandle.WaitOne(-1);
//最多等待1000ms  用来做超时控制的
asynResult.AsyncWaitHandle.WaitOne(1000);
```



### EndInvoke完成等待和回去返回值



```c#
//模拟调用远程接口
private int RemoteService()
{
	long lResult=0;
	for (int i=0;i<1000000;i++)
	{
	lResult +=i;
	}
	return DateTime.Now.Day;
}


Func<int> func=RemoteService;
int iResult2=func.Invoke();
//异步
IAsyncResult asyncResult =func.BeginInvoke(null,null);//异步调用结果，描述异步操作的
//如要获取异步调用的真实结果,只能EndInvoke
int iRsult3=func.EndInvoke(asyncResult);//也可以等待
```

在回调里去使用EndInvoke

```c#
IAsyncResult asyncResult =func.BeginInvoke(ar=>{
	int iResult4=func.EndInovke(ar);
},null)
```



## Thread

### Thread操作线程的各种API和Thread的缺陷

```c#
//******1.0版本的写法
//ThreadStart 类是一个委托
ThreadStart threadStart=()=>
{
	Console.WriteLine("DO something```")
};

Thread t1=new Thread(threadStart);
t1.Start();
t1.Suspend();//挂起
t1.Resume();//挂起回复
t1.Join();
t1.IsBackGround();
t1.Abort();//销毁
t1.ResetAbort();//销毁恢复
//Thread线程资源是操作系统管理的响应不灵敏
//启动线程是没有控制的，可能导致死机

```



### Thread Pool线程池的优势与缺点

池化资源管理设计思想，线程是一种资源，之前每次要用线程，就要申请一个线程，使用完后，需释放掉

线程池化就是做一个容器，容器提前申请5个线程，程序需要使用线程，直接找容器获取，用完后再放回容器（控制状态），避免频繁的申请和销毁，容器还会根据限制的数量去申请和释放

1.线程复用 2.可以限制最大线程数量

1.Api太少了，线程等待顺序控制特别弱，MRE,影响实战

```c#
//WaitCallBack类是一个委托
WaitCallBack callback=ar=>
{
	Console.WriteLine("DO something```")
};

ThreadPool.QueueUserWorkItem(callback);
```



### 多线程Parallel的特点和Parallel控制线程

parallel 可以启动多线程，主线程参与计算，节约一个线程

可以通过Invoke第一个参数ParallelOptions轻松控制最大并发数量 

```c#
Parallel.Invoke(
	()=>
    {
        Console.WriteLine("DO something1```");
    },
    ()=>
    {
        Console.WriteLine("DO something2```");
    },
    ()=>
    {
        Console.WriteLine("DO something3```");
    },
    ()=>
    {
        Console.WriteLine("DO something4```");
    }
);
```



## Task

### 多线程Task （ 多线程的最佳实践）

全部是线程池线程     提供了丰富的API

```c#
 Action action = () =>
 { 
 	Console.WriteLine("Do Something1")
 }
Task task=new Task(action);
task.Start();
```



```c#
Console.WriteLine("工程开始")
    
List<Task> taskList=new List<Task>();
Task.Run(()=>Coding("王"))；
Task.Run(()=>Coding("孙"))；
Task.Run(()=>Coding("李"))；
Task.Run(()=>Coding("赵"))；

Console.WriteLine("工程结束")
    
 //方法
Coding(string name)
{
    Console.WriteLine("{name}开始干活")；
    Console.WriteLine("{name}结束干活")；
}
```

以上写法不行，不等干活工程就结束了，主线程应在输出工程结束时阻塞等待子线程结束

```c#
Console.WriteLine("工程开始")
    
List<Task> taskList=new List<Task>();
taskList.Add(Task.Run(()=>Coding("王")))；
taskList.Add(Task.Run(()=>Coding("孙")))；
taskList.Add(Task.Run(()=>Coding("李")))；
taskList.Add(Task.Run(()=>Coding("赵")))；

    
//等待任一任务完成后，启动一个新的task来完成后续动作
taskFactory.ContinueWhenAny(taskList.ToArry(),t=>
                            {
                                Console.WriteLine("第一个完成了")；
                            })；

//等待全部任务完成后，启动一个新的task来完成后续动作
//像这样 将所有人完成输出庆功宴 加入taskList 确保“庆功宴”输出在“工程结束”前
taskList.Add(taskFactory.ContinueWhenAll(taskList.ToArry(),tArry=>
                            {
                                Console.WriteLine("庆功宴")；
                            }))；
//后续的线程，可能是新线程，可能是刚完成任务的线程，还可能是同一个线程，不可能是主线程

    
//以下输出符合，但是会造成主线程阻塞（主界面卡住）
//阻塞当前线程，直到某一任务结束
Task.WaitAny(taskList.ToArray());
Console.WriteLine("第一个人完成")
//阻塞当前线程，直到全部任务结束
Task.WaitAll(taskList.toArry());
TaskFactory taskFactory=new TaskFactory();

Console.WriteLine("工程结束")

    
 //方法
Coding(string name)
{
    Console.WriteLine("{name}开始干活")；
    Console.WriteLine("{name}结束干活")；
}
```



### 多线程安全

以下例子i在输出时都是5 与预期不符

可以在循环里加入k  输出0 1 2 3 4

```c#
for(int i=0;i<=5;i++)
{
	int k=i;
	Task.Run(()=>{
		Console.WriteLine($"This is{i}  {k} Start...");
		Thread.Sleep(2000);
		Console.WriteLine($"This is{i}  {k} Start...");
		});
}
```

多线程去访问一个集合，一般是没有问题的

线程安全问题都出在修改一个对象

```c#
List<int> intList=new List<int>();
for(int i=0;i<10000;i++)
{
    Tsjk.Run(()=>
             {
                intList.Add(i); 
             });
}
Thread.Slepp(5000);
Console.WriteLine(intList.Count);
```

以上输出结果小于10000，且每次运行情况不同 9992，9996

以上代码，单线程执行与多线程执行结果不一致，就表明有线程安全问题

以上代码现象的原因，List是一个数组结构，在内存上是连续摆放的，假如同一时刻，去增加一个数据，都是操作同一个内存位置，2个cpu同时发出了命令，内存先执行一个再执行一个，就会出现覆盖

以上代码线程安全解决方法（加锁）

```c#

List<int> intList=new List<int>();
for(int i=0;i<10000;i++)
{
    Tsjk.Run(()=>
             {
                 lock(LOCK)
             	{
                 intList.Add(i); 
            	 }
                
             });
}
Thread.Slepp(5000);
Console.WriteLine(intList.Count);
```

### 线程锁Lock

加lock就能解决线程安全问题，就是单线程化，Lock就是保障方法块儿任意时刻只有一个线程能进去，其他线程排队

lock原理，等价于Monitor，锁定一个内存引用地址，所以不能是值类型，也不能是null，因为Lock就是占据引用，需要一个引用



使用同一个线程锁会使两组线程同时锁住（先执行完线程组1，再执行线程组2），如果想两组线程并发进行就要加不同的锁

```c#
private static readonly object LOCK=new object();
private static readonly object LOCKTest=new object(); 

for(int i=0;i<5;i++)
{
    int k=i;
    Task.Run(()=>
             {
                lock(LOCK)
                {
                Console.WriteLine($"This is {i}  {k}Mainshow Start...");
                Thread.Sleep(2000);
                Console.WriteLine($"This is {i}  {k}Mainshow End...");
                } 
             });
}

for(int i=0;i<5;i++)
{
    int k=i;
    Task.Run(()=>
             {
                lock(LOCKTest)//如果使用LOCK加锁两组线程不会并发
                {
                Console.WriteLine($"This is {i}  {k}Mainshow Start...");
                Thread.Sleep(2000);
                Console.WriteLine($"This is {i}  {k}Mainshow End...");
                } 
             });
}
```



调用类里带有线程锁

```c#
class Testlock
{
	private  readonly object LOCK=new object();
	public void showtemp(int index)
    {
       for(int i=0;i<5;i++)
		{
    		int k=i;
    		Task.Run(()=>
         	{
             	lock(LOCK)
            	 {
             		Console.WriteLine($"This is {i}  {k}Mainshow Start...");
               		 Thread.Sleep(2000);
                	Console.WriteLine($"This is {i}  {k}Mainshow End...");
             	} 
         	});
		}
    }
}

//实例化
Testlock testlock1=new Testlock();
Testlock testlock2=new Testlock();
testlock1.showtemp(1);
testlock2.showtemp(2);
//两组线程可并发，因为实例化后LOCK已不同，但是如果使用static修饰LOCK则不能并发
```



以下情况不能并发，锁是应用，字符串共享

```c#
private static readonly object LOCK="静待花开";
private static readonly object LOCKTest="静待花开"; 

for(int i=0;i<5;i++)
{
    int k=i;
    Task.Run(()=>
             {
                lock(LOCK)
                {
                Console.WriteLine($"This is {i}  {k}Mainshow Start...");
                Thread.Sleep(2000);
                Console.WriteLine($"This is {i}  {k}Mainshow End...");
                } 
             });
}

for(int i=0;i<5;i++)
{
    int k=i;
    Task.Run(()=>
             {
                lock(LOCKTest)
                {
                Console.WriteLine($"This is {i}  {k}Mainshow Start...");
                Thread.Sleep(2000);
                Console.WriteLine($"This is {i}  {k}Mainshow End...");
                } 
             });
}
```



泛型类在类型参数相同是才是同一个类，类型参数不同是是不同的类

```c#
class TestlockGeneric<T> 
{
	private static readonly object LOCK=new object();
	public static void Show(int index)
    {
       for(int i=0;i<5;i++)
		{
    		int k=i;
    		Task.Run(()=>
         	{
             	lock(LOCK)
            	 {
             		Console.WriteLine($"This is {i}  {k}Mainshow Start...");
               		 Thread.Sleep(2000);
                	Console.WriteLine($"This is {i}  {k}Mainshow End...");
             	} 
         	});
		}
    }
}

TestlockGeneric<int>.Show(1);
TestlockGeneric<int>.Show(2);
TestlockGeneric<string>.Show(3);
//1与2 不能并发    1，2与3 可以并发
```



this也可以当锁来使用  （不用创建直接用）

对象也可以当锁 但是与对象内部方法调用this当锁的线程不能并发

死锁？



### await,async

执行和演练

```c#
public class AwaitAsyncClassNew
{
    public async Task DoSomething()
    {
        await Task.Run(()=>{
            Console.WriteLine("******");
        });
    }
}
```

async 可以随便添加，可以不用await

await 只能出现在task前面，但是方法必须声明 async，不能单独出现

await/async 之后，原本没有返回值的，可以返回 Task

​								原本返回值 X 类型，可以返回Task<x>

### 无返回值

```c#
public async void NoReturn()
    
{
    Console.Wreite($"This NOReturn Start{Thread.CurrnetThread.ManagedId}")
        Task task =Task.Run(()=>{
            Console.WriteLine($"This no Reuturn Task Start{Thread.CurrnetThread.ManagedId}");
            Thread.Sleep(2000)；
            Console.WriteLine($"This no Return Task End{Thread.CurrentThread.ManagedId}");
        })
   await task;
    Console.WriteLine($"This is NoReturn End{Thread.CurrnetThread.ManagedId}")
}
```



```c#
public async Task NoReturn()
    
{
    Console.Wreite($"This NOReturn Start{Thread.CurrnetThread.ManagedId}")
    Task task =Task.Run(()=>{
            Console.WriteLine($"This no Reuturn Task Start{Thread.CurrnetThread.ManagedId}");
            Thread.Sleep(2000)；
            Console.WriteLine($"This no Return Task End{Thread.CurrentThread.ManagedId}");
        })
   await task;
    Console.WriteLine($"This is NoReturn End{Thread.CurrnetThread.ManagedId}")
}
```

### 有返回值

```c#




//返回long的方法
public long ReturnLong()
{
   Console.WriteLine("Return start{Thread.CurrentThread,ManagedThreadId}");
    long result =0;
    Task task = Task.Run(()=>{
       Console.WriteLine($"NoReturn Task Start{Thread.CurrentThread.MangedThreadId}");
        for(int i=0;i<10000;i++)
        {
             result += i;                   
        }
        Console.WriteLine($"NoReturn Task Start{Thread.CurrentThread.ManagrdThreadId}");
        return result;
    });
}

```

























