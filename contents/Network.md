# Network

# Basic: Linux Network Virtualization

## network namespce

### 配置

**namespace: "隔离内核资源"**

- Mount
- UTS
- IPC
- PID
- network
- user

netns 通过系统调用创建，调用Linux的clone()（UNIX系统调用fork()的延伸）API创建一个通用的namespace，然后传入CLONE_NEWNET参数创建一个network namespace.

```bash
# create one netns
$ ip netns add netns1
# del one netns
$ ip netns delete netns1

# check netns
$ ls /var/run/netns/netns1

# 查询配置
$ ip netns exec netns1 ip link list

# ns 自带的lo设备状态默认为Down
$ ip netns exec netns1 ping 127.0.0.1
# connect: Network is unreachable

# start lo
$ ip netns exec netns1 ip link set dev lo up
```

NS只有仅有lo无法与外界通信，需要有一对虚拟的以太网卡: **veth pair**

**veth pair总是成对出现且相互连接**

```bash
# 创建一对 veth pair
$ ip link add veth0 type veth peer name veth1
# mv veth1 from /ns to netns1
$ ip link set veth1 netns netns1

# start up
$ ip netns exec netns1 ifconfig veth1 10.1.1.1/24 up # 初始化一个ip addr
$ ifconfig veth0 10.1.1.2/24 up # / ns veth0 set ip addr
# ping veth0/1 OK but public net is not ok
```

**用户可随意将虚拟网络设备分配到自定义的ns中，但连接物理设备的则必须在根netns中，并且一个设备只能在一个ns中**

```bash
# root 进程可以任意移动不同ns中的设备
$ ip netns exec netns1 ip link set veth1 netns 1
```

### API的使用

network namespace的API涉及三个Linux系统调用: clone、unshare、setns，以及一些系统/proc目录下的文件

clone()、unshare()、setns()系统调用会使用CLONE_NEW*常量来区别要操作的namespace类型

- CLONE_NEWIPC
- CLONE_NEWNS
- CLONE_NEWNET
- CLONE_NEWPID
- CLONE_NEWUSER
- CLONE_NEWUTS
1. 创建namespace: clone系统调用
2. 维持namespace存在: /proc/PID/ns目录

    作用:

    a. 若两个进程在同一个namespace中，则两个进程/proc/PID/ns目录下的对应符号连接文件的inode数字相同

    b. 文件描述符保持open状态，对应的namespace就会一致存在，即使这个namespace里的进程都终止运行了

3. 往namespace里添加进程: setns系统调用

    setns(): 把一个进程加入一个已经存在的namespace中

    ```c
    int setns(int fd, int nstype);
    ```

4. 帮助进程逃离namespace: unshare系统调用

    unshare(): 用于帮助进程"逃离"namespace

    ```c
    int unshare(int flags); 
    ```

    unshare() 与 clone()区别:

    - unshare()作用在一个已存在的进程上
    - clone()会创建一个新的进程
5. 一个完整的例子

    ```c
    # test ns
    ```

## veth pair

## veth

是虚拟以太网卡。总是成对出现，veth pair在转发过程中不会篡改数据包的内容。

```bash
# 创建veth pair
$ ip link add veth0 type veth peer name veth1
# 查看
$ ip link list
$ ip link set dev veth0 up
$ ip link set dev veth1 up
# set ip
$ ifconfig veth0 10.20.30.40/24
# 移动namespace
$ ip link set veth1 netns newn
```

容器组网模型: veth pair + bridge的模式

```bash
# 查看对应关系
# method one
$ cat /sys/class/net/eth0/iflink
# 遍历/sys/class/net 子目录ifindex的值和上面的值相等 对应的就是veth pair的对端
$ for i in `ls /sys/class/net/`; do echo $i; cat /sys/class/net/$i/ifindex ;done

# method two
$ ip link show eth0 # num
$ ip link show | grep num
```

### Linux bridge