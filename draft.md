#### Basic concept: ####

A **cgroup** associates a set of tasks with a set of parameters for one
or more subsystems.

**"**A **subsystem** is a module that makes use of the task grouping
facilities provided by cgroups to treat groups of tasks in
particular ways.**"** A subsystem is typically a "resource controller" that
schedules a resource or applies per-cgroup limits, but it may be
anything that wants to act on a group of processes, e.g. a
virtualization subsystem.

A **hierarchy** is a set of cgroups arranged in a tree, such that
every task in the system is in exactly one of the cgroups in the
hierarchy, and a set of subsystems; each subsystem has system-specific
state attached to each cgroup in the hierarchy.  Each hierarchy has
an instance of the cgroup virtual filesystem associated with it.


按照资源的划分，系统被划分成了不同的子系统(subsystem)，正如我们上面列出的cpu, cpuset, blkio...每种资源独立构成一个subsystem.

可以将cgroup的架构抽象的理解为多根的树结构，一个hierarchy代表一棵树，树上绑定一个或多个subsystem.而树的叶子则是cgroup,一个cgroup具体的限制了某种资源。一个或多个cgroup组成一个css_set。简单来讲，就是一个资源限制集合(css_set)对一种subsystem(cpu，devices)的限制条件只能有一个，这是显然的吧...最终的task(进程)同css_set关联，从而达到限制资源的目的。具体cgroup和css_set 关联的方式, **see the second chart**


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
2. 一个subsystem不能从某个hierarchy解绑然后绑定到其他的hierarchy 上。 但是当且仅当这些hirearchy的subsystem相同时可以绑定，可以理解为给hierarchy起了别名。(subsystem不能出现在不同的hierarchy上，但是你拷贝了一个hierarchy则是可以的)。
3. 创建一个hierarchy的时候，系统所有css_set都会和此hierarchy的root cgroup关联。也就相当于所有的task都和root cgroup关联。但是在同一个hierarchy中，一个css_set只能和一个cgroup关联。
4. fork子进程的时候父子进程在同一个cgroup.但是后续可以修改。


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
