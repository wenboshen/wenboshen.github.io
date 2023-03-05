---
title: "Understanding Fork System Call"
collection: posts
permalink: /posts/2016-04-16-kernel-fork
date: 2016-04-16
description: ""
category: 
tags: []
---
* TOC
{:toc}

### System call fork  
Fork syscall is defined [`SYSCALL_DEFINE0(fork)`](https://elixir.bootlin.com/linux/v3.18/source/kernel/fork.c#L1703) in kernel/fork.c. The call chain is: [`do_fork`](https://elixir.bootlin.com/linux/v3.18/source/kernel/fork.c#L1623) --> [`copy_process`](https://elixir.bootlin.com/linux/v3.18/source/kernel/fork.c#L1182), which will call [`dup_task_struct`](https://elixir.bootlin.com/linux/v3.18/source/kernel/fork.c#L305), [`copy_mm`](https://elixir.bootlin.com/linux/v3.18/source/kernel/fork.c#L885), [`copy_thread`](https://elixir.bootlin.com/linux/v3.18/source/arch/arm64/kernel/process.c#L245) and so on. In dup_task_struct, it will allocate memory for new task struct and new thread_union, and then just set the stack points to the new thread_union, which means **the kernel stack never gets copied, the child process, regardless kernel thread or regular processes, just use a fresh empty kernel stack**.

```
static struct task_struct *dup_task_struct(struct task_struct *orig)
{
        struct task_struct *tsk;
        struct thread_info *ti;
        ......
        tsk = alloc_task_struct_node(node);
        ......
        ti = alloc_thread_info_node(tsk, node);
        ......
        tsk->stack = ti;
}
```

### Kernel thread vs user thread
From [cpu_context and pt_regs](https://wenboshen.org/posts/2015-12-18-kernel-stack.html#cpu_context-and-pt_regs), we know that after the process forking, the first time the new process gets context switch in, it starts from `task_struct.thread.cpu_context.pc`.

* For kernel thread, [`kernel_thread(fn, ...)`](https://elixir.bootlin.com/linux/v4.9.35/source/kernel/fork.c#L2000) --> [`_do_fork(xx, stack_start)`](https://elixir.bootlin.com/linux/v4.9.35/source/kernel/fork.c#L1911). The kernel thread function pointer is passed to kernel_thread as `fn`, then gets passed to _do_fork as `stack_start`, then at `copy_thread`, assign to `p->thread.cpu_context.pc`.

```
int copy_thread(......)
{
    ......
    if (likely(!(p->flags & PF_KTHREAD))) {
        ......
    } else { //KERNEL THREAD
        ......
        p->thread.cpu_context.x19 = stack_start; //kernel thread fn
        p->thread.cpu_context.x20 = stk_sz;
    }
    p->thread.cpu_context.pc = (unsigned long)ret_from_fork;
    p->thread.cpu_context.sp = (unsigned long)childregs;
    ......
}

ENTRY(ret_from_fork)
    bl schedule_tail
    cbz x19, 1f    // not a kernel thread
    mov x0, x20
    blr x19    //jump to kernel thread fn
1:  get_thread_info tsk
    b ret_to_user
ENDPROC(ret_from_fork)

```

After forking, when the new thread first time gets context switch in (get CPU), it starts from [`ret_from_fork`](https://elixir.bootlin.com/linux/v3.18/source/arch/arm64/kernel/entry.S#L631), which will check `x19`, and jump to it if it is not 0.

* For user processes, it is tricky, the child process needs to start run right after the fork() syscall in the parent's code, which means the child will return to same user space address with its parent. Remember that the user space address is pushed to pt_regs in [`kernel_entry`](https://elixir.bootlin.com/linux/v3.18/source/arch/arm64/kernel/entry.S#L66), so the child process only needs to copy pt_regs [src](https://elixir.bootlin.com/linux/v3.18/source/arch/arm64/kernel/process.c#L254), then in ret_from_fork, user space registers saved in pt_regs will be reload in [`kernel_exit`](https://elixir.bootlin.com/linux/v3.18/source/arch/arm64/kernel/entry.S#L116) and execution path goes to user space.

### How can fork() return two values
We all know that fork() can return different value to child and parent process? What's the corresponding kernel code?
For parent process, fork() is just a syscall, it will return value to user space. The code is at [the bottom of do_fork()](https://elixir.bootlin.com/linux/v3.18/source/kernel/fork.c#L1657).
```
if (!IS_ERR(p)) {
        ......
        pid = get_task_pid(p, PIDTYPE_PID);
        nr = pid_vnr(pid);
        ......
} else {
        nr = PTR_ERR(p);
}
return nr;
```

For child process, from ret_from_fork, it will return back to user space. As all the user space registers, including PC are saved in pt_regs, and the PC register is copied from parent, so the child will return to the same place as its parent does. The return register for the child is x0, and fork() set `pt_reg->x0` to 0, so 0 will be the return value for child process. The code is in [`copy_thread`](https://elixir.bootlin.com/linux/v3.18/source/arch/arm64/kernel/process.c#L255).
```
if (likely(!(p->flags & PF_KTHREAD))) {
        *childregs = *current_pt_regs();
        childregs->regs[0] = 0;
        ......
}
```
### fork vs vfork
fork and vfork are all implemented by do_fork, only the flags are different.  The only difference is that in copy_mm, if the CLONE_VM flag is set, the mm of the forked process will point to its parent's mm.
```
static int copy_mm(unsigned long clone_flags, struct task_struct *tsk)
{
        struct mm_struct *mm, *oldmm;
        int retval;
        ......
        tsk->mm = NULL;
        tsk->active_mm = NULL;
        ......
        if (clone_flags & CLONE_VM) {
                atomic_inc(&oldmm->mm_users);
                mm = oldmm;
                goto good_mm;
        }
 
        retval = -ENOMEM;
        mm = dup_mm(tsk);
        if (!mm)
                goto fail_nomem;
 
good_mm:
        tsk->mm = mm;
        tsk->active_mm = mm;
        return 0;
 
fail_nomem:
        return retval;
}
```
If CLONE_VM is set, [`dup_mm`](https://elixir.bootlin.com/linux/v3.18/source/kernel/fork.c#L848) will be skipped. dup_mm will call [`dup_mmap`](https://elixir.bootlin.com/linux/v3.18/source/kernel/fork.c#L367), which will copy all vma and the [`copy_page_range`](https://elixir.bootlin.com/linux/v3.18/source/mm/memory.c#L1006) in it will copy all the page table.
