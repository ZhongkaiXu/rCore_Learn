# 引言

- 多道程序os：内存里存在多道程序，按给定的顺序去执行、
- 多道程序协作os：支持app自己放弃CPU转去运行其他程序
- 分时多任务os：时间片到了就抢占

# 多道程序的加载和放置

和批处理不一样，是一次把全部程序都加载进内存，但是不能动态移动，必须保持位置不变。

- link_user.S：嵌入app的elf文件。
- 改造批处理的appmanager
  - loader：负责加载
  - task：负责执行和转换
- 链接脚本：build.py，构建每个app之前把linker.ld起始地址改了，然后用cargo build --bin构建单个文件，app以某间隔排列在内存上。记得还原linker.ld的内容。
- loader：把多个文件加载到不同位置，类似上面的build.py

**（这里的build.py应该是直接在裸机上运行的时候用的脚本，loader是在内核基础上运行时候用的加载的脚本）**

# 任务切换

**从多道程序os到多道程序协作式os**

app的一次执行过程->一个任务->一个片段：任务片：空闲/计算。

任务切换和trap切换的不同：
任务切换是在内核中的trap控制流的交换，实际上就是一个在内核中的函数调用，trap进内核后__switch，运行另一个trap控制流。

* [X] 从方法上来看，trap切换是直接跳转到alltrap，而switch作为一个函数来使用，所以编译器帮助完成保存调用者reg。

既然是函数调用就必须保存ra，作为返回地址，由于换栈，还要保存sp和一些寄存器。

e.g.A->B

- switch之前，A的内核栈有A的trapcontext和trap处理的调用栈信息
- 切换前把任务上下文（ra、sp、s0-s11）存到全局变量taskmanager里面。
- 接下来根据B的任务上下文恢复寄存器（主要是ra可以实现跳转，sp实现了换栈），通过换栈实现了控制流的切换.
- 保存了s0-s11，这是被调用者(__switch)需要做的，其他的调用作保存和临时reg，编译器自己做
- ret以后CPU会跳转到当前ra执行。

# 多道程序和协作式调度


任务：

- 任务运行状态：未初始化、准备、正在、已经退出
- 控制块：管理上下文，控制执行和暂停
- 系统调用：暂停yield，退出exit

**批处理的缺点 - CPU忙等**

CPU在app进行IO的时候要不断test某个寄存器看IO结束了没。

所以希望app在进行IO的时候主动暂停，系统调用sys_yield让出CPU。

**任务控制块TCB**

- TaskControlBlock
  - task_status: TaskStatus
  - task_cx: TaskContext
    - ra,sp,s0-s11(被调用者保存)

**全局任务管理器**

- num_app: usize
- Inner(内部可变性，分离变量)
  - tasks: [TCB;num_app]
  - current_task

**sys_yield/sys_exit**

1. 修改当前task的状态
2. 找下一个ready的task
3. __switch

**第一次进入用户态**

```
// os/src/task/mod.rs

for i in 0..num_app {
    tasks[i].task_cx = TaskContext::goto_restore(init_app_cx(i));
    tasks[i].task_status = TaskStatus::Ready;
}

// os/src/task/context.rs

impl TaskContext {
    pub fn goto_restore(kstack_ptr: usize) -> Self {
        extern "C" { fn __restore(); }
        Self {
            ra: __restore as usize,
            sp: kstack_ptr,
            s: [0; 12],
        }
    }
}

// os/src/loader.rs

pub fn init_app_cx(app_id: usize) -> usize {
    KERNEL_STACK[app_id].push_context(
        TrapContext::app_init_context(get_base_i(app_id), USER_STACK[app_id].get_sp()),
    )
}
```

1. init trapcontext:
   - x0 - 31: 保存了usr_sp
   - sstatus: User
   - spec: app_addr (_restore结束sret后CPU跳转的地址)
2. init_cx压栈，返回kernel_sp
3. goto_restore: 构建task_cx,ra->__restore,sp->kernel_sp,s0-s11
4. run_first_app:取task0，状态设为running，调用switch，传入两个相同ptr，CPU->task0_cx.ra->restore->批处理的初始化。

# 抢占式

时间片轮转算法：维护一个任务队列，队头执行一个时间片，放到队尾，重复，直到所有app完成

**riscv的中断**

依靠硬件的时钟中断。中断也是一种trap，而异常是能够追溯到某条指令的执行，中断是异步于正在执行的指令，来自外设。

> riscv：软件中断、时钟中断、外部中断（外设）

中断的特权级决定了会不会被屏蔽：和当前CPU的特权级比较，<不会处理，>=通过CSR判断是否被屏蔽。S级中断相关的CSR：sstatus + sie。

默认在M级处理中断，但是trap到的特权级不能低于中断的特权级，比如U可以trap到S，但是S到U只能用sret。

U.app -> S.kernel    S.interrupt -> S.kernel

默认情况下不会发生嵌套中断，中断处理的时候，同级中断都被屏蔽。

**时钟中断和计时器**

mtime CSR：上电以来经过了多少个时钟周期

mtimecmp CSR： mtime一超过mtimecmp就会触发时钟中断

以上两个M级的CSR，需要通过sbi接口访问

- get_time: get mtime
- set_timer: set mtimecmp
- set_next_trigger: get mtime,set mtimecmp为mtime+10ms，这样10ms后就触发时钟中断
- get time us: 上电以来运行了多久，us为单位

# 抢占式调度

trap_handler增加分支：

当发生时钟中断，设置10ms的计时器，暂停当前任务切换到下一个任务。

**全过程总结**

内核初始化后，加载app的elf镜像到合适的地址，创建trapcontext和taskcontext，设置stie=1允许S级时钟中断，而sstatus的sie为0暂时没事，因为U->S必然触发，app运行10ms后时钟中断，trap到handler中，再次设置一个10ms的时钟中断，然后暂停当前应用切换下一个，10ms后再中断，再设置10ms，再切换...
