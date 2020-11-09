# Network

# Basic: Linux Network Virtualization

## network namespce

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
# 
```