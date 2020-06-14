---
title: aarch64-write-syscall
date: 2020-06-14 13:31:42
tags:
 - aarch64 linux write 系统调用
categories:
 - [linux]
---


> 参考文章
```
https://blog.csdn.net/yusakul/article/details/105706674
https://blog.csdn.net/weixin_42523774/article/details/103341058（glibc）
https://blog.csdn.net/yhb1047818384/article/details/51842607（寄存器）
https://www.cnblogs.com/nufangrensheng/p/3890856.html（::memory）
https://baijiahao.baidu.com/s?id=1604601045858159778&wfr=spider&for=pc（系统调用实现文件）
```

### 1、write流程

1、user调用write系统调用

2、glibc中转换为系统调用号，将系统调用号保存在x8寄存器中，同时保存返回地址在x0，接着触发svc异常

3、从EL0切换到EL1，进入内核通过aarch64的中断向量表，找到异常向量入口el0_sync（arch/arm64/kernel/entry.S）

4、el0_sync函数中判断异常类型是svc触发的异常

```
lsr x24, x25, #ESR_ELx_EC_SHIFT // exception class
```

5、接着进入el0_svc

```
b.eq    el0_svc
```

6、加载异常系统调用表

```
adrp    stbl, sys_call_table        // load syscall table pointer
uxtw    scno, w8            // syscall number in w8，从x8寄存起低32位中找打系统调用号
ldr x16, [stbl, scno, lsl #3]   // address in the syscall table
blr x16             // call sys_* routine
b   ret_fast_syscall		// 系统调用返回
```

7、第六步找通过系统调用号，在系统调用表中找到write的系统调用的位置

```
include/uapi/asm-generic/unistd.h	
    /* fs/read_write.c */
    #define __NR_write 64

fs/read_write.c
	SYSCALL_DEFINE3(write, unsigned int, fd, const char __user *, buf,size_t, count)
```

8、写入操作

```
 577 SYSCALL_DEFINE3(write, unsigned int, fd, const char __user *, buf,
 578         size_t, count)
 579 {
 580     struct fd f = fdget_pos(fd);
 581     ssize_t ret = -EBADF;
 582 
 583     if (f.file) {
 584         loff_t pos = file_pos_read(f.file);
 585         ret = vfs_write(f.file, buf, count, &pos);
 586         if (ret >= 0)
 587             file_pos_write(f.file, pos);
 588         fdput_pos(f);		// 写入
 589     }
 590 
 591     return ret;
 592 }
```



### 2、系统调用表

```
arch/arm64/kernel/sys.c
    void * const sys_call_table[__NR_syscalls] __aligned(4096) = {
    	[0 ... __NR_syscalls - 1] = sys_ni_syscall,	// 没有实现的系统调用会跳转到这里，内部直接返回
    #include <asm/unistd.h>	 // arch/arm64/include/asm/unistd.h
    };

arch/arm64/include/asm/unistd.h:56
	#define NR_syscalls (__NR_syscalls)
	
arch/arm64/include/uapi/asm/unistd.h
	#include <asm-generic/unistd.h>

include/asm-generic/unistd.h
	#include <uapi/asm-generic/unistd.h>
	
include/uapi/asm-generic/unistd.h	
	该文件中说明了系统调用的实现的文件
		write系统调用在fs/read_write.c  //SYSCALL_DEFINE3(write, unsigned int, fd, const char __user *, buf,size_t, count)
		getpid在kernel/sys.c // SYSCALL_DEFINE0(getpid)
	#define __NR_syscalls 285  // 当前系统有285个系统调用
	
include/linux/syscalls.h
	对系统调用申明
```

### 3、系统调用write的原子性讨论

https://blog.csdn.net/dog250/article/details/78879600

append模式通过锁主inode来保证原子，write系统调用本身并步保证原子