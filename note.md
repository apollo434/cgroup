### Cgroup基本概念

A **cgroup** associates a set of tasks with a set of parameters for one
or more subsystems.

 Cgroups 是 control groups 的缩写,通俗的来说，cgroups可以限制、记录、隔离进程组所使用的物理资源（包括：CPU、memory、IO等），为容器实现虚拟化提供了基本保证，是构建Docker等一系列虚拟化管理工具的基石。最初由 google 的工程师提出,后来被整合进 Linux 内核。Cgroups 也是 LXC 为实现虚拟化所使用的资源管理手段,可以说没有 cgroups 就没有 LXC。

 Cgroups 最初的目标是为资源管理提供的一个**统一的框架**,既整合现有的 cpuset 等子系统,
也为未来开发新的子系统提供接口。

对开发者来说，cgroups 有如下四个有趣的特点：

1. cgroups 的 API 以一个**伪文件系统**的方式实现，即用户可以通过文件操作实现 cgroups 的组织管理。</br>
2. cgroups 的组织管理操作单元可以细粒度到线程级别，**用户态代码也可以针对系统分配的资源创建和销毁 cgroups**, 从而实现资源再分配和管理。</br>
3. 所有资源管理的功能都以“subsystem（子系统）”的方式实现，**接口统一**。</br>
4. 子进程创建之初与其父进程处于**同一个 cgroups 的控制组**。</br>


本质上来说，cgroups 是内核附加在程序上的**一系列钩子**（hooks），通过程序运行时对资源的调度触发相应的钩子以达到资源追踪和限制的目的。</br>

实现 cgroups 的主要目的是为不同用户层面的资源管理，提供一个**统一化的接口。** </br>

subsystem: 它类似于我们在netfilter中的过滤hook.比如上面的CPU占用率就是一个subsystem.简而言之.subsystem就是cgroup中可添加删除的模块.在cgroup架构的封装下为cgroup提供多种行为控制.subsystem在下文中简写成subsys.</br>

**"**A **subsystem** is a module that makes use of the task grouping
facilities provided by cgroups to treat groups of tasks in
particular ways.**"** A subsystem is typically a "resource controller" that
schedules a resource or applies per-cgroup limits, but it may be
anything that wants to act on a group of processes, e.g. a
virtualization subsystem.

>>
#### cgroups子系统 ####
>
1. cpu 子系统，主要限制进程的 cpu 使用率.
2. cpuacct 子系统，可以统计 cgroups 中的进程的 cpu 使用报告。
3. cpuset 子系统，可以为 cgroups 中的进程分配单独的 cpu 节点或者内存节点。
4. memory 子系统，可以限制进程的 memory 使用量。
5. blkio 子系统，可以限制进程的块设备 io。
6. devices 子系统，可以控制进程能够访问某些设备。
7. net_cls 子系统，可以标记 cgroups 中进程的网络数据包，然后可以使用 tc 模块
8. （traffic control）对数据包进行控制。
9. net_prio — 这个子系统用来设计网络流量的优先级
10. freezer 子系统，可以挂起或者恢复 cgroups 中的进程。
11. ns 子系统，可以使不同 cgroups 下面的进程使用不同的 namespace
12. hugetlb — 这个子系统主要针对于HugeTLB系统进行限制，这是一个大页文件系统。

A **hierarchy** is a set of cgroups arranged in a tree, such that
every task in the system is in exactly one of the cgroups in the
hierarchy, and a set of subsystems; each subsystem has system-specific
state attached to each cgroup in the hierarchy.  Each hierarchy has
an instance of the cgroup virtual filesystem associated with it.


按照资源的划分，系统被划分成了不同的子系统(subsystem)，正如我们上面列出的cpu, cpuset, blkio...每种资源独立构成一个subsystem.

可以将cgroup的架构抽象的理解为多根的树结构，一个hierarchy代表一棵树，树上绑定一个或多个subsystem.而树的叶子则是cgroup,一个cgroup具体的限制了某种资源。一个或多个cgroup组成一个css_set。简单来讲，就是一个资源限制集合(css_set)对一种subsystem(cpu，devices)的限制条件只能有一个，这是显然的吧...最终的task(进程)同css_set关联，从而达到限制资源的目的。具体cgroup和css_set 关联的方式, **see the second chart**

### cgroups 的作用

1. 资源限制（Resource Limitation）：cgroups 可以对进程组使用的资源总额进行限制。如设定应用运行时使用内存的上限，一旦超过这个配额就发出 OOM（Out of Memory）。
2. 优先级分配（Prioritization）：通过分配的 CPU 时间片数量及硬盘 IO 带宽大小，实际上就相当于控制了进程运行的优先级。
3. 资源统计（Accounting）： cgroups 可以统计系统的资源使用量，如 CPU 使用时长、内存用量等等，这个功能非常适用于计费。
4. 进程控制（Control）：cgroups 可以对进程组执行挂起、恢复等操作。

**NOTE:** 以上四条非常非常重要，洗完可以对照着下表去体会:</br>
![Alt text](/pic/1.png)</br>

##### 术语表

1. task（任务）：cgroups 的术语中，task 就表示系统的一个进程。
2. cgroup（控制组）：cgroups 中的**资源控制**都以**cgroup 为单位**实现。cgroup 表示按某种资源控制标准划分而成的任务组，包含一个或多个子系统。一个任务可以加入某个 cgroup，也可以从某个 cgroup 迁移到另外一个 cgroup。
3. subsystem（子系统）：cgroups 中的 subsystem 就是一个**资源调度控制器**（Resource Controller）。比如 CPU 子系统可以控制 CPU 时间分配，内存子系统可以限制 cgroup 内存使用量。
4. hierarchy（层级树）：hierarchy 由**一系列 cgroup** 以一个**树状**结构排列而成，每个 hierarchy 通过**绑定**对应的 **subsystem** 进行资源调度。hierarchy 中的 cgroup 节点可以包含零或多个子节点，子节点继承父节点的属性。整个系统可以有多个 hierarchy。</br>
**NOTE:** cgroups 的模型则是由多个 hierarchy 构成的森林</br>
Why?</br>
Because:如果只有一个 hierarchy，那么所有的 task 都要受到绑定其上的 subsystem 的限制，会给那些不需要这些限制的 task 造成麻烦。</br>

创建了 cgroups 层级结构中的节点（cgroup 结构体）之后，可以把进程加入到某一个节点的控制任务列表中，一个节点的控制列表中的所有进程都会受到当前节点的资源限制。同时某一个进程也可以被加入到不同的 cgroups 层级结构的节点中，因为不同的 cgroups 层级结构可以负责不同的系统资源。所以说进程和 cgroup 结构体是一个多对多的关系。

![Alt text](/pic/cgroup.png)</br>

##### 重要规则：
**规则 1**： 同一个 hierarchy 可以附加一个或多个 subsystem。如下图 1，cpu 和 memory 的 subsystem 附加到了一个 hierarchy。</br>
![Alt text](/pic/pic_1.png)</br>
**图 1 同一个 hierarchy 可以附加一个或多个 subsystem**</br>

**规则 2**： 一个 subsystem 可以附加到多个 hierarchy，当且仅当这些 hierarchy 只有这唯一一个 subsystem。如下图 2，小圈中的数字表示 subsystem 附加的时间顺序，CPU subsystem 附加到 hierarchy A 的同时不能再附加到 hierarchy B，因为 hierarchy B 已经附加了 memory subsystem。如果 hierarchy B 与 hierarchy A 状态相同，没有附加过 memory subsystem，那么 CPU subsystem 同时附加到两个
 hierarchy 是可以的。</br>
![Alt text](/pic/pic_2.png)</br>
**图 2 一个已经附加在某个 hierarchy 上的 subsystem 不能附加到其他含有别的 subsystem 的 hierarchy 上**</br>

**规则 3**： 系统每次新建一个 hierarchy 时，该系统上的**所有 task 默认构成了这个新建的 hierarchy 的初始化 cgroup**，这个 cgroup 也称为 **root cgroup**。对于你创建的每个 hierarchy，task 只能存在于其中一个 cgroup 中，即**一个 task 不能存在于同一个 hierarchy 的不同 cgroup 中**，但是一个 task 可以存在在不同 hierarchy 中的多个 cgroup 中。如果操作时把一个 task 添加到同一个 hierarchy 中的另一个 cgroup 中，则会从第一个 cgroup 中移除。在下图 3 中可以看到，httpd进程已经加入到 hierarchy A 中的/cg1而不能加入同一个 hierarchy 中的/cg2，但是可以加入 hierarchy B 中的/cg3。实际上不允许加入同一个 hierarchy 中的其他 cgroup 野生,**为了防止出现矛盾**，如 CPU subsystem 为/cg1分配了 30%，而为/cg2分配了 50%，此时如果httpd在这两个 cgroup 中，就会出现矛盾。</br>
![Alt text](/pic/pic_3.png)</br>
**图 3 一个 task 不能属于同一个 hierarchy 的不同 cgroup**</br>


**规则 4**： 进程（task）在 fork 自身时创建的子任务（child task）默认与原 task 在同一个 cgroup 中，但是 child task 允许被移动到不同的 cgroup 中。即 fork 完成后，父子进程间是完全独立的。如下图 4 中，小圈中的数字表示 task 出现的时间顺序，当httpd刚 fork 出另一个httpd时，在同一个 hierarchy 中的同一个 cgroup 中。但是随后如果 PID 为 4840 的httpd需要移动到其他 cgroup 也是可以的，因为父子任务间已经独立。总结起来就是：初始化时子任务与父任务在同一个 cgroup，但是这种关系随后可以改变。</br>
![Alt text](/pic/pic_4.png)</br>
**图 4 刚 fork 出的子进程在初始状态与其父进程处于同一个 cgroup**</br>

**NOTE:** subsystem 实际上就是 cgroups 的资源控制系统，每种 subsystem 独立地控制一种资源</br>

### 实践操作
查看cgroup挂载点（centos7.5）:

```

1 [root@k8s-master ~]# mount -t cgroup
2 cgroup on /sys/fs/cgroup/systemd type cgroup (rw,nosuid,nodev,noexec,relatime,xattr,release_agent=/usr/lib/systemd/systemd-cgroups-agent,name=systemd)
3 cgroup on /sys/fs/cgroup/pids type cgroup (rw,nosuid,nodev,noexec,relatime,pids)
4 cgroup on /sys/fs/cgroup/cpuset type cgroup (rw,nosuid,nodev,noexec,relatime,cpuset)
5 cgroup on /sys/fs/cgroup/memory type cgroup (rw,nosuid,nodev,noexec,relatime,memory)
6 cgroup on /sys/fs/cgroup/cpu,cpuacct type cgroup (rw,nosuid,nodev,noexec,relatime,cpuacct,cpu)
7 cgroup on /sys/fs/cgroup/blkio type cgroup (rw,nosuid,nodev,noexec,relatime,blkio)
8 cgroup on /sys/fs/cgroup/freezer type cgroup (rw,nosuid,nodev,noexec,relatime,freezer)
9 cgroup on /sys/fs/cgroup/hugetlb type cgroup (rw,nosuid,nodev,noexec,relatime,hugetlb)
10 cgroup on /sys/fs/cgroup/perf_event type cgroup (rw,nosuid,nodev,noexec,relatime,perf_event)
11 cgroup on /sys/fs/cgroup/devices type cgroup (rw,nosuid,nodev,noexec,relatime,devices)
12 cgroup on /sys/fs/cgroup/net_cls,net_prio type cgroup (rw,nosuid,nodev,noexec,relatime,net_prio,net_cls)


```

#### 创建隔离组 ####

```
[root@k8s-master ~]# cd /sys/fs/cgroup/cpu

[root@k8s-master cpu]# mkdir cpu_test

```

#### 目录创建完成会自动生成以下文件 ####

[root@k8s-master cpu]# ls cpu_test/

![Alt text](/pic/lz1.png)

写个死循环测试程序增加cpu使用率

```


  1 int main(void)
  2 {
  3     int i = 0;
  4     for(;;) i++;
  5     return 0;
  6 }
```
启动程序后cpu使用100%

![Alt text](/pic/lz2.png)

默认-1不限制，现在改成20000，可以理解使用率限制在20%

[root@k8s-master cpu]# echo 20000 > /sys/fs/cgroup/cpu/cpu_test/cpu.cfs_quota_us

找到进程号增加到cpu tasks里面，在看top  cpu使用率很快就下来

[root@k8s-master ~]# echo 23732 >> /sys/fs/cgroup/cpu/cpu_test/tasks

![Alt text](/pic/lz3.png)

### cgroups 实现方式及工作原理简介

#### cgroups 实现结构讲解

![Alt text](/pic/cgroup1.png)</br>
**图 5 cgroups 相关结构体一览**</br>
**NOTE：** 一个 task 只对应一个css_set结构，但是一个css_set可以被多个 task 使用.</br>

**task**:
1. 一个是css_set * cgroups，表示指向css_set（包含进程相关的 cgroups 信息）的指针.
2. list_head cg_list，是一个链表的头指针，这个链表包含了所有的链到同一个css_set的 task 进程.</br>

**css_set**:
每个css_set结构中都包含了一个指向cgroup_subsys_state（**包含进程与一个特定子系统相关的信息**）的指针数组。cgroup_subsys_state则指向了cgroup结构（包含一个 cgroup 的所有信息），通过这种方式间接的把一个进程和 cgroup 联系了起来，如下图 6。</br>
**task ----> cgroup**</br>
![Alt text](/pic/cgroup2.png)</br>
**图 6 从 task 结构开始找到 cgroup 结构**</br>

cgroup结构体中有一个list_head css_sets字段，它是一个头指针，指向由cg_cgroup_link（包含 cgroup 与 task 之间多对多关系的信息，后文还会再解释）形成的链表。由此获得的每一个cg_cgroup_link都包含了一个指向css_set * cg字段，指向了每一个 task 的css_set。css_set结构中则包含tasks头指针，指向所有链到此css_set的 task 进程构成的链表。至此，我们就明白如何查看在同一个 cgroup 中的 task 有哪些了，如下图
**cgroup ----> task**</br>
![Alt text](/pic/cgroup3.png)</br>
**图 7 cglink 多对多双向查询**</br>

css_set中有两种方式定位到cgroup：
1. css->cgroup_subsys_state->cgroup
2. css->cg_cgroup_link->cgroup</br>
**Q**:
but, why there are two ways to locate the cgroup?</br>
**Because**:task 与 cgroup 之间是多对多的关系.</br>
Moreover,如果两张表是多对多的关系，那么如果不加入第三张关系表，就必须为一个字段的不同添加许多行记录，导致大量冗余。通过从主表和副表各拿一个主键新建一张关系表，可以提高数据查询的灵活性和效率。<br>

一个 task 可能处于不同的 cgroup，只要这些 cgroup 在不同的 hierarchy 中，并且每个 hierarchy 挂载的子系统不同；另一方面，一个 cgroup 中可以有多个 task，这是显而易见的，但是这些 task 因为可能还存在在别的 cgroup 中，所以它们对应的css_set也不尽相同，所以一个 cgroup 也可以对应多个·css_set。</br>

**NOTE:** 在系统运行之初，内核的主函数就会对root cgroups和css_set进行初始化，每次 task 进行 fork/exit 时，**都会附加（attach）/ 分离（detach）对应的css_set**。</br>

**NOTE:** 添加cg_cgroup_link主要是出于**性能方面**的考虑，一是节省了task_struct结构体占用的内存，二是提升了进程fork()/exit()的速度。</br>

![Alt text](/pic/cgroup4.png)</br>
**图 8 css_set 与 hashtable 关系**
当 task 从一个 cgroup 中移动到另一个时，它会得到一个新的css_set指针。如果所要加入的 cgroup 与现有的 cgroup 子系统相同，那么就重复使用现有的css_set，否则就分配一个新css_set。所有的css_set通过一个哈希表进行存放和查询，如上图 8 中所示，hlist_node hlist就指向了css_set_table这个 hash 表。</br>


定义子系统的结构体是cgroup_subsys，在图 9 中可以看到，cgroup_subsys中定义了一组函数的接口，让各个子系统自己去实现，类似的思想还被用在了cgroup_subsys_state中，cgroup_subsys_state并没有定义控制信息，只是定义了各个子系统都需要用到的公共信息，由各个子系统各自按需去定义自己的控制信息结构体，最终在自定义的结构体中把cgroup_subsys_state包含进去，然后内核通过container_of（这个宏可以通过一个结构体的成员找到结构体自身）等宏定义来获取对应的结构体。</br>
![Alt text](/pic/cgroup5.png)</br>
**图 9 cgroup 子系统结构体**</br>

对于container_of这部分，以下面的freezer为例：</br>
```
struct freezer {
	struct cgroup_subsys_state	css;
	unsigned int			state;
};

static DEFINE_MUTEX(freezer_mutex);

static inline struct freezer *css_freezer(struct cgroup_subsys_state *css)
{
	return css ? container_of(css, struct freezer, css) : NULL;
}

```
or memory:</br>
```
struct mem_cgroup {
  struct cgroup_subsys_state css;
  struct res_counter res;
  struct res_counter memsw;
  struct mem_cgroup_lru_info info;
  spinlock_t reclaim_param_lock;
  int prev_priority;
  int last_scanned_child;
  bool use_hierarchy;
  atomic_t oom_lock;
  atomic_t refcnt;
  unsigned int swappiness;
  int oom_kill_disable;
  bool memsw_is_minimum;
  struct mutex thresholds_lock;
  struct mem_cgroup_thresholds thresholds;
  struct mem_cgroup_thresholds memsw_thresholds;
  struct list_head oom_notify;
  unsigned long move_charge_at_immigrate;
  struct mem_cgroup_stat_cpu *stat;
};

```

#### 基于 cgroups 实现结构的用户层体现
不多说了，下面这篇博客写的太棒了，直接copy过来：</br>
https://www.infoq.cn/article/docker-kernel-knowledge-cgroups-resource-isolation </br>
了解了 cgroups 实现的代码结构以后，再来看用户层在使用 cgroups 时的限制，会更加清晰。

在实际的使用过程中，你需要通过挂载（mount）cgroup文件系统新建一个层级结构，挂载时指定要绑定的子系统，缺省情况下默认绑定系统所有子系统。把 cgroup 文件系统挂载（mount）上以后，你就可以像操作文件一样对 cgroups 的 hierarchy 层级进行浏览和操作管理（包括权限管理、子文件管理等等）。**除了 cgroup 文件系统以外，内核没有为 cgroups 的访问和操作添加任何系统调用**。

如果新建的层级结构要绑定的子系统与目前已经存在的层级结构完全相同，那么新的挂载会重用原来已经存在的那一套（**指向相同的 css_set**）。否则如果要绑定的子系统已经被别的层级绑定，就会返回挂载失败的错误。如果一切顺利，挂载完成后层级就被激活并与相应子系统关联起来，可以开始使用了。

**NOTE:**</br>
**目前无法将一个新的子系统绑定到激活的层级上，或者从一个激活的层级中解除某个子系统的绑定。**

当一个顶层的 cgroup 文件系统被卸载（umount）时，如果其中创建后代 cgroup 目录，那么就算上层的 cgroup 被卸载了，层级也是激活状态，其后代 cgoup 中的配置依旧有效。只有递归式的卸载层级中的所有 cgoup，那个层级才会被真正删除。

层级激活后，/proc目录下的每个 task PID 文件夹下都会新添加一个名为cgroup的文件，列出 task 所在的层级，对其进行控制的子系统及对应 cgroup 文件系统的路径。

一个 cgroup 创建完成，不管绑定了何种子系统，其目录下都会生成以下几个文件，用来描述 cgroup 的相应信息。同样，把相应信息写入这些配置文件就可以生效，内容如下。

tasks：这个文件中罗列了所有在该 cgroup 中 task 的 PID。该文件并不保证 task 的 PID 有序，把一个 task 的 PID 写到这个文件中就意味着把这个 task 加入这个 cgroup 中。</br>
cgroup.procs：这个文件罗列所有在该 cgroup 中的线程组 ID。该文件并不保证线程组 ID 有序和无重复。写一个线程组 ID 到这个文件就意味着把这个组中所有的线程加到这个 cgroup 中。</br>
notify_on_release：填 0 或 1，表示是否在 cgroup 中最后一个 task 退出时通知运行release agent，默认情况下是 0，表示不运行。
release_agent：指定 release agent 执行脚本的文件路径（该文件在最顶层 cgroup 目录中存在），**在这个脚本通常用于自动化umount无用的 cgroup。**</br>
除了上述几个通用的文件以外，绑定特定子系统的目录下也会有其他的文件进行子系统的参数配置。

在创建的 hierarchy 中创建文件夹，就类似于 fork 中一个后代 cgroup，后代 cgroup 中默认继承原有 cgroup 中的配置属性，但是你可以根据需求对配置参数进行调整。这样就把一个大的 cgroup 系统分割成一个个嵌套的、可动态变化的“软分区”。


#### cgroups 的使用方法:</br>
**查询 cgroup 及子系统挂载状态:**</br>
1. 查看所有的 cgroup：lscgroup
2. 查看所有支持的子系统：lssubsys -a
3. 查看所有子系统挂载的位置： lssubsys –m
4. 查看单个子系统（如 memory）挂载位置：lssubsys –m memory</br>
**创建 hierarchy 层级并挂载子系统:**</br>
使用 cgroup 的最佳方式是：为想要管理的每个或每组资源创建单独的 cgroup 层级结构。而创建 hierarchy 并不神秘，实际上就是**做一个标记**，通过挂载一个 tmpfs{![基于内存的临时文件系统，详见：http://en.wikipedia.org/wiki/Tmpfs]}文件系统，并给一个好的名字就可以了，系统默认挂载的 cgroup 就会进行如下操作。</br>

```
mount -t tmpfs cgroups /sys/fs/cgroup

```
其中-t即指定挂载的文件系统类型，其后的cgroups是会出现在mount展示的结果中用于标识，可以选择一个有用的名字命名，最后的目录则表示文件的挂载点位置。</br>

挂载完成tmpfs后就可以通过mkdir命令创建相应的文件夹。</br>
```
mkdir /sys/fs/cgroup/cg1

```
再把子系统挂载到相应层级上，挂载子系统也使用 mount 命令，语法如下。</br>
```
mount -t cgroup -o subsystems name /cgroup/name

```
其​​​中​​​ subsystems 是​​​使​​​用​​​,（逗号）​​​分​​​开​​​的​​​子​​​系​​​统​​​列​​​表，name 是​​​层​​​级​​​名​​​称​​​。具体我们以挂载 cpu 和 memory 的子系统为例，命令如下。</br>
```
mount –t cgroup –o cpu,memory cpu_and_mem /sys/fs/cgroup/cg1

```
从mount命令开始，-t后面跟的是挂载的文件系统类型，即cgroup文件系统。-o后面跟要挂载的子系统种类如cpu、memory，用逗号隔开，其后的cpu_and_mem不被 cgroup 代码的解释，但会出现在 /proc/mounts 里，可以使用任何有用的标识字符串。最后的参数则表示挂载点的目录位置。</br>

说明：如果挂载时提示mount: agent already mounted or /cgroup busy，则表示子系统已经挂载，需要先卸载原先的挂载点，通过第二条中描述的命令可以定位挂载点。</br>

**卸载 cgroup**</br>
目前cgroup文件系统虽然支持重新挂载，但是官方不建议使用，重新挂载虽然可以改变绑定的子系统和release agent，但是它要求对应的 hierarchy 是空的并且 release_agent 会被传统的fsnotify（内核默认的文件系统通知）代替，这就导致重新挂载很难生效，未来重新挂载的功能可能会移除。你可以通过卸载，再挂载的方式处理这样的需求。

卸载 cgroup 非常简单，你可以通过cgdelete命令，也可以通过rmdir，以刚挂载的 cg1 为例，命令如下。</br>
```
rmdir /sys/fs/cgroup/cg1

```
rmdir 执行成功的必要条件是 cg1 下层没有创建其它 cgroup，cg1 中没有添加任何 task，并且它也没有被别的 cgroup 所引用。

cgdelete cpu,memory:/ 使用cgdelete命令可以递归的删除 cgroup 及其命令下的后代 cgroup，并且如果 cgroup 中有 task，那么 task 会自动移到上一层没有被删除的 cgroup 中，如果所有的 cgroup 都被删除了，那 task 就不被 cgroups 控制。但是一旦再次创建一个新的 cgroup，所有进程都会被放进新的 cgroup 中。</br>

**设置 cgroups 参数**</br>
设置 cgroups 参数非常简单，直接对之前创建的 cgroup 对应文件夹下的文件写入即可，举例如下。</br>
```
设置 task 允许使用的 cpu 为 0 和 1. echo 0-1 > /sys/fs/cgroup/cg1/cpuset.cpus

```
使用cgset命令也可以进行参数设置，对应上述允许使用 0 和 1cpu 的命令为：</br>
```
cgset -r cpuset.cpus=0-1 cpu,memory:/

```
**添加 task 到 cgroup**</br>
通过文件操作进行添加 echo [PID] > /path/to/cgroup/tasks 上述命令就是把进程 ID 打印到 tasks 中，如果 tasks 文件中已经有进程，需要使用">>"向后添加。

通过cgclassify将进程添加到 cgroup cgclassify -g subsystems:path_to_cgroup pidlist 这个命令中，subsystems指的就是子系统（如果使用 man 命令查看，可能也会使用 controllers 表示）​​​，如果 mount 了多个，就是用","隔开的子系统名字作为名称，类似cgset命令。

通过cgexec直接在 cgroup 中启动并执行进程 cgexec -g subsystems:path_to_cgroup command arguments command和arguments就表示要在 cgroup 中执行的命令和参数。cgexec常用于执行临时的任务。</br>

**权限管理**</br>
  与文件的权限管理类似，通过chown就可以对 cgroup 文件系统进行权限管理。</br>
  ```
  chown uid:gid /path/to/cgroup

  ```
  uid 和 gid 分别表示所属的用户和用户组。</br>

  #### subsystem 配置参数用法</br>
这部分没什么可说的，直接看下面的博客吧，都是些硬指标：</br>
https://www.infoq.cn/article/docker-kernel-knowledge-cgroups-resource-isolation

--------
### cgroup框架分析

在讲 cgroup 文件系统的实现之前,必须简单的介绍一下 Linux VFS。</br>
VFS 是所谓的虚拟文件系统转换,是一个内核软件层,用来处理与 Unix 标准文件系统的所有系
统调用。VFS 对用户提供统一的读写等文件操作调用接口,当用户调用读写等函数时,内核则调
用特定的文件系统实现。具体而言,文件在内核内存中是一个 file 数据结构来表示的。这个数
据结构包含一个 f_op 的字段,该字段中包含了一组指向特定文件系统实现的函数指针。当用户
执行 read()操作时,内核调用 sys_read(),然后 sys_read()查找到指向该文件属于的文件
系统的读函数指针,并调用它,即 file->f_op->read().</br>
VFS 其实是面向对象的,在这里,对象是一个软件结构,既定义数据也定义了之上的操作。处于
效率,Linux 并没有采用 C++之类的面向对象的语言,而是采用了 C 的结构体,然后在结构体里
面定义了一系列函数指针,这些函数指针对应于对象的方法。</br>

VFS 文件系统定义了以下对象模型:</br>
超级块对象(superblock object)</br>
存放已安装文件系统的有关信息。</br>
索引节点对象(inode object)</br>
存放关于具体文件的一般信息。</br>
文件对象(file object)</br>
存放打开文件与进程之间的交互信息</br>
目录项对象(dentry object)</br>
存放目录项与对应文件进行链接的有关信息。</br>

基于 VFS 实现的文件系统,都必须实现定义这些对象,并实现这些对象中定义的函数指针。</br>
cgroup 文件系统也不例外,下面我们来看 cgroups 中这些对象的定义。</br>

cgroup 文件系统的定义:
```
static struct file_system_type cgroup_fs_type = {
.name = "cgroup",
.get_sb = cgroup_get_sb,
.kill_sb = cgroup_kill_sb,
};

```
这里有定义了两个函数指针,定义了一个文件系统必须实现了的两个操作
get_sb,kill_sb,即获得超级块和释放超级块。这两个操作会在使用 mount 系统调用挂载
cgroup 文件系统时使用。
cgroup 超级块的定义:

```
static const struct super_operations cgroup_ops = {
.statfs = simple_statfs,
.drop_inode = generic_delete_inode,
.show_options = cgroup_show_options,
.remount_fs = cgroup_remount,
};

```
Cgroup 索引块定义:
```
static const struct inode_operations cgroup_dir_inode_operations = {
.lookup = simple_lookup,
.mkdir = cgroup_mkdir,
.rmdir = cgroup_rmdir,
.rename = cgroup_rename,
};

```
在 cgroup 文件系统中,使用 mkdir 创建 cgroup 或者用 rmdir 删除 cgroup 时,就会调用
相应的函数指针指向的函数。比如:使用 mkdir 创建 cgroup 时,会调用 cgroup_mkdir,
然后在 cgroup_mkdir 中再调用具体实现的 cgroup_create 函数。
Cgroup 文件操作定义:

```
static const struct file_operations cgroup_file_operations = {
.read = cgroup_file_read,
.write = cgroup_file_write,
.llseek = generic_file_llseek,
.open = cgroup_file_open,
.release = cgroup_file_release,
};
```

在 cgroup 文件系统中,对目录下的控制文件进行操作时,会调用该结构体中指针指向的函
数。比如:对文件进行读操作时,会调用 cgroup_file_read,在 cgroup_file_read 中,
会根据需要调用该文件对应的 cftype 结构体定义的对应读函数。</br>
我们再来看 cgroup 文件系统中的 cgroups 控制文件。 Cgroups 定义一个 cftype 的结
构体来管理控制文件。下面我们来看 cftype 的定义:</br>

```
struct cftype {
char name[MAX_CFTYPE_NAME];
int private; /*
mode_t mode;
size_t max_write_len;
int (*open)(struct inode *inode, struct file *file);
ssize_t (*read)(struct cgroup *cgrp, struct cftype *cft,
struct file *file,
char __user *buf, size_t nbytes, loff_t *ppos);
u64 (*read_u64)(struct cgroup *cgrp, struct cftype *cft);
s64 (*read_s64)(struct cgroup *cgrp, struct cftype *cft);
int (*read_map)(struct cgroup *cont, struct cftype *cft,
struct cgroup_map_cb *cb);
int (*read_seq_string)(struct cgroup *cont, struct cftype *cft,
struct seq_file *m);
ssize_t (*write)(struct cgroup *cgrp, struct cftype *cft,
struct file *file,
const char __user *buf, size_t nbytes, loff_t *ppos);
int (*write_u64)(struct cgroup *cgrp, struct cftype *cft, u64 val);
int (*write_s64)(struct cgroup *cgrp, struct cftype *cft, s64 val);
int (*write_string)(struct cgroup *cgrp, struct cftype *cft,
const char *buffer);
int (*trigger)(struct cgroup *cgrp, unsigned int event);
int (*release)(struct inode *inode, struct file *file);
int (*register_event)(struct cgroup *cgrp, struct cftype *cft,
struct eventfd_ctx *eventfd, const char *args); /*
void (*unregister_event)(struct cgroup *cgrp, struct cftype *cft,
struct eventfd_ctx *eventfd);
};

```
cftype 中除了定义文件的名字和相关权限标记外,主要是定义了对文件进行操作的函数指
针。不同的文件可以有不同的操作,对文件进行操作时,相关函数指针指向的函数会被调用。</br>
综合上面的分析,cgroups 通过实现 cgroup 文件系统来为用户提供管理 cgroup 的工
具,而 cgroup 文件系统是基于 Linux VFS 实现的。相应地,cgroups 为控制文件定义了
相应的数据结构 cftype,对其操作由 cgroup 文件系统定义的通过操作捕获,再调用 cftype
定义的具体实现。

#### mount介绍

更详细信息参考：
linux内核mount系统调用源码分析http://blog.csdn.net/wugj03/article/details/41958029/ </br>
linux系统调用mount全过程分析http://blog.csdn.net/skyflying2012/article/details/9748133 </br>
在系统启动时，mount需要的CGroup子系统：</br>

```
mount cgroup none /dev/cpuctl cpu
```
在用户空间将mount命令转换成系统调用sys_mount：</br>

```
asmlinkage long sys_mount(char __user *dev_name, char __user *dir_name,char __user *type, unsigned long flags,void __user *data);

```

从sys_mount到具体文件系统的.mount调用流程如下：</br>

```
sys_mount(fs/namespace.c) 
  -->do_mount(kernel_dev, dir_name, kernel_type, flags, (void *)data_pate)   
    -->do_new_mount (&path, type_page, flags, mnt_flags,dev_name, data_page)     
      --> vfs_kern_mount(type, flags, name, data)       
        --> mount_fs(type, flags, name, data)         
          --> type->mount(type, flags, name, data)           
            --> cgroup_mount(fs_type, flags, unused_dev_name, data)
```

#### cgroup重要结构体

**ok，再温习下概念**</br>
我们把每种**资源**叫做**子系统**，比如CPU子系统，内存子系统。为什么叫做子系统呢，因为它是从整个操作系统的资源衍生出来的。然后我们创建**一种虚拟的节点**，叫做**cgroup**，然后这个虚拟节点可以扩展，以**树形**的结构，有root节点，和子节点。这个父节点和各个子节点就形成了**层级**（hierarchiy）。每个层级都可以附带**继承**一个或者多个**子系统**，就意味着，**我们把资源按照分割到多个层级系统中，层级系统中的每个节点对这个资源的占比各有不同**。</br>

**css_set**</br>
下面我们想法子把进程分组，**进程分组的逻辑** 叫做css_set。这里的css是cgroup_subsys_state的缩写。所以**css_set和进程的关系是一对多**的关系。另外，在cgroup眼中，进程请不要叫做进程，叫做task。这个可能是为了和内核中进程的名词区分开吧。

**css_set & cgroup & subsystem 关系**</br>
进程分组css_set，不同层级中的节点cgroup也都有了。那么，就要把节点cgroup和层级进行关联，和数据库中关系表一样。这个事一个多对多的关系。为什么呢？首先，**一个节点可以隶属于多个css_set，这就代表着这批css_set中的进程都拥有这个cgroup所代表的资源**。其次，**一个css_set需要多个cgroup。因为一个层级的cgroup只代表一种或者几种资源，而一般进程是需要多种资源的集合体**。</br>

#### 重要结构体 <base on 4.8.1>

因为Atom里面对代码中注释的颜色 我还不会加，先把这篇博客地址post这里:</br>
https://www.cnblogs.com/muahao/p/10280998.html</br>

**task_struct**</br>
```
#ifdef CONFIG_CGROUPS
     // 设置这个进程属于哪个css_set
    struct css_set __rcu *cgroups;
     // cg_list是用于将所有同属于一个css_set的task连成一起
    struct list_head cg_list;
#endif

```
**css_set**</br>
include/linux/cgroup-defs.h</br>
```
struct css_set {
    // 引用计数，gc使用，如果子系统有引用到这个css_set,则计数＋1
    atomic_t refcount;

     // TODO: 列出有相同hash值的cgroup（还不清楚为什么）
    struct hlist_node hlist;

     // 将所有的task连起来。mg_tasks代表迁移的任务
    struct list_head tasks;
    struct list_head mg_tasks;
     // 将这个css_set对应的cgroup连起来
    struct list_head cgrp_links;

    // 默认连接的cgroup
    struct cgroup *dfl_cgrp;

     // 包含一系列的css(cgroup_subsys_state)，css就是子系统，这个就代表了css_set和子系统的多对多的其中一面
    struct cgroup_subsys_state *subsys[CGROUP_SUBSYS_COUNT];

    // 内存迁移的时候产生的系列数据
    struct list_head mg_preload_node;
    struct list_head mg_node;
    struct cgroup *mg_src_cgrp;
    struct cgroup *mg_dst_cgrp;
    struct css_set *mg_dst_cset;

    // 把->subsys[ssid]->cgroup->e_csets[ssid]结构展平放在这里，提高迭代效率
    struct list_head e_cset_node[CGROUP_SUBSYS_COUNT];

     // 所有迭代任务的列表，这个补丁参考:https://patchwork.kernel.org/patch/7368941/
    struct list_head task_iters;

    // 这个css_set是否已经无效了
    bool dead;

    // rcu锁所需要的callback等信息
    struct rcu_head rcu_head;
};

```
**Q**:</br>// 将这个css_set对应的cgroup连起来</br>
    struct list_head cgrp_links;</br>
因为css_set会对应多个不同的cgroup，因为每个cgroup节点代表不同的资源（susbsystem）,那么不同的资源节点即cgroup是一种什么样的组织关系呢？</br>

**Key Point**:</br>
// 包含一系列的css(cgroup_subsys_state)，css就是子系统，这个就代表了css_set和子系统的多对多的其中一面</br>
   struct cgroup_subsys_state</br> * subsys[CGROUP_SUBSYS_COUNT];</br>

这里插一句有关rcu的知识：</br>

当所有的CPU都通过静止状态之后，call_rcu()接受rcu_head描述符（通常嵌在要被释放的数据结构中）的地址和将要调用的回调函数的地址(* func)作为参数。一旦回调函数被执行，它通常释放数据结构的旧副本。
struct rcu_head {
    struct rcu_head * next;
    void (* func)(struct rcu_head * head);
};

函数call_rcu()把回调函数和其参数的地址存放在rcu_head描述符中，然后把描述符通过next字段插入回调函数的每CPU（per-CPU）链表中。内核每经过一个时钟滴答就周期性地检查本地CPU是否经过了一个静止状态，即上述三种情况。如果所有CPU都经过了静止状态，本地tasklet的描述符存放在每CPU变量rcu_tasklet中就执行每CPU链表中的所有回调函数，释放数据结构的旧副本。

```
struct callback_head {
    struct callback_head *next;
    void (*func)(struct callback_head *head);
} __attribute__((aligned(sizeof(void *))));
#define rcu_head callback_head

```


**cgroup_subsys_state**</br>
**cgroup_subsys**</br>
这个结构最重要的就是存储的进程与特定子系统相关的信息。通过它，可以将task_struct和cgroup连接起来了：**task_struct->css_set->cgroup_subsys_state->cgroup**</br>

```
struct cgroup_subsys_state {
    // 对应的cgroup
    struct cgroup *cgroup;

    // 子系统
    struct cgroup_subsys *ss;

    // 带cpu信息的引用计数（不大理解）
    struct percpu_ref refcnt;

    // 父css
    struct cgroup_subsys_state *parent;

    // 兄弟和孩子链表串
    struct list_head sibling;
    struct list_head children;

    // css的唯一id
    int id;

     // 可设置的flag有：CSS_NO_REF/CSS_ONLINE/CSS_RELEASED/CSS_VISIBLE
    unsigned int flags;

    // 为了保证遍历的顺序性，设置遍历按照这个字段的升序走
    u64 serial_nr;

    // 计数，计算本身css和子css的活跃数，当这个数大于1，说明还有有效子css
    atomic_t online_cnt;

    // TODO: 带cpu信息的引用计数使用的rcu锁（不大理解）
    struct rcu_head rcu_head;
    struct work_struct destroy_work;
};

```
**NOTE:**</br>
为什么需要task_struct->css_set->cgroup_subsys_state.</br>
就是为了表达css_set的task组对应的资源，而资源需要通过cgroup虚拟节点来表达。</br>

**NOTE:**</br>
这里再强调一下重要的两颗树：</br>
struct list_head sibling，children; ===> 用来表示对某一种资源的分配形式，比如某个节点是20%，另一个节点是30%</br>

而cgroup中的struct cgroup_root；====> 用来表达对一族task所拥有一个或某几个资源的分配情况</br>

而css_set中的 struct list_head tasks;====>用来表达一组task所应对的cgroup虚拟节点，这个节点中拥有一个或几个资源，同时不同的cgroup节点表达了，这组task所拥有的所有的资源。</br>


cgroup_subsys结构体在include/linux/cgroup-defs.h里面</br>

```
struct cgroup_subsys{  

     /* 下面的是函数指针，定义了子系统对css_set结构的系列操作 */
    struct cgroup_subsys_state *(*css_alloc)(struct cgroup_subsys_state *parent_css);
    int (*css_online)(struct cgroup_subsys_state *css);
    void (*css_offline)(struct cgroup_subsys_state *css);
    void (*css_released)(struct cgroup_subsys_state *css);
    void (*css_free)(struct cgroup_subsys_state *css);
    void (*css_reset)(struct cgroup_subsys_state *css);

     // 这些函数指针表示了对子系统对进程task的一系列操作
    int (*can_attach)(struct cgroup_taskset *tset);
    void (*cancel_attach)(struct cgroup_taskset *tset);
    void (*attach)(struct cgroup_taskset *tset);
    void (*post_attach)(void);
    int (*can_fork)(struct task_struct *task);
    void (*cancel_fork)(struct task_struct *task);
    void (*fork)(struct task_struct *task);
    void (*exit)(struct task_struct *task);
    void (*free)(struct task_struct *task);

    void (*bind)(struct cgroup_subsys_state *root_css);

     // 是否在前期初始化了
    bool early_init:1;

     // 如果设置了true，那么在cgroup.controllers和cgroup.subtree_control就不会显示, TODO:
    bool implicit_on_dfl:1;

     // 如果设置为false，则子cgroup会继承父cgroup的子系统资源，否则不继承或者只继承一半
     // 但是现在，我们规定，不允许一个cgroup有不可继承子系统仍然可以衍生出cgroup。如果做类似操作，我们会根据
     // warned_broken_hierarch出现错误提示。
    bool broken_hierarchy:1;
    bool warned_broken_hierarchy:1;

    int id;
    const char *name;

     // 如果子cgroup的结构继承子系统的时候没有设置name，就会沿用父系统的子系统名字，所以这里存的就是父cgroup的子系统名字
    const char *legacy_name;

    struct cgroup_root *root;  // 这个就是子系统指向的层级中的root的cgroup

    struct idr css_idr; // 对应的css的idr

     // 对应的文件系统相关信息
    struct list_head cfts;
    struct cftype *dfl_cftypes;    /* 默认的文件系统 */
    struct cftype *legacy_cftypes;    /* 继承的文件系统 */

     // 有的子系统是依赖其他子系统的，这里是一个掩码来表示这个子系统依赖哪些子系统
    unsigned int depends_on;
};

```
这里的结构体再查看时，一定要对照的本文第一个图，可以帮助理清所有的关系。</br>

**Key Point**:</br>
struct cgroup_root * root;  // 这个就是子系统指向的层级中的root的cgroup</br>

这个root是指向本层级中的root ====> 这个是到目前为止我的理解，还需要进一步查看代码进行验证。</br>

**Key Point**:</br>
这里特别说一下cftype。它是cgroup_filesystem_type的缩写。这个要从我们的linux虚拟文件系统说起（VFS）。VFS封装了标准文件的所有系统调用。那么我们使用cgroup，也抽象出了一个文件系统，自然也需要实现这个VFS。实现这个VFS就是使用这个cftype结构。

这里说一下idr。这个是linux的整数id管理机制。你可以把它看成一个map，这个map是把id和制定指针关联在一起的机制。它的原理是使用基数树。一个结构存储了一个idr，就能很方便根据id找出这个id对应的结构的地址了。http://blog.csdn.net/dlutbrucezhang/article/details/10103371 </br>

**Q:**</br>
什么是基数树？===> 这个问题问baogen吧，他研究过~~^_ ^ </br>


**cgroup**</br>
cgroup结构也在相同文件，但是**cgroup_root**和**子节点cgroup**是使用两个不同结构表示的。</br>
```
struct cgroup {
    // cgroup所在css
    struct cgroup_subsys_state self;

    //cgroup的标志
    unsigned long flags;

    int id;

    // 这个cgroup所在层级中，当前cgroup的深度
    int level;

    // 每当有个非空的css_set和这个cgroup关联的时候，就增加计数1
    int populated_cnt;

    struct kernfs_node *kn;        /* cgroup kernfs entry */
    struct cgroup_file procs_file;    /* handle for "cgroup.procs" */
    struct cgroup_file events_file;    /* handle for "cgroup.events" */

    // TODO: 不理解
    u16 subtree_control;
    u16 subtree_ss_mask;
    u16 old_subtree_control;
    u16 old_subtree_ss_mask;

    // 一个cgroup属于多个css，这里就是保存了cgroup和css直接多对多关系的另一半
    struct cgroup_subsys_state __rcu *subsys[CGROUP_SUBSYS_COUNT];

     // 根cgroup
    struct cgroup_root *root;

    // 相同css_set的cgroup链表
    struct list_head cset_links;

    // 这个cgroup使用的所有子系统的每个链表
    struct list_head e_csets[CGROUP_SUBSYS_COUNT];

    // TODO: 不理解
    struct list_head pidlists;
    struct mutex pidlist_mutex;

    // 用来保存下线task
    wait_queue_head_t offline_waitq;

    // TODO: 用来保存释放任务？(不理解)
    struct work_struct release_agent_work;

    // 保存每个level的祖先
    int ancestor_ids[];
};

```
这里看到一个新的结构，wait_queue_head_t，这个结构是用来将一个资源挂在等待队列中，具体参考：http://www.cnblogs.com/lubiao/p/4858086.html

**Key Point**:</br>
// 一个cgroup属于多个css，这里就是保存了cgroup和css直接多对多关系的另一半</br>
```
struct cgroup_subsys_state __rcu  *subsys[CGROUP_SUBSYS_COUNT];</br>

```

// 相同css_set的cgroup链表 </br>
struct list_head cset_links;</br>

// 这个cgroup使用的所有子系统的每个链表</br>
struct list_head e_csets[CGROUP_SUBSYS_COUNT];</br>

**NOTE:**</br>
1. 在cgroup中，有以cgroup_root为跟的一颗树。
2. 有具有相同css_set的所有cgroup的链表。
3. 有本cgroup所包含的css的资源。
4. 本cgroup所使用的所有的子系统的每个链表。

这些cgroup中的树和链表，可以对照着cgroup_subsys_state所拥有的树和链表来对照着看，用于理清期间的关系。</br>

还有一个结构是**cgroup_root**</br>
```
struct cgroup_root {
     //cgroup文件系统的超级块
    struct kernfs_root *kf_root;

    // 子系统掩码
    unsigned int subsys_mask;

    // 层级的id
    int hierarchy_id;

    // 根部的cgroup，这里面就有下级cgroup
    struct cgroup cgrp;

    // 相等于cgrp->ancester_ids[0]
    int cgrp_ancestor_id_storage;

    // 这个root层级下的cgroup数，初始化的时候为1
    atomic_t nr_cgrps;

    // 串起所有的cgroup_root
    struct list_head root_list;

    unsigned int flags;

    // TODO: 不清楚
    struct idr cgroup_idr;

    /* The path to use for release notifications. */
    // 我们在后面再来详细分析
    char release_agent_path[PATH_MAX];

    // 这个层级的名称，有可能为空
    char name[MAX_CGROUP_ROOT_NAMELEN];
};

```

**NOTE:** </br>
其实,struct cgroupfs_root和struct cgroup就是表示了一种空间层次关系,它就对应着挂着点下面的文件示图.</br>

-------

### cgroup的函数
#### cgroup初始化

##### 总览

CGoup核心主要创建一系列sysfs文件，用户空间可以通过这些节点控制CGroup各子系统行为。各子系统模块根据参数，在执行过程中或调度进程道不同CPU上，或控制CPU占用时间，或控制IO带宽等等。另，在每个进程的proc文件系统中都有一个cgroup，显示该进程对应的CGroup各子系统信息。

如果CGroup需要early_init，start_kernel调用cgroup_init_early在系统启动时进行CGroup初始化。

CGroup的起点是start_kernel->cgroup_init，进入CGroup的初始化，主要注册cgroup文件系统和创建、proc文件，初始化不需要early_init的子系统。

Cgroup的初始化包括两个部份.即cgroup_init_early()和cgroup_init().分别表示在系统初始时的初始化和系统初始化完成时的初始化.分为这两个部份是因为有些subsys是要在系统刚启动的时候就必须要初始化的.

**cgroup_init_early**</br>
本函数主要完成:</br>
1. 初始化cgroup_root.
2. 初始化全局量init_css_set.
3. 对一些需要在系统启动时初始化的subsys进行初始化.

```
int __init cgroup_init_early(void)
{
    // 初始化cgroup_root，就是一个cgroup_root的结构
    init_cgroup_root(&cgrp_dfl_root, &opts);
    cgrp_dfl_root.cgrp.self.flags |= CSS_NO_REF;

    //初始化全局量init_css_set
    RCU_INIT_POINTER(init_task.cgroups, &init_css_set);

    //对一些需要在系统启动时初始化的subsys进行初始化
    for_each_subsys(ss, i) {
        WARN(!ss->css_alloc || !ss->css_free || ss->name || ss->id,
             "invalid cgroup_subsys %d:%s css_alloc=%p css_free=%p id:name=%d:%s\n",
             i, cgroup_subsys_name[i], ss->css_alloc, ss->css_free,
             ss->id, ss->name);
        WARN(strlen(cgroup_subsys_name[i]) > MAX_CGROUP_TYPE_NAMELEN,
             "cgroup_subsys_name %s too long\n", cgroup_subsys_name[i]);

        ss->id = i;
        ss->name = cgroup_subsys_name[i];
        if (!ss->legacy_name)
            ss->legacy_name = cgroup_subsys_name[i];

        //需要在early init初始化的subsystem，这里就进行初始化
        if (ss->early_init)
            cgroup_init_subsys(ss, true);
    }
    return 0;
}

```
这个函数初始化的cgroup_root是一个全局的变量。定义在kernel/cgroup.c中。
```
struct cgroup_root cgrp_dfl_root;
EXPORT_SYMBOL_GPL(cgrp_dfl_root);
```

**init_cgroup_root**</br>

它先初始化root中的几条链表.因为root中有一个top_cgroup，即(root->cgrp).因此将root->nr_cgrps置为1.然后,对root->cgrp进行初始化.使root->cgrp.root指向root. root->cgrp.top_cgroup指向它的本身.因为root->cgrp就是目录下的第一个cgroup.
最后在init_cgroup_housekeeping()初始化cgroup的链表和读写锁.
```
static void init_cgroup_root(struct cgroup_root *root,
			     struct cgroup_sb_opts *opts)
{
	struct cgroup *cgrp = &root->cgrp;

	INIT_LIST_HEAD(&root->root_list);
	atomic_set(&root->nr_cgrps, 1);
	cgrp->root = root;

  //初始化cgroup的链表和读写锁
	init_cgroup_housekeeping(cgrp);
	idr_init(&root->cgroup_idr);

	root->flags = opts->flags;
	if (opts->release_agent)
		strcpy(root->release_agent_path, opts->release_agent);
	if (opts->name)
		strcpy(root->name, opts->name);
	if (opts->cpuset_clone_children)
		set_bit(CGRP_CPUSET_CLONE_CHILDREN, &root->cgrp.flags);
}

```

**cgroup_init_subsys**</br>
在这个函数中:
1. 将每个要注册的subsys->root都指向rootnode.
2. 调用ss->css_alloc()生成一个cgroup_subsys_state.
3. 调用init_and_link_css()将dummytop.subsys[i]设置成ss->create()生成的cgroup_subsys_state
4. 更新init_css_set->subsys()对应项的值.
5. 将ss->active设为1.表示它已经初始化了.

```
static void __init cgroup_init_subsys(struct cgroup_subsys *ss, bool early)
{
	struct cgroup_subsys_state *css;

	pr_debug("Initializing cgroup subsys %s\n", ss->name);

	mutex_lock(&cgroup_mutex);

	idr_init(&ss->css_idr);
	INIT_LIST_HEAD(&ss->cfts);

	/* Create the root cgroup state for this subsystem */
	ss->root = &cgrp_dfl_root;
	css = ss->css_alloc(cgroup_css(&cgrp_dfl_root.cgrp, ss));
	/* We don't handle early failures gracefully */
	BUG_ON(IS_ERR(css));
	init_and_link_css(css, ss, &cgrp_dfl_root.cgrp);

	/*
	 * Root csses are never destroyed and we can't initialize
	 * percpu_ref during early init.  Disable refcnting.
	 */
	css->flags |= CSS_NO_REF;

	if (early) {
		/* allocation can't be done safely during early init */
		css->id = 1;
	} else {
		css->id = cgroup_idr_alloc(&ss->css_idr, css, 1, 2, GFP_KERNEL);
		BUG_ON(css->id < 0);
	}

	/* Update the init_css_set to contain a subsys
	 * pointer to this state - since the subsystem is
	 * newly registered, all tasks and hence the
	 * init_css_set is in the subsystem's root cgroup. */
	init_css_set.subsys[ss->id] = css;

	have_fork_callback |= (bool)ss->fork << ss->id;
	have_exit_callback |= (bool)ss->exit << ss->id;
	have_free_callback |= (bool)ss->free << ss->id;
	have_canfork_callback |= (bool)ss->can_fork << ss->id;

	/* At system boot, before all subsystems have been
	 * registered, no tasks have been forked, so we don't
	 * need to invoke fork callbacks here. */
	BUG_ON(!list_empty(&init_task.tasks));

	BUG_ON(online_css(css));

	mutex_unlock(&cgroup_mutex);
}

```

**cgroup_init()**</br>
本函数在early cgroup初始化之后，cgroup_init()是cgroup的第二阶段的初始化。

这个函数比较简单.首先.它将剩余的subsys初始化.然后将init_css_set添加进哈希数组css_set_table[ ]中.在上面的代码中css_set_hash()是css_set_table的哈希函数.它是css_set->subsys为哈希键值,到css_set_table[ ]中找到对应项.然后调用hlist_add_head()将init_css_set添加到冲突项中.
然后,注册了cgroup文件系统.这个文件系统也是我们在用户空间使用cgroup时必须挂载的.
最后,在proc的根目录下创建了一个名为cgroups的文件.用来从用户空间观察cgroup的状态.</br>

```
/**
 * cgroup_init - cgroup initialization
 *
 * Register cgroup filesystem and /proc file, and initialize
 * any subsystems that didn't request early init.
 */
int __init cgroup_init(void)
{
	struct cgroup_subsys *ss;
	int ssid;

	BUILD_BUG_ON(CGROUP_SUBSYS_COUNT > 16);
	BUG_ON(percpu_init_rwsem(&cgroup_threadgroup_rwsem));
	BUG_ON(cgroup_init_cftypes(NULL, cgroup_dfl_base_files));
	BUG_ON(cgroup_init_cftypes(NULL, cgroup_legacy_base_files));

	get_user_ns(init_cgroup_ns.user_ns);

	mutex_lock(&cgroup_mutex);

	/*
	 * Add init_css_set to the hash table so that dfl_root can link to
	 * it during init.
	 */
	hash_add(css_set_table, &init_css_set.hlist,
		 css_set_hash(init_css_set.subsys));

	BUG_ON(cgroup_setup_root(&cgrp_dfl_root, 0));

	mutex_unlock(&cgroup_mutex);

  //将剩下的(不需要在系统启动时初始化的subsys)的subsys进行初始化
	for_each_subsys(ss, ssid) {
		if (ss->early_init) {
			struct cgroup_subsys_state *css =
				init_css_set.subsys[ss->id];

			css->id = cgroup_idr_alloc(&ss->css_idr, css, 1, 2,
						   GFP_KERNEL);
			BUG_ON(css->id < 0);
		} else {
			cgroup_init_subsys(ss, false);
		}

		list_add_tail(&init_css_set.e_cset_node[ssid],
			      &cgrp_dfl_root.cgrp.e_csets[ssid]);

		/*
		 * Setting dfl_root subsys_mask needs to consider the
		 * disabled flag and cftype registration needs kmalloc,
		 * both of which aren't available during early_init.
		 */
		if (cgroup_disable_mask & (1 << ssid)) {
			static_branch_disable(cgroup_subsys_enabled_key[ssid]);
			printk(KERN_INFO "Disabling %s control group subsystem\n",
			       ss->name);
			continue;
		}

		if (cgroup_ssid_no_v1(ssid))
			printk(KERN_INFO "Disabling %s control group subsystem in v1 mounts\n",
			       ss->name);

		cgrp_dfl_root.subsys_mask |= 1 << ss->id;

		if (ss->implicit_on_dfl)
			cgrp_dfl_implicit_ss_mask |= 1 << ss->id;
		else if (!ss->dfl_cftypes)
			cgrp_dfl_inhibit_ss_mask |= 1 << ss->id;

		if (ss->dfl_cftypes == ss->legacy_cftypes) {
			WARN_ON(cgroup_add_cftypes(ss, ss->dfl_cftypes));
		} else {
			WARN_ON(cgroup_add_dfl_cftypes(ss, ss->dfl_cftypes));
			WARN_ON(cgroup_add_legacy_cftypes(ss, ss->legacy_cftypes));
		}

		if (ss->bind)
			ss->bind(init_css_set.subsys[ssid]);
	}

	/* init_css_set.subsys[] has been updated, re-hash */
	hash_del(&init_css_set.hlist);
	hash_add(css_set_table, &init_css_set.hlist,
		 css_set_hash(init_css_set.subsys));

	WARN_ON(sysfs_create_mount_point(fs_kobj, "cgroup"));

  //注册cgroup文件系统
	WARN_ON(register_filesystem(&cgroup_fs_type));
	WARN_ON(register_filesystem(&cgroup2_fs_type));

  //在proc文件系统的根目录下创建一个名为cgroups的文件
	WARN_ON(!proc_create("cgroups", 0, NULL, &proc_cgroupstats_operations));

	return 0;
}

```

#### 阶段性总结：

经过cgroup的两个阶段的初始化, init_css_set, rootnode,subsys已经都初始化完成.表面上看起来它们很复杂,其实,它们只是表示cgroup的初始化状态而已.例如,如果subsys->root等于rootnode,那表示subsys没有被其它的cgroup所使用.</br>


##### 基本操作
Cgroup初始化
cgroup_init

       kernel启动时，调用cgroup_init，完成cgroup初始化：


    cgroup_init_subsys()

       /linux/cgroup_subsys.h定义了kernel支持的所有子系统。当前系统支持的子系统：

       Cpuset、debug、cpu_cgroup、cpuacct、mem_cgroup、devices、freezer、net_cls、

       Blkio、perf、net_prio、hugetlb



       Cgroup_init_subsys完成一个子系统的所有初始化工作，即：

       子系统在初始化时，默认处于cgroup_dummy_root层级；

       创建该子系统的一个初始状态，且隶属于cgroup_dummy_root的root cgroup:cgroup_dummy_top

       将该子系统的初始状态初始化init进程的子系统集合init_css_set；

       总结，在kernel boot过程中，系统支持的所有子系统都处于cgroup_dummy_root层级，并且为cgroup_dummy_top创建每个子系统的初始状态。
      register_filesystem()

       注册文件系统cgroup_fs_type:

static struct file_system_type cgroup_fs_type = {

       .name = "cgroup",

       .mount = cgroup_mount,

       .kill_sb = cgroup_kill_sb,

};
      proc_create()

         为cgroup机制建立proc接口

static const struct file_operations proc_cgroupstats_operations = {

              .open = cgroupstats_open,

              .read = seq_read,

              .llseek = seq_lseek,

              .release = single_release,

};




创建层级mount:

       mount -t cgroup -o subsystems name /cgroup/name



       static struct dentry *cgroup_mount(struct file_system_type *fs_type,

                      int flags, const char *unused_dev_name,

                      void *data)
Cgroup_mount


parse_cgroupfs_options

       /* First find the desired set of subsystems */

static int parse_cgroupfs_options(char *data, struct cgroup_sb_opts *opts)

       解析包含预挂载子系统名称的data，若没有设置none，也没有制定子系统名称，则在该层级启动所有子系统。即设置subsys_mask的一个过程，为后续流程指定此次挂载了哪些子系统。
cgroup_root_from_opts

       分配一个新的层级，除非重用已经存在的层级。此外，初始化该层级的顶层cgroup。

       sget()在当前文件系统中寻找或者创建一个super block，初始化super block。
      新的层级

       每次创建新的层级，分配cgroupfs_root时会静态分配root_cgroup，在初始化cgroupfs_root时，对root_cgroup进行初始化。
cgroup_init_root_id

       初始化hierarchy_id
cgroup_get_rootdir

       构建层级的super block, dentry, inode之间的关系。相关回调在此处初始化，之后就可以从文件系统层面操作层级，及相关的cgroup，包括它们所相关的属性文件。
cgroup_addrm_files

       向该层级的顶层cgroup添加cgroup_base_files 通用cftype文件，即一些有关cgroup的属性文件（这里不区分不同的子系统）
cgroup_populate_dir

       遍历该层级存在的子系统，为每一个子系统建立自己的属性文件。
rebind_subsystems

       默认子系统挂载在cgroup_dummy_root层级，现将子系统挂载在当前创建的层级；此外， 有些子系统可能退出该层级，则将这些退出的层级挂载到cgroup_dummy_root层级（考虑通过remount命令中心挂载层级的子系统，到达卸载部分子系统的意图）。Cgroup_dummy_root负责回收其他层级不在使用的子系统。

       在初始化时，cgroup_dummy_top这个顶层cgroup存储了子系统的初始化状态，利用该状态初始化该层级的顶层cgroup相关的子系统。

       cgroup_populate_dir为该层级的顶层cgroup创建每个子系统专属的cftype文件。
cgroup_roots

       最后，将创建的层级添加到系统cgroup_roots
cgroup群组的建立

cgcreate -t uid:gid -a uid:gid -g subsystems:path

static int cgroup_mkdir(struct inode *dir, struct dentry *dentry, umode_t mode)
cgroup_mkdir
cgroup_create

       文件系统通过目录的dentry寻找parent cgroup，进而在parent cgroup基础上进一步创建child cgroup，每个子系统挂载的层级会有初始的root cgroup。

       初始化child cgroup，child cgroup、parent cgroup、dentry之间的关系：

       遍历该层级的每一个子系统，创建子系统状态，根据parent cgroup的子系统状态初始化新建的cgroup对应的状态。
cgroup_create_file

       cgroup_create传入的child dentry参数，则需要给dentry分配inode，根据参数mode是否是目录，确定inode的call back。
online_css

       cgroup创建完成，构建了cgroup对应的各个子系统状态后，即需要通知每一个子系统。将该子系统状态指针赋给cgroup中相应子系统指针。

       即完成cgroup与子系统状态之间的衔接。
cgroup_addrm_files

       构建属于这个cgroup的cgroup_base_files，此处不区分各个子系统。
```
    static struct cftype cgroup_base_files[] = {  
        {  
            .name = "cgroup.procs",  
            .open = cgroup_procs_open,  
            .write_u64 = cgroup_procs_write,  
            .release = cgroup_pidlist_release,  
            .mode = S_IRUGO | S_IWUSR,  
        },  
        {  
             .name = "cgroup.event_control",  
             .write_string = cgroup_write_event_control,  
             .mode = S_IWUGO,  
         },  
         {  
             .name = "cgroup.clone_children",  
             .flags = CFTYPE_INSANE,  
             .read_u64 = cgroup_clone_children_read,  
             .write_u64 = cgroup_clone_children_write,  
         },  
         {  
             .name = "cgroup.sane_behavior",  
             .flags = CFTYPE_ONLY_ON_ROOT,  
             .read_seq_string = cgroup_sane_behavior_show,  
         },  

         /*
          * Historical crazy stuff.  These don't have "cgroup."  prefix and
          * don't exist if sane_behavior.  If you're depending on these, be
          * prepared to be burned.
          */  
         {  
             .name = "tasks",  
             .flags = CFTYPE_INSANE,     /* use "procs" instead */  
             .open = cgroup_tasks_open,  
             .write_u64 = cgroup_tasks_write,  
             .release = cgroup_pidlist_release,  
             .mode = S_IRUGO | S_IWUSR,  
         },  
         {  
             .name = "notify_on_release",  
             .flags = CFTYPE_INSANE,  
             .read_u64 = cgroup_read_notify_on_release,  
             .write_u64 = cgroup_write_notify_on_release,  
         },  
         {  
             .name = "release_agent",  
             .flags = CFTYPE_INSANE | CFTYPE_ONLY_ON_ROOT,  
             .read_seq_string = cgroup_release_agent_show,  
             .write_string = cgroup_release_agent_write,  
             .max_write_len = PATH_MAX,  
         },  
         { } /* terminate */  

53. };  
cgroup_populate_dir


```
       构建属于这个cgroup的，且为每一个子系统特有的属性文件或者配置文件。
撤销cgroup组

cgdelete subsystems:path

static int cgroup_rmdir(struct inode *unused_dir, struct dentry *dentry)
cgroup_rmdir

destroy的过程分为两个阶段：

    验证cgroup是否可以被destroyed，移除在用户空间中有关cgroup的相关部分，kill per_cpu refcnts of css’s[Z1]
    调用css_offline，标记cgroup为dead，处理destruction的剩余过程。一旦cgroup的引用为零，则释放cgroup。

两个阶段由cgroup_destroy_locked和cgroup+destrou_css_killed完成
cgroup_destroy_locked

      需要确保cgroup对应的子状态集合为空，即cgroup中没有任何进程，否则无法对cgroup做撤销；

       cgroup的child cgroup是否为dead，不是，则无法对cgroup做撤销；

       遍历cgroup相关子系统，调用kill_css，destroy css and offline css;

       标记cgroup为dead；

       若cgroup没有任何状态，需要在此调用cgroup_destroy_css_killed完成第二阶段任务；

        cgroup_addrm_file和cgroup_d_remove_dir：清除cgroup对应的通用属性和配置文件，移除目录dentry；

       unregister events and notify userspace
cgroup_destroy_css_killed

      只有满足：cgroup中不存在任何子系统状态且cgroup状态为dead，才允许调用cgroup_destroy_css_killed完成第二阶段任务。

      若cgroup_destroy_locked函数处理中，cgroup中不存在任何子系统状态，则直接调用cgroup_destroy_css_killed；

      若子系统的refcnt被证实killed，因此css_tryget()无法成功时，会触发调用cgroup_destroy_css_killed.

       调用此函数前，与cgroup相关的所有css已经offline，主要任务：

               撤销cgroup对应的dentry，释放dentry；

                将cgroup从parent cgroup中脱离；

                通知用户空间。
将进程和线程加入cgroup

cgclassify -g subsystems:path_to_cgroup pidlist

      针对进程和线程，调用不同文件操作函数。

       将进程和线程加入cgroup的实质：将进程id和线程id加入cgroup的相关属性文件，这两个属性文件与子系统和层级无关。

       tasks: 所有附属于这个cgroup的进程ID列表。tasks文件中增加进程ID，表示将进程加入这个cgroup，进程能够使用的资源受cgroup限制。

       cgroup.procs: 所有附属于这个cgroup线程组ID，将TGID写入这个文件后，TGID所在进程包含的所有线程都加入这个cgroup，这些线程受cgroup限制。

       即只将线程组的组ID加入cgrop.procs。
```
struct cgroup_taskset {

       struct task_and_cgroup   single;

       struct flex_array     *tc_array;

       int                 tc_array_len;

       int                 idx;

       struct cgroup          *cur_cgrp;

};


```
      构建具体的cgroup与具体的taskset（一个具体进程）的对应关系。tc_array存储了cgroup中与这个线程组有关的所有线程的相关信息，即task_struct与cgroup对应关系，以便后续线程可以找到对应的cgroup，cgroup通过线程组组长进而找到含有的线程。
attach_task_by_pid

       根据是否是一个线程组，选取线程组的leader，或者一个进程，加入cgroup。将线程组和进程加入cgroup的函数进行了封装。
cgroup_attach_task

       首先，构建为进程和所有线程构建task_and_cgroup，即需要为所有线程和进程寻找一个cgroup。

    检查cgroup中的子系统是否允许这个进程或线程加入这个cgroup；
    若涉及线程的cgroup迁移，需要为线程建立一个css_set；此外针对线程，需要保存线程组的所有线程的信息，构建线程与cgroup关系；
    为所有线程和进程进行cgroup迁移操作；      
    调用cgroup涉及的各个子系统的attach函数，将每个进程或线程加入这些子系统。

进程fork中有关cgroup处理
Cgroup_fork

       即子进程的cgroup继承父进程的cgroup，arch_dup_task_struct函数已将父进程的css_set指针赋给新建的进程，但是此处仅仅是给cgroup指针赋值，担心在子进程创建过程中，父进程的cgroup会做修改，同时将

css_set引用原子增加，子进程可以使用这个子系统状态，不必担心父进程做迁移带来的问题。还有一个地方不太明白，在进程加入一个cgroup中，首先会查看子系统资源限制，在进程fork中，没有做相关检查？

```

void cgroup_fork(struct task_struct *child)
{
    task_lock(current);
    get_css_set(task_css_set(current));
    child->cgroups = current->cgroups;
    task_unlock(current);
    INIT_LIST_HEAD(&child->cg_list);
}


```

cgroup_post_fork

        cgroup_post_fork将子系统状态集合css_set与进程相关联，前提是新建的进程加入系统的进程链表。为什么在这个地方做cgroup_post_fork，我的理解是担心进程创建失败引起的问题，如会给子系统带来扰动。

```
void cgroup_post_fork(struct task_struct *child)
{
    struct cgroup_subsys *ss;
    int i;

    if (use_task_css_set_links) {
        write_lock(&css_set_lock);
        task_lock(child);
        if (list_empty(&child->cg_list))
10.             list_add(&child->cg_list, &task_css_set(child)->tasks);
11.         task_unlock(child);
12.         write_unlock(&css_set_lock);
13.     }
14.  
15.     /*
16.      * Call ss->fork().  This must happen after @child is linked on
17.      * css_set; otherwise, @child might change state between ->fork()
18.      * and addition to css_set.
19.      */
20.     if (need_forkexit_callback) {
21.  
22.         for_each_builtin_subsys(ss, i)
23.             if (ss->fork)
24.                 ss->fork(child);
25.     }


 ```

--------

### 各个子系统:

Please refer to the cgroups introduce.pdf

#### Memory 子系统：

--------

在内存资源管理方面，除了cpuset子系统按内存节点管理的方式外，memory子系统提供了以内存页面为粒度来管理内存的机制。Memory管理内存的方式比较直接，通过memory.limit_in_bytes接口，用户可以设定进程组使用的最大内存上限。另外，每个进程组都会有一个内存使用计数器，用于记录进程组当前的内存使用量。每当系统为进程分配一个内存页面时，会先检查该进程所属进程组的内存使用量是否会超限，如果没有超限，则计数器增加4KB；如果会超限，则先尝试在进程组中回收内存，如果回收内存成功，则检查通过，计数器加4KB，否则，则触发OOM，在进程组中找出一个最应该被杀死的进程，发送SIGKILL信号将进程杀死。</br>
对于匿名页和文件页的统计，memory子系统采用了不同的策略。对于匿名页，由于这些页面都是和进程的地址空间建立了映射关系的，所以在内存分配时，可以很容易找到使用这个匿名页的进程，并将内存使用统计到对应的进程组中；而对于文件页面，即通常所说的pagecache页面，内核何时分配内存是和使用它的进程无关的，对于这种情况，memory子系统采用的策略是，在内核分配pagecache页面时，将这些页面统计到current进程所属的进程组中。</br>
除了对内存做限制外，内存子系统还提供了一个memory.memsw.limit_in_bytes接口，用于控制进程组memory+swap的使用量。当memory.memsw.limit_in_bytes和memory.limit_in_bytes 限制相等时，则当前进程组只能使用memory，而不能使用swap；否则，进程组最多能使用memory.limit_in_bytes限制的memory，当超过时，会触发内存回收，把一部分匿名页替换到swap分区。</br>
除了对内存使用的限制，memory子系统还提供了丰富的统计信息，帮助我们了解当前进程组的内存使用详情，包括匿名页的使用量、pagecache页面的使用量以及进程组使用内存的历史峰值等等。</br>
当前内核版本的memory子系统只能对用户态进程使用的内存进行管理，而对于内核线程使用的内存则没有限制；另外，对于kswapd异步回收内存的机制支持也不够完善，虽然提供了softlimit接口让用户可以设置进程组在异步回收内存的阈值，但是如果系统整体内存使用量偏低，没有触发kswapd开始回收内存，那么这个softlimit设置就没有意义，只能等到进程组内存使用上限达到时触发同步回收才可以将进程组内存使用量降下来。对于这些问题，目前开源社区已经有了初步的解决方案，但是要进入主线估计还得需要时间。</br>

-----------

##### Key Structure (3.10.0)

```

/*
 * The core object. the cgroup that wishes to account for some
 * resource may include this counter into its structures and use
 * the helpers described beyond
 */

struct res_counter {
    /*
     * the current resource consumption level
     */
    unsigned long long usage;
    /*
     * the maximal value of the usage from the counter creation
     */
    unsigned long long max_usage;
    /*
     * the limit that usage cannot exceed
     */
    unsigned long long limit;
    unsigned long long low_wmark_limit;
    unsigned long long high_wmark_limit;
    /*
     * the limit that usage can be exceed
     */
    unsigned long long soft_limit;
    /*
     * the number of unsuccessful attempts to consume the resource
     */
    unsigned long long failcnt;
    /*
     * the lock to protect all of the above.
     * the routines below consider this to be IRQ-safe
     */
    spinlock_t lock;
    /*
     * Parent counter, used for hierarchial resource accounting
     */
    struct res_counter *parent;
};

Accounting

        +--------------------+
        |  mem_cgroup     |
        |  (res_counter)     |
        +--------------------+
         /            ^      \
        /             |       \
           +---------------+  |        +---------------+
           | mm_struct     |  |....    | mm_struct     |
           |               |  |        |               |
           +---------------+  |        +---------------+
                              |
                              + --------------+
                                              |
           +---------------+           +------+--------+
           | page          +---------->  page_cgroup|
           |               |           |               |
           +---------------+           +---------------+

             (Figure 1: Hierarchy of Accounting)


Figure 1 shows the important aspects of the controller

1. Accounting happens per cgroup
2. Each mm_struct knows about which cgroup it belongs to
3. Each page has a pointer to the page_cgroup, which in turn knows the cgroup it belongs to

```
###### page 和 page_cgroup的关系
通过下面的两个函数就能够知道:</br>

根据page得到页帧号pfn，再通过page得到对应node上的page_cgroup的base地址，再用页帧号pfn减去page所在node的起始页帧号得到offset，再用base + offset得到对应的page_cgroup的指针

```
(init/main.c)
main()
{
....
  page_cgroup_init();
....
}

(mm/page_cgroup.c)
void __init page_cgroup_init(void)
{
  unsigned long pfn;
    int nid;

    if (mem_cgroup_disabled())
        return;

    for_each_node_state(nid, N_MEMORY) {
        unsigned long start_pfn, end_pfn;

        start_pfn = node_start_pfn(nid);
        end_pfn = node_end_pfn(nid);
        /*
         * start_pfn and end_pfn may not be aligned to SECTION and the
         * page->flags of out of node pages are not initialized.  So we
         * scan [start_pfn, the biggest section's pfn < end_pfn) here.
         */
        for (pfn = start_pfn;
             pfn < end_pfn;
                     pfn = ALIGN(pfn + 1, PAGES_PER_SECTION)) {

            if (!pfn_valid(pfn))
                continue;
            /*
             * Nodes's pfns can be overlapping.
             * We know some arch can have a nodes layout such as
             * -------------pfn-------------->
             * N0 | N1 | N2 | N0 | N1 | N2|....
             */
            if (pfn_to_nid(pfn) != nid)
                continue;
            if (init_section_page_cgroup(pfn, nid))
                goto oom;
        }
    }
    hotplug_memory_notifier(page_cgroup_callback, 0);
    printk(KERN_INFO "allocated %ld bytes of page_cgroup\n", total_usage);
    printk(KERN_INFO "please try 'cgroup_disable=memory' option if you "
             "don't want memory cgroups\n");
    return;
oom:
    printk(KERN_CRIT "try 'cgroup_disable=memory' boot option\n");
    panic("Out of memory");
}

(mm/page_cgroup.c)
struct page_cgroup *lookup_page_cgroup(struct page *page)
{
    unsigned long pfn = page_to_pfn(page);
    unsigned long offset;
    struct page_cgroup *base;

    base = NODE_DATA(page_to_nid(page))->node_page_cgroup;
#ifdef CONFIG_DEBUG_VM
    /*
     * The sanity checks the page allocator does upon freeing a
     * page can reach here before the page_cgroup arrays are
     * allocated when feeding a range of pages to the allocator
     * for the first time during bootup or memory hotplug.
     */
    if (unlikely(!base))
        return NULL;
#endif
    offset = pfn - NODE_DATA(page_to_nid(page))->node_start_pfn;
    return base + offset;
}
```


![Alt text](/pic/mem_relationship.png)

##### 3.10.0

```



```

###### Key Function:

**Init Function**</br>
这个函数用于初始化一个 res_counter。</br>
```
void res_counter_init(struct res_counter *counter, struct res_counter *parent)
{
    spin_lock_init(&counter->lock);
    counter->limit = RESOURCE_MAX;
    counter->soft_limit = RESOURCE_MAX;
    counter->low_wmark_limit = RESOURCE_MAX;
    counter->high_wmark_limit = RESOURCE_MAX;
    counter->parent = parent;
}

```

**res_counter_charge()**:</br>
当资源将要被分配的 时候，资源就要被记录到相应的res_counter里。这个函数作用就是记录进程组使用的资源。 在这个函数中有:</br>

```
for (c = counter; c != limit; c = c->parent) {
    spin_lock(&c->lock);
    r = res_counter_charge_locked(c, val, force);
    spin_unlock(&c->lock);
    if (r < 0 && !ret) {
        ret = r;
        if (limit_fail_at)
            *limit_fail_at = c;
        if (!force)
            break;
    }
}

```

```
int res_counter_charge(struct res_counter *counter, unsigned long val,
            struct res_counter **limit_fail_at)
{
    return __res_counter_charge_until(counter, val, NULL, limit_fail_at, false);
}

static int __res_counter_charge_until(struct res_counter *counter,
        unsigned long val, struct res_counter *limit,
                struct res_counter **limit_fail_at, bool force)
{
    int ret, r;
    unsigned long flags;
    struct res_counter *c, *u;

    r = ret = 0;
    if (limit_fail_at)
        *limit_fail_at = NULL;
    local_irq_save(flags);
    for (c = counter; c != limit; c = c->parent) {
        spin_lock(&c->lock);
        r = res_counter_charge_locked(c, val, force);
        spin_unlock(&c->lock);
        if (r < 0 && !ret) {
            ret = r;
            if (limit_fail_at)
                *limit_fail_at = c;
            if (!force)
                break;
        }
    }

    if (ret < 0 && !force) {
        for (u = counter; u != c; u = u->parent) {
            spin_lock(&u->lock);
            res_counter_uncharge_locked(u, val);
            spin_unlock(&u->lock);
        }
    }
    local_irq_restore(flags);

    return ret;
}

```

**res_counter_uncharge（）**</br>

```
u64 res_counter_uncharge(struct res_counter *counter, unsigned long val)
{
    return res_counter_uncharge_until(counter, NULL, val);
}

u64 res_counter_uncharge_until(struct res_counter *counter,
                  struct res_counter *top,
                  unsigned long val)
{
    unsigned long flags;
    struct res_counter *c;
    u64 ret = 0;

    local_irq_save(flags);
    for (c = counter; c != top; c = c->parent) {
        u64 r;
        spin_lock(&c->lock);
        r = res_counter_uncharge_locked(c, val);
        if (c == counter)
            ret = r;
        spin_unlock(&c->lock);
    }
    local_irq_restore(flags);
    return ret;
}

```
**Q:**/br
1. 为什么对资源的查找都是从下往上查？如意res_counter处于中间会怎么办？
2. res_counter 和 mem_cgroup 和 page_cgroup 以及 cgroup 之间的关系?

#### Another Key Strucutre:

```
/*
 * The memory controller data structure. The memory controller controls both
 * page cache and RSS per cgroup. We would eventually like to provide
 * statistics based on the statistics developed by Rik Van Riel for clock-pro,
 * to help the administrator determine what knobs to tune.
 *
 * TODO: Add a water mark for the memory controller. Reclaim will begin when
 * we hit the water mark. May be even add a low water mark, such that
 * no reclaim occurs from a cgroup at it's low water mark, this is
 * a feature that will be implemented much later in the future.
 */
struct mem_cgroup {
    struct cgroup_subsys_state css;
    /*
     * the counter to account for memory usage
     */
    struct res_counter res;

    /* vmpressure notifications */
    struct vmpressure vmpressure;

    union {
      /*
       * the counter to account for mem+swap usage.
       */
      struct res_counter memsw;

      /*
       * rcu_freeing is used only when freeing struct mem_cgroup,
       * so put it into a union to avoid wasting more memory.
       * It must be disjoint from the css field.  It could be
       * in a union with the res field, but res plays a much
       * larger part in mem_cgroup life than memsw, and might
       * be of interest, even at time of free, when debugging.
       * So share rcu_head with the less interesting memsw.
       */
      struct rcu_head rcu_freeing;
      /*
       * We also need some space for a worker in deferred freeing.
       * By the time we call it, rcu_freeing is no longer in use.
       */
      struct work_struct work_freeing;
  };

  /*
   * the counter to account for kernel memory usage.
   */
  struct res_counter kmem;
  /*
   * Should the accounting and control be hierarchical, per subtree?
   */
  bool use_hierarchy;
  unsigned long kmem_account_flags; /* See KMEM_ACCOUNTED_*, below */

  bool        oom_lock;
  atomic_t    under_oom;
  atomic_t    oom_wakeups;

  atomic_t    refcnt;

  int swappiness;
  /* OOM-Killer disable */
  int     oom_kill_disable;

  struct vm_dirty_param dirty_param;

  unsigned int    killmode;
  unsigned int    killpriority;
  int             reclaimpriority;

  bool        background_reclaim;
  bool        dirty_throttle;
  bool        enable_priority_reclaim;

  /* set when res.limit == memsw.limit */
  bool        memsw_is_minimum;
  /* protect arrays of thresholds */
  struct mutex thresholds_lock;

  /* thresholds for memory usage. RCU-protected */
  struct mem_cgroup_thresholds thresholds;

  /* thresholds for mem+swap usage. RCU-protected */
  struct mem_cgroup_thresholds memsw_thresholds;

  /* For oom notifier event fd */
  struct list_head oom_notify;

  /*
   * Should we move charges of a task when a task is moved into this
   * mem_cgroup ? And what type of charges should we move ?
   */
  unsigned long   move_charge_at_immigrate;

  atomic_t kswapd_running;
  wait_queue_head_t memcg_kswapd_end;
  struct list_head memcg_kswapd_wait_list;

  /*
   * set > 0 if pages under this cgroup are moving to other cgroup.
   */
  atomic_t    moving_account;
  /* taken only while moving_account > 0 */
  spinlock_t  move_lock;
  /*
   * percpu counter.
   */
  struct mem_cgroup_stat_cpu __percpu *stat;
  /*
   * used when a cpu is offlined or other synchronizations
   * See mem_cgroup_read_stat().
   */
  struct mem_cgroup_stat_cpu nocpu_base;
  spinlock_t pcp_counter_lock;

  u64 high_wmark_distance;
  u64 low_wmark_distance;

  atomic_t    dead_count;
#if defined(CONFIG_MEMCG_KMEM) && defined(CONFIG_INET)
  struct tcp_memcontrol tcp_mem;
#endif
#if defined(CONFIG_MEMCG_KMEM)
  /* analogous to slab_common's slab_caches list. per-memcg */
  struct list_head memcg_slab_caches;
  /* Not a spinlock, we can take a lot of time walking the list */
  struct mutex slab_caches_mutex;
      /* Index in the kmem_cache->memcg_params->memcg_caches array */
  int kmemcg_id;
#endif

  int last_scanned_node;
#if MAX_NUMNODES > 1
  nodemask_t  scan_nodes;
  atomic_t    numainfo_events;
  atomic_t    numainfo_updating;
#endif

  /*
   * Per cgroup active and inactive list, similar to the
   * per zone LRU lists.
   *
   * WARNING: This has to be the last element of the struct. Don't
   * add new fields after this point.
   */
  struct mem_cgroup_lru_info info;
};

```

mem_cgroup中包含了两个res_counter成员，分别用于管理memory资源和memory+swap 资源，如果memsw_is_minimum为true，则res.limit=memsw.limit，即当进程组使用的 内存超过memory的限制时，不能通过swap来缓解. </br>
use_hierarchy则用来标记资源控制和记录时是否是层次性的。</br> oom_kill_disable则表示是否使用oom-killer。</br>
oom_notify指向一个oom notifier event fd链表。</br>

##### Plus, one more key structure:

```
/*
 * Page Cgroup can be considered as an extended mem_map.
 * A page_cgroup page is associated with every page descriptor. The
 * page_cgroup helps us identify information about the cgroup
 * All page cgroups are allocated at boot or memory hotplug event,
 * then the page cgroup for pfn always exists.
 */
struct page_cgroup {
    unsigned long flags;
    struct mem_cgroup *mem_cgroup;
    unsigned short id;
};

```
**Q:**
1. page_cgroup 包含 mem_cgroup。。。。 这是一种神马关系呢？</br>

```
page_cgroup
  -> mem_cgroup
  ->res_counter ->cgroup_subsys_state ->cgroup ->task

```
2. page_cgroup 又是被谁调用呢？ I mean 被哪个结构体包含？如果是page 结构体，它们之间是如何联系起来的呢？</br>
**A:** maybe, 每个 mm_struct知道它属于的进程，进而知道所属的mem_cgroup，而每个page都知道它属于的 page_cgroup，进而也知道所属的mem_cgroup，而内存使用量的计算是按**cgroup**为单位的， 这样以来，内存资源的管理就可以实现了。</br>

如果是这样的话，那么对应的关系是这样？below:</br>
page->mm_struct->task->cgroup->mem_cgroup->page_cgroup</br>
I'm not sure.</br>

-----------

###### 上面说了memcg管理和统计内存，都是以page为最小单位的 那么内核中使用page的地方那么多，我们怎么去一一统计呢

-----------

##### Charge memory的时机：

1. page fault发生时，有两种情况内核需要给进程分配新的页框。一种是进程请求调页 (demand paging)，另一种是copy on write。
2. 内核在handle_pte_fault中进行处理时，还有一种情况是pte存在且页又没有映射文件。
3. 当执行swapoff系统调用(关掉swap空间)，内核也会执行 页面换入操作，因此mem_cgroup_try_charge_swapin也被植入到了相应的函数中.
4. 当 内 核 将 page 加 入 到 page cache 中 时 ， 也 需 要 进 行 charge 操 作。
5. 最后mem_cgroup_prepare_migration是用于处理内存迁移中的charge操作。


Regarding **NO.1~3**, 下面总结的非常好：</br>
内核在handle_pte_fault中进行处理。 其 中 ， do_linear_fault 处 理 pte 不 存 在 且 页 面 线 性 映 射 了 文 件 的 情 况 ， do_anonymous_page处理pte不存在且页面没有映射文件的情况，do_nonlinear_fault 处理pte存在且页面非线性映射文件的情况，do_wp_page则处理copy on write的情况。 其中do_linear_fault和do_nonlinear_fault都会调用__do_fault来处理。Memory子系 统则__do_fault、do_anonymous_page、do_wp_page植入mem_cgroup_newpage_charge 来进行charge操作。</br>

实际上就是将下面函数进行的总结：</br>
```
/*
 * These routines also need to handle stuff like marking pages dirty
 * and/or accessed for architectures that don't do it in hardware (most
 * RISC architectures).  The early dirtying is also good on the i386.
 *
 * There is also a hook called "update_mmu_cache()" that architectures
 * with external mmu caches can use to update those (ie the Sparc or
 * PowerPC hashed page tables that act as extended TLBs).
 *
 * We enter with non-exclusive mmap_sem (to exclude vma changes,
 * but allow concurrent faults), and pte mapped but not yet locked.
 * We return with mmap_sem still held, but pte unmapped and unlocked.
 */
static int handle_pte_fault(struct mm_struct *mm,
             struct vm_area_struct *vma, unsigned long address,
             pte_t *pte, pmd_t *pmd, unsigned int flags)
{
    pte_t entry;
    spinlock_t *ptl;

    entry = *pte;
    if (!pte_present(entry)) {
        if (pte_none(entry)) {
            if (vma->vm_ops)
                return do_linear_fault(mm, vma, address,
                        pte, pmd, flags, entry); //处 理 pte 不 存 在 且 页 面 线 性 映 射 了 文 件 的 情 况
            return do_anonymous_page(mm, vma, address,
                         pte, pmd, flags); //处理pte不存在且页面没有映射文件的情况
        }
        if (pte_file(entry))
            return do_nonlinear_fault(mm, vma, address,
                    pte, pmd, flags, entry); //处理pte存在且页面非线性映射文件的情况
        return do_swap_page(mm, vma, address,
                    pte, pmd, flags, entry); //处理pte存在且页又没有映射文件
    }

    if (pte_numa(entry))
        return do_numa_page(mm, vma, address, entry, pte, pmd);

    ptl = pte_lockptr(mm, pmd);
    spin_lock(ptl);
    if (unlikely(!pte_same(*pte, entry)))
        goto unlock;
    if (flags & FAULT_FLAG_WRITE) {
        if (!pte_write(entry))
            return do_wp_page(mm, vma, address,
                    pte, pmd, ptl, entry); //处理copy on write的情况
        entry = pte_mkdirty(entry);
    }
    entry = pte_mkyoung(entry);
    if (ptep_set_access_flags(vma, address, pte, entry, flags & FAULT_FLAG_WRITE)) {
        update_mmu_cache(vma, address, pte);
    } else {
        /*
         * This is needed only for protection faults but the arch code
         * is not yet telling us if this is a protection fault or not.
         * This still avoids useless tlb flushes for .text page faults
         * with threads.
         */
        if (flags & FAULT_FLAG_WRITE)
            flush_tlb_fix_spurious_fault(vma, address);
    }
unlock:
    pte_unmap_unlock(pte, ptl);
    return 0;
}

```

调用关系:</br>
```
do_linear_fault
do_nonlinear_fault
  -> __do_fault
    -> mem_cgroup_newpage_charge

do_anonymous_page
  -> mem_cgroup_newpage_charge

do_wp_page
  -> mem_cgroup_newpage_charge

do_swap_page
  -> mem_cgroup_try_charge_swapin //处理页面换入时的charge
```

Regarding **NO.4**:</br>
```
(mm/filemap.c)
__add_to_page_cache_locked
  -> mem_cgroup_cache_charge

(mm/shmem.c)
shmem_unuse
shmem_getpage_gfp
  ->mem_cgroup_cache_charge
```

Regarding **NO.5**:</br>
```
__unmap_and_move
migrate_misplaced_transhuge_page
  ->mem_cgroup_prepare_migration

```

------------

###### charge主要在page cache和anon page的几个使用场景发生，另外，还有一个迁移的情况。(uncharge)

------------

##### Uncharge memory 时机：

1. mem_cgroup_uncharge_page用于当匿名页完全unmaped的时候。
2. mem_cgroup_uncharge_cache_page用于page cache从radix-tree删除的时候.
3. mem_cgroup_uncharge_swapcache用于swap cache从radix-tree删除的时候.
4. mem_cgroup_uncharge_swap用于swap_entry的引用数减到0的时候.
5. mem_cgroup_end_migration用于内存迁移结束时相关的uncharge操作。


#### cgroup_subsys

跟其他子系统一样，memory子系统也实现了一个cgroup_subsys。

```
struct cgroup_subsys mem_cgroup_subsys = {
    .name = "memory",
    .subsys_id = mem_cgroup_subsys_id,
    .css_alloc = mem_cgroup_css_alloc,
    .css_online = mem_cgroup_css_online,
    .css_offline = mem_cgroup_css_offline,
    .css_free = mem_cgroup_css_free,
    .can_attach = mem_cgroup_can_attach,
    .cancel_attach = mem_cgroup_cancel_attach,
    .attach = mem_cgroup_move_task,
    .bind = mem_cgroup_bind,
    .base_cftypes = mem_cgroup_files,
    .early_init = 0,
    .use_id = 1,
};

```

Memory子系统中重要的文件有</br>
```
static struct cftype memsw_cgroup_files[] = {
  ....
  {
      .name = "memsw.limit_in_bytes",
      .private = MEMFILE_PRIVATE(_MEMSWAP, RES_LIMIT),
      .write_string = mem_cgroup_write,
      .read = mem_cgroup_read,
  },
  {
    .name = "limit_in_bytes",
    .private = MEMFILE_PRIVATE(_MEM, RES_LIMIT),
    .write_string = mem_cgroup_write,
    .read = mem_cgroup_read,
  },
  ....
}
```

仔细研究这个子系统之前，先看下mem_cgroup的应用场景：
Memcg的主要应用场景有：

a.     隔离一个或一组应用程序的内存使用

对于内存饥渴型的应用程序，我们可以通过memcg将其可用内存限定在一定的数量以内，实现与其他应用程序内存使用上的隔离。

b.    创建一个有内存使用限制的控制组

比如在启动的时候就设置mem=XXXX。

c.     在虚拟化方案中，控制虚拟机的内存大小

比如可应用在LXC的容器方案中。

d.    确保应用的内存使用量

比如在录制CD/DVD时，通过限制系统中其他应用可以使用的内存大小，可以保证录制CD/DVD的进程始终有足够的内存使用，以避免因为内存不足导致录制失败。

e.     其他

各种通过memcg提供的特性可应用到的场景。

 

为了支撑以上场景，这里也简单列举一下memcg可以提供的功能特性：

a.     统计anonymous pages, file caches, swap caches的使用并限制它们的使用；

b.    所有page都链接在per-memcg的LRU链表中，将不再存在global的LRU；

c.     可以选择统计和限制memory+swap的内存；

d.    对hierarchical的支持；

e.     Soft limit；

f.     可以选择在移动一个进程的时候，同时移动对该进程的page统计计数；

g.     内存使用量的阈值超限通知机制；

h.    可以选择关闭oom-killer，并支持oom的通知机制；

i.      Root cgroup不存在任何限制；

————————————————

**memcg** 主要有两个纬度的限制：
1. Res – 物理内存。
2. Memsw – memory + swap，即物理内存 + swap内。

**res_counter** 结构中可以看出，每个维度又有三个指标：
1. Usage – 组内进程已经使用的内存。
2. Soft_limit – 软限制，非强制内存上限，usage超过这个上限后，组内进程使用的内存可能会加快步伐进行回收。
3. Hard_limit – 硬限制，强制内存上限，usage不能超过这个上限，如果试图超过，则会触发同步的内存回收，或者触发OOM。

hard_limit是真正的内存限制，soft_limit只是为了实现更好的内存使用效果而做的辅助，而usage则是内核实时统计该组进程内存的使用值。

![Alt text](/pic/relationship.jpeg)

概括为三点：

a.     统计针对每一个cgroup进行；

b.    每个cgroup中的进程，它的mm_struct知道自己属于哪个cgroup；

c.     每个page对应一个page_cgroup，而page_cgroup知道自己属于哪个memcg；

##### 简单总结下

某进程在需要统计的地方调用mem_cgroup_charge()来进行必要的结构体设置（增加计数等），判断增加计数后进程所在的cgroup的内存使用是否超过限制，如果超过了，则触发reclaim机制进行内存回收，如果回收后依然超过限制，则触发oom或阻塞机制等待；如果增加计数后没有超过限制，则更新相应page对应的page_cgroup，完成统计计数的修改，并将相应的page放到对应的LRU中进行管理。</br>


在memcg的内存统计逻辑中，有几个 **基本思想**：

a.     一个page最多只会被charge一次，并且一般就charge在第一次使用这个page的那个进程所在的memcg上。

b.    如果有多个memcg的进程引用了同一个page，该page也只会被统计在一个memcg中。

c.     Unchage往往跟page的释放相对应，所以可能存在某个进程不再使用某个page，但是对该page的统计还是记录在进程所在的memcg中，因为可能还有其他memcg中的进程在使用这个page，只要page无法释放，memcg就无法unchage。


Page_cgroup：每个page对应一个，跟page结构体一样，在系统启动的时候或内存热插入的时候分配，在内存热拔除的时候释放。</br>
```
struct page_cgroup {
    unsigned long flags;
    struct mem_cgroup *mem_cgroup;
    unsigned short id;
};

```

Swap_cgroup：每个对应一个swp_entry，在swapon()的时候分配，swapoff()的时候释放.</br>
```
struct swap_cgroup {
    unsigned short      id;
};

```
其中的id是对应memcg在cgroup体系中的id，通过它可以找到对应的memcg.</br>

----------

###### charge 的合法性检查：（不知道是否可以charge成功，或者charge是否是合法）
1. 提供三类函数：
  a)Mem_cgroup_try_charge_XXX
  b)Mem_cgroup_commit_charge_XXX
  c)Mem_cgroup_cancel_charge_XXX
2. 在try_charge的时候，不会设置flag来说明“这个page已经被charge过了”，而只是做 usage += PAGE_SIZE：

在cancel的时候，只是简单的做usage -= PAGE_SIZE

----------

#### Memcg的reclaim流程:
说个大前提：memory cgroup的reclaim的流程和内核本身的回收代码基本融合。</br>

首先看一下，页框回收算法的执行三中基本情况：</br>
1. 内存紧缺回收
2. 睡眠回收
3. 周期回收

了解了基本页框回收算法后，再看下memory cgroup的整个reclaim流程：</br>
下面虚线框起来的部分，只有在全局reclaim的时候才会走:</br>
```
mem_cgroup_do_charge
mem_cgroup_resize_limit
mem_cgroup_memsw_limit
  -> mem_cgroup_reclaim
    -> try_to_free_mem_cgroup_pages
      -> do_try_to_free_pages
        -> shrink_zones
          -------------------------------------------
          | -> mem_cgroup_soft_limit_reclaim
          |   -> mem_cgroup_soft_reclaim
          |     -> mem_cgroup_shrink_node_zone
          |       -> shrink_lruvec
          -------------------------------------------
          |-> shrink_zone
            ->shrink_lruvec
```

即在三种情况下触发reclaim：（还有一种就是kswapd 全局回收，同时kswapd是异步回收，其他是同步）

a.     Do_charge时发现超过limit限制。

b.    修改limit设置。

c.     修改memsw_limit设置。

(d.    kswapd 会先调用mem_cgroup_soft_limit_reclaim进行group内的内存回收)

在使用memory cgroup之后，reclaim中会经常看到两个概念：**global reclaim** 和 **target reclaim**，即全局回收和局部/目标回收，前者的对象是所有的内存，后者是针对单个cgroup，但全局回收也是以单个memcg为单位的.</br>

---------

**这个地方再细分下3.10和4.14内核的情况**

**3.10**</br>
```
mem_cgroup_resize_limit
  mem_cgroup_reclaim  ========> MEM_CGROUP_RECLAIM_SHRINK | MEM_CGROUP_RECLAIM_NOSWAP
    try_to_free_mem_cgroup_pages
      do_try_to_free_pages
        shrink_zones
          shrink_zone
            shrink_lruvec //This is a basic per-zone page freer.  Used by both kswapd and direct reclaim.
              shrink_list
                shrink_active_list
                shrink_inactive_list
                  shrink_page_list //returns the number of reclaimed pages


kswapd ======================> only soft? noswap and shrink
  balance_pgdat
    kswapd_shrink_zone
      shrink_zone
      shrink_slab
    mem_cgroup_soft_limit_reclaim
      mem_cgroup_soft_reclaim
        mem_cgroup_shrink_node_zone
          shrink_lruvec

mem_cgroup_do_charge
  mem_cgroup_reclaim
    try_to_free_mem_cgroup_pages
      do_try_to_free_pages
        shrink_zones

__mem_cgroup_try_charge
  mem_cgroup_do_charge

mem_cgroup_force_empty
  try_to_free_mem_cgroup_pages

```

可以看到，在内存子系统中进行内存回收的过程和全局系统进行内存回收的过程基本一致，需要注意以下几点:
1. 每个memory group都有自己的内存域和LRU链表，内存子系统进行内存回收只是回收本组中的内存，不会对全局内存回收造成影响。（这个冲突主要发生在隔离内存页后，由于被隔离出来的页都是放到临时链表上处理，这时它不属于任何一个LRU链，这时候谁回收都可以，因为它们接下来做的工作都一样，最终页还是会被扔回buddy）
2. 在memory group中进行的内存回收，不会影响slab分配器的状态.

**Key Structure**</br>
```
struct scan_control {
    /* Incremented by the number of inactive pages that were scanned */
    unsigned long nr_scanned;

    /* Number of pages freed so far during a call to shrink_zones() */
    unsigned long nr_reclaimed;

    /* How many pages shrink_list() should reclaim */
    unsigned long nr_to_reclaim;

    unsigned long hibernation_mode;

    /* This context's GFP mask */
    gfp_t gfp_mask;

    int may_writepage;

    /* Can mapped pages be reclaimed? */
    int may_unmap;

    /* Can pages be swapped as part of reclaim? */
    int may_swap;

    int order;

    /* Scan (total_size >> priority) pages at once */
    int priority;

    bool resize;

   /*
    * The memory cgroup that hit its limit and as a result is the
    * primary target of this reclaim invocation.
    */
    struct mem_cgroup *target_mem_cgroup;

   /*
    * Nodemask of nodes allowed by the caller. If NULL, all nodes
    * are scanned.
    */
    nodemask_t  *nodemask;
};

```

**balance_pgdat**</br>

----------

对于kswapd，balance_pgdat（）将在该节点的所有区域中工作，直到它们都位于高页面（区域）。

返回kswapd在回收的最终订单

对于满是固定页的区域，这里有特殊的处理方法。如果所有页都被锁定，或者设备驱动程序（如ZONE_DMA）都在使用这些页，就会发生这种情况。或者它们都被hugetlb使用。我们要做的是检测这样的情况：区域中的所有页面都被扫描了两次，并且没有成功的回收。将区域标记为死区，从现在开始，只执行一次短扫描。基本上，当问题消失的时候，我们会对区域进行轮询。

kswapd扫描highmem->normal->dma方向的区域。它会跳过具有free_pages > high_wmark_pages(zone)，但一旦发现某个区域具有free_pages <= high_wmark_pages(zone)，我们将扫描该区域和较低区域，而不考虑较低区域中可用页的数量。此操作与页分配器回退方案进行互操作，以确保跨区域平衡页的老化。

----------

**shrink_page_list**</br>
![Alt text](/pic/shrink_page_list.jpeg)</br>
```
1、如果页面被锁住了，放入继续将页面保留在inactive list中，后就再扫描到底时候再试图回收这些page
2、如果回写控制结构体标记了不允许进行unmap操作，将那些在pte表项中有映射到页面保留在inactive list中。
3、对于正在回写中的页面，如果是同步操作，等待页面回写完成。如果是异步操作，将page继续留在inactive list中，等待以后扫描再回收释放。
4、如果检查到page又被访问了，这个时候page有一定的机会回到active list链表中。必须满足一下条件
a、page被访问，page_referenced检查
b、order小于3，也就是系统趋向于回收较大的页面。对于较小的页面趋向于保留在active list中
c、page_mapping_inuse检查，这个函数在上面的文章中已经分析

5、如果是匿名页面，并且不在swap缓冲区中，将page加入到swap的缓冲区中。
6、如果页面被映射了，调用unmap函数
7、如果页面是脏也，需要向将页面内容换出，调用pateout
8、如果页面和buffer相关联，将buffer释放掉，调用try_to_release_page函数
9、调用__remove_mapping 将页面回收归还伙伴系统。
```

有关shrink_zones的函数注释：</br>

--------------

这是页分配进程的直接回收路径。我们只尝试从满足调用方分配请求的区域中回收页面。

我们从一个区域中回收，即使该区域超过了high_wmark_pages(zone)。因为：

a）调用者可能试图释放*额外*页以满足更higher-order的分配，或者

b）目标区域可能位于high_wmark_pages(zone)，但低区域必须超过high_wmark_pages(zone)），以满足“增量最小”区域防御算法。

如果一个区域被认为满是固定页面，那么只需轻轻扫描一下，然后放弃。

如果正在为代价高昂的high-order分配回收区域，并且压缩准备就绪，则此函数返回true。这向调用方指示，它应考虑重试分配，而不是进一步回收。

---------------

**4.14**</br>
```


```

##### Soft_limit
###### 1. shrink_lruvec
```
shrink_lruvec
```
在页面回收中，shrink_lruvec后面还有很多事情要做，还要调用shrink_list – shrink_inactive_list – shrink_page_list，然后调用try_to_unmap解除映射或pageout将脏页写回磁盘，然后调free_hot_cold_page_list回收页面内存。这里还省略了很多细节，不过这些太过底层，是内核内存回收的底层固有机制，跟mem_cgroup无关，故不在本文的讨论范围内，在这里我们只需要关心如何获取到lruvec，然后调用shrink_lruvec即可完成内存回收。</br>

```
struct lruvec {
    struct list_head lists[NR_LRU_LISTS];
    struct zone_reclaim_stat reclaim_stat;
#ifdef CONFIG_MEMCG
    struct zone *zone;
#endif
};

```
就是一个lru链表的集合，可能包括active或inactive等。

所有的page都会挂载某个mem_cgroup的某个mem_cgroup_per_zone对应的lru链表上.</br>
```
mem_cgroup
  ->mem_cgroup_lru_info
    ->mem_cgroup_per_node
      ->mem_cgroup_per_zone
        ->lruvec

        struct lruvec {
            struct list_head lists[NR_LRU_LISTS];
            struct zone_reclaim_stat reclaim_stat;
        #ifdef CONFIG_MEMCG
            struct zone *zone;
        #endif
        };
```

**mem_cgroup_zoneinfo**</br>
用于获取mem_cgroup_per_zone的函数。</br>
```
static struct mem_cgroup_per_zone *
mem_cgroup_zoneinfo(struct mem_cgroup *memcg, int nid, int zid)
{
    VM_BUG_ON((unsigned)nid >= nr_node_ids);
    return &memcg->info.nodeinfo[nid]->zoneinfo[zid];
}

```
即每个mem_cgroup中有个类型为mem_cgroup_lru_info的成员info，通过它以及node_id和zone_id，即可找到对应的mem_cgroup_per_zone，从而得到lruvec。</br>
### 重点理解这几个结构体
**几个重要结构体的关系**</br>
```
mem_cgroup
->mem_cgroup_per_node
  ->mem_cgroup_per_zone[0]
        .
        .
        .
  ->mem_cgroup_per_zone[MAX_NR_ZONES]
    ->lruvec

```
**NOTE:**</br>
1. 注意 node->zone->lruvec的关系</br>
2. 一个mem_cgroup_per_zone维护了一个mem_cgroup在某个zone上使用的内存，它跟mem_cgroup是多对一的关系。</br>
3. **对内存回收的单位是lruvec**</br>
4. 内存回收的接口，其参数都是**zone**，而对于物理内存，首先分内存节点，即node，然后每个内存节点上有多个zone，而一个zone上的内存可能被多个mem_cgroup使用，但反过来，一个mem_cgroup可能使用多个zone，甚至多个node上的内存。所以mem_cgroup结构体中的info成员可以通过node_id和zone_id找到对的mem_cgroup_per_zone.</br>

**mem_cgroup_soft_limit_reclaim**
该函数的目的是回收该zone上超过soft_limit最多的mem_cgroup在该zone上mem_cgroup_per_zone对应的lru链表。

##### Soft_limit_tree
//![Alt text](/pic/soft_limit_tree.jpeg)</br>

```
/*
 * Cgroups above their limits are maintained in a RB-Tree, independent of
 * their hierarchy representation
 */

struct mem_cgroup_tree_per_zone {
    struct rb_root rb_root;
    spinlock_t lock;
};

struct mem_cgroup_tree_per_node {
    struct mem_cgroup_tree_per_zone rb_tree_per_zone[MAX_NR_ZONES];
};

struct mem_cgroup_tree {
    struct mem_cgroup_tree_per_node *rb_tree_per_node[MAX_NUMNODES];
};

         mem_cgroup_tree
      /                   \
     /                     \
mem_cgroup_tree_per_node    mem_cgroup_tree_per_node
     /                  \
    /                     \
mem_cgroup_tree_per_zone   mem_cgroup_tree_per_zone
        (rb_root)
            |
            |
    mem_cgroup_per_zone
        (rb_node)
        /         \
      /             \
mem_cgroup_per_zone  mem_cgroup_per_zone
      (rb_node)         (rb_node)
                      /           \
                     /              \
              mem_cgroup_per_zone   mem_cgroup_per_zone

```
**关于上面数的两点**
1. 只有在charge/uncharge/move时，才挂到soft_limit_tree上。
2. 当没有超过soft_limit的上限时，则将mem_cgroup_per_zone从soft_limit_tree上摘下来。
3. 综上所述，这颗soft_limit_tree是动态树.

**Q:WHY** need this rb tree structure?</br>
**A:** 目前明确有关这颗Soft_limit_tree仅用于全局回收，目的是找到超过Softlimit最多的zone。</br>
**A:** 在cgroup中的局部回收中，就目前看的代码来说，是不涉及这颗树的，所以，这样就可以明确，这棵树实际也可以认为是cgroup中的mem_cgroup间接生成的，因为引入了cgroup的概念，所以，才会有这颗的存在。</br>
**A:** 因为是mem_cgroup引入的Soft_limit_tree的新数据结构，那么，虽然代码还没看到，但我认为在开启了cgroup的情况下，也是可以进行全局回收的，需要具体看下命令。</br>

###### Regarding how to maintain this Tree

**mem_cgroup_soft_limit_tree_init**</br>

```
static void __init mem_cgroup_soft_limit_tree_init(void)
{
    struct mem_cgroup_tree_per_node *rtpn;
    struct mem_cgroup_tree_per_zone *rtpz;
    int tmp, node, zone;

    for_each_node(node) {
        tmp = node;
        if (!node_state(node, N_NORMAL_MEMORY))
            tmp = -1;
        rtpn = kzalloc_node(sizeof(*rtpn), GFP_KERNEL, tmp);
        BUG_ON(!rtpn);

        soft_limit_tree.rb_tree_per_node[node] = rtpn;

        for (zone = 0; zone < MAX_NR_ZONES; zone++) {
            rtpz = &rtpn->rb_tree_per_zone[zone];
            rtpz->rb_root = RB_ROOT;
            spin_lock_init(&rtpz->lock);
        }
    }
}

```
在初始化调用**mem_cgroup_soft_limit_tree_init**</br>
```
mem_cgroup_init
  mem_cgroup_soft_limit_tree_init

```

什么时机开始创建这颗so_limit_tree呢？check it below：</br>
```
mem_cgroup_uncharge_common
mem_cgroup_remove_from_trees
__mem_cgroup_commit_charge
  memcg_check_events
    mem_cgroup_update_tree
      __mem_cgroup_remove_exceeded
      __mem_cgroup_insert_exceeded


mem_cgroup_uncharge_common    mem_cgroup_remove_from_trees  __mem_cgroup_commit_charge
                          \             |                 /
                            \           |               /
                              \         |             /
                                \       |         /
                                  \     |       /
                                    \   |     /
                                      \ |  /
                                  memcg_check_events
                                        |
                                        |
                               mem_cgroup_update_tree
                               /                    \
                              /                       \
                             /                          \
              __mem_cgroup_remove_exceeded         __mem_cgroup_insert_exceeded

```
---------

**Conclution:**有关mem cgroup的回收</br>
没有开启CONFIG_MEMCG的情况下：</br>
page->lru,通过lru字段加入到lru链表。</br>

开启了CONFIG_MEMCG的情况：</br>
page->mem_cgroup_per_zone====> mem_cgroup.</br>
此时已经没有全局链表的概念了

相当于在zone->lru中间加入了一层node和zone

全局：</br>
&zone->lruvec

cgroup：</br>
&(memcg->info.nodeinfo[nid]->zoneinfo[zid])->lruvec

-----------

**Q：** 是每次charge/uncharge/move的时候，都会将超softlimit的zone加入到Soft_limit_tree中吗？即使是cgroup的局部回收的情况？</br>

**Q:** 通过阅读shrink_zone有关局部回收的代码，有些疑问，如下：</br>
比如一个cgroup中的某个进程进行charge，此时内存不足，那么进行reclaim操作，为什么在reclaim时，会有zone树的存在？还需要进行zonelist的循环？本mem_cgroup对应的各个类型的zong不是可以了吗？为什么在某个zone类型下面扫描树呢？</br>

**A:** 我觉得可能的原因是，cgroup的节点，毕竟是一个进程组，那么不同的进程使用的内存会间接使zone形成树。</br>
有关这个问题，看了下mem_cgroup_iter()函数代码，发现在开启root->use_hierarchy时，会涉及zone的树问题(确切说，这棵树是per subtree)，需要进一步看代码分析。</br>

下面一段总结很好：</br>
Memcg的hierarchy

对于memcg，作为一个cgroup的subsystem，它遵循hierarchy的所有规则，另外，对于hierarchy中cgroup的层级对memcg管理规则的影响，主要分两方面：

1、 如果不启用hierarchy，即mem_cgroup->use_hierarchy =false，则所有的memcg之间都是互相独立，互不影响的，即使是父子cgroup之间，也跟两个单独创建的cgroup一样。

2、 如果启用hierarchy，即mem_cgroup->use_hierarchy =true，则memcg的统计需要考虑hierarchy中的层级关系，其影响因素主要有：

a.     Charge/uncharge

如果子cgroup中charge/uncharge了一个page，则其父cgroup和所有祖先cgroup都要charge/uncharge该page。

b.    Reclaim

因为父cgroup的统计中包含了所有子cgroup中charge的page，所以在回收父cgroup中使用的内存时，也可以回收子cgroup中进程使用的内存。

**就目前看的代码情况看，这个hierarchy不仅仅时cgroup节点对应的树，还应该包括每个节点内部的subtree，类似父子进程的情况，都会创建mem_cgroup**</br>

```
static void shrink_zone(struct zone *zone, struct scan_control *sc)
{
  ...
  do {
    ...
    memcg = mem_cgroup_iter(root, NULL, &reclaim);
    do {
        lruvec = mem_cgroup_zone_lruvec(zone, memcg);
        shrink_lruvec(lruvec, sc, memcg);
        ...
        memcg = mem_cgroup_iter(root, memcg, &reclaim);
    } while (memcg);

  } while (should_continue_reclaim(zone, sc->nr_reclaimed - nr_reclaimed,sc->nr_scanned - nr_scanned, sc));
  ...
}

```

#### Memcg的命令的实现：</br>



#### CPU的子系统实现：</br>
理解CPU子系统的实现机制，关键是理解内核中对进程的调度过程。调度器在进程选择时不会区分进程和进程组，所以在进程选择时可以把进程组当做进程来看待，只是在选择出“进程”之后才会对他们进行区分。对进程组shares值的修改和对进程nice值修改达到的效果其实是相似的。</br>

对cpu实现隔离是通过cpu带宽限制实现，某个进程组没有时间片了，就需要throttle。</br>
对task进行throttle的本质，就是将task group的 task se 从rq中删除，并同时将se所在的cfs_rq挂到throttle_list链表上。</br>

![Alt text](/pic/task_group_relationship.png)</br>
在每个CPU上都有一个全局的就绪队列struct rq，在4个CPU的系统上会有4个全局就绪队列，如图中紫色结构体。系统默认只有一个根task_group叫做root_task_group。rq->cfs_rq指向系统根CFS就绪队列。根CFS就绪队列维护一棵红黑树，红黑树上一共10个就绪态调度实体，其中9个是task se，1个group se（图上蓝色se）。group se的my_q成员指向自己的就绪队列。该就绪队列的红黑树上共9个task se。其中parent成员指向group se。每个group se对应一个group cfs_rq。4个CPU会对应4个group se和group cfs_rq，分别存储在task_group结构体se和cfs_rq成员。se->depth成员记录se嵌套深度。最顶层CFS就绪队列下的se的深度为0，group se往下一层层递增。cfs_rq->nr_runing成员记录CFS就绪队列所有调度实体个数，不包含子就绪队列。cfs_rq->h_nr_running成员记录就绪队列层级上所有调度实体的个数，包含group se对应group cfs_rq上的调度实体。例如，图中上半部，nr_runing和h_nr_running的值分别等于10和19，多出的9是group cfs_rq的h_nr_running。group cfs_rq由于没有group se，因此nr_runing和h_nr_running的值都等于9。</br>


关于进程组权重的计算：CPU0上group se权重是439（1024*3072/(3072+4096)）,CPU1上group se权重是585（1024-439）</br>
在引入权重之后，分配给进程的时间计算公式如下：</br>
分配给进程的时间 = 总的cpu时间 * 进程的权重/就绪队列（runqueue）所有进程权重之和 </br>
![Alt text](/pic/weight.png)</br>
