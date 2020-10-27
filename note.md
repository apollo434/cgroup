### 用于记录目前看到的Key Point!

对开发者来说，cgroups 有如下四个有趣的特点：

1. cgroups 的 API 以一个**伪文件系统**的方式实现，即用户可以通过文件操作实现 cgroups 的组织管理。</br>
2. cgroups 的组织管理操作单元可以细粒度到线程级别，**用户态代码也可以针对系统分配的资源创建和销毁 cgroups**, 从而实现资源再分配和管理。</br>
3. 所有资源管理的功能都以“subsystem（子系统）”的方式实现，**接口统一**。</br>
4. 子进程创建之初与其父进程处于**同一个 cgroups 的控制组**。</br>


本质上来说，cgroups 是内核附加在程序上的**一系列钩子**（hooks），通过程序运行时对资源的调度触发相应的钩子以达到资源追踪和限制的目的。</br>

实现 cgroups 的主要目的是为不同用户层面的资源管理，提供一个**统一化的接口。** </br>

subsystem: 它类似于我们在netfilter中的过滤hook.比如上面的CPU占用率就是一个subsystem.简而言之.subsystem就是cgroup中可添加删除的模块.在cgroup架构的封装下为cgroup提供多种行为控制.subsystem在下文中简写成subsys.</br>

##### cgroups 的作用

1. 资源限制（Resource Limitation）：cgroups 可以对进程组使用的资源总额进行限制。如设定应用运行时使用内存的上限，一旦超过这个配额就发出 OOM（Out of Memory）。
2. 优先级分配（Prioritization）：通过分配的 CPU 时间片数量及硬盘 IO 带宽大小，实际上就相当于控制了进程运行的优先级。
3. 资源统计（Accounting）： cgroups 可以统计系统的资源使用量，如 CPU 使用时长、内存用量等等，这个功能非常适用于计费。
4. 进程控制（Control）：cgroups 可以对进程组执行挂起、恢复等操作。

**NOTE:** 以上四条非常非常重要，洗完可以对照着下表去体会:</br>
![Alt text](/pic/cgroup.png)</br>

##### 术语表

1. task（任务）：cgroups 的术语中，task 就表示系统的一个进程。
2. cgroup（控制组）：cgroups 中的**资源控制**都以**cgroup 为单位**实现。cgroup 表示按某种资源控制标准划分而成的任务组，包含一个或多个子系统。一个任务可以加入某个 cgroup，也可以从某个 cgroup 迁移到另外一个 cgroup。
3. subsystem（子系统）：cgroups 中的 subsystem 就是一个**资源调度控制器**（Resource Controller）。比如 CPU 子系统可以控制 CPU 时间分配，内存子系统可以限制 cgroup 内存使用量。
4. hierarchy（层级树）：hierarchy 由**一系列 cgroup** 以一个**树状**结构排列而成，每个 hierarchy 通过**绑定**对应的 **subsystem** 进行资源调度。hierarchy 中的 cgroup 节点可以包含零或多个子节点，子节点继承父节点的属性。整个系统可以有多个 hierarchy。</br>
**NOTE:** cgroups 的模型则是由多个 hierarchy 构成的森林</br>
Why?</br>
Because:如果只有一个 hierarchy，那么所有的 task 都要受到绑定其上的 subsystem 的限制，会给那些不需要这些限制的 task 造成麻烦。</br>


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


#### cgroups 实现方式及工作原理简介

##### cgroups 实现结构讲解

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
### 框架分析

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
struct cgroup_subsys {
     // 下面的是函数指针，定义了子系统对css_set结构的系列操作
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

--------

### 父子进程之间的cgroup关联:

在上面看到的代码中.将init_task.cgroup设置为了init_css_set.我们知道,init_task是系统的第一个进程.所有的过程都是由它创建的.init_task.cgroup到底会在它后面的子进程造成什么样的影响呢?接下来我们就来分析这个问题.</br>

#### 创建进程时的父子进程cgroup关联
在进程创建的时候,有:do_fork()àcopy_process(),有如下代码片段:</br>
```
static struct task_struct *copy_process(unsigned long clone_flags,
                    unsigned long stack_start,
                    struct pt_regs *regs,
                    unsigned long stack_size,
                    int __user *child_tidptr,
                    struct pid *pid,
                    int trace)
{
    ……
    ……
    cgroup_fork(p);
    ……
    cgroup_can_ fork(p);
    ……
    cgroup_post_fork(p);
    ……
}

```
上面的代码片段是创建新进程的时候与cgroup关联的函数.挨个分析如下
```
/**
 * cgroup_fork - initialize cgroup related fields during copy_process()
 * @child: pointer to task_struct of forking parent process.
 *
 * A task is associated with the init_css_set until cgroup_post_fork()
 * attaches it to the parent's css_set.  Empty cg_list indicates that
 * @child isn't holding reference to its css_set.
 */
void cgroup_fork(struct task_struct *child)
{
	RCU_INIT_POINTER(child->cgroups, &init_css_set);
	INIT_LIST_HEAD(&child->cg_list);
}

```
如上面代码所示,子进程和父进程指向同一个cgroups.并且由于增加了一次引用.所以要调用get_css_set()来增加它的引用计数.最后初始化child->cg_list链表.</br>
如代码注释上说的,这里就有一个问题了:在dup_task_struct()为子进程创建struct task_struct的时候不是已经复制了父进程的cgroups么?为什么这里还要对它进行一次赋值呢?这里因为在dup_task_struct()中没有持有保护锁.而这里又是一个竞争操作.因为在cgroup_attach_task()中可能会更改进程的cgroups指向.因此通过cgroup_attach_task()所得到的cgroups可能是一个无效的指向.在递增其引用计数的时候就会因为它是一个无效的引用而发生错误.所以,这个函数在加锁的情况下进行操作.确保了父子进程之间的同步.</br>


```
/**
 * cgroup_can_fork - called on a new task before the process is exposed
 * @child: the task in question.
 *
 * This calls the subsystem can_fork() callbacks. If the can_fork() callback
 * returns an error, the fork aborts with that error code. This allows for
 * a cgroup subsystem to conditionally allow or deny new forks.
 */
int cgroup_can_fork(struct task_struct *child)
{
	struct cgroup_subsys *ss;
	int i, j, ret;

	do_each_subsys_mask(ss, i, have_canfork_callback) {
		ret = ss->can_fork(child);
		if (ret)
			goto out_revert;
	} while_each_subsys_mask();

	return 0;

out_revert:
	for_each_subsys(ss, j) {
		if (j >= i)
			break;
		if (ss->cancel_fork)
			ss->cancel_fork(child);
	}

	return ret;
}

```

```
/**
 * cgroup_post_fork - called on a new task after adding it to the task list
 * @child: the task in question
 *
 * Adds the task to the list running through its css_set if necessary and
 * call the subsystem fork() callbacks.  Has to be after the task is
 * visible on the task list in case we race with the first call to
 * cgroup_task_iter_start() - to guarantee that the new task ends up on its
 * list.
 */
void cgroup_post_fork(struct task_struct *child)
{
	struct cgroup_subsys *ss;
	int i;

	/*
	 * This may race against cgroup_enable_task_cg_lists().  As that
	 * function sets use_task_css_set_links before grabbing
	 * tasklist_lock and we just went through tasklist_lock to add
	 * @child, it's guaranteed that either we see the set
	 * use_task_css_set_links or cgroup_enable_task_cg_lists() sees
	 * @child during its iteration.
	 *
	 * If we won the race, @child is associated with %current's
	 * css_set.  Grabbing css_set_lock guarantees both that the
	 * association is stable, and, on completion of the parent's
	 * migration, @child is visible in the source of migration or
	 * already in the destination cgroup.  This guarantee is necessary
	 * when implementing operations which need to migrate all tasks of
	 * a cgroup to another.
	 *
	 * Note that if we lose to cgroup_enable_task_cg_lists(), @child
	 * will remain in init_css_set.  This is safe because all tasks are
	 * in the init_css_set before cg_links is enabled and there's no
	 * operation which transfers all tasks out of init_css_set.
	 */
	if (use_task_css_set_links) {
		struct css_set *cset;

		spin_lock_irq(&css_set_lock);
		cset = task_css_set(current);
		if (list_empty(&child->cg_list)) {
			get_css_set(cset);
			css_set_move_task(child, NULL, cset, false);
		}
		spin_unlock_irq(&css_set_lock);
	}

	/*
	 * Call ss->fork().  This must happen after @child is linked on
	 * css_set; otherwise, @child might change state between ->fork()
	 * and addition to css_set.
	 */
	do_each_subsys_mask(ss, i, have_fork_callback) {
		ss->fork(child);
	} while_each_subsys_mask();
}

```
