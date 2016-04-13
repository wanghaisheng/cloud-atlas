# 默认的Cgroup层次结构

默认情况下`systemd`自动创建一个`slice`（片），`scope`（范围）和`service`（服务）单元来提供cgroup树的统一结构。使用`systemctl`命令，可以通过创建定制的`slice`来修改这个层次结构。并且，`systemd`为重要的内核资源控制器在`/sys/fs/cgroup/`目录下自动挂载层次结构。

> **警告**

通过`libcgroup`软件包提供的`cgconfig`工具可以挂载和处理控制器的层次结构，但不被`systemd`所支持（尤其是`net-prio`控制器）。绝对不要使用`libcgroup`工具来修改通过`systemd`所挂载的默认层次结构，因为这样会导致不可预知的错误。`libcgroup`软件包将在未来的Red Hat Enterprise Linux版本中移除。

## Systemd单元类型

> `systemd`是Red Hat Enterprise Linux 7开始取代`init`的服务管理系统，本段落内容适用于RHEL 7。

所有运行在系统（RHEL 7）中的进程都是`systemd` init进程的子进程。`systemd`提供了三种单元类型用于资源控制目的：

* `Service`服务 - 一个进程或者一组进程，由`systemd`基于单元(unit)配置文件启动。服务包装了特定的进程，这样它们可以作为一个整体进行启动或停止。服务被命名成如下方式：

```bash
name.service
```

* `Scope`范围 - 一组外部创建的进程。scope包装了通过`fork()`功能启动和停止的进程，然后在运行时由`systemd`注册进程。例如，用户会话，容器和虚拟机就被视为`scope`。scope被命名成如下方式：

```bash
name.scope
```

* `slice`分片 - 使用unit层次化组织的组。`slice`不包含进程，它是由`scope`和`service`组成的层次结构。活跃的进程则包含在scope或service中。在层次结构树中，每个`slice`单元的命名都对应了层级结构中指向目标的路径。符号`-`就表示路径组件特征。例如，如果一个slice类似如下：

```bash
parent-name.slice
```

则表示这个`parent-name.slice`是`parent.slice`的`subslice`，这个`slice`也可以有自己的`subslice`，例如`parent-name-name2.slice`，依次类推。这里的根`slice`是`-.slice`。

`service`，`scope`和`slice`单元在cgroup树中直接被映射到对象。当这些单元被激活，每个单元都各自直接映射到单元名字对应的cgroup路径。例如位于`test-waldo.slice`中的`ex.service`被映射到cgroup的`test.slice/test-waldo.slice/ex.service/`。

`service`，`scope`和`slice`可以由系统管理员手工创建或者由程序动态创建。默认情况，操作系统定义了一组运行系统所必须的服务。即，默认有4个`slice`：

* `-.slice` - 根`slice`
* `system.slice` - 所有系统服务的默认存放位置
* `user.slice` - 所有用户会话的默认存放位置
* `machine.slice` - 所有虚拟机和容器的默认存放位置

注意，所有用户会话被自动放到一个独立的scope单元，虚拟机和容器进程也是这样处理的。而且，所有用户被指派到一个默认的subslice。在这个默认配置中，系统管理员可以定义一个新的slice并指定service和scope到这个slice中。

以下是`systemd-cgls`命令的输出案例：

```bash
├─1 /usr/lib/systemd/systemd --switched-root --system --deserialize 21
├─user.slice
│ └─user-1000.slice
│   └─session-2.scope
│     ├─2069 sshd: vagrant [priv]
│     ├─2072 sshd: vagrant@pts/0
│     ├─2073 -bash
│     ├─2253 systemd-cgls
│     └─2254 systemd-cgls
└─system.slice
  ├─postfix.service
  │ ├─1200 /usr/libexec/postfix/master -w
  │ ├─1216 qmgr -l -t unix -u
  │ └─2250 pickup -l -t unix -u
  ├─sshd.service
  │ └─888 /usr/sbin/sshd -D
  ...
```

上述案例可以看到`service`和`scope`包含进程，并且`service`和`scope`又被包含在`slice`中，而`slice`是不直接包含进程的。唯一的例外是`PID`为`1`的进程，位于特殊的`systemd.slice`中。还要注意，`-.slice`并没有显示在输出中，这是因为它表示了整个cgroup树。

`service`和`slice`单元可以通过单元文件配置或者动态通过调用`PID 1`的API来创建。`scope`单元则只能使用单元配置文件的方法来创建。通过API调用方法动态创建的单元是临时的并且只在运行期间存在。这些短暂存在的单元在单元结束时、禁用时或系统重启时自动释放。

# Linux内核中的资源控制器

资源控制器（resource controller），也称为cgroup子系统（cgroup subsystem），相当于一个单一资源，例如CPU时间或内存。Linux内核提供了一系列的资源控制器，它们都通过`systemd`自动挂载。

在`/proc/cgroups`中列出了当前挂载的resource controller列表

```bash
cat /proc/cgroups
```

输出内容

```bash
#subsys_name	hierarchy	num_cgroups	enabled
cpuset	7	1	1
cpu	5	1	1
cpuacct	5	1	1
memory	8	1	1
devices	10	1	1
freezer	6	1	1
net_cls	3	1	1
blkio	2	1	1
perf_event	4	1	1
hugetlb	9	1	1
```

也可以通过`lssubsys`(该命令属于`libcgroup-tools`软件包)，其中`-i`参数表示层次结构，`-m`参数表示输出挂载

```bash
lssubsys -i
```

输出

```bash
cpuset 7
cpu,cpuacct 5
memory 8
devices 10
freezer 6
net_cls 3
blkio 2
perf_event 4
hugetlb 9
```

```bash
lssubsys -m
```

```bash
cpuset /sys/fs/cgroup/cpuset
cpu,cpuacct /sys/fs/cgroup/cpu,cpuacct
memory /sys/fs/cgroup/memory
devices /sys/fs/cgroup/devices
freezer /sys/fs/cgroup/freezer
net_cls /sys/fs/cgroup/net_cls
blkio /sys/fs/cgroup/blkio
perf_event /sys/fs/cgroup/perf_event
hugetlb /sys/fs/cgroup/hugetlb
```

在RHEL 7中可用的资源控制器：

* `blkio` - 设置访问块设备的`I/O`存取限制
* `cpu` - 设置CPU调度算法提供cgroup调度任务访问CPU。这个挂载是和`cpuacct`控制器同时挂载的
* `cpuacct` - 通过使用cgroup的任务创建CPU资源的自动报告。这个挂载是和`cpu`控制器同时挂载的
* `cpuset` - 在多核系统中指定独立的CPU和内存节点给cgroup的任务
* `devices` - 允许或拒绝cgroup中的任务访问设备
* `freezer` - 挂起或继续cgroup重大任务
* `memory` - 通过cgroup中的任务设置内存限制，并通过其他任务来自动创建内存资源的报告
* `net_cls` - 通过分类标记（classid）来标记网络数据包，这样Linux流量控制器（`tc`命令）从参与的cgroup任务中标记数据包。
* `perf_event` - 通过`perf`工具激活监控cgroups
* `hugetlb` - 允许使用大的虚拟内存也，并强化资源限制在这些内存页

Linux内核针对可以通过`systemd`配置的资源控制来提供广泛的性能调优参数。

# 相关信息资源

## cgroup相关的systemd文档

以下手册页（man）包含了`systemd`下的cgroup通用信息

* `systemd.resource-control` - 描述通过system单元共享的资源控制配置选项
* `systemd.unit` - 描述所有单元配置文件的常用项
* `systemd.slice` - 提供`.slice`单元的通用信息
* `systemd.scope` - 提供`.scope`单元的通用信息
* `systemd.service` - 提供`.service`单元的通用信息

## 控制器相关内核文档

内核文档软件包提供了详细的有关所有资源控制器的文档。这个软件包包含了可选的订阅通道。

```bash
yum install kernel-doc
```

安装完成后，相关文件位于 `/usr/share/doc/kernel-doc-<kernel_version>/Documentation/cgroups/`目录。

## 在线文档

* [Red Hat Enterprise Linux 7 System Administrators Guide](https://access.redhat.com/documentation/en-US/Red_Hat_Enterprise_Linux/7/html/System_Administrators_Guide/index.html) - 详细解释了systemd概念和相关的服务管理方法
* [The D-Bus API of systemd](https://www.freedesktop.org/wiki/Software/systemd/dbus/) - 有关使用D-Bus API命令和`systemd`交互的参考手册