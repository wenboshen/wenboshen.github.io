---
layout: post
title: "Understanding Linux Execve System Call"
date: 2016-09-15
description: ""
category: 
tags: []
---
* TOC
{:toc}

do_execve is called in three places, one for [`syscall execve`](https://elixir.bootlin.com/linux/v4.9.35/source/fs/exec.c#L1910), one for init process in [`run_init_process`](https://elixir.bootlin.com/linux/v4.9.35/source/init/main.c#L892) and another one in [`user mode helper`](https://elixir.bootlin.com/linux/v4.9.35/source/kernel/kmod.c#L252). 



### System call execve
Syscall execve usually is used after syscall fork() to execute a different binary for child process, the typical scenario is
```c
int main(int argc, char * argv[]){
    char * ls_args[4] = { "ls", "-l", NULL} ;
    pid_t child_pid;
    int status;
    if (child_pid == 0){
        child_pid = fork();
        printf("In child\n");
    }else if (child_pid > 0){
        execvp( ls_args[0], ls_args);
        wait(&status);
    }
    printf("In parent: finished\n");
    return 0; //return success
}
```

In C library, execl, execlp, execle, execv, execvp, execvpe are all supported by the [`syscall execve`](https://elixir.bootlin.com/linux/v4.9.35/source/fs/exec.c#L1910), the call chain is `SyS_execve` -> `do_execve` -> `do_execveat_common`. The main body of [`do_execveat_common`](https://elixir.bootlin.com/linux/v4.9.35/source/fs/exec.c#L1677) is:
```
retval = bprm_mm_init(bprm);
retval = prepare_binprm(bprm);
retval = copy_strings_kernel(1, &bprm->filename, bprm);
retval = copy_strings(bprm->envc, envp, bprm);
retval = exec_binprm(bprm);
retval = copy_strings(bprm->argc, argv, bprm);
```

`exec_binprm` will invoke `search_binary_handler(bprm)` to find the corresponding handler for each binary format. `search_binary_handler` will search a global list called [`formats`](https://elixir.bootlin.com/linux/v4.9.35/source/fs/exec.c#L72). The [`__register_binfmt`](https://elixir.bootlin.com/linux/v4.9.35/source/fs/exec.c#L75) will add a linux_binfmt struct into the list. The linux_binfmt contains three important function pointers, `load_binary`, `load_shlib`, and `core_dump`.

```
struct linux_binfmt {
    struct list_head lh;
    struct module *module;
    int (*load_binary)(struct linux_binprm *);
    int (*load_shlib)(struct file *);
    int (*core_dump)(struct coredump_params *cprm);
    unsigned long min_coredump;     /* minimal dump size */
};

static struct linux_binfmt elf_format = {
    .module         = THIS_MODULE,
    .load_binary    = load_elf_binary,
    .load_shlib     = load_elf_library,
    .min_coredump   = ELF_EXEC_PAGESIZE,
    .core_dump      = elf_core_dump,
};
```

Take the elf format as an example, [`init_elf_binfmt`](https://elixir.bootlin.com/linux/v4.9.35/source/fs/binfmt_elf.c#L2323) will call register_binfmt to put elf_format into the formats list.

`exec_binprm` -> `search_binary_handler`, which will in turn call fmt-> load_binary, which is [`load_elf_binary`](https://elixir.bootlin.com/linux/v4.9.35/source/fs/binfmt_elf.c#L668).The critical code snippet of load_elf_binay is
```
retval = flush_old_exec(bprm);
...
setup_new_exec(bprm);
...
retval = setup_arg_pages(bprm, randomize_stack_top(STACK_TOP), executable_stack);
...
current->mm->end_code = end_code;
current->mm->end_data = end_data;
current->mm->start_code = start_code;
current->mm->start_data = start_data;
...
current->mm->start_stack = bprm->p;
start_thread(regs, elf_entry, bprm->p);
```

1. When does the child give up mm struct copied from parent?

[`flush_old_exec`](https://elixir.bootlin.com/linux/v4.9.35/source/fs/exec.c#L1254) -> [`exec_mmap`](https://elixir.bootlin.com/linux/v4.9.35/source/fs/exec.c#L1015) frees the parent's mm, the details code logic is in [`exec_mmap`](https://elixir.bootlin.com/linux/v4.9.35/source/fs/exec.c#L1015) -> [`mmput`](https://elixir.bootlin.com/linux/v4.9.35/source/kernel/fork.c#L880) -> [`exit_mmap`](https://elixir.bootlin.com/linux/v4.9.35/source/mm/mmap.c#L2945).

2. When the new forked process get its name?

`load_elf_binary `--> `setup_new_exec` --> `__set_task_comm`
So when the process gets forked, its name is not set yet, when it gets to run the execve, its name will be updated.

#### How the user space arguments pass to the program?

`do_execveat_common` -> `bprm_mm_init` -> ` __bprm_mm_init`, in which it will initialize a vma, and the vma has only one page which is stack.

`do_execveat_common` --> `copy_string` will find and link a page frame with bprm through [`get_arg_page`](https://elixir.bootlin.com/linux/v4.9.35/source/fs/exec.c#L546), and then will kmap the arg_page and copy all the user space arguments into that page. This page will be set as stack in `load_elf_binary` -> [`setup_arg_pages`](https://elixir.bootlin.com/linux/v4.9.35/source/fs/exec.c#L677). In `setup_arg_pages`, `bprm->p` saves the stack top, and it is passed to `start_thread` and saved to `pt_regs->sp`, which is loaded to stack pointer register after returning to user space.
```
static inline void start_thread(struct pt_regs *regs, unsigned long pc,
				unsigned long sp)
{
	start_thread_common(regs, pc);
	regs->pstate = PSR_MODE_EL0t;
	regs->sp = sp;
}
```

#### How the entry point (not main) in elf is reached?

* For syscall SyS_execve, the call chain is SyS_execve -> do_execve -> do_execve_common -> exec_binprm -> search_binary_handler -> load_elf_binary -> start_thread. In start_thread, the pc in pt_regs will be set to the entry point in elf.

```
static int load_elf_binary(struct linux_binprm *bprm)
{
	......
	if (elf_interpreter) {
	......
	} else {
		elf_entry = loc->elf_ex.e_entry;
		if (BAD_ADDR(elf_entry)) {
			retval = -EINVAL;
			goto out_free_dentry;
		}
	}
	......
	start_thread(regs, elf_entry, bprm->p);
}

static inline void start_thread_common(struct pt_regs *regs, unsigned long pc)
{
	memset(regs, 0, sizeof(*regs));
	regs->syscallno = ~0UL;
	regs->pc = pc;
}

static inline void start_thread(struct pt_regs *regs, unsigned long pc,
				unsigned long sp)
{
	start_thread_common(regs, pc);
	regs->pstate = PSR_MODE_EL0t;
	regs->sp = sp;
}
```

* The execution path will return to ret_fast_syscall, as `b ret_fast_syscall` in [`el0_svc`](https://elixir.bootlin.com/linux/v4.9.35/source/arch/arm64/kernel/entry.S#L742). ret_fast_syscall will reload pt_regs and return to the user space. Now the pc points to the entry point in elf, the execution path goes to elf.



### Init process - from kernel thread to user thread
init process begins as a kernel thread, `start_kernel` -> `rest_init` -> `kernel_thread` 
The kernel_init function pointer is passed to kernel_thread as the first parameter. 

Details of how kernel thread is forked can be found in [Kernel thread vs user thread](https://wenboshen.org/posts/2016-04-11-kernel-fork.html#kernel-thread-vs-user-thread). 
Here we want to focus on how init process becomes a user thread. 

```
static int __ref kernel_init(void *unused)
{
	......
	if (execute_command) {
		ret = run_init_process(execute_command);
		if (!ret)
			return 0;
		panic("Requested init %s failed (error %d).",
		      execute_command, ret);
	}
	if (!try_to_run_init_process("/sbin/init") ||
	    !try_to_run_init_process("/etc/init") ||
	    !try_to_run_init_process("/bin/init") ||
	    !try_to_run_init_process("/bin/sh"))
		return 0;

	panic("No working init found.  Try passing init= option to kernel. "
	      "See Linux Documentation/init.txt for guidance.");
}

static int run_init_process(const char *init_filename)
{
	argv_init[0] = init_filename;
	return do_execve(getname_kernel(init_filename),
		(const char __user *const __user *)argv_init,
		(const char __user *const __user *)envp_init);
}
```

Here we can see that `kernel_init` -> `run_init_process` -> `do_execve`, which is the same with a regular execve syscall, the argument is `init` binary. 


### User mode helper
user mode helper allows kernel to run a user space binary, one example is in kernel/reboot.c, `run_cmd` calls user mode helper.

user mode helper code is mainly in [kernel/kmod.c](https://elixir.bootlin.com/linux/v4.9.35/source/kernel/kmod.c#L616). [`call_usermodehelper`](https://elixir.bootlin.com/linux/v4.9.35/source/kernel/kmod.c#L616) -> [`call_usermodehelper_exec`](https://elixir.bootlin.com/linux/v4.9.35/source/kernel/kmod.c#L555) ->  [`call_usermodehelper_exec_work`](https://elixir.bootlin.com/linux/v4.9.35/source/kernel/kmod.c#L321) which will create a kernel_thread and pass [`call_usermodehelper_exec_async`](https://elixir.bootlin.com/linux/v4.9.35/source/kernel/kmod.c#L215) as the function parameter, same to the init process. call_usermodehelper_exec_async will call do_execve, same to kernel_init function. The do_execve will load the file in argv[0], which is reboot_cmd `/sbin/reboot`.

Here, as the reboot_cmd is global read/write string, it give the attacker chance to overwrite it to any string path. For example, as show in [New Reliable Android Kernel Root
Exploitation Techniques](http://powerofcommunity.net/poc2016/x82.pdf), the attacker can use arbitrary kernel memory read to write it to be a binary file in /data, which is controlled by the attacker. Then the attacker can trigger the kernel thread (uid is root) to load a binary which will give him the root shell.

```
char poweroff_cmd[POWEROFF_CMD_PATH_LEN] = "/sbin/poweroff";
static const char reboot_cmd[] = "/sbin/reboot";
 
static int run_cmd(const char *cmd)
{
    char **argv;
    static char *envp[] = {
        "HOME=/",
        "PATH=/sbin:/bin:/usr/sbin:/usr/bin",
        NULL
    };
    int ret;
    argv = argv_split(GFP_KERNEL, cmd, NULL);
    if (argv) {
        ret = call_usermodehelper(argv[0], argv, envp, UMH_WAIT_EXEC);
        argv_free(argv);
    } else {
        ret = -ENOMEM;
    }
 
    return ret;
}
 
static int __orderly_reboot(void)
{
    int ret;
    ret = run_cmd(reboot_cmd);
    ......
    return ret;
}
```

### References
1. [New Reliable Android Kernel Root
Exploitation Techniques](http://powerofcommunity.net/poc2016/x82.pdf)
