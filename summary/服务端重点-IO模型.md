# I/O模型

I/O 是什么？

I/O 即 input/output，输入/输出。

对计算机I/O设备。输入设备向计算机输入信息，比如键盘，鼠标。显示器，打印机就是输出设备。

对磁盘I/O。输入就是从磁盘读取数据到内存，输出就是将内存数据写入磁盘。

## 操作系统I/O

操作系统负责计算机的资源管理和进程调度。当应用程序想做磁盘文件读写、内存读写时，这些操作需要调用操作系统开放的API才能实现。

### 什么是虚拟地址空间？用户空间和内核空间呢?

[用户空间和内核空间的区别](https://zhuanlan.zhihu.com/p/343597285)

[如何理解虚拟地址空间？](https://www.zhihu.com/question/290504400/answer/1964845950)

#### 虚拟地址空间

假设现在有一个4G物理内存的32位Linux操作系统。

为了在进程运行时，能访问4G物理内存。操作系统为每一个进程都分配了4G(2的32次方)的虚拟地址空间。

操作系统会提供一种机制，将不同进程的虚拟地址和不同内存的物理地址映射起来。

如果程序要访问虚拟地址，操作系统通过[CPU 芯片中的内存管理单元（MMU）的映射关系]将虚拟内存转换成不同的物理地址，这样不同的进程运行的时候，写入的是不同的物理地址，这就避免了多进程可能操作同一个物理内存地址导致的冲突。

当开启大量进程导致物理内存紧张时，操作系统会通过内存交换技术，把不常使用的内存暂时存放到硬盘（换出），在需要的时候再装载回物理内存（换入），这就解决了物理内存不足的问题。

在 Linux 操作系统中，这4G可访问的虚拟地址空间又被分为内核空间和用户空间两部分，不同位数的系统，地址空间的范围也不同。

以32 位系统为例，内核空间占用 1G，位于最高处，剩下的 3G 是用户空间。

内核空间是操作系统内核才能访问的区域，是受保护的内存空间。而用户空间是用户应用程序访问的内存区域。

也就是说

- 进程在用户态时，只能访问用户空间内存，CPU指令也会受到限制；
- 只有进入内核态后，才可以访问内核空间的内存，CPU指令不再受限；

#### 什么是用户态/内核态？以及他们是如何切换的？

当进程/线程运行在内核空间时就处于内核态，运行在用户空间时则处于用户态。

可以通过什么方式实现用户态到内核态的切换？

- 系统调用。进程执行系统调用时会陷入内核代码中执行。
- 软中断是指进程发生了异常事件；
- 硬中断就有很多种，例如时钟周期、I/O等。

虽然每个进程都各自有独立的虚拟内存，但是每个虚拟内存中的内核地址，其实关联的都是相同的物理内存。这样，进程切换到内核态后，就可以很方便地访问内核空间内存。

也就是说，每个进程都可以通过系统调用进入内核态，因此，Linux内核由系统内的所有进程共享。

#### 32位Linux操作系统虚拟内存分布图

![32位Linux操作系统虚拟内存分布图](https://picx.zhimg.com/80/v2-a4c01da530e342cccd2d178d33f0e16a_720w.webp?source=1940ef5c)

以C/C++编写的程序为例，从低到高分别是 7 种不同的内存段：

- 程序文件段，包括二进制可执行代码；
- 已初始化数据段，包括静态常量；
- 未初始化数据段，包括未初始化的静态变量；
- 堆段，包括动态分配的内存，从低地址开始向上增长；
- 文件映射段，包括动态库、共享内存等，从低地址开始向上增长（跟硬件和内核版本有关）；
- 栈段，包括局部变量和函数调用的上下文等。栈的大小是固定的，一般是 8 MB。当然系统也提供了参数，以便我们自定义大小；
- 内核空间，虚拟地址从0xC0000000到0xFFFFFFFF

这样，就实现了每个进程可以操纵 4G 字节的虚拟空间的目标。

## 操作系统的一次 I/O 过程

实际上，应用程序不能实现真正的I/O，真正的I/O必然是在操作系统层面完成的。

也就是说，应用程序的I/O分成两种动作：I/O调用和I/O执行。

进程在用户态发起I/O系统调用，然后操作系统内核代码开始I/O执行。

最终操作系统通过某种方式将数据从内核态传输回用户态。

在操作系统内核完成I/O执行的过程中还包括两个步骤：

- 准备数据阶段：内核等待I/O设备准备好数据
- 拷贝数据阶段：将数据从内核缓冲区拷贝到用户进程缓冲区

如图：
![操作系统的一次I/O过程](https://s6.51cto.com/oss/202112/01/fffa9054b8c8605c78d1aabecd340454.jpg)

后面将会围绕这个过程展开关于I/O模型的分析，让有限的CPU和内存资源能提供高效的服务。在网络I/O层面，就是设计出高并发的高吞吐的网络服务器。

## 阻塞 I/O 模型（BIO）

假设应用程序的进程发起I/O调用，但是如果内核的数据还没准备好的话，**进程就一直在阻塞等待**，一直等到内核数据准备好了，操作系统将数据从内核拷贝到用户空间，才返回成功提示。

此次I/O操作，因为进程从发起系统调用起一直被阻塞，直到系统调用成功返回。所以称之为阻塞I/O。

![阻塞I/O模型](https://s2.51cto.com/oss/202112/01/de7ae40ad632328a2e4021dad080b337.jpg)

阻塞I/O阻塞了太久用户进程，浪费计算机性能，可以使用非阻塞I/O优化。

## 非阻塞I/O模型（NIO）

如果内核数据还没准备好，可以先返回错误信息给用户进程，让它不需要等待，然后通过**轮询**的方式再来请求。这就是非阻塞I/O。

![非阻塞I/O模型](https://s5.51cto.com/oss/202112/01/5bccfc91f778ae5c64b42f574ae989c4.jpg)

非阻塞I/O的流程如下：

- 应用进程向操作系统内核，发起recvfrom读取数据。
- 操作系统内核数据没有准备好，立即返回EWOULDBLOCK错误码。
- 应用程序进程轮询调用，继续向操作系统内核发起recvfrom读取数据。
- 操作系统内核数据准备好了，从内核缓冲区拷贝到用户空间。
- 完成调用，返回成功提示。

它相对于阻塞I/O，虽然大幅提升了性能，但是它依然存在性能问题，即**频繁的轮询，导致频繁的系统调用，同样会消耗大量的CPU资源**。可以考虑I/O多路复用模型，去解决这个问题。

## I/O多路复用模型

如果等到内核数据准备好了，主动通知应用进程再次发起系统调用呢？

先回顾一下文件描述符

### 文件描述符 File Descriptor

[存储基础 — 文件描述符 fd 究竟是什么？](https://zhuanlan.zhihu.com/p/364617329)

[彻底弄懂 Linux 下的文件描述符](https://blog.csdn.net/yushuaigee/article/details/107883964)

在Linux系统中一切皆可以看成是文件，文件又可分为：普通文件、目录文件、链接文件和设备文件。

在操作这些所谓的文件的时候，我们每操作一次就找一次名字，这会耗费大量的时间和效率。所以Linux中规定每一个文件对应一个索引，这样要操作文件的时候，我们直接找到索引就可以对其进行操作了。

文件描述符（file descriptor）就是内核为了高效管理这些已经被打开的文件所创建的索引，它是一个非负整数（通常是小整数），用于指代被打开的文件，所有执行I/O操作的系统调用都通过文件描述符来实现。

同时还规定系统刚刚启动的时候，0是标准输入，1是标准输出，2是标准错误。这意味着如果此时去打开一个新的文件，它的文件描述符会是3。

Linux内核对所有打开的文件有一个文件描述符表，里面存储了每个文件描述符作为索引与一个打开文件相对应的关系。

实际上关于文件描述符，Linux内核维护了3个数据结构：

- 进程级的文件描述符表
- 系统级的打开文件描述符表
- 文件系统的i-node表

这里只涉及进程级文件描述符表。

一个 Linux 进程启动后，会在内核空间中创建一个 PCB 控制块，PCB 内部有一个文件描述符表（File descriptor table），记录着当前进程所有可用的文件描述符，也即当前进程所有打开的文件。进程级的描述符表的每一条记录了单个进程所使用的文件描述符的相关信息，进程之间相互独立。

当打开一个文件时，内核向进程返回一个文件描述符（ open 系统调用得到 ）。后续 read、write 这个文件时，则只需要用这个文件描述符来标识该文件，将其作为参数传入 read、write 。

但是应用进程不能阻塞地等待一个文件的读写事件完成后才返回结果，我们需要在操作系统对文件的读写过程中让应用进程可以抽身处理更多文件的读写请求，并转发给操作系统请求处理更多文件读写操作。让CPU、网络、多线程忙碌起来。

操作系统对文件读写的返回方面是不同I/O模型的关键，可以是告诉进程一直阻塞等待直到操作系统将数据传输到进程的用户空间（I/O多路复用之select模型），或是叫进程先不要阻塞等待了等操作系统处理好了会异步地通过信号告诉进程内核把数据传到了用户空间的哪里（异步IO模型）等等。

I/O复用模型核心思路是：

**系统给我们提供一类函数(如我们耳濡目染的select、poll、epoll函数)，它们可以同时监控多个fd的操作，任何一个返回内核数据就绪，应用进程再发起recvfrom系统调用**。

多路是指多个想要访问I/O模型的输入/输出端，在网络I/O中大部分情况下指多个TCP连接，也就是多个Socket 或者多个Channel；

复用是指复用一个或多个线程资源。

在网络I/O中，I/O多路复用意思就是说，一个或多个线程处理多个 TCP 连接。尽可能地减少系统开销，无需创建和维护过多的进程/线程。

### I/O多路复用之select

进程通过调用select函数，可以同时监控多个fd，**在select函数监控的fd中，只要有任何一个数据状态准备就绪了，select函数就会返回可读状态**，这时应用进程再发起recvfrom请求去读取数据。

![I/O多路复用之select](https://s3.51cto.com/oss/202112/01/3243e4b923429ceb74ca48c6b581eabe.jpg)

非阻塞I/O模型(NIO)中，需要N(N>=1)次轮询系统调用，然而借助select的I/O多路复用模型，只需要发起一次询问就够了,大大优化了性能。

但是呢，select有几个缺点：

监听的I/O最大连接数有限，在Linux系统上一般为1024。

select函数返回后，是通过遍历fdset，找到就绪的描述符fd。

也就是说**仅知道有I/O事件发生，却不知是哪几个流，所以要遍历所有流**

#### 重点解释Linux中select()用法

[Linux ： select()详解 和 实现原理](https://www.cnblogs.com/sky-heaven/p/7205491.html)

select的原型为

```c
int select (int nfds, fd_set *readfds, fd_set *writefds, fd_set *exceptfds, struct timeval *timeout);
```

函数的最后一个参数timeout是一个超时时间值。

第2、3、4三个参数是一样的类型 `fd_set *`。即我们在程序里要申请几个fd_set类型的变量，比如rdfds，wtfds，exfds，然后把这个变量的地址&rdfds,&wtfds,&exfds传递给select函数。

这三个参数都是一个句柄的集合。

- 第一个rdfds是用来保存这样的句柄的:当句柄的状态变成可读时系统就告诉select函数返回
- 第二个函数是指向有句柄状态变成可写时系统就会告诉select函数返回
- 第三个参数exfds是特殊情况，即句柄上有特殊情况发生时系统会告诉select函数返回。

总体来说，select会循环遍历它所监测的 `fd_set`（一组文件描述符(fd)的集合）内的所有文件描述符对应的驱动程序的poll函数。驱动程序提供的poll函数首先会将调用select的用户进程插入到该设备驱动对应资源的等待队列(如读/写等待队列)，然后返回一个bitmask告诉select当前资源哪些可用。当select循环遍历完所有fd_set内指定的文件描述符对应的poll函数后，如果没有一个资源可用(即没有一个文件可供操作)，则select让该进程睡眠，一直等到有资源可用为止，进程被唤醒(或者timeout)继续往下执行。

#### 重点解释Go中select()的实现

[Go 语言设计与实现 - select](https://draveness.me/golang/docs/part2-foundation/ch05-keyword/golang-select/)

C 语言的 select 系统调用可以同时监听多个文件描述符的可读或者可写的状态，Go 语言中的 select 也能够让 Goroutine 同时等待多个 Channel 可读或者可写，在多个文件或者 Channel状态改变之前，select 会一直阻塞当前线程或者 Goroutine。

设计逻辑：

- select 能在 Channel 上进行非阻塞的收发操作；
- select 在遇到多个 Channel 同时响应时，会随机执行一种情况；
- 当select语句中有default时
  - 当存在可以收发的 Channel 时，直接处理该 Channel 对应的 case；
  - 当不存在可以收发的 Channel 时，执行 default 中的语句；
- 同时有多个 case 就绪时，select 会随机选择一个已就绪的 case 执行（避免饥饿问题）

实现原理：

编译器根据select语句内的分支结构，决定将select翻译成不同的语句

- select 不存在任何的 case：直接阻塞Goroutine，并且无法被唤醒。
- select 只存在一个 case：编译器改写成if。如果channel是nil也会阻塞无法唤醒。
- select 存在两个 case，其中一个 case 是 default：改写成非阻塞模式收发Channel，哪怕缓冲区不足也会收发。
- select 存在多个 case：
  - 引入随机，为多个Channel确定轮询顺序（避免饥饿）和加锁顺序（避免死锁），并改写成if，然后for循环查找是否存在已经准备就绪的Channel。如果找到，就可以对Channel执行收发操作了。
  - 如果循环后并没有找到准备就绪的Channel，就将当前Goroutine加入到Channel的收发队列中(sendq或recvq)并挂起，等待其他Goroutine对Channel的操作来唤醒当前Goroutine。
  - 当其他Goroutine操作完Channel，也就是Channel准备就绪后，已挂起的Goroutine将被调度器唤醒。
  - 被唤醒的Goroutine不知道是哪个Channel就绪了，所以需要再次执行第一步的操作：for循环查找哪个Channel已就绪，此时就能找到那一个或另一个已就绪Channel执行收发操作了。

Go的实现无非是比C中多了对default的处理，其他方面用了Go自己的实现方式而已。

这正好印证了上面说的select缺点：**仅知道有I/O事件发生，却不知是哪几个流，所以要遍历所有流**

### I/O多路复用之poll

因为存在连接数限制，所以后来又提出了poll。与select用数组相比，**poll使用链表存储fd**，解决了连接数限制问题。但是呢，select和poll一样，还是需要通过遍历文件描述符来获取已经就绪的socket。如果同时连接的大量客户端，在一时刻可能只有极少处于就绪状态，伴随着监视的描述符数量的增长，效率也会线性下降。

因此经典的多路复用模型epoll诞生。

### I/O多路复用之epoll

epoll模型采用事件通知机制来触发相关的I/O操作。

相对于select和poll来说，epoll没有描述符个数限制，不需要在用户空间遍历文件描述符数组或链表，而是使用一个“虚拟”文件描述符管理多个实际的文件描述符，将用户关心的文件描述符的事件存放到内核的一个事件表中，这样在用户空间和内核空间只需copy一次。

epoll模型仅在Linux中实现。

![I/O多路复用之epoll](https://s3.51cto.com/oss/202112/01/c5310b9ceaf303e1fc9440b15318fd01.jpg)

[epoll内核源码详解](https://blog.csdn.net/runner668/article/details/80498202)

[epoll内核源码](https://tomoyo.osdn.jp/cgi-bin/lxr/source/fs/eventpoll.c?v=linux-6.1.52#L136)

#### 基本知识

##### 等待队列waitqueue

```c
 37 struct wait_queue_head {
 38         spinlock_t              lock;
 39         struct list_head        head;
 40 };
```

队列头结构(`wait_queue_head`)持有一个队列，队列头是资源生产者，队列成员是资源消费者。当头的资源ready后, 会逐个执行每个成员指定的回调函数，来通知它们资源已经ready了。

用网络服务器举例，epoll模型主要依赖三个函数。

#### `epoll_create()`函数：创建epoll

主要目的是创建一个epoll实例并返回相应的文件描述符。

进程调用 `epoll_create()`时，内核创建并初始化一个eventpoll对象ep。

```c
172 /*
173  * This structure is stored inside the "private_data" member of the file
174  * structure and represents the main data structure for the eventpoll
175  * interface.
176  */
177 struct eventpoll {
178         /*
179          * This mutex is used to ensure that files are not removed
180          * while epoll is using them. This is held during the event
181          * collection loop, the file cleanup path, the epoll file exit
182          * code and the ctl operations.
183          */
184         struct mutex mtx;
185 
186         /* Wait queue used by sys_epoll_wait() */
187         wait_queue_head_t wq;
188 
189         /* Wait queue used by file->poll() */
190         wait_queue_head_t poll_wait;
191 
192         /* List of ready file descriptors */
193         struct list_head rdllist;
194 
195         /* Lock which protects rdllist and ovflist */
196         rwlock_t lock;
197 
198         /* RB tree root used to store monitored fd structs */
199         struct rb_root_cached rbr;
200 
201         /*
202          * This is a single linked list that chains all the "struct epitem" that
203          * happened while transferring ready events to userspace w/out
204          * holding ->lock.
205          */
206         struct epitem *ovflist;
207 
208         /* wakeup_source used when ep_scan_ready_list is running */
209         struct wakeup_source *ws;
210 
211         /* The user that created the eventpoll descriptor */
212         struct user_struct *user;
213 
214         struct file *file;
215 
216         /* used to optimize loop detection check */
217         u64 gen;
218         struct hlist_head refs;
219 
220 #ifdef CONFIG_NET_RX_BUSY_POLL
221         /* used to track busy poll napi_id */
222         unsigned int napi_id;
223 #endif
224 
225 #ifdef CONFIG_DEBUG_LOCK_ALLOC
226         /* tracks wakeup nests for lockdep validation */
227         u8 nests;
228 #endif
229 };
```

ep对象初始化成员，包括自旋锁 `spin_lock`、互斥锁mutex、等待队列waitqueue、就绪链表rdllist（ready link list）、user信息（用户、最大监听fd数量等）等。另外ep对象还将初始化一个红黑树对象 `struct rb_root_cached rbr`，rbr用来存储受到监听的fd structs。

由于在文件描述符一节说道：

> 一个 Linux 进程启动后，会在内核空间中创建一个 PCB 控制块，PCB 内部有一个文件描述符表（File descriptor table），记录着当前进程所有可用的文件描述符，也即当前进程所有打开的文件。

现在内核中ep对象没有和任何fd关联，也就是说进程还不知道如何访问内核中的ep对象。为了让文件操作（fd）能找到真正的操作实现（eventpoll），需要让fd能找到struct file，struct file再找到eventpoll（内核eventpoll中文件操作的实现过程后面再说）。

所以要创建一个“虚拟”文件。创建一个匿名fd，并分配真实的struct file，再让struct file中的一个私有指针private_data指向ep对象。由于这个匿名fd具有和实际fd一样的访问方式，也就是说用户进程通过fd可以找到对应的内核fd（struct file），这样就实现了用户态fd->内核态fd（struct file）->eventpoll对象的查找过程了。

这个fd，就是epollfd，也叫epfd，将会返回给用户空间。

我们后面将用此epfd到eventpoll中管理我们真正关心的fd的I/O操作。

#### `epoll_ctl()`函数：设置epoll事件

主要目的是在内核创建epoll对象并返回epfd后，进程可以用 `epoll_ctl()`控制给定的文件描述符epfd指向fd相关的epoll实例。比如将我们关心的epoll对象添加到（或删除）要监听的fd（比如socket）中。

`epoll_ctl()`的内核实现：

其中op是ADD,MOD,DEL。fd是要监听的描述符。event是我们关心的events(event含data和events，data.fd是文件描述符比如listen_sock，events是一个字节的掩码如EPOLLIN | EPOLLET)。

```c
SYSCALL_DEFINE4(epoll_ctl, int, epfd, int, op, int, fd, struct epoll_event __user *, event)
{
    ...
    mutex_lock(&ep->mtx);
    /*
     * Try to lookup the file inside our RB tree, Since we grabbed "mtx"
     * above, we can be sure to be able to use the item looked up by
     * ep_find() till we release the mutex.
     */
    /* 对于每一个监听的fd, 内核都有分配一个epitem结构,
     * 而且我们也知道, epoll是不允许重复添加fd的,
     * 所以我们首先查找该fd是不是已经存在了.
     * ep_find()其实就是RBTREE查找, 跟C++STL的map差不多一回事, O(lgn)的时间复杂度.
     */
    epi = ep_find(ep, tfile, fd);
    error = -EINVAL;
    switch (op) {
        /* 首先我们关心添加 */
    case EPOLL_CTL_ADD:
        if (!epi) {
            /* 之前的find没有找到有效的epitem, 证明是第一次插入, 接受!
             * 这里我们可以知道, POLLERR和POLLHUP事件内核总是会关心的
             * */
            epds.events |= POLLERR | POLLHUP;
            /* rbtree插入, 详情见ep_insert()的分析
             * 其实我觉得这里有insert的话, 之前的find应该
             * 是可以省掉的... */
            error = ep_insert(ep, &epds, tfile, fd);
        } else
            /* 找到了!? 重复添加! */
            error = -EEXIST;
        break;
        /* 删除和修改操作都比较简单 */
    case EPOLL_CTL_DEL:
        if (epi)
            error = ep_remove(ep, epi);
        else
            error = -ENOENT;
        break;
    case EPOLL_CTL_MOD:
        if (epi) {
            epds.events |= POLLERR | POLLHUP;
            error = ep_modify(ep, epi, &epds);
        } else
            error = -ENOENT;
        break;
    }
    mutex_unlock(&ep->mtx);
    ...
}
```

解释一下整个epoll_ctl过程。

进程调用 `epoll_ctl()`后，内核从用户空间将要监听的 `epoll_event`结构copy到内核空间，存储到内核中的 `epoll_event`对象epds中。这里后面还会保存用户态传来的其他数据。

```c
 83 struct epoll_event {
 84         __poll_t events;
 85         __u64 data;
 86 } EPOLL_PACKED;
```

注意 `epoll_event`不是前面初始化的 `eventpoll`。`epoll_event`对象是*event类型，是保存用户数据的结构，最终将作为epitem结构的成员。

由于进程在 `epoll_create()`的返回值中拿到了一个外观上和普通fd一样的epfd。所以在内核的文件描述符表中能通过epfd找到一个对应的struct file结构，拿到这个文件file。

对于网络服务器来说，每新建一个连接的时候，就代表我们想添加一个想监听的fd（如socket连接）进I/O模型。这个fd是创建网络连接时内核返回的，所以在内核文件描述符表中当然也有个对应的struct file结构，现在拿到这个文件tfile。

注意epfd和fd不是一回事，file和tfile也不是一个文件。

想监听的tfile有特殊要求：

- tfile需要实现poll()方法，原因：内核后面将通过tfile->poll()的实现调用到callback()
- epoll不能监听自己，也就是tfile不能是file本身

通过file的私有成员private_data可以找到eventpoll对象ep。

前面提到过ep对象有一个用来存储受到监听的fd structs 的红黑树根成员rbr（reb black tree root cache pointer），现在要将受到监听的fd数据存储到红黑树。

接下来将会在ep的mutex锁中进行操作：

- 在ep对象中用tfile和fd查找（`ep_find`）数据(返回 `struct epitem`)。
  - 这个过程是红黑树(RBTREE)查找。遍历红黑树的节点 `ep->rbr.rb_root.rb_node`，找到匹配fd的节点，组装成epitem结构返回。时间复杂度O(logn)。能找到就表明当前fd已经受到ep对象的监听了，找不到就需要添加。
- switch观察op
  - ADD操作。如果ep对象中查找数据失败，表明现在是第一次插入，就调用 `ep_insert`将event对象epds的数据（存储了用户event数据的临时变量）及tfile、fd插入到内核ep对象中。
  - DEL操作。调用ep_remove
  - MOD修改操作。调用ep_modify

##### `ep_insert()`

```c
static int ep_insert(struct eventpoll *ep, struct epoll_event *event, struct file *tfile, int fd)
```

`ep_insert`创建一个 `struct epitem` 对象 epi，和一个 `struct ep_pqueue` 对象 epq。

```c
130 /*
131  * Each file descriptor added to the eventpoll interface will
132  * have an entry of this type linked to the "rbr" RB tree.
133  * Avoid increasing the size of this struct, there can be many thousands
134  * of these on a server and we do not want this to take another cache line.
135  */
136 struct epitem {
137         union {
138                 /* RB tree node links this structure to the eventpoll RB tree */
139                 struct rb_node rbn;
140                 /* Used to free the struct epitem */
141                 struct rcu_head rcu;
142         };
143 
144         /* List header used to link this structure to the eventpoll ready list */
145         struct list_head rdllink;
146 
147         /*
148          * Works together "struct eventpoll"->ovflist in keeping the
149          * single linked chain of items.
150          */
151         struct epitem *next;
152 
153         /* The file descriptor information this item refers to */
154         struct epoll_filefd ffd;
155 
156         /* List containing poll wait queues */
157         struct eppoll_entry *pwqlist;
158 
159         /* The "container" of this item */
160         struct eventpoll *ep;
161 
162         /* List header used to link this item to the "struct file" items list */
163         struct hlist_node fllink;
164 
165         /* wakeup_source used when EPOLLWAKEUP is set */
166         struct wakeup_source __rcu *ws;
167 
168         /* The structure that describe the interested events and the source fd */
169         struct epoll_event event;
170 };
```

epitem是一个链表。它的成员包括要链接到 `ep->rbr`的对象rbn、要链接到 `eventpoll ready list`的对象rdllink、关联的fd对象ffd、等待队列 `poll wait queues`对象pwqlist、epitem的容器ep的指针、关联的 `struct file`对象fllink、`struct epoll_event`对象 event等。

```c
231 /* Wrapper struct used by poll queueing */
232 struct ep_pqueue {
233         poll_table pt;
234         struct epitem *epi;
235 };
```

临时结构epq是用来串联epitem与eventpoll关系的，能通过受监听的event找到对应的callback回调函数。epq包含成员epitem。初始化epq的pt成员（`poll_table`）其实就是初始化包含 `poll_queue_proc _qproc`和 `__poll_t _key`的 `struct poll_table_struct`。其中 `pt->_qproc`就是可调用的callback函数，`pt->_key`是 `epi->event.events`，也就是如果某events就绪就要调用某callback。epq初始化之后，`__ep_eventpoll_poll()`会将pt结构插入到通过 `epi->ffd.file`也就是tfile找到的ep->poll_wait等待队列中。epq的作用到此为止。

每个对文件I/O的监听行为，都会调用一次 `ep_insert()`，每次都会产生一个epitem。这些监听某个文件的epitem会放到此文件的 `tfile->f_ep_links`链起来。也就是说一个文件上会链上所有监听自己的epitem。

从此开始，内核将不停地对每个tfile调用 `poll(tfile, &ept.pt)`方法，也就是遍历 `ep.poll_wait.epitem.epoll_event`。不同的tfile实现poll()的方式不同，比如tfile是一个socket连接文件，它的poll()就会经过层层 `xxx_poll()`调用来到回调函数。此时，如果内核操作已就绪，就调用指定的回调函数 `ep_ptable_queue_proc()`。如果内核操作未就绪，这次poll就不调用回调函数，等待下一次poll()来再判断。

最后，每个epitem对象epi都要链入到eventpoll对象ep的红黑树对象中。这才能满足前面遍历ep对象的rbr节点查找匹配fd的epitem结构的逻辑。

##### 总结 epoll 中内核的工作

梳理一下整个链路关系：

- fd能找到tfile，epfd能找到文件file和ep
- `epoll_ctl()`把用户空间传输来的event数据，通过epitem的周转，存储在红黑树对象ep.rbr中，另外 `epitem.epoll_evnet`也存储了event数据
- 临时变量epq.pt存储了用户提供的event到callback的映射，多个epq.pt链入ep.poll_wait等待队列
- 除了作为epq的成员，epitem还被链入epi.rddlink(就绪队列ep.rdllist)
- 内核循环监听就绪队列ep.rdllist当发现epitem中用户要求的event就绪时就通过event到 `ep.poll_wait.epitem.epoll_event`中找到对应的 `callback()`执行
- 用户空间在提交 `epoll_ctl()`后悔一直处于 `epoll_wait()`阻塞中，等待内核执行 `callback()`

这样的设计，保证用户空间通过 `epoll_ctl()`能把event和callback交给内核的eq对象随后阻塞等待事件回调 `epoll_wait()`，内核遍历events时发现有就绪的event就调用对应的 `callback()`，用户空间wait时等来了 `callback()`就表示某fd对应的某event就绪了，随后阻塞地执行系统调用从内核向用户空间一次性拷贝所有准备好的数据就可以了。

#### `epoll_wait()`函数：等待epoll事件回调

```c
SYSCALL_DEFINE4(epoll_wait, int, epfd, struct epoll_event __user *, events, int, maxevents, int, timeout)
```

其中maxevents告知内核有多少个events，必须要大于0。

进程在通过 `epoll_ctl()`提交了event和对应的callback后，就会阻塞地调用 `epoll_wait()`函数等待内核对 `callback()`的回调。

在内核函数 `do_epoll_wait()`中获取epfd的struct file，获取 `file->private_data`，得到eventpoll对象ep。

内核遍历就绪列表的过程也会进入睡眠，等待事件就绪的唤醒或一次周期到期：

```c
ep_poll(ep, events, maxevents, timeout)
```

在 `ep_poll()`中，为ep加 `spin_lock_irqsave`。如果ep的就绪链表rdllist（ready  linked list）为空，就把当前进程加入到waitqueue等待队列中。现在进入for死循环阻塞状态，直到就绪链表rdllist非空了，或者收到中断信息才退出循环。如果这两个条件不满足就只能在for中先解锁，再隔timeout时间睡眠后，重新加锁循环。

前面说到等待队列头的资源ready后, 会逐个执行每个成员指定的回调函数，来通知它们资源已经ready了。在实现上，内核event就绪，也就是等待队列的头ready了，会依次执行每个成员指定的回调函数，并把当前epitem放入就绪链表rdlist。epoll模型无论当前在阻塞还是休眠，都应该退出当前状态，并立即调用 `ep_send_events()`发出fd已就绪的通知到用户态。回调函数 `ep_poll_callback()`是被监听的fd的具体实现，是用户态决定的实现方式。

`ep_send_events()`将利用ep对象，发送events就绪可读的通知到用户空间。

在 `ep_send_events()`中，`ep_scan_ready_list()`以一种允许扫描代码调用 `f_op->poll()`的方式扫描就绪列表 `ep->rdllist`(ready linked list)并将要传输的数据 `struct ep_send_events_data esed`从内核空间拷贝到用户空间。

由于rdllist可能会源源不断地被塞入新的epitem，所以传输过程中需要对ep加锁，并把rdllist中所有监听到events的epitem转移到临时变量txlist中并清空rdllist。然后就在txlist中调用回调函数 `ep_send_events_proc`处理epitem并将数据从内核空间拷贝到用户空间。

注意在内核空间向用户空间拷贝数据时，允许将没处理完的epitem继续插入到rdllist中，再次进行之前的处理流程：判断rdllist非空时，调用callback，拷贝函数等。

简单说下 `ep_send_events_proc`，它循环整个txlist：

- 取出第一个成员epitem，移除 `epi->rdllink`。
- 再次 `epi->ffd.file->f_op->poll()`拿到revents。因为某些I/O设备驱动不一定支持将events传入等待队列，而且我们也希望拿到最新的events（万一events有了变化），所以需要主动拉取一次。
- 将内核空间的 `revents`或 `epi->event.events`组成 `struct ep_send_events_data`对象esed，将esed.events的data和events通过 `__put_user()`传递到用户空间。

```c
struct ep_send_events_data {
    int maxevents;
    struct epoll_event __user *events;
};
```

##### ET LT 传递数据

从内核向用户复制时有ET（`edge-triggered`）和非ET（LT，`level-triggered`）的区别。

- 之前的非ET模式会在拷贝结束后，再次将 `epitem->rdllink`添加到 `ep->rdllist`，在下一次 `epoll_wait`时会立即返回, 并通知给用户空间。当然如果这个被监听的fds确实没事件也没数据了, `epoll_wait`会返回一个0，空转一次。
- ET模式, epitem不会再进入到ready linked list，除非fd再次发生了状态改变, `ep_poll_callback`被调用。因为只有消息到来才会触发数据发送行为，所以这也要求数据要一次读完，否则剩余数据等到下次发送会造成应用端延迟。

ET与LT相比，耗费较少的CPU资源。某些网络库默认的触发模式就是ET。例如某个场景下，不需要特别依赖 I/O 复用函数的读事件信号，可以使用 ET 模式，由于触发次数少，可以减少一些不必要的触发，节省 CPU 时间片，提高效率。

```c
        revents = epi->ffd.file->f_op->poll(epi->ffd.file, NULL) &
            epi->event.events;
        if (revents) {
            /* 将当前的事件和用户传入的数据都copy给用户空间,
             * 就是epoll_wait()后应用程序能读到的那一堆数据. */
            if (__put_user(revents, &uevent->events) ||
                __put_user(epi->event.data, &uevent->data)) {
                list_add(&epi->rdllink, head);
                return eventcnt ? eventcnt : -EFAULT;
            }
            eventcnt++;
            uevent++;
            if (epi->event.events & EPOLLONESHOT)
                epi->event.events &= EP_PRIVATE_BITS;
            else if (!(epi->event.events & EPOLLET)) {
                /* 嘿嘿, EPOLLET和非ET的区别就在这一步之差呀~
                 * 如果是ET, epitem是不会再进入到ready list,
                 * 除非fd再次发生了状态改变, ep_poll_callback被调用.
                 * 如果是非ET, 不管你还有没有有效的事件或者数据,
                 * 都会被重新插入到ready list, 再下一次epoll_wait
                 * 时, 会立即返回, 并通知给用户空间. 当然如果这个
                 * 被监听的fds确实没事件也没数据了, epoll_wait会返回一个0,
                 * 空转一次.
                 */
                list_add_tail(&epi->rdllink, &ep->rdllist);
            }
        }

```

总结一下。

用户空间在 `epoll_ctl()`提交了event和callback后，就一直在中阻塞着。

内核在for死循环中阻塞监听ep的就绪链表rdllist，如果为空将以timeout为周期唤醒或睡眠。

内核I/O在读写操作就绪后将会加入到rdllist中。当for循环中监听到rdllist非空或睡眠timeout到期时将被唤醒。在唤醒期间，如果发现就绪链表rdllist有就绪的资源，就到等待队列 `wait_queue`中根据队列头 `struct wait_queue_head`找到它的成员指定的回调函数，依次调用每个回调函数通知用户空间前来认领已就绪的资源。

此时，用户空间还在 `epoll_wait()`的阻塞中。当内核调用了回调函数时，对应的fd就被激活。相对select和poll模型来说，就避免了轮询fd列表确认内核就绪状态了，相当于避免了多次用户态到内核态的请求。用户空间立即对fd进行系统调用recvfrom请求数据。内核将当前的事件（含events和data）和用户传入的数据都copy给用户空间。这个过程中，select和poll的多次阻塞拷贝数据过程变成了一次性传递所有数据。

#### 官方的使用demo

```c
#define MAX_EVENTS 10
struct epoll_event  ev, events[MAX_EVENTS];
int         listen_sock, conn_sock, nfds, epollfd;


/* Code to set up listening socket, 'listen_sock',
 * (socket(), bind(), listen()) omitted */

epollfd = epoll_create1( 0 );
if ( epollfd == -1 )
{
    perror( "epoll_create1" );
    exit( EXIT_FAILURE );
}

ev.events   = EPOLLIN;
ev.data.fd  = listen_sock;
if ( epoll_ctl( epollfd, EPOLL_CTL_ADD, listen_sock, &ev ) == -1 )
{
    perror( "epoll_ctl: listen_sock" );
    exit( EXIT_FAILURE );
}

for (;; )
{
    nfds = epoll_wait( epollfd, events, MAX_EVENTS, -1 );
    if ( nfds == -1 )
    {
        perror( "epoll_wait" );
        exit( EXIT_FAILURE );
    }

    for ( n = 0; n < nfds; ++n )
    {
        if ( events[n].data.fd == listen_sock )
        {
            conn_sock = accept( listen_sock,
                        (struct sockaddr *) &local, &addrlen );
            if ( conn_sock == -1 )
            {
                perror( "accept" );
                exit( EXIT_FAILURE );
            }
            setnonblocking( conn_sock );
            ev.events   = EPOLLIN | EPOLLET;
            ev.data.fd  = conn_sock;
            if ( epoll_ctl( epollfd, EPOLL_CTL_ADD, conn_sock,
                    &ev ) == -1 )
            {
                perror( "epoll_ctl: conn_sock" );
                exit( EXIT_FAILURE );
            }
        } else {
            do_use_fd( events[n].data.fd );
        }
    }
}
```

I/O多路复用的方案到此为止。

能不能酱紫：不用我老是去问你数据是否准备就绪，等我发出请求后，你数据准备好了通知我就行了，这就诞生了信号驱动I/O模型。

## 信号驱动模型

信号驱动I/O不再用主动询问的方式去确认数据是否就绪，而是向内核发送一个信号(调用sigaction的时候建立一个SIGIO的信号)，然后应用用户进程可以去做别的事，不用阻塞。

当内核数据准备好后，再通过SIGIO信号通知应用进程，数据准备好后的可读状态。

进程收到信号之后，立即调用recvfrom，去读取数据。

![信号驱动模型](https://s3.51cto.com/oss/202112/01/81ad3fba52d386979338876b1cedfbc4.jpg)

信号驱动I/O模型，在应用进程发出信号后，是立即返回的，不会阻塞进程。

##### 信号驱动的缺点

[](https://www.itzhai.com/articles/it-seems-not-so-perfect-signal-driven-io.html)

注意：POSIX信号通常不排队，这也就意味着，假如SIGIO处理函数正在执行，又有两个新的SIGIO信号传过来，会被当前处理函数阻塞掉，当当前SIGIO函数处理完，最后只有新的一个SIGIO会触发继续执行SIGIO函数，丢失了一个信号。

类似于多进程或者多线程程序的同步处理，操作需要保证原子性，就必须通过一定的手段来实现，这里主要是通过手动执行sigprocmask阻塞SIGIO来实现避免共享变量的竞争关系的。

信号机制在操作系统中似乎有点被过渡设计，当有大量IO操作的时候，可能会因为信号队列溢出导致没法进行通知。

所以，稳定性是信号I/O的最大问题。

它已经有异步操作的感觉了。但是你细看上面的流程图，发现数据复制到应用缓冲的时候，应用进程还是阻塞的。

回过头来看下，不管是BIO，还是NIO，还是信号驱动，在数据从内核复制到应用缓冲的时候，都是阻塞的。

还有没有优化方案呢?AIO(真正的异步I/O)!

## 异步I/O(AIO)

前面讲的BIO，NIO和信号驱动，在数据从内核复制到应用缓冲的时候，都是阻塞的，因此都不算是真正的异步。

AIO实现了I/O全流程的非阻塞，就是应用进程发出系统调用后，是立即返回的，但是立即返回的不是处理结果，而是表示提交成功类似的意思。

等内核数据准备好，将数据拷贝到用户进程缓冲区，发送信号通知用户进程I/O操作执行完毕。

![异步I/O](https://s3.51cto.com/oss/202112/01/16e73ad7806ab8fd3dcffe6b560472ed.jpg)

异步I/O的优化思路很简单，只需要向内核发送一次请求，就可以完成数据状态询问和数据拷贝的所有操作，并且不用阻塞等待结果。日常开发中，有类似思想的业务场景：

比如发起一笔批量转账，但是批量转账处理比较耗时，这时候后端可以先告知前端转账提交成功，等到结果处理完，再通知前端结果即可。

## 网络服务器

## 主要参考

[看一遍就理解：IO 模型详解](https://www.51cto.com/article/693213.html)

[IO多路复用与epoll原理探究](https://wendeng.github.io/2019/06/09/%E6%9C%8D%E5%8A%A1%E7%AB%AF%E7%BC%96%E7%A8%8B/IO%E5%A4%9A%E8%B7%AF%E5%A4%8D%E7%94%A8%E4%B8%8Eepoll%E5%8E%9F%E7%90%86%E6%8E%A2%E7%A9%B6/)

[同步IO（阻塞IO、非阻塞IO）, 异步IO的理解](https://blog.csdn.net/weixin_44306626/article/details/122990856)

[在 golang 中是如何对 epoll 进行封装的？](https://zhuanlan.zhihu.com/p/484458312)

[谈谈你对IO多路复用机制的理解](https://www.51cto.com/article/717096.html)

[Epoll的使用详解](https://www.jianshu.com/p/ee381d365a29)

[Linux源码 TOMOYO Linux Cross Reference linux-6.1.52](https://tomoyo.osdn.jp/cgi-bin/lxr/source/fs/eventpoll.c?v=linux-6.1.52#L973)
