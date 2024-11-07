



execve 系统调用的主要流程：

```shell
execve
	|--> do_execve
	|		|--> do_execveat_common
	|		|		|--> __do_execve_file
    |       | 		|		|--> exec_binprm
    |       |       |  		|		|-->search_binary_handler
	|		|		|		|		|		|-->load_binary 
```



```c
SYSCALL_DEFINE3(execve,
		const char __user *, filename,
		const char __user *const __user *, argv,
		const char __user *const __user *, envp)
{
	return do_execve(getname(filename), argv, envp);
}
```

