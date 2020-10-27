### Cgroup基本概念 ###

A **cgroup** associates a set of tasks with a set of parameters for one
or more subsystems.

 Cgroups 是 control groups 的缩写,通俗的来说，cgroups可以限制、记录、隔离进程组所使用的物理资源（包括：CPU、memory、IO等），为容器实现虚拟化提供了基本保证，是构建Docker等一系列虚拟化管理工具的基石。最初由 google 的工程师提出,后来被整合进 Linux 内核。Cgroups 也是 LXC 为实现虚拟化所使用的资源管理手段,可以说没有 cgroups 就没有 LXC。

 Cgroups 最初的目标是为资源管理提供的一个**统一的框架**,既整合现有的 cpuset 等子系统,
 也为未来开发新的子系统提供接口。

 对开发者来说，cgroups有如下四个有趣的特点：
1. cgroups的API以一个**伪文件系统**的方式实现，即用户可以通过文件操作实现 cgroups的组织管理
2. cgroups的组织管理操作单元可以细粒度到**线程级别**，用户态代码也可以针对系统分配的资源创建和销毁cgroups，从而实现资源再分配和管理。
3. 所有资源管理的功能都以“**subsystem（子系统）**”的方式实现，接口统一。
4. 子进程创建之初与其父进程处于**同一个cgroups的控制组**。

#### cgroups的作用 ####

 实现cgroups的主要目的是为不同用户层面的资源管理，提供一个统一化的接口。从单个进程的资源控制到操作系统层面的虚拟化。Cgroups提供了以下四大功能</br>

**资源限制**（Resource Limitation）：cgroups可以对进程组使用的资源总额进行限制。如设定应用运行时使用内存的上限，一旦超过这个配额就发出OOM（Out of Memory）。</br>
**优先级分配**（Prioritization）：通过分配的CPU时间片数量及硬盘IO带宽大小，实际上就相当于控制了进程运行的优先级。</br>
**资源统计**（Accounting）： cgroups可以统计系统的资源使用量，如CPU使用时长、内存用量等等，这个功能非常适用于计费。</br>
**进程控制**（Control）：cgroups可以对进程组执行挂起、恢复等操作。</br>

过去有一段时间，内核开发者甚至把namespace也作为一个cgroups的subsystem加入进来，也就是说cgroups曾经甚至还包含了资源隔离的能力。但是资源隔离会给cgroups带来许多问题，如PID在循环出现的时候cgroup却出现了命名冲突、cgroup创建后进入新的namespace导致脱离了控制等等

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


创建了 cgroups 层级结构中的节点（cgroup 结构体）之后，可以把进程加入到某一个节点的控制任务列表中，一个节点的控制列表中的所有进程都会受到当前节点的资源限制。同时某一个进程也可以被加入到不同的 cgroups 层级结构的节点中，因为不同的 cgroups 层级结构可以负责不同的系统资源。所以说进程和 cgroup 结构体是一个多对多的关系。

![Alt text](/pic/1.png)


常见的4个

task（任务）：cgroups的术语中，task就表示系统的一个进程。

cgroup（控制组）：cgroups 中的资源控制都以cgroup为单位实现。cgroup表示按某种资源控制标准划分而成的任务组，包含一个或多个子系统。一个任务可以加入某个cgroup，也可以从某个cgroup迁移到另外一个cgroup。

subsystem（子系统）：cgroups中的subsystem就是一个资源调度控制器（Resource Controller）。比如CPU子系统可以控制CPU时间分配，内存子系统可以限制cgroup内存使用量。

hierarchy（层级树）：hierarchy由一系列cgroup以一个树状结构排列而成，每个hierarchy通过绑定对应的subsystem进行资源调度。hierarchy中的cgroup节点可以包含零或多个子节点，子节点继承父节点的属性。整个系统可以有多个hierarchy。

另外整理的：

1. css_set:一组关联cgroup的集合.
2. cgroupfs_root:代表一个hierarchy
3. cgroup_subsys:代表一个subsystem

subsystem和hierarchy绑定的限制关系：

1.  一个hierarchy上可以绑定一个或者多个subsystem.例如 cpu & memory 绑定到了同一个hierarchy
2. 一个subsystem不能从某个hierarchy解绑然后绑定到其他的hierarchy 上。 但是当且仅当这些hirearchy的subsystem相同时可以绑定，可以理解为给hierarchy起了别名。(subsystem不能出现在不同的hierarchy上，但是你拷贝了一个hierarchy则是可以的)。**why???????**
3. 创建一个hierarchy的时候，系统所有css_set都会和此hierarchy的root cgroup关联。也就相当于所有的task都和root cgroup关联。但是在同一个hierarchy中，一个css_set只能和一个cgroup关联。
4. fork子进程的时候父子进程在同一个cgroup.但是后续可以修改。

#### 实践操作 ####
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

#### cgroup how to create: ####

/cgroup
```
cgroup_init
  err = register_filesystem(&cgroup_fs_type);

static struct file_system_type cgroup_fs_type = {
        .name = "cgroup",
        .mount = cgroup_mount,
        .kill_sb = cgroup_kill_sb,
};

```

Create a directory in cgroup, like /cgroup/xxxx
```
cgroup_mount
  ret = cgroup_setup_root(root, opts.subsys_mask);

cgroup_setup_root
  root->kf_root = kernfs_create_root(&cgroup_kf_syscall_ops,KERNFS_ROOT_CREATE_DEACTIVATED,root_cgrp)

static struct kernfs_syscall_ops cgroup_kf_syscall_ops = {
        .remount_fs             = cgroup_remount,
        .show_options           = cgroup_show_options,
        .mkdir                  = cgroup_mkdir,
        .rmdir                  = cgroup_rmdir,
        .rename                 = cgroup_rename,
};


cgroup_mkdir()
{



}
```

**the key flow chart**

![Alt text](/pic/cgroup.png)
