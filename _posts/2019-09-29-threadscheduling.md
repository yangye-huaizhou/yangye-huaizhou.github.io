---
layout: post
title: 用户定义的linux进程调度
date:   2019-09-29 16:56:39
categories: operation-system
---

进程调度是现代操作系统一个重要的组成部分，理论上它会为进程提供多种不同的运行状态，以及在CPU核上、核间调度的策略。因为项目实践需要，我们需要**在一个CPU核上用自己的调度器来运行多个进程，运行策略由用户态程序决定，在特定的时候唤醒特定的进程**。这次就来分享一下进程调度的一些基本概念和我们的这种纯用户空间进程调度的实现。

# 1.进程状态

在linux操作系统，用```top```命令我们就能看到有许许多多正在运行的进程：
![top命令输出.png](/assets/picture/schedule1.png)
*这些属性中与进程调度有关的有NI、S，他们分别对应着进程的优先级和运行状态。*

**在linux系统中，进程的运行状态主要分为5种：**

* **Running/Runnable**：Running进程为当前正在使用CPU的进程，Runnable进程是具备运行条件且仅在等待CPU的进程。进程结构体state字段为*TASK_RUNNING*。

* **Sleeping**：Sleeping进程是等待资源(例如：I/O操作完成)或事件(例如：定时器，经过一定时间)的进程。在linux系统中Sleeping进程又分为两种：可中断睡眠状态(S)与不可中断睡眠状态(D)。它们之间的区别在于，前者可以用信号(signal)来唤醒，而后者则不能。假设一个进程在唤醒之前正在等待I/O操作完成。如果在此期间它收到终止信号(SIGKILL)，它将会在处理I/O请求返回的数据前被杀死(唤醒即杀死)。这就是为什么I/O操作通常在等待结果时进入不可中断睡眠的原因：操作准备就绪时，它们会唤醒，处理结果，然后检查是否有任何待处理的信号。而那些可以在满足唤醒条件且可以在没有任何后果之前终止的进程通常使用可中断睡眠。另外睡眠状态在真正使用中还有诸多限制，需要结合实际情况非常小心，比如不能带锁睡眠等。可中断睡眠状态的进程state字段为*TASK_INTERRUPTIBLE*，不可中断睡眠为*TASK_UNINTERRUPTIBLE*。

* **Stopped(T)**：当进程收到SIGSTOP信号时就停止了(例如，在终端输入```ctrl+z```时)。当停止时，进程执行将被挂起，并且它将处理的唯一信号是SIGKILL和SIGCONT。前者杀死该进程，而后者将使该进程返回“Running/Runnable”状态。其进程state字段为*TASK_STOPPED*。

* **Zombie(Z)**：当进程通过```exit()```系统调用结束时，其状态需要由其父进程“获得”(调用```wait()```)；同时，子进程仍处于僵尸状态(没有生命也没有死亡)。其进程state字段为*TASK_ZOMBIE*。

进程在这些状态间来回切换的图示如下：

![进程状态转换.png](/assets/picture/schedule2.png)

进程的这些运行状态是为了让众多进程在有限的CPU核上跑起来而提出的。在现代多核处理器上，同一时间CPU只能被一个进程使用，也就意味着如何，实际上操作系统做了一些调度策略，比方说每个进程运行一段时间进入sleep状态给其他进程使用CPU的机会。只要这个周期够短，就能让用户感觉不到自己是在“运行-睡眠-运行-睡眠”的，这又被称为*“时间多路复用”*。同时为了保证同一个进程不会一直占据某个CPU，linux默认也是会将进程缓慢地在核间调度的。

# 2.调度

在linux操作系统中调度策略有三种——**SCHED_OTHER**、**SCHED_FIFO**、**SCHED_RR**，按照传统分法，它们分成两大类：
* **普通进程调度**

普通进程是用于区分实时进程的概念，在linux系统中，默认的进程都是普通进程，采用**SCHED_OTHER**调度。这是一种CFS（Completely Fair Schedule，完全公平调度算法）调度算法，它为每个进程分配一次运行的CPU时间片（slice），时间片运行完即让出CPU。

在时间片的确定上，不直接使用优先级，将优先级换算成基本时间片：
>当进程静态优先级<120时，基本时间片=(140-静态优先级)×20；
当进程静态优先级>=120时，基本时间片=(140-静态优先级)×5

另外动态优先级是用来计算睡眠时间的，就不展开讲了。
对于用户而言，可以通过```nice```系统调用来调节进程运行的时间片间隔。

* **实时进程调度**：

linux系统中的实时进程调度有两种：**SCHED_FIFO**和**SCHED_RR**。只要人为设置进程使用这两种调度策略的进程都是实时进程，每个实时进程都有一个优先级，范围从1(最高)到99(最低)。调度程序会让优先级高的进程运行，而禁止优先级低的进程运行。这是区别于普通进程调度的，实时进程总是被当成活动进程。而当有几个优先级相同的进程需要运行时，调度程序会选择本地CPU运行队列链表中的第一个进程来运行。

对于两种调度策略区别在于：
使用**SCHED_RR**策略的进程是基于时间片轮转来调度，进程的时间片用完，系统将重新分配时间片，并置于就绪队列尾。
使用**SCHED_FIFO**的进程一旦占用cpu则一直运行。一直运行直到有更高优先级任务到达或自己放弃。一些系统调用如```sched_yield()```可以主动让出CPU。

使用实时进程调度时，该进程是不可被抢占的；而一个核上有一个普通进程正在运行，现加入一个实时进程，那么该实时进程将会抢占普通进程。

从用户角度，既可以在代码中指定当前进程的调度策略和优先级，也可以在运行的过程中用chrt命令来改变进程的调度策略和优先级：
```
chrt -p $PID         # 可以查看 pid=$PID 的进程的 调度策略, 输出如下:
      pid $PID's current scheduling policy: SCHED_OTHER
      pid $PID's current scheduling priority: 0

chrt -p -f 10 $PID   # 修改进程$PID的调度策略为 SCHED_FIFO, 并且优先级为10
chrt -p $PID         # 再次查看调度策略
      pid $PID's current scheduling policy: SCHED_FIFO
      pid $PID's current scheduling priority: 10
```

# 3.用户空间实现唤醒式调度

重新回到前面，我们的需求：***1）多个worker进程在一个核上工作；2）有一个单独的进程做中央调度进程，不定期去唤醒对应的worker进程；3）每个worker进程执行完自己的一轮任务后主动放弃CPU进入sleeping状态，能且仅能被调度进程唤醒；4）worker进程处于运行状态时不可被别的进程打断抢占。***

注意到这里worker进程的调度是**不可抢占的，非时间片触发的**，那么只有**SCHED_FIFO**一种调度适合。
>*使用FIFO调度需要特别注意，因为它不能被抢占，如果一旦由于bug进入了不可中断睡眠状态，那么这个进程几乎是杀不死的，只能重启；而且它在一个核上运行的时候会霸占整个核，最好的方式是在grub中使用核隔离*

进入睡眠状态的方式有直接调用```sleep```、调用```schedule()```、等待I/O资源等，但由于我们的进程均处于用户态，无法直接调用内核未暴露接口的函数，且sleep和等待I/O这些操作无法做到精确控制。那么只能写一个内核模块，调度程序用ioctl与之通信，把对应的worker进程从sleeping状态置为running状态。但是在精确度上还是有所欠缺。

这时候想到了利用多线程的思路，worker进程可以创建一个pthread来做这个工作，自己主线程等这个真正工作的pthread退出才退出。这样可以使用多线程的条件变量来实现控制从线程sleeping还是running。

每个worker进程逻辑如下：
```
#主线程:收到调度进程的信号，就调用一次cond_signal解除从线程阻塞
      pthread_mutex_lock(&lock);
      pthread_cond_signal(&needProduct);
      pthread_mutex_unlock(&lock);

#从线程:work函数中没运行一轮workload，调用cond_wait阻塞进入sleeping状态
void *work(void *arg)
{
    while(1)
    {
       ... #workload
       pthread_mutex_lock(&lock);
       pthread_cond_wait(&needProduct,NULL);
       pthread_mutex_unlock(&lock);
    }
}
```

至于调度进程与worker进程如何通信，可以使用信号、套接字、管道、消息队列、共享内存等一系列进程间通信的方式。

下面附上一个demo，这个例子使用的信号来做进程间通信，使用了```SIGUSR1```信号，直接使用终端命令```kill -s 10 $PID```也可以实现调度器功能，每发一个信号执行一次loop：

worker.c创建一个从线程，运行work函数内容

```
//worker.c
#include<stdio.h>
#include<unistd.h>
#define __USE_GNU 
#include<sched.h>
#include<pthread.h>
#include <sys/prctl.h>
#include <sys/syscall.h>
#include <signal.h>
static pthread_mutex_t lock=PTHREAD_MUTEX_INITIALIZER;
static pthread_cond_t needProduct=PTHREAD_COND_INITIALIZER;

static void
signal_handler(int signum)
{
        pthread_mutex_lock(&lock);
        pthread_cond_signal(&needProduct);//解除条件变量的阻塞
        pthread_mutex_unlock(&lock);
}

void *work(void *arg)
{
        signal(SIGUSR1, signal_handler); //注册信号
        prctl(PR_SET_NAME,"slave");
        int a=0;
        int i=0;
        pthread_setcancelstate(PTHREAD_CANCEL_ENABLE, NULL);
        pthread_setcanceltype(PTHREAD_CANCEL_DEFERRED, NULL);
        pthread_detach(pthread_self());
        while(1)
        {
                a+=1;
                a=a%9;
                printf("%d:",syscall(SYS_gettid));
                for(i=0;i<a;i++)
                {
                        printf("*");
                }
                printf("\n");
                pthread_testcancel();
                pthread_mutex_lock(&lock);
                pthread_cond_wait(&needProduct,&lock);
                pthread_mutex_unlock(&lock);
                //放弃CPU自己被阻塞
        }
        return NULL;
}

int main()
{
    signal(SIGUSR1, signal_handler);
    pthread_attr_t attr;
    pthread_attr_init(&attr);
    pthread_attr_setschedpolicy(&attr,SCHED_FIFO);
    struct sched_param param;
    param.sched_priority=30;
    pthread_attr_setschedparam(&attr,&param);
    pthread_attr_setinheritsched(&attr,PTHREAD_EXPLICIT_SCHED);
    cpu_set_t mask;
    CPU_ZERO(&mask);
    CPU_SET(6,&mask);
    if(pthread_attr_setaffinity_np(&attr,sizeof(mask),&mask)==-1)
    {
        printf("pthread_attr_setaffinity_np erro\n");
    }
    //设置调度模式和亲和性等，绑定从线程到6号核上，设置调度为SCHED_FIFO，优先级为30
    pthread_t t;
    int error = pthread_create(&t, &attr, work, NULL);
    if(error!=0)
    {
        printf("can't create thread\n");
    }

    pthread_attr_destroy(&attr);
    pthread_join(t,NULL);
    return 0;
}
```
编译```gcc worker.c -g -Wall -lpthread -o worker```，运行```./worker```。

schedule.c非常简单，直接调用kill函数向worker进程发送```SIGUSR1```信号。
```
//schedule.c
#include<stdio.h>
#include <stdlib.h>
#define __USE_GNU
#include<sched.h>
#include<pthread.h>
#include<signal.h>
int main(int argc,char** argv)
{
    int pid=atoi(argv[1]);
    //唤醒对应的进程
    kill(pid, SIGUSR1);
    return 0;
}
```

编译```gcc schedule.c -o schedule```，运行```./schedule $PID```($PID为worker进程的进程号)。

效果是每运行一次schedule就会看到输出来一层*号。

![运行效果.png](/assets/picture/schedule3.png)


*引用：*

[1] Linux Process States and Signals, https://medium.com/@cloudchef/linux-process-states-and-signals-a967d18fab64

[2] 《深入理解LINUX内核》