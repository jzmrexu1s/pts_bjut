# 实验一 THREAD 通过alarm clock tests

## 重新实现timer_sleep函数
调用timer_sleep的时候直接把线程阻塞掉，然后给线程结构体加一个成员ticks_blocked来记录这个线程被sleep了多少时间， 然后利用操作系统自身的时钟中断（每个tick会执行一次）加入对线程状态的检测， 每次检测将ticks_blocked减1, 如果减到0就唤醒这个线程。

## 代码理解
### timer.c中的timer_sleep实现:
>##### void timer_sleep (int64_t ticks) 
{
  int64_t start = timer_ticks ();  //timer_ticks 返回自操作系统启动以来的计时器滴答次数
  ASSERT (intr_get_level () == INTR_ON);  //intr_get_level 返回当前中断状态
  while (timer_elapsed (start) < ticks) 
  thread_yield ();
}
>>##### timer_ticks代码：
int64_t timer_ticks (void) 
{
  enum intr_level old_level = intr_disable ();  
  //intr_level代表能否被中断
  //intr_disable:
  1. 调用intr_get_level() 
  2. 直接执行汇编代码，调用汇编指令来保证这个线程不能被中断。
    int64_t t = ticks;
    intr_set_level (old_level); 
    //intr_set_level 按级别启用或禁用中断，并返回上一个中断状态,如果之前是允许中断的（INTR_ON）则enable否则就disable
    barrier ();
    return t;
}

>>##### intr_get_level 代码：
intr_get_level (void) 
{
uint32_t flags;
asm volatile ("pushfl; popl %0" : "=g" (flags));  
//调用了汇编指令，把标志寄存器的东西放到处理器棧上，然后把值pop到flags（代表标志寄存器IF位）上，通过判断flags来返回当前终端状态(intr_level)
return flags & FLAG_IF ? INTR_ON : INTR_OFF;
}

>>##### intr_set_level 代码：
intr_set_level (enum intr_level level) 
{
	return level == INTR_ON ? intr_enable () : intr_disable ();
}

>>>###### intr_enable
{
enum intr_level old_level = intr_get_level ();
ASSERT (!intr_context ());  //intr_context函数返回结果的false
asm volatile ("sti");  //汇编指令：sti指令将IF位置为1
return old_level;
}

>>>>###### intr_context:  
//是否外中断的标志in_external_intr，ASSERT断言这个中断不是外中断（IO等， 硬中断）而是操作系统正常线程切换流程里的内中断（软中断）
intr_context (void) 
{
	return in_external_intr;
}

#### pintos操作系统如何利用中断机制来确保一个原子性的操作的
##### 1. intr_get_level返回了intr_level的值
##### 2. intr_disable获取了当前的中断状态， 然后将当前中断状态改为不能被中断， 然后返回执行之前的中断状态。
##### 3.timer_ticks 获取ticks的当前值返回, 禁止当前行为被中断, 保存禁止被中断前的中断状态（用old_level储存）

>##### ticks:
static int64_t ticks; //操作系统启动后的计时器计时次数

>while (timer_elapsed (start) < ticks)
thread_yield();
##### //start获取了起始时间，ASSERT必须可以被中断,不然会一直死循环下去

>>timer_elapsed (int64_t then)
{
return timer_ticks () - then;
//返回了当前时间距离then的时间间隔,循环实质ticks的时间内不断执行thread_yield
}

>###### void thread_yield (void) 
{
  struct thread *cur = thread_current ();   //thread_current 返回当前线程起始指针位置
>enum intr_level old_level;  
  ASSERT (!intr_context ());      
  old_level = intr_disable ();  //软中断
  if (cur != idle_thread)   //如何当前线程不是空闲的线程就调用list_push_back把当前线程的元素扔到就绪队列里面， 并把线程改成THREAD_READY状态
    list_push_back (&ready_list, &cur->elem);     //线程机制保证的一个原子性操作
  cur->status = THREAD_READY;
  schedule ();
  intr_set_level (old_level);
}

###### schedule先把当前线程丢到就绪队列，如果下一个线程和当前线程不一样的话切换线程
>###### schedule (void) 
{
######   //获取当前线程cur和调用next_thread_to_run获取下一个要run的线程
  struct thread *cur = running_thread ();
  struct thread *next = next_thread_to_run ();
  struct thread *prev = NULL;
> ###### //确保不能被中断， 当前线程是RUNNING_THREAD
  ASSERT (intr_get_level () == INTR_OFF);
  ASSERT (cur->status != THREAD_RUNNING);
  ASSERT (is_thread (next));
> ###### //如果当前线程和下一个要跑的线程不是同一个的话调用switch_threads返回给prev
  if (cur != next)
    prev = switch_threads (cur, next);
  schedule_tail (prev); //参数prev是NULL或者在下一个线程的上下文中的当前线程指针
  //schedule_tail 通过激活新线程的页表来完成线程切换，如果前一个线程正在死亡，则销毁它
}
>>struct thread *switch_threads (struct thread *cur, struct thread *next); 
// threads/switch.S中
>
static tid_t
allocate_tid (void) 
{
  static tid_t next_tid = 1;
  tid_t tid;
>
  lock_acquire (&tid_lock);
  tid = next_tid++;
  lock_release (&tid_lock);
>
  return tid;
}
##### timer_sleep就是在ticks时间内,如果线程处于running状态就不断把他扔到就绪队列不让他执行


## 实现
#### 给线程的结构体加 sleep_ticks:
>  int64_t sleep_ticks;
t->sleep_ticks = 0;  //线程被创建的时候初始化为0

#### 修改时钟中断处理函数， 加入线程sleep时间的检测
>thread_foreach(check_blocked_thread, NULL);
>
//thread_foreach对每个线程都执行check_blocked_thread函数
//给thread.h添加一个方法blocked_thread_check
>>void check_blocked_thread(struct thread *t, void *aux)
{
  if(t->status==THREAD_BLOCKED && t->sleep_ticks > 0)
  {
    t->sleep_ticks-=1;
    if(t->sleep_ticks==0)
    {
      thread_unblock(t); //thread_unblock 线程丢到就绪队列里继续跑
    }
  }
}