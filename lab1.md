# nachos lab1 threads
## Exercise 1  源代码阅读
- code/threads/main.cc  
初始化、自检并启动nachos kernel, 处理参数,启动相应的功能

- code/threads/threadtest.cc  
nachos4.1中无此文件，而在code/threads/thread.cc中定义SelfTest()和ThreadTest().SelfTest()通过自身和fork出的线程调用SimpleTest()，交替yield主动放弃CPU，测试thread switching
- code/threads/thread.h and code/threads/thread.cc  
声明并定义管理Thread的数据结构，字段部分有线程持有的寄存器、线程状态、线程名、栈顶指针、栈底地址等，方法部分主要有:  
    - Fork(VoidFunctionPtr func, void \*arg) 让线程运行(\*func)(arg)函数  

    - ChechOverflow() 检查线程栈是否溢出  
    - Finsh() 当fork出的线程运行结束后调用，做清理工作  
    - Yield() 当有其他处于等待状态的线程时，让出CPU  
    - Sleep(bool finishing) 将当前线程状态设为blocked，让出CPU. 参数finishing不做处理，直接传给Scheduler::Run
## Exercise 2  扩展线程的数据结构 
在thread.h中添加userId和threadId字段，并添加相应的get方法  
```
privite: 
    int userId;
    int threadId;
public:
    int getUserId() { return userId; }
    int getThreadId() { return threadId; }
```
在thread.cc的构造函数和析构函数中添加维护语句
```
in Thread::Thread
userId = getuid(); // from POSIX API
threadId = kernel->AllocateThreadId();
if (threadId == -1) {
    ASSERT(false);
}

in Thread::~Thread
kernel->DeallocateThreadId(threadId);  
```
kernel中分配和销毁threadId的函数为
```
int Kernel::AllocateThreadId()                                                  
{
    threadNum++;
    for (int i = 0; i < maxThreadNum; i++) {
        if (!threadPool[i]) {
            threadPool[i] = true;
            return i;
        }
    }
    return -1;
}

void Kernel::DeallocateThreadId(int tid)
{
    threadNum--;
    threadPool[tid] = false;
}
```
threadNum和threadPool为实现全局线程管理的变量，见Exercise 3
## Exercise 3  增加全局线程管理机制 
在kernel中设置变量maxThreadNum=128. 变量threadNum记录当前线程数量，数组threadPool为bool[maxThreadNum]，false表示当前id尚未分配，true为已分配. 为测试能否限制线程数在maxThreadNum以内，编写Thread::MaxThreadNumTest()，见下
```
void Thread::MaxThreadNumTest()
{
    DEBUG(dbgThread, "Entering Thread::MaxThreadNumTest");
    Thread** t = new Thread*[130];
    for (int i = 0; i < 130; i++) {
        t[i] = new Thread("forked thread");
    }
    //kernel->TS();
}
```
测试结果见下
```
➜  build.linux git:(master) ✗ ./nachos -K2 -d t
Entering main


tests summary: ok:0
Creating new thread with id:0. Curent threadNum: 1
Creating new thread with id:1. Curent threadNum: 2
Forking thread: postal worker f(a): 134603500 0x9517940
Putting thread on ready list: postal worker
Entering Thread::SelfTest
Entering Thread::MaxThreadNumTest
Creating new thread with id:2. Curent threadNum: 3
Creating new thread with id:3. Curent threadNum: 4
Creating new thread with id:4. Curent threadNum: 5
...
...
Creating new thread with id:125. Curent threadNum: 126
Creating new thread with id:126. Curent threadNum: 127
Creating new thread with id:127. Curent threadNum: 128
No unused thread id left. ASSERT
```
在kernel.cc中增加打印线程状态的函数TS()
```
void Kernel::TS()
{
    for(ListIterator<Thread*> *iter = new ListIterator<Thread*>(kernel->threadList); !iter->IsDone(); iter->Next())
    {
        cout << "threadId : " << iter->Item()->getThreadId() 
            << ", threadName : " << iter->Item()->getName()
            << ", userId : " << iter->Item()->getUserId()
            << ", priority : " << iter->Item()->getPriority();
        switch(iter->Item()->getStatus()) {
            case 0: cout << ", status : JUST_CREATED" << endl; break;
            case 1: cout << ", status : READY" << endl; break;
            case 2: cout << ", status : RUNNING" << endl; break;
            case 3: cout << ", status : BLOCKED" << endl; break;
            default: ASSERT(false);
        }
    }
}
```
新建一个线程并调用TS()，结果见下
```
➜  build.linux git:(master) ✗ ./nachos -K1


tests summary: ok:0
threadId : 0, threadName : main, userId : 1000, priority : 128, status : READY
threadId : 1, threadName : postal worker, userId : 1000, priority : 128, status : RUNNING
threadId : 2, threadName : forked thread, userId : 1000, priority : 128, status : JUST_CREATED
Machine halting!
```
## Exercise 4  源代码阅读 
- code/threads/scheduler.h and code/threads/scheduler.cc  
声明并定义管理Scheduler的数据结构，默认算法为FIFO，字段部分有就绪队列readyList，主要方法有:  
    - ReadyToRun(Thread\* thread) 将线程状态设为ready，并放入readyList  

    - FindNextToRun(bool sleep) 根据当前算法从readyList中取出下一个上CPU的线程  
    - Run(Thread\* nextThread, bool finishing) 通过调用SWITCH保存老线程的状态，加载新线程的状态并将它调上CPU，如果finishing为true，即老线程已运行结束，那么会将toBeDestroyed指针指向它  
    - CheckToBeDestroyed() 销毁toBeDestroyed指向的线程  
  
- code/threads/switch.s  
与机器架构对应的汇编码写成的SWITCH(oldThread, newThread)方法，保存oldThread的CPU寄存器状态，加载newThread的CPU寄存器状态    
- code/machine/timer.h and code/machine/timer.cc  
通过调度中断模拟硬件时钟，主要方法有:  
    - CallBack() 唤醒nachos interrupt handler，调用SetInterrupt()  

    - SetInterrupt() 根据调用nachos时的参数，随机或以TimerTicks为延迟生成时钟中断，并插入中断队列  
## Exercise 5  线程调度算法扩展 
实现基于优先级的抢占式线程调度.  

在thread.h与thread.cc中添加优先级priority与相应的set、get函数，priority的范围为[0,255]，prioruty值越小优先级越大. 在Thread的构造函数增加优先级参数，默认为128. 将scheduler的readyList的类型改为SortedList，并添加对线程优先级进行比较的函数:
```
static int ThreadPriorityCompare(Thread* t1, Thread* t2) { return t1->getPriority() - t2->getPriority(); }
```
以此为SortedList的构造参数，这样readyList的队首永远是优先级最高的线程.  

修改Scheduler::FindNextToRun(bool). 原方法采用FIFO，并且会删除队首元素. 现增加参数bool sleep，当该函数被要休眠的线程调用时，参数为true，直接返回readyList队首元素. 当队首元素优先级大于当前线程时，返回对手元素，否则返回NULL，这就是没有在该函数内删除队首元素的原因，在调用该函数处会根据该函数返回值是否为NULL决定是否删除队首元素. 更改后内容见下:
```
Thread* Scheduler::FindNextToRun(bool sleep)
{
    ASSERT(kernel->interrupt->getLevel() == IntOff);
    // preemptive priority scheduling
    if (sleep) { // current thread is about to sleep, return next thread 
        return readyList->Front();
    }
    if (readyList->IsEmpty()) {
        return NULL;
    } 
    Thread* nextThread = readyList->Front();
    if (nextThread->getPriority() < kernel->currentThread->getPriority()) {
        return nextThread;
    }
    return NULL;
} 
```
编写测试函数Thread::PreemptivePrioritySchedulingTest()，见下
```
void Thread::PreemptivePrioritySchedulingTest()
{   
    Thread *t1 = new Thread("forked thread", 127);
    Thread *t2 = new Thread("forked thread", 126);
    Thread *t3 = new Thread("forked thread", 125);

    t1->Fork((VoidFunctionPtr) SimpleThread, (void *) t1->getThreadId());
    t2->Fork((VoidFunctionPtr) SimpleThread, (void *) t2->getThreadId());
    t3->Fork((VoidFunctionPtr) SimpleThread, (void *) t3->getThreadId());
    kernel->TS();
    kernel->currentThread->Yield();
    kernel->TS();
}
```
在SelfTest()中调用测试函数，结果见下
```
➜  build.linux git:(master) ✗ ./nachos -K3


tests summary: ok:0
threadId : 0, threadName : main, userId : 1000, priority : 128, status : READY
threadId : 1, threadName : postal worker, userId : 1000, priority : 128, status : RUNNING
threadId : 2, threadName : forked thread, userId : 1000, priority : 127, status : RUNNING
threadId : 3, threadName : forked thread, userId : 1000, priority : 126, status : RUNNING
threadId : 4, threadName : forked thread, userId : 1000, priority : 125, status : RUNNING
*** thread 4 looped 0 times
*** thread 4 looped 1 times
*** thread 4 looped 2 times
*** thread 4 looped 3 times
*** thread 4 looped 4 times
*** thread 3 looped 0 times
*** thread 3 looped 1 times
*** thread 3 looped 2 times
*** thread 3 looped 3 times
*** thread 3 looped 4 times
*** thread 2 looped 0 times
*** thread 2 looped 1 times
*** thread 2 looped 2 times
*** thread 2 looped 3 times
*** thread 2 looped 4 times
threadId : 0, threadName : main, userId : 1000, priority : 128, status : READY
threadId : 1, threadName : postal worker, userId : 1000, priority : 128, status : BLOCKED
Machine halting!
```
优先级更高的线程优先运行，与期望符合
## \*Challenge  线程调度算法扩展
实现时间片轮转算法. (未完成）

## \*To be completed
- 时间片轮转算法未完成

- 在执行一些关于线程的功能时还会报segment fault，没有全部改完  

- 代码逻辑很乱，需要整理
