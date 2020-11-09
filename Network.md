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
```