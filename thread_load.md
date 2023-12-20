# 进程加载
进程0存放在bootsector中，进程1是共享进程0的数据。因此这两个进程都是事先预定好的
那么如何真正创建一个新的进程并且加载呢?

---

bash/shell 资源管理器(作为父进程) 从而 创建子进程
fork()完后，该进程处于就绪态, 有task_struct 并且处于 task数组中

子进程的线性地址空间 copy_mem
物理地址与父进程一致，需要有页表项和页目录表项  父进程的 copy_page_tables
他们指向的是同一个物理地址。 给父子进程都加了锁 (this_table &= ~2)

---

文件上的代码 拷贝到 线性地址的代码区 ？ 无法直接做到

ip指针 如果是c语言程序 那么指向当前子进程的main
但是这里也是有问题的。因为父进程当时也

线性地址空间要不要动?  copy_mem 线性地址 copy_page_table 物理地址
还是要动的，段基址段限长 通常父子进程不一致。因此根据子进程重新调整线性地址
物理地址肯定也需要重新定。

物理地址只有16M，线性地址最大64M(单个进程)， 因此一次性拷贝到物理地址可能不行。

如果开始执行 eip->main
只有线性地址空间，没有物理地址空间，因此会导致缺页中断 do_no_page

---
do_no_page

我们页是实现分好的，必须讲究4k对齐(copy_page_table规定)
因此我们很容易算出来某一个地址缺少的是哪一个物理页，一个页4k,一个块1k， 因此我们从硬盘中对应的块读取到物理页中。

&这里我们假设硬盘和内存是同构的,这样才能够映射硬盘和内存。



先拖钩后加载

---

```c
task_struct {
    code_start, data_end, stack_start; //拷贝c语言中具体的数据段
}
```

1、fork task[] task_struct ldt dir page code data 共享
2、硬盘上的数据拷到 要执行的子进程的线性地址空间中。 虚拟拷贝——保证硬盘和线性地址同构
不同的操作系统文件构造的格式是不一样的，需要根据不同的os。 根据设置 来具体 安排线性地址空间，保证与硬盘中的数据一致
这些信息存储在文件头里。
3、实执行，依靠缺页中断

---

代码部分

```c
init () {
    if(! pid = fork()) //子进程从这里出来， eax =  0
    {// 子进程会执行，子进程自己加载程序
        if(open);
        execve(); //走syscall3 也走 int0x80
        // execve 调用 do_execve
        // 在这里是子进程加载过来的文件，但是这部分代码是父进程的代码。
        // 加载程序的代码本身也需要代码，这个代码是父进程提供的(共享父进程的)
    }
}
```

```c
do_execve() {
    // 硬中断压栈 从上往下走
    //& eip[i] 

    //namei 打开文件的i节点

    //判断是否有权限 uid 等 用户之间的权限
    bh = bread(inode->i_dev, inode->izone[0])
    // bread() 缓冲区 inode->i_zone[0]  可执行文件的第一个块， 第一个块就是文件头
    // 这个bread 只拷贝了一个块

    //exec 文件头的结构

    ex = (struct exec *) bh->b_data;  //我们读区该文件的文件头数据
    
    sys_close(i) //关闭共享的文件,文件脱钩

    // ldt[0] NULL ldt[1]用户进程代码段 ldt[2]用户进程数据段

    free_page_table(current->ldt[1]); // 脱钩父进程用户态
                                      //把自己的页表项清空，父进程只读。
                                      //目前跑的是内核代码
    p += change_ldt //根据自己的进程更改ldt
    current -> start_stack = p & 0xfffff000; 页对齐

    eip[0] = ex.a_entry // 更改了子进程的eip, 通过硬件压栈返回值，导致do_execve后直接执行子进程
    eip[3] = p //返回stack pointer
}
```

```c
struct exec{
    start;  //可执行文件的线性地址
    a_data //数据
    }
```

进程1创建进程2 init() 创建出来的pid一定不是0
但是eax返回为0，导致还会往下执行


