# 功能实现
在lab1中，我实现了位于os/syscall/process.rs的sys_trace函数，首先用trace!宏打印日志，接着match输入的第一个参数trace_request
- trace_request为0时，使用read_volatile读取id地址处的低八位返回
- trace_request为1时, 使用write_volatile写入id地址处的低八位，用0xFF做data的掩码
- trace_request为2时，调用task::get_syscall_count_for_current(id)，返回当前任务调用编号为 id 的系统调用的次数
- 其他情况返回-1

我在任务控制块中新增了一个字段syscall_counts: [usize; MAX_SYSCALL_ID + 1]，其中MAX_SYSCALL_ID定为512，定义在config.rs中，TASK_MANAGER的初始化也做相应调整。syscall每次都会先调用task::inc_syscall_count_for_current，通过全局的TASK_MANAGER,为当前任务的TCB中的对应系统调用计数加1。
# 问答题

1. 正确进入 U 态后，程序的特征还应有：使用 S 态特权指令，访问 S 态寄存器后会报错。 请同学们可以自行测试这些内容（运行 三个 bad 测例 (ch2b_bad_*.rs) ）， 描述程序出错行为，同时注意注明你使用的 sbi 及其版本。
bad_address.rs尝试向0x0地址写入数据, 产生了异常，trap_handler处理并输出日志
[kernel] PageFault in application, bad addr = 0x0, bad instruction = 0x804003a4, kernel killed it.

bad_instruction.rs输出如下，用户态程序尝试使用sret命令
[kernel] IllegalInstruction in application, kernel killed it.

bad_register.rs输出如下，用户态程序尝试读取sstatus
[kernel] IllegalInstruction in application, kernel killed it.

使用的 sbi 及其版本：
[rustsbi] RustSBI version 0.3.0-alpha.2, adapting to RISC-V SBI v1.0.0
[rustsbi] Implementation     : RustSBI-QEMU Version 0.2.0-alpha.2

2. 深入理解 trap.S 中两个函数 __alltraps 和 __restore 的作用，并回答如下问题:

L40：刚进入 __restore 时，sp 代表了什么值。请指出 __restore 的两种使用情景。
sp代表内核栈栈顶指针，指向要恢复的TrapContext
__restore可以用于开始执行用户态程序，也可以用于trap处理完后返回用户态

L43-L48：这几行汇编代码特殊处理了哪些寄存器？这些寄存器的的值对于进入用户态有何意义？请分别解释。

ld t0, 32*8(sp)
ld t1, 33*8(sp)
ld t2, 2*8(sp)
csrw sstatus, t0
csrw sepc, t1
csrw sscratch, t2
通过t0, t1, t2寄存器，从TrapContext中恢复sstatus, sepc, sscratch寄存器
sstatus给出 Trap 发生之前CPU处在哪个特权级，是否允许S态中断，S态是否响应定时器中断等信息
sepc是发生之前执行的最后一条指令的地址，可以通过sret返回
sscratch是临时CSR，这里装入用户栈栈顶指针，进入内核态时sp和sscratch交换，sscratch装内核栈栈顶指针
3. L50-L56：为何跳过了 x2 和 x4？

ld x1, 1*8(sp)
ld x3, 3*8(sp)
.set n, 5
.rept 27
    LOAD_GP %n
    .set n, n+1
.endr

x2是sp，当前指向内核栈，sscratch里才是要保存的用户栈指针
x4是线程指针，不需要使用

4. L60：该指令之后，sp 和 sscratch 中的值分别有什么意义？

csrrw sp, sscratch, sp
交换sp和sscratch，此后sp指向用户栈，sscratch保存内核栈sp

5. __restore：中发生状态切换在哪一条指令？为何该指令执行之后会进入用户态？
sret指令返回用户态，跳到sepc，同时根据sstatus恢复状态

6. L13：该指令之后，sp 和 sscratch 中的值分别有什么意义？

csrrw sp, sscratch, sp
交换sp和sscratch，此后sp指向内核栈，sscratch保存用户栈sp

7. 从 U 态进入 S 态是哪一条指令发生的？
使用系统调用ecall或者发生中断/异常


# 荣誉准则

1. 在完成本次实验的过程（含此前学习的过程）中，我曾分别与 以下各位 就（与本次实验相关的）以下方面做过交流，还在代码中对应的位置以注释形式记录了具体的交流对象及内容：

    ChatGPT、Gemini等大模型

2. 此外，我也参考了 以下资料 ，还在代码中对应的位置以注释形式记录了具体的参考来源及内容：

    《rCore-Tutorial-Guide-2025S文档》

3. 我独立完成了本次实验除以上方面之外的所有工作，包括代码与文档。 我清楚地知道，从以上方面获得的信息在一定程度上降低了实验难度，可能会影响起评分。

4. 我从未使用过他人的代码，不管是原封不动地复制，还是经过了某些等价转换。 我未曾也不会向他人（含此后各届同学）复制或公开我的实验代码，我有义务妥善保管好它们。 我提交至本实验的评测系统的代码，均无意于破坏或妨碍任何计算机系统的正常运转。 我清楚地知道，以上情况均为本课程纪律所禁止，若违反，对应的实验成绩将按“-100”分计。